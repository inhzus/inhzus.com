+++
title = "Vespa: Distributors and Buckets"
date = "2026-09-20"
description = "How Vespa routes document operations, maintains replicas, and redistributes buckets."

[extra]
toc = true
+++

*Note: This post was refined with AI assistance for clarity and structure.*

Where does a document go after a client sends it to Vespa? The request first needs an owner distributor, then storage nodes for its replicas. Both decisions start with the document's bucket, but they can lead to different nodes.

In the [Vespa comparison](@/posts/2024-10-12-vespa-vs-elasticsearch.md), I briefly mentioned bucket-based distribution. Following one Put makes the division of work clearer: the distributor tracks the copies, while the content nodes hold the documents. That distinction also explains what must be recovered when either one fails. Source references use Vespa revision [39bf68f4](https://github.com/vespa-engine/vespa/tree/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2).

## Feeding a document

The example uses an indexed content cluster with six content nodes and corresponding distributors, a flat distribution, 16 distribution bits, and redundancy 2. The document is `id:news:article::kitten-finds-home-2024`. Assume D3 owns its bucket and nodes 0 and 1 hold the replicas; these placements and the bucket values below are illustrative.

```text
Client
  -> Container: document processing and routing
  -> Owner distributor: resolve bucket and replicas
  -> Content nodes: apply the document operation
  -> Replies return through the distributor
```

### Components

A bucket is a group of documents whose ids share a number of low-order bits. It is the unit used for replica management, splitting, joining, and much of the storage-side synchronization.

A distributor handles document operations for a subset of the buckets. It keeps metadata about their replicas and sends operations to the appropriate content nodes. The documents themselves are stored on those nodes, so a distributor can reconstruct its bucket database after a restart.

The distributor and storage replicas are selected separately:

| Mapping | Result |
| --- | --- |
| Bucket to distributor | The owner that handles document operations |
| Bucket to storage nodes | The nodes that should hold its replicas |

Both mappings use the distribution algorithm. The owner can handle operations for a bucket whose replicas are stored on other nodes.

The Put goes through D3 to the two storage copies. The diagram's rows only organize this flat cluster; in a grouped configuration, each row shown would instead hold a full dataset copy.

[![Six nodes in two rows of three, with highlighted Put arrows from the container through owner D3 to bucket A's copies inside proton on nodes 0 and 1. Other components are muted. Each row contains sample buckets A through F once. In the flat cluster, rows only organize the drawing; in a grouped variant, each row represents one replica group holding a full dataset copy.](https://image.inhzus.io/posts/vespa-distributors-and-buckets/cluster-overview.svg)](https://image.inhzus.io/posts/vespa-distributors-and-buckets/cluster-overview.svg)

### BucketId

Routing starts by turning the document id into a bucket id. [BucketIdFactory][bucket-factory] constructs a 64-bit value with this layout, from the most significant bit to the least significant bit:

```text
63        58 57                      32 31                     0
+-----------+--------------------------+------------------------+
| used bits |       GID-derived bits   |        location        |
|  6 bits   |          26 bits         |        32 bits         |
+-----------+--------------------------+------------------------+
```

The top six bits say how many of the remaining 58 bits identify the bucket. For example, a bucket with 16 used bits contains documents with the same lowest 16 bits. Increasing the count gives a smaller set of documents.

For an ordinary id such as `id:news:article::kitten-finds-home-2024`, the location is derived from the MD5 hash of its namespace-specific part, `kitten-finds-home-2024`. The `n=` and `g=` schemes can override the location to co-locate related documents. The remaining bucket bits are derived from the document's global id. The details are in [IdString][id-string], [DocumentId][document-id], and the bucket factory.

The factory returns a full 58-bit identifier, which the distributor resolves to an existing ancestor bucket, usually with fewer used bits.

Since documents in a bucket share the low-order bits, [BucketId::toKey][bucket-key] reverses the data bits and retains the used-bit count in the database key. This keeps related buckets together for parent/child lookups.

### Distribution

Given the same bucket, cluster state, and configuration, the [distribution algorithm][distribution] computes the same ideal nodes. Nodes therefore do not need to ask a separate service for every bucket's placement.

For a flat cluster, the process is roughly:

1. Check that the bucket uses at least as many bits as the cluster's distribution bit count.
2. Derive a random seed from the bucket id.
3. Generate a deterministic score for each eligible node, adjusting for its capacity.
4. Select the highest-scoring nodes up to the required replica count.

Hierarchical configurations first select replica groups, then nodes within those groups.

`getIdealDistributorNode` selects one distributor. `getIdealStorageNodes` selects the storage replicas. Their seeds are related, but the storage seed can incorporate additional high-order bucket bits after sufficiently deep splitting.

### Finding the distributor

After document processing, the MessageBus [ContentPolicy][content-policy] calculates the bucket id and uses its cached cluster state to select the owner distributor. The request is addressed directly to that distributor.

The receiving distributor checks ownership again. If the client's cluster state is stale, the request can receive `WrongDistribution`, carrying state information that lets routing retry. A distributor losing ownership in a pending state can also return `BUSY` during the transition.

### Selecting replicas

[PutOperation][put-operation] resolves the full document bucket id to an existing bucket in the database. If none exists, it can create an appropriate bucket using the configured minimum split depth.

#### Ideal state and current state

The ideal nodes are where the replicas should be. The bucket database records where replicas currently exist and what is known about them. During failures or redistribution, these can differ.

The [bucket database][bucket-db] uses a B-tree with single-writer/multiple-reader access. Each bucket entry records replica metadata, including node index, timestamp, checksum, document count, and size. It also records flags such as `trusted` and `active`.

`trusted` records the distributor's assessment of replica consistency, using metadata such as checksum and document count. `active` determines whether the copy participates in search; the activation policy also depends on the cluster's group configuration.

Storage replies and bucket-info requests update this metadata. Each distributor [stripe][stripe] owns a disjoint subset of buckets and sequences writes to its database, allowing the process to work across multiple stripes.

During redistribution, some replicas may still be on their old nodes. Target selection accounts for those existing copies along with ideal placement.

For our bucket, D3 sends bucket operations and Put commands to nodes 0 and 1.

### Applying the write

Each target now applies the Put. Its storage service layer coordinates the operation with bucket locking and dispatches it through the persistence SPI. [AsyncHandler::handlePut][async-put] eventually calls the provider's `putAsync`.

For indexed storage, the service layer runs inside the searchnode process. [ProtonServiceLayerProcess::getProvider][proton-provider] returns proton's persistence engine, so this handoff is an in-process call.

[PersistenceEngine::putAsync][persistence-put] finds the document-type handler and forwards the operation. Further down, [FeedHandler::performPut][feed-put] prepares it, appends it to the transaction log, and passes it to the feed view. Attribute, index, and document-store work is then handled by the relevant parts of proton.

The different index structures are updated separately. Their concurrency mechanisms protect individual accesses, but a search overlapping an unfinished write can observe partially applied changes. See [the thread-safety post](@/posts/2026-09-20-vespa-storage-engine-thread-safety.md) for the implementation.

### Returning the result

Storage replies carry updated bucket information, which the distributor incorporates into its database. The reply tracker also decides when to return the client response.

[PersistenceMessageTracker][reply-tracker] normally waits for the outstanding messages to finish. Early acknowledgement can let the client proceed sooner: it depends on the configured initial redundancy and may also require completion of the primary target. Our redundancy of 2 specifies the replica count; the acknowledgement settings determine when the response can return.

The [consistency documentation](https://docs.vespa.ai/en/content/consistency.html#write-durability-and-consistency) describes the external guarantee: acknowledged writes are durable, and search changes are visible after a successful response by default. A configured visibility delay changes the latter behavior. A failed response can still mean that some replicas applied the write.

## Search requests

The search path is shorter:

```text
Container -> Searchnodes -> Container
```

The container dispatches queries to searchnodes and merges their results. Distributors are not part of this request path. They do affect search availability indirectly by maintaining which bucket copies are active.

In the standard [content-node configuration][node-config], a storage node and its distributor share a host and distribution key, with a searchnode associated with the storage node.

The [matching overview](@/posts/2025-05-26-vespa-matching-top-to-down.md) covers the query side in more detail.

## Maintaining replicas

The Put has finished, but D3 still has work to do for this bucket. Its replicas may grow, diverge, or need to move after a cluster-state change. The distributor repeatedly compares its bucket database with ideal state. [IdealStateManager][ideal-state] runs an ordered set of checkers:

| Checker | Work |
| --- | --- |
| Bucket state | Activate or deactivate replicas |
| Split bucket | Split buckets exceeding configured limits |
| Split inconsistency | Reconcile inconsistent split layouts |
| Synchronize and move | Merge divergent replicas and move data toward ideal placement |
| Join buckets | Join eligible small buckets |
| Extra copies | Remove excess replicas |
| Garbage collection | Remove documents matching configured expiry rules |

The first applicable checker in the configured order supplies the selected maintenance result. The code still runs the remaining checkers to collect statistics.

A merge synchronizes the timestamped documents held by replicas. Moving a bucket can use the same mechanism: create a copy at the destination, merge the data into it, then remove an unnecessary old copy. The actual transfer therefore takes time even though calculating the new ideal nodes is inexpensive.

### Splitting and joining

As documents accumulate, large buckets make replica operations more expensive. Very small buckets incur more metadata overhead. Splitting and joining balance these costs.

The [distributor configuration][split-config] defines separate size and document-count limits. In this revision, the low-level split defaults are 16 MB and 1024 documents; the join defaults are 16 MB and 512 documents. For a running cluster, use its generated configuration to determine the effective limits.

A split can be triggered while processing a Put, by a background size/count check, by an inconsistent split layout, or by the need to reach the cluster's minimum distribution depth. [SplitOperation][split-operation] sends commands to replicas and updates the database from their replies.

#### Parent and child buckets

Suppose our bucket uses 16 bits and its location is `0x2717`. Both nodes 0 and 1 split their copy by one additional bit:

[![A parent bucket with the low 16 bits 0x2717 splits into 17-bit children 0x02717 and 0x12717, distinguished by bit 16. On both replica nodes, the parent is replaced by both children, leaving two copies of each child.](https://image.inhzus.io/posts/vespa-distributors-and-buckets/bucket-split.svg)](https://image.inhzus.io/posts/vespa-distributors-and-buckets/bucket-split.svg)

The two children differ at bit 16, counting from zero. Each document goes to the child matching that bit in its full identifier.

Splitting this 16-bit bucket into 17-bit children leaves the placement seed unchanged when the distribution bit count stays the same. In this revision, extra storage-seed mixing begins only above 33 used bits. A split therefore need not change the ideal nodes.

The split partitions documents on the nodes holding the parent. If the resulting placement requires migration, separate maintenance work moves the replicas.

Joining combines suitable small buckets. Both operations must coordinate the distributor's metadata with the content nodes while accounting for requests already in flight under the old bucket layout.

### When a node fails

Return to the two copies on nodes 0 and 1. If node 0 fails, node 1 still holds the documents. Once the cluster controller marks node 0 unavailable and publishes a new state, distributors recalculate ownership and placement. The bucket database transition removes unavailable copies from the usable view and incorporates information gathered for the new state.

If a surviving searchable copy needs activation, maintenance schedules it. Search coverage can be reduced during this period, and losing every copy of a bucket cannot be repaired from metadata alone.

Suppose the new ideal targets are nodes 1 and 2. Synchronization can copy the surviving data to node 2 and restore redundancy. Write availability during recovery depends on the cluster state and available targets, rather than a fixed majority rule.

During ownership transfer, [ExternalOperationHandler][ownership-check] can reject writes with `STALE_TIMESTAMP` before the configured safe time. This timing constraint accounts for operations associated with the previous owner while the transition settles; it does not establish strong consistency.

### When only a distributor fails

If only D3 fails, both storage copies can remain on nodes 0 and 1. The new cluster state assigns D3's buckets to other distributors; suppose D5 becomes the owner of this one. It needs to gather bucket information from the storage nodes to rebuild its metadata.

[StripeBucketDBUpdater][bucket-updater] handles this transition. The old owner drops buckets it no longer owns, while the new owner incorporates reports from storage nodes. Requests may be retried while this is happening.

With storage placement unchanged, recovery requires rebuilding the owner's view of the bucket. Losing a storage copy requires document transfer as well:

[![Before, transition, and recovered views for bucket A. A distributor-only failure transfers ownership from D3 to D5 through bucket metadata reports while copies remain on nodes 0 and 1. A storage-node failure leaves a surviving copy on node 1; the new ideal targets are nodes 1 and 2, and document synchronization restores the second copy on node 2.](https://image.inhzus.io/posts/vespa-distributors-and-buckets/failure-recovery.svg)](https://image.inhzus.io/posts/vespa-distributors-and-buckets/failure-recovery.svg)

*Dashed arrows: metadata. Green arrow: document transfer.*

[bucket-factory]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/document/src/vespa/document/bucket/bucketidfactory.cpp
[id-string]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/document/src/vespa/document/base/idstring.cpp#L155
[document-id]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/document/src/vespa/document/base/documentid.cpp#L40
[bucket-key]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/document/src/vespa/document/bucket/bucketid.h#L125
[distribution]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/vdslib/src/vespa/vdslib/distribution/distribution.cpp#L352
[bucket-db]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/bucketdb/btree_bucket_database.h
[stripe]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/distributor/distributor_stripe.h
[content-policy]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/documentapi/src/main/java/com/yahoo/documentapi/messagebus/protocol/ContentPolicy.java
[put-operation]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/distributor/operations/external/putoperation.cpp
[async-put]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/persistence/asynchandler.cpp#L144
[proton-provider]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/apps/proton/proton.cpp#L178
[persistence-put]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/vespa/searchcore/proton/persistenceengine/persistenceengine.cpp#L305
[feed-put]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/searchcore/src/vespa/searchcore/proton/server/feedhandler.cpp#L144
[reply-tracker]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/distributor/persistencemessagetracker.cpp#L116
[node-config]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/config-model/src/main/java/com/yahoo/vespa/model/content/StorageGroup.java#L547
[ideal-state]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/distributor/idealstatemanager.cpp
[ownership-check]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/distributor/externaloperationhandler.cpp
[bucket-updater]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/distributor/stripe_bucket_db_updater.cpp
[split-config]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/config/stor-distributormanager.def
[split-operation]: https://github.com/vespa-engine/vespa/blob/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2/storage/src/vespa/storage/distributor/operations/idealstate/splitoperation.cpp
