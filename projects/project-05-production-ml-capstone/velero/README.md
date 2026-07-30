# Velero — 集群备份与恢复

Velero 将 Kubernetes API 对象和持久化卷快照备份到 S3。它是 `docs/DISASTER_RECOVERY.md` 的灾难恢复对应工具，也是使该文档可执行而非空想的运行时工具。

## 防护范围

- **意外的命名空间删除。** 从昨天的备份恢复命名空间。
- **集群丢失。** 将整个集群的工作负载恢复到新集群（生产到预发布的演练）。
- **数据损坏。** 从损坏前的快照恢复 PVC。
- **区域故障。** 跨区域复制备份；在辅助区域启动新集群。

**不**防护以下内容：

- **备份本身被删除。** 这需要 S3 版本控制 + 对象锁定。
- **数据中的错误。** Velero 捕获的是存在的任何状态；如果备份时数据已损坏，恢复时同样损坏。

## 安装

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm upgrade --install velero vmware-tanzu/velero \
  -n velero --create-namespace \
  -f velero/values.yaml
```

前置条件（由 Terraform 在 `terraform/modules/iam` 和本章未展示的 S3 存储桶模块中配置）：

- 启用版本控制 + KMS 加密的 S3 存储桶。
- IRSA 角色，拥有存储桶的 `s3:GetObject`、`s3:PutObject`、`s3:DeleteObject` 权限以及 KMS 密钥的 `kms:Decrypt`/`Encrypt` 权限。
- EBS CSI 驱动（快照支持）。

## 日常操作

```bash
# 触发一个命名空间的临时备份
velero backup create model-api-now --include-namespaces model-api

# 列出最近的备份
velero backup get

# 查看备份内容
velero backup describe model-api-now --details
```

## 恢复

```bash
# 从最新的匹配备份恢复整个命名空间
velero restore create model-api-restore \
  --from-backup cluster-daily-2026-05-22 \
  --include-namespaces model-api

# 查看进度
velero restore describe model-api-restore --details
```

## 每季度验证

从未恢复过的备份只是希望，而非恢复计划。每季度运行一次有文档记录的恢复演练：

1. 将生产环境的备份保存到以 staging 为前缀的位置。
2. 将该备份恢复到干净的预发布集群。
3. 针对恢复后的堆栈运行 `loadtest/` 中的冒烟测试。
4. 测量：恢复时间、数据新鲜度、任何未自动恢复的内容（DNS、外部密钥等）。
5. 根据所学更新 `docs/DISASTER_RECOVERY.md`。

## 计划任务

`values.yaml` 中的两个默认计划：

| 名称 | 频率 | TTL | 范围 |
|---|---|---|---|
| `cluster-daily` | UTC 时间每天 01:00 | 30 天 | 除 kube-system/public/velero 外的所有命名空间 |
| `critical-hourly` | 每小时整点 | 7 天 | 仅 `model-api`、`mlflow`、`airflow` |

根据你的 RPO 调整频率和 TTL。如果可以从 CI 重建，每小时备份有些过度；如果不能，每日备份又过于粗糙。
