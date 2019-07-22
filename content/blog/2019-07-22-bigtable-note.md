---
title: "Big Table笔记"
description: Bigtable 是一个分布式存储系统，用于管理超大规模的结构化数据
date: '2019-07-22'
tags:
  - papers
  - google
keywords:
  - Distributed System
  - LSM-Tree
  - Column storage
  - Google
---

Bigtable 是一个分布式存储系统，用于管理超大规模的结构化数据

> Bigtable is a distributed storage system for managing structured data that is designed 
to scale to a very large size: petabytes of data across thousands of commodity servers

## 1 介绍(Introduction)

- PB级数据
- 数千台机器
- 适用性广泛(wide applicability)
- 可扩展(scalability)
- 高性能(high performance)
- 高可用(high availability)
- 类似数据库
- 提供简单的数据模型
- 数据都视为字符串

## 2 数据模型(Data Model)

- Bigtable 是一个稀疏的、分布式的、持久化存储的多维度排序 Map
- Map 通过行、列、时间戳索引
- Map 中的每个 value 都是一个未解析的 byte 数组

> A Bigtable is a sparse, distributed, persistent multi- dimensional sorted map.
> The map is indexed by a row key, column key, and a timestamp;
> each value in the map is an uninterpreted array of bytes.

```
(row:string, column:string, time:int64) -> string
```

##### 行(Rows)

- row key 是任意字符串，最大64kb
- 的读写都是原子操作
- 按字典序组织数据
- 每个 row 都可以动态分区，每个分区叫一个 tablet

##### 列族(Column Families)

- Column keys 按 Column Families 进行分组
- 访问控制、磁盘和内存的使用统计的基本单位
- 数据通常是同一类型
- 使用之前必须先创建
- 不能太多，最多几百个，Column 可以无限多
- column key 命名规则 `family:qualifier`

##### 时间戳(Timestamps)

- 每个 cell 可以保存多个版本的数据，通过 Timestamps 区分
- 64 bit 整数，毫秒/用户定义
- 倒序排列
- 过期策略：最近 N 个版本/最近 N 天数据

## 3 API

- 建立/删除 tables、column families
- 修改 cluster、table、column family metadata
- single-row 事务，原子的 read-modify-write，不支持 across row keys 事务
- 允许 cell 做为计数器
- 允许在服务器地址空间内执行脚本 [Sawzall]

```
// Open the table
Table *T = OpenOrDie("/bigtable/web/webtable");
// Write a new anchor and delete an old anchor
RowMutation r1(T, "com.cnn.www");
r1.Set("anchor:www.c-span.org", "CNN");
r1.Delete("anchor:www.abc.com");
Operation op;
Apply(&op, &r1);
```

```
Scanner scanner(T);
ScanStream *stream;
stream = scanner.FetchColumnFamily("anchor");
stream->SetReturnAllVersions();
scanner.Lookup("com.cnn.www");
for (; !stream->Done(); stream->Next()) {
    printf("%s %s %lld %s\n",
        scanner.RowName(),
        stream->ColumnName(),
        stream->MicroTimestamp(),
        stream->Value());
}
```

## 4 构件(Building Blocks)

- [GFS] 存储日志和数据
- 内部存储数据的文件是 SSTable 格式，持久化的、排序的、不可更改的 Map
- SSTable 使用块索引定位数据，索引加载到内存
- 高可用的、序列化的分布式锁服务组件 [Chubby]

Chubby 的作用

1. 确保在任何给定的时间内最多只有一个活动的 Master
2. 存储数据的 bootstrap 位置
3. 查找 tablet server，并在失效时进行善后
4. 存储 scheme 信息(每张表的 Column family)
5. 存储访问控制列表

## 5 实现(Implementation)

- 三个组件：client library、一个 master server，多个 tablet server
- 每个 table 包含一个 tablet 集合
- 每个 tablet 包含某个范围内 row 的所有数据

master server

- 分配 tablets 到 tablet server
- 检测 tablet server 的增减
- 平衡 tablet server 的负载
- GFS 文件的 gc
- 处理 schema 变更（table、column famliy）

tablet server

- 管理一组 tablet（10 - 1000）
- 处理读写请求（client 不经过 master，直接与 tablet 通信）
- 分割过大的 tablet(100 - 200MB)

### 5.1 Tablet 位置(Tablet Location)

三级 B+ 树保存 tablet location

- root tablet 的位置存储在 Chubby
- root tablet 存储 METADATA table 的所有 tablet 位置
- 每个 METADATA tablet 存储一组 user tablet 位置
- root tablet 是 METADATA table 的第一个 tablet，永远不会分裂
- METADATA table 的每一个 row key（由 tablet 标识和它的 end row 组成）存储一个 tablet 的位置存储在
- client 缓存 tablet 的位置，过期时从头开始遍历，最多需要读6次
- 二级信息（log，用于调试和性能分析）也存储在 METADATA table

### 5.2 Tablet 分配(Tablet Assignment)

- 每个 tablet 一次只会分配给一个 tablet server
- Master 保存所有活着的 tablet server 行踪，分配 tablet 给有足够空间的 tablet server
- Chubby 跟踪 tablet server 的服务状态
- tablet server 启动时创建并获取一个在 Chubby 目录上唯一命名的文件的独占锁
- Master 监控这个目录来发现 tablet server

Master 启动后执行的步骤

1. 在 Chubby 获取唯一的 master 锁，防止多个 master 实例
2. 扫描 Chubby server 目录找到所有活着的 server
3. 与活着的 server 通信发现分配了哪些 tablet
4. 扫描 METADATA table 找到 tablet 集合，有未分配的就分配给 tablet server

### 5.3 Tablet 服务(Tablet Serving)

数据持久化到 GFS 中

写流程

1. 检查格式合法、client 权限（通过 Chubby）
2. 写入 commit log
3. 插入 memtable

读流程

1. 检查格式合法、client 权限
2. 在 memtable 和 SSTable 的视图上合并

### 5.4 空间收缩(Compactions)

minor Compation：memtable 增长到一个阈值时被冻结，转化成 SSTable 写入 GFS，创建新的 memtable

- 降低内存使用
- 降低 server 挂掉时需要从日志读取数据的大小

merging compaction：合并多个 SSTable 和 memtable 形成一个新的 SSTable


## 6 优化(Refinements)
##### 局部性群组(Locality groups)

- 将多个 column famliy 组合成一个 locality group，生成单独的 SSTable，提高读取效率
- 频繁访问的 locality group 放入内存（METADATA）

##### 压缩(Compression)

- client 指定一个 locality group 的 SSTable 是否压缩、压缩格式
- 通常采用两遍压缩，第一遍 Bentley and McIlroy’s、第二遍快速压缩算法

##### 缓存提高读性能(Caching for read performance)

二级缓存策略

- 一级扫描缓存：Tablet server 通过 SSTable 接口获取的 key-value 对
- 二级 Block 缓存：从 GFS 读取的 SSTable 的 Block

##### 布隆过滤(Bloom filters)

- 使用 Bloom filter 查询一个 SSTable 是否包含了特定 row 和 column 的数据
- 使用少量内存显著减少磁盘访问次数

##### 提交日志的实现(Commit-log implementation)

- 如果每个 Tablet 的 commit-log 单独存储会产生大量的文件
- 每个 Tablet server 一个 commit-log 文件，混合多个 Tablet 的日志记录
- 恢复是时候首先按关键字（table、row name、log sequence number）排序，同一个 Tablet 的日志连续存放在一起
- 日志分割成 64MB 的段，不同的 tablet server 对段并行排序，由 master server 协同处理

##### 提升tablet的恢复速度(Speeding up tablet recovery)

- tablet 迁移时，进行两次 Minor Compaction，减少恢复时间

##### 利用不变性(Exploiting immutability)

利用 SSTable 不变对系统进行简化

- 从 SSTable 读取数据时，不必对访问操作进行同步
- 把永久删除转变为对 SSTable 的垃圾收集
- 分割 Tablet 操作非常便捷

## 7 性能评估(Performance Evaluation)

- 测试性能和可扩展性，建立包括 N 台 tablet server 的 Bigtable 集群
- tablet server、master、test client、GFS 运行在同一组机器
- 测试随机读、从内存随机读、随机写、序列读、序列写、扫描

##### 单个 Tablet 服务器的性能(Single tablet-server performance)

- 随机读的性能比其它操作慢一个数量级
- 内存中随机读快很多
- 随机和序列写操作的性能比随机读要好些
- 序列读的性能好于随机读
- 扫描的性能更高

##### 性能提升(Scaling)

- tablet server 从1台增加到500台，吞吐量增长100倍
- 多台服务器负载不均衡会造成性能下降
- 随机读的性能随 tablet server 增加	提升幅度最小

## 8 实际应用(Real Applications)

2006 年 8 月 google 有 388 个 bigtable 集群

### 8.1 Google Analytics

- Raw click table(200TB)，每行一个用户的会话，按时间顺序存储
- Summary table(20TB)，每个站点各种预定义信息的汇总，周期运行 MapReduce 任务生成

### 8.2 Google Earth

- 存储原始图像的表(70TB)，每一行代表一个独立的地理区域
- 索引GFS数据的表(500GB)，每秒几万个查询请求，上百个 tablet server，in-memory column family

### 8.3 个性化搜索(Personalized Search)

- 每个用户 id 和一个 column name 绑定，一个单独的 column family 被用来存储各种类型的行为

## 9 经验教训(Lessons)

很多类型的错误会导致大型分布式系统受损

- 网络中断
- 很多分布式协议中设想的 fail-stop
- 内存数据损坏
- 时钟偏差
- 机器挂起
- 扩展的和非对称的网络分区
- 其它系统的 Bug （Chubby）
- GFS 配额溢出
- 计划内和计划外的硬件维护

在彻底了解一个新特性会被如何使用之后，再决定是否添加这个新特性

- 只实现了单行事务

系统级的监控对 Bigtable 非常重要

- 扩展 RPC，详细记录 RPC 调用的重要操作
- 每个 Bigtable 集群都在 Chubby 中注册，跟踪集群状态、监控大小

简单的设计有价值

- 维护和调试带来巨大好处
- 重新设计简单的协议

## 10 相关工作(Related Work)

- [Boxwood]的组件与 Chubby、GFS、Bigtable 类似，但是更底层
- 广域网上的分布式数据存储或高级服务（分布式 Hash 表）：[CAN]、[Chord]、[Tapestry]、[Pastry]
- 分布式 B-Tree 、分布式 Hash 表提供的 Key-value pair 方式的模型有很大的局限性
- Bigtable 模型提供的组件比简单的 Key-value pair 丰富的多，它支持稀疏的、半结构化的数据
- 能存储海量数据的并行数据库系统：Oracle 的 [RAC]、IBM 的 [DB2 Parallel Edition]
- 基于列存储方案在压缩和磁盘读取方面有优势：[C-Store]、[Sybase IQ]、[SenSage]、[KDB+]、[MonetDB/X100]、[Daytona]、[Ailamaki]
- memtable 和 SSTable 存储对表的更新的方法与[LSM-tree]类似

## 11 结论(Conclusions)

- 一个分布式的结构化数据存储系统，7人年设计和实现
- 用户对高性能、高可用性很满意
- 编程接口并不常见
- 新特性：二级索引、多 Master 节点、跨数据中心复制的基础构件
- 自己设计的优势：系统极具灵活性、问题可以快速解决

## 参考资料

1. [Bigtable: A Distributed Storage System for Structured Data]
2. [Bigtable中文版]

<!-- links -->

[Bigtable: A Distributed Storage System for Structured Data]: https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/bigtable-osdi06.pdf
[Bigtable中文版]: http://blog.bizcloudsoft.com/wp-content/uploads/Google-Bigtable%E4%B8%AD%E6%96%87%E7%89%88_1.0.pdf

[Sawzall]: https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/sawzall-sciprog.pdf
[GFS]: https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/gfs-sosp2003.pdf
[Chubby]: https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/chubby-osdi06.pdf
[Boxwood]: https://www.usenix.org/legacy/event/osdi04/tech/full_papers/maccormick/maccormick.pdf
[CAN]: http://conferences.sigcomm.org/sigcomm/2001/p13-ratnasamy.pdf
[Chord]: https://pdos.csail.mit.edu/papers/chord:sigcomm01/chord_sigcomm.pdf
[Tapestry]: https://people.eecs.berkeley.edu/~adj/publications/paper-files/CSD-01-1141.pdf
[Pastry]: https://www.cs.rice.edu/~druschel/publications/Pastry.pdf
[RAC]: https://www.oracle.com/database/technologies/rac.html
[DB2 Parallel Edition]: https://pdfs.semanticscholar.org/9c08/4a56c1c625da116eca474208d9780539ceff.pdf
[C-Store]: http://db.csail.mit.edu/projects/cstore/vldb.pdf
[Sybase IQ]: https://dl.acm.org/citation.cfm?id=223871
[SenSage]: https://en.wikipedia.org/wiki/Sensage
[KDB+]: https://kx.com/products/database.php
[MonetDB/X100]: https://paperhub.s3.amazonaws.com/b451cd304d5194f7ee75fe7b6e034bc2.pdf
[Daytona]: http://www09.sigmod.org/sigmod/sigmod99/eproceedings/papers/greer.pdf
[Ailamaki]: http://research.cs.wisc.edu/multifacet/papers/vldb01_pax.pdf
[LSM-tree]: https://www.cs.umb.edu/~poneil/lsmtree.pdf
