# 练习 06：用于 ML 工作流的 Git - DVC 与模型版本控制

## 概述

**难度**：中级到高级
**预计时间**：90-120 分钟
**前置条件**：练习 05 - 协作，讲座 04 - 用于 ML 的高级 Git

在本练习中，你将学习 ML 特定的 Git 工作流，包括使用 DVC（Data Version Control）进行数据版本控制、使用 Git LFS 管理大型模型文件、实验跟踪以及在 ML 项目中维护可重现性。

## 学习目标

完成本练习后，你将：

- 设置并使用 DVC 进行数据版本控制
- 使用 Git LFS 跟踪大型模型文件
- 版本控制 ML 实验
- 使用 Git 维护可重现性
- 管理模型制品和数据集
- 创建可重现的 ML 流水线
- 跟踪模型溯源
- 实现 ML 特定的 Git hooks

## 场景

你正在构建一个需要跟踪以下内容的生产级 ML 系统：
- 训练数据集（多个版本）
- 模型权重和检查点
- 实验配置
- 训练脚本和超参数
- 模型评估指标

你需要对代码和数据进行版本控制，确保可重现性和可追溯性。

## 前置条件

```bash
# 安装 DVC
pip install dvc dvc-s3  # 或者 dvc-gs 用于 Google Cloud，dvc-azure 用于 Azure

# 安装 Git LFS
# Mac：
brew install git-lfs

# Linux：
# sudo apt-get install git-lfs

# Windows：
# 从 https://git-lfs.github.com/ 下载

# 初始化 Git LFS
git lfs install

# 验证安装
dvc version
git lfs version

# 创建项目
mkdir ml-classification-project
cd ml-classification-project
git init
git config init.defaultBranch main
```

---

## 第一部分：设置 DVC 进行数据版本控制

### 任务 1.1：初始化 DVC

设置 DVC 以跟踪数据。

**说明：**

```bash
# 在 Git 仓库中初始化 DVC
dvc init

# 这会创建 .dvc 目录和 .dvcignore
ls -la

# 提交 DVC 初始化
git status
git add .dvc .dvcignore
git commit -m "init: initialize DVC for data versioning"

# 配置 DVC 远程存储（目前使用本地）
mkdir -p /tmp/dvc-storage
dvc remote add -d local /tmp/dvc-storage

# 提交远程配置
git add .dvc/config
git commit -m "config: add DVC remote storage"

# 查看 DVC 配置
cat .dvc/config
```

**输出：**
```
['remote "local"']
    url = /tmp/dvc-storage
[core]
    remote = local
```

---

### 任务 1.2：使用 DVC 跟踪数据集

版本控制你的训练数据。

**说明：**

```bash
# 创建示例数据集
mkdir -p data/raw

# 生成示例训练数据
cat > data/raw/train.csv << 'EOF'
id,feature1,feature2,label
1,0.5,0.3,0
2,0.8,0.6,1
3,0.2,0.4,0
4,0.9,0.7,1
5,0.3,0.2,0
EOF

cat > data/raw/test.csv << 'EOF'
id,feature1,feature2,label
101,0.6,0.4,1
102,0.7,0.5,1
103,0.3,0.3,0
EOF

# 将数据集加入 DVC 跟踪
dvc add data/raw/train.csv
dvc add data/raw/test.csv

# 这会创建 .dvc 文件
ls data/raw/

# 输出：train.csv  train.csv.dvc  test.csv  test.csv.dvc

# .dvc 文件包含关于数据的元数据
cat data/raw/train.csv.dvc
```

**输出：**
```yaml
outs:
- md5: abc123def456...
  size: 145
  path: train.csv
```

```bash
# 将 .dvc 文件添加到 Git（而不是实际数据！）
git add data/raw/*.dvc data/.gitignore
git commit -m "data: add initial training and test datasets

Datasets:
- train.csv: 5 samples
- test.csv: 3 samples

Tracked with DVC for version control."

# 将数据推送到 DVC 远程
dvc push

# 实际的 CSV 文件在 DVC 存储中，而不是 Git！
```

---

### 任务 1.3：版本控制数据集变更

跟踪数据集的演进。

**说明：**

```bash
# 更新数据集（模拟数据收集）
cat >> data/raw/train.csv << 'EOF'
6,0.4,0.5,1
7,0.6,0.3,1
8,0.1,0.2,0
EOF

# 更新 DVC 跟踪
dvc add data/raw/train.csv

# .dvc 文件发生了变化（新的 MD5 哈希）
git diff data/raw/train.csv.dvc

# 提交变更
git add data/raw/train.csv.dvc
git commit -m "data: expand training set to 8 samples

Added 3 more training examples for better model coverage."

# 推送新版本到 DVC 存储
dvc push

# 现在你有版本化的数据集了！
git log --oneline -- data/raw/train.csv.dvc
```

---

### 任务 1.4：检索旧版本数据集

时光穿梭你的数据。

**说明：**

```bash
# 检出旧版本的 dataset
git checkout HEAD~1 data/raw/train.csv.dvc

# 拉取旧数据
dvc checkout data/raw/train.csv.dvc

# 查看数据（应该又是 5 行）
wc -l data/raw/train.csv

# 返回到最新版本
git checkout main data/raw/train.csv.dvc
dvc checkout
```

---

## 第二部分：用于大型模型文件的 Git LFS

### 任务 2.1：设置 Git LFS

为模型文件配置 LFS。

**说明：**

```bash
# 使用 LFS 跟踪模型文件
git lfs track "*.h5"
git lfs track "*.pkl"
git lfs track "*.pth"
git lfs track "*.onnx"
git lfs track "*.pb"

# 这会创建 .gitattributes
cat .gitattributes

# 提交 LFS 配置
git add .gitattributes
git commit -m "config: track model files with Git LFS

Configure LFS for:
- PyTorch (.pth)
- Keras (.h5)
- Pickle (.pkl)
- ONNX (.onnx)
- TensorFlow (.pb)"
```

---

### 任务 2.2：版本控制模型检查点

使用 LFS 跟踪模型权重。

**说明：**

```bash
# 创建模型目录
mkdir -p models/resnet50

# 模拟模型训练和保存
cat > models/resnet50/model_v1.0.0.pth << 'EOF'
# This would be actual PyTorch weights
# Simulating with dummy file for exercise
model_checkpoint_v1.0.0
EOF

# 创建模型元数据
cat > models/resnet50/model_v1.0.0.json << 'EOF'
{
  "model_name": "resnet50",
  "version": "1.0.0",
  "architecture": "ResNet-50",
  "framework": "pytorch",
  "created_at": "2024-01-15T10:00:00Z",
  "training_config": {
    "dataset": "imagenet",
    "epochs": 90,
    "batch_size": 256,
    "learning_rate": 0.1,
    "optimizer": "sgd"
  },
  "metrics": {
    "train_accuracy": 0.876,
    "val_accuracy": 0.852,
    "test_accuracy": 0.848
  },
  "data_version": "imagenet_v2023.1",
  "git_commit": "abc123def456"
}
EOF

# 添加到 Git（LFS 会处理 .pth 文件）
git add models/
git commit -m "model: add ResNet-50 v1.0.0

Initial model checkpoint:
- Architecture: ResNet-50
- Test accuracy: 84.8%
- Trained on ImageNet v2023.1

Model weights tracked with Git LFS."

# LFS 自动将大文件推送到 LFS 存储
git push  # (if you have remote configured)
```

---

### 任务 2.3：标记模型发布

为模型创建标记发布。

**说明：**

```bash
# 标记模型版本
git tag -a model-v1.0.0 -m "Model Release v1.0.0

ResNet-50 image classifier
- Test accuracy: 84.8%
- Production-ready

Training details:
- Dataset: ImageNet v2023.1
- 90 epochs, batch size 256
- SGD optimizer, LR 0.1

Deployment:
- Compatible with PyTorch 2.0+
- Input: 224x224 RGB images
- Output: 1000-class predictions"

# 查看标签
git show model-v1.0.0

# 列出所有模型标签
git tag -l "model-*"
```

---

## 第三部分：使用 Git 进行实验跟踪

### 任务 3.1：版本控制实验配置

跟踪 ML 实验。

**说明：**

```bash
# 创建实验目录
mkdir -p experiments

# 实验 1：基线
cat > experiments/exp-001-baseline.yaml << 'EOF'
experiment:
  id: "exp-001"
  name: "baseline-resnet50"
  description: "Baseline ResNet-50 with default hyperparameters"
  created_at: "2024-01-15T10:00:00Z"

model:
  architecture: "resnet50"
  pretrained: false

data:
  dataset: "imagenet"
  version: "2023.1"
  train_split: 0.8
  val_split: 0.1
  test_split: 0.1

training:
  epochs: 90
  batch_size: 256
  learning_rate: 0.1
  optimizer: "sgd"
  momentum: 0.9
  weight_decay: 0.0001
  lr_scheduler: "step"
  lr_decay_epochs: [30, 60, 80]
  lr_decay_rate: 0.1

augmentation:
  random_crop: true
  random_flip: true
  normalize: true

hardware:
  gpus: 4
  distributed: true

results:
  train_accuracy: 0.876
  val_accuracy: 0.852
  test_accuracy: 0.848
  train_loss: 0.342
  val_loss: 0.389
  training_time_hours: 48.5
EOF

git add experiments/exp-001-baseline.yaml
git commit -m "experiment: add baseline ResNet-50 experiment

Experiment ID: exp-001
Test accuracy: 84.8%
Training time: 48.5 hours on 4 GPUs"

# 实验 2：增加学习率
cat > experiments/exp-002-higher-lr.yaml << 'EOF'
experiment:
  id: "exp-002"
  name: "resnet50-higher-lr"
  description: "ResNet-50 with higher initial learning rate"
  parent_experiment: "exp-001"
  created_at: "2024-01-16T09:00:00Z"

model:
  architecture: "resnet50"
  pretrained: false

data:
  dataset: "imagenet"
  version: "2023.1"
  train_split: 0.8
  val_split: 0.1
  test_split: 0.1

training:
  epochs: 90
  batch_size: 256
  learning_rate: 0.3  # CHANGED: increased from 0.1
  optimizer: "sgd"
  momentum: 0.9
  weight_decay: 0.0001
  lr_scheduler: "step"
  lr_decay_epochs: [30, 60, 80]
  lr_decay_rate: 0.1

augmentation:
  random_crop: true
  random_flip: true
  normalize: true

hardware:
  gpus: 4
  distributed: true

results:
  train_accuracy: 0.891
  val_accuracy: 0.867
  test_accuracy: 0.862
  train_loss: 0.298
  val_loss: 0.356
  training_time_hours: 47.2
EOF

git add experiments/exp-002-higher-lr.yaml
git commit -m "experiment: higher learning rate improves accuracy

Experiment ID: exp-002
Test accuracy: 86.2% (+1.4% vs baseline)
Training time: 47.2 hours

Changed:
- Learning rate: 0.1 → 0.3

Result: Improved accuracy with faster convergence."

# 查看实验历史
git log --oneline -- experiments/
```

---

### 任务 3.2：每个实验一个分支

使用分支进行实验性工作。

**说明：**

```bash
# 创建实验分支
git switch -c experiment/data-augmentation

# 添加新的增强策略
cat > experiments/exp-003-augmentation.yaml << 'EOF'
experiment:
  id: "exp-003"
  name: "resnet50-advanced-augmentation"
  description: "ResNet-50 with advanced augmentation techniques"
  parent_experiment: "exp-002"
  created_at: "2024-01-17T08:00:00Z"

model:
  architecture: "resnet50"
  pretrained: false

data:
  dataset: "imagenet"
  version: "2023.1"
  train_split: 0.8
  val_split: 0.1
  test_split: 0.1

training:
  epochs: 90
  batch_size: 256
  learning_rate: 0.3
  optimizer: "sgd"
  momentum: 0.9
  weight_decay: 0.0001
  lr_scheduler: "step"
  lr_decay_epochs: [30, 60, 80]
  lr_decay_rate: 0.1

augmentation:
  random_crop: true
  random_flip: true
  normalize: true
  # NEW augmentations:
  random_rotation: true
  rotation_degrees: 15
  color_jitter: true
  jitter_brightness: 0.2
  jitter_contrast: 0.2
  cutout: true
  cutout_size: 16

hardware:
  gpus: 4
  distributed: true

results:
  train_accuracy: 0.903
  val_accuracy: 0.881
  test_accuracy: 0.876
  train_loss: 0.267
  val_loss: 0.324
  training_time_hours: 52.3
EOF

git add experiments/exp-003-augmentation.yaml
git commit -m "experiment: advanced augmentation improves accuracy

Experiment ID: exp-003
Test accuracy: 87.6% (+1.4% vs exp-002)

Added augmentations:
- Random rotation (±15°)
- Color jitter
- Cutout regularization

Trade-off: +5 hours training time but better accuracy."

# 如果实验成功，合并到 main
git switch main
git merge experiment/data-augmentation --no-edit

# 标记成功的实验
git tag -a exp-003-success -m "Successful experiment: advanced augmentation"
```

---

## 第四部分：可重现性

### 任务 4.1：锁定依赖

确保可重现的环境。

**说明：**

```bash
# 创建全面的依赖文件
cat > requirements.txt << 'EOF'
# Core ML Framework
torch==2.1.0
torchvision==0.16.0

# Data Processing
numpy==1.26.0
pandas==2.1.1
pillow==10.1.0

# Experiment Tracking
mlflow==2.8.0
wandb==0.15.12

# Version Control
dvc==3.30.0
dvc-s3==3.0.0

# Utilities
pyyaml==6.0.1
python-dotenv==1.0.0
tqdm==4.66.1
EOF

git add requirements.txt requirements.lock
git commit -m "deps: lock dependency versions for reproducibility

Added requirements.lock with exact versions to ensure
identical environments across training runs."

# 为 conda 用户创建 environment.yaml
cat > environment.yaml << 'EOF'
name: ml-classification
channels:
  - pytorch
  - conda-forge
  - defaults
dependencies:
  - python=3.11
  - pytorch=2.1.0
  - torchvision=0.16.0
  - numpy=1.26.0
  - pandas=2.1.1
  - pillow=10.1.0
  - pip
  - pip:
    - mlflow==2.8.0
    - wandb==0.15.12
    - dvc==3.30.0
EOF

git add environment.yaml
git commit -m "deps: add conda environment specification"
```

---

### 任务 4.2：记录训练过程

创建可重现的训练文档。

**说明：**

```bash
# 创建训练 README
cat > experiments/README.md << 'EOF'
# Experiment Tracking

## Reproducing Experiments

### Prerequisites
```bash
# Install dependencies
pip install -r requirements.lock

# Setup DVC
dvc pull
```

### Running Experiments

#### Experiment 001: Baseline
```bash
# Checkout experiment commit
git checkout <commit-hash>

# Get data
dvc pull

# Run training
python train.py --config experiments/exp-001-baseline.yaml

# Expected results:
# - Test accuracy: ~84.8%
# - Training time: ~48.5 hours on 4x V100
```

#### Experiment 002: Higher Learning Rate
```bash
git checkout <commit-hash>
dvc pull
python train.py --config experiments/exp-002-higher-lr.yaml

# Expected results:
# - Test accuracy: ~86.2%
```

## Experiment Naming Convention
```
exp-<number>-<description>.yaml

Examples:
- exp-001-baseline.yaml
- exp-002-higher-lr.yaml
- exp-003-augmentation.yaml
```

## Experiment Fields
Each experiment YAML must contain:
- `experiment.id`: Unique identifier
- `experiment.name`: Human-readable name
- `experiment.description`: What is being tested
- `model.*`: Model architecture and config
- `data.*`: Dataset version and splits
- `training.*`: Hyperparameters
- `results.*`: Metrics achieved

## Data Versioning
All experiments reference specific data versions:
```yaml
data:
  dataset: "imagenet"
  version: "2023.1"  # Tracked with DVC
```

To get specific data version:
```bash
git checkout <commit> data/
dvc checkout
```

## Model Versioning
Trained models are tagged:
```bash
git tag -l "model-*"
```

Get specific model:
```bash
git checkout model-v1.0.0
git lfs pull
```
EOF

git add experiments/README.md
git commit -m "docs: add experiment reproduction guide"
```

---

## 第五部分：ML 特定的 Git Hooks

### 任务 5.1：Pre-Commit 验证

预防常见的 ML 错误。

**说明：**

```bash
# 安装 pre-commit 框架
pip install pre-commit

# 创建 pre-commit 配置
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files
        args: ['--maxkb=1000']  # Prevent large files in Git
      - id: check-json
      - id: check-merge-conflict
      - id: detect-private-key

  - repo: https://github.com/psf/black
    rev: 23.11.0
    hooks:
      - id: black
        language_version: python3.11

  - repo: https://github.com/PyCQA/flake8
    rev: 6.1.0
    hooks:
      - id: flake8
        args: ['--max-line-length=88']

  # ML-specific checks
  - repo: local
    hooks:
      - id: check-model-metadata
        name: Check model metadata
        entry: python scripts/check_model_metadata.py
        language: python
        files: models/.*\.json$

      - id: validate-experiment-config
        name: Validate experiment configs
        entry: python scripts/validate_experiment.py
        language: python
        files: experiments/.*\.yaml$
EOF

# 创建验证脚本
mkdir -p scripts

cat > scripts/check_model_metadata.py << 'EOF'
#!/usr/bin/env python3
"""Validate model metadata files."""

import json
import sys
from pathlib import Path

REQUIRED_FIELDS = [
    "model_name",
    "version",
    "architecture",
    "framework",
    "created_at",
    "training_config",
    "metrics"
]

def validate_metadata(filepath):
    """Validate model metadata JSON."""
    with open(filepath) as f:
        metadata = json.load(f)

    missing = [field for field in REQUIRED_FIELDS if field not in metadata]

    if missing:
        print(f"Error in {filepath}: Missing fields: {missing}")
        return False

    # Check version format
    version = metadata.get("version", "")
    if not version or len(version.split(".")) != 3:
        print(f"Error in {filepath}: Invalid version format. Use X.Y.Z")
        return False

    return True

if __name__ == "__main__":
    files = sys.argv[1:]
    all_valid = all(validate_metadata(f) for f in files)
    sys.exit(0 if all_valid else 1)
EOF

cat > scripts/validate_experiment.py << 'EOF'
#!/usr/bin/env python3
"""Validate experiment configuration files."""

import yaml
import sys

REQUIRED_SECTIONS = ["experiment", "model", "data", "training"]

def validate_experiment(filepath):
    """Validate experiment YAML."""
    with open(filepath) as f:
        config = yaml.safe_load(f)

    missing = [s for s in REQUIRED_SECTIONS if s not in config]

    if missing:
        print(f"Error in {filepath}: Missing sections: {missing}")
        return False

    # Check experiment ID format
    exp_id = config.get("experiment", {}).get("id")
    if not exp_id or not exp_id.startswith("exp-"):
        print(f"Error in {filepath}: Invalid experiment ID format. Use exp-XXX")
        return False

    return True

if __name__ == "__main__":
    files = sys.argv[1:]
    all_valid = all(validate_experiment(f) for f in files)
    sys.exit(0 if all_valid else 1)
EOF

chmod +x scripts/*.py

# 安装 pre-commit hooks
pre-commit install

git add .pre-commit-config.yaml scripts/
git commit -m "ci: add pre-commit hooks for ML workflows

Hooks:
- Code formatting (Black)
- Linting (Flake8)
- YAML/JSON validation
- Model metadata validation
- Experiment config validation
- Prevent large files in Git"

# 测试 pre-commit
# pre-commit run --all-files
```

---

## 第六部分：DVC 流水线

### 任务 6.1：定义 ML 流水线

使用 DVC 创建可重现的 ML 流水线。

**说明：**

```bash
# 创建流水线阶段
dvc stage add -n prepare \
    -d data/raw \
    -o data/processed \
    python scripts/prepare_data.py

dvc stage add -n train \
    -d data/processed \
    -d experiments/exp-001-baseline.yaml \
    -o models/resnet50/model_v1.0.0.pth \
    -M metrics.json \
    python train.py --config experiments/exp-001-baseline.yaml

dvc stage add -n evaluate \
    -d models/resnet50/model_v1.0.0.pth \
    -d data/processed/test \
    -M evaluation.json \
    python evaluate.py

# 这会创建 dvc.yaml
cat dvc.yaml
```

**输出：**
```yaml
stages:
  prepare:
    cmd: python scripts/prepare_data.py
    deps:
      - data/raw
    outs:
      - data/processed

  train:
    cmd: python train.py --config experiments/exp-001-baseline.yaml
    deps:
      - data/processed
      - experiments/exp-001-baseline.yaml
    outs:
      - models/resnet50/model_v1.0.0.pth
    metrics:
      - metrics.json

  evaluate:
    cmd: python evaluate.py
    deps:
      - models/resnet50/model_v1.0.0.pth
      - data/processed/test
    metrics:
      - evaluation.json
```

```bash
# 运行流水线
# dvc repro

git add dvc.yaml dvc.lock
git commit -m "pipeline: define ML training pipeline with DVC

Pipeline stages:
1. prepare: Data preprocessing
2. train: Model training
3. evaluate: Model evaluation

Fully reproducible with 'dvc repro'"
```

---

## 验证

### 验证你的设置

```bash
# 检查 DVC 配置
dvc status
dvc dag  # View pipeline DAG

# 检查 Git LFS
git lfs ls-files

# 检查 pre-commit
pre-commit run --all-files

# 查看实验历史
git log --oneline -- experiments/

# 查看模型标签
git tag -l "model-*"

# 检查数据版本控制
git log --oneline -- data/
```

---

## 挑战任务

### 挑战 1：实现模型注册表

创建带有元数据的模型注册表：

```bash
# models/registry.json
{
  "models": [
    {
      "id": "model-001",
      "name": "resnet50_v1.0.0",
      "git_tag": "model-v1.0.0",
      "accuracy": 0.848,
      "deployed": true
    }
  ]
}
```

### 挑战 2：自动化实验跟踪

创建脚本以自动将实验记录到 MLflow/W&B，并包含 Git commit 信息。

### 挑战 3：数据质量检查

在训练前添加 DVC 流水线阶段进行数据验证。

---

## 故障排除

**问题**："DVC 远程存储已满"
```bash
# 清理旧数据版本
dvc gc -w  # Garbage collect
```

**问题**："Git LFS 配额超限"
```bash
# 修剪旧的 LFS 对象
git lfs prune
```

**问题**："无法重现实验"
```bash
# 确保精确的 commit 和数据版本
git checkout <commit>
dvc checkout
pip install -r requirements.lock
```

---

## 反思问题

1. **为什么使用 DVC 而不是将数据提交到 Git？**
2. **何时使用 Git LFS vs. DVC？**
3. **版本控制实验如何帮助 ML 开发？**
4. **什么使 ML 实验可重现？**
5. **如何跟踪从数据到部署的模型溯源？**

---

## 总结

恭喜！你已经掌握了 ML 特定的 Git 工作流：

- ✅ 使用 DVC 进行数据版本控制
- ✅ 使用 Git LFS 进行模型跟踪
- ✅ 实验版本控制和跟踪
- ✅ 使用锁定依赖实现可重现性
- ✅ 使用 DVC 的 ML 流水线
- ✅ ML 的 Pre-commit hooks
- ✅ 模型和数据溯源

### 学会的关键命令

```bash
dvc init                      # 初始化 DVC
dvc add <file>                # 使用 DVC 跟踪文件
dvc push/pull                 # 同步 DVC 存储
git lfs track <pattern>       # 使用 LFS 跟踪
git tag model-v1.0.0          # 标记模型版本
dvc repro                     # 重现流水线
pre-commit install            # 设置 hooks
```

### 下一步

- 练习 07：高级 Git 技术
- 应用到你的 ML 项目

**出色的工作！** 你现在可以使用适当的版本控制来管理 ML 项目了。
