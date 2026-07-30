# Lecture 02：面向 AI 基础设施的 Prometheus Metrics 管道

## 讲座概述
Prometheus 已成为云原生环境中 metrics 收集的事实标准。本讲座解释了如何为 AI 基础设施工作负载设计、部署和操作基于 Prometheus 的 metrics 栈。你将学习 Prometheus 数据模型、配置模式、exporter 生态系统、PromQL 查询设计以及在生产环境中扩展管道的策略。

**预计阅读时间：** 75–90 分钟  
**动手配套实验：** Exercise 02 – Prometheus 设置和 Exporters  
**前置知识：** Lecture 01（可观测性基础）、Docker 基础、熟悉 Linux 服务。

---

## 1. Prometheus 架构深入

### 1.1 核心组件
- **Prometheus Server：** 抓取目标、存储时间序列数据、运行 PromQL 查询、触发告警
- **Exporters：** 以 Prometheus 暴露格式通过 HTTP 暴露指标的处理过程
- **Pushgateway：** 临时/短期任务的缓冲区（谨慎使用）
- **Alertmanager：** 管理告警路由、去重和通知
- **Service Discovery：** 发现抓取目标的机制（Kubernetes API、Consul、基于文件、EC2 等）

### 1.2 Pull vs Push 模型
- Prometheus 按配置的时间间隔从目标**拉取**指标，简化了身份验证和生命周期管理
- Pushgateway 作为无法被抓取的批处理任务的桥梁。避免将其用于长期运行的服务

### 1.3 数据流
1. 目标通过 HTTP 端点（默认为 `/metrics`）暴露指标
2. Prometheus 以配置的间隔（`scrape_interval`）抓取端点
3. 样本存储在带有保留策略（默认为 15 天）的自定义时间序列数据库（TSDB）中
4. Recording rules 将指标聚合/转换为新的序列
5. PromQL 查询为 Dashboard、告警和自动化获取数据

### 1.4 存储内部
- TSDB 以 2 小时块组织数据（每个约 128 MB）
- 样本以 chunks 格式存储；使用预写日志（WAL）保证持久性
- 压缩合并块并应用保留策略
- 高基数（metric 名称 + labels 的许多唯一组合）会增加内存使用和查询时间

---

## 2. 安装和配置 Prometheus

### 2.1 使用 Docker Compose 快速启动
创建一个用于本地实验的最小栈：
```yaml
# docker-compose.prometheus.yml
version: "3.8"

services:
  prometheus:
    image: prom/prometheus:v2.51.0
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.retention.time=15d"
      - "--web.enable-lifecycle"

volumes:
  prometheus-data:
```
运行 `docker compose -f docker-compose.prometheus.yml up -d`。

### 2.2 配置文件结构（`prometheus.yml`）
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 30s
  scrape_timeout: 10s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

**关键部分：**
- `global`：抓取频率和规则评估的默认值
- `scrape_configs`：要抓取的作业列表
- `static_configs`：静态目标列表；在生产中使用 service discovery（例如：Kubernetes）
- `relabel_configs`：在抓取时修改 labels 或删除目标

### 2.3 Service Discovery 模式
- **Kubernetes：** 使用 `kubernetes_sd_configs` 发现 Pod、Service 或 Endpoint。应用 relabeling 以选择所需目标
- **文件：** `file_sd_configs` 读取列出目标的 JSON/YAML 文件；与部署工具集成
- **云提供商：** 使用 EC2、Azure 或 GCE discovery 自动注册实例

**示例：Kubernetes Pod 抓取**
```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: (.+):(?:\d+);(\d+)
        replacement: ${1}:${2}
        target_label: __address__
```

---

## 3. 面向 AI 基础设施的 Exporters

### 3.1 常见 Exporters
- **Node Exporter：** 主机级别指标（CPU、内存、磁盘）。必备基线
- **cAdvisor / kube-state-metrics：** 容器和 Kubernetes 指标
- **Blackbox Exporter：** HTTP/HTTPS/TCP 端点的合成探测
- **DCGM Exporter：** NVIDIA GPU 指标（SM 利用率、温度、ECC 错误）
- **Prometheus Python Client：** 检测自定义应用程序和管道
- **Kafka Exporter, Redis Exporter, Postgres Exporter：** 观察平台依赖

### 3.2 自定义 Exporter 设计
当现成的 exporters 不够用时：
1. 使用 Prometheus 客户端库（Python、Go、Java、Rust）
2. 使用相关的 metric families（counters、gauges、histograms、summaries）暴露 `/metrics` 端点
3. 确保 metric 注册时的线程/进程安全
4. 限制基数（避免每次请求的 labels）
5. 提供健康端点和集成测试

**Python 示例：**
```python
from prometheus_client import start_http_server, Gauge
import random
import time

GPU_MEM_UTIL = Gauge(
    "gpu_memory_utilization",
    "Percentage of GPU memory used",
    ["gpu_id", "model_name"]
)

def collect_metrics():
    while True:
        for gpu in range(4):
            GPU_MEM_UTIL.labels(gpu_id=gpu, model_name="vision-transformer").set(random.uniform(20, 90))
        time.sleep(10)

if __name__ == "__main__":
    start_http_server(8000)
    collect_metrics()
```

### 3.3 Exporter 配置最佳实践
- 将 exporters 作为 per-pod metrics 的 sidecar 运行，或作为节点级指标的 DaemonSet 运行
- 在不受信任的网络上对抓取端点使用 TLS/Basic 身份验证
- 监控 exporter 健康（正常运行时间、抓取失败）
- 文档化版本兼容性（例如：DCGM exporter + 驱动版本）

---

## 4. PromQL 核心要点

### 4.1 数据模型回顾
- 指标由 **metric 名称** + 一组 **label 键/值对**标识
- **Sample：** (`metric_name{label1="value1", ...}`, timestamp, value)
- 向量类型：instant vector（单个时间的样本）、range vector（一段时间内的样本）

### 4.2 查询基础
- Instant 查询：`http_requests_total{job="api", status="500"}`
- Range 查询：`rate(http_requests_total{job="api"}[5m])`
- 聚合：`sum(rate(http_requests_total[5m])) by (status)`
- 使用正则表达式过滤：`{pod=~"ml-inference-.*"}`

### 4.3 必须掌握的函数
- `rate()` / `irate()` – counters 的每秒速率
- `increase()` – 范围内的总增量（用于计数）
- `avg_over_time()`, `max_over_time()` – 检测趋势
- `histogram_quantile()` – 从 histogram buckets 计算分位数
- `sum by(...)`, `avg without(...)` – 分组修饰符
- `label_replace()` – 动态操作 labels

### 4.4 AI 工作负载的示例查询
**GPU 利用率：**
```promql
avg by (cluster, gpu) (DCGM_FI_DEV_GPU_UTIL)
```

**批处理推理吞吐量：**
```promql
rate(ai_infra_batch_predictions_total{job="batch-runner"}[10m])
```

**Error Budget Burn Rate（SLO 99% 延迟 < 300 ms）：**
```promql
(1 - sum(rate(ai_infra_inference_latency_seconds_bucket{le="0.3"}[1m])) /
     sum(rate(ai_infra_inference_latency_seconds_count[1m])))
    / (1 - 0.99)
```

**数据管道积压：**
```promql
max by (pipeline) (ai_infra_pipeline_queue_length{pipeline=~"feature-.*"})
```

### 4.5 Recording Rules
- 预计算昂贵的查询或标准化 SLO 计算
- 将规则存储在 `rules/*.yml` 中并通过 `rule_files` 在 `prometheus.yml` 中引用

示例：
```yaml
groups:
  - name: inference_slo.rules
    interval: 30s
    rules:
      - record: slo:inference_latency:ratio
        expr: |
          sum(rate(ai_infra_inference_latency_seconds_bucket{le="0.3"}[5m])) /
          sum(rate(ai_infra_inference_latency_seconds_count[5m]))
      - record: slo:inference_latency:burn_rate
        expr: (1 - slo:inference_latency:ratio) / (1 - 0.99)
```

优点：
- 减少 Dashboard 加载时间
- 确保跨 Dashboard、告警的计算一致性

---

## 5. 使用 Prometheus 和 Alertmanager 的告警

### 5.1 告警规则
使用 PromQL 定义告警表达式。

```yaml
groups:
  - name: inference_alerts.rules
    rules:
      - alert: InferenceLatencyHigh
        expr: slo:inference_latency:burn_rate{cluster="prod"} > 2
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "Inference latency SLO burn rate high in {{ $labels.cluster }}"
          description: "Burn rate {{ $value }} exceeds threshold. Check runbook go/inference-latency."
```

### 5.2 Alertmanager 配置
- 处理去重、分组、路由到接收者（PagerDuty、Slack、email）
```yaml
route:
  receiver: pagerduty-prod
  group_by: ['alertname', 'cluster']
  routes:
    - matchers:
        severity = 'page'
      receiver: pagerduty-prod
    - matchers:
        severity = 'ticket'
      receiver: jira-triage

receivers:
  - name: pagerduty-prod
    pagerduty_configs:
      - routing_key: ${PAGERDUTY_KEY}
```

### 5.3 告警卫生
- 添加 `for` 子句以避免瞬时峰值
- 包含指向 Grafana Dashboard、runbook 和 trace 搜索的链接
- 使用模拟指标或测试环境在 staging 中测试告警
- 定期审查告警噪音；移除或调整误报

---

## 6. 扩展 Prometheus

### 6.1 Federation
- 下层 Prometheus 服务器抓取特定区域/集群中的工作负载
- 顶层 Prometheus 联合聚合指标（求和、平均值）
- 适用于多区域、多团队的设置

### 6.2 分片和水平扩展
- 使用 **Prometheus Operator** 或 **Cortex/Mimir/Thanos** 实现水平可扩展性、长期存储和高可用性
- **Thanos** 增加了基于对象存储的历史数据层 + 跨多个 Prometheus 实例的查询层
- **Cortex/Mimir** 是完全分布式的 TSDB，支持大规模场景

### 6.3 高可用设置注意事项
- 以主动-主动模式运行 Prometheus 对，抓取相同的目标；在 Thanos 中去重查询
- 确保 Alertmanager 作为至少三个副本的集群运行以实现共识
- 将持久数据存储在 SSD 支持的磁盘上；规划备份策略

### 6.4 保留和存储管理
- 配置 `--storage.tsdb.retention.size` 或 `--storage.tsdb.retention.time`
- 将长期指标卸载到远程存储（Thanos、Cortex、VictoriaMetrics）
- 监控基数：使用 `prometheus_tsdb_head_series` 和 `prometheus_tsdb_head_memory_allocations_bytes`

---

## 7. 安全与多租户

### 7.1 保护 Prometheus
- 通过反向代理、mTLS 或 OAuth 代理限制对 `/` 和 `/api` 端点的访问
- 在 Kubernetes 中运行时使用 RBAC；限制权限
- 清除 secrets：避免在 labels 中嵌入凭据；使用环境变量或 secret 挂载

### 7.2 多租户注意事项
- 按团队的 Namespaces/label 约定（例如：`team="ml-platform"`）
- 通过 remote write 的速率限制或查询并发实现配额
- 治理：按团队实施保留策略、成本归属 Dashboard

---

## 8. 操作最佳实践

### 8.1 Runbooks
- 文档化重启过程、告警分流步骤、常见查询参考
- 提供重启的 `kubectl` 命令或 systemctl 指令
- 为 Prometheus 配置和 Dashboard JSON 维护 GitOps 仓库

### 8.2 Prometheus 自身的可观测性
- 抓取 Prometheus 指标：
  - `prometheus_tsdb_head_samples_appended_total`
  - `prometheus_engine_query_duration_seconds`
  - `prometheus_notifications_dropped_total`
- 设置抓取失败率高、查询延迟、磁盘使用率的告警

### 8.3 测试配置
- 使用 `promtool check config prometheus.yml` 验证配置更改
- 测试规则：`promtool test rules rules/*.yml`
- 使用 `--web.enable-lifecycle` 实现安全热重载（`curl -X POST http://localhost:9090/-/reload`）

### 8.4 变更管理
- 将配置和规则存储在 Git 中
- 执行代码审查和自动化 linting（Promtool、promql lints）
- 提供模拟生产负载的 staging Prometheus 环境

---

## 9. 用于 ML 特定指标的 Prometheus

### 9.1 检测训练管道
- 从训练任务暴露指标：周期持续时间、损失、学习率
- 使用 sidecar exporters 或将指标嵌入训练脚本中
- 在分布式训练（Horovod、Ray、PyTorch DDP）中，集中聚合指标

### 9.2 监控特征存储
- 跟踪特征新鲜度、摄入延迟、每个特征表的行数
- 监控数据质量检查计数（成功 vs 失败）

### 9.3 推理服务
- 使用 histograms 捕获请求延迟（例如：`inference_latency_seconds`）
- 包含模型版本、模型类型、区域的 labels
- 监控 CPU/GPU 利用率、队列长度、并发数
- 通过暴露 PSI 等指标集成漂移检测器

### 9.4 成本和效率指标
- 导出每次请求的成本、每项任务的 GPU 小时数、能源消耗
- 结合 Grafana Dashboard 实现 FinOps 可见性

---

## 10. 故障排除和常见陷阱

### 10.1 高基数
- 避免使用无限值的 labels（用户 ID、请求 ID）
- 使用基数分析工具（`prometheus_tsdb_cardinality`）
- 考虑通过 relabeling 删除 labels

### 10.2 抓取失败
- 检查 `up` 指标（`up == 0` 表示失败）
- 检查网络连接、TLS 证书过期
- 注意 exporter 资源饥饿；添加健康探测

### 10.3 查询性能
- 使用 `avg_over_time` 或 recording rules 进行长期计算
- 限制 `offset` 的使用；确保查询覆盖合理的窗口
- 监控 Prometheus CPU 使用率；重型查询可能使服务器过载

### 10.4 时间同步
- 确保服务器间启用 NTP；时钟不同步会破坏关联

---

## 11. 动手检查清单
- [ ] 通过 Docker Compose 或 Kubernetes 清单部署 Prometheus
- [ ] 配置 node exporter、自定义服务和应用程序检测的抓取任务
- [ ] 创建推理延迟 SLO 的 recording rule
- [ ] 为你创建的规则编写 promtool 单元测试
- [ ] 触发一个示例告警并验证其正确通过 Alertmanager 路由
- [ ] 在 README 中使用图表/ascii 文档化管道架构

---

## 12. 知识检查
1. 解释 Prometheus 数据模型以及为什么 label 基数很重要
2. 用具体用例区分 `rate`、`irate` 和 `increase`
3. 设计一个 PromQL 查询，列出 1 小时窗口内 GPU 消耗排名前 5 的模型
4. 概述从 Kubernetes Pod 抓取 Prometheus 指标的步骤
5. 描述两种扩展 Prometheus 以管理 100 万个活跃序列的方法

---

## 13. 附加资源
- Prometheus Docs: https://prometheus.io/docs/introduction/overview/
- Robust Perception blog（Julius Volz, Brian Brazil）– Prometheus 使用深入分析
- "Monitoring Distributed Systems"（云原生模式系列）
- Thanos 和 Cortex 文档，了解水平可扩展的 Prometheus
- NVIDIA DCGM exporter 文档，了解 GPU 监控
- Grafana Labs 的 PromQL 教程

---

## 14. 总结
- Prometheus 提供了灵活的基于拉取的 metrics 管道，非常适合云原生 AI 基础设施
- Exporters 和 service discovery 将覆盖范围扩展到基础设施、平台和 ML 工作负载
- PromQL 实现了强大的分析——recording rules 和告警将这些洞察转化为行动
- 扩展策略（federation、Thanos、Mimir）支持大规模部署
- 集成 ML 特定指标将可观测性扩展到传统基础设施信号之外

接下来，Lecture 03 将在 Prometheus 之上构建 Grafana，为你的 AI 平台打造有意义的 Dashboard 和告警工作流。
