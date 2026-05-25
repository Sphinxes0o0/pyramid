---
type: synthesis
tags: [NIDS, Snort3, Event, Alert, Comparison]
created: 2026-05-25
sources: [nids-current-architecture, nids-gap-analysis-roadmap, dps-market-research]
---

# NIDS vs Snort3 Event/Alert 对比

## 1. 架构概览

| 维度 | NIDS | Snort3 |
|------|------|--------|
| **核心组件** | EventEngine | Event System + Logger Plugins |
| **事件来源** | DetectionResult, RuntimeEvent | DetectionEngine (rule match) |
| **输出目标** | SOA Service, Local NLog | Multiple Logger plugins |
| **配置文件** | nids_conf.yaml | snort.lua |

### NIDS EventEngine 架构

```
DetectionEngine ──→ DetectionResult ──→ EventEngine::Process()
HealthMonitor   ──→ RuntimeEvent    ──→ EventEngine::EmitSystemEvent()
                                         │
                    ┌────────────────────┼────────────────────┐
                    ↓                                         ↓
               SOA Service                                 NLog
         (ReportCommonEventParam)              [ALERT][<nic>] sid:<sid> ...
```

### Snort3 Event 架构

```
DetectionEngine::detect()
        │
        └─→ queue_event(gid, sid)
                │
                └─→ fpLogEvent() (速率过滤/阈值检查)
                        │
                        └─→ fpFinalSelectEvent()
                                │
                                └─→ sfeventq_add() ──→ LoggerManager
                                                           │
                        ┌─────────────┬─────────────┬──────┴──────┐
                        ↓             ↓             ↓             ↓
                   alert_fast    alert_json    alert_csv      unified2
```

---

## 2. 事件数据结构

### NIDS event_info JSON

```json
{
  "sig_id": 2001002,
  "proto": "TCP",
  "nic": "PFE.VLAN1",
  "src": "172.20.1.2:52100",
  "dst": "172.20.1.1:22",
  "msg": "SSH Scan",
  "action": "alert",
  "hit_count": 3,
  "window_match_count": 60,
  "timestamp": "1747123456.789"
}
```

| 字段 | 说明 |
|------|------|
| sig_id | 触发规则的 SID |
| proto | L4 协议 (TCP/UDP/ICMP/...) |
| nic | 抓包 NIC 名称 |
| src/dst | IP:Port（无 L3 时为 null） |
| msg | 规则 msg 字段 |
| action | pass/log/alert/drop/block/reject |
| hit_count | 进程级累计触发次数（从 1 起） |
| window_match_count | 当前 threshold 窗口内原始命中次数 |
| timestamp | Unix 时间戳（秒.毫秒） |

### NIDS ReportCommonEventParam 映射

| proto 字段 | 规则命中事件 | 健康事件 |
|-----------|------------|---------|
| rule_version | cfg_.rule_version | cfg_.rule_version |
| domain_type | VDF_A | VDF_A |
| event_source | ETH_IDS (12) | ETH_IDS (12) |
| event_type | SID / 1_000_000 | EventType::Health = 99 |
| rule_id | hit.signature_id | 0 |
| score | hit.severity | 0 |

### NIDS EventType 枚举

```cpp
enum class EventType : uint32_t {
    AttemptedDos   = 1,   // flood / volumetric attack
    NetworkScan    = 2,   // port/host scan, sweep, DPSCAN
    AttemptedRecon = 3,   // OS/tool fingerprinting
    Health         = 99,  // health-monitor events (CPU/MEM threshold)
};
```

### Snort3 Event 类

```cpp
class Event {
    uint32_t ts_sec, ts_usec;      // 时间戳
    uint32_t event_id;             // 全局唯一事件 ID
    uint32_t event_reference;      // 引用其他事件
    const SigInfo& sig_info;       // GID/SID/REV + msg + class_type
    const char* action;            // 关联的动作
    const char** buffs_to_dump;     // 待转储的缓冲区
};
```

**GID (Generator ID) 常用值：**

| GID | 组件 |
|-----|------|
| 1 | snort 主检测引擎 |
| 2 | TCP 流分析器 |
| 3 | IP 分片重组 |
| 134 | 延迟管理 |
| 175 | 特定协议检测器 |

---

## 3. 动作系统 (Actions)

### NIDS action 字段

NIDS 的 action 直接体现在 event_info JSON 中：

| action | 语义 |
|--------|------|
| alert | 仅生成告警 |
| pass | 忽略该流量 |
| log | 记录日志 |
| drop | 丢弃数据包（无告警） |
| block | 阻断会话 |
| reject | 发送 TCP RST/ICMP 并告警 |

### Snort3 IpsAction 优先级体系

```cpp
enum IpsActionPriority : uint16_t {
    IAP_OTHER   = 1,   // 其他（最低）
    IAP_LOG     = 10,  // 日志
    IAP_ALERT   = 20,  // 告警
    IAP_REWRITE = 30,  // 重写
    IAP_DROP    = 40,  // 丢弃
    IAP_BLOCK   = 50,  // 阻断
    IAP_REJECT  = 60,  // 拒绝
    IAP_PASS    = 70,  // 通过（最高）
};
```

| 动作 | 优先级 | 丢弃流量 | 说明 |
|------|--------|----------|------|
| alert | 20 | 否 | 生成告警 |
| log | 10 | 否 | 记录日志 |
| pass | 70 | 否 | 标记为通过 |
| drop | 40 | 是 | 丢弃数据包并告警 |
| block | 50 | 是 | 阻断会话并告警 |
| reject | 60 | 是 | 发送 TCP RST/ICMP 并告警 |
| react | 41 | 是 | 发送 HTTP 403 页面并重置 |

**关键差异**：NIDS 的 action 是事件字段，Snort3 是独立对象且有优先级体系。

---

## 4. 日志/告警输出 (Loggers)

### NIDS 本地日志格式

```
[ALERT][<nic>] sid:<sid> proto:<proto> src:<ip:port> dst:<ip:port> msg:"<msg>"
```

### Snort3 内置 Logger

| Logger | 标志位 | 输出格式 |
|--------|--------|----------|
| alert_fast | ALERT | 简洁文本 |
| alert_json | ALERT | JSON 结构化 |
| alert_csv | ALERT | CSV 格式 |
| alert_syslog | ALERT | syslog |
| log_pcap | LOG | PCAP 格式 |
| log_hext | LOG | 十六进制 |
| unified2 | ALERT\|LOG | 统一格式 (Barnyard2 兼容) |

### Snort3 alert_fast 输出格式

```
[时间戳] [**] [GID:SID:REV] [消息] [**] [优先级] [应用ID] {协议} {IP信息}
```

### Snort3 alert_json 示例

```json
{
  "timestamp" : "04/15-13:45:32.123456",
  "pkt_num" : 12345,
  "proto" : "TCP",
  "dir" : "C2S",
  "src_ap" : "192.168.1.100:54321",
  "dst_ap" : "10.0.0.1:80",
  "rule" : "1:1000:1",
  "action" : "allow"
}
```

---

## 5. 事件队列管理

### NIDS

NIDS 无独立事件队列，检测结果直接通过 EventEngine 处理上报。

- hit_count: 进程级累计，detection_filter 放行计 1
- window_match_count: 当前 threshold 窗口内原始命中（含被抑制的 N-1 次）

### Snort3 sfeventq

Snort3 使用预分配内存的优先级队列：

```cpp
struct SF_EVENTQ {
    SF_EVENTQ_NODE* head;      // 队列头部
    SF_EVENTQ_NODE* last;      // 队列尾部
    SF_EVENTQ_NODE* node_mem;   // 节点内存池
    char* event_mem;           // 事件内存池
    char* reserve_event;       // 优先级比较用
    int max_nodes;             // 队列最大节点数
    int cur_nodes;            // 当前队列深度
    unsigned fails;           // 分配失败次数
};
```

**队列满时处理**：比较新事件与 last（最低优先级）事件，重要则替换，否则丢弃。

**EventQueue 配置 (snort.lua)**：

```lua
event_queue = {
    max_events = 8,          -- 队列最大事件数
    log_events = 3,          -- 每次输出事件数
    order = 'priority',      -- 排序模式: priority | content_len
    process_all_events = false
}
```

**排序模式**：

| 模式 | 说明 |
|------|------|
| SNORT_EVENTQ_PRIORITY | 按规则优先级排序 |
| SNORT_EVENTQ_CONTENT_LEN | 按匹配内容长度排序 |

---

## 6. 阈值与过滤机制

### NIDS threshold 语义

- `hit_count`：进程生命周期内该规则触发上报的总次数
- `window_match_count`：当前 detection_filter 时间窗口内所有原始匹配次数（含被抑制次数）
- threshold 规则中值 N = 本窗口内第 N 次匹配触发上报

### Snort3 过滤机制

| 机制 | 规则示例 | 说明 |
|------|----------|------|
| Rate Filtering | `count: 10, seconds: 5` | 速率超限触发动作 |
| Thresholding | `type limit, track by_src, count 5, seconds 60` | 每分钟最多记录 5 次 |
| Suppression | `suppress gen_id 1, sig_id 2001, ip 192.168.1.0/24` | 忽略特定源 |

---

## 7. 多线程模型

### NIDS

- 双线程 Pipeline per NIC：CaptureThread + WorkerThread
- EventEngine 在 WorkerThread 中运行
- hit_count 是进程级累计（跨 WorkerThread）

### Snort3

- 每个数据包处理线程拥有独立的 event_queue
- **线程私有**：无锁设计
- **预分配内存池**：O(1) 分配

```cpp
static THREAD_LOCAL SF_EVENTQ* event_queue = nullptr;
```

---

## 8. 设计哲学对比

| 维度 | NIDS | Snort3 |
|------|------|--------|
| **输出目标** | SOA Service + NLog | 可插拔 Logger 插件 |
| **事件模型** | 单一 JSON event_info | Event 类 + Logger 插件 |
| **action 处理** | 嵌入 event_info | 独立 IpsAction 对象 |
| **队列管理** | 无独立队列，直传 | sfeventq 优先级队列 |
| **过滤机制** | detection_filter (hit_count/window_match_count) | rate/threshold/suppression |
| **多线程** | 进程级 hit_count | 线程本地 event_queue |
| **配置方式** | nids_conf.yaml (YAML) | snort.lua (Lua) |

---

## 9. 关键差异总结

1. **架构复杂度**：Snort3 的 Logger 插件系统更灵活，支持多种输出格式；NIDS 面向 SOA 上报做了简化。

2. **事件标识**：NIDS 用 sig_id（自研 SID），Snort3 用 GID:SID:REV 三元组（社区标准）。

3. **action 语义**：NIDS action 直接在 JSON 中，Snort3 是优先级化对象系统。

4. **计数模型**：NIDS hit_count 是进程级窗口累计，Snort3 阈值规则独立 per-rule。

5. **健康事件**：NIDS 有专门的 Health EventType (99)，Snort3 通过 IpsAction 体系统一处理。

---

## Sources

- [NIDS EventEngine](../sources/event_engine.md)
- [NIDS nids_conf.yaml](../sources/nids_conf.md)
- [Snort3 Event System](../sources/github-snort3-event.md)
- [Snort3 Actions](../sources/github-snort3-actions.md)
- [Snort3 Loggers](../sources/github-snort3-loggers.md)
