# Module 006: Kubernetes 入门 - 评估测验

## 说明

本测验包含 25 道题，涵盖 Module 006 的所有方面。请尽最大能力回答每道题。答案在最后提供。

**时间限制**：60 分钟（建议）
**及格分数**：80%（25 题中 20 题正确）

## 问题

### 第 1 部分：Kubernetes 架构（第 1-6 题）

**问题 1**：哪个 Kubernetes 组件负责将 Pod 调度到节点？
- A) kubelet
- B) kube-scheduler
- C) kube-controller-manager
- D) kube-proxy

**问题 2**：Kubernetes 将集群状态和配置存储在哪里？
- A) API server 的内存中
- B) etcd 中
- C) 关系型数据库中
- D) 每个工作节点上

**问题 3**：kubelet 的作用是什么？
- A) 将 Pod 调度到节点
- B) 运行管理集群状态的控制器
- C) 管理特定节点上的容器
- D) 实现 Service 网络

**问题 4**：哪个组件在每个节点上实现 Service 负载均衡？
- A) kube-proxy
- B) kubelet
- C) container runtime
- D) CoreDNS

**问题 5**：Kubernetes API Server 使用什么通信协议？
- A) gRPC
- B) REST (HTTP/HTTPS)
- C) WebSockets
- D) MQTT

**问题 6**：在 Helm 3 中，Helm 将 release 信息存储在哪里？
- A) 在独立的 Tiller 服务中
- B) 在 Kubernetes Secrets 中
- C) 直接在 etcd 中
- D) 仅在客户端机器上

### 第 2 部分：Pod 和 Deployment（第 7-11 题）

**问题 7**：Kubernetes 中最小的可部署单元是什么？
- A) Container
- B) Pod
- C) Deployment
- D) ReplicaSet

**问题 8**：一个 Pod 可以包含多少个容器？
- A) 恰好一个
- B) 一个或多个
- C) 至少两个
- D) 最多五个

**问题 9**：删除 Deployment 时会发生什么？
- A) 仅删除 Deployment 对象
- B) Deployment 和 ReplicaSet 被删除，但 Pod 保留
- C) Deployment、ReplicaSet 和 Pod 都被删除
- D) 什么都不会发生；Deployment 不能被删除

**问题 10**：liveness probe 的目的是什么？
- A) 判断容器是否应该接收流量
- B) 判断 Kubernetes 是否应该重启容器
- C) 检查容器是否已成功启动
- D) 监控资源使用情况

**问题 11**：在 maxUnavailable=1 和 maxSurge=1 的滚动更新中，从 3 个副本开始，更新期间的最大 Pod 数量是多少？
- A) 3
- B) 4
- C) 5
- D) 6

### 第 3 部分：Service 和网络（第 12-15 题）

**问题 12**：Kubernetes 中默认的 Service 类型是什么？
- A) NodePort
- B) LoadBalancer
- C) ClusterIP
- D) ExternalName

**问题 13**：Service 如何找到其后端 Pod？
- A) 通过 IP 地址
- B) 通过名称
- C) 通过 label selectors
- D) 通过 namespace

**问题 14**：NodePort 服务使用的端口范围是什么？
- A) 1-1024
- B) 8000-9000
- C) 30000-32767
- D) 40000-50000

**问题 15**：名为 "api" 的 service 在 "production" namespace 中的完整 DNS 名称是什么？
- A) api.production
- B) api.production.svc
- C) api.production.svc.cluster.local
- D) production.api.cluster.local

### 第 4 部分：配置和存储（第 16-19 题）

**问题 16**：ConfigMap 和 Secret 有什么区别？
- A) Secret 在持久化时加密，ConfigMap 默认不加密
- B) ConfigMap 可以比 Secret 更大
- C) Secret 只能存储字符串，ConfigMap 可以存储任何数据
- D) 没有区别；它们可以互换

**问题 17**：当你更新一个作为卷挂载到正在运行的 Pod 中的 ConfigMap 时会发生什么？
- A) Pod 自动重启
- B) 文件（最终）会在容器中更新
- C) 什么都不会发生；Pod 看不到 ConfigMap 的更新
- D) 容器崩溃

**问题 18**：哪种访问模式允许多个节点挂载卷进行读写？
- A) ReadWriteOnce (RWO)
- B) ReadOnlyMany (ROX)
- C) ReadWriteMany (RWX)
- D) ReadWriteMulti (RWM)

**问题 19**：当 PersistentVolumeClaim 被创建时，什么会自动创建 PersistentVolume？
- A) kubelet
- B) 带有动态 provisioner 的 StorageClass
- C) PersistentVolume controller
- D) Volume plugin

### 第 5 部分：Helm（第 20-22 题）

**问题 20**：Helm chart 中哪个文件包含默认配置值？
- A) Chart.yaml
- B) values.yaml
- C) defaults.yaml
- D) config.yaml

**问题 21**：什么是 Helm release？
- A) Helm 软件的版本
- B) 安装在集群中的 chart 实例
- C) 打包的 Helm chart
- D) Helm 仓库

**问题 22**：如何将 Helm release 回滚到上一个版本？
- A) `helm undo`
- B) `helm revert`
- C) `helm rollback`
- D) `helm restore`

### 第 6 部分：运维和调试（第 23-25 题）

**问题 23**：哪个 kubectl 命令显示资源的详细信息，包括事件？
- A) kubectl get
- B) kubectl explain
- C) kubectl describe
- D) kubectl logs

**问题 24**：Pod 处于 "CrashLoopBackOff" 状态。哪个命令显示容器崩溃的原因？
- A) `kubectl logs <pod>`
- B) `kubectl logs <pod> --previous`
- C) `kubectl describe pod <pod>`
- D) B 和 C 都可以

**问题 25**：命令 `kubectl top pods` 显示什么？
- A) 列表中的前几个 Pod
- B) 当前的 CPU 和内存使用量
- C) Pod 优先级级别
- D) 重启次数最多的 Pod

---

## 答案

### 第 1 部分：Kubernetes 架构
1. **B** - kube-scheduler 负责将 Pod 分配到节点
2. **B** - etcd 使用 Raft 共识存储所有集群状态
3. **C** - kubelet 管理每个节点上的容器和 Pod 生命周期
4. **A** - kube-proxy 实现 Service 网络和负载均衡
5. **B** - API server 使用 REST (HTTP/HTTPS) 进行通信
6. **B** - Helm 3 将 release 作为 Secrets 存储在 Kubernetes 中

### 第 2 部分：Pod 和 Deployment
7. **B** - Pod 是最小的可部署单元
8. **B** - Pod 可以包含一个或多个容器
9. **C** - 删除 Deployment 会级联删除 ReplicaSet 和 Pod
10. **B** - Liveness probe 判断容器是否应该被重启
11. **B** - maxSurge=1 允许最多 4 个 Pod (3 + 1)，maxUnavailable=1 意味着至少 2 个在运行

### 第 3 部分：Service 和网络
12. **C** - ClusterIP 是默认的 Service 类型
13. **C** - Service 使用 label selectors 查找 Pod
14. **C** - NodePort 使用端口范围 30000-32767
15. **C** - 完整 DNS 名称格式：`<service>.<namespace>.svc.cluster.local`

### 第 4 部分：配置和存储
16. **A** - Secret 支持持久化加密，ConfigMap 默认不支持
17. **B** - 挂载的 ConfigMap 最终会在容器中更新
18. **C** - ReadWriteMany (RWX) 允许多节点读写访问
19. **B** - 带有动态 provisioner 的 StorageClass 会自动创建 PV

### 第 5 部分：Helm
20. **B** - values.yaml 包含默认配置值
21. **B** - Release 是安装在集群中的 chart 实例
22. **C** - `helm rollback` 回滚到之前的版本

### 第 6 部分：运维和调试
23. **C** - `kubectl describe` 显示包括事件在内的详细信息
24. **D** - `kubectl logs --previous` 和 `kubectl describe` 都有助于识别崩溃原因
25. **B** - `kubectl top pods` 显示当前的 CPU 和内存使用量

---

## 评分指南

- **25 正确 (100%)**：优秀！你已经掌握了 Kubernetes 基础知识
- **23-24 正确 (92-96%)**：非常好！建议做少量复习
- **20-22 正确 (80-88%)**：良好！复习你答错的题目
- **17-19 正确 (68-76%)**：一般！复习讲义和练习
- **低于 17 (< 68%)**：需要改进！复习所有材料

## 按部分划分的复习建议

**如果你在第 1 部分（架构）答错了问题**：
- 重新阅读 Lecture 01：Kubernetes 架构
- 复习控制平面和工作节点组件
- 理解每个组件的作用

**如果你在第 2 部分（Pod/Deployment）答错了问题**：
- 重新阅读 Lecture 02：部署应用
- 再次完成 Exercise 01
- 重点关注 Pod 生命周期和健康检查

**如果你在第 3 部分（Service）答错了问题**：
- 复习 Service 类型和 DNS 命名
- 练习创建 Service
- 理解 label selectors 和 endpoints

**如果你在第 4 部分（配置/存储）答错了问题**：
- 复习 ConfigMap、Secret 和 Volume
- 理解访问模式和动态 provisioning
- 练习挂载 ConfigMap 和 Secret

**如果你在第 5 部分（Helm）答错了问题**：
- 重新阅读 Lecture 03：Helm
- 再次完成 Exercise 02
- 练习使用 Helm 命令

**如果你在第 6 部分（运维）答错了问题**：
- 重新阅读 Lecture 04：Kubernetes 运维
- 再次完成 Exercise 03
- 练习调试工作流程

---

## 额外练习题

想要更多练习，试试这些附加题：

1. 解释 Deployment 和 StatefulSet 的区别
2. 描述从 Service 请求到 Pod 的完整流程
3. maxSurge=0 和 maxUnavailable=1 的滚动更新期间会发生什么？
4. 你会如何调试卡在 "Pending" 状态的 Pod？
5. `kubectl apply` 和 `kubectl create` 的区别是什么？
6. 如何在不重启 Pod 的情况下更新 Secret？
7. Ingress 资源的目的是什么？
8. Kubernetes 如何处理节点故障？
9. DaemonSet 和 Deployment 的区别是什么？
10. 如何确保两个 Pod 永远不会运行在同一节点上？

---

## 后续步骤

- **如果你通过了 (80%+)**：继续学习 Module 007（高级 Kubernetes）
- **如果你需要复习**：专注于薄弱领域并重新参加测验
- **所有学生**：如果还未完成，完成全部三个练习
- **额外挑战**：设置一个多节点集群并练习所有场景

## 反馈

跟踪你的测验结果：

```
考试日期：__________
分数：_____/25 (_____%)
用时：_____ 分钟

需要复习的领域：
- [ ] 架构
- [ ] Pod 和 Deployment
- [ ] Service 和网络
- [ ] 配置和存储
- [ ] Helm
- [ ] 运维和调试

备注：
_________________________________
_________________________________
_________________________________
```

---

**恭喜你完成了 Module 006 测验！** 本次评估验证了你对 AI 基础设施工作至关重要的 Kubernetes 基础知识的理解。

继续练习，记住：动手经验比死记硬背答案更有价值！
