# Lecture 02: 数据库设计与范式

## 目录
1. [引言](#introduction)
2. [数据库设计原则](#database-design-principles)
3. [实体-关系（ER）建模](#entity-relationship-er-modeling)
4. [关系类型](#types-of-relationships)
5. [从 ER 图到表](#from-er-diagram-to-tables)
6. [数据范式化](#data-normalization)
7. [第一范式（1NF）](#first-normal-form-1nf)
8. [第二范式（2NF）](#second-normal-form-2nf)
9. [第三范式（3NF）](#third-normal-form-3nf)
10. [巴科斯-科德范式（BCNF）](#boyce-codd-normal-form-bcnf)
11. [反范式化](#denormalization)
12. [设计 ML 数据库 Schema](#designing-ml-database-schemas)
13. [Schema 迁移策略](#schema-migration-strategies)
14. [最佳实践](#best-practices)
15. [总结与关键要点](#summary-and-key-takeaways)

## 引言

良好的数据库设计对于构建可扩展、可维护且高效的应用程序至关重要。糟糕的设计会导致数据冗余、不一致和性能问题。本讲座将教你设计健壮数据库 schema 的原理和技术，重点介绍范式化和 ML 使用场景。

### 学习目标

在本讲座结束时，你将：
- 理解数据库设计原则
- 创建实体-关系图
- 识别并实现不同的关系类型
- 应用范式化形式（1NF 到 BCNF）
- 知道何时为了性能而反范式化
- 为 ML 系统设计数据库 schema
- 规划并执行 schema 迁移

### 前置条件
- 讲座 01：数据库基础与 SQL 入门
- 理解关系型数据库概念
- 基本 SQL 知识（CREATE、SELECT 等）

### 预计时间
4-5 小时（包括设计练习）

## 数据库设计原则

### 1. 数据完整性

确保数据准确且一致：

**完整性差的示例：**
```
Users 表：
user_id | email             | email_verified
--------|-------------------|---------------
1       | alice@example.com | true
1       | bob@example.com   | false  ← 相同的 user_id，不同的 email！
```

**良好的完整性：**
```sql
CREATE TABLE users (
    user_id INTEGER PRIMARY KEY,           -- 唯一
    email VARCHAR(255) NOT NULL UNIQUE,    -- 必须存在，必须唯一
    email_verified BOOLEAN DEFAULT FALSE
);
```

### 2. 最小化冗余

不要在同一数据存储在多个地方：

**不好（冗余）：**
```
experiments 表：
exp_id | model_name | model_framework | model_version | accuracy
-------|------------|-----------------|---------------|----------
1      | BERT-base  | PyTorch         | 1.0.0         | 0.92
2      | BERT-base  | PyTorch         | 1.0.0         | 0.93
3      | BERT-base  | PyTorch         | 1.0.0         | 0.94
        ↑ 每次实验中重复的模型信息
```

**好（范式化）：**
```
models 表：
model_id | name      | framework | version
---------|-----------|-----------|--------
1        | BERT-base | PyTorch   | 1.0.0

experiments 表：
exp_id | model_id | accuracy
-------|----------|----------
1      | 1        | 0.92
2      | 1        | 0.93
3      | 1        | 0.94
        ↑ 引用模型，无重复
```

### 3. 逻辑组织

将相关数据分组在一起：

```sql
-- 好：相关数据放在一个表中
CREATE TABLE user_profiles (
    user_id INTEGER PRIMARY KEY,
    username VARCHAR(50),
    email VARCHAR(255),
    full_name VARCHAR(100),
    created_at TIMESTAMP
);

-- 避免：不必要地拆分
CREATE TABLE usernames (user_id INT, username VARCHAR(50));
CREATE TABLE emails (user_id INT, email VARCHAR(255));
CREATE TABLE names (user_id INT, full_name VARCHAR(100));
```

### 4. 可扩展性

为增长而设计：

```sql
-- 使用适当的数据类型
CREATE TABLE predictions (
    prediction_id BIGINT PRIMARY KEY,  -- 处理数十亿条记录
    model_id INTEGER,
    confidence DECIMAL(5,4),           -- 精确数字
    created_at TIMESTAMP
);

-- 为性能添加索引
CREATE INDEX idx_predictions_model_id ON predictions(model_id);
CREATE INDEX idx_predictions_created_at ON predictions(created_at);
```

### 5. 灵活性

为未来变更而设计：

```sql
-- 灵活：使用单独的表存储标签
CREATE TABLE model_tags (
    tag_id INTEGER PRIMARY KEY,
    model_id INTEGER,
    tag_name VARCHAR(50),
    FOREIGN KEY (model_id) REFERENCES models(model_id)
);
-- 无需更改 schema 即可轻松添加/删除标签

-- 不灵活：存储为逗号分隔的字符串
CREATE TABLE models (
    model_id INTEGER PRIMARY KEY,
    tags VARCHAR(255)  -- 'nlp,transformer,bert'
    -- 难以查询、维护和修改
);
```

## 实体-关系（ER）建模

### 什么是 ER 图？

**实体-关系图**直观表示数据库的结构：
- **实体**：我们要存储的事物（表）
- **属性**：实体的属性（列）
- **关系**：实体的连接方式

### ER 图组成部分

**实体（矩形）：**
```
┌─────────────┐
│   Models    │  ← 实体
└─────────────┘
```

**属性（椭圆）：**
```
     ┌──────────┐
     │model_name│  ← 属性
     └────┬─────┘
          │
     ┌────┴────┐
     │ Models  │
     └─────────┘
```

**主键（下划线）：**
```
model_id（下划线）= 主键
```

**关系（菱形）：**
```
┌─────────┐       ┌──────────┐       ┌────────────┐
│ Models  │───────│  trains  │───────│Experiments │
└─────────┘       └──────────┘       └────────────┘
```

### 示例：ML 模型注册 ER 图

```
                  1                     N
┌─────────┐              ┌──────────────┐
│ Models  │──────────────│ Experiments  │
└─────────┘   has many   └──────────────┘
│                         │
│ model_id (PK)           │ experiment_id (PK)
│ name                    │ model_id (FK)
│ framework               │ accuracy
│ version                 │ created_at
│ created_at              │
│                         │
│         1               │        N
│         │               │        │
│         └───────────────┴────────┘
│                 runs
│
│              1
│              │
│              │
│         ┌────┴──────┐
│         │           │
│    ┌────┴────┐  ┌──┴──────────┐
│    │ Users   │  │ Predictions │
│    └─────────┘  └─────────────┘
│    │            │
│    │user_id(PK) │ prediction_id (PK)
│    │username    │ model_id (FK)
│    │email       │ input_data
│                 │ prediction
│                 │ created_at
└─────────────────┘
      created by
```

### 基数表示法

**一对一（1:1）**
```
User ────── Profile
  1            1
```

**一对多（1:N）**
```
Model ────── Experiments
  1              N
```

**多对多（M:N）**
```
Models ────── Tags
  M              N
```

## 关系类型

### 1. 一对一（1:1）

一个实体恰好与另一个实体相关联。

**示例：用户与用户档案**

```sql
CREATE TABLE users (
    user_id INTEGER PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE user_profiles (
    profile_id INTEGER PRIMARY KEY,
    user_id INTEGER UNIQUE NOT NULL,  -- UNIQUE 使其成为 1:1
    bio TEXT,
    avatar_url VARCHAR(255),
    preferences JSON,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```

**何时使用：**
- 分离敏感数据（如账单信息）
- 可选的扩展信息
- 大型二进制数据（照片、文档）

### 2. 一对多（1:N）

一个实体与多个其他实体相关联。

**示例：模型有多个实验**

```sql
CREATE TABLE models (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE experiments (
    experiment_id INTEGER PRIMARY KEY,
    model_id INTEGER NOT NULL,  -- 多个实验 → 一个模型
    accuracy DECIMAL(5,4),
    created_at TIMESTAMP,
    FOREIGN KEY (model_id) REFERENCES models(model_id)
);
```

**可视化：**
```
Model (id=1, name='BERT')
    ├─ Experiment (id=101, model_id=1, accuracy=0.92)
    ├─ Experiment (id=102, model_id=1, accuracy=0.93)
    └─ Experiment (id=103, model_id=1, accuracy=0.94)
```

**何时使用：**
- 父子关系
- 最常见的关系类型
- 示例：用户→订单、模型→预测、数据集→样本

### 3. 多对多（M:N）

多个实体与多个其他实体相关联。

**示例：模型与标签**

**问题：** 无法在关系型数据库中直接表示 M:N。

**解决方案：** 使用**连接表**（也称为：桥接表、链接表、关联表）

```sql
CREATE TABLE models (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE tags (
    tag_id INTEGER PRIMARY KEY,
    tag_name VARCHAR(50) UNIQUE
);

-- 连接表
CREATE TABLE model_tags (
    model_id INTEGER,
    tag_id INTEGER,
    PRIMARY KEY (model_id, tag_id),  -- 组合键
    FOREIGN KEY (model_id) REFERENCES models(model_id),
    FOREIGN KEY (tag_id) REFERENCES tags(tag_id)
);
```

**可视化：**
```
Models:                      Tags:
1: BERT                      A: NLP
2: ResNet                    B: Vision
3: GPT-2                     C: Transformer

model_tags（连接表）：
model_id | tag_id
---------|--------
1        | A       ← BERT 有标签 NLP
1        | C       ← BERT 有标签 Transformer
2        | B       ← ResNet 有标签 Vision
3        | A       ← GPT-2 有标签 NLP
3        | C       ← GPT-2 有标签 Transformer
```

**查询示例：**
```sql
-- 获取 BERT 的所有标签
SELECT t.tag_name
FROM tags t
JOIN model_tags mt ON t.tag_id = mt.tag_id
JOIN models m ON mt.model_id = m.model_id
WHERE m.name = 'BERT';

-- 结果：NLP, Transformer
```

**何时使用：**
- 学生↔课程
- 产品↔类别
- 模型↔标签
- 用户↔角色

## 从 ER 图到表

### 步骤 1：每个实体 → 表

```
实体：Model
属性：model_id, name, framework

→

CREATE TABLE models (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    framework VARCHAR(50)
);
```

### 步骤 2：一对多 → 外键

```
Model (1) → (N) Experiments

→

CREATE TABLE experiments (
    experiment_id INTEGER PRIMARY KEY,
    model_id INTEGER,  -- 外键
    accuracy DECIMAL(5,4),
    FOREIGN KEY (model_id) REFERENCES models(model_id)
);
```

### 步骤 3：多对多 → 连接表

```
Models (M) ↔ (N) Tags

→

CREATE TABLE model_tags (
    model_id INTEGER,
    tag_id INTEGER,
    PRIMARY KEY (model_id, tag_id),
    FOREIGN KEY (model_id) REFERENCES models(model_id),
    FOREIGN KEY (tag_id) REFERENCES tags(tag_id)
);
```

### 完整示例：ML 平台 Schema

```sql
-- Users
CREATE TABLE users (
    user_id INTEGER PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Models（1:N 与 Experiments，M:N 与 Tags）
CREATE TABLE models (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    framework VARCHAR(50),
    created_by INTEGER,  -- FK 到 users
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (created_by) REFERENCES users(user_id)
);

-- Experiments（N:1 与 Models）
CREATE TABLE experiments (
    experiment_id INTEGER PRIMARY KEY,
    model_id INTEGER NOT NULL,
    accuracy DECIMAL(5,4),
    loss DECIMAL(10,6),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (model_id) REFERENCES models(model_id)
);

-- Tags
CREATE TABLE tags (
    tag_id INTEGER PRIMARY KEY,
    tag_name VARCHAR(50) UNIQUE NOT NULL
);

-- Model-Tags 连接表（M:N）
CREATE TABLE model_tags (
    model_id INTEGER,
    tag_id INTEGER,
    PRIMARY KEY (model_id, tag_id),
    FOREIGN KEY (model_id) REFERENCES models(model_id),
    FOREIGN KEY (tag_id) REFERENCES tags(tag_id)
);
```

## 数据范式化

### 什么是范式化？

**范式化**是组织数据的过程，旨在：
1. 消除冗余
2. 确保数据依赖关系合理
3. 减少 INSERT、UPDATE、DELETE 期间的数据异常

### 为什么要范式化？

**非范式化数据的问题：**

```
experiments_bad 表：
┌────┬───────────┬──────────┬───────────┬──────────┬─────────────┐
│ id │model_name │ framework│ dataset   │ accuracy │ model_author│
├────┼───────────┼──────────┼───────────┼──────────┼─────────────┤
│ 1  │BERT-base  │ PyTorch  │ IMDB      │ 0.92     │ Alice       │
│ 2  │BERT-base  │ PyTorch  │ SST-2     │ 0.88     │ Alice       │
│ 3  │ResNet-50  │TensorFlow│ ImageNet  │ 0.76     │ Bob         │
└────┴───────────┴──────────┴───────────┴──────────┴─────────────┘
```

**问题：**

1. **更新异常**：更改 BERT 的 framework 需要更新多行
2. **插入异常**：无法添加没有实验的模型
3. **删除异常**：删除所有 ResNet 实验会丢失模型信息
4. **空间浪费**：重复数据（BERT-base 重复出现）

### 范式化形式概述

- **1NF**：原子值，无重复组
- **2NF**：1NF + 无部分依赖
- **3NF**：2NF + 无传递依赖
- **BCNF**：3NF 的更严格版本

## 第一范式（1NF）

### 规则

1. 每列包含原子（不可分割）值
2. 每列包含相同类型的值
3. 每列有唯一名称
4. 顺序无关紧要

### 违反示例

**违反 1：非原子值**

```sql
-- ❌ 违反 1NF（tags 是一个列表）
CREATE TABLE models_bad (
    model_id INTEGER,
    name VARCHAR(100),
    tags VARCHAR(255)  -- 'nlp,transformer,bert'
);
```

**修复：**
```sql
-- ✓ 符合 1NF
CREATE TABLE models (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE model_tags (
    model_id INTEGER,
    tag VARCHAR(50),
    PRIMARY KEY (model_id, tag)
);
```

**违反 2：重复组**

```sql
-- ❌ 违反 1NF（重复列）
CREATE TABLE experiments_bad (
    model_id INTEGER,
    metric1_name VARCHAR(50),
    metric1_value DECIMAL(5,4),
    metric2_name VARCHAR(50),
    metric2_value DECIMAL(5,4),
    metric3_name VARCHAR(50),
    metric3_value DECIMAL(5,4)
);
```

**修复：**
```sql
-- ✓ 符合 1NF
CREATE TABLE experiment_metrics (
    experiment_id INTEGER,
    metric_name VARCHAR(50),
    metric_value DECIMAL(5,4),
    PRIMARY KEY (experiment_id, metric_name)
);
```

### 1NF 示例

**1NF 之前：**
```
┌────────┬───────────────┬─────────────────────┐
│model_id│ name          │ frameworks          │
├────────┼───────────────┼─────────────────────┤
│ 1      │ BERT          │ PyTorch,TensorFlow  │  ← 非原子
│ 2      │ ResNet        │ TensorFlow          │
└────────┴───────────────┴─────────────────────┘
```

**1NF 之后：**
```
models：
┌────────┬───────────┐
│model_id│ name      │
├────────┼───────────┤
│ 1      │ BERT      │
│ 2      │ ResNet    │
└────────┴───────────┘

model_frameworks：
┌────────┬────────────┐
│model_id│ framework  │
├────────┼────────────┤
│ 1      │ PyTorch    │
│ 1      │TensorFlow  │
│ 2      │TensorFlow  │
└────────┴────────────┘
```

## 第二范式（2NF）

### 规则

1. 必须满足 1NF
2. 无**部分依赖**于组合键

### 什么是部分依赖？

当一个非键属性仅依赖于组合主键的一部分。

### 违反示例

```sql
-- 组合键：(experiment_id, dataset_id)
CREATE TABLE experiment_datasets_bad (
    experiment_id INTEGER,
    dataset_id INTEGER,
    dataset_name VARCHAR(100),  -- ❌ 仅依赖于 dataset_id
    dataset_size INTEGER,       -- ❌ 仅依赖于 dataset_id
    accuracy DECIMAL(5,4),      -- ✓ 依赖于两者
    PRIMARY KEY (experiment_id, dataset_id)
);
```

**问题：** `dataset_name` 和 `dataset_size` 仅依赖于 `dataset_id`，而不依赖于完整键。

### 修复：分离表

```sql
-- ✓ 符合 2NF

-- Datasets（数据集属性单独存放）
CREATE TABLE datasets (
    dataset_id INTEGER PRIMARY KEY,
    dataset_name VARCHAR(100),
    dataset_size INTEGER
);

-- Experiment-Dataset 关系
CREATE TABLE experiment_datasets (
    experiment_id INTEGER,
    dataset_id INTEGER,
    accuracy DECIMAL(5,4),  -- 依赖于两个键
    PRIMARY KEY (experiment_id, dataset_id),
    FOREIGN KEY (dataset_id) REFERENCES datasets(dataset_id)
);
```

### 2NF 示例

**2NF 之前：**
```
experiment_results（组合键：experiment_id, metric_name）：
┌──────────────┬─────────────┬──────────────┬─────────┐
│experiment_id │ metric_name │ metric_type  │ value   │
├──────────────┼─────────────┼──────────────┼─────────┤
│ 1            │ accuracy    │ performance  │ 0.92    │
│ 1            │ f1_score    │ performance  │ 0.88    │
│ 2            │ accuracy    │ performance  │ 0.90    │
└──────────────┴─────────────┴──────────────┴─────────┘
                ↑ metric_type 仅依赖于 metric_name（部分依赖）
```

**2NF 之后：**
```
metrics：
┌─────────────┬──────────────┐
│ metric_name │ metric_type  │
├─────────────┼──────────────┤
│ accuracy    │ performance  │
│ f1_score    │ performance  │
│ loss        │ training     │
└─────────────┴──────────────┘

experiment_results：
┌──────────────┬─────────────┬─────────┐
│experiment_id │ metric_name │ value   │
├──────────────┼─────────────┼─────────┤
│ 1            │ accuracy    │ 0.92    │
│ 1            │ f1_score    │ 0.88    │
│ 2            │ accuracy    │ 0.90    │
└──────────────┴─────────────┴─────────┘
```

## 第三范式（3NF）

### 规则

1. 必须满足 2NF
2. 无**传递依赖**

### 什么是传递依赖？

当一个非键属性依赖于另一个非键属性。

```
A → B → C  （A 决定 B，B 决定 C）
因此：A → C 是传递依赖
```

### 违反示例

```sql
-- ❌ 违反 3NF
CREATE TABLE models_bad (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    framework VARCHAR(50),
    framework_creator VARCHAR(100),  -- ❌ 依赖于 framework，而非 model_id
    framework_year INTEGER           -- ❌ 依赖于 framework，而非 model_id
);

-- 传递依赖：
-- model_id → framework → framework_creator
```

**问题：** 如果更改 PyTorch 的创建者，必须更新所有 PyTorch 模型。

### 修复：分离框架信息

```sql
-- ✓ 符合 3NF

CREATE TABLE frameworks (
    framework_name VARCHAR(50) PRIMARY KEY,
    creator VARCHAR(100),
    release_year INTEGER
);

CREATE TABLE models (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    framework VARCHAR(50),
    FOREIGN KEY (framework) REFERENCES frameworks(framework_name)
);
```

### 3NF 示例

**3NF 之前：**
```
experiments：
┌────────────┬──────────┬────────────┬─────────────────┬─────────────┐
│experiment_id│ model_id │ model_name │ model_framework │ accuracy    │
├────────────┼──────────┼────────────┼─────────────────┼─────────────┤
│ 1          │ 10       │ BERT-base  │ PyTorch         │ 0.92        │
│ 2          │ 10       │ BERT-base  │ PyTorch         │ 0.93        │
└────────────┴──────────┴────────────┴─────────────────┴─────────────┘
             ↑ experiment_id → model_id → model_name（传递依赖）
```

**3NF 之后：**
```
models：
┌──────────┬────────────┬─────────────────┐
│ model_id │ model_name │ model_framework │
├──────────┼────────────┼─────────────────┤
│ 10       │ BERT-base  │ PyTorch         │
└──────────┴────────────┴─────────────────┘

experiments：
┌────────────┬──────────┬──────────┐
│experiment_id│ model_id │ accuracy │
├────────────┼──────────┼──────────┤
│ 1          │ 10       │ 0.92     │
│ 2          │ 10       │ 0.93     │
└────────────┴──────────┴──────────┘
```

## 巴科斯-科德范式（BCNF）

### 规则

1. 必须满足 3NF
2. 对于每个函数依赖 X → Y，X 必须是超键

**简化：** 每个决定因素必须是候选键。

### 示例（罕见边缘情况）

```sql
-- 学生-课程-教师
-- 规则：
-- - 每个学生跟随一位教师修读一门课程
-- - 每位教师只教一门课程
-- - 多位教师可以教同一门课程

CREATE TABLE enrollment (
    student_id INTEGER,
    course VARCHAR(50),
    instructor VARCHAR(50),
    PRIMARY KEY (student_id, course),
    -- 函数依赖：instructor → course
    -- 但 instructor 不是超键
    -- 违反 BCNF
);
```

**修复：**
```sql
-- 拆分为两个表
CREATE TABLE instructors (
    instructor VARCHAR(50) PRIMARY KEY,
    course VARCHAR(50)
);

CREATE TABLE enrollment (
    student_id INTEGER,
    instructor VARCHAR(50),
    PRIMARY KEY (student_id, instructor),
    FOREIGN KEY (instructor) REFERENCES instructors(instructor)
);
```

**注意：** BCNF 在实践中很少需要。3NF 通常就足够了。

## 反范式化

### 什么是反范式化？

有意引入冗余以提高读取性能。

### 何时反范式化？

1. **读密集型工作负载**：读取多于写入
2. **性能关键**：响应时间比存储更重要
3. **复杂连接**：查询连接多个表
4. **报表/分析**：预计算聚合

### 示例：预测日志记录

**范式化（3NF）：**
```sql
-- 写入快，读取慢
CREATE TABLE predictions (
    prediction_id BIGINT PRIMARY KEY,
    model_id INTEGER,
    user_id INTEGER,
    prediction TEXT,
    created_at TIMESTAMP,
    FOREIGN KEY (model_id) REFERENCES models(model_id),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

-- 查询需要连接
SELECT p.prediction, m.name, u.username
FROM predictions p
JOIN models m ON p.model_id = m.model_id
JOIN users u ON p.user_id = u.user_id
WHERE p.created_at > '2024-10-01';
```

**反范式化（用于读取）：**
```sql
-- 重复 model_name 和 username 以加快查询
CREATE TABLE predictions_denorm (
    prediction_id BIGINT PRIMARY KEY,
    model_id INTEGER,
    model_name VARCHAR(100),     -- ← 冗余（来自 models 表）
    user_id INTEGER,
    username VARCHAR(50),        -- ← 冗余（来自 users 表）
    prediction TEXT,
    created_at TIMESTAMP
);

-- 无需连接！
SELECT prediction, model_name, username
FROM predictions_denorm
WHERE created_at > '2024-10-01';
```

**权衡：**
- ✅ **读取更快**（无需连接）
- ✅ **查询更简单**
- ❌ **存储更多**（重复数据）
- ❌ **更新复杂性**（必须更新多个位置）
- ❌ **数据不一致风险**

### 物化视图（更好的替代方案）

```sql
-- PostgreSQL 物化视图
CREATE MATERIALIZED VIEW prediction_summary AS
SELECT
    p.prediction_id,
    p.prediction,
    m.name AS model_name,
    u.username,
    p.created_at
FROM predictions p
JOIN models m ON p.model_id = m.model_id
JOIN users u ON p.user_id = u.user_id;

-- 定期刷新
REFRESH MATERIALIZED VIEW prediction_summary;

-- 查询快速
SELECT * FROM prediction_summary
WHERE created_at > '2024-10-01';
```

**优势：**
- 原始表保持范式化
- 视图为性能而反范式化
- 自动更新（可定时）

## 设计 ML 数据库 Schema

### 使用场景 1：ML 模型注册

```sql
-- 核心表
CREATE TABLE models (
    model_id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    framework VARCHAR(50),
    architecture VARCHAR(100),
    created_by INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(20) CHECK (status IN ('training', 'ready', 'deployed', 'archived')),
    FOREIGN KEY (created_by) REFERENCES users(user_id)
);

CREATE TABLE model_versions (
    version_id SERIAL PRIMARY KEY,
    model_id INTEGER NOT NULL,
    version_number VARCHAR(20) NOT NULL,
    model_path VARCHAR(255),
    accuracy DECIMAL(5,4),
    precision_score DECIMAL(5,4),
    recall_score DECIMAL(5,4),
    f1_score DECIMAL(5,4),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (model_id) REFERENCES models(model_id),
    UNIQUE (model_id, version_number)
);

CREATE TABLE model_metrics (
    metric_id SERIAL PRIMARY KEY,
    version_id INTEGER NOT NULL,
    metric_name VARCHAR(50),
    metric_value DECIMAL(10,6),
    FOREIGN KEY (version_id) REFERENCES model_versions(version_id)
);
```

### 使用场景 2：实验跟踪

```sql
CREATE TABLE experiments (
    experiment_id SERIAL PRIMARY KEY,
    experiment_name VARCHAR(100),
    model_id INTEGER,
    dataset_id INTEGER,
    created_by INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    status VARCHAR(20),
    FOREIGN KEY (model_id) REFERENCES models(model_id),
    FOREIGN KEY (created_by) REFERENCES users(user_id)
);

CREATE TABLE experiment_hyperparameters (
    experiment_id INTEGER,
    param_name VARCHAR(50),
    param_value TEXT,
    PRIMARY KEY (experiment_id, param_name),
    FOREIGN KEY (experiment_id) REFERENCES experiments(experiment_id)
);

CREATE TABLE experiment_results (
    result_id SERIAL PRIMARY KEY,
    experiment_id INTEGER,
    epoch INTEGER,
    metric_name VARCHAR(50),
    metric_value DECIMAL(10,6),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (experiment_id) REFERENCES experiments(experiment_id)
);
```

### 使用场景 3：数据集管理

```sql
CREATE TABLE datasets (
    dataset_id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    dataset_type VARCHAR(50),  -- 'training', 'validation', 'test'
    size_bytes BIGINT,
    num_samples INTEGER,
    storage_path VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE dataset_splits (
    split_id SERIAL PRIMARY KEY,
    dataset_id INTEGER,
    split_name VARCHAR(20),  -- 'train', 'val', 'test'
    split_percentage DECIMAL(5,2),
    num_samples INTEGER,
    FOREIGN KEY (dataset_id) REFERENCES datasets(dataset_id)
);

CREATE TABLE dataset_features (
    feature_id SERIAL PRIMARY KEY,
    dataset_id INTEGER,
    feature_name VARCHAR(100),
    data_type VARCHAR(50),
    is_nullable BOOLEAN,
    FOREIGN KEY (dataset_id) REFERENCES datasets(dataset_id)
);
```

## Schema 迁移策略

### 1. 版本控制你的 Schema

```sql
-- migrations/001_create_models_table.sql
CREATE TABLE models (
    model_id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

-- migrations/002_add_framework_column.sql
ALTER TABLE models
ADD COLUMN framework VARCHAR(50);

-- migrations/003_add_accuracy_column.sql
ALTER TABLE models
ADD COLUMN accuracy DECIMAL(5,4);
```

### 2. 使用迁移工具

**Alembic（Python）：**
```python
# alembic/versions/001_create_models.py
def upgrade():
    op.create_table(
        'models',
        sa.Column('model_id', sa.Integer, primary_key=True),
        sa.Column('name', sa.String(100), nullable=False)
    )

def downgrade():
    op.drop_table('models')
```

**Flyway（Java）：**
```sql
-- V1__create_models_table.sql
CREATE TABLE models (
    model_id INTEGER PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);
```

### 3. 安全迁移实践

```sql
-- ❌ 危险：无法回滚
ALTER TABLE models DROP COLUMN old_column;

-- ✓ 更安全：多步迁移
-- 步骤 1：添加新列
ALTER TABLE models ADD COLUMN new_column VARCHAR(100);

-- 步骤 2：迁移数据
UPDATE models SET new_column = old_column;

-- 步骤 3：（稍后）删除旧列
-- ALTER TABLE models DROP COLUMN old_column;
```

### 4. 零停机迁移

```sql
-- 添加带默认值的列（大多数数据库即时完成）
ALTER TABLE models
ADD COLUMN status VARCHAR(20) DEFAULT 'active';

-- 并发创建索引（PostgreSQL）
CREATE INDEX CONCURRENTLY idx_models_status ON models(status);

-- 添加带有 NOT VALID 的外键（PostgreSQL）
ALTER TABLE experiments
ADD CONSTRAINT fk_model
FOREIGN KEY (model_id) REFERENCES models(model_id)
NOT VALID;

-- 稍后验证（可在低流量时进行）
ALTER TABLE experiments
VALIDATE CONSTRAINT fk_model;
```

## 最佳实践

### 1. 清晰命名

```sql
-- 好
CREATE TABLE models (...);
CREATE TABLE model_versions (...);
CREATE TABLE experiment_results (...);

-- 避免
CREATE TABLE tbl1 (...);
CREATE TABLE data (...);
CREATE TABLE temp_stuff (...);
```

### 2. 使用适当的数据类型

```sql
-- 好
model_id BIGINT           -- 可处理数十亿条记录
accuracy DECIMAL(5,4)     -- 精确：0.9234
created_at TIMESTAMP      -- 日期和时间
is_active BOOLEAN         -- 真/假

-- 避免
model_id VARCHAR(50)      -- 浪费空间，更慢
accuracy VARCHAR(10)      -- 无法进行数学运算
created_at VARCHAR(50)    -- 无法进行日期运算
is_active VARCHAR(5)      -- 'true' vs TRUE vs 1 混淆
```

### 3. 记录你的 Schema

```sql
-- 添加注释
COMMENT ON TABLE models IS '所有 ML 模型的注册表';
COMMENT ON COLUMN models.accuracy IS '模型在测试集上的准确率（0-1）';

-- 或者在迁移文件中
-- 迁移：添加模型版本控制
-- 目的：跟踪同一模型的多个版本
-- 作者：Alice
-- 日期：2024-10-18
```

### 4. 规划审计功能

```sql
CREATE TABLE models (
    model_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    created_by INTEGER,           -- 谁创建的？
    created_at TIMESTAMP,         -- 何时创建？
    updated_by INTEGER,           -- 最后更新者？
    updated_at TIMESTAMP,         -- 最后更新时间？
    deleted_at TIMESTAMP          -- 软删除（NULL = 未删除）
);
```

### 5. 为策略性列建立索引

```sql
-- 为外键建立索引
CREATE INDEX idx_experiments_model_id ON experiments(model_id);

-- 为频繁查询的列建立索引
CREATE INDEX idx_models_framework ON models(framework);
CREATE INDEX idx_predictions_created_at ON predictions(created_at);

-- 为常见查询模式的复合索引
CREATE INDEX idx_models_status_accuracy ON models(status, accuracy);
```

## 总结与关键要点

### 核心概念

1. **ER 图**直观表示数据库结构
2. **关系**连接实体（1:1、1:N、M:N）
3. **范式化**减少冗余和异常
4. **反范式化**用冗余换取性能

### 范式化形式

| 形式 | 规则 | 目的 |
|------|------|---------|
| 1NF  | 原子值 | 无重复组 |
| 2NF  | 无部分依赖 | 依赖于完整键 |
| 3NF  | 无传递依赖 | 非键不依赖于非键 |
| BCNF | 决定因素必须是键 | 更严格的 3NF |

### 设计原则

1. **从范式化开始**（3NF）
2. **仅在需要时反范式化**（有证据支持）
3. **使用外键**维护引用完整性
4. **策略性地使用索引**以提高查询性能
5. **版本控制**你的 schema
6. **仔细规划**迁移

### ML 特定模式

- **模型注册**：模型 + 版本 + 指标
- **实验跟踪**：实验 + 超参数 + 结果
- **数据集管理**：数据集 + 拆分 + 特征
- **预测日志记录**：预测 + 模型 + 用户

### 后续步骤

在下一讲中，我们将涵盖：
- 高级 SQL 查询（JOIN、子查询、CTE）
- 聚合与 GROUP BY
- 窗口函数
- 查询优化技术
- 索引策略
- 查询执行计划

---

**预计学习时间：** 4-5 小时
**动手练习：** 完成练习 02：数据库设计
**评估：** 测验问题涵盖 ER 图和范式
