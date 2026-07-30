# Lecture 04：Python 异步编程

## 目录
1. [引言](#引言)
2. [理解 Async/Await](#理解-asyncawait)
3. [并发运行多个任务](#并发运行多个任务)
4. [异步 HTTP 请求](#异步-http-请求)
5. [异步代码中的错误处理](#异步代码中的错误处理)
6. [异步上下文管理器](#异步上下文管理器)
7. [异步生成器](#异步生成器)
8. [AI 基础设施的常见异步模式](#ai-基础设施的常见异步模式)
9. [总结](#总结)

---

## 引言

本讲座涵盖 Python 中的异步编程，这是构建处理多个并发操作的高效 AI 基础设施系统的关键技能。

### 为什么异步编程很重要

**在 AI 基础设施中**：
- **处理多个 API 请求**：同时为多个客户端提供预测服务
- **并发模型操作**：同时检查多个模型端点的健康状态
- **高效 I/O 操作**：无阻塞地下载数据集或上传模型
- **可扩展服务**：构建能够处理高吞吐量的系统

**异步编程**允许你编写能够同时处理多个操作而不阻塞的并发代码。这对于 I/O 密集型操作至关重要，例如：
- 发起 HTTP API 调用
- 读取/写入数据库
- 文件 I/O 操作
- 网络通信

### 学习目标

完成本讲座后，你将能够：
- 理解何时以及为何使用异步编程
- 使用 `async`/`await` 语法编写异步函数
- 使用 `asyncio.gather()` 并发运行多个任务
- 处理异步代码中的错误
- 实现异步上下文管理器和生成器
- 将异步模式应用于 AI 基础设施场景

本讲座为你准备**练习 06：异步编程**，你将构建一个并发的模型监控系统。

---

## 理解 Async/Await

### 何时使用 Async

✅ **使用 Async 的情况**：
- I/O 密集型操作（网络调用、文件操作、数据库查询）
- 处理大量并发连接（Web 服务器、API 网关）
- 等待外部系统的长时间运行操作

❌ **不要使用 Async 的情况**：
- CPU 密集型操作（改用 multiprocessing）
- 没有并发操作的简单脚本
- 不涉及等待的代码

### 基本 Async 语法

```python
import asyncio

# 使用 'async def' 定义异步函数
async def fetch_data(url: str) -> dict:
    """从 URL 获取数据的异步函数"""
    await asyncio.sleep(1)  # 模拟网络延迟
    return {"url": url, "data": "sample data"}

# 运行异步函数
async def main():
    result = await fetch_data("https://api.example.com/data")
    print(result)

# 执行异步代码
if __name__ == "__main__":
    asyncio.run(main())
```

**关键概念**：
- `async def`：定义一个协程函数
- `await`：暂停执行直到等待的操作完成
- `asyncio.run()`：运行异步代码的入口点

---

## 并发运行多个任务

异步的强大之处在于一次运行多个操作：

```python
import asyncio
from typing import List

async def fetch_model_metadata(model_id: str) -> dict:
    """获取单个模型的元数据"""
    await asyncio.sleep(0.5)  # 模拟 API 调用
    return {
        "model_id": model_id,
        "version": "1.0",
        "accuracy": 0.95
    }

async def fetch_all_models(model_ids: List[str]) -> List[dict]:
    """并发获取多个模型的元数据"""
    # 为所有模型创建任务
    tasks = [fetch_model_metadata(mid) for mid in model_ids]

    # 并发运行所有任务
    results = await asyncio.gather(*tasks)
    return results

async def main():
    model_ids = [f"model-{i}" for i in range(5)]

    # 顺序执行（慢）
    print("Sequential execution:")
    import time
    start = time.time()
    results = []
    for mid in model_ids:
        result = await fetch_model_metadata(mid)
        results.append(result)
    print(f"Time: {time.time() - start:.2f}s")  # ~2.5 秒

    # 并发执行（快）
    print("\nConcurrent execution:")
    start = time.time()
    results = await fetch_all_models(model_ids)
    print(f"Time: {time.time() - start:.2f}s")  # ~0.5 秒

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 异步 HTTP 请求

使用 `httpx` 进行异步 HTTP 调用：

```python
import asyncio
import httpx
from typing import List, Dict

async def check_model_health(endpoint: str) -> Dict[str, str]:
    """检查模型服务端点的健康状态"""
    async with httpx.AsyncClient(timeout=5.0) as client:
        try:
            response = await client.get(f"{endpoint}/health")
            return {
                "endpoint": endpoint,
                "status": "healthy" if response.status_code == 200 else "unhealthy",
                "response_time": response.elapsed.total_seconds()
            }
        except Exception as e:
            return {
                "endpoint": endpoint,
                "status": "error",
                "error": str(e)
            }

async def monitor_models(endpoints: List[str]) -> List[Dict]:
    """并发监控多个模型端点"""
    tasks = [check_model_health(ep) for ep in endpoints]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    return results

# 使用
async def main():
    endpoints = [
        "http://model1.example.com",
        "http://model2.example.com",
        "http://model3.example.com"
    ]

    health_status = await monitor_models(endpoints)
    for status in health_status:
        print(f"{status['endpoint']}: {status['status']}")

if __name__ == "__main__":
    asyncio.run(main())
```

---

## 异步代码中的错误处理

```python
import asyncio
from typing import Optional

async def fetch_with_retry(url: str, max_retries: int = 3) -> Optional[dict]:
    """带重试逻辑的数据获取"""
    for attempt in range(max_retries):
        try:
            async with httpx.AsyncClient() as client:
                response = await client.get(url, timeout=10.0)
                response.raise_for_status()
                return response.json()
        except httpx.TimeoutException:
            print(f"Timeout on attempt {attempt + 1}")
            if attempt < max_retries - 1:
                await asyncio.sleep(2 ** attempt)  # 指数退避
        except httpx.HTTPError as e:
            print(f"HTTP error: {e}")
            return None

    return None  # 所有重试均失败
```

---

## 异步上下文管理器

```python
import asyncio
from typing import AsyncGenerator

class AsyncModelClient:
    """ML 模型服务的异步客户端"""

    async def __aenter__(self):
        """进入上下文时的设置"""
        self.client = httpx.AsyncClient()
        await self.connect()
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        """退出上下文时的清理"""
        await self.disconnect()
        await self.client.aclose()

    async def connect(self):
        """建立连接"""
        await asyncio.sleep(0.1)  # 模拟连接
        print("Connected to model server")

    async def disconnect(self):
        """关闭连接"""
        await asyncio.sleep(0.1)
        print("Disconnected from model server")

    async def predict(self, data: dict) -> dict:
        """进行预测"""
        response = await self.client.post(
            "http://model.example.com/predict",
            json=data
        )
        return response.json()

# 使用
async def main():
    async with AsyncModelClient() as client:
        result = await client.predict({"features": [1, 2, 3]})
        print(result)
```

---

## 异步生成器

```python
import asyncio
from typing import AsyncGenerator

async def stream_training_logs(job_id: str) -> AsyncGenerator[str, None]:
    """异步流式输出训练日志"""
    for i in range(10):
        await asyncio.sleep(0.5)
        yield f"[{job_id}] Epoch {i+1}/10 - loss: {1.0/(i+1):.4f}"

async def monitor_training():
    """监控训练作业日志"""
    async for log_line in stream_training_logs("job-123"):
        print(log_line)

if __name__ == "__main__":
    asyncio.run(monitor_training())
```

---

## AI 基础设施的常见异步模式

### 模式 1：并发模型部署

```python
import asyncio
from typing import List

async def deploy_model(model_id: str, environment: str) -> dict:
    """部署单个模型"""
    print(f"Deploying {model_id} to {environment}...")
    await asyncio.sleep(2)  # 模拟部署时间
    return {
        "model_id": model_id,
        "environment": environment,
        "status": "deployed"
    }

async def deploy_multiple_models(
    models: List[str],
    environment: str
) -> List[dict]:
    """并发部署多个模型"""
    tasks = [deploy_model(mid, environment) for mid in models]
    return await asyncio.gather(*tasks)
```

### 模式 2：异步数据管道

```python
import asyncio
from typing import List, Dict

async def fetch_training_data(source: str) -> List[Dict]:
    """从源获取训练数据"""
    await asyncio.sleep(1)
    return [{"id": i, "features": [i]*10} for i in range(100)]

async def preprocess_batch(batch: List[Dict]) -> List[Dict]:
    """预处理一批数据"""
    await asyncio.sleep(0.5)
    return [{"id": item["id"], "processed": True} for item in batch]

async def save_to_storage(data: List[Dict]) -> None:
    """将处理后的数据保存到存储"""
    await asyncio.sleep(0.3)
    print(f"Saved {len(data)} records")

async def data_pipeline():
    """使用异步操作执行数据管道"""
    # 获取数据
    raw_data = await fetch_training_data("s3://bucket/data")

    # 并发预处理批次
    batch_size = 25
    batches = [raw_data[i:i+batch_size] for i in range(0, len(raw_data), batch_size)]
    processed_batches = await asyncio.gather(*[preprocess_batch(b) for b in batches])

    # 展平结果
    processed_data = [item for batch in processed_batches for item in batch]

    # 保存
    await save_to_storage(processed_data)
```

---

## 总结

### 关键要点

#### 异步编程
- 使用 `async`/`await` 进行 I/O 密集型并发操作
- `asyncio.gather()` 并发运行多个任务
- 异步编程对于高效的 API 调用和数据库查询至关重要
- 始终使用 `asyncio.run()` 作为入口点
- 在异步函数中使用 try/except 处理错误
- 使用 `httpx` 进行异步 HTTP 请求
- 为资源管理实现异步上下文管理器
- 使用异步生成器进行数据流式处理

### AI 基础设施的最佳实践

1. **编写异步代码用于**：
   - 模型服务 API（多个并发请求）
   - 多模型调用的批量推理
   - 跨服务的健康检查监控
   - 数据管道操作
   - 并发模型部署

2. **性能考虑**：
   - 异步编程在 I/O 密集型操作中表现出色
   - 对 CPU 密集型任务使用 multiprocessing
   - 监控并发限制以避免压垮系统
   - 实现适当的超时和重试逻辑

3. **错误处理**：
   - 始终将异步操作包裹在 try/except 中
   - 在 `asyncio.gather()` 中使用 `return_exceptions=True` 处理部分失败
   - 为重试实现指数退避
   - 使用适当的上下文记录错误

### 下一步

- **完成练习 06：异步编程** - 构建一个并发的模型监控系统
- **继续到 Lecture 05：测试与代码质量** - 学习测试异步代码并维护质量标准
- 使用真实的 HTTP API 练习异步模式
- 尝试使用 `asyncio` 调试工具

### 附加资源

- [asyncio 文档](https://docs.python.org/3/library/asyncio.html)
- [Real Python：Async IO](https://realpython.com/async-io-python/)
- [HTTPX 文档](https://www.python-httpx.org/)
- [AsyncIO 速查表](https://github.com/crazyguitar/pysheeet/blob/master/docs/notes/python-asyncio.rst)

---

**讲座版本**：1.0
**最后更新**：2025 年 10 月
**预计时长**：6-8 小时

**准备好继续了吗？** 进入 `lecture-notes/05-testing-code-quality.md`
