# 练习 06：ML 系统的日志文件分析

## 概述

本练习教你分析来自 ML 训练系统的日志文件、排查问题以及从日志中提取洞察。你将处理应用程序日志、系统日志和训练指标，以诊断问题并监控 ML 工作负载。

## 学习目标

完成本练习后，你将能够：
- 读取和分析不同类型的日志文件
- 使用 grep、awk、sed 进行日志解析
- 使用 tail 实时监控日志
- 使用 journalctl 分析系统日志
- 从日志中提取训练指标
- 识别错误模式和根本原因
- 创建日志分析脚本
- 设置日志轮转和保留策略

## 先决条件

- 已完成练习 01-05
- 已完成讲座 07：文本处理工具
- 理解 Linux 命令行
- 基本正则表达式知识
- 熟悉文本编辑器

## 所需时间

- 预计：60-75 分钟
- 难度：中级

## 第 1 部分：理解日志文件

### 步骤 1：常见日志位置

```bash
mkdir -p ~/log-analysis-lab
cd ~/log-analysis-lab

# 创建示例日志文件
mkdir -p sample_logs

# 训练日志
cat > sample_logs/training.log << 'EOF'
2024-10-18 10:00:00 INFO Starting training session
2024-10-18 10:00:01 INFO Loading dataset from /data/train
2024-10-18 10:00:10 INFO Epoch 1/100 - loss: 2.3012 - accuracy: 0.1234 - val_loss: 2.1234 - val_accuracy: 0.1567
2024-10-18 10:01:30 INFO Epoch 2/100 - loss: 1.8765 - accuracy: 0.3456 - val_loss: 1.7890 - val_accuracy: 0.3789
2024-10-18 10:03:00 INFO Epoch 3/100 - loss: 1.5432 - accuracy: 0.4567 - val_loss: 1.4567 - val_accuracy: 0.4890
2024-10-18 10:04:15 ERROR CUDA out of memory - reducing batch size from 32 to 16
2024-10-18 10:05:30 INFO Epoch 4/100 - loss: 1.2345 - accuracy: 0.5678 - val_loss: 1.1890 - val_accuracy: 0.5901
2024-10-18 10:07:00 INFO Epoch 5/100 - loss: 1.0123 - accuracy: 0.6345 - val_loss: 0.9876 - val_accuracy: 0.6512
2024-10-18 10:08:30 INFO Early stopping triggered
2024-10-18 10:08:31 INFO Training completed in 8 minutes 31 seconds
EOF

# API 日志
cat > sample_logs/api.log << 'EOF'
2024-10-18 10:05:23 INFO [REQUEST] GET /predict - IP: 192.168.1.100
2024-10-18 10:05:23 INFO [RESPONSE] 200 - Prediction: class_2 - Confidence: 0.876 - Duration: 45ms
2024-10-18 10:06:45 INFO [REQUEST] POST /predict/batch - IP: 192.168.1.101
2024-10-18 10:06:46 INFO [RESPONSE] 200 - Batch size: 100 - Duration: 1234ms
2024-10-18 10:08:12 ERROR [RESPONSE] 400 - Invalid input format - Duration: 5ms
2024-10-18 10:10:01 ERROR [RESPONSE] 401 - Unauthorized - Duration: 2ms
EOF

# 错误日志
cat > sample_logs/errors.log << 'EOF'
2024-10-18 09:45:12 ERROR Failed to load model: File not found
2024-10-18 10:04:15 ERROR CUDA out of memory: Tried to allocate 2.50 GiB
2024-10-18 10:08:12 ERROR Invalid input shape: expected (224,224,3), got (256,256,3)
2024-10-18 11:23:45 ERROR Database connection failed: Connection refused
2024-10-18 12:15:30 ERROR Disk I/O error: No space left on device
EOF
```

## 第 2 部分：使用 grep 过滤日志

### 步骤 2：查找错误和模式

```bash
cd ~/log-analysis-lab

cat > grep_logs.sh << 'EOF'
#!/bin/bash
LOG_DIR="sample_logs"

echo "=== Find all ERROR messages ==="
grep "ERROR" $LOG_DIR/training.log

echo ""
echo "=== Count errors ==="
echo "Total errors: $(grep -c "ERROR" $LOG_DIR/errors.log)"

echo ""
echo "=== Find errors with context ==="
grep -C 2 "ERROR" $LOG_DIR/training.log

echo ""
echo "=== Case-insensitive search ==="
grep -i "cuda" $LOG_DIR/training.log

echo ""
echo "=== Invert match (exclude INFO) ==="
grep -v "INFO" $LOG_DIR/training.log

echo ""
echo "=== Extract epoch numbers ==="
grep -o "Epoch [0-9]*" $LOG_DIR/training.log
EOF

chmod +x grep_logs.sh
./grep_logs.sh
```

## 第 3 部分：使用 awk 的高级日志解析

### 步骤 3：解析训练指标

```bash
cd ~/log-analysis-lab

cat > awk_logs.sh << 'EOF'
#!/bin/bash
LOG_DIR="sample_logs"

echo "=== Count log levels ==="
awk '{print $3}' $LOG_DIR/training.log | sort | uniq -c

echo ""
echo "=== Extract training metrics ==="
awk '/Epoch [0-9]+\/100/ {
    printf "Epoch %s - Loss: %s - Accuracy: %s\n", $5, $8, $10
}' $LOG_DIR/training.log

echo ""
echo "=== Average response time ==="
awk -F'Duration: |ms' '/Duration/ {
    sum += $(NF-1); count++
} END {
    if (count > 0) print "Average response:", sum/count "ms"
}' $LOG_DIR/api.log

echo ""
echo "=== Count errors by type ==="
awk -F': ' '/ERROR/ {print $2}' $LOG_DIR/errors.log | sort | uniq -c | sort -rn
EOF

chmod +x awk_logs.sh
./awk_logs.sh
```

### 步骤 4：提取和可视化指标

```bash
cd ~/log-analysis-lab

cat > extract_metrics.sh << 'EOF'
#!/bin/bash
LOG_FILE="sample_logs/training.log"
OUTPUT_CSV="training_metrics.csv"

echo "epoch,loss,accuracy,val_loss,val_accuracy" > $OUTPUT_CSV

awk '/Epoch [0-9]+\/100/ {
    match($0, /loss: ([0-9.]+)/, loss);
    match($0, /accuracy: ([0-9.]+)/, acc);
    match($0, /val_loss: ([0-9.]+)/, val_loss);
    match($0, /val_accuracy: ([0-9.]+)/, val_acc);
    printf "%d,%.4f,%.4f,%.4f,%.4f\n", NR, loss[1], acc[1], val_loss[1], val_acc[1]
}' $LOG_FILE >> $OUTPUT_CSV

echo "Metrics saved to: $OUTPUT_CSV"
cat $OUTPUT_CSV
EOF

chmod +x extract_metrics.sh
./extract_metrics.sh
```

## 第 4 部分：错误模式分析

### 步骤 5：识别常见问题

```bash
cd ~/log-analysis-lab

cat > analyze_errors.sh << 'EOF'
#!/bin/bash
LOG_DIR="sample_logs"

echo "=== Error Analysis Report ==="
echo "Total errors: $(grep -c "ERROR" $LOG_DIR/*.log 2>/dev/null || echo 0)"

echo ""
echo "=== Top Error Types ==="
grep "ERROR" $LOG_DIR/*.log | sed 's/.*ERROR //' | cut -d':' -f1 | sort | uniq -c | sort -rn | head -5

echo ""
echo "=== Critical Issues ==="
grep -E "CUDA|memory|disk|space" -i $LOG_DIR/*.log
EOF

chmod +x analyze_errors.sh
./analyze_errors.sh
```

## 第 5 部分：系统日志分析

### 步骤 6：使用 journalctl

```bash
cd ~/log-analysis-lab

# journalctl 参考
cat > journalctl_reference.md << 'EOF'
# journalctl Reference

journalctl                       View all logs
journalctl -f                    Follow logs
journalctl -u service            Service-specific logs
journalctl --since "1 hour ago"  Recent logs
journalctl -n 100                Last 100 lines
journalctl -p err                Errors only
journalctl -b                    Current boot
journalctl --disk-usage          Check disk usage
EOF
```

## 第 6 部分：日志轮转

### 步骤 7：配置日志轮转

```bash
cd ~/log-analysis-lab

cat > logrotate_example.conf << 'EOF'
/var/log/ml/training/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 0640 mluser mluser
    postrotate
        systemctl reload ml-training || true
    endscript
}

/var/log/ml/api/*.log {
    size 100M
    rotate 10
    compress
    missingok
    notifempty
}
EOF
```

## 第 7 部分：综合日志分析工具

### 步骤 8：创建日志分析脚本

```bash
cd ~/log-analysis-lab

cat > log_analyzer.sh << 'EOF'
#!/bin/bash
set -euo pipefail

LOG_DIR="${1:-sample_logs}"
REPORT_FILE="analysis_report_$(date +%Y%m%d_%H%M%S).txt"

{
    echo "=== ML Log Analysis Report ==="
    echo "Generated: $(date)"
    echo ""

    echo "### Error Statistics"
    total_errors=$(grep -r "ERROR" "$LOG_DIR" 2>/dev/null | wc -l || echo 0)
    total_warnings=$(grep -r "WARNING" "$LOG_DIR" 2>/dev/null | wc -l || echo 0)
    echo "Total errors: $total_errors"
    echo "Total warnings: $total_warnings"

    echo ""
    echo "### Top Errors"
    grep -rh "ERROR" "$LOG_DIR" 2>/dev/null | sed 's/.*ERROR //' | cut -d':' -f1 | sort | uniq -c | sort -rn | head -10 || echo "No errors"

    echo ""
    echo "### Recommendations"
    if grep -riq "out of memory\|oom" "$LOG_DIR" 2>/dev/null; then
        echo "- Memory issues detected"
    fi
    if grep -riq "cuda.*error" "$LOG_DIR" 2>/dev/null; then
        echo "- CUDA errors detected"
    fi
} | tee "$REPORT_FILE"

echo "Report saved: $REPORT_FILE"
EOF

chmod +x log_analyzer.sh
./log_analyzer.sh
```

## 验证

```bash
cd ~/log-analysis-lab

cat > validate_exercise.sh << 'EOF'
#!/bin/bash

PASS=0; FAIL=0

if [ -d "sample_logs" ] && [ $(ls sample_logs/*.log 2>/dev/null | wc -l) -ge 3 ]; then
    echo "✓ Sample logs created"; ((PASS++))
fi

for script in grep_logs.sh awk_logs.sh extract_metrics.sh analyze_errors.sh log_analyzer.sh; do
    if [ -x "$script" ]; then
        echo "✓ Script: $script"; ((PASS++))
    fi
done

echo "Passed: $PASS, Failed: $FAIL"
EOF

chmod +x validate_exercise.sh
./validate_exercise.sh
```

---

**恭喜！** 你已经掌握了 ML 系统的日志文件分析！
