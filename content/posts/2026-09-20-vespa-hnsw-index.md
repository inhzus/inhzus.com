+++
title = "Vespa: HNSW Index Implementation"
date = "2026-09-20"
description = "How Vespa stores, searches, and updates its HNSW graph."

[extra]
toc = true
+++

The previous [index implementation post](@/posts/2024-12-04-vespa-index-impl.md) covered attributes and inverted indexes, but left out tensors. This post continues with the data structures behind Vespa's HNSW index, then follows a small example through a search.

The source references below point to Vespa revision [39bf68f4](https://github.com/vespa-engine/vespa/tree/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2). The example graph is illustrative; it is not the output of a particular insertion sequence.

## Overview

Vespa separates the graph from the vectors. `HnswGraph` stores nodes and their connections, while `TensorAttribute` provides the vector values used to calculate distances. `HnswIndex` implements insertion, search, and removal on top of these two parts.

There are two variants:

- `SINGLE`: one vector per document. The graph node id is the document's local id, so no extra mapping is needed.
- `MULTI`: several vector subspaces per document. Each subspace gets a graph node, and the implementation maintains the mapping between document ids and node ids.

We start with `SINGLE`. Unless stated otherwise, docid below means the local document id on one searchnode.

## Graph storage

The main storage members of [HnswGraph][graph] can be simplified to:

```cpp
NodeVector nodes;                         // nodeid -> levels ref
LevelArrayStore levels_store;             // array of link refs per node
LinkArrayStore links_store;               // array of neighbor nodeids
std::atomic<uint64_t> entry_nodeid_and_level;
```

There are also counters and a generation handler. We will come back to memory reclamation after looking at the layout.

### Nodes

`nodes` is an `RcuVector`, indexed by nodeid. For `SINGLE`, each element is an `HnswSimpleNode`, which holds only an `AtomicEntryRef` pointing into `levels_store`. Node 0 is reserved.

`AtomicEntryRef` wraps a `std::atomic<uint32_t>`. Its value encodes a buffer id and an offset, like the `EntryRef` used elsewhere in Vespa. It is not a native pointer. See [the definition][atomic-ref].

For `MULTI`, `HnswNode` additionally stores a document id and a subspace id. Distance calculation needs these to find the correct vector within the document.

### Levels and links

`levels_store` is an `ArrayStore<AtomicEntryRef>`. A node present at levels 0 through 2 has an array of three entries. Each entry points to that node's neighbor array at the corresponding level.

`links_store` is an `ArrayStore<uint32_t>`. Its arrays contain only neighbor nodeids. Following a graph edge therefore involves these lookups:

```text
nodes[nodeid]
    -> levels_store: [level 0 ref, level 1 ref, ...]
    -> links_store:  [neighbor id, neighbor id, ...]
```

The two stores use different `EntryRef` layouts. The level store reserves 10 bits for the buffer id and 22 for the offset; the link store uses 12 and 20 respectively. This gives them 1024 and 4096 possible buffers.

The maximum number of links also depends on the level. With `M = max-links-per-node`, the [factory][factory] configures up to `2 * M` links at level 0 and `M` at higher levels. Insertion may temporarily exceed this before pruning the neighbor lists.

### Entry point

The entry point is the node from which a search starts. Its nodeid and level are packed into one 64-bit atomic value:

```text
63                         32 31                          0
+----------------------------+----------------------------+
|            level           |           nodeid           |
+----------------------------+----------------------------+
```

`get_entry_node()` reads this value, acquires the node's levels reference, then checks the entry point again. If the observations are incompatible, it retries. This handles an entry point being changed or removed while another thread starts a search.

## Example

Assume five documents, each with one two-dimensional vector, and `M = 2`. Level 0 can then have up to four links per node, and higher levels up to two.

| docid | Vector | Highest level |
| --- | --- | --- |
| 1 | (0, 0) | 1 |
| 2 | (1, 0) | 0 |
| 3 | (0, 1) | 0 |
| 4 | (1, 1) | 0 |
| 5 | (2, 2) | 2 |

In normal insertion, the highest level is drawn by [InvLogLevelGenerator][level-generator]. The probability of level `k` is `(1/M)^k * (1 - 1/M)`. Here we simply choose the levels shown above.

Suppose the graph has these connections:

```text
Level 2:  5
Level 1:  1 <-> 5

Level 0:
    1 -> [2, 3, 5]
    2 -> [1, 4]
    3 -> [1, 4]
    4 -> [2, 3, 5]
    5 -> [1, 4]

Entry point: node 5, level 2
```

Every edge has a reverse edge, and the link counts are within the limits. These are properties of this example's completed graph; a concurrent reader need not observe both directions being updated at the same instant.

### In memory

The node vector contains references to five level arrays. The names below are symbolic references, not actual buffer offsets:

```text
nodes[0] -> invalid
nodes[1] -> levels_1
nodes[2] -> levels_2
nodes[3] -> levels_3
nodes[4] -> levels_4
nodes[5] -> levels_5

levels_1 = [links_1_0, links_1_1]
levels_2 = [links_2_0]
levels_3 = [links_3_0]
levels_4 = [links_4_0]
levels_5 = [links_5_0, links_5_1, links_5_2]
```

Each `links_<node>_<level>` resolves to an array in `links_store`:

```text
links_1_0 = [2, 3, 5]    links_1_1 = [5]
links_2_0 = [1, 4]
links_3_0 = [1, 4]
links_4_0 = [2, 3, 5]
links_5_0 = [1, 4]       links_5_1 = [1]    links_5_2 = []
```

Notice that none of these arrays contains `(0, 0)` or any other vector value. Nor do they contain distances. Those are obtained from the tensor attribute and calculated during the operation.

The 14 neighbor ids occupy 56 bytes of payload. That is only the link data: the node vector, level references, buffer capacity, allocator metadata, and memory awaiting reclamation all add to the total. Counting the links alone would substantially understate the index's memory use.

### Searching

Let the query vector be `(0.8, 0.9)`, with top-k equal to 2. The squared Euclidean distances are:

| docid | Squared distance |
| --- | --- |
| 1 | 1.45 |
| 2 | 0.85 |
| 3 | 0.65 |
| 4 | 0.05 |
| 5 | 2.65 |

Using squared distances preserves the ordering and avoids taking square roots.

The search starts at node 5, level 2. There are no neighbors at that level, so it descends to level 1. Node 1 is closer to the query than node 5, and becomes the starting point for level 0.

At level 0, [search_layer_helper][search-layer] maintains a nearest-first queue of candidates to explore and a bounded collection of the best results found so far. These serve different purposes: a node can remain in the exploration queue after being displaced from the result set.

For this example, assume the result collection holds two nodes, with no filter, no extra exploration slack, and no deadline reached:

1. Expand node 1. Its neighbors are 2, 3, and 5. Nodes 2 and 3 replace node 1 in the result set; node 5 is too far away.
2. Expand node 3, the closest queued candidate. Node 1 has already been visited. Node 4 is new, and replaces node 2 in the result set.
3. The next queued candidate is node 4. Its neighbors add no better result.
4. Node 2 remains queued, but its distance, 0.85, exceeds the worst retained distance, 0.65. The search stops.

The returned documents are 4 and 3. They happen to be the exact nearest neighbors in this small graph. In a larger graph, the search width controls how much exploration is done, and HNSW remains an approximate search.

## Updating the graph

### Replacing a neighbor array

Search threads may be reading a neighbor array when the writer needs to add or remove an edge. Mutating that array directly would invalidate the read.

[set_link_array][set-links] instead allocates a replacement array, publishes its reference, and retires the old one. Simplified:

```cpp
auto new_ref = links_store.add(new_links);
auto old_ref = levels[level].load_relaxed();
levels[level].store_release(new_ref);
links_store.remove(old_ref);
```

Here `remove` retires the storage through the generation mechanism. A reader holding the appropriate generation guard can finish using the old array before its memory is reused.

This makes individual accesses safe, but does not freeze the entire graph for the duration of a query. A search can encounter links published at different times. The distinction is discussed further in [the thread-safety post](@/posts/2026-09-20-vespa-storage-engine-thread-safety.md).

### Insertion

Insertion first chooses a level, descends through the graph, and searches for candidate neighbors at the relevant levels. Neighbor selection can use a heuristic that rejects a candidate when it is closer to an already-selected neighbor than to the new node. This avoids spending all the links on nearly the same part of the graph.

The writer then creates the node, connects it to the selected neighbors, adds reverse links, and prunes lists that exceed their limits. If the new node reaches a higher level than the current entry point, the entry point is updated. See [the insertion implementation][insertion].

The expensive search for neighbors can run before the write phase. `prepare_add_document` holds guards for both the tensor values and the graph; `complete_add_document` applies the prepared connections on the writer thread. Prepared references are checked again because nodes may have changed between the two phases. Small graphs use the direct write-thread path so that the first nodes are connected as they are added.

### Removal

[remove_node][remove-node] visits each level of the removed node, removes the reverse links from its neighbors, and calls `mutual_reconnect` to add connections among those neighbors where possible. If necessary, it also chooses a new entry point.

`mutual_reconnect` considers neighbor pairs in distance order and respects link limits. It generally limits the new connections per node, with an exception for isolated nodes. It does not reconstruct the whole graph.

Finally, the node's levels reference is invalidated and its arrays are retired. Search code still checks for invalid references because removal may overlap with a search. Those checks are needed in addition to unlinking the node.

## Query planning and persistence

An HNSW index is one of the execution options available to [NearestNeighborBlueprint][blueprint]. Depending on the query, estimated filter selectivity, and configuration, the planner may use approximate search, filtered graph traversal, or exact distance calculations. Building the graph does not mean every nearest-neighbor query must use it.

For `MULTI`, several graph nodes may belong to the same document. The mapping records both docid and subspace, and the result collection accounts for duplicate document ids. The vectors still come from the attribute's value store.

[HnswIndexSaver][saver] and the corresponding loader save and restore the graph as part of attribute persistence. Graph topology and tensor values have separate storage responsibilities. The index also exposes memory usage, level/link histograms, and reachability information through its explorer, which is useful when inspecting a graph with frequent updates.

For further reading, start with `HnswGraph` for the layout, then `HnswIndex::top_k_candidates` for search and `HnswIndex::remove_node` for deletion. These are easier to follow once the two levels of array references are clear.

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
