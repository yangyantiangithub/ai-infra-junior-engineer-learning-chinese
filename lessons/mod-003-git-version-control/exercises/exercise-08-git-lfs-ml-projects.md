# 练习 08：Git LFS 与管理大型 ML 项目

## 概述

本动手练习专注于使用 Git LFS（Large File Storage）管理 ML 制品，为 ML 项目实现合适的 .gitignore 模式，将模型与代码一起进行版本控制，以及与 DVC（Data Version Control）集成以实现完整的 ML 可重现性。

## 学习目标

完成本练习后，你将：
- 为 ML 制品安装和配置 Git LFS
- 使用 LFS 跟踪模型、数据集和检查点
- 为 ML 项目创建全面的 .gitignore
- 使用语义化版本控制模型
- 将 Git 与 DVC 集成用于数据版本控制
- 实现模型溯源跟踪
- 高效地克隆和使用 LFS 仓库
- 将现有项目迁移到 Git LFS

## 前置条件

- 完成本模块的练习 01-07
- 讲座 05（用于大规模 ML 项目的 Git）
- 理解 ML 工作流和制品
- Git 基础

## 所需时间

- 预计：90-120 分钟
- 难度：中级

---

## 第一部分：设置 Git LFS

### 任务 1.1：安装 Git LFS

```bash
# Ubuntu/Debian
sudo apt-get install git-lfs

# macOS
brew install git-lfs

# RHEL/CentOS
sudo yum install git-lfs

# 初始化 Git LFS（一次性设置）
git lfs install

# 验证安装
git lfs version
# 输出：git-lfs/3.x.x
```

### 任务 1.2：创建带 LFS 的 ML 项目

```bash
# 创建新项目
mkdir ml-model-repository
cd ml-model-repository
git init
git config init.defaultBranch main

# 配置 LFS 跟踪
git lfs track "*.pt"           # PyTorch 模型
git lfs track "*.pth"          # PyTorch 检查点
git lfs track "*.onnx"         # ONNX 模型
git lfs track "*.h5"           # Keras 模型
git lfs track "*.pb"           # TensorFlow 模型
git lfs track "*.safetensors"  # SafeTensors 格式
git lfs track "models/**"      # models/ 中的所有内容
git lfs track "*.parquet"      # 数据集文件
git lfs track "*.pkl"          # Pickle 文件

# 重要：始终提交 .gitattributes！
git add .gitattributes
git commit -m "Configure Git LFS for ML artifacts"

# 查看 LFS 跟踪
cat .gitattributes
```

---

## 第二部分：版本控制 ML 模型

### 任务 2.1：将模型添加到仓库

```bash
# 创建项目结构
mkdir -p models/production models/experiments
mkdir -p configs data/sample src tests

# 创建示例模型文件（模拟训练好的模型）
dd if=/dev/urandom of=models/production/bert-classifier-v1.0.0.onnx bs=1M count=100
# 创建一个 100MB 的虚拟模型文件

# 创建模型元数据
cat > models/production/bert-classifier-v1.0.0.yaml << 'EOF'
model:
  name: bert-classifier
  version: 1.0.0
  format: onnx
  architecture: BERT-base

training:
  dataset: sentiment-analysis-v2
  framework: pytorch
  framework_version: 2.0.1
  git_commit: abc123def456
  trained_date: 2024-01-15
  training_time_hours: 8.5

hyperparameters:
  learning_rate: 0.00002
  batch_size: 32
  epochs: 10
  max_seq_length: 512

performance:
  accuracy: 0.945
  f1_score: 0.938
  precision: 0.942
  recall: 0.934
  inference_latency_p50_ms: 12
  inference_latency_p99_ms: 45

artifacts:
  model_path: models/production/bert-classifier-v1.0.0.onnx
  config_path: configs/bert-classifier-production.yaml
  training_logs: s3://ml-artifacts/logs/exp_12345/
EOF

# 添加到 Git
git add models/production/bert-classifier-v1.0.0.*

# 提交模型
git commit -m "model: add BERT classifier v1.0.0

- Accuracy: 94.5%
- F1 Score: 93.8%
- Inference latency (p99): 45ms
- Trained on sentiment-analysis-v2 dataset"

# 标记模型版本
git tag -a model-bert-v1.0.0 -m "BERT Classifier v1.0.0 - Production Release"

# 验证 LFS 正在跟踪模型
git lfs ls-files
```

### 任务 2.2：创建全面的 .gitignore

```bash
cat > .gitignore << 'EOF'
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

# Jupyter Notebooks
.ipynb_checkpoints/
*.ipynb  # Optional: version notebooks in separate repo

# ML Frameworks - Training Artifacts (DON'T version these)
checkpoints/
experiments/*/checkpoints/
lightning_logs/
wandb/
mlruns/
runs/
outputs/
snapshots/

# Data - Use DVC or S3 instead
data/raw/
data/processed/
data/interim/
datasets/
*.csv
*.json
*.jsonl
*.tfrecord
# Exception: Keep small sample data for tests
!data/sample/*.csv

# Large model files during development (use LFS for production models only)
*.pt
*.pth
*.ckpt
# Exception: Production models tracked with LFS
!models/production/*.pt
!models/production/*.onnx
!models/production/*.h5

# Logs
*.log
logs/
*.out
*.err

# IDEs
.vscode/
.idea/
*.swp
*.swo
*~
.DS_Store

# Docker
*.tar
*.tar.gz
# Exception: Keep Dockerfiles
!Dockerfile
!docker-compose.yml

# Secrets
.env
.env.local
*.key
*.pem
credentials.json
secrets/
config/secrets.yaml

# OS
Thumbs.db
.DS_Store

# Build
build/
dist/
*.egg-info/

# Temporary files
*.tmp
*.temp
.cache/
EOF

git add .gitignore
git commit -m "chore: add comprehensive .gitignore for ML projects"
```

---

## 第三部分：使用 LFS 仓库

### 任务 3.1：克隆 LFS 仓库

```bash
# 创建远程仓库（模拟 GitHub）
cd ..
git clone --bare ml-model-repository ml-model-repository.git

# 克隆并下载 LFS（下载所有 LFS 文件）
git clone ml-model-repository.git ml-model-clone
cd ml-model-clone

# 验证模型已下载
ls -lh models/production/
# 应显示 100MB 模型文件

# 检查 LFS 状态
git lfs ls-files
```

### 任务 3.2：不下载 LFS 文件的情况下克隆（节省带宽）

```bash
# 不下载 LFS 文件的情况下克隆
cd ..
GIT_LFS_SKIP_SMUDGE=1 git clone ml-model-repository.git ml-model-clone-minimal
cd ml-model-clone-minimal

# LFS 文件是指针（很小）
ls -lh models/production/
# 显示 133 字节（指针文件，不是 100MB）

# 在需要时下载特定的 LFS 文件
git lfs pull --include="models/production/bert-classifier-v1.0.0.onnx"

# 现在文件是 100MB
ls -lh models/production/
```

---

## 第四部分：模型版本控制工作流

### 任务 4.1：发布新模型版本

```bash
cd ../ml-model-repository

# 模拟训练新模型
dd if=/dev/urandom of=models/production/bert-classifier-v1.1.0.onnx bs=1M count=105

# 创建元数据
cat > models/production/bert-classifier-v1.1.0.yaml << 'EOF'
model:
  name: bert-classifier
  version: 1.1.0  # Minor version bump (backward compatible improvement)
  format: onnx
  architecture: BERT-base

training:
  dataset: sentiment-analysis-v2.1  # Updated dataset
  improvements:
    - Added data augmentation
    - Improved preprocessing
  git_commit: def456abc789

performance:
  accuracy: 0.958  # +1.3% improvement
  f1_score: 0.951
  inference_latency_p99_ms: 42  # -3ms improvement
EOF

# 提交新版本
git add models/production/bert-classifier-v1.1.0.*
git commit -m "model: release BERT classifier v1.1.0

Improvements:
- Accuracy: 95.8% (+1.3%)
- F1 Score: 95.1%
- Latency: 42ms (-3ms)

Changes:
- Updated dataset to v2.1
- Added data augmentation
- Improved preprocessing pipeline"

# 标记新版本
git tag -a model-bert-v1.1.0 -m "BERT Classifier v1.1.0 - Accuracy Improvement"

# 推送到远程（包括标签和 LFS）
git push origin main
git push origin --tags
git lfs push origin main
```

### 任务 4.2：模型溯源跟踪

```bash
# 创建溯源跟踪文件
cat > MODELS.md << 'EOF'
# Model Registry

## Production Models

### BERT Classifier

| Version | Release Date | Accuracy | F1 Score | Latency (p99) | Git Tag | Status |
|---------|--------------|----------|----------|---------------|---------|--------|
| 1.1.0 | 2024-01-20 | 95.8% | 95.1% | 42ms | model-bert-v1.1.0 | **Production** |
| 1.0.0 | 2024-01-15 | 94.5% | 93.8% | 45ms | model-bert-v1.0.0 | Deprecated |

### Deployment Instructions

```bash
# Checkout specific model version
git checkout model-bert-v1.1.0

# Download model
git lfs pull --include="models/production/bert-classifier-v1.1.0.onnx"

# Deploy
kubectl apply -f deployment/bert-classifier-v1.1.0.yaml
```

### Rollback Procedure

```bash
# If v1.1.0 has issues, rollback to v1.0.0
git checkout model-bert-v1.0.0
git lfs pull --include="models/production/bert-classifier-v1.0.0.onnx"
```
EOF

git add MODELS.md
git commit -m "docs: add model registry and deployment guide"
```

---

## 第五部分：DVC 集成（Data Version Control）

### 任务 5.1：安装和设置 DVC

```bash
# 安装 DVC
pip install dvc dvc-s3

# 在 Git 仓库中初始化 DVC
dvc init

# 这会创建：
# - .dvc/ 目录
# - .dvc/.gitignore
# - .dvc/config

# 提交 DVC 配置
git add .dvc .dvcignore
git commit -m "chore: initialize DVC for data versioning"
```

### 任务 5.2：使用 DVC 跟踪数据集

```bash
# 创建示例数据集
mkdir -p data/raw
echo "Sample training data" > data/raw/train.csv
for i in {1..1000}; do
    echo "sample_id_$i,feature1,feature2,label" >> data/raw/train.csv
done

# 使用 DVC 跟踪数据集
dvc add data/raw/train.csv

# 这会创建 data/raw/train.csv.dvc（元数据文件）
# 并将 data/raw/train.csv 添加到 .gitignore

# 提交 DVC 元数据（而不是实际数据）
git add data/raw/train.csv.dvc data/raw/.gitignore
git commit -m "data: add training dataset v1 (tracked with DVC)"

# 配置远程存储（本练习使用 S3、GCS 或本地）
dvc remote add -d storage /tmp/dvc-storage
dvc remote modify storage type local

# 将数据推送到 DVC 远程
dvc push
```

### 任务 5.3：版本控制数据更新

```bash
# 更新数据集
echo "New sample data" >> data/raw/train.csv

# 跟踪新版本
dvc add data/raw/train.csv

# 提交元数据
git add data/raw/train.csv.dvc
git commit -m "data: update training dataset to v2"
git tag -a data-v2 -m "Training dataset v2"

# 推送数据
dvc push

# 推送 Git
git push origin main
git push origin --tags
```

---

## 第六部分：LFS 维护与优化

### 任务 6.1：检查 LFS 存储使用量

```bash
# 查看 LFS 文件列表
git lfs ls-files

# 检查 LFS 存储统计
git lfs ls-files --size

# 显示 LFS 带宽使用
git lfs env

# 查找大文件
git lfs ls-files --size | sort -k2 -hr | head -10
```

### 任务 6.2：修剪旧的 LFS 文件

```bash
# 移除当前检出不需要的 LFS 文件
git lfs prune

# 试运行以查看将被移除的内容
git lfs prune --dry-run

# 验证 LFS 对象
git lfs fsck

# 仅获取特定 commit 的 LFS 文件
git lfs fetch --recent  # Last 10 commits
```

---

## 第七部分：将现有项目迁移到 LFS

### 任务 7.1：转换现有仓库

```bash
# 如果你在没有 LFS 的情况下意外提交了大文件

# 1. 安装 BFG Repo-Cleaner
wget https://repo1.maven.org/maven2/com/madgag/bfg/1.14.0/bfg-1.14.0.jar

# 2. 配置 LFS 跟踪
git lfs track "*.onnx"
git lfs track "*.pt"

# 3. 迁移现有大文件
git lfs migrate import --include="*.onnx,*.pt" --everything

# 4. 验证迁移
git lfs ls-files

# 5. 强制推送（警告：会重写历史）
git push --force origin main
```

---

## 交付物

请提交以下内容：

1. **Git 仓库**：
   - 为 LFS 配置了 .gitattributes
   - 全面的 .gitignore
   - 至少 2 个带元数据的模型版本
   - MODELS.md 注册表文件

2. **DVC 设置**：
   - 已初始化 DVC
   - 至少有一个使用 DVC 跟踪的数据集
   - 已配置 DVC 远程

3. **文档**：
   - 解释项目结构的 README
   - 模型部署指南
   - 数据版本控制工作流

4. **证据**：
   - `git lfs ls-files` 输出
   - `git log --oneline --graph` 输出
   - `dvc status` 输出

---

## 挑战问题

1. **存储成本**：如果你的模型是 500MB，每年发布 100 个版本，你需要多少 LFS 存储？如何减少？

2. **带宽**：你的 CI/CD 流水线每天克隆仓库 1000 次。如何最小化 LFS 带宽成本？

3. **单仓库**：你在一个仓库中有 50 个模型。如何让团队只克隆他们模型的文件？

4. **灾难恢复**：如果 GitHub LFS 宕机，你如何确保模型仍然可访问？

5. **合规性**：你如何为审计目的跟踪哪个模型版本部署在哪个环境中？

---

## 额外资源

- [Git LFS 文档](https://git-lfs.github.com/)
- [DVC 文档](https://dvc.org/doc)
- [模型版本控制最佳实践](https://neptune.ai/blog/version-control-for-ml-models)
- [MLflow 模型注册表](https://www.mlflow.org/docs/latest/model-registry.html)
- [语义化版本控制](https://semver.org/)

---

**恭喜！** 你现在可以使用 Git LFS 管理大型 ML 项目，将模型与代码一起进行版本控制，并与 DVC 集成以实现完整的可重现性。
