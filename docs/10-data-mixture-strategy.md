# 数据配比与训练数据配方设计

> 连接数据资产、质量评价与模型训练的数据配方方法论

---

## 目录

- [1. 目标与定位](#1-目标与定位)
- [2. 为什么需要数据配方](#2-为什么需要数据配方)
- [3. 配比基本单位](#3-配比基本单位)
- [4. 配方维度设计](#4-配方维度设计)
- [5. Phase 1 初始配方策略](#5-phase-1-初始配方策略)
- [6. quality_pool 的使用策略](#6-quality_pool-的使用策略)
- [7. 多语言与代码数据配比](#7-多语言与代码数据配比)
- [8. 动态配比调整机制](#8-动态配比调整机制)
- [9. 训练反馈如何反向调整配方](#9-训练反馈如何反向调整配方)
- [10. 过拟合与高质量数据不足风险控制](#10-过拟合与高质量数据不足风险控制)
- [11. 配方版本管理](#11-配方版本管理)
- [12. 与训练团队的接口](#12-与训练团队的接口)
- [13. Phase 2/3 演进方向](#13-phase-23-演进方向)
- [14. 关键结论](#14-关键结论)

---

## 1. 目标与定位

本文档设计大模型训练数据的**配比与配方机制**，解决从数据资产到训练输入之间的关键问题：

> 已经经过质量评价的数据，如何被组合成一份可训练、可解释、可复现、可调优的数据配方？

前置文档已经回答：

```text
01：有哪些数据源
03：如何加工处理
09：如何判断质量和分池
08：训练过程如何消费数据
```

本文档负责连接这些能力：

```text
数据资产 → 质量池 → 多维配方 → 训练数据集 → 训练反馈 → 配方调整
```

### 1.1 数据配方是什么

数据配方不是简单的“拿多少 GB 数据”，而是一组可执行、可追溯的采样规则：

```text
从哪些 source 取数据
每类 content_type 取多少 token
不同 language 如何分配
candidate_pool 和 premium_pool 如何混合
代码、文档、网页、Issue 如何组合
新数据和历史数据如何 replay
```

### 1.2 本文档解决什么

| 问题 | 本文回答 |
|---|---|
| 配比按什么单位计算 | Phase 1 以 token 为主，GB/document 为辅助，Phase 2 引入 effective token |
| 初始配方怎么定 | 通用基座为主，代码和中文增强 |
| 质量池怎么用 | candidate 为主体，premium 做高质量增强 |
| 如何动态调配方 | 根据 loss、benchmark、bad case、质量监控调整 |
| 如何防止过拟合 | 控制高质量池占比、source 多样性、epoch 与 replay |
| 如何复现 | 每次训练保存 mixture_recipe_version 与 manifest |

---

## 2. 为什么需要数据配方

### 2.1 训练不是直接消耗全部高质量数据

数据链路产出的数据通常远大于训练预算：

```text
100PB 原始数据
  → 5-15PB 评测后候选数据
  → 1-2T tokens 实际训练数据
```

训练团队不会把所有数据都直接喂给模型，而是需要一份配方决定：

```text
哪些数据进训练
进多少
以什么权重进
在什么阶段进
是否重复采样
```

### 2.2 配方决定模型能力结构

不同配方会训练出不同能力倾向的模型：

| 配方倾向 | 可能结果 |
|---|---|
| 代码数据多 | 代码能力强，但自然语言能力可能下降 |
| 中文数据多 | 中文能力强，但英文 benchmark 可能下降 |
| premium 过多 | 高质量但多样性不足，可能过拟合 |
| 网页泛数据过多 | 覆盖广，但噪声上升 |
| Issue/PR 数据多 | 对话和问题解决能力增强，但噪声控制难 |

### 2.3 配方是训练反馈闭环的控制面

当训练指标异常时，最终调整的不是单条数据，而是配方：

```text
HumanEval 低 → 增加高质量代码 / bug-fix / test-rich 数据
C-Eval 低 → 增加中文高质量文档 / 教材 / 题目解析
幻觉严重 → 降低低可靠网页，提高权威文档
过拟合 → 降低重复高质量小池采样权重，扩大来源多样性
```

---

## 3. 配比基本单位

### 3.1 Phase 1：以 token 为主

Phase 1 推荐使用：

```text
token_count_estimate
```

作为主配比单位。

原因：

```text
训练真正消耗的是 token
不同语言相同 GB 产生的 token 数不同
不同内容类型 token 密度不同
GPU 训练预算以 tokens/s 和 total tokens 估算
```

### 3.2 GB / document / sample 作为辅助指标

| 单位 | 用途 | 局限 |
|---|---|---|
| GB | 存储、成本估算 | 不等价于训练量 |
| document | 数据资产管理 | 长短差异大 |
| sample | SFT/RLHF 有用 | pretrain 中样本长度差异大 |
| token | 训练配比主单位 | Phase 1 可能只是估算 |
| effective token | Phase 2/3 目标 | 需要训练反馈校准 |

### 3.3 为什么不能只按 GB 配比

示例：

```text
100GB 英文 ≈ 25B tokens
100GB 中文 ≈ 35B tokens
100GB 代码 ≈ 28B tokens
100GB 低资源语言 ≈ 40B tokens
```

如果按 GB 配比 1:1，实际 token 配比可能严重偏移。

### 3.4 Phase 2：引入 effective token

Phase 2/3 可引入：

```text
effective_token_weight
```

含义：

```text
同样 1 个 token，对训练效果的有效贡献不同
```

影响因素：

```text
quality_pool
source_reliability
content_type
语言稀缺性
重复度
训练反馈收益
```

示例：

```text
effective_tokens = raw_tokens * effective_token_weight
```

Phase 1 不强制使用 effective token，只预留字段。

---

## 4. 配方维度设计

训练配方至少应支持以下维度。

### 4.1 source

```text
github
web_crawl
api_doc
issue_pr_discussion
curated
synthetic
```

source 决定来源可靠性、内容形态和风险。

### 4.2 content_type

```text
code
natural_text
technical_doc
api_doc
dialogue
issue_pr
structured
mixed
```

content_type 决定训练能力方向。

### 4.3 language

```text
zh
en
multilingual
code_language: python/java/js/ts/go/rust/...
```

language 不能只按文档数量配比，必须看 token 占比。

### 4.4 quality_pool

来自 `09-data-quality-evaluation.md`：

```text
candidate_pool
premium_pool
```

默认不交付：

```text
blocked_pool
not_delivered
```

### 4.5 domain

可选维度：

```text
programming
math
science
finance
law
medical
education
product_doc
infra_doc
```

Phase 1 可空，Phase 2 补齐。

### 4.6 token_count

配方计算主字段：

```text
token_count_estimate
```

后续可升级：

```text
token_count_actual
effective_token_count
```

### 4.7 repo/site/source tier

用于避免优质来源过度集中。

示例：

```text
repo_quality_tier: L1 / L2 / L3
site_authority_tier: official / trusted / general / low
```

---

## 5. Phase 1 初始配方策略

### 5.1 总体策略

Phase 1 推荐采用：

```text
通用基座配方为主
代码和中文有明确增强权重
质量池用于增强但不牺牲多样性
```

不建议 Phase 1 做极端能力导向配方，例如：

```text
只强化代码
只强化中文
只使用 premium_pool
只使用权威文档
```

### 5.2 推荐初始配方结构

以下为默认建议，不是固定值。

| 维度 | 建议方向 |
|---|---|
| 自然语言通用文本 | 保持主体，覆盖广泛知识 |
| 技术文档/API 文档 | 明确增强，提升工具/API 理解 |
| 代码数据 | 明确增强，支撑代码能力 |
| Issue/PR/Discussion | 小比例引入，控制噪声 |
| 合成数据 | Phase 1 少量或不进入 pretrain 主体 |
| premium_pool | 作为高质量增强，不作为全部训练集 |

### 5.3 Phase 1 配方示例

示例仅用于说明结构：

```yaml
recipe_name: base_pretrain_v1
unit: estimated_tokens
total_tokens: 1.2T

mixture:
  natural_text:
    target_ratio: 45%
    quality_pool:
      candidate: 85%
      premium: 15%

  code:
    target_ratio: 25%
    quality_pool:
      candidate: 80%
      premium: 20%

  technical_doc_api_doc:
    target_ratio: 20%
    quality_pool:
      candidate: 75%
      premium: 25%

  issue_pr_discussion:
    target_ratio: 5%
    quality_pool:
      candidate: 90%
      premium: 10%

  synthetic:
    target_ratio: 5%
    usage: optional_or_sft_preferred
```

注意：

```text
上述比例是默认建议，用于形成讨论基线
最终比例需要与训练团队根据目标能力和数据量共同确定
```

---

## 6. quality_pool 的使用策略

### 6.1 candidate_pool 是主体

Phase 1 中，`candidate_pool` 应作为训练数据主体。

原因：

```text
覆盖广
多样性高
不容易过拟合
规模足够
```

### 6.2 premium_pool 是增强池

`premium_pool` 用于：

```text
高质量增强
能力专项增强
评估样本分析
高权重采样
```

不建议只用 `premium_pool` 训练。

原因：

```text
premium 仅占 5%-10%
来源可能集中
多样性不足
重复采样会增加过拟合风险
```

### 6.3 推荐使用方式

| 使用场景 | candidate | premium |
|---|---:|---:|
| 通用 pretrain | 主体 | 适度增强 |
| 代码能力增强 | 主体 | 较高增强 |
| 中文能力增强 | 主体 | 中等增强 |
| 小规模试训 | 主体 | 可提高比例 |
| 高质量继续训练 | 降低比例 | 提高比例，但需防过拟合 |

### 6.4 采样权重建议

可以设置：

```text
premium_sample_weight > candidate_sample_weight
```

但需要限制：

```text
单一 source 的最大占比
单一 repo/site 的最大占比
单一 domain 的最大占比
单一 language 的最大占比
```

---

## 7. 多语言与代码数据配比

### 7.1 多语言配比原则

多语言配比不能只看 GB，也不能只看文档数。

推荐使用：

```text
language_token_ratio
```

并监控：

```text
language_document_ratio
language_quality_pool_ratio
language_dedup_ratio
```

### 7.2 中文增强策略

如果目标模型需要较强中文能力，Phase 1 应明确中文增强权重。

建议：

```text
中文 token 占比单独设目标
中文 premium_pool 单独监控
中文网页与中文高质量文档分开配比
```

避免：

```text
中文占比看似高，但大量来自低质量网页
```

### 7.3 代码语言配比

代码数据应按语言维度配比：

```text
python
javascript/typescript
java
go
c/c++
rust
other
```

配比依据：

```text
训练目标
生态重要性
数据质量
可解析率
benchmark 需求
```

### 7.4 代码数据不要只按 repo star 配比

高 star repo 重要，但不能完全支配配方。

需要控制：

```text
单 repo 最大 token 占比
单组织最大 token 占比
高 star repo 与中等 repo 的平衡
框架/库/应用代码的平衡
```

### 7.5 Issue/PR 数据比例控制

Issue/PR/Discussion 数据有价值，但噪声高。

Phase 1 建议小比例引入：

```text
主要用于问题描述、修复链、真实开发对话
不作为大规模自然语言主体
```

---

## 8. 动态配比调整机制

### 8.1 调整信号来源

配方调整信号来自：

```text
训练 loss
validation loss
benchmark
bad case
数据质量监控
训练团队反馈
```

### 8.2 调整节奏

建议：

| 阶段 | 调整频率 |
|---|---|
| 小规模试训 | 每次试训后调整 |
| 大规模 pretrain | 阶段性 checkpoint 后评估，不频繁中途调整 |
| continue training | 可更频繁调整 |
| SFT/RLHF | 按任务效果和 bad case 调整 |

### 8.3 调整动作

常见动作：

```text
提高某 content_type 比例
降低某 source 比例
提高 premium_pool 采样权重
降低重复风险较高数据
增加某 language/domain
移除导致异常的 batch/source
```

### 8.4 配方调整示例

| 训练现象 | 可能原因 | 调整方向 |
|---|---|---|
| 代码 benchmark 低 | 代码数据不足或低质 | 增加 premium code / bug-fix / API usage |
| 中文 benchmark 低 | 中文高质量数据不足 | 增加中文技术文档、教材、权威网页 |
| loss 下降快但 benchmark 不涨 | 重复或低多样性 | 降低重复来源，提高多样性 |
| validation loss 上升 | 过拟合 | 降低 premium 重采样，扩大候选池 |
| 幻觉严重 | 来源可靠性不足 | 增加权威文档，降低低可靠网页 |
| 安全 bad case 多 | 风险过滤不足 | 提高 safety 阈值，调整 risk_score 权重 |

---

## 9. 训练反馈如何反向调整配方

### 9.1 训练反馈数据

需要从训练团队拿到：

```text
checkpoint_id
dataset_version
mixture_recipe_version
loss curve
validation loss
benchmark result
bad case tags
训练异常记录
```

### 9.2 反馈到数据维度

将训练问题映射回配方维度：

```text
source
content_type
language
quality_pool
domain
repo/site tier
length_bucket
```

### 9.3 bad case 到配方调整

示例：

```text
bad case: 模型不会使用某云厂商 SDK
  → 检查 api_doc / code_usage 对齐数据是否不足
  → 增加相关 API 文档 + 真实代码使用样本
  → 下轮配方提高该 domain/source 权重
```

### 9.4 训练反馈不直接替代质量分

训练反馈是配方优化信号，不应直接覆盖数据质量评价。

关系是：

```text
质量评价：判断数据是否可用、是否高质量
训练反馈：判断某类数据对当前模型目标是否有效
```

---

## 10. 过拟合与高质量数据不足风险控制

### 10.1 premium_pool 过采样风险

`premium_pool` 控制在 5%-10%，如果训练中高频重复使用，会带来：

```text
过拟合
能力偏科
benchmark 虚高
多样性下降
```

控制策略：

```text
premium 最大采样权重
单条样本最大重复次数
单 source 最大占比
单 repo/site 最大占比
```

### 10.2 高质量数据不足

如果 candidate/premium 数据不足：

```text
先扩大候选数据源
再优化过滤阈值
再考虑合成数据补充
最后考虑低质量池复核回收
```

不建议直接降低所有质量门槛。

### 10.3 重复数据风险

配方层需要额外控制：

```text
near_duplicate_cluster 最大采样数
同源镜像数据去重
相同教程/文档转载去重
代码 fork/repo mirror 去重
```

### 10.4 长尾数据保护

长尾语言和领域不能完全被质量分挤掉。

可设置：

```text
minimum_quota_by_language
minimum_quota_by_domain
long_tail_sampling_weight
```

但前提是满足基础可用性。

---

## 11. 配方版本管理

每次训练必须保存配方版本。

### 11.1 mixture_recipe_version

建议格式：

```text
mixture_recipe_version = recipe_name + version + timestamp
```

示例：

```text
base_pretrain_v1.0_20260616
code_heavy_v1.2_20260701
zh_enhanced_v0.3_20260715
```

### 11.2 配方内容

配方文件至少包含：

```yaml
recipe_name: base_pretrain_v1
recipe_version: 1.0
unit: estimated_tokens
total_target_tokens: 1.2T
source_snapshot_id: ...
quality_pipeline_version: ...

mixture:
  - selector:
      source: github
      content_type: code
      language: python
      quality_pool: candidate
    target_tokens: 120B
    sample_weight: 1.0

  - selector:
      source: github
      content_type: code
      language: python
      quality_pool: premium
    target_tokens: 30B
    sample_weight: 1.5
```

### 11.3 manifest 输出

每次配方执行后输出 dataset manifest：

```text
dataset_version
mixture_recipe_version
selected_item_count
selected_token_count
source distribution
content_type distribution
language distribution
quality_pool distribution
file list / shard list
Iceberg snapshot IDs
```

### 11.4 可复现要求

半年后必须能回答：

```text
某次训练用了哪份配方？
每类数据用了多少 token？
每个 shard 来自哪个 snapshot？
质量评估版本是什么？
训练 checkpoint 对应哪个 dataset_version？
```

---

## 12. 与训练团队的接口

### 12.1 数据团队交付

数据团队交付：

```text
数据集 manifest
mixture_recipe_version
shard 列表
质量池分布
source/content_type/language/token 分布
已知风险说明
```

### 12.2 训练团队输入

训练团队需要提供：

```text
目标 total_tokens
目标能力倾向
最大训练时长
tokenizer 版本
max_seq_len
global_batch_size
评估 benchmark
可接受的数据风险边界
```

### 12.3 双方协作边界

| 事项 | 数据团队 | 训练团队 |
|---|---|---|
| 数据清洗与质量分 | 主责 | 反馈效果 |
| 初始配方建议 | 主责 | 确认训练目标 |
| 训练执行 | 支持 | 主责 |
| 训练反馈分析 | 共同 | 主责提供指标 |
| 配方调整 | 共同 | 共同 |
| 数据版本复现 | 主责 | 使用 checkpoint 关联 |

---

## 13. Phase 2/3 演进方向

### 13.1 Phase 2：effective token 与类型专属配方

Phase 2 引入：

```text
effective_token_weight
source/content_type 专属配方模板
代码语言专属配方
中文增强专属配方
API 文档-代码对齐数据配方
```

### 13.2 Phase 2：配方实验平台

建设：

```text
recipe registry
recipe diff
recipe simulation
quality distribution preview
sampling dry-run
```

### 13.3 Phase 3：训练反馈驱动自动调配方

形成：

```text
benchmark → data dimension attribution → recipe suggestion
bad case → data gap detection → sampling weight adjustment
loss drift → source/content_type 回溯 → 配方修正
```

长期目标：

```text
数据配方从人工经验驱动
演进为质量分 + 训练反馈 + 约束优化共同驱动
```

---

## 14. 关键结论

1. 数据配方是数据资产到模型训练之间的控制面。
2. Phase 1 配比单位以 token 为主，GB/document/sample 只作辅助。
3. Phase 2/3 可引入 effective token，但 Phase 1 只预留字段。
4. 初始配方应以通用基座为主，同时对代码和中文有明确增强权重。
5. `candidate_pool` 是训练主体，`premium_pool` 是高质量增强池，不建议只用 premium 训练。
6. premium 采样必须限制单 source、单 repo/site、单 domain 的集中度，避免过拟合。
7. 配方调整应基于 loss、benchmark、bad case 和数据质量监控共同判断。
8. 每次训练必须保存 `mixture_recipe_version` 和 dataset manifest。
9. 配方版本必须能关联到 Iceberg snapshot、质量评估版本和训练 checkpoint。
10. 长期方向是通过训练反馈反向优化数据配方，而不是只靠人工经验调比例。
