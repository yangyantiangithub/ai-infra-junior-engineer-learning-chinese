# Great Expectations 配置 — 项目 03

Great Expectations 在模型训练运行*之前*验证训练数据集。检查点失败会导致 DVC `validate` 阶段以非零退出，从而阻止 `train` 在错误数据上运行。

## 目录说明

- `great_expectations.yml` — 项目级配置（数据源、存储、站点）
- `expectations/training_data_suite.json` — 训练数据的期望套件
- `checkpoints/training_data_checkpoint.yml` — 可运行的检查点，将期望套件与数据集配对

## 运行

从项目根目录执行：

```bash
# 运行验证
great_expectations checkpoint run training_data_checkpoint

# 在 Data Docs 中查看结果
great_expectations docs build
open great_expectations/uncommitted/data_docs/local_site/index.html
```

Airflow DAG `dags/ml_pipeline_dag.py` 在 `preprocessing` 和 `training` 任务之间调用相同的检查点，因此生产运行和本地运行使用相同的验证逻辑。

## 何时更新期望套件

- **模式变更**：当特征被添加、删除或重命名时。
- **范围变更**：当预期的 min/max/mean 因真实的业务变化（而非数据质量错误）而改变时。
- **漂移基线**：每月从最新的已知良好的生产训练集中刷新 KL-divergence 的 partition_object。

对期望套件的每次更改都需要提交 PR；不匹配的期望是导致误导性"数据没问题"结果的常见原因。

## 套件捕获的故障模式

| 期望 | 捕获的问题 |
|---|---|
| `expect_table_row_count_to_be_between` | 空提取；失控的 JOIN |
| `expect_table_columns_to_match_set` | 模式漂移，意外添加/删除列 |
| `expect_column_values_to_be_unique` (transaction_id) | 上游问题导致的重复行 |
| `expect_column_values_to_be_in_set` (label ∈ {0,1}) | 标签映射错误 |
| `expect_column_mean_to_be_between` (label) | 类别平衡崩溃（全 0 或全 1） |
| `expect_column_kl_divergence_to_be_less_than` (amount) | 指示上游管道变更的分布漂移 |
