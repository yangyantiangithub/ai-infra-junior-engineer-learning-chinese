# Lecture 01: 数据库基础与 SQL 入门

## 目录
1. [引言](#introduction)
2. [什么是数据库？](#what-is-a-database)
3. [数据库类型](#types-of-databases)
4. [关系型数据库概念](#relational-database-concepts)
5. [SQL 简介](#introduction-to-sql)
6. [设置数据库环境](#setting-up-your-database-environment)
7. [创建数据库和表](#creating-databases-and-tables)
8. [基本 SQL 查询（SELECT）](#basic-sql-queries-select)
9. [数据过滤（WHERE）](#filtering-data-where)
10. [排序与限制结果](#sorting-and-limiting-results)
11. [插入数据](#inserting-data)
12. [更新数据](#updating-data)
13. [删除数据](#deleting-data)
14. [ML 基础设施中的数据库](#databases-in-ml-infrastructure)
15. [最佳实践](#best-practices)
16. [总结与关键要点](#summary-and-key-takeaways)

## 引言

数据库是现代应用程序的基石，存储和管理着驱动从 Web 应用到机器学习系统一切的数据。对于 AI/ML 基础设施，数据库在存储训练数据、模型元数据、实验结果和生产指标方面发挥着关键作用。

本讲座将向你介绍数据库基础和 SQL（结构化查询语言），这是与关系型数据库交互的标准语言。

### 学习目标

在本讲座结束时，你将：
- 理解什么是数据库及其重要性
- 区分不同类型的数据库
- 理解关系型数据库概念（表、行、列、键）
- 编写基本 SQL 查询以检索数据
- 过滤、排序和限制查询结果
- 插入、更新和删除数据
- 将数据库概念应用于 ML 基础设施场景
- 遵循 SQL 最佳实践

### 前置条件
- 基本编程知识（推荐 Python）
- 命令行熟练度
- 理解数据结构（数组、字典）

### 预计时间
4-5 小时（包括动手练习）

## 什么是数据库？

### 定义

**数据库**是以电子方式存储的结构化数据的有机集合。数据库旨在：
- 高效存储大量数据
- 快速检索数据
- 维护数据完整性和一致性
- 支持多个用户并发访问
- 提供数据安全和备份能力

### 为什么使用数据库？

**没有数据库时：**
```python
# 将数据存储在文件中（问题重重！）
import json

# 保存数据
with open('models.json', 'w') as f:
    json.dump({"model1": {...}, "model2": {...}}, f)

# 问题：
# - 无并发访问控制
# - 难以高效查询
# - 无数据完整性保证
# - 难以扩展
# - 无事务支持
```

**使用数据库时：**
```sql
-- 高效、可扩展、并发访问
SELECT * FROM models WHERE accuracy > 0.9;
-- 从数百万条记录中瞬间获取结果！
```

### 数据库的主要优势

1. **数据完整性**：强制执行规则和约束
2. **并发性**：多个用户可以同时访问
3. **可扩展性**：处理不断增长的数据量
4. **性能**：针对快速查询进行了优化
5. **安全性**：访问控制和身份验证
6. **ACID 属性**：原子性、一致性、隔离性、持久性
7. **备份与恢复**：数据保护机制

## 数据库类型

### 1. 关系型数据库（SQL）

将数据存储在具有预定义 schema 的结构化表中。

**示例：**
- PostgreSQL（开源、强大）
- MySQL/MariaDB（流行、适合 Web）
- SQLite（轻量级、基于文件）
- Oracle（企业级）
- Microsoft SQL Server（企业级）

**特点：**
- 带 schema 的结构化数据
- ACID 事务
- SQL 查询语言
- 通过外键建立关系
- 强一致性

**使用场景：**
- 金融系统
- 电子商务
- 用户管理
- ML 实验跟踪
- 模型元数据存储

### 2. NoSQL 数据库

以灵活的非表格格式存储数据。

**类型：**

**a) 文档数据库**
- MongoDB、CouchDB
- 存储类似 JSON 的文档
- 灵活的 schema

```json
{
  "model_id": "bert-v1",
  "metrics": {
    "accuracy": 0.92,
    "f1": 0.89
  },
  "tags": ["nlp", "transformer"]
}
```

**b) 键值存储**
- Redis、DynamoDB
- 简单的键值对
- 极快

```
model:bert-v1 → {"accuracy": 0.92}
```

**c) 列族存储**
- Cassandra、HBase
- 针对大规模数据优化

**d) 图数据库**
- Neo4j、Amazon Neptune
- 存储关系和连接

**使用场景：**
- 缓存（Redis）
- 实时分析
- 非结构化数据
- 水平扩展
- ML 中的特征存储

### 3. 对比：SQL vs NoSQL

| 方面 | SQL（关系型） | NoSQL |
|--------|------------------|-------|
| Schema | 固定、预定义 | 灵活、动态 |
| 可扩展性 | 垂直（纵向扩展） | 水平（横向扩展） |
| 事务 | ACID 保证 | 最终一致性 |
| 查询 | 强大的 SQL | 因类型而异 |
| 数据结构 | 表、行、列 | 文档、键值等 |
| 最适合 | 结构化数据、复杂查询 | 非结构化数据、高容量 |
| ML 使用场景 | 元数据、实验 | 特征、嵌入、日志 |

### 何时使用哪种？

**使用 SQL 的情况：**
- 数据具有清晰的结构
- 需要复杂查询和连接
- ACID 事务至关重要
- 数据完整性是首要任务
- 示例：模型注册、实验跟踪

**使用 NoSQL 的情况：**
- Schema 频繁变化
- 需要大规模水平扩展
- 处理非结构化数据
- 速度优先于一致性
- 示例：特征缓存、日志存储、嵌入

## 关系型数据库概念

### 表

表是关系型数据库中的基本结构。可以将它们视为电子表格。

```
models 表：
┌────────┬─────────────┬───────────┬──────────┐
│ id     │ name        │ framework │ accuracy │
├────────┼─────────────┼───────────┼──────────┤
│ 1      │ BERT-base   │ PyTorch   │ 0.92     │
│ 2      │ ResNet-50   │ TensorFlow│ 0.88     │
│ 3      │ GPT-2       │ PyTorch   │ 0.95     │
└────────┴─────────────┴───────────┴──────────┘
```

**组成部分：**
- **表名**：`models`
- **列**：`id`、`name`、`framework`、`accuracy`
- **行**：每一行是一条记录（以上共 3 行）

### 列（字段）

列定义数据的属性。

```sql
列名      | 数据类型     | 约束
------------|--------------|-------------
id          | INTEGER      | PRIMARY KEY
name        | VARCHAR(100) | NOT NULL
framework   | VARCHAR(50)  | NOT NULL
accuracy    | DECIMAL(5,4) | CHECK (accuracy >= 0 AND accuracy <= 1)
created_at  | TIMESTAMP    | DEFAULT CURRENT_TIMESTAMP
```

### 行（记录）

每一行代表一个实体或记录。

```sql
-- 一行（一个模型）
(1, 'BERT-base', 'PyTorch', 0.92, '2024-01-15 10:30:00')
```

### 主键

**主键**唯一标识表中的每一行。

```sql
CREATE TABLE models (
    id INTEGER PRIMARY KEY,  -- 唯一标识符
    name VARCHAR(100)
);
```

**属性：**
- 每一行必须唯一
- 不能为 NULL
- 每个表只有一个主键
- 通常是自增的

### 外键

**外键**在表之间建立关系。

```sql
CREATE TABLE experiments (
    experiment_id INTEGER PRIMARY KEY,
    model_id INTEGER,
    accuracy DECIMAL(5,4),
    FOREIGN KEY (model_id) REFERENCES models(id)
);
```

这将实验与模型关联起来：
```
models 表：              experiments 表：
┌────┬──────────┐      ┌──────────────┬──────────┬──────────┐
│ id │ name     │      │ experiment_id│ model_id │ accuracy │
├────┼──────────┤      ├──────────────┼──────────┼──────────┤
│ 1  │ BERT     │ ←────┤ 101          │ 1        │ 0.92     │
│ 2  │ ResNet   │ ←────┤ 102          │ 2        │ 0.88     │
└────┴──────────┘      │ 103          │ 1        │ 0.93     │
                        └──────────────┴──────────┴──────────┘
```

### 数据类型

常见 SQL 数据类型：

**数值型：**
```sql
INTEGER         -- 整数：1、42、-10
BIGINT          -- 大整数：9223372036854775807
DECIMAL(10,2)   -- 固定精度：123.45
FLOAT           -- 浮点数：3.14159
```

**文本型：**
```sql
CHAR(10)        -- 固定长度：'BERT      '（填充）
VARCHAR(100)    -- 可变长度：'BERT-base'
TEXT            -- 无限制文本
```

**日期/时间：**
```sql
DATE            -- 仅日期：'2024-10-18'
TIME            -- 仅时间：'14:30:00'
TIMESTAMP       -- 日期和时间：'2024-10-18 14:30:00'
```

**布尔型：**
```sql
BOOLEAN         -- TRUE 或 FALSE
```

**其他：**
```sql
JSON            -- JSON 数据（PostgreSQL）
ARRAY           -- 数组（PostgreSQL）
BLOB            -- 二进制数据（图像、文件）
```

## SQL 简介

### 什么是 SQL？

**SQL（结构化查询语言）**是与关系型数据库交互的标准语言。

**SQL 分类：**

1. **DDL（数据定义语言）**：定义结构
   - `CREATE`、`ALTER`、`DROP`

2. **DML（数据操作语言）**：操作数据
   - `SELECT`、`INSERT`、`UPDATE`、`DELETE`

3. **DCL（数据控制语言）**：控制访问
   - `GRANT`、`REVOKE`

4. **TCL（事务控制语言）**：管理事务
   - `COMMIT`、`ROLLBACK`

### SQL 语法约定

```sql
-- SQL 关键字通常大写（约定）
SELECT * FROM models;

-- 但小写也可以
select * from models;

-- 语句以分号结尾
SELECT name FROM models;

-- 注释
-- 单行注释
/* 多行
   注释 */
```

**最佳实践：**
- SQL 关键字使用大写
- 表/列名使用小写
- 缩进以提高可读性
- 语句以分号结尾

## 设置数据库环境

### 选项 1：SQLite（推荐用于学习）

SQLite 基于文件，无需服务器设置。

```bash
# 安装（通常 Linux/Mac 已预装）
sudo apt-get install sqlite3  # Ubuntu/Debian

# 创建数据库
sqlite3 ml_database.db

# 你将看到：
# SQLite version 3.x.x
# sqlite>
```

### 选项 2：PostgreSQL（生产级）

```bash
# 安装 PostgreSQL
sudo apt-get install postgresql postgresql-contrib

# 启动服务
sudo systemctl start postgresql

# 访问 PostgreSQL
sudo -u postgres psql

# 创建数据库
CREATE DATABASE ml_database;

# 连接到数据库
\c ml_database
```

### 选项 3：Docker（可移植）

```bash
# 在 Docker 中运行 PostgreSQL
docker run --name postgres-ml \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=ml_database \
  -p 5432:5432 \
  -d postgres:14

# 连接
docker exec -it postgres-ml psql -U postgres -d ml_database
```

### Python 数据库连接

```python
# SQLite
import sqlite3
conn = sqlite3.connect('ml_database.db')
cursor = conn.cursor()

# PostgreSQL（需要 psycopg2）
import psycopg2
conn = psycopg2.connect(
    host="localhost",
    database="ml_database",
    user="postgres",
    password="mypassword"
)
cursor = conn.cursor()
```

## 创建数据库和表

### 创建数据库

```sql
-- PostgreSQL/MySQL
CREATE DATABASE ml_database;

-- 使用数据库
USE ml_database;  -- MySQL
\c ml_database    -- PostgreSQL
```

### 创建表

```sql
CREATE TABLE models (
    id INTEGER PRIMARY KEY AUTOINCREMENT,  -- SQLite
    -- id SERIAL PRIMARY KEY,               -- PostgreSQL
    name VARCHAR(100) NOT NULL,
    framework VARCHAR(50) NOT NULL,
    version VARCHAR(20) DEFAULT '1.0.0',
    accuracy DECIMAL(5, 4),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**约束：**
- `PRIMARY KEY`：唯一标识符
- `NOT NULL`：值必填
- `DEFAULT`：未指定时的默认值
- `UNIQUE`：所有值必须唯一
- `CHECK`：自定义验证

### 示例：ML 模型注册

```sql
-- Models 表
CREATE TABLE models (
    model_id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(100) NOT NULL UNIQUE,
    framework VARCHAR(50) NOT NULL,
    model_type VARCHAR(50),
    version VARCHAR(20) DEFAULT '1.0.0',
    accuracy DECIMAL(5, 4),
    f1_score DECIMAL(5, 4),
    precision_score DECIMAL(5, 4),
    recall_score DECIMAL(5, 4),
    model_path VARCHAR(255),
    created_by VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) DEFAULT 'training',
    CHECK (accuracy >= 0 AND accuracy <= 1),
    CHECK (status IN ('training', 'ready', 'deployed', 'archived'))
);

-- Experiments 表
CREATE TABLE experiments (
    experiment_id INTEGER PRIMARY KEY AUTOINCREMENT,
    model_id INTEGER NOT NULL,
    experiment_name VARCHAR(100),
    dataset_name VARCHAR(100),
    batch_size INTEGER,
    learning_rate DECIMAL(10, 8),
    epochs INTEGER,
    accuracy DECIMAL(5, 4),
    loss DECIMAL(10, 6),
    training_time_seconds INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (model_id) REFERENCES models(model_id)
);

-- Predictions 表（用于日志记录）
CREATE TABLE predictions (
    prediction_id INTEGER PRIMARY KEY AUTOINCREMENT,
    model_id INTEGER NOT NULL,
    input_data TEXT,
    prediction TEXT,
    confidence DECIMAL(5, 4),
    latency_ms INTEGER,
    user_id VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (model_id) REFERENCES models(model_id)
);
```

### 查看表结构

```sql
-- SQLite
.schema models

-- PostgreSQL
\d models

-- MySQL
DESCRIBE models;

-- 标准 SQL
SHOW COLUMNS FROM models;
```

## 基本 SQL 查询（SELECT）

### 选择所有列

```sql
-- 从 models 表获取所有数据
SELECT * FROM models;
```

**输出：**
```
model_id | name      | framework | accuracy | created_at
---------|-----------|-----------|----------|-------------------
1        | BERT-base | PyTorch   | 0.92     | 2024-10-18 10:00:00
2        | ResNet-50 | TensorFlow| 0.88     | 2024-10-18 11:00:00
3        | GPT-2     | PyTorch   | 0.95     | 2024-10-18 12:00:00
```

### 选择特定列

```sql
-- 只获取名称和准确率
SELECT name, accuracy FROM models;
```

**输出：**
```
name      | accuracy
----------|----------
BERT-base | 0.92
ResNet-50 | 0.88
GPT-2     | 0.95
```

### 列别名

```sql
-- 在输出中重命名列
SELECT
    name AS model_name,
    accuracy AS model_accuracy,
    framework AS ml_framework
FROM models;
```

**输出：**
```
model_name | model_accuracy | ml_framework
-----------|----------------|-------------
BERT-base  | 0.92           | PyTorch
ResNet-50  | 0.88           | TensorFlow
```

### SELECT 中的表达式

```sql
-- 计算准确率百分比
SELECT
    name,
    accuracy,
    accuracy * 100 AS accuracy_percent
FROM models;
```

**输出：**
```
name      | accuracy | accuracy_percent
----------|----------|------------------
BERT-base | 0.92     | 92.0
ResNet-50 | 0.88     | 88.0
```

### DISTINCT 值

```sql
-- 获取唯一的框架
SELECT DISTINCT framework FROM models;
```

**输出：**
```
framework
-----------
PyTorch
TensorFlow
```

## 数据过滤（WHERE）

### 基本 WHERE 子句

```sql
-- 获取准确率 > 0.90 的模型
SELECT * FROM models
WHERE accuracy > 0.90;
```

### 比较运算符

```sql
-- 等于
SELECT * FROM models WHERE framework = 'PyTorch';

-- 不等于
SELECT * FROM models WHERE framework != 'TensorFlow';
SELECT * FROM models WHERE framework <> 'TensorFlow';  -- 另一种写法

-- 大于
SELECT * FROM models WHERE accuracy > 0.90;

-- 大于或等于
SELECT * FROM models WHERE accuracy >= 0.90;

-- 小于
SELECT * FROM models WHERE accuracy < 0.90;

-- 小于或等于
SELECT * FROM models WHERE accuracy <= 0.90;
```

### 逻辑运算符

**AND：**
```sql
-- 既是 PyTorch 且准确率 > 0.90 的模型
SELECT * FROM models
WHERE framework = 'PyTorch' AND accuracy > 0.90;
```

**OR：**
```sql
-- 是 PyTorch 或 TensorFlow 的模型
SELECT * FROM models
WHERE framework = 'PyTorch' OR framework = 'TensorFlow';
```

**NOT：**
```sql
-- 不是 PyTorch 的模型
SELECT * FROM models
WHERE NOT framework = 'PyTorch';
```

**组合：**
```sql
-- 复杂条件
SELECT * FROM models
WHERE (framework = 'PyTorch' OR framework = 'TensorFlow')
  AND accuracy > 0.85
  AND status = 'deployed';
```

### IN 运算符

```sql
-- 在特定框架中的模型
SELECT * FROM models
WHERE framework IN ('PyTorch', 'TensorFlow', 'JAX');

-- 等价于：
SELECT * FROM models
WHERE framework = 'PyTorch'
   OR framework = 'TensorFlow'
   OR framework = 'JAX';
```

### BETWEEN 运算符

```sql
-- 准确率在 0.85 到 0.95 之间的模型
SELECT * FROM models
WHERE accuracy BETWEEN 0.85 AND 0.95;

-- 等价于：
SELECT * FROM models
WHERE accuracy >= 0.85 AND accuracy <= 0.95;
```

### LIKE 模式匹配

```sql
-- 名称以 'BERT' 开头的模型
SELECT * FROM models WHERE name LIKE 'BERT%';

-- 名称以 'Net' 结尾的模型
SELECT * FROM models WHERE name LIKE '%Net';

-- 名称包含 'GPT' 的模型
SELECT * FROM models WHERE name LIKE '%GPT%';

-- _ 匹配单个字符
SELECT * FROM models WHERE name LIKE 'GPT-_';  -- GPT-2, GPT-3
```

**通配符：**
- `%`：匹配任意字符序列
- `_`：匹配恰好一个字符

### NULL 处理

```sql
-- 查找没有准确率分数的模型
SELECT * FROM models WHERE accuracy IS NULL;

-- 查找有准确率分数的模型
SELECT * FROM models WHERE accuracy IS NOT NULL;

-- 注意：使用 IS NULL，而不是 = NULL
-- 这是错误的：
SELECT * FROM models WHERE accuracy = NULL;  -- 什么也不返回！
```

## 排序与限制结果

### ORDER BY

```sql
-- 按准确率排序（升序——最低在前）
SELECT * FROM models ORDER BY accuracy;

-- 按准确率排序（降序——最高在前）
SELECT * FROM models ORDER BY accuracy DESC;

-- 按多列排序
SELECT * FROM models
ORDER BY framework ASC, accuracy DESC;
-- 首先按框架字母排序，
-- 然后在每个框架内按准确率（高到低）排序
```

### LIMIT

```sql
-- 获取准确率最高的前 5 个模型
SELECT * FROM models
ORDER BY accuracy DESC
LIMIT 5;

-- 获取第 11-20 条记录（分页）
SELECT * FROM models
ORDER BY created_at DESC
LIMIT 10 OFFSET 10;
-- 跳过前 10 条，返回接下来的 10 条
```

### 组合 WHERE、ORDER BY、LIMIT

```sql
-- 按准确率排序的前 3 个 PyTorch 模型
SELECT name, framework, accuracy
FROM models
WHERE framework = 'PyTorch'
ORDER BY accuracy DESC
LIMIT 3;
```

## 插入数据

### 插入单行

```sql
INSERT INTO models (name, framework, accuracy, status)
VALUES ('BERT-large', 'PyTorch', 0.94, 'ready');
```

### 插入多行

```sql
INSERT INTO models (name, framework, accuracy, status)
VALUES
    ('ResNet-101', 'TensorFlow', 0.89, 'ready'),
    ('ViT-base', 'PyTorch', 0.91, 'ready'),
    ('EfficientNet-B0', 'TensorFlow', 0.87, 'training');
```

### 使用默认值插入

```sql
-- 使用 version、created_at、status 的默认值
INSERT INTO models (name, framework, accuracy)
VALUES ('MobileNet-v2', 'TensorFlow', 0.85);
-- version 将为 '1.0.0'，status 将为 'training'
-- created_at 将为当前时间戳
```

### 从 SELECT 插入

```sql
-- 从另一个表复制数据
INSERT INTO models_backup
SELECT * FROM models WHERE status = 'archived';
```

## 更新数据

### 更新单列

```sql
-- 更新模型状态
UPDATE models
SET status = 'deployed'
WHERE model_id = 1;
```

### 更新多列

```sql
-- 重新训练后更新模型
UPDATE models
SET
    accuracy = 0.96,
    version = '2.0.0',
    updated_at = CURRENT_TIMESTAMP
WHERE model_id = 1;
```

### 带条件的更新

```sql
-- 归档旧模型
UPDATE models
SET status = 'archived'
WHERE created_at < '2024-01-01' AND status != 'deployed';
```

### 更新所有行（危险！）

```sql
-- 这将更新所有行！
UPDATE models SET status = 'archived';

-- 除非确实要更新所有内容，否则始终使用 WHERE
```

## 删除数据

### 删除特定行

```sql
-- 删除特定模型
DELETE FROM models WHERE model_id = 5;
```

### 带条件的删除

```sql
-- 删除 2023 年已归档的模型
DELETE FROM models
WHERE status = 'archived'
  AND created_at < '2024-01-01';
```

### 删除所有行（非常危险！）

```sql
-- 这将删除所有行！
DELETE FROM models;

-- 更好：使用 WHERE 子句
DELETE FROM models WHERE 1=0;  -- 不删除任何内容（安全测试）
```

### TRUNCATE（更快的替代方案）

```sql
-- 快速删除所有行（重置自增）
TRUNCATE TABLE models;
-- 警告：某些数据库中无法回滚！
```

## ML 基础设施中的数据库

### 使用场景 1：模型注册

```sql
-- 跟踪所有已训练的模型
CREATE TABLE model_registry (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    framework VARCHAR(50),
    architecture VARCHAR(100),
    dataset VARCHAR(100),
    accuracy DECIMAL(5,4),
    training_date TIMESTAMP,
    model_path VARCHAR(255),
    status VARCHAR(20)
);

-- 查询：找到最佳模型
SELECT name, accuracy, framework
FROM model_registry
WHERE accuracy > 0.90 AND status = 'production'
ORDER BY accuracy DESC
LIMIT 10;
```

### 使用场景 2：实验跟踪

```sql
-- 记录所有训练实验
CREATE TABLE experiments (
    experiment_id INTEGER PRIMARY KEY,
    model_name VARCHAR(100),
    hyperparameters TEXT,  -- JSON 字符串
    accuracy DECIMAL(5,4),
    loss DECIMAL(10,6),
    training_time_seconds INTEGER,
    created_at TIMESTAMP
);

-- 查询：找到最佳超参数
SELECT hyperparameters, accuracy
FROM experiments
WHERE model_name = 'BERT-classifier'
ORDER BY accuracy DESC
LIMIT 1;
```

### 使用场景 3：预测日志记录

```sql
-- 记录所有预测以进行监控
CREATE TABLE prediction_logs (
    log_id INTEGER PRIMARY KEY,
    model_id INTEGER,
    input_hash VARCHAR(64),
    prediction TEXT,
    confidence DECIMAL(5,4),
    latency_ms INTEGER,
    timestamp TIMESTAMP,
    user_id VARCHAR(50)
);

-- 查询：监控模型性能
SELECT
    DATE(timestamp) as date,
    AVG(confidence) as avg_confidence,
    AVG(latency_ms) as avg_latency,
    COUNT(*) as prediction_count
FROM prediction_logs
WHERE model_id = 1
  AND timestamp >= DATE('now', '-7 days')
GROUP BY DATE(timestamp);
```

## 最佳实践

### 1. 始终在 UPDATE/DELETE 中使用 WHERE

```sql
-- 不好：意外更新所有行！
UPDATE models SET status = 'archived';

-- 好：更新特定行
UPDATE models SET status = 'archived'
WHERE model_id = 5;
```

### 2. 对多个操作使用事务

```sql
BEGIN TRANSACTION;

UPDATE models SET status = 'deployed' WHERE model_id = 1;
INSERT INTO deployment_log (model_id, deployed_at) VALUES (1, CURRENT_TIMESTAMP);

COMMIT;  -- 或者如果出错则 ROLLBACK
```

### 3. 为重要列建立索引

```sql
-- 创建索引以加快查询速度
CREATE INDEX idx_models_framework ON models(framework);
CREATE INDEX idx_models_accuracy ON models(accuracy);

-- 现在对这些列的查询快得多
SELECT * FROM models WHERE framework = 'PyTorch';
```

### 4. 使用有意义的名称

```sql
-- 不好
CREATE TABLE t1 (c1 INT, c2 VARCHAR(50));

-- 好
CREATE TABLE models (
    model_id INTEGER,
    model_name VARCHAR(50)
);
```

### 5. 使用约束验证数据

```sql
CREATE TABLE models (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    accuracy DECIMAL(5,4) CHECK (accuracy >= 0 AND accuracy <= 1),
    status VARCHAR(20) CHECK (status IN ('training', 'ready', 'deployed'))
);
```

### 6. 定期备份

```bash
# SQLite 备份
sqlite3 ml_database.db ".backup backup.db"

# PostgreSQL 备份
pg_dump ml_database > backup.sql
```

## 总结与关键要点

### 核心概念

1. **数据库**有效地组织和存储结构化数据
2. **SQL** 是与关系型数据库交互的标准语言
3. **表**由行（记录）和列（字段）组成
4. **主键**唯一标识行
5. **外键**在表之间建立关系

### 基本 SQL 命令

**数据检索：**
- `SELECT`：检索数据
- `WHERE`：过滤行
- `ORDER BY`：排序结果
- `LIMIT`：限制结果数量

**数据修改：**
- `INSERT`：添加新行
- `UPDATE`：修改现有行
- `DELETE`：删除行

**数据定义：**
- `CREATE TABLE`：定义新表
- `ALTER TABLE`：修改表结构
- `DROP TABLE`：删除表

### 最佳实践

1. 始终在 `UPDATE` 和 `DELETE` 中使用 `WHERE`
2. 对相关操作使用事务
3. 对频繁查询的列创建索引
4. 使用约束验证数据
5. 为表和列使用有意义的名称
6. 定期备份数据库

### ML 特定见解

数据库对于以下方面至关重要：
- 模型注册和版本控制
- 实验跟踪与比较
- 预测日志记录与监控
- 数据集元数据管理
- 性能指标存储

### 后续步骤

在下一讲中，我们将涵盖：
- 数据库设计原则
- 范式（1NF、2NF、3NF）
- 关系（一对多、多对多）
- 实体-关系图
- ML 系统的 Schema 设计

### 练习

1. 创建一个数据库来跟踪你的 ML 实验
2. 编写查询来查找表现最佳的模型
3. 为预测日志记录设计 schema
4. 练习使用复杂 WHERE 子句进行过滤

---

**预计学习时间：** 4-5 小时
**动手练习：** 完成练习 01：SQL 基础
**评估：** 测验 01 涵盖数据库基础和基本 SQL
