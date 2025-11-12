多模态数据湖可以是啥样的？

> 最近多模态数据湖的概念比较热门，记录下我理解中多模态数据湖的一个可能形态。

多模态数据湖：能在一套系统中同时处理结构化数据、半结构化数据和非结构化数据。

对于整套系统来说包括数据入湖，数据湖存储（主要是表格式），数据湖内数据的优化，Catalog，以及计算。

> 增加一个全景架构图（包括上面所有这些流程）

其中数据入湖主要需要支持的功能有
- 功能
  - schema evolution
  - 多种同步方式
  - 非结构化数据的同步
  - transformer 
- 性能
- 扩展性

数据湖
- 功能
  - 支持结构化、半结构化、非结构化数据的存储
  - ACID 等
- 性能
- 生态
  - 需要能对接更多的引擎

数据湖内数据治理
- 数据湖内的数据可能对读取性能不是那么友好，需要进行优化
- 会有 management 需求

catalog/metadata
- 跨源跨域等需求
- 权限

计算引擎
- 功能
- 性能

增量计算可以将整个流程的计算耗时大大降低（减少不必要的计算）

参考 FDAP 以及 [voltrondata](https://voltrondata.com/codex/accelerated-hardware) 的模型


BigQuery

- Napa: Powering Scalable Data Warehousing with Robust Query Performance at Google
- Big Metadata: When Metadata is Big Data
- Vortex: A Stream-oriented Storage Engine For Big Data Analytics
- BigLake: BigQuery's Evolution toward a Multi-Cloud Lakehouse
- SQL Has Problems. We Can Fix Them: Pipe Syntax in SQL
  - 这个还是挺好的，也有开源的实现可以参考
- Procella: Unifying serving and analytical data at YouTube
  - [Insights from paper procella unifying serving and analytical data at youtube](https://bytegoblin.io/blog/insights-from-paper-procella-unifying-serving-and-analytical-data-at-youtube.mdx)

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
