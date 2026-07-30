# 讲座 01：Git 基础

## 目录
1. [引言](#引言)
2. [理解版本控制](#理解版本控制)
3. [Git 架构与概念](#git-架构与概念)
4. [仓库初始化](#仓库初始化)
5. [Git 的三种状态](#git-的三种状态)
6. [基本 Git 操作](#基本-git-操作)
7. [查看历史](#查看历史)
8. [Git 用于 AI/ML 项目](#git-用于-aiml-项目)
9. [常见问题与故障排除](#常见问题与故障排除)
10. [总结与要点](#总结与要点)

---

## 引言

### 为什么版本控制对 AI 基础设施至关重要

想象一下：你正在开发一个机器学习训练流水线。经过数周的工作，它终于稳定运行实验了。然后你对数据预处理做了一个"小小的优化"。突然，你的模型准确率下降了 15%。恐慌随之而来。发生了什么变化？是哪行代码导致了这个问题？你能撤销吗？

没有版本控制，你只能手动比较文件，或者更糟的是，你已经完全覆盖了能正常工作的版本。

这就是 Git 大显身手的地方——也是拯救你职业生涯的地方。

对 AI 基础设施工程师来说，Git 不仅仅关乎代码。它还关乎：
- **可重现性**：每个实验必须能追溯到精确的代码版本
- **协作**：多个工程师在同一个基础设施上工作
- **安全性**：能够立即回滚有问题的部署
- **文档**：Commit 讲述了为什么做出决策的故事
- **CI/CD**：由 Git 操作触发的自动化测试和部署

### 学习目标

在本讲座结束时，你将：

- 理解 Git 的分布式架构及其与集中式版本控制的区别
- 掌握 Git 的三态模型（工作目录、暂存区、仓库）
- 为 ML 项目初始化仓库和配置 Git
- 执行基本的 Git 操作：add、commit、status、diff、log
- 为基础设施代码编写有效的 commit 消息
- 为 AI/ML 项目创建全面的 .gitignore 文件
- 理解 Git 在可重现 ML 实验中的作用
- 排查常见的 Git 问题

### AI 基础设施背景

AI 基础设施项目有独特的版本控制需求：

**需要跟踪的内容：**
- 基础设施即代码（Terraform、Kubernetes 清单）
- 训练和部署脚本
- 配置文件
- 数据预处理流水线
- 模型服务代码
- 监控和日志配置

**不要跟踪的内容：**
- 大型模型文件（改用 Git LFS 或 DVC）
- 训练数据集（使用数据版本控制工具）
- 虚拟环境
- 编译后的二进制文件
- 临时文件和日志
- API 密钥和密钥

我们将在本讲座中逐一解决这些问题。

---

## 理解版本控制

### 什么是版本控制？

版本控制是一个系统，它记录文件随时间的变化，以便你以后能回忆起特定版本。它回答了关键问题：

- **什么**发生了变化？
- **何时**发生了变化？
- **谁**做了更改？
- **为什么**做了更改？

### 集中式与分布式版本控制

#### 集中式 VCS（SVN、Perforce）

```
        中央服务器
             |
    +--------+--------+
    |        |        |
  开发者A   开发者B   开发者C
```

特点：
- 中央服务器上的单一数据源
- 开发者从服务器检出文件
- 大多数操作需要网络
- 服务器是单点故障

#### 分布式 VCS（Git、Mercurial）

```
    远程仓库（GitHub）
             |
    +--------+--------+
    |        |        |
  开发者A   开发者B   开发者C
  (完整)   (完整)   (完整)
  clone    clone    clone
```

特点：
- 每个开发者拥有完整的仓库历史
- 大多数操作在本地且快速
- 可以离线工作
- 无单点故障
- 更复杂但更强大

### 为什么 Git 赢了

Git 占主导地位有几个原因：

1. **速度**：极快的本地操作
2. **分支**：廉价且容易的分支/合并
3. **分布式**：本地拥有完整的仓库历史
4. **数据完整性**：加密 SHA-1 哈希防止损坏
5. **生态系统**：GitHub、GitLab、Bitbucket 提供协作平台
6. **CI/CD 集成**：与现代 DevOps 工具的原生集成

---

## Git 架构与概念

### Git 对象模型

Git 将所有内容存储为内容可寻址文件系统中的对象。有四种对象类型：

#### 1. Blob（二进制大对象）
存储文件内容。每个唯一的文件内容对应一个 blob。

```
blob 14
Hello, Git!
```

#### 2. Tree
表示一个目录。指向 blob 和其他 tree。

```
tree
100644 blob a906cb  README.md
100644 blob 3b18e5  main.py
040000 tree 8a7d00  src/
```

#### 3. Commit
项目在某个时间点的快照。

```
commit
tree 8a7d00
parent e3a2b1
author John Doe <john@example.com> 1698765432 +0000
committer John Doe <john@example.com> 1698765432 +0000

Add model training pipeline
```

#### 4. Tag
指向特定 commit 的命名引用。

### SHA-1 哈希

每个 Git 对象得到一个唯一的 40 字符 SHA-1 哈希：

```
a906cb2a4a904a152e80877d4088654daad0c859
```

该哈希根据对象的内容计算，使 Git 防篡改。如果单个字节发生变化，哈希将完全改变。

### 仓库结构

当你初始化一个 Git 仓库时，会创建以下结构：

```
my-ml-project/
├── .git/                    # Git 仓库数据（隐藏）
│   ├── objects/            # 所有 Git 对象（blob、tree、commit）
│   ├── refs/               # 引用（分支、标签）
│   │   ├── heads/         # 本地分支
│   │   └── remotes/       # 远程分支
│   ├── HEAD                # 指向当前分支
│   ├── config              # 仓库配置
│   ├── index               # 暂存区
│   └── hooks/              # Git hooks（自动化）
└── [你的项目文件]
```

**重要**：除非你完全知道自己在做什么，否则永远不要手动编辑 `.git/` 中的文件。

---

## 仓库初始化

### 开始一个新仓库

有两种方式开始使用 Git：

#### 选项 1：创建一个新仓库

```bash
# 创建项目目录
mkdir ml-inference-api
cd ml-inference-api

# 初始化 Git 仓库
git init

# 输出：
# Initialized empty Git repository in /path/to/ml-inference-api/.git/
```

这会创建 `.git` 目录并将你的文件夹变成 Git 仓库。

#### 选项 2：克隆已有仓库

```bash
# 从 GitHub 克隆
git clone https://github.com/username/ml-inference-api.git

# 使用不同的目录名克隆
git clone https://github.com/username/ml-inference-api.git my-api

# 克隆特定分支
git clone -b develop https://github.com/username/ml-inference-api.git
```

克隆会下载整个仓库历史，而不仅仅是最新版本。

### 初始 Git 配置

在制作 commit 之前，配置你的身份：

```bash
# 设置姓名和邮箱（全局配置）
git config --global user.name "Jane Smith"
git config --global user.email "jane.smith@company.com"

# 验证配置
git config --list

# 将默认分支名称设置为 'main'（现代约定）
git config --global init.defaultBranch main

# 设置 commit 消息的默认编辑器
git config --global core.editor "vim"  # 或 "nano", "code --wait" 等

# 启用彩色输出
git config --global color.ui auto
```

配置级别：
- `--system`：系统上的所有用户
- `--global`：当前用户（存储在 `~/.gitconfig`）
- `--local`：仅当前仓库（存储在 `.git/config`）

### 针对 AI/ML 项目的配置

对 ML 基础设施有用的额外配置：

```bash
# 处理换行符（对跨平台团队很重要）
git config --global core.autocrlf input  # Linux/Mac
git config --global core.autocrlf true   # Windows

# 为大文件增加 HTTP 缓冲区
git config --global http.postBuffer 524288000  # 500 MB

# 设置凭据缓存（避免重复输入密码）
git config --global credential.helper cache
git config --global credential.helper 'cache --timeout=3600'  # 1 小时

# 为常用命令设置别名
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
```

---

## Git 的三种状态

理解 Git 的三态架构是基础。文件可以处于三种状态之一：

```
工作目录  →  暂存区  →  Git 仓库
 (已修改)     (已暂存)    (已提交)
     ↓           ↓           ↓
  git add    git commit   历史
```

### 1. 工作目录（已修改）

你实际的项目文件。当你编辑一个文件时，它处于"已修改"状态。

```bash
# 编辑文件
echo "import torch" > model.py
```

文件 `model.py` 现在在你的工作目录中已被修改。

### 2. 暂存区 / Index（已暂存）

一个保存区域，用于存放你想包含在下一次 commit 中的变更。可以把它想象成准备打包发货。

```bash
# 暂存文件
git add model.py
```

现在 `model.py` 已暂存并准备好提交。

### 3. Git 仓库（已提交）

永久存储在 Git 数据库中，带有你的变更快照。

```bash
# 提交暂存的变更
git commit -m "Add initial model implementation"
```

现在变更已提交到仓库历史中。

### 可视化表示

```
+-------------------+       +------------------+       +------------------+
|     工作目录      |       |     暂存区       |       |     GIT 仓库     |
|                   | ----> |    (INDEX)       | ----> |    (.git 目录)   |
+-------------------+       +------------------+       +------------------+
| model.py          |       | model.py         |       | commit abc123:   |
| (已修改)          |       | (已暂存)          |       | + model.py       |
| data_prep.py      |       |                  |       |                  |
| (未跟踪)          |       |                  |       |                  |
+-------------------+       +------------------+       +------------------+
        ↓                           ↓                          ↓
    git add                    git commit                  永久历史
```

### 为什么是三种状态？

暂存区提供了细粒度的控制：

1. **选择性提交**：即使在同一文件中，也只暂存特定的变更
2. **逻辑分组**：将相关变更合并为有意义的 commit
3. **提交前审查**：仔细检查你要提交的内容
4. **原子性提交**：确保每个 commit 是一个完整、逻辑的单元

---

## 基本 Git 操作

### 检查仓库状态

最常用的 Git 命令：

```bash
git status
```

输出示例：

```
On branch main
Your branch is up to date with 'origin/main'.

Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        modified:   src/model.py

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   src/train.py

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        src/inference.py
```

Status 告诉你：
- 当前分支
- 与远程的同步状态
- 已暂存的变更
- 未暂存的变更
- 未跟踪的文件

**专业提示**：经常运行 `git status`。把它变成习惯。

### 添加文件到暂存区

```bash
# 暂存单个文件
git add src/model.py

# 暂存多个文件
git add src/model.py src/train.py

# 暂存目录中的所有文件
git add src/

# 暂存仓库中的所有变更
git add .

# 暂存所有已修改和已删除的文件（不包括新文件）
git add -u

# 交互式暂存（逐一审查每个变更）
git add -i

# 暂存文件的特定部分（补丁模式）
git add -p src/model.py
```

#### 交互式暂存示例

```bash
git add -p model.py

# Git 将显示每个变更并提示：
Stage this hunk [y,n,q,a,d,s,e,?]?
  y - stage this hunk
  n - do not stage this hunk
  q - quit; do not stage this or remaining hunks
  a - stage this and all remaining hunks
  d - do not stage this or remaining hunks
  s - split the current hunk into smaller hunks
  e - manually edit the current hunk
  ? - print help
```

这在从杂乱的工and作目录制作原子性 commit 时非常有用。

### 提交变更

```bash
# 使用内联消息提交
git commit -m "Add PyTorch model architecture"

# 使用详细消息提交（打开编辑器）
git commit

# 使用内联消息和描述提交
git commit -m "Add model architecture" -m "Implemented ResNet-50 with custom head for our classification task. Includes batch normalization and dropout layers."

# 一步暂存并提交（仅跟踪的文件）
git commit -a -m "Update training configuration"

# 修改上次 commit（添加遗漏的变更或修复消息）
git add forgotten_file.py
git commit --amend

# 不改变消息的情况下修改
git commit --amend --no-edit
```

### 编写良好的 Commit 消息

Commit 消息是你未来自己和团队的文档。遵循以下指南：

#### 优秀 Commit 消息的七条规则

1. **用空行分隔主题和正文**
2. **将主题行限制在 50 个字符**
3. **主题行首字母大写**
4. **不要在主题行末尾加句号**
5. **在主题行中使用祈使语气**（"Add feature"而不是"Added feature"）
6. **将正文限制在 72 个字符**
7. **用正文解释是什么和为什么，而不是怎么做**

#### 示例：糟糕的 Commit 消息

```
fixed bug
```

什么 bug？在哪里？为什么会发生？

#### 示例：良好的 Commit 消息

```
Fix memory leak in batch data loading

The DataLoader was not releasing GPU memory after each batch
due to improper tensor cleanup. This caused OOM errors during
long training runs.

- Added explicit tensor deletion after gradient computation
- Implemented garbage collection trigger every 100 batches
- Reduced batch size from 64 to 32 as additional safeguard

Closes #127
```

这讲述了一个完整的故事：是什么、为什么、怎么做，以及它修复了哪个问题。

#### ML 项目的 Commit 消息模板

```
[Type]: 简短描述（50 字符）

变更的详细解释（每行 72 字符）

为什么需要这个变更：
- 原因 1
- 原因 2

它如何影响系统：
- 影响 1
- 影响 2

已执行的测试：
- 测试 1
- 测试 2

相关： #issue-number
```

Types: `feat`, `fix`, `refactor`, `docs`, `test`, `perf`, `style`, `chore`

### 查看差异

```bash
# 显示未暂存的变更（工作目录 vs. 暂存区）
git diff

# 显示已暂存的变更（暂存区 vs. 上次 commit）
git diff --staged
# 或
git diff --cached

# 显示特定文件中的变更
git diff src/model.py

# 显示 commit 之间的变更
git diff abc123 def456

# 显示分支之间的变更
git diff main feature/new-model

# 只显示变更的文件名
git diff --name-only

# 显示关于变更的统计信息
git diff --stat

# 显示单词级别的 diff（对文档有用）
git diff --word-diff
```

#### 理解 Diff 输出

```diff
diff --git a/src/model.py b/src/model.py
index 3a4b5c6..7d8e9f0 100644
--- a/src/model.py
+++ b/src/model.py
@@ -15,7 +15,7 @@ class NeuralNetwork(nn.Module):
         self.fc1 = nn.Linear(784, 256)
-        self.fc2 = nn.Linear(256, 128)
+        self.fc2 = nn.Linear(256, 64)
         self.fc3 = nn.Linear(128, 10)
```

分解说明：
- `---` 表示旧版本（文件 a）
- `+++` 表示新版本（文件 b）
- `@@ -15,7 +15,7 @@` 表示变更的行号
- `-` 行被移除（红色）
- `+` 行被添加（绿色）

### 删除和移动文件

```bash
# 从 Git 和文件系统中删除文件
git rm src/old_model.py

# 从 Git 中删除但保留在文件系统中（取消跟踪）
git rm --cached config/secrets.yaml

# 移动或重命名文件
git mv old_name.py new_name.py

# 相当于：
mv old_name.py new_name.py
git rm old_name.py
git add new_name.py
```

始终使用 `git mv` 和 `git rm` 而不是常规的 shell 命令，这样 Git 可以正确跟踪变更。

---

## 查看历史

### 基本 Log 命令

```bash
# 显示 commit 历史
git log

# 显示简洁历史（每行一个 commit）
git log --oneline

# 显示带图的历史
git log --graph --oneline --all

# 显示最近 N 个 commit
git log -5

# 显示带有完整 diff 的 commit
git log -p

# 显示每个 commit 的统计信息
git log --stat

# 显示日期范围内的 commit
git log --since="2 weeks ago"
git log --until="2023-10-01"

# 按作者显示 commit
git log --author="Jane Smith"

# 搜索 commit 消息
git log --grep="training"

# 显示修改了特定文件的 commit
git log -- src/model.py

# 显示添加或删除了特定字符串的 commit
git log -S "batch_size" --oneline
```

### 格式优美的 Log

```bash
# 自定义格式
git log --pretty=format:"%h - %an, %ar : %s"

# 输出：
# a1b2c3d - Jane Smith, 2 hours ago : Add model training script
# e4f5g6h - John Doe, 1 day ago : Fix data preprocessing bug
```

常用的格式占位符：
- `%h` - 简化的 commit 哈希
- `%an` - 作者姓名
- `%ar` - 作者日期，相对时间
- `%s` - Commit 主题
- `%b` - Commit 正文

### 有用的别名

将这些添加到你的 `.gitconfig`：

```bash
git config --global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit --date=relative"

git config --global alias.last "log -1 HEAD --stat"

git config --global alias.today "log --since='midnight' --oneline"
```

然后使用：
```bash
git lg      # 漂亮的图形日志
git last    # 显示带统计信息的最后一个 commit
git today   # 显示今天的 commit
```

### 查看特定 Commit

```bash
# 显示特定 commit 的详细信息
git show abc123

# 只显示更改的文件
git show --name-only abc123

# 显示 commit 中的特定文件
git show abc123:src/model.py

# 比较工作目录与特定 commit
git diff abc123
```

---

## Git 用于 AI/ML 项目

### .gitignore 文件

这对 ML 项目至关重要。你不希望跟踪：
- 大型模型文件
- 数据集
- 虚拟环境
- 实验输出
- 缓存文件

在你的仓库根目录创建 `.gitignore` 文件：

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
*.egg-info/
.installed.cfg
*.egg

# Virtual Environments
.env
.venv
ENV/
env/
venv/

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

# Datasets
data/
datasets/
*.csv
*.tsv
*.dat
*.json
!config.json
*.parquet

# Experiment Tracking
mlruns/
.mlflow/
wandb/
runs/
tensorboard/
logs/
*.log

# IDE
.vscode/
.idea/
*.swp
*.swo
*~
.DS_Store

# Docker
*.tar
.dockerignore

# Credentials
.env
.env.local
secrets/
*.key
*.pem
credentials.json
service-account.json

# AWS
.aws/

# GCP
.gcloud/

# Large files (even if tracked, can cause issues)
*.zip
*.tar.gz
*.rar
```

### 否定模式

有时你想忽略一个目录但跟踪其中的特定文件：

```gitignore
# Ignore all files in config/
config/*

# But track this specific file
!config/default.yaml
```

### 全局 .gitignore

对于操作系统和编辑器文件，这些永远不应被跟踪：

```bash
# 创建全局 gitignore
cat > ~/.gitignore_global << EOF
.DS_Store
Thumbs.db
*.swp
.vscode/
.idea/
EOF

# 配置 Git 使用它
git config --global core.excludesfile ~/.gitignore_global
```

### 检查文件是否被忽略

```bash
# 检查文件是否被忽略
git check-ignore -v data/train.csv

# 输出：.gitignore:45:*.csv  data/train.csv
# 显示 .gitignore 中的文件、行号和匹配的模式
```

### 取消跟踪已跟踪的文件

如果你在将文件添加到 `.gitignore` 之前不小心提交了它们：

```bash
# 从 Git 中移除但保留在文件系统中
git rm --cached models/large_model.pth

# 移除整个目录
git rm -r --cached data/

# 提交移除
git commit -m "Stop tracking large data files"
```

### 使用 Git LFS 处理大文件

对于必须进行版本控制的模型文件：

```bash
# 安装 Git LFS
git lfs install

# 跟踪大文件类型
git lfs track "*.pth"
git lfs track "*.h5"
git lfs track "*.onnx"

# 这会创建/更新 .gitattributes
git add .gitattributes

# 现在正常添加和提交大文件
git add models/resnet50.pth
git commit -m "Add pre-trained ResNet-50 model"
```

Git LFS 将大文件分开存储，只在仓库中保留指针。

### ML 项目的仓库结构

```
ml-training-pipeline/
├── .git/
├── .gitignore
├── .gitattributes           # For Git LFS
├── README.md
├── requirements.txt
├── setup.py
├── .env.example            # Template, not actual secrets
├── src/
│   ├── __init__.py
│   ├── data/
│   │   ├── __init__.py
│   │   └── preprocessing.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── resnet.py
│   ├── training/
│   │   ├── __init__.py
│   │   └── train.py
│   └── utils/
│       ├── __init__.py
│       └── helpers.py
├── configs/
│   ├── default.yaml
│   └── experiment_001.yaml
├── tests/
│   ├── __init__.py
│   ├── test_preprocessing.py
│   └── test_model.py
├── scripts/
│   ├── setup_env.sh
│   └── run_training.sh
└── notebooks/              # For exploration only
    └── eda.ipynb
```

---

## 常见问题与故障排除

### 问题 1：意外提交了密钥

**症状**：你意外提交了 API 密钥或密码。

**解决方案**：
```bash
# 如果还没有推送
git reset --soft HEAD~1   # 撤销 commit，保留变更
# 移除敏感文件，添加到 .gitignore
git add .gitignore
git commit -m "Add .gitignore for secrets"

# 如果已经推送（终极方案）
# 使用 git-filter-branch 或 BFG Repo-Cleaner
# 然后立即轮换所有受损凭据
```

**预防**：使用 `.env.example` 文件作为模板，永远不提交 `.env`。

### 问题 2：文件未被忽略

**症状**：尽管在 `.gitignore` 中，文件仍然出现在 `git status` 中。

**原因**：文件在被添加到 `.gitignore` 之前已被跟踪。

**解决方案**：
```bash
# 取消跟踪文件
git rm --cached filename
git commit -m "Stop tracking filename"
```

### 问题 3：大文件拖慢操作

**症状**：`git clone` 或 `git pull` 需要很长时间。

**原因**：直接提交了大文件（模型、数据集）。

**解决方案**：
```bash
# 对大文件使用 Git LFS
git lfs migrate import --include="*.pth,*.h5"

# 或者对 CI/CD 使用浅克隆
git clone --depth 1 <repository>
```

### 问题 4：Detached HEAD 状态

**症状**：消息显示"You are in 'detached HEAD' state"。

**原因**：检出了特定的 commit 而不是分支。

**解决方案**：
```bash
# 只是看看然后离开
git checkout main

# 保留在 detached HEAD 中所做的更改
git branch temp-branch
git checkout main
git merge temp-branch
```

### 问题 5：错误的 Commit 消息

**症状**：上次 commit 消息中有拼写错误。

**解决方案**：
```bash
# 如果还没有推送
git commit --amend -m "Corrected message"

# 如果已经推送（如果其他人已经拉取则避免）
git commit --amend -m "Corrected message"
git push --force-with-lease
```

**警告**：不要在未与团队协调的情况下对共享分支强制推送。

### 问题 6：忘记在上次 commit 中添加文件

**解决方案**：
```bash
git add forgotten_file.py
git commit --amend --no-edit
```

### Git 故障排除命令

```bash
# 查看 Git 实际在做什么
git status -vv

# 检查 Git 配置
git config --list --show-origin

# 验证仓库完整性
git fsck

# 清理未跟踪的文件（小心！）
git clean -n    # 试运行，显示将被删除的内容
git clean -f    # 实际删除未跟踪的文件
git clean -fd   # 删除未跟踪的文件和目录

# 从远程恢复损坏的仓库
git fetch origin
git reset --hard origin/main
```

---

## 总结与要点

### 核心概念

1. **Git 是分布式的**：每个开发者拥有完整的仓库历史
2. **三种状态**：工作目录 → 暂存区 → 仓库
3. **Commit 是快照**：每个 commit 是一个完整的快照，而不是差异
4. **SHA-1 哈希**：每个对象都被唯一标识且防篡改
5. **暂存区**：对 commit 提供细粒度控制

### 必备命令速查表

```bash
# 仓库设置
git init                    # 初始化仓库
git clone <url>            # 克隆已有仓库

# 配置
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# 基本工作流
git status                 # 检查仓库状态
git add <file>            # 暂存变更
git commit -m "message"   # 提交暂存的变更
git diff                  # 查看未暂存的变更
git diff --staged         # 查看已暂存的变更

# 历史
git log                   # 查看 commit 历史
git log --oneline         # 简洁历史
git show <commit>         # 显示 commit 详情

# 文件操作
git rm <file>             # 删除文件
git mv <old> <new>        # 移动/重命名文件

# 撤销
git restore <file>        # 丢弃工作目录变更
git restore --staged <file>  # 取消暂存变更
git reset HEAD~1          # 撤销上次 commit（保留变更）
git commit --amend        # 修改上次 commit
```

### AI 基础设施的最佳实践

1. **频繁提交**：小的、逻辑清晰的 commit 更容易审查和回滚
2. **编写描述性消息**：解释为什么，而不仅仅是什么
3. **正确使用 .gitignore**：永不提交大文件、数据集或密钥
4. **选择性暂存**：使用 `git add -p` 进行原子性提交
5. **提交前审查**：始终运行 `git diff --staged`
6. **配置 Git LFS**：用于必须进行版本控制的模型文件
7. **永不提交密钥**：使用环境变量和 `.env` 文件
8. **保持 commit 原子性**：每个 commit 一个逻辑变更
9. **提交前测试**：提交代码前运行测试
10. **使用分支**：永远不要直接提交到 main（在下一讲中介绍）

### AI/ML 特定注意事项

- **可重现性**：将代码和配置一起提交
- **模型版本控制**：使用 Git LFS 或外部模型注册表
- **数据版本控制**：使用 DVC，而不是 Git
- **实验跟踪**：提交实验配置，而不是输出
- **大文件**：始终使用 .gitignore 或 Git LFS
- **凭据**：使用密钥管理工具，而不是 Git

### 下一步

在下一讲中，我们将介绍：
- ML 开发的分支策略
- 合并与冲突解决
- 团队的 Git 工作流
- 功能分支与发布管理

### 扩展阅读

- 官方 Git 文档：https://git-scm.com/doc
- Pro Git 书籍（免费）：https://git-scm.com/book/en/v2
- Git LFS 文档：https://git-lfs.github.com/
- DVC 用于数据版本控制：https://dvc.org/

---

**练习**：完成练习 01 和 02，通过真实 ML 项目场景的动手实践来巩固这些概念。
