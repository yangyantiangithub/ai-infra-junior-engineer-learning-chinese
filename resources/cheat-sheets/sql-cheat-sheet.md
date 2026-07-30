# SQL 速查表 — PostgreSQL 侧重

SQL 语法和 PostgreSQL 特有功能的快速参考。假设你已完成模块 008。

## 读取数据

```sql
-- 基础
SELECT * FROM users;
SELECT id, email, created_at FROM users WHERE created_at > NOW() - INTERVAL '7 days';

-- DISTINCT
SELECT DISTINCT country FROM users;

-- 排序 + 分页
SELECT id, email FROM users
ORDER BY created_at DESC
LIMIT 50 OFFSET 100;

-- IN / BETWEEN / LIKE / ILIKE / IS NULL
SELECT * FROM events WHERE event_type IN ('signup', 'purchase');
SELECT * FROM events WHERE amount BETWEEN 10 AND 100;
SELECT * FROM users WHERE email LIKE '%@example.com';
SELECT * FROM users WHERE name ILIKE 'jo%';        -- 不区分大小写
SELECT * FROM users WHERE deleted_at IS NULL;
```

## 聚合

```sql
SELECT
    country,
    COUNT(*)            AS user_count,
    COUNT(DISTINCT email) AS unique_emails,
    AVG(age)::numeric(10,2) AS avg_age,
    MIN(created_at)     AS earliest,
    MAX(created_at)     AS latest
FROM users
GROUP BY country
HAVING COUNT(*) > 100
ORDER BY user_count DESC;
```

## 连接

```sql
-- INNER
SELECT u.id, u.email, o.total
FROM users u
JOIN orders o ON o.user_id = u.id;

-- LEFT（包含没有订单的用户）
SELECT u.id, u.email, COALESCE(SUM(o.total), 0) AS lifetime_value
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.email;

-- 自连接
SELECT a.id, b.id AS referrer_id
FROM users a
JOIN users b ON a.referrer_id = b.id;

-- 反连接（没有订单的用户）
SELECT u.*
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;
-- 或更清晰的写法：
SELECT *
FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders WHERE user_id = u.id);
```

## CTE（WITH）

```sql
WITH recent_orders AS (
    SELECT user_id, SUM(total) AS spend
    FROM orders
    WHERE created_at > NOW() - INTERVAL '30 days'
    GROUP BY user_id
),
high_value AS (
    SELECT user_id FROM recent_orders WHERE spend > 1000
)
SELECT u.id, u.email, ro.spend
FROM users u
JOIN high_value hv ON hv.user_id = u.id
JOIN recent_orders ro ON ro.user_id = u.id;
```

## 窗口函数

```sql
-- 每个用户的累计总额
SELECT
    user_id, created_at, total,
    SUM(total) OVER (PARTITION BY user_id ORDER BY created_at) AS running_total
FROM orders;

-- 分区内排名
SELECT
    user_id, total,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total DESC) AS rn,
    RANK()       OVER (PARTITION BY user_id ORDER BY total DESC) AS rk,
    DENSE_RANK() OVER (PARTITION BY user_id ORDER BY total DESC) AS dr
FROM orders;

-- Lag/Lead
SELECT
    created_at, total,
    LAG(total)  OVER (ORDER BY created_at) AS prev_total,
    LEAD(total) OVER (ORDER BY created_at) AS next_total
FROM orders;
```

## 写入数据

```sql
-- INSERT
INSERT INTO users (email, name) VALUES ('a@b.com', 'A');

-- 多行
INSERT INTO users (email, name) VALUES
    ('a@b.com', 'A'),
    ('c@d.com', 'C');

-- INSERT … ON CONFLICT（upsert）
INSERT INTO users (id, email)
VALUES (1, 'new@example.com')
ON CONFLICT (id)
DO UPDATE SET email = EXCLUDED.email, updated_at = NOW();

-- UPDATE
UPDATE users SET name = 'X' WHERE id = 1;
UPDATE users u
SET total_orders = subq.cnt
FROM (SELECT user_id, COUNT(*) AS cnt FROM orders GROUP BY user_id) subq
WHERE u.id = subq.user_id;

-- DELETE
DELETE FROM users WHERE deleted_at < NOW() - INTERVAL '90 days';

-- RETURNING — 用于获取插入的 ID
INSERT INTO orders (user_id, total) VALUES (1, 50)
RETURNING id, created_at;
```

## DDL

```sql
CREATE TABLE users (
    id           BIGSERIAL PRIMARY KEY,
    email        TEXT NOT NULL UNIQUE,
    name         TEXT,
    age          INT CHECK (age >= 0),
    metadata     JSONB DEFAULT '{}'::jsonb,
    created_at   TIMESTAMPTZ DEFAULT NOW(),
    updated_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_email_lower ON users (LOWER(email));
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);
CREATE INDEX idx_users_metadata ON users USING GIN (metadata);

ALTER TABLE users ADD COLUMN tier TEXT NOT NULL DEFAULT 'free';
ALTER TABLE users DROP COLUMN unused_col;
ALTER TABLE users RENAME COLUMN old_name TO new_name;

DROP TABLE IF EXISTS old_users;
```

## 索引 — 经验法则

| 模式 | 索引 |
|---|---|
| 列等值查询 | B-tree（默认） |
| 范围或排序 | B-tree（列顺序：等值 → 范围 → 排序） |
| 全文搜索 | GIN + `tsvector` |
| JSONB 包含 | GIN |
| 地理数据 | GiST 或 BRIN |
| 顺序时间序列 | BRIN |

对频繁配对的列使用单个复合索引，而不是每列一个索引。始终运行 `EXPLAIN ANALYZE` 来确认规划器使用了你的索引。

## PostgreSQL JSONB

```sql
-- 获取字段
SELECT metadata->'name' FROM users;          -- jsonb
SELECT metadata->>'name' FROM users;         -- text

-- 路径
SELECT metadata#>'{address,city}' FROM users;

-- 包含
SELECT * FROM users WHERE metadata @> '{"tier": "gold"}';

-- 键存在
SELECT * FROM users WHERE metadata ? 'phone';

-- 更新特定字段
UPDATE users SET metadata = jsonb_set(metadata, '{tier}', '"gold"');
```

## 事务

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- 或 ROLLBACK;

-- 保存点
BEGIN;
INSERT INTO ...;
SAVEPOINT sp1;
INSERT INTO ...;
ROLLBACK TO SAVEPOINT sp1;
COMMIT;
```

隔离级别：`READ COMMITTED`（默认）、`REPEATABLE READ`、`SERIALIZABLE`。当正确性很重要且可以在冲突时重试时，使用 SERIALIZABLE。

## EXPLAIN

```sql
EXPLAIN SELECT ...;                  -- 执行计划
EXPLAIN ANALYZE SELECT ...;          -- 计划 + 实际执行时间
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
```

检查以下情况：
- 大表上的 `Seq Scan` → 缺少索引。
- 估算行数与实际行数差异大 → 统计信息过时，运行 `ANALYZE`。
- 排序操作溢出到磁盘 → `work_mem` 对该查询来说太小。

## 性能模式

```sql
-- 基于 keyset 的分页（在大表上比 OFFSET 快得多）
SELECT * FROM events
WHERE created_at < $1                -- 上次看到的时间戳
ORDER BY created_at DESC
LIMIT 50;

-- 不扫描表的估算计数
SELECT reltuples::bigint AS approx_count
FROM pg_class
WHERE relname = 'events';

-- 通过 DISTINCT ON 实现每组 Top-N
SELECT DISTINCT ON (user_id) user_id, created_at, total
FROM orders
ORDER BY user_id, created_at DESC;
```

## 有用的 psql 命令

```bash
psql "postgres://user:pass@host:5432/db"

\l               # 列出数据库
\dt              # 列出表
\d users         # 描述表
\di              # 列出索引
\du              # 列出角色/用户
\x               # 切换扩展显示
\timing          # 显示查询时间
\copy users TO 'users.csv' CSV HEADER
\copy events FROM 'events.csv' CSV HEADER
\watch 5         # 每 5 秒重新运行上次查询
```

## 常见陷阱

- **`NULL` 不等于任何东西。** `NULL = NULL` 是 `NULL`，不是 `TRUE`。使用 `IS NULL`。
- **`COUNT(*)` 与 `COUNT(col)`。** `COUNT(col)` 忽略空值。
- **PostgreSQL 中字符串比较默认区分大小写。** 使用 `LOWER()` 或 `ILIKE`。
- **时区。** 优先使用 `TIMESTAMPTZ` 而非 `TIMESTAMP`。内部以 UTC 存储。
- **隐式类型转换**可能导致索引失效。`WHERE int_column = '5'` 可能使用也可能不使用索引；显式转换。
- **OR 通常比 UNION ALL 慢。** 有时将 `WHERE a OR b` 拆分为两个通过 `UNION ALL` 连接的查询可以让规划器对每个部分使用索引。

## 另请参阅

- `EXPLAIN` 文档：https://www.postgresql.org/docs/current/using-explain.html
- pgAdmin、DBeaver、DataGrip 等 GUI 客户端
- pg_stat_statements 扩展用于生产环境查询分析
