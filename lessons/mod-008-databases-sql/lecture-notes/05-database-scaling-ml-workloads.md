# Lecture 05: ML 工作负载的数据库扩展与性能

## 目录
1. [引言](#introduction)
2. [数据库扩展基础](#database-scaling-fundamentals)
3. [只读副本与复制](#read-replicas-and-replication)
4. [数据库分片](#database-sharding)
5. [缓存策略](#caching-strategies)
6. [ML 的 NoSQL 数据库](#nosql-databases-for-ml)
7. [时序数据库](#time-series-databases)
8. [ML 的 Vector Database](#vector-databases-for-ml)
9. [数据库性能优化](#database-performance-optimization)
10. [总结与关键要点](#summary-and-key-takeaways)

## 引言

ML 应用生成海量数据：训练数据集、模型元数据、预测结果、指标、日志和嵌入向量。随着你的 ML 平台从每天数百次预测扩展到数百万次，数据库会成为瓶颈。

本讲座涵盖数据库扩展策略、性能优化以及针对 ML 工作负载的专业数据库。

### 学习目标

在本讲座结束时，你将：
- 理解垂直与水平扩展的权衡
- 实现只读副本以扩展读取密集型工作负载
- 设计用于水平扩展的分片策略
- 使用缓存减少数据库负载
- 为 ML 数据选择合适的 NoSQL 数据库
- 使用时序数据库处理指标
- 使用 vector database 进行相似性搜索
- 为 ML 工作负载优化数据库性能

### 前置条件
- 讲座 01-04（SQL 基础、查询优化、ORM）
- 理解 ML 工作流（训练、推理、指标）
- 基本缓存概念知识

**时长**: 90 分钟
**难度**: 中级到高级

---

## 1. 数据库扩展基础

### 垂直与水平扩展

```
垂直扩展（纵向扩展）：
┌─────────────────────────────┐
│  更大的数据库服务器         │
│  ├── 4 CPU  → 32 CPU       │
│  ├── 16GB RAM → 256GB RAM  │
│  └── 1TB SSD → 10TB SSD    │
└─────────────────────────────┘

优点：简单，无需代码更改
缺点：昂贵，硬件限制，单点故障

水平扩展（横向扩展）：
┌──────────┐ ┌──────────┐ ┌──────────┐
│ DB 节点 1│ │ DB 节点 2│ │ DB 节点 3│
│ 4 CPU    │ │ 4 CPU    │ │ 4 CPU    │
│ 16GB RAM │ │ 16GB RAM │ │ 16GB RAM │
└──────────┘ └──────────┘ └──────────┘

优点：经济高效，无限扩展，高可用
缺点：复杂，需要代码更改，数据一致性挑战
```

### ML 数据库工作负载特征

```
典型 ML 数据库访问模式：

训练阶段：
├── 读取：高（读取数据集、特征）
├── 写入：中（保存检查点、指标）
└── 查询模式：批量读取，顺序扫描

推理阶段：
├── 读取：高（加载用于预测的特征）
├── 写入：高（存储预测结果、日志）
└── 查询模式：点查询，高 QPS

指标/监控：
├── 读取：高（仪表盘、告警）
├── 写入：高（时序指标）
└── 查询模式：时间范围查询，聚合

解决方案：不同工作负载使用不同数据库！
```

---

## 2. 只读副本与复制

### 主-副本架构

```
┌──────────────────────────────────────┐
│        主库（Primary）               │
│     处理所有写入                     │
│  ├── INSERT predictions              │
│  ├── UPDATE model metadata           │
│  └── DELETE old experiments          │
└────────────┬─────────────────────────┘
             │ 复制
      ┌──────┴──────┬──────────┐
      │             │          │
┌─────▼─────┐ ┌────▼──────┐ ┌─▼──────────┐
│ 副本 1    │ │ 副本 2    │ │ 副本 3     │
│ 只读      │ │ 只读      │ │ 只读       │
│ 仪表盘    │ │ 分析      │ │ ML 训练    │
└───────────┘ └───────────┘ └────────────┘
```

### 设置只读副本（PostgreSQL）

**在主库上：**

```sql
-- 在 postgresql.conf 中启用复制
wal_level = replica
max_wal_senders = 3
wal_keep_size = 1GB

-- 创建复制用户
CREATE USER replicator REPLICATION LOGIN PASSWORD 'secure_password';
```

**在副本上：**

```bash
# 停止副本
sudo systemctl stop postgresql

# 删除旧数据
rm -rf /var/lib/postgresql/14/main

# 从主库复制数据
pg_basebackup -h master-host -D /var/lib/postgresql/14/main \
  -U replicator -v -P -W -R

# 启动副本
sudo systemctl start postgresql
```

**使用只读副本的应用代码：**

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
import random

# 主库用于写入
master_engine = create_engine('postgresql://user:pass@master-db:5432/mldb')
MasterSession = sessionmaker(bind=master_engine)

# 副本用于读取
replica_engines = [
    create_engine('postgresql://user:pass@replica1-db:5432/mldb'),
    create_engine('postgresql://user:pass@replica2-db:5432/mldb'),
    create_engine('postgresql://user:pass@replica3-db:5432/mldb')
]
ReplicaSession = sessionmaker(bind=random.choice(replica_engines))

def save_prediction(prediction_data):
    """写入主库"""
    session = MasterSession()
    try:
        prediction = Prediction(**prediction_data)
        session.add(prediction)
        session.commit()
    finally:
        session.close()

def get_model_metadata(model_id):
    """从副本读取"""
    session = ReplicaSession()
    try:
        return session.query(Model).filter_by(id=model_id).first()
    finally:
        session.close()
```

### 复制延迟

**问题**: 副本可能落后于主库（数秒到数分钟）

```python
# 检查复制延迟（PostgreSQL）
import psycopg2

conn = psycopg2.connect("dbname=mldb host=replica1")
cursor = conn.cursor()

cursor.execute("""
    SELECT
        pg_last_wal_receive_lsn() AS receive,
        pg_last_wal_replay_lsn() AS replay,
        pg_last_wal_receive_lsn() - pg_last_wal_replay_lsn() AS lag_bytes
""")

lag = cursor.fetchone()
print(f"Replication lag: {lag[2]} bytes")

# 如果延迟超过阈值，警告用户
if lag[2] > 1000000:  # 1MB 延迟
    print("WARNING: Replica is significantly behind master")
```

---

## 3. 数据库分片

### 什么是分片？

**分片** = 跨多个数据库的水平分区

```
单个数据库（1 亿行）：
┌────────────────────────────────┐
│  Predictions 表                │
│  ├── 用户 1 的预测             │
│  ├── 用户 2 的预测             │
│  ├── ...                       │
│  └── 用户 100 万预测           │
│  （查询慢，单台服务器）        │
└────────────────────────────────┘

分片（4 个分片，每个 2500 万行）：
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ 分片 0   │ │ 分片 1   │ │ 分片 2   │ │ 分片 3   │
│ 用户     │ │ 用户     │ │ 用户     │ │ 用户     │
│ 0-25 万  │ │ 25-50 万 │ │ 50-75 万 │ │ 75-100 万│
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

### 分片策略

#### 1. 基于 Hash 的分片

```python
import hashlib

NUM_SHARDS = 4

def get_shard(user_id):
    """基于用户 ID 的 hash 确定分片"""
    hash_value = int(hashlib.md5(str(user_id).encode()).hexdigest(), 16)
    shard_id = hash_value % NUM_SHARDS
    return shard_id

# 示例
user_id = 12345
shard = get_shard(user_id)
print(f"User {user_id} → Shard {shard}")
# User 12345 → Shard 2
```

**每个分片的数据库连接：**

```python
from sqlalchemy import create_engine

shard_engines = {
    0: create_engine('postgresql://user:pass@shard0-db:5432/mldb'),
    1: create_engine('postgresql://user:pass@shard1-db:5432/mldb'),
    2: create_engine('postgresql://user:pass@shard2-db:5432/mldb'),
    3: create_engine('postgresql://user:pass@shard3-db:5432/mldb')
}

def get_engine_for_user(user_id):
    shard_id = get_shard(user_id)
    return shard_engines[shard_id]

# 保存预测到正确的分片
def save_prediction(user_id, prediction_data):
    engine = get_engine_for_user(user_id)
    with engine.connect() as conn:
        conn.execute(
            "INSERT INTO predictions (user_id, result, timestamp) VALUES (%s, %s, %s)",
            (user_id, prediction_data['result'], prediction_data['timestamp'])
        )
```

#### 2. 基于范围的分片

```python
# 按 user_id 范围分片
def get_shard_by_range(user_id):
    if user_id < 250000:
        return 0
    elif user_id < 500000:
        return 1
    elif user_id < 750000:
        return 2
    else:
        return 3
```

**优点**: 容易向新分片添加新用户
**缺点**: 如果用户增长非线性则分布不均

#### 3. 地理分片

```python
# 按地理区域分片
SHARD_BY_REGION = {
    'us-east': 0,
    'us-west': 1,
    'eu': 2,
    'asia': 3
}

def get_shard_by_region(region):
    return SHARD_BY_REGION.get(region, 0)
```

**使用场景**: 数据驻留要求，低延迟

### 跨分片查询

**问题**: 跨多个分片的查询

```python
# 获取所有用户的预测（跨所有分片）
def get_all_predictions_last_hour():
    results = []

    # 查询每个分片
    for shard_id, engine in shard_engines.items():
        with engine.connect() as conn:
            shard_results = conn.execute("""
                SELECT * FROM predictions
                WHERE timestamp > NOW() - INTERVAL '1 hour'
            """).fetchall()
            results.extend(shard_results)

    return results

# 跨分片聚合
def get_total_predictions_count():
    total = 0
    for shard_id, engine in shard_engines.items():
        with engine.connect() as conn:
            count = conn.execute("SELECT COUNT(*) FROM predictions").scalar()
            total += count
    return total
```

**性能提示**: 使用异步查询进行并行化：

```python
import asyncio
import asyncpg

async def get_count_from_shard(shard_config):
    conn = await asyncpg.connect(**shard_config)
    count = await conn.fetchval("SELECT COUNT(*) FROM predictions")
    await conn.close()
    return count

async def get_total_count_parallel():
    tasks = [get_count_from_shard(cfg) for cfg in shard_configs]
    counts = await asyncio.gather(*tasks)
    return sum(counts)

# 比顺序执行快 4 倍
total = asyncio.run(get_total_count_parallel())
```

---

## 4. 缓存策略

### 为什么为 ML 使用缓存？

```
没有缓存：
每次预测请求 → 数据库查询 → 50ms 延迟

有缓存：
首次请求 → 数据库 → 缓存 → 50ms
后续请求 → 缓存 → 2ms（快 25 倍！）
```

### 用于 ML 缓存的 Redis

```python
import redis
import json
import hashlib

redis_client = redis.Redis(host='localhost', port=6379, decode_responses=True)

def get_user_features(user_id):
    """获取用户特征，带缓存"""
    cache_key = f"user_features:{user_id}"

    # 先尝试缓存
    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # 缓存未命中 - 查询数据库
    features = db.query(f"SELECT * FROM user_features WHERE user_id = {user_id}")

    # 存储到缓存（TTL：1 小时）
    redis_client.setex(cache_key, 3600, json.dumps(features))

    return features
```

### 缓存失效

```python
def update_user_features(user_id, new_features):
    """更新特征并使缓存失效"""
    # 更新数据库
    db.execute(
        "UPDATE user_features SET features = %s WHERE user_id = %s",
        (json.dumps(new_features), user_id)
    )

    # 使缓存失效
    cache_key = f"user_features:{user_id}"
    redis_client.delete(cache_key)
```

### 缓存模型元数据

```python
def get_model_config(model_id):
    """缓存模型配置"""
    cache_key = f"model_config:{model_id}"

    cached = redis_client.get(cache_key)
    if cached:
        return json.loads(cached)

    # 从 DB 加载
    config = db.query(f"SELECT config FROM models WHERE id = {model_id}")

    # 缓存 24 小时（模型不频繁变更）
    redis_client.setex(cache_key, 86400, json.dumps(config))

    return config
```

---

## 5. ML 的 NoSQL 数据库

### 何时使用 NoSQL

```
使用 SQL 的情况：
✓ 结构化数据（表格形式）
✓ 复杂 JOIN
✓ ACID 事务
✓ 业务关键数据

使用 NoSQL 的情况：
✓ 非结构化/半结构化数据
✓ 大规模（数十亿条记录）
✓ 高写入吞吐量
✓ 灵活的 schema
```

### MongoDB 用于 ML 元数据

```python
from pymongo import MongoClient

client = MongoClient('mongodb://localhost:27017/')
db = client['ml_platform']

# 存储实验元数据
experiments = db['experiments']

experiment = {
    'experiment_id': 'exp_12345',
    'model_type': 'bert-classifier',
    'hyperparameters': {
        'learning_rate': 0.001,
        'batch_size': 32,
        'epochs': 10
    },
    'metrics': {
        'accuracy': 0.95,
        'f1_score': 0.94
    },
    'artifacts': {
        'model_path': 's3://models/exp_12345/model.pt',
        'config_path': 's3://models/exp_12345/config.yaml'
    },
    'created_at': '2024-01-15T10:30:00Z',
    'status': 'completed'
}

experiments.insert_one(experiment)

# 查询实验
best_experiments = experiments.find(
    {'metrics.accuracy': {'$gte': 0.90}}
).sort('metrics.accuracy', -1).limit(10)

for exp in best_experiments:
    print(f"{exp['experiment_id']}: {exp['metrics']['accuracy']}")
```

---

## 6. 时序数据库

### InfluxDB 用于 ML 指标

```python
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

client = InfluxDBClient(url="http://localhost:8086", token="my-token", org="ml-org")
write_api = client.write_api(write_options=SYNCHRONOUS)

# 写入指标
def log_inference_metrics(model_name, latency_ms, batch_size):
    point = Point("inference_metrics") \
        .tag("model", model_name) \
        .field("latency_ms", latency_ms) \
        .field("batch_size", batch_size)

    write_api.write(bucket="ml-metrics", record=point)

# 查询指标
query_api = client.query_api()

query = '''
from(bucket: "ml-metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "inference_metrics")
  |> filter(fn: (r) => r.model == "bert-classifier")
  |> mean()
'''

tables = query_api.query(query)
for table in tables:
    for record in table.records:
        print(f"Average latency: {record.get_value()} ms")
```

---

## 7. ML 的 Vector Database

### Pinecone 用于相似性搜索

```python
import pinecone

pinecone.init(api_key="your-api-key", environment="us-west1-gcp")

# 创建 index
pinecone.create_index("ml-embeddings", dimension=768, metric="cosine")

index = pinecone.Index("ml-embeddings")

# 存储嵌入向量
embeddings = [
    ("doc1", [0.1, 0.2, ...], {"text": "ML infrastructure"}),
    ("doc2", [0.3, 0.4, ...], {"text": "Model serving"}),
]

index.upsert(vectors=embeddings)

# 相似性搜索
query_embedding = [0.15, 0.25, ...]
results = index.query(query_embedding, top_k=10, include_metadata=True)

for match in results['matches']:
    print(f"ID: {match['id']}, Score: {match['score']}, Text: {match['metadata']['text']}")
```

---

## 8. 数据库性能优化

### 查询优化清单

```sql
-- 1. 在频繁查询的列上添加索引
CREATE INDEX idx_predictions_user_timestamp
ON predictions (user_id, timestamp DESC);

-- 2. 使用 EXPLAIN 分析查询计划
EXPLAIN ANALYZE
SELECT * FROM predictions
WHERE user_id = 12345 AND timestamp > NOW() - INTERVAL '1 hour';

-- 3. 避免 SELECT *（只选择需要的列）
SELECT id, result, timestamp FROM predictions WHERE user_id = 12345;

-- 4. 使用连接池（SQLAlchemy）
engine = create_engine(
    'postgresql://user:pass@db:5432/mldb',
    pool_size=20,
    max_overflow=40,
    pool_pre_ping=True
)
```

---

## 9. 总结与关键要点

### 核心概念

1. **ML 工作负载的扩展策略**
   - 只读副本：3-10 倍读取吞吐量
   - 分片：无限水平扩展
   - 垂直扩展：简单但有限

2. **缓存** 减少数据库负载
   - Redis 用于热数据（用户特征、模型配置）
   - 延迟降低 10-50 倍
   - 实现缓存失效

3. **NoSQL** 用于非结构化 ML 数据
   - MongoDB：实验元数据
   - InfluxDB：时序指标
   - Pinecone：向量相似性搜索

4. **性能优化** 至关重要
   - 为频繁查询的列建立索引
   - 使用 EXPLAIN 分析查询
   - 连接池提高效率

### 获得的实用技能

✅ 设置只读副本以扩展读取
✅ 实现数据库分片策略
✅ 使用 Redis 缓存减少延迟
✅ 选择合适的 NoSQL 数据库
✅ 使用时序和 vector database
✅ 优化数据库性能

### 后续步骤

- **练习**：在 PostgreSQL 中设置只读副本
- **实验**：为 ML API 实现 Redis 缓存
- **探索**：尝试使用 InfluxDB 跟踪模型指标
- **挑战**：为 1 亿用户设计分片策略

### 额外资源

- [PostgreSQL Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [Database Sharding Explained](https://www.mongodb.com/basics/sharding)
- [Redis Documentation](https://redis.io/documentation)
- [InfluxDB for ML Metrics](https://www.influxdata.com/)
- [Pinecone Vector Database](https://www.pinecone.io/)

---

**恭喜！** 你现在已经了解了如何为 ML 工作负载扩展数据库，从只读副本和分片到针对不同 ML 数据类型专用的 NoSQL 数据库。

**模块完成！**
