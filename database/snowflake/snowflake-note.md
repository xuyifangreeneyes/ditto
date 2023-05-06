# 瓦街千堆雪

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