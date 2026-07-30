# Lecture 01：Python 环境与依赖管理

## 目录
1. [引言](#引言)
2. [理解 Python 环境](#理解-python-环境)
3. [虚拟环境深入解析](#虚拟环境深入解析)
4. [使用 pip 进行包管理](#使用-pip-进行包管理)
5. [需求文件](#需求文件)
6. [Python 版本管理](#python-版本管理)
7. [AI 基础设施的最佳实践](#ai-基础设施的最佳实践)
8. [常见问题与故障排除](#常见问题与故障排除)
9. [总结与关键要点](#总结与关键要点)

---

## 引言

### 为什么环境管理很重要

想象这样一个场景：你花了数周时间开发一个 ML 模型训练流水线。它在你的笔记本电脑上完美运行。你将其部署到生产环境，突然出现神秘的错误。模型无法加载。库抛出版本不兼容警告。你精心调优的流水线崩溃了。

发生了什么？**环境不匹配**。

在 AI 基础设施工程中，环境管理不仅仅是最佳实践——它是生存之道。ML 生态涉及数百个依赖项，许多具有复杂的版本要求。PyTorch 需要特定的 CUDA 版本。TensorFlow 对 numpy 版本有严格要求。一个不匹配的包可能级联导致系统范围的故障。

本讲座教你通过严格的环境和依赖管理来防止这些灾难。

### 学习目标

完成本讲座后，你将能够：

- 理解 Python 如何查找和加载包
- 使用多种工具创建隔离的虚拟环境
- 有效使用 pip 管理依赖
- 编写可靠的 requirements 文件
- 在一个系统上处理多个 Python 版本
- 将环境管理应用于 AI/ML 工作流
- 排查常见的环境问题

### AI 基础设施背景

AI 基础设施工程师工作在以下交叉领域：
- **数据科学**：为数据科学家和 ML 工程师管理环境
- **软件工程**：构建生产级 Python 应用程序
- **DevOps**：跨多个环境部署和维护服务

你需要确保：
- 训练作业使用正确的库版本运行
- 模型服务 API 加载正确的 ML 框架版本
- 开发环境与生产环境完全匹配
- 团队成员可以重现彼此的工作
- CI/CD 流水线使用一致的依赖

---

## 理解 Python 环境

### Python 如何查找包

当你编写 `import tensorflow` 时，Python 通过特定过程搜索包：

```
1. 检查 sys.modules（已导入的包）
2. 搜索 sys.path 中的目录：
   - 当前目录
   - PYTHONPATH 环境变量
   - 标准库目录
   - Site-packages 目录（第三方包）
```

你可以自己检查：

```python
import sys
print(sys.path)
```

输出可能如下：
```
[
    '/home/user/my-project',  # 当前目录
    '/usr/lib/python3.11',    # 标准库
    '/usr/lib/python3.11/site-packages',  # 系统包
    '/home/user/.local/lib/python3.11/site-packages'  # 用户包
]
```

### 全局与本地包问题

**全局安装**（有问题）：
```bash
pip install tensorflow  # 安装到系统 Python
```

全局安装的问题：
- 每个项目使用相同的包版本
- 为一个项目升级会破坏其他项目
- 无法测试不同版本
- 系统 Python 有被破坏的风险
- 许多系统需要管理员/sudo 权限
- 项目需求之间的冲突

**本地安装**（隔离）：
```bash
python -m venv my-project-env
source my-project-env/bin/activate
pip install tensorflow  # 仅安装到此环境
```

隔离的好处：
- 每个项目有独立的依赖
- 安全地尝试版本实验
- 无需管理员权限
- 跨机器可重现
- 每个项目有清晰的依赖清单

### Site-Packages 目录结构

理解包的存放位置有助于调试导入问题：

```
my-venv/
├── bin/               # 可执行文件（python、pip、脚本）
│   ├── python -> python3.11
│   ├── pip
│   └── activate       # 激活脚本
├── include/           # 编译扩展的 C 头文件
├── lib/
│   └── python3.11/
│       └── site-packages/    # 所有已安装包都在这里
│           ├── tensorflow/
│           ├── numpy/
│           ├── requests/
│           └── ...
└── pyvenv.cfg         # 环境配置
```

当你激活虚拟环境时，它会修改：
- `PATH`：前置 `my-venv/bin/` 以使其 python 先运行
- `PYTHONPATH`：指向 `my-venv/lib/python3.11/site-packages`

---

## 虚拟环境深入解析

### 使用 venv（内置，推荐）

`venv` 模块随 Python 3.3+ 提供。它是标准的官方解决方案。

#### 创建环境

```bash
# 基本创建
python3 -m venv myenv

# 显式指定 Python 版本
python3.11 -m venv myenv

# 在特定位置创建
python3 -m venv /path/to/my/project/venv
```

**创建过程中发生了什么？**
1. 创建目录结构（bin、lib、include）
2. 复制或符号链接 Python 解释器
3. 安装 pip 和 setuptools
4. 为各种 shell 创建激活脚本

#### 激活环境

**Linux/macOS（bash/zsh）**：
```bash
source myenv/bin/activate
```

**Windows（命令提示符）**：
```cmd
myenv\Scripts\activate.bat
```

**Windows（PowerShell）**：
```powershell
myenv\Scripts\Activate.ps1
```

**激活操作的效果**：
```bash
# 激活前
which python
# /usr/bin/python

source myenv/bin/activate

# 激活后
which python
# /home/user/myproject/myenv/bin/python

echo $PATH
# /home/user/myproject/myenv/bin:/usr/local/bin:/usr/bin:...
```

你的提示符会改变以指示活动环境：
```bash
(myenv) user@host:~/project$
```

#### 在环境中工作

```bash
# 激活
source myenv/bin/activate

# 安装包（仅影响此环境）
pip install numpy pandas tensorflow

# 检查已安装的包
pip list

# 运行 Python 代码（使用环境的包）
python my_script.py

# 完成后停用
deactivate
```

#### 环境最佳实践

**1. 环境位置**
```bash
# 好：在项目根目录中，被 gitignore
my-ml-project/
├── venv/              # 或 .venv/
├── src/
├── tests/
├── requirements.txt
└── .gitignore         # 包含 venv/
```

**2. 命名约定**
```bash
venv        # 最常见
.venv       # 隐藏，保持目录整洁
env         # 简短替代
virtualenv  # 显式
```

**3. Gitignore 虚拟环境**
```gitignore
# .gitignore
venv/
.venv/
env/
*.pyc
__pycache__/
```

永远不要将虚拟环境提交到 git——它们是机器特定的且易于重新创建。

### 高级：virtualenv vs venv

虽然 `venv` 是内置的且足够使用，但 `virtualenv` 提供了额外功能：

```bash
# 安装 virtualenv
pip install virtualenv

# 创建环境
virtualenv myenv

# venv 没有的功能：
virtualenv --python=python3.9 myenv     # 指定任何 Python 版本
virtualenv --system-site-packages myenv  # 访问系统包
virtualenv --no-setuptools myenv         # 最小环境
```

**何时使用 virtualenv**：
- 需要 venv 无法管理的 Python 版本
- 想要额外的自定义选项
- 处理遗留的 Python 2.7 项目
- 需要更快的环境创建（它经过优化）

**对于 AI 基础设施工作，venv 通常就足够了。**

### 替代方案：pyenv 和 pyenv-virtualenv

用于管理多个 Python 版本和虚拟环境：

```bash
# 安装 pyenv（Linux/macOS）
curl https://pyenv.run | bash

# 安装特定 Python 版本
pyenv install 3.11.5

# 设置全局 Python 版本
pyenv global 3.11.5

# 使用 pyenv-virtualenv 创建虚拟环境
pyenv virtualenv 3.11.5 my-ml-project

# 进入目录时自动激活
cd my-ml-project
echo "my-ml-project" > .python-version
# 现在当你 cd 进入目录时会自动激活！
```

**ML 工作的优势**：
- 针对多个 Python 版本测试代码
- 每个目录自动激活
- 无需 root 权限即可管理 Python 版本
- 与 venv 无缝集成

---

## 使用 pip 进行包管理

### 理解 pip

`pip` 是 Python 的包安装程序，与 PyPI（Python 包索引）和其他仓库交互。

#### 基本 pip 命令

```bash
# 安装包
pip install requests

# 安装特定版本
pip install requests==2.31.0

# 安装版本约束
pip install "requests>=2.31.0,<3.0.0"

# 安装多个包
pip install requests numpy pandas

# 升级包
pip install --upgrade requests

# 卸载包
pip uninstall requests

# 显示包信息
pip show tensorflow

# 列出已安装的包
pip list

# 列出过时的包
pip list --outdated
```

#### 理解版本说明符

```bash
# 精确版本
pip install numpy==1.24.0

# 最低版本
pip install numpy>=1.24.0

# 兼容版本（允许补丁更新）
pip install numpy~=1.24.0     # 允许 1.24.x，不允许 1.25.0

# 排除版本
pip install "numpy>=1.24.0,<2.0.0,!=1.24.1"  # 避免有 bug 的 1.24.1

# 最新版本（生产环境中危险）
pip install numpy
```

**对于 AI 基础设施，始终在生产环境中固定版本！**

### Pip 配置

创建 `~/.pip/pip.conf`（Linux/macOS）或 `%APPDATA%\pip\pip.ini`（Windows）：

```ini
[global]
timeout = 60
index-url = https://pypi.org/simple

[install]
trusted-host = pypi.org
               pypi.python.org
               files.pythonhosted.org
```

**私有包仓库**：
```ini
[global]
index-url = https://my-company-pypi.example.com/simple
extra-index-url = https://pypi.org/simple
```

### 从不同来源安装

```bash
# 从 PyPI（默认）
pip install tensorflow

# 从 Git 仓库
pip install git+https://github.com/user/repo.git

# 从特定分支
pip install git+https://github.com/user/repo.git@dev-branch

# 从本地目录（开发模式）
pip install -e /path/to/package  # 可编辑安装

# 从本地 wheel 文件
pip install /path/to/package.whl

# 从需求文件
pip install -r requirements.txt
```

### 依赖解析

当你 `pip install tensorflow` 时，pip 必须：
1. 获取 tensorflow 的元数据
2. 读取其依赖（numpy、protobuf 等）
3. 递归获取依赖元数据
4. 解析所有包的兼容版本
5. 按正确顺序下载和安装

**依赖树示例**：
```
tensorflow==2.13.0
├── numpy>=1.24.0,<2.0.0
├── protobuf>=3.20.0
├── tensorboard>=2.13.0
│   ├── numpy>=1.12.0
│   └── protobuf>=3.19.0
└── keras>=2.13.0
    └── numpy>=1.23.0
```

Pip 找到满足所有约束的版本：`numpy>=1.24.0`（来自 tensorflow 最严格的要求）。

**依赖冲突**发生在没有版本满足所有约束时：
```
Package A 需要 numpy>=1.24.0
Package B 需要 numpy<1.20.0
# 没有 numpy 版本同时满足两者！pip 将报错。
```

---

## 需求文件

需求文件记录项目的依赖，实现可重现性。

### 基本 Requirements.txt

```txt
# requirements.txt
numpy==1.24.3
pandas==2.0.3
scikit-learn==1.3.0
tensorflow==2.13.0
requests==2.31.0
```

**从当前环境创建**：
```bash
pip freeze > requirements.txt
```

**从需求文件安装**：
```bash
pip install -r requirements.txt
```

### Requirements.txt 最佳实践

#### 1. 为生产环境固定精确版本

```txt
# 好：可重现的构建
numpy==1.24.3
tensorflow==2.13.0

# 不好：不可预测（可能在部署之间崩溃）
numpy
tensorflow>=2.0.0
```

#### 2. 使用注释提高清晰度

```txt
# 核心 ML 框架
tensorflow==2.13.0      # 模型训练
scikit-learn==1.3.0     # 预处理

# 数据处理
numpy==1.24.3
pandas==2.0.3

# API 服务
fastapi==0.103.0
uvicorn==0.23.2

# 监控
prometheus-client==0.17.1
```

#### 3. 分离开发和生产的依赖

**requirements.txt**（生产）：
```txt
tensorflow==2.13.0
numpy==1.24.3
fastapi==0.103.0
```

**requirements-dev.txt**（开发）：
```txt
-r requirements.txt     # 包含生产依赖

# 测试
pytest==7.4.0
pytest-cov==4.1.0
pytest-mock==3.11.1

# 代码质量
black==23.7.0
flake8==6.1.0
mypy==1.5.1

# 开发工具
ipython==8.14.0
jupyter==1.0.0
```

两者都安装：
```bash
pip install -r requirements-dev.txt
```

#### 4. 处理平台特定依赖

```txt
# 通用依赖
numpy==1.24.3
pandas==2.0.3

# Linux 特定（GPU 支持）
tensorflow-gpu==2.13.0; sys_platform == 'linux'

# macOS 特定（仅 CPU）
tensorflow==2.13.0; sys_platform == 'darwin'

# Windows 特定
pywin32==306; sys_platform == 'win32'
```

### 高级：Pip-tools 用于锁定文件

**问题**：固定版本的 `requirements.txt` 难以维护。

**解决方案**：使用 `pip-tools` 将高级需求与锁定版本分离。

```bash
pip install pip-tools
```

**requirements.in**（高级需求）：
```txt
tensorflow>=2.13.0,<3.0.0
numpy
pandas
```

**生成锁定的 requirements.txt**：
```bash
pip-compile requirements.in
```

**生成包含所有依赖都固定的 requirements.txt**：
```txt
# This file is autogenerated by pip-compile
# To update, run: pip-compile requirements.in

tensorflow==2.13.0
    # via -r requirements.in
numpy==1.24.3
    # via -r requirements.in
    # via pandas
    # via tensorflow
pandas==2.0.3
    # via -r requirements.in
protobuf==4.23.4
    # via tensorflow
# ... 所有传递依赖都被固定
```

**好处**：
- 维护简单的 `.in` 文件
- 获得完全固定、可重现的 `.txt`
- 易于更新：`pip-compile --upgrade`
- 查看每个依赖存在的原因

**工作流**：
```bash
# 编辑高级需求
vim requirements.in

# 重新生成锁定文件
pip-compile requirements.in

# 安装精确版本
pip-sync requirements.txt

# 提交两个文件
git add requirements.in requirements.txt
git commit -m "Update dependencies"
```

---

## Python 版本管理

AI/ML 项目通常需要特定的 Python 版本。TensorFlow 2.13 可能需要 Python 3.8-3.11。你需要灵活性。

### 使用 pyenv（推荐）

**安装（Linux/macOS）**：
```bash
curl https://pyenv.run | bash

# 添加到 ~/.bashrc 或 ~/.zshrc
export PATH="$HOME/.pyenv/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

**基本 pyenv 用法**：
```bash
# 列出可用的 Python 版本
pyenv install --list

# 安装特定版本
pyenv install 3.11.5
pyenv install 3.10.13
pyenv install 3.9.18

# 列出已安装的版本
pyenv versions

# 设置全局 Python 版本
pyenv global 3.11.5

# 为项目目录设置本地版本
cd my-ml-project
pyenv local 3.10.13
# 创建 .python-version 文件

# 临时使用特定版本
pyenv shell 3.9.18
```

**目录特定的 Python 版本**：
```bash
my-ml-project/
├── .python-version    # 包含 "3.10.13"
├── venv/
└── requirements.txt

# 当你 cd 进入目录时，Python 3.10.13 自动激活！
cd my-ml-project
python --version  # Python 3.10.13
```

### 将 pyenv 与虚拟环境结合

**使用特定 Python 版本创建环境**：
```bash
# 安装 Python 版本
pyenv install 3.11.5

# 使用该版本创建虚拟环境
pyenv virtualenv 3.11.5 my-project-env

# 激活它
pyenv activate my-project-env

# 或将其设置为目录的本地环境
cd my-project
pyenv local my-project-env  # 进入目录时自动激活
```

### Windows：管理 Python 版本

Windows 没有 pyenv，但有替代方案：

**选项 1：Python Launcher（py）**：
```cmd
REM 从 python.org 安装多个 Python 版本

REM 列出已安装的版本
py --list

REM 运行特定版本
py -3.11 script.py
py -3.10 -m venv myenv

REM 使用特定版本创建 venv
py -3.11 -m venv myenv-py311
```

**选项 2：Conda（重量级但跨平台）**：
```bash
conda create -n myenv python=3.11
conda activate myenv
```

---

## AI 基础设施的最佳实践

### 1. 可重现环境

**始终包含**：
```
my-ml-project/
├── .python-version        # 指定 Python 版本
├── requirements.txt       # 固定依赖
├── README.md             # 设置说明
└── Makefile 或 setup.sh  # 自动设置
```

**setup.sh 示例**：
```bash
#!/bin/bash
set -e

echo "Setting up ML training environment..."

# 检查 Python 版本
python --version | grep "3.11" || {
    echo "Error: Python 3.11 required"
    exit 1
}

# 创建虚拟环境
python -m venv venv

# 激活
source venv/bin/activate

# 升级 pip
pip install --upgrade pip

# 安装依赖
pip install -r requirements.txt

echo "Environment ready! Activate with: source venv/bin/activate"
```

使其可执行：
```bash
chmod +x setup.sh
./setup.sh
```

### 2. Docker 实现完全隔离

虚拟环境隔离 Python 包，但不隔离：
- 系统库（CUDA、cuDNN）
- Python 解释器本身
- 操作系统级依赖

Docker 提供完全隔离：

**Dockerfile**：
```dockerfile
FROM python:3.11-slim

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# 设置工作目录
WORKDIR /app

# 先复制需求文件（Docker 层缓存）
COPY requirements.txt .

# 安装 Python 依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用程序代码
COPY . .

# 运行应用程序
CMD ["python", "train.py"]
```

构建和运行：
```bash
docker build -t my-ml-training .
docker run my-ml-training
```

### 3. CI/CD 环境一致性

**GitHub Actions 示例**：
```yaml
# .github/workflows/test.yml
name: Test ML Pipeline

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3

    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'

    - name: Cache pip packages
      uses: actions/cache@v3
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
        pip install -r requirements-dev.txt

    - name: Run tests
      run: pytest tests/
```

这确保了测试每次都使用精确的依赖运行。

### 4. 用于配置的环境变量

永远不要硬编码配置。使用环境变量：

**.env 文件**：
```bash
# .env（永远不要提交到 git！）
MODEL_PATH=/data/models
BATCH_SIZE=32
LEARNING_RATE=0.001
AWS_ACCESS_KEY_ID=your-key-id
AWS_SECRET_ACCESS_KEY=your-secret
```

**在 Python 中加载**：
```python
import os
from dotenv import load_dotenv

load_dotenv()  # 加载 .env 文件

model_path = os.getenv('MODEL_PATH', '/default/path')
batch_size = int(os.getenv('BATCH_SIZE', '16'))
```

**包含在 .gitignore 中**：
```gitignore
.env
.env.local
.env.*.local
```

**提供模板**：
```bash
# .env.example（提交此文件）
MODEL_PATH=/data/models
BATCH_SIZE=32
LEARNING_RATE=0.001
AWS_ACCESS_KEY_ID=your-key-id-here
AWS_SECRET_ACCESS_KEY=your-secret-here
```

---

## 常见问题与故障排除

### 问题 1："Command 'python' not found"

**问题**：系统使用 `python3` 而不是 `python`。

**解决方案**：
```bash
# 选项 1：显式使用 python3
python3 -m venv myenv

# 选项 2：创建别名
echo "alias python=python3" >> ~/.bashrc
source ~/.bashrc

# 选项 3：符号链接
sudo ln -s /usr/bin/python3 /usr/bin/python
```

### 问题 2：权限被拒绝

**问题**：
```bash
pip install tensorflow
# ERROR: Could not install packages due to an OSError: [Errno 13] Permission denied
```

**解决方案**：不要使用 sudo！使用虚拟环境：
```bash
python -m venv myenv
source myenv/bin/activate
pip install tensorflow  # 没有权限问题
```

### 问题 3：依赖冲突

**问题**：
```
ERROR: Cannot install packageA and packageB because these package versions have conflicting dependencies.
```

**解决方案**：
```bash
# 1. 检查冲突内容
pip install --dry-run packageA packageB

# 2. 尝试顺序安装
pip install packageA
pip install packageB

# 3. 使用兼容版本
pip install "packageA==1.0.0" "packageB==2.0.0"

# 4. 最后手段：分离环境
python -m venv env-for-A
python -m venv env-for-B
```

### 问题 4：安装后包版本错误

**问题**：安装了新版本但导入时显示旧版本。

**解决方案**：
```bash
# 1. 检查实际安装的内容
pip show tensorflow

# 2. 重启 Python（旧导入已缓存）
# exit() 然后重启

# 3. 强制重新安装
pip install --force-reinstall tensorflow==2.13.0

# 4. 检查是否有系统级安装遮盖了 venv
which python  # 应显示 venv 路径
```

### 问题 5：安装后出现"No module named X"

**问题**：已安装包但 Python 找不到它。

**解决方案**：
```bash
# 1. 检查虚拟环境是否已激活
which python  # 应在 venv 路径中

# 2. 检查包是否安装在正确环境中
pip list | grep package-name

# 3. 在正确环境中安装
source venv/bin/activate
pip install package-name

# 4. 检查拼写错误
pip list  # 查看确切的包名
```

---

## 总结与关键要点

### 基本命令参考

```bash
# 虚拟环境
python -m venv myenv
source myenv/bin/activate  # Linux/macOS
myenv\Scripts\activate     # Windows
deactivate

# 包管理
pip install package-name
pip install -r requirements.txt
pip freeze > requirements.txt
pip list
pip show package-name

# 版本管理（pyenv）
pyenv install 3.11.5
pyenv global 3.11.5
pyenv local 3.11.5
```

### 最佳实践检查清单

✅ 始终为项目使用虚拟环境
✅ 在生产环境中固定 requirements.txt 中的精确版本
✅ 将开发依赖与生产依赖分离
✅ 永远不要将虚拟环境提交到 git
✅ 使用 .env 文件进行配置
✅ 在 README 中记录设置过程
✅ 在干净环境中测试依赖安装
✅ 对复杂项目使用 pip-tools 或 Poetry
✅ 考虑使用 Docker 实现完全环境可重现性
✅ 随着依赖变化保持 requirements.txt 更新

### 你学到的内容

1. **Python 的导入系统**以及它如何查找包
2. **虚拟环境**用于依赖隔离
3. **pip 包管理**和版本说明符
4. **需求文件**用于可重现安装
5. **Python 版本管理**使用 pyenv
6. **AI/ML 基础设施项目的最佳实践**
7. **排查**常见环境问题

### 下一步

你现在了解了 Python 环境管理——所有 AI 基础设施工作的基础。接下来：

1. 完成**练习 01**：创建一个具有适当环境设置的完整项目
2. 练习创建和管理多个环境
3. 尝试使用 pip-tools 或 Poetry
4. 进入**Lecture 02**：基础设施的高级 Python 模式

### 附加资源

- [Python 打包用户指南](https://packaging.python.org/)
- [pip 文档](https://pip.pypa.io/)
- [pyenv GitHub 仓库](https://github.com/pyenv/pyenv)
- [pip-tools 文档](https://pip-tools.readthedocs.io/)
- [PEP 405 - Python 虚拟环境](https://peps.python.org/pep-0405/)

---

**讲座版本**：1.0
**最后更新**：2025 年 10 月
**字数**：约 3,800 字
**预计阅读时间**：45-60 分钟

**准备好继续了吗？** 进入 `02-advanced-python.md` 学习类型提示、日志记录和配置管理。
