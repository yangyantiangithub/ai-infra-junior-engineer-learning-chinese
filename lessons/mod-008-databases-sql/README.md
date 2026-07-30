# Module 008: 数据库与 SQL

## 模块概述

数据库是 AI/ML 基础设施的基石，用于存储训练数据、模型元数据、应用状态和结果。本模块提供关系型数据库（PostgreSQL）、SQL 查询语言、数据库设计以及 Python 应用集成的全面培训。

你将学习数据库基础、从入门到高级的 SQL、schema 设计、索引、事务，以及如何使用 SQLAlchemy 在 Python 中操作数据库。你还将了解 NoSQL 数据库及其在 AI/ML 工作负载中的适用场景。

完成本模块后，你将能够自信地设计数据库 schema、编写高效的 SQL 查询、优化数据库性能，并将数据库集成到 Python 应用程序和 ML 管道中。

## 学习目标

完成本模块后，你将能够：

1. **理解关系型数据库概念**和 ACID 属性
2. **编写 SQL 查询**，从基本 SELECT 到复杂 JOIN 和子查询
3. **设计数据库 schema**，具备合理的范式和关系
4. **有效使用 PostgreSQL**，包括高级特性
5. **在 Python 中使用数据库**，通过 SQLAlchemy ORM
6. **优化查询性能**，使用索引和查询计划
7. **理解事务**和并发控制
8. **实现数据库迁移**和 schema 版本控制
9. **探索 NoSQL 数据库**并理解其适用场景
10. **将数据库应用于 ML 工作流**，用于存储特征、模型和结果

## 前置条件

- 必须完成模块 001（Python 基础）
- 建议完成模块 004（Python 用于基础设施）
- 建议完成模块 002（Linux 基础）
- 基本理解数据结构
- 熟悉命令行界面

**推荐环境配置：**
- 本地或通过 Docker 安装 PostgreSQL 14+
- Python 3.9+ 及虚拟环境
- 数据库客户端（pgAdmin、DBeaver 或 psql 命令行）
- 支持 SQL 的文本编辑器或 IDE
- Docker（可选，用于数据库容器）

## 时间投入

- **总计预估时间：** 40-50 小时
- **讲座与阅读：** 15-18 小时
- **动手练习：** 20-25 小时
- **项目：** 5-7 小时

**推荐节奏：**
- 兼职（5-10 小时/周）：4-6 周
- 全职（20-30 小时/周）：2-3 周

数据库技能通过实践培养。预计需要投入时间设计 schema、编写查询和优化性能。

## 模块结构

### 第 1 周：SQL 基础
- **主题：** 数据库概念、基本 SQL、数据类型、简单查询
- **关键技能：** SELECT、WHERE、INSERT、UPDATE、DELETE、基本过滤
- **练习：** 编写基本查询、数据操作

### 第 2 周：高级 SQL
- **主题：** JOIN、子查询、聚合、窗口函数
- **关键技能：** 复杂查询、分组、高级过滤
- **练习：** 多表查询、分析查询

### 第 3 周：数据库设计与 Python 集成
- **主题：** Schema 设计、范式、SQLAlchemy、迁移
- **关键技能：** ERD 设计、ORM 使用、数据库版本控制
- **练习：** 设计 schema、Python 数据库应用程序

### 第 4 周：性能与 NoSQL
- **主题：** 索引、查询优化、事务、NoSQL 入门
- **关键技能：** 性能调优、EXPLAIN 分析、NoSQL 基础
- **练习：** 优化练习、探索 NoSQL 数据库

## 详细主题分解

### 1. 数据库基础（5-6 小时）

#### 1.1 数据库简介
- 什么是数据库？
- 数据库管理系统（DBMS）
- 关系型与非关系型数据库
- 数据库在 AI/ML 中的应用场景
- ACID 属性（原子性、一致性、隔离性、持久性）
- CAP 定理概述
- 数据库架构

#### 1.2 关系型数据库概念
- 表、行和列
- 主键和外键
- 关系（一对一、一对多、多对多）
- 约束与数据完整性
- 引用完整性
- 范式概述
- Schema 设计原则

#### 1.3 PostgreSQL 简介
- 为什么选择 PostgreSQL？
- PostgreSQL vs MySQL vs SQLite
- 安装与配置
- psql 命令行工具
- PostgreSQL 架构
- 基本配置
- 用户与权限管理

#### 1.4 数据库工具
- psql 命令行客户端
- pgAdmin（GUI 客户端）
- DBeaver（通用数据库工具）
- SQL 编辑器和 IDE
- 数据库版本控制工具
- 备份与恢复工具
- 监控工具

### 2. SQL 基础（7-9 小时）

#### 2.1 数据类型
- 数值类型（INTEGER、DECIMAL、FLOAT）
- 字符串类型（VARCHAR、TEXT、CHAR）
- 日期和时间类型
- 布尔类型
- JSON 和 JSONB
- 数组
- 自定义类型
- 类型转换

#### 2.2 创建和管理表
- CREATE TABLE 语法
- 列定义
- 主键
- 外键
- 约束（NOT NULL、UNIQUE、CHECK）
- 默认值
- ALTER TABLE 操作
- DROP TABLE

#### 2.3 基本查询（SELECT）
- SELECT 语句结构
- 选择特定列
- 选择所有列（*）
- DISTINCT 关键字
- 列别名（AS）
- 限制结果（LIMIT）
- 排序结果（ORDER BY）
- 基本 WHERE 子句

#### 2.4 数据过滤
- WHERE 子句条件
- 比较运算符（=、!=、<、>、<=、>=）
- 逻辑运算符（AND、OR、NOT）
- IN 运算符
- BETWEEN 运算符
- LIKE 和模式匹配
- NULL 处理（IS NULL、IS NOT NULL）
- CASE 表达式

#### 2.5 数据操作（INSERT、UPDATE、DELETE）
- 插入单行
- 插入多行
- INSERT 与 SELECT 结合
- UPDATE 语句
- 带条件的 UPDATE
- DELETE 语句
- 带条件的 DELETE
- TRUNCATE vs DELETE
- RETURNING 子句

### 3. 高级 SQL（8-10 小时）

#### 3.1 连接（JOIN）
- 理解 JOIN
- INNER JOIN
- LEFT JOIN（LEFT OUTER JOIN）
- RIGHT JOIN（RIGHT OUTER JOIN）
- FULL OUTER JOIN
- CROSS JOIN
- 自连接
- 多表 JOIN
- JOIN 条件与性能

#### 3.2 子查询
- 什么是子查询？
- WHERE 子句中的子查询
- SELECT 子句中的子查询
- FROM 子句中的子查询
- 相关子查询
- EXISTS 和 NOT EXISTS
- IN 和 NOT IN 与子查询
- 子查询性能考虑

#### 3.3 聚合函数
- COUNT、SUM、AVG
- MIN 和 MAX
- GROUP BY 子句
- HAVING 子句
- 按多列分组
- 使用 DISTINCT 聚合
- 与 JOIN 结合使用的聚合函数
- 统计聚合

#### 3.4 窗口函数
- 窗口函数简介
- OVER 子句
- PARTITION BY
- 窗口函数中的 ORDER BY
- ROW_NUMBER、RANK、DENSE_RANK
- LAG 和 LEAD
- 累计和与移动平均
- 窗口帧规范
- 实际使用场景

#### 3.5 集合操作
- UNION 和 UNION ALL
- INTERSECT
- EXCEPT
- 组合查询
- 集合操作规则
- 性能考虑

#### 3.6 公用表表达式（CTE）
- WITH 子句
- 简单 CTE
- 多个 CTE
- 递归 CTE
- CTE vs 子查询
- CTE 的使用场景
- CTE 最佳实践

#### 3.7 PostgreSQL 高级特性
- JSON 和 JSONB 操作
- 数组操作
- 全文搜索
- 使用正则表达式的模式匹配
- 日期和时间函数
- 字符串函数
- 数学函数
- 条件表达式

### 4. 数据库设计（6-8 小时）

#### 4.1 Schema 设计原则
- 实体-关系建模
- 识别实体和属性
- 定义关系
- 基数和可选性
- ERD 符号与工具
- 将 ERD 转换为表
- 设计模式

#### 4.2 范式
- 什么是范式？
- 第一范式（1NF）
- 第二范式（2NF）
- 第三范式（3NF）
- 巴科斯-科德范式（BCNF）
- 反范式化（何时及为何）
- 范式在实践中的应用
- 权衡

#### 4.3 索引
- 什么是索引？
- B-tree 索引（默认）
- Hash 索引
- GiST 和 GIN 索引
- 创建索引
- 复合索引
- 唯一索引
- 部分索引
- 索引维护
- 何时使用索引
- 索引性能影响

#### 4.4 约束与数据完整性
- 主键约束
- 外键约束
- 唯一约束
- 检查约束
- 非空约束
- 约束命名约定
- 延迟约束
- 约束验证

#### 4.5 视图
- 创建视图
- 物化视图
- 视图的优势
- 可更新视图
- 视图性能
- 使用视图保障安全
- 视图维护

### 5. Python 与数据库集成（7-9 小时）

#### 5.1 Python 数据库驱动
- psycopg2（PostgreSQL 驱动）
- 数据库连接
- 执行查询
- 参数化查询
- 获取结果
- 连接池
- 错误处理
- 上下文管理器

#### 5.2 SQLAlchemy Core
- SQLAlchemy 简介
- Core vs ORM
- Engine 和连接
- Metadata 和 Table 对象
- SQL 表达式语言
- 执行语句
- 结果集
- 事务

#### 5.3 SQLAlchemy ORM
- 对象关系映射概念
- 定义模型
- 声明式基类
- 列类型
- 关系（一对多、多对多）
- 使用 ORM 查询
- Session 管理
- 懒加载 vs 即时加载

#### 5.4 使用 SQLAlchemy 进行 CRUD 操作
- 创建记录
- 读取记录（查询）
- 更新记录
- 删除记录
- 批量操作
- 查询过滤
- 排序与分页
- 聚合

#### 5.5 数据库迁移
- 为什么需要迁移？
- Alembic 简介
- 创建迁移
- 应用迁移
- 回滚策略
- 迁移最佳实践
- 团队协作与迁移
- 生产环境迁移策略

#### 5.6 数据库连接管理
- 连接池
- 连接生命周期
- 连接字符串
- 基于环境的配置
- 连接错误处理
- 连接测试
- 性能调优
- 云数据库连接

### 6. 性能与优化（5-7 小时）

#### 6.1 查询性能
- EXPLAIN 和 EXPLAIN ANALYZE
- 阅读查询计划
- 顺序扫描 vs 索引扫描
- 连接算法
- 查询优化策略
- 查询重写
- 避免 N+1 查询
- 缓存策略

#### 6.2 索引策略
- 识别慢查询
- 选择要索引的列
- 复合索引设计
- 索引覆盖
- 索引维护开销
- 监控索引使用
- 移除未使用的索引
- 索引膨胀

#### 6.3 事务与并发
- 事务基础（BEGIN、COMMIT、ROLLBACK）
- 实践中的 ACID 属性
- 隔离级别
- 并发问题（脏读、丢失更新）
- 锁机制
- 死锁与预防
- 事务最佳实践
- 长事务

#### 6.4 数据库维护
- VACUUM 和 ANALYZE
- 数据库统计信息
- 表膨胀管理
- 备份策略
- 时间点恢复
- 复制基础
- 监控数据库健康
- 性能调优参数

### 7. NoSQL 数据库（4-5 小时）

#### 7.1 NoSQL 简介
- 什么是 NoSQL？
- NoSQL vs SQL
- NoSQL 分类（文档、键值、图、列族）
- CAP 定理再探
- 最终一致性
- NoSQL 的使用场景
- 混合持久化

#### 7.2 文档数据库（MongoDB）
- 文档模型
- 集合与文档
- MongoDB 中的 CRUD 操作
- 查询文档
- MongoDB 中的索引
- 聚合管道
- 文档的 Schema 设计
- 何时使用 MongoDB

#### 7.3 键值存储（Redis）
- 键值模型
- Redis 数据类型
- Redis 命令
- 使用场景（缓存、会话、队列）
- Redis 持久化
- Python 中的 Redis
- 性能特性
- Redis vs Memcached

#### 7.4 AI/ML 中的 NoSQL
- 存储非结构化数据
- 特征存储
- 时序数据（InfluxDB、TimescaleDB）
- 用于嵌入向量的 vector database
- 用于关系的图数据库
- 选择合适的数据库
- 混合方法

### 8. ML 工作流中的数据库（3-4 小时）

#### 8.1 训练数据存储
- 存储原始数据
- 特征工程结果
- 数据版本控制
- 大数据集处理
- 数据质量检查
- 数据血缘追踪
- 分区策略

#### 8.2 模型元数据存储
- 模型注册数据库
- 实验跟踪
- 超参数存储
- 模型版本控制
- 性能指标
- 模型血缘
- 元数据查询

#### 8.3 推理结果存储
- 预测日志记录
- 实时 vs 批量存储
- 时序预测数据
- 结果聚合
- 分析查询模式
- 数据保留策略
- GDPR 合规

#### 8.4 ML 管道数据库
- 管道编排元数据
- 任务状态与日志记录
- 依赖跟踪
- 调度器数据库
- 监控与告警数据
- 审计追踪

## 讲座大纲

### 讲座 1：数据库基础（90 分钟）
- 数据库概念
- 关系模型
- PostgreSQL 简介
- 安装与配置
- 第一个数据库和表
- **实验：** PostgreSQL 环境搭建

### 讲座 2：SQL 基础（90 分钟）
- SELECT 查询
- WHERE 过滤
- 数据类型
- INSERT、UPDATE、DELETE
- 基本数据操作
- **实验：** 基本 SQL 操作

### 讲座 3：高级 SQL 查询（120 分钟）
- JOIN 深入探讨
- 子查询
- 聚合函数
- GROUP BY 和 HAVING
- 窗口函数
- **实验：** 复杂查询挑战

### 讲座 4：数据库设计（90 分钟）
- Schema 设计原则
- 范式
- ERD 建模
- 关系与约束
- 设计模式
- **实验：** 设计数据库 schema

### 讲座 5：索引与性能（90 分钟）
- 理解索引
- 查询优化
- EXPLAIN 计划
- 性能调优
- 最佳实践
- **实验：** 查询优化练习

### 讲座 6：Python 与 SQLAlchemy（120 分钟）
- psycopg2 基础
- SQLAlchemy Core
- SQLAlchemy ORM
- 数据库迁移
- Python 数据库模式
- **实验：** Python 数据库应用程序

### 讲座 7：事务与并发（90 分钟）
- 事务基础
- 隔离级别
- 锁与并发
- 死锁处理
- 最佳实践
- **实验：** 事务场景

### 讲座 8：NoSQL 与 ML 数据库（90 分钟）
- NoSQL 概述
- MongoDB 和 Redis
- ML 工作流中的数据库
- 选择合适的数据库
- 混合持久化
- **实验：** NoSQL 动手实践

## 动手练习

### 练习分类

#### SQL 基础（10 个练习）
1. 创建表和 schema
2. 基本 SELECT 查询
3. 过滤与排序
4. 数据操作（INSERT、UPDATE、DELETE）
5. 处理 NULL 值
6. 日期和时间操作
7. 字符串操作
8. CASE 表达式
9. 子查询基础
10. 基本聚合

#### 高级 SQL（10 个练习）
11. INNER 和 OUTER 连接
12. 自连接
13. 复杂子查询
14. 窗口函数
15. CTE（公用表表达式）
16. 递归查询
17. 集合操作
18. JSON 操作
19. 高级聚合
20. 查询优化挑战

#### 数据库设计（5 个练习）
21. ERD 设计练习
22. 范式练习
23. Schema 实现
24. 约束设计
25. 索引策略

#### Python 集成（8 个练习）
26. psycopg2 基础
27. SQLAlchemy Core 查询
28. ORM 模型定义
29. 使用 ORM 进行 CRUD
30. ORM 中的关系
31. 使用 Alembic 进行数据库迁移
32. 连接池
33. 完整的 Python 数据库应用

#### 性能（4 个练习）
34. 查询性能分析
35. 索引优化
36. 事务管理
37. 并发场景

#### NoSQL（3 个练习）
38. MongoDB 操作
39. Redis 缓存
40. NoSQL 使用场景选择

## 评估与考核

### 知识检查
- 每个主要部分后的测验
- SQL 语法理解
- 数据库设计原则
- 性能概念
- NoSQL 理解

### 实践评估
- **SQL 能力：** 编写 20+ 个不同复杂度的查询
- **Schema 设计：** 为应用程序设计范式化数据库
- **Python 集成：** 构建带数据库后端的应用
- **性能：** 使用索引优化慢查询
- **ML 数据库：** 为 ML 工作流设计数据库

### 能力标准
要成功完成本模块，你应该能够：
- 独立编写复杂 SQL 查询
- 设计范式化数据库 schema
- 有效使用 PostgreSQL
- 将数据库与 Python 应用程序集成
- 优化查询性能
- 管理事务与并发
- 选择合适的数据库类型
- 将数据库应用于 ML 工作流
- 排查数据库问题
- 实现数据库迁移

### 结业项目
**ML 实验跟踪数据库：**
设计并实现完整的数据库系统，用于：
- 实验元数据存储
- 模型版本控制
- 训练指标
- 超参数跟踪
- 结果查询与分析
- 用于交互的 Python API
- 性能优化
- 迁移管理
- 完整文档

## 资源与参考

> **注意：** 参见 `resources/recommended-reading.md` 获取学习材料、书籍和在线资源的完整列表。

### 核心资源
- PostgreSQL 文档
- SQLAlchemy 文档
- SQL 教程（W3Schools、SQLBolt）
- PostgreSQL 练习

### 推荐书籍
- 《SQL Cookbook》—— Anthony Molinaro
- 《PostgreSQL: Up and Running》—— Regina Obe
- 《Learning SQL》—— Alan Beaulieu
- 《Database Design for Mere Mortals》—— Michael Hernandez

### 在线学习
- Mode SQL Tutorial
- SQLZoo
- PostgreSQL Tutorial
- SQLAlchemy 教程
- LeetCode 数据库问题

### 工具
- pgAdmin
- DBeaver
- DataGrip
- psql
- TablePlus

## 入门指南

### 步骤 1：安装 PostgreSQL
1. 本地安装或通过 Docker 安装 PostgreSQL
2. 验证安装
3. 创建测试数据库
4. 安装 GUI 客户端
5. 测试连接

### 步骤 2：设置 Python 环境
1. 安装 psycopg2 和 SQLAlchemy
2. 创建虚拟环境
3. 测试 Python 数据库连接
4. 安装 Alembic 用于迁移

### 步骤 3：从讲座 1 开始
- 学习数据库基础
- 设置 PostgreSQL
- 创建第一个表
- 执行基本查询

### 步骤 4：定期练习
- 完成 SQL 练习
- 设计练习 schema
- 构建 Python 应用程序
- 优化查询
- 参与项目

## 成功提示

1. **每天练习 SQL：** 定期练习培养查询娴熟度
2. **可视化 Schema：** 在实现前绘制 ERD
3. **使用 EXPLAIN：** 始终检查慢查询的查询计划
4. **从范式开始：** 首先设计范式化 schema
5. **从示例中学习：** 研究流行应用的 schema
6. **使用迁移：** 始终对 schema 变更进行版本控制
7. **测试事务：** 练习隔离级别和锁
8. **监控性能：** 使用数据库监控工具
9. **阅读文档：** PostgreSQL 文档内容全面
10. **构建项目：** 将技能应用于实际应用程序

## 后续步骤

完成本模块后，你将准备好：
- **模块 009：** 监控基础（监控数据库）
- **模块 010：** 云平台（托管数据库）
- 高级主题：数据库管理、复制、分片

## 开发状态

**当前状态：** 模板阶段——结构已完整搭建

**现已提供：**
- 完整的模块结构
- 详细主题分解
- 讲座大纲
- 练习框架

**开发中：**
- 完整讲座笔记
- SQL 练习集
- Schema 设计示例
- Python 集成教程
- 性能调优指南
- ML 数据库模式

**计划更新：**
- PostgreSQL 高级特性
- 数据库管理
- 复制与高可用
- 分片策略
- 时序数据库
- Vector database

## 反馈与贡献

帮助改进本模块：
- 报告问题
- 分享 schema 设计
- 贡献练习
- 推荐资源

---

**模块维护者：** AI 基础设施课程团队
**联系方式：** ai-infra-curriculum@joshua-ferguson.com
**最后更新：** 2025-10-18
**版本：** 1.0.0-template
