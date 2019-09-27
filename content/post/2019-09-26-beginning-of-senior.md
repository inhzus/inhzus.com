---
title: 大四开始时的琐事
date: 2019-09-27 10:57:49+08:00
tags: [linux, multi-thread, c++]
---

很久不写博客，聊聊最近的历程。

上个月一直在实习，且花了大量的时间折腾 [go redis 项目](https://github.com/inhzus/go-redis-impl)。这个项目前前后后大概花了三周时间，最终的 scale 还是比较小，cluster, master-slave 这些重要的功能，由于我认为实现可能需要再完全重构一版，所以暂时搁置了。回顾整个项目，我的看法如下：

- 熟悉了很多 Go 基本的特性，尤其是 goroutine 相关的通信原语；同时也熟悉了 redis 的大量原理。
- 过于拘泥于业务逻辑，我认为我实现 redis 的目标不应该是写大量的业务代码。

- 服务端的模型过于简陋且低效。为了方便，在实现过程中，使用的模型大概可以描绘为：An acceptor & One goroutine per request & A consumer。而在了解 [gev](https://www.v2ex.com/t/602376) 与 [鸟窝: 百万 Go TCP 连接的思考](https://colobu.com/2019/02/23/1m-go-tcp-connection/) 后，深感这个项目的幼稚。
- 至少激发了多线程服务端编程的很多的想法。

回到学校后，签了某大厂的录取意向书，闲下来受这个项目的启发，买了一些书。包括：UNP Volume 1&2, APUE, Effective Modern C++ 还有就是今天刚到的陈硕的 Linux 多线程服务端编程。前两周的时间里，大概翻阅了 UNP 中一些比较基础的章节，包括 TCP socket 编程、Posix IPC、SystemV IPC 等等。后来，认为不够主次分明，看得实在有些乏力。遂花了两天时间看了些小说调节心情。之后打算从一些实际的网络库开始入手，也许能抓住要害。

