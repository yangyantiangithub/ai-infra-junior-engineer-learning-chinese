# Exercise 05: 数据库优化与索引

## 概述

通过分析查询、引入索引和调整配置来优化 ML 注册数据库的性能。你还将建立基准监控仪表盘，以便及早发现性能退化。本练习将你从数据库用户转变为性能工程师，能够诊断瓶颈并为生产 ML 基础设施实施针对性优化。

**难度：** 中级 → 高级
**预计时间：** 3-4 小时
**前置条件：**
- 已完成练习 01–04（schema、高级 SQL、ORM 集成）
- 讲座 04（ORM 与数据库集成）
- 能解释 `EXPLAIN ANALYZE` 输出
- 基本了解数据库架构（页面、缓冲区、WAL）

## 学习目标

完成本练习后，你将能够：

1. **使用 PostgreSQL 诊断工具分析数据库性能**（pg_stat_statements、EXPLAIN ANALYZE）
2. **通过执行计划分析和指标识别查询瓶颈**
3. **针对不同查询模式设计索引策略**（B-tree、Hash、BRIN、GIN、partial indexes）
4. **通过前后对比衡量性能改进**
5. **实施维护程序**（VACUUM、ANALYZE、REINDEX）
6. **配置生产数据库健康监控**
7. **在应用代码和 ORM 使用中优化查询模式**
8. **应用数据库性能管理的生产最佳实践**

## 场景

你的 ML 模型注册已经平稳运行了 3 个月，但最近的增长导致了性能问题：

- **仪表盘查询**加载需要 5-10 秒（之前 <500ms）
- **训练运行导入**偶尔遇到超时
- **部署状态检查**在高峰时段导致 API 变慢
- **数据库 CPU 使用率**在营业时间飙升至 80-90%

作为当班的基础设施工程师，你必须：
1. 分析工作负载以识别慢查询
2. 应用针对性优化（索引、查询重写）
3. 建立监控以防止未来退化
4. 记录可重复的性能调优流程

管理层期望：
- ✅ P99 查询延迟降低 80%
- ✅ 仪表盘加载时间 < 1 秒
- ✅ 高峰时段零查询超时
- ✅ 自动监控和告警

---

## 第 1 部分：数据库性能分析

### 步骤 1.1：启用查询统计

PostgreSQL 提供 `pg_stat_statements` 用于跟踪查询性能。启用它：

```sql
-- 连接到你的数据库
psql -h localhost -U ml_user -d ml_registry

-- 启用扩展
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 验证是否工作
SELECT * FROM pg_stat_statements LIMIT 5;
```

**预期输出：**
```
 userid | dbid  | queryid  |         query          | calls | total_time | mean_time
--------+-------+----------+------------------------+-------+------------+-----------
  16384 | 16385 | 12345678 | SELECT * FROM models   |   142 |    1250.34 |      8.80
```

### 步骤 1.2：配置 PostgreSQL 性能跟踪

编辑 `postgresql.conf`（或通过 Docker 环境变量设置）：

```conf
# 跟踪语句执行统计
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000

# 启用慢查询的详细查询日志
log_min_duration_statement = 1000  # 记录 > 1 秒的查询
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_statement = 'ddl'

# 为 > 500ms 的查询启用自动分析
session_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 500
auto_explain.log_analyze = true
auto_explain.log_buffers = true
auto_explain.log_timing = true
auto_explain.log_format = 'json'

# 提高统计信息准确性
default_statistics_target = 100  # 默认为 100，复杂查询可增加
```

**重启 PostgreSQL** 以应用更改：

```bash
docker restart ml-registry-postgres
```

### 步骤 1.3：生成真实工作负载

创建一个工作负载模拟脚本以生成真实查询模式：

创建 `scripts/generate_workload.py`：

```python
"""
Generate realistic workload for ML model registry.
Simulates dashboard queries, training run ingestion, and deployment checks.
"""
import time
import random
from datetime import datetime, timedelta
from uuid import uuid4

import psycopg2
from psycopg2.extras import execute_values

# Database connection
conn = psycopg2.connect(
    host="localhost", port=5432, dbname="ml_registry",
    user="ml_user", password="ml_password"
)

def simulate_dashboard_query():
    """Simulate slow dashboard query - latest deployments with model info."""
    with conn.cursor() as cur:
        cur.execute("""
            SELECT m.model_name, mv.semver, d.environment_id, d.status, d.deployed_at, d.deployed_by
            FROM deployments d
            INNER JOIN model_versions mv ON d.version_id = mv.version_id
            INNER JOIN models m ON mv.model_id = m.model_id
            WHERE d.deployed_at > NOW() - INTERVAL '30 days'
            ORDER BY d.deployed_at DESC
            LIMIT 50;
        """)
        return len(cur.fetchall())

def simulate_training_runs_query():
    """Simulate training runs query by experiment."""
    with conn.cursor() as cur:
        experiment_name = random.choice(["fraud-exp-001", "churn-exp-002", "recommender-exp-003"])
        cur.execute("""
            SELECT run_id, model_name, status, accuracy, created_at
            FROM training_runs
            WHERE experiment_name = %s
              AND created_at > NOW() - INTERVAL '7 days'
            ORDER BY accuracy DESC NULLS LAST
            LIMIT 20;
        """, (experiment_name,))
        return len(cur.fetchall())

def simulate_model_stats_query():
    """Simulate aggregate stats query across all models."""
    with conn.cursor() as cur:
        cur.execute("""
            SELECT m.model_name, COUNT(DISTINCT mv.version_id) as version_count,
                   COUNT(DISTINCT tr.run_id) as training_runs,
                   COUNT(DISTINCT d.deployment_id) as deployments,
                   MAX(tr.accuracy) as best_accuracy
            FROM models m
            LEFT JOIN model_versions mv ON m.model_id = mv.model_id
            LEFT JOIN training_runs tr ON tr.version_id = mv.version_id
            LEFT JOIN deployments d ON d.version_id = mv.version_id
            WHERE m.is_active = true
            GROUP BY m.model_id, m.model_name
            ORDER BY deployments DESC;
        """)
        return len(cur.fetchall())

def simulate_deployment_status_check():
    """Simulate deployment status check for specific environment."""
    with conn.cursor() as cur:
        cur.execute("""
            SELECT m.model_name, mv.semver, d.status, d.replicas, d.deployed_at
            FROM deployments d
            INNER JOIN model_versions mv ON d.version_id = mv.version_id
            INNER JOIN models m ON mv.model_id = m.model_id
            WHERE d.environment_id = 1 AND d.status = 'active'
            ORDER BY d.deployed_at DESC;
        """)
        return len(cur.fetchall())

def run_workload(duration_seconds=60, queries_per_second=10):
    query_functions = [simulate_dashboard_query, simulate_training_runs_query, simulate_model_stats_query, simulate_deployment_status_check]
    start_time = time.time()
    query_count = 0
    print(f"Running workload for {duration_seconds}s at {queries_per_second} QPS...")
    while time.time() - start_time < duration_seconds:
        query_func = random.choice(query_functions)
        try:
            start = time.time()
            rows = query_func()
            duration = (time.time() - start) * 1000
            query_count += 1
            if duration > 100:
                print(f"  ⚠️  {query_func.__name__}: {duration:.1f}ms ({rows} rows)")
        except Exception as e:
            print(f"  ❌ Error in {query_func.__name__}: {e}")
        time.sleep(1.0 / queries_per_second)
    elapsed = time.time() - start_time
    print(f"\nWorkload complete:\n  Total queries: {query_count}\n  Duration: {elapsed:.1f}s\n  Actual QPS: {query_count / elapsed:.1f}")

if __name__ == "__main__":
    import sys
    duration = int(sys.argv[1]) if len(sys.argv) > 1 else 60
    qps = int(sys.argv[2]) if len(sys.argv) > 2 else 10
    try:
        run_workload(duration, qps)
    finally:
        conn.close()
```

**运行工作负载：**

```bash
python scripts/generate_workload.py 60 10
```

### 步骤 1.4：捕获基准性能指标

查询 `pg_stat_statements` 以识别慢查询：

```sql
-- 按总时间排名前 10 的慢查询
SELECT query, calls, total_exec_time / 1000 as total_time_sec,
       mean_exec_time as avg_time_ms, max_exec_time as max_time_ms,
       stddev_exec_time as stddev_ms, rows,
       100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS cache_hit_ratio
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat_statements%'
ORDER BY total_exec_time DESC
LIMIT 10;
```

**示例输出：**
```
 query                           | calls | total_time_sec | avg_time_ms | max_time_ms | cache_hit_ratio
---------------------------------+-------+----------------+-------------+-------------+-----------------
 SELECT m.model_name, mv.semver… |   245 |          35.67 |      145.58 |     1250.34 |           45.23
 SELECT run_id, model_name…      |   892 |          28.45 |       31.90 |      456.78 |           78.92
 SELECT m.model_name, COUNT…     |   156 |          22.34 |      143.21 |      890.12 |           38.45
```

**关键跟踪指标：**

1. **total_time_sec**：该查询总耗时（高 = 优先目标）
2. **avg_time_ms**：每次调用的平均执行时间
3. **max_time_ms**：最差情况延迟（P100）
4. **cache_hit_ratio**：在内存中找到数据的百分比 vs 磁盘（低 = I/O 瓶颈）
5. **calls**：查询运行频率（高 + 慢 = 关键）

### 步骤 1.5：使用 EXPLAIN 分析执行计划

对每个慢查询，捕获详细执行计划：

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, TIMING)
SELECT m.model_name, mv.semver, d.environment_id, d.status, d.deployed_at
FROM deployments d
INNER JOIN model_versions mv ON d.version_id = mv.version_id
INNER JOIN models m ON mv.model_id = m.model_id
WHERE d.deployed_at > NOW() - INTERVAL '30 days'
ORDER BY d.deployed_at DESC
LIMIT 50;
```

**示例执行计划（优化前）：**

```
Limit  (cost=1245.67..1246.05 rows=50 width=120) (actual time=145.234..145.567 rows=50 loops=1)
  Buffers: shared hit=2456 read=3421
  ->  Sort  (cost=1245.67..1256.78 rows=4450 width=120) (actual time=145.231..145.298 rows=50 loops=1)
        Sort Key: d.deployed_at DESC
        Sort Method: top-N heapsort  Memory: 35kB
        Buffers: shared hit=2456 read=3421
        ->  Hash Join  (cost=234.56..1134.89 rows=4450 width=120) (actual time=12.345..139.876 rows=4450 loops=1)
              Hash Cond: (mv.model_id = m.model_id)
              Buffers: shared hit=2456 read=3421
              ->  Hash Join  (cost=123.45..987.65 rows=4450 width=88) (actual time=5.678..125.432 rows=4450 loops=1)
                    Hash Cond: (d.version_id = mv.version_id)
                    Buffers: shared hit=1234 read=3421
                    ->  Seq Scan on deployments d  (cost=0.00..845.67 rows=4450 width=56) (actual time=0.023..112.345 rows=4450 loops=1)
                          Filter: (deployed_at > (now() - '30 days'::interval))
                          Rows Removed by Filter: 15678
                          Buffers: shared hit=567 read=3421
                    ->  Hash  (cost=98.76..98.76 rows=1975 width=32) (actual time=5.432..5.433 rows=1975 loops=1)
                          Buckets: 2048  Batches: 1  Memory Usage: 125kB
                          Buffers: shared hit=667
              ->  Hash  (cost=87.65..87.65 rows=1876 width=40) (actual time=6.543..6.544 rows=1876 loops=1)
                    Buckets: 2048  Batches: 1  Memory Usage: 145kB
                    Buffers: shared hit=555
                    ->  Seq Scan on models m  (cost=0.00..87.65 rows=1876 width=40) (actual time=0.015..4.234 rows=1876 loops=1)
                          Buffers: shared hit=555
Planning Time: 1.234 ms
Execution Time: 145.678 ms
```

**执行计划关键发现：**

1. **Seq Scan on deployments**（112ms）：全表扫描过滤 15,678 行 → **需要在 deployed_at 上建索引**
2. **缓存命中率低**（2,456 hit / 5,877 total = 42%）：数据不在内存中 → **I/O 瓶颈**
3. **Hash joins**：高效，但输入数据来自慢的顺序扫描
4. **未使用索引**：查询未受益于任何现有索引

### 步骤 1.6：记录基准性能

创建 `docs/performance-baseline.md`：

```markdown
# 性能基准报告

**日期**: 2025-10-23
**数据库**: ml_registry (PostgreSQL 14.x)
**工作负载**: 60 秒 @ 10 QPS 混合查询

---

## 执行摘要

- **最慢查询**: 仪表盘部署视图（平均 145ms，最大 1,250ms）
- **主要瓶颈**: `deployments` 表上的顺序扫描（`deployed_at` 无索引）
- **缓存命中率**: 45%（目标：>90%）
- **分析的查询总数**: 4

---

## 查询 1: 最新部署仪表盘

**查询**:
```sql
SELECT m.model_name, mv.semver, d.status, d.deployed_at
FROM deployments d
INNER JOIN model_versions mv ON d.version_id = mv.version_id
INNER JOIN models m ON mv.model_id = m.model_id
WHERE d.deployed_at > NOW() - INTERVAL '30 days'
ORDER BY d.deployed_at DESC
LIMIT 50;
```

**指标**:
- 调用次数: 245
- 总时间: 35.67 秒
- 平均时间: 145.58 ms
- 最大时间: 1,250.34 ms
- 扫描行数: 20,128 (deployments 表)
- 返回行数: 50
- 缓存命中率: 45.23%

**执行计划摘要**:
- `deployments` 顺序扫描 (112ms)
- 与 `model_versions` 和 `models` 的 Hash join (33ms)
- 排序和 limit (0.5ms)

**瓶颈**: 按 `deployed_at` 过滤的 Seq Scan

**建议**: 在 `deployments(deployed_at DESC)` 上添加 B-tree 索引

---

## 查询 2: 按实验的培训运行

**查询**:
```sql
SELECT run_id, model_name, status, accuracy, created_at
FROM training_runs
WHERE experiment_name = $1
  AND created_at > NOW() - INTERVAL '7 days'
ORDER BY accuracy DESC NULLS LAST
LIMIT 20;
```

**指标**:
- 调用次数: 892
- 总时间: 28.45 秒
- 平均时间: 31.90 ms
- 最大时间: 456.78 ms
- 扫描行数: 45,670
- 返回行数: 20
- 缓存命中率: 78.92%

**瓶颈**: 按 `experiment_name` 和 `created_at` 过滤的 Seq Scan

**建议**: `training_runs(experiment_name, created_at DESC)` 上的复合索引

---

## 查询 3: 模型统计聚合

**查询**:
```sql
SELECT m.model_name, COUNT(DISTINCT mv.version_id) as version_count,
       COUNT(DISTINCT tr.run_id) as training_runs, MAX(tr.accuracy) as best_accuracy
FROM models m
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
LEFT JOIN training_runs tr ON tr.version_id = mv.version_id
WHERE m.is_active = true
GROUP BY m.model_id, m.model_name
ORDER BY version_count DESC;
```

**指标**:
- 调用次数: 156
- 总时间: 22.34 秒
- 平均时间: 143.21 ms
- 最大时间: 890.12 ms
- 扫描行数: 67,890
- 返回行数: 42
- 缓存命中率: 38.45%

**瓶颈**: 多个 JOIN 加上 `training_runs` 的顺序扫描

**建议**: `training_runs(version_id)` 上的索引用于 JOIN 优化

---

## 查询 4: 按环境的活跃部署

**查询**:
```sql
SELECT m.model_name, mv.semver, d.status, d.deployed_at
FROM deployments d
INNER JOIN model_versions mv ON d.version_id = mv.version_id
INNER JOIN models m ON mv.model_id = m.model_id
WHERE d.environment_id = 1 AND d.status = 'active'
ORDER BY d.deployed_at DESC;
```

**指标**:
- 调用次数: 421
- 总时间: 18.92 秒
- 平均时间: 44.94 ms
- 最大时间: 234.56 ms
- 扫描行数: 20,128
- 返回行数: 12
- 缓存命中率: 56.78%

**瓶颈**: 按 `environment_id` 和 `status` 过滤的 Seq Scan

**建议**: 活跃生产部署上的 partial index

---

## 优化目标摘要

| 查询 | 平均时间 (优化前) | 目标时间 | 优先级 | 索引策略 |
|-------|-------------------|-------------|----------|----------------|
| Q1: 部署仪表盘 | 145ms | <30ms | 高 | deployed_at 的 B-tree |
| Q2: 按实验的培训运行 | 32ms | <10ms | 中 | 复合 (experiment, created_at) |
| Q3: 模型统计 | 143ms | <50ms | 高 | 外键索引 |
| Q4: 活跃部署 | 45ms | <15ms | 中 | Partial index (env + status) |

**预期总改进**: 查询延迟降低 75-85%
```

### 检查点 1

✅ 验证你的分析设置：

```bash
psql -h localhost -U ml_user -d ml_registry -c "SELECT COUNT(*) FROM pg_stat_statements;"
ls -la docs/performance-baseline.md
grep "Avg Time" docs/performance-baseline.md
```

---

## 第 2 部分：索引策略与实现

### 步骤 2.1：理解 PostgreSQL 索引类型

**索引类型与使用场景：**

| 索引类型 | 使用场景 | 示例 | 性能特征 |
|------------|----------|---------|----------------------------|
| **B-tree**（默认） | 等值、范围查询、排序 | `created_at`、`status`、外键 | 最佳通用，读写平衡 |
| **Hash** | 仅等值（=） | `uuid` 查找 | 等值比 B-tree 快，不支持范围 |
| **GIN**（通用倒排索引） | JSONB、数组、全文搜索 | `metadata @> '{"framework": "pytorch"}'` | 索引大，写入慢，读取快 |
| **GiST**（通用搜索树） | 几何数据、范围 | PostGIS、`tsrange` 重叠 | 有损，需要重新检查 |
| **BRIN**（块范围索引） | 具有自然顺序的大表 | 仅追加表上的 `created_at` | 索引极小，写入快，读取适中 |
| **Partial** | 行子集 | `WHERE status = 'active'` | 索引小，过滤查询非常快 |
| **Expression** | 函数结果 | `LOWER(email)` | 对计算值建索引 |

### 步骤 2.2：识别索引候选

使用此决策树：

```
查询有 WHERE/JOIN 子句？
├─ 是：过滤/连接的列是什么？
│  ├─ 单列 + 范围/排序 → B-tree
│  ├─ 多列 → 复合 B-tree（顺序很重要！）
│  ├─ JSONB 查询 → GIN
│  ├─ UUID/hash 仅等值 → Hash
│  └─ 大表上的时间序列 → BRIN
└─ 否：查询做聚合？
   └─ 是：对 GROUP BY/ORDER BY 中的列建索引
```

**分析每个慢查询：**

**查询 1 分析**：最新部署仪表盘

```sql
WHERE d.deployed_at > NOW() - INTERVAL '30 days'
ORDER BY d.deployed_at DESC
```

**决策**：`deployments(deployed_at DESC)` 上的 B-tree 索引
- **为什么 DESC？**：匹配 ORDER BY 方向，启用 index-only scan
- **为什么 B-tree？**：范围查询（>）和排序

---

**查询 2 分析**：按实验的培训运行

```sql
WHERE experiment_name = $1                -- 等值过滤
  AND created_at > NOW() - INTERVAL '7 days'  -- 范围过滤
ORDER BY accuracy DESC NULLS LAST
```

**决策**：`training_runs(experiment_name, created_at DESC, accuracy DESC)` 上的复合索引
- **列顺序**：选择性最高者优先（experiment_name），然后是范围过滤
- **包含 accuracy**：支持 ORDER BY，启用覆盖索引

---

**查询 3 分析**：模型统计

```sql
LEFT JOIN training_runs tr ON tr.version_id = mv.version_id  -- 外键连接
```

**决策**：`training_runs(version_id)` 上的 B-tree 索引
- **为什么？**：外键连接很常见，这启用嵌套循环连接
- **注意**：这应该在外键约束创建时就已创建！

---

**查询 4 分析**：活跃部署

```sql
WHERE d.environment_id = 1 AND d.status = 'active'  -- 子集过滤
```

**决策**：`deployments(version_id, deployed_at DESC) WHERE environment_id = 1 AND status = 'active'` 上的 partial index
- **为什么 partial？**：仅约 5% 的部署匹配过滤条件
- **好处**：索引更小，非生产部署的更新更快

### 步骤 2.3：计算索引选择性

在创建索引前，估计其效果：

```sql
-- 检查列选择性（越低 = 越具选择性 = 索引越好）
SELECT 'deployed_at' as column_name,
       COUNT(DISTINCT deployed_at)::float / COUNT(*)::float as selectivity,
       COUNT(*) as total_rows
FROM deployments
UNION ALL
SELECT 'status', COUNT(DISTINCT status)::float / COUNT(*)::float, COUNT(*) FROM deployments
UNION ALL
SELECT 'environment_id', COUNT(DISTINCT environment_id)::float / COUNT(*)::float, COUNT(*) FROM deployments;
```

**示例输出：**
```
 column_name    | selectivity | total_rows
----------------+-------------+------------
 deployed_at    |       0.956 |      20128  ← 高选择性，适合索引
 status         |       0.032 |      20128  ← 低选择性（仅有 5 种状态）
 environment_id |       0.015 |      20128  ← 低选择性（仅有 3 个环境）
```

**解读：**
- **高选择性（>0.8）**：优秀索引候选
- **中等选择性（0.3-0.8）**：适合复合索引或 partial index
- **低选择性（<0.3）**：仅在复合索引或 partial index 中使用

### 步骤 2.4：创建索引策略文档

创建 `docs/index-strategy.md`：

```markdown
# ML 模型注册索引策略

## 索引原则

1. **索引外键**：实现高效 JOIN
2. **索引 WHERE 子句**：频繁过滤的列
3. **索引 ORDER BY**：无需内存排序
4. **复合索引**：列顺序 = 选择性（高 → 低）
5. **覆盖索引**：包含列以实现 index-only scan
6. **Partial Index**：尽可能过滤到子集
7. **监控大小**：每个索引增加写入开销和存储

---

## 建议的索引

### 优先级 1: 外键索引（缺失！）

| 表 | 列 | 类型 | 理由 |
|-------|--------|------|-----------|
| `model_versions` | `model_id` | B-tree | 到 models 的 FK 连接 |
| `training_runs` | `version_id` | B-tree | 到 model_versions 的 FK 连接 |
| `deployments` | `version_id` | B-tree | 到 model_versions 的 FK 连接 |
| `deployments` | `environment_id` | B-tree | 到 environments 的 FK 连接 |
| `approvals` | `version_id` | B-tree | 到 model_versions 的 FK 连接 |

**影响**: 启用嵌套循环连接替代 hash join（小结果集快 10-50 倍）

---

### 优先级 2: 时间范围查询索引

| 表 | 索引定义 | 类型 | 使用场景 |
|-------|------------------|------|----------|
| `deployments` | `(deployed_at DESC)` | B-tree | 查询最近部署的仪表盘 |
| `training_runs` | `(created_at DESC)` | B-tree | 最近的培训运行，时间序列分析 |
| `training_runs` | `(started_at)` | BRIN | 大表，仅追加模式 |
| `model_versions` | `(registered_at DESC)` | B-tree | 最新版本查询 |

**影响**: 消除时间范围过滤上的顺序扫描（快 100 倍以上）

---

### 优先级 3: 常见查询模式的复合索引

| 表 | 索引定义 | 覆盖列 | 查询模式 |
|-------|------------------|------------------|---------------|
| `training_runs` | `(experiment_name, created_at DESC)` | `accuracy, status` | 实验仪表盘 |
| `training_runs` | `(model_name, status)` | `accuracy, created_at` | 模型成功率 |
| `deployments` | `(environment_id, status, deployed_at DESC)` | `version_id` | 按环境的活跃部署 |

**影响**: 在单次索引扫描中支持多个过滤 + 排序

---

### 优先级 4: 热查询的 Partial Index

| 表 | 索引定义 | 过滤条件 | 大小减少 |
|-------|------------------|------------------|----------------|
| `deployments` | `(version_id, deployed_at DESC)` | `WHERE environment_id = 1 AND status = 'active'` | 减小 95% |
| `training_runs` | `(model_name, created_at DESC)` | `WHERE status = 'succeeded'` | 减小 80% |
| `approvals` | `(version_id, approved_at DESC)` | `WHERE status = 'approved'` | 减小 70% |

**影响**: 大幅减小索引，更快的更新，定向查询

---

## 索引大小估算

| 索引类别 | 数量 | 估计大小 | 写入开销 |
|----------------|-------|----------------|----------------|
| 外键 | 5 | ~50 MB | +5% 写入时间 |
| 时间范围 | 4 | ~80 MB | +8% 写入时间 |
| 复合 | 3 | ~120 MB | +12% 写入时间 |
| Partial | 3 | ~30 MB | +2% 写入时间（已过滤） |
| JSONB | 3 | ~200 MB | +20% 写入时间 |
| **总计** | **18** | **~480 MB** | **+47% 写入开销** |

**权衡分析**: 480 MB 额外存储和约 50% 更慢写入，换 10-100 倍更快读取

**建议**: 立即实施优先级 1-3，根据工作负载评估优先级 4-5

---

## 维护计划

- **REINDEX**: 对频繁更新的索引每月执行
- **VACUUM ANALYZE**: 每周或在批量操作后执行
- **监控膨胀**: 索引膨胀 > 50% 需要 REINDEX
- **跟踪未使用的索引**: 如果 30 天后 `idx_scan = 0` 则删除
```

### 步骤 2.5：实现索引（DDL 脚本）

创建 `sql/50_indexes.sql`：

```sql
-- =============================================================================
-- ML Model Registry - Performance Indexes
-- =============================================================================
-- This script creates indexes to optimize query performance.
-- All indexes use CONCURRENTLY to avoid blocking production writes.
--
-- Estimated execution time: 5-15 minutes (depending on data volume)
-- =============================================================================

\timing on

\echo '================================'
\echo 'Creating Performance Indexes'
\echo '================================'

-- =============================================================================
-- Priority 1: Foreign Key Indexes (Critical)
-- =============================================================================

\echo '\n[1/18] Creating index on model_versions(model_id)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_model_versions_model_id
    ON model_versions (model_id);

\echo '[2/18] Creating index on training_runs(version_id)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_training_runs_version_id
    ON training_runs (version_id);

\echo '[3/18] Creating index on deployments(version_id)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_deployments_version_id
    ON deployments (version_id);

\echo '[4/18] Creating index on deployments(environment_id)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_deployments_environment_id
    ON deployments (environment_id);

\echo '[5/18] Creating index on approvals(version_id)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_approvals_version_id
    ON approvals (version_id);

-- =============================================================================
-- Priority 2: Time-Range Query Indexes
-- =============================================================================

\echo '[6/18] Creating index on deployments(deployed_at DESC)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_deployments_deployed_at_desc
    ON deployments (deployed_at DESC);

\echo '[7/18] Creating index on training_runs(created_at DESC)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_training_runs_created_at_desc
    ON training_runs (created_at DESC);

\echo '[8/18] Creating BRIN index on training_runs(started_at)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_training_runs_started_at_brin
    ON training_runs USING BRIN (started_at)
    WITH (pages_per_range = 128);

\echo '[9/18] Creating index on model_versions(registered_at DESC)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_model_versions_registered_at_desc
    ON model_versions (registered_at DESC);

-- =============================================================================
-- Priority 3: Composite Indexes for Common Queries
-- =============================================================================

\echo '[10/18] Creating composite index on training_runs(experiment_name, created_at)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_training_runs_exp_created
    ON training_runs (experiment_name, created_at DESC)
    INCLUDE (accuracy, status);

\echo '[11/18] Creating composite index on training_runs(model_name, status)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_training_runs_model_status
    ON training_runs (model_name, status)
    INCLUDE (accuracy, created_at);

\echo '[12/18] Creating composite index on deployments(environment, status, deployed_at)...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_deployments_env_status_deployed
    ON deployments (environment_id, status, deployed_at DESC)
    INCLUDE (version_id);

-- =============================================================================
-- Priority 4: Partial Indexes for Hot Queries
-- =============================================================================

\echo '[13/18] Creating partial index on active production deployments...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_deployments_prod_active
    ON deployments (version_id, deployed_at DESC)
    WHERE environment_id = 1 AND status = 'active';

\echo '[14/18] Creating partial index on successful training runs...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_training_runs_succeeded
    ON training_runs (model_name, created_at DESC)
    INCLUDE (accuracy, run_id)
    WHERE status = 'succeeded';

\echo '[15/18] Creating partial index on approved versions...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_approvals_approved
    ON approvals (version_id, approved_at DESC)
    WHERE status = 'approved';

-- =============================================================================
-- Priority 5: JSONB Indexes for Metadata Queries
-- =============================================================================

\echo '[16/18] Creating GIN index on training_runs.hyperparameters...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_training_runs_hyperparameters_gin
    ON training_runs USING GIN (hyperparameters jsonb_path_ops);

\echo '[17/18] Creating GIN index on training_runs.metrics...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_training_runs_metrics_gin
    ON training_runs USING GIN (metrics jsonb_path_ops);

\echo '[18/18] Creating GIN index on model_versions.metadata_...'
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_model_versions_metadata_gin
    ON model_versions USING GIN (metadata_ jsonb_path_ops);

-- =============================================================================
-- Verify Index Creation
-- =============================================================================

\echo '\n================================'
\echo 'Verifying Indexes'
\echo '================================'

SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_indexes
LEFT JOIN pg_class ON pg_class.relname = indexname
WHERE schemaname = 'public'
  AND indexname LIKE 'idx_%'
ORDER BY tablename, indexname;

-- =============================================================================
-- Analyze Tables to Update Statistics
-- =============================================================================

\echo '\n================================'
\echo 'Updating Table Statistics'
\echo '================================'

ANALYZE VERBOSE models;
ANALYZE VERBOSE model_versions;
ANALYZE VERBOSE training_runs;
ANALYZE VERBOSE deployments;
ANALYZE VERBOSE approvals;

\echo '\n================================'
\echo 'Index Creation Complete!'
\echo '================================'
```

**执行脚本：**

```bash
psql -h localhost -U ml_user -d ml_registry -f sql/50_indexes.sql

# 在另一个终端监控进度
watch -n 2 "psql -h localhost -U ml_user -d ml_registry -c \"
    SELECT now() - query_start AS duration, state, query
    FROM pg_stat_activity
    WHERE query LIKE '%CREATE INDEX%' AND state != 'idle';
\""
```

### 步骤 2.6：验证索引使用

创建索引后，验证它们是否被使用：

```sql
-- 检查索引大小
SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) as size,
       idx_scan as scans, idx_tup_read as tuples_read, idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;

-- 使用 EXPLAIN 重新运行慢查询
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT m.model_name, mv.semver, d.status, d.deployed_at
FROM deployments d
INNER JOIN model_versions mv ON d.version_id = mv.version_id
INNER JOIN models m ON mv.model_id = m.model_id
WHERE d.deployed_at > NOW() - INTERVAL '30 days'
ORDER BY d.deployed_at DESC
LIMIT 50;
```

**预期计划（优化后）：**

```
Limit  (cost=45.67..46.05 rows=50 width=120) (actual time=8.234..8.567 rows=50 loops=1)
  Buffers: shared hit=156 read=12
  ->  Nested Loop  (cost=1.23..89.45 rows=4450 width=120) (actual time=0.345..8.123 rows=50 loops=1)
        ->  Nested Loop  (cost=0.82..45.67 rows=4450 width=88) (actual time=0.234..5.678 rows=50 loops=1)
              ->  Index Scan Backward using idx_deployments_deployed_at_desc on deployments d
                  (cost=0.42..23.45 rows=4450 width=56) (actual time=0.123..2.345 rows=50 loops=1)
                    Index Cond: (deployed_at > (now() - '30 days'::interval))
              ->  Index Scan using model_versions_pkey on model_versions mv
                  (cost=0.40..0.48 rows=1 width=32) (actual time=0.034..0.035 rows=1 loops=50)
                    Index Cond: (version_id = d.version_id)
        ->  Index Scan using models_pkey on models m
              (cost=0.41..0.49 rows=1 width=40) (actual time=0.023..0.024 rows=1 loops=50)
              Index Cond: (model_id = mv.model_id)
Planning Time: 0.876 ms
Execution Time: 8.678 ms
```

**改进：**
- ✅ **执行时间**：145ms → 8.7ms（快 16 倍！）
- ✅ **索引扫描**替代了顺序扫描
- ✅ **嵌套循环**替代了 hash join（更适合 LIMIT 查询）
- ✅ **缓冲区使用**：5,877 blocks → 168 blocks（I/O 减少 35 倍）
- ✅ **反向索引扫描**利用了 DESC 索引

### 检查点 2

✅ 验证索引在运行：

```bash
psql -h localhost -U ml_user -d ml_registry -c "
    SELECT COUNT(*) as total_indexes
    FROM pg_indexes
    WHERE schemaname = 'public' AND indexname LIKE 'idx_%';
"
psql -h localhost -U ml_user -d ml_registry -c "
    SELECT tablename, indexname, idx_scan
    FROM pg_stat_user_indexes
    WHERE schemaname = 'public'
    ORDER BY idx_scan DESC
    LIMIT 10;
"
```

---

## 第 3 部分：性能测量与验证

### 步骤 3.1：优化后重新运行工作负载

```bash
psql -h localhost -U ml_user -d ml_registry -c "SELECT pg_stat_statements_reset();"
python scripts/generate_workload.py 60 10

psql -h localhost -U ml_user -d ml_registry -c "
    SELECT LEFT(query, 80) as query_preview, calls,
           ROUND(mean_exec_time::numeric, 2) as avg_ms,
           ROUND(max_exec_time::numeric, 2) as max_ms,
           ROUND(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 1) AS cache_hit_pct
    FROM pg_stat_statements
    WHERE query NOT LIKE '%pg_stat%'
    ORDER BY mean_exec_time DESC
    LIMIT 10;
"
```

### 步骤 3.2：创建前后对比

创建 `docs/performance-results.md`：

```markdown
# 性能优化结果

**优化日期**: 2025-10-23
**方法**: 索引实现（创建了 18 个索引）
**测试时长**: 60 秒 @ 10 QPS

---

## 改进摘要

| 指标 | 优化前 | 优化后 | 改进 |
|--------|--------|-------|-------------|
| **平均查询延迟 (P50)** | 145ms | 12ms | **快 92%** |
| **最大查询延迟 (P99)** | 1,250ms | 45ms | **快 96%** |
| **缓存命中率** | 45% | 94% | **+49 个百分点** |
| **>1s 的查询** | 18 (7.3%) | 0 (0%) | **100% 消除** |
| **总查询时间** | 35.67s | 2.89s | **减少 92%** |
| **数据库 CPU 使用率** | 78% | 23% | **-55 个百分点** |

---

## 各查询改进

### 查询 1: 最新部署仪表盘

| 指标 | 优化前 | 优化后 | 改进 |
|--------|--------|-------|-------------|
| 平均时间 | 145.58ms | 8.67ms | **快 94%** |
| 最大时间 | 1,250.34ms | 23.45ms | **快 98%** |
| 缓存命中 | 45.23% | 95.67% | +50.44pp |
| 执行计划 | Seq Scan → Sort | Index Scan (反向) | ✅ |

**使用的索引**: `idx_deployments_deployed_at_desc`

---

### 查询 2: 按实验的培训运行

| 指标 | 优化前 | 优化后 | 改进 |
|--------|--------|-------|-------------|
| 平均时间 | 31.90ms | 5.12ms | **快 84%** |
| 最大时间 | 456.78ms | 18.90ms | **快 96%** |
| 缓存命中 | 78.92% | 97.34% | +18.42pp |
| 执行计划 | Seq Scan → Filter | Index Scan | ✅ |

**使用的索引**: `idx_training_runs_exp_created`（复合 + 覆盖）

---

### 查询 3: 模型统计聚合

| 指标 | 优化前 | 优化后 | 改进 |
|--------|--------|-------|-------------|
| 平均时间 | 143.21ms | 18.34ms | **快 87%** |
| 最大时间 | 890.12ms | 67.89ms | **快 92%** |
| 缓存命中 | 38.45% | 91.23% | +52.78pp |
| 执行计划 | Hash Join | Nested Loop (带索引) | ✅ |

**使用的索引**: `idx_training_runs_version_id`、`idx_model_versions_model_id`

---

### 查询 4: 按环境的活跃部署

| 指标 | 优化前 | 优化后 | 改进 |
|--------|--------|-------|-------------|
| 平均时间 | 44.94ms | 2.34ms | **快 95%** |
| 最大时间 | 234.56ms | 8.12ms | **快 97%** |
| 缓存命中 | 56.78% | 98.45% | +41.67pp |
| 执行计划 | Seq Scan → Filter | Partial Index Scan | ✅ |

**使用的索引**: `idx_deployments_prod_active`（partial index）

---

## 索引效果分析

| 索引 | 大小 | 扫描 | 读取元组 | 效果 |
|-------|------|-------|-------------|---------------|
| `idx_deployments_deployed_at_desc` | 8.2 MB | 245 | 12,250 | ⭐⭐⭐⭐⭐ 高 |
| `idx_training_runs_exp_created` | 15.4 MB | 892 | 17,840 | ⭐⭐⭐⭐⭐ 高 |
| `idx_training_runs_version_id` | 12.1 MB | 1,248 | 67,890 | ⭐⭐⭐⭐⭐ 高 |
| `idx_deployments_prod_active` | 0.9 MB | 421 | 5,052 | ⭐⭐⭐⭐⭐ 高 (partial) |
| `idx_model_versions_model_id` | 6.7 MB | 2,134 | 89,234 | ⭐⭐⭐⭐⭐ 高 |
| `idx_training_runs_hyperparameters_gin` | 45.2 MB | 0 | 0 | ⚠️ 未使用（考虑删除） |
| `idx_training_runs_metrics_gin` | 38.9 MB | 0 | 0 | ⚠️ 未使用（考虑删除） |

---

## 成本效益分析

**收益**:
- 平均查询延迟降低 92%
- 零查询超时
- 缓存命中率 94%（接近最优）
- CPU 使用率减少 55%
- 支持扩展到当前流量的 10 倍

**成本**:
- 478 MB 额外存储（索引大小）
- INSERT/UPDATE 时间增加约 47%（已测量）
- 索引创建约 8 分钟（一次性）
- 持续维护（每月 REINDEX）

**建议**: **巨大胜利**。对于读取密集型的 ML 注册工作负载，权衡明显偏向索引。

---

## 经验教训

1. **外键索引至关重要**：最大改进来自索引 FK 列
2. **复合索引优于多个单列索引**：查询计划器使用一个精心设计的复合索引 vs 多个单独索引
3. **Partial Index 被低估**：热查询大小减少 95%
4. **JSONB 索引成本高**：仅当你有使用它们的查询时才创建
5. **测量而非猜测**：EXPLAIN ANALYZE 揭示了实际瓶颈

---

## 后续步骤

1. ✅ 监控索引使用 7 天
2. ✅ 删除未使用的 JSONB 索引
3. ✅ 为慢查询设置自动告警（>100ms）
4. ✅ 为高更新表安排每月 REINDEX
5. ✅ 在 runbook 中记录索引维护
```

### 步骤 3.3：识别未使用的索引

运行一周后，检查未使用的索引：

```sql
-- 查找从未被扫描过的索引
SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) as size,
       idx_scan as scans, idx_tup_read as tuples_read
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
  AND idx_scan = 0
  AND indexname LIKE 'idx_%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- 删除未使用的索引
DROP INDEX CONCURRENTLY IF EXISTS idx_training_runs_hyperparameters_gin;
DROP INDEX CONCURRENTLY IF EXISTS idx_training_runs_metrics_gin;
```

### 检查点 3

✅ 验证性能改进：

```bash
diff docs/performance-baseline.md docs/performance-results.md
psql -h localhost -U ml_user -d ml_registry -c "
    SELECT COUNT(*) as unused_indexes
    FROM pg_stat_user_indexes
    WHERE idx_scan = 0 AND indexname LIKE 'idx_%';
"
psql -h localhost -U ml_user -d ml_registry -c "
    SELECT ROUND(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit + blks_read), 0), 2) AS cache_hit_ratio
    FROM pg_stat_database
    WHERE datname = 'ml_registry';
"
```

---

## 第 4 部分：数据库维护与监控

### 步骤 4.1：理解 VACUUM 和 ANALYZE

**PostgreSQL 维护概念：**

1. **VACUUM**：回收死元组的空间（已删除/更新的行）
   - **VACUUM**：将空间标记为可重用（非阻塞）
   - **VACUUM FULL**：重写表以压缩空间（阻塞写入，谨慎使用）
   - **Autovacuum**：自动后台 vacuum 进程

2. **ANALYZE**：更新查询计划器的表统计信息
   - 跟踪数据分布以进行优化决策
   - 应在批量插入/更新后运行
   - 随 autovacuum 自动运行

3. **REINDEX**：重建膨胀的索引
   - 修复来自大量更新的索引膨胀
   - 使用 `REINDEX CONCURRENTLY`（PostgreSQL 12+）避免阻塞

### 步骤 4.2：手动维护操作

创建 `scripts/maintenance.sql`：

```sql
-- =============================================================================
-- Database Maintenance Script
-- Run weekly or after bulk operations
-- =============================================================================

\timing on
\echo '================================'
\echo 'Running Database Maintenance'
\echo '================================'

\echo '\n[1/4] VACUUM ANALYZE: models...'
VACUUM (VERBOSE, ANALYZE, SKIP_LOCKED) models;

\echo '[2/4] VACUUM ANALYZE: model_versions...'
VACUUM (VERBOSE, ANALYZE, SKIP_LOCKED) model_versions;

\echo '[3/4] VACUUM ANALYZE: training_runs...'
VACUUM (VERBOSE, ANALYZE, SKIP_LOCKED) training_runs;

\echo '[4/4] VACUUM ANALYZE: deployments...'
VACUUM (VERBOSE, ANALYZE, SKIP_LOCKED) deployments;

\echo '\n================================'
\echo 'Checking Table Bloat'
\echo '================================'

SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename)) AS index_size,
       ROUND(100 * (pg_total_relation_size(schemaname||'.'||tablename) - pg_relation_size(schemaname||'.'||tablename))::numeric / NULLIF(pg_total_relation_size(schemaname||'.'||tablename), 0), 2) AS index_ratio
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

CREATE EXTENSION IF NOT EXISTS pgstattuple;

\echo '\n================================'
\echo 'Checking Index Bloat'
\echo '================================'

SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
       ROUND((pgstatindex(indexrelid)).avg_leaf_density, 2) AS leaf_density,
       CASE
           WHEN (pgstatindex(indexrelid)).avg_leaf_density < 70 THEN '高膨胀 - 考虑 REINDEX'
           WHEN (pgstatindex(indexrelid)).avg_leaf_density < 85 THEN '中等膨胀'
           ELSE '健康'
       END AS status
FROM pg_indexes
JOIN pg_class ON pg_class.relname = indexname
WHERE schemaname = 'public'
  AND indexname LIKE 'idx_%'
  AND pg_relation_size(indexrelid) > 1024 * 1024
ORDER BY (pgstatindex(indexrelid)).avg_leaf_density ASC;

\echo '\n================================'
\echo 'Creating Extended Statistics'
\echo '================================'

CREATE STATISTICS IF NOT EXISTS training_runs_exp_created_stats (dependencies)
    ON experiment_name, created_at FROM training_runs;
CREATE STATISTICS IF NOT EXISTS deployments_env_status_stats (dependencies)
    ON environment_id, status FROM deployments;
ANALYZE training_runs;
ANALYZE deployments;

\echo '\n================================'
\echo 'Maintenance Complete!'
\echo '================================'
```

**执行维护：**

```bash
psql -h localhost -U ml_user -d ml_registry -f scripts/maintenance.sql
```

### 步骤 4.3：配置 Autovacuum

编辑 `postgresql.conf`：

```conf
# Autovacuum Configuration (aggressive for high-write workload)
autovacuum = on
autovacuum_max_workers = 4
autovacuum_naptime = 30s

autovacuum_vacuum_scale_factor = 0.2
autovacuum_vacuum_threshold = 10000

autovacuum_analyze_scale_factor = 0.1
autovacuum_analyze_threshold = 5000
```

**监控 autovacuum 活动：**

```sql
SELECT schemaname, relname, last_vacuum, last_autovacuum,
       last_analyze, last_autoanalyze,
       n_tup_ins as inserts, n_tup_upd as updates,
       n_tup_del as deletes, n_dead_tup as dead_tuples
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY n_dead_tup DESC;
```

### 步骤 4.4：使用 Prometheus 设置监控

创建 `docker-compose.monitoring.yml`：

```yaml
version: '3.8'

services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    container_name: postgres-exporter
    environment:
      DATA_SOURCE_NAME: "postgresql://ml_user:ml_password@ml-registry-postgres:5432/ml_registry?sslmode=disable"
      PG_EXPORTER_EXTEND_QUERY_PATH: "/etc/postgres_exporter/queries.yaml"
    ports:
      - "9187:9187"
    volumes:
      - ./monitoring/postgres_queries.yaml:/etc/postgres_exporter/queries.yaml:ro
    networks:
      - ml-registry-network

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
    networks:
      - ml-registry-network

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana
      - ./monitoring/grafana-dashboards:/etc/grafana/provisioning/dashboards:ro
      - ./monitoring/grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml:ro
    networks:
      - ml-registry-network

volumes:
  prometheus-data:
  grafana-data:

networks:
  ml-registry-network:
    external: true
```

创建 `monitoring/postgres_queries.yaml`：

```yaml
pg_database_size:
  query: "SELECT pg_database_size(current_database()) as bytes"
  metrics:
    - bytes:
        usage: "GAUGE"
        description: "Database size in bytes"

pg_slow_queries:
  query: |
    SELECT COUNT(*) as count
    FROM pg_stat_statements
    WHERE mean_exec_time > 100
  metrics:
    - count:
        usage: "GAUGE"
        description: "Number of queries with avg execution time > 100ms"

pg_cache_hit_ratio:
  query: |
    SELECT ROUND(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit + blks_read), 0), 2) AS ratio
    FROM pg_stat_database
    WHERE datname = current_database()
  metrics:
    - ratio:
        usage: "GAUGE"
        description: "Cache hit ratio percentage"

pg_table_sizes:
  query: |
    SELECT schemaname || '.' || tablename as table_name,
           pg_total_relation_size(schemaname||'.'||tablename) as bytes
    FROM pg_tables
    WHERE schemaname = 'public'
  metrics:
    - table_name:
        usage: "LABEL"
        description: "Table name"
    - bytes:
        usage: "GAUGE"
        description: "Total table size including indexes"

pg_index_usage:
  query: |
    SELECT schemaname || '.' || indexname as index_name,
           idx_scan as scans, pg_relation_size(indexrelid) as bytes
    FROM pg_stat_user_indexes
    WHERE schemaname = 'public'
  metrics:
    - index_name:
        usage: "LABEL"
        description: "Index name"
    - scans:
        usage: "COUNTER"
        description: "Number of index scans"
    - bytes:
        usage: "GAUGE"
        description: "Index size in bytes"

pg_deadlocks:
  query: "SELECT deadlocks as count FROM pg_stat_database WHERE datname = current_database()"
  metrics:
    - count:
        usage: "COUNTER"
        description: "Number of deadlocks detected"

pg_connections:
  query: |
    SELECT state, COUNT(*) as count
    FROM pg_stat_activity
    WHERE datname = current_database()
    GROUP BY state
  metrics:
    - state:
        usage: "LABEL"
        description: "Connection state"
    - count:
        usage: "GAUGE"
        description: "Number of connections in this state"
```

**启动监控栈：**

```bash
docker-compose -f docker-compose.monitoring.yml up -d
open http://localhost:3000  # admin/admin
open http://localhost:9090
```

### 步骤 4.5：创建 Grafana 仪表盘

创建 `monitoring/grafana-dashboards/postgres-performance.json`：

```json
{
  "dashboard": {
    "title": "ML Registry Database Performance",
    "panels": [
      {"title": "Query Latency (P50, P95, P99)", "targets": [{"expr": "histogram_quantile(0.50, rate(pg_stat_statements_exec_time_bucket[5m]))", "legendFormat": "P50"}, {"expr": "histogram_quantile(0.95, rate(pg_stat_statements_exec_time_bucket[5m]))", "legendFormat": "P95"}, {"expr": "histogram_quantile(0.99, rate(pg_stat_statements_exec_time_bucket[5m]))", "legendFormat": "P99"}]},
      {"title": "Cache Hit Ratio", "targets": [{"expr": "pg_cache_hit_ratio", "legendFormat": "Cache Hit %"}], "thresholds": [{"value": 90, "color": "green"}, {"value": 80, "color": "yellow"}, {"value": 0, "color": "red"}]},
      {"title": "Slow Queries (>100ms)", "targets": [{"expr": "pg_slow_queries", "legendFormat": "Slow Queries"}]},
      {"title": "Database Size", "targets": [{"expr": "pg_database_size_bytes", "legendFormat": "Database Size"}]},
      {"title": "Active Connections", "targets": [{"expr": "sum(pg_connections{state='active'})", "legendFormat": "Active"}, {"expr": "sum(pg_connections{state='idle'})", "legendFormat": "Idle"}]},
      {"title": "Deadlocks", "targets": [{"expr": "rate(pg_deadlocks[5m])", "legendFormat": "Deadlocks/sec"}]}
    ]
  }
}
```

### 步骤 4.6：配置告警规则

创建 `monitoring/alert-rules.yml`：

```yaml
groups:
  - name: postgres_alerts
    interval: 30s
    rules:
      - alert: HighQueryLatency
        expr: histogram_quantile(0.99, rate(pg_stat_statements_exec_time_bucket[5m])) > 1000
        for: 5m
        labels: {severity: warning}
        annotations: {summary: "P99 query latency above 1 second", description: "Database queries are slow (P99: {{ $value }}ms)"}
      - alert: LowCacheHitRatio
        expr: pg_cache_hit_ratio < 85
        for: 10m
        labels: {severity: warning}
        annotations: {summary: "Cache hit ratio below 85%", description: "Too many disk reads (cache hit: {{ $value }}%)"}
      - alert: HighConnectionCount
        expr: sum(pg_connections) > 80
        for: 5m
        labels: {severity: warning}
        annotations: {summary: "High number of database connections", description: "Connection count: {{ $value }} (limit: 100)"}
      - alert: DeadlocksDetected
        expr: rate(pg_deadlocks[5m]) > 0
        for: 1m
        labels: {severity: critical}
        annotations: {summary: "Database deadlocks detected", description: "Deadlock rate: {{ $value }}/sec"}
```

### 检查点 4

✅ 验证维护和监控：

```bash
psql -h localhost -U ml_user -d ml_registry -c "
    SELECT COUNT(*) as autovacuum_workers
    FROM pg_stat_activity
    WHERE query LIKE '%autovacuum%';
"
curl -s http://localhost:9090/api/v1/query?query=pg_cache_hit_ratio | jq '.data.result[0].value[1]'
curl -s http://localhost:3000/api/health | jq '.'
```

---

## 第 5 部分：高级优化技术

### 步骤 5.1：查询重写以提高性能

**常见性能反模式：**

**❌ 不好：不同列上的 OR 条件**

```sql
SELECT * FROM training_runs
WHERE model_name = 'fraud-detection' OR experiment_name = 'exp-001';
```

**✅ 好：对 OR 条件使用 UNION**

```sql
SELECT * FROM training_runs WHERE model_name = 'fraud-detection'
UNION
SELECT * FROM training_runs WHERE experiment_name = 'exp-001';
```

---

**❌ 不好：NOT IN 子查询**

```sql
SELECT * FROM models WHERE model_id NOT IN (SELECT model_id FROM model_versions);
```

**✅ 好：使用 LEFT JOIN 加 NULL 检查**

```sql
SELECT m.*
FROM models m
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
WHERE mv.model_id IS NULL;
```

---

**❌ 不好：使用 OFFSET 的 SELECT * 分页**

```sql
SELECT * FROM training_runs ORDER BY created_at DESC LIMIT 20 OFFSET 10000;
```

**✅ 好：Keyset 分页**

```sql
SELECT * FROM training_runs
WHERE created_at < '2024-01-15 10:00:00'
ORDER BY created_at DESC
LIMIT 20;
```

---

**❌ 不好：WHERE 子句中的函数调用**

```sql
SELECT * FROM training_runs WHERE DATE(created_at) = '2024-01-15';
```

**✅ 好：范围查询**

```sql
SELECT * FROM training_runs
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16';
```

### 步骤 5.2：大表分区

对于 > 1000 万行或 > 50GB 的表，考虑分区：

```sql
CREATE TABLE training_runs_partitioned (
    run_id UUID PRIMARY KEY, model_name TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE training_runs_2024_01 PARTITION OF training_runs_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE training_runs_2024_02 PARTITION OF training_runs_partitioned
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

### 步骤 5.3：使用 PgBouncer 的连接池

对于有许多短连接的应用，添加连接池：

```yaml
services:
  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DATABASE_URL: "postgresql://ml_user:ml_password@ml-registry-postgres:5432/ml_registry"
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 1000
      DEFAULT_POOL_SIZE: 25
      SERVER_IDLE_TIMEOUT: 600
      MAX_DB_CONNECTIONS: 100
    ports: ["6432:6432"]
```

### 步骤 5.4：用于扩展读取的只读副本

对于读取密集型工作负载，设置流复制：

```yaml
services:
  postgres-primary:
    image: postgres:14
    environment:
      POSTGRES_USER: ml_user
      POSTGRES_PASSWORD: ml_password
      POSTGRES_DB: ml_registry
    ports: ["5432:5432"]

  postgres-replica:
    image: postgres:14
    ports: ["5433:5432"]
    depends_on: [postgres-primary]
```

### 检查点 5

✅ 验证高级优化：

```bash
psql -h localhost -U ml_user -d ml_registry -c "
    SELECT tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
    FROM pg_tables WHERE schemaname = 'public' AND tablename LIKE '%_partition_%'
    ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
"
```

---

## 第 6 部分：生产 Runbook 与文档

### 步骤 6.1：创建性能调优 Runbook

创建 `docs/runbook-performance-tuning.md`：

```markdown
# 数据库性能调优 Runbook

## 快速参考

| 问题 | 症状 | 操作 | 优先级 |
|-------|---------|--------|----------|
| 慢查询 | P99 > 500ms | 运行 EXPLAIN ANALYZE，检查索引 | 🔴 高 |
| 缓存命中率低 | < 90% | 增加 `shared_buffers`，检查查询模式 | 🟡 中 |
| 高连接数 | > 80% 最大值 | 添加 PgBouncer，检查连接泄漏 | 🔴 高 |
| 表膨胀 | > 50% | 在维护窗口运行 VACUUM FULL | 🟡 中 |
| 索引膨胀 | 叶密度 < 70% | REINDEX CONCURRENTLY | 🟠 低 |
| 死锁 | > 0/小时 | 检查事务逻辑，添加重试 | 🔴 高 |

---

## 过程 1：调查慢查询

**触发**: 查询延迟 > 500ms 的告警触发

**步骤**:

1. **识别慢查询:**
   ```sql
   SELECT query, calls, mean_exec_time, max_exec_time
   FROM pg_stat_statements WHERE mean_exec_time > 100
   ORDER BY mean_exec_time DESC LIMIT 10;
   ```

2. **捕获执行计划:**
   ```sql
   EXPLAIN (ANALYZE, BUFFERS, VERBOSE) <slow_query>;
   ```

3. **检查缺失的索引:**
   - 在大表上查找 "Seq Scan"
   - 检查 "rows" vs "actual rows" 不匹配（统计信息过期）
   - 验证 WHERE/JOIN 列上是否存在索引

4. **应用修复:**
   - **如果缺失索引**: `CREATE INDEX CONCURRENTLY ...`
   - **如果统计信息过期**: `ANALYZE <table>;`
   - **如果查询糟糕**: 使用第 5 部分中的技术重写

5. **验证改进:**
   - 重新运行 EXPLAIN ANALYZE
   - 监控指标 24 小时
   - 记录在 `docs/performance-improvements.md`

**回滚计划**: `DROP INDEX CONCURRENTLY IF EXISTS <index_name>;`

---

## 过程 2：处理表膨胀

**触发**: 表大小增加了 > 2 倍而没有相应的数据增长

**步骤**:

1. **检查膨胀:**
   ```sql
   SELECT tablename, pg_size_pretty(pg_total_relation_size(tablename)) as size,
          n_dead_tup, ROUND(100 * n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0), 2) as dead_pct
   FROM pg_stat_user_tables WHERE schemaname = 'public'
   ORDER BY n_dead_tup DESC;
   ```

2. **如果 dead_pct > 20%**，运行 VACUUM:
   ```sql
   VACUUM (VERBOSE, ANALYZE) training_runs;
   ```

3. **如果膨胀持续存在**，安排 VACUUM FULL（需要维护窗口）:
   ```sql
   VACUUM FULL training_runs;
   ```

4. **调整 autovacuum** 以防止复发:
   ```sql
   ALTER TABLE training_runs SET (autovacuum_vacuum_scale_factor = 0.1, autovacuum_vacuum_threshold = 5000);
   ```

---

## 过程 3：紧急性能分类

**触发**: 数据库 CPU > 90%，查询超时

**立即操作（5 分钟内）**:

1. **识别阻塞查询:**
   ```sql
   SELECT pid, query, state, wait_event_type, wait_event
   FROM pg_stat_activity WHERE state != 'idle' ORDER BY query_start;
   ```

2. **杀死长时间运行的查询**（如果安全）:
   ```sql
   SELECT pg_terminate_backend(pid)
   FROM pg_stat_activity WHERE state = 'active'
     AND query_start < NOW() - INTERVAL '5 minutes'
     AND query NOT LIKE '%pg_stat_activity%';
   ```

3. **检查锁争用:**

4. **临时启用查询日志:**
   ```sql
   ALTER SYSTEM SET log_min_duration_statement = 0;
   SELECT pg_reload_conf();
   ```

5. **如果 15 分钟后问题仍然存在**，升级到 DBA

---

## 过程 4：每月维护

**日程**: 每月第一个星期日，UTC 凌晨 2 点

**检查清单**:

- [ ] 运行 `scripts/maintenance.sql`
- [ ] 检查未使用的索引（`idx_scan = 0`）
- [ ] REINDEX 膨胀索引（leaf_density < 70%）
- [ ] 检查 pg_stat_statements 中新的慢查询
- [ ] 使用当前指标更新 `docs/performance-baseline.md`
- [ ] 验证备份恢复（恢复到测试数据库）
- [ ] 检查告警阈值（根据流量变化调整）
```

### 步骤 6.2：记录优化历史

创建 `docs/performance-improvements.md`：

```markdown
# 性能优化历史

## 2025-10-23：初始索引创建

**问题**: 仪表盘查询超时（P99 > 1,000ms）

**根因**: 外键和时间范围过滤器上缺少索引

**解决方案**: 创建了 18 个索引，涵盖：
- 外键列（5 个索引）
- 时间范围查询（4 个索引）
- 复合索引（3 个索引）
- Partial index（3 个索引）
- JSONB 索引（3 个索引）

**结果**:
- 平均查询延迟降低 92%
- P99 延迟降低 96%
- 缓存命中率从 45% 提高到 94%
- 零查询超时

---

## 未来优化候选

1. **分区 training_runs 表**（优先级：中）
   - 理由：表接近 1000 万行
   - 预期收益：时间范围查询快 50%
   - 预计工作量：4 小时

2. **添加只读副本**（如果流量 > 100 QPS，优先级：高）
   - 理由：读取密集型工作负载（95% 读取）
   - 预期收益：读取能力提升 2 倍
   - 预计工作量：8 小时

3. **实现 PgBouncer**（当前优先级：低）
   - 理由：连接数稳定（< 50）
   - 重新考虑时机：连接数 > 80
```

### 检查点 6

✅ 验证 runbook 完整性：

```bash
ls -la docs/runbook-performance-tuning.md
ls -la docs/performance-improvements.md
ls -la docs/index-strategy.md
ls -la scripts/maintenance.sql
```

---

## 第 7 部分：动手挑战

### 挑战 1：优化复杂分析查询

**问题**: 以下查询速度慢（平均 850ms）：

```sql
SELECT m.model_name, COUNT(DISTINCT d.deployment_id) as total_deployments,
       COUNT(DISTINCT d.deployment_id) FILTER (WHERE d.status = 'active') as active_deployments,
       MAX(d.deployed_at) as last_deployment,
       AVG(tr.accuracy) FILTER (WHERE tr.status = 'succeeded') as avg_accuracy,
       COUNT(DISTINCT tr.run_id) as total_runs
FROM models m
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
LEFT JOIN deployments d ON mv.version_id = d.version_id
LEFT JOIN training_runs tr ON mv.version_id = tr.version_id
WHERE m.created_at > NOW() - INTERVAL '90 days'
GROUP BY m.model_id, m.model_name
HAVING COUNT(DISTINCT d.deployment_id) > 0
ORDER BY total_deployments DESC;
```

**任务**:
1. 运行 EXPLAIN ANALYZE 并识别瓶颈
2. 创建必要的索引
3. 考虑查询重写（CTE、子查询、物化视图）
4. 测量前后改进
5. 记录你的解决方案

**目标**: < 100ms 执行时间

### 挑战 2：设计分区策略

**问题**: `training_runs` 表有 5000 万行，每天增长 10 万行

**任务**:
1. 设计月度分区策略
2. 编写迁移脚本将现有表转换为分区表
3. 创建自动创建未来分区的函数
4. 使用 EXPLAIN 测试分区裁剪
5. 记录保留策略（保留 12 个月，归档更早的）

**交付物**: `sql/60_partition_training_runs.sql`

### 挑战 3：实现查询结果缓存

**问题**: 仪表盘统计查询每小时运行 1000 次，但数据每 5 分钟才变化一次

**任务**:
1. 为仪表盘统计创建物化视图
2. 设置刷新任务（pg_cron 或应用级）
3. 修改应用程序以查询物化视图
4. 测量缓存命中率
5. 比较与直接查询的性能

**目标**: 99% 缓存命中率，<5ms 查询时间

---

## 第 8 部分：总结与后续步骤

### 学到的内容

✅ **数据库分析**：使用 pg_stat_statements 和 EXPLAIN ANALYZE
✅ **索引策略设计**：针对不同查询模式
✅ **性能测量**：前后对比指标
✅ **维护程序**（VACUUM、ANALYZE、REINDEX）
✅ **监控设置**：使用 Prometheus 和 Grafana
✅ **生产 Runbook**：性能问题处理
✅ **高级技术**：分区、连接池、复制

### 关键要点

1. **先测量再优化**：使用 EXPLAIN ANALYZE 找到实际瓶颈
2. **索引很强大但非免费**：50% 写入开销换 10-100 倍读取加速
3. **外键索引至关重要**：始终对 FK 列建索引
4. **复合索引优于多个单列索引**：列顺序很重要
5. **Partial Index 被低估**：用过滤索引瞄准热查询
6. **监控防止退化**：在性能下降前设置告警
7. **维护必不可少**：VACUUM、ANALYZE 和 REINDEX 保持数据库健康

### 性能成果

| 指标 | 优化前 | 优化后 | 改进 |
|--------|--------|-------|-------------|
| 平均查询延迟 | 145ms | 12ms | **快 92%** |
| P99 延迟 | 1,250ms | 45ms | **快 96%** |
| 缓存命中率 | 45% | 94% | **+49pp** |
| 慢查询（>1s） | 7.3% | 0% | **100% 消除** |
| 数据库 CPU | 78% | 23% | **-55pp** |

### 生产就绪检查清单

部署到生产前：

- [ ] 所有索引用 `CONCURRENTLY` 创建
- [ ] EXPLAIN ANALYZE 确认索引使用
- [ ] 前后指标已记录
- [ ] 监控仪表盘已配置
- [ ] 告警规则已配置并测试
- [ ] Runbook 已记录并审查
- [ ] 维护计划已建立
- [ ] 备份/恢复已测试
- [ ] 性能回归测试已添加到 CI/CD
- [ ] 团队已接受故障处理程序培训

### 后续步骤

**模块 009：监控与日志基础**
- 在应用代码中仪表化数据库查询
- 设置分布式追踪（OpenTelemetry）
- 为业务 KPI 创建自定义指标
- 实现日志聚合（ELK、Loki）

**模块 010：云平台**
- 部署到托管 PostgreSQL（AWS RDS、GCP Cloud SQL）
- 配置自动备份和时间点恢复
- 设置跨区域复制
- 优化成本（预留实例、存储分层）

**实际项目**：
- 使用优化数据库后端构建完整 ML 平台
- 基于查询负载实现自动扩展
- 创建多租户数据库架构

### 资源

**书籍**：
- "PostgreSQL 14 Internals" by Egor Rogov
- "The Art of PostgreSQL" by Dimitri Fontaine

**工具**：
- [pgAnalyze](https://pganalyze.com/) - 查询性能监控
- [pghero](https://github.com/ankane/pghero) - 数据库健康仪表盘
- [pgBadger](https://github.com/darold/pgbadger) - 日志分析器
- [EXPLAIN Visualizer](https://explain.depesz.com/) - 可视化查询计划

**文档**：
- [PostgreSQL Performance Tips](https://www.postgresql.org/docs/current/performance-tips.html)
- [Index Types](https://www.postgresql.org/docs/current/indexes-types.html)
- [EXPLAIN Documentation](https://www.postgresql.org/docs/current/sql-explain.html)

---

## 恭喜！

你已完成练习 05，掌握了数据库性能优化！你现在拥有：

- 生产级索引策略
- 全面的监控设置
- 运维性能 Runbook
- 前后指标，证明 92% 改进
- 超越当前工作负载的高级技术

**你已准备好管理生产 ML 基础设施数据库！**

---

**练习 05 完成** | [返回模块 008 概述](../README.md) | 下一步：[模块 009：监控基础](../../mod-009-monitoring-basics/README.md)
