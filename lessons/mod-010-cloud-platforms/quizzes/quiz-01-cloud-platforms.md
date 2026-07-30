# Module 010 测验：云平台与 AWS

**时间限制**：90 分钟
**总题数**：50
**通过分数**：40/50（80%）
**格式**：选择题、多选题、场景题

---

## 说明

- 单选题选择**最佳**答案
- 多选题（标记为 [SELECT ALL]）选择**所有正确**答案
- 部分题目为场景题，需要应用知识
- 多选题不倒扣分
- 可使用草稿纸进行计算

---

## 第一部分：云基础（10 题）

### 第 1 题
哪种云服务模型让你对底层基础设施拥有**最大**的控制权？

A) Software as a Service（SaaS）
B) Platform as a Service（PaaS）
C) Infrastructure as a Service（IaaS）
D) Function as a Service（FaaS）

**答案**：C

**解析**：IaaS 提供虚拟机、存储和网络，让你完全控制操作系统和应用。PaaS 抽象了基础设施，SaaS 提供可直接使用的应用。

---

### 第 2 题
你的公司需要在本地数据中心和 AWS 之间部署应用，同时保持统一的管理界面。应该使用哪种云部署模型？

A) 公有云
B) 私有云
C) 混合云
D) 多云

**答案**：C

**解析**：混合云结合本地基础设施和公有云服务，允许工作负载在两者之间迁移。多云使用多个公有云提供商。

---

### 第 3 题
[SELECT ALL] 以下哪些是云计算相比本地基础设施的优势？

A) 资本支出（CapEx）替代运营支出（OpEx）
B) 能够根据需求上下调整资源
C) 无需容量规划
D) 只为使用的资源付费
E) 完全消除所有安全责任

**答案**：B、D

**解析**：云计算提供弹性（B）和按需付费定价（D）。但仍需要一定程度的容量规划（C 错误），云使用 OpEx 而非 CapEx（A 反了），你与提供商分担安全责任（E 错误 — 共享责任模型）。

---

### 第 4 题
一家初创公司希望最小化前期成本，只在应用活跃处理请求时才为计算资源付费。哪种服务模型**最**合适？

A) 专用服务器
B) 预留实例
C) 函数即服务（FaaS/无服务器）
D) 托管 Kubernetes 集群

**答案**：C

**解析**：FaaS（如 AWS Lambda）仅按实际执行时间计费，精确到毫秒，零前期成本，非常适合零星工作负载。

---

### 第 5 题
垂直扩展和水平扩展的主要区别是什么？

A) 垂直扩展增加更多服务器；水平扩展升级现有服务器
B) 垂直扩展比水平扩展更便宜
C) 垂直扩展增加现有服务器的资源；水平扩展增加更多服务器
D) 垂直扩展始终更适合云工作负载

**答案**：C

**解析**：垂直扩展（scale up）为现有服务器增加 CPU/内存。水平扩展（scale out）增加更多服务器。水平扩展在云环境中通常更受欢迎，具有更好的容错能力。

---

### 第 6 题
以下哪项是基础设施即代码（IaC）的例子？

A) 通过 AWS 控制台手动配置 EC2 实例
B) 使用 Terraform 定义和配置 AWS 资源
C) SSH 到服务器安装软件
D) 使用 AWS CloudFormation GUI 创建资源

**答案**：B

**解析**：IaC 使用代码（Terraform、CloudFormation 模板）定义基础设施。基于 GUI 的方式（A、D）和手动 SSH 配置（C）不是 IaC。

---

### 第 7 题
你的公司受到严格的数据驻留法规约束，要求所有数据必须保留在特定国家/地区内。哪个 AWS 功能可以帮助你合规？

A) 可用区
B) AWS 区域
C) 边缘节点
D) AWS Organizations

**答案**：B

**解析**：AWS 区域是地理上分离的位置。你可以选择特定区域以确保数据留在特定国家/地区内。可用区是区域内的，不是地理上分离的国家/地区。

---

### 第 8 题
AWS 可用区的主要目的是什么？

A) 减少全球用户的延迟
B) 在区域内提供容错能力
C) 实现多区域部署
D) 降低网络成本

**答案**：B

**解析**：可用区是区域内物理上分离的数据中心，提供冗余和容错能力。跨多个可用区部署可防止数据中心故障。

---

### 第 9 题
哪种云部署模型涉及使用多个云提供商（AWS、GCP、Azure）的服务？

A) 混合云
B) 多云
C) 私有云
D) 社区云

**答案**：B

**解析**：多云使用多个公有云提供商。混合云结合本地与云。私有云是专用基础设施，社区云由特定组织共享。

---

### 第 10 题
一家公司希望避免供应商锁定，确保其 ML 工作负载可以在任何云提供商上运行。哪种技术最**能**支持这一目标？

A) AWS SageMaker
B) Kubernetes（K8s）
C) Amazon ECS
D) AWS Lambda

**答案**：B

**解析**：Kubernetes 是一个开源容器编排平台，可以在任何云提供商或本地运行。SageMaker、ECS 和 Lambda 是 AWS 特定的服务。

---

## 第二部分：AWS 核心服务（12 题）

### 第 11 题
你需要在 AWS 上启动一台虚拟服务器来运行 Python 应用。应该使用哪个服务？

A) Amazon S3
B) Amazon EC2
C) AWS Lambda
D) Amazon RDS

**答案**：B

**解析**：Amazon EC2（Elastic Compute Cloud）提供虚拟机。S3 是对象存储，Lambda 是无服务器函数，RDS 是托管数据库。

---

### 第 12 题
哪种 EC2 定价模型为可以容忍中断的工作负载提供**最低**成本？

A) 按需实例
B) 预留实例
C) Spot 实例
D) 专用主机

**答案**：C

**解析**：Spot 实例相比按需实例可提供高达 90% 的折扣，但可能随时中断（提前 2 分钟通知）。非常适合容错工作负载，如批处理或 ML 训练。

---

### 第 13 题
[SELECT ALL] 以下哪些是 Amazon S3 的有效用例？

A) 托管静态网站内容
B) 存储 ML 训练数据集
C) 运行关系数据库
D) 存储应用日志
E) 托管 Docker 容器镜像

**答案**：A、B、D

**解析**：S3 是对象存储，适用于静态内容（A）、大型数据集（B）和日志（D）。不能运行数据库（C — 使用 RDS）。容器镜像应使用 ECR（E），不过技术上可以将它们存储在 S3 中。

---

### 第 14 题
你的应用需要一个具备自动备份、打补丁和 Multi-AZ 故障转移的 MySQL 数据库。应该使用哪个服务？

A) 在 EC2 上自行管理的 MySQL
B) Amazon RDS for MySQL
C) Amazon DynamoDB
D) 带版本控制的 Amazon S3

**答案**：B

**解析**：Amazon RDS 是一个托管关系数据库服务，自动处理备份、打补丁和 Multi-AZ 部署。DynamoDB 是 NoSQL，不是 MySQL。

---

### 第 15 题
使用 PUT 操作可以上传到 Amazon S3 的单个对象最大体积是多少？

A) 100 MB
B) 5 GB
C) 5 TB
D) 无限制

**答案**：B

**解析**：单次 PUT 操作限制为 5 GB。对于更大的对象（最多 5 TB），必须使用分段上传。

---

### 第 16 题
EC2 实例应该使用哪个 IAM 实体来访问 AWS 服务（如 S3），而无需在代码中嵌入凭证？

A) 带访问密钥的 IAM 用户
B) IAM 角色
C) IAM 组
D) 根账户凭证

**答案**：B

**解析**：IAM 角色为 EC2 实例提供临时凭证。切勿使用根账户凭证或在代码中嵌入访问密钥。

---

### 第 17 题
你需要确保 S3 对象在 90 天后自动转移到更便宜的存储，并在 1 年后删除。应该使用什么？

A) S3 版本控制
B) S3 生命周期策略
C) S3 复制
D) S3 存储桶策略

**答案**：B

**解析**：S3 生命周期策略根据时间或其他条件自动将对象转移到不同的存储类或删除它们。

---

### 第 18 题
[SELECT ALL] 哪些 EBS 卷类型针对高 IOPS 工作负载（数据库）进行了优化？

A) General Purpose SSD（gp3）
B) Provisioned IOPS SSD（io2）
C) Throughput Optimized HDD（st1）
D) Cold HDD（sc1）
E) Magnetic（standard）

**答案**：B、（A 根据解释可能也正确）

**解析**：Provisioned IOPS SSD（io2/io1）专为需要一致 IOPS 的高性能数据库设计。General Purpose SSD（gp3/gp2）适用于大多数工作负载，但 io2 专门针对高 IOPS 进行了优化。HDD 适用于吞吐量，而非 IOPS。

**注意**：B 是最佳答案，但 A（gp3）可以提供高达 16000 的 IOPS，对许多数据库来说已经足够。

---

### 第 19 题
Amazon ECR（Elastic Container Registry）的主要目的是什么？

A) 运行容器编排
B) 存储和管理 Docker 镜像
C) 管理 Kubernetes 集群
D) 提供无服务器函数

**答案**：B

**解析**：Amazon ECR 是一个托管的 Docker 容器镜像仓库，用于存储、管理和部署容器镜像。ECS/EKS 运行容器，ECR 不运行容器。

---

### 第 20 题
哪种 IAM 最佳实践为你的 AWS 根账户提供**最**强的安全性？

A) 将其用于日常管理任务
B) 与团队共享凭证
C) 启用 MFA 并创建 IAM 用户用于日常任务
D) 创建多个根账户

**答案**：C

**解析**：在根账户上启用 MFA，然后创建 IAM 用户/角色用于日常操作。切勿将根账户用于日常任务或共享凭证。

---

### 第 21 题
你需要根据 CPU 利用率自动扩缩 EC2 实例。应该使用哪个服务？

A) Elastic Load Balancer
B) Auto Scaling Groups
C) Amazon CloudWatch
D) AWS Lambda

**答案**：B

**解析**：Auto Scaling Groups 根据 CPU 利用率等指标自动添加/移除 EC2 实例。CloudWatch 提供指标，但 ASG 执行扩缩容操作。

---

### 第 22 题
当 EC2 实例停止时，存储在实例存储卷上的数据会发生什么？

A) 数据保留但实例无法重新启动
B) 数据自动备份到 S3
C) 数据永久丢失
D) 数据被移动到 EBS 卷

**答案**：C

**解析**：实例存储是附加到物理主机上的临时存储。实例停止或故障时数据丢失。对于持久性数据，请使用 EBS。

---

## 第三部分：网络与安全（12 题）

### 第 23 题
AWS VPC 的有效 IPv4 CIDR 块范围是多少？

A) /8 到 /32
B) /16 到 /28
C) /0 到 /32
D) /10 到 /24

**答案**：B

**解析**：AWS VPC CIDR 块必须在 /16（65536 个 IP）到 /28（16 个 IP）之间。这平衡了灵活性和避免 IP 浪费。

---

### 第 24 题
哪个组件允许私有子网中的资源访问互联网以获取软件更新？

A) Internet Gateway
B) NAT Gateway
C) VPC Peering
D) Virtual Private Gateway

**答案**：B

**解析**：NAT Gateway（或 NAT Instance）允许私有子网资源发起出站互联网连接，同时阻止入站访问。Internet Gateway 用于公有子网。

---

### 第 25 题
[SELECT ALL] 关于安全组，哪些说法是**正确**的？

A) 安全组是有状态的
B) 安全组支持 ALLOW 和 DENY 规则
C) 安全组在决定允许流量之前评估所有规则
D) 安全组可以引用其他安全组
E) 安全组在子网级别操作

**答案**：A、D

**解析**：安全组是有状态的（A）—— 返回流量自动允许。它们只支持 ALLOW 规则，不支持 DENY（B 错误）。如果有任何规则匹配则允许流量（C 错误）。它们可以引用其他安全组（D）。它们在实例级别操作，而非子网级别（E 错误 — NACL 是子网级别）。

---

### 第 26 题
安全组和 Network ACL 的主要区别是什么？

A) 安全组是无状态的；NACL 是有状态的
B) 安全组在实例级别操作；NACL 在子网级别操作
C) 安全组支持 DENY 规则；NACL 不支持
D) NACL 比安全组更安全

**答案**：B

**解析**：安全组是实例级别的有状态防火墙。NACL 是子网级别的无状态防火墙。两者都提供安全性，一起使用可实现纵深防御。

---

### 第 27 题
你希望仅允许从公有子网中的堡垒机通过 SSH 访问 EC2 实例。应该如何配置安全组？

A) 允许来自 0.0.0.0/0 的 SSH（端口 22）
B) 允许来自堡垒机公有 IP 的 SSH（端口 22）
C) 允许来自堡垒机安全组的 SSH（端口 22）
D) 允许来自堡垒机的所有流量

**答案**：C

**解析**：引用堡垒机的安全组（而非 IP 地址）更灵活、更安全。如果堡垒机的 IP 发生变化，规则仍然有效。选项 D 过于宽松。

---

### 第 28 题
哪个 VPC 组件为公有子网中的资源提供 VPC 与互联网之间的连接？

A) NAT Gateway
B) Internet Gateway
C) Virtual Private Gateway
D) VPC Endpoint

**答案**：B

**解析**：Internet Gateway 为公有子网提供双向互联网连接。NAT Gateway 用于私有子网（仅出站）。VPG 用于 VPN 连接。

---

### 第 29 题
VPC Endpoint 的目的是什么？

A) 连接到本地网络
B) 私有地访问 AWS 服务，无需经过互联网或 NAT Gateway
C) 启用 VPC 对等连接
D) 提供公有 IP 地址

**答案**：B

**解析**：VPC Endpoint 允许私有连接到 AWS 服务（如 S3、DynamoDB），无需使用 Internet Gateway 或 NAT Gateway，从而降低成本并提高安全性。

---

### 第 30 题
[SELECT ALL] 以下哪些可以被 VPC Flow Logs 记录？

A) 接受的流量
B) 拒绝的流量
C) 应用层负载（HTTP 请求体）
D) 源和目标 IP 地址
E) 传输的字节数

**答案**：A、B、D、E

**解析**：VPC Flow Logs 捕获 IP 流量的元数据（A、B、D、E），但**不**捕获数据包内容或应用负载（C 错误）。

---

### 第 31 题
你需要确保私有子网中的数据库服务器**永远不能**发起出站互联网连接。应该怎么做？

A) 移除 NAT Gateway
B) 使用 Network ACL 阻止所有出站流量
C) 删除 Internet Gateway
D) 从私有子网的路由表中移除指向 0.0.0.0/0 的路由

**答案**：D

**解析**：从私有子网的路由表中移除默认路由（0.0.0.0/0 → NAT Gateway）可阻止出站互联网访问。NAT Gateway 可以保留供其他子网使用。

---

### 第 32 题
哪个 AWS 服务为你的应用提供分布式拒绝服务（DDoS）保护？

A) AWS WAF
B) AWS Shield
C) 安全组
D) Network ACL

**答案**：B

**解析**：AWS Shield 提供 DDoS 保护。Shield Standard 是自动和免费的。Shield Advanced 提供额外的保护（需付费）。WAF 是一个用于第 7 层攻击的 Web 应用防火墙。

---

### 第 33 题
一个安全组最多可以有多少条规则？

A) 50 条入站、50 条出站
B) 60 条入站、60 条出站（默认配额）
C) 100 条入站、100 条出站
D) 无限制

**答案**：B

**解析**：默认配额是每个安全组 60 条入站和 60 条出站规则。可以通过向 AWS 申请提高配额来增加。

---

### 第 34 题
你希望只允许从办公室（203.0.113.0/24）SSH 访问堡垒机，但阻止所有其他 SSH 流量。应该创建哪条安全组规则？

A) 允许来自 0.0.0.0/0 的 TCP 端口 22
B) 允许来自 203.0.113.0/24 的 TCP 端口 22
C) 拒绝除 203.0.113.0/24 之外的所有 TCP 端口 22
D) 允许来自 203.0.113.0/24 的 TCP 端口 443

**答案**：B

**解析**：安全组只支持 ALLOW 规则（不支持 DENY），因此为你的办公室 CIDR 块创建允许规则。所有其他流量都被隐式拒绝。

---

## 第四部分：容器编排（8 题）

### 第 35 题
Amazon ECS 和 Amazon EKS 的主要区别是什么？

A) ECS 用于 Docker 容器；EKS 用于虚拟机
B) ECS 是 AWS 原生服务；EKS 使用 Kubernetes
C) ECS 比 EKS 更贵
D) EKS 不能与 AWS 服务集成

**答案**：B

**解析**：ECS 是 AWS 专有的容器编排服务。EKS 是 AWS 托管的 Kubernetes 服务。两者都运行 Docker 容器并与 AWS 服务集成。EKS 有额外的控制平面费用（0.10 美元/小时）。

---

### 第 36 题
哪种 ECS 启动类型无需管理底层 EC2 实例？

A) EC2 启动类型
B) Fargate 启动类型
C) Spot 启动类型
D) Lambda 启动类型

**答案**：B

**解析**：AWS Fargate 是容器的无服务器计算。你无需配置或管理服务器 — AWS 处理基础设施。EC2 启动类型需要管理 EC2 实例。

---

### 第 37 题
[SELECT ALL] 在 ECS 上运行容器需要哪些组件？

A) ECS 集群
B) 任务定义
C) Kubernetes Master 节点
D) ECS 服务（用于长期运行的任务）
E) ECR 仓库

**答案**：A、B、D

**解析**：ECS 需要一个集群（A）、任务定义（B），通常还需要一个服务用于长期运行的任务（D）。Kubernetes 组件（C）用于 EKS，而非 ECS。ECR（E）是推荐但不是必需的 — 你可以使用 Docker Hub 或其他镜像仓库。

---

### 第 38 题
什么是 Kubernetes Pod？

A) 单个 Docker 容器
B) 最小的可部署单元，包含一个或多个容器
C) 一组工作节点
D) 负载均衡器

**答案**：B

**解析**：Pod 是 Kubernetes 最小的可部署单元，可以包含一个或多个紧密耦合的容器，它们共享网络和存储。最常情况下，一个 pod 包含单个容器。

---

### 第 39 题
在 Kubernetes 中，哪个组件根据 CPU 利用率或自定义指标自动扩缩 pod 数量？

A) Deployment
B) Service
C) Horizontal Pod Autoscaler（HPA）
D) Ingress Controller

**答案**：C

**解析**：Horizontal Pod Autoscaler（HPA）根据观察到的指标（如 CPU、内存或自定义指标）自动调整 pod 数量。Deployment 管理 pod 副本，但 HPA 控制扩缩容。

---

### 第 40 题
你想部署一个容器化的 ML 推理服务，该服务根据 CPU 利用率自动从 2 个容器扩展到 10 个。为了最简单的设置，你会使用哪个 AWS 服务？

A) Amazon ECS with Fargate 和 Application Auto Scaling
B) 在 EC2 上自行管理的 Kubernetes
C) AWS Lambda
D) 带 Docker 的 EC2 实例

**答案**：A

**解析**：ECS Fargate 配合 Application Auto Scaling 提供了最简单且带自动扩缩容的容器化部署。无需管理服务器。Lambda 不支持自定义容器（在所暗示的规模下），自行管理的 K8s 更复杂。

---

### 第 41 题
Kubernetes Service 的目的是什么？

A) 运行容器工作负载
B) 为 pod 提供稳定的网络端点
C) 存储容器镜像
D) 管理集群节点

**答案**：B

**解析**：Kubernetes Services 为访问 pod 提供稳定的 DNS 名称和 IP 地址，即使在 pod 被创建和销毁时也是如此。它们实现了负载均衡和服务发现。

---

### 第 42 题
哪种类型的 Kubernetes Service 将你的应用暴露到互联网？

A) ClusterIP
B) NodePort
C) LoadBalancer
D) ExternalName

**答案**：C

**解析**：LoadBalancer 类型创建一个外部负载均衡器（如 AWS ALB/NLB），并将服务暴露到互联网。ClusterIP 仅内部可用，NodePort 暴露在节点 IP 上（不推荐用于面向生产环境的互联网应用）。

---

## 第五部分：ML 工作负载与 SageMaker（6 题）

### 第 43 题
哪个 AWS 服务为训练和部署 ML 模型提供完全托管的基础设施？

A) Amazon EC2
B) AWS Lambda
C) Amazon SageMaker
D) Amazon ECS

**答案**：C

**解析**：Amazon SageMaker 是 AWS 的完全托管 ML 平台，处理训练、调优和部署模型。EC2/ECS 需要手动设置。

---

### 第 44 题
在 SageMaker 训练作业中使用 Spot 实例的主要优势是什么？

A) 更快的训练时间
B) 更好的模型精度
C) 相比按需实例高达 90% 的成本节省
D) 保证的可用性

**答案**：C

**解析**：Spot 实例可大幅节省成本（高达 90%），但可能被中断。SageMaker 支持 checkpointing 以恢复中断的训练作业，使 Spot 适用于训练。

---

### 第 45 题
哪个 SageMaker 功能自动为你的模型搜索最佳超参数？

A) SageMaker Training Jobs
B) SageMaker Hyperparameter Tuning（HPO）
C) SageMaker Endpoints
D) SageMaker Pipelines

**答案**：B

**解析**：SageMaker Hyperparameter Tuning（也称为 Automatic Model Tuning）运行多个训练作业，使用不同的超参数组合来找到最佳模型。

---

### 第 46 题
[SELECT ALL] 哪些组件是 SageMaker 训练作业的一部分？

A) 算法（内置或自定义）
B) 计算资源（实例类型和数量）
C) 输入数据位置（S3）
D) 输出模型产物位置（S3）
E) 用于推理的正在运行的端点

**答案**：A、B、C、D

**解析**：训练作业需要算法（A）、计算资源（B）、输入数据（C）和输出位置（D）。端点（E）是单独的 — 在训练后创建以提供预测服务。

---

### 第 47 题
你的 SageMaker 端点接收不可预测的流量，长时间没有请求。最**具成本效益**的部署选项是什么？

A) 使用最大的实例类型来处理峰值负载
B) 部署最少 10 个实例以实现高可用
C) 使用 SageMaker Serverless Inference
D) 在 ml.m5.large 上 24/7 运行 SageMaker 端点

**答案**：C

**解析**：SageMaker Serverless Inference 在空闲期间自动缩放到零，并根据流量扩展。你只需在推理期间为计算时间付费，非常适合间歇性工作负载。

---

### 第 48 题
SageMaker Pipelines 的目的是什么？

A) 运行 Kubernetes 工作负载
B) 编排端到端 ML 工作流（数据准备、训练、部署）
C) 提供 VPN 连接
D) 管理 IAM 权限

**答案**：B

**解析**：SageMaker Pipelines 是一个用于 MLOps 的工作流编排服务，允许你定义从数据处理到模型部署的可重现 ML 工作流。

---

## 第六部分：成本优化与最佳实践（6 题）

### 第 49 题
[SELECT ALL] 哪些策略可以降低 ML 工作负载的 AWS 成本？

A) 使用 Spot 实例进行训练作业
B) 实施自动扩缩容以匹配需求
C) 删除未使用的资源（已停止的 EC2、旧快照）
D) 在所有工作负载上使用最大的实例类型
E) 使用 S3 生命周期策略将旧数据转移到更便宜的存储类

**答案**：A、B、C、E

**解析**：Spot 实例（A）、自动扩缩容（B）、删除未使用的资源（C）和 S3 生命周期策略（E）都能降低成本。使用最大的实例（D）通常会造成浪费，除非充分利用。

---

### 第 50 题
你需要按项目和环境（开发/生产）跟踪 AWS 成本。应该实施什么？

A) 为每个项目创建单独的 AWS 账户
B) 在所有资源上使用成本分配标签
C) 监控 CloudWatch 指标
D) 启用 AWS Budgets

**答案**：B

**解析**：成本分配标签（如 Project、Environment、CostCenter）允许你在 AWS Cost Explorer 中分类和跟踪支出。Budgets（D）用于告警，而非分类。单独的账户（A）比必要更复杂。

---

### 附加题 51（隐藏题）
一名初级工程师意外删除了生产数据库。哪个 AWS 服务本可以防止永久性数据丢失？

A) AWS CloudTrail
B) 自动 RDS 快照
C) AWS Config
D) IAM 权限

**答案**：B

**解析**：自动 RDS 快照会创建可恢复的定期备份。CloudTrail 记录 API 调用但不能防止数据丢失。Config 跟踪配置变化。IAM 应防止未经授权的删除，但不能备份数据。

---

## 答案汇总

| 题号 | 答案 | 题号 | 答案 | 题号 | 答案 | 题号 | 答案 |
|------|------|------|------|------|------|------|------|
| 1  | C      | 14 | B      | 27 | C      | 40 | A      |
| 2  | C      | 15 | B      | 28 | B      | 41 | B      |
| 3  | B,D    | 16 | B      | 29 | B      | 42 | C      |
| 4  | C      | 17 | B      | 30 | A,B,D,E| 43 | C      |
| 5  | C      | 18 | B      | 31 | D      | 44 | C      |
| 6  | B      | 19 | B      | 32 | B      | 45 | B      |
| 7  | B      | 20 | C      | 33 | B      | 46 | A,B,C,D|
| 8  | B      | 21 | B      | 34 | B      | 47 | C      |
| 9  | B      | 22 | C      | 35 | B      | 48 | B      |
| 10 | B      | 23 | B      | 36 | B      | 49 | A,B,C,E|
| 11 | B      | 24 | B      | 37 | A,B,D  | 50 | B      |
| 12 | C      | 25 | A,D    | 38 | B      | 51 | B      |
| 13 | A,B,D  | 26 | B      | 39 | C      |    |        |

---

## 评分指南

- **45-50 正确（90-100%）**：优秀！你已经掌握了云平台和 AWS。
- **40-44 正确（80-88%）**：通过 — 理解良好，存在少量差距。
- **35-39 正确（70-78%）**：复习材料，重新参加测验。
- **低于 35（< 70%）**：在重考之前重新学习讲座和练习。

---

## 按部分的复习主题

如果你在某一部分得分不佳，请复习：

**第一部分（云基础）**：讲座 01
**第二部分（AWS 核心服务）**：讲座 02、练习 01-02
**第三部分（网络与安全）**：讲座 03、练习 03
**第四部分（容器）**：讲座 04、练习 04
**第五部分（ML/SageMaker）**：讲座 04、练习 05
**第六部分（成本优化）**：练习 05

---

## 祝贺！

如果你通过了本次测验，说明你已经成功完成了 **Module 010：云平台**以及整个**初级 AI 基础设施工程师学习路径**（模块 001-010）！

🎉 **你已经准备好从事生产级 AI 基础设施工作了！**

**总学习时长**：10 个模块约 400+ 小时
**下一步**：申请初级 AI 基础设施工程师岗位，或继续学习 AI 基础设施工程师（中级）轨道！
