# 练习 03：速率限制和 CORS 配置

## 概述

使用生产就绪的功能增强你的 API，包括防止滥用的速率限制、面向 Web 应用程序的 CORS 配置以及安全头部。学习如何在允许合法跨域请求的同时保护你的 ML API 免受过量使用。

**难度：** 中级
**预计时间：** 2-3 小时
**先决条件：**
- 已完成练习 01 和 02
- 讲座 03：认证和安全
- 理解 HTTP 头部

## 学习目标

完成此练习后，你将：
- 实现速率限制以防止 API 滥用
- 为跨域请求配置 CORS
- 添加安全头部
- 基于用户角色创建分层速率限制
- 监控和记录速率限制违规
- 优雅地处理速率限制错误

## 场景

你的 ML API 越来越受欢迎，你需要：
- 通过限制请求速率来防止滥用
- 允许你的前端应用程序发起请求（不同域）
- 为免费用户和高级用户实现不同的速率限制
- 为生产部署添加安全头部

## 项目设置

### 安装依赖

```bash
pip install slowapi
pip install redis  # 可选：用于分布式速率限制
```

## 第一部分：基本速率限制

**文件：app/rate_limiter.py**

```python
from slowapi import Limiter
from slowapi.util import get_remote_address
from fastapi import Request
from typing import Callable

# TODO: 创建基本限制器
# limiter = Limiter(key_func=get_remote_address)

# TODO: 创建使用 JWT 中用户 ID 的自定义键函数
def get_user_identifier(request: Request) -> str:
    """
    获取用于速率限制的用户标识符。
    如果已认证则使用 JWT 中的用户 ID，否则使用 IP 地址。
    """
    # TODO: 检查 Authorization 头部是否存在
    # TODO: 如果存在，从 JWT 中提取用户 ID
    # TODO: 否则，返回 IP 地址
    # 提示：使用 get_remote_address(request) 获取 IP
    pass

# TODO: 创建带有自定义键函数的限制器
# user_limiter = Limiter(key_func=get_user_identifier)
```

## 第二部分：对 Endpoint 应用速率限制

**文件：app/main.py**（更新）

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from slowapi.middleware import SlowAPIMiddleware
from fastapi import FastAPI, Request

from app.rate_limiter import limiter, user_limiter

app = FastAPI(title="ML API with Rate Limiting")

# TODO: 配置速率限制
# app.state.limiter = limiter
# app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)
# app.add_middleware(SlowAPIMiddleware)

@app.get("/")
@limiter.limit("100/minute")
async def root(request: Request):
    """带速率限制的根 endpoint"""
    # TODO: 返回欢迎消息
    pass

@app.post("/api/v1/predict")
@limiter.limit("10/minute")  # 限制昂贵的 ML 预测
async def predict(
    request: Request,
    input_data: PredictionInput,
    current_user: User = Depends(get_current_active_user)
):
    """带速率限制的预测 endpoint"""
    # TODO: 进行预测
    # TODO: 返回结果
    pass

@app.post("/api/v1/predict/batch")
@limiter.limit("2/minute")  # 对批量操作更严格的限制
async def predict_batch(
    request: Request,
    input_data: BatchPredictionInput,
    current_user: User = Depends(get_current_active_user)
):
    """带严格速率限制的批量预测"""
    # TODO: 处理批量
    # TODO: 返回结果
    pass

# TODO: 添加不带速率限制的 endpoint（健康检查）
@app.get("/health")
async def health_check():
    """健康检查 - 无速率限制"""
    pass
```

## 第三部分：分层速率限制

**文件：app/models.py**（更新）

```python
from enum import Enum

class UserTier(str, Enum):
    FREE = "free"
    PRO = "pro"
    ENTERPRISE = "enterprise"

class User(BaseModel):
    username: str
    email: str
    tier: UserTier = UserTier.FREE
    disabled: bool = False
```

**文件：app/rate_limiter.py**（更新）

```python
from app.auth import get_current_user
from app.models import UserTier
from fastapi import Depends

# TODO: 定义每层的速率限制
RATE_LIMITS = {
    UserTier.FREE: "10/minute",
    UserTier.PRO: "100/minute",
    UserTier.ENTERPRISE: "1000/minute"
}

def get_rate_limit_for_user(user: User) -> str:
    """根据用户层级获取速率限制"""
    # TODO: 从 RATE_LIMITS 字典中返回速率限制
    pass

# TODO: 创建动态速率限制器
class DynamicRateLimiter:
    """根据用户层级调整的速率限制器"""

    def __init__(self):
        self.limiters = {
            tier: Limiter(key_func=get_user_identifier)
            for tier in UserTier
        }

    async def check_rate_limit(
        self,
        request: Request,
        user: User
    ):
        """根据用户层级检查速率限制"""
        # TODO: 获取适用于用户层级的限制器
        # TODO: 检查速率限制
        # TODO: 如果超过则抛出 RateLimitExceeded
        pass
```

## 第四部分：CORS 配置

**文件：app/main.py**（更新）

```python
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# TODO: 配置 CORS
# 根据环境确定允许的来源
import os

if os.getenv("ENVIRONMENT") == "production":
    # TODO: 为生产环境设置特定的允许来源
    allowed_origins = [
        "https://your-frontend.com",
        "https://app.your-frontend.com"
    ]
else:
    # TODO: 在开发环境中允许所有来源（谨慎使用！）
    allowed_origins = ["*"]

# TODO: 添加 CORS 中间件
app.add_middleware(
    CORSMiddleware,
    allow_origins=allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["*"],
    max_age=3600,
)
```

## 第五部分：安全头部中间件

**文件：app/security.py**

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    """向所有响应添加安全头部"""

    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        # TODO: 添加安全头部
        # X-Frame-Options: 防止点击劫持
        # response.headers["X-Frame-Options"] = "DENY"

        # X-Content-Type-Options: 防止 MIME 嗅探
        # response.headers["X-Content-Type-Options"] = "nosniff"

        # X-XSS-Protection: 启用 XSS 过滤器
        # response.headers["X-XSS-Protection"] = "1; mode=block"

        # Referrer-Policy: 控制 referrer 信息
        # response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        # Content-Security-Policy: 限制资源加载
        # response.headers["Content-Security-Policy"] = "default-src 'self'"

        # Strict-Transport-Security: 强制 HTTPS（仅在使用 HTTPS 时）
        # if request.url.scheme == "https":
        #     response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

        return response
```

**在 main.py 中应用中间件：**

```python
from app.security import SecurityHeadersMiddleware

app.add_middleware(SecurityHeadersMiddleware)
```

## 第六部分：速率限制信息头部

**文件：app/rate_limiter.py**（更新）

```python
from fastapi import Response

def add_rate_limit_headers(
    response: Response,
    limit: int,
    remaining: int,
    reset_time: int
):
    """向响应头部添加速率限制信息"""
    # TODO: 添加 X-RateLimit-Limit 头部
    # response.headers["X-RateLimit-Limit"] = str(limit)

    # TODO: 添加 X-RateLimit-Remaining 头部
    # response.headers["X-RateLimit-Remaining"] = str(remaining)

    # TODO: 添加 X-RateLimit-Reset 头部（Unix 时间戳）
    # response.headers["X-RateLimit-Reset"] = str(reset_time)

    pass
```

## 第七部分：自定义速率限制错误响应

**文件：app/main.py**（更新）

```python
from fastapi import Request, status
from fastapi.responses import JSONResponse
from slowapi.errors import RateLimitExceeded
import time

@app.exception_handler(RateLimitExceeded)
async def rate_limit_handler(request: Request, exc: RateLimitExceeded):
    """速率限制超出错误的自定义处理器"""

    # TODO: 计算重试时间
    # retry_after = 60  # 秒

    # TODO: 创建自定义错误响应
    response = JSONResponse(
        status_code=status.HTTP_429_TOO_MANY_REQUESTS,
        content={
            "error": "Rate limit exceeded",
            "message": "Too many requests. Please try again later.",
            "retry_after": 60,
            "documentation": "https://docs.your-api.com/rate-limits"
        }
    )

    # TODO: 添加 Retry-After 头部
    # response.headers["Retry-After"] = str(retry_after)

    return response
```

## 第八部分：测试 CORS 和速率限制

**文件：tests/test_rate_limit.py**

```python
from fastapi.testclient import TestClient
from app.main import app
import time

client = TestClient(app)

def test_rate_limit_enforced():
    """测试速率限制是否生效"""
    # TODO: 获取认证令牌
    # TODO: 发起达到限制的请求
    # TODO: 验证全部成功 (status 200)
    # TODO: 再发起一个请求
    # TODO: 验证失败并返回 429
    pass

def test_rate_limit_headers():
    """测试速率限制头部是否存在"""
    # TODO: 发起一个请求
    # TODO: 检查 X-RateLimit-Limit 头部
    # TODO: 检查 X-RateLimit-Remaining 头部
    # TODO: 验证头部值正确
    pass

def test_rate_limit_resets():
    """测试速率限制在时间窗口后重置"""
    # TODO: 耗尽速率限制
    # TODO: 等待重置周期
    # TODO: 验证请求再次可用
    pass

def test_different_limits_per_tier():
    """测试不同用户层级有不同的限制"""
    # TODO: 以免费用户身份登录
    # TODO: 发起 10 个请求（免费层限制）
    # TODO: 验证第 11 个请求失败

    # TODO: 以专业用户身份登录
    # TODO: 发起 100 个请求（专业层限制）
    # TODO: 验证全部成功
    pass

def test_cors_headers():
    """测试 CORS 头部是否存在"""
    # TODO: 发起 OPTIONS 请求
    # TODO: 检查 Access-Control-Allow-Origin
    # TODO: 检查 Access-Control-Allow-Methods
    # TODO: 检查 Access-Control-Allow-Headers
    pass

def test_security_headers():
    """测试安全头部是否存在"""
    # TODO: 发起任意请求
    # TODO: 检查 X-Frame-Options
    # TODO: 检查 X-Content-Type-Options
    # TODO: 检查 X-XSS-Protection
    # TODO: 检查 Content-Security-Policy
    pass

def test_rate_limit_error_format():
    """测试速率限制错误响应格式"""
    # TODO: 耗尽速率限制
    # TODO: 发起超出限制的请求
    # TODO: 验证响应包含错误消息
    # TODO: 验证响应包含 retry_after
    # TODO: 检查 Retry-After 头部
    pass
```

## 第九部分：前端 CORS 测试

创建一个简单的 HTML 文件来测试 CORS：

**文件：test_cors.html**

```html
<!DOCTYPE html>
<html>
<head>
    <title>CORS Test</title>
</head>
<body>
    <h1>API CORS Test</h1>
    <button onclick="testAPI()">Test API</button>
    <div id="result"></div>

    <script>
        async function testAPI() {
            const resultDiv = document.getElementById('result');

            try {
                // TODO: 向你的 API 发起 fetch 请求
                const response = await fetch('http://localhost:8000/api/v1/predict', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                        'Authorization': 'Bearer YOUR_TOKEN'
                    },
                    body: JSON.stringify({
                        text: 'This is a test'
                    })
                });

                if (response.ok) {
                    const data = await response.json();
                    resultDiv.innerHTML = `<p style="color: green">Success! Prediction: ${data.prediction}</p>`;
                } else {
                    resultDiv.innerHTML = `<p style="color: red">Error: ${response.status}</p>`;
                }
            } catch (error) {
                resultDiv.innerHTML = `<p style="color: red">CORS Error: ${error.message}</p>`;
            }
        }
    </script>
</body>
</html>
```

**测试 CORS：**
1. 在浏览器中打开 HTML 文件
2. 点击 "Test API" 按钮
3. 检查浏览器控制台是否有 CORS 错误
4. 验证是否收到响应

## 第十部分：监控速率限制使用情况

**文件：app/monitoring.py**

```python
from collections import defaultdict
from datetime import datetime
from typing import Dict

class RateLimitMonitor:
    """监控和记录速率限制违规"""

    def __init__(self):
        self.violations = defaultdict(list)
        self.requests_by_user = defaultdict(int)

    def log_violation(self, user_id: str, endpoint: str):
        """记录速率限制违规"""
        # TODO: 将违规添加到列表
        # self.violations[user_id].append({
        #     "endpoint": endpoint,
        #     "timestamp": datetime.utcnow()
        # })
        pass

    def increment_request_count(self, user_id: str):
        """增加用户请求计数"""
        # TODO: 增加计数器
        pass

    def get_user_violations(self, user_id: str) -> list:
        """获取特定用户的违规"""
        # TODO: 返回违规列表
        pass

    def get_top_users(self, limit: int = 10) -> Dict:
        """获取请求数最多的用户"""
        # TODO: 按请求数排序用户
        # TODO: 返回前 N 名用户
        pass

# 全局监控实例
monitor = RateLimitMonitor()
```

## 挑战和扩展

### 挑战 1：基于 Redis 的速率限制

使用 Redis 实现分布式速率限制：

```python
from slowapi.middleware import SlowAPIMiddleware
from slowapi import Limiter
from slowapi.util import get_remote_address
import redis

redis_client = redis.Redis(host='localhost', port=6379, db=0)

limiter = Limiter(
    key_func=get_remote_address,
    storage_uri="redis://localhost:6379"
)
```

### 挑战 2：动态速率限制调整

根据 API 负载调整速率限制：

```python
def get_dynamic_rate_limit() -> str:
    cpu_usage = get_cpu_usage()
    if cpu_usage > 80:
        return "5/minute"  # 高负载下减少限制
    elif cpu_usage > 50:
        return "20/minute"
    else:
        return "50/minute"
```

### 挑战 3：受信任 IP 的速率限制绕过

允许来自受信任 IP 的无限制请求：

```python
TRUSTED_IPS = ["10.0.0.1", "192.168.1.1"]

def should_check_rate_limit(request: Request) -> bool:
    client_ip = request.client.host
    return client_ip not in TRUSTED_IPS
```

### 挑战 4：速率限制仪表盘

创建查看速率限制统计信息的 endpoint：

```python
@app.get("/admin/rate-limits/stats")
async def get_rate_limit_stats(
    admin_user: User = Depends(require_admin)
):
    return {
        "total_requests": monitor.get_total_requests(),
        "violations": monitor.get_total_violations(),
        "top_users": monitor.get_top_users(10)
    }
```

### 挑战 5：每个 Endpoint 的自定义 CORS

不同 endpoint 的不同 CORS 策略：

```python
from fastapi.middleware.cors import CORSMiddleware

# 管理 endpoint 更严格的 CORS
admin_cors = CORSMiddleware(
    app=app,
    allow_origins=["https://admin.example.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST"],
)

# 公共 endpoint 宽松的 CORS
public_cors = CORSMiddleware(
    app=app,
    allow_origins=["*"],
    allow_credentials=False,
    allow_methods=["GET"],
)
```

## 常见问题及解决方案

### 问题 1：CORS 预检请求失败

**问题：** OPTIONS 请求返回 405

**解决方案：**
- 确保 CORS 中间件在路由之前添加
- 将 OPTIONS 添加到 allowed_methods
- 谨慎使用通配符 (*)

### 问题 2：速率限制不起作用

**问题：** 速率限制未生效

**解决方案：**
- 验证 SlowAPIMiddleware 已添加
- 检查 @limiter.limit() 装饰器是否已应用
- 确保限制器已添加到 app.state

### 问题 3：速率限制头部丢失

**问题：** X-RateLimit 头部不在响应中

**解决方案：**
- 在 endpoint 中手动添加头部
- 使用添加头部的自定义速率限制器
- 检查中间件顺序

## 交付物

- ✅ 速率限制已实现
- ✅ CORS 配置正确
- ✅ 安全头部已添加
- ✅ 基于用户角色的分层速率限制
- ✅ 速率限制的自定义错误响应
- ✅ 所有安全功能的测试
- ✅ 速率限制文档

## 关键要点

1. **速率限制：** 对于防止滥用和确保公平使用至关重要
2. **CORS：** Web 应用程序在不同域时必需
3. **安全头部：** 添加多层保护
4. **分层访问：** 不同用户层级的不同限制
5. **监控：** 跟踪违规和使用模式

## 下一步

- 练习 04：后台任务和异步操作
- 练习 05：使用 Docker 和 NGINX 部署 API
- 学习 API 网关模式

---

**预计时间：** 2-3 小时
**难度：** 中级
**重点：** 速率限制、CORS、安全头部、生产加固
