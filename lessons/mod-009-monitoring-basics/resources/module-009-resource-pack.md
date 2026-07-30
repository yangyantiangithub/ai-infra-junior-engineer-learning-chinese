# Module 009 资源包（监控与日志）

本清单记录了 Module 009 附带的补充资源。在实施过程中，将引用的文件附加到此目录（或链接到其位置）。使用状态列跟踪完成情况并突出显示未完成的 TODO。

| 资源 | 描述 | 建议位置 | 状态 / 备注 |
|-------|-------------|--------------------|----------------|
| `dashboards/inference-gateway.json` | 推理网关的 Grafana 服务级 Dashboard（Exercise 03） | `/home/claude/ai-infrastructure-project/repositories/learning/ai-infra-junior-engineer-learning/assets/dashboards/` | TODO – 在仓库构建阶段实现 Dashboard 后导出 |
| `dashboards/platform-health.json` | Grafana 平台健康 Dashboard（Exercise 03） | 同上 | TODO – 等待 Dashboard 实现 |
| `dashboards/executive-overview.json` | 管理层 KPI Dashboard | 同上 | TODO – 将与利益相关者 Dashboard 一同创建 |
| `alerting/inference-rules.json` | Grafana 告警定义或 Prometheus 告警规则包 | `assets/alerting/` | TODO – 从 Grafana/Prometheus 配置生成 |
| `prometheus/docker-compose.monitoring.yml` | Exercise 02 的参考栈配置 | `assets/prometheus/` | TODO – 测试后捕获规范 compose 文件 |
| `prometheus/rules/*.yml` | 练习中使用的 recording + alerting 规则 | `assets/prometheus/rules/` | TODO – 编写后从实验室输出复制 |
| `promtail/promtail-config.yaml` | 日志传输配置（Exercise 04） | `assets/logging/` | TODO – 日志管道验证后添加 |
| `loki/local-config.yaml` | Loki 部署配置 | `assets/logging/` | TODO – 日志管道验证后添加 |
| `docs/prometheus-architecture.png` | 架构图（Exercise 02） | `docs/diagrams/` | TODO – 待生成图表（使用 draw.io 模板） |
| `docs/logging-architecture.png` | 日志管道图（Exercise 04） | `docs/diagrams/` | TODO – 待生成图表 |
| `docs/alerting-policy.md` | 告警路由矩阵（Exercise 05） | `docs/` | TODO – 在告警配置期间编写 |
| `runbooks/inference-latency.md` | 事件响应 Runbook（Exercise 05） | `runbooks/` | TODO – 按 runbook 模板填充 |
| `incidents/INC-001-latency.md` | 示例事件时间线 | `incidents/` | TODO – 在事件模拟期间创建 |
| `validations/module-009-checklist.md` | 内容验证后的 QA 检查清单 | `validations/` | TODO – 由 QA/内容验证 agent 生成 |

## 使用说明
- 将此文件视为填充资源时的权威索引。更新状态列（`TODO`、`IN PROGRESS`、`COMPLETE`），以便 QA 可以快速审核进度。
- 当资源位于此仓库之外时，添加超链接或相对路径，以便未来的贡献者能够找到它们。
- 在结束 Module 009 的 Phase 8 之前，确保上述所有 TODO 要么已完成，要么附有理由/延迟说明供内容验证 agent 参考。
