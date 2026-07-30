# Exercise 04：部署容器化 ML 推理服务

**模块**：云平台
**难度**：中级-高级
**预计时间**：5-6 小时
**前置要求**：练习 01-03、讲座 04（部署与 ML 工作负载）

---

## 学习目标

完成本练习后，你将：
1. 使用 Docker 构建容器化 ML 推理应用
2. 将 Docker 镜像推送到 Amazon ECR（Elastic Container Registry）
3. 使用 ECS Fargate 和 Application Load Balancer 部署容器
4. 配置基于 CPU 和请求数的自动扩缩容
5. 实现零停机更新的蓝绿部署
6. 在 EKS（Kubernetes）上部署相同应用以作对比
7. 设置容器化工作负载的 CloudWatch 监控和告警
8. 实施容器部署的成本优化策略

---

## 概述

本练习使用练习 03 中的 VPC 部署生产级容器化 ML 推理服务。你将容器化一个 PyTorch 图像分类 API，并使用 ECS 和 EKS 进行部署，以了解它们的区别。

**实际场景**：你的团队构建了一个图像分类模型（ResNet50）。你需要将其部署为可扩展、高可用的 API，能够处理每秒 1000+ 请求，并具备自动扩缩容和零停机部署能力。

---

## 第一部分：构建 ML 推理应用

### 任务 1.1：创建用于图像分类的 Flask API

**项目结构**：
```bash
mkdir ml-inference-app && cd ml-inference-app

# 创建目录结构
mkdir -p src models tests

# 创建应用
cat > src/app.py <<'EOF'
from flask import Flask, request, jsonify
import torch
import torchvision.transforms as transforms
import torchvision.models as models
from PIL import Image
import io
import os
import logging
import time
from prometheus_client import Counter, Histogram, generate_latest

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# Prometheus 指标
REQUEST_COUNT = Counter('ml_inference_requests_total', 'Total inference requests')
REQUEST_LATENCY = Histogram('ml_inference_latency_seconds', 'Inference latency')
ERROR_COUNT = Counter('ml_inference_errors_total', 'Total errors')

# 加载模型
MODEL_PATH = os.environ.get('MODEL_PATH', '/models/resnet50.pth')
DEVICE = torch.device('cuda' if torch.cuda.is_available() else 'cpu')

logger.info(f"Loading model from {MODEL_PATH}")
model = models.resnet50(pretrained=True)
model.eval()
model.to(DEVICE)
logger.info(f"Model loaded successfully on {DEVICE}")

# 图像预处理
preprocess = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]),
])

# ImageNet 类别（前 10 个用于演示）
CLASSES = {
    281: 'tabby cat',
    282: 'tiger cat',
    283: 'Persian cat',
    284: 'Siamese cat',
    285: 'Egyptian cat',
    207: 'golden retriever',
    208: 'Labrador retriever',
    209: 'German shepherd',
    210: 'beagle',
    211: 'bloodhound'
}

@app.route('/health', methods=['GET'])
def health():
    """健康检查端点"""
    return jsonify({
        'status': 'healthy',
        'model': 'resnet50',
        'device': str(DEVICE)
    }), 200

@app.route('/ready', methods=['GET'])
def ready():
    """就绪检查端点"""
    try:
        dummy_input = torch.randn(1, 3, 224, 224).to(DEVICE)
        with torch.no_grad():
            _ = model(dummy_input)
        return jsonify({'status': 'ready'}), 200
    except Exception as e:
        logger.error(f"Readiness check failed: {e}")
        return jsonify({'status': 'not ready', 'error': str(e)}), 503

@app.route('/predict', methods=['POST'])
def predict():
    """图像分类端点"""
    REQUEST_COUNT.inc()
    start_time = time.time()

    try:
        if 'file' not in request.files:
            ERROR_COUNT.inc()
            return jsonify({'error': 'No file provided'}), 400

        file = request.files['file']
        if file.filename == '':
            ERROR_COUNT.inc()
            return jsonify({'error': 'Empty filename'}), 400

        img_bytes = file.read()
        image = Image.open(io.BytesIO(img_bytes)).convert('RGB')
        input_tensor = preprocess(image).unsqueeze(0).to(DEVICE)

        with torch.no_grad():
            output = model(input_tensor)
            probabilities = torch.nn.functional.softmax(output[0], dim=0)

        top5_prob, top5_idx = torch.topk(probabilities, 5)

        predictions = []
        for i in range(5):
            idx = top5_idx[i].item()
            prob = top5_prob[i].item()
            class_name = CLASSES.get(idx, f'class_{idx}')
            predictions.append({
                'class': class_name,
                'confidence': round(prob, 4),
                'class_id': idx
            })

        latency = time.time() - start_time
        REQUEST_LATENCY.observe(latency)

        logger.info(f"Prediction completed in {latency:.3f}s - Top class: {predictions[0]['class']}")

        return jsonify({
            'predictions': predictions,
            'latency_seconds': round(latency, 3),
            'model_version': '1.0'
        }), 200

    except Exception as e:
        ERROR_COUNT.inc()
        logger.error(f"Prediction error: {e}")
        return jsonify({'error': str(e)}), 500

@app.route('/metrics', methods=['GET'])
def metrics():
    """Prometheus 指标端点"""
    return generate_latest(), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

### 任务 1.2：创建 Dockerfile

**多阶段 Dockerfile**（更小的镜像体积）：
```bash
cat > Dockerfile <<'EOF'
FROM python:3.11-slim as builder

WORKDIR /app

RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.11-slim

WORKDIR /app

COPY --from=builder /root/.local /root/.local
COPY src/ ./src/

RUN mkdir -p /models

ENV PATH=/root/.local/bin:$PATH

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:5000/health')"

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "--threads", "4", "--timeout", "60", "src.app:app"]
EOF
```

---

## 第二部分：推送到 Amazon ECR

### 任务 2.1：创建 ECR 仓库

**创建 ECR 仓库**：
```bash
# 创建仓库
REPO_URI=$(aws ecr create-repository \
  --repository-name ml-inference \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256 \
  --tags Key=Project,Value=ml-infrastructure \
  --query 'repository.repositoryUri' \
  --output text)

echo "ECR 仓库：$REPO_URI"

# 设置生命周期策略（30 天后删除旧镜像）
cat > lifecycle-policy.json <<EOF
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Delete untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {
        "type": "expire"
      }
    },
    {
      "rulePriority": 2,
      "description": "Keep only last 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
EOF

aws ecr put-lifecycle-policy \
  --repository-name ml-inference \
  --lifecycle-policy-text file://lifecycle-policy.json
```

### 任务 2.2：推送 Docker 镜像到 ECR

**认证并推送**：
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"

# 登录到 ECR
aws ecr get-login-password --region $REGION | \
  docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com

# 为 ECR 标记镜像
docker tag ml-inference:v1.0 ${REPO_URI}:v1.0
docker tag ml-inference:v1.0 ${REPO_URI}:latest

# 推送到 ECR
docker push ${REPO_URI}:v1.0
docker push ${REPO_URI}:latest

# 验证推送
aws ecr describe-images --repository-name ml-inference
```

---

## 第三部分：使用 ECS Fargate 部署

### 任务 3.1：创建 ECS 集群

**创建集群**：
```bash
# 创建 ECS 集群
CLUSTER_NAME="ml-inference-cluster"
aws ecs create-cluster \
  --cluster-name $CLUSTER_NAME \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=2 \
    capacityProvider=FARGATE_SPOT,weight=4 \
  --tags key=Project,value=ml-infrastructure

echo "ECS 集群：$CLUSTER_NAME"
```

### 任务 3.2：创建任务定义

**创建任务定义**：
```bash
cat > task-definition.json <<EOF
{
  "family": "ml-inference-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ml-inference-task-role",
  "containerDefinitions": [
    {
      "name": "ml-inference",
      "image": "${REPO_URI}:v1.0",
      "essential": true,
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
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/ml-inference",
          "awslogs-region": "${REGION}",
          "awslogs-stream-prefix": "ecs",
          "awslogs-create-group": "true"
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
EOF

# 注册任务定义
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

---

## 第四部分：自动扩缩容配置

### 任务 4.1：配置 ECS 服务自动扩缩容

**注册可扩展目标**：
```bash
# 注册可扩展目标
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/$CLUSTER_NAME/ml-inference-service \
  --min-capacity 2 \
  --max-capacity 10

# 基于 CPU 的扩展策略
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/$CLUSTER_NAME/ml-inference-service \
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

---

## 第五部分：蓝绿部署

### 任务 5.1：更新应用（v1.1）

**修改代码并构建新镜像**：
```bash
# 添加版本端点
cat >> src/app.py <<'EOF'

@app.route('/version', methods=['GET'])
def version():
    return jsonify({'version': '1.1', 'features': ['version endpoint']}), 200
EOF

# 重建 Docker 镜像
docker build -t ml-inference:v1.1 .

# 推送到 ECR
docker tag ml-inference:v1.1 ${REPO_URI}:v1.1
docker push ${REPO_URI}:v1.1

# 注册新任务定义
sed 's/:v1.0/:v1.1/g' task-definition.json > task-definition-v1.1.json
aws ecs register-task-definition --cli-input-json file://task-definition-v1.1.json
```

### 任务 5.2：蓝绿部署

**更新服务**：
```bash
# 更新服务到新任务定义
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service ml-inference-service \
  --task-definition ml-inference-task:2 \
  --deployment-configuration "maximumPercent=200,minimumHealthyPercent=100"

# 监控部署
aws ecs wait services-stable --cluster $CLUSTER_NAME --services ml-inference-service

# 测试新版本
curl http://$ALB_DNS/version
```

---

## 第六部分：在 EKS（Kubernetes）上部署

### 任务 6.1：创建 EKS 集群

**使用 eksctl 创建集群**：
```bash
# 创建 EKS 集群（需要 15-20 分钟）
eksctl create cluster \
  --name ml-inference-eks \
  --region us-east-1 \
  --nodegroup-name ml-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed \
  --version 1.28

# 配置 kubectl
aws eks update-kubeconfig --region us-east-1 --name ml-inference-eks

# 验证集群
kubectl get nodes
```

### 任务 6.2：在 EKS 上部署应用

**创建 Kubernetes 清单**：
```bash
mkdir k8s && cd k8s

cat > deployment.yaml <<EOF
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
        image: ${REPO_URI}:v1.1
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
EOF

# 部署
kubectl apply -f deployment.yaml

# 检查部署
kubectl get deployments
kubectl get pods
kubectl get svc ml-inference-service
```

---

## 第七部分：监控与可观测性

### 任务 7.1：CloudWatch 仪表盘

**创建仪表盘**：
```bash
cat > cloudwatch-dashboard.json <<EOF
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/ECS", "CPUUtilization", {"stat": "Average", "label": "CPU %"}],
          [".", "MemoryUtilization", {"stat": "Average"}]
        ],
        "period": 300,
        "region": "us-east-1",
        "title": "ECS Resource Utilization"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/ApplicationELB", "TargetResponseTime", {"stat": "Average"}],
          [".", "RequestCount", {"stat": "Sum"}]
        ],
        "period": 60,
        "region": "us-east-1",
        "title": "ALB Metrics"
      }
    }
  ]
}
EOF

aws cloudwatch put-dashboard \
  --dashboard-name ml-inference-dashboard \
  --dashboard-body file://cloudwatch-dashboard.json
```

---

## 清理

**删除 ECS 资源**：
```bash
# 删除 ECS 服务
aws ecs update-service --cluster $CLUSTER_NAME --service ml-inference-service --desired-count 0
aws ecs delete-service --cluster $CLUSTER_NAME --service ml-inference-service

# 删除 ALB
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 delete-target-group --target-group-arn $TG_ARN

# 删除 ECS 集群
aws ecs delete-cluster --cluster $CLUSTER_NAME

# 删除 ECR 仓库
aws ecr delete-repository --repository-name ml-inference --force
```

**删除 EKS 资源**：
```bash
# 删除 EKS 集群
eksctl delete cluster --name ml-inference-eks
```

---

## 交付物

到本练习结束时，你应该有：

1. ✅ 容器化 ML 推理应用
2. ✅ 推送到 Amazon ECR 的 Docker 镜像
3. ✅ 带 ALB 的 ECS Fargate 部署
4. ✅ 基于 CPU 和请求数的自动扩缩容
5. ✅ 蓝绿部署策略
6. ✅ 用于对比的 EKS 部署
7. ✅ CloudWatch 监控和告警
8. ✅ 使用 Fargate Spot 的成本优化

---

出色的生产级容器化 ML 基础设施部署工作！🎉
