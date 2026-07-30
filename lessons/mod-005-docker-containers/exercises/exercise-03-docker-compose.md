# Exercise 03: Docker Compose Applications

## Exercise Overview

**Objective**: 掌握 Docker Compose，用于编排多 container 应用，包括数据库、web 服务、缓存层和 ML 推理服务。

**Difficulty**: 中级
**Estimated Time**: 3-4 小时
**Prerequisites**:
- Exercise 01 (Container Operations)
- Exercise 02 (Building Custom Images)
- Lecture 03 (Docker Compose)

**What You'll Learn**:
- 编写 docker-compose.yml 文件
- 多 container 应用架构
- 服务依赖和启动顺序
- 环境配置
- Compose 中的 volume 管理
- 网络隔离和服务发现
- 扩展服务
- 生产部署模式
- 带支持服务的 ML serving

---

## Part 1: 你的第一个 Compose 应用

### Step 1.1: 简单的 Web + 数据库栈

```bash
# 创建项目目录
mkdir -p ~/docker-exercises/compose-basics
cd ~/docker-exercises/compose-basics

# 创建简单的 Flask 应用
cat > app.py << 'EOF'
from flask import Flask, jsonify
import psycopg2
import os

app = Flask(__name__)

def get_db_connection():
    return psycopg2.connect(
        host=os.environ.get('DB_HOST', 'db'),
        database=os.environ.get('DB_NAME', 'myapp'),
        user=os.environ.get('DB_USER', 'postgres'),
        password=os.environ.get('DB_PASSWORD', 'postgres')
    )

@app.route('/')
def home():
    return jsonify({'message': 'Hello from Docker Compose!'})

@app.route('/db-test')
def db_test():
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute('SELECT version();')
        version = cur.fetchone()[0]
        cur.close()
        conn.close()
        return jsonify({'status': 'connected', 'db_version': version})
    except Exception as e:
        return jsonify({'status': 'error', 'message': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# 创建 requirements
cat > requirements.txt << 'EOF'
flask==3.0.0
psycopg2-binary==2.9.9
EOF

# 创建 Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

CMD ["python", "app.py"]
EOF
```

### Step 1.2: 创建 docker-compose.yml

```yaml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=db
      - DB_NAME=myapp
      - DB_USER=postgres
      - DB_PASSWORD=postgres
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
EOF
```

### Step 1.3: 运行你的第一个 Compose 栈

```bash
# 启动所有服务
docker compose up

# 在另一个终端中，测试应用
curl http://localhost:5000
curl http://localhost:5000/db-test

# 查看日志
docker compose logs
docker compose logs web
docker compose logs db

# 使用 Ctrl+C 停止，然后删除
docker compose down

# 以分离模式运行
docker compose up -d

# 检查状态
docker compose ps

# 以跟踪模式查看日志
docker compose logs -f

# 停止并删除所有内容，包括 volume
docker compose down -v
```

**Questions**:
1. 如果在数据库之前启动 web 服务，会发生什么？
2. Web 服务如何解析主机名 "db"？
3. 运行 `docker compose down` 时数据会发生什么？
4. 那么 `docker compose down -v` 呢？

✅ **Checkpoint**: 你可以使用 Docker Compose 运行多 container 应用。

---

## Part 2: 服务依赖和健康检查

### Step 2.1: 改进启动顺序

```yaml
cat > docker-compose-v2.yml << 'EOF'
version: '3.8'

services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=db
      - DB_NAME=myapp
      - DB_USER=postgres
      - DB_PASSWORD=postgres
    depends_on:
      db:
        condition: service_healthy
    restart: on-failure

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
EOF
```

```bash
# 使用健康检查运行
docker compose -f docker-compose-v2.yml up

# 观察服务变为健康状态
docker compose -f docker-compose-v2.yml ps
```

### Step 2.2: 添加初始化脚本

```bash
# 创建数据库初始化脚本
mkdir -p init-scripts

cat > init-scripts/01-create-tables.sql << 'EOF'
CREATE TABLE IF NOT EXISTS visits (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(50)
);

CREATE INDEX idx_timestamp ON visits(timestamp);

INSERT INTO visits (ip_address) VALUES ('127.0.0.1');
EOF
```

```yaml
cat > docker-compose-v3.yml << 'EOF'
version: '3.8'

services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=db
      - DB_NAME=myapp
      - DB_USER=postgres
      - DB_PASSWORD=postgres
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
EOF
```

```bash
# 使用初始化脚本启动
docker compose -f docker-compose-v3.yml up -d

# 验证表已创建
docker compose -f docker-compose-v3.yml exec db \
  psql -U postgres -d myapp -c "SELECT * FROM visits;"
```

✅ **Checkpoint**: 你可以管理服务依赖和初始化。

---

## Part 3: 环境配置

### Step 3.1: 使用 .env 文件

```bash
# 创建环境文件
cat > .env << 'EOF'
# 数据库配置
POSTGRES_DB=myapp
POSTGRES_USER=appuser
POSTGRES_PASSWORD=securepassword123

# 应用配置
FLASK_ENV=development
DEBUG=true
SECRET_KEY=dev-secret-key

# 端口
WEB_PORT=5000
DB_PORT=5432
EOF

# 创建 .env.production
cat > .env.production << 'EOF'
POSTGRES_DB=myapp_prod
POSTGRES_USER=produser
POSTGRES_PASSWORD=super-secure-production-password

FLASK_ENV=production
DEBUG=false
SECRET_KEY=production-secret-key-change-me

WEB_PORT=8000
DB_PORT=5432
EOF
```

### Step 3.2: 在 Compose 中使用变量

```yaml
cat > docker-compose-env.yml << 'EOF'
version: '3.8'

services:
  web:
    build: .
    ports:
      - "${WEB_PORT:-5000}:5000"
    environment:
      - DB_HOST=db
      - DB_NAME=${POSTGRES_DB}
      - DB_USER=${POSTGRES_USER}
      - DB_PASSWORD=${POSTGRES_PASSWORD}
      - FLASK_ENV=${FLASK_ENV:-development}
      - DEBUG=${DEBUG:-false}
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=${POSTGRES_DB}
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    ports:
      - "${DB_PORT:-5432}:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
EOF
```

```bash
# 使用默认 .env 运行
docker compose -f docker-compose-env.yml up -d

# 使用生产 env 运行
docker compose -f docker-compose-env.yml --env-file .env.production up -d

# 检查使用了哪些变量
docker compose -f docker-compose-env.yml config
```

### Step 3.3: 环境特定的覆盖

```yaml
# 创建基础配置
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    build: .
    environment:
      - DB_HOST=db
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
EOF

# 创建开发环境覆盖
cat > docker-compose.override.yml << 'EOF'
version: '3.8'

services:
  web:
    ports:
      - "5000:5000"
    volumes:
      - .:/app  # 挂载代码以实现热重载
    environment:
      - FLASK_ENV=development
      - DEBUG=true

  db:
    ports:
      - "5432:5432"  # 在开发环境中暴露数据库
EOF

# 创建生产环境覆盖
cat > docker-compose.prod.yml << 'EOF'
version: '3.8'

services:
  web:
    ports:
      - "8000:5000"
    environment:
      - FLASK_ENV=production
      - DEBUG=false
    restart: always
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M

  db:
    restart: always
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
EOF
```

```bash
# 开发环境（自动使用 base + override）
docker compose up -d

# 生产环境（显式覆盖）
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

✅ **Checkpoint**: 你可以管理环境特定的配置。

---

## Part 4: 添加更多服务

### Step 4.1: 完整的 Web 栈 (Web + DB + Redis + Nginx)

```bash
# 创建新项目
mkdir -p ~/docker-exercises/full-stack
cd ~/docker-exercises/full-stack

# 创建带 Redis 缓存的 Flask 应用
cat > app.py << 'EOF'
from flask import Flask, jsonify
import redis
import psycopg2
import os
import time

app = Flask(__name__)

# Redis 连接
cache = redis.Redis(
    host=os.environ.get('REDIS_HOST', 'redis'),
    port=6379,
    decode_responses=True
)

# 数据库连接
def get_db():
    return psycopg2.connect(
        host=os.environ.get('DB_HOST', 'db'),
        database=os.environ.get('DB_NAME', 'myapp'),
        user=os.environ.get('DB_USER', 'postgres'),
        password=os.environ.get('DB_PASSWORD', 'postgres')
    )

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

@app.route('/')
def home():
    # 在 Redis 中递增页面浏览次数
    cache.incr('page_views')
    views = cache.get('page_views')
    return jsonify({
        'message': 'Hello from Full Stack!',
        'page_views': views
    })

@app.route('/slow')
def slow_endpoint():
    # 先检查缓存
    cached = cache.get('slow_result')
    if cached:
        return jsonify({'result': cached, 'cached': True})

    # 模拟慢操作
    time.sleep(3)
    result = "Computed expensive result"

    # 缓存 60 秒
    cache.setex('slow_result', 60, result)

    return jsonify({'result': result, 'cached': False})

@app.route('/stats')
def stats():
    conn = get_db()
    cur = conn.cursor()
    cur.execute('SELECT COUNT(*) FROM visits;')
    visit_count = cur.fetchone()[0]
    cur.close()
    conn.close()

    return jsonify({
        'db_visits': visit_count,
        'cache_views': cache.get('page_views')
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Requirements
cat > requirements.txt << 'EOF'
flask==3.0.0
redis==5.0.1
psycopg2-binary==2.9.9
EOF

# Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import requests; requests.get('http://localhost:5000/health')" || exit 1

CMD ["python", "app.py"]
EOF

# Nginx 配置
mkdir -p nginx

cat > nginx/nginx.conf << 'EOF'
upstream web_backend {
    server web:5000;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://web_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /health {
        access_log off;
        proxy_pass http://web_backend;
    }
}
EOF
```

### Step 4.2: 完整的 Docker Compose

```yaml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      web:
        condition: service_healthy
    restart: always

  web:
    build: .
    environment:
      - DB_HOST=db
      - DB_NAME=myapp
      - DB_USER=postgres
      - DB_PASSWORD=postgres
      - REDIS_HOST=redis
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: always

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
  redis_data:
EOF
```

```bash
# 创建 init 脚本
mkdir -p init-scripts
cat > init-scripts/01-init.sql << 'EOF'
CREATE TABLE visits (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
INSERT INTO visits (timestamp) VALUES (NOW());
EOF

# 启动完整栈
docker compose up -d

# 等待服务健康
sleep 15

# 测试应用
curl http://localhost/
curl http://localhost/stats
curl http://localhost/slow  # 第一次调用 - 慢
curl http://localhost/slow  # 第二次调用 - 已缓存！

# 检查日志
docker compose logs -f web

# 检查所有服务
docker compose ps
```

✅ **Checkpoint**: 你可以编排包含多个服务的完整 web 栈。

---

## Part 5: ML 推理栈

### Step 5.1: 带支持服务的 ML Model Serving

```bash
# 创建 ML 项目
mkdir -p ~/docker-exercises/ml-stack
cd ~/docker-exercises/ml-stack

# 创建模型 serving 应用
cat > serve.py << 'EOF'
from flask import Flask, request, jsonify
import torch
import torchvision.models as models
import torchvision.transforms as transforms
from PIL import Image
import io
import redis
import json
import hashlib
import psycopg2
import os
from datetime import datetime

app = Flask(__name__)

# 加载模型
print("Loading model...")
model = models.resnet18(pretrained=True)
model.eval()
print("Model loaded!")

# Redis 用于缓存
cache = redis.Redis(host=os.environ.get('REDIS_HOST', 'redis'), decode_responses=True)

# 数据库用于日志记录
def get_db():
    return psycopg2.connect(
        host=os.environ.get('DB_HOST', 'db'),
        database=os.environ.get('DB_NAME', 'mlapp'),
        user=os.environ.get('DB_USER', 'postgres'),
        password=os.environ.get('DB_PASSWORD', 'postgres')
    )

# 图像预处理
transform = transforms.Compose([
    transforms.Resize(256),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225])
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

    # 创建缓存哈希
    img_hash = hashlib.md5(img_bytes).hexdigest()

    # 检查缓存
    cached_result = cache.get(f'prediction:{img_hash}')
    if cached_result:
        cache.incr('cache_hits')
        return jsonify({
            'predictions': json.loads(cached_result),
            'cached': True,
            'image_hash': img_hash
        })

    # 处理图像
    img = Image.open(io.BytesIO(img_bytes)).convert('RGB')
    img_tensor = transform(img).unsqueeze(0)

    # 预测
    with torch.no_grad():
        output = model(img_tensor)
        probabilities = torch.nn.functional.softmax(output[0], dim=0)
        top5_prob, top5_catid = torch.topk(probabilities, 5)

    results = []
    for i in range(5):
        results.append({
            'class_id': top5_catid[i].item(),
            'probability': float(top5_prob[i].item())
        })

    # 缓存结果
    cache.setex(f'prediction:{img_hash}', 3600, json.dumps(results))
    cache.incr('cache_misses')

    # 记录到数据库
    try:
        conn = get_db()
        cur = conn.cursor()
        cur.execute(
            'INSERT INTO predictions (image_hash, top_class, confidence) VALUES (%s, %s, %s)',
            (img_hash, results[0]['class_id'], results[0]['probability'])
        )
        conn.commit()
        cur.close()
        conn.close()
    except Exception as e:
        print(f"DB logging error: {e}")

    return jsonify({
        'predictions': results,
        'cached': False,
        'image_hash': img_hash
    })

@app.route('/metrics')
def metrics():
    cache_hits = int(cache.get('cache_hits') or 0)
    cache_misses = int(cache.get('cache_misses') or 0)
    total = cache_hits + cache_misses
    hit_rate = (cache_hits / total * 100) if total > 0 else 0

    # 从数据库获取预测计数
    try:
        conn = get_db()
        cur = conn.cursor()
        cur.execute('SELECT COUNT(*) FROM predictions')
        total_predictions = cur.fetchone()[0]
        cur.close()
        conn.close()
    except:
        total_predictions = 0

    return jsonify({
        'cache_hits': cache_hits,
        'cache_misses': cache_misses,
        'cache_hit_rate': f"{hit_rate:.2f}%",
        'total_predictions': total_predictions
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF

# Requirements
cat > requirements.txt << 'EOF'
flask==3.0.0
torch==2.1.0
torchvision==0.16.0
pillow==10.1.0
redis==5.0.1
psycopg2-binary==2.9.9
gunicorn==21.2.0
EOF

# Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && \
    apt-get install -y --no-install-recommends libgomp1 && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY serve.py .

HEALTHCHECK --interval=30s --timeout=5s --start-period=60s \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "2", "--timeout", "120", "serve:app"]
EOF
```

### Step 5.2: ML 栈 Docker Compose

```yaml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  ml-api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DB_HOST=db
      - DB_NAME=mlapp
      - DB_USER=postgres
      - DB_PASSWORD=postgres
      - REDIS_HOST=redis
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: always
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=mlapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

volumes:
  postgres_data:
  redis_data:
  prometheus_data:
EOF
```

```bash
# 创建数据库初始化脚本
mkdir -p init-db
cat > init-db/01-schema.sql << 'EOF'
CREATE TABLE predictions (
    id SERIAL PRIMARY KEY,
    image_hash VARCHAR(32) NOT NULL,
    top_class INTEGER,
    confidence REAL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_image_hash ON predictions(image_hash);
CREATE INDEX idx_created_at ON predictions(created_at);
EOF

# 创建 Prometheus 配置
cat > prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'ml-api'
    static_configs:
      - targets: ['ml-api:8000']
EOF

# 启动 ML 栈
docker compose up -d

# 等待服务
sleep 30

# 测试健康检查
curl http://localhost:8000/health

# 测试指标
curl http://localhost:8000/metrics
```

✅ **Checkpoint**: 你可以部署带缓存和监控的 ML 推理栈。

---

## Part 6: 扩展服务

### Step 6.1: 扩展 Web 服务

```bash
# 将 web 服务扩展到 3 个副本
docker compose up -d --scale web=3

# 检查所有实例
docker compose ps

# 测试负载分发
for i in {1..10}; do
  curl http://localhost/
done

# 查看所有副本的日志
docker compose logs web

# 缩减
docker compose up -d --scale web=1
```

### Step 6.2: 使用 Nginx 进行负载均衡

```bash
# 创建负载均衡项目
mkdir -p ~/docker-exercises/load-balance
cd ~/docker-exercises/load-balance

# 创建显示主机名的简单应用
cat > app.py << 'EOF'
from flask import Flask, jsonify
import socket
import os

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'hostname': socket.gethostname(),
        'message': 'Hello from instance'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

cat > requirements.txt << 'EOF'
flask==3.0.0
EOF

cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
EOF

# 带负载均衡的 Nginx 配置
mkdir -p nginx
cat > nginx/nginx.conf << 'EOF'
upstream web_backend {
    # 负载均衡配置
    least_conn;  # 使用最少连接算法

    server web:5000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;

    location / {
        proxy_pass http://web_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - web

  web:
    build: .
    deploy:
      replicas: 3
EOF
```

```bash
# 启动 3 个副本
docker compose up -d

# 测试负载均衡（查看不同的主机名）
for i in {1..10}; do
  curl http://localhost/ | grep hostname
  sleep 0.5
done
```

✅ **Checkpoint**: 你可以扩展服务并配置负载均衡。

---

## Part 7: 命名网络和隔离

### Step 7.1: 多网络架构

```yaml
cat > docker-compose-networks.yml << 'EOF'
version: '3.8'

services:
  # 前端网络
  nginx:
    image: nginx:alpine
    networks:
      - frontend
    ports:
      - "80:80"

  # 两个网络
  api:
    build: .
    networks:
      - frontend
      - backend
    environment:
      - DB_HOST=db
      - CACHE_HOST=cache

  # 仅后端网络
  db:
    image: postgres:15
    networks:
      - backend
    environment:
      - POSTGRES_PASSWORD=postgres

  cache:
    image: redis:7-alpine
    networks:
      - backend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # 无外部访问
EOF
```

```bash
# 启动带网络隔离的 compose
docker compose -f docker-compose-networks.yml up -d

# 验证 nginx 无法直接访问数据库
docker compose -f docker-compose-networks.yml exec nginx ping db
# 应该失败 — 不同的网络！

# 但 api 可以访问两者
docker compose -f docker-compose-networks.yml exec api ping nginx
docker compose -f docker-compose-networks.yml exec api ping db
```

### Step 7.2: 自定义网络配置

```yaml
cat > docker-compose-custom-net.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx
    networks:
      app_net:
        ipv4_address: 172.25.0.10

  api:
    build: .
    networks:
      app_net:
        ipv4_address: 172.25.0.11

  db:
    image: postgres:15
    networks:
      app_net:
        ipv4_address: 172.25.0.12
    environment:
      - POSTGRES_PASSWORD=postgres

networks:
  app_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.25.0.0/16
          gateway: 172.25.0.1
EOF
```

✅ **Checkpoint**: 你可以配置网络隔离和自定义网络。

---

## Part 8: Volume 管理模式

### Step 8.1: 命名 Volume vs Bind Mount

```yaml
cat > docker-compose-volumes.yml << 'EOF'
version: '3.8'

services:
  web:
    build: .
    volumes:
      # 用于开发的 bind mount（热重载）
      - ./app:/app
      # 用于依赖的命名 volume
      - node_modules:/app/node_modules
      # 用于临时文件的 tmpfs
      - type: tmpfs
        target: /tmp

  db:
    image: postgres:15
    volumes:
      # 用于持久化的命名 volume
      - db_data:/var/lib/postgresql/data
      # 用于备份的 bind mount
      - ./backups:/backups
    environment:
      - POSTGRES_PASSWORD=postgres

  logs:
    image: busybox
    volumes:
      # 服务之间共享的 volume
      - app_logs:/logs
    command: tail -f /logs/app.log

volumes:
  node_modules:
  db_data:
  app_logs:
EOF
```

### Step 8.2: 备份和恢复

```bash
# 备份 volume 数据
docker compose -f docker-compose-volumes.yml exec db \
  pg_dump -U postgres mydb > ./backups/backup-$(date +%Y%m%d).sql

# 或者备份整个 volume
docker run --rm \
  -v full-stack_postgres_data:/data \
  -v $(pwd)/backups:/backup \
  alpine tar czf /backup/postgres-data.tar.gz -C /data .

# 从备份恢复
docker run --rm \
  -v full-stack_postgres_data:/data \
  -v $(pwd)/backups:/backup \
  alpine tar xzf /backup/postgres-data.tar.gz -C /data
```

✅ **Checkpoint**: 你可以为不同用例管理 volume。

---

## Part 9: 生产环境最佳实践

### Step 9.1: 生产就绪的 Compose 文件

```yaml
cat > docker-compose.prod.yml << 'EOF'
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.prod
    restart: always
    environment:
      - ENVIRONMENT=production
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    networks:
      - app_network

  db:
    image: postgres:15
    restart: always
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password
    volumes:
      - db_data:/var/lib/postgresql/data
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app_network

secrets:
  db_password:
    file: ./secrets/db_password.txt

volumes:
  db_data:
    driver: local

networks:
  app_network:
    driver: bridge
EOF
```

### Step 9.2: 健康检查和监控

```yaml
cat > docker-compose-monitoring.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin

volumes:
  prometheus_data:
  grafana_data:
EOF
```

✅ **Checkpoint**: 你可以创建生产就绪的 Compose 配置。

---

## Part 10: 调试和排错

### Step 10.1: 常见问题

```bash
# 查看服务日志
docker compose logs service_name

# 实时跟踪日志
docker compose logs -f --tail=100

# 检查服务状态
docker compose ps

# 检查服务详情
docker compose exec service_name env
docker compose exec service_name ps aux

# 重启特定服务
docker compose restart service_name

# 重建并重启
docker compose up -d --build service_name

# 验证 compose 文件
docker compose config

# 查看解析后的配置
docker compose config --resolve-image-digests
```

### Step 10.2: 调试技巧

```bash
# 逐个启动服务
docker compose up db
docker compose up db redis
docker compose up  # 所有服务

# 在服务中运行命令
docker compose exec web bash
docker compose exec db psql -U postgres

# 检查网络连接
docker compose exec web ping db
docker compose exec web nc -zv db 5432

# 查看资源使用
docker stats

# 检查 volume
docker volume ls
docker volume inspect project_volume_name
```

✅ **Checkpoint**: 你可以调试和排错 Compose 应用。

---

## 实际挑战

### Challenge 1: 完整的电商栈

**Requirements**:
- 前端 (Nginx)
- API (Python/Flask)
- 数据库 (PostgreSQL)
- 缓存 (Redis)
- 消息队列 (RabbitMQ)
- Worker 服务
- 全部带有适当的健康检查
- 基于环境的配置

### Challenge 2: ML Pipeline

**Requirements**:
- 模型训练服务（定时执行）
- 模型 serving API
- 特征存储 (PostgreSQL)
- 模型注册中心
- 监控 (Prometheus + Grafana)
- 适当的模型 volume 管理

### Challenge 3: 微服务架构

**Requirements**:
- 用户服务
- 产品服务
- 订单服务
- API Gateway
- Service mesh
- 前后端分离网络
- 共享日志 volume

---

## Summary

**What You Accomplished**:
✅ 使用 Docker Compose 创建了多 container 应用
✅ 管理了服务依赖和启动顺序
✅ 配置了环境变量和密钥
✅ 构建了完整的 web 栈 (web + db + cache + proxy)
✅ 部署了带支持基础设施的 ML 推理服务
✅ 扩展了服务并配置了负载均衡
✅ 实现了网络隔离
✅ 为不同用例管理了 volume
✅ 应用了生产 best practices
✅ 调试和排错了 compose 应用

**Key Compose Features Mastered**:
- 服务定义和依赖
- 健康检查和重启策略
- 环境配置和密钥
- Volume 管理（命名、bind mount、tmpfs）
- 网络隔离和自定义网络
- 扩展和资源限制
- 日志配置
- 生产部署模式

---

## Next Steps

**Continue Learning**:
- **Exercise 04**: Advanced Networking Scenarios
- **Exercise 05**: Volume Management and Data Persistence
- **Exercise 06**: Production Deployment Strategies
- **Exercise 07**: Complete ML Application Containerization

---

**Exercise Version**: 1.0
**Last Updated**: October 2025
**Estimated Time**: 3-4 小时
**Difficulty**: 中级
