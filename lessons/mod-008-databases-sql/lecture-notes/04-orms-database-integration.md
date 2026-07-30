# Lecture 04: ORM 与数据库集成

## 目录
1. [引言](#introduction)
2. [什么是 ORM？](#what-is-an-orm)
3. [SQLAlchemy 简介](#introduction-to-sqlalchemy)
4. [设置 SQLAlchemy](#setting-up-sqlalchemy)
5. [定义模型](#defining-models)
6. [使用 ORM 进行 CRUD 操作](#crud-operations-with-orm)
7. [使用 SQLAlchemy 查询](#querying-with-sqlalchemy)
8. [SQLAlchemy 中的关系](#relationships-in-sqlalchemy)
9. [使用 Alembic 进行迁移](#migrations-with-alembic)
10. [连接池](#connection-pooling)
11. [事务管理](#transaction-management)
12. [ML 应用的 ORM 模式](#orm-patterns-for-ml-applications)
13. [最佳实践](#best-practices)
14. [总结与关键要点](#summary-and-key-takeaways)

## 引言

对象关系映射（ORM）弥合了面向对象编程与关系型数据库之间的鸿沟。你不必编写原始 SQL，而是使用自动映射到数据库表的 Python 对象。这使得开发更快、更安全、更易维护——对于 ML 基础设施应用尤其关键。

### 学习目标

在本讲座结束时，你将：
- 理解什么是 ORM 及其优势
- 设置和配置 SQLAlchemy
- 将数据库模型定义为 Python 类
- 使用 ORM 执行 CRUD 操作
- 使用 SQLAlchemy 编写复杂查询
- 实现模型之间的关系
- 使用 Alembic 管理数据库迁移
- 使用连接池和事务
- 将 ORM 模式应用于 ML 应用

### 前置条件
- 讲座 01-03（SQL 基础与高级概念）
- Python 编程熟练度
- 理解 Python 中的类和对象

### 预计时间
4-5 小时（包括动手 ORM 练习）

## 什么是 ORM？

### 定义

**对象关系映射器（ORM）**是一个库，它：
- 将数据库表映射到 Python 类
- 将表行映射到 Python 对象
- 将列映射到对象属性
- 自动在 SQL 和 Python 之间转换

### 没有 ORM（原始 SQL）

```python
import sqlite3

# 手动 SQL 字符串格式化（容易出错！）
conn = sqlite3.connect('database.db')
cursor = conn.cursor()

# 插入
model_name = "BERT-base"
framework = "PyTorch"
cursor.execute(
    "INSERT INTO models (name, framework) VALUES (?, ?)",
    (model_name, framework)
)

# 查询
cursor.execute("SELECT * FROM models WHERE framework = ?", (framework,))
rows = cursor.fetchall()

# 手动创建对象
models = []
for row in rows:
    model = {
        'id': row[0],
        'name': row[1],
        'framework': row[2]
    }
    models.append(model)

conn.commit()
conn.close()
```

**问题：**
- SQL 字符串容易出错
- 无类型检查
- 手动构造对象
- SQL 注入风险
- 数据库特定的 SQL

### 使用 ORM（SQLAlchemy）

```python
from sqlalchemy.orm import Session
from models import Model

# 面向对象的方法
with Session(engine) as session:
    # 创建
    model = Model(name="BERT-base", framework="PyTorch")
    session.add(model)
    session.commit()

    # 查询
    pytorch_models = session.query(Model).filter_by(framework="PyTorch").all()

    # 作为对象访问
    for model in pytorch_models:
        print(f"{model.name} ({model.framework})")
```

**优势：**
- Pythonic 代码
- 类型安全
- 自动生成 SQL
- SQL 注入防护
- 数据库无关性

### ORM 的优势

1. **生产力**：编写更少的代码
2. **可维护性**：更易阅读和修改
3. **类型安全**：在开发时捕获错误
4. **安全性**：防护 SQL 注入
5. **可移植性**：轻松切换数据库
6. **抽象性**：不需要成为 SQL 专家
7. **验证**：内置数据验证

### ORM 的劣势

1. **学习曲线**：需要学习 ORM API
2. **性能**：复杂查询可能比原始 SQL 慢
3. **抽象泄漏**：有时需要回到原始 SQL
4. **开销**：代码和数据库之间的额外层
5. **N+1 查询**：容易意外产生性能问题

## SQLAlchemy 简介

### 什么是 SQLAlchemy？

**SQLAlchemy** 是最流行的 Python ORM，由以下部分组成：

1. **Core**：SQL 生成和执行
2. **ORM**：对象关系映射
3. **Engine**：数据库连接管理
4. **Session**：事务和查询管理

### SQLAlchemy 架构

```
┌─────────────────────────────────────┐
│  你的 Python 应用程序              │
├─────────────────────────────────────┤
│  SQLAlchemy ORM 层                  │
│  （模型、查询、会话）              │
├─────────────────────────────────────┤
│  SQLAlchemy Core                    │
│  （SQL 表达式语言）                │
├─────────────────────────────────────┤
│  Engine（连接池）                  │
├─────────────────────────────────────┤
│  DBAPI（psycopg2、sqlite3 等）     │
├─────────────────────────────────────┤
│  数据库（PostgreSQL、MySQL 等）    │
└─────────────────────────────────────┘
```

### 支持的数据库

- PostgreSQL（推荐用于生产）
- MySQL/MariaDB
- SQLite（开发环境很好）
- Oracle
- Microsoft SQL Server

## 设置 SQLAlchemy

### 安装

```bash
# 安装 SQLAlchemy
pip install sqlalchemy

# 数据库驱动
pip install psycopg2-binary  # PostgreSQL
pip install pymysql          # MySQL
# SQLite 支持是内置的
```

### 创建数据库引擎

```python
from sqlalchemy import create_engine

# SQLite（基于文件）
engine = create_engine('sqlite:///ml_database.db', echo=True)

# PostgreSQL
engine = create_engine(
    'postgresql://username:password@localhost:5432/ml_database',
    echo=True  # 记录所有 SQL（用于开发）
)

# MySQL
engine = create_engine(
    'mysql+pymysql://username:password@localhost:3306/ml_database',
    echo=True
)
```

**连接字符串格式：**
```
dialect+driver://username:password@host:port/database
```

### 创建会话工厂

```python
from sqlalchemy.orm import sessionmaker

# 创建会话工厂
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)

# 使用会话
session = SessionLocal()
try:
    # 数据库操作
    pass
finally:
    session.close()

# 或者使用上下文管理器（推荐）
with SessionLocal() as session:
    # 数据库操作
    pass  # 自动关闭
```

## 定义模型

### 基本模型

```python
from sqlalchemy import Column, Integer, String, Float, DateTime
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

# 所有模型的基类
Base = declarative_base()

class Model(Base):
    __tablename__ = 'models'

    # 列
    model_id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(100), nullable=False, unique=True)
    framework = Column(String(50), nullable=False)
    version = Column(String(20), default='1.0.0')
    accuracy = Column(Float)
    created_at = Column(DateTime, default=datetime.utcnow)
    updated_at = Column(DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    def __repr__(self):
        return f"<Model(name='{self.name}', framework='{self.framework}')>"
```

### 常用列类型

```python
from sqlalchemy import (
    Integer, BigInteger, SmallInteger,
    String, Text,
    Float, Numeric,
    Boolean,
    Date, DateTime, Time,
    JSON, ARRAY
)

class Example(Base):
    __tablename__ = 'examples'

    id = Column(Integer, primary_key=True)

    # 整数
    count = Column(Integer)
    big_number = Column(BigInteger)

    # 字符串
    name = Column(String(100))       # VARCHAR(100)
    description = Column(Text)       # 无限制文本

    # 数字
    accuracy = Column(Float)         # 浮点数
    price = Column(Numeric(10, 2))   # 小数：12345678.90

    # 布尔值
    is_active = Column(Boolean, default=True)

    # 日期
    created_date = Column(Date)
    created_at = Column(DateTime)
    created_time = Column(Time)

    # JSON（PostgreSQL）
    metadata_json = Column(JSON)

    # 数组（PostgreSQL）
    tags = Column(ARRAY(String))
```

### 列约束

```python
from sqlalchemy import CheckConstraint

class Model(Base):
    __tablename__ = 'models'

    model_id = Column(Integer, primary_key=True)

    # NOT NULL
    name = Column(String(100), nullable=False)

    # UNIQUE
    email = Column(String(255), unique=True)

    # DEFAULT
    status = Column(String(20), default='training')

    # CHECK 约束
    accuracy = Column(
        Float,
        CheckConstraint('accuracy >= 0 AND accuracy <= 1', name='accuracy_range')
    )

    # 索引
    framework = Column(String(50), index=True)
```

### 创建表

```python
# 创建所有表
Base.metadata.create_all(engine)

# 创建特定表
Model.__table__.create(engine)

# 删除所有表
Base.metadata.drop_all(engine)
```

## 使用 ORM 进行 CRUD 操作

### 创建（INSERT）

```python
from models import Model
from sqlalchemy.orm import Session

with Session(engine) as session:
    # 创建单个对象
    model = Model(
        name="BERT-base",
        framework="PyTorch",
        accuracy=0.92
    )
    session.add(model)
    session.commit()

    # 访问自动生成的 ID
    print(f"Created model with ID: {model.model_id}")

    # 创建多个对象
    models = [
        Model(name="ResNet-50", framework="TensorFlow", accuracy=0.88),
        Model(name="GPT-2", framework="PyTorch", accuracy=0.95),
    ]
    session.add_all(models)
    session.commit()
```

### 读取（SELECT）

```python
with Session(engine) as session:
    # 获取所有
    all_models = session.query(Model).all()

    # 通过主键获取
    model = session.query(Model).get(1)  # ID = 1

    # 获取第一个
    first_model = session.query(Model).first()

    # 过滤
    pytorch_models = session.query(Model).filter_by(framework="PyTorch").all()

    # 复杂过滤
    accurate_models = session.query(Model).filter(Model.accuracy > 0.90).all()

    # 计数
    model_count = session.query(Model).count()
```

### 更新

```python
with Session(engine) as session:
    # 获取并修改
    model = session.query(Model).filter_by(name="BERT-base").first()
    model.accuracy = 0.93
    model.version = "2.0.0"
    session.commit()

    # 批量更新
    session.query(Model).filter_by(framework="PyTorch").update({
        "status": "deployed"
    })
    session.commit()
```

### 删除

```python
with Session(engine) as session:
    # 删除特定对象
    model = session.query(Model).filter_by(name="BERT-base").first()
    session.delete(model)
    session.commit()

    # 批量删除
    session.query(Model).filter_by(status="archived").delete()
    session.commit()
```

## 使用 SQLAlchemy 查询

### 基本查询

```python
# 所有记录
models = session.query(Model).all()

# 特定列
names = session.query(Model.name, Model.framework).all()
# 返回元组列表：[('BERT', 'PyTorch'), ...]

# 单个结果
model = session.query(Model).filter_by(name="BERT").one()  # 0 或 >1 时抛出异常
model = session.query(Model).filter_by(name="BERT").one_or_none()  # 未找到时返回 None
```

### 过滤

```python
# filter_by：简单等值
models = session.query(Model).filter_by(framework="PyTorch").all()

# filter：复杂条件
models = session.query(Model).filter(Model.accuracy > 0.90).all()

# 多个过滤条件（AND）
models = session.query(Model).filter(
    Model.framework == "PyTorch",
    Model.accuracy > 0.90
).all()

# OR 条件
from sqlalchemy import or_
models = session.query(Model).filter(
    or_(
        Model.framework == "PyTorch",
        Model.framework == "TensorFlow"
    )
).all()

# IN 运算符
from sqlalchemy import in_
models = session.query(Model).filter(
    Model.framework.in_(['PyTorch', 'TensorFlow'])
).all()

# LIKE
models = session.query(Model).filter(Model.name.like('%BERT%')).all()

# BETWEEN
models = session.query(Model).filter(Model.accuracy.between(0.85, 0.95)).all()

# IS NULL
models = session.query(Model).filter(Model.accuracy.is_(None)).all()
```

### 排序与限制

```python
# 排序
models = session.query(Model).order_by(Model.accuracy.desc()).all()

# 多条件排序
models = session.query(Model).order_by(Model.framework, Model.accuracy.desc()).all()

# 限制
top_5 = session.query(Model).order_by(Model.accuracy.desc()).limit(5).all()

# 偏移（分页）
page_2 = session.query(Model).limit(10).offset(10).all()
```

### 聚合

```python
from sqlalchemy import func

# 计数
count = session.query(func.count(Model.model_id)).scalar()

# 平均值
avg_accuracy = session.query(func.avg(Model.accuracy)).scalar()

# 最小和最大
min_acc = session.query(func.min(Model.accuracy)).scalar()
max_acc = session.query(func.max(Model.accuracy)).scalar()

# 分组
results = session.query(
    Model.framework,
    func.count(Model.model_id).label('count'),
    func.avg(Model.accuracy).label('avg_accuracy')
).group_by(Model.framework).all()

# Having
results = session.query(
    Model.framework,
    func.count(Model.model_id).label('count')
).group_by(Model.framework).having(func.count(Model.model_id) > 1).all()
```

## SQLAlchemy 中的关系

### 一对多关系

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship

class Model(Base):
    __tablename__ = 'models'

    model_id = Column(Integer, primary_key=True)
    name = Column(String(100))

    # 关系（一个模型有多个实验）
    experiments = relationship("Experiment", back_populates="model")

class Experiment(Base):
    __tablename__ = 'experiments'

    experiment_id = Column(Integer, primary_key=True)
    model_id = Column(Integer, ForeignKey('models.model_id'))
    accuracy = Column(Float)

    # 关系（多个实验属于一个模型）
    model = relationship("Model", back_populates="experiments")
```

**用法：**
```python
with Session(engine) as session:
    # 创建带有实验的模型
    model = Model(name="BERT")
    exp1 = Experiment(accuracy=0.92)
    exp2 = Experiment(accuracy=0.93)

    model.experiments.append(exp1)
    model.experiments.append(exp2)

    session.add(model)
    session.commit()

    # 访问实验
    model = session.query(Model).filter_by(name="BERT").first()
    for exp in model.experiments:
        print(f"Experiment {exp.experiment_id}: {exp.accuracy}")

    # 从实验访问模型
    exp = session.query(Experiment).first()
    print(f"Model: {exp.model.name}")
```

### 多对多关系

```python
# 关联表
model_tags = Table(
    'model_tags',
    Base.metadata,
    Column('model_id', Integer, ForeignKey('models.model_id')),
    Column('tag_id', Integer, ForeignKey('tags.tag_id'))
)

class Model(Base):
    __tablename__ = 'models'

    model_id = Column(Integer, primary_key=True)
    name = Column(String(100))

    # 多对多
    tags = relationship("Tag", secondary=model_tags, back_populates="models")

class Tag(Base):
    __tablename__ = 'tags'

    tag_id = Column(Integer, primary_key=True)
    tag_name = Column(String(50))

    # 多对多
    models = relationship("Model", secondary=model_tags, back_populates="tags")
```

**用法：**
```python
with Session(engine) as session:
    # 创建模型和标签
    model = Model(name="BERT")
    tag1 = Tag(tag_name="NLP")
    tag2 = Tag(tag_name="Transformer")

    model.tags.extend([tag1, tag2])

    session.add(model)
    session.commit()

    # 按标签查询模型
    nlp_models = session.query(Model).join(Model.tags).filter(Tag.tag_name == "NLP").all()
```

### 即时加载（避免 N+1 查询）

```python
from sqlalchemy.orm import joinedload, selectinload

# 没有即时加载（N+1 问题）：
models = session.query(Model).all()
for model in models:  # 1 次查询
    for exp in model.experiments:  # N 次查询（每个模型一次）
        print(exp.accuracy)

# 使用 joinedload（使用 JOIN）：
models = session.query(Model).options(joinedload(Model.experiments)).all()
for model in models:  # 总共 1 次查询
    for exp in model.experiments:
        print(exp.accuracy)

# 使用 selectinload（使用单独的 SELECT IN）：
models = session.query(Model).options(selectinload(Model.experiments)).all()
```

## 使用 Alembic 进行迁移

### 什么是 Alembic？

**Alembic** 是 SQLAlchemy 的数据库迁移工具，它：
- 跟踪 schema 随时间的变化
- 允许升级/降级数据库 schema
- 与版本控制系统配合使用

### 设置 Alembic

```bash
# 安装
pip install alembic

# 初始化
alembic init alembic

# 创建：
# alembic/
# ├── env.py
# ├── script.py.mako
# └── versions/
# alembic.ini
```

### 配置 Alembic

**alembic.ini：**
```ini
# 设置数据库 URL
sqlalchemy.url = postgresql://user:password@localhost/ml_database
```

**alembic/env.py：**
```python
# 导入你的模型
from models import Base
target_metadata = Base.metadata
```

### 创建迁移

```bash
# 从模型变化自动生成迁移
alembic revision --autogenerate -m "create models table"

# 手动迁移
alembic revision -m "add accuracy column"
```

**生成的迁移：**
```python
# alembic/versions/001_create_models_table.py
def upgrade():
    op.create_table(
        'models',
        sa.Column('model_id', sa.Integer(), primary_key=True),
        sa.Column('name', sa.String(100), nullable=False),
        sa.Column('framework', sa.String(50))
    )

def downgrade():
    op.drop_table('models')
```

### 运行迁移

```bash
# 升级到最新
alembic upgrade head

# 升级到特定版本
alembic upgrade abc123

# 降级一个版本
alembic downgrade -1

# 显示当前版本
alembic current

# 显示迁移历史
alembic history
```

## 连接池

### 什么是连接池？

重用数据库连接，而不是为每个查询创建新连接。

**优势：**
- 更快（无连接开销）
- 更少的资源使用
- 处理更多并发请求

### 配置池

```python
from sqlalchemy import create_engine

engine = create_engine(
    'postgresql://user:password@localhost/ml_database',
    pool_size=5,        # 保持 5 个连接打开
    max_overflow=10,    # 必要时允许 10 个额外连接
    pool_timeout=30,    # 等待可用连接最多 30 秒
    pool_recycle=3600,  # 1 小时后回收连接
)
```

### 池类型

```python
# QueuePool（大多数数据库默认）
from sqlalchemy.pool import QueuePool
engine = create_engine(..., poolclass=QueuePool)

# NullPool（无池化，每次新连接）
from sqlalchemy.pool import NullPool
engine = create_engine(..., poolclass=NullPool)

# StaticPool（单个连接，SQLite）
from sqlalchemy.pool import StaticPool
engine = create_engine('sqlite://', poolclass=StaticPool)
```

## 事务管理

### 基本事务

```python
with Session(engine) as session:
    try:
        # 一个事务中的多个操作
        model1 = Model(name="BERT", framework="PyTorch")
        model2 = Model(name="GPT-2", framework="PyTorch")

        session.add(model1)
        session.add(model2)

        session.commit()  # 要么全部保存，要么全部失败
    except Exception as e:
        session.rollback()  # 撤销所有更改
        raise e
```

### 显式事务

```python
with Session(engine) as session:
    with session.begin():  # 显式事务
        model = Model(name="BERT")
        session.add(model)
        # 如果没有异常则自动提交
        # 如果有异常则自动回滚
```

### 嵌套事务（保存点）

```python
with Session(engine) as session:
    with session.begin():
        model = Model(name="BERT")
        session.add(model)

        with session.begin_nested():  # 保存点
            exp = Experiment(model_id=1, accuracy=0.92)
            session.add(exp)
            # 如果需要可以回滚到此点

        session.commit()
```

## ML 应用的 ORM 模式

### 模式 1：模型注册

```python
class MLModel(Base):
    __tablename__ = 'ml_models'

    id = Column(Integer, primary_key=True)
    name = Column(String(100), unique=True)
    framework = Column(String(50))
    model_path = Column(String(255))
    accuracy = Column(Float)
    created_at = Column(DateTime, default=datetime.utcnow)
    status = Column(String(20), default='training')

    versions = relationship("ModelVersion", back_populates="model")
    experiments = relationship("Experiment", back_populates="model")

class ModelVersion(Base):
    __tablename__ = 'model_versions'

    id = Column(Integer, primary_key=True)
    model_id = Column(Integer, ForeignKey('ml_models.id'))
    version = Column(String(20))
    created_at = Column(DateTime, default=datetime.utcnow)

    model = relationship("MLModel", back_populates="versions")

# 用法
def register_model(session, name, framework, model_path):
    model = MLModel(
        name=name,
        framework=framework,
        model_path=model_path,
        status='ready'
    )
    session.add(model)
    session.commit()
    return model

def get_latest_model(session, name):
    return session.query(MLModel).filter_by(
        name=name,
        status='deployed'
    ).order_by(MLModel.created_at.desc()).first()
```

### 模式 2：实验跟踪

```python
class Experiment(Base):
    __tablename__ = 'experiments'

    id = Column(Integer, primary_key=True)
    name = Column(String(100))
    model_id = Column(Integer, ForeignKey('ml_models.id'))
    dataset = Column(String(100))
    created_at = Column(DateTime, default=datetime.utcnow)

    model = relationship("MLModel", back_populates="experiments")
    hyperparameters = relationship("Hyperparameter", back_populates="experiment")
    metrics = relationship("Metric", back_populates="experiment")

class Hyperparameter(Base):
    __tablename__ = 'hyperparameters'

    id = Column(Integer, primary_key=True)
    experiment_id = Column(Integer, ForeignKey('experiments.id'))
    name = Column(String(50))
    value = Column(String(100))

    experiment = relationship("Experiment", back_populates="hyperparameters")

class Metric(Base):
    __tablename__ = 'metrics'

    id = Column(Integer, primary_key=True)
    experiment_id = Column(Integer, ForeignKey('experiments.id'))
    name = Column(String(50))
    value = Column(Float)
    epoch = Column(Integer)

    experiment = relationship("Experiment", back_populates="metrics")

# 用法
def log_experiment(session, model_id, dataset, hyperparams, metrics):
    exp = Experiment(model_id=model_id, dataset=dataset)

    for name, value in hyperparams.items():
        hp = Hyperparameter(name=name, value=str(value))
        exp.hyperparameters.append(hp)

    for name, value in metrics.items():
        m = Metric(name=name, value=value)
        exp.metrics.append(m)

    session.add(exp)
    session.commit()
    return exp
```

### 模式 3：仓库模式

```python
class ModelRepository:
    def __init__(self, session):
        self.session = session

    def get_by_id(self, model_id):
        return self.session.query(MLModel).get(model_id)

    def get_by_name(self, name):
        return self.session.query(MLModel).filter_by(name=name).first()

    def get_all(self, framework=None, status=None):
        query = self.session.query(MLModel)
        if framework:
            query = query.filter_by(framework=framework)
        if status:
            query = query.filter_by(status=status)
        return query.all()

    def create(self, model):
        self.session.add(model)
        self.session.commit()
        return model

    def update(self, model):
        self.session.commit()
        return model

    def delete(self, model):
        self.session.delete(model)
        self.session.commit()

# 用法
with Session(engine) as session:
    repo = ModelRepository(session)

    # 创建
    model = MLModel(name="BERT", framework="PyTorch")
    repo.create(model)

    # 获取
    model = repo.get_by_name("BERT")

    # 更新
    model.accuracy = 0.95
    repo.update(model)

    # 删除
    repo.delete(model)
```

## 最佳实践

### 1. 使用上下文管理器

```python
# 好
with Session(engine) as session:
    model = session.query(Model).first()
    # 会话自动关闭

# 避免
session = Session(engine)
model = session.query(Model).first()
session.close()  # 容易忘记！
```

### 2. 使用事务

```python
# 好
with Session(engine) as session:
    try:
        model = Model(name="BERT")
        session.add(model)
        session.commit()
    except:
        session.rollback()
        raise

# 更好
with Session(engine) as session:
    with session.begin():
        model = Model(name="BERT")
        session.add(model)
        # 自动提交或回滚
```

### 3. 避免 N+1 查询

```python
# 不好（N+1 查询）
models = session.query(Model).all()
for model in models:
    for exp in model.experiments:  # 每个模型一次查询！
        print(exp.accuracy)

# 好（2 次查询）
models = session.query(Model).options(selectinload(Model.experiments)).all()
for model in models:
    for exp in model.experiments:
        print(exp.accuracy)
```

### 4. 使用批量操作

```python
# 慢
for i in range(1000):
    model = Model(name=f"model_{i}")
    session.add(model)
    session.commit()  # 1000 次提交！

# 快
models = [Model(name=f"model_{i}") for i in range(1000)]
session.bulk_save_objects(models)
session.commit()  # 1 次提交
```

### 5. 关闭会话

```python
# 始终关闭会话
with Session(engine) as session:
    # 执行工作
    pass  # 自动关闭

# 对于长时间运行的应用，使用作用域会话
from sqlalchemy.orm import scoped_session

SessionLocal = scoped_session(sessionmaker(bind=engine))

# 访问会话
session = SessionLocal()
# 使用会话
SessionLocal.remove()  # 关闭会话
```

## 总结与关键要点

### 我们学到的

1. **ORM** 自动将对象映射到数据库表
2. **SQLAlchemy** 是标准的 Python ORM
3. **模型**是表示表的 Python 类
4. **会话**管理事务和查询
5. **关系**连接模型（1:1、1:N、M:N）
6. **迁移**使用 Alembic 跟踪 schema 变化
7. **连接池**为性能重用连接
8. **事务**确保数据完整性

### ORM vs 原始 SQL

| 方面 | ORM | 原始 SQL |
|--------|-----|---------|
| 生产力 | 高 | 中 |
| 类型安全 | 是 | 否 |
| 可读性 | 高 | 中 |
| 性能 | 良好 | 最佳 |
| 复杂性 | 中 | 低 |
| 可移植性 | 高 | 低 |

**经验法则：** 90% 的操作使用 ORM，复杂/性能关键的查询使用原始 SQL。

### 何时使用哪种

**使用 ORM 的情况：**
- CRUD 操作
- 标准查询
- 类型安全重要
- 数据库可移植性重要

**使用原始 SQL 的情况：**
- 复杂聚合
- 批量操作
- 性能关键查询
- 需要数据库特定特性

### 后续步骤

你现在拥有 ML 基础设施所需的所有数据库知识：
- SQL 基础和高级查询
- 数据库设计和范式化
- 查询优化
- ORM 集成

**通过构建以下内容进行练习：**
- 模型注册应用
- 实验跟踪系统
- 预测日志记录服务
- 数据集管理工具

---

**预计学习时间：** 4-5 小时
**动手练习：** 完成练习 04：SQLAlchemy ORM
**最终评估：** 模块测验涵盖所有 4 个讲座
