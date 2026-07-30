# 练习 02：带认证的 ML 模型服务 API

## 概述

构建一个生产就绪的 ML 模型服务 API，加载实际的机器学习模型并进行预测。本练习将 JWT 认证、适当的模型加载和推理能力添加到你的 REST API 技能中。

**难度：** 中级
**预计时间：** 3-4 小时
**先决条件：**
- 已完成练习 01
- 讲座 02：FastAPI 框架
- 讲座 03：认证和安全
- 对 scikit-learn 或 PyTorch 的基本理解

## 学习目标

完成此练习后，你将：
- 加载和服务真实的 ML 模型
- 实现基于 JWT 的认证
- 创建安全的预测 endpoint
- 高效处理模型推理
- 为 ML 操作实现适当的错误处理
- 使用依赖注入加载模型
- 为认证 endpoint 编写测试

## 场景

你在构建一个服务预训练情感分析模型的 API。该 API 应：
- 使用 JWT 令牌认证用户
- 在启动时加载 ML 模型
- 提供预测 endpoint（单次和批量）
- 跟踪预测统计信息
- 优雅地处理错误

## 项目设置

### 1. 安装额外依赖

```bash
pip install python-jose[cryptography]
pip install passlib[bcrypt]
pip install python-multipart
pip install scikit-learn
pip install numpy
```

### 2. 项目结构

```
ml-serving-api/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── auth.py
│   ├── models.py
│   ├── ml_service.py
│   └── config.py
├── models/
│   └── sentiment_model.pkl
├── tests/
│   ├── __init__.py
│   └── test_api.py
├── scripts/
│   └── train_model.py
├── requirements.txt
└── .env
```

## 第一部分：训练简单模型

首先，创建一个用于测试的简单情感分析模型。

**文件：scripts/train_model.py**

```python
"""
为 API 训练一个简单的情感分析模型
"""
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
import pickle
import os

# TODO: 创建训练数据
# 示例正面和负面电影评论
positive_reviews = [
    "This movie was amazing! I loved every minute.",
    "Excellent film with great acting and plot.",
    "One of the best movies I've seen this year.",
    "Fantastic! Highly recommend this movie.",
    "Brilliant performances and wonderful story.",
]

negative_reviews = [
    "Terrible movie, waste of time.",
    "Very disappointing and boring.",
    "Poor acting and weak storyline.",
    "I didn't enjoy this film at all.",
    "One of the worst movies I've ever watched.",
]

# TODO: 合并评论并创建标签
# X = positive_reviews + negative_reviews
# y = ['positive'] * len(positive_reviews) + ['negative'] * len(negative_reviews)

# TODO: 创建包含 TfidfVectorizer 和 MultinomialNB 的 pipeline
# pipeline = Pipeline([...])

# TODO: 训练模型
# pipeline.fit(X, y)

# TODO: 测试模型
# test_texts = ["This was great!", "Really bad movie"]
# predictions = pipeline.predict(test_texts)
# print(predictions)

# TODO: 将模型保存到文件
# os.makedirs('models', exist_ok=True)
# with open('models/sentiment_model.pkl', 'wb') as f:
#     pickle.dump(pipeline, f)

print("Model trained and saved successfully!")
```

**运行训练脚本：**
```bash
python scripts/train_model.py
```

## 第二部分：配置管理

**文件：app/config.py**

```python
from pydantic import BaseSettings
from typing import Optional

class Settings(BaseSettings):
    """应用设置"""

    # TODO: 定义设置
    # app_name: str = "ML Serving API"
    # secret_key: str (用于 JWT 签名)
    # algorithm: str = "HS256"
    # access_token_expire_minutes: int = 30
    # model_path: str = "models/sentiment_model.pkl"

    class Config:
        env_file = ".env"

settings = Settings()
```

**文件：.env**

```bash
SECRET_KEY=your-secret-key-change-this-in-production
```

## 第三部分：实现认证

**文件：app/auth.py**

```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from pydantic import BaseModel
from app.config import settings

# TODO: 初始化密码上下文
# pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# TODO: 初始化 OAuth2 方案
# oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class Token(BaseModel):
    access_token: str
    token_type: str

class TokenData(BaseModel):
    username: Optional[str] = None

class User(BaseModel):
    username: str
    email: str
    disabled: bool = False

class UserInDB(User):
    hashed_password: str

# TODO: Mock 用户数据库
# 生产环境使用真实数据库
fake_users_db = {
    "alice": {
        "username": "alice",
        "email": "alice@example.com",
        "hashed_password": "$2b$12$...",  # "secret123" 的哈希
        "disabled": False,
    }
}

def verify_password(plain_password: str, hashed_password: str) -> bool:
    """验证密码与其哈希"""
    # TODO: 使用 pwd_context.verify()
    pass

def get_password_hash(password: str) -> str:
    """哈希密码"""
    # TODO: 使用 pwd_context.hash()
    pass

def get_user(username: str) -> Optional[UserInDB]:
    """从数据库获取用户"""
    # TODO: 在 fake_users_db 中查找用户
    # TODO: 如果找到返回 UserInDB，否则返回 None
    pass

def authenticate_user(username: str, password: str) -> Optional[UserInDB]:
    """认证用户"""
    # TODO: 从数据库获取用户
    # TODO: 验证密码
    # TODO: 如果认证通过返回用户，否则返回 None
    pass

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """创建 JWT 访问令牌"""
    # TODO: 复制数据
    # TODO: 添加过期时间
    # TODO: 使用密钥编码 JWT
    # TODO: 返回编码后的令牌
    pass

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    """从 JWT 令牌获取当前用户"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )

    # TODO: 解码 JWT 令牌
    # TODO: 从 payload 中提取用户名
    # TODO: 从数据库获取用户
    # TODO: 如果无效则抛出异常
    # TODO: 返回用户
    pass

async def get_current_active_user(
    current_user: User = Depends(get_current_user)
) -> User:
    """确保用户活跃"""
    # TODO: 检查用户是否被禁用
    # TODO: 如果禁用则抛出异常
    # TODO: 返回用户
    pass
```

## 第四部分：创建 ML 服务

**文件：app/ml_service.py**

```python
import pickle
import numpy as np
from typing import List, Dict
from datetime import datetime
from app.config import settings

class MLModelService:
    """用于加载和运行 ML 模型的服务"""

    def __init__(self):
        self.model = None
        self.model_loaded = False
        self.prediction_count = 0
        self.model_info = {
            "name": "Sentiment Analysis Model",
            "version": "1.0.0",
            "framework": "scikit-learn"
        }

    def load_model(self):
        """从文件加载 ML 模型"""
        # TODO: 打开模型文件
        # TODO: 使用 pickle 加载模型
        # TODO: 设置 model_loaded 为 True
        # TODO: 处理错误（文件未找到、文件损坏等）
        pass

    def predict(self, text: str) -> Dict:
        """对单个文本进行预测"""
        # TODO: 检查模型是否已加载
        # TODO: 进行预测
        # TODO: 如果可用则获取预测概率
        # TODO: 递增 prediction_count
        # TODO: 返回结果字典，包含：
        #   - prediction: str
        #   - confidence: float (可选)
        #   - timestamp: datetime
        pass

    def predict_batch(self, texts: List[str]) -> List[Dict]:
        """对一批文本进行预测"""
        # TODO: 检查模型是否已加载
        # TODO: 进行批量预测
        # TODO: 返回结果列表
        pass

    def get_stats(self) -> Dict:
        """获取模型统计信息"""
        # TODO: 返回字典，包含：
        #   - model_loaded: bool
        #   - total_predictions: int
        #   - model_info: dict
        pass

# 创建全局实例
ml_service = MLModelService()
```

## 第五部分：定义 API 模型

**文件：app/models.py**

```python
from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import datetime

class PredictionInput(BaseModel):
    """单次预测的输入"""
    text: str = Field(..., min_length=1, max_length=5000)

    class Config:
        schema_extra = {
            "example": {
                "text": "This movie was absolutely fantastic!"
            }
        }

class PredictionOutput(BaseModel):
    """单次预测的输出"""
    # TODO: 定义字段：
    # - text: str (原始输入)
    # - prediction: str (positive/negative)
    # - confidence: Optional[float]
    # - timestamp: datetime
    # - model_version: str
    pass

class BatchPredictionInput(BaseModel):
    """批量预测的输入"""
    # TODO: 定义字段：
    # - texts: List[str] 带验证（最多 100 项）
    pass

class BatchPredictionOutput(BaseModel):
    """批量预测的输出"""
    # TODO: 定义字段：
    # - results: List[PredictionOutput]
    # - total_count: int
    # - processing_time_ms: float
    pass

class ModelStats(BaseModel):
    """模型统计信息"""
    # TODO: 定义字段：
    # - model_loaded: bool
    # - total_predictions: int
    # - model_name: str
    # - model_version: str
    # - framework: str
    pass
```

## 第六部分：创建 API Endpoint

**文件：app/main.py**

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from datetime import timedelta
from typing import List
import time

from app.config import settings
from app.auth import (
    authenticate_user, create_access_token,
    get_current_active_user, Token, User
)
from app.models import (
    PredictionInput, PredictionOutput,
    BatchPredictionInput, BatchPredictionOutput,
    ModelStats
)
from app.ml_service import ml_service

# TODO: 创建 FastAPI 应用
app = FastAPI(
    title=settings.app_name,
    version="1.0.0",
    description="ML Model Serving API with Authentication"
)

@app.on_event("startup")
async def startup_event():
    """在启动时加载 ML 模型"""
    # TODO: 使用 ml_service.load_model() 加载模型
    # TODO: 记录成功/失败
    pass

@app.get("/")
def root():
    """根 endpoint"""
    # TODO: 返回欢迎消息和 API 信息
    pass

@app.get("/health")
def health_check():
    """健康检查 endpoint"""
    # TODO: 返回健康状态和模型加载状态
    pass

@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """登录 endpoint 获取 JWT 令牌"""
    # TODO: 认证用户
    # TODO: 如果认证失败，抛出 401 HTTPException
    # TODO: 创建带过期时间的访问令牌
    # TODO: 返回令牌
    pass

@app.get("/users/me", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_active_user)):
    """获取当前用户资料"""
    # TODO: 返回当前用户
    pass

@app.post("/api/v1/predict", response_model=PredictionOutput)
async def predict(
    input_data: PredictionInput,
    current_user: User = Depends(get_current_active_user)
):
    """单次预测（需认证）"""
    # TODO: 调用 ml_service.predict()
    # TODO: 处理错误（模型未加载、预测失败）
    # TODO: 返回预测结果
    pass

@app.post("/api/v1/predict/batch", response_model=BatchPredictionOutput)
async def predict_batch(
    input_data: BatchPredictionInput,
    current_user: User = Depends(get_current_active_user)
):
    """批量预测（需认证）"""
    # TODO: 记录开始时间
    # TODO: 调用 ml_service.predict_batch()
    # TODO: 计算处理时间
    # TODO: 返回带时序的批量结果
    pass

@app.get("/api/v1/model/stats", response_model=ModelStats)
async def get_model_stats(current_user: User = Depends(get_current_active_user)):
    """获取模型统计信息（需认证）"""
    # TODO: 调用 ml_service.get_stats()
    # TODO: 返回统计信息
    pass
```

## 第七部分：编写测试

**文件：tests/test_api.py**

```python
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

def get_auth_token() -> str:
    """辅助函数：获取认证令牌"""
    # TODO: 使用测试凭证登录
    # TODO: 提取并返回访问令牌
    pass

def test_root():
    """测试根 endpoint"""
    # TODO: 向 / 发起 GET 请求
    # TODO: 断言状态 200
    pass

def test_health_check():
    """测试健康 endpoint"""
    # TODO: 向 /health 发起 GET 请求
    # TODO: 断言状态 200
    # TODO: 断言 model_loaded 为 True
    pass

def test_login_success():
    """测试成功登录"""
    # TODO: 使用有效凭证 POST 到 /token
    # TODO: 断言状态 200
    # TODO: 断言 access_token 在响应中
    # TODO: 断言 token_type 为 "bearer"
    pass

def test_login_failure():
    """测试失败登录"""
    # TODO: 使用无效凭证 POST 到 /token
    # TODO: 断言状态 401
    pass

def test_predict_without_auth():
    """测试无认证的预测"""
    # TODO: POST 到 /api/v1/predict 不带令牌
    # TODO: 断言状态 401
    pass

def test_predict_with_auth():
    """测试带认证的预测"""
    # TODO: 获取认证令牌
    # TODO: 使用令牌 POST 到 /api/v1/predict
    # TODO: 断言状态 200
    # TODO: 断言预测为 "positive" 或 "negative"
    pass

def test_predict_positive_sentiment():
    """测试正面情感预测"""
    # TODO: 获取认证令牌
    # TODO: 使用正面文本进行预测
    # TODO: 断言预测为 "positive"
    pass

def test_predict_negative_sentiment():
    """测试负面情感预测"""
    # TODO: 获取认证令牌
    # TODO: 使用负面文本进行预测
    # TODO: 断言预测为 "negative"
    pass

def test_batch_predict():
    """测试批量预测"""
    # TODO: 获取认证令牌
    # TODO: POST 到 /api/v1/predict/batch 带多个文本
    # TODO: 断言状态 200
    # TODO: 断言结果数量正确
    # TODO: 断言 processing_time_ms 存在
    pass

def test_model_stats():
    """测试模型统计信息"""
    # TODO: 获取认证令牌
    # TODO: GET /api/v1/model/stats
    # TODO: 断言状态 200
    # TODO: 断言 model_loaded 为 True
    # TODO: 断言 total_predictions >= 0
    pass

def test_get_current_user():
    """测试获取当前用户资料"""
    # TODO: 获取认证令牌
    # TODO: GET /users/me
    # TODO: 断言状态 200
    # TODO: 断言用户名匹配
    pass
```

## 第八部分：运行和测试

### 1. 训练模型

```bash
python scripts/train_model.py
```

### 2. 运行 API

```bash
uvicorn app.main:app --reload
```

### 3. 测试认证流程

**获取令牌：**
```bash
curl -X POST "http://localhost:8000/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=alice&password=secret123"
```

响应：
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer"
}
```

**进行预测：**
```bash
curl -X POST "http://localhost:8000/api/v1/predict" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"text": "This movie was absolutely fantastic!"}'
```

**批量预测：**
```bash
curl -X POST "http://localhost:8000/api/v1/predict/batch" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "texts": [
      "Great movie!",
      "Terrible film.",
      "Amazing experience!"
    ]
  }'
```

### 4. 运行测试

```bash
pytest tests/ -v
```

## 挑战和扩展

### 挑战 1：用户注册

添加注册新用户的 endpoint：
```python
@app.post("/register")
async def register_user(username: str, password: str, email: str):
    # 验证输入
    # 哈希密码
    # 存储用户
    # 返回成功消息
    pass
```

### 挑战 2：模型版本化

支持多个模型版本：
```python
@app.post("/api/v1/predict")
async def predict(
    input_data: PredictionInput,
    model_version: str = "1.0.0",
    current_user: User = Depends(get_current_active_user)
):
    # 加载特定模型版本
    # 进行预测
    pass
```

### 挑战 3：预测历史

存储预测历史用于分析：
```python
class PredictionHistory:
    def __init__(self):
        self.history = []

    def add(self, user: str, input: str, prediction: str):
        self.history.append({
            "user": user,
            "input": input,
            "prediction": prediction,
            "timestamp": datetime.utcnow()
        })

@app.get("/api/v1/predictions/history")
async def get_prediction_history(
    current_user: User = Depends(get_current_active_user)
):
    # 返回用户的预测历史
    pass
```

### 挑战 4：模型指标

添加模型性能指标 endpoint：
```python
@app.get("/api/v1/model/metrics")
async def get_model_metrics():
    return {
        "accuracy": 0.92,
        "precision": 0.90,
        "recall": 0.94,
        "f1_score": 0.92
    }
```

### 挑战 5：异步预测

使用 async/await 提高性能：
```python
@app.post("/api/v1/predict")
async def predict_async(
    input_data: PredictionInput,
    background_tasks: BackgroundTasks,
    current_user: User = Depends(get_current_active_user)
):
    # 异步进行预测
    # 在后台记录到数据库
    pass
```

## 常见问题及解决方案

### 问题 1：模型未加载

**问题：** 模型文件未找到或损坏

**解决方案：**
- 检查配置中的模型路径
- 验证模型文件存在：`ls -l models/`
- 重新运行训练脚本

### 问题 2：JWT 令牌错误

**问题：** "Could not validate credentials"

**解决方案：**
- 检查 .env 中的 SECRET_KEY
- 验证令牌在 Authorization 头部中传递
- 检查令牌是否已过期

### 问题 3：认证始终失败

**问题：** 密码验证失败

**解决方案：**
- 生成正确的密码哈希：
  ```python
  from passlib.context import CryptContext
  pwd_context = CryptContext(schemes=["bcrypt"])
  print(pwd_context.hash("secret123"))
  ```
- 更新 fake_users_db 中的哈希

## 交付物

- ✅ 工作的 ML 模型服务 API
- ✅ JWT 认证实现
- ✅ 单次和批量预测 endpoint
- ✅ 模型统计信息跟踪
- ✅ 全面测试套件
- ✅ 文档和使用示例

## 关键要点

1. **认证：** JWT 令牌用于无状态认证
2. **模型加载：** 在启动时一次性加载模型以提高效率
3. **依赖注入：** 用于模型和用户管理
4. **错误处理：** 模型失败时优雅降级
5. **测试：** 测试已认证和未认证的场景

## 下一步

- 练习 03：添加速率限制和高级安全
- 练习 04：使用 Docker 部署 API
- 练习 05：添加监控和日志

---

**预计时间：** 3-4 小时
**难度：** 中级
**重点：** ML 模型服务、JWT 认证、生产模式
