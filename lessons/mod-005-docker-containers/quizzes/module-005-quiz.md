# Module 005: Docker & Containerization - Comprehensive Quiz

## Quiz Overview

**Total Questions**: 50
**Passing Score**: 80% (40/50 正确)
**Time Limit**: 90 分钟
**Question Types**: 多选题、判断题、场景题

**Topics Covered**:
- Docker 基础
- Dockerfile 和 image 构建
- Docker Compose
- 网络
- Volume 和数据持久化
- 生产部署
- ML containerization
- 安全和 best practices

---

## Section 1: Docker Fundamentals (10 题)

### Question 1
Docker image 和 Docker container 之间的主要区别是什么？

A) Image 是运行实例，container 是模板
B) Image 是模板，container 是运行实例
C) 它们是同一个东西
D) Image 用于开发，container 用于生产

**Answer**: B

---

### Question 2
哪个命令用于以分离模式运行 container，并将主机的 8080 端口映射到 container 的 80 端口？

A) `docker run -p 8080:80 nginx`
B) `docker run -d -p 80:8080 nginx`
C) `docker run -d -p 8080:80 nginx`
D) `docker run --detach --port 8080:80 nginx`

**Answer**: C

---

### Question 3
当 container 被删除时，container 内部的数据会发生什么？

A) 数据会自动备份
B) 数据在主机上持久存在
C) 除非存储在 volume 中，否则数据会丢失
D) 数据会被移动到新的 container

**Answer**: C

---

### Question 4
哪个命令仅显示正在运行的 container？

A) `docker ps -a`
B) `docker ps`
C) `docker containers ls --all`
D) `docker list running`

**Answer**: B

---

### Question 5
`docker exec` 的用途是什么？

A) 在运行中的 container 内执行命令
B) 启动已停止的 container
C) 创建新的 container
D) 导出 container 文件系统

**Answer**: A

---

### Question 6
如何删除所有已停止的 container？

A) `docker rm --all`
B) `docker container prune`
C) `docker clean containers`
D) `docker delete stopped`

**Answer**: B

---

### Question 7
`docker run -it ubuntu bash` 中的 `-it` 标志组合的作用是什么？

A) 交互式 + TTY 分配
B) Image + Tag 指定
C) 安装 + 测试模式
D) 内部 + 临时

**Answer**: A

---

### Question 8
哪个 Docker 命令显示 container 的资源使用统计？

A) `docker info`
B) `docker stats`
C) `docker top`
D) `docker inspect`

**Answer**: B

---

### Question 9
Docker container 的默认网络驱动是什么？

A) host
B) none
C) bridge
D) overlay

**Answer**: C

---

### Question 10
判断题：默认情况下，在默认 bridge 网络上的 container 可以通过 container 名称解析彼此。

A) 正确
B) 错误

**Answer**: B (错误 — 需要自定义 bridge 网络才能实现 DNS)

---

## Section 2: Dockerfiles & Image Building (10 题)

### Question 11
哪个 Dockerfile 指令设置后续命令的工作目录？

A) `CD`
B) `WORKDIR`
C) `DIR`
D) `SETDIR`

**Answer**: B

---

### Question 12
Dockerfile 中 `COPY` 和 `ADD` 的区别是什么？

A) 没有区别
B) `ADD` 可以解压 tar 文件并下载 URL
C) `COPY` 更快
D) `ADD` 仅适用于本地文件

**Answer**: B

---

### Question 13
在多阶段 Dockerfile 中，如何从名为 "builder" 的上一个阶段复制文件？

A) `COPY builder:/app /app`
B) `COPY --from=builder /app /app`
C) `COPY --stage=builder /app /app`
D) `FROM builder COPY /app /app`

**Answer**: B

---

### Question 14
哪个指令在 Docker image 中创建新的 layer？

A) `ENV`
B) `EXPOSE`
C) `RUN`
D) `CMD`

**Answer**: C

---

### Question 15
在 Dockerfile 中安装包的最佳实践是什么？

A) `RUN apt-get update && apt-get install -y package && rm -rf /var/lib/apt/lists/*`
B) 先 `RUN apt-get update` 再 `RUN apt-get install -y package`
C) `RUN apt-get install -y package`
D) 在 container 启动后安装包

**Answer**: A

---

### Question 16
`CMD` 和 `ENTRYPOINT` 的区别是什么？

A) 没有区别
B) `CMD` 可以轻松被覆盖，`ENTRYPOINT` 定义可执行程序
C) `CMD` 用于开发，`ENTRYPOINT` 用于生产
D) `ENTRYPOINT` 已弃用

**Answer**: B

---

### Question 17
`.dockerignore` 的作用是什么？

A) 构建时忽略错误
B) 从 build context 中排除文件
C) 阻止 container 启动
D) 隐藏敏感数据

**Answer**: B

---

### Question 18
生产 Python 应用的最佳优化基础 image 是哪个？

A) `python:3.11`
B) `python:3.11-slim`
C) `python:3.11-alpine`
D) 安装了 Python 的 `ubuntu:latest`

**Answer**: B (由于兼容性原因，slim 通常比 alpine 更适合 Python)

---

### Question 19
Dockerfile 中 `ARG` 的用途是什么？

A) 定义环境变量
B) 定义构建时变量
C) 向 CMD 传递参数
D) 配置 ENTRYPOINT 的参数

**Answer**: B

---

### Question 20
判断题：Layer 缓存通过重用未更改的 layer 来加速构建。

A) 正确
B) 错误

**Answer**: A

---

## Section 3: Docker Compose (8 题)

### Question 21
Docker Compose 使用什么文件格式？

A) JSON
B) YAML
C) XML
D) TOML

**Answer**: B

---

### Question 22
如何以分离模式启动 `docker-compose.yml` 中定义的所有 service？

A) `docker-compose start -d`
B) `docker-compose up --detach`
C) `docker-compose run -d`
D) `docker-compose deploy`

**Answer**: B

---

### Question 23
在 Docker Compose 中，service A 如何在启动前等待 service B 变为健康状态？

```yaml
services:
  A:
    depends_on:
      ?
```

A) `B: required`
B) `B: wait`
C) `B: { condition: service_healthy }`
D) `B: true`

**Answer**: C

---

### Question 24
如何在 Docker Compose 中将 service 扩展为 3 个副本？

A) `docker-compose scale service=3`
B) `docker-compose up --scale service=3`
C) `docker-compose replicas service 3`
D) 修改 `docker-compose.yml` 并设置 `replicas: 3`

**Answer**: B

---

### Question 25
哪个 Docker Compose 命令会删除 container、网络和 volume？

A) `docker-compose down`
B) `docker-compose down -v`
C) `docker-compose clean`
D) `docker-compose remove --all`

**Answer**: B

---

### Question 26
如何在 `docker-compose.yml` 中引用 `.env` 文件中的环境变量？

A) `$ENV_VAR`
B) `${ENV_VAR}`
C) `{ENV_VAR}`
D) `@ENV_VAR`

**Answer**: B

---

### Question 27
Docker Compose 中 `networks` 的用途是什么？

A) 配置互联网访问
B) 定义自定义网络以实现 service 隔离
C) 设置 VPN 连接
D) 配置 DNS 服务器

**Answer**: B

---

### Question 28
判断题：Docker Compose 会为所有 service 创建一个默认网络。

A) 正确
B) 错误

**Answer**: A

---

## Section 4: Networking (8 题)

### Question 29
哪个网络驱动提供 container 之间的自动 DNS 解析？

A) 默认 bridge
B) 自定义 bridge
C) host
D) none

**Answer**: B

---

### Question 30
`host` 网络模式的作用是什么？

A) 创建隔离网络
B) Container 直接使用主机的网络栈
C) 提供互联网访问
D) 启用 IPv6

**Answer**: B

---

### Question 31
如何创建自定义 Docker 网络？

A) `docker network new mynetwork`
B) `docker network create mynetwork`
C) `docker create network mynetwork`
D) `docker network add mynetwork`

**Answer**: B

---

### Question 32
在 Docker Compose 中，service 之间可以使用什么进行通信？

A) 仅 IP 地址
B) 使用 service 名称作为主机名
C) Container ID
D) MAC 地址

**Answer**: B

---

### Question 33
哪个端口映射使服务仅在 localhost 上可访问？

A) `-p 8080:80`
B) `-p 0.0.0.0:8080:80`
C) `-p 127.0.0.1:8080:80`
D) `-p localhost:8080:80`

**Answer**: C

---

### Question 34
Docker 中网络别名的用途是什么？

A) 重命名网络
B) 为 container 提供替代主机名
C) 配置 IP 地址
D) 设置网络桥接

**Answer**: B

---

### Question 35
如何检查名为 "mynet" 的 Docker 网络？

A) `docker network show mynet`
B) `docker network inspect mynet`
C) `docker inspect network mynet`
D) `docker network info mynet`

**Answer**: B

---

### Question 36
判断题：在同一个自定义 bridge 网络上的 container 可以在不发布端口的情况下进行通信。

A) 正确
B) 错误

**Answer**: A

---

## Section 5: Volumes & Data Persistence (6 题)

### Question 37
Docker volume/mount 的三种类型是什么？

A) 命名、匿名、临时
B) 命名 volume、bind mount、tmpfs
C) 持久、临时、缓存
D) 本地、远程、分布式

**Answer**: B

---

### Question 38
哪种 mount 类型将数据存储在主机的内存（RAM）中？

A) 命名 volume
B) Bind mount
C) tmpfs mount
D) 缓存 mount

**Answer**: C

---

### Question 39
如何创建命名 volume？

A) `docker volume new mydata`
B) `docker volume create mydata`
C) `docker create volume mydata`
D) `docker volume add mydata`

**Answer**: B

---

### Question 40
备份 Docker volume 的推荐方式是什么？

A) 从 `/var/lib/docker/volumes/` 复制
B) 使用 `docker volume backup`
C) 运行带有 volume 挂载的 container 并创建 tar 归档
D) 在 volume 上使用 `docker cp`

**Answer**: C

---

### Question 41
在哪种场景下你会使用 bind mount 而不是命名 volume？

A) 数据库存储
B) 带热重载的开发环境
C) 生产数据
D) container 之间共享数据

**Answer**: B

---

### Question 42
判断题：命名 volume 在 container 被删除后仍然存在。

A) 正确
B) 错误

**Answer**: A

---

## Section 6: Production & Security (8 题)

### Question 43
哪项是生产 container 的安全最佳实践？

A) 以 root 用户运行
B) 使用 `latest` 标签
C) 以非 root 用户运行
D) 禁用健康检查

**Answer**: C

---

### Question 44
`--read-only` 标志的作用是什么？

A) 以只读模式打开文件
B) 使根文件系统为只读
C) 阻止日志写入
D) 禁用网络访问

**Answer**: B

---

### Question 45
哪种部署策略允许以较小的流量百分比测试新版本？

A) 蓝绿部署
B) 滚动更新
C) 金丝雀部署
D) 重建

**Answer**: C

---

### Question 46
Dockerfile 中 `HEALTHCHECK` 的用途是什么？

A) 检查 image 是否健康
B) 监控 container 健康状态
C) 验证 Dockerfile 语法
D) 检查磁盘空间

**Answer**: B

---

### Question 47
如何限制 container 最多使用 512MB 内存？

A) `docker run --memory 512M myapp`
B) `docker run --mem-limit 512 myapp`
C) `docker run --max-memory 512MB myapp`
D) `docker run -m 512M myapp`

**Answer**: D（A 也正确 — D 是短格式）

---

### Question 48
生产环境推荐的日志驱动选项是什么？

A) `none`
B) 带轮转的 `json-file`
C) `console`
D) 无限制的 `json-file`

**Answer**: B

---

### Question 49
哪个工具可以扫描 Docker image 中的漏洞？

A) docker lint
B) docker check
C) Trivy 或 Docker Scout
D) docker scan（已弃用）

**Answer**: C

---

### Question 50
判断题：多阶段构建有助于减小最终 image 的大小。

A) 正确
B) 错误

**Answer**: A

---

## Answer Key

### Section 1: Fundamentals
1. B  2. C  3. C  4. B  5. A  6. B  7. A  8. B  9. C  10. B

### Section 2: Dockerfiles
11. B  12. B  13. B  14. C  15. A  16. B  17. B  18. B  19. B  20. A

### Section 3: Compose
21. B  22. B  23. C  24. B  25. B  26. B  27. B  28. A

### Section 4: Networking
29. B  30. B  31. B  32. B  33. C  34. B  35. B  36. A

### Section 5: Volumes
37. B  38. C  39. B  40. C  41. B  42. A

### Section 6: Production
43. C  44. B  45. C  46. B  47. D  48. B  49. C  50. A

---

## Scoring Guide

- **45-50 正确 (90-100%)**: 优秀！你已经掌握了 Docker。
- **40-44 正确 (80-88%)**: 良好！你通过了。复习做错的主题。
- **35-39 正确 (70-78%)**: 很接近！复习材料后重测。
- **低于 35 (<70%)**: 重测前复习所有讲座和练习。

---

## Topics to Review by Score

**如果你的分数低于 80%**，根据你最薄弱的环节复习以下主题：

- **Section 1**: 复习 Lecture 01 (Docker 基础)，Exercise 01
- **Section 2**: 复习 Lecture 02 (Dockerfile)，Exercise 02
- **Section 3**: 复习 Lecture 03 (Docker Compose)，Exercise 03
- **Section 4**: 复习 Lecture 04 (网络)，Exercise 04
- **Section 5**: 复习 Lecture 04 (Volume)，Exercise 05
- **Section 6**: 复习 Lecture 05 (Best Practices)，Exercise 06

---

**Quiz Version**: 1.0
**Last Updated**: October 2025
**Time to Complete**: 90 分钟
**Passing Score**: 80% (40/50)
