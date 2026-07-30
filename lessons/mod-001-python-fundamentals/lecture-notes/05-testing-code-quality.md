# Lecture 05：使用 pytest 进行测试与代码质量

## 目录
1. [引言](#引言)
2. [使用 pytest 进行测试](#使用-pytest-进行测试)
3. [代码质量与工具](#代码质量与工具)
4. [总结](#总结)

---

## 引言

本讲座涵盖 AI 基础设施中专业 Python 开发的两个关键主题：

1. **测试**：使用 pytest 构建全面的测试套件，确保代码可靠性
2. **代码质量**：使用 linter、formatter 和类型检查器维护专业标准

这些技能对于构建可靠、可维护和高性能的生产级 AI 基础设施系统至关重要。

### 为什么这些主题很重要

**在 AI 基础设施中**：
- **测试**：确保基础设施代码在部署到生产前正确运行
- **代码质量**：维护多个工程师能够有效协作的代码库
- **可靠性**：在开发周期的早期捕获 bug
- **可维护性**：保持代码在团队中可读且一致

### 学习目标

完成本讲座后，你将能够：
- 使用 pytest 编写全面的单元测试
- 使用夹具提供可重用的测试数据和设置
- 使用 pytest-asyncio 测试异步函数
- 模拟外部依赖以隔离测试
- 使用代码格式化工具（Black）、linter（Ruff）和类型检查器（mypy）
- 设置 pre-commit 钩子以自动化质量检查
- 理解代码覆盖率和质量指标

本讲座建立在你在讲座 1-4 中学到的所有内容之上，帮助你保持 Python 代码的专业标准。

---

## 使用 pytest 进行测试

### 为什么测试很重要

**AI 基础设施中的测试**：
- 在部署前捕获 bug
- 确保基础设施更改不会破坏现有功能
- 通过测试用例提供文档
- 支持自信的重构

### 设置 pytest

```bash
# 安装 pytest 和插件
pip install pytest pytest-cov pytest-asyncio pytest-mock

# 项目结构
my_project/
├── src/
│   ├── __init__.py
│   ├── models.py
│   └── deployment.py
├── tests/
│   ├── __init__.py
│   ├── test_models.py
│   └── test_deployment.py
├── pytest.ini
└── requirements-dev.txt
```

### 编写你的第一个测试

```python
# src/calculator.py
def add(a: int, b: int) -> int:
    """两个数相加"""
    return a + b

def divide(a: float, b: float) -> float:
    """两个数相除"""
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

# tests/test_calculator.py
import pytest
from src.calculator import add, divide

def test_add_positive_numbers():
    """测试正数相加"""
    assert add(2, 3) == 5

def test_add_negative_numbers():
    """测试负数相加"""
    assert add(-5, 3) == -2

def test_divide_normal():
    """测试正常除法"""
    assert divide(10, 2) == 5.0

def test_divide_by_zero():
    """测试除以零是否引发 ValueError"""
    with pytest.raises(ValueError, match="Cannot divide by zero"):
        divide(10, 0)
```

### 测试夹具

夹具提供可重用的测试数据和设置：

```python
import pytest
from typing import Dict, List

@pytest.fixture
def sample_model_config() -> Dict:
    """提供示例模型配置"""
    return {
        "model_name": "fraud_detector",
        "version": "1.0.0",
        "framework": "sklearn",
        "parameters": {
            "n_estimators": 100,
            "max_depth": 10
        }
    }

@pytest.fixture
def sample_training_data() -> List[Dict]:
    """提供示例训练数据"""
    return [
        {"features": [1, 2, 3], "label": 0},
        {"features": [4, 5, 6], "label": 1},
        {"features": [7, 8, 9], "label": 0}
    ]

def test_model_initialization(sample_model_config):
    """测试使用配置初始化模型"""
    from src.models import MLModel

    model = MLModel(sample_model_config)
    assert model.name == "fraud_detector"
    assert model.version == "1.0.0"

def test_model_training(sample_model_config, sample_training_data):
    """测试模型训练"""
    from src.models import MLModel

    model = MLModel(sample_model_config)
    result = model.train(sample_training_data)
    assert result["status"] == "success"
    assert "accuracy" in result
```

### 参数化测试

用一个测试函数测试多个场景：

```python
import pytest

@pytest.mark.parametrize("input_value,expected", [
    (0, "zero"),
    (1, "positive"),
    (-1, "negative"),
    (100, "positive"),
    (-50, "negative")
])
def test_classify_number(input_value, expected):
    """测试数字分类"""
    from src.utils import classify_number
    assert classify_number(input_value) == expected

@pytest.mark.parametrize("model_type,framework", [
    ("classification", "sklearn"),
    ("regression", "sklearn"),
    ("neural_network", "pytorch"),
    ("ensemble", "xgboost")
])
def test_model_creation(model_type, framework):
    """测试创建不同模型类型"""
    from src.models import create_model
    model = create_model(model_type, framework)
    assert model is not None
    assert model.model_type == model_type
```

### 测试异步函数

```python
import pytest
import asyncio

# 使用 @pytest.mark.asyncio 标记异步测试
@pytest.mark.asyncio
async def test_async_model_prediction():
    """测试异步模型预测"""
    from src.async_model import AsyncModelClient

    async with AsyncModelClient() as client:
        result = await client.predict({"features": [1, 2, 3]})
        assert "prediction" in result
        assert isinstance(result["prediction"], (int, float))

@pytest.mark.asyncio
async def test_concurrent_predictions():
    """测试多个并发预测"""
    from src.async_model import AsyncModelClient

    async with AsyncModelClient() as client:
        tasks = [
            client.predict({"features": [i, i+1, i+2]})
            for i in range(5)
        ]
        results = await asyncio.gather(*tasks)
        assert len(results) == 5
        assert all("prediction" in r for r in results)
```

### 模拟与修补

模拟外部依赖：

```python
import pytest
from unittest.mock import Mock, patch, AsyncMock

def test_with_mock_database():
    """使用模拟数据库进行测试"""
    from src.storage import DataStore

    # 创建模拟数据库连接
    mock_db = Mock()
    mock_db.query.return_value = [{"id": 1, "value": "test"}]

    store = DataStore(mock_db)
    result = store.get_all()

    assert len(result) == 1
    mock_db.query.assert_called_once()

@pytest.mark.asyncio
async def test_with_mock_api():
    """使用模拟异步 API 进行测试"""
    from src.api_client import ModelAPIClient

    # 为异步函数创建模拟
    mock_response = AsyncMock()
    mock_response.return_value = {"status": "success", "prediction": 0.95}

    with patch("httpx.AsyncClient.post", mock_response):
        client = ModelAPIClient()
        result = await client.predict({"features": [1, 2, 3]})
        assert result["status"] == "success"
```

### 测试异常和错误

```python
import pytest

def test_invalid_model_config():
    """测试无效配置是否引发 ValueError"""
    from src.models import MLModel

    invalid_config = {"model_name": ""}  # 空名称

    with pytest.raises(ValueError, match="model_name cannot be empty"):
        MLModel(invalid_config)

def test_model_prediction_with_invalid_input():
    """测试使用无效输入形状进行预测"""
    from src.models import MLModel

    model = MLModel({"model_name": "test", "input_dim": 10})

    # 输入具有错误的维度
    with pytest.raises(ValueError, match="Expected 10 features"):
        model.predict([1, 2, 3])  # 只有 3 个特征
```

### 测试覆盖率

```bash
# 带覆盖率运行测试
pytest --cov=src --cov-report=html --cov-report=term

# 查看覆盖率报告
# Coverage report will be in htmlcov/index.html

# 设置最小覆盖率要求
pytest --cov=src --cov-fail-under=80
```

### pytest 配置

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*

markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    unit: marks tests as unit tests
    asyncio: marks async tests

addopts =
    -v
    --strict-markers
    --tb=short
    --cov=src
    --cov-report=term-missing
    --cov-fail-under=70

asyncio_mode = auto
```

---

## 代码质量与工具

### 使用 Black 进行代码格式化

**Black** 是不妥协的 Python 代码格式化工具：

```bash
# 安装
pip install black

# 格式化所有 Python 文件
black src/ tests/

# 检查但不修改
black --check src/

# 设置行长度限制
black --line-length 100 src/
```

**配置**：
```toml
# pyproject.toml
[tool.black]
line-length = 100
target-version = ['py311']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.venv
  | build
  | dist
)/
'''
```

### 使用 isort 进行导入排序

```bash
# 安装
pip install isort

# 排序导入
isort src/ tests/

# 检查但不修改
isort --check-only src/
```

**配置**：
```toml
# pyproject.toml
[tool.isort]
profile = "black"
line_length = 100
multi_line_output = 3
include_trailing_comma = true
force_grid_wrap = 0
use_parentheses = true
ensure_newline_before_comments = true
```

### 使用 Ruff 进行代码检查

**Ruff** 是一个快速的 Python linter：

```bash
# 安装
pip install ruff

# 检查代码
ruff check src/ tests/

# 自动修复问题
ruff check --fix src/

# 监视模式
ruff check --watch src/
```

**配置**：
```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py311"

select = [
    "E",   # pycodestyle 错误
    "W",   # pycodestyle 警告
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
]

ignore = [
    "E501",  # 行太长（由 black 处理）
]

[tool.ruff.per-file-ignores]
"tests/*" = ["S101"]  # 允许在测试中使用 assert
```

### 使用 mypy 进行类型检查

```bash
# 安装
pip install mypy

# 类型检查代码
mypy src/

# 严格模式
mypy --strict src/

# 检查特定文件
mypy src/models.py src/deployment.py
```

**配置**：
```ini
# mypy.ini
[mypy]
python_version = 3.11
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
disallow_any_generics = True
check_untyped_defs = True
no_implicit_optional = True
warn_redundant_casts = True
warn_unused_ignores = True
warn_no_return = True
warn_unreachable = True
strict_equality = True

[mypy-tests.*]
disallow_untyped_defs = False
```

### Pre-commit 钩子

自动化代码质量检查：

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.5.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-added-large-files

  - repo: https://github.com/psf/black
    rev: 23.11.0
    hooks:
      - id: black

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort

  - repo: https://github.com/charliermarsh/ruff-pre-commit
    rev: v0.1.6
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.7.1
    hooks:
      - id: mypy
        additional_dependencies: [types-all]
```

```bash
# 安装 pre-commit
pip install pre-commit

# 安装钩子
pre-commit install

# 手动运行
pre-commit run --all-files
```

### 完整开发工作流

```bash
# 1. 格式化代码
black src/ tests/
isort src/ tests/

# 2. 检查代码
ruff check src/ tests/

# 3. 类型检查
mypy src/

# 4. 运行测试
pytest tests/ -v --cov=src

# 5. 检查覆盖率
pytest --cov=src --cov-report=html

# 或使用 Makefile
make format lint typecheck test
```

**Makefile**：
```makefile
.PHONY: format lint typecheck test quality

format:
	black src/ tests/
	isort src/ tests/

lint:
	ruff check src/ tests/

typecheck:
	mypy src/

test:
	pytest tests/ -v --cov=src

quality: format lint typecheck test
	@echo "All quality checks passed!"
```

---

## 总结

### 关键要点

#### 测试
- 使用 pytest 编写测试以确保可靠性
- 使用夹具提供可重用的测试数据
- 参数化测试以覆盖多个场景
- 模拟外部依赖以隔离测试
- 目标达到 >80% 的代码覆盖率
- 使用 `@pytest.mark.asyncio` 测试异步函数

#### 代码质量
- **Black**：自动代码格式化
- **isort**：导入语句组织
- **Ruff**：快速代码检查，支持自动修复
- **mypy**：静态类型检查
- **Pre-commit 钩子**：自动化质量检查

### AI 基础设施的最佳实践

1. **测试所有内容**：
   - 模型部署逻辑
   - 配置解析
   - API 端点
   - 错误处理
   - 边界情况

2. **维护质量**：
   - 使用格式化工具（无需争论风格）
   - 运行 linter（尽早捕获 bug）
   - 类型检查（记录接口）
   - 使用 pre-commit 钩子自动化

3. **覆盖率目标**：
   - 目标达到 >80% 的测试覆盖率
   - 首先关注关键路径
   - 测试错误条件
   - 模拟外部依赖

### 完成 Module 001

恭喜！你已经完成了 Module 001：AI 基础设施的 Python 基础的所有五个讲座。

**你已经学到了**：
- 环境和依赖管理（Lecture 01）
- 高级 Python 模式（Lecture 02）
- Python 用于 DevOps 操作（Lecture 03）
- 异步编程（Lecture 04）
- 测试与代码质量（Lecture 05）

**下一步**：
- 完成**练习 06：异步编程** - 构建并发模型监控
- 完成**练习 07：测试** - 编写全面的测试套件
- 在项目中练习使用代码质量工具
- 在所有仓库中设置 pre-commit 钩子
- 进入**Module 002：AI 基础设施的 Linux 基础**

### 附加资源

- [pytest 文档](https://docs.pytest.org/)
- [Black 文档](https://black.readthedocs.io/)
- [Ruff 文档](https://docs.astral.sh/ruff/)
- [mypy 文档](https://mypy.readthedocs.io/)
- [Real Python：Python 测试](https://realpython.com/python-testing/)
- [Pre-commit 框架](https://pre-commit.com/)

---

**Module 001, Lecture 05 完成！**

你现在拥有以下技能：
- 为并发操作编写高效的异步代码
- 使用 pytest 构建全面的测试套件
- 维护专业的代码质量标准
- 有效使用现代 Python 工具

这些技能构成了专业 AI 基础设施开发的基础。

**讲座版本**：1.0
**最后更新**：2025 年 10 月
**预计时长**：8-10 小时
