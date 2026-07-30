# 讲座 07：文本处理工具

## 目录
1. [简介](#简介)
2. [grep：模式匹配](#grep-模式匹配)
3. [sed：流编辑](#sed-流编辑)
4. [awk：文本处理和报告](#awk-文本处理和报告)
5. [使用管道组合工具](#使用管道组合工具)
6. [AI 基础设施应用](#ai-基础设施应用)
7. [总结和要点](#总结和要点)

## 简介

文本处理对 AI 基础设施工程师至关重要。你将分析训练日志、解析指标、处理数据集、从配置文件中提取信息，以及通过搜索海量日志文件来排查问题。三个核心文本处理工具——grep、sed 和 awk——是命令行的强力工具。

本讲座教你掌握这些工具，用于实际的 AI 基础设施任务。

### 学习目标

完成本讲座后，你将能够：
- 使用 grep 高效搜索文件和日志
- 使用 sed 转换文本流
- 使用 awk 处理结构化数据
- 在强大的管道中组合工具
- 将文本处理应用于 ML 日志分析
- 从训练指标中提取见解
- 通过日志分析调试基础设施问题

### 先决条件
- 讲座 01-06（从 Linux 基础到高级脚本）
- 熟悉命令行基础
- 理解正则表达式（有帮助但不是必需的）

### 为什么 AI 基础设施需要文本处理？

**日志分析**：从训练日志中提取错误、警告和指标
**指标处理**：解析和分析模型性能数据
**配置管理**：以编程方式修改配置文件
**数据预处理**：清洗和转换数据集
**故障排查**：通过搜索日志来调试问题
**自动化**：在脚本和管道中处理文本
**报告生成**：从原始数据生成摘要

**时长**：90 分钟
**难度**：中级

---

## grep：模式匹配

**grep**（全局正则表达式打印）在文本中搜索模式。

### grep 基本语法

```bash
grep [OPTIONS] PATTERN [FILE...]
```

### 简单模式搜索

```bash
# 在文件中搜索"error"
grep error training.log

# 搜索多个文件
grep error *.log

# 递归搜索所有文件
grep -r error /var/log/

# 从 stdin 搜索（管道）
cat training.log | grep error
```

### 基本 grep 选项

**-i**：不区分大小写搜索
```bash
# 查找"error"、"Error"、"ERROR"等
grep -i error training.log
```

**-v**：反向匹配（显示不匹配的行）
```bash
# 显示不带"DEBUG"的行
grep -v DEBUG application.log
```

**-n**：显示行号
```bash
# 显示匹配的行号
grep -n error training.log
# 输出：42:ERROR: Model failed to load
```

**-c**：计数匹配行
```bash
# 计算错误行数
grep -c error training.log
# 输出：15
```

**-l**：列出包含匹配的文件（不显示匹配本身）
```bash
# 哪些日志文件包含错误？
grep -l error *.log
# 输出：training.log api.log
```

**-r**：递归搜索
```bash
# 搜索目录树中的所有文件
grep -r "cuda error" /var/log/
```

**-A、-B、-C**：上下文行
```bash
# 显示匹配后 3 行
grep -A 3 "ERROR" training.log

# 显示匹配前 3 行
grep -B 3 "ERROR" training.log

# 显示匹配前后各 3 行
grep -C 3 "ERROR" training.log
```

**-w**：仅匹配完整单词
```bash
# 匹配"train"但不匹配"training"或"trainer"
grep -w train commands.log
```

**-o**：仅显示匹配的部分
```bash
# 仅提取匹配的文本
grep -o "[0-9]\+\.[0-9]\+" metrics.log
# 输出：0.95
#         0.87
#         0.92
```

### grep 的正则表达式

**基本正则表达式模式**：
```bash
# . - 任意单个字符
grep "model.h5" files.log

# * - 前一个字符出现零次或多次
grep "erro*r" log.txt     # er、error、errrror

# ^ - 行首
grep "^ERROR" log.txt     # 以 ERROR 开头的行

# $ - 行尾
grep "success$" log.txt   # 以 success 结尾的行

# [...] - 字符类
grep "[Ee]rror" log.txt   # Error 或 error
```

**扩展正则表达式**（使用 -E 或 egrep）：
```bash
# + - 一次或多次
grep -E "erro+r" log.txt

# ? - 零次或一次
grep -E "colou?r" log.txt  # color 或 colour

# | - OR
grep -E "error|warning" log.txt

# () - 分组
grep -E "(failed|error)" log.txt
```

### AI/ML 日志分析示例

**查找训练错误**：
```bash
# 所有错误行
grep -i error training.log

# 特别是 CUDA 错误
grep -i "cuda error" training.log

# 内存不足错误
grep -i "out of memory" training.log
grep -i "oom" training.log
```

**提取指标**：
```bash
# 查找包含 accuracy 的行
grep "accuracy" training.log

# 包含 epoch 信息的行
grep "Epoch [0-9]\+/" training.log

# 损失值
grep -o "loss: [0-9]\+\.[0-9]\+" training.log
```

**按严重级别过滤**：
```bash
# 仅错误和严重
grep -E "ERROR|CRITICAL" application.log

# 排除调试消息
grep -v DEBUG application.log
```

**查找特定 GPU 问题**：
```bash
# GPU 内存不足
grep -i "gpu.*memory" training.log

# GPU 未找到
grep -i "gpu.*not found" training.log

# CUDA 版本不匹配
grep -i "cuda.*version" training.log
```

### 实际示例：分析训练日志

```bash
#!/bin/bash
# analyze-training-log.sh

LOG_FILE="training.log"

echo "=== Training Log Analysis ==="
echo ""

# 总错误数
ERROR_COUNT=$(grep -c ERROR "$LOG_FILE" || echo 0)
echo "Total errors: $ERROR_COUNT"

# 警告数
WARN_COUNT=$(grep -c WARNING "$LOG_FILE" || echo 0)
echo "Total warnings: $WARN_COUNT"

# 已完成的 epoch 数
EPOCHS=$(grep -c "Epoch [0-9]\+/" "$LOG_FILE")
echo "Epochs completed: $EPOCHS"

# CUDA 错误
CUDA_ERRORS=$(grep -c "cuda error" "$LOG_FILE" || echo 0)
if [ $CUDA_ERRORS -gt 0 ]; then
    echo ""
    echo "CUDA errors found: $CUDA_ERRORS"
    grep "cuda error" "$LOG_FILE"
fi

# OOM 错误
OOM_ERRORS=$(grep -c -i "out of memory" "$LOG_FILE" || echo 0)
if [ $OOM_ERRORS -gt 0 ]; then
    echo ""
    echo "Out of memory errors: $OOM_ERRORS"
    grep -i "out of memory" "$LOG_FILE"
fi

# 最后 5 个错误
echo ""
echo "Last 5 errors:"
grep ERROR "$LOG_FILE" | tail -5
```

---

## sed：流编辑

**sed**（流编辑器）对流执行文本转换。

### sed 基本语法

```bash
sed [OPTIONS] 'COMMAND' [FILE...]
```

### 替换（最常见用途）

**基本替换**：
```bash
# 替换每行第一个匹配
sed 's/old/new/' file.txt

# 替换所有匹配（g = 全局）
sed 's/old/new/g' file.txt

# 仅匹配特定模式的行
sed '/pattern/s/old/new/g' file.txt
```

**替换标志**：
```bash
# g - 全局（行中所有匹配）
sed 's/error/ERROR/g' file.txt

# i - 不区分大小写
sed 's/error/ERROR/gi' file.txt

# 2 - 仅第二个匹配
sed 's/old/new/2' file.txt

# p - 打印发生替换的行
sed -n 's/error/ERROR/p' file.txt
```

### 原地编辑

**原地编辑文件**（-i 标志）：
```bash
# 原地编辑文件
sed -i 's/old/new/g' file.txt

# 创建备份（.bak 扩展名）
sed -i.bak 's/old/new/g' file.txt

# macOS 需要空字符串表示无备份
sed -i '' 's/old/new/g' file.txt  # macOS
```

### 行操作

**删除行**：
```bash
# 删除包含模式的行
sed '/pattern/d' file.txt

# 删除第 5-10 行
sed '5,10d' file.txt

# 删除第一行
sed '1d' file.txt

# 删除最后一行
sed '$d' file.txt

# 删除空行
sed '/^$/d' file.txt

# 删除以 # 开头的行
sed '/^#/d' file.txt
```

**打印行**：
```bash
# 打印特定行（使用 -n 抑制默认输出）
sed -n '5,10p' file.txt      # 第 5-10 行
sed -n '1p' file.txt          # 第一行
sed -n '$p' file.txt          # 最后一行

# 打印匹配模式的行
sed -n '/pattern/p' file.txt
```

**插入和追加**：
```bash
# 在第 5 行前插入
sed '5i\New line here' file.txt

# 在第 5 行后追加
sed '5a\New line here' file.txt
```

### 高级替换

**使用不同分隔符**（对路径很有用）：
```bash
# 替代 s/\/old\/path/\/new\/path/g
# 使用 | 或 # 作为分隔符
sed 's|/old/path|/new/path|g' file.txt
sed 's#/old/path#/new/path#g' file.txt
```

**多命令**：
```bash
# 使用 -e 执行多个命令
sed -e 's/old1/new1/g' -e 's/old2/new2/g' file.txt

# 或使用分号
sed 's/old1/new1/g; s/old2/new2/g' file.txt
```

### AI/ML 配置示例

**更新模型路径**：
```bash
# 在配置中更新模型路径
sed -i 's|/models/old|/models/new|g' config.yaml
```

**修改超参数**：
```bash
# 更改学习率
sed -i 's/learning_rate: 0.001/learning_rate: 0.0001/' config.yaml

# 更新批处理大小
sed -i 's/batch_size: 32/batch_size: 64/' config.yaml
```

**清除日志文件**：
```bash
# 删除 ANSI 颜色代码
sed 's/\x1b\[[0-9;]*m//g' colored.log > clean.log

# 删除时间戳
sed 's/^[0-9-]\+ [0-9:]\+ //' log.txt
```

### 实际示例：配置文件更新脚本

```bash
#!/bin/bash
# update-training-config.sh

CONFIG_FILE="training_config.yaml"
BACKUP_FILE="${CONFIG_FILE}.backup"

# 创建备份
cp "$CONFIG_FILE" "$BACKUP_FILE"

echo "Updating training configuration..."

# 更新学习率
sed -i 's/learning_rate: 0.001/learning_rate: 0.0001/' "$CONFIG_FILE"
echo "✓ Learning rate updated to 0.0001"

# 更新批处理大小
sed -i 's/batch_size: 32/batch_size: 64/' "$CONFIG_FILE"
echo "✓ Batch size updated to 64"

# 更新模型路径
sed -i 's|model_path: /models/v1|model_path: /models/v2|' "$CONFIG_FILE"
echo "✓ Model path updated to /models/v2"

# 启用混合精度
sed -i 's/mixed_precision: false/mixed_precision: true/' "$CONFIG_FILE"
echo "✓ Mixed precision enabled"

echo ""
echo "Configuration updated successfully"
echo "Backup saved to: $BACKUP_FILE"
```

---

## awk：文本处理和报告

**awk** 是一种强大的模式扫描和处理语言。

### awk 基本语法

```bash
awk 'PATTERN {ACTION}' file.txt
```

### 字段处理

**默认字段分隔符是空白**：
```bash
# 打印第一个字段
echo "one two three" | awk '{print $1}'
# 输出：one

# 打印多个字段
echo "one two three" | awk '{print $1, $3}'
# 输出：one three

# 打印所有字段
echo "one two three" | awk '{print $0}'
# 输出：one two three

# 打印最后一个字段
echo "one two three" | awk '{print $NF}'
# 输出：three
```

**自定义字段分隔符**（-F 标志）：
```bash
# CSV 文件（逗号分隔）
echo "name,age,city" | awk -F',' '{print $1, $2}'
# 输出：name age

# 冒号分隔（/etc/passwd）
awk -F':' '{print $1, $3}' /etc/passwd
```

### 内置变量

**常用 awk 变量**：
```bash
# NR - 记录数（当前行号）
awk '{print NR, $0}' file.txt

# NF - 字段数（当前行的字段数）
awk '{print NF}' file.txt

# FS - 字段分隔符（输入）
awk -v FS=',' '{print $1}' file.txt

# OFS - 输出字段分隔符
awk 'BEGIN {OFS=","} {print $1, $2}' file.txt
```

### 模式匹配

```bash
# 打印匹配模式的行
awk '/pattern/' file.txt

# 如果字段匹配则打印
awk '$1 == "value"' file.txt

# 数字比较
awk '$3 > 100' file.txt

# 多条件
awk '$1 == "ERROR" && $3 > 100' file.txt

# 正则匹配
awk '$2 ~ /pattern/' file.txt
```

### 计算和报告

**算术运算**：
```bash
# 对第 3 列的值求和
awk '{sum += $3} END {print sum}' numbers.txt

# 平均值
awk '{sum += $1; count++} END {print sum/count}' numbers.txt

# 查找最大值
awk 'BEGIN {max=0} {if($1>max) max=$1} END {print max}' numbers.txt
```

**计数和分组**：
```bash
# 计数出现次数
awk '{count[$1]++} END {for(word in count) print word, count[word]}' file.txt

# 按类别求和
awk '{sum[$1] += $2} END {for(cat in sum) print cat, sum[cat]}' file.txt
```

### AI/ML 日志分析中的 awk

**提取训练指标**：
```bash
# 提取 epoch 和 loss
awk '/Epoch/ {print $2, $NF}' training.log

# 从特定格式提取 accuracy
awk -F': ' '/acc:/ {print $NF}' training.log

# 计算平均损失
awk '/loss:/ {sum += $NF; count++} END {print "Avg loss:", sum/count}' training.log
```

**解析 GPU 指标**：
```bash
# nvidia-smi 输出处理
nvidia-smi --query-gpu=index,utilization.gpu,memory.used \
  --format=csv,noheader | \
  awk -F, '{print "GPU " $1 ": " $2 " util, " $3 " memory"}'
```

**分析 API 日志**：
```bash
# 按端点计数请求
awk '{print $7}' access.log | sort | uniq -c | sort -rn

# 平均响应时间
awk '{sum += $10; count++} END {print "Avg response:", sum/count "ms"}' api.log
```

### 高级 awk 特性

**BEGIN 和 END 块**：
```bash
awk '
BEGIN {
    print "Starting processing..."
    sum = 0
}
{
    sum += $1
}
END {
    print "Total:", sum
    print "Average:", sum/NR
}
' numbers.txt
```

**条件操作**：
```bash
awk '
{
    if ($3 > 90)
        print $1, "excellent"
    else if ($3 > 70)
        print $1, "good"
    else
        print $1, "needs improvement"
}
' scores.txt
```

### 实际示例：训练指标报告

```bash
#!/bin/bash
# generate-training-report.sh

LOG_FILE="training.log"

echo "=== Training Metrics Report ==="
echo ""

# 提取指标并生成报告
awk '
BEGIN {
    epoch_count = 0
    total_loss = 0
    total_acc = 0
    min_loss = 999999
    max_acc = 0
}

# 匹配 epoch 行
/Epoch [0-9]+\// {
    for(i=1; i<=NF; i++) {
        if($i ~ /loss:/) {
            loss = $(i+1)
            total_loss += loss
            if(loss < min_loss) min_loss = loss
        }
        if($i ~ /acc:/) {
            acc = $(i+1)
            total_acc += acc
            if(acc > max_acc) max_acc = acc
        }
    }
    epoch_count++
}

END {
    if(epoch_count > 0) {
        print "Total Epochs:", epoch_count
        print "Average Loss:", total_loss/epoch_count
        print "Average Accuracy:", total_acc/epoch_count
        print "Best Loss:", min_loss
        print "Best Accuracy:", max_acc
        print ""
        print "Training Trend:"
        if(min_loss < 0.1)
            print "  Loss: Excellent convergence"
        else if(min_loss < 0.5)
            print "  Loss: Good convergence"
        else
            print "  Loss: May need more training"

        if(max_acc > 0.95)
            print "  Accuracy: Excellent performance"
        else if(max_acc > 0.80)
            print "  Accuracy: Good performance"
        else
            print "  Accuracy: Needs improvement"
    } else {
        print "No epoch data found in log file"
    }
}
' "$LOG_FILE"
```

---

## 使用管道组合工具

真正的力量来自在管道中组合 grep、sed 和 awk。

### 简单管道

**过滤然后提取**：
```bash
# 查找错误，提取时间戳
grep ERROR log.txt | awk '{print $1, $2}'

# 查找 GPU 行，提取 GPU ID
grep GPU log.txt | sed 's/.*GPU \([0-9]\+\).*/\1/'

# 统计错误类型
grep ERROR log.txt | awk '{print $3}' | sort | uniq -c
```

**转换然后分析**：
```bash
# 清洗数据，然后计算
sed 's/[^0-9.]//g' metrics.txt | awk '{sum+=$1} END {print sum}'

# 提取值，排序，显示前 10
grep "accuracy:" log.txt | awk '{print $NF}' | sort -rn | head -10
```

### 复杂分析管道

**分析训练性能**：
```bash
#!/bin/bash
# 分析最近 24 小时的训练

grep "$(date +%Y-%m-%d)" training.log | \
grep "Epoch" | \
awk -F': ' '{print $2, $4}' | \
awk '{loss+=$1; acc+=$2; count++} END {
    print "Last 24 hours:"
    print "  Epochs:", count
    print "  Avg Loss:", loss/count
    print "  Avg Accuracy:", acc/count
}'
```

**查找瓶颈**：
```bash
#!/bin/bash
# 查找最慢的训练步骤

grep "step_time" training.log | \
awk '{print $5}' | \
sort -rn | \
head -10 | \
awk '{
    sum+=$1
    print "  Slow step:", $1 "s"
}
END {
    print "  Total time in slow steps:", sum "s"
}'
```

### 高级管道：日志分析工具

```bash
#!/bin/bash
# comprehensive-log-analysis.sh

LOG_FILE="$1"

if [ -z "$LOG_FILE" ]; then
    echo "Usage: $0 <log_file>"
    exit 1
fi

echo "=== Comprehensive Log Analysis ==="
echo "File: $LOG_FILE"
echo ""

# 时间线分析
echo "## Timeline"
grep -E "ERROR|WARN" "$LOG_FILE" | \
awk '{print $1, $2}' | \
awk -F: '{print $1":"$2}' | \
sort | uniq -c | \
awk '{printf "  %s: %d events\n", $2, $1}'
echo ""

# 错误分析
echo "## Error Distribution"
grep ERROR "$LOG_FILE" | \
awk '{
    for(i=3; i<=NF; i++)
        if($i ~ /^[A-Z]/)
            errors[$i]++
}
END {
    for(e in errors)
        printf "  %s: %d occurrences\n", e, errors[e]
}' | \
sort -t: -k2 -rn | \
head -10
echo ""

# 性能指标
echo "## Performance Metrics"
grep -E "time:|duration:" "$LOG_FILE" | \
sed 's/.*time: \([0-9.]\+\).*/\1/' | \
awk '{
    sum+=$1
    if(NR==1) min=max=$1
    if($1<min) min=$1
    if($1>max) max=$1
    count++
}
END {
    if(count>0) {
        print "  Count:", count
        printf "  Average: %.2fs\n", sum/count
        printf "  Min: %.2fs\n", min
        printf "  Max: %.2fs\n", max
    }
}'
echo ""
```

---

## 总结和要点

### 掌握的工具

**grep - 模式匹配**：
- 在文件中搜索模式
- 正则表达式
- 上下文行（-A、-B、-C）
- 递归搜索（-r）
- 计数匹配（-c）
- 不区分大小写（-i）

**sed - 流编辑**：
- 替换（s/old/new/g）
- 原地编辑（-i）
- 删除行（d）
- 插入/追加（i、a）
- 多命令
- 正则表达式

**awk - 文本处理**：
- 字段处理（$1、$2、$NF）
- 模式-动作范式
- 内置变量（NR、NF、FS）
- 算术运算
- 数组和循环
- 报告和摘要

### 关键概念

1. **管道**：链式工具实现强大文本处理
2. **模式**：匹配和提取特定信息
3. **字段**：逐列处理结构化数据
4. **转换**：以编程方式修改文本
5. **聚合**：计算统计信息和摘要

### AI 基础设施应用

**日志分析**：
- 提取错误和警告
- 解析训练指标
- 识别性能瓶颈
- 监控资源使用情况

**数据处理**：
- 清洗数据集
- 转换格式
- 提取特征
- 生成报告

**自动化**：
- 在脚本中处理日志
- 生成告警
- 创建仪表盘
- 自动化报告

### 最佳实践

✅ **从简单开始**：先使用基本 grep，再使用复杂 awk
✅ **增量测试**：逐步构建管道
✅ **使用合适的工具**：grep 用于搜索，sed 用于替换，awk 用于报告
✅ **引用模式**：防止 shell 解释
✅ **保存中间结果**：调试复杂管道
✅ **注释正则表达式**：记录复杂模式
✅ **使用 -n 配合 sed**：显式打印以保持清晰
✅ **命名 awk 变量**：可读性优于简洁

### 常见模式

```bash
# 搜索并计数
grep pattern file.txt | wc -l

# 提取并分析
grep pattern file.txt | awk '{print $3}' | sort | uniq -c

# 过滤并转换
grep ERROR log.txt | sed 's/ERROR://' | awk '{print $1, $2}'

# 计算指标
awk '/metric/ {sum+=$2; count++} END {print sum/count}' log.txt
```

### 快速参考

```bash
# grep
grep pattern file.txt           # 搜索
grep -i pattern file.txt        # 不区分大小写
grep -r pattern dir/            # 递归
grep -n pattern file.txt        # 行号
grep -A 3 pattern file.txt      # 之后 3 行
grep -E 'pat1|pat2' file.txt    # 扩展正则

# sed
sed 's/old/new/g' file.txt      # 全部替换
sed -i 's/old/new/g' file.txt   # 原地编辑
sed '/pattern/d' file.txt       # 删除行
sed -n '10,20p' file.txt        # 打印行 10-20

# awk
awk '{print $1}' file.txt       # 第一个字段
awk -F: '{print $1}' file.txt   # 自定义分隔符
awk '$3 > 100' file.txt         # 过滤
awk '{sum+=$1} END {print sum}' # 求列和

# 管道
grep ERROR log.txt | awk '{print $1}' | sort | uniq -c
```

---

**讲座 07：文本处理工具结束**

你现在已经掌握了强大的文本处理技能，用于分析日志、处理数据和自动化 AI 基础设施任务。这些工具对于日常运维是必不可少的。

**下一步**：讲座 08 - 网络基础
