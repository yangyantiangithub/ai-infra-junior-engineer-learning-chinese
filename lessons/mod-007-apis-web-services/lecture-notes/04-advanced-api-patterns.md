# 讲座 04：高级 API 模式和生产策略

## 目录
1. [介绍](#介绍)
2. [API 版本化策略](#api-版本化策略)
3. [ML API 的 GraphQL](#ml-api-的-graphql)
4. [用于高性能 API 的 gRPC](#用于高性能-api-的-grpc)
5. [速率限制和节流](#速率限制和节流)
6. [缓存策略](#缓存策略)
7. [API 网关](#api-网关)
8. [异步和后台任务](#异步和后台任务)
9. [Webhook 和事件驱动 API](#webhook-和事件驱动-api)
10. [API 监控和可观测性](#api-监控和可观测性)
11. [总结和关键要点](#总结和关键要点)

## 介绍

随着 ML API 的复杂性和规模的增长，基本的 REST 模式可能不够用。你需要高级技术来处理：
- **破坏性变更**而不中断现有客户端
- **高吞吐量**推理工作负载（10 万+ 请求/秒）
- 跨多个模型或数据源的 **复杂查询**
- **长时间运行**的训练或批量处理任务
- 模型重新训练时的 **实时更新**

本讲座涵盖专门针对 AI 基础设施的高级 API 模式和生产策略。

### 学习目标

在本讲座结束时，你将：
- 实现 API 版本化以管理破坏性变更
- 理解何时使用 GraphQL vs REST vs gRPC
- 为 ML 推理构建高性能 gRPC 服务
- 实现速率限制以保护 API 资源
- 为 ML 预测设计有效的缓存策略
- 使用 API 网关进行路由和管理
- 使用异步模式处理长时间运行的任务
- 通过适当的可观测性监控 API

### 先决条件
- 本模块的讲座 01-03（REST、FastAPI、认证）
- 理解 HTTP 和 API
- 基本的 Python 和 FastAPI 知识

**时长**：90 分钟
**难度**：中级到高级

---

## 1. API 版本化策略

### 为什么需要版本化 API？

```
场景：你需要更改 ML 模型输出格式

旧响应：
{
  "prediction": "cat",
  "confidence": 0.95
}

新响应（不兼容）：
{
  "predictions": [
    {"label": "cat", "score": 0.95},
    {"label": "dog", "score": 0.03}
  ],
  "model_version": "v2.0"
}

❌ 如果不上版本化就部署：
   - 现有客户端立即崩溃
   - 移动应用崩溃
   - 仪表盘显示错误
   - 用户投诉

✅ 有了版本化：
   - /v1/predict → 旧格式（仍然可用）
   - /v2/predict → 新格式（选择加入）
   - 数月内逐步迁移
```

### 版本化策略

#### 1. URL 路径版本化（推荐）

```python
from fastapi import FastAPI, APIRouter

app = FastAPI()

# V1 API
v1_router = APIRouter(prefix="/v1")

@v1_router.post("/predict")
async def predict_v1(input_data: dict):
    return {"prediction": "cat", "confidence": 0.95}

# V2 API
v2_router = APIRouter(prefix="/v2")

@v2_router.post("/predict")
async def predict_v2(input_data: dict):
    return {
        "predictions": [
            {"label": "cat", "score": 0.95},
            {"label": "dog", "score": 0.03}
        ],
        "model_version": "v2.0"
    }

app.include_router(v1_router)
app.include_router(v2_router)

# 两个 endpoint 都可用：
# POST /v1/predict
# POST /v2/predict
```

**优点**：清晰、简单、广泛理解
**缺点**：重复 URL 结构

#### 2. 头部版本化

```python
from fastapi import FastAPI, Header, HTTPException

app = FastAPI()

@app.post("/predict")
async def predict(
    input_data: dict,
    api_version: str = Header(default="v1", alias="API-Version")
):
    if api_version == "v1":
        return {"prediction": "cat", "confidence": 0.95}
    elif api_version == "v2":
        return {
            "predictions": [{"label": "cat", "score": 0.95}],
            "model_version": "v2.0"
        }
    else:
        raise HTTPException(400, f"Unsupported API version: {api_version}")

# 用法：
# curl -H "API-Version: v2" POST /predict
```

**优点**：URL 干净，更灵活
**缺点**：在 URL 中不可见，难以发现

#### 3. 查询参数版本化

```python
@app.post("/predict")
async def predict(input_data: dict, version: str = "v1"):
    if version == "v1":
        return old_format()
    elif version == "v2":
        return new_format()
```

**用法**：`POST /predict?version=v2`

**优点**：简单，与缓存兼容
**缺点**：污染查询参数，容易忘记

### 版本化最佳实践

1. **使用语义化版本控制**
   - 主要版本使用 `v1`、`v2`、`v3`
   - 次要更新（非破坏性）使用 `/v1.1/`

2. **逐步弃用**
   ```python
   @v1_router.post("/predict", deprecated=True)
   async def predict_v1(input_data: dict):
       """
       已弃用：请改用 /v2/predict。
       此 endpoint 将于 2024-12-31 移除。
       """
       # 添加弃用头部
       response.headers["X-API-Deprecated"] = "true"
       response.headers["X-API-Sunset-Date"] = "2024-12-31"
       return old_format()
   ```

3. **旧版本维护 6-12 个月**
4. **记录迁移指南**
5. **向客户端告警弃用**

---

## 2. ML API 的 GraphQL

### GraphQL vs REST

**REST 对 ML 的问题：**
```
# 场景：获取用户推荐

# REST：多个请求（N+1 问题）
GET /user/123                    # 用户信息
GET /user/123/preferences        # 用户偏好
GET /models/recommend?user=123   # 推荐
GET /models/123/metadata         # 模型元数据

# 4 个请求，过度获取数据

# GraphQL：单个请求
query {
  user(id: 123) {
    name
    preferences { categories }
    recommendations(limit: 10) {
      item_id
      score
      model { name, version }
    }
  }
}
```

### 何时为 ML 使用 GraphQL

✅ **适用于：**
- 复杂的嵌套数据查询
- 具有灵活数据需求的仪表盘
- 移动应用（减少带宽）
- 跨多个模型的批量预测

❌ **不适用于：**
- 高吞吐量、简单的推理（REST/gRPC 更好）
- 实时流式传输（WebSocket/gRPC 流式传输更好）

### 使用 Strawberry (FastAPI) 的 GraphQL 示例

```python
from fastapi import FastAPI
import strawberry
from strawberry.fastapi import GraphQLRouter

# 定义类型
@strawberry.type
class Prediction:
    label: str
    score: float

@strawberry.type
class Model:
    name: str
    version: str

@strawberry.type
class ModelPrediction:
    model: Model
    predictions: list[Prediction]

# 定义查询
@strawberry.type
class Query:
    @strawberry.field
    async def predict(self, input_data: str) -> ModelPrediction:
        # 运行推理
        predictions = await run_model(input_data)
        return ModelPrediction(
            model=Model(name="bert-classifier", version="v1.0"),
            predictions=[
                Prediction(label="positive", score=0.95),
                Prediction(label="negative", score=0.05)
            ]
        )

    @strawberry.field
    async def batch_predict(
        self, inputs: list[str], models: list[str]
    ) -> list[ModelPrediction]:
        # 跨多个模型运行预测
        results = []
        for model_name in models:
            model_predictions = await run_model_batch(model_name, inputs)
            results.append(model_predictions)
        return results

# 创建 schema
schema = strawberry.Schema(query=Query)

# 添加到 FastAPI
app = FastAPI()
graphql_app = GraphQLRouter(schema)
app.include_router(graphql_app, prefix="/graphql")

# 查询：
# {
#   predict(inputData: "This is great!") {
#     model { name version }
#     predictions { label score }
#   }
# }
```

---

## 3. 用于高性能 API 的 gRPC

### 为什么 ML 推理使用 gRPC？

```
性能对比：
┌──────────────────────────────────────┐
│  指标        │ REST  │ gRPC  │ Δ    │
│─────────────────────────────────────│
│  延迟        │ 50ms  │ 10ms  │ 5x ↓ │
│  吞吐量      │ 10k/s │ 100k/s│ 10x ↑│
│  带宽        │ 1MB/s │ 0.2MB │ 5x ↓ │
│  CPU 使用率  │ 40%   │ 15%   │ 2.7x↓│
└──────────────────────────────────────┘

为什么？
- 二进制协议 (Protobuf) vs JSON
- HTTP/2 多路复用 vs HTTP/1.1
- 代码生成 vs 手动序列化
- 内置流式传输
```

### 何时使用 gRPC

✅ **在以下情况使用 gRPC：**
- 微服务到微服务通信
- 高吞吐量 ML 推理（>1000 req/s）
- 流式预测（视频、音频）
- 模型服务后端（TensorFlow Serving、TorchServe）

❌ **不要将 gRPC 用于：**
- 面向公众的 API（浏览器支持有限）
- 简单的 CRUD 操作
- 当客户端无法使用 Protobuf 时

### ML 推理的 gRPC 示例

**1. 定义 protocol buffer（`.proto` 文件）：**

```protobuf
// inference.proto
syntax = "proto3";

package ml_inference;

service InferenceService {
  // 单次预测
  rpc Predict(PredictRequest) returns (PredictResponse);

  // 批量预测
  rpc BatchPredict(BatchPredictRequest) returns (BatchPredictResponse);

  // 流式预测
  rpc StreamPredict(stream PredictRequest) returns (stream PredictResponse);
}

message PredictRequest {
  string model_name = 1;
  repeated float features = 2;
  map<string, string> metadata = 3;
}

message Prediction {
  string label = 1;
  float score = 2;
}

message PredictResponse {
  repeated Prediction predictions = 1;
  string model_version = 2;
  float latency_ms = 3;
}

message BatchPredictRequest {
  string model_name = 1;
  repeated PredictRequest requests = 2;
}

message BatchPredictResponse {
  repeated PredictResponse responses = 1;
}
```

**2. 生成 Python 代码：**

```bash
python -m grpc_tools.protoc \
  -I. \
  --python_out=. \
  --grpc_python_out=. \
  inference.proto
```

**3. 实现服务器：**

```python
import grpc
from concurrent import futures
import inference_pb2
import inference_pb2_grpc
import numpy as np

class InferenceService(inference_pb2_grpc.InferenceServiceServicer):
    def __init__(self):
        # 加载模型
        self.model = load_model("bert-classifier")

    def Predict(self, request, context):
        # 运行推理
        features = np.array(request.features)
        predictions = self.model.predict(features)

        # 构建响应
        response = inference_pb2.PredictResponse(
            model_version="v1.0",
            latency_ms=12.5
        )

        for label, score in predictions:
            pred = response.predictions.add()
            pred.label = label
            pred.score = score

        return response

    def StreamPredict(self, request_iterator, context):
        # 流式预测
        for request in request_iterator:
            response = self.Predict(request, context)
            yield response

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    inference_pb2_grpc.add_InferenceServiceServicer_to_server(
        InferenceService(), server
    )
    server.add_insecure_port('[::]:50051')
    server.start()
    print("gRPC server started on port 50051")
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

**4. 实现客户端：**

```python
import grpc
import inference_pb2
import inference_pb2_grpc

def run_inference():
    channel = grpc.insecure_channel('localhost:50051')
    stub = inference_pb2_grpc.InferenceServiceStub(channel)

    request = inference_pb2.PredictRequest(
        model_name="bert-classifier",
        features=[0.1, 0.2, 0.3, 0.4]
    )

    response = stub.Predict(request)
    print(f"Predictions: {response.predictions}")
    print(f"Latency: {response.latency_ms}ms")
```

---

## 4. 速率限制和节流

### 为什么对 ML API 进行速率限制？

```
场景：没有速率限制的公共 ML API

用户每分钟发送 1,000,000 个请求
↓
GPU 资源耗尽
↓
所有人 API 宕机
↓
$10,000 云账单
```

### 速率限制策略

#### 1. 固定窗口

```python
from fastapi import FastAPI, Request, HTTPException
from collections import defaultdict
import time

app = FastAPI()

# 存储：{user_id: [(timestamp, count)]}
request_counts = defaultdict(list)

RATE_LIMIT = 100  # 每分钟请求数
WINDOW = 60  # 秒

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    user_id = request.headers.get("X-User-ID", "anonymous")
    current_time = time.time()

    # 清理旧请求
    request_counts[user_id] = [
        (ts, count) for ts, count in request_counts[user_id]
        if current_time - ts < WINDOW
    ]

    # 计算当前窗口中的请求数
    total_requests = sum(count for _, count in request_counts[user_id])

    if total_requests >= RATE_LIMIT:
        raise HTTPException(
            status_code=429,
            detail=f"Rate limit exceeded. Max {RATE_LIMIT} requests per minute."
        )

    # 记录请求
    request_counts[user_id].append((current_time, 1))

    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = str(RATE_LIMIT)
    response.headers["X-RateLimit-Remaining"] = str(RATE_LIMIT - total_requests - 1)
    return response
```

#### 2. 令牌桶（推荐）

更好地处理突发流量：

```python
import time
from fastapi import FastAPI, Request, HTTPException

class TokenBucket:
    def __init__(self, rate: int, capacity: int):
        self.rate = rate  # 每秒令牌数
        self.capacity = capacity  # 最大令牌数
        self.tokens = capacity
        self.last_update = time.time()

    def consume(self, tokens: int = 1) -> bool:
        current_time = time.time()
        # 补充令牌
        elapsed = current_time - self.last_update
        self.tokens = min(
            self.capacity,
            self.tokens + elapsed * self.rate
        )
        self.last_update = current_time

        # 检查是否有足够令牌
        if self.tokens >= tokens:
            self.tokens -= tokens
            return True
        return False

# 每个用户的桶
buckets = {}

@app.middleware("http")
async def token_bucket_middleware(request: Request, call_next):
    user_id = request.headers.get("X-User-ID", "anonymous")

    if user_id not in buckets:
        # 10 请求/秒，突发上限 50
        buckets[user_id] = TokenBucket(rate=10, capacity=50)

    if not buckets[user_id].consume():
        raise HTTPException(429, "Rate limit exceeded")

    return await call_next(request)
```

#### 3. 基于 Redis 的速率限制（生产环境）

```python
import aioredis
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()
redis = None

@app.on_event("startup")
async def startup():
    global redis
    redis = await aioredis.from_url("redis://localhost")

@app.middleware("http")
async def redis_rate_limit(request: Request, call_next):
    user_id = request.headers.get("X-User-ID", "anonymous")
    key = f"rate_limit:{user_id}"

    # 递增计数器
    current = await redis.incr(key)

    # 首次请求设置过期
    if current == 1:
        await redis.expire(key, 60)  # 60 秒窗口

    # 检查限制
    if current > 100:
        raise HTTPException(429, "Rate limit exceeded")

    return await call_next(request)
```

### 基于成本的速率限制

不同 endpoint 有不同的成本：

```python
COST_PER_ENDPOINT = {
    "/predict/small-model": 1,      # 1 个令牌
    "/predict/large-model": 10,     # 10 个令牌
    "/batch-predict": 50,           # 50 个令牌
}

@app.middleware("http")
async def cost_based_rate_limit(request: Request, call_next):
    user_id = request.headers.get("X-User-ID")
    endpoint = request.url.path

    cost = COST_PER_ENDPOINT.get(endpoint, 1)

    if not buckets[user_id].consume(cost):
        raise HTTPException(429, f"Insufficient quota (cost: {cost} tokens)")

    return await call_next(request)
```

---

## 5. 缓存策略

### 为什么缓存 ML 预测？

```
场景：图像分类 API

同一张图片被上传多次
├── 用户 1 上传 cat.jpg → 预测："cat" (100ms)
├── 用户 2 上传 cat.jpg → 预测："cat" (100ms) ← 浪费！
└── 用户 3 上传 cat.jpg → 预测："cat" (100ms) ← 浪费！

使用缓存：
├── 用户 1 上传 cat.jpg → 预测："cat" (100ms, 缓存 MISS)
├── 用户 2 上传 cat.jpg → 预测："cat" (2ms, 缓存 HIT) ✓
└── 用户 3 上传 cat.jpg → 预测："cat" (2ms, 缓存 HIT) ✓

节省：98ms × 2 = 196ms 节省，2× 更少的 GPU 使用
```

### 缓存策略

#### 1. 内存缓存 (LRU)

```python
from functools import lru_cache
import hashlib
import json

@lru_cache(maxsize=1000)
def predict_cached(input_hash: str):
    # 此函数被缓存
    return run_model(input_hash)

@app.post("/predict")
async def predict(input_data: dict):
    # 对输入进行哈希以获取缓存键
    input_json = json.dumps(input_data, sort_keys=True)
    input_hash = hashlib.md5(input_json.encode()).hexdigest()

    # 尝试缓存
    result = predict_cached(input_hash)
    return result
```

#### 2. Redis 缓存

```python
import aioredis
import json
import hashlib

redis = await aioredis.from_url("redis://localhost")

async def get_cached_prediction(input_data: dict):
    # 创建缓存键
    input_json = json.dumps(input_data, sort_keys=True)
    cache_key = f"pred:{hashlib.md5(input_json.encode()).hexdigest()}"

    # 尝试缓存
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # 缓存未命中 - 运行推理
    result = await run_model(input_data)

    # 存入缓存（TTL：1 小时）
    await redis.setex(cache_key, 3600, json.dumps(result))

    return result

@app.post("/predict")
async def predict(input_data: dict):
    return await get_cached_prediction(input_data)
```

### 缓存失效

```python
# 模型更新时使缓存失效
@app.post("/admin/update-model")
async def update_model(model_version: str):
    # 部署新模型
    deploy_model(model_version)

    # 清除预测缓存
    await redis.flushdb()  # 或使用模式：redis.delete("pred:*")

    return {"status": "Model updated, cache cleared"}
```

---

## 6. API 网关

### 什么是 API 网关？

```
没有网关：
客户端 → ML API 1
客户端 → ML API 2
客户端 → ML API 3
（每个客户端实现：认证、速率限制、日志、重试）

有网关：
客户端 → 网关 → ML API 1
               → ML API 2
               → ML API 3
（网关处理：认证、速率限制、日志、路由）
```

### 流行的 API 网关

- **Kong** - 开源、基于插件
- **AWS API Gateway** - 托管、无服务器
- **NGINX** - 高性能、灵活
- **Traefik** - Kubernetes 原生
- **Envoy** - 服务网格

### ML API 的 Kong 示例

```yaml
# kong.yml
services:
  - name: ml-inference-service
    url: http://ml-api:8000
    routes:
      - name: predict-route
        paths:
          - /predict
    plugins:
      - name: key-auth
      - name: rate-limiting
        config:
          minute: 100
          hour: 1000
      - name: request-transformer
        config:
          add:
            headers:
              - X-Gateway: Kong
      - name: prometheus
```

```bash
# 应用配置
curl -X POST http://localhost:8001/config \
  --data @kong.yml
```

---

## 7. 异步和后台任务

### 长时间运行的 ML 任务

```python
from fastapi import FastAPI, BackgroundTasks
from celery import Celery
import uuid

app = FastAPI()
celery_app = Celery('tasks', broker='redis://localhost:6379')

# 内存中的任务存储（生产环境使用 Redis）
jobs = {}

@celery_app.task
def train_model_task(job_id: str, config: dict):
    jobs[job_id] = {"status": "running"}
    try:
        # 训练模型（需要数小时）
        model = train_model(config)
        save_model(model)
        jobs[job_id] = {"status": "completed", "model_path": "models/model.pt"}
    except Exception as e:
        jobs[job_id] = {"status": "failed", "error": str(e)}

@app.post("/train")
async def start_training(config: dict):
    job_id = str(uuid.uuid4())
    train_model_task.delay(job_id, config)
    return {
        "job_id": job_id,
        "status": "queued",
        "status_url": f"/jobs/{job_id}"
    }

@app.get("/jobs/{job_id}")
async def get_job_status(job_id: str):
    if job_id not in jobs:
        return {"error": "Job not found"}
    return jobs[job_id]
```

---

## 8. 总结和关键要点

### 核心概念

1. **API 版本化**防止破坏性变更
   - 使用 URL 路径版本化（`/v1/`、`/v2/`）
   - 逐步弃用并提供停止日期
   - 旧版本维护 6-12 个月

2. **选择正确的协议**
   - REST：公共 API、简单 CRUD
   - GraphQL：复杂查询、灵活客户端
   - gRPC：高性能、微服务

3. **速率限制**保护资源
   - 使用令牌桶算法
   - 为 ML 实现基于成本的限制
   - 生产环境在 Redis 中存储状态

4. **缓存**节省计算资源
   - 按输入哈希缓存预测
   - 使用 Redis 进行分布式缓存
   - 模型更新时失效

5. **API 网关**集中处理横切关注点
   - 认证、速率限制、日志
   - 减少客户端复杂性
   - 简化监控

6. **异步任务**处理长时间运行的任务
   - 使用 Celery 或类似任务队列
   - 立即返回任务 ID
   - 提供状态 endpoint

### 获得的实践技能

✅ 版本化 API 以管理破坏性变更
✅ 实现高性能 gRPC 服务
✅ 添加速率限制以保护 ML 资源
✅ 缓存预测以降低延迟和成本
✅ 使用 API 网关进行集中管理
✅ 异步处理长时间运行的训练任务

### 下一步

- **练习**：为现有 API 添加版本化
- **实验**：比较 REST vs gRPC 性能
- **实现**：添加基于 Redis 的速率限制
- **部署**：为 ML API 设置 Kong 网关

### 附加资源

- [API 版本化最佳实践](https://www.troyhunt.com/your-api-versioning-is-wrong-which-is/)
- [GraphQL 官方文档](https://graphql.org/)
- [gRPC Python 快速入门](https://grpc.io/docs/languages/python/quickstart/)
- [Kong API 网关](https://konghq.com/)
- [FastAPI 后台任务](https://fastapi.tiangolo.com/tutorial/background-tasks/)

---

**恭喜！** 你现在了解了构建可扩展、高性能 ML API 的高级 API 模式和生产策略。

**下一讲**：模块 008 - 数据库和 SQL
