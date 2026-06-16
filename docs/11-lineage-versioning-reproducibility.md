# 数据血缘、版本管理与可复现实验设计

> 面向 70B 基座模型训练的数据版本、配方版本、质量版本与 checkpoint 追溯体系

---

## 目录

- [1. 目标与定位](#1-目标与定位)
- [2. 为什么血缘与可复现是核心能力](#2-为什么血缘与可复现是核心能力)
- [3. 核心对象与版本定义](#3-核心对象与版本定义)
- [4. Dataset Version 设计](#4-dataset-version-设计)
- [5. Manifest 设计](#5-manifest-设计)
- [6. 训练 Checkpoint 关联](#6-训练-checkpoint-关联)
- [7. 全链路血缘模型](#7-全链路血缘模型)
- [8. 数据 Diff 与变更分析](#8-数据-diff-与变更分析)
- [9. 回滚与问题定位](#9-回滚与问题定位)
- [10. 可复现实验流程](#10-可复现实验流程)
- [11. 表结构与元数据设计](#11-表结构与元数据设计)
- [12. 与训练团队和平台系统的接口](#12-与训练团队和平台系统的接口)
- [13. Phase 2/3 演进方向](#13-phase-23-演进方向)
- [14. 关键结论](#14-关键结论)

---

## 1. 目标与定位

本文档设计训练数据链路中的**血缘、版本管理与可复现实验体系**。

核心目标：

> 任意一个模型 checkpoint，都能追溯到它使用的 dataset version、mixture recipe、质量评估版本、处理规则版本、Iceberg snapshot，以及最终原始 URL / repo / 文件。

换句话说，要能回答：

```text
这个模型是用哪批数据训练出来的？
这批数据是按什么配方采样的？
这些数据经过了哪些处理？
质量分和过滤规则是什么版本？
如果模型异常，能否定位到问题来源？
半年后能否复现同一份训练数据？
```

### 1.1 本文档承接哪些前置设计

| 前置文档 | 关联点 |
|---|---|
| `02-data-lake.md` | Iceberg snapshot、Medallion 分层、S3 生命周期 |
| `03-processing-pipeline.md` | raw/bronze/silver/gold 多阶段处理 |
| `09-data-quality-evaluation.md` | quality_pool、overall_quality_score、data_quality_scores |
| `10-data-mixture-strategy.md` | mixture_recipe_version、dataset manifest |
| `08-training-overview.md` | checkpoint、训练状态、评估与上线 |

---

## 2. 为什么血缘与可复现是核心能力

### 2.1 训练异常必须能定位

典型问题：

```text
checkpoint_50000 之后 validation loss 异常上升
HumanEval 分数虚高
中文能力突然下降
模型生成了疑似隐私信息
某类 benchmark 结果不可解释
```

没有血缘时，只能猜测原因。

有血缘时，可以追溯：

```text
checkpoint → dataset_version → recipe_version → selected shards → gold snapshot → silver/bronze/raw → 原始 URL
```

### 2.2 合规审计必须能删除和回溯

当出现：

```text
版权投诉
隐私投诉
License 争议
安全数据误入
```

必须能回答：

```text
该数据是否进入过训练？
进入了哪个 dataset_version？
影响了哪些 checkpoint？
后续版本是否已经移除？
```

### 2.3 实验复现必须依赖版本闭环

训练效果对以下变量高度敏感：

```text
数据集版本
配方版本
tokenizer 版本
质量评估版本
过滤规则版本
shuffle seed
shard 顺序
训练框架版本
```

如果只保存“数据目录路径”，无法复现实验。

---

## 3. 核心对象与版本定义

### 3.1 核心对象

| 对象 | 含义 |
|---|---|
| `raw_snapshot_id` | 原始采集层 Iceberg snapshot |
| `bronze_snapshot_id` | 粗过滤后 snapshot |
| `silver_snapshot_id` | 加工、去重、质量评价后 snapshot |
| `gold_snapshot_id` | 可交付训练候选数据 snapshot |
| `quality_pipeline_version` | 质量评价规则/模型版本 |
| `mixture_recipe_version` | 数据配方版本 |
| `dataset_version` | 最终训练数据集版本 |
| `dataset_manifest_id` | 数据集清单版本 |
| `training_run_id` | 一次训练任务 ID |
| `checkpoint_id` | 某个模型 checkpoint ID |

### 3.2 版本之间的关系

```text
raw_snapshot
  ↓
bronze_snapshot
  ↓
silver_snapshot + quality_pipeline_version
  ↓
gold_snapshot
  ↓
mixture_recipe_version
  ↓
dataset_version + manifest
  ↓
training_run_id
  ↓
checkpoint_id
```

### 3.3 版本不可变原则

核心原则：

```text
任何已用于训练的 dataset_version、manifest、recipe 都不可原地修改
```

如果需要修正：

```text
创建新版本
保留旧版本
通过 supersedes 字段建立替代关系
```

---

## 4. Dataset Version 设计

### 4.1 Dataset Version 是什么

`dataset_version` 是训练团队实际消费的一份数据集版本。

它不是单个 Iceberg snapshot，也不是单个目录，而是以下内容的组合：

```text
Gold 数据 snapshot
质量评估版本
配方版本
采样结果
shard 文件列表
tokenizer 估算版本
shuffle 策略
manifest
```

### 4.2 命名规范

建议格式：

```text
{task_type}_{model_scale}_{recipe_name}_{date}_{version}
```

示例：

```text
pretrain_70b_base_v1_20260616_v1
pretrain_70b_code_heavy_20260701_v2
continue_70b_zh_enhanced_20260715_v1
sft_70b_api_usage_20260801_v1
```

### 4.3 Dataset Version 元数据

必须记录：

```yaml
dataset_version: pretrain_70b_base_v1_20260616_v1
task_type: pretrain
model_scale: 70B
created_at: 2026-06-16T10:00:00Z
created_by: data_team

snapshots:
  raw_snapshot_id: ...
  bronze_snapshot_id: ...
  silver_snapshot_id: ...
  gold_snapshot_id: ...

quality:
  quality_pipeline_version: quality_v1.0
  quality_model_version: distilbert_quality_v0.3

mixture:
  mixture_recipe_version: base_pretrain_v1.0_20260616
  unit: estimated_tokens
  target_tokens: 1.2T

format:
  output_format: jsonl.zst
  shard_count: 4096
  avg_shard_size: 1GB

training:
  tokenizer_version: qwen_tokenizer_vX_or_tbd
  max_seq_len: 8192
  shuffle_seed: 20260616
```

---

## 5. Manifest 设计

### 5.1 Manifest 的作用

Manifest 是数据集版本的物理清单。

它回答：

```text
训练实际读取了哪些文件？
每个文件包含哪些数据？
每个 shard 的 token 数是多少？
它来自哪个 snapshot？
```

### 5.2 Manifest 分层

建议两级 manifest：

```text
dataset_manifest.json
  └── shard_manifest_000001.json
  └── shard_manifest_000002.json
  └── ...
```

### 5.3 dataset_manifest 示例

```json
{
  "dataset_version": "pretrain_70b_base_v1_20260616_v1",
  "manifest_id": "manifest_20260616_001",
  "mixture_recipe_version": "base_pretrain_v1.0_20260616",
  "quality_pipeline_version": "quality_v1.0",
  "gold_snapshot_id": "7520426094023953954",
  "total_items": 1234567890,
  "total_estimated_tokens": 1200000000000,
  "shard_count": 4096,
  "format": "jsonl.zst",
  "shuffle_seed": 20260616,
  "created_at": "2026-06-16T10:00:00Z"
}
```

### 5.4 shard_manifest 示例

```json
{
  "shard_id": "shard_000001",
  "path": "s3://bucket/datasets/pretrain_70b_base_v1/shard_000001.jsonl.zst",
  "item_count": 300000,
  "estimated_tokens": 293000000,
  "source_distribution": {
    "github": 0.35,
    "web_crawl": 0.40,
    "api_doc": 0.15,
    "issue_pr": 0.10
  },
  "quality_pool_distribution": {
    "candidate": 0.85,
    "premium": 0.15
  },
  "source_snapshots": {
    "gold_snapshot_id": "7520426094023953954"
  },
  "checksum": "sha256:..."
}
```

### 5.5 Manifest 必须不可变

一旦训练使用，manifest 不允许覆盖。

如需修正：

```text
创建新 manifest_id
创建新 dataset_version
```

---

## 6. 训练 Checkpoint 关联

### 6.1 Checkpoint 必须记录的数据字段

训练 checkpoint 的 metadata 中必须记录：

```json
{
  "checkpoint_id": "checkpoint_step_50000",
  "training_run_id": "train_70b_20260620_001",
  "dataset_version": "pretrain_70b_base_v1_20260616_v1",
  "manifest_id": "manifest_20260616_001",
  "mixture_recipe_version": "base_pretrain_v1.0_20260616",
  "quality_pipeline_version": "quality_v1.0",
  "tokenizer_version": "tokenizer_v1",
  "global_step": 50000,
  "tokens_consumed": 500000000000,
  "epoch": 0.42,
  "data_cursor": {
    "shard_id": "shard_1702",
    "offset": 12345678
  }
}
```

### 6.2 为什么要记录 data_cursor

如果训练中途异常，需要知道：

```text
当前训练读到了哪个 shard
读到了哪个 offset
最近一段时间消费了哪些数据
```

这有助于排查：

```text
某个 shard 是否损坏
某类数据是否导致 loss spike
某批数据是否包含异常内容
```

### 6.3 checkpoint 到原始 URL 的追溯

追溯链路：

```text
checkpoint_id
  → training_run_id
  → dataset_version
  → manifest_id
  → shard_id / item_id
  → gold_snapshot_id
  → silver_snapshot_id
  → bronze_snapshot_id
  → raw_snapshot_id
  → raw_url / repo / file_path / commit_sha
```

---

## 7. 全链路血缘模型

### 7.1 血缘粒度

Phase 1 支持：

```text
dataset_version 级
shard 级
item/document 级
```

Phase 2 扩展：

```text
chunk/function/commit 级
```

### 7.2 item 级血缘字段

每条数据至少应包含：

```text
item_id
raw_item_id
source
source_url
repo_id / site_id
file_path
commit_sha
raw_snapshot_id
bronze_snapshot_id
silver_snapshot_id
gold_snapshot_id
processing_pipeline_version
quality_pipeline_version
```

### 7.3 pipeline 执行血缘

每次 pipeline 执行记录：

```text
pipeline_run_id
input_snapshot_id
output_snapshot_id
pipeline_version
rule_version
model_version
started_at
finished_at
status
metrics
```

### 7.4 处理参数血缘

必须记录：

```text
去重阈值
过滤规则版本
PII 规则版本
Embedding 模型版本
质量模型版本
LLM Judge prompt 版本
数据配方版本
```

---

## 8. 数据 Diff 与变更分析

### 8.1 为什么需要 dataset diff

训练团队会问：

```text
v2 比 v1 多了什么？
v2 删除了哪些数据？
质量分布有什么变化？
代码数据比例有没有变化？
中文数据增加了多少？
```

### 8.2 Diff 类型

| Diff 类型 | 说明 |
|---|---|
| item diff | 新增/删除/保留 item |
| token diff | 各维度 token 数变化 |
| quality diff | quality_pool 与 score 分布变化 |
| source diff | 来源分布变化 |
| language diff | 语言分布变化 |
| recipe diff | 配方比例变化 |
| rule diff | 规则/模型版本变化 |

### 8.3 Diff 输出示例

```yaml
from_dataset: pretrain_70b_base_v1
to_dataset: pretrain_70b_base_v2

summary:
  added_items: 120M
  removed_items: 30M
  unchanged_items: 900M
  token_delta: +180B

quality_pool_delta:
  candidate: +150B tokens
  premium: +30B tokens

source_delta:
  github: +80B
  api_doc: +40B
  web_crawl: +60B

rule_changes:
  dedup_threshold: 0.80 -> 0.82
  quality_pipeline_version: quality_v1.0 -> quality_v1.1
```

### 8.4 Diff 的使用场景

```text
训练前 review
训练异常排查
效果变化解释
合规删除影响评估
版本发布说明
```

---

## 9. 回滚与问题定位

### 9.1 数据回滚

如果发现某个 dataset_version 有问题：

```text
停止使用该 dataset_version
标记 deprecated
创建修复后的新 dataset_version
保留旧版本供审计
```

不能直接覆盖旧数据。

### 9.2 问题定位流程

示例：checkpoint loss spike。

```text
1. 定位异常 checkpoint_id 和 step 范围
2. 读取 checkpoint metadata 中的 dataset_version 和 data_cursor
3. 找到对应 shard 范围
4. 分析 shard 的 source/content_type/language/quality_pool 分布
5. 抽样检查异常 shard 的 item
6. 回溯到 gold/silver/raw snapshot
7. 判断是数据源问题、处理规则问题还是训练读取问题
```

### 9.3 合规删除流程

当某条原始数据需要删除：

```text
1. 根据 source_url / repo / file_path 找到 raw_item_id
2. 查询所有关联 item_id
3. 查询进入过哪些 dataset_version
4. 查询影响哪些 training_run / checkpoint
5. 后续 dataset_version 中排除该数据
6. 记录删除审计
```

### 9.4 污染数据修复

如果发现 benchmark 泄漏或污染数据：

```text
标记 contamination_reason
从后续 gold snapshot 排除
生成 dataset diff
通知训练团队影响范围
必要时重新训练或继续训练修复
```

---

## 10. 可复现实验流程

### 10.1 可复现输入

复现实验至少需要：

```text
dataset_version
manifest_id
mixture_recipe_version
quality_pipeline_version
processing_pipeline_version
tokenizer_version
shuffle_seed
training_config_version
code_commit
container_image
```

### 10.2 数据重放流程

```text
1. 读取 dataset_version metadata
2. 校验 manifest checksum
3. 按 manifest 拉取 shard
4. 使用相同 tokenizer_version
5. 使用相同 shuffle_seed / shard order
6. 启动相同 training_config
7. 比对 tokens_consumed 与 loss 曲线
```

### 10.3 可复现等级

| 等级 | 能力 |
|---|---|
| L1 数据可复现 | 能拿到同一份 shard 列表 |
| L2 采样可复现 | 能按同一配方和 seed 重放样本顺序 |
| L3 训练近似复现 | 能得到相近 loss/benchmark |
| L4 bit-level 复现 | 完全相同结果，通常成本高且不强求 |

Phase 1 目标：

```text
L1 + L2
```

Phase 2/3 逐步接近 L3。

---

## 11. 表结构与元数据设计

### 11.1 dataset_versions 表

```sql
CREATE TABLE dataset_versions (
  dataset_version STRING,
  task_type STRING,
  model_scale STRING,
  manifest_id STRING,
  mixture_recipe_version STRING,
  quality_pipeline_version STRING,
  processing_pipeline_version STRING,
  gold_snapshot_id STRING,
  total_items BIGINT,
  total_estimated_tokens BIGINT,
  shard_count INT,
  output_format STRING,
  shuffle_seed BIGINT,
  status STRING,
  created_at TIMESTAMP,
  created_by STRING
);
```

### 11.2 dataset_manifests 表

```sql
CREATE TABLE dataset_manifests (
  manifest_id STRING,
  dataset_version STRING,
  shard_id STRING,
  shard_path STRING,
  item_count BIGINT,
  estimated_tokens BIGINT,
  checksum STRING,
  source_distribution MAP<STRING, DOUBLE>,
  content_type_distribution MAP<STRING, DOUBLE>,
  language_distribution MAP<STRING, DOUBLE>,
  quality_pool_distribution MAP<STRING, DOUBLE>,
  created_at TIMESTAMP
);
```

### 11.3 item_lineage 表

```sql
CREATE TABLE item_lineage (
  item_id STRING,
  raw_item_id STRING,
  source STRING,
  source_url STRING,
  repo_id STRING,
  file_path STRING,
  commit_sha STRING,
  raw_snapshot_id STRING,
  bronze_snapshot_id STRING,
  silver_snapshot_id STRING,
  gold_snapshot_id STRING,
  processing_pipeline_version STRING,
  quality_pipeline_version STRING,
  created_at TIMESTAMP
);
```

### 11.4 training_dataset_links 表

```sql
CREATE TABLE training_dataset_links (
  training_run_id STRING,
  checkpoint_id STRING,
  dataset_version STRING,
  manifest_id STRING,
  mixture_recipe_version STRING,
  tokenizer_version STRING,
  global_step BIGINT,
  tokens_consumed BIGINT,
  epoch DOUBLE,
  shard_id STRING,
  shard_offset BIGINT,
  created_at TIMESTAMP
);
```

### 11.5 dataset_diffs 表

```sql
CREATE TABLE dataset_diffs (
  diff_id STRING,
  from_dataset_version STRING,
  to_dataset_version STRING,
  added_items BIGINT,
  removed_items BIGINT,
  unchanged_items BIGINT,
  token_delta BIGINT,
  quality_pool_delta MAP<STRING, BIGINT>,
  source_delta MAP<STRING, BIGINT>,
  language_delta MAP<STRING, BIGINT>,
  rule_changes ARRAY<STRING>,
  created_at TIMESTAMP
);
```

---

## 12. 与训练团队和平台系统的接口

### 12.1 数据团队交付

数据团队交付：

```text
dataset_version
manifest_id
mixture_recipe_version
shard 路径
checksum
质量分布报告
dataset diff 报告
已知风险说明
```

### 12.2 训练团队回写

训练团队必须回写：

```text
training_run_id
checkpoint_id
dataset_version
manifest_id
mixture_recipe_version
tokenizer_version
tokens_consumed
data_cursor
benchmark_result
```

### 12.3 实验管理系统集成

需要与以下系统打通：

```text
W&B / MLflow / 内部实验平台
Checkpoint 存储系统
训练日志系统
模型评估系统
Dataset Registry
```

### 12.4 查询接口示例

```text
GET /datasets/{dataset_version}
GET /datasets/{dataset_version}/manifest
GET /datasets/{dataset_version}/diff?from=...
GET /checkpoints/{checkpoint_id}/lineage
GET /items/{item_id}/lineage
GET /source-url/search?url=...
```

---

## 13. Phase 2/3 演进方向

### 13.1 Phase 2：更细粒度血缘

扩展到：

```text
chunk 级
function/class 级
commit diff 级
issue-comment-code 链路级
```

### 13.2 Phase 2：Dataset Registry

建设数据集注册中心：

```text
版本查询
manifest 查询
质量分布预览
diff 可视化
权限管理
废弃版本标记
```

### 13.3 Phase 3：训练效果归因

将训练效果与数据维度关联：

```text
某类数据增加 → benchmark 变化
某 source 下调 → loss 变化
某语言增强 → 对应评估变化
某 bad case → 追溯缺失数据类型
```

长期目标：

```text
checkpoint 不只是模型快照
也是数据版本、配方版本和质量版本的联合快照
```

---

## 14. 关键结论

1. `dataset_version` 不是单个目录，而是 snapshot、质量版本、配方版本、manifest 和输出 shard 的组合。
2. 所有已用于训练的 dataset、manifest、recipe 必须不可变。
3. 每个 checkpoint 必须记录 `dataset_version`、`manifest_id`、`mixture_recipe_version` 和 `tokens_consumed`。
4. 追溯链路必须支持从 checkpoint 回到原始 URL / repo / file。
5. Manifest 是训练数据可复现的核心物理清单，必须保存 checksum 和 shard 分布。
6. Dataset diff 是解释模型效果变化、合规删除影响和版本发布的关键工具。
7. Phase 1 目标是做到数据可复现 L1/L2：同一 shard 列表、同一采样顺序。
8. Phase 2/3 再扩展到 chunk/function/commit 级血缘和训练效果归因。
9. 数据团队和训练团队必须约定 metadata 回写接口，否则 checkpoint 无法真正追溯。
10. 长期来看，checkpoint 应被视为模型参数、数据版本、配方版本、质量版本的联合快照。
