# 练习 03：功能开发的分支

## 概述

**难度**：入门到中级
**预计时间**：75-90 分钟
**前置条件**：练习 02 - 处理 Commit 与历史，讲座 02 - 分支与合并

在本练习中，你将学习如何在 ML 项目中有效使用 Git 分支进行并行功能开发。你将练习创建分支、切换上下文、同时管理多个功能以及清理旧分支。

## 学习目标

完成本练习后，你将：

- 创建和切换功能分支
- 理解分支命名约定
- 同时处理多个功能
- 使用分支隔离实验性工作
- 查看和比较分支
- 删除已合并和未合并的分支
- 对 ML 项目应用分支策略
- 在不丢失工作的情况下在分支间导航
- 有效使用 `git switch` 和 `git checkout`

## 场景

你的 ML 推理 API 正在演进。团队同时在开发多个功能：
- 添加批量推理支持
- 实现模型缓存以加快预测速度
- 添加 Prometheus 指标导出
- 实验 ONNX 模型支持

每个功能需要独立开发，以便进行独立的测试和部署。你将使用 Git 分支来管理这种并行开发。

## 前置条件

开始之前，确保你有：

```bash
# 前面练习中的仓库
cd ml-inference-api

# 干净的工作目录
git status
# 应显示："nothing to commit, working tree clean"

# 如果你需要一个干净的仓库，提交或暂存更改
git add .
git commit -m "wip: save current work"
```

---

## 第一部分：创建和切换分支

### 任务 1.1：理解分支

查看你当前的分支结构。

**说明：**

```bash
# 查看当前分支
git branch

# 查看所有分支（包括远程）
git branch -a

# 查看带有最新 commit 的分支
git branch -v

# 查看 HEAD 指向哪里
cat .git/HEAD

# 查看当前分支指向的 commit
git log --oneline -1
```

**预期输出：**
```text
* main
```

`*` 表示你当前的分支。

---

### 任务 1.2：创建第一个功能分支

为批量推理支持创建一个分支。

**说明：**

```bash
# 创建批量推理功能分支
git branch feature/batch-inference

# 列出分支
git branch

# 注意：你创建了分支但没有切换
# HEAD 仍在 main 上

# 切换到新分支
git switch feature/batch-inference

# 替代方案（更旧的语法，仍然有效）：
# git checkout feature/batch-inference

# 验证你在新分支上
git branch
git status

# 一步创建并切换（快捷方式）
git switch -c feature/model-caching

# 替代方案：
# git checkout -b feature/model-caching

# 查看带有 commit 信息的分支
git branch -v
```

**预期输出：**
```text
* feature/model-caching
  feature/batch-inference
  main
```

---

### 任务 1.3：分支命名约定

学习并应用常见的分支命名模式。

**说明：**

```bash
# 切换回 main
git switch main

# 按照约定创建分支：

# 功能分支
git branch feature/prometheus-metrics
git branch feature/onnx-support

# Bug 修复分支
git branch fix/memory-leak-preprocessing
git branch fix/timeout-handling

# 热修复分支（紧急生产修复）
git branch hotfix/api-crash-on-large-images

# 实验分支
git branch experiment/quantization
git branch experiment/tensorrt

# 查看所有分支
git branch

# 以树形格式查看
git log --oneline --graph --all --decorate
```

**分支命名约定：**

| 前缀 | 使用场景 | 示例 |
|--------|----------|---------|
| `feature/` | 新功能 | `feature/batch-inference` |
| `fix/` | Bug 修复 | `fix/memory-leak` |
| `hotfix/` | 紧急生产修复 | `hotfix/security-patch` |
| `experiment/` | 实验性工作 | `experiment/new-model` |
| `refactor/` | 代码重构 | `refactor/api-structure` |
| `docs/` | 文档 | `docs/api-guide` |
| `test/` | 测试改进 | `test/integration-tests` |

---

## 第二部分：在功能分支上工作

### 任务 2.1：实现批量推理功能

在其分支上开发批量推理功能。

**说明：**

```bash
# 切换到批量推理分支
git switch feature/batch-inference

# 验证你在正确的分支上
git branch

# 创建批量推理模块
cat > src/api/batch.py << 'EOF'
"""
Batch Inference Module

Handles batch prediction requests for efficiency.
"""

from typing import List, Dict
from fastapi import UploadFile
import asyncio
import structlog

from src.models.classifier import ImageClassifier
from src.preprocessing.image import preprocess_image

logger = structlog.get_logger()


class BatchInferenceService:
    """Service for processing multiple images in batch."""

    def __init__(self, classifier: ImageClassifier, max_batch_size: int = 32):
        """
        Initialize batch inference service.

        Args:
            classifier: Image classifier instance
            max_batch_size: Maximum images per batch
        """
        self.classifier = classifier
        self.max_batch_size = max_batch_size
        logger.info("Batch inference service initialized", max_batch=max_batch_size)

    async def process_batch(self, files: List[UploadFile]) -> List[Dict]:
        """
        Process multiple images in batch.

        Args:
            files: List of uploaded image files

        Returns:
            List of prediction results
        """
        if len(files) > self.max_batch_size:
            raise ValueError(f"Batch size {len(files)} exceeds max {self.max_batch_size}")

        logger.info("Processing batch", num_images=len(files))

        # Preprocess all images concurrently
        preprocess_tasks = [
            preprocess_image(await f.read()) for f in files
        ]
        preprocessed_images = await asyncio.gather(*preprocess_tasks)

        # Run batch inference
        results = []
        for idx, image_tensor in enumerate(preprocessed_images):
            predictions = await self.classifier.predict(image_tensor)
            results.append({
                "file_index": idx,
                "filename": files[idx].filename,
                "predictions": predictions
            })

        logger.info("Batch processing complete", num_images=len(results))
        return results
EOF

# 提交模块
git add src/api/batch.py
git commit -m "feat(batch): add batch inference service

Implement BatchInferenceService for processing multiple images:
- Concurrent image preprocessing
- Configurable max batch size (default 32)
- Returns predictions for all images in batch

This improves throughput for bulk classification tasks."

# 添加批量端点到 API
cat > src/api/routes/batch.py << 'EOF'
"""
Batch Inference Routes

API endpoints for batch prediction.
"""

from typing import List
from fastapi import APIRouter, File, UploadFile, HTTPException
from fastapi.responses import JSONResponse

from src.api.batch import BatchInferenceService
from src.models.classifier import ImageClassifier

router = APIRouter()

# Initialize batch service (will be properly injected later)
classifier = ImageClassifier()
batch_service = BatchInferenceService(classifier, max_batch_size=32)


@router.post("/batch/predict")
async def batch_predict(files: List[UploadFile] = File(...)) -> JSONResponse:
    """
    Predict classifications for multiple images.

    Args:
        files: List of image files (max 32)

    Returns:
        JSON response with predictions for all images
    """
    if not files:
        raise HTTPException(status_code=400, detail="No files provided")

    if len(files) > 32:
        raise HTTPException(
            status_code=400,
            detail=f"Too many files. Maximum batch size is 32, got {len(files)}"
        )

    try:
        results = await batch_service.process_batch(files)
        return JSONResponse(content={
            "success": True,
            "batch_size": len(files),
            "results": results
        })
    except Exception as e:
        raise HTTPException(
            status_code=500,
            detail=f"Batch prediction failed: {str(e)}"
        )
EOF

# 创建目录并提交
mkdir -p src/api/routes
git add src/api/routes/batch.py
git commit -m "feat(batch): add batch prediction endpoint

Add POST /batch/predict endpoint:
- Accepts up to 32 images per request
- Returns predictions for all images
- Validates batch size
- Error handling for batch failures

Example usage:
  POST /batch/predict
  files: [image1.jpg, image2.jpg, ...]"

# 添加测试
cat > tests/unit/test_batch.py << 'EOF'
"""Tests for batch inference."""

import pytest
from src.api.batch import BatchInferenceService
from src.models.classifier import ImageClassifier


@pytest.fixture
def batch_service():
    """Create batch service for testing."""
    classifier = ImageClassifier()
    return BatchInferenceService(classifier, max_batch_size=5)


def test_batch_service_initialization(batch_service):
    """Test batch service initializes correctly."""
    assert batch_service.max_batch_size == 5
    assert batch_service.classifier is not None


def test_batch_size_validation(batch_service):
    """Test batch size validation."""
    # This would need actual UploadFile objects
    # Placeholder for proper test implementation
    assert batch_service.max_batch_size == 5
EOF

git add tests/unit/test_batch.py
git commit -m "test(batch): add batch inference tests

Add unit tests for batch service initialization and validation."

# 查看你在这个分支上的工作
git log --oneline
```

**检查点：**
- 这个分支上有多少个 commit？
- 这些 commit 在 `main` 分支上吗？

---

### 任务 2.2：实现模型缓存功能

切换到另一个功能分支并并行开发。

**说明：**

```bash
# 切换到模型缓存分支
git switch feature/model-caching

# 注意：批量推理文件消失了！
ls src/api/batch.py  # File not found

# 这是预期的 - 你在不同的分支上
# 查看分支状态
git log --oneline -3

# 现在实现缓存
cat > src/utils/cache.py << 'EOF'
"""
Model Caching Module

Implements LRU cache for model predictions.
"""

from functools import lru_cache
from typing import Optional, Tuple
import hashlib
import pickle
import structlog
from datetime import datetime, timedelta

logger = structlog.get_logger()


class PredictionCache:
    """Cache for model predictions with TTL."""

    def __init__(self, max_size: int = 1000, ttl_seconds: int = 3600):
        """
        Initialize prediction cache.

        Args:
            max_size: Maximum number of cached predictions
            ttl_seconds: Time to live for cache entries
        """
        self.max_size = max_size
        self.ttl_seconds = ttl_seconds
        self.cache = {}
        self.timestamps = {}
        logger.info("Cache initialized", max_size=max_size, ttl=ttl_seconds)

    def _generate_key(self, image_bytes: bytes) -> str:
        """
        Generate cache key from image bytes.

        Args:
            image_bytes: Raw image bytes

        Returns:
            SHA256 hash as cache key
        """
        return hashlib.sha256(image_bytes).hexdigest()

    def get(self, image_bytes: bytes) -> Optional[dict]:
        """
        Get cached prediction.

        Args:
            image_bytes: Raw image bytes

        Returns:
            Cached prediction or None if not found/expired
        """
        key = self._generate_key(image_bytes)

        if key not in self.cache:
            logger.debug("Cache miss", key=key[:16])
            return None

        # Check TTL
        timestamp = self.timestamps.get(key)
        if timestamp:
            age = (datetime.now() - timestamp).total_seconds()
            if age > self.ttl_seconds:
                logger.debug("Cache entry expired", key=key[:16], age=age)
                self._remove(key)
                return None

        logger.debug("Cache hit", key=key[:16])
        return self.cache[key]

    def set(self, image_bytes: bytes, prediction: dict):
        """
        Cache prediction.

        Args:
            image_bytes: Raw image bytes
            prediction: Prediction result to cache
        """
        key = self._generate_key(image_bytes)

        # Evict oldest if at max size
        if len(self.cache) >= self.max_size:
            self._evict_oldest()

        self.cache[key] = prediction
        self.timestamps[key] = datetime.now()
        logger.debug("Cached prediction", key=key[:16])

    def _remove(self, key: str):
        """Remove key from cache."""
        self.cache.pop(key, None)
        self.timestamps.pop(key, None)

    def _evict_oldest(self):
        """Evict oldest cache entry."""
        if not self.timestamps:
            return

        oldest_key = min(self.timestamps, key=self.timestamps.get)
        logger.debug("Evicting oldest entry", key=oldest_key[:16])
        self._remove(oldest_key)

    def clear(self):
        """Clear all cache entries."""
        self.cache.clear()
        self.timestamps.clear()
        logger.info("Cache cleared")

    def stats(self) -> dict:
        """Get cache statistics."""
        return {
            "size": len(self.cache),
            "max_size": self.max_size,
            "ttl_seconds": self.ttl_seconds
        }


# Global cache instance
prediction_cache = PredictionCache()
EOF

git add src/utils/cache.py
git commit -m "feat(cache): implement prediction caching with TTL

Add PredictionCache for caching model predictions:
- LRU eviction policy
- Configurable TTL (default 1 hour)
- SHA256-based cache keys
- Cache statistics

Benefits:
- Reduces redundant inference
- Improves response time for duplicate requests
- Configurable cache size (default 1000 entries)"

# 更新配置
cat >> configs/default.yaml << 'EOF'

# Prediction Cache
cache:
  enabled: true
  max_size: 1000
  ttl_seconds: 3600
EOF

git add configs/default.yaml
git commit -m "config: add cache configuration settings"

# 查看这个分支的历史
git log --oneline
```

**检查点：**
运行 `git log --oneline --all --graph` 查看两个分支！

---

### 任务 2.3：在 Main 上快速修复 Bug

练习为紧急修复切换分支。

**说明：**

```bash
# 你在 feature/model-caching 上
# 报告了 main 上的紧急 bug！

# 切换回 main
git switch main

# 验证你的功能工作安全存储
ls src/utils/cache.py  # File not found (它在另一个分支上)

# 从 main 创建热修复分支
git switch -c hotfix/fix-timeout

# 修复问题
cat > src/api/middleware.py << 'EOF'
"""
API Middleware

Custom middleware for request processing.
"""

from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
import time
import structlog
import asyncio

from src.utils.rate_limiter import RateLimiter

logger = structlog.get_logger()
rate_limiter = RateLimiter()


async def logging_middleware(request: Request, call_next):
    """Log all requests with timing."""
    start_time = time.time()

    logger.info(
        "Request started",
        method=request.method,
        path=request.url.path,
        client=request.client.host
    )

    response = await call_next(request)
    duration = time.time() - start_time

    logger.info(
        "Request completed",
        method=request.method,
        path=request.url.path,
        status=response.status_code,
        duration_ms=duration * 1000
    )

    return response


async def rate_limit_middleware(request: Request, call_next):
    """Apply rate limiting to requests."""
    if not rate_limiter.allow_request():
        wait_time = rate_limiter.get_wait_time()
        logger.warning(
            "Rate limit exceeded",
            client=request.client.host,
            wait_time=wait_time
        )
        return JSONResponse(
            status_code=429,
            content={
                "error": "Rate limit exceeded",
                "retry_after": wait_time
            }
        )

    return await call_next(request)


async def timeout_middleware(request: Request, call_next):
    """Enforce request timeout."""
    try:
        # 30 second timeout for all requests
        response = await asyncio.wait_for(
            call_next(request),
            timeout=30.0
        )
        return response
    except asyncio.TimeoutError:
        logger.error(
            "Request timeout",
            method=request.method,
            path=request.url.path
        )
        return JSONResponse(
            status_code=504,
            content={"error": "Request timeout"}
        )
EOF

git add src/api/middleware.py
git commit -m "fix: add timeout middleware to prevent hanging requests

Add timeout_middleware to enforce 30-second timeout on all requests.
Prevents server resources from being tied up by slow clients.

Returns 504 Gateway Timeout on timeout."

# 查看所有分支
git branch -v
```

---

## 第三部分：查看和比较分支

### 任务 3.1：比较分支

学习查看分支间的差异。

**说明：**

```bash
# 列出所有分支
git branch -a

# 查看 feature/batch-inference 上不在 main 中的 commit
git log main..feature/batch-inference --oneline

# 查看 feature/model-caching 上不在 main 中的 commit
git log main..feature/model-caching --oneline

# 查看两个功能分支上的所有 commit
git log main..feature/batch-inference main..feature/model-caching --oneline

# 比较分支之间的文件
git diff main feature/batch-inference

# 比较特定文件
git diff main feature/batch-inference -- src/api/

# 显示哪些文件不同
git diff --name-only main feature/batch-inference

# 所有分支的图形化视图
git log --oneline --graph --all --decorate

# 更详细的图形
git log --graph --all --decorate --pretty=format:"%h %an: %s"
```

**检查点练习：**

使用 Git 命令回答：

1. `feature/batch-inference` 上有多少个 commit？
2. `feature/model-caching` 上有多少个 commit？
3. `main` 和 `feature/batch-inference` 之间哪些文件不同？

```bash
# 答案：
git rev-list --count main..feature/batch-inference
git rev-list --count main..feature/model-caching
git diff --name-only main feature/batch-inference
```

---

### 任务 3.2：查看分支历史

可视化分支关系。

**说明：**

```bash
# 图形化历史
git log --oneline --graph --all

# 带有日期
git log --graph --all --pretty=format:"%h %ad | %s%d [%an]" --date=short

# 仅显示分支
git log --oneline --graph --all --simplify-by-decoration

# 查看分支分叉点
git merge-base main feature/batch-inference
git log --oneline $(git merge-base main feature/batch-inference)

# 显示从分支创建以来的 commit
git log --oneline main..feature/batch-inference
```

---

## 第四部分：管理多个功能

### 任务 4.1：在分支间切换

练习上下文切换。

**说明：**

```bash
# 在批量推理上工作
git switch feature/batch-inference

# 进行更改
cat >> src/api/batch.py << 'EOF'


def validate_batch_size(num_files: int, max_size: int = 32) -> bool:
    """Validate batch size is within limits."""
    return 0 < num_files <= max_size
EOF

# 检查状态
git status

# 保存工作（提交它）
git add src/api/batch.py
git commit -m "feat(batch): add batch size validation helper"

# 切换到缓存功能
git switch feature/model-caching

# 在那里进行更改
cat >> src/utils/cache.py << 'EOF'


def get_cache_stats_summary():
    """Get human-readable cache statistics."""
    stats = prediction_cache.stats()
    return f"Cache: {stats['size']}/{stats['max_size']} (TTL: {stats['ttl_seconds']}s)"
EOF

git add src/utils/cache.py
git commit -m "feat(cache): add cache stats summary helper"

# 查看你在哪里
git branch

# 查看所有分支的末端
git branch -v
```

---

### 任务 4.2：创建实验分支

使用分支进行实验性工作。

**说明：**

```bash
# 切换到 main
git switch main

# 创建实验分支
git switch -c experiment/onnx-runtime

# 添加实验性 ONNX 支持
mkdir -p src/models/onnx
cat > src/models/onnx/runtime.py << 'EOF'
"""
ONNX Runtime Integration (Experimental)

⚠️  EXPERIMENTAL - Not for production use!

Explores using ONNX Runtime for faster inference.
"""

import onnxruntime as ort
import numpy as np
from typing import List, Dict
import structlog

logger = structlog.get_logger()


class ONNXClassifier:
    """Experimental ONNX-based classifier."""

    def __init__(self, model_path: str):
        """
        Initialize ONNX classifier.

        Args:
            model_path: Path to ONNX model file
        """
        logger.warning("Loading experimental ONNX model")
        self.session = ort.InferenceSession(model_path)
        self.input_name = self.session.get_inputs()[0].name

    def predict(self, image_array: np.ndarray) -> List[Dict]:
        """
        Run inference with ONNX Runtime.

        Args:
            image_array: Preprocessed image array

        Returns:
            List of predictions
        """
        # Run inference
        outputs = self.session.run(None, {self.input_name: image_array})

        # Process outputs
        probabilities = outputs[0][0]
        top5_idx = np.argsort(probabilities)[-5:][::-1]

        predictions = []
        for idx in top5_idx:
            predictions.append({
                "class_id": int(idx),
                "confidence": float(probabilities[idx])
            })

        return predictions
EOF

git add src/models/onnx/
git commit -m "experiment: add ONNX Runtime integration

⚠️  EXPERIMENTAL BRANCH

Exploring ONNX Runtime for potential performance gains.

TODO:
- Benchmark against PyTorch
- Test accuracy parity
- Measure memory usage

DO NOT MERGE until thoroughly tested!"

# 在分支描述中标记为实验性
git config branch.experiment/onnx-runtime.description "Experimental ONNX Runtime support - do not merge"
```

---

## 第五部分：分支清理

### 任务 5.1：删除已合并的分支

学习清理已完成的分支。

**说明：**

```bash
# 首先，模拟一个已合并的分支
git switch main

# 创建一个小功能
git switch -c feature/update-readme

# 进行更改
cat >> README.md << 'EOF'

## Recent Updates

- Added batch inference support
- Implemented prediction caching
- Enhanced middleware with timeouts
EOF

git add README.md
git commit -m "docs: update README with recent features"

# 将其合并到 main（我们将在练习 04 中详细学习合并）
git switch main
git merge feature/update-readme --no-edit

# 现在功能分支已合并
git log --oneline -3

# 尝试删除分支
git branch -d feature/update-readme

# 成功！-d 标志只删除已合并的分支

# 列出剩余分支
git branch
```

**预期输出：**
```text
Deleted branch feature/update-readme (was abc123).
```

---

### 任务 5.2：删除未合并的分支

处理带有未合并工作的分支的删除。

**说明：**

```bash
# 尝试删除一个未合并的功能分支
git branch -d feature/batch-inference

# 错误！分支有未合并的变更
```

**预期输出：**
```text
error: The branch 'feature/batch-inference' is not fully merged.
If you are sure you want to delete it, run 'git branch -D feature/batch-inference'.
```

```bash
# Git 在保护你免于丢失工作！

# 如果你真的想删除它（小心！）
# git branch -D feature/batch-inference  # 不要运行这个

# 相反，让我们保留它
git branch

# 查看未合并的分支
git branch --no-merged

# 查看已合并的分支
git branch --merged
```

---

### 任务 5.3：重命名分支

练习重命名分支。

**说明：**

```bash
# 创建一个命名不佳的分支
git switch -c feature/new-stuff

# 哦，这是个糟糕的名字！
# 在分支上重命名它
git branch -m feature/prometheus-integration

# 验证
git branch

# 你也可以从另一个分支重命名
git switch main
git branch -m feature/prometheus-integration feature/monitoring-prometheus

# 验证
git branch
```

---

## 第六部分：高级分支操作

### 任务 6.1：跟踪远程分支

模拟使用远程分支。

**说明：**

```bash
# 查看包括远程跟踪的所有分支
git branch -a

# 目前你只有本地分支

# 模拟你可能看到的远程分支：
# * main
#   feature/batch-inference
#   feature/model-caching
#   remotes/origin/main
#   remotes/origin/feature/batch-inference

# 为分支设置上游
git switch feature/batch-inference

# 推送时，你会设置上游：
# git push -u origin feature/batch-inference

# 然后你可以看到跟踪信息
git branch -vv
```

---

### 任务 6.2：切换时 Stash 变更

处理切换分支时未提交的变更。

**说明：**

```bash
# 切换到一个功能分支
git switch feature/model-caching

# 进行更改但不提交
echo "# TODO: Add cache warming" >> src/utils/cache.py

# 检查状态
git status

# 尝试切换分支
git switch main

# 如果变更冲突，Git 可能会抱怨

# 解决方案 1：Stash 变更
git stash push -m "WIP: cache warming idea"

# 现在切换
git switch main

# 稍后，返回并恢复
git switch feature/model-caching
git stash list
git stash pop

# 解决方案 2：提交工作
git add src/utils/cache.py
git commit -m "wip: exploring cache warming"
```

---

## 第七部分：实用工作流

### 任务 7.1：完整功能开发工作流

练习完整的功能开发周期。

**说明：**

```bash
# 1. 从 main 开始
git switch main

# 2. 创建功能分支
git switch -c feature/health-check-enhancements

# 3. 开发功能
cat > src/api/health.py << 'EOF'
"""
Enhanced Health Check

Provides detailed health status.
"""

from fastapi import APIRouter
from src.models.classifier import ImageClassifier
import psutil
import os

router = APIRouter()


@router.get("/health/live")
async def liveness():
    """Liveness probe - is service running?"""
    return {"status": "alive"}


@router.get("/health/ready")
async def readiness(classifier: ImageClassifier):
    """Readiness probe - is service ready for traffic?"""
    ready = classifier.is_loaded()
    status_code = 200 if ready else 503
    return {
        "status": "ready" if ready else "not ready",
        "model_loaded": ready
    }, status_code


@router.get("/health/metrics")
async def health_metrics():
    """Health metrics for monitoring."""
    process = psutil.Process(os.getpid())
    return {
        "cpu_percent": psutil.cpu_percent(),
        "memory_mb": process.memory_info().rss / 1024 / 1024,
        "disk_usage_percent": psutil.disk_usage('/').percent
    }
EOF

git add src/api/health.py
git commit -m "feat(health): add enhanced health check endpoints

Add three health endpoints:
- /health/live: Liveness probe
- /health/ready: Readiness probe with model check
- /health/metrics: System resource metrics

Compatible with Kubernetes health checks."

# 4. 添加测试
cat > tests/unit/test_health.py << 'EOF'
"""Tests for health check endpoints."""

import pytest
from fastapi.testclient import TestClient
from src.api.app import app

client = TestClient(app)


def test_liveness():
    """Test liveness endpoint."""
    response = client.get("/health/live")
    assert response.status_code == 200
    assert response.json()["status"] == "alive"


def test_readiness():
    """Test readiness endpoint."""
    response = client.get("/health/ready")
    assert response.status_code in [200, 503]
    assert "model_loaded" in response.json()
EOF

git add tests/unit/test_health.py
git commit -m "test(health): add health endpoint tests"

# 5. 更新文档
cat >> docs/api.md << 'EOF'

## Health Check Endpoints

### GET /health/live
Kubernetes liveness probe.

### GET /health/ready
Kubernetes readiness probe. Returns 503 if model not loaded.

### GET /health/metrics
System resource metrics (CPU, memory, disk).
EOF

git add docs/api.md
git commit -m "docs: document health check endpoints"

# 6. 查看你的工作
git log --oneline

# 准备代码审查 / 合并！
```
