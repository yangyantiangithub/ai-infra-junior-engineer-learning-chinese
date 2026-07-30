# 模块 007：API 和 Web 服务

## 模块概述

API（应用程序编程接口）和 Web 服务是现代分布式系统和 AI/ML 基础设施的支柱。本模块提供使用 FastAPI 和 Flask 等 Python 框架设计、构建、测试和使用 RESTful API 的全面培训。

你将学习 API 设计原则、HTTP 基础、认证机制，以及如何构建用于服务机器学习模型的生产级 API。这些技能对于将 ML 模型作为服务暴露、将 AI 功能集成到应用程序以及构建可扩展基础设施至关重要。

完成本模块后，你将能够自信地构建健壮的 API、实现认证、全面测试 endpoint，并在容器化环境中部署 API 服务。

## 学习目标

完成本模块后，你将能够：

1. **理解 HTTP 协议**和 RESTful API 原则
2. **设计高效的 API**遵循 REST 约定和最佳实践
3. **使用 FastAPI 构建 API**利用现代 Python 特性
4. **使用 Flask 构建 API**适用于轻量级应用
5. **实现认证**包括 JWT、OAuth 和 API key
6. **处理数据验证**使用 Pydantic 和请求验证
7. **全面测试 API**使用 pytest 和自动化测试工具
8. **编写 API 文档**使用 OpenAPI/Swagger 规范
9. **部署 API**在容器化环境中
10. **构建 ML 模型服务 API**用于推理和预测

## 先决条件

- 需要完成模块 001（Python 基础）
- 建议完成模块 004（用于基础设施的 Python）
- 建议完成模块 005（Docker 和容器）
- 建议完成模块 003（Git 版本控制）
- 理解 HTTP 基础知识
- 熟悉命令行和终端

**推荐设置：**
- 已安装 Python 3.9+
- 虚拟环境工具（venv 或 conda）
- Postman 或类似的 API 测试工具
- Docker（用于部署练习）
- 支持 Python 的文本编辑器或 IDE
- cURL 或 HTTPie 用于命令行测试

## 时间投入

- **总计预计时间：** 35-45 小时
- **讲座和阅读：** 12-15 小时
- **动手练习：** 18-22 小时
- **项目：** 5-8 小时

**推荐进度：**
- 兼职（5-10 小时/周）：4-5 周
- 全职（20-30 小时/周）：2 周

API 开发需要练习构建、测试和优化 endpoint。预计需要频繁迭代和实验。

## 模块结构

### 第 1 周：HTTP 和 REST 基础
- **主题：** HTTP 协议、REST 原则、API 设计
- **关键技能：** HTTP 方法、状态码、REST 约定
- **练习：** 设计 API、分析现有 API

### 第 2 周：使用 FastAPI 构建 API
- **主题：** FastAPI 框架、路径操作、请求/响应处理
- **关键技能：** FastAPI 路由、Pydantic 模型、验证
- **练习：** 构建 CRUD API、数据验证

### 第 3 周：认证和高级特性
- **主题：** 认证机制、中间件、错误处理
- **关键技能：** JWT 认证、依赖注入、测试
- **练习：** 安全 API、全面测试

### 第 4 周：部署和 ML 模型服务
- **主题：** API 部署、性能、ML 模型 API
- **关键技能：** 容器化、监控、模型服务
- **练习：** 生产部署、ML 推理 API

## 详细主题分解

### 1. HTTP 和 REST 基础（5-6 小时）

#### 1.1 HTTP 协议基础
- 客户端-服务器架构
- 请求-响应周期
- HTTP 方法（GET、POST、PUT、DELETE、PATCH）
- HTTP 状态码（2xx、3xx、4xx、5xx）
- HTTP 头（常见头部及其用途）
- 请求和响应结构
- 内容类型和序列化（JSON、XML）

#### 1.2 RESTful API 原则
- 什么是 REST？
- REST 约束（无状态、客户端-服务器、可缓存）
- 资源和 URI 设计
- CRUD 操作映射到 HTTP 方法
- 资源命名约定
- REST 成熟度模型（Richardson）
- REST vs GraphQL vs gRPC

#### 1.3 API 设计最佳实践
- 面向资源的设计
- URI 结构和层次结构
- 版本化策略（/v1/、/v2/）
- 查询参数 vs 路径参数
- 分页策略
- 过滤和排序
- 错误响应设计
- HATEOAS 概念

#### 1.4 API 文档
- 文档的重要性
- OpenAPI 规范（Swagger）
- API 文档标准
- 交互式文档
- 代码示例和 SDK
- 变更日志和版本化
- 开发者体验考虑

### 2. FastAPI 入门（7-9 小时）

#### 2.1 FastAPI 基础
- 为什么选择 FastAPI？
- FastAPI vs Flask 对比
- 安装和设置
- 第一个 FastAPI 应用程序
- 自动交互式文档
- 类型提示和现代 Python
- ASGI vs WSGI
- 性能特性

#### 2.2 路径操作
- 定义路由和 endpoint
- HTTP 方法装饰器
- 路径参数
- 查询参数
- 请求体
- 响应模型
- 状态码
- 标签和元数据

#### 2.3 Pydantic 模型
- 使用 Pydantic 进行数据验证
- 创建模型
- 字段验证
- 复杂数据类型
- 嵌套模型
- 模型配置
- 自定义验证器
- JSON Schema 生成

#### 2.4 请求处理
- 读取请求数据
- 表单数据
- 文件上传
- 请求头
- Cookie
- 请求对象访问
- 多个请求来源
- 数据验证和转换

#### 2.5 响应处理
- 响应模型
- 响应状态码
- 自定义响应类型
- JSON 响应
- 文件响应
- 流式响应
- 响应头
- 响应验证

#### 2.6 路径操作配置
- 响应模型配置
- 状态码和响应
- 标签和分组
- 摘要和描述
- 弃用
- 操作 ID
- 响应示例
- OpenAPI 元数据

### 3. 构建完整 API（6-8 小时）

#### 3.1 CRUD 操作
- 创建 endpoint（POST）
- 读取 endpoint（GET）
- 更新 endpoint（PUT、PATCH）
- 删除 endpoint（DELETE）
- RESTful URL 模式
- 数据库集成（SQLAlchemy）
- 错误处理
- 输入验证

#### 3.2 依赖注入
- FastAPI 依赖系统
- 创建依赖
- 可复用依赖
- 嵌套依赖
- 基于类的依赖
- 依赖作用域
- 带依赖的测试
- 常见依赖模式

#### 3.3 数据库集成
- SQLAlchemy 基础
- 定义模型
- 数据库会话
- 使用数据库进行 CRUD 操作
- 异步数据库操作
- 连接池
- 迁移（Alembic）
- 事务管理

#### 3.4 错误处理
- HTTP 异常
- 自定义异常处理器
- 验证错误
- 错误状态码
- 错误响应格式
- 记录错误
- 用户友好的错误消息
- 异常中间件

#### 3.5 后台任务
- 运行后台任务
- 使用场景（邮件、处理）
- 任务队列（Celery 预览）
- 异步任务处理
- 进度跟踪
- 后台任务中的错误处理
- 后台任务最佳实践

### 4. Flask Web 框架（5-6 小时）

#### 4.1 Flask 基础
- Flask 哲学
- 安装和设置
- 应用程序结构
- 第一个 Flask 应用程序
- 开发服务器
- 调试模式
- 应用上下文
- 请求上下文

#### 4.2 路由和视图
- 路由定义
- URL 构建
- HTTP 方法
- 变量规则
- URL 转换器
- 重定向和错误
- 视图装饰器
- 用于模块化应用的 Blueprint

#### 4.3 请求和响应
- Request 对象
- Response 对象
- JSON 响应（jsonify）
- 文件上传
- Cookie 和会话
- 自定义响应类型
- 错误处理器
- 请求前/后钩子

#### 4.4 Flask 扩展
- Flask-RESTful 用于 API
- Flask-CORS
- Flask-SQLAlchemy
- Flask-Migrate
- Flask-JWT-Extended
- Flask-Marshmallow
- 扩展生态系统
- 配置管理

#### 4.5 Flask vs FastAPI
- 性能对比
- 用例差异
- 生态系统成熟度
- 学习曲线
- 类型安全
- 文档生成
- 何时选择哪个
- 迁移策略

### 5. 认证和授权（6-8 小时）

#### 5.1 认证基础
- 认证 vs 授权
- 无状态认证
- 基于令牌的认证
- 基于会话的认证
- API key
- OAuth 2.0 概述
- 认证流程
- 安全考虑

#### 5.2 JWT 认证
- 什么是 JWT？
- JWT 结构（header、payload、signature）
- 创建 JWT
- 验证 JWT
- 令牌过期
- 刷新令牌
- 令牌存储
- 安全最佳实践

#### 5.3 在 FastAPI 中实现认证
- 用户模型和数据库
- 密码哈希（bcrypt）
- 登录 endpoint
- 令牌生成
- 受保护的路由
- 用于认证的依赖注入
- 当前用户依赖
- 基于角色的访问控制

#### 5.4 OAuth 2.0 集成
- OAuth 2.0 流程
- 授权码流程
- 隐式流程
- 客户端凭证流程
- 第三方认证
- 社交登录（Google、GitHub）
- OAuth 库（Authlib）
- 安全考虑

#### 5.5 API Key 和速率限制
- API key 生成
- API key 验证
- 速率限制策略
- 速率限制实现
- 节流
- 配额管理
- API key 最佳实践
- DDoS 防护基础

### 6. 测试 API（5-7 小时）

#### 6.1 测试基础
- 为什么要测试 API？
- API 测试类型
- 测试金字塔
- 测试驱动开发（TDD）
- 测试工具概述
- 用于 API 测试的 pytest
- 测试组织
- 测试数据管理

#### 6.2 API Endpoint 单元测试
- 使用 TestClient 测试（FastAPI）
- 测试 Flask 应用程序
- Mock 依赖
- 数据库测试策略
- Fixture 使用
- 参数化测试
- 测试错误情况
- 代码覆盖率

#### 6.3 集成测试
- 测试完整工作流
- 数据库集成测试
- 测试认证
- 测试文件上传
- 测试外部 API
- 测试数据库
- 事务回滚
- 测试性能

#### 6.4 API 测试工具
- Postman 用于手动测试
- Thunder Client（VS Code）
- HTTPie 用于 CLI 测试
- Newman 用于自动化 Postman 测试
- Pytest 插件
- API 测试框架
- 负载测试（Locust 预览）
- 契约测试

#### 6.5 测试自动化
- 持续集成
- 用于 API 测试的 GitHub Actions
- Pre-commit 钩子
- 测试覆盖率报告
- 自动化测试工作流
- 测试环境
- 测试最佳实践
- 维护测试套件

### 7. 生产环境和部署（5-7 小时）

#### 7.1 生产环境考虑
- ASGI 服务器（Uvicorn、Gunicorn）
- 工作进程和线程
- 反向代理（Nginx）
- HTTPS 和 SSL/TLS
- 环境配置
- 密钥管理
- 日志和监控
- 健康检查

#### 7.2 容器化 API
- FastAPI 的 Dockerfile
- Flask 的 Dockerfile
- 多阶段构建
- 用于开发的 Docker Compose
- 容器中的环境变量
- 容器网络
- 卷管理
- 容器安全

#### 7.3 API 性能
- 异步编程优势
- 数据库查询优化
- 缓存策略（Redis）
- 连接池
- 响应压缩
- 静态资源的 CDN
- API 性能分析
- 性能监控

#### 7.4 API 监控和日志
- 结构化日志
- 日志聚合
- 错误追踪（Sentry）
- 性能监控（New Relic、Datadog）
- 指标和仪表盘
- 告警
- 分布式追踪
- APM（应用性能监控）

#### 7.5 API 安全
- HTTPS 强制
- CORS 配置
- 输入验证
- SQL 注入防护
- XSS 防护
- CSRF 防护
- 安全头部
- 依赖扫描
- 安全最佳实践

### 8. ML 模型服务 API（4-5 小时）

#### 8.1 通过 API 服务模型
- 为什么将模型作为 API 服务？
- 模型加载策略
- 推理 endpoint 设计
- 请求/响应格式
- 批量 vs 实时推理
- API 中的模型版本化
- A/B 测试设置
- 模型元数据 endpoint

#### 8.2 使用 FastAPI 构建 ML API
- 加载 ML 模型（scikit-learn、PyTorch）
- API 中的预处理
- 预测 endpoint
- ML 输入验证
- 输出格式化
- ML 错误处理
- 模型健康检查
- 示例实现

#### 8.3 性能优化
- 模型缓存
- 批量预测优化
- 异步推理
- GPU 利用
- 模型量化
- 响应时间优化
- 扩展策略
- ML API 负载测试

#### 8.4 ML API 最佳实践
- ML API 版本化
- 模型注册表集成
- 特征验证
- 监控预测
- 处理边缘情况
- 可解释性 endpoint
- 模型重训练触发
- 生产 ML 模式

## 讲座大纲

### 讲座 1：HTTP 和 REST 基础（90 分钟）
- HTTP 协议概述
- REST 原则
- API 设计最佳实践
- 状态码和方法
- 面向资源的设计
- **实验：** 分析和设计 API

### 讲座 2：FastAPI 基础（90 分钟）
- FastAPI 简介
- 第一个 API 应用程序
- 路径操作
- 查询和路径参数
- 自动文档
- **实验：** 构建基本 FastAPI endpoint

### 讲座 3：请求和响应处理（90 分钟）
- Pydantic 模型
- 数据验证
- 请求体处理
- 响应模型
- 状态码
- **实验：** 带验证的 CRUD API

### 讲座 4：数据库集成（90 分钟）
- FastAPI 与 SQLAlchemy
- 数据库模型
- CRUD 操作
- 依赖注入
- 错误处理
- **实验：** 带数据库后端的 API

### 讲座 5：认证和安全（120 分钟）
- 认证概念
- JWT 实现
- 密码哈希
- 受保护的路由
- OAuth 2.0 概述
- **实验：** 带认证的安全 API

### 讲座 6：Flask Web 框架（90 分钟）
- Flask 基础
- 路由和视图
- Flask-RESTful
- Flask vs FastAPI
- 何时使用哪个
- **实验：** 使用 Flask 构建 API

### 讲座 7：测试和质量保证（90 分钟）
- API 测试策略
- 用于 API 的 pytest
- 测试客户端使用
- 集成测试
- 用于 API 的 CI/CD
- **实验：** 全面测试套件

### 讲座 8：ML 模型服务（120 分钟）
- 模型服务架构
- 构建 ML API
- 性能优化
- 部署策略
- 监控和维护
- **实验：** 完整 ML 推理 API

## 动手练习

### 练习类别

#### HTTP 和 REST（5 个练习）
1. HTTP 方法和状态码练习
2. API 设计挑战
3. 分析现有 API
4. REST 原则应用
5. URI 设计模式

#### FastAPI 基础（8 个练习）
6. 第一个 FastAPI 应用程序
7. 路径和查询参数
8. Pydantic 模型和验证
9. 请求体处理
10. 响应模型
11. CRUD 操作
12. 错误处理
13. 依赖注入

#### 认证（5 个练习）
14. 密码哈希实现
15. JWT 令牌生成
16. 创建受保护 endpoint
17. 用户认证系统
18. 基于角色的访问控制

#### 测试（5 个练习）
19. Endpoint 单元测试
20. 集成测试套件
21. 测试认证
22. Mock 依赖
23. CI/CD 测试自动化

#### 部署（5 个练习）
24. 容器化 FastAPI 应用
25. 生产配置
26. 健康检查 endpoint
27. 日志实现
28. 性能优化

#### ML 模型服务（4 个练习）
29. 加载和服务模型
30. 预测 endpoint
31. 批量预测 API
32. 完整 ML 服务

## 评估和评价

### 知识检查
- 每个主要部分后的测验
- HTTP 和 REST 概念
- 框架对比
- 安全原则
- 最佳实践评估

### 实践评估
- **基本 API：** 构建带验证的 CRUD API
- **认证 API：** 实现完整认证
- **测试 API：** 达到 >80% 测试覆盖率
- **ML API：** 部署模型服务 endpoint
- **生产 API：** 容器化、文档化、监控

### 能力标准
成功完成本模块后，你应该能够：
- 按照最佳实践设计 RESTful API
- 使用 FastAPI 和 Flask 构建 API
- 实现认证和授权
- 编写全面的 API 测试
- 适当处理错误
- 高效记录 API 文档
- 在容器中部署 API
- 通过 API 服务 ML 模型
- 监控和维护生产 API
- 优化 API 性能

### 顶点项目
**ML 模型服务 API：**
构建一个生产就绪的 ML API，包含以下特性：
- 模型推理 endpoint
- JWT 认证
- 请求验证
- 全面测试
- Docker 容器化
- API 文档
- 监控和日志
- 性能优化
- 完整部署指南

## 资源和参考

> **注意：** 参见 `resources/recommended-reading.md` 获取学习材料、书籍和在线资源的完整列表。

### 核心资源
- FastAPI 文档
- Flask 文档
- REST API 教程
- HTTP 规范

### 推荐书籍
- 《RESTful Web APIs》作者 Leonard Richardson
- 《Flask Web Development》作者 Miguel Grinberg
- 《Building Microservices》作者 Sam Newman

### 在线学习
- FastAPI 教程
- Real Python API 教程
- 使用 Python 进行测试驱动开发
- REST API 设计课程

### 工具
- Postman
- Thunder Client
- HTTPie
- Swagger Editor
- JWT.io
- Python requests 库

## 入门指南

### 步骤 1：设置环境
1. 安装 Python 3.9+
2. 创建虚拟环境
3. 安装 FastAPI 和依赖
4. 安装测试工具
5. 安装 Postman 或类似工具

### 步骤 2：复习先决条件
- Python 基础
- HTTP 基础知识
- 熟悉命令行
- Docker 知识（用于部署）

### 步骤 3：从讲座 1 开始
- 学习 HTTP 和 REST
- 设计第一个 API
- 完成初始练习

### 步骤 4：逐步构建
- 从简单的 endpoint 开始
- 逐步增加复杂度
- 全面测试
- 部署到容器
- 构建 ML 服务 API

## 成功提示

1. **测试一切：** 在构建时编写测试
2. **尽早文档：** 从一开始就使用 OpenAPI 文档
3. **安全第一：** 生产中永远不要跳过认证
4. **验证输入：** 始终验证和清理输入
5. **处理错误：** 提供清晰的错误信息
6. **版本化 API：** 从一开始就规划版本化
7. **监控性能：** 跟踪响应时间
8. **使用类型提示：** 利用 Python 类型提示
9. **遵循标准：** 坚持 REST 约定
10. **阅读文档：** FastAPI 文档非常优秀

## 下一步

完成本模块后，你将准备好：
- **模块 009：** 监控基础（监控你的 API）
- **模块 010：** 云平台（将 API 部署到云端）
- 高级主题：微服务、API 网关、服务网格

## 开发状态

**当前状态：** 模板阶段 - 结构完整

**现已提供：**
- 完整模块结构
- 详细主题分解
- 讲座大纲
- 练习框架

**开发中：**
- 完整讲座笔记
- 分步骤练习
- 代码示例
- ML 服务示例
- 测试模板
- 部署指南

**计划更新：**
- GraphQL 介绍
- gRPC 基础
- 微服务模式
- 服务网格概念
- 高级认证

## 反馈和贡献

通过以下方式帮助改进本模块：
- 报告问题
- 建议改进
- 贡献示例
- 分享最佳实践

---

**模块维护者：** AI 基础设施课程团队
**联系方式：** ai-infra-curriculum@joshua-ferguson.com
**最后更新：** 2025-10-18
**版本：** 1.0.0-template
