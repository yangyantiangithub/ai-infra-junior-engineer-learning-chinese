# 项目 05：生产级 ML 系统（结业项目）

**时长：** 120 小时（全日制 3 周，非全日制 5-6 周）
**难度：** 中级+
**前置条件：** 项目 1-4（必须完成所有前置项目）

---

## 概述

欢迎来到你的结业项目！这是将所有知识融会贯通的关键环节。你将把前四个项目整合成一个全面、生产级的 ML 系统，展示你对 AI 基础设施工程的掌握程度。

### 目前已构建的内容

- **项目 1**：使用 Flask/FastAPI 的模型服务 API
- **项目 2**：Kubernetes 部署与编排
- **项目 3**：基于 MLflow 和 Airflow 的 ML 训练流水线
- **项目 4**：使用 Prometheus、Grafana 和 ELK 的完整可观测性

### 现在将要构建的内容

一个**完全集成的生产级 ML 系统**，具备以下特性：

- **统一架构**：所有先前组件无缝协同工作
- **CI/CD 流水线**：自动化测试与部署
- **生产安全**：认证、TLS、密钥管理、RBAC
- **高可用性**：自动扩缩容、零停机更新、灾难恢复
- **自动化 ML 生命周期**：从数据摄入到模型部署
- **完整可观测性**：全栈指标、日志、告警

这个项目将成为你作品集的核心亮点，证明你已经准备好胜任专业的 ML 基础设施岗位。

---

## 学习目标

完成此结业项目后，你将能够：

1. ✅ **将复杂的分布式系统集成**为统一的架构
2. ✅ **为 ML 基础设施实现 CI/CD 流水线**
3. ✅ **部署具备高可用和安全性的生产级系统**
4. ✅ **自动化 ML 生命周期**（从训练到部署）
5. ✅ **为生产系统创建全面的文档**
6. ✅ **构建达到作品集水准的演示**
7. ✅ **掌握生产最佳实践**（12-Factor 应用、GitOps、SRE 原则）
8. ✅ **培养运维技能**（事件响应、灾难恢复）

---

## 系统架构

### 高层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Production ML System                         │
└─────────────────────────────────────────────────────────────────┘

External Users/Applications
         │
         │ HTTPS (TLS)
         ▼
    ┌────────────────┐
    │ NGINX Ingress  │ ← cert-manager (Let's Encrypt)
    │ - Rate Limiting│
    │ - Auth (API Keys)
    └───────┬────────┘
            │
            ▼
┌───────────────────────────────────────────────────────────┐
│              Kubernetes Cluster (GKE/EKS/AKS)            │
│                                                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │    ML API Deployment (Project 1)                │    │
│  │  ┌────────┐ ┌────────┐ ┌────────┐              │    │
│  │  │ Pod 1  │ │ Pod 2  │ │ Pod N  │              │    │
│  │  └────────┘ └────────┘ └────────┘              │    │
│  │  HPA: 3-20 replicas based on CPU/Memory         │    │
│  └─────────────────────────────────────────────────┘    │
│                                                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │    ML Training Pipeline (Project 3)             │    │
│  │  ┌────────────────────────────────────────┐     │    │
│  │  │ Airflow DAG: ml_training_pipeline      │     │    │
│  │  │  [Ingest] → [Validate] → [Train]      │     │    │
│  │  │     ↓                                   │     │    │
│  │  │  [Evaluate] → [Register] → [Deploy]   │     │    │
│  │  └────────────────────────────────────────┘     │    │
│  └─────────────────────────────────────────────────┘    │
│                                                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │    Monitoring Stack (Project 4)                 │    │
│  │  Prometheus | Grafana | ELK | Alertmanager      │    │
│  └─────────────────────────────────────────────────┘    │
│                                                           │
│  ┌─────────────────────────────────────────────────┐    │
│  │    Data Layer                                   │    │
│  │  PostgreSQL | MinIO/S3 | DVC | MLflow Registry  │    │
│  └─────────────────────────────────────────────────┘    │
└───────────────────────────────────────────────────────────┘
```

### CI/CD 流水线流程

```
Developer Push → GitHub → CI Pipeline → CD Pipeline → Production

CI Steps:
1. Code Quality (linting, formatting, type checking)
2. Security Scanning (bandit, Trivy)
3. Testing (unit, integration, API tests)
4. Docker Build & Push
5. Image Vulnerability Scanning

CD Steps (Staging - Automatic):
1. Deploy to Staging with Helm
2. Run Smoke Tests
3. Run Integration Tests
4. Run Load Tests

CD Steps (Production - Manual Approval):
1. Manual Approval Gate
2. Canary Deployment (10% traffic)
3. Monitor Metrics for 10 minutes
4. Promote to Full Deployment or Rollback
5. Run Production Smoke Tests
6. Send Notifications
```

---

## Project Structure

```
project-05-production-ml-capstone/
├── README.md                        # This file
├── requirements.md                  # Detailed requirements
├── architecture.md                  # Architecture documentation
├── src/
│   └── main.py                      # Integrated application (STUB)
├── cicd/
│   └── .github/
│       └── workflows/
│           ├── ci.yml               # CI pipeline (STUB)
│           └── cd.yml               # CD pipeline (STUB)
├── kubernetes/
│   ├── base/
│   │   └── kustomization.yaml       # Kustomize base config
│   └── overlays/
│       ├── dev/
│       │   └── kustomization.yaml   # Dev environment overlay
│       ├── staging/
│       │   └── kustomization.yaml   # Staging environment overlay
│       └── production/
│           └── kustomization.yaml   # Production environment overlay
├── security/
│   ├── cert-manager.yaml            # TLS certificate configuration (STUB)
│   └── vault-config.yaml            # HashiCorp Vault configuration (STUB)
├── monitoring/
│   └── slos.yaml                    # Service Level Objectives
├── tests/
│   └── integration/
│       └── test_e2e.py              # End-to-end tests (STUB)
├── docs/
│   ├── DEPLOYMENT.md                # Deployment guide
│   └── DISASTER_RECOVERY.md         # Disaster recovery plan
└── .env.example                     # Multi-environment configuration
```

---

## 快速开始

### 前置条件

在开始本项目之前，请确保已完成：

- ✅ **项目 1**：模型服务 API
- ✅ **项目 2**：Kubernetes 部署
- ✅ **项目 3**：ML 训练流水线
- ✅ **项目 4**：监控与可观测性

并具备以下访问权限：

- ✅ Kubernetes 集群（Minikube、kind、GKE、EKS 或 AKS）
- ✅ 容器仓库（Docker Hub、GCR、ECR）
- ✅ GitHub 账户（用于 CI/CD）
- ✅ 域名（可选，用于生产部署）

### 安装步骤

1. **仔细阅读本 README 及所有文档文件**
2. **阅读 `requirements.md`** 以了解所有功能性和非功能性需求
3. **研究 `architecture.md`** 以了解系统设计
4. **审查代码存根** 以了解需要实现的内容
5. **阅读 `docs/DEPLOYMENT.md`** 获取部署说明
6. **使用 `.env.example` 作为模板设置环境**

### 开发工作流

这是一个大型项目，以下是推荐的工作方式：

#### 阶段 1：系统集成（第 1 周）
- 集成项目 1 的 API
- 集成项目 2 的 Kubernetes 配置
- 集成项目 3 的 ML 流水线
- 集成项目 4 的监控
- 创建统一的配置管理
- 在本地测试端到端工作流

#### 阶段 2：CI/CD 实施（第 2 周）
- 设置 GitHub Actions 用于 CI
- 实现自动化测试
- 实现 Docker 构建与推送
- 设置预发布环境的 CD
- 设置需要手动审批的生产环境 CD
- 测试部署流水线

#### 阶段 3：生产加固（第 3 周）
- 实现安全功能（TLS、认证、密钥）
- 设置高可用性（HPA、滚动更新）
- 实现灾难恢复（备份、恢复流程）
- 完成文档
- 负载测试与性能调优
- 准备演示与汇报

---

## 关键挑战

本项目将考验你的以下能力：

1. **集成复杂系统** — 让 4 个独立项目无缝协同工作
2. **管理配置** — 处理多个环境（开发、预发布、生产）
3. **确保可靠性** — 通过自动自愈实现 99.9% 可用性
4. **实施安全措施** — 在不影响可用性的前提下实现生产级安全
5. **自动化一切** — CI/CD、ML 生命周期、监控、备份
6. **全面的文档撰写** — 创建他人能实际使用的文档
7. **从运维角度思考** — 为故障做准备，而不仅仅是成功场景

---

## 成功标准

你的结业项目将从以下方面进行评估：

### 技术实现（60%）
- ✅ 成功集成所有前 4 个项目
- ✅ 可工作的 CI/CD 流水线（CI + 预发布 + 生产）
- ✅ 实现生产级安全
- ✅ 展示高可用性（自动扩缩容、零停机更新）
- ✅ 端到端自动化 ML 生命周期
- ✅ 完整的可观测性（指标、日志、告警）

### 文档（20%）
- ✅ 全面的 README
- ✅ 包含架构图的设计文档
- ✅ 部署指南
- ✅ 灾难恢复计划
- ✅ API 文档
- ✅ 故障排查指南

### 演示与汇报（20%）
- ✅ 系统实时演示
- ✅ 关键组件讲解
- ✅ 展示性能指标
- ✅ 专业的汇报材料

**最低通过分数：75/100**
**作品集级分数：90/100**

---

## 学习资源

### 必读资料

1. **[12-Factor App 方法论](https://12factor.net/)** — 生产应用最佳实践
2. **[Google SRE 书籍](https://sre.google/sre-book/table-of-contents/)** — 站点可靠性工程原则
3. **[Kubernetes 生产最佳实践](https://learnk8s.io/production-best-practices)** — K8s 生产清单
4. **[MLOps：持续交付](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning)** — ML 部署模式

### 推荐工具文档

- [GitHub Actions](https://docs.github.com/en/actions)
- [Helm 最佳实践](https://helm.sh/docs/chart_best_practices/)
- [Kustomize 文档](https://kustomize.io/)
- [cert-manager](https://cert-manager.io/docs/)
- [HashiCorp Vault](https://developer.hashicorp.com/vault/docs)

### 示例项目

- [Kubeflow](https://github.com/kubeflow/kubeflow) — Kubernetes 上的 ML 平台
- [Seldon Core](https://github.com/SeldonIO/seldon-core) — ML 部署平台
- [MLflow 示例](https://github.com/mlflow/mlflow/tree/master/examples)

---

## 后续步骤

1. **阅读该目录下的所有文档文件**
2. **审查代码存根和 TODO**
3. **设置开发环境**
4. **创建包含里程碑的项目计划**
5. **从阶段 1：系统集成开始**
6. **遇到困难时寻求帮助**（使用 GitHub Discussions 或 Stack Overflow）
7. **基于测试和反馈进行迭代改进**

---

## 支持

### 获取帮助

- **文档**：阅读 `docs/` 目录下的所有文件
- **代码注释**：审查代码存根中的 TODO 和注释
- **以往项目**：参考你在项目 1-4 中的实现
- **社区**：MLOps 社区、Kubernetes 论坛、Stack Overflow

### 常见问题

参见 `docs/TROUBLESHOOTING.md` 了解常见问题的解决方案。

---

## 致谢

此结业项目整合了以下来源的概念：
- AI 基础设施初级工程师课程的项目 1-4
- 来自 Google SRE、CNCF 和 MLOps 社区的行业最佳实践
- 真实世界的大规模生产级 ML 系统

---

## 许可证

本项目是 AI 基础设施课程的一部分。
仅供教育用途。

---

**祝你的结业项目顺利！** 🚀

这是你展示所学知识、创建令雇主印象深刻的作品集项目的机会。慢慢来，做出让你自豪的东西，最重要的是 — 从遇到的挑战中学习。

**你一定行！** 💪
