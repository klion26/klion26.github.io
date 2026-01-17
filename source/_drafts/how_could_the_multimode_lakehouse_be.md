多模态数据湖可以是啥样的？

> 最近多模态数据湖的概念比较热门，记录下我理解中多模态数据湖的一个可能形态。


多模态数据湖：能在一套系统中同时处理结构化数据、半结构化数据和非结构化数据。

对于整套系统来说包括数据入湖，数据湖存储（主要是表格式），数据湖内数据的优化，Catalog，以及计算。

> 增加一个全景架构图（包括上面所有这些流程）

从数据使用角度来看
- 数据最终是为了分析
  - 分析性能
    - 列存（至少是读取，可以用行存写入）
  - 可能需要不断的变换 schema（SchemaEvolution）
  - 需要支持 ACID
  - 能对接尽可能广的生态
  - 支持不同类型的数据
    - 结构化，半结构化，非结构化数据
  - 数据治理
    - 为了性能调整
    - 为了存储空间等治理
数据同步
  - reader + deserializer + transformer + serializer + writer
  - schema evolution

数据湖内数据治理
- 数据湖内的数据可能对读取性能不是那么友好，需要进行优化
- 会有 management 需求

分析会有 引擎的计算，而引擎和表的对接则会需要 catalog/metadata 等

catalog/metadata
- 跨源跨域等需求
- 权限

计算引擎
- 功能
- 性能

增量计算可以将整个流程的计算耗时大大降低（减少不必要的计算）

参考 FDAP 以及 [voltrondata](https://voltrondata.com/codex/accelerated-hardware) 的模型

数据湖可以看成是 FileFormat 的一个索引（可以配合 FileFormat 一起使用）

BigQuery

现在 SQL 使用起来会相对复杂，可以尝试使用 PipelinedSQL 而且这个 SQL 可以更好的转换为拖拽式的。

Dremeal A Decade of Interactive SQL Analysis at Web Scale
Bigtable A Distributed Storage System for Structured Data
Adaptive and Robust Query Execution for Lakehouse at Scale
Column Sketches A Scan Accelerator for Rapid and Robust Predicate Evaluation
- Napa: Powering Scalable Data Warehousing with Robust Query Performance at Google
- [x] Big Metadata: When Metadata is Big Data
- [x] Vortex: A Stream-oriented Storage Engine For Big Data Analytics
- BigLake: BigQuery's Evolution toward a Multi-Cloud Lakehouse
- [x] SQL Has Problems. We Can Fix Them: Pipe Syntax in SQL
  - 这个还是挺好的，也有开源的实现可以参考
- [x] Procella: Unifying serving and analytical data at YouTube
Unity Catalog

# Napa
2021-Powering Scalable Data Warehousing with Robust Query Performance at Google

We need to store and serve these planet-scale data sets under the extremely demanding requirements of scalability, sub-second query-response times, availability, and strong consistency; all this while ingesting a massive stream of updates from applciations used around the globe.

The following key aspects of Napa are the bedrock principles of its design and are aligned with our client requirements:
- Robust Query Performance: Consistent query performance is critical to data analytics users. Our clients expect low query latency, typically around a few hundreds of milliseconds, as well as low variance in latency regardless of the query and data ingestion load. Napa is able to guarantee robust query performance with consistent results despite daunting requirements for scale and system availability.
- Flexibility: While performance is important, our experience shows that it is not the only criterion for our clients. For instantce not all applications require millisecond response times, not all require the same freshness for ingested data, or for that matter, not all clients are willing to pay for "performance at any cost". Clients also require the flexibility to change system configurations to fit their dynamic requirements.
- High-throughput Data Ingestion. All Napa functions, including storage, materialized view maintenance, and indexing, must be performed under a massive update load. Napa implements a distributed table and view maintenance framework that is based on the LSM-tree paradigm. LSM is widely used in current generation of data warehouses and databases primarily to efficiently integrate and incorporate constantly emerging data into already existing data. Napa scales LSM to meet the challenges of Google's operating environment.

Napa's approach for robust query performance includes the aggressive use of amterialized views, which are maintained consistently as new data is ingested across multiple data centers.

权衡三角：Data freshness, Resource costs, Query performance 三者之间做权衡

A key design choice in Napa is to rely on materialized views for predicatable and high query performance.

Napa's high-level architecture consists of three main components as shown in the figure above.
- Napa's ingestion framework is responsible for committing updates into the tables. The deltas written by the ingestion framework only serve to satisfy the durability requirements of the ingestion framework, and hence are write optimized. These deltas need to be further consolidated before they can be applied to tables and their associated views.
- The storage framework incrementally applies the updates to tables and their views. Napa tables and their views are maintained incrementally as log-structured merge-forests. Thus, each table is a collection of updates. Deltas are constantly consolidated to form larger deltas; we call this process "compaction" The view maintenance layer transoforms table deltas into view deltas by applying the corresponding SQL transformation. The storage layer is also responsible for periodically compacting tables and views.
- Query serving is responsible for answering client queries. The system performs merging of necessary deltas of the table(or view) at query time. Note that query latency is a function of the query time merge effort, so the faster the storage subsystem can process updates, the fewer deltas need to be merged at query time. F1 Query is used as the query engine for data stored in Napa. We provide more details for query serving in Section 8.

Napa decouples ingestion from view maintenance, and view maintenance from query processing. This decoupling provides clients knobs to meet their requirements, allowing tradeoffs among freshness, performance, and cost.

Users specify their requirements in terms of expected query performance, data freshness, and costs.
Napa introduces the concept called `Queryable Timestamp (QT)` to provide clients with a live marker(just like an advancing timestamp). QT is the direct indicator of freshness since [NOW() - QT] indicates data delay. All data up the QT timestamp can be queried by the client.

> Napa architecture showing the major system components
![](https://raw.githubusercontent.com/klion26/ImageRepo/master/202512291645719.png)

Napa's high-level architecture consists of data and control planes as above. The architecture is deployed at multiple data centers to manage the replicas at each data center. The data plane consists of ingestion, storage, and query serving. The control plane is made up of a controller that coordinates work among the various subsystems. The control is also responsible for synchronizing and corrdinating metadata transactions across multiple data centers.

Napa clients use ETL pipelines to insert data into their tables. The ingestion framework can sustain very high load, such as tens of GB/s of compressed data. Client data are delivered to any of the Napa replicas and Napa ensures that the data ingestion is incorporated at all the data centers. This significantly simplifies the design of ingestion pipelines.

Query serving deals with the necessary caching, prefetching and merging of deltas at runtime. The goal of query serving is to serve queries with low latency and low variance. Low latency is archieved by directing the queries to precomputed materialized views as opposed to the base table, and parallel execution of queries. Low variance is achieved by controlling the fan-in of the merges as well as a range of other I/O reduction and tail tolerance techniques.

Napa relies on views as the main mechanism for good qauery performance. Napa's choice is largely motivated by the strict latency and resource requirements of its workloads, making it necessary to leverage indexed key lookups.

The Nap controller schedules compaction and view update tasks to keep the count of deltas for a table to a configurable value. These storage tasks are needed to keep the queryable Timestamp(QT) as fresh as possible given the cost tradeoffs.

The goal of the ingestion framework is to accept data, perform minima processing, and make it durable without considering the pace of subsequent view maintenance.

The queryable timestamp(QT) of a table is a timestamp which indicates the freshness of data that can be queried. If QT(table) = X, all data that was ingested into the table before time X can be queried by the client and the data after time X is not part of the query results.

An important criterion to ensure good query performance is to optimize the underlying data for reads and ensure views are available to speed up the queries.

A table in Napa is a collection of all of its delta files, each delta corresponding to updates received for the table over a window of time, as below
![](https://raw.githubusercontent.com/klion26/ImageRepo/master/202512291733204.png)

The non-queryable deltas correspond to newly received updates written by the ingestion framework in the most recent time window(typically seconds.) THe largest deltas, on the other hand, span a time window of weeks or even months. Each delta is sorted by its keys, range partitioned, and has a local B-tree like index.

The QT of the database is the minimum of the QT of all the tables in the database. QT is also used to give clients a consistent view of data across all Napa replicas.

Napa's storage subsystem is responsible for maintaining views and compacting deltas. It is also responsible for ensuring data integrity, durability via replication across data centers, and handling outages from individual machines to entire data centers.

Compaction improves query performance and reduces storage consumption by 1) sorting inputs together and 2)aggregating multiple updates to the same rows. Since the delta files are individually sorted, compaction is essentially merge sorting. It is intentionally kept large during compaction so that the height of the merge tree is small, thus minimizing key comparisions.

Robust query serving performance
query serving subsystem achieves robust performance, using Queryable Timestamp(QT), materialized views, and a range of other techniques.
1. Reducing data in the critical path
Whenever possible, Napa uses views to answer a query instead of the base table, since views with aggregation functions may have significantly less data. Nap maintains sparse B-tree indexes on its stored data, and uses them to quickly partition an input query ito thousands of subqueries that satisfy the filter predicates. This partitioning mechanism additionally looks at the latency budget and availability of query serviing resources to achieve good performance.

> 增加 Figure7

2. Minimizing Number of Sequential I/Os
When a query is issued, Napa uses the value of QT to decide the version of metadata to be processed. The metadata in turns determines what data has to be process. Therefore, mateadata reads are on the critical path of query serving. Nap aensures all metadata can always be served from memory without contacting the persistent storage. This is achieved by affinity-based distributed metadata caching with periodic background refresheds. A particular QT is delayed to wait for the completion of periodic background refresh of metadata.

Napa performs offline and online prefetching to further reduce the number of sequential I/Os in the critical path. Offline prefetching occurs as soon as data is ingested for frequently queried tables, before QT advances to make the new data available to query. Online prefetching starts when a query arrives and is performed by a shadow query executor which shares the data access pattern with the main query executor but skips all query processing steps.(like dry run?)

3. Combining Small I/Os
During query serving, Napa aggressively parallelizes the work by partitioning the query into fine grained units and then parallelizing I/O calls across deltas and across queried columns. To combat such amplification on tail letency, Napa uses QT to limit the number of queryable deltas. In addition, Napa also tries to combine small I/Os as much as possible, by using the fowlling two techniques: lazy merging across deltas and size-based disk layout.

- lazy mering across deltas: When there are thousands(N) of subqueries and serval tens(M) of deltas, the number of parallel I/Os are in the order of tens of thousands(N * M). However, due to the parallelism each subquery reads very little data from most deltas. Meanwhile, a large fraction of Napa queries require merging based on a subset of primary keys in the subsequent phase of the query plan. In these cases, Napa adapts the query plan to avoid cross-delta merging in Delta Server and lets each Delta Server only process one delta, combining N * M parallel I/Ss into close to N parallel I/Os （merge 的数据尽量不跨 delta server？）
- size-based disk layout: Napa uses a custom-built columnar storage format supporting multiple disk layout options, which are applied based on delta sizes. The PAX layout which can combine all column accesses into one I/O for lookup queries, is applied to small dltas. For large deltas, column-by-column layout is used that is I/O efficient for san queries but requires one I/O per column for lookup queries. This size-based choice ensures that Napa receives columnar storage benefits as well as reduces I/O operations.（基于大小选择不同的磁盘布局，PAX 作为小的布局，可以一次 IO 读取完毕，大文件则是按列存放，这样每一列需要一个 IO 但是 scan 更友好

4. Tolerating tails and failures
Napa adopts the principle of tolerating tail latency, rather than eliminating it, because eliminating all soruces of variability for such a complex and interdependent system is infeasible.

For a non-streaming RPC, such as the RPC between Metadata Server and Delta Server, Napa uses the machnism of *hedging*, which sends a secondary RPC identical to the original one to a different server after a certain delay, and waits for the faster reply.
For a streaming RPC, such as the RPC between F1 worker and Delta Server, Napa estimates its expected progress rate and requires the server executing it periodically to report progress, together with a *continuation* token. If the reported progress is below expectation or the report is missing, the last continuation token would be used to restart a new streaming RPC on a different server without losing progress. Pushdown operators like filtering and partial aggregation need to be carefully handled in progress reporting as they can significantly reduce the data size, causing progress reports to be superficially low or even missing. Napa uses bytes processed before filtering and partial aggregation as the progress rate metric and periodically forces these operators to flushes its internal state to generate a progress report with a continuation token.

Production metrics insights
Napa manages thousands of tables and views in production, where many tables are petabyte scale. It serves over a billion queries per day and ingests trillions of rows. Napa is able to provide robust query performance through three techniques: 1) by more actively using views, Napa reduces raw query performance and variance even at 99th percentile, 2) by chaning storage policies, Napa can reduce the number of deltas and hence the tail latency, and 3) by decoupling ingesting, view maintenance, and query execution; Napa can mitigate the impact of infrastructure and workload changes on query performance.

1. Views and QT Help achive robust query performance
First, most client queries are aggregation queries, and materialized views are typically at least an order of magnitude smaller than the base tables from which they are derived. Reading from views not only improves raw performance, but also improves tail latency as their smaller size is more cache friendly, and requires less compute resources, which reduces the change of continetion for query resources.

> Add the figure for #views with the query latency

Second, latency can be improved by reducing the number of deltas that have to be opend, read, and merged at query time.（相当于文件数量？）
> Add the figure for #deltas with the query latency

Figure 8(b) shows that as we change storage policies to reduce the number of deltas, the query latency improves significantly. The biggest impact is at the 99th percentile latency which reduces by more than 3.6x as the number of deltas is changed from 8 to 2. The main reasons are: 1) fewer deltas means there are less number of small, parallel IOs which are prone to cause latency tails, 2) fewer deltas also means that data is premerged and aggregated, and less processing is required at query time.（这个类似 MOR 中，文件数量少，则 IO 少，而且读取的时候需要进行的聚合计算也少）

2. Handling infrastructure issues
> Add figure 9
Figure 9 shows that Napa is able to guarantee its client stable query performance even when the ingestion load changes or there are infrastructure outages.  Napa decouples ingestion from view maintenance and querying which allows us to optimize for low variance in query latency, in some cases by trading off data freshness. Figure 9(a) shows that the client continuously sends data to Napa, with some variance in the input rate over the course of the week. Figure 9(b) shows that the view maintennace performance dropped for the duration between X and Y indicating an infrastructure issue which affected the tasks updating views. However, the query serving latency remains near constant (Figure 9(d)) throughout the whole duration. In this particular example, client queries continued to be fast, however , for certain parts during the outage data freshness was impacted, as seen in Figure 9(c) where the value of delay is high. 就算 view 的维护因为基建受影响了，那么不影响写入和读取的性能，但是能读到的数据的新鲜度会受影响

3. Client workloads
支持不同的 tradeoff，比如 [Client A: Tradoff freshness] wants moderate query performance and low costs, but can tolerate lower freshness.
[Client B: Tradeoff query performance] cares the most about low costs but can tolerate lower query performance
[Client C Tradeoff costs] has high freshness and high query performance requirements, and is willing to pay high costs to achieve them.
> Add figure 10 for different workloads

Napa is a fully indexed system that is optimized for key lookups, range scans, and efficient incremental maintenance of indexes on tables and views. Napa can easily support both adhoc queries and highly selective and less diverse queries.

Napa uses a varaint of B+-trees that exploits the fact that Napa tables have multi-part keys. Additionally, min/max keys(per-column min/max values) are stored along with each non-leaf block to eanble effective pruning. LSM adapt B-tree indexes for high update rates. Napa belongs to a class of  LSM systems that trade high write throughput for fast reads. (看起来是 LSM 和 B-tree 的结合）


# 2023-Progressive Partitioning for Parallelized Query Execution in Google's Napa
Napa holds Google's critical data warehouses in log-structured merge trees for real-time data ingestion and sub-second response for billions of queries per day. These queries are foten multi-key look-ups in highly skewed tables and indexes.

In our production experience, only progressive query-specific partitioning can achieve Napa's strict query latency SLOs.  Here we advocate good-enough partitioning that keeps the per-query partitioning time low without risking uneven work distribution. Our design combines pragmatic system choices and algorithmic innovations. For instance, B-trees are augmented with statistics of key distributions, thus serving the dual purpose of aiding lookups and partitioning. Furthermore, progressive partitioning is designed to be "good enough" thereby balancing partitioning time with performance. The resulting system is robust and successfully serves day-in-day-out billions of queries with very high quality of service forming a core infrastructure at Google

Napa's diverse query workload consists of large scans and many-key lookups. The analytical queries with many-key lookups have strict QoS requirements and is the main focus of this paper.

From our experience in running production services, we found the following three requirements important:
- Query-specific partitioning. Any approach that we take should meet the latency SLOs across a diverse set of query workloads. In our experience, the partitioning granularity needs to be adjusted on a per-query basis to meet the latency and resource budget requirements. The expectation from the partitioning step is that it is able to produce number of partitions denoted by the parallelization requirement within bounded amount of error.
- Evenness. Execution involves the partitioning of the tables into key ranges such that the partitioning results in even partitions. Partitioning should operate on tables with extreme skews where some keys span terabytes in disk. As an example, in Figure 1, a key range like < K1 = 1, K2 = 20 > may correspond to, say, a hundred GB portion of the table while another key range may be considerably smaller at a few MB.
- Progressiveness. Query partitioning must balance overall execution time against the time and effort to produce query-specific partitions. For instance, one can produce perfectly even partiions while still ending up missing the QoS requirement. It is imperative that the partitioning method has the notion of "good enough" in the sense that it stops when the partitioing is the sufficient quality. Our porposed technique in hte paper is progressive such that 1) the longer the alogirhtm runs the better the quality of the resultant partitioning; 2) the algorithm stops once the desired error bound has been met.

Our experimental results show that the use of statistics that is too find-grained can often result in queries spending too much time generating partitioning at the expense of overall query execution time.

We address this dilemma by leveraging B-trees to optimize access to the statistical information on the tables. Each LSM run has an associated B-tree index which are enhanced so that index nodes maintain size information for the associated key ranges. With these enhancements, we can estimate the input data size of the query starting with the root of the B-tree. If this estimation is not accurate enought or if the statistics point to areas of skews, we can descend to the next level in the index structure to obtain a finer level of statistical distributions for the key ranges overlapping with the query.

Our proposed algorithm traverses the B-tree to produce even and query-specific partitioning. It is progressvie in the sense that it descends and accesses additional index information only if it does not satisfy the stipulated error bounds. In addition, the refinement is selective that it only descends to the lower levels of trees for those partitions that do not have gight enough bounds on the error.

The following key aspects of Napa are the bedrock principles of its design and are aligned with our client requirements: i) Robust Qury Performance: Napa clients expect low query latency, typically sub-seconds, as well as low variance in latency under a wide spectrum of query and data ingestion load; ii) System Flexibility: While performance is important, our clients also require the flexibility to change system configurations to their dynamic requirements such as trading freshness or recency of ingested data for better performance; iii) High-throughput Data Ingestion: Napa's ingestion, storage and query serving functions performan udner a massive update load.

A Napa table consists of multiple data sets called deltas(corresponding to sorted runs of an LSM-tree) and we have one B-tree index for each delta. Not that each delta corresponds to updates on a table during a time window(e.g., last 1 minute, 1 day etc.).

Progressive partitioning using B-tree: The B-trees on the deltas are fairly generic except that it hierarchically stores statistics of the underlying data. In particular, we store the number of rows that is indexed by each sub-tree and aggregate that statistics up the level. The partitioning algorithm we propose in the paper takes queried keys as input, retrieves and merges relevant keys taken from these B-tree indices, to generate an approximate histogram. Starting with the highest level histograms (i.e., the B-tree root index blocks), it tries to find accurate enough paritions. If it needs more detailed information, the algorithm will retrieve requried index blocks from the next highest level, repeating this trial and retrieve until it reaches the desired accuracy.

LSM 让 partition 变的更复杂
The LSM data model complicates the task of query-specific partitioning in the following ways.
- First, the keys specified in the query may be present in may(possibly all) of the deltas. Note that the same query worker must process all deltas where the keys are present in order to reconcile all relevant version.
- Second, this proves to be a serious challenge for the evenness of the partitioning since some deltas may contribute many more matched rows than others.
- Third, a query may involve seeking across multiple deltas and not all deltasequally contribute to the query. Thus, the paritioning effort may not be uniform across the deltas. For some deltas, it may be sufficient to perform coarser partitioning while others require significantly more effort in producing equal splits.

Why rely on progressive query-specific partitioning?
The ideal partitioning unit size can vary from 10MB(e.g., when reading 1GB data using 100 scan workers) to 1GB(e.g., when reading 1TB data using 1000 scan workers) on a per-query basis. 
The next complexity here is that it is possible that there is one particular "Date = d3", which accounts for most of the data to be scanned in a query. So, that means that we need to partition the range <c1, d3>, <c2, d3> and <c3, d3> much more finely than the other key ranges. This means the partitioning algorithm needs to be progressive such that it can stop early on other dates and focus on generating finer grained partitions for "Date = d3" 
Standard write-time partitioning mechanisms are not able to address the requirements discussed above.

Partitioning is highly specific to query under consideration and existing write-time partitioning is inadequate for workloads with a wide spectrum of query workloads. We have also established that one needs a way of producing both fine and coarse grained partitions even on the same key range, based on the query predicates, latency target and resource budget.

Progressive partitioning
The progressive query-specific partitioning algorithm using *size-enhanced* B-trees. Our solution is to enhance typical B-trees with the statistics on data size in a hierarchical manner. For each delta, we maintain a B-tree pointing to data blocks, e.g. a block in the PAX layout. The index not only helps a query to efficiently seek on data using prefix keys but also provides statistical information for partitioning. Both querying and query-specific partitioning traverse the B-tree. The querying traverses the B-tree through the leaf level to visit dat ablocks. The query-specific partitioning does not visit data blocks and visits the index node at the leaf level only when required for finding "good enough" partitions as we describe below.
> 使用 size-enhanced B-Tree，对于每一个 delta 都维护一个 B-tree 指向 data blocks，这样可以提供前缀查询，以及统计信息等

For a given query Q, the algorithm divides D deltas into P key range partitions that are nearly even with an error margin of \{thelta}. Each delta has a size-enhanced B-tree, which carries keys and the size of data between these keys. We analyze the estimated size of matching data by combining keys and sizes from D indices. To get the most precise matching estimate from B-trees, we may need to visit the leaf index entries matching with the query. 但是读所有的 leaf 会很耗时，另外一种方式就是通过 root 来推测 partition 的数量，这是两个极端，文中从中间进行选择：从 root 开始，然后仅在需要下一层提供更多信息的时候才进行继续读取。

文中有一个具体的例子
> 补充 fig 4 和 fig 5

其中两个 Delta，然后希望分成两个 partition（分 parition 是希望能够并行处理），然后考虑第一层的时候（从 B-tree 获得信息），Delta 1 的 [K1, K3) 有 15 个，[K4, K8) 有 20 个； Delta 2 的 [K2, K5) 有 20 个，[K6, K7) 15 个。

那么把所有的数目都加在一起就是 15 +  20 + 20 + 15 = 70 个，然后会选择在 K4 和 K5 这里做分割，这样会把 [K4, K8) 和 [K2, K5) 做一些分割，然后对于第一个 partition 来说，大小可能是 15（也就是 [K2, K5) 的倾斜全在右侧，[K4, K8) 也倾斜），同样最大可能是 15+20+20 = 55,那么估算这个大小是 (15 + 55) / 2 = 35，然后 size margin = 55 -15 = 40,同样第二个 partition 类似，得到估算大小是 35, size margin 是 40,如果 (40/35, 40/35) -- 两个 parititon 的误差 -- 能满足 \{theata} 那么就按照这么切分，如果不行，就对 B-Tree 进行下一层读取，然后再进一步进行划分

> We represent this size-embedded index entry in a B-tree index node as e = <start, end, size, level, block>, which corresponds to the key range [e.start, e.end) having the estimated size e.size. The entry e is in a index block at tree level e.level(the value of 0 for e.level corresponds to the leaf level). Finally, e.block is a pointer to the child index block having range [e.start, e.end) (if e.level > 0)

> 根据和 Gemini 的沟通，大概理解 Napa 的存储结构如下
> 首先是 LSM Tree，然后每个 SSTable 是一个 B-tree，B-tree 就是文中的 Delta，B-tree 中包括多个实际的文件，以及对应的索引，然后渐进式则是「够用」就好，-- 「够用」是满足对应的误差

# SQL Has Problems
文章描述了 SQL 是一个很好的抽象，但是学习和维护不方便，重新造一个类似 SQL 的语言又很麻烦，所以有了 PipelinedSQL 这个渐进式改进的方案。
>SQL is not an easy language to learn or use. Even for expert users, SQL is challenging to read, write and work with, which hurts user productivity. Serveral alternative languages have been prposed, but none have gained widespread adoption or displaced SQL. Migrating away from existing SQL ecosystems is expensive and generally unappealing for users.

Piplined SQL 可以让 SQL 更好用，更具扩展性（more flexible, extensible and easy to use). 

In SQL, the standard clauses occur in one rigidly defined order. Expressing anything else requires subqueries or other workarounds. With Pipelined SQL operationcan be composed arbitrarily, in any order.

给了两个例子
```sql
SELECT c_count, COUNT(*) AS custdist
FROM
  (SELECT c_custkey, COUNT(o_orderkey) c_count
   FROM customer
   LEFT OUTER JOIN orders ON c_custkey = o_custkey
        AND o_comment NOT LIKE '%unusual%packages%'
   GROUP BY c_custkey
  ) AS c_orders
GROUP BY c_count
ORDER BY custdist DESC, c_count DESC;
```

然后 
```pipelined-sql
FROM customer
|> LEFT OUTER JOIN orders ON c_custkey = o_custkey
     AND o_comment NOT LIKE '%unusual%packages%'
|> AGGREGATE COUNT(o_orderkey) c_count
   GROUP BY c_custkey
|> AGGREGATE COUNT(*) AS custdist
   GROUP BY c_count
|> ORDER BY custdist DESC, c_count DESC;
```

有讲 SQL 和 PipelinedSQL 的语义情况
![](https://raw.githubusercontent.com/klion26/ImageRepo/master/202512161155193.png)

觉得 SQL 不错，那么就修复
- SQL's foundational semantics, from relational algebra, are excellent
- SQL's conceptual data model and top-level syntax, with statements representting requires, DDL, DML, etc, and composability via subqueries, works weel.
- SQL's basic operations within a query(clauses like JOIN, ORDER BY, etc) all work reasonably well
- SQL's syntactic structure for composing queries using those operations is **terrible**.
- SQL'ls localized syntax with English-like keyword phrases is and anachronism, but we can live with it.
- SQL's expression language is fine, and implementations typically includes a good library of existing functions.

所以只需要修复第四点中的问题就行

pipelined SQL 使用 '|>' 这个 suffix 来衔接

```
<query> :=
  {all existing query syntaxes}
  | "FROM" <from_body>             -- New: FROM as a query
  | <query> "|>" <pipe_operator>   -- New: pipe suffixes
```
论文中有具体算子的语法

然后剩下的是一些更具体的细节，以及相关效果。
这个如果成熟之后，应该也可以更方便的推广拖拽式的表现形式（？），能够更好的一一对应起来
然后 ZetaSQL（已开源） 也有实现

# Napa
# BigLake
# Procella
> These can be categorized as: reporting and dashboarding, embedded statistics in pages, time-series monitoring, and ad-hoc analysis. Typically, organizations build specialized infrastructure for each of these use cases. This however, creates silos of data and processing, and results in a complex, expensive, and harder to maintain infrastructure.
> At YouTube, we solved this problem by building a new SQL query engine -- Procella.

These workloads have differenet set of requirements:
- Reporting and dashboarding: Video creators, content owners, and various internal stakeholders at YouTube need access to detailed real time dashboards to understand how their videos and channels are performing. This requieres an engine that supports executing tens of thousands of canned queries per second with low latency (tens of milliseconds), while queries may be using filters, aggregations, set operations and joins. The unique challenge here is that while our data volumen is high (each data source often contains hundreds of billions of new rows per day), we require near real-time respoinse time and access to fresh data.
- Embedded statistics: YouTube exposes many realtime statistics to users, such as likes or views of a video, resulting in simple but every high cardinality queries. These values are constantly changin, so the system must support millions of e real-time updates concurrently with millions of low latency queries per second.
- Monitoring: Monitoring workloads shrae many properties with the dashboarding workload, such as relatively simple canned queries and need for fresh data. The query volumne is often lower since monitoring is typically used internally by engineers. However, there is a need for aditional data management functions, such as automatic downsampling and expiry of old data, and additional query features (for example, efficient approximation functions and additional time-series functions)
- Ad-hoc analysis: Various YouTube teams (data scientists, business analysts, product managers, engineers) need to perform complex ad-hoc analysis to understand usage trends and to determine how to improve the product. This requires low volume of queries(at most tens per second) and moderate latency (second to minutes) of complex queries (multiple levels of aggregations, set operations, analytic functions, joins, unpredcatable patterns, manipulating nested/repeated data, etc.) over enormous volumes(trillions of rows) of data. Query patterns are highly unpredictable, although some standard data modeling techniques, such as star and snowflake schemas, can be used.

独立系统的问题
- Data needed to be loaded into multiple systems using different Extract, Transform, and Load(ETL) processes, leading to significant additional resource consumption, data quality issues, data inconsistencies, slower loading times, high development and maintenance cost, and slower time-to-market.
- Since each internal system uses different languages and API's, migrating data across these systems, to enable the utilization of existing tools, resulted in reduced usability and high learning costs. In particular since many of these systems do not support full SQL, some applications could not be built by using some backends, leading to data duplication and accessibility issues across the organization.
- Several of the underlying componenets had performance, scalability and efficiency issues when dealing with data at YouTube scale.

Procella 的特性
- Rich API: Procella supports an almost complete implementation of standard SQL, including complex multistage joins, analytic functions and set operations, with serveral useful extensions such as approximate aggregations, ahdnling complex nested and repeated schemas, user defined functions, and more.
- High Scalability: Procella separates compute (running on Bord) and storage (on Colussus) enabling high scalability (thousands of servers, hundreds of petabytes of data) in a cost efficient way.
- High Performance: Procella uses state-of-the-art query execution techniques to eanble efficient execution of high volumen(millions of QPS) of queries with very low latency (milliseconds).
- Data Freshness; Procella supports high volume, low latency data ingestion in both batch and streaming modes, the ability to work directly on existing data, and native support for lambda architecture.

> Procella System Architecture

Procellas 的数据按照 table 进行划分，每个 table 会有多个文件/分区。Procella 使用自己的列存 Artus，也支持其他的 file format，存在 Colossus 中，这样可以存算分离

Procellas 没有使用 BTree 而是使用了更轻量级的 zonemaps, bloom filters, partition 以及 sort key 等这些二级索引，这些从文件头，或者 plan 时从 data server 获取，schema, table to file mapping, stats, 以及 zonemap 等这些存储在 BigTable/Spanner 中

支持标准的 SQL 操作，支持 batch ingestion 和 streaming ingestion
在 batch ingestion 的时候，可以根据 file header 获取，更耗时的元数据抽取会延迟处理；streaming ingestion 支持 RPC/PubSub 的方式，然后 Ingestion Server 转换为表的形式

然后会有后台线程周期性进行合并操作

Query lifecycle
Clients connect to the Root Server(RS) to issue SQL queries. The RS performs query rewrites, parsing, planning and optimizations to generate the execution plan. To this end, ti uses metadata such as schema, partitioning and index infromation from the Metadata Server(MDS) to prune the files to be read. It then orchestrates the query execution as it goes through the different stages enforcing timeing/data dependencies and throttling. To address the needs of executing complex distributed query plans(timeing/ data dependencies, diverse join strategies, etc.), the RS builds a tree composed of query blocs as nodes and data streams as edges (Aggregate, Execute Remotely, Stagger Execution, etc. ) and executes it accordingly. This enables functionality such as shuffle (that requires timing dependency), uncorrelated subquery, subquery broadcast joins(data dependency) and multi-level aggregation. THis graph structure allows several optimizations by inserting custom operations into the tree based on query structure. Once the RS receives the final results, it sends the respoinse back to the client, along with statistics, error and warning messages, and any other information requested by the client.

The Data Sever(DS) receives plan fragments from the RS or another DS and does most fo the heavy lifting, such as reading the required data (from local memory, remote and distributed files in Colossus, remote memory using RDMA,  or another DS), executing the plan fragment and sending the results back to the requesting RS or DS.

优化
- Cache
  - Colossus metadata caching  可以减少一次或多次文件打开的开销
  - Header caching  文件头(尾）有 start offset，column size，minimum and maximum 值，缓存这些可以减少更多的文件访问（单独的 LRU）
  - Data caching, DS 使用单独的 cache 缓存文件，列存 Artus 设计成 disk 和 memory 的 format 一致，这样可以让 cache 更友好(making cache population fairly cheap)，同时也缓存一些二级数据（比如bloom filter 等）
  - metadata caching: metadata 存在 bigtable 这样的分布式文件系统中，访问多可能会导致有瓶颈，因此用 LRU 来 cache 
  - Affinity scheduling: Caches are more effective when each server caches a subset of the data. Procella implements affinity scheduling to the data servers and the metadata servers to ensure that operations on the same data/metadata go to the same serve with high probability. 这样每个 server 只负责一部分数据，cache 有效性也变高. 但这个不是强约束，也就是说可能被调度到其他的 server -- 这样只是性能变差（cache 无效）。
- Data Format
  - 一开始使用 Capacitor，但是它主要是 Ad-hoc 中使用（主要是 scan 性能），Procella 需要 lookup 和 range scan，所以开发了 Artus
  - 使用自定义的 encoding。Uses custom encodings, avoiding generic compression algorithms like LZW. This ensures that it can seek to single rows efficiently without needing to decompress blocks of data, making it more suitable for small point lookups and range scans.
   - Does multi-pass adaptive encoding: 第一次收集轻量级信息（ndv, min, max, sort order 等），然后使用这些去选择更合适的 encoding
   - 选择对 sorted column 支持 binary search 的 encoding
   - 嵌套字段的处理和 parquet 不一样，把表的 schema 当成 tree，每个字段都存在为独立的列, 针对每个父字段（如果自己本身存在的话），会记录所有子字段出现的次数，对于可选字段，该值是 0 或者 1,重复字段该值为非负。但是不记录父子段不存在的信息  
   - Directly exposes dictionary indices, Run Length Encoding information, and other encoding information to the evaluation engine.
   - Records rich metadata in the file and column header.
   - Supports storing inverted inddexes. -- 搜索场景有用

支持 multi-level partitioning and clustering

Join 的类型
- Broadcast： 一方足够小可以加载到 DS 的内存。
- Co-partitioned: fact 和 dim 可以使用同样的 key 进行 partition
- Shuffle：数据太大，然后没有按照 join key 排序，就进行 shuffle
- Pipelined: RHS 是负责查询，但是结果可能比较小的时候
- Remote lookup: dim table 是分区的，但是 fact 没有

怎么解决 tail latency
- 因为数据有备份，所以如果某个数据节点的访问延迟超过了中位数(median)，就访问其他的节点 
- RS 会控制访问同一个 DS 的频率，批次等
- RS 会给每个请求附带一个优先级，通常来说小查询的优先级更高，大查询的优先级更低。DS 为高优和低优的分别分配资源，这样可以保证小查询的快速响应-- 不会被大查询 block

Intermediate merging
- 对于很重的聚合，最后的聚合可能会成为瓶颈 -- 需要在单机点操作很多数据。为了避免这个，添加中间节点用来做中间的聚合。

Query Optimization
- Virtual Tables 类似 MV
  - Index-aware aggreagte selection
  - Stitched queries
  - Lambda architecture awareness
  - Join awareness

Query Optimizer
使用静态和动态的查询优化技术 (makes use of static and adaptive query optimization techniques) At query compile time we use a rule based optimizer that applies standard logical rewrites (always benefical) such as filter push down, subquery decorrelation, constant folding, etc. At query execution time we use adaptive techniques to select/tune physical operators based on statistics (cardinality, distinct count, quantiles) collected on a smaple of the actual data used in the query.
- Adaptive Aggregation
- Adaptive Join
- Adaptive sorting

Adaptive 的不足：需要动态收集信息，可能导致短查询耗时更久（比如 10ms 左右的可能变成原来的数倍），这种可以让用户给定 hint 不走 adaptive optimization，另外用户愿意接受短查询变长，换来大查询的耗时变短（overhead 可能有 10% 左右）

针对 serving 做优化处理
- 数据写入后，会立即通知 dataserver 进行加载，这样可以避免冷读
- MDS 编译进 Root Server，可以减少 RPC 交互
- metadta 提前加载到内存，避免查询的时候加载 metadata
- query plans are aggressively cached to eliminate parsing and planning overhead.  This is very effective since the stats query patterns are highly predicatble.
- The RS batchs all request for the same key and sends them to a single pair of primary and secondary data servers.
- The problematic outlier tasks are automatically moved to other machines.


# Vortex
> Vortex: A Stream-oriented Storage Engine For Big Data Analytics

Vortex is a streaming-first storage system that supports both streaming and batch data analytics. Today, BigQuery uses Vortex to support petabyte scale data ingestion with sub-second data freshness and query latency.

Vortex has the following key properties:
- Consistent: Guaranteees ACID properties for all API operations.
- Unified API for batch and streaming: Vortex offers a single unified API with support for both streaming and batch data.
- Scalable: Vortex implements a fully distributed data and control plane and as a result supports tables of multiple petabytes size.
- Performant: The Vortex API offers sub-second tail write latencies that simplify client side application programming.

BigQuery storage provides a global namespace over all data in BigQuery. Data is organized into regional containers called datasets(analogous to schemata in traditional database management systems). Tables, logical views, metarialized views, search indexes, stored procedures, machine learning models etc. all reside in a dataset.

Vortex is BigQuery's scalable, distributed and synchronnously replicated storage engine that supports data ingestion, retrieval and curation.

A Vortex Stream is an entity to which rows can be appended to the current end. Each row in a Vortex Stream is identified by the Stream's identifier and its row offset within the Stream. Readers can concurrently read a Stream at different row offsets. A table is an unordered collection of Stream.
> Stream 有点像传统说的文件，比如 Parquet File？但是应该还不一样，毕竟还有 ColumnFormat

Streams are backed by the following entities
- Streamlets Vortex Streams provide durable sotrage of data. A Streamlet is a contiguous slice of rows in the Stream, all of which are present in the same 2 clusters. A Stream is an ordered list of one or more Streamlets. Given the Stream's append-only semantics, a Stream has at most one writable Streamlet. The writable Streamlet, if one exists, is always the last Streamlet in a Stream.

- Fragments: Each Streamlet is further split into contiguous blocks of rows called Fragments. Fragments typically are a range of rows inside a log file. Log files are stored in Colossus.
> Streamlets 和 Fragments 是 写入端的概念，还是类似表级别的概念
> Streamlets and Fragments are internal physical metadata entities; they aren't visible to the users of Vortex.
- Data formats: BigQuery operates broadly with data in two different classes of data formats. The write-optimized storage format(WOS) is the format in which data is written by Vortex's append API, The read-optimized storage format(ROS) is the format in which data is optimized for dat aprocessing. 
  BigQuery use Capacitor as ROS, BigLake use Parquet as ROS

![](https://raw.githubusercontent.com/klion26/ImageRepo/master/202512041721506.png)

架构如上
- Stream Metadata Server(SMS) is the control plane of Vortex.
The SMS assigns a Streamlet to a specific Stream Server. THe Stream Server maintains the set of fragments for the Streamlet.

- The Stream Server is the data plane of Vortex. It owns a set of STreamlets and creates Fragments for those Streamlets. The Stream Server has its own in memory metadata about its Streamlets and Fragments, and persists this by writing to a transaction log and periodically writing checkpoitns.
The Stream Server knows which Fragments belong to which Streamlet, their committed size, the minimum and maximum record timestamp in each Fragment, whether a Streamlet or Fragment is finalized, the schema version, and the partitioning and clustering columns of the table.

客户端通过 SMS 创建一个 Stream，SMS 会生成一个随机的 StreamId 然后 assign 到一个 Stream Server，Client 直接往 StreamServer 写数据，StreamServer 会周期性的通过心跳同步当前的数据。
Storage Optimization Service 会从 SMS 请求一批 Fragments 然后转为 ROS 的格式，然后把 Fragements 标记为 DELETED，再过一段时间之后，SMS 会通知 Stream Server 将过期的 Fragements 进行删除。

Stream Server 到 SMS 的心跳只包含元数据（与上一次心跳的 Streamlet 的不同数据）

- Fragment File Format
> Fragment 像是一个适合写入的文件格式（和 ROS 的 Parquet 对比）
> Header 包括了同一个 Streamlet 中之前为删除的 Fragment，后面包括了 bloomfilter 以及其他 footer 信息
Each Fragment begins with a header which contains the File Map. The File Map lists the committed size and record ranges of all previous Fragments in the same Streamlet which have not yet been deleted. The File Map is used for disaster resilience.

When a Fragment is finalized, the Stream Server appends a bloom filter, followed by a fixed length footer which describes the offset in the Fragment where the bloom filter starts. The bloom filter marks which key values are present for the partitioning and clustering columns.

The Fragment data format stores metadata records for FlushStream calls on BUFFERED streams. A flush operation is a metadata write to the Fragment which advances the committed row offset making all rows in the Streamlet(and the Stream) up to that point visible.

Vortex uses an end-to-end CRC to protect row data as it is sent from the client to the Stream Server, and from the Stream Server to Colossus.

Stream Server 会周期性（秒级）将 Streamlet 的变化同步给所有的 SMS，这些变化包括（新文件的创建，存量文件的写入，文件中的列属性）- 这个是 per-Streamlet 的，然后还有当前的 CPU，memory 以及 append throughput -- 这个用来做负载均衡。

> A background service continuously optimizes data in Vortex as it is written. The goal ofStorage Optimization is to pitmize the format and layout of the data for large scale analysis. In doing so, it maintains an LSM tree of Fragments, starting with Fragments in WOS at the deepest level of the tree, with progressively more optimized ROS versions as we climb up the ree.
使用 LSM tree 管理 Fragments，然后从 deepset 将 WOS 的转换为 ROS 的格式

> To track the lifetime of Fragments, each Fragment maintains two timestamps: a *creation_timesatmp* and a *deletion_timestamp*.  A Fragment is visible to requests that read the table at a snapshot timestamp that is within the interval [creation_timestamp, deletion_timestamp). At each step of the optimization, the optimizer atomically sets the deletion_timestamp for the previous version of the fragments and the creation_timestamp for the new version. This guarantees that a row is included exactly once when reading the data from storage.
Fragment 可以有 creation_timestamp 和 deletion_timestamp，会自动给已经合并的 Fragment 赋值 deletion_timestamp（这个要保证事物）

**Automatic Reclustering**, BigQuery allows users to cluster the data in a table on a set of specified columns. Clustering defines a "weak" sort order on the data blocks in the table. In other words, BigQuery attempts to distribute the data such that the blocks store non-overlapping ranges of values for the clustering keys. Orgamizing data in non-overlapping ranges results in more efficient processing at read time, by improving parititon prunning and or by reducing intermediate data transferred between query processing stages.
> 可以对数据进行重排序，而且可以做分布的排列，尽量做到不重叠，这样可以做到更好的 partition prunning
> Once the delta(WOS) is sufficiently large, the optimizer first range partitions the delta locally.
如果 WOS 对读影响太多，就本地先合并, 形成一个新的 base
> Figure 6 显示了一个 Automatic Reclustering 的具体例子, Base(ROS) + Delta(WOS) 可以合成新的 Base(ROS) 

> Vortex continuously tracks metadata for Streams, Streamlets, and Fragments.
> An example of coarse grained metadata is the state of a Streamlet that indicates whether the Streamlet is currently writable and its currentl  length. The source of truth for Streamlet is marked finalized. As the storage optimizer moves data between the layers in hte LSM tree, BigQuery's highly scalable metadat management system, called BigMetadata

> There is a tail of the Fragment and Streamlet metadata that may have not yet been indexed by Big Metadata. As the metadat of these blocks churns rapidly, we observe that scanning through the list of these tail blocks that need to be read to satisfy the snapshot read, can add latency to query processing. To address this, we continuously compact the metadat entries for the Fragments by keeping the entries corresponding to the live fragments(i.e. fragments with deletion_timestamp unset) together in our log.
对元数据周期性做 compaction, 尽可能只保留 live fragment 的元数据

Vortex 会周期性的做数据正确性验证(Vortex continuously traces requests to detect data correctness issues such as missing or duplicated records), Finally, we also verify that each record is reported as converted exactly once from WOS to ROS. Additionally, for each conversion, we validate that the output records are consistent tiwth the input records.

When Vortex SMS receives this request(read), it returns the union of the data in WOS and ROS.

Vortex allows a range of rows in a Fragment or Streamlet to be marked as deleted. To limit the size of these deletion masks, sometimes rows unaffected by the DML statement may also be marked deleted
> 有点类似 pos delete？为了减少 pos delete 的数目，干脆把这重写？

# BigMetadata
主要想法
- 使用类似系统表保存 metadata，这样可以横向扩展，而且可以使用 query engine 来处理 metadata 和 data
- 提出了 falsifiable expression，可以改写 condition，从而大规模过滤数据
- metadata 可能很大，所以在读取的时候可以使用 falsifiable expression，减少读取 metadata 的数据量。
写一个 falsifiable expression 的例子
> Traditionally, Big Data systems have tried to reduce the amount of metadata in order to scale the system, often compromising query performance. In Google BigQuery, we built a metadat management system that demonstrates that massive scale can be achived without such tradeoffs. We recognized the benefits that fine grained metadata provieds for query processing and we built a metadata system to manage it effictively. We use the same distributed query processing and data management techniques that we use for manageing data to handle Big metadata.

> We present a distributed metadata management system that stores fine grained column and block level metadata for arbitrarily large tables and organizes it as a system table. We use a novel approach to process large amounts of metdata in these system tables by generating a query plan that integrates the meatdata scan into the actual data scan. In doing so, we leverage the same distributeddquery processing techniques that we use on our data to our metadat, thereby achieving performanct access to it along with high scalability. Our system has the following salient properties;
- Consistent
- Built for both batch and streaming APIs
- (Almost) INfinitely scalable: Our design is built for scale and it supports tables of multiple petabytes isze, and due to its distributed nature scales in a way similar to the underlying data
- Performance

Most of the data in BigQuery is stored in columnar format blocks. Tables range from a few bytes to tens of petabytes. Large tables are stored in millions of columnar blocks. BigQuery supports tables with tens of thousands of columns. BigQuery maintains the entire mutation history of a table for a configurable amount of time in order to support queries on the table as of any timestamp in that history window

We classify storage metadata into two broad categoreis: Logical metadata and Physical metadata.
- Logical metadata is information about the table that is generally directly visible to the user. Some examples of such metadata are: Table schema, Partitioning and clustering specifications, column and row level ACLs. This information is generally small in size and lends itself to quick access. 
- Physical metadata is information about the table's storage that BigQuery maintains internally in order to map a table name to its actual data.

使用系统表来存储 physical metadata: To illustrate our idea, we describe the metadata layout using one such system table (hereafter referred to as CMETA) that stores column level information about the min/max values (called range constraints), hash bucket and modulus values (called hash constraints) and a dictionary of column values. Other variants of system tables include those that store posting lists of column values. Query optimizer chooses one or more such system tables for planning and executing the query

每一列会有 totalRows/totalNulls/totalBytes/min/max/dictionary/bloom_filter 等信息

Block 是 metadata 更新的最小单元

metadata 有 changelog，记录 properties/timestamp 等

A background process constantly performs LSM style merges on the change log to produce baselines and deltas of changes.

Fine grained metadata for rows that have not yet been compacted into capacitor blocks is maintained in the memory of ingestion servers.

BigMetadata 使用了一套 condition 重写的逻辑(falsifialbe expressions)，可以更好的进行过滤比如 x > c => max(x) <= c, x < c => min(x) >= c 等等

metadata 可能会很大，所以也使用 falsiiable expression 进行读取

文章说偏结论性的东西（自己理解），不要说太多细节
- 细节别人可能不是太感兴趣
- 纯细节不太能体现自己的东西
- 可以在最后附加参考文献

计算引擎的 execution plan 能否动态更新（根据数据的不同，动态的变动）
- Procella

Procella 的 meta 是树形结构，而且不仅仅保存叶子节

从上到下，结合 fdap（或者类似想法），就是组建做标准，系统搭积木

parquet 可以有不错的性能（比如 Influxdata 的优化，以及 LiquidCache 的做法等），新的 FileFormat 需要的整体工作量还挺大的
- 首先需要知道 parquet 的瓶颈（可以看新的 fileformat 的 PR 稿）
- parquet 的 ML 讨论（https://lists.apache.org/thread/8xmxc76nd00624qqps6s1qw6lhv1qwv5）/Influxdata 博客（https://www.influxdata.com/blog/how-good-parquet-wide-tables/）

引擎 bigquery
- 用户端
-    sql（可以优化 sql，包括 pipelined sql）
-    dataframe
-    其它（处理非结构化数据）
- 然后 query engine
-    plan 优化
-    优雅的调度和停止等
- exection
-     native（性能）
- 分布式执行
-     资源管理
-      多类型资源统一管理

table format
- catalog 
-    权限
- metadata
-    索引（不同类型的索引）
-    bigemetadata（sql 改写，下推，cache 等）
- file format
-     结构化数据（主要是宽表，宽列，然后 map，list 这种符合类型）
-     半结构化数据（类似 variant）
-     非结构化数据
-     新硬件
- 

还需要能够支持非结构化数据
- 现在有多种开源的 table/file format，其中 lance 由于国内字节在推，所以被知道的比较多。这里可以分为：1）better parquet，2）原生支持非结构化数据。其中第一个就是 parquet 现在支持不太好的情况（比如 list/map 等，这些在推荐等会有需要），点查（这个可以在 parquet 上用二级索引支持 -- 参考 datafusion 的博客），当然 lance 的二维拆分会是一个更高效的处理方式（尤其是模型处理的时候加维度）；第二个就是支持非结构化数据，这个有点类似高效支持 blob 数据，然后支持索引。


management（类似 amoro）
- 主要是各种service（file layout 重排，索引生成，event trigger 生成 view 等）
- 生命周期管理
- 和 catalog 等结合（轻量级权限，）
