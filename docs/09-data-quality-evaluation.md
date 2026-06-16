# 数据质量评价体系设计

> 面向 70B 基座模型训练数据链路的数据准入、质量评分与训练反馈闭环设计

---

## 目录

- [1. 目标与定位](#1-目标与定位)
- [2. 总体设计原则](#2-总体设计原则)
- [3. 分阶段演进路线](#3-分阶段演进路线)
- [4. 质量评价对象粒度](#4-质量评价对象粒度)
- [5. Phase 1：数据可用性准入体系](#5-phase-1数据可用性准入体系)
- [6. Phase 1：三池机制](#6-phase-1三池机制)
- [7. Phase 1：级联短路评测链路](#7-phase-1级联短路评测链路)
- [8. Phase 1：评分字段与计算逻辑](#8-phase-1评分字段与计算逻辑)
- [9. Phase 1：模型与评测信号选型](#9-phase-1模型与评测信号选型)
- [10. LLM Judge 抽样校准](#10-llm-judge-抽样校准)
- [11. 阈值与分池策略](#11-阈值与分池策略)
- [12. 元数据与表设计](#12-元数据与表设计)
- [13. 与训练团队的接口](#13-与训练团队的接口)
- [14. 质量监控体系](#14-质量监控体系)
- [15. Phase 2/3 扩展设计](#15-phase-23-扩展设计)
- [16. 关键结论](#16-关键结论)

---

## 1. 目标与定位

本文档设计大模型训练数据链路中的**数据质量评价体系**，目标不是单纯回答“哪些数据要过滤”，而是建立一个可持续演进的质量系统：

```text
Phase 1：数据准入标准
Phase 2：质量评分体系
Phase 3：训练效果闭环
```

核心问题是：

> 数据团队如何判断、证明、监控一批训练数据是“高质量”的，并让这个质量结论能被训练团队用于数据配比和训练决策。

### 1.1 本文档解决什么

| 问题 | 本文回答 |
|---|---|
| 什么数据可以进入训练候选集 | Phase 1 三池准入机制 |
| 如何高性能评估 PB 级数据 | 级联短路评测链路 |
| 如何计算质量分 | 硬门槛 + 可用性/质量/风险分项 |
| 如何避免质量分不可解释 | 独立质量结果表 + reason codes |
| 如何和训练团队协作 | 只暴露摘要字段，详细字段由数据团队管理 |
| 如何持续改进 | LLM Judge 抽样、监控、Phase 2/3 训练反馈闭环 |

### 1.2 Phase 1 的核心定位

Phase 1 不追求精准排序所有数据，也不追求建立最终质量模型。

Phase 1 的核心目标是：

```text
确保数据可用
```

具体含义：

```text
可读
可解析
非明显垃圾
非严重重复
非高风险
来源可追溯
不确定数据不交付训练
```

Phase 1 的质量分是**可用性质量分**，不是最终训练效果质量分。

---

## 2. 总体设计原则

### 2.1 分阶段、保持可扩展

质量体系按三阶段演进：

| 阶段 | 目标 | 特点 |
|---|---|---|
| Phase 1 | 保证数据可用 | 规则 + 轻量模型，高性能、低成本 |
| Phase 2 | 提升质量识别准确性 | 类型专属模型、LLM Judge 校准、少量人工 gold set |
| Phase 3 | 与训练效果闭环 | 接入 loss、benchmark、bad case、线上反馈 |

### 2.2 先准入，再排序

Phase 1 不能把 `overall_quality_score` 当成唯一准入依据。

必须先经过硬规则和基础可用性检查：

```text
硬风险数据不能因为其它维度高分而进入训练集
```

例如：

```text
一篇文档信息密度很高，但包含大量 API key → blocked_pool
一个 repo 星标很高，但 license 明确不可用 → blocked_pool
一段代码结构完整，但疑似 benchmark 泄漏 → 不交付训练，等待复核
```

### 2.3 风险不能被高质量分抵消

风险类字段必须独立处理：

```text
PII / secret / license / toxicity / benchmark contamination
```

这些字段不应简单参与加权求和后被其它高分“抵消”。

### 2.4 高性能优先

Phase 1 面向 PB 级数据处理，不允许所有数据都跑重模型。

主链路采用：

```text
L0：硬规则过滤
L1：统计特征 + 轻规则
L2：轻量模型
L3：LLM Judge 抽样
```

其中 L3 不进入全量主链路。

### 2.5 主表轻量，详情表可追溯

主数据表只保留训练团队最需要的摘要字段：

```text
quality_pool
overall_quality_score
```

完整评分过程、分项分数、模型版本、reason codes 写入独立质量结果表。

---

## 3. 分阶段演进路线

### 3.1 Phase 1：数据准入与可用性质量分

目标：保证交付训练的数据可用、风险可控。

| 模块 | Phase 1 设计 |
|---|---|
| 评价粒度 | 文档级为主，超长文档/API 文档可切 chunk |
| 质量维度 | 统一核心维度 |
| 准入机制 | blocked / candidate / premium 三池 |
| 评分来源 | 规则 + 统计特征 + KenLM + DistilBERT/轻量分类器 |
| LLM Judge | 每天 1-5 万条抽样，不进入全量主链路 |
| 人工评估 | 暂不做 |
| 训练接口 | candidate/premium + 摘要字段 |

### 3.2 Phase 2：类型专属质量模型

目标：从“数据可用”升级到“数据质量更准”。

| 数据类型 | Phase 2 质量方向 |
|---|---|
| 代码 | function/class 级质量分、可编译性、依赖健康、测试信号 |
| 网页文档 | boilerplate 更精细识别、事实性、原创性、结构完整性 |
| API 文档 | 参数完整性、示例质量、代码使用对齐 |
| Issue/PR | problem clarity、resolution status、solution traceability |
| 合成数据 | 多样性、真实性、一致性、模板化风险 |

### 3.3 Phase 3：训练效果闭环

目标：把数据质量与模型效果打通。

新增信号：

```text
training_feedback_score
benchmark_contribution_score
bad_case_feedback_score
downstream_impact_score
```

形成闭环：

```text
数据质量 → 训练 loss / benchmark → bad case → 数据重构 → 新一轮训练
```

---

## 4. 质量评价对象粒度

采用分阶段混合粒度。

### 4.1 Phase 1 粒度

```text
文档级为主
片段级只用于超长文档 / 网页 / API 文档
```

典型质量单元：

| 数据源 | Phase 1 质量单元 |
|---|---|
| GitHub 代码 | file / document 级 |
| README / 文档页 | page / document 级 |
| API 文档 | page 级，超长页面可 section/chunk |
| Issue / PR | thread 级 |
| 网页正文 | page 级，超长正文可 chunk |

### 4.2 Phase 2 粒度

```text
引入 chunk 级质量分
代码数据开始做 function/class 级评分
```

### 4.3 Phase 3 粒度

形成多粒度质量体系：

```text
repo / site / page / chunk / file / function / commit / issue
```

多层质量分联动示例：

```text
repo_quality_score
  └── file_quality_score
        └── function_quality_score
```

---

## 5. Phase 1：数据可用性准入体系

Phase 1 的准入不是简单“保留/删除”，而是先保证可用性。

### 5.1 硬性拦截

触发以下规则的数据进入 `blocked_pool`，禁止进入训练交付。

| 类别 | 硬性拦截条件 |
|---|---|
| 内容可读性 | 内容为空、无法读取、明显乱码 |
| 格式解析 | 解析失败且无法提取有效正文 |
| 重复 | SHA256 完全重复 |
| 安全 | PII / secret / token 高风险 |
| 垃圾内容 | 明显广告、模板页、导航页、灌水页 |
| 合规 | License 明确不可用 |
| 内容安全 | 有毒、成人、暴恐、攻击等高风险内容 |
| 长度 | token 数低于最低阈值 |

### 5.2 基础可用性检查

不触发 blocked 规则后，还需要满足基础可用标准，才可进入训练候选。

```text
可解析
非完全重复
语言识别置信度达标
安全风险低
来源可追溯
文本长度达标
基础结构合理
```

不满足基础可用标准的数据：

```text
不进入 blocked_pool
也不交付训练
保留在原始层 / 中间层，等待 Phase 2 复核机制
```

### 5.3 不确定数据处理原则

Phase 1 明确采用保守策略：

```text
不确定的数据先不交付训练
但不物理删除
```

这样既能保护训练集，也保留后续重评估空间。

---

## 6. Phase 1：三池机制

Phase 1 只实现三个核心池。

```text
blocked_pool    # 明确不可用，禁止进入训练
candidate_pool  # 默认可用，可进入训练候选
premium_pool    # 高质量优先，可优先进入训练配比
```

### 6.1 blocked_pool

进入条件：

```text
触发 hard_block_rule
```

用途：

```text
禁止交付训练
保留血缘与审计记录
可用于 blocked reason 统计和规则优化
```

### 6.2 candidate_pool

进入条件：

```text
通过硬规则
通过基础可用性检查
overall_quality_score >= candidate_min_threshold
```

用途：

```text
默认可交付训练团队
作为训练配比的主要候选池
```

### 6.3 premium_pool

进入条件：

```text
属于 candidate_pool
overall_quality_score >= premium_min_threshold
位于 source/content_type 内 top 5%-10%
```

用途：

```text
高质量优先池
用于高质量配比、少量高权重采样、质量样本分析
```

### 6.4 Phase 2 扩展池

Phase 2 增加：

```text
quarantine_pool    # 隔离池，需要 LLM/人工复核
low_quality_pool   # 可保留，但默认不训练
```

---

## 7. Phase 1：级联短路评测链路

为了支撑 PB 级数据处理，Phase 1 使用级联短路评测。

```text
L0：硬规则过滤（CPU/Spark）
  ↓
L1：统计特征 + 轻规则（CPU/Spark）
  ↓
L2：轻量模型（只对通过 L0/L1 的候选跑）
  ↓
L3：LLM Judge 抽样校准（不进主链路）
```

### 7.1 L0：硬规则过滤

目标：用最低成本过滤明确不可用数据。

典型规则：

```text
empty_content
invalid_encoding
exact_duplicate
high_pii_risk
high_secret_risk
license_blocked
toxicity_high_risk
too_short
```

执行框架：

```text
Spark CPU 批处理
正则 / 字典 / 哈希 / 解析器
```

### 7.2 L1：统计特征 + 轻规则

目标：产生基础质量信号。

特征包括：

```text
char_length
token_count_estimate
line_count
unique_token_ratio
compression_ratio
html_text_ratio
code_comment_ratio
language_confidence
source_reliability_score
structure_score
```

### 7.3 L2：轻量模型

目标：对候选数据做轻量语义质量判断。

Phase 1 主链路模型：

```text
KenLM / n-gram perplexity
DistilBERT / 轻量质量分类器
```

ModernBERT-base 不默认跑全量，只用于：

```text
premium 候选
高价值子集
抽样校准
质量模型训练样本
```

### 7.4 L3：LLM Judge 抽样校准

LLM Judge 只用于抽样校准，不参与全量过滤。

用途：

```text
监控质量趋势
校准 candidate/premium 阈值
发现规则/模型误判类型
积累 Phase 2 训练样本
```

---

## 8. Phase 1：评分字段与计算逻辑

Phase 1 使用：

```text
硬门槛 + 分项分数 + 总分合成
```

权重不写死，只给默认建议，后续通过 LLM Judge 抽样和训练反馈校准。

### 8.1 核心分项

| 字段 | 含义 | Phase 1 用途 |
|---|---|---|
| `usability_score` | 可用性分 | 判断是否满足训练候选基础条件 |
| `quality_score` | 内容质量分 | 衡量信息密度、可读性、模型质量分 |
| `risk_score` | 风险分 | 表示 PII、重复、毒性、License 等风险 |
| `source_reliability_score` | 来源可靠性分 | 衡量来源权威性和可追溯性 |
| `overall_quality_score` | 总质量分 | 用于 candidate/premium 分层 |

### 8.2 硬门槛示例

以下阈值仅为默认建议，不写死：

```text
safety_score < 0.8              → blocked_pool
language_confidence < 0.7       → not_delivered
source_traceability = false     → not_delivered
content_length < min_threshold  → not_delivered
dedup_confidence < 0.6          → not_delivered / blocked by exact duplicate
```

### 8.3 分项分数默认建议

#### usability_score

```text
parse_success
language_confidence
structure_score
length_score
source_traceability
```

#### quality_score

```text
information_density_score
readability_score
text_quality_model_score
perplexity_score
```

#### risk_score

```text
pii_risk
dedup_risk
toxicity_risk
license_risk
secret_risk
```

### 8.4 总分合成原则

默认建议：

```text
overall_quality_score =
  usability_score 与 quality_score 为主体
  source_reliability_score 作为加分项
  risk_score 作为独立惩罚项
```

关键原则：

```text
风险不能被高质量分抵消
可用性必须先过硬门槛
overall_quality_score 只是分层依据，不是唯一准入依据
```

---

## 9. Phase 1：模型与评测信号选型

### 9.1 模型分层方案

| 层级 | 评测信号 | 是否全量 | 说明 |
|---|---|---|---|
| L0/L1 | 规则 + 统计特征 | 是 | Spark CPU 批处理 |
| L2 | KenLM perplexity | 候选大规模 | 低成本文本流畅性判断 |
| L2 | DistilBERT / 轻量质量分类器 | 候选大规模 | 轻量语义质量判断 |
| L2 | ModernBERT-base | 小规模高价值候选 | 不默认全量跑 |
| L3 | LLM Judge | 抽样 | 仅用于校准和监控 |

### 9.2 KenLM / Perplexity

用途：

```text
识别乱码、低流畅度文本、异常语言分布
```

限制：

```text
不能判断事实性
不能判断代码是否好
对多语言和代码需要分语言/类型建模或谨慎解释
```

### 9.3 DistilBERT / 轻量分类器

用途：

```text
基础质量分类
垃圾/模板/低价值文本识别
候选池质量排序
```

特点：

```text
吞吐高
成本低
适合 Phase 1 大规模候选数据
```

### 9.4 ModernBERT-base

Phase 1 不进入默认全量链路。

适用场景：

```text
premium 候选复核
高价值 source 抽样
质量模型训练样本生成
Phase 2 类型专属模型基座
```

### 9.5 LLM Judge

Phase 1 只抽样，不全量。

原因：

```text
成本高
吞吐低
延迟大
不适合 PB 级主链路
```

---

## 10. LLM Judge 抽样校准

Phase 1 暂不做人审，只做 LLM Judge 抽样或最简单评测。

### 10.1 抽样规模

```text
每天 1-5 万条以内
```

具体数量根据当天数据量、LLM 推理成本和评测资源动态调整。

### 10.2 抽样结构

采用分层 + 边界 + 随机混合抽样。

```text
70% 分层抽样
20% 边界抽样
10% 随机抽样
```

### 10.3 分层抽样

按以下维度分桶：

```text
source
content_type
language
quality_bucket
length_bucket
dedup_risk_bucket
```

用途：

```text
监控不同来源/类型/语言的数据质量
发现某类数据的系统性问题
```

### 10.4 边界抽样

重点抽：

```text
candidate_threshold 附近
premium_threshold 附近
```

用途：

```text
校准阈值
发现误杀和漏放
```

### 10.5 随机抽样

用途：

```text
估计整体质量水平
避免只关注特定桶导致视野偏差
```

### 10.6 LLM Judge 输出

建议输出字段：

```text
llm_judge_score
llm_judge_confidence
reject_reason
issue_tags
suggested_pool
judge_model_version
judge_prompt_version
```

Phase 1 的 LLM Judge 结果只用于：

```text
监控
阈值校准
误判分析
Phase 2 质量模型样本积累
```

不直接作为强过滤依据。

---

## 11. 阈值与分池策略

采用：

```text
绝对阈值 + 分位数约束
```

### 11.1 blocked_pool

由硬规则决定：

```text
hard_block_rule = true → blocked_pool
```

### 11.2 candidate_pool

使用最低绝对阈值，不强制设比例上限。

默认建议：

```text
overall_quality_score >= candidate_min_threshold
```

候选比例过低时，需要排查：

```text
数据源质量是否变差
规则是否过严
模型分是否漂移
语言/类型是否被误判
```

### 11.3 premium_pool

使用最低绝对阈值 + 分位数约束。

默认建议：

```text
overall_quality_score >= premium_min_threshold
并且位于各 source/content_type 的 top 5%-10%
```

premium_pool 控制在 5%-10% 的原因：

```text
避免 premium 概念膨胀
保证高质量优先池有明确含义
便于训练团队做高质量数据配比
```

### 11.4 阈值校准

Phase 1：

```text
阈值为默认建议，不写死
通过 LLM Judge 抽样观察误判
按周或按批次调整
```

Phase 2：

```text
引入 gold set / 少量人工复核
按 source/content_type/language 校准阈值
```

Phase 3：

```text
根据训练反馈动态调整阈值
```

---

## 12. 元数据与表设计

采用分阶段存储设计。

### 12.1 Phase 1 主表摘要字段

主数据表只保留：

```text
quality_pool
overall_quality_score
```

原因：

```text
训练团队读取简单
主表不膨胀
避免误用内部质量细节
```

### 12.2 独立质量结果表

新建 Iceberg 表：

```text
data_quality_scores
```

建议字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `item_id` | string | 数据项 ID |
| `source_snapshot_id` | string | 来源 snapshot |
| `dataset_snapshot_id` | string | 当前数据集 snapshot |
| `quality_pipeline_version` | string | 质量管线版本 |
| `quality_model_version` | string | 质量模型版本，可空 |
| `quality_pool` | enum | blocked / candidate / premium / not_delivered |
| `overall_quality_score` | float | 总质量分 |
| `usability_score` | float | 可用性分 |
| `quality_score` | float | 内容质量分 |
| `risk_score` | float | 风险分 |
| `source_reliability_score` | float | 来源可靠性分 |
| `language_confidence` | float | 语言识别置信度 |
| `dedup_confidence` | float | 去重置信度 |
| `reason_codes` | array<string> | 规则/模型原因码 |
| `blocked_reason` | string | blocked 原因，可空 |
| `llm_judge_score` | float | 抽样 LLM Judge 分，可空 |
| `llm_judge_confidence` | float | LLM Judge 置信度，可空 |
| `eval_timestamp` | timestamp | 评估时间 |

### 12.3 Phase 2 类型专属质量表

Phase 2 增加：

```text
code_quality_scores
doc_quality_scores
api_quality_scores
synthetic_quality_scores
```

### 12.4 Phase 3 训练反馈表

Phase 3 增加：

```text
training_feedback_quality_scores
```

字段示例：

```text
item_id
dataset_version
checkpoint_id
training_feedback_score
benchmark_contribution_score
bad_case_feedback_score
downstream_impact_score
```

---

## 13. 与训练团队的接口

### 13.1 Phase 1 默认交付字段

训练团队默认看到：

```text
quality_pool
overall_quality_score
source
content_type
language
token_count
```

默认交付：

```text
candidate_pool
premium_pool
```

不交付：

```text
blocked_pool
not_delivered 数据
```

### 13.2 使用方式

训练团队可以：

```text
只采样 candidate/premium
控制 premium_pool 在训练配方中的比例
在 candidate_pool 内按 overall_quality_score 做排序或采样权重
按 source/content_type/language/token_count 与 quality_pool 组合配比
```

### 13.3 不建议的使用方式

不建议：

```text
直接把 overall_quality_score 当作绝对真理
只用 premium_pool 训练
完全忽略 source/content_type/language 分布
跨数据类型直接比较质量分而不看分布
```

### 13.4 Phase 2/3 接口

Phase 2：

```text
训练团队可通过 dataset registry / metadata API 查询详细质量信息
但默认训练交付仍以摘要字段为主
```

Phase 3：

```text
训练反馈闭环打通后
训练团队可以按更多 quality features 做实验
```

---

## 14. 质量监控体系

质量评价必须持续监控，而不是一次性打分。

### 14.1 Phase 1 监控指标

Phase 1 监控：

```text
1. 数据量
2. 三池比例
3. overall_quality_score 分布
4. usability_score / quality_score / risk_score 分布
5. blocked reason Top N
6. source / content_type / language 基础切片
7. LLM Judge 抽样均分和 reject reason
```

### 14.2 数据量与三池比例

每日监控：

```text
raw_count
blocked_count
candidate_count
premium_count
not_delivered_count
blocked_ratio
candidate_ratio
premium_ratio
not_delivered_ratio
```

### 14.3 分数分布

监控：

```text
overall_quality_score distribution
usability_score distribution
quality_score distribution
risk_score distribution
source_reliability_score distribution
language_confidence distribution
dedup_confidence distribution
```

### 14.4 blocked reason Top N

用于发现规则异常或数据源劣化。

示例：

```text
high_secret_risk ↑
invalid_encoding ↑
too_short ↑
license_blocked ↑
html_boilerplate ↑
```

### 14.5 基础切片

按以下维度切片：

```text
source
content_type
language
quality_pool
length_bucket
```

示例监控问题：

```text
GitHub Java 代码 candidate_ratio 是否突然下降？
Web 文档 blocked_ratio 是否突然上升？
中文网页 risk_score 是否异常升高？
premium_pool 是否集中在少数 source？
```

### 14.6 LLM Judge 抽样趋势

监控：

```text
llm_judge_score 平均值
reject_reason 分布
suggested_pool 与 pipeline_pool 一致率
边界样本误判率
```

### 14.7 Phase 2/3 监控扩展

Phase 2：

```text
reason_codes drift
pool ratio drift
LLM Judge 抽样质量趋势
source/content_type/domain 细切片
```

Phase 3：

```text
数据质量变化与 training loss 的关联
数据质量变化与 benchmark 的关联
数据质量变化与 bad case 的关联
```

---

## 15. Phase 2/3 扩展设计

### 15.1 Phase 2：类型专属质量模型

#### 代码数据

新增维度：

```text
syntax_validity
function_complexity
maintainability_score
test_signal
dependency_health
comment_quality
benchmark_leakage_risk
```

#### 网页/自然语言文档

新增维度：

```text
factuality_score
originality_score
coherence_score
boilerplate_score
domain_authority_score
```

#### API 文档

新增维度：

```text
completeness_score
parameter_detail_score
example_quality_score
code_alignment_score
version_freshness_score
```

#### Issue / PR / Discussion

新增维度：

```text
problem_clarity_score
solution_traceability_score
discussion_signal_score
noise_ratio
resolution_status
```

### 15.2 Phase 2：人工 Gold Set

Phase 1 暂不做人审，Phase 2 再引入小规模人工 gold set。

用途：

```text
校准 LLM Judge
训练专属质量模型
评估误杀/漏放
形成长期质量标尺
```

### 15.3 Phase 3：训练反馈质量分

Phase 3 引入训练侧反馈：

```text
某类数据加入后 loss 是否更稳
某类数据是否提升 benchmark
某类数据是否导致过拟合
某类数据是否减少线上 bad case
```

形成数据闭环：

```text
质量评价 → 数据配比 → 模型训练 → Benchmark/Bad Case → 数据质量规则调整
```

---

## 16. 关键结论

1. `09-data-quality-evaluation.md` 的定位是分阶段质量系统，不是单纯过滤规则。
2. Phase 1 的核心目标是**确保数据可用**，不追求最终训练效果最优排序。
3. Phase 1 采用三池机制：`blocked_pool`、`candidate_pool`、`premium_pool`。
4. 不确定数据 Phase 1 不交付训练，但不物理删除，保留在原始层/中间层。
5. Phase 1 使用规则硬门槛 + 分数分层，质量分是“可用性质量分”。
6. 主链路采用级联短路评测，优先高效率、高性能。
7. Phase 1 全量用规则 + 轻量模型，LLM Judge 只做每天 1-5 万条抽样，不做人审。
8. `overall_quality_score` 权重不写死，只给默认建议，后续通过 LLM Judge 和训练反馈校准。
9. `premium_pool` 控制在各 source/content_type 的 top 5%-10%。
10. 主数据表只保留 `quality_pool` 和 `overall_quality_score`，完整结果写入独立质量表。
11. Phase 1 训练团队默认只消费 candidate/premium 和摘要字段，blocked 不交付。
12. Phase 2 再引入类型专属质量模型和人工 gold set；Phase 3 接入训练反馈闭环。
