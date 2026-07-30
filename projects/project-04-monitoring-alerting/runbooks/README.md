# 手册

手册是值班工程师在告警触发时遵循的操作指南。目标是让凌晨 3 点的事件与下午 3 点的事件一样容易处理。每个手册涵盖一个告警（或一个紧密相关的系列），并回答以下问题：

1. **这个告警是什么意思？** 用通俗语言描述出了什么问题。
2. **有多严重？** 谁受影响，什么功能受损。
3. **首先检查什么。** 用于确认诊断的具体仪表盘和查询。
4. **可能的原因。** 按概率排序。
5. **缓解步骤。** 具体的命令和链接。
6. **回滚。** 如果缓解措施使情况恶化。
7. **事后审查触发条件。** 何时此事件需要书面事后审查。

## 约定

- 每个手册以 `prometheus/alerts.yml` 中的告警名称命名。
- 告警的 `runbook_url` 注释应指向渲染后的手册 URL。
- 缓解命令假设 kubectl 上下文已设置；显示完整命令包括命名空间。
- "可能的原因"应反映生产历史中实际发生过的问题，而非理论可能性。

## 索引

| 告警 | 严重级别 | 手册 |
|---|---|---|
| `HighErrorRate` | critical | [high-error-rate.md](high-error-rate.md) |
| `SlowResponse` | warning | [slow-response.md](slow-response.md) |
| `TargetDown` | critical | [target-down.md](target-down.md) |
| `ModelDriftDetected` | warning | [model-drift.md](model-drift.md) |
| `LogIngestionLag` | warning | [log-ingestion-lag.md](log-ingestion-lag.md) |
