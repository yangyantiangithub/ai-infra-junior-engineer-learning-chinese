# 手册：LogIngestionLag

**告警：** `LogIngestionLag`
**严重级别：** warning
**是否呼叫：** 否

## 含义

Logstash 从输入到 Elasticsearch 的延迟已超过 5 分钟。新日志仍在到达集群，但在 Kibana 中尚不可搜索。

## 有多严重？

正在调查活跃事件的操作员可能会看到陈旧的日志并做出错误的决策。如果延迟超过源 Pod 的保留窗口，日志可能会完全丢失。

## 首先检查

1. **Logstash 流水线是否积压？**
   ```promql
   logstash_pipeline_events_in_total - logstash_pipeline_events_out_total
   ```
2. **Elasticsearch 索引是否缓慢？**
   ```promql
   elasticsearch_indices_indexing_index_time_seconds_total
   ```
3. **输入速率是否激增？** 来自异常应用的日志突增可能会冲垮流水线。
4. **Elasticsearch 数据节点磁盘压力？**

## 可能的原因

1. **索引设置过时。** 新建大索引但没有滚动策略。
2. **Elasticsearch 上慢查询积压。** 大量 Kibana 查询阻塞索引线程。
3. **嘈杂的生产者。** 单个 Pod 以 MB/s 速率产生日志。
4. **集群红色/黄色状态**，由节点重启或分片重新分配导致。

## 缓解措施

- 通过索引大小和速率识别嘈杂的生产者：
  ```
  GET _cat/indices?v&s=store.size:desc
  ```
- 添加 Logstash 过滤器或更改容器级别的日志级别以减少日志量。
- 对于 Elasticsearch 压力：扩展数据节点或关闭低优先级索引。
- 如果索引长期缓慢，增加 Logstash 工作线程数或批处理大小。

## 何时需要进行事后审查

- 日志丢失（不仅仅是延迟）。
- 导致事件的 MTTR 增加超过 30 分钟。
