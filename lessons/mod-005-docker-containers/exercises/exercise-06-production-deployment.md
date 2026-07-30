# Exercise 06: Production Deployment Strategies

## Exercise Overview

**Objective**: 掌握生产就绪的 Docker 部署模式，包括安全加固、资源管理、监控、日志记录、高可用性和 CI/CD 集成。

**Difficulty**: 高级
**Estimated Time**: 4-5 小时
**Prerequisites**:
- Exercise 01-05 (所有之前的 Docker 练习)
- 所有 Docker 讲座

**What You'll Learn**:
- 生产安全最佳实践
- 资源限制和配额
- 健康检查和监控
- 日志记录策略
- 高可用模式
- 蓝绿部署和金丝雀部署
- CI/CD 集成
- 密钥管理
- 多阶段部署
- 回滚策略

---

## Part 1: 安全加固

### Step 1.1: 非 Root 用户模式

```dockerfile
# 创建生产安全的 Dockerfile
mkdir -p ~/docker-exercises/production
cd ~/docker-exercises/production

cat > Dockerfile.secure << 'EOF'
FROM python:3.11-slim

# 创建非 root 用户
RUN groupadd -r appuser && \
    useradd -r -g appuser -u 1000 -m -s /bin/bash appuser

# 设置工作目录
WORKDIR /app

# 以 root 身份安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 以正确所有权复制应用文件
COPY --chown=appuser:appuser . .

# 切换到非 root 用户
USER appuser

# 运行应用
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
EOF
```

### Step 1.2: 最小化攻击面

```dockerfile
cat > Dockerfile.minimal << 'EOF'
# 多阶段构建
FROM python:3.11 AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# 最小化运行时
FROM python:3.11-slim

# 仅安装运行时依赖
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        libpq5 \
    && rm -rf /var/lib/apt/lists/*

# 创建非 root 用户
RUN groupadd -r appuser && useradd -r -g appuser -u 1000 appuser

WORKDIR /app

# 仅复制必要文件
COPY --from=builder /root/.local /home/appuser/.local
COPY --chown=appuser:appuser app.py .

USER appuser

ENV PATH=/home/appuser/.local/bin:$PATH

CMD ["python", "app.py"]
EOF
```

### Step 1.3: 安全扫描

```bash
# 创建用于测试的简单应用
cat > app.py << 'EOF'
from flask import Flask
app = Flask(__name__)

@app.route('/health')
def health():
    return {'status': 'healthy'}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF

cat > requirements.txt << 'EOF'
flask==3.0.0
gunicorn==21.2.0
EOF

# 构建 image
docker build -f Dockerfile.secure -t myapp:secure .

# 扫描漏洞（使用 Docker Scout 或 Trivy）
# docker scout cves myapp:secure

# 或使用 Trivy
# docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
#   aquasec/trivy image myapp:secure
```

### Step 1.4: 只读根文件系统

```yaml
cat > docker-compose-readonly.yml << 'EOF'
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.secure
    read_only: true  # 根文件系统只读
    tmpfs:
      - /tmp  # 可写临时目录
      - /var/run
    volumes:
      - app-data:/app/data  # 可写数据目录
    security_opt:
      - no-new-privileges:true  # 防止权限提升
    cap_drop:
      - ALL  # 删除所有能力
    cap_add:
      - NET_BIND_SERVICE  # 仅添加必要的能力

volumes:
  app-data:
EOF
```

✅ **Checkpoint**: 你可以为 container 实现安全加固。

---

## Part 2: 资源管理

### Step 2.1: CPU 和内存限制

```yaml
cat > docker-compose-resources.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx
    deploy:
      resources:
        limits:
          cpus: '0.5'  # 最多 50% 的 CPU
          memory: 512M  # 最多 512MB RAM
        reservations:
          cpus: '0.25'  # 保证 25% CPU
          memory: 256M  # 保证 256MB RAM
    restart: always

  api:
    build: .
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
      replicas: 3

  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
        reservations:
          cpus: '1.0'
          memory: 1G
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
EOF
```

### Step 2.2: 测试资源限制

```bash
# 启动带资源限制的 compose
docker compose -f docker-compose-resources.yml up -d

# 监控资源使用
docker stats

# 测试内存限制
docker run -d \
  --name memory-test \
  --memory="100m" \
  --memory-swap="100m" \
  progrium/stress \
  --vm 1 --vm-bytes 150M

# Container 应被杀死（OOM）
docker logs memory-test
docker ps -a | grep memory-test

# 测试 CPU 限制
docker run -d \
  --name cpu-test \
  --cpus="0.5" \
  progrium/stress \
  --cpu 2

# 检查 CPU 使用（不应超过 50%）
docker stats cpu-test

# 清理
docker rm -f memory-test cpu-test
docker compose -f docker-compose-resources.yml down
```

### Step 2.3: 磁盘 I/O 限制

```bash
# 限制磁盘 I/O
docker run -d \
  --name io-test \
  --device-read-bps /dev/sda:1mb \
  --device-write-bps /dev/sda:1mb \
  alpine sh -c "while true; do dd if=/dev/zero of=/tmp/test bs=1M count=10; done"

# 监控 I/O
docker stats io-test

# 清理
docker rm -f io-test
```

✅ **Checkpoint**: 你可以配置和实施资源限制。

---

## Part 3: 健康检查和监控

### Step 3.1: 全面的健康检查

```dockerfile
cat > Dockerfile.health << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    pip install requests  # 用于健康检查

COPY app.py health_check.py ./

# 全面的健康检查
HEALTHCHECK --interval=30s \
            --timeout=10s \
            --start-period=40s \
            --retries=3 \
  CMD python health_check.py || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:app"]
EOF

cat > health_check.py << 'EOF'
#!/usr/bin/env python3
import sys
import requests
import psutil

def check_app_health():
    """检查应用是否响应"""
    try:
        r = requests.get('http://localhost:8000/health', timeout=5)
        return r.status_code == 200
    except:
        return False

def check_system_health():
    """检查系统资源"""
    # 检查 CPU 使用
    cpu = psutil.cpu_percent(interval=1)
    if cpu > 90:
        print(f"High CPU usage: {cpu}%")
        return False

    # 检查内存使用
    memory = psutil.virtual_memory()
    if memory.percent > 90:
        print(f"High memory usage: {memory.percent}%")
        return False

    # 检查磁盘使用
    disk = psutil.disk_usage('/')
    if disk.percent > 90:
        print(f"High disk usage: {disk.percent}%")
        return False

    return True

if __name__ == '__main__':
    app_healthy = check_app_health()
    system_healthy = check_system_health()

    if app_healthy and system_healthy:
        print("Health check passed")
        sys.exit(0)
    else:
        print("Health check failed")
        sys.exit(1)
EOF
```

### Step 3.2: Docker Compose 健康检查

```yaml
cat > docker-compose-health.yml << 'EOF'
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    volumes:
      - db-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  api:
    build:
      context: .
      dockerfile: Dockerfile.health
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "health_check.py"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: on-failure:3

volumes:
  db-data:
EOF
```

### Step 3.3: 使用 Prometheus 监控

```yaml
cat > docker-compose-monitoring.yml << 'EOF'
version: '3.8'

services:
  # 带指标端点的应用
  app:
    build: .
    ports:
      - "8000:8000"
    labels:
      prometheus.scrape: "true"
      prometheus.port: "8000"
      prometheus.path: "/metrics"

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'

  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    depends_on:
      - prometheus

  # 用于系统指标的 Node exporter
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'

volumes:
  prometheus-data:
  grafana-data:
EOF

cat > prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'app'
    static_configs:
      - targets: ['app:8000']

  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
EOF
```

✅ **Checkpoint**: 你可以实现健康检查和监控。

---

## Part 4: 日志记录策略

### Step 4.1: 集中式日志记录

```yaml
cat > docker-compose-logging.yml << 'EOF'
version: '3.8'

services:
  # 应用
  app:
    build: .
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
        labels: "app,environment"
    labels:
      app: "myapp"
      environment: "production"

  # Elasticsearch
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.10.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  # Logstash
  logstash:
    image: docker.elastic.co/logstash/logstash:8.10.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    depends_on:
      - elasticsearch

  # Kibana
  kibana:
    image: docker.elastic.co/kibana/kibana:8.10.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  # 用于日志收集的 Filebeat
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.10.0
    user: root
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    command: filebeat -e -strict.perms=false
    depends_on:
      - elasticsearch

volumes:
  es-data:
EOF
```

### Step 4.2: 结构化日志记录

```python
cat > logging_app.py << 'EOF'
import logging
import json
from datetime import datetime
from flask import Flask, request

app = Flask(__name__)

# 结构化日志记录
class JSONFormatter(logging.Formatter):
    def format(self, record):
        log_obj = {
            'timestamp': datetime.utcnow().isoformat(),
            'level': record.levelname,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno
        }
        if hasattr(record, 'request_id'):
            log_obj['request_id'] = record.request_id
        if hasattr(record, 'user_id'):
            log_obj['user_id'] = record.user_id
        return json.dumps(log_obj)

handler = logging.StreamHandler()
handler.setFormatter(JSONFormatter())
logger = logging.getLogger()
logger.addHandler(handler)
logger.setLevel(logging.INFO)

@app.route('/api/<path:path>')
def api(path):
    logger.info('API request', extra={
        'request_id': request.headers.get('X-Request-ID'),
        'path': path,
        'method': request.method,
        'ip': request.remote_addr
    })
    return {'status': 'ok'}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF
```

### Step 4.3: 日志聚合模式

```yaml
cat > docker-compose-fluentd.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: "app.{{.Name}}"

  fluentd:
    image: fluent/fluentd:latest
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    volumes:
      - ./fluentd.conf:/fluentd/etc/fluent.conf
      - fluentd-data:/fluentd/log

volumes:
  fluentd-data:
EOF
```

✅ **Checkpoint**: 你可以实现生产日志记录策略。

---

## Part 5: 高可用模式

### Step 5.1: 副本管理

```yaml
cat > docker-compose-ha.yml << 'EOF'
version: '3.8'

services:
  # 负载均衡器
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx-lb.conf:/etc/nginx/nginx.conf
    depends_on:
      - app

  # 带多个副本的应用
  app:
    build: .
    deploy:
      replicas: 3
      update_config:
        parallelism: 1  # 一次更新一个
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # 带复制的数据库
  db-primary:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_REPLICATION_MODE=master
      - POSTGRES_REPLICATION_USER=replicator
      - POSTGRES_REPLICATION_PASSWORD=replicator_password
    volumes:
      - db-primary-data:/var/lib/postgresql/data

  db-replica:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_MASTER_SERVICE_HOST=db-primary
      - POSTGRES_REPLICATION_MODE=slave
      - POSTGRES_REPLICATION_USER=replicator
      - POSTGRES_REPLICATION_PASSWORD=replicator_password
    volumes:
      - db-replica-data:/var/lib/postgresql/data
    depends_on:
      - db-primary

volumes:
  db-primary-data:
  db-replica-data:
EOF
```

### Step 5.2: 自动故障转移

```yaml
cat > docker-compose-failover.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    deploy:
      replicas: 3
      placement:
        max_replicas_per_node: 1  # 跨节点分布
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 40s
    restart: always

  # 带 Sentinel 自动故障转移的 Redis
  redis-master:
    image: redis:7-alpine
    command: redis-server --appendonly yes

  redis-replica:
    image: redis:7-alpine
    command: redis-server --appendonly yes --slaveof redis-master 6379
    deploy:
      replicas: 2
    depends_on:
      - redis-master

  redis-sentinel:
    image: redis:7-alpine
    command: >
      redis-sentinel
      --sentinel monitor mymaster redis-master 6379 2
      --sentinel down-after-milliseconds mymaster 5000
      --sentinel parallel-syncs mymaster 1
      --sentinel failover-timeout mymaster 10000
    deploy:
      replicas: 3
    depends_on:
      - redis-master
EOF
```

✅ **Checkpoint**: 你可以实现高可用模式。

---

## Part 6: 部署策略

### Step 6.1: 蓝绿部署

```yaml
cat > docker-compose-bluegreen.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx-bluegreen.conf:/etc/nginx/nginx.conf
    environment:
      - ACTIVE_ENV=${ACTIVE_ENV:-blue}

  # 蓝色环境（当前生产环境）
  app-blue:
    build:
      context: .
      args:
        VERSION: blue
    environment:
      - VERSION=blue
    deploy:
      replicas: 3

  # 绿色环境（新版本）
  app-green:
    build:
      context: .
      args:
        VERSION: green
    environment:
      - VERSION=green
    deploy:
      replicas: 3
EOF

# 蓝绿部署的 Nginx 配置
cat > nginx-bluegreen.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream blue {
        server app-blue:8000;
    }

    upstream green {
        server app-green:8000;
    }

    # 在蓝色和绿色之间切换
    map $http_upgrade $backend {
        default blue;  # 改为 'green' 以切换
    }

    server {
        listen 80;

        location / {
            proxy_pass http://$backend;
            proxy_set_header Host $host;
        }
    }
}
EOF
```

```bash
# 部署蓝色（当前）
ACTIVE_ENV=blue docker compose -f docker-compose-bluegreen.yml up -d app-blue

# 在蓝色旁边部署绿色（新版本）
docker compose -f docker-compose-bluegreen.yml up -d app-green

# 测试绿色环境
curl -H "Host: green.example.com" http://localhost/

# 将流量切换到绿色（更新 nginx 配置）
# 然后重新加载 nginx
docker compose -f docker-compose-bluegreen.yml exec nginx nginx -s reload

# 如果成功，删除蓝色
# 如果失败，切换回蓝色
```

### Step 6.2: 金丝雀部署

```yaml
cat > docker-compose-canary.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx-canary.conf:/etc/nginx/nginx.conf

  # 稳定版本（90% 流量）
  app-stable:
    build:
      context: .
      args:
        VERSION: stable
    deploy:
      replicas: 9
    environment:
      - VERSION=stable

  # 金丝雀版本（10% 流量）
  app-canary:
    build:
      context: .
      args:
        VERSION: canary
    deploy:
      replicas: 1
    environment:
      - VERSION=canary
EOF

cat > nginx-canary.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream backend {
        # 90% 到 stable
        server app-stable:8000 weight=9;
        # 10% 到 canary
        server app-canary:8000 weight=1;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;

            # 添加标头以标识版本
            add_header X-App-Version $upstream_addr;
        }
    }
}
EOF
```

### Step 6.3: 滚动更新

```yaml
cat > docker-compose-rolling.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    deploy:
      replicas: 6
      update_config:
        parallelism: 2  # 一次更新 2 个
        delay: 10s  # 批次之间等待 10s
        failure_action: rollback  # 失败时回滚
        monitor: 30s  # 监控 30 秒
        max_failure_ratio: 0.3  # 如果 >30% 失败则回滚
        order: start-first  # 先启动新的，再停止旧的
      rollback_config:
        parallelism: 2
        delay: 5s
        failure_action: pause
        monitor: 30s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
EOF
```

```bash
# 初始部署
docker compose -f docker-compose-rolling.yml up -d

# 更新到新版本
docker compose -f docker-compose-rolling.yml build
docker compose -f docker-compose-rolling.yml up -d --no-deps --build app

# 监控滚动更新
watch docker compose -f docker-compose-rolling.yml ps
```

✅ **Checkpoint**: 你可以实现各种部署策略。

---

## Part 7: 密钥管理

### Step 7.1: Docker Secrets

```bash
# 创建密钥
echo "db_password_secret" | docker secret create db_password -
echo "api_key_secret" | docker secret create api_key -

# 列出密钥
docker secret ls

# 在 Docker Compose 中使用
cat > docker-compose-secrets.yml << 'EOF'
version: '3.8'

services:
  app:
    image: myapp
    secrets:
      - db_password
      - api_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - API_KEY_FILE=/run/secrets/api_key

  db:
    image: postgres:15
    secrets:
      - db_password
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password

secrets:
  db_password:
    external: true
  api_key:
    external: true
EOF
```

### Step 7.2: 环境特定密钥

```yaml
cat > docker-compose-env-secrets.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    secrets:
      - source: db_password
        target: /run/secrets/db_password
        mode: 0400  # 仅所有者只读
    environment:
      - ENVIRONMENT=${ENVIRONMENT:-production}

secrets:
  db_password:
    file: ./secrets/${ENVIRONMENT}/db_password.txt
EOF
```

```bash
# 创建密钥目录结构
mkdir -p secrets/development secrets/production

echo "dev_password" > secrets/development/db_password.txt
echo "prod_password" > secrets/production/db_password.txt

# 使用不同环境运行
ENVIRONMENT=development docker compose -f docker-compose-env-secrets.yml up -d
ENVIRONMENT=production docker compose -f docker-compose-env-secrets.yml up -d
```

### Step 7.3: Vault 集成（概念）

```python
# app_with_vault.py
cat > app_with_vault.py << 'EOF'
import hvac
import os
from flask import Flask

app = Flask(__name__)

# 初始化 Vault 客户端
vault_client = hvac.Client(
    url=os.environ.get('VAULT_ADDR', 'http://vault:8200'),
    token=os.environ.get('VAULT_TOKEN')
)

# 从 Vault 读取密钥
def get_secret(path):
    try:
        secret = vault_client.secrets.kv.v2.read_secret_version(path=path)
        return secret['data']['data']
    except Exception as e:
        print(f"Error reading secret: {e}")
        return None

# 使用密钥
db_credentials = get_secret('database/credentials')
api_keys = get_secret('api/keys')

@app.route('/health')
def health():
    return {'status': 'healthy'}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF
```

✅ **Checkpoint**: 你可以安全地管理密钥。

---

## Part 8: CI/CD 集成

### Step 8.1: GitHub Actions 工作流

```yaml
# .github/workflows/docker-deploy.yml
cat > docker-deploy.yml << 'EOF'
name: Docker Build and Deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2

      - name: Log in to registry
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
            type=sha

      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:buildcache,mode=max

      - name: Run security scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.meta.outputs.version }}
          format: 'sarif'
          output: 'trivy-results.sarif'

      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to production
        run: |
          # SSH 到服务器并更新 container
          # docker-compose pull
          # docker-compose up -d
          echo "Deploy step would go here"
EOF
```

### Step 8.2: GitLab CI/CD

```yaml
# .gitlab-ci.yml
cat > gitlab-ci.yml << 'EOF'
stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_REF_SLUG

build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG

test:
  stage: test
  image: $IMAGE_TAG
  script:
    - python -m pytest tests/

deploy_production:
  stage: deploy
  only:
    - main
  script:
    - docker-compose pull
    - docker-compose up -d
  environment:
    name: production
    url: https://app.example.com
EOF
```

### Step 8.3: 自动化测试

```bash
# 创建测试脚本
cat > test-deployment.sh << 'EOF'
#!/bin/bash

set -e

echo "Starting deployment tests..."

# 构建 image
docker build -t myapp:test .

# 启动服务
docker compose -f docker-compose.test.yml up -d

# 等待服务健康
echo "Waiting for services..."
timeout 60 sh -c 'until docker compose -f docker-compose.test.yml ps | grep -q healthy; do sleep 2; done'

# 运行集成测试
docker compose -f docker-compose.test.yml exec -T app pytest tests/integration/

# 运行冒烟测试
curl -f http://localhost:8000/health || exit 1
curl -f http://localhost:8000/api/status || exit 1

# 清理
docker compose -f docker-compose.test.yml down -v

echo "All tests passed!"
EOF

chmod +x test-deployment.sh
```

✅ **Checkpoint**: 你可以将 Docker 与 CI/CD pipeline 集成。

---

## Part 9: 完整生产栈

### Step 9.1: 完整生产配置

```yaml
cat > docker-compose.production.yml << 'EOF'
version: '3.8'

services:
  # 反向代理
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/prod.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx-cache:/var/cache/nginx
    depends_on:
      - app
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # 应用
  app:
    build:
      context: .
      dockerfile: Dockerfile.prod
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1.0'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    secrets:
      - db_password
      - api_key
    environment:
      - ENVIRONMENT=production
      - DB_HOST=db
      - REDIS_HOST=redis
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    read_only: true
    tmpfs:
      - /tmp
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # 数据库
  db:
    image: postgres:15
    secrets:
      - db_password
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./backups:/backups
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 2G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: always
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # 缓存
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    restart: always

  # 监控
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
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
      - GF_USERS_ALLOW_SIGN_UP=false
    secrets:
      - grafana_password
    depends_on:
      - prometheus
    restart: always

volumes:
  db-data:
    driver: local
  redis-data:
    driver: local
  prometheus-data:
    driver: local
  grafana-data:
    driver: local
  nginx-cache:
    driver: local

secrets:
  db_password:
    file: ./secrets/prod/db_password.txt
  api_key:
    file: ./secrets/prod/api_key.txt
  grafana_password:
    file: ./secrets/prod/grafana_password.txt

networks:
  default:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
EOF
```

✅ **Checkpoint**: 你可以部署完整的生产栈。

---

## Summary

**What You Accomplished**:
✅ 实现了安全加固
✅ 配置了资源限制
✅ 设置了健康检查和监控
✅ 实现了集中式日志记录
✅ 构建了高可用模式
✅ 掌握了部署策略
✅ 安全地管理了密钥
✅ 集成了 CI/CD
✅ 部署了完整生产栈

**Production Checklist**:
- [x] 非 root 用户
- [x] 只读文件系统
- [x] 资源限制
- [x] 健康检查
- [x] 日志配置
- [x] 监控启用
- [x] 密钥管理
- [x] 高可用
- [x] 自动化部署
- [x] 回滚策略

---

## Next Steps

**Continue Learning**:
- **Exercise 07**: Complete ML Application Containerization
- **Module Quiz**: 测试你的 Docker 知识

---

**Exercise Version**: 1.0
**Last Updated**: October 2025
**Estimated Time**: 4-5 小时
**Difficulty**: 高级
