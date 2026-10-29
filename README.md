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
| `grafana/kafka/` | Kafka (Optimized) | `kafka-opt-11962` | 59 | 基于 jmx_exporter，集群健康/吞吐/延迟/请求异常/消费组/JVM 分区（8 卡片 + 42 图 + 2 明细表） |
| `grafana/sqlserver/` | SQL Server (Optimized) | `sqlserver-opt` | 34 | 基于 mssql_exporter，连接阻塞/内存缓存/吞吐等待/日志明细分区（8 卡片 + 20 图 + 数据库明细表） |
| `grafana/zookeeper/` | ZooKeeper (Optimized) | `zookeeper-opt-10465` | 83 | ZK 自带指标端点 + JVM，九大分区、三列紧凑布局（8 卡片 + 57 图 + 1 明细表），JVM 区紧随选主区 |
| `grafana/openobserve/` | OpenObserve (Optimized) | `openobserve-opt` | 58 | 基于 OpenObserve 自带 `zo_*` 指标端点，概览 / 元数据与存储 / Ingest 写入 / 压缩与合并 / 查询 / 接口与运行时六大分区（8 卡片 + 4 规模卡 + 39 图 + 1 明细表），指标口径已逐条对照源码与线上 `/metrics` 核实 |
| `grafana/softraid/` | SoftRAID (Optimized) | `softraid-opt-155` | 20 | 基于 raid_monitor → Pushgateway 上报的 `raid_status`，阵列概览 / 状态趋势 / 采集健康 / 阵列明细四分区（8 卡片 + 6 图 + 2 表），含 Pushgateway 判活与状态码映射着色 |
| `grafana/idc_temp/` | IDC Temp (Optimized) | `idc-temp-opt` | 26 | 基于 snmp_exporter `dell_idrac` 模块采集 iDRAC 温度探针，概览 / 冷通道进风 / 热通道与温差 / 部件温度与变化率 / 探针明细五分区（8 卡片 + 12 图 + 明细表），图例含机器名/iDRAC 带外地址/hostIP |
| `grafana/traffic_network/` | Traffic_Network(Optimized) | `traffic-network-opt` | 29 | 基于 snmp_exporter IF-MIB 的网络设备（交换机/防火墙）接口流量，概览 / 设备流量 / 接口流量 / 接口健康 / 明细五分区（8 卡片 + 14 图 + 2 表），64 位计数器优先、带宽利用率与错误丢弃速率齐备 |
| `grafana/traffic_host/` | Traffic_Host(Optimized) | `traffic-host-opt` | 21 | 基于 node_exporter / windows_exporter 的主机网卡流量，流量概览 / 全部主机 / 分组对比 / 主机明细四分区（8 卡片 + 8 图 + 1 明细表） |
| `grafana/minio/` | MinIO (Optimized) | `minio-opt-15305` | 53 | 基于 MinIO v2 指标端点（cluster / node / bucket / resource），概览 / 容量与磁盘 / S3 请求 / 流量 / Bucket 与对象 / 节点资源 / 自愈与扫描 / 采集信息八分区（8 卡片 + 35 图 + 2 表）；采集过滤精确匹配 `job` 而非前缀正则，避免四端点同指标重复累加 |
| `grafana/xenserver/` | XenServer(Optimized) | `_kRTLECvz` | 41 | 基于 xenserver 指标端点，总览 / CPU 与负载 / 内存与交换 / 磁盘 / 文件系统 / 网络六分区（9 卡片 + 3 singlestat + 21 图 + 2 表），由社区面板改造 |
| `grafana/blackbox/` | Blackbox (Optimized) | `blackbox-opt-9719` | 22 | 基于 blackbox_exporter（icmp / http / tcp），拨测概览 / 状态明细 / 耗时分析 / 对比分析四分区（6 卡片 + 11 图 + 1 表） |
| `grafana/docker/` | Docker (Optimized) | `docker-opt-8321` | 22 | 基于 cadvisor，概览 / 容器清单 / 容器资源 / 网络 / 磁盘 IO 五分区（8 卡片 + 8 图 + 1 表） |

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
| iDRAC（机房温度） | [prometheus/snmp_exporter](https://github.com/prometheus/snmp_exporter) + `dell_idrac` 模块，经 SNMP v2c 抓取 Dell iDRAC 的 `IDRAC-MIB-SMIv2` 温度探针（`temperatureProbeReading`），模块需 walk 到该 OID 表 |
| 网络设备（交换机/防火墙流量） | [prometheus/snmp_exporter](https://github.com/prometheus/snmp_exporter)，模块需覆盖 `IF-MIB::interfaces` 与 `IF-MIB::ifXTable`（64 位 `ifHCInOctets`/`ifHCOutOctets`、`ifHighSpeed` 在 ifXTable 中），接口名 `ifDescr` 需在模块 walk 或 lookups 范围内 |
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
- **OpenObserve 计数器与 gauge**：`zo_ingest_bytes`、`zo_ingest_records`、`zo_storage_read/write_bytes`、`zo_storage_read/write_requests`、`zo_storage_time`、`zo_compact_merged_bytes/files`、`zo_compact_used_time`、`zo_query_disk_cache_hit/miss_count`、`zo_grpc_incoming_requests`、`zo_http_incoming_requests`、`zo_stream_stats_scan_total`、`zo_file_list_id_select_count` 都是自启动起的累计值，必须用 `rate`；`zo_meta_*`、`zo_storage_records/files/original_bytes`、`zo_ingest_memtable_*`、`zo_ingest_parquet_files`、`zo_ingest_wal_used_bytes`、`zo_query_*_nums`、`zo_compact_pending_jobs`、`zo_tokio_runtime_tasks` 是瞬时 gauge，套 `rate` 只会得到无意义锯齿。
- **OpenObserve 语义易错三处**：`process_cpu_seconds_total` 是进程累计 **CPU 时间**而非运行时间（应 `rate` 后按「核」展示）；`zo_ingest_records` 是**记录条数**不是字节（平均记录长度需由 `zo_ingest_bytes / zo_ingest_records` 派生）；`zo_stream_stats_last_scan_timestamp` 是**微秒级 Unix 时间戳**，必须 `/1e6` 再与 `time()` 相减，直接写 `time() - 该值` 会得到天文数字或负数。
- **OpenObserve 直方图分组**：`zo_file_access_time`、`zo_http/grpc_response_time`、`zo_ingest_wal_lock_time`、`zo_ingest_memtable_lock_time`、`zo_stream_stats_scan_duration_seconds`、`zo_query_parquet_cache_ratio(_node)` 都是原生直方图。按 `organization`/`scan_type` 分组时必须写成 `sum by (organization, le)`——漏掉 `le` 会让 `histogram_quantile` 拿到互不相交的桶，直接返回空结果（面板整块 No data）。
- **OpenObserve 元数据口径**：`zo_meta_num_nodes/organizations/streams/users_total`、`zo_meta_storage_keys/bytes` 以及 `zo_storage_records/files/original_bytes` 属**组织/集群级快照**，多实例会各自上报同一份全量值，聚合一律 `max` 去重（`sum` 会按实例数成倍放大）。若部署形态确实是按实例分片统计，需改用 `sum` 才是集群总量——仪表盘描述里已注明这一前提。- **OpenObserve 分位数面板的空态（重要）**：`histogram_quantile` 在窗口内**没有新观测**时，各 `le` 桶的 `rate()` 增量全为 0，函数返回 **NaN**；Grafana 对 NaN 序列不画点，面板就显示 **No data**（实测：NaN 序列在 `/api/v1/query_range` 里返回 **0 个点**）。这属于「近期无流量」的正常空态，不是采集故障。**裸 `or vector(0)` 救不了**——NaN 是一个值而不是缺失序列，`or` 不会触发；正确写法是先用比较过滤把 NaN 滤掉再兜底：`(histogram_quantile(...) >= 0) or vector(0)`（实测：空闲时返回连续 0 值曲线，有流量时数值与未加壳写法完全一致）。本盘 20 条分位数查询已全部加此防护。
- **OpenObserve 指标类型易错两处**：`zo_file_list_id_select_count` 与 `zo_service_streams_queue_dropped` 在源码里是 **IntGaugeVec（`.set()` 覆盖写）**，不是计数器，套 `rate()` 只会得到 0 或锯齿；而 `zo_query_disk_cache_hit_count` / `zo_query_disk_cache_miss_count` 才是真正的 `_count` 计数器（需要 rate）。判断依据看源码是 `.inc()/.set()` 还是 `IntCounterVec`，不要只看名字里有没有 count。
- **OpenObserve 耗时指标单位不统一（易误标）**：`zo_ingest_wal_lock_time` 与 `zo_ingest_memtable_lock_time` 用 `start.elapsed().as_millis()` 观测 → 单位是**毫秒**（桶边界 0.2~2000）；`zo_http_response_time` / `zo_grpc_response_time` / `zo_storage_time` / `zo_compact_used_time` 用 `as_secs_f64()` → 单位是**秒**。把毫秒标成秒会让「20ms 的锁等待」显示成 20s，直接误判故障等级。
- **OpenObserve `zo_file_access_time` 不是文件读取耗时**：其 HELP 原文是 "Histogram showing query counts within time windows from 1h to 1week"，桶边界为 1/2/3/6/12/24/48/96/168，**单位是小时**，衡量的是「用户一次查询覆盖多长的数据」（实测均值 161.44h 落在 96~168 桶之间）。原模板把它命名为「查询次数」并标 `unit=s` 均属误读；本盘改名为「查询时间窗口分布」并按 `h` 单位展示。Grafana 7.1.4 有 `h` 单位，没有 `hours` 这个 id。
- **OpenObserve `zo_tokio_runtime_tasks` 的 -1 是哨兵值**：该 gauge 带 `runtime` + `metric_type` 两个标签（metric_type ∈ workers / alive_tasks / global_queue_depth / blocking_queue_depth / io_driver_fd_registered）。源码 `src/node/src/runtime_metrics.rs` 在**未以 `--cfg tokio_unstable` 编译**时只 `set(workers = -1)`，表示「拿不到运行时内部计数」——所以面板上看到 `http=-1 / job=-1` 并非真的只有 -1 个 worker，也不能当容量信号；启用 tokio_unstable 重新编译后才会出现其余 metric_type。另注意 `sum by (runtime)` 会把不同 metric_type 混加，必须 `by (runtime, metric_type)`。
- **OpenObserve `zo_query_running_nums` 有上游记账缺陷**：`search_service/src/cluster/flight.rs` 里只有 `.inc()`，配对的 `.dec()` 在 work_group 的异步守卫中，flight 路径会漏减，导致该 gauge 单调累积（实测已累积到 4142 且空闲不回落）。因此它不能当作「当前运行中查询数」，实时并发应看 `zo_query_pending_nums` 与请求速率。
- **OpenObserve 内存表字节是两套独立记账**：`zo_ingest_memtable_bytes` 累计 `data_json_size`、`zo_ingest_memtable_arrow_bytes` 累计 `data_arrow_size`（源码 `partition.rs` 分别 `.add()`），二者**不是「总量与其子集」**的关系，Arrow 曲线高于 JSON 曲线属正常，不要读成占比。
- **OpenObserve `role` 是 const label**：`create_const_labels()` 给所有 `zo_*` 注入 `cluster` / `instance` / `role` 三个常量标签，单机部署下 `role` 恒为 `all`、`instance` 是容器 ID（如 `6c80a69220f5`）而非 `IP:端口`。因此 `by (role)` 只有一条无区分度的线，聚合应按 `organization` / `stream_type` / `instance` 拆分。
- **【制作流程坑】不要用 PowerShell 双引号 here-string 改 Grafana JSON 里的 `$变量`**：在 `pwsh -c "...@\"...\"@"` 里，双引号内的 `$job` / `$instance` 会被 PowerShell 当作自身变量插值成空字符串，于是 `{job=~\"$job\"}` 落盘后变成 `{job=~\"\"}`。这种空 matcher 不报错、不违反 PromQL 语法，但会**永远匹配不到任何序列**，面板表现为 No data，且极难从 JSON 里看出来（原文看上去只是引号里少了个词）。本轮就因此让「虚拟内存」与「Tokio 运行时」两张图变空。**正确做法**：改 JSON 一律用 Python 脚本 + `io.open()`/`json.dump` 读写，或用单引号 here-string；改完必须跑一次「空 matcher 扫描」门禁：`re.search(r'=~""', expr)`，命中即失败。同时建议对全部表达式做变量展开后的语法门禁，并断言每条 `zo_*`/`process_*` 查询都同时带 `job=~` 与 `instance=~`。

- **SoftRAID 属性全在标签上**：`raid_status` 的**值**只有状态码 0/1/2，而 `raid_level`、`raid_active_devices`、`raid_failed_devices`、`raid_spare_devices`、`raid_total_devices`、`raid_array_size`、`raid_create_time`、`raid_update_time`、`raid_rebuild_status` 全是**标签（字符串）**。PromQL 无法把标签转成数值，所以盘数与容量只能计数或按标签过滤（`raid_failed_devices=~"[1-9][0-9]*"`），重建进度（形如 `recovering 45%`）**算不出百分比**——要真实进度必须补 `node_md_blocks` / `node_md_blocks_synced`。
- **SoftRAID 判活不能用指标本身**：经 Pushgateway 上报的样本会永久保留，抓取只刷新样本时间戳，阵列状态序列永远「新鲜」，即使采集脚本已停摆数天、面板上的降级信息仍是最后一次推送的过期快照。判采集中断只能用 Pushgateway 自带的 `push_time_seconds`：`time() - max by (exported_job) (push_time_seconds{job="pushgateway", exported_job="raid_monitor"})`。
- **SoftRAID 的 job 标签被改名**：`prometheus.yml` 的 `pushgateway` job 未开 `honor_labels`，推送时的 `job`/`instance` 被改写为 `exported_job`/`exported_instance`，样本上的 `job` 恒为 `pushgateway`。过滤阵列必须写 `exported_job`，写成 `job=~"raid.*"` 会查不到任何序列。整机下线时 keepalived/脚本侧条目直接消失、序列随之消失，因此「阵列总数下降台阶」是发现阵列悄悄消失的唯一手段（降级计数看不见它）。
- **iDRAC 温度读数的单位与哨兵值**：`temperatureProbeReading` 的单位是**十分之一摄氏度**，需除以 10；探针不可用时 iDRAC 按 MIB 规范上报 **-128.0**（不同固件在原始值上表现为 -128 或 -1280）。因此所有查询都先按**原始值** `> -30` 过滤再除以 10，两种哨兵编码都能剔除；若只在除 10 之后过滤，-1280 会变成 -128℃、-128 会变成 -12.8℃，把纵轴压成一条平线并让真实曲线完全不可见。
- **iDRAC 探针位置是独立指标**：`temperatureProbeLocationName` 在 snmp_exporter 里是**单独的字符串指标**（值放在同名标签上），并不附着在读数序列上。要做「按位置筛选/显示位置」必须 `ignoring(temperatureProbeLocationName) group_right()` 把它 join 回读数——反过来用 `group_left()` 会因标签集不匹配而丢数据。join 之后才带出 `name`（机器名）、`ip`（业务网主机地址）、`instance`（iDRAC 带外地址，形如 `192.168.16.226:161`，即 `prometheus.yml` 中 `replacement: ${1}:161` 拼出的结果）。
- **iDRAC 不提供湿度**：服务器机箱内没有湿度传感器，`dell_idrac` 模块无论怎么改都取不到湿度。机房湿度、露点、精密空调（CRAC）、UPS/PDU、漏水与烟感属**动环层**数据，需另行接入机房级传感器（动环主机 SNMP、`modbus_exporter`）或改用 `redfish_exporter` / `ipmi_exporter` 直连带外口。
- **iDRAC 可选探针位置随机型变化**：常见为 `System Board Inlet Temp`、`System Board Exhaust Temp`、`CPU1 Temp`、`CPU2 Temp`，机型未提供的位置对应面板为空属正常，不是采集故障。进排风温差 ΔT 只统计同时上报进风与排风探针的机器，缺任一探针的机型不出现在该面板。
- **网络设备流量优先用 64 位计数器**：`ifInOctets`/`ifOutOctets` 是 32 位计数器，按 2^32 bit 回绕（100Mbps 约 5.7 分钟、1Gbps 约 34 秒、10Gbps 约 3.4 秒一圈），而 `rate()`/`irate()` 遇到值变小一律按「计数器重置」处理，结果**速率被严重低估且不报错**。仪表盘统一优先取 ifXTable 的 `ifHCInOctets`/`ifHCOutOctets`，仅当该接口没有 64 位序列时才回落 32 位版本。
- **`ifSpeed` 在万兆口失效**：它是 Gauge32，上限 4294967295（≈4.29Gbps），5G/10G/25G/40G/100G 链路上恒为该最大值。带宽利用率的分母因此优先取 `ifHighSpeed`（单位 Mbps），回落 `ifSpeed`，都缺失时按 1Mbps 兜底——这也是面板上可能出现 >100% 读数的原因（协商速率上报与实际不符，或聚合口只上报单成员速率）。
- **`ifDescr` 的两种暴露形态**：snmp_exporter 的官方 `if_mib` 模块自 v0.15.0 起把 `ifDescr`/`ifName`/`ifAlias` 作为 **lookups 标签**并入读数序列（metrics 段里 `ignore: true`，没有独立指标）；但自建模块若没写 lookups，就只能拿到 `ifIndex`，或 `ifDescr` 以「独立字符串指标」形式出现（值恒为 1、接口名放在同名标签上）。仪表盘的接口维度查询**两种形态都兼容**：先 `group_right()` join 独立 `ifDescr` 指标，再用 `unless` 补上「读数自带 ifDescr 标签」而未覆盖的接口——两者归一化为同一标签集，同一接口不会出现两条曲线。
- **接口名可能为空或带乱码**：华为 iBMC 等设备的 DisplayString 尾部有 NUL 填充，标签值形如 `CPU1 Core Rem\x00\x00`，界面看着正常但 PromQL 永远匹配不上；snmp_exporter v0.24.0 之前非法 UTF-8 还会直接让整个 target 抓取失败（之后替换为 `�`）。另外若设备未实现 `1.3.6.1.2.1.2.2.1.2`（仅提供 ifXTable），lookup 取不到值会让标签为空、图例只剩索引。因此本盘图例除接口名外还保留 ifIndex，明细表另有「索引」列用于区分同名接口。
- **多个计数器求和不能用 `or` 串联**：`rate()` 的输出**不带 `__name__`**，四个错误/丢弃计数器在同一接口上的标签集完全相同，`rate(A) or rate(B) or ...` 只会保留最左侧存在的那一项，其余三项被静默丢弃。正确写法是逐项各自 `or` 零基线（`ifOperStatus * 0`）再相加。
- **graph 面板无法用 mappings 显示枚举（7.1.4 限制）**：`fieldConfig.mappings` 只有 stat / table 系列面板会应用；graph 面板的图例走 `TimeSeries.formatValue()`（`public/app/core/time_series2.ts`），只用 `getValueFormat(unit)` 格式化数字，而 `graph.ts` 给 tooltip 构造 fieldConfig 时也只传 `decimals` 与 `links`。因此 `ifOperStatus` 这类 0~7 状态码**不能**靠面板配置显示成 up/down。本盘的做法是在查询里给序列打 `state` 标签：每支写成 `label_replace((状态) == N, "state", "名称", ...)`，再 `or` 合并，最后用 `group_right(state)` 关联一次接口名，图例写成 `{{instance}} · {{ifDescr}} · {{state}}`。要点：①`X == N` 返回**原值**（不是 1）且会过滤掉不匹配的序列，所以每接口仍只有一条曲线、曲线高度仍是原始状态码；②**必须补非标准值的兜底**——IF-MIB 只定义了 1~7，但真实设备会出现 0 / 8 / 255（接口刚创建未初始化、厂商私有扩展），只写 7 支精确匹配会让这些接口**整条曲线消失**（不是显示未知，而是没了），兜底用 `(状态) < 1 or (状态) > 7` 两支即可，**不要用 `unless` 重算 7 支**——实测那样会让 24h 查询直接超时；③图例只保留 current 一列并靠右，否则接口多时三列表格会把面板挤满、名称与状态被截断。代价是选择器按分支数重复求值（9 支约为单支的 9 倍，24h 实测 0.7s→6.8s），故该面板单独把 `interval` 放宽到 5m、`maxDataPoints` 压到 300。

- **`ifOperStatus` 状态轴必须 `fill=0`**：有填充时值为 1 的序列色块会连成一片，把掉到 2 的接口整条盖住；判断依据是曲线的台阶而不是「有没有线」。取值 1=up、2=down、3=testing、4=unknown、5=dormant、6=notPresent、7=lowerLayerDown。
- **长时间窗（24h）的性能设计**：网络设备盘一条查询要处理上百台设备数千个接口，24 小时档下很容易卡住。本版针对该场景做了五项处理：①**计数器回落的去重去掉重复 rate**——`rate()` 丢弃 `__name__` 后两个分支标签集相同，`A or B` 本身就是「左优先、左缺取右」，不必再写 `B unless on(..) A` 把 A 重算一遍；②**接口名双形态合并的右支改用裸选择器做存在性探测**，不再重算整个 join 分支；③**利用率/水位类先聚合再 join**，序列数从「每接口」降到 1 条；④**所有面板限制 `maxDataPoints`**（图 400 / 卡片 200 / 表 500）并设 `interval=2m`，避免按像素宽度拉回上千点；⑤**刷新周期 2m**。合计使表达式中的 `rate()` 求值次数下降 51%、`unless` 下降 78%，24h 档位下查询明显变快。若某台设备接口数极多仍嫌慢，优先把「接口」变量收窄到具体设备，TopN 调小。
- **`×8` 的位置必须在整个回落表达式之外**：PromQL 中 `*` 的优先级高于 `or`，写成 `A or (B unless C) * 8` 会被解析为 `A or ((B unless C) * 8)`——**只有回落分支乘了 8，主路径（64 位计数器）漏乘**，于是同一张图里两种接口的数值差 8 倍。正确写法是 `(A or B) * 8` 或把 `sum()` 提到外面再乘。

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
    ├── idc_temp/
    │   ├── idc_temp.json                       # 社区原版
    │   └── idc_temp-optimized.json             # 优化版（推荐）
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
    ├── openobserve/
    │   ├── openobserve.json                    # 社区原版
    │   └── openobserve-optimized.json          # 优化版（推荐）
    ├── rabbitmq/
    │   ├── RabbitMQ.json                       # 社区原版
    │   └── rabbitmq-optimized.json             # 优化版（推荐）
    ├── redis/
    │   ├── redis.json                          # 社区原版
    │   └── redis-optimized.json                # 优化版（推荐）
    ├── sqlserver/
    │   ├── sqlserver.json                      # 社区原版
    │   └── sqlserver-optimized.json            # 优化版（推荐）
    ├── softraid/
    │   ├── softraid.json                       # 社区原版
    │   └── softraid-optimized.json             # 优化版（推荐）
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
