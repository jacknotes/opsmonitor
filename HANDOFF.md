# opsmonitor 监控面板项目交接文档

> 交接日期：2026-10-23。本文档供后续接手的大模型/工程师快速上手，覆盖环境、数据链路、硬性规则、验证方法、已踩坑与当前状态。更细粒度的历史经验见记忆文件 project_memory.md（本机 Trae 记忆系统）。

## 1. 项目概况

为多集群 Kubernetes + 传统主机环境制作 Grafana 监控 dashboard，全部交付物为 JSON 文件，按「原模板 + 优化版」成对存放。**所有优化修改仅作用于 -optimized / 监控 命名的文件，原模板文件一律不动。**

- 工作区：`f:\project\opsmonitor`（prometheus 配置、grafana dashboard）
- Grafana **7.1.4**（老版本，有大量兼容性限制，见 §4）
- 数据源：中心 VictoriaMetrics（Prometheus remote_write 汇入），Grafana 查询指向中心
- 三个 K8s 集群：`kubernetes`（生产）、`prepro-kubernetes`、`test-kubernetes`
- 导出器版本：node_exporter 1.8.2、windows_exporter 多版本（wmi_exporter 0.7.0/0.8.3、windows_exporter 0.20.0）、kube-state-metrics 2.2.4、etcd 3.5.1、apiserver 1.23、CoreDNS 1.8.3（prepro）与 1.7.0（test 的 node-local-dns）、argo-rollouts 老版本

## 2. 数据链路（核心背景）

### 2.1 中心 VM 的两类数据来源
1. **中心直抓**（job 名内嵌集群名，带 `kubernetes_cluster="<集群名>"` 标签）：
   - `kubernetes/<集群名>-node`（node_exporter）、`<集群名>-node-cadvisor`（cadvisor）、`<集群名>-apiserver`、`<集群名>-etcd`
2. **联邦拉取**（in-cluster Prometheus → 中心）：KSM、CoreDNS、argo-rollouts、ArgoCD 等指标，经 `/federate?match[]={job="kubernetes-service-endpoints"}` 拉回；ArgoCD 单独以 `match[]={job="argocd-server-metrics"}` 拉取（该 job 在 prod 内包含 application-controller:8082、server:8083、repo-server:8084 三个目标，argocd_* 全量指标均由它们上报）。**honor_labels 冲突导致标签改名：原生 `namespace`→`exported_namespace`、原生 `node`→`exported_node`，`namespace` 恒为 kube-system、`node` 恒为联邦目标主机、`instance`=KSM Pod IP（随调度漂移）**。K8s 节点过滤必须用 `exported_node`，命名空间用 `exported_namespace`，区分集群用 `kubernetes_cluster`。ArgoCD 的 `argocd_app_info` 自身 namespace 恒为安装命名空间（argocd），不受该改名影响，但按业务命名空间过滤无意义。

### 2.2 验证路径（中心端有认证不可直查）
- prepro 集群内 Prometheus 免认证：`http://prepro-prometheus.k8s.hs.com`（`/api/v1/query`、`/targets`）
- 指标 dump 文件（`c:\Users\0799\.trae-cn\attachments\` 下 *.metrics）用于中心端指标名比对
- 集群内无 node_exporter 指标（kubernetes-nodes job 为 kubelet 端点），node 指标只能靠 dump 比对

## 3. 硬性规则（用户约束，必须遵守）

1. 原模板文件不动，只改优化版文件
2. **每个面板必须带中文 description**（指标含义/异常逻辑）
3. 布局紧凑：避免满宽长表格纵向堆叠，多表并排 w12×2 等宽；首行 stat 卡片 w3×8 h5
4. graph 图例配置为面板下方表格模式，显示 Min/Max/Current
5. 不得要求用户恢复 prometheus.yml 中被主动丢弃的高基数指标
6. Windows 面板需兼容新旧多版本采集器（查询带新旧指标名 or 回退）；wmi 与 windows 版分别独立文件
7. 非用户手动选择的变量设为 hide=2，但**隐藏变量的 current 不能写死猜测值**（会导致全板无数据且用户无法纠正）——`current: null` + `refresh: 2` 自动解析
8. 文件命名（2026-10-23 起）：目录全小写——顶层产品目录小写（`iis`/`linux`/`nginx`/`windows`/`mysql`/`redis`/`rabbitmq`/`elasticsearch`/`k8s`），k8s 子目录 kebab-case（`argo-rollouts`/`argocd`/`coredns`/`kube-state-metrics`/`kubernetes-api-server`/`kubernetes-cluster`/`kubernetes-etcd`/`kubernetes-nodes`/`kubernetes-pods`/`metrics`）；优化版统一 `<对象>-optimized.json`（如 `kubernetes-cluster-optimized.json`、`kube-state-metrics-01-cluster-overview-optimized.json`）；原版模板保持原名不动

## 4. Grafana 7.1.4 兼容性要点（最重要）

- 支持面板类型：graph / stat / table / table-old / singlestat；**不支持 timeseries**（报 Panel plugin not found）
- 模板变量 query 必须是**纯字符串 + 补充 definition 字段**（对象格式报 `e.replace is not a function`）
- **多查询单表唯一可靠做法 = table-old + `"transform": "table"`**：所有查询 instant + format=table + legendFormat 留空，聚合到完全相同的标签签名；列名固定 `Value #<refId>`，在 styles 里 pattern+alias 改中文列名；新 table 的 merge 转换不可靠
- table-old 列顺序跟随 refId 顺序（A→B→…），调整列序须同步 targets 与 styles
- table-old styles 着色：**colors 语义 =「值低于 thresholds[0] 用第1色、之间第2色、高于 thresholds[1] 第3色」**，thresholds 必须是数字/字符串数组（如 `["0.5","1"]`）；写成新版 fieldConfig 对象格式 `[{color,level}]` 旧前端解析失败导致全列错色
- 新 table（单查询）：organize 转换 renameByName/excludeByName（单查询值列叫 `Value`，多查询才叫 `Value #<refId>`）；排序用 `options.sortBy`（sortBy **转换**引用重命名列会令整链失败、面板全黑）
- **stat 面板要有迷你趋势线：targets 必须 `format="time_series"` + `instant=false`**（instant 单点数据 graphMode=area 不渲染 sparkline）
- 变量消除打开卡顿：`current: null` + `refresh: 2` + `includeAll: true` + `allValue: ".*"`（导入后保存被写回 `$__all` 属正常）

## 5. PromQL 指标坑速查

- **KSM 0/1 矩阵**：`kube_*_status_condition` 等每实体×每状态各一条序列。`kube_node_status_condition` 标签是 (node, condition, **status**)，status∈true/false/unknown——必须加 `status="true"` 且不过滤 `==1`，直接 avg 得干净 0/1（不加 status 过滤 avg 得 1/3 错误值；只 ==1 过滤则健康节点整列空白）。`kube_node_spec_unschedulable` 是单值布尔直接 avg
- **count 类条件查询**：条件全不满足返回空序列而非 0，stat 必须补 `or vector(0)`；`kube_pod_status_phase` 等 sum 按 phase 不受矩阵影响无需兜底
- **通用指标名**（process_*/grpc_server_*/up）必须加 `job=~".*etcd.*"` 之类限定，否则混入全 VM 数据
- **VictoriaMetrics 下 group_right 多指标 join 不可靠**（表格列空）；group_left 相对可靠。Pod 名显示方案：`label_replace(instance 提取 IP) + group_left join kube_pod_info`，右侧必须限定目标 Pod 集合防 many-to-many
- **异常 Pod 清单**：未就绪列用 for:5m 持续门槛（`min_over_time(kube_pod_status_ready{condition="false"}[5m])==1 and on(...) (time()-kube_pod_start_time>=300) and on(...) kube_pod_status_phase{phase="Running"}==1`）；只列异常项的表格健康时显示 No data 属正确，空态语义写进标题/描述
- **用户偏好：TopN 用所选时间窗增量 `increase(...[$__range])` + topk**，不用累计值排序；倾向老版 graph 阶梯折线（非表格）；折线 TopN 取 10
- ksm 2.2.4 HPA 指标实名 `kube_horizontalpodautoscaler_*`（kube_hpa_* 是 v1.x 旧名）；排查无数据先确认采集端真实指标名再查链路
- CoreDNS 1.8.x：`coredns_cache_misses_total` 无 type 标签（只能做总命中率单线）；响应码只显示出现过的 rcode
- wmi_exporter 0.8.3（172.168.2.111）原生磁盘容量损坏，所有磁盘容量查询写「custom_wmi_* 优先 or 原生回退」；0.7.0 主机无 custom_wmi
- etcd 3.5.1：`etcd_debugging_mvcc_db_total_size_in_bytes` 与 `etcd_network_peer_sent_failures_total` **不存在**（前者用 `etcd_mvcc_db_total_size_in_bytes`，后者由心跳失败覆盖）；peer RTT 标签 `To`、received_bytes 标签 `From`；quota 默认 8GiB；slow_*/heartbeat_send_failures 是累计计数器需 increase
- apiserver 1.23：Watch 事件实名 `apiserver_watch_events_total`；客户端证书到期只有 histogram（剩余天数=histogram_quantile(0.05,...)/86400）；webhook 拒绝标签是 `name` 非 webhook
- 表格颜色阈值顺序须与源文件语义一致，防低值标红高值标绿反转；基于 100-速率 的使用率公式加 `clamp_min(..., 0)`
- 【LVS / node_exporter ipvs collector】节点级计数器（node_ipvs_connections_total、incoming/outgoing_bytes/packets_total）只有节点维度，服务级维度仅三个 gauge：backend_connections_active/inactive/weight（标签 ip/local_address/local_port/proto/remote_address/remote_port）。面板只用 ip 标签锁节点——local_address 是 VIP 不是节点 IP，用节点 IP 过滤 local_address 必空。权重 0=表内归零/排空；整机下线时 keepalived 直接删条目、序列消失（权重不变 0），下线看后端总数下降台阶
- 【变量 All 语义】正则（regex）只过滤下拉选项，选 All 时取值来自 allValue——allValue=".*" 会扫到所有带该指标的主机（曾出现 All=30 而单台 9+6=15）。需要限定 All 范围时必须显式枚举 allValue（如 `(192.168.13.218|192.168.13.219)`），与 regex 同口径；instance 带端口时用正则捕获组归一化成裸 IP
- 【ESXi / vmware_exporter】`vmware_host_*` 与 `vmware_vm_*` 自带 `host_name`/`vm_name` 标签，过滤统一用 `host_name=~"$esxhost"`；但 **`vmware_datastore_*` 没有 host_name**，只有 `ds_name`/`dc_name`/`ds_cluster`，本环境靠 Consul 元数据注入的 `app`（值=ESXi 主机名）定位，写 `app=~"$esxhost"`；原模板把数据存储写成 `app=~"$esxhost"` 而主机写成 `host_name` 混用是巧合正确，不能互换。共享存储被多台主机挂载时同卷会重复上报，聚合必须 `max by (ds_name)` 去重，否则 All 模式出多组曲线/重复行。`*_summation` 是 vCenter 20s 窗口的毫秒累计值（不是计数器），除以 20000 得「等效单核占比」，不能再套 rate/increase。关机主机不上报 boot/cpu/mem 序列（采集循环 continue），电源卡片不能写死 `or vector(0)`（会让故障掩盖为 0），用 `min()` 无兜底或 `count(...) or count(...)*0` 形式。传感器 `state`：0 红 / 1 黄 / 2 绿 / 3 未知；`redundancy`：1 正常 / 0 失效；`tools_running_status` 的取值在 `tools_status` 标签上（值恒为 1），表格要取标签列而非 Value 列
- 【跨节点聚合口径】VRRP 主备 director 服务表相同时，按节点累加的卡片（count 序列）与去重合并的 count by(...) 不一致（All=15 vs 6）——需按节点计数的聚合分组里必须带 ip；table-old 分组签名同理加 ip，并加首列「LVS节点」区分，否则 All 模式下两台相同 VIP→后端 行合并、连接数被错误求和
- 【ESXi 表格列重复 / 单位换算】`seriesToColumns` 会把**每个 value 序列自身的标签列**都并进来，所以多查询表格若各查询都带 `dc_name`/`host_name`/`ds_name`，就会出现「数据中心 / 主机」等列重复多份。修法：**除真正需要该字段的那一条查询外，其余查询统一只 `by` 主键聚合**（如 `max by (host_name) (...)`），需要额外标签的字段只保留在唯一一条查询里（如 `max by (host_name, dc_name, cluster_name)`）。这是 Grafana 7 下多查询表格的通用去重手法，不限于 ESXi
- 【vmware_exporter 单位速查】`vmware_vm_cpu_usage_average` / `vmware_vm_mem_usage_average` 是 **1/100 百分比**（原始 574 = 5.74%，已用 `usagemhz ÷ max_cpu_usage` 交叉验证一致），面板必须 `/ 100`，否则出现 553%、3099% 的假读数；`vmware_host_cpu_usage` 是 MHz、`vmware_host_memory_usage` 是 MB（绝对量，需除以 max 才是百分比）；`vmware_host_cpu_demand/usagemhz_average` 是 MHz；`vmware_host_net_bytes*_average`、`vmware_host_disk_*_average`、`vmware_vm_disk_usage_average`、`vmware_vm_net_*_average` 是 KB/s；`vmware_host_mem_*`（vmmemctl/active/swapin/swapout）与 `vmware_vm_mem_consumed/active_average` 是 KB；`vmware_datastore_*_size` 是字节；`vmware_vm_memory_max` 是 MB
- 【Grafana 7.1.4 单位 id】KB/s 的正确 id 是 **`KBs`**，`KBps` 不存在（不会报错，但会原样显示数字、看起来像「单位不生效」）。改单位时要同时改 `fieldConfig.overrides[].properties[] {id:unit}` 与 `yaxes[].format` 两处，只改一处会出现表头 KB/s、图轴仍是裸数字。轴标题写死单位（如 label=KB/s）会与 `KBs` 的自动换档（1.57 MB/s）打架，轴标题应写语义名（吞吐/容量）
- 【count 卡片兜底】`count(...) or vector(0)` 在采集断链时会把故障显示成 0，属「用假数据掩盖故障」；应写 `count(条件) or (count(全量) * 0)`——无匹配序列时取 0，对象整体缺失时保持空
- 【stat 卡片坑】reduceOptions.fields 必须为 ""，写成空数组 [] 会选不中任何字段显示 No data（查询有数据、旁边卡片正常，极难排查）
- 【Pushgateway / nginx_upstream_status】该指标值**恒为 1**，后端状态全在 `status` 标签上（标签集 `env/nginx/upstream/name/type/port/status`），所以所有统计都必须是 `count(...)` 与 `count(...status=~"up|UP|1|true")` 取差，绝不能对值做 sum/rate。**不能拿它判活**：Pushgateway 永久保留最后一次推送的值、抓取时只刷新样本时间戳，序列永远"新鲜"，脚本停了也照常返回。判推送是否中断只能看 Pushgateway 自带 `push_time_seconds{}`（`time() - max by (exported_job, exported_instance) (push_time_seconds{job="pushgateway"})`）。`job` 是 pushgateway 抓取任务，业务身份在 `exported_job`/`exported_instance`（honor_labels 冲突改名规律同 §2.1）。后端离线≠序列消失：整条 upstream 删除/实例改名只让 `count()` 下降
- 【单查询 table-old 列名】多查询才是 `Value #A`/`Value #B`，**单查询的列名是 `Value`**（无 refId 后缀），styles 里 pattern 写 `Value #A` 会匹配不到、列不显示别名与着色。写表前先数 targets 数量
- 【7.1.4 无 bargauge】面板类型只有 graph / stat / table / table-old / singlestat（§4 已记 timeseries 不支持，**bargauge 同样不存在**，导入报 Panel plugin not found）。想表达"排名/占比"只能用 graph 阶梯线，或在 table-old 里靠阈值着色
- 【空态语义：No data ≠ 故障，但用户不接受空白】只查「异常序列」的面板在全部健康时必然返回空集，Grafana 显示 No data / No data to show，用户会当成面板坏了。**离线类面板必须保留 0 基线**：分组级用 `count by (...) (异常) or (占位序列 and on() (count(全量) > 0) unless on() (count(异常) > 0))`——`and on()` 保证指标存在才出占位（采集断链仍为空白，不掩盖故障）、`unless on()` 保证有真实异常时占位消失。**不能用裸 `or`**：`A or B` 的匹配键是完整标签集，占位序列标签与 A 不同时会同时显示，正常时凭空多一行「全部在线」。趋势图用 `count(异常) or (count(全量) * 0)`、TopN 用「总数−异常数」而非 `topk(异常)`，否则健康时段整块空白或整条线消失
- 【`== bool` 会丢掉整个标签集】`(A == bool B)` 只对**A 中存在的标签集**求值；A 缺失的标签集（如某后端所有检查项都离线，导致「在线」侧为空）会整行消失。必须先补齐：`((A or (B * 0)) == bool B)`。同理 `A - B` 在 B 为空时也需要 `B or (A*0)`
- 【Pushgateway 兜底不能看指标本身的新鲜度】`timestamp()` 取到的是抓取时刻而非推送时刻，Pushgateway 保留旧值照样"新鲜"。判推送中断只能用 `push_time_seconds`（见上一条）
- 【填充遮挡：0/1 状态轴必须 fill=0】graph 的 `fill>0` 会从 0 铺到当前值，于是 0/1 状态图里**所有值为 1 的序列填充连成整块色带，把掉到 0 的序列整条盖住**——实测 nginx_upstream「后端状态矩阵」8 个后端全在线时整块糊成一片褐红，xxljob 那 3 个已掉到 0 的后端（图例 current=0）完全看不见。**铁律：纵轴 min=0/max=1 的状态类面板一律 `fill=0, fillGradient=0`**，只留线条，离线表现为落到底部的断崖。判定不能只看 targets 数量，要看**实际序列数**：`by (...)`、`topk(...)` 都会把单个 query 展开成多条序列（本盘 16/18 就因此漏判），改完必须按秒级真实数据或 `by` 子句复核。同一缺陷在 windows/windows-wmi 的「采集器健康」（id 139，多采集器 0/1 轴）也存在，已一并修。连续值的面积图（CPU/内存/流量）保持 fill 不受此限
- 【0/1 状态图图例排序】去掉填充后多条线重合在 y=1，靠图例区分谁掉线，故图例设 `sort:"current"` + `sortDesc:false`（**升序**：值 0 的离线项排最前）；写 `sortDesc:true` 会把离线的压到列表最末，等于白排
- 【table-old 列序由标签名字母序决定，不是 styles 顺序】这是极易踩错的一点：`styles` 数组只负责别名/格式/着色，**排不了列**。实测 `nginx_upstream` 表里 styles 写成 nginx→upstream→name→type→port→status，界面渲染却是 name→nginx→port→status→type→upstream（纯字母序）。要固定列序只能给标签键加排序前缀：`count by (a1_nginx, a2_upstream, a3_name, ...) (label_replace(label_replace(...), "a1_nginx","$1","nginx","(.*)"))`——`count by` 只保留新标签（原标签自动丢弃，不会产生重复列），列序即由 a1..a5 决定。代价是 pattern 要跟着改成新标签名。**排序号务必两位以内对齐**（a1..a9 而非 a1..a10，否则 a10 会排到 a2 前面）
- 【table-old 冗余列】列值恒为同一个数（如 `port` 全为 0）就应隐藏——若该信息在别的列里已含（后端地址本身是 `ip:port`），保留只会误导。写法：把该标签从 `by(...)` 里去掉即可整列消失，比 `type: hidden` 更干净（后者仍占用查询与序列标签）
- 【table-old 枚举列写法（0/1 → 中文）】在 style 上加 `valueMaps` 即可，**三个字段必须同时满足**：①`type="string"`（写成 number 则 valueMaps 不生效，仍显示 0/1）；②`valueMaps=[{"op":"=","value":"0","text":"正常"},{"op":"=","value":"1","text":"异常"}]`；③`colorMode="cell"` 才能给整格着色（`colorMode:"value"` 只染文字）。**着色方向极易搞反**：`colors` 语义是「值 < thresholds[0] → colors[0]；thresholds[0]~[1] → colors[1]；值 ≥ thresholds[1] → colors[2]」，所以 **0 落在 colors[0]、1 落在 colors[2]**。对「0=健康、1=故障」的列（副本不足/抓取状态）必须 `colors[0]=绿, colors[2]=红`；对「0=中性、1=正常」的角色列（控制器：从节点/主控制器）则 `colors[0]=灰, colors[2]=绿`。写反会出现「正常标红、故障标绿」的致命误读，改完务必用真实取值逐格核对颜色。
- 【Kafka 单位/表格列类型坑（2026-10-27 审计）】1) **table-old 的标签列必须 `type=string`**：`instance`/`topic` 是字符串，写成 `type=number` 会让表格按数值渲染（列宽、对齐、排序异常）；只有 `Value #X` 才是 number。2) **FailedProduceRequests / FailedFetchRequests 没有 `topic` 标签**（实测三台 exporter 均为空标签集），按 Topic 拆失败请求会得到一行空 Topic 名的无效面板——要按 Topic 拆只能用 `totalproducerequests_total`/`totalfetchrequests_total`（有 topic 标签），失败请求只能按 Broker 或按 `error` 标签（`kafka_network_requestmetrics_errors_total`）拆。3) **柱状图（bars + xaxis mode=series）基线必须 `min=0`**，否则柱子高度与实际值不成比例；且 `values` 只能是 `["current"]`，写成 `["total"]` 会把所有时间点累加（曾出现 3.17TiB 假值）。4) **二次轴单位必须跟着真实数据走**：把字节量放在标着 ms 的右轴会彻底误读，本盘 `日志清理` 的右轴已改 bytes。5) `rate(process_cpu_seconds_total)` 单位是「核」不是百分比，用 `short` 正确；要用 percent 需再除以核数。6) `rate(jvm_gc_collection_seconds_sum)` 单位是「秒/秒」=0~1 比例，必须用 `percentunit` 而非 `percent`，否则显示成 0.5% 而实际是 50%。
- 【Kafka 延迟语义：Fetch 的 p99 天然是 500ms】`kafka_network_requestmetrics_totaltimems{request="FetchConsumer"}` 的耗时**几乎全部落在 `remotetimems`**，实测 ≈501/502ms——这不是故障，而是长轮询机制：消费者发起 Fetch 后若无新数据，Broker 会挂起请求直到 `fetch.max.wait.ms`（默认 500ms）。因此 FetchConsumer/FetchFollower 的 p99 稳定在 500ms 附近属正常，**判断标准是它是否显著超出该阈值**，不能简单地"看 p99 高低"。真正要看的是：`localtimems`（磁盘读）、`requestqueuetimems`（线程饱和）、以及 Produce 侧（实测 Produce p99 仅 2~6ms，且时间全在 localtimems）。本仪表盘因此新增两张「延迟阶段分解」面板，把总耗时拆成排队/本地/远端/回包四段来定位瓶颈
- 【Kafka 值得补采的指标（源模板未覆盖）】经实测 177/178/179 三台 `/metrics` 确认存在且有数据：`kafka_server_replicamanager_underminisrpartitioncount`（ISR 跌破 min.insync.replicas，acks=all 随时失败）、`kafka_server_replicamanager_offlinereplicacount`、`kafka_log_logmanager_offlinelogdirectorycount`（磁盘离线，最严重）、`kafka_controller_controllerstats_uncleanleaderelections_total`（出现即可能已丢数据）、`kafka_network_requestmetrics_errors_total`（带 `error` 标签，实测能区分 FENCED_LEADER_EPOCH/SECURITY_DISABLED 等 14 种错误码）、`kafka_network_requestchannel_requestqueuesize`/`responsequeuesize`（延迟恶化先行指标）、`kafka_server_brokertopicmetrics_replicationbytesin/out_total`（副本同步流量，只有无 topic 的 bare 序列）、`kafka_coordinator_group_groupmetadatamanager_numgroups*`（消费组数与各状态，仅 Group Coordinator 上报）、`jvm_threads_state`（BLOCKED>0 = 锁竞争）、`jvm_classes_currently_loaded`（类加载器泄漏）、`process_open_fds`（句柄泄漏）、`process_resident_memory_bytes`。**注意 `kafka_server_fetcherlagmetrics_consumerlag` 名字像消费组 lag 其实是 Follower 副本拉取 lag**（标签为 clientId/topic/partition），消费组积压仍需 kafka_exporter/kminion
- 【Kafka / jmx_exporter 口径】**已用 177/178/179 三台 exporter 的 `/metrics` 实测核对**：1) `kafka_server_brokertopicmetrics_*` 同时有**无 topic 标签的 All Topics 聚合序列**与逐 Topic 序列，集群汇总卡片与按 Broker 曲线必须用 `topic=""`，逐 Topic 下钻才用 `topic!=""`，混加数值翻倍；2) **`kafka_controller_kafkacontroller_*` 只有当前 Controller 上报非零**（实测 177=56/1/3，178 与 179 = 0），属全集群唯一值 → 取 `max` 且**不能带 `instance=~"$broker"` 过滤**，否则用户只选非 Controller 的 Broker 时卡片会假报 0 并标红；3) **`kafka_server_replicamanager_partitioncount` 的语义是「本机承载的分区副本数」**（三台各 56，副本因子 3），集群分区总数必须用 `kafka_controller_kafkacontroller_globalpartitioncount` 取 max，`sum(partitioncount)` 会得到 3 倍；4) `kafka_network_requestmetrics_totaltimems` 本身是分位毫秒值（`quantile` 标签区分，实测导出 `0.99`/`0.999`），直接取值，套 rate/increase 无意义；5) **真实指标名易错三处**：请求处理空闲率是 `kafka_server_kafkarequesthandlerpool_requesthandleravgidle_percent`（不是 `...avgidlepercent`，后者只有 `_count_total` 这个无意义的累计计数），`*rateandtimems` 的计数器后缀是 **`_count_total`**（不是 `_count`），ZK 耗时 `zookeeperrequestlatencyms` 有 6 个 quantile 序列、必须显式 `quantile="0.99"` 否则被 sum 相加；6) 两个空闲率原值都是 0~1，占用率 = `100 − 空闲率×100`；实测均 ≈ 99.99% 空闲（0.01% 占用），面板上看到接近 0 是正常空闲；7) **`kafka_log_log_size` 是逐分区 gauge**，graph 的 x 轴用 `mode="series"` + `values:["total"]` 会把时间序列所有采样点累加成天文数字（曾出现 3.17 TiB），必须改 `values:["current"]` 且关掉对数轴，否则柱高还会误读；8) **消费组积压 `kafka_consumergroup_lag` 是 Kafka Exporter 的指标，jmx_exporter 没有**，本环境 `kafka_exporter-ops` 已注释、只留 `jmx_prometheus_javaagent`，故不含积压面板
- 【PromQL 正则里不要写反斜杠转义】`quantile=~"0\.99|99"` 会让 Prometheus 报语法错（PromQL 字符串按 Go 规则解析，`\.` 是非法转义），直接写 `quantile=~"0.99|99"` 即可——PromQL 的标签正则是**全匹配**语义，不需要 `^...$` 也不需要转义点号
- 【JMX 指标命名双形态】jmx_exporter 的 `jvm_*` 存在两种命名（旧版 `jvm_memory_bytes_used` / `jvm_memory_pool_bytes_used`，新版 `jvm_memory_used_bytes` / `jvm_memory_pool_used_bytes`，重命名由 exporter 版本决定）。实测本环境 Kafka 用的是**新版**那一套。稳妥写法是用 `{__name__=~"jvm_memory_(bytes_)?used_bytes"}` 一次匹配两种，避免换 exporter 版本后整块 JVM 面板空掉
- 【ZooKeeper / 内置 Prometheus provider 口径】1) **副本一致类指标取 max 不能 sum**：`znode_count`、`ephemerals_count`、`global_sessions`、`approximate_data_size`、`quorum_size`、`leader_uptime` 在每台 ZK 上都上报同一集群值，`sum` 会按副本数翻倍；`num_alive_connections`、`watch_count`、`local_sessions` 才是各节点自己的量。2) **声明类型不可全信**：`packets_sent` / `packets_received` 在 ZK 端点上注册为 gauge，但语义是自启动累计值，必须 `rate`（直接画原值是单调直线）；真正的 gauge 是 `watch_count`、`outstanding_requests`、`max/min/avg_latency`、`open_file_descriptor_count`、`last/max/min_proposal_size`、`connection_drop_probability`。累计计数器族：`snap_count`、`commit_count`、`proposal_count`、`diff_count`、`looking_count`、`dead_watchers_queued/cleared`、`outstanding_changes_queued/removed`、`request_commit_queued`、`prep_processor_request_queued`、`sync_processor_request_queued`、`tls_handshake_exceeded`、`ensemble_auth_*`、`response_packet_*_cache_hits/misses`、`node_*_watch_count` 实际导出为 `_sum`/`_count`（Summary 型）。3) **耗时指标必须算单次平均**：`fsynctime`、`readlatency`、`updatelatency`、`proposal_latency`、`quorum_ack_latency`、`commit_propagation_latency`、`*_queue_time_ms`、`*_final_proc_time_ms` 等由 `AvgMinMaxCounter`/`AvgMinMaxPercentileCounter` 导出，后缀是 `_sum`/`_count`/`_avg`/`_min`/`_max`（百分位版本是 `p50_`/`p95_`/`p99_`/`p999_` 前缀），**注意不是 `_sum`/`_count` 的 Prometheus Summary 惯例中的 `_avg` 语义**：`rate(_sum)` 得到的是「每秒累计毫秒」，单次平均必须 `rate(_sum)/rate(_count)`，分母加 `> 0` 过滤以免除零产生 Inf 尖峰。原模板大量直接画 `rate(xxx_sum)` 属于口径错误。4) **SummarySet 的标签名固定是 `key`**（PrometheusMetricsProvider 里 `LABEL = "key"`），`read_per_namespace` / `write_per_namespace` 的命名空间、`learner_handler_qp_size` 的 learner 地址、`ack_latency` 的分组都在这一个标签上，不叫 `namespace`/`peer`。5) **Leader 专属指标**：`learners`、`synced_followers`、`synced_non_voting_followers`、`synced_observers`、`pending_syncs`、`leader_uptime`、`last/max/min_proposal_size` 只在 Leader 上报，非 Leader 为空属正常；`looking_count` 尖峰 + `leader_uptime` 归零 = 一次重新选主。6) **JVM 指标双来源**：ZK 3.6+ 的 provider 默认 `exportJvmInfo=true`，会在同端点额外暴露 `jvm_memory_*`/`jvm_gc_*`/`jvm_threads_*`/`jvm_classes_loaded`；关掉该开关或旧版本需同机 jmx_exporter javaagent 补齐。两条链路的 `instance` 不同（ZK 端口 vs agent 端口），并集查询要分别绑各自的实例变量，最后用 `max` 去重（`or` 后 `sum` 会把同值序列算两遍）；`rate()` 不能直接套在 `(a or b)` 上（range selector 只接受向量选择器），须先各自 `rate` 再 `or`。7) `jvm_memory_pool_bytes_used` 的池标签是 `pool`，`jvm_memory_bytes_used/max` 的区域标签是 `area="heap"|"nonheap"`；旧版 simpleclient 与新 client_java 命名不同（`jvm_memory_bytes_used` ↔ `jvm_memory_used_bytes`），跨版本时用 `{__name__=~"jvm_memory_(bytes_)?used_bytes"}` 兜底
8) **空闲时段不能整段丢数据（2026-10-27 实测修复）**：Summary 类指标（fsynctime / readlatency / election_time / snapshottime / follower_sync_time / om_*_process_time_ms / time_waiting_empty_pool_* / inflight_snap_count 等）在业务空闲时窗口内没有新观测，`rate(_count)` 为 0。**把分母写成 `(rate(_count) > 0)` 是错的**——PromQL 的比较运算是 filter，分母为 0 的时间点会连同整条序列一起被丢弃，这些低频面板就在空闲期整段 No data（曾误判为采集故障）。已用 prepro 实测确认：`sum(rate(up[5m]) > 0)` 返回 0 条序列，`sum(rate(up[5m])) / clamp_min(sum(rate(up[5m])), 1e-9)` 返回 1 条序列值 0。统一改 `clamp_min(分母, 1e-9)` 后，8 类面板在空闲模拟下全部返回连续 0 值曲线。同类写法适用于所有比值面板（堆使用率、fd 使用率、缓存命中率）：**分母一律 clamp_min，永远不要用比较运算符做"保护"**。详见 project_memory.md §1。
9) `read_per_namespace` / `write_per_namespace` 的观测值是**字节数**（DataTree：读记 totalBytes，写记 path.length()+bytes），不是操作次数，单位应为 Bps / 字节，不能标 ops。
- 【SQLServer / mssql_exporter 口径】1) **自启动累计计数器**：`mssql_deadlocks`、`mssql_user_errors`、`mssql_kill_connection_errors`、`mssql_batch_requests`（Batch Requests/sec）、`mssql_lock_waits`、`mssql_page_fault_count`、`mssql_log_growths`、`mssql_io_stall_total_seconds`、`mssql_wait_stats` —— 必须 rate/increase，原模板把 `mssql_tempdb_space_used_mb`（gauge）套 `irate` 属错误；2) **瞬时 gauge 直取**：`mssql_connections`、`mssql_blocking_sessions`、`mssql_long_running_queries`、`mssql_resident_memory_bytes`、`mssql_virtual_memory_bytes`、`mssql_page_life_expectancy_seconds`、`mssql_buffer_cache_hit_ratio_percent`、`mssql_log_space_used_percent`、`mssql_tempdb_space_used_mb`、`mssql_cpu_usage_percent`、`mssql_memory_utilization_percentage`；3) `mssql_wait_stats` 的 `stat` 标签只有 `wait_time_ms` / `waiting_tasks_count` 两个值，**两个累计值直接相除毫无意义**（分子分母都随时间增长），平均等待必须写 `increase(wait_time_ms)/increase(waiting_tasks_count)`，分母为 0 的类型用 `(sum by (wait_type)(increase(...)) > 0)` 过滤，否则 TopN 会被空序列占位；4) `mssql_os_memory` / `mssql_os_page_file` 只有 `state="used"|"available"`，**没有 total 序列**，总量要在面板内 `sum by (instance)` 求和；5) `mssql_memory_utilization_percentage` 是「工作集 / 已提交内存」之比，不是整机内存使用率，别和 `mssql_os_memory` 混用；6) `instance` 是 exporter 端点，`db` 是库名，同名库可跨实例出现，按库聚合必须同时 `by (instance, db)`；7) 原模板的 `legendFormat="{{ip}}"` 在本 job 下无 `ip` 标签（Consul labelmap 未注入），会显示为空，应改用 `{{instance}}`；8) 原模板 3 处拼写/格式坑：`mssql_buffer_cache_hit_ratio_percent {` 中间多空格（PromQL 容忍但易误改）、`mssql_page_life_expectancy_seconds{...}\r` 尾部含回车（Grafana 7 里 query 直接报错）、`mssql_lock_waits ` 面板名带尾随空格

## 6. 标准 制作/验证 流程（新模型照此执行）

1. **读原模板**：提取面板结构、查询、变量（PowerShell `ConvertFrom-Json -AsHashtable` 遍历输出 title/expr/gridPos）
2. **探测真实指标**：优先 dump 文件 grep 指标前缀；或集群内 API 实查；或 `kubectl get --raw ".../proxy/metrics" | grep -m5 <前缀>`
3. **修错误**：不存在的标签/指标、拼写、irate→rate、累计值画线、count 语义、join 标签集
4. **重写为套件统一结构**：变量（§4 规则）、w12×2 布局、中文 description、图例表格、or vector(0) 兜底
5. **语法门禁**：提取全部 targets expr，替换 `$kubernetes_cluster`→`.*`，逐条 POST `http://prepro-prometheus.k8s.hs.com/api/v1/query`，status=error 即语法错（结果为空属预期——如 node 指标不在集群内）
6. **数据抽查**：KSM 类查询在集群内应有真实返回（记录序列数与样本值）
7. **JSON 校验**：解析 OK、面板数、无 description 面板数=0、id 无重复、gridPos 无重叠、总高度
8. **交付说明**：修复清单、面板结构、验证方式、导入方式（覆盖导入原版保留）
9. 更新 project_memory.md 沉淀新经验

**JSON 工具坑**：`kubernetes-nodes-optimized.json` 存在 linewidth/lineWidth 大小写冲突键，普通 ConvertFrom-Json 报错，必须 `-AsHashtable`；对该文件只能文本级 String.Replace 修改，不能对象序列化写回（会合并丢键）。

## 7. dashboard 清单（全部已完成并通过验证）

### k8s 目录（12 个优化版）
| 文件 | 说明 |
|---|---|
| kube-state-metrics-01-cluster-overview-optimized.json（KSM 01 集群总览） | Kube State Metrics 套件，Deployment/StatefulSet/DaemonSet/HPA 四副本状态面板列结构完全统一 |
| argo-rollouts-optimized.json | 老版 rollout_* 指标；阶段列着色 Completed 绿/Progressing|Paused 黄/Error|Abort|Timeout 红 |
| argocd-optimized.json | 42 条目（36 面板 + 6 分类行），总高 94：应用健康/同步趋势、同步与协调、工作队列延迟、Git/gRPC/Redis、集群连接与缓存；补齐 API 资源种类与两张 Git 耗时热力图，新增 CPU、队列 P95、非 2xx、Redis P95；变量为集群/项目/目标集群/统计窗口 |
| coredns-optimized.json | 18 面板，prepro kube-dns + test node-local-dns（service 变量多选 includeAll） |
| kubernetes-api-server-optimized.json | 18 面板，29 查询 prepro 实测全通过 |
| kubernetes-etcd-optimized.json | 30 面板 8 stat+21 graph+1 table，51→41 查询，etcd 3.5.1 实名适配 |
| kubernetes-cluster-optimized.json | 29 面板（8 stat + 19 graph + 2 表格） |
| kubernetes-nodes-optimized.json | 41 面板 84 查询；注意 §6 键冲突坑 |
| kubernetes-pods-optimized.json | 38 条目（32 面板 + 6 分类行），总高 67；含指定 Pod 级联下钻区、Pod/容器级 Top10（容器级无网络维度，用磁盘 IOPS 补齐 6 图） |

### 其它目录（9 个优化版）
elasticsearch / iis / linux / mysql / nginx / rabbitmq / redis / windows（windows-optimized、windows-wmi-optimized）；Windows2003.json 为源表参考不动

### lvs 目录（1 个优化版，2026-10-27 交付）
| 文件 | 说明 |
|---|---|
| lvs-optimized.json（LVS (Optimized)，uid=lvs-opt-5672） | 26 条目 = 5 row + 8 stat(w3 h5 首行) + 11 graph(w12×2) + 1 table-old(全宽 w24 h9)，27 条查询语法门禁 0 失败；分区：概览卡片 / 后端明细（独立行，兼作服务汇总）/ 连接分析 / 流量分析 / 后端与权重；变量口径与 ipvs 指标坑见 §5 追加条目 |

### esxi 目录（1 个优化版，2026-10-27 交付）
| 文件 | 说明 |
|---|---|
| esxi-optimized.json（ESXi (Optimized)，uid=esxi-opt-10076） | 38 条目 = 5 row + 8 stat(w3 h5 首行) + 14 graph(w12×2) + 11 table，74 条查询语法门禁 0 失败；分区：概览（含主机信息表）/ 主机资源 / 数据存储 / 虚拟机 / 硬件传感器；数据存储用 app 标签定位、按 ds_name 去重；表格去重与单位换算口径见 §5 追加条目 |

### nginx_upstream 目录（1 个优化版，2026-10-27 交付）
| 文件 | 说明 |
|---|---|
| nginx_upstream-optimized.json（Nginx Upstream (Optimized)，uid=nginx-upstream-opt） | 24 条目 = 5 row + 8 stat(w3 h5 首行) + 10 graph + 3 table-old，23 条查询语法门禁 0 失败；分区：概览 / 后端状态明细 / 状态趋势 / 分组汇总 / 采集健康。原模板仅 2 面板（1 graph + 1 table），且三个变量都用 `job="pushgateway"` 硬编码、env/nginx 为单选且写死 current，已全部改为三级联动多选 All + current:null + refresh:2。指标只有 `nginx_upstream_status`（值恒 1，状态在 status 标签），故全部用 count/取反方式统计，无权重与性能维度；采集健康改用 Pushgateway 自带 `push_time_seconds`，因为该指标经 Pushgateway 保留后永远"新鲜"，只看它无法发现推送中断 |

### 已完成的全局清理
- 全部 22 个优化版的标题与描述已清除模板溯源信息（原模板「xxx」/已修复/新增面板/保留自原版/gnetId 引用等），只保留面板名与指标口径、版本适配、性能设计等技术说明

### kafka 目录（1 个优化版，2026-10-27 交付）
| 文件 | 说明 |
|---|---|
| kafka-optimized.json（Kafka (Optimized)，uid=kafka-opt-11962） | 52 面板 = 7 row（首行为 8 张 stat w3 h5）+ 42 graph + 2 table-old(w24 h9)，共 59 条目、总高 156；96 条查询语法门禁 0 失败；分区：集群概览 / 集群健康 / 吞吐 / 延迟 / 请求与异常 / 消费组 / JVM；布局按 3×w8、2×w12、w24 三种栅格，**每行严格铺满 24 列无空洞**；变量 Job(单选) + Broker/Topic(includeAll、current=null、refresh=2)；指标口径见 §5 Kafka 条目 |

### sqlserver 目录（1 个优化版，2026-10-27 交付）
| 文件 | 说明 |
|---|---|
| sqlserver-optimized.json（SQL Server (Optimized)，uid=sqlserver-opt） | 34 条目 = 5 row + 8 stat(w3 h5 首行) + 20 graph(w12×2) + 1 table-old(全宽 w24 h9)，总高 89，38 条查询语法门禁 0 失败；分区：概览 / 连接与阻塞 / 内存与缓存 / 吞吐与等待 / 日志与明细。原模板 21 个面板全是 w6 或 w3 的满宽堆叠、无分类行、无 row，多个面板缺 description；且 `mssql_buffer_cache_hit_ratio_percent` 后带空格、`mssql_page_life_expectancy_seconds` 的 expr 尾部含 `\r`、gauge 类指标（tempdb_space_used_mb）被套 irate、wait_stats 用两个累计值直接相除（得到越来越大的垃圾值）、`legendFormat` 用了不存在的 `{{ip}}` 标签。本版把 9 个自启动计数器统一改 rate/increase、11 个 gauge 直接取值、等待类型改为 increase 比值并新增「排队任务数」「等待累计时长」两个维度，补「数据库明细」快照表（连接/日志使用率/日志增长/IO 停顿，按 instance+db 分组）。变量 Job/Env/实例 三级联动，均 includeAll、current=null、refresh=2，job 的 allValue 限定 `.*mssql_exporter.*`；指标口径见 §5 SQLServer 条目 |

### zookeeper 目录（1 个优化版，2026-10-27 交付）
| 文件 | 说明 |
|---|---|
| zookeeper-optimized.json（ZooKeeper (Optimized)，uid=zookeeper-opt-10465） | 83 条目 = 9 row + 16 stat（概览 8×w3 h5 + JVM 8×w3 h5）+ 57 graph(w8×h6 三列) + 1 table-old(全宽 w24 h7)，总高 140（三列紧凑布局，同等信息量下比两列 w12×h7 布局矮 44%），170 条查询语法门禁 0 失败；分区顺序：概览 / 集群与选主（含实例状态总览表）/ **JVM** / 数据与 Watch / 请求与延迟 / 连接与会话 / 请求处理链 / 磁盘与文件句柄 / 安全与 TLS。原模板 111 面板（89 graph + 12 row + 3 bargauge + 3 stat + 3 singlestat + 1 gauge，其中 bargauge 在 7.1.4 根本不支持）只有裸指标名作标题、无概览卡片、无一条中文说明，并且多处语义错误：`rate(znode_count)` / `rate(ephemerals_count)` 给 gauge 套 rate、`irate(fsynctime_sum[1m])` 与对 Summary 只画 `_sum`（得到「每秒累计毫秒」而非单次耗时）、`dbinittime` / `startup_snap_load_time` / `startup_txns_loaded` 裸查询未带 job/instance 过滤、`count(quorum_size{job=~"$job"})` 把仲裁规模取成实例数。本版统一 gauge 取瞬时值（或 delta）、计数器取 rate、Summary 用 `rate(_sum)/rate(_count)` 且**分母 `clamp_min` 兜底（不可用 > 0 过滤，见 §5 第 8 条）**；副本一致类指标改 max 防翻倍；命名空间维度按字节（Bps）而非次数；补实例状态总览表、按命名空间读写吞吐与写量 Top10、Learner 复制队列、Observer Master、限流拒绝、Inflight 快照、FD 使用率、堆/非堆水位等面板；变量 Job(单选，allValue=`zookeeper.*`) / 实例(多选 All) / JVM 任务 / JVM 实例，均 current=null + refresh=2；指标口径见 §5 ZooKeeper 条目与 project_memory.md |

## 8. 遗留观察点

- 中心 VM 导入后需观察：group_left join（CoreDNS Pod 名显示）、and on 集合运算（异常 Pod 清单）在 VM 下是否正常（VM 的 join 有翻车史，group_left/集合运算相对可靠）
- test 集群节点名若为 hostname（非 IP），Node Status 总览表的 node_exporter/KSM 两侧对齐可能缺行（prepro 节点名=IP 已验证对齐）
- prepro 6 节点 .90/.92/.94 为 cordon 的 master（不可调度=1 属正常数据）
- ArgoCD dashboard 依赖联邦后附加的 `kubernetes_cluster` 标签与 `job="argocd-server-metrics"`；导入中心 Grafana 后先确认「集群/项目/目标集群」变量能正常解析（prod 内该 job 有 3 个 target，argocd_app_info 当前 988 条、4 个目标集群）
- ArgoCD 的「gRPC 非 OK」「Redis 失败」「K8s API 非 2xx」在健康时为空属预期，不要当成面板故障

## 9. 用户偏好与沟通

- 沟通语言：中文；对文件格式质量要求高；生成对外文档要去除内部引用
- 系统级操作前先咨询；操作非本人电脑前需征得同意并确认备份
- 「排行/TopN」不用累计值、偏好阶梯折线图；先要求「尽量全」再要求「紧凑」是常见节奏，最终以紧凑去重版为准
- 面板描述要求「干净」：不留制作过程/模板对比信息
