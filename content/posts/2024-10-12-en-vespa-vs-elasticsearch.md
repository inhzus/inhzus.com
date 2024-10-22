---
title: "Implementation of Vespa and Its Comparison with Elasticsearch"
date: 2024-10-15T22:33:53+08:00
toc: true
vice: true
---

After two years of Vespa-related development at this company, I was surprisingly at a loss when asked about the advantages of Vespa over Elasticsearch. Therefore, I took the time to scan Vespa’s code again, comparing it with some concepts I had learned from Elasticsearch (ES), and documented this article to summarize the differences between the two.

## True Real-Time Updates

In Lucene, an **index** consists of multiple segments.[^1] New documents are first written to an in-memory buffer and then flushed to disk after a certain period of time once they are written to the filesystem cache. Newly added documents cannot be searched until they are written to the filesystem cache. During a search request, the in-memory documents are flushed to files with the default flush frequency being 1 second.[^2]

Vespa has two types of index modes. One is called **index**, which is suitable for text search, supporting tokenizer, stemming, and normalization. It builds an inverted index similar to the keyword type in ES. The other is called **attribute**, which is a fully in-memory columnar index structure, stored in order by local docId in an RCU vector. When certain configurations are enabled, a B-Tree inverted index is built in memory.

In addition to the columnar indexes of index and attribute, Vespa also has a row-based forward index similar to stored fields in ES: the document store. Documents are written sequentially to disk and split into multiple parts as the file size grows (with a default size limit of 1GB).

A Vespa index consists of a **memory index** and one or more immutable **disk indexes**. Document updates immediately affect the memory index, which is periodically flushed to disk and then merged with the primary disk index (this process is called **fusion**) to form a new disk index. During searching, both memory and disk indexes are queried.[^3]

This leads to a question: While the number of segments stored in Lucene can reach hundreds, Vespa typically only has one or two disk indexes. Does this mean that the fusion process in Vespa is too costly and affects performance? The answer is no:[^4]

- Vespa can control the number of threads used for feeding, reducing the impact on searching.
- Lucene segments are row-based, while Vespa is column-based, with each field having its own memory/disk index.
- Lucene rewrites a large amount of data during segment merges, whereas in Vespa, due to the sorted nature of its inverted index and dictionary, the operations are mostly sequential reads and writes, making them much less costly (this will be explained in more detail later).
- Document updates are always written to memory first, reducing garbage collection and document merge frequency.
- Too many segments lead to more merges for the same document, increasing I/O operations.

In contrast, having fewer indexes in Vespa results in higher query efficiency compared to Lucene.

## Dynamic Scaling

ES does not support seamless scaling, requiring manual intervention in the data migration process to add shards.[^7][^8]

Vespa uses a mechanism similar to Key-Value stores, where documents are hashed into buckets for automatic management, eliminating the need for manual shard control. When nodes or shards are added, the distribution of buckets is recalculated, and documents are automatically migrated between storage nodes. Similarly, when nodes fail or are removed from the configuration, buckets are redistributed, and documents are automatically restored from backups on other nodes, with additional replicas created to meet the minimum replica requirement.[^9]

However, to implement this functionality, the code logic for Vespa’s bucket operations is relatively complex, requiring some maintenance effort.

## Partial Update

Lucene does not have true partial updates, as segments are immutable.[^5] When a document is updated, the document in the existing segment is marked as deleted, and a new document is added.[^6] Note that this approach can affect search performance when updates occur frequently.

Among Vespa’s three storage types, index and attribute are updated in memory and periodically flushed. The document store does not involve inverted index search and requires sequential writing of all fields of the updated document.

## Sparse Vector Performance

Sparse vectors (dense vector) in Lucene, like other types, are distributed across multiple segments. As a result, each time segments are merged, the HNSW graph needs to be recalculated. Every query searching involves running HNSW on multiple segments and then merging the results, introducing extra overhead.[^13]

Sparse vectors (tensor) in Vespa are treated as first-class citizens, with the HNSW graph generated in memory, storing vectors for all documents.[^11] In terms of performance, Vespa is clearly faster than ES.[^12] However, since ES stores vectors on disk, while Vespa stores them entirely in memory, Vespa does not support queries on vector spaces larger than available memory.

## Multi-Threaded Queries

Vespa is designed to abstract the matching process from the underlying index structures, treating local docId traversal as the primary operation. This design naturally allows the document ids to be split into segments for multi-threaded queries.

Of course, in search engine scenarios, more threads do not always imply better latency and throughput. However, in certain business scenarios and scales, setting a few concurrent threads for querying can lead to significant improvements.

## Machine-Learned Model Inference 

Vespa supports many machine-learned models, such as TensorFlow, ONNX, and also Huggingface, BERT, and others for embedding.[^10] However, we do not use these in our online environment.

TBC

[^1]: Zhihu: Lucene Analysis
[^2]: Elastic: Near real-time search
[^3]: Vespa: Index
[^4]: Vespa Slack: Performance of Fusion
[^5]: Elastic: Merge
[^6]: Elastic Blog: Lucene’s Handling of Deleted Documents
[^7]: Elastic: Shrink Index
[^8]: Elastic: Split Index
[^9]: Vespa: Elasticity
[^10]: Vespa: Ranking
[^11]: Vespa: ANN Search using HNSW
[^12]: GitHub: Dense Vector Ranking Performance
[^13]: Elastic: Vector Search in Elasticsearch
