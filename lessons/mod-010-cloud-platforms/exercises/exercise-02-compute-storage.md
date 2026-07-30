# Exercise 02：计算与存储基础

**模块**：云平台
**难度**：入门
**预计时间**：3-4 小时
**前置要求**：练习 01（AWS 账户与 IAM）、讲座 01-02

---

## 学习目标

完成本练习后，你将：
1. 为 ML 工作负载启动和管理 EC2 实例
2. 配置 EC2 实例元数据和 user data 以实现自动化
3. 使用 EBS 卷进行持久存储
4. 创建和管理带生命周期策略的 S3 存储桶
5. 实施 S3 版本控制和跨区域复制
6. 探索 EC2 Spot 实例以实现成本优化
7. 使用 CloudWatch 监控资源利用率

---

## 概述

这个动手练习教你配置和管理 AWS 上的计算（EC2）和存储（S3、EBS）资源。你将部署一个简单的 ML 推理服务器，将模型存储在 S3 中，并实施成本优化策略。

**实际场景**：你的团队需要部署一个 PyTorch 图像分类模型。你将配置 EC2 实例，将模型存储在 S3 中，并为日志和预测设置持久存储。

---

## 第一部分：EC2 实例基础

### 任务 1.1：启动你的第一个 EC2 实例

**架构**：
```
┌──────────────────────┐
│   Internet Gateway   │
└──────────┬───────────┘
           │
┌──────────▼───────────┐
│   公有子网            │
│   10.0.1.0/24         │
│  ┌────────────────┐  │
│  │  EC2 实例      │  │
│  │  t2.micro      │  │
│  │  公有 IP       │  │
│  └────────────────┘  │
└──────────────────────┘
```

**查找最新的 Ubuntu AMI**：
```bash
# 搜索 Ubuntu 22.04 LTS AMI
aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].{ID:ImageId,Name:Name,Date:CreationDate}' \
  --output table

# 保存 AMI ID
AMI_ID="ami-0c55b159cbfafe1f0"  # 替换为上面的输出
```

**创建安全组**：
```bash
# 获取默认 VPC ID
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text)

# 创建安全组
aws ec2 create-security-group \
  --group-name ml-inference-sg \
  --description "Security group for ML inference server" \
  --vpc-id $VPC_ID

# 获取安全组 ID
SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=ml-inference-sg" --query 'SecurityGroups[0].GroupId' --output text)

# 允许从你的 IP 进行 SSH
MY_IP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr ${MY_IP}/32

# 允许来自任何地方的 HTTP
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

# 允许来自任何地方的 ML 推理 API（端口 5000）
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 5000 \
  --cidr 0.0.0.0/0
```

**创建 SSH 密钥对**：
```bash
# 创建密钥对
aws ec2 create-key-pair \
  --key-name ml-inference-key \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/ml-inference-key.pem

# 设置正确权限
chmod 400 ~/.ssh/ml-inference-key.pem

# 验证密钥已创建
aws ec2 describe-key-pairs --key-names ml-inference-key
```

**启动 EC2 实例**：
```bash
# 启动实例
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name ml-inference-key \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=ml-inference-ec2-profile \
  --tag-specifications 'ResourceType=instance,Tags=[
    {Key=Name,Value=ml-inference-server},
    {Key=Project,Value=ml-infrastructure},
    {Key=Environment,Value=dev},
    {Key=Owner,Value=alice}
  ]' \
  --block-device-mappings '[
    {
      "DeviceName": "/dev/sda1",
      "Ebs": {
        "VolumeSize": 20,
        "VolumeType": "gp3",
        "DeleteOnTermination": true
      }
    }
  ]'

# 从输出中获取实例 ID
INSTANCE_ID="i-0abc123def456789"

# 等待实例运行
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# 获取公有 IP 地址
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "实例正在运行，IP 地址：$PUBLIC_IP"
```

**连接到实例**：
```bash
# SSH 到实例
ssh -i ~/.ssh/ml-inference-key.pem ubuntu@$PUBLIC_IP

# 连接后，验证实例元数据
curl http://169.254.169.254/latest/meta-data/instance-id
curl http://169.254.169.254/latest/meta-data/instance-type
curl http://169.254.169.254/latest/meta-data/placement/availability-zone

# 检查 IAM 角色是否已附加
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/

# 退出 SSH
exit
```

### 任务 1.2：使用 User Data 安装软件

**User Data 脚本**（在实例启动时运行）：

```bash
# 创建 user data 脚本
cat > user-data.sh <<'EOF'
#!/bin/bash

# 更新系统
apt-get update -y

# 安装 Python 和 ML 库
apt-get install -y python3-pip python3-venv

# 创建虚拟环境
python3 -m venv /home/ubuntu/ml-env
source /home/ubuntu/ml-env/bin/activate

# 安装 ML 包
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu
pip install flask pillow requests boto3

# 创建推理服务器
mkdir -p /home/ubuntu/ml-app

cat > /home/ubuntu/ml-app/server.py <<PYEOF
from flask import Flask, request, jsonify
import torch
import torchvision.transforms as transforms
from PIL import Image
import boto3
import io
import os

app = Flask(__name__)

# 从 S3 加载模型（目前为占位）
s3 = boto3.client('s3')
MODEL_BUCKET = os.environ.get('MODEL_BUCKET', 'ml-models-dev')
model = None

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'healthy', 'model_loaded': model is not None})

@app.route('/predict', methods=['POST'])
def predict():
    if 'image' not in request.files:
        return jsonify({'error': 'No image provided'}), 400

    # 占位预测
    return jsonify({
        'prediction': 'cat',
        'confidence': 0.95,
        'model_version': '1.0'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
PYEOF

# 创建 systemd 服务
cat > /etc/systemd/system/ml-inference.service <<SVCEOF
[Unit]
Description=ML Inference Server
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/ml-app
Environment="PATH=/home/ubuntu/ml-env/bin"
ExecStart=/home/ubuntu/ml-env/bin/python3 /home/ubuntu/ml-app/server.py
Restart=always

[Install]
WantedBy=multi-user.target
SVCEOF

# 设置所有权
chown -R ubuntu:ubuntu /home/ubuntu/ml-app
chown -R ubuntu:ubuntu /home/ubuntu/ml-env

# 启用并启动服务
systemctl daemon-reload
systemctl enable ml-inference
systemctl start ml-inference

# 记录完成
echo "ML inference server setup complete" > /var/log/user-data-complete.log
EOF
```

**使用 User Data 启动实例**：
```bash
# 使用 user data 启动新实例
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --key-name ml-inference-key \
  --security-group-ids $SG_ID \
  --iam-instance-profile Name=ml-inference-ec2-profile \
  --user-data file://user-data.sh \
  --tag-specifications 'ResourceType=instance,Tags=[
    {Key=Name,Value=ml-inference-server-v2},
    {Key=Project,Value=ml-infrastructure},
    {Key=Environment,Value=dev}
  ]'

# 获取新实例 ID
INSTANCE_ID_V2="i-0def456abc789012"

# 等待实例运行
aws ec2 wait instance-running --instance-ids $INSTANCE_ID_V2

# 获取公有 IP
PUBLIC_IP_V2=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID_V2 \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

# 等待 user data 脚本完成（可能需要 3-5 分钟）
sleep 180

# 测试健康检查端点
curl http://$PUBLIC_IP_V2:5000/health

# 预期输出：
# {"status": "healthy", "model_loaded": false}
```

### 任务 1.3：EC2 实例生命周期管理

**停止实例**（保留数据，停止计算计费）：
```bash
aws ec2 stop-instances --instance-ids $INSTANCE_ID_V2
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID_V2

# 验证实例状态
aws ec2 describe-instances --instance-ids $INSTANCE_ID_V2 \
  --query 'Reservations[0].Instances[0].State.Name'
# 输出："stopped"
```

**启动实例**：
```bash
aws ec2 start-instances --instance-ids $INSTANCE_ID_V2
aws ec2 wait instance-running --instance-ids $INSTANCE_ID_V2

# 注意：停止/启动后公有 IP 改变（除非使用弹性 IP）
PUBLIC_IP_NEW=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID_V2 \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "新公有 IP：$PUBLIC_IP_NEW"
```

**分配弹性 IP**（静态公有 IP）：
```bash
# 分配弹性 IP
EIP_ALLOC_ID=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
EIP_ADDRESS=$(aws ec2 describe-addresses --allocation-ids $EIP_ALLOC_ID --query 'Addresses[0].PublicIp' --output text)

# 与实例关联
aws ec2 associate-address \
  --instance-id $INSTANCE_ID_V2 \
  --allocation-id $EIP_ALLOC_ID

echo "弹性 IP $EIP_ADDRESS 已关联到实例"

# 现在停止/启动后 IP 不会改变
```

**重启实例**：
```bash
aws ec2 reboot-instances --instance-ids $INSTANCE_ID_V2
```

**终止实例**（删除实例和关联资源）：
```bash
# 终止第一个测试实例（我们将保留 v2）
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
```

---

## 第二部分：EBS（Elastic Block Store）卷

### 任务 2.1：创建并挂载 EBS 卷

**为什么需要 EBS？** 持久块存储，在实例终止后仍然存在。用于数据库、日志、检查点。

**创建 EBS 卷**：
```bash
# 获取实例可用区
AZ=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID_V2 \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text)

# 为 ML 训练数据创建 50GB gp3 卷
VOLUME_ID=$(aws ec2 create-volume \
  --availability-zone $AZ \
  --size 50 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --tag-specifications 'ResourceType=volume,Tags=[
    {Key=Name,Value=ml-training-data},
    {Key=Project,Value=ml-infrastructure},
    {Key=Environment,Value=dev}
  ]' \
  --query 'VolumeId' \
  --output text)

echo "已创建卷：$VOLUME_ID"

# 等待卷可用
aws ec2 wait volume-available --volume-ids $VOLUME_ID
```

**将卷挂载到实例**：
```bash
# 挂载卷
aws ec2 attach-volume \
  --volume-id $VOLUME_ID \
  --instance-id $INSTANCE_ID_V2 \
  --device /dev/sdf

# 等待挂载
aws ec2 wait volume-in-use --volume-ids $VOLUME_ID

# 验证挂载
aws ec2 describe-volumes --volume-ids $VOLUME_ID \
  --query 'Volumes[0].Attachments[0].State'
# 输出："attached"
```

**格式化和挂载卷**（在 EC2 实例上）：
```bash
# SSH 到实例
ssh -i ~/.ssh/ml-inference-key.pem ubuntu@$EIP_ADDRESS

# 检查卷是否已附加
lsblk
# 输出应显示 /dev/nvme1n1 或 /dev/xvdf（未分区）

# 创建文件系统
sudo mkfs -t ext4 /dev/nvme1n1

# 创建挂载点
sudo mkdir -p /mnt/ml-data

# 挂载卷
sudo mount /dev/nvme1n1 /mnt/ml-data

# 验证挂载
df -h | grep ml-data

# 设置所有权
sudo chown ubuntu:ubuntu /mnt/ml-data

# 创建测试文件
echo "ML training data storage" > /mnt/ml-data/README.txt

# 使挂载持久（添加到 /etc/fstab）
DEVICE_UUID=$(sudo blkid /dev/nvme1n1 -s UUID -o value)
echo "UUID=$DEVICE_UUID  /mnt/ml-data  ext4  defaults,nofail  0  2" | sudo tee -a /etc/fstab

# 退出 SSH
exit
```

### 任务 2.2：EBS 快照（备份）

**创建快照**：
```bash
# 创建快照
SNAPSHOT_ID=$(aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "Backup of ml-training-data volume" \
  --tag-specifications 'ResourceType=snapshot,Tags=[
    {Key=Name,Value=ml-training-data-backup-2025-10-23},
    {Key=Project,Value=ml-infrastructure}
  ]' \
  --query 'SnapshotId' \
  --output text)

echo "已创建快照：$SNAPSHOT_ID"

# 等待快照完成（可能需要几分钟）
aws ec2 wait snapshot-completed --snapshot-ids $SNAPSHOT_ID

# 验证快照
aws ec2 describe-snapshots --snapshot-ids $SNAPSHOT_ID \
  --query 'Snapshots[0].{ID:SnapshotId,Progress:Progress,State:State}'
```

**从快照恢复**：
```bash
# 从快照创建新卷（在同一或不同可用区）
RESTORED_VOLUME_ID=$(aws ec2 create-volume \
  --snapshot-id $SNAPSHOT_ID \
  --availability-zone $AZ \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[
    {Key=Name,Value=ml-training-data-restored}
  ]' \
  --query 'VolumeId' \
  --output text)

echo "已恢复卷：$RESTORED_VOLUME_ID"
```

**自动快照生命周期**：
```bash
# 为自动快照创建生命周期策略
cat > lifecycle-policy.json <<EOF
{
  "ExecutionRoleArn": "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/AWSDataLifecycleManagerDefaultRole",
  "Description": "Daily snapshots of ML volumes",
  "State": "ENABLED",
  "PolicyDetails": {
    "PolicyType": "EBS_SNAPSHOT_MANAGEMENT",
    "ResourceTypes": ["VOLUME"],
    "TargetTags": [
      {
        "Key": "Project",
        "Value": "ml-infrastructure"
      }
    ],
    "Schedules": [
      {
        "Name": "Daily snapshots",
        "CopyTags": true,
        "CreateRule": {
          "Interval": 24,
          "IntervalUnit": "HOURS",
          "Times": ["03:00"]
        },
        "RetainRule": {
          "Count": 7
        }
      }
    ]
  }
}
EOF

# 先创建 DLM 角色
aws iam create-role \
  --role-name AWSDataLifecycleManagerDefaultRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "dlm.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name AWSDataLifecycleManagerDefaultRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSDataLifecycleManagerServiceRole

# 等待角色传播
sleep 10

# 创建生命周期策略
aws dlm create-lifecycle-policy --cli-input-json file://lifecycle-policy.json
```

---

## 第三部分：S3（Simple Storage Service）

### 任务 3.1：为 ML 模型创建 S3 存储桶

**创建存储桶**：
```bash
# S3 存储桶名称必须全局唯一
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="ml-models-dev-${ACCOUNT_ID}"

# 创建存储桶
aws s3 mb s3://$BUCKET_NAME --region us-east-1

# 验证存储桶创建
aws s3 ls | grep ml-models
```

**配置存储桶加密**：
```bash
# 启用默认加密（AES-256）
aws s3api put-bucket-encryption \
  --bucket $BUCKET_NAME \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      },
      "BucketKeyEnabled": true
    }]
  }'

# 验证加密
aws s3api get-bucket-encryption --bucket $BUCKET_NAME
```

**启用版本控制**：
```bash
# 启用版本控制（防止意外删除）
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

# 验证版本控制
aws s3api get-bucket-versioning --bucket $BUCKET_NAME
# 输出：{"Status": "Enabled"}
```

**阻止公共访问**：
```bash
# 阻止所有公共访问（ML 模型的最佳实践）
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# 验证
aws s3api get-public-access-block --bucket $BUCKET_NAME
```

### 任务 3.2：上传 ML 模型和数据

**创建示例模型文件**：
```bash
# 创建虚拟 PyTorch 模型文件
cat > model.py <<'EOF'
import torch
import torch.nn as nn

# 用于图像分类的简单 CNN
class SimpleCNN(nn.Module):
    def __init__(self):
        super(SimpleCNN, self).__init__()
        self.conv1 = nn.Conv2d(3, 32, 3, padding=1)
        self.conv2 = nn.Conv2d(32, 64, 3, padding=1)
        self.fc1 = nn.Linear(64 * 56 * 56, 128)
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x):
        x = torch.relu(self.conv1(x))
        x = torch.max_pool2d(x, 2)
        x = torch.relu(self.conv2(x))
        x = torch.max_pool2d(x, 2)
        x = x.view(x.size(0), -1)
        x = torch.relu(self.fc1(x))
        x = self.fc2(x)
        return x

# 保存模型
model = SimpleCNN()
torch.save(model.state_dict(), 'resnet50.pth')
print("Model saved to resnet50.pth")
EOF

# 运行脚本（需要 PyTorch）
# python3 model.py

# 为本练习创建一个虚拟文件
echo "Placeholder PyTorch model" > resnet50.pth
```

**上传到 S3**：
```bash
# 上传模型并附带元数据
aws s3 cp resnet50.pth s3://$BUCKET_NAME/models/v1.0/resnet50.pth \
  --metadata model_version=1.0,framework=pytorch,accuracy=0.95 \
  --storage-class STANDARD

# 使用服务器端加密上传（KMS）
# aws s3 cp resnet50.pth s3://$BUCKET_NAME/models/v1.0/resnet50.pth \
#   --server-side-encryption aws:kms \
#   --ssekms-key-id <kms-key-id>

# 验证上传
aws s3 ls s3://$BUCKET_NAME/models/v1.0/

# 获取对象元数据
aws s3api head-object --bucket $BUCKET_NAME --key models/v1.0/resnet50.pth
```

**将目录同步到 S3**：
```bash
# 创建训练数据目录
mkdir -p training-data/images
echo "image1.jpg" > training-data/images/cat1.jpg
echo "image2.jpg" > training-data/images/dog1.jpg

# 同步整个目录
aws s3 sync training-data/ s3://$BUCKET_NAME/training-data/ --delete

# 验证同步
aws s3 ls s3://$BUCKET_NAME/training-data/ --recursive
```

### 任务 3.3：S3 生命周期策略

**为什么？** 自动将对象转移到更便宜的存储类或删除旧版本。

**创建生命周期策略**：
```bash
cat > s3-lifecycle-policy.json <<EOF
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
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        }
      ],
      "NoncurrentVersionTransitions": [
        {
          "NoncurrentDays": 7,
          "StorageClass": "STANDARD_IA"
        },
        {
          "NoncurrentDays": 30,
          "StorageClass": "GLACIER_IR"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 180
      }
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
    },
    {
      "Id": "cleanup-incomplete-uploads",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
EOF

# 应用生命周期策略
aws s3api put-bucket-lifecycle-configuration \
  --bucket $BUCKET_NAME \
  --lifecycle-configuration file://s3-lifecycle-policy.json

# 验证生命周期策略
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET_NAME
```

**存储类对比**：
| 类 | 成本（$/GB/月） | 检索时间 | 用例 |
|-------|-------------------|----------------|----------|
| **Standard** | $0.023 | 即时 | 活跃模型 |
| **Intelligent-Tiering** | $0.0025-0.023 | 即时 | 未知模式 |
| **Standard-IA** | $0.0125 | 即时 | 每月访问 <1 次的模型 |
| **Glacier Instant Retrieval** | $0.004 | 即时 | 归档模型（季度访问） |
| **Glacier Flexible Retrieval** | $0.0036 | 分钟-小时 | 长期归档 |
| **Deep Archive** | $0.00099 | 12 小时 | 合规归档 |

### 任务 3.4：S3 跨区域复制（CRR）

**为什么？** 灾难恢复、合规（数据本地性）、为全球用户降低延迟。

**创建目标存储桶**（在不同区域）：
```bash
# 在 us-west-2 创建存储桶
BUCKET_REPLICA="ml-models-replica-${ACCOUNT_ID}"
aws s3 mb s3://$BUCKET_REPLICA --region us-west-2

# 启用目标存储桶的版本控制（CRR 需要）
aws s3api put-bucket-versioning \
  --bucket $BUCKET_REPLICA \
  --versioning-configuration Status=Enabled \
  --region us-west-2
```

**创建复制角色**：
```bash
cat > replication-role-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "s3.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name s3-replication-role \
  --assume-role-policy-document file://replication-role-trust-policy.json

# 创建复制策略
cat > replication-role-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl"
      ],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete"
      ],
      "Resource": "arn:aws:s3:::${BUCKET_REPLICA}/*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name s3-replication-role \
  --policy-name s3-replication-policy \
  --policy-document file://replication-role-policy.json
```

**配置复制**：
```bash
cat > replication-config.json <<EOF
{
  "Role": "arn:aws:iam::${ACCOUNT_ID}:role/s3-replication-role",
  "Rules": [
    {
      "Status": "Enabled",
      "Priority": 1,
      "DeleteMarkerReplication": {"Status": "Enabled"},
      "Filter": {"Prefix": "models/"},
      "Destination": {
        "Bucket": "arn:aws:s3:::${BUCKET_REPLICA}",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {"Minutes": 15}
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": {"Minutes": 15}
        }
      }
    }
  ]
}
EOF

# 等待角色传播
sleep 10

# 应用复制配置
aws s3api put-bucket-replication \
  --bucket $BUCKET_NAME \
  --replication-configuration file://replication-config.json

# 验证复制
aws s3api get-bucket-replication --bucket $BUCKET_NAME
```

**测试复制**：
```bash
# 上传一个新文件
echo "test replication" > test-replication.txt
aws s3 cp test-replication.txt s3://$BUCKET_NAME/models/test-replication.txt

# 等待复制（最多 15 分钟）
sleep 60

# 检查文件是否存在于副本存储桶中
aws s3 ls s3://$BUCKET_REPLICA/models/ --region us-west-2
```

---

## 第四部分：用于成本优化的 EC2 Spot 实例

### 任务 4.1：了解 Spot 定价

**查看 Spot 价格历史**：
```bash
# 获取 us-east-1a 中 t2.micro 的当前 Spot 价格
aws ec2 describe-spot-price-history \
  --instance-types t2.micro \
  --product-descriptions "Linux/UNIX" \
  --availability-zone us-east-1a \
  --start-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --query 'SpotPriceHistory[0].{AZ:AvailabilityZone,Price:SpotPrice,Time:Timestamp}' \
  --output table

# 与按需价格比较（t2.micro = $0.0116/小时）
# 典型 Spot 价格：$0.0035/小时（节省 70%）
```

### 任务 4.2：启动 Spot 实例

**创建 Spot 实例请求**：
```bash
# 创建 Spot 请求规范
cat > spot-request.json <<EOF
{
  "ImageId": "$AMI_ID",
  "InstanceType": "t2.micro",
  "KeyName": "ml-inference-key",
  "SecurityGroupIds": ["$SG_ID"],
  "IamInstanceProfile": {"Name": "ml-inference-ec2-profile"},
  "TagSpecifications": [{
    "ResourceType": "instance",
    "Tags": [
      {"Key": "Name", "Value": "ml-training-spot"},
      {"Key": "Project", "Value": "ml-infrastructure"},
      {"Key": "Environment", "Value": "dev"}
    ]
  }]
}
EOF

# 请求 Spot 实例（最高价格 = 按需价格）
aws ec2 request-spot-instances \
  --spot-price "0.0116" \
  --instance-count 1 \
  --type "one-time" \
  --launch-specification file://spot-request.json

# 从输出中获取 Spot 请求 ID
SPOT_REQUEST_ID="sir-abc123"

# 检查 Spot 请求状态
aws ec2 describe-spot-instance-requests \
  --spot-instance-request-ids $SPOT_REQUEST_ID

# 履行后获取实例 ID
SPOT_INSTANCE_ID=$(aws ec2 describe-spot-instance-requests \
  --spot-instance-request-ids $SPOT_REQUEST_ID \
  --query 'SpotInstanceRequests[0].InstanceId' \
  --output text)

echo "Spot 实例：$SPOT_INSTANCE_ID"
```

**处理 Spot 中断**（2 分钟警告）：

在实例上创建中断处理程序：
```bash
# SSH 到 Spot 实例
ssh -i ~/.ssh/ml-inference-key.pem ubuntu@<spot-instance-ip>

# 创建中断监控脚本
cat > /home/ubuntu/spot-monitor.sh <<'EOF'
#!/bin/bash

while true; do
  # 检查 Spot 中断通知
  RESPONSE=$(curl -s http://169.254.169.254/latest/meta-data/spot/instance-action)

  if [ "$RESPONSE" != "404 - Not Found" ]; then
    echo "$(date): Spot interruption detected: $RESPONSE"

    # 保存检查点到 S3
    aws s3 sync /mnt/ml-data/checkpoints/ s3://ml-training-checkpoints/

    # 优雅关闭应用程序
    systemctl stop ml-inference

    exit 0
  fi

  sleep 5
done
EOF

chmod +x /home/ubuntu/spot-monitor.sh

# 作为后台服务运行（在生产中，使用 systemd）
nohup /home/ubuntu/spot-monitor.sh > /var/log/spot-monitor.log 2>&1 &
```

---

## 第五部分：CloudWatch 监控

### 任务 5.1：监控 EC2 指标

**查看内置指标**：
```bash
# 获取过去一小时的 CPU 利用率
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID_V2 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --query 'Datapoints | sort_by(@, &Timestamp)[-5:]' \
  --output table

# 获取网络入站（字节）
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkIn \
  --dimensions Name=InstanceId,Value=$INSTANCE_ID_V2 \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum
```

### 任务 5.2：创建 CloudWatch 仪表盘

**创建仪表盘**：
```bash
cat > dashboard-config.json <<EOF
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/EC2", "CPUUtilization", {"stat": "Average", "label": "CPU %"}]
        ],
        "period": 300,
        "region": "us-east-1",
        "title": "EC2 CPU Utilization"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/EC2", "NetworkIn", {"stat": "Sum"}],
          [".", "NetworkOut", {"stat": "Sum"}]
        ],
        "period": 300,
        "region": "us-east-1",
        "title": "EC2 Network Traffic"
      }
    }
  ]
}
EOF

aws cloudwatch put-dashboard \
  --dashboard-name ml-infrastructure-dashboard \
  --dashboard-body file://dashboard-config.json
```

---

## 第六部分：验证与成本分析

### 任务 6.1：验证所有资源

**检查资源清单**：
```bash
# 列出所有 EC2 实例
aws ec2 describe-instances \
  --filters "Name=tag:Project,Values=ml-infrastructure" \
  --query 'Reservations[].Instances[].[InstanceId,InstanceType,State.Name,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# 列出所有 EBS 卷
aws ec2 describe-volumes \
  --filters "Name=tag:Project,Values=ml-infrastructure" \
  --query 'Volumes[].[VolumeId,Size,VolumeType,State,Attachments[0].InstanceId]' \
  --output table

# 列出所有 S3 存储桶
aws s3 ls | grep ml-

# 获取 S3 存储桶大小
aws s3 ls s3://$BUCKET_NAME --recursive --summarize | tail -2
```

### 任务 6.2：估算每月成本

**计算估算成本**：
```bash
cat > cost-estimate.sh <<'EOF'
#!/bin/bash

echo "=== 每月成本估算 ==="
echo ""

# EC2 成本
echo "EC2（1x t2.micro，100% 运行时间）："
echo "  $0.0116/小时 × 730 小时 = $8.47/月"
echo ""

# EBS 成本
echo "EBS（50GB gp3）："
echo "  $0.08/GB × 50GB = $4.00/月"
echo "  IOPS（3000）已包含"
echo "  吞吐量（125 MB/s）已包含"
echo ""

# S3 成本（假设 10GB 存储，每月 1000 次请求）
echo "S3 Standard（10GB）："
echo "  存储：$0.023/GB × 10GB = $0.23/月"
echo "  PUT 请求：$0.005/1000 × 1 = $0.005/月"
echo "  GET 请求：$0.0004/1000 × 1 = $0.0004/月"
echo "  S3 总计：~$0.24/月"
echo ""

# 数据传输（假设每月 10GB 出站）
echo "数据传输："
echo "  前 1GB 免费，之后 $0.09/GB"
echo "  10GB - 1GB = 9GB × $0.09 = $0.81/月"
echo ""

# 总计
echo "=== 估算总成本 ==="
echo "EC2：          $8.47"
echo "EBS：          $4.00"
echo "S3：           $0.24"
echo "数据传输：     $0.81"
echo "------------------------"
echo "总计：         $13.52/月"
echo ""
echo "使用 Spot 实例（EC2 节省 70%）："
echo "  EC2 Spot：   $2.54"
echo "  其他：       $5.05"
echo "  总计：       $7.59/月（节省 44%）"
EOF

chmod +x cost-estimate.sh
./cost-estimate.sh
```

---

## 清理

**终止资源**（避免费用）：
```bash
# 终止 EC2 实例
aws ec2 terminate-instances --instance-ids $INSTANCE_ID_V2 $SPOT_INSTANCE_ID

# 释放弹性 IP
aws ec2 release-address --allocation-id $EIP_ALLOC_ID

# 删除 EBS 卷（先等待实例终止）
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID_V2
aws ec2 delete-volume --volume-id $VOLUME_ID
aws ec2 delete-volume --volume-id $RESTORED_VOLUME_ID

# 删除 EBS 快照
aws ec2 delete-snapshot --snapshot-id $SNAPSHOT_ID

# 删除 S3 存储桶（必须先清空）
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME

aws s3 rm s3://$BUCKET_REPLICA --recursive --region us-west-2
aws s3 rb s3://$BUCKET_REPLICA --region us-west-2

# 删除安全组
aws ec2 delete-security-group --group-id $SG_ID

# 删除密钥对
aws ec2 delete-key-pair --key-name ml-inference-key
rm ~/.ssh/ml-inference-key.pem
```

---

## 交付物

到本练习结束时，你应该有：

1. ✅ 使用 user data 自动化启动了 EC2 实例
2. ✅ 创建并挂载了 EBS 卷
3. ✅ 实施了 EBS 快照备份
4. ✅ 创建了带加密和版本控制的 S3 存储桶
5. ✅ 配置了 S3 生命周期策略
6. ✅ 设置了跨区域复制
7. ✅ 启动了 Spot 实例以节省成本
8. ✅ 使用 CloudWatch 监控资源
9. ✅ 计算了成本估算

**证据**：
- 运行中 EC2 实例的截图
- 显示模型和训练数据的 S3 存储桶列表
- 显示指标的 CloudWatch 仪表盘
- 成本估算输出

---

## 下一步

现在你可以配置计算和存储，你已经准备好：
- **练习 03**：构建具有多层网络的生产 VPC
- **练习 04**：使用 ECS/EKS 部署容器化 ML 服务
- **练习 05**：SageMaker 训练和成本优化

做得很好！你现在已经拥有了 AWS 核心服务的动手经验。
