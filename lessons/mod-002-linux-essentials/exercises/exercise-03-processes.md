# 练习 03：ML 训练作业的进程管理

## 概述

本练习教你如何在 ML 基础设施的背景下管理 Linux 进程。你将学习监控训练作业、控制 GPU 进程、管理系统服务和处理长时间运行的 ML 工作负载。这些技能对于运行高效、可靠的 ML 基础设施至关重要。

## 学习目标

完成本练习后，你将能够：
- 使用 ps、top 和 htop 监控运行中的进程
- 管理进程生命周期（启动、停止、暂停、恢复、终止）
- 使用作业控制处理后台和前台进程
- 使用 nvidia-smi 监控 GPU 进程
- 使用 systemctl 管理系统服务
- 处理卡住或失控的 ML 训练进程
- 为 ML 工作负载实施资源监控
- 使用 screen 和 tmux 进行持久训练会话

## 先决条件

- 已完成练习 01-02
- 已完成讲座 04：系统管理基础
- 可访问 Linux 系统
- 基本 Python 知识（用于 ML 训练示例）
- 可选：用于 GPU 监控部分的 NVIDIA GPU
- 至少 2GB RAM 和 2 个 CPU 核心

## 所需时间

- 预计：75-90 分钟
- 难度：中级

## 第 1 部分：理解进程

### 步骤 1：探索运行中的进程

```bash
# 创建工作空间
mkdir -p ~/ml-process-management
cd ~/ml-process-management

# 查看当前进程
ps

# 查看当前用户的所有进程
ps -u $(whoami)

# 查看系统上所有进程
ps aux

# 查看进程树
ps auxf
pstree

# 查找特定进程
ps aux | grep python
ps aux | grep bash

# 保存进程快照供分析
ps aux > process_snapshot.txt
```

### 步骤 2：实时进程监控

```bash
# top 交互式命令：
# P = 按 CPU 排序
# M = 按内存排序
# k = 终止进程
# r = 调整优先级
# q = 退出

# 如果需要，安装 htop
# sudo apt install htop

# 记录 top 输出供以后分析
top -b -n 1 > top_snapshot.txt
```

## 第 2 部分：进程控制和作业管理

### 步骤 3：后台和前台作业

```bash
cd ~/ml-process-management

# 创建长时间运行的脚本
cat > long_running_task.sh << 'EOF'
#!/bin/bash
for i in {1..60}; do
    echo "Processing epoch $i/60..."
    sleep 1
done
echo "Training complete!"
EOF

chmod +x long_running_task.sh

# 在后台运行
./long_running_task.sh &

# 列出后台作业
jobs

# 创建作业控制演示
cat > job_control_demo.sh << 'EOF'
#!/bin/bash
echo "Starting multiple background jobs..."
(for i in {1..30}; do echo "Job 1: $i"; sleep 1; done) &
(for i in {1..30}; do echo "Job 2: $i"; sleep 1; done) &
echo "Current jobs:"
jobs
wait
echo "All jobs completed"
EOF

chmod +x job_control_demo.sh
./job_control_demo.sh
```

### 步骤 4：进程信号和终止

```bash
cd ~/ml-process-management

# 创建信号测试脚本
cat > signal_test.py << 'EOF'
#!/usr/bin/env python3
import signal, sys, time, os

def signal_handler(signum, frame):
    print(f"\nReceived signal {signum}")
    if signum == signal.SIGTERM:
        print("Graceful shutdown...")
        sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)
signal.signal(signal.SIGINT, signal_handler)

print(f"Process started (PID: {os.getpid()})")
try:
    while True:
        print(f"Working... {time.time()}")
        time.sleep(2)
except KeyboardInterrupt:
    sys.exit(0)
EOF

chmod +x signal_test.py
```

## 第 3 部分：ML 训练进程管理

### 步骤 5：模拟 ML 训练进程

```bash
cd ~/ml-process-management
mkdir ml_training_sim
cd ml_training_sim

# 创建进程管理包装器
cat > manage_training.sh << 'EOF'
#!/bin/bash

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_DIR="$SCRIPT_DIR/logs"
PID_FILE="$SCRIPT_DIR/training.pid"

mkdir -p "$LOG_DIR"

start_training() {
    if [ -f "$PID_FILE" ]; then
        PID=$(cat "$PID_FILE")
        if ps -p $PID > /dev/null 2>&1; then
            echo "Training already running (PID: $PID)"
            return 1
        fi
    fi
    nohup python3 "$SCRIPT_DIR/train_model.py" --epochs 60 \
        > "$LOG_DIR/training.log" 2>&1 &
    PID=$!
    echo $PID > "$PID_FILE"
    echo "Training started (PID: $PID)"
}

stop_training() {
    if [ ! -f "$PID_FILE" ]; then
        echo "No training process found"
        return 1
    fi
    PID=$(cat "$PID_FILE")
    kill -TERM $PID
    for i in {1..10}; do
        if ! ps -p $PID > /dev/null 2>&1; then
            echo "Training stopped gracefully"
            rm "$PID_FILE"
            return 0
        fi
        sleep 1
    done
    kill -9 $PID
    rm "$PID_FILE"
    echo "Training stopped (forced)"
}

status_training() {
    if [ ! -f "$PID_FILE" ]; then
        echo "Status: Not running"
        return 1
    fi
    PID=$(cat "$PID_FILE")
    if ps -p $PID > /dev/null 2>&1; then
        echo "Status: Running (PID: $PID)"
        echo "Latest log entries:"
        tail -5 "$LOG_DIR/training.log"
    else
        echo "Status: Stopped"
        rm "$PID_FILE"
    fi
}

case "$1" in
    start) start_training ;;
    stop) stop_training ;;
    status) status_training ;;
    *) echo "Usage: $0 {start|stop|status}" ;;
esac
EOF

chmod +x manage_training.sh
```

## 第 4 部分：GPU 进程管理

### 步骤 6：监控 GPU 进程

```bash
cd ~/ml-process-management
mkdir gpu_management
cd gpu_management

# 创建 GPU 监控脚本
cat > monitor_gpu.sh << 'EOF'
#!/bin/bash

if ! command -v nvidia-smi &> /dev/null; then
    echo "nvidia-smi not found. GPU monitoring not available."
    exit 1
fi

echo "=== GPU Monitoring ==="
nvidia-smi --query-gpu=index,name,memory.total,driver_version --format=csv
echo ""
nvidia-smi --query-gpu=index,utilization.gpu,memory.used,memory.total,temperature.gpu --format=csv
echo ""
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv
EOF

chmod +x monitor_gpu.sh
```

## 第 5 部分：systemctl 系统服务

### 步骤 7：管理系统服务

```bash
cd ~/ml-process-management
mkdir service_management
cd service_management

# 创建示例服务单元文件
cat > ml-training.service << 'EOF'
[Unit]
Description=ML Training Service
After=network.target

[Service]
Type=simple
User=mluser
WorkingDirectory=/opt/ml/training
ExecStart=/usr/bin/python3 /opt/ml/training/train.py
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
CPUQuota=200%
MemoryLimit=8G
Environment="CUDA_VISIBLE_DEVICES=0"

[Install]
WantedBy=multi-user.target
EOF
```

## 第 6 部分：screen 和 tmux 的持久会话

### 步骤 8：为长时间运行的训练使用 screen

```bash
cd ~/ml-process-management
mkdir persistent_sessions
cd persistent_sessions

# screen 参考
# screen -S training    启动命名会话
# python train.py       在 screen 中运行训练
# Ctrl+a d              分离会话
# screen -r training    重新连接

# tmux 参考
# tmux new -s training  启动命名会话
# Ctrl+b d              分离
# tmux attach -t training 重新连接
```

## 最终验证

```bash
cd ~/ml-process-management

cat > validate_exercise.sh << 'EOF'
#!/bin/bash

PASS=0
FAIL=0

for dir in ml_training_sim gpu_management service_management persistent_sessions; do
    if [ -d "$dir" ]; then
        echo "✓ Directory: $dir"
        ((PASS++))
    fi
done

echo "Passed: $PASS, Failed: $FAIL"
EOF

chmod +x validate_exercise.sh
./validate_exercise.sh
```

---

**恭喜！** 你已经掌握了 ML 基础设施的 Linux 进程管理。
