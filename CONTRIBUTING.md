# 贡献指南 — AI Infrastructure Junior Engineer 学习资料库

感谢你对 AI Infrastructure 课程的兴趣！本仓库旨在帮助有志成为 AI 基础设施工程师的学习者掌握入门所需的技能。你的贡献将帮助这个学习资源变得更好，惠及所有人。

## 目录

- [行为准则](#行为准则)
- [如何贡献？](#如何贡献)
- [快速开始](#快速开始)
- [贡献指南](#贡献指南)
- [拉取请求流程](#拉取请求流程)
- [风格指南](#风格指南)
- [社区](#社区)

## 行为准则

本项目及所有参与者均受我们的[行为准则](CODE_OF_CONDUCT.md)约束。参与即表示你同意遵守该准则。请将不可接受的行为报告至 ai-infra-curriculum@joshua-ferguson.com。

## 如何贡献？

### 报告 Bug 或问题

如果你在代码、文档或学习材料中发现错误：

1. **检查现有 Issues**，看是否已被报告
2. 如果没有，**创建一个新的 Issue**，使用我们的 Bug 报告模板
3. 包含：
   - 清晰、描述性的标题
   - 复现步骤
   - 预期行为 vs 实际行为
   - 你的环境（操作系统、Python 版本等）
   - 截图或错误信息（如适用）

### 建议改进

有改进学习体验的想法？

1. **检查现有 Issues**，避免重复
2. **创建新的 Issue**，使用功能请求模板
3. 清晰描述：
   - 你发现的问题或差距
   - 你建议的解决方案
   - 为什么这对学习者有益
   - 你考虑过的任何替代方案

### 改进文档

文档改进始终受欢迎！包括：

- 修正拼写或语法错误
- 澄清令人困惑的说明
- 添加示例或图表
- 改进代码注释
- 扩展 README 或指南

对于小修正（拼写、语法），可以直接提交 PR。对于较大的更改，请先开启 Issue 进行讨论。

### 贡献代码或练习

我们欢迎以下贡献：

- 修复现有代码框架中的 Bug
- 改进练习的清晰度
- 添加更多测试用例
- 更好的代码示例
- 新的资源或参考资料

**重要**：这是一个**学习仓库**。代码框架应保持不完整，并带有清晰的 TODO 标记。请勿向本仓库提交完整的解决方案（这些应放在解答仓库中）。

### 改进学习材料

通过以下方式帮助改进课程：

- 澄清难懂的概念
- 添加真实世界的示例
- 创建图表或可视化
- 建议更多资源
- 改进练习说明
- 添加练习题目

## 快速开始

### 先决条件

- Python 3.11 或更高版本
- Git
- GitHub 账号
- 对 AI/ML 概念有基本了解（或愿意学习！）

### 配置开发环境

1. **Fork 仓库**到你的 GitHub

2. **克隆你的 Fork**到本地：
   ```bash
   git clone https://github.com/你的用户名/ai-infra-junior-engineer-learning.git
   cd ai-infra-junior-engineer-learning
   ```

3. **添加上游仓库**：
   ```bash
   git remote add upstream https://github.com/ai-infra-curriculum/ai-infra-junior-engineer-learning.git
   ```

4. **创建虚拟环境**：
   ```bash
   python -m venv venv
   source venv/bin/activate  # Windows: venv\Scripts\activate
   ```

5. **安装依赖**：
   ```bash
   pip install -r requirements.txt
   pip install -e .  # 以开发模式安装
   ```

6. **安装预提交钩子**（可选但推荐）：
   ```bash
   pre-commit install
   ```

### 运行测试

在提交更改前，确保所有测试通过：

```bash
# 运行所有测试
pytest

# 带覆盖率运行
pytest --cov

# 运行特定测试文件
pytest tests/test_specific.py
```

### 代码质量检查

我们使用多种工具来维护代码质量：

```bash
# 使用 black 格式化代码
black .

# 排序导入
isort .

# 使用 flake8 进行代码检查
flake8 .

# 使用 mypy 进行类型检查
mypy .

# 或使用预提交一次性运行所有检查
pre-commit run --all-files
```

## 贡献指南

### 一般原则

1. **保持教育性**：记住这是一个学习资源。代码应清晰且注释完善。

2. **维持难度级别**：这是初级工程师仓库。保持复杂度适合初学者。

3. **遵循现有模式**：与现有内容和结构保持一致。

4. **测试你的更改**：确保代码可用且测试通过。

5. **记录一切**：清晰的说明有助于学习者理解。

### 什么样的贡献是好的？

好的贡献是：

- **清晰且专注**：每个 PR 一个逻辑变更
- **经过充分测试**：根据需要包含或更新测试
- **文档完善**：说明更改了什么以及为什么
- **教学价值**：帮助学习者理解概念
- **易于理解**：使用清晰的语言并解释行话
- **完整**：包含所有必要的文件和更新

### 我们不想要的

请避免：

- 练习的完整解答（请使用解答仓库）
- 超出初级工程师范围的进阶主题
- 未经事先讨论的大规模重构
- 破坏现有功能的更改
- 文档或测试不完善的代码
- 抄袭内容

## 拉取请求流程

### 提交之前

1. **为你的更改创建新分支**：
   ```bash
   git checkout -b feature/你的功能名称
   # 或
   git checkout -b fix/你的修复名称
   ```

2. **保持你的 Fork 更新**：
   ```bash
   git fetch upstream
   git rebase upstream/main
   ```

3. **按照指南进行更改**

4. **运行测试和质量检查**：
   ```bash
   pytest
   black .
   flake8 .
   mypy .
   ```

5. **提交更改**，使用清晰的信息：
   ```bash
   git add .
   git commit -m "feat: add exercise for model deployment"
   ```

   我们遵循[约定式提交](https://www.conventionalcommits.org/)：
   - `feat:` 新功能或练习
   - `fix:` Bug 修复
   - `docs:` 文档更改
   - `test:` 添加或更新测试
   - `refactor:` 代码重构
   - `style:` 格式化更改
   - `chore:` 维护任务

6. **推送到你的 Fork**：
   ```bash
   git push origin feature/你的功能名称
   ```

### 提交 PR

1. 前往 GitHub 上的原始仓库
2. 点击 "New Pull Request"
3. 选择你的 Fork 和分支
4. 完整填写 PR 模板
5. 关联相关的 Issues
6. 提交 PR

### PR 审查流程

1. **自动化检查**将运行（测试、代码检查等）
2. **维护者将审查**你的代码（通常在 3-5 天内）
3. **处理反馈**：通过向你的分支推送新提交来解决
4. 一旦获批，维护者将**合并你的 PR**

### PR 合并后

1. **删除你的分支**：
   ```bash
   git branch -d feature/你的功能名称
   git push origin --delete feature/你的功能名称
   ```

2. **更新你的 Fork**：
   ```bash
   git checkout main
   git pull upstream main
   git push origin main
   ```

3. **庆祝！** 你为开源教育做出了贡献！

## 风格指南

### Python 代码风格

- 遵循 [PEP 8](https://pep8.org/)
- 使用 [Black](https://black.readthedocs.io/) 格式化（行长度：88）
- 使用 [isort](https://pycqa.github.io/isort/) 排序导入
- 使用 Python 3.11+ 语法添加类型提示
- 使用 [Google 风格](https://google.github.io/styleguide/pyguide.html#38-comments-and-docstrings)编写文档字符串

示例：

```python
from typing import List, Optional


def process_model_data(
    data: List[float],
    normalize: bool = True,
    scaling_factor: Optional[float] = None,
) -> List[float]:
    """处理模型输入数据，可选归一化。

    Args:
        data: 待处理的输入数据点
        normalize: 是否对数据进行归一化
        scaling_factor: 可选的比例因子

    Returns:
        已处理的数据，可用于模型推理

    Raises:
        ValueError: 如果数据为空或 scaling_factor 为负数
    """
    # TODO: 实现数据处理逻辑
    pass
```

### 文档风格

- 使用清晰、简单的语言
- 首次使用技术术语时给出定义
- 包含代码示例
- 在有助于理解时使用图表
- 保持段落简短且易于浏览
- 使用标题组织内容

### Markdown 风格

- 使用 ATX 风格标题（`#` 而非下划线）
- 标题前后加一个空行
- 使用带语言说明的围栏代码块
- 行长度保持在合理范围（80-100 个字符）
- 内部引用使用相对链接

### 提交信息风格

遵循[约定式提交](https://www.conventionalcommits.org/)：

```
type(scope): 简要描述

更长的说明（如需要）。在 72 个字符处换行。

- 可以使用项目符号
- 使用现在时："add" 而非 "added"
- 引用 Issues："fixes #123"

BREAKING CHANGE: 如涉及破坏性更改，在此描述
```

类型：
- `feat`：新功能或练习
- `fix`：Bug 修复
- `docs`：文档更改
- `test`：测试更改
- `refactor`：代码重构
- `style`：格式化更改
- `chore`：维护

### 练习和课程指南

创建或修改练习时：

1. **明确的学习目标**：说明学习者将达成什么
2. **先修要求**：列出所需的知识或已完成的课程
3. **分步指导**：将复杂任务分解为可管理的步骤
4. **预期结果**：描述成功的结果是什么
5. **常见陷阱**：提醒典型的错误
6. **资源**：链接到相关文档或教程
7. **预计时间**：给出合理的时间预期

示例结构：

```markdown
## 练习：部署你的第一个模型

**学习目标**：使用 FastAPI 部署训练好的 scikit-learn 模型

**先修要求**：
- 已完成课程 03：模型训练
- 基本了解 REST API
- 已安装 Python 3.11+

**预计时间**：45 分钟

### 操作说明

1. **设置 FastAPI 应用**
   ```python
   # TODO: 导入所需库
   # TODO: 创建 FastAPI 应用实例
   ```

2. **加载训练好的模型**
   - 模型文件位于 `models/classifier.pkl`
   - 使用 joblib 加载模型

   [继续详细的步骤...]

### 预期结果

你应该有一个运行中的 API，能够：
- 接收带有特征数据的 POST 请求
- 以 JSON 格式返回预测结果
- 优雅地处理错误

### 常见陷阱

- **模型加载错误**：确保 pickle 兼容性
- **端口冲突**：检查端口 8000 是否已被占用
- **数据验证**：使用 Pydantic 进行输入验证

### 资源

- [FastAPI 文档](https://fastapi.tiangolo.com/)
- [模型序列化指南](link)
```

## 社区

### 获取帮助

- **Issues**：浏览现有 Issues 或创建新的
- **Discussions**：使用 GitHub Discussions 提问
- **邮件**：ai-infra-curriculum@joshua-ferguson.com

### 认可

贡献者将：
- 被列入我们的贡献者名单
- 在发布说明中被致谢
- 在社区精选中被提及

### 保持联系

- Watch 仓库获取更新
- Star 仓库以示支持
- 使用 `#AIInfraCurriculum` 在社交媒体上分享你的学习历程

## 有问题？

不要犹豫，随时提问！我们在这里提供帮助：

- 开启 GitHub Discussion
- 在相关 Issue 中评论
- 发送邮件至 ai-infra-curriculum@joshua-ferguson.com

感谢你帮助让 AI 基础设施教育变得更好！

---

**记住**：这是一个学习社区。我们珍视耐心、善意以及帮助他人成功的意愿。每一份贡献，无论大小，都至关重要。
