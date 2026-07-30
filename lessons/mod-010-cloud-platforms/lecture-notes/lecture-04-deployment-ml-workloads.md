# Lecture 04：AWS 上的部署模式与 ML 工作负载

**模块**：云平台
**级别**：初级 AI 基础设施工程师
**时长**：90 分钟
**前置要求**：讲座 01-03（云基础、AWS 核心服务、网络与安全）

---

## 学习目标

到本讲座结束时，你将能够：
1. 使用基础设施即代码（IaC）和 Terraform 配置 AWS 资源
2. 使用 ECS 和 EKS 部署容器化 ML 应用
3. 使用 AWS Lambda 实现无服务器 ML 推理
4. 利用 AWS SageMaker 进行托管 ML 工作流
5. 应用 ML 工作负载的成本优化策略
6. 设计生产级部署架构

---

## 1. 使用 Terraform 的基础设施即代码（IaC）

### 1.1 为什么需要基础设施即代码？

**手动基础设施的问题**：
- **不一致**：不同环境（开发、预发布、生产）的配置不同
- **无版本控制**：无法跟踪更改或回滚
- **难以重现**：灾难发生后难以重建基础设施
- **速度慢**：在 AWS 控制台中点击操作耗时
- **无法协作**：团队难以共同处理基础设施

**IaC 的优势**：
- **版本控制**：基础设施更改在 Git 中跟踪
- **可重现**：相同代码 → 每次都相同的基础设施
- **快速**：几分钟内配置复杂基础设施
- **协作**：团队可以像审查代码一样审查基础设施更改
- **自我文档化**：代码描述了存在什么基础设施

### 1.2 Terraform 基础

**Terraform 工作流**：
```
编写代码 → 计划变更 → 应用变更 → 基础设施已创建
```

**Terraform 架构**：
```
┌──────────────────┐
│  Terraform 代码  │（.tf 文件）
│  （HCL 语言）    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Terraform CLI   │（terraform plan/apply）
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   AWS Provider   │（转换为 AWS API 调用）
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   AWS 资源       │（EC2、S3、VPC 等）
└──────────────────┘
```

### 1.3 Terraform 安装与配置

**安装 Terraform**（Ubuntu/Debian）：
```bash
# 下载并安装 Terraform
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# 验证安装
terraform --version
# 输出：Terraform v1.6.0
```

**配置 AWS Provider**：
```hcl
# provider.tf
terraform {
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

    # 启用状态锁定以防止并发修改
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "ML-Infrastructure"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

**为什么使用 S3 后端？**
- **远程状态**：团队成员可以共享 Terraform 状态
- **状态锁定**：DynamoDB 防止多人同时修改基础设施
- **加密**：状态文件包含敏感数据（IP 地址、资源 ID）

### 1.4 Terraform 示例：ML 推理基础设施

**变量**（`variables.tf`）：
```hcl
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  description = "EC2 instance type for ML inference"
  type        = string
  default     = "t3.medium"
}

variable "min_instances" {
  description = "Minimum number of instances in ASG"
  type        = number
  default     = 2
}

variable "max_instances" {
  description = "Maximum number of instances in ASG"
  type        = number
  default     = 10
}
```

**VPC 与网络**（`network.tf`）：
```hcl
# VPC
resource "aws_vpc" "ml_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "ml-inference-vpc-${var.environment}"
  }
}

# 公有子网（用于 ALB）
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.ml_vpc.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "ml-public-subnet-${count.index + 1}-${var.environment}"
    Tier = "Public"
  }
}

# 私有子网（用于 EC2 实例）
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.ml_vpc.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "ml-private-subnet-${count.index + 1}-${var.environment}"
    Tier = "Private"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.ml_vpc.id

  tags = {
    Name = "ml-igw-${var.environment}"
  }
}

# NAT Gateway（用于私有子网访问互联网）
resource "aws_eip" "nat" {
  count  = 2
  domain = "vpc"

  tags = {
    Name = "ml-nat-eip-${count.index + 1}-${var.environment}"
  }
}

resource "aws_nat_gateway" "main" {
  count         = 2
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "ml-nat-gateway-${count.index + 1}-${var.environment}"
  }
}

# 路由表
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.ml_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "ml-public-rt-${var.environment}"
  }
}

resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.ml_vpc.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "ml-private-rt-${count.index + 1}-${var.environment}"
  }
}

# 路由表关联
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count          = 2
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# 可用区数据源
data "aws_availability_zones" "available" {
  state = "available"
}
```

**安全组**（`security.tf`）：
```hcl
# ALB 安全组（允许来自互联网的 HTTP/HTTPS）
resource "aws_security_group" "alb" {
  name        = "ml-alb-sg-${var.environment}"
  description = "Security group for ML inference ALB"
  vpc_id      = aws_vpc.ml_vpc.id

  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP from internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ml-alb-sg-${var.environment}"
  }
}

# 应用安全组（仅允许来自 ALB 的流量）
resource "aws_security_group" "app" {
  name        = "ml-app-sg-${var.environment}"
  description = "Security group for ML inference application"
  vpc_id      = aws_vpc.ml_vpc.id

  ingress {
    description     = "Application traffic from ALB"
    from_port       = 5000
    to_port         = 5000
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "ml-app-sg-${var.environment}"
  }
}
```

**使用启动模板的 Auto Scaling 组**（`compute.tf`）：
```hcl
# EC2 实例的 IAM 角色
resource "aws_iam_role" "ec2_role" {
  name = "ml-inference-ec2-role-${var.environment}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy_attachment" "ec2_s3_access" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ml-inference-ec2-profile-${var.environment}"
  role = aws_iam_role.ec2_role.name
}

# 启动模板
resource "aws_launch_template" "ml_app" {
  name_prefix   = "ml-inference-${var.environment}-"
  image_id      = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  iam_instance_profile {
    name = aws_iam_instance_profile.ec2_profile.name
  }

  vpc_security_group_ids = [aws_security_group.app.id]

  user_data = base64encode(templatefile("${path.module}/user-data.sh", {
    model_bucket = aws_s3_bucket.models.bucket
    environment  = var.environment
  }))

  monitoring {
    enabled = true
  }

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "ml-inference-${var.environment}"
    }
  }
}

# Auto Scaling 组
resource "aws_autoscaling_group" "ml_app" {
  name                = "ml-inference-asg-${var.environment}"
  vpc_zone_identifier = aws_subnet.private[*].id
  target_group_arns   = [aws_lb_target_group.ml_app.arn]
  health_check_type   = "ELB"
  health_check_grace_period = 300

  min_size         = var.min_instances
  max_size         = var.max_instances
  desired_capacity = var.min_instances

  launch_template {
    id      = aws_launch_template.ml_app.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "ml-inference-instance-${var.environment}"
    propagate_at_launch = true
  }
}

# Auto Scaling 策略
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "ml-scale-up-${var.environment}"
  scaling_adjustment     = 2
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.ml_app.name
}

resource "aws_autoscaling_policy" "scale_down" {
  name                   = "ml-scale-down-${var.environment}"
  scaling_adjustment     = -1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.ml_app.name
}

# 用于 Auto Scaling 的 CloudWatch 告警
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "ml-high-cpu-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "120"
  statistic           = "Average"
  threshold           = "70"

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.ml_app.name
  }

  alarm_actions = [aws_autoscaling_policy.scale_up.arn]
}

resource "aws_cloudwatch_metric_alarm" "low_cpu" {
  alarm_name          = "ml-low-cpu-${var.environment}"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "120"
  statistic           = "Average"
  threshold           = "30"

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.ml_app.name
  }

  alarm_actions = [aws_autoscaling_policy.scale_down.arn]
}

# 最新 Ubuntu AMI 的数据源
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

**负载均衡器**（`load-balancer.tf`）：
```hcl
# Application Load Balancer
resource "aws_lb" "ml_app" {
  name               = "ml-alb-${var.environment}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = var.environment == "prod" ? true : false

  tags = {
    Name = "ml-alb-${var.environment}"
  }
}

# 目标组
resource "aws_lb_target_group" "ml_app" {
  name     = "ml-tg-${var.environment}"
  port     = 5000
  protocol = "HTTP"
  vpc_id   = aws_vpc.ml_vpc.id

  health_check {
    enabled             = true
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }

  deregistration_delay = 30

  tags = {
    Name = "ml-tg-${var.environment}"
  }
}

# 监听器
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.ml_app.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.ml_app.arn
  }
}
```

**用于模型的 S3 存储桶**（`storage.tf`）：
```hcl
resource "aws_s3_bucket" "models" {
  bucket = "ml-models-${var.environment}-${data.aws_caller_identity.current.account_id}"

  tags = {
    Name = "ml-models-${var.environment}"
  }
}

resource "aws_s3_bucket_versioning" "models" {
  bucket = aws_s3_bucket.models.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "models" {
  bucket = aws_s3_bucket.models.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

data "aws_caller_identity" "current" {}
```

**输出**（`outputs.tf`）：
```hcl
output "alb_dns_name" {
  description = "DNS name of the load balancer"
  value       = aws_lb.ml_app.dns_name
}

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.ml_vpc.id
}

output "model_bucket_name" {
  description = "S3 bucket for ML models"
  value       = aws_s3_bucket.models.bucket
}

output "autoscaling_group_name" {
  description = "Auto Scaling Group name"
  value       = aws_autoscaling_group.ml_app.name
}
```

### 1.5 Terraform 工作流命令

**初始化 Terraform**（下载提供者和模块）：
```bash
terraform init
```

**格式化代码**（标准化格式）：
```bash
terraform fmt -recursive
```

**验证配置**（检查错误）：
```bash
terraform validate
```

**计划变更**（预览将要创建的内容）：
```bash
terraform plan -var="environment=dev" -out=tfplan
```

**应用变更**（创建基础设施）：
```bash
terraform apply tfplan
```

**显示当前状态**：
```bash
terraform show
```

**销毁基础设施**（删除所有资源）：
```bash
terraform destroy -var="environment=dev"
```

**Terraform 状态管理**：
```bash
# 列出状态中的所有资源
terraform state list

# 显示特定资源的详细信息
terraform state show aws_lb.ml_app

# 从状态中移除资源（不删除实际资源）
terraform state rm aws_lb.ml_app
```

---

## 2. 容器编排：ECS 和 EKS

### 2.1 Amazon ECS（Elastic Container Service）

**ECS 架构**：
```
┌─────────────────────────────────────────────┐
│         Application Load Balancer           │
└────────────────┬────────────────────────────┘
                 │
      ┌──────────┴──────────┐
      │                     │
┌─────▼─────┐         ┌─────▼─────┐
│  ECS Task │         │  ECS Task │
│  Container│         │  Container│
│  Port 5000│         │  Port 5000│
└───────────┘         └───────────┘
      │                     │
┌─────▼──────────────────────▼─────┐
│        ECS Service               │
│  （期望数量：3 个任务）           │
└──────────────┬───────────────────┘
               │
┌──────────────▼───────────────────┐
│        ECS Cluster               │
│  （EC2 或 Fargate 启动类型）     │
└──────────────────────────────────┘
```

**ECS 概念**：
- **任务定义（Task Definition）**：应用的蓝图（类似 Dockerfile）
- **任务（Task）**：任务定义的运行实例
- **服务（Service）**：管理期望数量的任务，与负载均衡器集成
- **集群（Cluster）**：任务或服务的逻辑分组

### 2.2 ECS 任务定义示例

**创建任务定义 JSON**（`task-definition.json`）：
```json
{
  "family": "ml-inference-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ml-inference-task-role",
  "containerDefinitions": [
    {
      "name": "ml-inference-container",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/ml-inference:v1.0",
      "portMappings": [
        {
          "containerPort": 5000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "MODEL_PATH",
          "value": "/models/resnet50.pth"
        },
        {
          "name": "LOG_LEVEL",
          "value": "INFO"
        }
      ],
      "secrets": [
        {
          "name": "API_KEY",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:ml-api-key-abc123"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/ml-inference",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

**注册任务定义**：
```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

### 2.3 创建 ECS 服务

**创建 ECS 集群**（Fargate 启动类型）：
```bash
aws ecs create-cluster --cluster-name ml-inference-cluster
```

**创建带 ALB 集成的 ECS 服务**：
```bash
aws ecs create-service \
  --cluster ml-inference-cluster \
  --service-name ml-inference-service \
  --task-definition ml-inference-task:1 \
  --desired-count 3 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-0abc123,subnet-0def456],
    securityGroups=[sg-0ghi789],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/ml-tg/abc123,containerName=ml-inference-container,containerPort=5000" \
  --health-check-grace-period-seconds 60
```

**为 ECS 服务启用 Auto Scaling**：
```bash
# 注册可扩展目标
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/ml-inference-cluster/ml-inference-service \
  --min-capacity 2 \
  --max-capacity 10

# 基于 CPU 利用率的扩展策略
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/ml-inference-cluster/ml-inference-service \
  --policy-name cpu-scaling-policy \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

### 2.4 Amazon EKS（Elastic Kubernetes Service）

**为什么选择 EKS 而不是 ECS？**
- **可移植性**：在任何 Kubernetes 集群上运行（GCP GKE、Azure AKS、本地）
- **生态系统**：庞大的工具生态系统（Helm、Istio、ArgoCD）
- **灵活性**：对网络、存储、调度的更多控制
- **多云**：更容易采用多云策略

**EKS 架构**：
```
┌────────────────────────────────────────┐
│         EKS Control Plane              │
│  （AWS 管理 - API Server、etcd）       │
└────────────────┬───────────────────────┘
                 │
      ┌──────────┴──────────┐
      │                     │
┌─────▼─────┐         ┌─────▼─────┐
│  Worker   │         │  Worker   │
│  Node 1   │         │  Node 2   │
│  （EC2）  │         │  （EC2）  │
└───────────┘         └───────────┘
      │                     │
   ┌──▼──┐               ┌──▼──┐
   │ Pod │               │ Pod │
   └─────┘               └─────┘
```

### 2.5 使用 eksctl 创建 EKS 集群

**安装 eksctl**：
```bash
# 安装 eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# 验证安装
eksctl version
```

**创建 EKS 集群**（大约 15-20 分钟）：
```bash
eksctl create cluster \
  --name ml-inference-cluster \
  --region us-east-1 \
  --nodegroup-name ml-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed \
  --version 1.28
```

**配置 kubectl**：
```bash
aws eks update-kubeconfig --region us-east-1 --name ml-inference-cluster

# 验证连接
kubectl get nodes
```

### 2.6 在 EKS 上部署 ML 应用

**部署 YAML**（`ml-deployment.yaml`）：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-inference
  labels:
    app: ml-inference
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ml-inference
  template:
    metadata:
      labels:
        app: ml-inference
    spec:
      containers:
      - name: ml-inference
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/ml-inference:v1.0
        ports:
        - containerPort: 5000
        env:
        - name: MODEL_PATH
          value: "/models/resnet50.pth"
        - name: LOG_LEVEL
          value: "INFO"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ml-inference-service
spec:
  type: LoadBalancer
  selector:
    app: ml-inference
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5000
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ml-inference-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ml-inference
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

**部署到 EKS**：
```bash
# 应用部署
kubectl apply -f ml-deployment.yaml

# 检查部署状态
kubectl get deployments
kubectl get pods
kubectl get svc ml-inference-service

# 获取负载均衡器 URL
kubectl get svc ml-inference-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

# 测试推理端点
curl http://<load-balancer-url>/predict -X POST -H "Content-Type: application/json" -d '{"image_url": "https://example.com/cat.jpg"}'
```

**查看日志**：
```bash
# 获取 pod 名称
kubectl get pods -l app=ml-inference

# 查看特定 pod 的日志
kubectl logs <pod-name>

# 实时跟踪日志
kubectl logs -f <pod-name>
```

### 2.7 ECS vs EKS 决策矩阵

| 因素 | ECS | EKS |
|--------|-----|-----|
| **易用性** | ⭐⭐⭐⭐⭐ 简单，AWS 原生 | ⭐⭐⭐ 学习曲线较陡 |
| **成本** | ⭐⭐⭐⭐ 无控制平面成本 | ⭐⭐⭐ 控制平面 $0.10/小时 |
| **可移植性** | ⭐⭐ 仅 AWS | ⭐⭐⭐⭐⭐ 随处运行 |
| **生态系统** | ⭐⭐⭐ 增长中 | ⭐⭐⭐⭐⭐ 庞大的 Kubernetes 生态系统 |
| **集成度** | ⭐⭐⭐⭐⭐ 深度 AWS 集成 | ⭐⭐⭐⭐ AWS 集成良好 |
| **灵活性** | ⭐⭐⭐ 自定义有限 | ⭐⭐⭐⭐⭐ 高度可定制 |

**对初级工程师的建议**：
- **从 ECS Fargate 开始**以求简单
- 熟悉容器后**学习 Kubernetes 概念**和 EKS
- 如果多云或可移植性重要，**对生产 ML 使用 EKS**

---

## 3. 使用 AWS Lambda 的无服务器 ML 推理

### 3.1 何时为 ML 使用无服务器？

**适合 Lambda 的场景**：
- ✅ **不频繁预测**：零星流量（< 1000 请求/小时）
- ✅ **小型模型**：模型大小 < 250MB（未压缩）
- ✅ **快速推理**：预测时间 < 15 分钟
- ✅ **变化负载**：流量不可预测地波动
- ✅ **成本优化**：仅在运行时付费

**不适合 Lambda 的场景**：
- ❌ **大型模型**：深度学习模型 > 250MB
- ❌ **长时间推理**：预测耗时 > 15 分钟
- ❌ **高吞吐量**：持续高流量（更适合 ECS/EKS）
- ❌ **需要 GPU**：Lambda 不支持 GPU

### 3.2 Lambda 推理示例

**Lambda 函数**（`lambda_function.py`）：
```python
import json
import boto3
import pickle
import numpy as np

# 初始化 S3 客户端
s3 = boto3.client('s3')

# 用于缓存模型的全局变量（跨调用持久化）
model = None
MODEL_BUCKET = 'ml-models-prod'
MODEL_KEY = 'sklearn_model.pkl'

def load_model():
    """从 S3 加载模型（首次调用后缓存）"""
    global model
    if model is None:
        print("Loading model from S3...")
        obj = s3.get_object(Bucket=MODEL_BUCKET, Key=MODEL_KEY)
        model = pickle.loads(obj['Body'].read())
        print("Model loaded successfully")
    return model

def lambda_handler(event, context):
    """
    ML 推理的 Lambda 处理程序

    事件格式：
    {
        "features": [1.2, 3.4, 5.6, 7.8]
    }
    """
    try:
        # 解析输入
        body = json.loads(event['body']) if 'body' in event else event
        features = np.array(body['features']).reshape(1, -1)

        # 加载模型（首次调用后缓存）
        model = load_model()

        # 进行预测
        prediction = model.predict(features)
        probability = model.predict_proba(features)

        # 返回响应
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'prediction': int(prediction[0]),
                'probability': float(probability[0][1]),
                'model_version': '1.0'
            })
        }

    except Exception as e:
        print(f"Error during inference: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': str(e)
            })
        }
```

**打包 Lambda 函数**：
```bash
# 创建部署包目录
mkdir lambda-package
cd lambda-package

# 安装依赖
pip install numpy scikit-learn -t .

# 复制函数代码
cp ../lambda_function.py .

# 创建 ZIP 文件
zip -r lambda-package.zip .

# 上传到 S3
aws s3 cp lambda-package.zip s3://my-lambda-code-bucket/
```

**创建 Lambda 函数**：
```bash
# 为 Lambda 创建 IAM 角色
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ml-lambda-execution-role \
  --assume-role-policy-document file://trust-policy.json

# 附加策略
aws iam attach-role-policy \
  --role-name ml-lambda-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy \
  --role-name ml-lambda-execution-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 创建 Lambda 函数
aws lambda create-function \
  --function-name ml-inference-lambda \
  --runtime python3.11 \
  --role arn:aws:iam::123456789012:role/ml-lambda-execution-role \
  --handler lambda_function.lambda_handler \
  --code S3Bucket=my-lambda-code-bucket,S3Key=lambda-package.zip \
  --timeout 30 \
  --memory-size 1024 \
  --environment Variables={MODEL_BUCKET=ml-models-prod,MODEL_KEY=sklearn_model.pkl}
```

### 3.3 API Gateway 集成

**创建 REST API**：
```bash
# 创建 REST API
aws apigateway create-rest-api \
  --name "ML Inference API" \
  --description "Serverless ML inference API"

# 获取 API ID（从上面输出）
API_ID="abc123defg"

# 获取根资源 ID
aws apigateway get-resources --rest-api-id $API_ID

ROOT_ID="xyz789"

# 创建 /predict 资源
aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_ID \
  --path-part predict

PREDICT_RESOURCE_ID="pred123"

# 创建 POST 方法
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $PREDICT_RESOURCE_ID \
  --http-method POST \
  --authorization-type NONE

# 与 Lambda 集成
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $PREDICT_RESOURCE_ID \
  --http-method POST \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123456789012:function:ml-inference-lambda/invocations

# 授权 API Gateway 调用 Lambda
aws lambda add-permission \
  --function-name ml-inference-lambda \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:123456789012:$API_ID/*/POST/predict"

# 部署 API
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod
```

**测试 API**：
```bash
curl -X POST https://$API_ID.execute-api.us-east-1.amazonaws.com/prod/predict \
  -H "Content-Type: application/json" \
  -d '{"features": [1.2, 3.4, 5.6, 7.8]}'
```

---

## 4. 用于托管 ML 的 AWS SageMaker

### 4.1 SageMaker 组件

**SageMaker 平台**：
```
┌──────────────────────────────────────────────┐
│           SageMaker Studio                   │  （Jupyter 基础的 IDE）
└──────────────────┬───────────────────────────┘
                   │
    ┌──────────────┼──────────────┐
    │              │              │
┌───▼────┐   ┌─────▼─────┐  ┌────▼─────┐
│Training│   │Processing │  │Inference │
│  Jobs  │   │   Jobs    │  │Endpoints │
└────────┘   └───────────┘  └──────────┘
    │              │              │
    └──────────────┼──────────────┘
                   │
         ┌─────────▼─────────┐
         │   Model Registry  │
         └───────────────────┘
```

**关键 SageMaker 功能**：
- **内置算法**：XGBoost、Linear Learner、Image Classification
- **托管训练**：自动配置和扩展
- **超参数调优**：自动优化
- **模型注册表**：模型的版本控制
- **端点**：带自动扩缩容的托管推理
- **SageMaker Pipelines**：MLOps 工作流编排

### 4.2 SageMaker 训练作业示例

**使用 SageMaker 训练模型**：
```python
import sagemaker
from sagemaker import get_execution_role
from sagemaker.sklearn import SKLearn

# 初始化 SageMaker 会话
sagemaker_session = sagemaker.Session()
role = get_execution_role()

# 定义训练脚本
# （假设你本地目录中有 train.py）
sklearn_estimator = SKLearn(
    entry_point='train.py',
    role=role,
    instance_type='ml.m5.large',
    framework_version='1.2-1',
    py_version='py3',
    hyperparameters={
        'n_estimators': 100,
        'max_depth': 10,
        'learning_rate': 0.1
    }
)

# 开始训练作业
sklearn_estimator.fit({'training': 's3://my-bucket/training-data/'})
```

**训练脚本**（`train.py`）：
```python
import argparse
import os
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
import joblib

if __name__ == '__main__':
    parser = argparse.ArgumentParser()

    # 超参数
    parser.add_argument('--n_estimators', type=int, default=100)
    parser.add_argument('--max_depth', type=int, default=10)

    # SageMaker 特定参数
    parser.add_argument('--model-dir', type=str, default=os.environ.get('SM_MODEL_DIR'))
    parser.add_argument('--train', type=str, default=os.environ.get('SM_CHANNEL_TRAINING'))

    args = parser.parse_args()

    # 加载训练数据
    train_data = pd.read_csv(os.path.join(args.train, 'train.csv'))
    X_train = train_data.drop('target', axis=1)
    y_train = train_data['target']

    # 训练模型
    model = RandomForestClassifier(
        n_estimators=args.n_estimators,
        max_depth=args.max_depth
    )
    model.fit(X_train, y_train)

    # 保存模型
    joblib.dump(model, os.path.join(args.model_dir, 'model.joblib'))
```

### 4.3 将模型部署到 SageMaker 端点

**部署训练好的模型**：
```python
# 将模型部署到端点
predictor = sklearn_estimator.deploy(
    initial_instance_count=2,
    instance_type='ml.t2.medium',
    endpoint_name='ml-inference-endpoint'
)

# 进行预测
import numpy as np
test_data = np.array([[1.2, 3.4, 5.6, 7.8]])
prediction = predictor.predict(test_data)
print(f"Prediction: {prediction}")
```

**配置 Auto Scaling**：
```python
import boto3

client = boto3.client('application-autoscaling')

# 注册可扩展目标
client.register_scalable_target(
    ServiceNamespace='sagemaker',
    ResourceId='endpoint/ml-inference-endpoint/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    MinCapacity=1,
    MaxCapacity=5
)

# 创建扩展策略
client.put_scaling_policy(
    PolicyName='ml-endpoint-scaling-policy',
    ServiceNamespace='sagemaker',
    ResourceId='endpoint/ml-inference-endpoint/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    PolicyType='TargetTrackingScaling',
    TargetTrackingScalingPolicyConfiguration={
        'TargetValue': 70.0,
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'SageMakerVariantInvocationsPerInstance'
        },
        'ScaleInCooldown': 300,
        'ScaleOutCooldown': 60
    }
)
```

### 4.4 SageMaker Pipelines（MLOps）

**定义 ML 流水线**：
```python
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.steps import ProcessingStep, TrainingStep
from sagemaker.workflow.parameters import ParameterInteger

# 定义流水线参数
instance_count = ParameterInteger(name="InstanceCount", default_value=1)

# 处理步骤
processing_step = ProcessingStep(
    name="PreprocessData",
    # ... 处理配置
)

# 训练步骤
training_step = TrainingStep(
    name="TrainModel",
    estimator=sklearn_estimator,
    inputs={
        'training': processing_step.properties.ProcessingOutputConfig.Outputs['train'].S3Output.S3Uri
    }
)

# 创建流水线
pipeline = Pipeline(
    name="ml-training-pipeline",
    parameters=[instance_count],
    steps=[processing_step, training_step]
)

# 创建/更新流水线
pipeline.upsert(role_arn=role)

# 开始流水线执行
execution = pipeline.start()
```

---

## 5. ML 工作负载的成本优化

### 5.1 计算成本优化

**EC2 定价模型**：
| 模型 | 成本节省 | 用例 |
|-------|--------------|----------|
| **按需** | 基准（1x） | 生产推理（不可预测负载） |
| **预留实例** | 30-40% | 稳态工作负载（1-3 年承诺） |
| **Spot 实例** | 50-90% | 批处理训练（容错） |
| **Savings Plans** | 25-35% | 灵活承诺（任何实例系列） |

**使用 Spot 实例进行训练**：
```bash
# 使用 Spot 实例启动训练作业
aws sagemaker create-training-job \
  --training-job-name ml-training-spot \
  --algorithm-specification TrainingImage=<image-uri>,TrainingInputMode=File \
  --role-arn arn:aws:iam::123456789012:role/SageMakerRole \
  --input-data-config file://input-data-config.json \
  --output-data-config S3OutputPath=s3://my-bucket/output \
  --resource-config InstanceType=ml.p3.2xlarge,InstanceCount=1,VolumeSizeInGB=50 \
  --stopping-condition MaxRuntimeInSeconds=86400 \
  --enable-managed-spot-training \
  --checkpoint-config S3Uri=s3://my-bucket/checkpoints
```

**Spot 实例最佳实践**：
- ✅ **用于训练**：模型可以从检查点恢复
- ✅ **启用 Checkpointing**：频繁保存进度
- ✅ **使用多种实例类型**：提高 Spot 可用性
- ❌ **不要用于推理**：中断会影响用户请求

### 5.2 存储成本优化

**S3 存储类**：
| 类 | 成本 | 访问时间 | 用例 |
|-------|------|-------------|----------|
| **S3 Standard** | $0.023/GB | 即时 | 活跃模型 |
| **S3 Intelligent-Tiering** | $0.0025-0.023/GB | 即时 | 未知访问模式 |
| **S3 Glacier Instant Retrieval** | $0.004/GB | 即时 | 归档模型（每季度访问） |
| **S3 Glacier Deep Archive** | $0.00099/GB | 12 小时 | 长期归档 |

**生命周期策略示例**：
```json
{
  "Rules": [
    {
      "Id": "archive-old-models",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "models/"
      },
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ]
    },
    {
      "Id": "delete-old-logs",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "logs/"
      },
      "Expiration": {
        "Days": 30
      }
    }
  ]
}
```

**应用生命周期策略**：
```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket ml-models-bucket \
  --lifecycle-configuration file://lifecycle-policy.json
```

### 5.3 合理调整实例规格

**监控资源利用率**：
```bash
# 获取 EC2 CPU 利用率的 CloudWatch 指标
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc123def456 \
  --start-time 2025-10-01T00:00:00Z \
  --end-time 2025-10-23T23:59:59Z \
  --period 3600 \
  --statistics Average
```

**规格调整决策矩阵**：
- **CPU < 30%**：考虑更小的实例类型
- **CPU 30-70%**：最优利用率
- **CPU > 70%**：考虑更大实例或水平扩展
- **内存 < 50%**：内存过度配置
- **内存 > 80%**：OOM 错误风险，增加内存

### 5.4 成本监控与告警

**启用成本分配标签**：
```bash
aws ce create-cost-category-definition \
  --name "ML-Project-Costs" \
  --rules '[
    {
      "Value": "training",
      "Rule": {
        "Tags": {
          "Key": "Workload",
          "Values": ["training"]
        }
      }
    },
    {
      "Value": "inference",
      "Rule": {
        "Tags": {
          "Key": "Workload",
          "Values": ["inference"]
        }
      }
    }
  ]'
```

### 5.5 成本优化检查清单

**基础设施**：
- [ ] 对训练作业使用 Spot 实例
- [ ] 为推理启用自动扩缩容
- [ ] 根据利用率合理调整实例规格
- [ ] 对稳态工作负载使用预留实例
- [ ] 删除未使用的资源（已停止的 EC2、旧快照）

**存储**：
- [ ] 实施 S3 生命周期策略
- [ ] 对未知模式使用 S3 Intelligent-Tiering
- [ ] 存储前压缩大型数据集
- [ ] 删除旧的 CloudWatch 日志（> 30 天）
- [ ] 使用 EBS 快照而非保留卷

**网络**：
- [ ] 使用 VPC 端点避免 NAT Gateway 成本
- [ ] 最小化跨区域数据传输
- [ ] 使用 CloudFront 提供静态资产
- [ ] 仅在需要时启用 S3 Transfer Acceleration

**监控**：
- [ ] 设置成本分配标签
- [ ] 创建带告警的预算
- [ ] 每月查看 AWS Cost Explorer
- [ ] 使用 AWS Trusted Advisor 建议
- [ ] 启用 AWS Compute Optimizer

---

## 6. 生产级部署架构

### 6.1 完整 ML 基础设施架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         Route 53（DNS）                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                    CloudFront（CDN）                             │
│              （缓存预测结果、DDoS 防护）                         │
└────────────────────────────┬────────────────────────────────────┘
                             │
                 ┌───────────┴───────────┐
                 │                       │
        ┌────────▼────────┐     ┌────────▼────────┐
        │   VPC（us-east-1）│     │  VPC（us-west-2）│
        │                  │     │                  │
        │  ┌────────────┐  │     │  ┌────────────┐ │
        │  │    ALB     │  │     │  │    ALB     │ │
        │  └─────┬──────┘  │     │  └─────┬──────┘ │
        │        │         │     │        │        │
        │  ┌─────▼──────┐  │     │  ┌─────▼──────┐ │
        │  │  ECS/EKS   │  │     │  │  ECS/EKS   │ │
        │  │  （Fargate）│  │     │  │  （Fargate）│ │
        │  │  3 个任务   │  │     │  │  3 个任务   │ │
        │  └─────┬──────┘  │     │  └─────┬──────┘ │
        │        │         │     │        │        │
        │  ┌─────▼──────┐  │     │  ┌─────▼──────┐ │
        │  │    RDS     │  │     │  │ RDS 副本   │ │
        │  │ Multi-AZ   │  │     │  │ （只读）    │ │
        │  └────────────┘  │     │  └────────────┘ │
        └──────────────────┘     └──────────────────┘
                 │                       │
                 └───────────┬───────────┘
                             │
                    ┌────────▼────────┐
                    │  S3（模型）     │
                    │  - 版本控制     │
                    │  - 加密         │
                    │  - 生命周期     │
                    └─────────────────┘
                             │
                    ┌────────▼────────┐
                    │  CloudWatch     │
                    │  - 指标         │
                    │  - 日志         │
                    │  - 告警         │
                    └─────────────────┘
```

### 6.2 部署策略：蓝绿部署

**使用 ECS 的蓝绿部署**：
```bash
# 第 1 步：在现有版本（蓝）旁边部署新版本（绿）
aws ecs update-service \
  --cluster ml-inference-cluster \
  --service ml-inference-service \
  --task-definition ml-inference-task:2 \
  --desired-count 3 \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100"

# 第 2 步：等待新任务健康
aws ecs wait services-stable \
  --cluster ml-inference-cluster \
  --services ml-inference-service

# 第 3 步：监控指标（错误率、延迟）
# 如果指标正常，部署完成
# 如果指标异常，回滚：
aws ecs update-service \
  --cluster ml-inference-cluster \
  --service ml-inference-service \
  --task-definition ml-inference-task:1
```

**金丝雀部署**（逐步发布）：
```bash
# 将 10% 的流量路由到新版本
aws elbv2 modify-listener \
  --listener-arn <listener-arn> \
  --default-actions Type=forward,ForwardConfig='{
    "TargetGroups": [
      {"TargetGroupArn": "<blue-tg>", "Weight": 90},
      {"TargetGroupArn": "<green-tg>", "Weight": 10}
    ]
  }'

# 监控 30 分钟，然后增加到 50%
# 如果成功，将 100% 切换到绿色
```

---

## 7. 总结与最佳实践

### 关键要点

1. **基础设施即代码**：
   - 对生产始终使用 Terraform/CloudFormation
   - 对所有基础设施变更进行版本控制
   - 使用带锁定的远程状态

2. **容器编排**：
   - **ECS**：简单、AWS 原生、适合入门
   - **EKS**：可移植、强大、适合复杂工作负载
   - 对无服务器容器始终使用 Fargate

3. **无服务器 ML**：
   - 适合低流量、小型模型
   - 使用 Lambda + API Gateway 进行成本优化
   - 全局缓存模型以减少冷启动

4. **SageMaker**：
   - 托管平台减少运维开销
   - 用于端到端 ML 生命周期
   - 内置监控和自动扩缩容

5. **成本优化**：
   - 对训练使用 Spot 实例（节省 50-90%）
   - 为推理实施自动扩缩容
   - 根据实际使用情况合理调整实例规格
   - 使用 S3 生命周期策略管理存储

### 生产检查清单

**基础设施**：
- [ ] Multi-AZ 部署以实现高可用
- [ ] 配置了自动扩缩容（基于 CPU 和请求）
- [ ] 负载均衡器健康检查已启用
- [ ] 带公有和私有子网的 VPC
- [ ] 遵循最少权限的安全组

**监控**：
- [ ] 所有资源的 CloudWatch 指标
- [ ] 关键指标的 CloudWatch 告警
- [ ] 集中式日志（CloudWatch Logs 或 ELK）
- [ ] 成本监控和预算
- [ ] 性能监控（延迟、吞吐量）

**安全**：
- [ ] 最少权限的 IAM 角色
- [ ] 密钥存储在 Secrets Manager
- [ ] 静态和传输中加密
- [ ] 定期安全审计
- [ ] VPC Flow Logs 已启用

**运维**：
- [ ] 蓝绿或金丝雀部署策略
- [ ] 失败时自动回滚
- [ ] 备份和灾难恢复计划
- [ ] 运行手册文档
- [ ] 值班轮换和事件响应

---

## 下一步

在下一个模块中，你将：
- 完成部署 ML 基础设施的动手练习
- 构建完整的生产级 ML 平台
- 练习成本优化技术
- 实施来自模块 009 的监控和告警

**推荐阅读**：
- AWS Well-Architected Framework（ML Lens）
- Terraform AWS Provider 文档
- Kubernetes 文档
- SageMaker Developer Guide

---

**End of Lecture 04**
