---
title: "Vespa 详解及对比 Elasticsearch"
date: 2024-10-15T22:33:53+08:00
toc: true
---

在这家公司做了两年 Vespa 相关开发后，当被问到 Vespa 相比 Elasticsearch 的优势时，我居然很茫然。于是又重新好好地梳理了下 Vespa 的逻辑，参照之前了解的 ES（Elasticsearch）的一些概念，记录下这篇文章来总结这两者的区别。同时推荐读我的另外两篇更深入讲解 Vespa 原理的博客：[Vespa 索引结构实现](/posts/2024-12-04-vespa-index-impl/)，[Vespa 索引检索实现](/posts/2024-12-05-vespa-match-impl/)。

## 真正的实时更新

在 lucene 中，一个 index 由多个 segments 组成。[^1] 新的文档会先写入内存中的 buffer，一定时间后，写入 filesystem cache 后再 flush 至磁盘。新添加的文档是不可被搜的，直至写入 filesystem cache。有检索请求时，内存中文档 flush 至文件的频率默认为 1s。 [^2] 

Vespa 有两种索引模式，一种为 index，适合文本检索，支持 tokenizer / stemming / normalization，与 ES keyword 类型类似的会构建倒排索引；另一种为 attribute，是全内存的列存索引结构，以 local docId 为序存于 RCU Vector 中，打开某项配置后会在内存中构建倒排。

除了 index 和 attribute 两种列存索引，Vespa 还有类似于 ES 中 stored fields 的行存正排：document store。文档被顺序写入磁盘中，随着文件大小的增长分割为多份（默认不大于 1G）。

Vespa 的 index 由一份 memory index 和一份以上 immutable disk index 组成，文档更新会立刻作用于 memory index。Memory index 会不定期 flush 至磁盘，随后和 primary disk index 合并（称为 fusion）形成新的 disk index。在搜索时，memory index 和 disk index 都会被检索。[^3]

由此引出一个问题，Lucene 存储的 segments 的数量可能达到上百，而 Vespa 却往往只有一两份 disk index，是否 Vespa 的 fusion 开销会过大以至于影响性能？答案是否定的[^4]：

- Vespa 可以控制 feeding 使用到的线程比例，减少对 searching 的影响；
- Lucene segments 为行存，而 Vespa 是列存的，每个 field 都有单独的 memory/disk index；
- Lucene 合并 segments 重写了大量的数据，而 Vespa 中由于倒排和词典都是已排序的，几乎都是顺序读写，开销相对低很多（TODO：这里以后再具体展开）；
- 文档更新总是先写入内存，减少了垃圾回收和文档合并的频率；
- 过多的 Segments 使得同一个文档要经历更多的合并次数，增加 IO 读写。

反过来讲，较少的 index 数量带来比 Lucene 更高的查询效率。

## 动态扩缩容

ES 不能非常方便地扩缩容，需要人为参与到数据迁移的过程中，才能增加 shard。[^7][^8]

Vespa 使用与 Key-Value 存储类似的机制，文档被 hash 至 buckets 进行自动的管理，不需要手动控制分片。当增加 node/shard 时，会重新计算 bucket 的分布，存储节点间自动迁移文档至新的 node；而当 node 挂掉或在配置文件中被删除时，也会重新分配 bucket，文档会自动地从其它节点的备份中恢复，再复制出满足最少 replica 的备份。[^9]

相应的为了实现这样的功能，Vespa 的 bucket 操作的代码逻辑复杂度是相对比较高的，需要为此付出一些维护代价。

## Partial Update

Lucene 不存在真正的 partial update，segments 均是 immutable。[^5] 更新文档时，将已有 segment 中该文档处标记为已删除，之后的流程即是增加一个新的文档。[^6] 注意到当更新文档量频率较高时，该实现会在一定程度上影响检索性能。

Vespa 的三种存储形态中，index，attribute 为内存中更新，定期 flush 至内存。document store 不涉及倒排检索，需要顺序写入更新后文档的全部字段。

## 向量性能

Lucene 的向量（dense vector）和其余类型一样，分布存在多个 segments 中，自然地，为此每次合并 segments 都需要重新计算 HNSW 图，而每次搜索也需要在多个 segments 上计算 HNSW 后再合并，带来了额外的开销。[^13]

向量（tensor）在 Vespa 中是第一公民，在内存中生成 HNSW 图，存储全部文档的向量，[^11] 在性能上显然比 ES 更快。[^12] 但由于 ES 的向量存储在磁盘上，而 Vespa 只能全量放在内存中，不能支持大于内存的向量空间的查询。

## 多线程查询

Vespa 在设计上屏蔽了 matching 对下层多种索引结构的感知，抽象为了对本分片上 local docId 的“遍历”查询，于是可以天然地将文档 id 切割为多个段，用于多线程查询。

当然在搜索引擎的场景中并不是线程越多延迟和吞吐就会越好，不过对于特定的业务场景和规模，设置数个线程并发查询能带来明显的改善。

## 大模型 Ranking

Vespa 支持众多大模型，如 Tensorflow，ONNX 等；还支持 Huggingface，Bert 等进行 Embedding。[^10] 不过我们线上环境中没有使用。



TBC



[^1]: [知乎 Lucene 解析](https://zhuanlan.zhihu.com/p/35469104)
[^2]: [Elastic: Near real-time search](https://www.elastic.co/guide/en/elasticsearch/reference/current/near-real-time.html)
[^3]: [Vespa: Index](https://docs.vespa.ai/en/proton.html#index)
[^4]: [Vespa Slack: Performance of Fusion](https://vespatalk.slack.com/archives/C01QNBPPNT1/p1728892957219879)
[^5]: [Elastic: Merge](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-merge.html)
[^6]: [Elastic Blog: Lucene's Handling of Deleted Documents](https://www.elastic.co/blog/lucenes-handling-of-deleted-documents)
[^7]: [Elastic: Shrink Index](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-shrink-index.html)
[^8]: [Elastic: Split Index](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-split-index.html)
[^9]: [Vespa: Elasticity](https://docs.vespa.ai/en/elasticity.html)
[^10]: [Vespa: Ranking](https://docs.vespa.ai/en/ranking.html)
[^11]: [Vespa: ANN Search using HNSW](https://docs.vespa.ai/en/approximate-nn-hnsw.html)
[^12]: [GitHub: Dense Vector Ranking Performance](https://github.com/jobergum/dense-vector-ranking-performance)
[^13]: [Elastic: Vector Search in Elasticsearch](https://www.elastic.co/search-labs/blog/vector-search-elasticsearch-rationale)

