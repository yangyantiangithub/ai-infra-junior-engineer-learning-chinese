# 手册：HighErrorRate

**告警：** `HighErrorRate`
**严重级别：** critical
**是否呼叫：** 是（PagerDuty）
**仪表盘：** 模型 API — 概述 → 错误率面板

## 含义

模型 API 返回 5xx 响应的比例在 5 分钟窗口内超过了 5%。用户正在看到失败。

## 有多严重？

HPA 不会根据错误率进行伸缩。持续的高错误率意味着当前容量正常，但某些功能出了问题——到达 Pod 的请求正在失败而非排队。

## 首先检查（按顺序）

1. **确认告警反映实际情况。** 打开仪表盘。峰值仍在发生还是已经恢复？如果已恢复超过 10 分钟，降级为事后审查工单而非主动响应。
2. **识别失败的端点。**
   ```promql
   topk(5,
     sum(rate(http_requests_total{job="model-api",status=~"5.."}[5m]))
       by (path, status)
   )
   ```
3. **检查 Pod 重启次数。**
   ```
   kubectl -n model-api get pods -o wide
   kubectl -n model-api describe pod <name> | tail -50
   ```
4. **查看最近的部署。**
   ```
   kubectl -n model-api rollout history deployment/model-api
   ```
5. **采样日志。**
   ```
   kubectl -n model-api logs -l app.kubernetes.io/name=model-api --tail=200 | grep -E 'ERROR|Traceback'
   ```

## 可能的原因（按概率排序）

1. **错误的部署。** 最近的发布引入了 bug。检查发布历史（上面的第 4 步）。
   *缓解措施：* `kubectl -n model-api rollout undo deployment/model-api`
2. **下游依赖故障。** 数据库或特征存储不可达。查看它们的仪表盘。
   *缓解措施：* 呼叫所属团队；验证网络策略和凭据。
3. **内存压力。** Pod 被 OOM 杀死。
   *缓解措施：* 增加 `resources.limits.memory` 并发布。后续调查内存泄漏。
4. **错误的输入流量。** 客户端正在发送大量格式错误的请求。
   *缓解措施：* 通过客户端 IP/用户代理识别；在入口处限流。
5. **凭据过期。** 轮换的密钥尚未被运行中的 Pod 获取。
   *缓解措施：* 重启 Pod（`kubectl rollout restart`）。

## 缓解操作手册

### 快速回滚（如果是最近的部署）

```bash
kubectl -n model-api rollout history deployment/model-api
kubectl -n model-api rollout undo deployment/model-api
kubectl -n model-api rollout status deployment/model-api --timeout=5m
```

确认仪表盘上错误率在 5 分钟内下降。

### 扩容（如果是容量相关）

```bash
kubectl -n model-api scale deployment/model-api --replicas=10
```

仅在错误与高 CPU/内存压力相关时有帮助；对代码或依赖问题无效。

### 清退单个有问题的 Pod

```bash
kubectl -n model-api delete pod <name>
```

部署将替换它。如果所有 Pod 都报错则不要这样做——它会让你回到相同的状态。

## 回滚（如果缓解措施使情况恶化）

```bash
kubectl -n model-api rollout undo deployment/model-api --to-revision=<prior>
```

## 何时需要进行事后审查

- 持续超过 30 分钟且影响用户。
- 7 天内重复出现超过一次。
- 需要回滚。
- 解决后根因仍不明确。
