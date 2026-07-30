# 讲座 05：Shell 脚本入门

## 目录
1. [简介](#简介)
2. [Bash 脚本基础](#bash-脚本基础)
3. [变量和数据类型](#变量和数据类型)
4. [基本控制结构](#基本控制结构)
5. [简单函数](#简单函数)
6. [命令行参数](#命令行参数)
7. [基本错误处理](#基本错误处理)
8. [实际示例](#实际示例)
9. [总结和要点](#总结和要点)

## 简介

Shell 脚本是 AI 基础设施工程师实现自动化的超能力。无论是部署模型、管理数据集还是编排训练管道，shell 脚本都将重复的手动任务转变为可靠、可重复的流程。

本讲座介绍 Bash 脚本基础。你将学习编写简单但强大的脚本来自动化常见的 AI 基础设施任务。

### 学习目标

完成本讲座后，你将能够：
- 编写和执行基本 Bash 脚本
- 使用变量和命令替换
- 实现简单的控制结构（if/else、for/while 循环）
- 创建可重用的函数
- 处理命令行参数
- 处理基本错误
- 将脚本应用于 AI 基础设施自动化

### 先决条件
- 讲座 01-04（从 Linux 基础到系统管理）
- 熟悉命令行基础
- 熟悉文本编辑器（nano、vim 或 VS Code）

### 为什么 AI 基础设施需要 Shell 脚本？

**自动化**：部署模型、预置环境、编排训练
**一致性**：每次相同流程，无手动错误
**效率**：自动化重复性任务，每天节省数小时
**集成**：将不同工具和服务粘合在一起
**文档**：脚本作为可执行的文档

**时长**：90 分钟
**难度**：初级到中级

---

## Bash 脚本基础

### 创建你的第一个脚本

```bash
#!/bin/bash
# hello.sh - 我的第一个脚本

echo "Hello, AI Infrastructure!"
echo "Current directory: $(pwd)"
echo "Current user: $(whoami)"
echo "Date: $(date)"
```

### Shebang（#!）

第一行告诉系统使用哪个解释器：

```bash
#!/bin/bash              # 使用 bash
#!/bin/sh                # 使用 sh（POSIX 兼容）
#!/usr/bin/env bash      # 使用 bash（可移植 - 在 PATH 中查找 bash）
#!/usr/bin/env python3   # 使用 python3
```

**最佳实践**：bash 特定特性使用 `#!/bin/bash`，最大可移植性使用 `#!/bin/sh`。

### 使脚本可执行

```bash
# 创建脚本
cat > hello.sh << 'EOF'
#!/bin/bash
echo "Hello, World!"
EOF

# 使其可执行
chmod +x hello.sh

# 运行
./hello.sh
# 输出：Hello, World!

# 或显式用 bash 运行（不需要执行权限）
bash hello.sh
```

**重要**：脚本必须可执行（`chmod +x`）才能使用 `./script.sh` 语法运行。

### 脚本结构

一个组织良好的脚本遵循以下结构：

```bash
#!/bin/bash

#############################################
# 脚本名称：setup-ml-env.sh
# 描述：设置 ML 开发环境
# 作者：你的名字
# 日期：2024-10-28
# 版本：1.0
#############################################

# 全局变量
MODEL_PATH="/models/production"
LOG_FILE="/var/log/setup.log"

# 函数
function log_message() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

function setup_environment() {
    log_message "Setting up environment..."
    # 设置逻辑在此
}

# 主执行
main() {
    log_message "Starting setup"
    setup_environment
    log_message "Setup complete"
}

# 运行主函数
main "$@"
```

### 注释

```bash
# 单行注释
# 这一行被 shell 忽略

echo "Output"  # 行内注释

: '
多行注释
这些引号之间的所有内容都被忽略
适用于记录较大的代码块
'

# TODO：添加错误处理
# FIXME：这需要优化
# NOTE：重要信息
```

---

## 变量和数据类型

### 变量赋值

**基本赋值**（= 两边不能有空格）：

```bash
# 正确
NAME="Alice"
AGE=25
MODEL_PATH="/models/resnet50.h5"

# 错误（会导致错误）
NAME = "Alice"      # = 两边不允许有空格
AGE =25             # = 前不允许有空格
```

**使用变量**：

```bash
# 使用 $ 引用
echo $NAME
echo "My name is $NAME"

# 更好：使用大括号（推荐）
echo "Model: ${MODEL_PATH}"
echo "Age next year: $((AGE + 1))"
```

### 命令替换

将命令输出捕获到变量中：

```bash
# 现代语法（推荐）
CURRENT_DATE=$(date +%Y-%m-%d)
NUM_FILES=$(ls | wc -l)
GPU_COUNT=$(nvidia-smi --list-gpus | wc -l)

# 旧语法（反引号 - 避免）
HOSTNAME=`hostname`

# 在字符串中使用
echo "Today is ${CURRENT_DATE}"
echo "Found ${NUM_FILES} files"
```

### 算术运算

```bash
# 算术扩展
COUNT=5
COUNT=$((COUNT + 1))        # COUNT 现在是 6
TOTAL=$((5 * 10))           # TOTAL 是 50
RESULT=$((100 / 3))         # RESULT 是 33（整除）

# 常见运算
X=$((10 + 5))               # 加法
X=$((10 - 5))               # 减法
X=$((10 * 5))               # 乘法
X=$((10 / 5))               # 除法
X=$((10 % 3))               # 模运算（余数）

# 递增/递减
COUNT=$((COUNT + 1))        # 递增
COUNT=$((COUNT - 1))        # 递减

# 实际应用
BATCH_SIZE=32
NUM_BATCHES=$((1000 / BATCH_SIZE))
echo "Will process ${NUM_BATCHES} batches"
```

### 字符串操作

```bash
# 字符串长度
NAME="Alice"
echo ${#NAME}               # 5

# 子串提取
TEXT="Hello, World!"
echo ${TEXT:0:5}            # Hello（从 0 开始，长度 5）
echo ${TEXT:7}              # World!（从 7 开始，到结尾）

# 字符串替换
FILENAME="model_v1.h5"
echo ${FILENAME/v1/v2}      # model_v2.h5（替换第一个）
echo ${FILENAME//v1/v2}     # 替换所有

# 更改扩展名
echo ${FILENAME/.h5/.pt}    # model_v1.pt

# 默认值
echo ${VAR:-default}        # 如果 VAR 未设置，使用 'default'
VAR=${VAR:-default}         # 如果 VAR 未设置，将其设置为 'default'

# 示例：带默认值的配置
MODEL_NAME=${MODEL_NAME:-resnet50}
BATCH_SIZE=${BATCH_SIZE:-32}
echo "Training ${MODEL_NAME} with batch size ${BATCH_SIZE}"
```

### 引用规则

**单引号** - 字面字符串（无变量扩展）：
```bash
NAME="Alice"
echo 'Hello $NAME'          # 输出：Hello $NAME
```

**双引号** - 允许变量扩展：
```bash
NAME="Alice"
echo "Hello $NAME"          # 输出：Hello Alice
```

**无引号** - 单词分割和通配：
```bash
FILES=$(ls *.txt)
echo $FILES                 # 按空格分割单词
echo "$FILES"               # 保留换行符
```

**最佳实践**：始终引用变量，除非你特别需要单词分割。

```bash
# 好
echo "Processing file: $FILENAME"

# 不好（文件名中有空格时会出错）
echo Processing file: $FILENAME

# 好（保留空格和特殊字符）
cp "$SOURCE" "$DEST"

# 不好（有空格时会出错）
cp $SOURCE $DEST
```

### 变量作用域

```bash
# 全局变量
GLOBAL_VAR="I am global"

# 带局部变量的函数
function test_scope() {
    local LOCAL_VAR="I am local"
    echo "Inside function: $GLOBAL_VAR"
    echo "Inside function: $LOCAL_VAR"
}

test_scope
echo "Outside function: $GLOBAL_VAR"
echo "Outside function: $LOCAL_VAR"  # 空 - LOCAL_VAR 不可访问
```

---

## 基本控制结构

### If 语句

**基本 if**：
```bash
if [ -f "model.h5" ]; then
    echo "Model file exists"
fi
```

**If-else**：
```bash
if [ -d "/data/imagenet" ]; then
    echo "Dataset found"
else
    echo "Dataset not found"
    exit 1
fi
```

**If-elif-else**：
```bash
GPU_COUNT=2

if [ $GPU_COUNT -eq 0 ]; then
    echo "No GPUs available"
elif [ $GPU_COUNT -eq 1 ]; then
    echo "One GPU available"
else
    echo "$GPU_COUNT GPUs available"
fi
```

### 测试操作符

**文件测试**：
```bash
[ -e file ]         # 存在
[ -f file ]         # 是普通文件
[ -d dir ]          # 是目录
[ -L link ]         # 是符号链接
[ -r file ]         # 可读
[ -w file ]         # 可写
[ -x file ]         # 可执行
[ -s file ]         # 文件大小 > 0

# 示例
if [ -f "config.yaml" ]; then
    echo "Config file found"
fi

if [ ! -d "logs" ]; then
    mkdir logs
fi
```

**字符串测试**：
```bash
[ -z "$str" ]       # 字符串为空
[ -n "$str" ]       # 字符串不为空
[ "$s1" = "$s2" ]   # 字符串相等
[ "$s1" != "$s2" ]  # 字符串不相等

# 示例
if [ -z "$MODEL_NAME" ]; then
    echo "Error: MODEL_NAME is not set"
    exit 1
fi

if [ "$ENV" = "production" ]; then
    echo "Running in production mode"
fi
```

**数字测试**：
```bash
[ $a -eq $b ]       # 等于
[ $a -ne $b ]       # 不等于
[ $a -lt $b ]       # 小于
[ $a -le $b ]       # 小于或等于
[ $a -gt $b ]       # 大于
[ $a -ge $b ]       # 大于或等于

# 示例
EPOCHS=100
if [ $EPOCHS -gt 50 ]; then
    echo "Long training job"
fi

if [ $GPU_COUNT -ge 2 ]; then
    echo "Can run distributed training"
fi
```

**组合条件**：
```bash
# AND
if [ -f "model.h5" ] && [ -f "config.yaml" ]; then
    echo "Both files found"
fi

# OR
if [ ! -f "model.h5" ] || [ ! -f "config.yaml" ]; then
    echo "Missing required files"
fi

# 使用 [[ ]] 的复杂条件
if [[ -f "model.h5" && -f "config.yaml" ]]; then
    echo "Both files found"
fi
```

### For 循环

**遍历列表**：
```bash
# 简单列表
for model in resnet50 vgg16 inception; do
    echo "Processing $model"
done

# 遍历数组
MODELS=("resnet50" "vgg16" "inception")
for model in "${MODELS[@]}"; do
    echo "Training $model"
    python train.py --model "$model"
done
```

**遍历文件**：
```bash
# 处理所有 .h5 文件
for file in *.h5; do
    echo "Found model: $file"
done

# 处理目录中的文件
for file in /data/images/*.jpg; do
    echo "Processing image: $file"
done
```

**数字范围**：
```bash
# 使用大括号扩展的范围
for i in {1..5}; do
    echo "Iteration $i"
done

# 带步长的范围
for i in {0..10..2}; do
    echo "Even number: $i"
done

# C 风格 for 循环
for ((i=0; i<10; i++)); do
    echo "Count: $i"
done
```

**实际示例**：
```bash
# 在不同 GPU 上训练模型
MODELS=("resnet50" "vgg16" "mobilenet")
GPU=0

for model in "${MODELS[@]}"; do
    echo "Training $model on GPU $GPU"
    CUDA_VISIBLE_DEVICES=$GPU python train.py --model "$model" &
    GPU=$((GPU + 1))
done
wait  # 等待所有后台作业

# 处理数据集批次
for batch in {1..100}; do
    echo "Processing batch $batch"
    python process.py --batch $batch
done
```

### While 循环

**基本 while**：
```bash
COUNT=0
while [ $COUNT -lt 5 ]; do
    echo "Count: $COUNT"
    COUNT=$((COUNT + 1))
done
```

**逐行读取文件**：
```bash
while IFS= read -r line; do
    echo "Line: $line"
done < file.txt

# 处理 CSV 文件
while IFS=, read -r name age city; do
    echo "Name: $name, Age: $age, City: $city"
done < data.csv
```

**带 break 的无限循环**：
```bash
while true; do
    echo "Checking system health..."

    # 检查服务是否在运行
    if ! systemctl is-active ml-api; then
        echo "Service is down! Restarting..."
        systemctl restart ml-api
    fi

    sleep 60  # 等待 60 秒
done
```

**Until 循环**（一直运行直到条件为真）：
```bash
COUNT=0
until [ $COUNT -ge 5 ]; do
    echo "Count: $COUNT"
    COUNT=$((COUNT + 1))
done
```

### Break 和 Continue

```bash
# Break - 退出循环
for i in {1..10}; do
    if [ $i -eq 5 ]; then
        break  # 在第 5 次停止循环
    fi
    echo $i
done
# 输出：1 2 3 4

# Continue - 跳到下一次迭代
for i in {1..10}; do
    if [ $i -eq 5 ]; then
        continue  # 跳过 5
    fi
    echo $i
done
# 输出：1 2 3 4 6 7 8 9 10
```

---

## 简单函数

### 定义函数

**方法 1**（带 function 关键字）：
```bash
function greet() {
    echo "Hello, $1!"
}

greet "Alice"
# 输出：Hello, Alice!
```

**方法 2**（POSIX 风格）：
```bash
greet() {
    echo "Hello, $1!"
}

greet "Bob"
# 输出：Hello, Bob!
```

### 函数参数

函数通过特殊变量访问参数：
- `$1`、`$2`、`$3`、... - 各个参数
- `$@` - 所有参数作为独立单词
- `$#` - 参数数量

```bash
function train_model() {
    local model_name=$1
    local epochs=$2
    local gpu_id=${3:-0}  # 如果未提供，默认为 0

    echo "Training $model_name"
    echo "Epochs: $epochs"
    echo "GPU: $gpu_id"

    python train.py --model "$model_name" --epochs "$epochs" --gpu "$gpu_id"
}

# 调用函数
train_model "resnet50" 100 2
```

### 返回值

**退出码**（0-255）：
```bash
function check_file() {
    local file=$1

    if [ -f "$file" ]; then
        return 0  # 成功
    else
        return 1  # 失败
    fi
}

# 使用返回值
if check_file "model.h5"; then
    echo "File exists"
else
    echo "File not found"
fi
```

**通过 echo 返回字符串**：
```bash
function get_model_name() {
    local path=$1
    basename "$path" .h5
}

MODEL_NAME=$(get_model_name "/models/resnet50.h5")
echo $MODEL_NAME  # resnet50
```

### 局部变量

使用 `local` 使变量限制在函数作用域内：

```bash
GLOBAL_VAR="I am global"

function test_local() {
    local LOCAL_VAR="I am local"
    GLOBAL_VAR="Modified global"

    echo "Inside function:"
    echo "  GLOBAL_VAR: $GLOBAL_VAR"
    echo "  LOCAL_VAR: $LOCAL_VAR"
}

test_local

echo "Outside function:"
echo "  GLOBAL_VAR: $GLOBAL_VAR"    # 修改后的全局
echo "  LOCAL_VAR: $LOCAL_VAR"      # 空 - 不可访问
```

### 实际函数示例

**日志函数**：
```bash
function log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $@"
}

log "Starting training"
log "Epoch 1 complete"
log "Training finished"
```

**错误处理函数**：
```bash
function die() {
    echo "ERROR: $@" >&2
    exit 1
}

# 用法
if [ ! -f "config.yaml" ]; then
    die "Config file not found"
fi
```

**检查命令是否存在**：
```bash
function command_exists() {
    command -v "$1" &> /dev/null
}

if ! command_exists nvidia-smi; then
    echo "nvidia-smi not found. Please install NVIDIA drivers."
    exit 1
fi
```

---

## 命令行参数

### 位置参数

脚本通过特殊变量访问参数：

```bash
#!/bin/bash
# script.sh

echo "Script name: $0"
echo "First argument: $1"
echo "Second argument: $2"
echo "All arguments: $@"
echo "Number of arguments: $#"

# 示例用法：
# ./script.sh model.h5 config.yaml
# 输出：
# Script name: ./script.sh
# First argument: model.h5
# Second argument: config.yaml
# All arguments: model.h5 config.yaml
# Number of arguments: 2
```

### 处理参数

**基本参数处理**：
```bash
#!/bin/bash
# train.sh

if [ $# -lt 2 ]; then
    echo "Usage: $0 <model_name> <epochs>"
    exit 1
fi

MODEL_NAME=$1
EPOCHS=$2
GPU=${3:-0}  # 可选的第三个参数，默认为 0

echo "Training $MODEL_NAME for $EPOCHS epochs on GPU $GPU"
python train.py --model "$MODEL_NAME" --epochs "$EPOCHS" --gpu "$GPU"
```

### 使用 getopts 进行简单选项解析

`getopts` 是一个用于解析选项的内置命令：

```bash
#!/bin/bash
# train.sh - 带选项的训练脚本

usage() {
    echo "Usage: $0 [-m MODEL] [-e EPOCHS] [-g GPU] [-h]"
    echo "  -m MODEL    Model architecture (default: resnet50)"
    echo "  -e EPOCHS   Number of epochs (default: 100)"
    echo "  -g GPU      GPU ID (default: 0)"
    echo "  -h          Show this help"
    exit 1
}

# 默认值
MODEL="resnet50"
EPOCHS=100
GPU=0

# 解析选项
while getopts "m:e:g:h" opt; do
    case $opt in
        m) MODEL="$OPTARG" ;;
        e) EPOCHS="$OPTARG" ;;
        g) GPU="$OPTARG" ;;
        h) usage ;;
        *) usage ;;
    esac
done

# 跳过已处理的选项
shift $((OPTIND - 1))

echo "Model: $MODEL"
echo "Epochs: $EPOCHS"
echo "GPU: $GPU"
echo "Remaining args: $@"

# 运行训练
python train.py --model "$MODEL" --epochs "$EPOCHS" --gpu "$GPU"

# 示例用法：
# ./train.sh -m vgg16 -e 50 -g 2
# ./train.sh -h
```

---

## 基本错误处理

### 退出码

每个命令返回一个退出码：
- `0` = 成功
- 非零 = 错误

```bash
# 检查上一条命令的退出码
ls /tmp
echo $?   # 0（成功）

ls /nonexistent
echo $?   # 非零（错误）

# 在条件语句中使用
if ls /data; then
    echo "Directory exists"
else
    echo "Directory not found"
fi
```

### 出错时退出

使用 `set -e` 在任何错误时退出脚本：

```bash
#!/bin/bash
set -e  # 出错时退出

echo "Step 1"
command1

echo "Step 2"
command2  # 如果此命令失败，脚本立即退出

echo "Step 3"
command3  # 如果 command2 失败，此行不会运行
```

### 未定义变量时退出

使用 `set -u` 在未定义变量时退出：

```bash
#!/bin/bash
set -u  # 未定义变量时退出

echo $UNDEFINED_VAR  # 脚本退出并报错
```

### 组合选项

```bash
#!/bin/bash
# 严格模式
set -euo pipefail

# -e：出错时退出
# -u：未定义变量时退出
# -o pipefail：如果管道中任何命令失败则退出

# 这更安全：
command1 | command2 | command3
# 如果 command2 失败，脚本退出（由于 pipefail）
```

### 临时禁用错误退出

```bash
#!/bin/bash
set -e

# 此命令可能失败，没关系
set +e
optional_command_that_might_fail
EXIT_CODE=$?
set -e

if [ $EXIT_CODE -ne 0 ]; then
    echo "Optional command failed, but continuing"
fi

# 继续脚本
```

### 错误消息

**打印到 stderr**：
```bash
# 使用 >&2 重定向到 stderr
echo "ERROR: File not found" >&2

# 更好：创建错误函数
function error() {
    echo "ERROR: $@" >&2
}

error "Configuration file missing"
```

---

## 实际示例

### 示例 1：环境搭建脚本

```bash
#!/bin/bash
set -euo pipefail

# setup-ml-env.sh - 设置 ML 开发环境

# 颜色输出
RED='\033[0;31m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

function log() {
    echo -e "${GREEN}[INFO]${NC} $@"
}

function error() {
    echo -e "${RED}[ERROR]${NC} $@" >&2
}

function check_command() {
    if ! command -v "$1" &> /dev/null; then
        error "$1 not found"
        return 1
    fi
    return 0
}

# 主要设置
log "Setting up ML environment..."

# 检查先决条件
log "Checking prerequisites..."
check_command python3 || exit 1
check_command pip || exit 1

# 创建虚拟环境
log "Creating virtual environment..."
python3 -m venv ml-env

# 激活虚拟环境
source ml-env/bin/activate

# 升级 pip
log "Upgrading pip..."
pip install --upgrade pip

# 安装包
log "Installing ML packages..."
pip install numpy pandas scikit-learn tensorflow torch

log "Setup complete!"
log "To activate: source ml-env/bin/activate"
```

### 示例 2：模型训练自动化

```bash
#!/bin/bash
set -euo pipefail

# train-models.sh - 训练多个模型

MODELS=("resnet50" "vgg16" "mobilenet")
EPOCHS=10
LOG_DIR="logs"

# 创建日志目录
mkdir -p "$LOG_DIR"

function log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $@" | tee -a "$LOG_DIR/training.log"
}

log "Starting training pipeline"

# 训练每个模型
for model in "${MODELS[@]}"; do
    log "Training $model..."

    LOG_FILE="$LOG_DIR/${model}_$(date +%Y%m%d_%H%M%S).log"

    if python train.py --model "$model" --epochs "$EPOCHS" > "$LOG_FILE" 2>&1; then
        log "$model training completed successfully"
    else
        log "ERROR: $model training failed"
        exit 1
    fi
done

log "All models trained successfully"
```

### 示例 3：系统健康检查

```bash
#!/bin/bash

# health-check.sh - 检查系统健康

WARN_CPU=80
WARN_MEM=85
WARN_DISK=90

function check_cpu() {
    local cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
    local cpu_int=${cpu_usage%.*}

    if [ $cpu_int -gt $WARN_CPU ]; then
        echo "WARNING: High CPU usage: ${cpu_usage}%"
        return 1
    fi
    echo "OK: CPU usage: ${cpu_usage}%"
    return 0
}

function check_memory() {
    local mem_usage=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')

    if [ $mem_usage -gt $WARN_MEM ]; then
        echo "WARNING: High memory usage: ${mem_usage}%"
        return 1
    fi
    echo "OK: Memory usage: ${mem_usage}%"
    return 0
}

function check_disk() {
    local disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')

    if [ $disk_usage -gt $WARN_DISK ]; then
        echo "WARNING: High disk usage: ${disk_usage}%"
        return 1
    fi
    echo "OK: Disk usage: ${disk_usage}%"
    return 0
}

# 运行检查
echo "=== System Health Check $(date) ==="
echo ""

STATUS=0
check_cpu || STATUS=1
check_memory || STATUS=1
check_disk || STATUS=1

echo ""
if [ $STATUS -eq 0 ]; then
    echo "Overall status: HEALTHY"
else
    echo "Overall status: WARNING"
fi

exit $STATUS
```

### 示例 4：备份脚本

```bash
#!/bin/bash
set -euo pipefail

# backup-models.sh - 备份 ML 模型

SOURCE_DIR="/models/production"
BACKUP_DIR="/backups/models"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="models_backup_${TIMESTAMP}.tar.gz"

function log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $@"
}

# 检查源是否存在
if [ ! -d "$SOURCE_DIR" ]; then
    log "ERROR: Source directory not found: $SOURCE_DIR"
    exit 1
fi

# 创建备份目录
mkdir -p "$BACKUP_DIR"

# 创建备份
log "Creating backup: $BACKUP_NAME"
tar -czf "${BACKUP_DIR}/${BACKUP_NAME}" -C "$(dirname $SOURCE_DIR)" "$(basename $SOURCE_DIR)"

# 验证备份
if [ -f "${BACKUP_DIR}/${BACKUP_NAME}" ]; then
    SIZE=$(du -h "${BACKUP_DIR}/${BACKUP_NAME}" | cut -f1)
    log "Backup created successfully: ${BACKUP_NAME} (${SIZE})"
else
    log "ERROR: Backup failed"
    exit 1
fi

# 仅保留最近 7 天的备份
log "Cleaning old backups..."
find "$BACKUP_DIR" -name "models_backup_*.tar.gz" -mtime +7 -delete

log "Backup complete"
```

---

## 总结和要点

### 掌握的命令和语法

**脚本基础**：
- `#!/bin/bash` - Shebang 行
- `chmod +x script.sh` - 使其可执行
- `./script.sh` - 运行脚本

**变量**：
- `VAR=value` - 赋值（无空格！）
- `$VAR` 或 `${VAR}` - 引用变量
- `$(command)` - 命令替换
- `$((expression))` - 算术运算

**控制流**：
- `if [ condition ]; then ... fi` - 条件语句
- `for item in list; do ... done` - For 循环
- `while [ condition ]; do ... done` - While 循环

**函数**：
- `function name() { ... }` - 定义函数
- `$1, $2, $3` - 函数参数
- `return` - 退出码
- `local VAR` - 局部变量

**错误处理**：
- `set -e` - 出错时退出
- `set -u` - 未定义变量时退出
- `set -o pipefail` - 管道失败时退出
- `$?` - 上一次退出码

### 关键概念

1. **脚本即程序**：像编写任何其他程序一样编写脚本，包含结构和文档
2. **变量存储数据**：使用变量使脚本灵活且可维护
3. **控制流指导执行**：if/for/while 决定运行哪些代码
4. **函数促进重用**：编写一次，多次使用
5. **错误处理防止灾难**：始终优雅地处理错误

### 最佳实践

✅ **始终使用 shebang**：`#!/bin/bash`
✅ **引用变量**：`"$VAR"` 而不是 `$VAR`
✅ **在函数中使用 local**：防止变量冲突
✅ **检查退出码**：验证命令是否成功
✅ **添加注释**：解释做了什么和为什么
✅ **使用有意义的名称**：`model_name` 而不是 `x`
✅ **处理错误**：使用 `set -e` 或检查返回值
✅ **测试脚本**：在生产前使用测试数据运行

### AI 基础设施的常见模式

```bash
# 使用前检查文件是否存在
if [ ! -f "model.h5" ]; then
    echo "Model not found"
    exit 1
fi

# 遍历模型
for model in resnet50 vgg16 inception; do
    python train.py --model "$model"
done

# 为日志创建时间戳
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="training_${TIMESTAMP}.log"

# 带 GPU 选择运行
for gpu in 0 1 2 3; do
    CUDA_VISIBLE_DEVICES=$gpu python train.py --gpu $gpu &
done
wait

# 简单错误处理
if ! python train.py; then
    echo "Training failed"
    exit 1
fi
```

### 下一步

在**讲座 06：高级 Shell 脚本**中，你将学习：
- 高级控制结构（case 语句）
- 数组和关联数组
- 字符串操作和正则表达式
- 高级参数处理
- 复杂错误处理和调试
- 脚本模板和最佳实践

### 快速参考

```bash
# 变量
VAR="value"
echo "$VAR"
RESULT=$(command)
NUM=$((5 + 3))

# 条件语句
if [ condition ]; then
    # code
fi

# 循环
for item in list; do
    # code
done

while [ condition ]; do
    # code
done

# 函数
function name() {
    local var=$1
    echo "result"
    return 0
}

# 参数
$0  # 脚本名
$1  # 第一个参数
$#  # 参数个数
$@  # 所有参数

# 错误处理
set -e
if ! command; then
    echo "Error" >&2
    exit 1
fi
```

---

**讲座 05：Shell 脚本入门结束**

你现在已经具备了为 AI 基础设施自动化编写简单但强大的 shell 脚本的基础。练习为日常任务编写脚本，以建立信心和熟练度。

**下一步**：讲座 06 - 高级 Shell 脚本
