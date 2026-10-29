# opsmonitor 项目记忆

> 本文件是 HANDOFF.md 的补充，记录**可复用的排查手法与踩坑细节**，供后续接手的人或模型快速避坑。
> 最后更新：2026-10-27。环境与硬性规则以 HANDOFF.md 为准，这里只写"为什么"和"怎么验证"。

---

## 1. 本次最重要的教训：PromQL 的比较运算符是 filter，不是保护

### 现象

ZooKeeper 仪表盘里 10 个低频面板（选主耗时、fsync、快照、Observer Master、空池等待、Follower 同步、Sync 处理耗时、Inflight 快照等）**整段显示 No data**，密集采样区也是一片空白，看起来像采集断了。

### 根因

为了「单次平均耗时」写成了这样：

```promql
rate(fsynctime_sum[5m]) / (rate(fsynctime_count[5m]) > 0)
```

意图是"避免除以零"，但 **PromQL 里比较运算返回的是过滤结果**：`> 0` 会把分母为 0 的**整个时间点连同序列一起丢弃**。而这些 Summary 指标在业务空闲时窗口内本来就没有新观测，`rate(_count)` 恒为 0 —— 于是整条序列被判为空，面板显示 No data。

**关键认知：`A / (B > 0)` 与 `A / B` 在分母为 0 时的行为不是"更安全"，而是"更危险"**——前者让曲线消失（用户会误判为采集故障），后者产生 Inf/NaN（Grafana 显示断点）。

### 正确写法

```promql
rate(fsynctime_sum[10m]) / clamp_min(rate(fsynctime_count[10m]), 1e-9)
```

分母用 `clamp_min` 兜底：没有事件时分子也是 0，`0 / 1e-9 = 0`，**输出 0 而不是丢弃**。曲线保持连续，语义清晰：
- 曲线贴 0 = 这段时间没有该事件（健康）
- 整段消失 = 采集真的断了（故障）

判断"发生过选主"应看**曲线是否抬升**，而不是"有没有线"。

### 实测证据（prepro Prometheus）

| 写法 | 返回 |
|---|---|
| `sum(rate(up[5m]) > 0)` | **0 条序列** → 面板 No data |
| `sum(rate(up[5m])) / clamp_min(sum(rate(up[5m])), 1e-9)` | 1 条序列，值 `0` |

`up` 恒为 1、`rate(up[5m])` 恒为 0，正好模拟"窗口内无新观测"。

### 适用面（不止耗时面板）

任何「比值 / 平均值」面板都可能踩：
- 堆使用率 `used / (max > 0)`
- 文件描述符使用率 `used / (limit > 0)`
- 缓存命中率 `hit / clamp_min(hit + miss, 1e-9)`

**统一规则：分母一律 `clamp_min(x, 1e-9)`（或按量纲取 1），禁用比较运算符做"保护"。**

### 顺带发现的两个变体坑

1. **兜底分支也会踩**：`(主分支) or (avg_X) or (X{quantile="0.5"})` 里，如果主分支因过滤为空，`or` 才会去取兜底；一旦兜底指标也不存在，仍然 No data。所以主分支本身必须写对，不能指望兜底。
2. **`or` 的语义**：`A or B` 是"左空才取右"，不是"合并"。同一实例同时有两条链路（站点 `jvm_*` + javaagent `jvm_*`）时，用 `or` 会只取左侧，需要 `max` 去重而不是 `sum`（`sum` 会把同值序列算两遍）。

---

## 2. ZK 指标口径速查（源码级核对）

核对基准：**ZooKeeper release-3.6.4** 源码。核实路径：
- `server/ServerMetrics.java`（Summary / Counter 注册）
- `server/ZooKeeperServer.java`（gauge 注册：znode_count / watch_count / uptime / max_latency …）
- `server/quorum/LeaderZooKeeperServer.java`（Leader 专属：leader_uptime / learners / synced_* / proposal_size）
- `server/quorum/QuorumZooKeeperServer.java`（`quorum_size`，注意不在 QuorumPeer 里）
- `server/metric/AvgMinMaxCounter.java`（导出 `avg_/min_/max_/cnt_/sum_` 前缀）
- `server/metric/AvgMinMaxPercentileCounter.java`（额外 `p50_/p95_/p99_/p999_` 前缀）
- `zookeeper-metrics-providers/.../PrometheusMetricsProvider.java`（SummarySet 标签名 = `"key"`）

### 2.1 声明类型不可全信

`packets_sent` / `packets_received` 在 ZK 端点上 **注册为 gauge，但语义是自启动累计值**，必须 `rate`，直接画原值只会得到单调上升的直线。

### 2.2 对照表

| 类别 | 指标 | 正确用法 |
|---|---|---|
| 瞬时 gauge | `znode_count` `ephemerals_count` `watch_count` `global_sessions` `local_sessions` `num_alive_connections` `outstanding_requests` `approximate_data_size` `uptime` `leader_uptime` `max/min/avg_latency` `open_file_descriptor_count` `max_file_descriptor_count` `last/max/min_proposal_size` `last/min/max_client_response_size` `connection_drop_probability` `outstanding_tls_handshake` `learners` `synced_followers` `synced_observers` `synced_non_voting_followers` `pending_syncs` `quorum_size` `non_mtls_*_conn_count` | 直接取值，或 `delta()` 看变化 |
| **伪 gauge 累计量** | `packets_sent` `packets_received` | **必须 `rate`** |
| 累计 Counter | `snap_count` `commit_count` `proposal_count` `diff_count` `looking_count` `connection_request_count` `connection_rejected` `connection_drop_count` `bytes_received_count` `revalidate_count` `stale_sessions_expired` `sessionless_connections_expired` `dead_watchers_queued/cleared` `outstanding_changes_queued/removed` `prep_processor_request_queued` `sync_processor_request_queued` `request_commit_queued` `tls_handshake_exceeded` `ensemble_auth_*` `unrecoverable_error_count` `digest_mismatches_count` `quit_leading_due_to_disloyal_voter` `auth_failed_count` `add_dead_watcher_stall_time` `stale_requests*` `stale_replies` `connection_revalidate_count` `request_throttle_wait_count` `large_requests_rejected` | `rate([5m])` |
| Summary（耗时/长度） | `fsynctime` `snapshottime` `dbinittime` `readlatency` `updatelatency` `proposal_latency` `quorum_ack_latency` `ack_latency` `commit_propagation_latency` `propagation_latency` `proposal_ack_creation_latency` `follower_sync_time` `election_time` `connection_token_deficit` `inflight_snap_count` `inflight_diff_count` `om_*_process_time_ms` `dead_watchers_cleaner_latency` `prep_processor_queue_time_ms` `prep_processor_queue_size` `prep_process_time` `close_session_prep_time` `sync_process_time` `sync_processor_queue_size` `sync_processor_batch_size` `sync_processor_queue_flush_time_ms` `commit_process_time` `read/write_commitproc_time_ms` `server/local_write_committed_time_ms` `write_batch_time_in_commit_processor` `read/write_final_proc_time_ms` `time_waiting_empty_pool_in_commit_processor_read_ms` `startup_snap_load_time` `startup_txns_loaded` `requests_in_session_queue` `pending_session_queue_size` `reads_after_write_in_session_queue` `reads_issued_from_session_queue` `session_queues_drained` `read/write_commit_proc_req_queued` `commit_commit_proc_req_queued` `read/write_commit_proc_issued` `concurrent_request_processing_in_commit_processor` | `rate(_sum[10m]) / clamp_min(rate(_count[10m]), 1e-9)` |
| SummarySet（带 key 标签） | `read_per_namespace` `write_per_namespace` `learner_handler_qp_size` `learner_handler_qp_time_ms` `ack_latency` | 同上，`by (instance, key)` 分组 |

### 2.3 副本一致性：取 max 不取 sum

ZK 各副本数据**完全一致**，集群级指标在每台机器上都上报同一个值：

- 取 `max`：`znode_count` `ephemerals_count` `global_sessions` `approximate_data_size` `quorum_size` `leader_uptime`
- 取 `sum` 或按实例对比：`num_alive_connections` `watch_count` `local_sessions`（各节点自己的量）

`sum` 会让数值按副本数翻倍（3 副本集群 ×3）。

### 2.4 命名空间维度的单位是字节，不是次数

`DataTree.java` 里：
```java
ServerMetrics.getMetrics().READ_PER_NAMESPACE.add(namespace, totalBytes);      // 响应体字节
ServerMetrics.getMetrics().WRITE_PER_NAMESPACE.add(namespace, path.length() + bytes);  // 路径 + 数据字节
```

所以 `read_per_namespace` / `write_per_namespace` 是**吞吐（Bps）**，不是 QPS。按它做的 TopN 是"哪个命名空间数据量最大"，不能当请求量排名。

### 2.5 SummarySet 的标签名固定是 `key`

`PrometheusMetricsProvider` 里 `private static final String LABEL = "key";`。所以：
- `read_per_namespace{key="/app/config"}` —— 不是 `namespace`
- `learner_handler_qp_size{key="192.168.1.5:2888"}` —— 值是 learner 地址

### 2.6 Leader 专属指标

`learners` `synced_followers` `synced_observers` `synced_non_voting_followers` `pending_syncs` `leader_uptime` `last/max/min_proposal_size` **只在 Leader 上报**，非 Leader 为空属正常，不要当成采集故障。

### 2.7 JVM 指标双来源

ZK 3.6+ 的 `PrometheusMetricsProvider` 默认 `exportJvmInfo=true`，会在**同一端点**额外暴露 `jvm_memory_*` / `jvm_gc_*` / `jvm_threads_*` / `jvm_classes_loaded`。关掉该开关或旧版本需同机 jmx_exporter javaagent 补齐。

两条链路 **instance 不同**（ZK 端口 7000/2181 vs agent 端口 9404），查询要分别绑各自的实例变量：

```promql
max by (instance, pool)(
  jvm_memory_pool_bytes_used{job=~"$job", instance=~"$instance"}
  or jvm_memory_pool_bytes_used{job=~"$jmxjob", instance=~"$jvminstance"}
)
```

注意：
- `rate()` **不能直接套在 `(a or b)` 上**（range selector 只接受向量选择器），须先各自 `rate` 再 `or`
- 两条链路同时存在时用 `max` 去重，**不能用 `sum`**（同值序列会算两遍）
- `jvm_memory_pool_bytes_used` 的池标签是 `pool`；`jvm_memory_bytes_used/max` 的区域标签是 `area="heap"|"nonheap"`
- 旧版 simpleclient 与新 client_java 命名不同：`jvm_memory_bytes_used` ↔ `jvm_memory_used_bytes`，跨版本可用 `{__name__=~"jvm_memory_(bytes_)?used_bytes"}` 兜底

### 2.8 `quorum_size` 在哪个类

不在 `QuorumPeer.java`，在 **`QuorumZooKeeperServer.java:199`**：
```java
rootContext.registerGauge("quorum_size", () -> { ... });
```
查指标全树搜索时容易漏掉，注意 `quorum/` 子包下的 `QuorumZooKeeperServer`。

### 2.9 版本差异：Summary rotation

- 3.6.x / 3.7.x / 3.8.x：Summary **不做周期性 reset**，`rate(_sum)/rate(_count)` 安全
- master（较新）：引入 `summaryRotateSeconds = 60`，Summary **每 60 秒旋转一次**，此时 `rate(_sum)` 会因计数器归零而失真

目前生产用的是 3.6/3.7 形态；若升级到带 rotation 的版本，需要用 `X{quantile="0.5"}` 或 `avg_X` 取值替代 rate 计算。

---

## 3. Grafana 7.1.4 布局压缩手法

从「两列 w12×h7、总高 251」压到「三列 w8×h6、总高 140」（-44%），信息量不减反增（83 vs 83 面板）。

### 3.1 三列 w8×h6

```python
def trio(y, specs, h=6):
    """一行三列 w8×h6，specs = [(title, desc, targets, unit, ylabel, kwargs)]"""
    x = 0
    for title, desc, targets, unit, ylabel, kw in specs:
        graph(title, desc, targets, unit=unit, ylabel=ylabel, w=8, h=h, x=x, y=y, **kw)
        x += 8
    return y + h
```

要点：
- `w8 × 3 = 24`，正好满行，不留半行空白
- 每个分区凑满整行（面板数 = 3 的倍数）；不齐的用「合并同类项」补齐，而不是留空位
- `h=6` 比 `h=7` 省 1 格，三列下每行省 3 格视觉高度

### 3.2 合并窄面板成双轴图

原来拆开的「A 平均耗时」+「A 次数」两张 w12 图，合并成一张 w8 双轴图：

```python
graph("Fsync 平均耗时与次数", "...",
      [T(avg_of("fsynctime"), "{{instance}} 平均耗时"),
       T(rate_by("fsynctime_count"), "{{instance}} 次数", "B")],
      unit="ms", ylabel="耗时",
      y2unit="ops", y2label="速率",
      seriesOverrides=[{"alias": "/.*次数/", "yaxis": 2}])
```

- 左轴 `yaxes[0]` 放主指标（耗时 / 容量）
- 右轴 `yaxes[1]` 通过 `y2unit` / `y2label` 单独指定单位，**必须显式给右轴也标单位**，否则右轴显示裸数字
- `seriesOverrides` 的 `alias` 用正则匹配图例名，把对应系列推到 `yaxis: 2`

### 3.3 轴标题写语义名，不写单位

`label="毫秒"` 会与 `format="ms"` 的自动换档打架（1.57 s 时轴标题还写"毫秒"）。轴标题写「耗时 / 容量 / 吞吐 / 速率 / 数量」这类语义名，单位交给 `format`。

### 3.4 分区排序要重排 y，不要手工改坐标

移动整个分区（如把 JVM 组从末尾移到「数据与 Watch」之前）时，**不要逐个改 gridPos**（83 个面板手改必错）。正确做法是脚本按 row 切分：

```python
sections = []          # [{'row': row_panel, 'panels': [...]}]
cur = None
for p in panels:
    if p['type'] == 'row':
        cur = {'row': p, 'panels': [], 'title': p['title']}
        sections.append(cur)
    else:
        cur['panels'].append(p)

# 按 order key 排序后，从 y=0 重新堆叠，保留分区内部相对偏移
cursor = 0
for s in sorted(sections, key=key):
    base = s['row']['gridPos']['y']
    offsets = [(s['row'], 0, s['row']['gridPos']['h'])]
    offsets += [(p, p['gridPos']['y'] - base, p['gridPos']['h']) for p in s['panels']]
    height = max(off + h for _, off, h in offsets)
    for p, off, h in offsets:
        p['gridPos']['y'] = cursor + off
    cursor += height
```

好处：分区内部布局原样保留，只重算分区起点，不会破坏 3 列对齐。

---

## 4. 验证手法（可复用）

### 4.1 结构门禁脚本要点

```python
# 1) 网格重叠检测（逐格标记）
cells = {}
for p in panels:
    g = p['gridPos']
    for yy in range(g['y'], g['y'] + g['h']):
        for xx in range(g['x'], g['x'] + g['w']):
            assert (xx, yy) not in cells, "overlap at %s" % ((xx, yy),)
            cells[(xx, yy)] = p['id']
```

必查项：
- `id` 唯一
- 网格无重叠、`x + w <= 24`
- 每个面板有非空 `description`
- `stat` 面板 `reduceOptions.fields == ""`（写成 `[]` 会 No data）
- `stat` 的 targets 必须 `format="time_series"` + `instant=false`（否则无 sparkline）
- `table-old` 的 targets 全部 `instant=true` + `format="table"` + `legendFormat=""`
- 变量 `current` 为 `null`、`query` 为纯字符串、有 `definition`
- 面板标题全局唯一（同名会让折叠/搜索行为异常）

### 4.2 PromQL 语法门禁

逐条 POST `http://prepro-prometheus.k8s.hs.com/api/v1/query`，替换变量后看 `status`：

```python
q = (expr.replace("$instance", ".*").replace("$job", "zookeeper-pro")
         .replace("$jmxjob", "jmx_prometheus_javaagent-pro")
         .replace("$jvminstance", ".*").replace("$__range", "1h"))
```

`status=success` 即语法通过（结果为空属预期——本地没有该指标）。

**注意**：`prepro` 集群里没有 zookeeper / kafka 等抓取任务，只能验证语法，**不能验证数据**。要验证"面板是否真有数据"，得用第 4.3 的模拟法。

### 4.3 空闲场景模拟（验证 No data 类问题）

把面板里的指标替换成"恒定不变"的指标（`up`），模拟"窗口内无新观测"：

```python
# 把 rate(xxx{...}[5m]) 全部换成 rate(up[10m])
idle = re.sub(r'rate\((\w+)\{.*?\}\[[^\]]+\]\)', 'rate(up[10m])', main_branch)
```

若返回序列数 > 0 → 面板在空闲时会显示 0 值曲线（正确）
若返回 0 条序列 → 面板会 No data（说明写法有问题）

### 4.4 指标名对照源码

用 GitHub raw 拉源码，正则抽 `getSummary\("(\w+)"` / `registerGauge\("(\w+)"`，与仪表盘引用的指标名求差集：

```python
found |= set(re.findall(r'get(?:Summary|SummarySet|Counter|CounterSet)\("([a-z0-9_]+)"', txt))
found |= set(re.findall(r'registerGauge\w*\("([a-z0-9_]+)"', txt))
```

比翻文档可靠，能发现拼写错误和版本差异。注意要覆盖 **多个类**（`ServerMetrics` / `ZooKeeperServer` / `LeaderZooKeeperServer` / `QuorumZooKeeperServer`），漏类会误报。

---

## 5. 本次会话的协作经验

1. **截图报"无数据"时，先怀疑自己的查询写法，再怀疑数据源**。本次就是我在上一版引入的 `> 0` 过滤 bug，而非采集问题。
2. **"更安全的写法"要实测**。`(B > 0)` 直觉上比裸除法安全，实际恰恰相反 —— 用 prepro 跑一条对照实验，1 分钟就能定案。
3. **单位要回到源码看记账口径**。`read_per_namespace` 名字像次数，实际记的是字节；靠猜会错。
4. **布局调整用脚本切分区重排，不要手改坐标**。手改 83 个 gridPos 必然出错，而且难以 review。
5. **提交前先看 `git status`**。工作区可能有前几次会话遗留的未提交文件，要区分"本次改动"与"历史遗留"，避免把别人的半成品混进提交。

---

## 6. 指标名"存在"不等于"口径正确"：以 MinIO 为例的四层核对法（2026-10-28）

做 minio 优化版时发现：**所有指标名都能在官方文档里查到、所有 PromQL 都能通过语法门禁，面板依然是错的**。语法门禁只能证明"表达式能跑"，不能证明"数对"。这次被迫下到 MinIO 源码（`cmd/metrics-v2.go`、`metrics-resource.go`、`metrics-v3.go`、`peer-rest-server.go`、`notification.go`）逐条核对，才挖出四类只靠文档看不出来的坑。可复用的核对顺序如下。

### 6.1 第一层：指标名（文档 grep 即可）

用官方 `docs/metrics/prometheus/list.md` 抽出全部 `` `minio_*` `` 名称做白名单，与仪表盘里正则抓出的指标名求差集：

```python
official = set(re.findall(r"`(minio_[a-z0-9_]+)`", md))
used     = set(re.findall(r"minio_[a-z0-9_]+", expr))
unknown  = used - official      # 剩下的基本都是 job 名之类的误匹配
```

这一步只能抓拼写错误。**过了这关不代表对**。

### 6.2 第二层：端点归属（决定 job 过滤怎么写）——本次最大的坑

MinIO 有四个 v2 端点（cluster / node / bucket / resource），而**同一个指标组会被注册到多个 collector 上**：

```go
nodeGroups = []*MetricsGroupV2{
    getNodeHealthMetrics(...),   // → minio_cluster_nodes_online_total 也在 node 端点
    getHTTPMetrics(...),         // → minio_s3_requests_total 也在 node 端点
    getNetworkMetrics(),         // → minio_s3_traffic_* 也在 node 端点
    ...
}
clusterCollector = newMinioClusterCollector(allMetricsGroups)  // 含上面全部
```

本仓库 `prometheus.yml` 恰好用四个独立 job（`minio-job` / `-node` / `-bucket` / `-resource`）抓同一主机。于是写 `job=~"$minio_job.*"` 会同时命中四个任务，**同一条样本被累加两次，带宽、请求数、在线节点数全部翻倍**，而且曲线形状完全正常，肉眼极难发现。

**规则：MinIO 的 job 过滤一律精确匹配 `job="$minio_job"`，永远不要用前缀正则。** 判断某个指标属于哪个端点，别看文档表格，直接读 collector 的 group 列表。

### 6.3 第三层：指标形态（决定能不能套 rate/histogram_quantile）

名字里带 `_distribution` / `_histogram` 不代表是 Prometheus 原生直方图。MinIO 的 v2 导出器自己把内部 map 摊平成 **带 `le` 标签的累积计数 gauge**：

```go
// metrics-v2.go collectMetric
if metric.Description.Type == histogramMetric {
    for k, v := range metric.Histogram {
        ... prometheus.GaugeValue, float64(v) ...   // 注意是 Gauge，且无 _bucket 后缀
    }
}
```

所以 `minio_s3_requests_ttfb_seconds_distribution_bucket{le=...}` **不存在**（原模板就写错了）。正确写法是直接对同名指标按 le 聚合：

```promql
histogram_quantile(0.99, sum by (le) (rate(minio_s3_requests_ttfb_seconds_distribution{...}[5m])))
```

同理，`minio_bucket_objects_size_distribution` 的分桶标签是 **`range`**（值形如 `LESS_THAN_1KB`），而且**值是"对象个数"不是字节数**——单位标 bytes 会得到完全错误的数量级。

### 6.4 第四层：时间语义（最容易自欺欺人的一类）

名字叫 `..._last_activity_nano_seconds` 的指标，**值本身就是"距上次活动已经过去多久"**，不是时间戳：

```go
Value: float64(time.Since(bgSeq.lastHealActivity)),   // 已经是 elapsed
```

因此**只能 `/1e9` 换成秒，绝不能写 `time() - 该值`**——那样会得到负数或天文数字，且因为曲线依然连续，看不出错。原模板和大量社区面板都踩这个。

区分办法只有看源码那行赋值：出现 `time.Since(x)` / `Since` 就是 elapsed；出现 `x.Unix()` / `x.UnixNano()` 才是时间戳（才需要 `time() -`）。

### 6.5 顺带：百分比类指标的"已换算"陷阱

resource 端点的百分比是**在服务端就算好的百分数**：

```go
perc := math.Round(float64(stats.Used*100*100)/float64(stats.Total)) / 100   // 已是 %
cpuIdleVal := math.Round(ts.Idle/tot*100*100) / 100                          // 已是 %
```

所以 `100 - minio_node_cpu_avg_idle` 就是 CPU 使用率，**不能再乘 100**；同理 `minio_node_mem_used_perc` 直接取值。反过来，`minio_node_if_rx_bytes` 的 help 写着 "Bytes received ... in 60s"（采集间隔 `resourceMetricsCollectionInterval = time.Minute`），是**窗口字节数不是速率**，换算 B/s 要 `/60`。

### 6.6 一句话总结

**语法门禁只证明"能跑"，指标核对才证明"数对"。** 新接一个 exporter 时，投入产出比最高的动作是：① 拉官方指标清单做名字白名单；② 读源码确认该指标注册在哪个 collector（定 job 过滤）；③ 读源码确认导出类型（定要不要 rate / 怎么算分位）；④ 读源码确认那行赋值的时间语义与单位。四步都过再写面板，比事后对着"曲线形状正常但数值翻倍"猜要快得多。

---

## 7. "属性在标签里"的指标怎么画面板：以 SoftRAID 为例（2026-10-28）

做 softraid 优化版时遇到一类性质不同的指标：**值和属性被拆开了，且属性在标签上**。`raid_status` 只有值是数字（0/1/2 状态码），阵列级别、盘数、容量、创建/更新时间、重建状态**全是标签**，而标签是字符串。PromQL 没有"标签转数值"的算子，`label_replace` 提取出来的仍然是字符串标签，不能参与算术。

### 7.1 能做什么、不能做什么

| 需求 | 可行写法 | 说明 |
|---|---|---|
| 数有几个降级阵列 | `count(raid_status{...} == 1)` | 计数可用 |
| 失败盘非零的阵列 | `count(raid_status{..., raid_failed_devices=~"[1-9][0-9]*"})` | **用标签正则做数值判断**，前提是脚本把数量写成纯十进制整数 |
| 整机/整阵列容量 | ❌ 做不了 | `raid_array_size` 是字符串，`sum` 只能数条目数，不是容量 |
| 重建进度百分比 | ❌ 做不了 | `raid_rebuild_status` 是 `recovering 45%` 这类文本，抽不出数字 |
| 阵列创建时间排序 | ❌ 做不了 | `raid_create_time` 是字符串标签，`sort` 不了 |

**铁律：属性在标签上的指标，只能做「计数」和「按标签过滤」，永远不要试图聚合它的属性值。**原模板恰恰写了 `sum by(instance, exported_instance, raid_array_size) (raid_status{...})`——把字符串标签塞进 `by()` 的后果是每个阵列各出一条序列再被求和，卡片最终显示的是「阵列条数」而不是容量，而且因为标签值不同永远合并不掉。

要真正拿到进度/容量，只有两条路：① 换数值型指标（mdadm collector 的 `node_md_blocks` / `node_md_blocks_synced`，进度 = `blocks_synced/blocks`、ETA = `(blocks-blocks_synced)/rate(blocks_synced[10m])`）；② 改采集脚本输出**数值型**指标（textfile collector），把属性从标签移到独立指标。仅靠改 PromQL 是无解的。

### 7.2 Grafana 7.1.4 的两个连带限制

1. **`table-old` 不能靠 styles 排序**（HANDOFF §5 已记）：想让「主机 → 设备 → 状态」按业务顺序排，必须在查询里用 `label_replace` 造 `a1_`/`a2_`… 前缀标签，靠字母序定列序，`styles` 只改别名与着色。单查询表的列名是 `Value`（无 refId 后缀）。
2. **`stat` 的 `reduceOptions.calcs` 选 `mean` 会把状态码平均成小数**：0/1/2 的状态用 `mean` 在混合状态下会得到 0.33、1.5 这类值，阈值判色直接失真。状态类卡片一律用 `lastNotNull`（或 `max` 取最差），聚合口径要么 `max`（暴露最差阵列）要么 `count(== N)`（分类计数），**不要对状态码求平均**。

### 7.3 Pushgateway 数据必须单独做判活（与 nginx_upstream 同源问题）

`push_time_seconds` 是唯一可靠判据，写法：

```promql
time() - max by (exported_job) (push_time_seconds{job="pushgateway", exported_job="raid_monitor"})
```

三个坑：
- `honor_labels` 未开时推送侧 `job`/`instance` 被改名成 `exported_job`/`exported_instance`，样本上的 `job` 恒为 `pushgateway`。过滤业务必须写 `exported_job`，写 `job=~"raid.*"` 永远空。
- Pushgateway 重启后内存态丢失，**指标整个消失**（表现为 No data，不是 0）；无兜底时卡片空白，加 `or vector(0)` 又会把"没有推送分组"伪装成"刚刚推送过"——所以这张卡**故意不加兜底**，空白就是故障。
- 只有"推送成功"才刷新 `push_time_seconds`；推送一直失败时它不动，因此还要配 `push_failure_time_seconds` 才能区分"没推"和"推了但失败"。

#### 7.3.1 `push_time_seconds` 必须与业务指标的主机集合求交（2026-10-28 实测修复）

`push_time_seconds` 是 **Pushgateway 全局**指标：上面有**所有**推送分组，不止你的业务。只按 `exported_job` 过滤还不够——本次实测截图里「主机采集状态」表凭空多出一行：

| 主机 | 距上次推送 | 阵列数 | 降级阵列 | 未知阵列 |
|---|---|---|---|---|
| 172.168.2.199 | 49 s | 1 | 0 | 0 |
| **prometheus** | 47 s | – | – | – |

`prometheus` 是 Pushgateway **抓取任务自身**注册的分组，与阵列毫无关系，却因为 `push_time_seconds{exported_job="raid_monitor"}` 之外还存在同族序列、而判活查询只做了 `max by (exported_instance)` 而混入。危害有两层：① 表里多出信息残缺的假行；② `max()` 会取到**最久没推送的那个分组**，让新鲜度虚高，真故障被别的分组掩盖。

**修法：用业务指标的主机集合做 `and on` 求交。**

```promql
# 错：会把 Pushgateway 上所有推送分组都当成阵列主机
max by (exported_instance) (time() - push_time_seconds{job=~"pushgateway", exported_job=~"$raid_job"})

# 对：只保留真正上报了 raid_status 的主机
max by (exported_instance) (time() - push_time_seconds{job=~"pushgateway", exported_job=~"$raid_job"})
  and on (exported_instance)
    (max by (exported_instance) (raid_status{exported_job=~"$raid_job"}))
```

判据：**任何"外挂型"判活指标（push_time_seconds / node_textfile_mtime_seconds / up）在与业务数据同表时，都必须先与业务标签集合求交**，否则表里一定会混进与本业务无关的分组。同理，`up{job=~"pushgateway"}` 这类抓取状态卡也只能反映 Pushgateway 自身，不能替代业务判活。

### 7.4 降级/消失是两种不同的故障

阵列**降级**（掉一块盘但表项还在）会体现在状态码 1；阵列**消失**（未组装、被拆除，或 keepalived/mdadm 直接删条目）则是**序列整体消失**，状态码不会变成 1，只会少一条曲线。所以「阵列总数趋势」的下降台阶是发现后者的唯一手段，不能只盯着降级计数。这与 LVS 后端「整机下线时序列消失而非权重变 0」是同一个陷阱。

### 7.5 字符串标签列做 valueMaps 枚举的三个约束

`raid_state` / `raid_rebuild_status` 这类"状态在标签上"的列，用 `valueMaps` 翻译成中文可读性提升很大，但 7.1.4 下必须同时满足：

1. **`type` 必须写 `string`**。写成 `number` 则 valueMaps 完全不生效，列里仍是 `clean, degraded` 原文。
2. **枚举值要和真实取值逐字符一致**。`raid_state` 是内核 `State` 字段原文，组合值极多（`clean` / `active` / `clean, degraded` / `clean, degraded, recovering` / `clean, degraded, resyncing` …）。**枚举不完是正常的**——未命中的值 Grafana 会回落显示原文，不会丢信息，所以策略是"枚举最常见的一批 + 其余保留原文"，不要试图穷举（也不可能穷举，重建百分比类是动态字符串）。
3. **字符串列不要指望 `colorMode:cell` 着色**。cell 着色依赖**数值**阈值，对字符串列不可靠、语义也对不上；颜色统一交给「健康状态」这种**数值**列承担（`0/1/2` + `thresholds:["1","2"]` + `valueMaps` 三件套）。让"数值列着色 + 文本列翻译"分工，比两列都染色更稳。




---

## 8. 第五层核对：指标属于哪个端点（2026-10-28，MinIO 修正）

第 6 节讲了「指标名 → 形态 → 时间语义」四层核对，但漏了同样致命的一层：**同一个指标名可能只由某一个端点暴露**。写错端点不会报错、语法门禁全绿，只是面板静默 No data。

### 8.1 现象

minio 优化版交付后，用户反馈「节点变量选不出东西」。检查发现变量写的是：

```
label_values(minio_node_process_starttime_seconds{job="minio-job-node"}, server)
```

而该指标**只存在于 cluster 端点**。变量解析为空 → `$server` 为空 → 所有带 `instance=~"$server"` 过滤的面板连带全部无数据（17 个面板）。这种「一个变量错、半个仪表盘空」的故障，看起来像采集坏了，实际是查询写错。

### 8.2 为什么文档看不出来

官方 `list.md` 把指标按「Cluster / Node / Bucket / Resource」分节列出，但**同一指标组可以注册进多个 collector**。真正决定暴露端点的是 `cmd/metrics-v2.go` 里这几个 group 列表：

```go
clusterMetricsGroups = [...]          // /cluster 独有
peerMetricsGroups    = [...]          // 只并入 clusterCollector！
nodeGroups           = [...]          // /node
bucketMetricsGroups  = [...]          // /bucket
clusterCollector = newMinioClusterCollector(allMetricsGroups)  // = cluster + peer
```

关键：`getMinioProcMetrics()`（starttime / uptime / cpu_total / resident_memory / FD / goroutine / syscall / io）与 `getScannerNodeMetrics()` **只出现在 `peerMetricsGroups`**，而 peer 组只并入 clusterCollector。所以这些名字带 `node_` 前缀的指标，实际上**只有 cluster 端点有**。

### 8.3 可复用的判定流程

对任一指标，四步定端点：

1. 搜指标名对应的 `getXxxMD()`（定义里写着 Namespace/Subsystem/Name）；
2. 看这个 MD 被哪个 `MetricsGroupV2` 函数引用（`Description: getXxxMD()`）；
3. 看该 group 函数出现在 `clusterMetricsGroups` / `peerMetricsGroups` / `nodeGroups` / `bucketMetricsGroups` 的哪个列表；
4. cluster 端点 = `clusterMetricsGroups` **+** `peerMetricsGroups`（后者容易漏）。

可以脚本化：解析 Go 源码做「MD → group → endpoint」映射，再拿仪表盘里每个 `job="..."` + 指标名做交叉校验。本次用这个方法一次性查出 12 处错配：

```
minio_node_process_starttime_seconds  job=minio_job_node  -> 实际 ['cluster']
minio_node_file_descriptor_open_total job=minio_job_node  -> 实际 ['cluster']
minio_node_go_routine_total           job=minio_job_node  -> 实际 ['cluster']
minio_node_scanner_objects_scanned    job=minio_job_node  -> 实际 ['cluster']
...
```

### 8.4 顺带：`server` vs `instance`

MinIO 自己注入的节点标签是 **`server`**（`ReportMetrics` 里 `m.VariableLabels[serverName] = globalLocalNodeName`）；`instance` 是 Prometheus 抓取时加的「目标地址」。

- 四个 job 抓**同一个** target → `instance` 恒为单值，只能用 `server` 分节点；
- 每节点一个 target → 两者都能分节点，`instance` 更贴近「一台机器」。

选定后**过滤、分组、图例三者必须同口径**：

```
变量： label_values(<metric>{job="minio-job"}, instance)
过滤： instance=~"$server"
分组： sum by (instance) (...)
图例： {{instance}}
```

只改其中一个（如过滤用 instance 但分组仍 by (server)）会得到空图例或错位曲线。

### 8.5 教训

**「语法门禁 0 失败」与「面板有数据」之间，还差一个「端点归属」校验。** 后续接新 exporter，第 6 节的四层核对要扩成五层：名字 → 端点 → 形态 → 时间语义 → 单位。

---

## 9. 第六层核对：指标在什么部署形态下才存在（2026-10-28，MinIO 单机 No data 修正）

第 6、8 节讲了「名字 → 端点 → 形态 → 时间语义 → 单位」五层，还差一层：**指标是否在**当前部署形态**下才导出**。语法门禁、端点归属全对，面板依然可能整块 No data，因为该指标在这个部署模式下根本不会产生。

### 9.1 三个面板同时 No data 的真实原因

用户报「集群内部流量与错误」「自愈进度」「距上次自愈活动」三块全空。查源码发现都是**条件上报**：

```go
// metrics-v2.go getNetworkMetrics
if globalIsDistErasure {                 // 只有多节点纠删码集群才有节点间流量
    metrics = append(metrics, ... minio_inter_node_traffic_* ...)
}

// metrics-v2.go getMinioHealingMetrics
bgSeq, exists := globalBackgroundHealState.getHealSequenceByToken(bgHealingUUID)
if !exists { return metrics }            // 没有自愈轮次 -> 直接返回空
if bgSeq.lastHealActivity.IsZero() { return metrics }
```

用户环境是**单机单节点**：没有纠删码集群 → 无 internode 流量；单机从不触发后台自愈 → 无 heal 序列。所以 No data 是**预期行为，不是采集故障**。

同类条件指标还有：`minio_cluster_usage_*` 与 `minio_usage_last_activity_nano_seconds`（`dataUsageInfo.LastUpdate.IsZero()` 时为空，即刚部署/刚重启、扫描器没跑完首轮）、`minio_bucket_replication_*`（未配置桶复制）、`minio_cluster_kms_*`（未启用 KMS）。

### 9.2 处理手法：`or (max(up) * 0)` 补 0 基线，但**不能**用 `or vector(0)`

目标是区分三态：① 功能启用、有真实数据；② 功能未启用（指标不存在）但采集正常 → 显示 0 平线；③ 采集也断了 → 保持空白（不把故障伪装成 0）。

```promql
sum(rate(minio_inter_node_traffic_received_bytes{job="$minio_job"}[5m]))
  or (max(up{job="$minio_job"}) * 0)
```

`or` 的语义是「左支为空才取右支」，而右支 `max(up{...}) * 0` 只在 up 存在时才有值——正好编码了「采集正常」这个前提。prepro 实测三态：

| 场景 | 查询 | 返回 |
|---|---|---|
| 指标存在 | `sum(rate(up[5m]))` | 1 条，真实值 |
| 指标缺失 + up 正常 | `sum(rate(nonexistent[5m])) or (max(up{job=X}) * 0)` | **1 条，值 0** |
| 指标缺失 + up 也缺失 | `sum(rate(nonexistent[5m])) or (max(up{job=Y}) * 0)` | **0 条（空白）** |

对比 `or vector(0)`：它无条件补 0，采集全挂时也显示 0，属「用假数据掩盖故障」，禁用。

### 9.3 坑中坑：`or` 两侧的标签集必须对齐

`or` 是按**完整标签集**匹配的。左支若按 instance 分组，右支不分组，就会出问题：

```promql
# 错误：左支 by (instance)，右支无标签
sum by (instance) (nonexistent) or (max(up{job=X}) * 0)
#   实测返回 1 条「空标签」序列 -> 图例 {{instance}} 渲染成空白

# 正确：两侧同分组
sum by (instance) (nonexistent) or (max by (instance) (up{job=X}) * 0)
#   实测返回 4 条，各自带 instance 标签
```

规则：**左支 `sum by (X)` ⇔ 右支 `max by (X)`；左支无 `by` ⇔ 右支无 `by`**。写占位时成对检查，别只改一边。

### 9.4 教训

「指标名对、端点对、单位对」仍可能整块 No data——还要问一句**「这个指标在当前部署形态下会产生吗」**。接新 exporter / 新面板时，把这类条件指标列个清单（源码里的 `if xxx {` 分支、`if !exists { return }`、`if IsZero() { return }`），逐条决定：给占位、还是干脆不画、还是在描述里写清「本形态下为空属预期」。用户看到 No data 的第一反应永远是「采集坏了」，面板描述必须替他们回答这个问题。

---

## 10. 第七层核对：向量匹配的「标签集签名」（2026-10-29，traffic_network）

前面九节讲的都是**指标本身**对不对（名字、端点、形态、时间语义、单位、部署条件）。这一节讲另一类完全不同的错误：**指标全对，但两个向量没匹配上 / 匹配后重复**。这类错误同样「语法门禁全绿、面板静默 No data 或曲线翻倍」，而且比指标错误更难发现——因为没有名字可查。

### 10.1 机制：匹配是按「标签集签名」比的，不是按「几个关键标签」

Prometheus 的向量匹配（`A op B`、`A or B`、`A unless B`）签名规则：**除 `ignoring()` 列出的标签、以及 `__name__` 之外，其余全部标签都参与比较**。

由此推出两条极易踩的等价关系：

- **缺失标签 ≡ 空字符串**。所以 `x{foo=~".*"}` 会匹配到「根本没有 foo 标签」的序列；要排除它们必须写 `x{foo!=""}`。
- **多一个标签就匹配不上**。`A{instance,job,ifIndex,ifName,ifAlias}` 与 `B{instance,job,ifIndex}` 做 `A / ignoring(ifDescr) group_right B` 时，因为 A 还有 ifName/ifAlias 没被 ignoring 掉，签名不等 → **静默无匹配、结果为空，不报错**。

实测（prepro）：

| 写法 | 返回 |
|---|---|
| `A{…,extraA} / ignoring(extraB) group_right() B{…,extraB}` | **0 条** |
| `A{…,extraA} / ignoring(extraA, extraB) group_right() B{…,extraB}` | 1 条 |
| `A{…,extraA} / on(instance, job) group_right() B{…,extraB}` | 1 条 |

**结论：永远优先写 `on(...)` 显式列出匹配键，而不是 `ignoring(...)` 猜哪些标签多余。** 采集侧加一个 Consul 元数据标签，`ignoring` 写法就会悄悄失效。

### 10.2 `group_right` 保留哪一侧的标签

`A / ... group_right() B` 结果**以右侧 B 为标签基座**，再叠加 `group_right(...)` 括号里从 A 取的标签。这点决定了「接口名从哪来」：读数（多侧）在左、名字指标（一侧）在右时，结果天然带名字标签，可以直接 `{{ifDescr}}`。

### 10.3 `or` 的第二个陷阱：`rate()` 会丢掉 `__name__`

`or` 按完整标签集匹配，而 **`rate()` 的输出不带 `__name__`**。于是：

```promql
# 错误：四个错误计数器只算到「最左侧存在的那一项」
sum by (instance) (rate(ifInErrors[5m]) or rate(ifOutErrors[5m])
                   or rate(ifInDiscards[5m]) or rate(ifOutDiscards[5m]))
#   实测：4 个计数器只出 1 个的结果，其余 3 项被静默丢弃

# 正确：逐项各自 or 零基线，再相加
sum by (instance) (rate(ifInErrors[5m])   or (ifOperStatus * 0))
+ sum by (instance) (rate(ifOutErrors[5m]) or (ifOperStatus * 0))
+ ...
```

同类陷阱也出现在「64 位计数器优先、32 位回落」：

```promql
# 两条 rate() 的标签集完全相同（都没有 __name__）→ or 去重成一条？
# 不：or 只保留左侧，右侧的接口直接消失
rate(ifHCInOctets[5m]) or rate(ifInOctets[5m])
# 正确：把已有 64 位的接口从回落分支里剔掉
rate(ifHCInOctets[5m])
  or (rate(ifInOctets[5m]) unless on (instance, job, ifIndex) rate(ifHCInOctets[5m]))
```

**区分「该用 or 还是该用 +」的唯一标准**：两个分支是「**同一指标的两种来源、二选一**」→ 用 `or` + `unless` 去重；是「**不同指标要累加**」→ 各自 `or` 零基线后 `+`。

### 10.4 双形态兼容：同一语义的两套标签，合并必然重复

snmp_exporter 的 `ifDescr`（以及 `temperatureProbeLocationName`）有两种暴露形态：独立字符串指标 / lookups 标签。想同时兼容，就会写成「分支 a join 独立指标 `or` 分支 b 直接用标签」。实测发现**两种形态并存时同一接口出两条曲线**（一条有名字、一条名字为空）——因为 `or` 只能按标签集去重，一条带 `ifDescr="Gi0/0/1"`、另一条 `ifDescr` 为空，是两个不同标签集。

修法：右侧分支补 `unless`，排除已被左侧覆盖的接口

```promql
sum by (instance, job, ifIndex, ifDescr) (rate(X) / on (instance, job, ifIndex) group_right() ifDescr{...})
  or (sum by (instance, job, ifIndex, ifDescr) (rate(Y))
      unless on (instance, job, ifIndex)
      sum by (instance, job, ifIndex) (rate(X) / on (instance, job, ifIndex) group_right() ifDescr{...}))
```

**通用规则：任何「兼容两种数据形态」的 or 合并，右支都要 `unless on(主键)` 左支**，否则两种形态并存时必然重复。

### 10.5 分步仿真：把「面板能不能出数」变成可执行的验证

语法门禁只能证明「表达式能跑」。要验证「标签是否带出、每接口几条曲线、求和是否漏项」，用 prepro 上真实存在的 `up` 系列**伪造**目标指标：对每条 `up` 序列 `label_replace` 打上 `instance`/`ifIndex`/`ifDescr`，再 union 出「一台设备 N 个接口」的全套 IF-MIB 指标，然后把面板表达式里的指标选择器**按原样替换**（保留 matcher 语义，尤其保留 `ifDescr!=""` 的过滤效果），逐条 POST 检查：

- 每接口恰好 1 条（翻倍 → `or`/`unless` 写错）
- 图例里带出接口名（缺失 → join 方向或匹配键错）
- 数值等于各项之和（偏小 → `or` 链丢了项）

本次就靠这个手法在交付前抓到 **2 个真实 bug**：① 4 个错误计数器 `or` 串联只算到 1 项；② 双形态合并出 6 条曲线（应为 3 条）。两者语法门禁都是全绿。

### 10.6 教训

**「语法门禁 0 失败」离「面板正确」还差一次「标签集仿真」。** 标签类错误（匹配不上 / 匹配重复 / 求和漏项）不报错、不违反语法，但结果全错，且和指标名无关——查指标文档查不出答案。做多指标 join、多版本回落的仪表盘时，把「伪造序列 → 跑真实表达式 → 断言条数与数值」当成标准流程，比上线后对着空图猜快得多。

