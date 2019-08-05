---
title: "Percolator笔记"
description: Percolator构建于Bigtable之上，提供跨行事务，用于处理增量数据的索引
date: '2019-08-05'
tags:
  - papers
  - google
keywords:
  - Distributed System
  - transaction
  - Google
---

Google 的索引系统存储数十 PB 的数据，每天有数十亿的更新。
MapReduce 适合处理大批量的数据，处理少量更新效率较低。
Percolator 为处理增量更新而设计，处理同样数据量的文档时可以将平均时延降低 50%。

## 1 介绍(Introduction)

爬虫抓取页面之后需要一系列 MapReduce 操作来构建索引。
包括聚类排重（处理相同页面的内容，显示 PageRank 高的），构建倒排。
MapReduce 限制并行计算，计算完 PageRank 对应的 url 之后再构建倒排，不用担心 PageRank 变化。

少量页面变化时构建索引需要把全部页面处理一遍，导致更新速度过慢。
索引可以存储在 DBMS 中，通过事务更新单行数据，但是 DBMS 无法支持 Google 的数据量（分布在数千台机器的数十PB数据）。
分布式的 Bigtable 可以支持这个数据量，但是不支持跨行事务，无法保证并发更新时数据的不变性。

为增量索引更新优化的数据处理系统应该满足如下要求：

- 可以维护一个巨大的文档仓库，同时在爬取到新页面的时候可以高效的更新
- 并发处理很多小的更新时需要维持不变性（事务）
- 跟踪哪些更新被处理了

Percolator 被设计用来解决这个问题，具有如下特点：

- 随机访问数十 PB 数据的仓库（避免全局扫描）
- 多台机器上的多个进程并发访问数据（高吞吐）
- 提供 ACID 事务支持（快照隔离级别 snapshot isolation）
- 提供观察者（observers），跟踪增量计算的状态
- 系统由一组观察者组成，级联处理数据，第一个观察者由外部进程触发

Percolator 专为处理增量数据设计，不适合如下场景：

- 不能划分为小更新的计算（排序）更适合用 MapReduce
- 计算没有强一致性要求，直接使用 BigTable
- 资源需求较少（数据量小、CPU需求少等），直接使用传统 DBMS

Google 内部使用 Percolator 构建网页索引。
在爬虫运行的同时处理文档，可以将平均时延减少50%。
通过跟踪网页与它依赖的资源，在任意依赖发生变化的时候对页面重新进行处理。

## 2 设计(Design)

Percolator 为处理大规模增量数据提供了两个主要的抽象：

- 支持随机访问的 ACID 事务
- 观察者，组织增量数据的一种方式

集群中的每台机器包含3部分：

- Percolator worker
- Bigtable tablet server
- GFS chunkserver

观察者关联到 Percolator worker，它扫描 bigtable 有变化的列，通过 function call 在进程内调用对应的观察者。
观察者通过向 Bigtable tablet servers 发送读写RPC来执行事务。

依赖的其他服务：

- timestamp oracle：提供严格增长的时间戳，用于正确操作快照隔离协议
- 轻量级锁服务：使脏数据的通知更高效

Percolator 由少量的 table 组成，每个 table 由一组包含 row 和 column 的 cell 组成。
每个 cell 包含一个值（为支持快照隔离，每个 cell 表示为一系列由时间戳索引的值）。

设计需求：

- 运行在大规模尺度之上（at massive scales）
- 不需要特别低的延迟（可以简化设计）
- 使用懒惰（lazy）的方式处理失败事务遗留的锁
- 延迟可能高达十几秒（OLTP 任务无法接受）
- 没有中心化的事务管理，缺乏全局死锁检测（增加了事务冲突的延迟）

### 2.1 Bigtable 概述(Bigtable overview)

Percolator 构建于 Bigtable 分布式存储系统之上

- 多维 sorted map：key 是（row, column，timestamp）元组
- 提供按 row 的查找和更新
- 提供 row 事务，原子读写
- 处理 PB 级别事务，运行与大量不可靠机器之上
- 由一组 tablet server 组成，每个 tablet server 管理几个 tablets（连续的 key 空间）
- master 协调 tablet server 的操作，迁移 tablet
- tablet 存储一组 SSTable 格式的只读文件
- SSTable 存储在 GFS，提供高可靠性
- 一组 column 可以组成一个 locality group，存储在一个 SSTable，提高 scan 效率

构建于 Bigtable 之上决定了 Percolator 的整体状况

- 数据被组织成 Bigtable 的 row 和 column，metadata 存储为特殊的 column
- API 类似于 Bigtable，增加事务相关的操作
- 提供 Bigtable 不具备的跨行事务（multirow transactions）和观察者框架（observer framework）

### 2.2 事务(Transactions)

Percolator 通过快照隔离实现跨行跨表的事务

> Percolator provides cross-row, cross-table transactions with ACID snapshot-isolation semantics.

以下代码展示了简化版的通过内容哈希聚类(clustering)文档的过程。
如果 `Commit()` 返回 `false`，说明事务有冲突（两个 url 同时处理），需要重试。
`Get()` 和 `Commit()` 调用是阻塞的，通过在线程池同时运行多个事务来提高并发度。

```cpp
bool UpdateDocument(Document doc) {
  Transaction t(&cluster);
  t.Set(doc.url(), "contents", "document", doc.contents());
  int hash = Hash(doc.contents());
  // dups table maps hash → canonical URL
  string canonical;
  if (!t.Get(hash, "canonical-url", "dups", &canonical)) {
    // No canonical yet; write myself in
    t.Set(hash, "canonical-url", "dups", doc.url());
  } // else this document already exists, ignore new copy
  return t.Commit();
}
```

##### 快照隔离(Snapshot Isolation)

- 通过 Bigtable 的 timestamp 存储每个数据的多个版本（multiple versions）
- 防止写-写冲突（write-write conflicts），两个事务同时写一个 cell，只有一个能提交
- 没有实现 serializable，存在 write skew（读写冲突）
- 比 serializable 更好的读性能

##### 锁(Lock)

锁服务的需求：

- 机器挂掉时，也不能提交有冲突的事务
- 高吞吐：数千台机器同时请求锁
- 低延迟：每个 `Get()` 都需要加锁，这个延迟需要最小化

锁需要实现的功能：

- 多副本应对故障 replicated (to survive failure)
- 平衡的分布数据 distributed and balanced (to handle load)
- 持久化 write to a persistent data store

##### BigTable 的 Columns

在 BigTable 抽象五个 Columns，其中三个跟事务相关

Column   | Use
-------- | -----------------------------------
c:lock   | 未提交事务写，包含 primary lock 的位置
c:write  | 已提交数据，Bigtable timestamp 的数据
c:data   | 存储数据
c:notify | Hints：需要运行的观察者
c:ack_O  | 已运行的观察者：存储上一次成功运行的 start timestamp

##### 写流程(Write Process)

初始化阶段

1. 从 timestamp oracle 获取 start timestamp
2. 调用 `Set()` 把需要写的数据缓存起来，数据由 row、column、value 组成
3. 接着由 client 协调开始两阶段提交（prewrite、commit）

prewrite 阶段，锁定所有需要写的 cell

1. 任意指定一个 cell 为 primary，其余为 secondary
2. 检查 primary 的 write column 在 start timestamp 之后是否有数据，如果有则退出（write-write conflict）
3. 检查 primary 的 lock column 在任意时段是否有数据，如果有则退出（另一个已提交的事务未及时释放锁，概率很低）
4. 对 primary 的 data column 在 start timestamp 写入 value
5. 对 primary 的 lock column 在 start timestamp 写入 primary 的 row 和 column 位置
6. 对各个 secondary 依次执行2-5的步骤

commit 阶段

1. client 从 timestamp oracle 获取 commit timestamp
2. 检查 primary 的 lock column 在 start timestamp 是否有数据，如果没有则退出
3. 对 primary 的 write column 在 commit timestamp 写入 start timestamp（指向数据的指针）
4. 对 primary 的 lock column 的数据删除，释放锁，此时事务完成提交
5. 对 secondary 执行3、4的操作，可能会延后操作

##### 读流程(Read Process)

1. 从 timestamp oracle 获取 start timestamp
2. 读取 row 的 lock column 在 0 到 start timestamp 之间的数据（事务快照中可见的数据）
3. 如果存在锁，说明有其它事务在写数据，此时需要等待锁的释放
4. 如果没有锁，读取 row 的 write column 在 0 到 start timestamp 之间的数据，拿到 data timestamp 的指针
5. 如果指针存在，读取 row 的 data column 在 data timestamp 的数据

##### 锁的清理(Lock Cleanup)

如果 client 在事务进行中失败，会导致死锁。
Percolator 采用懒惰（lazy）的方式进行清理：
如果事务 A 发现事务 B 遗留的死锁，A 需要判断 B 是否失败并清除锁。

Percolator 在每个事务中指定一个 primary cell 作为同步点（synchronizing point）。
清理或提交事务需要检查 primary 的 lock，通过 Bigtable 的 row 事务保证安全。
事务 B 在提交之前需要检查自己是否还持有 lock，只有持有才可以提交。
事务 A 在清理之前检查 lock 是否存在来保证事务 A 还没有提交。

事务遇到锁的时候可以通过检查 primary lock，有两种 case ：

1. 如果 primary lock 已经被 write record 取代，事务已提交，lock 需要 roll forward
2. 否则 rollback

rollback 导致事务退出存在性能问题，因此只能清理已经死掉的 worker。
Percolator 提供一个简单的方式判断另一个事务是否存活：
运行的 worker 在 Chubby 中写入一个 token，其它 worker 通过这个 token 判断这个 worker 是否存活。
token 存在一个 wall time，worker 定期向 Chubby 更新 wall time，过期的 worker 会被清除。

##### 示例(Example)

key  | bal:data | bal:lock           | bal:write
---- | -------- | ------------------ | ----------
Bob  | 6:       | 6:                 | 6: data@5
     | 5: $10   | 5:                 | 5:
Joe  | 6:       | 6:                 | 6: data@5
     | 5: $2    | 5:                 | 5:

初始化状态：Joe 的账户有 $2，Bob 的账户有 $10

key  | bal:data | bal:lock           | bal:write
---- | -------- | ------------------ | ----------
Bob  | 7: $3    | 7: I am primary    | 7:
     | 6:       | 6:                 | 6: data@5
     | 5: $10   | 5:                 | 5:
Joe  | 6:       | 6:                 | 6: data@5
     | 5: $2    | 5:                 | 5:

转账事务开始，在 Bob 账户的 lock column(primay)写入数据，同时在 data column 写入新的账户余额，start timestamp 为7

key  | bal:data | bal:lock           | bal:write
---- | -------- | ------------------ | ----------
Bob  | 7: $3    | 7: I am primary    | 7:
     | 6:       | 6:                 | 6: data@5
     | 5: $10   | 5:                 | 5:
Joe  | 7: $9    | 7: primary@Bob.bal | 7:
     | 6:       | 6:                 | 6: data@5
     | 5: $2    | 5:                 | 5:

在 Joe 账户的 lock column(secondary) 写入指向 primary 的指针，同时在 data column 写入新的账户余额

key  | bal:data | bal:lock           | bal:write
---- | -------- | ------------------ | ----------
Bob  | 8:       | 8:                 | 8: data@7
     | 7: $3    | 7:                 | 7:
     | 6:       | 6:                 | 6: data@5
     | 5: $10   | 5:                 | 5:
Joe  | 7: $9    | 7: primary@Bob.bal | 7:
     | 6:       | 6:                 | 6: data@5
     | 5: $2    | 5:                 | 5:

进入提交点：在 Bob 账户删除 lock column 的数据，同时在 write column 写入指向 start timestamp data 的指针，
commit timestamp 为 8

key  | bal:data | bal:lock           | bal:write
---- | -------- | ------------------ | ----------
Bob  | 8:       | 8:                 | 8: data@7
     | 7: $3    | 7:                 | 7:
     | 6:       | 6:                 | 6: data@5
     | 5: $10   | 5:                 | 5:
Joe  | 8:       | 8:                 | 8: data@7
     | 7: $9    | 7:                 | 7:
     | 6:       | 6:                 | 6: data@5
     | 5: $2    | 5:                 | 5:

完成事务，在 Joe 账户删除 lock column 的数据，同时在 write column 写入指向 start timestamp data 的指针

##### 伪代码(Pseudocode)

```cpp
class Transaction {
  struct Write {
    Row row;
    Column col;
    string value;
  };
  vector<Write> writes_;
  int start_ts_;

  Transaction() : start_ts_(oracle.GetTimestamp()) {}
  void Set(Write w) {
    writes_.push_back(w);
  }
  bool Get(Row row, Column c, string* value) {
    while (true) {
      bigtable::Txn T = bigtable::StartRowTransaction(row);
      // Check for locks that signal concurrent writes.
      if (T.Read(row, c+"lock", [0, start_ts_])) {
        // There is a pending lock; try to clean it and wait
        BackoffAndMaybeCleanupLock(row, c);
        continue;
      }
      // Find the latest write below our start timestamp.
      latest write = T.Read(row, c+"write", [0, start_ts_]);
      if (!latest_write.found())
        return false; // no data
      int data_ts = latest_write.start_timestamp();
      *value = T.Read(row, c+"data", [data_ts, data_ts]);
      return true;
    }
  }
  // Prewrite tries to lock cell w, returning false in case of conflict.
  bool Prewrite(Write w, Write primary) {
    Column c = w.col;
    bigtable::Txn T = bigtable::StartRowTransaction(w.row);
    // Abort on writes after our start timestamp ...
    if (T.Read(w.row, c+"write", [start_ts_, ∞]))
      return false;
    // ... or locks at any timestamp.
    if (T.Read(w.row, c+"lock", [0, ∞]))
      return false;

    T.Write(w.row, c+"data", start_ts_, w.value);
    T.Write(w.row, c+"lock", start_ts_,
      {primary.row, primary.col}); // The primary’s location.
    return T.Commit();
  }

  bool Commit() {
    // The primary’s location.
    Write primary = writes_[0];
    vector<Write> secondaries(writes_.begin()+1, writes_.end());
    if (!Prewrite(primary, primary))
      return false;
    for (Write w : secondaries)
      if (!Prewrite(w, primary))
        return false;

    int commit_ts = oracle_.GetTimestamp();
    // Commit primary first.
    Write p = primary;
    bigtable::Txn T = bigtable::StartRowTransaction(p.row);
    if (!T.Read(p.row, p.col+"lock", [start_ts_, start_ts_]))
      return false; // aborted while working
    T.Write(p.row, p.col+"write", commit_ts, start_ts_); // Pointer to data written at start_ts_.
    T.Erase(p.row, p.col+"lock", commit_ts);
    if (!T.Commit())
      return false; // commit point
    // Second phase: write out write records for secondary cells.
    for (Write w : secondaries) {
      bigtable::Write(w.row, w.col+"write", commit_ts, start_ts_);
      bigtable::Erase(w.row, w.col+"lock", commit_ts);
    }
    return true;
  }
} // class Transaction
```

### 2.3 时间戳(Timestamps)

timestamp oracle 服务保证 timestamp 严格单调递增。
由于每个事务都取两次时间，必须有很好的伸缩性（scale well）。
oracle 定期分配一批 timestamp 放入内存，并持久化最大值。
重启时跳过最大值，保证时间不会倒退。
为节省 RPC，每个 Percolator worker 仅保留一个挂起的 RPC 来跨事务批量获取 timestamp。
每台 oracle 每秒可提供200万个 timestamp。

### 2.4 通知(Notifications)

##### 观察者(observers)

每个观察者(observer)注册一个函数，绑定一组 column，Percolator 在数据有变化时调用这个函数。
Percolator 应用由一系列观察者构成，每个观察者完成一项任务。
在索引系统中，爬虫抓取文档后首先触发索引文档（解析、提取链接等），接着触发聚类，最后将文档导出。
通知类似于数据库中的触发器(triggers)或事件(events)，但不是原子操作，不保证不变性。

##### 弱通知机制(weaker notification)

每个观察者附带一个 ack column，包含观察者运行的 timestamp，如果两个观察者同时启动，确认时将会发生冲突。
为了识别需要处理的 dity cell，单独维护一个 notify column，事务提交时写入，触发观察者并提交事务后删除。
为了快速扫描，将 notify column 存储在单独的 Bigtable locality group，多个线程同时随机扫描。
线程扫描之前需要从锁服务获取锁，避免扫描到相同区域导致的性能问题。
首次部署时扫描线程会聚集在一起，类似于公交系统，如果发现另一个线程也在扫描同一个区域时，重新选择位置开始扫描。

### 2.5 讨论(Discussion)

相比 MapReduce 需要更多的 RPC 请求，采用如下方式优化性能：

1. 修改 API，read-modify-write 合并
2. 批量处理，合并多个请求
3. 按照临近访问的原则，预取相邻的数据

## 3 评估(Evaluation)

Percolator 的性能介于 DBMS 和 MapReduce 之间

### 3.1 对比MapReduce(Converting from MapReduce)

基于 Percolator 的索引系统(Caffeine)抓取同样数量的文件比 MapReduce 快100倍，使用的资源多2倍。

每次更新数据量少的时候 Percolator 性能更好，数据量大的时候 MapReduce 性能更好。

### 3.2 基准测试(Microbenchmarks)

Percolator 读性能较 MapReduce 略差，写性能相差较多。

   -    | Bigtable | Percolator | Relative
------- | -------- | ---------- | ---------
Read/s  |   15513  |    14590   |   0.94
Write/s |   31003  |     7232   |   0.23

### 3.3 合成负载(Synthetic Workload)

采用 TPC-E 测试 Percolator 的性能

- 不是OLTP系统，不满足延迟目标
- 性能和资源使用的关系在几个数量级基本是线性的，从11个核到15,000个 CPU
- CPU 使用率约为基准系统的 30 倍

## 4 相关工作(Related Work)

- 批处理系统 [MapReduce]、[Hadoop]、[Dryad] 处理小批量的更新效率低
- [MapReduce Online] 通过将处理流程改为流水线(pipeline)来优化性能
- [MapReduce: A major step backwards] 批评 MapReduce 不支持索引
- [Twister]、[logothetis]、[Spark]、[DryadInc] 优化了 MapReduce
- Percolator 支持事务，迭代器和二级索引，但不支持完整的 SQL
- 数据存储类似无共享并行数据库(shared-nothing parallel databases)，通过 RPC 通信
- [snapshot isolation] 通过扩展[MVCC]的两阶段提交实现
- Percolator 通过牺牲并行数据库([Comparison])的一些灵活性和低延迟，为巨大的数据集提供足够可扩展性
- [Dynamo]、[Bigtable]、[PNUTS] 提供高可用的数据，但没有伴随的转换机制
- [Sinfonia]、[Sinfonia2] 扩展了分布式存储系统，提供了强大的一致性
- [CloudTPS] 也在分布式存储系统之上构建符合 ACID 标准的数据存储
- [ElasTraS] 是一个交易数据存储，其架构类似于Percolator

## 5 结论与未来工作(Conclusion and Future Work)

优化分布式系统带来的开销

## 参考资料

1. [Large-scale Incremental Processing Using Distributed Transactions and Notifications]
2. [Google Percolator 的事务模型]
3. [Accela推箱子 分布式系统-分布式事务（P1）]

<!-- links -->

[Large-scale Incremental Processing Using Distributed Transactions and Notifications]: https://ai.google/research/pubs/pub36726.pdf
[Google Percolator 的事务模型]: https://github.com/ngaut/builddatabase/tree/master/percolator
[Accela推箱子 分布式系统-分布式事务（P1）]: https://mp.weixin.qq.com/s?__biz=MzI5Mjk3NDUyNA==&mid=2247483746&idx=1&sn=8c712a0e252c395154857af4aa76c317&chksm=ec787bb1db0ff2a745bc9acb6ac06a0ccbf0ecb063ed0d0be2c8b2392297db182347b2971547&scene=21#wechat_redirect

[MapReduce]: https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf
[Hadoop]: http://hadoop.apache.org/
[Dryad]: https://www.microsoft.com/en-us/research/wp-content/uploads/2007/03/eurosys07.pdf
[MapReduce Online]: http://www.neilconway.org/docs/nsdi2010_hop.pdf
[MapReduce: A major step backwards]: https://pdfs.semanticscholar.org/08d1/2e771d811bcd0d4bc81fa3993563efbaeadb.pdf?_ga=2.16909371.344171820.1564986972-726453249.1563806464
[Twister]: http://www.iterativemapreduce.org/hpdc-camera-ready-submission.pdf
[logothetis]: https://cseweb.ucsd.edu/~dlogothetis/docs/socc10-logothetis.pdf
[Spark]: https://www.usenix.org/legacy/event/hotcloud10/tech/full_papers/Zaharia.pdf
[DryadInc]: https://dl.acm.org/citation.cfm?id=1855554
[snapshot isolation]: https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-95-51.pdf
[MVCC]: https://people.eecs.berkeley.edu/~brewer/cs262/concurrency-distributed-databases.pdf
[Dynamo]: https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
[Bigtable]: https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/bigtable-osdi06.pdf
[PNUTS]: https://pdfs.semanticscholar.org/876b/e80390bcaffb9b910ed05680b2e81a37d64d.pdf
[Comparison]: http://www.science.smith.edu/dftwiki/images/6/6a/ComparisonOfApproachesToLargeScaleDataAnalysis.pdf
[Sinfonia]: https://pdfs.semanticscholar.org/9081/eb3e302dc76957e68b7937ab7d37a83a7d11.pdf?_ga=2.12863285.344171820.1564986972-726453249.1563806464
[Sinfonia2]: http://www.sosp2007.org/papers/sosp064-aguilera.pdf
[CloudTPS]: http://www.globule.org/publi/CSTWAC_ircs53.pdf
[ElasTraS]: https://pdfs.semanticscholar.org/2226/56f0adc032ceb77c93b3ed44b2a78de812ce.pdf?_ga=2.219955355.344171820.1564986972-726453249.1563806464
