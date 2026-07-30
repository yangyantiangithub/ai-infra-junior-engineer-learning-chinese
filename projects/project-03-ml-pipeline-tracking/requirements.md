# 项目需求：带实验追踪的 ML 管道

**版本：** 1.0
**最后更新：** 2025 年 10 月 18 日

---

## 目录

1. [功能需求](#功能需求)
2. [非功能需求](#非功能需求)
3. [技术规格](#技术规格)
4. [验收标准](#验收标准)
5. [约束与假设](#约束与假设)

---

## 功能需求

### FR-1：数据管道

#### FR-1.1：数据摄入
**描述：** 系统必须从多个来源摄入数据

**需求：**
- 支持 CSV 文件摄入
- 支持 REST API 数据获取
- 支持数据库查询（PostgreSQL）
- 处理不同模式的数据
- 存储带时间戳的原始数据

**输入：**
- 来自本地文件系统或云存储的 CSV 文件
- 返回 JSON 的 REST API 端点
- 针对 PostgreSQL 数据库的 SQL 查询

**输出：**
- 原始数据保存到 `data/raw/` 目录
- 包含记录计数的摄入日志
- 关于数据源和时间戳的元数据

**验收标准：**
- [ ] 成功从 CSV 摄入 100K+ 条记录
- [ ] 从公共 API（如 OpenML、Kaggle API）获取数据
- [ ] 从 PostgreSQL 数据库查询数据
- [ ] 优雅处理网络错误并自动重试
- [ ] 记录所有摄入活动

---

#### FR-1.2：数据验证
**描述：** 使用 Great Expectations 验证数据质量

**需求：**
- 为数据集模式定义期望套件
- 验证行数（最小/最大边界）
- 验证列存在性和类型
- 验证值范围和类别
- 检测必填列中的空值
- 生成验证报告

**验证规则：**
- 最少 1,000 条记录，最多 1,000,000 条记录
- 必填列：`image_path`、`label`、`split`
- 标签值必须在定义的集合内（例如 ["cat", "dog", "bird", "fish"]）
- 关键列中不允许空值
- 图片文件路径必须存在

**输出：**
- 验证报告（HTML）
- 布尔验证结果（通过/失败）
- 包含失败详细错误信息

**验收标准：**
- [ ] 创建包含 5+ 个期望的期望套件
- [ ] 验证能捕获模式违规
- [ ] 验证能捕获缺失值
- [ ] 验证能捕获无效标签
- [ ] 验证失败时管道停止

---

#### FR-1.3：数据版本控制
**描述：** 使用 DVC（数据版本控制）对数据集进行版本控制

**需求：**
- 在项目中初始化 DVC
- 配置远程存储（MinIO/S3）
- 追踪原始数据集和处理后的数据集
- 使用有意义的名称标记版本
- 支持从任意版本检索数据

**DVC 工作流：**
```bash
# 将数据集添加到 DVC
dvc add data/raw/dataset.csv

# 将 DVC 文件提交到 Git
git add data/raw/dataset.csv.dvc
git commit -m "数据集 v1.0 - 初始导入"
git tag -a "data-v1.0" -m "数据集版本 1.0"

# 推送到远程
dvc push
```

**验收标准：**
- [ ] DVC 已初始化并配置
- [ ] 使用 DVC 追踪原始数据
- [ ] 使用 DVC 追踪处理后的数据
- [ ] 配置了远程存储（MinIO）
- [ ] 可通过 `dvc pull` 检索数据
- [ ] 至少标记了 3 个数据版本

---

#### FR-1.4：数据预处理
**描述：** 清洗和预处理数据以用于模型训练

**需求：**
- 删除重复记录
- 处理缺失值（删除或填充）
- 编码分类标签
- 归一化/标准化特征
- 创建训练/验证/测试集划分（70/15/15）
- 保存预处理工件（编码器、缩放器）

**预处理步骤：**
1. **清洗：**
   - 删除完全重复的行
   - 删除关键值缺失的行
   - 修复数据类型不一致

2. **编码：**
   - 对分类目标进行标签编码
   - 保存编码器供推理使用

3. **划分：**
   - 分层划分以保持类别平衡
   - 固定随机种子以确保可复现性

**输出：**
- `data/processed/train.csv`
- `data/processed/val.csv`
- `data/processed/test.csv`
- `artifacts/label_encoder.pkl`
- `artifacts/scaler.pkl`（如适用）

**验收标准：**
- [ ] 预处理删除重复项
- [ ] 适当处理缺失值
- [ ] 标签编码为整数
- [ ] 数据划分为训练/验证/测试集
- [ ] 划分比例遵循 70/15/15
- [ ] 编码器已保存供复用
- [ ] 使用固定种子可复现预处理

---

#### FR-1.5：特征存储（可选）
**描述：** 在 PostgreSQL 中存储处理后的特征

**需求：**
- 在 PostgreSQL 中创建特征表
- 存储带有 ID 的特征向量
- 支持按 ID 检索特征
- 追踪特征版本

**模式：**
```sql
CREATE TABLE features (
    id SERIAL PRIMARY KEY,
    image_id VARCHAR(255) UNIQUE,
    feature_vector FLOAT[],
    label INTEGER,
    created_at TIMESTAMP DEFAULT NOW()
);
```

**验收标准：**
- [ ] 特征表已创建
- [ ] 特征成功存储
- [ ] 可按 ID 检索特征
- [ ] 追踪特征版本

---

### FR-2：模型训练管道

#### FR-2.1：模型训练
**描述：** 使用 PyTorch 训练图像分类模型

**需求：**
- 支持多种模型架构（ResNet18、MobileNetV2）
- 实现带验证的训练循环
- 支持迁移学习（预训练模型）
- 追踪训练进度（损失、准确率）
- 基于验证准确率保存最佳模型
- 支持早停

**模型架构：**
1. **ResNet18**（基线）
2. **MobileNetV2**（轻量级）

**训练配置：**
```python
{
    "model_name": "resnet18",
    "num_epochs": 20,
    "batch_size": 32,
    "learning_rate": 0.001,
    "optimizer": "adam",
    "lr_scheduler": "step_lr",
    "lr_step_size": 7,
    "lr_gamma": 0.1,
    "early_stopping_patience": 5
}
```

**验收标准：**
- [ ] 模型成功训练
- [ ] 训练和验证损失下降
- [ ] 模型验证准确率超过 75%
- [ ] 基于验证准确率保存最佳模型
- [ ] 每次实验训练在 30 分钟内完成

---

#### FR-2.2：超参数调优
**描述：** 系统性的超参数优化

**需求：**
- 支持网格搜索或 Optuna 优化
- 至少调优 3 个超参数
- 至少运行 10 种配置
- 在 MLflow 中追踪所有实验

**待调优超参数：**
- 学习率：[0.0001, 0.001, 0.01]
- 批次大小：[16, 32, 64]
- 模型架构：[resnet18, mobilenet_v2]
- 优化器：[adam, sgd]

**搜索空间：**
```python
{
    "learning_rate": [0.0001, 0.001, 0.01],
    "batch_size": [16, 32, 64],
    "model_name": ["resnet18", "mobilenet_v2"],
    "optimizer": ["adam", "sgd"]
}
```

**验收标准：**
- [ ] 至少 10 次实验使用不同配置
- [ ] 所有实验在 MLflow 中追踪
- [ ] 确定最佳超参数
- [ ] 系统性地比较结果

---

#### FR-2.3：实验追踪
**描述：** 使用 MLflow 追踪所有训练运行

**需求：**
- 记录所有超参数
- 记录每个 epoch 的指标（训练/验证损失、准确率）
- 记录最终评估指标
- 保存模型工件
- 使用有意义的标识符标记运行
- 捕获代码版本（Git 提交哈希）

**需要记录的内容：**

**参数：**
```python
{
    "model_name": "resnet18",
    "num_epochs": 20,
    "batch_size": 32,
    "learning_rate": 0.001,
    "optimizer": "adam",
    "dataset_version": "v1.2",
    "git_commit": "a3b4c5d"
}
```

**指标（每个 epoch）：**
```python
{
    "train_loss": 0.345,
    "train_accuracy": 87.5,
    "val_loss": 0.412,
    "val_accuracy": 85.2,
    "learning_rate": 0.001
}
```

**工件：**
- `model.pth` - 保存的模型权重
- `training_plot.png` - 损失/准确率曲线
- `confusion_matrix.png` - 分类结果
- `metrics.json` - 最终评估指标

**验收标准：**
- [ ] 所有参数已记录
- [ ] 每个 epoch 的指标已记录
- [ ] 模型工件已保存
- [ ] 运行已适当标记
- [ ] 已捕获 Git 提交哈希
- [ ] MLflow UI 显示所有实验

---

#### FR-2.4：模型工件
**描述：** 保存和组织模型工件

**需求：**
- 以 PyTorch 格式保存模型权重
- 导出 ONNX 格式用于推理
- 保存训练图表（损失/准确率曲线）
- 保存评估指标（JSON）
- 按运行 ID 组织工件

**工件结构：**
```
mlruns/
└── 0/                          # 实验 ID
    └── abc123def456/           # 运行 ID
        └── artifacts/
            ├── model/
            │   ├── model.pth           # PyTorch 权重
            │   ├── model.onnx          # ONNX 导出
            │   └── config.json         # 模型配置
            ├── plots/
            │   ├── training_curves.png
            │   └── confusion_matrix.png
            └── metrics/
                └── evaluation.json
```

**验收标准：**
- [ ] 模型权重已保存
- [ ] ONNX 导出已创建
- [ ] 训练图表已生成
- [ ] 指标以 JSON 格式保存
- [ ] 工件组织合理

---

#### FR-2.5：模型评估
**描述：** 在保留的测试集上评估模型

**需求：**
- 在测试集上评估（训练期间从未见过）
- 计算多种指标（准确率、精确率、召回率、F1）
- 生成混淆矩阵
- 生成分类报告
- 与基线比较

**需要计算的指标：**
- 准确率
- 精确率（按类别和宏平均）
- 召回率（按类别和宏平均）
- F1 分数（按类别和宏平均）
- 混淆矩阵

**评估输出：**
```python
{
    "test_accuracy": 86.4,
    "test_precision": 85.7,
    "test_recall": 86.1,
    "test_f1": 85.9,
    "per_class_metrics": {
        "cat": {"precision": 88.2, "recall": 89.1, "f1": 88.6},
        "dog": {"precision": 87.5, "recall": 86.8, "f1": 87.1},
        "bird": {"precision": 82.1, "recall": 83.0, "f1": 82.5},
        "fish": {"precision": 85.0, "recall": 85.5, "f1": 85.2}
    }
}
```

**验收标准：**
- [ ] 执行了测试集评估
- [ ] 计算了多种指标
- [ ] 生成了混淆矩阵
- [ ] 计算了每个类别的指标
- [ ] 结果已记录到 MLflow

---

### FR-3：模型注册表

#### FR-3.1：模型注册
**描述：** 在 MLflow 模型注册表中注册模型

**需求：**
- 在成功训练后注册模型
- 分配语义化版本
- 添加模型元数据（描述、标签）
- 链接到训练运行
- 追踪注册时间戳

**注册流程：**
```python
# 注册模型
model_uri = f"runs:/{run_id}/model"
result = mlflow.register_model(
    model_uri=model_uri,
    name="image_classifier",
    tags={
        "architecture": "resnet18",
        "dataset_version": "v1.2",
        "framework": "pytorch"
    }
)

# 添加描述
client.update_model_version(
    name="image_classifier",
    version=result.version,
    description="在 v1.2 数据集上训练的 ResNet18，测试准确率 86.4%"
)
```

**验收标准：**
- [ ] 模型成功注册
- [ ] 版本顺序编号
- [ ] 元数据附加到模型
- [ ] 保留运行链接
- [ ] 在 MLflow UI 中可见注册信息

---

#### FR-3.2：模型生命周期阶段
**描述：** 通过阶段管理模型生命周期

**需求：**
- 支持阶段转换（None → Staging → Production → Archived）
- 追踪阶段历史
- 每次只允许一个 Production 模型
- 需要批准才能提升到 Production（手动步骤）

**生命周期阶段：**
1. **None** - 新注册的模型
2. **Staging** - 正在测试的模型
3. **Production** - 当前部署的模型
4. **Archived** - 已弃用的模型

**阶段转换：**
```python
client.transition_model_version_stage(
    name="image_classifier",
    version=3,
    stage="Production",
    archive_existing_versions=True  # 归档之前的 Production 模型
)
```

**验收标准：**
- [ ] 模型可以在阶段之间转换
- [ ] 每次只允许一个 Production 模型
- [ ] 追踪阶段历史
- [ ] 之前的 Production 模型自动归档

---

#### FR-3.3：模型元数据
**描述：** 为模型附加全面的元数据

**需求：**
- 记录训练数据集版本
- 记录评估指标
- 记录超参数
- 记录代码版本（Git 提交）
- 记录训练时长
- 记录使用的硬件

**元数据结构：**
```python
{
    "dataset_version": "v1.2",
    "data_size": 50000,
    "test_accuracy": 86.4,
    "hyperparameters": {
        "learning_rate": 0.001,
        "batch_size": 32,
        "num_epochs": 20
    },
    "code_version": "a3b4c5d",
    "training_duration": 1847,  # 秒
    "hardware": "NVIDIA Tesla T4",
    "trained_by": "airflow-pipeline",
    "training_date": "2025-10-18T14:32:00Z"
}
```

**验收标准：**
- [ ] 所有元数据字段已填充
- [ ] 元数据可在 MLflow 中搜索
- [ ] 可追溯血缘关系

---

#### FR-3.4：模型检索 API
**描述：** 用于检索生产模型的 API

**需求：**
- 获取最新的 Production 模型
- 获取特定模型版本
- 下载模型工件
- 检索模型元数据

**API 示例：**
```python
# 获取最新的 Production 模型
model = mlflow.pyfunc.load_model(
    model_uri="models:/image_classifier/Production"
)

# 获取特定版本
model = mlflow.pyfunc.load_model(
    model_uri="models:/image_classifier/3"
)

# 获取模型元数据
client = mlflow.tracking.MlflowClient()
model_version = client.get_model_version(
    name="image_classifier",
    version="3"
)
print(model_version.description)
```

**验收标准：**
- [ ] 可以加载 Production 模型
- [ ] 可以加载特定版本
- [ ] 可以检索元数据
- [ ] 模型可用于推理

---

### FR-4：工作流编排

#### FR-4.1：Airflow DAG 设计
**描述：** 将 ML 管道实现为 Airflow DAG

**需求：**
- 定义任务依赖关系
- 实现任务重试
- 配置任务超时
- 在任务之间传递数据（XCom）
- 在 Airflow UI 中可视化 DAG

**DAG 结构：**
```python
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

**任务配置：**
```python
default_args = {
    'owner': 'ml-team',
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'execution_timeout': timedelta(hours=2),
    'email_on_failure': True,
    'email': ['mlops@example.com']
}
```

**验收标准：**
- [ ] DAG 在 Airflow UI 中可见
- [ ] 任务依赖关系正确
- [ ] 任务按顺序执行
- [ ] 失败任务自动重试
- [ ] DAG 可以手动触发

---

#### FR-4.2：管道调度
**描述：** 安排自动管道执行

**需求：**
- 按计划运行管道（每周、每日等）
- 支持手动触发
- 支持参数化运行
- 处理并发运行

**调度配置：**
```python
dag = DAG(
    'ml_training_pipeline',
    schedule_interval='@weekly',  # 每周日午夜
    catchup=False,  # 不回填过去日期的运行
    max_active_runs=1,  # 不允许并发运行
    start_date=datetime(2025, 10, 1)
)
```

**验收标准：**
- [ ] 管道按计划运行
- [ ] 手动触发正常工作
- [ ] 回填已禁用
- [ ] 防止并发运行

---

#### FR-4.3：错误处理
**描述：** 健壮的错误处理和恢复

**需求：**
- 重试临时故障（最多 3 次）
- 发送失败通知（邮件/Slack）
- 记录错误及堆栈跟踪
- 尽可能继续下游任务
- 实现熔断器

**错误处理：**
```python
@task(retries=3, retry_delay=timedelta(minutes=5))
def train_model(**context):
    try:
        # 训练代码
        pass
    except MemoryError as e:
        logger.error(f"OOM 错误：{e}")
        raise AirflowFailException("内存不足")
    except Exception as e:
        logger.error(f"训练失败：{e}", exc_info=True)
        raise
```

**验收标准：**
- [ ] 失败任务重试 3 次
- [ ] 错误记录详细信息
- [ ] 发送失败邮件
- [ ] 严重失败时管道停止
- [ ] 尽可能优雅降级

---

#### FR-4.4：管道监控
**描述：** 监控管道执行和健康状态

**需求：**
- 追踪任务时长
- 监控成功/失败率
- 对 SLA 违规发出告警
- 可视化管道指标
- 生成执行报告

**监控指标：**
- 任务时长（平均、p95、p99）
- 成功率（最近 7 天、30 天）
- 按任务统计失败率
- 管道端到端时长
- 资源使用情况（CPU、内存）

**验收标准：**
- [ ] 记录任务时长
- [ ] 追踪成功率
- [ ] 检测 SLA 违规
- [ ] 在 Airflow UI 中可视化指标
- [ ] 生成每周执行报告

---

#### FR-4.5：通知
**描述：** 通知团队管道状态

**需求：**
- 失败时发送邮件通知
- 成功/失败时发送 Slack 通知
- 通知中包含运行详情
- 失败时附带日志
- 成功时发送摘要报告

**通知内容：**

**失败邮件：**
```
主题：[失败] ML 训练管道 - 2025-10-18

管道：ml_training_pipeline
状态：失败
失败任务：train_model
执行日期：2025-10-18 14:30:00
时长：45 分钟

错误：模型训练期间内存不足

日志：[已附加]
Airflow 链接：http://airflow:8080/dags/ml_training_pipeline/grid
```

**成功邮件：**
```
主题：[成功] ML 训练管道 - 2025-10-18

管道：ml_training_pipeline
状态：成功
执行日期：2025-10-18 14:30:00
时长：2 小时 15 分钟

结果：
- 测试准确率：86.4%
- 模型版本：12
- 模型阶段：Staging

MLflow 链接：http://mlflow:5000/experiments/0
```

**验收标准：**
- [ ] 发送失败通知
- [ ] 发送成功通知
- [ ] 通知包含相关详情
- [ ] 包含 Airflow 和 MLflow 的链接

---

## 非功能需求

### NFR-1：可复现性

**描述：** 所有实验必须完全可复现

**需求：**
- 所有操作使用固定随机种子
- 代码版本控制（Git）
- 数据版本控制（DVC）
- 固定依赖版本
- 记录环境设置

**可复现性检查清单：**
- [ ] 固定随机种子（`torch.manual_seed(42)`）
- [ ] `requirements.txt` 中固定依赖版本
- [ ] 使用 DVC 进行数据版本控制
- [ ] 使用 Git 进行代码版本控制
- [ ] 记录环境（Docker/Conda）
- [ ] 实验可从元数据复现

---

### NFR-2：性能

**描述：** 管道必须达到性能目标

**性能目标：**
- 数据摄入：10 分钟内处理 100K 条记录
- 数据验证：5 分钟内完成
- 预处理：10 分钟内完成
- 训练：每次实验在 30 分钟内完成
- MLflow UI：2 秒内加载实验
- Airflow：处理 10+ 个并发任务

**验收标准：**
- [ ] 数据管道在 25 分钟内完成
- [ ] 训练在 30 分钟内完成
- [ ] MLflow UI 响应迅速（<2 秒加载）
- [ ] Airflow 处理并发任务
- [ ] 长时间运行无内存泄漏

---

### NFR-3：可靠性

**描述：** 管道必须高度可靠

**可靠性目标：**
- 管道成功率：>95%
- 临时失败自动重试
- 数据验证防止错误数据
- 监控对异常发出告警

**验收标准：**
- [ ] 20 次运行中 95%+ 成功率
- [ ] 临时失败自动重试
- [ ] 错误数据被验证捕获
- [ ] 监控告警正常工作

---

### NFR-4：可扩展性

**描述：** 系统必须能够扩展以应对增长

**可扩展性目标：**
- MLflow 支持 100+ 个实验
- 处理高达 1GB 的数据集
- 模型注册表支持 50+ 个版本
- Airflow 调度 10+ 个管道

**验收标准：**
- [ ] 追踪 100+ 个实验
- [ ] 成功处理 1GB 数据集
- [ ] 注册 50+ 个模型版本
- [ ] 调度多个管道

---

### NFR-5：可用性

**描述：** 系统必须易于使用

**需求：**
- 清晰的错误消息
- 全面的文档
- 示例笔记本
- 设置自动化
- 直观的 UI 导航

**验收标准：**
- [ ] 错误消息可操作
- [ ] 文档完整
- [ ] 设置脚本正常运作
- [ ] 提供示例笔记本
- [ ] 新用户可在 1 小时内上手

---

### NFR-6：可维护性

**描述：** 代码必须可维护

**需求：**
- 模块化代码结构
- 全面的日志记录
- 组件单元测试
- 管道集成测试
- 代码文档（docstring）

**验收标准：**
- [ ] 代码按模块组织
- [ ] 所有函数有 docstring
- [ ] 适当的日志级别
- [ ] 70%+ 测试覆盖率
- [ ] 集成测试通过

---

## 技术规格

### 技术栈版本

```yaml
python: "3.11+"
pytorch: "2.1.0"
mlflow: "2.8.1"
airflow: "2.7.3"
dvc: "3.30.1"
great-expectations: "0.18.3"
postgresql: "15"
redis: "7"
minio: "latest"
```

### 基础设施需求

**最低硬件：**
- CPU：4 核
- RAM：16GB
- 存储：50GB
- GPU：可选（加速训练）

**推荐硬件：**
- CPU：8 核
- RAM：32GB
- 存储：100GB
- GPU：NVIDIA T4 或更高

### 网络需求

**需要暴露的端口：**
- 5000：MLflow UI
- 8080：Airflow UI
- 9000：MinIO API
- 9001：MinIO 控制台
- 5432：PostgreSQL
- 6379：Redis

---

## 验收标准

### 阶段一：基础设施 ✅

- [ ] 创建 Docker Compose 文件
- [ ] 所有服务成功启动
- [ ] MLflow UI 可通过 http://localhost:5000 访问
- [ ] Airflow UI 可通过 http://localhost:8080 访问
- [ ] MinIO 控制台可通过 http://localhost:9001 访问
- [ ] PostgreSQL 在端口 5432 可访问
- [ ] 服务在重启后数据持久化

### 阶段二：数据管道 ✅

- [ ] 数据摄入模块完成
- [ ] 从 CSV、API、数据库摄入
- [ ] 使用 Great Expectations 进行数据验证
- [ ] 验证套件有 5+ 个期望
- [ ] 预处理管道功能正常
- [ ] 创建训练/验证/测试集划分
- [ ] DVC 已初始化并配置
- [ ] 使用 DVC 进行数据版本控制
- [ ] 标记 3+ 个数据版本

### 阶段三：训练与追踪 ✅

- [ ] 训练模块完成
- [ ] 支持 ResNet18 和 MobileNetV2
- [ ] MLflow 追踪已集成
- [ ] 参数已记录
- [ ] 每个 epoch 的指标已记录
- [ ] 模型工件已保存
- [ ] 追踪 5+ 个实验
- [ ] 模型验证准确率超过 75%
- [ ] 训练在 30 分钟内完成

### 阶段四：模型注册表 ✅

- [ ] 模型在 MLflow 中注册
- [ ] 存在 3+ 个模型版本
- [ ] 生命周期阶段正常工作
- [ ] 指定了 Production 模型
- [ ] 模型元数据完整
- [ ] 测试了回滚到之前版本

### 阶段五：编排 ✅

- [ ] Airflow DAG 已实现
- [ ] DAG 中有 7+ 个任务
- [ ] 任务依赖关系正确
- [ ] DAG 成功端到端运行
- [ ] 管道每周调度
- [ ] 错误处理和重试正常工作
- [ ] 通知已配置
- [ ] 3+ 次成功的调度运行

### 阶段六：文档 ✅

- [ ] README.md 完整
- [ ] Architecture.md 包含图表
- [ ] 创建了设置指南
- [ ] MLflow 使用指南
- [ ] DVC 工作流文档
- [ ] 故障排除指南
- [ ] 代码注释和 docstring
- [ ] 示例笔记本

---

## 约束与假设

### 约束

1. **仅限本地开发**——无需云部署
2. **小数据集**——最大 1GB，以便快速迭代
3. **不需要 GPU**——CPU 训练可接受
4. **单机**——无分布式训练
5. **英文文档**——所有文档使用英文

### 假设

1. **Docker 可用**——用户已安装 Docker
2. **互联网访问**——用于下载模型和数据
3. **基础的 ML 知识**——用户了解 ML 概念
4. **Git 知识**——用户熟悉 Git
5. **Linux/MacOS**——主要开发平台（Windows 需要调整）

---

## 不包含范围

以下内容**不**在本项目要求范围内：

- ❌ 云部署（AWS、GCP、Azure）
- ❌ 分布式训练（多 GPU、多节点）
- ❌ 生产级基础设施
- ❌ 模型服务/部署（在项目 1-2 中已涵盖）
- ❌ A/B 测试框架
- ❌ 模型漂移检测
- ❌ 特征存储（Feast）——可选，非必需
- ❌ AutoML 集成
- ❌ 实时推理
- ❌ 模型可解释性（SHAP、LIME）

这些主题将在后续项目（高级工程师级别）中涵盖。

---

**需求版本：** 1.0
**批准：** AI 基础设施课程团队
**审核日期：** 2025 年 10 月 18 日
