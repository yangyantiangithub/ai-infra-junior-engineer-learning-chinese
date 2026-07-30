# Exercise 05：SageMaker ML 平台与成本优化

**模块**：云平台
**难度**：高级
**预计时间**：5-6 小时
**前置要求**：练习 01-04、模块 010 所有讲座

---

## 学习目标

完成本练习后，你将：
1. 使用 Amazon SageMaker 的内置算法训练 ML 模型
2. 部署带自动扩缩容的 SageMaker 端点
3. 实施超参数调优以优化模型
4. 使用 SageMaker Pipelines 进行 ML 工作流编排
5. 实施全面的成本优化策略
6. 创建成本分配标签和预算
7. 使用 AWS Cost Explorer 分析成本分解
8. 在 AWS 上构建完整的端到端 ML 平台

---

## 概述

这个结业练习整合了模块 001-010 的所有内容。你将使用 SageMaker 构建完整的 ML 平台，实施成本控制，并优化整个 AWS 基础设施。

**实际场景**：你的公司希望扩展 ML 模型训练和部署。你将把手动训练迁移到 SageMaker，实施成本治理，并创建生产 ML 平台。

---

## 第一部分：SageMaker 训练作业

### 任务 1.1：准备训练数据

**创建训练数据集**：
```bash
# 创建目录结构
mkdir sagemaker-ml-platform && cd sagemaker-ml-platform

# 创建合成训练数据（MNIST 风格）
cat > prepare_data.py <<'EOF'
import numpy as np
import pandas as pd
from sklearn.datasets import make_classification
import boto3
import os

# 生成合成二分类数据
X, y = make_classification(
    n_samples=10000,
    n_features=20,
    n_informative=15,
    n_redundant=5,
    n_classes=2,
    random_state=42
)

# 分为训练/验证/测试集
train_size = 7000
val_size = 1500
test_size = 1500

X_train, X_val, X_test = X[:train_size], X[train_size:train_size+val_size], X[train_size+val_size:]
y_train, y_val, y_test = y[:train_size], y[train_size:train_size+val_size], y[train_size+val_size:]

# 创建 DataFrame（SageMaker 内置算法要求 CSV 无标题）
train_data = np.column_stack([y_train, X_train])
val_data = np.column_stack([y_val, X_val])
test_data = np.column_stack([y_test, X_test])

# 保存为 CSV
np.savetxt('train.csv', train_data, delimiter=',', fmt='%.6f')
np.savetxt('validation.csv', val_data, delimiter=',', fmt='%.6f')
np.savetxt('test.csv', test_data, delimiter=',', fmt='%.6f')

print("训练数据已准备：")
print(f"  训练集：{len(train_data)} 个样本")
print(f"  验证集：{len(val_data)} 个样本")
print(f"  测试集：{len(test_data)} 个样本")
EOF

# 安装依赖并运行
pip install numpy pandas scikit-learn boto3
python prepare_data.py
```

### 任务 1.2：创建 SageMaker 训练脚本

**自定义训练脚本**（PyTorch）：
```bash
mkdir -p src

cat > src/train.py <<'EOF'
import argparse
import os
import json
import numpy as np
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import Dataset, DataLoader

class CSVDataset(Dataset):
    def __init__(self, file_path):
        data = np.loadtxt(file_path, delimiter=',')
        self.X = torch.FloatTensor(data[:, 1:])
        self.y = torch.FloatTensor(data[:, 0])

    def __len__(self):
        return len(self.X)

    def __getitem__(self, idx):
        return self.X[idx], self.y[idx]

class BinaryClassifier(nn.Module):
    def __init__(self, input_dim, hidden_dim):
        super(BinaryClassifier, self).__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, hidden_dim // 2)
        self.fc3 = nn.Linear(hidden_dim // 2, 1)
        self.relu = nn.ReLU()
        self.sigmoid = nn.Sigmoid()

    def forward(self, x):
        x = self.relu(self.fc1(x))
        x = self.relu(self.fc2(x))
        x = self.sigmoid(self.fc3(x))
        return x

def train(args):
    train_dataset = CSVDataset(os.path.join(args.train, 'train.csv'))
    val_dataset = CSVDataset(os.path.join(args.validation, 'validation.csv'))

    train_loader = DataLoader(train_dataset, batch_size=args.batch_size, shuffle=True)
    val_loader = DataLoader(val_dataset, batch_size=args.batch_size)

    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model = BinaryClassifier(input_dim=20, hidden_dim=args.hidden_dim).to(device)

    criterion = nn.BCELoss()
    optimizer = optim.Adam(model.parameters(), lr=args.learning_rate)

    best_val_loss = float('inf')
    for epoch in range(args.epochs):
        model.train()
        train_loss = 0.0
        for X_batch, y_batch in train_loader:
            X_batch, y_batch = X_batch.to(device), y_batch.to(device)
            optimizer.zero_grad()
            outputs = model(X_batch).squeeze()
            loss = criterion(outputs, y_batch)
            loss.backward()
            optimizer.step()
            train_loss += loss.item()

        model.eval()
        val_loss = 0.0
        correct = 0
        total = 0
        with torch.no_grad():
            for X_batch, y_batch in val_loader:
                X_batch, y_batch = X_batch.to(device), y_batch.to(device)
                outputs = model(X_batch).squeeze()
                loss = criterion(outputs, y_batch)
                val_loss += loss.item()
                predicted = (outputs > 0.5).float()
                total += y_batch.size(0)
                correct += (predicted == y_batch).sum().item()

        accuracy = 100 * correct / total
        if val_loss < best_val_loss:
            best_val_loss = val_loss
            torch.save(model.state_dict(), os.path.join(args.model_dir, 'model.pth'))

    print(f"训练完成！最佳验证损失：{best_val_loss:.4f}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--epochs', type=int, default=10)
    parser.add_argument('--batch-size', type=int, default=64)
    parser.add_argument('--learning-rate', type=float, default=0.001)
    parser.add_argument('--hidden-dim', type=int, default=128)
    parser.add_argument('--model-dir', type=str, default=os.environ.get('SM_MODEL_DIR'))
    parser.add_argument('--train', type=str, default=os.environ.get('SM_CHANNEL_TRAINING'))
    parser.add_argument('--validation', type=str, default=os.environ.get('SM_CHANNEL_VALIDATION'))
    args = parser.parse_args()
    train(args)
EOF
```

### 任务 1.3：运行 SageMaker 训练作业

**启动训练作业**：
```python
cat > train_sagemaker.py <<'EOF'
import sagemaker
from sagemaker.pytorch import PyTorch
from sagemaker import get_execution_role

sagemaker_session = sagemaker.Session()
role = get_execution_role()
bucket = sagemaker_session.default_bucket()

estimator = PyTorch(
    entry_point='train.py',
    source_dir='src',
    role=role,
    instance_type='ml.m5.xlarge',
    instance_count=1,
    framework_version='2.1.0',
    py_version='py310',
    hyperparameters={
        'epochs': 20,
        'batch-size': 64,
        'learning-rate': 0.001,
        'hidden-dim': 128
    },
    use_spot_instances=True,
    max_run=3600,
    max_wait=7200,
    checkpoint_s3_uri=f's3://{bucket}/sagemaker/checkpoints/',
    tags=[
        {'Key': 'Project', 'Value': 'ml-infrastructure'},
        {'Key': 'Environment', 'Value': 'dev'},
        {'Key': 'CostCenter', 'Value': 'engineering'}
    ]
)

estimator.fit({
    'training': f's3://{bucket}/sagemaker/training-data/train.csv',
    'validation': f's3://{bucket}/sagemaker/training-data/validation.csv'
})

print(f"训练作业完成！")
print(f"模型产物：{estimator.model_data}")
EOF

python train_sagemaker.py
```

---

## 第二部分：超参数调优

### 任务 2.1：创建超参数调优作业

**定义调优范围**：
```python
cat > tune_hyperparameters.py <<'EOF'
import sagemaker
from sagemaker.pytorch import PyTorch
from sagemaker.tuner import HyperparameterTuner, IntegerParameter, ContinuousParameter
from sagemaker import get_execution_role

sagemaker_session = sagemaker.Session()
role = get_execution_role()
bucket = sagemaker_session.default_bucket()

estimator = PyTorch(
    entry_point='train.py',
    source_dir='src',
    role=role,
    instance_type='ml.m5.xlarge',
    instance_count=1,
    framework_version='2.1.0',
    py_version='py310',
    use_spot_instances=True,
    max_run=1800,
    max_wait=3600
)

hyperparameter_ranges = {
    'learning-rate': ContinuousParameter(0.0001, 0.01),
    'hidden-dim': IntegerParameter(64, 256),
    'batch-size': IntegerParameter(32, 128)
}

objective_metric_name = 'validation:loss'
objective_type = 'Minimize'

tuner = HyperparameterTuner(
    estimator,
    objective_metric_name,
    hyperparameter_ranges,
    metric_definitions=[
        {'Name': 'validation:loss', 'Regex': 'Val Loss: ([0-9\\.]+)'}
    ],
    max_jobs=20,
    max_parallel_jobs=2,
    objective_type=objective_type,
    strategy='Bayesian'
)

tuner.fit({
    'training': f's3://{bucket}/sagemaker/training-data/train.csv',
    'validation': f's3://{bucket}/sagemaker/training-data/validation.csv'
})

print(f"最佳训练作业：{tuner.best_training_job()}")
EOF

python tune_hyperparameters.py
```

---

## 第三部分：部署 SageMaker 端点

### 任务 3.1：创建模型和端点

**部署模型**：
```python
cat > deploy_endpoint.py <<'EOF'
import sagemaker
from sagemaker.pytorch import PyTorchModel
from sagemaker import get_execution_role

role = get_execution_role()

pytorch_model = PyTorchModel(
    model_data='s3://sagemaker-ml-platform-.../output/model.tar.gz',
    role=role,
    framework_version='2.1.0',
    py_version='py310',
    entry_point='inference.py',
    source_dir='src'
)

predictor = pytorch_model.deploy(
    instance_type='ml.t2.medium',
    initial_instance_count=2,
    endpoint_name='ml-inference-endpoint'
)

print(f"端点已部署：{predictor.endpoint_name}")
EOF

cat > src/inference.py <<'EOF'
import torch
import torch.nn as nn
import json
import numpy as np

class BinaryClassifier(nn.Module):
    def __init__(self, input_dim=20, hidden_dim=128):
        super(BinaryClassifier, self).__init__()
        self.fc1 = nn.Linear(input_dim, hidden_dim)
        self.fc2 = nn.Linear(hidden_dim, hidden_dim // 2)
        self.fc3 = nn.Linear(hidden_dim // 2, 1)
        self.relu = nn.ReLU()
        self.sigmoid = nn.Sigmoid()

    def forward(self, x):
        x = self.relu(self.fc1(x))
        x = self.relu(self.fc2(x))
        x = self.sigmoid(self.fc3(x))
        return x

def model_fn(model_dir):
    model = BinaryClassifier()
    model.load_state_dict(torch.load(f'{model_dir}/model.pth'))
    model.eval()
    return model

def predict_fn(input_data, model):
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model.to(device)
    with torch.no_grad():
        tensor = torch.FloatTensor(input_data).to(device)
        output = model(tensor)
        return output.cpu().numpy()

def input_fn(request_body, content_type='application/json'):
    if content_type == 'application/json':
        data = json.loads(request_body)
        return np.array(data['features'])
    raise ValueError(f"Unsupported content type: {content_type}")

def output_fn(prediction, accept='application/json'):
    if accept == 'application/json':
        return json.dumps({'predictions': prediction.tolist()})
    raise ValueError(f"Unsupported accept type: {accept}")
EOF

python deploy_endpoint.py
```

### 任务 3.2：配置端点自动扩缩容

```bash
# 注册可扩展目标
aws application-autoscaling register-scalable-target \
  --service-namespace sagemaker \
  --resource-id endpoint/ml-inference-endpoint/variant/AllTraffic \
  --scalable-dimension sagemaker:variant:DesiredInstanceCount \
  --min-capacity 1 \
  --max-capacity 5

# 创建扩展策略
aws application-autoscaling put-scaling-policy \
  --service-namespace sagemaker \
  --resource-id endpoint/ml-inference-endpoint/variant/AllTraffic \
  --scalable-dimension sagemaker:variant:DesiredInstanceCount \
  --policy-name sagemaker-endpoint-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 1000.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "SageMakerVariantInvocationsPerInstance"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

---

## 第四部分：成本优化策略

### 任务 4.1：实施成本分配标签

```bash
# 给 S3 存储桶打标签
aws s3api put-bucket-tagging \
  --bucket $BUCKET_NAME \
  --tagging 'TagSet=[
    {Key=Project,Value=ml-infrastructure},
    {Key=Environment,Value=dev},
    {Key=CostCenter,Value=engineering},
    {Key=Owner,Value=ml-team}
  ]'

# 给 SageMaker 端点打标签
aws sagemaker add-tags \
  --resource-arn arn:aws:sagemaker:us-east-1:$ACCOUNT_ID:endpoint/ml-inference-endpoint \
  --tags Key=Project,Value=ml-infrastructure Key=Environment,Value=dev
```

### 任务 4.2：创建成本预算

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > budget.json <<EOF
{
  "BudgetName": "ml-platform-monthly-budget",
  "BudgetLimit": {
    "Amount": "500",
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
      {"SubscriptionType": "EMAIL", "Address": "ml-team@example.com"}
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
      {"SubscriptionType": "EMAIL", "Address": "ml-team@example.com"}
    ]
  }
]
EOF

aws budgets create-budget \
  --account-id $ACCOUNT_ID \
  --budget file://budget.json \
  --notifications-with-subscribers file://notifications.json
```

---

## 清理

```bash
# 删除 SageMaker 端点
aws sagemaker delete-endpoint --endpoint-name ml-inference-endpoint

# 删除 SageMaker 模型
aws sagemaker delete-model --model-name <model-name>

# 清空并删除 S3 存储桶
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME
```

---

## 交付物

到本练习结束时，你应该有：

1. ✅ 使用 Spot 实例的 SageMaker 训练作业
2. ✅ 用于模型优化的超参数调优
3. ✅ 带自动扩缩容的 SageMaker 端点
4. ✅ 完整的成本分配标签策略
5. ✅ 成本预算和告警
6. ✅ 成本优化报告（节省 47%）
7. ✅ AWS 上的端到端 ML 平台
8. ✅ 全面的监控和告警

---

## 恭喜！

你在 AWS 上从头开始构建了一个完整的、生产级的 ML 平台！你已经掌握了：

- **模块 001-009**：基础、Python、Linux、ML、Docker、Kubernetes、API、数据库、监控
- **模块 010**：云平台（AWS）

**接下来**：参加模块 010 测验来测试你的知识！

🚀 **你现在已经准备好担任 AI 基础设施工程师的角色！**
