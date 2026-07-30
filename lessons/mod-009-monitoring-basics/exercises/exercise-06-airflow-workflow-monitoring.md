# Exercise 06：Apache Airflow - 工作流编排与监控

**难度**：初级-中级
**时长**：3-4 小时
**前置要求**：Docker、Python 基础、监控基础知识

## 学习目标

完成本练习后，你将能够：

1. 理解什么是工作流编排以及为什么需要它
2. 安装和配置 Apache Airflow
3. 创建和运行 DAG（有向无环图）
4. 监控工作流执行和健康状况
5. 将 Airflow 与 Prometheus 集成以获取 metrics
6. 设置管道故障告警
7. 调试失败的工作流

## 你将构建的内容

一个完整的 Airflow 设置，包含：
- 在 Docker 中运行的 Airflow
- 示例 ML 数据管道 DAG
- Prometheus metrics 集成
- 用于工作流监控的 Grafana dashboard
- 针对失败任务的告警

## 背景：什么是 Apache Airflow？

### 问题：管理复杂工作流

**没有编排**：
```bash
# 手动工作流执行
python download_data.py
python preprocess_data.py
python train_model.py
python deploy_model.py

# 问题：
# - 如果步骤 2 失败怎么办？我们需要重新开始所有步骤吗？
# - 如何每天调度这个流程？
# - 如何处理重试？
# - 如何监控进度？
# - 如何管理依赖关系？
```

### 解决方案：Apache Airflow

**Airflow** 是一个工作流编排平台，让你能够：
- ✅ 将工作流定义为代码（Python）
- ✅ 调度自动化执行
- ✅ 管理任务间的依赖关系
- ✅ 监控任务执行
- ✅ 自动重试失败任务
- ✅ 查看日志和历史记录
- ✅ 失败时告警

### 关键概念

**DAG（有向无环图）**：
- 工作流定义
- 带有依赖关系的任务集合
- 不允许循环依赖

**任务**：
- 单个工作单元
- 可以是 Python 函数、bash 命令等
- 具有状态：success、failed、running、queued

**Operator**：
- 用于创建任务的模板
- PythonOperator、BashOperator 等

**Scheduler**：
- 按计划触发 DAG 运行
- 管理任务执行顺序

**Executor**：
- 运行任务（Local、Celery、Kubernetes 等）

**Webserver**：
- 用于监控和管理的 UI

## 第 1 部分：使用 Docker 设置 Airflow

### 步骤 1：创建项目目录

```bash
mkdir airflow-monitoring-lab
cd airflow-monitoring-lab
```

### 步骤 2：创建 Docker Compose 配置

创建 `docker-compose.yml`：

```yaml
version: '3.8'

x-airflow-common:
  &airflow-common
  image: apache/airflow:2.7.3
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth'
    AIRFLOW__METRICS__STATSD_ON: 'true'
    AIRFLOW__METRICS__STATSD_HOST: statsd-exporter
    AIRFLOW__METRICS__STATSD_PORT: 9125
  volumes:
    - ./dags:/opt/airflow/dags
    - ./logs:/opt/airflow/logs
    - ./plugins:/opt/airflow/plugins
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on:
    &airflow-common-depends-on
    postgres:
      condition: service_healthy

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 5s
      retries: 5
    restart: always

  airflow-webserver:
    <<: *airflow-common
    command: webserver
    ports:
      - 8080:8080
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost:8080/health"]
      interval: 10s
      timeout: 10s
      retries: 5
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-scheduler:
    <<: *airflow-common
    command: scheduler
    healthcheck:
      test: ["CMD-SHELL", 'airflow jobs check --job-type SchedulerJob --hostname "$${HOSTNAME}"']
      interval: 10s
      timeout: 10s
      retries: 5
    restart: always
    depends_on:
      <<: *airflow-common-depends-on
      airflow-init:
        condition: service_completed_successfully

  airflow-init:
    <<: *airflow-common
    entrypoint: /bin/bash
    command:
      - -c
      - |
        mkdir -p /sources/logs /sources/dags /sources/plugins
        chown -R "${AIRFLOW_UID:-50000}:0" /sources/{logs,dags,plugins}
        exec /entrypoint airflow version
    environment:
      <<: *airflow-common-env
      _AIRFLOW_DB_UPGRADE: 'true'
      _AIRFLOW_WWW_USER_CREATE: 'true'
      _AIRFLOW_WWW_USER_USERNAME: airflow
      _AIRFLOW_WWW_USER_PASSWORD: airflow
    user: "0:0"
    volumes:
      - .:/sources

  # StatsD Exporter for Prometheus
  statsd-exporter:
    image: prom/statsd-exporter:latest
    ports:
      - 9102:9102
      - 9125:9125/udp
    command:
      - '--statsd.mapping-config=/tmp/statsd_mapping.yml'
    volumes:
      - ./statsd_mapping.yml:/tmp/statsd_mapping.yml

  # Prometheus
  prometheus:
    image: prom/prometheus:latest
    ports:
      - 9090:9090
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'

  # Grafana
  grafana:
    image: grafana/grafana:latest
    ports:
      - 3000:3000
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana-data:/var/lib/grafana

volumes:
  postgres-db-volume:
  prometheus-data:
  grafana-data:
```

### 步骤 3：创建配置文件

创建 `statsd_mapping.yml`：

```yaml
mappings:
  - match: "airflow.dag_processing.*"
    name: "airflow_dag_processing"
    labels:
      job: "airflow"

  - match: "airflow.scheduler.*"
    name: "airflow_scheduler"
    labels:
      job: "airflow"

  - match: "airflow.executor.*"
    name: "airflow_executor"
    labels:
      job: "airflow"

  - match: "airflow.dagrun.*"
    name: "airflow_dagrun"
    labels:
      job: "airflow"

  - match: "airflow.task.*"
    name: "airflow_task"
    labels:
      job: "airflow"
```

创建 `prometheus.yml`：

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'airflow'
    static_configs:
      - targets: ['statsd-exporter:9102']
```

### 步骤 4：创建目录

```bash
mkdir -p dags logs plugins
```

### 步骤 5：启动 Airflow

```bash
# 设置 Airflow UID
echo -e "AIRFLOW_UID=$(id -u)" > .env

# 启动服务
docker-compose up -d

# 检查状态
docker-compose ps
```

**等待 1-2 分钟初始化**，然后访问：
- **Airflow UI**：http://localhost:8080（用户：airflow，密码：airflow）
- **Prometheus**：http://localhost:9090
- **Grafana**：http://localhost:3000（用户：admin，密码：admin）

## 第 2 部分：创建你的第一个 DAG

### 步骤 6：简单的数据管道 DAG

创建 `dags/ml_training_pipeline.py`：

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
import time
import random

# 所有任务的默认参数
default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=1),
}

def download_dataset(**context):
    """模拟下载训练数据集"""
    print("📥 Downloading dataset from S3...")
    time.sleep(3)
    print("✓ Downloaded 1.2GB of training data")

    # 通过 XCom 将数据传递给下一个任务
    return {"dataset_size_gb": 1.2, "num_samples": 10000}

def preprocess_data(**context):
    """模拟数据预处理"""
    # 获取前一个任务的数据
    ti = context['ti']
    dataset_info = ti.xcom_pull(task_ids='download_dataset')

    print(f"🔧 Preprocessing {dataset_info['num_samples']} samples...")
    time.sleep(4)
    print("✓ Data preprocessed: cleaned, normalized, augmented")

    return {"processed_samples": dataset_info['num_samples'], "validation_split": 0.2}

def train_model(**context):
    """模拟模型训练"""
    ti = context['ti']
    data_info = ti.xcom_pull(task_ids='preprocess_data')

    print(f"🤖 Training model on {data_info['processed_samples']} samples...")
    time.sleep(5)

    # 模拟 metrics
    accuracy = round(random.uniform(0.85, 0.95), 4)
    print(f"✓ Training complete! Validation accuracy: {accuracy}")

    return {"accuracy": accuracy, "model_size_mb": 450}

def evaluate_model(**context):
    """模拟模型评估"""
    ti = context['ti']
    model_info = ti.xcom_pull(task_ids='train_model')

    print(f"📊 Evaluating model (accuracy: {model_info['accuracy']})...")
    time.sleep(2)

    if model_info['accuracy'] < 0.88:
        raise ValueError(f"Model accuracy {model_info['accuracy']} below threshold 0.88")

    print("✓ Model passed evaluation!")
    return True

def deploy_model(**context):
    """模拟模型部署"""
    print("🚀 Deploying model to production...")
    time.sleep(3)
    print("✓ Model deployed successfully")
    return {"deployment_url": "https://api.example.com/model/v1"}

# 定义 DAG
with DAG(
    'ml_training_pipeline',
    default_args=default_args,
    description='ML model training and deployment pipeline',
    schedule_interval='0 2 * * *',  # 每天凌晨 2 点运行
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['ml', 'training', 'production'],
) as dag:

    # 任务 1：下载数据集
    download = PythonOperator(
        task_id='download_dataset',
        python_callable=download_dataset,
    )

    # 任务 2：预处理数据
    preprocess = PythonOperator(
        task_id='preprocess_data',
        python_callable=preprocess_data,
    )

    # 任务 3：训练模型
    train = PythonOperator(
        task_id='train_model',
        python_callable=train_model,
    )

    # 任务 4：评估模型
    evaluate = PythonOperator(
        task_id='evaluate_model',
        python_callable=evaluate_model,
    )

    # 任务 5：部署模型
    deploy = PythonOperator(
        task_id='deploy_model',
        python_callable=deploy_model,
    )

    # 任务 6：发送通知
    notify = BashOperator(
        task_id='send_notification',
        bash_command='echo "✅ ML pipeline completed successfully!"',
    )

    # 定义任务依赖关系
    download >> preprocess >> train >> evaluate >> deploy >> notify
```

**DAG 工作流**：
```
download_dataset → preprocess_data → train_model → evaluate_model → deploy_model → send_notification
```

### 步骤 7：验证 DAG

```bash
# 刷新 Airflow UI (http://localhost:8080)
# 你应该会看到 "ml_training_pipeline" DAG

# 或者通过 CLI 检查
docker-compose exec airflow-webserver airflow dags list
docker-compose exec airflow-webserver airflow dags show ml_training_pipeline
```

### 步骤 8：手动触发 DAG

**选项 1：通过 UI**：
1. 访问 http://localhost:8080
2. 找到 "ml_training_pipeline"
3. 点击右侧的播放按钮 ▶️
4. 点击 "Trigger DAG"

**选项 2：通过 CLI**：
```bash
docker-compose exec airflow-webserver airflow dags trigger ml_training_pipeline
```

**监控执行**：
1. 点击 DAG 名称
2. 查看 "Graph" 标签页以查看任务进度
3. 点击单个任务查看日志

## 第 3 部分：创建数据质量 DAG

### 步骤 9：数据质量管道

创建 `dags/data_quality_check.py`：

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator, BranchPythonOperator
from airflow.operators.bash import BashOperator
import random

default_args = {
    'owner': 'data-team',
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
}

def check_data_freshness(**context):
    """检查数据是否新鲜（< 24 小时）"""
    print("🕐 Checking data freshness...")
    # 模拟检查
    hours_old = random.randint(1, 30)
    print(f"Data is {hours_old} hours old")

    if hours_old > 24:
        print("⚠️  Data is stale!")
        return 'handle_stale_data'
    else:
        print("✓ Data is fresh")
        return 'check_data_completeness'

def check_data_completeness(**context):
    """检查是否所有预期数据都存在"""
    print("📋 Checking data completeness...")
    completeness = random.uniform(0.85, 1.0)
    print(f"Data completeness: {completeness:.2%}")

    if completeness < 0.95:
        raise ValueError(f"Data only {completeness:.2%} complete (expected >95%)")

    return True

def check_schema_validity(**context):
    """验证数据 schema"""
    print("🔍 Validating schema...")
    # 模拟 schema 检查
    print("✓ Schema is valid")
    return True

def check_data_quality_metrics(**context):
    """检查各种数据质量指标"""
    print("📊 Checking data quality metrics...")

    metrics = {
        'null_percentage': random.uniform(0, 0.05),
        'duplicate_percentage': random.uniform(0, 0.02),
        'outlier_percentage': random.uniform(0, 0.03),
    }

    print(f"Metrics: {metrics}")

    for metric, value in metrics.items():
        if value > 0.05:  # 5% 阈值
            raise ValueError(f"{metric} too high: {value:.2%}")

    print("✓ All quality metrics passed")
    return metrics

def handle_stale_data(**context):
    """处理数据过期场景"""
    print("🚨 Initiating data refresh process...")
    print("Sending alert to data team...")
    return "refresh_data_needed"

with DAG(
    'data_quality_pipeline',
    default_args=default_args,
    description='Data quality validation pipeline',
    schedule_interval='0 * * * *',  # 每小时运行
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['data-quality', 'monitoring'],
) as dag:

    freshness_check = BranchPythonOperator(
        task_id='check_data_freshness',
        python_callable=check_data_freshness,
    )

    completeness_check = PythonOperator(
        task_id='check_data_completeness',
        python_callable=check_data_completeness,
    )

    schema_check = PythonOperator(
        task_id='check_schema_validity',
        python_callable=check_schema_validity,
    )

    quality_check = PythonOperator(
        task_id='check_data_quality_metrics',
        python_callable=check_data_quality_metrics,
    )

    stale_data_handler = PythonOperator(
        task_id='handle_stale_data',
        python_callable=handle_stale_data,
    )

    success_notification = BashOperator(
        task_id='quality_check_passed',
        bash_command='echo "✅ All data quality checks passed!"',
    )

    # 定义工作流
    freshness_check >> [completeness_check, stale_data_handler]
    completeness_check >> schema_check >> quality_check >> success_notification
```

## 第 4 部分：监控 Airflow

### 步骤 10：在 Prometheus 中查看 Metrics

1. 访问 http://localhost:9090
2. 尝试以下查询：

```promql
# DAG 运行时长
airflow_dagrun_duration_seconds

# 任务失败
rate(airflow_task_failed[5m])

# Scheduler 心跳
airflow_scheduler_heartbeat

# DAG 处理时间
airflow_dag_processing_total_parse_time
```

### 步骤 11：创建 Grafana Dashboard

1. 访问 http://localhost:3000
2. 添加 Prometheus 数据源：
   - Configuration → Data Sources → Add data source
   - 选择 Prometheus
   - URL：`http://prometheus:9090`
   - Save & Test

3. 创建新 dashboard：
   - Create → Dashboard → Add new panel

**面板 1：任务成功率**
```promql
rate(airflow_task_success[5m]) / rate(airflow_task_start[5m])
```

**面板 2：DAG 运行时长**
```promql
avg(airflow_dagrun_duration_seconds) by (dag_id)
```

**面板 3：任务失败**
```promql
sum(rate(airflow_task_failed[5m])) by (dag_id)
```

### 步骤 12：查看日志

**通过 UI**：
1. 点击 DAG → 任务
2. 点击 "Log" 按钮
3. 查看实时或历史日志

**通过 CLI**：
```bash
# 查看任务日志
docker-compose exec airflow-webserver airflow tasks logs ml_training_pipeline download_dataset 2024-01-15

# 跟踪日志
docker-compose logs -f airflow-scheduler
```

## 第 5 部分：处理故障和告警

### 步骤 13：创建用于测试的故障 DAG

创建 `dags/failure_test.py`：

```python
from datetime import datetime
from airflow import DAG
from airflow.operators.python import PythonOperator
import random

def task_that_sometimes_fails():
    """50% 概率失败的任务"""
    if random.random() < 0.5:
        raise Exception("💥 Simulated failure!")
    print("✓ Task succeeded")

with DAG(
    'failure_test_pipeline',
    start_date=datetime(2024, 1, 1),
    schedule_interval=None,
    catchup=False,
    default_args={'retries': 3},
    tags=['testing'],
) as dag:

    failing_task = PythonOperator(
        task_id='sometimes_fails',
        python_callable=task_that_sometimes_fails,
    )
```

多次触发此 DAG 以查看失败和重试！

### 步骤 14：调试失败任务

当任务失败时：

1. **查看日志**：点击任务 → View Log
2. **检查 XCom**：查看任务间传递的数据
3. **清除并重新运行**：点击 "Clear" 以重试
4. **标记成功**：根据需要覆盖失败状态

## 挑战

### 挑战 1：添加邮件告警

为故障配置邮件告警：

```python
default_args = {
    'email': ['alerts@example.com'],
    'email_on_failure': True,
    'email_on_retry': False,
}
```

### 挑战 2：创建特征工程管道

构建一个 DAG，实现：
1. 从源提取数据
2. 计算特征
3. 存储在特征存储中
4. 验证特征质量

### 挑战 3：实现 SLA 监控

添加 SLA 以确保任务在时间限制内完成：

```python
train = PythonOperator(
    task_id='train_model',
    python_callable=train_model,
    sla=timedelta(hours=2),  # 如果超过 2 小时则告警
)
```

### 挑战 4：创建动态 DAG

根据配置动态生成任务：

```python
for model in ['resnet', 'vgg', 'efficientnet']:
    train_task = PythonOperator(
        task_id=f'train_{model}',
        python_callable=train_model,
        op_kwargs={'model_type': model},
    )
```

## 关键要点

1. **Airflow 编排工作流**：将复杂管道定义为代码
2. **DAG 代表工作流**：带有依赖关系的任务
3. **内置监控**：UI、日志、metrics 全部可用
4. **重试处理临时故障**：自动重试逻辑
5. **XCom 实现数据传递**：在任务间共享数据
6. **Prometheus 集成**：导出 metrics 用于监控
7. **灵活的调度**：Cron 表达式或间隔

## 生产最佳实践

### 1. 幂等任务
```python
def idempotent_task(**context):
    """可以安全重新运行的任务"""
    # 检查是否已完成
    if check_completion():
        print("Already done, skipping...")
        return

    # 执行工作
    process_data()
    mark_completed()
```

### 2. 任务超时
```python
train = PythonOperator(
    task_id='train_model',
    python_callable=train_model,
    execution_timeout=timedelta(hours=4),
)
```

### 3. 资源管理
```python
from airflow.models import Pool

# 在 UI 或通过 CLI 创建 pool
# airflow pools set gpu_pool 2 "GPU instances"

train = PythonOperator(
    task_id='train_model',
    python_callable=train_model,
    pool='gpu_pool',
)
```

### 4. 错误处理
```python
def safe_task(**context):
    try:
        process_data()
    except SpecificError as e:
        # 记录并优雅处理
        print(f"Known error: {e}")
        send_alert()
        raise  # 重新抛出以将任务标记为失败
    except Exception as e:
        # 意外错误
        print(f"Unexpected error: {e}")
        raise
```

## 下一步

1. **学习高级功能**：
   - Sensors（等待事件）
   - SubDAGs（嵌套工作流）
   - TaskGroups（组织任务）
   - 动态任务映射

2. **探索执行器**：
   - CeleryExecutor（分布式）
   - KubernetesExecutor（可扩展）
   - DaskExecutor（并行 Python）

3. **研究集成**：
   - AWS（S3、EMR、SageMaker）
   - GCP（BigQuery、Dataflow）
   - Spark、dbt、Great Expectations

4. **生产部署**：
   - Kubernetes 部署
   - 高可用设置
   - 安全和身份验证
   - DAG 的 CI/CD

## 资源

- [Apache Airflow Documentation](https://airflow.apache.org/docs/)
- [Airflow Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [Astronomer Guides](https://docs.astronomer.io/)
- [Awesome Apache Airflow](https://github.com/jghoman/awesome-apache-airflow)

## 清理

```bash
# 停止所有服务
docker-compose down

# 移除 volumes（数据将丢失）
docker-compose down -v
```

---

**恭喜！** 你已学习了使用 Apache Airflow 进行工作流编排以及如何监控数据管道。你现在可以构建、调度和监控复杂的 ML 工作流！
