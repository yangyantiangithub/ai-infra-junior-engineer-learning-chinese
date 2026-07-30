# 练习 05：协作与 Pull Request

## 概述

**难度**：中级
**预计时间**：90-120 分钟
**前置条件**：练习 04 - 合并与冲突，讲座 03 - 协作

在本练习中，你将学习协作式 Git 工作流，包括 fork 仓库、创建 pull request、进行代码审查和使用远程仓库。你将练习专业 ML 工程团队中使用的完整 GitHub 协作工作流。

## 学习目标

完成本练习后，你将：

- Fork 和 clone 仓库
- 使用远程仓库
- 创建和管理 pull request
- 进行有效的代码审查
- 理解 PR 最佳实践
- 处理审查反馈
- 使用 GitHub CLI 提高效率
- 在 ML 项目上协作
- 管理上游同步

## 场景

你将加入一个使用 GitHub 进行协作的 ML 基础设施团队。你将通过 fork 仓库、实现功能、创建 pull request 和参与代码审查来为团队的推理 API 项目做贡献。

## 前置条件

```bash
# 确保你已安装 GitHub CLI
gh --version

# 如果未安装：
# Mac：brew install gh
# Linux：从 https://cli.github.com/ 下载
# Windows：从 https://cli.github.com/ 下载

# 使用 GitHub 进行身份验证
gh auth login

# 按照提示进行身份验证
```

---

## 第一部分：Fork 和 Clone

### 任务 1.1：Fork 一个仓库

**说明：**

对于本练习，我们将在本地模拟 fork 工作流。在实际场景中：

```bash
# 在 GitHub 网站上：
# 1. 导航到仓库
# 2. 点击"Fork"按钮
# 3. Fork 到你的账号

# 使用 GitHub CLI：
# gh repo fork <owner>/<repo> --clone

# 对于本练习，创建一个模拟的"upstream"仓库
cd /tmp
mkdir ml-inference-api-upstream
cd ml-inference-api-upstream
git init
git config init.defaultBranch main

# 创建基本项目
cat > README.md << 'EOF'
# ML Inference API (Team Repository)

Production ML inference service.

## Contributing

1. Fork this repository
2. Create a feature branch
3. Make your changes
4. Submit a pull request
EOF

git add README.md
git commit -m "Initial commit"

# 标记为 v1.0.0
git tag -a v1.0.0 -m "Initial release"

# 现在"fork"它（通过克隆模拟）
cd ~
git clone /tmp/ml-inference-api-upstream ml-inference-api-fork
cd ml-inference-api-fork

# 添加上游 remote
git remote add upstream /tmp/ml-inference-api-upstream
git remote -v
```

**输出：**
```
origin  /tmp/ml-inference-api-upstream (fetch)
origin  /tmp/ml-inference-api-upstream (push)
upstream        /tmp/ml-inference-api-upstream (fetch)
upstream        /tmp/ml-inference-api-upstream (push)
```

---

### 任务 1.2：与上游同步

保持你的 fork 更新。

**说明：**

```bash
# 从上游获取更新
git fetch upstream

# 查看上游分支
git branch -r

# 将上游变更合并到你的 main
git switch main
git merge upstream/main

# 替代方案：使用 rebase 代替 merge
# git rebase upstream/main

# 推送到你的 fork
# git push origin main
```

---

## 第二部分：创建 Pull Request

### 任务 2.1：为 PR 创建功能分支

实现一个用于提交 pull request 的功能。

**说明：**

```bash
# 始终从最新的 main 创建分支
git switch main
git pull upstream main

# 使用描述性名称创建功能分支
git switch -c feature/add-model-metrics

# 实现功能
mkdir -p src/monitoring
cat > src/monitoring/model_metrics.py << 'EOF'
"""
Model Performance Metrics

Track model-specific performance indicators.
"""

from dataclasses import dataclass
from typing import Dict, List
import numpy as np


@dataclass
class ModelMetrics:
    """Track model performance metrics."""

    predictions: List[float] = None
    ground_truth: List[float] = None

    def __post_init__(self):
        """Initialize lists if None."""
        if self.predictions is None:
            self.predictions = []
        if self.ground_truth is None:
            self.ground_truth = []

    def add_prediction(self, prediction: float, truth: float = None):
        """
        Record a prediction.

        Args:
            prediction: Model prediction
            truth: Ground truth value (if available)
        """
        self.predictions.append(prediction)
        if truth is not None:
            self.ground_truth.append(truth)

    def calculate_accuracy(self) -> float:
        """
        Calculate prediction accuracy.

        Returns:
            Accuracy percentage (0-100)
        """
        if not self.predictions or not self.ground_truth:
            return 0.0

        if len(self.predictions) != len(self.ground_truth):
            raise ValueError("Predictions and ground truth lengths don't match")

        correct = sum(
            1 for pred, truth in zip(self.predictions, self.ground_truth)
            if abs(pred - truth) < 0.01
        )

        return (correct / len(self.predictions)) * 100

    def get_summary(self) -> Dict:
        """
        Get metrics summary.

        Returns:
            Dictionary of metrics
        """
        return {
            "total_predictions": len(self.predictions),
            "accuracy": self.calculate_accuracy() if self.ground_truth else None,
            "mean_prediction": np.mean(self.predictions) if self.predictions else 0,
            "std_prediction": np.std(self.predictions) if self.predictions else 0
        }


# Global metrics instance
model_metrics = ModelMetrics()
EOF

# 添加测试
mkdir -p tests/monitoring
cat > tests/monitoring/test_model_metrics.py << 'EOF'
"""Tests for model metrics."""

import pytest
from src.monitoring.model_metrics import ModelMetrics


def test_model_metrics_initialization():
    """Test metrics initialization."""
    metrics = ModelMetrics()
    assert metrics.predictions == []
    assert metrics.ground_truth == []


def test_add_prediction():
    """Test adding predictions."""
    metrics = ModelMetrics()
    metrics.add_prediction(0.95, 1.0)
    metrics.add_prediction(0.87, 1.0)

    assert len(metrics.predictions) == 2
    assert len(metrics.ground_truth) == 2


def test_calculate_accuracy():
    """Test accuracy calculation."""
    metrics = ModelMetrics()
    metrics.add_prediction(1.0, 1.0)
    metrics.add_prediction(1.0, 1.0)
    metrics.add_prediction(0.0, 1.0)

    accuracy = metrics.calculate_accuracy()
    assert accuracy == pytest.approx(66.67, rel=0.1)


def test_get_summary():
    """Test metrics summary."""
    metrics = ModelMetrics()
    metrics.add_prediction(0.9)
    metrics.add_prediction(0.8)

    summary = metrics.get_summary()
    assert summary["total_predictions"] == 2
    assert "mean_prediction" in summary
EOF

# 使用好的消息提交
git add src/monitoring/model_metrics.py
git commit -m "feat: add model performance metrics tracking

Implement ModelMetrics class for tracking:
- Prediction accuracy
- Statistical summaries
- Ground truth comparison

This enables monitoring model performance in production."

git add tests/monitoring/test_model_metrics.py
git commit -m "test: add tests for model metrics

Add comprehensive tests for ModelMetrics:
- Initialization
- Adding predictions
- Accuracy calculation
- Summary generation

All tests passing with 100% coverage."

# 更新文档
cat >> README.md << 'EOF'

## Model Monitoring

Track model performance with `ModelMetrics`:

```python
from src.monitoring.model_metrics import model_metrics

# Record predictions
model_metrics.add_prediction(prediction=0.95, truth=1.0)

# Get summary
summary = model_metrics.get_summary()
```
EOF

git add README.md
git commit -m "docs: add model metrics usage to README"

# 查看你的 commit
git log --oneline origin/main..HEAD
```

---

### 任务 2.2：推送并创建 Pull Request

提交你的功能进行审查。

**说明：**

```bash
# 将功能分支推送到你的 fork
# git push origin feature/add-model-metrics

# 使用 GitHub CLI 创建 PR
# gh pr create --title "Add model performance metrics" \
#              --body "## Summary
#
# Adds ModelMetrics class for tracking model performance in production.
#
# ## Changes
# - New ModelMetrics class with accuracy tracking
# - Comprehensive test coverage
# - Documentation updates
#
# ## Testing
# - All tests passing
# - Manual testing completed
#
# ## Screenshots
# (Add if relevant)
#
# Closes #123" \
#              --base main

# 或在 GitHub 网站上创建 PR：
# 1. 转到你的 fork
# 2. 点击"Compare & pull request"
# 3. 填写标题和描述
# 4. 点击"Create pull request"

# 对于本练习，模拟 PR 创建
cat > .git/PR_TEMPLATE.md << 'EOF'
# Pull Request: Add Model Performance Metrics

## Summary
Adds ModelMetrics class for tracking model performance in production.

## Type of Change
- [x] New feature
- [ ] Bug fix
- [ ] Breaking change
- [ ] Documentation update

## Changes
- New `ModelMetrics` class with accuracy tracking
- Comprehensive test coverage (100%)
- Documentation updates in README

## Testing
- [x] All existing tests passing
- [x] New tests added and passing
- [x] Manual testing completed
- [ ] Integration tests updated

## Checklist
- [x] Code follows project style guidelines
- [x] Self-review completed
- [x] Comments added for complex logic
- [x] Documentation updated
- [x] No new warnings generated
- [x] Tests added for new functionality
- [x] All tests passing

## Related Issues
Closes #123

## Additional Notes
This is foundational work for the model monitoring dashboard (future PR).

## Reviewer Notes
Please pay special attention to:
- Accuracy calculation logic
- Error handling in edge cases
EOF

echo "PR template created! This would be submitted to GitHub."
```

---

## 第三部分：代码审查流程

### 任务 3.1：审查 Pull Request

练习审查代码。

**说明：**

```bash
# 模拟收到一个需要审查的 PR

# 创建一个模拟他人工作的分支
git switch main
git switch -c review/teammate-pr

# 添加一些有问题的代码
cat > src/utils/image_processing.py << 'EOF'
"""Image processing utilities."""

def resize_image(img, size):
    # TODO: Add validation
    return img.resize(size)

def normalize(img):
    # FIXME: This doesn't handle edge cases
    return img / 255.0
EOF

git add src/utils/image_processing.py
git commit -m "Add image processing utils"

# 审查清单：
cat > review_checklist.md << 'EOF'
# Code Review Checklist

## Functionality
- [ ] Code does what it claims to do
- [ ] Edge cases handled
- [ ] Error handling present
- [ ] No obvious bugs

## Code Quality
- [ ] Follows project style guide
- [ ] DRY principle followed
- [ ] Functions are focused and single-purpose
- [ ] Variable/function names are clear
- [ ] No unnecessary complexity

## Testing
- [ ] Tests exist and pass
- [ ] Edge cases tested
- [ ] Test coverage adequate
- [ ] Tests are maintainable

## Documentation
- [ ] Code is self-documenting or well-commented
- [ ] Docstrings present
- [ ] README updated if needed
- [ ] API documentation updated

## Security
- [ ] No secrets in code
- [ ] Input validation present
- [ ] No SQL injection vulnerabilities
- [ ] Dependencies are safe

## Performance
- [ ] No obvious performance issues
- [ ] Algorithms are efficient
- [ ] No memory leaks
- [ ] Caching used where appropriate

## ML-Specific
- [ ] Model versioning handled
- [ ] Data validation present
- [ ] Metrics tracked
- [ ] Reproducibility maintained
EOF

# 要留下的审查评论：
cat > review_comments.md << 'EOF'
# Review Comments for PR #123

## Summary
Good start on image processing utilities! A few suggestions below.

## Comments

### src/utils/image_processing.py

**Line 5: `resize_image` function**
```suggestion
def resize_image(img, size):
    """
    Resize image to specified dimensions.

    Args:
        img: PIL Image object
        size: Tuple of (width, height)

    Returns:
        Resized PIL Image

    Raises:
        ValueError: If size is invalid
    """
    if not isinstance(size, tuple) or len(size) != 2:
        raise ValueError("Size must be tuple of (width, height)")
    if not all(isinstance(x, int) and x > 0 for x in size):
        raise ValueError("Dimensions must be positive integers")
    return img.resize(size)
```

Please add:
1. Docstring
2. Input validation
3. Type hints

**Line 9: `normalize` function**
⚠️ **Issue**: Division by 255.0 assumes RGB images. What about grayscale or other formats?

```suggestion
def normalize(img, max_value=255.0):
    """
    Normalize image pixel values to [0, 1] range.

    Args:
        img: Image array
        max_value: Maximum pixel value (default 255 for RGB)

    Returns:
        Normalized image array
    """
    if img.max() == 0:
        return img  # Avoid division by zero
    return img / max_value
```

## Required Changes
- [ ] Add input validation to `resize_image`
- [ ] Add docstrings to all functions
- [ ] Handle edge cases in `normalize`
- [ ] Add unit tests

## Nice to Have
- [ ] Add type hints
- [ ] Add integration test
- [ ] Consider adding batch processing support

## Verdict
**Request Changes** - Please address the required changes above.

Great work overall! Just needs a bit more robustness.
EOF

echo "Review completed! Comments ready to submit."
```

---

### 任务 3.2：回应审查反馈

回应代码审查评论。

**说明：**

```bash
# 回应反馈
cat > src/utils/image_processing.py << 'EOF'
"""
Image Processing Utilities

Common image preprocessing operations.
"""

from typing import Tuple
from PIL import Image
import numpy as np


def resize_image(img: Image.Image, size: Tuple[int, int]) -> Image.Image:
    """
    Resize image to specified dimensions.

    Args:
        img: PIL Image object
        size: Tuple of (width, height)

    Returns:
        Resized PIL Image

    Raises:
        ValueError: If size is invalid
        TypeError: If img is not a PIL Image
    """
    if not isinstance(img, Image.Image):
        raise TypeError("img must be a PIL Image")

    if not isinstance(size, tuple) or len(size) != 2:
        raise ValueError("Size must be tuple of (width, height)")

    if not all(isinstance(x, int) and x > 0 for x in size):
        raise ValueError("Dimensions must be positive integers")

    return img.resize(size)


def normalize(
    img: np.ndarray,
    max_value: float = 255.0,
    epsilon: float = 1e-8
) -> np.ndarray:
    """
    Normalize image pixel values to [0, 1] range.

    Args:
        img: Image array (numpy)
        max_value: Maximum pixel value (default 255 for RGB)
        epsilon: Small value to prevent division by zero

    Returns:
        Normalized image array

    Raises:
        ValueError: If img is empty or max_value <= 0
    """
    if img.size == 0:
        raise ValueError("Cannot normalize empty image")

    if max_value <= 0:
        raise ValueError("max_value must be positive")

    # Prevent division by zero
    actual_max = max(img.max(), epsilon)

    return img / actual_max
EOF

# 添加测试
cat > tests/utils/test_image_processing.py << 'EOF'
"""Tests for image processing utilities."""

import pytest
import numpy as np
from PIL import Image
from src.utils.image_processing import resize_image, normalize


def test_resize_image_valid():
    """Test resizing with valid input."""
    img = Image.new('RGB', (100, 100))
    resized = resize_image(img, (50, 50))
    assert resized.size == (50, 50)


def test_resize_image_invalid_size():
    """Test resizing with invalid size."""
    img = Image.new('RGB', (100, 100))
    with pytest.raises(ValueError):
        resize_image(img, (50,))  # Wrong tuple length


def test_resize_image_negative_size():
    """Test resizing with negative dimensions."""
    img = Image.new('RGB', (100, 100))
    with pytest.raises(ValueError):
        resize_image(img, (-50, 50))


def test_normalize_standard():
    """Test normalization with standard RGB values."""
    img = np.array([0, 127, 255])
    normalized = normalize(img, max_value=255.0)
    assert normalized[0] == pytest.approx(0.0)
    assert normalized[2] == pytest.approx(1.0)


def test_normalize_empty_image():
    """Test normalization with empty array."""
    img = np.array([])
    with pytest.raises(ValueError):
        normalize(img)


def test_normalize_all_zeros():
    """Test normalization with all-zero image."""
    img = np.zeros((10, 10))
    normalized = normalize(img)
    assert np.all(normalized == 0.0)
EOF

git add src/utils/image_processing.py tests/utils/test_image_processing.py
git commit -m "refactor: address code review feedback

Changes:
- Added type hints to all functions
- Added comprehensive docstrings
- Added input validation
- Added edge case handling in normalize()
- Added unit tests with 100% coverage

Addresses review comments from @reviewer"

# 留下评论
cat > review_response.md << 'EOF'
# Response to Review Comments

Thanks for the thorough review! I've addressed all the feedback:

## Changes Made

✅ Added input validation to `resize_image`
✅ Added docstrings to all functions
✅ Handle edge cases in `normalize` (division by zero, empty arrays)
✅ Added comprehensive unit tests

## Additional Improvements

- Added type hints throughout
- Increased test coverage to 100%
- Added more specific error messages

## Test Results

```
tests/utils/test_image_processing.py ......  [100%]

6 passed in 0.12s
```

Ready for another review!
EOF

echo "Feedback addressed! Ready for re-review."
```

---

## 第四部分：协作最佳实践

### 任务 4.1：编写好的 PR 描述

**说明：**

```bash
# 优秀 PR 描述的模板
cat > .github/PULL_REQUEST_TEMPLATE.md << 'EOF'
## Description
<!-- Provide a brief description of the changes -->

## Motivation and Context
<!-- Why is this change needed? What problem does it solve? -->
<!-- Link to related issues: Fixes #123, Relates to #456 -->

## Type of Change
<!-- Mark relevant items with [x] -->
- [ ] 🐛 Bug fix (non-breaking change fixing an issue)
- [ ] ✨ New feature (non-breaking change adding functionality)
- [ ] 💥 Breaking change (fix or feature causing existing functionality to break)
- [ ] 📝 Documentation update
- [ ] 🎨 Code style update (formatting, renaming)
- [ ] ♻️ Code refactoring
- [ ] ⚡ Performance improvement
- [ ] ✅ Test update

## Changes Made
<!-- List the specific changes in this PR -->
-
-
-

## Testing
<!-- Describe the tests you ran -->
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed
- [ ] All existing tests pass

### Test Results
```
# Paste test output here
```

## Screenshots (if applicable)
<!-- Add screenshots for UI changes -->

## Checklist
- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my code
- [ ] I have commented complex code sections
- [ ] I have updated the documentation
- [ ] My changes generate no new warnings
- [ ] I have added tests that prove my fix/feature works
- [ ] New and existing tests pass locally
- [ ] Any dependent changes have been merged

## Performance Impact
<!-- If applicable, describe performance implications -->
- Latency:
- Memory:
- Throughput:

## Security Considerations
<!-- If applicable, describe security implications -->

## Deployment Notes
<!-- Special deployment considerations -->
- Database migrations:
- Configuration changes:
- Dependencies added:

## Reviewer Focus Areas
<!-- Guide reviewers on what to focus on -->
Please pay special attention to:
-
-

## Additional Context
<!-- Any other context about the PR -->
EOF

git add .github/PULL_REQUEST_TEMPLATE.md
git commit -m "chore: add comprehensive PR template"
```

---

### 任务 4.2：有效的代码审查评论

**说明：**

创建代码审查指南：

```bash
cat > docs/CODE_REVIEW_GUIDE.md << 'EOF'
# Code Review Guidelines

## For Authors

### Before Creating PR
1. **Self-review** your code
2. **Run all tests** locally
3. **Update documentation**
4. **Write clear PR description**
5. **Link related issues**

### PR Title Format
```
<type>(<scope>): <subject>

Examples:
feat(api): add batch inference endpoint
fix(preprocessing): handle grayscale images correctly
docs(readme): update installation instructions
```

### PR Description
- **What** changed
- **Why** it changed
- **How** to test it
- **Any** breaking changes

## For Reviewers

### Review Checklist
- [ ] Code correctness
- [ ] Test coverage
- [ ] Documentation
- [ ] Performance
- [ ] Security
- [ ] ML-specific concerns

### Comment Types

**Blocking Issues** (must fix):
```
🚨 BLOCKER: This causes a memory leak in production.
```

**Important Suggestions**:
```
⚠️ Consider: Using a connection pool would improve performance.
```

**Nitpicks** (optional):
```
💅 Nitpick: Variable name could be more descriptive.
```

**Praise**:
```
👍 Nice: Excellent error handling!
```

**Questions**:
```
❓ Question: Why did you choose this approach over X?
```

### Comment Tone
✅ **Good**: "This function could benefit from error handling for null inputs."
❌ **Bad**: "This is wrong. You forgot error handling."

✅ **Good**: "Have you considered using a set here for O(1) lookup?"
❌ **Bad**: "Why aren't you using a set? This is inefficient."

### Approval Criteria
- ✅ **Approve**: No blocking issues, minor suggestions only
- 🔄 **Request Changes**: Blocking issues present
- 💬 **Comment**: Questions or non-blocking suggestions

## ML-Specific Review Points

### Model Changes
- [ ] Model versioning updated
- [ ] Backward compatibility maintained
- [ ] Migration path documented

### Data Processing
- [ ] Input validation present
- [ ] Data privacy maintained
- [ ] Reproducibility ensured

### Metrics
- [ ] Appropriate metrics tracked
- [ ] Logging comprehensive
- [ ] Monitoring alerts updated

### Performance
- [ ] Inference latency acceptable
- [ ] Memory usage reasonable
- [ ] Batching optimized
EOF

git add docs/CODE_REVIEW_GUIDE.md
git commit -m "docs: add code review guidelines"
```

---

## 第五部分：高级协作

### 任务 5.1：处理多个贡献者

管理并发 PR 的冲突。

**说明：**

```bash
# 场景：在你工作时，另一个开发者合并了他们的 PR

# 更新你的 main
git switch main
git fetch upstream
git merge upstream/main

# 变基你的功能分支
git switch feature/add-model-metrics
git rebase main

# 如果出现冲突，解决它们
# git rebase --continue

# 强制推送（因为历史已更改）
# git push --force-with-lease origin feature/add-model-metrics
```

---

### 任务 5.2：拆分大型 PR

将大功能分解为较小的 PR。

**说明：**

```bash
# 不是一个大型 PR，而是创建一系列：

# PR 1：基础
git switch -c feature/metrics-foundation
# Add base metrics class
git commit -m "feat: add base metrics infrastructure"

# PR 2：准确率跟踪（依赖于 PR 1）
git switch -c feature/metrics-accuracy
# Add accuracy calculations
git commit -m "feat: add accuracy tracking to metrics"

# PR 3：仪表板集成（依赖于 PR 2）
git switch -c feature/metrics-dashboard
# Add dashboard
git commit -m "feat: integrate metrics with dashboard"

# 每个 PR 更小、更容易审查，并且可以独立合并
```

---

## 验证

### 验证你的工作流

```bash
# 检查所有配置的 remote
git remote -v

# 查看所有分支
git branch -a

# 检查待处理的变更
git status

# 查看近期与 PR 相关的 commit
git log --oneline -10
```

---

## 挑战任务

### 挑战 1：设置分支保护

配置分支保护规则（在 GitHub 上）：
- 要求 PR 审查
- 要求状态检查通过
- 强制线性历史
- 要求签名提交

### 挑战 2：自动化 PR 检查

创建用于 PR 验证的 GitHub Actions 工作流：

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run tests
        run: pytest tests/
      - name: Lint
        run: flake8 src/
      - name: Type check
        run: mypy src/
```

---

## 故障排除

**问题**："PR 有冲突"
```bash
git fetch upstream
git rebase upstream/main
# Resolve conflicts
git rebase --continue
git push --force-with-lease
```

**问题**："意外推送到 main"
```bash
# 在 GitHub 上还原
# 或联系维护者移除 commit
```

---

## 反思问题

1. **为什么 fork 而不是直接分支？**
2. **什么构成了好的 PR 描述？**
3. **如何提供建设性的代码审查反馈？**
4. **何时应该将 PR 拆分为更小的 PR？**
5. **如何处理代码审查中的分歧？**

---

## 总结

恭喜！你已经掌握了 Git 协作：

- ✅ Fork 和 clone 仓库
- ✅ 创建专业的 pull request
- ✅ 进行有效的代码审查
- ✅ 回应审查反馈
- ✅ 协作最佳实践
- ✅ 管理并发开发
- ✅ PR 模板和指南

### 学会的关键命令

```bash
git remote add upstream <url>  # 添加上游 remote
git fetch upstream             # 获取上游变更
git rebase upstream/main       # 变基到上游
gh pr create                   # 创建 PR（GitHub CLI）
gh pr checkout <number>        # 在本地检出 PR
gh pr review                   # 审查 PR
git push --force-with-lease    # 安全强制推送
```

### 下一步

- 练习 06：ML 特定 Git 工作流
- 在团队项目中练习协作

**出色的工作！** 你已经为专业团队协作做好了准备。
