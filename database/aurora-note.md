# My notes on some papers and textbooks

a failure in durability can be modeled as a long- lasting availability event, and an availability event can be modeled as a long-lasting performance variation 
这句啥意思？？

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

read path
事务性 read committed 依赖于上层
读 VDL page 不能保证 read committed
replica 上的读写可见性？

找到的满足SCL>=read-point的segment有hole？可能吗？需要gossip补一下以后才行？