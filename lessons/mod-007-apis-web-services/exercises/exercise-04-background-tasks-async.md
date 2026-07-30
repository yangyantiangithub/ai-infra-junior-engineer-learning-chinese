# 练习 04：后台任务和异步操作

## 概述

学习在 FastAPI 中实现异步操作和后台任务，以高效处理长时间运行的 ML 操作。理解如何在不阻塞 API 响应的情况下处理任务，以及为模型训练和批量处理实现任务队列。

**难度：** 中级-高级
**预计时间：** 2-3 小时
**先决条件：**
- 已完成练习 01-03
- 讲座 02：FastAPI 框架
- 对 Python 中 async/await 的基本理解

## 学习目标

完成此练习后，你将：
- 实现 FastAPI BackgroundTasks 用于简单的异步操作
- 创建长时间运行的任务跟踪系统
- 使用 async/await 处理 I/O 密集型操作
- 为 ML 操作实现任务队列
- 处理任务状态轮询
- 使用服务器推送事件（SSE）流式传输结果

## 场景

你的 ML API 需要处理需要时间的操作：
- 模型训练（几分钟到几小时）
- 大型批量预测
- 数据预处理管道
- 模型评估

用户不应等待响应。相反，提交任务并轮询状态。

## 项目设置

### 安装依赖

```bash
pip install aiofiles
pip install sse-starlette  # 用于服务器推送事件
```

## 第一部分：简单后台任务

**文件：app/background.py**

```python
from fastapi import BackgroundTasks
import time
import logging

logger = logging.getLogger(__name__)

def log_prediction(user_id: str, text: str, prediction: str):
    """将预测记录到文件（在后台运行）"""
    # TODO: 以追加模式打开日志文件
    # TODO: 写入包含时间戳、用户、输入、预测的日志条目
    # TODO: 模拟一些处理时间
    pass

def send_notification(email: str, message: str):
    """发送邮件通知（模拟）"""
    # TODO: 模拟发送邮件
    # TODO: 记录通知已发送
    # time.sleep(1)  # 模拟邮件发送
    pass

def update_user_stats(user_id: str):
    """更新用户请求统计信息"""
    # TODO: 增加用户的请求计数
    # TODO: 更新 last_request_time
    pass
```

**文件：app/main.py**（更新）

```python
from fastapi import BackgroundTasks

@app.post("/api/v1/predict")
async def predict(
    input_data: PredictionInput,
    background_tasks: BackgroundTasks,
    current_user: User = Depends(get_current_active_user)
):
    """带后台日志记录的预测"""

    # 进行预测（快速）
    result = ml_service.predict(input_data.text)

    # TODO: 添加后台任务（不阻塞响应）
    # background_tasks.add_task(log_prediction, current_user.username, input_data.text, result["prediction"])
    # background_tasks.add_task(update_user_stats, current_user.username)

    # 立即返回
    return result
```

## 第二部分：任务队列系统

**文件：app/jobs.py**

```python
from enum import Enum
from typing import Dict, Any, Optional
from datetime import datetime
from pydantic import BaseModel
import uuid
import asyncio

class JobStatus(str, Enum):
    PENDING = "pending"
    RUNNING = "running"
    COMPLETED = "completed"
    FAILED = "failed"
    CANCELLED = "cancelled"

class Job(BaseModel):
    job_id: str
    job_type: str
    status: JobStatus
    created_at: datetime
    started_at: Optional[datetime] = None
    completed_at: Optional[datetime] = None
    progress: float = 0.0  # 0-100
    result: Optional[Dict[str, Any]] = None
    error: Optional[str] = None
    user_id: str

class JobQueue:
    """简单的内存任务队列"""

    def __init__(self):
        self.jobs: Dict[str, Job] = {}
        self.active_jobs = set()

    def create_job(self, job_type: str, user_id: str) -> str:
        """创建新任务"""
        # TODO: 使用 uuid 生成 job_id
        # TODO: 创建状态为 PENDING 的 Job 对象
        # TODO: 存储在 self.jobs 中
        # TODO: 返回 job_id
        pass

    def get_job(self, job_id: str) -> Optional[Job]:
        """按 ID 获取任务"""
        # TODO: 如果存在则从 self.jobs 返回任务
        pass

    def update_job_status(self, job_id: str, status: JobStatus, **kwargs):
        """更新任务状态和其他字段"""
        # TODO: 获取任务
        # TODO: 更新状态
        # TODO: 更新其他字段（started_at、completed_at 等）
        # TODO: 如果完成/失败，从 active_jobs 中移除
        pass

    def update_progress(self, job_id: str, progress: float):
        """更新任务进度"""
        # TODO: 获取任务
        # TODO: 更新进度（限制在 0-100 范围内）
        pass

    def set_result(self, job_id: str, result: Dict[str, Any]):
        """设置任务结果"""
        # TODO: 获取任务
        # TODO: 设置结果
        # TODO: 设置状态为 COMPLETED
        # TODO: 设置 completed_at
        pass

    def set_error(self, job_id: str, error: str):
        """设置任务错误"""
        # TODO: 获取任务
        # TODO: 设置错误
        # TODO: 设置状态为 FAILED
        # TODO: 设置 completed_at
        pass

    def get_user_jobs(self, user_id: str) -> list[Job]:
        """获取用户的所有任务"""
        # TODO: 按 user_id 过滤任务
        # TODO: 按 created_at 排序返回（最新的在前）
        pass

# 全局任务队列
job_queue = JobQueue()
```

## 第三部分：长时间运行的任务：批量预测

**文件：app/tasks.py**

```python
import asyncio
from app.jobs import job_queue, JobStatus
from app.ml_service import ml_service
import logging

logger = logging.getLogger(__name__)

async def process_batch_prediction(
    job_id: str,
    texts: list[str],
    user_id: str
):
    """以后台任务方式处理批量预测"""
    try:
        # TODO: 更新任务状态为 RUNNING
        # job_queue.update_job_status(job_id, JobStatus.RUNNING, started_at=datetime.utcnow())

        # TODO: 按 10 个一批进行处理
        batch_size = 10
        results = []

        for i in range(0, len(texts), batch_size):
            # TODO: 获取批次
            batch = texts[i:i+batch_size]

            # TODO: 处理批次（用 sleep 模拟）
            # await asyncio.sleep(1)  # 模拟处理
            batch_results = [ml_service.predict(text) for text in batch]
            results.extend(batch_results)

            # TODO: 更新进度
            progress = ((i + len(batch)) / len(texts)) * 100
            job_queue.update_progress(job_id, progress)

        # TODO: 设置结果
        job_queue.set_result(job_id, {
            "predictions": results,
            "total": len(results)
        })

        logger.info(f"Batch job {job_id} completed successfully")

    except Exception as e:
        # TODO: 设置错误
        logger.error(f"Batch job {job_id} failed: {e}")
        job_queue.set_error(job_id, str(e))

async def process_model_training(
    job_id: str,
    training_data: dict,
    user_id: str
):
    """以后台任务方式模拟模型训练"""
    try:
        # TODO: 更新状态为 RUNNING
        job_queue.update_job_status(job_id, JobStatus.RUNNING, started_at=datetime.utcnow())

        # TODO: 模拟训练轮次
        epochs = 10
        for epoch in range(epochs):
            # 模拟轮次处理
            await asyncio.sleep(2)

            # 更新进度
            progress = ((epoch + 1) / epochs) * 100
            job_queue.update_progress(job_id, progress)

        # TODO: 设置结果
        job_queue.set_result(job_id, {
            "model_id": f"model_{job_id[:8]}",
            "accuracy": 0.95,
            "epochs": epochs
        })

    except Exception as e:
        job_queue.set_error(job_id, str(e))
```

## 第四部分：任务 API Endpoint

**文件：app/main.py**（更新）

```python
from app.jobs import job_queue, Job, JobStatus
from app.tasks import process_batch_prediction, process_model_training
import asyncio

@app.post("/api/v1/jobs/batch-predict", response_model=dict)
async def create_batch_prediction_job(
    texts: list[str],
    current_user: User = Depends(get_current_active_user)
):
    """创建批量预测任务"""

    # TODO: 验证输入（最多 1000 条文本）
    if len(texts) > 1000:
        raise HTTPException(status_code=400, detail="Maximum 1000 texts allowed")

    # TODO: 创建任务
    job_id = job_queue.create_job("batch_predict", current_user.username)

    # TODO: 启动后台任务
    asyncio.create_task(process_batch_prediction(job_id, texts, current_user.username))

    # TODO: 返回任务信息
    return {
        "job_id": job_id,
        "status": "pending",
        "message": "Job created successfully",
        "poll_url": f"/api/v1/jobs/{job_id}"
    }

@app.post("/api/v1/jobs/train-model", response_model=dict)
async def create_training_job(
    training_data: dict,
    current_user: User = Depends(get_current_active_user)
):
    """创建模型训练任务"""

    # TODO: 创建任务
    job_id = job_queue.create_job("train_model", current_user.username)

    # TODO: 启动后台任务
    asyncio.create_task(process_model_training(job_id, training_data, current_user.username))

    # TODO: 返回任务信息
    return {
        "job_id": job_id,
        "status": "pending",
        "message": "Training job created",
        "poll_url": f"/api/v1/jobs/{job_id}"
    }

@app.get("/api/v1/jobs/{job_id}", response_model=Job)
async def get_job_status(
    job_id: str,
    current_user: User = Depends(get_current_active_user)
):
    """获取任务状态"""

    # TODO: 从队列获取任务
    job = job_queue.get_job(job_id)

    if not job:
        raise HTTPException(status_code=404, detail="Job not found")

    # TODO: 验证任务属于用户
    if job.user_id != current_user.username:
        raise HTTPException(status_code=403, detail="Not authorized")

    return job

@app.get("/api/v1/jobs", response_model=list[Job])
async def list_user_jobs(
    current_user: User = Depends(get_current_active_user)
):
    """列出当前用户的所有任务"""
    # TODO: 获取用户的任务
    jobs = job_queue.get_user_jobs(current_user.username)
    return jobs

@app.delete("/api/v1/jobs/{job_id}")
async def cancel_job(
    job_id: str,
    current_user: User = Depends(get_current_active_user)
):
    """取消任务"""

    job = job_queue.get_job(job_id)
    if not job:
        raise HTTPException(status_code=404, detail="Job not found")

    if job.user_id != current_user.username:
        raise HTTPException(status_code=403, detail="Not authorized")

    # TODO: 只允许取消待处理/运行中的任务
    if job.status in [JobStatus.PENDING, JobStatus.RUNNING]:
        job_queue.update_job_status(job_id, JobStatus.CANCELLED)
        return {"message": "Job cancelled"}
    else:
        raise HTTPException(status_code=400, detail="Cannot cancel completed job")
```

## 第五部分：用于实时更新的服务器推送事件

**文件：app/streaming.py**

```python
from sse_starlette.sse import EventSourceResponse
from fastapi import Request
import asyncio
import json

async def job_status_stream(job_id: str, request: Request):
    """使用 SSE 流式传输任务状态更新"""

    async def event_generator():
        last_status = None
        last_progress = None

        while True:
            # 检查客户端是否断开连接
            if await request.is_disconnected():
                break

            # 获取当前任务状态
            job = job_queue.get_job(job_id)
            if not job:
                yield {
                    "event": "error",
                    "data": json.dumps({"error": "Job not found"})
                }
                break

            # 如果状态或进度发生变化则发送更新
            if job.status != last_status or job.progress != last_progress:
                yield {
                    "event": "update",
                    "data": json.dumps({
                        "status": job.status,
                        "progress": job.progress,
                        "job_id": job_id
                    })
                }
                last_status = job.status
                last_progress = job.progress

            # 如果任务完成则停止流式传输
            if job.status in [JobStatus.COMPLETED, JobStatus.FAILED, JobStatus.CANCELLED]:
                yield {
                    "event": "complete",
                    "data": json.dumps({
                        "status": job.status,
                        "result": job.result,
                        "error": job.error
                    })
                }
                break

            # 等待下一次更新
            await asyncio.sleep(1)

    return EventSourceResponse(event_generator())
```

**在 main.py 中添加 endpoint：**

```python
from app.streaming import job_status_stream

@app.get("/api/v1/jobs/{job_id}/stream")
async def stream_job_status(
    job_id: str,
    request: Request,
    current_user: User = Depends(get_current_active_user)
):
    """通过 SSE 流式传输任务状态更新"""

    job = job_queue.get_job(job_id)
    if not job:
        raise HTTPException(status_code=404, detail="Job not found")

    if job.user_id != current_user.username:
        raise HTTPException(status_code=403, detail="Not authorized")

    return await job_status_stream(job_id, request)
```

## 第六部分：异步文件操作

**文件：app/file_ops.py**

```python
import aiofiles
import asyncio
from typing import List

async def save_results_async(filename: str, data: dict):
    """异步将结果保存到文件"""
    # TODO: 使用 aiofiles 写入 JSON
    # async with aiofiles.open(filename, 'w') as f:
    #     await f.write(json.dumps(data, indent=2))
    pass

async def read_large_file_async(filename: str) -> List[str]:
    """异步逐行读取大文件"""
    lines = []
    # TODO: 使用 aiofiles 读取文件
    # async with aiofiles.open(filename, 'r') as f:
    #     async for line in f:
    #         lines.append(line.strip())
    return lines

async def process_file_batch(filenames: List[str]) -> List[dict]:
    """并发处理多个文件"""
    # TODO: 为每个文件创建任务
    # tasks = [read_large_file_async(f) for f in filenames]

    # TODO: 收集结果
    # results = await asyncio.gather(*tasks)

    return results
```

## 第七部分：测试异步操作

**文件：tests/test_jobs.py**

```python
from fastapi.testclient import TestClient
from app.main import app
import time

client = TestClient(app)

def get_token():
    """辅助函数获取认证令牌"""
    response = client.post("/token", data={"username": "alice", "password": "secret123"})
    return response.json()["access_token"]

def test_create_batch_job():
    """测试创建批量预测任务"""
    token = get_token()
    headers = {"Authorization": f"Bearer {token}"}

    # TODO: 创建批量任务
    response = client.post(
        "/api/v1/jobs/batch-predict",
        json={"texts": ["test1", "test2", "test3"]},
        headers=headers
    )

    # TODO: 断言状态 200
    # TODO: 断言 job_id 在响应中
    # TODO: 断言 poll_url 在响应中
    pass

def test_get_job_status():
    """测试获取任务状态"""
    token = get_token()
    headers = {"Authorization": f"Bearer {token}"}

    # TODO: 创建任务
    create_response = client.post(
        "/api/v1/jobs/batch-predict",
        json={"texts": ["test"]},
        headers=headers
    )
    job_id = create_response.json()["job_id"]

    # TODO: 获取任务状态
    response = client.get(f"/api/v1/jobs/{job_id}", headers=headers)

    # TODO: 断言状态 200
    # TODO: 断言 job_id 匹配
    # TODO: 断言状态为 pending 或 running
    pass

def test_job_completion():
    """测试任务成功完成"""
    token = get_token()
    headers = {"Authorization": f"Bearer {token}"}

    # TODO: 创建小批量任务
    response = client.post(
        "/api/v1/jobs/batch-predict",
        json={"texts": ["test1", "test2"]},
        headers=headers
    )
    job_id = response.json()["job_id"]

    # TODO: 轮询直到完成（最多 30 秒）
    max_attempts = 30
    for _ in range(max_attempts):
        status_response = client.get(f"/api/v1/jobs/{job_id}", headers=headers)
        job = status_response.json()

        if job["status"] == "completed":
            # TODO: 断言结果存在
            # TODO: 断言 predictions 数组有 2 项
            break

        time.sleep(1)
    else:
        assert False, "Job did not complete in time"

def test_list_user_jobs():
    """测试列出用户的任务"""
    token = get_token()
    headers = {"Authorization": f"Bearer {token}"}

    # TODO: 创建 2 个任务
    for i in range(2):
        client.post(
            "/api/v1/jobs/batch-predict",
            json={"texts": [f"test{i}"]},
            headers=headers
        )

    # TODO: 列出任务
    response = client.get("/api/v1/jobs", headers=headers)

    # TODO: 断言状态 200
    # TODO: 断言至少返回 2 个任务
    pass

def test_cancel_job():
    """测试取消任务"""
    token = get_token()
    headers = {"Authorization": f"Bearer {token}"}

    # TODO: 创建长时间运行的任务
    response = client.post(
        "/api/v1/jobs/train-model",
        json={"data": "test"},
        headers=headers
    )
    job_id = response.json()["job_id"]

    # TODO: 取消任务
    cancel_response = client.delete(f"/api/v1/jobs/{job_id}", headers=headers)

    # TODO: 断言状态 200
    # TODO: 验证任务状态为 cancelled
    pass
```

## 挑战和扩展

### 挑战 1：任务持久化

将任务存储在数据库中而不是内存中：

```python
# 使用 SQLAlchemy 或 MongoDB
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# 将任务存储在数据库中，以便重启后持续存在
```

### 挑战 2：任务优先级队列

实现基于优先级的任务处理：

```python
import heapq

class PriorityJobQueue:
    def __init__(self):
        self.queue = []

    def add_job(self, job: Job, priority: int):
        heapq.heappush(self.queue, (priority, job))

    def get_next_job(self):
        if self.queue:
            return heapq.heappop(self.queue)[1]
        return None
```

### 挑战 3：Celery 集成

使用 Celery 作为生产任务队列：

```bash
pip install celery redis
```

```python
from celery import Celery

celery_app = Celery('tasks', broker='redis://localhost:6379')

@celery_app.task
def process_batch(texts: list):
    # 处理批量
    return results
```

### 挑战 4：WebSocket 更新

使用 WebSocket 替代 SSE 进行双向通信：

```python
from fastapi import WebSocket

@app.websocket("/ws/jobs/{job_id}")
async def websocket_job_updates(websocket: WebSocket, job_id: str):
    await websocket.accept()
    # 通过 WebSocket 发送更新
```

## 关键要点

1. **BackgroundTasks：** 不阻塞响应的简单异步操作
2. **任务队列：** 通过状态轮询跟踪长时间运行的操作
3. **Async/Await：** 高效处理 I/O 密集型操作
4. **SSE：** 无需轮询开销的实时更新
5. **进度跟踪：** 让用户了解任务状态

## 下一步

- 练习 05：使用 Docker 部署 API
- 学习 Celery 和 RabbitMQ 用于生产队列
- 实现 WebSocket 用于实时双向通信

---

**预计时间：** 2-3 小时
**难度：** 中级-高级
**重点：** 异步操作、后台任务、任务队列、流式传输
