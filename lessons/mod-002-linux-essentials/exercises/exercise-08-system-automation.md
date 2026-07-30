# 练习 08：ML 基础设施的系统自动化和维护

## 概述

这个动手练习专注于自动化 ML 基础设施的系统维护任务。你将创建自动化备份脚本、设置监控告警、配置日志轮转、安排清理任务以及实施健康检查——这些都是维护生产 ML 系统的关键技能。

## 学习目标

完成本练习后，你将能够：
- 为 ML 工件创建自动化备份脚本
- 使用 cron 和 systemd timers 设置计划任务
- 实施系统健康检查和监控
- 为 ML 应用程序配置日志轮转
- 为 GPU 服务器构建维护脚本
- 为关键问题创建告警机制
- 记录自动化工作流
- 测试和验证自动化系统

## 先决条件

- 已完成练习 01-07
- 模块 002 中的所有讲座，特别是讲座 04：系统管理基础和讲座 06：高级 Shell 脚本
- 理解 systemd 和 cron
- 可访问具有 sudo 权限的 Linux 系统
- 基本理解 ML 工作流

## 所需时间

- 预计：120 分钟
- 难度：中级到高级

---

## 第 1 部分：自动化模型备份系统

### 任务 1.1：创建备份脚本

```bash
# 创建项目目录
mkdir -p ~/ml-automation-lab/backups
cd ~/ml-automation-lab/backups

# 创建备份脚本
cat > backup_models.sh << 'EOF'
#!/bin/bash
set -euo pipefail

# 配置
MODEL_DIR="/opt/ml/models"
BACKUP_DIR="/backup/models"
RETENTION_DAYS=30
LOG_FILE="/var/log/ml-backup.log"
DATE=$(date +%Y%m%d_%H%M%S)

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

mkdir -p "$BACKUP_DIR"
log "Starting model backup..."

BACKUP_NAME="models_backup_${DATE}.tar.gz"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"

tar -czf "$BACKUP_PATH" \
    --exclude='*.tmp' \
    --exclude='checkpoints/temp' \
    -C "$(dirname $MODEL_DIR)" \
    "$(basename $MODEL_DIR)" 2>&1 | tee -a "$LOG_FILE"

if [ $? -eq 0 ]; then
    BACKUP_SIZE=$(du -h "$BACKUP_PATH" | cut -f1)
    log "Backup completed: $BACKUP_NAME (Size: $BACKUP_SIZE)"
else
    log "ERROR: Backup failed!"
    exit 1
fi

log "Cleaning up backups older than $RETENTION_DAYS days..."
find "$BACKUP_DIR" -name "models_backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete
log "Backup process completed"
EOF

chmod +x backup_models.sh
```

**测试脚本：**

```bash
# 创建测试模型目录
sudo mkdir -p /opt/ml/models/production
sudo mkdir -p /backup/models
sudo chown -R $USER:$USER /opt/ml /backup

# 创建虚拟模型文件
echo "model_v1.0" > /opt/ml/models/production/model.onnx
echo "config" > /opt/ml/models/production/config.yaml

# 运行备份
./backup_models.sh

# 验证备份
ls -lh /backup/models/
```

### 任务 1.2：安排每日备份

```bash
# 打开 crontab
crontab -e

# 添加此行（每天凌晨 2 点运行）：
0 2 * * * /home/$USER/ml-automation-lab/backups/backup_models.sh

# 验证 cron 作业
crontab -l

# 使用 systemd timer 的替代方案（推荐）：
sudo tee /etc/systemd/system/ml-backup.service > /dev/null << 'EOF'
[Unit]
Description=ML Model Backup Service
After=network.target
[Service]
Type=oneshot
User=$USER
ExecStart=/home/$USER/ml-automation-lab/backups/backup_models.sh
[Install]
WantedBy=multi-user.target
EOF

sudo tee /etc/systemd/system/ml-backup.timer > /dev/null << 'EOF'
[Unit]
Description=Run ML backup daily
[Timer]
OnCalendar=daily
OnCalendar=02:00
Persistent=true
[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable ml-backup.timer
sudo systemctl start ml-backup.timer
```

---

## 第 2 部分：GPU 健康监控

### 任务 2.1：创建 GPU 监控脚本

```bash
mkdir -p ~/ml-automation-lab/monitoring
cd ~/ml-automation-lab/monitoring

cat > monitor_gpus.sh << 'EOF'
#!/bin/bash
set -euo pipefail

TEMP_THRESHOLD=80
MEMORY_THRESHOLD=90
ALERT_EMAIL="ml-ops@company.com"
LOG_FILE="/var/log/gpu-monitor.log"

if ! command -v nvidia-smi &> /dev/null; then
    echo "ERROR: nvidia-smi not found. GPU monitoring requires NVIDIA drivers."
    exit 1
fi

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

send_alert() {
    local subject="$1"
    local message="$2"
    log "ALERT: $subject - $message"
    logger -t gpu-monitor -p user.warning "$subject: $message"
}

GPU_COUNT=$(nvidia-smi --query-gpu=count --format=csv,noheader | head -1)
log "Monitoring $GPU_COUNT GPU(s)..."

nvidia-smi --query-gpu=index,name,temperature.gpu,utilization.gpu,utilization.memory,memory.used,memory.total \
    --format=csv,noheader | while IFS=',' read -r idx name temp util_gpu util_mem mem_used mem_total; do
    
    temp=$(echo $temp | xargs)
    util_gpu=$(echo $util_gpu | xargs | sed 's/ %//')
    util_mem=$(echo $util_mem | xargs | sed 's/ %//')
    
    log "GPU $idx ($name): Temp=${temp}°C, GPU_Util=${util_gpu}%, Mem_Util=${util_mem}%"
    
    if [ "$temp" -gt "$TEMP_THRESHOLD" ]; then
        send_alert "GPU Temperature Alert" "GPU $idx temperature is ${temp}°C"
    fi
done

log "GPU monitoring check completed"
EOF

chmod +x monitor_gpus.sh
```

### 任务 2.2：安排 GPU 监控

```bash
# 每 5 分钟运行一次
crontab -e

# 添加：
*/5 * * * * /home/$USER/ml-automation-lab/monitoring/monitor_gpus.sh
```

---

## 第 3 部分：自动化日志轮转

### 任务 3.1：为 ML 应用程序配置日志轮转

```bash
sudo tee /etc/logrotate.d/ml-applications > /dev/null << 'EOF'
/var/log/ml-api/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 mluser mluser
    sharedscripts
    postrotate
        systemctl reload ml-api.service > /dev/null 2>&1 || true
    endscript
}

/var/log/ml-training/*.log {
    weekly
    rotate 8
    compress
    delaycompress
    missingok
    notifempty
    size 100M
    create 0640 mluser mluser
}
EOF

# 测试 logrotate 配置
sudo logrotate -d /etc/logrotate.d/ml-applications

# 强制轮转（用于测试）
sudo logrotate -f /etc/logrotate.d/ml-applications
```

---

## 第 4 部分：自动化清理任务

### 任务 4.1：创建清理脚本

```bash
mkdir -p ~/ml-automation-lab/cleanup
cd ~/ml-automation-lab/cleanup

cat > cleanup_ml_artifacts.sh << 'EOF'
#!/bin/bash
set -euo pipefail

LOG_FILE="/var/log/ml-cleanup.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "Starting ML artifact cleanup..."

# 清理旧实验输出（超过 30 天）
if [ -d "/opt/ml/experiments" ]; then
    find /opt/ml/experiments -type d -mtime +30 -exec rm -rf {} + 2>/dev/null || true
fi

# 清理旧检查点（保留最近 7 天）
if [ -d "/opt/ml/checkpoints" ]; then
    find /opt/ml/checkpoints -name "*.ckpt" -mtime +7 -delete 2>/dev/null || true
fi

# 清理未使用的 Docker 镜像
if command -v docker &> /dev/null; then
    docker image prune -a --filter "until=168h" -f 2>/dev/null || true
fi

# 清理 pip 缓存
pip cache purge 2>/dev/null || true

log "Cleanup completed"
EOF

chmod +x cleanup_ml_artifacts.sh
```

### 任务 4.2：安排每周清理

```bash
# 每周日凌晨 3 点运行
crontab -e

# 添加：
0 3 * * 0 /home/$USER/ml-automation-lab/cleanup/cleanup_ml_artifacts.sh
```

---

## 第 5 部分：系统健康检查脚本

### 任务 5.1：全面健康检查

```bash
mkdir -p ~/ml-automation-lab/health-check
cd ~/ml-automation-lab/health-check

cat > health_check.sh << 'EOF'
#!/bin/bash
set -euo pipefail

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

check_pass() { echo -e "${GREEN}✓${NC} $1"; }
check_fail() { echo -e "${RED}✗${NC} $1"; }
check_warn() { echo -e "${YELLOW}!${NC} $1"; }

echo "======================================"
echo "   ML Infrastructure Health Check"
echo "======================================"

# 1. 磁盘空间
DISK_USAGE=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -lt 80 ]; then
    check_pass "Disk usage: ${DISK_USAGE}%"
elif [ "$DISK_USAGE" -lt 90 ]; then
    check_warn "Disk usage: ${DISK_USAGE}%"
else
    check_fail "Disk usage: ${DISK_USAGE}%"
fi

# 2. 内存
MEM_USAGE=$(free | grep Mem | awk '{printf("%.0f", $3/$2 * 100)}')
if [ "$MEM_USAGE" -lt 85 ]; then
    check_pass "Memory usage: ${MEM_USAGE}%"
else
    check_warn "Memory usage: ${MEM_USAGE}%"
fi

# 3. CPU 负载
CPU_COUNT=$(nproc)
LOAD_AVG=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//')
check_pass "Load average: $LOAD_AVG (CPUs: $CPU_COUNT)"

# 4. 关键服务
SERVICES=("docker" "nginx")
for service in "${SERVICES[@]}"; do
    if systemctl is-active --quiet "$service" 2>/dev/null; then
        check_pass "$service is running"
    else
        check_fail "$service is not running"
    fi
done

# 5. GPU 状态
if command -v nvidia-smi &> /dev/null; then
    GPU_COUNT=$(nvidia-smi --query-gpu=count --format=csv,noheader | head -1)
    check_pass "Found $GPU_COUNT GPU(s)"
else
    check_warn "nvidia-smi not found"
fi

# 6. 网络连接
if ping -c 1 8.8.8.8 &> /dev/null; then
    check_pass "Internet connectivity OK"
else
    check_fail "No internet connectivity"
fi

echo ""
echo "Health Check Complete"
EOF

chmod +x health_check.sh
```

**运行健康检查：**

```bash
./health_check.sh
```

---

## 第 6 部分：集成和测试

### 任务 6.1：创建主自动化脚本

```bash
cat > ~/ml-automation-lab/run_all_maintenance.sh << 'EOF'
#!/bin/bash
set -euo pipefail

echo "=== ML Infrastructure Automated Maintenance ==="

echo "Running health check..."
~/ml-automation-lab/health-check/health_check.sh
echo ""

echo "Checking GPU status..."
~/ml-automation-lab/monitoring/monitor_gpus.sh
echo ""

echo "Running backup..."
~/ml-automation-lab/backups/backup_models.sh
echo ""

echo "Running cleanup..."
~/ml-automation-lab/cleanup/cleanup_ml_artifacts.sh
echo ""

echo "All maintenance tasks completed!"
EOF

chmod +x ~/ml-automation-lab/run_all_maintenance.sh
```

### 任务 6.2：测试所有自动化

```bash
# 运行所有维护任务
~/ml-automation-lab/run_all_maintenance.sh

# 验证 cron 作业
crontab -l

# 检查 systemd timers
systemctl list-timers

# 查看日志
tail -50 /var/log/ml-backup.log 2>/dev/null || true
tail -50 /var/log/gpu-monitor.log 2>/dev/null || true
```

---

## 挑战问题

1. **监控**：如何扩展 GPU 监控以在 Prometheus 中跟踪指标？

2. **备份策略**：什么是 3-2-1 备份规则？你的脚本如何符合该规则？

3. **告警**：如何与 PagerDuty 或 Slack 集成以实现关键告警？

4. **扩展**：如何修改这些脚本以在 100 台 GPU 服务器上工作？

5. **测试**：如何在不影响生产的情况下测试备份/恢复流程？

---

**恭喜！** 你已经为 ML 基础设施维护构建了一个完整的自动化套件。这些脚本构成了生产 ML 运维的基础。
