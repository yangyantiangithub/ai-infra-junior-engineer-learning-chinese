# 讲座 06：高级 Shell 脚本

## 目录
1. [简介](#简介)
2. [高级控制结构](#高级控制结构)
3. [数组和关联数组](#数组和关联数组)
4. [高级字符串操作](#高级字符串操作)
5. [高级参数处理](#高级参数处理)
6. [Bash 中的正则表达式](#bash-中的正则表达式)
7. [高级错误处理和调试](#高级错误处理和调试)
8. [脚本模板和最佳实践](#脚本模板和最佳实践)
9. [实际 AI 基础设施自动化](#实际-ai-基础设施自动化)
10. [总结和要点](#总结和要点)

## 简介

在讲座 05 的基础上，本讲座深入高级 shell 脚本技巧。你将学习编写复杂、生产就绪的脚本，以稳健和优雅的方式处理复杂的 AI 基础设施自动化任务。

### 学习目标

完成本讲座后，你将能够：
- 掌握 case 语句和高级条件语句
- 使用数组和关联数组
- 执行复杂的字符串操作
- 专业地处理命令行参数
- 使用正则表达式进行模式匹配
- 实现全面的错误处理
- 有效调试脚本
- 编写生产就绪的自动化脚本
- 将高级技巧应用到 AI 基础设施

### 先决条件
- 讲座 05（Shell 脚本入门）
- 熟悉基本 bash 脚本
- 理解变量、函数和基本控制流

### 为什么需要高级脚本？

**生产就绪**：基本脚本适合个人使用；高级脚本处理生产工作负载

**健壮性**：优雅地处理边缘情况、错误和意外输入

**可维护性**：结构良好的代码更容易调试和扩展

**可扩展性**：能处理一个模型的脚本必须能处理数百个

**专业性**：生产基础设施需要专业的代码质量

**时长**：120 分钟
**难度**：中级到高级

---

## 高级控制结构

### Case 语句

Case 语句提供比多个 if-elif-else 链更清晰的语法。

**基本 case 语句**：
```bash
#!/bin/bash

ACTION=$1

case "$ACTION" in
    start)
        echo "Starting service..."
        systemctl start ml-api
        ;;
    stop)
        echo "Stopping service..."
        systemctl stop ml-api
        ;;
    restart)
        echo "Restarting service..."
        systemctl restart ml-api
        ;;
    status)
        systemctl status ml-api
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

**case 中的模式匹配**：
```bash
#!/bin/bash
# 处理不同的模型格式

FILE=$1

case "$FILE" in
    *.h5)
        echo "Keras model detected"
        python convert.py --format keras "$FILE"
        ;;
    *.pt|*.pth)
        echo "PyTorch model detected"
        python convert.py --format pytorch "$FILE"
        ;;
    *.onnx)
        echo "ONNX model detected"
        python convert.py --format onnx "$FILE"
        ;;
    *.pb)
        echo "TensorFlow SavedModel detected"
        python convert.py --format tensorflow "$FILE"
        ;;
    *)
        echo "ERROR: Unknown model format: $FILE"
        exit 1
        ;;
esac
```

**每个 case 多个模式**：
```bash
#!/bin/bash

ENVIRONMENT=$1

case "$ENVIRONMENT" in
    dev|development)
        CONFIG="config.dev.yaml"
        REPLICAS=1
        RESOURCES="minimal"
        ;;
    staging|stg)
        CONFIG="config.staging.yaml"
        REPLICAS=2
        RESOURCES="medium"
        ;;
    prod|production)
        CONFIG="config.prod.yaml"
        REPLICAS=4
        RESOURCES="maximum"
        ;;
    test|testing)
        CONFIG="config.test.yaml"
        REPLICAS=1
        RESOURCES="minimal"
        ;;
    *)
        echo "ERROR: Unknown environment: $ENVIRONMENT"
        echo "Valid: dev, staging, prod, test"
        exit 1
        ;;
esac

echo "Deploying to $ENVIRONMENT"
echo "Config: $CONFIG"
echo "Replicas: $REPLICAS"
echo "Resources: $RESOURCES"
```

**带正则表达式的 case**（使用 [[ ]] 和 =~）：
```bash
#!/bin/bash

INPUT=$1

# 验证版本格式
if [[ "$INPUT" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Valid semantic version: $INPUT"
elif [[ "$INPUT" =~ ^[0-9]+\.[0-9]+$ ]]; then
    echo "Minor version format: $INPUT"
elif [[ "$INPUT" =~ ^[0-9]+$ ]]; then
    echo "Major version only: $INPUT"
else
    echo "Invalid version format: $INPUT"
    exit 1
fi
```

### 高级条件语句

**双中括号 [[ ]] 与单中括号 [ ]**：

```bash
# [[ ]] 支持额外特性
# - 使用 == 进行模式匹配
# - 使用 =~ 进行正则匹配
# - 逻辑运算符（&&, ||）无需引号
# - 字符串比较无需引号

# 模式匹配
if [[ "$FILENAME" == *.txt ]]; then
    echo "Text file"
fi

# 正则匹配
if [[ "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Valid version"
fi

# 多个条件无需转义
if [[ -f "model.h5" && -f "config.yaml" ]]; then
    echo "Both files exist"
fi

# 字符串比较无需引号
if [[ $STATUS == "success" ]]; then
    echo "Operation succeeded"
fi
```

**三元风格操作**：
```bash
# 使用 && 和 ||
[ -f "model.h5" ] && echo "File exists" || echo "File not found"

# 分配默认值
RESULT=$(command) || RESULT="default"

# 复杂的三元风格
GPU_AVAILABLE=$(command -v nvidia-smi &> /dev/null && echo "yes" || echo "no")
```

---

## 数组和关联数组

### 索引数组

**创建和填充数组**：
```bash
# 方法 1：直接赋值
MODELS=("resnet50" "vgg16" "inception" "mobilenet")

# 方法 2：单个元素
GPUS[0]=0
GPUS[1]=1
GPUS[2]=2
GPUS[3]=3

# 方法 3：来自命令输出
FILES=($(ls *.h5))
USERS=($(cut -d: -f1 /etc/passwd))
```

**访问数组元素**：
```bash
MODELS=("resnet50" "vgg16" "inception")

echo ${MODELS[0]}           # 第一个元素：resnet50
echo ${MODELS[1]}           # 第二个元素：vgg16
echo ${MODELS[-1]}          # 最后一个元素：inception
echo ${MODELS[@]}           # 所有元素（作为独立单词）
echo ${MODELS[*]}           # 所有元素（作为单个单词）
echo ${#MODELS[@]}          # 数组长度：3
```

**遍历数组**：
```bash
MODELS=("resnet50" "vgg16" "inception" "mobilenet")

# 方法 1：遍历值
for model in "${MODELS[@]}"; do
    echo "Training $model"
    python train.py --model "$model"
done

# 方法 2：遍历索引
for i in "${!MODELS[@]}"; do
    echo "Model $i: ${MODELS[$i]}"
done

# 方法 3：C 风格
for ((i=0; i<${#MODELS[@]}; i++)); do
    echo "Model $i: ${MODELS[$i]}"
done
```

**数组操作**：
```bash
# 添加元素
MODELS+=("squeezenet")

# 删除元素
unset MODELS[2]

# 切片数组
MODELS=("resnet50" "vgg16" "inception" "mobilenet" "squeezenet")
echo ${MODELS[@]:1:3}       # vgg16 inception mobilenet（起始:长度）

# 复制数组
NEW_MODELS=("${MODELS[@]}")

# 连接数组
ARRAY1=(1 2 3)
ARRAY2=(4 5 6)
COMBINED=("${ARRAY1[@]}" "${ARRAY2[@]}")
```

**实际示例 - 多 GPU 训练**：
```bash
#!/bin/bash
set -euo pipefail

# 要训练的模型
MODELS=("resnet50" "vgg16" "inception" "mobilenet")

# 可用的 GPU
GPUS=(0 1 2 3)

# 循环分配模型到 GPU
GPU_INDEX=0

for model in "${MODELS[@]}"; do
    gpu=${GPUS[$GPU_INDEX]}

    echo "Training $model on GPU $gpu"
    CUDA_VISIBLE_DEVICES=$gpu python train.py --model "$model" \
        --output "results/${model}" &

    # 下一个 GPU（循环）
    GPU_INDEX=$(( (GPU_INDEX + 1) % ${#GPUS[@]} ))
done

# 等待所有训练作业完成
wait

echo "All models trained successfully"
```

### 关联数组

关联数组使用字符串作为键（需要 Bash 4+）。

**创建关联数组**：
```bash
# 声明关联数组
declare -A MODEL_PATHS

# 添加键值对
MODEL_PATHS["resnet50"]="/models/resnet50.h5"
MODEL_PATHS["vgg16"]="/models/vgg16.h5"
MODEL_PATHS["inception"]="/models/inception.h5"

# 替代：声明并填充
declare -A CONFIG=(
    ["dev"]="config.dev.yaml"
    ["staging"]="config.staging.yaml"
    ["prod"]="config.prod.yaml"
)
```

**访问关联数组**：
```bash
declare -A MODEL_PATHS=(
    ["resnet50"]="/models/resnet50.h5"
    ["vgg16"]="/models/vgg16.h5"
)

# 按键访问
echo ${MODEL_PATHS["resnet50"]}      # /models/resnet50.h5

# 所有键
echo ${!MODEL_PATHS[@]}              # resnet50 vgg16

# 所有值
echo ${MODEL_PATHS[@]}               # /models/resnet50.h5 /models/vgg16.h5

# 元素数量
echo ${#MODEL_PATHS[@]}              # 2

# 检查键是否存在
if [[ -v MODEL_PATHS["resnet50"] ]]; then
    echo "Key exists"
fi
```

**遍历关联数组**：
```bash
declare -A MODEL_PATHS=(
    ["resnet50"]="/models/resnet50.h5"
    ["vgg16"]="/models/vgg16.h5"
    ["inception"]="/models/inception.h5"
)

# 遍历键
for model in "${!MODEL_PATHS[@]}"; do
    echo "$model -> ${MODEL_PATHS[$model]}"
done

# 更复杂的遍历
for model in "${!MODEL_PATHS[@]}"; do
    path="${MODEL_PATHS[$model]}"

    if [ -f "$path" ]; then
        echo "✓ $model: $path"
    else
        echo "✗ $model: $path (NOT FOUND)"
    fi
done
```

**实际示例 - 配置管理**：
```bash
#!/bin/bash
set -euo pipefail

# 环境配置
declare -A CONFIGS=(
    ["dev_api_url"]="http://localhost:8000"
    ["dev_replicas"]="1"
    ["dev_resources"]="minimal"

    ["prod_api_url"]="https://api.production.com"
    ["prod_replicas"]="10"
    ["prod_resources"]="maximum"
)

ENV=$1  # dev 或 prod

# 获取环境配置值
API_URL=${CONFIGS["${ENV}_api_url"]}
REPLICAS=${CONFIGS["${ENV}_replicas"]}
RESOURCES=${CONFIGS["${ENV}_resources"]}

echo "Deploying to $ENV"
echo "API URL: $API_URL"
echo "Replicas: $REPLICAS"
echo "Resources: $RESOURCES"

# 使用配置部署
kubectl apply -f deployment.yaml \
    --set apiUrl="$API_URL" \
    --set replicas="$REPLICAS" \
    --set resources="$RESOURCES"
```

---

## 高级字符串操作

### 字符串长度和子串

```bash
TEXT="Hello, World!"

# 长度
echo ${#TEXT}                    # 13

# 子串（偏移量:长度）
echo ${TEXT:0:5}                 # Hello
echo ${TEXT:7}                   # World!
echo ${TEXT:7:5}                 # World
echo ${TEXT: -6}                 # World!（注意空格）
```

### 字符串替换

```bash
FILENAME="model_v1_final_v1.h5"

# 替换第一个
echo ${FILENAME/v1/v2}           # model_v2_final_v1.h5

# 替换所有
echo ${FILENAME//v1/v2}          # model_v2_final_v2.h5

# 替换前缀
PATH_STR="/models/resnet50"
echo ${PATH_STR/#\/models/\/data}  # /data/resnet50

# 替换后缀
echo ${FILENAME/%.h5/.pt}        # model_v1_final_v1.pt
```

### 删除模式

```bash
FILEPATH="/data/models/resnet50.h5"

# 删除最短前缀匹配
echo ${FILEPATH#*/}              # data/models/resnet50.h5

# 删除最长前缀匹配
echo ${FILEPATH##*/}             # resnet50.h5（基本名称）

# 删除最短后缀匹配
echo ${FILEPATH%.*}              # /data/models/resnet50

# 删除最长后缀匹配
echo ${FILEPATH%%.*}             # /data/models/resnet50
```

**实际示例 - 路径操作**：
```bash
#!/bin/bash

# 从路径中提取组件
FULL_PATH="/data/models/production/resnet50_v2.h5"

# 获取目录
DIR=${FULL_PATH%/*}              # /data/models/production

# 获取文件名
FILENAME=${FULL_PATH##*/}        # resnet50_v2.h5

# 获取基本名称（无扩展名）
BASENAME=${FILENAME%.*}          # resnet50_v2

# 获取扩展名
EXTENSION=${FILENAME##*.}        # h5

echo "Full path: $FULL_PATH"
echo "Directory: $DIR"
echo "Filename: $FILENAME"
echo "Basename: $BASENAME"
echo "Extension: $EXTENSION"

# 转换为不同格式
NEW_PATH="${DIR}/${BASENAME}.onnx"
echo "New path: $NEW_PATH"
```

### 大小写转换

```bash
TEXT="Hello World"

# Bash 4+
echo ${TEXT^}                    # Hello World（首字符大写）
echo ${TEXT^^}                   # HELLO WORLD（全部大写）
echo ${TEXT,}                    # hello World（首字符小写）
echo ${TEXT,,}                   # hello world（全部小写）

# 旧 Bash 的替代方案
echo "$TEXT" | tr '[:lower:]' '[:upper:]'   # HELLO WORLD
echo "$TEXT" | tr '[:upper:]' '[:lower:]'   # hello world
```

### 默认值

```bash
# 未设置时使用默认值
echo ${VAR:-default}             # 如果 VAR 未设置，使用 'default'

# 未设置时设置默认值
VAR=${VAR:=default}              # 如果 VAR 未设置，将其设置为 'default'

# 已设置时使用替代值
echo ${VAR:+alternative}         # 如果 VAR 已设置，使用 'alternative'

# 未设置时出错
echo ${VAR:?error message}       # 如果 VAR 未设置，显示错误消息
```

**实际示例 - 带默认值的配置**：
```bash
#!/bin/bash

# ML 训练配置及默认值
MODEL_NAME=${MODEL_NAME:-resnet50}
BATCH_SIZE=${BATCH_SIZE:-32}
LEARNING_RATE=${LEARNING_RATE:-0.001}
EPOCHS=${EPOCHS:-100}
GPU_ID=${GPU_ID:-0}

echo "Training Configuration:"
echo "  Model: $MODEL_NAME"
echo "  Batch size: $BATCH_SIZE"
echo "  Learning rate: $LEARNING_RATE"
echo "  Epochs: $EPOCHS"
echo "  GPU: $GPU_ID"

# 运行训练
CUDA_VISIBLE_DEVICES=$GPU_ID python train.py \
    --model "$MODEL_NAME" \
    --batch-size "$BATCH_SIZE" \
    --lr "$LEARNING_RATE" \
    --epochs "$EPOCHS"

# 使用示例：
# ./train.sh                          # 全部默认
# MODEL_NAME=vgg16 ./train.sh         # 自定义模型
# BATCH_SIZE=64 EPOCHS=200 ./train.sh # 自定义参数
```

---

## 高级参数处理

### 长选项解析

```bash
#!/bin/bash
set -euo pipefail

# train.sh - 带长选项的训练脚本

usage() {
    cat << EOF
Usage: $0 [OPTIONS]

Train ML model with specified configuration.

OPTIONS:
    --model=MODEL       Model architecture (default: resnet50)
    --epochs=N          Number of epochs (default: 100)
    --batch-size=N      Batch size (default: 32)
    --gpu=N             GPU ID (default: 0)
    --output=DIR        Output directory (required)
    --config=FILE       Config file (optional)
    --verbose           Enable verbose logging
    --help              Show this help message

EXAMPLES:
    $0 --model=vgg16 --epochs=50 --output=results/
    $0 --config=config.yaml --gpu=2 --output=results/
EOF
    exit 1
}

# 默认值
MODEL="resnet50"
EPOCHS=100
BATCH_SIZE=32
GPU=0
OUTPUT=""
CONFIG=""
VERBOSE=false

# 解析参数
while [[ $# -gt 0 ]]; do
    case "$1" in
        --model=*)
            MODEL="${1#*=}"
            shift
            ;;
        --epochs=*)
            EPOCHS="${1#*=}"
            shift
            ;;
        --batch-size=*)
            BATCH_SIZE="${1#*=}"
            shift
            ;;
        --gpu=*)
            GPU="${1#*=}"
            shift
            ;;
        --output=*)
            OUTPUT="${1#*=}"
            shift
            ;;
        --config=*)
            CONFIG="${1#*=}"
            shift
            ;;
        --verbose)
            VERBOSE=true
            shift
            ;;
        --help)
            usage
            ;;
        *)
            echo "ERROR: Unknown option: $1"
            usage
            ;;
    esac
done

# 验证必需参数
if [ -z "$OUTPUT" ]; then
    echo "ERROR: --output is required"
    usage
fi

# 显示配置
echo "Training Configuration:"
echo "  Model: $MODEL"
echo "  Epochs: $EPOCHS"
echo "  Batch size: $BATCH_SIZE"
echo "  GPU: $GPU"
echo "  Output: $OUTPUT"
[ -n "$CONFIG" ] && echo "  Config: $CONFIG"
[ "$VERBOSE" = true ] && echo "  Verbose: enabled"

# 构建命令
CMD="python train.py --model $MODEL --epochs $EPOCHS --batch-size $BATCH_SIZE --gpu $GPU --output $OUTPUT"
[ -n "$CONFIG" ] && CMD="$CMD --config $CONFIG"
[ "$VERBOSE" = true ] && CMD="$CMD --verbose"

# 执行
echo "Running: $CMD"
eval "$CMD"
```

### 混合短选项和长选项

```bash
#!/bin/bash

# 同时解析短选项（-m）和长选项（--model=）

while [[ $# -gt 0 ]]; do
    case "$1" in
        -m|--model)
            MODEL="$2"
            shift 2
            ;;
        --model=*)
            MODEL="${1#*=}"
            shift
            ;;
        -e|--epochs)
            EPOCHS="$2"
            shift 2
            ;;
        --epochs=*)
            EPOCHS="${1#*=}"
            shift
            ;;
        -h|--help)
            usage
            ;;
        *)
            echo "Unknown option: $1"
            exit 1
            ;;
    esac
done

# 现在支持：
# ./script.sh -m resnet50
# ./script.sh --model resnet50
# ./script.sh --model=resnet50
```

### 选项后的位置参数

```bash
#!/bin/bash

# deploy.sh [OPTIONS] environment target

OPTIONS=""
while [[ $# -gt 0 ]]; do
    case "$1" in
        --*)
            OPTIONS="$OPTIONS $1"
            shift
            ;;
        -*)
            OPTIONS="$OPTIONS $1 $2"
            shift 2
            ;;
        *)
            break  # 在第一个位置参数处停止
            ;;
    esac
done

# 剩余参数是位置参数
ENVIRONMENT=$1
TARGET=$2

echo "Options: $OPTIONS"
echo "Environment: $ENVIRONMENT"
echo "Target: $TARGET"
```

---

## Bash 中的正则表达式

### 使用 =~ 进行模式匹配

```bash
# 正则匹配需要 [[ ]]

if [[ "$STRING" =~ pattern ]]; then
    echo "Match found"
fi
```

**验证版本号**：
```bash
VERSION="1.2.3"

if [[ "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Valid semantic version"
else
    echo "Invalid version format"
fi
```

**使用捕获组提取信息**：
```bash
INPUT="model_resnet50_epoch_100.h5"

if [[ "$INPUT" =~ model_([a-z0-9]+)_epoch_([0-9]+) ]]; then
    MODEL_NAME="${BASH_REMATCH[1]}"
    EPOCH="${BASH_REMATCH[2]}"

    echo "Model: $MODEL_NAME"
    echo "Epoch: $EPOCH"
fi
```

**验证 IP 地址**：
```bash
IP="192.168.1.100"

# 简单验证
if [[ "$IP" =~ ^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$ ]]; then
    echo "Valid IP format"
fi
```

**实际示例 - 解析训练日志**：
```bash
#!/bin/bash

# 从训练日志中提取指标

LOG_FILE="training.log"

while IFS= read -r line; do
    # 匹配模式：Epoch 50/100 - loss: 0.1234 - acc: 0.9876
    if [[ "$line" =~ Epoch[[:space:]]+([0-9]+)/([0-9]+).*loss:[[:space:]]+([0-9.]+).*acc:[[:space:]]+([0-9.]+) ]]; then
        EPOCH="${BASH_REMATCH[1]}"
        TOTAL="${BASH_REMATCH[2]}"
        LOSS="${BASH_REMATCH[3]}"
        ACCURACY="${BASH_REMATCH[4]}"

        echo "Epoch $EPOCH/$TOTAL: Loss=$LOSS, Accuracy=$ACCURACY"
    fi
done < "$LOG_FILE"
```

---

## 高级错误处理和调试

### 全面错误处理

```bash
#!/bin/bash

# 严格错误处理
set -euo pipefail
IFS=$'\n\t'

# 错误处理函数
function error_handler() {
    local line=$1
    local exit_code=$2

    echo "ERROR at line $line: Command exited with status $exit_code" >&2

    # 出错时清理
    cleanup

    exit $exit_code
}

# 捕获错误
trap 'error_handler ${LINENO} $?' ERR

# 清理函数
function cleanup() {
    echo "Performing cleanup..."
    # 删除临时文件、停止进程等
}

# 捕获退出
trap cleanup EXIT

# 捕获中断
trap 'echo "Interrupted"; exit 130' INT TERM
```

### 验证函数

```bash
#!/bin/bash

# 验证文件存在
function require_file() {
    local file=$1
    local description=${2:-"File"}

    if [ ! -f "$file" ]; then
        echo "ERROR: $description not found: $file" >&2
        return 1
    fi
}

# 验证目录存在
function require_directory() {
    local dir=$1

    if [ ! -d "$dir" ]; then
        echo "ERROR: Directory not found: $dir" >&2
        return 1
    fi
}

# 验证命令存在
function require_command() {
    local cmd=$1

    if ! command -v "$cmd" &> /dev/null; then
        echo "ERROR: Required command not found: $cmd" >&2
        return 1
    fi
}

# 验证数字值
function require_number() {
    local value=$1
    local name=$2

    if ! [[ "$value" =~ ^[0-9]+$ ]]; then
        echo "ERROR: $name must be a number: $value" >&2
        return 1
    fi
}

# 验证 GPU 可用
function require_gpu() {
    local gpu_id=$1

    if ! command -v nvidia-smi &> /dev/null; then
        echo "ERROR: nvidia-smi not found" >&2
        return 1
    fi

    local gpu_count=$(nvidia-smi --list-gpus | wc -l)
    if [ "$gpu_id" -ge "$gpu_count" ]; then
        echo "ERROR: GPU $gpu_id not found (available: 0-$((gpu_count-1)))" >&2
        return 1
    fi
}

# 在脚本中使用
require_file "config.yaml" "Configuration file"
require_directory "/data/models"
require_command "python3"
require_number "$EPOCHS" "EPOCHS"
require_gpu 0
```

### 重试逻辑

```bash
#!/bin/bash

# 带指数退避的重试函数
function retry() {
    local max_attempts=$1
    shift
    local delay=1
    local attempt=1

    while [ $attempt -le $max_attempts ]; do
        echo "Attempt $attempt/$max_attempts: $@"

        if "$@"; then
            echo "Success on attempt $attempt"
            return 0
        fi

        if [ $attempt -lt $max_attempts ]; then
            echo "Failed. Retrying in ${delay}s..."
            sleep $delay
            delay=$((delay * 2))  # 指数退避
        fi

        attempt=$((attempt + 1))
    done

    echo "Failed after $max_attempts attempts"
    return 1
}

# 用法
retry 3 curl -f https://api.example.com/health
retry 5 python train.py --resume
```

### 调试技巧

**调试模式**：
```bash
#!/bin/bash

# 使用环境变量启用调试模式
DEBUG=${DEBUG:-0}

function debug() {
    if [ "$DEBUG" = "1" ]; then
        echo "[DEBUG] $@" >&2
    fi
}

debug "Starting script"
debug "Model: $MODEL_NAME"
debug "GPU: $GPU_ID"

# 用法：
# DEBUG=1 ./script.sh
```

**跟踪执行**：
```bash
#!/bin/bash

# 为特定部分启用跟踪
set -x  # 开始跟踪
command1
command2
set +x  # 停止跟踪

# 或跟踪整个脚本
bash -x script.sh
```

**详细模式**：
```bash
#!/bin/bash

VERBOSE=false

function log() {
    if [ "$VERBOSE" = true ]; then
        echo "[$(date +'%Y-%m-%d %H:%M:%S')] $@"
    fi
}

# 解析 --verbose 标志
while [[ $# -gt 0 ]]; do
    case "$1" in
        --verbose|-v)
            VERBOSE=true
            shift
            ;;
        *)
            shift
            ;;
    esac
done

log "Starting training"
log "Loading model"
```

---

## 脚本模板和最佳实践

### 生产脚本模板

```bash
#!/bin/bash

################################################################################
# 脚本名称：production-template.sh
# 描述：包含所有最佳实践的生产就绪脚本模板
# 作者：你的名字
# 日期：$(date +%Y-%m-%d)
# 版本：1.0.0
#
# 用法：./production-template.sh [OPTIONS] <required_arg>
#
# 要求：
#   - Bash 4.0+
#   - 必需命令：python3、nvidia-smi
#
# 退出码：
#   0 - 成功
#   1 - 一般错误
#   2 - 无效参数
#   3 - 缺少依赖
################################################################################

# 严格模式
set -euo pipefail
IFS=$'\n\t'

################################################################################
# 全局变量
################################################################################

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"
readonly SCRIPT_VERSION="1.0.0"
readonly LOG_FILE="${LOG_FILE:-/var/log/${SCRIPT_NAME%.sh}.log}"

# 默认值
MODEL_NAME="${MODEL_NAME:-resnet50}"
EPOCHS="${EPOCHS:-100}"
VERBOSE=false
DRY_RUN=false

################################################################################
# 工具函数
################################################################################

# 打印彩色输出
function print_red()    { echo -e "\033[0;31m$@\033[0m"; }
function print_green()  { echo -e "\033[0;32m$@\033[0m"; }
function print_yellow() { echo -e "\033[0;33m$@\033[0m"; }
function print_blue()   { echo -e "\033[0;34m$@\033[0m"; }

# 日志函数
function log_info()  { echo "[$(date +'%Y-%m-%d %H:%M:%S')] [INFO]  $@" | tee -a "$LOG_FILE"; }
function log_warn()  { echo "[$(date +'%Y-%m-%d %H:%M:%S')] [WARN]  $@" | tee -a "$LOG_FILE"; }
function log_error() { echo "[$(date +'%Y-%m-%d %H:%M:%S')] [ERROR] $@" | tee -a "$LOG_FILE" >&2; }
function log_debug() { [ "$VERBOSE" = true ] && echo "[$(date +'%Y-%m-%d %H:%M:%S')] [DEBUG] $@" | tee -a "$LOG_FILE" >&2 || true; }

# 错误处理
function die() {
    log_error "$@"
    exit 1
}

################################################################################
# 错误处理
################################################################################

function error_handler() {
    local line=$1
    log_error "Script failed at line $line"
}

trap 'error_handler ${LINENO}' ERR

function cleanup() {
    log_info "Performing cleanup..."
    # 在此添加清理代码
}

trap cleanup EXIT
trap 'log_warn "Interrupted"; exit 130' INT TERM

################################################################################
# 验证函数
################################################################################

function check_dependencies() {
    log_debug "Checking dependencies..."

    local deps=("python3" "nvidia-smi")
    for cmd in "${deps[@]}"; do
        if ! command -v "$cmd" &> /dev/null; then
            die "Required command not found: $cmd"
        fi
    done
}

function validate_arguments() {
    log_debug "Validating arguments..."

    # 验证模型名称
    if [ -z "$MODEL_NAME" ]; then
        die "MODEL_NAME cannot be empty"
    fi

    # 验证 epochs
    if ! [[ "$EPOCHS" =~ ^[0-9]+$ ]] || [ "$EPOCHS" -lt 1 ]; then
        die "EPOCHS must be a positive integer"
    fi
}

################################################################################
# 用法和帮助
################################################################################

function usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS] <required_arg>

Description of what this script does.

OPTIONS:
    -m, --model=NAME      Model architecture (default: $MODEL_NAME)
    -e, --epochs=N        Number of epochs (default: $EPOCHS)
    -v, --verbose         Enable verbose output
    -n, --dry-run         Show what would be done without doing it
    -h, --help            Show this help message
    --version             Show version information

EXAMPLES:
    $SCRIPT_NAME --model=vgg16 --epochs=50
    $SCRIPT_NAME -m resnet50 -e 100 --verbose

EXIT CODES:
    0 - Success
    1 - General error
    2 - Invalid arguments
    3 - Missing dependencies

EOF
    exit 0
}

function version() {
    echo "$SCRIPT_NAME version $SCRIPT_VERSION"
    exit 0
}

################################################################################
# 参数解析
################################################################################

function parse_arguments() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -m|--model)
                MODEL_NAME="$2"
                shift 2
                ;;
            --model=*)
                MODEL_NAME="${1#*=}"
                shift
                ;;
            -e|--epochs)
                EPOCHS="$2"
                shift 2
                ;;
            --epochs=*)
                EPOCHS="${1#*=}"
                shift
                ;;
            -v|--verbose)
                VERBOSE=true
                shift
                ;;
            -n|--dry-run)
                DRY_RUN=true
                shift
                ;;
            -h|--help)
                usage
                ;;
            --version)
                version
                ;;
            *)
                die "Unknown option: $1"
                ;;
        esac
    done
}

################################################################################
# 主函数
################################################################################

function do_something() {
    log_info "Doing something with $MODEL_NAME for $EPOCHS epochs"

    if [ "$DRY_RUN" = true ]; then
        log_info "[DRY RUN] Would execute: python train.py --model $MODEL_NAME --epochs $EPOCHS"
        return
    fi

    # 实际工作在此
    python train.py --model "$MODEL_NAME" --epochs "$EPOCHS"
}

function main() {
    log_info "Starting $SCRIPT_NAME v$SCRIPT_VERSION"

    check_dependencies
    validate_arguments

    do_something

    log_info "Script completed successfully"
}

################################################################################
# 脚本执行
################################################################################

# 解析参数
parse_arguments "$@"

# 运行主函数
main
```

### 最佳实践检查清单

✅ **Shebang**：以 `#!/bin/bash` 开头
✅ **严格模式**：使用 `set -euo pipefail`
✅ **头部注释**：脚本名称、描述、作者、日期
✅ **常量**：使用 `readonly` 声明常量
✅ **函数**：将代码分解为可重用的函数
✅ **引用**：始终引用变量：`"$VAR"`
✅ **局部变量**：在函数中使用 `local`
✅ **错误处理**：使用 `trap` 和错误处理器
✅ **日志记录**：记录重要事件并带时间戳
✅ **验证**：使用前验证输入
✅ **用法函数**：提供清晰的用法信息
✅ **退出码**：使用有意义的退出码
✅ **注释**：解释为什么，而不是什么
✅ **空运行模式**：支持 --dry-run 用于测试
✅ **版本信息**：包含 --version 标志

---

## 实际 AI 基础设施自动化

### 完整模型部署管道

```bash
#!/bin/bash
set -euo pipefail

################################################################################
# deploy-ml-model.sh - 完整的 ML 模型部署管道
################################################################################

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly LOG_FILE="/var/log/model-deployment.log"

# 配置
MODEL_REGISTRY="/models/registry"
STAGING_DIR="/models/staging"
PRODUCTION_DIR="/models/production"
BACKUP_DIR="/backups/models"
SERVICE_NAME="ml-inference-api"

# 颜色
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
NC='\033[0m'

# 日志记录
function log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $@" | tee -a "$LOG_FILE"
}

function log_error() {
    echo -e "${RED}[ERROR]${NC} $@" | tee -a "$LOG_FILE" >&2
}

function log_success() {
    echo -e "${GREEN}[SUCCESS]${NC} $@" | tee -a "$LOG_FILE"
}

function log_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $@" | tee -a "$LOG_FILE"
}

# 验证模型
function validate_model() {
    local model_path=$1

    log "Validating model: $model_path"

    # 检查文件是否存在
    if [ ! -f "$model_path" ]; then
        log_error "Model file not found: $model_path"
        return 1
    fi

    # 检查文件大小
    local size=$(stat -f%z "$model_path" 2>/dev/null || stat -c%s "$model_path")
    if [ "$size" -lt 1000 ]; then
        log_error "Model file too small: $size bytes"
        return 1
    fi

    # 验证模型格式
    if ! python -c "import tensorflow as tf; tf.keras.models.load_model('$model_path')" &> /dev/null; then
        log_error "Invalid model format"
        return 1
    fi

    log_success "Model validation passed"
    return 0
}

# 备份当前模型
function backup_current_model() {
    log "Backing up current production model..."

    local current_model="$PRODUCTION_DIR/current.h5"

    if [ ! -f "$current_model" ]; then
        log_warning "No current model to backup"
        return 0
    fi

    local backup_name="model_$(date +%Y%m%d_%H%M%S).h5"
    local backup_path="$BACKUP_DIR/$backup_name"

    mkdir -p "$BACKUP_DIR"
    cp "$current_model" "$backup_path"

    log_success "Backup saved: $backup_path"

    # 仅保留最近 10 个备份
    log "Cleaning old backups..."
    ls -t "$BACKUP_DIR"/model_*.h5 | tail -n +11 | xargs -r rm
}

# 部署到暂存环境
function deploy_to_staging() {
    local source_model=$1

    log "Deploying to staging..."

    mkdir -p "$STAGING_DIR"
    cp "$source_model" "$STAGING_DIR/candidate.h5"

    log_success "Deployed to staging"
}

# 运行验证测试
function run_validation_tests() {
    log "Running validation tests..."

    local test_script="$SCRIPT_DIR/validate_model.py"

    if [ ! -f "$test_script" ]; then
        log_error "Test script not found: $test_script"
        return 1
    fi

    if python "$test_script" --model "$STAGING_DIR/candidate.h5"; then
        log_success "All validation tests passed"
        return 0
    else
        log_error "Validation tests failed"
        return 1
    fi
}

# 推广到生产
function promote_to_production() {
    log "Promoting to production..."

    backup_current_model

    mkdir -p "$PRODUCTION_DIR"
    cp "$STAGING_DIR/candidate.h5" "$PRODUCTION_DIR/current.h5"

    log_success "Model promoted to production"
}

# 重启服务
function restart_service() {
    log "Restarting $SERVICE_NAME..."

    if systemctl restart "$SERVICE_NAME"; then
        log_success "Service restarted successfully"

        # 等待服务就绪
        sleep 5

        # 健康检查
        if systemctl is-active "$SERVICE_NAME" &> /dev/null; then
            log_success "Service is running"
        else
            log_error "Service failed to start"
            return 1
        fi
    else
        log_error "Failed to restart service"
        return 1
    fi
}

# 回滚
function rollback() {
    log_warning "Rolling back deployment..."

    local latest_backup=$(ls -t "$BACKUP_DIR"/model_*.h5 | head -1)

    if [ -z "$latest_backup" ]; then
        log_error "No backup available for rollback"
        return 1
    fi

    cp "$latest_backup" "$PRODUCTION_DIR/current.h5"
    restart_service

    log_success "Rollback completed"
}

# 主部署管道
function main() {
    local model_path=$1

    if [ -z "$model_path" ]; then
        log_error "Usage: $0 <model_path>"
        exit 1
    fi

    log "=== Starting Model Deployment Pipeline ==="
    log "Model: $model_path"

    # 验证
    if ! validate_model "$model_path"; then
        log_error "Model validation failed"
        exit 1
    fi

    # 部署到暂存环境
    if ! deploy_to_staging "$model_path"; then
        log_error "Staging deployment failed"
        exit 1
    fi

    # 运行测试
    if ! run_validation_tests; then
        log_error "Validation tests failed"
        exit 1
    fi

    # 推广到生产
    if ! promote_to_production; then
        log_error "Production promotion failed"
        rollback
        exit 1
    fi

    # 重启服务
    if ! restart_service; then
        log_error "Service restart failed"
        rollback
        exit 1
    fi

    log_success "=== Deployment completed successfully ==="
}

# 执行
main "$@"
```

---

## 总结和要点

### 掌握的高级技巧

**控制结构**：
- Case 语句用于更清晰的条件判断
- 高级模式匹配
- 使用 [[ ]] 的复杂条件表达式

**数据结构**：
- 索引数组用于列表
- 关联数组用于键值对
- 数组操作和遍历

**字符串处理**：
- 高级替换和替代
- 模式删除（前缀/后缀）
- 大小写转换

**参数**：
- 长选项解析（--option=value）
- 混合短选项和长选项
- 带选项的位置参数

**正则表达式**：
- 使用 =~ 进行模式匹配
- 使用 BASH_REMATCH 的捕获组
- 输入验证

**错误处理**：
- 全面错误捕获
- 验证函数
- 重试逻辑
- 调试技巧

### 生产最佳实践

✅ **使用严格模式**：`set -euo pipefail`
✅ **验证所有输入**：永远不要信任用户输入
✅ **记录所有内容**：时间戳和严重级别
✅ **优雅处理错误**：捕获、清理、回滚
✅ **支持空运行模式**：不执行的情况下测试
✅ **提供清晰的用法**：带示例的 --help
✅ **对脚本进行版本控制**：跟踪随时间的变化
✅ **彻底记录**：注释、头部、README
✅ **使其幂等**：多次运行安全
✅ **广泛测试**：函数的单元测试

### AI 基础设施应用

**模型部署**：带验证和回滚的自动化管道
**训练自动化**：多模型、多 GPU 编排
**资源管理**：健康检查、告警、自动扩缩
**数据处理**：数据集的 ETL 管道
**监控**：指标收集和分析
**备份/恢复**：带保留策略的自动化备份

### 下一步

在**讲座 07：文本处理工具**中，你将学习：
- grep 用于模式匹配
- sed 用于流编辑
- awk 用于文本处理和报告
- 在管道中组合工具
- 分析 ML 日志和指标

### 快速参考

```bash
# Case 语句
case "$VAR" in
    pattern1) commands ;;
    pattern2) commands ;;
    *) default ;;
esac

# 数组
ARRAY=(item1 item2 item3)
echo ${ARRAY[0]}
for item in "${ARRAY[@]}"; do echo $item; done

# 关联数组
declare -A MAP
MAP["key"]="value"
echo ${MAP["key"]}

# 字符串操作
${VAR#pattern}      # 删除前缀
${VAR%pattern}      # 删除后缀
${VAR/old/new}      # 替换第一个
${VAR//old/new}     # 替换所有

# 正则匹配
if [[ "$VAR" =~ pattern ]]; then
    echo ${BASH_REMATCH[0]}
fi

# 错误处理
set -euo pipefail
trap cleanup EXIT
trap 'error_handler ${LINENO}' ERR
```

---

**讲座 06：高级 Shell 脚本结束**

你现在已经拥有了生产 AI 基础设施自动化的高级 shell 脚本技能。这些技巧使你能够编写健壮、可维护的脚本，处理真实的复杂性。

**下一步**：讲座 07 - 文本处理工具（grep、sed、awk）
