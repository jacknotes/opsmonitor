# OpsMonitor — Grafana 监控仪表盘合集

面向中间件运维监控的 Grafana Dashboard 合集，覆盖 **RabbitMQ、Redis、Elasticsearch** 三大常用组件。所有仪表盘基于 Prometheus 数据源，在社区原版基础上做了深度优化：修正失效查询、补齐监控指标、并为每个面板添加中文说明，开箱即用。

## 仪表盘一览

| 目录 | 仪表盘 | UID | 面板数 | 说明 |
| --- | --- | --- | --- | --- |
| `grafana/rabbitmq/` | RabbitMQ (Optimized v2) | `rmq-opt-4371-v2` | 31 | 推荐，指标最全 |
| `grafana/rabbitmq/` | RabbitMQ (Optimized) | `rmq-opt-4371` | 20 | 早期优化版 |
| `grafana/rabbitmq/` | RabbitMQ | `Z1Tw73yGk` | 12 | 社区原版（备份参考） |
| `grafana/redis/` | Redis (Optimized) | `redis-opt-763` | 43 | 优化版 |
| `grafana/redis/` | Redis | `OY8i3bdMk` | 26 | 社区原版（备份参考） |
| `grafana/elasticsearch/` | ElasticSearch (Optimized) | `es-opt-6483` | 93 | 优化版 |
| `grafana/elasticsearch/` | ElasticSearch | `_JI2VxBGz` | 93 | 社区原版（备份参考） |

## 优化版仪表盘特性

- **查询修复**：修正拼写错误、引用不存在的变量、Gauge 面板误用 `rate/irate`、重复序列等问题；对行为性查询使用 PromQL `or` 兜底，保证面板始终有数据。
- **指标补齐**：对照 exporter 官方文档逐项核对，补齐队列级指标（`message_bytes`、`memory`、`consumers`、`consumer_utilisation`、`messages_persistent`、`head_message_timestamp` 等）、Counter 类指标（`confirm`、`get`、`returned`、`disk_reads/writes`）、交换机收发量与集群 Overview 汇总。
- **中文注释**：每个面板的 description 字段均写入中文说明（悬停面板左上角 i 图标可见），dashboard 级说明同样为中文，降低团队使用门槛。
- **兼容性**：适配 Grafana v7.x（schemaVersion 26），不依赖 `$__rate_interval`，rate 窗口已显式写死，导入即可正常出图。

## 环境要求

| 组件 | 要求 |
| --- | --- |
| Grafana | v7.1.4+（schemaVersion 26） |
| 数据源 | Prometheus |
| RabbitMQ | [kbudde/rabbitmq_exporter](https://github.com/kbudde/rabbitmq_exporter) v0.29.0（注意 0.x 与 1.x 指标命名差异） |
| Redis | [oliver006/redis_exporter](https://github.com/oliver006/redis_exporter) |
| Elasticsearch | [justwatch/elasticsearch_exporter](https://github.com/justwatch/elasticsearch_exporter) 1.1.x |

> 注意：exporter 升级大版本后部分指标命名会变化，届时需要同步调整仪表盘查询。

## 使用方式

1. 在 Grafana 中添加 Prometheus 数据源；
2. 进入 **Dashboards → Import**，上传对应的 `*-optimized*.json` 文件（或通过 Dashboard ID 直接导入）；
3. 选择你的 Prometheus 数据源，保存即可。

优化版仪表盘使用了全新的 UID（如 `rmq-opt-4371-v2`），导入时不会覆盖你环境中已有的原版仪表盘。

## 目录结构

```
opsmonitor/
└── grafana/
    ├── rabbitmq/
    │   ├── RabbitMQ.json              # 社区原版
    │   ├── RabbitMQ-optimized.json    # 优化版
    │   └── RabbitMQ-optimized_v2.json # 优化版 v2（推荐）
    ├── redis/
    │   ├── redis.json                 # 社区原版
    │   └── redis-optimized.json       # 优化版
    └── elasticsearch/
        ├── elasticsearch.json         # 社区原版
        └── elasticsearch-optimized.json # 优化版
```

## License

MIT
