# Exercise 01：AWS 账户设置与 IAM 最佳实践

**模块**：云平台
**难度**：入门
**预计时间**：2-3 小时
**前置要求**：讲座 01-04

---

## 学习目标

完成本练习后，你将：
1. 使用适当的安全配置设置 AWS 账户
2. 使用凭证和配置文件配置 AWS CLI
3. 遵循最小权限创建 IAM 用户、组和角色
4. 实施多因素认证（MFA）以增强安全性
5. 为成本分配和治理创建标签策略
6. 建立账单告警和预算

---

## 概述

这个基础练习使用安全最佳实践设置你的 AWS 环境。你将学习管理 IAM 身份、配置 AWS CLI 和实施成本控制——这对所有 AWS 工作都至关重要。

**实际场景**：你是一名新加入团队的初级 AI 基础设施工程师。你的首要任务是按照公司安全政策，设置一个安全的 AWS 账户用于开发 ML 基础设施。

---

## 第一部分：AWS 账户设置与安全加固

### 任务 1.1：创建 AWS 账户（如果需要）

**如果你还没有 AWS 账户**：

1. 访问 https://aws.amazon.com/
2. 点击"Create an AWS Account"
3. 按照注册流程操作：
   - 邮箱地址（使用个人或工作邮箱）
   - 账户名称：`ml-infrastructure-dev`
   - 根用户密码（使用 16 位以上强密码）
   - 联系信息
   - 支付方式（需要信用卡）
   - 身份验证（电话）
   - 选择"Free Tier"支持计划

**重要**：根用户拥有无限访问权限。你需要保护此账户并创建有限权限的用户。

### 任务 1.2：保护根账户

**为什么？** 根账户可以在你的 AWS 账户中做**任何事情**（删除所有资源、更改账单、关闭账户）。必须保护它。

**步骤**：

1. **为根用户启用 MFA**：
   - 以根用户身份登录 AWS 控制台
   - 点击你的账户名（右上角）→"Security credentials"
   - 在"Multi-factor authentication（MFA）"部分，点击"Assign MFA device"
   - 选择"Virtual MFA device"
   - 使用认证器应用（Google Authenticator、Authy、1Password）：
     - 扫描二维码
     - 输入两个连续的 MFA 代码
     - 点击"Assign MFA"

2. **创建强密码**：
   - 进入"Security credentials"→"Password"
   - 将密码更改为唯一、强密码（16 位以上，大小写混合、数字、符号）
   - 将密码存储在密码管理器中（1Password、LastPass、Bitwarden）

3. **删除根用户访问密钥**（如果存在）：
   - 在"Security credentials"中，检查"Access keys"部分
   - 如果有密钥存在，删除它们
   - **永远不要**为根用户创建访问密钥

**验证**：
```bash
# 尝试再次登录 - 应提示输入 MFA
# 这确认 MFA 工作正常
```

### 任务 1.3：启用 CloudTrail（审计日志）

**为什么？** CloudTrail 记录你账户中的所有 API 调用。对安全审计、合规和故障排查至关重要。

**步骤**：

1. 打开 CloudTrail 控制台：https://console.aws.amazon.com/cloudtrail/
2. 点击"Create trail"
3. 配置追踪：
   - Trail name：`ml-infrastructure-audit-trail`
   - Storage location：创建新的 S3 存储桶
   - Bucket name：`ml-infrastructure-audit-logs-<your-account-id>`
   - Log file SSE-KMS encryption：已启用（创建新的 KMS 密钥）
   - Log file validation：已启用
   - SNS notification delivery：已禁用（暂时）
4. 选择日志事件：
   - Management events：✅ 读取和写入
   - Data events：❌（本练习不需要）
   - Insights events：❌（增加成本）
5. 点击"Create trail"

**验证**：
```bash
# 先安装 AWS CLI（见任务 2.1）
aws cloudtrail describe-trails

# 预期输出：
# {
#   "trailList": [
#     {
#       "Name": "ml-infrastructure-audit-trail",
#       "S3BucketName": "ml-infrastructure-audit-logs-...",
#       ...
#     }
#   ]
# }
```

### 任务 1.4：启用 AWS GuardDuty（威胁检测）

**为什么？** GuardDuty 使用机器学习检测可疑活动（凭证泄露、加密货币挖矿、未授权访问）。

**步骤**：

1. 打开 GuardDuty 控制台：https://console.aws.amazon.com/guardduty/
2. 点击"Get Started"
3. 点击"Enable GuardDuty"
4. GuardDuty 将立即开始监控（30 天免费试用，之后典型开发使用约为每月 $4）

**验证**：
```bash
aws guardduty list-detectors

# 预期输出：
# {
#   "DetectorIds": [
#     "abc123def456ghi789..."
#   ]
# }
```

---

## 第二部分：AWS CLI 配置

### 任务 2.1：安装 AWS CLI v2

**Ubuntu/Debian**：
```bash
# 下载 AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# 安装 unzip（如果尚未安装）
sudo apt-get update && sudo apt-get install -y unzip

# 解压安装程序
unzip awscliv2.zip

# 运行安装程序
sudo ./aws/install

# 验证安装
aws --version
# 预期：aws-cli/2.x.x Python/3.x.x Linux/...

# 清理
rm -rf aws awscliv2.zip
```

**macOS**：
```bash
# 下载并安装
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# 验证
aws --version
```

**Windows**（PowerShell）：
```powershell
# 下载安装程序
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi

# 验证（先重启终端）
aws --version
```

### 任务 2.2：创建 IAM 管理员用户

**为什么？** 永远不要将根凭证用于日常工作。创建具有管理员权限的 IAM 用户。

**通过控制台的步骤**：

1. 打开 IAM 控制台：https://console.aws.amazon.com/iam/
2. 导航到"Users"→"Create user"
3. 用户详情：
   - User name：`ml-admin`
   - 选中"Provide user access to the AWS Management Console"
   - Console password：自定义密码（或自动生成）
   - ✅ 选中"Users must create a new password at next sign-in"
4. 点击"Next"
5. 权限：
   - 选择"Attach policies directly"
   - 搜索并选择：`AdministratorAccess`
   - 点击"Next"
6. 审查并创建用户
7. **重要**：保存控制台登录 URL 和初始密码

### 任务 2.3：为 CLI 创建访问密钥

**步骤**：

1. 在 IAM 控制台中，点击 `ml-admin` 用户
2. 进入"Security credentials"选项卡
3. 滚动到"Access keys"部分→点击"Create access key"
4. 选择用例："Command Line Interface（CLI）"
5. 勾选"I understand the above recommendation..."
6. 点击"Next"
7. 描述标签：`CLI access for ML infrastructure development`
8. 点击"Create access key"
9. **关键**：下载 CSV 文件或复制凭证：
   - Access key ID：`AKIAIOSFODNN7EXAMPLE`
   - Secret access key：`wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`
10. 将这些存储在密码管理器中——以后将看不到秘密密钥！

### 任务 2.4：使用配置文件配置 AWS CLI

**配置默认配置文件**：
```bash
# 配置默认配置文件
aws configure

# 系统将提示：
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: json
```

**配置其他配置文件**：
```bash
# 配置"dev"配置文件
aws configure --profile dev
# 输入相同的凭证，或为开发账户使用不同凭证

# 配置"prod"配置文件（如果你有多个账户）
aws configure --profile prod
```

**验证配置**：
```bash
# 列出配置的配置文件
cat ~/.aws/config

# 预期输出：
# [default]
# region = us-east-1
# output = json
#
# [profile dev]
# region = us-east-1
# output = json

# 测试配置
aws sts get-caller-identity

# 预期输出：
# {
#   "UserId": "AIDAI...",
#   "Account": "123456789012",
#   "Arn": "arn:aws:iam::123456789012:user/ml-admin"
# }

# 使用特定配置文件测试
aws sts get-caller-identity --profile dev
```

**设置环境变量**（可选，为了方便）：
```bash
# 添加到 ~/.bashrc 或 ~/.zshrc
echo 'export AWS_PROFILE=dev' >> ~/.bashrc
echo 'export AWS_DEFAULT_REGION=us-east-1' >> ~/.bashrc
source ~/.bashrc

# 验证
echo $AWS_PROFILE
# 预期：dev
```

---

## 第三部分：IAM 用户、组和角色

### 任务 3.1：创建带策略的 IAM 组

**为什么？** 组允许你一次性管理多个用户的权限。将策略附加到组，而不是单个用户。

**组设计**：
```
ml-admins         → AdministratorAccess
ml-developers     → EC2、S3、RDS、SageMaker（读/写）
ml-data-scientists → SageMaker、S3（EC2 只读）
ml-readonly       → ViewOnlyAccess
```

**通过 CLI 创建组**：

```bash
# 创建 ml-developers 组
aws iam create-group --group-name ml-developers

# 将策略附加到 ml-developers
aws iam attach-group-policy \
  --group-name ml-developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

aws iam attach-group-policy \
  --group-name ml-developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

aws iam attach-group-policy \
  --group-name ml-developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonRDSFullAccess

aws iam attach-group-policy \
  --group-name ml-developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess

# 创建 ml-data-scientists 组
aws iam create-group --group-name ml-data-scientists

aws iam attach-group-policy \
  --group-name ml-data-scientists \
  --policy-arn arn:aws:iam::aws:policy/AmazonSageMakerFullAccess

aws iam attach-group-policy \
  --group-name ml-data-scientists \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam attach-group-policy \
  --group-name ml-data-scientists \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

# 创建 ml-readonly 组
aws iam create-group --group-name ml-readonly

aws iam attach-group-policy \
  --group-name ml-readonly \
  --policy-arn arn:aws:iam::aws:policy/ViewOnlyAccess
```

**验证**：
```bash
# 列出所有组
aws iam list-groups

# 获取附加到组的策略
aws iam list-attached-group-policies --group-name ml-developers
```

### 任务 3.2：创建 IAM 用户并分配到组

**创建用户**：

```bash
# 创建开发人员用户
aws iam create-user --user-name alice-developer

# 将用户添加到 ml-developers 组
aws iam add-user-to-group \
  --user-name alice-developer \
  --group-name ml-developers

# 为 Alice 创建控制台登录配置文件
aws iam create-login-profile \
  --user-name alice-developer \
  --password "TempPassword123!" \
  --password-reset-required

# 创建数据科学家用户
aws iam create-user --user-name bob-data-scientist

aws iam add-user-to-group \
  --user-name bob-data-scientist \
  --group-name ml-data-scientists

aws iam create-login-profile \
  --user-name bob-data-scientist \
  --password "TempPassword456!" \
  --password-reset-required

# 创建只读用户
aws iam create-user --user-name charlie-readonly

aws iam add-user-to-group \
  --user-name charlie-readonly \
  --group-name ml-readonly

aws iam create-login-profile \
  --user-name charlie-readonly \
  --password "TempPassword789!" \
  --password-reset-required
```

**验证**：
```bash
# 列出所有用户
aws iam list-users

# 获取特定用户的组
aws iam list-groups-for-user --user-name alice-developer

# 获取用户的有效权限
aws iam list-attached-user-policies --user-name alice-developer
aws iam list-groups-for-user --user-name alice-developer
```

### 任务 3.3：为 EC2 创建 IAM 角色

**为什么？** 角色允许 EC2 实例访问 AWS 服务，而无需在代码中嵌入凭证。

**为 ML 推理创建 EC2 角色**：

```bash
# 创建信任策略（允许 EC2 承担此角色）
cat > ec2-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# 创建角色
aws iam create-role \
  --role-name ml-inference-ec2-role \
  --assume-role-policy-document file://ec2-trust-policy.json \
  --description "Role for EC2 instances running ML inference"

# 将策略附加到角色
aws iam attach-role-policy \
  --role-name ml-inference-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam attach-role-policy \
  --role-name ml-inference-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# 创建用于将推理日志写入 S3 的自定义策略
cat > inference-logs-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::ml-inference-logs-*/*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ml-inference-logs-write \
  --policy-document file://inference-logs-policy.json

# 从上面输出获取策略 ARN，然后附加它
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam attach-role-policy \
  --role-name ml-inference-ec2-role \
  --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/ml-inference-logs-write

# 创建实例配置文件（将角色附加到 EC2 所需）
aws iam create-instance-profile \
  --instance-profile-name ml-inference-ec2-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name ml-inference-ec2-profile \
  --role-name ml-inference-ec2-role
```

**验证**：
```bash
# 列出角色
aws iam list-roles | grep ml-inference

# 获取角色详情
aws iam get-role --role-name ml-inference-ec2-role

# 列出附加到角色的策略
aws iam list-attached-role-policies --role-name ml-inference-ec2-role
```

### 任务 3.4：创建自定义 IAM 策略（最小权限）

**场景**：创建一个允许数据科学家使用 SageMaker 但将实例类型限制为经济高效选项的策略。

**创建自定义策略**：

```bash
cat > sagemaker-restricted-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SageMakerAllowedActions",
      "Effect": "Allow",
      "Action": [
        "sagemaker:CreateTrainingJob",
        "sagemaker:CreateNotebookInstance",
        "sagemaker:DescribeTrainingJob",
        "sagemaker:DescribeNotebookInstance",
        "sagemaker:StopTrainingJob",
        "sagemaker:StopNotebookInstance"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "sagemaker:InstanceTypes": [
            "ml.t3.medium",
            "ml.t3.large",
            "ml.m5.large",
            "ml.m5.xlarge"
          ]
        }
      }
    },
    {
      "Sid": "S3AccessForSageMaker",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::ml-training-data-*",
        "arn:aws:s3:::ml-training-data-*/*"
      ]
    },
    {
      "Sid": "DenyExpensiveInstances",
      "Effect": "Deny",
      "Action": [
        "sagemaker:CreateTrainingJob",
        "sagemaker:CreateNotebookInstance"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "sagemaker:InstanceTypes": [
            "ml.p3.8xlarge",
            "ml.p3.16xlarge",
            "ml.p4d.24xlarge"
          ]
        }
      }
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name sagemaker-cost-optimized-access \
  --policy-document file://sagemaker-restricted-policy.json \
  --description "Allows SageMaker usage but restricts expensive instance types"

# 附加到 ml-data-scientists 组
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam attach-group-policy \
  --group-name ml-data-scientists \
  --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/sagemaker-cost-optimized-access
```

**验证**：
```bash
# 以 bob-data-scientist 测试（需要 bob 的凭证）
# 这应该成功：
aws sagemaker create-notebook-instance \
  --notebook-instance-name test-notebook \
  --instance-type ml.t3.medium \
  --role-arn <role-arn>

# 这应该失败：
aws sagemaker create-notebook-instance \
  --notebook-instance-name test-notebook-expensive \
  --instance-type ml.p3.8xlarge \
  --role-arn <role-arn>
# 预期错误：User is not authorized to perform sagemaker:CreateNotebookInstance
```

---

## 第四部分：成本分配的标签策略

### 任务 4.1：定义标签标准

**为什么？** 标签允许你按项目、环境、团队和负责人跟踪成本。对成本优化至关重要。

**标签标准**：
```
必需标签（所有资源）：
  - Project      : ml-infrastructure | data-pipeline | model-training
  - Environment  : dev | staging | prod
  - Owner        : alice | bob | team-ml
  - CostCenter   : engineering | data-science | research

可选标签：
  - Name         : 人类可读的资源名称
  - ManagedBy    : terraform | cloudformation | manual
  - Version      : v1.0、v2.0 等
  - Compliance   : hipaa | gdpr | sox
```

**创建标签策略**（需要 AWS Organizations）：

```bash
# 如果你启用了 AWS Organizations：
cat > tagging-policy.json <<EOF
{
  "tags": {
    "Project": {
      "tag_key": {
        "@@assign": "Project"
      },
      "tag_value": {
        "@@assign": ["ml-infrastructure", "data-pipeline", "model-training"]
      },
      "enforced_for": {
        "@@assign": ["ec2:instance", "s3:bucket", "rds:db", "sagemaker:*"]
      }
    },
    "Environment": {
      "tag_key": {
        "@@assign": "Environment"
      },
      "tag_value": {
        "@@assign": ["dev", "staging", "prod"]
      },
      "enforced_for": {
        "@@assign": ["ec2:instance", "s3:bucket", "rds:db"]
      }
    }
  }
}
EOF

# 启用标签策略（需要 AWS Organizations）
# aws organizations enable-policy-type --root-id <root-id> --policy-type TAG_POLICY
# aws organizations create-policy --name ml-tagging-policy --type TAG_POLICY --content file://tagging-policy.json
```

### 任务 4.2：将标签应用到资源

**给现有资源打标签**：

```bash
# 给 S3 存储桶打标签
aws s3api put-bucket-tagging \
  --bucket ml-models-bucket \
  --tagging 'TagSet=[
    {Key=Project,Value=ml-infrastructure},
    {Key=Environment,Value=dev},
    {Key=Owner,Value=alice},
    {Key=CostCenter,Value=engineering}
  ]'

# 给 EC2 实例打标签
INSTANCE_ID="i-0abc123def456"
aws ec2 create-tags \
  --resources $INSTANCE_ID \
  --tags \
    Key=Project,Value=ml-infrastructure \
    Key=Environment,Value=dev \
    Key=Owner,Value=alice \
    Key=CostCenter,Value=engineering \
    Key=Name,Value=ml-inference-server

# 使用 AWS CLI 打标签（通用）
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list arn:aws:s3:::my-bucket \
  --tags Project=ml-infrastructure,Environment=dev
```

**按标签查询资源**：

```bash
# 查找所有开发资源
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Environment,Values=dev

# 查找 alice 拥有的所有资源
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Owner,Values=alice

# 查找 ml-infrastructure 项目资源
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=Project,Values=ml-infrastructure
```

---

## 第五部分：账单告警与成本预算

### 任务 5.1：启用账单告警

**步骤**：

1. 打开 Billing 控制台：https://console.aws.amazon.com/billing/
2. 转到"Billing Preferences"
3. 启用：
   - ✅ Receive PDF Invoice By Email
   - ✅ Receive Free Tier Usage Alerts
   - ✅ Receive Billing Alerts
4. 输入用于告警的电子邮件地址
5. 点击"Save preferences"

### 任务 5.2：创建 CloudWatch 账单告警

**创建用于告警的 SNS 主题**：

```bash
# 创建 SNS 主题
aws sns create-topic --name billing-alerts

# 订阅电子邮件到主题
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:billing-alerts \
  --protocol email \
  --notification-endpoint your-email@example.com

# 通过电子邮件确认订阅（检查收件箱）
```

**创建 CloudWatch 告警**（必须在 us-east-1）：

```bash
# 切换到 us-east-1 区域（账单指标仅在此区域可用）
aws cloudwatch put-metric-alarm \
  --alarm-name billing-alarm-50usd \
  --alarm-description "Alert when estimated charges exceed $50" \
  --metric-name EstimatedCharges \
  --namespace AWS/Billing \
  --statistic Maximum \
  --period 21600 \
  --evaluation-periods 1 \
  --threshold 50 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=Currency,Value=USD \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:billing-alerts \
  --region us-east-1
```

**验证**：
```bash
aws cloudwatch describe-alarms --alarm-names billing-alarm-50usd --region us-east-1
```

### 任务 5.3：创建 AWS 预算

**创建月度预算**（$100 限制）：

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > budget.json <<EOF
{
  "BudgetName": "ml-infrastructure-monthly-budget",
  "BudgetLimit": {
    "Amount": "100",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": {
    "TagKeyValue": ["user:Project\$ml-infrastructure"]
  }
}
EOF

cat > notifications.json <<EOF
[
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      {
        "SubscriptionType": "EMAIL",
        "Address": "your-email@example.com"
      }
    ]
  },
  {
    "Notification": {
      "NotificationType": "FORECASTED",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 100,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      {
        "SubscriptionType": "EMAIL",
        "Address": "your-email@example.com"
      }
    ]
  }
]
EOF

aws budgets create-budget \
  --account-id $ACCOUNT_ID \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```

**验证**：
```bash
aws budgets describe-budgets --account-id $ACCOUNT_ID
```

---

## 第六部分：IAM 最佳实践检查清单

### 任务 6.1：安全审计

**运行 IAM 安全审计**：

```bash
# 检查没有 MFA 的用户
aws iam get-credential-report || aws iam generate-credential-report
sleep 5  # 等待报告生成
aws iam get-credential-report --output text | \
  awk -F',' '$4 == "false" && $1 != "<root_account>" {print "User without MFA:", $1}'

# 检查未使用的访问密钥（> 90 天）
aws iam get-credential-report --output text | \
  awk -F',' -v today=$(date +%s) '
    NR > 1 {
      if ($11 != "N/A" && $11 != "no_information") {
        cmd = "date -d " $11 " +%s"
        cmd | getline key_date
        close(cmd)
        if ((today - key_date) > 7776000) {  # 90 天，以秒计
          print "Unused access key (>90 days):", $1, "Last used:", $11
        }
      }
    }
  '

# 检查过于宽松的策略
aws iam list-policies --scope Local --query 'Policies[?PolicyName==`AdminAccess`]'

# 检查根账户使用情况（应为零）
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=root \
  --max-results 10 \
  --query 'Events[].{Time:EventTime,Event:EventName}'
```

### 任务 6.2：实施 IAM 最佳实践

**检查清单**：

- [ ] **根账户**：
  - [ ] 根账户已启用 MFA
  - [ ] 根账户无访问密钥
  - [ ] 强密码（16 位以上）
  - [ ] 根账户仅用于账单/账户关闭

- [ ] **IAM 用户**：
  - [ ] 所有用户均已启用 MFA
  - [ ] 用户分配到组（非单个策略）
  - [ ] 访问密钥每 90 天轮换
  - [ ] 非活跃用户已禁用或删除

- [ ] **IAM 角色**：
  - [ ] EC2 实例使用角色（非嵌入凭证）
  - [ ] 应用最小权限原则
  - [ ] 定期审查信任策略

- [ ] **策略**：
  - [ ] 无 `"Effect": "Allow", "Action": "*", "Resource": "*"` 的策略
  - [ ] 在适当处使用条件块
  - [ ] 自定义策略遵循最小权限

- [ ] **监控**：
  - [ ] 所有区域已启用 CloudTrail
  - [ ] GuardDuty 已启用
  - [ ] 账单告警已配置
  - [ ] 已创建预算

**生成 IAM 最佳实践报告**：

```bash
cat > iam-audit.sh <<'EOF'
#!/bin/bash

echo "=== IAM Security Audit ==="
echo ""

# 检查根账户 MFA
echo "1. Root Account MFA Status:"
aws iam get-account-summary | grep AccountMFAEnabled

# 检查用户数
echo ""
echo "2. Total IAM Users:"
aws iam list-users --query 'Users | length(@)'

# 检查没有 MFA 的用户
echo ""
echo "3. Users Without MFA:"
aws iam get-credential-report --output text | \
  awk -F',' '$4 == "false" && $1 != "<root_account>" {print $1}'

# 检查 CloudTrail 状态
echo ""
echo "4. CloudTrail Status:"
aws cloudtrail describe-trails --query 'trailList[].{Name:Name,Status:IsMultiRegionTrail}'

# 检查 GuardDuty 状态
echo ""
echo "5. GuardDuty Detectors:"
aws guardduty list-detectors

echo ""
echo "=== Audit Complete ==="
EOF

chmod +x iam-audit.sh
./iam-audit.sh
```

---

## 第七部分：验证与测试

### 任务 7.1：验证 IAM 配置

**测试用户权限**：

```bash
# 测试 alice-developer（应该有 EC2 访问权限）
aws ec2 describe-instances --profile alice-dev
# 应该成功

# 测试 alice-developer 尝试删除 IAM 用户（应该失败）
aws iam delete-user --user-name test-user --profile alice-dev
# 预期错误：User alice-developer is not authorized to perform iam:DeleteUser

# 测试 bob-data-scientist（应该有 SageMaker 访问权限，但实例类型受限）
aws sagemaker list-notebook-instances --profile bob-ds
# 应该成功

# 测试 charlie-readonly（应该只有读访问权限）
aws s3 ls --profile charlie-ro
# 应该成功

aws s3 rm s3://my-bucket/test.txt --profile charlie-ro
# 预期错误：Access Denied
```

### 任务 7.2：测试 IAM 角色

**使用 IAM 角色启动 EC2 实例**：

```bash
# 查找最新的 Amazon Linux 2 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text)

# 使用 IAM 角色启动实例
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --iam-instance-profile Name=ml-inference-ec2-profile \
  --tag-specifications 'ResourceType=instance,Tags=[
    {Key=Name,Value=iam-role-test},
    {Key=Project,Value=ml-infrastructure},
    {Key=Environment,Value=dev}
  ]'

# 从输出中获取实例 ID
INSTANCE_ID="i-0abc123..."

# 等待实例运行
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# 连接到实例（需要 SSH 密钥设置）
# 连接后，测试角色权限：
# $ aws s3 ls  # 应该成功（S3 只读）
# $ aws s3 mb s3://test-bucket  # 应该失败（无写入权限）
```

### 任务 7.3：验证成本告警

**模拟成本告警**（可选，会产生费用）：

```bash
# 启动一个更大的实例以触发账单告警
# 警告：这将产生费用（~$0.10/小时）
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type m5.large \
  --tag-specifications 'ResourceType=instance,Tags=[
    {Key=Name,Value=billing-test},
    {Key=Project,Value=ml-infrastructure}
  ]'

# 几小时后检查账单告警
aws cloudwatch describe-alarms --alarm-names billing-alarm-50usd --region us-east-1

# 重要：测试完成后终止实例
aws ec2 terminate-instances --instance-ids <instance-id>
```

---

## 挑战任务（可选）

### 挑战 1：实施密码策略

为 IAM 用户创建强密码策略：

```bash
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 5 \
  --hard-expiry
```

### 挑战 2：创建服务控制策略（SCP）

如果你有 AWS Organizations，创建一个 SCP 来防止用户禁用 CloudTrail：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

### 挑战 3：实施跨账户访问

创建一个允许另一个 AWS 账户的用户访问特定资源的角色（适用于多账户设置）。

---

## 交付物

到本练习结束时，你应该有：

1. ✅ 使用 MFA 保护了 AWS 根账户
2. ✅ 安装并配置了 AWS CLI 和配置文件
3. ✅ 遵循最小权限创建了 IAM 用户、组和角色
4. ✅ 实施了自定义 IAM 策略
5. ✅ 定义并应用了标签标准
6. ✅ 配置了账单告警和预算
7. ✅ 启用了 CloudTrail 和 GuardDuty
8. ✅ 生成了 IAM 安全审计报告

**证据**：
- IAM 仪表盘显示用户和组的截图
- `aws iam get-account-summary` 的输出
- `aws budgets describe-budgets --account-id <account-id>` 的输出
- IAM 审计脚本结果

---

## 故障排查

**问题**：安装后 `aws` 命令未找到

**解决方案**：
```bash
# 检查 AWS CLI 是否在 PATH 中
which aws

# 如果不在，添加到 PATH（Ubuntu/Debian）
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
```

**问题**：运行 AWS CLI 命令时提示访问被拒绝

**解决方案**：
```bash
# 验证凭证已配置
aws configure list

# 检查凭证是否有效
aws sts get-caller-identity

# 如果使用配置文件，明确指定配置文件
aws s3 ls --profile dev
```

**问题**：账单告警未触发

**解决方案**：
- 账单指标需要 6-8 小时更新
- 确保告警在 **us-east-1** 区域创建
- 验证 SNS 主题订阅已确认（检查电子邮件）
- 检查告警状态：`aws cloudwatch describe-alarms --region us-east-1`

---

## 下一步

现在你的 AWS 账户已正确配置，你已经准备好：
- **练习 02**：配置计算（EC2）和存储（S3、EBS）
- **练习 03**：使用 Terraform 构建生产 VPC
- **练习 04**：使用 ECS/EKS 部署容器化 ML 应用
- **练习 05**：SageMaker 训练和成本优化

**保护你的 IAM 凭证安全**——永远不要将它们提交到 Git 或共享！
