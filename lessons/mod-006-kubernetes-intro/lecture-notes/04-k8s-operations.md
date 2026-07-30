# Lecture 04: Kubernetes 运维

## 目录
1. [引言](#引言)
2. [kubectl 基础](#kubectl-基础)
3. [调试 Pod 和 Deployment](#调试-pod-和-deployment)
4. [日志和事件](#日志和事件)
5. [在容器中执行命令](#在容器中执行命令)
6. [资源监控](#资源监控)
7. [常见问题排查](#常见问题排查)
8. [kubectl 效率技巧](#kubectl-效率技巧)
9. [集群维护](#集群维护)
10. [安全运维](#安全运维)
11. [总结和关键要点](#总结和关键要点)

## 引言

理解 Kubernetes 架构和如何部署应用至关重要，但日常运维需要一套不同的技能。本讲座涵盖了你作为 AI 基础设施工程师将定期执行的实际运维任务：调试失败的 Pod、查看日志、监控资源和排查问题。

### 学习目标

完成本讲座后，你将：
- 掌握日常运维的基本 kubectl 命令
- 有效调试失败的 Pod 和 Deployment
- 查看和分析日志与事件
- 在运行中的容器内执行命令
- 监控跨集群的资源使用情况
- 排查常见的 Kubernetes 问题
- 使用别名和快捷键高效使用 kubectl
- 执行基本的集群维护任务
- 在运维中应用安全最佳实践

### 前置要求
- Lecture 01-03（架构、部署、Helm）
- kubectl 已安装和配置
- 可访问 Kubernetes 集群
- 基本的 Linux 命令行能力

## kubectl 基础

### kubectl 上下文和配置

kubectl 使用 kubeconfig 文件连接到集群。

```bash
# 查看当前上下文
kubectl config current-context

# 列出所有上下文
kubectl config get-contexts

# 切换上下文
kubectl config use-context production-cluster

# 查看 kubeconfig
kubectl config view

# 设置默认 namespace
kubectl config set-context --current --namespace=production
```

**Kubeconfig 结构**：
```yaml
# ~/.kube/config
apiVersion: v1
kind: Config
clusters:
- cluster:
    server: https://k8s.example.com:6443
    certificate-authority-data: <base64-cert>
  name: production-cluster
contexts:
- context:
    cluster: production-cluster
    namespace: default
    user: admin-user
  name: production
current-context: production
users:
- name: admin-user
  user:
    client-certificate-data: <base64-cert>
    client-key-data: <base64-key>
```

### 基本 kubectl 命令

**获取资源**：
```bash
# 列出资源
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get nodes

# 所有资源
kubectl get all

# 所有 namespace 中的所有资源
kubectl get all --all-namespaces
kubectl get all -A  # 简写

# 宽输出（更多信息）
kubectl get pods -o wide

# JSON 输出
kubectl get pod nginx -o json

# YAML 输出
kubectl get pod nginx -o yaml

# 自定义列
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# JSONPath 查询
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# 监控变化
kubectl get pods -w
```

**描述资源**：
```bash
# 包括事件的详细信息
kubectl describe pod nginx-pod
kubectl describe deployment nginx-deployment
kubectl describe node worker-1

# 多个资源
kubectl describe pods
```

**创建/应用资源**：
```bash
# 从文件创建
kubectl create -f manifest.yaml

# 应用（创建或更新）
kubectl apply -f manifest.yaml

# 应用目录
kubectl apply -f ./manifests/

# 应用 URL
kubectl apply -f https://example.com/manifest.yaml

# 从命令行创建
kubectl create deployment nginx --image=nginx
kubectl create service clusterip nginx --tcp=80:80
```

**删除资源**：
```bash
# 按名称删除
kubectl delete pod nginx-pod

# 从文件删除
kubectl delete -f manifest.yaml

# 删除所有带有标签的 pod
kubectl delete pods -l app=nginx

# 删除 namespace 中的所有资源
kubectl delete all --all -n my-namespace

# 强制删除（跳过优雅终止）
kubectl delete pod nginx-pod --force --grace-period=0
```

**编辑资源**：
```bash
# 原地编辑（打开编辑器）
kubectl edit deployment nginx-deployment

# 使用特定编辑器编辑
KUBE_EDITOR="nano" kubectl edit pod nginx-pod

# 修补资源
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'
```

### 资源快捷键

```bash
# 常用快捷键
po    # pods
deploy # deployments
svc   # services
ns    # namespaces
no    # nodes
pv    # persistentvolumes
pvc   # persistentvolumeclaims
cm    # configmaps
sa    # serviceaccounts

# 示例
kubectl get po
kubectl describe deploy nginx
kubectl delete svc nginx
```

### Namespace 操作

```bash
# 创建 namespace
kubectl create namespace development

# 列出 namespaces
kubectl get namespaces
kubectl get ns

# 设置默认 namespace
kubectl config set-context --current --namespace=development

# 从特定 namespace 获取资源
kubectl get pods -n production

# 从所有 namespaces 获取资源
kubectl get pods --all-namespaces
kubectl get pods -A
```

## 调试 Pod 和 Deployment

### Pod 状态和阶段

**Pod 阶段**：
- **Pending**：Pod 已接受但尚未调度或容器未创建
- **Running**：Pod 已绑定到节点，容器已创建
- **Succeeded**：所有容器成功终止
- **Failed**：所有容器已终止，至少一个失败
- **Unknown**：无法确定 Pod 状态

**容器状态**：
- **Waiting**：容器尚未运行（拉取镜像等）
- **Running**：容器正在执行
- **Terminated**：容器执行结束

### 调试工作流

```
1. 检查 Pod 状态
   ↓
2. 描述 Pod（查看事件）
   ↓
3. 检查日志
   ↓
4. 检查资源约束
   ↓
5. 检查节点健康
   ↓
6. 进入容器（如果正在运行）
```

### 第 1 步：检查 Pod 状态

```bash
# 查看 Pod 状态
kubectl get pods

# 常见状态及其含义：
# Running：Pod 正常运行
# Pending：等待调度
# ContainerCreating：拉取镜像/创建容器
# CrashLoopBackOff：容器反复崩溃
# ImagePullBackOff：无法拉取容器镜像
# Error：容器以错误退出
# Completed：Job/batch Pod 成功完成
# Terminating：Pod 正在被删除
# Unknown：节点失去联系

# 使用宽输出检查状态
kubectl get pods -o wide

# 检查特定 Pod
kubectl get pod nginx-pod
```

### 第 2 步：描述 Pod

```bash
# 描述 Pod 以查看事件
kubectl describe pod nginx-pod

# 要检查的关键部分：
# - Status：当前阶段
# - Conditions：Ready， ContainersReady， PodScheduled
# - Containers：容器状态
# - Events：发生了什么（对调试至关重要！）
```

**示例事件**：
```
Events:
  Type     Reason            Message
  ----     ------            -------
  Normal   Scheduled         Successfully assigned default/nginx-pod to worker-1
  Normal   Pulling           Pulling image "nginx:1.21"
  Normal   Pulled            Successfully pulled image
  Normal   Created           Created container nginx
  Normal   Started           Started container nginx

  # 或者错误事件：
  Warning  Failed            Failed to pull image "nginx:invalid-tag"
  Warning  BackOff           Back-off restarting failed container
```

### 第 3 步：常见 Pod 问题和解决方案

**问题：ImagePullBackOff**

```bash
# 症状
NAME        READY   STATUS             RESTARTS   AGE
my-pod      0/1     ImagePullBackOff   0          2m

# 检查事件
kubectl describe pod my-pod

# 常见原因：
# 1. 镜像不存在
# 2. 镜像标签错误
# 3. 私有仓库需要认证
# 4. 镜像拉取频率限制已超

# 解决方案：
# 1. 修复镜像名称/标签
# 2. 创建 imagePullSecret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypass

# 添加到 Pod spec
imagePullSecrets:
- name: regcred
```

**问题：CrashLoopBackOff**

```bash
# 症状
NAME        READY   STATUS             RESTARTS   AGE
my-pod      0/1     CrashLoopBackOff   5          5m

# 容器启动后反复崩溃
# 检查日志以了解原因
kubectl logs my-pod

# 检查崩溃前的容器日志
kubectl logs my-pod --previous

# 常见原因：
# 1. 应用启动时出错
# 2. 缺少环境变量
# 3. 配置错误
# 4. 资源不足
# 5. Liveness probe 过早失败

# 解决方案取决于日志，但常见修复：
# - 修复应用代码
# - 添加所需的环境变量/配置
# - 增加资源
# - 调整 liveness probe 的 initialDelaySeconds
```

**问题：Pending（未调度）**

```bash
# 症状
NAME        READY   STATUS    RESTARTS   AGE
my-pod      0/1     Pending   0          5m

# 检查为什么未调度
kubectl describe pod my-pod

# 事件中显示的常见原因：
# 1. 节点 CPU/内存不足
# 2. 没有节点匹配 nodeSelector/affinity
# 3. Taint 阻止调度
# 4. PVC 不可用

# 示例事件：
Events:
  Warning  FailedScheduling  0/3 nodes are available: 3 Insufficient memory.

# 解决方案：
# 1. 添加更多节点或调整现有节点
# 2. 减少资源请求
# 3. 修复 nodeSelector/affinity
# 4. 移除 taint 或添加 toleration
# 5. 创建/修复 PVC
```

**问题：ContainerCreating（卡住）**

```bash
# 症状
NAME        READY   STATUS              RESTARTS   AGE
my-pod      0/1     ContainerCreating   0          5m

# 检查事件
kubectl describe pod my-pod

# 常见原因：
# 1. 卷挂载问题
# 2. 镜像拉取慢
# 3. CNI 插件问题

# 示例：
Events:
  Warning  FailedMount  Unable to mount volumes: timeout expired waiting for volumes to attach

# 解决方案：
# - 检查 PVC 状态
# - 验证 StorageClass 是否存在
# - 检查节点的 kubelet 日志
```

**问题：Pod 就绪但未接收流量**

```bash
# 症状
# Pod 处于 Running 状态但 Service 未路由流量

# 检查 Pod readiness
kubectl get pod my-pod -o json | jq '.status.conditions'

# 检查 Pod 是否有 readiness probe
kubectl get pod my-pod -o yaml | grep -A5 readinessProbe

# 检查 Pod 是否匹配 Service selector
kubectl describe service my-service

# 检查 endpoints
kubectl get endpoints my-service

# 如果 endpoints 为空，说明标签不匹配
# 修复：更新 Pod 标签或 Service selector
```

### 调试 Deployment

```bash
# 检查 Deployment 状态
kubectl get deployment nginx-deployment

# 详细状态
kubectl describe deployment nginx-deployment

# 检查 rollout 状态
kubectl rollout status deployment/nginx-deployment

# 检查 rollout 历史
kubectl rollout history deployment/nginx-deployment

# 检查 ReplicaSet
kubectl get replicaset
kubectl describe replicaset <replicaset-name>

# 常见 Deployment 问题：
# 1. 镜像拉取错误
# 2. 资源约束
# 3. Liveness probe 失败
# 4. 配置错误

# 通过检查 Deployment 创建的 Pod 进行调试
kubectl get pods -l app=nginx

# 然后按上述方法调试单个 Pod
```

## 日志和事件

### 查看日志

**基本日志命令**：
```bash
# 查看 Pod 日志
kubectl logs nginx-pod

# 持续跟踪日志（流式）
kubectl logs -f nginx-pod

# 特定容器的日志（多容器 Pod）
kubectl logs nginx-pod -c sidecar-container

# 崩溃前的容器日志
kubectl logs nginx-pod --previous

# 从某个时间点开始的日志
kubectl logs nginx-pod --since=1h
kubectl logs nginx-pod --since-time=2023-01-01T00:00:00Z

# 最后 N 行
kubectl logs nginx-pod --tail=100

# 带时间戳的日志
kubectl logs nginx-pod --timestamps

# 匹配标签的所有 Pod 的日志
kubectl logs -l app=nginx

# Pod 中所有容器的日志
kubectl logs nginx-pod --all-containers=true
```

**ML 训练 Job 日志示例**：
```bash
# 流式传输训练日志
kubectl logs -f training-job-xyz --tail=50

# 将日志保存到文件
kubectl logs training-job-xyz > training-logs.txt

# 监控多个 Pod
kubectl logs -l job-name=training-job -f --max-log-requests=10
```

### 查看事件

```bash
# 集群范围的事件
kubectl get events

# 特定 namespace 中的事件
kubectl get events -n production

# 按时间戳排序的事件
kubectl get events --sort-by='.lastTimestamp'

# 监控事件
kubectl get events -w

# 特定资源的事件
kubectl describe pod nginx-pod  # 包含事件

# 过滤事件
kubectl get events --field-selector involvedObject.kind=Pod
kubectl get events --field-selector type=Warning
kubectl get events --field-selector reason=Failed

# 人类可读的时间戳
kubectl get events --sort-by='.lastTimestamp' -o custom-columns=LAST:.lastTimestamp,TYPE:.type,REASON:.reason,MESSAGE:.message
```

**事件类型**：
- **Normal**：常规操作（Created， Started， Pulled 等）
- **Warning**：需要注意的问题（Failed， BackOff， Unhealthy 等）

### 日志聚合

对于生产环境，使用集中式日志：

**EFK Stack** (Elasticsearch, Fluentd, Kibana)：
```bash
# Fluentd 作为 DaemonSet 运行
kubectl get daemonset fluentd -n kube-system

# 日志发送到 Elasticsearch
# 在 Kibana Web UI 中查看
```

**PLG Stack** (Promtail, Loki, Grafana)：
```bash
# Promtail 收集日志
# Loki 存储日志
# Grafana 可视化

# 在 Grafana 中使用 LogQL 查询日志
{namespace="production", app="ml-model"} |= "error"
```

**ELK Stack** (Elasticsearch, Logstash, Kibana)：
```bash
# 类似于 EFK 但使用 Logstash
# 更适合复杂的日志处理
```

## 在容器中执行命令

### kubectl exec

```bash
# 在 Pod 中执行命令
kubectl exec nginx-pod -- ls /usr/share/nginx/html

# 交互式 shell
kubectl exec -it nginx-pod -- /bin/bash
kubectl exec -it nginx-pod -- /bin/sh  # 如果没有 bash

# 特定容器（多容器 Pod）
kubectl exec -it nginx-pod -c sidecar -- /bin/sh

# 以特定用户执行
kubectl exec nginx-pod -- su - appuser -c "whoami"

# 设置环境变量
kubectl exec nginx-pod -- env DEBUG=true /app/script.sh
```

### 容器内常用调试命令

```bash
# 交互式 shell
kubectl exec -it nginx-pod -- /bin/bash

# 在容器内：

# 检查进程
ps aux

# 检查网络
netstat -tlnp
ss -tlnp

# 检查文件
ls -la /app
cat /etc/config/app.properties

# 检查环境
env | grep -i db

# 检查磁盘空间
df -h

# 检查内存
free -h

# 测试连接
curl localhost:8080/health
wget -qO- http://api-service:8080/

# DNS 解析
nslookup redis-service
dig redis-service.default.svc.cluster.local

# 检查挂载的卷
mount | grep /data

# 检查日志（如果应用写入文件）
tail -f /var/log/app.log

# 安装调试工具（如果需要，临时）
apt-get update && apt-get install -y curl
apk add curl  # 针对基于 alpine 的镜像
```

### 调试容器（临时容器）

对于没有 shell 或调试工具的容器：

```bash
# 向运行中的 Pod 添加临时调试容器
kubectl debug -it nginx-pod --image=busybox --target=nginx

# 使用不同的镜像调试
kubectl debug nginx-pod --image=ubuntu --target=nginx

# 创建带有调试工具的 Pod 副本
kubectl debug nginx-pod -it --copy-to=nginx-debug --container=debug --image=busybox
```

### kubectl run 用于快速测试

```bash
# 创建临时测试 Pod
kubectl run test-pod --rm -it --image=busybox -- /bin/sh

# 测试网络连接
kubectl run test-curl --rm -it --image=curlimages/curl -- curl http://nginx-service

# 测试 DNS
kubectl run test-dns --rm -it --image=busybox -- nslookup kubernetes.default

# 测试 PostgreSQL 连接
kubectl run test-pg --rm -it --image=postgres:14 -- psql -h postgres-service -U admin
```

## 资源监控

### kubectl top

需要安装 Metrics Server。

```bash
# 节点资源使用
kubectl top nodes

# 输出：
# NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# master     250m         12%    1024Mi          51%
# worker-1   1200m        30%    4096Mi          68%
# worker-2   800m         20%    3072Mi          51%

# Pod 资源使用
kubectl top pods

# 所有 namespaces
kubectl top pods -A

# 特定 namespace
kubectl top pods -n production

# 按内存排序
kubectl top pods --sort-by=memory

# 按 CPU 排序
kubectl top pods --sort-by=cpu

# Pod 中的容器
kubectl top pod nginx-pod --containers
```

### 资源配额和限制

```bash
# 查看资源配额
kubectl get resourcequota

# 描述配额
kubectl describe resourcequota compute-quota

# 查看限制范围
kubectl get limitrange

# 检查实际与请求的资源
kubectl describe node worker-1

# "Allocated resources" 部分显示：
# Resource           Requests      Limits
# --------           --------      ------
# cpu                2500m (62%)   5000m (125%)
# memory             8Gi (50%)     16Gi (100%)
# nvidia.com/gpu     2             2
```

### 监控 ML 工作负载

```bash
# 监控 GPU 使用（需要容器中有 nvidia-smi）
kubectl exec -it ml-training-pod -- nvidia-smi

# 监控 GPU Pod
kubectl top pods -l gpu=true --sort-by=memory

# 检查 GPU 分配
kubectl describe node gpu-node-1 | grep -A5 "Allocated resources"

# 输出：
# nvidia.com/gpu: 2 (of 4 available)
```

### 监控工具

**Prometheus + Grafana**：
```bash
# 部署 Prometheus Operator
helm install prometheus prometheus-community/kube-prometheus-stack

# 访问 Grafana
kubectl port-forward svc/prometheus-grafana 3000:80

# 查看预构建的仪表盘：
# - Kubernetes 集群监控
# - Node exporter 指标
# - Pod 资源使用
```

**Kubernetes Dashboard**：
```bash
# 部署 dashboard
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

# 创建 admin 用户并获取 token
kubectl -n kubernetes-dashboard create token admin-user

# 访问 dashboard
kubectl proxy
# 访问：http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

## 常见问题排查

### 节点问题

**Node NotReady**：
```bash
# 检查节点状态
kubectl get nodes

# NAME       STATUS     ROLES    AGE   VERSION
# worker-1   NotReady   <none>   5d    v1.25.0

# 描述节点
kubectl describe node worker-1

# 检查条件
kubectl get node worker-1 -o json | jq '.status.conditions'

# 常见原因：
# 1. kubelet 未运行
# 2. 网络插件问题
# 3. 磁盘压力
# 4. 内存压力

# SSH 到节点并检查：
# kubelet 状态
systemctl status kubelet

# kubelet 日志
journalctl -u kubelet -f

# 重启 kubelet
systemctl restart kubelet
```

**节点磁盘压力**：
```bash
# 症状
kubectl describe node worker-1
# Conditions:
#   DiskPressure    True

# 解决方案：清理磁盘
# SSH 到节点
docker system prune -a
# 或
crictl rmi --prune

# 删除旧的容器日志
find /var/log/pods -type f -mtime +7 -delete
```

### 网络问题

**Pod 无法访问 Service**：
```bash
# 从 Pod 测试
kubectl run test --rm -it --image=busybox -- wget -qO- http://nginx-service

# 如果失败，检查：

# 1. Service 是否存在
kubectl get service nginx-service

# 2. Endpoints 是否已填充
kubectl get endpoints nginx-service

# 3. Pod 标签是否匹配 service selector
kubectl get pods --show-labels
kubectl describe service nginx-service

# 4. Network policy 是否阻止流量
kubectl get networkpolicy

# 5. CoreDNS 是否正常工作
kubectl run test --rm -it --image=busybox -- nslookup nginx-service
```

**DNS 解析问题**：
```bash
# 检查 CoreDNS Pod
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 检查 CoreDNS 日志
kubectl logs -n kube-system -l k8s-app=kube-dns

# 从 Pod 测试 DNS
kubectl run test --rm -it --image=busybox -- nslookup kubernetes.default

# 检查 Pod 中的 DNS 配置
kubectl exec test-pod -- cat /etc/resolv.conf

# 应显示：
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
```

### 存储问题

**PVC 卡在 Pending**：
```bash
# 检查 PVC 状态
kubectl get pvc

# 描述 PVC
kubectl describe pvc my-pvc

# 事件中的常见原因：
# 1. 没有可用的 PV
# 2. StorageClass 不存在
# 3. 访问模式不匹配
# 4. 存储空间不足

# 检查 PV
kubectl get pv

# 检查 StorageClass
kubectl get storageclass

# 解决方案：创建 PV 或修复 StorageClass
```

**Pod 无法挂载卷**：
```bash
# 症状
kubectl describe pod my-pod
# Events:
#   Warning  FailedMount  MountVolume.SetUp failed

# 常见原因：
# 1. PVC 不存在
# 2. PVC 未绑定
# 3. 卷已挂载到不同节点（RWO）
# 4. NFS 服务器不可达

# 检查 PVC 状态
kubectl get pvc

# 检查卷是否挂载到别处
kubectl get pods -o json | jq '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="my-pvc") | .metadata.name'

# 解决方案：修复 PVC 或确保多 Pod 访问使用 RWX 访问模式
```

### 配置问题

**ConfigMap/Secret 未找到**：
```bash
# 症状
kubectl describe pod my-pod
# Events:
#   Warning  Failed  Error: configmap "app-config" not found

# 检查是否存在
kubectl get configmap app-config
kubectl get secret app-secret

# 检查 namespace
kubectl get configmap app-config -n correct-namespace

# 解决方案：创建 ConfigMap/Secret 或修复引用
```

**环境变量错误**：
```bash
# 检查 Pod 中的实际环境变量
kubectl exec my-pod -- env

# 与预期比较
kubectl get pod my-pod -o yaml | grep -A10 env:

# 检查 ConfigMap/Secret 内容
kubectl get configmap app-config -o yaml
kubectl get secret app-secret -o jsonpath='{.data.password}' | base64 -d
```

### 性能问题

**高 CPU 使用**：
```bash
# 识别高 CPU Pod
kubectl top pods --sort-by=cpu

# 检查 CPU 节流
kubectl describe pod high-cpu-pod

# 检查容器限制
kubectl get pod high-cpu-pod -o jsonpath='{.spec.containers[].resources}'

# 解决方案：
# 1. 增加 CPU 限制
# 2. 优化应用代码
# 3. 水平扩展（更多副本）
```

**高内存使用 / OOMKilled**：
```bash
# 检查 Pod 状态
kubectl get pods

# NAME      READY   STATUS      RESTARTS   AGE
# my-pod    0/1     OOMKilled   5          10m

# 检查事件
kubectl describe pod my-pod
# Events:
#   Warning  OOMKilling  Memory cgroup out of memory: Kill process

# 检查内存限制
kubectl get pod my-pod -o jsonpath='{.spec.containers[].resources.limits.memory}'

# 解决方案：
# 1. 增加内存限制
# 2. 修复应用中的内存泄漏
# 3. 使用更节省内存的算法
```

## kubectl 效率技巧

### 别名和快捷键

```bash
# 添加到 ~/.bashrc 或 ~/.zshrc

# 基本别名
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias kex='kubectl exec -it'
alias kap='kubectl apply -f'

# Namespace 快捷键
alias kgp='kubectl get pods'
alias kgd='kubectl get deployments'
alias kgs='kubectl get services'
alias kgn='kubectl get nodes'

# 高级别名
alias kgpa='kubectl get pods --all-namespaces'
alias kgpw='kubectl get pods -o wide'
alias kgpwa='kubectl get pods -o wide --all-namespaces'
alias kdp='kubectl describe pod'
alias kdd='kubectl describe deployment'
alias kds='kubectl describe service'

# 监控
alias kgpw='watch kubectl get pods'

# 日志
alias klf='kubectl logs -f'
alias klp='kubectl logs -f --previous'

# 进入 Pod shell
alias ksh='kubectl exec -it $1 -- /bin/sh'
alias kbash='kubectl exec -it $1 -- /bin/bash'

# 使用示例：
# k get pods
# kg deploy
# kl my-pod
# kex my-pod -- /bin/bash
```

### kubectl 插件

**krew** (kubectl 插件管理器)：
```bash
# 安装 krew
# https://krew.sigs.k8s.io/docs/user-guide/setup/install/

# 流行插件
kubectl krew install ctx      # 切换上下文
kubectl krew install ns       # 切换 namespace
kubectl krew install tree     # 显示资源层级
kubectl krew install tail     # 跟踪多个 Pod 的日志
kubectl krew install neat     # 清理 kubectl 输出
kubectl krew install whoami   # 显示当前用户信息

# 使用
kubectl ctx                   # 列出上下文
kubectl ctx production        # 切换到生产环境
kubectl ns production         # 切换 namespace
kubectl tree deployment nginx # 显示 deployment 层级
kubectl tail -l app=nginx     # 跟踪所有 nginx pod 的日志
```

**有用的插件**：
- **kubectl-tree**：显示资源层级
- **kubectl-tail**：多 Pod 日志跟踪
- **kubectl-neat**：去除 kubectl 输出中的杂乱内容
- **kubectl-images**：显示集群中使用的镜像
- **kubectl-resource-capacity**：显示资源容量

### Shell 补全

```bash
# Bash
echo 'source <(kubectl completion bash)' >>~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -F __start_kubectl k' >>~/.bashrc

# Zsh
echo 'source <(kubectl completion zsh)' >>~/.zshrc
echo 'alias k=kubectl' >>~/.zshrc
echo 'complete -F __start_kubectl k' >>~/.zshrc

# 测试
k get po<TAB>  # 应自动补全为 'pods'
```

### 高级 kubectl 技巧

**JSONPath 查询**：
```bash
# 获取 Pod IP
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# 获取容器镜像
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}' | tr -s ' ' '\n'

# 获取 Pod 和节点
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'

# 获取失败的 Pod
kubectl get pods --field-selector=status.phase!=Running

# 获取资源限制
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].resources.limits}{"\n"}{end}'
```

**Field Selectors**：
```bash
# 按状态
kubectl get pods --field-selector status.phase=Running
kubectl get pods --field-selector status.phase!=Running

# 按节点
kubectl get pods --field-selector spec.nodeName=worker-1

# 按 namespace
kubectl get pods --all-namespaces --field-selector metadata.namespace!=kube-system

# 多个条件
kubectl get pods --field-selector status.phase=Running,spec.nodeName=worker-1
```

**Label Selectors**：
```bash
# 基于等值
kubectl get pods -l app=nginx
kubectl get pods -l app=nginx,tier=frontend

# 基于集合
kubectl get pods -l 'environment in (production, staging)'
kubectl get pods -l 'tier notin (frontend)'
kubectl get pods -l 'app,environment'  # 同时具有两个标签
kubectl get pods -l '!app'  # 没有 app 标签
```

## 集群维护

### 节点排空

```bash
# 排空节点（驱逐所有 pod 以进行维护）
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data

# 封锁节点（阻止调度但不驱逐）
kubectl cordon worker-1

# 取消封锁节点（允许再次调度）
kubectl uncordon worker-1

# 工作流：
# 1. 排空节点
# 2. 执行维护（升级、重启等）
# 3. 取消封锁节点
```

### 备份资源

```bash
# 备份所有资源
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml

# 备份特定 namespace
kubectl get all -n production -o yaml > production-backup.yaml

# 备份特定资源类型
kubectl get configmap,secret,pvc --all-namespaces -o yaml > configs-backup.yaml

# 使用 velero 备份（推荐）
velero backup create my-backup --include-namespaces production

# 恢复
velero restore create --from-backup my-backup
```

### 资源清理

```bash
# 删除已完成的 Job
kubectl delete jobs --field-selector status.successful=1

# 删除被驱逐的 Pod
kubectl get pods --all-namespaces --field-selector status.phase=Failed -o json | kubectl delete -f -

# 删除旧的 ReplicaSet
kubectl delete replicaset --all-namespaces --field-selector status.replicas=0

# 清理未使用的 PV
kubectl get pv --field-selector status.phase=Released

# 查找并删除未使用的 ConfigMap/Secret（手动检查）
# 这需要分析以确保它们未被引用
```

## 安全运维

### RBAC 调试

```bash
# 检查当前用户权限
kubectl auth can-i list pods
kubectl auth can-i delete deployments --namespace production

# 检查 service account 的权限
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa

# 检查用户可以访问哪些资源
kubectl auth can-i --list

# 描述 role/rolebinding
kubectl describe role developer
kubectl describe rolebinding developer-binding

# 获取所有 roles/rolebindings
kubectl get roles --all-namespaces
kubectl get rolebindings --all-namespaces
kubectl get clusterroles
kubectl get clusterrolebindings
```

### 安全扫描

```bash
# 扫描镜像漏洞（trivy）
trivy image nginx:1.21

# 扫描集群配置（kubesec）
kubectl get deploy nginx -o yaml | kubesec scan -

# 检查 Pod Security Standards
kubectl label namespace production pod-security.kubernetes.io/enforce=restricted
```

### 审计日志

```bash
# 查看审计日志（在控制平面节点上）
cat /var/log/kubernetes/audit.log | grep "user.username"

# 按用户过滤
cat /var/log/kubernetes/audit.log | jq 'select(.user.username=="admin")'

# 按 namespace 过滤
cat /var/log/kubernetes/audit.log | jq 'select(.objectRef.namespace=="production")'

# 按操作过滤
cat /var/log/kubernetes/audit.log | jq 'select(.verb=="delete")'
```

## 总结和关键要点

### 基本 kubectl 命令

```bash
# 获取信息
kubectl get pods/deployments/services
kubectl describe pod <name>
kubectl logs <pod>
kubectl top pods/nodes

# 调试
kubectl exec -it <pod> -- /bin/bash
kubectl debug <pod>
kubectl get events

# 运维操作
kubectl apply -f manifest.yaml
kubectl delete pod <name>
kubectl scale deployment <name> --replicas=5
kubectl rollout status/history/undo

# 上下文管理
kubectl config use-context <context>
kubectl config set-context --current --namespace=<ns>
```

### 调试清单

1. **检查 Pod 状态**：`kubectl get pods`
2. **描述 Pod**：`kubectl describe pod <name>`
3. **查看事件**：在 describe 输出中检查 Events 部分
4. **检查日志**：`kubectl logs <pod>`
5. **崩溃前日志**（如果崩溃了）：`kubectl logs <pod> --previous`
6. **进入 Pod**：`kubectl exec -it <pod> -- /bin/bash`
7. **检查 Service/Endpoints**：`kubectl get endpoints <service>`
8. **检查资源**：`kubectl top pods`
9. **检查节点**：`kubectl describe node <node>`

### 常见问题快速参考

| 问题 | 检查内容 | 解决方案 |
|-------|-------|----------|
| ImagePullBackOff | 镜像名称、仓库认证 | 修复镜像，添加 imagePullSecret |
| CrashLoopBackOff | 日志、资源限制 | 修复应用，调整资源 |
| Pending | 节点资源、PVC | 添加节点，修复 PVC |
| 未接收流量 | Service selector、readiness probe | 修复标签，调整探针 |
| OOMKilled | 内存限制 | 增加限制 |
| Node NotReady | kubelet、磁盘空间 | 重启 kubelet，清理磁盘 |

### 最佳实践

1. **使用 namespaces** 组织资源
2. **为所有资源添加标签** 便于过滤
3. **始终设置资源 requests/limits**
4. **使用 readiness probes** 进行流量路由
5. **使用 liveness probes** 实现自动重启
6. **集中日志**（EFK、PLG、ELK）
7. **监控资源**（Prometheus、Grafana）
8. **定期备份**（Velero）
9. **实践 RBAC** 最小权限原则
10. **实现自动化** 使用脚本和 CI/CD

### 效率技巧

- 为常用命令使用**别名**
- 安装 **kubectl 插件**（krew）
- 启用 **shell 补全**
- 使用 **kubectl explain** 查看资源文档
- 掌握 **JSONPath** 和 **field/label selectors**
- 保持 **kubeconfig 整洁**
- 对多集群使用 **contexts**

### 后续步骤

- 在 Exercise 03 中练习调试场景
- 使用 Prometheus 设置监控
- 配置集中式日志
- 用脚本自动化运维
- 了解 GitOps（ArgoCD、Flux）
- 学习 Kubernetes operator
- 探索服务网格（Istio、Linkerd）

### 延伸阅读

- kubectl 速查表：https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- 调试指南：https://kubernetes.io/docs/tasks/debug/
- 最佳实践：https://kubernetes.io/docs/concepts/configuration/overview/
- 监控：https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/

---

**恭喜！** 你现在拥有有效管理 Kubernetes 集群的运维技能。定期练习这些技能，以建立肌肉记忆和排障直觉。
