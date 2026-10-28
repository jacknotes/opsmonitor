# OpsMonitor — Grafana 仪表盘与 Prometheus 告警规则合集

面向中间件、主机系统与 Kubernetes 集群的 Grafana 仪表盘合集，覆盖 **RabbitMQ、Redis、Elasticsearch、MySQL、Nginx、IIS、Linux、Windows、LVS、ESXi、Kafka、SQL Server、ZooKeeper**，以及 **KSM、CoreDNS、Argo Rollouts、ArgoCD、API Server、Etcd、Cluster、Nodes、Pods** 等 K8s 监控视角。包含两部分内容：基于 Prometheus 数据源的 Grafana Dashboard（社区原版基础上深度优化：修正失效查询、补齐监控指标、每个面板中文说明，开箱即用），以及配套的 Prometheus 告警规则与 Alertmanager 配置。

## 仪表盘一览

| 目录 | 仪表盘 | UID | 面板数 | 说明 |
| --- | --- | --- | --- | --- |
| `grafana/rabbitmq/` | RabbitMQ (Optimized) | `rmq-opt-4371` | 35 | 指标最全，按 6 个分类行组织 |
| `grafana/redis/` | Redis (Optimized) | `redis-opt-763` | 46 | 分区布局 |
| `grafana/elasticsearch/` | ElasticSearch (Optimized) | `es-opt4-6483` | 119 | 适配 exporter 1.7.0 |
| `grafana/mysql/` | MySQL (Optimized) | `mysql_overview_optimized` | 45 | 实例概览与复制状态 |
| `grafana/nginx/` | Nginx (Optimized) | `nginx-opt-2949` | 29 | 公网入口场景；聚合面板不做 host 过滤（5000+ 分区基数过大），Top-N 用 topk 限流，未启用 proxy_cache 已移除缓存分区 |
| `grafana/iis/` | IIS (Optimized) | `iis-opt-12495` | 46 | 基于 windows_exporter iis 采集器，站点/应用池/请求与连接 |
| `grafana/linux/` | Linux (Optimized) | `linux-opt-9276` | 41 | 基于 node_exporter，CPU/内存/磁盘/网络/文件系统 |
| `grafana/windows/` | Windows (Optimized) | `win-opt-9182` | 47 | 基于 windows_exporter 0.20，兼容旧版指标命名 |
| `grafana/windows/` | Windows wmi_exporter (Optimized) | `win-wmi-opt-9182` | 36 | 独立适配 wmi_exporter 0.7.x / 0.8.x，避免与新版指标冲突 |
| `grafana/k8s/kube-state-metrics/` | KSM 01 集群总览 | `ksm-overview` | 10 | 集群级 Pod/命名空间/节点状态汇总 |
| `grafana/k8s/kube-state-metrics/` | KSM 02 节点 | `ksm-nodes` | 9 | 节点容量、状态、污点与不可调度 |
| `grafana/k8s/kube-state-metrics/` | KSM 03 工作负载 | `ksm-workloads` | 8 | Deployment/StatefulSet/DaemonSet/HPA 副本状态 |
| `grafana/k8s/kube-state-metrics/` | KSM 04 存储 | `ksm-storage` | 7 | PV/PVC/StorageClass 容量与状态 |
| `grafana/k8s/coredns/` | CoreDNS 监控 | `coredns-optimized` | 18 | 请求/响应码/缓存/转发/上游延迟，适配 kube-dns 与 node-local-dns |
| `grafana/k8s/argo-rollouts/` | Argo Rollouts 发布状态 | `argo-rollouts` | 9 | 发布阶段、副本可用性与异常事件 |
| `grafana/k8s/argocd/` | ArgoCD 应用发布监控 | `argocd` | 42 | 应用健康/同步状态、协调与工作队列、Git/gRPC/Redis、集群连接与缓存（36 面板 + 6 分类行） |
| `grafana/k8s/kubernetes-api-server/` | Kubernetes API Server 监控 | `kube-apiserver-optimized` | 18 | 请求延迟、错误率、Watch、准入与证书到期 |
| `grafana/k8s/kubernetes-cluster/` | Kubernetes Cluster 监控 | `k8s-cluster-monitor` | 29 | 节点状态总览、资源水位、Requests、限流与网络 |
| `grafana/k8s/kubernetes-etcd/` | Etcd 监控 | `etcd-monitor` | 30 | 成员健康、Leader、磁盘/DB 大小、gRPC 与延迟分位 |
| `grafana/k8s/kubernetes-nodes/` | Kubernetes Nodes(Optimized) | `qNcFB-jvz` | 41 | 节点资源、kubelet/cadvisor、文件系统与网络 |
| `grafana/k8s/kubernetes-pods/` | Kubernetes Pods 监控 | `k8s-pods-monitor` | 38 | 节点资源概览、指定 Pod 下钻、状态/重启/限流，Pod 与容器级 Top10（32 面板 + 6 分类行） |
| `grafana/lvs/` | LVS (Optimized) | `lvs-opt-5672` | 26 | 基于 node_exporter ipvs collector，VIP/后端/连接/流量/权重分区（8 卡片 + 11 图 + 后端明细表） |
| `grafana/nginx_upstream/` | Nginx Upstream (Optimized) | `nginx-upstream-opt` | 24 | 基于 Pushgateway 上报的 `nginx_upstream_status`，后端在线/离线、分组在线率与采集健康（8 卡片 + 10 图 + 3 表） |
| `grafana/esxi/` | ESXi (Optimized) | `esxi-opt-10076` | 38 | 基于 vmware_exporter，主机资源/数据存储/虚拟机/硬件传感器分区（8 卡片 + 15 图 + 10 表） |
| `grafana/kafka/` | Kafka (Optimized) | `kafka-opt-11962` | 52 | 基于 jmx_exporter，集群健康/吞吐/延迟/请求异常/消费组/JVM 分区（8 卡片 + 42 图 + 2 明细表） |
| `grafana/sqlserver/` | SQL Server (Optimized) | `sqlserver-opt` | 34 | 基于 mssql_exporter，连接阻塞/内存缓存/吞吐等待/日志明细分区（8 卡片 + 20 图 + 数据库明细表） |
| `grafana/zookeeper/` | ZooKeeper (Optimized) | `zookeeper-opt-10465` | 83 | ZK 自带指标端点 + JVM，九大分区、三列紧凑布局（8 卡片 + 57 图 + 1 明细表），JVM 区紧随选主区 |

每个目录同时保留社区原版模板（文件名不带 `-optimized`，如 `RabbitMQ.json`、`Kubernetes Cluster.json`），仅作对照参考，不参与导入；ArgoCD 目录同样为 `argocd.json`（原版）+ `argocd-optimized.json`（优化版）。

## 文件命名规范

- 目录：全部小写；K8s 子目录使用 kebab-case（`kubernetes-pods`、`kube-state-metrics`、`argo-rollouts`）。
- 文件：原版模板为 `<名称>.json`，优化版为 `<名称>-optimized.json`（如 `nginx-optimized.json`、`kubernetes-cluster-optimized.json`、`kube-state-metrics-01-cluster-overview-optimized.json`）。
- 导入时选择 `*-optimized.json`；原版仅用于对照与回溯。

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
| Nginx | [hnlq715/nginx-vts-exporter](https://github.com/hnlq715/nginx-vts-exporter)（实测 0.10.3 / nginx 1.17.3） |
| IIS / Windows | [prometheus-community/windows_exporter](https://github.com/prometheus-community/windows_exporter)（iis、os、net、service、process、system 采集器；旧环境可用 wmi_exporter 0.7.x/0.8.x） |
| Linux | [prometheus/node_exporter](https://github.com/prometheus/node_exporter) 1.8.x |
| Kafka | [prometheus/jmx_exporter](https://github.com/prometheus/jmx_exporter) javaagent（`kafka-2_0_0.yml` 采集规则），暴露 `kafka_*` 与 `jvm_*` 指标 |
| SQL Server | [burningalchemist/sql_exporter](https://github.com/burningalchemist/sql_exporter) / [awaragi/prometheus-mssql-exporter](https://github.com/awaragi/prometheus-mssql-exporter)（`mssql_*` 指标），账号需具备 `VIEW SERVER STATE` 与 `VIEW ANY DEFINITION` 权限 |
| ZooKeeper | 内置指标端点 `PrometheusMetricsProvider`（`metricsProvider.httpPort`，默认 7000，3.6+ 已自带 `jvm_*`）；旧版本可用 `dabealu/zookeeper-exporter` 或同机 jmx_exporter javaagent 补 JVM 指标 |
| Kubernetes 基础 | KSM 2.2.x、kubelet 内置 cadvisor、etcd 3.5、kube-apiserver 1.23、CoreDNS 1.7/1.8、argo-rollouts、ArgoCD（application-controller / server / repo-server metrics） |
| Prometheus | 规则文件兼容 Prometheus 2.x / VictoriaMetrics |
| Alertmanager | 邮件 / 企业微信 / 钉钉 webhook 路由 |

> 注意：exporter 升级大版本后部分指标命名会变化，届时需要同步调整仪表盘查询。

## 已知说明

- **RabbitMQ Head Message Age**：`rabbitmq_queue_head_message_timestamp` 为条件性指标，仅当生产者发布消息时设置了 AMQP `timestamp` 属性且队首消息处于 paged-in 状态时才会出现。经实测本环境业务消息均未设置该属性（管理接口对所有队列返回 null，exporter v0.29.0 与 classic 队列类型已排除其他原因），故优化版未包含该面板。若将来业务设置该属性，可按此指标自行补充。
- **Consumer Utilisation**：RabbitMQ 仅在利用率低于 1（消费者被 prefetch/网络限流）时上报该指标，面板无数据即消费健康，出现数据说明消费能力已跟不上。
- **同名集群区分**：Elasticsearch 优化版的变量链以 Exporter 实例为先选条件，面板查询均同时带 `instance` 过滤，避免同名集群互相串数据。
- **K8s 标签口径**：经 KSM 联邦后命名空间/节点标签为 `exported_namespace` / `exported_node`；cadvisor 与 kubelet 侧指标用 `kubernetes_io_hostname` 标识节点，集群通过 `job=~"$kubernetes_cluster.*-cadvisor$|.*-node$"` 区分。KSM 的 Pod 指标自身不带节点标签，节点过滤通过 `and on (exported_namespace, pod) kube_pod_info{exported_node=~"^$Node$"}` 关联。
- **K8s 面板数据起点**：下钻面板（如「指定 Pod 详情」）只展示对象存在期间的数据，起点为 Pod/容器创建时间而非仪表盘时间选择器的起点；排查历史问题请选择覆盖对象生命周期的 Pod。
- **容器级网络**：cadvisor 的 `container_network_*` 仅提供 Pod 沙箱维度（`container="POD"`）与 Pod 汇总，没有容器级网络序列，因此容器级排行提供 CPU/内存/磁盘读写与 IOPS，容器网络见 Pod 级 Top10。
- **ArgoCD 指标口径**：数据来自 `argocd-server-metrics` 抓取任务下的三个组件（application-controller、server、repo-server）。`go_*`、`grpc_server_*`、`process_*`、`workqueue_*` 是通用指标名，查询必须带 `job="argocd-server-metrics"` 限定，避免混入其它抓取目标。`argocd_app_info` 的 `namespace` 标签恒为 ArgoCD 安装命名空间，不能用于按业务命名空间过滤，仪表盘改用 `project` 与 `dest_server`（目标集群）两个变量。
- **ArgoCD 空态语义**：「gRPC 非 OK 响应」「Redis 失败请求」「K8s API 非 2xx 响应」只显示窗口内大于 0 的序列，空图表示当前健康；源模板的「Operations」依赖 `operation` 标签，该标签在当前版本恒为空（面板恒为 0），优化版已替换为「异常应用」。
- **ArgoCD Redis 耗时**：新版（`argocd_redis_request_duration_seconds_*`）与旧版（`argocd_redis_request_duration_*`）指标由不同组件上报，P95 面板用两条线分别呈现，不能直接相加或合并。
- **Nginx Upstream 指标口径**：数据来自 nginx 侧健康检查脚本经 Pushgateway 上报的 `nginx_upstream_status`，值恒为 1，在线与否由 `status` 标签表达（判据 `up|UP|1|true`）。该指标只有健康状态，没有权重、响应时间、请求数与连接数，性能分析请用 Nginx VTS 仪表盘。
- **Nginx Upstream 采集健康**：Pushgateway 会永久保留最后一次推送的值、抓取时只刷新样本时间戳，因此 `nginx_upstream_status` 永远「看起来是新的」。判断推送是否还在继续必须看 Pushgateway 自带的 `push_time_seconds`（面板「推送组最后推送距今」），它单调上涨即代表脚本已停止上报。
- **Nginx Upstream 离线语义**：后端离线由主动健康检查判定，存在 1~2 个检查周期延迟，短时抖动表现为离线数尖峰后回落；整条 upstream 被删除或实例改名不会记为离线，而是让「后端总数」下降，需与采集健康分区结合判断。
- **Kafka 汇总口径**：`kafka_server_brokertopicmetrics_*` 同时提供 `topic=""` 的 All Topics 聚合序列与逐 Topic 序列，集群汇总卡片与按 Broker 曲线只用 `topic=""`，按 Topic 下钻才用 `topic!=""`——两者相加会把数值放大一倍。`GlobalTopicCount` 由每个 Broker 都上报，Topic 总数必须取 `max` 而非 `sum`。
- **Kafka 延迟指标**：`kafka_network_requestmetrics_totaltimems` 本身就是分位毫秒值（由 `quantile` 标签区分 p99/p999），直接用取值，不能再套 `rate`/`increase`；不同 jmx_exporter 配置下该标签可能写作 `0.99` 或 `99`，故查询用 `quantile=~"0.99|99"` 兼容两种形式。
- **Kafka 线程饱和度**：`networkprocessoravgidlepercent` 与 `requesthandleravgidlepercent` 原值是 0~1 的比例，乘 100 才是百分比，面板展示的是「占用率 = 100 − 空闲率×100」，越低越空闲。请求处理线程饱和通常先由磁盘写入跟不上引起，不要一味加线程数。
- **Kafka 空态语义**：失败请求、ISR 抖动、ZooKeeper 断连等面板在健康时为空（或恒为 0）属正常，不要当成采集故障；KRaft 模式集群没有 ZooKeeper 相关序列，对应面板为空亦属正常。
- **Kafka 覆盖范围**：本仪表盘覆盖 Broker 侧与 JVM 侧指标，不含消费组积压（consumer group lag）——该指标由 Kafka Exporter 的 `kafka_consumergroup_lag` 提供，jmx_exporter 无此数据，需在消费侧或 kminion 单独采集。
- **SQL Server 计数器与 gauge 区分**：`mssql_deadlocks`、`mssql_user_errors`、`mssql_kill_connection_errors`、`mssql_batch_requests`、`mssql_lock_waits`、`mssql_page_fault_count`、`mssql_log_growths`、`mssql_io_stall_total_seconds`、`mssql_wait_stats` 均为自启动累计值，必须用 `rate`/`increase`；而 `mssql_connections`、`mssql_blocking_sessions`、`mssql_long_running_queries`、`mssql_resident_memory_bytes`、`mssql_virtual_memory_bytes`、`mssql_page_life_expectancy_seconds`、`mssql_buffer_cache_hit_ratio_percent`、`mssql_log_space_used_percent`、`mssql_tempdb_space_used_mb`、`mssql_cpu_usage_percent`、`mssql_memory_utilization_percentage` 是瞬时 gauges，直接取值，套 rate 会得到无意义的锯齿。
- **SQL Server 等待类型口径**：`mssql_wait_stats{stat="wait_time_ms"}` 与 `{stat="waiting_tasks_count"}` 是同一 `wait_type` 下的两个维度，平均等待时长按 `increase(等待毫秒)/increase(等待次数)` 计算，累计影响面按 `increase(等待毫秒)` 排序；两者都只保留窗口内确实发生等待的类型（分母为 0 的类型剔除），避免空序列干扰 TopN 排名。
- **SQL Server 内存与页文件**：`mssql_os_memory` 与 `mssql_os_page_file` 都按 `state` 标签拆成 `used`/`available` 两条序列，总量需在面板内另行求和，指标本身不提供 total；`mssql_memory_utilization_percentage` 是工作集占已提交内存的比例，不是整机内存使用率。
- **SQL Server 多实例**：`instance` 是 exporter 端点（IP:端口），`db` 是库名，同名库可能出现在多台实例上；因此按库的聚合与明细表一律同时按 `instance` 分组，避免跨实例求和。
- **ZooKeeper 副本一致性**：ZK 各副本的数据完全相同，`znode_count`、`ephemerals_count`、`global_sessions`、`approximate_data_size` 等集群级指标在每台机器上都上报同一个值，汇总必须取 `max`，取 `sum` 会按副本数翻倍；`num_alive_connections`、`watch_count` 是各节点自己的量，才需要按实例求和或对比。
- **ZooKeeper 计数器与 gauge**：`packets_sent`/`packets_received` 在 ZK 指标端点上虽被声明为 gauge，语义却是自启动起的累计量，必须用 `rate`；`snap_count`、`commit_count`、`proposal_count`、`dead_watchers_*`、`outstanding_changes_*`、`request_commit_queued`、`tls_handshake_exceeded`、`auth_failed_count` 等同样是累计量。反观 `watch_count`、`outstanding_requests`、`max_latency`、`open_file_descriptor_count`、`last_proposal_size` 等是瞬时值，套 `rate` 只会得到无意义锯齿。
- **ZooKeeper 耗时指标**：`fsynctime`、`readlatency`、`proposal_latency`、`*_queue_time_ms` 等由 Summary 型数据导出，同时提供 `_sum` 与 `_count`，单次平均耗时 = `rate(_sum) / rate(_count)`（窗口 10m）；直接对 `_sum` 取 rate 得到的是「每秒累计毫秒」（吞吐×耗时），数值含义不同。
- **ZooKeeper 空闲时段不能整段丢数据**：这类 Summary 指标在业务空闲时窗口内没有新观测，`rate(_count)` 为 0。若把分母写成 `(rate(_count) > 0)` 这样的比较过滤，PromQL 会**把该时间点整条序列丢弃**，低频面板（选主耗时、fsync、快照、Observer Master、空池等待等）就会在空闲期整段 No data，看起来像采集故障。正确写法是 `clamp_min(rate(_count), 1e-9)`，分子为 0 时输出 0，曲线保持连续；判断「有没有选主」应看曲线是否抬升，而不是看有没有线。
- **ZooKeeper 命名空间维度**：`read_per_namespace` / `write_per_namespace` 是 SummarySet，标签名固定为 `key`（不是 `namespace`），值为 ZK 内的命名空间路径；同名标签在 `learner_handler_qp_size`（值却是 learner 地址）里也复用，聚合时按 `key` 分组即可。**注意观测值是字节数而非操作次数**：`DataTree` 里读记响应体 `totalBytes`、写记 `path.length() + 数据长度`，所以面板单位是 Bps / 字节，按它排名得到的是「哪个命名空间数据量最大」，不能当作 QPS。
- **ZooKeeper 选主类指标**：`learners`、`synced_followers`、`synced_observers`、`pending_syncs`、`leader_uptime`、`last/max/min_proposal_size` 只有 Leader 上报，非 Leader 为空属正常；`looking_count` 出现尖峰且 `leader_uptime` 归零即代表发生了一次重新选主。
- **ZooKeeper JVM 指标来源**：ZK 3.6+ 的 `PrometheusMetricsProvider` 默认 `exportJvmInfo=true`，会在同一端点暴露 `jvm_memory_*`、`jvm_gc_*`、`jvm_threads_*` 等；若关闭该开关或使用旧版本，需要同机 jmx_exporter javaagent 补齐。优化版对两条链路取并集（`or` 后 `max` 去重），并分别绑定「实例」与「JVM 实例」变量，任一侧有数据即可出图。

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
    ├── elasticsearch/
    │   ├── elasticsearch.json                  # 社区原版
    │   └── elasticsearch-optimized.json        # 优化版（推荐）
    ├── esxi/
    │   ├── esxi.json                           # 社区原版
    │   └── esxi-optimized.json                 # 优化版（推荐）
    ├── iis/
    │   ├── IIS.json                            # 社区原版
    │   └── iis-optimized.json                  # 优化版（推荐）
    ├── kafka/
    │   ├── kafka.json                          # 社区原版
    │   └── kafka-optimized.json                # 优化版（推荐）
    ├── linux/
    │   ├── Linux.json                          # 社区原版
    │   └── linux-optimized.json                # 优化版（推荐）
    ├── lvs/
    │   ├── lvs.json                            # 社区原版
    │   └── lvs-optimized.json                  # 优化版（推荐）
    ├── mysql/
    │   ├── mysql.json                          # 社区原版
    │   └── mysql-optimized.json                # 优化版（推荐）
    ├── nginx/
    │   ├── Nginx.json                          # 社区原版
    │   └── nginx-optimized.json                # 优化版
    ├── nginx_upstream/
    │   ├── nginx_upstream.json                 # 社区原版
    │   └── nginx_upstream-optimized.json       # 优化版（推荐）（推荐）
    ├── rabbitmq/
    │   ├── RabbitMQ.json                       # 社区原版
    │   └── rabbitmq-optimized.json             # 优化版（推荐）
    ├── redis/
    │   ├── redis.json                          # 社区原版
    │   └── redis-optimized.json                # 优化版（推荐）
    ├── sqlserver/
    │   ├── sqlserver.json                      # 社区原版
    │   └── sqlserver-optimized.json            # 优化版（推荐）
    ├── windows/
    │   ├── Windows.json                        # 社区原版（windows_exporter）
    │   ├── windows-optimized.json              # 优化版（windows_exporter）
    │   ├── Windows2003.json                    # 源表参考
    │   └── windows-wmi-optimized.json          # 优化版（wmi_exporter 0.7/0.8）
    ├── zookeeper/
    │   ├── zookeeper.json                      # 社区原版
    │   └── zookeeper-optimized.json            # 优化版（推荐）
    └── k8s/
        ├── argocd/
        │   ├── argocd.json                      # 社区原版
        │   └── argocd-optimized.json            # 优化版（推荐）
        ├── argo-rollouts/
        │   └── argo-rollouts-optimized.json
        ├── coredns/
        │   ├── CoreDNS.json                    # 社区原版
        │   └── coredns-optimized.json
        ├── kube-state-metrics/
        │   ├── kube-state-metrics-01-cluster-overview-optimized.json
        │   ├── kube-state-metrics-02-node-optimized.json
        │   ├── kube-state-metrics-03-workload-optimized.json
        │   └── kube-state-metrics-04-storage-optimized.json
        ├── kubernetes-api-server/
        │   ├── Kubernetes API Server.json      # 社区原版
        │   └── kubernetes-api-server-optimized.json
        ├── kubernetes-cluster/
        │   ├── Kubernetes Cluster.json         # 社区原版
        │   └── kubernetes-cluster-optimized.json
        ├── kubernetes-etcd/
        │   ├── Kubernetes Etcd.json            # 社区原版
        │   └── kubernetes-etcd-optimized.json
        ├── kubernetes-nodes/
        │   ├── Kubernetes Nodes.json           # 社区原版
        │   └── kubernetes-nodes-optimized.json
        └── kubernetes-pods/
            ├── Kubernetes Pods.json            # 社区原版
            └── kubernetes-pods-optimized.json
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
