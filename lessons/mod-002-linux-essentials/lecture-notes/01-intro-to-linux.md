# 讲座 01：Linux 和命令行简介

## 目录
1. [简介](#简介)
2. [Linux 的历史和哲学](#linux-的历史和哲学)
3. [Linux 发行版](#linux-发行版)
4. [终端模拟器和 Shell](#终端模拟器和-shell)
5. [基本命令结构](#基本命令结构)
6. [获取帮助和文档](#获取帮助和文档)
7. [环境变量](#环境变量)
8. [AI 基础设施的最佳实践](#ai-基础设施的最佳实践)
9. [总结和要点](#总结和要点)

## 简介

作为一名 AI 基础设施工程师，Linux 是你的基石。无论是部署机器学习模型、管理 GPU 服务器还是编排容器，Linux 驱动着超过 90% 的云基础设施，并且是 AI/ML 工作负载的标准。

本讲座向你介绍 Linux、其哲学、命令行环境以及为你后续深入技术工作做准备的基本概念。

### 学习目标

完成本讲座后，你将能够：
- 理解 Linux 的历史和哲学
- 浏览 Linux 发行版全景
- 使用终端模拟器并理解不同的 shell
- 构建和执行基本 Linux 命令
- 访问帮助和文档系统
- 使用环境变量
- 将 Linux 基础应用到 AI 基础设施场景

### 先决条件
- 可访问 Linux 系统（Ubuntu 22.04 LTS、CentOS 8 或类似系统）
- 基本计算机素养
- 愿意学习命令行界面

### 为什么 AI 基础设施选择 Linux？

**行业标准**：超过 90% 的云基础设施运行在 Linux 上。TensorFlow、PyTorch 和 CUDA 等主要 AI 平台都针对 Linux 环境进行了优化。

**性能**：Linux 提供直接硬件访问，这对 GPU 计算和高性能 ML 工作负载至关重要。

**自动化**：Shell 脚本和命令行工具能够实现大规模基础设施自动化。

**开源**：免费、可定制，并得到广泛社区资源的支持。

**容器原生**：Docker、Kubernetes 和现代编排工具都是为 Linux 构建的。

## Linux 的历史和哲学

### Unix 的诞生（1969）

故事始于贝尔实验室的 **Unix**，由 Ken Thompson 和 Dennis Ritchie 创建。Unix 引入了革命性的概念：
- 层级文件系统
- 一切都是文件
- 小型、可组合的程序
- 用于连接程序的管道
- 跨硬件可移植

**Unix 哲学**："做好一件事，并且做好它"

### GNU 项目（1983）

Richard Stallman 发起了 **GNU 项目**（GNU's Not Unix），旨在创建一个自由的类 Unix 操作系统：
- 自由软件基金会
- GNU 工具（bash、gcc、make 等）
- GPL（通用公共许可证）
- **自由**：使用、研究、修改、分发软件

### Linux 内核（1991）

Linus Torvalds，一位芬兰计算机科学系学生，创建了 **Linux 内核**：
```
日期：1991 年 8 月 25 日
来自：torvalds@klaava.Helsinki.FI（Linus Benedict Torvalds）
主题：你最想在 minix 中看到什么？

各位使用 minix 的朋友们好 -

我正在为 386(486) AT 兼容机编写一个（自由的）操作系统（只是个爱好，不会像 gnu 那样大而专业）。这是从四月份开始酝酿的，现在开始逐渐成形了。
```

**关键创新**：
- 从第一天开始就开源
- 宏内核架构
- 社区驱动的开发
- 快速迭代和改进

### GNU/Linux 组合

**GNU + Linux 内核 = 完整的操作系统**
- GNU 提供用户空间工具
- Linux 提供内核
- 两者结合：一个自由、强大、类 Unix 的操作系统

### Linux 今天

**统计数据（2024）**：
- 100% 的超级计算机运行 Linux
- 96.3% 的前一百万 web 服务器运行 Linux
- 85% 的智能手机（Android）使用 Linux 内核
- 主要云提供商（AWS、Azure、GCP）主要使用 Linux
- NASA、CERN、SpaceX 使用 Linux

## Linux 发行版

**Linux 发行版**将 Linux 内核与软件、工具、包管理器和配置打包在一起。

### 主要发行版家族

#### 1. Debian 家族
**Debian**：稳定、社区驱动、巨大的包仓库
- **Ubuntu**：最流行、用户友好、社区强大
  - **Ubuntu Server**：用于服务器和云
  - **Ubuntu LTS**：长期支持（5 年）
- **Linux Mint**：桌面导向、对初学者友好

**包管理器**：`apt`、`dpkg`
**AI/ML 用途**：非常常见，优秀的 NVIDIA/CUDA 支持

#### 2. Red Hat 家族
**Red Hat Enterprise Linux（RHEL）**：企业级、商业支持
- **CentOS**：免费的 RHEL 克隆（正在被 Rocky/Alma 取代）
- **Fedora**：前沿、Red Hat 赞助
- **Rocky Linux**：CentOS 替代品
- **AlmaLinux**：另一个 CentOS 替代品

**包管理器**：`yum`、`dnf`、`rpm`
**AI/ML 用途**：在企业环境中常见

#### 3. SUSE 家族
**SUSE Linux Enterprise**：企业级 Linux，在欧洲流行
- **openSUSE**：社区版本

**包管理器**：`zypper`
**AI/ML 用途**：不太常见但受支持

#### 4. Arch 家族
**Arch Linux**：滚动发布、前沿、DIY
- **Manjaro**：用户友好的 Arch 衍生版

**包管理器**：`pacman`
**AI/ML 用途**：开发者工作站，生产环境不太常见

### 为 AI 基础设施选择发行版

**用于学习**（本课程推荐）：
- **Ubuntu 22.04 LTS**：最佳文档、社区支持
- **Ubuntu 20.04 LTS**：仍广泛使用，非常稳定

**用于生产**：
- **Ubuntu Server LTS**：云、GPU 服务器、ML 平台
- **RHEL/Rocky/Alma**：企业级、受监管行业
- **Amazon Linux 2**：AWS 优化

**关键因素**：
- LTS（长期支持）确保稳定性
- 硬件支持（NVIDIA 驱动、CUDA）
- 包可用性（ML 框架）
- 社区规模和文档
- 企业支持需求

## 终端模拟器和 Shell

### 什么是终端？

**终端**是你与命令行环境的接口。

**历史背景**：
- 最初：物理硬件终端（VT100 等）
- 现代：在图形环境中运行的软件模拟器

### 常见终端模拟器

**Linux 桌面**：
- **GNOME Terminal**（Ubuntu 默认）
- **Konsole**（KDE）
- **Terminator**：分割面板、高级功能
- **Alacritty**：GPU 加速、快速
- **Kitty**：基于 GPU、现代

**远程访问**：
- **SSH 客户端**：PuTTY（Windows）、iTerm2（macOS）、内置 ssh

**专业工具**：
- **tmux**：终端复用器（分割面板、持久化）
- **screen**：较老的复用器，仍广泛使用

### 理解 Shell

**Shell**是你和内核之间的命令解释器。

#### 常见 Shell

**Bash（Bourne Again Shell）**：
- 大多数 Linux 系统的默认 shell
- 丰富的脚本能力
- 广泛的文档
- **AI 基础设施自动化的最常用 shell**

```bash
# 检查你当前的 shell 一个环境变量，值是当前用户默认登录 Shell 的绝对路径
echo $SHELL
# /bin/bash

# Bash 版本
bash --version
# GNU bash, version 5.1.16
```

**Zsh（Z Shell）**：
- 高级特性、插件
- Oh-My-Zsh 框架
- 更好的自动补全
- 在开发者中流行

```bash
# 安装 zsh
sudo apt install zsh

# 更改默认 shell
chsh -s $(which zsh)
```

**Fish（友好的交互式 Shell）**：
- 用户友好、语法高亮
- 自动建议
- 与 bash 不同的语法

**Sh（Bourne Shell）**：
- 原始 Unix shell
- 符合 POSIX、可移植
- 用于系统脚本

#### Shell 比较

| 特性 | Bash | Zsh | Fish | Sh |
|---------|------|-----|------|-----|
| 可移植性 | 高 | 中 | 低 | 非常高 |
| 脚本能力 | 优秀 | 优秀 | 不同 | 基本 |
| 交互式 | 好 | 优秀 | 优秀 | 基本 |
| 兼容性 | 标准 | 类似 Bash | 独特 | POSIX |
| AI/ML 脚本 | ✓✓✓ | ✓✓ | ✓ | ✓✓ |

**建议**：先学习 Bash。它是通用的，是系统管理和自动化的标准。

## 基本命令结构

### 命令组成

```
command [options] [arguments]
   │        │           │
   │        │           └─ 操作对象
   │        └─ 如何修改行为
   └─ 要执行的操作
```

**示例**：
```bash
ls                      # 仅命令
ls -l                   # 命令 + 选项
ls -l /home            # 命令 + 选项 + 参数
ls -la /home/alice     # 命令 + 多个选项 + 参数
```

### 选项（标志）

**短选项**（单短横线）：
```bash
ls -l                   # 长格式
ls -a                   # 显示所有（包括隐藏）
ls -h                   # 人类可读大小
ls -lah                 # 组合多个选项
```

**长选项**（双短横线）：
```bash
ls --all                # 等同于 -a
ls --human-readable     # 等同于 -h
ls --help               # 显示帮助
```

### 参数

参数指定命令的操作对象：
```bash
cat file.txt            # 显示 file.txt
mkdir new_directory     # 创建 new_directory
cp source.txt dest.txt  # 复制 source 到 dest
```

### 常见命令模式

**列出文件**：
```bash
ls                      # 当前目录
ls /var/log            # 特定目录
ls *.py                # 模式匹配（所有 Python 文件）
```

**导航**：
```bash
pwd                     # 打印工作目录
cd /home/alice         # 更改到绝对路径
cd documents           # 更改到相对路径
cd ..                  # 上一级
cd ~                   # 主目录
cd -                   # 上一个目录
```

**文件查看**：
```bash
cat file.txt           # 显示整个文件
head file.txt          # 前 10 行
tail file.txt          # 最后 10 行
tail -f logfile.log    # 跟踪文件（实时更新）
less large_file.txt    # 分页浏览文件
```

**系统信息**：
```bash
whoami                 # 当前用户名
hostname               # 计算机名
uname -a               # 系统信息
date                   # 当前日期/时间
uptime                 # 系统运行时间
```

### 特殊字符

**通配符**：
```bash
*                      # 匹配任意字符
?                      # 匹配单个字符
[abc]                  # 匹配 a、b 或 c
[0-9]                  # 匹配任意数字

# 示例
ls *.py                # 所有 Python 文件
ls file?.txt           # file1.txt、fileA.txt 等
ls [a-z]*.log          # 以小写字母开头的日志
```

**重定向**：
```bash
>                      # 将输出重定向到文件（覆盖）
>>                     # 将输出重定向到文件（追加）
<                      # 从文件重定向输入
2>                     # 重定向 stderr
&>                     # 重定向 stdout 和 stderr

# 示例
echo "Hello" > file.txt        # 写入文件
echo "World" >> file.txt       # 追加到文件
python train.py > output.log   # 保存输出
python train.py 2> error.log   # 只保存错误
python train.py &> all.log     # 保存所有内容
```

**管道**：
```bash
|                      # 将输出发送到下一个命令

# 示例
ls -l | grep ".py"     # 列出然后过滤
cat file.txt | wc -l   # 计算行数
ps aux | grep python   # 查找 Python 进程
```

**后台和控制**：
```bash
&                      # 在后台运行
Ctrl+C                 # 终止当前进程
Ctrl+Z                 # 挂起当前进程
Ctrl+D                 # 结束输入/登出
Ctrl+L                 # 清屏
```

## 获取帮助和文档

### Man 手册页（Manual）

Linux 中的主要文档系统：

```bash
# 查看命令手册
man ls
man grep
man python

# 搜索 man 手册页
man -k "network"       # 按关键字搜索
apropos "copy"         # 等同于 man -k

# Man 手册页章节
man 1 passwd           # 用户命令
man 5 passwd           # 文件格式
```

**Man 手册页导航**：
- **Space**：下一页
- **b**：上一页
- **/pattern**：向前搜索
- **?pattern**：向后搜索
- **n**：下一个搜索结果
- **q**：退出

**Man 手册页结构**：
```
NAME        - 命令名和简要描述
SYNOPSIS    - 使用语法
DESCRIPTION - 详细解释
OPTIONS     - 可用的标志和选项
EXAMPLES    - 使用示例
SEE ALSO    - 相关命令
```

### 帮助选项

大多数命令有内置帮助：

```bash
# 简短帮助
ls --help
grep --help
python --help

# 通常显示：
# - 使用语法
# - 常见选项
# - 快速示例
```

### Info 页面

比 man 手册页更详细：
```bash
info ls
info bash
info coreutils
```

### 命令特定文档

```bash
# Python 帮助
python -m pydoc os

# Git 帮助
git help commit

# Systemctl 帮助
systemctl --help
```

### 在线资源

**官方文档**：
- Ubuntu 文档：https://help.ubuntu.com
- RHEL 文档：https://access.redhat.com
- Arch Wiki：https://wiki.archlinux.org（即使对其他发行版也很优秀）

**命令示例**：
- **tldr**：简化的 man 手册页，带有示例
  ```bash
  sudo apt install tldr
  tldr tar
  tldr find
  ```

- **cheat**：社区驱动的速查表
  ```bash
  pip install cheat
  cheat grep
  ```

- **explainshell.com**：基于 web 的命令解释器

**AI 基础设施特定**：
- NVIDIA CUDA 文档
- Docker 文档
- Kubernetes 文档
- TensorFlow/PyTorch 安装指南

## 环境变量

环境变量存储系统配置和路径。

### 查看环境变量

```bash
# 查看所有环境变量
env
printenv

# 查看特定变量
echo $HOME
echo $USER
echo $PATH
printenv PATH
```

### 常见环境变量

**系统变量**：
```bash
$HOME                  # 用户的主目录
$USER                  # 当前用户名
$SHELL                 # 当前 shell
$PWD                   # 当前工作目录
$OLDPWD               # 上一个目录
$HOSTNAME             # 计算机名
$LANG                 # 语言/区域设置
```

**$PATH - 命令搜索路径**：
```bash
echo $PATH
# /usr/local/bin:/usr/bin:/bin:/usr/games

# Linux 如何查找命令：
which python
# 搜索 $PATH 中的每个目录，直到找到
```

**AI/ML 特定变量**：
```bash
# CUDA（GPU 计算）
$CUDA_VISIBLE_DEVICES  # 要使用的 GPU
$CUDA_HOME             # CUDA 安装路径

# Python
$PYTHONPATH            # Python 模块搜索路径
$VIRTUAL_ENV           # 激活的虚拟环境

# TensorFlow
$TF_CPP_MIN_LOG_LEVEL # TensorFlow 日志级别
```

### 设置环境变量

**临时**（仅当前会话）：
```bash
# 设置变量
export MODEL_PATH="/models/production"
export CUDA_VISIBLE_DEVICES=0

# 使用变量
echo $MODEL_PATH
python train.py  # 可以访问 CUDA_VISIBLE_DEVICES
```

**持久**（跨会话）：
```bash
# 添加到 ~/.bashrc（用户特定）
echo 'export CUDA_HOME=/usr/local/cuda' >> ~/.bashrc
source ~/.bashrc

# 添加到 /etc/environment（系统级）
sudo sh -c 'echo "MODEL_DIR=/opt/models" >> /etc/environment'
```

### 实用 AI 示例

**GPU 选择**：
```bash
# 仅使用 GPU 0
export CUDA_VISIBLE_DEVICES=0
python train.py

# 使用 GPU 2 和 3
export CUDA_VISIBLE_DEVICES=2,3
python multi_gpu_train.py

# 仅使用 CPU（无 GPU）
export CUDA_VISIBLE_DEVICES=""
python cpu_train.py
```

**Python 路径管理**：
```bash
# 添加自定义模块目录
export PYTHONPATH="/opt/ml-project/src:$PYTHONPATH"
python -c "import custom_module"
```

**模型配置**：
```bash
# 为应用设置模型路径
export MODEL_PATH="/models/resnet50/model.h5"
export CONFIG_PATH="/config/production.yaml"
python serve.py  # 读取这些变量
```

## AI 基础设施的最佳实践

### 1. 在生产中使用绝对路径

```bash
# 不好：依赖于当前目录
python train.py --data data/train

# 好：明确且无歧义
python /opt/ml/train.py --data /data/ml/train
```

### 2. 正确设置环境

```bash
# 创建设置脚本以确保一致性
cat > ~/setup-ml-env.sh << 'EOF'
#!/bin/bash
export CUDA_HOME=/usr/local/cuda
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH
export PYTHONPATH=/opt/ml-project/src:$PYTHONPATH
EOF

# 工作前 source
source ~/setup-ml-env.sh
```

### 3. 记录你的环境

```bash
# 为可重现性保存环境
env | grep -E "(CUDA|PYTHON|MODEL)" > environment.txt

# 保存系统信息
uname -a > system_info.txt
lsb_release -a >> system_info.txt
python --version >> system_info.txt
nvidia-smi >> system_info.txt
```

### 4. 为脚本使用版本控制

```bash
# 跟踪你的自动化脚本
mkdir -p ~/ml-scripts
cd ~/ml-scripts
git init
git add setup-env.sh deploy-model.sh
git commit -m "Initial automation scripts"
```

### 5. 利用 Tab 补全

```bash
# 按 TAB 自动补全
cd /var/lo<TAB>        # 补全为 /var/log/
python train<TAB>      # 显示 train.py、train_v2.py 等

# 按两次 TAB 显示选项
ls --<TAB><TAB>        # 显示所有 -- 选项
```

## 总结和要点

### 基本概念

**Linux 基础**：
- 诞生自 Unix 哲学：简单、可组合的工具
- 开源、社区驱动
- AI/ML 基础设施的行业标准

**发行版**：
- 生产环境选择 Ubuntu 或 RHEL 家族
- LTS 版本保证稳定性
- 包管理器因家族而异

**命令行**：
- 终端 + Shell = 你的界面
- Bash 是脚本和自动化的标准
- 命令遵循：`command [options] [arguments]`

**获取帮助**：
- `man command` 获取文档
- `command --help` 获取快速参考
- 在线资源（Arch Wiki、tldr 等）

**环境变量**：
- 配置系统和应用程序
- `$PATH`、`$HOME`、`$USER` 是基本变量
- AI 特定：`$CUDA_VISIBLE_DEVICES`、`$PYTHONPATH`

### 介绍的命令

```bash
# 基本导航
pwd                    # 打印工作目录
cd                     # 更改目录
ls                     # 列出文件

# 系统信息
whoami                 # 当前用户
hostname               # 计算机名
uname -a               # 系统详情

# 获取帮助
man command            # 手册页
command --help         # 快速帮助
info command           # 详细信息

# 环境
env                    # 查看所有变量
echo $VARIABLE         # 查看特定变量
export VAR=value       # 设置变量
```

### 最佳实践

✅ 学习 Bash - 它是通用的
✅ 使用 man 手册页和帮助选项
✅ 利用 Tab 补全
✅ 一致地设置环境
✅ 记录你的配置
✅ 在生产中使用绝对路径
✅ 对脚本进行版本控制
✅ 了解你的发行版的包管理器

### AI 基础设施应用

- 通过 SSH 进行远程服务器管理
- GPU 环境配置
- Python 路径和依赖管理
- 模型部署自动化
- 系统监控和日志记录
- 分布式训练编排

### 下一步

在**讲座 02：文件系统和导航**中，你将学习：
- Linux 文件系统层级
- 高级导航技巧
- 文件操作和管理
- 快速查找文件
- 使用归档文件
- 组织 ML 项目

### 练习

1. 探索你的 Linux 系统：
   ```bash
   uname -a
   lsb_release -a
   echo $SHELL
   echo $PATH
   ```

2. 练习基本命令：
   ```bash
   pwd
   cd /tmp
   ls -la
   whoami
   hostname
   ```

3. 阅读文档：
   ```bash
   man ls
   man bash
   ls --help
   ```

4. 设置环境：
   ```bash
   export TEST_VAR="Hello Linux"
   echo $TEST_VAR
   ```

5. 探索你的系统：
   ```bash
   which python
   which bash
   echo $HOME
   cd ~
   pwd
   ```

### 快速参考卡

```bash
# 导航
pwd                    # 我在哪里？
cd <directory>         # 前往目录
ls                     # 列出文件

# 帮助
man <command>          # 完整手册
<command> --help       # 快速帮助

# 环境
echo $VAR              # 显示变量
export VAR=value       # 设置变量
env                    # 显示所有变量

# 系统信息
whoami                 # 当前用户
hostname               # 计算机名
uname -a               # 系统详情
date                   # 当前日期/时间
```

---

**讲座 01 结束**

继续学习**讲座 02：文件系统和导航**，了解 Linux 文件系统结构和 AI 基础设施工作的基本文件操作。
