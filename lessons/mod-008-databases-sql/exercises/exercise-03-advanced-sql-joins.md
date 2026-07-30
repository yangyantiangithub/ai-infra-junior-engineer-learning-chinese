# Exercise 03: 高级 SQL 连接与分析查询

## 练习概述

**目标**：掌握所有连接类型的高级 SQL 查询模式，包括窗口函数、CTE 和复杂聚合。为 ML 基础设施仪表盘和报告编写生产级分析查询。

**难度**：中级 → 高级
**预计时间**：3-4 小时
**前置条件**：
- 已完成练习 01（SQL 基础与 CRUD）
- 已完成练习 02（数据库设计）并加载 schema
- 讲座 03（高级 SQL 与查询优化）
- 理解集合论（INNER vs OUTER join）

**你将学到**：
- 所有 JOIN 类型（INNER、LEFT、RIGHT、FULL OUTER、CROSS、SELF）
- 窗口函数（ROW_NUMBER、RANK、LAG、LEAD、SUM OVER）
- 公用表表达式（CTE）多步骤使用
- 子查询（相关和非相关）
- 条件聚合（FILTER、CASE WHEN）
- 用于复杂场景的 LATERAL 连接
- 使用 EXPLAIN 进行查询性能分析
- 用于缓存昂贵查询的物化视图

---

## 真实场景

你是**高级 ML 基础设施工程师**，负责构建支撑以下功能的分析：

1. **高管仪表盘**：模型部署健康度、成功率、成本
2. **合规报告**：审批工作流、审计追踪、SLA 跟踪
3. **运维指标**：资源利用率、故障分析、性能趋势
4. **数据科学洞察**：模型性能比较、数据集使用模式

这些查询将由以下人员使用：
- **高管**：月度业务评审
- **合规团队**：监管审计
- **MLOps 团队**：每日站会和事件响应
- **数据科学家**：模型选择与优化

---

## 第 1 部分：理解 JOIN

### 步骤 1.1：JOIN 类型概览

**可视化参考**：
```
表 A (models)        表 B (model_versions)
┌─────────┐         ┌─────────┐
│ A only  │         │ B only  │
│    ┌────┼─────────┼────┐    │
│    │ A∩B│         │ A∩B│    │
│    └────┼─────────┼────┘    │
└─────────┘         └─────────┘

INNER JOIN:  A ∩ B  (仅交集)
LEFT JOIN:   A       (所有 A + 匹配的 B)
RIGHT JOIN:       B  (所有 B + 匹配的 A)
FULL OUTER:  A ∪ B  (两者中的所有内容)
CROSS JOIN:  A × B  (笛卡尔积)
```

### 步骤 1.2：设置示例数据

```sql
-- ============================================
-- Exercise 03 的额外示例数据
-- ============================================
-- File: sql/20_sample_data_advanced.sql

-- 添加更多模型版本
DO $$
DECLARE
    model_id_var UUID;
BEGIN
    -- 获取情感分类器模型 ID
    SELECT model_id INTO model_id_var
    FROM models WHERE model_name = 'sentiment-classifier';

    INSERT INTO model_versions (model_id, semver, artifact_uri, framework, registered_by, status) VALUES
        (model_id_var, '1.2.0', 's3://ml-artifacts/sentiment-classifier/1.2.0/', 'pytorch', 'alice@example.com', 'registered'),
        (model_id_var, '1.3.0', 's3://ml-artifacts/sentiment-classifier/1.3.0/', 'pytorch', 'alice@example.com', 'validated'),
        (model_id_var, '2.0.0', 's3://ml-artifacts/sentiment-classifier/2.0.0/', 'pytorch', 'bob@example.com', 'deployed');
END $$;

-- 添加训练运行
INSERT INTO training_runs (version_id, run_name, status, accuracy, precision_score, recall_score, f1_score, gpu_hours, created_at)
SELECT
    mv.version_id,
    'run-' || mv.semver,
    CASE WHEN RANDOM() < 0.8 THEN 'succeeded' ELSE 'failed' END,
    0.8 + RANDOM() * 0.15,
    0.75 + RANDOM() * 0.2,
    0.75 + RANDOM() * 0.2,
    0.75 + RANDOM() * 0.2,
    RANDOM() * 20,
    NOW() - (RANDOM() * INTERVAL '30 days')
FROM model_versions mv
WHERE mv.framework = 'pytorch'
LIMIT 10;

-- 添加部署
INSERT INTO deployments (version_id, environment_id, deployed_by, status, deployed_at)
SELECT
    mv.version_id,
    e.environment_id,
    'ops@example.com',
    'active',
    NOW() - (RANDOM() * INTERVAL '60 days')
FROM model_versions mv
CROSS JOIN environments e
WHERE e.environment_name IN ('staging', 'prod')
  AND mv.status = 'deployed'
LIMIT 5;

-- 验证数据
SELECT 'Data loaded' AS status;
SELECT COUNT(*) AS total_models FROM models;
SELECT COUNT(*) AS total_versions FROM model_versions;
SELECT COUNT(*) AS total_runs FROM training_runs;
SELECT COUNT(*) AS total_deployments FROM deployments;
```

✅ **检查点**：包含模型、版本、训练运行和部署的示例数据已加载。

---

## 第 2 部分：INNER JOIN - 基础

### 步骤 2.1：基本 INNER JOIN

```sql
-- ==========================================
-- INNER JOIN 示例
-- ==========================================

-- 查询 1：包含版本的模型（INNER JOIN）
-- 仅返回有版本的模型
SELECT
    m.model_name,
    m.display_name,
    mv.semver,
    mv.status AS version_status,
    mv.registered_at
FROM models m
INNER JOIN model_versions mv ON m.model_id = mv.model_id
ORDER BY m.model_name, mv.semver DESC;
```

**关键点**：INNER JOIN 排除了没有版本的模型！

### 步骤 2.2：多表 INNER JOIN

```sql
-- 查询 2：完整模型谱系（models → versions → training runs）
SELECT
    m.model_name,
    mv.semver,
    tr.run_name,
    tr.status AS run_status,
    tr.accuracy,
    tr.gpu_hours,
    tr.created_at
FROM models m
INNER JOIN model_versions mv ON m.model_id = mv.model_id
INNER JOIN training_runs tr ON mv.version_id = tr.version_id
WHERE tr.status = 'succeeded'
ORDER BY m.model_name, mv.semver DESC, tr.accuracy DESC;
```

### 步骤 2.3：带聚合的 INNER JOIN

```sql
-- 查询 3：统计每个模型的训练运行数
SELECT
    m.model_name,
    m.display_name,
    COUNT(tr.run_id) AS total_runs,
    COUNT(*) FILTER (WHERE tr.status = 'succeeded') AS successful_runs,
    ROUND(AVG(tr.accuracy), 4) AS avg_accuracy,
    SUM(tr.gpu_hours) AS total_gpu_hours
FROM models m
INNER JOIN model_versions mv ON m.model_id = mv.model_id
INNER JOIN training_runs tr ON mv.version_id = tr.version_id
GROUP BY m.model_id, m.model_name, m.display_name
HAVING COUNT(tr.run_id) > 0
ORDER BY total_gpu_hours DESC;
```

✅ **检查点**：你理解 INNER JOIN 仅返回两个表中都匹配的行。

---

## 第 3 部分：LEFT JOIN - 包含左表所有内容

### 步骤 3.1：基本 LEFT JOIN

```sql
-- ==========================================
-- LEFT JOIN（LEFT OUTER JOIN）示例
-- ==========================================

-- 查询 4：所有模型及其版本（如果有）
-- 即使没有版本也返回模型
SELECT
    m.model_name,
    m.display_name,
    m.created_at,
    mv.semver,
    mv.status AS version_status
FROM models m
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
ORDER BY m.model_name, mv.semver DESC NULLS LAST;
```

**关键点**：LEFT JOIN 保留左表（models）的所有行，为缺失的匹配添加 NULL！

### 步骤 3.2：查找缺失关系

```sql
-- 查询 5：查找没有任何版本的模型（质量检查）
SELECT
    m.model_name,
    m.display_name,
    m.created_at,
    m.created_by,
    COUNT(mv.version_id) AS version_count
FROM models m
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
GROUP BY m.model_id, m.model_name, m.display_name, m.created_at, m.created_by
HAVING COUNT(mv.version_id) = 0
ORDER BY m.created_at DESC;
```

### 步骤 3.3：带 COALESCE 的 LEFT JOIN

```sql
-- 查询 6：所有模型及其版本数（无则为 0）
SELECT
    m.model_name,
    m.display_name,
    m.risk_level,
    COALESCE(COUNT(mv.version_id), 0) AS version_count,
    COALESCE(MAX(mv.registered_at), m.created_at) AS last_activity
FROM models m
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
GROUP BY m.model_id, m.model_name, m.display_name, m.risk_level, m.created_at
ORDER BY version_count DESC, last_activity DESC;
```

### 步骤 3.4：多级 LEFT JOIN

```sql
-- 查询 7：即使版本/训练运行不存在也显示完整模型信息
SELECT
    m.model_name,
    m.display_name,
    t.team_name,
    mv.semver,
    tr.run_name,
    tr.accuracy,
    d.dataset_name
FROM models m
LEFT JOIN teams t ON m.team_id = t.team_id
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
LEFT JOIN training_runs tr ON mv.version_id = tr.version_id
LEFT JOIN run_datasets rd ON tr.run_id = rd.run_id
LEFT JOIN datasets d ON rd.dataset_id = d.dataset_id
WHERE m.model_name = 'sentiment-classifier'
ORDER BY mv.semver DESC NULLS LAST, tr.created_at DESC NULLS LAST;
```

✅ **检查点**：你理解 LEFT JOIN 保留所有左表行，缺失匹配处为 NULL。

---

## 第 4 部分：RIGHT JOIN 和 FULL OUTER JOIN

### 步骤 4.1：RIGHT JOIN（不常用）

```sql
-- ==========================================
-- RIGHT JOIN 示例
-- ==========================================

-- 查询 8：所有版本及其模型（如果模型存在）
-- RIGHT JOIN 是 LEFT JOIN 的反向
SELECT
    m.model_name,
    m.display_name,
    mv.semver,
    mv.status,
    mv.registered_at
FROM models m
RIGHT JOIN model_versions mv ON m.model_id = mv.model_id
ORDER BY mv.registered_at DESC;
```

**最佳实践**：大多数开发者偏好 LEFT JOIN 并重新排列表，而非使用 RIGHT JOIN。

### 步骤 4.2：FULL OUTER JOIN

```sql
-- ==========================================
-- FULL OUTER JOIN 示例
-- ==========================================

-- 查询 9：所有模型和所有标签（无论是否链接）
SELECT
    m.model_name,
    t.tag_category,
    t.tag_value,
    CASE
        WHEN m.model_id IS NULL THEN '未使用的标签'
        WHEN t.tag_id IS NULL THEN '未标记的模型'
        ELSE '已标记'
    END AS relationship_status
FROM models m
FULL OUTER JOIN model_tags mt ON m.model_id = mt.model_id
FULL OUTER JOIN tags t ON mt.tag_id = t.tag_id
ORDER BY relationship_status, m.model_name NULLS LAST;
```

### 步骤 4.3：实用的 FULL OUTER JOIN

```sql
-- 查询 10：协调预期与实际部署
WITH expected_deployments AS (
    SELECT
        mv.version_id,
        e.environment_id,
        mv.semver,
        e.environment_name
    FROM model_versions mv
    CROSS JOIN environments e
    WHERE mv.status = 'deployed'
      AND e.environment_type IN ('staging', 'production')
),
actual_deployments AS (
    SELECT
        version_id,
        environment_id,
        deployed_at,
        status
    FROM deployments
    WHERE status = 'active'
)
SELECT
    ex.semver,
    ex.environment_name,
    CASE
        WHEN act.version_id IS NULL THEN '缺失 - 预期部署但未执行'
        WHEN ex.version_id IS NULL THEN '额外 - 已部署但未预期'
        ELSE '正常 - 按预期部署'
    END AS deployment_status,
    act.deployed_at,
    act.status
FROM expected_deployments ex
FULL OUTER JOIN actual_deployments act
    ON ex.version_id = act.version_id
    AND ex.environment_id = act.environment_id
ORDER BY deployment_status, ex.semver;
```

✅ **检查点**：你理解 FULL OUTER JOIN 返回两个表中的所有行。

---

## 第 5 部分：CROSS JOIN 和 SELF JOIN

### 步骤 5.1：CROSS JOIN（笛卡尔积）

```sql
-- ==========================================
-- CROSS JOIN 示例
-- ==========================================

-- 查询 11：所有可能的模型-环境组合
SELECT
    m.model_name,
    e.environment_name,
    e.environment_type
FROM models m
CROSS JOIN environments e
WHERE e.environment_type IN ('staging', 'production')
ORDER BY m.model_name, e.priority;
```

### 步骤 5.2：带条件的 CROSS JOIN

```sql
-- 查询 12：识别缺失的部署
SELECT
    m.model_name,
    mv.semver,
    e.environment_name,
    '缺失部署' AS status
FROM models m
CROSS JOIN model_versions mv
CROSS JOIN environments e
WHERE mv.model_id = m.model_id
  AND mv.status = 'deployed'
  AND e.environment_name = 'prod'
  AND NOT EXISTS (
      SELECT 1
      FROM deployments d
      WHERE d.version_id = mv.version_id
        AND d.environment_id = e.environment_id
        AND d.status = 'active'
  )
ORDER BY m.model_name, mv.semver;
```

### 步骤 5.3：SELF JOIN

```sql
-- ==========================================
-- SELF JOIN 示例
-- ==========================================

-- 查询 13：比较模型版本（当前 vs 前一个）
SELECT
    m.model_name,
    curr.semver AS current_version,
    curr.registered_at AS current_registered,
    prev.semver AS previous_version,
    prev.registered_at AS previous_registered,
    EXTRACT(DAYS FROM (curr.registered_at - prev.registered_at)) AS days_between
FROM models m
INNER JOIN model_versions curr ON m.model_id = curr.model_id
LEFT JOIN model_versions prev ON m.model_id = prev.model_id
    AND prev.registered_at < curr.registered_at
    AND NOT EXISTS (
        SELECT 1 FROM model_versions mid
        WHERE mid.model_id = m.model_id
          AND mid.registered_at > prev.registered_at
          AND mid.registered_at < curr.registered_at
    )
WHERE m.model_name = 'sentiment-classifier'
ORDER BY curr.registered_at DESC;
```

### 步骤 5.4：使用 SELF JOIN 查找重复项

```sql
-- 查询 14：查找潜在的重复模型（相似名称）
SELECT DISTINCT
    m1.model_name AS model_1,
    m2.model_name AS model_2,
    m1.objective AS objective_1,
    m2.objective AS objective_2
FROM models m1
INNER JOIN models m2 ON m1.model_id < m2.model_id  -- 避免重复对
WHERE SIMILARITY(m1.model_name, m2.model_name) > 0.6  -- PostgreSQL pg_trgm 扩展
   OR m1.objective = m2.objective;
```

✅ **检查点**：你理解 CROSS JOIN（所有组合）和 SELF JOIN（表与自身连接）。

---

## 第 6 部分：窗口函数

### 步骤 6.1：ROW_NUMBER 和 RANK

```sql
-- ==========================================
-- 窗口函数
-- ==========================================

-- 查询 15：按框架内准确率排名模型
SELECT
    m.model_name,
    mv.framework,
    tr.accuracy,
    ROW_NUMBER() OVER (PARTITION BY mv.framework ORDER BY tr.accuracy DESC) AS row_num,
    RANK() OVER (PARTITION BY mv.framework ORDER BY tr.accuracy DESC) AS rank,
    DENSE_RANK() OVER (PARTITION BY mv.framework ORDER BY tr.accuracy DESC) AS dense_rank
FROM models m
INNER JOIN model_versions mv ON m.model_id = mv.model_id
INNER JOIN training_runs tr ON mv.version_id = tr.version_id
WHERE tr.status = 'succeeded'
  AND tr.accuracy IS NOT NULL
ORDER BY mv.framework, tr.accuracy DESC;
```

### 步骤 6.2：获取每组最新记录

```sql
-- 查询 16：每个模型的最新部署（最常见模式！）
SELECT
    model_name,
    semver,
    environment_name,
    deployed_at,
    deployed_by,
    status
FROM (
    SELECT
        m.model_name,
        mv.semver,
        e.environment_name,
        d.deployed_at,
        d.deployed_by,
        d.status,
        ROW_NUMBER() OVER (
            PARTITION BY m.model_id, e.environment_id
            ORDER BY d.deployed_at DESC
        ) AS rn
    FROM models m
    INNER JOIN model_versions mv ON m.model_id = mv.model_id
    INNER JOIN deployments d ON mv.version_id = d.version_id
    INNER JOIN environments e ON d.environment_id = e.environment_id
) ranked
WHERE rn = 1
ORDER BY model_name, environment_name;
```

### 步骤 6.3：LAG 和 LEAD

```sql
-- 查询 17：比较与前一次训练运行的准确率
SELECT
    m.model_name,
    mv.semver,
    tr.run_name,
    tr.accuracy AS current_accuracy,
    LAG(tr.accuracy) OVER (
        PARTITION BY m.model_id
        ORDER BY tr.created_at
    ) AS previous_accuracy,
    tr.accuracy - LAG(tr.accuracy) OVER (
        PARTITION BY m.model_id
        ORDER BY tr.created_at
    ) AS accuracy_delta,
    CASE
        WHEN tr.accuracy > LAG(tr.accuracy) OVER (PARTITION BY m.model_id ORDER BY tr.created_at)
        THEN '↑ 提升'
        WHEN tr.accuracy < LAG(tr.accuracy) OVER (PARTITION BY m.model_id ORDER BY tr.created_at)
        THEN '↓ 下降'
        ELSE '→ 持平'
    END AS trend
FROM models m
INNER JOIN model_versions mv ON m.model_id = mv.model_id
INNER JOIN training_runs tr ON mv.version_id = tr.version_id
WHERE tr.status = 'succeeded'
  AND tr.accuracy IS NOT NULL
ORDER BY m.model_name, tr.created_at;
```

### 步骤 6.4：运行总计和移动平均

```sql
-- 查询 18：每个模型的 GPU 小时运行总计
SELECT
    m.model_name,
    tr.created_at::DATE AS run_date,
    tr.gpu_hours,
    SUM(tr.gpu_hours) OVER (
        PARTITION BY m.model_id
        ORDER BY tr.created_at
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_gpu_hours,
    AVG(tr.gpu_hours) OVER (
        PARTITION BY m.model_id
        ORDER BY tr.created_at
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7_runs
FROM models m
INNER JOIN model_versions mv ON m.model_id = mv.model_id
INNER JOIN training_runs tr ON mv.version_id = tr.version_id
WHERE tr.status = 'succeeded'
ORDER BY m.model_name, tr.created_at;
```

### 步骤 6.5：NTILE 用于百分位

```sql
-- 查询 19：模型准确率的四分位数分析
SELECT
    m.model_name,
    tr.accuracy,
    NTILE(4) OVER (ORDER BY tr.accuracy) AS accuracy_quartile,
    PERCENT_RANK() OVER (ORDER BY tr.accuracy) AS percentile_rank,
    CASE
        WHEN NTILE(4) OVER (ORDER BY tr.accuracy) = 4 THEN '前 25%'
        WHEN NTILE(4) OVER (ORDER BY tr.accuracy) = 3 THEN '高于平均'
        WHEN NTILE(4) OVER (ORDER BY tr.accuracy) = 2 THEN '低于平均'
        ELSE '后 25%'
    END AS performance_category
FROM models m
INNER JOIN model_versions mv ON m.model_id = mv.model_id
INNER JOIN training_runs tr ON mv.version_id = tr.version_id
WHERE tr.status = 'succeeded'
  AND tr.accuracy IS NOT NULL
ORDER BY tr.accuracy DESC;
```

✅ **检查点**：你可以使用窗口函数进行排名、比较和运行计算。

---

## 第 7 部分：公用表表达式（CTE）

### 步骤 7.1：简单 CTE

```sql
-- ==========================================
-- 公用表表达式（WITH 子句）
-- ==========================================

-- 查询 20：使用 CTE 进行多步骤分析
WITH production_models AS (
    -- 步骤 1：查找生产中的模型
    SELECT DISTINCT
        m.model_id,
        m.model_name,
        m.risk_level
    FROM models m
    INNER JOIN model_versions mv ON m.model_id = mv.model_id
    INNER JOIN deployments d ON mv.version_id = d.version_id
    INNER JOIN environments e ON d.environment_id = e.environment_id
    WHERE e.environment_type = 'production'
      AND d.status = 'active'
),
model_metrics AS (
    -- 步骤 2：计算这些模型的指标
    SELECT
        pm.model_id,
        pm.model_name,
        COUNT(tr.run_id) AS total_runs,
        AVG(tr.accuracy) AS avg_accuracy,
        MAX(tr.accuracy) AS best_accuracy,
        SUM(tr.gpu_hours) AS total_gpu_cost
    FROM production_models pm
    INNER JOIN model_versions mv ON pm.model_id = mv.model_id
    LEFT JOIN training_runs tr ON mv.version_id = tr.version_id
    WHERE tr.status = 'succeeded'
    GROUP BY pm.model_id, pm.model_name
)
-- 步骤 3：带衍生字段的最终输出
SELECT
    mm.model_name,
    mm.total_runs,
    ROUND(mm.avg_accuracy::numeric, 4) AS avg_accuracy,
    ROUND(mm.best_accuracy::numeric, 4) AS best_accuracy,
    mm.total_gpu_cost,
    CASE
        WHEN mm.total_runs < 10 THEN '测试不足'
        WHEN mm.avg_accuracy < 0.85 THEN '需要改进'
        ELSE '健康'
    END AS health_status
FROM model_metrics mm
ORDER BY mm.total_gpu_cost DESC;
```

### 步骤 7.2：多个 CTE 与 JOIN

```sql
-- 查询 21：部署就绪报告
WITH latest_versions AS (
    SELECT DISTINCT ON (m.model_id)
        m.model_id,
        m.model_name,
        m.compliance_required,
        mv.version_id,
        mv.semver,
        mv.status
    FROM models m
    INNER JOIN model_versions mv ON m.model_id = mv.model_id
    ORDER BY m.model_id, mv.registered_at DESC
),
approval_status AS (
    SELECT
        lv.model_id,
        lv.version_id,
        COUNT(*) FILTER (WHERE a.status = 'approved') AS approvals_granted,
        COUNT(*) FILTER (WHERE a.status = 'pending') AS approvals_pending,
        COUNT(*) FILTER (WHERE a.status = 'rejected') AS approvals_rejected
    FROM latest_versions lv
    LEFT JOIN approvals a ON lv.version_id = a.version_id
    GROUP BY lv.model_id, lv.version_id
),
deployment_status AS (
    SELECT
        lv.model_id,
        COUNT(*) FILTER (WHERE d.status = 'active' AND e.environment_type = 'production') AS prod_deployments
    FROM latest_versions lv
    LEFT JOIN deployments d ON lv.version_id = d.version_id
    LEFT JOIN environments e ON d.environment_id = e.environment_id
    GROUP BY lv.model_id
)
SELECT
    lv.model_name,
    lv.semver,
    lv.status AS version_status,
    lv.compliance_required,
    COALESCE(ast.approvals_granted, 0) AS approvals_granted,
    COALESCE(ast.approvals_pending, 0) AS approvals_pending,
    COALESCE(dst.prod_deployments, 0) AS prod_deployments,
    CASE
        WHEN lv.compliance_required AND COALESCE(ast.approvals_granted, 0) = 0
            THEN '⚠️ 阻塞 - 需要审批'
        WHEN COALESCE(ast.approvals_rejected, 0) > 0
            THEN '❌ 阻塞 - 审批被拒'
        WHEN lv.status != 'validated'
            THEN '⏳ 未就绪 - 需要验证'
        WHEN COALESCE(dst.prod_deployments, 0) > 0
            THEN '✅ 已部署到生产'
        ELSE '✅ 可部署'
    END AS readiness_status
FROM latest_versions lv
LEFT JOIN approval_status ast ON lv.model_id = ast.model_id
LEFT JOIN deployment_status dst ON lv.model_id = dst.model_id
ORDER BY readiness_status, lv.model_name;
```

### 步骤 7.3：递归 CTE（高级）

```sql
-- 查询 22：模型版本历史树（如果版本引用父版本）
-- 注意：这需要在 model_versions 中有 parent_version_id 列

-- 递归 CTE 示例：
WITH RECURSIVE version_lineage AS (
    -- 基础情况：根版本（无父版本）
    SELECT
        version_id,
        model_id,
        semver,
        parent_version_id,
        1 AS depth,
        semver::TEXT AS lineage_path
    FROM model_versions
    WHERE parent_version_id IS NULL

    UNION ALL

    -- 递归情况：子版本
    SELECT
        mv.version_id,
        mv.model_id,
        mv.semver,
        mv.parent_version_id,
        vl.depth + 1,
        vl.lineage_path || ' → ' || mv.semver
    FROM model_versions mv
    INNER JOIN version_lineage vl ON mv.parent_version_id = vl.version_id
    WHERE vl.depth < 10  -- 防止无限递归
)
SELECT
    m.model_name,
    vl.semver,
    vl.depth,
    vl.lineage_path
FROM version_lineage vl
INNER JOIN models m ON vl.model_id = m.model_id
ORDER BY m.model_name, vl.depth, vl.semver;
```

✅ **检查点**：你可以使用 CTE 将复杂查询分解为可读的步骤。

---

## 第 8 部分：子查询

### 步骤 8.1：WHERE 子句中的子查询

```sql
-- ==========================================
-- 子查询
-- ==========================================

-- 查询 23：准确率高于平均的模型
SELECT
    m.model_name,
    AVG(tr.accuracy) AS avg_accuracy
FROM models m
INNER JOIN model_versions mv ON m.model_id = mv.model_id
INNER JOIN training_runs tr ON mv.version_id = tr.version_id
WHERE tr.status = 'succeeded'
  AND tr.accuracy IS NOT NULL
GROUP BY m.model_id, m.model_name
HAVING AVG(tr.accuracy) > (
    -- 子查询：计算全局平均值
    SELECT AVG(accuracy)
    FROM training_runs
    WHERE status = 'succeeded'
      AND accuracy IS NOT NULL
)
ORDER BY avg_accuracy DESC;
```

### 步骤 8.2：相关子查询

```sql
-- 查询 24：模型及其最佳训练运行
SELECT
    m.model_name,
    m.display_name,
    (
        SELECT MAX(tr.accuracy)
        FROM model_versions mv
        INNER JOIN training_runs tr ON mv.version_id = tr.version_id
        WHERE mv.model_id = m.model_id
          AND tr.status = 'succeeded'
    ) AS best_accuracy,
    (
        SELECT COUNT(*)
        FROM model_versions mv
        WHERE mv.model_id = m.model_id
    ) AS version_count
FROM models m
ORDER BY best_accuracy DESC NULLS LAST;
```

### 步骤 8.3：EXISTS 和 NOT EXISTS

```sql
-- 查询 25：至少有一个失败训练运行的模型
SELECT
    m.model_name,
    m.display_name,
    m.risk_level
FROM models m
WHERE EXISTS (
    SELECT 1
    FROM model_versions mv
    INNER JOIN training_runs tr ON mv.version_id = tr.version_id
    WHERE mv.model_id = m.model_id
      AND tr.status = 'failed'
)
ORDER BY m.risk_level DESC, m.model_name;

-- 查询 26：没有成功部署的模型
SELECT
    m.model_name,
    m.display_name,
    m.created_at
FROM models m
WHERE NOT EXISTS (
    SELECT 1
    FROM model_versions mv
    INNER JOIN deployments d ON mv.version_id = d.version_id
    WHERE mv.model_id = m.model_id
      AND d.status = 'active'
)
ORDER BY m.created_at;
```

### 步骤 8.4：FROM 中的子查询（派生表）

```sql
-- 查询 27：按多项指标排名模型
SELECT
    metrics.model_name,
    metrics.avg_accuracy,
    metrics.total_runs,
    metrics.total_cost,
    NTILE(3) OVER (ORDER BY metrics.avg_accuracy DESC) AS accuracy_tier,
    NTILE(3) OVER (ORDER BY metrics.total_cost ASC) AS cost_efficiency_tier
FROM (
    SELECT
        m.model_name,
        AVG(tr.accuracy) AS avg_accuracy,
        COUNT(tr.run_id) AS total_runs,
        SUM(tr.gpu_hours) * 2.5 AS total_cost  -- $2.50/GPU 小时
    FROM models m
    INNER JOIN model_versions mv ON m.model_id = mv.model_id
    INNER JOIN training_runs tr ON mv.version_id = tr.version_id
    WHERE tr.status = 'succeeded'
    GROUP BY m.model_id, m.model_name
    HAVING COUNT(tr.run_id) >= 3
) AS metrics
ORDER BY metrics.avg_accuracy DESC;
```

✅ **检查点**：你可以在 WHERE、SELECT、FROM 以及使用 EXISTS 的子查询中使用子查询。

---

## 第 9 部分：LATERAL 连接（高级）

### 步骤 9.1：带函数调用的 LATERAL

```sql
-- ==========================================
-- LATERAL 连接
-- ==========================================

-- 查询 28：每个模型的前 3 次训练运行
SELECT
    m.model_name,
    top_runs.run_name,
    top_runs.accuracy,
    top_runs.created_at
FROM models m
CROSS JOIN LATERAL (
    SELECT
        tr.run_name,
        tr.accuracy,
        tr.created_at
    FROM model_versions mv
    INNER JOIN training_runs tr ON mv.version_id = tr.version_id
    WHERE mv.model_id = m.model_id
      AND tr.status = 'succeeded'
      AND tr.accuracy IS NOT NULL
    ORDER BY tr.accuracy DESC
    LIMIT 3
) AS top_runs
ORDER BY m.model_name, top_runs.accuracy DESC;
```

### 步骤 9.2：用于最新记录模式的 LATERAL

```sql
-- 查询 29：模型及其最新部署（使用 LATERAL）
SELECT
    m.model_name,
    m.display_name,
    latest_deploy.environment_name,
    latest_deploy.deployed_at,
    latest_deploy.status
FROM models m
CROSS JOIN LATERAL (
    SELECT
        e.environment_name,
        d.deployed_at,
        d.status
    FROM model_versions mv
    INNER JOIN deployments d ON mv.version_id = d.version_id
    INNER JOIN environments e ON d.environment_id = e.environment_id
    WHERE mv.model_id = m.model_id
      AND e.environment_type = 'production'
    ORDER BY d.deployed_at DESC
    LIMIT 1
) AS latest_deploy
ORDER BY m.model_name;
```

✅ **检查点**：你理解用于行级子查询的 LATERAL 连接。

---

## 第 10 部分：使用 EXPLAIN 进行性能分析

### 步骤 10.1：理解 EXPLAIN

```sql
-- ==========================================
-- 查询性能分析
-- ==========================================

-- 查询 30：基本 EXPLAIN
EXPLAIN
SELECT m.model_name, COUNT(mv.version_id)
FROM models m
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
GROUP BY m.model_id, m.model_name;
```

### 步骤 10.2：EXPLAIN ANALYZE（实际执行）

```sql
-- 查询 31：EXPLAIN ANALYZE 用于实际运行时间
EXPLAIN ANALYZE
SELECT
    m.model_name,
    COUNT(tr.run_id) AS total_runs,
    AVG(tr.accuracy) AS avg_accuracy
FROM models m
INNER JOIN model_versions mv ON m.model_id = mv.model_id
INNER JOIN training_runs tr ON mv.version_id = tr.version_id
WHERE tr.status = 'succeeded'
GROUP BY m.model_id, m.model_name
ORDER BY avg_accuracy DESC;
```

### 步骤 10.3：性能比较

```sql
-- 查询 32：比较有索引和无索引的查询

-- 无 training_runs.status 索引
EXPLAIN ANALYZE
SELECT * FROM training_runs WHERE status = 'failed';

-- 创建索引
CREATE INDEX idx_training_runs_status_temp ON training_runs(status);

-- 有索引
EXPLAIN ANALYZE
SELECT * FROM training_runs WHERE status = 'failed';

-- 检查性能差异
\timing on
SELECT COUNT(*) FROM training_runs WHERE status = 'failed';
```

✅ **检查点**：你可以使用 EXPLAIN ANALYZE 理解查询性能。

---

## 第 11 部分：物化视图

### 步骤 11.1：创建物化视图

```sql
-- ==========================================
-- 物化视图
-- ==========================================

-- 查询 33：为昂贵查询创建物化视图
CREATE MATERIALIZED VIEW mv_model_health_summary AS
SELECT
    m.model_id,
    m.model_name,
    m.display_name,
    m.risk_level,
    COUNT(DISTINCT mv.version_id) AS total_versions,
    COUNT(DISTINCT tr.run_id) AS total_runs,
    COUNT(DISTINCT tr.run_id) FILTER (WHERE tr.status = 'succeeded') AS successful_runs,
    ROUND(AVG(tr.accuracy) FILTER (WHERE tr.status = 'succeeded')::numeric, 4) AS avg_accuracy,
    SUM(tr.gpu_hours) AS total_gpu_hours,
    MAX(tr.created_at) AS last_run_date,
    COUNT(DISTINCT d.deployment_id) FILTER (
        WHERE d.status = 'active'
    ) AS active_deployments
FROM models m
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
LEFT JOIN training_runs tr ON mv.version_id = tr.version_id
LEFT JOIN deployments d ON mv.version_id = d.version_id
GROUP BY m.model_id, m.model_name, m.display_name, m.risk_level;

-- 在物化视图上创建索引
CREATE INDEX idx_mv_model_health_model_name ON mv_model_health_summary(model_name);

-- 查询物化视图（快速！）
SELECT * FROM mv_model_health_summary
WHERE risk_level = 'critical'
ORDER BY avg_accuracy DESC NULLS LAST;
```

### 步骤 11.2：刷新物化视图

```sql
-- 刷新物化视图（更新数据）
REFRESH MATERIALIZED VIEW mv_model_health_summary;

-- 不阻塞读取的刷新（PostgreSQL 9.4+）
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_model_health_summary;
```

### 步骤 11.3：自动刷新策略

```sql
-- 创建刷新物化视图的函数
CREATE OR REPLACE FUNCTION refresh_model_health_summary()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY mv_model_health_summary;
END;
$$ LANGUAGE plpgsql;
```

✅ **检查点**：你可以创建物化视图来缓存昂贵查询。

---

## 第 12 部分：动手挑战

### 挑战 1：高管仪表盘查询

编写一个提供以下信息的查询：
- 模型名称
- 总版本数
- 最新版本
- 生产部署状态（是/否）
- 所有成功运行的平均准确率
- GPU 总成本（小时数 × $2.50）
- 健康状态（基于准确率/部署的正常/警告/严重）

<details>
<summary>解决方案</summary>

```sql
WITH latest_versions AS (
    SELECT DISTINCT ON (model_id)
        model_id,
        semver AS latest_version
    FROM model_versions
    ORDER BY model_id, registered_at DESC
),
prod_status AS (
    SELECT DISTINCT
        mv.model_id,
        TRUE AS has_prod_deployment
    FROM model_versions mv
    INNER JOIN deployments d ON mv.version_id = d.version_id
    INNER JOIN environments e ON d.environment_id = e.environment_id
    WHERE e.environment_type = 'production' AND d.status = 'active'
)
SELECT
    m.model_name,
    COUNT(DISTINCT mv.version_id) AS total_versions,
    lv.latest_version,
    COALESCE(ps.has_prod_deployment, FALSE) AS in_production,
    ROUND(AVG(tr.accuracy) FILTER (WHERE tr.status = 'succeeded')::numeric, 4) AS avg_accuracy,
    ROUND((SUM(tr.gpu_hours) * 2.50)::numeric, 2) AS total_cost_usd,
    CASE
        WHEN AVG(tr.accuracy) FILTER (WHERE tr.status = 'succeeded') >= 0.90 AND ps.has_prod_deployment THEN '健康'
        WHEN AVG(tr.accuracy) FILTER (WHERE tr.status = 'succeeded') >= 0.80 THEN '警告'
        ELSE '严重'
    END AS health_status
FROM models m
LEFT JOIN model_versions mv ON m.model_id = mv.model_id
LEFT JOIN training_runs tr ON mv.version_id = tr.version_id
LEFT JOIN latest_versions lv ON m.model_id = lv.model_id
LEFT JOIN prod_status ps ON m.model_id = ps.model_id
GROUP BY m.model_id, m.model_name, lv.latest_version, ps.has_prod_deployment
ORDER BY health_status DESC, avg_accuracy DESC NULLS LAST;
```
</details>

### 挑战 2：合规审计报告

编写一个显示以下信息的查询：
- 所有生产部署
- 关联的审批（审批类型、审批人、日期）
- 缺少所需审批的模型
- SLA 合规性（审批后 5 天内部署）

### 挑战 3：资源优化查询

识别：
- 利用率低的模型（过去 30 天内运行次数 < 5）
- 高成本低准确率的模型（GPU 小时 > 50，准确率 < 0.80）
- 推荐操作（弃用、优化、重新训练）

---

## 第 13 部分：总结与交付物

### 学到的内容

✅ **JOIN 类型**：
- INNER JOIN（交集）
- LEFT/RIGHT JOIN（保留一侧）
- FULL OUTER JOIN（并集）
- CROSS JOIN（笛卡尔积）
- SELF JOIN（比较同一表中的行）

✅ **窗口函数**：
- ROW_NUMBER、RANK、DENSE_RANK
- LAG、LEAD（与前后行比较）
- 使用 SUM OVER 的运行总计
- 使用窗口帧的移动平均
- NTILE 用于百分位数

✅ **高级模式**：
- 用于可读多步骤查询的 CTE
- 子查询（相关和非相关）
- 用于高效存在性检查的 EXISTS
- 用于行级子查询的 LATERAL 连接

✅ **性能**：
- EXPLAIN 和 EXPLAIN ANALYZE
- 索引使用分析
- 用于缓存的物化视图

### 交付物清单

- [ ] `sql/30_advanced_queries.sql` - 所有示例查询
- [ ] 至少 3 个用于仪表盘的复杂分析查询
- [ ] 3 个以上查询的 EXPLAIN ANALYZE 输出
- [ ] 已创建 1 个物化视图
- [ ] 性能发现文档
- [ ] 反思总结（300-500 字）

### 反思问题

1. **复杂性**：哪种连接类型最具挑战性，为什么？
2. **窗口函数**：与子查询相比，它们如何简化问题？
3. **性能**：你最大的性能洞察是什么？
4. **生产环境**：你如何在生产中监控这些查询？
5. **测试**：你如何在 CI/CD 中测试这些查询？

---

## 后续步骤

- **练习 04**：SQLAlchemy ORM 集成 - 将 Python 应用程序连接到该 schema
- **练习 05**：优化与索引 - 深入查询性能

---

## 额外资源

- [PostgreSQL Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)
- [Understanding EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [SQL Joins Visualized](https://joins.spathon.com/)
- [PostgreSQL Performance Tips](https://wiki.postgresql.org/wiki/Performance_Optimization)

---

**练习完成！** 🎉

你已经掌握了高级 SQL 连接和分析查询。现在可以为 ML 基础设施仪表盘和报告编写复杂的生产级查询。

**预计时间**：3-4 小时
**难度**：中级 → 高级
**代码行数**：约 200 个 SQL 查询
**学习目标**：✅ 全部达成
**进阶准备**：练习 04 - SQLAlchemy ORM 集成
