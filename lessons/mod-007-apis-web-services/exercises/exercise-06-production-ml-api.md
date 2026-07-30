# 练习 06：构建生产就绪的 ML API

## 概述

本综合练习将指导你使用 FastAPI 构建一个完整的生产就绪 ML API，实现适当的认证、速率限制、缓存、监控、错误处理和部署。你将创建一个端到端的推理 API，遵循行业最佳实践。

## 学习目标

完成此练习后，你将：
- 使用 FastAPI 构建完整的 ML 推理 API
- 实现 JWT 认证和 API key 验证
- 添加速率限制以防止滥用
- 为频繁请求的预测实现缓存
- 设置全面的监控和日志
- 使用适当的状态码优雅地处理错误
- 使用 OpenAPI/Swagger 创建 API 文档
- 使用适当的 CI/CD 部署 API
- 负载测试和优化性能

## 先决条件

- 已完成本模块的练习 01-05
- 所有 API 讲座（REST、FastAPI、认证、高级模式）
- Python FastAPI 知识
- 对 ML 推理的基本理解

## 所需时间

- 预计：150-180 分钟
- 难度：高级

---

## 第一部分：项目设置

### 任务 1.1：创建项目结构

```bash
mkdir ml-inference-api
cd ml-inference-api

# 创建目录结构
mkdir -p app/{api,core,models,schemas,services}
mkdir -p tests
mkdir -p scripts
mkdir -p deployment/{docker,kubernetes}

# 创建文件
touch app/__init__.py
touch app/main.py
touch app/api/__init__.py
touch app/core/{config.py,security.py,logging_config.py}
touch app/models/__init__.py
touch app/schemas/{request.py,response.py}
touch app/services/{inference.py,cache.py}
touch requirements.txt
touch .env.example
touch Dockerfile
touch docker-compose.yml
```

### 任务 1.2：安装依赖

**requirements.txt：**
```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
pydantic-settings==2.1.0
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.6
redis==5.0.1
prometheus-client==0.19.0
slowapi==0.1.9
onnxruntime==1.16.3
numpy==1.24.3
httpx==0.25.2
pytest==7.4.3
pytest-asyncio==0.21.1
```

```bash
# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # 在 Windows 上：venv\Scripts\activate

# 安装依赖
pip install -r requirements.txt
```

---

## 第二部分：配置和安全

### 任务 2.1：环境配置

**app/core/config.py：**
```python
from pydantic_settings import BaseSettings
from functools import lru_cache

class Settings(BaseSettings):
    # 应用设置
    APP_NAME: str = "ML Inference API"
    APP_VERSION: str = "1.0.0"
    DEBUG: bool = False

    # 安全
    SECRET_KEY: str
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    API_KEY_HEADER: str = "X-API-Key"

    # Redis
    REDIS_HOST: str = "localhost"
    REDIS_PORT: int = 6379
    REDIS_DB: int = 0
    CACHE_TTL: int = 3600  # 1 小时

    # 速率限制
    RATE_LIMIT_PER_MINUTE: int = 60
    RATE_LIMIT_PER_HOUR: int = 1000

    # 模型
    MODEL_PATH: str = "models/model.onnx"

    class Config:
        env_file = ".env"

@lru_cache()
def get_settings():
    return Settings()
```

**.env.example：**
```bash
SECRET_KEY=your-secret-key-change-this-in-production
REDIS_HOST=localhost
REDIS_PORT=6379
MODEL_PATH=models/model.onnx
DEBUG=False
```

### 任务 2.2：安全实现

**app/core/security.py：**
```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import HTTPException, Security
from fastapi.security import APIKeyHeader, HTTPBearer, HTTPAuthorizationCredentials
from app.core.config import get_settings

settings = get_settings()

# 密码哈希
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# API Key 认证
api_key_header = APIKeyHeader(name=settings.API_KEY_HEADER, auto_error=False)

# JWT Bearer 认证
security = HTTPBearer()

# 有效的 API key（生产环境中存储在数据库中）
VALID_API_KEYS = {
    "dev-key-123": {"user": "developer", "tier": "premium"},
    "test-key-456": {"user": "tester", "tier": "free"}
}

def verify_api_key(api_key: str = Security(api_key_header)):
    """验证 API key"""
    if not api_key or api_key not in VALID_API_KEYS:
        raise HTTPException(
            status_code=403,
            detail="Invalid or missing API key"
        )
    return VALID_API_KEYS[api_key]

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    """创建 JWT 访问令牌"""
    to_encode = data.copy()
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)

    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, settings.SECRET_KEY, algorithm=settings.ALGORITHM)
    return encoded_jwt

def verify_token(credentials: HTTPAuthorizationCredentials = Security(security)):
    """验证 JWT 令牌"""
    try:
        payload = jwt.decode(
            credentials.credentials,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        return payload
    except JWTError:
        raise HTTPException(
            status_code=401,
            detail="Invalid authentication credentials"
        )
```

---

## 第三部分：核心服务

### 任务 3.1：推理服务

**app/services/inference.py：**
```python
import onnxruntime as ort
import numpy as np
from typing import List
import logging
from app.core.config import get_settings

logger = logging.getLogger(__name__)
settings = get_settings()

class InferenceService:
    def __init__(self):
        self.session = None
        self.model_loaded = False

    def load_model(self):
        """加载 ONNX 模型"""
        try:
            self.session = ort.InferenceSession(settings.MODEL_PATH)
            self.model_loaded = True
            logger.info(f"Model loaded from {settings.MODEL_PATH}")
        except Exception as e:
            logger.error(f"Failed to load model: {e}")
            raise

    def predict(self, features: List[float]) -> float:
        """运行推理"""
        if not self.model_loaded:
            raise RuntimeError("Model not loaded")

        try:
            # 准备输入
            input_data = np.array([features], dtype=np.float32)

            # 运行推理
            outputs = self.session.run(None, {"input": input_data})
            prediction = float(outputs[0][0])

            return prediction

        except Exception as e:
            logger.error(f"Inference error: {e}")
            raise

# 全局推理服务
inference_service = InferenceService()
```

### 任务 3.2：缓存服务

**app/services/cache.py：**
```python
import redis
import json
import hashlib
from typing import Optional
from app.core.config import get_settings

settings = get_settings()

class CacheService:
    def __init__(self):
        self.redis_client = redis.Redis(
            host=settings.REDIS_HOST,
            port=settings.REDIS_PORT,
            db=settings.REDIS_DB,
            decode_responses=True
        )

    def _generate_key(self, features: list) -> str:
        """从特征生成缓存键"""
        features_str = json.dumps(features, sort_keys=True)
        return f"prediction:{hashlib.md5(features_str.encode()).hexdigest()}"

    def get(self, features: list) -> Optional[float]:
        """获取缓存的预测"""
        key = self._generate_key(features)
        cached = self.redis_client.get(key)
        return float(cached) if cached else None

    def set(self, features: list, prediction: float):
        """缓存预测"""
        key = self._generate_key(features)
        self.redis_client.setex(key, settings.CACHE_TTL, str(prediction))

    def clear(self):
        """清除所有缓存的预测"""
        keys = self.redis_client.keys("prediction:*")
        if keys:
            self.redis_client.delete(*keys)

cache_service = CacheService()
```

---

## 第四部分：API Schema

### 任务 4.1：请求/响应模型

**app/schemas/request.py：**
```python
from pydantic import BaseModel, Field
from typing import List

class PredictionRequest(BaseModel):
    features: List[float] = Field(
        ...,
        description="Input features for prediction",
        min_items=1,
        max_items=100
    )

    class Config:
        json_schema_extra = {
            "example": {
                "features": [1.0, 2.0, 3.0, 4.0, 5.0]
            }
        }

class BatchPredictionRequest(BaseModel):
    requests: List[PredictionRequest] = Field(
        ...,
        max_items=100
    )
```

**app/schemas/response.py：**
```python
from pydantic import BaseModel
from typing import List, Optional

class PredictionResponse(BaseModel):
    prediction: float
    latency_ms: float
    cached: bool = False
    model_version: str = "1.0.0"

class BatchPredictionResponse(BaseModel):
    predictions: List[PredictionResponse]
    total_latency_ms: float

class HealthResponse(BaseModel):
    status: str
    model_loaded: bool
    version: str

class ErrorResponse(BaseModel):
    error: str
    detail: Optional[str] = None
```

---

## 第五部分：API Endpoint

### 任务 5.1：主应用程序

**app/main.py：**
```python
from fastapi import FastAPI, Depends, HTTPException, Request
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from prometheus_client import Counter, Histogram, generate_latest
import time
import logging

from app.core.config import get_settings
from app.core.security import verify_api_key
from app.services.inference import inference_service
from app.services.cache import cache_service
from app.schemas.request import PredictionRequest, BatchPredictionRequest
from app.schemas.response import (
    PredictionResponse,
    BatchPredictionResponse,
    HealthResponse,
    ErrorResponse
)

# 初始化设置
settings = get_settings()

# 初始化速率限制器
limiter = Limiter(key_func=get_remote_address)

# 初始化 FastAPI 应用
app = FastAPI(
    title=settings.APP_NAME,
    version=settings.APP_VERSION,
    description="Production ML Inference API",
    docs_url="/docs",
    redoc_url="/redoc"
)

# 添加速率限制器到应用
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# CORS 中间件
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Prometheus 指标
PREDICTION_COUNT = Counter('predictions_total', 'Total predictions')
PREDICTION_LATENCY = Histogram('prediction_latency_seconds', 'Prediction latency')
CACHE_HITS = Counter('cache_hits_total', 'Total cache hits')
CACHE_MISSES = Counter('cache_misses_total', 'Total cache misses')
ERROR_COUNT = Counter('errors_total', 'Total errors', ['error_type'])

# 配置日志
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

@app.on_event("startup")
async def startup_event():
    """在启动时加载模型"""
    logger.info("Starting ML Inference API...")
    inference_service.load_model()
    logger.info("Model loaded successfully")

@app.get("/", tags=["Root"])
async def root():
    """API 根 endpoint"""
    return {
        "service": settings.APP_NAME,
        "version": settings.APP_VERSION,
        "endpoints": {
            "/health": "Health check",
            "/predict": "Single prediction (requires API key)",
            "/batch-predict": "Batch predictions (requires API key)",
            "/metrics": "Prometheus metrics",
            "/docs": "API documentation"
        }
    }

@app.get("/health", response_model=HealthResponse, tags=["Health"])
async def health_check():
    """健康检查 endpoint"""
    return HealthResponse(
        status="healthy",
        model_loaded=inference_service.model_loaded,
        version=settings.APP_VERSION
    )

@app.post(
    "/predict",
    response_model=PredictionResponse,
    tags=["Predictions"],
    dependencies=[Depends(verify_api_key)]
)
@limiter.limit(f"{settings.RATE_LIMIT_PER_MINUTE}/minute")
async def predict(request: Request, pred_request: PredictionRequest):
    """进行预测"""
    start_time = time.time()

    try:
        # 先检查缓存
        cached_result = cache_service.get(pred_request.features)

        if cached_result is not None:
            CACHE_HITS.inc()
            latency = (time.time() - start_time) * 1000

            return PredictionResponse(
                prediction=cached_result,
                latency_ms=latency,
                cached=True
            )

        # 缓存未命中 - 运行推理
        CACHE_MISSES.inc()
        prediction = inference_service.predict(pred_request.features)

        # 缓存结果
        cache_service.set(pred_request.features, prediction)

        # 指标
        latency = (time.time() - start_time) * 1000
        PREDICTION_COUNT.inc()
        PREDICTION_LATENCY.observe(time.time() - start_time)

        return PredictionResponse(
            prediction=prediction,
            latency_ms=latency,
            cached=False
        )

    except Exception as e:
        ERROR_COUNT.labels(error_type=type(e).__name__).inc()
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.post(
    "/batch-predict",
    response_model=BatchPredictionResponse,
    tags=["Predictions"],
    dependencies=[Depends(verify_api_key)]
)
@limiter.limit(f"{settings.RATE_LIMIT_PER_MINUTE}/minute")
async def batch_predict(request: Request, batch_request: BatchPredictionRequest):
    """进行批量预测"""
    start_time = time.time()

    try:
        predictions = []

        for pred_req in batch_request.requests:
            pred_start = time.time()
            prediction = inference_service.predict(pred_req.features)
            pred_latency = (time.time() - pred_start) * 1000

            predictions.append(PredictionResponse(
                prediction=prediction,
                latency_ms=pred_latency,
                cached=False
            ))

            PREDICTION_COUNT.inc()

        total_latency = (time.time() - start_time) * 1000

        return BatchPredictionResponse(
            predictions=predictions,
            total_latency_ms=total_latency
        )

    except Exception as e:
        ERROR_COUNT.labels(error_type=type(e).__name__).inc()
        logger.error(f"Batch prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/metrics", tags=["Monitoring"])
async def metrics():
    """Prometheus 指标 endpoint"""
    return JSONResponse(content=generate_latest().decode(), media_type="text/plain")

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """全局异常处理器"""
    ERROR_COUNT.labels(error_type="unhandled").inc()
    logger.error(f"Unhandled exception: {exc}")
    return JSONResponse(
        status_code=500,
        content={"error": "Internal server error", "detail": str(exc)}
    )

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## 第六部分：测试

### 任务 6.1：单元测试

**tests/test_api.py：**
```python
import pytest
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def test_root():
    response = client.get("/")
    assert response.status_code == 200
    assert "service" in response.json()

def test_health():
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"

def test_predict_no_api_key():
    response = client.post(
        "/predict",
        json={"features": [1.0, 2.0, 3.0]}
    )
    assert response.status_code == 403

def test_predict_with_api_key():
    response = client.post(
        "/predict",
        json={"features": [1.0, 2.0, 3.0]},
        headers={"X-API-Key": "dev-key-123"}
    )
    assert response.status_code == 200
    assert "prediction" in response.json()
    assert "latency_ms" in response.json()

def test_batch_predict():
    response = client.post(
        "/batch-predict",
        json={
            "requests": [
                {"features": [1.0, 2.0]},
                {"features": [3.0, 4.0]}
            ]
        },
        headers={"X-API-Key": "dev-key-123"}
    )
    assert response.status_code == 200
    assert len(response.json()["predictions"]) == 2

# 运行测试
# pytest tests/test_api.py -v
```

---

## 第七部分：部署

### 任务 7.1：Docker 部署

**Dockerfile：**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用程序
COPY app/ ./app/
COPY models/ ./models/

# 暴露端口
EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8000/health || exit 1

# 运行应用程序
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**docker-compose.yml：**
```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - REDIS_HOST=redis
      - SECRET_KEY=dev-secret-key-change-in-prod
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    restart: unless-stopped
```

---

## 交付物

提交以下内容：

1. **完整 API 代码**：
   - 包含所有 endpoint 的 FastAPI 应用程序
   - 认证和授权
   - 速率限制和缓存
   - 错误处理

2. **测试**：
   - 所有 endpoint 的单元测试
   - 集成测试
   - 性能测试

3. **文档**：
   - API 文档（使用 FastAPI 自动生成）
   - 部署指南
   - 使用示例

4. **证据**：
   - Swagger UI 截图
   - 负载测试结果
   - Prometheus 指标仪表盘

---

## 挑战问题

1. **安全**：如何防止 API key 在日志中泄露？

2. **扩展**：如何处理 100,000 请求/秒？

3. **监控**：你会为生产环境设置哪些告警？

4. **版本化**：如何在不破坏 v1 客户端的情况下部署 API v2？

5. **成本**：如何优化月请求量 10 亿的 API 的成本？

---

## 附加资源

- [FastAPI 文档](https://fastapi.tiangolo.com/)
- [API 安全最佳实践](https://owasp.org/www-project-api-security/)
- [速率限制模式](https://cloud.google.com/architecture/rate-limiting-strategies)
- [Prometheus 指标](https://prometheus.io/docs/practices/naming/)

---

**恭喜！** 你已经构建了一个带有认证、缓存、监控和适当错误处理的生产就绪 ML API。
