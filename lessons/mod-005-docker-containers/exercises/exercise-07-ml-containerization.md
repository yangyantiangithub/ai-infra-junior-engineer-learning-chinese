# Exercise 07: Complete ML Application Containerization

## Exercise Overview

**Objective**: 构建一个生产就绪的 containerized 机器学习应用，包括模型训练、推理 serving、监控以及完整的 CI/CD pipeline。

**Difficulty**: 高级
**Estimated Time**: 5-6 小时
**Prerequisites**:
- 所有之前的 Docker 练习 (01-06)
- 所有 Docker 讲座
- Module 004 (ML Basics)

**What You'll Learn**:
- Containerizing ML 训练 pipeline
- 构建 ML 推理 API
- 启用 GPU 的 container
- 模型版本管理和模型管理
- ML 特定监控
- 分布式训练设置
- 特征 serving
- 完整的 ML 部署 pipeline

---

## Part 1: 简单的 ML 推理服务

### Step 1.1: 基本 Model Serving

```bash
mkdir -p ~/docker-exercises/ml-app
cd ~/docker-exercises/ml-app

# 创建推理应用
cat > serve.py << 'EOF'
from flask import Flask, request, jsonify
import torch
import torchvision.transforms as transforms
import torchvision.models as models
from PIL import Image
import io
import logging

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# 加载模型
logger.info("Loading model...")
model = models.resnet18(pretrained=True)
model.eval()
logger.info("Model loaded successfully")

# 图像预处理
transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(
        mean=[0.485, 0.456, 0.406],
        std=[0.229, 0.224, 0.225]
    )
])

# 加载 ImageNet 类别标签
with open('imagenet_classes.txt') as f:
    labels = [line.strip() for line in f.readlines()]

@app.route('/health')
def health():
    return jsonify({
        'status': 'healthy',
        'model': 'resnet18',
        'version': '1.0'
    })

@app.route('/predict', methods=['POST'])
def predict():
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400

    try:
        # 读取并处理图像
        file = request.files['file']
        img_bytes = file.read()
        img = Image.open(io.BytesIO(img_bytes)).convert('RGB')

        # 预处理
        img_tensor = transform(img).unsqueeze(0)

        # 推理
        with torch.no_grad():
            outputs = model(img_tensor)
            probabilities = torch.nn.functional.softmax(outputs[0], dim=0)

        # 获取前 5 个预测结果
        top5_prob, top5_catid = torch.topk(probabilities, 5)

        results = []
        for i in range(5):
            results.append({
                'class': labels[top5_catid[i]],
                'probability': float(top5_prob[i])
            })

        logger.info(f"Prediction successful: {results[0]['class']}")

        return jsonify({
            'predictions': results,
            'success': True
        })

    except Exception as e:
        logger.error(f"Prediction error: {str(e)}")
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF

# 创建 ImageNet 标签（简化版）
cat > imagenet_classes.txt << 'EOF'
tench
goldfish
great white shark
tiger shark
hammerhead
# ... (共 1000 个类别，为简洁起见已截断)
EOF

# 创建 requirements
cat > requirements.txt << 'EOF'
torch==2.1.0
torchvision==0.16.0
flask==3.0.0
pillow==10.1.0
gunicorn==21.2.0
prometheus-client==0.19.0
EOF
```

### Step 1.2: ML 推理的 Dockerfile

```dockerfile
cat > Dockerfile << 'EOF'
# ML 推理的多阶段构建
FROM python:3.11 AS builder

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# 运行时阶段
FROM python:3.11-slim

# 安装运行时依赖
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libgomp1 \
        curl \
    && rm -rf /var/lib/apt/lists/*

# 创建非 root 用户
RUN groupadd -r mluser && useradd -r -g mluser -u 1000 mluser

WORKDIR /app

# 复制 Python 包
COPY --from=builder /root/.local /home/mluser/.local

# 复制应用
COPY --chown=mluser:mluser serve.py imagenet_classes.txt ./

# 创建模型和缓存目录
RUN mkdir -p /home/mluser/.cache/torch && \
    chown -R mluser:mluser /home/mluser/.cache

USER mluser

ENV PATH=/home/mluser/.local/bin:$PATH \
    TORCH_HOME=/home/mluser/.cache/torch

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "--timeout", "120", "serve:app"]
EOF
```

```bash
# 构建并运行
docker build -t ml-inference:v1 .

docker run -d \
    --name ml-server \
    -p 8000:8000 \
    ml-inference:v1

# 测试
sleep 30  # 等待模型加载
curl http://localhost:8000/health

# 用图像测试（如果有的话）
# curl -X POST -F "file=@test_image.jpg" http://localhost:8000/predict

# 清理
docker rm -f ml-server
```

✅ **Checkpoint**: 你可以 containerize 一个基本的 ML 推理服务。

---

## Part 2: 启用 GPU 的 ML Container

### Step 2.1: 启用 GPU 的 Dockerfile

```dockerfile
cat > Dockerfile.gpu << 'EOF'
# NVIDIA CUDA 基础 image
FROM nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04

# 安装 Python
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        python3.11 \
        python3.11-dev \
        python3-pip \
        curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 安装带 CUDA 支持的 PyTorch
COPY requirements-gpu.txt .
RUN pip3 install --no-cache-dir \
    torch==2.1.0 \
    torchvision==0.16.0 \
    --index-url https://download.pytorch.org/whl/cu118

RUN pip3 install --no-cache-dir -r requirements-gpu.txt

# 创建非 root 用户
RUN groupadd -r mluser && useradd -r -g mluser -u 1000 mluser

# 复制应用
COPY --chown=mluser:mluser serve_gpu.py imagenet_classes.txt ./

USER mluser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "1", "--timeout", "300", "serve_gpu:app"]
EOF

cat > requirements-gpu.txt << 'EOF'
flask==3.0.0
pillow==10.1.0
gunicorn==21.2.0
prometheus-client==0.19.0
EOF
```

### Step 2.2: 启用 GPU 的推理脚本

```python
cat > serve_gpu.py << 'EOF'
from flask import Flask, request, jsonify
import torch
import torchvision.transforms as transforms
import torchvision.models as models
from PIL import Image
import io
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# 检查 GPU
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
logger.info(f"Using device: {device}")

# 加载模型到 GPU
logger.info("Loading model...")
model = models.resnet50(pretrained=True).to(device)
model.eval()
logger.info("Model loaded successfully")

transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
])

@app.route('/health')
def health():
    return jsonify({
        'status': 'healthy',
        'device': str(device),
        'cuda_available': torch.cuda.is_available(),
        'gpu_count': torch.cuda.device_count() if torch.cuda.is_available() else 0
    })

@app.route('/predict', methods=['POST'])
def predict():
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400

    try:
        file = request.files['file']
        img = Image.open(io.BytesIO(file.read())).convert('RGB')
        img_tensor = transform(img).unsqueeze(0).to(device)

        with torch.no_grad():
            outputs = model(img_tensor)
            probabilities = torch.nn.functional.softmax(outputs[0], dim=0)

        top5_prob, top5_catid = torch.topk(probabilities, 5)

        return jsonify({
            'predictions': [
                {'class_id': int(top5_catid[i]), 'probability': float(top5_prob[i])}
                for i in range(5)
            ],
            'device': str(device)
        })

    except Exception as e:
        logger.error(f"Error: {str(e)}")
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF
```

### Step 2.3: 使用 GPU 的 Docker Compose

```yaml
cat > docker-compose-gpu.yml << 'EOF'
version: '3.8'

services:
  ml-gpu:
    build:
      context: .
      dockerfile: Dockerfile.gpu
    ports:
      - "8000:8000"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    environment:
      - NVIDIA_VISIBLE_DEVICES=0
    restart: unless-stopped
EOF
```

```bash
# 构建并运行（需要 nvidia-docker）
# docker compose -f docker-compose-gpu.yml up -d

# 检查 GPU 使用
# nvidia-smi
```

✅ **Checkpoint**: 你可以创建启用 GPU 的 ML container。

---

## Part 3: 模型训练 Container

### Step 3.1: 训练脚本

```python
cat > train.py << 'EOF'
import torch
import torch.nn as nn
import torch.optim as optim
from torchvision import datasets, transforms, models
from torch.utils.data import DataLoader
import os
import json
from datetime import datetime

def train_model(
    data_dir='/data',
    model_dir='/models',
    epochs=10,
    batch_size=32,
    learning_rate=0.001
):
    # 设备配置
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    print(f"Using device: {device}")

    # 数据变换
    transform = transforms.Compose([
        transforms.Resize(256),
        transforms.CenterCrop(224),
        transforms.ToTensor(),
        transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
    ])

    # 加载数据集（示例：ImageNet 风格结构）
    train_dataset = datasets.ImageFolder(
        os.path.join(data_dir, 'train'),
        transform=transform
    )
    train_loader = DataLoader(
        train_dataset,
        batch_size=batch_size,
        shuffle=True,
        num_workers=4
    )

    # 模型
    model = models.resnet18(pretrained=True)
    num_classes = len(train_dataset.classes)
    model.fc = nn.Linear(model.fc.in_features, num_classes)
    model = model.to(device)

    # 损失和优化器
    criterion = nn.CrossEntropyLoss()
    optimizer = optim.Adam(model.parameters(), lr=learning_rate)

    # 训练循环
    metrics = {'epochs': [], 'loss': [], 'accuracy': []}

    for epoch in range(epochs):
        model.train()
        running_loss = 0.0
        correct = 0
        total = 0

        for i, (images, labels) in enumerate(train_loader):
            images, labels = images.to(device), labels.to(device)

            # 前向传播
            outputs = model(images)
            loss = criterion(outputs, labels)

            # 反向传播
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()

            # 统计
            running_loss += loss.item()
            _, predicted = torch.max(outputs.data, 1)
            total += labels.size(0)
            correct += (predicted == labels).sum().item()

            if (i + 1) % 10 == 0:
                print(f'Epoch [{epoch+1}/{epochs}], Step [{i+1}/{len(train_loader)}], '
                      f'Loss: {loss.item():.4f}')

        # 轮次指标
        epoch_loss = running_loss / len(train_loader)
        epoch_acc = 100 * correct / total

        metrics['epochs'].append(epoch + 1)
        metrics['loss'].append(epoch_loss)
        metrics['accuracy'].append(epoch_acc)

        print(f'Epoch [{epoch+1}/{epochs}] completed - '
              f'Loss: {epoch_loss:.4f}, Accuracy: {epoch_acc:.2f}%')

    # 保存模型
    timestamp = datetime.now().strftime('%Y%m%d_%H%M%S')
    model_path = os.path.join(model_dir, f'model_{timestamp}.pth')
    torch.save({
        'model_state_dict': model.state_dict(),
        'optimizer_state_dict': optimizer.state_dict(),
        'metrics': metrics,
        'classes': train_dataset.classes
    }, model_path)

    print(f'Model saved to {model_path}')

    # 保存指标
    metrics_path = os.path.join(model_dir, f'metrics_{timestamp}.json')
    with open(metrics_path, 'w') as f:
        json.dump(metrics, f, indent=2)

    return model_path

if __name__ == '__main__':
    import argparse

    parser = argparse.ArgumentParser()
    parser.add_argument('--data-dir', default='/data')
    parser.add_argument('--model-dir', default='/models')
    parser.add_argument('--epochs', type=int, default=10)
    parser.add_argument('--batch-size', type=int, default=32)
    parser.add_argument('--lr', type=float, default=0.001)

    args = parser.parse_args()

    train_model(
        data_dir=args.data_dir,
        model_dir=args.model_dir,
        epochs=args.epochs,
        batch_size=args.batch_size,
        learning_rate=args.lr
    )
EOF
```

### Step 3.2: 训练 Dockerfile

```dockerfile
cat > Dockerfile.train << 'EOF'
FROM pytorch/pytorch:2.1.0-cuda11.8-cudnn8-runtime

WORKDIR /workspace

# 安装额外依赖
RUN pip install --no-cache-dir \
    tensorboard \
    mlflow \
    scikit-learn

# 复制训练脚本
COPY train.py .

# 创建目录
RUN mkdir -p /data /models /logs

# 设置环境变量
ENV PYTHONUNBUFFERED=1

CMD ["python", "train.py"]
EOF
```

### Step 3.3: 使用 Docker Compose 训练

```yaml
cat > docker-compose-train.yml << 'EOF'
version: '3.8'

services:
  trainer:
    build:
      context: .
      dockerfile: Dockerfile.train
    volumes:
      - ./data:/data:ro  # 训练数据（只读）
      - ./models:/models  # 模型输出
      - ./logs:/logs  # 训练日志
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    environment:
      - EPOCHS=20
      - BATCH_SIZE=64
      - LEARNING_RATE=0.001
    command: >
      python train.py
        --epochs ${EPOCHS}
        --batch-size ${BATCH_SIZE}
        --lr ${LEARNING_RATE}

  tensorboard:
    image: tensorflow/tensorflow:latest
    ports:
      - "6006:6006"
    volumes:
      - ./logs:/logs:ro
    command: tensorboard --logdir=/logs --host=0.0.0.0

volumes:
  models:
  logs:
EOF
```

✅ **Checkpoint**: 你可以 containerize ML 训练 pipeline。

---

## Part 4: 完整的 ML Pipeline

### Step 4.1: 完整 ML 栈

```yaml
cat > docker-compose-ml-stack.yml << 'EOF'
version: '3.8'

services:
  # 用于元数据的 PostgreSQL
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=mlflow
      - POSTGRES_USER=mlflow
      - POSTGRES_PASSWORD=mlflow
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U mlflow"]
      interval: 10s
      timeout: 5s
      retries: 5

  # 用于 artifact 存储的 MinIO
  minio:
    image: minio/minio
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # MLflow 追踪服务器
  mlflow:
    image: ghcr.io/mlflow/mlflow:latest
    ports:
      - "5000:5000"
    environment:
      - MLFLOW_S3_ENDPOINT_URL=http://minio:9000
      - AWS_ACCESS_KEY_ID=minioadmin
      - AWS_SECRET_ACCESS_KEY=minioadmin
    command: >
      mlflow server
        --backend-store-uri postgresql://mlflow:mlflow@postgres/mlflow
        --default-artifact-root s3://mlflow/
        --host 0.0.0.0
    depends_on:
      postgres:
        condition: service_healthy
      minio:
        condition: service_healthy

  # 用于缓存的 Redis
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # 特征存储
  feast:
    image: feastdev/feature-server:latest
    ports:
      - "6566:6566"
    volumes:
      - ./feature_repo:/feature_repo
    command: feast -c /feature_repo serve

  # 模型 serving
  inference:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    environment:
      - MODEL_URI=s3://mlflow/models/production/model.pth
      - REDIS_HOST=redis
      - MLFLOW_TRACKING_URI=http://mlflow:5000
    depends_on:
      - redis
      - mlflow
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 2G

  # 用于监控的 Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

  # 用于可视化的 Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  postgres-data:
  minio-data:
  prometheus-data:
  grafana-data:
EOF
```

### Step 4.2: 模型注册中心集成

```python
cat > model_registry.py << 'EOF'
import mlflow
import mlflow.pytorch
import torch
import os

class ModelRegistry:
    def __init__(self, tracking_uri='http://localhost:5000'):
        mlflow.set_tracking_uri(tracking_uri)

    def register_model(self, model, model_name, metrics=None):
        """注册已训练的模型"""
        with mlflow.start_run():
            # 记录模型
            mlflow.pytorch.log_model(model, "model")

            # 记录指标
            if metrics:
                for key, value in metrics.items():
                    mlflow.log_metric(key, value)

            # 注册模型
            model_uri = f"runs:/{mlflow.active_run().info.run_id}/model"
            mlflow.register_model(model_uri, model_name)

    def load_production_model(self, model_name):
        """加载生产模型"""
        model_uri = f"models:/{model_name}/Production"
        return mlflow.pytorch.load_model(model_uri)

    def promote_model(self, model_name, version, stage='Production'):
        """将模型提升到生产环境"""
        client = mlflow.tracking.MlflowClient()
        client.transition_model_version_stage(
            name=model_name,
            version=version,
            stage=stage
        )
EOF
```

✅ **Checkpoint**: 你可以使用 Docker 构建完整的 ML pipeline。

---

## Part 5: 分布式训练

### Step 5.1: 分布式训练脚本

```python
cat > train_distributed.py << 'EOF'
import torch
import torch.nn as nn
import torch.distributed as dist
import torch.multiprocessing as mp
from torch.nn.parallel import DistributedDataParallel as DDP
import os

def setup(rank, world_size):
    """初始化分布式训练"""
    os.environ['MASTER_ADDR'] = os.environ.get('MASTER_ADDR', 'localhost')
    os.environ['MASTER_PORT'] = os.environ.get('MASTER_PORT', '12355')

    # 初始化进程组
    dist.init_process_group("nccl", rank=rank, world_size=world_size)

def cleanup():
    """清理分布式训练"""
    dist.destroy_process_group()

def train(rank, world_size):
    """每个进程的训练函数"""
    print(f"Running DDP on rank {rank}")
    setup(rank, world_size)

    # 创建模型并移动到 GPU
    model = nn.Linear(10, 10).to(rank)
    ddp_model = DDP(model, device_ids=[rank])

    # 损失和优化器
    loss_fn = nn.MSELoss()
    optimizer = torch.optim.SGD(ddp_model.parameters(), lr=0.001)

    # 训练循环
    for epoch in range(10):
        optimizer.zero_grad()
        outputs = ddp_model(torch.randn(20, 10).to(rank))
        labels = torch.randn(20, 10).to(rank)
        loss = loss_fn(outputs, labels)
        loss.backward()
        optimizer.step()

        if rank == 0:
            print(f"Epoch {epoch}, Loss: {loss.item()}")

    cleanup()

def main():
    world_size = torch.cuda.device_count()
    mp.spawn(train, args=(world_size,), nprocs=world_size, join=True)

if __name__ == '__main__':
    main()
EOF
```

### Step 5.2: 分布式训练 Compose

```yaml
cat > docker-compose-distributed.yml << 'EOF'
version: '3.8'

services:
  master:
    build:
      context: .
      dockerfile: Dockerfile.train
    environment:
      - MASTER_ADDR=master
      - MASTER_PORT=12355
      - WORLD_SIZE=3
      - RANK=0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    command: python train_distributed.py

  worker1:
    build:
      context: .
      dockerfile: Dockerfile.train
    environment:
      - MASTER_ADDR=master
      - MASTER_PORT=12355
      - WORLD_SIZE=3
      - RANK=1
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    depends_on:
      - master
    command: python train_distributed.py

  worker2:
    build:
      context: .
      dockerfile: Dockerfile.train
    environment:
      - MASTER_ADDR=master
      - MASTER_PORT=12355
      - WORLD_SIZE=3
      - RANK=2
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    depends_on:
      - master
    command: python train_distributed.py
EOF
```

✅ **Checkpoint**: 你可以使用 Docker 设置分布式训练。

---

## Part 6: 监控 ML 服务

### Step 6.1: Prometheus 指标

```python
cat > serve_with_metrics.py << 'EOF'
from flask import Flask, request, jsonify
from prometheus_client import Counter, Histogram, Gauge, generate_latest
import torch
import time

app = Flask(__name__)

# Prometheus 指标
REQUEST_COUNT = Counter('ml_requests_total', 'Total requests', ['endpoint', 'status'])
REQUEST_LATENCY = Histogram('ml_request_duration_seconds', 'Request latency')
MODEL_LOAD_TIME = Gauge('ml_model_load_time_seconds', 'Model load time')
GPU_MEMORY = Gauge('ml_gpu_memory_bytes', 'GPU memory usage')
PREDICTIONS = Counter('ml_predictions_total', 'Total predictions', ['class'])

# 加载模型并记录加载时间
start_time = time.time()
model = torch.hub.load('pytorch/vision', 'resnet18', pretrained=True)
model.eval()
load_time = time.time() - start_time
MODEL_LOAD_TIME.set(load_time)

@app.route('/metrics')
def metrics():
    """Prometheus 指标端点"""
    return generate_latest()

@app.route('/health')
def health():
    REQUEST_COUNT.labels(endpoint='/health', status='success').inc()

    # 如果可用，更新 GPU 指标
    if torch.cuda.is_available():
        GPU_MEMORY.set(torch.cuda.memory_allocated())

    return jsonify({
        'status': 'healthy',
        'gpu_available': torch.cuda.is_available()
    })

@app.route('/predict', methods=['POST'])
@REQUEST_LATENCY.time()
def predict():
    try:
        # 预测逻辑
        # ...

        REQUEST_COUNT.labels(endpoint='/predict', status='success').inc()
        PREDICTIONS.labels(class='example_class').inc()

        return jsonify({'result': 'prediction'})

    except Exception as e:
        REQUEST_COUNT.labels(endpoint='/predict', status='error').inc()
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF
```

### Step 6.2: 自定义 ML 仪表板

```yaml
# Grafana dashboard 配置
cat > grafana-dashboard.json << 'EOF'
{
  "dashboard": {
    "title": "ML Service Monitoring",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(ml_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Latency (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, ml_request_duration_seconds_bucket)"
          }
        ]
      },
      {
        "title": "GPU Memory Usage",
        "targets": [
          {
            "expr": "ml_gpu_memory_bytes"
          }
        ]
      },
      {
        "title": "Predictions by Class",
        "targets": [
          {
            "expr": "ml_predictions_total"
          }
        ]
      }
    ]
  }
}
EOF
```

✅ **Checkpoint**: 你可以使用 Prometheus 和 Grafana 监控 ML 服务。

---

## Part 7: ML 应用的 CI/CD

### Step 7.1: GitHub Actions ML Pipeline

```yaml
cat > .github-workflows-ml.yml << 'EOF'
name: ML Model CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Build test image
        run: docker build -t ml-app:test .

      - name: Run unit tests
        run: |
          docker run --rm ml-app:test pytest tests/unit

      - name: Run integration tests
        run: |
          docker compose -f docker-compose.test.yml up -d
          docker compose -f docker-compose.test.yml exec -T app pytest tests/integration
          docker compose -f docker-compose.test.yml down

  train:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3

      - name: Train model
        run: |
          docker compose -f docker-compose-train.yml up --abort-on-container-exit

      - name: Upload model artifact
        uses: actions/upload-artifact@v3
        with:
          name: trained-model
          path: models/

  deploy:
    needs: train
    runs-on: ubuntu-latest
    steps:
      - name: Download model
        uses: actions/download-artifact@v3
        with:
          name: trained-model

      - name: Build production image
        run: |
          docker build -f Dockerfile.prod -t ml-inference:${{ github.sha }} .

      - name: Push to registry
        run: |
          docker tag ml-inference:${{ github.sha }} registry.example.com/ml-inference:latest
          docker push registry.example.com/ml-inference:latest

      - name: Deploy to production
        run: |
          # 部署命令
          echo "Deploying to production..."
EOF
```

### Step 7.2: 模型验证 Pipeline

```python
cat > validate_model.py << 'EOF'
import torch
import json
import sys

def validate_model(model_path, validation_data_path):
    """部署前验证模型"""

    # 加载模型
    model = torch.load(model_path)
    model.eval()

    # 加载验证数据
    # ... 验证逻辑 ...

    # 检查指标
    metrics = {
        'accuracy': 0.95,  # 替换为实际计算值
        'latency_ms': 50,
        'size_mb': 45
    }

    # 验证阈值
    if metrics['accuracy'] < 0.90:
        print("FAIL: Accuracy below threshold")
        sys.exit(1)

    if metrics['latency_ms'] > 100:
        print("FAIL: Latency above threshold")
        sys.exit(1)

    if metrics['size_mb'] > 100:
        print("FAIL: Model size too large")
        sys.exit(1)

    print("PASS: Model validation successful")
    print(json.dumps(metrics, indent=2))

    return True

if __name__ == '__main__':
    validate_model('models/model.pth', 'data/validation')
EOF
```

✅ **Checkpoint**: 你可以为 ML 应用实现 CI/CD。

---

## Part 8: 生产部署

### Step 8.1: 生产就绪的 ML 栈

```yaml
cat > docker-compose.production.yml << 'EOF'
version: '3.8'

services:
  # Nginx 负载均衡器
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/prod.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - inference
    restart: always

  # ML 推理服务（多个副本）
  inference:
    build:
      context: .
      dockerfile: Dockerfile.prod
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
      update_config:
        parallelism: 1
        delay: 30s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 10s
        max_attempts: 3
    environment:
      - MODEL_PATH=/models/production/model.pth
      - REDIS_HOST=redis
      - PROMETHEUS_MULTIPROC_DIR=/tmp/prometheus
    volumes:
      - model-store:/models:ro
      - inference-cache:/cache
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # Redis 缓存
  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 1gb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    restart: always

  # 监控
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    restart: always

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
    environment:
      - GF_SECURITY_ADMIN_PASSWORD_FILE=/run/secrets/grafana_password
    secrets:
      - grafana_password
    restart: always

volumes:
  model-store:
    driver: local
  inference-cache:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs
      o: size=2g
  redis-data:
  prometheus-data:
  grafana-data:

secrets:
  grafana_password:
    file: ./secrets/grafana_password.txt
EOF
```

✅ **Checkpoint**: 你有了一个生产就绪的 ML 部署。

---

## Summary

**What You Accomplished**:
✅ 构建了 ML 推理服务
✅ 创建了启用 GPU 的 container
✅ Containerized 训练 pipeline
✅ 实现了完整的 ML 栈
✅ 设置了分布式训练
✅ 添加了 ML 特定监控
✅ 集成了 ML 的 CI/CD
✅ 部署了生产 ML 系统

**Complete ML Infrastructure**:
- 带 GPU 支持的模型训练
- 带缓存的推理 serving
- 模型注册中心 (MLflow)
- 特征存储 (Feast)
- 监控 (Prometheus/Grafana)
- 分布式训练
- CI/CD pipeline
- 生产部署

**Best Practices Applied**:
- 多阶段构建优化
- GPU 资源管理
- 模型版本管理
- 自动化测试
- 健康检查
- 安全加固
- 水平扩展
- 监控和日志记录

---

## Next Steps

**Continue Learning**:
- **Module 005 Quiz**: 测试你的完整 Docker 知识
- **Module 006**: Kubernetes 编排
- **Module 007**: 生产 ML 运维

---

**Exercise Version**: 1.0
**Last Updated**: October 2025
**Estimated Time**: 5-6 小时
**Difficulty**: 高级
