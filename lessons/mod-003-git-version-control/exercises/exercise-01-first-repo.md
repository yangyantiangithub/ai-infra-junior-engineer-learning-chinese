# 练习 01：创建你的第一个 ML 项目仓库

## 概述

**难度**：入门
**预计时间**：60-90 分钟
**前置条件**：讲座 01 - Git 基础

在本练习中，你将从头开始为机器学习项目创建你的第一个 Git 仓库。你将学习正确的仓库初始化、文件组织、`.gitignore` 配置以及制作你的第一个 commit。本练习模拟启动一个新的 ML 推理 API 项目。

## 学习目标

完成本练习后，你将：

- 正确初始化 Git 仓库
- 为 ML 项目配置合适的 Git 设置
- 为 Python 和 ML 项目创建合适的 `.gitignore` 文件
- 为 ML 项目组织仓库结构
- 制作原子性、文档良好的 commit
- 在实践中理解 Git 三态工作流
- 有效使用 `git status` 和 `git diff`
- 编写专业的 commit 消息

## 场景

你的任务是创建一个新的机器学习推理 API，该 API 将使用预训练的图像分类模型提供预测服务。该项目从头开始，你需要从一开始就正确设置仓库。

该 API 将：
- 通过 REST API 接受图像上传
- 使用 PyTorch 模型运行推理
- 返回带有置信度分数的分类结果
- 记录预测以进行监控
- 支持多个模型版本

## 前置条件

开始之前，确保你有：

```bash
# 安装 Git
git --version
# 应显示：git version 2.x.x or higher

# 安装 Python 3.11+
python3 --version

# 文本编辑器（VS Code、vim、nano 等）
```

---

## 第一部分：初始设置与配置

### 任务 1.1：配置 Git 身份

在创建你的第一个仓库之前，配置你的 Git 身份。

**说明：**

```bash
# 设置你的姓名（使用真实姓名或 GitHub 用户名）
git config --global user.name "Your Name"

# 设置你的邮箱（使用你的邮箱）
git config --global user.email "your.email@example.com"

# 验证配置
git config --list --show-origin
```

**预期输出：**
```text
file:/home/user/.gitconfig    user.name=Your Name
file:/home/user/.gitconfig    user.email=your.email@example.com
```

**检查点问题：**
- 为什么在制作 commit 之前设置 user.name 和 user.email 很重要？

---

### 任务 1.2：创建项目目录并初始化仓库

创建项目目录并将其初始化为 Git 仓库。

**说明：**

```bash
# 创建项目目录
mkdir ml-inference-api
cd ml-inference-api

# 初始化 Git 仓库
git init

# 验证仓库已创建
ls -la
```

**预期输出：**
```text
Initialized empty Git repository in /path/to/ml-inference-api/.git/
```

你应该看到 `.git` 目录（它是隐藏的，所以使用 `ls -la`）。

**检查点问题：**
- `.git` 目录包含什么？
- 什么命令显示仓库的当前状态？

现在运行 `git status`。它显示什么？

---

### 任务 1.3：配置仓库特定设置

设置 ML 项目特定的 Git 配置。

**说明：**

```bash
# 将默认分支名称设置为 'main'（现代约定）
git config init.defaultBranch main

# 配置换行符（对跨平台团队很重要）
# Linux/Mac：
git config core.autocrlf input
# Windows：
# git config core.autocrlf true

# 为大文件增加缓冲区大小
git config http.postBuffer 524288000

# 创建有用的别名
git config alias.st status
git config alias.co checkout
git config alias.br branch
git config alias.ci commit
git config alias.unstage 'reset HEAD --'
git config alias.last 'log -1 HEAD --stat'

# 验证仓库配置
git config --local --list
```

**检查点：**
测试一个别名：
```bash
git st
# 应显示：On branch main, No commits yet, nothing to commit
```

---

## 第二部分：创建项目结构

### 任务 2.1：创建目录结构

为 ML 项目创建专业的目录结构。

**说明：**

```bash
# 创建目录结构
mkdir -p src/{api,models,preprocessing,utils}
mkdir -p tests/{unit,integration}
mkdir -p configs
mkdir -p scripts
mkdir -p docs

# 创建初始文件
touch README.md
touch requirements.txt
touch requirements-dev.txt
touch setup.py
touch .env.example
touch .gitignore
touch .gitattributes

# 创建源代码文件
touch src/__init__.py
touch src/api/__init__.py
touch src/api/app.py
touch src/models/__init__.py
touch src/models/classifier.py
touch src/preprocessing/__init__.py
touch src/preprocessing/image.py
touch src/utils/__init__.py
touch src/utils/logging.py

# 创建测试文件
touch tests/__init__.py
touch tests/unit/__init__.py
touch tests/unit/test_classifier.py
touch tests/integration/__init__.py

# 创建配置文件
touch configs/default.yaml
touch configs/production.yaml

# 创建脚本
touch scripts/download_model.sh
touch scripts/run_server.sh

# 验证结构
tree -L 2
```

**预期结构：**
```text
ml-inference-api/
├── .env.example
├── .gitattributes
├── .gitignore
├── README.md
├── requirements-dev.txt
├── requirements.txt
├── setup.py
├── configs/
│   ├── default.yaml
│   └── production.yaml
├── docs/
├── scripts/
│   ├── download_model.sh
│   └── run_server.sh
├── src/
│   ├── __init__.py
│   ├── api/
│   ├── models/
│   ├── preprocessing/
│   └── utils/
└── tests/
    ├── __init__.py
    ├── integration/
    └── unit/
```

**检查点：**
运行 `git status`。它显示了哪些文件？

---

### 任务 2.2：创建 .gitignore 文件

为 ML 项目创建全面的 `.gitignore` 文件。

**说明：**

创建 `.gitignore` 并写入以下内容：

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
ENV/
env.bak/
venv.bak/
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
pip-wheel-metadata/
share/python-wheels/
*.egg-info/
.installed.cfg
*.egg
MANIFEST

# Virtual Environments
.env
.venv
ENV/
env/
venv/

# IDEs
.vscode/
.idea/
*.swp
*.swo
*~
.DS_Store

# Jupyter Notebooks
.ipynb_checkpoints
*.ipynb_checkpoints/

# Machine Learning
*.h5
*.hdf5
*.pkl
*.pickle
*.pt
*.pth
*.onnx
*.pb
saved_models/
checkpoints/
models/
*.model
*.weights

# Data
data/
datasets/
*.csv
*.tsv
*.dat
*.json
!configs/*.json
!configs/*.yaml
*.parquet
*.feather

# Logs and Experiment Tracking
logs/
*.log
mlruns/
.mlflow/
wandb/
runs/
tensorboard/

# Credentials and Secrets
.env
.env.local
.env.*.local
secrets/
*.key
*.pem
credentials.json
service-account.json
.aws/
.gcloud/

# Testing
.coverage
.pytest_cache/
.tox/
htmlcov/
.hypothesis/

# Documentation
docs/_build/
site/

# Compiled files
*.com
*.class
*.dll
*.exe
*.o

# Archives
*.zip
*.tar
*.tar.gz
*.rar
*.7z

# Temporary files
*.tmp
*.bak
*.swp
.cache/
```

**保存文件。**

**检查点：**
```bash
git status
```

注意 `data/`、`models/` 和其他被忽略的目录不会出现在状态中。

---

### 任务 2.3：创建 .gitattributes 文件

为 ML 项目配置文件处理。

**说明：**

创建 `.gitattributes` 并写入以下内容：

```gitattributes
# Auto detect text files and perform LF normalization
* text=auto

# Python files
*.py text eol=lf
*.pyi text eol=lf
*.pyx text eol=lf

# YAML files
*.yml text eol=lf
*.yaml text eol=lf

# Markdown
*.md text eol=lf

# Shell scripts
*.sh text eol=lf

# Binary files (don't attempt to diff)
*.pkl binary
*.pickle binary
*.h5 binary
*.hdf5 binary
*.pt binary
*.pth binary
*.onnx binary
*.pb binary
*.parquet binary
*.feather binary

# Images
*.jpg binary
*.jpeg binary
*.png binary
*.gif binary
*.ico binary
*.svg text

# Archives
*.zip binary
*.tar binary
*.gz binary
*.7z binary
*.rar binary
```

**保存文件。**

---

## 第三部分：创建初始内容

### 任务 3.1：创建 README.md

为项目创建专业的 README。

**说明：**

编辑 `README.md`：

```markdown
# ML Inference API

A production-ready REST API for serving image classification predictions using PyTorch.

## Overview

This API provides endpoints for:
- Image classification with confidence scores
- Model version management
- Prediction logging and monitoring
- Health checks and metrics

## Tech Stack

- **Framework**: FastAPI
- **ML Framework**: PyTorch
- **Model**: ResNet-50 (pre-trained on ImageNet)
- **Containerization**: Docker
- **Deployment**: Kubernetes

## Project Status

🚧 **In Development**

Current version: 0.1.0

## Getting Started

### Prerequisites

- Python 3.11+
- pip
- Virtual environment tool (venv, conda, etc.)

### Installation

```bash
# Clone repository
git clone <repository-url>
cd ml-inference-api

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Download model weights
bash scripts/download_model.sh
```

### Running Locally

```bash
# Start development server
bash scripts/run_server.sh
```

API will be available at `http://localhost:8000`

### Testing

```bash
# Run tests
pytest tests/

# Run with coverage
pytest tests/ --cov=src
```

## API Endpoints

- `POST /predict` - Upload image for classification
- `GET /health` - Health check
- `GET /metrics` - Prometheus metrics
- `GET /models` - List available model versions

## Configuration

Configuration files in `configs/`:
- `default.yaml` - Default settings
- `production.yaml` - Production overrides

## Project Structure

```
ml-inference-api/
├── src/           # Source code
├── tests/         # Test suite
├── configs/       # Configuration files
├── scripts/       # Utility scripts
└── docs/          # Documentation
```

## Contributing

1. Create feature branch: `git checkout -b feature/new-feature`
2. Make changes and commit: `git commit -m "Add new feature"`
3. Push to branch: `git push origin feature/new-feature`
4. Create Pull Request

## License

MIT License - see LICENSE file for details

## Authors

- Your Name <your.email@example.com>

## Acknowledgments

- PyTorch team for the pre-trained models
- FastAPI for the excellent web framework
```

**保存文件。**

---

### 任务 3.2：创建 requirements.txt

为项目创建依赖文件。

**说明：**

编辑 `requirements.txt`：

```text
# Web Framework
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0

# Machine Learning
torch==2.1.0
torchvision==0.16.0
numpy==1.26.0
pillow==10.1.0

# Utilities
pyyaml==6.0.1
python-dotenv==1.0.0
python-multipart==0.0.6

# Logging and Monitoring
structlog==23.2.0
prometheus-client==0.19.0

# HTTP Client
httpx==0.25.1
```

编辑 `requirements-dev.txt`：

```text
-r requirements.txt

# Testing
pytest==7.4.3
pytest-cov==4.1.0
pytest-asyncio==0.21.1

# Linting and Formatting
black==23.11.0
flake8==6.1.0
mypy==1.7.0
isort==5.12.0

# Pre-commit hooks
pre-commit==3.5.0
```

**保存两个文件。**

---

### 任务 3.3：创建配置文件

创建初始配置文件。

**说明：**

编辑 `configs/default.yaml`：

```yaml
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
  device: "cpu"  # Options: cpu, cuda
  batch_size: 32

preprocessing:
  image_size: [224, 224]
  normalize_mean: [0.485, 0.456, 0.406]
  normalize_std: [0.229, 0.224, 0.225]

api:
  max_upload_size: 10485760  # 10 MB
  allowed_extensions:
    - jpg
    - jpeg
    - png
  rate_limit: 100  # requests per minute

logging:
  level: "INFO"
  format: "json"
  output: "stdout"

monitoring:
  enable_metrics: true
  metrics_port: 9090
```

编辑 `configs/production.yaml`：

```yaml
# Production Configuration Overrides

server:
  reload: false
  workers: 8

model:
  device: "cuda"
  batch_size: 64

logging:
  level: "WARNING"
  output: "file"
  file_path: "/var/log/ml-api/app.log"

monitoring:
  enable_metrics: true
```

**保存两个文件。**

---

### 任务 3.4：创建 .env.example

创建环境变量模板。

**说明：**

编辑 `.env.example`：

```bash
# Environment Variables Template
# Copy this file to .env and fill in actual values
# NEVER commit .env to Git!

# Application
ENVIRONMENT=development
DEBUG=true
SECRET_KEY=your-secret-key-here

# Model Storage
MODEL_STORAGE_PATH=./models
MODEL_CACHE_SIZE=1000

# AWS Credentials (if using S3 for model storage)
# AWS_ACCESS_KEY_ID=your-access-key
# AWS_SECRET_ACCESS_KEY=your-secret-key
# AWS_REGION=us-west-2
# S3_BUCKET=ml-models

# Database (if needed)
# DATABASE_URL=postgresql://user:password@localhost:5432/mlapi

# Redis (if using for caching)
# REDIS_URL=redis://localhost:6379/0

# Monitoring
# SENTRY_DSN=your-sentry-dsn
```

**保存文件。**

**重要：** 在本练习中不要创建实际的 `.env` 文件。`.gitignore` 会阻止它被跟踪。

---

## 第四部分：制作你的第一个 Commit

### 任务 4.1：暂存并审查变更

现在我们有了内容，让我们准备我们的第一个 commit。

**说明：**

```bash
# 检查状态
git status

# 你应该会看到许多"Untracked files"

# 将所有文件添加到暂存区
git add .

# 再次检查状态
git status

# 审查已暂存的内容
git diff --staged
```

**预期输出：**
```text
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   .env.example
        new file:   .gitattributes
        new file:   .gitignore
        new file:   README.md
        ... (many more files)
```

**检查点问题：**
- `git diff` 和 `git diff --staged` 之间的区别是什么？
- 为什么我们在暂存区中看不到 `data/` 或 `models/` 目录？

---

### 任务 4.2：制作第一个 Commit

使用合适的消息创建你的第一个 commit。

**说明：**

```bash
# 使用描述性消息提交
git commit -m "Initial project structure

Set up ML inference API project with:
- Directory structure for source code, tests, and configs
- Python package structure with __init__.py files
- Comprehensive .gitignore for ML projects
- .gitattributes for proper file handling
- README with project overview
- Requirements files for dependencies
- Configuration templates (default and production)
- Environment variable template

This establishes the foundation for the image classification API."

# 查看 commit
git log

# 查看 commit 详情
git show

# 查看 commit 统计信息
git show --stat
```

**预期输出：**
```text
[main (root-commit) abc123] Initial project structure
 25 files changed, 350 insertions(+)
 create mode 100644 .env.example
 create mode 100644 .gitattributes
 ... (list of files)
```

**检查点：**
你的仓库现在有一个 commit 了！运行 `git log --oneline` 查看它。

---

### 任务 4.3：添加源代码文件

现在让我们向项目添加一些实际代码。

**说明：**

编辑 `src/api/app.py`：

```python
"""
ML Inference API - Main Application

FastAPI application for serving ML model predictions.
"""

from fastapi import FastAPI, File, UploadFile, HTTPException
from fastapi.responses import JSONResponse
import structlog
from typing import Dict, List

from src.models.classifier import ImageClassifier
from src.preprocessing.image import preprocess_image

# Configure logging
logger = structlog.get_logger()

# Initialize FastAPI app
app = FastAPI(
    title="ML Inference API",
    description="REST API for image classification",
    version="0.1.0"
)

# Initialize classifier (will be loaded on startup)
classifier = None


@app.on_event("startup")
async def startup_event():
    """Load model on application startup."""
    global classifier
    logger.info("Loading model...")
    classifier = ImageClassifier()
    await classifier.load_model()
    logger.info("Model loaded successfully")


@app.get("/")
async def root():
    """Root endpoint - API information."""
    return {
        "name": "ML Inference API",
        "version": "0.1.0",
        "status": "running"
    }


@app.get("/health")
async def health_check():
    """Health check endpoint."""
    if classifier is None or not classifier.is_loaded():
        raise HTTPException(status_code=503, detail="Model not loaded")

    return {
        "status": "healthy",
        "model_loaded": True
    }


@app.post("/predict")
async def predict(file: UploadFile = File(...)) -> JSONResponse:
    """
    Predict image classification.

    Args:
        file: Uploaded image file

    Returns:
        JSON response with predictions
    """
    # Validate file type
    if not file.content_type.startswith("image/"):
        raise HTTPException(
            status_code=400,
            detail="File must be an image"
        )

    try:
        # Read and preprocess image
        image_bytes = await file.read()
        processed_image = await preprocess_image(image_bytes)

        # Get predictions
        predictions = await classifier.predict(processed_image)

        logger.info(
            "Prediction made",
            filename=file.filename,
            top_class=predictions[0]["class"]
        )

        return JSONResponse(content={
            "success": True,
            "predictions": predictions
        })

    except Exception as e:
        logger.error("Prediction failed", error=str(e))
        raise HTTPException(
            status_code=500,
            detail=f"Prediction failed: {str(e)}"
        )


@app.get("/models")
async def list_models() -> Dict[str, str]:
    """List available models."""
    if classifier is None:
        return {"models": []}

    return {
        "current_model": classifier.model_name,
        "version": classifier.version
    }
```

**保存文件。**

编辑 `src/models/classifier.py`：

```python
"""
Image Classifier Module

Handles loading and inference for image classification models.
"""

import torch
import torchvision.models as models
from typing import List, Dict
import structlog

logger = structlog.get_logger()


class ImageClassifier:
    """Image classification model wrapper."""

    def __init__(self, model_name: str = "resnet50", device: str = "cpu"):
        """
        Initialize classifier.

        Args:
            model_name: Name of the model architecture
            device: Device to run inference on (cpu or cuda)
        """
        self.model_name = model_name
        self.device = device
        self.model = None
        self.version = "1.0.0"

    async def load_model(self):
        """Load pre-trained model."""
        logger.info(f"Loading {self.model_name} model...")

        # Load pre-trained ResNet50
        self.model = models.resnet50(pretrained=True)
        self.model.eval()
        self.model.to(self.device)

        logger.info(f"Model loaded on {self.device}")

    def is_loaded(self) -> bool:
        """Check if model is loaded."""
        return self.model is not None

    async def predict(self, image_tensor: torch.Tensor) -> List[Dict]:
        """
        Make prediction on preprocessed image.

        Args:
            image_tensor: Preprocessed image tensor

        Returns:
            List of predictions with class and confidence
        """
        if self.model is None:
            raise RuntimeError("Model not loaded")

        # Move to device
        image_tensor = image_tensor.to(self.device)

        # Inference
        with torch.no_grad():
            outputs = self.model(image_tensor)
            probabilities = torch.nn.functional.softmax(outputs[0], dim=0)

        # Get top 5 predictions
        top5_prob, top5_catid = torch.topk(probabilities, 5)

        predictions = []
        for i in range(5):
            predictions.append({
                "class_id": int(top5_catid[i]),
                "class": f"class_{top5_catid[i]}",  # Placeholder
                "confidence": float(top5_prob[i])
            })

        return predictions
```

**保存文件。**

---

### 任务 4.4：提交代码变更

为代码制作一个单独的 commit。

**说明：**

```bash
# 检查什么发生了变化
git status

# 暂存文件
git add src/api/app.py src/models/classifier.py

# 提交前审查变更
git diff --staged

# 提交
git commit -m "Add FastAPI app and image classifier

Implemented:
- FastAPI application with predict endpoint
- ImageClassifier class for model inference
- Health check and model info endpoints
- Structured logging throughout
- Error handling for invalid inputs

The API can now accept image uploads and return
classification predictions using ResNet-50."

# 查看日志
git log --oneline
```

**预期输出：**
```text
abc123 Add FastAPI app and image classifier
def456 Initial project structure
```

---

### 任务 4.5：添加更多文件并练习选择性暂存

添加预处理代码，但选择性暂存文件。

**说明：**

编辑 `src/preprocessing/image.py`：

```python
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

    # Convert to RGB if needed
    if image.mode != "RGB":
        image = image.convert("RGB")

    # Apply preprocessing
    transform = get_preprocessing_transform()
    image_tensor = transform(image)

    # Add batch dimension
    image_tensor = image_tensor.unsqueeze(0)

    return image_tensor


def validate_image(image_bytes: bytes) -> bool:
    """
    Validate that bytes represent a valid image.

    Args:
        image_bytes: Raw bytes

    Returns:
        True if valid image, False otherwise
    """
    try:
        Image.open(io.BytesIO(image_bytes))
        return True
    except Exception:
        return False
```

编辑 `src/utils/logging.py`：

```python
"""
Logging Configuration

Structured logging setup for the application.
"""

import structlog
import logging
import sys


def configure_logging(level: str = "INFO", format_type: str = "json"):
    """
    Configure structured logging.

    Args:
        level: Logging level (DEBUG, INFO, WARNING, ERROR)
        format_type: Output format (json or console)
    """
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, level.upper())
    )

    # Configure structlog
    if format_type == "json":
        processors = [
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.JSONRenderer()
        ]
    else:
        processors = [
            structlog.dev.ConsoleRenderer()
        ]

    structlog.configure(
        processors=processors,
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )
```

**现在练习选择性暂存：**

```bash
# 检查状态
git status

# 只暂存预处理文件
git add src/preprocessing/image.py

# 检查状态 - logging.py 应该仍未暂存
git status

# 提交预处理
git commit -m "Add image preprocessing module

Implemented preprocessing pipeline:
- Image loading from bytes
- Resize and center crop to 224x224
- ImageNet normalization
- Tensor conversion
- Image validation

Preprocessing follows standard ImageNet transform pipeline
for compatibility with pre-trained models."

# 现在暂存并提交 logging
git add src/utils/logging.py

git commit -m "Add structured logging configuration

Implemented logging setup with:
- Structured logging using structlog
- JSON and console output formats
- Configurable log levels
- Timestamp inclusion

This provides consistent, parseable logs for production
monitoring and debugging."

# 查看日志
git log --oneline
```

**预期输出：**
```text
ghi789 Add structured logging configuration
abc123 Add image preprocessing module
def456 Add FastAPI app and image classifier
jkl012 Initial project structure
```

---

## 第五部分：使用 Git 历史

### 任务 5.1：探索你的 Commit

学习浏览你的 commit 历史。

**说明：**

```bash
# 查看 commit 历史（完整）
git log

# 查看 commit 历史（每行一个 commit）
git log --oneline

# 查看最后一个 commit 的详情
git show

# 查看特定 commit
git show def456  # 使用日志中的实际哈希

# 只查看 commit 消息
git log --oneline -1

# 查看带有统计信息的 commit
git log --stat

# 查看带有实际变更的 commit
git log -p

# 查看最近 2 个 commit
git log -2
```

**检查点问题：**
每个 commit 包含哪些信息？

---

### 任务 5.2：检查仓库状态

练习在不同阶段使用 `git status`。

**说明：**

```bash
# 当前状态（应为干净）
git status

# 创建一个新文件
echo "# TODO: Add deployment documentation" > docs/deployment.md

# 检查状态 - 未跟踪文件
git status

# 添加文件
git add docs/deployment.md

# 检查状态 - 已暂存文件
git status

# 修改文件
echo "## Prerequisites" >> docs/deployment.md

# 检查状态 - 已修改的已暂存文件
git status

# 查看差异
git diff
git diff --staged
```

**检查点问题：**
- `git diff` 和 `git diff --staged` 显示的有什么区别？
- 如何取消暂存一个文件？

---

### 任务 5.3：练习取消暂存

学习在改变主意时取消暂存文件。

**说明：**

```bash
# 文件当前已暂存
git status

# 取消暂存
git restore --staged docs/deployment.md

# 检查状态
git status

# 文件应在工作目录中，不在暂存区
```

---

### 任务 5.4：丢弃变更

练习丢弃不需要的变更。

**说明：**

```bash
# 文件有未暂存的变更
git status

# 丢弃变更（小心 - 这是永久性的！）
git restore docs/deployment.md

# 检查状态
git status

# 文件应该已消失（在取消暂存后变为未跟踪）
```

**警告：** `git restore` 会永久丢弃变更。无法撤销！

---

## 第六部分：查看差异

### 任务 6.1：进行变更并比较

练习有效使用 `git diff`。

**说明：**

创建一个测试脚本：

```bash
# 创建新文件
cat > scripts/test_api.sh << 'EOF'
#!/bin/bash
# Test script for ML Inference API

echo "Starting API tests..."

# Test health endpoint
curl http://localhost:8000/health

# Test root endpoint
curl http://localhost:8000/

echo "Tests complete"
EOF

# 使其可执行
chmod +x scripts/test_api.sh

# 添加并提交
git add scripts/test_api.sh
git commit -m "Add API testing script"
```

现在修改它：

```bash
# 修改文件
cat >> scripts/test_api.sh << 'EOF'

# Test models endpoint
curl http://localhost:8000/models
EOF

# 查看变更
git diff

# 带有颜色高亮查看
git diff --color-words

# 暂存变更
git add scripts/test_api.sh

# 查看已暂存的变更
git diff --staged

# 提交
git commit -m "Add models endpoint test to testing script"
```

---

## 验证与测试

### 任务 7.1：验证仓库状态

运行这些命令以验证你的仓库是否正确：

**说明：**

```bash
# 应显示干净的工作树
git status

# 应显示约 6 个 commit
git log --oneline

# 验证 .gitignore 是否生效
mkdir data
echo "test" > data/test.csv
git status
# data/test.csv 应不出现

# 验证目录结构
tree -L 2

# 验证文件存在
ls -la src/api/
ls -la src/models/
ls -la configs/
```

**预期结果：**
- 干净的工作目录
- 历史中有多个 commit
- 正确的目录结构
- `.gitignore` 阻止不需要的文件被跟踪

---

## 挑战任务（可选）

准备好更多挑战了吗？试试以下附加任务：

### 挑战 1：添加测试

创建测试文件并提交：

```bash
# 创建测试文件
cat > tests/unit/test_classifier.py << 'EOF'
import pytest
from src.models.classifier import ImageClassifier

def test_classifier_initialization():
    """Test classifier can be initialized."""
    classifier = ImageClassifier()
    assert classifier.model_name == "resnet50"
    assert classifier.device == "cpu"
    assert not classifier.is_loaded()

def test_classifier_device():
    """Test classifier device setting."""
    classifier = ImageClassifier(device="cuda")
    assert classifier.device == "cuda"
EOF

git add tests/unit/test_classifier.py
git commit -m "Add unit tests for classifier initialization"
```

### 挑战 2：修复一个"Bug"

模拟发现并修复一个 bug：

```bash
# 通过修改文件创建一个"bug"
echo "# FIXME: Handle CUDA out of memory errors" >> src/models/classifier.py

# 暂存并提交
git add src/models/classifier.py
git commit -m "Add FIXME note for CUDA OOM handling"

# 现在"修复"它
# 编辑文件添加适当的错误处理
# 然后提交修复
git add src/models/classifier.py
git commit -m "Fix: Add CUDA OOM error handling

Handle torch.cuda.OutOfMemoryError by automatically
falling back to CPU inference when GPU memory is exhausted.

Fixes #1"
```

### 挑战 3：编写更好的 Commit 消息

练习编写详细的 commit 消息：

```bash
# 进行变更
echo "strict = true" >> setup.cfg

git add setup.cfg
git commit
# 编辑器打开 - 按照讲座中的格式写一条详细消息
```

---

## 反思问题

回答这些问题以巩固你的理解：

1. **为什么 `.gitignore` 对 ML 项目很重要？**
   - 如果你意外提交了一个 5GB 的模型文件会发生什么？

2. **跟踪和未跟踪文件的区别是什么？**
   - Git 对它们有何不同处理？

3. **为什么制作多个小 commit 而不是一个大 commit？**
   - 有什么好处？

4. **"原子性 commit"是什么意思？**
   - 从本练习中举一个例子。

5. **好的 commit 消息应该包含哪些信息？**
   - 回顾讲座 01 中的 commit 消息指南。

6. **为什么在提交前要暂存文件？**
   - 为什么不直接从工作目录提交？

7. **如果你犯了错误，如何撤销最后一个 commit？**
   - （提示：这在讲座 04 中有涉及，但思考一下什么可能有效）

---

## 总结

恭喜！你已经完成了练习 01。你学会了：

- ✅ 如何正确初始化 Git 仓库
- ✅ 为 ML 项目配置 Git
- ✅ 创建正确的目录结构
- ✅ 编写全面的 `.gitignore` 文件
- ✅ 制作原子性、文档良好的 commit
- ✅ 使用 Git 三态工作流
- ✅ 浏览 commit 历史
- ✅ 选择性暂存和取消暂存文件
- ✅ 有效使用 `git status` 和 `git diff`

### 学会的关键命令

```bash
git init                    # 初始化仓库
git config                  # 配置 Git
git status                  # 检查仓库状态
git add                     # 暂存变更
git commit                  # 创建 commit
git log                     # 查看历史
git show                    # 显示 commit 详情
git diff                    # 查看未暂存的变更
git diff --staged           # 查看已暂存的变更
git restore                 # 丢弃或取消暂存变更
```

### 下一步

- 完成练习 02，了解更多关于 commit 和历史的知识
- 如有不清楚的概念，复习讲座 01
- 练习为其他项目创建仓库

**做得很好！** 你已经迈出了 ML 项目版本控制的第一步。
