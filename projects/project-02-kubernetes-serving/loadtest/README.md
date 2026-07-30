# 负载测试 — Project 02

本目录包含 Model API 的负载测试计划。每个测试的目标是发现部署的特定特征，而不仅仅是"发送大量请求"。

## 测试计划

| 计划 | 用户数 | 时长 | 目标 |
|---|---|---|---|
| **冒烟测试** | 5 | 1 分钟 | 确认部署响应且基本端点正常工作 |
| **爬坡测试** | 3 分钟内从 0 到 200，保持 5 分钟 | 8 分钟 | 找到舒适的稳态 RPS 并识别延迟何时恶化 |
| **浸泡测试** | 100 | 60 分钟 | 捕捉内存泄漏、GC 暂停、缓慢的日志累积 |
| **尖峰测试** | 30 秒内从 50 到 500，保持 2 分钟 | 3 分钟 | 验证 HPA 响应且无级联故障 |

## 运行

### 冒烟测试

```bash
locust -f loadtest/locustfile.py \
  --host https://model-api.example.com \
  --headless --users 5 --spawn-rate 5 --run-time 1m \
  --csv loadtest/results/smoke
```

### 爬坡测试

```bash
locust -f loadtest/locustfile.py \
  --host https://model-api.example.com \
  --headless --users 200 --spawn-rate 1 --run-time 8m \
  --csv loadtest/results/ramp
```

### 浸泡测试

```bash
locust -f loadtest/locustfile.py \
  --host https://model-api.example.com \
  --headless --users 100 --spawn-rate 10 --run-time 60m \
  --csv loadtest/results/soak
```

### 尖峰测试

```bash
locust -f loadtest/locustfile.py \
  --host https://model-api.example.com \
  --headless --users 500 --spawn-rate 15 --run-time 3m \
  --csv loadtest/results/spike
```

## 验收标准

本项目通过负载测试应显示：

- **冒烟测试：** 100% 成功率，p95 < 500ms。
- **爬坡测试：** 持续 100 RPS，p95 < 800ms，无 5xx 错误。
- **浸泡测试：** 60 分钟内内存增长 < 10%，错误率无漂移。
- **尖峰测试：** HPA 在 60 秒内扩容副本，前 30 秒后无请求丢失，尖峰期间 p99 < 2s。

## 记录结果

每次运行将 CSV 汇总写入 `loadtest/results/<运行名称>_*.csv`：

- `*_stats.csv` — 每个端点的请求数、失败数、延迟百分位数
- `*_failures.csv` — 错误分类
- `*_stats_history.csv` — 用于绘图的时间序列

使用附带的 notebook（`loadtest/analyze.ipynb`，可选）或任何绘图工具绘制时间序列。将截图附到你的项目提交中。

## 可配置的环境变量

| 变量 | 默认值 | 描述 |
|---|---|---|
| `FEATURE_COUNT` | `10` | 每个 `/predict` 请求的特征数量 |
| `MODEL_VERSIONS` | `latest` | 逗号分隔的模型版本，用于轮询 |

## 关于测试环境的说明

始终在与生产环境相同配置的专用测试环境中运行负载测试（相同的实例类型、相同的 HPA 限制、相同的模型）。在共享环境中进行测试会产生误导，并可能影响其他团队。
