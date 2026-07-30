# Lecture 02：用于 AI 基础设施的 AWS 核心服务

## 讲座概述

Amazon Web Services 提供 200+ 服务，但掌握一套核心基础服务就能让你构建健壮的 AI/ML 基础设施。本讲座提供你日常使用的 AWS 基本服务的动手实践：EC2 用于计算、S3 用于存储、RDS 用于数据库、IAM 用于安全和访问管理。

你将通过实际的 AWS CLI 示例学习如何配置虚拟机、管理对象存储、部署托管数据库以及实施最小权限安全策略。到本讲座结束时，你将能够自信地使用 AWS 控制台、通过 CLI 自动化任务，并为 ML 工作负载设计基本的云基础设施。

**预计阅读时间：** 75-90 分钟
**动手实验：** 练习 02 – 计算与存储基础
**前置要求：** 带免费套餐的 AWS 账户、已安装并配置 AWS CLI、基础 Linux 命令行技能

---

## 1. AWS 账户设置与配置

### 1.1 AWS 免费套餐概览

AWS 为新账户提供 **12 个月免费套餐**，加上始终免费的服务：

**12 个月免费套餐：**
- **EC2**：每月 750 小时 t2.micro 或 t3.micro 实例
- **S3**：5GB 标准存储、20000 次 GET 请求、2000 次 PUT 请求
- **RDS**：每月 750 小时 db.t2.micro、20GB 存储
- **Lambda**：每月 100 万次请求、400000 GB-秒计算
- **CloudWatch**：10 个自定义指标、10 个告警

**始终免费：**
- **DynamoDB**：25GB 存储、25 个读/写容量单位
- **Lambda**：每月 100 万次请求（持续）
- **SNS**：每月 100 万次发布

**试用优惠（限时）：**
- **SageMaker**：每月 250 小时 notebook 实例（2 个月）
- **Rekognition**：每月 5000 张图片（12 个月）

**成本管理提示：**
- **设置账单告警**：当费用超过 $10、$50、$100 时告警
- **使用 Cost Explorer**：按服务跟踪支出
- **给资源打标签**：`Environment=dev`、`Owner=your-name`
- **不使用时停止实例**：t2.micro 每小时 $0.0116 = 如果保持运行每月 $8.40

### 1.2 AWS CLI 安装与配置

**安装 AWS CLI v2：**

```bash
# Linux/macOS
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 验证
aws --version
# 输出：aws-cli/2.15.0 Python/3.11.6 Linux/5.10.0 exe/x86_64.ubuntu.22
```

**配置 AWS CLI：**

```bash
# 交互式配置
aws configure

# 提示：
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: json

# 配置存储在：
~/.aws/credentials  # 访问密钥
~/.aws/config       # 默认区域和输出格式
```

**为 CLI 创建 IAM 用户（最佳实践）：**

```bash
# 不要使用根账户访问密钥！
# 通过控制台创建 IAM 用户：
# IAM → 用户 → 添加用户 → 编程访问 → 附加 AdministratorAccess 策略

# 为不同账户/角色配置命名配置文件
aws configure --profile ml-dev
AWS Access Key ID: AKIAIOSFODNN7DEVEXAMPLE
AWS Secret Access Key: ...
Default region name: us-west-2
Default output format: json

# 使用配置文件：
aws s3 ls --profile ml-dev
```

**测试配置：**

```bash
# 列出 S3 存储桶（新账户应返回空列表）
aws s3 ls

# 获取调用者身份（验证你认证的是哪个账户/用户）
aws sts get-caller-identity
{
  "UserId": "AIDACKCEVSQ6C2EXAMPLE",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

### 1.3 AWS 区域与可用区

**区域：** AWS 拥有数据 centers 的地理位置（us-east-1、eu-west-1、ap-southeast-1 等）

**选择区域：**
- **延迟**：选择离用户最近的区域（美东用户选择 us-east-1，欧洲选择 eu-west-1）
- **合规**：数据驻留要求（GDPR、HIPAA）
- **服务可用性**：并非所有服务在所有区域都可用（查看 AWS 区域服务列表）
- **成本**：不同区域定价不同（us-east-1 通常最便宜）

**可用区（AZ）：** 区域内隔离的数据中心（us-east-1a、us-east-1b、us-east-1c）

**为什么 AZ 很重要：**
- **故障隔离**：每个 AZ 有独立的电源、制冷、网络
- **高可用性**：跨多个 AZ 部署；一个故障，其他继续运行
- **低延迟**：同一区域内的 AZ 通过高速、低延迟网络连接

**示例：**
```bash
# 列出你区域中的可用区
aws ec2 describe-availability-zones --region us-east-1

# 输出：
{
  "AvailabilityZones": [
    {
      "ZoneName": "us-east-1a",
      "State": "available",
      "RegionName": "us-east-1"
    },
    {
      "ZoneName": "us-east-1b",
      "State": "available",
      "RegionName": "us-east-1"
    },
    ...
  ]
}
```

---

## 2. EC2（Elastic Compute Cloud）- 虚拟机

### 2.1 EC2 基础

**EC2 = 云中的虚拟机**

**实例类型：**
- **通用型**：t3、m5（CPU、内存、网络均衡）
- **计算优化型**：c5、c6i（高 CPU，用于计算密集型工作负载）
- **内存优化型**：r5、x1（高 RAM，用于内存中的大数据集）
- **存储优化型**：i3、d2（高磁盘吞吐量，用于数据库）
- **加速计算型**：p3、p4（用于 ML 训练/推理的 GPU 实例）、inf1（用于推理的 AWS Inferentia）

**实例命名规则：**
```
p3.2xlarge
│ │  │
│ │  └─ 大小（nano、micro、small、medium、large、xlarge、2xlarge...）
│ └──── 代次（3 = 第 3 代）
└────── 系列（p = GPU 优化）
```

**免费套餐实例：** `t2.micro` 或 `t3.micro`（1 vCPU、1GB RAM、每月 750 小时免费）

**用于 ML 的 GPU 实例：**
- **p3.2xlarge**：1 个 NVIDIA V100 GPU、8 vCPU、61GB RAM（~$3/小时）
- **p3.8xlarge**：4 个 V100 GPU、32 vCPU、244GB RAM（~$12/小时）
- **p4d.24xlarge**：8 个 A100 GPU、96 vCPU、1.1TB RAM（~$32/小时）
- **g4dn.xlarge**：1 个 NVIDIA T4 GPU、4 vCPU、16GB RAM（~$0.50/小时，经济高效的推理）

### 2.2 启动 EC2 实例（CLI）

**第 1 步：创建密钥对（用于 SSH 访问）：**

```bash
# 创建 SSH 密钥对
aws ec2 create-key-pair \
  --key-name my-ml-keypair \
  --query 'KeyMaterial' \
  --output text > my-ml-keypair.pem

# 设置正确权限
chmod 400 my-ml-keypair.pem

# 密钥对存储在 AWS 中；私钥保存在本地
```

**第 2 步：查找 Amazon Machine Image（AMI）：**

```bash
# 在 us-east-1 中查找 Ubuntu 22.04 LTS AMI
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'Images[*].[ImageId,Name,CreationDate]' \
  --output table | head -20

# 记下最新的 ImageId（例如 ami-0c55b159cbfafe1f0）
```

**第 3 步：启动实例：**

```bash
# 启动 t2.micro 实例（免费套餐）
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --key-name my-ml-keypair \
  --security-group-ids sg-0123456789abcdef0 \
  --subnet-id subnet-0abcdef1234567890 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ml-dev-server},{Key=Environment,Value=dev},{Key=Owner,Value=your-name}]'

# 输出包含 InstanceId（例如 i-0abcdef1234567890）
```

**第 4 步：等待实例启动：**

```bash
# 检查实例状态
aws ec2 describe-instances \
  --instance-ids i-0abcdef1234567890 \
  --query 'Reservations[0].Instances[0].[State.Name,PublicIpAddress]' \
  --output table

# 等待 "running" 状态并记下 PublicIpAddress
```

**第 5 步：SSH 连接到实例：**

```bash
# 使用密钥对 SSH
ssh -i my-ml-keypair.pem ubuntu@<PUBLIC_IP>

# 连接后：
ubuntu@ip-172-31-0-123:~$ python3 --version
Python 3.10.6

# 安装 ML 库
sudo apt update
sudo apt install -y python3-pip
pip3 install torch torchvision numpy pandas
```

### 2.3 EC2 实例生命周期管理

**停止实例（保留数据、无计算费用）：**

```bash
# 停止实例
aws ec2 stop-instances --instance-ids i-0abcdef1234567890

# 实例状态：running → stopping → stopped
# 你仍需为 EBS 存储付费，但不为计算付费
```

**启动实例：**

```bash
# 启动已停止的实例
aws ec2 start-instances --instance-ids i-0abcdef1234567890

# 实例获得新的公有 IP（除非使用弹性 IP）
```

**终止实例（永久删除）：**

```bash
# 终止实例
aws ec2 terminate-instances --instance-ids i-0abcdef1234567890

# 实例状态：running → shutting-down → terminated
# EBS 卷被删除（除非设置为保留）
```

**重启实例：**

```bash
# 重启（优雅重启、保持相同 IP）
aws ec2 reboot-instances --instance-ids i-0abcdef1234567890
```

**最佳实践：**
- **给所有资源打标签**：Name、Environment、Owner、Project、CostCenter
- **不使用时停止实例**（开发/测试环境）
- **使用 CloudWatch 告警**自动停止闲置实例
- **为生产实例启用终止保护**

### 2.4 EC2 User Data（引导脚本）

**User Data** = 在实例启动时运行的脚本（自动化设置）

**示例：启动预装 PyTorch 的 EC2：**

```bash
# 创建 user data 脚本
cat > user-data.sh <<'EOF'
#!/bin/bash
# 该脚本在实例启动时运行

# 更新系统
apt-get update -y

# 安装 Python 和 pip
apt-get install -y python3-pip

# 安装 ML 库
pip3 install torch torchvision torchaudio \
  pandas numpy scikit-learn jupyter matplotlib

# 创建 Jupyter 配置
mkdir -p /home/ubuntu/.jupyter
cat > /home/ubuntu/.jupyter/jupyter_notebook_config.py <<EOL
c.NotebookApp.ip = '0.0.0.0'
c.NotebookApp.open_browser = False
c.NotebookApp.port = 8888
EOL

# 在后台启动 Jupyter
su - ubuntu -c "nohup jupyter notebook --no-browser --port=8888 > /home/ubuntu/jupyter.log 2>&1 &"

echo "Setup complete! Jupyter running on port 8888"
EOF

# 使用 user data 启动实例
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --key-name my-ml-keypair \
  --user-data file://user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ml-jupyter-server}]'

# 实例启动后，Jupyter 正在运行
# 访问：http://<PUBLIC_IP>:8888
```

**User Data 日志：**
```bash
# SSH 到实例并检查 user data 执行
sudo cat /var/log/cloud-init-output.log
```

### 2.5 EC2 存储选项

**1. 实例存储（临时）**
- **物理附加**到宿主机
- **临时**：实例停止/终止时丢失
- **高性能**：NVMe SSD，低延迟
- **用例**：临时处理、缓存、临时空间

**2. EBS（Elastic Block Store）**
- **网络附加**的块存储（像硬盘）
- **持久**：数据在实例停止/启动后保留
- **类型**：
  - **gp3**：通用型 SSD（3000-16000 IOPS，$0.08/GB/月）
  - **io2**：预置 IOPS SSD（最高 64000 IOPS，低延迟数据库）
  - **st1**：吞吐优化 HDD（大数据、日志处理）
  - **sc1**：冷 HDD（归档、不频繁访问）

**创建并挂载 EBS 卷：**

```bash
# 在与实例相同的可用区创建 100GB EBS 卷
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 100 \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=ml-data-volume}]'

# 记下 VolumeId（例如 vol-0abcdef1234567890）

# 挂载到实例
aws ec2 attach-volume \
  --volume-id vol-0abcdef1234567890 \
  --instance-id i-0abcdef1234567890 \
  --device /dev/sdf

# SSH 到实例并格式化/挂载
sudo mkfs -t ext4 /dev/sdf
sudo mkdir /mnt/ml-data
sudo mount /dev/sdf /mnt/ml-data

# 使挂载在重启后持久
echo '/dev/sdf /mnt/ml-data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
```

**EBS 快照（备份）：**

```bash
# 创建 EBS 卷的快照
aws ec2 create-snapshot \
  --volume-id vol-0abcdef1234567890 \
  --description "ML data backup before experiment"

# 从快照恢复（创建新卷）
aws ec2 create-volume \
  --snapshot-id snap-0123456789abcdef0 \
  --availability-zone us-east-1a
```

### 2.6 EC2 定价策略

**1. 按需（默认）**
- **价格**：t2.micro 每小时 $0.0116
- **用途**：开发、不可预测的工作负载
- **计费**：按秒（某些实例最少 1 小时）

**2. 预留实例**
- **承诺**：1 或 3 年
- **折扣**：相比按需最高 72%
- **付款**：全预付、部分预付、无预付
- **用途**：稳态生产工作负载

**3. Spot 实例**
- **价格**：最高 90% 折扣（基于市场定价）
- **风险**：可能被终止（提前 2 分钟警告）
- **用途**：容错工作负载（批处理、带 checkpointing 的 ML 训练）

**示例 Spot 请求：**

```bash
# 请求 spot 实例
aws ec2 request-spot-instances \
  --spot-price "0.05" \
  --instance-count 1 \
  --type "one-time" \
  --launch-specification '{
    "ImageId": "ami-0c55b159cbfafe1f0",
    "InstanceType": "p3.2xlarge",
    "KeyName": "my-ml-keypair"
  }'

# 监控 spot 实例
aws ec2 describe-spot-instance-requests
```

**4. Savings Plans**
- **承诺**：$/小时 持续 1-3 年
- **灵活性**：可以更改实例类型/区域
- **折扣**：最高 72%
- **用途**：灵活的计算承诺

**成本对比（p3.2xlarge，1 GPU）：**
- **按需**：$3.06/小时 = 每月 $2,203（24/7）
- **1 年预留**：$1.84/小时 = 每月 $1,325（节省 40%）
- **3 年预留**：$1.23/小时 = 每月 $886（节省 60%）
- **Spot**：$0.92/小时 = 每月 $662（节省 70%、可中断）

---

## 3. S3（Simple Storage Service）- 对象存储

### 3.1 S3 基础

**S3 = 对象存储**（文件 + 元数据，通过 HTTP/HTTPS 访问）

**关键概念：**
- **存储桶（Bucket）**：对象的容器（全局唯一名称）
- **对象（Object）**：文件 + 元数据（最大 5TB）
- **键（Key）**：对象的唯一标识符（类似文件路径）

**AI/ML 的用例：**
- **数据湖**：存储原始训练数据（图片、文本、日志）
- **模型产物**：存储训练好的模型（.pt、.h5、.pkl 文件）
- **特征存储**：存储预计算的特征（Parquet 文件）
- **结果**：存储预测输出、实验记录

**存储桶命名规则：**
- 在所有 AWS 账户中全局唯一
- 3-63 个字符，小写，无下划线
- 示例：`my-ml-training-data-123456789`

### 3.2 S3 存储桶操作（CLI）

**创建存储桶：**

```bash
# 在 us-east-1 创建存储桶
aws s3 mb s3://my-ml-training-data-123456789 --region us-east-1

# 验证存储桶已创建
aws s3 ls
# 输出：2025-10-23 12:00:00 my-ml-training-data-123456789
```

**上传文件：**

```bash
# 上传单个文件
aws s3 cp model.pth s3://my-ml-training-data-123456789/models/

# 上传目录（递归）
aws s3 cp ./training-data/ s3://my-ml-training-data-123456789/data/ --recursive

# 同步目录（仅上传更改的文件）
aws s3 sync ./training-data/ s3://my-ml-training-data-123456789/data/
```

**下载文件：**

```bash
# 下载单个文件
aws s3 cp s3://my-ml-training-data-123456789/models/model.pth ./

# 下载目录
aws s3 cp s3://my-ml-training-data-123456789/data/ ./training-data/ --recursive

# 同步到本地（仅下载新/更改的文件）
aws s3 sync s3://my-ml-training-data-123456789/data/ ./training-data/
```

**列出对象：**

```bash
# 列出存储桶中的所有对象
aws s3 ls s3://my-ml-training-data-123456789/ --recursive

# 列出对象并显示大小和时间戳
aws s3 ls s3://my-ml-training-data-123456789/models/ --human-readable

# 输出：
2025-10-23 12:30:00  150.5 MiB model-v1.0.pth
2025-10-23 13:45:00  200.1 MiB model-v1.1.pth
```

**删除对象：**

```bash
# 删除单个对象
aws s3 rm s3://my-ml-training-data-123456789/models/old-model.pth

# 删除前缀下的所有对象
aws s3 rm s3://my-ml-training-data-123456789/experiments/failed/ --recursive

# 删除存储桶（必须先清空）
aws s3 rb s3://my-ml-training-data-123456789 --force
# --force 先删除所有对象，再删除存储桶
```

### 3.3 S3 存储类与生命周期策略

**存储类：**

| 类 | 用例 | 检索 | 成本（$/GB/月） |
|-------|----------|-----------|-------------------|
| **S3 Standard** | 频繁访问 | 即时 | $0.023 |
| **S3 Intelligent-Tiering** | 未知访问模式 | 即时 | $0.023-$0.0125（自动） |
| **S3 Standard-IA** | 不频繁访问 | 即时 | $0.0125 |
| **S3 One Zone-IA** | 不频繁、非关键 | 即时 | $0.01 |
| **S3 Glacier Instant Retrieval** | 归档、即时访问 | 毫秒 | $0.004 |
| **S3 Glacier Flexible Retrieval** | 归档 | 分钟-小时 | $0.0036 |
| **S3 Glacier Deep Archive** | 长期归档 | 小时 | $0.00099 |

**生命周期策略示例：**

```bash
# 创建生命周期策略 JSON
cat > lifecycle-policy.json <<'EOF'
{
  "Rules": [
    {
      "Id": "MoveOldExperimentsToGlacier",
      "Status": "Enabled",
      "Prefix": "experiments/",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 365
      }
    },
    {
      "Id": "DeleteTempFiles",
      "Status": "Enabled",
      "Prefix": "temp/",
      "Expiration": {
        "Days": 7
      }
    }
  ]
}
EOF

# 将生命周期策略应用到存储桶
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-ml-training-data-123456789 \
  --lifecycle-configuration file://lifecycle-policy.json
```

**成本节省示例：**
- **场景**：1TB 实验数据，仅在前 30 天访问
- **S3 Standard（始终）**：每月 $23 × 12 个月 = 每年 $276
- **使用生命周期**：
  - 第 1 个月：Standard（$23）
  - 第 2-3 个月：Standard-IA（$12.50 × 2 = $25）
  - 第 4-12 个月：Glacier（$3.60 × 9 = $32.40）
  - **总计**：每年 $80.40（节省 71%）

### 3.4 S3 版本控制与备份

**启用版本控制：**

```bash
# 在存储桶上启用版本控制
aws s3api put-bucket-versioning \
  --bucket my-ml-training-data-123456789 \
  --versioning-configuration Status=Enabled

# 现在每次上传都创建一个新版本（旧版本保留）
```

**访问先前版本：**

```bash
# 列出对象的所有版本
aws s3api list-object-versions \
  --bucket my-ml-training-data-123456789 \
  --prefix models/model.pth

# 下载特定版本
aws s3api get-object \
  --bucket my-ml-training-data-123456789 \
  --key models/model.pth \
  --version-id "3/L4kqtJlcpXroDTDmpUMLUo" \
  ./model-old-version.pth
```

**为什么 ML 需要版本控制：**
- **模型回滚**：新版本表现不佳时回退到先前版本
- **实验可重现性**：访问过去实验中使用的确切数据/代码
- **意外删除保护**：恢复已删除的文件

### 3.5 S3 安全与访问控制

**存储桶策略（基于资源）：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-public-models-bucket/*"
    }
  ]
}
```

应用策略：
```bash
aws s3api put-bucket-policy \
  --bucket my-public-models-bucket \
  --policy file://bucket-policy.json
```

**IAM 策略（基于用户）：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-ml-training-data-123456789",
        "arn:aws:s3:::my-ml-training-data-123456789/*"
      ]
    }
  ]
}
```

**加密：**

```bash
# 启用默认加密（AES-256）
aws s3api put-bucket-encryption \
  --bucket my-ml-training-data-123456789 \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# 所有新对象自动加密
```

**访问日志：**

```bash
# 启用 S3 访问日志
aws s3api put-bucket-logging \
  --bucket my-ml-training-data-123456789 \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "my-logs-bucket",
      "TargetPrefix": "s3-access-logs/"
    }
  }'
```

---

## 4. RDS（Relational Database Service）

### 4.1 RDS 概述

**RDS = 托管关系数据库**

**支持的引擎：**
- **MySQL**
- **PostgreSQL**（推荐用于 ML：支持 JSON、向量扩展）
- **MariaDB**
- **Oracle**
- **SQL Server**
- **Aurora**（AWS 专有，兼容 MySQL/PostgreSQL，快 5 倍）

**为什么 ML 需要 RDS：**
- **元数据存储**：实验跟踪（MLflow 后端）
- **特征存储**：为实时推理提供特征
- **应用后端**：用户数据、预测历史
- **分析**：查询结果、指标

**RDS vs 自行管理数据库：**
- **RDS 处理**：备份、打补丁、高可用性、扩展
- **你处理**：模式设计、查询、应用集成

### 4.2 启动 RDS 实例（CLI）

**创建 PostgreSQL 数据库：**

```bash
# 创建 RDS 实例（db.t3.micro，符合免费套餐条件）
aws rds create-db-instance \
  --db-instance-identifier ml-metadata-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.3 \
  --master-username admin \
  --master-user-password 'YourSecurePassword123!' \
  --allocated-storage 20 \
  --storage-type gp3 \
  --backup-retention-period 7 \
  --publicly-accessible \
  --tags Key=Name,Value=ml-metadata-db Key=Environment,Value=dev

# 等待 "available" 状态（需要 5-10 分钟）
aws rds wait db-instance-available --db-instance-identifier ml-metadata-db

# 获取连接端点
aws rds describe-db-instances \
  --db-instance-identifier ml-metadata-db \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text
# 输出：ml-metadata-db.c9akznl4qo5q.us-east-1.rds.amazonaws.com
```

**连接到 RDS：**

```bash
# 安装 PostgreSQL 客户端
sudo apt install -y postgresql-client

# 连接
psql -h ml-metadata-db.c9akznl4qo5q.us-east-1.rds.amazonaws.com \
     -U admin \
     -d postgres

# 在 psql 内部：
postgres=> CREATE DATABASE mlflow;
postgres=> \c mlflow
mlflow=> CREATE TABLE experiments (
  experiment_id SERIAL PRIMARY KEY,
  name VARCHAR(255),
  artifact_location VARCHAR(500)
);
```

### 4.3 RDS 高可用性（Multi-AZ）

**Multi-AZ 部署：**

```bash
# 创建 Multi-AZ RDS（自动故障转移）
aws rds create-db-instance \
  --db-instance-identifier ml-prod-db \
  --db-instance-class db.t3.small \
  --engine postgres \
  --engine-version 15.3 \
  --master-username admin \
  --master-user-password 'YourSecurePassword123!' \
  --allocated-storage 100 \
  --multi-az \
  --backup-retention-period 30

# AWS 自动：
# - 在不同 AZ 中维护备用副本
# - 同步复制
# - 主库故障时 60-120 秒自动故障转移
```

**只读副本（用于读扩展）：**

```bash
# 创建只读副本
aws rds create-db-instance-read-replica \
  --db-instance-identifier ml-prod-db-replica \
  --source-db-instance-identifier ml-prod-db

# 使用只读副本进行分析查询（卸载主库）
```

### 4.4 RDS 快照与备份

**自动备份：**
- 默认启用，保留 1-35 天
- 时间点恢复（恢复到保留期内的任意秒）

**手动快照：**

```bash
# 创建手动快照
aws rds create-db-snapshot \
  --db-instance-identifier ml-metadata-db \
  --db-snapshot-identifier ml-metadata-backup-2025-10-23

# 列出快照
aws rds describe-db-snapshots \
  --db-instance-identifier ml-metadata-db

# 从快照恢复（创建新实例）
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier ml-metadata-db-restored \
  --db-snapshot-identifier ml-metadata-backup-2025-10-23
```

### 4.5 RDS 成本优化

**定价：**
- **实例**：按小时计费（db.t3.micro：$0.017/小时 = 每月 $12）
- **存储**：$0.115/GB/月（gp3）
- **备份**：前 100GB 免费，之后 $0.095/GB/月
- **数据传输**：入站免费，出站 $0.09/GB

**成本优化技巧：**
- **停止开发/测试中的实例**（存储费用继续）
- **为生产使用预留实例**（最高节省 69%）
- **删除旧快照**（仅保留必要的）
- **为变化工作负载使用 Aurora Serverless**（按秒付费）

```bash
# 停止 RDS 实例（开发环境）
aws rds stop-db-instance --db-instance-identifier ml-metadata-db

# 需要时启动
aws rds start-db-instance --db-instance-identifier ml-metadata-db
```

---

## 5. IAM（Identity and Access Management）

### 5.1 IAM 基础

**IAM = 谁能在 AWS 中做什么**

**核心概念：**
- **用户**：个人或应用
- **组（Group）**：用户的集合（如"开发人员"、"数据科学家"）
- **角色（Role）**：用于服务/应用的临时凭证
- **策略（Policy）**：定义权限的 JSON 文档

**最佳实践：**
- **永远不要**将根账户用于日常任务
- **启用 MFA**（多因素认证）
- **最小权限原则**：授予最低必要权限
- **为 EC2/Lambda 使用角色**，而非硬编码的访问密钥

### 5.2 IAM 用户和组

**创建 IAM 用户：**

```bash
# 创建用户
aws iam create-user --user-name alice

# 将用户添加到组
aws iam add-user-to-group --user-name alice --group-name DataScientists

# 创建访问密钥（用于 CLI/SDK）
aws iam create-access-key --user-name alice
{
  "AccessKey": {
    "AccessKeyId": "AKIAIOSFODNN7EXAMPLE",
    "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "UserName": "alice"
  }
}

# Alice 配置她的 CLI
aws configure
# 使用上述访问密钥和秘密密钥
```

**创建带策略的 IAM 组：**

```bash
# 创建组
aws iam create-group --group-name DataScientists

# 附加 AWS 托管策略
aws iam attach-group-policy \
  --group-name DataScientists \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 附加自定义策略（见下文）
aws iam put-group-policy \
  --group-name DataScientists \
  --policy-name SageMakerAccess \
  --policy-document file://sagemaker-policy.json
```

### 5.3 IAM 策略

**策略结构：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",  // 或 "Deny"
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::my-ml-training-data-123456789/*"
      ],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

**示例策略：数据科学家访问：**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3DataAccess",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::ml-training-data",
        "arn:aws:s3:::ml-training-data/*"
      ]
    },
    {
      "Sid": "SageMakerFullAccess",
      "Effect": "Allow",
      "Action": "sagemaker:*",
      "Resource": "*"
    },
    {
      "Sid": "EC2ReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyExpensiveInstances",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:InstanceType": [
            "t2.micro",
            "t3.micro",
            "t3.small"
          ]
        }
      }
    }
  ]
}
```

**将策略附加到用户：**

```bash
# 将策略保存到文件
cat > data-scientist-policy.json <<'EOF'
{...策略 JSON...}
EOF

# 创建自定义策略
aws iam create-policy \
  --policy-name DataScientistPolicy \
  --policy-document file://data-scientist-policy.json

# 附加到用户
aws iam attach-user-policy \
  --user-name alice \
  --policy-arn arn:aws:iam::123456789012:policy/DataScientistPolicy
```

### 5.4 IAM 角色（用于服务）

**为什么需要角色？**
- EC2 实例需要访问 S3（不要硬编码访问密钥！）
- Lambda 函数需要写入 DynamoDB
- ECS 任务需要从 ECR 拉取镜像

**为 EC2 创建角色：**

```bash
# 创建信任策略（谁可以承担此角色）
cat > ec2-trust-policy.json <<'EOF'
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
  --role-name EC2-S3-Access-Role \
  --assume-role-policy-document file://ec2-trust-policy.json

# 将 S3 访问策略附加到角色
aws iam attach-role-policy \
  --role-name EC2-S3-Access-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

# 创建实例配置文件（EC2 特定包装器用于角色）
aws iam create-instance-profile --instance-profile-name EC2-S3-Profile

# 将角色添加到实例配置文件
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-Profile \
  --role-name EC2-S3-Access-Role

# 使用角色启动 EC2
aws ec2 run-instances \
  --image-id ami-0c55b159cbfafe1f0 \
  --instance-type t2.micro \
  --iam-instance-profile Name=EC2-S3-Profile
```

**现在 EC2 实例无需访问密钥即可访问 S3：**

```bash
# SSH 到实例
ssh -i keypair.pem ubuntu@<PUBLIC_IP>

# AWS CLI 自动使用实例角色
aws s3 ls s3://my-ml-training-data-123456789/
# 可以工作！实例上未配置凭证
```

### 5.5 MFA（多因素认证）

**为 IAM 用户启用 MFA：**

```bash
# 启用虚拟 MFA 设备（Google Authenticator、Authy）
aws iam create-virtual-mfa-device \
  --virtual-mfa-device-name alice-mfa \
  --outfile alice-mfa-qr.png \
  --bootstrap-method QRCodePNG

# 使用认证器应用扫描二维码，获取两个代码

# 激活 MFA
aws iam enable-mfa-device \
  --user-name alice \
  --serial-number arn:aws:iam::123456789012:mfa/alice-mfa \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

**MFA 保护的操作：**

```json
{
  "Effect": "Deny",
  "Action": "ec2:TerminateInstances",
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "aws:MultiFactorAuthPresent": "false"
    }
  }
}
```

---

## 6. 综合应用：ML 推理架构

**场景：** 部署训练好的 PyTorch 模型用于实时推理。

**架构：**

```
用户请求 → [ALB] → [EC2 Auto-Scaling Group]
                         ├─ EC2 实例 1（附带 IAM 角色）
                         ├─ EC2 实例 2
                         └─ EC2 实例 3
                               ↓
                         [S3：模型产物]
                               ↓
                         [RDS：请求日志、预测结果]
```

**实施步骤：**

**1. 将模型存储在 S3 中：**
```bash
aws s3 cp model.pth s3://my-ml-models/pytorch-classifier/v1.0/model.pth
```

**2. 为 EC2 创建 IAM 角色（S3 + RDS 访问）：**
```bash
# 创建具有 S3 读 + RDS 写权限的角色（参见第 5.4 节）
```

**3. 使用 User Data 启动 EC2 实例：**
```bash
# user-data.sh
#!/bin/bash
apt-get update -y
apt-get install -y python3-pip postgresql-client
pip3 install torch flask boto3 psycopg2-binary

# 从 S3 下载模型
aws s3 cp s3://my-ml-models/pytorch-classifier/v1.0/model.pth /opt/model.pth

# 启动推理服务器
cat > /opt/server.py <<EOF
import torch
from flask import Flask, request, jsonify
import psycopg2

app = Flask(__name__)
model = torch.load('/opt/model.pth')
model.eval()

@app.route('/predict', methods=['POST'])
def predict():
    data = request.json['features']
    input_tensor = torch.tensor(data)
    with torch.no_grad():
        prediction = model(input_tensor).item()

    # 记录到 RDS
    conn = psycopg2.connect(
        host='ml-metadata-db.abc123.us-east-1.rds.amazonaws.com',
        database='predictions',
        user='admin',
        password='SecurePass123!'
    )
    cur = conn.cursor()
    cur.execute("INSERT INTO predictions (input, output) VALUES (%s, %s)", (str(data), prediction))
    conn.commit()
    conn.close()

    return jsonify({'prediction': prediction})

app.run(host='0.0.0.0', port=5000)
EOF

python3 /opt/server.py
```

**4. 创建负载均衡器（此处省略 CLI，使用控制台或 Terraform）：**
- 指向 Auto-Scaling Group
- 健康检查：`/health` 端点
- 在实例间分发流量

**5. 测试：**
```bash
curl -X POST http://my-alb-123456.us-east-1.elb.amazonaws.com/predict \
  -H "Content-Type: application/json" \
  -d '{"features": [0.5, 0.3, 0.8]}'

# 响应：{"prediction": 0.92}
```

---

## 7. 关键要点

1. **EC2** 提供灵活的计算；根据工作负载选择实例类型（训练用 GPU、轻量推理用 CPU）。

2. **S3** 是 ML 数据存储的支柱；使用生命周期策略优化成本（将旧实验移到 Glacier）。

3. **RDS** 替你管理数据库；用于元数据、特征服务和应用后端。

4. **IAM** 对安全至关重要；为 EC2/Lambda 使用角色、执行最小权限、启用 MFA。

5. **AWS CLI** 可以自动化所有操作；学习它来编写基础设施配置脚本并减少手动工作。

6. **成本管理**：不使用时停止实例、为训练使用 Spot 实例、利用免费套餐、设置账单告警。

---

## 下一步是什么？

**Lecture 03** 介绍**网络与安全**——VPC、子网、安全组、NACL 和高级 IAM 策略。你将学习为生产 ML 系统设计安全、隔离的网络架构。

**练习 02** 提供配置 EC2 实例、挂载 EBS 卷、将数据集上传到 S3 以及部署 RDS 数据库的动手实践——全部通过 Terraform 和 AWS CLI 完成。

---

## 延伸阅读

- **AWS EC2 User Guide**：https://docs.aws.amazon.com/ec2/
- **AWS S3 User Guide**：https://docs.aws.amazon.com/s3/
- **AWS RDS User Guide**：https://docs.aws.amazon.com/rds/
- **AWS IAM Best Practices**：https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- **AWS CLI Reference**：https://awscli.amazonaws.com/v2/documentation/api/latest/index.html

---

**End of Lecture 02**
