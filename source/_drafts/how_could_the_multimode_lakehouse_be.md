多模态数据湖可以是啥样的？

BigQuery

- Napa: Powering Scalable Data Warehousing with Robust Query Performance at Google
- Big Metadata: When Metadata is Big Data
- Vortex: A Stream-oriented Storage Engine For Big Data Analytics
- BigLake: BigQuery's Evolution toward a Multi-Cloud Lakehouse
- SQL Has Problems. We Can Fix Them: Pipe Syntax in SQL
- 




从上到下，结合 fdap（或者类似想法），就是组建做标准，系统搭积木

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

management（类似 amoro）
- 主要是各种service（file layout 重排，索引生成，event trigger 生成 view 等）
- 生命周期管理
- 和 catalog 等结合（轻量级权限，）
