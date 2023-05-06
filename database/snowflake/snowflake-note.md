# 瓦街千堆雪

> 我不说话 等待黑暗 泪能落下

## Overview

Snowflake 是一款云原生的数仓产品，最初在2012年底开始开发，在2015年6月发布了第一个 GA（General Available）版本。Snowflake 在 SIGMOD16 上发表了论文 The Snowflake Elastic Data Warehouse，描述了一些设计和实现。数仓是一个古老而成熟的领域，诸如 TeraData、Oracle 等都有很成熟的数仓产品。然而云计算的兴起给这个行业带来了变化。作者表示，传统数仓一般会假设可用资源（机器）是固定的，而云计算的一大特点就是弹性，因此要重新设计数仓的架构来充分利用这种弹性。摘一段原文：

> The shared infrastructure of the cloud promises increased economies of scale, extreme scalability and availability, and a pay-as-you-go cost model that adapts to unpredictable usage demands. But these advantages can only be captured if the software itself is able to scale elastically over the pool of commodity resources that is the cloud.

除了充分利用弹性资源以外，研发新的数仓的动机还包括 ETL 太复杂了、knob tuning 太复杂了、数据越来越多了、大数据引擎（Hadoop/Spark）不能取代数仓。关于最后一点原文是

> They(Big Data platforms) still lack much of the efficiency and feature set of established data warehousing technology. But most importantly, they require significant engineering effort to roll out and use.

Snowflake 有如下特点：

1. Pure Software-as-a-Service (SaaS) Experience：开箱即用，简单易用，不需要买机器部署，不需要 DBA，不用调参数，不用执行数据的物理分布，不需要进行 storage grooming 的操作（数据备份、归档、删除、压缩、移动等）。
2. Relational：完整支持 ANSI SQL 和 ACID，无痛上车。作者还在 related work 里吐槽了 BigQuery 只支持狗家自己的 SQL 方言。
3. Semi-Structured：支持 JSON、Avro 等 semi-structured 数据，提供原生的操作 semi-structured 数据的函数和 SQL 语法，进行自动的类型转换和列存优化，让操作 semi-structured 数据的性能接近关系型数据。
4. Elastic：计算和存储资源都可以独立扩展，且在扩展的时候不会影响可用性和性能。
5. Highly Available：可以容忍节点甚至整个数据中心的失败，软硬件升级的时候也不影响可用性。
6. Durable：数据存 S3 上不会丢，外加有复制、备份、闪回之类的保险措施。
7. Cost-efficient：按需分配计算资源，数据压缩节省存储资源。
8. Secure：原始数据、中间临时数据、网络数据都加密过（好奇加密解密有多大开销），防止云厂商偷看，打消用户上云顾虑，提供细粒度的权限管理机制。

## Storage vs Compute

传统的数仓一般是 shared-nothing 架构，比如 Redshift。数据会被切分，每个节点负责存一小部分数据。每个节点的计算任务也根据数据切分而确定。比如进行 join 的时候大表数据已经均匀分布在每个节点上了，只要把小表广播给各个节点，让他们执行各自任务就行。但是，计算和存储耦合在一起有如下几个问题：

1. Heterogeneous Workload：不同的 workload 对硬件和配置的要求不同，比如 bulk loading 是 io 密集型任务，复杂查询是计算密集型任务，对 io 和 cpu 的要求不一样。但在 shared-nothing 架构中每个节点都是同质的，用的硬件和配置都是一样的，只能选一个折中的方案，造成资源利用不充分。比如在做 bulk loading 的时候会 io 吃紧但 cpu 有余，而进行复杂查询的时候会 cpu 吃紧而 io 有余。
2. Membership Changes：因为每个节点要负责一部分数据，如果需要加入或者下线某个节点，需要进行数据转移，进而会影响性能。如果是某节点失败的情况，也需要数据转移，另外本身就需要多副本来保证 highly availablity 和 durablity。
3. Online Upgrade：软硬件升级要求每个节点都重启一遍。虽然可以做滚动升级，但依次重启节点都需要数据转移，性能肯定有影响。

在 on-premise 部署的情况里，一般机器是固定数量的，规划好以后很少扩缩容，机器故障率也不高，升级也很少，一年一次算很勤快了，所以对于 2、3 两点忍忍也可以接受。对于 1 也没啥办法，只能资源利用率低点。

但云上情况不一样，节点失败会更频繁，而且充分利用弹性的话肯定会有频繁的扩缩容，云上的升级迭代也很频繁。考虑到这些原因，Snowflake 采用了存算分离的架构。存储层和计算层都可以独立扩展。计算层 Snowflake 从头实现，用 EC2 去跑，存储层直接用 S3。为了降低存储层和计算层之间的网络开销，计算节点会在 local disk 上缓存数据。

Note：

1. S3 作为一种存储服务，可扩展性高，保证 99.99% data availability 和 99.999999999% durability，能容忍整个 AZ 的失败，存储成本也便宜，但 latency 很高。数据访问也有二八原则，可能只有 20% 是热数据，80% 是冷数据，那么只要把热数据捞到计算节点的 local disk 上降低数据访问的 latency（一份数据就行，不需要多副本，因为 S3 会兜底），感觉相比于所有数据都在 local disk 上且用多副本保证 availability 和 durability 的方案，可以达到性能接近的程度（cache hit rate 足够高的话）而且存储成本会明显下降。性能接近这一点文章中只提了一句 once the caches are warm, performance approaches or even exceeds that of a pure shared-nothing system，我猜理论上这是可能的，但当时应该工程上的优化还差比较多所以不太能比。存储成本这一点整篇文章都没提，按理说冷热分离的方案是能降成本的吧，还是说我理解不到位。
2. 添加、删除无状态的计算节点是很容易的，针对 2、3 两点问题，其实 Snowflake 是把问题的复杂度转移给 S3 去处理了。 
3. 针对 1 这个问题，后面会讲到 Snowflake 用不同类型的 EC2 去完成不同的 workload。

## Architecture

然后我们来具体看 Snowflake 的架构，分为三层，每层都具备 scalability 和 high availability。

1. Data Storage：用 S3 存数据和查询结果。
2. Virtual Warehouses：负责查询的执行。
3. Cloud Services：用来管理 virtual warehouses、查询和事务，存储各种元信息，包括 database schemas、access control information、encryption keys、usage statistics 等等。

![Multi-Cluster, Shared Data Architecture](architecture.png)

### Data Storage

Snowflake 曾考虑过要不要基于 HDFS 自研存储层, 但是发现打不过 S3，决定把有限的精力投入到更有性价比的事情上，比如 Virtual Warehouses 层的 local caching 和 skew resilience。

S3 是一种对象存储服务，提供 PUT/GET/DELETE 等接口，文件不能修改或者 append，GET 可以读文件中的一部分内容。这些特点影响了 Snowflake 的 table file format 和并发控制。一张表会被水平切分成若干个不可变的 table file（对应 block/page 的概念），按 PAX 格式存储（单个 table file 里是列存）。table file header 里记录了每列的起始位置，这样可以用 GET 操作仅读取想要的那几列。

除了存表的数据以外，S3 还会存中间结果（避免计算节点 out-of-memory 或者 out-of-disk）和查询结果（客户端没必要一直跟数据库保持连接，只要等查询结果全部出了以后来 S3 上拿就行）。

诸如表结构、表在 S3 上的文件分布、统计信息、锁、事务日志（不太清楚是啥，好像当时 Snowflake 还没有 redu-undo log）等都存在 Cloud Services 层的 Metadata Storage 里。Metadata Storage 用一个 scalable transactional key-value store，印象里好像是 FoundationDB。

### Virtual Warehouses

一个 virtual warehouse（VW）由若干个 EC2 组成，这些 EC2 是执行计算任务的 worker nodes。

#### Elasticity and Isolation

VW 是纯粹的计算资源，用户可以根据需求决定开多少个 VW，没查询的时候可以关掉所有 VW。

VW 是计算资源隔离的单位。一个查询只会跑在一个 VW 里，不同的 VW 之间不存在相互干扰，天然适合多租户。当然，有些场景没那么在意隔离性，不同 VW 之间可以分享 worker node 降本增效（VW1 的工作量少，VW2 的工作量多，就可以抽调 VW1 的 worker node 去支援 VW2），Snowflake 表示这个会作为 future work 去做。

当 VW 开始执行一个查询时，一些 worker node（数量取决于查询大小）会各自启动一个 worker process 去执行这个查询，查询完成以后 worker process 会销毁，当然对于小查询来说进程启动也是不可忽视的开销，所以未来会考虑针对小查询复用 worker processes。Snowflake 目前没有部分重试的功能，出现失败就得从头再来。 

Note：

1. 好奇为啥是进程模型而不是线程模型。可能一个好处是一个进程因为 OOM 被 OS 干掉的时候不会连累其他查询的进程。多进程做 cpu/mem 等资源控制是不是比多线程难一些。
2. 感觉部分重试的功能在 Spark 这样的大数据引擎里比较常见，MPP 数据库里比较少见，可能跟数据和查询的量级有关，量级越大从头重试的代价就越高。

不同的 VM 还可以使用不同类型的 EC2 以适应不同的 workload（即解决 shared-nothing 架构的第 1 个问题），比如负责日常查询的 VM 就用 Compute Optimized EC2，某天有 bulk loading 任务就新开一个 VM 专门负责，这个 VM 里的 EC2 都是 Storage Optimized。

Note：

1. 文中没有详细讲 Snowflake 的写入流程。如果没有 redo-undo log 的话，写入必须同步到 S3 上才行。这样的话，负责 bulk loading 的 EC2 用 Storage Optimized 的类型是不是也没太大意义。

除了 VM 是弹性的之外，VM 里的 worker node 也是弹性的，这样就可以通过加 worker node 来实现性能增加但成本不变。比如，4 节点做 data load 任务需要 15 小时，32 节点做同样的任务只要 2 小时，成本是 60 : 64。如果查询能力能随着节点数量增加而线性增强的话，这个 feature 会非常有吸引力。

#### Local Caching and File Stealing

If the set of nodes changes; either as a result of node failures, or because the user chooses to resize the system; large amounts of data need to be reshuffled. Since the very same nodes are responsible for both data shuffling and query processing, a sig- nificant performance impact can be observed, limiting elasticity and availability.
三副本保证高可用


写回 s3
写性能差，local disk 什么时候写回 s3，还是只写 wal 就行

省存储成本


上云更好地搜集 workload 并进行调整

tikv 多租户难做，S3 怎么做的

snowflake 需要一个怎么样的存储层
S3 的 iops cpu 等等

中间结果保存 类似 spart

It is common for Snowflake users to have several VWs for queries from different organizational units, often running continuously, and periodically launch on-demand VWs, for instance for bulk loading.

VW 是隔离单位

共享 VW 提高利用率

云最大的特点就是资源弹性

improve the hit rate
avoid redundant caching 

用了 consistent hashing
分布均匀，且能容忍节点加入或者删除
本地命中，没命中先找其他 worker，没有再去 s3
怎么跟多版本协调？将多版本看做不同的个体

"Eager cache" 通常指的是一种缓存机制，其中缓存项在首次访问时就会被立即加载到缓存中，而不是在需要时才进行加载。相对于“懒惰缓存”（lazy cache）需要在第一次访问缓存项时才进行加载，eager cache 有助于提高访问速度和性能，因为缓存项已经预加载到内存中，可以快速地响应后续的请求。但是，eager cache 的缺点是，它可能会占用大量的内存空间，因为它需要在缓存中存储所有的预加载数据。因此，eager cache 应该在适当的时候使用，并且需要注意内存使用情况，以避免出现内存不足的情况。

优化器怎么感知？cache hit rate， sense data distribution
不同表数据之间怎么做 cache 管理

但 file-stealing 不会去找其他 worker，是直接读 s3

优化器怎么切分任务给各个 worker locality

resulting in much better availability than an eager cache or a pure shared-nothing system which would need to immediately shuffle large amounts of table data across nodes

shared-nothing 架构没有 s3 作为 ground truth，有节点退出必须交接数据

There is little value in being able to execute a query over 1,000 nodes if another system can do it in the same time using 10 such nodes.

creating additional opportunities for sharing and pipelining of intermediate results

pipeline 的中间结果？

Most queries scan large amounts of data. Using memory for table buffering versus operation is a bad trade-off here.

对于大查询来说，保留 page 在内存 buffer 里还不如留着内存做计算。即使 mysql 对查询的 buffer 好像也是比较谨慎的。

a pure main-memory engine, while leaner and perhaps faster, is too restrictive to handle all interesting workloads. Analytic workloads can feature extremely large joins or aggregations.

成本对比

连接保持？load balance？

The plan space is further reduced by postponing many decisions until execution time, for example the type of data distribution for joins.
数据怎么发 broadcast join or shuffle join


adaptive execution

Changes to a file can only be made by replacing it with a different file that includes the changes

It would certainly be possible to defer changes to table files through the introduction of a redo-undo log, perhaps in combination with a delta store

看起来就是写流程做的很简单

File additions and removals are tracked in the metadata (in the global key-value store), in a form which allows the set of files that belong to a specific table version to be computed very efficiently.

总觉得 file 自己得维护一些版本链的信息，但似乎都存 meta store 了，oltp meta store point 读写性能更好


min-max based pruning 
多大程度有效
又记一个可以抄进 tidb 的东西（划掉，tiflash 已经有了
塞个 bloom filter 也不是不行

感觉这跟 partition 也相关

Here, the system maintains the data distribution information for a given chunk of data (set of records, file, block etc.), in particular minimum and max- imum values within the chunk.

看起来是写在 meta store 里？如果写在 file 里每个 file 还是要读

runtime filter

二八原则
s3 本地盘缓存 比全用本地盘慢不了太多？

the UI allows not only SQL opera- tions, but also gives access to the database catalog, user and system management, monitoring, usage information

诊断、支持

storage grooming tasks
备份、归档、删除、压缩、移动

metadata store is also dis- tributed and replicated across multiple AZs

foundationdb
Building a metadata layer that can support hundreds of users concur- rently was a very challenging and complex task

session 应该在 cloud service 或者 load balancer 那一层，更像在后者？

升降级是否可怕取决于工程管理

easier etl ETL 会占用更多的存储，这也跟 s3 存储便宜有关？
这部分市场（ETL 工具）也可以被 snowflake 吃下？

The internal encoding makes extraction very efficient. A child element is just a pointer inside the parent element; no copying is required.

按理说序列化之后没法拿到指针，反序列化以后才行，还是说存磁盘的 encode 是特别设计过的，还是可以拿到指针？

SQL Lateral View 是一种在 Hive 和 Spark SQL 中使用的特殊操作，用于展开复杂数据结构中的元素，以便更方便地进行数据分析。Lateral View 可以将嵌套的数据结构（例如数组、Map 和 Struct）展开为行格式，这样可以更轻松地进行查询和分析。

To assess the combined effects of columnar storage, opti- mistic conversion, and pruning over semi-structured data on query performance
消融实验


time travel
用的场景，误删

90 天不 gc？s3 成本低？


no configuration or management

hierarchical key model

constrain the amount of data each key protects
ensures isolation of user data in its multi-tenant architecture, because each user account has a separate account key

redshift share-nothing

mpp 数据库为啥不做中间结果落盘，影响性能？即使落盘，出发点也是降低内存使用避免 oom 等，不是出于计算结果的复用
或者说不落盘也可以做重试，可能抽象的好一点就行？或者是因为大数据的机器比较破比较多，总体失败率比较高

自动做 data partition，性能会差一些吗

不是支持了 json 就是文档数据库了，存储和计算要对 semi-structure data 做优化，否则性能很差

core performance turned out to be almost never an issue for our users
性能不是用户选择产品的唯一因素
The reason is that elastic compute via virtual warehouses can offer the perfor- mance boost occasionally needed.
性能不行就加机器

skew handling and load balancing strategies, whose impor- tance increases along with the scale of our users’ workloads.



the transi- tion to a full self-service model, where users can sign up and interact with the system without our involvement at any phase.

no tuning knob 是其中的一点

htap 做法，应用场景



delta 结构？

物化

升级在云上更重要更频繁

cost 比较