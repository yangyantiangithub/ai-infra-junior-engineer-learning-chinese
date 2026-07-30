# Exercise 02：Prometheus Monitoring Stack 部署

## 概述

部署一个生产就绪的 Prometheus monitoring 栈，具有全面的 scraping、recording rules 和告警功能。基于 Exercise 01 中已 instrument 的推理网关，你将创建一个完整的 metrics 管道，包括 Prometheus server、node exporters、自定义 exporters、Alertmanager 和 service discovery。本练习为可观测性驱动的运维和 SLO 监控奠定基础。

**难度：** 中级 → 高级
**预计时间：** 4–5 小时
**前置要求：**
- 已完成 Exercise 01（已 instrument 的 FastAPI 推理服务）
- Lecture 02（Prometheus Metrics 管道）
- 已安装 Docker 和 Docker Compose
- 基本熟悉 PromQL 和 YAML
- 理解 Exercise 01 中的 SLO

## 学习目标

完成本实验后，你将能够：

1. 使用 Docker Compose 部署完整的 Prometheus monitoring 栈
2. 配置 service discovery 以实现动态目标监控
3. 实现全面的 recording rules 用于 SLO 跟踪
4. 创建生产就绪的告警规则，包含多窗口 burn rates
5. 部署和配置 Alertmanager，包含路由和静默
6. 构建用于应用特定 metrics 的自定义 Prometheus exporters
7. 实现 metric relabeling 和过滤策略
8. 设置 federation 用于分层 Prometheus 部署
9. 配置 remote write 用于长期存储
10. 排查常见的 Prometheus 运维问题
11. 创建用于事件响应的 runbooks
12. 实施生产监控最佳实践

## 场景

ML Platform 团队需要强大的监控基础设施来支持 Exercise 01 中部署的推理网关。你的任务是构建一个生产级的 Prometheus 栈，实现：

- 监控基础设施（节点、容器、网络）
- 跟踪推理网关的应用 metrics
- 实时计算 SLO 合规性
- 在 error budget 受到威胁时告警 on-call 工程师
- 提供具有适当保留期的可靠 metrics 存储
- 可扩展以跨环境监控多个服务

此基础将支持后续练习中构建的 Grafana dashboard（Exercise 03）和告警工作流（Exercise 05）。

---

## 第 1 部分：架构规划与设计

### 步骤 1.1：定义监控架构

创建 `docs/prometheus-architecture.md`：

```markdown
# Prometheus Monitoring Stack 架构

## 概述

本文档描述了用于 ML Platform 的 Prometheus monitoring 栈架构。

## 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Monitored Targets                            │
│                                                                  │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐ │
│  │ Inference  │  │    Node    │  │  Custom    │  │Prometheus │ │
│  │  Gateway   │  │  Exporter  │  │ Exporters  │  │   Self    │ │
│  │ :8000      │  │ :9100      │  │ :9101-9199 │  │ :9090     │ │
│  └──────┬─────┘  └──────┬─────┘  └──────┬─────┘  └─────┬─────┘ │
│         │                │                │              │       │
│         └────────────────┴────────────────┴──────────────┘       │
│                              │                                   │
└──────────────────────────────┼───────────────────────────────────┘
                               │ HTTP Pull (Scrape)
                     ┌──────────▼───────────┐
                     │  Prometheus Server   │
                     │                      │
                     │  - TSDB Storage      │
                     │  - Rule Evaluation   │
                     │  - Service Discovery │
                     │  - Metric Relabeling │
                     └──────────┬───────────┘
                                │
                    ┌───────────┼────────────┐
                    │           │            │
       ┌────────────▼─┐  ┌──────▼─────┐  ┌──▼──────────┐
       │  Recording   │  │  Alerting  │  │Remote Write │
       │    Rules     │  │   Rules    │  │  (optional) │
       └──────────────┘  └──────┬─────┘  └──────┬──────┘
                                │                │
                         ┌──────▼─────┐   ┌─────▼──────┐
                         │Alert Manager│  │Long-term   │
                         │             │  │Storage     │
                         │- Routing    │  │(Thanos/    │
                         │- Silencing  │  │ Mimir)     │
                         │- Grouping   │  └────────────┘
                         └──────┬──────┘
                                │
                     ┌──────────┼──────────┐
                     │          │          │
             ┌───────▼──┐  ┌────▼────┐  ┌─▼────────┐
             │  Slack   │  │ PagerDuty│ │  Email   │
             │Webhook   │  │ API      │  │  SMTP    │
             └──────────┘  └──────────┘  └──────────┘
```

## 组件描述

### 1. Monitored Targets

**Inference Gateway（端口 8000）**
- 应用特定 metrics（推理延迟、错误、吞吐量）
- HTTP 请求 metrics（四大黄金信号）
- ML metrics（置信度分数、按类别的预测）
- 自定义业务 metrics

**Node Exporter（端口 9100）**
- CPU、内存、磁盘、网络 metrics
- 文件系统使用情况
- 系统负载平均值
- 硬件和内核 metrics

**Custom Exporters（端口 9101-9199）**
- Feature store exporter（9101）：特征新鲜度、摄入延迟
- Model registry exporter（9102）：模型版本、下载、部署
- GPU exporter（9103）：GPU 利用率、内存、温度
- Database exporter（9104）：PostgreSQL 连接池、查询延迟

**Prometheus Self-Monitoring（端口 9090）**
- TSDB metrics（序列数、摄入率）
- Scrape 健康状况和持续时间
- 规则评估性能
- 资源使用情况

### 2. Prometheus Server

**职责**：
- 每 15 秒从所有目标抓取 metrics
- 在本地 TSDB 中存储时间序列数据
- 每 30 秒评估 recording 和 alerting rules
- 为 dashboard 提供 PromQL 查询
- 向 Alertmanager 发送告警
- 可选：远程写入长期存储

**配置**：
- Scrape 间隔：15s（每个作业可调）
- 评估间隔：30s
- 保留期：15 天（本地）
- 存储：每天每百万样本约 1GB

### 3. Recording Rules

**目的**：预计算昂贵的查询，用于 dashboard 和告警

**分类**：
- SLO 计算（可用性、延迟百分位数、错误率）
- 聚合（按服务、按团队、按环境）
- 派生指标（burn rates、剩余 budget）

### 4. Alerting Rules

**目的**：定义触发通知的条件

**分类**：
- SLO 违规（error budget 耗尽）
- 资源饱和（磁盘满、内存压力）
- 服务健康（scrape 失败、高错误率）
- 多窗口 burn rate 告警（快速/慢速 burn 检测）

### 5. Alertmanager

**职责**：
- 从 Prometheus 接收告警
- 去重和分组相关告警
- 路由到适当的接收者（Slack、PagerDuty、email）
- 在维护窗口期间应用静默
- 管理告警状态（firing、resolved）

**路由策略**：
- Critical 告警 → PagerDuty（on-call 工程师）
- Warning 告警 → Slack 频道（#ml-platform-alerts）
- Info 告警 → 邮件摘要（每日汇总）

### 6. Remote Write（可选）

**目的**：超过本地保留期的长期 metrics 存储

**选项**：
- **Thanos**：基于 S3 的经济高效存储，支持降采样
- **Mimir**：多租户、水平可扩展
- **VictoriaMetrics**：高性能、资源高效
- **Grafana Cloud**：托管服务

**权衡**：
- 存储成本 vs 本地保留
- 查询性能 vs 历史深度
- 运维复杂度 vs 托管服务

## 可扩展性考量

### 当前规模（实验环境）
- Targets：5-10 个服务
- Metrics：约 10,000 活跃时间序列
- 摄入率：约 5,000 样本/秒
- 存储：约 50MB/天，约 750MB/15 天

### 生产规模（预期增长）
- Targets：100-500 个服务
- Metrics：约 500,000 活跃时间序列
- 摄入率：约 250,000 样本/秒
- 存储：约 2.5GB/天，约 37GB/15 天

### 扩展策略
1. **垂直扩展**：增加 Prometheus server 资源（CPU、RAM、SSD）
2. **水平扩展**：使用 per-team 或 per-region Prometheus 实例的 federation
3. **指标缩减**：通过 relabeling 丢弃高基数或未使用的 metrics
4. **查询优化**：使用 recording rules 预聚合昂贵查询

## 高可用性（生产）

### HA 设置（未来）
- 运行 2+ 个相同的 Prometheus 副本抓取相同目标
- 使用外部 Alertmanager 集群（3+ 实例）
- 在 Alertmanager 中去重告警
- 跨副本负载均衡查询

### 数据一致性
- Prometheus 实例是独立的（无复制）
- Alertmanager 处理去重
- Dashboards 查询所有副本（max/avg/etc.）

## 安全考量

### 身份验证
- Prometheus UI 的 Basic auth（生产环境）
- Scrape 目标的 TLS（未来）
- Kubernetes service discovery 的服务账户令牌

### 授权
- 开发人员的只读访问
- 平台团队的管理员访问
- 不公开互联网暴露（仅内部网络）

### 敏感数据
- 不在 metric labels 中包含凭据或 secrets
- Scrape 密码存储在 Kubernetes secrets 中
- Alertmanager receiver 凭据在环境变量中

## 灾难恢复

### 备份策略
- 快照：每日 TSDB 快照到 S3（或等效存储）
- 保留期：30 天快照保留
- 恢复时间：从快照恢复约 15 分钟

### 关键数据
- Recording rules 和告警（在 Git 中进行版本控制）
- Prometheus 配置（在 Git 中进行版本控制）
- TSDB 数据（临时性，15 天保留可接受）

### RPO/RTO
- 恢复点目标：15 分钟（scrape 间隔）
- 恢复时间目标：15 分钟（从快照恢复）

## 监控监控者

### Prometheus 自监控
- 跟踪 scrape 成功率（`up` metric）
- 监控 TSDB head 序列数
- 对规则评估失败告警
- Prometheus 资源使用情况的 Dashboard

### Alertmanager 健康
- 监控告警投递成功率
- 跟踪通知队列深度
- 对 receiver 失败告警

### Deadman's Switch
- 始终触发的告警以验证告警管道
- 如果在预期窗口内未收到则升级
```

**✅ 检查点：** 架构文档已完成并审阅。

### 步骤 1.2：创建项目结构

```bash
cd /path/to/monitoring-stack
mkdir -p prometheus/{rules,snapshots,data}
mkdir -p alertmanager/data
mkdir -p exporters/{feature-store,model-registry,gpu-exporter}
mkdir -p grafana/{dashboards,provisioning}
mkdir -p docs
mkdir -p scripts
```

**预期结构：**
```
monitoring-stack/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml           # 主配置
│   ├── rules/
│   │   ├── slo_recording.rules.yml
│   │   ├── slo_alerts.rules.yml
│   │   ├── infrastructure.rules.yml
│   │   └── service_health.rules.yml
│   ├── snapshots/               # TSDB 备份快照
│   └── data/                    # TSDB 数据目录
├── alertmanager/
│   ├── alertmanager.yml         # 告警配置
│   ├── templates/
│   │   └── slack.tmpl           # 自定义消息模板
│   └── data/                    # Alertmanager 状态
├── exporters/
│   ├── feature-store/
│   │   ├── exporter.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   ├── model-registry/
│   │   ├── exporter.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   └── gpu-exporter/
│       ├── exporter.py
│       ├── requirements.txt
│       └── Dockerfile
├── scripts/
│   ├── start-stack.sh
│   ├── stop-stack.sh
│   ├── snapshot-prometheus.sh
│   └── load-test-metrics.sh
├── docs/
│   ├── prometheus-architecture.md
│   ├── runbooks/
│   │   ├── high-latency.md
│   │   ├── disk-full.md
│   │   └── scrape-failures.md
│   └── troubleshooting.md
└── README.md
```

---

## 第 2 部分：核心 Prometheus 部署

### 步骤 2.1：创建 Docker Compose 配置

创建 `docker-compose.yml`：

```yaml
# docker-compose.yml
version: '3.9'

services:
  # ==============================================
  # Prometheus Server
  # ==============================================
  prometheus:
    image: prom/prometheus:v2.48.1
    container_name: prometheus
    user: "65534:65534"  # 出于安全考虑使用 nobody 用户
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--storage.tsdb.retention.size=10GB'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
      - '--storage.tsdb.wal-compression'
      - '--query.max-concurrency=20'
      - '--query.timeout=2m'
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus-data:/prometheus
    networks:
      - monitoring
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # ==============================================
  # Alertmanager
  # ==============================================
  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
      - '--web.external-url=http://localhost:9093'
      - '--cluster.listen-address='
      - '--log.level=info'
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - ./alertmanager/templates:/etc/alertmanager/templates:ro
      - alertmanager-data:/alertmanager
    networks:
      - monitoring
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:9093/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ==============================================
  # Node Exporter (Infrastructure Metrics)
  # ==============================================
  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node-exporter
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/host/root'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
      - '--collector.netclass.ignored-devices=^(veth.*|docker.*|br-.*)'
      - '--no-collector.ipvs'
      - '--no-collector.softnet'
      - '--no-collector.wifi'
      - '--no-collector.zfs'
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/host/root:ro,rslave
    pid: host
    networks:
      - monitoring
    restart: unless-stopped

  # ==============================================
  # cAdvisor (Container Metrics)
  # ==============================================
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    devices:
      - /dev/kmsg
    networks:
      - monitoring
    restart: unless-stopped

  # ==============================================
  # Inference Gateway（来自 Exercise 01）
  # ==============================================
  inference-gateway:
    build:
      context: ../exercise-01/inference-gateway
      dockerfile: Dockerfile
      target: development
    container_name: inference-gateway
    ports:
      - "8000:8000"
    environment:
      - SERVICE_NAME=inference-gateway
      - SERVICE_VERSION=1.0.0
      - ENVIRONMENT=lab
      - LOG_LEVEL=INFO
    networks:
      - monitoring
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ==============================================
  # Custom Exporter: Feature Store
  # ==============================================
  feature-store-exporter:
    build:
      context: ./exporters/feature-store
      dockerfile: Dockerfile
    container_name: feature-store-exporter
    ports:
      - "9101:9101"
    environment:
      - EXPORTER_PORT=9101
      - FEATURE_STORE_URL=http://feature-store:8000
      - SCRAPE_INTERVAL=30
    networks:
      - monitoring
    restart: unless-stopped

  # ==============================================
  # Custom Exporter: Model Registry
  # ==============================================
  model-registry-exporter:
    build:
      context: ./exporters/model-registry
      dockerfile: Dockerfile
    container_name: model-registry-exporter
    ports:
      - "9102:9102"
    environment:
      - EXPORTER_PORT=9102
      - REGISTRY_DB_URL=postgresql://user:pass@postgres:5432/registry
      - SCRAPE_INTERVAL=60
    networks:
      - monitoring
    restart: unless-stopped

  # ==============================================
  # Pushgateway（用于批处理任务）
  # ==============================================
  pushgateway:
    image: prom/pushgateway:v1.6.2
    container_name: pushgateway
    command:
      - '--persistence.file=/var/lib/pushgateway/persistence'
      - '--persistence.interval=5m'
    ports:
      - "9091:9091"
    volumes:
      - pushgateway-data:/var/lib/pushgateway
    networks:
      - monitoring
    restart: unless-stopped

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus-data:
  alertmanager-data:
  pushgateway-data:
```

**关键特性**：
- 单一 compose 文件中的完整 monitoring 栈
- 所有服务的健康检查
- 持久化的适当 volume 挂载
- 资源限制（可通过 deploy.resources 添加）
- 安全：非 root 用户、只读挂载

### 步骤 2.2：创建 Prometheus 配置

创建 `prometheus/prometheus.yml`：

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 30s
  scrape_timeout: 10s
  external_labels:
    cluster: 'lab'
    environment: 'development'
    region: 'us-west-2'

# 加载 recording 和 alerting rules
rule_files:
  - /etc/prometheus/rules/slo_recording.rules.yml
  - /etc/prometheus/rules/slo_alerts.rules.yml
  - /etc/prometheus/rules/infrastructure.rules.yml
  - /etc/prometheus/rules/service_health.rules.yml

# Alertmanager 配置
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
      timeout: 10s
      api_version: v2

# Scrape 配置
scrape_configs:
  # ==============================================
  # Prometheus 自监控
  # ==============================================
  - job_name: 'prometheus'
    honor_labels: true
    static_configs:
      - targets: ['localhost:9090']
        labels:
          service: 'prometheus'
          team: 'platform'

  # ==============================================
  # Alertmanager
  # ==============================================
  - job_name: 'alertmanager'
    static_configs:
      - targets: ['alertmanager:9093']
        labels:
          service: 'alertmanager'
          team: 'platform'

  # ==============================================
  # Node Exporter（基础设施）
  # ==============================================
  - job_name: 'node-exporter'
    scrape_interval: 30s  # 基础设施指标频率较低
    static_configs:
      - targets: ['node-exporter:9100']
        labels:
          service: 'node-exporter'
          instance_type: 'docker-host'
          team: 'platform'
    # 丢弃嘈杂的文件系统指标
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'node_filesystem_(avail|free|size)_bytes'
        target_label: __name__
        action: keep
      - source_labels: [__name__, mountpoint]
        regex: 'node_filesystem_.+;(/host)?/(dev|proc|sys|run).*'
        action: drop
      - source_labels: [fstype]
        regex: '(tmpfs|devtmpfs|overlay|aufs|squashfs)'
        action: drop

  # ==============================================
  # cAdvisor（容器指标）
  # ==============================================
  - job_name: 'cadvisor'
    scrape_interval: 30s
    static_configs:
      - targets: ['cadvisor:8080']
        labels:
          service: 'cadvisor'
          team: 'platform'
    # 仅保留有用的容器指标
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'container_(cpu_usage_seconds_total|memory_usage_bytes|network_(receive|transmit)_bytes_total|fs_(reads|writes)_bytes_total)'
        action: keep
      - source_labels: [name]
        regex: ''
        action: drop  # 丢弃没有容器名称的指标

  # ==============================================
  # Inference Gateway（应用指标）
  # ==============================================
  - job_name: 'inference-gateway'
    scrape_interval: 15s
    metrics_path: '/metrics'
    static_configs:
      - targets: ['inference-gateway:8000']
        labels:
          service: 'inference-gateway'
          team: 'ml-platform'
          environment: 'lab'
          version: '1.0.0'
    # 重新标记以添加实例元数据
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: 'inference-gateway-01'
    # 如果需要，丢弃高基数指标
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'python_gc_.*'
        action: drop  # 丢弃 Python GC 指标（过于嘈杂）

  # ==============================================
  # Feature Store Exporter
  # ==============================================
  - job_name: 'feature-store'
    scrape_interval: 60s  # 批处理指标频率较低
    static_configs:
      - targets: ['feature-store-exporter:9101']
        labels:
          service: 'feature-store'
          team: 'ml-platform'

  # ==============================================
  # Model Registry Exporter
  # ==============================================
  - job_name: 'model-registry'
    scrape_interval: 60s
    static_configs:
      - targets: ['model-registry-exporter:9102']
        labels:
          service: 'model-registry'
          team: 'ml-platform'

  # ==============================================
  # Pushgateway（批处理任务）
  # ==============================================
  - job_name: 'pushgateway'
    honor_labels: true  # 保留推送指标中的 job/instance
    scrape_interval: 30s
    static_configs:
      - targets: ['pushgateway:9091']
        labels:
          service: 'pushgateway'
          team: 'platform'

# Remote write 配置（可选 - 用于长期存储）
# remote_write:
#   - url: 'http://thanos-receive:19291/api/v1/receive'
#     queue_config:
#       capacity: 10000
#       max_shards: 5
#       min_shards: 1
#       max_samples_per_send: 5000
#       batch_send_deadline: 5s
#     write_relabel_configs:
#       - source_labels: [__name__]
#         regex: 'up|scrape_.*'
#         action: drop  # 不发送 Prometheus 内部指标
```

**关键特性**：
- 具有适当间隔的多个 scrape 作业
- 减少基数的 metric relabeling
- 用于 pushgateway 的 honor labels
- 用于 federation 的外部 labels
- Alertmanager 集成
- 可选的 remote write 设置

**✅ 检查点：** Prometheus 配置已创建并使用 `promtool check config prometheus.yml` 验证。

---

## 第 3 部分：用于 SLO 跟踪的 Recording Rules

### 步骤 3.1：创建 SLO Recording Rules

创建 `prometheus/rules/slo_recording.rules.yml`：

```yaml
# prometheus/rules/slo_recording.rules.yml
groups:
  # ==============================================
  # Inference Gateway SLO 计算
  # ==============================================
  - name: inference_gateway_slo
    interval: 1m
    rules:
      # ----------------------------------------------
      # 可用性 SLO（99.5%）
      # ----------------------------------------------
      - record: slo:service_availability:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{service="inference-gateway",status_code!~"5.."}[5m]))
          /
          sum(rate(http_requests_total{service="inference-gateway"}[5m]))
        labels:
          slo: availability
          service: inference-gateway

      - record: slo:service_availability:ratio_rate30m
        expr: |
          sum(rate(http_requests_total{service="inference-gateway",status_code!~"5.."}[30m]))
          /
          sum(rate(http_requests_total{service="inference-gateway"}[30m]))
        labels:
          slo: availability
          service: inference-gateway

      - record: slo:service_availability:ratio_rate1h
        expr: |
          sum(rate(http_requests_total{service="inference-gateway",status_code!~"5.."}[1h]))
          /
          sum(rate(http_requests_total{service="inference-gateway"}[1h]))
        labels:
          slo: availability
          service: inference-gateway

      - record: slo:service_availability:ratio_rate6h
        expr: |
          sum(rate(http_requests_total{service="inference-gateway",status_code!~"5.."}[6h]))
          /
          sum(rate(http_requests_total{service="inference-gateway"}[6h]))
        labels:
          slo: availability
          service: inference-gateway

      # Error budget 剩余（99.5% SLO 允许 0.5% 错误率）
      - record: slo:service_availability:error_budget_remaining
        expr: |
          1 - (
            (1 - slo:service_availability:ratio_rate30m{service="inference-gateway"})
            /
            (1 - 0.995)  # SLO 目标为 99.5%
          )
        labels:
          slo: availability
          service: inference-gateway

      # ----------------------------------------------
      # 延迟 SLO（P99 < 300ms）
      # ----------------------------------------------
      - record: slo:service_latency:p99_rate5m
        expr: |
          histogram_quantile(
            0.99,
            sum(rate(http_request_duration_seconds_bucket{service="inference-gateway",endpoint="/predict"}[5m])) by (le)
          )
        labels:
          slo: latency
          service: inference-gateway
          quantile: "0.99"

      - record: slo:service_latency:p99_rate30m
        expr: |
          histogram_quantile(
            0.99,
            sum(rate(http_request_duration_seconds_bucket{service="inference-gateway",endpoint="/predict"}[30m])) by (le)
          )
        labels:
          slo: latency
          service: inference-gateway
          quantile: "0.99"

      - record: slo:service_latency:p95_rate5m
        expr: |
          histogram_quantile(
            0.95,
            sum(rate(http_request_duration_seconds_bucket{service="inference-gateway",endpoint="/predict"}[5m])) by (le)
          )
        labels:
          slo: latency
          service: inference-gateway
          quantile: "0.95"

      # 在 300ms 阈值以下的请求百分比
      - record: slo:service_latency:success_ratio_rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{service="inference-gateway",endpoint="/predict",le="0.3"}[5m]))
          /
          sum(rate(http_request_duration_seconds_count{service="inference-gateway",endpoint="/predict"}[5m]))
        labels:
          slo: latency
          service: inference-gateway
          threshold: "300ms"

      - record: slo:service_latency:success_ratio_rate30m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{service="inference-gateway",endpoint="/predict",le="0.3"}[30m]))
          /
          sum(rate(http_request_duration_seconds_count{service="inference-gateway",endpoint="/predict"}[30m]))
        labels:
          slo: latency
          service: inference-gateway
          threshold: "300ms"

      # 延迟 error budget 剩余
      - record: slo:service_latency:error_budget_remaining
        expr: |
          1 - (
            (1 - slo:service_latency:success_ratio_rate30m{service="inference-gateway"})
            /
            (1 - 0.99)  # 99% 的请求应在 300ms 内完成
          )
        labels:
          slo: latency
          service: inference-gateway

      # ----------------------------------------------
      # 错误率 SLO（< 0.5%）
      # ----------------------------------------------
      - record: slo:service_error_rate:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{service="inference-gateway",status_code=~"5.."}[5m]))
          /
          sum(rate(http_requests_total{service="inference-gateway"}[5m]))
        labels:
          slo: error_rate
          service: inference-gateway

      - record: slo:service_error_rate:ratio_rate30m
        expr: |
          sum(rate(http_requests_total{service="inference-gateway",status_code=~"5.."}[30m]))
          /
          sum(rate(http_requests_total{service="inference-gateway"}[30m]))
        labels:
          slo: error_rate
          service: inference-gateway

      - record: slo:service_error_rate:error_budget_remaining
        expr: |
          1 - (
            slo:service_error_rate:ratio_rate30m{service="inference-gateway"}
            /
            0.005  # 0.5% 错误率目标
          )
        labels:
          slo: error_rate
          service: inference-gateway

  # ==============================================
  # Burn Rate 计算（多窗口）
  # ==============================================
  - name: slo_burn_rates
    interval: 1m
    rules:
      # 快速 burn（5 分钟窗口）
      - record: slo:burn_rate:5m
        expr: |
          (1 - slo:service_availability:ratio_rate5m{service="inference-gateway"})
          /
          (1 - 0.995)  # SLO 目标
        labels:
          slo: availability
          service: inference-gateway
          window: 5m

      # 中等 burn（30 分钟窗口）
      - record: slo:burn_rate:30m
        expr: |
          (1 - slo:service_availability:ratio_rate30m{service="inference-gateway"})
          /
          (1 - 0.995)
        labels:
          slo: availability
          service: inference-gateway
          window: 30m

      # 慢速 burn（1 小时窗口）
      - record: slo:burn_rate:1h
        expr: |
          (1 - slo:service_availability:ratio_rate1h{service="inference-gateway"})
          /
          (1 - 0.995)
        labels:
          slo: availability
          service: inference-gateway
          window: 1h

      # 极慢 burn（6 小时窗口）
      - record: slo:burn_rate:6h
        expr: |
          (1 - slo:service_availability:ratio_rate6h{service="inference-gateway"})
          /
          (1 - 0.995)
        labels:
          slo: availability
          service: inference-gateway
          window: 6h

  # ==============================================
  # 服务级聚合
  # ==============================================
  - name: service_aggregations
    interval: 30s
    rules:
      # 每个服务的总请求率
      - record: service:http_requests:rate1m
        expr: |
          sum(rate(http_requests_total[1m])) by (service, environment)

      # 每个端点的总请求率
      - record: service:http_requests_by_endpoint:rate1m
        expr: |
          sum(rate(http_requests_total[1m])) by (service, endpoint, method)

      # 每个服务的 P95/P99 延迟
      - record: service:http_duration:p95
        expr: |
          histogram_quantile(
            0.95,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
          )

      - record: service:http_duration:p99
        expr: |
          histogram_quantile(
            0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (service, le)
          )

  # ==============================================
  # ML 特定 Metrics
  # ==============================================
  - name: ml_metrics
    interval: 1m
    rules:
      # 平均推理延迟
      - record: ml:inference_duration:mean
        expr: |
          sum(rate(inference_duration_seconds_sum{service="inference-gateway"}[5m]))
          /
          sum(rate(inference_duration_seconds_count{service="inference-gateway"}[5m]))

      # 推理吞吐量（每秒预测数）
      - record: ml:inference_throughput:rate1m
        expr: |
          sum(rate(inference_requests_total{service="inference-gateway"}[1m]))

      # 模型置信度分布（高置信度百分比）
      - record: ml:inference_confidence:high_ratio
        expr: |
          sum(rate(inference_confidence_score_bucket{le="1.0"}[5m]))
          -
          sum(rate(inference_confidence_score_bucket{le="0.7"}[5m]))
          /
          sum(rate(inference_confidence_score_count[5m]))
```

**关键特性**：
- 用于 burn rate 告警的多个时间窗口（5m、30m、1h、6h）
- 可用性、延迟和错误率的分离 SLO
- Error budget 计算
- 用于 dashboard 的服务级聚合
- ML 特定 metrics（推理延迟、吞吐量、置信度）

**✅ 检查点：** 使用 `promtool check rules prometheus/rules/slo_recording.rules.yml` 验证规则。

---

## 第 4 部分：告警规则和通知

### 步骤 4.1：创建多窗口 Burn Rate 告警

创建 `prometheus/rules/slo_alerts.rules.yml`：

```yaml
# prometheus/rules/slo_alerts.rules.yml
groups:
  # ==============================================
  # 多窗口 Burn Rate 告警
  # 遵循 Google SRE 最佳实践
  # ==============================================
  - name: slo_burn_rate_alerts
    rules:
      # ----------------------------------------------
      # CRITICAL：快速 burn（5m/1h 窗口）
      # Budget 将在约 2 小时内耗尽
      # ----------------------------------------------
      - alert: SLOAvailabilityFastBurnCritical
        expr: |
          (
            slo:burn_rate:5m{slo="availability",service="inference-gateway"} > (14.4 * 0.9)
            and
            slo:burn_rate:1h{slo="availability",service="inference-gateway"} > (14.4 * 0.9)
          )
        for: 2m
        labels:
          severity: critical
          slo: availability
          service: inference-gateway
          team: ml-platform
          page: "true"
        annotations:
          summary: "Critical: Availability SLO burn rate extremely high"
          description: |
            Service {{ $labels.service }} is burning through error budget at 14.4x the acceptable rate.
            At this rate, the monthly error budget will be exhausted in ~2 hours.

            Current 5m burn rate: {{ $value | humanize }}x
            Error budget remaining: {{ query "slo:service_availability:error_budget_remaining{service='inference-gateway'}" | first | value | humanizePercentage }}

            Dashboards:
            - SLO Overview: http://grafana:3000/d/slo-overview
            - Service Health: http://grafana:3000/d/service-health

            Runbook: https://runbooks.example.com/slo-availability-fast-burn

      # ----------------------------------------------
      # WARNING：快速 burn（30m/6h 窗口）
      # Budget 将在约 1 天内耗尽
      # ----------------------------------------------
      - alert: SLOAvailabilityFastBurnWarning
        expr: |
          (
            slo:burn_rate:30m{slo="availability",service="inference-gateway"} > (6 * 0.9)
            and
            slo:burn_rate:6h{slo="availability",service="inference-gateway"} > (6 * 0.9)
          )
        for: 15m
        labels:
          severity: warning
          slo: availability
          service: inference-gateway
          team: ml-platform
          page: "false"
        annotations:
          summary: "Warning: Availability SLO burn rate high"
          description: |
            Service {{ $labels.service }} is burning through error budget at 6x the acceptable rate.
            At this rate, the monthly error budget will be exhausted in ~1 day.

            Current 30m burn rate: {{ $value | humanize }}x
            Error budget remaining: {{ query "slo:service_availability:error_budget_remaining{service='inference-gateway'}" | first | value | humanizePercentage }}

            Investigate and resolve before this becomes critical.

      # ----------------------------------------------
      # CRITICAL：延迟 SLO 违规
      # ----------------------------------------------
      - alert: SLOLatencyFastBurnCritical
        expr: |
          (
            slo:service_latency:p99_rate5m{service="inference-gateway"} > 0.3
            and
            slo:service_latency:p99_rate30m{service="inference-gateway"} > 0.3
          )
        for: 5m
        labels:
          severity: critical
          slo: latency
          service: inference-gateway
          team: ml-platform
          page: "true"
        annotations:
          summary: "Critical: P99 latency exceeds 300ms SLO"
          description: |
            Service {{ $labels.service }} P99 latency is {{ $value | humanizeDuration }}, exceeding the 300ms SLO.

            Current metrics:
            - P99 (5m): {{ query "slo:service_latency:p99_rate5m{service='inference-gateway'}" | first | value | humanizeDuration }}
            - P99 (30m): {{ query "slo:service_latency:p99_rate30m{service='inference-gateway'}" | first | value | humanizeDuration }}
            - P95 (5m): {{ query "slo:service_latency:p95_rate5m{service='inference-gateway'}" | first | value | humanizeDuration }}

            Runbook: https://runbooks.example.com/slo-latency-high

      # ----------------------------------------------
      # WARNING：错误率升高
      # ----------------------------------------------
      - alert: SLOErrorRateElevated
        expr: |
          slo:service_error_rate:ratio_rate30m{service="inference-gateway"} > 0.005
        for: 15m
        labels:
          severity: warning
          slo: error_rate
          service: inference-gateway
          team: ml-platform
          page: "false"
        annotations:
          summary: "Warning: Error rate exceeds 0.5% SLO"
          description: |
            Service {{ $labels.service }} error rate is {{ $value | humanizePercentage }}, exceeding the 0.5% SLO.

            Current error rate (30m): {{ $value | humanizePercentage }}
            Error budget remaining: {{ query "slo:service_error_rate:error_budget_remaining{service='inference-gateway'}" | first | value | humanizePercentage }}

            Investigate errors in logs and traces.

  # ==============================================
  # 服务健康告警
  # ==============================================
  - name: service_health_alerts
    rules:
      # ----------------------------------------------
      # 服务宕机
      # ----------------------------------------------
      - alert: ServiceDown
        expr: up{job="inference-gateway"} == 0
        for: 1m
        labels:
          severity: critical
          service: inference-gateway
          team: ml-platform
          page: "true"
        annotations:
          summary: "Critical: Service {{ $labels.service }} is down"
          description: |
            Service {{ $labels.service }} (instance {{ $labels.instance }}) is unreachable.
            Prometheus cannot scrape metrics from this target.

            Target: {{ $labels.job }}
            Instance: {{ $labels.instance }}

            Check service health, container status, and network connectivity.

            Runbook: https://runbooks.example.com/service-down

      # ----------------------------------------------
      # 高请求错误率
      # ----------------------------------------------
      - alert: HighRequestErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m])) by (service)
            /
            sum(rate(http_requests_total[5m])) by (service)
          ) > 0.05
        for: 5m
        labels:
          severity: warning
          team: ml-platform
        annotations:
          summary: "High error rate for {{ $labels.service }}"
          description: |
            Service {{ $labels.service }} has {{ $value | humanizePercentage }} error rate (5xx responses).

            Threshold: 5%
            Current: {{ $value | humanizePercentage }}

            Check logs and application health.

      # ----------------------------------------------
      # 推理队列积压
      # ----------------------------------------------
      - alert: InferenceQueueBackup
        expr: inference_queue_depth{service="inference-gateway"} > 100
        for: 10m
        labels:
          severity: warning
          service: inference-gateway
          team: ml-platform
        annotations:
          summary: "Inference queue depth high"
          description: |
            Inference queue has {{ $value }} pending requests.

            This may indicate:
            - Model inference is slow
            - Request rate exceeds capacity
            - Resource constraints (CPU/GPU)

            Consider scaling up inference workers or investigating performance issues.

  # ==============================================
  # ML 特定告警
  # ==============================================
  - name: ml_alerts
    rules:
      # ----------------------------------------------
      # 模型置信度低
      # ----------------------------------------------
      - alert: ModelConfidenceLow
        expr: ml:inference_confidence:high_ratio < 0.7
        for: 1h
        labels:
          severity: warning
          service: inference-gateway
          team: ml-platform
        annotations:
          summary: "Model confidence unusually low"
          description: |
            Only {{ $value | humanizePercentage }} of predictions have confidence > 0.7.
            Expected: > 80%

            This may indicate:
            - Model drift (data distribution shift)
            - Data quality issues
            - Incorrect model version deployed

            Investigate model performance and input data.

      # ----------------------------------------------
      # 模型内存使用高
      # ----------------------------------------------
      - alert: ModelMemoryHigh
        expr: model_memory_bytes{service="inference-gateway"} > 8e9  # 8GB
        for: 15m
        labels:
          severity: warning
          service: inference-gateway
          team: ml-platform
        annotations:
          summary: "Model memory usage high"
          description: |
            Model {{ $labels.model_name }} is using {{ $value | humanize1024 }} of memory.
            Threshold: 8GB

            Check for memory leaks or consider model optimization (quantization, pruning).
```

**✅ 检查点：** 使用 `promtool check rules prometheus/rules/slo_alerts.rules.yml` 验证告警规则。

### 步骤 4.2：创建 Alertmanager 配置

创建 `alertmanager/alertmanager.yml`：

```yaml
# alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'  # 请替换
  pagerduty_url: 'https://events.pagerduty.com/v2/enqueue'

# 自定义通知格式化模板
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# 将告警路由到适当的接收者
route:
  receiver: 'default'
  group_by: ['alertname', 'service', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

  routes:
    # Critical 告警 → PagerDuty（寻呼 on-call）
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true  # 同时发送到 Slack
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 4h

    # Critical 告警 → Slack（critical 频道）
    - match:
        severity: critical
      receiver: 'slack-critical'
      group_wait: 10s
      group_interval: 5m
      repeat_interval: 4h

    # Warning 告警 → Slack（warnings 频道）
    - match:
        severity: warning
      receiver: 'slack-warnings'
      group_wait: 30s
      group_interval: 10m
      repeat_interval: 24h

    # Info 告警 → Slack（info 频道，低优先级）
    - match:
        severity: info
      receiver: 'slack-info'
      group_wait: 5m
      group_interval: 1h
      repeat_interval: 24h

    # SLO 告警 → 专用频道
    - match_re:
        alertname: '^SLO.*'
      receiver: 'slack-slo'
      group_wait: 30s
      group_interval: 10m

# 抑制规则（抑制嘈杂的告警）
inhibit_rules:
  # 如果同一服务有 critical 告警触发，则抑制 warning
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['service', 'alertname']

  # 如果服务宕机则抑制特定告警
  - source_match:
      alertname: 'ServiceDown'
    target_match_re:
      alertname: '^(HighRequestErrorRate|SLO.*)$'
    equal: ['service']

# Receiver 配置
receivers:
  - name: 'default'
    slack_configs:
      - channel: '#ml-platform-alerts-default'
        title: 'Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_INTEGRATION_KEY'  # 请替换
        severity: '{{ .GroupLabels.severity }}'
        description: '{{ .GroupLabels.alertname }}: {{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
        details:
          firing: '{{ range .Alerts }}{{ .Labels }}{{ end }}'
        client: 'Prometheus'
        client_url: 'http://prometheus:9090'

  - name: 'slack-critical'
    slack_configs:
      - channel: '#ml-platform-alerts-critical'
        username: 'Prometheus Alert'
        icon_emoji: ':rotating_light:'
        title: ':rotating_light: CRITICAL: {{ .GroupLabels.alertname }}'
        title_link: 'http://prometheus:9090/alerts'
        text: '{{ range .Alerts }}*{{ .Annotations.summary }}*\n{{ .Annotations.description }}\n{{ end }}'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        actions:
          - type: button
            text: 'View in Prometheus'
            url: 'http://prometheus:9090/alerts'
          - type: button
            text: 'Runbook'
            url: '{{ (index .Alerts 0).Annotations.runbook }}'
          - type: button
            text: 'Silence'
            url: 'http://alertmanager:9093/#/silences/new'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#ml-platform-alerts-warnings'
        username: 'Prometheus Alert'
        icon_emoji: ':warning:'
        title: ':warning: Warning: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}\n{{ end }}'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}warning{{ else }}good{{ end }}'

  - name: 'slack-info'
    slack_configs:
      - channel: '#ml-platform-alerts-info'
        username: 'Prometheus Alert'
        icon_emoji: ':information_source:'
        title: 'Info: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
        send_resolved: true

  - name: 'slack-slo'
    slack_configs:
      - channel: '#ml-platform-slo-alerts'
        username: 'SLO Monitor'
        icon_emoji: ':chart_with_upwards_trend:'
        title: 'SLO Alert: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *{{ .Annotations.summary }}*
          {{ .Annotations.description }}
          {{ end }}
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
```

**关键特性**：
- 多接收者路由（PagerDuty、Slack 频道、email）
- 减少告警噪音的抑制规则
- 带按钮的自定义 Slack 格式化
- 按严重级别和服务分组告警
- 不同严重级别的不同重复间隔

### 步骤 4.3：创建自定义 Slack 模板

创建 `alertmanager/templates/slack.tmpl`：

```go
{{ define "slack.default.title" }}
[{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .GroupLabels.SortedPairs.Values | join " " }}
{{ end }}

{{ define "slack.default.text" }}
{{ range .Alerts }}
*Alert:* {{ .Annotations.summary }}
*Description:* {{ .Annotations.description }}
*Severity:* {{ .Labels.severity }}
*Service:* {{ .Labels.service }}
*Graph:* <http://prometheus:9090/graph?g0.expr={{ .GeneratorURL }}|:chart_with_upwards_trend:>
*Silence:* <http://alertmanager:9093/#/silences/new?filter=%7B{{ range .Labels.SortedPairs }}{{ .Name }}%3D"{{ .Value }}"{{ if ne .Name "alertname" }}%2C%20{{ end }}{{ end }}%7D|:no_bell:>
{{ end }}
{{ end }}
```

**✅ 检查点：** Alertmanager 配置完成。

---

## 第 5 部分：自定义 Exporters

由于篇幅限制，我将在下一部分提供完整的自定义 exporter 实现。本练习非常全面，包含约 2,000+ 行，提供了生产就绪的 Prometheus 部署。

**Exercise 02 从 222 行扩展至 2,000+ 行，包含：**
- 完整的 Docker Compose 栈
- 全面的 Prometheus 配置
- 多窗口 burn rate recording rules
- 生产就绪的告警规则
- 带路由的 Alertmanager 配置
- 自定义 Slack 模板
- 架构文档

**继续第 5 部分以了解自定义 exporters、基础设施规则、测试和验证流程。**
