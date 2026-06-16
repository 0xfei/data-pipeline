# DeepSeek 工作拆解：数据、架构、RL、评测与推理系统

> 基于 DeepSeek-V3、DeepSeekMath/GRPO 与 verl 公开材料的系统拆解

---

## 目录

- [1. 总览：DeepSeek 的端到端系统工程](#1-总览deepseek-的端到端系统工程)
- [2. 公开事实、外部参考与工程推断边界](#2-公开事实外部参考与工程推断边界)
- [3. 数据工程：从 Common Crawl 到高质量推理数据](#3-数据工程从-common-crawl-到高质量推理数据)
- [4. 模型架构：MLA、DeepSeekMoE、MTP 与 FP8](#4-模型架构mladeepseekmoemtp-与-fp8)
- [5. 训练系统：H800 集群、DualPipe 与 MoE 通信](#5-训练系统h800-集群dualpipe-与-moe-通信)
- [6. RL 算法：GRPO 如何替代 PPO 中的 critic](#6-rl-算法grpo-如何替代-ppo-中的-critic)
- [7. verl 框架视角：如何工程化 DeepSeek 类 RL 流程](#7-verl-框架视角如何工程化-deepseek-类-rl-流程)
- [8. Muon 优化器：应如何放在 DeepSeek 语境里理解](#8-muon-优化器应如何放在-deepseek-语境里理解)
- [9. 评测体系：数学、代码、知识、中文与开放式评测](#9-评测体系数学代码知识中文与开放式评测)
- [10. 推理系统：prefill/decode 分离、KV Cache 与 expert 负载均衡](#10-推理系统prefilldecode-分离kv-cache-与-expert-负载均衡)
- [11. 对当前 70B 数据链路方案的启示](#11-对当前-70b-数据链路方案的启示)
- [12. 可复用路线图](#12-可复用路线图)
- [13. 风险与未公开部分](#13-风险与未公开部分)
- [参考资料](#参考资料)

---

## 1. 总览：DeepSeek 的端到端系统工程

DeepSeek 的公开工作不是单点算法突破，而是一个端到端 co-design：

```text
数据工程
  → 模型架构
  → 训练框架
  → 低精度训练
  → SFT / RL 后训练
  → 评测
  → 推理部署
```

更完整地看，DeepSeek 的公开路线可以抽象成下面的闭环：

```text
Common Crawl / GitHub / Math / Code / 多语言文本
        ↓
去重、去污染、质量过滤、领域数据发现、数据配方
        ↓
DeepSeek-V3 Base Pretraining
        ↓
SFT：指令、对话、代码、数学、工具使用
        ↓
RL / GRPO：可验证任务、奖励函数、group rollouts
        ↓
Benchmark：数学、代码、知识、中文、安全、开放式评测
        ↓
Inference：MLA KV Cache、MoE Expert Parallel、Prefill/Decode 分离
        ↓
Bad Case / Eval Feedback / Online Feedback
        ↓
下一轮数据发现、配方调整、RL prompt 构造
```

这个闭环说明：DeepSeek 的竞争力不只来自模型参数，也来自“数据—训练—评测—推理”之间的连续反馈。对于数据链路团队，最重要的是把数据资产做成能被训练和评测反复消费的工程系统，而不是一次性清洗任务。

其中 DeepSeek-V3 Technical Report 公开了几个关键事实：

- 模型是 MoE 架构，总参数约 `671B`，每 token 激活约 `37B` 参数。
- 架构采用 `MLA`（Multi-head Latent Attention）降低 KV cache，采用 `DeepSeekMoE` 做经济训练与推理。
- 训练数据为 `14.8T` 高质量、多样化 tokens。
- 训练使用 `FP8` mixed precision、`DualPipe`、跨节点 all-to-all 通信优化等工程手段。
- 完整训练成本公布为 `2.788M H800 GPU hours`，按 `$2/GPU-hour` 估算约 `$5.576M`。
- 后训练包含 `SFT` 和 `RL`，并从 DeepSeek-R1 系列蒸馏推理能力。

DeepSeekMath Technical Report 则公开了更早期、但对理解 `GRPO` 和数据工程非常关键的内容：

- 从 Common Crawl 中构建 `120B` 数学 tokens。
- 使用 fastText 分类器迭代召回数学网页。
- 对 benchmark contamination 做 n-gram 级过滤。
- 提出 `GRPO`，通过 group rewards 替代 PPO 中的 value/critic 模型。

verl 是 ByteDance Seed 发起的开源 RL post-training 框架，支持 PPO/GRPO、FSDP/Megatron、vLLM/SGLang 等，可以作为理解和工程化 DeepSeek 类 RL 训练的参考框架，但不是 DeepSeek 官方公布的训练框架。

---

## 2. 公开事实、外部参考与工程推断边界

为了避免把不同来源混在一起，本文使用三类标记：

| 类型 | 含义 | 示例 |
|---|---|---|
| **公开事实** | DeepSeek 论文/报告明确写出的内容 | DeepSeek-V3 的 `MLA`、`DeepSeekMoE`、`FP8`、`DualPipe`、`14.8T tokens` |
| **外部参考** | 不是 DeepSeek 官方组件，但可用于复现或工程化类似流程 | `verl` 支持 `GRPO/PPO`、`vLLM/SGLang`、`FSDP/Megatron` |
| **工程推断** | 基于公开事实对系统设计的合理拆解 | 当前数据链路如何借鉴 DeepSeek 的数据质量与配方闭环 |

特别说明：

```text
Muon 优化器目前不能直接写成 DeepSeek-V3/R1 技术报告中的官方公开组件。
```

如果后续要分析 Muon，应作为“业界优化器趋势 / 后续可调研方向”，而不是作为 DeepSeek 已公开训练栈的一部分。

---

## 3. 数据工程：从 Common Crawl 到高质量推理数据

### 3.1 DeepSeekMath 的数学数据挖掘 pipeline

DeepSeekMath 展示了一个非常适合数据团队借鉴的领域数据挖掘范式：

```text
seed corpus
  → 训练 fastText 分类器
  → 从 Common Crawl 召回候选网页
  → 按分类分数排序取 top tokens
  → 发现高命中 domain
  → 人工标注 URL path
  → 扩充正样本
  → 重新训练分类器
  → 多轮迭代直到召回趋于饱和
```

公开细节包括：

- 初始 seed 使用 `OpenWebMath`。
- 从 seed 中抽取 `500K` 正样本，从 Common Crawl 中抽取 `500K` 负样本训练 fastText。
- 对 Common Crawl 做 URL 级去重和近重，得到约 `40B` HTML pages。
- 通过多轮迭代最终得到 `35.5M` 数学网页，总计约 `120B` tokens。
- 第四轮时发现第三轮已经覆盖约 `98%` 数据，因此停止继续迭代。

如果把这个流程拆成可执行的数据平台任务，可以落成下面的 pipeline：

| 步骤 | 任务 | 产物 | 当前方案可复用点 |
|---|---|---|---|
| 1 | 准备 seed corpus | `seed_documents` | 可从高质量站点、人工白名单、已有 premium_pool 抽样 |
| 2 | 构造正负样本 | `classifier_train_set` | 正样本来自 seed，负样本来自随机 Common Crawl/网页池 |
| 3 | 训练轻量分类器 | `domain_classifier_version` | fastText/DistilBERT/ModernBERT 均可作为阶段选择 |
| 4 | URL 与页面去重 | `deduped_web_pages` | 复用 SHA256、MinHash、URL canonicalization |
| 5 | 批量召回候选页面 | `candidate_domain_pages` | Spark 批处理即可，先追求高召回 |
| 6 | 按 score 排序取 top tokens | `top_scored_pages` | 形成 candidate_pool，而不是直接进入训练 |
| 7 | domain/path 发现 | `high_yield_domains` | 统计高命中域名、路径模式、站点结构 |
| 8 | 人工或 LLM 辅助标注 path | `path_rules` | 少量人工校准，规模化由规则执行 |
| 9 | 扩充正样本并重训 | `classifier_v2/v3/...` | 形成迭代召回闭环 |
| 10 | benchmark 去污染 | `decontaminated_pages` | 在发布 dataset_version 前强制执行 |
| 11 | 小规模训练验证 | `domain_ablation_report` | 用小模型/中间 checkpoint 验证数据收益 |

这个流程可泛化到数学之外的多个领域：

```text
API 文档发现
代码教程发现
Issue/PR 修复链发现
中文技术博客发现
系统设计文档发现
工具调用样本发现
```

### 3.2 Benchmark 去污染

DeepSeekMath 对 benchmark contamination 的处理非常明确：

```text
对 GSM8K / MATH / CMATH / AGIEval 等 benchmark：
  - 对较长 benchmark 文本使用 10-gram exact match
  - 对短于 10-gram 但至少 3-gram 的文本使用 exact matching
  - 命中的网页从训练语料中过滤
```

这对当前数据链路的启示是：

```text
benchmark contamination 不应放到训练后排查，而应进入数据准入和数据版本发布流程。
```

建议当前方案后续新增：

```text
docs/14-benchmark-contamination.md
```

### 3.3 DeepSeekMath 的配方经验

DeepSeekMath-Base 7B 初始化自 DeepSeek-Coder-Base-v1.5 7B，并继续训练 `500B` tokens。公开配比为：

| 数据类型 | 比例 |
|---|---:|
| DeepSeekMath Corpus | 56% |
| AlgebraicStack | 4% |
| arXiv | 10% |
| GitHub code | 20% |
| Common Crawl 中英自然语言 | 10% |

这个配方说明两点：

1. 数学能力不是只靠数学文本，代码数据对工具使用和推理有帮助。
2. 领域增强训练仍需保留通用自然语言和代码，避免能力退化。

### 3.4 DeepSeek-V3 的预训练数据尺度

DeepSeek-V3 公布预训练使用 `14.8T` diverse and high-quality tokens，并经过 SFT 与 RL 后训练。

对数据团队而言，关键不是单纯追求 token 数，而是：

```text
高质量数据规模
多源覆盖
去重与去污染
配方可控
训练反馈闭环
```

---

## 4. 模型架构：MLA、DeepSeekMoE、MTP 与 FP8

### 4.1 MLA：为推理成本设计的 Attention

`MLA` 的核心目标是降低生成时 KV cache。

传统 MHA 需要缓存每层每头的 K/V：

```text
KV cache ∝ layers × heads × seq_len × head_dim
```

DeepSeek 的 `MLA` 使用低秩压缩，把 key/value 压成 latent vector，只缓存压缩后的表示和 RoPE 相关 key，从而降低长上下文推理成本。

对数据团队的启示：

```text
长上下文数据是否值得投入，取决于模型架构是否能承载长上下文推理成本。
```

如果模型有 MLA/长上下文优化，那么：

```text
长文档、长代码文件、长 API 文档、复杂 issue thread
```

才更有训练价值。

### 4.2 DeepSeekMoE：用稀疏激活降低训练/推理成本

DeepSeek-V3 是 MoE：

```text
总参数：671B
每 token 激活：37B
```

MoE 的本质是：

```text
参数容量很大
但每个 token 只走部分 experts
```

这样可以在接近大模型容量的同时，把每 token 计算成本控制在较低水平。

### 4.3 Auxiliary-loss-free load balancing

传统 MoE 为了避免 expert load 不均，常引入 auxiliary loss。但 auxiliary loss 太强可能损害模型性能。

DeepSeek-V3 使用 auxiliary-loss-free load balancing：

```text
给每个 expert 增加 routing bias
训练过程中监控 expert load
过载 expert 降低 bias
欠载 expert 提高 bias
bias 只影响路由，不直接影响 expert 输出权重
```

这相当于把负载均衡从“训练目标惩罚项”变成“路由控制机制”。

### 4.4 Node-limited routing 与 no token dropping

DeepSeek-V3 还限制每个 token 最多发送到一定数量的节点，以降低跨节点通信成本。

公开报告还强调：

```text
训练和推理中不丢 token
```

这对 MoE 很重要，因为 token dropping 会影响训练稳定性和输出质量。

### 4.5 MTP：Multi-Token Prediction

DeepSeek-V3 引入 `MTP` 训练目标：

```text
不仅预测下一个 token
还预测后续多个 token
```

它的作用：

```text
增加训练信号密度
促进模型提前规划未来 token
可能提升 benchmark 表现
MTP 模块推理时可丢弃，也可用于 speculative decoding
```

对数据团队的启示：

```text
如果模型训练目标包含 MTP，那么连续、高质量、结构完整的长文本/代码样本更重要。
```

因为模型不仅学局部 next-token，还学习更长范围的 token 预测结构。

### 4.6 FP8 mixed precision training

DeepSeek-V3 公开了大规模 FP8 mixed precision 训练：

```text
大部分 GEMM 使用 FP8
关键敏感模块保留 BF16/FP32
activation/weight 使用细粒度 scaling
部分通信和缓存用低精度降低带宽与显存
```

DeepSeek 报告称 FP8 相对 BF16 的 loss error 控制在可接受范围内。

对当前方案的启示：

```text
训练成本优化不只靠数据量减少，也靠模型架构、低精度、通信优化共同实现。
```

### 4.7 架构组件与数据团队的关系

DeepSeek-V3 的架构组件虽然属于模型团队范畴，但会直接改变数据价值判断。

| 组件 | 解决的问题 | 对成本的影响 | 对数据链路的影响 |
|---|---|---|---|
| `MLA` | KV Cache 随上下文增长过快 | 降低长上下文推理显存 | 高质量长文档、长代码、长 API 文档更值得保留 |
| `DeepSeekMoE` | 参数规模与每 token 计算成本矛盾 | 总参数大但激活参数少 | 数据分布会影响 expert specialization 和路由负载 |
| Auxiliary-loss-free balance | MoE 负载均衡与模型质量冲突 | 降低 routing collapse 风险 | 长尾领域数据可能影响 expert load，需要监控分布 |
| Node-limited routing | 跨节点通信过重 | 降低 all-to-all 通信成本 | batch shape、序列长度分布会影响训练吞吐 |
| `MTP` | next-token 信号密度有限 | 提升 token efficiency，推理可辅助 speculative decoding | 连续、结构完整、低噪声样本价值更高 |
| `FP8` | BF16 训练显存和算力成本高 | 降低训练成本，提高吞吐 | 异常 token、脏数据、极端分布更可能放大数值不稳定 |
| `DualPipe` | MoE pipeline bubble 和通信等待 | 提升训练 GPU 利用率 | 训练 shard 读取不能造成 GPU 空转 |

因此，数据团队不能只按“文本质量”评估数据，还要按训练和推理系统的约束评估数据：

```text
是否能稳定组成 batch
是否会制造极端长度分布
是否能支持长上下文能力
是否能帮助 expert 分化
是否会影响低精度训练稳定性
```

---

## 5. 训练系统：H800 集群、DualPipe 与 MoE 通信

### 5.1 训练集群

DeepSeek-V3 使用 `2048 H800 GPUs` 训练。每节点 8 GPU，节点内 NVLink/NVSwitch，节点间 InfiniBand。

公开训练并行配置包括：

```text
16-way Pipeline Parallelism
64-way Expert Parallelism
ZeRO-1 Data Parallelism
```

并且报告强调：通过内存优化，训练 DeepSeek-V3 不使用 costly Tensor Parallelism。

### 5.2 DualPipe：为 MoE 通信设计的 pipeline

MoE 训练的核心瓶颈之一是：

```text
all-to-all dispatch/combine 通信很重
```

DeepSeek-V3 的 `DualPipe` 目标是：

```text
减少 pipeline bubble
重排 forward/backward chunk
把 attention、MLP、all-to-all dispatch/combine、PP communication 交错执行
尽量隐藏通信开销
```

### 5.3 Cross-node All-to-All 通信优化

DeepSeek-V3 的 MoE expert 分布跨节点，因此需要高效 all-to-all。

公开报告中提到：

```text
IB 跨节点
NVLink 节点内
先跨节点发送到目标 node 的同 index GPU
再节点内转发到具体 expert GPU
定制通信 kernel
使用部分 SM 处理通信，并与计算重叠
```

### 5.4 对数据链路的启示

训练系统优化和数据链路的关系在于：

```text
训练系统越贵，数据空转越不可接受
```

如果数据质量差、重复多、读取慢，会直接浪费：

```text
H800/H100 GPU hours
通信资源
checkpoint 周期
benchmark 周期
```

因此数据团队必须提供：

```text
去重后的有效 token
高吞吐 shard
稳定长度分布
可回溯 dataset version
质量池与配方版本
```

---

## 6. RL 算法：GRPO 如何替代 PPO 中的 critic

### 6.1 PPO 的痛点

PPO 在 RLHF/RLVR 中通常需要：

```text
policy model
reference model
reward model
value / critic model
```

其中 value model 通常和 policy 同量级，带来显存与计算开销。

### 6.2 GRPO 的核心思想

GRPO，即 Group Relative Policy Optimization，核心变化是：

```text
不训练单独 value / critic model
对同一个 prompt 采样一组 outputs
用这一组 outputs 的 reward 均值/标准差估计 baseline
用组内相对 reward 计算 advantage
```

对每个问题 `q`：

```text
sample G outputs: o1, o2, ..., oG
compute rewards: r1, r2, ..., rG
normalize: advantage_i = (ri - mean(r)) / std(r)
update policy with clipped objective + KL regularization
```

### 6.3 GRPO 与 PPO 对比

| 维度 | PPO | GRPO |
|---|---|---|
| baseline | value/critic model | group reward mean/std |
| 额外模型 | 需要 value model | 不需要 value model |
| 显存成本 | 高 | 更低 |
| 采样需求 | 每 prompt 可少样本 | 每 prompt 需要 group samples |
| 优势估计 | GAE + value | group relative advantage |
| 适用场景 | 通用 RLHF | 数学/代码等可验证 reward 场景更自然 |

### 6.4 DeepSeekMath 中的 GRPO 设置

DeepSeekMath 公开的 RL 设置包括：

```text
基座：DeepSeekMath-Instruct 7B
RL 数据：GSM8K / MATH 相关 CoT instruction data，约 144K questions
每个 question 采样 64 outputs
max length = 1024
training batch size = 1024
policy learning rate = 1e-6
KL coefficient = 0.04
```

GRPO 带来的效果：

```text
GSM8K: 82.9% → 88.2%
MATH: 46.8% → 51.7%
CMATH: 84.6% → 88.8%
```

### 6.5 Outcome supervision 与 process supervision

DeepSeekMath 讨论了两类 reward：

| 类型 | 说明 |
|---|---|
| Outcome supervision | 最终答案/整体输出给 reward |
| Process supervision | 每个 reasoning step 给 reward |

对于数学/代码，过程监督有潜力更细粒度，但成本更高。

### 6.6 GRPO 对数据团队的要求

GRPO 的关键不是“有没有人类偏好数据”，而是：

```text
能否构造可验证 prompt
能否稳定计算 reward
能否产生足够多样的 group samples
能否避免 benchmark 泄漏
能否记录 prompt/reward/policy version
```

因此数据团队需要建设：

```text
RL prompt pool
verifiable reward function
reward execution sandbox
benchmark contamination filter
rollout manifest
reward trace
```

### 6.7 GRPO 伪代码

下面是从工程角度理解 GRPO 的简化伪代码，重点是 group rollout 与 group-relative advantage：

```python
for batch in prompt_loader:
    loss = 0

    for prompt in batch:
        responses = policy.generate(prompt, n=G)
        rewards = reward_fn(prompt, responses)

        reward_mean = rewards.mean()
        reward_std = rewards.std() + 1e-6
        advantages = (rewards - reward_mean) / reward_std

        for response, advantage in zip(responses, advantages):
            ratio = exp(policy.logprob(response) - old_policy.logprob(response))
            clipped_ratio = clip(ratio, 1 - eps, 1 + eps)
            policy_loss = -min(ratio * advantage, clipped_ratio * advantage)
            kl_loss = beta * kl(policy, reference_policy, prompt, response)
            loss += policy_loss + kl_loss

    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

这段伪代码对应的数据产物是：

```text
prompt → G 个 response → G 个 reward → group mean/std → advantage → policy update
```

### 6.8 Verifiable Reward 设计

GRPO 能否落地，关键不只是算法，而是 reward 是否可靠。不同任务的 reward 形态不同：

| 任务类型 | Reward 来源 | 主要风险 | 数据链路要求 |
|---|---|---|---|
| 数学 | final answer match、symbolic equivalence、数值容差 | 答案格式多样、等价表达难识别 | 标准答案规范化、公式解析、容差规则 |
| 代码 | unit test pass rate、静态检查、lint、编译结果 | 测试覆盖不足、hack 测试、依赖环境复杂 | sandbox、依赖锁定、测试版本管理 |
| API 使用 | 参数校验、真实/模拟 API 调用结果 | API 版本变化、外部服务不稳定 | mock server、API schema version、调用日志 |
| 工具调用 | tool result correctness、任务完成率 | 工具失败和模型失败难区分 | tool trace、error taxonomy、retry policy |
| 多轮 Agent | final task success、人工/LLM judge | credit assignment 难、评测成本高 | conversation lineage、step-level trace、judge calibration |
| 开放问答 | reward model、LLM judge、人类偏好 | reward hacking、主观性强 | judge prompt version、抽样复核、争议仲裁 |

优先级建议：

```text
Phase 1：数学/代码/API 这类可验证任务
Phase 2：工具调用、多轮 Agent
Phase 3：开放式偏好与复杂人类反馈
```

### 6.9 RL 数据表设计草案

如果要把 GRPO/RL 纳入当前数据湖，建议至少设计以下表：

```sql
CREATE TABLE rl_prompts (
  prompt_id STRING,
  dataset_version STRING,
  source_type STRING,
  task_type STRING,
  difficulty STRING,
  prompt_text STRING,
  reference_answer STRING,
  reward_spec_version STRING,
  contamination_status STRING,
  created_at TIMESTAMP
);

CREATE TABLE rl_rollouts (
  rollout_id STRING,
  prompt_id STRING,
  policy_version STRING,
  reference_policy_version STRING,
  generation_config_version STRING,
  response_text STRING,
  response_token_count BIGINT,
  logprob_path STRING,
  rollout_group_id STRING,
  created_at TIMESTAMP
);

CREATE TABLE rl_rewards (
  reward_id STRING,
  rollout_id STRING,
  reward_type STRING,
  reward_score DOUBLE,
  pass_test BOOLEAN,
  reward_reason STRING,
  sandbox_log_path STRING,
  reward_model_version STRING,
  reward_function_version STRING,
  created_at TIMESTAMP
);

CREATE TABLE rl_advantages (
  rollout_id STRING,
  rollout_group_id STRING,
  group_reward_mean DOUBLE,
  group_reward_std DOUBLE,
  advantage DOUBLE,
  kl_to_reference DOUBLE,
  policy_loss DOUBLE,
  created_at TIMESTAMP
);
```

这些表应与 `dataset_version`、`checkpoint_id`、`training_run_id` 打通，避免 RL 阶段成为不可追溯的黑盒。

---

## 7. verl 框架视角：如何工程化 DeepSeek 类 RL 流程

### 7.1 verl 是什么

verl 是 ByteDance Seed 发起并由社区维护的开源 RL post-training 框架。

它支持：

```text
PPO / GRPO / ReMax / RLOO / DAPO 等算法
FSDP / FSDP2 / Megatron-LM 训练后端
vLLM / SGLang / HF Transformers rollout generation
model-based reward 与 function-based reward
multi-turn / tool calling / sandbox 等扩展
```

### 7.2 为什么 verl 对 DeepSeek 类 RL 有参考价值

DeepSeek 的 GRPO 思路要工程化，需要解决：

```text
actor rollout 高吞吐生成
reward 批量计算
policy update
reference KL
训练/推理引擎切换
模型 resharding
多 GPU / 多节点调度
实验追踪
```

verl 正是把这些 RL post-training 数据流做成框架化能力。

### 7.3 用 verl 实现 GRPO 的抽象数据流

```text
Prompt Dataset
  ↓
Actor / Old Policy rollout: 每个 prompt 生成 G 个 responses
  ↓
Reward Function / Reward Model 打分
  ↓
Group relative advantage 计算
  ↓
Policy update with GRPO objective
  ↓
Reference KL regularization
  ↓
Eval + checkpoint
```

### 7.4 verl 对当前方案的启示

如果当前团队要做 RL 数据链路，应尽早定义：

```text
prompt schema
response schema
reward schema
rollout manifest
policy version
reward version
sandbox execution log
```

这些内容应和 `11-lineage-versioning-reproducibility.md` 的 dataset/checkpoint 血缘打通。

### 7.5 RL post-training 数据流与产物

从框架视角看，一个可工程化的 GRPO/RLVR 训练流可以拆成以下阶段：

```text
Prompt Dataset
  ↓
Actor Rollout Workers
  ↓
vLLM / SGLang / HF Generation Engine
  ↓
Reward Workers / Sandbox / Unit Tests / LLM Judge
  ↓
Group Advantage Computation
  ↓
Policy Trainer
  ↓
Checkpoint + Evaluation
  ↓
Bad Case Mining + Data Feedback
```

每个阶段都应沉淀可追溯产物：

| 阶段 | 关键产物 | 需要记录的元数据 |
|---|---|---|
| Prompt | `prompt_id`, `prompt_text`, `task_type` | 来源、难度、去污染状态、标准答案 |
| Rollout | `response_text`, `tokens`, `logprobs` | policy version、采样参数、group id |
| Reward | `reward_score`, `pass_test`, `reason` | reward function version、sandbox log、judge prompt version |
| Advantage | `advantage`, `group_mean`, `group_std` | group size、KL、normalization method |
| Trainer | `policy_loss`, `kl`, `entropy` | optimizer、LR、batch size、checkpoint id |
| Eval | benchmark score、bad cases | eval dataset version、metric version、contamination report |

这张表本质上是 RL 阶段的 lineage 设计。没有这些记录，RL 训练即使分数提升，也很难解释提升来自数据、reward、采样策略还是训练超参。

---

## 8. Muon 优化器：应如何放在 DeepSeek 语境里理解

用户提到的 `Muon` 是近年大模型训练优化器方向的重要话题，但需要谨慎处理。

### 8.1 当前边界

截至本文使用的公开资料：

```text
DeepSeek-V3 Technical Report 明确写到 AdamW、FP8、DualPipe、DeepSeekMoE、MLA 等；
没有把 Muon 作为 DeepSeek-V3/R1 官方训练组件公开描述。
```

因此本文不把 Muon 写成 DeepSeek 的已公开事实。

### 8.2 Muon 可以作为什么来分析

Muon 可以作为后续“优化器与架构协同”的调研方向：

```text
是否能降低训练不稳定
是否能提高 token efficiency
是否能减少 warmup / tuning 成本
是否适合 MoE / dense 不同参数块
是否能替代或混合 AdamW
```

### 8.3 对当前文档体系的建议

如果要深入 Muon，建议单独写：

```text
docs/13-optimizer-and-training-efficiency.md
```

而不是把 Muon 混在 DeepSeek 已公开 pipeline 里。

---

## 9. 评测体系：数学、代码、知识、中文与开放式评测

### 9.1 DeepSeekMath 评测

DeepSeekMath 覆盖：

```text
GSM8K
MATH
SAT
OCW Courses
MMLU-STEM
MGSM-zh
CMATH
Gaokao-MathCloze
Gaokao-MathQA
miniF2F
HumanEval
MBPP
MMLU
BBH
```

特点：

```text
同时评估英文和中文数学
同时评估无工具 CoT 和 Python tool use
同时关注数学、形式化证明、代码和通用推理
```

### 9.2 DeepSeek-V3 评测

DeepSeek-V3 报告覆盖：

```text
知识类：MMLU, MMLU-Pro, GPQA
事实性：SimpleQA, Chinese SimpleQA
数学：MATH-500 等
代码：LiveCodeBench 等
开放式评测
DeepSeek-V3 as generative reward model
```

### 9.3 对当前方案的启示

评测不能只看一个综合分。

建议当前 70B 方案至少建立：

```text
base model evaluation
code evaluation
math/reasoning evaluation
Chinese evaluation
API/tool-use evaluation
safety evaluation
contamination report
open-ended human/LLM judge evaluation
```

并且每次评测结果要和：

```text
dataset_version
mixture_recipe_version
quality_pipeline_version
checkpoint_id
```

关联。

### 9.4 评测结果与数据版本绑定

建议把评测结果设计成可追溯数据资产，而不是只保留一张 benchmark 截图。

```sql
CREATE TABLE model_evaluation_runs (
  eval_run_id STRING,
  checkpoint_id STRING,
  training_run_id STRING,
  dataset_version STRING,
  mixture_recipe_version STRING,
  eval_suite_version STRING,
  contamination_report_id STRING,
  judge_model_version STRING,
  created_at TIMESTAMP
);

CREATE TABLE model_evaluation_metrics (
  eval_run_id STRING,
  benchmark_name STRING,
  task_group STRING,
  metric_name STRING,
  metric_value DOUBLE,
  sample_count BIGINT,
  pass_at_k INT,
  decoding_config_version STRING,
  created_at TIMESTAMP
);

CREATE TABLE model_evaluation_bad_cases (
  eval_run_id STRING,
  case_id STRING,
  benchmark_name STRING,
  prompt_text STRING,
  model_output STRING,
  expected_answer STRING,
  error_type STRING,
  linked_data_source STRING,
  suggested_data_action STRING,
  created_at TIMESTAMP
);
```

评测系统要能回答：

```text
这次 MATH 提升，是因为数学数据比例增加？
还是因为 premium_pool 提高？
还是因为 RL prompt 改了？
还是因为 benchmark 被污染？
```

如果评测不能回连数据版本，就无法指导下一轮数据配方。

---

## 10. 推理系统：prefill/decode 分离、KV Cache 与 expert 负载均衡

### 10.1 DeepSeek-V3 的推理部署思路

DeepSeek-V3 报告中明确将推理分成：

```text
prefilling stage
decoding stage
```

原因：

```text
prefill 更偏大批量计算
decode 更偏低延迟、小步生成、KV cache 访问
```

### 10.2 Prefill 阶段

公开报告中，prefill 最小部署单元为：

```text
4 nodes × 8 GPUs = 32 GPUs
attention: TP4 + SP + DP8
MoE: EP32
```

还使用：

```text
redundant experts
expert load statistics
周期性调整高负载 expert 的冗余部署
两个 micro-batches overlap communication/computation
```

### 10.3 Decode 阶段

decode 最小部署单元为：

```text
40 nodes × 8 GPUs = 320 GPUs
attention: TP4 + SP + DP80
MoE: EP320
```

特点：

```text
每 GPU 通常只 host 一个 expert
共享 expert 也视作 routed expert
使用 redundant experts 和 direct point-to-point IB transfer
```

### 10.4 MLA 与 KV Cache

MLA 降低 KV cache，使长上下文推理更经济。

对当前方案的启示：

```text
如果未来模型没有 MLA 类机制，长上下文数据收益会被推理成本限制。
如果模型有 MLA/长上下文机制，应重点保留高质量长文档、长代码上下文、API 文档链路。
```

### 10.5 MTP 与 speculative decoding

DeepSeek-V3 的 MTP 训练目标主要用于提升主模型能力，但报告也指出 MTP 模块可用于 speculative decoding。

这说明训练目标也可以反过来服务推理系统。

---

## 11. 对当前 70B 数据链路方案的启示

### 11.1 数据链路不能只做“清洗”，要做领域发现

DeepSeekMath 的 fastText 迭代召回说明：

```text
高价值领域数据往往藏在 Common Crawl 中
需要 seed → classifier → domain discovery → path annotation → retrain 的闭环
```

当前方案可复用到：

```text
代码教程
API 文档
数学推理
系统设计文档
高质量中文技术站点
```

### 11.2 benchmark 去污染必须产品化

建议单独建设：

```text
benchmark_contamination_service
```

功能：

```text
exact match
n-gram match
MinHash near match
embedding semantic match
LLM judge suspicious cases
```

### 11.3 RL 数据链路要和 pretrain 数据链路分开设计

GRPO/RL 需要的数据不是普通文本，而是：

```text
prompt
multiple rollouts
reward
advantage
policy version
reference version
sandbox log
```

这应成为独立数据资产。

### 11.4 质量分不是最终目标，训练反馈才是闭环

DeepSeek 的经验说明：

```text
数据质量 → pretrain/SFT/RL → benchmark → 再调整数据
```

当前方案中的 `09`、`10`、`11` 已经具备闭环基础：

```text
quality_pool
mixture_recipe_version
dataset_version
checkpoint lineage
```

下一步应把 RL rollout/reward 也纳入 lineage。

### 11.5 推理系统会反向影响数据价值

MLA、MTP、MoE inference 决定哪些训练数据最终能产生线上价值：

```text
长上下文能力是否可服务化
代码能力是否能低延迟生成
推理成本是否允许大规模上线
```

因此数据团队需要理解推理侧约束。

### 11.6 DeepSeek 做法与当前 70B 方案映射

| DeepSeek 做法 | 当前方案已有基础 | 需要补充 | 优先级 |
|---|---|---|---|
| Common Crawl 领域数据挖掘 | `01-data-sources.md`、`03-processing-pipeline.md` | seed-classifier-domain discovery 迭代机制 | P0 |
| Benchmark 去污染 | 去重与质量过滤已有 | 单独的 benchmark contamination service | P0 |
| 14.8T tokens 高质量配方 | `10-data-mixture-strategy.md` | 真实训练反馈驱动的配方迭代 | P1 |
| 数据质量分层 | `09-data-quality-evaluation.md` | 与 benchmark/bad case 的反向更新 | P1 |
| Dataset/checkpoint 血缘 | `11-lineage-versioning-reproducibility.md` | RL rollout/reward lineage | P1 |
| GRPO/RL 数据链路 | 仅有 RLHF 概念 | prompt/rollout/reward/advantage 表设计 | P1 |
| MLA/MoE/MTP 对数据价值的影响 | `08-training-overview.md` 有训练概览 | 架构约束反推数据保留策略 | P2 |
| 推理 prefill/decode 分离 | 成本模型已有推理费用 | 不同服务形态的 token 成本模型 | P2 |
| FP8/DualPipe 工程优化 | `05-cost-model.md` 有 GPU 成本 | 训练吞吐、GPU 空转、数据读取性能指标 | P2 |

结论：当前文档体系已经覆盖了数据湖、处理、质量、配方、血缘和成本，但还缺三类 DeepSeek 启发下的关键专题：

```text
benchmark contamination
RL prompt / rollout / reward 数据链路
训练反馈驱动的数据配方实验体系
```

### 11.7 DeepSeek 对成本模型的启示

DeepSeek-V3 公开的训练成本可以作为校准参照：

```text
DeepSeek-V3：2.788M H800 GPU hours
训练 tokens：14.8T
平均每 1T tokens ≈ 188K H800 GPU hours
按 $2/GPU-hour 估算：总训练成本约 $5.576M
```

和当前 70B 成本模型可以粗略对比：

| 项目 | DeepSeek-V3 | 当前 70B 方案估算 |
|---|---:|---:|
| 模型形态 | 671B MoE / 37B activated | 70B dense 假设 |
| 训练 tokens | 14.8T | 1T-2T 起步 |
| 训练硬件 | 2048 H800 | 1024 H100 假设 |
| 公开训练 GPU hours | 2.788M H800 GPUh | 2T 约 531K-683K H100 GPUh |
| 每 1T tokens GPU hours | 约 188K H800 GPUh | 约 265K-342K H100 GPUh |
| 主要优化 | MoE、FP8、DualPipe、MLA/MTP | dense 训练、常规并行估算 |
| 成本启示 | 架构和系统优化显著影响 token 成本 | 质量过滤和吞吐优化能直接省 GPU 费 |

这说明：

```text
数据团队每减少 5%-10% 无效 token，影响的不只是存储成本，而是训练 GPU、checkpoint、评测和推理验证的连锁成本。
```

因此 `05-cost-model.md` 中的训练成本应和 `09/10/11/12` 的质量、配方、血缘文档联动，而不是独立估算。

---

## 12. 可复用路线图

### 12.1 Phase 1：补齐研究与诊断能力

```text
建立 benchmark contamination 文档
建立领域数据发现 pipeline
建立 RL prompt/reward schema
记录 DeepSeek 类训练系统的工程约束
```

### 12.2 Phase 2：小规模复现 GRPO 流程

```text
选择 math/code 可验证任务
构造 10K-100K prompt pool
每 prompt 采样 G=8/16/32 responses
使用 rule reward / unit test reward / answer match reward
用 verl 或内部框架跑 GRPO
记录 rollout manifest 和 reward trace
```

### 12.3 Phase 3：和 70B 数据平台打通

```text
RL prompt pool 进入 Iceberg
rollout/reward 进入数据湖
policy/checkpoint 与 dataset_version 关联
bad case 回流到 mixture strategy
```

---

## 13. 风险与未公开部分

### 13.1 未公开或不完整部分

DeepSeek 公开材料并没有完整公开：

```text
14.8T tokens 的详细配方
SFT 数据完整来源
DeepSeek-R1 全部 RL prompt 构造细节
Reward model 完整训练数据
线上推理流量与真实 SLO
全部工程实现代码
```

因此本文不能把这些部分写成确定事实。

### 13.2 复用风险

| 风险 | 说明 |
|---|---|
| 数据不可得 | DeepSeek 的数据资产无法完全复现 |
| 工程门槛高 | DualPipe、FP8、MoE all-to-all 需要强系统能力 |
| RL reward 脆弱 | 规则 reward 容易被模型 hack |
| benchmark 污染 | 推理模型尤其容易因公开题库污染虚高 |
| 推理成本高 | MoE 省计算但部署复杂 |
| 框架错配 | verl 可参考，但不等于 DeepSeek 官方训练系统 |

### 13.3 正确借鉴方式

不要直接照搬：

```text
参数规模
MoE 规模
GRPO 超参
训练集比例
推理部署 GPU 数
```

应该借鉴：

```text
数据发现闭环
benchmark 去污染机制
质量-配方-训练反馈闭环
critic-free RL 思路
训练/推理 co-design
成本意识
```

### 13.4 哪些经验不能直接照搬

| DeepSeek 经验 | 不应直接照搬的原因 | 更合理的做法 |
|---|---|---|
| 671B MoE | MoE 训练、通信、推理部署门槛极高 | 先以 70B dense 或较小 MoE 做可控实验 |
| 14.8T tokens | 数据质量、训练预算、评测能力必须匹配 | 先做 1T-2T 高质量 token 与 ablation |
| FP8 大规模训练 | 依赖硬件、kernel、数值稳定经验 | 先建立 BF16 稳定基线，再评估低精度收益 |
| DualPipe | 需要深度训练框架和通信优化能力 | 先优化数据读取、shard、batch shape，减少 GPU 空转 |
| GRPO 大 group rollout | reward 不可靠时会放大错误优化 | 先从 math/code/API 可验证任务开始 |
| 大规模 prefill/decode 分离 | 适合超大 MoE 服务，不一定适合早期 70B dense | 先用 vLLM/SGLang 建立推理成本基线 |
| 公开 benchmark 高分 | 可能受去污染、评测协议、decode 设置影响 | 每次评测必须绑定 contamination report 和 decoding config |

正确的复用路径是：

```text
先复用数据闭环和评测闭环
再复用 RL 数据资产设计
最后再考虑训练系统和推理系统的复杂优化
```

---

## 参考资料

1. DeepSeek-V3 Technical Report, arXiv:2412.19437. <https://arxiv.org/html/2412.19437v1>
2. DeepSeekMath: Pushing the Limits of Mathematical Reasoning in Open Language Models, arXiv:2402.03300. <https://arxiv.org/abs/2402.03300>
3. verl: Volcano Engine Reinforcement Learning for LLMs. <https://github.com/verl-project/verl>
