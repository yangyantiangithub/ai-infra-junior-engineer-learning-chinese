# Exercise 02: Building Custom Docker Images

## Exercise Overview

**Objective**: 掌握使用 Dockerfile 创建自定义 Docker image 的艺术，包括针对 ML 应用的优化技术和 best practices。

**Difficulty**: 初级到中级
**Estimated Time**: 3-4 小时
**Prerequisites**:
- Exercise 01 (Container Operations)
- Lecture 02 (Dockerfiles & Image Building)

**What You'll Learn**:
- 从头编写 Dockerfile
- 多阶段构建优化
- Layer 缓存策略
- 构建 Python/ML 应用
- Image 标记和版本管理
- 推送到 Docker registry
- 安全 best practices
- 调试构建问题

---

## Part 1: 你的第一个 Dockerfile

### Step 1.1: 简单的 Python 应用

```bash
# 创建项目目录
mkdir -p ~/docker-exercises/python-app
cd ~/docker-exercises/python-app

# 创建简单的 Python 应用
cat > app.py << 'EOF'
#!/usr/bin/env python3
"""
用于 Docker 练习的简单 Flask web 应用
"""
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'message': 'Hello from Docker!',
        'hostname': socket.gethostname(),
        'environment': os.environ.get('ENV', 'development')
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# 创建 requirements
cat > requirements.txt << 'EOF'
flask==3.0.0
gunicorn==21.2.0
EOF
```

### Step 1.2: 编写你的第一个 Dockerfile

```dockerfile
# 创建 Dockerfile
cat > Dockerfile << 'EOF'
# 使用官方 Python 运行时作为基础
FROM python:3.11

# 设置工作目录
WORKDIR /app

# 先复制 requirements（layer 缓存！）
COPY requirements.txt .

# 安装依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY app.py .

# 暴露端口
EXPOSE 5000

# 运行应用
CMD ["python", "app.py"]
EOF
```

### Step 1.3: 构建并运行

```bash
# 构建 image
docker build -t my-python-app:v1 .

# 发生了什么？
# 1. 下载 Python 3.11 基础 image
# 2. 创建工作目录
# 3. 安装依赖
# 4. 复制应用代码
# 5. 将 image 标记为 my-python-app:v1

# 检查 image
docker images | grep my-python-app

# 运行 container
docker run -d -p 5000:5000 --name pyapp my-python-app:v1

# 测试
curl http://localhost:5000
curl http://localhost:5000/health

# 检查日志
docker logs pyapp
```

**Questions**:
1. 你的 image 有多大？（`docker images my-python-app:v1`）
2. 如果修改 `app.py` 并重新构建，会发生什么？
3. 哪些 layer 被缓存了？

✅ **Checkpoint**: 你可以在 Docker 中构建并运行自定义 Python 应用。

---

## Part 2: 优化 Image

### Step 2.1: 使用更小的基础 Image

```dockerfile
# 创建优化的 Dockerfile
cat > Dockerfile.slim << 'EOF'
# 使用 slim 变体（小得多！）
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

```bash
# 使用 slim 基础构建
docker build -f Dockerfile.slim -t my-python-app:v2-slim .

# 比较大小
docker images | grep my-python-app

# 输出应显示显著的大小差异：
# my-python-app  v1       <timestamp>  1.02GB
# my-python-app  v2-slim  <timestamp>  182MB
```

### Step 2.2: 多阶段构建

```dockerfile
# 创建多阶段 Dockerfile
cat > Dockerfile.multistage << 'EOF'
# 阶段 1：构建
FROM python:3.11 AS builder

WORKDIR /app

# 将依赖安装到本地目录
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# 阶段 2：运行时
FROM python:3.11-slim

WORKDIR /app

# 仅从构建阶段复制已安装的包
COPY --from=builder /root/.local /root/.local

# 复制应用
COPY app.py .

# 确保脚本在 PATH 中
ENV PATH=/root/.local/bin:$PATH

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

```bash
# 构建多阶段 image
docker build -f Dockerfile.multistage -t my-python-app:v3-multi .

# 比较所有版本
docker images | grep my-python-app

# 运行并验证
docker run -d -p 5001:5000 --name pyapp-multi my-python-app:v3-multi
curl http://localhost:5001
```

### Step 2.3: Layer 缓存优化

```dockerfile
# 创建缓存优化的 Dockerfile
cat > Dockerfile.optimized << 'EOF'
FROM python:3.11-slim

WORKDIR /app

# 首先复制并安装依赖
# 除非 requirements 变更，否则此 layer 将被缓存
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 最后复制源代码
# 这经常变化但不会使依赖缓存失效
COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
```

```bash
# 首次构建
time docker build -f Dockerfile.optimized -t my-python-app:v4 .

# 修改 app.py
echo "# Comment added" >> app.py

# 重新构建 — 注意速度有多快！
time docker build -f Dockerfile.optimized -t my-python-app:v4 .

# 依赖被缓存了！
```

**Challenge**: 修改 `requirements.txt` 并重新构建。构建时间发生了什么变化？

✅ **Checkpoint**: 你可以使用 slim 基础和多阶段构建优化 image。

---

## Part 3: ML 应用 Image

### Step 3.1: 简单的 Model Serving 应用

```bash
# 创建 ML app 目录
mkdir -p ~/docker-exercises/ml-app
cd ~/docker-exercises/ml-app

# 创建模型 serving 应用
cat > serve.py << 'EOF'
"""
使用 Flask 的简单 ML model serving
"""
from flask import Flask, request, jsonify
import torch
import torchvision.models as models
import torchvision.transforms as transforms
from PIL import Image
import io

app = Flask(__name__)

# 加载模型
print("Loading model...")
model = models.resnet18(pretrained=True)
model.eval()
print("Model loaded successfully!")

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

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'model': 'resnet18'})

@app.route('/predict', methods=['POST'])
def predict():
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400

    file = request.files['file']
    img_bytes = file.read()
    img = Image.open(io.BytesIO(img_bytes))

    # 预处理和预测
    img_tensor = transform(img).unsqueeze(0)

    with torch.no_grad():
        output = model(img_tensor)
        probabilities = torch.nn.functional.softmax(output[0], dim=0)
        top5_prob, top5_catid = torch.topk(probabilities, 5)

    results = []
    for i in range(5):
        results.append({
            'class_id': top5_catid[i].item(),
            'probability': top5_prob[i].item()
        })

    return jsonify({'predictions': results})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF

# 创建 ML 应用的 requirements
cat > requirements.txt << 'EOF'
flask==3.0.0
torch==2.1.0
torchvision==0.16.0
pillow==10.1.0
gunicorn==21.2.0
EOF
```

### Step 3.2: ML 应用的 Dockerfile

```dockerfile
cat > Dockerfile << 'EOF'
# ML 应用的多阶段构建
FROM python:3.11 AS builder

WORKDIR /app

# 安装依赖
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# 运行时阶段
FROM python:3.11-slim

# 安装 PyTorch 的系统依赖
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 复制已安装的包
COPY --from=builder /root/.local /root/.local

# 复制应用
COPY serve.py .

# 更新 PATH
ENV PATH=/root/.local/bin:$PATH

EXPOSE 8000

# 使用 gunicorn 用于生产
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "--timeout", "120", "serve:app"]
EOF
```

```bash
# 构建 ML image（这将需要几分钟）
docker build -t ml-serve:v1 .

# 检查大小
docker images ml-serve:v1

# 运行 ML 服务器
docker run -d -p 8000:8000 --name ml-server ml-serve:v1

# 等待模型加载
sleep 10

# 检查健康状态
curl http://localhost:8000/health

# 测试预测（你需要一个图像文件）
# curl -X POST -F "file=@test_image.jpg" http://localhost:8000/predict
```

### Step 3.3: 优化 ML Image 大小

```dockerfile
# 创建优化的 ML Dockerfile
cat > Dockerfile.optimized << 'EOF'
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    libgomp1 \
    && rm -rf /var/lib/apt/lists/*

# 安装仅 CPU 版本的 PyTorch（小得多！）
COPY requirements.txt .
RUN pip install --no-cache-dir \
    flask==3.0.0 \
    pillow==10.1.0 \
    gunicorn==21.2.0 \
    torch==2.1.0+cpu \
    torchvision==0.16.0+cpu \
    --index-url https://download.pytorch.org/whl/cpu

COPY serve.py .

EXPOSE 8000

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "--timeout", "120", "serve:app"]
EOF
```

```bash
# 构建优化版本
docker build -f Dockerfile.optimized -t ml-serve:v2-cpu .

# 比较大小
docker images | grep ml-serve

# CPU 版本应该显著更小！
```

✅ **Checkpoint**: 你可以使用优化 image containerize ML 应用。

---

## Part 4: Dockerfile 指令深入解析

### Step 4.1: 理解 Layers

```dockerfile
# 创建包含多个 layer 的示例
cat > Dockerfile.layers << 'EOF'
FROM python:3.11-slim

# 每个 RUN 创建一个新 layer
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get install -y vim

# 更好：合并命令
# RUN apt-get update && \
#     apt-get install -y curl wget vim && \
#     rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY app.py .

CMD ["python", "app.py"]
EOF
```

```bash
# 构建并检查 layers
docker build -f Dockerfile.layers -t layer-demo .

# 查看 layer 历史
docker history layer-demo

# 注意每个 RUN 命令都创建一个 layer
```

### Step 4.2: COPY vs ADD

```bash
# 创建测试文件
mkdir -p ~/docker-exercises/copy-test
cd ~/docker-exercises/copy-test

echo "Hello" > file.txt
tar czf archive.tar.gz file.txt
```

```dockerfile
cat > Dockerfile.copy << 'EOF'
FROM alpine:latest

WORKDIR /app

# COPY：简单的文件复制
COPY file.txt /app/

# ADD：可以自动解压归档文件
ADD archive.tar.gz /app/extracted/

CMD ["sh"]
EOF
```

```bash
# 构建并检查
docker build -f Dockerfile.copy -t copy-demo .

# 运行并探索
docker run --rm -it copy-demo sh
# 在 container 内部：
# ls -la /app
# ls -la /app/extracted
# exit
```

**Best Practice**: 除非你特别需要 `ADD` 的 tar 解压功能，否则使用 `COPY`。

### Step 4.3: ARG and ENV

```dockerfile
cat > Dockerfile.args << 'EOF'
FROM python:3.11-slim

# 构建时变量（仅在构建期间）
ARG PYTHON_VERSION=3.11
ARG APP_ENV=development

# 运行时变量（在 container 中可用）
ENV APP_ENV=${APP_ENV}
ENV DEBUG=true

WORKDIR /app

RUN echo "Building for environment: ${APP_ENV}"
RUN echo "Python version: ${PYTHON_VERSION}"

COPY app.py .

CMD ["python", "app.py"]
EOF
```

```bash
# 使用默认参数构建
docker build -f Dockerfile.args -t arg-demo:dev .

# 使用自定义参数构建
docker build -f Dockerfile.args -t arg-demo:prod \
  --build-arg APP_ENV=production \
  --build-arg PYTHON_VERSION=3.11 .

# 运行并检查环境变量
docker run --rm arg-demo:dev python -c "import os; print(os.environ.get('APP_ENV'))"
docker run --rm arg-demo:prod python -c "import os; print(os.environ.get('APP_ENV'))"
```

### Step 4.4: USER 指令（安全）

```dockerfile
cat > Dockerfile.user << 'EOF'
FROM python:3.11-slim

WORKDIR /app

# 创建非 root 用户
RUN groupadd -r appuser && useradd -r -g appuser appuser

# 以 root 身份安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用并设置所有权
COPY --chown=appuser:appuser app.py .

# 切换到非 root 用户
USER appuser

CMD ["python", "app.py"]
EOF
```

```bash
# 构建
docker build -f Dockerfile.user -t secure-app .

# 运行并检查用户
docker run --rm secure-app whoami
# 输出：appuser（不是 root！）

# 验证安全性
docker run --rm secure-app id
# 输出显示非 root UID/GID
```

✅ **Checkpoint**: 你理解 Dockerfile 指令和 best practices。

---

## Part 5: Build Context 和 .dockerignore

### Step 5.1: 理解 Build Context

```bash
# 创建包含多个文件的项目
mkdir -p ~/docker-exercises/build-context
cd ~/docker-exercises/build-context

# 创建一些文件
echo "print('app')" > app.py
echo "flask" > requirements.txt

# 创建大型的不必要文件
dd if=/dev/zero of=large_file.bin bs=1M count=100
mkdir -p data
dd if=/dev/zero of=data/dataset.bin bs=1M count=500

# 简单的 Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
EOF

# 构建 — 注意它会将所有文件发送到 daemon
docker build -t context-demo .
# 输出：Sending build context to Docker daemon  614.4MB
```

### Step 5.2: 创建 .dockerignore

```bash
# 创建 .dockerignore 文件
cat > .dockerignore << 'EOF'
# 忽略大型数据文件
*.bin
data/
datasets/

# 忽略 git
.git/
.gitignore

# 忽略 Python 缓存
__pycache__/
*.pyc
*.pyo
*.pyd
.Python

# 忽略虚拟环境
venv/
env/
ENV/

# 忽略 IDE 文件
.vscode/
.idea/
*.swp
*.swo

# 忽略操作系统文件
.DS_Store
Thumbs.db

# 忽略文档
docs/
*.md
!README.md

# 忽略测试（如果 image 中不需要）
tests/
*.test.py
EOF

# 重新构建
docker build -t context-demo:v2 .
# 输出：Sending build context to Docker daemon  3.072kB
# 好多了！
```

**Challenge**: 创建一个包含测试、文档和数据的 Python 项目。编写一个 `.dockerignore`，只包含生产文件。

✅ **Checkpoint**: 你可以使用 .dockerignore 优化 build context。

---

## Part 6: Image 标记和版本管理

### Step 6.1: 标记策略

```bash
# 使用多个标签构建
docker build -t myapp:1.0.0 .
docker build -t myapp:1.0 .
docker build -t myapp:1 .
docker build -t myapp:latest .

# 或者标记现有 image
docker tag myapp:1.0.0 myapp:latest
docker tag myapp:1.0.0 myapp:stable

# 语义版本控制
docker build -t myapp:1.0.0-alpha .
docker build -t myapp:1.0.0-beta .
docker build -t myapp:1.0.0-rc1 .
docker build -t myapp:1.0.0 .

# 包含 git commit
GIT_COMMIT=$(git rev-parse --short HEAD)
docker build -t myapp:${GIT_COMMIT} .
docker build -t myapp:dev-${GIT_COMMIT} .
```

### Step 6.2: 标签的最佳实践

```bash
# CI/CD 的良好标记策略
VERSION="2.1.3"
BUILD_DATE=$(date +%Y%m%d)
GIT_SHA=$(git rev-parse --short HEAD)

# 使用全面的标签构建
docker build \
  -t myapp:${VERSION} \
  -t myapp:${VERSION}-${BUILD_DATE} \
  -t myapp:${VERSION}-${GIT_SHA} \
  -t myapp:latest \
  .

# 列出所有标签
docker images myapp
```

**Tagging Best Practices**:
1. **始终使用特定版本标签** (1.2.3)
2. **使用语义版本控制** (MAJOR.MINOR.PATCH)
3. **标记 latest** 用于最新稳定版
4. **避免在生产中使用 latest**（固定版本！）
5. **在需要时包含构建元数据**

✅ **Checkpoint**: 你可以正确地版本管理和标记 image。

---

## Part 7: 使用 Registry

### Step 7.1: Docker Hub

```bash
# 登录 Docker Hub
docker login

# 标记为 Docker Hub（用户名/image:tag）
docker tag myapp:1.0.0 yourusername/myapp:1.0.0
docker tag myapp:1.0.0 yourusername/myapp:latest

# 推送到 Docker Hub
docker push yourusername/myapp:1.0.0
docker push yourusername/myapp:latest

# 从 Docker Hub 拉取
docker pull yourusername/myapp:1.0.0

# 登出
docker logout
```

### Step 7.2: 私有 Registry（本地）

```bash
# 运行本地 registry
docker run -d -p 5000:5000 --name registry registry:2

# 标记为本地 registry
docker tag myapp:1.0.0 localhost:5000/myapp:1.0.0

# 推送到本地 registry
docker push localhost:5000/myapp:1.0.0

# 从本地 registry 拉取
docker pull localhost:5000/myapp:1.0.0

# 列出 registry 中的 image
curl http://localhost:5000/v2/_catalog
curl http://localhost:5000/v2/myapp/tags/list
```

### Step 7.3: Registry 认证

```bash
# 创建 htpasswd 文件
docker run --rm \
  --entrypoint htpasswd \
  httpd:2 -Bbn testuser testpass > htpasswd

# 运行带认证的 registry
docker run -d -p 5001:5000 \
  --name secure-registry \
  -v $(pwd)/htpasswd:/auth/htpasswd \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e "REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd" \
  registry:2

# 登录安全 registry
docker login localhost:5001
# Username: testuser
# Password: testpass

# 推送到安全 registry
docker tag myapp:1.0.0 localhost:5001/myapp:1.0.0
docker push localhost:5001/myapp:1.0.0
```

✅ **Checkpoint**: 你可以使用 Docker registry。

---

## Part 8: 调试构建问题

### Step 8.1: 常见构建错误

```dockerfile
# 创建有问题的 Dockerfile
cat > Dockerfile.broken << 'EOF'
FROM python:3.11-slim

WORKDIR /app

# 问题 1：文件未找到
COPY nonexistent.txt .

# 问题 2：命令执行失败
RUN pip install fake-package-that-doesnt-exist

# 问题 3：权限被拒绝
COPY app.py /root/protected/

CMD ["python", "app.py"]
EOF
```

```bash
# 尝试构建 — 会失败
docker build -f Dockerfile.broken -t broken-app .

# 调试技巧 1：构建到正常工作的阶段
# 注释掉失败的行并递增构建

# 调试技巧 2：检查 build context
ls -la

# 调试技巧 3：使用 --no-cache
docker build --no-cache -f Dockerfile.broken -t broken-app .
```

### Step 8.2: 交互式调试

```bash
# 构建到某个点
cat > Dockerfile.debug << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

# 在此处停止以进行调试
CMD ["/bin/bash"]
EOF

# 构建
docker build -f Dockerfile.debug -t debug-app .

# 以交互方式运行
docker run --rm -it debug-app bash

# 在 container 内部，测试命令：
# pwd
# ls -la
# python --version
# pip list
# exit
```

### Step 8.3: 检查构建 Layers

```bash
# 使用详细输出构建
docker build --progress=plain -t myapp .

# 检查特定 layer
docker history myapp

# 获取详细信息
docker inspect myapp

# 找出哪个 layer 增加了大小
docker history myapp --human --no-trunc
```

**Debugging Checklist**:
- [ ] 文件在 build context 中吗？
- [ ] 检查 .dockerignore 是否排除了需要的文件
- [ ] 验证基础 image 存在且可访问
- [ ] 检查下载的网络访问
- [ ] 检查文件权限
- [ ] 确保 Dockerfile 语法正确
- [ ] 使用 --no-cache 避免过期 layer

✅ **Checkpoint**: 你可以调试 Dockerfile 构建问题。

---

## Part 9: 高级模式

### Step 9.1: Dockerfile 中的健康检查

```dockerfile
cat > Dockerfile.health << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

# 添加健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:5000/health')" || exit 1

CMD ["python", "app.py"]
EOF
```

```bash
# 构建并运行
docker build -f Dockerfile.health -t health-app .
docker run -d -p 5000:5000 --name health-demo health-app

# 检查健康状态
docker ps  # 显示健康状态
docker inspect health-demo | grep -A 10 Health
```

### Step 9.2: 构建时密钥

```bash
# 创建密钥文件（API key, 密码等）
echo "super-secret-api-key" > api_key.txt

# 使用构建时密钥的 Dockerfile
cat > Dockerfile.secrets << 'EOF'
# syntax=docker/dockerfile:1.4

FROM python:3.11-slim

WORKDIR /app

# 在构建过程中使用密钥（不存储在 image 中！）
RUN --mount=type=secret,id=api_key \
    API_KEY=$(cat /run/secrets/api_key) && \
    echo "Using API key: ${API_KEY:0:5}..." && \
    # 使用 API key 进行构建操作
    echo "Build completed with secret"

COPY app.py .

CMD ["python", "app.py"]
EOF

# 使用密钥构建
docker build --secret id=api_key,src=api_key.txt \
  -f Dockerfile.secrets -t secret-app .

# 密钥不在 image 中！
docker history secret-app
```

### Step 9.3: 多阶段构建中的目标阶段

```dockerfile
cat > Dockerfile.targets << 'EOF'
# 基础阶段
FROM python:3.11-slim AS base
WORKDIR /app
COPY requirements.txt .

# 开发阶段
FROM base AS development
RUN pip install -r requirements.txt
RUN pip install pytest black flake8
COPY . .
CMD ["python", "app.py"]

# 测试阶段
FROM development AS testing
RUN pytest tests/
CMD ["pytest"]

# 生产阶段
FROM base AS production
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["gunicorn", "app:app"]
EOF
```

```bash
# 构建特定阶段
docker build --target development -t myapp:dev -f Dockerfile.targets .
docker build --target production -t myapp:prod -f Dockerfile.targets .
docker build --target testing -t myapp:test -f Dockerfile.targets .

# 每个阶段有不同的工具和配置！
docker run --rm myapp:dev python --version
docker run --rm myapp:prod gunicorn --version
```

✅ **Checkpoint**: 你可以使用高级 Dockerfile 模式。

---

## Part 10: 生产就绪的 Dockerfile

### Step 10.1: 完整的生产示例

```dockerfile
cat > Dockerfile.production << 'EOF'
# syntax=docker/dockerfile:1.4

# 构建阶段
FROM python:3.11 AS builder

WORKDIR /app

# 安装构建依赖
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖
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
RUN groupadd -r appuser && \
    useradd -r -g appuser -u 1000 appuser

WORKDIR /app

# 从构建阶段复制 Python 包
COPY --from=builder /root/.local /root/.local

# 以正确的所有权复制应用
COPY --chown=appuser:appuser . .

# 更新 PATH
ENV PATH=/root/.local/bin:$PATH \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# 切换到非 root 用户
USER appuser

EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 使用生产服务器
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "--timeout", "120", "serve:app"]
EOF
```

**Production Checklist**:
- [x] 多阶段构建
- [x] 最小基础 image (slim)
- [x] 非 root 用户
- [x] 健康检查
- [x] 生产服务器 (gunicorn)
- [x] 环境变量
- [x] Layer 中无缓存
- [x] 正确的信号处理
- [x] 安全 best practices

✅ **Checkpoint**: 你可以创建生产就绪的 Dockerfile。

---

## Part 11: 实际挑战

### Challenge 1: 优化臃肿的 Image

给你这个 Dockerfile：

```dockerfile
FROM ubuntu:latest

RUN apt-get update
RUN apt-get install -y python3
RUN apt-get install -y python3-pip
RUN apt-get install -y curl wget vim git

WORKDIR /app

COPY . .

RUN pip3 install flask
RUN pip3 install requests
RUN pip3 install numpy

CMD ["python3", "app.py"]
```

**Task**: 优化到 < 200MB

**Solution**:
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# 合并 RUN 命令并清理
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# 使用 requirements.txt
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
```

### Challenge 2: 多服务应用

**Task**: 创建一个可用于以下场景的 Dockerfile：
1. Web 服务器（生产）
2. Worker 进程（后台任务）
3. 数据库迁移

**Solution**:
```dockerfile
FROM python:3.11-slim AS base

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# 默认：web 服务器
CMD ["gunicorn", "app:app"]

# 覆盖方式：
# docker run myapp python worker.py
# docker run myapp python migrate.py
```

### Challenge 3: 构建时配置

**Task**: 创建一个接受以下参数的 Dockerfile：
- 环境 (dev/staging/prod)
- 功能开关
- 版本号

**Solution**:
```dockerfile
ARG ENV=production
ARG ENABLE_FEATURE_X=false
ARG VERSION=1.0.0

FROM python:3.11-slim

LABEL version="${VERSION}"
LABEL environment="${ENV}"

ENV APP_ENV=${ENV} \
    FEATURE_X=${ENABLE_FEATURE_X} \
    VERSION=${VERSION}

WORKDIR /app

COPY requirements.${ENV}.txt ./requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

```bash
# 为不同环境构建
docker build --build-arg ENV=development --build-arg VERSION=1.2.3 -t myapp:dev .
docker build --build-arg ENV=production --build-arg VERSION=1.2.3 -t myapp:prod .
```

---

## Part 12: 测试你的 Image

### Step 12.1: 创建测试脚本

```bash
cat > test_image.sh << 'EOF'
#!/bin/bash

IMAGE=$1

echo "Testing image: ${IMAGE}"

# 测试 1：Image 存在
echo "Test 1: Checking image exists..."
docker images ${IMAGE} | grep -q ${IMAGE}
if [ $? -eq 0 ]; then
    echo "✓ Image exists"
else
    echo "✗ Image not found"
    exit 1
fi

# 测试 2：Container 启动
echo "Test 2: Checking container starts..."
CONTAINER=$(docker run -d ${IMAGE})
sleep 2
docker ps | grep -q ${CONTAINER}
if [ $? -eq 0 ]; then
    echo "✓ Container started"
else
    echo "✗ Container failed to start"
    docker logs ${CONTAINER}
    exit 1
fi

# 测试 3：健康检查
echo "Test 3: Checking health endpoint..."
PORT=$(docker port ${CONTAINER} | head -1 | cut -d: -f2)
curl -f http://localhost:${PORT}/health
if [ $? -eq 0 ]; then
    echo "✓ Health check passed"
else
    echo "✗ Health check failed"
fi

# 清理
docker rm -f ${CONTAINER}

echo "All tests passed!"
EOF

chmod +x test_image.sh

# 运行测试
./test_image.sh my-python-app:v1
```

### Step 12.2: 自动化测试

```bash
# 创建测试矩阵
cat > test_matrix.sh << 'EOF'
#!/bin/bash

IMAGES=(
    "my-python-app:v1"
    "my-python-app:v2-slim"
    "my-python-app:v3-multi"
    "ml-serve:v1"
)

for image in "${IMAGES[@]}"; do
    echo "========================================="
    echo "Testing: ${image}"
    echo "========================================="
    ./test_image.sh ${image}
    if [ $? -ne 0 ]; then
        echo "FAILED: ${image}"
        exit 1
    fi
done

echo "All images passed tests!"
EOF

chmod +x test_matrix.sh
./test_matrix.sh
```

✅ **Checkpoint**: 你可以测试你的 Docker image。

---

## Validation and Reflection

### Validation Script

创建全面的验证：

```bash
#!/bin/bash
# validate_exercise.sh

echo "Docker Image Building Exercise Validation"
echo "==========================================="

# 检查所有 image 是否已构建
REQUIRED_IMAGES=(
    "my-python-app:v1"
    "my-python-app:v2-slim"
    "my-python-app:v3-multi"
    "ml-serve:v1"
)

for img in "${REQUIRED_IMAGES[@]}"; do
    if docker images | grep -q "${img%%:*}.*${img##*:}"; then
        echo "✓ ${img}"
    else
        echo "✗ ${img} - MISSING"
    fi
done

# 检查 image 大小
echo ""
echo "Image Size Analysis:"
echo "-------------------"
docker images | grep "my-python-app\|ml-serve" | awk '{print $1":"$2" - "$7$8}'

# 检查 .dockerignore
if [ -f ".dockerignore" ]; then
    echo "✓ .dockerignore file exists"
else
    echo "✗ .dockerignore file missing"
fi

echo ""
echo "Validation complete!"
```

### Reflection Questions

1. **`COPY` 和 `ADD` 有什么区别？**
   - 你的答案：

2. **为什么要使用多阶段构建？**
   - 你的答案：

3. **如何最小化 image 大小？**
   - 你的答案：

4. **什么时候使用 `--no-cache-dir` 与 pip？**
   - 你的答案：

5. **为什么 layer 缓存很重要？**
   - 你的答案：

6. **非 root 用户有什么好处？**
   - 你的答案：

---

## Summary

**What You Accomplished**:
✅ 从 Dockerfile 创建了自定义 Docker image
✅ 使用多阶段构建优化了 image
✅ Containerized Python 和 ML 应用
✅ 掌握了 Dockerfile 指令
✅ 实现了安全 best practices
✅ 正确标记和版本管理了 image
✅ 使用了 Docker registry
✅ 调试了构建问题
✅ 创建了生产就绪的 Dockerfile

**Key Concepts Mastered**:
- **Dockerfile 语法**和 best practices
- **Layer 缓存**和构建优化
- **多阶段构建**用于更小的 image
- **安全实践**（非 root 用户、最小基础）
- **Image 标记**和版本管理策略
- **Registry 操作**（push/pull）
- **Build context**和 .dockerignore
- **健康检查**和生产模式

**Image Size Comparison**:
```
Full Python:        1.02 GB
Slim Python:        182 MB
Multi-stage:        ~150 MB
Optimized ML (CPU): ~400 MB
```

---

## Next Steps

**Practice More**:
1. 为不同类型的应用构建 image
2. 尝试不同的基础 image
3. 优化现有项目
4. 为你的团队创建可重用的基础 image

**Continue Learning**:
- **Exercise 03**: Docker Compose Applications
- **Exercise 04**: Networking and Service Discovery
- **Exercise 05**: Volume Management and Persistence

---

**Exercise Version**: 1.0
**Last Updated**: October 2025
**Estimated Time**: 3-4 小时
**Difficulty**: 初级到中级
