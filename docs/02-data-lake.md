# 数据湖架构——详细设计

> 主文档：[README.md](../README.md) 第 3 章

---

## 目录

- [1. 架构总览](#1-架构总览)
- [2. 存储格式选型](#2-存储格式选型)
  - [2.1 分层存储策略](#21-分层存储策略)
  - [2.2 Iceberg + Lance 分工](#22-iceberg--lance-分工)
  - [2.3 为什么不用 LanceDB 全托管](#23-为什么不用-lancedb-全托管)
- [3. Catalog 选型](#3-catalog-选型)
- [4. 表设计](#4-表设计)
  - [4.1 数据湖四层架构](#41-数据湖四层架构)
  - [4.2 分区策略](#42-分区策略)
  - [4.3 各层表 Schema](#43-各层表-schema)
- [5. 增量数据管理](#5-增量数据管理)
  - [5.1 追加式写入 vs Compaction](#51-追加式写入-vs-compaction)
  - [5.2 Compaction 调度策略](#52-compaction-调度策略)
  - [5.3 增量处理流水线](#53-增量处理流水线)
  - [5.4 数据重处理与回溯](#54-数据重处理与回溯)
- [6. Snapshot 与时间旅行](#6-snapshot-与时间旅行)
  - [6.1 Snapshot 保留策略](#61-snapshot-保留策略)
  - [6.2 血缘追踪实现](#62-血缘追踪实现)
- [7. S3 存储生命周期](#7-s3-存储生命周期)
- [8. 权限与安全](#8-权限与安全)
- [9. 运维与监控](#9-运维与监控)

---

## 1. 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        数据湖架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────┐    ┌──────────────────────────────┐   │
│  │   REST Catalog        │    │   Spark / Ray / Dagster       │   │
│  │   (Iceberg 元数据)     │◄───│   (计算引擎 + 调度)            │   │
│  └──────────┬───────────┘    └──────────────────────────────┘   │
│             │                                                    │
│    ┌────────┴────────┐                                          │
│    │                  │                                          │
│    ▼                  ▼                                          │
│  ┌──────────────┐  ┌──────────────────────────────────────┐     │
│  │  Iceberg 表   │  │  Lance Datasets                      │     │
│  │  (Parquet)    │  │  (列式存储 + 向量索引)                 │     │
│  │              │  │                                      │     │
│  │  raw/        │  │  silver_emb/                         │     │
│  │  bronze/     │  │  ├── github_code_cluster/            │     │
│  │  silver/     │  │  ├── web_doc_en_cluster/             │     │
│  │  gold/       │  │  ├── web_doc_zh_cluster/             │     │
│  │              │  │  └── ...                              │     │
│  └──────┬───────┘  └────────────────┬─────────────────────┘     │
│         │                           │                            │
│         └───────────┬───────────────┘                            │
│                     │                                            │
│                     ▼                                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    S3 兼容对象存储                           │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐ │   │
│  │  │ Standard │ │  IA      │ │ Glacier  │ │ Deep Archive │ │   │
│  │  │ (热)     │ │  (温)    │ │  (冷)    │ │  (归档)      │ │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**设计原则**：
- **元数据与数据分离**：Iceberg 管理元数据（表结构、snapshot、分区），Parquet/Lance 存储实际数据
- **文本与向量分离**：文本/结构化数据走 Iceberg + Parquet，embedding/ANN 索引走 Lance
- **全链路可追溯**：通过 Iceberg snapshot ID + Lance dataset version 联合定位任意时刻的数据状态

---

## 2. 存储格式选型

### 2.1 分层存储策略

**核心决策：Iceberg + Parquet 存储文本和结构化元数据，Lance 存储 embedding 向量和 ANN 索引。**

这是经过对比分析后的推荐方案：

| 方案 | 做法 | 优势 | 劣势 | 结论 |
|---|---|---|---|---|
| **A) Iceberg 统一管理 Lance 文件** | 在 Iceberg 表的 `format-version=2` 中使用 Lance 作为底层文件格式 | 统一元数据管理、snapshot/time travel 单一路径 | Lance 不是 Iceberg 原生支持的文件格式，需自定义 FileIO 插件，工程风险高 | ❌ |
| **B) 分层存储（推荐）** | Iceberg + Parquet 管文本和结构化元数据，Lance 独立管 embedding 和 ANN 索引 | 成熟稳定、各自用最擅长的格式、低耦合 | 两套元数据体系、跨表关联需通过 ID 手动 join | ✅ |
| **C) LanceDB 全托管** | 不用 Iceberg，用 LanceDB 原生管理数据 + 向量索引 | 最简单，LanceDB 原生支持 | 丢失 Iceberg 的 snapshot/time travel/血缘能力，无法满足全链路追溯需求 | ❌ |

### 2.2 Iceberg + Lance 分工

| 存储层 | 格式 | 存储内容 | 数据库/表 | 访问方式 |
|---|---|---|---|---|
| **raw** | Iceberg + Parquet | 原始采集数据（文本、HTML、文件二进制） | 按 source 分表 | Spark / SQL |
| **bronze** | Iceberg + Parquet | 粗过滤后的文本 + 过滤结果元数据 | 按 source 分表 | Spark / SQL |
| **silver** | Iceberg + Parquet | 粗加工/细加工后的文本 + 完整元数据（不含 embedding 向量值） | 单表 `all_sources` | Spark / SQL |
| **silver_emb** | Lance | 每条数据的 content_embedding + summary_embedding + ANN 索引 | 按簇分 dataset（如 `github_code`、`web_doc_en`） | Lance API / Ray |
| **gold** | Iceberg + Parquet | 评测后 + 格式转换的最终训练数据集 | 按数据集分表（如 `pretrain_v1`、`sft_v1`） | Spark / SQL / 训练框架 |

**关联方式**：`silver` 表和 `silver_emb` Lance dataset 之间通过 `id` 字段关联。血缘追踪时，在 Iceberg snapshot metadata 中记录对应的 Lance dataset 版本号：

```json
{
  "snapshot-id": "7520426094023953954",
  "timestamp-ms": 1718409600000,
  "summary": {
    "operation": "embedding-generation",
    "lance-dataset-version": "v12",
    "lance-clusters": ["github_code", "web_doc_en", "web_doc_zh", "api_doc"]
  }
}
```

### 2.3 为什么不用 LanceDB 全托管

虽然简单，但：
- **丢失 snapshot/time travel**：无法回溯到「2026 年 6 月 15 日的训练数据快照」
- **丢失血缘追踪**：从模型 checkpoint → 训练数据集 → 原始 URL 的全链路断开
- **多引擎兼容性差**：Spark 对 LanceDB 的支持远不如 Iceberg
- **运维风险**：LanceDB 作为独立数据库需单独运维，而 Iceberg 只是文件 + Catalog，零数据库运维

---

## 3. Catalog 选型

### 3.1 环境分析

- 底层存储：云厂商 S3 兼容接口（非一定 AWS S3）
- 计算引擎：Spark + Ray
- 调度框架：Dagster / DolphinScheduler

**结论：选 REST Catalog**。原因：
- AWS Glue 仅适用于纯 AWS 环境，与云厂商 S3 接口可能不兼容
- Hive Metastore 需维护独立服务，分区数 > 10 万时性能下降
- REST Catalog 是 Iceberg 社区推荐的下一代 Catalog，轻量、无状态、多云友好

### 3.2 Catalog 对比

| 维度 | REST Catalog | AWS Glue | Hive Metastore | JDBC Catalog |
|---|---|---|---|---|
| **部署方式** | 自建无状态服务 | AWS 托管 | 自建 | 自建 |
| **S3 兼容性** | ✅ 与底层对象存储无关 | ⚠️ 仅 AWS S3 | ✅ | ✅ |
| **多引擎支持** | ✅ 原生支持 Spark/Trino/Flink | ✅ EMR/Athena | ✅ Hadoop 生态 | ⚠️ 有限 |
| **千万分区性能** | ✅ 线性扩展 | ⚠️ > 10 万分区的 API 限流 | ❌ 瓶颈明显 | ❌ 不适合 |
| **运维复杂度** | 低（无状态、可 K8s 部署） | 极低（托管） | 高（需维护 Metastore DB） | 低 |
| **原子操作** | ✅ 支持 | ✅ 支持 | ✅ 支持 | ✅ 支持 |
| **社区成熟度** | 快速增长（Iceberg 1.4+ 推荐） | 成熟 | 最成熟 | 一般 |

### 3.3 REST Catalog 部署方案

```
部署拓扑:
  - 2 副本（高可用），K8s Deployment
  - 后端存储：PostgreSQL / MySQL（仅存 namespace 和 table 映射，非数据）
  - 与 Spark/Ray/Dagster 通过 REST API 通信

关键配置:
  - catalog.warehouse: s3://bucket/
  - catalog.io-impl: org.apache.iceberg.aws.s3.S3FileIO
  - catalog.s3.endpoint: https://s3.your-cloud-provider.com  (云厂商 S3 端点)
```

---

## 4. 表设计

### 4.1 数据湖四层架构

采用经典 Medallion Architecture（Bronze → Silver → Gold）：

```
┌──────────────────────────────────────────────┐
│                RAW 层（原始数据）               │
│  表: raw_github_static, raw_web_directed, ... │
│  存储: S3 Glacier Instant Retrieval          │
│  不可修改、不可删除（仅追加）                    │
│  保留: 10 年                                  │
└──────────────────┬───────────────────────────┘
                   │ 粗过滤
                   ▼
┌──────────────────────────────────────────────┐
│              BRONZE 层（粗过滤后）              │
│  表: bronze_github_code, bronze_web_doc, ...  │
│  存储: S3 Standard-IA                        │
│  含 SHA256 + PII 审计日志 + 过滤结果           │
└──────────────────┬───────────────────────────┘
                   │ 粗加工 + 细加工
                   ▼
┌──────────────────────────────────────────────┐
│              SILVER 层（加工后）                │
│  表: silver_all_sources (Iceberg)            │
│  向量: silver_emb/ (Lance, 按簇分 dataset)    │
│  存储: S3 Standard (近期) / Standard-IA (历史)  │
│  含完整元数据 + embedding + 索引               │
└──────────────────┬───────────────────────────┘
                   │ 评测 + 格式转换
                   ▼
┌──────────────────────────────────────────────┐
│               GOLD 层（训练就绪）               │
│  表: gold_pretrain_v1, gold_sft_v1, ...      │
│  存储: S3 Standard                           │
│  最终训练数据集，含完整 meta                   │
└──────────────────────────────────────────────┘
```

### 4.2 分区策略

#### 4.2.1 RAW / BRONZE 层：日期 + source 分区

```
分区键: dt=YYYY-MM-DD / source={github,web_directed,web_general,api_doc,issue_pr}

示例:
  raw_github_static/
    dt=2026-06-15/
      source=github/
        00000-0.parquet
        00001-0.parquet
```

**选择理由**：
- 数据天然按 source 隔离，避免跨源扫描
- 增量写入按天追加，每个 source 独立分区，写入不冲突
- 查询时可按 source 过滤

#### 4.2.2 SILVER / GOLD 层：两种可选方案

| 维度 | **方案 A：按 source 分区** | **方案 B：按 hash bucket 分区** |
|---|---|---|
| **分区键** | `dt=YYYY-MM-DD / source=github` | `dt=YYYY-MM-DD / bucket=N`（N=0..255） |
| **每分区大小** | 不均匀（GitHub 代码 > API 文档） | 均匀（256 个 bucket） |
| **写入性能** | 好（无跨分区的 shuffle） | 中（需 hash 重分布） |
| **按 source 查询** | **极快**（只扫对应分区） | 慢（需全表扫描） |
| **全量扫描** | 文件大小不均匀，可能出现 straggler | **均匀**，最优并行度 |
| **与多级检索索引的匹配** | **完美**（source 正好是检索的第一层路由） | 不匹配（同 bucket 内含多种 source，需跨簇检索） |
| **与其他维度查询**（如按 lang） | 慢（需所有 source 分区） | 均匀 |
| **Compaction 效率** | 中（不均匀分区 compaction 复杂） | **高**（均匀分区易于并行） |
| **Meta 数据量** | 低（source 数量少，分区数少） | 中（256 bucket × 天数） |

**推荐**：

- **Phase 1**：使用**方案 A（按 source 分区）**。因为 Phase 1 的查询模式以按 source 过滤为主（检索时先路由到 source 簇），且 source 分区与多级索引架构天然匹配。source 数量有限（4-5 个），分区数可控。
- **Phase 2**：如果按 lang 或 content_type 的查询成为主要模式，或者全量扫描的 straggler 问题严重，再迁移到**方案 B（hash bucket）**。Iceberg 的 partition evolution 支持在线变更分区策略。

### 4.3 各层表 Schema

#### RAW 层

```
raw_github_static:
  id              UUID        (PRIMARY KEY)
  content         BINARY      (原始文件内容)
  source          STRING      (固定值 "github")
  url             STRING      (原始 URL)
  fetch_ts        TIMESTAMP   (采集时间戳)
  dt              STRING      (分区键: YYYY-MM-DD)

raw_web_directed / raw_web_general:
  id              UUID
  content         BINARY      (原始 HTML)
  source          STRING
  url             STRING
  fetch_ts        TIMESTAMP
  dt              STRING
```

#### BRONZE 层

```
bronze_{source}:
  id              UUID
  content         STRING      (脱敏后文本)
  char_length     INT
  token_count     INT
  lang            STRING      (代码语言 / zh / en / multilingual)
  lang_detail     STRING      (python, javascript, ...)
  raw_url         STRING
  pii_audit       JSON        (PII 脱敏审计日志)
  pass_rule1      BOOLEAN     (格式检查)
  pass_rule2      BOOLEAN     (去重检查)
  pass_rule3      BOOLEAN     (质量检查)
  pass_rule4      BOOLEAN     (PII 脱敏完成)
  filtered_at     TIMESTAMP
  dt              STRING      (分区键)
  source          STRING      (分区键)
```

#### SILVER 层

```
silver_all_sources (Iceberg 表):
  -- 基础字段
  id                UUID
  content           STRING
  -- 分类与标签
  source            STRING      (github / web_crawl / api_doc / curated)
  content_type      STRING      (code / natural_text / structured / mixed)
  lang              STRING      (zh / en / multilingual / code)
  lang_detail       STRING
  quality_level     STRING      (raw / filtered / high_potential) -- 可 NULL
  tags              ARRAY<STRING> -- 可 NULL, ≤ 10 个
  -- 重要性
  importance        FLOAT       -- 可 NULL
  authority_score   FLOAT       -- 可 NULL
  density_score     FLOAT       -- 可 NULL
  -- 标识与来源
  hash              STRING      (SHA256)
  license           STRING      -- 可 NULL
  char_length       INT
  token_count       INT
  url               STRING      -- 可 NULL
  fetch_ts          TIMESTAMP
  repo_id           UUID        -- 可 NULL, GitHub 数据关联到 repo
  -- 去重
  dup_status        STRING      (unique / dup / pending)
  dup_of            UUID        -- 可 NULL
  -- 评测
  ppl_score         FLOAT       -- 可 NULL
  class_pred        STRING      -- 可 NULL
  quality_score     FLOAT       -- 可 NULL  (L2 文本质量模型)
  llm_judge_score   FLOAT       -- 可 NULL
  human_score       FLOAT       -- 可 NULL
  quality_status    STRING      -- 可 NULL
  evaluated_at      TIMESTAMP   -- 可 NULL
  -- 版本化
  pipeline_version  STRING
  snapshot_ts       TIMESTAMP
  -- 分区
  dt                STRING      (分区键: YYYY-MM-DD)
  source            STRING      (分区键, Phase 1)
  -- 扩展
  metadata_json     JSON        -- 可 NULL, 预留扩展
```

**注意**：`content_emb` 和 `summary_emb` 不存于 Iceberg 表中，而是存于 Lance。

#### SILVER_EMB 层（Lance）

```
Lance Dataset: silver_emb__{cluster_name}
  id              STRING      (与 silver 表的 id 一致)
  content_emb     FIXED_SIZE_LIST<FLOAT>[384]  (Phase 1: 384d)
  summary_emb     FIXED_SIZE_LIST<FLOAT>[384]  -- 可 NULL
  emb_model_ver   STRING      (embedding 模型版本)
  emb_ts          TIMESTAMP

  -- 每个 Lance dataset 独立建 ANN 索引:
  --   column: content_emb
  --   index_type: IVF_PQ (Phase 1) / DISKANN (Phase 2)
  --   metric: cosine
```

Lance dataset 命名规则：`silver_emb__{source}__{content_type}__{lang}`

例如：
- `silver_emb__github__code__code`
- `silver_emb__web_crawl__natural_text__en`
- `silver_emb__api_doc__structured__multilingual`

#### GOLD 层

```
gold_{dataset_name}:
  id              UUID
  text            STRING      (最终文本内容)
  meta            JSON        (完整的元数据: 来源/质量分/embedding信息/评测结果/...)
  dt              STRING      (分区键)
```

**`meta` JSON 结构**（包含 silver 层所有字段的投影）：

```json
{
  "source": "github",
  "content_type": "code",
  "lang": "python",
  "lang_detail": "python",
  "quality_level": "high_potential",
  "importance": 0.85,
  "hash": "abc123...",
  "license": "MIT",
  "tags": ["algorithms", "data_structure"],
  "url": "https://github.com/xxx/yyy/blob/main/zzz.py",
  "quality_score": 87.3,
  "ppl_score": 12.5,
  "dup_status": "unique",
  "emb_cluster": "github__code__code",
  "pipeline_version": "v1.2.3",
  "iceberg_snapshot_id": "7520426094023953954",
  "lance_dataset_version": "v12",
  "repo_score": 0.82
}
```

---

## 5. 增量数据管理

### 5.1 追加式写入 vs Compaction

```
数据到达 (每天 TB 级)
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  热数据 (近 30 天): 追加式写入 (Append-Only)           │
│  - 新数据独立处理，写入新 partition (按 dt 分区)       │
│  - 不触碰已处理的历史数据                              │
│  - 去重/合并延迟到读阶段或定期 Compaction               │
│  存储: S3 Standard                                    │
└─────────────────────────────────────────────────────┘
    │
    │ (30 天后)
    ▼
┌─────────────────────────────────────────────────────┐
│  冷数据 (超过 30 天): 定期 Compaction                  │
│  - 合并小文件、重建 LSH 索引                           │
│  - 迁移到 S3 Standard-IA                              │
│  - 不阻塞读取 (Iceberg snapshot 隔离)                  │
└─────────────────────────────────────────────────────┘
```

### 5.2 Compaction 调度策略

| 级别 | 触发条件 | 操作 | 时间窗口 | 资源 |
|---|---|---|---|---|
| **微小合并** | 单分区文件数 > 100 或单文件 < 128MB | 合并同一天同一 source 的小文件 | 每天凌晨 03:00 | 轻量 Spark job |
| **每周中合并** | 过去 7 天分区有微小文件碎片 | 合并 7 天内的文件为目标大小（256-512MB） | 每周日 03:00 | 中量 Spark job |
| **月度大合并** | 30 天以上数据迁移为冷分区 | 合并 30 天前的分区，重建 LSH 索引，迁移到 IA | 每月第一个周日 02:00 | 重量 Spark job（全量重写历史分区） |

**文件目标大小**：
- Parquet 文件：256-512MB（Iceberg 推荐大小，平衡并行度和元数据量）
- Lance 文件：512MB-1GB（embedding 列宽，更大文件有利于 ANN 索引效率）

**Compaction 不阻塞读取**：利用 Iceberg 的 optimistic concurrency 和 snapshot 隔离：
1. Compaction 写入新的文件 → 创建新的 snapshot
2. 旧 snapshot 仍然可读
3. Compaction 完成后，下游自动切换到新 snapshot
4. 旧文件由 `expire_snapshots` 清理

### 5.3 增量处理流水线

详见 [03-processing-pipeline.md §9](./03-processing-pipeline.md#9-增量数据合并策略)

```
每天定时触发 (02:00 UTC):

  ┌─ 1. 从 raw 表读取过去 24 小时的新 partition
  │    (Spark: SELECT * FROM raw_* WHERE dt = '{yesterday}')
  │
  ├─ 2. 依次通过 粗过滤 → 粗加工 → 细加工 → 细过滤 → 评测
  │    (Spark + Ray 混合)
  │
  ├─ 3. 写入对应层的当天 partition
  │    (append: INSERT INTO bronze_* PARTITION (dt='{yesterday}'))
  │
  ├─ 4. 更新 Iceberg snapshot
  │    (自动，每次 commit 生成新 snapshot)
  │
  └─ 5. 更新 Lance embedding + ANN 索引
       (Ray: 生成新 embedding → 追加到 Lance dataset → 增量更新 ANN 索引)
```

### 5.4 数据重处理与回溯

当粗过滤/粗加工/评测的规则或模型升级时，需要对历史数据重新处理：

```
重处理触发条件:
  - PII 脱敏规则升级 (新的 PII 类型被定义)
  - 去重阈值调整 (Jaccard 从 0.8 改为 0.85)
  - 分类/标签体系变更 (新增/合并标签)
  - 质量模型升级 (BERT → DeBERTa-v3)
  - Embedding 模型升级 (384d → 768d)

重处理流程:
  1. 从指定的 Iceberg snapshot 读取数据
     spark.read.option("snapshot-id", "7520426094023953954").table("silver_all_sources")

  2. 重新运行目标阶段及所有下游阶段

  3. 生成新的 snapshot
     (append 模式写入新 partition，overwrite 不推荐)

  4. 旧 snapshot 保留（time travel），新 snapshot 成为当前版本

  5. 通知下游消费者（训练团队）数据版本已更新
     - 通过 gold 层的数据集版本号
     - 更新实验管理系统中的 dataset tag
```

---

## 6. Snapshot 与时间旅行

### 6.1 Snapshot 保留策略

**A + B 混合策略**：

| 规则 | 适用对象 | 保留周期 | 触发方式 |
|---|---|---|---|
| **自动保留** | 所有常规写入的 snapshot | 保留近 90 天全部 snapshot | 每天自动 `expire_snapshots` |
| **里程碑保留** | 每次训练数据集发布时的 snapshot | **永久保留** | 数据发布时手动标记 |
| **里程碑保留** | 每次重处理完成时的 snapshot | **永久保留** | 重处理完成时自动标记 |
| **周快照保留** | 超过 90 天的 snapshot | 保留每周一个（每周日） | `expire_snapshots` 策略自动执行 |

**Iceberg expire_snapshots 配置**：

```sql
-- 每天执行（Dagster/DolphinScheduler 调度）
CALL catalog.system.expire_snapshots(
  table => 'silver_all_sources',
  older_than => TIMESTAMP '2026-03-15 00:00:00',  -- 90 天前
  retain_last => 1,                                 -- 每个分支至少保留 1 个
  max_concurrent_deletes => 4
);

-- 里程碑 snapshot 保护
-- 在 tagging 时用 retain_last 保护
```

### 6.2 血缘追踪实现

**血缘链路**：

```
模型 Checkpoint
  └─ W&B/MLflow run metadata
       └─ gold dataset version (e.g., "gold_pretrain_v1@snap_20260615")
            └─ silver snapshot ID (e.g., "7520426094023953954")
                 └─ bronze snapshot ID
                      └─ raw snapshot ID
                           └─ 原始 URL + fetch_ts
```

**实现方式**：

1. **Iceberg 层**：利用 Iceberg 的 `snapshot_id` 和 `history()` 函数追溯表变更历史
2. **Lance 层**：在 Iceberg snapshot metadata 的 `summary` 中记录 Lance dataset version
3. **跨层关联**：通过 pipeline 执行的元数据表记录每次处理的输入/输出 snapshot

**Pipeline 执行元数据表**（血缘核心）：

```sql
CREATE TABLE pipeline_executions (
  execution_id    UUID PRIMARY KEY,
  pipeline_name   STRING,        -- "daily_incremental_v1" / "reprocess_v2"
  pipeline_version STRING,
  input_snapshot  MAP<STRING, STRING>,  -- {"bronze_github_code": "snap_001", ...}
  output_snapshot MAP<STRING, STRING>,  -- {"silver_all_sources": "snap_002", ...}
  lance_versions  MAP<STRING, STRING>,  -- {"github_code_cluster": "v12", ...}
  params_json     JSON,          -- 处理参数
  status          STRING,        -- running / success / failed
  started_at      TIMESTAMP,
  completed_at    TIMESTAMP,
  triggered_by    STRING         -- "daily_schedule" / "manual_reprocess"
);
```

---

## 7. S3 存储生命周期

### 7.1 按表分层策略

| Iceberg 表 / Lance Dataset | 写入后自动过渡 | 最终存储类别 | 理由 |
|---|---|---|---|
| **raw_*** | 30 天后 → Glacier Instant Retrieval → 365 天后 → Deep Archive | **Glacier Instant Retrieval → Deep Archive** | 原始数据几乎不直接读取，保留用于合规和审计。取回延迟可接受（分钟级） |
| **bronze_*** | 90 天后 → Standard-IA | **Standard-IA** | 偶尔在重处理时读取，毫秒级取回 |
| **silver_all_sources**（近 30 天分区） | 保持 Standard | **Standard** | 日常加工、检索、查询频繁 |
| **silver_all_sources**（30 天以上分区） | 30 天后 → Standard-IA | **Standard-IA** | 历史数据查询频率低 |
| **silver_emb/*** (Lance) | 保持 Standard | **Standard** | ANN 检索延迟敏感，不可降级 |
| **gold_*** | 保持 Standard | **Standard** | 训练团队频繁读取 |

### 7.2 生命周期规则配置

```
S3 Bucket Lifecycle Rules:

规则 1: raw 表自动降级
  前缀: raw/
  操作:
    30 天后 → 转换为 Glacier Instant Retrieval
    365 天后 → 转换为 Glacier Deep Archive
    10 年后 → 过期删除（与 raw 数据保留 10 年一致）

规则 2: bronze 表自动降级
  前缀: bronze/
  操作:
    90 天后 → 转换为 Standard-IA

规则 3: silver 历史分区降级
  前缀: silver/
  操作:
    30 天后（按对象最后修改时间） → 转换为 Standard-IA

规则 4: silver_emb/ 和 gold/ 不降级
  前缀: silver_emb/, gold/
  操作: 无（保持 Standard）

规则 5: 未完成 multipart upload 清理
  前缀: *
  操作: 7 天后删除未完成的 multipart upload parts（节约成本）
```

### 7.3 Raw 数据保留 10 年的成本影响

| 年份 | 数据累积量 | 存储类别 | 年成本估算 |
|---|---|---|---|
| 第 1 年 | ~50PB（写入） | Standard → Glacier Instant Retrieval → Deep Archive | ~$500K（Standard 短周期） |
| 第 3 年 | ~150PB 累积 | 大部分在 Deep Archive | ~$150K/年 |
| 第 5 年 | ~250PB 累积 | 大部分在 Deep Archive | ~$250K/年 |
| 第 10 年 | ~500PB 累积 | 大部分在 Deep Archive | ~$500K/年 |

> 注：以上为粗略估算，以 S3 Standard ~$0.023/GB/月、Deep Archive ~$0.001/GB/月 为基准。取回费用另计。

---

## 8. 权限与安全

| 层面 | 策略 | 实现方式 |
|---|---|---|
| **S3 访问控制** | 基于 IAM Role 的最小权限原则 | Spark/Ray 的 Service Role 仅允许读写特定 bucket/前缀 |
| **Catalog 访问** | REST Catalog 内建认证 | 读写分离：数据团队可以写，训练团队只读 |
| **数据加密** | S3 服务端加密（SSE-S3 或 SSE-KMS） | Bucket 级别默认加密 |
| **PII 数据** | 粗过滤阶段已脱敏，raw 层原始数据加密但不可直接读 | PII 审计日志记录脱敏位置，但不存原始 PII |
| **审计日志** | 所有数据访问记录到 CloudTrail / S3 access logs | S3 access logs 开启，定期导出分析 |

---

## 9. 运维与监控

### 9.1 关键运维指标

| 指标 | 目标 | 监控方式 |
|---|---|---|
| **S3 存储用量** | 按表/layer 拆分 | CloudWatch / S3 metrics + Grafana |
| **Snapshot 数量** | 单表 < 1000 | Iceberg `system.snapshots` 表 + 定期巡检 |
| **孤儿文件（Orphan Files）** | < 1% 存储 | Iceberg `remove_orphan_files` + 每周巡检 |
| **Compaction 效果** | 单分区文件数 < 50 | Compaction job 日志 + 分区文件计数 |
| **Catalog 可用性** | > 99.9% | REST Catalog health check + K8s 探活 |
| **S3 请求延迟** | P99 < 100ms | S3 metrics + CloudWatch |
| **Lance ANN 索引新鲜度** | 每日更新 | 对比最新 silver snapshot 时间和 Lance dataset 更新时间 |

### 9.2 日常运维任务

| 任务 | 频率 | 工具/命令 |
|---|---|---|
| **Snapshot 过期清理** | 每天 | `CALL system.expire_snapshots(...)` |
| **孤儿文件清理** | 每周 | `CALL system.remove_orphan_files(...)` |
| **元数据压缩** | 每周（如果 manifest 数量 > 1000） | `CALL system.rewrite_manifests(...)` |
| **Compaction** | 每天微小 / 每周中 / 每月大 | Spark job |
| **Lance 索引重建** | 每月 | Lance `optimize()` + `compact_files()` |
| **存储成本报告** | 每月 | S3 cost explorer + 自定义报表 |
| **Catalog 元数据备份** | 每天 | PostgreSQL 备份（REST Catalog 后端） |

### 9.3 告警规则

| 告警 | 条件 | 严重级别 |
|---|---|---|
| 单日数据增量暴跌（< 正常值的 50%） | bronze 层写入行数 < 阈值 | 🔴 P0 |
| Compaction 失败 | Compaction job 连续 3 次失败 | 🟡 P1 |
| Snapshot 数超限 | 单表 snapshot 数 > 1000 | 🟡 P1 |
| 存储成本异常增长 | 月度存储费用环比增长 > 30% | 🟡 P1 |
| REST Catalog 不可用 | Health check 失败 > 5min | 🔴 P0 |
| S3 读写错误率上升 | 5xx 错误率 > 1% | 🔴 P0 |
| Lance ANN 索引滞后 > 3 天 | 最新 silver snapshot 时间 - Lance 更新时间 > 3d | 🟡 P1 |