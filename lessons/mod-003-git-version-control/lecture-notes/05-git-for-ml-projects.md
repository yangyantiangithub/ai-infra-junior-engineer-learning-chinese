# 讲座 05：用于大规模 ML 项目的 Git

## 目录
1. [引言](#引言)
2. [Git Large File Storage (LFS)](#git-large-file-storage-lfs)
3. [管理 ML 制品](#管理-ml-制品)
4. [单仓库策略](#单仓库策略)
5. [ML 团队的 Git 工作流](#ml-团队的-git-工作流)
6. [性能优化](#性能优化)
7. [ML 特定最佳实践](#ml-特定最佳实践)
8. [总结与要点](#总结与要点)

## 引言

机器学习项目为版本控制带来了独特的挑战：
- **大型二进制文件**：模型、数据集、Docker 镜像
- **实验跟踪**：管理数百次训练运行
- **复杂依赖关系**：ML 框架、数据流水线、基础设施代码
- **团队协作**：数据科学家、ML 工程师、基础设施工程师
- **可重现性**：确保实验可以被重新创建

本讲座使用专为 ML 基础设施设计的 Git 策略来应对这些挑战。

### 学习目标

在本讲座结束时，你将：
- 使用 Git LFS 高效管理大型 ML 制品
- 组织 ML 项目以实现可扩展性和团队协作
- 为 ML 工作流实现有效的分支策略
- 优化大型仓库的 Git 性能
- 应用 ML 特定最佳实践以维护干净、可重现的项目
- 处理 ML 版本控制中的常见陷阱

### 前置条件
- 完成本模块的讲座 01-04
- 理解 ML 工作流（训练、评估、部署）
- 熟悉 ML 制品（模型、数据集、检查点）

**时长**：90 分钟
**难度**：中级到高级

---

## 1. Git Large File Storage (LFS)

### Git 中大文件的问题

Git 是为源代码（文本文件）设计的，而不是大型二进制文件。当你提交一个大文件时：

```
不好：常规 Git 提交 model.pt（500MB）
┌────────────────────────────────────┐
│  .git/ 目录                        │
│  ├── model_v1.pt (500MB)          │
│  ├── model_v2.pt (500MB)          │
│  ├── model_v3.pt (500MB)          │
│  └── model_v4.pt (500MB)          │
│  总计：4 个版本 2GB！               │
└────────────────────────────────────┘

结果：
✗ 克隆缓慢（下载所有版本）
✗ 推送/拉取缓慢
✗ 仓库膨胀
✗ 网络带宽浪费
```

### Git LFS 解决方案

Git LFS（Large File Storage）将大文件存储在 Git 仓库之外：

```
好：Git LFS 用于 model.pt
┌────────────────────────────────────┐
│  .git/ 目录                        │
│  ├── model.pt → 指针 (123 字节)    │
└────────────────────────────────────┘
        ↓
┌────────────────────────────────────┐
│  LFS 存储（单独）                   │
│  └── model_v4.pt (500MB)          │
│      仅当前版本！                   │
└────────────────────────────────────┘

结果：
✓ 快速克隆（仅下载所需版本）
✓ 仓库大小小
✓ 高效带宽使用
```

### 安装 Git LFS

```bash
# Ubuntu/Debian
sudo apt-get install git-lfs

# macOS
brew install git-lfs

# RHEL/CentOS
sudo yum install git-lfs

# 初始化 Git LFS（每个用户一次）
git lfs install
```

### 使用 Git LFS

**1. 用 LFS 跟踪文件类型：**

```bash
# 跟踪所有 PyTorch 模型
git lfs track "*.pt"
git lfs track "*.pth"

# 跟踪 TensorFlow 模型
git lfs track "*.pb"
git lfs track "*.h5"

# 跟踪 ONNX 模型
git lfs track "*.onnx"

# 跟踪模型检查点
git lfs track "checkpoints/**"

# 跟踪数据集
git lfs track "*.parquet"
git lfs track "*.csv.gz"

# 验证跟踪
cat .gitattributes
```

**2. `.gitattributes` 文件：**

运行 `git lfs track` 后，Git 创建/更新 `.gitattributes`：

```
*.pt filter=lfs diff=lfs merge=lfs -text
*.pth filter=lfs diff=lfs merge=lfs -text
*.onnx filter=lfs diff=lfs merge=lfs -text
checkpoints/** filter=lfs diff=lfs merge=lfs -text
```

**始终提交 `.gitattributes`：**

```bash
git add .gitattributes
git commit -m "Configure Git LFS for ML artifacts"
```

**3. 添加和提交 LFS 文件：**

```bash
# 添加模型文件
git add models/bert-base.pt

# 验证它被 LFS 跟踪
git lfs ls-files

# 提交
git commit -m "Add BERT base model"

# 推送（上传到 LFS 存储）
git push origin main
```

### 高级 LFS 操作

**查看 LFS 文件：**

```bash
# 列出所有 LFS 文件
git lfs ls-files

# 显示 LFS 文件信息
git lfs ls-files -l

# 检查 LFS 状态
git lfs status
```

**获取 LFS 文件：**

```bash
# 获取当前 commit 的所有 LFS 文件
git lfs fetch

# 获取所有分支的 LFS 文件
git lfs fetch --all

# 获取特定文件
git lfs fetch origin main models/bert.pt
```

**不下载 LFS 文件的情况下拉取（节省带宽）：**

```bash
# 不下载 LFS 文件的情况下克隆
GIT_LFS_SKIP_SMUDGE=1 git clone <repo>

# 稍后，下载特定的 LFS 文件
git lfs pull --include="models/bert-base.pt"
```

**修剪旧的 LFS 文件：**

```bash
# 移除不需要的旧 LFS 文件
git lfs prune

# 试运行以查看将被移除的内容
git lfs prune --dry-run

# 验证 LFS 对象
git lfs fsck
```

### LFS 最佳实践

1. **不要跟踪所有内容** - 要有选择性
   ```bash
   # ✓ 好：只跟踪最终模型
   git lfs track "models/production/*.onnx"

   # ✗ 不好：跟踪所有中间检查点
   # git lfs track "experiments/**/*.pt"  # 可能 100GB+
   ```

2. **对大型实验数据使用 .gitignore**
   ```bash
   # .gitignore
   experiments/*/checkpoints/  # 不版本控制中间检查点
   data/raw/                   # 不版本控制原始数据（存储在 S3/GCS）
   *.log                       # 不版本控制日志
   wandb/                      # 不版本控制 W&B 制品
   ```

3. **为团队设置 LFS 服务器**
   - 使用 GitHub、GitLab 或 Bitbucket（包含 LFS 存储）
   - 或者为敏感模型自托管 LFS 服务器

4. **监控 LFS 配额**
   ```bash
   # 检查 LFS 存储使用量（GitHub）
   # Settings → Billing → Git LFS data
   ```

---

## 2. 管理 ML 制品

### 应进行版本控制的内容

```
ML 项目结构 - 哪些放在 Git 中？

✓ 版本控制：
├── src/                       # Python 代码
│   ├── training/
│   ├── inference/
│   └── preprocessing/
├── configs/                   # 训练配置（YAML、JSON）
│   ├── bert-base.yaml
│   └── production.yaml
├── models/
│   └── production/            # 仅最终模型（使用 LFS）
│       └── model_v1.0.0.onnx
├── tests/                     # 单元/集成测试
├── docker/                    # Dockerfile
├── kubernetes/                # K8s 清单
├── .github/workflows/         # CI/CD 流水线
├── requirements.txt           # 依赖
├── README.md                  # 文档
└── .gitattributes            # LFS 配置

✗ 不要版本控制：
├── data/raw/                  # 原始数据集（使用 S3/GCS）
├── data/processed/            # 已处理数据（从原始数据重新生成）
├── experiments/               # 实验输出（使用 MLflow/W&B）
│   ├── checkpoints/           # 训练检查点
│   ├── logs/                  # 训练日志
│   └── tensorboard/           # TensorBoard 日志
├── .venv/                     # 虚拟环境
├── __pycache__/               # Python 缓存
└── *.pyc                      # 编译的 Python
```

### ML 项目综合 `.gitignore`

```bash
# .gitignore for ML projects

# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
.venv/
venv/
ENV/
env/
pip-log.txt
pip-delete-this-directory.txt

# Jupyter Notebooks
.ipynb_checkpoints/
*.ipynb

# ML Frameworks
# PyTorch
*.pt
*.pth
*.ckpt
# (unless in models/production/ with LFS)

# TensorFlow
*.pb
*.h5
saved_model/
# (unless in models/production/ with LFS)

# Data
data/raw/
data/processed/
data/interim/
datasets/
*.csv
*.parquet
*.tfrecord
# Exception: small sample data for tests

# Experiment Tracking
experiments/
runs/
outputs/
mlruns/
wandb/
tensorboard/
lightning_logs/

# Model Artifacts
checkpoints/
snapshots/
*.safetensors
# Exception: models/production/ with LFS

# Logs
*.log
logs/
*.out

# IDEs
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# Docker
*.tar
*.tar.gz

# Secrets
.env
.env.local
*.key
*.pem
credentials.json
secrets/
```

### 跟踪实验

**不要将 Git 用于实验跟踪。** 使用专门的工具：

```python
# ✗ 不好：提交每个实验
# 导致数千个 commit，仓库膨胀

# ✓ 好：使用 MLflow
import mlflow

mlflow.start_run()
mlflow.log_param("learning_rate", 0.001)
mlflow.log_param("batch_size", 32)
mlflow.log_metric("accuracy", 0.95)
mlflow.log_artifact("model.pt")
mlflow.end_run()

# 或者 Weights & Biases
import wandb

wandb.init(project="my-ml-project")
wandb.config.update({"learning_rate": 0.001})
wandb.log({"accuracy": 0.95})
wandb.save("model.pt")
```

**Git 用于代码版本，MLflow/W&B 用于实验版本。**

---

## 3. 单仓库策略

### 单仓库 vs 多仓库

**多仓库（Poly-Repo）**
```
ml-training/          # 单独的仓库
ml-inference/
ml-data-pipeline/
ml-infrastructure/
```

**优点**：清晰的所有权，独立的版本控制
**缺点**：依赖地狱，代码重复，难以重构

**单仓库（Monorepo）**
```
ml-platform/
├── training/
├── inference/
├── data-pipeline/
└── infrastructure/
```

**优点**：易于共享代码，原子性变更，简化依赖
**缺点**：仓库更大，需要纪律

### 何时对 ML 使用单仓库

✓ **使用单仓库当：**
- 多个服务共享代码（例如特征工程、工具函数）
- 需要跨服务进行原子性变更
- 团队规模小到中等（<50 名工程师）
- 强大的 CI/CD 来处理复杂性

✗ **避免单仓库当：**
- 团队非常大且独立
- 服务使用不同的语言，没有共享代码
- 需要按服务进行严格的访问控制

### 组织 ML 单仓库

```
ml-platform/
├── libs/                      # 共享库
│   ├── feature-engineering/
│   ├── model-utils/
│   └── monitoring/
├── services/
│   ├── training-service/
│   ├── inference-api/
│   ├── data-ingestion/
│   └── model-registry/
├── models/                    # 模型定义
│   ├── bert-classifier/
│   └── resnet-detector/
├── infrastructure/            # IaC
│   ├── terraform/
│   └── kubernetes/
├── tools/                     # 开发者工具
│   └── scripts/
├── docs/                      # 文档
└── .github/
    └── workflows/             # CI/CD
```

### 单仓库工具

**Bazel** - Google 的构建系统

```python
# BUILD file
py_library(
    name = "feature_engineering",
    srcs = ["feature_eng.py"],
    deps = ["//libs/utils:data_utils"],
)

py_binary(
    name = "train",
    srcs = ["train.py"],
    deps = [
        ":feature_engineering",
        "//libs/model-utils:training",
    ],
)
```

**优点**：
- 仅构建/测试更改的代码
- 缓存构建制品
- 强制依赖图

**Nx（用于 Node.js/Python）**
**Pants（用于 Python）**
**Turborepo（用于 JavaScript）**

---

## 4. ML 团队的 Git 工作流

### Feature Branch 工作流（推荐）

```
main (production)
  │
  ├── feature/new-model-architecture
  │
  ├── feature/data-pipeline-optimization
  │
  └── feature/inference-performance
```

**流程：**

```bash
# 1. 创建功能分支
git checkout -b feature/transformer-model

# 2. 开发
# ... 进行更改 ...

# 3. 定期提交
git add src/models/transformer.py
git commit -m "Add transformer model architecture"

# 4. 推送到远程
git push origin feature/transformer-model

# 5. 创建 Pull Request
# ... 代码审查 ...

# 6. 合并到 main
git checkout main
git merge feature/transformer-model
git push origin main

# 7. 删除功能分支
git branch -d feature/transformer-model
git push origin --delete feature/transformer-model
```

### ML 项目的 Git Flow

```
main (production models)
  │
  ├── develop (integration branch)
  │   │
  │   ├── feature/bert-fine-tuning
  │   ├── feature/data-augmentation
  │   └── feature/model-compression
  │
  ├── release/v1.2.0 (preparing release)
  │
  └── hotfix/inference-bug (emergency fix)
```

**何时使用 Git Flow：**
- 多个版本同时在产
- 严格的发布周期
- 需要在不影响开发的情况下进行热修复

### 主干开发

```
main (always deployable)
  │
  ├── short-lived feature branches (<1 day)
  └── direct commits (for small changes)
```

**最适合：**
- 快节奏的 ML 实验
- 强大的 CI/CD 流水线
- 用于不完整功能的功能开关

---

## 5. 性能优化

### 大型仓库问题

ML 仓库可能会变得很大：
- 大量 commit（每日实验）
- 大文件（即使使用 LFS，历史也会增长）
- 深层历史（多年的开发）

**症状：**
- 慢 `git clone`
- 慢 `git status` / `git log`
- 大型 `.git` 文件夹

### 浅克隆

**仅克隆近期历史：**

```bash
# 仅克隆最近 10 个 commit
git clone --depth 10 <repo-url>

# 仅克隆最后一个 commit
git clone --depth 1 <repo-url>

# 稍后，获取更多历史
git fetch --deepen=100
```

**使用场景：** 不需要完整历史的 CI/CD 流水线

### Sparse Checkout

**仅克隆特定目录：**

```bash
# 克隆仓库
git clone --filter=blob:none --sparse <repo-url>
cd repo

# 仅检出特定路径
git sparse-checkout init --cone
git sparse-checkout set services/inference-api

# 稍后添加更多路径
git sparse-checkout add infrastructure/kubernetes
```

**使用场景：** 大型单仓库，你只在一个服务上工作

### 部分克隆

**不下载所有 blob 的情况下克隆：**

```bash
# 不下载文件内容的情况下克隆
git clone --filter=blob:none <repo-url>

# Git 根据需要下载文件

# 或者排除大文件
git clone --filter=blob:limit=1m <repo-url>  # 跳过 >1MB 的文件
```

### Git 维护

```bash
# 优化仓库
git gc --aggressive --prune=now

# 验证仓库完整性
git fsck

# 显示仓库大小
du -sh .git

# 高效地重新打包对象
git repack -ad

# 修剪不可达对象
git prune
```

---

## 6. ML 特定最佳实践

### 1. 模型的语义化版本控制

```
model_v{MAJOR}.{MINOR}.{PATCH}.onnx

示例：
model_v1.0.0.onnx  # 初始生产模型
model_v1.1.0.onnx  # 改进准确率（向后兼容）
model_v2.0.0.onnx  # 新架构（破坏性变更）
```

**模型版本的 Git 标签：**

```bash
# 标记生产模型
git tag -a model-v1.0.0 -m "BERT classifier - 95% accuracy"
git push origin model-v1.0.0

# 列出模型版本
git tag -l "model-v*"

# 检出特定模型版本
git checkout model-v1.0.0
```

### 2. Commit 消息约定

**格式：**
```
<type>(<scope>): <subject>

<body>

<footer>
```

**类型：**
- `feat`：新功能或模型
- `fix`：Bug 修复
- `perf`：性能改进
- `data`：数据流水线变更
- `model`：模型架构变更
- `exp`：实验（不用于 main 分支）
- `docs`：文档
- `test`：测试
- `ci`：CI/CD 变更

**示例：**

```bash
feat(training): add data augmentation pipeline

- Implement random crop and rotation
- Add color jittering
- Improves validation accuracy by 2%

Closes #123

---

fix(inference): correct batch size handling

TensorRT inference was failing for batch_size=1
due to dimension mismatch.

Fixes #456

---

perf(model): quantize model to int8

- Reduces model size from 500MB to 125MB
- Improves inference latency from 100ms to 25ms
- Accuracy drop: 95.2% → 94.8% (acceptable)
```

### 3. 分支命名约定

```
<type>/<description>

示例：
feature/add-transformer-model
fix/data-loading-bug
experiment/bert-vs-roberta
hotfix/inference-memory-leak
release/v2.0.0
```

### 4. 模型溯源跟踪

**使用 Git 跟踪模型来源：**

```yaml
# models/production/model_v1.0.0.yaml
model:
  name: bert-classifier
  version: 1.0.0
  architecture: BERT-base

training:
  commit: abc123def456  # 训练该模型的 Git commit
  date: 2024-01-15
  dataset: training-data-v2.1
  config: configs/bert-base.yaml

performance:
  accuracy: 0.952
  f1_score: 0.948
  latency_p99: 45ms
```

**将元数据与模型一起提交：**

```bash
git add models/production/model_v1.0.0.onnx
git add models/production/model_v1.0.0.yaml
git commit -m "model: release BERT classifier v1.0.0"
git tag -a model-v1.0.0 -m "BERT classifier production release"
```

### 5. 数据版本控制（DVC）

**Git 处理大型数据集效果不佳。** 使用 **DVC（Data Version Control）**：

```bash
# 安装 DVC
pip install dvc

# 初始化 DVC
dvc init

# 跟踪数据集
dvc add data/train.csv

# 这会创建 data/train.csv.dvc（小文本文件）
git add data/train.csv.dvc .gitignore
git commit -m "data: add training dataset v1"

# 将数据推送到远程存储（S3、GCS 等）
dvc remote add -d storage s3://my-bucket/dvc-storage
dvc push

# 在另一台机器上拉取数据
dvc pull
```

**DVC 与 Git 集成但将数据分开存储。**

### 6. 可重现环境

**固定所有依赖：**

```txt
# requirements.txt
torch==2.0.1
transformers==4.30.0
numpy==1.24.3
scikit-learn==1.3.0

# Generated with: pip freeze > requirements.txt
```

**或者使用 Poetry/Pipenv 进行更好的依赖管理：**

```toml
# pyproject.toml (Poetry)
[tool.poetry.dependencies]
python = "^3.9"
torch = "2.0.1"
transformers = "4.30.0"
```

**提交锁定文件：**

```bash
git add requirements.txt poetry.lock Pipfile.lock
git commit -m "deps: pin dependencies for reproducibility"
```

---

## 7. 总结与要点

### 核心概念

1. **Git LFS** 管理大型 ML 制品
   - 使用 `git lfs track` 跟踪模型、检查点
   - 保持仓库大小小
   - 对于 >50MB 的二进制文件至关重要

2. **选择性版本控制** 减少仓库膨胀
   - 对代码、配置、最终模型进行版本控制
   - 不对原始数据、实验、日志进行版本控制
   - 大量使用 `.gitignore`

3. **单仓库** 可以用于 ML 平台
   - 跨服务轻松共享代码
   - 需要纪律和良好的工具
   - 适用于 <50 名工程师的团队

4. **ML 特定工作流** 改善协作
   - 对实验使用功能分支
   - 用版本标记生产模型
   - 提交模型元数据以进行溯源

5. **性能优化** 保持 Git 快速
   - CI/CD 使用浅克隆
   - 大型单仓库使用 sparse checkout
   - 使用 `git gc` 定期维护

6. **可重现性** 至关重要
   - 固定依赖
   - 将配置和代码一起版本控制
   - 标记生产发布
   - 对数据集使用 DVC

### 获得的实用技能

✅ 配置和使用 Git LFS 处理 ML 制品
✅ 为团队协作构建 ML 项目结构
✅ 为 ML 团队选择合适的 Git 工作流
✅ 优化大型仓库的 Git 性能
✅ 对模型和数据应用版本控制最佳实践
✅ 确保跨团队和时间的可重现性

### 应避免的常见陷阱

❌ 提交大文件而不使用 LFS（膨胀仓库）
❌ 在 Git 中对原始数据进行版本控制（使用 S3/GCS + DVC）
❌ 对虚拟环境进行版本控制（使用 requirements.txt）
❌ 不固定依赖（破坏可重现性）
❌ 创建长寿功能分支（导致合并冲突）
❌ 不标记生产模型版本

### 下一步

- **练习**：为模型训练项目设置 Git LFS
- **尝试**：为你的团队比较单仓库 vs 多仓库
- **实施**：对你的项目应用 commit 约定
- **探索**：尝试使用 DVC 进行数据集版本控制

### 额外资源

- [Git LFS 文档](https://git-lfs.github.com/)
- [DVC - Data Version Control](https://dvc.org/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [MLflow Model Registry](https://www.mlflow.org/docs/latest/model-registry.html)
- [Monorepo 工具比较](https://monorepo.tools/)
- [Git 性能最佳实践](https://github.blog/2021-11-10-git-performance/)

---

**恭喜！** 你现在了解了如何在大规模 ML 项目中有效使用 Git，从使用 LFS 管理大文件到组织复杂的 ML 仓库和工作流。

**下一模块**：模块 004 - ML 基础
