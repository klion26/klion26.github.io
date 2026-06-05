---
title: About Me
date: 2026-03-26
---

# 主要工作
工作主要包括大数据计算和存储，包括异构数据同步，实时数据计算，数据湖的构建和优化等，同时支持过数仓、搜索、广告（样本、训练以及样本）等业务。

异构数据入湖基于 FlinkCDC，性能较开源提升 2+ 倍，唯一一家支持上游分库分表的 SchemaEvolution 自动同步（无约束）；实时计算支持过包括双十一在内重大业务场景，主导开发过 Flink Runtime、State&Checkpoint 等重大特性，在功能（单点重启），性能（StateStore）以及稳定性（Checkpoint 小文件合并，Checkpoint 快速恢复）等方面均有良好的效果；在数据湖方面大规模落地基于 Iceberg 的湖仓一体架构，单客户支持 13W+ 表，千亿单表日更新 100% 的情况下做到 10 分钟时效性，且查询效率明显优于传统数仓，支持多模态数据。Arrow-rs 项目中 parquet-variant 特性核心贡献者，正在构建基于 arrow/datafusion/iceberg 等开源组件的 [多模态数据湖系统](https://www.klion26.com/2026/03/18/multi-mode-datalake-from-google-papers/) 。

# Talks
- 2025 年 CommunityOverCode Asia
- 2024 年 Flink Forward Asia
- 2022 年 CommunityOverCode Asia
- 2022 年 DataFun Summit
- 2021 年 Flink Forward Asia
- 2019 年 Flink Forward Asia

# 经历
- 2024.12 Apache Amoro PMC member
- 2023.8 - 天翼云 大数据产线
- 2020.10 Apache Flink Committer
- 2020.5 - 2023.8 腾讯科技有限公司  数据平台部
- 2018.3 - 2020.5 阿里巴巴 Blink 团队
- 2015.7 - 2018.3 美团 数据平台部
- 2008.9 - 2015.6 中南大学 计算机学院
