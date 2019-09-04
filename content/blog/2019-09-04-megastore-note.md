---
title: "MegaStore笔记"
description: "Megastore 是一个可扩展，高度可用的数据存储，提供事务性 ACID支持，使用 Paxos 在数据中心间进行同步复制"
date: '2019-09-04'
tags:
  - papers
  - google
keywords:
  - Distributed System
  - Database Management
  - Google
---

Megastore 融合了 NoSQL 的扩展性(scalability)和 RDBMS 的便利性(convenience)，
提供强一致性保证(strong consistency guarantees) 和高可用性(high availability)。
在数据的细粒度分区(fine-grained partitions)中提供了完全可序列化的 ACID 语义。
在广域网同步复制(synchronously replicate)每一次写操作，具有合理的延迟(reasonable latency)，
支持在数据中心间无缝故障转移(seamless failover)。

## 1 介绍(INTRODUCTION)

在线业务上云给存储带来的挑战：

- 高可扩展(highly scalable)
- 快速开发(rapid development)
- 低延迟(low latency)
- 数据的一致视图(consistent view of the data)
- 高可用(highly available)

现状：

- RDBMS 扩展性差
- NoSQL 开发困难（有限的 API，松散一致性模型）

Megastore 融合 RDBMS 和 NoSQL 的优势。
通过同步复制(synchronous replication) 达到高可用(high availability)
和数据的一致视图(consistent view of the data)。

- 对数据分区并分别复制，分区内提供 ACID 语义，分区间提供有限的一致性保证
- 提供部分数据库功能，例如二级索引
- 使用 Paxos 复制数据（首创）

## 2 提供可用性和扩展性(TOWARD AVAILABILITY AND SCALE)

设计目标：

- 全球化(global)
- 可靠的(reliable)
- 任意大规模(arbitrarily large in scale)

方法：

- 可用性(availability): 针对长距离优化的同步容错日志复制器(synchronous, fault-tolerant log replicator)
- 扩展性(scale): 数据划分小的数据库，复制日志存储在每个副本的 NoSQL 数据库中

### 2.1 复制(Replication)

为满足云上存储对高可用性的需求，必须在全球各地保存多个副本数据

#### 2.1.1 策略(Strategies)

备选方案：

- 异步主/从(Asynchronous Master/Slave)：可能会丢数据
- 同步主/从(Synchronous Master/Slave)：延迟太高
- 乐观复制(无主)(Optimistic Replication)：不支持事务

#### 2.1.2 进入Paxos(Enter Paxos)

Paxos 是一种经过验证的(proven)、最优的(optimal)、容错(fault-tolerant)的一致性算法(consensus algorithm)。

### 2.2 分区和局部性(Partitioning and Locality)

通过数据分区和本地化为应用提供细粒度(fine-grained)的控制。

#### 2.2.1 实体组(Entity Groups)

数据划分为实体组(entity groups)的集合，实体组内的数据复制到多个数据中心。
多数事务在实体组内，通过单阶段 ACID 事务提交，
跨实体组事务通常通过异步消息传递，少量通过两阶段提交。

{{< figure
    src="/img/blog/megastore-1.png"
    caption="Figure 1: Scalable Replication"
    alt="Figure 1: Scalable Replication"
    width="600" >}}

{{< figure
    src="/img/blog/megastore-2.png"
    caption="Figure 2: Operations Across Entity Groups"
    alt="Figure 2: Operations Across Entity Groups"
    width="700" >}}

#### 2.2.2 选择实体组边界(Selecting Entity Group Boundaries)

- 实体组粒度太细会导致过多的跨组操作
- 单个实体组放入太多不相关数据会导致写入序列化，降低吞吐量

例子：

- Email：按账号划分
- Blogs：按用户、帖子、博文
- Map：没有合适的自然粒度

#### 2.2.3 物理布局(Physical Layout)

使用 BigTable 提供单个数据中心内实现可扩展的容错存储。
通过控制数据的位置来最小化延迟(minimize latency)、最大化吞吐(maximize throughput)。

- 数据放置到离用户近的地方
- 实体组的数据保持连续的行

## 3 MEGASTORE 之旅(A TOUR OF MEGASTORE)

### 3.1 API 设计理念(API Design Philosophy)

不支持 join 查询

- 性能更重要
- 读多写少
- 存储和查询分层很简单(BigTable)

设计数据模型(data model)和模式语言(schema language)，提供细粒度控制。
通过分层布局和声明取代 join 查询，需要时，join 可以在应用层实现（提供 merge 算法）。
还可以通过并行查询实现 outer join。

### 3.2 数据模型(Data Model)

类 SQL 的数据模型

- string、各种 numbers、Protocol Buffers
- required、optional、repeated
- 主键必须唯一
- 两种表：root table、child table
- child table 通过外键关联 root table
- 一个实体组由一个 root table 和若干 child table 组成

```sql
CREATE SCHEMA PhotoApp;

CREATE TABLE User {
  required int64 user_id;
  required string name;
} PRIMARY KEY(user_id), ENTITY GROUP ROOT;

CREATE TABLE Photo {
  required int64 user_id;
  required int32 photo_id;
  required int64 time;
  required string full_url;
  optional string thumbnail_url;
  repeated string tag;
} PRIMARY KEY(user_id, photo_id),
  IN TABLE User,
  ENTITY GROUP KEY(user_id) REFERENCES User;

CREATE LOCAL INDEX PhotosByTime
  ON Photo(user_id, time);

CREATE GLOBAL INDEX PhotosByTag
  ON Photo(tag) STORING (thumbnail_url);
```

#### 3.2.1 key 的预 joining(Pre-Joining with Keys)

不同于传统关系数据库将主键连续排列，Megastore 按实体聚类，将父子关系的表连续排列。
例子中的 `User` 和 `Photo` 共享公共的 `user_id` 作为前缀。两个表放入同一个 Bigtable 中加速查询。
key 可以按升序或降序排列，或者不排序，`SCATTER` 属性为 key 添加一个双字节散列，防止 Bigtable 热点。

#### 3.2.2 索引(Indexes)

实体组内的任何字段都可以添加二级索引，索引分为两类：

- 本地索引(local index)：实体组内，例如`PhotosByTime`
- 全局索引(global index)：跨实体组，例如`PhotosByTag`

##### 3.2.2.1 存储子句(Storing Clause)

通过索引访问实体数据的步骤：

1. 读取索引找到主键
2. 通过主键获取实体组

通过添加 `STORING` 子句索引可以直接归一化部分实体数据，例如 `PhotosByTag` 存储照片缩略图。

##### 3.2.2.2 重复索引(Repeated Indexes)

支持 repeated 字段构建索引，可代替子表，例如 `PhotosByTag`。

##### 3.2.2.3 内联索引(Inline Indexes)

引用父表的主键作为索引的一部分，索引可以作为父表的一个虚拟重复列。例如 `PhotosByTime`。

#### 3.2.3 映射到Bigtable(Mapping to Bigtable)

Bigtable 列名是 Megastore 表名和属性名的串联，不同表的数据映射到相同的 Bigtable 行。

Row key | User name | Photo time | Photo tag     | Photo url
------- | --------- | ---------- | ------------- | ---------
101     | John      |            |               |
101,500 |           | 12:30:01   | Dinner, Paris | ...
101,502 |           | 12:15:22   | Betty, Paris  | ...
102     | Mary      |            |               |

在 root entity 的 Bigtable 行中，存储事务和元数据，包括事务日志。
索引表示为一个 Bigtable 行。

### 3.3 事务和并发控制(Transactions and Concurrency Control)

每个实体组做为一个小型的数据库，提供序列化的 ACID 语义。
利用 Bigtable 可以保存多版本数据实现 MVCC(multiversion concurrency control)
提供流读(current)，快照读(snapshot)和不一致读(inconsistent)。
写入事务始终以流读开始，以确定下一个可用的日志位置。

完整的事务生命周期：

1. 读取(Read)：获取上次提交的事务的时间戳和日志位置
2. 应用程序逻辑(Application logic)：从 Bigtable 读取并将写入收集到一个日志实体中
3. 提交(Commit)：使用 Paxos 达成共识，将日志实体附加到日志中
4. 应用(Apply)：将变量写入 Bigtable 中的实体和索引
5. 清理(Clean up)：删除不再需要的数据

#### 3.3.1 队列(Queues)

队列在实体组之间提供事务性消息传递，用户跨组操作。
声明队列会在每个实体组创建一个收件箱。

#### 3.3.2 两阶段提交(Two-Phase Commit)

支持跨实体组的两阶段提交，但是具有更高的延迟和更大的冲突风险，鼓励使用队列。

### 3.4 其他功能(Other Features)

- 集成 Bigtable 的全文索引
- 备份系统支持定期复制快照和事务日志的增量复制
- 应用程序可以加密数据

## 4 复制(REPLICATION)

复制方案：Paxos 的低延迟实现

### 4.1 概述(Overview)

- 提供数据的单一、一致视图(a single, consistent view of the data)
- 读写可以从任意副本发起，支持 ACID 语义
- 复制通过在每个实体组内同步复制法定数量的事务日志实现
- 写需要通过一轮数据中心间的通信实现，正常的读(healthy-case reads)可以在本地完成
- 流读(Current reads)提供如下保证：读总是观察到最后确认的写；一个写被观察到之后，未来所有的读都会观察到这个写

### 4.2 Paxos的简要总结(Brief Summary of Paxos)

- 使用 Paxos 复制事务日志
- Paxos 需要多轮通信，延迟较大
- 实际的系统通过减少轮次来实现

### 4.3基于主程序的方法(Master Based Approaches)

- master 参与所有写，始终处于最新状态，读 master 的时候不需要网络通信就可以保证一致
- 通过每次提交(accept)时捎带下一次的准备(prepare)来将通信减少到一轮
- 通过批量写来提高吞吐量(throughput)

- 依赖 master 限制了读和写的灵活度，事务必须在 master 的附近的副本处理，避免顺序读延迟的积累
- 潜在的 master 副本必须具备完成完全工作量足够的资源，
- master 故障恢复(failover)需要复杂的状态机(complicated state machine)，一系列定时器必须在服务恢复前消逝
- 很难避免用户可见的运行中断(outages)

### 4.4 Megastore的方法(Megastore’s Approach)

对 Paxos 的优化和创新

#### 4.4.1 快速读(Fast Reads)

- 流读(current reads)可以在任意副本执行，不需要副本间 RPC
- 本地读(local reads)带来更好的利用率、更低的延迟、细粒度(fine-grained)的读故障恢复、更简单的编程体验
- 每个副本的数据中心有一个协调器(Coordinator)，跟踪一组副本观察到所有 Paxos 写的实体组
- 如果副本在 Bigtable 写入失败，则在从该副本的协调器中删除该组的密钥之前，不能将其视为已提交
- 协调器比 BigTable 更可靠、更快速地做出响应

#### 4.4.2 快速写(Fast Writes)

- 采用基于 master 的预准备优化(pre-preparing optimization)来实现一轮写入
- 每个成功的写包含一个隐含的准备消息(implied prepare message)，赋予 master 接收下一条日志位置的权利
- 如果写成功，准备就完成了，下一个写直接进入提交阶段
- 每个日志位置运行一个单独的 Paxos 算法实例
- 每个日志位置的 leader 是由前一个日志位置的共识值选出副本
- leader 仲裁哪个值可以使用 proposal number zero
- 第一个向 leader 提交值的 writer 赢得询问所有副本是否接受那个值作为 proposal number zero
- 其它 writer 退回到两阶段 Paxos
- 大多少应用反复从同一区域提交，在观察者附近选举 leader 可以使用最近的副本

#### 4.4.3 副本类型(Replica Types)

- 完整(full)副本，包含所有实体和索引数据
- 证人(witnesses)副本，只记录日志，不包含实体和索引数据，更低的存储消耗，没有足够的法定人数时使用
- 只读(Read-only)副本，包含完整快照，用于可以忍受不是最新数据的读

### 4.5 架构(Architecture)

- client library：实现 Paxos 算法，选择要读取的副本，捕获滞后的副本
- 每个应用程序都有一个指定的本地副本
- 本地数据中心直接向 Bigtable 提交事务
- 远程数据中心通过无状态的复制服务器(replication servers)操作
- 复制服务器(replication servers)定期扫描不完整的写入，通过 Paxos 提议一个 [no-op] 值，来完成操作

{{< figure
    src="/img/blog/megastore-5.png"
    caption="Figure 5: Megastore Architecture Example"
    alt="Figure 5: Megastore Architecture Example"
    width="600" >}}

### 4.6 数据结构和算法(Data Structures and Algorithms)

#### 4.6.1 复制日志(Replicated Logs)

- 每个副本存储该组已知日志条目的突变和元数据(mutations and metadata)
- 日志条目存储在 Bigtable 中的独立 cell

{{< figure
    src="/img/blog/megastore-6.png"
    caption="Figure 6: Write Ahead Log"
    alt="Figure 6: Write Ahead Log"
    width="600" >}}

#### 4.6.2 读(Reads)

读取和写入之前，必须至少更新一个副本：将先前提交的所有突变复制到该副本，这个过程叫追赶(catchup)。

1. 本地查询(Query Local)：查询本地副本的协调器确定实体组是否是最新的
2. 查找位置(Find Position)：确定最高可能的日志位置，并选择通过该日志位置应用的副本
  - a. 本地读取(Local read)：如果步骤1本地副本是最新的，则从本地读取
  - b. 多数读取(Majority read)：如果本地副本不是最新的（或者1或2a超时），选择大多数副本中已被观察到的最大日志位置
3. 追赶(Catchup)：选中副本后，追赶到最大已知日志位置
  - a. 对于所选副本不知道各个日志位置的情况，从另一个副本读取；
  对于任何没有已知提交值的日志位置，触发 Paxos 提交一个 no-op 写，Paxos 会驱动大多数副本聚焦到一个值
  - b. 顺序应用所有未应用的日志位置的共识值，将副本推进到分布式共识状态
4. 验证(Validate)：如果本地副本不是最新的，向协调器发送一条验证消息，声明（实体组，副本）反映了所有已提交的写入
5. 查询数据(Query Data)：使用所选副本的时间戳读取数据，如果所选副本不可用，选择备用副本，执行追赶并读取。
可以从多个副本透明地组装一个查询

实践中，1和2a并行执行

{{< figure
    src="/img/blog/megastore-7.png"
    caption="Figure 7: Timeline for reads with local replica A"
    alt="Figure 7: Timeline for reads with local replica A"
    width="600" >}}

#### 4.6.3 写(Writes)

- 完成读取算法后，观察下一个未使用的日志位置，最后一次写入的时间戳以及下一个 leader 副本
- 在提交时，对状态的所有未决更改打包并提议，带有时间戳和下一个 leader 提名，作为下一个日志位置的一致值
- 如果此值赢得分布式共识，则将其应用于所有完整副本的状态
- 否则整个事务中止，必须从读取阶段开始重试

- 协调器跟踪其副本中最新的实体组
- 如果副本上未接受写入，则必须从该副本的协调器中删除实体组的密钥，此过程失效
- 在将写入视为已提交并准备应用之前，所有完整副本必须已接受或使其协调器对该实体组无效

写入流程如下：

1. 接受领导者(Accept Leader)：要求 leader 接受该提议编号为 number zero 的值。如果成功，跳至步骤3
2. 准备(Prepare)：在所有复本上运行 Paxos 准备阶段，编号比所有日志位置更大。如果有建议的最大值，使用这个值
3. 接受(Accept)：要求剩余的副本接受该值。如果在大多数副本上失败，则在随机时间后回退到步骤2
4. 无效(Invalidate)：在所有不接受该值的完整副本中使协调器无效
5. 应用(Apply)：将值的变化应用于尽可能多的副本。如果所选值与最初建议的值不同，则返回冲突错误

{{< figure
    src="/img/blog/megastore-8.png"
    caption="Figure 8: Timeline for writes"
    alt="Figure 8: Timeline for writes"
    width="600" >}}

- 传统数据库系统中，提交点与可见点相同
- Megastore 算法中，提交点在步骤3之后，当写入赢得Paxos轮次，但是可见性点在步骤4之后
- 只有在所有完整副本已经接受或者其协调器无效之后才能确认写入并且更改应用
- 在步骤4之前确认违反的一致性保证：在副本中读取其失效的当前读取可能无法观察到已确认的写入

### 4.7 协调器可用性(Coordinator Availability)

- 协调器在每个数据中心运行，并仅保持本地副本的状态
- 写入时，每个完整的副本必须 accept，否则要将协调器置为无效
- 单个副本失败（Bigtable 或协调器）都会导致不可用
- 协调器功能简单，没有外部依赖，没有持久存储，比 Bigtable 稳定，失败概率较低

#### 4.7.1 错误检查(Failure Detection)

- 为了解决网络分区问题，协调器使用带外协议(out-of-band protocol)识别其它协调器的启动、健康状况和是否可达
- 使用 Chubby 来维护可用性，启动时获取锁，系统崩溃后重启进入默认的保守状态，认为所有实体组已过期，需要从其它副本获取日志位置
- writer 通过判断协调器是否持有锁来判断本地数据是否有效
- 协调器不可用时，所有 writer 需要短暂的（几十秒）等待
- 协调器容易受到网络分区的影响，如果协调器持有锁，但是与 proposer 失去了联系，实体组写入会中断，此时需要手动处理

#### 4.7.2 验证竞争(Validation Races)

协调器需要处理各种有效写入和无效写入的竞争问题。

协调器从其它数据中心读取数据会带来一定的开销，但是可以通过以下方式缓解：

- 比 Bigtable 简单得多，依赖更少，具有更高的可用性
- 简单，同质的工作使其容易实现，且可​​预测
- 少量的网络流量使其可以实现更可靠的连接
- 出现问题时管理员可以禁用协调器
- Chubby 锁可以检测大多数网络分区和节点不可用问题

### 4.8 写吞吐量(Write Throughput)

- 多个数据中心同时对同一实体组和日志位置写入会产生冲突，只有一个能成功，其它需要重试
- 通过更精细地分割实体组或确保将副本放置在同一区域来缩小写入吞吐量，从而降低延迟和冲突率
- 使用批量处理可以降低冲突率，提高聚合吞吐量
- 对于必须同时写入的实体组，可以通过协调器变为背靠背的顺序事务来降低冲突

### 4.9 运维问题(Operational Issues)

完整副本不可靠或失去连接会影响性能，可以通过修改路由、禁用协调器、完全禁用副本解决

### 4.10 生成指标(Production Metrics)

- 已在 Google 使用多年，超过100个生产应用
- 机器故障、网络中断、数据中心中断和其他故障很多，但是大多数应用有极高的可用性，5个9(99.999%)
- 平均读延迟几十毫秒（大部分是本地读）
- 平均写延迟100-400毫秒，取决于数据中心的距离，正在希尔的数据大小和完整复制的数量

## 5 经验(EXPERIENCE)

- 伪随机测试框架有助于系统的开发，它探索所有可能的排序空间和模拟节点或线程之间的通信延迟，并能够重现
- 读写性能可以满足应用的需求
- 应用程序需要使用 Megastore 模式语言来建模数据
- 高可用可能会屏蔽故障，导致潜在的问题无法发现
- 流量控制(flow control)算法不考虑慢速的节点，大多数节点按照自己的节奏处理请求
- 预写日志(write-ahead log)的好处是易于集成外部系统，任何幂等操作都可以成为应用日志条目的一个步骤
- 为了复杂查询达到更好的性能，需要检查数据在 BigTable的分布
- 查询速度慢时，需要跟踪 BigTable 来查出速度低于预期的原因

## 6 相关工作(RELATEDWORK)

- NoSQL 牺牲 RDBMS 的属性(事务、模式、查询)实现扩展性(scalability)：[Bigtable]、[Cassandra]、[PNUTS]
- [G-Store] 将事务范围缩小到单一密钥访问的粒度
- [simpledb] 将事务范围扩展到单个表中的多个行
- Megastore 实体组内提供了事务性 ACID 保证，并提供类 SQL 的 schema
- 跨数据中心复制通常使用弱一致性模型
- [Cassandra]、[HBase]、[CouchDB]、[Dynamo] 使用最终一致性模型(eventual consistency model)
- [PNUTS] 使用时间轴一致性模型("timeline" consistency)
- 传统 RDBMS 的同步复制存在性能问题并且难以扩展 [Dangers of Replication]
- 几种解决方案：[Strongly consistent replication]、[Ganymed]、[Determinism]
- 使用 Paxos 的同步复制：[paxos on DHTs]、[Keyspace]
- Megastore 是第一个在数据中心之间实现基于 Paxos 复制的大型存储系统

## 7 总结(CONCLUSION)

Megastore 是一个可扩展，高度可用的数据存储，旨在满足交互式 Internet 服务的存储要求。

> a scalable, highly available datastore designed to meet
> the storage requirements of interactive Internet services

## 参考资料

1. [Megastore: Providing Scalable, Highly Available Storage for Interactive Services]
2. [Megastore 中文版]

<!-- links -->

[Megastore: Providing Scalable, Highly Available Storage for Interactive Services]: http://cidrdb.org/cidr2011/Papers/CIDR11_Paper32.pdf
[Megastore 中文版]: http://duanple.com/?p=147

[Bigtable]: https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/bigtable-osdi06.pdf
[Cassandra]: http://cassandra.apache.org/
[HBase]: http://hbase.apache.org/
[CouchDB]: http://couchdb.apache.org/
[Dynamo]: http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
[PNUTS]: https://pdfs.semanticscholar.org/876b/e80390bcaffb9b910ed05680b2e81a37d64d.pdf
[G-Store]: http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.209.1087&rep=rep1&type=pdf
[simpledb]: https://aws.amazon.com/simpledb/

[Dangers of Replication]: http://db.cs.berkeley.edu/cs286/papers/dangers-sigmod1996.pdf
[Strongly consistent replication]: https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/samehe-icde2010.strongconsistency.pdf
[Ganymed]: https://www.research-collection.ethz.ch/bitstream/handle/20.500.11850/69294/eth-4754-01.pdf
[Determinism]: http://www.cs.umd.edu/~abadi/papers/determinism-vldb10.pdf

[paxos on DHTs]: http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.528.9172&rep=rep1&type=pdf
[Keyspace]: https://arxiv.org/pdf/1209.3913.pdf
[no-op]: http://en.wikipedia.org/wiki/No-op
