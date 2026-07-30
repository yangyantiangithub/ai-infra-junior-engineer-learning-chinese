# Lecture 01: Kubernetes 架构

## 目录
1. [引言](#引言)
2. [什么是 Kubernetes](#什么是-kubernetes)
3. [Kubernetes 架构概览](#kubernetes-架构概览)
4. [控制平面组件](#控制平面组件)
5. [工作节点组件](#工作节点组件)
6. [Kubernetes 对象和 API](#kubernetes-对象和-api)
7. [集群网络](#集群网络)
8. [Kubernetes 如何调度工作负载](#kubernetes-如何调度工作负载)
9. [高可用和容错](#高可用和容错)
10. [Kubernetes 在 AI/ML 基础设施中的应用](#kubernetes-在-aiml-基础设施中的应用)
11. [总结和关键要点](#总结和关键要点)

## 引言

Kubernetes（通常简写为 K8s，其中 "8" 代表 "K" 和 "s" 之间的 8 个字母）是世界上采用最广泛的容器编排平台。最初由 Google 基于其内部 Borg 系统开发，Kubernetes 于 2014 年开源，并于 2015 年捐赠给 Cloud Native Computing Foundation (CNCF)。

理解 Kubernetes 架构对于处理现代云原生应用和 AI/ML 基础设施至关重要。本讲座深入介绍 Kubernetes 的底层工作原理，帮助你有效部署、管理和排查容器化应用。

### 学习目标

完成本讲座后，你将：
- 理解整体 Kubernetes 架构和设计理念
- 识别并解释每个控制平面组件的用途
- 理解工作节点组件及其角色
- 掌握 Kubernetes API 和对象模型
- 理解 Kubernetes 网络工作原理
- 理解调度和生命周期管理流程
- 将这些知识应用于 AI/ML 基础设施场景

### 前置要求
- Module 003: Docker 基础
- 分布式系统的基本理解
- 熟悉 YAML 语法
- Linux 命令行能力

## 什么是 Kubernetes

### 核心目的

Kubernetes 是一个**容器编排平台**，自动化容器化应用的部署、扩缩和管理。它解决了跨多台机器大规模运行容器的挑战，并提供：

- **自动化部署**和回滚
- **服务发现**和负载均衡
- **自愈**能力
- **水平扩缩**
- **Secret 和配置**管理
- **存储编排**
- **批处理执行**和作业管理

### Kubernetes 解决的问题

在 Kubernetes 之前，大规模部署容器化应用需要：
- 跨服务器手动放置容器
- 用于健康监控和重启的自定义脚本
- 复杂的负载均衡配置
- 根据需求手动扩缩
- 困难的发布和回滚
- 没有标准的配置管理方式

Kubernetes 提供**声明式**方法：你告诉 Kubernetes 应用的期望状态，它持续工作以维持该状态。

### 声明式与命令式

**命令式方法**（传统方式）：
```bash
# 手动部署应用的步骤
ssh server1
docker run -d --name app1 myapp:v1
ssh server2
docker run -d --name app2 myapp:v1
# 配置负载均衡器...
# 设置健康检查...
```

**声明式方法**（Kubernetes）：
```yaml
# 用 YAML 描述期望状态
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: myapp:v1
```

Kubernetes 读取这个 manifest 并使其成为现实，持续确保期望状态与实际状态匹配。

### AI/ML 工作负载的关键优势

1. **资源管理**：高效分配 GPU/CPU 资源用于训练和推理
2. **可扩展性**：根据请求负载自动扩缩模型服务
3. **可移植性**：在任何云或本地运行相同的 ML 工作负载
4. **实验**：快速创建/销毁 ML 实验环境
5. **作业调度**：运行批处理训练作业、超参数调优等
6. **存储**：使用持久卷管理数据和模型存储

## Kubernetes 架构概览

Kubernetes 遵循**主从**（也称为控制平面和工作节点）架构。Kubernetes 集群由以下部分组成：

1. **控制平面**（主节点）：对集群做出全局决策
2. **工作节点**：运行实际的应用工作负载
3. **插件**：DNS、监控等可选组件

### 高层架构图

```
                    Kubernetes Cluster
    ┌──────────────────────────────────────────────┐
    │                                              │
    │  ┌────────────────────────────────────┐     │
    │  │        Control Plane               │     │
    │  │  ┌──────────────────────────────┐  │     │
    │  │  │   API Server (kube-apiserver)│  │     │
    │  │  └────────────┬─────────────────┘  │     │
    │  │               │                     │     │
    │  │  ┌────────────┴─────────────┐      │     │
    │  │  │                          │      │     │
    │  │  ▼                          ▼      │     │
    │  │ ┌──────────┐          ┌─────────┐ │     │
    │  │ │Scheduler │          │  etcd   │ │     │
    │  │ │(kube-    │          │ (key-   │ │     │
    │  │ │scheduler)│          │  value  │ │     │
    │  │ └──────────┘          │  store) │ │     │
    │  │                       └─────────┘ │     │
    │  │ ┌──────────────────────────────┐  │     │
    │  │ │ Controller Manager           │  │     │
    │  │ │ (kube-controller-manager)    │  │     │
    │  │ └──────────────────────────────┘  │     │
    │  │                                    │     │
    │  │ ┌──────────────────────────────┐  │     │
    │  │ │ Cloud Controller Manager     │  │     │
    │  │ │ (cloud-specific logic)       │  │     │
    │  │ └──────────────────────────────┘  │     │
    │  └────────────────────────────────────┘     │
    │                                              │
    │  ┌────────────────────────────────────┐     │
    │  │         Worker Nodes               │     │
    │  │                                    │     │
    │  │  Node 1      Node 2      Node 3   │     │
    │  │ ┌──────┐    ┌──────┐    ┌──────┐  │     │
    │  │ │kubelet│    │kubelet│    │kubelet│  │     │
    │  │ └───┬──┘    └───┬──┘    └───┬──┘  │     │
    │  │     │           │           │      │     │
    │  │ ┌───▼───┐   ┌──▼────┐   ┌──▼───┐  │     │
    │  │ │kube-  │   │kube-  │   │kube- │  │     │
    │  │ │proxy  │   │proxy  │   │proxy │  │     │
    │  │ └───────┘   └───────┘   └──────┘  │     │
    │  │     │           │           │      │     │
    │  │ ┌───▼───────────▼───────────▼───┐  │     │
    │  │ │  Container Runtime (Docker,  │  │     │
    │  │ │  containerd, CRI-O)          │  │     │
    │  │ └──────────────────────────────┘  │     │
    │  │     │           │           │      │     │
    │  │ ┌───▼───┐   ┌──▼────┐   ┌──▼───┐  │     │
    │  │ │ Pods  │   │ Pods  │   │ Pods │  │     │
    │  │ │░░░░░░░│   │░░░░░░░│   │░░░░░░│  │     │
    │  │ └───────┘   └───────┘   └──────┘  │     │
    │  └────────────────────────────────────┘     │
    │                                              │
    └──────────────────────────────────────────────┘
```

### 组件通信

所有组件都通过 **API Server** 通信。这是一个关键的架构原则：

- 工作节点之间不直接通信
- Scheduler 不直接告诉节点要运行什么
- Controller 监控 API Server 的变化
- 所有通信都通过 API Server 的 RESTful API

这种集中式通信模型提供：
- **一致性**：单一数据源（通过 API Server 访问 etcd）
- **安全性**：统一进行认证和授权
- **可审计性**：所有操作都可记录
- **可扩展性**：易于添加监控 API 的新组件

## 控制平面组件

控制平面是 Kubernetes 集群的"大脑"。它对集群做出全局决策（如调度），并检测和响应集群事件（如在 Deployment 的 replicas 字段未满足时启动新的 Pod）。

### 1. API Server (kube-apiserver)

**用途**：Kubernetes 控制平面的前端。所有通信都通过 API Server。

**关键职责**：
- 暴露 Kubernetes API（RESTful HTTP API）
- 认证和授权请求
- 验证和处理 API 请求
- 使用集群状态更新 etcd
- 作为所有组件的通信中心

**工作原理**：
```
User/kubectl → API Server → Validate → Authenticate → Authorize →
               Admission Controllers → Write to etcd → Return response
```

**交互示例**：
```bash
# 当你运行这个命令时：
kubectl create deployment nginx --image=nginx

# API Server：
# 1. 认证你（确认你的身份）
# 2. 授权你（确认你有权创建 Deployment）
# 3. 验证请求（检查 Deployment 规范是否有效）
# 4. 运行准入控制器（变更/验证对象）
# 5. 将 Deployment 存储在 etcd 中
# 6. 向 kubectl 返回成功
```

**关键特性**：
- **无状态**：本身不存储任何数据（使用 etcd）
- **水平可扩展**：可以运行多个副本以实现高可用
- **RESTful**：使用标准 HTTP 方法（GET、POST、PUT、DELETE、PATCH）
- **版本化 API**：支持多个 API 版本（v1、v1beta1 等）

**API 组和版本**：
```yaml
# 核心 API 组（旧版，无组名）
apiVersion: v1
kind: Pod

# Apps API 组
apiVersion: apps/v1
kind: Deployment

# Batch API 组
apiVersion: batch/v1
kind: Job
```

### 2. etcd

**用途**：一致且高可用的键值存储，用作 Kubernetes 所有集群数据的后端存储。

**关键职责**：
- 存储整个集群状态
- 使用 Raft 共识算法提供一致性
- 支持 watch 操作实现实时更新
- 支持集群配置存储

**etcd 中存储的内容**：
- 所有 Kubernetes 对象（Pod、Deployment、Service 等）
- 集群配置
- Secret 和 ConfigMap
- 资源配额和限制
- 网络策略
- Service Account token

**示例 etcd 结构**：
```
/registry/
  /pods/
    /default/
      /nginx-pod
  /deployments/
    /default/
      /nginx-deployment
  /services/
    /default/
      /nginx-service
  /secrets/
    /default/
      /my-secret
```

**关键特性**：
- **强一致性**：使用 Raft 共识（需要法定人数）
- **高可用**：通常运行 3 或 5 个实例
- **Watch 能力**：组件可以监听变化
- **快速**：针对频繁的小读写操作进行了优化

**为什么使用 Raft 共识？**：
Raft 确保所有 etcd 实例就集群状态达成一致，即使在网络分区或节点故障的情况下。它需要多数（法定人数）来做出决策：
- 3 个节点：可以容忍 1 个故障
- 5 个节点：可以容忍 2 个故障
- 7 个节点：可以容忍 3 个故障（很少需要）

**安全注意事项**：
etcd 包含所有集群 Secret（即使是加密的）。保护 etcd 至关重要：
- 使用 TLS 进行客户端-服务器通信
- 使用 TLS 进行对等通信
- 启用静态加密
- 限制网络访问
- 定期备份

### 3. Scheduler (kube-scheduler)

**用途**：监控新创建的 Pod 并根据资源需求、约束和策略将它们分配到工作节点。

**关键职责**：
- 为每个 Pod 选择最佳节点
- 考虑资源需求（CPU、内存、GPU）
- 遵守约束（节点选择器、亲和性、taint/toleration）
- 跨节点平衡工作负载
- 处理优先级和抢占

**调度过程**：
```
1. 监控 API Server 中未调度的 Pod（spec.nodeName == ""）
2. 过滤阶段：排除不合适的节点
   - 资源不足？
   - 节点选择器不匹配？
   - Taint/toleration 冲突？
   - Pod 亲和性/反亲和性规则？
3. 评分阶段：对剩余节点进行排名
   - 资源平衡（分散工作负载）
   - 节点亲和性偏好
   - Pod 间亲和性
   - 镜像本地性
4. 将 Pod 绑定到最高评分节点
5. 更新 API Server（设置 spec.nodeName）
```

**示例调度决策**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  containers:
  - name: trainer
    image: pytorch:latest
    resources:
      requests:
        memory: "16Gi"
        cpu: "4"
        nvidia.com/gpu: "1"  # 需要 GPU
  nodeSelector:
    gpu-type: "nvidia-v100"  # 必须匹配节点标签
```

**调度决策过程**：
1. **过滤**：没有 V100 GPU 的节点 → 排除
2. **过滤**：没有 16Gi 内存的节点 → 排除
3. **过滤**：没有可用 GPU 的节点 → 排除
4. **评分**：剩余节点按以下标准排名：
   - 可用资源
   - 工作负载分布
   - 镜像是否已存在？
5. **绑定**：Pod 分配给最高评分节点

**高级调度**：
- **亲和性/反亲和性**：将 Pod 放置在一起或分开
- **Taint 和 Toleration**：将节点专用于特定工作负载
- **优先级类**：必要时抢占低优先级 Pod
- **自定义 Scheduler**：为特殊需求编写自己的 scheduler

**AI/ML 示例 - GPU 节点选择**：
```yaml
# 对 GPU 节点添加 taint，使只有 GPU 工作负载在其上运行
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule

# ML Pod 容忍 taint
apiVersion: v1
kind: Pod
metadata:
  name: ml-inference
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: model-server
    image: tensorflow/serving:latest-gpu
    resources:
      limits:
        nvidia.com/gpu: 1
```

### 4. Controller Manager (kube-controller-manager)

**用途**：运行调节集群状态的控制器进程，持续努力将当前状态移动到期望状态。

**关键职责**：
- 在单个进程中运行多个控制器
- 通过 API Server 监控变化
- 做出更改使实际状态与期望状态匹配
- 处理节点故障、副本、端点、service account

**核心控制器**：

**a) Node Controller**：
- 监控节点健康
- 如果节点停止心跳，将其标记为不健康
- 从故障节点驱逐 Pod

```
每 10 秒：检查节点心跳
如果错过 40 秒：将节点标记为 Unknown
如果 Unknown 持续 5 分钟：从节点驱逐 Pod
```

**b) Replication Controller / ReplicaSet Controller**：
- 确保运行正确数量的 Pod 副本
- 创建/删除 Pod 以匹配期望的副本数

```yaml
# 期望状态：3 个副本
spec:
  replicas: 3

# Controller 持续确保：
# - 如果存在 2 个副本 → 创建 1 个
# - 如果存在 4 个副本 → 删除 1 个
# - 如果存在 3 个副本 → 不做任何操作
```

**c) Endpoints Controller**：
- 填充 Endpoints 对象（将 Service 链接到 Pod）
- 监控 Service 和 Pod
- 在 Pod 添加/删除时更新端点

```
Service "web" with selector "app=nginx"
  → Controller 找到所有带有标签 "app=nginx" 的 Pod
  → 使用 Pod IP 创建 Endpoints 对象
  → kube-proxy 使用 Endpoints 进行负载均衡
```

**d) Service Account & Token Controllers**：
- 为新 namespace 创建默认 service account
- 为 service account 生成 API 访问 token

**e) Deployment Controller**：
- 管理 Deployment 对象
- 创建/更新 ReplicaSet
- 处理滚动更新和回滚

```yaml
# 用户更新 deployment image
kubectl set image deployment/nginx nginx=nginx:1.20

# Deployment controller：
# 1. 使用 nginx:1.20 创建新的 ReplicaSet
# 2. 逐步扩展新的 ReplicaSet
# 3. 逐步缩减旧的 ReplicaSet
# 4. 监控 rollout 健康状态
# 5. 如果更新失败则回滚
```

**f) Job Controller**：
- 管理 Job 对象
- 创建 Pod 执行任务直到完成
- 跟踪成功/失败的完成次数

**g) CronJob Controller**：
- 按计划创建 Job
- 管理 Job 历史和清理

**Controller 模式**（协调循环）：
```go
// 控制器工作原理的伪代码
for {
  desiredState = getFromAPIServer()
  currentState = getFromCluster()

  if currentState != desiredState {
    makeChangesToReachDesiredState()
  }

  sleep(reconciliationPeriod)
}
```

这种**电平触发**（而非边沿触发）方法意味着：
- Controllers 定期检查状态（对错过的事件具有弹性）
- 如果手动更改了状态，controllers 会修复它们
- 自愈行为

### 5. Cloud Controller Manager (cloud-controller-manager)

**用途**：嵌入特定于云的控制逻辑，允许 Kubernetes 与底层云提供商的 API 交互。

**关键职责**：
- 管理云特定资源（负载均衡器、卷、路由）
- 从核心 Kubernetes 中移除云提供商代码
- 支持云提供商插件

**云特定 Controllers**：

**a) Node Controller**：
- 在节点停止响应后检查云中是否已删除该节点
- 使用云提供商特定信息更新节点（区域、实例类型）

**b) Route Controller**：
- 在云中设置路由，使不同节点上的容器可以通信

**c) Service Controller**：
- 为 LoadBalancer 类型的 Service 创建/删除云负载均衡器

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ml-api
spec:
  type: LoadBalancer  # Cloud controller creates cloud LB
  selector:
    app: ml-api
  ports:
  - port: 80
    targetPort: 8080
```

**d) Volume Controller**：
- 创建/挂载/附加云卷（EBS、Persistent Disk 等）

**云提供商示例**：
- AWS：ELB/ALB/NLB 用于服务，EBS 用于卷
- GCP：Cloud Load Balancer，Persistent Disk
- Azure：Azure Load Balancer，Azure Disk
- OpenStack：Neutron LBaaS，Cinder

## 工作节点组件

工作节点是运行容器化应用的机器（物理或虚拟）。每个工作节点运行维护 Pod 并提供 Kubernetes 运行时环境所需的组件。

### 节点结构

```
Worker Node
┌─────────────────────────────────────────────┐
│                                             │
│  ┌────────────────────────────────────┐    │
│  │         kubelet                    │    │
│  │  (Node agent, Pod lifecycle)       │    │
│  └────────┬───────────────────────────┘    │
│           │                                 │
│           ▼                                 │
│  ┌────────────────────────────────────┐    │
│  │    Container Runtime               │    │
│  │  (Docker, containerd, CRI-O)       │    │
│  └────────┬───────────────────────────┘    │
│           │                                 │
│           ▼                                 │
│  ┌────────────────────────────────────┐    │
│  │         Pods                       │    │
│  │  ┌──────────┐  ┌──────────┐       │    │
│  │  │Container │  │Container │       │    │
│  │  │    1     │  │    2     │       │    │
│  │  └──────────┘  └──────────┘       │    │
│  └────────────────────────────────────┘    │
│                                             │
│  ┌────────────────────────────────────┐    │
│  │       kube-proxy                   │    │
│  │  (Network proxy, load balancing)   │    │
│  └────────────────────────────────────┘    │
│                                             │
│  ┌────────────────────────────────────┐    │
│  │    Optional: Device Plugins        │    │
│  │  (GPU, FPGA, InfiniBand, etc.)     │    │
│  └────────────────────────────────────┘    │
│                                             │
└─────────────────────────────────────────────┘
```

### 1. kubelet

**用途**：在每个工作节点上运行的主要节点代理。负责确保容器按规范在 Pod 中运行。

**关键职责**：
- 向 API Server 注册节点
- 监控 API Server 中分配给其节点的 Pod
- 通过 container runtime 启动/停止容器
- 监控 Pod 和容器健康
- 向 API Server 报告节点和 Pod 状态
- 执行 liveness 和 readiness 探针
- 挂载卷

**kubelet 工作流**：
```
1. 向 API Server 注册（发送节点信息）
2. 监控 spec.nodeName == <此节点> 的 Pod
3. 对每个 Pod：
   a. 拉取容器镜像
   b. 通过 CRI (Container Runtime Interface) 创建容器
   c. 挂载卷
   d. 设置网络
   e. 启动容器
   f. 监控健康探针
   g. 向 API Server 报告状态
4. 如果 Pod 应被删除：
   a. 停止容器
   b. 清理卷
   c. 从节点移除 Pod
```

**健康探针**：

kubelet 执行三种类型的探针：

**Liveness Probe**：容器是否仍在运行？
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 10
# 如果失败，kubelet 重启容器
```

**Readiness Probe**：容器是否准备好接收流量？
```yaml
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
# 如果失败，Pod 从 Service endpoints 中移除
```

**Startup Probe**：容器是否已成功启动？
```yaml
startupProbe:
  exec:
    command: ["/bin/sh", "-c", "cat /app/initialized"]
  failureThreshold: 30
  periodSeconds: 10
# 保护启动慢的容器免受 liveness probe 误杀
```

**kubelet 管理的 Pod 生命周期**：
```
Pending → Running → Succeeded/Failed
           ↓
         CrashLoopBackOff (如果反复失败)
```

**资源监控**：
kubelet 监控：
- 每个容器的 CPU 使用率
- 每个容器的内存使用率
- 文件系统使用情况
- 网络 I/O

这些数据通过 Metrics API 暴露，供 horizontal pod autoscaler (HPA) 使用。

### 2. Container Runtime

**用途**：负责运行容器的软件。Kubernetes 支持任何符合 CRI 的运行时。

**支持的运行时**：
- **containerd**：轻量级、行业标准运行时（CNCF 毕业项目）
- **CRI-O**：专为 Kubernetes 设计的轻量级运行时
- **Docker Engine**：通过 cri-dockerd 适配器（K8s 1.24+ 已弃用）

**Container Runtime Interface (CRI)**：
Kubernetes 使用 CRI 与 container runtime 通信，提供插件接口：

```
kubelet → CRI API → Container Runtime → Containers
```

**CRI 操作**：
- `RunPodSandbox`：创建 Pod sandbox（网络命名空间）
- `CreateContainer`：创建容器
- `StartContainer`：启动容器
- `StopContainer`：停止容器
- `RemoveContainer`：移除容器
- `ListContainers`：列出容器
- `PullImage`：拉取容器镜像

**为什么有多种运行时？**：
不同的用例受益于不同的运行时：
- **containerd**：通用、轻量级
- **CRI-O**：最小化，仅设计用于 Kubernetes
- **gVisor**：增强安全性（沙箱化容器）
- **Kata Containers**：类似 VM 的隔离，具有容器的用户体验

**镜像管理**：
Container runtime 处理：
- 从仓库拉取镜像
- 镜像缓存
- 镜像层和存储
- 镜像垃圾回收

### 3. kube-proxy

**用途**：在每个节点上运行的网络代理，实现 Kubernetes Service 网络。

**关键职责**：
- 维护节点上的网络规则
- 启用 Service 抽象（ClusterIP）
- 实现到 Pod 后端的负载均衡
- 支持不同的 Service 类型（ClusterIP、NodePort、LoadBalancer）

**Service 工作原理**（配合 kube-proxy）：

```
Client → Service IP:Port (虚拟，非真实)
            ↓
         kube-proxy (拦截流量)
            ↓
         负载均衡到 Pod IP
            ↓
         Pod 1, Pod 2, or Pod 3 (后端)
```

**代理模式**：

**a) iptables 模式**（默认）：
- 使用 Linux iptables 规则进行路由
- 随机选择后端 Pod
- 比 userspace 模式开销低
- 如果选中的 Pod 无响应，不会重试

```bash
# kube-proxy 创建的 iptables 规则示例
iptables -t nat -A KUBE-SERVICES \
  -d 10.96.0.1/32 -p tcp --dport 80 \
  -j KUBE-SVC-XYZABC

iptables -t nat -A KUBE-SVC-XYZABC \
  -m statistic --mode random --probability 0.33 \
  -j KUBE-SEP-POD1

iptables -t nat -A KUBE-SVC-XYZABC \
  -m statistic --mode random --probability 0.50 \
  -j KUBE-SEP-POD2

iptables -t nat -A KUBE-SVC-XYZABC \
  -j KUBE-SEP-POD3
```

**b) ipvs 模式**（IP Virtual Server）：
- 使用 Linux IPVS（内核级负载均衡器）
- 对于大量 Service 更高效
- 更好的负载均衡算法（轮询、最少连接等）
- 更好的性能

**c) userspace 模式**（旧版）：
- kube-proxy 充当实际代理
- 较高开销
- 可以重试失败的后端

**kube-proxy 实现的 Service 类型**：

1. **ClusterIP**：集群内部 IP（默认）
2. **NodePort**：在每个节点 IP 的静态端口上暴露服务
3. **LoadBalancer**：使用云提供商的负载均衡器
4. **ExternalName**：映射到 DNS 名称

**NodePort 流程示例**：
```
External Client → Node IP:30001 (NodePort)
                      ↓
                  kube-proxy rule
                      ↓
                  Service ClusterIP:80
                      ↓
                  负载均衡到 Pod
                      ↓
                  Pod:8080
```

### 4. Device Plugins（可选）

**用途**：使 Kubernetes 能够管理专用硬件资源，如 GPU、FPGA、高性能网卡等。

**Device Plugin 工作原理**：
```
Hardware → Device Plugin → kubelet → Scheduler
                                        ↓
                                   Pod with resource request
```

**GPU 示例**（NVIDIA Device Plugin）：
```yaml
# 节点通告 GPU 资源
kubectl describe node gpu-node-1

Capacity:
  nvidia.com/gpu: 4

# Pod 请求 GPU
apiVersion: v1
kind: Pod
metadata:
  name: gpu-job
spec:
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.0-base
    resources:
      limits:
        nvidia.com/gpu: 1
```

**常见的 Device Plugin**：
- **NVIDIA GPU Plugin**：用于 NVIDIA GPU
- **AMD GPU Plugin**：用于 AMD GPU
- **Intel QAT Plugin**：用于 Intel QuickAssist
- **RDMA Plugin**：用于高性能网络
- **FPGA Plugin**：用于 FPGA

**对 AI/ML 至关重要**：
Device plugins 对于 AI 基础设施至关重要，它们支持：
- 为训练和推理分配 GPU
- 多 GPU 工作负载
- GPU 共享（使用 NVIDIA MPS 等工具）
- 资源跟踪和核算

## Kubernetes 对象和 API

### 理解 Kubernetes 对象

Kubernetes 对象是 Kubernetes 系统中的持久化实体，代表集群的状态：
- **哪些**容器化应用正在运行
- **在哪里**运行（哪些节点）
- **多少个**副本存在
- 关于重启、更新和扩缩的**策略**

### 对象规范

每个 Kubernetes 对象有两个重要字段：

**spec**：期望状态（你想要什么）
**status**：当前状态（实际存在什么）

Kubernetes 持续努力使 **status** 匹配 **spec**。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:                    # 期望状态
  replicas: 3           # 想要 3 个 Pod
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
status:                  # 当前状态（只读，由系统设置）
  replicas: 3           # 当前 3 个 Pod
  readyReplicas: 3      # 3 个就绪
  conditions:
  - type: Available
    status: "True"
```

### 必需字段

每个 Kubernetes 对象 manifest 需要：

1. **apiVersion**：使用的 API 版本
2. **kind**：要创建的对象类型
3. **metadata**：唯一标识对象的数据
4. **spec**：对象的期望状态

```yaml
apiVersion: v1           # API 版本
kind: Pod                # 对象类型
metadata:                # 标识信息
  name: my-pod
  namespace: default
  labels:
    app: myapp
spec:                    # 期望状态
  containers:
  - name: nginx
    image: nginx:1.20
```

### Namespace

Namespace 提供在单个集群内隔离资源组的机制。

**默认 Namespace**：
- **default**：未指定 namespace 的对象的默认 namespace
- **kube-system**：用于 Kubernetes 系统组件
- **kube-public**：所有用户可公开读取
- **kube-node-lease**：用于节点心跳/租约对象

**创建 Namespace**：
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ml-training
```

```bash
kubectl create namespace ml-training
```

**使用 Namespace**：
```bash
# 在 namespace 中创建资源
kubectl create deployment nginx --image=nginx -n ml-training

# 查看 namespace 中的资源
kubectl get pods -n ml-training

# 查看所有 namespaces
kubectl get namespaces
```

**资源配额**（按 namespace）：
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ml-quota
  namespace: ml-training
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    requests.nvidia.com/gpu: "4"
    persistentvolumeclaims: "10"
```

### 标签和选择器

**标签**：附加到对象的键值对，用于识别和组织。

```yaml
metadata:
  labels:
    environment: production
    app: ml-api
    version: v1.2.3
    tier: backend
    owner: ml-team
```

**选择器**：按标签查询对象。

**基于等值**：
```bash
kubectl get pods -l environment=production
kubectl get pods -l environment!=staging
kubectl get pods -l 'environment in (production, qa)'
```

**基于集合**：
```yaml
selector:
  matchLabels:
    app: ml-api
  matchExpressions:
  - key: tier
    operator: In
    values:
    - backend
    - api
  - key: environment
    operator: NotIn
    values:
    - development
```

**标签的重要性**：
- Service 使用选择器查找 Pod
- Deployment 使用选择器管理 ReplicaSet
- Network Policy 使用选择器进行访问控制
- Operator 和控制器按标签查询

### 注解

注解将非标识性元数据附加到对象。与标签不同，注解不用于选择。

```yaml
metadata:
  annotations:
    description: "ML model serving API v1.2.3"
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    kubectl.kubernetes.io/last-applied-configuration: |
      {...}
    deployed-by: "jenkins-pipeline"
    git-commit: "a3b5f9c"
```

**常见用途**：
- 构建/发布信息
- 客户端库配置
- 工具特定配置
- 文档/描述

## 集群网络

Kubernetes 网络模型有四个关键要求：

1. **所有 Pod 可以与所有其他 Pod 通信**，无需 NAT
2. **所有节点可以与所有 Pod 通信**，无需 NAT
3. **Pod 看到的自身 IP** 与其他 Pod 看到的相同
4. **同一 Pod 中的容器**共享网络命名空间

### Pod 网络

每个 Pod 获得自己的 IP 地址。Pod 内的容器：
- 共享相同的 IP 地址
- 共享相同的网络命名空间
- 通过 localhost 通信
- 共享同一端口空间（没有端口冲突！）

```
Pod (IP: 10.244.1.5)
┌─────────────────────────────┐
│  Container 1                │
│  listening on :8080         │
│           ↕                 │
│       localhost             │
│           ↕                 │
│  Container 2                │
│  listening on :9090         │
└─────────────────────────────┘
```

### Container Network Interface (CNI)

Kubernetes 使用 CNI 插件设置网络。流行的 CNI 插件：

**Calico**：
- 使用 BGP 的第 3 层网络
- 网络策略执行
- 适合大规模部署

**Flannel**：
- 简单的覆盖网络
- 易于设置
- 使用 VXLAN 或 host-gw 后端

**Weave Net**：
- 网状网络
- 自动发现
- 支持加密

**Cilium**：
- 基于 eBPF
- 高级网络策略
- 服务网格能力

### Service 网络

Service 为动态 Pod 集合提供稳定的 IP 地址和 DNS 名称。

**ClusterIP**（默认）：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ml-api
spec:
  type: ClusterIP         # 集群内部 IP
  selector:
    app: ml-api
  ports:
  - port: 80              # Service 端口
    targetPort: 8080      # 容器端口
```

**DNS 名称**：
Service 自动获得 DNS 名称：
```
<service-name>.<namespace>.svc.cluster.local

# 示例：
ml-api.default.svc.cluster.local
redis.cache.svc.cluster.local
```

### 网络策略

网络策略控制 Pod 之间的流量（防火墙规则）。

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ml-api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: ml-api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

此策略：
- 适用于标签为 `app=ml-api` 的 Pod
- 允许来自标签为 `role=frontend` 的 Pod 的入站流量，端口 8080
- 允许到标签为 `app=database` 的 Pod 的出站流量，端口 5432
- 拒绝所有其他流量（默认拒绝）

## Kubernetes 如何调度工作负载

### 端到端部署流程

让我们追踪创建 Deployment 时发生的事情：

```bash
kubectl create deployment nginx --image=nginx --replicas=3
```

**分步过程**：

**1. kubectl → API Server**：
```
kubectl constructs API request:
POST /apis/apps/v1/namespaces/default/deployments
{
  "metadata": {"name": "nginx"},
  "spec": {
    "replicas": 3,
    "selector": {"matchLabels": {"app": "nginx"}},
    "template": {...}
  }
}
```

**2. API Server**：
- 认证请求
- 授权请求（RBAC）
- 验证 Deployment spec
- 运行准入控制器
- 将 Deployment 存储在 etcd 中
- 向 kubectl 返回成功

**3. Deployment Controller**（监控 API Server）：
```
Deployment controller 注意到新的 Deployment
  → 创建包含期望副本数 3 的 ReplicaSet
  → 将 ReplicaSet 存储在 etcd 中（通过 API Server）
```

**4. ReplicaSet Controller**（监控 API Server）：
```
ReplicaSet controller 注意到新的 ReplicaSet
  → 当前副本数：0
  → 期望副本数：3
  → 创建 3 个 Pod 对象（spec.nodeName 为空）
  → 将 Pod 存储在 etcd 中（通过 API Server）
```

**5. Scheduler**（监控 API Server 中未设置 nodeName 的 Pod）：
```
Scheduler 注意到 3 个未调度的 Pod
对于每个 Pod：
  → 过滤合适的节点（资源、亲和性、taint）
  → 对节点评分
  → 选择最佳节点
  → 更新 Pod，设置 spec.nodeName = "node-2"
  → 存储在 etcd 中（通过 API Server）
```

**6. kubelet on node-2**（监控 API Server 中分配给其节点的 Pod）：
```
kubelet 注意到 Pod 被调度到其节点
  → 拉取 nginx 镜像（如果未缓存）
  → 创建 Pod sandbox（网络命名空间）
  → 启动 nginx 容器
  → 监控容器健康
  → 向 API Server 报告状态
```

**7. Endpoints Controller**（如果存在 Service）：
```
Endpoints controller 注意到具有匹配标签的新 Pod
  → 使用 Pod IP 更新 Service Endpoints
  → 存储在 etcd 中（通过 API Server）
```

**8. kube-proxy**（监控 API Server 中的 Service/Endpoints）：
```
kube-proxy 注意到更新的 Endpoints
  → 更新 iptables/ipvs 规则
  → 发往 Service 的流量现在负载均衡到新的 Pod
```

### 可视化时间线

```
Time  Component             Action
─────────────────────────────────────────────────────────
  0s  kubectl               创建 Deployment
  0s  API Server            将 Deployment 存储在 etcd 中
  1s  Deployment Controller 创建 ReplicaSet
  1s  API Server            将 ReplicaSet 存储在 etcd 中
  2s  ReplicaSet Controller 创建 3 个 Pod 对象
  2s  API Server            将 Pod 存储在 etcd 中
  3s  Scheduler             将 Pod 分配到节点
  3s  API Server            用 nodeName 更新 Pod
  4s  kubelet (node-1)      拉取镜像，启动容器
  4s  kubelet (node-2)      拉取镜像，启动容器
  4s  kubelet (node-3)      拉取镜像，启动容器
 10s  kubelet (nodes)       容器正在运行
 10s  API Server            Pod 状态：Running
 10s  Endpoints Controller  更新 Service endpoints
 11s  kube-proxy (all)      更新网络规则
 11s  完成                  Pod 开始服务流量
```

## 高可用和容错

### 控制平面 HA

生产集群运行多个控制平面节点：

```
       Load Balancer (API Server endpoint)
                  │
         ┌────────┼────────┐
         ▼        ▼        ▼
     API Srv  API Srv  API Srv
         │        │        │
         └────────┼────────┘
                  │
            ┌─────┴─────┐
            ▼           ▼           ▼
         etcd-1      etcd-2      etcd-3
      (Raft leader) (follower) (follower)
```

**关键点**：
- **多个 API Server**：负载均衡，无状态
- **etcd 集群**：3 或 5 个节点，使用 Raft 共识
- **单一 Controller Manager**：领导者选举（只有一个活跃）
- **单一 Scheduler**：领导者选举（只有一个活跃）

**领导者选举**：
Controller Manager 和 Scheduler 使用领导者选举确保只有一个活跃：
```
3 个 Controller Manager 在运行
  → 一个持有领导者租约
  → 其他处于待命状态
  → 如果领导者故障，待命者在几秒内接管
```

### 工作节点容错

**节点故障检测**：
```
kubelet 停止发送心跳
  → Node controller 将节点标记为 Unknown（40 秒后）
  → Pod 被标记为 Unknown（5 分钟后）
  → Scheduler 不将新 Pod 放置到该节点
  → Pod 被驱逐（5 分钟后）
  → ReplicaSet 在健康节点上重新创建 Pod
```

**Pod 驱逐宽限期**：
```yaml
# 在 Pod spec 中
terminationGracePeriodSeconds: 30

# 驱逐时：
1. 向容器发送 SIGTERM
2. 等待最多 30 秒
3. 如果仍在运行，发送 SIGKILL
4. 移除 Pod
```

### 自愈

Kubernetes 自动从故障中恢复：

**容器崩溃**：
```
容器因错误退出
  → kubelet 重启容器
  → 如果重复失败，指数级退避
  → CrashLoopBackOff 状态（10s, 20s, 40s, 80s, 160s, 5m 最大）
```

**节点故障**：
```
节点变得不可达
  → Pod 被重新调度到健康节点
  → StatefulSet 的 Pod 需要手动干预（默认情况下）
```

**应用故障**：
```
Liveness probe 失败
  → 容器重启
Readiness probe 失败
  → Pod 从 Service endpoints 中移除（无流量）
  → 容器继续运行
```

## Kubernetes 在 AI/ML 基础设施中的应用

### 为什么在 ML 中使用 Kubernetes？

1. **资源管理**：高效使用昂贵的 GPU 资源
2. **可扩展性**：将模型服务从 1 个扩展到数千个副本
3. **可移植性**：在任何地方运行（云、本地、混合）
4. **隔离**：分离开发、预发布、生产环境
5. **自动化**：自动化部署、扩缩、恢复

### K8s 上的 ML 工作负载模式

**1. 模型训练**（批处理 Job）：
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: model-training
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: trainer
        image: pytorch/pytorch:latest
        command: ["python", "train.py"]
        resources:
          limits:
            nvidia.com/gpu: 4  # 4 GPUs
            memory: "64Gi"
        volumeMounts:
        - name: training-data
          mountPath: /data
        - name: model-output
          mountPath: /models
      volumes:
      - name: training-data
        persistentVolumeClaim:
          claimName: training-data-pvc
      - name: model-output
        persistentVolumeClaim:
          claimName: model-storage-pvc
```

**2. 模型服务**（Deployment）：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: model-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: model-server
  template:
    metadata:
      labels:
        app: model-server
    spec:
      containers:
      - name: tensorflow-serving
        image: tensorflow/serving:latest-gpu
        ports:
        - containerPort: 8501
        resources:
          limits:
            nvidia.com/gpu: 1
        env:
        - name: MODEL_NAME
          value: "my_model"
        volumeMounts:
        - name: model-volume
          mountPath: /models
      volumes:
      - name: model-volume
        persistentVolumeClaim:
          claimName: model-pvc
```

**3. Jupyter Notebook**（StatefulSet）：
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: jupyter
spec:
  serviceName: jupyter
  replicas: 1
  selector:
    matchLabels:
      app: jupyter
  template:
    metadata:
      labels:
        app: jupyter
    spec:
      containers:
      - name: jupyter
        image: jupyter/tensorflow-notebook:latest
        ports:
        - containerPort: 8888
        resources:
          limits:
            nvidia.com/gpu: 1
        volumeMounts:
        - name: notebook-storage
          mountPath: /home/jovyan
  volumeClaimTemplates:
  - metadata:
      name: notebook-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

**4. 分布式训练**（Kubeflow, PyTorch Operator）：
```yaml
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: distributed-training
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:latest
            resources:
              limits:
                nvidia.com/gpu: 1
    Worker:
      replicas: 3
      template:
        spec:
          containers:
          - name: pytorch
            image: pytorch/pytorch:latest
            resources:
              limits:
                nvidia.com/gpu: 1
```

### ML 特定 Kubernetes 工具

- **Kubeflow**：Kubernetes 上的端到端 ML 平台
- **KServe**：模型服务框架
- **Argo Workflows**：ML 管道编排
- **MLflow**：实验跟踪、模型注册中心
- **Seldon Core**：高级模型服务
- **NVIDIA GPU Operator**：自动化 GPU 管理

## 总结和关键要点

### 核心概念

1. **Kubernetes 架构**是主从架构，控制平面管理工作节点
2. **API Server** 是中央通信中心——所有通信都通过它
3. **etcd** 使用 Raft 共识存储所有集群状态
4. **Scheduler** 根据资源和约束将 Pod 分配到节点
5. **Controllers** 持续将期望状态与实际状态进行协调
6. **kubelet** 是管理 Pod 生命周期的节点代理
7. **kube-proxy** 实现 Service 网络和负载均衡

### 关键原则

- **声明式**：描述期望状态，Kubernetes 使其成为现实
- **自愈**：自动从故障中恢复
- **电平触发**：Controllers 定期检查状态
- **API 中心化**：所有组件通过 API Server 交互
- **可扩展**：添加自定义资源、控制器、调度器

### 思维模型

将 Kubernetes 视为容器的**分布式操作系统**：
- **API Server** = 内核（所有调用都通过它）
- **etcd** = 文件系统（存储所有状态）
- **Scheduler** = 进程调度器（将工作分配给 CPU/节点）
- **Controllers** = 系统守护进程（保持系统运行）
- **kubelet** = init 系统（管理每台机器上的进程/容器）

### 对 AI/ML 工程师

- Kubernetes 为 ML 工作负载提供**标准化基础设施**
- 通过 device plugin 进行**GPU 管理**对于训练/推理至关重要
- **Job** 用于训练，**Deployment** 用于服务，**StatefulSet** 用于 notebook
- **Namespace** 隔离团队和环境
- **资源配额**防止资源争用
- **水平扩缩**使模型服务具有弹性

### 下一步

现在你已经理解了 Kubernetes 架构，你可以：
1. 部署应用（Lecture 02）
2. 使用 Helm 进行包管理（Lecture 03）
3. 运维和调试集群（Lecture 04）
4. 在 Kubernetes 上构建真实的 ML 基础设施

### 延伸阅读

- 官方 Kubernetes 文档: https://kubernetes.io/docs/
- Kubernetes API 参考: https://kubernetes.io/docs/reference/
- CNCF Landscape: https://landscape.cncf.io/
- Kubernetes Enhancement Proposals (KEPs): https://github.com/kubernetes/enhancements

---

**恭喜！** 你现在理解了 Kubernetes 的底层工作原理。这个基础知识将贯穿你在 AI 基础设施领域的整个职业生涯。在下一讲中，我们将通过向 Kubernetes 部署应用来将所学知识付诸实践。
