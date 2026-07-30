# Exercise 07：使用 Terraform 的基础设施即代码基础

**难度**：入门
**时长**：3-4 小时
**前置要求**：已配置 CLI 的 AWS 账户、基础云概念

## 学习目标

到本练习结束时，你将：

1. 理解什么是基础设施即代码（IaC）以及为什么它很重要
2. 安装和配置 Terraform
3. 编写基本的 Terraform 配置文件
4. 使用 Terraform 创建云资源
5. 使用 Terraform 修改和销毁基础设施
6. 理解 Terraform 状态和最佳实践
7. 在 Terraform 中使用变量和输出

## 你将构建的内容

使用 Terraform，你将创建：
- 用于存储 ML 数据集的 AWS S3 存储桶
- 用于 ML 工作负载的 EC2 实例
- 用于网络访问控制的安全组
- IAM 角色和策略
- 一个简单的 ML 基础设施堆栈

全部以代码定义，可以进行版本控制、审查和重用！

## 背景：为什么需要基础设施即代码？

### 问题：手动基础设施管理

**IaC 之前（旧方式）**：
```
1. 登录 AWS 控制台
2. 点击 20+ 个屏幕来创建 EC2 实例
3. 手动配置安全组
4. 设置网络
5. 创建存储
6. 记录你做了什么（也许）
7. 为开发、预发布、生产环境重复
8. 希望每次点击了相同的按钮
```

**问题**：
- ❌ 不可重现
- ❌ 容易出错（拼写错误、遗漏步骤）
- ❌ 无版本控制
- ❌ 难以审查变更
- ❌ 难以克隆环境
- ❌ 无审计跟踪

### 解决方案：基础设施即代码

**使用 IaC**：
```hcl
# infrastructure.tf
resource "aws_instance" "ml_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.large"

  tags = {
    Name = "ML Training Server"
  }
}
```

**优点**：
- ✅ 可重现（运行相同代码 → 获得相同基础设施）
- ✅ 版本控制（Git 跟踪所有变更）
- ✅ 可审查（基础设施的拉取请求）
- ✅ 可审计（谁在何时更改了什么）
- ✅ 可测试（应用前验证）
- ✅ 有文档（代码即文档）

### 为什么选择 Terraform？

**Terraform**（由 HashiCorp 开发）是最流行的 IaC 工具：
- **多云**：适用于 AWS、GCP、Azure 及 100+ 提供商
- **声明式**：描述你想要什么，而非如何创建
- **计划再应用**：在做出更改前预览变更
- **状态跟踪**：知道实际部署了什么
- **模块化**：可重用组件
- **开源**：免费使用

**替代方案**：
- AWS CloudFormation（仅 AWS）
- Pulumi（使用真实编程语言）
- Ansible（配置管理 + IaC）

## 第一部分：Terraform 设置

### 第 1 步：安装 Terraform

**macOS**：
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

**Linux**：
```bash
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
```

**Windows**：
```powershell
choco install terraform
# 或从 https://www.terraform.io/downloads 下载
```

**验证安装**：
```bash
terraform version
```

预期输出：
```
Terraform v1.6.0
```

### 第 2 步：设置 AWS 凭证

Terraform 需要 AWS 凭证来创建资源。

**选项 1：AWS CLI**（推荐）：
```bash
aws configure
# 输入你的 AWS Access Key ID、Secret Access Key 和区域
```

**选项 2：环境变量**：
```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

**验证凭证**：
```bash
aws sts get-caller-identity
```

### 第 3 步：创建项目目录

```bash
mkdir terraform-ml-infrastructure
cd terraform-ml-infrastructure
```

## 第二部分：你的第一个 Terraform 配置

### 第 4 步：创建简单的 S3 存储桶

创建 `main.tf`：

```hcl
# 配置 AWS Provider
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.0"
}

provider "aws" {
  region = "us-east-1"
}

# 为 ML 数据集创建 S3 存储桶
resource "aws_s3_bucket" "ml_datasets" {
  bucket = "my-ml-datasets-bucket-${random_id.bucket_suffix.hex}"

  tags = {
    Name        = "ML Datasets Bucket"
    Environment = "Dev"
    Purpose     = "Store training datasets"
    ManagedBy   = "Terraform"
  }
}

# 生成唯一存储桶名称的随机后缀
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

# 阻止对存储桶的公共访问
resource "aws_s3_bucket_public_access_block" "ml_datasets" {
  bucket = aws_s3_bucket.ml_datasets.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### 第 5 步：初始化 Terraform

```bash
terraform init
```

**这会做什么**：
- 下载 AWS Provider 插件
- 初始化后端（状态存储位置）
- 创建 `.terraform` 目录

### 第 6 步：计划变更

```bash
terraform plan
```

**这会做什么**：
- 显示 Terraform 将要创建/更改/销毁的内容
- 验证配置语法
- 检查状态与期望配置的对比

### 第 7 步：应用变更

```bash
terraform apply
```

输入 `yes` 并按 Enter 确认。

### 第 8 步：理解 Terraform 状态

应用后，Terraform 创建 `terraform.tfstate`：

```bash
ls -la
```

你会看到：
```
main.tf
terraform.tfstate
terraform.tfstate.backup
.terraform/
```

**什么是状态？**
- 跟踪实际基础设施的 JSON 文件
- 将 Terraform 配置映射到真实资源
- **关键**：不要删除或手动编辑！
- **重要**：不要提交到 Git（包含敏感数据）

创建 `.gitignore`：
```
# .gitignore
.terraform/
terraform.tfstate
terraform.tfstate.backup
*.tfvars
.terraform.lock.hcl
```

## 第三部分：添加变量和输出

### 第 9 步：使用变量

创建 `variables.tf`：

```hcl
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "project_name" {
  description = "Project name for resource tagging"
  type        = string
  default     = "ml-infrastructure"
}

variable "enable_versioning" {
  description = "Enable versioning for S3 bucket"
  type        = bool
  default     = true
}
```

### 第 10 步：添加输出

创建 `outputs.tf`：

```hcl
output "bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.ml_datasets.bucket
}

output "bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.ml_datasets.arn
}
```

## 第四部分：为 ML 创建 EC2 实例

### 第 11 步：添加 EC2 实例

添加到 `main.tf`：

```hcl
# 数据源：获取最新的 Amazon Linux 2 AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# EC2 实例的安全组
resource "aws_security_group" "ml_instance" {
  name        = "${var.project_name}-ml-instance-sg"
  description = "Security group for ML training instance"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "SSH access"
  }

  ingress {
    from_port   = 8888
    to_port     = 8888
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Jupyter Notebook"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound"
  }

  tags = {
    Name        = "${var.project_name}-ml-sg"
    Environment = var.environment
  }
}

# 用于 ML 训练的 EC2 实例
resource "aws_instance" "ml_training" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  vpc_security_group_ids = [aws_security_group.ml_instance.id]

  root_block_device {
    volume_size = 50
    volume_type = "gp3"
  }

  tags = {
    Name        = "${var.project_name}-ml-training"
    Environment = var.environment
    Purpose     = "ML Training"
  }
}
```

## 第五部分：使用 Terraform Workspaces

### 第 12 步：多环境

创建独立环境（dev、staging、prod）：

```bash
# 创建工作区
terraform workspace new dev
terraform workspace new staging

# 在工作区之间切换
terraform workspace select dev
```

## 第六部分：Terraform 命令参考

### 基本命令

```bash
# 初始化目录
terraform init

# 验证配置
terraform validate

# 格式化代码
terraform fmt

# 显示执行计划
terraform plan

# 应用变更
terraform apply

# 销毁所有资源
terraform destroy

# 显示当前状态
terraform show

# 列出状态中的资源
terraform state list

# 查看输出
terraform output
```

## 第七部分：清理

### 第 13 步：销毁基础设施

**重要**：始终清理以避免 AWS 费用！

```bash
terraform destroy
```

审查计划，输入 `yes` 确认。

## 关键要点

1. **IaC 至关重要**：版本控制、可重现的基础设施
2. **Terraform 是声明式的**：描述期望状态，而非步骤
3. **计划再应用**：始终预览变更
4. **状态至关重要**：跟踪真实基础设施，保护好它
5. **使用变量**：使配置可重用
6. **给所有资源打标签**：组织和成本跟踪
7. **工作区用于环境**：管理开发/预发布/生产
8. **模块用于可重用性**：不要重复自己

## 基础设施考量

### 成本估算

应用 `terraform apply` 前估算成本：
- **t3.medium EC2**：约每月 $30（如果 24/7 运行）
- **50GB EBS**：约每月 $5
- **S3 存储**：约 $0.023/GB/月
- **数据传输**：可变

### 生产检查清单

投入生产前：
- [ ] 远程状态后端（S3 + DynamoDB 用于锁定）
- [ ] 已启用状态加密
- [ ] 团队协作使用 Terraform Cloud/Enterprise
- [ ] 自动化计划的 CI/CD 集成
- [ ] 基础设施变更的代码审查流程
- [ ] 环境使用独立的 AWS 账户
- [ ] 限制 IAM 权限（最小权限）
- [ ] 全面的标签策略
- [ ] 备份和灾难恢复计划
- [ ] 已配置监控和告警

---

**恭喜！** 你已经学会了基础设施即代码的基础知识。现在你可以使用代码来定义、版本管理和云基础设施了——这是现代基础设施工程师的关键技能！
