# 讲座 05：Flask 框架

## 目录
1. [介绍](#介绍)
2. [什么是 Flask？](#什么是-flask)
3. [Flask vs FastAPI：何时选择哪个](#flask-vs-fastapi何时选择哪个)
4. [安装和项目布局](#安装和项目布局)
5. [你的第一个 Flask 应用程序](#你的第一个-flask-应用程序)
6. [路由和视图函数](#路由和视图函数)
7. [请求处理](#请求处理)
8. [响应构建](#响应构建)
9. [应用工厂模式和 Blueprint](#应用工厂模式和-blueprint)
10. [配置管理](#配置管理)
11. [错误处理](#错误处理)
12. [中间件和钩子](#中间件和钩子)
13. [Flask 扩展](#flask-扩展)
14. [构建 Flask ML API](#构建-flask-ml-api)
15. [生产部署](#生产部署)
16. [测试 Flask 应用程序](#测试-flask-应用程序)
17. [最佳实践](#最佳实践)
18. [总结和关键要点](#总结和关键要点)

## 介绍

Flask 是一个轻量级、成熟的 Python Web 框架，已经为生产 API 提供动力超过十年。虽然像 FastAPI 这样的新框架在 ML 社区引起了关注，但 Flask 在遗留系统、内部工具以及任何需要简单、无偏见、经过实战检验的框架的场合中仍然被广泛使用。

本讲座从头开始教授 Flask，专注于你在 Flask endpoint 后部署 ML 模型时实际会遇到的模式。最后，你将能够阅读现有的 Flask 代码库、熟练地编写新的 endpoint，并在新项目中对 Flask 和 FastAPI 做出明智的选择。

### 学习目标

在本讲座结束时，你将：
- 理解 Flask 的设计理念及其与 FastAPI 的不同之处
- 使用应用工厂模式设置 Flask 项目
- 正确定义路由、视图函数和处理 HTTP 动词
- 解析请求体、查询字符串、头部和上传的文件
- 构建 JSON、HTML 和流式响应
- 使用 Blueprint 组织大型应用程序
- 安全地跨环境管理配置
- 使用自定义错误页面和 JSON 错误响应处理错误
- 使用常见扩展：Flask-RESTful、Flask-SQLAlchemy、Flask-CORS、Flask-Smorest
- 在 Flask 中构建完整的 ML 推理 endpoint
- 在 Gunicorn 或 uWSGI 后部署 Flask
- 使用 pytest 测试 Flask 应用程序

### 先决条件
- 模块 001：Python 基础
- 模块 007 - 讲座 01：API 基础和 REST
- 模块 007 - 讲座 02：FastAPI 框架（用于对比）
- 对 WSGI 的基本理解

### 预计时间
3 小时（包括动手编码）

---

## 什么是 Flask？

Flask 是 Armin Ronacher 于 2010 年创建的 **WSGI 微框架**。"微框架"一词描述了它的哲学：Flask 提供构建 Web 应用程序所需的最少内容——路由、请求/响应对象、通过 Jinja2 的模板以及开发服务器——其余所有内容（数据库访问、认证、表单验证）都留给扩展或你自己的代码。

### 主要特性

1. **最小的核心、庞大的生态系统。** Flask 本身很小（几千行）。功能通过精心策划的扩展如 Flask-SQLAlchemy、Flask-Login、Flask-Smorest 添加。
2. **基于 WSGI。** Flask 遵循 Python Web 服务器网关接口标准（PEP 3333），使其与任何 WSGI 服务器兼容：Gunicorn、uWSGI、mod_wsgi、Waitress。
3. **默认同步。** 每个请求由一个工作线程或进程处理。Flask 2.0+ 支持 `async def` 视图函数，但不像 FastAPI 那样在事件循环上运行。
4. **无偏见。** Flask 不规定项目结构、ORM 或模板引擎。你组装你想要的堆栈。
5. **成熟稳定。** 经过十多年的生产使用，框架的 API 不太可能让你意外，文档也非常丰富。

### 为什么 Flask 在 ML 基础设施中仍然重要

- **遗留系统。** 许多生产 ML 服务先于 FastAPI 出现，并在 Flask 上运行。在处理现有平台时，熟悉它是必不可少的。
- **轻量级内部工具。** 一个简单的管理 endpoint、模型健康检查器或一次性批处理触发器在 Flask 中通常比在任何更重的框架中更容易。
- **运维熟悉度。** 多年来运行 Flask + Gunicorn 的运维团队拥有完善的 runbook、告警和调优知识。
- **WSGI 生态系统。** 像 New Relic、Datadog APM 和许多 WAF 都有一流的 WSGI 中间件支持。
- **稳定的模式。** 当你不需要异步 I/O 或自动生成文档时，Flask 的简单性减少了认知负担。

---

## Flask vs FastAPI：何时选择哪个

| 方面 | Flask | FastAPI |
|---|---|---|
| 风格 | WSGI、同步 | ASGI、异步原生 |
| 速度 | 适当；受限于同步 I/O | I/O 密集型工作负载的更高吞吐量 |
| 类型提示 | 可选、不强制 | 路由/验证必需 |
| 数据验证 | 手动或扩展（marshmallow、pydantic） | 通过 Pydantic 内置 |
| 自动文档 | 通过扩展（Flask-Smorest、apispec） | 内置 OpenAPI + Swagger UI |
| 异步 I/O | 有限（Flask 2.0+，但受限于工作进程） | 一流支持 |
| 成熟度 | 非常成熟（自 2010 年起） | 成熟（自 2018 年起），快速成长 |
| 学习曲线 | 平缓 | 稍陡（Pydantic、async） |
| 最适合 | 内部工具、遗留服务、简单 API | 高并发 ML 服务、现代新建 API |

### 决策指南

**选择 Flask 当：**
- 你在现有的 Flask 代码库上工作。
- 你的 endpoint 是 CPU 密集型（模型推理）而非 I/O 密集型——异步没有好处。
- 团队已经对 Flask + Gunicorn 运维有深入经验。
- 你需要一个很小的单文件服务。

**选择 FastAPI 当：**
- 你在构建新的高流量 ML 服务层。
- 你的工作负载是 I/O 密集型（数据库调用、调用其他服务、流式传输）。
- 你想要自动生成文档而不需要配置扩展。
- 你希望默认有更严格的输入验证。

具体到模型推理，差异通常不大：模型延迟主导框架开销。选择团队最能维护的那个。

---

## 安装和项目布局

```bash
# 推荐：首先创建虚拟环境
python -m venv .venv
source .venv/bin/activate

pip install flask
```

小型服务的最小项目结构：

```
myapi/
├── app.py
├── requirements.txt
└── tests/
    └── test_app.py
```

对于任何会超越原型的项目，使用应用工厂模式（如下所述）：

```
myapi/
├── app/
│   ├── __init__.py          # create_app() 工厂
│   ├── routes/
│   │   ├── __init__.py
│   │   ├── health.py
│   │   └── predict.py
│   ├── models/              # SQLAlchemy 模型，不是 ML 模型
│   ├── ml/
│   │   ├── loader.py
│   │   └── inference.py
│   ├── schemas.py           # marshmallow/pydantic
│   └── config.py
├── tests/
├── wsgi.py                  # Gunicorn 入口点
└── requirements.txt
```

---

## 你的第一个 Flask 应用程序

```python
# app.py
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/")
def index():
    return jsonify(message="Hello, Flask")

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000, debug=True)
```

运行它：

```bash
python app.py
# 或
flask --app app run --debug
```

访问 `http://localhost:8000/`。

### 发生了什么

- `Flask(__name__)` 创建应用程序对象。`__name__` 参数告诉 Flask 在哪里查找模板和静态文件。
- `@app.route("/")` 是一个装饰器，将 `index` 注册为路径 `/` 的视图函数。
- `jsonify` 将 Python 字典转换为带有 `Content-Type: application/json` 的 JSON 响应。
- `app.run(...)` 启动 **开发服务器。** 永远不要在生产中使用它。

---

## 路由和视图函数

### HTTP 方法

默认情况下，路由只接受 `GET`。对其他方法使用 `methods` 参数：

```python
@app.route("/predict", methods=["POST"])
def predict():
    ...

@app.route("/model/<model_id>", methods=["GET", "DELETE"])
def model(model_id):
    ...
```

你也可以使用 Flask 2.0 中新增的便捷装饰器：

```python
@app.get("/health")
def health():
    return {"status": "ok"}

@app.post("/predict")
def predict():
    ...
```

### URL 变量和转换器

```python
@app.get("/users/<int:user_id>")
def user(user_id: int):
    return {"id": user_id}

@app.get("/files/<path:filepath>")
def file(filepath: str):
    # 'path' 转换器允许斜杠
    return {"path": filepath}
```

内置转换器：`string`（默认）、`int`、`float`、`path`、`uuid`。自定义转换器可以在 `app.url_map.converters` 上注册。

### url_for() — 构建 URL，不要硬编码

```python
from flask import url_for

@app.get("/")
def index():
    return {"predict_url": url_for("predict", _external=True)}
```

`url_for` 按视图函数名称查找路由，因此重命名 URL 不会破坏其他地方的链接。

---

## 请求处理

与请求绑定的 `request` 对象暴露了关于传入请求的一切。

```python
from flask import request

@app.post("/predict")
def predict():
    # JSON 体
    payload = request.get_json(silent=True) or {}

    # 查询参数：/predict?model=resnet50
    model_name = request.args.get("model", "default")

    # 头部
    auth = request.headers.get("Authorization")

    # 路径
    path = request.path  # "/predict"

    # 表单数据
    # value = request.form["field_name"]

    # 上传的文件 (multipart/form-data)
    # f = request.files["file"]
    # f.save("/tmp/upload.bin")

    return {"model": model_name, "received": payload}
```

### 解析和验证 JSON

原始的 `request.get_json()` 返回客户端发送的任何内容。你应该进行验证。两种常见方法：

**方法 1：手动验证**

```python
@app.post("/predict")
def predict():
    payload = request.get_json(silent=True)
    if not isinstance(payload, dict):
        return {"error": "JSON object required"}, 400

    features = payload.get("features")
    if not isinstance(features, list) or len(features) != 10:
        return {"error": "features must be a list of length 10"}, 400

    return {"prediction": run_model(features)}
```

**方法 2：marshmallow schema**

```python
from marshmallow import Schema, fields, ValidationError

class PredictRequest(Schema):
    features = fields.List(fields.Float(), required=True)
    model_version = fields.String(load_default="latest")

schema = PredictRequest()

@app.post("/predict")
def predict():
    try:
        data = schema.load(request.get_json() or {})
    except ValidationError as err:
        return {"errors": err.messages}, 400

    return {"prediction": run_model(data["features"], data["model_version"])}
```

对于已经使用 Pydantic 的项目，`flask-pydantic` 扩展提供了类似的使用体验。

---

## 响应构建

视图函数可以返回：

```python
# 元组：(body, status, headers)
return {"ok": True}, 201, {"X-Request-Id": "abc"}

# 字典 — 自动转换为 JSON (Flask 1.1+)
return {"ok": True}

# 用于完全控制的 Flask Response 对象
from flask import make_response
resp = make_response({"ok": True}, 200)
resp.headers["X-Request-Id"] = "abc"
return resp

# 流式响应（例如，大型推理批次）
from flask import Response
def generate():
    yield "["
    for i, result in enumerate(stream_predictions()):
        if i > 0: yield ","
        yield json.dumps(result)
    yield "]"
return Response(generate(), mimetype="application/json")
```

### 标准状态码

| 状态码 | 用途 |
|---|---|
| 200 | 成功带请求体 |
| 201 | 资源已创建 (POST 到集合) |
| 202 | 已接受，正在异步处理 |
| 204 | 成功，无请求体 |
| 400 | 客户端请求错误（验证） |
| 401 | 未认证 |
| 403 | 已认证但禁止 |
| 404 | 资源未找到 |
| 422 | 语义无效的请求 |
| 500 | 服务器错误 |
| 503 | 暂时不可用（例如，模型未加载） |

---

## 应用工厂模式和 Blueprint

对于大于单个文件的项目，使用工厂和 blueprint 进行组织。

### 工厂

```python
# app/__init__.py
from flask import Flask
from .config import load_config
from .routes.health import bp as health_bp
from .routes.predict import bp as predict_bp

def create_app(config_name: str = "default") -> Flask:
    app = Flask(__name__)
    app.config.update(load_config(config_name))

    app.register_blueprint(health_bp)
    app.register_blueprint(predict_bp, url_prefix="/v1")

    return app
```

### Blueprint

```python
# app/routes/predict.py
from flask import Blueprint, request, current_app

bp = Blueprint("predict", __name__)

@bp.post("/predict")
def predict():
    model = current_app.config["MODEL"]
    payload = request.get_json() or {}
    return {"prediction": model.predict(payload["features"])}
```

### WSGI 入口点

```python
# wsgi.py
from app import create_app
app = create_app("production")
```

使用 Gunicorn 运行：

```bash
gunicorn --workers 4 --bind 0.0.0.0:8000 wsgi:app
```

优点：
- 用于不同配置测试的多个应用实例。
- 按功能区域清晰分离关注点。
- 更容易挂载子应用和复用路由。

---

## 配置管理

永远不要硬编码密钥。Flask 支持多种配置加载模式。

```python
# 环境变量
app.config["MODEL_PATH"] = os.environ["MODEL_PATH"]
app.config["SECRET_KEY"] = os.environ["SECRET_KEY"]

# 从 Python 类
class ProductionConfig:
    DEBUG = False
    MODEL_PATH = os.environ["MODEL_PATH"]
    LOG_LEVEL = "INFO"

app.config.from_object(ProductionConfig)

# 从带环境前缀的加载器 (Flask 2.1+)
app.config.from_prefixed_env(prefix="MYAPI")
# MYAPI_MODEL_PATH=/models/v1 → app.config["MODEL_PATH"]
```

对于生产环境，更倾向于使用由你的编排器管理的环境变量（Kubernetes secrets、AWS Secrets Manager）。永远不要提交包含真实密钥的 `.env`。

---

## 错误处理

### 自定义错误页面和 JSON 错误

```python
from werkzeug.exceptions import HTTPException

@app.errorhandler(HTTPException)
def handle_http_error(err):
    return {
        "error": err.name,
        "detail": err.description,
        "status": err.code,
    }, err.code

@app.errorhandler(Exception)
def handle_unexpected(err):
    app.logger.exception("Unhandled error")
    return {"error": "internal_server_error"}, 500
```

### 提前中止

```python
from flask import abort

@app.get("/models/<model_id>")
def get_model(model_id):
    model = registry.find(model_id)
    if model is None:
        abort(404, description=f"Model '{model_id}' not found")
    return model.to_dict()
```

---

## 中间件和钩子

Flask 将这些称为"请求钩子"：

```python
@app.before_request
def log_request():
    app.logger.info("%s %s", request.method, request.path)

@app.after_request
def add_request_id(response):
    response.headers["X-Request-Id"] = request.headers.get("X-Request-Id", uuid.uuid4().hex)
    return response

@app.teardown_request
def cleanup(exc):
    # 始终运行，即使有异常。在此关闭数据库会话。
    pass
```

对于真正的 WSGI 中间件（例如，Prometheus 仪表化），包装 `app.wsgi_app`：

```python
from werkzeug.middleware.proxy_fix import ProxyFix
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1)
```

---

## Flask 扩展

Flask 生态系统包括许多高质量的扩展。你在 ML 基础设施中最常遇到的：

| 扩展 | 用途 |
|---|---|
| **Flask-SQLAlchemy** | 关系数据库的 ORM 集成 |
| **Flask-Migrate** | 基于 Alembic 的 schema 迁移 |
| **Flask-Smorest** | OpenAPI 生成 + marshmallow 集成（最接近 FastAPI 自动文档的方案） |
| **Flask-RESTful** | 基于类的资源路由（较老，仍然常见） |
| **Flask-CORS** | 跨域请求处理 |
| **Flask-Login** | 基于会话的用户认证 |
| **Flask-JWT-Extended** | 基于 JWT 的认证 |
| **Flask-Limiter** | 速率限制 |
| **prometheus-flask-exporter** | 用于 Prometheus 抓取的 `/metrics` endpoint |

### 示例：Flask-Smorest 用于自动生成文档

```python
from flask import Flask
from flask_smorest import Api, Blueprint
from marshmallow import Schema, fields

app = Flask(__name__)
app.config["API_TITLE"] = "Model API"
app.config["API_VERSION"] = "v1"
app.config["OPENAPI_VERSION"] = "3.0.3"
app.config["OPENAPI_URL_PREFIX"] = "/"
app.config["OPENAPI_SWAGGER_UI_PATH"] = "/docs"
app.config["OPENAPI_SWAGGER_UI_URL"] = "https://cdn.jsdelivr.net/npm/swagger-ui-dist/"

api = Api(app)
blp = Blueprint("predict", "predict", url_prefix="/v1")

class PredictIn(Schema):
    features = fields.List(fields.Float(), required=True)

class PredictOut(Schema):
    prediction = fields.Float()

@blp.route("/predict", methods=["POST"])
@blp.arguments(PredictIn)
@blp.response(200, PredictOut)
def predict(payload):
    return {"prediction": sum(payload["features"]) / len(payload["features"])}

api.register_blueprint(blp)
```

访问 `/docs`，你将拥有与 FastAPI 相当的 Swagger UI。

### 示例：Prometheus 指标

```python
from prometheus_flask_exporter import PrometheusMetrics

metrics = PrometheusMetrics(app)
metrics.info("app_info", "Model API", version="1.0.0")
```

这会暴露带有请求计数、延迟和异常计数器的 `/metrics` —— 准备好供 Prometheus 抓取。

---

## 构建 Flask ML API

一个实际的最小 ML 服务 endpoint：

```python
# app/ml/loader.py
import joblib
from threading import Lock

_lock = Lock()
_model = None

def get_model(model_path: str):
    global _model
    with _lock:
        if _model is None:
            _model = joblib.load(model_path)
        return _model
```

```python
# app/routes/predict.py
from flask import Blueprint, request, current_app, abort
from marshmallow import Schema, fields, ValidationError
import time

from ..ml.loader import get_model

bp = Blueprint("predict", __name__)

class PredictRequest(Schema):
    features = fields.List(fields.Float(), required=True)

schema = PredictRequest()

@bp.post("/predict")
def predict():
    try:
        data = schema.load(request.get_json() or {})
    except ValidationError as err:
        return {"errors": err.messages}, 400

    if len(data["features"]) != current_app.config["FEATURE_COUNT"]:
        abort(400, description="wrong feature count")

    model = get_model(current_app.config["MODEL_PATH"])

    start = time.perf_counter()
    pred = model.predict([data["features"]])[0]
    elapsed_ms = (time.perf_counter() - start) * 1000

    return {
        "prediction": float(pred),
        "latency_ms": round(elapsed_ms, 2),
    }
```

```python
# app/__init__.py
from flask import Flask
from prometheus_flask_exporter import PrometheusMetrics

def create_app(config):
    app = Flask(__name__)
    app.config.update(config)

    from .routes.predict import bp as predict_bp
    app.register_blueprint(predict_bp, url_prefix="/v1")

    PrometheusMetrics(app)

    @app.get("/health")
    def health():
        return {"status": "ok"}

    return app
```

### Flask 中 ML 的重要考虑

1. **模型加载。** 在启动时或首次请求时惰性加载模型，在锁后。永远不要每次请求都重新加载。
2. **工作进程模型。** 每个 Gunicorn 工作进程是一个单独的 Python 进程，拥有自己的模型副本。相应地规划内存。
3. **GIL 和 CPU 工作。** 推理通常是 CPU 密集型，这意味着 GIL 会在工作进程内序列化工作。为了更高的吞吐量，增加工作进程数而不是线程数。
4. **冷启动。** 进程启动后的第一个请求承担模型加载成本。使用 readiness 探针在流量到达之前预热模型。
5. **批量 endpoint。** 为了高吞吐量，每个请求接受一批输入，而不是一次一个预测。

---

## 生产部署

Flask 开发服务器不用于生产。使用 WSGI 服务器如 Gunicorn：

```bash
gunicorn \
  --workers 4 \
  --threads 2 \
  --bind 0.0.0.0:8000 \
  --timeout 60 \
  --access-logfile - \
  --error-logfile - \
  wsgi:app
```

### 工作进程调优

- **工作进程数**：对于 I/O 密集型工作负载，从 `(2 × CPU 核心数) + 1` 开始。对于 CPU 密集型的 ML 推理，使用 `CPU 核心数`，有时更少。
- **线程数**：对 I/O 密集型工作有用；对于 CPU 密集型推理，由于 GIL 的存在，效果较差。
- **超时**：设置在你的 p99 推理延迟之上；在你的负载均衡器的请求超时之下。

### 在反向代理后

在 Gunicorn 前运行 NGINX 或云负载均衡器，用于 TLS 终止、静态资产服务和连接池。使用 `werkzeug.middleware.proxy_fix.ProxyFix` 告诉 Flask 信任转发的头部。

### 容器化

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .

ENV PYTHONUNBUFFERED=1
EXPOSE 8000

CMD ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8000", "wsgi:app"]
```

---

## 测试 Flask 应用程序

Flask 提供了一个测试客户端，可以在进程中运行你的应用——不需要真实的网络：

```python
# tests/test_predict.py
import pytest
from app import create_app

@pytest.fixture
def client():
    app = create_app({"TESTING": True, "MODEL_PATH": "tests/fixtures/model.joblib", "FEATURE_COUNT": 4})
    with app.test_client() as c:
        yield c

def test_health(client):
    r = client.get("/health")
    assert r.status_code == 200
    assert r.get_json() == {"status": "ok"}

def test_predict_happy_path(client):
    r = client.post("/v1/predict", json={"features": [1.0, 2.0, 3.0, 4.0]})
    assert r.status_code == 200
    body = r.get_json()
    assert "prediction" in body
    assert isinstance(body["prediction"], float)

def test_predict_validation(client):
    r = client.post("/v1/predict", json={"features": "not a list"})
    assert r.status_code == 400
```

使用 `pytest` 运行。对于负载测试，使用 Locust（在 `04-api-testing.md` 练习中涵盖）。

---

## 最佳实践

1. **使用应用工厂。** 避免依赖于环境的模块级状态。
2. **保持视图函数精简。** 将业务逻辑推入单独模块。
3. **验证输入。** 在边界处以 400 拒绝坏请求，永远不要让坏数据到达你的模型。
4. **返回结构化错误。** 一致的 JSON 错误形状（`{"error": "...", "detail": "..."}`）更易于使用。
5. **结构化日志。** 在生产环境中使用 JSON 格式的 `app.logger`。
6. **仪表化一切。** 从第一天起添加 Prometheus 指标、分布式追踪和结构化请求日志。
7. **永远不要在生产中运行开发服务器。** 始终使用 Gunicorn 或 uWSGI。
8. **锁定依赖。** 使用带有明确版本的 `requirements.txt`，由 `pip-compile` 或 `poetry export` 生成。
9. **健康检查和就绪检查 endpoint。** 如果进程启动，`/health` 总是 200。仅当模型加载且依赖可访问时，`/ready` 才为 200。
10. **优雅关闭。** Gunicorn 处理 `SIGTERM`。确保长时间运行的预测能够正常完成或被取消。

---

## 总结和关键要点

- Flask 是一个同步 WSGI 微框架——最小的核心，通过扩展实现其他所有功能。
- 选择 Flask 用于简单服务、遗留系统和 CPU 密集型的 ML 推理（异步没有好处）。
- 对于任何超过原型规模的项目，使用应用工厂 + blueprint。
- 使用 marshmallow、pydantic 或手动检查验证输入。不要信任 JSON 载荷。
- 要获得与 FastAPI 相当的自动生成文档，使用 **Flask-Smorest**。
- 每个工作进程加载 ML 模型一次；永远不要每次请求重新加载。
- 使用 Gunicorn 部署，前方放置 NGINX 或云负载均衡器。永远不要在生产中使用开发服务器。
- 使用 `app.test_client()` 进行测试——快速、进程内、无需网络。
- 从第一天开始使用 `prometheus-flask-exporter` 和结构化日志进行仪表化。

在下一个模块中，你将容器化同样的 Flask 服务并将其部署到 Kubernetes，与 FastAPI 变体并列，从而可以直接比较运维特性。
