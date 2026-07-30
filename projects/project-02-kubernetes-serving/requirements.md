# Project 02：详细需求

## 目录
1. [功能需求](#functional-requirements)
2. [非功能需求](#non-functional-requirements)
3. [Kubernetes 特定需求](#kubernetes-specific-requirements)
4. [验收标准](#acceptance-criteria)

---

## 功能需求

### FR-1: Kubernetes 部署配置

#### FR-1.1: 多副本部署
**需求：** 将模型 API 部署为具有 3 个副本的 Kubernetes Deployment 以实现高可用性。

**详情：**
- 使用 `apps/v1` Deployment API
- 设置 `spec.replicas: 3` 作为基准可用性
- 为服务发现配置适当的 Pod 标签
- 实现正确的选择器匹配

**验收标准：**
- 所有 3 个 Pod 运行中且处于 Ready 状态
- Pod 分布到可用节点上（如果是多节点集群）
- Deployment 显示 3/3 可用副本
- Pod 在故障时自动重建

**测试：**
```bash
kubectl get deployment model-api -n ml-serving
# 应显示：READY 3/3, UP-TO-DATE 3, AVAILABLE 3

kubectl get pods -n ml-serving
# 应显示 3 个 STATUS 为 Running 的 Pod
```

---

#### FR-1.2: 资源管理
**需求：** 配置 CPU 和内存资源 requests 和 limits 以实现可预测的性能。

**详情：**
- **CPU Requests：** 500m（0.5 核）- 保证的 CPU 分配
- **CPU Limits：** 1000m（1 核）- 最大 CPU 使用量
- **内存 Requests：** 1Gi - 保证的内存分配
- **内存 Limits：** 2Gi - 最大内存使用量（防止 OOM 杀死）

**理由：**
- Requests 确保调度到资源充足的节点
- Limits 防止其他 Pod 资源饥饿
- 内存 limits 应为 requests 的 2 倍以应对流量高峰
- CPU limits 允许高负载时突发

**验收标准：**
- Pod 成功调度并拥有资源保证
- 正常负载下无 OOMKilled 事件
- CPU 限流仅在持续使用超过 70% 时发生
- 负载测试期间内存使用保持在限制内

**测试：**
```bash
kubectl describe pod <pod-name> -n ml-serving | grep -A 8 Limits
# 应显示配置的 requests 和 limits

kubectl top pod -n ml-serving
# 监控实际资源使用情况
```

---

#### FR-1.3: 存活探针
**需求：** 实现存活探针以检测应用死锁并进行恢复。

**详情：**
- **端点：** `GET /health`
- **初始延迟：** 30 秒（允许模型加载时间）
- **周期：** 10 秒（检查频率）
- **超时：** 5 秒（请求超时）
- **失败阈值：** 3（重启前的连续失败次数）

**健康端点行为：**
- 应用响应正常时返回 200 OK
- 应用处于错误状态时返回 500
- 检查应用内部状态（不仅仅是 HTTP 服务器）

**验收标准：**
- 如果 Pod 30 秒无响应则自动重启（3 次失败 × 10 秒）
- 正常运行期间无误报
- Pod 事件中可见存活探针失败
- 应用重启后成功恢复

**测试：**
```bash
# 模拟应用挂起
kubectl exec <pod-name> -n ml-serving -- kill -STOP 1

# 等待 30-40 秒，Pod 应重启
kubectl get events --field-selector involvedObject.name=<pod-name> -n ml-serving
# 应显示存活探针失败
```

---

#### FR-1.4: 就绪探针
**需求：** 实现就绪探针以防止流量到达未准备好提供服务的 Pod。

**详情：**
- **端点：** `GET /health`
- **初始延迟：** 10 秒（比存活探针更快以便快速启动）
- **周期：** 5 秒（更频繁的检查）
- **超时：** 3 秒
- **失败阈值：** 3

**健康端点需求：**
- 验证模型已加载到内存
- 检查数据库连接（如适用）
- 验证所需配置已加载
- 仅当完全就绪时返回 200

**验收标准：**
- 新 Pod 在模型加载完成前不接收流量
- 未就绪的 Pod 从 Service 端点中移除
- 滚动更新等待新 Pod 就绪
- Pod 启动期间无 500 错误

**测试：**
```bash
# 部署新版本
kubectl set image deployment/model-api model-api=model-api:v1.1 -n ml-serving

# 观察端点
kubectl get endpoints model-api-service -n ml-serving --watch
# 应仅显示就绪的 Pod
```

---

#### FR-1.5: 配置管理
**需求：** 使用 ConfigMap 进行应用配置，将配置与代码分离。

**详情：**
- **ConfigMap 名称：** `model-config`
- **配置键：**
  - `model_name`：要加载的模型（例如 "resnet50"）
  - `log_level`：日志详细级别（DEBUG、INFO、WARNING、ERROR）
  - `max_batch_size`：最大推理批次大小
  - `timeout`：请求超时（秒）

**环境变量注入：**
```yaml
env:
  - name: MODEL_NAME
    valueFrom:
      configMapKeyRef:
        name: model-config
        key: model_name
```

**验收标准：**
- 配置更改无需重建镜像
- ConfigMap 更新后 Pod 自动重启（使用适当的注解）
- Pod 启动时验证配置
- 无效配置阻止 Pod 变为就绪状态

**测试：**
```bash
# 更新 ConfigMap
kubectl edit configmap model-config -n ml-serving

# 验证 Pod 重新加载（如果使用 ConfigMap 卷或 reloader）
kubectl rollout status deployment/model-api -n ml-serving
```

---

### FR-2: 服务与负载均衡

#### FR-2.1: ClusterIP Service
**需求：** 创建内部 ClusterIP Service 用于 Pod 间通信。

**详情：**
- **Service 类型：** ClusterIP（默认，仅内部）
- **端口：** 80（标准 HTTP）
- **目标端口：** 5000（应用端口）
- **选择器：** `app: model-api`
- **会话亲和性：** 无（均匀分配）

**验收标准：**
- Service 可从集群内部访问
- DNS 名称解析：`model-api-service.ml-serving.svc.cluster.local`
- 流量跨 Pod 轮询分发
- Pod 变化时 Service 自动更新

**测试：**
```bash
# 从集群内部
kubectl run test-pod --image=curlimages/curl -it --rm -n ml-serving -- \
  curl http://model-api-service/health

# 检查端点
kubectl get endpoints model-api-service -n ml-serving
```

---

#### FR-2.2: LoadBalancer Service
**需求：** 创建面向外部的 LoadBalancer Service 以提供互联网访问。

**详情：**
- **Service 类型：** LoadBalancer（云提供商集成）
- **外部端口：** 80（HTTP）
- **目标端口：** 5000
- **负载均衡器来源范围：**（可选）按 IP 限制

**云提供商行为：**
- **AWS EKS：** 创建 Classic ELB 或 NLB
- **GCP GKE：** 创建 TCP/UDP Load Balancer
- **Azure AKS：** 创建 Azure Load Balancer
- **Minikube：** 使用 `minikube tunnel` 进行本地测试

**验收标准：**
- 2 分钟内分配外部 IP
- Service 可从互联网访问
- 负载均衡器级别配置健康检查
- SSL/TLS 终止（如果已配置）

**测试：**
```bash
# 获取外部 IP
kubectl get svc model-api-lb -n ml-serving
EXTERNAL_IP=$(kubectl get svc model-api-lb -n ml-serving -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 从集群外部测试
curl http://$EXTERNAL_IP/health
```

---

#### FR-2.3: 会话亲和性（可选）
**需求：** 如果需要有状态会话，配置会话亲和性。

**详情：**
- **类型：** ClientIP
- **超时：** 3600 秒（1 小时）
- **使用场景：** 当客户端需要一致的路由到同一 Pod 时

**注意：** 对于无状态 ML 推理，通常不需要会话亲和性。

---

#### FR-2.4: Ingress 配置
**需求：** 实现基于路径规则的 NGINX Ingress HTTP 路由。

**详情：**
- **Ingress Class：** nginx
- **主机：** `model-api.example.com`
- **路径：**
  - `/predict` → model-api-service:80
  - `/health` → model-api-service:80
  - `/metrics` → model-api-service:80
- **TLS：** 可选（使用 cert-manager 获取自动证书）

**注解：**
- `nginx.ingress.kubernetes.io/rewrite-target: /`
- `nginx.ingress.kubernetes.io/ssl-redirect: "true"`
- `nginx.ingress.kubernetes.io/rate-limit: "100"`（每秒请求数）

**验收标准：**
- Ingress 控制器成功部署
- 基于主机的路由正常工作
- 基于路径的路由指向正确的端点
- TLS 证书有效（如果启用）

**测试：**
```bash
# 获取 Ingress IP
kubectl get ingress -n ml-serving

# 使用主机头测试
curl -H "Host: model-api.example.com" http://<ingress-ip>/health
```

---

### FR-3: 自动伸缩

#### FR-3.1: Horizontal Pod Autoscaler (HPA)
**需求：** 配置 HPA 以根据资源利用率自动伸缩 Pod。

**详情：**
- **最小副本数：** 3（可用性基准）
- **最大副本数：** 10（成本控制和集群容量）
- **目标 CPU 利用率：** 70%
- **目标内存利用率：** 80%

**伸缩行为：**
- **扩容：** 激进（快速响应负载）
  - 每 30 秒最多增加 100%
  - 或每 30 秒最多增加 2 个 Pod
  - 无稳定窗口（立即伸缩）

- **缩容：** 保守（避免振荡）
  - 每 60 秒最多减少 50%
  - 5 分钟稳定窗口
  - 防止短暂的流量高峰后快速缩容

**验收标准：**
- HPA 每 15 秒监控 Pod 指标
- 达到 70% CPU 后 1 分钟内扩容
- 低利用率持续 5 分钟后缩容
- 副本数无快速振荡
- 伸缩事件在 HPA 状态中可见

**测试：**
```bash
# 生成负载
kubectl run load-generator --image=busybox -n ml-serving -- /bin/sh -c \
  "while true; do wget -q -O- http://model-api-service/predict; done"

# 观察 HPA
kubectl get hpa model-api-hpa -n ml-serving --watch

# 检查伸缩事件
kubectl describe hpa model-api-hpa -n ml-serving
```

---

#### FR-3.2: Metrics Server
**需求：** 部署并配置 Kubernetes Metrics Server 以提供 HPA 数据。

**详情：**
- Metrics Server 从 kubelet 收集资源指标
- 向 HPA 控制器提供 CPU 和内存数据
- 每 15 秒更新

**安装：**
```bash
# 对于 Minikube
minikube addons enable metrics-server

# 对于云集群
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**验收标准：**
- `kubectl top nodes` 返回数据
- `kubectl top pods` 返回数据
- HPA 显示当前 CPU/内存利用率
- 指标在 30 秒内更新

---

### FR-4: 滚动更新与回滚

#### FR-4.1: RollingUpdate 策略
**需求：** 使用 RollingUpdate 策略实现零停机部署更新。

**详情：**
- **策略类型：** RollingUpdate
- **Max Surge：** 1（更新期间可创建 1 个额外 Pod）
- **Max Unavailable：** 0（始终维持最少副本数）
- **Min Ready Seconds：** 10（将 Pod 视为就绪前的等待时间）

**更新过程：**
1. 使用新版本创建 1 个新 Pod
2. 等待新 Pod 就绪
3. 终止 1 个旧 Pod
4. 重复直到所有 Pod 更新完成

**验收标准：**
- 更新期间零请求失败
- 始终保持至少 3 个 Pod 可用
- 3→3 Pod 更新在 5 分钟内完成
- 失败的更新自动回滚

**测试：**
```bash
# 更新镜像版本
kubectl set image deployment/model-api model-api=model-api:v1.1 -n ml-serving

# 实时观察滚动更新状态
kubectl rollout status deployment/model-api -n ml-serving

# 更新期间监控请求（应无错误）
while true; do curl http://<service-ip>/health; sleep 1; done
```

---

#### FR-4.2: 回滚能力
**需求：** 支持快速回滚到之前的部署版本。

**详情：**
- Kubernetes 维护滚动更新历史（默认：10 个修订版本）
- 一键回滚到上一个版本
- 回滚使用相同的 RollingUpdate 策略

**命令：**
```bash
# 查看滚动更新历史
kubectl rollout history deployment/model-api -n ml-serving

# 回滚到上一个版本
kubectl rollout undo deployment/model-api -n ml-serving

# 回滚到指定修订版本
kubectl rollout undo deployment/model-api --to-revision=2 -n ml-serving
```

**验收标准：**
- 回滚在 2 分钟内完成
- 回滚期间零停机
- 滚动更新历史显示所有修订版本
- 可回滚到任意之前的修订版本

---

### FR-5: 监控与可观测性

#### FR-5.1: Prometheus 部署
**需求：** 部署 Prometheus 用于指标采集和存储。

**详情：**
- **部署方式：** Helm chart (prometheus-community/prometheus)
- **保留期：** 7 天
- **抓取间隔：** 30 秒
- **抓取目标：**
  - Kubernetes API 服务器指标
  - 节点指标（通过 node-exporter）
  - Pod 指标（通过 cAdvisor）
  - 应用指标（通过 /metrics 端点）

**ServiceMonitor 配置：**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: model-api-metrics
spec:
  selector:
    matchLabels:
      app: model-api
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

**验收标准：**
- Prometheus 成功抓取所有目标
- 抓取失败率不超过 1%
- 指标在 Prometheus UI 中可见
- 数据保留 7 天

---

#### FR-5.2: Grafana 仪表盘
**需求：** 部署 Grafana 并创建全面的仪表盘。

**详情：**
- **部署方式：** Helm chart (grafana/grafana)
- **数据源：** Prometheus
- **必需仪表盘：**
   1. **集群概览：** 节点数、总 CPU/内存
   2. **应用指标：** 请求速率、延迟、错误
   3. **Pod 指标：** Pod 数、重启次数、资源使用
   4. **HPA 指标：** 当前/期望副本数、伸缩事件

**关键面板：**
- **Pod 数量变化：** `count(kube_pod_info{namespace="ml-serving"})`
- **CPU 使用率：** `rate(container_cpu_usage_seconds_total{pod=~"model-api.*"}[5m])`
- **内存使用量：** `container_memory_usage_bytes{pod=~"model-api.*"}`
- **请求速率：** `rate(model_api_requests_total[5m])`
- **错误率：** `rate(model_api_requests_total{status=~"5.."}[5m])`
- **P95 延迟：** `histogram_quantile(0.95, rate(model_api_request_duration_seconds_bucket[5m]))`

**验收标准：**
- 所有面板显示实时数据
- 仪表盘每 30 秒刷新
- 可选择的時間范围（1h、6h、24h、7d）
- 可导出为 JSON 进行版本控制

---

#### FR-5.3: 应用指标
**需求：** 在应用 `/metrics` 端点暴露 Prometheus 指标。

**详情：**
使用 `prometheus_client` 库的必需指标：

1. **Counter 指标：**
   - `model_api_requests_total{method, endpoint, status}` - 总请求数
   - `model_api_predictions_total{model_name, status}` - 总预测数

2. **Histogram 指标：**
   - `model_api_request_duration_seconds{method, endpoint}` - 请求延迟
   - `model_api_inference_duration_seconds{model_name}` - 模型推理时间

3. **Gauge 指标：**
   - `model_api_model_loaded{model_name, version}` - 模型加载状态
   - `model_api_active_connections` - 当前连接数

**验收标准：**
- `/metrics` 端点返回 Prometheus 格式
- 请求时指标正确递增
- 标签正确填充
- Histogram 桶覆盖预期的延迟范围（0.01 秒到 10 秒）

---

#### FR-5.4: 告警
**需求：** 配置关键条件的告警。

**详情：**
**必需告警：**

1. **高错误率：**
   ```yaml
   - alert: HighErrorRate
     expr: rate(model_api_requests_total{status=~"5.."}[5m]) > 0.05
     for: 2m
     annotations:
       summary: "检测到高错误率"
   ```

2. **Pod 崩溃循环：**
   ```yaml
   - alert: PodCrashLooping
     expr: rate(kube_pod_container_status_restarts_total{pod=~"model-api.*"}[15m]) > 0
     for: 5m
   ```

3. **高内存使用：**
   ```yaml
   - alert: HighMemoryUsage
     expr: container_memory_usage_bytes{pod=~"model-api.*"} / container_spec_memory_limit_bytes > 0.9
     for: 5m
   ```

**验收标准：**
- 条件满足时告警正确触发
- 发送告警通知（邮件、Slack、PagerDuty）
- 条件消除时告警自动解决
- 告警历史在 Prometheus UI 中可见

---

## 非功能需求

### NFR-1: 性能

#### NFR-1.1: 吞吐量
**需求：** 集群整体支持至少每秒 1000 个请求。

**测量：**
```bash
# 使用 k6 负载测试
k6 run --vus 100 --duration 5m load-test.js
```

**验收标准：**
- 持续 5 分钟保持 1000 RPS
- P95 延迟 < 500ms
- 错误率 < 1%
- 测试期间无 Pod 重启

---

#### NFR-1.2: 延迟
**需求：** 在负载下保持低延迟。

**目标：**
- **P50（中位数）：** < 100ms
- **P95：** < 300ms
- **P99：** < 500ms
- **P99.9：** < 1000ms

**测量：** 使用 Grafana histogram 查询 `model_api_request_duration_seconds`

---

#### NFR-1.3: 自动伸缩响应性
**需求：** 自动伸缩快速响应负载变化。

**目标：**
- **扩容：** 超过阈值后 2 分钟内新 Pod 就绪
- **缩容：** 5 分钟稳定窗口后移除 Pod
- **HPA 决策：** 每 15 秒评估指标

---

### NFR-2: 可靠性

#### NFR-2.1: 可用性
**需求：** 99.9% 正常运行时间（每月允许 43 分钟停机）。

**计算：**
```
正常运行时间 % = (总时间 - 停机时间) / 总时间 × 100
```

**要求：**
- 冗余副本防止单 Pod 故障
- 健康检查检测和恢复故障
- 滚动更新维持可用性

---

#### NFR-2.2: 零停机部署
**需求：** 所有部署必须在不中断服务的情况下完成。

**验证：**
- 部署期间持续监控请求
- 零 503 Service Unavailable 错误
- 保持 P99 延迟 < 500ms

---

#### NFR-2.3: 故障恢复
**需求：** Pod 故障自动恢复。

**目标：**
- **检测时间：** < 30 秒（通过存活探针）
- **恢复时间：** < 60 秒（Pod 重启 + 就绪）
- **总停机时间：** 每个 Pod 故障 < 90 秒

---

### NFR-3: 可伸缩性

#### NFR-3.1: 水平伸缩
**需求：** 支持在 3 到 10 个副本之间无缝伸缩。

**验证：**
- 测试手动伸缩：`kubectl scale deployment model-api --replicas=10`
- 在负载下测试自动伸缩
- 验证最大伸缩时无性能下降

---

#### NFR-3.2: 流量突增处理
**需求：** 优雅处理 10 倍流量突增。

**场景：** 从 100 RPS 突然增加到 1000 RPS

**预期行为：**
- HPA 在 2 分钟内扩容
- 扩容期间错误率保持在 < 5%
- 延迟暂时增加但会恢复
- 无 Pod 崩溃

---

### NFR-4: 安全性

#### NFR-4.1: Secrets 管理
**需求：** 所有敏感数据存储在 Kubernetes Secrets 中，而非 ConfigMap。

**示例：**
- API 密钥
- 数据库密码
- TLS 证书

**使用方式：**
```yaml
env:
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: api-secrets
        key: api-key
```

---

#### NFR-4.2: Pod 安全标准
**需求：** 应用 Pod 安全标准（受限配置文件）。

**详情：**
- 以非 root 用户运行
- 删除所有能力
- 只读根文件系统（尽可能）
- 不允许特权提升

---

#### NFR-4.3: 网络策略
**需求：** 使用 Network Policy 限制 Pod 间通信。

**规则：**
- 仅允许来自 Ingress 控制器的入站流量
- 仅允许出站到必需的服务
- 默认拒绝所有其他流量

---

## Kubernetes 特定需求

### KR-1: 命名空间隔离
**需求：** 将所有资源部署到专用命名空间 `ml-serving` 中。

**优势：**
- 资源隔离
- RBAC 范围限制
- 资源配额执行
- 更易清理

---

### KR-2: 标签与注解
**需求：** 使用一致的标签进行资源组织。

**标准标签：**
```yaml
labels:
  app: model-api
  version: v1.0
  component: inference
  managed-by: helm
  part-of: ml-platform
```

**标准注解：**
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "5000"
  prometheus.io/path: "/metrics"
```

---

### KR-3: ConfigMap 版本管理
**需求：** 对 ConfigMap 进行版本管理以触发滚动更新。

**策略：** 在 ConfigMap 名称中添加哈希并在 Deployment 中引用
```yaml
configMapRef:
  name: model-config-v2-abc123
```

**备选方案：** 使用 ConfigMap 卷和 Pod 注解

---

## 验收标准汇总

### 部署标准
- ✅ 3 个 Pod 运行中且就绪
- ✅ 健康检查通过
- ✅ 资源配置正确
- ✅ ConfigMap 加载正确

### 网络标准
- ✅ Service 内外均可访问
- ✅ Ingress 路由正确
- ✅ 跨所有 Pod 负载均衡
- ✅ DNS 解析正常工作

### 伸缩标准
- ✅ HPA 监控指标
- ✅ 负载下扩容（已验证）
- ✅ 空闲时缩容（已验证）
- ✅ 伸缩期间保持可用性

### 更新标准
- ✅ 滚动更新成功完成
- ✅ 零停机已验证
- ✅ 回滚正常工作
- ✅ 更新历史已维护

### 监控标准
- ✅ Prometheus 抓取指标
- ✅ Grafana 仪表盘显示数据
- ✅ 告警已配置并测试
- ✅ 应用指标已暴露

### 性能标准
- ✅ 持续 1000+ RPS
- ✅ P99 延迟 < 500ms
- ✅ 错误率 < 1%
- ✅ 自动伸缩响应 < 2 分钟

---

**文档版本：** 1.0
**最后更新：** 2025 年 10 月
**状态：** 最终版
