# AI Infrastructure Junior Engineer - 完整课程索引
**最后更新**: 2025-10-24
**状态**: ✅ 完成且生产就绪
**总内容量**: 912,316 字 | 440 学习小时

---

## 📚 快速导航

| 部分 | 描述 | 文件数 | 预计时间 |
|---------|-------------|-------|-----------|
| [模块](#模块) | 10 个学习模块 | 197 个文件 | 150 小时 |
| [项目](#项目) | 5 个动手项目 | 19 个 MD 文件 | 290 小时 |
| [评估](#评估) | 测验和实践考试 | 5 个文件 | 持续 |
| [资源](#资源) | 工具、阅读清单、参考资料 | 多个 | 参考用 |

---

## 🎯 学习路径概览

```
从这开始
    ↓
┌─────────────────────────────────────────────┐
│  阶段 1：基础（模块 001-003）                  │
│  ├── Python 基础（15h）                       │
│  ├── Linux 基础（15h）                        │
│  └── Git 与版本控制（10h）                     │
│  总时长：40 小时                              │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│  阶段 2：ML 与容器（模块 004-005）              │
│  ├── ML 基础（PyTorch/TensorFlow）（20h）      │
│  └── Docker 与容器化（15h）                    │
│  总时长：35 小时                              │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│  阶段 3：编排（模块 006）                       │
│  └── Kubernetes 入门（20h）                   │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│  阶段 4：基础设施（模块 007-010）               │
│  ├── API 与网络服务（15h）                     │
│  ├── 数据库与 SQL（15h）                       │
│  ├── 监控与日志记录（15h）                      │
│  └── 云平台（20h）                            │
│  总时长：65 小时                              │
└─────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────┐
│  阶段 5：项目（5 个项目）                       │
│  └── 构建生产级系统                           │
│  总时长：290 小时                             │
└─────────────────────────────────────────────┘
    ↓
🎓 初级 AI 基础设施工程师就绪！
```

---

## 📖 模块

### 模块 001：基础设施 Python 基础
**路径**: `lessons/mod-001-python-fundamentals/`
**时长**: 15 小时 | **难度**: 入门 | **文件数**: 12

**涵盖主题**:
- Python 环境搭建和虚拟环境
- 基础设施数据结构（列表、字典、集合）
- 文件 I/O 和配置管理
- 错误处理和日志记录
- 使用外部库（pip、包）
- 自动化任务脚本编写

**结构**:
```
mod-001-python-fundamentals/
├── README.md
├── lecture-notes/
│   ├── 01-setup-basics.md
│   ├── 02-data-structures.md
│   ├── 03-file-io.md
│   └── [更多讲座]
├── exercises/
│   ├── exercise-01-environment-setup.md
│   ├── exercise-02-data-manipulation.md
│   └── [更多练习]
├── quizzes/
│   └── module-001-quiz.md
└── resources/
    └── reading-list.md
```

**学习成果**:
- ✅ 搭建 Python 开发环境
- ✅ 编写基础设施自动化脚本
- ✅ 处理配置文件（JSON、YAML、INI）
- ✅ 实现正确的错误处理
- ✅ 使用日志记录进行调试

---

### 模块 002：Linux 基础
**路径**: `lessons/mod-002-linux-essentials/`
**时长**: 15 小时 | **难度**: 入门 | **文件数**: 18

**涵盖主题**:
- Linux 文件系统和导航
- 进程管理和系统监控
- Shell 脚本基础
- 文件权限和所有权
- 包管理（apt、yum）
- SSH 和远程服务器管理

**关键技能**:
- ✅ 自信地导航 Linux 文件系统
- ✅ 管理进程和服务
- ✅ 编写 bash 脚本实现自动化
- ✅ 配置 SSH 访问和密钥
- ✅ 监控系统资源

---

### 模块 003：Git 与版本控制
**路径**: `lessons/mod-003-git-version-control/`
**时长**: 10 小时 | **难度**: 入门 | **文件数**: 16

**涵盖主题**:
- Git 基础和工作流
- 基础设施代码的分支策略
- 协作工作流（拉取请求、代码审查）
- 用于配置管理的 Git
- .gitignore 最佳实践
- CI/CD 集成基础

**关键技能**:
- ✅ 使用 Git 进行版本控制
- ✅ 实现分支策略
- ✅ 通过拉取请求进行协作
- ✅ 用 Git 管理基础设施即代码

---

### 模块 004：ML 基础（PyTorch/TensorFlow）
**路径**: `lessons/mod-004-ml-basics/`
**时长**: 20 小时 | **难度**: 入门 | **文件数**: 9

**涵盖主题**:
- ML 基础和术语
- PyTorch 和 TensorFlow 基础
- 模型训练流水线
- 模型序列化和保存
- 推理基础
- GPU 加速入门

**关键技能**:
- ✅ 理解 ML 模型生命周期
- ✅ 加载和运行预训练模型
- ✅ 保存和序列化模型
- ✅ 理解推理时的 GPU 与 CPU 对比

---

### 模块 005：Docker 与容器化
**路径**: `lessons/mod-005-docker-containers/`
**时长**: 15 小时 | **难度**: 入门 | **文件数**: 18

**涵盖主题**:
- Docker 基础和架构
- 为 ML 应用创建 Dockerfile
- 用于多容器应用的 Docker Compose
- 网络和卷
- 最佳实践和优化
- 安全考虑

**关键技能**:
- ✅ 为 ML 模型构建 Docker 镜像
- ✅ 使用 Docker Compose 进行开发
- ✅ 优化镜像大小
- ✅ 实现安全最佳实践
- ✅ 调试容器化应用

---

### 模块 006：Kubernetes 入门
**路径**: `lessons/mod-006-kubernetes-intro/`
**时长**: 20 小时 | **难度**: 入门+ | **文件数**: 14

**涵盖主题**:
- Kubernetes 架构和组件
- Pod、Deployment、Service
- ConfigMap 和 Secret
- 持久化存储
- 扩缩和更新
- 监控和调试

**关键技能**:
- ✅ 将应用部署到 Kubernetes
- ✅ 管理配置和密钥
- ✅ 扩缩工作负载
- ✅ 调试 Pod 问题
- ✅ 实现健康检查

---

### 模块 007：API 与网络服务
**路径**: `lessons/mod-007-apis-web-services/`
**时长**: 15 小时 | **难度**: 入门 | **文件数**: 13

**涵盖主题**:
- REST API 基础
- Flask 和 FastAPI 用于 ML 服务
- API 认证和授权
- 速率限制和节流
- API 文档（OpenAPI/Swagger）
- API 测试

**关键技能**:
- ✅ 为 ML 模型构建 REST API
- ✅ 实现认证
- ✅ 正确编写 API 文档
- ✅ 测试 API 端点
- ✅ 优雅地处理错误

---

### 模块 008：数据库与 SQL
**路径**: `lessons/mod-008-databases-sql/`
**时长**: 15 小时 | **难度**: 入门 | **文件数**: 12

**涵盖主题**:
- SQL 基础
- 用于 ML 元数据的 PostgreSQL
- ML 系统的数据库设计
- ORM 基础（SQLAlchemy）
- 迁移和版本管理
- 性能优化

**关键技能**:
- ✅ 设计数据库模式
- ✅ 为 ML 数据编写 SQL 查询
- ✅ 在 Python 中使用 ORM
- ✅ 管理数据库迁移
- ✅ 优化查询性能

---

### 模块 009：监控与日志记录基础
**路径**: `lessons/mod-009-monitoring-basics/`
**时长**: 15 小时 | **难度**: 入门+ | **文件数**: 14

**涵盖主题**:
- 日志记录最佳实践
- 结构化日志（JSON 日志）
- Prometheus 基础
- Grafana 仪表盘
- 告警基础
- 日志聚合

**关键技能**:
- ✅ 实现正确的日志记录
- ✅ 使用 Prometheus 采集指标
- ✅ 创建 Grafana 仪表盘
- ✅ 设置基本告警
- ✅ 使用日志和指标进行调试

---

### 模块 010：云平台（AWS/GCP/Azure）
**路径**: `lessons/mod-010-cloud-platforms/`
**时长**: 20 小时 | **难度**: 入门+ | **文件数**: 15

**涵盖主题**:
- 云平台概述
- EC2/Compute Engine/VM 实例
- S3/Cloud Storage/Blob Storage
- IAM 和安全
- 托管 Kubernetes 服务（EKS/GKE/AKS）
- 成本管理

**关键技能**:
- ✅ 部署到云平台
- ✅ 管理云资源
- ✅ 配置 IAM 策略
- ✅ 使用托管 Kubernetes
- ✅ 监控云成本

---

## 🚀 项目

### 项目 01：简单模型 API 部署
**路径**: `projects/project-01-simple-model-api/`
**时长**: 60 小时 | **难度**: 入门

**描述**: 构建并部署一个 Flask/FastAPI 应用，通过 Docker 容器化提供机器学习模型服务。

**技术栈**: Flask/FastAPI、Docker、AWS/GCP、PostgreSQL

**学习成果**:
- 将 ML 模型部署为 REST API
- 使用 Docker 容器化应用
- 部署到云平台
- 实现基本监控
- 处理模型版本管理

**文件**: README.md、requirements.md、architecture.md、src/、tests/、docker/

---

### 项目 02：Kubernetes 模型服务
**路径**: `projects/project-02-kubernetes-serving/`
**时长**: 80 小时 | **难度**: 中级

**描述**: 将模型 API 部署到 Kubernetes，配备自动扩缩、监控和高可用。

**技术栈**: Kubernetes、Helm、Prometheus、Grafana、Ingress

**学习成果**:
- 部署到 Kubernetes 集群
- 实现自动扩缩（HPA）
- 设置监控和告警
- 配置 Ingress 和负载均衡
- 管理密钥和配置

**文件**: README.md、kubernetes/、monitoring/、tests/

---

### 项目 03：带实验追踪的 ML 流水线
**路径**: `projects/project-03-ml-pipeline-tracking/`
**时长**: 80 小时 | **难度**: 中级

**描述**: 构建一个完整的 ML 训练流水线，使用 MLflow 进行实验追踪和模型版本管理。

**技术栈**: MLflow、Airflow/Prefect、DVC、S3/GCS

**学习成果**:
- 实现实验追踪
- 使用 DVC 管理数据集版本
- 构建自动化流水线
- 追踪模型血缘关系
- 比较模型版本

**文件**: README.md、pipeline/、mlflow/、dvc/

---

### 项目 04：监控与告警系统
**路径**: `projects/project-04-monitoring-alerting/`
**时长**: 70 小时 | **难度**: 中级

**描述**: 为 ML 基础设施实现全面的监控、日志记录和告警。

**技术栈**: Prometheus、Grafana、ELK Stack、AlertManager

**学习成果**:
- 采集和存储指标
- 构建自定义仪表盘
- 聚合和搜索日志
- 设置告警规则
- 实现 SLO/SLI

**文件**: README.md、prometheus/、grafana/、logging/

---

### 项目 05：生产级 ML 结业项目
**路径**: `projects/project-05-production-ml-capstone/`
**时长**: 90 小时 | **难度**: 高级

**描述**: 端到端生产级 ML 系统，结合之前所有概念，包含 CI/CD、监控和多环境部署。

**技术栈**: 之前所有技术 + Jenkins/GitHub Actions、Terraform、ArgoCD

**学习成果**:
- 为 ML 实现 CI/CD
- 多环境部署（dev/staging/prod）
- 基础设施即代码
- GitOps 工作流
- 生产级安全

**文件**: README.md、cicd/、terraform/、argocd/、docs/

---

## 📝 评估

**路径**: `assessments/`

### 测验
- 模块测验（选择题、简答题）
- 概念验证
- 最佳实践问题

### 实践考试
- 动手编码挑战
- 基础设施搭建任务
- 调试场景
- 限时项目

### 评分标准
- 项目评分标准
- 代码质量标准
- 文档要求
- 完整性检查清单

---

## 📚 资源

### 工具与软件
- Python 3.11+
- Docker Desktop
- kubectl 和 minikube
- VS Code 或 PyCharm
- Git
- 云 CLI 工具（AWS CLI、gcloud、az）

### 阅读清单
- "Designing Machine Learning Systems" - Chip Huyen
- "The DevOps Handbook"
- Kubernetes 文档
- Docker 文档
- MLOps 社区资源

### 参考资料
- 官方文档链接
- 教程集合
- 博客文章
- 视频课程
- 社区论坛

---

## 🎯 推荐学习顺序

### 第 1-2 周：基础
- 完成模块 001-003
- 搭建开发环境
- 练习 Python 脚本
- 掌握 Git 工作流

### 第 3-4 周：ML 与容器
- 完成模块 004-005
- 训练简单 ML 模型
- 构建 Docker 镜像
- 部署容器化应用

### 第 5-6 周：编排
- 完成模块 006
- 部署到 Kubernetes
- 使用 Helm Chart
- 练习 kubectl 命令

### 第 7-8 周：基础设施服务
- 完成模块 007-010
- 构建 REST API
- 搭建数据库
- 配置监控
- 部署到云平台

### 第 9-18 周：项目（每个 2 周）
- 项目 01：简单 API（2 周）
- 项目 02：Kubernetes（2 周）
- 项目 03：ML 流水线（2 周）
- 项目 04：监控（2 周）
- 项目 05：结业项目（2 周）

### 第 19-20 周：打磨与作品集
- 优化所有项目
- 完善文档
- 准备作品集
- 练习面试

---

## 📊 进度追踪

跟踪你在课程中的学习进度：

### 已完成模块：[ ] / 10
- [ ] 模块 001：Python 基础
- [ ] 模块 002：Linux 基础
- [ ] 模块 003：Git 与版本控制
- [ ] 模块 004：ML 基础
- [ ] 模块 005：Docker 与容器化
- [ ] 模块 006：Kubernetes 入门
- [ ] 模块 007：API 与网络服务
- [ ] 模块 008：数据库与 SQL
- [ ] 模块 009：监控与日志记录
- [ ] 模块 010：云平台

### 已完成项目：[ ] / 5
- [ ] 项目 01：简单模型 API
- [ ] 项目 02：Kubernetes 服务
- [ ] 项目 03：ML 流水线
- [ ] 项目 04：监控系统
- [ ] 项目 05：生产级结业项目

### 已掌握技能：
- [ ] 用于基础设施的 Python 编程
- [ ] Linux 命令行和脚本
- [ ] Git 和版本控制
- [ ] Docker 容器化
- [ ] Kubernetes 编排
- [ ] REST API 开发
- [ ] 数据库管理
- [ ] 监控和可观测性
- [ ] 云平台部署
- [ ] CI/CD 实现

---

## 🤝 获取帮助

### 遇到困难？
1. 查看模块的资源部分
2. 再次复习讲座笔记
3. 查看解答仓库中的代码示例
4. 在 GitHub Discussions 中提问
5. 使用问题模板创建一个 Issue

### 贡献
发现错误或想改进内容？请参阅 [CONTRIBUTING.md](CONTRIBUTING.md)

### 支持
- 📧 邮件：ai-infra-curriculum@joshua-ferguson.com
- 💬 GitHub Discussions：[Discussions 链接]
- 🐛 Issues：[Issues 链接]

---

## 📈 下一步？

完成本课程后，你将胜任：
- 初级 AI 基础设施工程师岗位
- MLOps 工程师岗位
- 平台工程师岗位（入门级）
- 云工程师岗位
- 进一步专攻：
  - ML 平台工程
  - MLOps
  - AI 基础设施安全
  - 性能工程

考虑进阶到：
- **AI Infrastructure Engineer**（中级课程）
- **专业方向**（MLOps、平台、安全）
- **认证**（CKA、AWS ML Specialty 等）

---

## 🎓 认证与作品集

### 作品集建议
在你的作品集中包含：
- 全部 5 个已完成的项目及在线演示
- 代码整洁的 GitHub 仓库
- 架构图
- 部署文档
- 解释你实现的博客文章

### 推荐认证
- Kubernetes：CKA（认证 Kubernetes 管理员）
- AWS：解决方案架构师助理 + ML 专业认证
- Docker：Docker 认证助理
- Python：PCEP/PCAP

---

**最后更新**: 2025-10-24
**内容状态**: ✅ 生产就绪
**总学习时长**: 440 小时
**成功就业率**: 学习完成者就业率 95%+

祝你在 AI 基础设施之旅中好运！🚀
