# Junior AI Infrastructure Engineer - 学习资料库

<!-- aicg:site-banner -->
> 🎓 属于免费开源 **AI Career Curriculum** 生态系统的一部分 — [Infrastructure](https://github.com/ai-infra-curriculum) · [ML Engineering](https://github.com/ml-engineering-curriculum) · [AI Engineering](https://github.com/ai-engineering-curriculum) · [Governance](https://github.com/ai-governance-curriculum). 实时学习小组和团队课程：**[ai-infra-curriculum.github.io](https://ai-infra-curriculum.github.io/)**。
<!-- /aicg:site-banner -->

<!-- aicg:sponsor -->
> 💜 **[赞助本课程](https://github.com/sponsors/AI-Infra-Curriculum)** — 您的赞助让整个开源 AI Career Curriculum 保持免费并持续发展。
<!-- /aicg:sponsor -->

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub Issues](https://img.shields.io/github/issues/ai-infra-curriculum/ai-infra-junior-engineer-learning)](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning/issues)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

欢迎来到 **Junior AI Infrastructure Engineer 学习资料库**！本综合课程旨在带你从零基础到具备就业能力的 AI/ML 基础设施工程师，包含动手实践项目和行业相关技能。

## 🎯 学习目标

完成本课程后，你将能够：

- **通过 Flask/FastAPI 的 REST API 部署 ML 模型**
- **使用 Docker 容器化应用**
- **在 Kubernetes 上编排部署**
- **构建带实验追踪的 ML 流水线（MLflow）**
- **使用 Prometheus 和 Grafana 实现监控**
- **使用 DVC 进行数据集版本管理**
- **使用 Airflow/Prefect 自动化工作流**
- **部署到云平台（AWS/GCP/Azure）**
- **为 ML 应用实现 CI/CD**
- **为生产系统应用安全最佳实践**

## ✨ 最新动态

**2026 年 5 月更新 — 内容完整性优化：**
- 🐍 **Flask 框架讲座**（模块 007）- 何时选择 Flask vs FastAPI、应用工厂、蓝图、扩展、ML 服务模式
- 🧮 **NoSQL 基础讲座**（模块 008）- MongoDB、Redis、向量数据库、ML 平台的混合持久化
- ⎈ **Helm Chart**（项目 02）- 生产级 Chart，包含 HPA、ServiceMonitor、可配置模型存储
- 📊 **Grafana 仪表盘**（项目 02 和 04）- 模型 API 概览、ML 漂移追踪、SLI/SLO 错误预算
- 🔥 **负载测试计划**（项目 02）- 包含烟雾/爬坡/持久/突发场景及验收标准的 Locust 文件
- 📐 **DVC 流水线**（项目 03）- 5 阶段流水线（数据摄入 → 预处理 → 验证 → 训练 → 评估），带参数管理
- ✅ **Great Expectations 套件**（项目 03）- 13 条期望规则，阻止在问题数据上训练
- ♻️ **重训练 DAG**（项目 03）- 每周定时调度，带漂移检测传感器和门控 MLflow 升级
- 🚨 **Alertmanager + 应急预案**（项目 04）- 路由树、5 个关键告警的结构化应急预案
- 🏗️ **Terraform 模块**（项目 05）- VPC、EKS、RDS、IAM 模块，支持 dev/staging/prod 环境
- 💾 **Velero 备份**（项目 05）- 含每日/每小时备份计划和季度恢复演练的 Helm values
- 📒 **速查表** - Python、Linux、SQL、Prometheus 快速参考
- 🗺️ **中级 + 高级阅读路径** - 6 个月和 2-5 年的精选阅读清单

**更早的补充：**
- 🤖 **LLM 基础练习**（模块 004）- 使用 Hugging Face Transformers 运行第一个语言模型
- ⚡ **GPU 基础练习**（模块 004）- 学习 ML 推理的 GPU 加速
- 🏗️ **Terraform/IaC 练习**（模块 010）- 通过动手 AWS 部署实践基础设施即代码
- 🔄 **Airflow 工作流练习**（模块 009）- 带监控的 ML 流水线编排
- 📋 **技术版本指南** - 所有工具的完整版本规格
- 🗺️ **课程交叉引用** - 与 Engineer 轨道的完整对照
- 📈 **职业发展指南** - 从初级到首席工程师的职业路线

## 📚 课程概览

**总时长：** 440 小时（全日制 11 周，非全日制 22 周）
**难度：** 入门到中级
**先修要求：** 基础 Python 编程，命令行基础

### 学习模块（10 个模块，约 150 小时）

| 模块 | 主题 | 时长 | 难度 | 新增内容 |
|--------|-------|----------|------------|-------------|
| **001** | 基础设施 Python 基础 | 15 小时 | 入门 | — |
| **002** | Linux 基础 | 15 小时 | 入门 | — |
| **003** | Git 与版本控制 | 10 小时 | 入门 | — |
| **004** | ML 基础（PyTorch/TensorFlow） | 20 小时 | 入门 | ✨ **+LLM 基础 +GPU 基础** |
| **005** | Docker 与容器化 | 15 小时 | 入门 | — |
| **006** | Kubernetes 入门 | 20 小时 | 入门+ | — |
| **007** | API 与网络服务 | 15 小时 | 入门 | ✨ **+Flask 框架讲座** |
| **008** | 数据库与 SQL | 15 小时 | 入门 | ✨ **+NoSQL 基础讲座** |
| **009** | 监控与日志记录基础 | 15 小时 | 入门+ | ✨ **+Airflow 工作流** |
| **010** | 云平台（AWS/GCP/Azure） | 20 小时 | 入门+ | ✨ **+Terraform/IaC** |

### 动手实践项目（5 个项目，约 290 小时）

| 项目 | 描述 | 时长 | 技术栈 |
|---------|-------------|--------------|--------------|
| **01** | [简单模型 API 部署](projects/project-01-simple-model-api/) | 60 小时 | Flask/FastAPI, Docker, AWS/GCP |
| **02** | [Kubernetes 模型服务](projects/project-02-kubernetes-serving/) | 80 小时 | Kubernetes, Helm, Prometheus |
| **03** | [带实验追踪的 ML 流水线](projects/project-03-ml-pipeline-tracking/) | 100 小时 | MLflow, Airflow, DVC |
| **04** | [监控与告警系统](projects/project-04-monitoring-alerting/) | 80 小时 | Prometheus, Grafana, ELK Stack |
| **05** | [生产级 ML 系统（结业项目）](projects/project-05-production-ml-capstone/) | 120 小时 | 以上所有 + CI/CD |

## 🚀 快速开始

### 先决条件

开始前，请确保你具备：

- 已安装 **Python 3.11+**
- **Docker Desktop** 或 Docker Engine
- **Git** 用于版本控制
- **代码编辑器**（VS Code、PyCharm 等）
- **云账号**（免费套餐）：AWS、GCP 或 Azure
- **基本命令行技能**

### 设置步骤

1. **克隆本仓库：**
   ```bash
   git clone https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning.git
   cd ai-infra-junior-engineer-learning
   ```

2. **配置 Python 环境：**
   ```bash
   python3 -m venv venv
   source venv/bin/activate  # Windows: venv\Scripts\activate
   pip install -r requirements.txt
   ```

3. **安装 Docker：**
   - 按照 [docs.docker.com](https://docs.docker.com/get-docker/) 的操作说明
   - 验证：`docker --version`

4. **选择你的学习路径：**
   - **零基础：** 从模块 001 开始
   - **有一定经验：** 复习模块 001-003，从模块 004 开始
   - **准备做项目：** 直接跳到项目 01

### 推荐学习路径

```
第 1-2 周：  模块 001-003（Python、Linux、Git）
第 3-4 周：  模块 004-005（ML 基础、Docker）
第 5 周：    模块 006（Kubernetes）+ 项目 01
第 6-7 周：  项目 01（简单模型 API）
第 8 周：    模块 007-008（API、数据库）+ 项目 02
第 9-10 周： 项目 02（Kubernetes 服务）
第 11 周：   模块 009-010（监控、云平台）
第 12-14 周：项目 03（ML 流水线）
第 15-16 周：项目 04（监控）
第 17-20 周：项目 05（结业项目）
第 21-22 周：作品集打磨、求职投递
```

## 📖 仓库结构

```
ai-infra-junior-engineer-learning/
├── README.md                    # 本文件
├── CURRICULUM.md                # 详细课程指南
├── CODE_OF_CONDUCT.md          # 社区准则
├── CONTRIBUTING.md             # 贡献指南
├── LICENSE                     # MIT 许可证
├── requirements.txt            # Python 依赖
├── .github/                    # GitHub 模板和工作流
│   ├── workflows/              # CI/CD 自动化
│   └── ISSUE_TEMPLATE/         # Issue 模板
├── lessons/                    # 学习模块
│   ├── mod-001-python-fundamentals/
│   │   ├── README.md           # 模块概述
│   │   ├── lecture-notes/      # 讲座内容
│   │   ├── exercises/          # 动手练习
│   │   ├── quizzes/            # 知识测验
│   │   └── resources/          # 附加材料
│   ├── mod-002-linux-essentials/
│   └── ...                     # 更多模块
├── projects/                   # 动手项目
│   ├── project-01-simple-model-api/
│   │   ├── README.md           # 项目指南
│   │   ├── requirements.md     # 需求规格
│   │   ├── architecture.md     # 架构深度解析
│   │   ├── src/                # 代码框架
│   │   ├── tests/              # 测试框架
│   │   └── docker/             # Docker 配置
│   ├── project-02-kubernetes-serving/
│   │   ├── kubernetes/         # 原始 K8s 清单（教学用）
│   │   ├── helm/model-api/     # Helm Chart（运维用）
│   │   ├── grafana/            # 仪表盘 JSON
│   │   ├── loadtest/           # Locust 负载测试
│   │   └── monitoring/         # ServiceMonitor
│   ├── project-03-ml-pipeline-tracking/
│   │   ├── dags/               # Airflow DAG（含重训练）
│   │   ├── dvc/                # DVC 流水线 + 参数
│   │   ├── great_expectations/ # 数据验证套件
│   │   ├── mlflow/             # MLflow 项目
│   │   └── src/                # 流水线阶段
│   ├── project-04-monitoring-alerting/
│   │   ├── prometheus/         # Prometheus + 告警
│   │   ├── alertmanager/       # 路由 + Slack/PagerDuty
│   │   ├── grafana/dashboards/ # ML 概览、漂移、SLI/SLO
│   │   ├── elasticsearch/      # ELK 配置
│   │   └── runbooks/           # 事故处理手册
│   └── project-05-production-ml-capstone/
│       ├── terraform/          # VPC + EKS + RDS + IAM（dev/staging/prod）
│       ├── kubernetes/         # Kustomize base + overlays
│       ├── security/           # cert-manager + Vault
│       ├── velero/             # 集群备份
│       ├── monitoring/         # SLO
│       ├── cicd/               # GitHub Actions
│       └── docs/               # 部署 + 灾难恢复计划
├── assessments/                # 测验与考试
│   ├── quizzes/                # 模块测验
│   ├── practical-exams/        # 动手评估
│   └── rubrics/                # 评分标准
├── resources/                  # 附加资源
│   ├── cheat-sheets/           # Python、Linux、SQL、Git、Docker、K8s、Prometheus
│   ├── reading-lists/          # 入门、中级、高级路径
│   └── tools.md                # 工具推荐
├── progress/                   # 学习进度追踪
│   ├── overall-progress.md     # 总体进度
│   ├── module-progress.md      # 模块检查清单
│   └── project-progress.md     # 项目进度
└── community/                  # 社区资源
    ├── FAQ.md                  # 常见问题
    └── study-groups.md         # 学习小组信息
```

## 🎓 评估与认证

### 评估结构

每个项目包含：
- **100 分评分标准**，包含清晰标准
- **及格分数：** 70 分
- **优秀：** 90 分以上
- **自我评估清单**
- **同行评审指南**

### 评分细则

- **代码质量**（25%）：整洁、可维护、文档齐全的代码
- **功能完整性**（30%）：满足所有需求，按预期工作
- **基础设施**（20%）：正确部署、可扩展性、可靠性
- **文档**（15%）：清晰的 README、API 文档、部署指南
- **性能**（10%）：满足性能基准

### 作品集构建

每个项目都设计为 **值得放入作品集**：
- 专业的 README 文件
- 在线演示（如有可能）
- 架构图
- 性能指标
- GitHub Actions CI/CD

**向潜在雇主展示你的结业项目**（项目 05）！

## 🛠️ 技术栈

### 核心技术

- **语言：** Python 3.11+、Bash、YAML
- **ML 框架：** PyTorch 2.0+、TensorFlow 2.13+、Scikit-learn 1.3+
- **容器与编排：** Docker 24.0+、Kubernetes 1.28+、Helm 3.12+
- **MLOps：** MLflow 2.8+、DVC 3.30+、Airflow 2.7+ / Prefect 2.13+
- **监控：** Prometheus 2.47+、Grafana 10.2+、ELK Stack 8.11+
- **云平台：** AWS / GCP / Azure（选择一个）
- **数据库：** PostgreSQL 15+、Redis 7+
- **CI/CD：** GitHub Actions、GitLab CI

### 开发工具

- **IDE：** VS Code、PyCharm 或类似
- **版本控制：** Git、GitHub
- **API 测试：** Postman、cURL、httpie
- **容器管理：** Docker Desktop、Minikube、Kind
- **云 CLI：** aws-cli、gcloud、az

## 💼 职业发展

### 学完后你将能胜任

完成本课程后，你将具备以下岗位的资格：

- **初级 ML 基础设施工程师**
- **初级 MLOps 工程师**
- **DevOps 工程师（ML 方向）**
- **云基础设施工程师（入门级）**
- **ML 平台工程师（初级）**

### 预期薪资范围（美国，2025 年）

- **入门级：** $67,000 - $95,000
- **1-2 年经验：** $80,000 - $110,000
- **地域差异：** 科技中心（旧金山、纽约、西雅图）更高

### 你将掌握的技能

**技术方面：**
- ML 模型部署与提供
- 容器编排（Docker、Kubernetes）
- 基础设施即代码（Terraform 基础）
- CI/CD 流水线开发
- 监控与可观测性
- 云平台管理
- 数据流水线开发

**职业素养：**
- 解决问题能力
- 技术文档写作
- 代码审查
- 敏捷方法论
- 协作与沟通

## 📝 如何使用本仓库

### 自学者

1. **从模块 001 开始**，按顺序学习
2. **完成每个模块的所有练习**
3. **按照需求构建每个项目**
4. **使用提供的评分标准进行自我评估**
5. **在 GitHub 上分享你的作品**，构建作品集
6. **加入社区**，获得支持和人脉

### 教师

本仓库提供：
- **可直接使用的课程**，适合一学期课程
- **全面的讲座材料**，附带练习
- **动手实践项目**，配有详细说明
- **评估评分标准**，用于评分
- **分层学习**，从入门到中级

**建议课程结构：** 16 周学期，每周 10 小时

### 学习小组

- 使用 **讨论论坛** 进行协作
- 安排 **每周聚会** 回顾模块
- **结对编程** 完成项目
- **互相审查** 彼此的代码
- 分享资源和技巧

## 🤝 参与贡献

欢迎贡献！请参阅 [CONTRIBUTING.md](CONTRIBUTING.md) 了解指南。

**贡献方式：**
- 报告 Bug 或问题
- 提出改进建议
- 添加练习或资源
- 分享你的项目解决方案（在讨论中）
- 改进文档
- 创建教程视频

## 📞 支持与社区

### 获取帮助

- **GitHub Discussions：** 提问、分享见解
- **GitHub Issues：** 报告 Bug 或内容问题
- **FAQ：** 参见 [community/FAQ.md](community/FAQ.md)
- **邮件：** ai-infra-curriculum@joshua-ferguson.com

### 社区资源

- **学习小组：** 通过 [GitHub Discussions](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning/discussions) 加入或创建小组。
- **主要讨论渠道：** GitHub Discussions 和 Issues 是当前支持的渠道。目前未安排专门的聊天/办公时间——如果你想与其他学习者协调，请开启一个 Discussion。

## 📜 许可证

本项目采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件。

## 🙏 致谢

- **贡献者：** 感谢所有贡献者！
- **评审者：** 验证本课程的行业专家
- **开源社区：** 感谢我们所依赖的出色工具
- **学生：** 你们的反馈让本课程更好

## 🔗 相关资源

### 课程文档

- 📋 **[技术版本指南](VERSIONS.md)** - 所有工具和框架的推荐版本
- 🗺️ **[课程交叉引用](https://github.com/ai-infra-curriculum/.github/blob/main/CURRICULUM_CROSS_REFERENCE.md)** - Junior 和 Engineer 轨道之间的对照
- 📈 **[职业发展指南](https://github.com/ai-infra-curriculum/.github/blob/main/CAREER_PROGRESSION.md)** - 从初级到首席工程师的完整职业阶梯

### 其他课程级别

- **下一级别：** [AI Infrastructure Engineer](https://github.com/ai-infra-curriculum/ai-infra-engineer-learning)
- **专业方向：**
  - [ML Platform Engineer](https://github.com/ai-infra-curriculum/ai-infra-ml-platform-learning)
  - [MLOps Engineer](https://github.com/ai-infra-curriculum/ai-infra-mlops-learning)
  - [Security Engineer](https://github.com/ai-infra-curriculum/ai-infra-security-learning)
  - [Performance Engineer](https://github.com/ai-infra-curriculum/ai-infra-performance-learning)
- **架构方向：** [AI Infrastructure Architect](https://github.com/ai-infra-curriculum/ai-infra-architect-learning)
- **领导力方向：** [Team Lead / Manager](https://github.com/ai-infra-curriculum/ai-infra-team-lead-learning)

### 解答仓库

寻找完整的解决方案和分步指南？
👉 **[Junior AI Infrastructure Engineer - 解答](https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-solutions)**

**注意：** 我们建议在参考解答前先自己尝试完成项目！

---

**准备好开始你的 AI 基础设施职业生涯了吗？** 🚀
从 [模块 001：Python 基础](lessons/mod-001-python-fundamentals/) 开始

**有问题？** 开启一个 Issue 或加入我们的社区讨论！

**最后更新：** 2026 年 5 月
**版本：** 1.1.0

---

<!-- aicg:maintained-by -->
由 [VeriSwarm.ai](https://veriswarm.ai) 维护
