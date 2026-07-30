# Module 010 — 云平台资源

## 官方文档

### AWS

- **AWS 文档** — [docs.aws.amazon.com](https://docs.aws.amazon.com/)。权威参考。
- **AWS Well-Architected Framework** — [aws.amazon.com/architecture/well-architected](https://aws.amazon.com/architecture/well-architected/)。读起来像一本生产级云架构教科书。
- **AWS Machine Learning Lens** — [docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens](https://docs.aws.amazon.com/wellarchitected/latest/machine-learning-lens/)。Well-Architected 应用于 ML 工作负载。

### GCP

- **GCP 文档** — [cloud.google.com/docs](https://cloud.google.com/docs)。
- **GCP Architecture Framework** — [cloud.google.com/architecture/framework](https://cloud.google.com/architecture/framework)。GCP 版的 AWS Well-Architected。

### Azure

- **Azure 文档** — [learn.microsoft.com/en-us/azure](https://learn.microsoft.com/en-us/azure/)。
- **Azure Well-Architected Framework** — [learn.microsoft.com/en-us/azure/architecture/framework](https://learn.microsoft.com/en-us/azure/architecture/framework/)。

## 认证

值得了解这些认证的存在；有些雇主看重它们。但不要为了认证而考证。

- **AWS Certified Cloud Practitioner** — 入门级，覆盖面广。
- **AWS Solutions Architect Associate** — 标准中级认证。
- **AWS Machine Learning Specialty** — ML 专项。
- **GCP Associate Cloud Engineer** — 对标 AWS Solutions Architect Associate 级别。
- **Azure Fundamentals（AZ-900）** — Azure 入门级。

## 书籍

- **《AWS Cookbook（第 2 版）》**— John Culkin & Mike Zazon。菜谱风格。优秀参考书。
- **《Cloud Native Patterns》**— Cornelia Davis。跨云适用的架构模式。
- **《Architecting Modern Data Platforms》**（O'Reilly）。跨云数据架构。

## ML 特定云服务

- **AWS SageMaker 文档** — [docs.aws.amazon.com/sagemaker](https://docs.aws.amazon.com/sagemaker/)。AWS 的托管 ML 平台。
- **GCP Vertex AI 文档** — [cloud.google.com/vertex-ai/docs](https://cloud.google.com/vertex-ai/docs)。GCP 的托管 ML 平台。
- **Azure ML 文档** — [learn.microsoft.com/en-us/azure/machine-learning](https://learn.microsoft.com/en-us/azure/machine-learning/)。Azure 的托管 ML 平台。

三者的权衡：上线速度更快，但托管组件（训练流水线、模型注册表、监控）存在供应商锁定。要有意识地决策，不要随波逐流。

## 基础设施即代码

- **Terraform 文档** — [developer.hashicorp.com/terraform/docs](https://developer.hashicorp.com/terraform/docs)。跨云 IaC 的事实标准。
- **AWS CDK** — [aws.amazon.com/cdk](https://aws.amazon.com/cdk/)。AWS 的 IaC 框架，支持你选择的语言（Python、TypeScript 等）。
- **Pulumi** — [pulumi.com/docs](https://www.pulumi.com/docs/)。使用真实编程语言的多云 IaC。
- **OpenTofu** — [opentofu.org](https://opentofu.org/)。Terraform 许可证变更后的社区分支。

## 成本管理

- **AWS Cost Explorer + Budgets** — 内置工具。
- **GCP Cost Management** — [cloud.google.com/cost-management](https://cloud.google.com/cost-management)。
- **OpenCost** — [opencost.io](https://www.opencost.io/)。Kubernetes 成本归因的开放标准。
- **Vantage / Kubecost** — 商业成本可视化工具。

对于 ML 工作负载，成本意外通常来自：闲置的 GPU 实例、出站流量、S3 存储类、未跟踪的开发环境。设置告警。

## 多云与跨云

- **Cilium ClusterMesh** — [docs.cilium.io/en/stable/network/clustermesh](https://docs.cilium.io/en/stable/network/clustermesh/)。多集群网络。
- **Karpenter** — [karpenter.sh](https://karpenter.sh/)。AWS 原生的节点自动伸缩器；适合 ML 突发容量。
- **Crossplane** — [crossplane.io](https://www.crossplane.io/)。Kubernetes 原生的跨云 IaC。

## 本课程中的交叉参考

- 模块 006（Kubernetes）— 运行在每个云之上的编排层。
- 工程师轨道的 `mod-102-cloud-computing` 深入涵盖生产云模式。
- 工程师轨道的 `mod-109-infrastructure-as-code` 涉及 Terraform + GitOps。
- 高级工程师轨道的 `mod-205-multi-cloud` 涉及跨云架构。

## 关于"应该学哪个云"的说明

如果你是新手：**AWS** 市场份额最大，就业市场也最广。先学 AWS。等你对云基础熟悉后，再学习 GCP/Azure —— 大多数概念是相通的；只是具体实现有所不同。
