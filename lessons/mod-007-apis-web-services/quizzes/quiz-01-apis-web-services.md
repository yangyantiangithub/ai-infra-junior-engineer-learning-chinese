# 模块 007 测验：API 和 Web 服务

## 测验信息

- **模块：** API 和 Web 服务
- **总题数：** 30
- **时间限制：** 45 分钟
- **及格分数：** 80%（24/30 正确）
- **涵盖主题：**
  - API 基础和 REST
  - FastAPI 框架
  - 认证和安全

## 说明

1. 仔细阅读每个问题
2. 选择题选择最佳答案
3. 有些问题可能有多个正确答案
4. 不扣分
5. 提交前可以复查和更改答案

---

## 第 1 节：REST API 基础（10 题）

### 问题 1
REST 代表什么？

A) Remote Execution State Transfer
B) Representational State Transfer
C) Remote State Transaction
D) Representational Service Technology

**正确答案：** B

**解释：** REST 代表 Representational State Transfer，是一种用于设计网络应用的架构风格。

---

### 问题 2
在 RESTful API 中，应该使用哪个 HTTP 方法来创建新资源？

A) GET
B) PUT
C) POST
D) PATCH

**正确答案：** C

**解释：** POST 用于创建新资源。PUT 也可以创建资源，但通常用于在特定 URI 处进行更新。

---

### 问题 3
以下哪项不是六项 REST 约束之一？

A) 无状态
B) 客户端-服务器
C) 数据库驱动
D) 可缓存

**正确答案：** C

**解释：** 六项 REST 约束是：客户端-服务器、无状态、可缓存、统一接口、分层系统和按需代码（可选）。数据库驱动不是 REST 约束。

---

### 问题 4
当资源成功创建时应返回什么 HTTP 状态码？

A) 200 OK
B) 201 Created
C) 204 No Content
D) 202 Accepted

**正确答案：** B

**解释：** 201 Created 是成功创建资源的适当状态码。它还应包含指向新资源 URI 的 Location 头部。

---

### 问题 5
在 REST 中，请求"幂等"是什么意思？

A) 请求必须加密
B) 请求可以被缓存
C) 多次发起相同请求与发起一次的效果相同
D) 请求需要认证

**正确答案：** C

**解释：** 幂等意味着多次相同的请求与单次请求具有相同的效果。GET、PUT、DELETE、HEAD 是幂等的；POST 不是。

---

### 问题 6
哪个 HTTP 状态码表示客户端未认证？

A) 400 Bad Request
B) 401 Unauthorized
C) 403 Forbidden
D) 404 Not Found

**正确答案：** B

**解释：** 401 Unauthorized 表示客户端必须进行认证。403 Forbidden 表示客户端已认证但未授权。

---

### 问题 7
PUT 和 PATCH 有什么区别？

A) PUT 用于创建，PATCH 用于删除
B) PUT 替换整个资源，PATCH 应用部分修改
C) PUT 需要认证，PATCH 不需要
D) 没有区别

**正确答案：** B

**解释：** PUT 用提供的表示替换整个资源，而 PATCH 对资源应用部分修改。

---

### 问题 8
以下哪项是正确的 RESTful URL 结构？

A) `/api/getUsers`
B) `/api/users/get`
C) `/api/users`
D) `/api/user/all`

**正确答案：** C

**解释：** RESTful URL 使用名词（资源）的复数形式，而不是动词。HTTP 方法表示操作。

---

### 问题 9
当超过速率限制时应返回什么 HTTP 状态码？

A) 400 Bad Request
B) 403 Forbidden
C) 429 Too Many Requests
D) 503 Service Unavailable

**正确答案：** C

**解释：** 429 Too Many Requests 是速率限制的标准状态码。它应包含 Retry-After 头部。

---

### 问题 10
在 RESTful 设计中，表示层次关系的正确方式是什么？

A) `/users/123/posts/456`
B) `/getUserPosts?userId=123&postId=456`
C) `/posts/456?user=123`
D) `/api/user-posts/123-456`

**正确答案：** A

**解释：** 层次关系使用嵌套路径表示：`/resource/{id}/subresource/{id}`。

---

## 第 2 节：FastAPI 框架（10 题）

### 问题 11
在 FastAPI 中使用类型提示的主要优势是什么？

A) 更快的代码执行
B) 自动请求验证和文档生成
C) 更小的代码体积
D) 仅提供更好的错误消息

**正确答案：** B

**解释：** FastAPI 使用 Python 类型提示来自动验证请求、序列化响应和生成 API 文档。

---

### 问题 12
FastAPI 使用哪个库进行数据验证？

A) Marshmallow
B) Cerberus
C) Pydantic
D) Voluptuous

**正确答案：** C

**解释：** FastAPI 使用 Pydantic 基于 Python 类型提示进行数据验证和序列化。

---

### 问题 13
在 FastAPI 中如何将查询参数标记为必填？

A) 添加 `required=True` 到参数
B) 不提供默认值
C) 使用 `Query(...)`
D) B 和 C 都对

**正确答案：** D

**解释：** 如果参数没有默认值，则是必填的。你也可以使用 `Query(...)` 显式标记为必填。

---

### 问题 14
FastAPI 中的依赖注入用于什么？

A) 仅用于数据库连接
B) 共享逻辑、管理状态、处理认证和减少代码重复
C) 仅用于认证
D) 向代码中注入错误

**正确答案：** B

**解释：** 依赖注入是 FastAPI 的一个强大特性，用于共享逻辑、管理资源、认证等。

---

### 问题 15
在 FastAPI 中哪个装饰器用于将函数标记为启动事件处理器？

A) `@app.startup()`
B) `@app.on_event("startup")`
C) `@app.before_start()`
D) `@app.init()`

**正确答案：** B

**解释：** `@app.on_event("startup")` 标记一个在应用程序启动时运行的函数，用于加载模型等操作。

---

### 问题 16
在 FastAPI 中添加后台任务的正确方式是什么？

A) 使用 `threading.Thread()`
B) 使用 `background_tasks.add_task()`
C) 使用 `asyncio.create_task()`
D) 使用 `multiprocessing.Process()`

**正确答案：** B

**解释：** FastAPI 提供 BackgroundTasks 用于在不阻塞响应的情况下运行任务。

---

### 问题 17
FastAPI 如何自动生成 API 文档？

A) 通过读取代码注释
B) 仅通过使用 docstring
C) 通过使用类型提示、Pydantic 模型和 endpoint 元数据
D) 仅通过手动配置

**正确答案：** C

**解释：** FastAPI 使用类型提示、Pydantic 模型和 endpoint 元数据自动生成 OpenAPI (Swagger) 文档。

---

### 问题 18
FastAPI 中 `response_model` 的作用是什么？

A) 验证请求体
B) 过滤和验证响应数据
C) 设置 HTTP 状态码
D) 启用 CORS

**正确答案：** B

**解释：** `response_model` 验证和序列化响应数据，确保只返回指定的字段。

---

### 问题 19
推荐在生产环境中使用哪个 HTTP 服务器运行 FastAPI？

A) Flask 的内置服务器
B) Uvicorn 或带 Uvicorn worker 的 Gunicorn
C) Django 的 runserver
D) SimpleHTTPServer

**正确答案：** B

**解释：** Uvicorn (ASGI 服务器) 或带 Uvicorn worker 的 Gunicorn 是生产环境 FastAPI 部署的推荐方案。

---

### 问题 20
`async def` 关键字在 FastAPI endpoint 中启用了什么？

A) 更快的代码执行
B) 并发处理 I/O 密集型操作
C) 并行处理
D) 自动缓存

**正确答案：** B

**解释：** `async def` 允许异步处理 I/O 密集型操作，支持并发请求处理。

---

## 第 3 节：认证和安全（10 题）

### 问题 21
认证和授权有什么区别？

A) 没有区别
B) 认证验证身份，授权确定权限
C) 认证用于 API，授权用于网站
D) 授权验证身份，认证确定权限

**正确答案：** B

**解释：** 认证回答"你是谁？"的问题，而授权回答"你能做什么？"的问题。

---

### 问题 22
以下哪种是存储密码最安全的方式？

A) 明文
B) MD5 哈希
C) bcrypt 或 argon2 哈希
D) Base64 编码

**正确答案：** C

**解释：** bcrypt 和 argon2 是专为密码哈希设计的，内置了加盐机制。MD5 已被破解，明文/base64 不安全。

---

### 问题 23
JWT 令牌的三个部分是什么？

A) 用户名、密码、时间戳
B) Header、payload、signature
C) 令牌、刷新、过期
D) 键、值、哈希

**正确答案：** B

**解释：** JWT 由三部分组成，用点分隔：header（算法和类型）、payload（声明/数据）和 signature（验证）。

---

### 问题 24
为什么 JWT 访问令牌应该设置较短的过期时间？

A) 节省服务器内存
B) 降低令牌被泄露的安全风险
C) 提高性能
D) 产生更多收入

**正确答案：** B

**解释：** 短生命周期的访问令牌限制了令牌被盗后的可利用时间窗口。刷新令牌允许获取新的访问令牌。

---

### 问题 25
什么是 CORS，为什么它很重要？

A) 一种加密数据的方式
B) 一种安全机制，控制哪些来源可以访问你的 API
C) 一种认证类型
D) 一种数据库技术

**正确答案：** B

**解释：** CORS (跨域资源共享) 是一种安全功能，控制哪些域名可以向你的 API 发起请求。

---

### 问题 26
API 请求中 JWT 令牌应放在哪个头部？

A) `X-API-Key`
B) `Authorization: Bearer <token>`
C) `Access-Token`
D) `X-Auth-Token`

**正确答案：** B

**解释：** 标准做法是使用 Authorization 头部和 "Bearer" 方案：`Authorization: Bearer <token>`。

---

### 问题 27
速率限制的目的是什么？

A) 使 API 更快
B) 防止滥用并确保公平的资源分配
C) 加密数据
D) 认证用户

**正确答案：** B

**解释：** 速率限制防止 API 滥用，保护基础设施，并确保用户之间公平分配资源。

---

### 问题 28
以下哪项不是推荐的安全头部？

A) X-Frame-Options
B) X-Content-Type-Options
C) X-Database-Password
D) Content-Security-Policy

**正确答案：** C

**解释：** X-Database-Password 不是一个真实的安全头部。永远不要在头部中暴露密码！

---

### 问题 29
什么是最小权限原则？

A) 默认使一切公开
B) 仅授予必要的最低权限
C) 给所有用户管理员权限
D) 移除所有安全措施

**正确答案：** B

**解释：** 最小权限原则意味着只授予用户/系统完成其任务所需的最低权限。

---

### 问题 30
为什么 API 应该始终使用 HTTPS 而不是 HTTP？

A) HTTPS 更快
B) HTTPS 加密传输中的数据，保护凭证和敏感数据
C) HTTPS 是法律要求的
D) HTTPS 使用更少的带宽

**正确答案：** B

**解释：** HTTPS 使用 TLS/SSL 加密所有传输中的数据，保护凭证、令牌和敏感数据免遭窃听。

---

## 答案键

### 第 1 节：REST API 基础
1. B - Representational State Transfer
2. C - POST 用于创建资源
3. C - 数据库驱动不是 REST 约束
4. B - 201 Created 用于成功创建
5. C - 多次请求效果相同
6. B - 401 用于未认证
7. B - PUT 替换，PATCH 修改
8. C - `/api/users` 是正确的
9. C - 429 用于速率限制
10. A - 嵌套路径表示层次关系

### 第 2 节：FastAPI 框架
11. B - 自动验证和文档
12. C - Pydantic
13. D - 无默认值或 Query(...)
14. B - 依赖注入的多种用途
15. B - @app.on_event("startup")
16. B - background_tasks.add_task()
17. C - 类型提示和 Pydantic
18. B - 过滤和验证响应
19. B - Uvicorn/Gunicorn
20. B - 并发 I/O 处理

### 第 3 节：认证和安全
21. B - 认证=身份，授权=权限
22. C - bcrypt 或 argon2
23. B - Header、payload、signature
24. B - 降低泄露风险窗口
25. B - 控制跨域访问
26. B - Authorization: Bearer <token>
27. B - 防止滥用
28. C - X-Database-Password（不是真实头部）
29. B - 最小权限
30. B - 传输加密

---

## 评分指南

- **27-30 正确（90-100%）：** 优秀！你对 API 和 Web 服务有很强的理解。
- **24-26 正确（80-86%）：** 良好！你以扎实的概念掌握通过了测试。
- **20-23 正确（67-76%）：** 一般。复习你遇到困难的部分材料。
- **低于 20（< 67%）：** 需要改进。在重新测试前重新学习讲座和练习。

## 下一步

完成此测验后：

1. **复习错误答案**并理解原因
2. **重新阅读相关的讲座部分**针对你答错的题目
3. **完成所有练习**进行动手实践
4. **构建一个项目**结合所有学到的概念
5. **进入模块 008：数据库和 SQL**

---

## 附加练习问题

练习以下场景：

1. 为图书管理系统设计一个 RESTful API
2. 为 ML 模型 API 实现 JWT 认证
3. 为前后端应用配置 CORS
4. 为不同用户层级设计速率限制策略
5. 创建包含示例的全面 API 文档

**复习时间：** 15-20 分钟
**建议重考间隔：** 如果分数 < 80%，3-5 天
