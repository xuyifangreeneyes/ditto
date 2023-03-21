# 五彩极光欧若拉 

> 神秘北极圈 阿拉斯加的山巅

## Overview

Aurora 是亚麻的一款云原生的 OLTP 关系型数据库。2015年7月 Aurora 完成 GA（General Available）。2017年亚麻发表论文 Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases 描述了一些设计原理和实现细节。 

Aurora 把传统单机数据库（比如 MySQL）拆成了计算和存储两个部分。计算层都是无状态的节点（挂了重新拉一个起来，scale up/down 也很方便），而且可以复用 MySQL 的大部分代码逻辑（优化器、执行器、事务等等）。存储层从本地磁盘换成了一个分布式、高可用、可扩展的存储服务（一份数据复制给多个存储节点，提供 higher durability, availability, IOPS and reduction of jitter)，分离出分储层还能带来更好的 crash recovery 和 backup/restore。计算层通过网络往存储层写 log（而不是 data），存储层把 log 转化为 data。作者认为实现高吞吐的瓶颈在于网络，而写 log 的设计能大幅降低网络开销。

![Move logging and storage off the database engine](decouple.png)

## Quorum Model

Aurora 的存储层用了 quorum-based voting protocol 来面对各种失败（网络断了，磁盘坏了，机房被淹）。假设有 V 个节点，读操作需要收到 V_r 个节点的回复才能完全，写操作要收到 V_w 个节点的回复才能完成。为了保证读操作能读到最新的写入，必须有 V_r + V_w > V。为了避免写冲突，必须有 V_w > V/2。如果要容忍一个节点的失败，一般配置是 V = 3，V_w = 2，V_r = 2。但 Aurora 觉得这种程度的 durability 和 availability 不行。假设一共有 3 个 AZ（Availability Zone），每个 AZ 一个节点，然后每个 AZ 会有 continuous low level background noise of node, disk and network path failures（我猜大概就是磁盘网络偶尔会抽风出点毛病）。假设一个 AZ 着火或者被淹了，剩下两个 AZ 中的某一个偶发抽风一下，服务就不可用了（我们不知道剩下那个节点是否有最新的数据）。Aurora 想要容忍 AZ + 1 个节点的失败（某个 AZ 里的节点全挂了加上另一个 AZ 中一个节点挂了），就设计成如下的配置：3 个 AZ，每个 AZ 两个节点的，V = 6，V_w = 4，V_r = 3。

1. 如果一个 AZ 挂了加上另外一个 AZ 中的一个节点挂了（一共 3 个节点挂了），这时候没法写，但可以读，通过 quorum read 可以确认最新的数据，然后把最新数据拷贝到一个新的节点来让写恢复。
2. 如果 2 个节点挂了（一个 AZ 挂了或者属于不同 AZ 的 2 个节点挂了），读写都还可以继续进行。


Note：
1. 不太清楚跨 AZ 流量贵吗。然后 latency 会高吗。写操作要收到 4 个回复，肯定有跨 AZ 的。
2. 不太说得上来 quorum 和 paxos/raft 的本质区别在哪里，设计存储产品的时候该用哪一个。quorum 只保证了读可见最新的写、没有写岔（不同节点对最新值达成共识），然后 quorum 是无主的。paxos/raft 能保证操作有一个全局的顺序。感觉 TiKV 和 Cassandra/Dynamo 是差不多的产品（存疑），但前者用了 raft，后两者用了 quorum，在产品形态上会带来什么差异吗。
3. a failure in durability can be modeled as a long-lasting availability event, and an availability event can be modeled as a long-lasting performance variation。没理解这句啥意思。

当一个节点出现故障的时候，我们希望尽快修复它，避免屋漏偏逢连夜雨。但如果一个节点的数据量很大，修起来就比较慢。所以 Aurora 也做了分片的操作。数据会切分成 10GB 大小的 segment，然后每个 PG（Protection Group）由 6 个 segment 组成。10GB 的 segment 出故障的时候修起来快多了（10Gbps 网络下只需要 10s 左右）。感觉这里的修法就是卸载坏的盘，然后从其他副本把最新数据拷贝到新盘上然后挂载到这个节点上。

在这种设计下，有个节点暂时慢了甚至不可用，对整体没太啥影响，Aurora 利用这点来做一些运维操作。比如某个节点过热要换掉、要升级或者打补丁，都可以直接操作这个节点，暂时不可用没啥问题。

## Log Is Database

首先，作者展示了 MySQL 要达到 AZ + 1 的容错需要配置成什么样（具体细节可以参考 [mit6.824 aurora notes](https://pdos.csail.mit.edu/6.824/notes/l-aurora.txt)）。

![Network IO in mirrored MySQL](mysql.png)

但这里我有疑惑：

1. 感觉这图上很多箭头可以优化掉。至少两个 instance 之间同步只传 binlog 就行吧。
2. 文章里说 steps 1, 3, and 5 are sequential and synchronous 不太理解。[mit6.824 aurora notes](https://pdos.csail.mit.edu/6.824/notes/l-aurora.txt) 的说法似乎更合理一些：要达到 AZ + 1 的容错，DB write has to wait for all four EBS replicas to respond。

在 MySQL 里，写路径是先写 redo log，然后再 apply redo log 去写对应的 page（如果 page 在 buffer cache 里直接写，没在的话先读到 cache 再写，然后会把 dirty page 再 flush 到磁盘）。这个过程只有写 redo log 是同步的，apply redo log 写对应的 page 可以异步做。Aurora 改造了这个流程。计算层只要通过网络把 redo log 写到存储层就可以回应客户端，然后由存储层担任 log applicator 的角色。Apply log（也就是 page materialization）可以在存储层后台持续异步执行，甚至在没有读请求的时候可以不 page materialization。然后作者强调了在 crash recovery 的时候，MySQL 是从 checkpoint 恢复，要 apply 所有之后的 redo log（一般数量比较多，因为 checkpoint 是比较重的操作，不太会很频繁地做），而 Aurora 故障恢复不需要 checkpoint，每个 page 只要 apply 跟它有关的 redo log 就行（一般数量比较少），不同 page 的 materialization 可以让多个存储节点并发进行。 

Note：
1. MySQL 和 Aurora 的 apply redo log 步骤都是异步的，但 Aurora 把 apply redo log 推到存储层。
2. Page materialization 确实比 checkpoint 在 crash recovery 上更有优势。但单机 MySQL 应该也可以改造成 page materialization 的 crash recovery 吧。


https://pdos.csail.mit.edu/6.824/notes/l-aurora.txt

跨 AZ 流量，多贵，latency 高？

6/4/4 用户需要这么高的可用性吗

database-server on ec2 防止脑裂，etcd 选主就行，或者 lease 选主
A murky aspect of Aurora is how it deals with failure or partition of
the main database server. Somehow the system has to cut over to a
different server, while ruling out the possibility of split brain. The
paper doesn't say how that works. Spanner's use of Paxos, and avoidance
of anything like a single DB server, makes Spanner a little more clearly
able to avoid split brain while automatically recovering from failure.


quorum 和 raft/paxos 的不同
 
quorum 

read after write
only for kv 

6/4/4

cannot vote on content! 啥意思

调节读写比？

version 号

dead or slow server，

raft/paxos

gossip catch up

线性一致性

事务是构建在 kv 之上，
tikv 能用 quorum 吗，quorum 能分片吗 cassendra/dynamo 对等的产品？


Note
  Raft uses quorums too
    Leader can't commit unless majority of peers receive
    R/W overlap guarantees new Raft leader sees every committed write
  Raft is more capable:         
    Raft handles complex operations, b/c it orders operations.
    Raft changes leaders automatically w/o split brain.


aurora 能不能支持多写？为什么不能？
A huge difference is that Aurora is a single-writer system -- only the
one database on one machine does any writing. This allows a number of
simplifications, for example there's only one source of log sequence
numbers. In contrast, Spanner can support multiple writers (sources of
transactions); it has a distributed transaction and locking system
(involving two-phase commit).
LSN 有一个全局的 ground truth 可以吗，这样就无法避免 2pc 吗
多写就很复杂了。


没有消融实验

log only vs full data

4/4 vs 6/4

faster network RDMA
PolarDB


Any read request for a data page may require some redo records to be applied if the page is not current. 
这个单机的 MySQL 不能做吗

no buffer pool in storage node --> replica node to read so no buffer pool is needed.

PG 是物理分片吗 yep

http://nil.lcs.mit.edu/6.824/2020/papers/aurora-faq.txt

undo local disk 高可用，没必要？


write path 

commit 异步处理，那么 commit 需要一条 redo log 吗，mysql 里有吗

As a list of relevant pending log entries per data page.
没见过存储结构的时候不知道 redo log 属于哪个 data page？

apply log 的时候可以超过 VDL 吗，crash recovery 的时候需要 undo log 取消？不如不超过 VDL。。。会被 PGMRPL 卡住，但没有读的时候应该也不会超过 VDL

  DB finds highest log record such that log is complete up to that point.
    Log record exists if it's on any reachable storage server.
    The VCL (Volume Complete LSN) of Section 4.1.
  DB tells storage servers to delete all log entries after the VCL.
  DB issues compensating "undo" log entries for transactions that
    started before VCL but didn't commit. 

VCL or VDL

read path
事务性 read committed 依赖于上层
读 VDL page 不能保证 read committed
replica 上的读写可见性？

  DB server reads from any storage server whose SCL >= highest committed LSN. highest committed LSN 跟 read-point 不太一样，约束更强，能保证 read committed
    That storage server may need to apply some log records to the data page.


找到的满足SCL>=read-point的segment有hole？可能吗？不可能，看SCL的定义。需要gossip补一下以后才行？可能的，有可能所有的 SCL 都 < read-point，这时候要 gossip 补一下才行。

为啥会有 cache page LSN >= VDL 

6 份能优化吗，每个 AZ 一份写 log apply log 另一份只写 log，定期从兄弟那里拿 checkpoint 恢复会慢一点吗


sysbench
MySQL 的五倍性能

PostgreSQL 的三倍性能

重启时间缩短到 60 秒以下。

why 15 replicas
支持最多 15 个低延迟只读副本

cache miss 的时候大概率读同 AZ 的 storage node，小概率读跨 AZ 的 storage node?

recovery

aurora 的 recovery 感觉 mysql 也能做，为啥不能做

并行恢复

aurora 比 mysql 省成本

很云原生，都不能 op 部署

write on writer and read on replica 能符合事务吗

share-nothing
支持多写，更高的可扩展性
从 NoSQL 运动中回归到关系型数据库 
兼容性很痛苦

share-everything
单机到分布式的一次扩展
easy to extend from mysql/pg
不支持多写

## Reference

1. https://pdos.csail.mit.edu/6.824/notes/l-aurora.txt
2. http://nil.lcs.mit.edu/6.824/2020/papers/aurora-faq.txt