# Lecture 02: 在 Kubernetes 上部署应用

## 目录
1. [引言](#引言)
2. [Pod：原子单元](#pod原子单元)
3. [ReplicaSet：维持期望状态](#replicaset维持期望状态)
4. [Deployment：声明式更新](#deployment声明式更新)
5. [Service：稳定的网络端点](#service稳定的网络端点)
6. [ConfigMap：配置管理](#configmap配置管理)
7. [Secret：敏感数据管理](#secret敏感数据管理)
8. [持久化存储](#持久化存储)
9. [完整应用示例](#完整应用示例)
10. [最佳实践](#最佳实践)
11. [总结和关键要点](#总结和关键要点)

## 引言

在上一讲中，你学习了 Kubernetes 的架构工作原理。现在是时候将这些知识付诸实践，向 Kubernetes 部署实际应用了。本讲座涵盖了你作为 AI 基础设施工程师每天都会用到的核心工作负载资源。

### 学习目标

完成本讲座后，你将：
- 有效创建和管理 Pod
- 使用 ReplicaSet 维持期望的副本数
- 使用 Deployment 部署应用并执行滚动更新
- 使用不同类型的 Service 暴露应用
- 使用 ConfigMap 和 Secret 管理应用配置
- 理解使用 Volume 和 PersistentVolumeClaim 的持久化存储
- 部署完整的多层应用
- 应用生产部署的最佳实践

### 前置要求
- Lecture 01：Kubernetes 架构
- 安装并配置了 kubectl
- 可访问 Kubernetes 集群（Docker Desktop、Minikube 或云环境）
- 基本理解 YAML 语法

### 验证你的环境

```bash
# 检查集群访问
kubectl cluster-info

# 确认节点已就绪
kubectl get nodes

# 检查当前 context
kubectl config current-context

# 为本讲座创建测试 namespace
kubectl create namespace lecture-02-demo
kubectl config set-context --current --namespace=lecture-02-demo
```

## Pod：原子单元

**Pod** 是 Kubernetes 中最小的可部署单元。它代表集群中运行的单个进程实例。

### Pod 基础知识

**关键特性**：
- Pod 包含一个或多个容器（通常为一个）
- Pod 中的容器共享相同的网络命名空间（相同 IP、相同端口）
- Pod 中的容器共享相同的存储卷
- Pod 是临时的——它们可以随时被创建和销毁
- 每个 Pod 获得唯一的 IP 地址（集群内）

### 单容器 Pod

最常见的模式：每个 Pod 一个容器。

```yaml
# simple-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
      name: http
```

**创建 Pod**：
```bash
kubectl apply -f simple-pod.yaml

# 观察启动过程
kubectl get pods -w

# 查看详情
kubectl describe pod nginx-pod

# 查看日志
kubectl logs nginx-pod

# 访问 Pod（端口转发）
kubectl port-forward nginx-pod 8080:80
# 访问 http://localhost:8080
```

### 多容器 Pod

有时需要在 Pod 中有多个容器（边车模式）。

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logging
spec:
  containers:
  # 主应用容器
  - name: app
    image: busybox
    command: ['sh', '-c', 'while true; do echo "$(date) - Application log" >> /var/log/app.log; sleep 5; done']
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

  # 用于日志处理的边车容器
  - name: log-shipper
    image: busybox
    command: ['sh', '-c', 'tail -f /var/log/app.log']
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log

  volumes:
  - name: shared-logs
    emptyDir: {}
```

**关键点**：
- 两个容器共享相同的网络（可以通过 localhost 通信）
- 两个容器共享相同的卷（shared-logs）
- 如果任一容器失败，整个 Pod 失败

```bash
kubectl apply -f multi-container-pod.yaml

# 查看特定容器的日志
kubectl logs app-with-logging -c app
kubectl logs app-with-logging -c log-shipper
```

### 常见的多容器模式

**1. Sidecar 模式**：辅助容器增强主容器
```
Main Container → App
Sidecar Container → 日志发送、指标收集、代理
```

**2. Ambassador 模式**：外部服务的代理容器
```
Main Container → 连接 localhost:6379
Ambassador Container → 代理到外部 Redis 集群
```

**3. Adapter 模式**：标准化输出/接口
```
Main Container → 自定义指标格式
Adapter Container → 转换为 Prometheus 格式
```

### Pod 生命周期

```
Pending → ContainerCreating → Running → Succeeded/Failed
                                  ↓
                          CrashLoopBackOff (如果失败)
```

**Pod 阶段**：
- **Pending**：Pod 已接受，但容器尚未创建
- **Running**：Pod 已绑定到节点，至少一个容器在运行
- **Succeeded**：所有容器成功终止
- **Failed**：所有容器已终止，至少一个失败
- **Unknown**：无法确定 Pod 状态

### 资源请求和限制

始终为生产工作负载指定资源需求。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:       # 最低保证资源
        memory: "128Mi"
        cpu: "250m"   # 250 millicores = 0.25 CPU
      limits:         # 最大允许资源
        memory: "256Mi"
        cpu: "500m"   # 500 millicores = 0.5 CPU
```

**资源单位**：
- **CPU**：`1` = 1 CPU 核心，`1000m` = 1 核心，`100m` = 0.1 核心
- **内存**：`128Mi` = 128 mebibytes，`1Gi` = 1 gibibyte，`1000M` = 1000 megabytes

**行为**：
- **Requests**：用于调度器找到合适的节点
- **Limits**：由 kubelet 强制执行；超过则容器被杀死（内存）或节流（CPU）

### 健康探针

Kubernetes 可以监控容器健康并采取行动。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: healthy-pod
spec:
  containers:
  - name: web
    image: nginx:1.21
    ports:
    - containerPort: 80

    # Liveness probe - 容器是否存活？
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 2
      failureThreshold: 3

    # Readiness probe - 容器是否准备好接收流量？
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 3

    # Startup probe - 启动慢的容器是否已启动？
    startupProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 0
      periodSeconds: 10
      failureThreshold: 30  # 30 * 10 = 5 分钟启动时间
```

**探针类型**：
1. **exec**：在容器中执行命令（退出码 0 = 成功）
2. **httpGet**：HTTP GET 请求（状态码 200-399 = 成功）
3. **tcpSocket**：TCP 连接（连接成功 = 成功）
4. **grpc**：gRPC 健康检查（需要 gRPC 健康检查协议）

**何时使用每种探针**：
- **Liveness**：检测死锁/挂起的容器 → 重启
- **Readiness**：控制 Pod 何时接收流量 → 从 Service 移除
- **Startup**：保护启动慢的容器免受 liveness probe 误杀

### Init 容器

Init 容器在应用容器之前运行，必须成功完成。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-pod
spec:
  # Init 容器按顺序在主容器之前运行
  initContainers:
  - name: download-data
    image: busybox
    command: ['sh', '-c', 'wget -O /data/model.pkl https://example.com/model.pkl']
    volumeMounts:
    - name: data
      mountPath: /data

  - name: check-dependencies
    image: busybox
    command: ['sh', '-c', 'until nslookup redis-service; do echo waiting for redis; sleep 2; done']

  # Init 容器成功后主应用容器启动
  containers:
  - name: app
    image: ml-inference:latest
    volumeMounts:
    - name: data
      mountPath: /data

  volumes:
  - name: data
    emptyDir: {}
```

**常见用例**：
- 下载依赖或配置
- 等待外部服务就绪
- 设置权限或数据
- 填充共享卷

### Pod 最佳实践

1. **使用标签**：有效组织和选择 Pod
2. **设置资源 requests/limits**：实现正确调度并避免资源争用
3. **使用健康探针**：启用自愈和正确的流量路由
4. **不以 root 运行**：使用 securityContext 降低权限
5. **每个容器单一职责**：保持容器职责明确
6. **使用特定镜像标签**：生产中避免使用 `latest`

## ReplicaSet：维持期望状态

**ReplicaSet** 确保在任何时候都运行指定数量的 Pod 副本。

### ReplicaSet 基础

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3           # Pod 期望数量
  selector:             # 要管理哪些 Pod
    matchLabels:
      app: nginx
      tier: frontend
  template:             # Pod 模板
    metadata:
      labels:           # 必须匹配 selector
        app: nginx
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f replicaset.yaml

# 查看 ReplicaSet
kubectl get replicasets
kubectl get rs  # 简写

# 查看 ReplicaSet 创建的 Pod
kubectl get pods -l app=nginx

# 扩展 ReplicaSet
kubectl scale replicaset nginx-replicaset --replicas=5

# 删除一个 Pod - 观察它被重新创建！
kubectl delete pod <pod-name>
kubectl get pods -w
```

### ReplicaSet 工作原理

```
ReplicaSet Controller 监控：
- ReplicaSet 对象
- 具有匹配标签的 Pod 对象

协调循环：
1. 统计当前具有匹配标签的 Pod
2. 与期望副本数比较
3. 如果计数 < 期望值，创建新 Pod
4. 如果计数 > 期望值，删除多余的 Pod
```

### 标签选择器

ReplicaSet 使用标签来识别其管理的 Pod。

```yaml
# 基于等值的选择器
selector:
  matchLabels:
    app: nginx
    environment: production

# 基于集合的选择器
selector:
  matchExpressions:
  - key: tier
    operator: In
    values: [frontend, backend]
  - key: environment
    operator: NotIn
    values: [development]
```

**重要**：ReplicaSet 选择器在创建后是**不可变**的！

### 什么时候不应该直接使用 ReplicaSet

ReplicaSet 很少直接创建。而是使用 **Deployment**，它会为你管理 ReplicaSet 并提供额外功能：
- 滚动更新
- 回滚能力
- 更新策略
- 版本历史

## Deployment：声明式更新

**Deployment** 为 Pod 和 ReplicaSet 提供声明式更新。这是部署无状态应用的推荐方式。

### 基本 Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

```bash
kubectl apply -f deployment.yaml

# 查看 Deployment
kubectl get deployments
kubectl get deploy  # 简写

# 查看 Deployment 创建的 ReplicaSet
kubectl get replicasets

# 查看 Pod
kubectl get pods

# 查看 deployment 详情
kubectl describe deployment nginx-deployment

# 查看 rollout 状态
kubectl rollout status deployment/nginx-deployment
```

### Deployment 层级结构

```
Deployment
    ↓ (创建和管理)
ReplicaSet
    ↓ (创建和管理)
Pods
```

### 扩展 Deployment

```bash
# 命令式扩展
kubectl scale deployment nginx-deployment --replicas=5

# 声明式扩展（推荐）
# 编辑 deployment.yaml: replicas: 5
kubectl apply -f deployment.yaml

# 自动扩展 (HPA)
kubectl autoscale deployment nginx-deployment --min=2 --max=10 --cpu-percent=80
```

### 滚动更新

Deployment 通过滚动更新实现零停机更新。

```yaml
# 更新策略配置
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 更新期间超出期望值的最大 Pod 数
      maxUnavailable: 1  # 更新期间低于期望值的最大 Pod 数
```

**更新 Deployment**：
```bash
# 方法 1：更新镜像
kubectl set image deployment/nginx-deployment nginx=nginx:1.22

# 方法 2：原地编辑
kubectl edit deployment nginx-deployment

# 方法 3：更新 manifest 并应用（推荐）
# 编辑 deployment.yaml: image: nginx:1.22
kubectl apply -f deployment.yaml

# 监控 rollout
kubectl rollout status deployment/nginx-deployment

# 查看 rollout 历史
kubectl rollout history deployment/nginx-deployment
```

**滚动更新过程**：
```
初始状态：nginx:1.21 的 3 个副本

1. 为 nginx:1.22 创建新的 ReplicaSet
2. 将新 ReplicaSet 扩展到 1 个（surge）
   旧：3, 新：1  (总计：4, maxSurge: 1)
3. 等待新 Pod 就绪
4. 将旧 ReplicaSet 缩减到 2 个
   旧：2, 新：1  (总计：3)
5. 将新 ReplicaSet 扩展到 2 个
   旧：2, 新：2  (总计：4)
6. 将旧 ReplicaSet 缩减到 1 个
   旧：1, 新：2  (总计：3)
7. 将新 ReplicaSet 扩展到 3 个
   旧：1, 新：3  (总计：4)
8. 将旧 ReplicaSet 缩减到 0 个
   旧：0, 新：3  (总计：3)
9. 完成！
```

### 回滚

如果更新出现问题，回滚到之前的版本。

```bash
# 查看 rollout 历史
kubectl rollout history deployment/nginx-deployment

# 查看特定版本
kubectl rollout history deployment/nginx-deployment --revision=2

# 回滚到上一个版本
kubectl rollout undo deployment/nginx-deployment

# 回滚到特定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# 暂停 rollout（用于排障）
kubectl rollout pause deployment/nginx-deployment

# 恢复 rollout
kubectl rollout resume deployment/nginx-deployment
```

### 更新策略

**1. RollingUpdate**（默认）：
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # 可以是数字或百分比
    maxUnavailable: 25%
```
- 逐步用新 Pod 替换旧 Pod
- 零停机
- 两个版本同时运行（临时）

**2. Recreate**：
```yaml
strategy:
  type: Recreate
```
- 所有旧 Pod 终止后创建新 Pod
- 会发生停机
- 一次只运行一个版本
- 当同时运行多个版本会引发问题时有用

### ML 模型服务的 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-model-server
  labels:
    app: ml-model
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ml-model
  template:
    metadata:
      labels:
        app: ml-model
        version: v1
    spec:
      containers:
      - name: model-server
        image: tensorflow/serving:2.8.0
        ports:
        - name: http
          containerPort: 8501
        - name: grpc
          containerPort: 8500
        env:
        - name: MODEL_NAME
          value: "my_model"
        - name: MODEL_BASE_PATH
          value: "/models"
        resources:
          requests:
            memory: "2Gi"
            cpu: "1"
          limits:
            memory: "4Gi"
            cpu: "2"
            nvidia.com/gpu: "1"
        livenessProbe:
          httpGet:
            path: /v1/models/my_model
            port: 8501
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /v1/models/my_model
            port: 8501
          initialDelaySeconds: 10
          periodSeconds: 5
        volumeMounts:
        - name: model-storage
          mountPath: /models
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: model-pvc
```

## Service：稳定的网络端点

Pod 是临时的，具有动态 IP 地址。**Service** 提供稳定的网络端点和负载均衡。

### 为什么需要 Service？

**问题**：Pod IP 在以下情况会变化：
- Pod 被重新调度
- Deployment 被更新
- Pod 崩溃并重新创建

**解决方案**：Service 提供：
- 稳定的 IP 地址（ClusterIP）
- 稳定的 DNS 名称
- 跨 Pod 后端的负载均衡
- 服务发现

### Service 类型

1. **ClusterIP**（默认）：仅集群内部 IP
2. **NodePort**：在每个节点 IP 的静态端口上暴露服务
3. **LoadBalancer**：配置云负载均衡器
4. **ExternalName**：映射到外部 DNS 名称

### ClusterIP Service

最常用的内部服务类型。

```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: ClusterIP           # 默认类型，可以省略
  selector:                 # 具有这些标签的 Pod 是后端
    app: nginx
  ports:
  - name: http
    protocol: TCP
    port: 80                # Service 端口
    targetPort: 80          # 容器端口
```

```bash
kubectl apply -f service-clusterip.yaml

# 查看 service
kubectl get services
kubectl get svc  # 简写

# 查看 service 详情（显示端点）
kubectl describe service nginx-service

# Service 在集群内可访问
kubectl run test-pod --rm -it --image=busybox -- sh
# 在 Pod 内：
wget -qO- nginx-service
wget -qO- nginx-service.default.svc.cluster.local
```

**DNS 名称**：
```
<service-name>.<namespace>.svc.cluster.local

# 示例：
nginx-service.default.svc.cluster.local
redis-cache.production.svc.cluster.local

# 短格式（同 namespace）：
nginx-service
```

### Service 工作原理

```
1. 创建 Service，selector: app=nginx
2. Endpoints controller 监控 Service 和 Pod
3. 查找所有标签为 app=nginx 的 Pod
4. 使用 Pod IP 创建 Endpoints 对象：
   - 10.244.1.5:80
   - 10.244.2.3:80
   - 10.244.3.7:80
5. kube-proxy 监控 Service 和 Endpoints
6. 更新 iptables/ipvs 规则：
   - 发往 Service ClusterIP 的流量 → 负载均衡到 Pod IP
```

**查看 Endpoints**：
```bash
kubectl get endpoints nginx-service
kubectl get ep nginx-service  # 简写
```

### NodePort Service

在每个节点 IP 的静态端口上暴露服务（30000-32767）。

```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - name: http
    protocol: TCP
    port: 80              # Service 端口（集群内部）
    targetPort: 80        # 容器端口
    nodePort: 30080       # 节点端口（可选，省略时自动分配）
```

```bash
kubectl apply -f service-nodeport.yaml
kubectl get service nginx-nodeport

# 从集群外部访问
# 获取节点 IP
kubectl get nodes -o wide

# 访问服务
curl http://<NODE-IP>:30080
```

**流量流程**：
```
External Client
    ↓
<任意节点-IP>:30080 (NodePort)
    ↓
Service ClusterIP:80
    ↓
负载均衡到 Pod:80
```

### LoadBalancer Service

配置云负载均衡器（在云平台上工作）。

```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
```

```bash
kubectl apply -f service-loadbalancer.yaml

# 获取外部 IP（在云上大约需要 1 分钟）
kubectl get service nginx-loadbalancer -w

# 通过外部 IP 访问
curl http://<EXTERNAL-IP>
```

**云配置**：
- AWS：创建 ELB/ALB/NLB
- GCP：创建 Google Cloud Load Balancer
- Azure：创建 Azure Load Balancer

**流量流程**：
```
External Client
    ↓
Cloud Load Balancer (external IP)
    ↓
NodePort on nodes
    ↓
Service ClusterIP
    ↓
Pod backends
```

### ExternalName Service

将服务映射到外部 DNS 名称（无选择器或端点）。

```yaml
# service-externalname.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.example.com
```

```bash
# Pod 可以通过内部名称访问外部服务
mysql -h external-database -u user -p
# 实际连接到 database.example.com
```

### Headless Service

没有 ClusterIP 的 Service，用于直接 Pod 访问。

```yaml
# service-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None       # Headless service
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

```bash
# DNS 直接返回 Pod IP，而非 Service IP
nslookup nginx-headless.default.svc.cluster.local

# 返回：
# nginx-headless.default.svc.cluster.local has address 10.244.1.5
# nginx-headless.default.svc.cluster.local has address 10.244.2.3
# nginx-headless.default.svc.cluster.local has address 10.244.3.7
```

**用例**：
- StatefulSet（需要寻址特定 Pod）
- 客户端负载均衡
- 无需负载均衡的服务发现

### Service 最佳实践

1. **默认使用 ClusterIP**：仅在需要时外部暴露
2. **命名端口**：使用命名端口以提高清晰度和灵活性
3. **会话亲和性**：必要时使用 `sessionAffinity: ClientIP`
4. **健康检查**：确保 Pod 有 readiness 探针
5. **Namespace 服务**：使用 namespace 组织服务

## ConfigMap：配置管理

**ConfigMap** 以键值对形式存储非机密配置数据。

### 创建 ConfigMap

**方法 1：从字面值**：
```bash
kubectl create configmap app-config \
  --from-literal=database_host=postgres.default \
  --from-literal=database_port=5432 \
  --from-literal=log_level=info
```

**方法 2：从文件**：
```bash
# 创建配置文件
cat > app.properties <<EOF
database.host=postgres.default
database.port=5432
log.level=info
EOF

kubectl create configmap app-config --from-file=app.properties
```

**方法 3：从 YAML**：
```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_host: "postgres.default"
  database_port: "5432"
  log_level: "info"
  # 多行配置
  app.properties: |
    database.host=postgres.default
    database.port=5432
    log.level=info
    feature.experimental=true
```

```bash
kubectl apply -f configmap.yaml

# 查看 ConfigMap
kubectl get configmaps
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml
```

### 使用 ConfigMap

**1. 环境变量**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    # 单个 key
    - name: DATABASE_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database_host

    # 所有 keys 作为环境变量
    envFrom:
    - configMapRef:
        name: app-config
```

**2. 卷挂载**（用于配置文件）：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

容器中创建的文件：
```
/etc/config/database_host
/etc/config/database_port
/etc/config/log_level
/etc/config/app.properties
```

**3. 特定 Key 作为文件**：
```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
    items:
    - key: app.properties
      path: application.properties
```

创建：`/etc/config/application.properties`

### ML 配置示例

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ml-model-config
data:
  MODEL_NAME: "resnet50"
  MODEL_VERSION: "1.0.0"
  BATCH_SIZE: "32"
  MAX_BATCH_DELAY: "5000"
  NUM_THREADS: "4"
  model_config.json: |
    {
      "model_name": "resnet50",
      "model_platform": "tensorflow",
      "input_format": "NHWC",
      "output_nodes": ["predictions"],
      "optimization_level": "2"
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-model-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ml-model
  template:
    metadata:
      labels:
        app: ml-model
    spec:
      containers:
      - name: model-server
        image: tensorflow/serving:latest
        envFrom:
        - configMapRef:
            name: ml-model-config
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: ml-model-config
          items:
          - key: model_config.json
            path: model_config.json
```

### ConfigMap 更新

```bash
# 更新 ConfigMap
kubectl edit configmap app-config

# 或应用更新后的 YAML
kubectl apply -f configmap.yaml
```

**重要**：
- Pod **不会**在 ConfigMap 更新时自动重启
- 环境变量**不会**更新（在 Pod 创建时设置）
- 挂载的文件**会**更新（最终一致，约 1 分钟）

**强制 Pod 更新**：
```bash
# 通过更新 deployment annotation 触发 rollout
kubectl patch deployment myapp -p \
  '{"spec":{"template":{"metadata":{"annotations":{"configmap-version":"'$(date +%s)'"}}}}}'
```

## Secret：敏感数据管理

**Secret** 存储密码、token 和密钥等敏感数据。

### 创建 Secret

**方法 1：从字面值**：
```bash
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=super-secret-password
```

**方法 2：从文件**：
```bash
echo -n 'admin' > username.txt
echo -n 'super-secret-password' > password.txt
kubectl create secret generic db-credentials \
  --from-file=username.txt \
  --from-file=password.txt
```

**方法 3：从 YAML**（值必须是 base64 编码）：
```bash
# 编码值
echo -n 'admin' | base64  # YWRtaW4=
echo -n 'super-secret-password' | base64  # c3VwZXItc2VjcmV0LXBhc3N3b3Jk
```

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=
  password: c3VwZXItc2VjcmV0LXBhc3N3b3Jk
```

```bash
kubectl apply -f secret.yaml

# 查看 secret（值被隐藏）
kubectl get secret db-credentials
kubectl describe secret db-credentials

# 解码 secret
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 -d
```

### Secret 类型

- **Opaque**（默认）：任意用户定义数据
- **kubernetes.io/service-account-token**：Service account token
- **kubernetes.io/dockerconfigjson**：Docker 仓库凭证
- **kubernetes.io/tls**：TLS 证书和密钥
- **kubernetes.io/ssh-auth**：SSH 凭证
- **kubernetes.io/basic-auth**：基本认证

### 使用 Secret

**1. 环境变量**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    env:
    - name: DB_USERNAME
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
```

**2. 卷挂载**（更安全）：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-volume
    secret:
      secretName: db-credentials
```

容器中创建的文件：
```
/etc/secrets/username
/etc/secrets/password
```

**在应用中读取**：
```python
# 从文件读取（比环境变量更安全）
with open('/etc/secrets/username') as f:
    username = f.read()
with open('/etc/secrets/password') as f:
    password = f.read()
```

### Docker Registry Secret

用于从私有容器仓库拉取镜像：

```bash
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-pod
spec:
  containers:
  - name: app
    image: myregistry/private-image:latest
  imagePullSecrets:
  - name: regcred
```

### TLS Secret

用于 HTTPS 端点：

```bash
kubectl create secret tls tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

### Secret 最佳实践

1. **使用卷挂载**而不是环境变量（更安全）
2. **限制 RBAC 访问** secret
3. **在 etcd 中启用静态加密**
4. **使用外部 secret 管理器**（AWS Secrets Manager、HashiCorp Vault）
5. **定期轮换 secret**
6. **不要将 secret 提交到 Git**（使用 .gitignore）
7. **使用 namespace 隔离 secret**
8. **限制 secret 大小**（每个 secret 最大 1MB）

### External Secrets Operator

对于生产环境，考虑使用 External Secrets Operator：

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secret-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-west-2
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secret-store
  target:
    name: db-credentials
  data:
  - secretKey: username
    remoteRef:
      key: prod/db/credentials
      property: username
  - secretKey: password
    remoteRef:
      key: prod/db/credentials
      property: password
```

## 持久化存储

Pod 是临时的，但数据通常需要持久化。Kubernetes 提供了几种存储抽象。

### 存储层次结构

```
Storage Class
    ↓ (配置)
Persistent Volume (PV)
    ↓ (绑定到)
Persistent Volume Claim (PVC)
    ↓ (挂载到)
Pod
```

### 卷类型

**1. emptyDir**：临时存储（Pod 删除时清除）
```yaml
volumes:
- name: temp-storage
  emptyDir: {}
```

**2. hostPath**：节点的文件系统（不推荐用于生产）
```yaml
volumes:
- name: host-storage
  hostPath:
    path: /data
    type: Directory
```

**3. PersistentVolumeClaim**：持久化存储
```yaml
volumes:
- name: data-storage
  persistentVolumeClaim:
    claimName: my-pvc
```

### PersistentVolume (PV)

由管理员或动态配置的集群存储资源。

```yaml
# pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
  - ReadWriteOnce      # 单节点读写
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast
  hostPath:            # 仅用于演示；生产环境使用云卷
    path: /mnt/data
```

**访问模式**：
- **ReadWriteOnce** (RWO)：单节点，读写
- **ReadOnlyMany** (ROX)：多节点，只读
- **ReadWriteMany** (RWX)：多节点，读写（罕见，通常为 NFS）

**回收策略**：
- **Retain**：手动回收（手动创建 PV 的默认值）
- **Delete**：PVC 删除时删除卷（动态配置的默认值）
- **Recycle**：清理卷并使其可用（已弃用）

### PersistentVolumeClaim (PVC)

用户对存储的请求。

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi    # 请求 50Gi（PV 有 100Gi）
  storageClassName: fast
```

```bash
kubectl apply -f pvc.yaml

# 查看 PVC
kubectl get pvc

# 查看绑定
kubectl describe pvc my-pvc
```

**绑定过程**：
```
1. 用户创建 PVC，请求 50Gi，accessMode RWO
2. Kubernetes 找到具有以下条件的 PV：
   - 至少 50Gi 容量
   - 访问模式 RWO
   - 匹配的 storageClassName
3. 将 PVC 绑定到 PV
4. PVC 状态：Bound
```

### 在 Pod 中使用 PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: my-pvc
```

### 动态配置

StorageClass 启用动态 PV 配置。

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs  # 云提供商 provisioner
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**带 StorageClass 的 PVC**：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: fast-ssd  # 使用 StorageClass
```

```bash
kubectl apply -f dynamic-pvc.yaml

# PV 自动创建！
kubectl get pv
kubectl get pvc
```

### ML 存储示例

```yaml
# ML 模型存储
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-storage
spec:
  accessModes:
  - ReadWriteMany  # 多个 Pod 需要访问
  resources:
    requests:
      storage: 1Ti
  storageClassName: nfs
---
# ML 训练数据
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: training-data
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Ti
  storageClassName: s3-csi  # S3 后端存储
---
# ML 训练 job
apiVersion: batch/v1
kind: Job
metadata:
  name: training-job
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: trainer
        image: pytorch/pytorch:latest
        command: ["python", "train.py"]
        volumeMounts:
        - name: training-data
          mountPath: /data
          readOnly: true
        - name: model-output
          mountPath: /models
      volumes:
      - name: training-data
        persistentVolumeClaim:
          claimName: training-data
      - name: model-output
        persistentVolumeClaim:
          claimName: model-storage
```

### 卷快照

创建卷的备份：

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: model-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: model-storage
```

## 完整应用示例

让我们部署一个完整的多层应用：Web 前端、API 后端和数据库。

### 应用架构

```
Internet
    ↓
LoadBalancer Service (frontend)
    ↓
Frontend Deployment (3 replicas)
    ↓
ClusterIP Service (api)
    ↓
Backend Deployment (3 replicas)
    ↓
ClusterIP Service (database)
    ↓
Database StatefulSet (1 replica)
    ↓
PersistentVolumeClaim
```

### 数据库层

```yaml
# database.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  POSTGRES_DB: "myapp"
---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
type: Opaque
data:
  POSTGRES_USER: YWRtaW4=        # admin
  POSTGRES_PASSWORD: cGFzc3dvcmQ=  # password
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        ports:
        - containerPort: 5432
        envFrom:
        - configMapRef:
            name: postgres-config
        - secretRef:
            name: postgres-secret
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  clusterIP: None  # Headless service for StatefulSet
```

### 后端 API 层

```yaml
# backend.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  DATABASE_HOST: "postgres"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "myapp"
  LOG_LEVEL: "info"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      containers:
      - name: api
        image: myapp/backend:v1
        ports:
        - containerPort: 8080
        envFrom:
        - configMapRef:
            name: backend-config
        env:
        - name: DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_USER
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: POSTGRES_PASSWORD
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
  - port: 8080
    targetPort: 8080
```

### 前端层

```yaml
# frontend.yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-config
data:
  API_URL: "http://backend:8080"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        version: v1
    spec:
      containers:
      - name: web
        image: myapp/frontend:v1
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: frontend-config
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

### 部署应用

```bash
# 按顺序部署
kubectl apply -f database.yaml
kubectl wait --for=condition=ready pod -l app=postgres --timeout=60s

kubectl apply -f backend.yaml
kubectl wait --for=condition=ready pod -l app=backend --timeout=60s

kubectl apply -f frontend.yaml

# 检查状态
kubectl get all

# 获取外部 IP
kubectl get service frontend

# 测试应用
curl http://<EXTERNAL-IP>
```

## 最佳实践

### 资源管理

1. **始终设置资源 requests 和 limits**：
```yaml
resources:
  requests:     # 用于调度
    memory: "128Mi"
    cpu: "100m"
  limits:       # 防止资源滥用
    memory: "256Mi"
    cpu: "200m"
```

2. **按 namespace 使用 ResourceQuota**：
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "100"
    requests.memory: "200Gi"
    limits.cpu: "200"
    limits.memory: "400Gi"
    persistentvolumeclaims: "10"
```

3. **使用 LimitRange 设置默认值**：
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - default:
      memory: "256Mi"
      cpu: "200m"
    defaultRequest:
      memory: "128Mi"
      cpu: "100m"
    type: Container
```

### 健康检查

始终实现健康探针：

```yaml
livenessProbe:    # 不健康时重启
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:   # 未就绪时从 service 移除
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

### 安全

1. **不以 root 运行**：
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000
  capabilities:
    drop:
    - ALL
```

2. **使用只读根文件系统**：
```yaml
securityContext:
  readOnlyRootFilesystem: true
```

3. **使用 Pod Security Standards**：
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

### 标签和注解

使用一致的标签：

```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: myapp-prod
    app.kubernetes.io/version: "1.2.3"
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: myapp
    app.kubernetes.io/managed-by: helm
    environment: production
    team: ml-platform
```

### 滚动更新

配置为零停机：

```yaml
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # 确保零停机
```

### 就绪和优雅关闭

```yaml
spec:
  # ... deployment spec
  template:
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
```

## 总结和关键要点

### 核心资源层次结构

```
Deployment（声明式更新）
    → ReplicaSet（维持副本数）
        → Pods（运行容器）
Service（稳定网络端点）
    → Endpoints（Pod IPs）
ConfigMap/Secret（配置）
    → 挂载到 Pods 作为文件或环境变量
PersistentVolumeClaim（存储请求）
    → PersistentVolume（集群存储）
```

### 关键命令

```bash
# Pods
kubectl run/apply/get/delete/logs/exec

# Deployments
kubectl create/get/describe/apply/scale/set image
kubectl rollout status/history/undo/pause/resume

# Services
kubectl create/get/describe/apply/delete

# ConfigMaps and Secrets
kubectl create/get/describe

# Storage
kubectl get pv/pvc/storageclass
```

### 最佳实践

1. **始终设置资源 requests 和 limits**
2. **使用健康探针**（liveness、readiness、startup）
3. **使用标签选择器**正确组织资源
4. **首选声明式管理**（`kubectl apply`）
5. **将配置与代码分离**（ConfigMap、Secret）
6. **使用滚动更新**实现零停机部署
7. **调试时使用 `kubectl describe` 和 `kubectl logs`**
8. **定期清理未使用的资源**
9. **遵循安全最佳实践**（非 root、RBAC、网络策略）
10. **备份 etcd 和 PVC 数据**

### 下一步

- **Lecture 03**：Helm - Kubernetes 包管理
- **Exercise 01**：首次部署练习
- 练习创建更复杂的应用
- 尝试不同的 Service 类型
- 实现您自己的 ConfigMap 和 Secret

### 延伸阅读

- Kubernetes Pods：https://kubernetes.io/docs/concepts/workloads/pods/
- Kubernetes Deployments：https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Kubernetes Services：https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes Storage：https://kubernetes.io/docs/concepts/storage/

---

**恭喜！** 你已掌握了在 Kubernetes 上部署应用的基础知识。在下一讲中，我们将学习 Helm，通过模板化和包管理来简化这些工作流程。
