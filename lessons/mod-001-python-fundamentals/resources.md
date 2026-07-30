# Module 001 — Python 基础资源

## 官方文档

- **Python 文档** — [docs.python.org](https://docs.python.org/3/)。权威参考。请收藏教程和标准库索引。
- **PEP 8 — Python 代码风格指南** — [peps.python.org/pep-0008](https://peps.python.org/pep-0008/)。通读一遍；经常回顾。
- **PEP 20 — Python 之禅** — [peps.python.org/pep-0020](https://peps.python.org/pep-0020/)。简短，值得定期重读。

## 书籍

### 入门 / 中级

- **《流畅的 Python（第 2 版）》** — Luciano Ramalho 著。标准的 Python 惯用语法参考。涵盖了将"Java 工程师写的 Python"与"Python 工程师写的 Python"区分开来的语言部分。
- **《Effective Python（第 2 版）》** — Brett Slatkin 著。90 条具体的最佳实践。是《流畅的 Python》的绝佳伴侣。
- **《Python 编程从入门到实践》** — Eric Matthes 著。如果你是编程新手，这是一本友好的入门书。

### 专门针对 ML 工程师

- **《利用 Python 进行数据分析（第 3 版）》** — Wes McKinney（pandas 创建者）著。NumPy/pandas 数据层的参考资料。
- **《设计机器学习系统》** — Chip Huyen 著。不是 Python 特定的，但它是权威的 ML 系统文本——有助于理解本模块技能为何重要的框架。

## 在线课程

- **Real Python** — [realpython.com](https://realpython.com/)。涵盖所有级别的高质量教程。值得订阅。
- **MIT 6.0001 / 6.0002 计算机科学导论与 Python** — MIT OpenCourseWare 免费提供。现有最全面的免费入门课程。
- **CS50P（哈佛大学 Python 编程导论）** — [cs50.harvard.edu/python](https://cs50.harvard.edu/python/)。免费，节奏适当。

## 现代 Python 工具

- **uv** — [astral.sh/uv](https://docs.astral.sh/uv/)。快速（10-100x）替代 pip/pip-tools/virtualenv。新项目请使用此工具。
- **ruff** — [docs.astral.sh/ruff](https://docs.astral.sh/ruff/)。快速的 linter + formatter，可直接替代 black + flake8 + isort。
- **mypy** — [mypy.readthedocs.io](https://mypy.readthedocs.io/)。静态类型检查器。添加到你的 CI 中。
- **pytest** — [docs.pytest.org](https://docs.pytest.org/)。事实上的 Python 测试框架。

## ML 特定 Python 参考

- **NumPy 文档** — [numpy.org/doc](https://numpy.org/doc/stable/)。每个 ML 库的基础。
- **pandas 文档** — [pandas.pydata.org/docs](https://pandas.pydata.org/docs/)。DataFrame + Series，表格 ML 的工作界面。
- **PyTorch 教程** — [pytorch.org/tutorials](https://pytorch.org/tutorials/)。在后续模块中使用。

## 应避免的反模式

- 在组合更适合时使用继承。
- 在异步代码中使用 `requests`（同步）。
- 在函数就能解决问题时为所有东西编写类。
- 捕获裸的 `except:` 子句（应捕获特定异常）。
- `from X import *`（失去可追溯性）。

## 社区 + 练习

- **/r/learnpython** — [reddit.com/r/learnpython](https://www.reddit.com/r/learnpython/)。欢迎初学者提问。
- **Stack Overflow** — `[python]` 标签。精选答案。
- **PyCon** — 年度会议；许多演讲免费发布在 YouTube 上。
- **Python Bytes 播客** — [pythonbytes.fm](https://pythonbytes.fm/)。每周新闻 + 库。

## 本课程中的交叉引用

- Module 003（Git）— 对你的 Python 代码进行版本控制。
- Module 004（ML 基础）— 将 Python 应用于 ML 工作负载。
- 工程师轨道的 mod-101 — 深入的生产级 Python 模式。
