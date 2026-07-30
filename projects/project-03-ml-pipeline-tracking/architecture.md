# 架构：带实验追踪的 ML 管道

**版本：** 1.0
**最后更新：** 2025 年 10 月 18 日

---

## 目录

1. [系统架构](#系统架构)
2. [组件设计](#组件设计)
3. [数据流](#数据流)
4. [基础设施架构](#基础设施架构)
5. [MLflow 架构](#mlflow-架构)
6. [Airflow 架构](#airflow-架构)
7. [技术决策](#技术决策)
8. [设计模式](#设计模式)
9. [安全考虑](#安全考虑)
10. [可扩展性考虑](#可扩展性考虑)

---

## 系统架构

### 高层概览

```
┌────────────────────────────────────────────────────────────────┐
│                   ML 管道生态系统                              │
└────────────────────────────────────────────────────────────────┘

                    ┌──────────────────┐
                    │   数据源         │
                    │  - CSV 文件      │
                    │  - REST API      │
                    │  - 数据库        │
                    └────────┬─────────┘
                             │
                             ▼
         ┌───────────────────────────────────────┐
         │      数据摄入与验证                    │
         │   - DataIngestion 类                   │
         │   - Great Expectations 验证器          │
         └─────────┬─────────────────────────────┘
                   │
                   ▼
         ┌─────────────────────┐    ┌──────────────┐
         │   数据版本控制      │◄───│     DVC      │
         │   - 原始数据        │    │  - MinIO     │
         │   - 处理后数据      │    │  - Git       │
         └─────────┬───────────┘    └──────────────┘
                   │
                   ▼
         ┌───────────────────────────────────────┐
         │      数据预处理                       │
         │   - 清洗                              │
         │   - 特征工程                          │
         │   - 训练/验证/测试集划分              │
         └─────────┬─────────────────────────────┘
                   │
                   ▼
         ┌───────────────────────────────────────┐
         │      特征存储 (PostgreSQL)            │
         │   - 结构化特征                        │
         │   - 特征元数据                        │
         └─────────┬─────────────────────────────┘
                   │
                   ▼
         ┌───────────────────────────────────────┐
         │      模型训练                         │
         │   - PyTorch 模型                      │
         │   - 超参数调优                        │
         │   - 交叉验证                          │
         └─────────┬─────────────────────────────┘
                   │
                   ├──────────────────────────┐
                   │                          │
                   ▼                          ▼
         ┌──────────────────┐      ┌──────────────────┐
         │  实验            │      │  模型            │
         │  追踪            │      │  工件            │
         │  - MLflow        │      │  - PyTorch       │
         │  - 参数           │      │  - ONNX          │
         │  - 指标           │      │  - 图表          │
         └──────────────────┘      └──────────────────┘
                   │
                   ▼
         ┌───────────────────────────────────────┐
         │      模型评估                         │
         │   - 测试集指标                        │
         │   - 混淆矩阵                          │
         │   - 按类别性能                        │
         └─────────┬─────────────────────────────┘
                   │
                   ▼
         ┌───────────────────────────────────────┐
         │      模型注册表                       │
         │   - 版本化模型                        │
         │   - 生命周期阶段                      │
         │   - 生产部署                          │
         └───────────────────────────────────────┘

                   ┌──────────────────────┐
                   │  工作流              │
                   │  编排                │
                   │  - Apache Airflow    │
                   │  - 任务调度           │
                   │  - 监控              │
                   └──────────────────────┘
```

---

## 组件设计

### 1. 数据摄入组件

**用途：** 从多个来源加载数据到管道中

**类图：**
```python
class DataIngestion:
    """
    处理来自多个来源的数据摄入。

    职责：
    - 从 CSV、API、数据库加载数据
    - 验证源连接
    - 保存带元数据的原始数据
    """

    def __init__(self, config: Dict[str, Any])
    def ingest_from_csv(self, file_path: str) -> pd.DataFrame
    def ingest_from_api(self, api_url: str, params: Dict) -> pd.DataFrame
    def ingest_from_database(self, conn_string: str, query: str) -> pd.DataFrame
    def save_raw_data(self, df: pd.DataFrame, filename: str) -> Path
    def get_metadata(self) -> Dict[str, Any]
```

**输入/输出：**
- **输入：** 文件路径、API URL、数据库连接字符串
- **输出：** 原始数据保存到 `data/raw/`，DataFrame 对象
- **副作用：** 创建目录、写入文件

**错误处理：**
- 网络错误：使用指数退避重试（最多 3 次）
- 文件未找到：记录错误并抛出 FileNotFoundError
- 数据库错误：记录连接详情（已脱敏）并抛出

---

### 2. 数据验证组件

**用途：** 在处理前验证数据质量

**类图：**
```python
class DataValidator:
    """
    使用 Great Expectations 验证数据。

    职责：
    - 定义期望套件
    - 运行验证检查
    - 生成验证报告
    """

    def __init__(self, context_root_dir: str)
    def create_expectation_suite(self, suite_name: str) -> ExpectationSuite
    def validate_data(self, df: pd.DataFrame, suite_name: str) -> bool
    def get_validation_report(self) -> str
    def add_expectation(self, expectation: Dict) -> None
```

**验证规则：**
```python
expectations = [
    # 模式验证
    {
        "type": "expect_table_column_count_to_equal",
        "value": 10
    },
    # 数据质量
    {
        "type": "expect_column_values_to_not_be_null",
        "column": "image_path"
    },
    # 领域验证
    {
        "type": "expect_column_values_to_be_in_set",
        "column": "label",
        "value_set": ["cat", "dog", "bird", "fish"]
    },
    # 范围验证
    {
        "type": "expect_table_row_count_to_be_between",
        "min_value": 1000,
        "max_value": 1000000
    }
]
```

**管道集成：**
- 验证在摄入之后运行
- 验证失败时管道停止
- 验证报告保存到 `reports/` 目录

---

### 3. 数据预处理组件

**用途：** 清洗和转换数据以用于训练

**类图：**
```python
class DataPreprocessor:
    """
    预处理数据用于模型训练。

    职责：
    - 清洗数据（重复项、缺失值）
    - 编码标签
    - 划分数据
    - 保存工件
    """

    def __init__(self, config: Dict[str, Any])
    def clean_data(self, df: pd.DataFrame) -> pd.DataFrame
    def encode_labels(self, df: pd.DataFrame) -> pd.DataFrame
    def create_train_test_split(self, df: pd.DataFrame) -> Tuple[pd.DataFrame, ...]
    def save_processed_data(self, train, val, test) -> None
    def run_pipeline(self, df: pd.DataFrame) -> Tuple[pd.DataFrame, ...]
```

**处理管道：**
```
原始数据
    ↓
删除重复项 (df.drop_duplicates())
    ↓
处理缺失值 (dropna/fillna)
    ↓
编码分类标签 (LabelEncoder)
    ↓
分层划分 (train_test_split, stratify=y)
    ↓
保存划分结果 + 工件
```

**生成的工件：**
- `artifacts/label_encoder.pkl` - 用于推理
- `artifacts/scaler.pkl` - 如果应用了归一化
- `artifacts/preprocessing_config.json` - 可复现性

---

### 4. 模型训练组件

**用途：** 使用实验追踪训练 ML 模型

**类图：**
```python
class ModelTrainer:
    """
    使用 MLflow 追踪训练模型。

    职责：
    - 创建模型架构
    - 带验证的训练
    - 记录到 MLflow
    - 保存最佳模型
    """

    def __init__(self, config: Dict, mlflow_tracker: MLflowTracker)
    def create_model(self, num_classes: int, model_name: str) -> nn.Module
    def train_epoch(self, model, train_loader, criterion, optimizer) -> Tuple[float, float]
    def validate(self, model, val_loader, criterion) -> Tuple[float, float]
    def train(self, train_loader, val_loader, num_classes: int, params: Dict) -> Tuple[nn.Module, float]
```

**训练循环：**
```python
for epoch in range(num_epochs):
    # 训练
    train_loss, train_acc = train_epoch(model, train_loader, criterion, optimizer)

    # 验证
    val_loss, val_acc = validate(model, val_loader, criterion)

    # 记录到 MLflow
    mlflow.log_metrics({
        "train_loss": train_loss,
        "train_acc": train_acc,
        "val_loss": val_loss,
        "val_acc": val_acc
    }, step=epoch)

    # 保存最佳模型
    if val_acc > best_val_acc:
        best_val_acc = val_acc
        torch.save(model.state_dict(), "best_model.pth")

    # 早停
    if no_improvement_for(patience):
        break
```

**支持的架构：**
1. **ResNet18** - 良好的基线，1100 万参数
2. **MobileNetV2** - 轻量级，350 万参数

---

### 5. MLflow 追踪器组件

**用途：** 集中管理 MLflow 操作

**类图：**
```python
class MLflowTracker:
    """
    MLflow 追踪操作的包装器。

    职责：
    - 开始/结束运行
    - 记录参数、指标、工件
    - 注册模型
    - 管理模型阶段
    """

    def __init__(self, tracking_uri: str, experiment_name: str)
    def start_run(self, run_name: str, tags: Dict) -> Run
    def log_params(self, params: Dict) -> None
    def log_metrics(self, metrics: Dict, step: int) -> None
    def log_artifact(self, artifact_path: str) -> None
    def log_model(self, model, artifact_path: str) -> None
    def register_model(self, model_uri: str, model_name: str) -> ModelVersion
    def transition_model_stage(self, model_name: str, version: int, stage: str) -> None
    def end_run(self) -> None
```

**使用模式：**
```python
# 初始化
tracker = MLflowTracker(
    tracking_uri="http://mlflow:5000",
    experiment_name="image_classification"
)

# 开始运行
tracker.start_run(run_name="resnet18_exp1", tags={"model": "resnet18"})

# 训练期间记录
tracker.log_params({"lr": 0.001, "batch_size": 32})
tracker.log_metrics({"val_acc": 85.2}, step=10)

# 记录模型
tracker.log_model(model, "model")

# 注册模型
model_version = tracker.register_model(
    model_uri=f"runs:/{run_id}/model",
    model_name="image_classifier"
)

# 提升为生产
tracker.transition_model_stage(
    model_name="image_classifier",
    version=model_version.version,
    stage="Production"
)

# 结束运行
tracker.end_run()
```

---

### 6. 模型评估器组件

**用途：** 全面的模型评估

**类图：**
```python
class ModelEvaluator:
    """
    在测试集上评估模型。

    职责：
    - 计算指标（准确率、精确率、召回率、F1）
    - 生成混淆矩阵
    - 创建可视化
    - 将结果记录到 MLflow
    """

    def __init__(self, config: Dict, mlflow_tracker: MLflowTracker)
    def evaluate(self, model, test_loader) -> Dict[str, float]
    def compute_metrics(self, y_true, y_pred) -> Dict
    def generate_confusion_matrix(self, y_true, y_pred) -> np.ndarray
    def plot_confusion_matrix(self, cm: np.ndarray, class_names: List[str]) -> plt.Figure
    def generate_classification_report(self, y_true, y_pred) -> str
```

**计算的指标：**
```python
metrics = {
    "test_accuracy": accuracy_score(y_true, y_pred),
    "test_precision": precision_score(y_true, y_pred, average='macro'),
    "test_recall": recall_score(y_true, y_pred, average='macro'),
    "test_f1": f1_score(y_true, y_pred, average='macro'),
    "per_class_precision": precision_score(y_true, y_pred, average=None),
    "per_class_recall": recall_score(y_true, y_pred, average=None),
    "per_class_f1": f1_score(y_true, y_pred, average=None)
}
```

---

## 数据流

### 端到端管道流

```
┌─────────────────────┐
│  1. 数据摄入        │
│  - 从源加载         │
│  - 保存到 raw/      │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  2. 数据验证        │
│  - GE 期望          │
│  - 通过/失败检查    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  3. 预处理          │
│  - 清洗数据         │
│  - 编码标签         │
│  - 划分数据         │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  4. DVC 版本控制    │
│  - dvc add          │
│  - git commit       │
│  - dvc push         │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  5. 模型训练        │
│  - 加载数据         │
│  - 训练模型         │
│  - 在 MLflow 中追踪 │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  6. 模型评估        │
│  - 测试指标         │
│  - 混淆矩阵         │
│  - 记录到 MLflow    │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  7. 模型注册表      │
│  - 注册模型         │
│  - 阶段转换         │
│  - 更新元数据       │
└─────────────────────┘
```

### 数据持久化流

```
┌───────────────┐
│   原始数据     │
│   (CSV/API)   │
└───────┬───────┘
        │
        ▼
┌───────────────────┐      ┌─────────────┐
│  data/raw/        │─────▶│  DVC 追踪   │
│  dataset.csv      │      │  .dvc 文件  │
└───────┬───────────┘      └─────────────┘
        │                          │
        │                          ▼
        │                  ┌───────────────┐
        │                  │  MinIO/S3     │
        │                  │  (远程)        │
        │                  └───────────────┘
        ▼
┌───────────────────┐
│  data/processed/  │
│  - train.csv      │
│  - val.csv        │
│  - test.csv       │
└───────┬───────────┘
        │
        ▼
┌───────────────────┐
│  PostgreSQL       │
│  特征存储         │
└───────────────────┘
```

### MLflow 工件流

```
┌──────────────┐
│  训练        │
│  执行        │
└──────┬───────┘
       │
       ├────────────────────────────┐
       │                            │
       ▼                            ▼
┌──────────────┐            ┌──────────────┐
│  参数        │            │   指标       │
│  - lr: 0.001 │            │  - 准确率    │
│  - batch: 32 │            │  - 损失      │
└──────┬───────┘            └──────┬───────┘
       │                            │
       ▼                            ▼
┌──────────────────────────────────────┐
│      PostgreSQL (MLflow 后端)        │
│      - 运行元数据                    │
│      - 参数                          │
│      - 指标                          │
└──────────────────────────────────────┘
       │
       ▼
┌──────────────┐
│  工件        │
│  - model.pth │
│  - plots/    │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────┐
│      MinIO/S3 (工件存储)             │
│      - 模型文件                      │
│      - 图表                          │
│      - 日志                          │
└──────────────────────────────────────┘
```

---

## 基础设施架构

### Docker Compose 架构

```yaml
┌─────────────────────────────────────────────────────┐
│               Docker Compose 网络                   │
└─────────────────────────────────────────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │    Redis     │  │    MinIO     │
│   端口 5432  │  │   端口 6379  │  │  端口 9000   │
│              │  │              │  │       9001   │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       │                 │                 │
       ▼                 │                 │
┌──────────────┐         │                 │
│   MLflow     │         │                 │
│  追踪服务器  │◄────────┘                 │
│   端口 5000  │◄──────────────────────────┘
└──────────────┘
       ▲
       │
┌──────┴───────────────────────────────────┐
│          Airflow 组件                    │
├──────────────────────────────────────────┤
│  ┌────────────┐  ┌─────────┐  ┌────────┐│
│  │ Web 服务器 │  │ 调度器  │  │ 工作器 ││
│  │ 端口 8080  │  │         │  │        ││
│  └────────────┘  └─────────┘  └────────┘│
└──────────────────────────────────────────┘
```

### 容器规格

**PostgreSQL 容器：**
```yaml
postgres:
  image: postgres:15
  environment:
    POSTGRES_USER: mlflow
    POSTGRES_PASSWORD: mlflow
    POSTGRES_DB: mlflow
  volumes:
    - postgres_data:/var/lib/postgresql/data
  ports:
    - "5432:5432"
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U mlflow"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**MLflow 容器：**
```yaml
mlflow:
  build: ./mlflow
  ports:
    - "5000:5000"
  environment:
    MLFLOW_BACKEND_STORE_URI: postgresql://mlflow:mlflow@postgres:5432/mlflow
    MLFLOW_S3_ENDPOINT_URL: http://minio:9000
    AWS_ACCESS_KEY_ID: minioadmin
    AWS_SECRET_ACCESS_KEY: minioadmin
  depends_on:
    postgres:
      condition: service_healthy
    minio:
      condition: service_started
  command: >
    mlflow server
    --backend-store-uri postgresql://mlflow:mlflow@postgres:5432/mlflow
    --default-artifact-root s3://mlflow/
    --host 0.0.0.0
    --port 5000
```

**MinIO 容器：**
```yaml
minio:
  image: minio/minio:latest
  command: server /data --console-address ":9001"
  environment:
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin
  ports:
    - "9000:9000"   # API
    - "9001:9001"   # 控制台
  volumes:
    - minio_data:/data
```

**Airflow Web 服务器：**
```yaml
airflow-webserver:
  build: ./airflow
  ports:
    - "8080:8080"
  environment:
    AIRFLOW__CORE__EXECUTOR: CeleryExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql://mlflow:mlflow@postgres:5432/airflow
    AIRFLOW__CELERY__BROKER_URL: redis://redis:6379/0
    AIRFLOW__CELERY__RESULT_BACKEND: db+postgresql://mlflow:mlflow@postgres:5432/airflow
  depends_on:
    - postgres
    - redis
  command: webserver
```

---

## MLflow 架构

### MLflow 组件栈

```
┌──────────────────────────────────────────────┐
│          MLflow 追踪服务器                    │
│              (端口 5000)                      │
└────────────┬─────────────────────────────────┘
             │
             ├─────────────────────┬───────────────────┐
             │                     │                   │
             ▼                     ▼                   ▼
    ┌─────────────────┐   ┌─────────────────┐  ┌──────────────┐
    │  实验           │   │  模型注册表     │  │  工件        │
    │  - 运行         │   │  - 版本         │  │  - 模型      │
    │  - 参数         │   │  - 阶段         │  │  - 图表      │
    │  - 指标         │   │  - 元数据       │  │  - 日志      │
    └────────┬────────┘   └────────┬────────┘  └──────┬───────┘
             │                     │                   │
             ▼                     ▼                   ▼
    ┌──────────────────────────────────────────────────────┐
    │             PostgreSQL 后端存储                      │
    │  - 实验元数据                                        │
    │  - 运行参数和指标                                    │
    │  - 模型注册表数据                                    │
    └──────────────────────────────────────────────────────┘
                               │
                               │
    ┌──────────────────────────────────────────────────────┐
    │             MinIO/S3 工件存储                       │
    │  - 模型工件 (model.pth)                              │
    │  - 图表和可视化                                      │
    │  - 日志和其他文件                                    │
    └──────────────────────────────────────────────────────┘
```

### MLflow 数据模型

**实验表：**
```sql
CREATE TABLE experiments (
    experiment_id INTEGER PRIMARY KEY,
    name VARCHAR(256) UNIQUE NOT NULL,
    artifact_location VARCHAR(256),
    lifecycle_stage VARCHAR(32),
    creation_time BIGINT,
    last_update_time BIGINT
);
```

**运行表：**
```sql
CREATE TABLE runs (
    run_uuid VARCHAR(32) PRIMARY KEY,
    name VARCHAR(256),
    source_type VARCHAR(20),
    source_name VARCHAR(500),
    entry_point_name VARCHAR(50),
    user_id VARCHAR(256),
    status VARCHAR(20),
    start_time BIGINT,
    end_time BIGINT,
    source_version VARCHAR(50),
    lifecycle_stage VARCHAR(20),
    artifact_uri VARCHAR(200),
    experiment_id INTEGER,
    FOREIGN KEY (experiment_id) REFERENCES experiments(experiment_id)
);
```

**模型注册表：**
```sql
CREATE TABLE registered_models (
    name VARCHAR(256) PRIMARY KEY,
    creation_time BIGINT,
    last_updated_time BIGINT,
    description VARCHAR(5000)
);

CREATE TABLE model_versions (
    name VARCHAR(256),
    version INTEGER,
    creation_time BIGINT,
    last_updated_time BIGINT,
    description VARCHAR(5000),
    user_id VARCHAR(256),
    current_stage VARCHAR(20),
    source VARCHAR(500),
    run_id VARCHAR(32),
    status VARCHAR(20),
    status_message VARCHAR(500),
    PRIMARY KEY (name, version),
    FOREIGN KEY (name) REFERENCES registered_models(name)
);
```

---

## Airflow 架构

### Airflow 组件

```
┌────────────────────────────────────────────────┐
│              Airflow 架构                      │
└────────────────────────────────────────────────┘

┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  Web 服务器  │      │  调度器      │      │  工作器      │
│  端口 8080   │      │  (核心)      │      │  (执行器)    │
│              │      │              │      │              │
│ - UI         │      │ - 解析 DAG   │      │ - 运行任务   │
│ - REST API   │      │ - 调度       │      │ - 报告       │
│ - 认证       │      │ - 触发       │      │   状态       │
└──────┬───────┘      └──────┬───────┘      └──────┬───────┘
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │   PostgreSQL   │
                    │   元数据 DB    │
                    │                │
                    │ - DAG 元数据   │
                    │ - 任务状态     │
                    │ - 运行历史     │
                    └────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │     Redis      │
                    │   (代理)       │
                    │                │
                    │ - 任务队列     │
                    │ - 消息         │
                    └────────────────┘
```

### DAG 执行流程

```
用户触发 DAG
       │
       ▼
调度器拾取 DAG
       │
       ▼
解析 DAG Python 文件
       │
       ▼
创建 DAG Run 实例
       │
       ▼
按依赖顺序对每个任务：
     │
     ├─▶ 创建 Task Instance
     │       │
     │       ▼
     │   将任务排队到 Redis
     │       │
     │       ▼
     │   工作器拾取任务
     │       │
     │       ▼
     │   执行任务 Python 代码
     │       │
     │       ▼
     │   向 PostgreSQL 报告状态
     │       │
     │       ▼
     │   通过 XCom 传递数据（如需要）
     │
     └─▶ 对下一个任务重复
```

### DAG 结构

```python
# ML 训练管道 DAG
DAG(
    dag_id='ml_training_pipeline',
    schedule_interval='@weekly',
    start_date=datetime(2025, 10, 1),
    catchup=False,
    max_active_runs=1,
    default_args={
        'owner': 'ml-team',
        'retries': 3,
        'retry_delay': timedelta(minutes=5),
        'execution_timeout': timedelta(hours=2)
    }
)

# 任务依赖图
ingest_data
    ↓
validate_data
    ↓
preprocess_data
    ↓
version_data_dvc
    ↓
train_model
    ↓
evaluate_model
    ↓
register_model
    ↓
notify_success
```

---

## 技术决策

### 为什么选择 Apache Airflow？

**优点：**
✅ 行业标准（被 Airbnb、Netflix、Adobe 使用）
✅ 丰富的 UI 用于监控和调试
✅ 广泛的插件生态系统
✅ Python 原生（便于 ML 团队使用）
✅ 支持复杂的依赖关系
✅ 内置重试和错误处理
✅ XCom 用于任务间通信

**缺点：**
❌ 初学者设置复杂
❌ 资源消耗大
❌ 对简单管道来说过于复杂

**考虑的替代方案：**
- **Prefect 2.0**：更现代，更容易设置，但不够成熟
- **Kubeflow Pipelines**：Kubernetes 原生，但复杂
- **Luigi**：更简单，但功能较少

**决策：** 选择 Airflow 是因为其行业相关性和学习价值。

---

### 为什么选择 MLflow？

**优点：**
✅ 开源，供应商中立
✅ 最受欢迎的 MLOps 平台（15K+ 星标）
✅ 完整的 ML 生命周期管理
✅ 易于与 PyTorch、TensorFlow、sklearn 集成
✅ 内置模型注册表
✅ REST API 用于自动化
✅ 活跃的社区和开发

**缺点：**
❌ UI 可以更现代化
❌ 内置模型服务有限
❌ 在超大规模场景下存在扩展挑战

**考虑的替代方案：**
- **Weights & Biases**：更好的 UI，但是商业/SaaS 产品
- **Neptune.ai**：功能更多，但是商业产品
- **TensorBoard**：局限于 TensorFlow 生态系统
- **Sacred**：维护不太活跃

**决策：** 选择 MLflow 是因为其开源、完整性和行业采用度。

---

### 为什么选择 DVC？

**优点：**
✅ 类似 Git 的数据工作流
✅ 存储无关（S3、GCS、Azure、SSH）
✅ 与 Git 无缝集成
✅ 轻量且快速
✅ 支持管道和指标追踪
✅ 开源且免费

**缺点：**
❌ 需要 Git 知识
❌ 对于非常大的文件可能较慢
❌ 团队采用有学习曲线

**考虑的替代方案：**
- **Git LFS**：功能有限，没有版本控制逻辑
- **Pachyderm**：更复杂，依赖 Kubernetes
- **LakeFS**：数据湖的 Git，对本项目来说过于复杂

**决策：** 选择 DVC 是因为其简单性和 Git 集成。

---

### 为什么选择 PostgreSQL？

**优点：**
✅ 久经考验的可靠性
✅ ACID 合规
✅ 丰富的索引和查询优化
✅ 支持 JSON 元数据
✅ 免费且开源
✅ 优秀的 Python 支持（psycopg2）

**缺点：**
❌ 对于大数据集不如 NoSQL 可扩展
❌ 垂直扩展限制

**考虑的替代方案：**
- **MySQL**：类似，但 PostgreSQL 的 JSON 支持更好
- **MongoDB**：NoSQL，但 ACID 合规对元数据很重要
- **SQLite**：对多服务设置来说太有限

**决策：** 选择 PostgreSQL 是因为其可靠性和功能丰富性。

---

## 设计模式

### 1. 仓库模式
将数据访问逻辑封装在单独的类中。

```python
class ExperimentRepository:
    """抽象化 MLflow 实验操作"""

    def get_all_runs(self, experiment_id: str) -> List[Run]:
        return mlflow.search_runs(experiment_ids=[experiment_id])

    def get_best_run(self, experiment_id: str, metric: str) -> Run:
        runs = self.get_all_runs(experiment_id)
        return runs.sort_values(f"metrics.{metric}", ascending=False).iloc[0]
```

### 2. 策略模式
允许不同的数据摄入策略。

```python
class DataIngestionStrategy(ABC):
    @abstractmethod
    def ingest(self) -> pd.DataFrame:
        pass

class CSVIngestionStrategy(DataIngestionStrategy):
    def ingest(self) -> pd.DataFrame:
        return pd.read_csv(self.path)

class APIIngestionStrategy(DataIngestionStrategy):
    def ingest(self) -> pd.DataFrame:
        response = requests.get(self.url)
        return pd.DataFrame(response.json())
```

### 3. 工厂模式
根据配置创建模型。

```python
class ModelFactory:
    @staticmethod
    def create_model(model_name: str, num_classes: int) -> nn.Module:
        if model_name == "resnet18":
            return create_resnet18(num_classes)
        elif model_name == "mobilenet_v2":
            return create_mobilenet_v2(num_classes)
        else:
            raise ValueError(f"未知模型：{model_name}")
```

### 4. 观察者模式
MLflow 记录作为训练事件的观察者。

```python
class TrainingObserver(ABC):
    @abstractmethod
    def on_epoch_end(self, epoch: int, metrics: Dict) -> None:
        pass

class MLflowObserver(TrainingObserver):
    def on_epoch_end(self, epoch: int, metrics: Dict) -> None:
        mlflow.log_metrics(metrics, step=epoch)
```

---

## 安全考虑

### 1. 凭据管理
- 将凭据存储在环境变量中
- 永远不要将密钥提交到 Git
- 使用 `.env` 文件（已 gitignore）
- 生产环境考虑使用 HashiCorp Vault

### 2. 网络安全
- 使用内部 Docker 网络进行服务通信
- 只向主机暴露必要的端口
- 使用 PostgreSQL 认证
- 需要 MinIO 访问密钥

### 3. 数据隐私
- 本项目不涉及 PII（个人身份信息）
- 对于真实项目：加密静态数据和传输中数据
- 实施访问控制

### 4. MLflow 安全
- MLflow UI 默认无认证（本地开发 OK）
- 生产环境：启用认证插件
- 限制工件存储访问

---

## 可扩展性考虑

### 当前限制
- 单机部署
- 无水平扩展
- 限于本地数据集（<1GB）
- 单个 Airflow 工作器

### 扩展策略（未来）

**水平扩展：**
- 多个 Airflow 工作器（Celery Executor）
- 多个 MLflow 追踪服务器（负载均衡）
- PostgreSQL 只读副本

**数据扩展：**
- 分区大数据集
- 开发时使用数据采样
- 实现增量加载
- 考虑对超大规模数据使用 Spark

**计算扩展：**
- GPU 加速训练
- 分布式训练（PyTorch DDP）
- 对密集型工作负载进行云端爆发

**存储扩展：**
- 对象存储（S3、GCS）用于工件
- 分布式文件系统（HDFS）
- 数据湖架构

---

**架构版本：** 1.0
**审核人：** AI 基础设施课程团队
**最后更新：** 2025 年 10 月 18 日
