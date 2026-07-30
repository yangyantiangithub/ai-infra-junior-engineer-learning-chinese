# 练习 07：真实世界故障排查场景

## 概述

本练习通过真实的 ML 基础设施故障排查场景来检验你的 Linux 技能。你将诊断和修复常见问题，如磁盘已满错误、权限问题、进程挂起、内存问题和网络连接问题。

## 学习目标

完成本练习后，你将能够：
- 诊断系统资源问题（磁盘、内存、CPU）
- 排查权限和所有权问题
- 处理失控和卡住的进程
- 调试网络连接问题
- 解决包依赖冲突
- 修复 GPU 和 CUDA 相关问题
- 创建系统化排查工作流
- 记录解决方案以供将来参考

## 先决条件

- 已完成所有先前练习（01-06）
- 模块 002 中的所有讲座
- 对 Linux 命令行有信心
- 可访问测试用 Linux 系统

## 所需时间

- 预计：90 分钟
- 难度：中级到高级

## 场景 1：磁盘已满错误

### 问题

```
ML 训练作业失败，错误："OSError: [Errno 28] No space left on device"
模型正在向 /var/ml/checkpoints/ 保存检查点
```

### 调查

```bash
mkdir -p ~/troubleshooting-lab/scenario1
cd ~/troubleshooting-lab/scenario1

cat > scenario1_investigate.sh << 'EOF'
#!/bin/bash
echo "=== Disk Full Investigation ==="

echo "Step 1: Check disk usage"
df -h

echo ""
echo "Step 2: Find large directories"
du -sh /var/* 2>/dev/null | sort -hr | head -10

echo ""
echo "Step 3: Check inode usage"
df -i

echo ""
echo "Step 4: Find old checkpoint files"
find /var/ml/checkpoints -type f -mtime +7 2>/dev/null | wc -l
EOF

chmod +x scenario1_investigate.sh

cat > scenario1_solution.sh << 'EOF'
#!/bin/bash
# Solution: Free up disk space

cat << 'SOLUTION'
1. Remove old checkpoints (keep last 5)
   ls -t /var/ml/checkpoints/*.h5 | tail -n +6 | xargs rm

2. Compress old checkpoints
   find /var/ml/checkpoints -name "*.h5" -mtime +7 -exec gzip {} \;

3. Clean temporary files
   rm -rf /tmp/*

4. Clean package caches
   sudo apt clean  # Ubuntu
   sudo yum clean all  # RHEL

5. Set up log rotation
   sudo logrotate -f /etc/logrotate.conf

Prevention:
- Implement checkpoint retention policy
- Monitor disk usage with alerting
- Use separate partition for ML data
SOLUTION
EOF

chmod +x scenario1_solution.sh
```

## 场景 2：权限被拒绝

### 问题

```
训练脚本失败："PermissionError: [Errno 13] Permission denied: '/data/processed/train.csv'"
文件存在，但训练进程无法读取它。
```

### 调查

```bash
cd ~/troubleshooting-lab
mkdir scenario2
cd scenario2

cat > scenario2_debug.sh << 'EOF'
#!/bin/bash
echo "=== Permission Investigation ==="

# 模拟环境
mkdir -p /tmp/ml_scenario2/data/processed
echo "sample,data" > /tmp/ml_scenario2/data/processed/train.csv
chmod 600 /tmp/ml_scenario2/data/processed/train.csv

DATA_FILE="/tmp/ml_scenario2/data/processed/train.csv"

echo "Step 1: Check file permissions"
ls -l $DATA_FILE 2>&1

echo ""
echo "Step 2: Check ownership"
ls -l $DATA_FILE | awk '{print "Owner:", $3, "Group:", $4}'

echo ""
echo "Step 3: Check current user"
echo "User: $(whoami)"
echo "Groups: $(groups)"

echo ""
echo "=== Solution ==="
cat << 'SOLUTION'
Possible fixes:
1. chmod 644 /data/processed/train.csv
2. sudo chown $(whoami) /data/processed/train.csv
3. chmod 755 /data/processed (for parent directory)
4. setfacl -m u:ml-user:r /data/processed/train.csv
SOLUTION

rm -rf /tmp/ml_scenario2
EOF

chmod +x scenario2_debug.sh
./scenario2_debug.sh
```

## 场景 3：进程挂起/无响应

### 问题

```
训练进程（PID 12345）已运行 6 小时但没有任何进展。
GPU 利用率为 0%，但进程对 Ctrl+C 无响应。
```

### 调查

```bash
cd ~/troubleshooting-lab
mkdir scenario3
cd scenario3

cat > scenario3_debug.sh << 'EOF'
#!/bin/bash
echo "=== Hung Process Investigation ==="

DEMO_PID=$$
echo "Investigating hung process (demo PID: $DEMO_PID)"

echo "Step 1: Check process state"
ps -p $DEMO_PID -o pid,stat,time,cmd

echo ""
echo "Step 2: Check open files"
lsof -p $DEMO_PID 2>/dev/null | head -10 || echo "lsof not available"

echo ""
echo "=== Solution ==="
cat << 'SOLUTION'
1. Try graceful termination first:
   kill -TERM 12345
   sleep 10

2. If still running, force kill:
   kill -9 12345

3. If zombie process:
   ps -o ppid= -p 12345
   kill -9 <parent_pid>

4. Debug with strace:
   sudo strace -p 12345

Common causes:
- Network I/O timeout
- Disk I/O wait
- Deadlock in code
- Waiting for GPU
SOLUTION
EOF

chmod +x scenario3_debug.sh
./scenario3_debug.sh
```

## 场景 4：内存不足（OOM）

### 问题

```
训练进程被意外终止，显示"Killed"消息
dmesg 显示："Out of memory: Kill process 12345 (python)"
```

### 调查

```bash
cd ~/troubleshooting-lab
mkdir scenario4
cd scenario4

cat > scenario4_debug.sh << 'EOF'
#!/bin/bash
echo "=== OOM Investigation ==="

echo "Step 1: Check current memory usage"
free -h

echo ""
echo "Step 2: Check swap usage"
swapon -s 2>/dev/null || echo "No swap"

echo ""
echo "Step 3: Find memory-hungry processes"
ps aux --sort=-%mem | head -10

echo ""
echo "=== Solution ==="
cat << 'SOLUTION'
Immediate fixes:
1. Reduce batch size: batch_size=16 instead of 32
2. Enable swap: sudo fallocate -l 8G /swapfile; sudo mkswap /swapfile; sudo swapon /swapfile
3. Use gradient accumulation: accumulation_steps=4
4. Use mixed precision training
5. Enable memory growth (TensorFlow):
   tf.config.experimental.set_memory_growth(gpu, True)

Prevention:
- Profile memory before full training
- Start with small dataset to test
- Monitor memory during training
SOLUTION
EOF

chmod +x scenario4_debug.sh
./scenario4_debug.sh
```

## 场景 5：CUDA/GPU 不可用

### 问题

```
训练脚本报告："No CUDA-capable device is detected"
但 nvidia-smi 显示 GPU 存在
```

### 调查

```bash
cd ~/troubleshooting-lab
mkdir scenario5
cd scenario5

cat > scenario5_debug.sh << 'EOF'
#!/bin/bash
echo "=== CUDA Investigation ==="

echo "Step 1: Check GPU detection"
lspci | grep -i nvidia || echo "No NVIDIA GPU found"

echo ""
echo "Step 2: Check NVIDIA driver"
nvidia-smi 2>&1 || echo "nvidia-smi failed"

echo ""
echo "Step 3: Check CUDA installation"
nvcc --version 2>&1 || echo "nvcc not found"

echo ""
echo "Step 4: Check library path"
echo "LD_LIBRARY_PATH: ${LD_LIBRARY_PATH:-not set}"

echo ""
echo "=== Solution ==="
cat << 'SOLUTION'
Common fixes:
1. Load driver: sudo modprobe nvidia
2. Check version compatibility
3. Set library path:
   export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
4. Add user to video group: sudo usermod -aG video $USER
5. For Docker: docker run --gpus all ...
6. Reinstall CUDA-enabled packages

Verification:
python3 -c "import tensorflow as tf; print(tf.config.list_physical_devices('GPU'))"
python3 -c "import torch; print(torch.cuda.is_available())"
SOLUTION
EOF

chmod +x scenario5_debug.sh
./scenario5_debug.sh
```

## 场景 6：网络连接问题

### 问题

```
训练脚本无法下载数据集："ConnectionError: Failed to establish connection"
wget 和 curl 到该 URL 也失败
```

### 调查

```bash
cd ~/troubleshooting-lab
mkdir scenario6
cd scenario6

cat > scenario6_debug.sh << 'EOF'
#!/bin/bash
echo "=== Network Investigation ==="

echo "Step 1: Check network interface"
ip addr show | grep "inet " | head -5

echo ""
echo "Step 2: Check default route"
ip route show | grep default

echo ""
echo "Step 3: Check DNS resolution"
nslookup google.com || echo "DNS resolution failed"

echo ""
echo "Step 4: Ping gateway"
ip route | grep default | awk '{print $3}' | xargs ping -c 2 2>&1 || echo "Cannot reach gateway"

echo ""
echo "Step 5: Ping external server"
ping -c 2 8.8.8.8 || echo "Cannot reach internet"

echo ""
echo "=== Solution ==="
cat << 'SOLUTION'
1. DNS not working:
   echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf

2. No internet:
   sudo systemctl restart NetworkManager

3. Firewall blocking:
   sudo ufw status
   sudo ufw allow out 443/tcp

4. Proxy required:
   export HTTP_PROXY=http://proxy.example.com:8080

5. Test connectivity:
   nc -zv google.com 80
SOLUTION
EOF

chmod +x scenario6_debug.sh
./scenario6_debug.sh
```

## 综合排查检查清单

```bash
cd ~/troubleshooting-lab

cat > troubleshooting_checklist.md << 'EOF'
# ML Infrastructure Troubleshooting Checklist

## System Resources
- [ ] Check disk: df -h
- [ ] Check memory: free -h
- [ ] Check load: uptime
- [ ] Check GPU: nvidia-smi

## Process Issues
- [ ] Check if running: ps aux | grep process
- [ ] Try graceful kill: kill -TERM PID
- [ ] Force kill: kill -9 PID

## Permissions
- [ ] Check permissions: ls -l file
- [ ] Check ownership: stat file
- [ ] Check user groups: groups

## Network
- [ ] Check interface: ip addr
- [ ] Test DNS: nslookup example.com
- [ ] Test connectivity: ping 8.8.8.8

## Logs
- [ ] Check application logs
- [ ] Check system logs: journalctl -xe
- [ ] Check for errors: grep -i error /var/log/syslog
EOF
```

## 验证

```bash
cd ~/troubleshooting-lab

cat > validate_exercise.sh << 'EOF'
#!/bin/bash

PASS=0; FAIL=0

for i in {1..6}; do
    if [ -d "scenario$i" ]; then
        echo "✓ Scenario $i directory"
        ((PASS++))
    fi
done

for i in {1..6}; do
    if [ -x "scenario$i/scenario${i}_debug.sh" ]; then
        echo "✓ Scenario $i script"
        ((PASS++))
    fi
done

if [ -f "troubleshooting_checklist.md" ]; then
    echo "✓ Checklist created"; ((PASS++))
fi

echo "Passed: $PASS, Failed: $FAIL"
EOF

chmod +x validate_exercise.sh
./validate_exercise.sh
```

---

**恭喜！** 你已经完成了所有 Linux 基础练习！现在你已具备处理真实 ML 基础设施挑战的能力。
