# Module 009 测验：监控与可观测性

## 测验信息

- **模块：** 监控与可观测性基础
- **题目总数：** 50
- **时间限制：** 90 分钟
- **通过分数：** 80%（50 题中答对 40 题）
- **涵盖主题：**
  - 可观测性基础、SLO/SLA/error budgets
  - Prometheus 架构、exporters、PromQL、告警
  - Grafana Dashboard、模板化、provisioning、告警
  - 日志管道（Loki/Elasticsearch）、结构化日志、关联
  - 事件响应、ML 特定监控（漂移、公平性、GPU 使用）

## 说明

1. 回答所有问题；除非明确说明，否则只有一个答案正确
2. 部分场景题需要选择最佳操作方案
3. 每个答案后附有解释以强化核心概念
4. 教师用答案密钥位于解决方案仓库（`ai-infra-junior-engineer-solutions`）

---

### 1. 哪项陈述最能区分监控与可观测性？
A) 监控关注日志；可观测性关注指标  
B) 监控回答系统是否健康；可观测性帮助解释*为什么*系统表现出某种行为  
C) 监控是被动的；可观测性是主动的；两者使用相同数据  
D) 监控仅用于基础设施；可观测性仅用于应用程序

**正确答案：** B  
**解释：** 监控检查预定义的指标，而可观测性通过 metrics、logs 和 traces 等信号理解内部状态。

---

### 2. 如果 SLO 是 30 天内 99.5% 的可用性，error budget 是多少？
A) 允许 0.5% 的停机时间（约 3.6 小时）  
B) 允许 0.5% 的正常运行时间（约 3.6 小时）  
C) 允许 99.5% 的正常运行时间（约 28.5 天）  
D) 允许 0.05% 的停机时间（约 22 分钟）

**正确答案：** A  
**解释：** Error budget 是 SLO 的补集。对于 99.5%，0.5% 的停机时间 ≈ 每 30 天窗口约 3.6 小时。

---

### 3. 哪个 Prometheus 组件负责存储指标和评估规则？
A) Alertmanager  
B) Pushgateway  
C) Prometheus server  
D) Exporter

**正确答案：** C  
**解释：** Prometheus server 抓取目标、存储时间序列，并在将告警转发到 Alertmanager 之前评估 recording/alerting 规则。

---

### 4. 何时应使用 Prometheus Pushgateway？
A) 用于防火墙后面的长期运行服务  
B) 用于无法直接抓取的短期批处理任务  
C) 用于向 Jaeger 导出 traces  
D) 用于水平扩展 Prometheus

**正确答案：** B  
**解释：** Pushgateway 缓冲来自临时任务的指标；不适用于长期运行的服务或扩展。

---

### 5. 哪个 PromQL 函数返回单调递增 counter 在范围内的每秒速率？
A) `increase()`  
B) `rate()`  
C) `avg_over_time()`  
D) `histogram_quantile()`

**正确答案：** B  
**解释：** `rate()` 计算指定 range vector 内 counters 的每秒平均速率；`increase()` 返回总增量，而非每秒速率。

---

### 6. Prometheus 中高基数指标的主要风险是什么？
A) 告警变得过于敏感  
B) Dashboard 无法显示它们  
C) TSDB 内存使用和查询性能急剧下降  
D) Counter 重置变得无法检测

**正确答案：** C  
**解释：** 每个唯一的 label 集都会增加序列数；高基数会膨胀内存、存储和查询成本。

---

### 7. 在 Grafana 中，哪个功能允许同一个 Dashboard 按环境、集群或团队过滤而无需复制面板？
A) Transformations  
B) Canvas panels  
C) 模板化变量  
D) Dashboard 权限

**正确答案：** C  
**解释：** 变量（模板化）支持参数化查询，使 Dashboard 适应不同上下文。

---

### 8. 将 Grafana Dashboard 作为代码进行 provision 有什么好处？
A) 无需重启即可进行实时编辑  
B) 实现基于 Git 的版本控制和跨环境的一致部署  
C) 自动生成告警规则  
D) 将 PromQL 转换为 SQL

**正确答案：** B  
**解释：** Provisioning 确保 Dashboard 可重现、通过 Git 审查，并通过 CI/CD 或 GitOps 工作流部署。

---

### 9. 在 Loki 管道中，哪个阶段解析 JSON 字段以便它们可以用作 labels 或 filters？
A) `docker {}`  
B) `json {}`  
C) `match {}`  
D) `drop {}`

**正确答案：** B  
**解释：** `json` 阶段从结构化日志中提取字段，支持按 `level` 或 `trace_id` 等键进行过滤。

---

### 10. 哪个指标最能捕捉推理延迟的 SLO 合规性？
A) `avg_over_time(inference_latency_seconds[5m])`  
B) `histogram_quantile(0.99, sum(rate(inference_latency_seconds_bucket[5m])) by (le))`  
C) `max(inference_latency_seconds)`  
D) `sum(inference_latency_seconds_count)`

**正确答案：** B  
**解释：** 使用带有 `histogram_quantile` 的 histogram buckets 计算延迟百分位数（例如：P99），直接映射到延迟 SLO。

---

### 11. 多窗口、多 burn rate 告警的目的是什么？
A) 在生成告警前验证 Kubernetes 自动扩缩  
B) 结合快速和慢速检测窗口，使得仅当持续 SLO 违规时才触发告警  
C) 同时告警延迟和错误率  
D) 减少 Alertmanager 接收者数量

**正确答案：** B  
**解释：** 使用不同的窗口在响应性和降噪之间取得平衡——快速窗口捕获严重峰值，长窗口确认持续性。

---

### 12. 哪个 Grafana 面板类型最适合可视化跨 GPU ID 的延迟分布？
A) Stat panel  
B) Gauge panel  
C) Heatmap  
D) Bar gauge

**正确答案：** C  
**解释：** Heatmaps 能有效显示两个维度的变化（例如：GPU ID vs 时间），强度代表利用率或延迟。

---

### 13. 在通过数据库迁移删除 Dashboard 依赖的列之前，应该做什么？
A) 立即删除列，稍后修复 Dashboard  
B) 扩展 Prometheus 以避免停机  
C) 审计 Dashboard 查询、更新或提供回退视图，并通知利益相关者  
D) 将列转换为 JSONB

**正确答案：** C  
**解释：** 可观测性 artifacts 通常依赖 schema；审计并分阶段更改以防止 Dashboard/告警损坏。

---

### 14. 在使用 OpenTelemetry 时，哪个工具可以用 trace ID 丰富 FastAPI 日志？
A) 使用手动 span 上下文提取的 `structlog`  
B) Promtail 管道阶段  
C) Grafana Tempo  
D) Prometheus relabeling

**正确答案：** A  
**解释：** Trace 上下文必须在服务中捕获；例如：`structlog` 可以通过 `get_current_span()` 提取 ID 并将其添加到日志记录中。

---

### 15. 结构化 JSON 日志相对于纯文本的主要优势是什么？
A) 减少磁盘使用  
B) 增加日志冗长度  
C) 在日志分析工具中实现可靠的解析、过滤和关联  
D) 消除日志聚合的需求

**正确答案：** C  
**解释：** 结构化日志允许下游系统一致地解析字段，实现跨日志、指标和 traces 的过滤和关联。

---

### 16. 哪个 Prometheus 查询检测到目标完全停止暴露指标？
A) `up == 1`  
B) `increase(up[5m])`  
C) `absent(up{job="inference-gateway"})`  
D) `rate(up[5m]) > 0`

**正确答案：** C  
**解释：** `absent()` 在没有任何匹配序列时返回一个向量；用于告警丢失抓取。

---

### 17. Alertmanager 在 Prometheus 生态系统中的主要角色是什么？
A) 运行 PromQL 查询  
B) 存储时间序列数据  
C) 去重、静默和路由告警到通知  
D) 渲染 Grafana Dashboard

**正确答案：** C  
**解释：** Alertmanager 管理告警投递（Slack、PagerDuty、email）并处理分组/静默。

---

### 18. 在事件期间，on-call Dashboard 顶部应出现哪些面板？
A) 实验结果和功能标志  
B) SLO 合规、错误率、请求量和高层资源饱和  
C) 成本分析和营销 KPI  
D) 历史发布说明

**正确答案：** B  
**解释：** On-call Dashboard 必须立即展示可操作的 SLI/SLO 指标和资源健康状态，以便快速分流。

---

### 19. 哪个描述最符合 burn rate 告警？
A) CPU 使用率 >90% 持续 5 分钟时告警  
B) 当错误与预算错误的比例在窗口内超过阈值时告警  
C) 当 Prometheus 无法到达 Alertmanager 时告警  
D) 当 Grafana Dashboard 被编辑时告警

**正确答案：** B  
**解释：** Burn rate 比较实际 error budget 消耗与允许速率，实现 SLO 执行。

---

### 20. 为什么日志应避免使用 `user_id` 等高基数 labels？
A) 它们使 Dashboard 不可读  
B) 它们使告警不可能  
C) 它们引入隐私/合规风险并造成存储/查询爆炸  
D) 它们阻止 tracing 集成

**正确答案：** C  
**解释：** 日志中的高基数字段会膨胀存储并可能暴露敏感数据；优选哈希或聚合标识符。

---

### 21. 哪个指标指示推理工作负载的 GPU 过热风险？
A) `ai_infra_inference_requests_total`  
B) `node_disk_io_time_seconds_total`  
C) `DCGM_FI_DEV_GPU_TEMP`  
D) `container_memory_rss`

**正确答案：** C  
**解释：** DCGM exporter 暴露 GPU 指标；温度可以启用主动节流或扩缩容决策。

---

### 22. 存储 Grafana 告警规则定义的最佳实践是什么？
A) 仅手动输入  
B) 嵌入 Prometheus 配置中  
C) 通过代码或 API 导出管理，存储在 Git 中，与 Dashboard 对齐  
D) 存储在 Excel 电子表格中

**正确答案：** C  
**解释：** 将告警规则视为代码——版本化、审查，并与 Dashboard 一同部署，确保一致性和可审计性。

---

### 23. 哪个 LogQL 查询统计过去 10 分钟内推理服务的 error 级别日志？
A) `{service="inference-gateway"} |= "error"`  
B) `count_over_time({service="inference-gateway", level="error"}[10m])`  
C) `sum(rate({service="inference-gateway"}[10m]))`  
D) `increase({level="error"}[10m])`

**正确答案：** B  
**解释：** `count_over_time` 返回范围内的出现次数；如果需要分组，结合 `sum` 使用。

---

### 24. 哪个步骤最能帮助 on-call 工程师在重大发布前做好准备？
A) 禁用所有告警以减少噪音  
B) 审查 runbook、确认 Dashboard/告警引用即将发布的版本，并安排维护窗口  
C) 缩减 Prometheus 以降低成本  
D) 仅监控模型，忽略基础设施

**正确答案：** B  
**解释：** 发布前准备包括对齐 runbook、验证可观测性资产，以及必要时规划静默。

---

### 25. 在 ML 可观测性中，哪个指标检测 covariate drift？
A) Error budget burn rate  
B) 比较特征分布的 Population Stability Index (PSI)  
C) 请求延迟  
D) CPU 饱和

**正确答案：** B  
**解释：** PSI 和类似指标跟踪输入特征中的分布偏移，表明 covariate drift。

---

### 26. 哪种方法确保日志数据符合隐私法规？
A) 永远保留所有原始日志  
B) 无论策略如何，每天删除日志  
C) 在摄取时应用脱敏/屏蔽、按日志类别定义保留、强制访问控制和删除工作流  
D) 将日志存储在公共对象存储中以保持透明

**正确答案：** C  
**解释：** 合规要求有意的保留、脱敏和访问管理；策略必须以编程方式强制执行。

---

### 27. 为什么将告警注释与 runbook 链接配对很重要？
A) 它们自动解决告警  
B) 它们提供即时的上下文和下一步操作，减少平均缓解时间  
C) 它们自动更改告警严重级别  
D) 它们禁用重复告警

**正确答案：** B  
**解释：** 带有 runbook 链接的注释使响应者能够快速按照相关说明行动。

---

### 28. 在 CI/CD 中验证可观测性更改的最有效方法是什么？
A) 直接部署到生产；如果出现问题则回滚  
B) 运行 promtool/linting 检查、部署到 staging、执行冒烟测试，并在生产 rollout 前监控回归  
C) 每周手动检查 Dashboard  
D) 为加速禁用 CI 检查

**正确答案：** B  
**解释：** 将可观测性资产视为代码——在推广到生产之前进行 lint/测试、staging 和观察，以防止事件。

---

### 29. 哪个 KPI 最能反映 on-call 轮换的健康状况？
A) 创建的 Dashboard 数量  
B) 平均确认时间（MTTA）和每班次的告警量  
C) 使用的日志存储量  
D) Prometheus 规则数量

**正确答案：** B  
**解释：** MTTA 和告警负载表明响应者疲劳和有效性，指导告警和人员配置的调整。

---

### 30. 在事后审查中，重点应该是什么？
A) 将责任归咎于个人  
B) 确认停机处罚已收取  
C) 识别系统性改进、记录时间线、根本原因以及带有所有者的行动项  
D) 减少日志以防止未来事件

**正确答案：** C  
**解释：** 无指责的事后分析强调学习和系统性修复，产生可操作的任务以防止复发。

---

### 31. 在 SLO 监控中，多窗口 burn rate 告警的目的是什么？
A) 通过要求多次连续违规来减少告警噪音
B) 检测快速（短窗口）和慢速（长窗口）的 error budget 消耗
C) 向多位 on-call 工程师发送告警
D) 基于 SLO 违规自动扩展基础设施

**正确答案：** B
**解释：** 多窗口告警（例如：5m/1h 窗口）捕获快速 SLO 违规（短窗口）和逐渐退化（长窗口），提供全面覆盖。

---

### 32. 给定 99.9% 可用性 SLO，哪个 burn rate 乘数指示在 <6 小时内耗尽 error budget？
A) 1x
B) 6x
C) 14.4x
D) 100x

**正确答案：** C
**解释：** 14.4x 的 burn rate 意味着你消耗 error budget 的速度是可接受速度的 14.4 倍，在 <6 小时（持续速率）内耗尽 30 天的预算。

---

### 33. 哪个 Alertmanager 功能可在多个相关告警触发时防止重复通知？
A) Grouping
B) Inhibition rules
C) Silences
D) Time intervals

**正确答案：** B
**解释：** Inhibition rules 基于其他活跃告警抑制告警（例如：如果集群宕机则抑制节点告警）。

---

### 34. 生产 runbook 的推荐结构是什么？
A) 仅列出告警名称和联系人
B) 影响、症状、分流步骤（<5 分钟）、调查查询、缓解过程、升级路径、事后任务
C) 仅包括 Prometheus 查询
D) 将所有细节推迟到口头沟通

**正确答案：** B
**解释：** 全面的 runbook 通过提供完整的故障排除工作流和升级流程来实现快速响应。

---

### 35. 在事件期间，哪个指标衡量从问题发生到检测的时间？
A) MTTR（平均修复时间）
B) MTTD（平均检测时间）
C) MTTA（平均确认时间）
D) MTBF（平均故障间隔时间）

**正确答案：** B
**解释：** MTTD 是问题开始与监控/告警检测到它之间的时间。

---

### 36. 哪个 PromQL 查询从 histogram buckets 计算 P95 延迟？
A) `rate(http_request_duration_seconds_sum[5m])`
B) `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))`
C) `avg(http_request_duration_seconds)`
D) `max(http_request_duration_seconds)`

**正确答案：** B
**解释：** `histogram_quantile()` 从 histogram buckets 计算分位数；0.95 = P95，rate() 提供每秒速率。

---

### 37. 指示显著特征漂移的 Population Stability Index (PSI) 阈值是多少？
A) PSI < 0.1
B) PSI 0.1-0.2
C) PSI > 0.2
D) PSI = 1.0

**正确答案：** C
**解释：** PSI > 0.2 表示显著分布偏移；PSI > 0.25 通常触发模型重新训练。

---

### 38. 在无指责的事后分析中，主要重点应放在什么？
A) 识别引起问题的工程师
B) 记录时间线、根本原因分析（系统性）和预防措施项
C) 计算经济处罚
D) 减少监控以防止未来告警

**正确答案：** B
**解释：** 无指责的事后分析侧重于系统性改进，而非个人指责，以促进学习和防止复发。

---

### 39. 哪种日志脱敏技术对 PII 合规最有效？
A) 删除所有日志
B) 在摄取时应用正则表达式模式以屏蔽/脱敏敏感数据（SSN、信用卡、email）
C) 以纯文本存储日志以保持透明
D) 仅记录错误，不记录 info 级别日志

**正确答案：** B
**解释：** 在摄取时自动脱敏（使用 Promtail 管道阶段或类似方法）可防止 PII 进入存储。

---

### 40. 事件模拟和 runbook 实践的推荐方法是什么？
A) 等待真实事件来练习
B) 创建自动化模拟脚本，注入故障、触发告警并生成事件时间线
C) 仅在工作时间于生产环境中测试
D) 避免测试以防止误报

**正确答案：** B
**解释：** 定期的事件模拟（混沌工程、游戏日）验证 runbook 并建立肌肉记忆，而不会影响用户。

---

### 41. 何时应将事件升级到下一级？
A) 所有事件立即升级
B) 30 分钟后如果未解决、影响在增加或根本原因不明确
C) 绝不升级以避免看起来不称职
D) 仅在工作时间

**正确答案：** B
**解释：** 升级标准应基于时间、影响或复杂性，以确保适当的专业知识参与。

---

### 42. 哪个 Grafana 功能允许从 Prometheus label 值填充 Dashboard 变量？
A) Static variables
B) 带有 label_values() 的 Query variables
C) Constant variables
D) Text box variables

**正确答案：** B
**解释：** 使用 `label_values(metric, label)` 的 Query variables 基于 Prometheus 数据动态填充下拉过滤器。

---

### 43. 分布式 tracing 中关联 ID 的好处是什么？
A) 它们降低日志存储成本
B) 它们实现跨多个服务的端到端请求跟踪，并将 traces 链接到日志
C) 它们自动修复性能问题
D) 它们禁用不必要的指标

**正确答案：** B
**解释：** 关联 ID（trace ID）将跨微服务的单个请求的日志和 traces 链接起来，实现全面调试。

---

### 44. 哪个指标最能指示推理服务中的 GPU 饱和？
A) CPU 使用率
B) GPU 利用率在 95-100% 且队列深度增加
C) 网络带宽
D) 磁盘 I/O

**正确答案：** B
**解释：** 高 GPU 利用率加上不断增长的队列深度表明饱和；需要额外的 GPU 容量。

---

### 45. 响应严重 SLO burn rate 告警时，正确的操作顺序是什么？
A) 回滚 → 调查 → 通知利益相关者 → 文档化
B) 验证告警 → 评估范围 → 调查根本原因 → 缓解 → 文档化 → 事后审查
C) 禁用告警 → 稍后修复 → 重新启用告警
D) 等待其自行解决

**正确答案：** B
**解释：** 结构化的事件响应遵循：验证（避免误报）、评估（范围/影响）、调查、缓解、文档化和学习。

---

### 46. 哪个 LogQL 查询检测日志摄取中断？
A) `{job="logs"} |= "error"`
B) `absent_over_time({job="docker-logs"}[10m])`
C) `rate({job="logs"}[5m])`
D) `count({job="logs"})`

**正确答案：** B
**解释：** `absent_over_time()` 在时间范围内不存在日志时返回 1，表明摄取失败。

---

### 47. Alertmanager 的 `group_wait` 参数的目的是什么？
A) 在发送一组告警的首次通知之前等待多长时间
B) 重新发送通知之间等待多长时间
C) 在内存中保留告警多长时间
D) 等待用户确认多长时间

**正确答案：** A
**解释：** `group_wait` 短暂缓冲告警以将相关告警分组到单个通知中，减少噪音。

---

### 48. 在监控模型公平性时，哪个指标检测人口均等违规？
A) 延迟 P95
B) 不同人口群体之间预测率的绝对差异超过阈值（例如：> 0.1）
C) Error budget 消耗
D) GPU 温度

**正确答案：** B
**解释：** 人口均等比较跨群体的正面预测率；显著差异（> 10%）表明潜在偏差。

---

### 49. 哪种做法可减少 on-call 轮换中的告警疲劳？
A) 将所有告警发送到寻呼器以最大程度地提高意识
B) 实施基于 SLO 的告警、每季度调整阈值、自动化常见缓解措施并跟踪每人每周的告警数
C) 永久禁用所有非严重告警
D) 仅在工作时间告警

**正确答案：** B
**解释：** 通过关注用户影响（SLO）、定期调整阈值、自动化和监控工作负载指标来减少告警疲劳。

---

### 50. 事件时间线和事后分析的建议保留期限是多长？
A) 解决后删除
B) 保留 30 天
C) 无限期归档以用于组织学习和合规
D) 仅当事件严重时才保留

**正确答案：** C
**解释：** 事件文档为未来问题提供历史背景、支持合规并实现长期可靠性改进。

---

## 测验完成！

**教师提醒：** 详细的答案密钥和解决方案指南位于相应的解决方案仓库（`ai-infra-junior-engineer-solutions/quizzes/module-009/quiz-01-monitoring-observability-answer-key.md`）。修改测验内容时，请同时更新学习和解决方案仓库。
