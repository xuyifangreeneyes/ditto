volcano 模型：并发隐藏在算子里，shared state is avoided,


pipeline 概念感觉很模糊，jit 是 operator level 吗

vector-at-a-time 就是向量化吧

jit + vector hyper 都有。。。

refregment to avoid skew 所有情况下都适用吗，hash build 就不适用吧

In order to write NUMA-locally and to avoid synchronization 
写 local 可以避免同步？？ again numa mem arch 是怎么样的


To preserve NUMA-locality in further processing stages, the storage area of a particular core is locally allocated on the same socket.

same socket

numa 的内存沟通是走 socket 吗

fig3 似乎没有

global hash table 会被不同 socket 上的 thread 访问

如果放在一个 numa local area 里会有很多竞争吗，交通不均衡？？

具体怎么放呢，lb？？

pipeline 依赖关系 dag，（内存消耗会比 pull 模型多吗，三表 hash join 那里好像没有，都得物化


Preemption of a task occurs at morsel boundaries – thereby elimi- nating potentially costly interrupt mechanisms.

没有 oi 操作，没有阻塞。

pull vs push pull 天然带流控

The dispatcher’s code is then executed by the work- requesting query evaluation thread itself

应该还有一些去 dispatcher 里放任务的工作吧，比如一个 pipeline 的依赖都完成了，要把它放进 dispatcher 里，这种是低频操作？而且 morsel 队列可以由 worker 自己更新。

pipeline 的添加也可以用 worker 完成？yep

内存数据库落盘策略啥的

bushy parallelism
两个 hash table building 不能并行

一个 morsel 大概要执行多久

kill next 粒度控制

false sharing 
mesi 协议

一个集中的数据结构管理线程
好处，自己执行调度规则
坏处，很难写，容易变成瓶颈，

Second, if more than one query is executed concurrently, the pressure on the data structure is further reduced.
减少竞争

hash join
outer side is build side
marked
probe 完再扫一遍，这个也可以 morsel-driven 吗

perfect hash size
collision

tagging is also very beneficial during aggregation when most keys are unique.
agg 的时候怎么用

mmap
hash table 是中间结果，干嘛落盘

numa aware 的技巧是不是跟多节点 shuffle 数据的技巧异曲同工

local join 好像跟它说的 hash join 不太一样，哦 mmap 写的时候大概率是 local 的

2-phase agg 好像 tidb mpp 啊

blu agg
fixed size hash + partition
对 ndv 很大或者很小都考虑了

ndv 很小，直接进 hash table
ndv 很大，建大的 hash table 得不偿失，所以直接 partition。

In main memory, hash-based algorithms are usually faster than sorting [4]. Therefore, we currently do not use sort-based join or aggregation, and only sort to implement the order by or top-k clause.

这跟 ap mpp 很像，全是 hash join，hash agg 基本不考虑 order/sort

medium of medium