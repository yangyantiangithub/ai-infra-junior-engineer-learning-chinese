# 讲座 03：API 的认证和安全

## 目录
1. [介绍](#介绍)
2. [安全基础](#安全基础)
3. [认证 vs 授权](#认证-vs-授权)
4. [API Key 认证](#api-key-认证)
5. [JWT (JSON Web Tokens)](#jwt-json-web-tokens)
6. [OAuth2 和 OpenID Connect](#oauth2-和-openid-connect)
7. [在 FastAPI 中实现认证](#在-fastapi-中实现认证)
8. [密码安全](#密码安全)
9. [HTTPS 和传输安全](#https-和传输安全)
10. [CORS (跨域资源共享)](#cors-跨域资源共享)
11. [速率限制和节流](#速率限制和节流)
12. [输入验证和清理](#输入验证和清理)
13. [安全头部](#安全头部)
14. [ML API 的安全](#ml-api-的安全)
15. [最佳实践和清单](#最佳实践和清单)
16. [总结和关键要点](#总结和关键要点)

## 介绍

安全不是可选项——它是任何生产 API 的基本要求。一个安全漏洞可能导致数据泄露、未授权访问、服务中断或更严重的后果。这对于经常处理敏感数据和昂贵计算资源的 ML API 尤其重要。

本讲座涵盖基本的安全概念和构建安全 API 的实用实现策略。

### 学习目标

在本讲座结束时，你将：
- 理解认证和授权的区别
- 实现各种认证方法（API key、JWT、OAuth2）
- 使用适当的哈希技术确保密码安全
- 配置 HTTPS 和传输层安全
- 正确配置 CORS 策略
- 添加速率限制以防止滥用
- 验证和清理用户输入
- 应用安全头部
- 理解 ML 特定的安全问题
- 遵循安全最佳实践

### 先决条件
- 模块 007 - 讲座 01：API 基础
- 模块 007 - 讲座 02：FastAPI 框架
- 对密码学概念的基本理解
- HTTP 协议知识

### 预计时间
4-5 小时（包括动手实现）

## 安全基础

### CIA 三元组

信息安全的基础：

1. **机密性**：只有授权用户可以访问数据
   - 加密（静态数据和传输中的数据）
   - 访问控制
   - 认证

2. **完整性**：数据是准确的且未被篡改
   - 校验和和哈希
   - 数字签名
   - 验证

3. **可用性**：系统在需要时可访问
   - 负载均衡
   - DDoS 防护
   - 速率限制

### 最小权限原则

只授予必要的最低权限：

```python
# 不良：给每个人管理员权限
user_permissions = "admin"

# 良好：细粒度权限
user_permissions = {
    "read_models": True,
    "create_models": False,
    "delete_models": False,
    "predict": True
}
```

### 纵深防御

多层安全：

```
┌─────────────────────────────┐
│ 应用层                      │ ← 输入验证
├─────────────────────────────┤
│ 认证层                      │ ← JWT/OAuth2
├─────────────────────────────┤
│ 网络层                      │ ← 防火墙、VPN
├─────────────────────────────┤
│ 传输层                      │ ← HTTPS/TLS
├─────────────────────────────┤
│ 基础设施层                  │ ← 安全组
└─────────────────────────────┘
```

### 安全设计

从一开始就构建安全，而不是事后添加：

- 使用安全的框架（FastAPI、Django）
- 遵循安全指南（OWASP Top 10）
- 定期安全审计
- 自动化安全测试
- 保持依赖更新

## 认证 vs 授权

这些术语经常混淆，但它们服务于不同的目的：

### 认证

**"你是谁？"**

验证用户或系统的身份。

```python
# 用户提供凭证
username = "alice"
password = "secret123"

# 系统验证身份
if verify_credentials(username, password):
    # 用户已认证
    issue_token(username)
```

**方法：**
- 用户名/密码
- API key
- 令牌 (JWT)
- 证书
- 生物识别

### 授权

**"你能做什么？"**

确定已认证用户被允许做什么。

```python
# 用户已认证为 "alice"
user = get_current_user()  # alice

# 检查授权
if user.has_permission("delete_models"):
    delete_model(model_id)
else:
    raise PermissionDeniedError()
```

**方法：**
- 基于角色的访问控制 (RBAC)
- 基于属性的访问控制 (ABAC)
- 访问控制列表 (ACL)

### 示例流程

```python
from fastapi import FastAPI, Depends, HTTPException
from typing import Optional

app = FastAPI()

# 认证：验证他们是谁
def authenticate_user(api_key: str) -> Optional[User]:
    user = db.get_user_by_api_key(api_key)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid API key")
    return user

# 授权：检查他们能做什么
def authorize_action(user: User, action: str) -> bool:
    return action in user.permissions

@app.delete("/models/{model_id}")
def delete_model(
    model_id: str,
    user: User = Depends(authenticate_user)
):
    # 授权检查
    if not authorize_action(user, "delete_models"):
        raise HTTPException(status_code=403, detail="Insufficient permissions")

    # 执行操作
    db.delete_model(model_id)
    return {"message": "Model deleted"}
```

**关键区别：**

| 方面 | 认证 | 授权 |
|--------|---------------|---------------|
| 问题 | 你是谁？ | 你能做什么？ |
| 过程 | 验证身份 | 检查权限 |
| 状态码 | 401 Unauthorized | 403 Forbidden |
| 顺序 | 先 | 后（认证之后） |
| 示例 | 用密码登录 | 访问管理面板 |

## API Key 认证

API 最简单形式的认证。

### 什么是 API Key？

用于认证请求的唯一标识符：

```
API-Key: sk_live_51H8K2jLkJ...
```

### 优点

- 实现简单
- 用户易于理解
- 不需要会话管理
- 无状态

### 缺点

- 如果泄露，任何人都可以使用
- 没有内置过期机制
- 难以安全轮换
- 关于用户的信息有限

### 实现

**生成 API Key：**

```python
import secrets
import hashlib

def generate_api_key() -> str:
    """生成安全的随机 API key"""
    return secrets.token_urlsafe(32)

def hash_api_key(api_key: str) -> str:
    """哈希 API key 以便存储"""
    return hashlib.sha256(api_key.encode()).hexdigest()

# 用法
api_key = generate_api_key()
print(f"API Key: {api_key}")
# 在数据库中存储哈希值
hashed = hash_api_key(api_key)
```

**验证 API Key：**

```python
from fastapi import FastAPI, HTTPException, Security
from fastapi.security import APIKeyHeader

app = FastAPI()

API_KEY_NAME = "X-API-Key"
api_key_header = APIKeyHeader(name=API_KEY_NAME, auto_error=False)

# 模拟数据库
API_KEYS = {
    "hashed_key_1": {"username": "alice", "permissions": ["read", "write"]},
    "hashed_key_2": {"username": "bob", "permissions": ["read"]}
}

def verify_api_key(api_key: str = Security(api_key_header)):
    if api_key is None:
        raise HTTPException(status_code=401, detail="API key missing")

    hashed = hash_api_key(api_key)
    if hashed not in API_KEYS:
        raise HTTPException(status_code=401, detail="Invalid API key")

    return API_KEYS[hashed]

@app.get("/protected")
def protected_route(user_info: dict = Depends(verify_api_key)):
    return {"message": f"Hello {user_info['username']}"}
```

**使用 API Key（客户端）：**

```python
import requests

headers = {"X-API-Key": "your-api-key-here"}
response = requests.get("http://api.example.com/protected", headers=headers)
```

### 最佳实践

1. **使用 HTTPS**：明文 HTTP 中的 API key 可能被截获
2. **哈希 key**：在数据库中存储哈希版本
3. **前缀 key**：使用前缀标识 key 类型（`sk_live_`、`pk_test_`）
4. **轮换**：提供轮换 key 的机制
5. **作用域**：限制 key 的权限
6. **监控**：记录 API key 使用并检测滥用
7. **速率限制**：防止暴力破解攻击

## JWT (JSON Web Tokens)

JWT 是一种紧凑的、URL 安全的方式，用于在双方之间表示声明。

### 什么是 JWT？

JWT 由三部分组成：
```
header.payload.signature
```

示例：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**组成部分：**

1. **Header**：算法和令牌类型
   ```json
   {
     "alg": "HS256",
     "typ": "JWT"
   }
   ```

2. **Payload**：声明（数据）
   ```json
   {
     "sub": "1234567890",
     "name": "Alice Smith",
     "role": "admin",
     "exp": 1634567890
   }
   ```

3. **Signature**：验证完整性
   ```
   HMACSHA256(
     base64UrlEncode(header) + "." +
     base64UrlEncode(payload),
     secret
   )
   ```

### 优点

- **无状态**：不需要服务器端会话存储
- **紧凑**：易于在 URL、头部中传输
- **自包含**：包含所有用户信息
- **安全**：经过加密签名
- **可扩展**：跨多个服务器工作

### 缺点

- **无法撤销**：在过期前一直有效
- **大小**：比简单的 API key 大
- **密钥管理**：必须保护签名密钥

### 标准声明

- **iss** (issuer)：谁创建了令牌
- **sub** (subject)：用户 ID
- **aud** (audience)：令牌面向谁
- **exp** (expiration)：令牌何时过期
- **iat** (issued at)：令牌何时创建
- **nbf** (not before)：令牌在此时间之前无效

### 在 FastAPI 中的实现

**安装依赖：**
```bash
pip install python-jose[cryptography]
pip install passlib[bcrypt]
pip install python-multipart
```

**创建 JWT 工具函数：**

```python
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext

# 配置
SECRET_KEY = "your-secret-key-keep-it-secret"  # 使用环境变量！
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    """创建 JWT 访问令牌"""
    to_encode = data.copy()

    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)

    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt

def verify_token(token: str) -> dict:
    """验证并解码 JWT 令牌"""
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        return payload
    except JWTError:
        raise HTTPException(
            status_code=401,
            detail="Could not validate credentials",
            headers={"WWW-Authenticate": "Bearer"}
        )
```

**登录 endpoint：**

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel

app = FastAPI()

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

class Token(BaseModel):
    access_token: str
    token_type: str

class User(BaseModel):
    username: str
    email: str
    disabled: bool = False

# 模拟用户数据库
fake_users_db = {
    "alice": {
        "username": "alice",
        "email": "alice@example.com",
        "hashed_password": "$2b$12$...",  # bcrypt 哈希
        "disabled": False
    }
}

def authenticate_user(username: str, password: str):
    """验证用户名和密码"""
    user = fake_users_db.get(username)
    if not user:
        return False
    if not pwd_context.verify(password, user["hashed_password"]):
        return False
    return user

@app.post("/token", response_model=Token)
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    """登录并获取访问令牌"""
    user = authenticate_user(form_data.username, form_data.password)

    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"}
        )

    # 创建令牌
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": user["username"]},
        expires_delta=access_token_expires
    )

    return {"access_token": access_token, "token_type": "bearer"}
```

**受保护的 endpoint：**

```python
async def get_current_user(token: str = Depends(oauth2_scheme)):
    """从令牌获取当前用户"""
    payload = verify_token(token)
    username: str = payload.get("sub")

    if username is None:
        raise HTTPException(status_code=401, detail="Invalid token")

    user = fake_users_db.get(username)
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")

    return User(**user)

@app.get("/users/me", response_model=User)
async def read_users_me(current_user: User = Depends(get_current_user)):
    """获取当前用户资料"""
    return current_user

@app.get("/protected")
async def protected_route(current_user: User = Depends(get_current_user)):
    return {"message": f"Hello {current_user.username}"}
```

**客户端用法：**

```python
import requests

# 1. 登录获取令牌
response = requests.post(
    "http://localhost:8000/token",
    data={"username": "alice", "password": "secret"}
)
token = response.json()["access_token"]

# 2. 使用令牌进行认证请求
headers = {"Authorization": f"Bearer {token}"}
response = requests.get("http://localhost:8000/users/me", headers=headers)
print(response.json())
```

### 刷新令牌

访问令牌应该短命。使用刷新令牌进行长期访问：

```python
def create_refresh_token(data: dict):
    """创建长寿命刷新令牌"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=7)
    to_encode.update({"exp": expire, "type": "refresh"})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

@app.post("/token/refresh")
async def refresh_access_token(refresh_token: str):
    """使用刷新令牌获取新的访问令牌"""
    payload = verify_token(refresh_token)

    if payload.get("type") != "refresh":
        raise HTTPException(status_code=401, detail="Invalid refresh token")

    # 创建新的访问令牌
    access_token = create_access_token(data={"sub": payload.get("sub")})
    return {"access_token": access_token, "token_type": "bearer"}
```

## OAuth2 和 OpenID Connect

### OAuth2 概述

OAuth2 是一个授权框架，使第三方应用程序能够获得对服务的有限访问权限。

**使用场景：**
- "使用 Google 登录"
- "使用 GitHub 登录"
- 第三方应用程序访问

**角色：**
1. **资源所有者**：用户
2. **客户端**：请求访问的应用程序
3. **授权服务器**：颁发令牌
4. **资源服务器**：托管受保护的资源

**流程：**

```
┌──────────┐                               ┌─────────────┐
│  Client  │                               │   Resource  │
│          │                               │    Owner    │
└────┬─────┘                               └──────┬──────┘
     │                                             │
     │  1. Authorization Request                   │
     │────────────────────────────────────────────>│
     │                                             │
     │  2. Authorization Grant                     │
     │<────────────────────────────────────────────│
     │                                             │
     │  3. Authorization Grant                     │
     │────────────────────────────>┌──────────────┴───┐
     │                             │  Authorization   │
     │  4. Access Token            │     Server       │
     │<────────────────────────────┤                  │
     │                             └──────────────────┘
     │  5. Access Token            ┌──────────────────┐
     │────────────────────────────>│   Resource       │
     │                             │    Server        │
     │  6. Protected Resource      │                  │
     │<────────────────────────────┤                  │
     │                             └──────────────────┘
```

### FastAPI OAuth2 实现

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import OAuth2AuthorizationCodeBearer

app = FastAPI()

oauth2_scheme = OAuth2AuthorizationCodeBearer(
    authorizationUrl="https://provider.com/oauth/authorize",
    tokenUrl="https://provider.com/oauth/token"
)

@app.get("/protected")
async def protected_route(token: str = Depends(oauth2_scheme)):
    # 使用提供商验证令牌
    user_info = verify_with_provider(token)
    return {"user": user_info}
```

### OpenID Connect

OAuth2 的扩展，用于认证：

```python
from authlib.integrations.starlette_client import OAuth
from starlette.config import Config

config = Config('.env')
oauth = OAuth(config)

oauth.register(
    name='google',
    client_id=config('GOOGLE_CLIENT_ID'),
    client_secret=config('GOOGLE_CLIENT_SECRET'),
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid email profile'}
)

@app.get('/login/google')
async def login_google(request: Request):
    redirect_uri = request.url_for('auth_google')
    return await oauth.google.authorize_redirect(request, redirect_uri)

@app.get('/auth/google')
async def auth_google(request: Request):
    token = await oauth.google.authorize_access_token(request)
    user = await oauth.google.parse_id_token(request, token)
    return {"user": user}
```

## 密码安全

### 永远不要存储明文密码

**不良：**
```python
users = {
    "alice": {"password": "secret123"}  # 永远不要这样做！
}
```

**良好：**
```python
users = {
    "alice": {"password_hash": "$2b$12$KIX..."}  # 哈希后的密码
}
```

### 哈希 vs 加密

| 方面 | 哈希 | 加密 |
|--------|---------|------------|
| 可逆 | 否 | 是 |
| 目的 | 验证完整性 | 保护数据 |
| 用例 | 密码 | 敏感数据 |
| 算法 | bcrypt、argon2 | AES、RSA |

### 使用 bcrypt

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# 哈希密码
password = "user_password_123"
hashed = pwd_context.hash(password)
print(hashed)
# $2b$12$KIXl.W9T8qI9GqN3z2GKsOzp8qjZ6q...

# 验证密码
is_correct = pwd_context.verify("user_password_123", hashed)
print(is_correct)  # True

is_correct = pwd_context.verify("wrong_password", hashed)
print(is_correct)  # False
```

### 密码要求

```python
from pydantic import BaseModel, validator
import re

class UserCreate(BaseModel):
    username: str
    password: str

    @validator("password")
    def validate_password(cls, v):
        if len(v) < 8:
            raise ValueError("Password must be at least 8 characters")
        if not re.search(r"[A-Z]", v):
            raise ValueError("Password must contain uppercase letter")
        if not re.search(r"[a-z]", v):
            raise ValueError("Password must contain lowercase letter")
        if not re.search(r"\d", v):
            raise ValueError("Password must contain digit")
        if not re.search(r"[!@#$%^&*(),.?\":{}|<>]", v):
            raise ValueError("Password must contain special character")
        return v
```

### 安全密码重置

```python
import secrets
from datetime import datetime, timedelta

def create_password_reset_token(email: str) -> str:
    """创建安全的密码重置令牌"""
    token = secrets.token_urlsafe(32)

    # 存储令牌及过期时间
    reset_tokens[token] = {
        "email": email,
        "expires": datetime.utcnow() + timedelta(hours=1)
    }

    return token

@app.post("/password-reset/request")
async def request_password_reset(email: str):
    """请求密码重置"""
    user = get_user_by_email(email)
    if not user:
        # 不要透露用户是否存在
        return {"message": "If email exists, reset link will be sent"}

    token = create_password_reset_token(email)
    send_reset_email(email, token)

    return {"message": "If email exists, reset link will be sent"}

@app.post("/password-reset/confirm")
async def confirm_password_reset(token: str, new_password: str):
    """使用令牌重置密码"""
    if token not in reset_tokens:
        raise HTTPException(status_code=400, detail="Invalid token")

    reset_info = reset_tokens[token]

    if datetime.utcnow() > reset_info["expires"]:
        raise HTTPException(status_code=400, detail="Token expired")

    # 更新密码
    update_user_password(reset_info["email"], new_password)

    # 使令牌失效
    del reset_tokens[token]

    return {"message": "Password reset successful"}
```

## HTTPS 和传输安全

### 为什么需要 HTTPS？

没有 HTTPS：
- 密码以明文发送
- API key 对攻击者可见
- 数据可能在传输中被修改
- 没有服务器认证

有了 HTTPS：
- 通信加密
- 数据完整性
- 服务器认证
- 客户端信任

### TLS/SSL 证书

**开发环境（自签名）：**
```bash
# 生成自签名证书
openssl req -x509 -newkey rsa:4096 -nodes \
  -out cert.pem -keyout key.pem -days 365
```

**生产环境：**
使用 Let's Encrypt（免费）或商业 CA。

### 使用 HTTPS 运行 FastAPI

```python
# 使用 uvicorn 配合 SSL
import uvicorn

if __name__ == "__main__":
    uvicorn.run(
        "main:app",
        host="0.0.0.0",
        port=443,
        ssl_keyfile="./key.pem",
        ssl_certfile="./cert.pem"
    )
```

或通过命令行：
```bash
uvicorn main:app --ssl-keyfile=./key.pem --ssl-certfile=./cert.pem
```

### 强制 HTTPS

```python
from fastapi import FastAPI, Request
from fastapi.responses import RedirectResponse

app = FastAPI()

@app.middleware("http")
async def force_https(request: Request, call_next):
    if request.url.scheme != "https" and not request.url.hostname in ["localhost", "127.0.0.1"]:
        url = request.url.replace(scheme="https")
        return RedirectResponse(url, status_code=301)
    return await call_next(request)
```

### 安全头部

```python
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from starlette.middleware.httpsredirect import HTTPSRedirectMiddleware

# 强制 HTTPS
app.add_middleware(HTTPSRedirectMiddleware)

# 限制允许的主机
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts=["example.com", "*.example.com"]
)
```

## CORS (跨域资源共享)

### 什么是 CORS？

限制哪些来源可以访问你的 API 的安全功能。

**场景：**
- 你的 API：`https://api.example.com`
- 前端：`https://app.example.com`
- 不同的来源（不同的子域名）

没有 CORS 配置，浏览器会阻止请求。

### CORS 头部

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Max-Age: 3600
```

### 在 FastAPI 中实现 CORS

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# 配置 CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=[
        "https://app.example.com",
        "https://dashboard.example.com"
    ],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE"],
    allow_headers=["Content-Type", "Authorization"],
)
```

### 开发环境配置

```python
# 警告：仅用于开发环境！
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # 允许所有来源
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 生产环境配置

```python
import os

# 从环境加载允许的来源
ALLOWED_ORIGINS = os.getenv("ALLOWED_ORIGINS", "").split(",")

app.add_middleware(
    CORSMiddleware,
    allow_origins=ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allow_headers=["Content-Type", "Authorization", "X-API-Key"],
    max_age=3600,
)
```

## 速率限制和节流

### 为什么需要速率限制？

- 防止滥用和 DoS 攻击
- 确保公平的资源分配
- 保护基础设施免受过载
- 盈利（基于层级的限制）

### 策略

1. **固定窗口**：每个时间窗口 X 个请求
2. **滑动窗口**：比固定窗口更精确
3. **令牌桶**：允许突发流量
4. **漏桶**：平滑流量

### 使用 SlowAPI 实现

```bash
pip install slowapi
```

```python
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded
from fastapi import FastAPI, Request

app = FastAPI()

# 初始化限制器
limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# 对 endpoint 应用速率限制
@app.post("/predict")
@limiter.limit("10/minute")
async def predict(request: Request, input_data: PredictionInput):
    return {"prediction": "result"}

# 不同 endpoint 的不同限制
@app.get("/expensive-operation")
@limiter.limit("2/minute")
async def expensive_operation(request: Request):
    return {"status": "processing"}

# 多重限制
@app.get("/api/data")
@limiter.limit("100/hour")
@limiter.limit("1000/day")
async def get_data(request: Request):
    return {"data": []}
```

### 基于用户的速率限制

```python
def get_user_id(request: Request) -> str:
    """从请求中提取用户 ID"""
    token = request.headers.get("Authorization", "").replace("Bearer ", "")
    if token:
        payload = verify_token(token)
        return payload.get("sub", "anonymous")
    return "anonymous"

limiter = Limiter(key_func=get_user_id)

@app.post("/predict")
@limiter.limit("100/minute")  # 按用户，而不是按 IP
async def predict(request: Request, input_data: PredictionInput):
    return {"prediction": "result"}
```

### 基于用户层级的自定义速率限制

```python
from functools import wraps

def get_user_rate_limit(user: User) -> str:
    """根据用户层级获取速率限制"""
    limits = {
        "free": "10/minute",
        "pro": "100/minute",
        "enterprise": "1000/minute"
    }
    return limits.get(user.tier, "10/minute")

@app.post("/predict")
async def predict(
    request: Request,
    input_data: PredictionInput,
    current_user: User = Depends(get_current_user)
):
    # 应用自定义限制
    limit = get_user_rate_limit(current_user)
    # 使用自定义限制器逻辑
    return {"prediction": "result"}
```

### 速率限制头部

在响应中包含速率限制信息：

```python
from fastapi import Response

@app.get("/api/data")
async def get_data(response: Response):
    # 添加速率限制头部
    response.headers["X-RateLimit-Limit"] = "100"
    response.headers["X-RateLimit-Remaining"] = "95"
    response.headers["X-RateLimit-Reset"] = "1634567890"

    return {"data": []}
```

## 输入验证和清理

### SQL 注入防护

**不良（易受攻击）：**
```python
# 永远不要这样做！
query = f"SELECT * FROM users WHERE username = '{username}'"
db.execute(query)
```

**良好（参数化查询）：**
```python
query = "SELECT * FROM users WHERE username = ?"
db.execute(query, (username,))
```

### XSS (跨站脚本) 防护

```python
from markupsafe import escape

@app.post("/comments")
async def create_comment(comment: str):
    # 转义 HTML
    safe_comment = escape(comment)
    db.save_comment(safe_comment)
    return {"comment": safe_comment}
```

### 命令注入防护

```python
import subprocess
import shlex

# 不良
filename = user_input
subprocess.run(f"cat {filename}", shell=True)  # 危险！

# 良好
filename = shlex.quote(user_input)
subprocess.run(["cat", filename])  # 安全
```

### 路径遍历防护

```python
from pathlib import Path

@app.get("/files/{filename}")
async def get_file(filename: str):
    # 防止路径遍历
    base_dir = Path("/safe/directory")
    file_path = (base_dir / filename).resolve()

    # 确保文件在基础目录内
    if not file_path.is_relative_to(base_dir):
        raise HTTPException(status_code=400, detail="Invalid file path")

    return FileResponse(file_path)
```

### Pydantic 验证

```python
from pydantic import BaseModel, validator, constr, conint
from typing import List

class ModelConfig(BaseModel):
    name: constr(min_length=1, max_length=100, regex="^[a-zA-Z0-9-_]+$")
    batch_size: conint(ge=1, le=128)
    tags: List[constr(max_length=50)]

    @validator("tags")
    def validate_tags(cls, v):
        if len(v) > 10:
            raise ValueError("Maximum 10 tags allowed")
        return v
```

## 安全头部

### 重要的安全头部

```python
from fastapi import FastAPI
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        response = await call_next(request)

        # 防止点击劫持
        response.headers["X-Frame-Options"] = "DENY"

        # XSS 防护
        response.headers["X-Content-Type-Options"] = "nosniff"

        # 控制 referrer 信息
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        # 内容安全策略
        response.headers["Content-Security-Policy"] = "default-src 'self'"

        # 严格传输安全（仅 HTTPS）
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

        # 权限策略
        response.headers["Permissions-Policy"] = "geolocation=(), microphone=()"

        return response

app = FastAPI()
app.add_middleware(SecurityHeadersMiddleware)
```

## ML API 的安全

### 模型安全

**1. 模型文件保护：**
```python
import os
from pathlib import Path

MODEL_DIR = Path("/secure/models")

def load_model(model_id: str):
    # 验证模型 ID
    if not model_id.isalnum():
        raise ValueError("Invalid model ID")

    model_path = MODEL_DIR / f"{model_id}.pkl"

    # 检查文件存在且在允许的目录内
    if not model_path.is_file() or not model_path.is_relative_to(MODEL_DIR):
        raise FileNotFoundError("Model not found")

    # 设置限制性权限（仅所有者读写）
    os.chmod(model_path, 0o600)

    return pickle.load(open(model_path, "rb"))
```

**2. ML 模型的输入验证：**
```python
from pydantic import BaseModel, validator
import numpy as np

class ImagePredictionInput(BaseModel):
    image_data: str  # Base64 编码
    max_size_mb: int = 10

    @validator("image_data")
    def validate_image(cls, v):
        import base64

        try:
            decoded = base64.b64decode(v)
        except Exception:
            raise ValueError("Invalid base64 encoding")

        # 检查大小
        size_mb = len(decoded) / (1024 * 1024)
        if size_mb > cls.max_size_mb:
            raise ValueError(f"Image too large: {size_mb:.2f}MB > {cls.max_size_mb}MB")

        return v
```

**3. 资源限制：**
```python
from contextlib import contextmanager
import signal

@contextmanager
def time_limit(seconds: int):
    """上下文管理器，限制执行时间"""
    def signal_handler(signum, frame):
        raise TimeoutError("Prediction timed out")

    signal.signal(signal.SIGALRM, signal_handler)
    signal.alarm(seconds)
    try:
        yield
    finally:
        signal.alarm(0)

@app.post("/predict")
async def predict(input_data: PredictionInput):
    try:
        with time_limit(5):  # 5 秒超时
            prediction = model.predict(input_data.features)
            return {"prediction": prediction}
    except TimeoutError:
        raise HTTPException(status_code=504, detail="Prediction timeout")
```

**4. 对抗性输入检测：**
```python
def detect_adversarial_input(input_data: np.ndarray) -> bool:
    """基础对抗性输入检测"""
    # 检查极端值
    if np.any(np.abs(input_data) > 1000):
        return True

    # 检查 NaN 或 Inf
    if np.any(~np.isfinite(input_data)):
        return True

    return False

@app.post("/predict")
async def predict(input_data: PredictionInput):
    features = np.array(input_data.features)

    if detect_adversarial_input(features):
        raise HTTPException(status_code=400, detail="Invalid input detected")

    return model.predict(features)
```

### 数据隐私

**1. PII 保护：**
```python
import re

def mask_pii(text: str) -> str:
    """屏蔽个人身份信息"""
    # 邮箱
    text = re.sub(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b', '[EMAIL]', text)

    # 电话
    text = re.sub(r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b', '[PHONE]', text)

    # 社会安全号码
    text = re.sub(r'\b\d{3}-\d{2}-\d{4}\b', '[SSN]', text)

    return text

@app.post("/analyze")
async def analyze_text(text: str):
    # 在记录或处理前屏蔽 PII
    safe_text = mask_pii(text)
    logger.info(f"Processing: {safe_text}")

    return {"result": "processed"}
```

**2. 审计日志：**
```python
import logging
from datetime import datetime

audit_logger = logging.getLogger("audit")

@app.post("/predict")
async def predict(
    input_data: PredictionInput,
    current_user: User = Depends(get_current_user)
):
    # 记录访问
    audit_logger.info({
        "timestamp": datetime.utcnow().isoformat(),
        "user": current_user.username,
        "action": "prediction",
        "model": "bert-base",
        "ip": request.client.host
    })

    return model.predict(input_data.features)
```

## 最佳实践和清单

### 安全清单

**认证和授权：**
- [ ] 所有 endpoint 使用 HTTPS
- [ ] 实现适当的认证（JWT、OAuth2）
- [ ] 使用 bcrypt 或 argon2 哈希密码
- [ ] 实现基于角色的访问控制
- [ ] 服务间通信使用 API key
- [ ] 实现令牌过期和刷新

**输入验证：**
- [ ] 使用 Pydantic 验证所有输入
- [ ] 清理用户输入
- [ ] 实现文件上传限制
- [ ] 检查 SQL 注入漏洞
- [ ] 防止命令注入
- [ ] 防范路径遍历

**速率限制：**
- [ ] 为每个用户/IP 实现速率限制
- [ ] 为每个 endpoint 设置适当的限制
- [ ] 返回速率限制头部
- [ ] 优雅地处理超过速率限制

**头部和 CORS：**
- [ ] 正确配置 CORS
- [ ] 设置安全头部
- [ ] 实施 CSP 策略
- [ ] 对 HTTPS 强制使用 HSTS

**监控和日志：**
- [ ] 记录认证尝试
- [ ] 记录授权失败
- [ ] 监控可疑活动
- [ ] 设置安全事件告警
- [ ] 实现审计日志

**依赖：**
- [ ] 保持依赖更新
- [ ] 扫描漏洞
- [ ] 仅使用经过验证的包
- [ ] 锁定依赖版本

**ML 特定：**
- [ ] 验证模型输入
- [ ] 实现资源限制
- [ ] 保护模型文件
- [ ] 处理对抗性输入
- [ ] 在日志中屏蔽 PII

## 总结和关键要点

### 核心概念

1. **认证 vs 授权**
   - 认证：你是谁？
   - 授权：你能做什么？

2. **认证方法**
   - API Key：简单、无状态
   - JWT：自包含令牌
   - OAuth2：第三方访问

3. **安全基础**
   - 无处不在的 HTTPS
   - 哈希密码，绝不存储明文
   - 验证所有输入
   - 实现速率限制
   - 设置安全头部

4. **ML 特定安全**
   - 验证模型输入
   - 保护模型文件
   - 实现超时
   - 屏蔽 PII 数据
   - 审计日志

### 实现优先级

1. **必须有 (P0)：**
   - HTTPS/TLS
   - 认证
   - 输入验证
   - 密码哈希

2. **应该有 (P1)：**
   - 速率限制
   - 安全头部
   - CORS 配置
   - 审计日志

3. **最好有 (P2)：**
   - OAuth2 集成
   - 高级监控
   - 异常检测

### 下一步

- 完成练习 02：实现认证系统
- 复习 OWASP Top 10 API 安全风险
- 练习实现安全功能
- 学习安全测试和渗透测试

### 资源

- [OWASP API 安全 Top 10](https://owasp.org/www-project-api-security/)
- [FastAPI 安全文档](https://fastapi.tiangolo.com/tutorial/security/)
- [JWT.io](https://jwt.io/)
- [OAuth 2.0 简化版](https://www.oauth.com/)

---

**预计学习时间**：4-5 小时
**动手练习**：完成练习 02：认证和安全
**评估**：测验涵盖认证、授权和安全最佳实践
