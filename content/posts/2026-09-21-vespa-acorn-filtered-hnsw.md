+++
title = "Vespa: Filtered HNSW Search with ACORN-1"
date = "2026-09-21"
description = "How Vespa skips distance calculations for filtered-out vectors while exploring their graph links."

[extra]
toc = true
+++

*Note: This post was refined with AI assistance for clarity and structure.*

Adding a category or access-control filter to an HNSW query creates a navigation problem. A document that cannot appear in the results may still connect two useful parts of the graph. Can the search follow its links without paying to calculate its vector distance?

Vespa's ACORN-1-inspired traversal does this by exploring a neighborhood before scoring its matching nodes. It extends the search described in the [HNSW index post](@/posts/2026-09-20-vespa-hnsw-index.md); the implementation is introduced in [Additions to HNSW in Vespa][vespa-blog]. Source references use revision [39bf68f4][revision]. As in the previous example, each document has one vector, so nodeid and local docid coincide.

## Filtering and graph connectivity

Consider this path through a graph built before the filter was known:

```text
1 (matches) --- 2 (rejected) --- 3 (matches)
```

Skipping node 2 entirely cuts this route from 1 to 3. Reading node 2's links, however, lets the search discover node 3 without calculating node 2's distance. Node 3 can then compete for a place in the search queues.

Ordinary filtered HNSW still calculates distances for rejected nodes and may put them in the exploration queue. Node 2 can therefore help the search reach node 3, but its vector must be loaded and scored along the way. With a restrictive filter, rejected vectors can account for much of that work.

Post-filtering has a different limitation. Searching without the filter and removing rejected hits afterward can return too few results. Increasing the number of retrieved candidates helps, but does not guarantee that enough will pass.

## ACORN-1

The 2024 [ACORN paper][paper] studies search over the subgraph satisfying a query predicate. ACORN-1 uses the idea illustrated above: expand through one-hop and two-hop neighbors, apply the filter, and collect a bounded number of matching nodes for distance calculation.

Vespa's graph construction differs from the paper's. The paper's ACORN-1 omits the usual HNSW neighbor-pruning heuristic. Vespa keeps its existing graph construction and applies the traversal to that graph, without building a denser graph or a separate index per filter.

For the path above, the saving is node 2's vector load and distance calculation. The added work is reading its links and checking the filter on the nodes they lead to. Their relative cost determines whether the traversal is faster.

## Exploring a neighborhood

The main change is in [search_layer_filter_first_helper][filter-first]. The outer search still maintains a nearest-first candidate queue and a bounded collection of the best results. Before calculating distances, it calls [exploreNeighborhood][neighborhood] to collect nodes that pass the filter.

The neighborhood search uses two deques:

- `todo`: nodeids whose links can be followed in the next hop, whether or not they pass the filter.
- `found`: previously unvisited nodes that pass the filter and should have their distances calculated.

Each call to `exploreNeighborhoodByOneHop` processes only the entries already in `todo` when the call begins. Neighbors appended during that call are left for the next hop. Matching nodes enter `found`, and collection stops when its size reaches the layer's maximum link count.

Starting from node 1 in the three-node path, the first hop encounters node 2. It fails the filter, so it contributes links for the next hop but no entry in `found`. The second hop can discover node 3 and add it to `found`, ready for distance calculation. This separates following a route from scoring every vector along it.

The traversal can be simplified to:

```cpp
limit = max_links_for_level(level);
todo = {candidate.nodeid};
found = {};

explore_one_hop(todo, found, limit);
explore_one_hop(todo, found, limit);

if (todo.size() < exploration * limit * limit) {
    explore_one_hop(todo, found, limit);
}
```

After these calls, the caller calculates distances for `found` and updates the search queues. Each neighborhood has its own collection limit, independent of the query's target hit count and overall search width.

Both matching and rejected nodes can supply links for another hop. The visited tracker prevents repeated collection into `found`, while `todo` may contain repeated entries. Its size counts queued entries, which matters for the third-hop condition.

### A third hop

Two hops may still be insufficient when the filter leaves large gaps between matching nodes. A third hop can reach farther, but its cost grows quickly. With `max-links-per-node = 16`, level 0 permits 32 links per node; unrestricted three-hop expansion could encounter up to `32 * 32 * 32 = 32768` link paths before accounting for overlap.

Vespa only attempts the third hop when the remaining `todo` is smaller than `exploration * limit * limit`. The same `found` limit still applies. Increasing `exploration` allows the extra hop in more neighborhoods, potentially improving recall at the cost of more traversal.

## Selecting the search path

[NearestNeighborBlueprint][blueprint] compares the filter's hit ratio with `filter-first-threshold`. Here, hit ratio means the fraction of documents that pass: a smaller value means a more restrictive filter. When filtered approximate search is selected and the ratio is below the threshold, it requests the filter-first traversal.

[top_k_candidates][top-k] uses the filter-first traversal only at level 0. Upper-level navigation and entry-point scoring still calculate distances as before. A rejected entry point leaves the result collection but can seed exploration; subsequent neighborhood expansions skip scoring the rejected neighbors they discover.

When sufficiently few documents pass, the planner can choose exact search instead: a restrictive filter directly reduces the number of vectors it must score. Filter-first traversal is used within the approximate-search path, after that choice.

## Parameters and trade-offs

The [query API documentation][query-api], checked on September 21, 2026, lists these defaults:

| Query parameter | Default | Effect |
| --- | --- | --- |
| `ranking.matching.filterFirstThreshold` | `0.2` | Select filter-first traversal below this hit ratio, when using filtered graph search |
| `ranking.matching.filterFirstExploration` | `0.01` | Control when to attempt three-hop expansion |

Their rank-profile equivalents are `filter-first-threshold` and `filter-first-exploration`. The 2025 blog post listed defaults of `0.0` and `0.30`; those values have since changed.

The same hit ratio can produce different search costs. Matching nodes clustered near the query leave little wasted distance work for ACORN-1 to remove. Scattered matches can make extended neighborhoods useful. Matches far from the query's natural path may still be missed by bounded expansion.

Measure latency at the same recall, and record both the filter hit ratio and its correlation with the query. Vespa tracks visited nodes and computed distances separately; looking at both helps explain where the time goes. A lower distance count alone does not establish a faster search: reaching node 3 without scoring node 2 still required reading node 2's links.

[vespa-blog]: https://blog.vespa.ai/additions-to-hnsw/
[paper]: https://arxiv.org/abs/2403.04871
[revision]: https://github.com/vespa-engine/vespa/tree/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2
[filter-first]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_index.cpp#L453
[neighborhood]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_index.cpp#L543
[blueprint]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/queryeval/nearest_neighbor_blueprint.cpp#L196
[top-k]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchlib/src/vespa/searchlib/tensor/hnsw_index.cpp#L1077
[query-api]: https://docs.vespa.ai/en/reference/api/query.html#ranking.matching.filterFirstThreshold
