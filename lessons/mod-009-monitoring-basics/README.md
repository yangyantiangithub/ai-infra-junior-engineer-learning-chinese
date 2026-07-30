# Module 009: 监控与可观测性基础

## 模块概述

监控与可观测性是基础设施工程师的关键技能，使您能够理解系统行为、及早发现问题并维护可靠性。本模块提供监控基础、Prometheus 指标收集、Grafana 可视化、日志策略和基本告警的全面培训。

您将学习可观测性的三大支柱（metrics、logs、traces），如何对应用进行 instrument、构建 dashboard、设置告警以及建立监控最佳实践。这些技能对于维护生产级 AI/ML 基础设施和确保系统可靠性至关重要。

完成本模块后，您将能自信地设置监控系统、收集 metrics、构建 dashboard、分析日志以及为基础设施和应用程序建立告警工作流。

## 学习目标

完成本模块后，您将能够：

1. **理解可观测性基础**，包括 metrics、logs 和 traces
2. **使用 Prometheus 收集 metrics**，并理解 Prometheus 数据模型
3. **使用 Grafana 构建 dashboard**，用于可视化系统和应用 metrics
4. **实现结构化日志记录**，使用适当的日志级别和格式
5. **高效查询日志**，使用 grep、awk 和日志聚合工具
6. **设置告警**，使用 Prometheus Alertmanager
7. **监控应用程序**，包括对 Python 代码进行 instrument
8. **监控基础设施**，包括服务器、容器和服务
9. **应用 SLI、SLO 和 SLA**，用于定义可靠性目标
10. **监控 ML 系统**，包括模型性能和数据漂移

## 前置知识

- 需要完成 Module 002（Linux 基础）
- 建议完成 Module 005（Docker 和容器）
- 建议完成 Module 007（API 和 Web 服务）
- 基本网络知识
- 熟悉命令行界面

**推荐设置：**
- 已安装 Docker 和 Docker Compose
- Python 3.9+ 及虚拟环境
- 8GB+ 内存用于运行 monitoring 栈
- 20GB+ 可用磁盘空间
- 用于 dashboard 的 Web 浏览器
- 文本编辑器或 IDE

## 时间投入

- **预计总时间：** 35-45 小时
- **讲座与阅读：** 12-15 小时
- **动手练习：** 18-22 小时
- **项目：** 5-8 小时

**推荐进度：**
- 兼职（5-10 小时/周）：4-5 周
- 全职（20-30 小时/周）：2 周

掌握监控需要动手经验。预计需要花费时间设置系统、分析 metrics 和排查问题。

## 模块结构

### 第 1 周：监控基础
- **主题：** 可观测性概念、监控基础、metrics 类型
- **关键技能：** 理解可观测性、metric 类型、监控策略
- **实践：** 分析现有监控系统

### 第 2 周：Prometheus 和 Metrics
- **主题：** Prometheus 架构、PromQL、exporters、instrumentation
- **关键技能：** 收集 metrics、编写查询、对应用进行 instrument
- **实践：** 设置 Prometheus、收集 metrics

### 第 3 周：Grafana 和可视化
- **主题：** Grafana dashboard、panels、告警、最佳实践
- **关键技能：** 构建 dashboard、可视化技术、dashboard 设计
- **实践：** 创建全面的 dashboard

### 第 4 周：日志、告警和 ML 监控
- **主题：** 结构化日志、日志聚合、告警、ML metrics
- **关键技能：** 日志管理、告警配置、ML 系统监控
- **实践：** 完整 monitoring 栈实现

## 详细主题分解

### 1. 可观测性基础（5-6 小时）

#### 1.1 可观测性简介
- 什么是可观测性？
- 监控 vs 可观测性
- 可观测性的重要性
- 三大支柱：metrics、logs、traces
- 分布式系统中的可观测性
- 可观测性成熟度模型
- 成本效益分析

#### 1.2 Metrics
- 什么是 metrics？
- Metric 类型（counters、gauges、histograms、summaries）
- 时间序列数据
- 基数考量
- 选择测量指标
- 四大黄金信号（latency、traffic、errors、saturation）
- RED 方法（Rate、Errors、Duration）
- USE 方法（Utilization、Saturation、Errors）

#### 1.3 日志
- 什么是日志？
- 日志级别（DEBUG、INFO、WARN、ERROR、FATAL）
- 结构化 vs 非结构化日志
- 日志聚合
- 日志保留策略
- 日志最佳实践
- 安全与合规考量

#### 1.4 分布式 Tracing
- 什么是 tracing？
- Spans 和 traces
- Tracing 分布式系统
- OpenTelemetry 简介
- Trace 采样
- Trace 分析
- 何时使用 tracing

#### 1.5 监控策略
- 监控什么
- 设置基线
- 定义正常行为
- 识别关键 metrics
- 监控层次（基础设施、应用、业务）
- 开发环境 vs 生产环境的监控
- 团队职责

### 2. Prometheus（8-10 小时）

#### 2.1 Prometheus 基础
- 什么是 Prometheus？
- Prometheus 架构
- Pull vs push 模型
- 时间序列数据库
- 数据模型（metrics、labels）
- Prometheus 组件（server、exporters、alertmanager）
- Prometheus 生态系统

#### 2.2 安装和配置 Prometheus
- 安装方法
- 配置文件（prometheus.yml）
- Scrape 配置
- Service discovery
- Relabeling
- Recording rules
- Federation 基础
- 存储配置

#### 2.3 Metrics 收集
- Exporters（node_exporter、blackbox_exporter）
- 应用 instrumentation
- Prometheus 客户端库
- 自定义 exporters
- Pushgateway（何时使用）
- Service discovery 机制
- Scraping 间隔和配置

#### 2.4 PromQL（Prometheus 查询语言）
- PromQL 基础
- 选择 metrics
- Label 匹配
- Range vectors vs instant vectors
- 运算符（算术、比较、逻辑）
- 聚合运算符（sum、avg、max、min、count）
- 函数（rate、irate、increase、histogram_quantile）
- 子查询
- 查询性能考量

#### 2.5 Recording Rules
- 什么是 recording rules？
- 创建 recording rules
- 预聚合查询
- 性能优化
- 规则评估
- 最佳实践
- 命名约定

#### 2.6 面向应用的 Prometheus
- 对 Python 应用进行 instrument
- Counter metrics
- Gauge metrics
- Histogram metrics
- Summary metrics
- 自定义 metrics
- Labeling 策略
- 性能影响

### 3. Grafana（7-9 小时）

#### 3.1 Grafana 基础
- 什么是 Grafana？
- Grafana 架构
- 安装和设置
- 数据源
- 组织和用户
- Dashboards 和 panels
- Grafana 生态系统

#### 3.2 创建 Dashboard
- Dashboard 基础
- 添加 panels
- Panel 类型（graph、stat、gauge、table、heatmap）
- 时间范围控制
- 变量和模板化
- 注释
- Dashboard 组织
- Dashboard JSON 模型

#### 3.3 可视化类型
- 时间序列图
- 条形图和直方图
- Single stat 和 gauge
- 表格
- Heatmaps
- 饼图和环形图
- Logs panel
- 选择适当的可视化

#### 3.4 查询构建
- Prometheus 查询构建器
- 在 Grafana 中编写 PromQL
- 查询变量
- 每个 panel 多个查询
- 查询转换
- 覆盖和阈值
- 查询性能

#### 3.5 Dashboard 设计最佳实践
- Dashboard 布局原则
- 信息层次
- 颜色使用
- 阈值和告警可视化
- 面向不同受众的 Dashboard
- 移动端友好的 Dashboard
- Dashboard 模板
- 可复用 panels

#### 3.6 变量和模板化
- Dashboard 变量
- 查询变量
- 常量变量
- 自定义变量
- 变量链式引用
- 多值变量
- 动态 Dashboard
- 变量最佳实践

#### 3.7 Grafana 告警
- Grafana 中的告警规则
- 告警条件
- 通知渠道
- 告警路由
- Silences 和静音
- 告警测试
- 告警最佳实践
- 从旧版告警迁移

### 4. 日志和日志管理（6-8 小时）

#### 4.1 日志基础
- 日志的目的
- 记录什么
- 不记录什么（secrets、PII）
- 日志级别及使用时机
- 日志频率考量
- 性能影响
- 日志安全

#### 4.2 结构化日志
- JSON 日志记录
- 键值对
- 日志上下文
- 关联 ID
- Python 中的结构化日志
- 日志库（logging、structlog）
- 日志 schema 设计
- 解析结构化日志

#### 4.3 应用日志
- Python logging 模块
- Logger 配置
- Log handlers
- Log formatters
- API 日志记录
- 错误和异常记录
- 请求/响应日志
- 日志最佳实践

#### 4.4 日志聚合
- 为什么要聚合日志？
- 集中式日志架构
- ELK Stack 简介（Elasticsearch、Logstash、Kibana）
- Loki（Grafana Loki）用于日志
- 日志传输（Fluentd、Filebeat）
- 日志保留和轮转
- 日志存储考量

#### 4.5 日志分析
- 搜索日志（grep、awk、jq）
- 日志模式和异常
- 错误调查
- 从日志进行性能分析
- 与 metrics 的关联
- 基于日志的 metrics
- 常见日志分析场景

#### 4.6 容器日志
- 容器日志策略
- Docker 日志
- 日志驱动
- stdout/stderr 日志记录
- 日志记录 sidecar 模式
- Kubernetes 日志记录
- 从容器收集日志

### 5. 告警（5-7 小时）

#### 5.1 告警基础
- 为什么要告警？
- 告警疲劳
- 可操作的告警
- 告警严重级别
- On-call 考量
- 升级策略
- 告警文档

#### 5.2 Prometheus Alertmanager
- Alertmanager 架构
- 安装 Alertmanager
- Prometheus 中的告警规则
- Alertmanager 配置
- 路由树
- Receivers 和集成
- 分组和限流
- Silences 和抑制

#### 5.3 编写告警规则
- 告警规则语法
- 告警条件
- 基于阈值的告警
- 变化率告警
- 缺失告警
- 告警 labels 和 annotations
- 告警严重级别
- 测试告警规则

#### 5.4 告警设计
- 基于 SLO 的告警
- 症状 vs 原因告警
- 告警上下文和 runbooks
- 告警命名约定
- 多窗口告警
- 减少误报
- 告警调优
- 告警审查流程

#### 5.5 通知渠道
- 邮件通知
- Slack 集成
- PagerDuty 集成
- Webhook 通知
- 多渠道
- 升级工作流
- 通知格式化
- 测试通知

#### 5.6 事件响应
- On-call 流程
- 事件分类
- 使用告警进行诊断
- 事后审查
- 告警改进
- 事件文档
- 从事件中学习

### 6. 基础设施监控（5-6 小时）

#### 6.1 服务器监控
- CPU metrics
- 内存 metrics
- 磁盘 metrics
- 网络 metrics
- Node Exporter
- 系统健康检查
- 容量规划
- 性能基线

#### 6.2 容器监控
- Docker metrics
- cAdvisor 用于容器
- 容器资源使用
- 容器健康检查
- Docker 事件
- 容器性能
- 多容器监控

#### 6.3 服务监控
- 服务健康检查
- 端点监控
- Blackbox 监控
- 响应时间 metrics
- 错误率
- 服务依赖
- 合成监控

#### 6.4 数据库监控
- 数据库 metrics
- 连接池监控
- 查询性能
- 慢查询检测
- 数据库 exporters
- 复制延迟
- 数据库容量

#### 6.5 网络监控
- 网络流量 metrics
- 带宽利用率
- 丢包
- 延迟监控
- DNS 监控
- SSL 证书监控
- 网络拓扑感知

### 7. 应用性能监控（4-5 小时）

#### 7.1 APM 基础
- 什么是 APM？
- APM vs 基础设施监控
- 应用 instrumentation
- 事务 tracing
- 依赖映射
- 用户体验监控
- 真实用户监控 vs 合成监控

#### 7.2 Python 应用监控
- 对 Python 应用进行 instrument
- 请求/响应 metrics
- 数据库查询跟踪
- 外部 API 监控
- 后台任务监控
- 内存分析
- 性能分析

#### 7.3 API 监控
- API 端点 metrics
- 响应时间分布
- 错误率跟踪
- 请求率监控
- 状态码分布
- 端点特定 dashboard
- API SLO 监控

#### 7.4 业务 Metrics
- 应用级 metrics
- 用户活动 metrics
- 功能使用情况
- 业务 KPI
- 转化漏斗
- 自定义业务 metrics
- 指标驱动开发

### 8. 监控 ML 系统（4-5 小时）

#### 8.1 ML 特定监控
- ML 监控的不同之处
- 模型性能 metrics
- 预测延迟
- 预测吞吐量
- Metrics 中的模型版本管理
- A/B 测试 metrics
- Champion/challenger 监控

#### 8.2 数据质量监控
- 输入数据验证
- 特征分布
- 缺失值检测
- 数据 schema 变更
- 数据新鲜度
- 数据量监控
- 异常值检测

#### 8.3 模型性能监控
- 在线 vs 离线 metrics
- 生产环境中的准确率、精确率、召回率
- 模型漂移检测
- Concept drift
- Feature drift
- 性能退化告警
- 重新训练触发条件

#### 8.4 ML 管道监控
- 训练管道 metrics
- 数据处理 metrics
- 训练时长
- 训练期间资源利用率
- 模型部署 metrics
- 管道故障
- 端到端管道可观测性

### 9. SLI、SLO 和 SLA（3-4 小时）

#### 9.1 服务等级指标（SLI）
- 什么是 SLI？
- 选择 SLI
- 基于请求的 SLI
- 基于窗口的 SLI
- 测量 SLI
- SLI 实现
- SLI 最佳实践

#### 9.2 服务等级目标（SLO）
- 什么是 SLO？
- 设定切合实际的 SLO
- Error budget 概念
- SLO 计算
- SLO 监控
- 基于 SLO 的告警
- SLO 迭代

#### 9.3 服务等级协议（SLA）
- SLA vs SLO
- SLA 组成部分
- 协商 SLA
- SLA 监控
- SLA 违规
- 内部 vs 外部 SLA
- 法律考量

#### 9.4 Error Budgets
- Error budget 概念
- 计算 error budget
- Error budget 策略
- 使用 error budgets
- Error budget 耗尽
- 平衡速度与可靠性
- Error budget 文化

## 讲座大纲

### 讲座 1：可观测性基础（90 分钟）
- 可观测性简介
- Metrics、logs、traces
- 监控策略
- 四大黄金信号
- 可观测性文化
- **实验：** 分析现有系统

### 讲座 2：Prometheus 基础（120 分钟）
- Prometheus 架构
- 安装和配置
- Metrics 收集
- Exporters
- 基础 PromQL
- **实验：** 设置 Prometheus

### 讲座 3：PromQL 和 Recording Rules（90 分钟）
- PromQL 深入
- 查询构建
- 聚合和函数
- Recording rules
- 查询优化
- **实验：** 编写复杂查询

### 讲座 4：应用 Instrumentation（90 分钟）
- 对 Python 应用进行 instrument
- 自定义 metrics
- Labeling 策略
- 最佳实践
- 性能考量
- **实验：** 对 API 进行 instrument

### 讲座 5：Grafana 和 Dashboards（120 分钟）
- Grafana 基础
- 构建 dashboard
- 可视化类型
- 变量和模板化
- Dashboard 设计
- **实验：** 创建全面 dashboard

### 讲座 6：日志和分析（90 分钟）
- 结构化日志
- 日志聚合
- 日志分析技术
- 容器日志
- 最佳实践
- **实验：** 实现结构化日志

### 讲座 7：告警和事件响应（90 分钟）
- 告警基础
- Prometheus Alertmanager
- 编写告警规则
- 通知渠道
- 事件响应
- **实验：** 设置告警

### 讲座 8：ML 系统监控（90 分钟）
- ML 特定挑战
- 模型性能监控
- 数据漂移检测
- ML 管道可观测性
- ML 的 SLI/SLO
- **实验：** 监控 ML 模型服务

## 动手练习

### 练习分类

#### 监控基础（5 个练习）
1. 识别系统关键 metrics
2. 分析监控策略
3. Metric 类型选择
4. 监控层设计
5. 建立基线

#### Prometheus（10 个练习）
6. Prometheus 安装和配置
7. 设置 exporters
8. 编写 PromQL 查询
9. 对 Python 应用进行 instrument
10. 自定义 metrics 实现
11. 创建 recording rules
12. Service discovery 配置
13. Prometheus 性能调优
14. 多目标 scraping
15. Federation 设置

#### Grafana（8 个练习）
16. 创建第一个 dashboard
17. Panel 类型探索
18. Dashboard 变量
19. 告警配置
20. API 监控 Dashboard
21. 基础设施 Dashboard
22. 创建模板 Dashboard
23. 高级可视化技术

#### 日志（5 个练习）
24. 结构化日志实现
25. 日志聚合设置
26. 日志分析场景
27. 容器日志
28. 基于日志的 metrics

#### 告警（4 个练习）
29. 创建告警规则
30. Alertmanager 配置
31. 告警路由设置
32. 事件响应模拟

#### ML 监控（3 个练习）
33. 模型性能 dashboard
34. 数据漂移检测
35. 完整 ML monitoring 栈

## 评估与考核

### 知识检查
- 每个主要部分后的小测验
- PromQL 理解
- Dashboard 设计原则
- 告警设计最佳实践
- ML 监控概念

### 实践评估
- **Prometheus 设置：** 配置完整 Prometheus 栈
- **Dashboard 创建：** 构建全面监控 dashboard
- **应用 Instrumentation：** 为 Python 应用添加 metrics
- **告警配置：** 设置可操作的告警
- **ML 监控：** 实现模型监控系统

### 能力标准
要成功完成本模块，您应能够：
- 设置 Prometheus 和 Grafana
- 编写 PromQL 查询
- 使用 metrics 对应用进行 instrument
- 构建有效的 dashboard
- 实现结构化日志
- 配置告警系统
- 监控基础设施和应用程序
- 应用 SLI 和 SLO
- 监控 ML 系统
- 使用监控数据响应事件

### 毕业项目
**完整 Monitoring 栈：**
为 ML 应用部署全面监控：
- Prometheus metrics 收集
- 应用 instrumentation
- 基础设施监控
- Grafana dashboard（基础设施、应用、ML 模型）
- 结构化日志
- 带有 Alertmanager 的告警规则
- SLI/SLO 定义
- 文档和 runbooks

## 资源和参考

> **注意：** 参见 `resources/recommended-reading.md` 获取全面的学习材料、书籍和在线资源列表。

### 核心资源
- Prometheus 文档
- Grafana 文档
- SRE 手册（Google）
- 可观测性工程书籍

### 推荐书籍
- "Site Reliability Engineering" by Google
- "The Art of Monitoring" by James Turnbull
- "Practical Monitoring" by Mike Julian
- "Observability Engineering" by Charity Majors et al.

### 在线学习
- Prometheus 教程
- Grafana 教程
- PromLabs 培训
- Monitoring Weekly 通讯

### 工具
- Prometheus
- Grafana
- Alertmanager
- cAdvisor
- Node Exporter
- Promtool

## 开始上手

### 步骤 1：设置 Monitoring 栈
1. 安装 Docker 和 Docker Compose
2. 使用 Docker 部署 Prometheus
3. 部署 Grafana
4. 验证连接
5. 探索示例 metrics

### 步骤 2：探索现有 Metrics
1. 浏览 Prometheus UI
2. 尝试 PromQL
3. 创建简单 Grafana dashboard
4. 理解 metric 结构

### 步骤 3：从讲座 1 开始
- 学习可观测性基础
- 理解监控策略
- 完成初始练习

### 步骤 4：逐步构建
- 对示例应用进行 instrument
- 创建 dashboard
- 设置告警
- 监控完整系统

## 成功技巧

1. **从简单开始：** 从基础 metrics 开始，逐步增加复杂度
2. **练习 PromQL：** 查询语言熟练度来自实践
3. **设计 Dashboard：** 关注可用性和清晰度
4. **明智地告警：** 使用可操作的告警避免告警疲劳
5. **监控一切：** 从第一天起就对应用进行 instrument
6. **使用 Labels：** 利用 labels 实现灵活查询
7. **记录 Dashboard：** 添加描述和注释
8. **测试告警：** 验证告警正确触发
9. **从事件中学习：** 使用监控数据进行根本原因分析
10. **迭代：** 持续改进监控策略

## 下一步

完成本模块后，您将准备好：
- **Module 010：** 云平台（云原生监控）
- 高级主题：分布式 tracing、高级 APM、混沌工程
- 将监控应用于所有未来项目

## 开发状态

**当前状态：** 模板阶段——完整结构已就位

**现已提供：**
- 完整的模块结构
- 详细的主题分解
- 讲座大纲
- 练习框架

**开发中：**
- 完整的讲座笔记
- PromQL 练习
- Dashboard 模板
- 告警规则示例
- ML 监控模式
- 完整 monitoring 栈示例

**计划更新：**
- 深入分布式 tracing
- 高级 PromQL 模式
- 多集群监控
- 成本优化
- 可观测性平台比较

## 反馈与贡献

帮助改进本模块：
- 报告问题
- 分享 dashboard 设计
- 贡献 PromQL 查询
- 建议监控模式

---

**模块维护者：** AI 基础设施课程团队
**联系方式：** ai-infra-curriculum@joshua-ferguson.com
**最后更新：** 2025-10-18
**版本：** 1.0.0-template
