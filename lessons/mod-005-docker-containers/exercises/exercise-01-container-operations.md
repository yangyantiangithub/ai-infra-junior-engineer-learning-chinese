# Exercise 01: Docker Container Operations

## Exercise Overview

**Objective**: 掌握基本的 Docker container 操作，包括运行、管理和调试 container。

**Difficulty**: 初级
**Estimated Time**: 2-3 小时
**Prerequisites**: Lecture 01 (Docker Fundamentals)

**What You'll Learn**:
- 在不同模式下运行 container
- 管理 container 生命周期
- 查看日志和调试
- 端口映射和网络基础
- 环境变量
- 资源约束
- Container 检查

---

## Part 1: Environment Setup

### Step 1.1: Verify Docker Installation

```bash
# 检查 Docker 版本
docker --version

# 检查 Docker 信息
docker info

# 使用 hello-world 测试
docker run hello-world
```

**Expected Output**:
```
Docker version 24.0.6, build ed223bc
...
Hello from Docker!
```

✅ **Checkpoint**: Docker 已安装并正在运行。

---

## Part 2: Running Containers

### Step 2.1: Run Your First Container

```bash
# 在前台运行 nginx
docker run nginx

# 按 Ctrl+C 停止

# 发生了什么？
# 1. 下载 nginx image（如果不存在）
# 2. 从 image 创建 container
# 3. 启动 nginx 进程
# 4. 附加到 container 输出
```

### Step 2.2: Detached Mode

```bash
# 在后台运行（-d = detached）
docker run -d nginx

# 输出：Container ID
# abc123def456...

# Container 在后台运行！
```

### Step 2.3: Naming Containers

```bash
# 给 container 一个友好的名称
docker run -d --name my-nginx nginx

# 验证它正在运行
docker ps

# 输出显示你的命名 container
```

**Questions**:
1. 为什么命名 container 很有用？
2. 如果尝试创建另一个同名 container 会发生什么？

✅ **Checkpoint**: 你可以使用自定义名称以分离模式运行 container。

---

## Part 3: Managing Container Lifecycle

### Step 3.1: List Containers

```bash
# 列出运行中的 container
docker ps

# 列出所有 container（包括已停止的）
docker ps -a

# 使用自定义格式列出
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### Step 3.2: Stop and Start Containers

```bash
# 停止运行中的 container
docker stop my-nginx

# 验证它已停止
docker ps -a

# 再次启动
docker start my-nginx

# 重启 container
docker restart my-nginx
```

### Step 3.3: Pause and Unpause

```bash
# 暂停 container（冻结进程）
docker pause my-nginx

# 检查状态
docker ps

# 恢复
docker unpause my-nginx
```

### Step 3.4: Remove Containers

```bash
# 先停止 container
docker stop my-nginx

# 删除 container
docker rm my-nginx

# 强制删除运行中的 container
docker rm -f my-nginx

# 运行在停止时自动删除的 container
docker run -d --rm --name temp-nginx nginx
docker stop temp-nginx
docker ps -a  # Container 已消失！
```

**Task**: 创建管理 nginx 生命周期的脚本

```bash
#!/bin/bash
# lifecycle.sh

echo "Starting nginx..."
docker run -d --name web nginx

echo "Waiting 5 seconds..."
sleep 5

echo "Stopping nginx..."
docker stop web

echo "Removing nginx..."
docker rm web

echo "Done!"
```

✅ **Checkpoint**: 你可以管理 container 生命周期（start、stop、remove）。

---

## Part 4: Port Mapping

### Step 4.1: Expose Container Ports

```bash
# 将主机端口 8080 映射到 container 端口 80
docker run -d -p 8080:80 --name web nginx

# 在浏览器或使用 curl 测试
curl http://localhost:8080

# 你应该看到 nginx 欢迎页面！
```

### Step 4.2: Multiple Port Mappings

```bash
# 映射多个端口
docker run -d \
  -p 8080:80 \
  -p 8443:443 \
  --name web-secure \
  nginx

# 检查端口映射
docker port web-secure
```

### Step 4.3: Random Port Assignment

```bash
# 让 Docker 分配随机主机端口
docker run -d -P --name web-random nginx

# 查找分配的端口
docker port web-random

# 输出：80/tcp -> 0.0.0.0:32768
```

### Step 4.4: Bind to Specific Interface

```bash
# 仅限 localhost 访问
docker run -d -p 127.0.0.1:8080:80 --name web-local nginx

# 可从任何接口访问
docker run -d -p 0.0.0.0:8080:80 --name web-public nginx
```

**Challenge**: 在端口 8081、8082、8083 上运行 3 个 nginx container

```bash
# 你的代码
docker run -d -p 8081:80 --name web1 nginx
docker run -d -p 8082:80 --name web2 nginx
docker run -d -p 8083:80 --name web3 nginx

# 验证全部运行
docker ps

# 测试所有端口
curl http://localhost:8081
curl http://localhost:8082
curl http://localhost:8083
```

✅ **Checkpoint**: 你可以将 container 端口映射到主机。

---

## Part 5: Environment Variables

### Step 5.1: Pass Environment Variables

```bash
# 单个变量
docker run -d -e MY_VAR=hello --name env-test alpine sleep 3600

# 验证
docker exec env-test env | grep MY_VAR
# MY_VAR=hello

# 多个变量
docker run -d \
  -e DB_HOST=localhost \
  -e DB_PORT=5432 \
  -e DB_NAME=mydb \
  --name app-with-env \
  alpine sleep 3600
```

### Step 5.2: Use Environment File

```bash
# 创建 .env 文件
cat > app.env << EOF
DB_HOST=postgres
DB_PORT=5432
DB_USER=appuser
DB_PASSWORD=secret123
DEBUG=true
EOF

# 使用 env 文件运行
docker run -d --env-file app.env --name app-env alpine sleep 3600

# 验证
docker exec app-env env
```

### Step 5.3: Real Example - PostgreSQL

```bash
# 使用环境变量运行 PostgreSQL
docker run -d \
  --name postgres-db \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydb \
  -p 5432:5432 \
  postgres:15

# 检查日志
docker logs postgres-db

# 连接到数据库
docker exec -it postgres-db psql -U myuser -d mydb

# 在 psql 中：
# \l  - 列出数据库
# \q  - 退出
```

✅ **Checkpoint**: 你可以使用环境变量配置 container。

---

## Part 6: Viewing Logs

### Step 6.1: Basic Logging

```bash
# 查看日志
docker logs web

# 跟踪日志（类似 tail -f）
docker logs -f web

# 最后 50 行
docker logs --tail 50 web

# 自特定时间起的日志
docker logs --since 10m web  # 最近 10 分钟

# 带时间戳
docker logs -f --timestamps web
```

### Step 6.2: Generate Some Logs

```bash
# 生成 nginx 流量
for i in {1..10}; do
  curl http://localhost:8080
done

# 查看访问日志
docker logs web
```

### Step 6.3: Log Driver Configuration

```bash
# 使用 JSON 日志驱动运行（默认）
docker run -d \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --name web-logs \
  nginx

# 不记录日志（用于性能）
docker run -d --log-driver none --name no-logs nginx
```

✅ **Checkpoint**: 你可以查看和管理 container 日志。

---

## Part 7: Executing Commands in Containers

### Step 7.1: Basic Exec

```bash
# 执行单个命令
docker exec web ls /usr/share/nginx/html

# 运行交互式 shell
docker exec -it web bash

# 在 container 内部：
ls
pwd
whoami
exit
```

### Step 7.2: Execute as Different User

```bash
# 以 root 用户执行（默认）
docker exec -u root web whoami

# 以 www-data 用户执行
docker exec -u www-data web whoami
```

### Step 7.3: Practical Examples

```bash
# 检查 nginx 配置
docker exec web nginx -t

# 查看 nginx 进程
docker exec web ps aux

# 检查磁盘使用情况
docker exec web df -h

# 安装工具并使用
docker exec web apt-get update
docker exec web apt-get install -y curl
docker exec web curl localhost
```

**Challenge**: 创建 Python container 并执行 Python 命令

```bash
# 运行 Python container
docker run -d --name python-app python:3.11 sleep 3600

# 执行 Python 命令
docker exec python-app python --version
docker exec python-app pip list
docker exec -it python-app python

# 在 Python REPL 中：
# >>> print("Hello from Docker!")
# >>> exit()
```

✅ **Checkpoint**: 你可以在运行中的 container 内部执行命令。

---

## Part 8: Inspecting Containers

### Step 8.1: Container Inspection

```bash
# 获取所有详情
docker inspect web

# 获取特定字段
docker inspect web --format='{{.State.Status}}'
docker inspect web --format='{{.NetworkSettings.IPAddress}}'
docker inspect web --format='{{json .Config.Env}}'
```

### Step 8.2: Resource Usage

```bash
# 实时统计
docker stats

# 单个 container
docker stats web

# 不持续输出（单次）
docker stats --no-stream
```

### Step 8.3: Process Information

```bash
# 查看 container 中的进程
docker top web

# 使用自定义格式
docker top web aux
```

### Step 8.4: File System Changes

```bash
# 查看 container 中的文件变更
docker diff web

# 图例：
# A - 添加
# D - 删除
# C - 修改
```

✅ **Checkpoint**: 你可以检查 container 详情和资源使用情况。

---

## Part 9: Resource Constraints

### Step 9.1: Memory Limits

```bash
# 内存限制为 512MB
docker run -d --memory="512m" --name limited-mem nginx

# 带 swap 限制的内存
docker run -d \
  --memory="512m" \
  --memory-swap="1g" \
  --name mem-with-swap \
  nginx

# 验证限制
docker stats limited-mem --no-stream
```

### Step 9.2: CPU Limits

```bash
# 限制为 1 个 CPU
docker run -d --cpus="1.0" --name limited-cpu nginx

# CPU 份额（相对权重）
docker run -d --cpu-shares=512 --name low-priority nginx
docker run -d --cpu-shares=1024 --name high-priority nginx
```

### Step 9.3: Test Resource Limits

```bash
# 运行会消耗内存的 container
docker run -d \
  --memory="100m" \
  --name stress-test \
  progrium/stress --vm 1 --vm-bytes 150M

# 观察会发生什么
docker logs stress-test
docker stats stress-test

# Container 应因 OOM 被杀死
```

✅ **Checkpoint**: 你可以设置和监控资源限制。

---

## Part 10: Interactive Containers

### Step 10.1: Interactive Mode

```bash
# 以交互方式运行 Ubuntu
docker run -it ubuntu bash

# 在 container 内部：
apt-get update
apt-get install -y curl
curl https://api.github.com
exit
```

### Step 10.2: Attach to Running Container

```bash
# 运行 container
docker run -d --name myapp alpine sh -c "while true; do echo Hello; sleep 2; done"

# 附加以查看输出
docker attach myapp

# 分离：Ctrl+P, Ctrl+Q（保持 container 运行）
# 或 Ctrl+C（停止 container）
```

### Step 10.3: Copy Files

```bash
# 创建测试文件
echo "Hello from host" > test.txt

# 复制到 container
docker cp test.txt web:/tmp/

# 验证
docker exec web cat /tmp/test.txt

# 从 container 复制
docker exec web sh -c "echo 'From container' > /tmp/from-container.txt"
docker cp web:/tmp/from-container.txt .
cat from-container.txt
```

✅ **Checkpoint**: 你可以与 container 进行交互式操作。

---

## Part 11: Cleanup and Maintenance

### Step 11.1: Remove Stopped Containers

```bash
# 删除所有已停止的 container
docker container prune

# 确认：y
```

### Step 11.2: Remove All Containers

```bash
# 停止所有运行中的 container
docker stop $(docker ps -q)

# 删除所有 container
docker rm $(docker ps -a -q)
```

### Step 11.3: System Cleanup

```bash
# 查看磁盘使用情况
docker system df

# 详细视图
docker system df -v

# 清理所有未使用的资源
docker system prune

# 包括 volume
docker system prune -a --volumes
```

✅ **Checkpoint**: 你可以清理 Docker 资源。

---

## Part 12: Practical Application

### Challenge: Web Application Stack

**Task**: 设置完整的 web 应用环境

```bash
# 1. 运行 PostgreSQL 数据库
docker run -d \
  --name db \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=appdb \
  postgres:15

# 2. 运行 Redis 缓存
docker run -d \
  --name cache \
  redis:7-alpine

# 3. 等待服务启动
sleep 5

# 4. 检查所有正在运行
docker ps

# 5. 测试数据库
docker exec db psql -U postgres -d appdb -c "SELECT version();"

# 6. 测试 Redis
docker exec cache redis-cli ping

# 7. 查看日志
docker logs db
docker logs cache

# 8. 检查资源使用情况
docker stats --no-stream

# 9. 清理
docker stop db cache
docker rm db cache
```

### Challenge: Debug a Failing Container

```bash
# 运行会失败的 container
docker run -d --name failing-app nginx:nonexistent

# 出了什么问题？调试：
# 1. 检查状态
docker ps -a

# 2. 查看日志
docker logs failing-app

# 3. 检查
docker inspect failing-app

# 4. 修复并重新运行
docker rm failing-app
docker run -d --name working-app nginx:latest
```

---

## Validation and Testing

### Test Your Knowledge

创建一个 bash 脚本：
1. 在不同端口上运行 3 个 nginx container
2. 向每个生成流量
3. 查看所有 container 的日志
4. 检查资源使用情况
5. 停止并删除所有 container

```bash
#!/bin/bash
# solution.sh

# 你的实现
echo "Starting containers..."
docker run -d -p 8081:80 --name web1 nginx
docker run -d -p 8082:80 --name web2 nginx
docker run -d -p 8083:80 --name web3 nginx

echo "Waiting for startup..."
sleep 3

echo "Generating traffic..."
for i in {1..5}; do
  curl -s http://localhost:8081 > /dev/null
  curl -s http://localhost:8082 > /dev/null
  curl -s http://localhost:8083 > /dev/null
done

echo "Viewing logs..."
docker logs --tail 5 web1
docker logs --tail 5 web2
docker logs --tail 5 web3

echo "Resource usage..."
docker stats --no-stream web1 web2 web3

echo "Cleanup..."
docker stop web1 web2 web3
docker rm web1 web2 web3

echo "Done!"
```

---

## Reflection Questions

1. **`docker stop` 和 `docker kill` 有什么区别？**
   - 你的答案：

2. **什么时候会使用 `--rm` 标志？**
   - 你的答案：

3. **如何调试一个立即退出的 container？**
   - 你的答案：

4. **`docker run` 和 `docker exec` 有什么区别？**
   - 你的答案：

5. **为什么限制 container 资源很重要？**
   - 你的答案：

---

## Summary

**What You Accomplished**:
✅ 在多种模式下运行了 container
✅ 管理了 container 生命周期
✅ 配置了端口映射
✅ 使用了环境变量
✅ 查看并分析了日志
✅ 在 container 中执行了命令
✅ 设置了资源限制
✅ 检查了 container 详情
✅ 清理了资源

**Key Commands Mastered**:
- `docker run` - 创建和启动 container
- `docker ps` - 列出 container
- `docker logs` - 查看 container 日志
- `docker exec` - 执行命令
- `docker stop/start/restart` - 管理生命周期
- `docker rm` - 删除 container
- `docker stats` - 监控资源
- `docker inspect` - 查看详情

---

## Next Steps

**Practice More**:
1. 运行不同类型的应用（数据库、web 服务器等）
2. 尝试不同的资源限制
3. 练习调试失败的 container
4. 创建更复杂的多 container 设置

**Continue Learning**:
- **Exercise 02**: Building Custom Images
- **Exercise 03**: Docker Compose Applications

---

**Exercise Version**: 1.0
**Last Updated**: October 2025
**Estimated Time**: 2-3 小时
**Difficulty**: 初级
