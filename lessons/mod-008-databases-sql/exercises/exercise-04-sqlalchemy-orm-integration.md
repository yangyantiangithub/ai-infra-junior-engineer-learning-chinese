# Exercise 04: SQLAlchemy ORM 集成

## 概述

使用 SQLAlchemy ORM 将数据库设计与 Python 应用程序桥接。你将建模 ML 注册领域，实现查询模式，并公开准备用于 FastAPI 集成的服务层。本练习将练习 02 中的 SQL schema 转换为生产级 Python 包，包含 ORM 模型、仓库、迁移和全面测试。

**难度：** 中级 → 高级
**预计时间：** 3-4 小时
**前置条件：**
- 已完成之前的练习（schema 设计 + 高级 SQL）
- 讲座 04（ORM 与数据库集成）
- Python 3.9+，虚拟环境基础
- 熟悉类型提示和现代 Python 模式

## 学习目标

完成本练习后，你将能够：

1. **将关系表映射到 SQLAlchemy ORM 模型**，具有适当的关系和约束
2. **配置会话和连接池**用于生产数据库访问
3. **实现仓库模式**，封装 CRUD 和复杂查询
4. **使用 Alembic 管理 schema 迁移**用于数据库演化
5. **为数据库交互编写全面测试**，使用 fixture 和 factory
6. **应用生产模式**，包括连接池、错误处理和日志记录
7. **与 FastAPI 集成**构建由数据库支持的 RESTful API

## 场景

你的 ML 基础设施团队需要一个可重用的 Python 包 `ml_registry_db`，供其他服务导入。该包将提供：

- **ORM 模型**，镜像练习 02 中的规范化 schema
- **仓库层**，包含 `get_latest_deployment(model_name)` 和 `record_training_run(...)` 等函数
- **迁移管理**，使用 Alembic 进行 schema 演化
- **CLI 工具**，使用种子数据引导数据库
- **测试套件**，验证映射和业务逻辑
- **FastAPI 集成**，展示真实世界 API 用法

该包将成为你 ML 模型注册服务的基础，使团队能够以编程方式跟踪模型、版本、部署和训练运行。

---

## 第 1 部分：项目脚手架与设置

### 步骤 1.1：创建项目结构

```bash
mkdir -p ml-registry-db/{src/ml_registry_db,alembic/versions,scripts,tests}
cd ml-registry-db
```

完整项目布局：

```
ml-registry-db/
├── pyproject.toml              # Poetry 依赖管理
├── README.md                   # 文档
├── .env.example                # 环境变量模板
├── .gitignore                  # Git 忽略规则
├── src/
│   └── ml_registry_db/
│       ├── __init__.py         # 包初始化
│       ├── config.py           # 配置管理
│       ├── db.py               # 数据库会话和引擎
│       ├── models.py           # SQLAlchemy ORM 模型
│       ├── repositories.py     # 仓库模式实现
│       ├── schemas.py          # 用于验证的 Pydantic 模型
│       └── cli.py              # 命令行界面
├── alembic/                    # 数据库迁移
│   ├── env.py                  # Alembic 环境配置
│   ├── script.py.mako          # 迁移模板
│   ├── alembic.ini             # Alembic 配置
│   └── versions/               # 迁移版本文件
├── scripts/
│   ├── seed_data.py            # 用测试数据填充数据库
│   ├── create_tables.py        # 初始化数据库 schema
│   └── reset_db.py             # 删除并重建数据库
└── tests/
    ├── __init__.py
    ├── conftest.py             # Pytest fixture
    ├── factories.py            # Factory Boy 测试数据生成器
    ├── test_models.py          # ORM 模型测试
    ├── test_repositories.py    # 仓库函数测试
    └── test_integration.py     # 端到端集成测试
```

### 步骤 1.2：初始化 Poetry 项目

创建 `pyproject.toml`：

```toml
[tool.poetry]
name = "ml-registry-db"
version = "0.1.0"
description = "ML Model Registry Database Package with SQLAlchemy ORM"
authors = ["Your Name <you@example.com>"]
readme = "README.md"
packages = [{include = "ml_registry_db", from = "src"}]

[tool.poetry.dependencies]
python = "^3.9"
SQLAlchemy = "^2.0"
alembic = "^1.12"
psycopg2-binary = "^2.9"
pydantic = "^2.5"
pydantic-settings = "^2.1"
python-dotenv = "^1.0"
typer = "^0.9"
rich = "^13.7"

[tool.poetry.group.dev.dependencies]
pytest = "^7.4"
pytest-asyncio = "^0.23"
pytest-cov = "^4.1"
factory-boy = "^3.3"
faker = "^21.0"
black = "^23.12"
ruff = "^0.1"
mypy = "^1.7"
sqlalchemy-stubs = "^0.4"

[tool.poetry.scripts]
ml-registry = "ml_registry_db.cli:app"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"

[tool.black]
line-length = 100
target-version = ['py39', 'py310', 'py311']

[tool.ruff]
line-length = 100
select = ["E", "F", "W", "I", "N", "UP"]

[tool.mypy]
python_version = "3.9"
warn_return_any = true
warn_unused_configs = true
disallow_untyped_defs = true
plugins = ["pydantic.mypy", "sqlalchemy.ext.mypy.plugin"]

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py"]
python_functions = ["test_*"]
addopts = "-v --cov=ml_registry_db --cov-report=term-missing"
```

### 步骤 1.3：环境配置

创建 `.env.example`：

```bash
# Database Configuration
DATABASE_URL=postgresql://ml_user:ml_password@localhost:5432/ml_registry
TEST_DATABASE_URL=postgresql://ml_user:ml_password@localhost:5432/ml_registry_test

# Database Connection Pool Settings
DB_POOL_SIZE=5
DB_MAX_OVERFLOW=10
DB_POOL_TIMEOUT=30
DB_POOL_RECYCLE=3600

# Application Settings
ENVIRONMENT=development
LOG_LEVEL=INFO
DEBUG=true

# Optional: Async database support
ASYNC_DATABASE_URL=postgresql+asyncpg://ml_user:ml_password@localhost:5432/ml_registry
```

复制到 `.env` 并自定义：

```bash
cp .env.example .env
# Edit .env with your actual credentials
```

### 步骤 1.4：安装依赖

```bash
curl -sSL https://install.python-poetry.org | python3 -
poetry install
poetry shell
```

### 步骤 1.5：创建 `.gitignore`

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
env/
venv/
ENV/
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg

# Virtual environments
.venv/
.poetry/

# Environment variables
.env
.env.local

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# Testing
.coverage
.pytest_cache/
htmlcov/

# Database
*.db
*.sqlite

# Alembic
alembic/versions/*.pyc

# OS
.DS_Store
Thumbs.db
```

### 检查点 1

✅ 验证你的设置：

```bash
poetry --version
poetry show  # 列出已安装依赖
ls -la src/ml_registry_db/  # 确认包结构
cat .env  # 检查环境变量（不要提交此文件！）
```

---

## 第 2 部分：配置管理

### 步骤 2.1：创建配置模块

创建 `src/ml_registry_db/config.py`：

```python
"""
Configuration management using Pydantic Settings.
Loads from environment variables and .env files.
"""
from functools import lru_cache
from typing import Optional

from pydantic import Field, PostgresDsn, field_validator
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Application settings with environment variable support."""

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # Database settings
    database_url: PostgresDsn = Field(
        ...,
        description="PostgreSQL database URL",
        examples=["postgresql://user:pass@localhost/db"],
    )

    test_database_url: Optional[PostgresDsn] = Field(
        None,
        description="Test database URL (defaults to database_url with _test suffix)",
    )

    # Connection pool settings
    db_pool_size: int = Field(
        default=5,
        ge=1,
        le=20,
        description="Number of connections to keep in the pool",
    )

    db_max_overflow: int = Field(
        default=10,
        ge=0,
        le=50,
        description="Maximum overflow connections beyond pool_size",
    )

    db_pool_timeout: int = Field(
        default=30,
        ge=1,
        le=300,
        description="Seconds to wait before timing out on connection checkout",
    )

    db_pool_recycle: int = Field(
        default=3600,
        ge=300,
        description="Seconds after which to recycle connections",
    )

    db_echo: bool = Field(
        default=False,
        description="Echo SQL statements to stdout (for debugging)",
    )

    # Application settings
    environment: str = Field(
        default="development",
        pattern="^(development|staging|production|test)$",
    )

    log_level: str = Field(
        default="INFO",
        pattern="^(DEBUG|INFO|WARNING|ERROR|CRITICAL)$",
    )

    debug: bool = Field(default=False)

    @field_validator("test_database_url", mode="before")
    @classmethod
    def build_test_url(cls, v: Optional[str], info) -> Optional[str]:
        if v is not None:
            return v
        db_url = info.data.get("database_url")
        if db_url:
            return str(db_url).rsplit("/", 1)[0] + "/ml_registry_test"
        return None

    @property
    def is_development(self) -> bool:
        return self.environment == "development"

    @property
    def is_production(self) -> bool:
        return self.environment == "production"

    @property
    def is_test(self) -> bool:
        return self.environment == "test"


@lru_cache()
def get_settings() -> Settings:
    return Settings()


settings = get_settings()
```

### 关键配置模式

**为什么要使用 Pydantic Settings？**

1. **类型安全**：自动验证和类型转换
2. **文档**：内置字段描述和示例
3. **默认值**：合理的默认值，可覆盖
4. **验证**：约束（ge、le、pattern）确保配置有效
5. **12-Factor App**：支持云部署的环境变量

**安全最佳实践：**

- ✅ 永远不要将 `.env` 文件提交到版本控制
- ✅ 为开发/暂存/生产使用不同的凭据
- ✅ 定期轮换数据库密码
- ✅ 在云环境中使用 IAM 身份验证（AWS RDS IAM、GCP Cloud SQL）
- ✅ 在 CI/CD 密钥中加密 DATABASE_URL

---

## 第 3 部分：数据库会话和引擎配置

### 步骤 3.1：创建数据库模块

创建 `src/ml_registry_db/db.py`：

```python
"""
Database engine, session factory, and connection management.
Provides context managers for session lifecycle.
"""
import logging
from contextlib import contextmanager
from typing import Generator

from sqlalchemy import create_engine, event, pool
from sqlalchemy.engine import Engine
from sqlalchemy.orm import Session, sessionmaker, declarative_base

from .config import settings

logger = logging.getLogger(__name__)

Base = declarative_base()


def create_db_engine(url: str = None, echo: bool = None) -> Engine:
    db_url = url or str(settings.database_url)
    sql_echo = echo if echo is not None else settings.db_echo

    engine = create_engine(
        db_url,
        echo=sql_echo,
        pool_size=settings.db_pool_size,
        max_overflow=settings.db_max_overflow,
        pool_timeout=settings.db_pool_timeout,
        pool_recycle=settings.db_pool_recycle,
        pool_pre_ping=True,
        poolclass=pool.QueuePool,
    )

    @event.listens_for(engine, "connect")
    def receive_connect(dbapi_conn, connection_record):
        logger.debug(f"New database connection established: {id(dbapi_conn)}")

    @event.listens_for(engine, "checkout")
    def receive_checkout(dbapi_conn, connection_record, connection_proxy):
        logger.debug(f"Connection checked out from pool: {id(dbapi_conn)}")

    return engine


_engine: Engine = None


def get_engine() -> Engine:
    global _engine
    if _engine is None:
        _engine = create_db_engine()
        logger.info("Database engine initialized")
    return _engine


SessionLocal = sessionmaker(
    bind=None,
    autocommit=False,
    autoflush=False,
    expire_on_commit=True,
)


def get_session_factory() -> sessionmaker:
    SessionLocal.configure(bind=get_engine())
    return SessionLocal


@contextmanager
def get_session() -> Generator[Session, None, None]:
    session = get_session_factory()()
    try:
        yield session
        session.commit()
    except Exception as e:
        session.rollback()
        logger.error(f"Session rollback due to exception: {e}")
        raise
    finally:
        session.close()


def get_db() -> Generator[Session, None, None]:
    session = get_session_factory()()
    try:
        yield session
    finally:
        session.close()


def init_db(engine: Engine = None) -> None:
    from . import models
    db_engine = engine or get_engine()
    Base.metadata.create_all(bind=db_engine)
    logger.info("Database schema initialized")


def drop_db(engine: Engine = None) -> None:
    from . import models
    db_engine = engine or get_engine()
    Base.metadata.drop_all(bind=db_engine)
    logger.warning("Database schema dropped")


def reset_db(engine: Engine = None) -> None:
    drop_db(engine)
    init_db(engine)
    logger.info("Database reset complete")


def get_pool_status() -> dict:
    engine = get_engine()
    pool_obj = engine.pool
    return {
        "pool_size": pool_obj.size(),
        "checked_in": pool_obj.checkedin(),
        "checked_out": pool_obj.checkedout(),
        "overflow": pool_obj.overflow(),
        "total_connections": pool_obj.size() + pool_obj.overflow(),
    }
```

### 理解连接池

**为什么要使用连接池？**

创建数据库连接成本高昂（TCP 握手、身份验证等）。连接池维护一组可重用的连接。

**关键参数：**

- **pool_size=5**：保持 5 个持久连接打开
- **max_overflow=10**：在负载下允许最多 10 个额外临时连接（总计：15）
- **pool_timeout=30**：等待可用连接最多 30 秒后引发错误
- **pool_recycle=3600**：1 小时后关闭并替换连接（防止过期连接）
- **pool_pre_ping=True**：使用前测试连接健康（检测断开的连接）

---

## 第 4 部分：带关系的 ORM 模型

### 步骤 4.1：创建 ORM 模型

创建 `src/ml_registry_db/models.py`：

```python
"""
SQLAlchemy ORM models for ML Model Registry.
Maps to the normalized schema from Exercise 02.
"""
from datetime import datetime
from typing import List, Optional
from uuid import UUID, uuid4

from sqlalchemy import (
    Boolean, CheckConstraint, Column, DateTime,
    Enum as SQLEnum, Float, ForeignKey, Index,
    Integer, JSON, Numeric, String, Table, Text, UniqueConstraint,
)
from sqlalchemy.dialects.postgresql import JSONB, UUID as PGUUID
from sqlalchemy.orm import Mapped, mapped_column, relationship
from sqlalchemy.sql import func

from .db import Base

import enum


class FrameworkEnum(str, enum.Enum):
    PYTORCH = "pytorch"
    TENSORFLOW = "tensorflow"
    SKLEARN = "sklearn"
    XGBOOST = "xgboost"
    JAX = "jax"
    MXNET = "mxnet"


class RunStatusEnum(str, enum.Enum):
    QUEUED = "queued"
    RUNNING = "running"
    SUCCEEDED = "succeeded"
    FAILED = "failed"
    CANCELLED = "cancelled"
    TIMEOUT = "timeout"


class DeploymentStatusEnum(str, enum.Enum):
    PENDING = "pending"
    ACTIVE = "active"
    FAILED = "failed"
    ROLLED_BACK = "rolled_back"
    DECOMMISSIONED = "decommissioned"


class ApprovalStatusEnum(str, enum.Enum):
    PENDING = "pending"
    APPROVED = "approved"
    REJECTED = "rejected"
    REVOKED = "revoked"


# Junction Tables
model_tags = Table(
    "model_tags", Base.metadata,
    Column("model_id", PGUUID(as_uuid=True), ForeignKey("models.model_id", ondelete="CASCADE")),
    Column("tag_id", Integer, ForeignKey("tags.tag_id", ondelete="CASCADE")),
    Column("tagged_at", DateTime(timezone=True), server_default=func.now()),
    UniqueConstraint("model_id", "tag_id", name="unique_model_tag"),
)

dataset_tags = Table(
    "dataset_tags", Base.metadata,
    Column("dataset_id", PGUUID(as_uuid=True), ForeignKey("datasets.dataset_id", ondelete="CASCADE")),
    Column("tag_id", Integer, ForeignKey("tags.tag_id", ondelete="CASCADE")),
    Column("tagged_at", DateTime(timezone=True), server_default=func.now()),
    UniqueConstraint("dataset_id", "tag_id", name="unique_dataset_tag"),
)

run_datasets = Table(
    "run_datasets", Base.metadata,
    Column("run_id", PGUUID(as_uuid=True), ForeignKey("training_runs.run_id", ondelete="CASCADE")),
    Column("dataset_id", PGUUID(as_uuid=True), ForeignKey("datasets.dataset_id", ondelete="CASCADE")),
    Column("split_type", String(20), nullable=False),
    Column("linked_at", DateTime(timezone=True), server_default=func.now()),
    CheckConstraint("split_type IN ('train', 'validation', 'test')", name="valid_split_type"),
)


class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), nullable=False)
    updated_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now(), nullable=False)


class UUIDPrimaryKeyMixin:
    id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)


class Team(Base):
    __tablename__ = "teams"
    team_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    team_name: Mapped[str] = mapped_column(String(100), unique=True, nullable=False)
    team_email: Mapped[str] = mapped_column(String(255), nullable=False)
    team_slack_channel: Mapped[Optional[str]] = mapped_column(String(100))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    models: Mapped[List["Model"]] = relationship("Model", back_populates="team")


class Environment(Base):
    __tablename__ = "environments"
    environment_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    environment_name: Mapped[str] = mapped_column(String(50), unique=True, nullable=False)
    description: Mapped[Optional[str]] = mapped_column(Text)
    deployments: Mapped[List["Deployment"]] = relationship("Deployment", back_populates="environment")
    __table_args__ = (CheckConstraint("environment_name IN ('development', 'staging', 'production', 'canary', 'shadow')", name="valid_environment"),)


class Model(Base, TimestampMixin):
    __tablename__ = "models"
    model_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    model_name: Mapped[str] = mapped_column(String(255), unique=True, nullable=False, index=True)
    description: Mapped[Optional[str]] = mapped_column(Text)
    team_id: Mapped[int] = mapped_column(Integer, ForeignKey("teams.team_id", ondelete="SET NULL"))
    is_active: Mapped[bool] = mapped_column(Boolean, default=True, nullable=False)
    team: Mapped["Team"] = relationship("Team", back_populates="models")
    versions: Mapped[List["ModelVersion"]] = relationship("ModelVersion", back_populates="model", cascade="all, delete-orphan", order_by="ModelVersion.registered_at.desc()")
    tags: Mapped[List["Tag"]] = relationship("Tag", secondary=model_tags, back_populates="models")


class ModelVersion(Base, TimestampMixin):
    __tablename__ = "model_versions"
    version_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    model_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), ForeignKey("models.model_id", ondelete="CASCADE"), nullable=False)
    semver: Mapped[str] = mapped_column(String(50), nullable=False)
    artifact_uri: Mapped[str] = mapped_column(String(500), nullable=False)
    metadata_: Mapped[dict] = mapped_column("metadata", JSONB, default=dict)
    registered_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    registered_by: Mapped[Optional[str]] = mapped_column(String(255))
    model: Mapped["Model"] = relationship("Model", back_populates="versions")
    deployments: Mapped[List["Deployment"]] = relationship("Deployment", back_populates="version", cascade="all, delete-orphan")
    approvals: Mapped[List["Approval"]] = relationship("Approval", back_populates="version", cascade="all, delete-orphan")
    training_runs: Mapped[List["TrainingRun"]] = relationship("TrainingRun", back_populates="model_version")
    __table_args__ = (UniqueConstraint("model_id", "semver", name="unique_model_version"), Index("idx_model_versions_registered_at", "registered_at"))


class TrainingRun(Base, TimestampMixin):
    __tablename__ = "training_runs"
    run_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    model_name: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    framework: Mapped[str] = mapped_column(SQLEnum(FrameworkEnum, name="framework_enum", create_type=False), nullable=False)
    experiment_name: Mapped[str] = mapped_column(String(255), nullable=False, index=True)
    run_name: Mapped[Optional[str]] = mapped_column(String(255))
    status: Mapped[str] = mapped_column(SQLEnum(RunStatusEnum, name="run_status_enum", create_type=False), nullable=False, default=RunStatusEnum.QUEUED, index=True)
    accuracy: Mapped[Optional[float]] = mapped_column(Numeric(5, 4))
    loss: Mapped[Optional[float]] = mapped_column(Numeric(8, 5))
    f1_score: Mapped[Optional[float]] = mapped_column(Numeric(5, 4))
    precision: Mapped[Optional[float]] = mapped_column(Numeric(5, 4))
    recall: Mapped[Optional[float]] = mapped_column(Numeric(5, 4))
    hyperparameters: Mapped[dict] = mapped_column(JSONB, default=dict)
    metrics: Mapped[dict] = mapped_column(JSONB, default=dict)
    tags_json: Mapped[dict] = mapped_column("tags", JSONB, default=dict)
    started_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    finished_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    duration_seconds: Mapped[Optional[int]] = mapped_column(Integer)
    git_commit_hash: Mapped[Optional[str]] = mapped_column(String(40))
    run_by: Mapped[Optional[str]] = mapped_column(String(255))
    artifact_uri: Mapped[Optional[str]] = mapped_column(String(500))
    version_id: Mapped[Optional[UUID]] = mapped_column(PGUUID(as_uuid=True), ForeignKey("model_versions.version_id", ondelete="SET NULL"))
    model_version: Mapped[Optional["ModelVersion"]] = relationship("ModelVersion", back_populates="training_runs")
    datasets: Mapped[List["Dataset"]] = relationship("Dataset", secondary=run_datasets, back_populates="training_runs")
    __table_args__ = (Index("idx_training_runs_experiment_status", "experiment_name", "status"), Index("idx_training_runs_started_at", "started_at"), CheckConstraint("accuracy IS NULL OR (accuracy >= 0 AND accuracy <= 1)", name="valid_accuracy"), CheckConstraint("loss IS NULL OR loss >= 0", name="valid_loss"))


class Deployment(Base, TimestampMixin):
    __tablename__ = "deployments"
    deployment_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    version_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), ForeignKey("model_versions.version_id", ondelete="RESTRICT"), nullable=False)
    environment_id: Mapped[int] = mapped_column(Integer, ForeignKey("environments.environment_id", ondelete="RESTRICT"), nullable=False)
    status: Mapped[str] = mapped_column(SQLEnum(DeploymentStatusEnum, name="deployment_status_enum", create_type=False), nullable=False, default=DeploymentStatusEnum.PENDING)
    deployed_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    deployed_by: Mapped[str] = mapped_column(String(255), nullable=False)
    endpoint_url: Mapped[Optional[str]] = mapped_column(String(500))
    replicas: Mapped[int] = mapped_column(Integer, default=1, nullable=False)
    cpu_limit: Mapped[Optional[str]] = mapped_column(String(20))
    memory_limit: Mapped[Optional[str]] = mapped_column(String(20))
    gpu_count: Mapped[int] = mapped_column(Integer, default=0)
    config: Mapped[dict] = mapped_column(JSONB, default=dict)
    rollback_version_id: Mapped[Optional[UUID]] = mapped_column(PGUUID(as_uuid=True))
    version: Mapped["ModelVersion"] = relationship("ModelVersion", back_populates="deployments")
    environment: Mapped["Environment"] = relationship("Environment", back_populates="deployments")
    __table_args__ = (Index("idx_deployments_environment_status", "environment_id", "status"), Index("idx_deployments_deployed_at", "deployed_at"), CheckConstraint("replicas >= 0", name="valid_replicas"), CheckConstraint("gpu_count >= 0", name="valid_gpu_count"))


class Dataset(Base, TimestampMixin):
    __tablename__ = "datasets"
    dataset_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    dataset_name: Mapped[str] = mapped_column(String(255), unique=True, nullable=False, index=True)
    description: Mapped[Optional[str]] = mapped_column(Text)
    storage_uri: Mapped[str] = mapped_column(String(500), nullable=False)
    size_bytes: Mapped[Optional[int]] = mapped_column(Integer)
    row_count: Mapped[Optional[int]] = mapped_column(Integer)
    schema_: Mapped[dict] = mapped_column("schema", JSONB, default=dict)
    checksum: Mapped[Optional[str]] = mapped_column(String(64))
    training_runs: Mapped[List["TrainingRun"]] = relationship("TrainingRun", secondary=run_datasets, back_populates="datasets")
    tags: Mapped[List["Tag"]] = relationship("Tag", secondary=dataset_tags, back_populates="datasets")


class Approval(Base):
    __tablename__ = "approvals"
    approval_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, default=uuid4)
    version_id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), ForeignKey("model_versions.version_id", ondelete="CASCADE"), nullable=False)
    approver: Mapped[str] = mapped_column(String(255), nullable=False)
    status: Mapped[str] = mapped_column(SQLEnum(ApprovalStatusEnum, name="approval_status_enum", create_type=False), nullable=False, default=ApprovalStatusEnum.PENDING)
    comments: Mapped[Optional[str]] = mapped_column(Text)
    approved_at: Mapped[Optional[datetime]] = mapped_column(DateTime(timezone=True))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    version: Mapped["ModelVersion"] = relationship("ModelVersion", back_populates="approvals")
    __table_args__ = (Index("idx_approvals_status", "status"),)


class Tag(Base):
    __tablename__ = "tags"
    tag_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    tag_name: Mapped[str] = mapped_column(String(100), unique=True, nullable=False, index=True)
    category: Mapped[Optional[str]] = mapped_column(String(50))
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
    models: Mapped[List["Model"]] = relationship("Model", secondary=model_tags, back_populates="tags")
    datasets: Mapped[List["Dataset"]] = relationship("Dataset", secondary=dataset_tags, back_populates="tags")
```

### 理解 SQLAlchemy 2.0 模式

**使用的关键特性：**

1. **Mapped[Type] 注解**：更好的 IDE 支持和 mypy 检查的类型提示
2. **mapped_column()**：新的 SQLAlchemy 2.0 API，替代 Column()
3. **relationship()**：定义 ORM 关系（一对多、多对多）
4. **Mixins**：用于 DRY 代码的可重用模式（TimestampMixin）
5. **Enums**：类型安全的状态字段
6. **JSONB**：PostgreSQL 特定的 JSON 存储，支持索引
7. **Cascade Rules**：控制删除行为（CASCADE、RESTRICT、SET NULL）
8. **Indexes**：优化查询性能

---

## 第 5 部分：用于 API 验证的 Pydantic Schema

### 步骤 5.1：创建 Pydantic Schema

创建 `src/ml_registry_db/schemas.py`：

```python
"""
Pydantic models for request/response validation.
Used for FastAPI integration and data validation.
"""
from datetime import datetime
from typing import Dict, List, Optional
from uuid import UUID

from pydantic import BaseModel, ConfigDict, Field, field_validator


class BaseSchema(BaseModel):
    model_config = ConfigDict(from_attributes=True, populate_by_name=True)


class TeamBase(BaseSchema):
    team_name: str = Field(..., max_length=100, examples=["ml-platform-team"])
    team_email: str = Field(..., max_length=255, examples=["ml-team@company.com"])
    team_slack_channel: Optional[str] = Field(None, max_length=100, examples=["#ml-platform"])


class TeamCreate(TeamBase):
    pass


class TeamRead(TeamBase):
    team_id: int
    created_at: datetime


class ModelBase(BaseSchema):
    model_name: str = Field(..., max_length=255, examples=["fraud-detection-v1"])
    description: Optional[str] = Field(None, examples=["Fraud detection model using XGBoost"])
    team_id: Optional[int] = None
    is_active: bool = True


class ModelCreate(ModelBase):
    pass


class ModelUpdate(BaseSchema):
    description: Optional[str] = None
    team_id: Optional[int] = None
    is_active: Optional[bool] = None


class ModelRead(ModelBase):
    model_id: UUID
    created_at: datetime
    updated_at: datetime


class ModelWithVersions(ModelRead):
    versions: List["ModelVersionRead"] = []


class ModelVersionBase(BaseSchema):
    semver: str = Field(..., pattern=r"^\d+\.\d+\.\d+$", examples=["1.0.0", "2.1.3"])
    artifact_uri: str = Field(..., max_length=500, examples=["s3://models/fraud-detection/v1.0.0"])
    metadata_: Dict = Field(default_factory=dict, alias="metadata")
    registered_by: Optional[str] = Field(None, max_length=255, examples=["alice@company.com"])


class ModelVersionCreate(ModelVersionBase):
    model_id: UUID


class ModelVersionRead(ModelVersionBase):
    version_id: UUID
    model_id: UUID
    registered_at: datetime
    created_at: datetime
    updated_at: datetime


class TrainingRunBase(BaseSchema):
    model_name: str = Field(..., max_length=255)
    framework: str = Field(..., pattern="^(pytorch|tensorflow|sklearn|xgboost|jax|mxnet)$")
    experiment_name: str = Field(..., max_length=255)
    run_name: Optional[str] = Field(None, max_length=255)
    hyperparameters: Dict = Field(default_factory=dict)
    tags_json: Dict = Field(default_factory=dict, alias="tags")
    git_commit_hash: Optional[str] = Field(None, pattern=r"^[a-f0-9]{40}$")
    run_by: Optional[str] = None


class TrainingRunCreate(TrainingRunBase):
    pass


class TrainingRunUpdate(BaseSchema):
    status: Optional[str] = Field(None, pattern="^(queued|running|succeeded|failed|cancelled|timeout)$")
    accuracy: Optional[float] = Field(None, ge=0, le=1)
    loss: Optional[float] = Field(None, ge=0)
    f1_score: Optional[float] = Field(None, ge=0, le=1)
    precision: Optional[float] = Field(None, ge=0, le=1)
    recall: Optional[float] = Field(None, ge=0, le=1)
    metrics: Optional[Dict] = None
    started_at: Optional[datetime] = None
    finished_at: Optional[datetime] = None
    duration_seconds: Optional[int] = Field(None, ge=0)
    artifact_uri: Optional[str] = None
    version_id: Optional[UUID] = None


class TrainingRunRead(TrainingRunBase):
    run_id: UUID
    status: str
    accuracy: Optional[float] = None
    loss: Optional[float] = None
    f1_score: Optional[float] = None
    precision: Optional[float] = None
    recall: Optional[float] = None
    metrics: Dict
    started_at: Optional[datetime] = None
    finished_at: Optional[datetime] = None
    duration_seconds: Optional[int] = None
    artifact_uri: Optional[str] = None
    version_id: Optional[UUID] = None
    created_at: datetime
    updated_at: datetime


class DeploymentBase(BaseSchema):
    version_id: UUID
    environment_id: int
    deployed_by: str = Field(..., max_length=255)
    endpoint_url: Optional[str] = Field(None, max_length=500)
    replicas: int = Field(default=1, ge=0)
    cpu_limit: Optional[str] = Field(None, max_length=20, examples=["500m", "2"])
    memory_limit: Optional[str] = Field(None, max_length=20, examples=["1Gi", "512Mi"])
    gpu_count: int = Field(default=0, ge=0)
    config: Dict = Field(default_factory=dict)


class DeploymentCreate(DeploymentBase):
    pass


class DeploymentUpdate(BaseSchema):
    status: Optional[str] = Field(None, pattern="^(pending|active|failed|rolled_back|decommissioned)$")
    replicas: Optional[int] = Field(None, ge=0)
    rollback_version_id: Optional[UUID] = None


class DeploymentRead(DeploymentBase):
    deployment_id: UUID
    status: str
    deployed_at: datetime
    rollback_version_id: Optional[UUID] = None
    created_at: datetime
    updated_at: datetime


class DatasetBase(BaseSchema):
    dataset_name: str = Field(..., max_length=255)
    description: Optional[str] = None
    storage_uri: str = Field(..., max_length=500, examples=["s3://datasets/fraud-training-2024"])
    size_bytes: Optional[int] = Field(None, ge=0)
    row_count: Optional[int] = Field(None, ge=0)
    schema_: Dict = Field(default_factory=dict, alias="schema")
    checksum: Optional[str] = Field(None, pattern=r"^[a-f0-9]{64}$")


class DatasetCreate(DatasetBase):
    pass


class DatasetRead(DatasetBase):
    dataset_id: UUID
    created_at: datetime
    updated_at: datetime


class ApprovalBase(BaseSchema):
    version_id: UUID
    approver: str = Field(..., max_length=255)
    comments: Optional[str] = None


class ApprovalCreate(ApprovalBase):
    pass


class ApprovalUpdate(BaseSchema):
    status: str = Field(..., pattern="^(pending|approved|rejected|revoked)$")
    comments: Optional[str] = None


class ApprovalRead(ApprovalBase):
    approval_id: UUID
    status: str
    approved_at: Optional[datetime] = None
    created_at: datetime


class TagBase(BaseSchema):
    tag_name: str = Field(..., max_length=100, examples=["production", "experimental"])
    category: Optional[str] = Field(None, max_length=50, examples=["environment", "team"])


class TagCreate(TagBase):
    pass


class TagRead(TagBase):
    tag_id: int
    created_at: datetime


ModelWithVersions.model_rebuild()
```

### 为什么选择 Pydantic？

**优势：**

1. **自动验证**：类型检查、字段约束、模式匹配
2. **序列化**：自动 JSON 编码/解码
3. **文档**：为 FastAPI 生成 OpenAPI schema
4. **类型安全**：完整的 mypy 和 IDE 支持
5. **ORM 集成**：`from_attributes=True` 将 SQLAlchemy 模型转换为 Pydantic

---

## 第 6 部分：仓库模式实现

### 步骤 6.1：创建仓库层

创建 `src/ml_registry_db/repositories.py`：

```python
"""
Repository layer for database operations.
Encapsulates CRUD logic and complex queries.
"""
import logging
from datetime import datetime
from typing import List, Optional
from uuid import UUID

from sqlalchemy import select, func, and_, or_, desc
from sqlalchemy.orm import Session, selectinload, joinedload
from sqlalchemy.exc import IntegrityError

from . import models, schemas

logger = logging.getLogger(__name__)


class RepositoryException(Exception):
    pass


class NotFoundError(RepositoryException):
    pass


class DuplicateError(RepositoryException):
    pass


class ValidationError(RepositoryException):
    pass


def create_team(session: Session, data: schemas.TeamCreate) -> models.Team:
    try:
        team = models.Team(**data.model_dump())
        session.add(team)
        session.flush()
        logger.info(f"Team created: {team.team_name}")
        return team
    except IntegrityError as e:
        raise DuplicateError(f"Team '{data.team_name}' already exists") from e


def get_team(session: Session, team_id: int) -> Optional[models.Team]:
    return session.get(models.Team, team_id)


def list_teams(session: Session, limit: int = 100) -> List[models.Team]:
    stmt = select(models.Team).limit(limit)
    return list(session.scalars(stmt))


def register_model(session: Session, data: schemas.ModelCreate) -> models.Model:
    if data.team_id:
        team = session.get(models.Team, data.team_id)
        if not team:
            raise ValidationError(f"Team {data.team_id} not found")
    try:
        model = models.Model(**data.model_dump())
        session.add(model)
        session.flush()
        logger.info(f"Model registered: {model.model_name} ({model.model_id})")
        return model
    except IntegrityError as e:
        raise DuplicateError(f"Model '{data.model_name}' already exists") from e


def get_model(session: Session, model_id: UUID) -> Optional[models.Model]:
    stmt = select(models.Model).where(models.Model.model_id == model_id).options(selectinload(models.Model.versions))
    return session.scalar(stmt)


def get_model_by_name(session: Session, model_name: str) -> Optional[models.Model]:
    stmt = select(models.Model).where(models.Model.model_name == model_name).options(selectinload(models.Model.versions))
    return session.scalar(stmt)


def list_models(session: Session, *, team_id: Optional[int] = None, is_active: Optional[bool] = None, limit: int = 50, offset: int = 0) -> List[models.Model]:
    stmt = select(models.Model).options(selectinload(models.Model.versions))
    if team_id is not None:
        stmt = stmt.where(models.Model.team_id == team_id)
    if is_active is not None:
        stmt = stmt.where(models.Model.is_active == is_active)
    stmt = stmt.order_by(desc(models.Model.created_at)).limit(limit).offset(offset)
    return list(session.scalars(stmt))


def update_model(session: Session, model_id: UUID, data: schemas.ModelUpdate) -> models.Model:
    model = get_model(session, model_id)
    if not model:
        raise NotFoundError(f"Model {model_id} not found")
    update_data = data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(model, field, value)
    session.flush()
    logger.info(f"Model updated: {model.model_name}")
    return model


def delete_model(session: Session, model_id: UUID) -> None:
    model = get_model(session, model_id)
    if not model:
        raise NotFoundError(f"Model {model_id} not found")
    session.delete(model)
    session.flush()
    logger.warning(f"Model deleted: {model.model_name} ({model_id})")


def add_model_version(session: Session, data: schemas.ModelVersionCreate) -> models.ModelVersion:
    model = get_model(session, data.model_id)
    if not model:
        raise NotFoundError(f"Model {data.model_id} not found")
    try:
        version = models.ModelVersion(**data.model_dump())
        session.add(version)
        session.flush()
        logger.info(f"Version registered: {model.model_name} v{version.semver}")
        return version
    except IntegrityError as e:
        raise DuplicateError(f"Version {data.semver} already exists for model {data.model_id}") from e


def get_model_version(session: Session, version_id: UUID) -> Optional[models.ModelVersion]:
    stmt = select(models.ModelVersion).where(models.ModelVersion.version_id == version_id).options(joinedload(models.ModelVersion.model), selectinload(models.ModelVersion.deployments))
    return session.scalar(stmt)


def get_latest_model_version(session: Session, model_name: str) -> Optional[models.ModelVersion]:
    stmt = select(models.ModelVersion).join(models.Model).where(models.Model.model_name == model_name).order_by(desc(models.ModelVersion.registered_at)).limit(1)
    return session.scalar(stmt)


def list_model_versions(session: Session, model_id: UUID, limit: int = 20) -> List[models.ModelVersion]:
    stmt = select(models.ModelVersion).where(models.ModelVersion.model_id == model_id).order_by(desc(models.ModelVersion.registered_at)).limit(limit)
    return list(session.scalars(stmt))


def record_training_run(session: Session, data: schemas.TrainingRunCreate) -> models.TrainingRun:
    run = models.TrainingRun(**data.model_dump())
    session.add(run)
    session.flush()
    logger.info(f"Training run recorded: {run.run_id} ({run.model_name})")
    return run


def get_training_run(session: Session, run_id: UUID) -> Optional[models.TrainingRun]:
    return session.get(models.TrainingRun, run_id)


def update_training_run(session: Session, run_id: UUID, data: schemas.TrainingRunUpdate) -> models.TrainingRun:
    run = get_training_run(session, run_id)
    if not run:
        raise NotFoundError(f"Training run {run_id} not found")
    update_data = data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(run, field, value)
    session.flush()
    logger.info(f"Training run updated: {run_id} (status={run.status})")
    return run


def list_training_runs(session: Session, *, model_name: Optional[str] = None, experiment_name: Optional[str] = None, status: Optional[str] = None, limit: int = 50, offset: int = 0) -> List[models.TrainingRun]:
    stmt = select(models.TrainingRun)
    if model_name:
        stmt = stmt.where(models.TrainingRun.model_name == model_name)
    if experiment_name:
        stmt = stmt.where(models.TrainingRun.experiment_name == experiment_name)
    if status:
        stmt = stmt.where(models.TrainingRun.status == status)
    stmt = stmt.order_by(desc(models.TrainingRun.created_at)).limit(limit).offset(offset)
    return list(session.scalars(stmt))


def get_best_training_run(session: Session, model_name: str, metric: str = "accuracy", maximize: bool = True) -> Optional[models.TrainingRun]:
    stmt = select(models.TrainingRun).where(and_(models.TrainingRun.model_name == model_name, models.TrainingRun.status == "succeeded"))
    metric_col = getattr(models.TrainingRun, metric, None)
    if not metric_col:
        raise ValidationError(f"Invalid metric: {metric}")
    if maximize:
        stmt = stmt.order_by(desc(metric_col))
    else:
        stmt = stmt.order_by(metric_col)
    stmt = stmt.limit(1)
    return session.scalar(stmt)


def create_deployment(session: Session, data: schemas.DeploymentCreate) -> models.Deployment:
    version = get_model_version(session, data.version_id)
    if not version:
        raise NotFoundError(f"Model version {data.version_id} not found")
    env = session.get(models.Environment, data.environment_id)
    if not env:
        raise NotFoundError(f"Environment {data.environment_id} not found")
    deployment = models.Deployment(**data.model_dump())
    session.add(deployment)
    session.flush()
    logger.info(f"Deployment created: {deployment.deployment_id} to {env.environment_name}")
    return deployment


def get_deployment(session: Session, deployment_id: UUID) -> Optional[models.Deployment]:
    stmt = select(models.Deployment).where(models.Deployment.deployment_id == deployment_id).options(joinedload(models.Deployment.version).joinedload(models.ModelVersion.model), joinedload(models.Deployment.environment))
    return session.scalar(stmt)


def update_deployment(session: Session, deployment_id: UUID, data: schemas.DeploymentUpdate) -> models.Deployment:
    deployment = get_deployment(session, deployment_id)
    if not deployment:
        raise NotFoundError(f"Deployment {deployment_id} not found")
    update_data = data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(deployment, field, value)
    session.flush()
    logger.info(f"Deployment updated: {deployment_id} (status={deployment.status})")
    return deployment


def get_latest_deployment(session: Session, model_name: str, environment_name: str = "production") -> Optional[models.Deployment]:
    stmt = select(models.Deployment).join(models.ModelVersion).join(models.Model).join(models.Environment).where(and_(models.Model.model_name == model_name, models.Environment.environment_name == environment_name, models.Deployment.status == "active")).order_by(desc(models.Deployment.deployed_at)).limit(1)
    return session.scalar(stmt)


def list_deployments(session: Session, *, model_name: Optional[str] = None, environment_id: Optional[int] = None, status: Optional[str] = None, limit: int = 50) -> List[models.Deployment]:
    stmt = select(models.Deployment).options(joinedload(models.Deployment.version).joinedload(models.ModelVersion.model), joinedload(models.Deployment.environment))
    if model_name:
        stmt = stmt.join(models.ModelVersion).join(models.Model).where(models.Model.model_name == model_name)
    if environment_id:
        stmt = stmt.where(models.Deployment.environment_id == environment_id)
    if status:
        stmt = stmt.where(models.Deployment.status == status)
    stmt = stmt.order_by(desc(models.Deployment.deployed_at)).limit(limit)
    return list(session.scalars(stmt))


def register_dataset(session: Session, data: schemas.DatasetCreate) -> models.Dataset:
    try:
        dataset = models.Dataset(**data.model_dump())
        session.add(dataset)
        session.flush()
        logger.info(f"Dataset registered: {dataset.dataset_name}")
        return dataset
    except IntegrityError as e:
        raise DuplicateError(f"Dataset '{data.dataset_name}' already exists") from e


def get_dataset(session: Session, dataset_id: UUID) -> Optional[models.Dataset]:
    return session.get(models.Dataset, dataset_id)


def get_dataset_by_name(session: Session, dataset_name: str) -> Optional[models.Dataset]:
    stmt = select(models.Dataset).where(models.Dataset.dataset_name == dataset_name)
    return session.scalar(stmt)


def list_datasets(session: Session, limit: int = 50) -> List[models.Dataset]:
    stmt = select(models.Dataset).order_by(desc(models.Dataset.created_at)).limit(limit)
    return list(session.scalars(stmt))


def create_approval(session: Session, data: schemas.ApprovalCreate) -> models.Approval:
    version = get_model_version(session, data.version_id)
    if not version:
        raise NotFoundError(f"Model version {data.version_id} not found")
    approval = models.Approval(**data.model_dump())
    session.add(approval)
    session.flush()
    logger.info(f"Approval created: {approval.approval_id} for version {data.version_id}")
    return approval


def update_approval(session: Session, approval_id: UUID, data: schemas.ApprovalUpdate) -> models.Approval:
    approval = session.get(models.Approval, approval_id)
    if not approval:
        raise NotFoundError(f"Approval {approval_id} not found")
    if data.status == "approved" and approval.status != "approved":
        approval.approved_at = datetime.utcnow()
    update_data = data.model_dump(exclude_unset=True)
    for field, value in update_data.items():
        setattr(approval, field, value)
    session.flush()
    logger.info(f"Approval updated: {approval_id} (status={approval.status})")
    return approval


def list_approvals(session: Session, version_id: UUID) -> List[models.Approval]:
    stmt = select(models.Approval).where(models.Approval.version_id == version_id).order_by(desc(models.Approval.created_at))
    return list(session.scalars(stmt))


def create_tag(session: Session, data: schemas.TagCreate) -> models.Tag:
    try:
        tag = models.Tag(**data.model_dump())
        session.add(tag)
        session.flush()
        logger.info(f"Tag created: {tag.tag_name}")
        return tag
    except IntegrityError as e:
        raise DuplicateError(f"Tag '{data.tag_name}' already exists") from e


def get_tag_by_name(session: Session, tag_name: str) -> Optional[models.Tag]:
    stmt = select(models.Tag).where(models.Tag.tag_name == tag_name)
    return session.scalar(stmt)


def list_tags(session: Session, category: Optional[str] = None) -> List[models.Tag]:
    stmt = select(models.Tag)
    if category:
        stmt = stmt.where(models.Tag.category == category)
    return list(session.scalars(stmt))


def add_tag_to_model(session: Session, model_id: UUID, tag_name: str) -> None:
    model = get_model(session, model_id)
    if not model:
        raise NotFoundError(f"Model {model_id} not found")
    tag = get_tag_by_name(session, tag_name)
    if not tag:
        tag = create_tag(session, schemas.TagCreate(tag_name=tag_name))
    if tag not in model.tags:
        model.tags.append(tag)
        session.flush()
        logger.info(f"Tag '{tag_name}' added to model {model.model_name}")


def remove_tag_from_model(session: Session, model_id: UUID, tag_name: str) -> None:
    model = get_model(session, model_id)
    if not model:
        raise NotFoundError(f"Model {model_id} not found")
    tag = get_tag_by_name(session, tag_name)
    if tag and tag in model.tags:
        model.tags.remove(tag)
        session.flush()
        logger.info(f"Tag '{tag_name}' removed from model {model.model_name}")
```

### 仓库模式的优势

**为什么使用仓库？**

1. **关注点分离**：业务逻辑与数据库访问分离
2. **可测试性**：在单元测试中易于模拟仓库
3. **可重用性**：常见查询封装在函数中
4. **一致性**：标准化的错误处理和日志记录
5. **可维护性**：查询更改集中在一处

**关键模式：**

- **即时加载**：使用 `selectinload()` 和 `joinedload()` 避免 N+1 查询
- **自定义异常**：清晰的错误类型（NotFoundError、DuplicateError、ValidationError）
- **过滤**：灵活查询的可选参数
- **分页**：大型结果集的 limit/offset
- **事务管理**：函数内 flush()，调用代码中 commit()

---

## 第 7 部分：Alembic 数据库迁移

### 步骤 7.1：初始化 Alembic

```bash
alembic init alembic
```

### 步骤 7.2：配置 Alembic

编辑 `alembic/env.py`：

```python
"""Alembic environment configuration."""
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context
from ml_registry_db.db import Base
from ml_registry_db.config import settings
import ml_registry_db.models

config = context.config
if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata
config.set_main_option("sqlalchemy.url", str(settings.database_url))


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(url=url, target_metadata=target_metadata, literal_binds=True, dialect_opts={"paramstyle": "named"})
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    connectable = engine_from_config(config.get_section(config.config_ini_section), prefix="sqlalchemy.", poolclass=pool.NullPool)
    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### 步骤 7.3：创建初始迁移

```bash
alembic revision --autogenerate -m "create initial schema"
alembic upgrade head
alembic current
```

### 步骤 7.4：常用迁移命令

```bash
alembic revision --autogenerate -m "add new column"
alembic upgrade head
alembic downgrade -1
alembic history --verbose
alembic downgrade abc123
alembic current
alembic upgrade head --sql
alembic stamp head
```

### Schema 演化示例

假设你要向 training runs 添加一个 `priority` 字段：

```python
# 在 models.py 中，向 TrainingRun 添加：
priority: Mapped[int] = mapped_column(Integer, default=5, nullable=False)
```

```bash
alembic revision --autogenerate -m "add priority to training runs"
alembic upgrade head
```

---

## 第 8 部分：使用 Pytest 进行测试

### 步骤 8.1：创建测试 fixture

创建 `tests/conftest.py`：

```python
"""Pytest configuration and fixtures."""
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import Session, sessionmaker
from ml_registry_db.db import Base
from ml_registry_db.config import settings


@pytest.fixture(scope="session")
def test_engine():
    test_url = str(settings.test_database_url)
    engine = create_engine(test_url, echo=False)
    Base.metadata.create_all(engine)
    yield engine
    Base.metadata.drop_all(engine)
    engine.dispose()


@pytest.fixture(scope="function")
def session(test_engine):
    connection = test_engine.connect()
    transaction = connection.begin()
    SessionLocal = sessionmaker(bind=connection)
    session = SessionLocal()
    yield session
    session.close()
    transaction.rollback()
    connection.close()


@pytest.fixture
def sample_team(session: Session):
    from ml_registry_db import models
    team = models.Team(team_name="test-team", team_email="test@company.com")
    session.add(team)
    session.commit()
    session.refresh(team)
    return team


@pytest.fixture
def sample_model(session: Session, sample_team):
    from ml_registry_db import models
    model = models.Model(model_name="test-model", description="Test ML model", team_id=sample_team.team_id)
    session.add(model)
    session.commit()
    session.refresh(model)
    return model


@pytest.fixture
def sample_model_version(session: Session, sample_model):
    from ml_registry_db import models
    version = models.ModelVersion(model_id=sample_model.model_id, semver="1.0.0", artifact_uri="s3://test/model-v1", metadata_={"framework": "pytorch"})
    session.add(version)
    session.commit()
    session.refresh(version)
    return version


@pytest.fixture
def sample_environment(session: Session):
    from ml_registry_db import models
    env = models.Environment(environment_name="production", description="Production environment")
    session.add(env)
    session.commit()
    session.refresh(env)
    return env
```

### 步骤 8.2：创建模型测试

创建 `tests/test_models.py`：

```python
"""Tests for ORM models."""
import pytest
from sqlalchemy.exc import IntegrityError
from ml_registry_db import models


def test_create_team(session):
    team = models.Team(team_name="ml-team", team_email="ml@company.com", team_slack_channel="#ml-platform")
    session.add(team)
    session.commit()
    assert team.team_id is not None
    assert team.team_name == "ml-team"
    assert team.created_at is not None


def test_create_model_with_team(session, sample_team):
    model = models.Model(model_name="fraud-detection", description="Fraud detection model", team_id=sample_team.team_id)
    session.add(model)
    session.commit()
    assert model.model_id is not None
    assert model.team.team_name == sample_team.team_name


def test_model_version_relationship(session, sample_model):
    v1 = models.ModelVersion(model_id=sample_model.model_id, semver="1.0.0", artifact_uri="s3://models/v1")
    v2 = models.ModelVersion(model_id=sample_model.model_id, semver="1.1.0", artifact_uri="s3://models/v1.1")
    session.add_all([v1, v2])
    session.commit()
    session.refresh(sample_model)
    assert len(sample_model.versions) == 2
    assert sample_model.versions[0].model_id == sample_model.model_id


def test_duplicate_model_name_fails(session):
    model1 = models.Model(model_name="duplicate-test")
    model2 = models.Model(model_name="duplicate-test")
    session.add(model1)
    session.commit()
    session.add(model2)
    with pytest.raises(IntegrityError):
        session.commit()


def test_cascade_delete_model_versions(session, sample_model):
    version = models.ModelVersion(model_id=sample_model.model_id, semver="1.0.0", artifact_uri="s3://test")
    session.add(version)
    session.commit()
    version_id = version.version_id
    session.delete(sample_model)
    session.commit()
    deleted_version = session.get(models.ModelVersion, version_id)
    assert deleted_version is None


def test_training_run_creation(session):
    run = models.TrainingRun(model_name="test-model", framework=models.FrameworkEnum.PYTORCH, experiment_name="exp-001", status=models.RunStatusEnum.SUCCEEDED, accuracy=0.95, loss=0.12, hyperparameters={"lr": 0.001, "batch_size": 32})
    session.add(run)
    session.commit()
    assert run.run_id is not None
    assert run.accuracy == 0.95
    assert run.hyperparameters["lr"] == 0.001


def test_deployment_foreign_key_constraint(session, sample_model_version, sample_environment):
    deployment = models.Deployment(version_id=sample_model_version.version_id, environment_id=sample_environment.environment_id, deployed_by="alice@company.com", replicas=3)
    session.add(deployment)
    session.commit()
    assert deployment.deployment_id is not None


def test_many_to_many_model_tags(session, sample_model):
    tag1 = models.Tag(tag_name="production", category="environment")
    tag2 = models.Tag(tag_name="critical", category="priority")
    session.add_all([tag1, tag2])
    session.commit()
    sample_model.tags.append(tag1)
    sample_model.tags.append(tag2)
    session.commit()
    session.refresh(sample_model)
    assert len(sample_model.tags) == 2
```

### 步骤 8.3：创建仓库测试

创建 `tests/test_repositories.py`：

```python
"""Tests for repository layer."""
import pytest
from ml_registry_db import repositories, schemas


def test_register_model(session, sample_team):
    data = schemas.ModelCreate(model_name="new-model", description="Test model", team_id=sample_team.team_id)
    model = repositories.register_model(session, data)
    session.commit()
    assert model.model_id is not None
    assert model.model_name == "new-model"


def test_register_duplicate_model_fails(session):
    data = schemas.ModelCreate(model_name="duplicate-model")
    repositories.register_model(session, data)
    session.commit()
    with pytest.raises(repositories.DuplicateError):
        repositories.register_model(session, data)
        session.commit()


def test_get_model_by_name(session, sample_model):
    model = repositories.get_model_by_name(session, sample_model.model_name)
    assert model is not None
    assert model.model_id == sample_model.model_id


def test_list_models_with_filters(session, sample_team):
    data1 = schemas.ModelCreate(model_name="team1-model", team_id=sample_team.team_id)
    data2 = schemas.ModelCreate(model_name="team2-model")
    repositories.register_model(session, data1)
    repositories.register_model(session, data2)
    session.commit()
    team_models = repositories.list_models(session, team_id=sample_team.team_id)
    assert len(team_models) >= 1
    assert all(m.team_id == sample_team.team_id for m in team_models)


def test_add_model_version(session, sample_model):
    data = schemas.ModelVersionCreate(model_id=sample_model.model_id, semver="2.0.0", artifact_uri="s3://models/v2")
    version = repositories.add_model_version(session, data)
    session.commit()
    assert version.version_id is not None
    assert version.semver == "2.0.0"


def test_get_latest_model_version(session, sample_model):
    for i in range(3):
        data = schemas.ModelVersionCreate(model_id=sample_model.model_id, semver=f"1.{i}.0", artifact_uri=f"s3://models/v1.{i}")
        repositories.add_model_version(session, data)
    session.commit()
    latest = repositories.get_latest_model_version(session, sample_model.model_name)
    assert latest is not None
    assert latest.semver == "1.2.0"


def test_record_training_run(session):
    data = schemas.TrainingRunCreate(model_name="test-model", framework="pytorch", experiment_name="exp-001", hyperparameters={"lr": 0.001})
    run = repositories.record_training_run(session, data)
    session.commit()
    assert run.run_id is not None
    assert run.status == "queued"


def test_update_training_run_status(session):
    data = schemas.TrainingRunCreate(model_name="test-model", framework="pytorch", experiment_name="exp-001")
    run = repositories.record_training_run(session, data)
    session.commit()
    update_data = schemas.TrainingRunUpdate(status="succeeded", accuracy=0.92, loss=0.15)
    updated_run = repositories.update_training_run(session, run.run_id, update_data)
    session.commit()
    assert updated_run.status == "succeeded"
    assert updated_run.accuracy == 0.92


def test_get_best_training_run(session):
    for i, acc in enumerate([0.85, 0.92, 0.88]):
        data = schemas.TrainingRunCreate(model_name="test-model", framework="pytorch", experiment_name="exp-001", run_name=f"run-{i}")
        run = repositories.record_training_run(session, data)
        update = schemas.TrainingRunUpdate(status="succeeded", accuracy=acc)
        repositories.update_training_run(session, run.run_id, update)
    session.commit()
    best = repositories.get_best_training_run(session, "test-model", metric="accuracy")
    assert best is not None
    assert best.accuracy == 0.92


def test_create_deployment(session, sample_model_version, sample_environment):
    data = schemas.DeploymentCreate(version_id=sample_model_version.version_id, environment_id=sample_environment.environment_id, deployed_by="alice@company.com", replicas=2)
    deployment = repositories.create_deployment(session, data)
    session.commit()
    assert deployment.deployment_id is not None
    assert deployment.replicas == 2


def test_get_latest_deployment(session, sample_model, sample_model_version, sample_environment):
    data = schemas.DeploymentCreate(version_id=sample_model_version.version_id, environment_id=sample_environment.environment_id, deployed_by="alice@company.com")
    deployment = repositories.create_deployment(session, data)
    update = schemas.DeploymentUpdate(status="active")
    repositories.update_deployment(session, deployment.deployment_id, update)
    session.commit()
    latest = repositories.get_latest_deployment(session, sample_model.model_name, sample_environment.environment_name)
    assert latest is not None
    assert latest.deployment_id == deployment.deployment_id


def test_add_tag_to_model(session, sample_model):
    repositories.add_tag_to_model(session, sample_model.model_id, "production")
    repositories.add_tag_to_model(session, sample_model.model_id, "critical")
    session.commit()
    session.refresh(sample_model)
    tag_names = [tag.tag_name for tag in sample_model.tags]
    assert "production" in tag_names
    assert "critical" in tag_names
```

### 步骤 8.4：运行测试

```bash
pytest
pytest --cov=ml_registry_db --cov-report=html
pytest tests/test_repositories.py
pytest tests/test_models.py::test_create_team
pytest -v
```

---

## 第 9 部分：CLI 工具

### 步骤 9.1：创建 CLI 模块

创建 `src/ml_registry_db/cli.py`：

```python
"""Command-line interface for ml-registry-db."""
import logging
from typing import Optional
import typer
from rich import print as rprint
from rich.console import Console
from rich.table import Table
from . import repositories, schemas
from .db import get_session, init_db, drop_db, get_pool_status

app = typer.Typer(help="ML Registry Database CLI")
console = Console()


@app.command()
def init():
    """Initialize database schema (create all tables)."""
    try:
        init_db()
        rprint("[green]✅ Database schema initialized successfully[/green]")
    except Exception as e:
        rprint(f"[red]❌ Error initializing database: {e}[/red]")
        raise typer.Exit(code=1)


@app.command()
def drop():
    """Drop all database tables (WARNING: destructive!)."""
    confirm = typer.confirm("Are you sure you want to drop all tables?")
    if not confirm:
        rprint("[yellow]Operation cancelled[/yellow]")
        return
    try:
        drop_db()
        rprint("[yellow]⚠️  All tables dropped[/yellow]")
    except Exception as e:
        rprint(f"[red]❌ Error dropping tables: {e}[/red]")
        raise typer.Exit(code=1)


@app.command()
def pool_status():
    """Show connection pool statistics."""
    try:
        stats = get_pool_status()
        table = Table(title="Connection Pool Status")
        table.add_column("Metric", style="cyan")
        table.add_column("Value", style="green")
        for key, value in stats.items():
            table.add_row(key.replace("_", " ").title(), str(value))
        console.print(table)
    except Exception as e:
        rprint(f"[red]❌ Error getting pool status: {e}[/red]")


@app.command()
def register_model(name: str = typer.Argument(..., help="Model name"), description: str = typer.Option(None, help="Model description"), team_id: Optional[int] = typer.Option(None, help="Team ID")):
    try:
        with get_session() as session:
            data = schemas.ModelCreate(model_name=name, description=description, team_id=team_id)
            model = repositories.register_model(session, data)
            rprint(f"[green]✅ Model registered: {model.model_name}[/green]")
            rprint(f"   Model ID: {model.model_id}")
    except repositories.DuplicateError:
        rprint(f"[red]❌ Model '{name}' already exists[/red]")
        raise typer.Exit(code=1)
    except Exception as e:
        rprint(f"[red]❌ Error: {e}[/red]")
        raise typer.Exit(code=1)


@app.command()
def list_models(team_id: Optional[int] = typer.Option(None, help="Filter by team ID"), limit: int = typer.Option(20, help="Maximum results")):
    try:
        with get_session() as session:
            models = repositories.list_models(session, team_id=team_id, limit=limit)
            if not models:
                rprint("[yellow]No models found[/yellow]")
                return
            table = Table(title=f"Models ({len(models)} found)")
            table.add_column("Model Name", style="cyan")
            table.add_column("Team", style="green")
            table.add_column("Versions", style="yellow")
            table.add_column("Active", style="magenta")
            for model in models:
                table.add_row(model.model_name, model.team.team_name if model.team else "N/A", str(len(model.versions)), "✓" if model.is_active else "✗")
            console.print(table)
    except Exception as e:
        rprint(f"[red]❌ Error: {e}[/red]")
        raise typer.Exit(code=1)


@app.command()
def seed_data():
    """Populate database with sample data for testing."""
    try:
        with get_session() as session:
            team1 = repositories.create_team(session, schemas.TeamCreate(team_name="ml-platform", team_email="ml-platform@company.com"))
            team2 = repositories.create_team(session, schemas.TeamCreate(team_name="data-science", team_email="ds@company.com"))
            fraud_model = repositories.register_model(session, schemas.ModelCreate(model_name="fraud-detection", description="XGBoost fraud detection model", team_id=team1.team_id))
            churn_model = repositories.register_model(session, schemas.ModelCreate(model_name="churn-prediction", description="Customer churn prediction", team_id=team2.team_id))
            repositories.add_model_version(session, schemas.ModelVersionCreate(model_id=fraud_model.model_id, semver="1.0.0", artifact_uri="s3://models/fraud/v1.0.0"))
            repositories.add_model_version(session, schemas.ModelVersionCreate(model_id=churn_model.model_id, semver="1.0.0", artifact_uri="s3://models/churn/v1.0.0"))
            repositories.record_training_run(session, schemas.TrainingRunCreate(model_name="fraud-detection", framework="xgboost", experiment_name="fraud-exp-001", hyperparameters={"max_depth": 6, "learning_rate": 0.1}))
            rprint("[green]✅ Sample data created successfully[/green]")
            rprint("   Teams: 2\n   Models: 2\n   Versions: 2\n   Training runs: 1")
    except Exception as e:
        rprint(f"[red]❌ Error seeding data: {e}[/red]")
        raise typer.Exit(code=1)


if __name__ == "__main__":
    app()
```

### 步骤 9.2：使用 CLI

```bash
ml-registry init
ml-registry seed-data
ml-registry list-models
ml-registry register-model "recommendation-engine" --description "Collaborative filtering recommender" --team-id 1
ml-registry pool-status
```

---

## 第 10 部分：FastAPI 集成示例

创建 `scripts/fastapi_example.py`：

```python
"""
Example FastAPI application using ml-registry-db.
Demonstrates RESTful API endpoints backed by the database.
"""
from typing import List
from uuid import UUID
from fastapi import FastAPI, Depends, HTTPException, status
from sqlalchemy.orm import Session
from ml_registry_db import repositories, schemas
from ml_registry_db.db import get_db

app = FastAPI(title="ML Model Registry API", description="RESTful API for ML model management", version="1.0.0")


@app.post("/models", response_model=schemas.ModelRead, status_code=status.HTTP_201_CREATED)
def create_model(model_data: schemas.ModelCreate, db: Session = Depends(get_db)):
    try:
        model = repositories.register_model(db, model_data)
        db.commit()
        return model
    except repositories.DuplicateError as e:
        raise HTTPException(status_code=status.HTTP_409_CONFLICT, detail=str(e))
    except repositories.ValidationError as e:
        raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail=str(e))


@app.get("/models", response_model=List[schemas.ModelRead])
def list_models(team_id: int = None, is_active: bool = None, limit: int = 50, offset: int = 0, db: Session = Depends(get_db)):
    return repositories.list_models(db, team_id=team_id, is_active=is_active, limit=limit, offset=offset)


@app.get("/models/{model_id}", response_model=schemas.ModelWithVersions)
def get_model(model_id: UUID, db: Session = Depends(get_db)):
    model = repositories.get_model(db, model_id)
    if not model:
        raise HTTPException(status_code=404, detail="Model not found")
    return model


@app.patch("/models/{model_id}", response_model=schemas.ModelRead)
def update_model(model_id: UUID, model_data: schemas.ModelUpdate, db: Session = Depends(get_db)):
    try:
        model = repositories.update_model(db, model_id, model_data)
        db.commit()
        return model
    except repositories.NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


@app.post("/versions", response_model=schemas.ModelVersionRead, status_code=201)
def create_model_version(version_data: schemas.ModelVersionCreate, db: Session = Depends(get_db)):
    try:
        version = repositories.add_model_version(db, version_data)
        db.commit()
        return version
    except repositories.NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))
    except repositories.DuplicateError as e:
        raise HTTPException(status_code=409, detail=str(e))


@app.get("/models/{model_name}/latest", response_model=schemas.ModelVersionRead)
def get_latest_version(model_name: str, db: Session = Depends(get_db)):
    version = repositories.get_latest_model_version(db, model_name)
    if not version:
        raise HTTPException(status_code=404, detail="Model not found or has no versions")
    return version


@app.post("/training-runs", response_model=schemas.TrainingRunRead, status_code=201)
def create_training_run(run_data: schemas.TrainingRunCreate, db: Session = Depends(get_db)):
    run = repositories.record_training_run(db, run_data)
    db.commit()
    return run


@app.patch("/training-runs/{run_id}", response_model=schemas.TrainingRunRead)
def update_training_run(run_id: UUID, run_data: schemas.TrainingRunUpdate, db: Session = Depends(get_db)):
    try:
        run = repositories.update_training_run(db, run_id, run_data)
        db.commit()
        return run
    except repositories.NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


@app.get("/training-runs", response_model=List[schemas.TrainingRunRead])
def list_training_runs(model_name: str = None, experiment_name: str = None, status: str = None, limit: int = 50, db: Session = Depends(get_db)):
    return repositories.list_training_runs(db, model_name=model_name, experiment_name=experiment_name, status=status, limit=limit)


@app.post("/deployments", response_model=schemas.DeploymentRead, status_code=201)
def create_deployment(deployment_data: schemas.DeploymentCreate, db: Session = Depends(get_db)):
    try:
        deployment = repositories.create_deployment(db, deployment_data)
        db.commit()
        return deployment
    except repositories.NotFoundError as e:
        raise HTTPException(status_code=404, detail=str(e))


@app.get("/deployments/latest", response_model=schemas.DeploymentRead)
def get_latest_deployment(model_name: str, environment: str = "production", db: Session = Depends(get_db)):
    deployment = repositories.get_latest_deployment(db, model_name, environment)
    if not deployment:
        raise HTTPException(status_code=404, detail="No active deployment found")
    return deployment


@app.get("/health")
def health_check():
    from ml_registry_db.db import get_pool_status
    pool_stats = get_pool_status()
    return {"status": "healthy", "database": "connected", "pool": pool_stats}


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

运行 API：

```bash
poetry add fastapi uvicorn
python scripts/fastapi_example.py
curl http://localhost:8000/health
curl http://localhost:8000/models
open http://localhost:8000/docs
```

---

## 第 11 部分：生产环境考虑因素

### 连接池最佳实践

**开发：**

```python
engine = create_engine(url, pool_size=5, max_overflow=5, echo=True)
```

**生产：**

```python
engine = create_engine(url, pool_size=20, max_overflow=40, pool_recycle=1800, pool_pre_ping=True, echo=False)
```

### 监控与日志记录

```python
import logging
from sqlalchemy import event

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("ml_registry_db")

@event.listens_for(Engine, "before_cursor_execute")
def log_query(conn, cursor, statement, parameters, context, executemany):
    import time
    conn.info.setdefault("query_start_time", []).append(time.time())

@event.listens_for(Engine, "after_cursor_execute")
def log_query_duration(conn, cursor, statement, parameters, context, executemany):
    total = time.time() - conn.info["query_start_time"].pop()
    if total > 0.5:  # Log slow queries (>500ms)
        logger.warning(f"Slow query ({total:.2f}s): {statement[:200]}")
```

### 依赖注入（FastAPI）

```python
from fastapi import FastAPI, Depends
from sqlalchemy.orm import Session
from ml_registry_db.db import get_db

app = FastAPI()

@app.get("/models")
def get_models(db: Session = Depends(get_db)):
    return db.query(Model).all()
```

### 安全最佳实践

```python
# 使用环境变量（不要硬编码）
DATABASE_URL = os.getenv("DATABASE_URL")

# 使用连接加密
engine = create_engine(url, connect_args={"sslmode": "require"})

# 限制连接用户权限
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO ml_app_user;
```

---

## 总结与交付物

### 学到的内容

✅ **ORM 模型**：将 SQL schema 映射到 Python 类
✅ **仓库模式**：封装 CRUD 和复杂查询
✅ **Alembic 迁移**：版本化 schema 变更
✅ **Pydantic 验证**：请求/响应 schema
✅ **FastAPI 集成**：RESTful API 端点
✅ **测试**：使用 pytest 和 fixture 进行数据库测试
✅ **连接池**：高效连接管理
✅ **生产模式**：日志记录、监控、安全

### 交付物清单

- [ ] `src/ml_registry_db/` - 完整包结构
- [ ] `pyproject.toml` - 依赖配置
- [ ] `alembic/` - 迁移配置
- [ ] `tests/` - 测试套件
- [ ] 种子数据脚本
- [ ] FastAPI 集成示例
- [ ] 文档（README.md）

### 后续步骤

- **练习 05**：数据库优化与索引
- **生产部署**：添加 Kubernetes 部署配置
- **扩展**：添加异步支持（asyncpg、FastAPI async 端点）
- **监控**：集成 OpenTelemetry 追踪

**练习完成！** 🎉 你已经使用 Python 和 SQLAlchemy ORM 构建了一个生产级的 ML 模型注册数据库包。
