# OpsMonitor — Grafana 仪表盘与 Prometheus 告警规则合集

面向中间件运维监控的合集，覆盖 **RabbitMQ、Redis、Elasticsearch、MySQL** 四大常用组件。包含两部分内容：基于 Prometheus 数据源的 Grafana Dashboard（社区原版基础上深度优化：修正失效查询、补齐监控指标、每个面板中文说明，开箱即用），以及配套的 Prometheus 告警规则与 Alertmanager 配置。

## 仪表盘一览

| 目录 | 仪表盘 | UID | 面板数 | 说明 |
| --- | --- | --- | --- | --- |
| `grafana/rabbitmq/` | RabbitMQ (Optimized) | `rmq-opt-4371` | 35 | 推荐，指标最全，按 6 个分类行组织 |
| `grafana/rabbitmq/` | RabbitMQ | `Z1Tw73yGk` | 12 | 社区原版（备份参考） |
| `grafana/redis/` | Redis (Optimized) | `redis-opt-763` | 46 | 推荐，分区布局 |
| `grafana/redis/` | Redis | `OY8i3bdMk` | 26 | 社区原版（备份参考） |
| `grafana/elasticsearch/` | ElasticSearch (Optimized) | `es-opt4-6483` | 119 | 推荐，适配 exporter 1.7.0 |
| `grafana/elasticsearch/` | ElasticSearch | `_JI2VxBGz` | 93 | 社区原版（备份参考） |
| `grafana/mysql/` | MySQL (Optimized) | `mysql_overview_optimized` | 45 | 推荐 |
| `grafana/mysql/` | Mysql | `MQWgroiiz` | 39 | 社区原版（备份参考） |

## 优化版仪表盘特性

- **查询修复**：修正拼写错误、引用不存在的变量、Gauge 面板误用 `rate/irate`、重复序列等问题；对行为性查询使用 PromQL `or` 兜底，保证面板始终有数据。
- **指标补齐**：对照 exporter 官方文档逐项核对，补齐队列级指标（`message_bytes`、`memory`、`consumers`、`consumer_utilisation`、`messages_persistent/ram` 等）、Counter 类指标（`confirm`、`get`、`returned`、`disk_reads/writes`）、交换机收发量与集群 Overview 汇总。
- **中文注释**：每个面板的 description 字段均写入中文说明（悬停面板左上角 i 图标可见），dashboard 级说明同样为中文，降低团队使用门槛。
- **分类布局**：面板按功能分类行组织（如 RabbitMQ 的集群概览 / 消息总量与速率 / 队列深度·积压·消费者 / 交换机与路由 / 节点资源与健康 / 连接与对象总数），支持按分区折叠聚焦。
- **兼容性**：适配 Grafana v7.x（schemaVersion 26），不依赖 `$__rate_interval`，rate 窗口已显式写死，导入即可正常出图。

## 环境要求

| 组件 | 要求 |
| --- | --- |
| Grafana | v7.1.4+（schemaVersion 26） |
| 数据源 | Prometheus |
| RabbitMQ | [kbudde/rabbitmq_exporter](https://github.com/kbudde/rabbitmq_exporter) v0.29.0（注意 0.x 与 1.x 指标命名差异） |
| Redis | [oliver006/redis_exporter](https://github.com/oliver006/redis_exporter) |
| Elasticsearch | [justwatch/elasticsearch_exporter](https://github.com/justwatch/elasticsearch_exporter) 1.7.0 |
| MySQL | [prometheus/mysqld_exporter](https://github.com/prometheus/mysqld_exporter) |
| Prometheus | 规则文件兼容 Prometheus 2.x / VictoriaMetrics |
| Alertmanager | 邮件 / 企业微信 / 钉钉 webhook 路由 |

> 注意：exporter 升级大版本后部分指标命名会变化，届时需要同步调整仪表盘查询。

## 已知说明

- **RabbitMQ Head Message Age**：`rabbitmq_queue_head_message_timestamp` 为条件性指标，仅当生产者发布消息时设置了 AMQP `timestamp` 属性且队首消息处于 paged-in 状态时才会出现。经实测本环境业务消息均未设置该属性（管理接口对所有队列返回 null，exporter v0.29.0 与 classic 队列类型已排除其他原因），故优化版未包含该面板。若将来业务设置该属性，可按此指标自行补充。
- **Consumer Utilisation**：RabbitMQ 仅在利用率低于 1（消费者被 prefetch/网络限流）时上报该指标，面板无数据即消费健康，出现数据说明消费能力已跟不上。
- **同名集群区分**：Elasticsearch 优化版的变量链以 Exporter 实例为先选条件，面板查询均同时带 `instance` 过滤，避免同名集群互相串数据。

## 使用方式

1. 在 Grafana 中添加 Prometheus 数据源；
2. 进入 **Dashboards → Import**，上传对应的 `*-optimized*.json` 文件（或通过 Dashboard ID 直接导入）；
3. 选择你的 Prometheus 数据源，保存即可。

优化版仪表盘的 UID 与社区原版不同，导入不会覆盖环境中的原版仪表盘；但同一 UID 的优化版重复导入会更新已有仪表盘（用于版本迭代）。

## Prometheus 告警规则

`prometheus/rules/` 下按 exporter 拆分为四个规则文件，共 43 条告警，格式对齐 `node_exporter.rule`，指标查询与优化版仪表盘保持一致，job 过滤匹配 `.*<exporter>_exporter.*`（对应 Consul 注册的 job 名）：

| 文件 | 规则数 | 核心覆盖 |
| --- | --- | --- |
| `mysql.rule` | 11 | 实例存活、异常重启、主从 IO/SQL 线程、复制错误号与延迟、连接数占比、慢查询、exporter 抓取错误 |
| `redis.rule` | 10 | 实例存活、主从链路、内存/maxmemory 占比、连接数、持久化失败（RDB/AOF）、key 驱逐、拒绝连接、碎片率、慢日志 |
| `rabbitmq.rule` | 10 | exporter 存活、服务运行状态、内存/磁盘水位告警、网络分区、队列积压、unacked、业务队列专项、无消费者 |
| `elasticsearch.rule` | 12 | exporter 存活、集群健康 red/yellow、节点数变化、JVM 堆与 GC、CPU、熔断器、unassigned shards、数据盘水位、pending tasks |

规则设计要点：severity 分 High / critical 两级对接 Alertmanager 路由；描述统一携带 `app`/`env` 标签；对环境特化场景做了取舍（如 ES yellow 仅对有副本的多节点集群告警，避免单节点集群常驻误报；Redis 碎片率仅在使用量超 1G 时告警）。部分队列级告警需按业务实际排除特定模式的队列（如拉模式消费的缓冲队列）。

`prometheus/rules/old/` 保留了重构前生产在用的四个规则文件，仅作参考对照，不参与加载（Prometheus 的 `rule_files` glob 不递归子目录）。

## 目录结构

```
opsmonitor/
├── alertmanager/
│   ├── alertmanager.yml           # 路由/抑制/接收器配置
│   ├── email.tmpl                 # 邮件模板（生产）
│   ├── email_dev.tmpl             # 邮件模板（开发）
│   └── wechat.tmpl                # 企业微信模板
├── prometheus/
│   ├── prometheus.yml             # 主配置（Consul 服务发现、k8s 抓取、remote_write）
│   └── rules/
│       ├── node_exporter.rule     # 主机层告警（生产沿用）
│       ├── mysql.rule             # MySQL 告警（11 条）
│       ├── redis.rule             # Redis 告警（10 条）
│       ├── rabbitmq.rule          # RabbitMQ 告警（10 条）
│       ├── elasticsearch.rule     # Elasticsearch 告警（12 条）
│       └── old/                   # 重构前生产规则（仅参考，不加载）
└── grafana/
    ├── rabbitmq/
    │   ├── RabbitMQ.json              # 社区原版
    │   └── RabbitMQ-optimized.json    # 优化版（推荐）
    ├── redis/
    │   ├── redis.json                 # 社区原版
    │   └── redis-optimized.json       # 优化版（推荐）
    ├── elasticsearch/
    │   ├── elasticsearch.json         # 社区原版
    │   └── elasticsearch-optimized.json # 优化版（推荐）
    └── mysql/
        ├── mysql.json                 # 社区原版
        └── mysql-optimized.json       # 优化版（推荐）
```

## 脱敏说明

本仓库为对外可见副本，已做统一脱敏，**请勿将仓库内配置直接部署到生产**：

- 内网 IP 一律掩码为前两段 + `xxx`（如 `192.168.xxx`），正则中的 IP 同步处理；
- Prometheus 的 `bearer_token` 与 Alertmanager 的企业微信 `api_secret` 替换为 `<REDACTED>`；
- 内部域名统一替换为 `example.com`；
- 邮箱与 SMTP 信息为占位示例（`test.com` / `PASS`）。

真实环境部署时需还原以上地址与凭据。Grafana 仪表盘 JSON 不含地址类敏感信息，导入使用不受影响。

## License

MIT
