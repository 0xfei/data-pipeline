# 技术选型总览——详细设计

> 主文档：[README.md](../README.md) 第 9 章

---

## 目录

- [1. 选型总览表](#1-选型总览表)
- [2. 基础设施层](#2-基础设施层)
  - [2.1 容器编排（K8s）](#21-容器编排k8s)
  - [2.2 计算资源规划](#22-计算资源规划)
- [3. 计算引擎层](#3-计算引擎层)
  - [3.1 Apache Spark](#31-apache-spark)
  - [3.2 Ray](#32-ray)
  - [3.3 Spark ↔ Ray 数据流转](#33-spark--ray-数据流转)
- [4. 存储层](#4-存储层)
  - [4.1 对象存储（S3）](#41-对象存储s3)
  - [4.2 表格式（Iceberg）](#42-表格式iceberg)
  - [4.3 列式格式（Parquet + Lance）](#43-列式格式parquet--lance)
  - [4.4 Catalog（REST Catalog）](#44-catalogrest-catalog)
- [5. 模型层](#5-模型层)
  - [5.1 Embedding 模型](#51-embedding-模型)
  - [5.2 质量评测模型](#52-质量评测模型)
  - [5.3 分类模型](#53-分类模型)
  - [5.4 代码解析模型](#54-代码解析模型)
  - [5.5 LLM Judge / 合成数据模型](#55-llm-judge--合成数据模型)
  - [5.6 辅助模型](#56-辅助模型)
- [6. 数据处理工具链](#6-数据处理工具链)
  - [6.1 去重算法](#61-去重算法)
  - [6.2 向量索引](#62-向量索引)
  - [6.3 爬虫与采集](#63-爬虫与采集)
  - [6.4 PII 脱敏](#64-pii-脱敏)
- [7. Pipeline 编排](#7-pipeline-编排)
  - [7.1 编排框架对比](#71-编排框架对比)
- [8. 监控与可观测性](#8-监控与可观测性)
- [9. 实验管理对接](#9-实验管理对接)
- [10. 版本兼容性矩阵](#10-版本兼容性矩阵)
- [11. 资源规划与成本估算](#11-资源规划与成本估算)
- [12. Embedding 推理调度方案对比](#12-embedding-推理调度方案对比)

---

## 1. 选型总览表

| 层级 | 领域 | 选型 | 版本 | 阶段 |
|---|---|---|---|---|
| 基础设施 | 容器编排 | **Kubernetes** | 1.28+ | Phase 1 |
| 基础设施 | 对象存储 | **S3 兼容接口**（云厂商） | — | Phase 1 |
| 计算引擎 | 批处理 ETL | **Apache Spark** | 3.5.x | Phase 1 |
| 计算引擎 | GPU 推理 | **Ray** | 2.10+ | Phase 1 |
| 计算引擎 | 部署模式 | **K8s**（Spark Operator + KubeRay） | — | Phase 1 |
| 存储 | 表格式 | **Apache Iceberg** | 1.5+ | Phase 1 |
| 存储 | 文本/结构化 | **Parquet** | — | Phase 1 |
| 存储 | 向量/索引 | **Lance** | 0.14+ | Phase 1 |
| 存储 | Catalog | **REST Catalog** | 1.5+ | Phase 1 |
| 模型 | Embedding（Phase 1） | **multilingual-e5-small** | 384d | Phase 1 |
| 模型 | Embedding（Phase 2） | **multilingual-e5-base** | 768d | Phase 2 |
| 模型 | 代码 Embedding | **StarCoder-Embed** | 768d | Phase 2 |
| 模型 | 文本质量（L2） | **ModernBERT-base** | 149M | Phase 1 |
| 模型 | 分类（L1） | **DistilBERT** | 66M | Phase 1 |
| 模型 | Perplexity | **KenLM** | 5-gram | Phase 1 |
| 模型 | PII 实体识别 | **spaCy** | 3.7+ | Phase 1 |
| 模型 | 语言检测 | **fastText** / **langdetect** | — | Phase 1 |
| 模型 | LLM Judge | **内部模型**（70B+） | — | Phase 1 |
| 模型 | 合成数据生成 | **内部模型 + 开源模型自部署** | — | Phase 2 |
| 工具 | 多语言 AST | **Tree-sitter** | latest | Phase 1 |
| 工具 | 去重（精确） | **SHA256** | — | Phase 1 |
| 工具 | 去重（近重复） | **MinHash + LSH** | — | Phase 1 |
| 工具 | 向量索引 | **IVF-PQ**（Phase 1）→ **DiskANN**（Phase 2） | — | Phase 1 |
| 工具 | 爬虫 | **Scrapy** / **Playwright** | — | Phase 1 |
| 编排 | Pipeline 调度 | **Dagster**（首选） / **DolphinScheduler**（备选） | — | Phase 1 |
| 监控 | 指标采集 | **Prometheus + Grafana** | — | Phase 1 |
| 监控 | 日志 | **ELK** / **Loki** | — | Phase 1 |
| 实验 | 实验管理 | **W&B / MLflow** | — | Phase 1 |

---

## 2. 基础设施层

### 2.1 容器编排（K8s）

**选型**：Kubernetes 1.28+

**理由**：
- Spark 和 Ray 共享 K8s 集群，统一资源调度
- 弹性伸缩（HPA / Cluster Autoscaler），应对每天 TB 级增量的计算波峰
- 与 Dagster / DolphinScheduler 的 K8s 部署模式兼容

**节点池规划**：

| 节点池 | 用途 | 实例类型 | 资源 | 扩缩策略 |
|---|---|---|---|---|
| **CPU-Spark** | Spark Executor（ETL、去重、评测 L1） | 高 CPU 实例（如 c6i.8xlarge） | 32 vCPU / 64 GB | Spot 优先，On-Demand 兜底 |
| **GPU-Ray** | Ray Embedding 推理 + 评测 L2 | 单 GPU 实例（如 p4d.xlarge / A10G） | 8 vCPU / 32 GB / 1×A100 | On-Demand，预留实例 |
| **CPU-General** | Dagster/DS 调度器、Web Server、REST Catalog | 通用实例 | 4 vCPU / 16 GB | On-Demand |

**关键隔离策略**：
- Spark 只用 CPU 节点池，Ray 独占 GPU 节点池
- 通过 K8s `nodeSelector` 和 `taints/tolerations` 物理隔离
- 避免 CPU 任务抢占 GPU 节点导致昂贵 GPU 空转

### 2.2 计算资源规划

| 阶段 | 引擎 | 节点池 | 日常资源 | 峰值资源 |
|---|---|---|---|---|
| 粗过滤（规则引擎） | Spark | CPU-Spark | 50 executor × 4 vCPU | 200 executor × 4 vCPU |
| 粗加工（分类/标签） | Spark | CPU-Spark | 30 executor × 4 vCPU | 100 executor × 4 vCPU |
| 细加工（Embedding） | Ray | GPU-Ray | 8 GPU worker | 32 GPU worker |
| 细过滤（MinHash+LSH） | Spark | CPU-Spark | 20 executor × 4 vCPU | 80 executor × 4 vCPU |
| 评测 L1（Perplexity+分类） | Spark | CPU-Spark | 30 executor × 4 vCPU | 80 executor × 4 vCPU |
| 评测 L2（文本质量） | Ray | GPU-Ray | 4 GPU worker | 16 GPU worker |
| 格式转换 | Spark | CPU-Spark | 10 executor × 4 vCPU | 30 executor × 4 vCPU |
| Compaction | Spark | CPU-Spark | 20 executor × 4 vCPU | 50 executor × 4 vCPU |

---

## 3. 计算引擎层

### 3.1 Apache Spark

**选型**：Spark 3.5.x，K8s 部署（Spark Operator）

**关键配置**：

| 配置项 | 推荐值 | 说明 |
|---|---|---|
| `spark.sql.adaptive.enabled` | true | 自适应查询执行，自动优化 shuffle partition |
| `spark.sql.adaptive.coalescePartitions.enabled` | true | 自动合并小分区 |
| `spark.sql.adaptive.skewJoin.enabled` | true | 自动处理数据倾斜 |
| `spark.sql.extensions` | `org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions` | Iceberg SQL 扩展 |
| `spark.sql.catalog.{name}` | `org.apache.iceberg.spark.SparkCatalog` | Iceberg Catalog 配置 |
| `spark.dynamicAllocation.enabled` | true | 动态资源分配 |
| `spark.dynamicAllocation.maxExecutors` | 200 | 最大 executor 数 |
| `spark.sql.files.maxPartitionBytes` | 268435456 (256MB) | 每分区最大字节数 |

**与 Iceberg 集成**：

```scala
// SparkSession 配置
val spark = SparkSession.builder()
  .config("spark.sql.extensions", 
    "org.apache.iceberg.spark.extensions.IcebergSparkSessionExtensions")
  .config("spark.sql.catalog.rest", 
    "org.apache.iceberg.spark.SparkCatalog")
  .config("spark.sql.catalog.rest.catalog-impl", 
    "org.apache.iceberg.rest.RESTCatalog")
  .config("spark.sql.catalog.rest.uri", 
    "http://iceberg-rest-catalog:8181")
  .config("spark.sql.catalog.rest.warehouse", 
    "s3://bucket/")
  .getOrCreate()
```

### 3.2 Ray

**选型**：Ray 2.10+，KubeRay 部署

**关键配置**：

| 配置项 | 推荐值 | 说明 |
|---|---|---|
| Ray Cluster | 1 Head + N GPU Workers | Head 节点负责调度，Worker 执行推理 |
| GPU Worker 规格 | 8 vCPU / 32 GB RAM / 1 GPU | 每 GPU 一个 Worker |
| `batch_size` | 256-512 | 批处理推理大小 |
| `num_gpus` per task | 0.5 | 允许一个 GPU 同时跑 2 个 task（如果模型够小） |
| `max_restarts` | 3 | worker 失败自动重启 |

**Ray Data 批处理推理**：

```python
import ray
from ray.data import read_iceberg

# 读取 Iceberg 表
ds = read_iceberg(
    table_identifier="silver_all_sources",
    catalog_kwargs={"name": "rest", "uri": "http://..."}
)

# 批处理 embedding 生成
class EmbeddingGenerator:
    def __init__(self):
        from sentence_transformers import SentenceTransformer
        self.model = SentenceTransformer("intfloat/multilingual-e5-small")
    
    def __call__(self, batch):
        texts = [f"passage: {t}" for t in batch["content"]]
        embeddings = self.model.encode(texts, batch_size=256, show_progress_bar=False)
        return {"content_emb": embeddings.tolist()}

ds_with_emb = ds.map_batches(
    EmbeddingGenerator,
    batch_size=512,
    num_gpus=0.5,
    concurrency=16
)
```

### 3.3 Spark ↔ Ray 数据流转

**数据格式**：Iceberg + Parquet 作为中间格式。

```
Spark (ETL) 写入Iceberg表 → Ray 从Iceberg表读取 → Ray处理 → Ray写回Iceberg表
                               ↓
                          Lance 独立存储 embedding
```

**性能关键**：Spark 和 Ray 通过 Iceberg 表做数据交换，避免自定义序列化协议。Parquet 格式是两者都原生支持的。

---

## 4. 存储层

### 4.1 对象存储（S3）

**选型**：云厂商 S3 兼容接口

**关键配置**：

| 配置项 | 值 |
|---|---|
| 端点 | `https://s3.{cloud-provider}.com` |
| 认证 | IAM Role（EC2/K8s Service Account） |
| 加密 | SSE-S3（默认） |
| 生命周期 | 按表分层（详见 [02-data-lake.md §7](./02-data-lake.md#7-s3-存储生命周期)） |

### 4.2 表格式（Iceberg）

**选型**：Apache Iceberg 1.5+

**关键特性**：

| 特性 | 说明 |
|---|---|
| Snapshot 隔离 | 每次写入生成新 snapshot，读不阻塞写 |
| Time Travel | 可按 snapshot ID 或时间戳回溯历史数据 |
| Partition Evolution | 在线变更分区策略，不重写数据 |
| Schema Evolution | 在线添加/删除/重命名列 |
| Hidden Partitioning | 分区对用户透明，查询无需指定分区键 |
| 行级删除 | `delete` 文件支持，去重不物理删除 |
| 文件格式 | 底层用 Parquet（文本） |

### 4.3 列式格式（Parquet + Lance）

| 格式 | 用途 | 推荐配置 |
|---|---|---|
| **Parquet** | Iceberg 底层文本/结构化数据 | 压缩：ZSTD（level 3），文件大小 256-512MB |
| **Lance** | Embedding 向量 + ANN 索引 | 压缩：LZ4，文件大小 512MB-1GB，索引类型 IVF-PQ（Phase 1） |

### 4.4 Catalog（REST Catalog）

**选型**：Iceberg REST Catalog 1.5+

**部署**：

```
K8s Deployment:
  - replicas: 2
  - image: apache/iceberg-rest-fixture (或自建)
  - backend: PostgreSQL 15
  - health check: GET /v1/config
  - service: ClusterIP (内部) + LoadBalancer (外部工具)
```

**关键端点**：

| 端点 | 用途 |
|---|---|
| `GET /v1/{prefix}/namespaces` | 列出命名空间 |
| `POST /v1/{prefix}/namespaces/{ns}/tables` | 创建表 |
| `GET /v1/{prefix}/namespaces/{ns}/tables/{table}` | 加载表元数据 |
| `POST /v1/{prefix}/namespaces/{ns}/tables/{table}/metrics` | 提交表变更 |

---

## 5. 模型层

### 5.1 Embedding 模型

| 阶段 | 模型 | 参数量 | 维度 | 吞吐 (A100) | 用途 |
|---|---|---|---|---|---|
| **Phase 1** | `intfloat/multilingual-e5-small` | 118M | 384d | ~6000 docs/s | 全量数据快速建索引 |
| **Phase 2** | `intfloat/multilingual-e5-base` | 278M | 768d | ~2500 docs/s | 高价值数据精编码 |
| **Phase 2** | `bigcode/starcoder-embed` | 125M | 768d | ~1500 docs/s | 代码数据专用 |

**模型对比**：

| 模型 | 维度 | 最大 Token | 吞吐 (A100) | 语言 | 推荐场景 |
|---|---|---|---|---|---|
| multilingual-e5-small | 384 | 512 | ~6000 | 100+ | Phase 1 全量 |
| multilingual-e5-base | 768 | 512 | ~2500 | 100+ | Phase 2 升级 |
| BGE-M3 | 1024 | 8192 | ~1500 | 100+ | 长文本场景 |
| all-MiniLM-L6-v2 | 384 | 256 | ~8000 | 英 | 仅英文兜底 |
| StarCoder-Embed | 768 | 8192 | ~1500 | 代码 | 代码专用 |
| BGE-small-zh | 512 | 512 | ~6000 | 中/英 | 中文优化 |
| BGE-small-en | 384 | 512 | ~7000 | 英 | 英文优化 |

**e5 模型的标准使用方式**：

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer("intfloat/multilingual-e5-small")

# 重要：e5 模型需要 prefix
texts = ["passage: " + t for t in raw_texts]
embeddings = model.encode(texts, batch_size=256, normalize_embeddings=True)
```

### 5.2 质量评测模型

#### L1：Perplexity（KenLM）

**选型**：KenLM 5-gram 模型

**训练**：
- 在高质量中文/英文/代码语料上分别训练
- 模型大小：每个语言 ~500MB（5-gram ARPA 格式）
- 推理方式：Spark UDF，纯 CPU，无需 GPU

#### L2：文本质量模型（ModernBERT-base）

**选型**：`answerdotai/ModernBERT-base`（149M 参数）

| 维度 | 说明 |
|---|---|
| 参数量 | 149M |
| 推理速度 | ~3500 docs/s (A100) |
| 微调方式 | 在人工标注（高质量/低质量）二分类数据上 fine-tune |
| 输出 | 0-100 连续质量分 |
| 框架 | Ray（GPU 批处理推理） |

**选择 ModernBERT 而非 DeBERTa-v3 的理由**：
- ModernBERT 是 2024 年底发布的新模型，在速度和效果上做了显式 trade-off
- 推理速度比 DeBERTa-v3 快 ~75%，在每天 TB 级评测中差距显著
- 质量判别力与 DeBERTa-v3 相当（在多个文本质量 benchmark 上持平）

### 5.3 分类模型

**选型**：`distilbert-base-uncased`（66M 参数）

| 维度 | 说明 |
|---|---|
| 参数量 | 66M |
| 推理速度 | ~5000 docs/s (A100) |
| 分类类别 | code / natural_text / structured / noise / other |
| 微调方式 | 人工标注多分类数据 |
| 框架 | Spark（CPU，轻量级推理） |

**选择 DistilBERT 而非 XLM-RoBERTa 的理由**：
- Phase 1 用统一多语言模型，分类任务不需要多语言精度
- 66M 参数在 CPU 上也能高效推理，不占用 GPU 资源
- 分类类别简单（5 类），不需要 278M 的大模型

### 5.4 代码解析模型

**选型**：Tree-sitter（非 ML 模型，是增量解析器）

| 语言 | 成熟度 | 是否 Phase 1 支持 |
|---|---|---|
| Python | ⭐⭐⭐⭐⭐ | ✅ |
| JavaScript | ⭐⭐⭐⭐⭐ | ✅ |
| TypeScript | ⭐⭐⭐⭐⭐ | ✅ |
| Java | ⭐⭐⭐⭐⭐ | ✅ |
| Go | ⭐⭐⭐⭐⭐ | ✅ |
| C | ⭐⭐⭐⭐⭐ | ✅ |
| C++ | ⭐⭐⭐⭐ | ✅ |
| Rust | ⭐⭐⭐⭐⭐ | ✅ |
| Ruby | ⭐⭐⭐⭐ | ✅ |
| PHP | ⭐⭐⭐⭐ | ✅ |
| C# | ⭐⭐⭐⭐ | ✅ |
| Swift | ⭐⭐⭐ | ✅ |
| Kotlin | ⭐⭐⭐ | ✅ |
| Scala | ⭐⭐⭐ | ✅ |
| Lua | ⭐⭐⭐ | ✅ |
| 其余语言 | — | ❌（纯文本处理，按需补充） |

### 5.5 LLM Judge / 合成数据模型

| 用途 | 模型 | 部署方式 |
|---|---|---|
| **LLM Judge（抽样校准）** | 内部 70B+ 模型（如 DeepSeek-V3/Qwen 等） | 内部推理集群，API 调用 |
| **合成数据生成** | 内部模型 + 开源模型（Qwen/Mixtral/DeepSeek）自部署 | K8s GPU 推理服务 |
| **双向验证** | 多模型交叉验证 | 调用不同模型，对比输出 |

### 5.6 辅助模型

| 用途 | 模型 | 框架 |
|---|---|---|
| **语言检测** | fastText（`lid.176.bin`） | Spark UDF，CPU |
| **PII 实体识别** | spaCy 3.7+（`en_core_web_lg` + `zh_core_web_lg`） | Spark UDF，CPU |
| **HTML 解析** | selectolax / BeautifulSoup4 | Spark UDF，CPU |

---

## 6. 数据处理工具链

### 6.1 去重算法

| 算法 | 用途 | 参数 | 框架 |
|---|---|---|---|
| **SHA256** | 精确哈希去重 | — | Spark UDF |
| **MinHash + LSH** | 近重复文档去重 | 签名 128 / band 16 / Jaccard 0.8+ | Spark（datasketch 库） |
| **Embedding 语义去重** | 语义重复去重 | 余弦相似度 ≥ 0.95 | Lance ANN 索引 |

**MinHash 实现**（Spark）：

```python
from datasketch import MinHash, MinHashLSH

# 文档级 MinHash
def compute_minhash(text, num_perm=128, ngram_size=5):
    m = MinHash(num_perm=num_perm)
    for i in range(len(text) - ngram_size + 1):
        m.update(text[i:i+ngram_size].encode('utf-8'))
    return m

# LSH 索引
lsh = MinHashLSH(threshold=0.8, num_perm=128)
for doc_id, minhash in minhashes:
    lsh.insert(doc_id, minhash)
    
# 查询近重复
result = lsh.query(minhash)
```

### 6.2 向量索引

| 索引类型 | 适用阶段 | 构建速度 | 检索精度 | 内存占用 | 推荐参数 |
|---|---|---|---|---|---|
| **IVF-PQ** | Phase 1 | 快 | ~90% recall@10 | 低 | nlist=1024, M=8, nbits=8 |
| **DiskANN** | Phase 2 | 慢 | ~95% recall@10 | 中 | L=50, R=32, C=500 |

**Lance 索引创建**：

```python
import lance

dataset = lance.dataset("s3://bucket/silver_emb/github_code_chinese/")

# IVF-PQ 索引
dataset.create_index(
    column="content_emb",
    index_type="IVF_PQ",
    metric_type="cosine",
    num_partitions=1024,
    num_sub_vectors=48,  # 384d / 8 = 48 sub-vectors
    num_bits=8           # 8 bits per sub-vector
)

# DiskANN 索引 (Phase 2)
dataset.create_index(
    column="content_emb",
    index_type="DISKANN",
    metric_type="cosine",
    num_edges=32,
    search_list_size=50
)
```

### 6.3 爬虫与采集

| 工具 | 用途 | 适用场景 |
|---|---|---|
| **Scrapy** | 静态 HTML 页面抓取 | 通用网页抓取 |
| **Playwright** | 动态渲染页面（JS 渲染） | 文档站、SPA 页面 |
| **GitHub Archive（BigQuery）** | 离线批量分析 GitHub 元数据 | Repo 重要性评估、Issue/PR 分析 |
| **GitHub API** | 在线获取 repo 内容、文件列表 | 下载代码文件、README |

### 6.4 PII 脱敏

| 检测方式 | PII 类型 | 工具 |
|---|---|---|
| 正则 | 邮箱、手机号、IP、API Key、私钥、身份证号 | Python `re` |
| NER 模型 | 姓名、地址、组织名 | spaCy |

---

## 7. Pipeline 编排

### 7.1 编排框架对比

| 维度 | **Dagster** | **DolphinScheduler** |
|---|---|---|
| **编程模型** | Asset-based（数据资产导向） | Task-based DAG，可视化为主 |
| **Iceberg 集成** | **原生支持**：直接追踪 snapshot，asset lineage 与表版本打通 | 需自建：手动调用 Iceberg API |
| **增量调度** | **强**：asset 粒度自动检测上游变更，按需触发下游 | 弱：主要靠时间调度 + 手动触发 |
| **数据血缘** | **自动生成** asset 级 lineage 可视化 | 需自建血缘追踪 |
| **Spark 集成** | Spark Op 原生支持 | Task Plugin 调用 |
| **Ray 集成** | Ray Op 原生支持 | 需自建 Plugin |
| **运维成本** | 中（K8s + PostgreSQL） | **低**（Java 生态，国内团队熟悉） |
| **监控告警** | Prometheus 内建集成 | 内建告警 |
| **社区与生态** | 增长快，英文文档为主 | 中国开源社区活跃，中文文档完善 |
| **学习曲线** | 中（Asset 概念需适应） | 低（可视化 DAG 拖拽） |

**推荐**：Dagster（Iceberg 原生 + Asset 模型，与数据湖架构天然匹配）。如内部已有 DolphinScheduler 且团队熟练，可作为备选，但需：
- 自建 Iceberg 集成（写自定义 Task Plugin）
- 自建 Asset 粒度管理和血缘追踪
- 额外开发工作量约 2-3 人月

---

## 8. 监控与可观测性

| 层面 | 工具 | 用途 |
|---|---|---|
| **指标采集** | Prometheus + Grafana | Pipeline 指标、S3 指标、K8s 资源 |
| **日志** | ELK（Elasticsearch + Logstash + Kibana）或 Loki | 应用日志聚合分析 |
| **告警** | Alertmanager | 管道级告警（job 失败、延迟、资源异常） |
| **追踪** | OpenTelemetry（可选） | 分布式追踪（Spark → Ray 调用链） |
| **S3 监控** | S3 Access Logs + CloudWatch | 存储用量、请求延迟、错误率 |

---

## 9. 实验管理对接

**选型**：W&B / MLflow（与训练团队现有平台一致）

**对接方式**：

```python
import wandb

# 在 gold 层数据集发布时记录
wandb.log({
    "dataset": "gold_pretrain_v1",
    "iceberg_snapshot_id": "7520426094023953954",
    "lance_dataset_version": "v12",
    "total_docs": 1_200_000_000,
    "total_tokens": 800_000_000_000,
    "source_distribution": {"github": 0.40, "web": 0.35, ...},
    "quality_score_avg": 72.5,
    "pipeline_version": "v1.2.3"
})
```

**回溯链路**：

```
模型 Checkpoint → W&B/MLflow run → gold dataset version
  → Iceberg snapshot ID → 各处理阶段 snapshot
  → 原始 URL + 处理参数
```

---

## 10. 版本兼容性矩阵

| 组件 | 版本 | 依赖关系 |
|---|---|---|
| K8s | 1.28+ | — |
| Spark | 3.5.x | Iceberg 1.5+, Parquet 1.13+ |
| Iceberg | 1.5+ | Spark 3.5, REST Catalog 1.5 |
| REST Catalog | 1.5+ | Iceberg 1.5 |
| Ray | 2.10+ | KubeRay 1.0+ |
| Lance | 0.14+ | PyArrow 14+ |
| Python | 3.10-3.12 | Spark 3.5, Ray 2.10 |
| Java | 17 | Spark 3.5 |
| Dagster | 1.6+ | Python 3.10+ |
| DolphinScheduler | 3.2+ | Java 8+ |
| sentence-transformers | 2.7+ | PyTorch 2.0+ |
| spaCy | 3.7+ | Python 3.9+ |
| PostgreSQL | 15+ | REST Catalog 后端 |

---

## 11. 资源规划与成本估算

### 11.1 GPU 资源（Embedding 推理）

**Phase 1（multilingual-e5-small, 384d）**：

| 参数 | 值 |
|---|---|
| 单卡 A100 吞吐 | ~6000 docs/s |
| 每天需处理 | ~500M docs（TB 级增量） |
| 所需 GPU 卡时 | 500M / 6000 / 3600 ≈ 23 GPU-hours/天 |
| 推荐 GPU 数 | 8 × A100（冗余+并行处理全量历史数据） |

**全量 100PB 初扫**（一次性）：

| 参数 | 值 |
|---|---|
| 总文档数估算 | ~50B docs（假设 100PB 中平均 2KB/doc） |
| 所需 GPU 卡时 | 50B / 6000 / 3600 ≈ 2,315 GPU-hours |
| 32 卡并行 | ~72 小时（3 天） |
| 成本估算 | 2,315 × $2.5/h ≈ $5,800 |

### 11.2 CPU 资源（Spark ETL）

| 参数 | 值 |
|---|---|
| 日常 Spark executor | 50-200 个 × 4 vCPU |
| 每天 ETL 时间 | 6-8 小时 |
| 月度 vCPU-hour | ~50K-100K |

---

## 12. Embedding 推理调度方案对比

| 维度 | **方案 A：每模型独立 Ray Job** | **方案 B：单一 Ray Job 动态切换模型** |
|---|---|---|
| **实现方式** | 每个模型类型（文本/代码/轻量）一个独立 Ray Job，各自处理对应簇 | 一个 Ray Job 内按簇路由，`map_batches` 动态加载对应模型 |
| **GPU 利用率** | 中（模型切换开销为 0，但不同模型的 GPU 负载不均，可能出现空闲） | 高（同构 GPU 资源统一调度，减少空闲时间） |
| **调度复杂度** | 低（每个 Job 独立，互不干扰） | 中（需要模型切换逻辑和 batch 合并策略） |
| **容错性** | 高（一个模型失败不影响其他） | 中（Job 级失败影响所有模型，需重试机制） |
| **模型加载开销** | 无（每个 Job 启动时加载一次） | 有（每次切换模型需重新加载到 GPU 内存，约 5-15 秒） |
| **代码复杂度** | 低（每个 Job 代码简单独立） | 中（需维护模型路由映射 + batch 合并） |
| **Phase 1 推荐** | ✅ **推荐**（简单可靠，适合快速落地） | 备选（资源利用率更高，Phase 2 优化时考虑） |
| **Phase 2 推荐** | 备选 | ✅ **推荐**（GPU 资源紧张时切换，节省 20-30% GPU 成本） |

**Phase 1 实施建议**：先用方案 A（每模型独立 Job），快速上线，GPU 利用率可接受。Phase 2 根据实际 GPU 利用率和成本压力，评估是否切换到方案 B。