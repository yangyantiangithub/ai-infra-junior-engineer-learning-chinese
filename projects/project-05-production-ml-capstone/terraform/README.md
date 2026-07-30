# Terraform — 项目 05 结业项目

这是项目 05 结业项目的基础设施即代码层。它配置 Kubernetes 部署以下的所有内容：网络、集群、托管数据库、IAM、密钥。

## 目录结构

```
terraform/
├── modules/                    # 可复用的组合模块
│   ├── vpc/                    # VPC、子网、路由表、NAT
│   ├── eks/                    # EKS 集群 + 托管节点组
│   ├── rds/                    # 应用数据的 PostgreSQL
│   └── iam/                    # IRSA 角色、KMS 密钥
└── environments/               # 每个环境一个根模块
    ├── dev/
    ├── staging/
    └── prod/
```

每个环境是一个独立的 Terraform 状态，通过 S3 后端和 DynamoDB 锁表隔离。

## 为什么需要三个环境

- **dev** — 小型、临时。单可用区。无高可用。频繁销毁/重建。
- **staging** — 生产级形态。多可用区。真实但缩放的资源。发布在进入生产前在此落地。
- **prod** — 完整高可用。锁定状态文件。变更需通过 CODEOWNERS 审查。

## 模块使用示例

```hcl
# environments/prod/main.tf
module "vpc" {
  source             = "../../modules/vpc"
  name               = "ml-prod"
  cidr_block         = "10.20.0.0/16"
  availability_zones = ["us-west-2a", "us-west-2b", "us-west-2c"]
  enable_nat_gateway = true
  single_nat_gateway = false
}

module "eks" {
  source             = "../../modules/eks"
  cluster_name       = "ml-prod"
  kubernetes_version = "1.30"
  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids

  node_groups = {
    general = {
      instance_types = ["m6i.large"]
      desired_size   = 3
      min_size       = 3
      max_size       = 10
    }
    inference = {
      instance_types = ["c6i.xlarge"]
      desired_size   = 2
      min_size       = 2
      max_size       = 12
      labels         = { workload = "inference" }
      taints         = [{ key = "workload", value = "inference", effect = "NO_SCHEDULE" }]
    }
  }
}
```

## 状态后端

使用每个环境的 S3 后端配合 DynamoDB 锁。切勿提交 `*.tfstate`。

```hcl
terraform {
  backend "s3" {
    bucket         = "company-tfstate"
    key            = "ml/prod/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "tfstate-locks"
    encrypt        = true
  }
}
```

## 标准流程

```bash
cd terraform/environments/dev
terraform init
terraform plan -out=plan.tfplan
terraform apply plan.tfplan
```

切勿在未先审查 `terraform plan` 的情况下运行 `terraform apply`。CI 应强制在 PR 上执行 plan，合并后执行 apply。

## 常见陷阱

- **EKS 集群更新需要 20 分钟以上。** 不要让 CI 被 `terraform apply` 阻塞；请在超时时间更长的流水线上运行。
- **不要从 `terraform destroy` 中销毁有状态资源**（RDS、S3 存储桶）。使用 `lifecycle { prevent_destroy = true }`。
- **状态中的密钥。** Terraform 状态文件包含机密值。静态加密，通过 IAM 限制访问，切勿下载到共享工作站上的本地磁盘。
