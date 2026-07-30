# Exercise 08: Production-Ready ML Docker Deployment

## Overview

本综合练习将指导你为 ML 应用构建生产就绪的 Docker image，实现多阶段构建优化，设置健康检查，配置资源限制，并使用适当的监控和安全措施部署到生产环境。

## Learning Objectives

完成本练习后，你将能够：
- 为 ML 应用构建优化的多阶段 Docker image
- 实现健康检查和就绪探测
- 配置 GPU 和 CPU 工作负载的资源限制
- 设置适当的日志记录和监控
- 实现安全最佳实践
- 创建生产部署配置
- 安全地处理密钥和环境变量
- 构建用于自动化 Docker 构建的 CI/CD pipeline

## Prerequisites

- 已完成本模块中的练习 01-07
- Module 005 中的所有 Docker 讲座
- 理解 ML 推理工作流
- Kubernetes 基本知识（有帮助但非必需）

## Time Required

- 预计时间：120-150 分钟
- 难度：高级

---

## Part 1: ML 应用的多阶段 Dockerfile

### Task 1.1: 创建生产优化的 Dockerfile

创建一个带多阶段构建的 FastAPI ML 推理服务：

```dockerfile
# Dockerfile.production
# 阶段 1：带开发工具的构建阶段
FROM python:3.11-slim as builder

# 安装构建依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# 创建虚拟环境
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 先复制 requirements（更好的缓存）
COPY requirements.txt .

# 安装 Python 依赖
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# 阶段 2：运行时阶段（生产）
FROM python:3.11-slim

# 为安全添加非 root 用户
RUN groupadd -r mluser && useradd -r -g mluser mluser

# 从构建阶段复制虚拟环境
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# 设置工作目录
WORKDIR /app

# 复制应用代码
COPY --chown=mluser:mluser app/ /app/
COPY --chown=mluser:mluser models/ /app/models/

# 切换到非 root 用户
USER mluser

# 暴露端口
EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 运行应用
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Task 1.2: 创建应用文件

**requirements.txt:**
```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
onnxruntime==1.16.3
numpy==1.24.3
prometheus-client==0.19.0
```

**app/main.py:**
```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import onnxruntime as ort
import numpy as np
from prometheus_client import Counter, Histogram, generate_latest
import time
import logging

# 配置日志
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Prometheus 指标
PREDICTION_COUNT = Counter('predictions_total', 'Total predictions')
PREDICTION_LATENCY = Histogram('prediction_latency_seconds', 'Prediction latency')
ERROR_COUNT = Counter('prediction_errors_total', 'Total prediction errors')

app = FastAPI(title="ML Inference API", version="1.0.0")

# 启动时加载模型
model_session = None

@app.on_event("startup")
async def load_model():
    global model_session
    try:
        model_session = ort.InferenceSession("models/model.onnx")
        logger.info("Model loaded successfully")
    except Exception as e:
        logger.error(f"Failed to load model: {e}")
        raise

class PredictionRequest(BaseModel):
    features: list[float]

class PredictionResponse(BaseModel):
    prediction: float
    latency_ms: float

@app.get("/health")
async def health_check():
    """负载均衡器的健康检查端点"""
    if model_session is None:
        raise HTTPException(status_code=503, detail="Model not loaded")
    return {"status": "healthy", "model": "loaded"}

@app.get("/ready")
async def readiness_check():
    """Kubernetes 的就绪检查"""
    if model_session is None:
        raise HTTPException(status_code=503, detail="Not ready")
    return {"status": "ready"}

@app.post("/predict", response_model=PredictionResponse)
async def predict(request: PredictionRequest):
    """进行预测"""
    start_time = time.time()

    try:
        # 准备输入
        input_data = np.array([request.features], dtype=np.float32)

        # 运行推理
        outputs = model_session.run(None, {"input": input_data})
        prediction = float(outputs[0][0])

        # 指标
        latency = (time.time() - start_time) * 1000
        PREDICTION_COUNT.inc()
        PREDICTION_LATENCY.observe(time.time() - start_time)

        return PredictionResponse(
            prediction=prediction,
            latency_ms=latency
        )

    except Exception as e:
        ERROR_COUNT.inc()
        logger.error(f"Prediction error: {e}")
        raise HTTPException(status_code=500, detail=str(e))

@app.get("/metrics")
async def metrics():
    """Prometheus 指标端点"""
    return generate_latest()

@app.get("/")
async def root():
    return {
        "service": "ML Inference API",
        "version": "1.0.0",
        "endpoints": {
            "/health": "Health check",
            "/ready": "Readiness check",
            "/predict": "Prediction endpoint (POST)",
            "/metrics": "Prometheus metrics"
        }
    }
```

### Task 1.3: 构建和测试

```bash
# 构建多阶段 image
docker build -t ml-inference:v1.0.0 -f Dockerfile.production .

# 检查 image 大小（应小于单阶段构建）
docker images ml-inference:v1.0.0

# 运行 container
docker run -d \
    --name ml-api \
    -p 8000:8000 \
    --memory="2g" \
    --cpus="2.0" \
    ml-inference:v1.0.0

# 测试健康检查
curl http://localhost:8000/health

# 测试预测
curl -X POST http://localhost:8000/predict \
    -H "Content-Type: application/json" \
    -d '{"features": [1.0, 2.0, 3.0, 4.0]}'

# 检查指标
curl http://localhost:8000/metrics

# 查看日志
docker logs ml-api

# 检查资源使用
docker stats ml-api

# 停止 container
docker stop ml-api && docker rm ml-api
```

---

## Part 2: 开发环境的 Docker Compose

### Task 2.1: 创建 docker-compose.yml

```yaml
version: '3.8'

services:
  ml-api:
    build:
      context: .
      dockerfile: Dockerfile.production
    container_name: ml-inference-api
    ports:
      - "8000:8000"
    environment:
      - LOG_LEVEL=INFO
      - MODEL_PATH=/app/models/model.onnx
    volumes:
      - ./models:/app/models:ro  # 只读模型挂载
    networks:
      - ml-network
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '1.0'
          memory: 1G
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 3s
      retries: 3
      start_period: 40s
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - ml-network
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - ml-network
    restart: unless-stopped

networks:
  ml-network:
    driver: bridge

volumes:
  prometheus-data:
  grafana-data:
```

### Task 2.2: Prometheus 配置

**prometheus.yml:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ml-api'
    static_configs:
      - targets: ['ml-api:8000']
    metrics_path: '/metrics'
```

### Task 2.3: 启动栈

```bash
# 启动所有服务
docker-compose up -d

# 查看日志
docker-compose logs -f ml-api

# 检查服务
docker-compose ps

# 测试 API
curl http://localhost:8000/health

# 访问 Prometheus
open http://localhost:9090

# 访问 Grafana (admin/admin)
open http://localhost:3000

# 停止所有服务
docker-compose down

# 停止并删除 volume
docker-compose down -v
```

---

## Part 3: GPU 支持

### Task 3.1: 带 CUDA 的 Dockerfile

```dockerfile
# Dockerfile.gpu
FROM nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04

# 安装 Python
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3.11 \
    python3-pip \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 安装带 CUDA 的 PyTorch
RUN pip3 install --no-cache-dir \
    torch==2.0.1+cu118 \
    torchvision==0.15.2+cu118 \
    --extra-index-url https://download.pytorch.org/whl/cu118

# 复制 requirements 和应用
COPY requirements.txt .
RUN pip3 install --no-cache-dir -r requirements.txt

WORKDIR /app
COPY app/ /app/

EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["python3", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Task 3.2: 使用 GPU 运行

```bash
# 构建 GPU image
docker build -t ml-inference:v1.0.0-gpu -f Dockerfile.gpu .

# 使用 GPU 访问运行
docker run -d \
    --name ml-api-gpu \
    --gpus all \
    -p 8000:8000 \
    --memory="4g" \
    ml-inference:v1.0.0-gpu

# 验证 container 内部的 GPU 访问
docker exec ml-api-gpu nvidia-smi

# 监控 GPU 使用
watch -n 1 docker exec ml-api-gpu nvidia-smi
```

---

## Part 4: 安全最佳实践

### Task 4.1: 扫描漏洞

```bash
# 安装 Trivy
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

# 扫描 image 漏洞
trivy image ml-inference:v1.0.0

# 仅扫描高/严重级别
trivy image --severity HIGH,CRITICAL ml-inference:v1.0.0

# 生成报告
trivy image --format json --output report.json ml-inference:v1.0.0
```

### Task 4.2: 实现密钥管理

**使用 Docker secrets：**

```bash
# 创建密钥
echo "my-api-key-12345" | docker secret create api_key -

# 使用密钥运行（Swarm 模式）
docker service create \
    --name ml-api \
    --secret api_key \
    --publish 8000:8000 \
    ml-inference:v1.0.0
```

**使用环境文件：**

**.env（不要提交到 Git!）：**
```bash
API_KEY=my-secure-api-key
MODEL_PATH=/app/models/model.onnx
LOG_LEVEL=INFO
```

```bash
# 使用 env 文件运行
docker run -d \
    --env-file .env \
    -p 8000:8000 \
    ml-inference:v1.0.0
```

---

## Part 5: CI/CD Pipeline

### Task 5.1: GitHub Actions 工作流

**.github/workflows/docker-build.yml：**
```yaml
name: Build and Push Docker Image

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          file: ./Dockerfile.production
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels}}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

---

## Part 6: Kubernetes 部署

### Task 6.1: 创建 Kubernetes 清单

**deployment.yaml：**
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
      - name: ml-api
        image: ml-inference:v1.0.0
        ports:
        - containerPort: 8000
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "2Gi"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
        env:
        - name: LOG_LEVEL
          value: "INFO"
---
apiVersion: v1
kind: Service
metadata:
  name: ml-inference-service
spec:
  selector:
    app: ml-inference
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: LoadBalancer
```

---

## Deliverables

提交以下内容：

1. **Dockerfiles**：
   - CPU 的多阶段 Dockerfile
   - GPU 工作负载的 Dockerfile
   - 完整栈的 docker-compose.yml

2. **Application Code**：
   - FastAPI 推理服务
   - 健康检查端点
   - Prometheus 指标

3. **CI/CD**：
   - GitHub Actions 工作流
   - 自动化漏洞扫描

4. **Documentation**：
   - 部署指南
   - 安全检查清单
   - 排错手册

5. **Evidence**：
   - 构建日志
   - Trivy 扫描结果
   - 性能基准测试

---

## Challenge Questions

1. **Image Size**: 你的 image 是 2GB。如何将其减少到 <500MB？

2. **Security**: ML Docker image 中的前 5 大安全风险是什么？

3. **Performance**: 如何优化大于 1GB 模型的 container 启动时间？

4. **Scaling**: 如何基于 GPU 利用率进行自动扩展？

5. **Monitoring**: 生产 ML container 应跟踪哪些指标？

---

## Additional Resources

- [Docker Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [NVIDIA Container Toolkit](https://github.com/NVIDIA/nvidia-docker)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)

---

**Congratulations!** 你已经构建了一个生产就绪的 ML Docker 部署，包含安全、监控和 CI/CD 自动化。
