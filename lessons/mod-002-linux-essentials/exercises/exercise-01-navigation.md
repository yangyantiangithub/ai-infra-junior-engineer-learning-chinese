# 练习 01：Linux 导航和文件系统掌握

## 概述

本练习旨在建立 Linux 文件系统导航、文件操作和目录管理的实用技能。你将创建一个完整的 ML 项目结构，高效地导航，并像专业 AI 基础设施工程师一样管理文件。

## 学习目标

完成本练习后，你将能够：
- 使用 cd、ls、pwd 自信地导航 Linux 文件系统
- 为 ML 项目创建目录结构
- 安全地执行文件操作（复制、移动、删除）
- 使用 find 和 locate 搜索文件
- 创建和管理符号链接
- 应用 ML 项目组织的最佳实践

## 先决条件

- 已完成讲座 01：Linux 和命令行简介
- 已完成讲座 02：文件系统和导航
- 可访问 Linux 系统（VM、WSL 或云实例）
- 终端模拟器
- 至少 1GB 可用磁盘空间

## 所需时间

- 预计：60-90 分钟
- 难度：初级

## 第 1 部分：设置和环境探索

### 步骤 1：验证你的环境

```bash
# 检查当前位置
pwd

# 列出主目录内容
ls -la ~

# 检查可用磁盘空间
df -h ~

# 验证你有必要的权限
touch /tmp/test_file && rm /tmp/test_file && echo "Permissions OK" || echo "Permission issue"
```

**预期输出**：
- 当前工作目录路径
- 主目录中的文件列表
- 可用磁盘空间
- "Permissions OK" 消息

**验证**：
- [ ] pwd 显示当前目录
- [ ] ls 显示文件和隐藏文件（以 . 开头）
- [ ] df 显示至少 1GB 可用空间
- [ ] 权限测试成功

### 步骤 2：探索文件系统

```bash
# 查看根目录结构
ls -l /

# 探索常用目录
ls -l /etc       # 配置文件
ls -l /var/log   # 日志文件
ls -l /usr/bin   # 用户二进制文件
ls -l /opt       # 可选软件

# 在层级中找到你的主目录
cd ~
pwd
echo $HOME
```

**要回答的问题**：
1. 你的主目录的完整路径是什么？
2. 根（/）目录中有多少个目录？
3. /etc 中的文件由哪个用户拥有？
4. /var/log 的总大小是多少？

## 第 2 部分：创建 ML 项目结构

### 步骤 3：构建标准 ML 项目布局

创建一个专业的 ML 项目结构：

```bash
# 导航到主目录
cd ~

# 创建项目根目录
mkdir -p projects/ml-image-classifier

# 进入项目
cd projects/ml-image-classifier

# 创建完整目录结构
mkdir -p data/{raw,processed,external}
mkdir -p models/{checkpoints,final,experiments}
mkdir -p notebooks
mkdir -p src/{data,models,training,evaluation,utils}
mkdir -p tests
mkdir -p logs
mkdir -p configs
mkdir -p scripts
mkdir -p docs

# 验证结构
tree -L 2
# 或如果 tree 未安装：
find . -type d | sort
```

**预期结构**：
```
ml-image-classifier/
├── configs/
├── data/
│   ├── external/
│   ├── processed/
│   └── raw/
├── docs/
├── logs/
├── models/
│   ├── checkpoints/
│   ├── experiments/
│   └── final/
├── notebooks/
├── scripts/
├── src/
│   ├── data/
│   ├── evaluation/
│   ├── models/
│   ├── training/
│   └── utils/
└── tests/
```

### 步骤 4：创建项目文件

```bash
# 创建 README 文件
echo "# ML Image Classifier Project" > README.md
echo "Raw dataset storage - immutable" > data/raw/README.md
echo "Processed datasets ready for training" > data/processed/README.md
echo "Training checkpoints - autosaved during training" > models/checkpoints/README.md

# 创建 Python 包标记
touch src/__init__.py
touch src/data/__init__.py
touch src/models/__init__.py
touch src/training/__init__.py
touch src/evaluation/__init__.py
touch src/utils/__init__.py

# 创建配置文件
cat > configs/default.yaml << 'EOF'
# Default configuration
model:
  architecture: resnet50
  input_shape: [224, 224, 3]
  num_classes: 10

training:
  batch_size: 32
  epochs: 100
  learning_rate: 0.001

data:
  train_path: data/processed/train
  val_path: data/processed/val
  test_path: data/processed/test
EOF

# 创建 .gitignore
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
*.so

# Virtual environments
venv/
env/
ENV/

# Jupyter
.ipynb_checkpoints/

# Data and models
data/raw/*
data/processed/*
!data/raw/README.md
!data/processed/README.md
models/checkpoints/*
models/experiments/*
!models/checkpoints/.gitkeep
logs/*.log

# IDE
.vscode/
.idea/
*.swp
EOF

# 创建占位文件
touch data/raw/.gitkeep
touch data/processed/.gitkeep
touch models/checkpoints/.gitkeep
touch logs/.gitkeep

# 创建 requirements.txt
cat > requirements.txt << 'EOF'
numpy==1.24.3
pandas==2.0.3
tensorflow==2.13.0
scikit-learn==1.3.0
matplotlib==3.7.2
pyyaml==6.0.1
tqdm==4.66.1
EOF
```

## 第 3 部分：导航练习

### 步骤 5：高效导航

练习在你的项目结构中高效移动。

```bash
# 返回项目根目录
cd ~/projects/ml-image-classifier

# 使用绝对路径导航到特定目录
cd /home/$USER/projects/ml-image-classifier/src/training
pwd

# 使用相对路径导航
cd ../../data/raw
pwd

# 使用快捷方式
cd ~                    # 主目录
cd -                    # 上一个目录（切换）
cd ../processed         # 父目录，然后同级别
cd ../../..             # 上 3 级
```

## 第 4 部分：文件操作

### 步骤 6：复制和移动文件

```bash
# 返回项目根目录
cd ~/projects/ml-image-classifier

# 创建示例训练脚本
cat > src/training/train.py << 'EOF'
#!/usr/bin/env python3
"""Training script for ML model"""

import yaml
import numpy as np

def main():
    print("Training started...")
    # Training code here
    print("Training completed!")

if __name__ == "__main__":
    main()
EOF

# 使其可执行
chmod +x src/training/train.py

# 复制配置用于实验
cp configs/default.yaml configs/experiment_001.yaml
cp configs/experiment_001.yaml configs/experiment_002.yaml
sed -i 's/resnet50/vgg16/' configs/experiment_001.yaml
sed -i 's/epochs: 100/epochs: 50/' configs/experiment_001.yaml
```

### 步骤 7：安全删除

```bash
# 创建用于练习的临时文件
mkdir -p /tmp/ml-practice
cd /tmp/ml-practice

# 创建测试文件
touch file1.txt file2.txt file3.log
mkdir test_dir
touch test_dir/nested.txt

# 带确认的安全删除
rm -i file1.txt
# 提示：remove regular empty file 'file1.txt'? y

# 按模式删除文件
rm *.log

# 删除目录及其内容
rm -r test_dir
```

## 第 5 部分：查找文件

### 步骤 8：使用 find

```bash
cd ~/projects/ml-image-classifier

# 查找所有 Python 文件
find . -name "*.py"

# 查找所有 YAML 文件
find . -name "*.yaml"

# 查找空目录
find . -type d -empty

# 查找最近一天内修改的文件
find . -type f -mtime -1

# 查找大文件（>1MB）
find . -type f -size +1M

# 查找并删除 __pycache__ 目录
find . -type d -name "__pycache__" -exec rm -rf {} +
```

### 步骤 9：使用链接

```bash
cd ~/projects/ml-image-classifier

# 创建指向当前最佳模型的符号链接
ln -s models/final/resnet50_best.h5 current_model.h5

# 创建指向数据目录的符号链接以便快速访问
ln -s data/processed ~/ml-data

# 验证链接
ls -l current_model.h5
ls -l ~/ml-data

# 列出项目中所有符号链接
find . -type l -ls
```

## 第 6 部分：高级挑战

### 挑战 1：项目统计

```bash
cat > scripts/project_stats.sh << 'EOF'
#!/bin/bash

echo "=== ML Project Statistics ==="
echo ""
echo "Total directories: $(find . -type d | wc -l)"
echo "Total files: $(find . -type f | wc -l)"
echo "Python files: $(find . -name "*.py" | wc -l)"
echo "Config files: $(find . -name "*.yaml" -o -name "*.yml" | wc -l)"
echo "Markdown files: $(find . -name "*.md" | wc -l)"
echo ""
echo "Lines of Python code: $(find . -name "*.py" -exec cat {} \; | wc -l)"
echo ""
echo "Disk usage: $(du -sh . | cut -f1)"
EOF

chmod +x scripts/project_stats.sh
./scripts/project_stats.sh
```

### 挑战 2：备份脚本

```bash
cat > scripts/backup_project.sh << 'EOF'
#!/bin/bash

PROJECT_ROOT="$HOME/projects/ml-image-classifier"
BACKUP_DIR="$HOME/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="ml-classifier_${DATE}.tar.gz"

mkdir -p "$BACKUP_DIR"

echo "Creating backup: $BACKUP_NAME"

tar -czf "$BACKUP_DIR/$BACKUP_NAME" \
    --exclude='models/checkpoints/*' \
    --exclude='models/experiments/*' \
    --exclude='data/raw/*' \
    --exclude='data/processed/*' \
    --exclude='__pycache__' \
    --exclude='.ipynb_checkpoints' \
    -C "$HOME/projects" \
    ml-image-classifier

echo "Backup created: $BACKUP_DIR/$BACKUP_NAME"
ls -lh "$BACKUP_DIR/$BACKUP_NAME"
EOF

chmod +x scripts/backup_project.sh
./scripts/backup_project.sh
```

### 挑战 3：清理脚本

```bash
cat > scripts/cleanup.sh << 'EOF'
#!/bin/bash

echo "Cleaning up project..."

# Remove Python cache
find . -type d -name "__pycache__" -exec rm -rf {} + 2>/dev/null
find . -type f -name "*.pyc" -delete
find . -type f -name "*.pyo" -delete

# Remove Jupyter checkpoints
find . -type d -name ".ipynb_checkpoints" -exec rm -rf {} + 2>/dev/null

# Remove old logs (older than 30 days)
find logs/ -name "*.log" -mtime +30 -delete 2>/dev/null

# Remove temporary files
find . -name "*.tmp" -delete
find . -name "*.bak" -delete

# Remove empty directories
find . -type d -empty -delete

echo "Cleanup complete!"
EOF

chmod +x scripts/cleanup.sh
./scripts/cleanup.sh
```

## 最终验证

```bash
cd ~/projects/ml-image-classifier

# 1. 检查目录结构
echo "=== Directory Structure ==="
tree -L 2 || find . -type d | sort

# 2. 检查文件统计
echo "=== File Statistics ==="
echo "Total files: $(find . -type f | wc -l)"
echo "Python files: $(find . -name "*.py" | wc -l)"
echo "Config files: $(find . -name "*.yaml" | wc -l)"

# 3. 检查脚本是否可执行
echo "=== Executable Scripts ==="
ls -l scripts/*.sh

# 4. 检查符号链接
echo "=== Symbolic Links ==="
find . -type l -ls

# 5. 验证备份
echo "=== Backups ==="
ls -lh ~/backups/

# 6. 测试导航
echo "=== Navigation Test ==="
cd src/training && echo "✓ Can access src/training"
cd ../../data/raw && echo "✓ Can access data/raw"
cd ~/projects/ml-image-classifier && echo "✓ Returned to root"
```

**预期结果**：
- [ ] 创建了 18+ 个子目录
- [ ] 创建了 15+ 个文件
- [ ] 3+ 个可执行脚本
- [ ] 创建了 2+ 个符号链接
- [ ] 创建了 1+ 个备份
- [ ] 所有导航测试通过

---

**恭喜！** 你已经掌握了 ML 项目的 Linux 导航和文件管理。这些技能是你所有基础设施工作的基础。
