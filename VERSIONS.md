# 技术版本 - Junior Engineer 方向

**最后更新**: 2025 年 1 月

本文档规定了 AI Infrastructure Junior Engineer 课程中使用的所有技术的推荐和测试版本。

## 核心语言

| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **Python** | 3.11+ | 推荐：3.11 或 3.12。最低：3.11 |
| **Bash** | 5.0+ | 现代 Linux/macOS 的标准 |
| **SQL** | PostgreSQL 15+ | 用于数据库模块 |

## Python 包生态系统

### 核心 ML 框架
| 包名 | 版本 | 用途 |
|---------|---------|---------|
| **PyTorch** | 2.1.0+ | 深度学习框架 |
| **TensorFlow** | 2.15.0+ | 备选 ML 框架 |
| **scikit-learn** | 1.3.0+ | 传统 ML 算法 |
| **numpy** | 1.24.0+ | 数值计算 |
| **pandas** | 2.1.0+ | 数据处理 |

### ML 基础设施
| 包名 | 版本 | 用途 |
|---------|---------|---------|
| **transformers** | 4.35.0+ | Hugging Face transformers（LLM） |
| **onnx** | 1.15.0+ | 模型格式转换 |
| **onnxruntime** | 1.16.0+ | ONNX 推理 |

### Web 与 API
| 包名 | 版本 | 用途 |
|---------|---------|---------|
| **Flask** | 3.0.0+ | Web 框架 |
| **FastAPI** | 0.104.0+ | 现代异步 API 框架 |
| **requests** | 2.31.0+ | HTTP 客户端 |
| **uvicorn** | 0.24.0+ | ASGI 服务器 |

### 测试与质量
| 包名 | 版本 | 用途 |
|---------|---------|---------|
| **pytest** | 7.4.0+ | 测试框架 |
| **black** | 23.11.0+ | 代码格式化 |
| **ruff** | 0.1.6+ | 快速代码检查 |
| **mypy** | 1.7.0+ | 类型检查 |

### 监控与日志
| 包名 | 版本 | 用途 |
|---------|---------|---------|
| **prometheus-client** | 0.19.0+ | Prometheus 指标 |
| **psutil** | 5.9.0+ | 系统监控 |
| **structlog** | 23.2.0+ | 结构化日志 |

## 容器技术

| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **Docker** | 24.0+ | 容器运行时 |
| **Docker Compose** | 2.23+ | 多容器编排 |
| **containerd** | 1.7+ | 容器运行时（备选） |

### 基础镜像
| 镜像 | 标签 | 用途 |
|-------|-----|----------|
| `python` | 3.11-slim | 轻量级 Python 应用 |
| `python` | 3.11 | 完整 Python 环境 |
| `nvidia/cuda` | 12.2.0-runtime-ubuntu22.04 | 支持 GPU 的容器 |
| `pytorch/pytorch` | 2.1.0-cuda12.1-cudnn8-runtime | 带 GPU 的 PyTorch |

## Kubernetes

| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **Kubernetes** | 1.28+ | 编排平台 |
| **kubectl** | 1.28+ | CLI 工具（与集群版本匹配） |
| **Minikube** | 1.32+ | 本地开发集群 |
| **Kind** | 0.20+ | Docker 中的 Kubernetes |
| **Helm** | 3.13+ | 包管理器 |

### Kubernetes 附加组件
| 工具 | 版本 | 用途 |
|------|---------|---------|
| **metrics-server** | 0.6+ | 资源指标 |
| **NVIDIA device plugin** | 0.14+ | GPU 支持 |

## 云平台

### AWS
| 服务/工具 | 版本 | 说明 |
|--------------|---------|-------|
| **AWS CLI** | 2.15+ | 命令行接口 |
| **boto3** | 1.34+ | Python SDK |
| **Amazon Linux 2023** | 最新 | 推荐 AMI |

### 常用 AMI
| AMI 类型 | ID 模式 | 用途 |
|----------|-----------|----------|
| Amazon Linux 2023 | `al2023-ami-*` | 通用计算 |
| Ubuntu 22.04 LTS | `ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-*` | ML 工作负载 |
| Deep Learning AMI | `Deep Learning AMI GPU PyTorch 2.*` | GPU 训练 |

## 数据库

| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **PostgreSQL** | 15+ | 关系型数据库 |
| **Redis** | 7.2+ | 缓存/内存存储 |
| **MongoDB** | 7.0+ | NoSQL 文档数据库 |
| **SQLite** | 3.40+ | 嵌入式数据库 |

## 监控栈

| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **Prometheus** | 2.48+ | 指标采集 |
| **Grafana** | 10.2+ | 可视化 |
| **Alertmanager** | 0.26+ | 告警路由 |
| **Node Exporter** | 1.7+ | 系统指标 |
| **cAdvisor** | 0.47+ | 容器指标 |

### 日志
| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **Elasticsearch** | 8.11+ | 日志存储 |
| **Logstash** | 8.11+ | 日志处理 |
| **Kibana** | 8.11+ | 日志可视化 |
| **Fluentd** | 1.16+ | 日志收集器 |

## 基础设施即代码

| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **Terraform** | 1.6+ | IaC 工具 |
| **Ansible** | 2.16+ | 配置管理 |

### Terraform Provider
| Provider | 版本 | 说明 |
|----------|---------|-------|
| `hashicorp/aws` | ~> 5.0 | AWS provider |
| `hashicorp/random` | ~> 3.6 | 随机资源 |

## 版本控制

| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **Git** | 2.40+ | 版本控制 |
| **GitHub CLI** | 2.40+ | GitHub 操作 |

## 工作流编排

| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **Apache Airflow** | 2.7+ | 工作流编排 |

## 操作系统

| 系统 | 版本 | 说明 |
|-----|---------|-------|
| **Ubuntu** | 22.04 LTS | 推荐用于生产 |
| **Amazon Linux** | 2023 | AWS 优化 |
| **macOS** | 13+（Ventura） | 开发用 |
| **Windows** | 11 + WSL2 | Windows 开发 |

## CUDA 与 GPU

| 技术 | 版本 | 说明 |
|------------|---------|-------|
| **CUDA Toolkit** | 12.2+ | NVIDIA GPU 编程 |
| **cuDNN** | 8.9+ | 深度学习基础库 |
| **NVIDIA Driver** | 535+ | GPU 驱动 |
| **nvidia-docker** | 2.14+ | Docker GPU 支持 |

## 开发工具

| 工具 | 版本 | 说明 |
|------|---------|-------|
| **VS Code** | 1.85+ | 推荐 IDE |
| **JupyterLab** | 4.0+ | 交互式 notebook |
| **tmux** | 3.3+ | 终端复用器 |
| **vim/neovim** | 9.0+ / 0.9+ | 文本编辑器 |

## 版本策略

### 兼容性承诺

- **向后兼容**：练习适用于指定版本及更新版本
- **测试**：所有内容均使用推荐版本测试
- **更新**：文档每季度更新一次（1 月、4 月、7 月、10 月）

### 版本选择标准

1. **稳定性**：优先选择 LTS 或稳定版
2. **安全**：使用具有活跃安全支持的版本
3. **社区**：选择广泛采用的版本
4. **云兼容性**：匹配主要云服务商的默认配置

### 升级策略

- **次要版本**：可用时升级（例如 3.11.1 → 3.11.2）
- **主要版本**：升级前充分测试
- **依赖**：生产环境锁定版本，开发环境灵活使用

## 安装快速参考

### Python 环境搭建
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install python3.11 python3.11-venv python3-pip

# macOS
brew install python@3.11

# 验证
python3 --version  # 应显示 3.11.x 或 3.12.x
```

### Docker 安装
```bash
# Ubuntu
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# 验证
docker --version  # 应显示 24.0+
docker compose version  # 应显示 2.23+
```

### AWS CLI 安装
```bash
# Linux/macOS
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 验证
aws --version  # 应显示 2.15+
```

### Kubernetes 工具
```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install kubectl /usr/local/bin/

# Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# 验证
kubectl version --client
minikube version
```

### Terraform
```bash
# 使用包管理器
brew install terraform  # macOS
# 或
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip
unzip terraform_1.6.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/

# 验证
terraform version
```

## 已弃用的版本

以下版本在课程中不再受支持：

| 技术 | 已弃用版本 | 原因 | 迁移路径 |
|------------|-------------------|---------|----------------|
| Python | < 3.11 | 生命周期即将结束 | 升级到 3.11+ |
| TensorFlow | 1.x | Google 已弃用 | 升级到 2.15+ |
| Kubernetes | < 1.26 | 不再受支持 | 升级到 1.28+ |
| Docker | < 20.10 | 安全漏洞 | 升级到 24.0+ |

## 云服务商托管服务

### AWS 托管服务（备选版本）
| 服务 | 托管版本 | 自行管理版本 |
|---------|----------------|----------------------|
| RDS PostgreSQL | 15.x | PostgreSQL 15+ |
| ElastiCache Redis | 7.x | Redis 7.2+ |
| EKS | 1.28 | Kubernetes 1.28+ |
| SageMaker | 托管 | PyTorch 2.1.0+ |

## 破坏性变更与迁移指南

### Python 3.11 → 3.12
- **影响**：较小
- **变更**：性能改进、新语法特性
- **操作**：充分测试，根据需要更新类型提示

### TensorFlow 2.14 → 2.15
- **影响**：较低
- **变更**：Keras 3.0 集成
- **操作**：检查 Keras 代码的兼容性

### Kubernetes 1.27 → 1.28
- **影响**：中等
- **变更**：API 弃用
- **操作**：更新清单文件，检查弃用的 API

## 支持策略

| 版本类型 | 支持时长 | 更新频率 |
|--------------|------------------|------------------|
| 主要版本（例如 Python 3.11 → 3.12） | 提前 12 个月通知 | 每年 |
| 次要版本（例如 3.11.1 → 3.11.2） | 立即 | 随发布 |
| 安全补丁 | 立即 | 随发布 |

## 获取帮助

- **版本问题**：查看 GitHub Issues 了解已知问题
- **兼容性**：参阅官方文档
- **更新**：关注仓库中 VERSIONS.md 的更新

---

**注意**：本文档由课程团队维护。如有问题或版本特定问题，请开启 GitHub Issue。
