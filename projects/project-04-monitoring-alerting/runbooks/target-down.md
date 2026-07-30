# 手册：TargetDown

**告警：** `TargetDown`
**严重级别：** critical
**是否呼叫：** 是

## 含义

Prometheus 已经 5 分钟无法采集配置的目标。我们失去了该目标的指标——依赖此目标指标的告警被静默。

## 有多严重？

**严重程度取决于目标。** 一个宕机的 API Pod 会影响用户。一个宕机的 Prometheus exporter 仅影响监控。

## 首先检查

1. **识别缺失的目标。** 告警标签 `instance` 和 `job` 告诉你什么缺失了。
2. **Pod 是否真的在运行？**
   ```
   kubectl get pods -A -o wide | grep <instance>
   ```
3. **网络路径是否正常？**
   ```
   kubectl run -it --rm debug --image=curlimages/curl -- \
     curl -sf http://<instance>/metrics
   ```
4. **Prometheus 本身是否健康？** 如果多个 `TargetDown` 告警同时触发，Prometheus 本身或其网络可能出了问题。

## 可能的原因

1. **Pod 重启循环。** 在 exporter 响应之前崩溃。
2. **网络策略阻止采集。** 最近的 NetworkPolicy 变更。
3. **服务网格/sidecar 问题。** mTLS 配置不匹配。
4. **Prometheus 采集配置删除了该目标。** 最近的 ConfigMap 编辑。
5. **Prometheus Pod 中的 DNS 解析失败。**

## 缓解措施

- 重启受影响的 Pod：`kubectl rollout restart deployment/<name> -n <ns>`。
- 验证 Prometheus 目标页面：`kubectl port-forward -n monitoring svc/prometheus 9090:9090` → http://localhost:9090/targets。
- 如果原因是 NetworkPolicy，在所属团队调查期间，临时将 Prometheus 命名空间添加到其 `from` 选择器中。

## 何时需要进行事后审查

- 导致真实事件中错过呼叫。
- 诊断耗时超过 1 小时。
