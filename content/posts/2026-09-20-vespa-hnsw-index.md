+++
title = "Vespa: HNSW Index Implementation"
date = "2026-09-20"
description = "How Vespa stores, searches, and updates its HNSW graph."

[extra]
toc = true
+++

*Note: This post was refined with AI assistance for clarity and structure.*

The previous [index implementation post](@/posts/2024-12-04-vespa-index-impl.md) left tensors for later. Vector search adds a graph alongside the attribute values: a query follows its links to find promising vectors, and a writer changes those links as documents arrive or disappear.

Five documents are enough to trace the full path from graph storage to search queues. The same arrays also show why changing a neighbor list requires more than overwriting its contents. Source links point to Vespa revision [39bf68f4](https://github.com/vespa-engine/vespa/tree/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2).

## Overview

Vespa separates the graph from the vectors. `HnswGraph` stores nodes and their connections, while `TensorAttribute` provides the vector values used to calculate distances. `HnswIndex` implements insertion, search, and removal on top of these two parts.

There are two variants:

- `SINGLE`: one vector per document. The graph node id is the document's local id, so no extra mapping is needed.
- `MULTI`: several vector subspaces per document. Each subspace gets a graph node, and the implementation maintains the mapping between document ids and node ids.

For `MULTI`, the mapping records both docid and subspace so distance calculation can find the vector in the attribute's value store. The result collection accounts for multiple graph nodes belonging to the same document.

The example uses `SINGLE`. Docid below means the local document id on one searchnode.

## A five-document graph {#example}

Take five documents with one two-dimensional vector each and `M = max-links-per-node = 2`. Level 0 permits up to four links per node; higher levels permit two. This graph is illustrative, rather than the output of a particular insertion sequence.

| docid | Vector | Highest level |
| --- | --- | --- |
| 1 | (0, 0) | 1 |
| 2 | (1, 0) | 0 |
| 3 | (0, 1) | 0 |
| 4 | (1, 1) | 0 |
| 5 | (2, 2) | 2 |

In normal insertion, the highest level is drawn by [InvLogLevelGenerator][level-generator]. The probability of level `k` is `(1/M)^k * (1 - 1/M)`. Here we simply choose the levels shown above.

Node 5 is the entry point at level 2. The amber path shows the upper-level descent for query `(0.8, 0.9)`, which we will follow after unpacking the storage layout.

[![Three graph levels: level 2 contains entry node 5; level 1 connects nodes 1 and 5; level 0 has bidirectional links 1–2, 1–3, 1–5, 2–4, 3–4, and 4–5. The highlighted search descends from node 5 at level 2, moves from 5 to 1 at level 1, and descends to node 1 at level 0.](https://image.inhzus.io/posts/vespa-hnsw-index-implementation/graph-levels.svg)](https://image.inhzus.io/posts/vespa-hnsw-index-implementation/graph-levels.svg)

The completed graph has reverse edges and stays within the link limits. During an update, a concurrent reader may see one direction change before the other.

## Graph storage

To follow a link out of node 5, the query first needs its level array, then that level's neighbor array. [HnswGraph][graph] stores these separately:

```cpp
NodeVector nodes;                         // nodeid -> levels ref
LevelArrayStore levels_store;             // array of link refs per node
LinkArrayStore links_store;               // array of neighbor nodeids
std::atomic<uint64_t> entry_nodeid_and_level;
```

The graph also holds counters and a generation handler for reclaiming replaced storage.

### Nodes

`nodes` is an `RcuVector`, indexed by nodeid. For `SINGLE`, each element is an `HnswSimpleNode`, which holds only an `AtomicEntryRef` pointing into `levels_store`. Node 0 is reserved.

[AtomicEntryRef][atomic-ref] wraps a `std::atomic<uint32_t>` encoding a buffer id and offset. Resolving that reference locates the array within the store.

The `MULTI` variant uses `HnswNode`, which additionally stores the document id and subspace id described above.

### Levels and links

`levels_store` is an `ArrayStore<AtomicEntryRef>`. Node 5 occupies levels 0 through 2, so its level array has three entries, each referring to the neighbor array for one level.

`links_store` is an `ArrayStore<uint32_t>`. Its arrays contain only neighbor nodeids.

The two stores use different `EntryRef` layouts. The level store reserves 10 bits for the buffer id and 22 for the offset; the link store uses 12 and 20 respectively. This gives them 1024 and 4096 possible buffers.

The [factory][factory] sets the link limits used in our example: `2 * M` at level 0 and `M` above it. Insertion may temporarily exceed a limit before pruning the neighbor list.

### In memory

For node 5, the first lookup reaches its three-entry level array. The second reaches a neighbor array: `[1, 4]` at level 0, `[1]` at level 1, or `[]` at level 2.

Those arrays give the query nodeids to visit. Calculating a distance requires a separate lookup of the corresponding vector in `TensorAttribute`.

[![The node vector contains level references, with node 0 invalid. Node 5's EntryRef resolves to a three-entry level array. Its level 0, 1, and 2 references resolve to neighbor arrays [1, 4], [1], and [] respectively. TensorAttribute separately holds the vector (2, 2) for docid 5.](https://image.inhzus.io/posts/vespa-hnsw-index-implementation/graph-storage.svg)](https://image.inhzus.io/posts/vespa-hnsw-index-implementation/graph-storage.svg)

Across the example graph, 14 neighbor ids take 56 bytes. The full memory footprint also includes the node vector, level references, spare buffer capacity, allocator metadata, and retired storage awaiting reclamation.

<span id="query-planning-and-persistence"></span>

[HnswIndexSaver][saver] and its loader save and restore the graph as part of attribute persistence; the tensor values have their own storage.

### Entry point

Our query starts at node 5 on level 2. The entry point's nodeid and level are packed into one 64-bit atomic value:

```text
63                         32 31                          0
+----------------------------+----------------------------+
|            level           |           nodeid           |
+----------------------------+----------------------------+
```

`get_entry_node()` reads this value, acquires the node's levels reference, then checks the entry point again. If the observations are incompatible, it retries. This handles an entry point being changed or removed while another thread starts a search.

## Following the query {#searching}

For query `(0.8, 0.9)` and top-k 2, the squared Euclidean distances to our five documents are:

| docid | Squared distance |
| --- | --- |
| 1 | 1.45 |
| 2 | 0.85 |
| 3 | 0.65 |
| 4 | 0.05 |
| 5 | 2.65 |

Squared distances preserve the ordering without square roots. The upper-level path in the graph diagram starts at node 5, moves to node 1 at level 1, and enters level 0 there.

At level 0, [search_layer_helper][search-layer] keeps a nearest-first candidate queue and a bounded result collection. The candidate queue supplies the next node to expand; the result collection retains the best nodes found so far.

For this example, assume the result collection holds two nodes, with no filter, no extra exploration slack, and no deadline reached:

[![Level-0 search trace, showing nodeid and squared distance. Seed 1 puts 1 at 1.45 in both collections. Expanding 1 gives candidates and results 3 at 0.65 and 2 at 0.85. Expanding 3 gives candidates 4 at 0.05 and 2 at 0.85, but retains results 4 and 3. Expanding 4 leaves 2 queued. The search stops before expanding 2 because 0.85 exceeds the worst retained distance, 0.65; the results remain 4 and 3.](https://image.inhzus.io/posts/vespa-hnsw-index-implementation/search-queues.svg)](https://image.inhzus.io/posts/vespa-hnsw-index-implementation/search-queues.svg)

After expanding node 3, the results contain 4 and 3, but node 2 is still queued for exploration. Expanding node 4 does not improve the results, so the search stops before expanding 2: its distance, 0.85, exceeds the worst retained distance, 0.65. Documents 4 and 3 happen to be the exact nearest neighbors here. On a larger graph, the search width limits exploration, and the result remains approximate.

This trace assumes the planner chose an unfiltered graph search. [NearestNeighborBlueprint][blueprint] also considers exact search and filtered traversal based on the query, filter estimate, and configuration. The [ACORN post](@/posts/2026-09-21-vespa-acorn-filtered-hnsw.md) follows what changes when some graph nodes fail a filter.

## Updating the graph

### Replacing a neighbor array

Node 5's level-0 reference currently resolves to `[1, 4]`. Suppose a query has acquired that array when the writer needs to publish a different neighbor list. Overwriting the two entries would change memory the query is still reading.

[set_link_array][set-links] allocates a replacement and changes the reference in node 5's level array. Simplified:

```cpp
auto new_ref = links_store.add(new_links);
auto old_ref = levels[level].load_relaxed();
levels[level].store_release(new_ref);
links_store.remove(old_ref);
```

The reader that already acquired `[1, 4]` can finish with that array. A reader acquiring the newly published reference gets `new_links`. Here `remove` retires the old storage through the generation mechanism; its memory remains available while the appropriate generation guards are held.

A query can encounter links published at different times as it moves between nodes. The guard protects the storage it reads, without freezing the whole graph. The [thread-safety post](@/posts/2026-09-20-vespa-storage-engine-thread-safety.md) explains publication and reclamation in more detail.

### Insertion

Inserting another document starts with the same kind of descent and neighborhood search as our query, after choosing a level for the new node. The search supplies candidate neighbors at the relevant levels. Neighbor selection can reject a candidate when it is closer to an already-selected neighbor than to the new node, avoiding links concentrated in nearly the same part of the graph.

The writer then creates the node, connects it to the selected neighbors, adds reverse links, and prunes lists that exceed their limits. If the new node reaches a higher level than the current entry point, the entry point is updated. See [the insertion implementation][insertion].

The expensive search for neighbors can run before the write phase. `prepare_add_document` holds guards for both the tensor values and the graph; `complete_add_document` applies the prepared connections on the writer thread. Prepared references are checked again because nodes may have changed between the two phases. Small graphs use the direct write-thread path so that the first nodes are connected as they are added.

### Removal

Removing node 5 would affect more than its own arrays: nodes 1 and 4 link to it at level 0, node 1 links to it at level 1, and it is the entry point. [remove_node][remove-node] visits each level, removes reverse links, and calls `mutual_reconnect` to add connections among the former neighbors where possible. It also chooses a new entry point when necessary.

`mutual_reconnect` considers pairs among those neighbors in distance order and respects link limits. It usually caps the number of new connections per node, with an exception for isolated nodes.

The writer then invalidates the node's levels reference and retires its arrays. A concurrent query may still arrive at node 5 through a link it read earlier. It checks for invalid references before continuing, while its guard keeps already-acquired storage alive until it finishes.

[graph]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_graph.h
[atomic-ref]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/vespalib/src/vespa/vespalib/datastore/atomic_entry_ref.h
[factory]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/default_nearest_neighbor_index_factory.cpp
[level-generator]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/inv_log_level_generator.h
[search-layer]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_index.cpp#L364
[set-links]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_graph.cpp#L78
[insertion]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_index.cpp#L704
[remove-node]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_index.cpp#L866
[blueprint]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/queryeval/nearest_neighbor_blueprint.cpp
[saver]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_index_saver.cpp
