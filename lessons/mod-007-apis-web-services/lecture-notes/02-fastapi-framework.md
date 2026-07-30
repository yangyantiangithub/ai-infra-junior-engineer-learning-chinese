# 讲座 02：FastAPI 框架

## 目录
1. [介绍](#介绍)
2. [什么是 FastAPI？](#什么是-fastapi)
3. [FastAPI 安装和设置](#fastapi-安装和设置)
4. [你的第一个 FastAPI 应用程序](#你的第一个-fastapi-应用程序)
5. [路径操作和路由](#路径操作和路由)
6. [请求和响应模型](#请求和响应模型)
7. [路径参数和查询参数](#路径参数和查询参数)
8. [请求体和验证](#请求体和验证)
9. [响应模型和状态码](#响应模型和状态码)
10. [依赖注入](#依赖注入)
11. [错误处理和自定义异常](#错误处理和自定义异常)
12. [后台任务](#后台任务)
13. [使用 FastAPI 构建 ML API](#使用-fastapi-构建-ml-api)
14. [测试 FastAPI 应用程序](#测试-fastapi-应用程序)
15. [最佳实践](#最佳实践)
16. [总结和关键要点](#总结和关键要点)

## 介绍

FastAPI 是一个现代、高性能的 Web 框架，用于使用基于标准 Python 类型提示的 Python 3.7+ 构建 API。它已迅速成为构建生产级 API 的最流行框架之一，特别是在机器学习和数据科学社区。

本讲座将指导你了解 FastAPI 的核心概念和特性，教你如何为 AI/ML 应用程序构建健壮、高性能的 API。

### 学习目标

在本讲座结束时，你将：
- 理解 FastAPI 的独特之处以及为什么它适合 ML 应用程序
- 设置 FastAPI 开发环境
- 使用各种 HTTP 方法创建路径操作
- 使用自动验证定义请求和响应模型
- 实现路径参数、查询参数和请求体
- 使用 Pydantic 进行数据验证和序列化
- 使用依赖注入编写更清晰的代码
- 使用自定义异常优雅地处理错误
- 构建异步 endpoint 以提高性能
- 创建完整的 ML 模型服务 API
- 有效测试 FastAPI 应用程序

### 先决条件
- 模块 001：Python 基础
- 模块 007 - 讲座 01：API 基础和 REST
- 理解 Python 类型提示
- 基本的 async/await 概念（有帮助但不是必需的）

### 预计时间
4-5 小时（包括动手编码）

## 什么是 FastAPI？

### 概述

**FastAPI** 是一个现代、快速（高性能）的 Web 框架，用于使用基于标准 Python 类型提示的 Python 3.7+ 构建 API。由 Sebastián Ramírez 于 2018 年创建，由于其性能、开发者体验和自动文档生成而迅速获得采用。

### 主要特性

1. **快速**：非常高的性能，与 NodeJS 和 Go 相当（得益于 Starlette 和 Pydantic）
2. **编码快**：开发速度提升 200-300%
3. **更少错误**：减少约 40% 的人为错误
4. **直观**：出色的编辑器支持，处处自动补全
5. **简单**：设计易于使用和学习
6. **简洁**：最小化代码重复
7. **健壮**：获得带有自动交互式文档的生产就绪代码
8. **基于标准**：基于 OpenAPI 和 JSON Schema

### 为什么 FastAPI 适合 ML 基础设施？

1. **性能**：对低延迟模型推理至关重要
2. **异步支持**：高效处理多个并发预测请求
3. **自动验证**：在将输入数据送入模型之前进行验证
4. **类型安全**：在开发时（而非生产时）捕获错误
5. **自动文档**：开箱即用的交互式 API 文档
6. **易于集成**：与 ML 框架配合良好（PyTorch、TensorFlow、sklearn）
7. **现代 Python**：使用最新的 Python 特性（类型提示、async/await）

### FastAPI vs Flask

| 特性 | FastAPI | Flask |
|---------|---------|-------|
| 性能 | 非常高（异步） | 中等（同步） |
| 类型提示 | 内置、必需 | 可选、通过扩展 |
| 验证 | 自动（Pydantic） | 手动或通过扩展 |
| 文档 | 自动生成 | 手动或通过扩展 |
| 异步支持 | 原生 | 通过扩展 |
| 学习曲线 | 中等 | 低 |
| 最适合 | 生产 API、ML 服务 | 原型、简单应用 |

## FastAPI 安装和设置

### 安装

```bash
# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # 在 Windows 上：venv\Scripts\activate

# 安装 FastAPI 和 ASGI 服务器
pip install fastapi
pip install "uvicorn[standard]"

# 用于开发
pip install "fastapi[all]"  # 包含所有可选依赖
```

**安装的内容：**
- `fastapi`：FastAPI 框架
- `uvicorn`：用于运行应用程序的 ASGI 服务器
- `pydantic`：使用 Python 类型提示进行数据验证
- `starlette`：FastAPI 使用的 Web 框架

### 项目结构

对于一个简单的 ML API：

```
ml-api/
├── app/
│   ├── __init__.py
│   ├── main.py           # FastAPI 应用程序
│   ├── models.py         # Pydantic 模型
│   ├── routers/          # API 路由
│   │   ├── __init__.py
│   │   ├── predictions.py
│   │   └── health.py
│   ├── services/         # 业务逻辑
│   │   ├── __init__.py
│   │   └── ml_model.py
│   └── dependencies.py   # 共享依赖
├── tests/
│   ├── __init__.py
│   └── test_api.py
├── models/               # ML 模型文件
│   └── model.pkl
├── requirements.txt
├── .env
└── README.md
```

## 你的第一个 FastAPI 应用程序

### Hello World

创建 `main.py`：

```python
from fastapi import FastAPI

# 创建 FastAPI 实例
app = FastAPI()

# 定义路径操作
@app.get("/")
def read_root():
    return {"message": "Hello World"}

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

### 运行应用程序

```bash
# 使用 uvicorn 运行
uvicorn main:app --reload

# 选项：
# --reload：代码更改时自动重新加载（仅开发环境）
# --host 0.0.0.0：允许从任何 IP 访问
# --port 8000：指定端口（默认：8000）
# --workers 4：工作进程数（生产环境）
```

输出：
```
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [12345] using watchfiles
INFO:     Started server process [12346]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

### 测试你的 API

**使用 curl：**
```bash
curl http://localhost:8000/
# {"message":"Hello World"}

curl http://localhost:8000/health
# {"status":"healthy"}
```

**使用 Python requests：**
```python
import requests

response = requests.get("http://localhost:8000/")
print(response.json())
# {'message': 'Hello World'}
```

### 交互式文档

FastAPI 自动生成交互式 API 文档：

- **Swagger UI**：http://localhost:8000/docs
- **ReDoc**：http://localhost:8000/redoc

你可以直接通过这些界面在浏览器中测试你的 API！

### OpenAPI Schema

FastAPI 自动生成 OpenAPI schema：

```bash
curl http://localhost:8000/openapi.json
```

这个 schema 可用于：
- 客户端代码生成
- API 测试工具
- 第三方集成
- 文档

## 路径操作和路由

### HTTP 方法

FastAPI 为所有 HTTP 方法提供了装饰器：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/items")
def read_items():
    return {"message": "Get all items"}

@app.post("/items")
def create_item():
    return {"message": "Create item"}

@app.put("/items/{item_id}")
def update_item(item_id: int):
    return {"message": f"Update item {item_id}"}

@app.patch("/items/{item_id}")
def partial_update_item(item_id: int):
    return {"message": f"Partial update item {item_id}"}

@app.delete("/items/{item_id}")
def delete_item(item_id: int):
    return {"message": f"Delete item {item_id}"}

@app.head("/items")
def items_head():
    return None  # 只返回头部

@app.options("/items")
def items_options():
    return {"methods": ["GET", "POST"]}
```

### 路径操作配置

为你的 endpoint 添加元数据：

```python
@app.get(
    "/models/{model_id}",
    summary="Get model details",
    description="Retrieve detailed information about a specific model",
    response_description="Model information including metadata and status",
    tags=["Models"],
    deprecated=False
)
def get_model(model_id: str):
    return {"model_id": model_id, "status": "ready"}
```

### 用于组织结构的标签

将相关 endpoint 分组：

```python
@app.get("/models", tags=["Models"])
def list_models():
    return {"models": []}

@app.post("/models", tags=["Models"])
def create_model():
    return {"model_id": "new-model"}

@app.get("/datasets", tags=["Datasets"])
def list_datasets():
    return {"datasets": []}

@app.post("/predictions", tags=["Inference"])
def predict():
    return {"prediction": 0.95}
```

标签在自动生成的文档中创建部分。

### 用于模块化的路由

对于大型应用程序，使用 `APIRouter`：

**routers/models.py：**
```python
from fastapi import APIRouter

router = APIRouter(
    prefix="/api/v1/models",
    tags=["Models"],
    responses={404: {"description": "Not found"}}
)

@router.get("/")
def list_models():
    return {"models": []}

@router.get("/{model_id}")
def get_model(model_id: str):
    return {"model_id": model_id}

@router.post("/")
def create_model():
    return {"model_id": "new-model"}
```

**main.py：**
```python
from fastapi import FastAPI
from routers import models

app = FastAPI()

# 包含路由
app.include_router(models.router)
```

现在你的 endpoint 可在以下地址访问：
- `GET /api/v1/models/`
- `GET /api/v1/models/{model_id}`
- `POST /api/v1/models/`

## 请求和响应模型

### Pydantic 模型

FastAPI 使用 **Pydantic** 进行数据验证和序列化。Pydantic 模型是定义数据形状的 Python 类。

```python
from pydantic import BaseModel, Field
from typing import Optional, List
from datetime import datetime

class Model(BaseModel):
    id: str
    name: str
    version: str = "1.0.0"
    framework: str = Field(..., description="ML framework (pytorch, tensorflow, etc.)")
    created_at: datetime
    tags: List[str] = []
    accuracy: Optional[float] = None

    class Config:
        # 用于文档的示例
        schema_extra = {
            "example": {
                "id": "bert-base",
                "name": "BERT Base Model",
                "version": "1.0.0",
                "framework": "pytorch",
                "created_at": "2024-10-18T14:30:00Z",
                "tags": ["nlp", "transformer"],
                "accuracy": 0.92
            }
        }
```

### 字段验证

```python
from pydantic import BaseModel, Field, validator
from typing import Literal

class TrainingConfig(BaseModel):
    batch_size: int = Field(gt=0, le=128, description="Batch size (1-128)")
    learning_rate: float = Field(gt=0, le=1, description="Learning rate (0-1)")
    epochs: int = Field(ge=1, le=1000, description="Number of epochs")
    optimizer: Literal["adam", "sgd", "rmsprop"] = "adam"

    @validator("learning_rate")
    def validate_lr(cls, v):
        if v > 0.1:
            raise ValueError("Learning rate above 0.1 is usually too high")
        return v

    @validator("batch_size")
    def validate_batch_size(cls, v):
        # 必须是 2 的幂
        if v & (v - 1) != 0:
            raise ValueError("Batch size should be power of 2 for optimal performance")
        return v
```

### 在路径操作中使用模型

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class PredictionRequest(BaseModel):
    text: str
    max_length: int = 512

class PredictionResponse(BaseModel):
    prediction: str
    confidence: float
    processing_time_ms: int

@app.post("/predict", response_model=PredictionResponse)
def predict(request: PredictionRequest):
    # FastAPI 自动：
    # 1. 根据 PredictionRequest 验证请求体
    # 2. 将 JSON 转换为 PredictionRequest 对象
    # 3. 根据 PredictionResponse 验证响应
    # 4. 将 PredictionResponse 转换为 JSON

    return PredictionResponse(
        prediction="positive",
        confidence=0.95,
        processing_time_ms=45
    )
```

## 路径参数和查询参数

### 路径参数

```python
@app.get("/models/{model_id}")
def get_model(model_id: str):
    return {"model_id": model_id}

# 带类型验证
@app.get("/models/{model_id}/versions/{version_num}")
def get_model_version(model_id: str, version_num: int):
    return {"model_id": model_id, "version": version_num}

# 使用 Enum 限制值
from enum import Enum

class ModelFramework(str, Enum):
    pytorch = "pytorch"
    tensorflow = "tensorflow"
    sklearn = "sklearn"

@app.get("/models/framework/{framework}")
def get_models_by_framework(framework: ModelFramework):
    return {"framework": framework, "models": []}
```

### 查询参数

```python
from typing import Optional, List

# 可选查询参数
@app.get("/models")
def list_models(skip: int = 0, limit: int = 10):
    # GET /models?skip=0&limit=10
    return {"skip": skip, "limit": limit}

# 必填查询参数（无默认值）
@app.get("/search")
def search_models(q: str):
    # GET /search?q=bert
    # GET /search  → 422 错误（缺少必填参数）
    return {"query": q}

# 可选带 None 默认值
@app.get("/models")
def list_models(status: Optional[str] = None):
    # GET /models
    # GET /models?status=ready
    if status:
        return {"filter": status}
    return {"message": "All models"}

# 多值（List）
@app.get("/models")
def list_models(tags: List[str] = []):
    # GET /models?tags=nlp&tags=transformer
    return {"tags": tags}

# 布尔查询参数
@app.get("/models")
def list_models(include_archived: bool = False):
    # GET /models?include_archived=true
    # GET /models?include_archived=1
    # GET /models?include_archived=yes
    # 全部转换为 True
    return {"include_archived": include_archived}
```

### 结合路径和查询参数

```python
@app.get("/models/{model_id}")
def get_model(
    model_id: str,
    include_metrics: bool = False,
    version: Optional[str] = None
):
    # GET /models/bert-base
    # GET /models/bert-base?include_metrics=true
    # GET /models/bert-base?include_metrics=true&version=2.0.0

    result = {"model_id": model_id}
    if include_metrics:
        result["metrics"] = {"accuracy": 0.92}
    if version:
        result["version"] = version
    return result
```

## 请求体和验证

### 简单请求体

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None

@app.post("/items")
def create_item(item: Item):
    # FastAPI 验证请求体
    total = item.price + (item.tax or 0)
    return {"item": item, "total": total}
```

### 嵌套模型

```python
class DatasetInfo(BaseModel):
    name: str
    size: int
    path: str

class TrainingJobConfig(BaseModel):
    model_type: str
    dataset: DatasetInfo
    batch_size: int
    epochs: int

@app.post("/training-jobs")
def create_training_job(config: TrainingJobConfig):
    return {
        "status": "created",
        "model": config.model_type,
        "dataset": config.dataset.name
    }

# 请求体：
# {
#   "model_type": "bert",
#   "dataset": {
#     "name": "imdb",
#     "size": 50000,
#     "path": "/data/imdb"
#   },
#   "batch_size": 32,
#   "epochs": 10
# }
```

### 列表和字典

```python
from typing import List, Dict

class BatchPredictionRequest(BaseModel):
    inputs: List[str]
    model_id: str

@app.post("/batch-predict")
def batch_predict(request: BatchPredictionRequest):
    predictions = [f"pred_{i}" for i in range(len(request.inputs))]
    return {"predictions": predictions}

# 请求：
# {
#   "model_id": "bert-base",
#   "inputs": ["text 1", "text 2", "text 3"]
# }

class FlexibleConfig(BaseModel):
    parameters: Dict[str, float]

@app.post("/configure")
def configure(config: FlexibleConfig):
    return config

# 请求：
# {
#   "parameters": {
#     "learning_rate": 0.001,
#     "dropout": 0.1,
#     "temperature": 0.7
#   }
# }
```

### 高级验证

```python
from pydantic import BaseModel, Field, validator, root_validator

class PredictionInput(BaseModel):
    text: str = Field(..., min_length=1, max_length=10000)
    language: str = Field("en", regex="^[a-z]{2}$")  # 双字母语言代码
    confidence_threshold: float = Field(0.5, ge=0.0, le=1.0)

    @validator("text")
    def text_must_not_be_only_whitespace(cls, v):
        if not v.strip():
            raise ValueError("Text cannot be only whitespace")
        return v.strip()

    @root_validator
    def check_language_compatibility(cls, values):
        text = values.get("text")
        language = values.get("language")

        # 示例：检查文本是否匹配语言
        # 实际中，你会使用语言检测库

        return values

@app.post("/predict")
def predict(input_data: PredictionInput):
    return {"status": "validated", "text_length": len(input_data.text)}
```

## 响应模型和状态码

### 响应模型

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ModelResponse(BaseModel):
    id: str
    name: str
    status: str

@app.get("/models/{model_id}", response_model=ModelResponse)
def get_model(model_id: str):
    # 即使你返回额外字段，只有 ModelResponse 中的字段会出现在响应中
    return {
        "id": model_id,
        "name": "BERT Model",
        "status": "ready",
        "internal_field": "won't be shown"  # 被过滤掉
    }
```

### 排除字段

```python
class UserInDB(BaseModel):
    username: str
    email: str
    full_name: str
    hashed_password: str  # 敏感信息！

class UserOut(BaseModel):
    username: str
    email: str
    full_name: str

@app.get("/users/{user_id}", response_model=UserOut)
def get_user(user_id: str):
    # 从数据库获取
    user_in_db = {
        "username": "alice",
        "email": "alice@example.com",
        "full_name": "Alice Smith",
        "hashed_password": "$2b$12$..."
    }
    # hashed_password 不会在响应中出现
    return user_in_db
```

### 响应状态码

```python
from fastapi import FastAPI, status

app = FastAPI()

@app.post("/models", status_code=status.HTTP_201_CREATED)
def create_model():
    return {"id": "new-model"}

@app.delete("/models/{model_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_model(model_id: str):
    # 204 响应不返回请求体
    return None

@app.get("/models/{model_id}", status_code=status.HTTP_200_OK)
def get_model(model_id: str):
    return {"id": model_id}
```

### 多个响应模型

```python
from typing import Union
from fastapi import Response

class SuccessResponse(BaseModel):
    data: dict
    message: str

class ErrorResponse(BaseModel):
    error: str
    details: str

@app.get(
    "/models/{model_id}",
    responses={
        200: {"model": SuccessResponse, "description": "Model found"},
        404: {"model": ErrorResponse, "description": "Model not found"}
    }
)
def get_model(model_id: str, response: Response):
    if model_id == "unknown":
        response.status_code = 404
        return ErrorResponse(error="Not Found", details=f"Model {model_id} doesn't exist")

    return SuccessResponse(data={"id": model_id}, message="Success")
```

## 依赖注入

依赖注入是 FastAPI 的一个强大特性，允许你：
- 跨 endpoint 共享逻辑
- 管理数据库连接
- 处理认证
- 验证权限
- 减少代码重复

### 基本依赖

```python
from fastapi import Depends

def get_current_user(token: str):
    # 实际中，解码并验证令牌
    if token == "valid-token":
        return {"username": "alice"}
    raise HTTPException(status_code=401, detail="Invalid token")

@app.get("/protected")
def protected_route(user: dict = Depends(get_current_user)):
    return {"message": f"Hello {user['username']}"}
```

### 数据库连接依赖

```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

@app.get("/models")
def list_models(db: Session = Depends(get_db)):
    models = db.query(ModelTable).all()
    return models
```

### 层级依赖

```python
from fastapi import Header, HTTPException

def verify_token(x_token: str = Header(...)):
    if x_token != "secret-token":
        raise HTTPException(status_code=401, detail="Invalid token")
    return x_token

def get_current_user(token: str = Depends(verify_token)):
    # 令牌已验证通过
    return {"username": "alice", "token": token}

def get_admin_user(user: dict = Depends(get_current_user)):
    if user["username"] != "admin":
        raise HTTPException(status_code=403, detail="Not an admin")
    return user

@app.get("/admin/dashboard")
def admin_dashboard(admin: dict = Depends(get_admin_user)):
    # 只有管理员可以访问
    return {"message": f"Welcome admin {admin['username']}"}
```

### 基于类的依赖

```python
from typing import Optional

class CommonQueryParams:
    def __init__(self, skip: int = 0, limit: int = 10, q: Optional[str] = None):
        self.skip = skip
        self.limit = limit
        self.q = q

@app.get("/items")
def list_items(commons: CommonQueryParams = Depends()):
    return {"skip": commons.skip, "limit": commons.limit, "q": commons.q}

# 用法：
# GET /items?skip=10&limit=20&q=search
```

### ML 模型加载依赖

```python
import pickle
from typing import Optional

# 全局模型缓存
_model_cache = {}

def get_ml_model(model_id: str):
    """加载和缓存 ML 模型的依赖"""
    if model_id not in _model_cache:
        # 加载模型（昂贵操作）
        with open(f"models/{model_id}.pkl", "rb") as f:
            _model_cache[model_id] = pickle.load(f)
    return _model_cache[model_id]

@app.post("/predict/{model_id}")
def predict(
    model_id: str,
    input_data: PredictionInput,
    model = Depends(get_ml_model)
):
    # 模型已加载并缓存
    prediction = model.predict([input_data.features])
    return {"prediction": prediction[0]}
```

## 错误处理和自定义异常

### HTTPException

```python
from fastapi import HTTPException, status

@app.get("/models/{model_id}")
def get_model(model_id: str):
    # 模拟数据库查找
    if model_id not in ["bert-base", "gpt-2"]:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"Model '{model_id}' not found",
            headers={"X-Error": "Model-Not-Found"}
        )

    return {"model_id": model_id, "status": "ready"}
```

### 自定义异常类

```python
class ModelNotFoundError(Exception):
    def __init__(self, model_id: str):
        self.model_id = model_id

class InvalidModelError(Exception):
    def __init__(self, model_id: str, reason: str):
        self.model_id = model_id
        self.reason = reason

# 异常处理器
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(ModelNotFoundError)
async def model_not_found_handler(request: Request, exc: ModelNotFoundError):
    return JSONResponse(
        status_code=404,
        content={
            "error": "Model not found",
            "model_id": exc.model_id,
            "suggestion": "Check /api/models for available models"
        }
    )

@app.exception_handler(InvalidModelError)
async def invalid_model_handler(request: Request, exc: InvalidModelError):
    return JSONResponse(
        status_code=422,
        content={
            "error": "Invalid model",
            "model_id": exc.model_id,
            "reason": exc.reason
        }
    )

# 使用自定义异常
@app.get("/models/{model_id}")
def get_model(model_id: str):
    if model_id == "unknown":
        raise ModelNotFoundError(model_id)
    if model_id == "corrupted":
        raise InvalidModelError(model_id, "Model file is corrupted")
    return {"model_id": model_id}
```

### 全局异常处理器

```python
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    # 记录异常
    logger.error(f"Unhandled exception: {exc}", exc_info=True)

    return JSONResponse(
        status_code=500,
        content={
            "error": "Internal server error",
            "message": "An unexpected error occurred",
            "request_id": str(uuid.uuid4())
        }
    )
```

## 后台任务

对于不需要阻塞响应的操作：

```python
from fastapi import BackgroundTasks

def log_prediction(model_id: str, input_data: dict, prediction: dict):
    """记录预测到数据库或文件"""
    # 模拟记录
    with open("predictions.log", "a") as f:
        f.write(f"{model_id}: {input_data} -> {prediction}\n")

@app.post("/predict")
def predict(
    input_data: PredictionInput,
    background_tasks: BackgroundTasks
):
    # 进行预测
    prediction = {"result": "positive", "confidence": 0.95}

    # 在发送响应后添加要运行的任务
    background_tasks.add_task(
        log_prediction,
        model_id="bert-base",
        input_data=input_data.dict(),
        prediction=prediction
    )

    # 立即返回
    return prediction
```

### 多个后台任务

```python
def send_notification(email: str, message: str):
    # 发送邮件通知
    pass

def update_metrics(model_id: str):
    # 更新预测计数
    pass

@app.post("/predict")
def predict(
    input_data: PredictionInput,
    background_tasks: BackgroundTasks
):
    prediction = {"result": "positive"}

    background_tasks.add_task(log_prediction, "bert", input_data, prediction)
    background_tasks.add_task(send_notification, "admin@example.com", "Prediction made")
    background_tasks.add_task(update_metrics, "bert-base")

    return prediction
```

## 使用 FastAPI 构建 ML API

### 完整的模型服务示例

```python
from fastapi import FastAPI, HTTPException, Depends
from pydantic import BaseModel, Field
from typing import List, Optional
import pickle
import numpy as np
from datetime import datetime

app = FastAPI(title="ML Model API", version="1.0.0")

# 模型
class PredictionInput(BaseModel):
    features: List[float] = Field(..., min_items=4, max_items=4)

    class Config:
        schema_extra = {
            "example": {
                "features": [5.1, 3.5, 1.4, 0.2]
            }
        }

class PredictionOutput(BaseModel):
    prediction: str
    confidence: float
    probabilities: dict
    model_version: str
    timestamp: datetime

class ModelInfo(BaseModel):
    name: str
    version: str
    framework: str
    features: int
    classes: List[str]
    accuracy: float

# 全局模型存储
_models = {}

# 启动事件加载模型
@app.on_event("startup")
async def load_models():
    """在应用程序启动时加载 ML 模型"""
    try:
        with open("models/iris_classifier.pkl", "rb") as f:
            _models["iris"] = pickle.load(f)
        print("Models loaded successfully")
    except Exception as e:
        print(f"Error loading models: {e}")

# 获取模型的依赖
def get_model():
    if "iris" not in _models:
        raise HTTPException(status_code=503, detail="Model not loaded")
    return _models["iris"]

# 健康检查
@app.get("/health")
def health_check():
    return {
        "status": "healthy",
        "models_loaded": len(_models),
        "timestamp": datetime.utcnow()
    }

# 模型信息
@app.get("/models/iris", response_model=ModelInfo)
def get_model_info():
    return ModelInfo(
        name="Iris Classifier",
        version="1.0.0",
        framework="scikit-learn",
        features=4,
        classes=["setosa", "versicolor", "virginica"],
        accuracy=0.96
    )

# 预测 endpoint
@app.post("/predict", response_model=PredictionOutput)
def predict(
    input_data: PredictionInput,
    model = Depends(get_model)
):
    try:
        # 准备输入
        features = np.array([input_data.features])

        # 进行预测
        prediction = model.predict(features)[0]
        probabilities = model.predict_proba(features)[0]

        # 获取类别名称
        classes = ["setosa", "versicolor", "virginica"]

        return PredictionOutput(
            prediction=classes[prediction],
            confidence=float(max(probabilities)),
            probabilities={cls: float(prob) for cls, prob in zip(classes, probabilities)},
            model_version="1.0.0",
            timestamp=datetime.utcnow()
        )

    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Prediction failed: {str(e)}"
        )

# 批量预测
@app.post("/batch-predict")
def batch_predict(
    inputs: List[PredictionInput],
    model = Depends(get_model)
):
    results = []
    for input_data in inputs:
        features = np.array([input_data.features])
        prediction = model.predict(features)[0]
        results.append({
            "features": input_data.features,
            "prediction": ["setosa", "versicolor", "virginica"][prediction]
        })

    return {"predictions": results, "count": len(results)}
```

## 测试 FastAPI 应用程序

### 使用 TestClient

```python
from fastapi.testclient import TestClient
from main import app

client = TestClient(app)

def test_read_root():
    response = client.get("/")
    assert response.status_code == 200
    assert response.json() == {"message": "Hello World"}

def test_create_item():
    response = client.post(
        "/items",
        json={"name": "Test Item", "price": 10.0}
    )
    assert response.status_code == 200
    assert "item" in response.json()

def test_invalid_item():
    response = client.post(
        "/items",
        json={"name": "Test", "price": "invalid"}  # 错误类型
    )
    assert response.status_code == 422  # 验证错误

def test_prediction():
    response = client.post(
        "/predict",
        json={"features": [5.1, 3.5, 1.4, 0.2]}
    )
    assert response.status_code == 200
    data = response.json()
    assert "prediction" in data
    assert "confidence" in data
```

### Pytest Fixture

```python
import pytest
from fastapi.testclient import TestClient

@pytest.fixture
def client():
    from main import app
    return TestClient(app)

@pytest.fixture
def auth_headers():
    return {"Authorization": "Bearer test-token"}

def test_protected_endpoint(client, auth_headers):
    response = client.get("/protected", headers=auth_headers)
    assert response.status_code == 200
```

## 最佳实践

### 1. 项目结构

使用路由组织代码：
```
app/
├── main.py
├── routers/
│   ├── models.py
│   ├── predictions.py
│   └── health.py
├── models/
│   ├── schemas.py
│   └── ml_models.py
├── services/
│   ├── model_service.py
│   └── prediction_service.py
└── dependencies.py
```

### 2. 配置管理

```python
from pydantic import BaseSettings

class Settings(BaseSettings):
    app_name: str = "ML API"
    model_path: str = "/models"
    max_batch_size: int = 100
    redis_url: str = "redis://localhost"

    class Config:
        env_file = ".env"

settings = Settings()
```

### 3. 日志

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@app.post("/predict")
def predict(input_data: PredictionInput):
    logger.info(f"Prediction request received: {input_data}")
    try:
        result = model.predict(input_data.features)
        logger.info(f"Prediction successful: {result}")
        return result
    except Exception as e:
        logger.error(f"Prediction failed: {e}", exc_info=True)
        raise
```

### 4. CORS 配置

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],  # 前端 URL
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 5. 请求验证

始终验证输入：
```python
class PredictionInput(BaseModel):
    text: str = Field(..., min_length=1, max_length=5000)

    @validator("text")
    def validate_text(cls, v):
        if not v.strip():
            raise ValueError("Text cannot be empty")
        return v.strip()
```

### 6. 异步 Endpoint

用于 I/O 密集型操作：
```python
import aiofiles

@app.get("/logs")
async def get_logs():
    async with aiofiles.open("app.log", mode="r") as f:
        content = await f.read()
    return {"logs": content}
```

### 7. 速率限制

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter

@app.post("/predict")
@limiter.limit("10/minute")
def predict(request: Request, input_data: PredictionInput):
    return {"prediction": "result"}
```

## 总结和关键要点

### 我们学到了什么

1. **FastAPI 基础**
   - 高性能异步框架
   - 自动验证和文档
   - 类型提示提高开发速度和安全性

2. **核心特性**
   - 使用 HTTP 方法的路径操作
   - 用于请求/响应验证的 Pydantic 模型
   - 带类型检查的路径和查询参数
   - 用于清晰代码架构的依赖注入

3. **高级主题**
   - 使用自定义异常进行错误处理
   - 用于异步操作的后台任务
   - 具有适当结构的模型服务
   - 使用 TestClient 进行测试

4. **ML 特定模式**
   - 模型加载和缓存
   - 批量预测 endpoint
   - 健康检查和监控
   - 生产环境的正确错误处理

### 下一步

在下一讲中，我们将涵盖 **认证和安全**，包括：
- API key 认证
- JWT 令牌
- OAuth2
- 速率限制
- 输入清理
- HTTPS 和安全头部

### 练习

构建一个完整的模型服务 API：
1. 加载 scikit-learn 模型
2. 创建预测 endpoint
3. 添加健康检查
4. 实现错误处理
5. 编写测试
6. 添加文档

---

**预计学习时间**：4-5 小时
**动手练习**：完成练习 01 和 02
**评估**：测验将涵盖 FastAPI 概念和实际实现
