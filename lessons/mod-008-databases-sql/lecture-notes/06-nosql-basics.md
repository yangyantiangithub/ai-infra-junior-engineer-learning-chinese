# Lecture 06: NoSQL 基础

## 目录
1. [引言](#introduction)
2. [什么是 NoSQL？](#what-is-nosql)
3. [NoSQL 分类](#nosql-categories)
4. [何时使用 NoSQL 替代 SQL](#when-to-use-nosql-instead-of-sql)
5. [MongoDB：文档数据库](#mongodb-document-database)
6. [Redis：键值存储与数据结构](#redis-key-value-and-data-structures)
7. [ML 的 Vector Database](#vector-databases-for-ml)
8. [一致性、可用性与 CAP 定理](#consistency-availability-and-the-cap-theorem)
9. [ML 系统中的混合持久化](#polyglot-persistence-in-ml-systems)
10. [运维关注点](#operational-concerns)
11. [总结与关键要点](#summary-and-key-takeaways)

## 引言

关系型数据库是许多问题的正确答案，但并非所有问题。NoSQL（"Not Only SQL"）数据库牺牲了关系模型的一些约束——固定 schema、ACID 事务、JOIN——以换取关系型数据库难以提供的特性：水平扩展、灵活文档、亚毫秒级键查找和专门的数据类型（如向量和时间序列）。

在 ML 基础设施团队中，你将经常遇到：
- **Redis** 作为特征存储缓存、会话存储和任务队列
- **MongoDB** 作为实验元数据、模型制品目录或用户事件日志的灵活存储
- **vector database**（Pinecone、Weaviate、Qdrant、pgvector）用于嵌入向量相似性搜索

本讲座将向你介绍你最可能第一天就遇到的两种 NoSQL 系统——MongoDB 和 Redis——并介绍 vector database 作为 ML 持久化的新兴第三极。这不是对它们的深入探讨；那是专门数据库培训的领域。目标是胜任的使用、准确的决策以及与专家交流的语言。

### 学习目标

在本讲座结束时，你将：
- 区分主要的 NoSQL 类别及其适用场景
- 使用 PyMongo 从 Python 读写 MongoDB 文档
- 使用 Redis 进行缓存、队列和速率限制
- 在工作层面理解 CAP 定理
- 为 ML 基础设施问题选择 NoSQL vs SQL 并给出明确理由
- 安全地运维 NoSQL 服务（认证、备份、监控）

### 前置条件
- 模块 008 讲座 01–05（SQL 基础、ORM、扩展）
- Python 基础
- 熟悉 JSON

### 预计时间
3 小时（包括动手练习）

---

## 什么是 NoSQL？

NoSQL 是不使用关系模型的数据库的松散总称。"Not Only SQL"这个扩展是后来附加的，以缓和原始含义，并强调 NoSQL 系统可以补充而非仅仅替代关系存储。

NoSQL 系统的共同特征：

1. **Schema 灵活**。你可以向文档添加新字段而无需执行 `ALTER TABLE`。
2. **水平可扩展**。从一开始就设计为跨多个节点分片。
3. **单一用途的数据模型**。每个 NoSQL 类别擅长解决一种形态的问题，而非充分解决所有问题。
4. **宽松的事务保证**。大多数 NoSQL 系统为了性能和可扩展性放弃了一些 ACID 属性（尤其是跨文档/分片）。
5. **文档、键或图访问模式**。通过 primary key 查找极快；临时 JOIN 不是强项。

NoSQL **并非** PostgreSQL 的神奇升级版。现代 PostgreSQL 能胜任处理 JSON 文档、全文搜索、时间序列工作负载，甚至向量（通过 `pgvector`）。如果你的工作负载规模小到中等，并且没有明确约束表明 PostgreSQL 无法满足，请保持关系型。

---

## NoSQL 分类

| 类别 | 示例 | 适用场景 |
|---|---|---|
| **键值** | Redis, DynamoDB, etcd, Memcached | 通过精确键进行亚毫秒查找；缓存；计数器；队列 |
| **文档** | MongoDB, Couchbase, Firestore | 嵌套、半结构化记录，schema 演化；按文档访问 |
| **列族** | Cassandra, HBase, ScyllaDB | 大规模高写入吞吐量；高基数时间序列 |
| **图** | Neo4j, ArangoDB, JanusGraph | 多跳关系查询；推荐图谱；欺诈团伙 |
| **时间序列** | InfluxDB, TimescaleDB, Prometheus | 基于时间查询的高吞吐量仅追加指标 |
| **向量** | Pinecone, Weaviate, Qdrant, Milvus, pgvector | 对学习嵌入向量进行近似最近邻搜索 |

你最常接触的是**键值**和**文档**类别；vector database 是 ML 领域的新星。

---

## 何时使用 NoSQL 替代 SQL

NoSQL 是正确选择，当以下一项或多项明确成立时：

1. **访问模式主要是按键点查找**。关系表也可以做到，但键值存储更便宜、更快。
2. **记录是嵌套的且形态多变**。一篇带有评论和标签的博客文章更适合放在一个文档中，而不是三个连接的表。ML 实验元数据是典型例子：每次运行记录不同的指标和参数集。
3. **你需要大规模亚毫秒延迟**。Redis 从 RAM 中为每个节点提供数百万操作/秒。
4. **工作负载是写入密集型且仅追加，具有可预测的分区键**。Cassandra 和 DynamoDB 在此表现出色。
5. **你需要水平扩展，且可以围绕最终一致性建模**。关系主库超过某点后无法线性扩展；分片的 NoSQL 存储可以。
6. **查询本质上是对向量的相似性搜索**。没有 `pgvector`（或类似扩展）的关系引擎无法高效处理此问题。

使用 SQL 的情况：

- 你需要跨多个实体的 **JOIN**
- 你需要跨行的**强事务保证**（财务账本、库存）
- 数据高度结构化且很少改变形态
- 团队规模小，一个无聊的关系数据库涵盖一切

---

## MongoDB：文档数据库

MongoDB 将 **BSON**（二进制编码的 JSON）文档存储在位于**数据库**中的**集合**中。集合中的文档不必共享 schema，但实际上应通过应用代码或 MongoDB 的可选 schema 验证来强制执行一个。

### 核心概念

| MongoDB | 关系类比 |
|---|---|
| Database | Database |
| Collection | Table |
| Document | Row |
| Field | Column |
| _id | Primary key |
| Index | Index |

每个文档由 `_id` 唯一标识，默认为 12 字节的 `ObjectId`。

### Python 快速入门：PyMongo

```python
from pymongo import MongoClient, ASCENDING
from datetime import datetime

client = MongoClient("mongodb://localhost:27017", appname="ml-platform")
db = client["mlops"]
runs = db["experiment_runs"]

# 插入一个文档
run_doc = {
    "experiment": "fraud-detector",
    "model_version": "v3",
    "started_at": datetime.utcnow(),
    "params": {"learning_rate": 0.01, "batch_size": 64},
    "metrics": {"auc": 0.91, "f1": 0.84},
    "tags": ["baseline", "team-payments"],
}
result = runs.insert_one(run_doc)
print(result.inserted_id)

# 查询
recent_baselines = list(
    runs.find(
        {"experiment": "fraud-detector", "tags": "baseline"},
        {"_id": 0, "model_version": 1, "metrics.auc": 1},
    ).sort("started_at", -1).limit(10)
)

# 更新
runs.update_one(
    {"_id": result.inserted_id},
    {"$set": {"metrics.recall": 0.78}},
)

# 为主查询模式创建索引
runs.create_index([("experiment", ASCENDING), ("started_at", -1)])
```

### 查询运算符

常见运算符：

- 等值、比较：`{"value": {"$gt": 0.9}}`、`{"$in": [...]}`、`{"$ne": ...}`
- 逻辑：`{"$and": [...]}`、`{"$or": [...]}`、`{"$not": ...}`
- 数组：`{"tags": {"$all": ["a", "b"]}}`、`{"$size": 3}`
- 存在性：`{"deprecated": {"$exists": False}}`
- 模式：`{"name": {"$regex": "^prod-"}}`

### 聚合管道

MongoDB 的聚合框架是 SQL `GROUP BY` 及更高阶功能的类比。它是对文档应用的一系列阶段。

```python
pipeline = [
    {"$match": {"experiment": "fraud-detector"}},
    {"$group": {
        "_id": "$model_version",
        "best_auc": {"$max": "$metrics.auc"},
        "runs":    {"$sum": 1},
    }},
    {"$sort": {"best_auc": -1}},
]

for row in runs.aggregate(pipeline):
    print(row)
```

### 索引——唯一重要的运维技能

未索引查询的默认行为是**集合扫描**，随集合大小线性扩展。始终：

1. 分析你的真实查询模式。
2. 创建与查询形态匹配的复合索引（等值字段优先，范围/排序字段在后——"ESR 规则"）。
3. 使用 `explain("executionStats")` 确认索引被使用。

```python
runs.create_index([
    ("experiment", ASCENDING),
    ("tags",       ASCENDING),
    ("started_at", -1),
])

print(runs.find({"experiment": "x"}).explain("executionStats")["executionStats"])
```

### Schema 验证

MongoDB 默认 schema 灵活，但你可以对集合强制执行 JSON Schema：

```javascript
db.runCommand({
  collMod: "experiment_runs",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["experiment", "model_version", "started_at"],
      properties: {
        experiment:    { bsonType: "string" },
        model_version: { bsonType: "string" },
        started_at:    { bsonType: "date" },
        metrics:       { bsonType: "object" },
      }
    }
  },
  validationLevel: "moderate"
})
```

实际上，通过应用中的 Pydantic 模型进行验证通常比 DB 端验证更灵活且更易演化。

### 事务

MongoDB 支持副本集和分片集群上的多文档事务。它们可以工作，但相比单文档原子操作有更高的开销。建议将数据建模使得逻辑事务映射到单个文档更新；仅在无法避免跨文档操作时才使用事务。

---

## Redis：键值存储与数据结构

Redis 是一个内存数据存储。它可以持久化到磁盘，但其优势在于"中小规模、热数据、延迟关键"领域。它还包括内置的数据结构——列表、集合、有序集合、哈希、流——这使它能够替代许多专门系统。

### Redis 在 ML 中的用途

1. **特征存储缓存**。在线推理按实体 ID 查找特征；Redis 在微秒内返回。
2. **结果缓存**。相同的预测请求（幂等、确定性模型）可以被缓存。
3. **任务队列**。Celery、RQ 和 Dramatiq 使用 Redis 作为批量推理和训练任务的消息代理。
4. **速率限制**。Redis 中的令牌桶计数器保护预测端点免受失控客户端的攻击。
5. **分布式锁**。跨副本协调"一次只有一个训练器"的语义。
6. **实时排行榜/计数器**。有序集合保存模型延迟排名或 A/B 分组计数器。

### Python 快速入门：redis-py

```python
import redis, json, time

r = redis.Redis(host="localhost", port=6379, db=0, decode_responses=True)

# 字符串
r.set("model:current", "v3.2.1")
print(r.get("model:current"))               # "v3.2.1"

# TTL：缓存特征向量 60 秒
r.set("features:user:42", json.dumps([0.1, 0.4, 0.7]), ex=60)

# 哈希——将相关字段存储在一起
r.hset("user:42", mapping={"age": 35, "tier": "gold", "country": "US"})
print(r.hgetall("user:42"))

# 列表——简单的 FIFO 队列
r.lpush("inference:queue", json.dumps({"req_id": "abc", "features": [1, 2, 3]}))
job = r.brpop("inference:queue", timeout=5)  # 阻塞弹出

# 集合——唯一成员
r.sadd("seen:request_ids", "abc", "def")
print(r.sismember("seen:request_ids", "abc"))

# 有序集合——排名
r.zadd("model:latency_p95", {"m1": 12.3, "m2": 9.1, "m3": 15.4})
print(r.zrange("model:latency_p95", 0, -1, withscores=True))

# 原子计数器——适用于速率限制
count = r.incr("rate:user:42")
if count == 1:
    r.expire("rate:user:42", 60)  # 60 秒后重置窗口
if count > 100:
    raise RuntimeError("rate limit exceeded")
```

### 值得了解的模式

**缓存旁路。**

```python
def get_features(user_id: str) -> list[float]:
    key = f"features:{user_id}"
    cached = r.get(key)
    if cached:
        return json.loads(cached)

    features = feature_store.fetch(user_id)        # 慢
    r.set(key, json.dumps(features), ex=300)        # 5 分钟 TTL
    return features
```

**管道**以批量操作并减少往返：

```python
pipe = r.pipeline()
for uid in user_ids:
    pipe.get(f"features:{uid}")
results = pipe.execute()
```

**发布/订阅**用于跨服务通知：

```python
# 生产者
r.publish("model.deployed", json.dumps({"version": "v3.2.1"}))

# 消费者
sub = r.pubsub()
sub.subscribe("model.deployed")
for msg in sub.listen():
    if msg["type"] == "message":
        handle(json.loads(msg["data"]))
```

**流**用于持久、可重放的事件日志（Redis 5+）——一个小规模的 Kafka 替代品。

### 运维现状

- **内存是约束条件**。仔细规划容量。满时使用 `maxmemory-policy allkeys-lru` 淘汰冷键。
- **持久化需要权衡**。RDB 快照便宜但可能丢失数秒的写入；AOF 以写入放大的代价提供持久性。
- **单线程命令执行**。长时间运行的命令（在大数据库上执行 `KEYS *`、大型 `LRANGE`）会阻塞整个服务器。使用 `SCAN` 并避免无限扫描。
- **集群模式**按哈希槽跨节点分片键。当需要对同一分片执行原子多键操作时，使用哈希标签（`{tag}`）。

---

## ML 的 Vector Database

Vector database 是随着基于嵌入向量的 ML（推荐、语义搜索、RAG）兴起而出现的 NoSQL 类别。

### 问题

你有 N 百万个条目嵌入向量（稠密浮点向量，维度 D，通常为 384、768 或 1536）。查询时，给定一个新的嵌入向量，你需要按余弦相似度或 L2 距离找到 top-k 最相似的向量。朴素实现是 O(N·D) 每查询，不可扩展。

### 工具

近似最近邻（ANN）算法——HNSW、IVF、ScaNN——以少量召回率为代价换取更好的查询延迟。Vector database 将这些算法与持久化、复制、元数据过滤和客户端 API 打包在一起。

| 选项 | 何时考虑 |
|---|---|
| **pgvector** | 你已经在运行 PostgreSQL，且数据集小到中等（< ~1000 万向量） |
| **Qdrant**, **Weaviate**, **Milvus** | 自托管，更大规模，更高级的过滤和量化 |
| **Pinecone** | 全托管；按使用付费；API 非常简单 |
| **Vespa**, **OpenSearch k-NN** | 如果你已为全文搜索运行它们 |

### 最小示例：Qdrant

```python
from qdrant_client import QdrantClient
from qdrant_client.http.models import Distance, VectorParams, PointStruct

client = QdrantClient(host="localhost", port=6333)

client.recreate_collection(
    collection_name="products",
    vectors_config=VectorParams(size=384, distance=Distance.COSINE),
)

client.upsert(
    collection_name="products",
    points=[
        PointStruct(id=1, vector=[...384 floats...], payload={"category": "shoes"}),
        PointStruct(id=2, vector=[...], payload={"category": "shirts"}),
    ],
)

hits = client.search(
    collection_name="products",
    query_vector=[...384 floats...],
    limit=5,
    query_filter={"must": [{"key": "category", "match": {"value": "shoes"}}]},
)
```

Vector database 将在后续模块（RAG 类应用、嵌入管道）中反复出现。将其视为一个入门介绍。

---

## 一致性、可用性与 CAP 定理

CAP 定理指出，在**网络分区**的情况下，分布式系统必须在**一致性**（每次读取都能看到最新写入）和**可用性**（每个请求都能收到响应）之间做出选择。许多 NoSQL 系统倾向于可用性并接受最终一致性。

在实践中：

- **PostgreSQL 主库** → 强一致性；如果主库分区则失去可用性。
- **MongoDB** → 可调节：默认从主库读取是一致的；从副本读取可能有过时数据。
- **Cassandra / DynamoDB** → 每次查询可调节（`ONE`、`QUORUM`、`ALL`）；通常为了吞吐量而使用最终一致性运行。
- **Redis** → 单节点强一致；集群复制在副本间可能最终一致。

正确的问题很少是"这个数据库一致吗？"而是**"我的工作负载需要什么样的一致性？"** 模型服务延迟缓存 → 最终一致性可接受。银行余额 → 强一致性。特征查找 → "尽可能新鲜。"

---

## ML 系统中的混合持久化

成熟的 ML 平台通常一起使用多个数据库：

```
              ┌────────────────────────────────────────────┐
              │                                            │
   ┌──────────┴──────────┐                  ┌──────────────┴──────────────┐
   │  PostgreSQL          │                  │  Redis                       │
   │  用户 / 账单         │                  │  在线特征、                  │
   │  数据源真相           │                  │  速率限制、缓存              │
   └──────────┬──────────┘                  └──────────────┬──────────────┘
              │                                            │
   ┌──────────┴──────────┐                  ┌──────────────┴──────────────┐
   │  MongoDB             │                  │  Vector DB (Qdrant/pgvector)│
   │  实验元数据          │                  │  嵌入向量相似性             │
   │  模型制品目录         │                  │  用于 RAG / 推荐系统        │
   └──────────────────────┘                  └──────────────────────────────┘
              │
   ┌──────────┴──────────┐
   │  S3 / GCS / Blob     │
   │  模型文件、原始       │
   │  数据、大型日志      │
   └──────────────────────┘
```

这就是**混合持久化**：根据访问模式选择正确的存储。代价是操作复杂性和跨存储的数据一致性协调。好处是单个数据库无法良好服务的各种工作负载。

---

## 运维关注点

无论你采用哪个 NoSQL 系统，运维检查清单大体相同：

1. **身份验证和授权**。默认部署通常关闭认证。在暴露服务前务必启用。使用传输层 TLS。
2. **网络暴露**。在没有防火墙和认证的情况下，切勿将 Redis 或 MongoDB 绑定到公共接口。它们历来是互联网范围数据泄露的源头。
3. **备份**。Redis 使用 RDB/AOF，MongoDB 使用 mongodump 或文件系统快照，在服务提供的情况下进行时间点恢复。每季度测试恢复。
4. **复制**。运行副本集/集群以实现高可用。通过在测试环境中杀死主库来验证故障转移。
5. **监控**。内存使用、操作/秒、延迟 p50/p99、复制延迟、缓存命中率。
6. **资源限制**。应用 Kubernetes 资源请求和限制。内存密集型服务超出 RAM 会被 OOM 杀死并丢失内存状态。
7. **容量规划**。Redis 的键计数 × 平均大小；MongoDB 的文档计数 × 平均大小 + 索引；vector DB 的向量数 × 维度 × 字节/元素。

---

## 总结与关键要点

- **NoSQL 是一个家族，而非单个数据库。** 选择适合工作负载的类别——键值、文档、列族、图、时间序列、向量。
- **不要默认选择 NoSQL。** 现代 PostgreSQL 在小到中等规模下能很好地处理 JSON、时间序列甚至向量。
- **当你需要缓存、队列、速率限制器或微秒级键查找时，选择 Redis。**
- **当记录深度嵌套且 schema 因记录而异，且你的查询主要是按文档访问时，选择 MongoDB。**
- **当查询是"给我 top-k 最相似的嵌入向量"时，选择 vector database。**
- **CAP 定理**是思考一致性与可用性的工作框架——根据你的工作负载进行调整，而非数据库默认值。
- **混合持久化**是 ML 平台的常态：SQL 用于数据源真相，Redis 用于热数据，MongoDB 用于灵活元数据，对象存储用于制品，vector DB 用于相似性搜索。
- **谨慎运维：** 认证、TLS、备份、复制、监控、容量。像对待关系数据库一样严格对待 NoSQL 存储。

本模块的练习包括一个动手操作的 Redis 缓存练习和一个 MongoDB 实验跟踪器小项目，你将在此处介绍的模式中应用所学知识。
