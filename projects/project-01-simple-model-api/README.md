# Project 01: 简单模型 API 部署

**角色：** 初级 AI 基础设施工程师（入门级）
**时长：** 60 小时（全职 1-2 周，兼职 2-3 周）
**难度：** 初学者
**前置要求：** 基本的 Python 知识，熟悉命令行

---

## 项目概述

欢迎来到你的第一个 AI 基础设施项目！在这个项目中，你将构建并部署一个 REST API，用于提供来自预训练机器学习模型的预测。这是每位 AI 基础设施工程师必须掌握的基础技能。

### 你将构建什么

你将创建一个可用于生产环境的 API，它能：
- 加载预训练的图像分类模型（ResNet-50 或 MobileNetV2）
- 通过 HTTP 请求接受图片上传
- 返回带有置信度分数的预测结果
- 在 Docker 容器中运行
- 部署到云平台（AWS、GCP 或 Azure）
- 包含监控和日志记录

### 实际场景

数据科学家在本地笔记本电脑或 Notebook 中训练模型。作为基础设施工程师，你的工作是获取这些模型文件，并通过可扩展、可靠的 API 将其提供给用户。本项目模拟的就是这个场景——你是 ML 研究与生产部署之间的桥梁。

---

## 学习目标

完成本项目后，你将能够：

1. **理解 ML 模型部署** - 学习如何加载预训练模型并提供预测服务
2. **构建 REST API** - 使用 Flask 或 FastAPI 设计和实现 HTTP API
3. **容器化应用** - 使用 Docker 打包应用以实现一致的部署
4. **部署到云平台** - 获得云平台（EC2、GCE 或 Azure VM）的实际操作经验
5. **实现监控** - 添加健康检查、日志记录和基本可观测性
6. **编写文档** - 创建清晰、全面的部署文档
7. **应用最佳实践** - 学习错误处理、验证和安全方面的行业标准

---

## 技术栈

### 核心技术
- **Python 3.11+** - 主要编程语言
- **Flask 3.0+ 或 FastAPI 0.100+** - REST API 的 Web 框架
- **PyTorch 2.0+ 或 TensorFlow 2.13+** - 模型推理的 ML 框架
- **Pillow (PIL)** - 图像处理库
- **Docker 24.0+** - 容器化平台
- **云平台** - AWS EC2、GCP Compute Engine 或 Azure VM

### 推荐工具
- **VS Code**（带 Python 扩展）用于开发
- **Postman 或 cURL** 用于 API 测试
- **Git/GitHub** 用于版本控制
- **Docker Desktop** 用于本地容器测试

---

## 项目需求

### 功能需求

#### FR-1：模型加载与推理
- 启动时加载预训练的图像分类模型
- 通过 HTTP POST 接受图片上传
- 根据模型要求预处理图片
- 返回前 5 个预测结果，包含类别名称和置信度分数
- 优雅地处理模型加载错误

**验收标准：**
- 应用启动时模型加载成功
- 对于 224x224 图片（CPU）推理延迟 < 500ms
- 返回格式正确的 JSON 响应
- 无效图片格式返回 HTTP 400 错误

#### FR-2：REST API 实现
- **`/predict`** 端点用于单张图片推理
- **`/health`** 端点用于健康检查
- **`/info`** 端点返回模型元数据
- 支持 multipart/form-data 图片上传
- 请求验证（文件大小限制、格式检查）

**验收标准：**
- 所有 3 个端点正确响应
- API 遵循 RESTful 约定
- 请求/响应格式已记录
- 返回适当的 HTTP 状态码

#### FR-3：错误处理
- 带有错误码的结构化错误响应
- 优雅地处理内存不足错误
- 验证图片输入（损坏、格式错误、过大）
- 实现请求超时处理（最长 30 秒）

**验收标准：**
- 所有错误情况均已测试和处理
- 无效输入不会导致崩溃
- 向用户返回清晰的错误信息

### 非功能需求

#### NFR-1：性能
- P99 预测延迟 < 1 秒
- 支持至少 10 个并发请求
- 正常负载下内存使用 < 2GB
- 启动时间 < 30 秒

#### NFR-2：可靠性
- 正常条件下 99% 正常运行时间
- 资源受限时优雅降级
- 健康检查端点响应时间 < 100ms

#### NFR-3：安全性
- 输入验证以防止注入攻击
- 文件大小限制以防 DOS（每张图片最大 10MB）
- 不执行上传文件中的任意代码
- 基本速率限制（可选：每 IP 每分钟 100 次请求）

#### NFR-4：可维护性
- 代码遵循 PEP 8 风格指南
- 函数包含文档字符串
- 配置外部化（非硬编码）
- 在适当的级别记录日志（INFO、WARNING、ERROR）

---

## 项目结构

```
project-01-simple-model-api/
├── README.md                      # 本文件
├── requirements.md                # 详细需求说明
├── architecture.md                # 系统架构与设计
├── src/
│   ├── README.md                  # 代码结构指南
│   ├── app.py                     # 主 Flask/FastAPI 应用（存根）
│   ├── model_loader.py            # 模型加载逻辑（存根）
│   └── config.py                  # 配置管理（存根）
├── tests/
│   ├── test_app.py                # API 端点测试（存根）
│   └── test_model.py              # 模型功能测试（存根）
├── docker/
│   ├── Dockerfile                 # 容器定义（存根）
│   └── docker-compose.yml         # 本地开发配置
└── .env.example                   # 环境变量模板
```

---

## 快速开始

### 前置条件

开始本项目前，请确保你已具备：

1. 已安装 **Python 3.11 或更高版本**
2. 已安装并运行 **Docker Desktop**
3. **Git** 用于版本控制
4. **一个云账户**（AWS Free Tier、GCP Free Trial 或 Azure Free Account）
5. **基本的 Python 知识**（函数、类、导入）
6. **熟悉命令行**（浏览目录、运行命令）

### 设置说明

1. **先阅读所有文档**
   ```bash
   # 按顺序阅读以下文件：
   # 1. README.md（本文件）- 概览和设置
   # 2. requirements.md - 详细需求
   # 3. architecture.md - 系统设计与架构
   # 4. src/README.md - 代码实现指南
   ```

2. **设置开发环境**
   ```bash
   # 创建项目目录
   mkdir model-api-deployment
   cd model-api-deployment

   # 创建 Python 虚拟环境
   python3.11 -m venv venv
   source venv/bin/activate  # Windows 上：venv\Scripts\activate

   # 安装依赖（你需要创建此文件）
   pip install flask torch torchvision pillow python-dotenv pytest
   ```

3. **审查代码存根**
   - 导航到 `src/` 目录
   - 阅读所有存根文件
   - 每个存根包含详细的 TODO 注释，说明要实现的内容
   - 提供了类型提示来指导你的实现

4. **实现应用**
   - 从 `src/config.py` 开始 - 配置管理
   - 然后 `src/model_loader.py` - 模型加载和推理
   - 最后 `src/app.py` - REST API 实现
   - 边开发边在 `tests/` 目录中编写测试

5. **本地测试**
   ```bash
   # 运行应用
   python src/app.py

   # 在另一个终端中测试端点
   curl http://localhost:5000/health
   curl http://localhost:5000/info
   curl -X POST -F "file=@test_image.jpg" http://localhost:5000/predict

   # 运行测试
   pytest tests/
   ```

6. **容器化**
   ```bash
   # 构建 Docker 镜像
   docker build -f docker/Dockerfile -t model-api:v1.0 .

   # 运行容器
   docker run -p 5000:5000 model-api:v1.0

   # 或者使用 Docker Compose
   docker-compose -f docker/docker-compose.yml up
   ```

7. **部署到云平台**
   - 按照 `docs/DEPLOYMENT.md` 中的部署指南操作（你需要创建此文件）
   - 选择一个云平台：AWS、GCP 或 Azure
   - 设置安全组/防火墙规则
   - 部署你的容器化应用
   - 测试公共端点

---

## 实施阶段

### 阶段 1：本地开发（20 小时）
- 搭建开发环境
- 实现模型加载逻辑
- 使用 Flask/FastAPI 构建 REST API
- 添加错误处理和验证
- 使用示例图片进行本地测试
- 编写基本文档

### 阶段 2：容器化（15 小时）
- 创建经过优化的 Dockerfile
- 构建并测试 Docker 镜像
- 优化镜像大小（目标 < 2GB）
- 创建用于本地开发的 docker-compose.yml
- 本地测试容器

### 阶段 3：云部署（20 小时）
- 设置云账户和权限
- 配置 VM 实例（EC2/GCE/Azure）
- 在 VM 上安装 Docker
- 部署容器化应用
- 配置安全性和网络
- 设置基本监控和日志记录

### 阶段 4：测试与验证（5 小时）
- 所有端点的功能测试
- 性能测试（延迟、并发）
- 多请求负载测试
- 安全验证
- 文档审查和更新

---

## 交付物

完成后，你应具备：

### 1. 源代码仓库
- [ ] 完整的 Python 应用，包含所有功能
- [ ] 单元测试，覆盖率 > 80%
- [ ] Dockerfile 和 docker-compose.yml
- [ ] 配置文件和示例
- [ ] .gitignore 和 requirements.txt

### 2. 文档
- [ ] 全面的 README，包含设置说明
- [ ] API 文档（端点、请求/响应格式）
- [ ] 部署指南，包含逐步说明
- [ ] 架构图
- [ ] 故障排除指南

### 3. 部署的应用
- [ ] 部署到云平台且可正常工作的 API
- [ ] 可通过公共 URL 访问
- [ ] 健康检查通过
- [ ] 云控制台中可查看日志
- [ ] 测试期间 99% 正常运行时间

### 4. 演示
- [ ] 5 分钟演示视频或现场演示
- [ ] 展示可正常工作的 API 及示例预测
- [ ] 解释架构和关键决策
- [ ] 讨论挑战和经验

---

## 评估标准

你的项目将按以下标准评估：

- **代码质量（25%）** - 组织、风格、文档、错误处理、测试
- **功能（30%）** - API 端点、模型推理、验证、错误响应、性能
- **容器化（15%）** - Dockerfile 质量、镜像大小、构建流程
- **部署（20%）** - 云设置、可访问性、监控、安全性
- **文档（10%）** - README、API 文档、部署指南、架构

**及格分数：** 70/100 分

详见 `requirements.md` 中的详细评分标准。

---

## 常见陷阱

1. **硬编码值** - 使用环境变量进行配置
2. **缺少错误处理** - 每个端点都应优雅地处理错误
3. **Docker 镜像过大** - 使用多阶段构建和精简基础镜像进行优化
4. **不安全的部署** - 始终正确配置安全组/防火墙
5. **缺少日志记录** - 日志记录对于调试生产问题至关重要
6. **内存泄漏** - 每次请求后清理资源
7. **仅测试正常路径** - 全面测试错误情况和边界情况

---

## 时间管理建议

- **不要急于完成阶段 1** - 扎实的本地实现能节省后期的调试时间
- **为 Docker 预留额外时间** - 容器调试可能很慢
- **预期部署会失败** - 计划 2-3 次迭代才能正确完成云部署
- **边做边记录** - 不要将文档留到最后
- **保留缓冲时间** - 预留 10-15 小时应对意外问题

---

## 资源

### 官方文档
- [Flask 文档](https://flask.palletsprojects.com/)
- [FastAPI 文档](https://fastapi.tiangolo.com/)
- [PyTorch 文档](https://pytorch.org/docs/)
- [Docker 文档](https://docs.docker.com/)
- [AWS EC2 文档](https://docs.aws.amazon.com/ec2/)

### 教程
- 使用 Flask 构建 ML API
- 用于机器学习的 Docker
- 部署到 AWS EC2

### 工具
- [Postman](https://www.postman.com/) - API 测试
- [Docker Desktop](https://www.docker.com/products/docker-desktop) - 本地容器
- [VS Code](https://code.visualstudio.com/) - 代码编辑器

---

## 获取帮助

- **答疑时间** - 参加每周答疑时间获取指导
- **讨论论坛** - 提问并帮助同伴
- **文档** - 在提问前仔细阅读文档
- **调试** - 大量使用 print 语句和日志记录
- **Stack Overflow** - 搜索类似问题（但不要复制代码）

---

## 后续步骤

完成本项目后：

1. **添加到作品集** - 创建一个带有详细 README 的 GitHub 仓库
2. **撰写博客文章** - 记录你的经验和学习成果
3. **更新简历** - 将 ML API 部署添加到你的技能和项目经历中
4. **迭代改进** - 考虑添加批量预测、模型版本管理或 GPU 加速等功能
5. **进入项目 2** - 下一个项目将基于此基础使用 Kubernetes

---

## 评估检查清单

提交前，请验证：

**代码质量**
- [ ] 代码遵循 PEP 8 风格指南
- [ ] 所有函数都有文档字符串
- [ ] 所有函数都有类型提示
- [ ] 没有硬编码值
- [ ] 已实现错误处理
- [ ] 已编写单元测试（覆盖率 > 80%）

**功能**
- [ ] `/health` 端点正常工作
- [ ] `/info` 端点正常工作
- [ ] `/predict` 端点正常工作
- [ ] 返回前 5 个预测结果
- [ ] 处理无效输入
- [ ] 满足性能目标

**容器化**
- [ ] Dockerfile 构建成功
- [ ] 镜像大小 < 2GB
- [ ] 容器无错误运行
- [ ] 已配置健康检查

**部署**
- [ ] 已部署到云平台
- [ ] 可通过公共 URL 访问
- [ ] 安全配置正确
- [ ] 日志可见

**文档**
- [ ] README 完整
- [ ] API 文档完整
- [ ] 部署指南完整
- [ ] 包含架构图

---

**祝你好运！记住：从简单开始，频繁测试，遇到困难时寻求帮助。**

**项目版本：** 1.0
**最后更新：** 2025 年 10 月
**维护者：** AI 基础设施课程团队 (ai-infra-curriculum@joshua-ferguson.com)
