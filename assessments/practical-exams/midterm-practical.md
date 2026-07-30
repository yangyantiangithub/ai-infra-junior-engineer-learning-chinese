# 期中实践考试

## 概述

**目的**：评估 Docker 容器化、基础模型部署和问题排查的动手能力

**时长**：3 小时

**先决条件**：完成模块 1-2 和项目 1-2

**及格分数**：75/100 分

**形式**：动手编码和部署任务

**环境**：已安装 Docker 的本地开发机器

---

## 考试说明

### 开始之前

1. **设置工作空间**：
   ```bash
   mkdir midterm-exam
   cd midterm-exam
   git init
   ```

2. **所需工具**：
   - Python 3.9+
   - Docker 和 Docker Compose
   - 文本编辑器/IDE
   - Git

3. **时间管理**：
   - 任务 1：45 分钟
   - 任务 2：60 分钟
   - 任务 3：45 分钟
   - 复习/测试：30 分钟

4. **提交要求**：
   - 所有代码在 Git 仓库中
   - README.md 包含搭建说明
   - Docker 容器正常运行
   - 记录遇到的任何问题

### 通用规则

- 可以使用官方文档
- 不得与他人协作
- 不得使用预构建的解决方案
- 所有代码必须是你自己的原创
- 经常保存你的工作

---

## 任务 1：容器化机器学习模型（30 分）

### 场景

你得到了一个 Python 脚本，用于加载预训练的情感分析模型。你的任务是容器化此应用程序并通过 REST API 暴露它。

### 提供的代码

**model_server.py**（你需要根据需求创建）：

```python
# TODO: 导入必要的库 (Flask/FastAPI, transformers 等)

# TODO: 加载预训练的情感分析模型
# 建议：使用 transformers 库中的小型模型
# "distilbert-base-uncased-finetuned-sst-2-english"

# TODO: 创建 API 端点 POST /predict
# 输入：{"text": "some text to analyze"}
# 输出：{"sentiment": "POSITIVE/NEGATIVE", "score": 0.95}

# TODO: 创建健康检查端点 GET /health

# TODO: 在端口 8000 上运行服务器
```

### 要求

1. **应用程序**（12 分）：
   - [ ] 实现模型服务器，完成所有 TODO
   - [ ] POST /predict 端点接收文本并返回情感分析结果
   - [ ] GET /health 端点返回服务状态
   - [ ] 对无效输入进行适当的错误处理
   - [ ] 模型在启动时成功加载

2. **Dockerfile**（10 分）：
   - [ ] 使用合适的基础镜像（Python 3.9+）
   - [ ] 安装所有依赖
   - [ ] 复制应用程序代码
   - [ ] 暴露正确的端口（8000）
   - [ ] 以非 root 用户运行
   - [ ] 镜像大小 < 2GB

3. **文档**（5 分）：
   - [ ] README 包含构建说明
   - [ ] 如何运行容器
   - [ ] 示例 API 请求
   - [ ] 已知限制

4. **测试**（3 分）：
   - [ ] 成功构建，无错误
   - [ ] 容器运行且保持健康状态
   - [ ] API 正确响应请求

### 交付物

```
task1/
├── model_server.py
├── requirements.txt
├── Dockerfile
├── .dockerignore
└── README.md
```

### 评估标准

| 标准 | 分数 | 你的得分 |
|------|------|----------|
| API 端点正常工作 | 6 | |
| 错误处理 | 3 | |
| 模型正确加载 | 3 | |
| Dockerfile 最佳实践 | 6 | |
| 镜像优化 | 4 | |
| 文档质量 | 5 | |
| 成功运行 | 3 | |
| **总计** | **30** | |

---

## 任务 2：带监控的多容器应用程序（40 分）

### 场景

扩展你的任务 1 应用程序，加入 PostgreSQL 数据库用于记录预测结果，以及 Prometheus 用于监控。使用 Docker Compose 编排所有服务。

### 要求

1. **增强版应用程序**（15 分）：
   - [ ] 修改 model_server.py 将预测记录到 PostgreSQL
   - [ ] 包含 predictions 表（id, text, sentiment, score, timestamp）的数据库 schema
   - [ ] 新增端点 GET /predictions 用于检索历史记录
   - [ ] 在 /metrics 暴露 Prometheus metrics
   - [ ] 追踪：请求计数、请求时长、预测分布

2. **Docker Compose 配置**（15 分）：
   - [ ] 服务：model-server（你的应用程序）
   - [ ] 服务：postgres（带数据卷用于持久化）
   - [ ] 服务：prometheus（带配置文件）
   - [ ] 服务之间的正确网络连接
   - [ ] 用于配置的环境变量
   - [ ] 所有服务的健康检查

3. **数据库集成**（5 分）：
   - [ ] 成功连接到 PostgreSQL
   - [ ] 预测结果成功保存
   - [ ] 检索端点正常工作
   - [ ] 对数据库问题的适当错误处理

4. **监控搭建**（5 分）：
   - [ ] Prometheus 抓取 metrics
   - [ ] 自定义 metrics 在 Prometheus UI 中可见
   - [ ] Prometheus 可通过 localhost:9090 访问

### 交付物

```
task2/
├── model_server.py（已更新）
├── requirements.txt（已更新）
├── Dockerfile（已更新）
├── docker-compose.yml
├── prometheus.yml
├── init-db.sql（可选）
├── .env.example
└── README.md
```

### 数据库 Schema

创建具有以下结构的表：

```sql
CREATE TABLE predictions (
    id SERIAL PRIMARY KEY,
    text TEXT NOT NULL,
    sentiment VARCHAR(20) NOT NULL,
    confidence FLOAT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Prometheus 配置模板

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'model-server'
    static_configs:
      - targets: ['model-server:8000']
```

### 示例 API 交互

**进行预测**：
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"text": "This is amazing!"}'
```

**获取预测历史**：
```bash
curl http://localhost:8000/predictions?limit=10
```

**检查 metrics**：
```bash
curl http://localhost:8000/metrics
```

### 评估标准

| 标准 | 分数 | 你的得分 |
|------|------|----------|
| 数据库集成正常 | 8 | |
| 预测结果正确存储 | 4 | |
| 历史端点功能正常 | 3 | |
| Docker Compose 配置 | 10 | |
| 服务网络连接 | 5 | |
| Prometheus metrics 已暴露 | 3 | |
| Metrics 有意义 | 2 | |
| 所有服务成功启动 | 5 | |
| **总计** | **40** | |

---

## 任务 3：问题排查与文档（30 分）

### 场景

你接手了一个损坏的 Docker 应用程序。你的任务是识别和修复问题，然后记录问题和解决方案。

### 提供的文件

你将获得以下文件（故意破坏的版本）：

**broken_app.py**：
```python
from flask import Flask, request, jsonify
import os

app = Flask(__name__)

# TODO: 修复问题 - 缺少数据库连接
DATABASE_URL = os.getenv("DATABASE_URL")

@app.route('/api/data', methods=['GET'])
def get_data():
    # TODO: 修复问题 - 不当的错误处理
    data = fetch_from_database()
    return jsonify(data)

@app.route('/api/data', methods=['POST'])
def post_data():
    # TODO: 修复问题 - 没有输入验证
    content = request.json
    save_to_database(content)
    return jsonify({"status": "success"})

# TODO: 修复问题 - 函数未实现
def fetch_from_database():
    pass

def save_to_database(data):
    pass

if __name__ == '__main__':
    # TODO: 修复问题 - 以 root 运行，绑定到错误端口
    app.run(host='0.0.0.0', port=5000)
```

**Broken Dockerfile**：
```dockerfile
# TODO: 修复问题 - 使用过时的基础镜像
FROM python:2.7

# TODO: 修复问题 - 未设置工作目录
COPY . .

# TODO: 修复问题 - 安装不必要的包
RUN pip install flask requests numpy pandas scikit-learn tensorflow

# TODO: 修复问题 - 暴露错误端口
EXPOSE 8080

# TODO: 修复问题 - 以 root 运行
CMD ["python", "broken_app.py"]
```

**Broken docker-compose.yml**：
```yaml
version: '3.8'

services:
  app:
    build: .
    ports:
      # TODO: 修复问题 - 端口映射不匹配
      - "5000:8080"
    environment:
      # TODO: 修复问题 - 暴露敏感数据
      - DATABASE_URL=postgresql://admin:password123@db:5432/mydb
    # TODO: 修复问题 - 缺少依赖关系

  # TODO: 修复问题 - 数据库服务未定义
```

### 你的任务

1. **识别问题**（10 分）：
   - [ ] 列出代码中发现的所有问题
   - [ ] 按严重程度分类（严重、高、中、低）
   - [ ] 解释每个问题为何存在风险

2. **修复所有问题**（15 分）：
   - [ ] 修正 Python 应用程序
   - [ ] 修复 Dockerfile
   - [ ] 修复 docker-compose.yml
   - [ ] 添加适当的错误处理
   - [ ] 实现缺失的函数（可以是存根）
   - [ ] 添加输入验证

3. **文档**（5 分）：
   - [ ] 创建 ISSUES.md 列出所有问题
   - [ ] 创建 FIXES.md 解释所有修正
   - [ ] 更新 README.md 包含正确的搭建说明

### 预期发现并修复的问题

你应至少识别这些问题：

**代码问题**：
- 缺少数据库连接实现
- 端点中没有错误处理
- 没有输入验证
- 缺少函数实现
- 安全问题（暴露的凭据）

**Dockerfile 问题**：
- Python 版本过时
- 未设置工作目录
- 安装过多包
- 暴露错误端口
- 以 root 用户运行
- 缺少 .dockerignore

**Docker Compose 问题**：
- 端口映射不匹配
- 硬编码凭据（应使用 secrets/env 文件）
- 缺少数据库服务
- 没有健康检查
- 缺少 depends_on

### 交付物

```
task3/
├── broken_app.py（已修复）
├── requirements.txt
├── Dockerfile（已修复）
├── docker-compose.yml（已修复）
├── .dockerignore
├── .env.example
├── ISSUES.md
├── FIXES.md
└── README.md
```

### ISSUES.md 模板

```markdown
# 发现的问题

## 严重问题
1. **问题**：描述
   - **位置**：文件和行号
   - **影响**：为何严重
   - **修复**：如何解决

## 高优先级问题
...

## 中优先级问题
...

## 低优先级问题
...
```

### FIXES.md 模板

```markdown
# 已应用的修复

## 问题 1：[标题]
- **原始代码**：```[有问题的代码]```
- **修复后代码**：```[修正后的代码]```
- **解释**：为什么这个修复有效
- **测试**：如何验证修复
```

### 评估标准

| 标准 | 分数 | 你的得分 |
|------|------|----------|
| 识别了严重问题 | 5 | |
| 识别了所有问题 | 5 | |
| 修复了应用程序代码 | 5 | |
| 修复了 Dockerfile | 5 | |
| 修复了 docker-compose.yml | 5 | |
| 文档质量 | 3 | |
| 测试验证 | 2 | |
| **总计** | **30** | |

---

## 提交指南

### 提交内容

1. **Git 仓库**，包含全部三个任务：
   ```
   midterm-exam/
   ├── task1/
   ├── task2/
   ├── task3/
   └── EXAM_SUMMARY.md
   ```

2. **EXAM_SUMMARY.md** 包含：
   - 每个任务花费的时间
   - 遇到的挑战
   - 如何解决问题
   - 自我评估分数
   - 参考的资源

### 如何提交

1. **创建最终提交**：
   ```bash
   git add .
   git commit -m "Midterm exam submission"
   ```

2. **验证所有功能正常**：
   - [ ] 所有 Docker 镜像成功构建
   - [ ] 所有容器运行无错误
   - [ ] 所有 API 端点正确响应
   - [ ] 文档完整

3. **导出你的工作**：
   ```bash
   git archive --format=zip HEAD > midterm-exam-submission.zip
   ```

4. **通过学习平台**或按照指示提交

---

## 评分标准汇总

| 部分 | 分数 | 你的得分 |
|------|------|----------|
| 任务 1：容器化模型 | 30 | |
| 任务 2：多容器应用 | 40 | |
| 任务 3：问题排查 | 30 | |
| **总计** | **100** | |

### 分数解读

- **90-100**：卓越 - 高级理解
- **75-89**：精通 - 可以继续
- **60-74**：合格 - 建议复习
- **<60**：需要改进 - 复习后重考

---

## 时间管理建议

### 第一个小时
- 完整阅读所有需求
- 设置工作空间
- 从任务 1 开始（最直接）

### 第二个小时
- 完成任务 1
- 开始任务 2
- 边做边测试

### 第三个小时
- 完成任务 2
- 攻克任务 3（问题排查）
- 记录发现

### 最后 30 分钟
- 检查所有交付物
- 测试一切正常运行
- 编写 EXAM_SUMMARY.md
- 提交并准备最终提交

---

## 允许使用的资源

### 允许的
- Python 官方文档
- Docker 文档
- Flask/FastAPI 文档
- PostgreSQL 文档
- Prometheus 文档
- Stack Overflow（用于语法参考）
- 你自己在已完成模块中的笔记

### 不允许的
- 复制完整解决方案
- 使用 AI 完成整个实现
- 与其他学生协作
- 使用预构建的项目模板

---

## 需避免的常见陷阱

1. **没有仔细阅读需求** - 确保理解题目要求
2. **在一个任务上花费太多时间** - 卡住了就先跳过，稍后回来
3. **没有增量测试** - 每构建一个组件就测试一个
4. **时间管理不当** - 注意时间，合理分配
5. **文档不完整** - 不要跳过 README 文件
6. **没有处理错误** - 始终包含 try-catch 块
7. **硬编码值** - 使用环境变量
8. **以 root 运行** - 始终在 Docker 中创建非 root 用户

---

## 考试之后

### 如果你通过了（75%+）
1. 复习薄弱环节
2. 继续到模块 3
3. 保存你的解决方案作为参考
4. 在学习小组中帮助他人

### 如果你未通过（<75%）
1. 查看评分反馈
2. 找出具体的知识盲区
3. 重新学习相关课程
4. 练习薄弱环节
5. 1 周后重考

---

## 学习建议

参加本考试之前，确保你能：
- [ ] 从零编写 Dockerfile
- [ ] 创建 Flask/FastAPI 应用程序
- [ ] 使用 Docker Compose 管理多个服务
- [ ] 从 Python 连接到 PostgreSQL
- [ ] 暴露 Prometheus metrics
- [ ] 调试容器问题
- [ ] 阅读和编写技术文档

---

## 问题或疑问

如果考试期间遇到技术问题：
1. 详细记录问题
2. 将其包含在 EXAM_SUMMARY.md 中
3. 展示你的排查步骤
4. 尝试变通方案

技术问题如果得到妥善记录，不会影响你的分数。

---

**祝你好运！** 记住：这场考试测试的是你作为初级 AI 基础设施工程师日常使用的实践技能。不要着急，充分测试，做好文档。
