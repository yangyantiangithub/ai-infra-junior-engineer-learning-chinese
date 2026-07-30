# Project 02: Kubernetes 模型服务

**时长：** 80 小时（全日制 2 周，兼职 3-4 周）
**难度：** 入门+
**前置条件：** Project 01（简单模型 API 部署）

---

## 概述

将 Project 01 中的基础 Flask/FastAPI 模型服务应用，转变为一个生产级、可扩展的 Kubernetes 部署。本项目教授容器编排基础知识、自动伸缩、负载均衡、滚动更新以及使用行业标准工具进行全方位监控。

### 真实场景

Project 01 中的简单模型 API 取得了成功！业务现在要求：
- **高可用性**：99.9% 正常运行时间保证
- **可伸缩性**：在高峰期每秒处理 1000+ 请求
- **零停机部署**：更新时服务不中断
- **可观测性**：实时指标和告警

Kubernetes——容器编排的事实标准——正是解决这些挑战的方案。

---

## 学习目标

完成本项目后，你将能够：

1. **使用 Deployment、Service 和 ConfigMap 将容器化应用部署到 Kubernetes**
2. **基于 CPU 和内存指标实现 Horizontal Pod Autoscaler (HPA) 自动伸缩**
3. **使用 Kubernetes Service 和 Ingress 控制器配置负载均衡**
4. **执行零停机的滚动更新并具备自动回滚能力**
5. **使用 Prometheus 和 Grafana 搭建监控基础设施**
6. **使用 Helm 打包应用以实现可重复、可配置的部署**
7. **使用存活探针和就绪探针实现健康检查**
8. **使用 ConfigMap 和 Secret 管理应用配置**
9. **使用 kubectl、日志和监控工具调试 Kubernetes 部署**
10. **通过 requests 和 limits 优化资源分配**

---

## 架构概览

### 高层架构

```
                    互联网
                       │
                       ▼
              ┌─────────────────┐
              │  LoadBalancer   │ ← 外部 IP
              │    Service      │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │     Ingress     │ ← NGINX 控制器
              │  /predict       │
              │  /health        │
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  ClusterIP      │ ← 内部负载均衡器
              │   Service       │
              └─────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
   ┌────────┐     ┌────────┐     ┌────────┐
   │ Pod 1  │     │ Pod 2  │     │ Pod 3  │
   │ API v1 │     │ API v1 │     │ API v1 │
   └────────┘     └────────┘     └────────┘
        │              │              │
        └──────────────┴──────────────┘
                       │
                       ▼
              ┌─────────────────┐
              │      HPA        │ ← 自动伸缩器
              │  最小值: 3      │
              │  最大值: 10     │
              │  目标: 70% CPU  │
              └─────────────────┘
```

### 监控架构

```
┌─────────────────────────────────────┐
│           Prometheus                │
│  ┌─────────────────────────────┐   │
│  │ 抓取指标来源：               │   │
│  │ • Pod（通过 /metrics）       │   │
│  │ • Kubernetes API            │   │
│  │ • Node exporter             │   │
│  └─────────────────────────────┘   │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│            Grafana                  │
│  ┌─────────────────────────────┐   │
│  │ 仪表盘：                    │   │
│  │ • Pod 数量与健康状态         │   │
│  │ • CPU/内存使用率            │   │
│  │ • 请求速率与延迟            │   │
│  │ • 错误率                    │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

---

## 技术栈

### 核心技术

| 组件 | 技术 | 用途 |
|-----------|------------|---------|
| **编排** | Kubernetes 1.28+ | 容器编排与调度 |
| **本地集群** | Minikube / Kind | 本地开发与测试 |
| **云平台** | EKS / GKE / AKS | 生产 Kubernetes 集群 |
| **包管理器** | Helm 3.12+ | 应用模板化与部署 |
| **监控** | Prometheus + Grafana | 指标采集与可视化 |
| **Ingress** | NGINX Ingress 控制器 | HTTP 路由与负载均衡 |
| **CLI 工具** | kubectl, helm | 集群管理 |

### 推荐工具

- **K9s** - Kubernetes 集群管理的终端 UI
- **Lens** - 可视化集群探索的 Kubernetes IDE
- **kubectx/kubens** - 快速上下文和命名空间切换
- **stern** - 多 Pod 日志跟踪
- **k6** - 负载测试工具

---

## 项目结构

```
project-02-kubernetes-serving/
├── README.md                          # 本文件
├── requirements.md                    # 详细需求
├── architecture.md                    # 架构文档
├── .env.example                       # 环境配置模板
│
├── src/
│   ├── app.py                        # 从 Project 01 增强的 API（待实现）
│   ├── model.py                      # 模型加载逻辑（待实现）
│   ├── metrics.py                    # Prometheus 指标（待实现）
│   └── requirements.txt              # Python 依赖
│
├── kubernetes/
│   ├── deployment.yaml               # K8s Deployment 清单（待实现）
│   ├── service.yaml                  # K8s Service 清单（待实现）
│   ├── ingress.yaml                  # Ingress 配置（待实现）
│   ├── hpa.yaml                      # Horizontal Pod Autoscaler（待实现）
│   ├── configmap.yaml                # 应用配置（待实现）
│   └── secrets.yaml.example          # Secrets 模板
│
├── monitoring/
│   ├── servicemonitor.yaml           # Prometheus ServiceMonitor（待实现）
│   └── grafana-dashboard.json        # Grafana 仪表盘模板
│
├── tests/
│   ├── test_k8s.py                   # Kubernetes 部署测试（待实现）
│   ├── test_api.py                   # API 集成测试（待实现）
│   └── load-test.js                  # K6 负载测试脚本（待实现）
│
└── docs/
    ├── SETUP.md                      # 设置说明
    ├── OPERATIONS.md                 # 运维手册
    └── TROUBLESHOOTING.md            # 常见问题与解决方案
```

---

## 需求

### 功能需求

#### FR-1: Kubernetes 部署
- 将模型 API 部署为 3 个副本的 Kubernetes Deployment
- 配置资源 requests（CPU: 500m，内存: 1Gi）和 limits（CPU: 1000m，内存: 2Gi）
- 实现检查 `/health` 端点的存活探针
- 实现就绪探针，确保模型在接收流量前已加载
- 使用 ConfigMap 管理应用配置（模型名称、日志级别）

#### FR-2: 服务与负载均衡
- 创建用于内部访问的 ClusterIP Service
- 创建用于外部访问的 LoadBalancer Service
- 实现基于路径路由的 Ingress HTTP 路由
- 验证跨所有 Pod 的负载分布

#### FR-3: 自动伸缩
- 配置 HPA 在 3 到 10 个 Pod 之间伸缩
- 设置目标 CPU 利用率为 70%
- 定义扩容和缩容策略
- 在有负载的情况下测试自动伸缩

#### FR-4: 滚动更新
- 实现 RollingUpdate 策略
- 配置 maxSurge=1 和 maxUnavailable=0 以实现零停机
- 测试从 v1.0 到 v1.1 的更新
- 实现回滚能力

#### FR-5: 监控与可观测性
- 部署 Prometheus 用于指标采集
- 部署 Grafana 用于可视化
- 创建展示 Pod 数量、CPU、内存、请求速率的仪表盘
- 配置高错误率或 Pod 故障的告警

### 非功能需求

#### NFR-1: 性能
- 集群整体支持 1000+ 请求/秒
- 负载下 P99 延迟 < 500ms
- 自动伸缩在 2 分钟内响应

#### NFR-2: 可靠性
- 99.9% 正常运行时间（每月停机时间 < 43 分钟）
- 部署期间零停机
- 30 秒内自动恢复 Pod

#### NFR-3: 可伸缩性
- 支持 3-10 个 Pod 副本
- 优雅处理 10 倍流量突增
- 资源高效的伸缩

#### NFR-4: 安全性
- Secrets 存储在 Kubernetes Secrets 中（而非 ConfigMap）
- 应用 Pod 安全标准
- 限制 Pod 通信的网络策略
- 配置最小权限 RBAC

---

## 开始使用

### 前置条件

在开始本项目之前，请确保你已具备：

1. **完成 Project 01**：理解 Flask/FastAPI 模型服务
2. **已安装 Docker**：用于构建容器镜像
3. **基本的 Linux/Unix 知识**：命令行熟练度
4. **Git**：版本控制基础
5. **云账户**（可选）：用于生产部署的 AWS、GCP 或 Azure

### 设置步骤

1. **安装 Kubernetes 工具**
   ```bash
   # 安装 kubectl
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

   # 安装 Minikube
   curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
   sudo install minikube-linux-amd64 /usr/local/bin/minikube

   # 安装 Helm
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

2. **启动本地 Kubernetes 集群**
   ```bash
   # 以充足资源启动 Minikube
   minikube start --cpus=4 --memory=8192 --driver=docker

   # 验证安装
   kubectl version
   kubectl get nodes
   ```

3. **启用所需的插件**
   ```bash
   # 启用 metrics server 以供 HPA 使用
   minikube addons enable metrics-server

   # 启用 ingress 控制器
   minikube addons enable ingress
   ```

4. **克隆项目并安装依赖**
   ```bash
   cd project-02-kubernetes-serving

   # 安装 Python 依赖
   pip install -r src/requirements.txt

   # 复制环境模板
   cp .env.example .env
   ```

5. **构建 Docker 镜像**
   ```bash
   # 构建镜像
   docker build -t model-api:v1.0 .

   # 对于 Minikube，使用其 Docker 守护进程
   eval $(minikube docker-env)
   docker build -t model-api:v1.0 .
   ```

---

## 实施阶段

### 阶段 1：本地 Kubernetes 搭建（20 小时）
- 设置 Minikube 集群
- 创建 Kubernetes 清单（Deployment、Service、ConfigMap、HPA）
- 在本地部署应用
- 测试健康检查和负载均衡

### 阶段 2：监控搭建（15 小时）
- 使用 Helm 安装 Prometheus 和 Grafana
- 配置应用指标端点
- 创建 Grafana 仪表盘
- 设置基础告警

### 阶段 3：Helm Chart 创建（20 小时）
- 初始化 Helm chart 结构
- 创建模板化清单
- 定义包含配置选项的 values.yaml
- 测试 chart 安装和升级

### 阶段 4：云部署（15 小时）
- 创建云 Kubernetes 集群（EKS/GKE/AKS）
- 将应用部署到云上
- 配置 DNS 和 SSL/TLS
- 进行负载测试

### 阶段 5：测试与文档（10 小时）
- 编写自动化部署测试
- 创建运维手册
- 记录故障排查流程
- 准备演示

---

## 交付物

### 必需交付物

1. **Kubernetes 清单**
   - 可工作的 Deployment、Service、ConfigMap、HPA、Ingress
   - 正确配置的健康检查
   - 资源 requests 和 limits

2. **Helm Chart**
   - 包含模板的完整 chart
   - 全面的 values.yaml
   - 包含安装说明的 README

3. **监控设置**
   - Prometheus 部署
   - Grafana 仪表盘（导出的 JSON）
   - 告警规则配置

4. **文档**
   - 架构图
   - 部署指南
   - 运维手册
   - 故障排查指南

5. **测试套件**
   - Kubernetes 部署测试
   - API 集成测试
   - 负载测试脚本和结果

6. **演示**
   - 10 分钟视频或现场演示，展示：
     - 部署过程
     - 自动伸缩实战
     - 零停机滚动更新
     - 监控仪表盘

---

## 评估标准

你的项目将按以下标准评估：

### Kubernetes 配置（30%）
- 配置正确的 Deployment 及健康检查
- 正确的 Service 和 Ingress 设置
- 带有适当阈值的自动伸缩功能
- 合理的资源管理

### 监控与可观测性（20%）
- Prometheus 指标采集
- 信息丰富的 Grafana 仪表盘
- 有意义的告警
- 应用级别指标

### Helm Chart（20%）
- 结构良好的 chart，包含可复用模板
- 全面且有文档的 values.yaml
- 对不同环境的灵活性
- 清晰的文档

### 运维（15%）
- 顺畅的滚动更新
- 快速回滚能力
- 响应迅速的自动伸缩
- 清晰的运维流程

### 性能（15%）
- 满足负载下的延迟要求
- 高效的资源利用
- 合理的伸缩行为
- 处理流量突增

**合格分数：** 70/100

---

## 资源

### 官方文档
- [Kubernetes 文档](https://kubernetes.io/docs/)
- [Helm 文档](https://helm.sh/docs/)
- [Prometheus 文档](https://prometheus.io/docs/)
- [Grafana 文档](https://grafana.com/docs/)

### 推荐教程
- [Kubernetes 基础教程](https://kubernetes.io/docs/tutorials/kubernetes-basics/)
- [HPA 演练](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)
- [Helm 快速入门](https://helm.sh/docs/intro/quickstart/)

### 书籍
- 《Kubernetes Up & Running》by Kelsey Hightower
- 《The Kubernetes Book》by Nigel Poulton

### 工具
- [K9s](https://k9scli.io/) - Kubernetes 终端 UI
- [Lens](https://k8slens.dev/) - Kubernetes IDE
- [k6](https://k6.io/) - 负载测试工具

---

## 常见陷阱

1. **没有资源限制**：Pod 可能耗尽所有节点资源导致崩溃
2. **健康检查不当**：就绪探针过于激进或完全缺失
3. **错误的 HPA 指标**：使用不合适的指标或阈值进行伸缩
4. **没有滚动更新策略**：更新导致停机
5. **硬编码值**：不使用 ConfigMap 导致更新困难
6. **缺少监控**：没有指标就无法调试问题
7. **没有命名空间隔离**：资源混在一起使管理复杂化

## 专家建议

1. **使用命名空间**：隔离开发/预发布/生产环境
2. **标记所有资源**：使资源选择与调试更加容易
3. **先在本地测试**：在昂贵的云部署之前使用 Minikube
4. **尽早使用 Helm**：避免为复杂应用管理原始 YAML
5. **从第一天开始监控**：在问题出现前就搭建 Prometheus + Grafana
6. **边走边记**：未来的你会感激现在的你
7. **彻底进行负载测试**：在生产流量之前找到你的极限

---

## 支持

如果遇到问题：

1. **查看 TROUBLESHOOTING.md** 了解常见问题
2. **检查 Kubernetes 日志**：`kubectl logs <pod-name>`
3. **描述资源**：`kubectl describe pod <pod-name>`
4. **检查事件**：`kubectl get events --sort-by='.lastTimestamp'`
5. **使用监控仪表盘**：在 Grafana 中查看异常

---

## 后续步骤

完成本项目后，你可以继续：

- **Project 03**：高级监控与可观测性
- **Project 04**：模型部署的 CI/CD 流水线
- **Project 05**：带流量路由的多模型服务

---

**项目版本：** 1.0
**最后更新：** 2025 年 10 月
**维护者：** AI 基础设施课程团队
