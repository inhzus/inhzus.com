+++
title = "Vespa: Thread Safety in the Storage Engine"
date = "2026-09-20"
description = "How proton coordinates writes, protects readers, and replaces memory and disk indexes."

[extra]
toc = true
+++

*Note: This post was refined with AI assistance for clarity and structure.*

A query has found a posting list and is walking through it. Meanwhile, a writer replaces some of its nodes. The query already holds references into the old tree, so publishing a new root is only half the job: the old nodes must remain alive until that query finishes.

The earlier posts described the [index structures](@/posts/2024-12-04-vespa-index-impl.md) and [matching code](@/posts/2024-12-05-vespa-match-impl.md). The question here is how proton, Vespa's searchnode, lets a reader finish while those structures change underneath it. Source links refer to revision [39bf68f4](https://github.com/vespa-engine/vespa/tree/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2).

## Publishing changes and retaining old data {#overview}

Vespa sequences writes to many mutable structures so that each has one writer at a time. Readers continue using published references while that writer prepares changes. There are two separate obligations: publish data so readers can safely access it, and delay reclamation until nobody needs the old storage. Atomic loads and stores, frozen B-tree nodes, and locks handle the first; generation guards handle the second.

A guard keeps old memory alive. It does not give the query a fixed snapshot across fields or related documents: the query can observe changes published at different times. The [consistency documentation](https://docs.vespa.ai/en/content/consistency.html#read-consistency) explicitly allows partial changes to be visible while a write is in progress.

The same lifetime question returns during a disk-index replacement, where a query holds an entire collection of indexes. Start with a smaller object: one array that a reader is still using.

## Following a reader through an update {#generationhandler}

Suppose a reader holds generation 10 and has a reference to array A. The writer allocates array B, publishes its reference, and retires A at generation 10. It can then advance to generation 11. Array A stays on hold until the reader releases its guard and the reclamation boundary advances past 10.

[![Timeline showing a reader holding generation 10 while the writer publishes array B and retires array A. A becomes reclaimable only after the guard is released and the oldest-used generation advances past 10.](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/generation-lifetime.svg)](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/generation-lifetime.svg)

A slow reader delays this reclamation. During heavy updates, the writer may retire many more allocations before that reader finishes, increasing on-hold memory.

[GenerationHandler][generation-handler] supplies the information needed to decide when A can be freed. It has one writer and tracks the generations held by multiple readers.

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

The handler retains the hold entries for reuse, so a reader attempting acquisition can safely encounter an entry that has left the active list.

### Advancing the generation

The writer calls `incGeneration()` after the surrounding structure has performed its publication work.

If the newest hold has no readers, the handler can reuse it with the next generation number. Otherwise, it allocates or reuses another hold and publishes it through `_last` with release ordering.

`update_oldest_used_generation()` walks from the oldest entry until it reaches one still in use, or the current entry. Stores use this oldest-used generation to decide which retired allocations can be reclaimed.

## DataStore and B-trees

The handler tracks readers; the data store tracks the allocations they may still need. [DataStoreBase][datastore] records retired elements and buffers on hold lists and assigns them a generation. In our example, A belongs to generation 10, so the store waits for the oldest reader generation to advance past 10 before reclaiming it.

Compaction follows the same rule. Moving live data to another buffer does not immediately make the old buffer reusable: a reader may already have resolved an old reference into it.

### Frozen B-tree nodes

[BTreeNodeAllocator][btree] applies this to tree nodes. It supports freezing nodes; when the writer later changes a frozen node, it creates a writable copy. Unchanged subtrees can remain shared.

For the query walking our posting list, the old root still leads to usable nodes. Replaced nodes go on hold, just as array A did, until the reader no longer needs them.

[![An existing reader retains an old frozen B-tree root and leaf. A newly published root points to a copied, updated leaf; both roots share an unchanged frozen subtree.](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/frozen-btree.svg)](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/frozen-btree.svg)

The memory index puts these steps together in `FieldIndex::commit()`:

```cpp
freeze();
assign_generation();
incGeneration();
reclaim_memory();
```

This excerpt omits the remover flush. Attributes arrange publication differently, so their commit path needs a separate look.

## Threading at the proton level

[ExecutorThreadingService][executors] assigns work within each document database. The master thread coordinates operations, while field writers can update different fields concurrently:

| Executor | Work |
| --- | --- |
| Master | Sequence document-database operations and coordinate state changes |
| Index | Handle index work and dispatch into the memory-index pipeline |
| Summary | Handle document-store work |
| Field writer | Sequence per-field mutations |
| Shared | Run shared tasks, including eligible preparation work |

Flush tasks have separate scheduling and call back into these services to coordinate with writes.

## Attributes

### Write sequencing

Proton's [AttributeWriter][attribute-writer] groups attribute work into write contexts with executor ids. Work for a given context is sent through a sequenced executor, so its mutations are ordered. Other contexts can execute in parallel.

Each attribute therefore has a single mutation sequence without needing a mutex around every value access.

### Commit and publication

[AttributeVector::commit][attribute-commit] invokes the concrete attribute's `onCommit()`, updates the committed document-id limit, and updates statistics. Its serial-number overload also records the last sync token.

Generation advancement has its own method, `incGeneration()`: it calls `before_inc_generation`, advances the handler, and reclaims unused memory. The base `commit()` has no unconditional call to it. Each concrete attribute arranges publication and generation advancement to suit its storage layout.

The reader's access follows that storage layout: an atomic value load, an acquired reference to a replacement array, or a frozen tree root. This is why the base `commit()` alone cannot explain how every attribute becomes readable.

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

Stable enum access adds a requirement beyond retaining storage: enum-handle mappings must remain stable too. The shared lock protects those mappings against operations requiring the corresponding exclusive lock. Many ordinary value and posting-list reads avoid this reader/writer lock.

### Imported attributes

An imported attribute reads through a reference to another document's attribute. Keeping only the final value alive is insufficient: the target local id or the reference mapping could be reused in the meantime.

[ImportedAttributeVectorReadGuard][imported-guard] holds three guards: one for the target attribute, one for the target document meta store, and one for the reference attribute. These retain the objects and mappings needed to follow the reference.

### Flushing

An attribute flush begins with work on the attribute's writer executor. The [Flusher constructor][attribute-flush] commits through the selected serial number and calls `initSave()` to create a saver. Sequencing this setup with writes lets the saver capture the state it needs safely.

The saver then acts as another reader during the separate file-writing task. Depending on the attribute type, it may retain generation-protected storage or copied data. A saver holding a guard can delay reclamation just like the slow query holding array A. For slow-disk saves, the implementation can shorten that delay by serializing to a memory target, releasing the saver, and then writing the target to disk.

The snapshot directory is first marked invalid and becomes valid only after a successful save. Incomplete snapshots can then be distinguished during recovery.

### Tensor attributes and HNSW

HNSW has a relatively expensive preparation step: searching the graph for neighbors of a new vector. Vespa can run that step outside the writer thread, while leaving graph mutation on the writer sequence.

The [prepared operation][hnsw-prepare] holds a guard for the attribute's vector storage and another for the graph. Completion checks prepared references before applying connections, since the graph may have changed during preparation.

Neighbor-list replacement follows the array A/B example directly. Node removal also removes incoming links and attempts to reconnect neighbors before retiring storage. The [HNSW implementation post](@/posts/2026-09-20-vespa-hnsw-index.md) follows these updates through the graph's arrays.

## Memory index

Unlike an attribute that may store one numeric value per document, a text index must update a dictionary, posting lists, and ranking features. Vespa handles this per field.

A `FieldIndex` contains a word store, a dictionary B-tree, posting-list storage, and a feature store. These use the generation-managed containers described above.

### Inversion and push

[MemoryIndex][memory-index] separates updating into two asynchronous phases:

1. Invert documents into per-field terms and occurrences on the invert executors.
2. Push the accumulated changes into field indexes on the push executors when committing.

The push work is sequenced per field, so two pushes do not concurrently modify the same field index. Different fields can progress independently. A shared completion callback tracks the lifetime of the submitted work.

Making the terms searchable requires the asynchronous commit step; returning from `insertDocument()` alone is insufficient.

### Reader lifetime

[FieldIndex::make_term_blueprint][term-blueprint] acquires a generation guard before obtaining a frozen posting-list iterator:

```cpp
auto guard = takeGenerationGuard();
auto posting_itr = findFrozen(term);
return std::make_unique<MemoryTermBlueprint<interleaved_features>>(
    std::move(guard), std::move(posting_itr), getFeatureStore(),
    field, field_id, term, use_bit_vector);
```

This is where the posting-list reader from the opening gets its protection. The blueprint owns both the frozen iterator and the guard, keeping the old tree's storage available as the writer publishes new trees.

### Flushing the whole index

A normal commit freezes B-tree nodes but leaves the memory index open for later writes. To flush the whole index, `MemoryIndex::freeze()` stops further updates. New writes go to a replacement memory index while the maintainer dumps the frozen one to disk.

Freezing stops writes, but it does not end queries already using that index. To follow its lifetime through the flush, we need to look at the collection from which queries acquire their indexes.

## Disk index

Flush and fusion create new index directories, such as `index.flush.N` and `index.fusion.N`. Their posting and dictionary data are written during construction and stay stable during query traversal. The handover happens by replacing the collection of indexes available to searches.

Management files such as the schema have separate update rules and locks.

### Replacing the searchable collection

[IndexMaintainer][index-maintainer] holds a collection of memory and disk indexes in `_source_list`. Updating this collection is coordinated by the document database's master thread.

`replaceSource()` copies the collection, replaces the relevant source, and calls `swapInNewIndex()`. Locks protect assignment to the ordinary `shared_ptr` member `_source_list`.

The reader side is short:

```cpp
std::shared_ptr<searchcorespi::IndexSearchable> getSearchable() const override {
    LockGuard lock(_new_search_lock);
    return _source_list;
}
```

After acquiring the pointer, the reader can keep using that collection even if the maintainer installs another one. The lock covers acquisition and replacement, rather than the entire query.

The retained collection can still contain an active memory index. Queries use that index's generation guards to protect their reads while it changes.

In this flush, a disk index replaces the frozen memory index while unchanged sources remain shared. The old query keeps its original collection. Warmup, covered below, can delay the handover to the new one.

[![Before a flush replacement, the current collection contains a frozen memory index, an active memory index, and an existing disk index. Afterwards, an old query retains that collection while new queries acquire a collection containing the disk replacement. Both collections share the active memory index and existing disk index.](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/collection-replacement.svg)](https://image.inhzus.io/posts/vespa-thread-safety-in-the-storage-engine/collection-replacement.svg)

Flush work copies the state it needs, performs disk work separately, and schedules the resulting state change on the master thread. If intervening changes make the prepared state unsuitable, the operation may need to retry with updated state.

### Flush and fusion

The flush shown above loads the newly written disk index before replacing the frozen memory source. Fusion uses the same handover to replace several disk indexes with one combined index. In both cases, the old collection remains usable by queries that already acquired it.

Old directories cannot be deleted merely because they disappeared from the newest collection. [DiskIndexes][disk-indexes] tracks indexes still in use, and cleanup waits until they are no longer needed. [DiskIndexCleaner][disk-cleaner] removes `serial.dat` and syncs the directory before deleting its remaining contents. A directory without that validity marker is recognized as invalid during cleanup after restart.

### Warmup

If configured, [WarmupIndexCollection][warmup] temporarily retains both the previous and next collections. Queries are served from the previous collection while observed query terms cause work against the new disk index to be scheduled on a background executor.

After the configured period, the handover can complete. Only terms observed during warmup have caused background work, so later queries can still encounter cold pages. Caches, lazy loading, and statistics may also require synchronization during disk reads, even though the posting data itself stays unchanged.

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
