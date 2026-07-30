# model-api Helm Chart

Project 02 中 ML 模型服务 API 的 Helm chart。

该 chart 将 `../kubernetes/` 中的部署清单打包为一个可复用、参数化的 chart，适用于在开发、预发布和生产环境之间推送同一制品。

## 快速开始

```bash
# 安装到当前命名空间
helm install model-api ./helm/model-api

# 按环境覆盖值
helm upgrade --install model-api ./helm/model-api -f values-prod.yaml

# 在应用前检查渲染后的清单
helm template model-api ./helm/model-api
```

## 配置项

详见 `values.yaml` 中的全部可配置选项。最常覆盖的配置项：

| 键 | 默认值 | 用途 |
|---|---|---|
| `image.repository` | `model-api` | 容器镜像仓库 |
| `image.tag` | `1.0.0` | 镜像标签（每次部署设置） |
| `replicaCount` | `2` | 自动伸缩禁用时的静态副本数 |
| `autoscaling.enabled` | `true` | 启用/禁用 HPA |
| `autoscaling.minReplicas` / `maxReplicas` | `2` / `10` | HPA 范围 |
| `ingress.hosts[0].host` | `model-api.example.com` | Ingress 主机名 |
| `resources` | 200m/512Mi req | 容器资源 |
| `modelStorage.type` | `pvc` | `pvc`、`s3`、`gcs` 或 `initContainer` |
| `env` | `LOG_LEVEL`、`MODEL_PATH`、`WORKERS` | 容器环境变量 |
| `serviceMonitor.enabled` | `true` | 创建 Prometheus ServiceMonitor |

## 跨环境推送

维护按环境区分的 values 文件：

```
helm/
  model-api/
    values.yaml           # 默认值
    values-dev.yaml
    values-staging.yaml
    values-prod.yaml
```

```bash
helm upgrade --install model-api ./helm/model-api \
  -n model-api-prod --create-namespace \
  -f ./helm/model-api/values-prod.yaml \
  --set image.tag=$GIT_SHA \
  --atomic --timeout 5m
```

`--atomic` 在失败时回滚；`--timeout` 限制滚动更新等待时间。

## 从原始清单迁移

`../kubernetes/` 中的原始清单与此 chart 在默认值下生成等效的资源。一旦 chart 投入使用，原始清单仅作为参考/教学辅助——运维操作通过 Helm 进行。

## 故障排查

```bash
# 将应用什么内容？
helm template model-api ./helm/model-api -f values-prod.yaml

# 与当前版本对比差异
helm diff upgrade model-api ./helm/model-api -f values-prod.yaml

# 回滚
helm rollback model-api
```
