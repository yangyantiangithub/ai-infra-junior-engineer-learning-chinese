# Exercise 06：在 AWS 上部署完整 ML 基础设施

## 概述

这个结业练习指导你使用基础设施即代码（Terraform）在 AWS 上部署完整、生产级的 ML 基础设施，包括模型训练、推理、监控和 CI/CD。你将构建一个展示本模块学到的所有云技能的真实 ML 平台。

## 学习目标

完成本练习后，你将：
- 使用 Terraform（IaC）部署 ML 基础设施
- 设置用于数据和模型存储的 S3
- 配置 EC2/ECS 用于模型训练和服务
- 实施 VPC 网络和安全组
- 使用 CloudWatch 设置监控
- 配置推理工作负载的自动扩缩容
- 实施成本优化策略
- 构建用于自动化部署的 CI/CD 流水线

## 前置要求

- 已完成本模块的练习 01-05
- 所有云平台讲座
- AWS 账户（免费套餐可用）
- 已配置 AWS CLI
- 已安装 Terraform
- 基础的 ML 和 Docker 知识

## 所需时间

- 预计：180-240 分钟
- 难度：高级

---

## 第一部分：AWS 账户设置

### 任务 1.1：创建 AWS 账户并配置 CLI

```bash
# 安装 AWS CLI（如果尚未安装）
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 配置 AWS CLI
aws configure
# 输入：
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region：us-east-1
# - Default output：json

# 验证配置
aws sts get-caller-identity
```

### 任务 1.2：安装 Terraform

```bash
# 安装 Terraform
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# 验证安装
terraform version
```

---

## 第二部分：基础设施即代码（Terraform）

### 任务 2.1：创建项目结构

```bash
mkdir ml-infrastructure-aws
cd ml-infrastructure-aws

# 创建 Terraform 结构
mkdir -p terraform/{modules,environments/{dev,prod}}
mkdir -p terraform/modules/{networking,compute,storage,security}

# 创建文件
touch terraform/main.tf
touch terraform/variables.tf
touch terraform/outputs.tf
touch terraform/terraform.tfvars
```

### 任务 2.2：定义网络模块

**terraform/modules/networking/main.tf：**
```hcl
# VPC
resource "aws_vpc" "ml_vpc" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.project_name}-vpc"
    Environment = var.environment
  }
}

# 公有子网
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.ml_vpc.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-public-subnet-${count.index + 1}"
  }
}

# 私有子网
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.ml_vpc.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.project_name}-private-subnet-${count.index + 1}"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.ml_vpc.id

  tags = {
    Name = "${var.project_name}-igw"
  }
}

# 路由表
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.ml_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

data "aws_availability_zones" "available" {
  state = "available"
}
```

### 任务 2.3：定义存储模块（S3）

**terraform/modules/storage/main.tf：**
```hcl
# 用于模型的 S3 存储桶
resource "aws_s3_bucket" "ml_models" {
  bucket = "${var.project_name}-models-${var.environment}"

  tags = {
    Name        = "${var.project_name}-models"
    Environment = var.environment
  }
}

# 启用版本控制
resource "aws_s3_bucket_versioning" "ml_models_versioning" {
  bucket = aws_s3_bucket.ml_models.id
  versioning_configuration {
    status = "Enabled"
  }
}

# 启用加密
resource "aws_s3_bucket_server_side_encryption_configuration" "ml_models_encryption" {
  bucket = aws_s3_bucket.ml_models.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# 阻止公共访问
resource "aws_s3_bucket_public_access_block" "ml_models_public_access" {
  bucket = aws_s3_bucket.ml_models.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### 任务 2.4：主 Terraform 配置

**terraform/main.tf：**
```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "ml-infrastructure/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}

module "networking" {
  source = "./modules/networking"
  project_name = var.project_name
  environment  = var.environment
  vpc_cidr     = var.vpc_cidr
}

module "storage" {
  source = "./modules/storage"
  project_name = var.project_name
  environment  = var.environment
}
```

---

## 第三部分：部署基础设施

### 任务 3.1：初始化和应用 Terraform

```bash
cd terraform

# 初始化 Terraform
terraform init

# 验证配置
terraform validate

# 计划部署
terraform plan

# 应用（创建基础设施）
terraform apply

# 保存输出
terraform output > ../outputs.txt
```

---

## 第四部分：部署 ML 应用到 ECS

### 任务 4.1：创建 ECS 任务定义

```json
{
  "family": "ml-inference-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ml-platform-ecs-task-execution-role",
  "containerDefinitions": [
    {
      "name": "ml-api",
      "image": "YOUR_ECR_REPO/ml-inference:latest",
      "portMappings": [{"containerPort": 8000, "protocol": "tcp"}],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/ml-platform",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ml-api"
        }
      }
    }
  ]
}
```

---

## 第五部分：成本优化

### 任务 5.1：实施自动扩缩容

**autoscaling.tf：**
```hcl
resource "aws_appautoscaling_target" "ecs_target" {
  max_capacity       = 10
  min_capacity       = 1
  resource_id        = "service/${aws_ecs_cluster.ml_cluster.name}/${aws_ecs_service.ml_inference.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "ecs_policy_cpu" {
  name               = "cpu-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace
  target_tracking_scaling_policy_configuration {
    target_value = 70.0
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
  }
}
```

---

## 第六部分：清理

### 任务 6.1：销毁基础设施

```bash
# 销毁所有 Terraform 管理的资源
terraform destroy

# 确保 S3 存储桶已清空
aws s3 rm s3://ml-platform-models-dev --recursive
aws s3 rm s3://ml-platform-data-dev --recursive
```

---

## 交付物

提交以下内容：

1. **Terraform 代码**：
   - VPC、S3、ECS 的完整 IaC
   - 模块化结构
   - 变量和输出

2. **部署证据**：
   - AWS 控制台显示资源的截图
   - Terraform apply 输出
   - 成本估算

3. **文档**：
   - 架构图
   - 部署指南
   - 成本分解

4. **清理证据**：
   - Terraform destroy 输出
   - 确认所有资源已删除

---

**恭喜！** 你已使用基础设施即代码在 AWS 上部署了完整的生产 ML 基础设施！
