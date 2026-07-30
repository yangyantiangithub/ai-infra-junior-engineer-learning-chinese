# 练习 02：处理 Commit 与历史

## 概述

**难度**：入门
**预计时间**：75-90 分钟
**前置条件**：练习 01 - 创建你的第一个仓库，讲座 01 - Git 基础

在本练习中，你将学习高级 commit 技术以及如何有效使用 Git 历史。你将练习制作有效的 commit、查看和搜索历史、修改 commit 以及在真实的 ML 项目场景中还原变更。

## 学习目标

完成本练习后，你将：

- 制作原子性、文档良好的 commit，遵循最佳实践
- 使用常规格式编写专业的 commit 消息
- 使用各种 `git log` 选项浏览 commit 历史
- 搜索 commit 历史以查找特定变更
- 修改 commit 以修复错误
- 安全地还原变更而不丢失历史
- 使用 `git show` 检查特定 commit
- 理解何时以及如何修改 commit
- 对 ML 基础设施项目应用 Git 最佳实践

## 场景

你正在继续练习 01 中 ML 推理 API 的开发工作。你的团队已经建立了 commit 消息约定和代码审查实践。你需要实现新功能、修复 bug 并维护一个干净、可搜索的 Git 历史，帮助团队理解代码库的演进。

今天的任务：
- 添加模型性能监控
- 修复图像预处理中的 bug
- 实现请求频率限制
- 添加全面的日志功能
- 记录 API 端点

## 前置条件

开始之前，确保你有：

```bash
# 练习 01 中的 Git 仓库
cd ml-inference-api

# 验证你在正确的位置
git status
git log --oneline

# 应为干净的工作树和之前的 commit
```

如果你没有练习 01 的仓库，创建一个新的：

```bash
mkdir ml-inference-api
cd ml-inference-api
git init
git config init.defaultBranch main
# 按照练习 01 创建基本结构
```

---

## 第一部分：编写有效的 Commit 消息

### 任务 1.1：使用 Conventional Commits 实现功能

添加一个性能监控模块，遵循常规 commit 格式。

**说明：**

创建监控模块：

```bash
# 创建监控文件
cat > src/utils/monitoring.py << 'EOF'
"""
Performance Monitoring Module

Tracks model inference performance metrics.
"""

import time
from typing import Dict, List
from dataclasses import dataclass, field
from collections import defaultdict
import structlog

logger = structlog.get_logger()


@dataclass
class PerformanceMetrics:
    """Container for performance metrics."""

    inference_times: List[float] = field(default_factory=list)
    request_counts: Dict[str, int] = field(default_factory=lambda: defaultdict(int))
    error_counts: Dict[str, int] = field(default_factory=lambda: defaultdict(int))

    def record_inference(self, duration: float):
        """Record inference duration."""
        self.inference_times.append(duration)

    def record_request(self, endpoint: str):
        """Record API request."""
        self.request_counts[endpoint] += 1

    def record_error(self, error_type: str):
        """Record error occurrence."""
        self.error_counts[error_type] += 1

    def get_average_inference_time(self) -> float:
        """Calculate average inference time."""
        if not self.inference_times:
            return 0.0
        return sum(self.inference_times) / len(self.inference_times)

    def get_p95_inference_time(self) -> float:
        """Calculate 95th percentile inference time."""
        if not self.inference_times:
            return 0.0
        sorted_times = sorted(self.inference_times)
        idx = int(len(sorted_times) * 0.95)
        return sorted_times[idx]

    def get_metrics_summary(self) -> Dict:
        """Get summary of all metrics."""
        return {
            "average_inference_ms": self.get_average_inference_time() * 1000,
            "p95_inference_ms": self.get_p95_inference_time() * 1000,
            "total_requests": sum(self.request_counts.values()),
            "total_errors": sum(self.error_counts.values()),
            "requests_by_endpoint": dict(self.request_counts),
            "errors_by_type": dict(self.error_counts)
        }


class PerformanceMonitor:
    """Monitor and track performance metrics."""

    def __init__(self):
        """Initialize performance monitor."""
        self.metrics = PerformanceMetrics()
        self.start_time = time.time()

    def track_inference(self, func):
        """Decorator to track inference performance."""
        async def wrapper(*args, **kwargs):
            start = time.time()
            try:
                result = await func(*args, **kwargs)
                duration = time.time() - start
                self.metrics.record_inference(duration)
                logger.info("Inference completed", duration_ms=duration * 1000)
                return result
            except Exception as e:
                self.metrics.record_error(type(e).__name__)
                raise
        return wrapper

    def get_uptime(self) -> float:
        """Get service uptime in seconds."""
        return time.time() - self.start_time

    def reset_metrics(self):
        """Reset all metrics."""
        self.metrics = PerformanceMetrics()
        logger.info("Metrics reset")


# Global monitor instance
monitor = PerformanceMonitor()
EOF
```

现在使用常规 commit 格式提交：

```bash
# 暂存文件
git add src/utils/monitoring.py

# 使用常规格式提交
git commit -m "feat: add performance monitoring for inference tracking

Implement PerformanceMetrics and PerformanceMonitor classes to track:
- Inference latency (average and p95)
- Request counts per endpoint
- Error counts by type
- Service uptime

This enables real-time monitoring and helps identify performance
bottlenecks in the ML inference pipeline.

The monitor uses a decorator pattern for easy integration with
existing inference functions."
```

**检查点问题：**
- 消息中的"feat:"表示什么？
- 为什么消息中要包括变更内容和变更原因？
- 这在审查历史时有什么帮助？

---

### 任务 1.2：用适当的文档修复 Bug

查找并修复预处理模块中的 bug。

**说明：**

首先，创建 bug：

```bash
# 修改预处理文件以包含 bug
cat > src/preprocessing/image.py << 'EOF'
"""
Image Preprocessing Module

Handles image loading and preprocessing for model inference.
"""

import torch
from torchvision import transforms
from PIL import Image
import io
from typing import Union

# Standard ImageNet normalization
IMAGENET_MEAN = [0.485, 0.456, 0.406]
IMAGENET_STD = [0.229, 0.224, 0.225]


def get_preprocessing_transform():
    """
    Get image preprocessing transform.

    Returns:
        Composed transform for preprocessing
    """
    return transforms.Compose([
        transforms.Resize(256),
        transforms.CenterCrop(224),
        transforms.ToTensor(),
        transforms.Normalize(mean=IMAGENET_MEAN, std=IMAGENET_STD)
    ])


async def preprocess_image(image_bytes: bytes) -> torch.Tensor:
    """
    Preprocess image bytes for model input.

    Args:
        image_bytes: Raw image bytes

    Returns:
        Preprocessed image tensor
    """
    # Load image
    image = Image.open(io.BytesIO(image_bytes))

    # BUG: Not converting to RGB - causes issues with grayscale/RGBA images

    # Apply preprocessing
    transform = get_preprocessing_transform()
    image_tensor = transform(image)

    # Add batch dimension
    image_tensor = image_tensor.unsqueeze(0)

    return image_tensor


def validate_image(image_bytes: bytes, max_size_mb: int = 10) -> bool:
    """
    Validate that bytes represent a valid image.

    Args:
        image_bytes: Raw bytes
        max_size_mb: Maximum allowed file size in MB

    Returns:
        True if valid image, False otherwise
    """
    # Check file size
    size_mb = len(image_bytes) / (1024 * 1024)
    if size_mb > max_size_mb:
        return False

    try:
        image = Image.open(io.BytesIO(image_bytes))
        # Verify it's a valid image by loading
        image.verify()
        return True
    except Exception:
        return False
EOF

# 提交有 bug 的版本
git add src/preprocessing/image.py
git commit -m "refactor: improve image validation

Add file size checking to image validation function"
```

现在修复 bug：

```bash
# 修复 bug
cat > src/preprocessing/image.py << 'EOF'
"""
Image Preprocessing Module

Handles image loading and preprocessing for model inference.
"""

import torch
from torchvision import transforms
from PIL import Image
import io
from typing import Union

# Standard ImageNet normalization
IMAGENET_MEAN = [0.485, 0.456, 0.406]
IMAGENET_STD = [0.229, 0.224, 0.225]


def get_preprocessing_transform():
    """
    Get image preprocessing transform.

    Returns:
        Composed transform for preprocessing
    """
    return transforms.Compose([
        transforms.Resize(256),
        transforms.CenterCrop(224),
        transforms.ToTensor(),
        transforms.Normalize(mean=IMAGENET_MEAN, std=IMAGENET_STD)
    ])


async def preprocess_image(image_bytes: bytes) -> torch.Tensor:
    """
    Preprocess image bytes for model input.

    Args:
        image_bytes: Raw image bytes

    Returns:
        Preprocessed image tensor
    """
    # Load image
    image = Image.open(io.BytesIO(image_bytes))

    # Convert to RGB if needed (handles grayscale and RGBA)
    if image.mode != "RGB":
        image = image.convert("RGB")

    # Apply preprocessing
    transform = get_preprocessing_transform()
    image_tensor = transform(image)

    # Add batch dimension
    image_tensor = image_tensor.unsqueeze(0)

    return image_tensor


def validate_image(image_bytes: bytes, max_size_mb: int = 10) -> bool:
    """
    Validate that bytes represent a valid image.

    Args:
        image_bytes: Raw bytes
        max_size_mb: Maximum allowed file size in MB

    Returns:
        True if valid image, False otherwise
    """
    # Check file size
    size_mb = len(image_bytes) / (1024 * 1024)
    if size_mb > max_size_mb:
        return False

    try:
        image = Image.open(io.BytesIO(image_bytes))
        # Verify it's a valid image by loading
        image.verify()
        return True
    except Exception:
        return False
EOF

# 使用适当的文档提交修复
git add src/preprocessing/image.py
git commit -m "fix: convert images to RGB before preprocessing

The preprocessing pipeline now converts all images to RGB mode
before applying transformations. This fixes inference failures
when processing:
- Grayscale images (mode 'L')
- RGBA images with alpha channel
- Other non-RGB image formats

Without this conversion, the model receives incorrect tensor
dimensions causing inference to fail.

Fixes issue where grayscale medical images caused 500 errors."
```

**预期输出：**
```
[main abc123] fix: convert images to RGB before preprocessing
 1 file changed, 4 insertions(+), 1 deletion(-)
```

**检查点：**
查看你的 commit 历史：
```bash
git log --oneline -3
```

---

### 任务 1.3：添加配置功能

添加频率限制配置。

**说明：**

```bash
# 更新配置
cat >> configs/default.yaml << 'EOF'

# Rate Limiting
rate_limiting:
  enabled: true
  requests_per_minute: 100
  burst_size: 20
  strategy: "sliding_window"  # Options: fixed_window, sliding_window

# Request Timeout
timeout:
  request_timeout_seconds: 30
  inference_timeout_seconds: 10
EOF

# 创建频率限制器实现
cat > src/utils/rate_limiter.py << 'EOF'
"""
Rate Limiting Module

Implements request rate limiting for API endpoints.
"""

import time
from collections import deque
from typing import Optional
import structlog

logger = structlog.get_logger()


class RateLimiter:
    """Token bucket rate limiter."""

    def __init__(
        self,
        requests_per_minute: int = 100,
        burst_size: int = 20
    ):
        """
        Initialize rate limiter.

        Args:
            requests_per_minute: Maximum requests allowed per minute
            burst_size: Maximum burst capacity
        """
        self.requests_per_minute = requests_per_minute
        self.burst_size = burst_size
        self.tokens = burst_size
        self.last_update = time.time()
        self.requests = deque()

        logger.info(
            "Rate limiter initialized",
            rpm=requests_per_minute,
            burst=burst_size
        )

    def _refill_tokens(self):
        """Refill tokens based on elapsed time."""
        now = time.time()
        elapsed = now - self.last_update

        # Add tokens based on time elapsed
        tokens_to_add = elapsed * (self.requests_per_minute / 60.0)
        self.tokens = min(self.burst_size, self.tokens + tokens_to_add)
        self.last_update = now

    def allow_request(self) -> bool:
        """
        Check if request is allowed.

        Returns:
            True if request allowed, False if rate limited
        """
        self._refill_tokens()

        if self.tokens >= 1:
            self.tokens -= 1
            self.requests.append(time.time())
            logger.debug("Request allowed", tokens_remaining=self.tokens)
            return True

        logger.warning("Request rate limited", tokens_remaining=self.tokens)
        return False

    def get_wait_time(self) -> float:
        """
        Get time to wait before next request allowed.

        Returns:
            Seconds to wait, or 0 if request would be allowed
        """
        self._refill_tokens()

        if self.tokens >= 1:
            return 0.0

        # Calculate time needed to get 1 token
        return (1.0 - self.tokens) / (self.requests_per_minute / 60.0)

    def reset(self):
        """Reset rate limiter state."""
        self.tokens = self.burst_size
        self.last_update = time.time()
        self.requests.clear()
        logger.info("Rate limiter reset")
EOF

# 暂存两个文件
git add configs/default.yaml src/utils/rate_limiter.py

# 使用合适的 scope 提交
git commit -m "feat(api): implement request rate limiting

Add token bucket rate limiter to prevent API abuse:

Configuration:
- Configurable requests per minute limit
- Burst capacity for traffic spikes
- Sliding window strategy

Implementation:
- Token bucket algorithm
- Automatic token refill
- Wait time calculation
- Request tracking

Default settings: 100 requests/min with burst of 20.

This protects the service from abuse while allowing legitimate
traffic spikes."
```

---

## 第二部分：查看和搜索历史

### 任务 2.1：探索 Commit 历史

学习查看 Git 历史的不同方式。

**说明：**

```bash
# 基本日志
git log

# 紧凑单行格式
git log --oneline

# 最近 5 个 commit
git log --oneline -5

# 带有文件统计信息
git log --stat

# 带有实际变更
git log -p

# 特定数量的 commit
git log -p -2

# 图形化表示（与分支一起使用时很有用）
git log --oneline --graph --all

# 带有作者和日期的漂亮格式
git log --pretty=format:"%h - %an, %ar : %s"

# 最近 24 小时的 commit
git log --since="24 hours ago"

# 特定作者的 commit
git log --author="Your Name"

# 影响特定文件的 commit
git log -- src/utils/monitoring.py

# 消息中包含特定单词的 commit
git log --grep="fix"

# 查看每个 commit 的变化
git log --oneline --stat
```

**检查点练习：**

使用 `git log` 回答：

1. 你做了多少个 commit？
2. 你最近的 commit 消息是什么？
3. 你的第三个 commit 修改了哪些文件？
4. 有多少 commit 修改了 `configs/default.yaml`？

---

### 任务 2.2：搜索特定变更

练习搜索 commit 历史中的内容。

**说明：**

```bash
# 查找添加/删除了"RateLimiter"的 commit
git log -S "RateLimiter"

# 在 diff 中查找"RateLimiter"（更具体）
git log -G "RateLimiter"

# 显示实际的变更
git log -S "RateLimiter" -p

# 查找函数被添加的时间
git log -S "def preprocess_image" --source --all

# 搜索特定文件中的变更
git log -p -- src/preprocessing/image.py

# 显示更改了特定行范围的 commit
git log -L 30,50:src/utils/monitoring.py
```

**实用练习：**

使用 Git 回答：

1. `PerformanceMonitor` 类是何时添加的？
2. 哪个 commit 修复了 RGB 转换 bug？
3. `default.yaml` 做了哪些变更？

```bash
# 问题 1 的示例答案：
git log -S "class PerformanceMonitor" --oneline

# 问题 2 的示例：
git log --grep="RGB" --oneline

# 问题 3 的示例：
git log --oneline -- configs/default.yaml
git show <commit-hash>
```

---

### 任务 2.3：检查特定 Commit

使用 `git show` 详细检查 commit。

**说明：**

```bash
# 显示最新的 commit
git show

# 显示特定 commit
git show HEAD~1  # 前一个 commit
git show HEAD~2  # 两个 commit 前

# 通过哈希显示 commit
git log --oneline -5  # 获取哈希
git show abc123  # 替换为实际哈希

# 仅显示 commit 消息和统计信息
git show --stat HEAD

# 显示 commit 中的特定文件
git show HEAD:src/utils/monitoring.py

# 显示来自前一个 commit 的文件
git show HEAD~1:configs/default.yaml

# 比较 commit 之间的文件
git diff HEAD~1 HEAD -- src/preprocessing/image.py
```

**检查点：**

练习这些命令：

```bash
# 频率限制 commit 中更改了哪些文件？
git log --grep="rate limiting" --oneline
git show <hash> --stat

# 对预处理做了哪些精确更改？
git log -- src/preprocessing/image.py --oneline
git show <hash>
```

---

## 第三部分：修改 Commit

### 任务 3.1：修复 Commit 消息

练习修改最后一个 commit 的消息。

**说明：**

```bash
# 首先，制作一个有拼写错误的 commit
echo "# API Documentation" > docs/api.md
git add docs/api.md
git commit -m "doc: add API documentaton"  # 注意拼写错误

# 哦！修复 commit 消息
git commit --amend -m "docs: add API documentation

Create initial API documentation file for endpoint reference."

# 查看更正后的 commit
git log --oneline -1
```

**重要说明：**
- `--amend` 重写最后一个 commit
- 只修改尚未推送的 commit！
- 修改会变更 commit 哈希

---

### 任务 3.2：添加遗漏的文件

练习将文件添加到最后一个 commit。

**说明：**

```bash
# 添加端点文档
cat > docs/api.md << 'EOF'
# ML Inference API Documentation

## Endpoints

### POST /predict
Upload image for classification.

**Request:**
```
POST /predict
Content-Type: multipart/form-data

file: <image-file>
```

**Response:**
```json
{
  "success": true,
  "predictions": [
    {
      "class_id": 281,
      "class": "tabby_cat",
      "confidence": 0.94
    }
  ]
}
```

### GET /health
Check API health status.

**Response:**
```json
{
  "status": "healthy",
  "model_loaded": true
}
```

### GET /models
List available models.

**Response:**
```json
{
  "current_model": "resnet50",
  "version": "1.0.0"
}
```
EOF

# 提交它
git add docs/api.md
git commit -m "docs: add API endpoint documentation"

# 哦！忘记添加 README 更新
cat >> README.md << 'EOF'

## API Documentation

See [API Documentation](docs/api.md) for detailed endpoint information.
EOF

# 将遗漏的文件添加到最后一个 commit
git add README.md
git commit --amend --no-edit

# --no-edit 标志保留相同的 commit 消息
# 查看修改后的 commit
git show --stat
```

**检查点：**
- 最后一个 commit 现在包含哪些文件？
- 修改时 commit 哈希会怎样？

---

### 任务 3.3：同时修改消息和文件

练习同时修改消息和内容。

**说明：**

```bash
# 创建不完整的实现
cat > src/api/middleware.py << 'EOF'
"""
API Middleware

Custom middleware for request processing.
"""

from fastapi import Request
import time

async def logging_middleware(request: Request, call_next):
    """Log all requests."""
    start_time = time.time()
    response = await call_next(request)
    duration = time.time() - start_time
    print(f"{request.method} {request.url.path} - {duration:.3f}s")
    return response
EOF

git add src/api/middleware.py
git commit -m "feat: add middleware"

# 意识到我们应该添加频率限制并改进消息
cat > src/api/middleware.py << 'EOF'
"""
API Middleware

Custom middleware for request processing.
"""

from fastapi import Request, HTTPException
from fastapi.responses import JSONResponse
import time
import structlog

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
EOF

# 使用更好的消息和更新的文件来修改
git add src/api/middleware.py
git commit --amend -m "feat(api): add request logging and rate limiting middleware

Implement two middleware components:

1. Logging Middleware:
   - Logs all incoming requests
   - Tracks request duration
   - Uses structured logging

2. Rate Limiting Middleware:
   - Enforces rate limits per client
   - Returns 429 status when limited
   - Includes retry-after in response

These middleware protect the API and provide observability."

# 查看结果
git show
```

---

## 第四部分：还原变更

### 任务 4.1：还原一个 Commit

学习安全地撤销 commit。

**说明：**

```bash
# 首先，做一个坏的变更
cat > src/utils/cache.py << 'EOF'
"""
Caching Module - EXPERIMENTAL

WARNING: This implementation has issues!
"""

cache = {}  # Global cache - not thread-safe!

def get_from_cache(key):
    return cache.get(key)

def add_to_cache(key, value):
    cache[key] = value
EOF

git add src/utils/cache.py
git commit -m "feat: add caching module"

# 检查历史
git log --oneline -3

# 哦，这个缓存实现有问题！
# 还原该 commit（创建一个撤销变更的新 commit）
git revert HEAD

# Git 会打开编辑器，显示默认消息：
# "Revert "feat: add caching module""
#
# 你可以修改使其更具描述性：
git revert HEAD --no-commit  # 让我们用 no-commit 再试一次

# 实际上，让我们正确地做：
git revert --abort  # 取消上述操作
git revert HEAD     # 用编辑器还原

# 在编辑器中，将消息改为：
# Revert "feat: add caching module"
#
# The cache implementation has thread-safety issues.
# Will reimplement using proper locking mechanisms.

# 检查结果
git log --oneline -4
git show HEAD
```

**重要：** `git revert` 创建一个撤销变更的**新** commit。它不会删除历史！

---

### 任务 4.2：还原多个 Commit

练习还原一系列 commit。

**说明：**

```bash
# 做几个坏的 commit
echo "config1" > bad_config1.txt
git add bad_config1.txt
git commit -m "bad commit 1"

echo "config2" > bad_config2.txt
git add bad_config2.txt
git commit -m "bad commit 2"

echo "config3" > bad_config3.txt
git add bad_config3.txt
git commit -m "bad commit 3"

# 检查历史
git log --oneline -6

# 还原全部三个（按相反顺序）
git revert HEAD HEAD~1 HEAD~2 --no-edit

# 检查结果
git log --oneline -9
ls *.txt  # 文件应该已经消失
```

---

### 任务 4.3：Revert vs Reset（理解区别）

**说明：**

```bash
# 仅演示 - 不要在重要代码上运行

# 显示当前状态
git log --oneline

# git reset 移动分支指针（破坏性！）
# git reset --hard HEAD~1  # ⚠️ 删除最后一个 commit 及其变更！

# git revert 创建新 commit（安全！）
# git revert HEAD  # ✅ 保留历史，撤销变更

# 让我们以安全的方式演示 reset
# 创建一个测试分支
git branch test-branch
git checkout test-branch

# 制作一个 commit
echo "test" > test.txt
git add test.txt
git commit -m "test commit"

# 重置它（只在 test 分支上）
git reset --hard HEAD~1

# test.txt 消失，commit 消失
ls test.txt  # 文件未找到

# 回到 main
git checkout main

# test 分支的 commit 消失了，但 main 安全
```

**关键区别：**

| 命令 | 效果 | 历史 | 使用时机 |
|---------|--------|---------|----------|
| `git revert` | 撤销变更的新 commit | 保留 | 变更已推送/共享 |
| `git reset` | 移动分支指针 | 重写 | 变更仅本地 |

**规则：** 对已发布的 commit 使用 `revert`，仅对本地 commit 使用 `reset`！

---

## 第五部分：实用工作流练习

### 任务 5.1：完整功能开发工作流

按照最佳实践实现一个完整的功能。

**说明：**

```bash
# 1. 添加指标端点
cat > src/api/metrics.py << 'EOF'
"""
Metrics Endpoint

Exposes performance metrics for monitoring.
"""

from fastapi import APIRouter
from src.utils.monitoring import monitor

router = APIRouter()


@router.get("/metrics")
async def get_metrics():
    """
    Get performance metrics.

    Returns:
        Dictionary of current metrics
    """
    metrics = monitor.metrics.get_metrics_summary()
    metrics["uptime_seconds"] = monitor.get_uptime()
    return metrics


@router.post("/metrics/reset")
async def reset_metrics():
    """Reset all metrics (admin only)."""
    monitor.reset_metrics()
    return {"status": "metrics reset"}
EOF

git add src/api/metrics.py
git commit -m "feat(api): add metrics endpoint for monitoring

Expose /metrics endpoint to retrieve:
- Average inference time
- P95 inference latency
- Request counts by endpoint
- Error counts by type
- Service uptime

Includes /metrics/reset for admin use.

This enables integration with Prometheus and Grafana
for infrastructure monitoring."

# 2. 意识到需要将其添加到主应用
cat >> src/api/app.py << 'EOF'

# Import metrics router
from src.api.metrics import router as metrics_router

# Include metrics routes
app.include_router(metrics_router, tags=["monitoring"])
EOF

git add src/api/app.py
git commit -m "feat(api): register metrics router in main application

Include metrics endpoints in FastAPI app routing."

# 3. 添加测试
cat > tests/unit/test_metrics.py << 'EOF'
"""Tests for metrics endpoint."""

import pytest
from fastapi.testclient import TestClient
from src.api.app import app

client = TestClient(app)


def test_metrics_endpoint():
    """Test metrics endpoint returns data."""
    response = client.get("/metrics")
    assert response.status_code == 200
    data = response.json()
    assert "uptime_seconds" in data
    assert "average_inference_ms" in data


def test_metrics_reset():
    """Test metrics can be reset."""
    response = client.post("/metrics/reset")
    assert response.status_code == 200
    assert response.json()["status"] == "metrics reset"
EOF

git add tests/unit/test_metrics.py
git commit -m "test: add unit tests for metrics endpoint

Test metrics retrieval and reset functionality."

# 4. 更新文档
cat >> docs/api.md << 'EOF'

### GET /metrics
Get current performance metrics.

**Response:**
```json
{
  "average_inference_ms": 45.2,
  "p95_inference_ms": 78.5,
  "total_requests": 1523,
  "total_errors": 3,
  "uptime_seconds": 86400
}
```

### POST /metrics/reset
Reset all metrics (admin only).

**Response:**
```json
{
  "status": "metrics reset"
}
```
EOF

git add docs/api.md
git commit -m "docs: document metrics endpoints

Add documentation for metrics retrieval and reset endpoints."

# 5. 查看你的工作
git log --oneline -10
git log --graph --oneline --all
```

---

## 验证与审查

### 任务 6.1：审计你的 Commit 历史

审查你的 commit 质量。

**说明：**

```bash
# 查看所有 commit 消息
git log --oneline

# 检查 commit 消息质量
git log --pretty=format:"%s" | head -10

# 查找没有常规格式的 commit
git log --pretty=format:"%s" --grep="^feat:" --grep="^fix:" --grep="^docs:" --all-match --invert-grep

# 查看带有统计信息的 commit
git log --stat --oneline -10

# 检查大 commit（应为原子性）
git log --shortstat | grep -E "files? changed"
```

**质量检查清单：**

- [ ] Commits 遵循常规格式（feat:、fix:、docs: 等）
- [ ] Commit 消息解释为什么，而不仅仅是什么
- [ ] Commits 是原子性的（每个一个逻辑变更）
- [ ] 没有"WIP"或"fix typo"的 commit（应该被修改）
- [ ] 没有提交密钥或大文件

---

## 挑战任务（可选）

### 挑战 1：交互式 Rebase 预览

了解交互式 rebase（在练习 07 中会更多介绍）：

```bash
# 查看交互式 rebase 的样子
git rebase -i HEAD~5 --dry-run

# 先不要实际运行 - 只是查看界面
# 这显示了如何重新排序、压缩或编辑 commit
```

### 挑战 2：Bisect 查找 Bug

使用 `git bisect` 查找 bug 是何时引入的：

```bash
# 我们将模拟这个过程
# git bisect start
# git bisect bad                 # 当前版本坏了
# git bisect good HEAD~10        # 10 个 commit 前是好的
# Git 会检出中间的 commit
# 测试它，然后：
# git bisect good   # 或 bad
# 重复直到 Git 找到有问题的 commit
# git bisect reset  # 完成后
```

### 挑战 3：创建发布标签

标记一个重要的 commit：

```bash
# 将当前 commit 标记为发布
git tag -a v0.1.0 -m "Release version 0.1.0

Features:
- ML inference API
- Performance monitoring
- Rate limiting
- Comprehensive logging

This is the first stable release."

# 查看标签
git tag -l

# 查看标签详情
git show v0.1.0
```

---

## 故障排除

### 常见问题

**问题**："无法修改因为 commit 已推送"
```bash
# 解决方案：不要修改已推送的 commit！
# 而是，用修复创建一个新 commit
```

**问题**："意外修改了错误的 commit"
```bash
# 解决方案：使用 reflog 查找原始 commit
git reflog
git reset --hard HEAD@{1}  # 后退一步
```

**问题**："还原产生了冲突"
```bash
# 解决方案：手动解决冲突
git status  # 查看冲突文件
# 编辑文件解决冲突
git add <resolved-files>
git revert --continue
```

**问题**："找不到特定的 commit"
```bash
# 解决方案：使用更多搜索选项
git log --all --grep="search-term"
git log --all -S "code-snippet"
git reflog  # 显示所有 HEAD 移动
```

---

## 反思问题

1. **为什么使用常规 commit 格式（feat:、fix:、docs:）？**
   - 这对自动化和 changelog 生成有何帮助？

2. **何时应该修改 commit vs. 创建一个新 commit？**
   - 修改的风险是什么？

3. **`git revert` 和 `git reset` 之间的区别是什么？**
   - 何时使用哪个？

4. **为什么原子性 commit 实践很重要？**
   - 从本练习中举一个例子。

5. **良好的 commit 历史如何帮助调试？**
   - 你如何使用 `git log` 来查找 bug 是何时引入的？

6. **Commit 消息正文中应包含哪些信息？**
   - 为什么不能只依赖主题行？

---

## 总结

恭喜！你已经完成了练习 02。你学会了：

- ✅ 使用常规格式编写有效的 commit 消息
- ✅ 使用各种 `git log` 选项浏览 Git 历史
- ✅ 搜索 commit 历史以查找特定变更
- ✅ 使用 `git show` 检查 commit
- ✅ 修改 commit 以修复消息或添加文件
- ✅ 安全地还原 commit 而不丢失历史
- ✅ 理解 revert 与 reset 的区别
- ✅ 遵循完整的功能开发工作流
- ✅ 审计 commit 历史的质量

### 学会的关键命令

```bash
git commit -m "message"           # 创建 commit
git commit --amend                # 修改最后一个 commit
git log                           # 查看历史
git log --oneline                 # 紧凑历史
git log --stat                    # 带有文件统计信息
git log -p                        # 带有变更
git log --grep="term"             # 搜索消息
git log -S "code"                 # 搜索代码
git show <commit>                 # 显示 commit 详情
git revert <commit>               # 安全地撤销 commit
git reset                         # 移动分支（小心！）
git tag                           # 创建标签
```

### 下一步

- 完成练习 03 学习分支策略
- 在你的项目中练习 conventional commits
- 复习讲座 02 关于分支与合并的内容

**出色的工作！** 你现在拥有了维护干净、专业 Git 历史的技能。
