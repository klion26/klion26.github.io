# Enabling Agenti Workflows on Iceberg
Aggregations: decouples works
The next consumer isn't a human analyst, three thinks break immediately: fast retrieval, semantic context, security

Two types of indexes: file attached, decoupled

Aggregations: decoupled works, Parquet has no standard for partial aggregate state -- accumulators, HLL sketches, percentile digests, The engine has to own this regardless

Decoupled indexes need a translation layer

Indexing is a primary goal of Iceberg V4, The current direction: Secondary Index Tables, -- indexes stored as sperated tables.

讲了 Vector 和 Semantic search 的例子

Security, row-level security belongs in the engine

The world we want: Engine as accelerator, Iceberg as foundaton

# [Engine][Flink][SPJ] Expanding Apach  Iceberg Engine Support, Storage Partitioned Join(SPJ) for Flink Batch at Pinterest

Flink  Batch User Cases at Pinterest
Backfill capability for streaming jobs
- Streaming job backfill
- Training data backfill

介绍了怎么对 Flink 做修改，适配 SPJ

Modeling Partition Metadata in Flink's Connector SPI
- SupportsPartitioing,  
- KeyGroupedPartitioning

Results
- -50% number of task managers need on same query
- -15% approx, Runtime reduction

Future Improvements
- Partition mismatch problem: When joining tables with different partition counts, the join operation becomes unbalanced
- Correlated Partition Number: When tables share the same partition key but differ in bucket count(e.g., bucket(c2, 4) vs bucket(c2, 8)), partitions don't map one-to-one across tables

# [Rest][Catalog] Fine Grained Metadata Commits

commit
- The client
  - writes manifests & manifest lists
  - validates conflicts
  - constructs snapshots
  - retries through FileIO on failure
- The catalog
  - Simple pointer swap

The consequences
- Client must write metadata files
  - On conflict, client retries
- Catalog is blind to intent
- Every client language must reimplement the full metadata machinery

What if the catalog did more?

Every Iceberg write operation boils down to
- Added data fiels
- Removed data fiels
- Added delete files
- Removed delete files
- Expectegd state of table

Commit Validations, cover all know iceberg conflict anomalies

| Validation | Purpos |
| -- | -- |
| not-allowed-added-data-files | no new data in my scope |
| not-allowed-added-delete-fiels | no new deletes in my scope |
| rquired -data-files| these files must still exist |
| required-delete-files | tehese deletes must exist |
| not-allowed-new-deletes-for-data-files| no new deletes on my files|

The catalog can be smart
- Treiggered Compaction
- Stashed Commits
- Pending state scans
- Merge on Write

# [Eco][Metadata] From Batch to Streaming and AI, Iceberg for Everyone by Everyone 

Iceberg V1 
- Metadata and Data files

Iceberg V2 row-level update

V3 Functionality
- More efficient row replacement
- New Types:Variant, Geo
- Encryption

Issues with Streaming
- V3, many files -> V4 The Root Manifest, Adaptive Manifest

What does AI need from Tables?
- wide table
- tables have turned(can added column -- 动态加列)

Column-Level update
- Replace Columns
- Add Columns

Row/Column update can apply to Metadata also


# [Eco][Kafka] From Balck BHox to Breakpoints: Debugging the Kafka Connet Iceberg Sink Connector
# [Migration][Practise] From Proprietary Storage to OpenLakeHouse: Lessons from a High-Throughput Iceberg Migration

Mission: Share operational hurdles and the specific solution patterns developed in a Iceberg production environment, samll teams navigating the interoperability complexities of setting up managed Iceberg with established vendors

6+ Billion Customer Profiles, 800+ Million Cross-channel messages sent per day, 100+ Million Users processed through our workflows per hour, 30+ Billion Data points handled for our largest customer

Production Iceberg Journey
- Ingest: Pulsar -> Snowflake Native(existing setup)
- Transform and Attribute: Snowpark -> S3(Iceberg Format)
- Consume: Databricks(ML workloads)

Problem(Scaling DML on Iceberg
- High "Cost per row updated" and metadata bloat
- "Write Amplification" caused by attribution merges tourching files across the entire table
Solution
- Use hierarchical clustering and reduced look-back windows to "fence in" the DML

Problem(Multi-Engine Paradox' - Manifest bloat)
- High-frequency commits outpaced background maintenance, causing manifest list growth that stalled the reader refresh
- Because the reader requires a full metadat arefresh per query, uncompacted lists directly degraeded read latency
Solution
- Tune compaction frequency to keep metadat "reader-ready" while minimizing writer compute costs
- Refresh metadat afrequently on dedicated reader compute

Problem: Multi-Engine Paradox - Catalog Integrations
- Inital integeration architecture bloced metadat requests while a write or recluster operation was generating a new snapshot
- Synchronous locking on the writer side triggered systemic reader timeouts and broke the intergration
Solution
- MOve from a "maintenace window" stopgap to a non-blocking API patter
- Return the latest available metadata while active generation is still in progress.

Predication Pushdown: trust but verify
- Implicit engine optimizations for internal tables faield to trigger across the exteral catalog boundary
Solution
- Develop standardized query wrappers to enforce type consistency and "pruning before process" by default
- Centralized heavy ETL into shared libraries so optimized wrippers carry the performance burden

Iteroperatbility Papercuts
- Persistent mapping issues with nested STRUCT data stypes across the catalog boundary forced  a shift to flat schemas
- A writer-side optimization for nullable booleans broke older readers, exposing cross-version Parquet gaps
Solution
- Adopt the "lowest common denominator" approach across engines
- Prefer the most boring, standard implementation that every engine in the ecosystem can read reliably

# [Engine][SQL UDF][Spec] Iceberg UDF Spec: Portable SQL Functions Across Engines

The problem: UDFs are siloed
- Engine-specific, not portable
  - Spark UDF -> Spark only
  - Snowflake UDF -> Snowflake only
  - ..
- Same logic rewrite multiple times
- Subtle drift -> inconsistent results across engines
- No central visibility or governance

What you get?
 - Write once, run everywhere
 - Central governane
 - Versioned, auditable logic
- Why it matters
 - No duplicatoin
 - No drift
 - Aligh UDFs, with tables and views
Use cases1: Reade Restriction for FGAC, Use case2: Constraints

Design principles
- Engine neutrality
- Self-contained metadata
- Versioning&Immutability
- Security & Governance

Why SQL first?
- Universal baseline
- Readable & auditable
- Deterministic inlining
- INcremental strategy

A UDF is just metadata
- Function signature
- Versioned logic
- SQL per engine
- Stored in the catalog

Type System
- Based on the Iceberg type system
- Primitive and semi-structured types are using the same JSON represetntations
- Nested types(List, Map, Struct) representattions are adjusted

Catalog: Store & version metdata, Serve via REST API, Enforce access control
Engine: Resolve overloads, Parse & inline SQL body, Optimize execution
End User: Write SQL per dialect, ensuere equivalence, manage version

# [Metadadata][V3] Iceberg V3 Deletion Vectors and Roaring Bitmaps
Deletion vectors identify deleted rows within a single referenced data file by postiion, these vectors are stored using the `deletion-vector-v1` blob definition from the Puffion spec

Roaring bitmaps
- divide your data(32-bit integer) into fixed size Containers
- look at each container and update it to the most appropriate storage option(Bitmaps, array, runs) as data gets updated

# [Maintain][Catalog][Rest][Security] Iceberg's Iron Wall: Open Governance vai IRC for enforcing row and coolumn security

Sampe query, different results, Why is this hard
- Engine has access to the entire data file -- no way to restrict at row ro column level
- Policies are written in SQL -- but every engine speaks a different dialect
- Define policy once, enforce consistently everywhere

Two choices
- Catalog enforced
  - Catalog applies applicable policies and generates governed datasets
  - Engine nver sees raw files
  - For: unstructed/unknown engines
- Engine enforced
  - Catalog sends a security contract
  - Engine enforces at query time
  - For: trusted connectors

Why IRC?
- Already in the rquest path -- every loadTable() goes through it, zero extra hops
- Single enforcement point -- one catalog, N engines, consistent policy
- Already has identity context -- knows who's asking, what they're accessing

Use Iceberg UDF  to mask senstive data

# [Encryption] Launching Iceberg Table ENcryption

protect the data 的两种方式：hiding senstive ；加解密

Native encryption and tamper-proof mechanism: implented inside iceberg libraries
- Cloud-agnostic, works the same with any storage
- Allows to work with multiple storage backends in parallel
- Or to migrate sensitive tables from one storage to another: just moves files - no need to decrypt/extract/encrypt
- Efficient sharing of senstive tables: no need to decrypt//extract/encrypt a copy for each user; simply provide eligible users with access to table key
- "Client-side" encryption - data and keys invisible to storage backends(and admins)

# [Engine][Table] Multi-Engine Aces with Apache Iceberg: One Table, Multiple Tools
Challenge
- Data duplication
- Cost
- Schema Drift

从 全量拷贝，外表，再到 Iceberg 表
之前 7B 数据 > 8Hours，后来 < 5mins

# [Maintence] Reaching Warehouse-Class Performance: Optimizing Spark + Iceberg at scale 

Five Pillars to optimize Lakehouse Performance
- Data Icebereg Partiion & Layout
  - Column stats and usage
  - Automate partition & layout generation
- ETL Pipelines, Scale Out & Control Input Size
  - Workload profileing
  - Template-driven provision
- Job: Configuration Tuning
  - Resources for consistent execution
  - Maximize throughtput
- Query: Query rewrite
  - Process massive datasets effectively
  - Optimal data pruning
- Platform  Runtime Optimization
  - Adaptive execution
  - Cost-efficient provision


Column Transform: Control Partition Count, Stabilize Partition Size Over Time, Reduce Data Skew 三个维度进行评估分区函数

# [Eco][Flink][Variant][SQL][Sink] Schema flexible INgestion with Flink SQL Variant, and Dynamic Iceberg Sink

Challenge: New Topcis, add and restart pipeline?

Dynamic write patterns -- N : M mapping between topic/table

Topic A                       |-> Table A
Topic B   ---> DynamicRecord -- -> Table B
Topic ?                       |--> Table ?

Why is this good?
- No downtime
- Flexible schema/write pattern
- Easier to maintain and scales to thousands of tables

Use Flink variant to do this(flexible)

# [Metadata][Variant][Semi-structured] The evolution of semi-structured data: Moveing from json string to Iceberg V3 variants

Semi-structured data processing
- IoT
- Data pipelines
- Real-time analytics

Json problem
- Full parse on every read
- No columnar compression
- Schema-on-read = runtime surprise

Benchmark: 5m rows of evetn data with nested JSON payload(~42 kb avg payload size)
- 100% bytes read
- 0% pushdown
- ~ 3x storage bloat

Variant Performance/Storage Efficiency
- ~ 10x faster point reads  shredded column read vs full-blob JSON parse + extract
- ~95% I/O reduction  row-group prunning via shredded column stats
- ~3x smaller storage   binary encoding + dictionary on shredded cols eliminates key repetition

# [Migration][Hive] What Actually breaks in zero-copy Iceberg Migrations(and How we operated Through it)

Hive --> Iceberg

Challenges
- Active ingestion appending data every 5 minutes
- Spark-based dedupe uses internal iceberg tables to process and then moves to Hive partiton directories, which means ORC files have iceberg metadata
- Tables federated across multiple HDFS Name Nodes
- How to migrate with zero-downtime?

Scale: 14000+ Kafka topci, 2PB+ daily Kafka throughput, Capacity: Occupied 75% of HDFS, Cost/Time 所以无法直接复制

Migration Orchestrator

Start ingestion to iceberg -- proceed after --> wait 4 days --> validation complete --> [rename to iceberg directory -> update hive partition, sync permissions -> register files to iceberg](for each datepartition) -> all partitions complete --> wait 7 days --> rollback window closes --> resume table maintenance jobs

Hiccup # 1 The Field ID Minefield
- Years of table recreations and ad-hoc copies left incosistent Field IDs in ORC files
- Read queries will fail because of mismatched columns in ORC files
Solutions
- Add a feature in OpenHouse to match the field IDs of the new table with the legacy one.
- Compare the field IDs during Iceberg registration and do a in-place stripe-based ORC rewrite to remove Iceberg fields in ORC footer.

HDFS  NameNode Storm & Mapping
- HDFS  renames take a global write lock, renaming at the directory level(not file level) kept lock times manageable

Icebereg row counts were higher thant Hive counts: Hive ignores file starting with __(sidecars). Our registration logic did not

Query result inconsistency
- Spark automatically falls back to column names if field IDs are not available in the ORC files
- Trino relies on schema.name-mapping.default table property to resolve fields correctly.

Iceberg is case-sensitive, some old hive partitions had ORC fiels with all lowercase columns, -> Rewrite the ORC files in-place with the correct schema casing.

# [Eco][Performance][Firebolt] What breaks at trillon rwos: eliminating iceberg bottlenecks in firebolt

Data
4 trillion rows, 4 million parquet data files, 20PBs, 600  Avro manifest files, 10s GBs of metadata in manifests, 400 columns, partitioned on 4 columns, updated few times a day

使用 bigmetadata 类似过滤
Streaming: Don't block until all data files are discovered, and do it in parallel, -> Model list of files as result set of TVF
Pruning:  Partition prunning(manifest), data file pruning(manifest), row group pruning & Page pruning (data file)
    Use Iceberg metadata to skip data files
    Create metadata pruning expressions

Node Assignment: Make "optimal" assignment of data files to nodes，---> Arsenal of window functions to compute node assignment
Join planning, use iceberg metadata for join planning  -> While planning, run metadata quires to fetch row counts

# [Catalog][Rest] You deserve a Rest How the Iceberg Rest Spec Unlocks Lakehouse

Iceberg is 4 specs, not one

Rest catalog spec
View spec(cross-engine SQL view)
Puffin spec(stats&deletion vectors)
Table spec(Immutable files, snapshots, schema evolution)

Catalog as security control plane
Server-side conflict resolution

# [Practise] 2+ Years On Iceberg Transforming Manufacturing Data an Looking Ahead
介绍使用

# [Performance] Accelerating Petabyte Scale Iceberg Tables Using Multi--threaded I/O 

Parquet file read pattern
- Read last 8 bytes to parse metadata lenght { seek (lenght -8), read(8) }
- Read metadata from end of file { seek (lenght-8-metadata_lenght), read(metadata_lenght) }
- Fetch required file ranges from qualified row groups sequentially calling seek() and read() in FileIO

Vectored IO
- Read last 8 bytes to parse metadata lenght { seek (lenght -8), read(8) }
- Read metadata from end of file { seek (lenght-8-metadata_lenght), read(metadata_lenght) }
- Call readVectored() with list of all qualified file ranges in FileIO, it is up to FileIO on how to fetch it

Vectored IO implementation in Apache Iceberg
- Default implementation implementation in RangeReadable interface
- Sorts List of ranges and merge overlapping ranges
- For each range in sorted range, calls readFully sequentially

Parallelizing Vectored IO
- Implemented in gcs-analytics-cors's GoogleCloudStorageInputStream
- Sort List of ranges and merge overlapping and nearby ranges
- Fetch all ranges in sorted list in parallel in a thread pool

Benchmaring on GCS
- TPCDS & TPCH
- Metrics measured: End to end execution time, and scan time (custom metric added by patching BatchDataReader

Benchmark results 
1. Total scan time improved by 36+%
2. Total execution time improved by 13+%


# [Catalog][Rest][Polaris][Security] Architecting a Secure maanged Iceberg platform with Apache  Polaris & OPA

Iceberg Client ---->  Polaris Server(Rest) ----PolarisAuthorizer SPI --> OPA(Open Policy Agent)

Iceberg AuthManager SPI
- Standard client-side authentication Service Provider Interface
- Pluggable token acquisition through identity providers
- Avoids forking Iceberg client

Apache Polaris
- Iceberg Rest Catalog Implementation
- Centralized enforcement layer for table access

Open Policy Agent(OPA)
- General-purpose policy decision engine
- Policies evolve independently from Poloaris

# [Mantainence] Branch first backfills, Updating 200 iceberg tables without breaking live APIs

1000+ Iceberg tables
Live client apis
10000+ Views

Properties of Safe Data Update
- Updates in complete isolation
- No Intermediate state
- Test and release
- Clean rollback

Use branch
1 create branch
2 update branch
3 DQ
4 Promote to main

Difference 
| | Blue-Green Principle | Branch first equivalent |
| -- | -- | -- |
| Isolated environment | migration branch - fully isolated |
| Production stable | Clients stay on main |
| Atomic cutover | Promote branch to main |
| Roolback | Move to previous update |

- Changes are gated
- Not a workaround
- Re-usable pattern

# [Eco][GCS][Google Cloud] Build an open, interoperable and high-performant lakehouse with Apache Iceberg on Google Cloud

Our vision: Openness without overhead
- Open & enterprise-ready
- Unified Governance and Security
- AI-ready

# [Table][Metadata][Column-update] Column Families in Iceberg: Enabling efficient feature updates for wide tables

Core Idea: Column Updates

Idea:
- MoR but flip the axis!
- No DVs required
- Writes: projected read + projected write
- Reads: Stitching
- Expectations:
- Faster writes
- Overhead on reads
- Less storage cost
Join the vonversation

Read Path -- Splits
- Row Groups not aligned
  - Split makes sense for base file only
  - Dedicated base file reader or all equal?
- Advance all column update readers
  - _POS from base render
  - move other readers until reaching _POS
  - Read overhead: 1st split 4x 2nd 3x, ...(512MB, 4 row grups)

Select sum(hash(*)) FROM tbl, overhead 10% ~ 35%

# [Eco][CDC][Streaming] Completing the CDC Puzzle: Adding Merge-on-Read Support to Iceberg Changelog Scans

Iceberg's CDC solution: Incremental scans
- Query chagnes between snapshot A and snapshot B
- Highly efficient because it operates at the file metadata level
- The limitation: Historically worked flawlessly only for Append-only tables


MoR CDC, implementa few optimization strategies
- Preventing Driver from OOM
  By using Lazy Evaluation(via the Supplier) and Partition Pruning, we only load the delete files that are strictly necessary for the partitions being queried
- Drastically Reducing Scan Planning Time(CPU&IO)
  Manifest Pruning: Iceberg checks if a manifest's partiton stats overlap with the query filter. If a manifest only contains delets for date='2020-01-01' and you query is for date='2026-04-06', Iceberg skips reading that manifest entirely
  Dynamic Pruning by Affected Partitions: The code builds a filter based on which partitions actually changed in the snapshot range. If only 2 partitions out of 10,000 were modified, it restrictes the delete file search to just theose 2 partitions. This turns an O(N) operation(where N is total delets) into an O(M) operation(where M is modified partitions)
- Solving the "Equality Delete" Correctness challenge
  If we are lazy and only look at deletes in the queried partitions, we might miss a global Equality Delete written in a unpartitioned that applies to our data
  The Optimization: The code checks if there are any Equality Deletes in the scanning range. If there are noen, it safely uses the super-fast lazy path. If there are equality deletes, it falls back to an eager build to ensure correctness, sacrificing some performance to ensure data integerity.
- Cumulative Deletes Pruning
  可能会有 compacton 导致文件变化 Iceberg builds a cumulative view but prunes out delets that were removed in the middle of the range. This ensures we don't apply stale deeletes to data files that were processed later in the stream.

# [Community] Contributing to Iceberg 101

# [Catalog] Driving Iceberg Adoption with Open Catalog and Open datasets

Data Sharing
- Shares all datasets publicly
- No more FTPs

# [Spark] Iceberg + Spark DSv2: Growing Up Fast
DSv2 
- A protocol for integrating external connectors into Spark
- Designed to fix gaps in orginal Data Source API
  - High-level abstractions
  - Complexity shifts to Spark, maiking connectors simple
  - Consistent behavior controlled by Spark
- DSv2 capabilities define Iceberg <> spark integration

Change Data Capture (Spark 4.2)
Merge into (with schema evolution)

# [Pinot] Beyond the Forklift: Unlocking Revenue-Critical Workloads on Iceberg

为什么用 Iceberg
Federate queries across Real-Time & Data Lakes
什么地方用
Use cases
- Observability + Security

做的修改
构建了一些索引（可选），包括多层索引
metadata prune，index 等，这些降低了 5 倍的数据传输量

