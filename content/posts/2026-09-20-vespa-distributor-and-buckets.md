+++
title = "Vespa: Distributors and Buckets"
date = "2026-09-20"
description = "How Vespa routes document operations, maintains replicas, and redistributes buckets."

[extra]
toc = true
+++

In the [Vespa comparison](@/posts/2024-10-12-vespa-vs-elasticsearch.md), I briefly mentioned that Vespa manages document distribution through buckets. This post expands on that part: how a document maps to a bucket, how the distributor chooses storage nodes, and what happens when the cluster changes.

The source references use Vespa revision [39bf68f4](https://github.com/vespa-engine/vespa/tree/39bf68f4ce9ea144c5f3a2bb2fd22890fd680ec2). The feeding example assumes an indexed content cluster with a flat distribution configuration. Grouped configurations add another level of placement decisions.

## Components

A bucket is a group of documents whose ids share a number of low-order bits. It is the unit used for replica management, splitting, joining, and much of the storage-side synchronization.

A distributor handles document operations for a subset of the buckets. It keeps metadata about their replicas and sends operations to the appropriate content nodes. The documents themselves are stored on those nodes, so a distributor can reconstruct its bucket database after a restart.

It helps to keep two mappings separate:

| Mapping | Result |
| --- | --- |
| Bucket to distributor | The owner that handles document operations |
| Bucket to storage nodes | The nodes that should hold its replicas |

Both use the distribution algorithm, but answer different questions. A distributor may own a bucket whose data is stored elsewhere.

## BucketId

[BucketIdFactory][bucket-factory] constructs a 64-bit value from the document id. From the most significant bit to the least significant bit, its layout is:

```text
63        58 57                      32 31                     0
+-----------+--------------------------+------------------------+
| used bits |       GID-derived bits   |        location        |
|  6 bits   |          26 bits         |        32 bits         |
+-----------+--------------------------+------------------------+
```

The top six bits say how many of the remaining 58 bits identify the bucket. For example, a bucket with 16 used bits contains documents with the same lowest 16 bits. Increasing the count gives a smaller set of documents.

For an ordinary id such as `id:news:article::kitten-finds-home-2024`, the location is derived from the MD5 hash of its namespace-specific part, `kitten-finds-home-2024`. The `n=` and `g=` schemes can override the location to co-locate related documents. The remaining bucket bits are derived from the document's global id. The details are in [IdString][id-string], [DocumentId][document-id], and the bucket factory.

The factory returns a full 58-bit identifier. This does not mean Vespa creates a separate bucket for every such id. The distributor looks for the existing ancestor bucket, which usually uses fewer bits.

### Parent and child buckets

Suppose a bucket uses 16 bits and its location is `0x2717`. Splitting by one additional bit gives:

```text
16 bits:       0x2717
              /      \
17 bits:  0x02717    0x12717
```

The two children differ at bit 16, counting from zero. Each document goes to the child matching that bit in its full identifier.

The shared bits are at the low end of the number. To keep related buckets near one another in the bucket database, [BucketId::toKey][bucket-key] reverses the data bits and retains the used-bit count as part of the key. Parent/child lookups can then follow this ordering.

## Distribution

Given the same bucket, cluster state, and configuration, the [distribution algorithm][distribution] computes the same ideal nodes. Nodes therefore do not need to ask a separate service for every bucket's placement.

For a flat cluster, the process is roughly:

1. Check that the bucket uses at least as many bits as the cluster's distribution bit count.
2. Derive a random seed from the bucket id.
3. Generate a deterministic score for each eligible node, adjusting for its capacity.
4. Select the highest-scoring nodes up to the required replica count.

The random sequence is reproducible. Its purpose is to spread buckets across nodes while letting all participants calculate the same result. With hierarchical groups, the algorithm first determines which groups should receive replicas, then selects nodes within those groups.

`getIdealDistributorNode` selects one distributor. `getIdealStorageNodes` selects the storage replicas. Their seeds are related, but the storage seed can incorporate additional high-order bucket bits after sufficiently deep splitting.

This last point is easy to miss: splitting a bucket does not necessarily change its ideal nodes. In this revision, the extra storage-seed mixing begins only when the bucket uses more than 33 bits. A split from 16 to 17 bits, with the distribution bit count unchanged, does not by itself introduce a new seed.

### Ideal state and current state

The ideal nodes are where the replicas should be. The bucket database records where replicas currently exist and what is known about them. During failures or redistribution, these can differ.

The [bucket database][bucket-db] uses a B-tree with single-writer/multiple-reader access. Each bucket entry records replica metadata, including node index, timestamp, checksum, document count, and size. It also records flags such as `trusted` and `active`.

`trusted` records the distributor's assessment of replica consistency, using metadata such as checksum and document count. `active` determines whether the copy participates in search; the activation policy also depends on the cluster's group configuration.

The distributor updates this metadata from storage replies and bucket-info requests. Its work is partitioned into [stripes][stripe], each responsible for a disjoint subset of buckets. The single-writer restriction applies to each database; it does not mean the whole distributor process has only one worker thread.

## Feeding a document

Assume eight content nodes, with corresponding distributors, a flat distribution, 16 distribution bits, and redundancy 2. We will follow a Put for `id:news:article::kitten-finds-home-2024`.

```text
Client
  -> Container: document processing and routing
  -> Owner distributor: resolve bucket and replicas
  -> Content nodes: apply the document operation
  -> Replies return through the distributor
```

### Finding the distributor

After document processing, the MessageBus [ContentPolicy][content-policy] calculates the bucket id and uses its cached cluster state to select the owner distributor. The request is addressed directly to that distributor.

The receiving distributor checks ownership again. If the client's cluster state is stale, the request can receive `WrongDistribution`, carrying state information that lets routing retry. A distributor losing ownership in a pending state can also return `BUSY` during the transition.

### Selecting replicas

[PutOperation][put-operation] resolves the full document bucket id to an existing bucket in the database. If none exists, it can create an appropriate bucket using the configured minimum split depth.

Target selection takes both the existing replicas and the ideal placement into account. This matters during redistribution: simply ignoring every non-ideal copy would lose track of data still being moved.

For the rest of the example, suppose the chosen storage targets are nodes 1 and 0. These are illustrative targets, not a calculated result for this document. The distributor sends the corresponding bucket operations and Put commands to the targets.

### Applying the write

On a content node, the storage service layer coordinates the operation with bucket locking and dispatches it through the persistence SPI. [AsyncHandler::handlePut][async-put] eventually calls the provider's `putAsync`.

In an indexed cluster, this provider is inside the searchnode process. [ProtonServiceLayerProcess::getProvider][proton-provider] returns proton's persistence engine. Thus the call from the storage service layer to proton is an in-process call; there is no additional network hop between them.

[PersistenceEngine::putAsync][persistence-put] finds the document-type handler and forwards the operation. Further down, [FeedHandler::performPut][feed-put] prepares it, appends it to the transaction log, and passes it to the feed view. Attribute, index, and document-store work is then handled by the relevant parts of proton.

The different index structures are updated separately. Their concurrency mechanisms protect individual accesses, but a search overlapping an unfinished write can observe partially applied changes. See [the thread-safety post](@/posts/2026-09-20-vespa-storage-engine-thread-safety.md) for the implementation.

### Returning the result

Storage replies carry updated bucket information, which the distributor incorporates into its database. The reply tracker also decides when to return the client response.

Without early acknowledgement enabled, [PersistenceMessageTracker][reply-tracker] waits for the outstanding messages to finish. Early replies use the configured initial redundancy and may require the primary target to have completed. Redundancy 2 alone does not imply that the first successful reply is sufficient.

The majority-success helper in that class is used when handling partial test-and-set failures. It is not the general rule for acknowledging every Put.

The [consistency documentation](https://docs.vespa.ai/en/content/consistency.html#write-durability-and-consistency) describes the external guarantee: acknowledged writes are durable, and search changes are visible after a successful response by default. A configured visibility delay changes the latter behavior. A failed response can still mean that some replicas applied the write.

## Search requests

The search path is shorter:

```text
Container -> Searchnodes -> Container
```

The container dispatches queries to searchnodes and merges their results. Distributors are not part of this request path. They do affect search availability indirectly by maintaining which bucket copies are active.

In the standard [content-node configuration][node-config], a storage node and its distributor share the host and distribution key, and a searchnode is associated with the storage node. For indexed storage, the storage service layer runs inside proton as described above. A Kubernetes pod is one possible deployment unit; it is not part of the bucket algorithm itself.

The [matching overview](@/posts/2025-05-26-vespa-matching-top-to-down.md) covers the query side in more detail.

## Maintaining replicas

The distributor repeatedly compares its bucket database with ideal state and schedules maintenance work. [IdealStateManager][ideal-state] runs an ordered set of checkers:

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

### When a node fails

Once the cluster controller marks a node unavailable and publishes a new state, distributors recalculate ownership and placement. The bucket database transition removes unavailable copies from the usable view and incorporates information gathered for the new state.

If a surviving searchable copy needs activation, maintenance schedules it. Search coverage can be reduced during this period, and losing every copy of a bucket cannot be repaired from metadata alone.

With a surviving copy, synchronization work can restore redundancy on other eligible nodes. Writes can continue when the cluster state and available targets permit them; there is no fixed majority rule that guarantees availability for every failure pattern.

Ownership changes also have a timing constraint. [ExternalOperationHandler][ownership-check] can reject writes with `STALE_TIMESTAMP` before the configured ownership-transfer safe time. This accounts for operations associated with the previous owner while the transition is settling. It is not a replacement for strong consistency.

### When only a distributor fails

If the storage nodes remain available, losing a distributor does not remove their documents. The new cluster state assigns its buckets to other distributors, which gather bucket information from the storage nodes and rebuild the relevant metadata.

[StripeBucketDBUpdater][bucket-updater] handles this transition. The old owner drops buckets it no longer owns, while the new owner incorporates reports from storage nodes. Requests may be retried while this is happening.

If storage placement is unchanged, this ownership change does not require moving the documents. Only the responsibility for routing and maintenance has changed.

## Splitting and joining

Besides failures, ordinary data growth creates maintenance work. Large buckets make replica operations more expensive, while very small buckets consume more metadata. Splitting and joining balance these costs.

The [distributor configuration][split-config] defines separate size and document-count limits. In this revision, the low-level split defaults are 16 MB and 1024 documents; the join defaults are 16 MB and 512 documents. These are configuration-definition defaults, so the generated configuration should be checked when investigating a running cluster.

A split can be triggered while processing a Put, by a background size/count check, by an inconsistent split layout, or by the need to reach the cluster's minimum distribution depth. [SplitOperation][split-operation] sends commands to replicas and updates the database from their replies.

The split itself partitions documents into child buckets on the nodes holding the parent. Replica migration, if required by the resulting placement, is separate maintenance work. As noted earlier, an extra split bit does not always change placement.

Joining performs the reverse operation for suitable small buckets. Both operations must keep the distributor's metadata aligned with the content nodes, including requests that were already in flight when the bucket layout changed. This coordination is a substantial part of the implementation, even though the basic bit partition is simple.

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
