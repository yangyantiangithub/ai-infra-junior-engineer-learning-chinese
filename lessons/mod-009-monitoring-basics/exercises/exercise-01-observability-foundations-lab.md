# Exercise 01：Observability Foundations Lab

## 概述

通过为一个生产就绪的 FastAPI 推理服务实现全面的可观测性（metrics、logs 和 traces），开启 Module 009 的学习。你将定义 SLI/SLO、实现结构化日志、暴露 Prometheus metrics、捕获 OpenTelemetry traces，并建立作为整个 monitoring 栈基础的可观测性最佳实践。

**难度：** 初级 → 中级
**预计时间：** 3–4 小时
**前置知识：**
- 已完成 Lecture 01（可观测性基础）
- 熟悉 Python（Module 001）和 FastAPI 基础（Module 007）
- Docker 和 Docker Compose（Module 005）
- 对 Prometheus 和 OpenTelemetry 概念有基本了解

## 学习目标

完成本实验后，你将能够：

1. 为微服务设计和文档化实际的 SLI 和 SLO
2. 使用 Prometheus metrics（counters、histograms、gauges、summaries）对 FastAPI 应用进行 instrument
3. 实现带有关联元数据和 trace 上下文的结构化 JSON 日志
4. 使用自动和手动 instrumentation 集成 OpenTelemetry tracing
5. 通过日志传播 trace 上下文以实现统一可观测性
6. 实现四大黄金信号（latency、traffic、errors、saturation）
7. 为生产服务创建可观测性就绪评估
8. 应用基数最佳实践以避免 metric 爆炸
9. 为监控系统实现健康检查端点
10. 文档化可观测性架构和 SLO 承诺

## 场景

`inference-gateway` 服务是一个关键的微服务，它暴露了一个封装 PyTorch ResNet-50 图像分类模型的 `/predict` 端点。产品领导层在向生产流量广泛推广之前，设定了积极的可靠性目标（99.5% 可用性，P99 延迟 < 300ms）。

作为初级 AI 基础设施工程师，你的任务是为平台团队实现全面的可观测性基础，以便：
- 实时监控 SLO 合规性
- 快速检测和调试生产事件
- 理解模型性能和资源利用率
- 将此服务集成到更广泛的 monitoring 栈（Prometheus、Grafana、Loki）中

本练习构建了可观测性基础，将在后续练习中通过 Prometheus、Grafana、集中化日志和告警进行扩展。

---

## 第 1 部分：环境设置和 Docker 配置

### 步骤 1.1：创建项目结构

为推理网关服务创建一个新目录，包含完整的项目结构：

```bash
mkdir -p inference-gateway
cd inference-gateway

# 创建目录结构
mkdir -p app/{api,core,instrumentation,models}
mkdir -p tests/{unit,integration}
mkdir -p docs
mkdir -p config
mkdir -p scripts

# 创建初始文件
touch app/__init__.py
touch app/api/__init__.py
touch app/core/__init__.py
touch app/instrumentation/__init__.py
touch app/models/__init__.py
touch tests/__init__.py
touch README.md
touch .env.example
touch .dockerignore
touch docker-compose.yml
```

**预期结构：**
```
inference-gateway/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI 应用入口点
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes.py           # API 端点
│   │   └── dependencies.py     # 依赖注入
│   ├── core/
│   │   ├── __init__.py
│   │   ├── config.py           # 配置管理
│   │   └── exceptions.py       # 自定义异常
│   ├── instrumentation/
│   │   ├── __init__.py
│   │   ├── metrics.py          # Prometheus metrics
│   │   ├── logging.py          # 结构化日志
│   │   └── tracing.py          # OpenTelemetry tracing
│   └── models/
│       ├── __init__.py
│       └── inference.py        # 模型推理逻辑
├── tests/
│   ├── __init__.py
│   ├── unit/
│   └── integration/
├── docs/
│   ├── slo.md
│   ├── observability-readiness.md
│   └── architecture.md
├── config/
│   └── prometheus.yml          # 本地 Prometheus 配置
├── scripts/
│   ├── setup.sh
│   └── load_test.sh
├── requirements.txt
├── requirements-dev.txt
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── .dockerignore
├── pytest.ini
└── README.md
```

### 步骤 1.2：定义依赖

创建包含生产依赖的 `requirements.txt`：

```txt
# requirements.txt
# FastAPI and web server
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
pydantic-settings==2.1.0

# Observability - Metrics
prometheus-client==0.19.0

# Observability - Logging
structlog==23.2.0
python-json-logger==2.0.7

# Observability - Tracing
opentelemetry-api==1.21.0
opentelemetry-sdk==1.21.0
opentelemetry-instrumentation-fastapi==0.42b0
opentelemetry-instrumentation-logging==0.42b0
opentelemetry-exporter-otlp==1.21.0
opentelemetry-exporter-otlp-proto-http==1.21.0

# ML Libraries (for model inference)
torch==2.1.1
torchvision==0.16.1
Pillow==10.1.0
numpy==1.26.2

# Utilities
python-multipart==0.0.6
httpx==0.25.2
tenacity==8.2.3
```

创建用于开发和测试的 `requirements-dev.txt`：

```txt
# requirements-dev.txt
-r requirements.txt

# Testing
pytest==7.4.3
pytest-asyncio==0.21.1
pytest-cov==4.1.0
pytest-mock==3.12.0
httpx==0.25.2

# Code Quality
black==23.12.1
ruff==0.1.8
mypy==1.7.1

# Load Testing
locust==2.19.1
```

### 步骤 1.3：创建 Dockerfile

创建一个生产就绪的多阶段 `Dockerfile`：

```dockerfile
# Dockerfile
FROM python:3.11-slim as base

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Create app user for security
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# ============================================
# Development stage
# ============================================
FROM base as development

# Install development dependencies
COPY requirements-dev.txt .
RUN pip install --no-cache-dir -r requirements-dev.txt

# Copy application code
COPY . .

# Change ownership
RUN chown -R appuser:appuser /app

USER appuser

# Expose ports
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/healthz || exit 1

# Run with auto-reload for development
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# ============================================
# Production stage
# ============================================
FROM base as production

# Install production dependencies only
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ ./app/

# Download and cache model weights (optional - remove if too large)
# RUN python -c "import torchvision.models as models; models.resnet50(pretrained=True)"

# Change ownership
RUN chown -R appuser:appuser /app

USER appuser

# Expose ports
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/healthz || exit 1

# Run with production settings
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### 步骤 1.4：创建 Docker Compose 配置

创建用于本地开发且包含可观测性栈的 `docker-compose.yml`：

```yaml
# docker-compose.yml
version: '3.9'

services:
  # Inference Gateway Service
  inference-gateway:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    container_name: inference-gateway
    ports:
      - "8000:8000"
      - "8001:8001"  # 可选：管理/调试端口
    environment:
      - LOG_LEVEL=INFO
      - ENVIRONMENT=development
      - SERVICE_NAME=inference-gateway
      - SERVICE_VERSION=1.0.0
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318
      - OTEL_SERVICE_NAME=inference-gateway
      - OTEL_TRACES_EXPORTER=otlp
      - OTEL_METRICS_EXPORTER=none
      - OTEL_LOGS_EXPORTER=none
    volumes:
      - ./app:/app/app
      - ./tests:/app/tests
      - ./docs:/app/docs
    depends_on:
      - otel-collector
    networks:
      - observability
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # OpenTelemetry Collector（用于 trace 收集）
  otel-collector:
    image: otel/opentelemetry-collector:0.91.0
    container_name: otel-collector
    command: ["--config=/etc/otel-collector-config.yml"]
    volumes:
      - ./config/otel-collector-config.yml:/etc/otel-collector-config.yml
    ports:
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"   # OTLP HTTP receiver
      - "13133:13133" # health_check extension
    networks:
      - observability

  # Jaeger（用于 trace 可视化 - 本练习可选）
  jaeger:
    image: jaegertracing/all-in-one:1.52
    container_name: jaeger
    ports:
      - "16686:16686"  # Jaeger UI
      - "14268:14268"  # Jaeger collector HTTP
    environment:
      - COLLECTOR_OTLP_ENABLED=true
    networks:
      - observability

  # Prometheus（用于 metrics - 将在 Exercise 02 中扩展）
  prometheus:
    image: prom/prometheus:v2.48.1
    container_name: prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--web.enable-lifecycle'
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - observability

networks:
  observability:
    driver: bridge

volumes:
  prometheus-data:
```

### 步骤 1.5：创建 OpenTelemetry Collector 配置

创建 `config/otel-collector-config.yml`：

```yaml
# config/otel-collector-config.yml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

  # 添加资源属性
  resource:
    attributes:
      - key: environment
        value: development
        action: upsert

exporters:
  # 导出到 Jaeger 进行可视化
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

  # 用于调试的 Console exporter
  logging:
    loglevel: info

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [otlp/jaeger, logging]

  extensions: [health_check]

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
```

### 步骤 1.6：创建 Prometheus 配置

创建 `config/prometheus.yml`：

```yaml
# config/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'dev'
    environment: 'development'

scrape_configs:
  # 抓取推理网关服务
  - job_name: 'inference-gateway'
    static_configs:
      - targets: ['inference-gateway:8000']
        labels:
          service: 'inference-gateway'
          team: 'ml-platform'

  # 抓取 Prometheus 自身
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### 步骤 1.7：创建环境配置

创建 `.env.example`：

```bash
# .env.example
# Application Settings
SERVICE_NAME=inference-gateway
SERVICE_VERSION=1.0.0
ENVIRONMENT=development
LOG_LEVEL=INFO

# Server Configuration
HOST=0.0.0.0
PORT=8000
WORKERS=4

# Model Configuration
MODEL_NAME=resnet50
MODEL_DEVICE=cpu
MODEL_BATCH_SIZE=1

# OpenTelemetry Configuration
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_SERVICE_NAME=inference-gateway
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=none
OTEL_LOGS_EXPORTER=none

# Prometheus Configuration
METRICS_PORT=8000
METRICS_PATH=/metrics

# SLO Targets (for documentation/reference)
SLO_AVAILABILITY_TARGET=99.5
SLO_P99_LATENCY_MS=300
SLO_ERROR_RATE_MAX=0.5
```

复制到 `.env`：
```bash
cp .env.example .env
```

**✅ 检查点：** 验证目录结构和配置文件已正确创建。

---

## 第 2 部分：使用 Pydantic Settings 的配置管理

### 步骤 2.1：创建配置模块

创建 `app/core/config.py`：

```python
# app/core/config.py
"""
Configuration management using Pydantic Settings.
Loads configuration from environment variables with validation.
"""
from typing import Literal
from pydantic import Field
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    """Application settings loaded from environment variables."""

    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=False,
        extra="ignore",
    )

    # Application Settings
    service_name: str = Field(default="inference-gateway", description="Service name")
    service_version: str = Field(default="1.0.0", description="Service version")
    environment: Literal["development", "staging", "production"] = Field(
        default="development", description="Deployment environment"
    )
    log_level: Literal["DEBUG", "INFO", "WARNING", "ERROR", "CRITICAL"] = Field(
        default="INFO", description="Logging level"
    )

    # Server Configuration
    host: str = Field(default="0.0.0.0", description="Server host")
    port: int = Field(default=8000, ge=1024, le=65535, description="Server port")
    workers: int = Field(default=4, ge=1, le=16, description="Number of workers")

    # Model Configuration
    model_name: str = Field(default="resnet50", description="Model architecture")
    model_device: Literal["cpu", "cuda", "mps"] = Field(
        default="cpu", description="Device for model inference"
    )
    model_batch_size: int = Field(
        default=1, ge=1, le=128, description="Inference batch size"
    )

    # OpenTelemetry Configuration
    otel_exporter_otlp_endpoint: str = Field(
        default="http://localhost:4318", description="OTLP endpoint"
    )
    otel_service_name: str = Field(
        default="inference-gateway", description="OTEL service name"
    )
    otel_traces_exporter: str = Field(
        default="otlp", description="Trace exporter type"
    )

    # Metrics Configuration
    metrics_port: int = Field(
        default=8000, ge=1024, le=65535, description="Metrics port"
    )
    metrics_path: str = Field(default="/metrics", description="Metrics endpoint path")

    # SLO Targets (for reference)
    slo_availability_target: float = Field(
        default=99.5, ge=0.0, le=100.0, description="Availability SLO target (%)"
    )
    slo_p99_latency_ms: int = Field(
        default=300, ge=1, description="P99 latency SLO target (ms)"
    )
    slo_error_rate_max: float = Field(
        default=0.5, ge=0.0, le=100.0, description="Max error rate SLO (%)"
    )

    @property
    def is_production(self) -> bool:
        """Check if running in production environment."""
        return self.environment == "production"

    @property
    def is_development(self) -> bool:
        """Check if running in development environment."""
        return self.environment == "development"


# Global settings instance
settings = Settings()
```

**关键特性：**
- 带有 Pydantic 验证的类型安全配置
- 环境变量加载
- 合理的默认值
- 生产/开发环境检查
- 用于文档的 SLO 目标

### 步骤 2.2：创建自定义异常

创建 `app/core/exceptions.py`：

```python
# app/core/exceptions.py
"""Custom exceptions for the inference gateway service."""


class InferenceGatewayException(Exception):
    """Base exception for all service-specific errors."""

    def __init__(self, message: str, error_code: str = "UNKNOWN_ERROR"):
        self.message = message
        self.error_code = error_code
        super().__init__(self.message)


class ModelNotLoadedError(InferenceGatewayException):
    """Raised when model is not loaded or initialization failed."""

    def __init__(self, message: str = "Model not loaded"):
        super().__init__(message, error_code="MODEL_NOT_LOADED")


class InferenceError(InferenceGatewayException):
    """Raised when inference fails."""

    def __init__(self, message: str = "Inference failed"):
        super().__init__(message, error_code="INFERENCE_ERROR")


class InvalidInputError(InferenceGatewayException):
    """Raised when input validation fails."""

    def __init__(self, message: str = "Invalid input"):
        super().__init__(message, error_code="INVALID_INPUT")


class ServiceUnavailableError(InferenceGatewayException):
    """Raised when service is temporarily unavailable."""

    def __init__(self, message: str = "Service temporarily unavailable"):
        super().__init__(message, error_code="SERVICE_UNAVAILABLE")
```

**✅ 检查点：** 配置模块能通过环境变量正确加载。

---

## 第 3 部分：实现带有 Trace 上下文的结构化日志

### 步骤 3.1：创建日志模块

创建 `app/instrumentation/logging.py`：

```python
# app/instrumentation/logging.py
"""
Structured logging configuration with OpenTelemetry trace context.
Outputs JSON logs with correlation IDs for unified observability.
"""
import logging
import sys
from typing import Any

import structlog
from opentelemetry import trace
from pythonjsonlogger import jsonlogger

from app.core.config import settings


def add_trace_context(
    logger: logging.Logger, method_name: str, event_dict: dict[str, Any]
) -> dict[str, Any]:
    """
    Add OpenTelemetry trace context to log entries.

    Adds trace_id and span_id to every log for correlation with traces.
    """
    span = trace.get_current_span()
    if span and span.get_span_context().is_valid:
        span_context = span.get_span_context()
        event_dict["trace_id"] = format(span_context.trace_id, "032x")
        event_dict["span_id"] = format(span_context.span_id, "016x")
        event_dict["trace_flags"] = span_context.trace_flags
    return event_dict


def add_service_context(
    logger: logging.Logger, method_name: str, event_dict: dict[str, Any]
) -> dict[str, Any]:
    """Add service metadata to all log entries."""
    event_dict["service"] = settings.service_name
    event_dict["version"] = settings.service_version
    event_dict["environment"] = settings.environment
    return event_dict


def configure_logging() -> None:
    """
    Configure structured logging with JSON output and trace context.

    Sets up structlog with:
    - JSON formatting for machine readability
    - OpenTelemetry trace context injection
    - Service metadata (name, version, environment)
    - Timestamp and log level
    - Consistent field naming
    """
    # Configure standard library logging
    logging.basicConfig(
        format="%(message)s",
        stream=sys.stdout,
        level=getattr(logging, settings.log_level),
    )

    # Configure structlog
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            add_service_context,
            add_trace_context,
            structlog.processors.TimeStamper(fmt="iso", utc=True),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.stdlib.BoundLogger,
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        cache_logger_on_first_use=True,
    )


def get_logger(name: str) -> structlog.BoundLogger:
    """
    Get a logger instance for the specified module.

    Args:
        name: Logger name (typically __name__)

    Returns:
        Configured structlog logger
    """
    return structlog.get_logger(name)


# Initialize logging on module import
configure_logging()
```

**关键特性：**
- 自动 trace 上下文注入（trace_id、span_id）
- 每条日志的服务元数据
- 用于日志聚合的 JSON 输出
- UTC 的 ISO 时间戳
- 用于过滤的结构化字段

**示例日志输出：**
```json
{
  "event": "request_completed",
  "timestamp": "2025-10-23T10:15:30.123456Z",
  "level": "info",
  "logger": "app.api.routes",
  "service": "inference-gateway",
  "version": "1.0.0",
  "environment": "development",
  "trace_id": "5f3a2b1c0d9e8f7a6b5c4d3e2f1a0b9c",
  "span_id": "1a2b3c4d5e6f7a8b",
  "method": "POST",
  "path": "/predict",
  "status_code": 200,
  "latency_ms": 145.67,
  "model_version": "resnet50-v1"
}
```

**✅ 检查点：** 导入日志模块并验证 JSON 输出：

```python
from app.instrumentation.logging import get_logger

logger = get_logger(__name__)
logger.info("test_log", foo="bar", count=42)
```

---

## 第 4 部分：实现 Prometheus Metrics

### 步骤 4.1：创建 Metrics 模块

创建 `app/instrumentation/metrics.py`：

```python
# app/instrumentation/metrics.py
"""
Prometheus metrics instrumentation for the inference gateway.

Implements the Four Golden Signals:
- Latency: Request duration histograms
- Traffic: Request rate counters
- Errors: Error rate counters
- Saturation: Queue depth and resource usage gauges
"""
from typing import Callable
import time
from functools import wraps

from prometheus_client import (
    Counter,
    Histogram,
    Gauge,
    Summary,
    Info,
    CollectorRegistry,
    REGISTRY,
)

from app.core.config import settings


# ============================================
# Metric Definitions
# ============================================

# Traffic: Request counters
http_requests_total = Counter(
    name="http_requests_total",
    documentation="Total HTTP requests by method, endpoint, and status",
    labelnames=["method", "endpoint", "status_code"],
    registry=REGISTRY,
)

inference_requests_total = Counter(
    name="inference_requests_total",
    documentation="Total inference requests by model, version, and status",
    labelnames=["model_name", "model_version", "status"],
    registry=REGISTRY,
)

# Latency: Request duration histograms
http_request_duration_seconds = Histogram(
    name="http_request_duration_seconds",
    documentation="HTTP request duration in seconds",
    labelnames=["method", "endpoint"],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
    registry=REGISTRY,
)

inference_duration_seconds = Histogram(
    name="inference_duration_seconds",
    documentation="Model inference duration in seconds",
    labelnames=["model_name", "model_version"],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.15, 0.2, 0.3, 0.5, 0.75, 1.0, 2.0],
    registry=REGISTRY,
)

inference_preprocessing_duration_seconds = Histogram(
    name="inference_preprocessing_duration_seconds",
    documentation="Input preprocessing duration in seconds",
    labelnames=["model_name"],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5],
    registry=REGISTRY,
)

inference_postprocessing_duration_seconds = Histogram(
    name="inference_postprocessing_duration_seconds",
    documentation="Output postprocessing duration in seconds",
    labelnames=["model_name"],
    buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5],
    registry=REGISTRY,
)

# Errors: Error counters
http_errors_total = Counter(
    name="http_errors_total",
    documentation="Total HTTP errors by method, endpoint, and error type",
    labelnames=["method", "endpoint", "error_type"],
    registry=REGISTRY,
)

inference_errors_total = Counter(
    name="inference_errors_total",
    documentation="Total inference errors by model, version, and error type",
    labelnames=["model_name", "model_version", "error_type"],
    registry=REGISTRY,
)

# Saturation: Queue depth and resource usage
inference_queue_depth = Gauge(
    name="inference_queue_depth",
    documentation="Current number of requests in inference queue",
    registry=REGISTRY,
)

model_memory_bytes = Gauge(
    name="model_memory_bytes",
    documentation="Model memory usage in bytes",
    labelnames=["model_name"],
    registry=REGISTRY,
)

# ML-Specific Metrics
inference_batch_size = Histogram(
    name="inference_batch_size",
    documentation="Inference batch size distribution",
    labelnames=["model_name"],
    buckets=[1, 2, 4, 8, 16, 32, 64, 128],
    registry=REGISTRY,
)

inference_confidence_score = Histogram(
    name="inference_confidence_score",
    documentation="Model confidence score distribution",
    labelnames=["model_name", "predicted_class"],
    buckets=[0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 0.95, 0.99, 1.0],
    registry=REGISTRY,
)

model_predictions_total = Counter(
    name="model_predictions_total",
    documentation="Total model predictions by class",
    labelnames=["model_name", "predicted_class"],
    registry=REGISTRY,
)

# Application Info
app_info = Info(
    name="app",
    documentation="Application information",
    registry=REGISTRY,
)

# Initialize app info
app_info.info(
    {
        "service": settings.service_name,
        "version": settings.service_version,
        "environment": settings.environment,
    }
)


# ============================================
# Metric Utilities
# ============================================

class MetricsMiddleware:
    """
    Middleware to automatically track HTTP request metrics.

    Records:
    - Request count by method, endpoint, status
    - Request duration by method, endpoint
    - Error count by method, endpoint, error type
    """

    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        method = scope["method"]
        path = scope["path"]

        # Skip metrics endpoint to avoid recursion
        if path == "/metrics":
            await self.app(scope, receive, send)
            return

        start_time = time.perf_counter()
        status_code = 500  # Default to error

        async def send_wrapper(message):
            nonlocal status_code
            if message["type"] == "http.response.start":
                status_code = message["status"]
            await send(message)

        try:
            await self.app(scope, receive, send_wrapper)
        except Exception as e:
            http_errors_total.labels(
                method=method, endpoint=path, error_type=type(e).__name__
            ).inc()
            raise
        finally:
            duration = time.perf_counter() - start_time

            # Record metrics
            http_requests_total.labels(
                method=method, endpoint=path, status_code=status_code
            ).inc()

            http_request_duration_seconds.labels(method=method, endpoint=path).observe(
                duration
            )


def track_inference_time(model_name: str, model_version: str) -> Callable:
    """
    Decorator to track inference duration and count.

    Usage:
        @track_inference_time("resnet50", "v1")
        async def run_inference(input_data):
            ...
    """

    def decorator(func: Callable) -> Callable:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            start_time = time.perf_counter()
            status = "success"

            try:
                result = await func(*args, **kwargs)
                return result
            except Exception as e:
                status = "failure"
                inference_errors_total.labels(
                    model_name=model_name,
                    model_version=model_version,
                    error_type=type(e).__name__,
                ).inc()
                raise
            finally:
                duration = time.perf_counter() - start_time
                inference_duration_seconds.labels(
                    model_name=model_name, model_version=model_version
                ).observe(duration)

                inference_requests_total.labels(
                    model_name=model_name, model_version=model_version, status=status
                ).inc()

        return wrapper

    return decorator


# ============================================
# Metric Helpers
# ============================================

def record_preprocessing_time(model_name: str, duration: float) -> None:
    """Record preprocessing duration."""
    inference_preprocessing_duration_seconds.labels(model_name=model_name).observe(
        duration
    )


def record_postprocessing_time(model_name: str, duration: float) -> None:
    """Record postprocessing duration."""
    inference_postprocessing_duration_seconds.labels(model_name=model_name).observe(
        duration
    )


def record_prediction(
    model_name: str, predicted_class: str, confidence: float
) -> None:
    """Record model prediction and confidence."""
    model_predictions_total.labels(
        model_name=model_name, predicted_class=predicted_class
    ).inc()

    inference_confidence_score.labels(
        model_name=model_name, predicted_class=predicted_class
    ).observe(confidence)


def set_queue_depth(depth: int) -> None:
    """Set current inference queue depth."""
    inference_queue_depth.set(depth)


def set_model_memory(model_name: str, memory_bytes: int) -> None:
    """Set model memory usage in bytes."""
    model_memory_bytes.labels(model_name=model_name).set(memory_bytes)
```

**关键特性：**
- **四大黄金信号**：延迟、流量、错误、饱和度
- **HTTP metrics**：请求计数、时长、错误
- **推理 metrics**：时长、预处理、后处理
- **ML 特定 metrics**：置信度分数、按类别的预测、批次大小
- **自动跟踪**：装饰器和中间件模式
- **基数控制**：限制 label 值以防止 metric 爆炸

**Histogram Bucket 设计：**
- HTTP 请求：5ms 到 10s（Web 服务延迟范围）
- 推理：10ms 到 2s（模型执行范围）
- 预处理/后处理：1ms 到 500ms（数据转换范围）

**✅ 检查点：** Metrics 已定义并可无错误导入。

---

## 第 5 部分：实现 OpenTelemetry Tracing

### 步骤 5.1：创建 Tracing 模块

创建 `app/instrumentation/tracing.py`：

```python
# app/instrumentation/tracing.py
"""
OpenTelemetry tracing configuration for distributed tracing.

Provides:
- Automatic instrumentation for FastAPI
- Manual span creation for custom operations
- Trace context propagation
- OTLP export to collector
"""
import logging
from typing import Optional

from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor, ConsoleSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.logging import LoggingInstrumentor

from app.core.config import settings

logger = logging.getLogger(__name__)


def configure_tracing() -> None:
    """
    Configure OpenTelemetry tracing with OTLP export.

    Sets up:
    - Resource attributes (service name, version)
    - Trace provider
    - OTLP exporter (or console exporter for development)
    - Batch span processor
    """
    # Create resource with service information
    resource = Resource.create(
        {
            SERVICE_NAME: settings.otel_service_name,
            SERVICE_VERSION: settings.service_version,
            "deployment.environment": settings.environment,
        }
    )

    # Create tracer provider
    provider = TracerProvider(resource=resource)

    # Configure span processor and exporter
    if settings.environment == "development":
        # Use console exporter for local development
        console_exporter = ConsoleSpanExporter()
        provider.add_span_processor(BatchSpanProcessor(console_exporter))
        logger.info("Tracing configured with ConsoleSpanExporter")

    try:
        # Try to use OTLP exporter
        otlp_exporter = OTLPSpanExporter(
            endpoint=f"{settings.otel_exporter_otlp_endpoint}/v1/traces"
        )
        provider.add_span_processor(BatchSpanProcessor(otlp_exporter))
        logger.info(
            f"Tracing configured with OTLPSpanExporter: {settings.otel_exporter_otlp_endpoint}"
        )
    except Exception as e:
        logger.warning(f"Failed to configure OTLP exporter: {e}")
        # Fallback to console exporter
        console_exporter = ConsoleSpanExporter()
        provider.add_span_processor(BatchSpanProcessor(console_exporter))

    # Set global tracer provider
    trace.set_tracer_provider(provider)

    # Instrument logging to add trace context
    LoggingInstrumentor().instrument(set_logging_format=False)

    logger.info("OpenTelemetry tracing configured successfully")


def instrument_fastapi(app) -> None:
    """
    Instrument FastAPI application with automatic tracing.

    Args:
        app: FastAPI application instance
    """
    FastAPIInstrumentor.instrument_app(app)
    logger.info("FastAPI instrumented with OpenTelemetry")


def get_tracer(name: str) -> trace.Tracer:
    """
    Get a tracer instance for manual span creation.

    Args:
        name: Tracer name (typically module name)

    Returns:
        OpenTelemetry tracer
    """
    return trace.get_tracer(name)


def get_current_trace_id() -> Optional[str]:
    """
    Get current trace ID as hex string.

    Returns:
        Trace ID or None if no active span
    """
    span = trace.get_current_span()
    if span and span.get_span_context().is_valid:
        return format(span.get_span_context().trace_id, "032x")
    return None


def get_current_span_id() -> Optional[str]:
    """
    Get current span ID as hex string.

    Returns:
        Span ID or None if no active span
    """
    span = trace.get_current_span()
    if span and span.get_span_context().is_valid:
        return format(span.get_span_context().span_id, "016x")
    return None
```

**关键特性：**
- 自动 FastAPI instrumentation
- 导出到 collector 的 OTLP
- 用于开发的 Console export
- Trace 上下文辅助函数
- 资源属性（service、version、environment）

**✅ 检查点：** Tracing 模块可无错误导入。

---

## 第 6 部分：实现模型推理

### 步骤 6.1：创建推理模块

创建 `app/models/inference.py`：

```python
# app/models/inference.py
"""
Model inference logic with observability instrumentation.

Implements:
- Model loading and initialization
- Input preprocessing
- Inference execution
- Output postprocessing
- Comprehensive metrics tracking
"""
import time
from typing import List, Dict, Any
import io

import torch
import torchvision.transforms as transforms
import torchvision.models as models
from PIL import Image
import numpy as np

from opentelemetry import trace

from app.core.config import settings
from app.core.exceptions import ModelNotLoadedError, InferenceError, InvalidInputError
from app.instrumentation.logging import get_logger
from app.instrumentation import metrics

logger = get_logger(__name__)
tracer = trace.get_tracer(__name__)

# ImageNet class labels (abbreviated for this exercise)
# In production, load from a complete file
IMAGENET_CLASSES = [
    "tench", "goldfish", "great_white_shark", "tiger_shark", "hammerhead",
    # ... (1000 classes total)
]


class ModelInferenceService:
    """
    Singleton service for model inference with observability.

    Handles model lifecycle, preprocessing, inference, and postprocessing
    with comprehensive instrumentation.
    """

    def __init__(self):
        self.model: Optional[torch.nn.Module] = None
        self.device: torch.device = torch.device(settings.model_device)
        self.model_name = settings.model_name
        self.model_version = "v1.0"
        self.transform = self._create_transform()
        self._is_loaded = False

    def _create_transform(self) -> transforms.Compose:
        """Create image preprocessing transform pipeline."""
        return transforms.Compose([
            transforms.Resize(256),
            transforms.CenterCrop(224),
            transforms.ToTensor(),
            transforms.Normalize(
                mean=[0.485, 0.456, 0.406], std=[0.229, 0.224, 0.225]
            ),
        ])

    @tracer.start_as_current_span("load_model")
    def load_model(self) -> None:
        """
        Load model weights and move to device.

        Raises:
            ModelNotLoadedError: If model loading fails
        """
        if self._is_loaded:
            logger.info("Model already loaded", model_name=self.model_name)
            return

        try:
            logger.info(
                "Loading model", model_name=self.model_name, device=str(self.device)
            )

            # Load pretrained ResNet-50
            if self.model_name == "resnet50":
                self.model = models.resnet50(pretrained=True)
            else:
                raise ModelNotLoadedError(f"Unknown model: {self.model_name}")

            # Move to device and set to eval mode
            self.model.to(self.device)
            self.model.eval()

            self._is_loaded = True

            # Record model memory usage (approximate)
            param_size = sum(p.numel() * p.element_size() for p in self.model.parameters())
            buffer_size = sum(b.numel() * b.element_size() for b in self.model.buffers())
            total_memory = param_size + buffer_size

            metrics.set_model_memory(self.model_name, total_memory)

            logger.info(
                "Model loaded successfully",
                model_name=self.model_name,
                device=str(self.device),
                memory_bytes=total_memory,
            )

        except Exception as e:
            logger.error(
                "Model loading failed", model_name=self.model_name, error=str(e)
            )
            raise ModelNotLoadedError(f"Failed to load model: {e}") from e

    @tracer.start_as_current_span("preprocess_image")
    def preprocess_image(self, image_bytes: bytes) -> torch.Tensor:
        """
        Preprocess image bytes to model input tensor.

        Args:
            image_bytes: Raw image bytes

        Returns:
            Preprocessed tensor ready for inference

        Raises:
            InvalidInputError: If image is invalid
        """
        start_time = time.perf_counter()

        try:
            # Load image from bytes
            image = Image.open(io.BytesIO(image_bytes)).convert("RGB")

            # Apply transformations
            tensor = self.transform(image)

            # Add batch dimension
            tensor = tensor.unsqueeze(0)

            # Move to device
            tensor = tensor.to(self.device)

            duration = time.perf_counter() - start_time
            metrics.record_preprocessing_time(self.model_name, duration)

            logger.debug(
                "Image preprocessed",
                model_name=self.model_name,
                duration_ms=duration * 1000,
                tensor_shape=list(tensor.shape),
            )

            return tensor

        except Exception as e:
            logger.error("Preprocessing failed", error=str(e))
            raise InvalidInputError(f"Failed to preprocess image: {e}") from e

    @tracer.start_as_current_span("run_inference")
    @metrics.track_inference_time("resnet50", "v1.0")
    async def predict(self, image_bytes: bytes) -> Dict[str, Any]:
        """
        Run inference on image and return top predictions.

        Args:
            image_bytes: Raw image bytes

        Returns:
            Dictionary with predictions and metadata

        Raises:
            ModelNotLoadedError: If model not loaded
            InferenceError: If inference fails
        """
        if not self._is_loaded:
            raise ModelNotLoadedError("Model not loaded. Call load_model() first.")

        with tracer.start_as_current_span("inference_pipeline") as span:
            try:
                # Preprocess
                input_tensor = self.preprocess_image(image_bytes)

                # Inference
                start_time = time.perf_counter()

                with torch.no_grad():
                    outputs = self.model(input_tensor)

                inference_duration = time.perf_counter() - start_time

                span.set_attribute("inference.duration_ms", inference_duration * 1000)
                span.set_attribute("inference.model_name", self.model_name)

                # Postprocessing
                postprocess_start = time.perf_counter()

                probabilities = torch.nn.functional.softmax(outputs[0], dim=0)
                top5_prob, top5_idx = torch.topk(probabilities, 5)

                predictions = []
                for i in range(5):
                    class_idx = top5_idx[i].item()
                    confidence = top5_prob[i].item()

                    # Use placeholder class name (in production, load full ImageNet labels)
                    class_name = IMAGENET_CLASSES[class_idx] if class_idx < len(IMAGENET_CLASSES) else f"class_{class_idx}"

                    predictions.append({
                        "class": class_name,
                        "class_id": class_idx,
                        "confidence": round(confidence, 4),
                    })

                    # Record prediction metrics
                    if i == 0:  # Only record top prediction to control cardinality
                        metrics.record_prediction(
                            self.model_name, class_name, confidence
                        )

                postprocess_duration = time.perf_counter() - postprocess_start
                metrics.record_postprocessing_time(self.model_name, postprocess_duration)

                # Record batch size
                metrics.inference_batch_size.labels(model_name=self.model_name).observe(
                    input_tensor.shape[0]
                )

                logger.info(
                    "Inference completed",
                    model_name=self.model_name,
                    top_prediction=predictions[0]["class"],
                    top_confidence=predictions[0]["confidence"],
                    inference_duration_ms=inference_duration * 1000,
                )

                return {
                    "predictions": predictions,
                    "model_name": self.model_name,
                    "model_version": self.model_version,
                    "inference_duration_ms": round(inference_duration * 1000, 2),
                }

            except (ModelNotLoadedError, InvalidInputError):
                raise
            except Exception as e:
                logger.error("Inference failed", model_name=self.model_name, error=str(e))
                raise InferenceError(f"Inference failed: {e}") from e


# Global singleton instance
model_service = ModelInferenceService()
```

**关键特性：**
- 模型生命周期管理
- 全面的错误处理
- 带有 spans 的分布式 tracing
- 预处理、推理、后处理的 metrics
- ML 特定 metrics（置信度、按类别的预测）
- 内存跟踪

**✅ 检查点：** 推理模块可正确导入。

---

## 第 7 部分：创建 FastAPI 应用和路由

### 步骤 7.1：创建 API 依赖

创建 `app/api/dependencies.py`：

```python
# app/api/dependencies.py
"""FastAPI dependencies for dependency injection."""
from typing import Generator

from app.models.inference import model_service, ModelInferenceService


def get_model_service() -> Generator[ModelInferenceService, None, None]:
    """Dependency to get model service instance."""
    yield model_service
```

### 步骤 7.2：创建 API 路由

创建 `app/api/routes.py`：

```python
# app/api/routes.py
"""API route handlers for the inference gateway."""
from typing import Dict, Any

from fastapi import APIRouter, Depends, HTTPException, File, UploadFile, status
from opentelemetry import trace

from app.models.inference import ModelInferenceService
from app.api.dependencies import get_model_service
from app.core.exceptions import (
    ModelNotLoadedError,
    InferenceError,
    InvalidInputError,
)
from app.instrumentation.logging import get_logger
from app.instrumentation import metrics

logger = get_logger(__name__)
tracer = trace.get_tracer(__name__)

router = APIRouter()


@router.get("/healthz", status_code=status.HTTP_200_OK, tags=["health"])
async def health_check() -> Dict[str, str]:
    """
    Health check endpoint for liveness probe.

    Returns:
        Status message
    """
    return {"status": "healthy"}


@router.get("/readyz", status_code=status.HTTP_200_OK, tags=["health"])
async def readiness_check(
    model_service: ModelInferenceService = Depends(get_model_service),
) -> Dict[str, Any]:
    """
    Readiness check endpoint for readiness probe.

    Verifies model is loaded and ready to serve requests.

    Returns:
        Status message with model info

    Raises:
        HTTPException: If model not ready
    """
    if not model_service._is_loaded:
        logger.warning("Readiness check failed: model not loaded")
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Model not loaded",
        )

    return {
        "status": "ready",
        "model_name": model_service.model_name,
        "model_version": model_service.model_version,
        "device": str(model_service.device),
    }


@router.post("/predict", status_code=status.HTTP_200_OK, tags=["inference"])
async def predict(
    file: UploadFile = File(..., description="Image file to classify"),
    model_service: ModelInferenceService = Depends(get_model_service),
) -> Dict[str, Any]:
    """
    Run inference on uploaded image.

    Args:
        file: Uploaded image file
        model_service: Injected model service

    Returns:
        Prediction results with top-5 classes and confidences

    Raises:
        HTTPException: If inference fails
    """
    with tracer.start_as_current_span("predict_endpoint") as span:
        try:
            # Read image bytes
            image_bytes = await file.read()

            span.set_attribute("file.name", file.filename or "unknown")
            span.set_attribute("file.size_bytes", len(image_bytes))

            # Run inference
            result = await model_service.predict(image_bytes)

            logger.info(
                "Prediction successful",
                filename=file.filename,
                top_prediction=result["predictions"][0]["class"],
                confidence=result["predictions"][0]["confidence"],
            )

            return result

        except ModelNotLoadedError as e:
            logger.error("Model not loaded", error=str(e))
            metrics.http_errors_total.labels(
                method="POST", endpoint="/predict", error_type="ModelNotLoadedError"
            ).inc()
            raise HTTPException(
                status_code=status.HTTP_503_SERVICE_UNAVAILABLE, detail=str(e)
            )

        except InvalidInputError as e:
            logger.error("Invalid input", error=str(e), filename=file.filename)
            metrics.http_errors_total.labels(
                method="POST", endpoint="/predict", error_type="InvalidInputError"
            ).inc()
            raise HTTPException(status_code=status.HTTP_400_BAD_REQUEST, detail=str(e))

        except InferenceError as e:
            logger.error("Inference failed", error=str(e))
            metrics.http_errors_total.labels(
                method="POST", endpoint="/predict", error_type="InferenceError"
            ).inc()
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR, detail=str(e)
            )

        except Exception as e:
            logger.error("Unexpected error", error=str(e), error_type=type(e).__name__)
            metrics.http_errors_total.labels(
                method="POST", endpoint="/predict", error_type="UnexpectedError"
            ).inc()
            raise HTTPException(
                status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
                detail="Internal server error",
            )
```

**关键特性：**
- 健康检查和就绪检查
- 文件上传处理
- 全面的错误处理
- Metrics 跟踪
- 分布式 tracing
- 结构化日志

### 步骤 7.3：创建主应用

创建 `app/main.py`：

```python
# app/main.py
"""
FastAPI application entry point with observability instrumentation.

Configures:
- FastAPI application
- Prometheus metrics endpoint
- OpenTelemetry tracing
- Structured logging
- Error handlers
- CORS middleware
"""
from contextlib import asynccontextmanager

from fastapi import FastAPI, Request, status
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware
from prometheus_client import make_asgi_app

from app.core.config import settings
from app.core.exceptions import InferenceGatewayException
from app.api.routes import router
from app.models.inference import model_service
from app.instrumentation.logging import get_logger
from app.instrumentation.tracing import configure_tracing, instrument_fastapi
from app.instrumentation.metrics import MetricsMiddleware

logger = get_logger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    Application lifespan handler.

    Startup:
    - Configure tracing
    - Load ML model

    Shutdown:
    - Cleanup resources
    """
    # Startup
    logger.info(
        "Starting application",
        service=settings.service_name,
        version=settings.service_version,
        environment=settings.environment,
    )

    # Configure OpenTelemetry tracing
    configure_tracing()

    # Load model
    try:
        model_service.load_model()
        logger.info("Model loaded successfully")
    except Exception as e:
        logger.error("Failed to load model during startup", error=str(e))
        # Continue startup even if model loading fails
        # Readiness check will catch this

    yield

    # Shutdown
    logger.info("Shutting down application")


# Create FastAPI application
app = FastAPI(
    title="Inference Gateway",
    description="ML model inference service with comprehensive observability",
    version=settings.service_version,
    lifespan=lifespan,
    docs_url="/docs" if settings.is_development else None,
    redoc_url="/redoc" if settings.is_development else None,
)

# Add CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"] if settings.is_development else [],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Add metrics middleware
app.add_middleware(MetricsMiddleware)

# Instrument with OpenTelemetry
instrument_fastapi(app)

# Include API routes
app.include_router(router)

# Mount Prometheus metrics endpoint
metrics_app = make_asgi_app()
app.mount("/metrics", metrics_app)


# Exception handlers
@app.exception_handler(InferenceGatewayException)
async def inference_gateway_exception_handler(
    request: Request, exc: InferenceGatewayException
) -> JSONResponse:
    """Handle custom application exceptions."""
    logger.error(
        "Application exception",
        error_code=exc.error_code,
        error_message=exc.message,
        path=request.url.path,
    )

    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={"error": exc.error_code, "message": exc.message},
    )


@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception) -> JSONResponse:
    """Handle unexpected exceptions."""
    logger.error(
        "Unhandled exception",
        error_type=type(exc).__name__,
        error_message=str(exc),
        path=request.url.path,
    )

    return JSONResponse(
        status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
        content={"error": "INTERNAL_SERVER_ERROR", "message": "Internal server error"},
    )


if __name__ == "__main__":
    import uvicorn

    uvicorn.run(
        "app.main:app",
        host=settings.host,
        port=settings.port,
        reload=settings.is_development,
        log_level=settings.log_level.lower(),
    )
```

**关键特性：**
- 启动/关闭的生命周期处理器
- 在 `/metrics` 挂载的 Prometheus metrics
- OpenTelemetry instrumentation
- CORS 中间件
- 异常处理器
- 条件性 API 文档（仅开发环境）

**✅ 检查点：** 应用成功启动：

```bash
# Start with Docker Compose
docker-compose up --build

# Or run locally
python -m app.main
```

验证端点：
- http://localhost:8000/docs (Swagger UI)
- http://localhost:8000/healthz (健康检查)
- http://localhost:8000/readyz (就绪检查)
- http://localhost:8000/metrics (Prometheus metrics)

---

## 第 8 部分：定义 SLI 和 SLO

创建 `docs/slo.md`：

```markdown
# Service Level Objectives (SLOs)

## 概述

本文档定义了 `inference-gateway` 服务的 Service Level Indicators (SLI) 和 Service Level Objectives (SLO)。

**服务**: inference-gateway
**所有者**: ML Platform Team
**最后更新**: 2025-10-23
**审查周期**: 季度

---

## SLO 概要

| SLI | 目标 | 测量窗口 | Error Budget |
|-----|-----------|-------------------|--------------|
| 可用性 | 99.5% | 28 天滚动 | 0.5%（约 3.6 小时/月） |
| P99 延迟 | < 300ms | 28 天滚动 | - |
| 错误率 | < 0.5% | 28 天滚动 | 0.5% 的请求 |
| 吞吐量 | 支持 100 req/s | 1 分钟平均 | - |

---

## 详细 SLI

### 1. 可用性

**定义**: 成功 HTTP 请求（状态码 200-399）占总请求的百分比。

**Metric**: `sum(rate(http_requests_total{status_code!~"5.."}[28d])) / sum(rate(http_requests_total[28d])) * 100`

**目标**: 99.5%

**测量窗口**: 28 天滚动窗口

**Error Budget**: 0.5% 停机时间 = 每月约 3.6 小时 = 每周约 0.86 小时

**告警**:
- **警告**: Error budget burn rate > 2x（本周剩余 0.86 小时）
- **严重**: Error budget burn rate > 5x（立即寻呼）

**理由**: 产品要求生产流量达到 99.5% 可用性。这允许每月约 3.6 小时的停机时间用于维护、事件和部署。

---

### 2. 请求延迟（P99）

**定义**: `/predict` 端点的成功请求持续时间的第 99 百分位数。

**Metric**: `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket{endpoint="/predict"}[28d]))`

**目标**: < 300ms

**测量窗口**: 28 天滚动窗口

**告警**:
- **警告**: P99 延迟 > 300ms 持续 5 分钟
- **严重**: P99 延迟 > 500ms 持续 5 分钟

**理由**: 用户体验研究表明 300ms 是"即时"响应的阈值。超过此阈值会降低用户满意度评分。

---

### 3. 错误率

**定义**: 失败请求（状态码 500-599）占总请求的百分比。

**Metric**: `sum(rate(http_requests_total{status_code=~"5.."}[28d])) / sum(rate(http_requests_total[28d])) * 100`

**目标**: < 0.5%

**测量窗口**: 28 天滚动窗口

**Error Budget**: 0.5% 的请求 = 每 100,000 个请求约 432 个错误

**告警**:
- **警告**: 错误率 > 0.5% 持续 5 分钟
- **严重**: 错误率 > 1% 持续 5 分钟

**理由**: 内部服务可靠性目标要求错误率 < 1%。我们以 0.5% 为目标，为意外故障保持缓冲。

---

### 4. 吞吐量容量

**定义**: 服务在满足延迟/错误 SLO 的同时可处理的持续每秒请求数。

**Metric**: `rate(http_requests_total{endpoint="/predict"}[1m])`

**目标**: 支持 100 req/s 持续负载

**测量窗口**: 1 分钟平均

**告警**:
- **警告**: 请求率 > 80 req/s（接近容量）
- **严重**: 请求率 > 100 req/s 且 P99 延迟 > 300ms（超出容量）

**理由**: 当前流量预测估计峰值为 50 req/s。目标 100 req/s 为增长提供了 2 倍余量。

---

## ML 特定 SLI

### 5. 模型推理延迟（P95）

**定义**: 模型推理持续时间的第 95 百分位数（不包括预处理/后处理）。

**Metric**: `histogram_quantile(0.95, rate(inference_duration_seconds_bucket{model_name="resnet50"}[28d]))`

**目标**: < 150ms

**测量窗口**: 28 天滚动窗口

**告警**:
- **警告**: P95 推理延迟 > 150ms 持续 10 分钟
- **严重**: P95 推理延迟 > 200ms 持续 5 分钟

**理由**: 推理延迟直接影响总体请求延迟。150ms 的推理时间允许约 100ms 的网络/处理开销，以保持在 300ms 总计以下。

---

### 6. 模型置信度覆盖

**定义**: 置信度 > 0.7（高置信度）的预测百分比。

**Metric**: `sum(rate(inference_confidence_score_bucket{le="1.0"}[28d])) - sum(rate(inference_confidence_score_bucket{le="0.7"}[28d])) / sum(rate(inference_confidence_score_count[28d])) * 100`

**目标**: > 80%

**测量窗口**: 28 天滚动窗口

**告警**:
- **警告**: 高置信度率 < 80% 持续 1 小时
- **严重**: 高置信度率 < 70% 持续 30 分钟

**理由**: 低置信度预测表明潜在的模型漂移、数据质量问题或分布偏移。监控此指标有助于及早发现退化。

---

## Error Budget 策略

### Error Budget 剩余

**公式**: `(1 - (actual_availability / target_availability)) * measurement_window`

**示例**: 对于 28 天内 99.5% 的目标：
- Error budget = 28 天的 0.5% = 约 3.6 小时
- 如果实际可用性为 99.7%，剩余 budget = 1.44 小时
- 如果实际可用性为 99.3%，超出 budget 1.44 小时

### 策略操作

| Budget 剩余 | 操作 |
|------------------|--------|
| > 75% | 一切照常 - 继续功能开发 |
| 50-75% | 谨慎 - 审查近期变更，加强监控 |
| 25-50% | 放慢 - 暂停非关键部署，专注于可靠性 |
| < 25% | 冻结 - 停止所有功能部署，仅关注可靠性改进 |
| 已耗尽 | 事件 - 宣布事件，全员参与恢复 |

### Budget Burn Rate 告警

**1 小时 burn rate**: 如果以当前错误率继续，1 小时内会消耗多少 budget？

**告警阈值**:
- 🟡 警告：2 倍正常 burn rate 持续 15 分钟
- 🔴 严重：5 倍正常 burn rate 持续 5 分钟

---

## 监控和 Dashboard

### 主要 Dashboard

**Grafana Dashboard**: `inference-gateway-slo-overview`

**面板**:
1. SLO 合规状态（当前 vs 目标）
2. Error Budget 剩余（仪表 + 时间序列）
3. 可用性（28 天滚动）
4. P99 延迟（时间序列）
5. 错误率（时间序列）
6. 吞吐量（req/s）
7. ML Metrics（推理延迟、置信度分布）

### 告警

所有 SLO 告警路由到：
- **警告**: #ml-platform-alerts Slack 频道
- **严重**: PagerDuty → On-call 工程师

---

## 审查和迭代

### 审查计划

- **每周**: 团队同步中的 Error budget 状态审查
- **每月**: 向利益相关者的 SLO 合规报告
- **每季度**: SLO 定义审查和调整

### 调整触发条件

在以下情况下重新评估 SLO：
- 持续超额表现（3 个月 > 99.9%）
- 持续欠佳表现（2 个月 < 99%）
- 产品需求变更
- 用户反馈表明未满足期望
- 业务优先级变化

---

## 参考

- [Google SRE Book - SLIs, SLOs, SLAs](https://sre.google/sre-book/service-level-objectives/)
- [Prometheus Queries for SLO Monitoring](https://prometheus.io/docs/prometheus/latest/querying/examples/)
- Internal: [ML Platform SLO Standards](https://wiki.internal/ml-platform-slos)
```

**✅ 检查点：** SLO 文档已创建并可访问。

---

## 第 9 部分：创建可观测性就绪评估

创建 `docs/observability-readiness.md`：

```markdown
# Observability Readiness Assessment

**服务**: inference-gateway
**评估日期**: 2025-10-23
**评估人**: Junior AI Infrastructure Engineer
**状态**: ✅ 已为生产监控集成准备就绪

---

## 执行摘要

`inference-gateway` 服务在 metrics、logs 和 traces 方面实现了**全面的可观测性覆盖**。所有关键信号均已 instrument 并准备与更广泛的 monitoring 栈（Prometheus、Grafana、Loki、Jaeger）集成。

**总体就绪度**: 95%（20/20 标准已满足）

---

## 评估标准

### 1. Metrics（Prometheus）

| 标准 | 状态 | 备注 |
|-----------|--------|-------|
| ✅ 服务暴露 `/metrics` 端点 | 完成 | Prometheus exposition 格式 |
| ✅ 四大黄金信号已 instrument | 完成 | 延迟、流量、错误、饱和度 |
| ✅ HTTP 请求 metrics | 完成 | 按方法、端点、状态的 Counter + histogram |
| ✅ 应用特定 metrics | 完成 | 推理延迟、置信度、预测 |
| ✅ 资源使用 metrics | 完成 | 模型内存、队列深度 |
| ✅ Histogram buckets 适当 | 完成 | 与 SLO 目标对齐 |
| ✅ 基数控制 | 完成 | 限制 label 值，无无限维度 |
| ✅ Metric 命名约定 | 完成 | 遵循 Prometheus 最佳实践 |

**Metrics 覆盖率**: 100%（8/8）

**可用 Metrics**:
- `http_requests_total`
- `http_request_duration_seconds`
- `http_errors_total`
- `inference_requests_total`
- `inference_duration_seconds`
- `inference_confidence_score`
- `model_predictions_total`
- `inference_queue_depth`
- `model_memory_bytes`

---

### 2. 日志（结构化 JSON）

| 标准 | 状态 | 备注 |
|-----------|--------|-------|
| ✅ 结构化 JSON 日志 | 完成 | 所有日志以 JSON 格式输出 |
| ✅ 一致的日志 schema | 完成 | 时间戳、级别、事件、服务、trace_id |
| ✅ 日志中的 Trace 上下文 | 完成 | trace_id 和 span_id 自动注入 |
| ✅ 服务元数据 | 完成 | 服务名称、版本、环境 |
| ✅ 请求关联 | 完成 | 请求 ID 和 trace ID 用于关联 |
| ✅ 适当的日志级别 | 完成 | 正确使用 DEBUG、INFO、WARNING、ERROR |
| ⚠️ 日志聚合集成 | 待定 | 将在 Exercise 04 中集成 Loki/Elasticsearch |
| ✅ PII/敏感数据脱敏 | 完成 | 日志中不包含凭据或 PII |

**日志覆盖率**: 87.5%（7/8）- 聚合待定

**示例日志条目**:
```json
{
  "event": "inference_completed",
  "timestamp": "2025-10-23T10:15:30.123456Z",
  "level": "info",
  "service": "inference-gateway",
  "version": "1.0.0",
  "environment": "development",
  "trace_id": "5f3a2b1c0d9e8f7a6b5c4d3e2f1a0b9c",
  "span_id": "1a2b3c4d5e6f7a8b",
  "model_name": "resnet50",
  "top_prediction": "golden_retriever",
  "confidence": 0.9876,
  "inference_duration_ms": 145.67
}
```

---

### 3. Tracing（OpenTelemetry）

| 标准 | 状态 | 备注 |
|-----------|--------|-------|
| ✅ OpenTelemetry SDK 已配置 | 完成 | OTLP exporter 到 collector |
| ✅ 自动 HTTP instrumentation | 完成 | FastAPI 自动 instrument |
| ✅ 自定义 span 创建 | 完成 | 用于预处理、推理、后处理的手动 spans |
| ✅ Span 属性 | 完成 | 模型名称、延迟、文件大小 |
| ✅ Trace 上下文传播 | 完成 | Trace ID 传播到日志 |
| ✅ Trace 采样 | 完成 | 默认采样（开发环境 100%） |
| ✅ OTLP collector 集成 | 完成 | 通过 collector 导出到 Jaeger |

**Tracing 覆盖率**: 100%（7/7）

**已 Instrument 的 Spans**:
- `predict_endpoint` - 整体请求 span
- `load_model` - 模型加载
- `preprocess_image` - 图像预处理
- `run_inference` - 模型推理执行
- `inference_pipeline` - 完整推理流程

---

### 4. 健康检查

| 标准 | 状态 | 备注 |
|-----------|--------|-------|
| ✅ Liveness 探测端点 | 完成 | GET /healthz |
| ✅ Readiness 探测端点 | 完成 | GET /readyz（检查模型是否加载） |
| ✅ Startup 探测（如需要） | 不适用 | 此服务不需要 |

**健康检查覆盖率**: 100%（2/2）

---

### 5. SLO 定义

| 标准 | 状态 | 备注 |
|-----------|--------|-------|
| ✅ SLI 已定义 | 完成 | 可用性、延迟、错误率、吞吐量 |
| ✅ SLO 已记录 | 完成 | 参见 docs/slo.md |
| ✅ Error budgets 已计算 | 完成 | 0.5% 停机时间 = 3.6 小时/月 |
| ✅ 告警规则已定义 | 完成 | 警告和严重阈值 |

**SLO 覆盖率**: 100%（4/4）

---

## 就绪摘要

| 类别 | 达标标准 | 百分比 | 状态 |
|----------|--------------|------------|--------|
| Metrics | 8/8 | 100% | ✅ 就绪 |
| 日志 | 7/8 | 87.5% | ⚠️ 聚合待定 |
| Tracing | 7/7 | 100% | ✅ 就绪 |
| 健康检查 | 2/2 | 100% | ✅ 就绪 |
| SLO | 4/4 | 100% | ✅ 就绪 |
| **总体** | **28/29** | **96.6%** | ✅ **生产就绪** |

---

## 差距分析

### 1. 日志聚合集成（Exercise 04）

**状态**: ⚠️ 待定
**优先级**: 高
**计划**: 在 Exercise 04（日志管道）中集成 Loki/Elasticsearch
**时间线**: 下一个练习

**操作**:
- [ ] 配置 Loki 作为日志聚合后端
- [ ] 设置日志传输（Promtail 或 Fluent Bit）
- [ ] 创建保留策略
- [ ] 在 Grafana 中构建日志探索 dashboard

---

## 生产部署清单

部署到生产之前：

- [ ] 配置生产 OTLP collector 端点
- [ ] 设置 Prometheus scrape targets
- [ ] 创建用于 SLO 监控的 Grafana dashboard
- [ ] 配置 Prometheus/Alertmanager 中的告警规则
- [ ] 设置 PagerDuty 集成
- [ ] 配置日志保留策略
- [ ] 审查和调整 trace 采样率（100% → 1-10%）
- [ ] 使用真实流量模式进行负载测试
- [ ] 验证 SLO 查询返回预期结果
- [ ] 为常见告警记录 runbooks

---

## 建议

### 立即

1. **负载测试**：运行 locust 负载测试以验证 SLO 目标在真实流量下的表现
2. **Dashboard 创建**：根据 SLO 定义构建 Grafana dashboard
3. **告警调优**：部署告警规则并调整阈值以减少噪音

### 短期（下一个 Sprint）

1. **日志聚合**：完成 Loki 集成（Exercise 04）
2. **分布式 Tracing**：对下游依赖进行 instrument（如有）
3. **成本优化**：审查生产环境的 trace 采样率

### 长期（下个季度）

1. **异常检测**：在 metrics 上实现基于 ML 的异常检测
2. **持续分析**：添加 CPU/内存分析以优化性能
3. **混沌工程**：注入故障以验证事件期间的可观测性

---

## 参考

- [OpenTelemetry Best Practices](https://opentelemetry.io/docs/concepts/signals/)
- [Prometheus Metric Types](https://prometheus.io/docs/concepts/metric_types/)
- [Structured Logging with structlog](https://www.structlog.org/en/stable/)
- Internal: [ML Platform Observability Standards](https://wiki.internal/observability)

---

**评估完成**: ✅
**下次审查**: Exercise 02（Prometheus Stack）之后
```

**✅ 检查点：** 可观测性就绪文档已完成。

---

## 第 10 部分：测试和验证

### 步骤 10.1：创建测试脚本

创建 `scripts/load_test.sh`：

```bash
#!/bin/bash
# scripts/load_test.sh
# Simple load test script using curl

set -e

SERVICE_URL="http://localhost:8000"
IMAGE_FILE="sample_image.jpg"  # You need to provide a sample image

echo "=== Load Test: Inference Gateway ==="
echo "Service: $SERVICE_URL"
echo "Requests: 50"
echo ""

# Download a sample image if not exists
if [ ! -f "$IMAGE_FILE" ]; then
    echo "Downloading sample image..."
    curl -o "$IMAGE_FILE" "https://upload.wikimedia.org/wikipedia/commons/3/3a/Cat03.jpg"
fi

echo "Running 50 requests..."

for i in {1..50}; do
    echo -n "Request $i/50... "

    response=$(curl -s -w "%{http_code}" -X POST \
        -F "file=@$IMAGE_FILE" \
        "$SERVICE_URL/predict")

    status_code="${response: -3}"

    if [ "$status_code" = "200" ]; then
        echo "✓ Success"
    else
        echo "✗ Failed (HTTP $status_code)"
    fi

    sleep 0.1
done

echo ""
echo "Load test complete!"
echo "Check metrics at: $SERVICE_URL/metrics"
echo "Check traces at: http://localhost:16686 (Jaeger UI)"
```

使其可执行：
```bash
chmod +x scripts/load_test.sh
```

### 步骤 10.2：运行负载测试

```bash
# Start services
docker-compose up -d

# Wait for services to be ready
sleep 10

# Run load test
./scripts/load_test.sh
```

### 步骤 10.3：验证可观测性

**1. 检查 Prometheus Metrics：**
```bash
curl http://localhost:8000/metrics | grep inference
```

预期输出（示例）：
```
# HELP inference_requests_total Total inference requests by model, version, and status
# TYPE inference_requests_total counter
inference_requests_total{model_name="resnet50",model_version="v1.0",status="success"} 50.0

# HELP inference_duration_seconds Model inference duration in seconds
# TYPE inference_duration_seconds histogram
inference_duration_seconds_bucket{le="0.01",model_name="resnet50",model_version="v1.0"} 0.0
inference_duration_seconds_bucket{le="0.1",model_name="resnet50",model_version="v1.0"} 35.0
inference_duration_seconds_bucket{le="0.5",model_name="resnet50",model_version="v1.0"} 50.0
inference_duration_seconds_count{model_name="resnet50",model_version="v1.0"} 50.0
inference_duration_seconds_sum{model_name="resnet50",model_version="v1.0"} 6.785
```

**2. 检查结构化日志：**
```bash
docker-compose logs inference-gateway | tail -20
```

预期是带有 trace 上下文的 JSON 日志。

**3. 在 Jaeger 中检查 Traces：**

在浏览器中打开 http://localhost:16686：
- 选择 "inference-gateway" 服务
- 点击 "Find Traces"
- 检查单个 spans

**✅ 检查点：** 所有可观测性信号（metrics、logs、traces）均正常工作。

---

## 第 11 部分：创建架构文档

创建 `docs/architecture.md`：

```markdown
# Inference Gateway - Architecture Overview

## System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Inference Gateway                         │
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   FastAPI    │───▶│   Metrics    │───▶│  Prometheus  │  │
│  │  Application │    │ (prometheus  │    │              │  │
│  │              │    │  -client)    │    │              │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                                                    │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Logging    │───▶│  Trace CTX   │───▶│     Loki     │  │
│  │ (structlog)  │    │  Injection   │    │  (planned)   │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                                                    │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Tracing    │───▶│OTLP Collector│───▶│    Jaeger    │  │
│  │(OpenTelemetry)    │              │    │              │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                                                    │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │ Model Inference                                          │
│  │  (PyTorch)   │                                           │
│  └──────────────┘                                           │
└─────────────────────────────────────────────────────────────┘
```

## 组件描述

### FastAPI Application (`app/main.py`)
- 用于推理请求的 HTTP 服务器
- 用于 metrics 和 tracing 的中间件
- 异常处理器
- 生命周期管理

### Metrics (`app/instrumentation/metrics.py`)
- Prometheus 客户端库
- 四大黄金信号实现
- ML 特定 metrics
- 用于自动跟踪的中间件

### Logging (`app/instrumentation/logging.py`)
- 结构化 JSON 日志
- Trace 上下文注入
- 服务元数据
- ISO 时间戳

### Tracing (`app/instrumentation/tracing.py`)
- OpenTelemetry SDK
- FastAPI 自动 instrument
- 手动 span 创建
- OTLP 导出

### Model Inference (`app/models/inference.py`)
- PyTorch 模型加载
- 预处理管道
- 推理执行
- 后处理
- 全面的 instrumentation

## 数据流

### 请求路径

1. **客户端** → HTTP POST /predict（图像文件）
2. **FastAPI** → MetricsMiddleware（启动计时器）
3. **FastAPI** → OpenTelemetry（创建 span）
4. **路由处理器** → 依赖注入（模型服务）
5. **模型服务** → 预处理图像（带 span）
6. **模型服务** → 运行推理（带 span）
7. **模型服务** → 后处理结果（带 span）
8. **路由处理器** → 返回响应
9. **MetricsMiddleware** → 记录 metrics
10. **OpenTelemetry** → 导出 trace 到 collector
11. **日志** → 发出带有 trace 上下文的结构化日志

### 可观测性数据流

**Metrics**：
- 由 prometheus-client 生成
- 在 GET /metrics 暴露
- 由 Prometheus 每 15 秒抓取

**Logs**：
- 由 structlog 生成
- 以 JSON 格式输出到 stdout
- （未来）由 Promtail 传输到 Loki

**Traces**：
- 由 OpenTelemetry SDK 生成
- 通过 OTLP HTTP 导出到 collector
- 转发到 Jaeger 进行可视化

## 技术栈

| 组件 | 技术 | 版本 |
|-----------|-----------|---------|
| Web 框架 | FastAPI | 0.104.1 |
| HTTP 服务器 | Uvicorn | 0.24.0 |
| ML 框架 | PyTorch | 2.1.1 |
| Metrics | prometheus-client | 0.19.0 |
| 日志 | structlog | 23.2.0 |
| Tracing | OpenTelemetry | 1.21.0 |
| 配置 | Pydantic Settings | 2.1.0 |

## 部署架构

### 开发环境（Docker Compose）

```yaml
services:
  - inference-gateway (port 8000)
  - otel-collector (port 4318)
  - jaeger (port 16686)
  - prometheus (port 9090)
```

### 生产环境（Kubernetes）- 计划中

```
┌─────────────────────────────────────────┐
│            Ingress Controller            │
└─────────────────┬───────────────────────┘
                  │
         ┌─────────▼─────────┐
         │  Service (LB)     │
         └─────────┬─────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
┌────▼────┐  ┌────▼────┐  ┌────▼────┐
│  Pod 1  │  │  Pod 2  │  │  Pod 3  │
│ Gateway │  │ Gateway │  │ Gateway │
└─────────┘  └─────────┘  └─────────┘
     │            │            │
     └────────────┼────────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
┌────▼────┐  ┌────▼────┐  ┌────▼────┐
│Prometheus│ │   Loki  │  │ Jaeger  │
└─────────┘  └─────────┘  └─────────┘
```

**✅ 检查点：** 架构文档已完成。

---

## 第 12 部分：可选增强

### 增强 1：单元测试

创建 `tests/unit/test_metrics.py`：

```python
# tests/unit/test_metrics.py
"""Unit tests for metrics instrumentation."""
import pytest
from prometheus_client import REGISTRY

from app.instrumentation import metrics


def test_metrics_exist():
    """Verify all expected metrics are registered."""
    metric_names = [
        "http_requests_total",
        "inference_requests_total",
        "http_request_duration_seconds",
        "inference_duration_seconds",
        "inference_errors_total",
    ]

    for name in metric_names:
        assert name in REGISTRY._names_to_collectors, f"Metric {name} not registered"


def test_record_prediction():
    """Test recording predictions updates counters."""
    model_name = "test_model"
    predicted_class = "test_class"
    confidence = 0.95

    # Record prediction
    metrics.record_prediction(model_name, predicted_class, confidence)

    # Verify counter incremented (check via registry)
    # In real tests, use prometheus_client test utilities
    assert True  # Placeholder
```

创建 `tests/unit/test_config.py`：

```python
# tests/unit/test_config.py
"""Unit tests for configuration."""
import pytest
from pydantic import ValidationError

from app.core.config import Settings


def test_settings_defaults():
    """Verify default settings load correctly."""
    settings = Settings()

    assert settings.service_name == "inference-gateway"
    assert settings.environment in ["development", "staging", "production"]
    assert settings.port >= 1024


def test_settings_validation():
    """Verify settings validation works."""
    with pytest.raises(ValidationError):
        Settings(port=99999)  # Port out of range
```

### 增强 2：集成测试

创建 `tests/integration/test_api.py`：

```python
# tests/integration/test_api.py
"""Integration tests for API endpoints."""
import pytest
from httpx import AsyncClient

from app.main import app


@pytest.mark.asyncio
async def test_health_check():
    """Test health check endpoint."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/healthz")

    assert response.status_code == 200
    assert response.json() == {"status": "healthy"}


@pytest.mark.asyncio
async def test_readiness_check():
    """Test readiness check endpoint."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/readyz")

    # May be 503 if model not loaded, or 200 if loaded
    assert response.status_code in [200, 503]


@pytest.mark.asyncio
async def test_metrics_endpoint():
    """Test Prometheus metrics endpoint."""
    async with AsyncClient(app=app, base_url="http://test") as client:
        response = await client.get("/metrics")

    assert response.status_code == 200
    assert "http_requests_total" in response.text
```

运行测试：
```bash
pytest tests/ -v --cov=app
```

### 增强 3：Locust 负载测试

创建 `tests/load/locustfile.py`：

```python
# tests/load/locustfile.py
"""Locust load test for inference gateway."""
from locust import HttpUser, task, between
import random


class InferenceGatewayUser(HttpUser):
    """Simulated user for load testing."""

    wait_time = between(0.5, 2)

    def on_start(self):
        """Load test image on start."""
        with open("sample_image.jpg", "rb") as f:
            self.image_bytes = f.read()

    @task(10)
    def predict(self):
        """Run prediction request."""
        files = {"file": ("test.jpg", self.image_bytes, "image/jpeg")}
        with self.client.post("/predict", files=files, catch_response=True) as response:
            if response.status_code == 200:
                response.success()
            else:
                response.failure(f"Got status {response.status_code}")

    @task(1)
    def health_check(self):
        """Health check request."""
        self.client.get("/healthz")

    @task(1)
    def readiness_check(self):
        """Readiness check request."""
        self.client.get("/readyz")
```

运行 Locust：
```bash
locust -f tests/load/locustfile.py --host=http://localhost:8000
```

访问 UI 于 http://localhost:8089

---

## 第 13 部分：更新 README

创建最终的 `README.md`。

**✅ 检查点：** README 全面且专业。

---

## 第 14 部分：总结与反思

### 总结

恭喜！你已成功构建了一个生产就绪的推理网关服务，并具有全面的可观测性 instrumentation：

**你构建的内容**：
1. ✅ 带有 ResNet-50 模型的 FastAPI 推理服务
2. ✅ Prometheus metrics（四大黄金信号 + ML 特定）
3. ✅ 带有 trace 上下文的结构化 JSON 日志
4. ✅ OpenTelemetry 分布式 tracing
5. ✅ 健康检查端点（liveness/readiness）
6. ✅ 全面的错误处理
7. ✅ 带 error budgets 的 SLO 定义
8. ✅ 可观测性就绪评估
9. ✅ Docker Compose 开发环境
10. ✅ 文档（SLO、架构、README）

**获得的技能**：
- 为微服务设计 SLI 和 SLO
- 实现四大黄金信号
- 使用可观测性对 ML 模型进行 instrument
- 结构化日志最佳实践
- OpenTelemetry tracing 模式
- Prometheus metric 设计
- Metrics 的基数控制
- Kubernetes 的健康检查模式
- 生产就绪的错误处理

### 交付物清单

- ✅ 带有 metrics/logging/tracing 的已 instrument FastAPI 服务
- ✅ 包含全面 SLO 定义的 `docs/slo.md`
- ✅ 包含就绪评估的 `docs/observability-readiness.md`
- ✅ 包含系统架构的 `docs/architecture.md`
- ✅ 记录可观测性设置的更新版 `README.md`
- ✅ 包含 OTLP collector、Jaeger、Prometheus 的 Docker Compose 栈
- ✅ 演示功能的负载测试脚本
- ✅ 所有代码结构正确并已文档化

---

## 反思问题

1. **哪些 SLI/SLO 最难定义？为什么？**
   - 考虑：ML 特定 SLI（置信度覆盖）、测量窗口、error budgets
   - 反思严格性与可实现性之间的权衡

2. **你会如何为新服务自动化可观测性就绪评估？**
   - 思考：自动检查、CI/CD 集成、标准化
   - 考虑：清单自动化、policy-as-code、服务目录

3. **如果此服务处理真实客户流量，你会捕获哪些额外的信号（metrics/logs）？**
   - 考虑：面向用户的指标、业务指标、成本跟踪
   - 思考：隐私影响、数据保留、合规性

4. **结构化日志如何在事件期间改善跨团队协作？**
   - 反思：跨服务的关联、可查询性、标准化
   - 考虑：On-call 体验、平均修复时间（MTTR）

5. **高基数 metrics 的风险是什么？你是如何缓解的？**
   - 考虑：Label 设计、聚合策略、采样
   - 思考：成本影响、查询性能、存储

6. **你如何调整生产环境中的 trace 采样以平衡成本和可见性？**
   - 考虑：基于 head 的 vs 基于 tail 的采样、错误采样、重要性评分
   - 思考：成本效益分析、保留策略

---

## 下一步

**在 Exercise 02**（Prometheus Stack）中，你将：
- 设置完整的 Prometheus monitoring 栈
- 创建 recording 和 alerting rules
- 构建全面的 service discovery 设置
- 实现带路由和静默的 alertmanager
- 创建用于可用性测试的合成监控器

**继续到：** [Exercise 02: Prometheus Monitoring Stack](exercise-02-prometheus-stack.md)

---

**Exercise 01 完成！** 🎉

你已构建了坚实的可观测性基础，它将作为 Module 009 其余部分的基础。
