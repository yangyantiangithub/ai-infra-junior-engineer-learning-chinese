# Lecture 04：日志、告警和面向 ML 的监控工作流

## 讲座概述
Logs 通过记录上下文事件、错误详情和审计跟踪来完善可观测性三元组。结合告警管道和 ML 特定的监控实践，它们能够实现快速事件响应和模型性能的持续保证。本讲座涵盖端到端的日志架构（收集、聚合、存储、查询）、告警策略、事件管理和高级 ML 可观测性模式，为 Module 009 画上句号。

**预计阅读时间：** 80–95 分钟  
**动手配套实验：** Exercise 04 – 日志管道、Exercise 05 – 告警与事件响应  
**前置知识：** Lectures 01–03、熟悉 Prometheus/Grafana、Kubernetes 或容器化部署的基础知识。

---

## 1. AI 基础设施的日志基础

### 1.1 日志的作用
- 捕获离散事件（错误、警告、状态变化）
- 提供合规审计跟踪（GDPR、HIPAA）
- 提供聚合指标缺失的上下文（堆栈跟踪、负载标识符）
- 驱动事后分析（时间线重建、影响评估）

### 1.2 结构化 vs 非结构化日志
- **结构化日志：** JSON 或键/值格式。更易于解析、可字段过滤、机器可读
- **非结构化日志：** 自由格式文本。查询困难、容易出现解析问题
- 标准化结构化日志并带有 schema 版本控制，以最小化下游问题

### 1.3 日志级别和指南
| 级别 | 目的 | AI/ML 示例 |
|-------|---------|----------------|
| DEBUG | 详细诊断 | 开发运行的特征值（非生产） |
| INFO | 正常操作 | "训练任务启动"、"模型版本已部署" |
| WARN | 意外事件，可自行恢复 | 因网络超时重试 |
| ERROR | 用户可见或不可恢复 | 推理请求失败、数据验证错误 |
| FATAL | 系统无法继续 | 数据库连接丢失、模型文件损坏 |

**指南：** 避免记录敏感数据（PII、秘密密钥）；应用脱敏或哈希。采用适合合规的保留策略（例如：INFO 保留 30 天、审计日志 180 天）。

---

## 2. 日志架构模式

### 2.1 日志传输管道
1. **收集：** Agents（Fluent Bit、Vector、Filebeat）或 sidecars 从 stdout/文件读取
2. **传输：** 通过 gRPC/HTTP 发送日志到聚合系统
3. **聚合/索引：** Loki、Elasticsearch/OpenSearch、Splunk
4. **存储：** 本地、对象存储、托管服务
5. **查询和可视化：** Grafana Loki 面板、Kibana Dashboard、SQL 引擎

### 2.2 Kubernetes 日志
- 容器将日志输出到 stdout/stderr
- Fluent Bit/Fluentd DaemonSet 从 `/var/log/containers` 追踪日志
- 应用过滤器以丰富日志元数据（namespace、pod、labels）
- 通过 labels 支持多租户（team、environment）

### 2.3 Loki 栈（PLG：Prometheus + Loki + Grafana）
- 轻量级、对于高容量日志成本效益高
- 日志流由 labels 索引（限制 label 基数）
- 查询语言 LogQL 类似 PromQL
  - 示例：`{app="inference-gateway"} |= "timeout" | json | latency > 300`
- 在对象存储中存储压缩块（S3、GCS）
- 集中使用 Grafana 进行可视化和告警

### 2.4 Elastic Stack（ELK / OpenSearch）
- Beats/Logstash 摄取日志、转换、发送到 Elasticsearch
- Kibana 用于 Dashboard；通过 Elasticsearch 数据源集成到 Grafana
- 搜索能力强大，但资源密集（仔细规划容量）

---

## 3. 日志丰富与上下文传播

### 3.1 通用字段
- `timestamp`、`level`、`message`、`service`、`environment`、`region`
- 关联 ID：`trace_id`、`span_id`、`request_id`
- 模型元数据：`model_name`、`model_version`、`experiment_id`
- 用户元数据（如果需要）：确保隐私（哈希用户 ID）
- 资源信息：`pod_name`、`node`、`gpu_id`

### 3.2 与 Metrics 和 Traces 的关联
- 使用 OpenTelemetry 检测传播 trace ID；包含在日志和 metrics labels 中
- 示例 Python 代码片段：
```python
from opentelemetry.trace import get_current_span

span = get_current_span()
trace_id = span.get_span_context().trace_id
log.info("prediction_failed",
         extra={"trace_id": format(trace_id, '032x'),
                "model_version": model_version,
                "latency_ms": latency})
```
- Grafana exemplars 将指标链接到 traces；日志提供逐步上下文

### 3.3 脱敏与合规
- 在摄取时屏蔽或删除敏感字段（PII、secrets）
- 按日志类型定义数据保留（审计日志保留更长时间）
- 应用访问控制以限制对敏感日志的查询能力
- 需要时提供合规导出（搜索/删除请求）

---

## 4. 日志查询与分析

### 4.1 LogQL 基础（适用于 Loki）
- 精确匹配：`{app="inference-gateway", level="error"}`
- 过滤包含：`|= "timeout"`
- 排除：`!= "healthcheck"`
- 解析 JSON：`| json | latency_ms > 200`
- 聚合：
  ```logql
  sum(count_over_time({app="inference-gateway", level="error"}[5m]))
  ```
  用于错误率趋势。

### 4.2 Elasticsearch 查询示例
- Kibana DSL 或类 SQL 查询：
```json
GET /inference-logs/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "service": "feature-store" }},
        { "range": { "@timestamp": { "gte": "now-15m" }}}
      ],
      "filter": [
        { "term": { "level": "ERROR" }}
      ]
    }
  }
}
```
- 聚合：按模型版本分组、统计错误数。

### 4.3 命令行工具
- `kubectl logs`、`stern`、`kail` 用于快速调查
- `rg`、`awk`、`jq` 用于本地日志解析
- 使用 CLI 进行分流，转向集中式系统进行长期分析

---

## 5. 告警管道设计

### 5.1 告警生命周期
1. **检测：** 指标或日志阈值触发告警
2. **通知：** 告警路由到 on-call（PagerDuty、Slack、email）
3. **确认：** On-call 确认、分流
4. **解决：** 应用缓解措施、服务恢复
5. **分析：** 事后审查、根本原因文档

### 5.2 告警质量标准
- **SLO 对齐：** 源自影响用户的 SLI
- **无重复：** 每个问题单个告警；使用告警关联
- **可操作：** 清晰的说明；不含糊的告警
- **自适应：** 支持不同严重级别（寻呼 vs 工单）
- **经过测试：** 在 staging 中验证、运行合成测试

### 5.3 告警类型
- **基于症状：** 错误率、延迟、SLO burn rate（首选）
- **基于资源：** CPU/内存/GPU 饱和（备用）
- **基于事件：** 部署失败、cron 任务错过
- **基于日志：** 检测模式（例如：重复的认证失败）

### 5.4 告警路由和升级
- 主要 on-call 轮换 → 二级工程负责人 → 事件指挥官
- 使用日程表实现太阳跟随覆盖
- 升级策略确保未确认的告警在定义的时间间隔后升级

### 5.5 Burn Rate 告警回顾
- 多窗口、多 burn 方法减少噪音
- 示例：Burn rate >14 持续 5m AND >4 持续 1h 触发寻呼；确保持续性问题后才告警

---

## 6. AI 服务的事件响应

### 6.1 严重级别
- **SEV0：** 完全中断、重大业务影响（例如：推理全球宕机）
- **SEV1：** 性能下降（延迟 >SLO、部分中断）
- **SEV2：** 轻微影响、尚未客户可见（预测会违规）
- **SEV3：** 非紧急问题、通过工单跟踪

### 6.2 事件角色
- **Event Commander (IC)：** 协调响应、沟通
- **通讯负责人：** 保持利益相关者更新
- **领域专家：** 模型所有者、基础设施工程师
- **记录员：** 记录时间线、关键决策（用于事后分析）

### 6.3 ML 特定的事件注意事项
- 验证数据管道（schema 漂移导致预测错误）
- 检查模型版本；必要时回滚到之前的模型
- 检查特征存储健康；回退到缓存特征或默认值
- 评估因失控训练任务导致的成本超支

### 6.4 事后审查
- 识别促成因素、检测缺口、流程问题
- 突出 ML 特定的教训（例如：需要自动化数据验证）
- 相应更新 runbook、Dashboard、告警

---

## 7. ML 可观测性深入

### 7.1 模型性能监控
- 跟踪实时指标：准确率、精确率、召回率（当标签快速可用时）
- 对于延迟标签，使用代理指标（点击率、转化率）
- 按地理位置、用户群体、设备类型切片指标
- 通过 Grafana 或专门的 ML 可观测性工具进行可视化

### 7.2 数据质量监控
- 指标：缺失值、分布偏移、相关性变化
- 工具：Evidently AI、Great Expectations、TFDV
- 通过自定义 exporters 或推送指标到监控栈与 Prometheus 集成
- 在持续偏差超过阈值时告警

### 7.3 漂移检测
- **Covariate drift：** 特征分布变化（监控 PSI、JS 散度）
- **Concept drift：** 特征与标签之间的关系变化（监控准确率下降、残差、公平性）
- **Label drift：** 输出分布变化
- 对于实时检测，运行小批次统计检验
- 提供比较训练与服务分布的 Dashboard

### 7.4 ML 特定告警
- 新标签丢失（超过 24 小时无 ground truth）
- 漂移指标超过阈值
- 模型公平性指标低于目标（例如：群体召回率 < 整体的 90%）
- 特征管道积压影响训练计划

### 7.5 Champion-Challenger 监控
- 比较基线与金丝雀模型指标
- 使用 Dashboard 显示拆分：请求路由、延迟差异、准确率
- 当金丝雀违反阈值时实现自动回滚

---

## 8. 混合/边缘部署中的日志和监控

### 8.1 边缘注意事项
- 连接有限：本地缓冲日志、连接时转发
- 优先处理关键指标和日志（模型健康、硬件状态）
- 使用轻量级 agents（例如：Fluent Bit、Vector）

### 8.2 混合云
- 确保跨环境的 schema 和 labeling 一致
- 使用 VPN/TLS 隧道实现安全的日志传输
- 集中保留策略以满足法规要求

### 8.3 数据驻留和隐私
- 在区域内存储日志以遵守当地法规
- 在源头脱敏 PII；考虑对用户数据使用差分隐私

---

## 9. 自动化和工具

### 9.1 基础设施即代码
- 通过 Terraform/Helm 管理日志和告警资源
- 示例：`helm install loki grafana/loki-stack`
- 使用 GitOps（Argo CD、Flux）实现声明式部署

### 9.2 持续集成检查
- 使用 `promtool check rules` lint 告警规则
- 通过单元测试验证 Loki 管道配置（Vector/Fluent Bit 测试工具）
- 每次部署后在 staging 环境中运行冒烟测试以验证检测

### 9.3 可观测性 CI/CD 管道
1. 开发者更新检测或 Dashboard
2. CI 运行测试（promtool、linting、单元测试）
3. 可观测性变更经过审查、合并到主分支
4. CD 部署到 staging；冒烟测试；手动 QA
5. 推广到生产、监控回归

---

## 10. 治理、安全与合规

### 10.1 基于角色的访问控制
- 限制谁可以查询敏感日志、管理告警、修改 Dashboard
- 使用 SSO/身份提供商配合组映射
- 实行最小权限原则（仅查看 vs 编辑）

### 10.2 审计日志
- 跟踪告警规则、Dashboard、保留策略的更改
- 维护防篡改的日志，单独存储（一次写入存储）
- 确保记录事件备注以供合规

### 10.3 保留和删除策略
- 与法律要求对齐；为每个日志类别定义保留期限
- 使用生命周期规则自动化删除（S3 生命周期策略、Loki 保留）
- 记录"被遗忘权"请求的流程

### 10.4 安全传输和存储
- 加密传输中的日志（TLS）和静态日志（KMS、密钥管理）
- 分段网络；限制日志聚合集群的出站
- 使用防火墙规则、安全组加固访问

---

## 11. 实践检查清单

### 11.1 日志就绪
- [ ] 跨服务的标准化结构化日志库
- [ ] 日志摄取管道已部署（agents、aggregator、storage）
- [ ] 日志和 traces 中包含关联 ID
- [ ] 日志保留策略已文档化并强制执行
- [ ] 敏感字段的脱敏/屏蔽已实现
- [ ] 为常见场景构建了查询 Dashboard（错误日志、审计跟踪）

### 11.2 告警就绪
- [ ] 基于 SLO 的告警已定义并在 staging 中测试
- [ ] 告警路由和升级策略已配置
- [ ] On-call 排班表已文档化并可访问
- [ ] Runbook 已链接到告警注释中
- [ ] 每月审查告警噪音（事后回顾）
- [ ] 关键端口的合成监控（blackbox）告警已就位

### 11.3 ML 监控就绪
- [ ] 模型性能指标已跟踪和可视化
- [ ] 数据漂移检测已集成到监控栈
- [ ] 关键模型的公平性和伦理指标已定义
- [ ] Champion-challenger Dashboard 可用
- [ ] 事件流程包含 ML 特定的检查清单

---

## 12. 知识检查
1. 从成本、可扩展性和功能集方面比较 Loki 和 Elasticsearch 用于日志聚合
2. 概述实现端到端 tracing 并确保日志携带 trace ID 的步骤
3. 设计一个针对模型准确率因延迟标签降至阈值以下的告警——你会使用什么信号和阈值？
4. 解释如何在生产中检测和缓解 concept drift
5. 描述保持日志符合数据隐私法规所需的治理措施

---

## 13. 附加资源
- CNCF Observability Landscape (https://landscape.cncf.io/)
- Grafana Loki 文档和最佳实践
- OpenTelemetry Collector 配置指南
- "Machine Learning Observability: Best Practices"（Fiddler、Arize、WhyLabs 白皮书）
- Google SRE Workbook – 事件响应章节
- MITRE ATLAS 用于对抗性 ML 威胁（与安全日志相关）

---

## 14. 总结
- Logs 提供了补充 metrics 和 traces 的细粒度上下文，对于诊断 AI 基础设施问题至关重要
- 结构化日志、关联 ID 和集中式聚合能够实现高效查询
- 告警必须与 SLO 对齐并包含 ML 特定的考虑（漂移、公平性）
- AI 系统的事件响应包括数据管道和模型性能的额外检查
- 治理、安全和合规是可持续监控和日志实践的重要组成部分

完成全部四个讲座后，Module 009 现在为 AI 基础设施中的监控和可观测性提供了全面基础。继续进行练习以动手巩固这些概念，并通过汇编知识检查测验来完成模块。
