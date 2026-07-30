# 课程提案：Junior AI Infrastructure Engineer — 2026-06

由 `aicg org research --apply` 生成。上限：

- `max_modules_per_run`：1
- `max_exercises_per_run`：3
- `max_projects_per_run`：0
- `min_evidence_count`：每个项目 3 份岗位信息

## 理由

在分析了约 2025-09 至 2026-05 期间的 32 份 Junior/Associate/Entry-Level/I-grade AI 基础设施、MLOps、ML 平台和 Junior DevOps-AI 平台岗位后，没有任何一个现有 10 模块 + 5 项目的 level-10 课程尚未拥有的需求达到证据阈值，足以证明在 level-10 增加范围的合理性。稳定核心技术栈（Python、Linux、Git、Docker、Kubernetes、AWS/GCP、Prometheus/Grafana、CI/CD、Terraform 基础、Flask/FastAPI、PostgreSQL、MLflow、Airflow/Prefect）覆盖了 >80% 的明确必需需求，且模块 001-010 + 项目 01-05 已全部覆盖。2026 年有实质性变动的两个领域——LLM 服务栈（vLLM/Triton/TRT-LLM，出现在 6/32 份岗位中）和向量数据库 / RAG 基础设施（7/32）——在初级层级始终显示为优先而非必需，并且正确地归属于更高级别（ai-infra-engineer-learning level 20 拥有 LLM 服务；ai-infra-ml-platform-learning level 30 拥有 RAG / 向量数据库）。根据所有权规则以及优先选择零新增而非弱证据支持的指导原则，不提议新增任何模块、练习或项目。JOB_REQUIREMENTS.md 为这些需求提供了指向更高级别持有者的链接。两个部分覆盖的说明（超越结业项目范围的 CI/CD ML 流水线，以及模块 009 中的值班生命周期）有意不提升为模块，因为它们各自只有 <8 份支持性岗位，且分别已被现有项目 README 和现有模块内容自然吸收——添加它们会稀释而非强化 level-10 的范围。

## 已接受

### 模块

_无。_

### 练习

_无。_

### 项目

_无。_

## 已拒绝

_无。_

---

## 审批

**本提案不会自动向课程添加任何内容。**

如果你批准部分/全部已接受的项目，请合并此 PR。合并后，
运行 `aicg org promote-plan --role <role>`（或等待每周
审计）以将筛选后的差异应用于 `curriculum-plan.json` 并生成
骨架。

如果你拒绝所有内容，请关闭 PR。无需进一步操作；
下一次研究运行将重新提议任何仍有岗位市场支持的变更。
