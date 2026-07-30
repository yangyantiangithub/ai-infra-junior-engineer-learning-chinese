# Lecture 01: Docker Fundamentals

## Learning Objectives

通过本讲座，你将能够：

- 理解什么是 container 以及它们为何重要
- 解释 container 与虚拟机的区别
- 理解 Docker 架构及其组件
- 在你的系统上安装 Docker
- 运行你的第一个 container
- 自信地使用基本 Docker 命令
- 理解 image 和 container 之间的关系

**Duration**: 90 分钟
**Difficulty**: 初级
**Prerequisites**: Module 002 (Linux Essentials)

---

## 1. Introduction to Containerization

### 什么是 Container？

**Containers** 是轻量级、独立的软件包，包含运行软件所需的一切：代码、运行时、系统工具、库和设置。

**Analogy**: 可以把 container 想象成海运集装箱：
- 标准化格式
- 可在不同系统间移植
- 与其他 container 隔离
- 可以堆叠且易于管理

```
传统部署:
┌─────────────────────────┐
│   物理服务器            │
│  ┌──────────────────┐  │
│  │  应用 A + 依赖   │  │
│  │  应用 B + 依赖   │  │  ← 依赖冲突！
│  │  应用 C + 依赖   │  │     难以管理！
│  └──────────────────┘  │
└─────────────────────────┘

Container 部署:
┌─────────────────────────┐
│   物理服务器            │
│  ┌────┐ ┌────┐ ┌────┐  │
│  │应用│ │应用│ │应用│  │
│  │ A  │ │ B  │ │ C  │  │  ← 隔离！
│  │+依赖│ │+依赖│ │+依赖│  │     可移植！
│  └────┘ └────┘ └────┘  │
└─────────────────────────┘
```

### 为什么 Container 很重要

**对于基础设施工程师：**

1. **一致性**："在我机器上能运行" → "在任何地方都能运行"
2. **可移植性**：在开发、测试和生产环境之间无缝移动
3. **效率**：启动速度比 VM 快，资源利用率更高
4. **隔离性**：应用之间不会相互干扰
5. **可扩展性**：易于复制和水平扩展
6. **版本控制**：Image 可以像代码一样进行版本管理

**对于 AI/ML 基础设施：**

- 使用精确的依赖关系打包模型
- 一致的训练环境
- 简单的模型 serving 部署
- GPU 资源管理
- 可重现的研究环境

---

## 2. Containers vs Virtual Machines

### 架构对比

```
虚拟机:
┌─────────────────────────────────┐
│        物理服务器                │
│  ┌──────────────────────────┐  │
│  │     Hypervisor           │  │
│  └──────────────────────────┘  │
│  ┌────┐  ┌────┐  ┌────┐       │
│  │VM1 │  │VM2 │  │VM3 │       │
│  │OS  │  │OS  │  │OS  │       │  ← 每个 VM 都有完整 OS！
│  │App │  │App │  │App │       │     重量级！
│  └────┘  └────┘  └────┘       │
└─────────────────────────────────┘

Containers:
┌─────────────────────────────────┐
│        物理服务器                │
│  ┌──────────────────────────┐  │
│  │     主机 OS              │  │
│  └──────────────────────────┘  │
│  ┌──────────────────────────┐  │
│  │   Container Runtime      │  │
│  └──────────────────────────┘  │
│  ┌────┐  ┌────┐  ┌────┐       │
│  │C1  │  │C2  │  │C3  │       │  ← 共享主机 OS 内核！
│  │App │  │App │  │App │       │     轻量级！
│  └────┘  └────┘  └────┘       │
└─────────────────────────────────┘
```

### Comparison Table

| 对比项 | 虚拟机 | Containers |
|--------|------------------|------------|
| **启动时间** | 分钟级 | 秒级 |
| **大小** | GB 级别 | MB 级别 |
| **性能** | 较慢（有开销） | 接近原生 |
| **隔离性** | 完全（硬件级别） | 进程级别 |
| **操作系统** | 每个 VM 有完整 OS | 共享主机 OS 内核 |
| **可移植性** | 有限 | 高度可移植 |
| **资源使用** | 重量级 | 轻量级 |
| **使用场景** | 需要强隔离 | 微服务、CI/CD |

### 何时使用什么？

**使用 VM 的场景：**
- 需要不同的操作系统
- 要求完全隔离（安全）
- 运行遗留应用
- 需要硬件级别功能

**使用 Container 的场景：**
- 构建微服务
- CI/CD 流水线
- 开发环境
- 云原生应用
- ML model serving

**Best Practice**: 通常两者一起使用！VM 提供隔离，VM 内部的 container 提供效率。

---

## 3. Docker Architecture

### Overview

Docker 使用 **客户端-服务器架构**：

```
┌─────────────────────────────────────────────────┐
│                Docker 客户端                      │
│              (docker 命令)                        │
└────────────────────┬────────────────────────────┘
                     │ REST API
                     ↓
┌─────────────────────────────────────────────────┐
│              Docker Daemon                       │
│               (dockerd)                          │
│  ┌────────────────────────────────────────┐    │
│  │     Container 管理                      │    │
│  ├────────────────────────────────────────┤    │
│  │     Image 管理                          │    │
│  ├────────────────────────────────────────┤    │
│  │     Volume 管理                         │    │
│  ├────────────────────────────────────────┤    │
│  │     Network 管理                        │    │
│  └────────────────────────────────────────┘    │
│                     │                           │
│                     ↓                           │
│              containerd                         │
│                     │                           │
│                     ↓                           │
│                  runc                           │
└─────────────────────────────────────────────────┘
                     │
                     ↓
┌─────────────────────────────────────────────────┐
│              Containers                          │
│    [Container 1] [Container 2] [Container 3]    │
└─────────────────────────────────────────────────┘
```

### Key Components

**1. Docker 客户端 (`docker` CLI)**
- 你使用的命令行界面
- 向 Docker daemon 发送命令
- 示例：`docker run`、`docker build`、`docker ps`

**2. Docker Daemon (`dockerd`)**
- 管理 container 的后台服务
- 监听 Docker API 请求
- 管理 image、container、网络、volume
- 可以与其他 daemon 通信

**3. containerd**
- 行业标准的 container runtime
- 管理 container 生命周期
- 镜像传输和存储
- 由 Docker daemon 启动/停止

**4. runc**
- 底层 container runtime
- 实际创建和运行 container
- 符合 OCI（Open Container Initiative）标准

**5. Docker Registry**
- 存储 Docker image
- Docker Hub：公共 registry
- 企业级私有 registry

### 所有组件如何协同工作

```
$ docker run nginx

1. 客户端：向 daemon 发送 'run nginx' 命令
2. Daemon：检查 nginx image 是否本地存在
3. Daemon：如果不存在，从 registry（Docker Hub）拉取
4. containerd：解压 image layer
5. runc：从 image 创建 container
6. Container：nginx 开始运行！
```

---

## 4. Installing Docker

### 在 Linux 上安装 (Ubuntu/Debian)

```bash
# 更新包索引
sudo apt-get update

# 安装先决条件
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 添加 Docker 官方 GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 设置仓库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 验证安装
sudo docker run hello-world
```

### 安装后：无需 `sudo` 运行 Docker

```bash
# 创建 docker 组（通常已存在）
sudo groupadd docker

# 将你的用户添加到 docker 组
sudo usermod -aG docker $USER

# 应用组成员身份（或注销/重新登录）
newgrp docker

# 验证
docker run hello-world  # 不需要 sudo 了！
```

### 在 Mac/Windows 上安装

**Docker Desktop**（推荐用于开发）：

1. 从 https://www.docker.com/products/docker-desktop 下载
2. 运行安装程序
3. 启动 Docker Desktop
4. 在终端中验证：`docker --version`

**包含的内容：**
- Docker Engine
- Docker CLI
- Docker Compose
- Kubernetes（可选）
- 仪表盘 UI

### 验证

```bash
# 检查版本
docker --version
# 输出：Docker version 24.0.6, build ed223bc

# 检查详细信息
docker version

# 检查 Docker 系统信息
docker info

# 运行测试 container
docker run hello-world
```

**hello-world 的预期输出**：
```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
```

---

## 5. Understanding Images and Containers

### Image-Container 关系

**Image** = 蓝图（OOP 中的类）
**Container** = 运行实例（OOP 中的对象）

```
Image (nginx:latest)
      │
      ├──→ Container 1 (web-server-1)
      ├──→ Container 2 (web-server-2)
      └──→ Container 3 (web-server-3)

一个 image，多个 container！
```

### Docker Image 是什么？

**Image**：用于创建 container 的只读模板，包含指令。

**Key Concepts:**
- 由多个 layer 组成
- 每个 layer 代表一个文件系统变更
- Layer 被缓存并可重用
- Image 是不可变的

```
Image Layers (示例)：
┌─────────────────────┐
│  应用代码 Layer      │  ← 顶层
├─────────────────────┤
│  pip install 依赖    │
├─────────────────────┤
│  Python 3.11         │
├─────────────────────┤
│  Ubuntu 基础         │  ← 底层
└─────────────────────┘
```

### Container 是什么？

**Container**：image 的可运行实例。

**Key Concepts:**
- 在 image 之上有可写 layer
- 可以启动、停止、移动、删除
- 与其他 container 隔离
- 可以连接到网络、附加存储

```
Container = Image + 可写 Layer + 运行时配置

┌─────────────────────┐
│  可写 Layer          │  ← Container 的变更
├─────────────────────┤
│  Image Layers        │  ← 只读
└─────────────────────┘
```

---

## 6. 你的第一个 Container

### 运行简单 Container

```bash
# 运行 nginx web 服务器
docker run nginx

# 发生了什么：
# 1. 下载 nginx image（如果不存在）
# 2. 从 image 创建 container
# 3. 启动 nginx 进程
# 4. 附加到 container 输出（日志）

# 使用 Ctrl+C 停止
```

### 以分离模式运行

```bash
# 在后台运行（-d = detached）
docker run -d nginx

# 输出：container ID
# a1b2c3d4e5f6...

# Container 在后台运行！
```

### 命名 Container

```bash
# 给 container 一个名称
docker run -d --name my-nginx nginx

# 通过名称引用比通过 ID 更容易
```

### Port Mapping

```bash
# 将 container 端口映射到主机端口
docker run -d -p 8080:80 nginx

# 格式：-p 主机端口:容器端口
# 通过 http://localhost:8080 访问 nginx

# 打开浏览器访问 http://localhost:8080
# 你应该看到 "Welcome to nginx!"
```

### 交互式 Container

```bash
# 以交互方式运行 Ubuntu
docker run -it ubuntu bash

# -i：保持 STDIN 打开（交互式）
# -t：分配伪 TTY（终端）

# 你现在在 container 内部！
root@a1b2c3d4e5f6:/# ls
root@a1b2c3d4e5f6:/# whoami
# root

root@a1b2c3d4e5f6:/# exit  # 退出 container
```

### 环境变量

```bash
# 传递环境变量
docker run -d -e MY_VAR=hello -e DB_HOST=localhost nginx

# 使用多个 -e 标志传递多个变量
```

---

## 7. 基本 Docker 命令

### 管理 Container

```bash
# 列出运行中的 container
docker ps

# 列出所有 container（包括已停止的）
docker ps -a

# 停止 container
docker stop my-nginx

# 启动已停止的 container
docker start my-nginx

# 重启 container
docker restart my-nginx

# 删除 container
docker rm my-nginx

# 强制删除运行中的 container
docker rm -f my-nginx

# 删除所有已停止的 container
docker container prune
```

### 查看 Container 详情

```bash
# 查看 container 日志
docker logs my-nginx

# 跟踪日志（类似 tail -f）
docker logs -f my-nginx

# 最近 100 行
docker logs --tail 100 my-nginx

# 检查 container
docker inspect my-nginx

# 查看 container 统计（CPU、内存）
docker stats my-nginx
```

### 在 Container 中执行命令

```bash
# 在运行中的 container 内执行命令
docker exec my-nginx ls /etc

# 交互式 shell
docker exec -it my-nginx bash

# 现在你在运行中的 container 内部！
```

### 管理 Image

```bash
# 列出 image
docker images

# 从 registry 拉取 image
docker pull python:3.11

# 删除 image
docker rmi nginx

# 删除未使用的 image
docker image prune

# 查看 image 历史（layers）
docker history nginx
```

---

## 8. 实际示例

### Example 1: 简单的 Web 服务器

```bash
# 在端口 8080 上运行 nginx
docker run -d -p 8080:80 --name webserver nginx

# 验证它正在运行
docker ps

# 检查日志
docker logs webserver

# 在浏览器中访问：http://localhost:8080

# 停止并删除
docker stop webserver
docker rm webserver
```

### Example 2: Python 应用

```bash
# 运行 Python REPL
docker run -it python:3.11 python

>>> print("Hello from Docker!")
>>> exit()

# 运行 Python 脚本
echo 'print("Hello from Docker!")' > hello.py
docker run -v $(pwd):/app python:3.11 python /app/hello.py
```

### Example 3: 数据库 Container

```bash
# 运行 PostgreSQL
docker run -d \
  --name postgres-db \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  postgres:15

# 检查它是否在运行
docker logs postgres-db

# 使用 psql 连接
docker exec -it postgres-db psql -U postgres

# 清理
docker stop postgres-db
docker rm postgres-db
```

### Example 4: 开发环境

```bash
# 运行 Jupyter notebook
docker run -d \
  --name jupyter \
  -p 8888:8888 \
  -v $(pwd):/home/jovyan/work \
  jupyter/scipy-notebook

# 获取访问令牌
docker logs jupyter | grep token

# 使用令牌访问 http://localhost:8888
```

---

## 9. 常见问题与排错

### Issue 1: "Cannot connect to Docker daemon"

**Error**: `Cannot connect to the Docker daemon. Is the docker daemon running?`

**解决方案**：
```bash
# 检查 Docker 是否在运行
sudo systemctl status docker

# 启动 Docker
sudo systemctl start docker

# 检查用户是否在 docker 组中
groups $USER

# 如果不在，将用户添加到 docker 组（见第 4 节）
```

### Issue 2: 端口已被占用

**Error**: `Bind for 0.0.0.0:8080 failed: port is already allocated`

**解决方案**：
```bash
# 查找哪个程序正在使用该端口
sudo lsof -i :8080

# 使用不同的主机端口
docker run -p 8081:80 nginx

# 或者停止冲突的服务
```

### Issue 3: Container 立即退出

**Problem**: Container 启动后立即停止

**Diagnosis**:
```bash
# 检查退出代码
docker ps -a

# 查看日志
docker logs <container-name>

# 以交互方式运行以调试
docker run -it <image> sh
```

### Issue 4: 磁盘空间不足

**Error**: `no space left on device`

**解决方案**：
```bash
# 检查 Docker 磁盘使用情况
docker system df

# 清理未使用的资源
docker system prune

# 删除所有未使用的 container、网络、image
docker system prune -a

# 删除特定项目
docker container prune  # 删除已停止的 container
docker image prune      # 删除悬挂的 image
docker volume prune     # 删除未使用的 volume
```

---

## 10. 最佳实践（基础）

### 1. 始终命名你的 Container

```bash
# 不好
docker run -d nginx

# 好
docker run -d --name my-webserver nginx
```

### 2. 使用特定的 Image Tag

```bash
# 不好（使用 'latest'，不可预测）
docker run nginx

# 好（特定版本）
docker run nginx:1.25.3
```

### 3. 定期清理

```bash
# 删除已停止的 container
docker container prune

# 删除未使用的 image
docker image prune

# 完整清理（小心！）
docker system prune -a
```

### 4. 通过日志进行调试

```bash
# 始终先检查日志
docker logs <container-name>

# 实时排错
docker logs -f <container-name>
```

### 5. 使用 `docker exec` 进行检查

```bash
# 不要创建新的 container 来调试
# 在运行中的 container 上使用 exec
docker exec -it <container-name> bash
```

---

## Key Takeaways

✅ **Containers** 是轻量级、隔离的应用包
✅ **Containers ≠ VMs**：Container 共享主机 OS，轻量得多
✅ **Docker** 是构建和运行 container 的平台
✅ **Images** 是蓝图，**containers** 是运行实例
✅ **Docker 使用客户端-服务器架构**，包含 daemon
✅ **基本工作流**：拉取 image → 运行 container → 管理生命周期
✅ **端口映射** 将 container 连接到主机网络
✅ **docker ps** 显示运行中的 container
✅ **docker logs** 显示 container 输出
✅ **docker exec** 在运行中的 container 内执行命令

---

## Quick Reference

### Essential Commands

```bash
# 运行 container
docker run [OPTIONS] IMAGE [COMMAND]

# 列出 container
docker ps [-a]

# 停止 container
docker stop CONTAINER

# 删除 container
docker rm CONTAINER

# 查看日志
docker logs CONTAINER

# 执行命令
docker exec [OPTIONS] CONTAINER COMMAND

# 列出 image
docker images

# 拉取 image
docker pull IMAGE[:TAG]

# 删除 image
docker rmi IMAGE
```

### Common Options

```
-d         分离模式（后台）
-it        交互式终端
-p         端口映射（主机:容器）
-e         环境变量
--name     命名 container
-v         Volume 挂载
--rm       Container 退出时自动删除
```

---

## 后续内容

在下一讲中，我们将深入探讨：
- **Dockerfile 基础**：构建你自己的 image
- **Image Layers**：理解 image 是如何构建的
- **构建 Image**：为你的应用创建自定义 image
- **Image 优化**：使 image 更小、更快

**下一讲前练习**：
1. 运行至少 5 个不同的 container
2. 练习 start/stop/remove
3. 使用 `docker exec` 探索运行中的 container
4. 尝试不同应用的端口映射

继续学习 `lecture-notes/02-dockerfiles-basics.md`

---

**Lecture Version**: 1.0
**Last Updated**: October 2025
**Estimated Time**: 90 分钟
**Difficulty**: 初级
