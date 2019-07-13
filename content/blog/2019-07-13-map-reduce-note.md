---
title: "Map Reduce笔记"
date: '2019-07-13'
tags:
  - papers
  - google
---

## 1 介绍(Introduction)

处理原始数据

- 文档抓取
- Web 请求日志

处理衍生数据

- 倒排索引
- Web 文档的图结构(graph structure) 表示
- 每台主机爬虫抓取数量汇总
- 每天被请求最多的的查询的集合

难点

- 并行计算
- 分发数据
- 处理错误 

大多数运算包含相同的操作

1. 输入数据应用 Map 得到一个 key/value pair 集合
2. 在相同 key 值的 value 上应用 Reduce 操作，合并数据，得到结果

MapReduce 框架模型

- 处理并行计算、容错、数据分布、负载均衡
- 通过简单的接口来实现自动的并行化和大规模的分布式计算

## 2 编程模型(Programming Model)

利用一个输入 key/value pair 集合来产生一个输出的 key/value pair 集合

- 用户自定义的 Map 函数接收 key/value pair，输出 key/value pair 集合
- MapReduce 库把所有相同 key 的 value 集合后传递给 Reduce 函数
- 用户自定义的 Reduce 函数合并 value 值，形成较小的 value 值集合

### 2.1 例子(Example)

计算一个大的文档集合中每个单词出现的次数

```
map(String key, String value):
    // key: document name
    // value: document contents for each word w in value:
    for each word w in value:
        EmitIntermediate(w, "1");

reduce(String key, Iterator values):
    // key: a word
    // values: a list of counts
    int result = 0;
    for each v in values:
        result += ParseInt(v);
    Emit(AsString(result));
```

### 2.2 类型(Types)

Map 和 Reduce 函数的类型是相关联的

```
map    (k1,v1)       -> list(k2,v2)
reduce (k2,list(v2)) -> list(v2)
```

### 2.3 更多的例子(More Examples)

##### 分布式 Grep(Distributed Grep)

- Map 输出匹配某个模式的一行
- Reduce 把中间数据复制到输出（恒等函数）

##### 计算 URL 访问频率(Count of URL Access Frequency)

- Map 处理日志中 web 页面请求的记录，输出(URL, 1)
- Reduce 把相同 URL 的 value 值累加起来，产生(URL, total count)

##### 反转 Web-Link 图(Reverse Web-Link Graph)

- Map 在 source 中搜索所有的 target，输出 (target,source)
- Reduce 把给定(target)的链接组合成一个列表，输出 (target,list(source))

##### 每个 Host 的检索词向量 (Term-Vector per Host)

- Map 为每一个输入文档输出(hostname, term vector)，host来自文档的 URL
- Reduce 按 word 累加 term vector，丢弃低频 term vector

##### 倒排索引(Inverted Index)

- Map 分析每个文档，输出一个 (word, document ID) 列表
- Reduce 排序 document ID，输出 (word, document ID)

##### 分布式排序(Distributed Sort)

- Map 从每个记录提取 key，输出 (key,record)
- Reduce 函数不改变任何的值。这个运算依赖分区机制(4.1)和排序属性(4.2)

## 3 实现(Implementation)

Google 内部广泛使用的运行环境

1. X86、Linux、双 CPU、2-4G 内存
2. 百兆或千兆带宽
3. 上千台机器，故障是常态
4. IDE 硬盘，GFS管理数据
5. 用户提交 job 给调度系统，每个 job 包含一系列 task，调度系统将 task 调度到集群中多台可用的机器

### 3.1 执行概况(Execution Overview)

MapReduce 执行流程

1. 将文件分为 M 个数据片段，每个 16MB - 64MB(可配置)，在集群中创建大量副本
2. master 分配 map task 或 reduce task 给一个空闲的 worker
3. map worker 读取输入数据，解析出 key/value pair，传递给用户的 Map 函数，输出中间 key/value pair，存储在内存中
4. key/value pair 通过分区函数分为 R 个区域，周期性的写入到本地磁盘，把位置回传给 master
5. reduce worker 通过 RPC 读取数据，通过对 key 排序，使相同的 key 聚合到一起（数据太大需要外部排序）
6. reduce worker 遍历排序后的数据，将每一个 key 和相关的 value 集合传递给用户的 Reduce 函数，输出追加到所属分区的输出文件
7. 所有 Map 和 Reduce task 完成后，调用返回

### 3.2 Master 数据结构(Master Data Structures)

- Map 和 Reduce task 的状态（空闲、工作中、完成）
- worker 机器的标识
- Map task 产生的 R 个中间文件存储区域的大小和位置

### 3.3 容错(Fault Tolerance)

##### Worker 故障(Worker Failure)

- master 周期性的 ping 每个 worker，约定时间内没返回标记为失效，task 标记为空闲，等待重新调度
- map task 的输出存储在本地，需要重新执行，reduce task 存储在 GFS，不需要重新执行
- map task 重新执行会通知所有 reduce worker，重新读取数据
- 可以处理大规模 worker 失效的情况

##### Master 故障(Master Failure)

- 周期性的将数据写入磁盘，记录 checkpoint，新进程通过 checkpoint 继续执行
- 只有一个 master 进程，恢复麻烦，master 失效直接中止 MapReduce

##### 在失效方面的处理机制(Semantics in the Presence of Failures)

- Map 和 Reduce 都是确定性函数时，在任何情况下的输出都和没有出现错误、顺序执行产生的输出相同
- Map 和 Reduce task 的输出是原子提交

### 3.4 存储位置(Locality)

尽量将 task 调度到包含输入数据(GFS)的机器上执行，节约网络带宽

### 3.5 任务粒度(Task Granularity)

- Map 拆分为 M 个片段，Reduce 拆分为 R 个片段
- master 执行 O(M+R) 次调度，内存中保存 O(M\*R)个状态
- R 由用户指定，选择合适的 M，使每个独立 task 处理 16-64MB 数据
- R 值设置为想使用的机器数量的小倍数
- 通常的 MapReduce 比例：M=200000，R=5000，2000台机器

### 3.6 备用任务(Backup Tasks)

某一台机器执行慢，导致总时间超时

- 硬盘出问题，导致读取慢
- 与其它 task 竞争 CPU、内存、本地磁盘、带宽
- 初始化代码有 bug，导致关闭缓存

调优机制：当 MapReduce 操作接近完成时，master 调用 backup task 处理 in-progress task

## 4 技巧(Refinements)

### 4.1 分区函数(Partitioning Function)

- 默认分区函数使用 hash(key) mod R
- 用户可自定义分区函数

### 4.2 顺序保证(Ordering Guarantees)

给定分区中，中间 key/value pair 按照 key 值增量顺序处理

### 4.3 Combiner 函数(Combiner Function)

允许用户指定 combiner 函数，合并中间记录，减少重复数据的网络传输

### 4.4 输入和输出的类型(Input and Output Types)

支持不同的数据输入/输出方式

- 文本
- 数据库
- 内存中的数据结构
- 实现 Reader 接口支持新的输入类型

### 4.5 副作用(Side-effects)

某些情况增加辅助的输出文件，输出全部数据后，使用系统级的原子 rename 这些文件

### 4.6 跳过损坏的记录(Skipping Bad Records)

用户程序 bug 导致系统 crash，worker 向 master 发送记录序号，多次发生后，master 可标记跳过这条记录

### 4.7 本地执行(Local Execution)

MapReduce 库的本地版本简化调试、profile、本地测试

### 4.8 状态信息(Status Information)

master 内嵌 HTTP 服务显示状态信息，展示执行进度

### 4.9 计数器(Counters)

- 使用计数器统计不同事件发生的次数（已经处理了多少单次、已经索引了多少篇文档）
- 计数器周期性的从 worker 汇总到 master（附加在 ping 应答中）

## 5 性能(Performance)

两个典型的例子

- 对数据格式进行转换：对 1T 数据排序
- 从海量数据中抽取感兴趣的数据：1T 数据中进行模式匹配

### 5.1 集群配置(Cluster Configuration)

1800 台机器的集群

- 2个2G主频、支持超线程的 Intel Xeon CPU
- 4G物理内存
- 2个160G的IDE硬盘
- 一个千兆网卡

### 5.2 Grep(Grep)

- 扫描10的10次方个由100字节组成的记录 1TB
- 输入拆分为64MB的块，M=15000
- 输出一个文件，R=1
- 耗时150秒，包括1分钟的启动时间，峰值速度30GB/s

### 5.3 排序(Sort)

- 处理10的10次方个由100字节组成的记录 1TB
- 排序结果输出到两路复制的 GFS 2TB
- 输入数据读取速度最快（本地读取）
- 排序速度比输出速度快（输出数据写了两份）

### 5.4 高效的备用任务(Effect of Backup Tasks)

如果没有 backup task，需要多执行300秒，时间增加44%

### 5.5 失效的机器(Machine Failures)

kill 掉1746个进程中的200个，比正常执行多了5%

## 6 经验(Experience)

MapReduce 广泛应用于 Google

- 大规模机器学习
- Google news 和 Froogle
- 公共查询产品的报告中抽取数据
- 大量新应用和新产品中提取有用信息
- 大规模图形计算

应用数：从 2003 年的 0 个增长到 2004 年 9 月 900 个

### 6.1 大规模索引(Large-Scale Indexing)

最成功的应用：重写 Google 网络索引系统

处理20TB保存在 GFS 中的原始内容，经过5到10次 MapReduce 来建立索引

## 7 相关工作(Related Work)

- 并行计算的经典模型[scan_primitive]、[SEP_scan]、[PPC]
- [BSP]、[MPI]提供更高级别的并行处理抽象
- [Diamond]、[active disks]提供数据本地化策略的灵感
- [Charlotte System]提出 eager 调度机制，类似备用任务机制
- [Condor]提供集群管理系统的理念
- [Now-Sort]提供类似的排序机制
- [River]提供一个编程模型，处理进程通过分布式队列传送数据的方式进行互相通讯
- [BAD-FS]面向广域网，重新执行机制防止数据丢失、数据本地化调度
- [TACC]用于简化构造高可用性网络服务的系统，重新执行机制实现容错

## 参考资料

1. [MapReduce: Simplified Data Processing on Large Clusters]
2. [MapReduce中文版]
3. [6.824-lab-1]

<!-- links -->

[MapReduce: Simplified Data Processing on Large Clusters]: https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/mapreduce-osdi04.pdf
[MapReduce中文版]: http://blog.bizcloudsoft.com/wp-content/uploads/Google-MapReduce%E4%B8%AD%E6%96%87%E7%89%88_1.0.pdf
[6.824-lab-1]: https://pdos.csail.mit.edu/6.824/labs/lab-1.html

[scan_primitive]: https://people.eecs.berkeley.edu/~driscoll/cs267/papers/scan_primitive.pdf
[SEP_scan]: https://link.springer.com/chapter/10.1007%2FBFb0024729
[PPC]: https://dl.acm.org/citation.cfm?id=322232
[BSP]: https://people.eecs.berkeley.edu/~driscoll/cs267/papers/BSP.pdf
[MPI]: http://www.hds.bme.hu/~fhegedus/00%20-%20Numerics/B2015%20Using%20MPI%20-%20Portable%20Parallel%20Programming%20with%20the%20Message-Passing%20Interface.pdf
[Diamond]: https://www.usenix.org/legacy/publications/library/proceedings/fast04/tech/full_papers/huston/huston_html/diamond-html.html
[active disks]: https://ieeexplore.ieee.org/document/928624
[Charlotte System]: https://www.sciencedirect.com/science/article/pii/S0167739X99000096
[Condor]: http://research.cs.wisc.edu/htcondor/doc/condor-practice.pdf
[Now-Sort]: https://people.eecs.berkeley.edu/~culler/papers/p243-arpaci-dusseau.pdf
[River]: http://now.cs.berkeley.edu/files/recent/river.pdf
[BAD-FS]: http://pages.cs.wisc.edu/~remzi/Classes/736/Fall2003/Papers/badfs.pdf
[TACC]: https://people.eecs.berkeley.edu/~brewer/papers/TACC-sosp.pdf

