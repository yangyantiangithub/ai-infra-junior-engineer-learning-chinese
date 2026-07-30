# Exercise 05: Docker Volume Management

## Exercise Overview

**Objective**: 掌握 Docker volume 管理，包括持久化存储、container 间数据共享、备份策略以及适用于 ML 工作负载的 volume 驱动。

**Difficulty**: 中级
**Estimated Time**: 3-4 小时
**Prerequisites**:
- Exercise 01-04 (Container Operations, Images, Compose, Networking)
- Lecture 04 (Networking & Volumes)

**What You'll Learn**:
- Volume 类型（命名 volume、bind mount、tmpfs）
- 数据持久化策略
- Container 间数据共享
- Volume 备份和恢复
- Volume 驱动和插件
- ML 模型和数据集管理
- 性能考量
- 生产 volume 模式

---

## Part 1: 理解 Volume 类型

### Step 1.1: 命名 Volume

```bash
# 创建项目目录
mkdir -p ~/docker-exercises/volumes
cd ~/docker-exercises/volumes

# 创建命名 volume
docker volume create my-data

# 检查
docker volume inspect my-data

# 使用 volume
docker run -d \
  --name app1 \
  -v my-data:/data \
  alpine sh -c "echo 'Hello from app1' > /data/message.txt && sleep 3600"

# 检查文件已创建
docker exec app1 cat /data/message.txt

# 停止并删除 container
docker stop app1
docker rm app1

# 数据持久存在！使用相同 volume 启动新 container
docker run -d \
  --name app2 \
  -v my-data:/data \
  alpine sleep 3600

# 文件仍然存在！
docker exec app2 cat /data/message.txt

# 清理
docker rm -f app2
docker volume rm my-data
```

**Key Points**:
- 命名 volume 在 container 删除后仍然存在
- 由 Docker 管理（存储在 `/var/lib/docker/volumes/`）
- 可在 container 之间共享
- 最适合生产数据

### Step 1.2: Bind Mount

```bash
# 在主机上创建目录
mkdir -p ~/docker-data/app

# 创建文件
echo "Hello from host" > ~/docker-data/app/host-file.txt

# 将主机目录挂载到 container
docker run -d \
  --name bind-test \
  -v ~/docker-data/app:/app \
  alpine sleep 3600

# 文件在 container 中可访问
docker exec bind-test cat /app/host-file.txt

# 从 container 创建文件
docker exec bind-test sh -c "echo 'From container' > /app/container-file.txt"

# 文件出现在主机上！
cat ~/docker-data/app/container-file.txt

# 清理
docker rm -f bind-test
```

**Use Cases**:
- 开发（热重载源代码）
- 配置文件
- 主机生成的数据
- 需要直接文件系统访问

### Step 1.3: tmpfs Mount

```bash
# 创建 tmpfs mount（基于内存，不持久化）
docker run -d \
  --name tmpfs-test \
  --tmpfs /tmp:rw,size=100m \
  alpine sleep 3600

# 写入 tmpfs
docker exec tmpfs-test sh -c "echo 'Temporary data' > /tmp/temp.txt"
docker exec tmpfs-test cat /tmp/temp.txt

# 检查挂载
docker exec tmpfs-test df -h /tmp

# 停止并删除 container
docker rm -f tmpfs-test

# 启动新 container — 数据消失（tmpfs 不持久化）
docker run -d --name tmpfs-test2 --tmpfs /tmp alpine sleep 3600
docker exec tmpfs-test2 ls /tmp
# 空！

docker rm -f tmpfs-test2
```

**Use Cases**:
- 临时缓存
- 敏感数据（密码、令牌）
- 高性能临时存储
- ML 推理临时文件

### Step 1.4: 比较

```bash
# 创建测试脚本
cat > test-volumes.sh << 'EOF'
#!/bin/bash

# 命名 volume
docker volume create test-named
docker run --rm -v test-named:/data alpine sh -c "echo 'named' > /data/file.txt"

# Bind mount
mkdir -p /tmp/test-bind
docker run --rm -v /tmp/test-bind:/data alpine sh -c "echo 'bind' > /data/file.txt"

# tmpfs
docker run --rm --tmpfs /data alpine sh -c "echo 'tmpfs' > /data/file.txt"

echo "Named volume file:"
docker run --rm -v test-named:/data alpine cat /data/file.txt

echo "Bind mount file:"
cat /tmp/test-bind/file.txt

echo "tmpfs file: (not persisted)"

# 清理
docker volume rm test-named
rm -rf /tmp/test-bind
EOF

chmod +x test-volumes.sh
./test-volumes.sh
```

✅ **Checkpoint**: 你理解了三种 Docker volume 类型。

---

## Part 2: Volume 管理操作

### Step 2.1: 基本 Volume 命令

```bash
# 创建 volume
docker volume create app-data
docker volume create db-data
docker volume create cache-data

# 列出 volume
docker volume ls

# 检查特定 volume
docker volume inspect app-data

# 查找 volume 在主机上的存储位置
docker volume inspect app-data --format '{{.Mountpoint}}'

# 删除 volume
docker volume rm cache-data

# 删除所有未使用的 volume
docker volume prune

# 强制删除
docker volume rm -f app-data db-data
```

### Step 2.2: Volume 标签和元数据

```bash
# 创建带标签的 volume
docker volume create \
  --label environment=production \
  --label app=myapp \
  --label version=1.0 \
  prod-data

# 列出具有特定标签的 volume
docker volume ls --filter "label=environment=production"

# 检查标签
docker volume inspect prod-data --format '{{json .Labels}}' | python -m json.tool

# 清理
docker volume rm prod-data
```

### Step 2.3: Volume 驱动

```bash
# 默认驱动（local）
docker volume create --driver local my-local-volume

# 使用驱动选项创建 volume
docker volume create \
  --driver local \
  --opt type=tmpfs \
  --opt device=tmpfs \
  --opt o=size=100m,uid=1000 \
  tmpfs-volume

# 检查驱动信息
docker volume inspect tmpfs-volume

# 清理
docker volume rm my-local-volume tmpfs-volume
```

✅ **Checkpoint**: 你可以使用 Docker CLI 管理 volume。

---

## Part 3: Container 间共享数据

### Step 3.1: 多个 Container，一个 Volume

```bash
# 创建共享 volume
docker volume create shared-data

# Container 1：写入
docker run -d \
  --name writer \
  -v shared-data:/data \
  alpine sh -c 'while true; do echo "$(date): Message" >> /data/log.txt; sleep 5; done'

# Container 2：读取
docker run -d \
  --name reader \
  -v shared-data:/data \
  alpine sh -c 'while true; do tail -5 /data/log.txt; sleep 10; done'

# 查看 writer 日志
docker logs -f writer

# 查看 reader 日志（在另一个终端）
docker logs -f reader

# 两个 container 共享相同的数据！

# 清理
docker rm -f writer reader
docker volume rm shared-data
```

### Step 3.2: 数据 Container 模式（旧版）

```bash
# 创建纯数据 container（旧模式）
docker create \
  --name data-container \
  -v /data \
  alpine

# 从数据 container 使用数据
docker run -d \
  --name app1 \
  --volumes-from data-container \
  alpine sh -c "echo 'App 1' > /data/app1.txt && sleep 3600"

docker run -d \
  --name app2 \
  --volumes-from data-container \
  alpine sh -c "echo 'App 2' > /data/app2.txt && sleep 3600"

# 检查两个文件都存在
docker exec app1 ls /data
docker exec app2 ls /data

# 清理
docker rm -f data-container app1 app2
```

**Note**: 现代方法改用命名 volume。

### Step 3.3: Docker Compose Volume 共享

```yaml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # 日志生成器
  app:
    image: alpine
    command: sh -c 'while true; do echo "$(date): Application log" >> /logs/app.log; sleep 2; done'
    volumes:
      - app-logs:/logs

  # 日志处理器
  processor:
    image: alpine
    command: sh -c 'while true; do echo "=== Last 5 logs ===" && tail -5 /logs/app.log; sleep 10; done'
    volumes:
      - app-logs:/logs
    depends_on:
      - app

  # 日志分析器
  analyzer:
    image: alpine
    command: sh -c 'while true; do wc -l /logs/app.log; sleep 15; done'
    volumes:
      - app-logs:/logs:ro  # 只读挂载！
    depends_on:
      - app

volumes:
  app-logs:
EOF
```

```bash
# 启动服务
docker compose up -d

# 查看日志
docker compose logs -f processor

# 验证分析器有只读访问权限
docker compose exec analyzer sh -c "echo 'test' >> /logs/app.log"
# 应该失败：Read-only file system

# 清理
docker compose down -v
```

✅ **Checkpoint**: 你可以共享多个 container 之间的 volume。

---

## Part 4: 数据库持久化

### Step 4.1: 带持久化存储的 PostgreSQL

```bash
mkdir -p ~/docker-exercises/db-persistence
cd ~/docker-exercises/db-persistence

# 创建数据库 volume
docker volume create postgres-data

# 使用持久化 volume 运行 PostgreSQL
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

# 等待启动
sleep 10

# 创建表并插入数据
docker exec postgres psql -U postgres -d myapp -c "
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);
INSERT INTO users (name, email) VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com');
"

# 查询数据
docker exec postgres psql -U postgres -d myapp -c "SELECT * FROM users;"

# 停止并删除 container
docker stop postgres
docker rm postgres

# 使用相同 volume 启动新 container
docker run -d \
  --name postgres-new \
  -e POSTGRES_PASSWORD=secret \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15

sleep 10

# 数据持久存在！
docker exec postgres-new psql -U postgres -d myapp -c "SELECT * FROM users;"

# 清理
docker rm -f postgres-new
docker volume rm postgres-data
```

### Step 4.2: 使用 Compose 的完整数据库栈

```yaml
cat > docker-compose-db.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_USER=appuser
      - POSTGRES_PASSWORD=apppass
      - POSTGRES_DB=appdb
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-db:/docker-entrypoint-initdb.d
      - ./backups:/backups
    ports:
      - "5432:5432"

  mongo:
    image: mongo:7
    environment:
      - MONGO_INITDB_ROOT_USERNAME=admin
      - MONGO_INITDB_ROOT_PASSWORD=secret
    volumes:
      - mongo-data:/data/db
      - ./mongo-init:/docker-entrypoint-initdb.d
    ports:
      - "27017:27017"

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"

volumes:
  postgres-data:
  mongo-data:
  redis-data:
EOF
```

```bash
# 创建初始化脚本
mkdir -p init-db backups mongo-init

cat > init-db/01-schema.sql << 'EOF'
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200),
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products (name, price) VALUES
    ('Widget', 19.99),
    ('Gadget', 29.99);
EOF

# 启动所有数据库
docker compose -f docker-compose-db.yml up -d

# 等待启动
sleep 15

# 验证数据
docker compose -f docker-compose-db.yml exec postgres \
  psql -U appuser -d appdb -c "SELECT * FROM products;"

# 测试 MongoDB
docker compose -f docker-compose-db.yml exec mongo \
  mongosh -u admin -p secret --eval "db.adminCommand('listDatabases')"

# 测试 Redis 持久化
docker compose -f docker-compose-db.yml exec redis \
  redis-cli SET mykey "persistent value"

docker compose -f docker-compose-db.yml exec redis \
  redis-cli GET mykey

# 停止并重启 — 数据持久存在
docker compose -f docker-compose-db.yml down
docker compose -f docker-compose-db.yml up -d

sleep 15

# 数据仍然存在！
docker compose -f docker-compose-db.yml exec redis redis-cli GET mykey

# 清理
docker compose -f docker-compose-db.yml down -v
```

✅ **Checkpoint**: 你可以实现持久化数据库存储。

---

## Part 5: 备份和恢复策略

### Step 5.1: Volume 备份

```bash
# 创建带数据的 volume
docker volume create app-data

# 添加一些数据
docker run --rm -v app-data:/data alpine sh -c "
  echo 'Important data' > /data/file1.txt
  echo 'More data' > /data/file2.txt
  mkdir -p /data/subdir
  echo 'Nested data' > /data/subdir/file3.txt
"

# 将 volume 备份为 tar 归档
docker run --rm \
  -v app-data:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/app-data-backup.tar.gz -C /source .

# 验证备份
ls -lh app-data-backup.tar.gz

# 列出内容
tar tzf app-data-backup.tar.gz

# 删除 volume
docker volume rm app-data

# 从备份恢复
docker volume create app-data-restored

docker run --rm \
  -v app-data-restored:/target \
  -v $(pwd):/backup \
  alpine tar xzf /backup/app-data-backup.tar.gz -C /target

# 验证恢复的数据
docker run --rm -v app-data-restored:/data alpine ls -laR /data

# 清理
docker volume rm app-data-restored
rm app-data-backup.tar.gz
```

### Step 5.2: 数据库备份脚本

```bash
cat > backup-database.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
CONTAINER_NAME="postgres"
DB_NAME="myapp"

# 创建备份
docker exec $CONTAINER_NAME pg_dump -U postgres $DB_NAME > \
  ${BACKUP_DIR}/backup_${TIMESTAMP}.sql

# 压缩
gzip ${BACKUP_DIR}/backup_${TIMESTAMP}.sql

# 仅保留最近 7 天
find ${BACKUP_DIR} -name "backup_*.sql.gz" -mtime +7 -delete

echo "Backup completed: backup_${TIMESTAMP}.sql.gz"
EOF

chmod +x backup-database.sh
```

### Step 5.3: 使用 Compose 自动备份

```yaml
cat > docker-compose-backup.yml << 'EOF'
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD=secret
      - POSTGRES_DB=myapp
    volumes:
      - db-data:/var/lib/postgresql/data

  backup:
    image: postgres:15
    depends_on:
      - db
    volumes:
      - ./backups:/backups
      - db-data:/var/lib/postgresql/data:ro
    command: |
      bash -c '
      while true; do
        TIMESTAMP=$$(date +%Y%m%d_%H%M%S)
        pg_dump -h db -U postgres myapp > /backups/backup_$${TIMESTAMP}.sql
        gzip /backups/backup_$${TIMESTAMP}.sql
        echo "Backup created: backup_$${TIMESTAMP}.sql.gz"
        sleep 86400  # 每天备份
      done
      '
    environment:
      - PGPASSWORD=secret

volumes:
  db-data:
EOF
```

```bash
mkdir -p backups
docker compose -f docker-compose-backup.yml up -d

# 手动触发备份
docker compose -f docker-compose-backup.yml exec backup \
  sh -c 'pg_dump -h db -U postgres myapp | gzip > /backups/manual_backup.sql.gz'

# 列出备份
ls -lh backups/

# 清理
docker compose -f docker-compose-backup.yml down -v
```

### Step 5.4: 从备份恢复

```bash
# 创建数据库
docker run -d \
  --name postgres-restore \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=myapp \
  -v postgres-restore-data:/var/lib/postgresql/data \
  postgres:15

sleep 10

# 从备份恢复
gunzip -c backups/manual_backup.sql.gz | \
  docker exec -i postgres-restore psql -U postgres -d myapp

# 验证数据
docker exec postgres-restore psql -U postgres -d myapp -c "\dt"

# 清理
docker rm -f postgres-restore
docker volume rm postgres-restore-data
```

✅ **Checkpoint**: 你可以备份和恢复 Docker volume。

---

## Part 6: ML 模型和数据集管理

### Step 6.1: 模型存储 Volume

```bash
mkdir -p ~/docker-exercises/ml-volumes
cd ~/docker-exercises/ml-volumes

# 创建模型存储
docker volume create model-store

# 下载并存储模型
cat > download_model.py << 'EOF'
import torch
import torchvision.models as models
import os

# 下载模型
print("Downloading ResNet18...")
model = models.resnet18(pretrained=True)

# 保存到 volume
model_path = '/models/resnet18.pth'
torch.save(model.state_dict(), model_path)
print(f"Model saved to {model_path}")

# 验证
size = os.path.getsize(model_path)
print(f"Model size: {size / 1024 / 1024:.2f} MB")
EOF

# 运行下载
docker run --rm \
  -v model-store:/models \
  pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime \
  sh -c "pip install torchvision && python -c \"$(cat download_model.py)\""

# 验证模型存在
docker run --rm \
  -v model-store:/models \
  alpine ls -lh /models
```

### Step 6.2: 数据集管理

```yaml
cat > docker-compose-ml.yml << 'EOF'
version: '3.8'

services:
  # 数据预处理
  preprocessor:
    image: python:3.11
    volumes:
      - raw-data:/data/raw:ro  # 只读
      - processed-data:/data/processed  # 写入
      - ./scripts:/scripts
    command: python /scripts/preprocess.py
    environment:
      - INPUT_DIR=/data/raw
      - OUTPUT_DIR=/data/processed

  # 训练
  trainer:
    image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
    volumes:
      - processed-data:/data:ro
      - models:/models
      - ./train.py:/workspace/train.py
    command: python /workspace/train.py
    environment:
      - DATA_DIR=/data
      - MODEL_DIR=/models

  # 推理服务器
  inference:
    image: pytorch/pytorch:2.0.1-cuda11.7-cudnn8-runtime
    volumes:
      - models:/models:ro  # 只读模型
      - inference-cache:/cache
    ports:
      - "8000:8000"
    environment:
      - MODEL_PATH=/models/best_model.pth
      - CACHE_DIR=/cache

volumes:
  raw-data:
  processed-data:
  models:
  inference-cache:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs
      o: size=1g
EOF
```

### Step 6.3: 共享模型注册中心

```yaml
cat > docker-compose-registry.yml << 'EOF'
version: '3.8'

services:
  # 模型注册中心存储
  registry:
    image: postgres:15
    environment:
      - POSTGRES_DB=model_registry
      - POSTGRES_PASSWORD=secret
    volumes:
      - registry-db:/var/lib/postgresql/data

  # 模型文件存储
  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - model-storage:/data
    environment:
      - MINIO_ROOT_USER=admin
      - MINIO_ROOT_PASSWORD=secretkey

  # 模型版本追踪
  mlflow:
    image: ghcr.io/mlflow/mlflow:latest
    ports:
      - "5000:5000"
    volumes:
      - mlflow-data:/mlflow
    environment:
      - MLFLOW_BACKEND_STORE_URI=postgresql://postgres:secret@registry/model_registry
      - MLFLOW_DEFAULT_ARTIFACT_ROOT=/mlflow/artifacts
    depends_on:
      - registry

volumes:
  registry-db:
  model-storage:
  mlflow-data:
EOF
```

```bash
# 启动模型注册中心
docker compose -f docker-compose-registry.yml up -d

# 访问 MLflow UI
echo "MLflow: http://localhost:5000"
echo "MinIO: http://localhost:9001"

# 清理
docker compose -f docker-compose-registry.yml down -v
```

✅ **Checkpoint**: 你可以使用 volume 管理 ML 模型和数据集。

---

## Part 7: 性能考量

### Step 7.1: Volume 性能比较

```bash
# 创建测试脚本
cat > volume-performance.sh << 'EOF'
#!/bin/bash

echo "Testing volume performance..."

# 测试 1：命名 volume
echo "1. Named volume:"
docker volume create perf-named
time docker run --rm \
  -v perf-named:/data \
  alpine sh -c "dd if=/dev/zero of=/data/test bs=1M count=100"
docker volume rm perf-named

# 测试 2：Bind mount
echo "2. Bind mount:"
mkdir -p /tmp/perf-bind
time docker run --rm \
  -v /tmp/perf-bind:/data \
  alpine sh -c "dd if=/dev/zero of=/data/test bs=1M count=100"
rm -rf /tmp/perf-bind

# 测试 3：tmpfs
echo "3. tmpfs:"
time docker run --rm \
  --tmpfs /data:rw,size=200m \
  alpine sh -c "dd if=/dev/zero of=/data/test bs=1M count=100"

echo "Performance test complete!"
EOF

chmod +x volume-performance.sh
./volume-performance.sh
```

### Step 7.2: 用于性能的只读挂载

```yaml
cat > docker-compose-readonly.yml << 'EOF'
version: '3.8'

services:
  # 提供静态文件的 Web 服务器
  web:
    image: nginx:alpine
    volumes:
      # 静态文件只读（更好的性能）
      - ./static:/usr/share/nginx/html:ro
      # 配置只读
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      # 日志可写
      - logs:/var/log/nginx
    ports:
      - "80:80"

  # 使用只读模型的 ML 推理
  inference:
    image: pytorch/pytorch
    volumes:
      - models:/models:ro  # 模型从不更改
      - cache:/cache  # 可写缓存
    environment:
      - MODEL_DIR=/models

volumes:
  logs:
  models:
  cache:
    driver: local
    driver_opts:
      type: tmpfs
      device: tmpfs
      o: size=2g
EOF
```

### Step 7.3: 用于性能的 Volume 选项

```bash
# 使用性能选项创建 volume
docker volume create \
  --driver local \
  --opt type=tmpfs \
  --opt device=tmpfs \
  --opt o=size=1g,uid=1000 \
  fast-cache

# 使用特定挂载选项创建 volume
docker volume create \
  --driver local \
  --opt type=none \
  --opt device=/mnt/fast-ssd \
  --opt o=bind \
  ssd-storage

# 在 container 中使用
docker run -d \
  --name perf-app \
  -v fast-cache:/cache \
  -v ssd-storage:/data \
  alpine sleep 3600

# 清理
docker rm -f perf-app
docker volume rm fast-cache ssd-storage
```

✅ **Checkpoint**: 你理解了 volume 性能特性。

---

## Part 8: Volume 驱动和插件

### Step 8.1: NFS Volume 驱动

```bash
# 安装 NFS 工具（如果需要）
# sudo apt-get install nfs-common

# 创建 NFS volume（需要 NFS 服务器）
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/path/to/nfs/share \
  nfs-volume

# 使用 NFS volume
docker run -d \
  --name nfs-app \
  -v nfs-volume:/data \
  alpine sleep 3600

# 多个 container 可以在跨主机共享 NFS volume
```

### Step 8.2: 云存储 Volume（概念）

```bash
# AWS EFS volume（示例）
# 需要 docker-volume-efs 插件

# 安装插件
# docker plugin install rexray/efs EFS_REGION=us-east-1

# 创建 EFS volume
# docker volume create \
#   --driver rexray/efs \
#   --name efs-volume \
#   --opt fileSystemId=fs-12345678

# 在 container 中使用
# docker run -v efs-volume:/data myapp
```

### Step 8.3: 自定义 Volume 插件

```yaml
# 示例：使用第三方 volume 插件实现分布式存储
cat > docker-compose-distributed.yml << 'EOF'
version: '3.8'

services:
  app1:
    image: alpine
    volumes:
      - shared-storage:/data
    command: sh -c 'echo "App1: $(hostname)" >> /data/log.txt && sleep 3600'

  app2:
    image: alpine
    volumes:
      - shared-storage:/data
    command: sh -c 'echo "App2: $(hostname)" >> /data/log.txt && sleep 3600'

volumes:
  shared-storage:
    driver: local  # 在生产中使用分布式驱动
    # 示例：
    # driver: rexray/efs  # AWS EFS
    # driver: pxd  # Portworx
    # driver: netshare:nfs  # NFS
EOF
```

✅ **Checkpoint**: 你理解了 volume 驱动和插件。

---

## Part 9: 生产模式

### Step 9.1: Volume 生命周期管理

```yaml
cat > docker-compose-lifecycle.yml << 'EOF'
version: '3.8'

services:
  app:
    image: myapp
    volumes:
      # 应用数据（持久化）
      - app-data:/data
      # 配置（外部）
      - ./config:/config:ro
      # 日志（外部管理）
      - /var/log/app:/logs
      # 缓存（临时）
      - type: tmpfs
        target: /cache
        tmpfs:
          size: 500m

volumes:
  app-data:
    driver: local
    labels:
      backup: "daily"
      retention: "30d"
EOF
```

### Step 9.2: 多环境 Volume 策略

```yaml
# 开发环境
cat > docker-compose.dev.yml << 'EOF'
version: '3.8'

services:
  app:
    image: myapp
    volumes:
      # 用于热重载的 bind mount
      - ./src:/app/src
      - ./tests:/app/tests
      # 用于依赖的命名 volume
      - node_modules:/app/node_modules

volumes:
  node_modules:
EOF

# 生产环境
cat > docker-compose.prod.yml << 'EOF'
version: '3.8'

services:
  app:
    image: myapp
    volumes:
      # 仅命名 volume（无 bind mount）
      - app-data:/app/data
      - app-logs:/app/logs
    deploy:
      replicas: 3

volumes:
  app-data:
    driver: rexray/efs  # 分布式存储
  app-logs:
    driver: syslog
EOF
```

### Step 9.3: 灾难恢复设置

```bash
# 创建灾难恢复脚本
cat > dr-backup.sh << 'EOF'
#!/bin/bash

VOLUMES=("db-data" "app-data" "models")
BACKUP_PATH="/backups/dr"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

mkdir -p ${BACKUP_PATH}

for vol in "${VOLUMES[@]}"; do
    echo "Backing up ${vol}..."

    docker run --rm \
        -v ${vol}:/source:ro \
        -v ${BACKUP_PATH}:/backup \
        alpine \
        tar czf /backup/${vol}_${TIMESTAMP}.tar.gz -C /source .

    # 上传到 S3 或远程存储
    # aws s3 cp ${BACKUP_PATH}/${vol}_${TIMESTAMP}.tar.gz s3://my-backups/
done

echo "Disaster recovery backup complete!"
EOF

chmod +x dr-backup.sh
```

✅ **Checkpoint**: 你可以实现生产 volume 模式。

---

## Part 10: 排错和最佳实践

### Step 10.1: 常见问题

```bash
# 问题 1：权限被拒绝
# 解决方案：检查文件所有权

# 创建 volume
docker volume create test-perms

# 以非 root 用户运行
docker run --rm \
  -u 1000:1000 \
  -v test-perms:/data \
  alpine sh -c "echo 'test' > /data/file.txt"

# 如果权限被拒绝，修复所有权：
docker run --rm \
  -v test-perms:/data \
  alpine chown -R 1000:1000 /data

# 问题 2：Volume 未找到
docker volume ls
docker volume inspect volume-name

# 问题 3：Volume 已满
docker system df -v
docker volume ls -qf dangling=true | xargs docker volume rm

# 问题 4：性能慢
# 检查 volume 驱动和挂载选项
docker volume inspect volume-name --format '{{.Driver}}'
```

### Step 10.2: Volume 最佳实践

```yaml
cat > docker-compose-best-practices.yml << 'EOF'
version: '3.8'

services:
  app:
    image: myapp
    volumes:
      # ✓ 使用命名 volume 存储数据
      - app-data:/app/data

      # ✓ 尽可能使用只读
      - ./config:/config:ro

      # ✓ tmpfs 用于临时文件
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100m

      # ✓ 指定具体路径，而非根目录
      - logs:/app/logs

      # ✗ 避免 bind mount 根目录
      # - ./:/app  # 不好的做法！

volumes:
  app-data:
    # ✓ 添加标签以便管理
    labels:
      com.example.backup: "true"
      com.example.environment: "production"
  logs:
    # ✓ 设置驱动选项
    driver: local
    driver_opts:
      type: none
      device: /var/log/myapp
      o: bind
EOF
```

### Step 10.3: 监控 Volume 使用

```bash
# 创建监控脚本
cat > monitor-volumes.sh << 'EOF'
#!/bin/bash

echo "Docker Volume Usage Report"
echo "=========================="
echo

# 总体系统使用
echo "System Overview:"
docker system df

echo
echo "Volume Details:"
docker volume ls --format "table {{.Name}}\t{{.Driver}}\t{{.Scope}}"

echo
echo "Volume Sizes:"
for vol in $(docker volume ls -q); do
    size=$(docker run --rm -v $vol:/data alpine du -sh /data 2>/dev/null | awk '{print $1}')
    echo "$vol: $size"
done

echo
echo "Dangling Volumes:"
docker volume ls -qf dangling=true
EOF

chmod +x monitor-volumes.sh
./monitor-volumes.sh
```

✅ **Checkpoint**: 你可以排错并遵循 volume 最佳实践。

---

## 实际挑战

### Challenge 1: 完整的备份系统

**Requirements**:
- 所有 volume 的自动每日备份
- 保留策略（保留最近 7 天）
- 备份验证
- 记录恢复过程
- 异地备份到 S3/云存储

### Challenge 2: ML Pipeline 存储

**Requirements**:
- 原始数据 volume（pipeline 只读）
- 处理后数据 volume
- 带版本管理的模型存储
- 实验追踪存储
- 推理缓存
- 所有内容具有适当的性能设置

### Challenge 3: 多区域部署

**Requirements**:
- 跨区域复制的数据库 volume
- 共享只读配置
- 区域特定缓存
- 集中式日志存储
- 灾难恢复策略

---

## Summary

**What You Accomplished**:
✅ 掌握了三种 volume 类型（命名、bind mount、tmpfs）
✅ 管理了 volume 生命周期操作
✅ 实现了 container 之间数据共享
✅ 创建了持久化数据库存储
✅ 构建了备份和恢复策略
✅ 管理了 ML 模型和数据集
✅ 优化了 volume 性能
✅ 使用了 volume 驱动和插件
✅ 应用了生产 volume 模式
✅ 排错了 volume 问题

**Key Concepts Mastered**:
- Volume 类型和用例
- 数据持久化策略
- 备份和灾难恢复
- 性能优化
- 生产最佳实践
- ML 特定 volume 模式

**Volume Strategy**:
- **Development**: 用于热重载的 bind mount
- **Staging**: 带备份的命名 volume
- **Production**: 带复制的分布式 volume

---

## Next Steps

**Continue Learning**:
- **Exercise 06**: Production Deployment Strategies
- **Exercise 07**: Complete ML Application Containerization

---

**Exercise Version**: 1.0
**Last Updated**: October 2025
**Estimated Time**: 3-4 小时
**Difficulty**: 中级
