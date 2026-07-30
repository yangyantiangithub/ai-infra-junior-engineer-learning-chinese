# Module 010: 云平台基础

## 模块概述

云计算是现代 AI/ML 基础设施的基础。本模块提供云平台基础的综合培训，聚焦主流云服务商（AWS、Google Cloud、Azure），并以 AWS 为实操重点。你将学习核心云服务、基础设施管理、成本优化，以及如何将应用和 ML 工作负载部署到云端。

你将了解云计算模型、虚拟机、存储系统、网络、身份管理和无服务器计算。这些技能对于构建可扩展、弹性的 AI 基础设施和管理生产级 ML 系统至关重要。

完成本模块后，你将能够自信地配置云资源、部署应用、管理云基础设施，并将云服务应用于 AI/ML 工作流。

## 学习目标

完成本模块后，你将能够：

1. **理解云计算基础**，包括 IaaS、PaaS、SaaS 模型
2. **掌握主流云平台**（AWS、GCP、Azure）及其服务
3. **管理计算资源**，包括 EC2、虚拟机和容器
4. **配置云存储**，包括对象存储、块存储和数据库
5. **搭建云网络**，包括 VPC、子网、安全组和负载均衡器
6. **实现 IAM**（身份和访问管理）保障安全
7. **使用基础设施即代码**，如 Terraform 等工具
8. **优化云成本**并理解定价模型
9. **部署应用到云端**，包括容器化和无服务器应用
10. **利用云服务支持 ML**，包括托管 ML 平台和 GPU 实例

## 前置要求

- 需要完成模块 002（Linux 基础）
- 建议完成模块 005（Docker 和容器）
- 建议完成模块 003（Git 版本控制）
- 基础网络知识
- 理解虚拟化概念

**推荐配置：**
- AWS 免费套餐账户（主要）
- Google Cloud 免费套餐账户（可选）
- Azure 免费套餐账户（可选）
- 安装并配置 AWS CLI
- 安装 Terraform
- 生成 SSH 密钥对
- 信用卡用于账户验证（免费套餐不会收费）

## 时间投入

- **预计总时间：** 45-55 小时
- **讲座与阅读：** 18-22 小时
- **动手练习：** 22-28 小时
- **项目：** 5-8 小时

**推荐节奏：**
- 兼职（5-10 小时/周）：5-7 周
- 全职（20-30 小时/周）：2-3 周

云专业知识需要大量的动手实践。预计需要在云控制台和服务实验中投入大量时间。

## 模块结构

### 第 1-2 周：云基础与计算
- **主题：** 云概念、计算服务、虚拟机
- **关键技能：** EC2 管理、实例类型、SSH 访问
- **实践：** 启动实例、管理计算资源

### 第 3 周：存储与数据库
- **主题：** 对象存储、块存储、托管数据库
- **关键技能：** S3、EBS、RDS 配置与管理
- **实践：** 数据存储场景、数据库部署

### 第 4 周：网络与安全
- **主题：** VPC、子网、安全组、IAM
- **关键技能：** 网络配置、安全最佳实践
- **实践：** 安全网络架构、IAM 策略

### 第 5 周：部署与 ML 服务
- **主题：** IaC、容器服务、ML 平台、成本优化
- **关键技能：** Terraform、ECS/EKS、SageMaker 基础、账单
- **实践：** 基础设施自动化、ML 工作负载部署

## 详细主题分解

### 1. 云计算基础（5-6 小时）

#### 1.1 云计算简介
- 什么是云计算？
- 云的历史与演进
- 云计算的优势
- 云 vs 本地部署
- 上云考量
- 云原生应用
- 多云与混合云

#### 1.2 云服务模型
- 基础设施即服务（IaaS）
- 平台即服务（PaaS）
- 软件即服务（SaaS）
- 函数即服务（FaaS）
- 选择合适模型
- 服务模型示例
- 权衡与考量

#### 1.3 云部署模型
- 公有云
- 私有云
- 混合云
- 多云策略
- 边缘计算
- 部署模型选择
- 合规与监管考量

#### 1.4 主要云服务商概览
- Amazon Web Services（AWS）
- Google Cloud Platform（GCP）
- Microsoft Azure
- 服务对比
- 优势与用例
- 供应商锁定考量
- 认证路径

#### 1.5 云经济学
- 按需付费定价
- 预留实例
- Spot/抢占式实例
- 基于承诺的折扣
- 总拥有成本（TCO）
- 云成本优化
- FinOps 原则

### 2. AWS 入门（6-7 小时）

#### 2.1 AWS 基础
- AWS 账户创建
- AWS Management Console
- AWS 区域和可用区
- AWS 全球基础设施
- AWS 服务分类
- AWS 免费套餐
- 账户安全基础

#### 2.2 AWS CLI 和 SDK
- 安装 AWS CLI
- 配置凭证
- AWS CLI 基础
- 常用 CLI 命令
- AWS SDK（Python 的 boto3）
- 编程访问
- CLI 脚本

#### 2.3 AWS 身份和访问管理（IAM）
- IAM 基础
- 用户、组和角色
- IAM 策略
- 策略语法和结构
- 权限边界
- 最佳实践（最小权限）
- MFA（多因素认证）
- 服务角色

#### 2.4 AWS 资源组织
- AWS Organizations
- 服务控制策略（SCP）
- 资源标签
- 成本分配标签
- 账户结构策略
- 账单和成本管理
- AWS Resource Groups

#### 2.5 AWS 支持与文档
- AWS 文档
- AWS 支持层级
- Trusted Advisor
- Personal Health Dashboard
- AWS 论坛和社区
- Architecture Center
- Well-Architected Framework

### 3. 计算服务（8-10 小时）

#### 3.1 Amazon EC2 基础
- EC2 概述与概念
- 实例类型与系列
- AMI（Amazon Machine Images）
- 启动实例
- 实例生命周期
- 实例元数据
- User data 脚本
- EC2 定价模型

#### 3.2 EC2 管理
- 连接实例（SSH）
- 安全组
- 密钥对
- 弹性 IP
- 实例状态
- 使用 CloudWatch 监控
- EC2 标签与组织
- 终止保护

#### 3.3 EC2 实例类型
- 通用型（T、M 系列）
- 计算优化型（C 系列）
- 内存优化型（R、X 系列）
- 加速计算型（用于 ML 的 P、G 系列）
- 存储优化型（I、D 系列）
- 选择实例类型
- 实例规格
- 成本与性能权衡

#### 3.4 EC2 存储选项
- 实例存储
- EBS（Elastic Block Store）
- EBS 卷类型（gp3、io2、st1、sc1）
- EBS 快照
- EBS 加密
- 卷挂载与管理
- 快照策略

#### 3.5 Auto Scaling
- Auto Scaling 概念
- Launch templates
- Auto Scaling 组
- 扩缩容策略
- 目标跟踪扩缩容
- 定时扩缩容
- 健康检查
- 与负载均衡器集成

#### 3.6 负载均衡
- Elastic Load Balancing（ELB）
- Application Load Balancer（ALB）
- Network Load Balancer（NLB）
- Gateway Load Balancer
- 目标组
- 健康检查
- SSL/TLS 终止
- 负载均衡器监控

#### 3.7 容器服务
- Amazon ECS（Elastic Container Service）
- ECS 概念（集群、任务、服务）
- Fargate vs EC2 启动类型
- Amazon EKS（Elastic Kubernetes Service）
- 容器镜像仓库（ECR）
- 何时使用容器
- 容器编排对比

#### 3.8 无服务器计算
- AWS Lambda 基础
- Lambda 函数
- 触发器与事件
- Lambda 定价
- 无服务器用例
- Lambda 限制
- 无服务器框架
- Lambda 用于 ML 推理

### 4. 存储服务（6-8 小时）

#### 4.1 Amazon S3（Simple Storage Service）
- S3 概念（存储桶、对象、键）
- 创建存储桶
- 上传和下载对象
- S3 存储类
- 对象生命周期策略
- 版本控制
- S3 加密
- S3 访问控制

#### 4.2 S3 高级功能
- S3 静态网站托管
- S3 事件通知
- S3 Transfer Acceleration
- S3 Select
- S3 Batch Operations
- 分段上传
- 预签名 URL
- 用于数据湖的 S3

#### 4.3 EBS（Elastic Block Store）
- EBS 基础
- 卷类型与性能
- 创建和挂载卷
- EBS 快照
- 快照生命周期
- EBS 加密
- 卷扩容
- EBS 优化

#### 4.4 Amazon EFS（Elastic File System）
- EFS 概述
- 创建文件系统
- 挂载目标
- EFS 性能模式
- EFS 存储类
- EFS vs EBS vs S3
- EFS 用例
- 与容器配合使用的 EFS

#### 4.5 Storage Gateway
- Storage Gateway 概述
- File Gateway
- Volume Gateway
- Tape Gateway
- 混合存储用例
- 本地集成

### 5. 数据库服务（5-7 小时）

#### 5.1 Amazon RDS（Relational Database Service）
- RDS 概述
- 支持的数据库引擎
- 创建 RDS 实例
- Multi-AZ 部署
- 只读副本
- RDS 备份与恢复
- RDS 安全
- RDS 监控与性能

#### 5.2 RDS 数据库引擎
- RDS 上的 PostgreSQL
- RDS 上的 MySQL
- RDS 上的 MariaDB
- RDS 上的 Oracle
- RDS 上的 SQL Server
- 选择数据库引擎
- 迁移策略

#### 5.3 Amazon Aurora
- Aurora 概述
- Aurora 架构
- Aurora Serverless
- Aurora Global Database
- Aurora vs RDS
- 性能特性
- Aurora 定价

#### 5.4 NoSQL 数据库
- Amazon DynamoDB
- DynamoDB 概念（表、项、属性）
- 主键
- 索引（GSI、LSI）
- DynamoDB 容量模式
- DynamoDB Streams
- 用于 ML 的 DynamoDB
- DocumentDB（兼容 MongoDB）
- ElastiCache（Redis、Memcached）

#### 5.5 数据仓库与分析
- Amazon Redshift
- 数据仓库概念
- Redshift 架构
- Redshift Spectrum
- Amazon Athena
- 使用 SQL 查询 S3
- 数据湖分析
- QuickSight 可视化

### 6. 网络（6-8 小时）

#### 6.1 Amazon VPC（Virtual Private Cloud）
- VPC 基础
- CIDR 块
- 子网（公有和私有）
- 创建 VPC
- 默认 VPC
- VPC 组件
- VPC 最佳实践
- 网络设计模式

#### 6.2 互联网连接
- Internet Gateway
- NAT Gateway
- NAT Instance
- Egress-only Internet Gateway
- 公有 vs 私有子网
- 路由表
- 路由配置

#### 6.3 安全组与 NACL
- 安全组基础
- 入站和出站规则
- 安全组最佳实践
- Network ACL（NACL）
- 有状态 vs 无状态
- NACL 规则
- 安全层级

#### 6.4 VPC 连接
- VPC 对等连接
- Transit Gateway
- VPN 连接
- AWS Direct Connect
- PrivateLink
- VPC 端点
- 混合连接

#### 6.5 DNS 与内容分发
- Amazon Route 53
- DNS 基础
- 托管区域
- 路由策略
- 健康检查
- CloudFront（CDN）
- CloudFront 分发
- 边缘节点

#### 6.6 网络监控
- VPC Flow Logs
- 流量分析
- 网络性能监控
- CloudWatch 指标
- 网络故障排查
- 安全监控

### 7. 基础设施即代码（5-6 小时）

#### 7.1 IaC 基础
- 什么是基础设施即代码？
- IaC 的优势
- IaC 工具对比
- 声明式 vs 命令式
- 状态管理
- IaC 最佳实践
- 基础设施的版本控制

#### 7.2 AWS CloudFormation
- CloudFormation 概述
- 模板（JSON 和 YAML）
- 堆栈
- 资源和属性
- 参数和输出
- 内置函数
- 堆栈更新
- 变更集

#### 7.3 Terraform 基础
- Terraform 概述
- 安装 Terraform
- HCL（HashiCorp Configuration Language）
- 提供者和资源
- Terraform 工作流（init、plan、apply）
- Terraform 状态
- 变量和输出
- 模块

#### 7.4 用于 AWS 的 Terraform
- AWS 提供者配置
- 使用 Terraform 创建 EC2 实例
- VPC 配置
- 使用 Terraform 管理 S3 存储桶
- 使用 Terraform 管理 RDS
- Terraform 最佳实践
- 远程状态存储
- Terraform workspaces

#### 7.5 基础设施的 CI/CD
- GitOps 用于基础设施
- 自动化部署
- 测试基础设施代码
- 基础设施流水线
- 漂移检测
- 策略即代码
- 安全扫描

### 8. ML 与 AI 服务（5-7 小时）

#### 8.1 AWS ML 服务概览
- AWS AI/ML 服务组合
- SageMaker
- Rekognition
- Comprehend
- Translate
- Transcribe
- Polly
- 选择合适的服务

#### 8.2 Amazon SageMaker 基础
- SageMaker 概述
- SageMaker Studio
- Notebook 实例
- 训练作业
- 超参数调优
- 模型托管
- SageMaker Pipelines
- SageMaker 定价

#### 8.3 AWS 上的 ML 基础设施
- 配备 GPU 的 EC2（P 和 G 实例）
- Deep Learning AMI
- 用于 ML 工作负载的 ECS/EKS
- 用于 ML 的批处理
- 用于 ML 的数据存储（S3、EFS）
- 分布式训练
- 模型服务架构

#### 8.4 AWS 上的 ML 工作流
- 数据摄取（S3、Kinesis）
- 数据处理（EMR、Glue）
- 特征工程
- 模型训练
- 模型注册表
- 模型部署
- 监控与重新训练
- 端到端流水线

### 9. 监控与管理（4-5 小时）

#### 9.1 Amazon CloudWatch
- CloudWatch 概述
- 指标
- 告警
- 仪表盘
- CloudWatch Logs
- 日志组和流
- CloudWatch Insights
- CloudWatch Events/EventBridge

#### 9.2 AWS 监控最佳实践
- 监控什么
- 自定义指标
- 日志聚合
- 应用监控
- 基础设施监控
- 成本监控
- 安全监控
- 性能监控

#### 9.3 AWS Systems Manager
- Systems Manager 概述
- Parameter Store
- Session Manager
- Patch Manager
- Run Command
- State Manager
- 舰队管理

#### 9.4 成本管理
- AWS Cost Explorer
- 成本和用量报告
- 预算与告警
- 成本优化策略
- 合理调整规格
- 预留实例
- Savings Plans
- 使用 Spot 实例节省成本

### 10. 安全与合规（4-5 小时）

#### 10.1 AWS 安全基础
- 共享责任模型
- 安全支柱
- 纵深防御
- 安全最佳实践
- 合规框架
- AWS 安全服务

#### 10.2 深入身份和访问管理
- IAM 策略类型
- 策略评估逻辑
- 跨账户访问
- 身份联合
- AWS SSO
- 服务的 IAM 角色
- 权限边界
- Access Analyzer

#### 10.3 数据保护
- 静态加密
- 传输加密
- KMS（Key Management Service）
- Secrets Manager
- Certificate Manager
- 数据分类
- 备份策略

#### 10.4 安全服务
- AWS WAF（Web Application Firewall）
- AWS Shield（DDoS 防护）
- Amazon GuardDuty
- AWS Security Hub
- Amazon Inspector
- AWS Config
- 用于审计的 CloudTrail
- 用于数据发现的 Macie

#### 10.5 合规
- 合规计划
- Artifact
- 合规文档
- GDPR 考量
- AWS 上的 HIPAA
- PCI DSS
- 审计与治理

### 11. 多云对比（3-4 小时）

#### 11.1 Google Cloud Platform（GCP）概述
- GCP 服务概览
- Compute Engine
- Cloud Storage
- Cloud SQL
- BigQuery
- GKE（Google Kubernetes Engine）
- Vertex AI
- GCP vs AWS 对比

#### 11.2 Microsoft Azure 概述
- Azure 服务概览
- Virtual Machines
- Blob Storage
- Azure SQL
- Azure Kubernetes Service
- Azure Machine Learning
- Azure vs AWS 对比

#### 11.3 多云策略
- 采用多云的原因
- 多云的挑战
- 抽象层
- 云无关工具
- 数据引力
- 供应商多样性
- 多云管理

## 讲座大纲

### 讲座 1：云计算基础（90 分钟）
- 云计算简介
- 服务和部署模型
- 主要云服务商
- 云经济学
- AWS 账户设置
- **实验：** 创建 AWS 账户并探索控制台

### 讲座 2：AWS 计算服务（120 分钟）
- EC2 基础
- 实例类型与选择
- 启动实例
- 安全组
- 负载均衡基础
- **实验：** 启动并连接到 EC2 实例

### 讲座 3：AWS 存储服务（120 分钟）
- S3 基础
- EBS 概述
- 存储类选择
- 数据生命周期管理
- 存储最佳实践
- **实验：** 使用 S3 和 EBS

### 讲座 4：AWS 网络（120 分钟）
- VPC 基础
- 子网和路由
- 互联网连接
- 安全组和 NACL
- VPC 设计模式
- **实验：** 构建 VPC

### 讲座 5：AWS 数据库（90 分钟）
- RDS 基础
- 数据库引擎选择
- DynamoDB 基础
- 云数据库设计
- 备份与恢复
- **实验：** 部署 RDS 实例

### 讲座 6：IAM 与安全（90 分钟）
- IAM 深入
- 策略创建
- 安全最佳实践
- 加密
- 合规
- **实验：** 实施 IAM 策略

### 讲座 7：基础设施即代码（120 分钟）
- IaC 基础
- Terraform 基础
- AWS CloudFormation
- 用于 AWS 的 Terraform
- IaC 最佳实践
- **实验：** 使用 Terraform 部署基础设施

### 讲座 8：容器与无服务器服务（90 分钟）
- ECS 和 EKS 概述
- 容器部署
- Lambda 基础
- 无服务器模式
- 何时使用哪种
- **实验：** 部署容器化应用

### 讲座 9：AWS 上的 ML 服务（120 分钟）
- AWS ML 服务组合
- SageMaker 概述
- 用于 ML 的 GPU 实例
- AWS 上的 ML 工作流
- ML 基础设施模式
- **实验：** 在 AWS 上运行 ML 工作负载

### 讲座 10：监控与成本优化（90 分钟）
- CloudWatch 基础
- 成本管理
- 监控最佳实践
- 优化策略
- Well-Architected Framework
- **实验：** 设置监控和成本告警

## 动手练习

### 练习分类

#### 入门（5 个练习）
1. AWS 账户设置与配置
2. AWS CLI 配置
3. 创建 IAM 用户和角色
4. AWS 控制台导航
5. 资源标签策略

#### 计算（8 个练习）
6. 启动 EC2 实例
7. 实例类型选择
8. 安全组配置
9. SSH 密钥管理
10. 负载均衡器设置
11. Auto Scaling 配置
12. 使用 ECS 部署容器
13. 创建 Lambda 函数

#### 存储（6 个练习）
14. S3 存储桶创建与管理
15. S3 生命周期策略
16. EBS 卷管理
17. EBS 快照策略
18. EFS 设置与挂载
19. 存储类选择

#### 网络（6 个练习）
20. VPC 创建
21. 子网设计
22. Internet Gateway 设置
23. NAT Gateway 配置
24. 安全组设计
25. Route 53 DNS 配置

#### 数据库（5 个练习）
26. RDS 实例部署
27. 数据库备份与恢复
28. DynamoDB 表创建
29. 数据库安全配置
30. 只读副本设置

#### IaC（6 个练习）
31. 第一个 Terraform 配置
32. 使用 Terraform 管理 EC2
33. 使用 Terraform 管理 VPC
34. 完整基础设施堆栈
35. Terraform 模块
36. 基础设施 CI/CD

#### ML 工作负载（4 个练习）
37. GPU 实例设置
38. SageMaker notebook
39. 在 AWS 上训练模型
40. 模型部署与提供服务

## 评估与考核

### 知识检测
- 每个主要部分后的小测验
- 服务选择场景
- 架构设计问题
- 安全最佳实践
- 成本优化策略

### 实践评估
- **计算：** 在 EC2 上部署带负载均衡器的 Web 应用
- **存储：** 实施全面的存储策略
- **网络：** 设计和实现 VPC 架构
- **IaC：** 使用 Terraform 部署基础设施
- **ML：** 部署 ML 模型服务基础设施

### 能力标准
要成功完成本模块，你应该能够：
- 自信地使用 AWS 控制台和 CLI
- 配置和管理计算资源
- 适当配置存储方案
- 设计和实现 VPC 架构
- 应用 IAM 最佳实践
- 有效使用基础设施即代码
- 部署容器化应用
- 利用云服务处理 ML 工作负载
- 监控并优化成本
- 实施安全最佳实践

### 结业项目
**基于云的 ML 应用：**
在 AWS 上部署完整的 ML 应用，包括：
- 带公有和私有子网的 VPC
- 带 Auto Scaling 的 EC2 实例
- Application Load Balancer
- RDS 数据库
- 用于数据和模型存储的 S3
- ML 模型服务（SageMaker 或自定义）
- CloudWatch 监控
- IAM 安全配置
- 基础设施即代码（Terraform）
- 成本优化实现
- 完整文档

## 资源与参考

> **注意：** 详细的学习资料、书籍和在线资源列表请参见 `resources/recommended-reading.md`。

### 核心资源
- AWS 文档
- AWS Well-Architected Framework
- AWS 免费套餐指南
- Terraform 文档

### 推荐书籍
- 《Amazon Web Services in Action》— Michael Wittig 和 Andreas Wittig
- 《AWS Certified Solutions Architect Study Guide》
- 《Terraform: Up & Running》— Yevgeniy Brikman

### 在线学习
- AWS 培训与认证
- A Cloud Guru
- Linux Academy
- AWS YouTube 频道
- AWS 研讨会和教程

### 认证
- AWS Certified Cloud Practitioner
- AWS Certified Solutions Architect - Associate
- AWS Certified Developer - Associate
- Terraform Associate Certification

### 工具
- AWS CLI
- Terraform
- AWS SDK for Python（boto3）
- AWS CloudFormation Designer
- Infracost（成本估算）

## 开始入门

### 第 1 步：创建 AWS 账户
1. 注册 AWS 免费套餐
2. 验证邮箱和支付方式
3. 为根账户设置 MFA
4. 创建 IAM 管理员用户
5. 安装 AWS CLI

### 第 2 步：配置本地环境
1. 安装并配置 AWS CLI
2. 设置 AWS 凭证
3. 安装 Terraform
4. 生成 SSH 密钥对
5. 探索 AWS 控制台

### 第 3 步：从讲座 1 开始
- 学习云基础
- 了解 AWS 服务
- 完成账户设置
- 启动第一个资源

### 第 4 步：循序渐进
- 按顺序完成练习
- 尝试各种服务
- 构建越来越复杂的架构
- 将学习应用到项目
- 仔细监控成本

## 成功提示

1. **监控成本：** 立即设置账单告警
2. **使用免费套餐：** 利用免费套餐学习
3. **清理资源：** 始终终止未使用的资源
4. **给所有资源打标签：** 使用标签进行组织和成本追踪
5. **使用 IaC：** 从一开始就实践基础设施即代码
6. **安全优先：** 从第一天起就应用最小权限
7. **阅读文档：** AWS 文档很全面
8. **安全实验：** 使用单独的账户进行实验
9. **加入社区：** AWS 论坛和用户组
10. **考取认证：** 用认证来验证知识

## 成本管理

**重要：** 使用 AWS 免费套餐时，请注意：
- EC2：每月 750 小时 t2.micro 或 t3.micro
- S3：5GB 存储、20000 次 GET 请求、2000 次 PUT 请求
- RDS：每月 750 小时 db.t2.micro、db.t3.micro 或 db.t4g.micro
- 始终设置账单告警
- 练习结束后删除资源
- 部署前使用成本计算器

## 后续步骤

完成本模块后，你将能够：
- 构建生产基础设施
- 准备 AWS 认证
- 高级主题：Kubernetes、无服务器架构、大规模 ML
- 将云知识应用到所有未来项目中

## 开发状态

**当前状态：** 模板阶段 — 结构已全面就绪

**现已有：**
- 完整模块结构
- 详细主题分解
- 讲座大纲
- 练习框架

**开发中：**
- 完整的讲座笔记（含图）
- 分步 AWS 练习
- Terraform 示例
- 架构图
- 成本优化指南
- ML 部署模式
- 完整项目模板

**计划更新：**
- GCP 和 Azure 深入
- Kubernetes 在云上
- 无服务器架构
- 多云策略
- 高级 ML 服务
- 云安全深入

## 反馈与贡献

帮助改进本模块：
- 报告问题
- 分享架构模式
- 贡献 Terraform 模块
- 建议练习
- 分享成本优化技巧

---

**模块维护者：** AI 基础设施课程团队
**联系方式：** ai-infra-curriculum@joshua-ferguson.com
**最后更新：** 2025-10-18
**版本：** 1.0.0-template
