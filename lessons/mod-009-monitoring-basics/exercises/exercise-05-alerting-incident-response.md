# Exercise 05：面向 AI 基础设施的告警与事件响应 Runbook

## 概述
结合 metrics、logs 和 tracing，为推理平台实现一个强大的告警管道和事件响应工作流。你将配置 Alertmanager/Grafana 告警路由、开发 runbooks、模拟事件（延迟峰值、数据漂移、日志中断），并进行小型事后审查以巩固运维就绪度。

**难度：** 高级
**预计时间：** 3–4 小时
**前置要求：**
- Exercises 01–04 已完成
- Lecture 04（日志、告警、ML 监控）
- Alertmanager 或 Grafana OnCall 可用
- 可访问协作工具（Slack/Teams）用于通知

## 学习目标
完成本练习后，你将：
1. 实现与 SLO 和团队结构对齐的告警路由策略
2. 在 Prometheus/Grafana 中配置多窗口 burn rate 告警和基于日志的告警
3. 使用可观测性工具模拟事件并执行 runbooks
4. 生成事件时间线和事后审查，记录经验教训
5. 为可观测性和 on-call 流程建立持续改进积压
6. 为 ML 系统构建生产就绪的告警基础设施
7. 通过逼真的模拟开发事件响应的肌肉记忆

## 你将构建的内容

完成本练习后，你将拥有：

- **完整的告警栈**：Alertmanager 带多渠道路由（PagerDuty、Slack、Email）
- **生产告警规则**：基于 SLO 的多窗口 burn rate 告警、error budgets、漂移检测
- **专业的 Runbooks**：5+ 个遵循 Google SRE 最佳实践的详细 runbooks
- **事件模拟框架**：用于触发逼真事件的脚本
- **事后审查流程**：模板和完成的示例
- **持续改进系统**：积压跟踪和 metrics

这直接构建在 Exercises 01-04 的可观测性基础之上，完成你的生产就绪 ML monitoring 栈。

## 第 1 部分：告警路由架构

### 1.1 目录结构设置

为告警和事件管理创建以下目录结构：

```bash
mkdir -p alerting/{config,rules,runbooks,incidents,docs}
cd alerting/

# 创建子目录
mkdir -p config/{alertmanager,grafana}
mkdir -p rules/{prometheus,grafana}
mkdir -p runbooks/{inference,platform,ml-operations}
mkdir -p incidents/{2025-01,templates}
mkdir -p docs/policies
```

### 1.2 告警策略设计

创建 `docs/policies/alerting-policy.md`：

```markdown
# AI 基础设施告警策略

## 告警严重级别

### 严重（寻呼）
- **影响**：服务降级或完全中断影响用户
- **示例**：
  - SLO burn rate 快速消耗 error budget（多窗口告警）
  - 推理服务完全宕机（100% 错误率）
  - 阻止模型更新的数据管道故障
- **响应时间**：5 分钟
- **通知**：PagerDuty（立即寻呼）
- **升级**：10 分钟未确认后自动升级
- **需要 On-Call**：是

### 高（紧急）
- **影响**：潜在服务降级、error budget 消耗
- **示例**：
  - 错误率升高（5-10%）
  - 延迟降级（P95 > 200ms 但未达到 SLO 违规）
  - 检测到模型漂移（PSI > 0.2）
  - 基础设施告警（高 CPU、磁盘空间警告）
- **响应时间**：工作时间内 15 分钟
- **通知**：Slack #ai-infra-urgent + 邮件
- **需要 On-Call**：工作时间内需要

### 中（警告）
- **影响**：早期预警信号、容量规划
- **示例**：
  - 中等错误率增长（2-5%）
  - Feature store 缓存未命中率升高
  - 日志量峰值
  - Prometheus scrape 失败
- **响应时间**：1 小时
- **通知**：Slack #ai-infra-alerts
- **需要 On-Call**：否

### 低（信息）
- **影响**：信息性，无需立即操作
- **示例**：
  - 计划维护通知
  - 部署通知
  - 容量规划指标
- **响应时间**：尽力而为
- **通知**：Slack #ai-infra-info
- **需要 On-Call**：否

## 升级链

### 主要 On-Call
- **响应时间**：严重 5 分钟，高 15 分钟
- **职责**：
  - 确认告警
  - 执行 runbooks
  - 如果 30 分钟内无法解决则升级

### 二级 On-Call
- **升级触发条件**：10 分钟未确认，或主要人员请求后备
- **职责**：
  - 支持主要 on-call
  - 主要人员不可用时接管

### 事件指挥官（经理/技术主管）
- **升级触发条件**：事件持续时间 > 1 小时或多服务影响
- **职责**：
  - 协调跨团队响应
  - 外部沟通
  - 主持事后审查

### 领域专家
- **ML 工程**：模型特定问题、漂移、性能
- **平台工程**：Kubernetes、基础设施、扩展
- **数据工程**：特征存储、数据管道

## 路由矩阵

| 告警类型 | 严重级别 | 主要渠道 | 二级渠道 | 升级 |
|------------|----------|-----------------|-------------------|------------|
| SLO Burn Rate | 严重 | PagerDuty | Slack #incidents | 10 分钟 |
| 错误率高 | 严重 | PagerDuty | Slack #incidents | 10 分钟 |
| 延迟违规 | 高 | Slack #ai-infra-urgent | 邮件 | 30 分钟 |
| 模型漂移 | 高 | Slack #ai-infra-urgent | 邮件 | 1 小时 |
| 特征错误 | 中 | Slack #ai-infra-alerts | - | - |
| 基础设施 | 中 | Slack #ai-infra-alerts | - | - |
| 部署 | 低 | Slack #ai-infra-info | - | - |

## 维护窗口

### 计划内维护
- **通知**：提前 48 小时在 #ai-infra-info 中通知
- **告警静默**：通过 Alertmanager silences API 自动静默
- **时长**：通常 2-4 小时
- **批准**：工程经理 + 产品负责人

### 紧急维护
- **通知**：立即通知 #incidents
- **告警静默**：手动静默并附带事件工单引用
- **时长**：根据需要
- **批准**：On-call 工程师权限

## 自动静默规则

```yaml
# 在已知部署窗口期间静默告警
- matchers:
    - deployment_type = "scheduled"
  duration: 2h
  created_by: "ci/cd-pipeline"

# 静默测试环境夜间告警
- matchers:
    - environment = "test"
    - time_range = "22:00-06:00 UTC"
  duration: 8h
  created_by: "automation"
```

## On-Call 轮换

- **轮换时长**：1 周（周一 9am - 周一 9am）
- **主要/二级**：始终同时分配两者
- **交接**：周一早上同步（15 分钟）
- **PTO 覆盖**：提前 1 周安排调换
- **补偿**：每 on-call 周额外 1 天 PTO
```

### 1.3 完整 Alertmanager 配置

创建 `config/alertmanager/alertmanager.yml`。

**关键特性**：
- 多接收者路由（PagerDuty、Slack 频道、email）
- 抑制规则减少告警噪音
- 带按钮的自定义 Slack 格式化
- 按时段路由（工作时间）
- 时间间隔定义

### 1.4 Secrets 管理

创建用于 Kubernetes 的 `config/alertmanager/secrets.yaml` 和用于 Docker Compose 的 `.env.alertmanager`。

### 1.5 Alertmanager 部署

**验证：**

```bash
# 启动 Alertmanager
docker-compose up -d alertmanager

# 检查 Alertmanager UI
open http://localhost:9093

# 检查配置是否有效
docker exec alertmanager amtool check-config /etc/alertmanager/alertmanager.yml

# 查看当前告警
docker exec alertmanager amtool alert query

# 测试创建静默
docker exec alertmanager amtool silence add alertname=Test --duration=1h --comment="Testing silence"
```

**✅ 检查点：** Alertmanager 已部署并运行。

---

## 第 2 部分：告警实现

### 2.1 Metrics 告警（Prometheus）

创建 `rules/prometheus/inference-alerts.yml`。

**关键告警规则：**
- **InferenceLatencyBurnRateCritical** - 快速 burn（5m/1h 窗口），14.4x 阈值
- **InferenceLatencyBurnRateHigh** - 中等 burn（1h/6h 窗口），6x 阈值
- **InferenceErrorRateCritical** - 可用性 SLO burn rate
- **InferenceErrorRateHigh** - 错误率 > 5%
- **ModelLoadFailure** - 模型加载失败
- **FeatureStoreErrorsHigh** - 特征存储错误率 > 10%
- **GPUUtilizationLow** - GPU 利用率 < 20%
- **GPUMemoryHigh** - GPU 内存 > 90%
- **PrometheusScrapeFailure** - Scrape 失败
- **InferenceLatencyElevated** - 延迟早期预警
- **InferenceRequestRateAnomalous** - 请求率异常

### 2.2 基于日志的告警（Grafana 告警）

创建 `rules/grafana/log-alerts.json`。

**关键告警：**
- **FeatureNotFound 错误高** - 基于日志的模式检测
- **日志摄取中断** - 使用 `absent_over_time` 检测
- **错误日志率峰值** - 错误率阈值
- **日志中检测到 PII（安全）** - 合规告警

### 2.3 数据/模型漂移告警

创建 `rules/prometheus/ml-monitoring-alerts.yml`。

**关键 ML 告警：**
- **FeatureDriftDetected** - PSI > 0.2
- **PredictionDriftDetected** - 预测分布偏移
- **ModelPerformanceDegraded** - AUC-ROC 下降
- **MissingFeaturesHigh** - 特征缺失率 > 10%
- **ModelFairnessViolation** - 人口均等偏差 > 0.1
- **InputValidationFailuresHigh** - 输入验证失败率 > 5%

**✅ 检查点：** 所有告警规则已创建并验证。

---

## 第 3 部分：Runbook 开发

### 3.1 Runbook 模板

所有 runbook 应遵循此标准结构以确保一致性：

创建 `runbooks/templates/runbook-template.md`：

```markdown
# Runbook：[告警名称]

**告警名称**：`AlertNameFromPrometheus`
**严重级别**：严重 / 高 / 警告
**团队**：ML Platform / Data Engineering / Platform
**最后更新**：YYYY-MM-DD

## 概述
此告警指示什么以及为什么重要的简要描述。

## 影响
- **用户影响**：此告警触发时用户的体验
- **业务影响**：收入、SLA、声誉影响
- **范围**：受影响的用户/请求数

## 症状
此问题发生时的可观察迹象：
- 特定指标升高/降低
- 日志模式
- 用户报告的问题

## 分流步骤（5 分钟）

### 1. 验证告警
- [ ] 检查 Grafana dashboard：[链接]
- [ ] 在 Prometheus 中确认指标：[查询]
- [ ] 审查近期变更（部署、配置变更）

### 2. 评估范围
- [ ] 多少实例受影响？
- [ ] 多大比例的流量受影响？
- [ ] 问题是孤立的还是广泛的？

### 3. 快速检查
- [ ] 检查依赖健康：[列出依赖]
- [ ] 审查错误日志：[日志查询]
- [ ] 检查资源利用率

## 调查

### Prometheus 查询
```promql
# 相关 Prometheus 查询列表
```

### Grafana Dashboards
- 主要：[dashboard 链接]
- 二级：[相关 dashboards]

### 日志查询（LogQL）
```logql
# 用于调查的 Loki 查询
```

### 常见根本原因
1. **原因 1**：描述和如何识别
2. **原因 2**：描述和如何识别
3. **原因 3**：描述和如何识别

## 缓解

### 立即操作（< 15 分钟）
逐步停止出血的说明：
1. 操作 1
2. 操作 2
3. 操作 3

### 回滚流程（如果与部署相关）
```bash
# 回滚命令
```

### 扩展响应（如果与容量相关）
```bash
# 扩展资源命令
```

## 升级

### 何时升级
- 30 分钟内未解决问题
- 影响在增加
- 根本原因不明确

### 升级路径
1. **主要**：[姓名/角色] - @slack-handle - [电话]
2. **二级**：[姓名/角色] - @slack-handle - [电话]
3. **经理**：[姓名/角色] - @slack-handle - [电话]

## 事后任务

### 立即（24 小时内）
- [ ] 在事件跟踪器中记录时间线
- [ ] 创建事后审查草稿
- [ ] 识别行动项

### 后续
- [ ] 根本原因分析
- [ ] 实施预防措施
- [ ] 用经验教训更新此 runbook

## 相关 Runbooks
- [相关 runbook 1]
- [相关 runbook 2]

## 参考
- 文档：[链接]
- 架构图：[链接]
- 以往事件：[链接]

---
**Runbook 维护者**：[团队]
**审查周期**：季度
**下次审查**：YYYY-MM-DD
```

### 3.2 推理延迟 Runbook

创建 `runbooks/inference/inference-latency.md`。

包含完整的 SLO 违规调查流程，涵盖：
- GPU 饱和
- 特征存储延迟
- 模型大小/复杂度
- 资源争用
- 流量峰值

**缓解步骤**包括立即扩展、回滚和功能降级模式的命令。

### 3.3 推理错误率 Runbook

创建 `runbooks/inference/inference-error-rate.md`。

涵盖 5 种常见根本原因和服务降级模式。

### 3.4 特征漂移 Runbook

创建 `runbooks/ml-operations/feature-drift.md`。

涵盖 PSI 分析、模型性能评估和重新训练决策。

### 3.5 在告警中链接 Runbooks

```yaml
- alert: InferenceLatencyBurnRateCritical
  annotations:
    summary: "CRITICAL: Inference latency burn rate extremely high"
    description: "..."
    runbook_url: "https://runbooks.example.com/inference-latency"
    runbook_file: "file:///runbooks/inference/inference-latency.md"
    dashboard_url: "http://grafana:3000/d/inference-slo"
```

**最佳实践**：将 runbooks 存储在 Git 仓库中，并与 monitoring 栈一起部署：

```bash
# 将 runbooks 部署为 ConfigMap 以便轻松访问
kubectl create configmap runbooks \
  --from-file=runbooks/ \
  --namespace=monitoring

# 挂载到 Grafana pod 中进行内联查看
```

**✅ 检查点：** 至少创建了 3 个 runbooks。

---

## 第 4 部分：事件模拟

### 4.1 模拟框架设置

创建 `scripts/incident-simulation.sh`：

```bash
#!/bin/bash
# 事件模拟框架
# 目的：触发逼真的生产事件以进行 runbook 练习

set -e

NAMESPACE="monitoring"
DEPLOYMENT="inference-gateway"

# 输出颜色
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

# 函数：等待告警触发
wait_for_alert() {
    local alert_name=$1
    local timeout=${2:-300}  # 默认 5 分钟

    log_info "Waiting for alert '$alert_name' to fire (timeout: ${timeout}s)..."

    local elapsed=0
    while [ $elapsed -lt $timeout ]; do
        # 检查 Alertmanager API 中是否有触发的告警
        if curl -s http://localhost:9093/api/v2/alerts | grep -q "$alert_name"; then
            log_info "✓ Alert '$alert_name' is FIRING!"
            return 0
        fi
        sleep 10
        elapsed=$((elapsed + 10))
        echo -n "."
    done

    log_error "✗ Alert did not fire within ${timeout}s"
    return 1
}

# 清理函数
cleanup() {
    log_info "Cleaning up simulation artifacts..."
    kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE 2>/dev/null || true
    kubectl delete configmap incident-simulation -n $NAMESPACE 2>/dev/null || true
}

trap cleanup EXIT

echo "======================================"
echo "   Incident Simulation Framework"
echo "======================================"
echo ""
echo "Available Scenarios:"
echo "  1. Latency Spike (Scenario A)"
echo "  2. Feature Store Outage (Scenario B)"
echo "  3. Error Rate Spike (Scenario C)"
echo "  4. Data Drift (Scenario D)"
echo "  5. GPU OOM (Scenario E)"
echo ""
read -p "Select scenario (1-5): " scenario

case $scenario in
    1) source ./scenarios/latency-spike.sh ;;
    2) source ./scenarios/feature-store-outage.sh ;;
    3) source ./scenarios/error-rate-spike.sh ;;
    4) source ./scenarios/data-drift.sh ;;
    5) source ./scenarios/gpu-oom.sh ;;
    *) log_error "Invalid scenario"; exit 1 ;;
esac
```

### 4.2 场景 A：延迟峰值模拟

创建 `scripts/scenarios/latency-spike.sh`。

模拟推理延迟突然降级，需要立即响应。

### 4.3 场景 B：特征存储中断模拟

模拟特征存储故障导致推理错误。

### 4.4 场景 C：错误率峰值模拟

模拟 500 错误突然增加。

### 4.5 场景 D：数据漂移检测模拟

模拟逐渐的特征分布偏移。

### 4.6 事件文档模板

创建 `incidents/templates/incident-template.md`：

```markdown
# 事件报告：[标题]

**事件 ID**：INC-YYYYMMDD-HHMM-category
**日期**：YYYY-MM-DD
**严重级别**：[严重/高/中/低]
**状态**：[调查中/缓解中/已解决/已关闭]
**事件指挥官**：[姓名]

## 快速摘要
[1-2 句话描述发生了什么]

## 影响
- **开始时间**：YYYY-MM-DD HH:MM:SS UTC
- **结束时间**：YYYY-MM-DD HH:MM:SS UTC（或"进行中"）
- **持续时间**：X 小时 Y 分钟
- **受影响用户**：[数量或百分比]
- **受影响服务**：[列出服务]
- **SLO 影响**：[消耗的 error budget]

## 时间线

| 时间（UTC） | 事件 | 采取的操作 | 操作人 |
|------------|-------|--------------|----------|
| HH:MM:SS | 事件开始 | [描述] | 系统 |
| HH:MM:SS | 告警触发 | InferenceLatencyHigh | Alertmanager |
| HH:MM:SS | 已确认 | 打开事件频道 | On-call 工程师 |
| HH:MM:SS | 开始调查 | 检查 Grafana dashboard | On-call 工程师 |
| HH:MM:SS | 已识别根本原因 | [描述] | On-call 工程师 |
| HH:MM:SS | 开始缓解 | [操作] | On-call 工程师 |
| HH:MM:SS | 事件已解决 | [解决方案] | On-call 工程师 |
| HH:MM:SS | 已安排事后审查 | - | 事件指挥官 |

## 检测
- **检测方法**：[Prometheus 告警 / 用户报告 / 监控 dashboard]
- **告警名称**：[AlertName]
- **MTTD（平均检测时间）**：[事件开始到检测的时间]
- **检测有效性**：[检测及时吗？适当吗？]

## 根本原因
[详细解释导致事件的原因]

## 解决方案
[描述事件是如何解决的]

**缓解步骤**：
1. [步骤 1]
2. [步骤 2]
3. [步骤 3]

**MTTR（平均修复时间）**：[检测到解决的时间]

## 后续操作

| 行动项 | 负责人 | 优先级 | 截止日期 | 状态 |
|-------------|-------|----------|----------|--------|
| [操作 1] | [姓名] | [P0/P1/P2] | YYYY-MM-DD | [开放/进行中/已完成] |

## 经验教训

### 做得好
1. [项 1]

### 可改进
1. [项 1]

---
**事后审查会议**：[日期/时间]
**与会者**：[列表]
```

**✅ 检查点：** 事件模拟框架已创建并测试。

---

## 第 5 部分：事后审查模板

### 5.1 无指责的事后审查文化

**关键原则**：
- **无指责**：关注系统和流程，而不是个人
- **以学习为中心**：记录发生了什么、为什么以及如何预防
- **面向行动**：具体的行动项，带有负责人和截止日期
- **透明**：在组织内分享经验教训

### 5.2 事后审查章节

模板应涵盖：
- **摘要**：2-3 句话描述发生了什么
- **影响**：受影响的用户、持续时间、SLO 消耗
- **时间线**：事件和操作的详细时间线
- **检测**：MTTD、检测方法有效性
- **响应**：MTTR、缓解操作、runbook 有效性
- **根本原因分析**：什么、为什么、促成因素（5 Whys 方法）
- **做得好**：响应的积极方面
- **可改进**：差距和改进
- **行动项**：具体任务，带有负责人、优先级、截止日期
- **经验教训**：团队的关键收获

### 5.3 完成事后审查

**流程**：
1. 在解决后 48 小时内安排事后审查会议
2. 邀请所有参与者（on-call、升级人员、利益相关者）
3. 使用时间线逐步回顾事件
4. 应用 5 Whys 识别根本原因（而非表面症状）
5. 使用 SMART 目标记录行动项
6. 发布事后审查到团队 wiki/文档
7. 在项目管理系统中跟踪行动项
8. 在每周团队会议中审查行动项完成情况

**5 Whys 示例**：
- **为什么延迟会飙升？** → GPU 队列深度增长到 500
- **为什么 GPU 队列增长？** → 流量比正常增加了 3 倍
- **为什么流量增加？** → 营销活动启动了
- **为什么我们没有主动扩展？** → 没有收到活动的通知
- **为什么没有通知？** → 缺乏工程/营销协调的流程
- **根本原因**：缺乏影响流量事件的跨团队沟通流程
- **行动项**：创建 Slack 频道 + 启动协调流程

**✅ 检查点：** 事后审查流程已建立。

---

## 第 6 部分：持续改进积压

### 6.1 可观测性积压创建

创建 `docs/observability-backlog.md`：

```markdown
# 可观测性与可靠性积压

**最后更新**：2025-01-15
**所有者**：ML Platform Team

## 优先级级别
- **P0（严重）**：阻塞生产，必须尽快修复
- **P1（高）**：对可靠性重要，在 1 个 sprint 内完成
- **P2（中）**：有价值的改进，在 1 个月内完成
- **P3（低）**：锦上添花，积压整理

## 告警改进

| 任务 | 优先级 | 工作量 | 负责人 | 截止日期 | 状态 |
|------|----------|--------|-------|----------|--------|
| 为可用性 SLO 添加多窗口 burn rate 告警 | P1 | 2 天 | @ml-platform | 2025-01-30 | 开放 |
| 实现告警疲劳跟踪（每人每周告警数） | P2 | 3 天 | @platform | 2025-02-15 | 开放 |
| 添加 PagerDuty 升级策略测试 | P2 | 1 天 | @ml-platform | 2025-02-15 | 开放 |
| 创建合成监控/金丝雀检查 | P1 | 5 天 | @ml-platform | 2025-01-30 | 开放 |

## Runbook 改进

| 任务 | 优先级 | 工作量 | 负责人 | 截止日期 | 状态 |
|------|----------|--------|-------|----------|--------|
| 添加 GPU OOM 事件的 runbook | P1 | 2 天 | @ml-platform | 2025-01-30 | 开放 |
| 创建模型注册表中断的 runbook | P2 | 2 天 | @ml-ops | 2025-02-15 | 开放 |
| 添加自动化 runbook 测试（验证命令有效） | P2 | 5 天 | @platform | 2025-02-28 | 开放 |
| Runbook 版本控制和变更日志 | P3 | 1 天 | @ml-platform | TBD | 开放 |

## 监控改进

| 任务 | 优先级 | 工作量 | 负责人 | 截止日期 | 状态 |
|------|----------|--------|-------|----------|--------|
| 自动化漂移检测管道（每日 PSI 计算） | P1 | 5 天 | @ml-ops | 2025-01-31 | 开放 |
| 添加 GPU 内存泄漏检测 | P1 | 3 天 | @ml-platform | 2025-01-30 | 开放 |
| 实现金丝雀部署护栏和自动回滚 | P1 | 8 天 | @ml-ops | 2025-02-15 | 开放 |
| 扩展日志脱敏覆盖（信用卡、电话号码） | P1 | 2 天 | @security | 2025-01-25 | 开放 |
| 添加模型性能跟踪（当 ground truth 可用时） | P2 | 5 天 | @ml-eng | 2025-02-28 | 开放 |
| 实现分布式 tracing span 分析 | P2 | 5 天 | @ml-platform | 2025-03-15 | 开放 |

## Dashboard 改进

| 任务 | 优先级 | 工作量 | 负责人 | 截止日期 | 状态 |
|------|----------|--------|-------|----------|--------|
| 创建 on-call dashboard（活跃事件、告警摘要） | P1 | 3 天 | @platform | 2025-01-30 | 开放 |
| 添加 SLO error budget burn-down 图表 | P1 | 2 天 | @ml-platform | 2025-01-30 | 开放 |
| 创建模型比较 dashboard（A/B 测试结果） | P2 | 5 天 | @ml-ops | 2025-02-15 | 开放 |
| 添加成本监控 dashboard（GPU 小时、推理成本） | P2 | 3 天 | @finops | 2025-02-28 | 开放 |

## 流程改进

| 任务 | 优先级 | 工作量 | 负责人 | 截止日期 | 状态 |
|------|----------|--------|-------|----------|--------|
| 实施 on-call 交接检查清单 | P1 | 1 天 | @ml-platform | 2025-01-20 | 开放 |
| 创建事件严重级别分类指南 | P1 | 2 天 | @eng-manager | 2025-01-25 | 开放 |
| 将事后行动项跟踪添加到每周团队会议 | P1 | 1 天 | @eng-manager | 2025-01-18 | 开放 |
| 季度 on-call 回顾流程 | P2 | 2 天 | @eng-manager | 2025-03-31 | 开放 |

## 基础设施改进

| 任务 | 优先级 | 工作量 | 负责人 | 截止日期 | 状态 |
|------|----------|--------|-------|----------|--------|
| 实现 Grafana OnCall 用于轮换管理 | P1 | 3 天 | @platform | 2025-02-15 | 开放 |
| 在维护窗口期间添加自动静默 | P2 | 3 天 | @platform | 2025-02-28 | 开放 |
| 使用 Prometheus `predict_linear` 进行异常检测 | P2 | 5 天 | @ml-platform | 2025-03-15 | 开放 |
| 与 ServiceNow 集成用于事件跟踪 | P3 | 10 天 | @platform | TBD | 开放 |
```

### 6.2 优先级排序框架

使用此框架对积压项进行优先级排序：

**影响 vs 工作量矩阵**：
```
高影响、低工作量 → P0/P1（速赢）
高影响、高工作量 → P1（战略举措）
低影响、低工作量 → P2（填充容量）
低影响、高工作量 → P3（降低优先级）
```

**可靠性影响评分**：
- **MTTD 减少**：这有助于更快检测问题吗？
- **MTTR 减少**：这有助于更快解决问题吗？
- **事件预防**：这能完全预防事件吗？
- **On-Call 生活质量**：这能减少辛劳或告警疲劳吗？
- **客户影响**：这能改善用户体验吗？

每个维度评分 1-5，求和以确定优先级。

**✅ 检查点：** 持续改进积压已创建。

---

## 第 7 部分：验证清单

### 7.1 告警基础设施验证

- [ ] **Alertmanager 已部署**：Alertmanager 运行中且可访问（http://localhost:9093）
- [ ] **告警路由已配置**：使用适当路由实施告警策略
- [ ] **联系人点已测试**：
  - [ ] Slack webhook 已配置和测试（发送测试通知）
  - [ ] PagerDuty 集成密钥已配置
  - [ ] 邮件 SMTP 设置已配置
- [ ] **Secrets 管理**：凭据安全存储（Kubernetes secrets / .env 文件不在 Git 中）
- [ ] **抑制规则正常工作**：验证严重告警抑制警告告警

### 7.2 告警规则验证

- [ ] **SLO Burn Rate 告警**：
  - [ ] 多窗口（5m/1h）burn rate 告警已配置
  - [ ] Burn rate 窗口的 recording rules 存在
  - [ ] 延迟超过 SLO 阈值时告警触发
- [ ] **错误率告警**：
  - [ ] 严重错误率告警已配置（>14.4x burn）
  - [ ] 警告错误率告警已配置（>5%）
  - [ ] 告警包含 runbook 和 dashboard 链接
- [ ] **基于日志的告警**：
  - [ ] Grafana 中配置了 FeatureNotFound 日志告警
  - [ ] 日志摄取中断告警已配置
  - [ ] 检测到日志模式时告警触发
- [ ] **ML 监控告警**：
  - [ ] 特征漂移 PSI 告警已配置
  - [ ] 预测漂移告警已配置
  - [ ] 缺失特征告警已配置

### 7.3 Runbook 验证

- [ ] **Runbooks 已创建**：至少记录了 3 个 runbooks（延迟、错误、漂移）
- [ ] **Runbook 完整性**：每个 runbook 包含：
  - [ ] 影响和症状
  - [ ] 分流步骤（< 5 分钟）
  - [ ] 调查查询（Prometheus、Loki）
  - [ ] 缓解流程（逐步）
  - [ ] 升级联系人
  - [ ] 事后任务
- [ ] **Runbook 链接**：告警注释包含 runbook URL
- [ ] **Runbook 测试**：在模拟期间至少执行 1 个 runbook（验证命令有效）

### 7.4 事件模拟验证

- [ ] **模拟脚本**：事件模拟框架已创建
- [ ] **场景 A（延迟）**：
  - [ ] 延迟注入成功
  - [ ] 告警在 5 分钟内触发
  - [ ] Runbook 成功遵循
  - [ ] 时间线已记录
  - [ ] 事件已解决
- [ ] **场景 B（特征存储）**：
  - [ ] 特征存储中断已模拟
  - [ ] 基于日志的告警触发
  - [ ] 降级模式已激活或存储已恢复
  - [ ] 已记录
- [ ] **场景 C（错误）**：
  - [ ] 错误率已增加
  - [ ] 告警触发
  - [ ] 回滚已执行
  - [ ] 错误已解决

### 7.5 事后审查验证

- [ ] **模板已创建**：事后审查模板存在
- [ ] **完成的审查**：至少 2 次模拟已完成事后审查
- [ ] **行动项已识别**：每次事后审查有 2+ 个具体行动项
- [ ] **MTTD/MTTR 已计算**：为每次事件计算了指标
- [ ] **无指责文化**：审查关注系统而非个人

### 7.6 持续改进验证

- [ ] **积压已创建**：可观测性积压已记录
- [ ] **优先级排序**：任务按 P0/P1/P2/P3 优先级排序
- [ ] **工作量估算**：每个任务都有工作量估算
- [ ] **负责人已分配**：任务有明确的负责人
- [ ] **积压集成**：链接到项目管理工具（Jira/GitHub Issues/Linear）

### 7.7 端到端验证

- [ ] **全流程测试**：触发告警 → 接收通知 → 遵循 runbook → 解决 → 记录
- [ ] **告警疲劳检查**：非严重级别告警每人每周不超过 5 个
- [ ] **Runbook 可访问性**：Runbooks 可从告警通知访问（链接有效）
- [ ] **文档已发布**：所有产物已提交到 Git 并与团队共享

---

## 第 8 部分：拓展目标

### 8.1 高级告警

- **Grafana OnCall 集成**
- **自动化维护静默**
- **异常检测**（predict_linear、Z-score）

### 8.2 事件管理平台集成

- **ServiceNow 集成**
- **Opsgenie 集成**

### 8.3 混沌工程

- **用于 Kubernetes 的 Chaos Mesh**
- **自动化 Runbook 验证**

### 8.4 高级 Metrics

- **SLO Error Budget Dashboard**
- **On-Call 健康 Metrics**

---

## 反思问题

### 问题 1：告警灵敏度 vs 检测速度

**你如何平衡告警灵敏度（避免噪音）与快速检测的需求？**

**讨论点**：
- **基于 SLO 的告警**：告警与用户影响关联，而非任意阈值
- **多窗口 burn rates**：快速 burn（快速发现关键问题）+ 慢速 burn（发现逐渐退化）
- **告警疲劳指标**：跟踪每人每周告警数；如果 >20/周，调整阈值
- **告警审查**：季度审查所有告警；禁用或调整信号弱的告警
- **严重级别分层**：仅对严重（影响用户）进行寻呼；警告发送到 Slack
- **测试**：混沌工程以验证告警适当触发

### 问题 2：On-Call 可持续性

**什么指标表明 on-call 工作负载不可持续？**

**警示信号**：
- 每人每周告警 > 20
- 事件经常持续 > 2 小时
- 相同告警重复触发（未解决）
- On-call 工程师报告倦怠
- On-call 轮换中高离职率
- 仅在工作时间发生的事件（监控不足）
- 团队回避 on-call 职责

**补救策略**：
- 增加团队规模或轮换规模
- 自动化常见的缓解任务
- 改进预防（修复反复出现的问题）
- 添加二级 on-call 作为后备
- 减少非关键告警
- 构建 runbooks 以加快 MTTR
- 跟进事后行动项

### 问题 3：多区域 Runbooks

**你会如何为多区域部署调整 runbooks？**

**考量**：
- **区域故障切换流程**：将流量切换到健康区域的步骤
- **区域特定联系人点**：每个区域/时区的不同 on-call 团队
- **跨区域依赖**：记录区域间如何相互依赖
- **区域告警路由**：将告警路由到适当的区域团队
- **降级模式**：定义区域降级 vs 完全中断
- **数据本地性**：考虑数据驻留要求
- **时区**：Runbooks 应注明 UTC 时间和本地时间转换

### 问题 4：积压优先级排序

**积压中的哪些操作对未来可靠性影响最大？**

**评估框架**：
- **对 MTTD 的影响**：这能减少多少检测时间？
- **对 MTTR 的影响**：这能减少多少解决时间？
- **预防**：这能预防整类事件吗？
- **工作量 vs 影响**：这是速赢还是战略投资？
- **频率**：这处理的是常见事件还是罕见边缘情况？

**高影响项**（来自典型积压）：
1. **自动化漂移检测**：防止模型退化（高频、高用户影响）
2. **金丝雀部署护栏**：防止错误部署到达生产（事件预防）
3. **On-Call Dashboard**：通过提供快速概览减少 MTTR（低工作量、即时价值）
4. **Runbook 自动化**：减少常见事件的 MTTR（随团队扩展）
5. **SLO Error Budget 跟踪**：提供可靠性趋势可见性（战略性）

---

## 练习完成！

恭喜！你已为 ML 基础设施构建了生产就绪的告警和事件响应系统，包括：

✅ **完整的告警栈**：
- 带多渠道路由的 Alertmanager
- 基于 SLO 的 burn rate 告警
- 基于日志的应用错误告警
- ML 特定的漂移和性能告警

✅ **专业的 Runbooks**：
- 遵循 Google SRE 最佳实践的延迟、错误率和漂移 runbooks
- 逐步缓解流程
- 调查查询和命令
- 升级路径

✅ **事件模拟框架**：
- 自动化事件注入
- 时间线和文档生成
- 动手 runbook 练习

✅ **事后审查流程**：
- 无指责的事后审查模板
- MTTD/MTTR 跟踪
- 行动项管理

✅ **持续改进**：
- 优先级排序的可观测性积压
- 工作量估算和所有权

**下一步**：
1. 完成 Module 009 测验以测试你的知识
2. 复习 Module 009 以巩固概念
3. 继续到 Module 010：云平台

---

**Module 009：监控与日志基础** - Exercise 05 完成
