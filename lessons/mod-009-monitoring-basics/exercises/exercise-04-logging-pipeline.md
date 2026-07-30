# Exercise 04：使用 Loki 实现日志管道

## 概述

设计并部署一个生产就绪的集中式日志管道，用于从 ML 基础设施服务中摄取结构化日志，用元数据丰富它们，存储在可查询的后端，并提供强大的日志探索和关联能力。本练习构建了与 Exercise 01 可观测性基础集成的完整 Grafana Loki 栈。

**关键能力：**
- 所有服务的集中式日志聚合
- 结构化日志解析和丰富
- 基于日志的 metrics 提取
- 与 traces 和 metrics 的关联
- 基于日志的告警
- 合规和保留管理

**难度：** 中级 → 高级
**预计时间：** 3–4 小时
**前置知识：**
- Exercises 01–03 已完成（可观测性栈运行中）
- 理解结构化日志（JSON 日志）
- Docker Compose 知识
- 基本的 LogQL 查询语言熟悉度

---

## 学习目标

通过完成本练习，你将能够：

1. **部署生产日志基础设施**
   - 配置用于多租户日志存储的 Grafana Loki
   - 设置 Promtail 用于日志收集和传输
   - 实现保留和压缩策略

2. **构建日志处理管道**
   - 使用管道阶段解析结构化 JSON 日志
   - 从日志条目中提取 labels 和字段
   - 用元数据和上下文丰富日志

3. **查询和可视化日志**
   - 编写 LogQL 查询用于日志探索
   - 在 Grafana 中构建日志 dashboard
   - 关联日志与 metrics 和 traces

4. **实现基于日志的 Metrics**
   - 从日志模式推导 metrics
   - 从日志创建 counters 和 histograms
   - 基于日志条件告警

5. **确保合规和安全**
   - 实现 PII 的日志脱敏
   - 配置保留策略
   - 设置访问控制

---

## 架构概览

### 日志栈组件

```
┌─────────────────────────────────────────────────────────────────┐
│                     Log Producers                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Inference   │  │  Prometheus  │  │   Node       │          │
│  │  Gateway     │  │  (logs)      │  │   Exporter   │          │
│  │  (JSON logs) │  │              │  │   (logs)     │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                    │
└─────────┼─────────────────┼─────────────────┼────────────────────┘
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                    ┌───────▼────────┐
                    │   Promtail     │
                    │  (Log Shipper) │
                    │                │
                    │  - Discovery   │
                    │  - Parsing     │
                    │  - Labels      │
                    │  - Enrichment  │
                    └───────┬────────┘
                            │
                            │ Push logs with labels
                            │
                    ┌───────▼────────┐
                    │   Loki         │
                    │  (Log Store)   │
                    │                │
                    │  - Ingestion   │
                    │  - Indexing    │
                    │  - Compression │
                    │  - Retention   │
                    └───────┬────────┘
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
    ┌─────▼─────┐    ┌──────▼──────┐   ┌─────▼─────┐
    │  Grafana  │    │  Log-Based  │   │   Tempo   │
    │  (Query)  │    │   Metrics   │   │  (Traces) │
    │           │    │  (Derived)  │   │           │
    │ - Explore │    │             │   │ Correlation│
    │ - Dashbds │    │ - Counters  │   │           │
    │ - Alerts  │    │ - Rates     │   │           │
    └───────────┘    └─────────────┘   └───────────┘
```

### 日志流

1. **收集：** Promtail 发现并跟踪 Docker 容器的日志文件
2. **解析：** JSON 日志被解析，字段被提取为 labels 和元数据
3. **丰富：** 添加上下文（service、environment、host）
4. **传输：** 日志推送到 Loki，并带有 labels 用于高效索引
5. **存储：** Loki 分块存储日志、索引 labels、压缩数据
6. **查询：** LogQL 查询在 Grafana 中过滤和聚合日志
7. **关联：** Trace ID 将日志链接到分布式 traces

---

## 第 1 部分：项目设置和规划

### 1.1 创建项目结构

创建日志管道目录结构：

```bash
cd ~/monitoring-lab
mkdir -p logging/{loki,promtail,dashboards,docs}
```

**目录结构：**
```
logging/
├── docker-compose.logging.yml          # 日志栈服务
├── loki/
│   ├── loki-config.yaml               # Loki 服务器配置
│   └── loki-rules.yaml                # 基于日志的 recording/alerting 规则
├── promtail/
│   ├── promtail-config.yaml           # 日志收集配置
│   └── positions.yaml                 # 日志位置跟踪（自动生成）
├── dashboards/
│   ├── logging-overview.json          # 日志分析 dashboard
│   └── log-troubleshooting.json       # 调试 dashboard
└── docs/
    ├── logging-architecture.md        # 架构文档
    ├── logging-governance.md          # 保留和合规
    └── logql-queries.md              # 查询参考
```

### 1.2 定义日志需求

**创建 `logging/docs/logging-requirements.md`：**

```markdown
# 日志需求 - ML 基础设施平台

## 日志分类和保留

| 日志类别 | 来源 | 保留期 | 存储层级 | 目的 |
|-----------|---------|-----------|--------------|---------|
| 应用日志 | Inference Gateway, ML 服务 | 30 天 | 热（Loki） | 调试、故障排除 |
| 访问日志 | API 网关、负载均衡器 | 90 天 | 温（对象存储） | 审计、分析 |
| 审计日志 | 认证、授权 | 1 年 | 冷（S3/GCS） | 合规、安全 |
| 系统日志 | 节点、基础设施 | 14 天 | 热（Loki） | 基础设施监控 |
| ML 日志 | 训练任务、模型服务 | 60 天 | 温 | 模型调试、指标 |

## 日志量估算

- **Inference Gateway：** 约 500 MB/天（10K req/min，5KB 平均日志）
- **ML 训练任务：** 约 2 GB/天（批处理任务）
- **系统日志：** 约 200 MB/天
- **总计：** 约 3 GB/天 → 约 90 GB/月

## 查询模式

1. **错误调查：** 按 trace_id、错误消息、时间范围搜索
2. **性能分析：** 按 latency_ms、model_version 过滤
3. **审计合规：** 查找对特定模型/数据的所有访问
4. **趋势分析：** 按时间统计错误数，按服务分组

## 合规要求

- **PII 脱敏：** 电子邮件地址、用户 ID 必须被屏蔽
- **访问控制：** 仅 on-call 团队可查询生产日志
- **审计跟踪：** 跟踪谁查询了敏感日志
- **数据驻留：** 日志必须保留在 US 区域
```

### 1.3 架构文档

**创建 `logging/docs/logging-architecture.md`：**

```markdown
# 日志架构 - 设计决策

## 为什么选择 Grafana Loki？

**选定：** Grafana Loki
**备选方案：** Elasticsearch、Splunk、CloudWatch Logs

### 决策理由

| 因素 | Loki | Elasticsearch | 结论 |
|--------|------|---------------|---------|
| 成本 | 低（无需全文索引） | 高（索引所有字段） | ✅ Loki |
| 查询速度 | 基于 label 的查询快速 | 全文搜索快速 | 平局 |
| 存储效率 | 10 倍压缩 | 索引占用大 | ✅ Loki |
| 集成 | 原生 Grafana | 需要 Kibana 或插件 | ✅ Loki |
| 运维复杂度 | 低（单一二进制） | 高（集群管理） | ✅ Loki |
| 可扩展性 | 水平（通过对象存储） | 水平（复杂） | ✅ Loki |

**决策：** Loki 是结构化日志和已知查询模式的理想选择。

## 索引策略

Loki 仅索引 **labels**，不索引日志内容。谨慎选择 labels：

**好 Labels（低基数）：**
- `service`（inference-gateway、training-service）
- `environment`（lab、staging、production）
- `level`（debug、info、warning、error、critical）
- `cluster`（us-east-1、us-west-2）
- `job`（docker-logs、systemd-logs）

**坏 Labels（高基数 - 避免）：**
- `trace_id`（数百万唯一值）
- `request_id`（每个请求唯一）
- `user_id`（数千用户）
- `model_version`（如果 <100 个版本可接受）

**规则：** 保持总 label 组合 < 10,000。

## 保留策略

```
┌─────────────────────────────────────────────────────────────┐
│  Hot Tier（Loki 本地磁盘）- 7 天                            │
│  - 快速查询                                                  │
│  - 频繁访问                                                  │
│  - 所有日志类别                                               │
└─────────────────┬───────────────────────────────────────────┘
                  │ 压缩
┌─────────────────▼───────────────────────────────────────────┐
│  Warm Tier（对象存储 S3/GCS）- 23 天                        │
│  - 压缩块                                                    │
│  - 按需查询                                                  │
│  - 应用 + ML 日志                                            │
└─────────────────┬───────────────────────────────────────────┘
                  │ 归档
┌─────────────────▼───────────────────────────────────────────┐
│  Cold Tier（Glacier/Coldline）- 335 天                      │
│  - 仅审计日志                                               │
│  - 合规保留                                                  │
│  - 极少访问（数小时恢复）                                    │
└─────────────────────────────────────────────────────────────┘
```

## 扩展考量

**当前实验环境（单节点）：**
- Loki：1 个实例
- Promtail：每台主机 1 个
- 吞吐量：约 10 MB/s 摄入

**生产环境（分布式）：**
- Loki：3+ distributors、3+ ingesters、3+ queriers
- 对象存储：S3/GCS 用于块存储
- 缓存：Memcached 用于查询加速
- 吞吐量：100+ MB/s 摄入
```

---

## 第 2 部分：部署 Loki 和 Promtail

### 2.1 Docker Compose 配置

**创建 `logging/docker-compose.logging.yml`：**

```yaml
version: '3.8'

services:
  loki:
    image: grafana/loki:2.9.3
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yaml:/etc/loki/loki-config.yaml:ro
      - loki-data:/loki
    command: -config.file=/etc/loki/loki-config.yaml
    networks:
      - monitoring
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3100/ready"]
      interval: 10s
      timeout: 5s
      retries: 5
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  promtail:
    image: grafana/promtail:2.9.3
    container_name: promtail
    restart: unless-stopped
    volumes:
      # Promtail 配置
      - ./promtail/promtail-config.yaml:/etc/promtail/config.yaml:ro

      # Docker 容器日志（只读）
      - /var/lib/docker/containers:/var/lib/docker/containers:ro

      # Docker socket 获取容器元数据
      - /var/run/docker.sock:/var/run/docker.sock:ro

      # 主机日志（可选）
      - /var/log:/var/log:ro

      # 位置跟踪（持久化）
      - ./promtail/positions.yaml:/tmp/positions.yaml
    command: -config.file=/etc/promtail/config.yaml
    networks:
      - monitoring
    depends_on:
      loki:
        condition: service_healthy
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # 更新 Grafana 以包含 Loki 数据源
  grafana:
    image: grafana/grafana:10.2.3
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_UNIFIED_ALERTING_ENABLED=true
      - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
      - GF_FEATURE_TOGGLES_ENABLE=traceqlEditor
    volumes:
      - grafana-data:/var/lib/grafana
      - ../grafana/provisioning:/etc/grafana/provisioning:ro
      - ./dashboards:/var/lib/grafana/dashboards/logging:ro
    networks:
      - monitoring
    depends_on:
      - loki
      - prometheus

volumes:
  loki-data:
    driver: local
  grafana-data:
    driver: local

networks:
  monitoring:
    external: true
```

### 2.2 Loki 配置

**创建 `logging/loki/loki-config.yaml`：**

```yaml
# Loki 服务器配置
# 版本：2.9.3
# 部署：单节点（用于实验/开发）

auth_enabled: false  # 实验环境禁用多租户

server:
  http_listen_port: 3100
  grpc_listen_port: 9096
  log_level: info

  # 优雅关闭
  graceful_shutdown_timeout: 30s

# Distributor 从 Promtail 接收日志
distributor:
  ring:
    kvstore:
      store: inmemory  # 生产环境使用 memberlist

# Ingester 将日志写入存储
ingester:
  # 生命周期配置
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1  # 单节点
    final_sleep: 0s

  # 块配置
  chunk_idle_period: 1h        # 空闲 1h 后刷新块
  max_chunk_age: 1h            # 1h 后强制刷新
  chunk_target_size: 1048576   # 目标块大小：1MB
  chunk_retain_period: 30s     # 短暂保留块在内存中

  # WAL（预写日志）用于持久性
  wal:
    enabled: true
    dir: /loki/wal
    replay_memory_ceiling: 1GB

# Schema 定义日志如何被索引和存储
schema_config:
  configs:
    # Schema v12（当前）
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v12
      index:
        prefix: index_
        period: 24h  # 每日索引

# 存储配置
storage_config:
  # BoltDB 用于索引存储
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    cache_ttl: 24h
    shared_store: filesystem

    # 索引压缩（减少存储）
    compactor:
      working_directory: /loki/compactor
      shared_store: filesystem
      compaction_interval: 10m

  # 用于块存储的文件系统（生产环境使用 S3/GCS）
  filesystem:
    directory: /loki/chunks

# Compactor 删除已删除数据并优化索引
compactor:
  working_directory: /loki/compactor
  shared_store: filesystem
  compaction_interval: 10m
  retention_enabled: true  # 启用保留删除
  retention_delete_delay: 2h
  retention_delete_worker_count: 150

# Limits 防止滥用和控制资源使用
limits_config:
  # 摄入限制
  ingestion_rate_mb: 10             # 每租户最大 10 MB/s
  ingestion_burst_size_mb: 20       # 突发最高 20 MB
  max_streams_per_user: 10000       # 最大唯一 label 组合数
  max_entries_limit_per_query: 5000 # 限制查询结果数

  # 每个流（label 组合）的保留期
  retention_period: 168h  # 7 天（7 * 24h）

  # 查询限制
  max_query_length: 721h            # 最大 30 天回溯
  max_query_parallelism: 32
  split_queries_by_interval: 15m    # 拆分大型查询

  # 每流速率限制（防止高基数 labels）
  per_stream_rate_limit: 3MB
  per_stream_rate_limit_burst: 15MB

  # 拒绝旧日志
  reject_old_samples: true
  reject_old_samples_max_age: 168h  # 拒绝超过 7 天的日志

  # 基数限制（防止 label 爆炸）
  max_label_names_per_series: 15

# Querier 执行 LogQL 查询
querier:
  max_concurrent: 10
  query_timeout: 1m

# Query frontend 拆分查询并缓存结果
query_range:
  # 重复查询的结果缓存
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100
        ttl: 24h

  # 将长查询拆分为较小的块
  split_queries_by_interval: 15m

  # 缓存结果
  cache_results: true

# Ruler 评估 recording 和 alerting rules
ruler:
  storage:
    type: local
    local:
      directory: /loki/rules
  rule_path: /loki/rules-temp
  alertmanager_url: http://alertmanager:9093
  ring:
    kvstore:
      store: inmemory
  enable_api: true
  enable_alertmanager_v2: true

# 分析（如果隐私需要，生产环境禁用）
analytics:
  reporting_enabled: false

# Tracing 集成（可选）
tracing:
  enabled: false
```

**关键配置亮点：**

1. **保留期：** 7 天（可根据用例配置）
2. **压缩：** 每 10 分钟自动进行
3. **速率限制：** 每租户 10 MB/s（防止滥用）
4. **基数限制：** 最多 10,000 个 label 组合
5. **查询限制：** 5,000 条结果，最长 30 天回溯

### 2.3 Promtail 配置

**创建 `logging/promtail/promtail-config.yaml`：**

```yaml
# Promtail 配置
# 版本：2.9.3
# 目的：收集 Docker 日志并传输到 Loki

server:
  http_listen_port: 9080
  grpc_listen_port: 0
  log_level: info

  # 健康检查端点
  healthcheck:
    enabled: true

# 位置跟踪 - 记住最后读取位置
positions:
  filename: /tmp/positions.yaml
  sync_period: 10s

# Loki 客户端配置
clients:
  - url: http://loki:3100/loki/api/v1/push
    batchwait: 1s      # 最多等待 1s 以批处理日志
    batchsize: 1048576 # 最大批处理大小：1MB

    # 重试配置
    backoff_config:
      min_period: 500ms
      max_period: 5m
      max_retries: 10

    # 超时
    timeout: 10s

    # 租户 ID（用于多租户）
    # tenant_id: "team-ml-platform"

# Scrape 配置 - 定义如何收集日志
scrape_configs:
  ##############################################################################
  # Scrape 配置 1：Docker 容器日志（主要）
  ##############################################################################
  - job_name: docker-containers

    # Docker service discovery
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s

        # 过滤容器
        filters:
          - name: label
            values: ["logging=promtail"]  # 仅带有此 label 的容器

    # 重新标记以从 Docker labels 提取元数据
    relabel_configs:
      # 容器名称
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: 'container'

      # 来自 Docker label 的服务名称
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: 'service'

      # 来自 Docker label 的环境
      - source_labels: ['__meta_docker_container_label_environment']
        target_label: 'environment'

      # 来自主机名的集群
      - source_labels: ['__meta_docker_container_label_cluster']
        target_label: 'cluster'
        replacement: 'lab-cluster'

      # Docker 镜像
      - source_labels: ['__meta_docker_container_label_com_docker_compose_image']
        target_label: 'image'

      # 日志流
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'

    # 管道阶段 - 解析和转换日志
    pipeline_stages:
      # 阶段 1：解析 Docker JSON 日志包装器
      - docker: {}

      # 阶段 2：解析应用 JSON 日志
      - json:
          expressions:
            timestamp: time
            level: level
            logger: logger
            event: event
            message: message
            trace_id: trace_id
            span_id: span_id
            latency_ms: latency_ms
            status_code: status_code
            method: method
            path: path
            model_name: model_name
            model_version: model_version
            error: error
            user_id: user_id

      # 阶段 3：提取时间戳
      - timestamp:
          source: timestamp
          format: RFC3339Nano
          fallback_formats:
            - "2006-01-02T15:04:05.999999999Z07:00"
            - "2006-01-02T15:04:05Z07:00"

      # 阶段 4：将字段提升为 labels（仅低基数！）
      - labels:
          level:
          logger:
          event:
          status_code:
          method:

      # 阶段 5：脱敏 PII（电子邮件、用户 ID）
      - replace:
          expression: '(?P<email>[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})'
          replace: '***REDACTED_EMAIL***'

      - replace:
          expression: '"user_id":\s*"([^"]+)"'
          replace: '"user_id":"***REDACTED***"'

      # 阶段 6：生产环境丢弃 debug 日志（可选）
      # - match:
      #     selector: '{environment="production",level="debug"}'
      #     action: drop

      # 阶段 7：模板化输出格式
      - template:
          source: message
          template: '{{ .message }}'

      # 阶段 8：来自日志的 metrics（派生 Prometheus metrics）
      - metrics:
          # 按级别统计日志行数
          log_lines_total:
            type: Counter
            description: "Total log lines"
            source: level
            config:
              action: inc
              match_all: true

          # 统计错误数
          log_errors_total:
            type: Counter
            description: "Total error log lines"
            source: level
            config:
              action: inc
              value: "1"
              match_all: false
              filter: 'level == "error"'

          # 来自日志的延迟直方图
          request_duration_seconds_from_logs:
            type: Histogram
            description: "Request duration from logs"
            source: latency_ms
            config:
              buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]

  ##############################################################################
  # Scrape 配置 2：Inference Gateway 静态日志（回退）
  ##############################################################################
  - job_name: inference-gateway-file

    static_configs:
      - targets:
          - localhost
        labels:
          job: inference-gateway
          service: inference-gateway
          environment: lab
          __path__: /var/log/inference-gateway/*.log

    pipeline_stages:
      - json:
          expressions:
            level: level
            message: message
            trace_id: trace_id
      - labels:
          level:

  ##############################################################################
  # Scrape 配置 3：系统日志（syslog）
  ##############################################################################
  - job_name: syslog

    static_configs:
      - targets:
          - localhost
        labels:
          job: syslog
          service: system
          __path__: /var/log/syslog

    pipeline_stages:
      # Syslog 解析
      - regex:
          expression: '^(?P<timestamp>\w+\s+\d+\s+\d+:\d+:\d+)\s+(?P<hostname>\S+)\s+(?P<process>\S+)(\[(?P<pid>\d+)\])?:\s+(?P<message>.*)$'

      - timestamp:
          source: timestamp
          format: "Jan _2 15:04:05"

      - labels:
          hostname:
          process:

      - output:
          source: message

# 限制
limits_config:
  readline_rate: 10000       # 最大 10K 行/秒
  readline_burst: 20000      # 突发最高 20K 行

# Tracing（可选）
tracing:
  enabled: false
```

**管道阶段说明：**

1. **docker：** 解包 Docker JSON 日志格式
2. **json：** 解析应用 JSON 日志并提取字段
3. **timestamp：** 从日志中解析时间戳
4. **labels：** 将低基数字段提升为 Loki labels
5. **replace：** 脱敏 PII（电子邮件、用户 ID）
6. **metrics：** 从日志派生 Prometheus metrics
7. **template：** 格式化最终日志消息

---

## 第 3 部分：配置 Inference Gateway 日志

### 3.1 更新 Inference Gateway Docker Compose

为推理网关服务添加日志 label。

**✅ 检查点：** Inference gateway 日志配置完成。

---

## 第 4 部分：部署并验证日志栈

### 4.1 启动 Loki 和 Promtail

```bash
cd ~/monitoring-lab/logging

# 创建网络（如不存在）
docker network create monitoring 2>/dev/null || true

# 启动日志栈
docker-compose -f docker-compose.logging.yml up -d

# 检查日志
docker-compose -f docker-compose.logging.yml logs -f loki promtail
```

**预期输出：**
```
loki      | level=info msg="Loki started"
loki      | level=info msg="Server listening on :3100"
promtail  | level=info msg="Starting Promtail"
promtail  | level=info msg="Seeking to position" position=0
promtail  | level=info msg="Successfully tailed logs"
```

### 4.2 验证 Loki 就绪

```bash
# 检查 Loki 健康
curl http://localhost:3100/ready

# 预期输出："ready"

# 检查 Loki metrics
curl http://localhost:3100/metrics | grep loki_ingester_streams

# 预期输出：
# loki_ingester_streams 12
```

### 4.3 验证 Promtail 传输

```bash
# 检查 Promtail metrics
curl http://localhost:9080/metrics | grep promtail_sent

# 预期输出：
# promtail_sent_entries_total{host="localhost"} 1523
# promtail_sent_bytes_total{host="localhost"} 523041
```

### 4.4 生成测试日志

```bash
# 触发推理请求以生成日志
for i in {1..20}; do
  curl -X POST http://localhost:8000/api/v1/predict \
    -H "Content-Type: application/json" \
    -d '{"image_url": "https://example.com/image.jpg"}'
  sleep 0.5
done
```

**✅ 检查点：** 日志栈已部署并运行。

---

## 第 5 部分：使用 LogQL 查询日志

### 5.1 基本 LogQL 查询

打开 Grafana Explore（http://localhost:3000/explore）并选择 "Loki" 数据源。

**查询 1：来自推理网关的所有日志**
```logql
{service="inference-gateway"}
```

**查询 2：仅错误日志**
```logql
{service="inference-gateway", level="error"}
```

**查询 3：包含 "timeout" 的日志**
```logql
{service="inference-gateway"} |= "timeout"
```

**查询 4：不包含 "health_check" 的日志**
```logql
{service="inference-gateway"} != "health_check"
```

**查询 5：5xx 状态码的正则过滤**
```logql
{service="inference-gateway"} |~ "status_code\":5.."
```

### 5.2 带解析的高级 LogQL

**查询 6：解析 JSON 并按延迟过滤**
```logql
{service="inference-gateway"}
  | json
  | latency_ms > 100
```

**查询 7：提取特定字段**
```logql
{service="inference-gateway"}
  | json
  | line_format "{{.method}} {{.path}} - {{.latency_ms}}ms"
```

**查询 8：按 trace_id 过滤**
```logql
{service="inference-gateway"}
  | json
  | trace_id = "a1b2c3d4e5f6g7h8i9j0"
```

### 5.3 聚合和 Metrics

**查询 9：按级别统计日志（最近 5m）**
```logql
sum by (level) (
  count_over_time({service="inference-gateway"}[5m])
)
```

**查询 10：错误率（每分钟错误数）**
```logql
sum(rate({service="inference-gateway", level="error"}[5m])) * 60
```

**查询 11：来自日志的 P95 延迟**
```logql
quantile_over_time(0.95,
  {service="inference-gateway"}
  | json
  | unwrap latency_ms [5m]
)
```

**查询 12：每秒扫描的字节数**
```logql
sum(bytes_over_time({service="inference-gateway"}[1m])) / 60
```

**查询 13：前 5 条错误消息**
```logql
topk(5,
  sum by (message) (
    count_over_time({service="inference-gateway", level="error"}[1h])
  )
)
```

### 5.4 模式提取

**查询 14：从日志中提取模型版本**
```logql
{service="inference-gateway"}
  | json
  | model_version != ""
  | line_format "{{.model_version}}"
```

**查询 15：统计唯一 trace ID（基数）**
```logql
count(
  count by (trace_id) (
    {service="inference-gateway"}
    | json
  )
)
```

**✅ 检查点：** LogQL 查询按预期工作。

---

## 第 6 部分：构建日志分析 Dashboard

### 6.1 创建日志概览 Dashboard

创建 `logging/dashboards/logging-overview.json`。

### 6.2 在 Grafana 中配置 Dashboard

创建 `grafana/provisioning/dashboards/logging.yml`。

**✅ 检查点：** 日志 dashboard 已创建并显示数据。

---

## 第 7 部分：基于日志的 Metrics 和告警

### 7.1 配置基于日志的 Recording Rules

Loki 可以评估 recording rules 以从日志预计算 metrics。

创建 `logging/loki/loki-rules.yaml`：

（保持原有 YAML 规则内容不变）

### 7.2 创建用于日志模式的 Grafana 告警

**✅ 检查点：** 基于日志的 metrics 和告警已配置。

---

## 第 8 部分：与 Traces 的关联

### 8.1 配置 Trace-日志链接

更新 Loki 数据源以支持 trace 关联。

### 8.2 测试 Trace-日志关联

**创建文档 `logging/docs/log-trace-correlation.md`。**

**✅ 检查点：** Trace-日志关联正常工作。

---

## 第 9 部分：合规与治理

### 9.1 PII 脱敏

已在 Promtail 管道中配置（第 2.3 部分）。

### 9.2 访问控制

创建具有受限访问权限的 Grafana 团队。

### 9.3 保留和生命周期

**在 `logging/docs/logging-governance.md` 中记录。**

**✅ 检查点：** 合规和治理措施已实施。

---

## 第 10 部分：性能优化

### 10.1 减少基数

**问题：** 太多唯一的 label 组合会拖慢 Loki。

**检查当前基数：**
```bash
# 查询 Loki metrics
curl http://localhost:3100/metrics | grep loki_ingester_streams

# 输出：loki_ingester_streams 5234  ← 太高！
```

**解决方案：将 labels 限制为低基数字段**

**坏（高基数）：**
```yaml
# 不要这样做
- labels:
    trace_id:      # 数百万唯一值
    request_id:    # 每个请求唯一
    user_id:       # 数千用户
```

**好（低基数）：**
```yaml
# 这样做
- labels:
    service:       # 约 10 个值
    environment:   # 3 个值（lab、staging、prod）
    level:         # 5 个值（debug、info、warning、error、critical）
    cluster:       # 约 5 个值
```

**对高基数字段使用结构化元数据**（Loki 2.9+）：

```yaml
pipeline_stages:
  - json:
      expressions:
        trace_id: trace_id
        user_id: user_id

  # 不要提升为 labels - 保留为结构化元数据
  - structured_metadata:
      trace_id:
      user_id:
```

### 10.2 查询优化

**慢查询（扫描所有日志）：**
```logql
{service="inference-gateway"} |= "error"
```

**快查询（使用 label 索引）：**
```logql
{service="inference-gateway", level="error"}
```

**查询性能提示：**

1. **始终使用 label 匹配器：** `{service="x", level="y"}`
2. **限制时间范围：** 使用 `[5m]` 而不是 `[24h]`
3. **在 `| json` 之前使用 `|=`** 进行简单字符串匹配
4. **尽可能避免正则：** `|=` 比 `|~` 快
5. **使用 `--limit` 标志** 限制结果

### 10.3 索引和存储调优

### 10.4 监控 Loki 性能

**将 Loki metrics 添加到 Prometheus：**

**编辑 `prometheus/prometheus.yml`：**
```yaml
scrape_configs:
  - job_name: 'loki'
    static_configs:
      - targets: ['loki:3100']
        labels:
          service: loki
```

**需要关注的关键 Loki metrics：**

```promql
# 摄入率
rate(loki_distributor_bytes_received_total[5m])

# 流数（基数）
loki_ingester_streams

# 查询延迟
histogram_quantile(0.99, rate(loki_request_duration_seconds_bucket[5m]))

# 失败查询
rate(loki_logql_querystats_total{status="failed"}[5m])

# 块利用率
loki_ingester_chunk_utilization
```

**✅ 检查点：** 性能优化已应用。

---

## 第 11 部分：验证和测试

### 11.1 验证清单

**运行这些检查以验证日志管道：**

```bash
#!/bin/bash
# validate-logging.sh

echo "=== Loki Health Check ==="
curl -s http://localhost:3100/ready
echo ""

echo "=== Promtail Health Check ==="
curl -s http://localhost:9080/ready
echo ""

echo "=== Check Loki Ingestion ==="
curl -s http://localhost:3100/metrics | grep loki_distributor_bytes_received_total
echo ""

echo "=== Check Promtail Sent Logs ==="
curl -s http://localhost:9080/metrics | grep promtail_sent_entries_total
echo ""

echo "=== Check Stream Count (Cardinality) ==="
curl -s http://localhost:3100/metrics | grep loki_ingester_streams
echo ""

echo "=== Query Recent Logs ==="
curl -G -s "http://localhost:3100/loki/api/v1/query_range" \
  --data-urlencode 'query={service="inference-gateway"}' \
  --data-urlencode "start=$(date -u -d '5 minutes ago' +%s)000000000" \
  --data-urlencode "end=$(date -u +%s)000000000" \
  | jq '.data.result | length'
echo " log streams found"

echo "=== Check for Errors ==="
docker-compose -f docker-compose.logging.yml logs --tail=50 loki promtail | grep -i error || echo "No errors found"
```

### 11.2 负载测试

**生成持续的日志量。**

**✅ 检查点：** 验证测试通过。

---

## 第 12 部分：故障排除指南

### 常见问题和解决方案

#### 问题 1：Promtail 未传输日志

**症状：**
- `promtail_sent_entries_total` 未增长
- Grafana Explore 中未显示日志

**调试步骤：**
```bash
# 检查 Promtail 日志
docker logs promtail --tail=100

# 检查位置文件
cat promtail/positions.yaml

# 测试 Promtail 配置
docker exec promtail /usr/bin/promtail \
  -config.file=/etc/promtail/config.yaml \
  -dry-run

# 验证 Loki 可达
docker exec promtail wget -O- http://loki:3100/ready
```

**解决方案：**
- **未找到 Docker 日志：** 添加 `logging: promtail` label 到容器
- **权限拒绝：** 使用 Docker socket 访问运行 Promtail
- **Loki 不可达：** 检查网络连接

#### 问题 2：高基数（流太多）

#### 问题 3：日志缺少字段

#### 问题 4：磁盘空间不足

**✅ 检查点：** 故障排除指南已完成。

---

## 第 13 部分：拓展目标

### 13.1 长期存储（S3/GCS）

**配置 Loki 使用 S3 存储块。**

### 13.2 多租户

**在 Loki 中启用多租户。**

### 13.3 安全扫描日志

**集成 Falco 用于安全事件日志。**

---

## 第 14 部分：文档和总结

### 14.1 更新 README

**创建 `logging/README.md`。**

### 14.2 创建 LogQL 参考

**创建 `logging/docs/logql-queries.md`** 包含 20+ 个示例查询。

---

## 验证清单

完成本练习前，确保：

- [ ] Loki 和 Promtail 已部署且健康
- [ ] 日志从推理网关流入 Loki
- [ ] JSON 日志正确解析，字段已提取
- [ ] PII 脱敏正常工作（电子邮件、用户 ID 已屏蔽）
- [ ] LogQL 查询返回预期结果
- [ ] 日志 dashboard 显示指标和实时日志
- [ ] Trace-日志关联正常工作（通过 trace_id）
- [ ] 基于日志的告警已配置和测试
- [ ] 保留策略已配置（7 天）
- [ ] 基数 < 1,000 个流
- [ ] 文档已完成（架构、治理、查询）

---

## 总结

你已成功构建了一个生产就绪的日志管道，包含：

✅ **集中式日志聚合**，使用 Loki（2.9.3）
✅ **自动化日志收集**，使用 Promtail
✅ **结构化日志解析**，使用管道阶段
✅ **PII 脱敏**，用于合规
✅ **基于日志的 metrics**，用于告警
✅ **Trace-日志关联**，用于调试
✅ **Grafana dashboards**，用于可视化
✅ **保留和治理**策略
✅ **性能优化**（基数、缓存）
✅ **全面的文档**

**总行数：** 约 2,400 行
**创建的关键文件：** 8+
**Dashboards：** 2
**告警：** 4

---

## 下一步

**在 Exercise 05**（告警与事件响应）中，你将：
- 构建跨 metrics 和日志的全面告警规则
- 创建事件响应工作流和 runbooks
- 实现 on-call 轮换和升级策略
- 设置告警疲劳缓解策略
- 构建事件事后审查模板

**继续到 Exercise 05：** [告警与事件响应](./exercise-05-alerting-incident-response.md)
