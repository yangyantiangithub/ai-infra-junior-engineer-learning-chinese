# 讲座 04：系统管理基础

## 目录
1. [简介](#简介)
2. [包管理](#包管理)
3. [使用 systemd 进行服务管理](#使用-systemd-进行服务管理)
4. [进程管理](#进程管理)
5. [系统监控](#系统监控)
6. [日志文件和分析](#日志文件和分析)
7. [总结和要点](#总结和要点)

## 简介

系统管理是 AI 基础设施工程的支柱。作为 AI 基础设施工程师，你将安装 ML 框架、管理 GPU 服务、监控训练进程和调试系统问题。掌握系统管理确保你的 ML 基础设施可靠高效地运行。

本讲座涵盖基本系统管理技能：包管理、服务控制、进程管理、系统监控和日志分析 - 这些都是维护生产 AI 系统的关键。

### 学习目标

完成本讲座后，你将能够：
- 使用 apt/yum/dnf 管理系统包
- 使用 systemd 控制服务
- 有效监控和管理进程
- 跟踪系统资源使用情况（CPU、内存、磁盘）
- 分析系统日志以进行故障排查
- 将这些技能应用到 AI 基础设施场景

### 先决条件
- 讲座 01-03（Linux 基础、文件系统、权限）
- 对 Linux 系统的 root/sudo 访问权限
- 基本理解命令行操作

### 为什么 AI 需要系统管理？

**基础设施可靠性**：ML 训练作业运行数小时或数天。系统故障会浪费时间和资源。

**资源管理**：GPU、内存和存储必须受到监控和优化。

**依赖管理**：ML 框架具有必须仔细维护的复杂依赖关系。

**服务编排**：API、数据库和模型服务器必须无缝协同工作。

**故障排查**：当出现问题时（一定会出现），你需要快速诊断和修复问题。

**时长**：120 分钟
**难度**：中级

---

## 包管理

### 理解包管理

一个**包**是一个压缩归档，包含：
- 编译好的二进制文件或脚本
- 配置文件
- 文档
- 元数据（依赖、版本、描述）
- 安装/卸载脚本

**包管理器**自动化软件的安装、更新和依赖解析。

### 包管理器类型

**高级**（用户友好，处理依赖）：
- `apt` / `apt-get`（Debian、Ubuntu）
- `yum` / `dnf`（RHEL、CentOS、Fedora）
- `zypper`（openSUSE）

**低级**（手动处理依赖）：
- `dpkg`（Debian 包）
- `rpm`（Red Hat 包）

### APT - Debian/Ubuntu 包管理器

#### 基本 APT 命令

**更新包列表**：
```bash
# 从软件源下载包信息
sudo apt update

# 安装前始终运行以获取最新包信息
```

**升级包**：
```bash
# 升级所有已安装包
sudo apt upgrade

# 带智能冲突解决的升级
sudo apt full-upgrade

# 不确认的升级（用于脚本）
sudo apt upgrade -y
```

**安装包**：
```bash
# 安装单个包
sudo apt install nginx

# 安装多个包
sudo apt install python3 python3-pip python3-venv

# 安装特定版本
sudo apt install nginx=1.18.0-0ubuntu1

# 无提示安装
sudo apt install -y docker.io

# 模拟安装（空运行）
sudo apt install --dry-run cuda-11-8
```

**删除包**：
```bash
# 删除包（保留配置）
sudo apt remove nginx

# 删除包和配置
sudo apt purge nginx

# 删除包和未使用的依赖
sudo apt autoremove nginx

# 清理
sudo apt autoremove        # 删除未使用的依赖
sudo apt autoclean         # 删除旧的包文件
sudo apt clean             # 从缓存中删除所有包文件
```

**搜索包**：
```bash
# 搜索包
apt search python3

# 仅在包名中搜索
apt search --names-only cuda

# 显示包详情
apt show python3-pip

# 列出已安装包
apt list --installed
apt list --installed | grep python

# 列出可升级包
apt list --upgradable
```

**包信息**：
```bash
# 显示包详情
apt show nvidia-driver-530

# 显示包依赖
apt depends python3-pip

# 显示反向依赖（什么需要这个包）
apt rdepends python3

# 哪个包提供了文件
dpkg -S /usr/bin/python3

# 列出包中的文件
dpkg -L python3-pip
```

#### 管理软件源

软件源配置：`/etc/apt/sources.list` 和 `/etc/apt/sources.list.d/`

```bash
# 查看当前软件源
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

# 添加软件源（PPA）
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update

# 手动添加软件源
echo "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list

# 为软件源添加 GPG 密钥
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# 删除软件源
sudo add-apt-repository --remove ppa:deadsnakes/ppa
sudo rm /etc/apt/sources.list.d/docker.list
```

#### AI/ML 包安装示例

**Python 开发环境**：
```bash
sudo apt update
sudo apt install -y \
    python3 \
    python3-pip \
    python3-venv \
    python3-dev \
    build-essential \
    git \
    curl \
    wget
```

**系统 ML 依赖**：
```bash
# 线性代数库
sudo apt install -y \
    libblas-dev \
    liblapack-dev \
    libatlas-base-dev

# 用于大数据集的 HDF5
sudo apt install -y libhdf5-dev

# OpenCV 依赖
sudo apt install -y \
    libopencv-dev \
    python3-opencv

# 数据库库
sudo apt install -y \
    libpq-dev \
    libmysqlclient-dev \
    libsqlite3-dev
```

**NVIDIA CUDA 安装**：
```bash
# 添加 NVIDIA 软件源
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# 安装 CUDA 工具包
sudo apt install -y cuda-toolkit-11-8

# 安装 cuDNN
sudo apt install -y libcudnn8 libcudnn8-dev

# 验证安装
nvcc --version
```

**Docker 安装**：
```bash
# 添加 Docker 软件源
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 将用户添加到 docker 组
sudo usermod -aG docker $USER

# 验证
docker --version
```

### YUM/DNF - RHEL/CentOS 包管理器

**DNF** 是 YUM 的现代替代品（两种命令都适用）。

#### 基本 YUM/DNF 命令

**更新和升级**：
```bash
# 更新包缓存
sudo dnf check-update

# 升级所有包
sudo dnf upgrade

# 无提示升级
sudo dnf upgrade -y
```

**安装包**：
```bash
# 安装包
sudo dnf install nginx

# 安装多个包
sudo dnf install python3 python3-pip python3-devel

# 安装特定版本
sudo dnf install nginx-1.20.1

# 安装组
sudo dnf groupinstall "Development Tools"

# 重新安装包
sudo dnf reinstall nginx
```

**删除包**：
```bash
# 删除包
sudo dnf remove nginx

# 连带依赖删除
sudo dnf autoremove nginx

# 清理缓存
sudo dnf clean all
```

**搜索和信息**：
```bash
# 搜索包
dnf search python

# 显示包信息
dnf info python3-pip

# 列出已安装
dnf list installed

# 列出可用
dnf list available | grep cuda

# 查找哪个包提供了文件
dnf provides /usr/bin/python3

# 显示依赖
dnf deplist python3-pip
```

#### 管理软件源

```bash
# 列出启用的软件源
dnf repolist

# 列出所有软件源
dnf repolist all

# 启用软件源
sudo dnf config-manager --set-enabled powertools

# 禁用软件源
sudo dnf config-manager --set-disabled epel

# 添加软件源
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 安装 EPEL（企业版 Linux 额外包）
sudo dnf install -y epel-release
```

#### RHEL/CentOS 上的 AI/ML

```bash
# 开发工具
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y python3 python3-pip python3-devel

# ML 依赖
sudo dnf install -y \
    openblas-devel \
    lapack-devel \
    hdf5-devel

# NVIDIA CUDA（RHEL 8）
sudo dnf config-manager --add-repo \
    https://developer.download.nvidia.com/compute/cuda/repos/rhel8/x86_64/cuda-rhel8.repo
sudo dnf install -y cuda-toolkit-11-8

# Docker
sudo dnf config-manager --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
```

### 保持和锁定包

有时你需要防止某些包被更新。

```bash
# 保持包（防止更新）- APT
sudo apt-mark hold cuda-toolkit-11-8

# 显示已保持的包
apt-mark showhold

# 解除保持
sudo apt-mark unhold cuda-toolkit-11-8

# 为什么要保持包？
# - CUDA 版本与 ML 框架的兼容性
# - 在关键操作期间防止 Docker 更新
# - 为驱动兼容性锁定内核版本
```

---

## 使用 systemd 进行服务管理

现代 Linux 系统使用 **systemd** 作为初始化系统和服务管理器。

### 理解 systemd

**systemd** 管理：
- 系统服务（守护进程）
- 启动过程
- 设备管理
- 系统日志（journald）
- 基于定时器的激活（替代 cron）

**单元文件**：定义服务的配置文件，存储在：
- `/etc/systemd/system/` - 系统单元（最高优先级）
- `/usr/lib/systemd/system/` - 包单元（默认）
- `/run/systemd/system/` - 运行时单元

### 基本 systemctl 命令

```bash
# 服务状态
systemctl status nginx
systemctl status ml-inference

# 启动/停止服务
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx      # 不重启而是重新加载配置

# 启用/禁用开机自启
sudo systemctl enable nginx      # 开机时启动
sudo systemctl disable nginx     # 开机时不启动
sudo systemctl enable --now nginx  # 启用并立即启动

# 检查是否启用
systemctl is-enabled nginx
systemctl is-active nginx
systemctl is-failed nginx

# 列出所有服务
systemctl list-units --type=service
systemctl list-units --type=service --state=running
systemctl list-units --type=service --state=failed

# 查看服务依赖
systemctl list-dependencies nginx
```

### 创建自定义服务单元

**示例：ML 推理服务**

创建 `/etc/systemd/system/ml-inference.service`：

```ini
[Unit]
Description=ML Model Inference Service
After=network.target
Requires=postgresql.service

[Service]
Type=simple
User=mlservice
Group=mlservice
WorkingDirectory=/opt/ml-inference
Environment="PATH=/opt/ml-inference/venv/bin"
Environment="MODEL_PATH=/models/production/model.onnx"
ExecStart=/opt/ml-inference/venv/bin/python /opt/ml-inference/serve.py
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

# 资源限制
MemoryMax=8G
CPUQuota=400%

[Install]
WantedBy=multi-user.target
```

**激活服务**：
```bash
# 重新加载 systemd 以识别新服务
sudo systemctl daemon-reload

# 启动并启用
sudo systemctl start ml-inference
sudo systemctl enable ml-inference

# 检查状态
systemctl status ml-inference

# 查看日志
journalctl -u ml-inference -f
```

### 服务单元文件部分

**[Unit]** - 常规信息：
- `Description`：服务描述
- `After`：在这些单元之后启动
- `Before`：在这些单元之前启动
- `Requires`：硬依赖（如果不可用则失败）
- `Wants`：软依赖（如果不可用则继续）

**[Service]** - 服务行为：
- `Type`：simple、forking、oneshot、notify
- `User/Group`：以特定用户身份运行
- `WorkingDirectory`：工作目录
- `Environment`：环境变量
- `EnvironmentFile`：从文件加载环境变量
- `ExecStart`：启动命令
- `ExecStop`：停止命令
- `ExecReload`：重新加载命令
- `Restart`：重启策略（no、always、on-failure、on-abnormal）
- `RestartSec`：重启前等待时间
- `StandardOutput/StandardError`：输出发送到哪里

**[Install]** - 安装信息：
- `WantedBy`：要附加的目标（通常是 multi-user.target）

### systemd 中的资源限制

```ini
[Service]
# CPU 限制
CPUQuota=200%                # 最多 2 个核心
CPUWeight=100                # 相对权重（1-10000）

# 内存限制
MemoryMax=8G                 # 硬限制
MemoryHigh=6G                # 软限制（在此点节流）

# I/O 限制
IOReadBandwidthMax=/dev/sda 100M
IOWriteBandwidthMax=/dev/sda 50M

# 任务限制
TasksMax=500                 # 最大任务/线程数
```

### 示例：训练作业调度器

创建 `/etc/systemd/system/ml-training@.service`：

```ini
[Unit]
Description=ML Training Job for GPU %i
After=network.target

[Service]
Type=oneshot
User=mluser
WorkingDirectory=/opt/ml-training
Environment="CUDA_VISIBLE_DEVICES=%i"
ExecStart=/opt/ml-training/venv/bin/python train.py \
    --gpu %i \
    --config /etc/ml-training/configs/gpu%i.yaml
StandardOutput=journal
StandardError=journal
TimeoutSec=48h

[Install]
WantedBy=multi-user.target
```

**使用方式**：
```bash
# 在 GPU 0 上开始训练
sudo systemctl start ml-training@0

# 在多个 GPU 上开始训练
sudo systemctl start ml-training@{0,1,2,3}

# 检查状态
systemctl status ml-training@0
```

### 使用 journalctl 查看日志

```bash
# 查看所有日志
journalctl

# 特定服务的日志
journalctl -u ml-inference

# 实时跟踪日志
journalctl -u ml-inference -f

# 最近日志
journalctl -u ml-inference -n 100     # 最后 100 行
journalctl -u ml-inference --since "1 hour ago"
journalctl -u ml-inference --since "2024-01-15 14:00"
journalctl -u ml-inference --since today

# 优先级级别
journalctl -p err                     # 仅错误
journalctl -p warning                 # 警告及以上

# 按启动次数
journalctl -b                         # 当前启动
journalctl -b -1                      # 上次启动

# 输出格式
journalctl -u ml-inference -o json-pretty
journalctl -u ml-inference -o cat     # 无元数据

# 磁盘使用量
journalctl --disk-usage

# 清理旧日志
sudo journalctl --vacuum-time=30d     # 保留 30 天
sudo journalctl --vacuum-size=1G      # 保留 1GB
```

---

## 进程管理

### 理解进程

**进程**是正在运行的程序实例。每个进程有：
- **PID**（进程 ID）：唯一标识符
- **PPID**（父进程 ID）：创建它的进程
- **UID/GID**：用户和组所有权
- **状态**：运行、睡眠、停止、僵尸
- **内存**：代码、数据、栈、堆
- **优先级**：CPU 调度优先级

### 进程状态

**R - 运行/可运行**：正在执行或准备执行
**S - 睡眠（可中断）**：等待事件（I/O、定时器）
**D - 磁盘睡眠（不可中断）**：等待磁盘 I/O
**T - 停止**：被信号挂起（Ctrl+Z）
**Z - 僵尸**：已终止但父进程尚未读取退出状态
**I - 空闲**：内核线程

### 使用 ps 查看进程

```bash
# 简单视图（仅当前终端）
ps

# 用户的所有进程
ps -u alice
ps -u $(whoami)

# 所有进程（BSD 风格）
ps aux

# 所有进程（Unix 风格）
ps -ef

# 按 CPU 使用率排序
ps aux --sort=-%cpu | head -10

# 按内存使用率排序
ps aux --sort=-%mem | head -10

# 查找特定进程
ps aux | grep python
ps aux | grep [p]ython          # 排除 grep 本身

# 自定义输出格式
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head

# 进程树（层级）
ps auxf                         # ASCII 树
ps -ejH                         # 替代树格式
pstree                          # 专用树命令
pstree -p                       # 带 PID
```

### AI/ML 进程查询

```bash
# 查找所有 Python 进程
ps aux | grep python

# 查找 GPU 进程
ps aux | grep cuda
ps aux | grep nvidia

# 查找训练作业
ps aux | grep train

# Jupyter 笔记本进程
ps aux | grep jupyter

# 显示使用最多内存的进程
ps aux --sort=-%mem | head -20

# 显示使用最多 CPU 的进程
ps aux --sort=-%cpu | head -20
```

### 使用 top 和 htop 监控

**top** - 实时进程监控：

```bash
top

# 交互式命令：
# P - 按 CPU 使用率排序
# M - 按内存使用率排序
# T - 按运行时间排序
# k - 终止进程
# r - 调整进程优先级
# u - 显示特定用户的进程
# 1 - 切换单个 CPU 核心
# q - 退出
```

**htop** - 增强版：

```bash
# 安装 htop
sudo apt install htop        # Ubuntu/Debian
sudo yum install htop        # RHEL/CentOS

# 运行
htop

# 特性：
# - 颜色编码显示
# - 鼠标支持
# - 水平/垂直滚动
# - 可视化的 CPU 和内存仪表
# - 默认树形视图
# - 更容易的进程终止

# 功能键：
# F1 - 帮助
# F3 - 搜索进程
# F4 - 过滤进程
# F5 - 树形视图
# F6 - 按列排序
# F9 - 终止进程
# F10 - 退出
```

### 使用信号管理进程

**常见信号**：
- **SIGTERM（15）**：礼貌终止请求（默认）
- **SIGKILL（9）**：立即终止（无法捕获）
- **SIGHUP（1）**：挂起（重新加载配置）
- **SIGINT（2）**：中断（Ctrl+C）
- **SIGSTOP（19）**：暂停进程
- **SIGCONT（18）**：恢复进程

```bash
# 列出所有信号
kill -l

# 优雅终止（允许清理）
kill 1234
kill -15 1234
kill -SIGTERM 1234              # 全部等效

# 强制终止（立即，无清理）
kill -9 1234
kill -SIGKILL 1234

# 重新加载配置
kill -HUP 1234

# 暂停进程
kill -STOP 1234

# 恢复进程
kill -CONT 1234
```

### killall 和 pkill

```bash
# 按名称终止所有进程
killall python
killall -9 python               # 强制终止

# 终止特定用户的进程
killall -u alice python

# 按模式匹配终止
pkill python                    # 终止所有 python 进程
pkill -f train.py               # 终止命令中包含 'train.py' 的进程
pkill -u alice                  # 终止 alice 的所有进程
pkill -9 -f "experiment_*"      # 强制终止实验进程

# 组合条件
pkill -u alice -f python        # Alice 的 python 进程
```

### pgrep - 查找进程 ID

```bash
# 按名称查找 PID
pgrep python

# 使用完整命令行查找
pgrep -f train.py

# 获取详细信息
pgrep -l python                 # 带进程名
pgrep -a python                 # 带完整命令

# 在脚本中使用
TRAINING_PID=$(pgrep -f train.py)
if [ -n "$TRAINING_PID" ]; then
    echo "Training job running: $TRAINING_PID"
    kill $TRAINING_PID
fi
```

### 作业控制

**前台与后台**：
- **前台**：进程控制终端，接收键盘输入
- **后台**：进程独立运行，终端可用

```bash
# 使用 & 在后台启动
python train.py &
# [1] 5678                     # 作业编号和 PID

# 检查作业状态
jobs

# 挂起前台作业
# 按 Ctrl+Z

# 在后台恢复
bg

# 在前台恢复
fg

# 恢复特定作业
fg %1                           # 将作业 1 带到前台
bg %2                           # 在后台继续作业 2

# 终止特定作业
kill %1                         # 终止作业 1
```

### nohup - 不受挂断影响

```bash
# 基本 nohup
nohup python train.py &
# 输出进入 nohup.out

# 重定向输出
nohup python train.py > training.log 2>&1 &

# 替代方案：disown
python train.py &
disown                          # 从 shell 分离
```

### 进程优先级（nice/renice）

**Nice** 值控制进程优先级：
- 范围：-20（最高优先级）到 +19（最低优先级）
- 默认：0
- 较低的 nice 值 = 较高优先级 = 更多 CPU 时间
- 只有 root 可以设置负 nice 值

```bash
# 以较低优先级启动（nice 值 +10）
nice -n 10 python train.py

# 以较高优先级启动（需要 sudo）
sudo nice -n -10 python priority_job.py

# 检查 nice 值
ps -eo pid,ppid,ni,comm

# 更改运行中进程的优先级
renice +10 5678                 # 设置 nice 值为 10
sudo renice -10 5678            # 设置 nice 值为 -10（需要 sudo）

# 按用户调整优先级
sudo renice +5 -u alice

# AI 基础设施用例：
# 实验性训练（低优先级）
nice -n 15 python experimental_training.py &

# 生产推理（高优先级）
sudo nice -n -5 python production_inference.py &
```

---

## 系统监控

### CPU 监控

**top/htop** - 实时监控：
```bash
# 查看 CPU 使用率
top
htop

# 检查负载平均值
uptime
# 14:32:15 up 5 days, 2:15, 3 users, load average: 4.52, 3.87, 2.91

# 负载平均值解释：
# 4.52 = 1 分钟平均值
# 3.87 = 5 分钟平均值
# 2.91 = 15 分钟平均值
# 值 > CPU 核心数 = 系统过载
```

**mpstat** - 详细 CPU 统计：
```bash
# 安装 sysstat 包
sudo apt install sysstat

# 显示 CPU 使用率
mpstat 1 5                      # 1 秒间隔，5 次报告

# 显示所有 CPU
mpstat -P ALL 1
```

**示例监控脚本**：
```bash
#!/bin/bash
# 训练期间监控 CPU

while true; do
    echo "=== $(date) ==="
    mpstat 1 1 | tail -1
    sleep 60
done
```

### 内存监控

**free** - 内存使用情况：
```bash
# 显示内存使用情况
free -h                         # 人类可读

# 输出：
#               total        used        free      shared  buff/cache   available
# Mem:           62Gi        30Gi       8.0Gi       2.0Gi        24Gi        28Gi
# Swap:         8.0Gi       0.0Gi       8.0Gi

# 每列的含义：
# total：物理 RAM 总量
# used：程序使用的 RAM
# free：完全未使用的 RAM
# shared：tmpfs 使用的 RAM
# buff/cache：用于磁盘缓存的 RAM（需要时可以释放）
# available：新程序可用的 RAM（空闲 + 可回收缓存）
```

**vmstat** - 虚拟内存统计：
```bash
# 显示内存和 CPU 统计
vmstat 1 5                      # 1 秒间隔，5 次报告

# 输出列：
# r：运行中的进程
# b：阻塞的进程
# swpd：使用的虚拟内存
# free：空闲内存
# buff：缓冲区内存
# cache：缓存内存
# si：从磁盘换入的内存
# so：换出到磁盘的内存
```

**监控 OOM（内存不足）事件**：
```bash
# 检查 OOM killer 活动
dmesg | grep -i "out of memory"
dmesg | grep -i "oom"

# 查看进程的 OOM 评分（越高 = 越可能被杀死）
cat /proc/1234/oom_score
```

### 磁盘监控

**df** - 磁盘空间使用情况：
```bash
# 显示磁盘使用情况
df -h                           # 人类可读

# 显示特定文件系统
df -h /data

# 显示 inode
df -i                           # Inode 使用情况

# 示例输出：
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1       100G   45G   50G  48% /
# /dev/sdb1       1.0T  800G  200G  80% /data
```

**du** - 目录磁盘使用情况：
```bash
# 显示目录大小
du -sh /data/models

# 显示子目录大小
du -h --max-depth=1 /data

# 查找最大目录
du -h /data | sort -rh | head -20

# 查找大文件
find /data -type f -size +1G -exec ls -lh {} \;
```

**iostat** - I/O 统计：
```bash
# 安装 sysstat
sudo apt install sysstat

# 显示 I/O 统计
iostat -x 1 5                   # 扩展统计，1s 间隔

# 关键指标：
# %util：设备忙碌时间百分比
# r/s：每秒读取次数
# w/s：每秒写入次数
# rkB/s：每秒读取 KB
# wkB/s：每秒写入 KB
```

**iotop** - 实时 I/O 监控：
```bash
# 安装 iotop
sudo apt install iotop

# 运行 iotop
sudo iotop

# 仅显示活动 I/O
sudo iotop -o
```

### GPU 监控

**nvidia-smi** - NVIDIA GPU 监控：
```bash
# 显示 GPU 状态
nvidia-smi

# 监视 GPU 使用情况（每秒更新）
watch -n 1 nvidia-smi

# 查询特定信息
nvidia-smi --query-gpu=index,name,temperature.gpu,utilization.gpu,memory.used,memory.total \
    --format=csv

# 查询计算进程
nvidia-smi --query-compute-apps=pid,name,used_memory --format=csv

# 查找哪个进程正在使用 GPU
nvidia-smi pmon

# 将 GPU 指标记录到文件
while true; do
    nvidia-smi --query-gpu=timestamp,temperature.gpu,utilization.gpu,memory.used \
        --format=csv >> gpu_metrics.log
    sleep 60
done
```

### 综合系统监控脚本

```bash
#!/bin/bash
# system-health.sh - 监控系统资源

LOG_FILE="/var/log/system-health.log"

echo "=== System Health Report $(date) ===" | tee -a "$LOG_FILE"

# CPU
echo "" | tee -a "$LOG_FILE"
echo "--- CPU ---" | tee -a "$LOG_FILE"
uptime | tee -a "$LOG_FILE"
mpstat 1 1 | tail -1 | tee -a "$LOG_FILE"

# 内存
echo "" | tee -a "$LOG_FILE"
echo "--- Memory ---" | tee -a "$LOG_FILE"
free -h | tee -a "$LOG_FILE"

# 磁盘
echo "" | tee -a "$LOG_FILE"
echo "--- Disk ---" | tee -a "$LOG_FILE"
df -h | grep -v tmpfs | grep -v loop | tee -a "$LOG_FILE"

# CPU 消耗最高的进程
echo "" | tee -a "$LOG_FILE"
echo "--- Top CPU Processes ---" | tee -a "$LOG_FILE"
ps aux --sort=-%cpu | head -6 | tee -a "$LOG_FILE"

# 内存消耗最高的进程
echo "" | tee -a "$LOG_FILE"
echo "--- Top Memory Processes ---" | tee -a "$LOG_FILE"
ps aux --sort=-%mem | head -6 | tee -a "$LOG_FILE"

# GPU（如果可用）
if command -v nvidia-smi &> /dev/null; then
    echo "" | tee -a "$LOG_FILE"
    echo "--- GPU ---" | tee -a "$LOG_FILE"
    nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total \
        --format=csv | tee -a "$LOG_FILE"
fi

echo "" | tee -a "$LOG_FILE"
```

---

## 日志文件和分析

### 理解 Linux 日志

**系统日志位置**：`/var/log/`

**常见日志文件**：
- `/var/log/syslog` - 通用系统日志（Debian/Ubuntu）
- `/var/log/messages` - 通用系统日志（RHEL/CentOS）
- `/var/log/auth.log` - 认证日志
- `/var/log/kern.log` - 内核日志
- `/var/log/dmesg` - 启动消息
- `/var/log/nginx/` - Nginx web 服务器日志
- `/var/log/apache2/` - Apache web 服务器日志

### 查看日志文件

```bash
# 查看整个日志
cat /var/log/syslog

# 查看最后几行
tail /var/log/syslog

# 实时跟踪日志
tail -f /var/log/syslog

# 使用分页器查看
less /var/log/syslog

# 查看特定行数
tail -n 100 /var/log/syslog
head -n 50 /var/log/syslog
```

### 搜索日志

```bash
# 搜索错误
grep "ERROR" /var/log/syslog
grep -i "error" /var/log/syslog        # 不区分大小写

# 搜索失败认证
grep "Failed password" /var/log/auth.log

# 搜索多个文件
grep "cuda" /var/log/*.log

# 显示匹配周围上下文
grep -A 5 "ERROR" /var/log/syslog      # 之后 5 行
grep -B 5 "ERROR" /var/log/syslog      # 之前 5 行
grep -C 5 "ERROR" /var/log/syslog      # 前后各 5 行

# 计数出现次数
grep -c "ERROR" /var/log/syslog

# 带行号搜索
grep -n "ERROR" /var/log/syslog
```

### journalctl - systemd 日志

现代系统使用 systemd 的日志进行日志记录。

```bash
# 查看所有日志
journalctl

# 查看特定服务的日志
journalctl -u nginx
journalctl -u ml-inference

# 实时跟踪日志
journalctl -f
journalctl -u ml-inference -f

# 查看特定时间以来的日志
journalctl --since "2024-01-15 14:00"
journalctl --since "1 hour ago"
journalctl --since today
journalctl --since yesterday

# 查看时间之间的日志
journalctl --since "2024-01-15 14:00" --until "2024-01-15 15:00"

# 按优先级过滤
journalctl -p err                      # 仅错误
journalctl -p warning                  # 警告及以上
# 优先级：emerg、alert、crit、err、warning、notice、info、debug

# 查看内核消息
journalctl -k
journalctl --dmesg

# 查看特定启动的日志
journalctl -b                          # 当前启动
journalctl -b -1                       # 上次启动
journalctl --list-boots                # 列出所有启动

# 反向顺序（最新的在前）
journalctl -r

# 限制输出
journalctl -n 50                       # 最后 50 行
journalctl -u nginx -n 100             # nginx 的最后 100 行

# 输出格式
journalctl -o json-pretty              # JSON 格式
journalctl -o cat                      # 仅消息，无元数据
journalctl -o short-iso                # ISO 时间戳

# 磁盘使用量
journalctl --disk-usage

# 清理旧日志
sudo journalctl --vacuum-time=2weeks   # 保留 2 周
sudo journalctl --vacuum-size=500M     # 保留 500MB
```

### 日志轮转

**logrotate** 自动轮转、压缩和删除旧日志。

配置：`/etc/logrotate.conf` 和 `/etc/logrotate.d/`

**示例配置**（`/etc/logrotate.d/ml-training`）：
```
/var/log/ml-training/*.log {
    daily                      # 每日轮转
    rotate 7                   # 保留 7 天
    compress                   # 压缩旧日志
    delaycompress              # 延迟压缩 1 次轮转
    missingok                  # 日志缺失时不报错
    notifempty                 # 为空时不轮转
    create 0640 mluser mlgroup # 以这些权限创建新日志
    sharedscripts              # 为所有日志运行一次脚本
    postrotate
        systemctl reload ml-training > /dev/null 2>&1 || true
    endscript
}
```

**测试 logrotate**：
```bash
# 测试配置
sudo logrotate -d /etc/logrotate.d/ml-training

# 强制轮转
sudo logrotate -f /etc/logrotate.d/ml-training
```

### AI 基础设施日志分析示例

**查找训练错误**：
```bash
# 搜索 CUDA 错误
journalctl -u ml-training | grep -i "cuda"

# 搜索内存不足错误
journalctl -u ml-training | grep -i "out of memory"
dmesg | grep -i "oom"

# 查找失败的模型加载
grep "Failed to load model" /var/log/ml-inference/*.log

# 检查 GPU 错误
nvidia-smi --query-gpu=index,name,ecc.errors.corrected.aggregate.total \
    --format=csv
```

**分析训练性能**：
```bash
# 从训练日志中提取 epoch 时间
grep "Epoch" /var/log/ml-training/training.log | \
    awk '{print $NF}' | \
    awk '{sum+=$1; count++} END {print "Avg:", sum/count}'

# 查找最长的推理时间
grep "inference_time" /var/log/ml-api/api.log | \
    sort -k5 -n -r | head -20

# 统计每小时的 API 请求数
grep "$(date +%Y-%m-%d)" /var/log/nginx/access.log | \
    cut -d: -f2 | sort | uniq -c
```

**安全监控**：
```bash
# 失败的 SSH 尝试
grep "Failed password" /var/log/auth.log | \
    awk '{print $(NF-5)}' | sort | uniq -c | sort -rn

# 成功的 root 登录
grep "session opened for user root" /var/log/auth.log

# Sudo 使用情况
grep "sudo" /var/log/auth.log | tail -20
```

---

## 总结和要点

### 掌握的命令

**包管理**：
- `apt update` / `apt upgrade` - 更新和升级包
- `apt install` / `apt remove` - 安装和删除包
- `apt search` / `apt show` - 搜索和显示包信息
- `dnf install` / `dnf remove` - RHEL/CentOS 包管理

**服务管理**：
- `systemctl start/stop/restart` - 控制服务
- `systemctl enable/disable` - 配置启动行为
- `systemctl status` - 检查服务状态
- `journalctl -u service` - 查看服务日志

**进程管理**：
- `ps aux` - 查看所有进程
- `top` / `htop` - 交互式监控
- `kill` / `pkill` / `killall` - 终止进程
- `nice` / `renice` - 控制进程优先级
- `pgrep` - 查找进程 ID

**系统监控**：
- `free -h` - 内存使用情况
- `df -h` - 磁盘使用情况
- `du -sh` - 目录大小
- `uptime` - 负载平均值
- `nvidia-smi` - GPU 监控

**日志分析**：
- `journalctl` - systemd 日志
- `tail -f` - 跟踪日志文件
- `grep` - 搜索日志
- `logrotate` - 自动日志轮转

### 关键概念

1. **包管理**：使用 apt/dnf 自动化软件安装
2. **服务控制**：使用 systemd 管理长时间运行的服务
3. **进程监控**：跟踪和控制运行中的程序
4. **资源跟踪**：监控 CPU、内存、磁盘和 GPU 使用情况
5. **日志分析**：通过系统日志排查问题
6. **自动化**：用于计划任务的 systemd timers 和 cron

### AI 基础设施最佳实践

✅ **锁定依赖版本**以实现可重现性
✅ **使用 systemd 服务**用于生产 ML 应用程序
✅ **设置资源限制**以防止资源耗尽
✅ **持续监控 GPU 使用情况**在训练期间
✅ **实施日志轮转**以管理磁盘空间
✅ **使用 nice 值**来优先处理生产工作负载
✅ **定期系统更新**以获得安全补丁
✅ **使用 systemd timers**自动化健康检查

### 故障排查检查清单

当出现问题时，检查：
1. 服务状态：`systemctl status service-name`
2. 服务日志：`journalctl -u service-name -n 100`
3. 进程列表：`ps aux | grep service`
4. 资源使用情况：`top`、`free -h`、`df -h`
5. 系统日志：`journalctl -p err --since "1 hour ago"`
6. GPU 状态：`nvidia-smi`（如适用）
7. 磁盘空间：`df -h`、`du -sh /var/log`

### 常见问题和解决方案

**服务无法启动**：
```bash
systemctl status ml-inference
journalctl -u ml-inference -n 50
# 检查：权限、缺失文件、端口冲突
```

**CPU 使用率高**：
```bash
top -o %CPU
# 识别进程，检查是否预期，必要时调整优先级
```

**内存不足**：
```bash
free -h
dmesg | grep -i oom
# 检查内存泄漏，减小批处理大小，添加 swap
```

**磁盘已满**：
```bash
df -h
du -h / | sort -rh | head -20
# 清理日志，删除旧模型，压缩数据
```

### 下一步

在**讲座 05：Shell 脚本入门**中，你将学习：
- 编写用于自动化的 bash 脚本
- 变量、循环和条件语句
- 函数和参数处理
- 错误处理基础
- 自动化系统管理任务

### 快速参考卡

```bash
# 包管理
sudo apt update && sudo apt upgrade -y
sudo apt install package-name
apt search keyword

# 服务管理
sudo systemctl restart service-name
systemctl status service-name
journalctl -u service-name -f

# 进程管理
ps aux --sort=-%cpu | head
kill -9 PID
nice -n 10 command

# 监控
top
htop
free -h
df -h
nvidia-smi

# 日志
journalctl -u service-name -f
tail -f /var/log/syslog
grep ERROR /var/log/syslog
```

---

**讲座 04：系统管理基础结束**

你现在已经掌握了管理 AI 基础设施所需的基础系统管理技能。定期练习这些命令以建立肌肉记忆和信心。

**下一步**：讲座 05 - Shell 脚本入门
