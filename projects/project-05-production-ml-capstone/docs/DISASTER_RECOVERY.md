# 灾难恢复计划：生产 ML 系统

**最后更新：** 2025年10月18日
**版本：** 1.0
**分类：** 机密

---

## 目录

1. [概述](#overview)
2. [恢复目标](#recovery-objectives)
3. [备份策略](#backup-strategy)
4. [恢复流程](#recovery-procedures)
5. [测试计划](#testing-schedule)
6. [角色与职责](#roles-and-responsibilities)

---

## 概述

本灾难恢复（DR）计划概述了在发生灾难性故障时恢复生产 ML 系统的流程，包括：

- 完整集群故障
- 数据中心宕机
- 勒索软件攻击
- 意外数据删除
- 区域云服务商宕机
- 级联系统故障

### 范围

**范围内：**
- Kubernetes 集群和工作负载
- PostgreSQL 数据库（MLflow 元数据）
- 对象存储（S3/MinIO）— 模型和工件
- 配置（Git 仓库）
- 密钥和凭据

**范围外：**
- 云提供商基础设施（假设可通过 IaC 恢复）
- 网络基础设施
- 第三方 SaaS 服务

---

## 恢复目标

### RTO（恢复时间目标）

系统必须恢复前的最大可接受停机时间：

| 组件 | RTO | 优先级 |
|-----------|-----|----------|
| **ML API（生产）** | 1 小时 | 关键 |
| **MLflow** | 4 小时 | 高 |
| **监控** | 8 小时 | 中 |
| **ML 流水线** | 24 小时 | 低 |

### RPO（恢复点目标）

最大可接受数据丢失量：

| 组件 | RPO | 备份频率 |
|-----------|-----|------------------|
| **PostgreSQL** | 24 小时 | 每日 |
| **模型工件** | 24 小时 | 每日 |
| **配置** | 0（Git 历史） | 持续 |
| **日志** | 1 周 | 每周归档 |

### 服务等级协议（SLA）

- **生产 API 可用性：** 99.9% 可用性
- **数据持久性：** 99.999%（无数据丢失）
- **平均恢复时间：** < 2 小时

---

## 备份策略

### 需要备份的内容

1. **Kubernetes 资源**
   - Deployment、Service、ConfigMap
   - Ingress、Secret（加密）
   - PersistentVolumeClaim

2. **数据库**
   - PostgreSQL（MLflow 元数据）
   - 完整数据库转储
   - 事务日志

3. **对象存储**
   - 模型工件（S3/MinIO）
   - 训练数据集
   - 实验日志

4. **配置**
   - Git 仓库（GitHub）
   - 基础设施即代码（Terraform/Pulumi）
   - Helm chart 和 values

5. **密钥（加密）**
   - API 密钥
   - 数据库凭据
   - TLS 证书

### 备份计划

| 备份类型 | 频率 | 保留期 | 存储位置 |
|-------------|-----------|-----------|------------------|
| **Kubernetes 清单** | 每日 | 30 天 | S3（us-west-2） |
| **PostgreSQL 完整备份** | 每日 UTC 时间凌晨 2 点 | 30 天 | S3（us-west-2 + us-east-1） |
| **PostgreSQL 增量备份** | 每小时 | 7 天 | S3（us-west-2） |
| **模型工件** | 每日 | 90 天 | S3（us-west-2 + us-east-1） |
| **Git 仓库** | 持续 | 无限制 | GitHub + 镜像 |
| **密钥（加密）** | 每周 | 30 天 | S3（加密）+ Vault |

### 备份自动化

#### 每日 Kubernetes 备份脚本

```bash
#!/bin/bash
# /scripts/backup-kubernetes.sh

BACKUP_DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/backups/${BACKUP_DATE}"
S3_BUCKET="s3://ml-system-backups"

# 创建备份目录
mkdir -p ${BACKUP_DIR}

# 备份生产命名空间中的所有资源
kubectl get all -n ml-system-production -o yaml > ${BACKUP_DIR}/all-resources.yaml
kubectl get configmap -n ml-system-production -o yaml > ${BACKUP_DIR}/configmaps.yaml
kubectl get secret -n ml-system-production -o yaml > ${BACKUP_DIR}/secrets.yaml
kubectl get pvc -n ml-system-production -o yaml > ${BACKUP_DIR}/pvcs.yaml
kubectl get ingress -n ml-system-production -o yaml > ${BACKUP_DIR}/ingress.yaml

# 备份 PostgreSQL
kubectl exec -n ml-system-production postgresql-0 -- \
  pg_dump -U mlflow mlflow > ${BACKUP_DIR}/mlflow-db.sql

# 压缩备份
tar -czf ${BACKUP_DIR}.tar.gz ${BACKUP_DIR}

# 上传到 S3（主区域）
aws s3 cp ${BACKUP_DIR}.tar.gz ${S3_BUCKET}/${BACKUP_DATE}.tar.gz

# 复制到辅助区域
aws s3 cp ${S3_BUCKET}/${BACKUP_DATE}.tar.gz \
  s3://ml-system-backups-dr/$(basename ${BACKUP_DIR}).tar.gz \
  --source-region us-west-2 \
  --region us-east-1

# 清理本地备份（仅保留最近 7 天）
find /backups -type d -mtime +7 -exec rm -rf {} \;

echo "备份完成：${BACKUP_DATE}"
```

#### 自动备份 CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cronjob
  namespace: ml-system-production
spec:
  schedule: "0 2 * * *"  # 每日 UTC 时间凌晨 2 点
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-sa
          containers:
          - name: backup
            image: backup-tools:latest
            command: ["/scripts/backup-kubernetes.sh"]
            volumeMounts:
            - name: backups
              mountPath: /backups
          volumes:
          - name: backups
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### 备份验证

**每周验证：**
- 将备份恢复到测试环境
- 验证数据完整性
- 测试应用功能
- 记录任何问题

```bash
# 验证脚本
#!/bin/bash
# /scripts/verify-backup.sh

LATEST_BACKUP=$(aws s3 ls s3://ml-system-backups/ | sort | tail -n 1 | awk '{print $4}')

# 下载备份
aws s3 cp s3://ml-system-backups/${LATEST_BACKUP} /tmp/

# 解压
tar -xzf /tmp/${LATEST_BACKUP} -C /tmp/

# 恢复到测试命名空间
kubectl apply -f /tmp/backups/*/all-resources.yaml --namespace=ml-system-test

# 运行冒烟测试
pytest tests/integration/test_e2e.py --api-url=https://test.example.com

echo "备份验证完成"
```

---

## 恢复流程

### 场景 1：单个 Pod 故障

**症状：** 一个或多个 Pod 崩溃或无响应

**RTO：** < 2 分钟（自动）

**恢复：**
1. Kubernetes 自动重启故障 Pod
2. 健康检查将不健康 Pod 从服务中移除
3. 无需手动干预

**验证：**
```bash
kubectl get pods -n ml-system-production
kubectl logs -n ml-system-production <pod-name>
```

---

### 场景 2：节点故障

**症状：** 整个节点不可用

**RTO：** < 5 分钟（自动）

**恢复：**
1. Kubernetes 检测到节点故障
2. 将 Pod 重新调度到健康节点
3. 服务以降低的容量继续运行
4. 自动扩缩容可能触发以增加容量

**手动干预（如果需要）：**
```bash
# 排空故障节点
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# 删除节点
kubectl delete node <node-name>

# 添加新节点（通过云提供商）
# 节点通过自动扩缩组自动加入集群
```

**验证：**
```bash
kubectl get nodes
kubectl get pods -n ml-system-production -o wide
```

---

### 场景 3：数据库故障

**症状：** PostgreSQL Pod 崩溃或数据损坏

**RTO：** < 30 分钟

**恢复：**

#### 步骤 1：识别故障
```bash
# 检查数据库 Pod
kubectl get pods -n ml-system-production | grep postgres

# 检查日志
kubectl logs -n ml-system-production postgresql-0
```

#### 步骤 2：尝试自动恢复
如果使用带复制的 PostgreSQL：
```bash
# 将副本提升为主节点
# （这取决于你的 PostgreSQL 设置：Patroni、Stolon 等）
```

#### 步骤 3：从备份恢复
```bash
# 获取最新备份
LATEST_BACKUP=$(aws s3 ls s3://ml-system-backups/ | grep mlflow-db | sort | tail -n 1 | awk '{print $4}')

# 下载备份
aws s3 cp s3://ml-system-backups/${LATEST_BACKUP} /tmp/mlflow-db.sql.gz

# 解压
gunzip /tmp/mlflow-db.sql.gz

# 删除现有数据库（如果已损坏）
kubectl exec -n ml-system-production postgresql-0 -- dropdb -U mlflow mlflow

# 创建新数据库
kubectl exec -n ml-system-production postgresql-0 -- createdb -U mlflow mlflow

# 从备份恢复
kubectl exec -i -n ml-system-production postgresql-0 -- \
  psql -U mlflow mlflow < /tmp/mlflow-db.sql
```

#### 步骤 4：验证恢复
```bash
# 检查数据库连接
kubectl exec -n ml-system-production postgresql-0 -- psql -U mlflow -c "SELECT COUNT(*) FROM experiments;"

# 重启 ML API Pod 以重新连接
kubectl rollout restart deployment/ml-api -n ml-system-production
```

**数据丢失：** 最多 24 小时（上一次备份）

---

### 场景 4：完整集群故障

**症状：** 整个 Kubernetes 集群不可用

**RTO：** < 1 小时

**恢复：**

#### 步骤 1：部署新集群

```bash
# 如果使用托管 Kubernetes（GKE/EKS/AKS）：
# 通过云提供商控制台或 CLI 创建新集群

# GKE 示例：
gcloud container clusters create ml-cluster-dr \
  --zone us-east1-a \
  --num-nodes 5 \
  --machine-type n1-standard-8 \
  --enable-autoscaling --min-nodes 3 --max-nodes 20

# 获取凭据
gcloud container clusters get-credentials ml-cluster-dr --zone us-east1-a
```

#### 步骤 2：安装核心基础设施

```bash
# 安装 NGINX Ingress
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# 安装 cert-manager
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# 安装 Prometheus/Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace
```

#### 步骤 3：恢复应用

```bash
# 下载最新备份
LATEST_BACKUP=$(aws s3 ls s3://ml-system-backups/ | sort | tail -n 1 | awk '{print $4}')
aws s3 cp s3://ml-system-backups/${LATEST_BACKUP} /tmp/
tar -xzf /tmp/${LATEST_BACKUP} -C /tmp/

# 创建命名空间
kubectl create namespace ml-system-production

# 恢复密钥（如果加密，先解密）
kubectl apply -f /tmp/backups/*/secrets.yaml

# 恢复 ConfigMap
kubectl apply -f /tmp/backups/*/configmaps.yaml

# 恢复 PVC
kubectl apply -f /tmp/backups/*/pvcs.yaml

# 恢复应用
kubectl apply -f /tmp/backups/*/all-resources.yaml

# 等待 Pod 就绪
kubectl wait --for=condition=Ready pods -l app=ml-api \
  -n ml-system-production --timeout=10m
```

#### 步骤 4：恢复数据库

```bash
# 恢复 PostgreSQL（同场景 3 步骤 3）
```

#### 步骤 5：更新 DNS

```bash
# 获取新的 LoadBalancer IP
NEW_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 更新 DNS A 记录指向 NEW_IP
# （在你的 DNS 提供商处操作）

echo "更新 DNS：api.example.com → ${NEW_IP}"
```

#### 步骤 6：验证恢复

```bash
# 等待 DNS 传播
# dig api.example.com

# 测试端点
curl https://api.example.com/health

# 运行集成测试
export API_URL=https://api.example.com
export API_KEY=$PRODUCTION_API_KEY
pytest tests/integration/test_e2e.py -v
```

#### 步骤 7：监控

- 查看 Grafana 仪表盘
- 检查错误率和延迟
- 审查日志中的问题
- 监控 24 小时

---

### 场景 5：勒索软件攻击 / 数据删除

**症状：** 意外数据删除、加密或损坏

**RTO：** < 4 小时

**恢复：**

#### 步骤 1：隔离与评估

```bash
# 立即隔离受影响的系统
# 如有必要，断开网络连接

# 评估损害
# 确定受影响的数据/系统
```

#### 步骤 2：从不可变备份恢复

```bash
# 使用攻击前的备份
# 备份应存储在不可变存储中（S3 Object Lock）

# 列出可用备份
aws s3 ls s3://ml-system-backups/ | grep $(date -d '7 days ago' +%Y%m%d)

# 从攻击前备份恢复
# 按照场景 4 的步骤操作
```

#### 步骤 3：安全审查

- 更改所有凭据和 API 密钥
- 轮换 TLS 证书
- 审查审计日志
- 确定攻击向量
- 修补漏洞

#### 步骤 4：验证数据完整性

```bash
# 检查恢复的数据完整性
# 与已知良好备份的校验和进行比较
# 运行数据验证测试
```

---

## 测试计划

### 备份测试

| 测试类型 | 频率 | 负责人 | 时长 |
|-----------|-----------|-------------|----------|
| **备份验证** | 每周 | DevOps | 30 分钟 |
| **单 Pod 恢复** | 每月 | DevOps | 15 分钟 |
| **数据库恢复** | 每季度 | DevOps + DBA | 2 小时 |
| **完整 DR 演练** | 每年 | 所有团队 | 8 小时 |

### 年度 DR 演练

**日期：** 第一季度（一月）的第一个周六
**时长：** 8 小时
**参与人员：** DevOps、SRE、工程、管理

**流程：**
1. **模拟故障** — 关闭生产集群（在维护窗口内）
2. **执行恢复** — 按照 DR 流程操作
3. **验证功能** — 运行完整测试套件
4. **记录问题** — 记录遇到的任何问题
5. **更新计划** — 根据经验教训修订 DR 计划
6. **报告结果** — 向领导层汇报

---

## 角色与职责

### 事件指挥官
- **主要负责人：** 工程经理
- **后备人员：** 高级 DevOps 工程师
- **职责：**
  - 宣告灾难
  - 协调恢复工作
  - 与利益相关者沟通
  - 就恢复策略做出决策

### 恢复团队

**DevOps 工程师：**
- 执行恢复流程
- 恢复基础设施
- 验证系统健康状态

**数据库管理员：**
- 恢复数据库
- 验证数据完整性
- 恢复后优化查询

**软件工程师：**
- 验证应用功能
- 修复发现的任何问题
- 支持测试

**SRE/值班人员：**
- 监控恢复进度
- 升级问题
- 提供 24/7 覆盖

### 沟通

**内部：**
- 每 30 分钟更新状态
- Slack #incident-response 频道
- 向领导层发送邮件更新

**外部：**
- 状态页面更新（status.example.com）
- 客户通知（如果 SLA 受影响）
- 事后分析报告（如果合适则公开）

---

## 紧急联系信息

| 角色 | 姓名 | 电话 | 邮箱 |
|------|------|-------|-------|
| **事件指挥官** | 待定 | 待定 | 待定 |
| **DevOps 负责人** | 待定 | 待定 | 待定 |
| **工程经理** | 待定 | 待定 | 待定 |
| **CTO** | 待定 | 待定 | 待定 |
| **云提供商支持** | - | - | support@cloudprovider.com |
| **安全团队** | - | - | security@example.com |

**紧急升级路径：**
1. 值班工程师（PagerDuty）
2. DevOps 负责人
3. 工程经理
4. CTO

---

## 恢复后操作

### 立即行动（0-4 小时）

1. **验证系统健康状态**
   - 所有服务正常运行
   - 无数据丢失或损坏
   - 性能指标正常

2. **持续监控**
   - 关注异常情况
   - 检查错误率
   - 审查日志

3. **沟通状态**
   - 更新状态页面
   - 通知利益相关者
   - 发送解除警报消息

### 短期行动（4-24 小时）

1. **根因分析**
   - 确定故障原因
   - 记录时间线
   - 收集证据

2. **评估影响**
   - 计算停机时间
   - 衡量数据丢失
   - 估算业务影响

3. **创建行动项**
   - 预防措施
   - 流程改进
   - 基础设施变更

### 长期行动（1-4 周）

1. **事后分析**
   - 撰写详细报告
   - 与团队分享
   - 向领导层汇报

2. **实施改进**
   - 修复根因
   - 更新 Runbook
   - 增强监控

3. **更新 DR 计划**
   - 纳入经验教训
   - 修订流程
   - 更新文档

4. **培训**
   - 培训团队新流程
   - 进行桌面推演
   - 更新知识库

---

## 持续改进

此 DR 计划应定期审查和更新：

- **每季度：** 每次 DR 测试后
- **事件后：** 任何实际灾难恢复后
- **变更时：** 系统架构变更时
- **每年：** 全面审查

**文档版本：** 1.0
**上次测试时间：** [首次测试后填写]
**下次审查：** 2026年1月18日
**负责人：** DevOps 团队

---

**记住：** 熟能生巧。你越频繁地测试 DR 流程，实际恢复时就越顺利！
