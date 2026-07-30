# 研究提案 — Junior AI Infrastructure Engineer — 2026-07

格式：**curriculum-plan v2**（每个角色的清单位于内容生成仓库中的 `manifest/curriculum_plan.junior-engineer.manifest.json`）。

## 摘要

- 基线需求数量：**15**
- 新增：**0**
- 更新：**0**
- 移除：**0**

## 连续性检查

验证器未标记此差异为需要明确批准。

验证器注释：

_（无）_

## 理由

本周期市场未发生变化。约 30 天的更新轮次（2026-06-01 → 2026-07-01）在 6 月基线（32 份）基础上新增了 10 份岗位，滚动窗口达到最近 90 天内观察到的 42 份岗位（2026-04-01 → 2026-07-01）。稳定的 level-10 核心技术栈（Python、Linux、Git、Docker、Kubernetes、一个云平台、CI/CD、Prometheus/Grafana、Flask/FastAPI、PostgreSQL、Terraform 基础、MLflow、Airflow/Prefect）仍然在 >60% 的岗位必需技能列表中占主导地位。6 月数据包标记为初级层级加速但优先的四个项目（通过 vLLM/Triton/TRT-LLM 的 LLM 服务、向量数据库 + RAG、GPU/CUDA/NCCL 深度工作、通过 Argo CD/Flux 的 GitOps）在 7 月样本中在初级层级仍为优先而非必需：LLM 服务 6/42（0.14）、向量数据库 + RAG 7/42（0.17）、GPU/CUDA 8/42（0.19）、GitOps 7/42（0.17）——均远低于 0.30 的新增阈值。没有需求变得过时。根据连续性偏差和所有权规则，正确的输出是空的新增/更新/移除——每个更高级别的项目已链接到其所属角色（level 20 / 30 / 35），且每个 level-10 需求已由现有模块或项目覆盖。

## 如何应用（审批后）

```sh
aicg org plan-delta-apply \
  --role junior-engineer \
  --delta .aicg/curriculum-plan-delta-v2.json
```

CLI 会重新验证差异与当前基线。如果验证器的标记或拒绝自本提案编写以来发生了变化（例如，新的岗位进入了另一个角色），应用将在写入前显示这些问题。
