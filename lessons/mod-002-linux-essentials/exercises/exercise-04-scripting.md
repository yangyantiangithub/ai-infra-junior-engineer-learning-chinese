# 练习 04：ML 部署自动化的 Bash 脚本

## 概述

本练习教你编写 bash 脚本来自动化 ML 部署工作流。你将创建用于模型部署、数据管道自动化、系统监控和备份操作的脚本。这些是构建可靠、可重现的 ML 基础设施的基本技能。

## 学习目标

完成本练习后，你将能够：
- 使用正确的语法和最佳实践编写结构化的 bash 脚本
- 有效使用变量、函数和控制结构
- 处理命令行参数和用户输入
- 实现错误处理和日志记录
- 为 ML 模型创建部署自动化脚本
- 使用 bash 构建数据处理管道
- 开发监控和告警脚本
- 为 ML 基础设施编写备份和恢复脚本

## 先决条件

- 已完成练习 01-03
- 已完成讲座 05：Shell 脚本入门
- 已完成讲座 06：高级 Shell 脚本
- 熟练使用 Linux 命令行
- 基本理解 ML 工作流
- 文本编辑器（vim、nano 或 VS Code）

## 所需时间

- 预计：90 分钟
- 难度：中级

## 第 1 部分：Bash 脚本基础

### 步骤 1：你的第一个 ML 自动化脚本

```bash
# 创建工作空间
mkdir -p ~/ml-scripting-lab
cd ~/ml-scripting-lab

# 创建第一个脚本
cat > hello_ml.sh << 'EOF'
#!/bin/bash
echo "Hello from ML Infrastructure!"
echo "Current user: $(whoami)"
echo "Current directory: $(pwd)"
echo "Today's date: $(date)"

if command -v nvidia-smi &> /dev/null; then
    echo "CUDA available: Yes"
    nvidia-smi --query-gpu=name --format=csv,noheader
else
    echo "CUDA available: No"
fi
EOF

chmod +x hello_ml.sh
./hello_ml.sh
```

### 步骤 2：变量和数据类型

```bash
cd ~/ml-scripting-lab

cat > variables_demo.sh << 'EOF'
#!/bin/bash

# 字符串变量
MODEL_NAME="resnet50"
VERSION="1.2.3"

# 数字变量
EPOCHS=100
BATCH_SIZE=32

# 数组
MODELS=("resnet50" "vgg16" "inception_v3")
echo "Available models: ${MODELS[@]}"
echo "Number of models: ${#MODELS[@]}"

# 关联数组
declare -A MODEL_CONFIGS
MODEL_CONFIGS["resnet50"]="config/resnet50.yaml"
MODEL_CONFIGS["vgg16"]="config/vgg16.yaml"

echo "Model configurations:"
for model in "${!MODEL_CONFIGS[@]}"; do
    echo "  $model: ${MODEL_CONFIGS[$model]}"
done

# 命令替换
CURRENT_DATE=$(date +%Y-%m-%d)
echo "Date: $CURRENT_DATE"

# 字符串操作
FULL_MODEL_NAME="${MODEL_NAME}_v${VERSION}"
echo "Full model name: $FULL_MODEL_NAME"
EOF

chmod +x variables_demo.sh
./variables_demo.sh
```

### 步骤 3：控制结构

```bash
cd ~/ml-scripting-lab

cat > control_structures.sh << 'EOF'
#!/bin/bash

check_gpu() {
    if command -v nvidia-smi &> /dev/null; then
        GPU_COUNT=$(nvidia-smi --query-gpu=name --format=csv,noheader | wc -l)
        echo "GPUs detected: $GPU_COUNT"
        if [ $GPU_COUNT -gt 1 ]; then
            echo "Multi-GPU system"
        fi
    else
        echo "No GPU detected - using CPU"
    fi
}

for model in resnet50 vgg16 mobilenet; do
    echo "Testing $model..."
    sleep 1
    echo "$model: PASSED"
done

check_gpu
EOF

chmod +x control_structures.sh
./control_structures.sh
```

## 第 2 部分：实用 ML 部署脚本

### 步骤 4：模型部署脚本

```bash
cd ~/ml-scripting-lab
mkdir deployment
cd deployment

cat > deploy_model.sh << 'EOF'
#!/bin/bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_DIR="$SCRIPT_DIR/logs"
BACKUP_DIR="$SCRIPT_DIR/backups"
STAGING_DIR="$SCRIPT_DIR/staging"
PRODUCTION_DIR="$SCRIPT_DIR/production"

mkdir -p "$LOG_DIR" "$BACKUP_DIR" "$STAGING_DIR" "$PRODUCTION_DIR"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_DIR/deployment.log"
}

error_exit() {
    log "ERROR: $1"
    exit 1
}

validate_model() {
    local model_path=$1
    [ -f "$model_path" ] || error_exit "Model file not found: $model_path"
    log "Model validation passed"
}

deploy() {
    local model_path=$1
    local environment=$2
    log "Starting deployment: $model_path to $environment"
    validate_model "$model_path"
    cp "$model_path" "$STAGING_DIR/current_model"
    log "Deployed to staging"
    if [ "$environment" = "production" ]; then
        cp "$STAGING_DIR/current_model" "$PRODUCTION_DIR/current_model"
        log "Promoted to production"
    fi
    log "Deployment completed"
}

usage() {
    echo "Usage: $0 <model_path> <environment>"
    exit 1
}

[ $# -eq 2 ] || usage
deploy "$1" "$2"
EOF

chmod +x deploy_model.sh

# 测试部署脚本
touch test_model.h5
dd if=/dev/zero of=test_model.h5 bs=1M count=10 2>/dev/null
./deploy_model.sh test_model.h5 staging
```

### 步骤 5：系统监控脚本

```bash
cd ~/ml-scripting-lab
mkdir monitoring
cd monitoring

cat > monitor_system.sh << 'EOF'
#!/bin/bash
set -euo pipefail

LOG_DIR="logs"
mkdir -p "$LOG_DIR"

CPU_THRESHOLD=80
MEM_THRESHOLD=85
DISK_THRESHOLD=90

check_cpu() {
    local cpu_idle=$(top -bn1 | grep "Cpu(s)" | awk '{print $8}' | cut -d'%' -f1)
    local cpu_usage=$(echo "100 - $cpu_idle" | bc)
    echo "$cpu_usage"
    if (( $(echo "$cpu_usage > $CPU_THRESHOLD" | bc -l) )); then
        echo "WARNING: High CPU: ${cpu_usage}%"
    fi
}

check_memory() {
    local mem_usage=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100}')
    echo "$mem_usage"
    if [ "$mem_usage" -gt "$MEM_THRESHOLD" ]; then
        echo "WARNING: High memory: ${mem_usage}%"
    fi
}

check_disk() {
    local disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
    echo "$disk_usage"
    if [ "$disk_usage" -gt "$DISK_THRESHOLD" ]; then
        echo "CRITICAL: High disk: ${disk_usage}%"
    fi
}

echo "=== System Monitoring ==="
echo "CPU: $(check_cpu)%"
echo "Memory: $(check_memory)%"
echo "Disk: $(check_disk)%"
EOF

chmod +x monitor_system.sh
./monitor_system.sh
```

### 步骤 6：备份和恢复脚本

```bash
cd ~/ml-scripting-lab
mkdir backup_restore
cd backup_restore

cat > backup_ml_project.sh << 'EOF'
#!/bin/bash
set -euo pipefail

BACKUP_ROOT="$HOME/ml-backups"
PROJECT_ROOT="${ML_PROJECT_ROOT:-$HOME/ml-projects}"
LOG_FILE="$BACKUP_ROOT/backup.log"

mkdir -p "$BACKUP_ROOT"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

create_backup() {
    local project_name=$1
    local project_path="$PROJECT_ROOT/$project_name"
    
    [ -d "$project_path" ] || { log "ERROR: Project not found"; return 1; }
    
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_name="${project_name}_${timestamp}.tar.gz"
    
    tar -czf "$BACKUP_ROOT/$backup_name" \
        --exclude='*.pyc' --exclude='__pycache__' \
        -C "$PROJECT_ROOT" "$project_name"
    
    log "Backup created: $backup_name"
}

list_backups() {
    echo "Available Backups:"
    ls -lt "$BACKUP_ROOT"/*.tar.gz 2>/dev/null || echo "No backups found"
}

case "${1:-}" in
    backup) create_backup "${2:-}" ;;
    list) list_backups ;;
    *) echo "Usage: $0 {backup <project>|list}" ;;
esac
EOF

chmod +x backup_ml_project.sh

# 测试备份
mkdir -p ~/ml-projects/test-project
echo "Test content" > ~/ml-projects/test-project/README.md
./backup_ml_project.sh backup test-project
./backup_ml_project.sh list
```

## 验证

```bash
cd ~/ml-scripting-lab

cat > validate_exercise.sh << 'EOF'
#!/bin/bash

PASS=0
FAIL=0

for dir in deployment monitoring backup_restore; do
    if [ -d "$dir" ]; then
        echo "✓ Directory: $dir"
        ((PASS++))
    fi
done

for script in deployment/deploy_model.sh monitoring/monitor_system.sh backup_restore/backup_ml_project.sh; do
    if [ -x "$script" ]; then
        echo "✓ Executable: $script"
        ((PASS++))
    fi
done

echo "Passed: $PASS, Failed: $FAIL"
EOF

chmod +x validate_exercise.sh
./validate_exercise.sh
```

---

**恭喜！** 你已经掌握了 ML 部署自动化的 bash 脚本！
