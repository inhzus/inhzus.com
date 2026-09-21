+++
title = "Vespa: Thread Safety in the Storage Engine"
date = "2026-09-20"
description = "How proton coordinates writes, protects readers, and replaces memory and disk indexes."

[extra]
toc = true
+++

The earlier posts covered [index data structures](@/posts/2024-12-04-vespa-index-impl.md) and [matching](@/posts/2024-12-05-vespa-match-impl.md). One question remains: while a query is using those structures, how can another thread update them or reclaim their memory?

This post looks at that question in proton, Vespa's searchnode. We start with generation tracking, then follow its use in attributes and the memory index. Disk indexes use a different mechanism, which comes last. Source links refer to revision [39bf68f4](https://github.com/vespa-engine/vespa/tree/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2).

## Overview

For many mutable structures, Vespa sequences writes so that only one writer operates on a structure at a time. Different fields can still be updated concurrently. Readers use published references, frozen B-tree nodes, and generation guards to access data while writes continue.

There are two separate problems here:

- Publishing a change so that readers can access correctly initialized data.
- Keeping replaced data alive until readers have finished using it.

Atomic loads/stores, tree freezing, and locks help with the first. Generation tracking handles much of the second. Taking a generation guard alone does not give a query a fixed snapshot of every field.

For disk indexes, the posting and dictionary files are stable after construction. A new index is written separately, and the searchable collection is replaced under a lock. Readers retain shared ownership of the collection they acquired.

## GenerationHandler

[GenerationHandler][generation-handler] tracks the current generation and older generations still held by readers. It has one writer and supports multiple readers.

Its main state consists of a current generation number, a linked list of `GenerationHold` entries, and a free list for entries that can be reused. `_last` identifies the generation acquired by new readers; `_first` identifies the oldest retained entry.

### Acquiring a guard

`takeGuard()` can be simplified to:

```cpp
GenerationGuard guard(_last.load(std::memory_order_acquire));
while (!guard.valid()) {
    guard = GenerationGuard(_last.load(std::memory_order_acquire));
}
return guard;
```

The constructor tries to acquire the `GenerationHold`. If that entry was invalidated concurrently, acquisition fails and the reader retries. A successful guard releases its reference when destroyed.

[GenerationHold][generation-hold] packs the reader count and an invalid flag into one atomic integer. Each reader adds 2, leaving the lowest bit for the flag. `setInvalid()` succeeds only when it can change the value from 0 to 1, so an entry with an active reader cannot be invalidated through that path.

The hold entries remain allocated for reuse while the handler is alive. Retiring an entry from the active list is therefore different from freeing the entry's allocation underneath a reader attempting to acquire it.

### Advancing the generation

The writer calls `incGeneration()` after the surrounding structure has performed its publication work.

If the newest hold has no readers, the handler can reuse it with the next generation number. Otherwise, it allocates or reuses another hold and publishes it through `_last` with release ordering.

`update_oldest_used_generation()` walks from the oldest entry until it reaches one still in use, or the current entry. Stores use this oldest-used generation to decide which retired allocations can be reclaimed.

For example, suppose a reader holds generation 10 and a writer replaces an array associated with that generation. The writer can publish the replacement and proceed to generation 11, but the old array must remain on hold. Once the reader leaves and the reclamation boundary advances past 10, the store can reuse that memory.

[![Timeline showing a reader holding generation 10 while the writer publishes array B and retires array A. A becomes reclaimable only after the guard is released and the oldest-used generation advances past 10.](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/generation-lifetime.svg)](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/generation-lifetime.svg)

*A generation guard protects storage lifetime; it does not select a fixed version of every value.*

A slow reader can consequently keep a considerable amount of replaced data alive. This is why on-hold memory matters when examining memory usage during heavy updates.

## DataStore and B-trees

Generation tracking needs cooperation from the data structures that own memory. [DataStoreBase][datastore] records retired elements and buffers on hold lists, assigns them a generation, and reclaims them when the oldest reader generation has advanced far enough.

Compaction follows the same rule. Moving live data to another buffer does not immediately make the old buffer reusable: a reader may already have resolved an old reference into it.

### Frozen B-tree nodes

[BTreeNodeAllocator][btree] supports freezing nodes. When the writer later changes a frozen node, it creates a writable copy. Readers can continue traversing the frozen structure they acquired.

After replacement, the old nodes are placed on hold and eventually reclaimed. Thus copying and generation tracking solve different parts of the operation: copying lets the writer change the tree; tracking keeps the old traversal safe.

[![An existing reader retains an old frozen B-tree root and leaf. A newly published root points to a copied, updated leaf; both roots share an unchanged frozen subtree.](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/frozen-btree.svg)](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/frozen-btree.svg)

*Copying permits mutation; generation protection keeps the old traversal alive.*

The memory index makes the sequence particularly clear:

```cpp
freeze();
assign_generation();
incGeneration();
reclaim_memory();
```

This is the relevant part of `FieldIndex::commit()`, with its remover flush omitted. Attribute types have their own publication hooks, discussed below.

## Attributes

### Write sequencing

Proton's [AttributeWriter][attribute-writer] groups attribute work into write contexts with executor ids. Work for a given context is sent through a sequenced executor, so its mutations are ordered. Other contexts can execute in parallel.

This provides a single writer for each attribute's mutation sequence without requiring a mutex around every value access. Operations spanning the document database are coordinated by its master thread, with field work dispatched to the corresponding executors.

### Commit and publication

[AttributeVector::commit][attribute-commit] invokes the concrete attribute's `onCommit()`, updates the committed document-id limit, and updates statistics. Its serial-number overload also records the last sync token.

`incGeneration()` is a separate method. It calls the attribute's `before_inc_generation` hook, advances the generation handler, and reclaims unused memory. Concrete attribute implementations arrange these operations according to their storage layout.

The distinction matters when reading the code. The base `commit()` method does not itself contain an unconditional call to `incGeneration()`, and advancing a generation is not by itself the operation that writes all new field values.

Readers obtain values through the publication mechanism appropriate to that attribute: for example, atomic value accesses, released references to replacement arrays, or frozen tree roots. The generation guard keeps the referenced storage available.

### AttributeReadGuard

[makeReadGuard][attribute-guard] constructs a guard containing a `GenerationGuard` and, optionally, a shared lock on the enum store:

```cpp
std::unique_ptr<AttributeReadGuard>
AttributeVector::makeReadGuard(bool stableEnumGuard) const {
    return std::make_unique<ReadGuard>(
        this,
        _genHandler.takeGuard(),
        stableEnumGuard ? &_enumLock : nullptr);
}
```

The generation guard protects memory lifetime. When stable enum access is requested, the shared lock additionally protects enum-handle mappings against operations that require the corresponding exclusive lock.

This is one reason to be specific about lock-free reads. Many value and posting-list accesses avoid reader/writer locks, but stable enum access can take one, and acquiring an index collection also takes a lock as we will see below.

### Imported attributes

An imported attribute reads through a reference to another document's attribute. Keeping only the final value alive is insufficient: the target local id or the reference mapping could be reused in the meantime.

[ImportedAttributeVectorReadGuard][imported-guard] therefore holds guards for the target attribute, the target document meta store, and the reference attribute. Together these protect the required objects and mappings. They do not establish a single transactional snapshot across the related documents.

### Flushing

An attribute flush begins with work on the attribute's writer executor. The [Flusher constructor][attribute-flush] commits through the selected serial number and calls `initSave()` to create a saver. Sequencing this setup with writes lets the saver capture the state it needs safely.

The later file-writing task is separate. Depending on the attribute type, the saver may retain generation-protected storage or copied data. For a saver holding a generation guard on a slow disk, the implementation can serialize to a memory target first, release the saver, and then write that target to disk. This shortens the time the original storage remains on hold.

The snapshot directory is first marked invalid and becomes valid only after a successful save. Incomplete snapshots can then be distinguished during recovery.

### Tensor attributes and HNSW

HNSW has a relatively expensive preparation step: searching the graph for neighbors of a new vector. Vespa can run that step outside the writer thread, while leaving graph mutation on the writer sequence.

The [prepared operation][hnsw-prepare] holds a guard for the attribute's vector storage and another for the graph. Completion checks prepared references before applying connections, since the graph may have changed during preparation.

When changing a neighbor list, the writer allocates another array and publishes its reference. The old array is retired through the generation mechanism. Removing a node also removes incoming links and attempts to reconnect its neighbors before retiring its storage.

The [HNSW implementation post](@/posts/2026-09-20-vespa-hnsw-index.md) shows these arrays and operations in more detail. A query can safely follow them while updates occur, but does not see an immutable snapshot of the whole graph.

## Memory index

Unlike an attribute that may store one numeric value per document, a text index must update a dictionary, posting lists, and ranking features. Vespa handles this per field.

A `FieldIndex` contains a word store, a dictionary B-tree, posting-list storage, and a feature store. These use the generation-managed containers described above.

### Inversion and push

[MemoryIndex][memory-index] separates updating into two asynchronous phases:

1. Invert documents into per-field terms and occurrences on the invert executors.
2. Push the accumulated changes into field indexes on the push executors when committing.

The push work is sequenced per field, so two pushes do not concurrently modify the same field index. Different fields can progress independently. A shared completion callback tracks the lifetime of the submitted work.

`insertDocument()` returning therefore does not mean that its terms are already searchable. The API explicitly requires the asynchronous commit step.

### Reader lifetime

[FieldIndex::make_term_blueprint][term-blueprint] acquires a generation guard before obtaining a frozen posting-list iterator:

```cpp
auto guard = takeGenerationGuard();
auto posting_itr = findFrozen(term);
return std::make_unique<MemoryTermBlueprint<interleaved_features>>(
    std::move(guard), std::move(posting_itr), getFeatureStore(),
    field, field_id, term, use_bit_vector);
```

The blueprint owns the guard alongside the iterator. This keeps the storage needed by the query alive while the writer publishes newer trees and retires older nodes.

The guarantee is local to those acquired structures. Independently obtained term or attribute views need not all come from the same instant.

### Flushing the whole index

Freezing B-tree nodes during a commit and freezing a whole `MemoryIndex` serve different purposes.

A normal commit leaves the memory index available for later writes. `MemoryIndex::freeze()` instead stops further updates to that index. The index maintainer can then dump it to disk while new writes go to a replacement memory index.

Queries may still need the frozen memory index until its disk replacement is ready. Its lifetime is handled as part of the searchable collection.

## Disk index

Disk indexes avoid rewriting posting and dictionary data in place. Flush and fusion create new index directories, such as `index.flush.N` and `index.fusion.N`, then make the resulting indexes available to searches.

Some management files, such as the schema, have their own update rules and locks. The immutability relevant to query traversal concerns the index data being read.

### Replacing the searchable collection

[IndexMaintainer][index-maintainer] holds a collection of memory and disk indexes in `_source_list`. Updating this collection is coordinated by the document database's master thread.

`replaceSource()` creates a new collection based on the old one, replaces the relevant source, and calls `swapInNewIndex()`. Its assignment to `_source_list` is protected by locks; the member is an ordinary `shared_ptr`, not an `atomic<shared_ptr>`.

The reader side is short:

```cpp
std::shared_ptr<searchcorespi::IndexSearchable> getSearchable() const override {
    LockGuard lock(_new_search_lock);
    return _source_list;
}
```

After acquiring the pointer, the reader can keep using that collection even if the maintainer installs another one. The lock covers acquisition and replacement, rather than the entire query.

Shared ownership protects the collection's lifetime and membership. It does not make a mutable memory index inside the collection immutable; that index still needs its own reader protection.

The following flush example omits warmup: a disk index replaces a frozen memory index in the new collection, while unchanged sources remain shared.

[![Before a flush replacement, the current collection contains a frozen memory index, an active memory index, and an existing disk index. Afterwards, an old query retains that collection while new queries acquire a collection containing the disk replacement. Both collections share the active memory index and existing disk index.](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/collection-replacement.svg)](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/collection-replacement.svg)

*Replacing the current collection does not invalidate a collection already held by a query.*

### Locks

The lock annotations in `IndexMaintainer` are useful when following flush and fusion:

| Lock | Main responsibility |
| --- | --- |
| `_state_lock` | Read or update a coordinated set of maintainer state |
| `_index_update_lock` | Index-update state, including generation-change tracking |
| `_new_search_lock` | Access to the searchable collection |
| `_fusion_lock` | Fusion specification |
| `_schemaUpdateLock` | Schema rewrites |
| `_remove_lock` | Index-directory removal |

Some variables are protected by more than one lock. The source comment specifies that changing such a variable requires all its associated locks; reading it requires any one of them.

Flush work copies the state it needs, performs disk work separately, and schedules the resulting state change on the master thread. If intervening changes make the prepared state unsuitable, the operation may need to retry with updated state.

### Flush and fusion

A memory-index flush writes a frozen index to disk, loads the result as a disk index, and replaces the corresponding source in the collection. Existing readers can finish against the old source.

Fusion reads several disk indexes and writes a new combined index. After the new index is ready, a collection containing it replaces the old collection. Readers that still hold the old collection continue using its files.

Old directories cannot be deleted merely because they disappeared from the newest collection. [DiskIndexes][disk-indexes] tracks indexes still in use, and cleanup waits until they are no longer needed. [DiskIndexCleaner][disk-cleaner] removes `serial.dat` and syncs the directory before deleting its remaining contents. A directory without that validity marker is recognized as invalid during cleanup after restart.

### Warmup

If configured, [WarmupIndexCollection][warmup] temporarily retains both the previous and next collections. Queries are served from the previous collection while observed query terms cause work against the new disk index to be scheduled on a background executor.

After the configured warmup period, the handover can complete. This reduces the impact of cold index data, though it cannot guarantee that every page needed by a later query has already been read.

Disk reads can still involve synchronization for caches, lazy loading, or statistics. The useful property here is that a query does not need to coordinate posting-list traversal with an in-place rewrite of the same file.

## Threading at the proton level

[ExecutorThreadingService][executors] ties these mechanisms together:

| Executor | Work |
| --- | --- |
| Master | Sequence document-database operations and coordinate state changes |
| Index | Handle index work and dispatch into the memory-index pipeline |
| Summary | Handle document-store work |
| Field writer | Sequence per-field mutations |
| Shared | Run shared tasks, including eligible preparation work |

Flush tasks have their own scheduling and call back into these services where coordination is required.

A useful way to read the implementation is to ask, for each structure: which executor may mutate it, how is a new reference published, and what keeps an old reference alive? The answers differ between a scalar attribute, a posting-list tree, and a collection of disk indexes.

Finally, these mechanisms provide safe concurrent access, with more specific consistency guarantees at each layer. The [Vespa consistency documentation](https://docs.vespa.ai/en/content/consistency.html#read-consistency) explicitly allows searches to observe partial changes while a write is in progress. A generation guard or a retained `shared_ptr` should not be taken as a transaction boundary for the whole query.

[generation-handler]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/vespalib/src/vespa/vespalib/util/generationhandler.cpp
[generation-hold]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/vespalib/src/vespa/vespalib/util/generation_hold.cpp
[datastore]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/vespalib/src/vespa/vespalib/datastore/datastorebase.h
[btree]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/vespalib/src/vespa/vespalib/btree/btreenodeallocator.h
[attribute-writer]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/vespa/searchcore/proton/attribute/attribute_writer.cpp
[attribute-commit]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/attribute/attributevector.cpp#L173
[attribute-guard]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/attribute/attributevector.cpp#L668
[imported-guard]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/attribute/imported_attribute_vector_read_guard.h
[attribute-flush]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/vespa/searchcore/proton/attribute/flushableattribute.cpp#L65
[hnsw-prepare]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_index.cpp#L117
[memory-index]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/memoryindex/memory_index.h
[term-blueprint]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/memoryindex/field_index.cpp#L284
[index-maintainer]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/vespa/searchcorespi/index/indexmaintainer.h
[disk-indexes]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/vespa/searchcorespi/index/disk_indexes.h
[disk-cleaner]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/vespa/searchcorespi/index/diskindexcleaner.cpp
[warmup]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/vespa/searchcorespi/index/warmupindexcollection.cpp
[executors]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/vespa/searchcore/proton/server/executorthreadingservice.h
