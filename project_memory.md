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
