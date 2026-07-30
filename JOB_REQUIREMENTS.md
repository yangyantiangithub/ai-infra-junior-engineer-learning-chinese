# Junior AI Infrastructure Engineer — 岗位需求报告

**角色:** Junior AI Infrastructure Engineer（level 10）
**仓库:** `ai-infra-junior-engineer-learning`
**生成日期:** 2026-07-01（2026-06-01 数据包的更新）
**分析岗位数:** 42（时间窗口：约 2025-09 至 2026-07；其中 41 份为最近 90 天内）
**机器可读源文件:** [`.aicg/job-requirements.json`](.aicg/job-requirements.json)
**课程计划差异（v2）:** [`.aicg/curriculum-plan-delta-v2.json`](.aicg/curriculum-plan-delta-v2.json)

---

## 1. 如何阅读本文档

- 每个需求按类别分组，列出**所属角色**（根据课程层级），并指明在本仓库*或*更高级别仓库中的覆盖位置。
- 所有权遵循最低层级原则：一个需求由真正需要它的最低层级拥有，更高级别的角色只有在需要更深层次、架构或领导力背景时才重新拥有它。
- 频率是分析过的岗位（共 42 份）中明确列出该需求的数量；初级岗位集中在稳定的核心技能（Docker、K8s、Python、AWS/GCP、Prometheus、CI/CD），并将 LLM / GPU / RAG 视为优先而非必需。
- 所有证据岗位和引用均在 [`.aicg/job-requirements.json`](.aicg/job-requirements.json) 的 `postings[]` 中（通过下方 ID 引用）。

---

## 2. 市场概况（最近 90 天）

- **头衔多样。** 几乎没有岗位使用"Junior AI Infrastructure Engineer"这个确切的头衔。该角色以以下名称招聘：初级 MLOps、初级 DevOps（AI 平台）、初级平台运维、初级数据平台、初级 IaC、助理 MLOps、ML 工程师 I、AI 基础设施工程师 I、新毕业 ML 基础设施。
- **经验区间。** 0-3 年是最常见的要求；"Associate"/"I"/"Early Career"头衔明确为入门级。少数（如 NYT MLOps 要求 2+ 年）处于 level 20 的边界。
- **薪资区间。** 真正的入门级 USD 约 $60k-$150k 基本工资；列出薪资的岗位中位数集中在 $85k-$125k。
- **稳定的核心技术栈（>60% 岗位）：** Python、Linux、Git、Docker、Kubernetes、一个云平台（AWS 占主导）、CI/CD、Prometheus/Grafana、Terraform 基础、REST API 框架（Flask / FastAPI）。
- **初级岗位优先而非必需：** MLflow、Airflow/Prefect、Helm 编写、GitOps（Argo CD）、向量数据库、LLM 服务（vLLM、Triton、TRT-LLM）、CUDA/NCCL、Spark/Ray。
- **2026 年趋势信号：** LLM / RAG / 向量数据库出现在 32 份岗位中的 6-7 份——已有一定数量，但仍为优先项，集中在 NYT、RingCentral、Citi、NVIDIA 新毕业等岗位。同一批岗位已明确指出这对初级来说是加分项。

### 2.1 2026 年 7 月更新（自 2026-06-01 以来的变化）

约 30 天的更新轮次新增了 10 份岗位（`p33-lockheed-ai-infra-platform-ops-junior` … `p42-dv01-mlops-cloud-eng`），滚动窗口达到 42 份，其中 41 份日期为 2026-04-01 或之后（最近 90 天）。

- **没有新的稳定核心技能跨越 0.30 的添加阈值。** Python、Docker、Kubernetes、一个云平台、CI/CD、Prometheus/Grafana、Terraform 基础、MLflow 和 Airflow/Prefect/Dagster 仍然是主要的必需技术栈。
- **优先而非必需项在初级层级保持稳定：** LLM 服务 6/42（0.14）、向量数据库 + RAG 7/42（0.17）、GPU/CUDA/NCCL 8/42（0.19）、GitOps 7/42（0.17）。均低于 0.30 的添加阈值，且仍由其更高级别方向持有（见 §3.9）。
- **值得注意的新增：** NVIDIA 的"AI/ML Infra SWE - GPU Clusters（New Grad 2026）"强化了 GPU 集群 / Slurm / PyTorch DDP+FSDP 工作在 level 35（性能工程师）牢固持有。Lockheed 的初级 AI-Infra 平台运维岗位是本样本中第一个在 GitLab CI / Jenkins 之外*明确列出* Argo CD 的入门级岗位——仍仅一份岗位，仍归 level 20 持有。
- **结论。** 市场未发生实质性变化。课程计划差异 v2 有意为空（`additions: []`、`updates: []`、`removals: []`）——参见 [`.aicg/curriculum-plan-delta-v2.json`](.aicg/curriculum-plan-delta-v2.json)。

---

## 3. 按类别划分的需求

### 3.1 语言与本地工具（level 10 持有 ✅）

| ID | 需求 | 持有者 | 覆盖位置 | 证据 |
|----|-------------|-------|----------|----------|
| req-001 | **Python**（生产质量，而非 notebook） | level 10 | [`lessons/mod-001-python-fundamentals/`](lessons/mod-001-python-fundamentals/) | p01, p02, p03, p04, p13, p14, p20, p25 |
| req-002 | **Linux** 管理 + shell | level 10 | [`lessons/mod-002-linux-essentials/`](lessons/mod-002-linux-essentials/) | p01, p14, p16, p21, p28 |
| req-003 | **Git / GitHub / GitLab** 工作流 | level 10 | [`lessons/mod-003-git-version-control/`](lessons/mod-003-git-version-control/) | p02, p04, p14 |
| req-021 | **Bash / shell 脚本** | level 10 | [`lessons/mod-002-linux-essentials/`](lessons/mod-002-linux-essentials/) | p01, p04, p14 |

现有覆盖足够——无需新增。

### 3.2 容器与编排（level 10 持有 ✅）

| ID | 需求 | 持有者 | 覆盖位置 | 证据 |
|----|-------------|-------|----------|----------|
| req-004 | **Docker**（Dockerfile、多阶段、Compose） | level 10 | [`lessons/mod-005-docker-containers/`](lessons/mod-005-docker-containers/) | p01, p02, p03, p04, p17, p32 |
| req-005 | **Kubernetes**（Pod/Deployment/Service/HPA/kubectl） | level 10 | [`lessons/mod-006-kubernetes-intro/`](lessons/mod-006-kubernetes-intro/) | p01, p02, p03, p04, p30, p32 |
| req-022 | **Helm**（消费者层级） | level 10 | [`lessons/mod-006-kubernetes-intro/`](lessons/mod-006-kubernetes-intro/) | p01, p02 |

### 3.3 API、数据和 ML 基础（level 10 持有 ✅）

| ID | 需求 | 持有者 | 覆盖位置 | 证据 |
|----|-------------|-------|----------|----------|
| req-006 | 用于 ML 服务的 **REST API**（Flask / FastAPI） | level 10 | [`lessons/mod-007-apis-web-services/`](lessons/mod-007-apis-web-services/) | p03, p17, p19 |
| req-007 | 用于 ML 元数据的 **SQL 与 PostgreSQL** | level 10 | [`lessons/mod-008-databases-sql/`](lessons/mod-008-databases-sql/) | p02, p13 |
| req-010 | **PyTorch / TensorFlow / Scikit-learn**（加载和运行预训练模型） | level 10 | [`lessons/mod-004-ml-basics/`](lessons/mod-004-ml-basics/) | p05, p20, p21, p23, p25 |

### 3.4 云与运维（level 10 持有 ✅）

| ID | 需求 | 持有者 | 覆盖位置 | 证据 |
|----|-------------|-------|----------|----------|
| req-009 | **AWS / GCP / Azure** 基础 | level 10 | [`lessons/mod-010-cloud-platforms/`](lessons/mod-010-cloud-platforms/) | p01, p02, p03, p04, p12, p14, p17, p32 |
| req-020 | **密钥管理与基本安全** | level 10 | [`projects/project-05-production-ml-capstone/`](projects/project-05-production-ml-capstone/) | p01, p10, p14 |

### 3.5 监控与可观测性（level 10 持有 ✅）

| ID | 需求 | 持有者 | 覆盖位置 | 证据 |
|----|-------------|-------|----------|----------|
| req-008 | **Prometheus 与 Grafana** | level 10 | [`lessons/mod-009-monitoring-basics/`](lessons/mod-009-monitoring-basics/) | p01, p02, p03, p07, p14, p15 |
| req-013 | **ML 模型监控与数据漂移**基础 | level 10 | [`projects/project-04-monitoring-alerting/`](projects/project-04-monitoring-alerting/) | p02, p03, p04, p17 |
| req-016 | **事件响应与值班**基础 | level 10 | [`lessons/mod-009-monitoring-basics/`](lessons/mod-009-monitoring-basics/)（部分——见 §4） | p07, p14, p15 |

### 3.6 自动化、IaC、CI/CD（level 10 持有 ✅，部分）

| ID | 需求 | 持有者 | 覆盖位置 | 证据 |
|----|-------------|-------|----------|----------|
| req-011 | 用于 ML 的 **CI/CD 流水线**（GitHub Actions / Jenkins / GitLab CI） | level 10 | [`projects/project-05-production-ml-capstone/`](projects/project-05-production-ml-capstone/)（部分——见 §4） | p01, p02, p03, p04, p08, p17, p32 |
| req-012 | **Terraform** 基础（IaC） | level 10 | [`projects/project-05-production-ml-capstone/`](projects/project-05-production-ml-capstone/) | p01, p02, p03, p08, p09, p12, p14, p30 |

### 3.7 MLOps 工作流（level 10 持有 ✅）

| ID | 需求 | 持有者 | 覆盖位置 | 证据 |
|----|-------------|-------|----------|----------|
| req-014 | **MLflow** 实验追踪 + 注册表 | level 10 | [`projects/project-03-ml-pipeline-tracking/`](projects/project-03-ml-pipeline-tracking/) | p03, p17, p32 |
| req-015 | **Airflow / Prefect** 数据 + ML 流水线 | level 10 | [`projects/project-03-ml-pipeline-tracking/`](projects/project-03-ml-pipeline-tracking/) | p02, p03, p13, p32 |

### 3.8 软技能（level 10 持有 ✅）

| ID | 需求 | 持有者 | 覆盖位置 | 证据 |
|----|-------------|-------|----------|----------|
| req-026 | 沟通 / 跨职能协作 | level 10 | [`CURRICULUM.md#professional-skills`](CURRICULUM.md) | p04, p05, p07 |

### 3.9 level 10 范围之外——链接到所属角色

这些需求出现在岗位中，但被明确视为初级角色的优先而非必需项，且属于更高级别的方向。初级课程指向它们；当前证据不足以证明需要 level-10 模块。

| ID | 需求 | 所属角色（级别） | 所在位置 |
|----|-------------|--------------------|----------------|
| req-017 | **LLM 服务**（vLLM / TensorRT-LLM / Triton） | `ai-infra-engineer-learning`（20） | [mod-110-llm-infrastructure](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning/blob/main/lessons/mod-110-llm-infrastructure/README.md) |
| req-018 | **向量数据库与 RAG 基础设施** | `ai-infra-ml-platform-learning`（30） | [ai-infra-ml-platform-learning](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning) |
| req-019 | **GPU / CUDA / NCCL** 深度工作 | `ai-infra-performance-learning`（35） | [ai-infra-performance-learning](https://github.com/ai-infra-curriculum/ai-infra-performance-learning) |
| req-023 | **分布式处理**（Spark / Dask / Ray） | `ai-infra-engineer-learning`（20） | [ai-infra-engineer-learning](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning) |
| req-024 | **裸金属 / Slurm / HPC** 调度器 | `ai-infra-performance-learning`（35） | [ai-infra-performance-learning](https://github.com/ai-infra-curriculum/ai-infra-performance-learning) |
| req-025 | **GitOps**（Argo CD / Flux） | `ai-infra-engineer-learning`（20） | [ai-infra-engineer-learning](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning) |

对于每一项，模块 004 / 模块 010 / 项目 02 已经提供了初级所需的认识层面（例如，"GPU 与 CPU 进行推理的对比"、"托管 Kubernetes 服务"、"模型版本管理"），以便他们识别该技术并移交给下一级别。

---

## 4. 覆盖缺口（部分）— 以及建议

两个需求在技术上有覆盖，但深度不及当前市场要求。两者均在现有模块/项目内部解决，而非通过新增模块——当前证据不足以证明新增 level-10 模块的合理性。

### 4.1 用于 ML 的 CI/CD 流水线（req-011）

- **当前情况。** 结业项目（`project-05`）将 GitHub Actions 和 CI/CD 列为交付物；没有单独的模块拥有端到端流水线模式（构建 → 测试 → 镜像 → 部署 → 触发重训练）。
- **证据。** 32 份岗位中有 7 份明确要求用于 ML 部署的 CI/CD（p01、p02、p03、p04、p08、p17、p32）。NYT 的岗位是最明确的说明："开发稳健的 CI/CD 流水线，用于自动化模型训练、验证、部署和重训练。"
- **建议。** 在现有项目 README 中加强说明。未提议变更课程计划差异；现有项目范围已要求该交付物。

### 4.2 事件响应与值班基础（req-016）

- **当前情况。** 模块 009 教授告警规则设计，但未讲解值班生命周期（寻呼、分类、应急预案、事后复盘）。
- **证据。** 3 份岗位明确提到值班轮换是初级职责（p07 Capgemini、p14 Trulioo "24/7 值班轮换"、p15 SBC Innovations）。
- **建议。** 这是一个狭窄的缺口——仅三份岗位——更适合在模块 009 内部通过扩展其现有的告警内容来教授。视为内容层面的加强，而非课程计划差异模块。如需模块级别的提案，请参见 [`.aicg/curriculum-plan-delta.json`](.aicg/curriculum-plan-delta.json)——目前按设计为空。

---

## 5. 本课程已与市场匹配的部分

在 32 份岗位中，level-10 稳定核心技术栈（Python / Linux / Git / Docker / Kubernetes / 一个云平台 / CI/CD / Prometheus / FastAPI / Postgres / MLflow / Airflow）覆盖了 >80% 的明确必需需求。10 模块 + 5 项目的结构与最常被引用的技能一一对应，每个必需的 level-10 需求目前已得到覆盖。

角色层级也按预期工作：每个在 level-10 层面*可能*需要增加的内容（LLM 服务、GPU 工作、向量数据库、分布式 Spark/Ray、GitOps、Slurm）已在角色适当的更高级别中得到覆盖。链接而非重复，是正确的做法。

---

## 6. 范围外项目的外部资源

对于希望在学习 level-10 范围之外进行扩展，然后再晋升到更高级别角色的学习者：

- **LLM 服务** — vLLM 文档（`https://docs.vllm.ai/`）、NVIDIA Triton 文档（`https://docs.nvidia.com/deeplearning/triton-inference-server/`）、TensorRT-LLM 仓库。
- **向量数据库与 RAG** — pgvector 文档、Weaviate / Qdrant / Milvus 快速入门、LangChain "RAG from scratch" 指南。
- **GPU/CUDA** — NVIDIA "加速计算"学习路径；NCP-AII 认证方向。
- **分布式处理** — Ray、Dask 和 Spark 官方快速入门。
- **GitOps** — Argo CD 入门、Flux v2 文档。
- **Slurm / HPC** — SchedMD "十分钟了解 Slurm"、NVIDIA DGX SuperPOD 参考架构。

这些仅供参考，有意不纳入 level-10 课程。

---

## 7. 方法论

- **来源：** Indeed、Built In、ZipRecruiter、Glassdoor、LinkedIn、Greenhouse 托管的招聘板、Workday 托管的招聘板（Workday、BCBS、Astreya、NVIDIA、HPE、CMU、Lockheed）、Breezy（Accrete）、Workable（Quadric）、Remoterocketship、Remotive、speedyapply/2026-AI-College-Jobs 精选新毕业生列表、Hacker News "Who is hiring" 存档。
- **时间窗口：** 2025 年 9 月 → 2026 年 7 月（偏向最近 90 天：41/42 份岗位日期为 2026-04-01 或之后；较旧的异常值仅在列出具体技能时保留）。
- **分析了 42 份岗位**（要求 ≥25 份）；30 份有可检索的完整描述，其余 12 份来自聚合/付费墙列表，其中头衔/雇主/级别可确认，但完整岗位正文并非始终可完全检索（Indeed、Glassdoor、部分 LinkedIn）——这些仍被计入，因为它们确认了角色的*普遍性*和*级别*，即使无法总是提取具体技能。
- **2026 年 7 月更新轮次**在 32 份岗位的 6 月基线基础上新增了 10 份岗位（`p33`–`p42`）。本次更新未导致任何需求、证据模式或所有权分配发生变化；课程计划差异 v2 按照连续性偏差原则有意为空。
- **所有权规则适用：** 一个需求出现在真正需要它的最低角色层面。如果更高级别的仓库已拥有某个需求（例如 level 20 拥有 LLM 服务），则进行**链接**而非重复。

---

**由 VeriSwarm.ai 维护**
