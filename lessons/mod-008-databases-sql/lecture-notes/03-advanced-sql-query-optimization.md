# Lecture 03: 高级 SQL 与查询优化

## 目录
1. [引言](#introduction)
2. [深入理解 JOIN](#joins-in-depth)
3. [子查询](#subqueries)
4. [公用表表达式（CTE）](#common-table-expressions-ctes)
5. [聚合与 GROUP BY](#aggregations-and-group-by)
6. [窗口函数](#window-functions)
7. [查询优化基础](#query-optimization-fundamentals)
8. [索引与性能](#indexes-and-performance)
9. [EXPLAIN 与查询计划](#explain-and-query-plans)
10. [ML 的高级 SQL 模式](#advanced-sql-patterns-for-ml)
11. [最佳实践](#best-practices)
12. [总结与关键要点](#summary-and-key-takeaways)

## 引言

高级 SQL 使你能够执行复杂的数据分析、组合来自多个来源的数据以及优化查询性能。对于 ML 基础设施，这些技能对于分析实验结果、计算指标和高效管理大规模数据管道至关重要。

### 学习目标

在本讲座结束时，你将：
- 掌握不同类型的 JOIN 及其使用时机
- 编写子查询和公用表表达式（CTE）
- 有效使用聚合函数和 GROUP BY
- 应用窗口函数进行高级分析
- 理解查询优化原则
- 策略性地创建和使用索引
- 阅读和解释查询执行计划
- 为 ML 工作负载优化 SQL 查询

### 前置条件
- 讲座 01：数据库基础与 SQL 入门
- 讲座 02：数据库设计与范式
- 熟练掌握基本 SQL（SELECT、WHERE、INSERT、UPDATE、DELETE）

### 预计时间
5-6 小时（包括动手优化练习）

## 深入理解 JOIN

### 示例 Schema

```sql
-- Models 表
models:
model_id | name      | framework
---------|-----------|----------
1        | BERT      | PyTorch
2        | ResNet    | TensorFlow
3        | GPT-2     | PyTorch

-- Experiments 表
experiments:
exp_id | model_id | accuracy | dataset
-------|----------|----------|--------
101    | 1        | 0.92     | IMDB
102    | 1        | 0.88     | SST-2
103    | 2        | 0.76     | ImageNet
104    | NULL     | 0.85     | CIFAR-10
```

### INNER JOIN

仅返回在两个表中都有匹配值的行。

```sql
SELECT
    m.name,
    e.accuracy,
    e.dataset
FROM models m
INNER JOIN experiments e ON m.model_id = e.model_id;
```

**结果：**
```
name   | accuracy | dataset
-------|----------|----------
BERT   | 0.92     | IMDB
BERT   | 0.88     | SST-2
ResNet | 0.76     | ImageNet
```

**注意：** GPT-2（无实验）和实验 104（无模型）被排除。

**何时使用：** 最常见的 JOIN 类型；仅当你想要匹配的记录时使用。

### LEFT（OUTER）JOIN

返回左表中的所有行，以及右表中匹配的行（若无匹配则为 NULL）。

```sql
SELECT
    m.name,
    e.accuracy,
    e.dataset
FROM models m
LEFT JOIN experiments e ON m.model_id = e.model_id;
```

**结果：**
```
name   | accuracy | dataset
-------|----------|----------
BERT   | 0.92     | IMDB
BERT   | 0.88     | SST-2
ResNet | 0.76     | ImageNet
GPT-2  | NULL     | NULL      ← 无实验，但仍被包含
```

**何时使用：** 查找没有实验的模型、没有订单的用户等。

**查找没有实验的模型：**
```sql
SELECT m.name
FROM models m
LEFT JOIN experiments e ON m.model_id = e.model_id
WHERE e.exp_id IS NULL;

-- 结果：GPT-2
```

### RIGHT（OUTER）JOIN

返回右表中的所有行，以及左表中匹配的行（若无匹配则为 NULL）。

```sql
SELECT
    m.name,
    e.accuracy,
    e.dataset
FROM models m
RIGHT JOIN experiments e ON m.model_id = e.model_id;
```

**结果：**
```
name   | accuracy | dataset
-------|----------|----------
BERT   | 0.92     | IMDB
BERT   | 0.88     | SST-2
ResNet | 0.76     | ImageNet
NULL   | 0.85     | CIFAR-10  ← 没有模型的实验
```

**注意：** RIGHT JOIN 不常见；通常可以改写为 LEFT JOIN。

### FULL OUTER JOIN

返回两个表中的所有行，无匹配处为 NULL。

```sql
SELECT
    m.name,
    e.accuracy,
    e.dataset
FROM models m
FULL OUTER JOIN experiments e ON m.model_id = e.model_id;
```

**结果：**
```
name   | accuracy | dataset
-------|----------|----------
BERT   | 0.92     | IMDB
BERT   | 0.88     | SST-2
ResNet | 0.76     | ImageNet
GPT-2  | NULL     | NULL      ← 无实验的模型
NULL   | 0.85     | CIFAR-10  ← 无模型的实验
```

**何时使用：** 查找两侧的孤立记录。

### CROSS JOIN

笛卡尔积：第一个表中的每一行与第二个表中的每一行配对。

```sql
SELECT m.name, f.framework_name
FROM models m
CROSS JOIN (VALUES ('PyTorch'), ('TensorFlow'), ('JAX')) AS f(framework_name);
```

**结果：** 3 个模型 × 3 个框架 = 9 行

**何时使用：** 生成组合、创建测试数据。

### 自连接

将表与自身连接。

```sql
-- 查找具有相似准确率的实验
SELECT
    e1.exp_id AS exp1,
    e2.exp_id AS exp2,
    e1.accuracy
FROM experiments e1
JOIN experiments e2 ON ABS(e1.accuracy - e2.accuracy) < 0.01
WHERE e1.exp_id < e2.exp_id;  -- 避免重复
```

### 多表 JOIN

```sql
-- 获取实验结果以及模型和用户信息
SELECT
    u.username,
    m.name AS model_name,
    e.accuracy,
    e.dataset
FROM experiments e
INNER JOIN models m ON e.model_id = m.model_id
INNER JOIN users u ON m.created_by = u.user_id
WHERE e.accuracy > 0.85
ORDER BY e.accuracy DESC;
```

## 子查询

嵌套在另一个查询中的查询。

### WHERE 中的子查询

```sql
-- 查找比平均水平更准确的模型
SELECT name, accuracy
FROM models
WHERE accuracy > (
    SELECT AVG(accuracy)
    FROM models
    WHERE accuracy IS NOT NULL
);
```

### 带 IN 的子查询

```sql
-- 查找在实验中使用的模型
SELECT name
FROM models
WHERE model_id IN (
    SELECT DISTINCT model_id
    FROM experiments
    WHERE model_id IS NOT NULL
);

-- 使用 JOIN 的替代方案（通常更快）：
SELECT DISTINCT m.name
FROM models m
INNER JOIN experiments e ON m.model_id = e.model_id;
```

### SELECT 中的子查询

```sql
-- 计算每个模型的实验数
SELECT
    name,
    (SELECT COUNT(*)
     FROM experiments e
     WHERE e.model_id = m.model_id) AS experiment_count
FROM models m;
```

### 相关子查询

子查询引用外部查询。

```sql
-- 查找每个模型的最佳准确率
SELECT
    m.name,
    (SELECT MAX(e.accuracy)
     FROM experiments e
     WHERE e.model_id = m.model_id) AS best_accuracy
FROM models m;
```

**注意：** 相关子查询可能较慢；考虑改用 JOIN 或窗口函数。

### EXISTS

检查子查询是否返回任何行。

```sql
-- 有实验的模型
SELECT name
FROM models m
WHERE EXISTS (
    SELECT 1
    FROM experiments e
    WHERE e.model_id = m.model_id
);

-- 没有实验的模型
SELECT name
FROM models m
WHERE NOT EXISTS (
    SELECT 1
    FROM experiments e
    WHERE e.model_id = m.model_id
);
```

## 公用表表达式（CTE）

CTE 通过将复杂查询分解为命名步骤来提高可读性。

### 基本 CTE

```sql
WITH high_accuracy_experiments AS (
    SELECT model_id, accuracy, dataset
    FROM experiments
    WHERE accuracy > 0.90
)
SELECT
    m.name,
    h.accuracy,
    h.dataset
FROM models m
INNER JOIN high_accuracy_experiments h ON m.model_id = h.model_id;
```

### 多个 CTE

```sql
WITH
pytorch_models AS (
    SELECT model_id, name
    FROM models
    WHERE framework = 'PyTorch'
),
recent_experiments AS (
    SELECT model_id, accuracy
    FROM experiments
    WHERE created_at > '2024-01-01'
)
SELECT
    p.name,
    AVG(r.accuracy) AS avg_accuracy
FROM pytorch_models p
INNER JOIN recent_experiments r ON p.model_id = r.model_id
GROUP BY p.name;
```

### 递归 CTE

适用于层次结构或图形数据。

```sql
-- 示例：模型谱系（从其他模型衍生的模型）
WITH RECURSIVE model_lineage AS (
    -- 基础情况：根模型
    SELECT model_id, name, parent_model_id, 1 AS level
    FROM models
    WHERE parent_model_id IS NULL

    UNION ALL

    -- 递归情况：子模型
    SELECT m.model_id, m.name, m.parent_model_id, ml.level + 1
    FROM models m
    INNER JOIN model_lineage ml ON m.parent_model_id = ml.model_id
)
SELECT * FROM model_lineage
ORDER BY level, name;
```

## 聚合与 GROUP BY

### 聚合函数

```sql
-- 统计所有实验
SELECT COUNT(*) FROM experiments;

-- 统计非 NULL 的准确率
SELECT COUNT(accuracy) FROM experiments;

-- 统计不同模型
SELECT COUNT(DISTINCT model_id) FROM experiments;

-- 平均准确率
SELECT AVG(accuracy) FROM experiments;

-- 最小和最大准确率
SELECT MIN(accuracy), MAX(accuracy) FROM experiments;

-- 所有准确率之和
SELECT SUM(accuracy) FROM experiments;
```

### GROUP BY

```sql
-- 每个模型的平均准确率
SELECT
    model_id,
    AVG(accuracy) AS avg_accuracy,
    COUNT(*) AS experiment_count
FROM experiments
GROUP BY model_id;
```

**结果：**
```
model_id | avg_accuracy | experiment_count
---------|--------------|------------------
1        | 0.90         | 2
2        | 0.76         | 1
NULL     | 0.85         | 1
```

### 带 JOIN 的 GROUP BY

```sql
-- 每个模型的平均准确率（含模型名称）
SELECT
    m.name,
    m.framework,
    AVG(e.accuracy) AS avg_accuracy,
    COUNT(e.exp_id) AS experiment_count
FROM models m
LEFT JOIN experiments e ON m.model_id = e.model_id
GROUP BY m.model_id, m.name, m.framework
ORDER BY avg_accuracy DESC NULLS LAST;
```

### HAVING

过滤组（类似于 WHERE，但用于聚合数据）。

```sql
-- 平均准确率 > 0.85 的模型
SELECT
    model_id,
    AVG(accuracy) AS avg_accuracy
FROM experiments
GROUP BY model_id
HAVING AVG(accuracy) > 0.85;
```

**注意：** WHERE 在分组之前过滤行；HAVING 在聚合之后过滤组。

```sql
-- 查找超过 1 个模型且平均准确率 > 0.88 的框架
SELECT
    framework,
    COUNT(*) AS model_count,
    AVG(accuracy) AS avg_accuracy
FROM models m
LEFT JOIN experiments e ON m.model_id = e.model_id
WHERE m.created_at > '2024-01-01'  -- 先过滤行
GROUP BY framework
HAVING COUNT(*) > 1                -- 然后过滤组
   AND AVG(accuracy) > 0.88;
```

### 多列 GROUP BY

```sql
-- 每个模型每个数据集的实验数
SELECT
    model_id,
    dataset,
    COUNT(*) AS experiment_count,
    AVG(accuracy) AS avg_accuracy
FROM experiments
GROUP BY model_id, dataset;
```

## 窗口函数

窗口函数在相关行之间执行计算而不折叠它们。

### ROW_NUMBER()

为每个分区内的行分配唯一编号。

```sql
-- 对每个模型的实验编号
SELECT
    model_id,
    exp_id,
    accuracy,
    ROW_NUMBER() OVER (PARTITION BY model_id ORDER BY accuracy DESC) AS rank_in_model
FROM experiments;
```

**结果：**
```
model_id | exp_id | accuracy | rank_in_model
---------|--------|----------|---------------
1        | 101    | 0.92     | 1
1        | 102    | 0.88     | 2
2        | 103    | 0.76     | 1
```

### RANK() 和 DENSE_RANK()

```sql
-- RANK：并列时排名有间隔
-- DENSE_RANK：无间隔
SELECT
    exp_id,
    accuracy,
    RANK() OVER (ORDER BY accuracy DESC) AS rank,
    DENSE_RANK() OVER (ORDER BY accuracy DESC) AS dense_rank
FROM experiments;
```

**结果：**
```
exp_id | accuracy | rank | dense_rank
-------|----------|------|------------
101    | 0.92     | 1    | 1
102    | 0.88     | 2    | 2
104    | 0.85     | 3    | 3
103    | 0.76     | 4    | 4
```

### NTILE()

将行划分为 N 个相等的桶。

```sql
-- 将实验按准确率分为四分位数
SELECT
    exp_id,
    accuracy,
    NTILE(4) OVER (ORDER BY accuracy) AS quartile
FROM experiments;
```

### LAG() 和 LEAD()

访问前一个/下一个行的值。

```sql
-- 比较每个实验与前一个实验
SELECT
    exp_id,
    accuracy,
    LAG(accuracy) OVER (ORDER BY exp_id) AS prev_accuracy,
    accuracy - LAG(accuracy) OVER (ORDER BY exp_id) AS improvement
FROM experiments;
```

### 聚合窗口函数

```sql
-- 准确率的运行平均值
SELECT
    exp_id,
    accuracy,
    AVG(accuracy) OVER (
        ORDER BY exp_id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_avg
FROM experiments;

-- 移动平均（最近 3 个实验）
SELECT
    exp_id,
    accuracy,
    AVG(accuracy) OVER (
        ORDER BY exp_id
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS moving_avg_3
FROM experiments;
```

### 实用 ML 示例

```sql
-- 查找每个模型的最佳实验，以及相较于之前的改进
WITH ranked_experiments AS (
    SELECT
        model_id,
        exp_id,
        accuracy,
        created_at,
        ROW_NUMBER() OVER (PARTITION BY model_id ORDER BY accuracy DESC) AS rank,
        LAG(accuracy) OVER (PARTITION BY model_id ORDER BY created_at) AS prev_accuracy
    FROM experiments
)
SELECT
    model_id,
    exp_id AS best_experiment,
    accuracy AS best_accuracy,
    accuracy - prev_accuracy AS improvement
FROM ranked_experiments
WHERE rank = 1;
```

## 查询优化基础

### 理解查询执行

SQL 查询按此顺序执行：

```sql
SELECT column_list          -- 5. 选择列
FROM table                  -- 1. 获取数据
JOIN other_table ON ...     -- 2. 连接表
WHERE conditions            -- 3. 过滤行
GROUP BY columns            -- 4. 分组行
HAVING group_conditions     -- 4a. 过滤组
ORDER BY columns            -- 6. 排序结果
LIMIT n;                    -- 7. 限制结果
```

### 优化原则

#### 1. 只选择需要的列

```sql
-- 不好：获取所有列（浪费）
SELECT * FROM experiments WHERE accuracy > 0.90;

-- 好：只获取所需内容
SELECT exp_id, model_id, accuracy FROM experiments WHERE accuracy > 0.90;
```

#### 2. 使用 WHERE 尽早过滤

```sql
-- 不好：连接所有数据然后过滤
SELECT m.name, e.accuracy
FROM models m
JOIN experiments e ON m.model_id = e.model_id
WHERE e.accuracy > 0.90;

-- 较好：如果可能，在连接前过滤
SELECT m.name, e.accuracy
FROM models m
JOIN (SELECT * FROM experiments WHERE accuracy > 0.90) e
  ON m.model_id = e.model_id;

-- 最佳：数据库优化器通常能处理，但显式指定有助于优化
```

#### 3. 对于大型子查询，使用 EXISTS 而非 IN

```sql
-- 可能较慢：使用 IN 的大型子查询
SELECT name FROM models
WHERE model_id IN (SELECT model_id FROM experiments);

-- 更快：EXISTS（在第一个匹配处停止）
SELECT name FROM models m
WHERE EXISTS (SELECT 1 FROM experiments e WHERE e.model_id = m.model_id);
```

#### 4. 尽可能避免 SELECT DISTINCT

```sql
-- 慢：DISTINCT 需要排序/去重
SELECT DISTINCT model_id FROM experiments;

-- 更快：如果需要聚合，使用 GROUP BY
SELECT model_id, COUNT(*) FROM experiments GROUP BY model_id;
```

#### 5. 尽早限制结果

```sql
-- 获取按准确率排序的前 10 个模型
SELECT name, accuracy
FROM models
WHERE accuracy IS NOT NULL
ORDER BY accuracy DESC
LIMIT 10;  -- 找到 10 个结果后停止
```

## 索引与性能

### 什么是索引？

索引是一种数据结构，可加快数据检索速度，但代价是写入速度变慢和存储空间增加。

**类比：** 书的索引让你无需阅读每一页就能快速找到主题。

### 索引类型

#### B-Tree 索引（默认）

最适合等值和范围查询。

```sql
CREATE INDEX idx_experiments_accuracy ON experiments(accuracy);

-- 现在这样查询很快：
SELECT * FROM experiments WHERE accuracy > 0.90;
SELECT * FROM experiments WHERE accuracy BETWEEN 0.85 AND 0.95;
```

#### Hash 索引

仅最适合等值查询。

```sql
CREATE INDEX idx_models_name_hash ON models USING HASH (name);

-- 快：
SELECT * FROM models WHERE name = 'BERT';

-- 不会使用索引（hash 不支持范围）：
SELECT * FROM models WHERE name LIKE 'BERT%';
```

#### 复合索引

基于多个列的索引。

```sql
CREATE INDEX idx_exp_model_accuracy ON experiments(model_id, accuracy);

-- 使用索引：
SELECT * FROM experiments WHERE model_id = 1 AND accuracy > 0.90;
SELECT * FROM experiments WHERE model_id = 1;  -- 使用第一列

-- 不会高效使用索引：
SELECT * FROM experiments WHERE accuracy > 0.90;  -- 跳过第一列
```

**顺序很重要：** 将选择性最高的列放在前面。

#### 唯一索引

强制唯一性并加快查找速度。

```sql
CREATE UNIQUE INDEX idx_models_name_unique ON models(name);

-- 防止重复，同时使查找快速
```

### 何时建立索引

**为这些建立索引：**
- 主键（自动索引）
- 外键
- WHERE 子句中的列
- JOIN 条件中的列
- ORDER BY 中的列
- GROUP BY 中的列

**不要过度索引：**
```sql
-- 不好：索引每一列
CREATE INDEX idx1 ON experiments(exp_id);        -- 已经是主键
CREATE INDEX idx2 ON experiments(model_id);      -- 可能有用
CREATE INDEX idx3 ON experiments(accuracy);      -- 可能有用
CREATE INDEX idx4 ON experiments(loss);          -- 可能不需要
CREATE INDEX idx5 ON experiments(created_at);    -- 取决于查询
CREATE INDEX idx6 ON experiments(batch_size);    -- 可能不需要
```

**每个索引：**
- ✅ 加快读取速度
- ❌ 减慢写入速度（INSERT、UPDATE、DELETE）
- ❌ 占用存储空间

### 监控索引使用情况

```sql
-- PostgreSQL：检查未使用的索引
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan AS index_scans,
    idx_tup_read AS tuples_read,
    idx_tup_fetch AS tuples_fetched
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%_pkey';  -- 排除主键

-- 删除未使用的索引
DROP INDEX idx_rarely_used;
```

## EXPLAIN 与查询计划

### 使用 EXPLAIN

```sql
-- 查看查询执行计划
EXPLAIN SELECT * FROM experiments WHERE accuracy > 0.90;

-- 包含实际执行统计信息
EXPLAIN ANALYZE SELECT * FROM experiments WHERE accuracy > 0.90;
```

### 阅读 EXPLAIN 输出

**示例：**
```
QUERY PLAN
--------------------------------------------------------------
Seq Scan on experiments  (cost=0.00..35.50 rows=5 width=40)
  Filter: (accuracy > 0.90)
```

**组成部分：**
- **Seq Scan**：顺序扫描（读取整个表）
- **cost=0.00..35.50**：预估成本（启动..总计）
- **rows=5**：预估返回行数
- **width=40**：每行预估字节数

### 使用索引后

```sql
CREATE INDEX idx_experiments_accuracy ON experiments(accuracy);

EXPLAIN SELECT * FROM experiments WHERE accuracy > 0.90;
```

**输出：**
```
Index Scan using idx_experiments_accuracy on experiments
  (cost=0.15..8.17 rows=5 width=40)
  Index Cond: (accuracy > 0.90)
```

**Index Scan** 取代 **Seq Scan** = 更快！

### 常见计划节点

- **Seq Scan**：全表扫描（对大表慢）
- **Index Scan**：使用索引（快）
- **Index Only Scan**：从索引获取数据，不访问表（最快）
- **Bitmap Index Scan**：扫描索引，然后获取行（中等）
- **Hash Join**：使用哈希表连接
- **Merge Join**：连接已排序的数据
- **Nested Loop**：对于第一个表中的每一行，扫描第二个表

### 基于 EXPLAIN 的优化

```sql
-- 慢查询
EXPLAIN ANALYZE
SELECT m.name, AVG(e.accuracy)
FROM models m
JOIN experiments e ON m.model_id = e.model_id
WHERE e.created_at > '2024-01-01'
GROUP BY m.name;

-- 如果看到 experiments 上的 Seq Scan，添加索引：
CREATE INDEX idx_experiments_created_at ON experiments(created_at);

-- 如果看到慢连接，确保外键已索引：
CREATE INDEX idx_experiments_model_id ON experiments(model_id);
```

## ML 的高级 SQL 模式

### 模式 1：数据透视表（按模型查看指标）

```sql
-- 将指标从行转换为列
SELECT
    model_id,
    MAX(CASE WHEN metric_name = 'accuracy' THEN metric_value END) AS accuracy,
    MAX(CASE WHEN metric_name = 'precision' THEN metric_value END) AS precision,
    MAX(CASE WHEN metric_name = 'recall' THEN metric_value END) AS recall,
    MAX(CASE WHEN metric_name = 'f1_score' THEN metric_value END) AS f1_score
FROM experiment_metrics
GROUP BY model_id;
```

### 模式 2：同期群分析

```sql
-- 按训练月份分析模型性能
SELECT
    DATE_TRUNC('month', created_at) AS training_month,
    framework,
    COUNT(*) AS models_trained,
    AVG(accuracy) AS avg_accuracy,
    MAX(accuracy) AS best_accuracy
FROM models
WHERE created_at >= '2024-01-01'
GROUP BY training_month, framework
ORDER BY training_month, framework;
```

### 模式 3：百分位分析

```sql
-- 查找推理延迟的第 95 百分位
SELECT
    model_id,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_latency
FROM predictions
GROUP BY model_id;
```

### 模式 4：时间序列聚合

```sql
-- 每日预测计数，附带 7 天移动平均
SELECT
    DATE(created_at) AS date,
    COUNT(*) AS predictions_count,
    AVG(COUNT(*)) OVER (
        ORDER BY DATE(created_at)
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM predictions
GROUP BY DATE(created_at)
ORDER BY date;
```

### 模式 5：查找异常值

```sql
-- 查找结果异常的实验（距离均值 > 2 个标准差）
WITH stats AS (
    SELECT
        AVG(accuracy) AS mean_accuracy,
        STDDEV(accuracy) AS stddev_accuracy
    FROM experiments
)
SELECT e.exp_id, e.accuracy, e.model_id
FROM experiments e, stats s
WHERE ABS(e.accuracy - s.mean_accuracy) > 2 * s.stddev_accuracy;
```

## 最佳实践

### 1. 编写可读的查询

```sql
-- 好：清晰、格式良好
SELECT
    m.name,
    m.framework,
    COUNT(e.exp_id) AS experiment_count,
    AVG(e.accuracy) AS avg_accuracy
FROM models m
LEFT JOIN experiments e ON m.model_id = e.model_id
WHERE m.created_at > '2024-01-01'
GROUP BY m.model_id, m.name, m.framework
HAVING COUNT(e.exp_id) > 0
ORDER BY avg_accuracy DESC
LIMIT 10;

-- 不好：不可读
select m.name,m.framework,count(e.exp_id),avg(e.accuracy) from models m left join experiments e on m.model_id=e.model_id where m.created_at>'2024-01-01' group by m.model_id,m.name,m.framework having count(e.exp_id)>0 order by avg_accuracy desc limit 10;
```

### 2. 复杂查询使用 CTE

```sql
-- 使用 CTE 而非嵌套子查询
WITH recent_models AS (
    SELECT * FROM models WHERE created_at > '2024-01-01'
),
high_accuracy_experiments AS (
    SELECT * FROM experiments WHERE accuracy > 0.90
)
SELECT
    m.name,
    COUNT(e.exp_id) AS successful_experiments
FROM recent_models m
JOIN high_accuracy_experiments e ON m.model_id = e.model_id
GROUP BY m.name;
```

### 3. 在样本数据上测试查询

```sql
-- 先在小子集上测试
SELECT * FROM large_table WHERE ... LIMIT 100;

-- 然后移除 LIMIT 进行完整查询
```

### 4. 监控查询性能

```sql
-- PostgreSQL：查找慢查询
SELECT
    query,
    calls,
    total_time,
    mean_time,
    max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;
```

### 5. 定期维护

```sql
-- PostgreSQL：更新统计信息以获得更好的查询计划
ANALYZE experiments;

-- 如果索引碎片化则重建
REINDEX TABLE experiments;

-- 清理已删除的行
VACUUM experiments;
```

## 总结与关键要点

### 核心 SQL 技术

1. **JOIN**：组合来自多个表的数据（INNER、LEFT、RIGHT、FULL OUTER）
2. **子查询**：用于复杂过滤的嵌套查询
3. **CTE**：用于可读性的命名查询块
4. **GROUP BY**：按类别聚合数据
5. **窗口函数**：不折叠行的高级分析

### 优化原则

1. **选择特定列**，而非 `*`
2. **尽早过滤**，使用 WHERE
3. **策略性地使用索引**（外键、WHERE 列、JOIN 列）
4. **避免过度索引**（每个索引都有成本）
5. **使用 EXPLAIN** 理解查询计划
6. **监控性能**并优化慢查询

### 高级模式

- **数据透视**：将行转换为列
- **百分位数**：统计分析
- **移动平均**：时间序列分析
- **异常值检测**：发现异常
- **同期群分析**：按时间段分组

### ML 特定应用

- 跨模型比较实验结果
- 跟踪模型随时间的性能
- 分析预测模式
- 计算聚合指标
- 监控推理延迟
- 查找最佳超参数

### 后续步骤

在下一讲中，我们将涵盖：
- 对象关系映射（ORM）
- 用于 Python 的 SQLAlchemy
- 数据库迁移
- 连接池
- 事务管理
- ML 应用的集成模式

---

**预计学习时间：** 5-6 小时
**动手练习：** 完成练习 03：高级 SQL 查询
**评估：** 测验涵盖 JOIN、聚合、窗口函数和优化
