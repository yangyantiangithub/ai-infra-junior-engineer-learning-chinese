# Python 速查表 — AI 基础设施版

在 ML 基础设施工作中会用到的 Python 特性和库的快速参考。不是教程——假设你已完成模块 001。

## 环境和打包

```bash
# 创建并激活虚拟环境
python -m venv .venv
source .venv/bin/activate           # macOS/Linux
.venv\Scripts\activate              # Windows

# 锁定版本以确保可重现安装
pip install -r requirements.txt
pip freeze > requirements.txt

# 现代替代方案：uv（比 pip 快 10-100 倍）
uv venv
uv pip install -r requirements.txt
uv pip compile requirements.in -o requirements.txt
```

| 工具 | 用途 |
|---|---|
| `pip` + `venv` | 标准方案，Python 自带 |
| `uv` | 直接替代，快得多 |
| `poetry` | 依赖解析 + 打包，配置严格 |
| `pip-tools` | 编译 `requirements.in` → 锁定版 `requirements.txt` |
| `conda` | 当你需要捆绑非 Python 依赖（CUDA、MKL）时 |

## 类型注解

```python
from typing import Optional
from collections.abc import Iterable, Mapping

def predict(features: list[float], model_version: str = "latest") -> float:
    ...

def fetch(urls: Iterable[str]) -> Mapping[str, bytes]:
    ...

def lookup(user_id: int) -> Optional[dict]:  # 或：dict | None（3.10+）
    ...
```

在 CI 中运行 `mypy` 或 `pyright`。将类型错误视为编译错误。

## 日志（不要用 print）

```python
import logging
logger = logging.getLogger(__name__)
logger.info("加载模型", extra={"model_version": "v3", "size_mb": 42})

# 结构化 JSON 日志
import json, logging.config
logging.config.dictConfig({
    "version": 1,
    "formatters": {
        "json": {"()": "pythonjsonlogger.json.JsonFormatter",
                  "format": "%(asctime)s %(levelname)s %(name)s %(message)s"}
    },
    "handlers": {"console": {"class": "logging.StreamHandler", "formatter": "json"}},
    "root": {"level": "INFO", "handlers": ["console"]},
})
```

级别：`DEBUG < INFO < WARNING < ERROR < CRITICAL`。

## 错误处理

```python
try:
    result = risky()
except (TimeoutError, ConnectionError) as exc:
    logger.warning("临时错误", exc_info=exc)
    raise                       # 重新抛出，保留调用栈
except ValueError as exc:
    raise InvalidInput(str(exc)) from exc   # 异常链
finally:
    cleanup()
```

不要使用裸 `except`。除非在最顶层边界，否则不要捕获 `Exception`。

## 文件 I/O 和路径

```python
from pathlib import Path

p = Path("/data/raw") / "events.json"
text = p.read_text()
data = json.loads(text)

# 遍历
for child in Path(".").rglob("*.parquet"):
    print(child)

# 原子写入
tmp = p.with_suffix(".tmp")
tmp.write_text(payload)
tmp.replace(p)
```

## JSON / YAML

```python
import json, yaml

data = json.loads(text)
text = json.dumps(data, indent=2, default=str)   # default=str 处理日期时间

with open("config.yaml") as f:
    cfg = yaml.safe_load(f)                       # 始终使用 safe_load，不要用 load
```

## 子进程（必须调用外部命令时）

```python
import subprocess
res = subprocess.run(
    ["kubectl", "get", "pods", "-o", "json"],
    capture_output=True, text=True, check=True, timeout=30,
)
pods = json.loads(res.stdout)
```

`check=True` 在非零退出时抛出异常。始终设置 `timeout`。切勿将 `shell=True` 与用户输入一起使用。

## 使用 argparse 的 CLI

```python
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("--model", required=True)
parser.add_argument("--threshold", type=float, default=0.5)
parser.add_argument("-v", "--verbose", action="store_true")
args = parser.parse_args()
```

对于较大型的 CLI：使用 `click` 或 `typer`。

## 使用 requests 的 HTTP

```python
import requests

resp = requests.post(
    "https://api.example.com/predict",
    json={"features": [...]},
    headers={"Authorization": f"Bearer {token}"},
    timeout=(3.0, 10.0),    # （连接超时，读取超时）
)
resp.raise_for_status()
data = resp.json()
```

始终设置超时。始终调用 `raise_for_status()` 或检查 `resp.ok`。

## 使用 asyncio 的异步

```python
import asyncio, httpx

async def fetch(url):
    async with httpx.AsyncClient() as client:
        r = await client.get(url, timeout=10)
        return r.json()

async def main():
    results = await asyncio.gather(*(fetch(u) for u in urls))

asyncio.run(main())
```

## 并发速查

| 工作负载 | 工具 |
|---|---|
| I/O 密集型，多任务 | `asyncio` |
| I/O 密集型，少量任务，简单 | `concurrent.futures.ThreadPoolExecutor` |
| CPU 密集型 | `concurrent.futures.ProcessPoolExecutor` |
| 混合 | 使用 `loop.run_in_executor` 将进程池与事件循环分离 |

## 使用 pytest 的测试

```python
import pytest

def test_predict_returns_float():
    assert isinstance(predict([1.0, 2.0]), float)

@pytest.mark.parametrize("inp,expected", [([1.0], 1.0), ([2.0], 2.0)])
def test_identity(inp, expected):
    assert predict(inp) == expected

@pytest.fixture
def client():
    app = create_app({"TESTING": True})
    with app.test_client() as c:
        yield c
```

运行：`pytest -v --cov=src --cov-report=term-missing`

## 工具

| 工具 | 用途 |
|---|---|
| `black` | 自动格式化（没有配置争议） |
| `ruff` | 快速代码检查 + 格式化（替代 flake8、isort、pyupgrade） |
| `mypy` / `pyright` | 静态类型检查 |
| `pytest` | 测试运行器 |
| `pytest-cov` | 覆盖率 |
| `pre-commit` | 提交时运行格式化/检查 |
| `bandit` | 安全扫描 |

## 标准项目结构

```
myproject/
├── pyproject.toml         # 项目元数据 + 工具配置
├── src/
│   └── mypackage/
│       ├── __init__.py
│       └── ...
├── tests/
│   └── test_*.py
├── requirements.txt
└── README.md
```

`src/` 布局可防止从工作目录意外导入。

## 常见陷阱

- **可变默认参数。** 不要写 `def f(x=[]):` — 使用 `def f(x=None): x = x or []`。
- **闭包延迟绑定。** `[lambda: i for i in range(3)]` — 所有 lambda 返回 `2`。使用 `lambda i=i: i`。
- **`is` 与 `==`。** `is` 是身份比较。仅在 `is None`、`is True`、`is False` 时使用。
- **浮点数。** `0.1 + 0.2 != 0.3`。使用 `math.isclose` 或 `decimal.Decimal` 处理货币。
- **GIL。** 在 CPython 中，线程不能并行执行 CPU 任务。使用多进程。
- **`__init__.py` 中的顺序。** 不要在包初始化中放入有副作用的代码。

## 有用的标准库模块

| 模块 | 用途 |
|---|---|
| `pathlib` | 文件系统路径 |
| `dataclasses` | 简单的数据类 |
| `enum` | 命名常量 |
| `collections` | `Counter`、`defaultdict`、`deque` |
| `itertools` | `chain`、`groupby`、`product`、`islice` |
| `functools` | `lru_cache`、`partial`、`reduce` |
| `contextlib` | `contextmanager`、`suppress`、`ExitStack` |
| `concurrent.futures` | 基于池的并行 |
| `secrets` | 加密安全随机数 |
| `uuid` | 唯一 ID |
| `datetime` + `zoneinfo` | 时区感知的日期时间 |

## 另请参阅

- [Real Python](https://realpython.com/) — 高质量教程
- [Python type checking guide](https://mypy.readthedocs.io/)
- [The pytest docs](https://docs.pytest.org/)
