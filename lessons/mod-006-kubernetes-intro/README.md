# Module 006: Kubernetes 入门

## 模块概述

欢迎来到 Module 006，在这里你将学习 Kubernetes (K8s) 的基础知识，这是业界标准的容器编排平台。本模块全面介绍 Kubernetes 架构、核心概念和 AI 基础设施工作中必不可少的实践操作。

Kubernetes 已成为部署、扩展和管理容器化应用（包括 ML 模型和 AI 工作负载）的事实标准。理解 Kubernetes 对任何 AI 基础设施工程师都至关重要，因为它为构建可扩展、有弹性的 ML 系统奠定了基础。

## 学习目标

完成本模块后，你将能够：

- **理解 Kubernetes 架构**：掌握 Kubernetes 的核心组件，包括控制平面、工作节点及其交互方式
- **部署应用**：创建和管理 Pod、Deployment、Service 等 Kubernetes 资源
- **使用 Helm**：使用 Helm chart 打包和部署应用
- **运维 Kubernetes 集群**：有效使用 kubectl、调试问题并监控集群健康状态
- **将 K8s 应用于 ML 工作负载**：理解 Kubernetes 如何支持 AI/ML 基础设施需求

## 前置要求

在开始本模块之前，你应该具备：

### 必需知识
- **Module 003**：Docker 基础（容器化概念）
- **Module 004**：容器编排基础
- **Linux CLI**：基本的命令行能力
- **YAML**：理解 YAML 语法和结构
- **网络**：基础网络概念（端口、DNS、负载均衡）

### 必需工具
- 启用了 Kubernetes 的 Docker Desktop **或** 安装了 Minikube
- kubectl CLI 工具 (1.25+ 版本)
- Helm CLI (3.10+ 版本)
- 代码编辑器（推荐 VS Code 配合 Kubernetes 扩展）
- 至少 8GB 内存可用于本地集群

### 推荐设置
```bash
# 验证 Docker
docker --version

# 验证 Kubernetes 集群
kubectl cluster-info

# 验证 Helm
helm version

# 检查集群节点
kubectl get nodes
```

如果你还没有本地 Kubernetes 集群，请参阅 `resources/cluster-setup-guide.md` 获取详细安装说明。

## 模块结构

本模块分为四个主要部分：

### 1. Kubernetes 架构（Lecture 01）
**时长**: 3-4 小时

- 控制平面组件（API Server、Scheduler、Controller Manager、etcd）
- 工作节点组件（kubelet、kube-proxy、container runtime）
- Kubernetes 对象和 API 资源
- 集群网络模型
- Kubernetes 如何调度和管理容器

### 2. 部署应用（Lecture 02）
**时长**: 4-5 小时

- Pod：最小的可部署单元
- Deployment：管理应用生命周期
- Service：暴露应用
- ConfigMap 和 Secret：管理配置
- 使用 Volume 和 PersistentVolumeClaim 的持久化存储

### 3. Helm 包管理（Lecture 03）
**时长**: 3-4 小时

- Helm 和包管理简介
- Helm chart 结构和模板
- 安装和管理 release
- 创建自定义 Helm chart
- Helm 仓库和 chart 分发

### 4. Kubernetes 运维（Lecture 04）
**时长**: 3-4 小时

- kubectl 命令参考和快捷键
- 调试 Pod 和 Deployment
- 查看日志和在容器中执行命令
- 资源监控和指标
- 常见问题排查

## 学习路径

### 第一周：基础
**第 1-2 天**: Kubernetes 架构
- 阅读 lecture-notes/01-k8s-architecture.md
- 观看推荐的 K8s 架构视频
- 设置本地 Kubernetes 集群
- 使用 kubectl 探索集群组件

**第 3-4 天**: 部署应用
- 阅读 lecture-notes/02-deploying-apps.md
- 完成 exercise-01-first-deployment.md
- 练习创建 Pod、Deployment、Service
- 实验不同的 Service 类型

### 第二周：高级主题
**第 5-6 天**: Helm 包管理
- 阅读 lecture-notes/03-helm.md
- 完成 exercise-02-helm-chart.md
- 从 Helm Hub 安装应用
- 创建你的第一个 Helm chart

**第 7 天**: 运维和调试
- 阅读 lecture-notes/04-k8s-operations.md
- 完成 exercise-03-debugging.md
- 练习排障场景
- 回顾 kubectl 命令快捷键

### 第三周：练习和评估
**第 8-9 天**: 动手实践
- 在 K8s 上构建完整的 ML 模型服务应用
- 使用 Helm 部署
- 实现监控和日志
- 记录你的部署

**第 10 天**: 评估
- 完成 module-006-quiz.md
- 学习目标的自我评估
- 确定进一步学习的领域

## 动手练习

本模块包含三个综合练习：

### Exercise 01：首次部署
将简单的 Web 应用部署到 Kubernetes，通过 Service 暴露它，并上下扩展。此练习强化 Pod、Deployment 和 Service 的核心概念。

**练习的技能**：
- 编写 Kubernetes manifest
- 使用 kubectl 管理资源
- 暴露应用
- 扩展应用

### Exercise 02：Helm Chart
为 Python Flask 应用创建 Helm chart，参数化部署，并使用不同的配置进行安装。

**练习的技能**：
- Helm chart 结构
- 使用 Go template 进行模板化
- values 自定义
- Release 管理

### Exercise 03：调试
排查损坏的 Kubernetes 部署，使用 kubectl 识别问题，修复配置问题，验证应用正常工作。

**练习的技能**：
- 使用 kubectl 调试
- 读取日志和事件
- 识别常见问题
- 修复错误配置

## 评估标准

你应该能够证明：

1. **概念理解** (25%)
   - 解释 Kubernetes 架构组件
   - 描述 Kubernetes 如何调度工作负载
   - 理解 Kubernetes 网络模型
   - 解释 Pod、Deployment 和 Service 之间的关系

2. **实践技能** (50%)
   - 使用 kubectl 部署应用
   - 编写 Kubernetes manifest 文件
   - 创建和自定义 Helm chart
   - 使用 kubectl 检查和调试资源
   - 排查常见的 Kubernetes 问题

3. **最佳实践** (25%)
   - 应用资源 limits 和 requests
   - 有效使用 labels 和 selectors
   - 实现健康检查（liveness/readiness probes）
   - 使用 ConfigMap 和 Secret 管理配置
   - 遵循安全最佳实践

## 你将掌握的关键 Kubernetes 资源

```yaml
# 核心工作负载资源
- Pod                    # 运行进程的单个实例
- Deployment            # Pod 的声明式更新
- ReplicaSet            # 维持 Pod 副本的期望数量
- StatefulSet           # 用于有状态应用
- DaemonSet             # 在所有/选定节点上运行
- Job                   # 运行完成即结束的任务
- CronJob               # 定时任务

# 服务和网络
- Service               # 暴露应用
- Ingress               # HTTP/HTTPS 路由
- NetworkPolicy         # 网络访问控制

# 配置和存储
- ConfigMap             # 配置数据
- Secret                # 敏感数据
- PersistentVolume      # 集群存储
- PersistentVolumeClaim # 存储请求

# 集群资源
- Namespace             # 虚拟集群
- Node                  # 工作节点
- ServiceAccount        # 进程的标识
```

## 真实世界的 AI/ML 用例

在整个模块中，我们将 Kubernetes 概念与 AI 基础设施场景联系起来：

- **模型服务**：将 ML 模型部署为具有自动扩缩能力的微服务
- **训练任务**：使用 Job 和 GPU 资源运行分布式训练工作负载
- **数据处理**：使用 CronJob 调度 ETL 管道
- **特征存储**：使用 StatefulSet 部署有状态特征存储
- **实验跟踪**：在 Kubernetes 上运行 MLflow 或类似工具
- **Jupyter Notebook**：提供按需 notebook 环境

## 时间投入

- **讲义阅读**：10-12 小时
- **动手练习**：8-10 小时
- **练习和探索**：6-8 小时
- **评估**：2 小时

**总计**：大约 26-32 小时，分布在 2-3 周内

这是密集的材料，所以请保持自己的节奏，确保在继续之前理解每个概念。

## 成功指标

当你能够做到以下几点时，即成功完成本模块：

- [ ] 解释每个 Kubernetes 控制平面组件的作用
- [ ] 使用 kubectl 创建和管理 Pod
- [ ] 编写带有适当资源限制的 Deployment manifest
- [ ] 使用不同的 Service 类型暴露应用
- [ ] 创建和安装 Helm chart
- [ ] 调试失败的 Pod 和 Deployment
- [ ] 监控集群资源
- [ ] 应用生产部署的最佳实践
- [ ] 将完整的 ML 应用部署到 Kubernetes

## 常见挑战和技巧

### 挑战 1: YAML 语法错误
**解决方案**: 使用 YAML 验证器，启用编辑器语法检查，从工作示例开始。

### 挑战 2: 资源无法启动
**解决方案**: 始终检查 `kubectl describe pod <name>` 的事件和 `kubectl logs <pod>` 的应用日志。

### 挑战 3: 网络问题
**解决方案**: 理解 Service 类型，验证 selector 匹配 Pod labels，检查集群网络是否正常工作。

### 挑战 4: Helm 模板错误
**解决方案**: 使用 `helm template` 在本地渲染 chart，验证 YAML 输出，逐步测试。

### 挑战 5: 集群资源耗尽
**解决方案**: 设置资源 requests/limits，监控集群容量，定期清理未使用的资源。

## 获取帮助

如果遇到困难：

1. **检查讲义**：大多数常见问题都有解答
2. **回顾练习**：提供了逐步指导
3. **查阅官方文档**：https://kubernetes.io/docs/
4. **使用 kubectl explain**：`kubectl explain deployment.spec` 显示资源文档
5. **系统化调试**：按照 Lecture 04 中的排障指南操作
6. **社区资源**：Kubernetes Slack、Stack Overflow、Reddit r/kubernetes

## 后续步骤

完成本模块后：

- **Module 007**：高级 Kubernetes - StatefulSet、自定义资源、operator
- **Module 008**：Kubernetes 监控 - Prometheus、Grafana、日志
- **Module 009**：CI/CD 与 Kubernetes - GitOps、ArgoCD、自动化部署
- **Project 03**：在 Kubernetes 上部署完整的 ML 模型服务系统

## 附加资源

请参阅 `resources/recommended-reading.md` 获取：
- 官方 Kubernetes 文档链接
- 教程视频和课程
- 书籍和博客文章
- 练习环境（Katacoda、Play with Kubernetes）
- 社区资源

## 模块文件

```
mod-006-kubernetes-intro/
├── README.md (本文件)
├── lecture-notes/
│   ├── 01-k8s-architecture.md
│   ├── 02-deploying-apps.md
│   ├── 03-helm.md
│   └── 04-k8s-operations.md
├── exercises/
│   ├── exercise-01-first-deployment.md
│   ├── exercise-02-helm-chart.md
│   └── exercise-03-debugging.md
├── quizzes/
│   └── module-006-quiz.md
└── resources/
    ├── recommended-reading.md
    └── cluster-setup-guide.md
```

## 更新日志

- **2025-10-18**: 初始模块创建
- 内容与初级 AI 基础设施工程师角色要求对齐
- 练习设计用于动手技能发展
- 评估测验覆盖所有学习目标

---

**准备好开始了吗？** 从 `lecture-notes/01-k8s-architecture.md` 开始，了解 Kubernetes 如何在底层工作。

**有问题或反馈？** 联系 ai-infra-curriculum@joshua-ferguson.com
