# 练习 04：合并与冲突解决

## 概述

**难度**：中级
**预计时间**：90-120 分钟
**前置条件**：练习 03 - 分支，讲座 02 - 分支与合并

在本练习中，你将学习如何合并功能分支、处理合并冲突以及在 ML 项目文件（包括代码、配置和数据文件）中解决冲突。你将练习不同的合并策略并学习何时使用每种方法。

## 学习目标

完成本练习后，你将：

- 执行快进合并
- 执行三方合并
- 识别和解决合并冲突
- 处理 Python 代码、YAML 配置和文档中的冲突
- 有效使用合并工具
- 理解合并策略（merge、squash、rebase）
- 中止和重试合并
- 处理复杂的多文件冲突
- 预防常见的合并错误

## 场景

你的团队一直在并行开发多个功能。现在是时候将它们整合回主分支了。有些功能可以干净地合并，而有些会有冲突需要手动解决。你需要仔细合并变更，同时保持代码质量和功能完整性。

需要合并的功能：
- 批量推理（在练习 03 中开发）
- 模型缓存
- 增强的健康检查
- 来自 hotfix 分支的 bug 修复

## 前置条件

```bash
# 从练习 03 的仓库继续
cd ml-inference-api

# 确保你有练习 03 中的分支
git branch

# 应显示：
# - main
# - feature/batch-inference
# - feature/model-caching
# - feature/health-check-enhancements
# - hotfix/fix-timeout
```

---

## 第一部分：快进合并

### 任务 1.1：理解快进

合并一个简单的、无冲突的分支。

**说明：**

```bash
# 切换到 main
git switch main

# 查看当前 commit
git log --oneline -1

# 查看健康检查分支
git log --oneline feature/health-check-enhancements

# 合并健康检查功能（快进）
git merge feature/health-check-enhancements

# 查看结果
git log --oneline --graph -5
```

**预期输出：**
```
Updating abc123..def456
Fast-forward
 docs/api.md             | 8 ++++++++
 src/api/health.py       | 45 +++++++++++++++++++++++++++++++++++++++++++
 tests/unit/test_health.py | 20 +++++++++++++++++++
 3 files changed, 73 insertions(+)
```

**检查点：**
- "Fast-forward"是什么意思？
- 为什么这次合并很简单？

---

### 任务 1.2：非快进合并

即使快进可能，也强制创建合并 commit。

**说明：**

```bash
# 创建一个简单分支
git switch -c feature/add-version-info

# 添加版本信息
cat > src/version.py << 'EOF'
"""Version information."""

__version__ = "0.2.0"
__api_version__ = "v1"

def get_version_info():
    """Get version information."""
    return {
        "version": __version__,
        "api_version": __api_version__
    }
EOF

git add src/version.py
git commit -m "feat: add version information module"

# 切换回来并使用 --no-ff 合并
git switch main
git merge --no-ff feature/add-version-info -m "Merge feature: version information

Adds version tracking module for API versioning."

# 查看图形 - 注意合并 commit
git log --oneline --graph -5
```

**为什么使用 --no-ff？**
- 保留分支历史
- 显示功能何时被集成
- 有助于代码审查跟踪

---

## 第二部分：三方合并

### 任务 2.1：基本三方合并

合并已经分叉的分支。

**说明：**

```bash
# main 和 feature/batch-inference 都有新的 commit
# 这需要一个三方合并

# 查看分歧
git log --oneline --graph --all -10

# 合并批量推理功能
git merge feature/batch-inference

# Git 打开编辑器输入合并 commit 消息
# 默认消息就好，或自定义：
# "Merge feature: batch inference
#
# Integrates batch prediction capability:
# - Process multiple images in single request
# - Configurable batch size (max 32)
# - Concurrent preprocessing
# - Batch validation"

# 查看合并 commit
git log --oneline --graph -8
git show HEAD
```

**预期结果：**
一个带有两个父 commit 的合并 commit（三方合并）。

---

## 第三部分：解决合并冲突

### 任务 3.1：创建并解决简单冲突

故意制造冲突以练习解决。

**说明：**

```bash
# 创建两个修改同一文件的分支

# 分支 1：更新默认配置
git switch main
git switch -c feature/config-update-1

cat >> configs/default.yaml << 'EOF'

# Feature Toggle
features:
  batch_inference: true
  caching: false
  metrics_export: true
EOF

git add configs/default.yaml
git commit -m "feat: add feature toggles to config"

# 分支 2：也更新配置（冲突变更）
git switch main
git switch -c feature/config-update-2

cat >> configs/default.yaml << 'EOF'

# Performance Settings
performance:
  max_workers: 4
  timeout_seconds: 30
  enable_profiling: false
EOF

git add configs/default.yaml
git commit -m "feat: add performance settings to config"

# 合并第一个分支
git switch main
git merge feature/config-update-1 --no-edit

# 尝试合并第二个分支 - 冲突！
git merge feature/config-update-2
```

**预期输出：**
```
Auto-merging configs/default.yaml
CONFLICT (content): Merge conflict in configs/default.yaml
Automatic merge failed; fix conflicts and then commit the result.
```

**解决方法：**

```bash
# 检查状态
git status

# 查看冲突
cat configs/default.yaml

# 你会看到冲突标记：
# <<<<<<< HEAD
# (your changes)
# =======
# (their changes)
# >>>>>>> feature/config-update-2

# 编辑文件来解决
# 移除冲突标记并保留两个部分：

cat > configs/default.yaml << 'EOF'
# Default Configuration for ML Inference API

server:
  host: "0.0.0.0"
  port: 8000
  workers: 4
  reload: false

model:
  name: "resnet50"
  version: "1.0.0"
  weights_path: "models/resnet50_weights.pth"
  device: "cpu"
  batch_size: 32

preprocessing:
  image_size: [224, 224]
  normalize_mean: [0.485, 0.456, 0.406]
  normalize_std: [0.229, 0.224, 0.225]

api:
  max_upload_size: 10485760
  allowed_extensions:
    - jpg
    - jpeg
    - png
  rate_limit: 100

logging:
  level: "INFO"
  format: "json"
  output: "stdout"

monitoring:
  enable_metrics: true
  metrics_port: 9090

# Rate Limiting
rate_limiting:
  enabled: true
  requests_per_minute: 100
  burst_size: 20
  strategy: "sliding_window"

# Request Timeout
timeout:
  request_timeout_seconds: 30
  inference_timeout_seconds: 10

# Prediction Cache
cache:
  enabled: true
  max_size: 1000
  ttl_seconds: 3600

# Feature Toggles
features:
  batch_inference: true
  caching: false
  metrics_export: true

# Performance Settings
performance:
  max_workers: 4
  timeout_seconds: 30
  enable_profiling: false
EOF

# 暂存已解决的文件
git add configs/default.yaml

# 检查状态 - 应显示"all conflicts fixed"
git status

# 完成合并
git commit -m "Merge feature: config updates

Resolved conflicts by integrating both feature toggles
and performance settings into default configuration."

# 验证
git log --oneline --graph -5
```

---

### 任务 3.2：解决 Python 代码冲突

处理源代码中的冲突。

**说明：**

```bash
# 创建对 API 的冲突更改

# 分支 1：添加请求验证
git switch main
git switch -c feature/request-validation

# 修改 app.py
cat > src/api/app.py << 'EOF'
"""ML Inference API - Main Application"""

from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.responses import JSONResponse
import structlog

from src.models.classifier import ImageClassifier
from src.preprocessing.image import preprocess_image, validate_image

logger = structlog.get_logger()

app = FastAPI(
    title="ML Inference API",
    description="REST API for image classification",
    version="0.2.0"
)

classifier = None

@app.on_event("startup")
async def startup_event():
    global classifier
    logger.info("Loading model...")
    classifier = ImageClassifier()
    await classifier.load_model()
    logger.info("Model loaded successfully")

@app.get("/")
async def root():
    return {
        "name": "ML Inference API",
        "version": "0.2.0",
        "status": "running"
    }

@app.get("/health")
async def health_check():
    if classifier is None or not classifier.is_loaded():
        raise HTTPException(status_code=503, detail="Model not loaded")
    return {
        "status": "healthy",
        "model_loaded": True
    }

@app.post("/predict")
async def predict(file: UploadFile = File(...)) -> JSONResponse:
    # Validate file type
    if not file.content_type.startswith("image/"):
        raise HTTPException(status_code=400, detail="File must be an image")

    # Read file
    image_bytes = await file.read()

    # Validate image
    if not validate_image(image_bytes):
        raise HTTPException(status_code=400, detail="Invalid image file")

    try:
        processed_image = await preprocess_image(image_bytes)
        predictions = await classifier.predict(processed_image)

        logger.info("Prediction made", filename=file.filename)

        return JSONResponse(content={
            "success": True,
            "predictions": predictions
        })
    except Exception as e:
        logger.error("Prediction failed", error=str(e))
        raise HTTPException(status_code=500, detail=f"Prediction failed: {str(e)}")
EOF

git add src/api/app.py
git commit -m "feat: add image validation to predict endpoint"

# 分支 2：添加指标跟踪
git switch main
git switch -c feature/metrics-tracking

cat > src/api/app.py << 'EOF'
"""ML Inference API - Main Application"""

from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.responses import JSONResponse
import structlog
import time

from src.models.classifier import ImageClassifier
from src.preprocessing.image import preprocess_image
from src.utils.monitoring import monitor

logger = structlog.get_logger()

app = FastAPI(
    title="ML Inference API",
    description="REST API for image classification",
    version="0.2.0"
)

classifier = None

@app.on_event("startup")
async def startup_event():
    global classifier
    logger.info("Loading model...")
    classifier = ImageClassifier()
    await classifier.load_model()
    logger.info("Model loaded successfully")

@app.get("/")
async def root():
    return {
        "name": "ML Inference API",
        "version": "0.2.0",
        "status": "running"
    }

@app.get("/health")
async def health_check():
    if classifier is None or not classifier.is_loaded():
        raise HTTPException(status_code=503, detail="Model not loaded")
    return {
        "status": "healthy",
        "model_loaded": True
    }

@app.post("/predict")
async def predict(file: UploadFile = File(...)) -> JSONResponse:
    # Track request
    monitor.metrics.record_request("/predict")
    start_time = time.time()

    # Validate file type
    if not file.content_type.startswith("image/"):
        monitor.metrics.record_error("InvalidFileType")
        raise HTTPException(status_code=400, detail="File must be an image")

    try:
        image_bytes = await file.read()
        processed_image = await preprocess_image(image_bytes)
        predictions = await classifier.predict(processed_image)

        # Record inference time
        inference_time = time.time() - start_time
        monitor.metrics.record_inference(inference_time)

        logger.info("Prediction made", filename=file.filename, duration=inference_time)

        return JSONResponse(content={
            "success": True,
            "predictions": predictions
        })
    except Exception as e:
        monitor.metrics.record_error(type(e).__name__)
        logger.error("Prediction failed", error=str(e))
        raise HTTPException(status_code=500, detail=f"Prediction failed: {str(e)}")
EOF

git add src/api/app.py
git commit -m "feat: add metrics tracking to predict endpoint"

# 合并第一个功能
git switch main
git merge feature/request-validation --no-edit

# 合并第二个 - 冲突！
git merge feature/metrics-tracking
```

**解决冲突：**

```bash
# 查看冲突
git diff

# predict 函数有冲突
# 需要结合两个变更：验证 AND 指标

# 编辑 src/api/app.py 包含两个功能：

cat > src/api/app.py << 'EOF'
"""ML Inference API - Main Application"""

from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.responses import JSONResponse
import structlog
import time

from src.models.classifier import ImageClassifier
from src.preprocessing.image import preprocess_image, validate_image
from src.utils.monitoring import monitor

logger = structlog.get_logger()

app = FastAPI(
    title="ML Inference API",
    description="REST API for image classification",
    version="0.2.0"
)

classifier = None

@app.on_event("startup")
async def startup_event():
    global classifier
    logger.info("Loading model...")
    classifier = ImageClassifier()
    await classifier.load_model()
    logger.info("Model loaded successfully")

@app.get("/")
async def root():
    return {
        "name": "ML Inference API",
        "version": "0.2.0",
        "status": "running"
    }

@app.get("/health")
async def health_check():
    if classifier is None or not classifier.is_loaded():
        raise HTTPException(status_code=503, detail="Model not loaded")
    return {
        "status": "healthy",
        "model_loaded": True
    }

@app.post("/predict")
async def predict(file: UploadFile = File(...)) -> JSONResponse:
    # Track request (from metrics-tracking branch)
    monitor.metrics.record_request("/predict")
    start_time = time.time()

    # Validate file type
    if not file.content_type.startswith("image/"):
        monitor.metrics.record_error("InvalidFileType")
        raise HTTPException(status_code=400, detail="File must be an image")

    # Read file
    image_bytes = await file.read()

    # Validate image (from request-validation branch)
    if not validate_image(image_bytes):
        monitor.metrics.record_error("InvalidImage")
        raise HTTPException(status_code=400, detail="Invalid image file")

    try:
        processed_image = await preprocess_image(image_bytes)
        predictions = await classifier.predict(processed_image)

        # Record inference time (from metrics-tracking branch)
        inference_time = time.time() - start_time
        monitor.metrics.record_inference(inference_time)

        logger.info("Prediction made", filename=file.filename, duration=inference_time)

        return JSONResponse(content={
            "success": True,
            "predictions": predictions
        })
    except Exception as e:
        monitor.metrics.record_error(type(e).__name__)
        logger.error("Prediction failed", error=str(e))
        raise HTTPException(status_code=500, detail=f"Prediction failed: {str(e)}")
EOF

# 暂存并提交
git add src/api/app.py
git commit -m "Merge feature: metrics tracking

Combined metrics tracking with image validation.
Both features now work together:
- Validate images before processing
- Track all requests and errors
- Record inference timing"

# 验证
git log --oneline --graph -6
```

---

### 任务 3.3：解决多文件冲突

处理跨多个文件的冲突。

**说明：**

```bash
# 尝试合并模型缓存功能
git merge feature/model-caching

# 如果 app.py 被修改过，可能会有冲突

# 检查哪些文件有冲突
git status

# 解决每个冲突
# 然后：
git add <resolved-files>
git commit -m "Merge feature: model caching

Integrated prediction caching with existing features."
```

---

## 第四部分：合并策略

### 任务 4.1：Squash 合并

将功能分支的所有 commit 合并为一个。

**说明：**

```bash
# 创建一个包含多个小 commit 的功能
git switch -c feature/logging-improvements

echo "improvement 1" >> src/utils/logging.py
git add src/utils/logging.py
git commit -m "logging: add timestamp format"

echo "improvement 2" >> src/utils/logging.py
git add src/utils/logging.py
git commit -m "logging: add log rotation"

echo "improvement 3" >> src/utils/logging.py
git add src/utils/logging.py
git commit -m "logging: add structured fields"

# 查看 commit
git log --oneline -3

# 切换并 squash 合并
git switch main
git merge --squash feature/logging-improvements

# 所有变更已暂存，但尚未提交
git status

# 作为单个 commit 提交
git commit -m "feat: improve logging system

Consolidated improvements:
- Timestamp formatting
- Log rotation
- Structured field support"

# 注意：main 上只有一个 commit，不是三个
git log --oneline -3
```

**何时使用 squash：**
- 功能有许多 WIP commit
- 希望在 main 上保持干净的历史
- 单独的 commit 不重要

---

### 任务 4.2：中止合并

学会取消有问题的合并。

**说明：**

```bash
# 开始一个会产生冲突的合并
git switch -c feature/conflicting-change

# 进行冲突性变更
cat > src/api/app.py << 'EOF'
# Completely different implementation
print("This will conflict!")
EOF

git add src/api/app.py
git commit -m "Conflicting change"

git switch main
git merge feature/conflicting-change

# 大量冲突！
git status

# 决定中止并稍后重试
git merge --abort

# 回到合并前状态
git status
git diff  # No changes
```

---

## 第五部分：高级冲突解决

### 任务 5.1：使用合并工具

配置和使用合并工具。

**说明：**

```bash
# 配置合并工具（可选）
git config merge.tool vimdiff
# 或：meld, kdiff3, vscode 等

# 当你有冲突时：
# git mergetool

# 这会打开一个可视化差异工具
# 进行更改，保存，退出

# 验证解决
# git add <file>
# git commit
```

---

### 任务 5.2：选择一方

完全保留一个版本。

**说明：**

```bash
# 在冲突期间，选择一方：

# 保留"ours"（当前分支）
git checkout --ours path/to/file
git add path/to/file

# 或保留"theirs"（合并分支）
git checkout --theirs path/to/file
git add path/to/file

# 然后提交
git commit
```

---

## 第六部分：预防合并问题

### 任务 6.1：合并前检查

在合并前进行验证。

**说明：**

```bash
# 在合并前：

# 1. 更新 main
git switch main
git pull  # (if working with remote)

# 2. 在不合并的情况下检查冲突
git merge --no-commit --no-ff feature/branch-name

# 审查变更
git status
git diff --cached

# 如果没问题，提交
git commit

# 如果不行，中止
git merge --abort
```

---

### 任务 6.2：合并后测试

创建合并验证脚本。

**说明：**

```bash
cat > scripts/post-merge-check.sh << 'EOF'
#!/bin/bash
# Post-merge validation

echo "Running post-merge checks..."

# Check syntax
echo "1. Checking Python syntax..."
python -m py_compile src/**/*.py

# Run tests
echo "2. Running tests..."
pytest tests/ -v

# Check for conflict markers
echo "3. Checking for unresolved conflicts..."
if grep -r "<<<<<<< HEAD" src/; then
    echo "ERROR: Unresolved conflict markers found!"
    exit 1
fi

echo "All checks passed!"
EOF

chmod +x scripts/post-merge-check.sh

# 每次合并后运行
./scripts/post-merge-check.sh
```

---

## 验证

### 验证你的合并

```bash
# 查看合并历史
git log --oneline --graph --all

# 检查未解决的冲突
git status

# 验证所有功能已集成
ls src/api/
ls src/utils/

# 运行测试
# pytest tests/

# 检查 commit 历史
git log --merges --oneline
```

---

## 挑战任务

### 挑战 1：带冲突的三方合并

创建并解决一个包含多个冲突文件的复杂三方合并。

### 挑战 2：Cherry-Pick Commits

使用 `git cherry-pick` 选择性应用 commit：

```bash
# 从另一个分支挑选特定 commit
git log feature/some-branch --oneline
git cherry-pick <commit-hash>
```

### 挑战 3：合并策略比较

比较不同的合并策略并记录何时使用每种策略。

---

## 故障排除

**问题**："无法合并，有未提交的变更"
```bash
git stash
git merge <branch>
git stash pop
```

**问题**："合并产生了意外结果"
```bash
# 撤销合并
git reset --hard HEAD~1
```

**问题**："忘记了我正在合并什么"
```bash
git status  # Shows merge in progress
cat .git/MERGE_HEAD  # Shows branch being merged
```

---

## 反思问题

1. **快进合并和三方合并之间的区别是什么？**
2. **何时应该使用 `--squash` 而不是常规合并？**
3. **如何识别合并冲突的原因？**
4. **处理复杂冲突最安全的方式是什么？**
5. **为什么要在合并后进行验证？**

---

## 总结

恭喜！你已经掌握了合并和冲突解决：

- ✅ 快进合并
- ✅ 三方合并
- ✅ 代码和配置中的冲突解决
- ✅ 合并策略（squash、no-ff）
- ✅ 中止有问题的合并
- ✅ 使用合并工具
- ✅ 合并前和合并后验证

### 学会的关键命令

```bash
git merge <branch>           # 合并分支
git merge --no-ff <branch>   # 强制创建合并 commit
git merge --squash <branch>  # Squash 合并
git merge --abort            # 取消合并
git checkout --ours <file>   # 保留我们的版本
git checkout --theirs <file> # 保留他们的版本
git mergetool                # 打开合并工具
git log --merges             # 显示合并 commit
```

### 下一步

- 练习 05：协作工作流
- 在团队项目中练习合并

**出色的工作！** 你现在可以自信地合并和解决冲突了。
