# 数据源矩阵——详细设计

> 主文档：[README.md](../README.md) 第 2 章

---

## 目录

- [1. 数据源全景](#1-数据源全景)
- [2. GitHub 代码数据](#2-github-代码数据)
  - [2.1 三级处理分层](#21-三级处理分层)
  - [2.2 Repo 重要性评估模型](#22-repo-重要性评估模型)
  - [2.3 有价值的 Commit 过滤](#23-有价值的-commit-过滤)
  - [2.4 代码去重策略](#24-代码去重策略)
  - [2.5 Issue-Commit-Code 完整修复链](#25-issue-commit-code-完整修复链)
  - [2.6 代码数据配比策略](#26-代码数据配比策略)
- [3. 网页文档数据](#3-网页文档数据)
- [4. API 文档数据](#4-api-文档数据)
- [5. GitHub Issue/PR/Discussion](#5-github-issueprdiscussion)
- [6. 数据源优先级矩阵](#6-数据源优先级矩阵)

---

## 1. 数据源全景

| 数据源 | 采集策略 | 采集规模 | 价值等级 | Phase 1 优先级 |
|---|---|---|---|---|
| **GitHub 代码（静态快照）** | GitHub Archive + API，全量 repo 文件树 | 1000 万+ repo，PB 级 | ⭐⭐⭐⭐⭐ | 🔴 最高 |
| **GitHub 代码（演进式挖掘）** | 精选 repo 的 git history + diff | 10 万 repo，TB 级 | ⭐⭐⭐⭐⭐ | 🟡 Phase 2 主力 |
| **网页文档** | 定向深挖 + 通用抓取，隔离处理 | 10-100PB 级 | ⭐⭐⭐⭐ | 🔴 高 |
| **API 文档** | 定向抓取结构化文档站 | TB 级 | ⭐⭐⭐ | 🟡 可并行 |
| **GitHub Issue/PR/Discussion** | GitHub Archive + API | TB 级 | ⭐⭐⭐⭐ | 🔴 高 |

---

## 2. GitHub 代码数据

### 2.1 三级处理分层

并非所有 repo 都走相同的处理深度。按重要性分级投入资源：

```
        ┌─────────────────────────────────┐
        │        L1: 全量快照              │
        │  所有通过粗过滤的 repo            │
        │  ~1000 万 repo                   │
        │  只下载 master/main 最新文件快照   │
        │  产出：代码语料（函数/类/文件级）   │
        └──────────────┬──────────────────┘
                       │
                       ↓
        ┌─────────────────────────────────┐
        │        L2: 高价值 repo（深度）    │
        │  重要性评分 top 5%                │
        │  ~50 万 repo                     │
        │  L1 + 完整 commit history + diff  │
        │  + Issue/PR 文本                  │
        │  产出：有价值 commit 的代码变更    │
        └──────────────┬──────────────────┘
                       │
                       ↓
        ┌─────────────────────────────────┐
        │      L3: 精选 repo（全链路挖掘）   │
        │  重要性评分 top 1%                │
        │  ~10 万 repo                     │
        │  + 额外筛选条件                   │
        │  L2 + Issue-Commit-Code 关联     │
        │  + Solution Extraction          │
        │  + Verification                 │
        │  产出：完整修复链训练数据           │
        └─────────────────────────────────┘
```

**L3 额外筛选条件**：
- Issue 中明确引用了 commit hash（可自动关联）
- 有明确的 bug-fix 语义（commit message 含 "fix"/"bug"/"resolve"/"close"）
- CI 通过率 > 80%（修复是有效的，不是引入新 bug）
- 项目语言在 Top-15 中（AST 可解析）
- 有至少 10 个以上符合上述条件的历史 Issue

---

### 2.2 Repo 重要性评估模型

#### 2.2.1 设计原则

- 不能只用 stars（严重滞后，简单 Demo 可能万星，高质量基础设施可能只有几百星）
- 多维度加权，综合反映社区活跃度、代码质量和维护健康度
- 数据源结合 GitHub Archive（BigQuery）做离线批量分析，避免 API rate limit

#### 2.2.2 四维度评估模型

| 维度 | 权重 | 信号 | 数据来源 | 说明 |
|---|---|---|---|---|
| **社区活跃度** | 30% | stars、forks、近 6 个月 issue/PR 数量、contributor 数 | GitHub Archive (BigQuery) | 反映"当下谁在用、谁在贡献" |
| **代码质量代理** | 25% | CI 通过率、测试文件占比、lint 工具配置检测 | GitHub API + 文件分析 | 反映"代码是否写得好" |
| **维护健康度** | 20% | 最近一次 commit 时间、近 6 个月 commit 频率、未关闭 issue 比例 | GitHub Archive | 反映"是否仍在维护" |
| **领域权威性** | 25% | 是否官方组织维护、被其他高星 repo 依赖次数、pkg 下载量（npm/PyPI/crates.io） | GitHub Archive + 包管理器 API | 反映"是否被社区认可" |

#### 2.2.3 评分计算

```
repo_score = 0.30 × community_score
           + 0.25 × code_quality_score
           + 0.20 × maintenance_score
           + 0.25 × authority_score

每个子分数归一化到 0-1
```

**community_score 计算**：

```
community_score = clamp(
    0.30 × log10(stars + 1) / log10(100000)       # 上限 10 万星
  + 0.20 × log10(forks + 1) / log10(10000)
  + 0.25 × min(recent_issues_per_month / 50, 1.0)  # 近期活跃
  + 0.25 × min(contributors / 100, 1.0)
, 0, 1)
```

**code_quality_score 计算**：

```
code_quality_score = clamp(
    0.35 × ci_pass_rate                               # CI 通过率
  + 0.30 × test_file_ratio                            # 测试文件占比
  + 0.20 × has_lint_config                            # 是否有 lint 配置
  + 0.15 × has_contributing_guide                     # 是否有贡献指南
, 0, 1)
```

**maintenance_score 计算**：

```
maintenance_score = clamp(
    0.35 × (1 - days_since_last_commit / 365)         # 越近越好
  + 0.30 × min(commits_last_6m / 100, 1.0)            # 近 6 月活跃度
  + 0.35 × (1 - open_issues_ratio)                    # 未关闭 issue 越少越好
, 0, 1)
```

**authority_score 计算**：

```
authority_score = clamp(
    0.30 × is_official_org                              # 官方组织（0/1）
  + 0.30 × min(dependent_repos / 1000, 1.0)            # 被依赖 repo 数
  + 0.40 × min(pkg_downloads_per_month / 1e6, 1.0)     # 包下载量
, 0, 1)
```

#### 2.2.4 数据采集流程

```
1. 从 GitHub Archive (BigQuery) 离线批量查询：
   - 所有 repo 的 stars/forks/issue/PR/contributor 统计
   - 近 6 个月的 commit/issue/PR 活动数据
   - 依赖关系（package.json / requirements.txt / go.mod 中的引用）

2. 从 GitHub API 补充：
   - CI 配置检测（.github/workflows/ 目录）
   - lint 配置检测（.eslintrc / .pylintrc / .golangci.yml 等）
   - 最近一次 commit 时间

3. 从包管理器 API 补充：
   - npm (registry.npmjs.org)
   - PyPI (pypi.org)
   - crates.io
   - 其他

4. 批量计算 score，存入 repo_metadata 表
```

#### 2.2.5 更新频率

| 数据 | 更新频率 | 说明 |
|---|---|---|
| 社区活跃度 | 每月 | 从 GitHub Archive 月度快照更新 |
| 代码质量代理 | 每季度 | CI/测试覆盖率变化慢 |
| 维护健康度 | 每月 | 与社区活跃度同步 |
| 领域权威性 | 每季度 | 依赖关系变化慢 |

---

### 2.3 有价值的 Commit 过滤

#### 2.3.1 为什么需要过滤

L2 级别 50 万 repo，平均每个 repo 1000 commits，总计 5 亿次 commit。其中 80% 以上是自动生成、微小变更、格式调整等无训练价值的 commit。需要三层过滤规则。

#### 2.3.2 三层过滤规则

**L1：格式过滤（自动丢弃）**

| 过滤条件 | 判定 | 理由 |
|---|---|---|
| commit message 为空或纯数字 | 丢弃 | 无意义 |
| 纯 Merge commit（`Merge branch 'xxx' into yyy`） | 丢弃 | 自动合并，非人工意图 |
| commit message 仅含 bot 签名（`[bot]` / `dependabot` / `renovate` / `github-actions`） | 丢弃 | 自动化提交 |
| commit message 仅含 `Auto commit` / `CI build` 等 | 丢弃 | 自动化提交 |

**L2：语义过滤**

| 关键词模式 | 语义分类 | 动作 |
|---|---|---|
| `fix typo` / `fix comment` / `update docs` / `update readme` / `format` / `lint` / `style` / `whitespace` / `cosmetic` | 仅格式/文档 | **丢弃** |
| `fix` / `bug` / `resolve` / `close #` / `patch` / `hotfix` / `workaround` / `regression` | **bugfix** | 保留，最高优先级 |
| `add` / `implement` / `feature` / `support` / `introduce` / `new` | **feature** | 保留 |
| `refactor` / `rewrite` / `restructure` / `cleanup` / `simplify` / `extract` | **refactor** | 保留 |
| `update` / `modify` / `change` / `improve` / `tweak` / `enhance` / `optimize` / `perf` | **improvement** | 保留 |
| `remove` / `deprecate` / `drop` / `delete` | **removal** | 保留 |
| `test` / `add test` / `test coverage` | **test** | 保留（优先级较低） |
| 其他 / 无关键词匹配 | **other** | 保留，但优先级最低 |

**L3：Diff 规模过滤**

| 条件 | 判定 | 理由 |
|---|---|---|
| 变更文件数 > 50 | 丢弃 | 批量重构/自动生成/依赖更新，无法精准定位 |
| 变更行数 > 1000 | 丢弃 | 大范围重构，上下文过于庞大 |
| 变更行数 < 3（仅代码行，不含注释/空行） | 丢弃 | 太微小，无学习价值 |
| diff 中仅含 import/依赖变更、版本号修改 | 丢弃 | 非代码逻辑变更 |
| diff 中仅含测试文件变更 | 保留但标记 `type=test_only` | 优先级低，可用于测试生成任务 |

#### 2.3.3 过滤后分类优先级

| 优先级 | 类型 | 用途 | 预估占比 |
|---|---|---|---|
| P0 | bugfix | Issue-Commit-Code 完整修复链 | ~15% |
| P1 | feature | 了解功能实现模式 | ~25% |
| P2 | refactor | 代码重构模式 | ~20% |
| P3 | improvement | 一般代码改进 | ~25% |
| P4 | test / removal / other | 特定场景 | ~15% |

---

### 2.4 代码去重策略

#### 2.4.1 代码去重的特殊挑战

代码去重比文档去重更复杂，因为：
- 同一个函数可能有不同的格式化版本（缩进、换行、变量命名）
- fork 仓库产生大量完全相同的文件
- 模板项目（create-react-app、boilerplate）生成大量结构相似的代码
- 第三方库的 vendored 代码（如 `vendor/`、`node_modules/`）

#### 2.4.2 三层代码去重架构

```
┌─────────────────────────────────────────────────────┐
│ 第 1 层：文件级 SHA256 精确去重                        │
│ 适用：fork 仓库完全相同文件、vendored 库                │
│ 做法：文件内容 SHA256，全量哈希表比对                    │
│ 框架：Spark，全量（粗过滤阶段）                          │
│ 产出：去除完全相同的文件（保留 importance 最高的那份）    │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ 第 2 层：函数级 MinHash + LSH 近重复去重               │
│ 适用：格式化变体、变量重命名、小幅修改的同一函数           │
│ 做法：                                                │
│   1. AST 解析后提取每个函数（Top-15 语言）              │
│   2. 对函数体做代码标准化（normalize）：                │
│      - 统一缩进                                       │
│      - 替换变量名为 placeholder（var_1, var_2, ...）   │
│      - 去除注释                                        │
│   3. 标准化后的代码做字符 3-gram MinHash（128 签名）     │
│   4. LSH 分桶，Jaccard ≥ 0.9 视为近重复                │
│ 框架：Spark（粗加工阶段）                               │
│ 产出：去除近重复函数（保留 importance 最高的那份）        │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ 第 3 层：函数级 Embedding 语义去重                     │
│ 适用：功能等价但实现不同（如不同排序算法实现同一功能）    │
│ 做法：                                                │
│   1. 对函数代码生成 code embedding                     │
│   2. 在同一语言簇内做 ANN 检索 top-K                    │
│   3. 余弦相似度 ≥ 0.95 视为语义重复                     │
│ 框架：Lance ANN 索引（细加工阶段）                      │
│ ⚠️ Phase 2 启用，仅对高价值代码子集执行                  │
└─────────────────────────────────────────────────────┘
```

#### 2.4.3 特殊处理

| 场景 | 处理方式 |
|---|---|
| **vendored 代码**（`vendor/`、`node_modules/`、`third_party/`） | 路径规则直接丢弃，不进入去重流程 |
| **自动生成代码**（`*.pb.go`、`*.generated.*`、`__init__.py` 空文件） | 文件名规则丢弃 |
| **模板/脚手架项目** | 第 2 层 MinHash 会捕获，Jaccard 阈值可上调至 0.95 |
| **测试代码** | 保留但标记 `is_test = true`，在配比阶段单独控制比例 |

#### 2.4.4 代码标准化（Normalization）规则

```
输入：原始函数体代码
处理：
  1. 用 Tree-sitter 解析 AST
  2. 遍历 AST 节点：
     - identifier 节点 → 替换为 var_1, var_2, ...（按首次出现顺序）
     - comment 节点 → 删除
     - string_literal 节点 → 替换为 "STR"（保留字符串结构）
     - number_literal 节点 → 替换为 "0"（保留数字位置）
  3. 统一缩进为 4 空格
  4. 统一换行为 \n
输出：标准化后的代码字符串
```

**为什么不直接对比 AST 树结构？**
- AST 完全相等过于严格（变量名不同就判为不同）
- AST 编辑距离计算成本高，不如 MinHash + LSH 可规模化
- 标准化后的 MinHash 已经捕获了足够的结构相似性

---

### 2.5 Issue-Commit-Code 完整修复链

#### 2.5.1 目标

从 L3 精选 repo 中构造完整的代码修复训练数据，自动串联以下环节：

```
Issue 描述 → 关联的 Commit(s) → 修复前/后代码 → 验证通过
```

#### 2.5.2 Issue-Commit 关联（三种方式）

| 关联方式 | 可靠性 | 预估覆盖率 | 实现方式 |
|---|---|---|---|
| **方式 1：GitHub 自动引用** | ⭐⭐⭐⭐⭐ 极高 | ~30-40% | commit message 正则提取 `fixes #123` / `closes #123` / `resolves #123` |
| **方式 2：PR 关联** | ⭐⭐⭐⭐ 高 | ~20-30% | 解析 PR body 中的 `closes #issue` 引用 + PR 的 merge commit 关联 |
| **方式 3：语义模糊匹配** | ⭐⭐⭐ 中 | ~10-20% | 若 issue 关闭时间与某 fix commit 接近（±3 天），用 LLM 判断 issue 描述与 commit message 是否语义相关 |

**方式 3 的 LLM 判断 Prompt**：

```
你是一个代码审查专家。请判断以下 Issue 和 Commit 是否描述的是同一个 bug 修复。
只回答 YES 或 NO。

Issue 标题：{issue_title}
Issue 描述：{issue_body}

Commit 标题：{commit_message}
Commit 修改的文件：{changed_files}
Commit 修改摘要：{diff_summary}

这个 Commit 是否修复了 Issue 中描述的 bug？
```

**关联结果存储**：

```
issue_commit_link 表:
  issue_id:      UUID
  commit_sha:    string
  repo_id:       UUID
  link_method:   enum(github_ref, pr_link, semantic_match)
  confidence:    float (0-1, 语义匹配方式有置信度)
  linked_at:     timestamp
```

#### 2.5.3 修复前后代码上下文提取

| 上下文粒度 | 提取内容 | 实现方式 | 用途 |
|---|---|---|---|
| **函数级** | bug 所在函数在修复前的完整代码 + 修复后的完整代码 | 对 diff 中每个变更文件，用 Tree-sitter 定位变更行所在的函数，提取修复前后两个版本的函数体 | Instruction 格式训练数据（"修复这个函数中的 bug"） |
| **文件级** | bug 所在文件修复前/后完整内容（限制 2000 行） | 直接从 git 提取修复前后两个版本的文件内容 | 大范围 bug 横跨多个函数、Agent 轨迹格式训练数据 |

**提取算法**：

```
输入: commit_sha, repo_path
输出: List[(buggy_code, fixed_code, context_level)]

1. 获取 commit 的 parent commit
2. 对比 parent 和 commit 之间的 diff
3. 对每个变更文件:
   a. 用 Tree-sitter 解析 parent 版本和 commit 版本
   b. 对 diff 中每个变更 hunk:
      - 找到变更行所在的函数（向上遍历 AST 找到 FunctionDef 节点）
      - 提取该函数在 parent 版本中的完整代码（buggy_code）
      - 提取该函数在 commit 版本中的完整代码（fixed_code）
   c. 同时提取文件级上下文（buggy_file, fixed_file）
4. 返回 (buggy_code, fixed_code, context_level=function) 和 (buggy_file, fixed_file, context_level=file)
```

#### 2.5.4 最终产出两种训练数据格式

**格式 A：Instruction 格式**

```json
{
  "id": "uuid",
  "type": "bug_fix",
  "instruction": "根据以下 Issue 描述修复代码中的 bug：\n\n{issue_title}\n{issue_body}",
  "context": {
    "repo": "owner/repo",
    "language": "python",
    "buggy_code": "def calculate(x):\n    return x / 0  # buggy code",
    "file_path": "src/utils.py",
    "context_level": "function"
  },
  "output": "def calculate(x):\n    if x == 0:\n        raise ValueError('x cannot be zero')\n    return 100 / x",
  "meta": {
    "issue_id": "issue_123",
    "commit_sha": "abc123",
    "link_method": "github_ref",
    "confidence": 1.0,
    "repo_score": 0.85,
    "pipeline_version": "v1.0.0"
  }
}
```

**格式 B：Agent 轨迹格式（SWE-agent 风格）**

```json
{
  "id": "uuid",
  "type": "agent_trajectory",
  "task": {
    "issue_title": "Division by zero in calculate function",
    "issue_body": "When x is 0, the calculate function throws a ZeroDivisionError...",
    "repo": "owner/repo",
    "base_commit": "parent_sha"
  },
  "trajectory": [
    {
      "step": 1,
      "action": "search",
      "query": "def calculate",
      "observation": "Found in src/utils.py line 42"
    },
    {
      "step": 2,
      "action": "read",
      "file": "src/utils.py",
      "lines": "40-50",
      "observation": "def calculate(x):\n    return x / 0"
    },
    {
      "step": 3,
      "action": "edit",
      "file": "src/utils.py",
      "old_code": "def calculate(x):\n    return x / 0",
      "new_code": "def calculate(x):\n    if x == 0:\n        raise ValueError('x cannot be zero')\n    return 100 / x"
    },
    {
      "step": 4,
      "action": "submit",
      "result": "fix applied"
    }
  ],
  "verification": {
    "tests_passed": true,
    "ci_status": "success",
    "verification_method": "test_run"
  },
  "meta": {
    "issue_id": "issue_123",
    "commit_sha": "abc123",
    "link_method": "github_ref",
    "repo_score": 0.85
  }
}
```

#### 2.5.5 Verification（验证）

对每个修复链数据，在进入训练集之前进行验证：

| 验证方式 | 做法 | 适用条件 | 可靠性 |
|---|---|---|---|
| **Test 验证** | 提取 repo 中与 bug 相关的测试用例，在修复前 commit 运行 → 应失败，在修复后 commit 运行 → 应通过 | repo 有相关测试 | ⭐⭐⭐⭐⭐ |
| **LLM 验证** | 强模型判断 solution 是否真的解决了 Issue 描述的 problem | 无测试或测试不覆盖 | ⭐⭐⭐ |
| **质量分兜底** | 依靠评测阶段的 quality_score 间接筛选 | 前两种方式都不可用 | ⭐⭐ |

**验证结果**：`verified` / `unverified` / `rejected`，只有 `verified` 的数据进入训练集。

#### 2.5.6 完整修复链数据流

```
1. 从 L3 精选 repo 中提取所有 Issue
   ↓
2. 用三种方式关联 Issue → Commit
   ↓
3. 对关联到的 Commit 做三层过滤（格式/语义/规模）
   ↓
4. 对有价值的 Commit，提取修复前后代码（函数级 + 文件级）
   ↓
5. 并行生成两种格式的训练数据：
   - Instruction 格式（直接用于 SFT）
   - Agent 轨迹格式（用于 SWE-agent 风格训练）
   ↓
6. Verification（Test 验证 + LLM 验证 + 质量分兜底）
   ↓
7. 仅 verified 的数据进入训练集
   ↓
8. 存入数据湖 gold 层，标记数据类型为 `synthetic/bugfix`
```

---

### 2.6 代码数据配比策略

#### 2.6.1 配比维度

代码数据在整体训练数据中的配比由训练团队动态调整，但数据团队需提供以下维度供配比：

| 维度 | 粒度 | 说明 |
|---|---|---|
| **语言** | Python / JS / TS / Java / Go / C++ / Rust / ... | 按语言比例分配 |
| **代码类型** | 函数 / 类 / 完整文件 / 测试 | 不同粒度用于不同训练目标 |
| **质量等级** | 基于 repo 重要性评分 + 评测阶段 quality_score | 高质量代码优先 |
| **来源权威性** | 官方组织 / 高星社区 / 一般 repo | 权威代码优先 |
| **License** | MIT / Apache / GPL / 其他 / 未知 | 合规性过滤 |
| **变更类型** | bugfix / feature / refactor / improvement | 按需配比（仅 L2+ 数据） |

#### 2.6.2 代码数据的初始配比建议

```
Phase 1 建议（仅供训练团队参考，最终由训练团队决定）：

语言分布:
  Python     25%
  JavaScript 18%
  TypeScript 12%
  Java       10%
  Go          8%
  C++         7%
  Rust        5%
  C#          4%
  Others     11%

代码粒度:
  函数级  60%  (SFT 指令微调的主力)
  文件级  30%  (上下文理解)
  类级    10%  (OOP 设计模式)

质量过滤:
  repo_score ≥ 0.3 的代码才进入训练集
  quality_score ≥ 40 的代码优先
```

---

## 3. 网页文档数据

### 3.1 采集策略

| 通道 | 做法 | 种子 URL 示例 | 适用场景 |
|---|---|---|---|
| **定向深挖** | 预设种子 URL，按链接深度 ≤ 3 遍历 | Python 官方文档、MDN、Kubernetes 文档、各语言官方教程 | 高质量技术文档 |
| **通用抓取** | CommonCrawl 或自建爬虫广撒网 | 按 TLD 分布采样 | 通用文本语料 |

**隔离存储**：定向深挖和通用抓取写入不同的 Iceberg 表，避免种子 URL 泄露到通用抓取的"噪声"中。

### 3.2 处理链路

详见 [03-processing-pipeline.md §10.2](./03-processing-pipeline.md#102-网页文档数据)

---

## 4. API 文档数据

### 4.1 采集源清单

| 源 | 格式 | 采集方式 |
|---|---|---|
| **Swagger/OpenAPI** | JSON/YAML | 直接解析 spec 文件 |
| **ReadTheDocs** | HTML | 爬虫 + HTML 解析 |
| **MDN Web Docs** | HTML | 爬虫 + 结构化提取 |
| **各语言标准库文档** | HTML | 爬虫 + 结构化提取 |

### 4.2 API-代码对齐

详见 [README.md §6](../README.md#6-api-文档-代码对齐)

---

## 5. GitHub Issue/PR/Discussion

### 5.1 采集方式

- GitHub Archive (BigQuery) 批量导出
- 补充 GitHub API 获取完整 body 文本
- 过滤规则见 [03-processing-pipeline.md §3.2](./03-processing-pipeline.md#32-阶段二粗过滤规则引擎)

### 5.2 与代码数据的关联

- Issue 引用 PR → PR 关联 commit → 形成完整链路
- 用于构造 Issue-Commit-Code 完整修复链（见 §2.5）

---

## 6. 数据源优先级矩阵

| 优先级 | 数据源 | 理由 | Phase 1 投入 |
|---|---|---|---|
| **P0** | GitHub 静态快照（L1） | 代码语料是基座模型的核心能力来源 | 全力 |
| **P0** | 网页文档（定向深挖） | 高质量文档是知识理解的基础 | 全力 |
| **P1** | 网页文档（通用抓取） | 量大但质量方差大，先跑通链路 | 中等 |
| **P1** | GitHub Issue/PR/Discussion | 自然语言对话语料 | 中等 |
| **P2** | GitHub 演进式挖掘（L2） | 有价值 commit 过滤 + diff 提取 | 骨架 |
| **P2** | API 文档 | 量小但价值高，与代码对齐 | 骨架 |
| **P3** | GitHub 完整修复链（L3） | 工程量大，但产出质量最高 | 骨架（Phase 2 主力） |