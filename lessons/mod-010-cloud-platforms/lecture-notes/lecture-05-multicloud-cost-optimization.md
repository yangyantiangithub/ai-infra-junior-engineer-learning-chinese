# Lecture 05：多云策略与成本优化

## 目录
1. [简介](#简介)
2. [多云架构](#多云架构)
3. [云成本基础](#云成本基础)
4. [ML 工作负载的成本优化](#ml-工作负载的成本优化)
5. [AI 基础设施的 FinOps](#ai-基础设施的-finops)
6. [灾难恢复与业务连续性](#灾难恢复与业务连续性)
7. [云迁移策略](#云迁移策略)
8. [总结与关键要点](#总结与关键要点)

## 简介

如果没有适当规划，ML 工作负载的云成本可能失控。单个大型 GPU 实例 24/7 运行每年可能花费 25,000 美元。再乘以训练、推理和实验，成本很容易达到六位数或七位数。

本讲座涵盖优化云支出、设计多云解决方案以及确保 ML 基础设施业务连续性的策略。

### 学习目标

到本讲座结束时，你将：
- 理解多云架构模式及其权衡
- 识别并优化 ML 工作负载中的主要成本驱动因素
- 为 AI 基础设施实施 FinOps 实践
- 设计 ML 系统的灾难恢复策略
- 规划和执行云迁移
- 应用合理调整规格、调度和购买策略

### 前置要求
- 本模块的讲座 01-04（云基础、AWS 服务）
- 理解 ML 工作流（训练、推理、数据存储）
- 云定价模型的基础知识

**时长**：90 分钟
**难度**：中级

---

## 1. 多云架构

### 为什么需要多云？

```
单一云的风险：
┌────────────────────────────────────┐
│ 所有 ML 基础设施都在 AWS 上      │
│                                    │
│ 风险：                             │
│ ✗ 供应商锁定                      │
│ ✗ 单点故障                        │
│ ✗ 价格上涨（无谈判筹码）          │
│ ✗ 受限于一个供应商的功能          │
│ ✗ 区域性故障 = 完全停机           │
└────────────────────────────────────┘

多云的优势：
┌────────────────────────────────────┐
│ ML 基础设施跨 AWS/GCP             │
│                                    │
│ 优势：                            │
│ ✓ 最佳服务组合                    │
│ ✓ 谈判筹码                        │
│ ✓ 灾难恢复                        │
│ ✓ 监管合规（跨区域数据驻留）     │
└────────────────────────────────────┘
```

### 多云策略

#### 1. 最佳服务组合（使用每个云的优势）

```
架构：
┌─────────────────────────────────────┐
│ AWS                                 │
│ ├── S3：数据湖存储                 │
│ ├── EKS：模型服务（Kubernetes）    │
│ └── RDS：元数据数据库              │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ GCP                                 │
│ ├── Vertex AI：模型训练            │
│ ├── BigQuery：数据仓库             │
│ └── TPU：大型模型训练              │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ Azure                               │
│ ├── Azure ML：AutoML 实验          │
│ └── Cognitive Services：预训练模型  │
└─────────────────────────────────────┘
```

**示例理由：**
- AWS：强大的生态系统、成熟的 Kubernetes（EKS）、最佳 S3 定价
- GCP：最适合数据分析（BigQuery）、用于 Transformer 的 TPU
- Azure：企业集成（Active Directory）、混合云

#### 2. 主备灾难恢复

```
主：AWS（活跃）
├── 所有生产流量
├── 100% 推理请求
└── 所有模型训练

备：GCP（被动/温备）
├── 数据持续复制
├── 基础设施就绪（缩放到 0）
└── 仅在 AWS 故障时激活
```

#### 3. 双活高可用

```
两个云都处理生产流量：

AWS（60% 流量）         GCP（40% 流量）
├── 美东用户            ├── 美西用户
├── 欧洲用户            ├── 亚太用户
└── 通过跨云同步        └── 通过跨云同步
    共享数据                共享数据

优点：
✓ 更高的可用性
✓ 更低的延迟（地理分布）
✗ 复杂（数据一致性、同步）
✗ 更高的成本（同时运行两个）
```

### 多云挑战

1. **数据同步**
   ```python
   # 将数据从 AWS S3 复制到 GCS
   from google.cloud import storage as gcs_storage
   import boto3

   s3_client = boto3.client('s3')
   gcs_client = gcs_storage.Client()

   # 列出 AWS S3 对象
   objects = s3_client.list_objects_v2(Bucket='ml-models-aws')

   for obj in objects.get('Contents', []):
       # 从 S3 下载
       s3_obj = s3_client.get_object(
           Bucket='ml-models-aws',
           Key=obj['Key']
       )

       # 上传到 GCS
       bucket = gcs_client.bucket('ml-models-gcp')
       blob = bucket.blob(obj['Key'])
       blob.upload_from_string(s3_obj['Body'].read())

   print(f"从 AWS 同步了 {len(objects)} 个对象到 GCP")
   ```

2. **身份和访问管理（IAM）**
   - AWS IAM ≠ GCP IAM ≠ Azure RBAC
   - 使用集中式身份提供商（Okta、Azure AD）
   - 为 SSO 实施 SAML/OIDC

3. **网络**
   ```
   连接 AWS VPC 到 GCP VPC：
   ┌─────────────┐         ┌─────────────┐
   │ AWS VPC     │         │ GCP VPC     │
   │ 10.0.0.0/16 │◄───────►│ 10.1.0.0/16 │
   └─────────────┘         └─────────────┘
        VPN 或 Direct Connect / Cloud Interconnect
   ```

4. **监控**
   - 使用云无关工具：Prometheus、Grafana、Datadog
   - 聚合日志：ELK 栈、Loki、Splunk

### 多云决策框架

| 使用多云当： | 避免多云当： |
|---------------------|------------------------|
| 关键正常运行时间要求（99.99%+） | 团队规模小（<10 名工程师） |
| 监管需求（数据驻留） | 成本比可用性更重要 |
| 利用每个云的最佳服务 | 复杂性超过收益 |
| 规模足够大可以谈判定价 | 已锁定在一个云中 |

---

## 2. 云成本基础

### 云定价模型

#### 按需实例

**优点**：灵活性、无承诺
**缺点**：最贵（基准）

```
AWS EC2 p3.2xlarge（1x V100 GPU）：
- 按需：$3.06/小时
- 每月成本（24/7）：$2,203/月
- 每年成本：$26,438/年
```

#### 预留实例（1-3 年承诺）

**节省**：相比按需 30-70%

```
AWS EC2 p3.2xlarge：
- 按需：$3.06/小时
- 1 年预留：$1.84/小时（节省 40%）
- 3 年预留：$1.10/小时（节省 64%）

年度成本（3 年预留）：
$1.10 × 24 × 365 = $9,636 vs $26,438 按需
节省：每年 $16,802（64%）
```

**用于**：基线、可预测的工作负载（生产推理）

#### Spot 实例（备用容量，可中断）

**节省**：相比按需 50-90%

```
AWS EC2 p3.2xlarge spot：
- 按需：$3.06/小时
- Spot（平均）：$0.92/小时（节省 70%）

但是：可能被中断，提前 2 分钟通知！
```

**用于**：容错工作负载（带 checkpointing 的训练）

#### Savings Plans（灵活承诺）

**AWS Compute Savings Plans**：承诺 $/小时 持续 1-3 年

```
承诺 $1/小时 持续 3 年：
- AWS 将折扣应用于任何计算使用
- 比预留实例更灵活
- 根据承诺可节省 17-60%
```

---

## 3. ML 工作负载的成本优化

### ML 中的主要成本驱动因素

```
典型 ML 基础设施成本：
┌────────────────────────────────────┐
│ GPU 计算：         60%  $60k/年   │
│ 存储（数据/模型）： 20%  $20k/年   │
│ 网络：             10%  $10k/年   │
│ 托管服务：          7%   $7k/年   │
│ 数据传输：          3%   $3k/年   │
│ 总计：            100% $100k/年   │
└────────────────────────────────────┘
```

### 优化策略 1：合理调整规格

**问题**：过度配置的实例浪费金钱

```python
# 分析实际使用情况
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch')

# 获取过去 30 天的 CPU 利用率
response = cloudwatch.get_metric_statistics(
    Namespace='AWS/EC2',
    MetricName='CPUUtilization',
    Dimensions=[{'Name': 'InstanceId', 'Value': 'i-1234567890'}],
    StartTime=datetime.now() - timedelta(days=30),
    EndTime=datetime.now(),
    Period=3600,  # 1 小时
    Statistics=['Average', 'Maximum']
)

avg_cpu = sum(d['Average'] for d in response['Datapoints']) / len(response['Datapoints'])
print(f"平均 CPU：{avg_cpu:.1f}%")

# 如果 avg_cpu < 20%，实例过度配置！
```

**措施**：降低实例类型

```
当前：ml.p3.8xlarge（4 GPU）@ $12.24/小时
平均 GPU 利用率：30%

调整为：ml.p3.2xlarge（1 GPU）@ $3.06/小时
节省：$9.18/小时 × 720 小时/月 = 每月 $6,610
```

### 优化策略 2：自动扩缩容

**问题**：固定容量在低需求期间浪费金钱

```python
# 自动扩展推理端点
import boto3

autoscaling = boto3.client('application-autoscaling')

# 注册可扩展目标
autoscaling.register_scalable_target(
    ServiceNamespace='sagemaker',
    ResourceId='endpoint/ml-inference-endpoint/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    MinCapacity=1,  # 低流量时缩放到 1 个实例
    MaxCapacity=10  # 高峰时扩展到 10 个
)

# 配置扩展策略（目标 70% CPU）
autoscaling.put_scaling_policy(
    PolicyName='cpu-scaling-policy',
    ServiceNamespace='sagemaker',
    ResourceId='endpoint/ml-inference-endpoint/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    PolicyType='TargetTrackingScaling',
    TargetTrackingScalingPolicyConfiguration={
        'TargetValue': 70.0,  # 目标 70% CPU 利用率
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'SageMakerVariantInvocationsPerInstance'
        },
        'ScaleInCooldown': 300,   # 缩减前等待 5 分钟
        'ScaleOutCooldown': 60    # 扩展前等待 1 分钟
    }
)
```

**节省**：
```
无自动扩缩容：10 个实例 × 24/7 = 每月 7200 个实例小时
有自动扩缩容：平均 4 个实例 = 每月 2880 个实例小时
节省：减少 60% = 每月节省 $13,220
```

### 优化策略 3：定时扩缩容

**问题**：开发/测试环境不必要时 24/7 运行

```python
# 在非工作时间停止实例
import boto3
from datetime import datetime

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    current_hour = datetime.now().hour

    # 工作时间：上午 9 点 - 下午 6 点 PST
    if 9 <= current_hour < 18:
        # 启动开发实例
        ec2.start_instances(InstanceIds=['i-dev-12345'])
    else:
        # 停止开发实例
        ec2.stop_instances(InstanceIds=['i-dev-12345'])
```

**节省**：
```
无调度：24 小时/天 × 30 天 = 每月 720 小时
有调度（上午 9 点 - 下午 6 点）：9 小时/天 × 22 个工作日 = 每月 198 小时
节省：减少 72% = 每个实例每月节省 $1,590
```

### 优化策略 4：为训练使用 Spot 实例

```python
# 使用 Spot 实例训练（AWS SageMaker）
import sagemaker

estimator = sagemaker.estimator.Estimator(
    image_uri='my-training-image',
    role='SageMakerRole',
    instance_count=4,
    instance_type='ml.p3.2xlarge',
    use_spot_instances=True,  # 使用 Spot！
    max_run=3600,  # 最多 1 小时
    max_wait=7200,  # 最多等待 2 小时以获取 Spot 容量
    checkpoint_s3_uri='s3://my-bucket/checkpoints/'  # 保存检查点
)

estimator.fit({'training': 's3://my-bucket/data'})
```

**节省**：
```
训练作业：4 × p3.2xlarge × 8 小时
按需：$3.06/小时 × 4 × 8 = $97.92
Spot：$0.92/小时 × 4 × 8 = $29.44
节省：$68.48（70%）
```

### 优化策略 5：存储分层

**问题**：所有数据都存储在昂贵的热存储中

```python
# 将旧模型移到更便宜的存储
import boto3
from datetime import datetime, timedelta

s3 = boto3.client('s3')

# 列出 S3 中的模型
response = s3.list_objects_v2(Bucket='ml-models')

for obj in response.get('Contents', []):
    last_modified = obj['LastModified'].replace(tzinfo=None)
    age_days = (datetime.now() - last_modified).days

    # 将 90 天前的模型移到 Glacier
    if age_days > 90:
        s3.copy_object(
            Bucket='ml-models',
            Key=obj['Key'],
            CopySource={'Bucket': 'ml-models', 'Key': obj['Key']},
            StorageClass='GLACIER'
        )
        print(f"已将 {obj['Key']} 移动到 Glacier（年龄：{age_days} 天）")
```

**存储成本对比**：
```
S3 Standard：$0.023/GB/月
S3 Infrequent Access：$0.0125/GB/月（节省 46%）
S3 Glacier：$0.004/GB/月（节省 83%）

1TB 旧模型：
Standard：$23.60/月
Glacier：$4.10/月
节省：每月 $19.50（每年 $234）
```

---

## 4. AI 基础设施的 FinOps

### 什么是 FinOps？

**FinOps**（财务运营）将财务问责制引入云支出。

**原则**：
1. **团队协作**（工程、财务、业务）
2. **每个人对自己的使用负责**
3. **中央团队推动 FinOps**
4. **支出可见性**
5. **决策由业务价值驱动**

### 成本分配与标签

```python
# 为成本跟踪给所有资源打标签
import boto3

ec2 = boto3.client('ec2')

# 给实例打标签
ec2.create_tags(
    Resources=['i-1234567890'],
    Tags=[
        {'Key': 'Project', 'Value': 'recommendation-system'},
        {'Key': 'Environment', 'Value': 'production'},
        {'Key': 'Team', 'Value': 'ml-platform'},
        {'Key': 'CostCenter', 'Value': '12345'},
        {'Key': 'Owner', 'Value': 'alice@company.com'}
    ]
)
```

**按标签的成本仪表盘**：
```
按项目划分的成本（最近 30 天）：
├── recommendation-system：$45,234（45%）
├── fraud-detection：$32,890（33%）
├── search-ranking：$15,432（15%）
└── experimentation：$7,123（7%）

按环境划分的成本：
├── production：$75,234（75%）
├── staging：$15,432（15%）
└── dev：$10,023（10%）
```

### 预算与告警

```python
# 设置带告警的预算
import boto3

budgets = boto3.client('budgets')

budgets.create_budget(
    AccountId='123456789012',
    Budget={
        'BudgetName': 'ML-Platform-Monthly',
        'BudgetLimit': {
            'Amount': '100000',  # 每月 $100k
            'Unit': 'USD'
        },
        'TimeUnit': 'MONTHLY',
        'BudgetType': 'COST'
    },
    NotificationsWithSubscribers=[
        {
            'Notification': {
                'NotificationType': 'ACTUAL',
                'ComparisonOperator': 'GREATER_THAN',
                'Threshold': 80,  # 达到预算 80% 时告警
                'ThresholdType': 'PERCENTAGE'
            },
            'Subscribers': [
                {
                    'SubscriptionType': 'EMAIL',
                    'Address': 'ml-team@company.com'
                }
            ]
        },
        {
            'Notification': {
                'NotificationType': 'FORECASTED',
                'ComparisonOperator': 'GREATER_THAN',
                'Threshold': 100,  # 预测达到 100% 时告警
                'ThresholdType': 'PERCENTAGE'
            },
            'Subscribers': [
                {
                    'SubscriptionType': 'EMAIL',
                    'Address': 'finance@company.com'
                }
            ]
        }
    ]
)
```

---

## 5. 灾难恢复与业务连续性

### RPO 和 RTO 目标

```
RPO（恢复点目标）：
你能承受损失多少数据？
├── RPO = 1 小时 → 最多损失 1 小时数据
└── 需要：持续复制或每小时备份

RTO（恢复时间目标）：
系统需在多快时间内恢复在线？
├── RTO = 4 小时 → 4 小时内系统恢复
└── 需要：温备或更快
```

### ML 的 DR 策略

#### 策略 1：备份和恢复（成本最低，RTO 最长）

```
RPO：数小时到数天
RTO：数小时到数天
成本：低（$$$）

实施：
1. 每日模型备份到 S3
2. 数据库备份到 S3
3. Git 中的基础设施即代码
4. 灾难时：从备份重建

最适合：非关键系统
```

#### 策略 2：Pilot Light（低成本，中等 RTO）

```
RPO：分钟
RTO：数小时
成本：低-中（$$$$）

实施：
主 AWS：
├── 完整生产环境
└── 所有流量

备 GCP（最小化）：
├── 数据库副本（实时）
├── S3 → GCS 复制（实时）
└── 基础设施代码就绪（未运行）

灾难时：
1. 扩展 GCP 基础设施
2. 将 DNS 指向 GCP
3. 恢复运营（1-4 小时）

最适合：具有中等正常运行时间需求的重要系统
```

#### 策略 3：温备（中等成本，低 RTO）

```
RPO：秒
RTO：分钟
成本：中（$$$$$$）

实施：
主 AWS：
├── 完整生产（100% 流量）

备 GCP：
├── 缩小的副本（以 20% 容量运行）
├── 实时数据复制
└── 准备就绪，可即时扩展

灾难时：
1. 将 GCP 扩展到 100%
2. 通过 DNS/负载均衡器重定向流量
3. <15 分钟内运营

最适合：业务关键的 ML 服务
```

#### 策略 4：多区域双活（成本最高，RTO 最短）

```
RPO：接近零
RTO：秒（自动故障转移）
成本：高（$$$$$$$$）

实施：
两个区域同时处理流量：
AWS 美东：          AWS 美西：
├── 50% 流量        ├── 50% 流量
├── 实时数据同步 ←→ └── 实时数据同步

灾难时（美东故障）：
- 美西自动处理 100% 流量
- 无需手动干预
- 用户可能不会注意到故障

最适合：任务关键型系统
```

---

## 6. 总结与关键要点

### 核心概念

1. **多云**提供冗余和灵活性
   - 最佳服务组合：使用每个云的优势
   - 主备：灾难恢复
   - 考虑复杂性 vs 收益

2. **成本优化**对 ML 至关重要
   - 合理调整实例规格（分析实际使用情况）
   - 自动扩展推理端点
   - 对训练使用 Spot 实例
   - 调度开发/测试环境
   - 存储分层（S3 Standard → Glacier）

3. **FinOps**带来问责制
   - 为成本分配给所有资源打标签
   - 设置预算和告警
   - 与团队定期进行成本审查

4. **灾难恢复**确保业务连续性
   - 定义 RPO/RTO 目标
   - 根据关键性选择 DR 策略
   - 定期测试 DR 计划

### 获得的实践技能

✅ 设计多云架构
✅ 优化 ML 工作负载成本（可实现 60-70% 节省）
✅ 实施 FinOps 实践（标签、预算）
✅ 规划灾难恢复策略
✅ 合理调整规格、自动扩展和调度资源

### 应避免的常见陷阱

❌ 开发/测试 24/7 运行（使用调度）
❌ "以防万一"过度配置（合理调整规格）
❌ 不为训练使用 Spot 实例
❌ 将所有数据存储在热存储中
❌ 无成本标签（无法跟踪支出）
❌ 无灾难恢复计划

### 下一步

- **分析**：查看当前云支出，识别最高成本
- **优化**：实施自动扩缩容和调度
- **打标签**：为成本跟踪给所有资源打标签
- **测试**：运行 DR 演习以验证恢复流程

### 其他资源

- [AWS Cost Optimization Best Practices](https://aws.amazon.com/pricing/cost-optimization/)
- [GCP Rightsizing Recommendations](https://cloud.google.com/compute/docs/instances/apply-sizing-recommendations-for-instances)
- [FinOps Foundation](https://www.finops.org/)
- [AWS Well-Architected Framework - Cost Optimization](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/)

---

**恭喜！** 你现在了解了如何构建具有成本效益、弹性的多云 ML 基础设施，并具备适当的财务治理和灾难恢复能力。

**最终模块完成！**
