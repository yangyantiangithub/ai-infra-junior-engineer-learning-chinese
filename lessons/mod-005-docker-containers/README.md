# Module 005: Docker and Containers

## Module Overview

Containerization 彻底改变了我们构建、交付和运行应用程序的方式，尤其是在 AI/ML 基础设施领域。本模块提供 Docker（领先的 containerization 平台）的全面培训，教你如何将应用程序及其依赖打包、管理 container 生命周期以及编排多 container 应用。

你将学习 container 基础、Docker 架构、创建高效 container image 的最佳实践，以及如何使用 Docker Compose 管理多 container 应用。这些技能对于现代基础设施工程至关重要，可以在开发、测试和生产环境之间提供一致的运行环境。

完成本模块后，你将能够自信地 containerize 应用、构建优化 image、管理 container 网络和存储，并在 AI/ML 工作流中使用 container。

## Learning Objectives

完成本模块后，你将能够：

1. **理解 containerization 概念**以及 container 与虚拟机的区别
2. **在不同平台上安装和配置 Docker**
3. **使用 Docker image**，包括拉取、构建和发布
4. **创建 Dockerfile**，遵循高效和安全的 best practices
5. **管理 container 生命周期**，包括运行、停止和监控
6. **配置 container 网络**以实现 container 之间以及与外部系统的通信
7. **使用 volume 和 bind mount 管理持久化数据**
8. **使用 Docker Compose** 定义和运行多 container 应用
9. **优化 image** 的大小、安全性和构建性能
10. **将 container 应用于 AI/ML 工作流**，包括模型 serving 和训练

## Prerequisites

- 需要完成 Module 002 (Linux Essentials)
- 建议完成 Module 003 (Git Version Control)
- 建议完成 Module 001 (Python Fundamentals)
- 熟悉命令行界面
- 基本的网络概念理解

**Recommended Setup:**
- Docker Desktop (Windows/Mac) 或 Docker Engine (Linux) 版本 20.10+
- 至少 8GB RAM（AI/ML container 建议 16GB）
- 50GB+ 可用磁盘空间
- 终端访问
- 文本编辑器或 IDE

## Time Commitment

- **Total Estimated Time:** 40-50 小时
- **Lectures & Reading:** 15-18 小时
- **Hands-on Exercises:** 20-25 小时
- **Projects:** 5-7 小时

**Recommended Pace:**
- 兼职 (5-10 小时/周): 4-6 周
- 全职 (20-30 小时/周): 2-3 周

掌握 Container 需要大量动手实践。请花时间实验、排错和构建真实场景。

## Module Structure

### Week 1: Docker Fundamentals
- **Topics:** Container 概念、Docker 架构、基本命令
- **Key Skills:** pull, run, ps, logs, stop, rm
- **Practice:** 运行 container、基本生命周期管理

### Week 2: Building Images
- **Topics:** Dockerfile、image layer、构建优化
- **Key Skills:** build, tag, push, Dockerfile 语法
- **Practice:** 创建自定义 image、多阶段构建

### Week 3: Networking and Storage
- **Topics:** Container 网络、volume、bind mount
- **Key Skills:** network 命令、volume 管理、数据持久化
- **Practice:** 多 container 通信、数据管理

### Week 4: Docker Compose and Advanced Topics
- **Topics:** Docker Compose、安全、优化、AI/ML 用例
- **Key Skills:** docker-compose.yml、best practices、生产环境考量
- **Practice:** 多 container 应用、ML model serving

## Detailed Topic Breakdown

### 1. Introduction to Containers (5-6 小时)

#### 1.1 Containerization Fundamentals
- 什么是 container？
- Containerization 的历史（chroot, LXC, Docker）
- Container vs 虚拟机
- Containerization 的优势
- 在 AI/ML 基础设施中的用例
- Container 生态系统概述

#### 1.2 Docker Architecture
- Docker Engine 组件
- 客户端-服务器架构
- containerd 和 runc
- Docker daemon
- Docker CLI
- Docker registry（Docker Hub、私有 registry）
- Image 和 container 的关系

#### 1.3 Installing Docker
- 在 Linux 上安装
- Windows/Mac 上的 Docker Desktop
- 安装后配置
- 无需 sudo 运行 Docker (Linux)
- 验证安装
- Docker 版本管理
- 资源分配和限制

#### 1.4 Container Images and Registries
- 理解 image 和 layer
- Image 命名和 tagging 约定
- Docker Hub 和公共 registry
- 官方 vs 社区 image
- 基础 image（Alpine、Ubuntu、Python）
- Image 安全考量
- Registry 认证

### 2. Working with Docker Containers (7-9 小时)

#### 2.1 Running Containers
- 基本 `docker run` 命令
- 交互模式 vs 分离模式
- Container 命名
- 端口映射和发布
- 环境变量
- 资源约束（CPU、内存）
- Container 重启策略

#### 2.2 Managing Container Lifecycle
- 列出 container（`docker ps`）
- 启动和停止 container
- 暂停和恢复
- 重启 container
- 删除 container
- Container 日志
- 在运行中的 container 中执行命令

#### 2.3 Container Inspection and Debugging
- 检查 container
- 查看日志
- 附加到 container
- 向/从 container 复制文件
- Container 统计和监控
- 调试 container 问题
- 常见排错场景

#### 2.4 Working with Images
- 从 registry 拉取 image
- 搜索 image
- 列出本地 image
- 删除 image
- Image 历史和 layer
- 标记 image
- 保存和加载 image

### 3. Building Docker Images (9-11 小时)

#### 3.1 Dockerfile Basics
- Dockerfile 语法和结构
- FROM 指令（基础 image）
- RUN 指令（构建时命令）
- COPY 和 ADD 指令
- WORKDIR 指令
- ENV 指令（环境变量）
- EXPOSE 指令（端口）
- CMD vs ENTRYPOINT
- 理解 build context

#### 3.2 Image Layers and Caching
- Docker layer 如何工作
- Layer 缓存机制
- 优化 layer 顺序
- 缓存失效
- BuildKit 和缓存管理
- Layer 大小考量
- .dockerignore 文件

#### 3.3 Multi-Stage Builds
- 为什么使用多阶段构建？
- 语法和结构
- 命名构建阶段
- 在阶段之间复制 artifact
- 用例（编译语言、最小化大小）
- AI/ML 应用的多阶段构建
- 构建目标

#### 3.4 Building Efficient Images
- 最小化 image 大小
- 选择合适的基础 image
- Alpine Linux 的优势
- 合并 RUN 命令
- 删除不必要的文件
- 安全扫描
- 生产环境 image 的 best practices

#### 3.5 Building Images for Python/AI Applications
- Python 基础 image
- 高效安装依赖
- Docker 中的 requirements.txt
- Container 中的 Poetry 和 Pipenv
- CUDA 和 GPU 支持
- ML 框架 image（PyTorch、TensorFlow）
- Image 中的模型打包

#### 3.6 Publishing Images
- 推送到 Docker Hub
- 私有 registry 设置
- 认证和授权
- Image 版本控制策略
- 自动构建
- CI/CD 集成
- Image 签名和信任

### 4. Container Networking (6-8 小时)

#### 4.1 Docker Networking Fundamentals
- 默认网络行为
- 网络类型（bridge、host、none）
- 理解 bridge 网络
- Container DNS
- 端口映射深入解析
- 网络隔离
- 检查网络

#### 4.2 User-Defined Networks
- 创建自定义网络
- 将 container 连接到网络
- 网络别名
- 多网络 container
- 网络驱动
- Overlay 网络（Swarm 预览）
- 服务发现

#### 4.3 Container Communication
- Container 间通信
- 使用 container 名称作为主机名
- 链接 container（旧版）
- 用于配置的环境变量
- 服务发现模式
- 负载均衡基础
- 网络安全

#### 4.4 Advanced Networking
- Host 网络模式
- macvlan 和 ipvlan
- 网络排错
- 网络性能
- 对外暴露服务
- 反向代理模式（nginx）
- 网络策略

### 5. Container Storage and Data Persistence (5-7 小时)

#### 5.1 Understanding Container Storage
- Container 文件系统 layer
- 临时 vs 持久数据
- 存储驱动
- 联合文件系统
- 写时复制机制
- 存储性能考量

#### 5.2 Docker Volumes
- 创建和管理 volume
- 命名 volume vs 匿名 volume
- Volume 驱动
- 列出和检查 volume
- 删除未使用的 volume
- Volume 备份和恢复
- Volume 性能

#### 5.3 Bind Mounts
- 什么是 bind mount？
- 何时使用 bind mount
- 只读 mount
- 用于开发的 bind mount
- 安全考量
- 主机路径问题
- 权限问题

#### 5.4 tmpfs Mounts
- 内存存储
- tmpfs 的用例
- 性能优势
- 大小限制
- 敏感数据处理

#### 5.5 Data Management Patterns
- 数据库 container 和持久化
- 备份策略
- 数据 volume container（旧版）
- Container 间 volume 共享
- 数据初始化
- 迁移策略
- 管理大型数据集（ML 训练数据）

### 6. Docker Compose (7-9 小时)

#### 6.1 Introduction to Docker Compose
- 为什么使用 Docker Compose？
- Compose vs 单独的 docker run 命令
- 安装 Docker Compose
- Compose 文件格式和版本
- docker-compose.yml 结构
- 常见用例
- Compose CLI 概述

#### 6.2 Defining Services
- Service 配置
- Image vs build
- 环境变量
- 端口映射
- Volume 挂载
- Compose 中的网络
- 依赖和启动顺序

#### 6.3 Compose Commands
- 启动 service（`up`）
- 停止和删除（`down`）
- 查看日志
- 执行命令
- 扩展 service
- 构建 image
- Service 状态和管理

#### 6.4 Multi-Container Applications
- 带数据库的 Web 应用
- 带消息队列的应用
- 微服务架构
- Service 依赖
- 健康检查
- 重启策略
- 资源限制

#### 6.5 Development Workflows
- 用于开发环境的 Compose
- 热重载和 bind mount
- 覆盖文件（docker-compose.override.yml）
- 多个 Compose 文件
- 特定环境的配置
- 开发 vs 生产配置
- 使用 Compose 进行测试

#### 6.6 Compose for AI/ML Applications
- 使用 Compose 的 ML 训练 pipeline
- Model serving 架构
- Jupyter notebook 环境
- ML 工作流编排
- Compose 中的 GPU 访问
- 分布式训练设置
- 使用 Compose 集成 MLflow

### 7. Docker Best Practices and Security (4-5 小时)

#### 7.1 Dockerfile Best Practices
- 官方 Docker best practices
- 最小化 layer
- 高效排列指令
- 使用特定的基础 image 版本
- 非 root 用户
- 健康检查
- 标签和元数据

#### 7.2 Security Considerations
- 以非 root 用户运行
- 扫描漏洞
- 密钥管理
- 最小化攻击面
- 只读文件系统
- 安全扫描工具
- AppArmor 和 SELinux

#### 7.3 Production Readiness
- 健康检查和监控
- 日志记录 best practices
- 资源限制
- 重启策略
- 优雅关闭
- 配置管理
- 十二要素应用原则

#### 7.4 Image Optimization
- Layer 缓存策略
- 多阶段构建模式
- 最小化 image 大小
- 构建参数
- Squashing layer（适当时）
- Image tagging 策略
- Registry 清理

#### 7.5 Performance Tuning
- 存储驱动选择
- 网络性能
- 资源分配
- 构建性能
- Registry 缓存
- Image 拉取策略
- 监控 container 性能

## Lecture Outline

### Lecture 1: Introduction to Containers and Docker (90 分钟)
- Containerization 概念
- Container vs VM
- Docker 架构
- 安装和设置
- 第一个 container
- **Lab:** 运行你的第一个 container

### Lecture 2: Working with Containers (90 分钟)
- Container 生命周期
- 基本 Docker 命令
- 管理 container
- Container 检查
- 日志和调试
- **Lab:** Container 管理练习

### Lecture 3: Building Docker Images (120 分钟)
- Dockerfile 基础
- Image layer 和缓存
- 构建 image
- Best practices
- .dockerignore
- **Lab:** 创建自定义 image

### Lecture 4: Multi-Stage Builds and Optimization (90 分钟)
- 多阶段构建概念
- Image 大小优化
- 安全考量
- 构建 Python 应用
- **Lab:** 优化后的应用 image

### Lecture 5: Container Networking (90 分钟)
- 网络基础
- 网络类型
- Container 通信
- 端口映射
- 用户自定义网络
- **Lab:** 多 container 网络

### Lecture 6: Data Persistence (90 分钟)
- 存储选项
- Volume vs bind mount
- 数据管理模式
- 备份和恢复
- 性能考量
- **Lab:** 持久化数据场景

### Lecture 7: Docker Compose (120 分钟)
- Compose 基础
- docker-compose.yml 语法
- 多 container 应用
- Service 编排
- 开发工作流
- **Lab:** 完整应用栈

### Lecture 8: Containers for AI/ML (90 分钟)
- ML container 考量的因素
- GPU 支持
- Model serving container
- 训练 pipeline
- Jupyter 环境
- 使用 container 的生产 ML
- **Lab:** Containerized ML 应用

## Hands-On Exercises

### Exercise Categories

#### Basic Container Operations (8 个练习)
1. 运行和管理 container
2. 使用日志和调试
3. 端口映射和网络
4. 环境变量
5. 资源限制
6. Container 检查
7. Image 管理
8. Registry 操作

#### Building Images (8 个练习)
9. 基本 Dockerfile 创建
10. Layer 优化
11. 多阶段构建
12. Python 应用 containerization
13. 带依赖的 Web 应用
14. 优化 image 大小
15. 安全扫描
16. 发布到 registry

#### Networking and Storage (8 个练习)
17. 自定义网络
18. Container 通信
19. Volume 管理
20. Bind mount 场景
21. 多 container 网络
22. 数据库持久化
23. 备份和恢复
24. 性能测试

#### Docker Compose (8 个练习)
25. 基本 Compose 应用
26. 带数据库的 Web 应用
27. 微服务架构
28. 开发环境
29. 生产配置
30. ML 训练 pipeline
31. Model serving 栈
32. 完整 ML 工作流

## Assessment and Evaluation

### Knowledge Checks
- 每个主要部分后的小测验（共 7 个）
- Dockerfile 语法问题
- 网络场景分析
- 排错练习
- Best practices 评估

### Practical Assessments
- **Container Operations:** 执行完整的 container 生命周期管理
- **Image Building:** 创建优化、生产就绪的 image
- **Networking:** 配置多 container 通信
- **Compose:** 部署完整的应用栈
- **ML Application:** Containerize 并部署 ML 模型

### Competency Criteria
要成功完成本模块，你应该能够：
- 自信地运行和管理 Docker container
- 遵循 best practices 编写高效的 Dockerfile
- 使用多阶段构建优化 image
- 配置各种场景下的 container 网络
- 使用 volume 管理持久数据
- 使用 Docker Compose 编排多 container 应用
- 应用安全 best practices
- Containerize Python/ML 应用
- 排解常见 container 问题
- 部署 containerized 应用

### Capstone Project
**Containerized ML Application:**
构建一个完整的 ML 应用，展示：
- 优化的 Docker image
- Model serving container
- 训练 pipeline container
- 数据持久化
- Container 网络
- Docker Compose 编排
- 生产 best practices
- 文档

## Resources and References

> **Note:** 请参阅 `resources/recommended-reading.md` 获取学习材料、书籍和在线资源的完整列表。

### Essential Resources
- 官方 Docker 文档
- Docker Hub
- Play with Docker（基于浏览器的 playground）
- Docker CLI 参考

### Recommended Books
- "Docker Deep Dive" by Nigel Poulton
- "Docker in Action" by Jeff Nickoloff
- "Docker: Up & Running" by Karl Matthias and Sean Kane

### Online Learning
- Docker 官方教程
- Docker Labs（动手实践教程）
- Katacoda Docker 场景
- Docker Mastery 课程

### Tools
- Docker Desktop
- Docker Extensions
- Dive（image layer 分析）
- Hadolint（Dockerfile linter）
- Trivy（安全扫描器）

## Getting Started

### Step 1: Install Docker
1. 下载 Docker Desktop (Windows/Mac) 或 Docker Engine (Linux)
2. 完成安装
3. 使用 `docker version` 验证
4. 运行 hello-world container
5. 分配足够的资源

### Step 2: Familiarize with CLI
1. 练习基本命令
2. 探索 Docker Hub
3. 拉取示例 image
4. 以交互方式运行 container
5. 查看日志和检查 container

### Step 3: Begin with Lecture 1
- 阅读 container 基础
- 理解 Docker 架构
- 完成第一个 container 实验
- 尝试命令操作

### Step 4: Build Progressively
- 按顺序完成练习
- 创建 Dockerfile
- 尝试网络配置
- 练习 Compose
- 构建真实应用

## Tips for Success

1. **定期练习：** 在个人项目中使用 Docker
2. **阅读 Dockerfile：** 研究 Docker Hub 上的 image
3. **安全实验：** Container 是可丢弃的，大胆尝试
4. **尽早优化：** 从一开始就练习高效的 Dockerfile 编写
5. **理解 Layer：** 可视化理解 layer 的工作方式
6. **使用 .dockerignore：** 保持 build context 整洁
7. **版本化一切：** 适当标记 image
8. **监控资源：** 关注磁盘空间和运行中的 container
9. **安全第一：** 绝不提交密钥，以非 root 运行
10. **文档化：** 为 Dockerfile 和 Compose 文件添加注释

## Next Steps

完成本模块后，你将准备好：
- **Module 006:** 基本 ML 概念（containerize ML 模型）
- **Module 009:** 监控基础（监控 containerized 服务）
- **Module 010:** 云平台（在云中运行 container）
- **后续:** Kubernetes 和 container 编排

## Development Status

**Current Status:** 模板阶段 - 结构已完整搭建

**Available Now:**
- 完整的模块结构
- 详细的主题分解
- 讲座大纲
- 练习框架

**In Development:**
- 完整的讲座笔记及示例
- 逐步练习
- Dockerfile 模板
- Compose 文件示例
- ML 特定场景
- 视频演示

**Planned Updates:**
- Kubernetes 预览
- 高级网络
- 生产部署模式
- CI/CD 集成
- GPU container 设置

## Feedback and Contributions

欢迎贡献和反馈。通过以下方式帮助改进本模块：
- 报告问题
- 提出改进建议
- 分享 Dockerfile 示例
- 贡献练习

---

**Module Maintainer:** AI Infrastructure Curriculum Team
**Contact:** ai-infra-curriculum@joshua-ferguson.com
**Last Updated:** 2025-10-18
**Version:** 1.0.0-template
