# Exercise 04: Docker Networking Scenarios

## Exercise Overview

**Objective**: 掌握 Docker 网络概念，包括网络驱动、服务发现、DNS 解析、网络隔离以及面向 ML 基础设施的高级网络模式。

**Difficulty**: 中级到高级
**Estimated Time**: 3-4 小时
**Prerequisites**:
- Exercise 01-03 (Container Operations, Images, Compose)
- Lecture 04 (Networking & Volumes)

**What You'll Learn**:
- Docker 网络驱动（bridge、host、overlay、macvlan）
- Container DNS 和服务发现
- 网络隔离和安全
- 自定义 bridge 网络
- Container 间通信
- 外部连接
- 负载均衡模式
- 多主机网络
- 网络问题排错

---

## Part 1: 理解网络驱动

### Step 1.1: Bridge 网络（默认）

```bash
# 创建项目目录
mkdir -p ~/docker-exercises/networking
cd ~/docker-exercises/networking

# 检查默认 bridge 网络
docker network ls
docker network inspect bridge

# 在默认 bridge 上运行 container
docker run -d --name web1 nginx
docker run -d --name web2 nginx

# 查看它们的 IP
docker inspect web1 --format='{{.NetworkSettings.IPAddress}}'
docker inspect web2 --format='{{.NetworkSettings.IPAddress}}'

# 尝试 DNS 解析（在默认 bridge 上会失败！）
docker exec web1 ping web2
# Error: ping: web2: Name or service not known

# Container 可以通过 IP 互 ping
WEB2_IP=$(docker inspect web2 --format='{{.NetworkSettings.IPAddress}}')
docker exec web1 ping -c 3 $WEB2_IP

# 清理
docker rm -f web1 web2
```

**Key Insight**: 默认 bridge 网络不支持自动 DNS 解析！

### Step 1.2: 自定义 Bridge 网络

```bash
# 创建自定义 bridge 网络
docker network create my-network

# 检查
docker network inspect my-network

# 在自定义网络上运行 container
docker run -d --name web1 --network my-network nginx
docker run -d --name web2 --network my-network nginx

# 现在 DNS 可以工作了！
docker exec web1 ping -c 3 web2
docker exec web2 ping -c 3 web1

# 成功！自定义 bridge 提供自动 DNS 解析
```

### Step 1.3: Host 网络

```bash
# 使用 host 网络运行 container
docker run -d --name web-host --network host nginx

# Container 直接使用主机的网络栈
# 无需端口映射 — nginx 可通过 80 端口访问

# 检查（如果 80 端口可用）
curl http://localhost:80

# 查看网络设置
docker inspect web-host --format='{{.NetworkSettings.Networks}}'

# 清理
docker rm -f web-host
```

**Use Cases**:
- 高性能网络（无 NAT 开销）
- 网络监控工具
- 需要主机网络接口时

**Limitations**:
- 没有网络隔离
- 不能在相同端口上运行多个 container
- 可移植性较差

### Step 1.4: None 网络

```bash
# 没有网络的 container
docker run -d --name isolated --network none alpine sleep 3600

# 验证没有网络接口（除 loopback 外）
docker exec isolated ip addr show

# 只看到 'lo'（loopback）

# 清理
docker rm -f isolated
```

**Use Cases**:
- 最大程度的网络隔离
- 测试场景
- 安全敏感性工作负载

✅ **Checkpoint**: 你理解了不同的 Docker 网络驱动。

---

## Part 2: 服务发现和 DNS

### Step 2.1: 基本服务发现

```bash
# 创建网络
docker network create app-net

# 运行数据库
docker run -d \
  --name postgres-db \
  --network app-net \
  -e POSTGRES_PASSWORD=secret \
  postgres:15

# 运行连接到 'postgres-db' 的应用
cat > app.py << 'EOF'
import psycopg2
import sys

try:
    # 使用服务名称 'postgres-db' 连接
    conn = psycopg2.connect(
        host='postgres-db',
        database='postgres',
        user='postgres',
        password='secret'
    )
    print("✓ Successfully connected to database!")
    print(f"Database: {conn.get_dsn_parameters()['host']}")
    conn.close()
except Exception as e:
    print(f"✗ Connection failed: {e}")
    sys.exit(1)
EOF

cat > Dockerfile << 'EOF'
FROM python:3.11-slim
RUN pip install psycopg2-binary
COPY app.py .
CMD ["python", "app.py"]
EOF

docker build -t db-client .

# 运行客户端 — 它通过名称找到数据库！
docker run --rm --network app-net db-client
```

### Step 2.2: DNS 轮询

```bash
# 创建具有相同网络别名的多个 container
docker network create lb-net

# 运行 3 个别名为 'web' 的 web 服务器
docker run -d --name web1 --network lb-net --network-alias web nginx
docker run -d --name web2 --network lb-net --network-alias web nginx
docker run -d --name web3 --network lb-net --network-alias web nginx

# 创建测试客户端
docker run -it --rm --network lb-net alpine sh

# 在 container 内部，尝试多次 DNS 查找：
# nslookup web
# wget -qO- http://web
# 多次请求会在多个 container 间轮询
```

### Step 2.3: Compose 中的服务发现

```yaml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  api:
    build: .
    environment:
      # 可以按名称引用其他服务
      - DB_HOST=database
      - CACHE_HOST=redis-cache
      - QUEUE_HOST=message-queue
    depends_on:
      - database
      - redis-cache
      - message-queue

  database:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret

  redis-cache:
    image: redis:7-alpine

  message-queue:
    image: rabbitmq:3-alpine
EOF
```

**How It Works**:
- Compose 创建一个默认网络
- 每个服务获得与其服务名称匹配的 DNS 条目
- 服务可以按名称解析彼此
- 多个副本获得轮询 DNS

✅ **Checkpoint**: 你理解了 Docker DNS 和服务发现。

---

## Part 3: 网络隔离模式

### Step 3.1: 前后端分离

```bash
mkdir -p ~/docker-exercises/network-isolation
cd ~/docker-exercises/network-isolation

# 创建简单 API
cat > api.py << 'EOF'
from flask import Flask, jsonify
import psycopg2
import os

app = Flask(__name__)

@app.route('/data')
def get_data():
    conn = psycopg2.connect(
        host='db',
        database='app',
        user='postgres',
        password=os.environ['DB_PASSWORD']
    )
    cur = conn.cursor()
    cur.execute('SELECT version();')
    version = cur.fetchone()[0]
    return jsonify({'db_version': version})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

cat > requirements.txt << 'EOF'
flask==3.0.0
psycopg2-binary==2.9.9
EOF

cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY api.py .
CMD ["python", "api.py"]
EOF
```

```yaml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # 面向公网（仅前端网络）
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    networks:
      - frontend
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf

  # API 层（两个网络）
  api:
    build: .
    networks:
      - frontend
      - backend
    environment:
      - DB_PASSWORD=secret

  # 数据库（仅后端网络 — 隔离！）
  db:
    image: postgres:15
    networks:
      - backend
    environment:
      - POSTGRES_DB=app
      - POSTGRES_PASSWORD=secret

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # 无外部互联网访问
EOF
```

```bash
# 启动服务
docker compose up -d

# 验证隔离
# Nginx 可以访问 API
docker compose exec nginx nc -zv api 5000

# Nginx 不能直接访问数据库（不同网络）
docker compose exec nginx nc -zv db 5432
# 应该失败！

# API 可以访问数据库（在后端网络上）
docker compose exec api nc -zv db 5432

# 数据库无法访问互联网（内部网络）
docker compose exec db ping -c 1 8.8.8.8
# 应该失败！
```

**Security Benefits**:
- 数据库不暴露给前端
- 后端网络无互联网访问
- API 充当受控网关
- 最小化攻击面

### Step 3.2: 多层隔离

```yaml
cat > docker-compose-3tier.yml << 'EOF'
version: '3.8'

services:
  # 第 1 层：公共 DMZ
  load-balancer:
    image: nginx:alpine
    ports:
      - "443:443"
    networks:
      - dmz

  # 第 2 层：应用层
  web-app:
    build: .
    networks:
      - dmz
      - app-tier
    depends_on:
      - api

  api:
    build: ./api
    networks:
      - app-tier
      - data-tier

  # 第 3 层：数据层
  database:
    image: postgres:15
    networks:
      - data-tier
    environment:
      - POSTGRES_PASSWORD=secret

  cache:
    image: redis:7-alpine
    networks:
      - data-tier

networks:
  dmz:
    driver: bridge
  app-tier:
    driver: bridge
    internal: true
  data-tier:
    driver: bridge
    internal: true
EOF
```

**Network Diagram**:
```
Internet → DMZ → App Tier → Data Tier
           ↓      ↓          ↓
          LB     Web/API    DB/Cache
```

✅ **Checkpoint**: 你可以为实现安全性实现网络隔离。

---

## Part 4: 自定义网络配置

### Step 4.1: 自定义子网和 IP 范围

```bash
# 使用自定义子网创建网络
docker network create \
  --driver bridge \
  --subnet 172.25.0.0/16 \
  --gateway 172.25.0.1 \
  --ip-range 172.25.5.0/24 \
  custom-net

# 为 container 分配特定 IP
docker run -d \
  --name web \
  --network custom-net \
  --ip 172.25.5.10 \
  nginx

docker run -d \
  --name api \
  --network custom-net \
  --ip 172.25.5.11 \
  nginx

# 验证 IP
docker inspect web --format='{{.NetworkSettings.Networks.custom_net.IPAddress}}'
docker inspect api --format='{{.NetworkSettings.Networks.custom_net.IPAddress}}'
```

### Step 4.2: 每个 Container 多个网络

```bash
# 创建两个网络
docker network create net-a
docker network create net-b

# 在 net-a 上运行 container
docker run -d --name app --network net-a nginx

# 也连接到 net-b
docker network connect net-b app

# 现在 container 在两个网络上都有 IP
docker inspect app --format='{{json .NetworkSettings.Networks}}' | python -m json.tool

# Container 可以访问任一网络上的服务！
```

### Step 4.3: 网络别名

```bash
docker network create multi-alias

# 使用多个别名运行 container
docker run -d \
  --name web \
  --network multi-alias \
  --network-alias webserver \
  --network-alias api \
  --network-alias frontend \
  nginx

# 所有这些 DNS 名称解析到同一个 container：
docker run --rm --network multi-alias alpine nslookup webserver
docker run --rm --network multi-alias alpine nslookup api
docker run --rm --network multi-alias alpine nslookup frontend
```

✅ **Checkpoint**: 你可以配置高级网络设置。

---

## Part 5: Container 通信模式

### Step 5.1: 请求-响应模式

```bash
mkdir -p ~/docker-exercises/communication
cd ~/docker-exercises/communication

# 服务 A（客户端）
cat > client.py << 'EOF'
import requests
import time

while True:
    try:
        response = requests.get('http://server:5000/data')
        print(f"Response: {response.json()}")
    except Exception as e:
        print(f"Error: {e}")
    time.sleep(5)
EOF

# 服务 B（服务器）
cat > server.py << 'EOF'
from flask import Flask, jsonify
import random

app = Flask(__name__)

@app.route('/data')
def data():
    return jsonify({
        'value': random.randint(1, 100),
        'timestamp': 'now'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

```yaml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  client:
    image: python:3.11-slim
    command: sh -c "pip install requests && python /app/client.py"
    volumes:
      - ./client.py:/app/client.py
    depends_on:
      - server

  server:
    image: python:3.11-slim
    command: sh -c "pip install flask && python /app/server.py"
    volumes:
      - ./server.py:/app/server.py
EOF
```

### Step 5.2: 发布-订阅模式

```bash
# 发布者
cat > publisher.py << 'EOF'
import redis
import time
import json

r = redis.Redis(host='redis', decode_responses=True)

counter = 0
while True:
    message = {'id': counter, 'data': f'Message {counter}'}
    r.publish('events', json.dumps(message))
    print(f"Published: {message}")
    counter += 1
    time.sleep(2)
EOF

# 订阅者
cat > subscriber.py << 'EOF'
import redis
import json

r = redis.Redis(host='redis', decode_responses=True)
pubsub = r.pubsub()
pubsub.subscribe('events')

print("Waiting for messages...")
for message in pubsub.listen():
    if message['type'] == 'message':
        data = json.loads(message['data'])
        print(f"Received: {data}")
EOF
```

```yaml
cat > docker-compose-pubsub.yml << 'EOF'
version: '3.8'

services:
  redis:
    image: redis:7-alpine

  publisher:
    image: python:3.11-slim
    command: sh -c "pip install redis && python /app/publisher.py"
    volumes:
      - ./publisher.py:/app/publisher.py
    depends_on:
      - redis

  subscriber1:
    image: python:3.11-slim
    command: sh -c "pip install redis && python /app/subscriber.py"
    volumes:
      - ./subscriber.py:/app/subscriber.py
    depends_on:
      - redis

  subscriber2:
    image: python:3.11-slim
    command: sh -c "pip install redis && python /app/subscriber.py"
    volumes:
      - ./subscriber.py:/app/subscriber.py
    depends_on:
      - redis
EOF
```

```bash
docker compose -f docker-compose-pubsub.yml up
```

### Step 5.3: 服务网格模式（Envoy 代理）

```yaml
cat > docker-compose-mesh.yml << 'EOF'
version: '3.8'

services:
  # 带 sidecar 代理的服务
  app:
    build: .
    networks:
      - mesh

  # Envoy sidecar
  envoy-proxy:
    image: envoyproxy/envoy:v1.28-latest
    volumes:
      - ./envoy.yaml:/etc/envoy/envoy.yaml
    networks:
      - mesh
    command: /usr/local/bin/envoy -c /etc/envoy/envoy.yaml

  # 另一个服务
  api:
    build: ./api
    networks:
      - mesh

  # 它的 sidecar
  envoy-api:
    image: envoyproxy/envoy:v1.28-latest
    volumes:
      - ./envoy-api.yaml:/etc/envoy/envoy.yaml
    networks:
      - mesh

networks:
  mesh:
    driver: bridge
EOF
```

✅ **Checkpoint**: 你理解了 container 通信模式。

---

## Part 6: 负载均衡和高可用

### Step 6.1: Nginx 负载均衡

```bash
mkdir -p ~/docker-exercises/load-balancing
cd ~/docker-exercises/load-balancing

# 后端应用
cat > app.py << 'EOF'
from flask import Flask, jsonify
import socket
import os

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'server': socket.gethostname(),
        'ip': socket.gethostbyname(socket.gethostname())
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

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
RUN pip install -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
EOF

# 带负载均衡的 Nginx 配置
mkdir -p nginx
cat > nginx/nginx.conf << 'EOF'
upstream backend {
    least_conn;  # 负载均衡算法

    # 健康检查配置
    server backend1:5000 max_fails=3 fail_timeout=30s;
    server backend2:5000 max_fails=3 fail_timeout=30s;
    server backend3:5000 max_fails=3 fail_timeout=30s;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # 失败时重试
        proxy_next_upstream error timeout http_502 http_503 http_504;
    }

    location /health {
        access_log off;
        proxy_pass http://backend/health;
    }
}
EOF
```

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
      - backend1
      - backend2
      - backend3
    networks:
      - loadbalancer

  backend1:
    build: .
    networks:
      - loadbalancer

  backend2:
    build: .
    networks:
      - loadbalancer

  backend3:
    build: .
    networks:
      - loadbalancer

networks:
  loadbalancer:
    driver: bridge
EOF
```

```bash
# 启动栈
docker compose up -d

# 测试负载均衡 — 看到不同的服务器
for i in {1..10}; do
  curl -s http://localhost/ | grep server
  sleep 0.5
done

# 模拟故障
docker compose stop backend2

# 请求仍然有效（分发到健康的后端）
for i in {1..5}; do
  curl -s http://localhost/ | grep server
done

# 恢复
docker compose start backend2
```

### Step 6.2: HAProxy 负载均衡

```bash
# HAProxy 配置
cat > haproxy.cfg << 'EOF'
global
    log stdout format raw local0
    maxconn 4096

defaults
    log     global
    mode    http
    option  httplog
    option  dontlognull
    timeout connect 5000ms
    timeout client  50000ms
    timeout server  50000ms

frontend http_front
    bind *:80
    default_backend http_back

backend http_back
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200

    server backend1 backend1:5000 check
    server backend2 backend2:5000 check
    server backend3 backend3:5000 check

listen stats
    bind *:8080
    stats enable
    stats uri /stats
    stats refresh 10s
EOF
```

```yaml
cat > docker-compose-haproxy.yml << 'EOF'
version: '3.8'

services:
  haproxy:
    image: haproxy:2.8-alpine
    ports:
      - "80:80"
      - "8080:8080"  # 状态页面
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
    depends_on:
      - backend1
      - backend2
      - backend3

  backend1:
    build: .

  backend2:
    build: .

  backend3:
    build: .
EOF
```

```bash
# 使用 HAProxy 启动
docker compose -f docker-compose-haproxy.yml up -d

# 测试
for i in {1..10}; do curl -s http://localhost/ | grep server; done

# 查看状态
open http://localhost:8080/stats
```

✅ **Checkpoint**: 你可以配置带健康检查的负载均衡。

---

## Part 7: 外部连接

### Step 7.1: 端口发布模式

```bash
# 绑定到所有接口
docker run -d -p 8080:80 --name web-all nginx

# 绑定到特定接口（仅 localhost）
docker run -d -p 127.0.0.1:8081:80 --name web-local nginx

# 随机端口分配
docker run -d -p 80 --name web-random nginx
docker port web-random

# 多个端口映射
docker run -d \
  -p 8082:80 \
  -p 8443:443 \
  --name web-multi \
  nginx

# 检查发布的端口
docker port web-multi
```

### Step 7.2: 暴露服务

```yaml
cat > docker-compose-expose.yml << 'EOF'
version: '3.8'

services:
  # 对外暴露
  web:
    image: nginx
    ports:
      - "8080:80"  # host:container

  # 不对外暴露（仅内部）
  api:
    image: nginx
    expose:
      - "80"  # 对其他服务可见，不对主机

  # 内部服务
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
    # 无 ports 或 expose — 完全内部
EOF
```

### Step 7.3: 主机网络访问

```bash
# Container 访问主机服务
docker run --rm \
  --add-host host.docker.internal:host-gateway \
  alpine ping -c 3 host.docker.internal

# 用于从 container 访问主机的服务
```

✅ **Checkpoint**: 你理解了外部连接模式。

---

## Part 8: 网络调试和排错

### Step 8.1: 网络检查工具

```bash
# 列出所有网络
docker network ls

# 检查网络详情
docker network inspect bridge

# 查看哪些 container 在网络中
docker network inspect my-network \
  --format='{{range .Containers}}{{.Name}} {{end}}'

# Container 的网络设置
docker inspect container-name \
  --format='{{json .NetworkSettings}}' | python -m json.tool
```

### Step 8.2: 连接测试

```bash
# 创建测试网络
docker network create test-net

# 运行测试 container
docker run -d --name server --network test-net nginx
docker run -d --name client --network test-net alpine sleep 3600

# 测试 DNS 解析
docker exec client nslookup server

# 测试连接
docker exec client ping -c 3 server

# 测试端口连接
docker exec client nc -zv server 80

# HTTP 测试
docker exec client wget -qO- http://server

# 安装调试工具
docker exec client apk add --no-cache curl tcpdump bind-tools

# 高级调试
docker exec client tcpdump -i eth0 -n
```

### Step 8.3: 常见问题及解决方案

```bash
# 问题 1：DNS 不工作
# 解决方案：使用自定义 bridge 网络（而非默认 bridge）

# 问题 2：无法连接到 container
# 检查是否在同一网络上
docker inspect container1 --format='{{json .NetworkSettings.Networks}}'
docker inspect container2 --format='{{json .NetworkSettings.Networks}}'

# 问题 3：端口已被占用
# 检查哪个程序在使用该端口
docker ps -a
netstat -tlnp | grep :8080

# 问题 4：网络清理
docker network prune

# 问题 5：Container 无法访问互联网
# 检查 Docker daemon DNS 设置
cat /etc/docker/daemon.json
# 添加：{"dns": ["8.8.8.8", "8.8.4.4"]}
```

### Step 8.4: 网络监控

```bash
# 实时网络统计
docker stats

# 使用 tcpdump 查看网络流量
docker run --rm --net=host \
  nicolaka/netshoot \
  tcpdump -i any port 80

# 网络诊断 container
docker run -it --rm \
  --network container:target-container \
  nicolaka/netshoot

# 在 netshoot 内部，你有：
# - ping, nslookup, dig
# - curl, wget
# - tcpdump, iperf
# - netstat, ss
```

✅ **Checkpoint**: 你可以调试和排错网络问题。

---

## Part 9: ML 基础设施网络

### Step 9.1: 分布式训练网络

```yaml
cat > docker-compose-training.yml << 'EOF'
version: '3.8'

services:
  # 主节点
  master:
    image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
    command: python train.py --role master --rank 0
    volumes:
      - ./train.py:/workspace/train.py
      - model-data:/models
    networks:
      training-net:
        aliases:
          - master-node
    environment:
      - MASTER_ADDR=master-node
      - MASTER_PORT=29500
      - WORLD_SIZE=3

  # Worker 节点
  worker1:
    image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
    command: python train.py --role worker --rank 1
    volumes:
      - ./train.py:/workspace/train.py
      - model-data:/models
    networks:
      - training-net
    environment:
      - MASTER_ADDR=master-node
      - MASTER_PORT=29500
      - WORLD_SIZE=3
    depends_on:
      - master

  worker2:
    image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
    command: python train.py --role worker --rank 2
    volumes:
      - ./train.py:/workspace/train.py
      - model-data:/models
    networks:
      - training-net
    environment:
      - MASTER_ADDR=master-node
      - MASTER_PORT=29500
      - WORLD_SIZE=3
    depends_on:
      - master

networks:
  training-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16

volumes:
  model-data:
EOF
```

### Step 9.2: 带 A/B 测试的模型 Serving

```yaml
cat > docker-compose-ab-testing.yml << 'EOF'
version: '3.8'

services:
  # 带 A/B 路由的负载均衡器
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx-ab.conf:/etc/nginx/conf.d/default.conf
    networks:
      - serving-net

  # 模型版本 A（80% 流量）
  model-v1:
    build:
      context: .
      args:
        MODEL_VERSION: v1
    deploy:
      replicas: 4
    networks:
      serving-net:
        aliases:
          - model-a

  # 模型版本 B（20% 流量）
  model-v2:
    build:
      context: .
      args:
        MODEL_VERSION: v2
    deploy:
      replicas: 1
    networks:
      serving-net:
        aliases:
          - model-b

  # 指标收集
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    networks:
      - serving-net

networks:
  serving-net:
    driver: bridge
EOF
```

**Nginx A/B Config**:
```nginx
# nginx-ab.conf
split_clients "${remote_addr}" $backend {
    80%     model-a;
    20%     model-b;
}

upstream model-a {
    server model-v1:8000;
}

upstream model-b {
    server model-v2:8000;
}

server {
    listen 80;
    location / {
        proxy_pass http://$backend;
    }
}
```

### Step 9.3: 特征存储架构

```yaml
cat > docker-compose-features.yml << 'EOF'
version: '3.8'

services:
  # 在线特征 serving
  feature-server:
    build: ./feature-server
    networks:
      - online-tier
      - cache-tier
    depends_on:
      - redis
      - postgres

  # 快速缓存层
  redis:
    image: redis:7-alpine
    networks:
      - cache-tier

  # 特征数据库
  postgres:
    image: postgres:15
    networks:
      - cache-tier
      - storage-tier
    volumes:
      - features-db:/var/lib/postgresql/data

  # 离线特征计算
  spark-master:
    image: bitnami/spark:3.4
    networks:
      - storage-tier
      - compute-tier
    environment:
      - SPARK_MODE=master

  spark-worker:
    image: bitnami/spark:3.4
    networks:
      - compute-tier
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
    deploy:
      replicas: 2

networks:
  online-tier:     # API 访问
  cache-tier:      # 快速特征访问
  storage-tier:    # 持久存储
  compute-tier:    # 批处理

volumes:
  features-db:
EOF
```

✅ **Checkpoint**: 你可以设计 ML 基础设施网络。

---

## Part 10: 生产网络模式

### Step 10.1: 零停机部署

```yaml
cat > docker-compose-rolling.yml << 'EOF'
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx-rolling.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app-blue
      - app-green

  # 蓝色部署（当前）
  app-blue:
    build: .
    environment:
      - VERSION=blue
    networks:
      - app-net
    deploy:
      replicas: 3

  # 绿色部署（新版本）
  app-green:
    build: .
    environment:
      - VERSION=green
    networks:
      - app-net
    deploy:
      replicas: 3

networks:
  app-net:
EOF
```

### Step 10.2: 断路器模式

```python
# circuit_breaker.py
cat > circuit_breaker.py << 'EOF'
import requests
import time
from enum import Enum

class CircuitState(Enum):
    CLOSED = 1   # 正常运行
    OPEN = 2     # 检测到故障，阻止请求
    HALF_OPEN = 3  # 测试服务是否恢复

class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_count = 0
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.state = CircuitState.CLOSED
        self.last_failure_time = None

    def call(self, url):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time > self.timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")

        try:
            response = requests.get(url, timeout=2)
            response.raise_for_status()

            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.CLOSED
                self.failure_count = 0

            return response

        except Exception as e:
            self.failure_count += 1
            self.last_failure_time = time.time()

            if self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN

            raise e

# 使用
breaker = CircuitBreaker()
try:
    response = breaker.call('http://api:5000/data')
except Exception as e:
    print(f"Service unavailable: {e}")
EOF
```

### Step 10.3: 使用 Istio 的服务网格（概念）

```yaml
# Istio 将处理：
# - 服务间加密（mTLS）
# - 流量管理（路由、重试、超时）
# - 可观测性（指标、日志、追踪）
# - 断路和故障注入

# 带 Istio sidecar 模式的示例服务
cat > docker-compose-sidecar.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    networks:
      - mesh

  # Envoy sidecar 代理
  envoy:
    image: envoyproxy/envoy:v1.28-latest
    volumes:
      - ./envoy-config.yaml:/etc/envoy/envoy.yaml
    networks:
      - mesh
    # 所有流量通过 Envoy

networks:
  mesh:
EOF
```

✅ **Checkpoint**: 你理解了生产网络模式。

---

## 实际挑战

### Challenge 1: 安全多层应用

**Requirements**:
- 3 层架构（web、api、database）
- 层之间的网络隔离
- 仅允许必要的连接
- 数据库无互联网访问
- API 层负载均衡

### Challenge 2: 微服务通信

**Requirements**:
- 5 个微服务
- 服务发现
- 一个服务可公开访问
- 内部服务网格
- 健康检查和断路器

### Challenge 3: ML Pipeline 网络

**Requirements**:
- 训练集群（主节点 + worker）
- 特征存储（在线 + 离线）
- Model serving（A/B 测试）
- 监控和日志记录
- 适当的网络分段

---

## Summary

**What You Accomplished**:
✅ 掌握了 Docker 网络驱动（bridge、host、overlay、none）
✅ 实现了服务发现和 DNS 解析
✅ 为实现安全性创建了网络隔离
✅ 配置了带有特定 IP 范围的自定义网络
✅ 实现了带健康检查的负载均衡
✅ 调试了网络连接问题
✅ 设计了 ML 基础设施网络
✅ 应用了生产网络模式

**Key Networking Concepts**:
- Bridge 网络和 DNS
- 网络隔离和安全
- 负载均衡算法
- 服务发现模式
- 断路器和弹性
- 网络排错
- 生产就绪架构

---

## Next Steps

**Continue Learning**:
- **Exercise 05**: Volume Management and Data Persistence
- **Exercise 06**: Production Deployment Strategies
- **Exercise 07**: Complete ML Application Containerization

---

**Exercise Version**: 1.0
**Last Updated**: October 2025
**Estimated Time**: 3-4 小时
**Difficulty**: 中级到高级
