都说 parquet 不太行，lance/nimble 这些新的 format 比较好，那么具体怎么说

列式存储的历史

- PAX <Data Page Layouts for Relational Databases on Deep Memory Hierarchies>
- <An Empirical Evaluation of Columnar Storage Formats>
- <LiquidCache Efficient Pushdown Caching for Cloud-Native Data Analytics>
- <F3 The Opensource Data File Format for the Future>


N-ary Storage Model (5/32 From Pax paper)

Decomposition Storage Model (6/32 From Pax paper)
> Vertical partitioning is the process of striping a relation into sub-relations, each containing the values of a subset of the initial relation's attributes. Vertical partitioning was initially proposed in order to reduce I/O-related costs[26]

Pax mode (7/32 From Pax paper)

然后有行存和列存的一些性能比较。


然后开源有了  Apache Parquet 和 APache ORC，

Parquet 的结构

然后随着时间的推移，基建和要求都发生了变化，现在慢慢会出现了一些新型的 FileFormat，比如 Nimble/Lance/Vortex 等，其中 Nimble 是 Facebook 的开源版本，Lance 由于国内字节在推动最近知名度较高，Vortex 则是另外一个差不多同时期的列存。

这些列存希望解决 Parquet 的两个问题
- 宽列（嵌套列）问题，对于 list/map 等嵌套列，如果要读取单个元素，需要先处理整个 list/map
- 宽表问题 -- 主要在于 parquet 中 meta 的组织形式。

在学术界也同样有一些相应的研究
- 三篇论文

其中分别介绍了一些列存的对比，提出了一个理想的模型，另外是一个工业和学术界结合的列存 cache，这个的原因在于完全推广一套列存所需要的时间会很长（生态、兼容等），但是提供一套内存（非持久化）的 FileFormat 则可以更自由的演进，同时可以跟上新技术/研究。

同时在工业界（Arrow-rs）中也有尝试在优化 parquet 的读写（上面的内存列存也是和这个相关），有通过增加索引，提速 thrift 读取等来加速 parquet 的读取。


----
来自元宝，需要再次确认
----

下表直观对比了它们的核心特性：
| 特性 | Apache Parquet | Lance | Nimble |
| -- | -- | -- | -- |
| 元数据管理 | 集中式 Thrift，需全解码 | 列式存储，支持快速点查 | FlatBuffers，支持零拷贝访问 |
| 宽表支持 | 一般（元数据是瓶颈）| 优秀（为宽表和随机访问优化） | 优秀（为超宽表设计） | 
|数据更新| 需重写文件（成本高） | 支持高效行级更新、删除 | 设计上支持更灵活的更新|
|随机访问速度| 较慢| 极快（适合样本级访问）| 快（元数据访问高效） |
| 特殊优势 | 生态成熟，工具链完善 | 内置版本控制、多模态数据支持 | 编码灵活，为未来硬件优化 |
💎 如何选择
Apache Parquet：对于传统的数仓和分析场景，尤其是**批量处理、全表扫描**为主的场景，Parquet 凭借其强大的生态系统和稳定性依然是优秀的选择。通过优化（如禁用统计信息、使用更高效的读取器），其宽表性能也能得到一定提升
。
Lance：如果你的工作流严重依赖**机器学习、需要频繁的随机访问、处理多模态数据（如图像、文本），或者需要强大的数据版本控制功能**，Lance 是更佳选择。它在需要快速查找特定特征或数据样本的场景下表现突出。
**Nimble**：如果你的主要挑战是处理**极宽的表格（数千甚至上万列）**，并且对元数据访问性能和未来的硬件加速有极高要求，Nimble 是一个非常具有潜力的选项。它特别适合构建大规模特征仓库或需要极致查询性能的系统。
-------

可以考虑 liquidcache（相当于在 format 上造一个 memory format 以及 cache）-- 主要是考虑成本和生态

要弄清楚 Parquet 为啥慢，是真的慢吗
Lance/nimble/vortex 等是真的快吗？

F3 的 column file format

慢的原因
- 使用 thrift 统一存储 meta（相当于行存，没有随机只读部分），flatbuffer 可以？
> 现在 arrow-rs 有优化 thrift 的读取/反序列化这块


parquet 格式

```
magic_char
[[column1][column2]..[column_k]] <- row group1
[[column1][column2]..[column_k]] <- row group2
...
[[column1][column2]..[column_k]] <- row groupn
metadata
metadata_size
magic_char
```

metadata
- version: int32
- schemama : List<SchemaElement>
  - [type: Type]  <- Not set if the current element is a non-leaf node
    - BOOLEAN/INT32/INT64/INT96/FLOAT/DOUBLE/BYTE_ARRAY/FIXED_LEN_BYTE_ARRAY
  - [type_length: int32]
  - [repetition_type: FieldRepetitionType]
  - name: string
  - [num_children: int32]
  - [converted_type: ConvertedType]
    - UTF8/MAP/MAP_KEY_VALUE/LIST/ENUM/EDCIMAL/DATE/TIME_MILLIS/TIME_MICROS/TIMESTAMP_MILLS/TIMESTAMP_MICROS/UINT_8/UINT_16/UNIT_32/UINT_64/INT_8/INT_16/INT_32/INT_64/JSON/BSON/INTERVAL
  - [scale: i32] <- DEPRECATED
  - [precision: int32] <- DEPRECATED
  - [field_id: int32]
  - [logicalType: LogicalType]
    - StringType/MapType/ListType/EnumType/DecimalType/DateType/TimeType/TimestampType/IntType/NullType/JsonType/BsonType/UUIDType/Float16Type/VariantType/GeometryType/GeographyType
- num_rows: int64
- row_groups: List<RowGroup>
  - columns: List<ColumnChunk>
    - [file_path: string]
    - file_offset: int64
    - [meta_data: ColumnMetaData]
      - type: Type
        - BOOLEAN/INT32/INT64/INT96/FLOAT/DOUBLE/BYTE_ARRAY/FIXED_LEN_BYTE_ARRAY
      - encodings: List<Encoding>
        - PLAIN/PLAIN_DICTIONARY/RLE/BIT_PACKED/DELTA_BINARY_PACKED/DELTA_LENGTH_BYTE_ARRAY/DELTA_BYTE_ARRAY/RLE_DICTIONARY/BYTE_STREAM_SPLIT
      - path_in_schema: List<Sting>
      - codec: CompressionCodec
        - UNCOMPRESSED/SNAPPY/GZIP/LZO/BROTLI/LZ4/ZSTD/LZ4_RAW
      - num_values: int64
      - total_uncompressed_size: int64
      - total_compressed_size: int64
      - [key_value_metadata: list<KeyValue>]
		- key: string
		- [value: string]
      - data_page_offset: i64
      - [index_page_offset: i64]
      - [dictionary_page_offset: i64]
      - [statistics: Statistics]
        - max: binary
        - min: binary
        - null_count: int64
        - distinct_count: int64
        - max_value: binary
        - min_value: binary
        - is_max_value_exact: bool
        - is_min_value_exact: bool
      - [encoding_stats: List<PageEncodingStats>]
        - page_type: PageType
          - DATA_PAGE/INDEX_PAGE/DICTIONARY_PAGE/DATA_PAGE_V2
        - encoding: Encoding
          - PLAIN/PLAIN_DICTIONARY/RLE/BIT_PACKED/DELTA_BINARY_PACKED/DELTA_LENGTH_BYTE_ARRAY/DELTA_BYTE_ARRAY/RLE_DICTIONARY/BYTE_STREAM_SPLIT
        - count: int32
      - [bloom_filter_offset: int64]
      - [bloom_filter_length: int32]
      - [size_statistics: SizeStatictics]
        - [unencoded_byte_array_data_bytes: i64]
        - [repetition_level_histogram: list<i64>]
        - [definition_level_histogram: list<i64>]
      - [geospatial_statistics: GeospatialStatistics]
        - [bbox: BoundingBox]
          - xmin: double
          - xmax: double
          - ymin: double
          - ymax: double
          - [zmin: double]
          - [zmax: double]
          - [mmin: double]
          - [mmax: double]
        - [geospatial_types: list<i32>]
    - [offset_index_offset: i64]
    - [offset_index_length: i32]
    - [column_index_offset: i64]
    - [column_index_length: i32]
    - [crypto_metadata: ColumnCryptoMetaData]
    - [encrypted_column_metadata: binary]
  - total_byte_size: i64
  - num_rows: i64
  - [sorting_columns: list<SortingColumn>]
	- column_idx: i32
	- descending: bool
	- nulls_first: bool
  - [file_offset: i64]
  - [total_compressed_size: i64]
  - [ordinal: i16]
- [key_value_metadata: List<KeyValue>]
  - key: string
  - [value: string]
- [created_by: string]
- [column_orders: List<ColumnOrder>]
  - 具体的列排序顺序
- [encryption_algorithm: EncryptionAlgorithm]
  - AesGcmV1/AesGcmCtrV1
    - AesGcmV1
	  - [aad_prefix: binary]
	  - [aad_file_unique: binary]
	  - [supply_aad_prefix]
	- AesGcmCtrV1
	  - [aad_prefix: binary]
	  - [aad_file_unique: binary]
	  - [supply_aad_prefix: bool]
- [footer_signing_key_metadata: binary]

PageHeader
- type: PageType
- uncompressed_page_size: i32
- compressed_page_size: i32
- [crc: i32]
---- Headers for page specific data. One only with be set.
- [data_page_header: DataPageHeader]
  - num_values: i32
  - encoding: Encoding
  - definition_level_encoding: Encoding
  - repetition_level_encoding: Encoding
  - statistics: Statistics
- [index_page_header: IndexPageHeader]
- [dictionary_page_header: DictionaryPageHeader]
  - num_values: i32
  - encoding: Encoding
  - [is_sorted: bool]
- [data_page_header_v2: DataPageHeaderV2]
  - num_values: i32
  - num_nulls: i32
  - num_rows: i32
  - encoding: Encoding
  - definition_levels_byte_length: i32
  - repetition_levels_byte_length: i32
  - [is_compressed=true: bool]
  - [statistics: Statistics]

ColumnChunk
- [file_path: string]
- [file_offset = 0: i64]
- [meta_data: ColumnMetaData]
  - type: Type
  - encodings: list<Encoding>
  - path_in_schema: list<string>
  - codec: CompressionCodec
  - num_values: i64
  - total_uncompressed_size: i64
  - total_compressed_size: i64
  - [key_value_metadata: list<KeyValue>]
  - data_page_offset: i64
  - [index_page_offset: i64]
  - dictionary_page_offset: i64]
  - [statistics: Statistics]
  - [encoding_stats: list<PageEncodingStats>]
  - [bloom_filter_offset: i64]
  - [bloom_filter_length: i32]
  - [size_statistics: SizeStatistics]
  - [geospatial_statistics: GeospatialStatistics]
- [offset_index_offset: i64]
- [offset_index_length: i32]
- [column_index_offset: i64]
- [column_index_length: i32]
- [crypto_metadata: ColumnCryptoMetaData]
  - EncryptionWithFooterKey/EncryptionWithColumnKey
- [encrypted_column_metadta: binary]

RowGroup
- columns: list<ColumnChunk>
- num_rows: i64
- [sorting_columns: list<SortingColumn>]
- [file_offset: i64]
- [total_compressed_size: i64]
- [ordinal: i16]


Parquet foot 性能提升
https://www.influxdata.com/blog/how-good-parquet-wide-tables/
https://github.com/alamb/parquet_footer_parsing
https://datafusion.apache.org/blog/2025/08/15/external-parquet-indexes/
https://lists.apache.org/thread/j9qv5vyg0r4jk6tbm6sqthltly4oztd3
 google doc --> https://docs.google.com/document/d/1kZS_DM_J8n6NKff3vDQPD1Y4xyDdRceYFANUE0bOfb0/edit?tab=t.0#heading=h.ccu4zzsy0tm5
