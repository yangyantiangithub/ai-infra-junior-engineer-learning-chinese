# Lecture 03：面向 AI 基础设施的 Grafana Dashboard 和可视化

## 讲座概述
Grafana 将原始 Prometheus metrics 转化为可操作的洞察，结合了灵活的 Dashboard、丰富的可视化选项和告警工作流。本讲座展示了如何设计加速 on-call 响应、向利益相关者传达平台健康状态以及捕捉 AI/ML 工作负载细微差别的 Dashboard。你将学习 Dashboard 架构、面板组成、Grafana 数据建模、模板化、共享、配置以及与 Prometheus 管道互补的告警配置。

**预计阅读时间：** 70–85 分钟  
**动手配套实验：** Exercise 03 – Grafana Dashboard  
**前置知识：** Lectures 01–02、熟悉 Prometheus 查询、了解目标 SLI/SLO。

---

## 1. Grafana 简介

### 1.1 为什么选择 Grafana？
- **统一的可观测性：** 连接到 Prometheus、Loki、Tempo、Elasticsearch、CloudWatch、Datadog 等
- **可组合的 Dashboard：** 拖放面板、可重复布局、变量模板化
- **告警和报告：** 定义阈值、路由通知、安排报告
- **协作：** 共享 Dashboard 快照、注释时间线、嵌入文档
- **基础设施即代码：** 通过 JSON 或 Grafana provisioning 文件配置 Dashboard

### 1.2 Grafana 架构
- **Grafana server：** 处理身份验证、数据源配置、Dashboard 渲染
- **后端数据源：** Prometheus、Loki 等。Grafana 默认不存储时间序列数据（告警状态除外）
- **Dashboard 和文件夹：** 存储在 Grafana 的数据库中或通过 provisioned 文件存储
- **告警和通知策略：** 集中管理告警规则和联系人点

Grafana 可以通过 Docker、Kubernetes 或托管服务（Grafana Cloud、AWS Managed Grafana）部署。建议使用多个环境（dev/staging/prod）在生产之前验证 Dashboard。

---

## 2. Dashboard 策略和信息架构

### 2.1 Dashboard 分类
1. **着陆 Dashboard（执行概览）：**
   - 业务 KPI、SLO 遵守情况、事件状态
   - 适用于领导层、产品管理
2. **服务 Dashboard（On-call 分流）：**
   - 高基数指标、延迟直方图、错误分解、资源饱和
   - 面向 on-call 工程师
3. **子系统 Dashboard（聚焦）：**
   - 特定组件（特征存储、训练管道、推理网关）
   - 供所有者深入分析
4. **操作 Runbooks：**
   - 针对重复任务的面板 + 说明（容量规划、部署就绪）
5. **实验/分析 Dashboard：**
   - 模型性能指标、A/B 测试结果、成本 Dashboard

### 2.2 布局原则
- **叙事流程：** 从顶层健康状态排列面板到细粒度诊断（从左到右或从上到下）
- **聚焦行动：** On-call Dashboard 应强调 SLI 合规、error budget burn、告警、容量
- **避免杂乱：** 每行限制 6-8 个面板；使用选项卡管理补充数据
- **响应式设计：** 使用行折叠、导航链接和一致的面板高度以支持不同屏幕尺寸
- **颜色语义：** 红色=严重、橙色=警告、绿色=健康；跨 Dashboard 保持一致阈值

### 2.3 跨团队协作
- 定义所有权元数据（面板描述包含所有者 + Slack 频道）
- 建立审查节奏（例如：月度 Dashboard 审计）
- 收集利益相关者的反馈（他们试图回答什么问题？）

---

## 3. 数据源配置

### 3.1 Prometheus 数据源
1. 访问 Grafana UI → **Configuration** → **Data sources** → **Add data source** → Prometheus
2. 提供 URL（例如：Docker Compose 中的 `http://prometheus:9090`）
3. 配置身份验证（basic auth、TLS 证书、bearer token）
4. 启用/禁用查询缓存（Grafana OSS v10+）
5. 保存并测试连接

### 3.2 其他数据源
- **Loki：** 用于日志；结合面板关联 metrics 和日志事件
- **Tempo/Jaeger：** 分布式 tracing；使用 exemplars 链接到 traces
- **Elasticsearch/OpenSearch：** 查询日志索引并通过 Loki/Elastic 面板连接
- **PostgreSQL/BigQuery：** 用于业务指标、成本数据

统一的 Dashboard 通常结合多个数据源以获得上下文：metrics + logs + traces + 事件。

---

## 4. 构建驱动行动的面板

### 4.1 面板类型
- **Time series：** 默认的 Prometheus 可视化；支持注释、堆叠序列
- **Stat：** 带阈值的单值；非常适合 SLO 合规、成本
- **Table：** 显示 Top-N 列表、聚合指标（`sum by` 结果）
- **Bar gauge：** 可视化分布、队列长度
- **Heatmap：** 跨群体的模型性能、GPU 利用率与时间
- **Histogram：** 延迟桶、推理时间分布
- **Logs & Traces：** 用于 Loki、Tempo 的嵌入面板
- **Geomap：** 区域流量分布（如果有地理空间数据）

### 4.2 最佳实践
- 标准化单位（ms vs 秒、bytes vs mebibytes）
- 设置适当的时间范围（on-call 默认 6 小时、执行 Dashboard 30 天）
- 提供面板描述和钻取指南（例如："点击通过 exemplar 在 Tempo 中查看 traces"）
- 对齐相似面板的坐标轴以比较趋势
- 使用转换来丰富数据（计算百分比、重命名字段、合并结果）

### 4.3 示例面板：推理延迟
- **查询：** `histogram_quantile(0.99, sum(rate(ai_infra_inference_latency_seconds_bucket[5m])) by (le))`
- **可视化：** 时间序列，阈值设为 200ms（警告）、300ms（严重）
- **注释：** 通过 Grafana 注释部署事件（链接到 Git SHA、变更日志）
- **转换：** 添加 7 天移动平均线用于趋势检测

### 4.4 示例面板：GPU 集群健康
- **查询：** `avg by (cluster, gpu) (DCGM_FI_DEV_GPU_UTIL)`
- **面板类型：** Heatmap（GPU ID vs 时间）
- **阈值：** 高亮 >85% 利用率持续 >30 分钟
- **钻取：** 面板链接到带有 node exporter 指标的 GPU 节点 Dashboard

---

## 5. Dashboard 模板化与变量

### 5.1 变量类型
- **Query variables：** 来自 Prometheus 的动态列表（例如：`label_values(job)`）
- **Constant variables：** 跨面板复用的硬编码值
- **Interval variables：** 控制 range vector 持续时间（例如：`5m`、`1h`）
- **Custom variables：** 手动枚举（`prod`、`staging`、`dev`）
- **Datasource variables：** 无需复制 Dashboard 即可在集群间切换

### 5.2 创建变量
1. Dashboard 设置 → Variables → Add variable
2. 示例：使用 label values 的 `cluster` 变量
   ```promql
   label_values(kube_pod_info, cluster)
   ```
3. 在查询中使用 `${cluster}`：`rate(ai_infra_inference_requests_total{cluster="${cluster}"}[5m])`
4. 链式变量（例如：选择 `team`，按 team label 过滤 `service` 变量）

### 5.3 重复面板和行
- 跨变量值重复面板（按区域、按模型）
- 结合动态阈值（面板覆盖）
- 确保 Dashboard 保持性能（对高基数限制重复；考虑通过自定义过滤器进行分页）

---

## 6. Grafana 告警

### 6.1 告警规则类型
- **Grafana 管理告警（统一告警）：** 跨数据源工作
- **Prometheus 告警集成：** 使用 Prometheus 告警规则；Grafana 通过 Alertmanager 数据源显示

### 6.2 创建告警规则
1. 选择面板 → Alert 标签 → "Create alert rule from panel"
2. 定义查询、缩减（last、avg）、条件（高于/低于阈值）
3. 配置评估间隔、无数据行为、持续持续时间
4. 设置告警规则 labels（service、severity、environment）
5. 映射到联系人点（Slack、email、PagerDuty）

**示例：** 当推理错误率 >1% 持续 10 分钟时告警
```promql
sum(rate(ai_infra_inference_errors_total{cluster="${cluster}"}[5m])) /
sum(rate(ai_infra_inference_requests_total{cluster="${cluster}"}[5m])) * 100
```
条件：`WHEN last() OF query (A) IS ABOVE 1`。

### 6.3 告警分组和路由
- 配置通知策略（Grafana Alerting UI）：
  - 按 `cluster` 和 `service` 分组告警
  - 将 severity=critical 路由到 on-call 寻呼器；severity=warning 路由到 Slack 频道
- 为计划维护设置静默窗口；使用 labels 实现自动化（例如：`maintenance="true"` 抑制告警）

### 6.4 告警 Runbooks
- 使用面板描述或注释链接到 runbook（`go/runbook-inference-latency`）
- 包含步骤：日志查询、Dashboard 部分、回滚

---

## 7. 将 Dashboard 作为代码配置

### 7.1 为什么要配置？
- 确保环境间的一致性
- 在 Git 中实现代码审查、测试和版本控制
- 通过 CI/CD 或 GitOps 自动化部署（例如：Argo CD、Flux）

### 7.2 文件夹配置
`provisioning/dashboards/dashboard.yml`：
```yaml
apiVersion: 1
providers:
  - name: ai-infra-dashboards
    folder: AI Infrastructure
    type: file
    disableDeletion: false
    allowUiUpdates: false
    options:
      path: /var/lib/grafana/dashboards
```

Dashboard JSON 存储在 `/var/lib/grafana/dashboards/` 下。通过 Grafana UI 生成 JSON（Dashboard → Share → Export）。

### 7.3 Terraform Provider
- 使用 `grafana_dashboard` 资源通过 Terraform 管理 Dashboard
- 结合 `grafana_folder`、`grafana_data_source` 和 `grafana_notification_policy`

### 7.4 测试 Dashboard 即代码
- 使用 `grafana-toolkit` 或 `grizzly` 验证 JSON Dashboard
- 添加 CI 检查以 lint JSON、确认必需面板存在、变量查询有效

---

## 8. 使用注释和 Exemplars 增强 Dashboard

### 8.1 注释
- 直接在时间序列图表上标记部署、事件、功能开关
- 与 GitOps 管道集成（例如：Argo CD 通过 Grafana API 发布注释）
- 提供上下文："Model version 2025-10-17 deployed" 或 "Event INC-2245 resolved"

### 8.2 Exemplars
- 通过在 Prometheus 中启用 exemplars（OpenTelemetry 检测）将指标与 trace exemplars 相结合
- 悬停在高延迟数据点上 → 链接到 Tempo/Jaeger 中的特定 trace
- 通过将指标与请求 traces 关联，帮助更快地定位延迟峰值根因

---

## 9. 案例研究和面板配方

### 9.1 推理网关 Dashboard
**部分：**
1. **SLI 概览：** 99% 延迟、错误率、请求量、剩余 error budget 的 Stat 面板
2. **延迟分解：** P50/P90/P99 时间序列、直方图桶热力图
3. **流量和错误：** 按状态码分请求率的堆叠面积图；按区域列出前十大错误码
4. **资源：** GPU 利用率热力图、Pod CPU/内存饱和
5. **实验：** 功能开关状态、金丝雀与基线对比
6. **告警和 Runbooks：** 链接到开放事件、runbook 链接、寻呼器状态的面板

**关键面板：**
- `histogram_quantile(0.99, sum(rate(ai_infra_inference_latency_seconds_bucket[5m])) by (le))`
- `sum by (status) (rate(ai_infra_inference_requests_total[5m]))`
- `avg by (deployment) (container_cpu_utilization_seconds_total{container="inference-gateway"})`

### 9.2 训练管道 Dashboard
聚焦编排器（Airflow/Kubeflow）：
1. DAG 成功/失败比率
2. 按类型的任务持续时间（ETL、特征工程、训练）
3. 资源使用情况（CPU/GPU 小时）
4. 队列长度和积压指标
5. 每次运行的成本（结合 FinOps 数据源）
6. 告警摘要（延迟的 DAG、重复的失败）

### 9.3 数据质量 Dashboard
利用 Great Expectations / 自定义检查的指标：
- 特征表的新鲜度（stat 面板 + sparklines）
- 每个特征的空值比率趋势
- Schema 漂移检测（包含最新告警的表）
- 按严重程度的数据验证失败计数
- 影响分析（受影响的下游模型）

---

## 10. 访问控制与协作

### 10.1 RBAC
- Grafana 角色：Viewer、Editor、Admin
- 使用文件夹权限限制敏感 Dashboard（例如：合规指标）
- 通过 OAuth、SAML 或 LDAP 集成 SSO（Okta、Azure AD）

### 10.2 共享 Dashboard
- **Snapshot sharing：** 生成 JSON 快照；在外部共享前清除敏感数据
- **Embed 模式：** 用于内部门户的 iFrames（确保身份验证）
- **Public dashboards：** 谨慎使用；仅限于经过清理的数据

### 10.3 文档
- 每个 Dashboard 应链接到 README 或 Confluence 中的文档
- 提供"如何解读此 Dashboard"部分：什么样子是正常的、已知注意事项
- 为培训材料存储屏幕截图更新（on-call runbook）

---

## 11. 性能优化

### 11.1 查询优化
- 为重型 PromQL 表达式使用 recording rules
- 缓存查询（Grafana Enterprise/Cloud）或在带查询缓存的 Mimir/Thanos 前运行 Grafana
- 限制昂贵面板的 Dashboard 时间范围；使用面板特定的时间范围
- 避免使用高基数查询重复面板

### 11.2 Dashboard 性能诊断
- Grafana Explore → `Profile` 检查查询时间
- 检查 Prometheus 指标：`grafana_http_request_duration_seconds`
- 监控 Grafana 后端负载（CPU、内存）
- 评估嵌套转换，如果 Dashboard 缓慢则降低复杂度

---

## 12. 集成日志和 Traces

### 12.1 Logs Panel（Loki）
- 添加 Loki 数据源
- 创建带日志查询的面板：`{app="inference-gateway", level="error"} |= "timeout"`
- 使用日志 labels 按区域、模型、客户进行透视
- 结合 Dashboard 链接的 Prometheus 面板

### 12.2 Trace Panel（Tempo/Jaeger）
- 查询最慢的 spans：`service="inference-gateway" operation="GET /predict"`
- 通过使用 OpenTelemetry 检测服务启用 exemplars；将 trace ID 附加到 Prometheus 指标（`exemplars`）

### 12.3 统一可观测性流程
- 面板关联 metrics（延迟）、logs（错误）、trace（调用图）。提供"在 Tempo 中调查"按钮
- 在 runbook 中记录工作流：从 metrics 面板开始 → 点击 exemplar → 检查 trace → 打开相关日志

---

## 13. 合规、审计与成本可见性

### 13.1 审计跟踪
- Grafana 保留审计日志（企业功能）或与捕获 Dashboard 更改、用户操作的日志管道集成
- 确保跟踪 PII 的 Dashboard 遵循治理（脱敏、受限视图）

### 13.2 成本 Dashboard
- 导入成本数据（AWS CUR、GCP 账单导出）
- 将摄入成本、GPU 成本、存储成本与使用指标结合
- 计算每 1000 次预测的成本、每训练周期的成本
- 为 FinOps 和领导层提供支出与预算的可见性

### 13.3 合规监控
- 为安全策略遵守创建 Dashboard：TLS 证书过期、未授权访问尝试、合规指标漂移（例如：公平性）
- 配合告警以应对合规违规

---

## 14. 实践检查清单
- [ ] 添加 Prometheus、Loki 和 Tempo 数据源并验证连接
- [ ] 为环境、集群、团队和模型创建模板变量
- [ ] 构建推理网关 Dashboard，包含延迟、错误率、饱和度和 SLO 面板
- [ ] 配置 Grafana 告警，错误率 >1% 持续 10 分钟路由到 on-call
- [ ] 通过代码配置 Dashboard 并提交到 Git 仓库
- [ ] 在面板描述中记录 Dashboard 目的、所有者和 runbook 链接
- [ ] 安排月度 Dashboard 审查以修剪过时的面板

---

## 15. 知识检查
1. 你会如何为管理层与 on-call 工程师构建 Dashboard？提供具体示例
2. 什么时候应该依赖 recording rules vs 在 Grafana 中实时 PromQL 查询？
3. 描述添加按 `team` 过滤 Dashboard 的变量的步骤
4. 解释 exemplars 如何将指标链接到 traces，以及为什么它们对延迟调查很重要
5. 概述将 Dashboard 作为代码配置并在部署前验证的流程

---

## 16. 附加资源
- Grafana Labs Tutorials (https://grafana.com/tutorials/)
- "Building Effective Dashboards"（GrafanaCON talks）
- Grafana Play (https://play.grafana.org) 查看实时示例
- "Observability Engineering"（Dashboard 设计章节）
- Grafana Terraform provider 文档
- Honeycomb + Grafana 关于 SLO Dashboard 的博客文章

---

## 17. 总结
- Grafana 弥合了原始指标与人类决策之间的差距
- 以明确的目的、受众和可操作洞察来设计 Dashboard
- 利用模板化、注释和 exemplars 增强态势感知
- 集成日志、tracing 和成本指标实现全面的可观测性
- 将 Dashboard 视为代码以确保可靠性和协作

在 Lecture 04 中，我们将通过处理日志管道、告警工作流和 ML 特定监控实践来完成可观测性栈。
