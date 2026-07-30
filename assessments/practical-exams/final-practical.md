# 期末实践考试

## 概述

**目的**：全面评估初级 AI 基础设施工程师的所有能力

**时长**：4 小时

**先决条件**：完成所有模块（1-4）和所有项目（1-5）

**及格分数**：80/100 分

**形式**：构建一个完整的端到端 ML 基础设施系统

**环境**：已安装 Docker、Kubernetes（kind/minikube）和 GitHub 账号的本地开发机器

---

## 考试说明

### 开始之前

1. **环境搭建**：
   ```bash
   mkdir final-exam
   cd final-exam
   git init
   git remote add origin [your-github-repo-url]
   ```

2. **所需工具**：
   - Python 3.9+
   - Docker 和 Docker Compose
   - Kubernetes（kind、minikube 或带 K8s 的 Docker Desktop）
   - 已配置 kubectl
   - GitHub 账号
   - Git

3. **时间分配**：
   - 搭建和规划：30 分钟
   - 核心实现：150 分钟
   - 测试和调试：60 分钟
   - 文档：30 分钟
   - 最终检查：30 分钟

4. **提交要求**：
   - 包含全部代码的 GitHub 仓库
   - 全面的 README
   - 架构文档
   - 可运行的部署
   - CI/CD 流水线

### 考试规则

- 这是**开卷**考试
- 可以使用官方文档
- 不得抄袭其他学生
- 不得使用 AI 生成完整的解决方案
- 所有工作必须是你的原创实现
- Git 提交应展示你的开发过程

---

## 项目要求

### 场景

**公司**：TechStartup AI
**职位**：初级 AI 基础设施工程师
**任务**：构建生产就绪的 ML 模型服务平台

你的团队需要一个平台来部署和监控多个 ML 模型。你的任务是创建一个概念验证系统，展示：
- 模型服务能力
- 监控和可观测性
- 数据持久化
- 通过 Kubernetes 实现可伸缩性
- 通过 CI/CD 实现自动部署

---

## 第 1 部分：核心应用程序（35 分）

### 要求

构建一个多模型服务平台，具备以下能力：

#### 1.1 模型服务应用程序（15 分）

**功能**：
- 至少提供 2 个不同的 ML 模型（如情感分析、文本摘要）
- 具有适当版本控制的 RESTful API（v1）
- 请求/响应验证
- 长时间运行预测的异步处理
- 通过 API 进行模型切换/选择

**所需端点**：

```python
# 健康检查和状态
GET /health
GET /api/v1/models  # 列出可用模型

# 预测
POST /api/v1/predict
{
  "model": "sentiment-analysis",
  "text": "input text here",
  "version": "1.0.0"
}

# 响应：
{
  "prediction_id": "uuid",
  "model": "sentiment-analysis",
  "result": {...},
  "confidence": 0.95,
  "processing_time_ms": 234
}

# 历史记录
GET /api/v1/predictions/{prediction_id}
GET /api/v1/predictions?model=sentiment-analysis&limit=10
```

**评估标准**：
- [ ] 所有端点功能正常（6 分）
- [ ] 使用 HTTP 状态码进行适当的错误处理（3 分）
- [ ] 输入验证和清理（2 分）
- [ ] 清晰、可维护的代码结构（4 分）

#### 1.2 数据库层（10 分）

**要求**：
- 用于存储预测结果的 PostgreSQL 数据库
- Schema 包含：predictions、models、model_versions 表
- 数据库迁移（Alembic 或类似的工具）
- 连接池
- 对频繁查询的列建立适当索引

**Schema 设计**：

```sql
-- 模型注册表
CREATE TABLE models (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 模型版本
CREATE TABLE model_versions (
    id SERIAL PRIMARY KEY,
    model_id INTEGER REFERENCES models(id),
    version VARCHAR(20) NOT NULL,
    artifact_path VARCHAR(255),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(model_id, version)
);

-- 预测日志
CREATE TABLE predictions (
    id UUID PRIMARY KEY,
    model_version_id INTEGER REFERENCES model_versions(id),
    input_data JSONB NOT NULL,
    output_data JSONB NOT NULL,
    confidence FLOAT,
    processing_time_ms INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 添加适当的索引
```

**评估标准**：
- [ ] Schema 设计合理且已规范化（4 分）
- [ ] 迁移正常运行（2 分）
- [ ] 带索引的高效查询（2 分）
- [ ] 数据库初始化自动化（2 分）

#### 1.3 监控和可观测性（10 分）

**要求**：
- 结构化日志记录，使用适当的日志级别
- 暴露 Prometheus metrics
- 追踪自定义业务 metrics
- Grafana 仪表板（导出为 JSON）

**所需 Metrics**：
- 按端点和模型的请求计数
- 请求延迟（histogram）
- 错误率
- 模型预测置信度分布
- 活跃模型数量
- 数据库连接池状态

**评估标准**：
- [ ] 全面的日志记录实现（3 分）
- [ ] Prometheus metrics 正确暴露（3 分）
- [ ] 有意义的自定义 metrics（2 分）
- [ ] Grafana 仪表板功能完整且有用（2 分）

---

## 第 2 部分：容器化和编排（30 分）

### 2.1 Docker 实现（15 分）

**要求**：
- 用于优化的多阶段构建 Dockerfile
- 用于本地开发的 Docker Compose
- 所有服务正确联网
- 配置了健康检查
- 基于环境的配置

**Docker Compose 服务**：
- `api`：你的模型服务应用程序
- `postgres`：带持久化的数据库
- `prometheus`：metrics 收集
- `grafana`：可视化（带预配置的仪表板）

**评估标准**：
- [ ] 优化的 Dockerfile（<1GB 镜像）（4 分）
- [ ] Docker Compose 正确配置（5 分）
- [ ] 所有服务启动并通信（4 分）
- [ ] 正确使用数据卷和网络（2 分）

### 2.2 Kubernetes 部署（15 分）

**要求**：
- 所有组件的 Kubernetes manifests
- 用于配置的 ConfigMaps
- 用于敏感数据的 Secrets
- Services 和 Ingress/NodePort
- 水平 Pod 自动伸缩器 (HPA)
- 定义了资源限制和请求

**所需 K8s 资源**：
```
k8s/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml（仅模板，不是实际 secrets）
├── deployment-api.yaml
├── deployment-postgres.yaml
├── service-api.yaml
├── service-postgres.yaml
├── hpa.yaml
└── README.md
```

**评估标准**：
- [ ] 所有 manifests 有效且成功应用（5 分）
- [ ] 应用程序在集群中可访问（4 分）
- [ ] ConfigMaps 和 Secrets 使用正确（3 分）
- [ ] HPA 配置且功能正常（3 分）

---

## 第 3 部分：CI/CD 流水线（20 分）

### 要求

**GitHub Actions 工作流**：
- 在推送到 main 和创建 pull request 时触发
- 运行 lint 和代码质量检查
- 运行单元测试和集成测试
- 构建 Docker 镜像并推送到 registry
- 部署到 staging 环境（可模拟）

**流水线阶段**：

```yaml
# .github/workflows/ci-cd.yml
stages:
  - lint
  - test
  - build
  - deploy
```

**所需检查**：
- Python lint 的 Black/Flake8
- 单元测试的 Pytest
- 覆盖率报告（最低 60%）
- Docker 镜像安全扫描（可选，加分项）
- Kubernetes manifest 验证

**评估标准**：
- [ ] 流水线正确触发（3 分）
- [ ] 所有测试通过（5 分）
- [ ] Docker 构建和推送正常（5 分）
- [ ] 代码质量检查已实现（4 分）
- [ ] 部署自动化（3 分）

---

## 第 4 部分：文档和最佳实践（15 分）

### 4.1 文档（10 分）

**所需文档**：

1. **README.md**（4 分）：
   - 项目概述和目的
   - 架构图
   - 先决条件
   - 快速入门指南
   - API 文档
   - 故障排除部分

2. **ARCHITECTURE.md**（3 分）：
   - 系统架构概述
   - 组件描述
   - 数据流图
   - 技术栈选型理由
   - 伸缩性考量

3. **DEPLOYMENT.md**（3 分）：
   - 本地开发环境搭建
   - Docker Compose 部署
   - Kubernetes 部署
   - 环境变量
   - 配置选项

**评估标准**：
- [ ] README 全面且清晰（4 分）
- [ ] 架构文档完善（3 分）
- [ ] 部署指南详细（3 分）

### 4.2 代码质量和最佳实践（5 分）

**要求**：
- 符合 PEP 8 规范的代码
- 全程使用类型提示
- 函数/类的文档字符串
- 无硬编码凭据
- 正确的 .gitignore
- 通过 .env 管理环境变量
- 全程错误处理
- 遵循安全最佳实践

**评估标准**：
- [ ] 代码风格和格式（2 分）
- [ ] 文档和注释（1 分）
- [ ] 安全实践（1 分）
- [ ] 配置管理（1 分）

---

## 加分功能（最多 +15 分）

实现以下任何功能可获得额外加分：

- [ ] **Redis 缓存层**（+3 分）
- [ ] **API 限流**（+2 分）
- [ ] **JWT 认证**（+3 分）
- [ ] **使用 Pydantic 进行 API 请求验证**（+2 分）
- [ ] **长时间运行任务的异步端点**（+3 分）
- [ ] **模型 A/B 测试端点**（+4 分）
- [ ] **K8s 部署的 Helm chart**（+5 分）
- [ ] **负载测试结果和报告**（+3 分）
- [ ] **CI/CD 中的安全扫描**（+2 分）

---

## 项目结构

你的最终项目应遵循以下结构：

```
final-exam/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── src/
│   ├── __init__.py
│   ├── main.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── base.py
│   │   └── sentiment.py
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes.py
│   │   └── schemas.py
│   ├── database/
│   │   ├── __init__.py
│   │   ├── models.py
│   │   └── connection.py
│   └── utils/
│       ├── __init__.py
│       ├── logging.py
│       └── metrics.py
├── tests/
│   ├── __init__.py
│   ├── test_api.py
│   ├── test_models.py
│   └── test_database.py
├── migrations/
│   └── (Alembic 迁移)
├── k8s/
│   └── (Kubernetes manifests)
├── monitoring/
│   ├── prometheus.yml
│   └── grafana-dashboard.json
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── docs/
│   ├── ARCHITECTURE.md
│   └── DEPLOYMENT.md
├── scripts/
│   ├── setup.sh
│   └── deploy.sh
├── .env.example
├── .gitignore
├── requirements.txt
├── README.md
└── pyproject.toml（用于 Black 等）
```

---

## 测试要求

### 单元测试

为以下内容创建测试：
- 模型加载和预测
- API 端点响应
- 数据库操作
- 输入验证

### 集成测试

测试：
- 端到端 API 流程
- 数据库持久化
- 多服务交互

### 最低覆盖率

- 整体：60%
- 关键路径：80%

### 运行测试

```bash
# 安装测试依赖
pip install pytest pytest-cov pytest-asyncio

# 运行测试并生成覆盖率
pytest --cov=src --cov-report=html

# 查看覆盖率报告
open htmlcov/index.html
```

---

## 部署验证

### Docker Compose 验证

```bash
# 构建并启动所有服务
docker-compose up -d

# 检查所有服务健康状态
docker-compose ps

# 测试 API
curl http://localhost:8000/health
curl -X POST http://localhost:8000/api/v1/predict \
  -H "Content-Type: application/json" \
  -d '{"model": "sentiment-analysis", "text": "This is great!", "version": "1.0.0"}'

# 检查 Prometheus
curl http://localhost:9090

# 检查 Grafana
open http://localhost:3000
```

### Kubernetes 验证

```bash
# 应用所有 manifests
kubectl apply -f k8s/

# 检查部署状态
kubectl get deployments
kubectl get pods
kubectl get services

# 端口转发以访问
kubectl port-forward svc/api-service 8000:8000

# 测试 API
curl http://localhost:8000/health

# 查看日志
kubectl logs -l app=api
```

---

## 提交检查清单

提交前请验证：

### 功能性
- [ ] 所有 API 端点正常工作
- [ ] Docker Compose 启动所有服务
- [ ] Kubernetes 部署成功
- [ ] CI/CD 流水线通过
- [ ] 测试达到最低覆盖率

### 代码质量
- [ ] 代码遵循 PEP 8
- [ ] 包含类型提示
- [ ] 无硬编码的 secrets
- [ ] 错误处理全面
- [ ] 日志记录已实现

### 文档
- [ ] README 完整
- [ ] 架构已记录
- [ ] 包含部署指南
- [ ] API 端点已记录
- [ ] 包含故障排除部分

### 仓库
- [ ] 有意义的提交信息
- [ ] .gitignore 正确配置
- [ ] 提供了 .env.example
- [ ] 未提交敏感数据

---

## 提交说明

1. **提交最终代码**：
   ```bash
   git add .
   git commit -m "Final exam submission"
   git push origin main
   ```

2. **创建发布标签**：
   ```bash
   git tag -a v1.0.0 -m "Final exam submission"
   git push origin v1.0.0
   ```

3. **创建 SUBMISSION.md**：
   ```markdown
   # 期末考试提交

   **学生**：[你的姓名]
   **日期**：[日期]
   **GitHub 仓库**：[URL]
   **提交 SHA**：[完整 SHA]

   ## 时间分配
   - 规划：X 小时
   - 实现：X 小时
   - 测试：X 小时
   - 文档：X 小时
   - 总计：X 小时

   ## 自我评估
   第 1 部分（核心应用）：__ / 35
   第 2 部分（容器化）：__ / 30
   第 3 部分（CI/CD）：__ / 20
   第 4 部分（文档）：__ / 15
   加分：__ / 15

   总计：__ / 100

   ## 遇到的挑战
   1. [挑战及解决方法]
   2. [挑战及解决方法]

   ## 我学到了什么
   [对考试的反思]

   ## 补充说明
   [对评审者的任何重要信息]
   ```

4. **通过学习平台**提交你的 GitHub 仓库 URL

---

## 评分标准

| 组件 | 分数 | 权重 | 你的得分 |
|------|------|------|----------|
| **第 1 部分：核心应用** | | | |
| - 模型服务 | 15 | 15% | |
| - 数据库层 | 10 | 10% | |
| - 监控 | 10 | 10% | |
| **第 2 部分：容器和 K8s** | | | |
| - Docker | 15 | 15% | |
| - Kubernetes | 15 | 15% | |
| **第 3 部分：CI/CD** | 20 | 20% | |
| **第 4 部分：文档** | 15 | 15% | |
| **小计** | **100** | **100%** | |
| **加分** | +15 | +15% | |
| **总计** | **115** | **115%** | |

### 通过/未通过标准

- **90-100+**：卓越 - 超出所有预期
- **80-89**：优秀 - 完全胜任，可以进入下一阶段
- **70-79**：良好 - 胜任，存在小差距
- **60-69**：合格 - 基本胜任，建议复习
- **<60**：不足 - 存在重大差距，需要重考

**最低通过分数**：80/100

---

## 常见问题与建议

### 性能建议
- 为数据库使用连接池
- 在适当的地方实现缓存
- 优化 Docker 镜像大小
- 对繁重任务使用异步处理

### 调试建议
- 查看日志：`docker-compose logs -f [service]`
- 边构建边增量测试
- 使用 `kubectl describe` 排查 K8s 问题
- 验证环境变量已设置

### 时间管理
- 从核心功能开始
- 先让 Docker 正常工作
- 然后再处理 Kubernetes
- CI/CD 和加分功能留到最后
- 为文档预留时间

### 安全提醒
- 永远不要提交 secrets
- 使用环境变量
- 以非 root 用户运行容器
- 验证所有输入
- 生产环境使用 HTTPS（仅文档说明）

---

## 考试之后

### 如果你通过了（80%+）
恭喜！你已经展示了作为初级 AI 基础设施工程师的能力。

**后续步骤**：
1. 查看反馈
2. 将此项目添加到你的作品集
3. 继续学习中级内容
4. 分享你的工作（GitHub、LinkedIn）

### 如果需要额外努力（<80%）
不要灰心！把这当作学习机会。

**行动计划**：
1. 查看详细反馈
2. 找出具体的薄弱环节
3. 重新学习相关模块
4. 练习特定技能
5. 2 周后重考

---

## 资源

### 文档
- [Flask](https://flask.palletsprojects.com/)
- [FastAPI](https://fastapi.tiangolo.com/)
- [Docker](https://docs.docker.com/)
- [Kubernetes](https://kubernetes.io/docs/)
- [Prometheus](https://prometheus.io/docs/)
- [GitHub Actions](https://docs.github.com/en/actions)

### 工具
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [kubectl 速查表](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [kind](https://kind.sigs.k8s.io/) 或 [minikube](https://minikube.sigs.k8s.io/)

---

## 考试期间的问题

如果遇到问题：
1. **技术问题**：在 SUBMISSION.md 中记录
2. **需求不明确**：做出合理假设并记录
3. **时间限制**：优先完成核心需求而非加分功能

---

## 学术诚信

本次考试测试你的个人能力：
- 可以自由使用官方文档
- 不得抄袭其他学生
- 不得使用 AI 生成完整的解决方案
- 通过 Git 提交展示你的工作
- 记录任何使用的外部代码片段

**违反学术诚信将导致不及格。**

---

## 最终检查清单

提交前，请确保：

- [ ] 应用程序在 Docker Compose 下成功运行
- [ ] 应用程序部署到 Kubernetes
- [ ] 所有测试通过
- [ ] CI/CD 流水线成功
- [ ] README 全面
- [ ] 仓库中没有 secrets
- [ ] Git 历史展示了你的开发过程
- [ ] SUBMISSION.md 已完成
- [ ] 仓库已公开或与讲师共享

---

**祝你好运！** 这场考试代表了你所学的全部内容。不要着急，有条不紊，展示你的技能。你可以的！

---

## 评估时间线

- **提交截止日期**：[按分配时间]
- **初审**：5 个工作日内
- **反馈提供**：7 个工作日内
- **重考时间**：收到反馈后 2 周

---

**记住**：这是对实践技能的综合测试。首先让功能正常工作，然后再优化和完善。质量优于数量，但完整性对通过至关重要。
