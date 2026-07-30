# 练习 01：构建基本 REST API

## 概述

在此练习中，你将使用 FastAPI 构建一个用于管理 ML 模型元数据的基本 REST API。本练习强化了 REST 架构、HTTP 方法、状态码和 FastAPI 基础的核心概念。

**难度：** 初级
**预计时间：** 2-3 小时
**先决条件：**
- 讲座 01：API 基础和 REST
- 讲座 02：FastAPI 框架
- 已安装 Python 3.7+
- 基本 Python 知识

## 学习目标

完成此练习后，你将：
- 从头开始设置 FastAPI 项目
- 按照 REST 原则实现 CRUD 操作
- 使用适当的 HTTP 方法和状态码
- 为请求/响应验证定义 Pydantic 模型
- 优雅地处理错误
- 使用交互式文档测试你的 API
- 理解 RESTful 资源设计

## 场景

你在构建一个简单的模型注册表 API，允许用户：
- 列出所有已注册的 ML 模型
- 获取特定模型的详细信息
- 注册新模型
- 更新模型信息
- 删除模型
- 按框架或状态搜索/过滤模型

## 项目设置

### 1. 创建项目目录

```bash
mkdir ml-model-registry
cd ml-model-registry
```

### 2. 创建虚拟环境

```bash
python -m venv venv
source venv/bin/activate  # 在 Windows 上：venv\Scripts\activate
```

### 3. 安装依赖

```bash
pip install fastapi uvicorn pydantic
pip install pytest httpx  # 用于测试
```

### 4. 创建项目结构

```
ml-model-registry/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models.py
│   └── database.py
├── tests/
│   ├── __init__.py
│   └── test_api.py
├── requirements.txt
└── README.md
```

## 第一部分：定义数据模型

**文件：app/models.py**

创建 Pydantic 模型来表示你的数据：

```python
from pydantic import BaseModel, Field
from typing import Optional, List, Literal
from datetime import datetime
from enum import Enum

# TODO: 定义 ModelFramework 枚举
# 提示：应包括 pytorch、tensorflow、sklearn、xgboost

# TODO: 定义 ModelStatus 枚举
# 提示：应包括 training、ready、deployed、archived

# TODO: 创建 Model 类 (BaseModel)
# 必填字段：
#   - id: str (唯一标识符)
#   - name: str (模型名称)
#   - framework: ModelFramework
#   - version: str (语义版本，默认 "1.0.0")
#   - status: ModelStatus (默认 "ready")
#   - created_at: datetime
#   - updated_at: datetime
# 可选字段：
#   - description: Optional[str]
#   - accuracy: Optional[float] (范围 0-1)
#   - tags: List[str] (默认空列表)
#   - model_size_mb: Optional[float]

# TODO: 创建 ModelCreate 类 (BaseModel)
# 这是用户创建模型时发送的数据
# 不应包含：id、created_at、updated_at (服务器生成)

# TODO: 创建 ModelUpdate 类 (BaseModel)
# 所有字段应为可选 (用于部分更新)
# 不应包含：id、created_at (这些不能更改)

# TODO: 添加 Config 类，提供文档示例
```

**你将学到的：**
- 如何使用 Pydantic 进行数据验证
- 请求模型和响应模型的区别
- 使用 Enum 限定值
- 带约束的字段验证

## 第二部分：实现内存数据库

**文件：app/database.py**

实现一个简单的内存数据库（Python 字典）：

```python
from typing import Dict, List, Optional
from app.models import Model
from datetime import datetime
import uuid

# TODO: 创建一个全局字典来存储模型
# 键：model_id (str)
# 值：Model 对象

class ModelDB:
    """模型的内存数据库"""

    def __init__(self):
        # TODO: 初始化空模型字典
        pass

    def create_model(self, model_data: dict) -> Model:
        """创建新模型"""
        # TODO: 使用 uuid.uuid4() 生成唯一 ID
        # TODO: 添加 created_at 和 updated_at 时间戳
        # TODO: 创建 Model 对象
        # TODO: 存储在字典中
        # TODO: 返回创建的模型
        pass

    def get_model(self, model_id: str) -> Optional[Model]:
        """按 ID 获取模型"""
        # TODO: 如果存在返回模型，否则返回 None
        pass

    def get_all_models(self,
                       skip: int = 0,
                       limit: int = 100,
                       framework: Optional[str] = None,
                       status: Optional[str] = None) -> List[Model]:
        """获取所有模型，带有可选过滤"""
        # TODO: 从字典中获取所有模型
        # TODO: 如果提供了 framework 则过滤
        # TODO: 如果提供了 status 则过滤
        # TODO: 应用分页 (skip, limit)
        # TODO: 返回模型列表
        pass

    def update_model(self, model_id: str, update_data: dict) -> Optional[Model]:
        """更新模型"""
        # TODO: 检查模型是否存在
        # TODO: 更新提供的字段
        # TODO: 更新 updated_at 时间戳
        # TODO: 返回更新后的模型
        pass

    def delete_model(self, model_id: str) -> bool:
        """删除模型"""
        # TODO: 检查模型是否存在
        # TODO: 从字典中删除
        # TODO: 如果删除返回 True，如果未找到返回 False
        pass

    def model_exists(self, model_id: str) -> bool:
        """检查模型是否存在"""
        # TODO: 如果 model_id 在字典中返回 True
        pass

# 创建全局数据库实例
db = ModelDB()
```

**你将学到的：**
- CRUD 操作实现
- 数据过滤和分页
- 处理可选参数

## 第三部分：创建 API Endpoint

**文件：app/main.py**

实现 REST API endpoint：

```python
from fastapi import FastAPI, HTTPException, status, Query
from typing import List, Optional
from app.models import Model, ModelCreate, ModelUpdate
from app.database import db

# TODO: 创建 FastAPI 应用实例，带标题和版本
app = FastAPI(
    title="ML Model Registry API",
    version="1.0.0",
    description="API for managing ML model metadata"
)

# TODO: 实现 GET / endpoint
# 返回欢迎消息和 API 信息
@app.get("/")
def root():
    pass

# TODO: 实现 GET /health endpoint
# 返回 API 的健康状态
@app.get("/health")
def health_check():
    pass

# TODO: 实现 GET /api/v1/models endpoint
# 参数：skip (默认 0)、limit (默认 100)、framework (可选)、status (可选)
# 返回：模型列表
# 响应模型：List[Model]
@app.get("/api/v1/models", response_model=List[Model])
def list_models(
    skip: int = Query(0, ge=0, description="Number of records to skip"),
    limit: int = Query(100, ge=1, le=1000, description="Maximum number of records to return"),
    framework: Optional[str] = Query(None, description="Filter by framework"),
    status: Optional[str] = Query(None, description="Filter by status")
):
    # TODO: 使用参数调用 db.get_all_models()
    # TODO: 返回结果
    pass

# TODO: 实现 POST /api/v1/models endpoint
# 请求体：ModelCreate
# 返回：创建的模型，状态 201
# 响应模型：Model
@app.post("/api/v1/models", response_model=Model, status_code=status.HTTP_201_CREATED)
def create_model(model: ModelCreate):
    # TODO: 使用模型数据调用 db.create_model()
    # TODO: 返回创建的模型
    pass

# TODO: 实现 GET /api/v1/models/{model_id} endpoint
# 返回：模型详情
# 响应模型：Model
# 错误：404 如果未找到
@app.get("/api/v1/models/{model_id}", response_model=Model)
def get_model(model_id: str):
    # TODO: 调用 db.get_model()
    # TODO: 如果未找到，抛出 HTTPException 404
    # TODO: 返回模型
    pass

# TODO: 实现 PUT /api/v1/models/{model_id} endpoint
# 请求体：ModelUpdate
# 返回：更新后的模型
# 响应模型：Model
# 错误：404 如果未找到
@app.put("/api/v1/models/{model_id}", response_model=Model)
def update_model(model_id: str, model_update: ModelUpdate):
    # TODO: 使用 db.model_exists() 检查模型是否存在
    # TODO: 如果未找到，抛出 HTTPException 404
    # TODO: 使用 model_update 中非空的字段调用 db.update_model()
    # TODO: 返回更新后的模型
    pass

# TODO: 实现 DELETE /api/v1/models/{model_id} endpoint
# 返回：204 No Content 成功
# 错误：404 如果未找到
@app.delete("/api/v1/models/{model_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_model(model_id: str):
    # TODO: 调用 db.delete_model()
    # TODO: 如果未找到，抛出 HTTPException 404
    # TODO: 返回 None (204 无响应体)
    pass

# TODO: 实现 GET /api/v1/models/search endpoint
# 查询参数：q (搜索词)
# 搜索模型名称、描述和标签
@app.get("/api/v1/models/search", response_model=List[Model])
def search_models(q: str = Query(..., min_length=1, description="Search query")):
    # TODO: 获取所有模型
    # TODO: 过滤出 q 出现在名称、描述或标签中的模型
    # TODO: 返回匹配的模型
    pass
```

**你将学到的：**
- 使用适当 HTTP 方法定义路径操作
- 使用路径参数和查询参数
- 使用 Pydantic 进行请求体验证
- 设置适当的状态码
- 使用 HTTPException 进行错误处理
- 响应模型规范

## 第四部分：运行和测试你的 API

### 1. 运行服务器

```bash
uvicorn app.main:app --reload
```

预期输出：
```
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [12345] using watchfiles
```

### 2. 访问交互式文档

打开浏览器并访问：
- Swagger UI：http://127.0.0.1:8000/docs
- ReDoc：http://127.0.0.1:8000/redoc

### 3. 使用 Swagger UI 测试 Endpoint

**创建模型：**
```json
POST /api/v1/models
{
  "name": "BERT Sentiment Classifier",
  "framework": "pytorch",
  "description": "Fine-tuned BERT for sentiment analysis",
  "version": "1.0.0",
  "accuracy": 0.94,
  "tags": ["nlp", "sentiment", "transformer"],
  "model_size_mb": 438.5
}
```

**列出模型：**
```
GET /api/v1/models
GET /api/v1/models?framework=pytorch
GET /api/v1/models?status=ready&limit=10
```

**获取特定模型：**
```
GET /api/v1/models/{model_id}
```

**更新模型：**
```json
PUT /api/v1/models/{model_id}
{
  "status": "deployed",
  "accuracy": 0.95
}
```

**删除模型：**
```
DELETE /api/v1/models/{model_id}
```

**搜索模型：**
```
GET /api/v1/models/search?q=bert
```

### 4. 使用 curl 测试

```bash
# 创建模型
curl -X POST "http://localhost:8000/api/v1/models" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "GPT-2 Text Generator",
    "framework": "pytorch",
    "version": "1.0.0",
    "tags": ["nlp", "generation"]
  }'

# 列出模型
curl "http://localhost:8000/api/v1/models"

# 获取特定模型
curl "http://localhost:8000/api/v1/models/{model_id}"

# 更新模型
curl -X PUT "http://localhost:8000/api/v1/models/{model_id}" \
  -H "Content-Type: application/json" \
  -d '{"status": "deployed"}'

# 删除模型
curl -X DELETE "http://localhost:8000/api/v1/models/{model_id}"
```

## 第五部分：编写测试

**文件：tests/test_api.py**

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_root():
    """测试根 endpoint"""
    # TODO: 向 / 发起 GET 请求
    # TODO: 断言状态码是 200
    # TODO: 断言响应包含预期消息
    pass

def test_health_check():
    """测试健康 endpoint"""
    # TODO: 向 /health 发起 GET 请求
    # TODO: 断言状态码是 200
    # TODO: 断言状态为 healthy
    pass

def test_create_model():
    """测试创建模型"""
    model_data = {
        "name": "Test Model",
        "framework": "pytorch",
        "version": "1.0.0",
        "tags": ["test"]
    }

    # TODO: 向 /api/v1/models 发起 POST 请求
    # TODO: 断言状态码是 201
    # TODO: 断言响应包含 id
    # TODO: 断言名称匹配
    pass

def test_get_model():
    """测试获取特定模型"""
    # TODO: 首先创建一个模型
    # TODO: 从响应中提取 model_id
    # TODO: 向 /api/v1/models/{model_id} 发起 GET 请求
    # TODO: 断言状态码是 200
    # TODO: 断言模型数据匹配
    pass

def test_get_nonexistent_model():
    """测试获取不存在的模型"""
    # TODO: 使用无效 ID 发起 GET 请求
    # TODO: 断言状态码是 404
    pass

def test_list_models():
    """测试列出模型"""
    # TODO: 创建 2-3 个模型
    # TODO: 向 /api/v1/models 发起 GET 请求
    # TODO: 断言状态码是 200
    # TODO: 断言响应是列表
    # TODO: 断言列表包含创建的模型
    pass

def test_update_model():
    """测试更新模型"""
    # TODO: 创建一个模型
    # TODO: 使用新数据更新模型
    # TODO: 断言状态码是 200
    # TODO: 断言 updated_at 已更改
    # TODO: 断言新值已反映
    pass

def test_delete_model():
    """测试删除模型"""
    # TODO: 创建一个模型
    # TODO: 删除模型
    # TODO: 断言状态码是 204
    # TODO: 再次尝试获取模型
    # TODO: 断言返回 404
    pass

def test_filter_by_framework():
    """测试按框架过滤模型"""
    # TODO: 创建不同框架的模型
    # TODO: 按框架过滤
    # TODO: 断言只返回匹配的模型
    pass

def test_search_models():
    """测试搜索功能"""
    # TODO: 创建具有特定名称/标签的模型
    # TODO: 搜索某个词
    # TODO: 断言返回匹配的模型
    pass
```

**运行测试：**
```bash
pytest tests/test_api.py -v
```

## 挑战和扩展

### 挑战 1：添加分页元数据

增强列表 endpoint 以返回分页元数据：

```json
{
  "items": [...],
  "total": 100,
  "page": 1,
  "size": 10,
  "pages": 10
}
```

### 挑战 2：添加排序

允许按不同字段排序：
```
GET /api/v1/models?sort_by=created_at&order=desc
```

### 挑战 3：添加模型统计 Endpoint

```
GET /api/v1/models/stats
{
  "total_models": 42,
  "by_framework": {
    "pytorch": 20,
    "tensorflow": 15,
    "sklearn": 7
  },
  "by_status": {
    "ready": 30,
    "deployed": 10,
    "archived": 2
  }
}
```

### 挑战 4：添加验证

- 模型名称必须唯一
- 版本必须遵循语义版本控制 (X.Y.Z)
- 标签应为小写且不含空格
- 准确度必须在 0 到 1 之间

### 挑战 5：添加批量操作

```python
POST /api/v1/models/batch
{
  "models": [
    {...},
    {...}
  ]
}
```

## 常见问题及解决方案

### 问题 1：导入错误

**问题：** `ModuleNotFoundError: No module named 'app'`

**解决方案：**
- 确保在项目根目录中
- 使用运行：`uvicorn app.main:app --reload`
- 或添加 `__init__.py` 文件使目录成为包

### 问题 2：Pydantic 验证错误

**问题：** 创建模型时出现 `ValidationError`

**解决方案：**
- 检查是否提供了所有必填字段
- 确保数据类型匹配（例如，accuracy 为 float 而非 string）
- 使用 `.dict()` 或 `.model_dump()` 将 Pydantic 模型转换为字典

### 问题 3：422 Unprocessable Entity

**问题：** 发送请求时 API 返回 422

**解决方案：**
- 检查请求体是否与预期的 Pydantic 模型匹配
- 确保 JSON 有效
- 检查枚举值是否正确

## 交付物

到本练习结束时，你应该拥有：

1. ✅ 带有 CRUD endpoint 的工作 FastAPI 应用程序
2. ✅ 用于请求/响应验证的 Pydantic 模型
3. ✅ 内存数据库实现
4. ✅ 所有响应的适当 HTTP 状态码
5. ✅ 无效请求的错误处理
6. ✅ 交互式 API 文档 (Swagger UI)
7. ✅ 90%+ 覆盖率的测试套件
8. ✅ 带有设置和使用说明的 README

## 提交清单

- [ ] 所有 endpoint 已实现并正常工作
- [ ] 所有测试通过 (`pytest`)
- [ ] 交互式文档可在 `/docs` 访问
- [ ] 代码遵循 Python 风格指南（使用 `black` 格式化）
- [ ] 无硬编码值（使用环境变量或配置）
- [ ] README.md 包含：
  - [ ] 设置说明
  - [ ] API endpoint 文档
  - [ ] 示例请求和响应
  - [ ] 如何运行测试

## 关键要点

完成此练习后，你应该理解：

1. **REST 原则：** 基于资源的 URL、HTTP 方法、状态码
2. **FastAPI 基础：** 路径操作、Pydantic 模型、自动验证
3. **CRUD 操作：** 创建、读取、更新、删除模式
4. **错误处理：** 适当的 HTTP 异常和状态码
5. **API 测试：** 使用 TestClient 进行自动化测试
6. **文档：** 自动生成的交互式文档

## 下一步

- 练习 02：构建带有实际 ML 模型的模型服务 API
- 练习 03：为你的 API 添加 JWT 认证
- 练习 04：将你的 API 部署到生产环境

---

**预计完成时间：** 2-3 小时
**难度：** 初级
**重点：** REST API 基础、FastAPI 基础、CRUD 操作
