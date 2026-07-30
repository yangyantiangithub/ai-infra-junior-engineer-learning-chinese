# 练习 05：API 文档和 OpenAPI

## 概述

使用 OpenAPI/Swagger 创建全面的 API 文档，自定义自动生成的文档，添加示例，并学习记录 ML API 的最佳实践。良好的文档对于 API 的采用和维护至关重要。

**难度：** 初级-中级
**预计时间：** 1-2 小时
**先决条件：**
- 已完成练习 01-02
- 理解 REST 原则
- 讲座 01-02

## 学习目标

完成此练习后，你将：
- 自定义 FastAPI 的自动生成文档
- 为 endpoint 添加详细的描述和示例
- 全面记录请求/响应模型
- 添加认证文档
- 创建自定义 API 文档页面
- 导出 OpenAPI schema 用于客户端生成

## 场景

你的 ML API 需要专业的文档用于：
- 集成 API 的内部开发者
- 使用你模型的外部合作伙伴
- 客户端 SDK 生成
- API 测试和验证

## 第一部分：增强的 Endpoint 文档

**文件：app/main.py**（增强现有 endpoint）

```python
from fastapi import FastAPI, status
from pydantic import BaseModel, Field
from typing import List, Optional

app = FastAPI(
    title="ML Model API",
    description="""
    # ML Model Serving API

    This API provides machine learning model inference capabilities with the following features:

    * **Authentication**: JWT-based authentication
    * **Rate Limiting**: Tiered limits based on subscription
    * **Models**: Multiple pre-trained models available
    * **Real-time**: Low-latency predictions
    * **Batch Processing**: Process multiple inputs efficiently

    ## Getting Started

    1. Obtain API credentials
    2. Authenticate to get access token
    3. Make prediction requests

    ## Rate Limits

    - Free tier: 10 requests/minute
    - Pro tier: 100 requests/minute
    - Enterprise: Custom limits

    For support, contact: support@example.com
    """,
    version="1.0.0",
    terms_of_service="https://example.com/terms",
    contact={
        "name": "API Support",
        "url": "https://example.com/support",
        "email": "support@example.com"
    },
    license_info={
        "name": "MIT License",
        "url": "https://opensource.org/licenses/MIT"
    }
)

# TODO: 添加详细的 endpoint 文档
@app.post(
    "/api/v1/predict",
    response_model=PredictionOutput,
    status_code=status.HTTP_200_OK,
    summary="Make a prediction",
    description="""
    Make a prediction using the sentiment analysis model.

    This endpoint analyzes text and returns sentiment classification
    with confidence scores.

    **Processing Time:** < 100ms for single predictions

    **Example Use Cases:**
    - Customer review analysis
    - Social media sentiment tracking
    - Product feedback classification
    """,
    response_description="Prediction result with confidence score",
    tags=["Predictions"],
    responses={
        200: {
            "description": "Successful prediction",
            "content": {
                "application/json": {
                    "example": {
                        "text": "This product is amazing!",
                        "prediction": "positive",
                        "confidence": 0.95,
                        "timestamp": "2024-10-18T14:30:00Z"
                    }
                }
            }
        },
        401: {
            "description": "Authentication failed",
            "content": {
                "application/json": {
                    "example": {"detail": "Invalid or expired token"}
                }
            }
        },
        422: {
            "description": "Validation error",
            "content": {
                "application/json": {
                    "example": {
                        "detail": [
                            {
                                "loc": ["body", "text"],
                                "msg": "field required",
                                "type": "value_error.missing"
                            }
                        ]
                    }
                }
            }
        },
        429: {
            "description": "Rate limit exceeded",
            "content": {
                "application/json": {
                    "example": {
                        "detail": "Too many requests",
                        "retry_after": 60
                    }
                }
            }
        }
    }
)
async def predict(
    input_data: PredictionInput,
    current_user: User = Depends(get_current_active_user)
):
    """
    Make a sentiment prediction on the provided text.

    Args:
        input_data: Text input for sentiment analysis

    Returns:
        Prediction result with sentiment label and confidence

    Raises:
        HTTPException: If model is not loaded or prediction fails
    """
    # 实现
    pass
```

## 第二部分：增强的 Pydantic 模型

**文件：app/models.py**

```python
from pydantic import BaseModel, Field, validator
from typing import Optional, List
from datetime import datetime
from enum import Enum

class SentimentLabel(str, Enum):
    """情感分类标签"""
    positive = "positive"
    negative = "negative"
    neutral = "neutral"

class PredictionInput(BaseModel):
    """
    情感预测的输入数据。

    模型接受最多 5000 个字符的文本并执行
    情感分析。
    """

    text: str = Field(
        ...,
        title="Input Text",
        description="Text to analyze for sentiment (1-5000 characters)",
        min_length=1,
        max_length=5000,
        example="This product exceeded my expectations! Highly recommend."
    )

    language: Optional[str] = Field(
        "en",
        title="Language Code",
        description="ISO 639-1 language code (e.g., 'en', 'es', 'fr')",
        regex="^[a-z]{2}$",
        example="en"
    )

    return_probabilities: bool = Field(
        False,
        title="Return Probabilities",
        description="Include probability scores for all classes",
        example=True
    )

    class Config:
        schema_extra = {
            "example": {
                "text": "This movie was absolutely fantastic! Great acting and plot.",
                "language": "en",
                "return_probabilities": True
            }
        }

class PredictionOutput(BaseModel):
    """
    情感分析的预测结果。

    包含预测的情感、置信度分数以及可选的所有类别的概率分布。
    """

    text: str = Field(
        ...,
        title="Original Text",
        description="The input text that was analyzed"
    )

    prediction: SentimentLabel = Field(
        ...,
        title="Predicted Sentiment",
        description="The predicted sentiment label"
    )

    confidence: float = Field(
        ...,
        title="Confidence Score",
        description="Model confidence in prediction (0.0-1.0)",
        ge=0.0,
        le=1.0,
        example=0.95
    )

    probabilities: Optional[dict] = Field(
        None,
        title="Class Probabilities",
        description="Probability distribution across all classes",
        example={
            "positive": 0.95,
            "negative": 0.03,
            "neutral": 0.02
        }
    )

    model_version: str = Field(
        ...,
        title="Model Version",
        description="Version of the model used for prediction",
        example="1.0.0"
    )

    processing_time_ms: Optional[int] = Field(
        None,
        title="Processing Time",
        description="Time taken to process prediction in milliseconds",
        example=45
    )

    timestamp: datetime = Field(
        ...,
        title="Timestamp",
        description="ISO 8601 timestamp of prediction",
        example="2024-10-18T14:30:00.000Z"
    )

    class Config:
        schema_extra = {
            "example": {
                "text": "This movie was absolutely fantastic!",
                "prediction": "positive",
                "confidence": 0.95,
                "probabilities": {
                    "positive": 0.95,
                    "negative": 0.03,
                    "neutral": 0.02
                },
                "model_version": "1.0.0",
                "processing_time_ms": 45,
                "timestamp": "2024-10-18T14:30:00.000Z"
            }
        }

class BatchPredictionInput(BaseModel):
    """批量预测的输入"""

    texts: List[str] = Field(
        ...,
        title="Text List",
        description="List of texts to analyze (max 100)",
        min_items=1,
        max_items=100,
        example=[
            "Great product!",
            "Very disappointed.",
            "It's okay, nothing special."
        ]
    )

    @validator('texts')
    def validate_texts(cls, v):
        """确保每个文本不为空"""
        for text in v:
            if not text.strip():
                raise ValueError("Text cannot be empty")
        return v

    class Config:
        schema_extra = {
            "example": {
                "texts": [
                    "This movie was amazing! Loved it.",
                    "Waste of time, very boring.",
                    "It was okay, nothing special.",
                    "Excellent performance by the actors."
                ]
            }
        }
```

## 第三部分：标签组织

**文件：app/main.py**（使用标签组织）

```python
from fastapi import FastAPI

# TODO: 定义标签元数据
tags_metadata = [
    {
        "name": "Authentication",
        "description": "Operations for user authentication and authorization",
    },
    {
        "name": "Predictions",
        "description": "Model inference endpoints for sentiment analysis",
        "externalDocs": {
            "description": "Model documentation",
            "url": "https://docs.example.com/models/sentiment"
        }
    },
    {
        "name": "Models",
        "description": "Model management and information",
    },
    {
        "name": "Jobs",
        "description": "Asynchronous job management for batch processing",
    },
    {
        "name": "Health",
        "description": "Health check and monitoring endpoints",
    }
]

app = FastAPI(
    title="ML Model API",
    openapi_tags=tags_metadata
)

# 将标签应用于 endpoint
@app.post("/token", tags=["Authentication"])
async def login():
    pass

@app.post("/api/v1/predict", tags=["Predictions"])
async def predict():
    pass

@app.get("/api/v1/models", tags=["Models"])
async def list_models():
    pass

@app.get("/health", tags=["Health"])
async def health_check():
    pass
```

## 第四部分：自定义文档页面

**文件：app/main.py**（添加自定义文档）

```python
from fastapi import FastAPI
from fastapi.openapi.docs import get_swagger_ui_html, get_redoc_html
from fastapi.openapi.utils import get_openapi

app = FastAPI(docs_url=None, redoc_url=None)  # 禁用默认文档

@app.get("/docs", include_in_schema=False)
async def custom_swagger_ui_html():
    """带自定义样式的 Swagger UI"""
    return get_swagger_ui_html(
        openapi_url=app.openapi_url,
        title=f"{app.title} - Documentation",
        swagger_favicon_url="https://fastapi.tiangolo.com/img/favicon.png"
    )

@app.get("/redoc", include_in_schema=False)
async def redoc_html():
    """自定义 ReDoc 文档"""
    return get_redoc_html(
        openapi_url=app.openapi_url,
        title=f"{app.title} - ReDoc",
        redoc_favicon_url="https://fastapi.tiangolo.com/img/favicon.png"
    )

def custom_openapi():
    """自定义 OpenAPI schema"""
    if app.openapi_schema:
        return app.openapi_schema

    openapi_schema = get_openapi(
        title="ML Model Serving API",
        version="1.0.0",
        description="Production ML API with authentication and rate limiting",
        routes=app.routes,
    )

    # TODO: 添加自定义 schema 属性
    openapi_schema["info"]["x-logo"] = {
        "url": "https://example.com/logo.png"
    }

    # TODO: 添加安全方案
    openapi_schema["components"]["securitySchemes"] = {
        "Bearer": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT",
            "description": "Enter your JWT token"
        }
    }

    app.openapi_schema = openapi_schema
    return app.openapi_schema

app.openapi = custom_openapi
```

## 第五部分：文档中的代码示例

**文件：app/docs_examples.py**

```python
"""
API 文档的代码示例
"""

# Python 示例
python_example = """
import requests

# 1. Authenticate
response = requests.post(
    'https://api.example.com/token',
    data={'username': 'your_username', 'password': 'your_password'}
)
token = response.json()['access_token']

# 2. Make prediction
headers = {'Authorization': f'Bearer {token}'}
response = requests.post(
    'https://api.example.com/api/v1/predict',
    headers=headers,
    json={'text': 'This product is amazing!'}
)
result = response.json()
print(f"Sentiment: {result['prediction']}, Confidence: {result['confidence']}")
"""

# cURL 示例
curl_example = """
# 1. Authenticate
curl -X POST "https://api.example.com/token" \\
  -H "Content-Type: application/x-www-form-urlencoded" \\
  -d "username=your_username&password=your_password"

# 2. Make prediction
curl -X POST "https://api.example.com/api/v1/predict" \\
  -H "Authorization: Bearer YOUR_TOKEN" \\
  -H "Content-Type: application/json" \\
  -d '{"text": "This product is amazing!"}'
"""

# JavaScript 示例
javascript_example = """
// 1. Authenticate
const authResponse = await fetch('https://api.example.com/token', {
  method: 'POST',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: new URLSearchParams({
    username: 'your_username',
    password: 'your_password'
  })
});
const { access_token } = await authResponse.json();

// 2. Make prediction
const predResponse = await fetch('https://api.example.com/api/v1/predict', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${access_token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    text: 'This product is amazing!'
  })
});
const result = await predResponse.json();
console.log(`Sentiment: ${result.prediction}, Confidence: ${result.confidence}`);
"""
```

## 第六部分：导出 OpenAPI Schema

**文件：scripts/export_openapi.py**

```python
"""
导出 OpenAPI schema 用于客户端生成
"""
import json
import yaml
from app.main import app

# 导出为 JSON
def export_json():
    """将 OpenAPI schema 导出为 JSON"""
    with open('openapi.json', 'w') as f:
        json.dump(app.openapi(), f, indent=2)
    print("OpenAPI schema exported to openapi.json")

# 导出为 YAML
def export_yaml():
    """将 OpenAPI schema 导出为 YAML"""
    with open('openapi.yaml', 'w') as f:
        yaml.dump(app.openapi(), f, sort_keys=False)
    print("OpenAPI schema exported to openapi.yaml")

if __name__ == "__main__":
    export_json()
    export_yaml()
```

**运行导出：**
```bash
python scripts/export_openapi.py
```

## 第七部分：生成客户端 SDK

### 使用 OpenAPI Generator

```bash
# 安装 OpenAPI Generator
npm install -g @openapitools/openapi-generator-cli

# 生成 Python 客户端
openapi-generator-cli generate \
  -i openapi.json \
  -g python \
  -o clients/python-client

# 生成 JavaScript 客户端
openapi-generator-cli generate \
  -i openapi.json \
  -g javascript \
  -o clients/js-client

# 生成 Java 客户端
openapi-generator-cli generate \
  -i openapi.json \
  -g java \
  -o clients/java-client
```

### 测试生成的客户端

```python
# 使用自动生成的 Python 客户端
from generated_client import ApiClient, PredictionsApi
from generated_client.models import PredictionInput

# 配置客户端
client = ApiClient(configuration)
client.configuration.access_token = 'your_jwt_token'

# 进行预测
api = PredictionsApi(client)
result = api.predict(PredictionInput(text="This is great!"))
print(result.prediction)
```

## 第八部分：文档测试

**文件：tests/test_documentation.py**

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_openapi_schema():
    """测试 OpenAPI schema 已生成"""
    response = client.get("/openapi.json")
    assert response.status_code == 200

    schema = response.json()
    assert "openapi" in schema
    assert schema["info"]["title"] == "ML Model API"
    assert "paths" in schema
    assert "components" in schema

def test_swagger_docs_accessible():
    """测试 Swagger UI 可访问"""
    response = client.get("/docs")
    assert response.status_code == 200
    assert "swagger" in response.text.lower()

def test_redoc_accessible():
    """测试 ReDoc 可访问"""
    response = client.get("/redoc")
    assert response.status_code == 200

def test_all_endpoints_documented():
    """测试所有 endpoint 都有描述"""
    schema = client.get("/openapi.json").json()

    for path, methods in schema["paths"].items():
        for method, details in methods.items():
            # 跳过非 HTTP 方法
            if method not in ["get", "post", "put", "delete", "patch"]:
                continue

            assert "summary" in details or "description" in details, \
                f"Endpoint {method.upper()} {path} missing documentation"

def test_models_have_examples():
    """测试模型在 schema 中有示例"""
    schema = client.get("/openapi.json").json()

    if "components" in schema and "schemas" in schema["components"]:
        for model_name, model_schema in schema["components"]["schemas"].items():
            # 检查是否存在示例
            has_example = (
                "example" in model_schema or
                ("properties" in model_schema and
                 any("example" in prop for prop in model_schema["properties"].values()))
            )
            print(f"{model_name}: has_example={has_example}")

def test_security_scheme_documented():
    """测试安全方案已记录"""
    schema = client.get("/openapi.json").json()

    assert "components" in schema
    assert "securitySchemes" in schema["components"]
    assert "Bearer" in schema["components"]["securitySchemes"]
```

## 交付物

- ✅ 全面的 endpoint 文档
- ✅ 带示例的增强 Pydantic 模型
- ✅ 组织有序的标签和章节
- ✅ 自定义文档页面
- ✅ 导出的 OpenAPI schema (JSON/YAML)
- ✅ 生成的客户端 SDK
- ✅ 文档测试

## 最佳实践清单

- [ ] 每个 endpoint 有清晰的摘要
- [ ] 每个 endpoint 有详细的描述
- [ ] 每个参数有描述
- [ ] 所有模型有示例
- [ ] 响应码已记录
- [ ] 错误响应已记录
- [ ] 认证已记录
- [ ] 速率限制已记录
- [ ] 提供了代码示例
- [ ] 维护了变更日志

## 关键要点

1. **自动生成：** FastAPI 自动生成 OpenAPI 文档
2. **自定义：** 通过 Pydantic 和装饰器进行广泛自定义
3. **示例：** 示例使 API 更易于使用
4. **客户端生成：** OpenAPI schema 支持自动客户端生成
5. **测试：** 文档应像代码一样被测试

## 下一步

- 学习 API 版本化策略
- 实现变更日志跟踪
- 创建交互式教程
- 搭建文档站点（Docusaurus、MkDocs）

---

**预计时间：** 1-2 小时
**难度：** 初级-中级
**重点：** API 文档、OpenAPI、示例、客户端生成
