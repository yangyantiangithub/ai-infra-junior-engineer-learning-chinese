# 讲座 01：API 基础和 REST

## 目录
1. [介绍](#介绍)
2. [什么是 API？](#什么是-api)
3. [API 的类型](#api-的类型)
4. [REST 架构](#rest-架构)
5. [HTTP 协议基础](#http-协议基础)
6. [RESTful API 设计原则](#restful-api-设计原则)
7. [API 请求和响应结构](#api-请求和响应结构)
8. [状态码和错误处理](#状态码和错误处理)
9. [API 版本化策略](#api-版本化策略)
10. [AI/ML 基础设施中的 API](#aiml-基础设施中的-api)
11. [最佳实践](#最佳实践)
12. [总结和关键要点](#总结和关键要点)

## 介绍

应用程序编程接口 (API) 是现代软件架构的支柱，使不同的系统、服务和应用程序能够相互通信。在 AI/ML 基础设施中，API 在模型服务、数据管道编排以及与各种工具和平台的集成中发挥着关键作用。

理解 API 的工作原理——特别是 RESTful API——对于构建、部署和维护 AI 基础设施至关重要。本讲座将为你提供 API 概念、REST 架构风格以及你将在整个职业生涯中使用的实际设计模式的全面基础。

### 学习目标

在本讲座结束时，你将：
- 理解什么是 API 以及为什么它们对现代软件至关重要
- 区分不同类型的 API 以及何时使用每种类型
- 掌握 REST 架构原则和约束
- 理解 HTTP 协议基础（方法、头部、状态码）
- 按照最佳实践设计 RESTful API
- 处理错误并实现适当的状态码响应
- 将 API 设计原则应用于 AI/ML 用例
- 理解 API 版本化策略

### 先决条件
- 模块 001：Python 基础
- 模块 002：Linux 和命令行
- 对客户端-服务器架构的基本理解
- 熟悉 JSON 数据格式

### 预计时间
4-5 小时（包括动手示例）

## 什么是 API？

### 定义

**应用程序编程接口 (API)** 是定义软件组件应如何交互的契约或接口。它规定了：
- 哪些操作可用
- 每个操作需要哪些输入
- 每个操作返回哪些输出
- 可能发生哪些错误

可以把 API 想象成餐厅的菜单：
- 菜单列出你可以点的菜（可用操作）
- 每道菜都有描述（文档）
- 你提出请求（下单）
- 厨房处理请求（后端逻辑）
- 你收到响应（你的食物，如果售罄则收到错误消息）

### API 为什么重要

API 实现了：

1. **关注点分离**：前端和后端可以独立开发
2. **可复用性**：一个 API 可以服务多个客户端（Web、移动端、IoT 设备）
3. **可扩展性**：服务可以独立扩展
4. **集成**：不同系统可以协同工作
5. **抽象**：将复杂的实现细节隐藏在简单接口背后

### 实际例子：ML 模型服务

当你部署机器学习模型时，你通过 API 暴露它：

```python
# 客户端发起请求
POST /api/v1/predict
{
  "features": [1.2, 3.4, 5.6, 7.8]
}

# 服务器返回预测结果
{
  "prediction": "class_A",
  "confidence": 0.92,
  "latency_ms": 45
}
```

客户端不需要知道：
- 你使用哪个 ML 框架（PyTorch、TensorFlow）
- 模型是如何训练的
- 模型权重存储在哪里
- 预处理是如何完成的

API 将所有细节抽象在简单接口背后。

## API 的类型

### 1. Web API

通过 HTTP/HTTPS 可访问的 API，用于客户端和服务器之间通过互联网进行通信。

**Web API 的类型：**

#### a) REST (Representational State Transfer)
- 使用 HTTP 方法（GET、POST、PUT、DELETE）
- 无状态的客户端-服务器通信
- 基于资源的 URL
- 公共 API 中最流行

**示例：**
```
GET https://api.example.com/models/bert-base
POST https://api.example.com/predictions
```

#### b) GraphQL
- API 的查询语言
- 客户端精确指定需要哪些数据
- 所有操作使用单个 endpoint
- 减少过度获取和获取不足

**示例：**
```graphql
query {
  model(id: "bert-base") {
    name
    version
    metrics {
      accuracy
      f1Score
    }
  }
}
```

#### c) SOAP (Simple Object Access Protocol)
- 基于 XML 的协议
- 严格的标准和契约 (WSDL)
- 常见于企业级和遗留系统
- 比 REST 开销更大

#### d) gRPC
- 高性能 RPC 框架
- 使用 Protocol Buffers 进行序列化
- 双向流
- 微服务通信的理想选择

**示例：**
```protobuf
service PredictionService {
  rpc Predict(PredictRequest) returns (PredictResponse);
}
```

### 2. 库/框架 API

编程库和框架提供的接口：

```python
# TensorFlow API
model = tf.keras.Sequential([
    tf.keras.layers.Dense(128, activation='relu'),
    tf.keras.layers.Dense(10, activation='softmax')
])

# PyTorch API
model = nn.Sequential(
    nn.Linear(784, 128),
    nn.ReLU(),
    nn.Linear(128, 10)
)
```

### 3. 操作系统 API

用于与操作系统交互的接口：

```python
import os
import subprocess

# 用于文件操作的 OS API
files = os.listdir('/data/models')

# 用于进程管理的 OS API
result = subprocess.run(['nvidia-smi'], capture_output=True)
```

### 4. 硬件 API

用于硬件交互的接口：

```python
import torch

# CUDA API（通过 PyTorch 抽象）
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model.to(device)
```

### 重点：Web API 和 REST

在本讲座的剩余部分，我们将专注于 **RESTful Web API**，因为它们是你将在 AI/ML 基础设施中最常使用的类型。

## REST 架构

### 什么是 REST？

REST (Representational State Transfer) 是一种 **架构风格**，而不是协议或标准。它由 Roy Fielding 在其 2000 年的博士论文中提出，并已成为构建 Web API 的主要方法。

### REST 约束

真正的 RESTful API 必须遵守六项约束：

#### 1. 客户端-服务器架构

客户端和服务器是 **分离的** 关注点：
- **客户端**：处理用户界面和用户体验
- **服务器**：处理数据存储和业务逻辑

**优点：**
- 客户端和服务器可以独立演进
- 更好的可扩展性
- 更简单的组件

```
┌─────────┐                    ┌─────────┐
│ Client  │ ←─── HTTP ────→ │ Server  │
│ (React) │                    │ (Python)│
└─────────┘                    └─────────┘
```

#### 2. 无状态

从客户端到服务器的每个请求必须包含 **所有信息** 来理解和处理该请求。服务器不在请求之间存储客户端上下文。

**无状态示例（良好）：**
```
GET /api/users/123
Authorization: Bearer eyJhbGciOiJ...
```
每个请求都包含认证令牌。

**有状态示例（不良）：**
```
POST /api/login
{ "username": "alice", "password": "***" }

# 服务器存储会话
# 后续请求不包含认证信息
GET /api/users/123
```

**无状态的优势：**
- 更容易水平扩展（任何服务器可以处理任何请求）
- 更简单的服务器实现
- 更好的可靠性（没有需要丢失的会话状态）
- 提高监控的可见性

#### 3. 可缓存

响应必须将自身定义为 **可缓存或不可缓存**，以防止客户端重复使用过时数据。

```http
HTTP/1.1 200 OK
Cache-Control: max-age=3600
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"

{
  "model_name": "bert-base",
  "version": "1.0.0"
}
```

**优点：**
- 减少延迟
- 减少服务器负载
- 改善用户体验

#### 4. 统一接口

客户端和服务器之间一致、标准化的接口。这是区分 REST 与其他架构的基本约束。

**四个子约束：**

**a) 资源标识**
每个资源有唯一的标识符 (URI)：
```
/api/models/bert-base
/api/datasets/imagenet
/api/jobs/training-job-123
```

**b) 通过表示操作资源**
客户端通过表示（通常是 JSON）操作资源：
```json
{
  "id": "bert-base",
  "name": "BERT Base Model",
  "parameters": 110000000,
  "status": "ready"
}
```

**c) 自描述消息**
每条消息包含足够的信息来描述如何处理它：
```http
POST /api/predictions HTTP/1.1
Content-Type: application/json
Accept: application/json

{"features": [1.2, 3.4, 5.6]}
```

**d) 超媒体作为应用状态的引擎 (HATEOAS)**
响应包含指向相关资源的链接：
```json
{
  "id": "bert-base",
  "name": "BERT Base Model",
  "_links": {
    "self": "/api/models/bert-base",
    "predict": "/api/models/bert-base/predict",
    "metrics": "/api/models/bert-base/metrics"
  }
}
```

#### 5. 分层系统

架构可以由层次化层组成，每层只了解与其直接交互的层。

```
┌──────────────┐
│  Client      │
├──────────────┤
│  Load Balancer│
├──────────────┤
│  API Gateway │
├──────────────┤
│  Auth Service│
├──────────────┤
│  API Server  │
├──────────────┤
│  Database    │
└──────────────┘
```

客户端不知道（也不关心）中间的各层。

**优点：**
- 提高安全性（中间层可以实施策略）
- 在不同层进行负载均衡和缓存
- 封装遗留服务

#### 6. 按需代码（可选）

服务器可以通过传输可执行代码（例如 JavaScript）临时扩展客户端功能。

这是唯一可选的约束，在 REST API 中很少使用。

### 资源：REST 的基础

在 REST 中，一切都是 **资源**。资源是可以命名的任何信息：
- 文档或图像
- 数据库记录
- 其他资源的集合
- 非虚拟对象（例如，一个人）

**ML 基础设施中的资源示例：**
- 模型：`/api/models/bert-base`
- 数据集：`/api/datasets/imagenet`
- 训练任务：`/api/jobs/training-job-123`
- 预测结果：`/api/predictions/pred-456`
- 实验：`/api/experiments/exp-789`

**资源命名约定：**
- 使用 **名词**，而不是动词：`/users` 而不是 `/getUsers`
- 使用 **复数** 形式：`/models` 而不是 `/model`
- 对关系使用 **层次结构**：`/models/bert-base/versions/1.0.0`
- 使用 **连字符** 提高可读性：`/training-jobs` 而不是 `/trainingJobs`
- 使用 **小写**：`/api/models` 而不是 `/API/Models`

## HTTP 协议基础

REST API 构建在 HTTP（超文本传输协议）之上。理解 HTTP 对于设计和消费 REST API 至关重要。

### HTTP 请求结构

HTTP 请求由以下组成：

```http
POST /api/v1/predictions HTTP/1.1
Host: ml-api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: application/json
User-Agent: Python-Requests/2.28.1

{
  "model_id": "bert-base",
  "input_text": "This is a test sentence.",
  "max_length": 512
}
```

**组成部分：**
1. **请求行**：方法、URI、HTTP 版本
2. **头部**：关于请求的元数据
3. **请求体**：正在发送的数据（可选，GET 中不使用）

### HTTP 方法（动词）

HTTP 方法定义了要对资源执行的 **操作**。

#### GET - 检索资源

**目的**：获取数据而不修改它

**属性：**
- 安全（不修改服务器状态）
- 幂等（多次相同请求与一次效果相同）
- 可缓存

**示例：**
```http
GET /api/models                    # 列出所有模型
GET /api/models/bert-base         # 获取特定模型
GET /api/models?status=ready      # 过滤模型
GET /api/models/bert-base/metrics # 获取模型指标
```

**响应：**
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": "bert-base",
  "name": "BERT Base Model",
  "status": "ready",
  "created_at": "2024-10-01T10:00:00Z"
}
```

#### POST - 创建新资源

**目的**：创建新资源或触发操作

**属性：**
- 不安全（修改服务器状态）
- 不幂等（多次请求创建多个资源）
- 不可缓存

**示例：**
```http
POST /api/models                  # 创建新模型
POST /api/predictions             # 请求预测
POST /api/jobs/training           # 启动训练任务
```

**请求：**
```http
POST /api/models HTTP/1.1
Content-Type: application/json

{
  "name": "Custom BERT Model",
  "base_model": "bert-base",
  "task": "sentiment-analysis"
}
```

**响应：**
```http
HTTP/1.1 201 Created
Location: /api/models/custom-bert-123
Content-Type: application/json

{
  "id": "custom-bert-123",
  "name": "Custom BERT Model",
  "status": "initializing",
  "created_at": "2024-10-18T14:30:00Z"
}
```

#### PUT - 更新/替换资源

**目的**：更新现有资源，如果不存在则创建（在特定 URI 处）

**属性：**
- 不安全
- **幂等**（多次相同请求效果相同）
- 不可缓存

**示例：**
```http
PUT /api/models/bert-base         # 替换整个模型配置
PUT /api/models/bert-base/config  # 替换模型配置
```

**请求：**
```http
PUT /api/models/bert-base/config HTTP/1.1
Content-Type: application/json

{
  "max_sequence_length": 512,
  "batch_size": 32,
  "learning_rate": 0.001
}
```

**关键点**：PUT 替换 **整个** 资源。如果省略某个字段，它可能被删除或设置为默认值。

#### PATCH - 部分更新

**目的**：对资源应用部分修改

**属性：**
- 不安全
- 可以是幂等的（取决于实现）
- 不可缓存

**示例：**
```http
PATCH /api/models/bert-base       # 更新特定字段
```

**请求：**
```http
PATCH /api/models/bert-base HTTP/1.1
Content-Type: application/json

{
  "status": "archived"
}
```

只有 `status` 字段被更新；其他字段保持不变。

#### DELETE - 删除资源

**目的**：删除资源

**属性：**
- 不安全
- **幂等**（多次删除与一次效果相同）
- 不可缓存

**示例：**
```http
DELETE /api/models/bert-base
DELETE /api/jobs/training-job-123
```

**响应：**
```http
HTTP/1.1 204 No Content
```

#### 其他方法

**HEAD**：与 GET 相同，但只返回头部（无请求体）
```http
HEAD /api/models/bert-base  # 检查资源是否存在
```

**OPTIONS**：描述资源的通信选项
```http
OPTIONS /api/models
# 响应包含 Allow 头部，列出支持的方法
```

### HTTP 头部

头部提供关于请求或响应的元数据。

#### 常见请求头部

```http
# 内容协商
Accept: application/json
Accept-Language: en-US
Accept-Encoding: gzip, deflate

# 认证
Authorization: Bearer <token>
Authorization: Basic <base64-credentials>

# 内容类型
Content-Type: application/json
Content-Length: 1234

# 客户端信息
User-Agent: Mozilla/5.0 ...
Referer: https://example.com/page

# 自定义头部（使用 X- 前缀或特定域前缀）
X-Request-ID: abc-123-def-456
X-API-Version: 2.0
```

#### 常见响应头部

```http
# 缓存
Cache-Control: max-age=3600, must-revalidate
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
Expires: Wed, 21 Oct 2024 07:28:00 GMT

# 内容信息
Content-Type: application/json; charset=utf-8
Content-Length: 1234
Content-Encoding: gzip

# CORS (跨域资源共享)
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, PUT, DELETE

# 速率限制
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 999
X-RateLimit-Reset: 1634567890

# 已创建资源的位置
Location: /api/models/new-model-123
```

## RESTful API 设计原则

### 1. 资源使用名词，而不是动词

**良好：**
```
GET    /api/models              # 获取所有模型
POST   /api/models              # 创建模型
GET    /api/models/bert-base    # 获取特定模型
PUT    /api/models/bert-base    # 更新模型
DELETE /api/models/bert-base    # 删除模型
```

**不良：**
```
GET  /api/getAllModels
POST /api/createModel
GET  /api/getModel/bert-base
POST /api/updateModel/bert-base
POST /api/deleteModel/bert-base
```

HTTP 方法已经指示了操作。URL 应标识资源。

### 2. 集合使用复数名词

**良好：**
```
/api/models
/api/datasets
/api/training-jobs
```

**不良：**
```
/api/model
/api/dataset
/api/training-job
```

一致性：无论你获取一个还是多个，你仍然在访问"models"集合。

### 3. 使用层次结构表示关系

```
GET /api/models/bert-base/versions           # 模型的所有版本
GET /api/models/bert-base/versions/1.0.0     # 特定版本
GET /api/datasets/imagenet/splits/train      # 训练拆分
GET /api/experiments/exp-123/runs/run-456    # 特定运行
```

### 4. 使用查询参数进行过滤、排序和分页

**过滤：**
```
GET /api/models?status=ready
GET /api/models?type=transformer&status=ready
GET /api/models?created_after=2024-01-01
```

**排序：**
```
GET /api/models?sort=created_at
GET /api/models?sort=-created_at          # 降序
GET /api/models?sort=name,created_at      # 多字段
```

**分页：**
```
GET /api/models?page=2&per_page=20
GET /api/models?offset=40&limit=20
GET /api/models?cursor=eyJpZCI6MTIzfQ==    # 基于游标
```

**全文搜索：**
```
GET /api/models?q=bert+sentiment
```

### 5. 返回适当的状态码

不要对所有内容都返回 200 OK！使用语义化的状态码。

### 6. 提供有意义的错误消息

**不良：**
```json
{
  "error": "Bad request"
}
```

**良好：**
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid model configuration",
    "details": [
      {
        "field": "batch_size",
        "issue": "must be a positive integer",
        "provided": -5
      }
    ],
    "documentation_url": "https://docs.example.com/api/models#validation"
  }
}
```

### 7. 对你的 API 进行版本化

API 随时间演变。版本化可以防止破坏现有客户端。

**URL 版本化：**
```
/api/v1/models
/api/v2/models
```

**头部版本化：**
```http
GET /api/models
Accept: application/vnd.example.v2+json
```

**查询参数版本化：**
```
/api/models?version=2
```

## API 请求和响应结构

### 请求结构

一个结构良好的 API 请求包括：

```http
POST /api/v1/predictions HTTP/1.1
Host: ml-api.example.com
Content-Type: application/json
Authorization: Bearer eyJ...
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000

{
  "model_id": "sentiment-bert-v2",
  "inputs": {
    "text": "This product exceeded my expectations!",
    "language": "en"
  },
  "parameters": {
    "max_length": 512,
    "return_probabilities": true
  }
}
```

### 响应结构

一个结构良好的 API 响应包括：

```http
HTTP/1.1 200 OK
Content-Type: application/json
X-Request-ID: 550e8400-e29b-41d4-a716-446655440000
X-RateLimit-Remaining: 4999
Cache-Control: no-cache

{
  "data": {
    "prediction": "positive",
    "confidence": 0.94,
    "probabilities": {
      "positive": 0.94,
      "neutral": 0.04,
      "negative": 0.02
    },
    "processing_time_ms": 23
  },
  "metadata": {
    "model_version": "2.1.0",
    "timestamp": "2024-10-18T14:30:00Z",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### 响应封装

**带封装（一致的结构）：**
```json
{
  "status": "success",
  "data": { ... },
  "metadata": { ... }
}
```

**无封装（直接数据）：**
```json
{
  "id": "model-123",
  "name": "BERT Model"
}
```

两种方法都有效。选择一种并保持一致。

## 状态码和错误处理

### HTTP 状态码类别

HTTP 状态码是三位数字，分为五个类别：

#### 1xx - 信息性
在 REST API 中很少使用。
- `100 Continue`：服务器已收到头部，客户端可以继续发送请求体

#### 2xx - 成功

- **200 OK**：请求成功（通用成功）
  ```http
  GET /api/models/bert-base
  → 200 OK
  ```

- **201 Created**：新资源创建成功
  ```http
  POST /api/models
  → 201 Created
  Location: /api/models/new-model-123
  ```

- **202 Accepted**：请求已接受处理（异步）
  ```http
  POST /api/jobs/training
  → 202 Accepted
  Location: /api/jobs/training-job-456
  ```

- **204 No Content**：成功，无请求体返回
  ```http
  DELETE /api/models/bert-base
  → 204 No Content
  ```

#### 3xx - 重定向

- **301 Moved Permanently**：资源已永久移动
  ```http
  GET /api/old-endpoint
  → 301 Moved Permanently
  Location: /api/new-endpoint
  ```

- **304 Not Modified**：资源未更改（缓存）
  ```http
  GET /api/models/bert-base
  If-None-Match: "abc123"
  → 304 Not Modified
  ```

#### 4xx - 客户端错误

- **400 Bad Request**：格式错误的请求或无效数据
  ```json
  {
    "error": "Invalid JSON syntax in request body"
  }
  ```

- **401 Unauthorized**：需要认证或认证失败
  ```json
  {
    "error": "Missing or invalid authentication token"
  }
  ```

- **403 Forbidden**：已认证但未授权
  ```json
  {
    "error": "You don't have permission to delete this model"
  }
  ```

- **404 Not Found**：资源不存在
  ```json
  {
    "error": "Model 'bert-base' not found"
  }
  ```

- **405 Method Not Allowed**：HTTP 方法不受支持
  ```http
  DELETE /api/models/read-only-model
  → 405 Method Not Allowed
  Allow: GET, HEAD
  ```

- **409 Conflict**：请求与服务器状态冲突
  ```json
  {
    "error": "Model with name 'bert-base' already exists"
  }
  ```

- **422 Unprocessable Entity**：语法正确但语义无效
  ```json
  {
    "error": "batch_size must be between 1 and 128",
    "provided": 200
  }
  ```

- **429 Too Many Requests**：超过速率限制
  ```http
  → 429 Too Many Requests
  Retry-After: 60
  X-RateLimit-Reset: 1634567890
  ```

#### 5xx - 服务器错误

- **500 Internal Server Error**：意外的服务器错误
  ```json
  {
    "error": "An unexpected error occurred. Please try again later.",
    "request_id": "550e8400-e29b-41d4-a716-446655440000"
  }
  ```

- **502 Bad Gateway**：来自上游服务器的无效响应
- **503 Service Unavailable**：服务器暂时不可用
  ```http
  → 503 Service Unavailable
  Retry-After: 120
  ```

- **504 Gateway Timeout**：上游服务器超时

### 错误响应结构

一个好的错误响应包括：

```json
{
  "error": {
    "code": "MODEL_NOT_FOUND",
    "message": "The requested model does not exist",
    "details": {
      "model_id": "non-existent-model",
      "available_models": ["/api/models"]
    },
    "request_id": "550e8400-e29b-41d4-a716-446655440000",
    "timestamp": "2024-10-18T14:30:00Z",
    "documentation": "https://docs.example.com/errors#model-not-found"
  }
}
```

**关键元素：**
1. **机器可读的代码**：用于编程式错误处理
2. **人类可读的消息**：供开发者调试
3. **详细信息**：特定上下文的信息
4. **请求 ID**：用于追踪和支持
5. **文档链接**：帮助用户解决问题

## API 版本化策略

随着你的 API 演进，你需要一种策略来在不破坏现有客户端的情况下引入变更。

### 1. URL 路径版本化

**优点：**
- 清晰明确
- 易于路由请求
- 客户端指定版本简单

**缺点：**
- 可能导致 URL 激增
- 打破了稳定的资源标识符的 REST 原则

```
https://api.example.com/v1/models
https://api.example.com/v2/models
```

### 2. 查询参数版本化

```
https://api.example.com/models?version=2
```

**优点：**
- 保持 URL 结构干净
- 可选（可以默认使用最新版）

**缺点：**
- 容易忘记
- 可能被错误缓存

### 3. 头部版本化

```http
GET /api/models
Accept: application/vnd.example.v2+json
```

或自定义头部：
```http
GET /api/models
X-API-Version: 2
```

**优点：**
- URL 干净
- 遵循 HTTP 内容协商模式

**缺点：**
- 不太可见（不在 URL 中）
- 在浏览器中测试较困难

### 4. 内容协商

```http
GET /api/models
Accept: application/vnd.example.model.v2+json
```

**优点：**
- 真正的 REST 方法
- 每个资源的细粒度版本化

**缺点：**
- 实现复杂
- 对客户端不太直观

### 版本化最佳实践

1. **从第一天起就以 v1 开始**
2. **只有重大变更**才需要新版本
3. **在合理期限内支持旧版本**（弃用策略）
4. **清晰沟通变更**通过：
   - 变更日志
   - 响应中的弃用警告
   - 文档
5. **使用 Sunset 头部**指示版本退役：
   ```http
   Sunset: Sat, 31 Dec 2024 23:59:59 GMT
   Deprecation: true
   Link: <https://api.example.com/v2/models>; rel="successor-version"
   ```

## AI/ML 基础设施中的 API

### 常见用例

#### 1. 模型服务/推理 API

暴露训练好的模型进行预测：

```python
POST /api/v1/models/bert-sentiment/predict
{
  "text": "This movie was amazing!",
  "return_confidence": true
}

Response:
{
  "sentiment": "positive",
  "confidence": 0.98,
  "latency_ms": 45
}
```

#### 2. 模型管理 API

管理模型生命周期：

```python
# 列出模型
GET /api/v1/models

# 注册新模型
POST /api/v1/models
{
  "name": "custom-bert",
  "framework": "pytorch",
  "model_url": "s3://models/custom-bert.pt"
}

# 更新模型元数据
PATCH /api/v1/models/custom-bert
{
  "status": "production",
  "tags": ["nlp", "sentiment-analysis"]
}

# 删除模型
DELETE /api/v1/models/custom-bert
```

#### 3. 训练任务 API

提交和监控训练任务：

```python
# 开始训练
POST /api/v1/training-jobs
{
  "model_type": "bert",
  "dataset": "imdb-sentiment",
  "config": {
    "epochs": 10,
    "batch_size": 32,
    "learning_rate": 0.001
  }
}

# 检查状态
GET /api/v1/training-jobs/job-123

# 获取指标
GET /api/v1/training-jobs/job-123/metrics
```

#### 4. 数据管道 API

编排数据处理：

```python
# 触发管道
POST /api/v1/pipelines/data-preprocessing/run
{
  "input_dataset": "raw-data-v2",
  "transformations": ["clean", "tokenize", "normalize"]
}

# 监控管道
GET /api/v1/pipelines/run-456/status
```

#### 5. 实验跟踪 API

跟踪 ML 实验：

```python
# 创建实验
POST /api/v1/experiments
{
  "name": "bert-hyperparameter-tuning",
  "description": "Testing different learning rates"
}

# 记录指标
POST /api/v1/experiments/exp-123/metrics
{
  "accuracy": 0.92,
  "loss": 0.15,
  "epoch": 5
}

# 记录参数
POST /api/v1/experiments/exp-123/params
{
  "learning_rate": 0.001,
  "batch_size": 32
}
```

### ML API 的设计考虑

1. **批量预测**：支持单次和批量推理
   ```python
   POST /api/v1/predict
   {
     "inputs": [
       {"text": "Sample 1"},
       {"text": "Sample 2"},
       {"text": "Sample 3"}
     ]
   }
   ```

2. **异步处理**：长时间运行的任务应该是异步的
   ```python
   POST /api/v1/training-jobs
   → 202 Accepted
   Location: /api/v1/training-jobs/job-123

   # 客户端轮询状态
   GET /api/v1/training-jobs/job-123
   {
     "status": "running",
     "progress": 0.45
   }
   ```

3. **流式响应**：用于实时或大型输出
   ```python
   GET /api/v1/logs/training-job-123
   # 服务器推送事件或 WebSocket 流
   ```

4. **模型版本化**：支持多个模型版本
   ```python
   POST /api/v1/models/bert-sentiment/v1.0.0/predict
   POST /api/v1/models/bert-sentiment/v2.0.0/predict
   ```

5. **健康检查**：部署中必不可少
   ```python
   GET /health
   {
     "status": "healthy",
     "model_loaded": true,
     "gpu_available": true,
     "latency_p95_ms": 120
   }
   ```

## 最佳实践

### 1. 文档

- **OpenAPI/Swagger**：使用标准的文档格式
- **示例**：提供请求/响应示例
- **错误码**：记录所有可能的错误
- **速率限制**：明确说明限制和策略

### 2. 安全

- **仅 HTTPS**：加密所有 API 流量
- **认证**：所有非公共 endpoint 需要认证
- **授权**：实现细粒度的权限控制
- **速率限制**：防止滥用
- **输入验证**：全面验证所有输入
- **API key**：定期轮换密钥

### 3. 性能

- **缓存**：适当使用 HTTP 缓存头部
- **压缩**：启用 gzip/brotli 压缩
- **分页**：大型集合始终分页
- **异步操作**：长时间运行的任务使用异步
- **连接池**：复用连接

### 4. 监控和可观测性

- **日志**：记录所有请求及其请求 ID
- **指标**：跟踪延迟、吞吐量、错误率
- **追踪**：实现分布式追踪
- **告警**：设置异常告警

### 5. 向后兼容性

- **增量变更**：添加新字段，不要移除旧字段
- **弃用警告**：在移除功能前发出警告
- **版本支持**：在过渡期内维护旧版本

## 总结和关键要点

### 核心概念

1. **API 是契约**，定义系统如何交互
2. **REST 是一种架构风格**，基于六项约束
3. **资源是名词**，由 URI 标识
4. **HTTP 方法是动词**，定义对资源的操作
5. **状态码传达结果**，具有语义含义

### RESTful 原则

- **无状态**：每个请求是独立的
- **可缓存**：响应定义可缓存性
- **统一接口**：一致的资源交互
- **分层系统**：通过层次进行抽象

### 设计最佳实践

- 对资源集合使用 **复数名词**
- 对关系使用 **层次结构**
- 对过滤和排序使用 **查询参数**
- 返回 **适当的状态码**
- 提供 **有意义的错误消息**
- 从一开始就 **对你的 API 进行版本化**

### ML 特定考虑

- 支持 **批量** 和 **流式** 推理
- 对训练实现 **异步** 操作
- 通过 API 暴露 **模型版本化**
- 包含 **健康检查** endpoint
- 设计以支持 **高吞吐量** 和 **低延迟**

### 下一步

在下一讲中，我们将深入探讨 **FastAPI**，一个用于构建高性能 REST API 的现代 Python 框架。你将学习如何使用 FastAPI 的功能（如自动文档、请求验证和异步支持）来实现这里涵盖的原则。

### 延伸阅读

- [Roy Fielding 的 REST 论文](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [HTTP/1.1 规范 (RFC 7231)](https://tools.ietf.org/html/rfc7231)
- [OpenAPI 规范](https://swagger.io/specification/)
- [RESTful API 设计最佳实践](https://restfulapi.net/)

---

**预计学习时间**：4-5 小时
**动手练习**：完成练习 01 来构建你的第一个 REST API
**评估**：测验 01 涵盖本讲座的所有概念
