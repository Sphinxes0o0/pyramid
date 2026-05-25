---
type: synthesis
tags: [nids, ids, architecture, security, safeos]
created: 2026-05-25
sources: [nids-design-docs]
---

# NIDS 当前架构分析

> 基于 9 份设计文档 + 配置文件综合分析，文档日期 2026-05-25

---

## 1. 整体架构

### 1.1 Pipeline 文字流程图

```
[NIC]
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  Frontend (CaptureThread)                                        │
│                                                                  │
│  [PcapSource] → [Preprocess (L2 only)] → [QueueWrite (SPSC)]   │
│       │                  │                                        │
│   libpcap           EtherType/VLAN            PacketSlot* 入队    │
│                    硬边界校验                                    │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼ (SPSC PacketPtrQueue)
┌─────────────────────────────────────────────────────────────────┐
│  Backend (WorkerThread)                                           │
│                                                                  │
│  [QueueRead] → [ProtocolDecoder] → [DetectionEngine]           │
│                     │                  │                          │
│              L3/L4 解析            规则匹配                       │
│               │                      │                            │
│              早期释放              [EventEngine]                  │
│         (Small/Std slot)           [Release]                    │
│                                   (Large slot)                   │
└─────────────────────────────────────────────────────────────────┘
                         │
                         ▼
                  [PacketPool 归还]
```

### 1.2 核心模块角色

| 模块 | 线程 | 职责 |
|------|------|------|
| `PcapSource` | CaptureThread | 从 NIC 抓包，写入 PacketSlot |
| `Preprocess` | CaptureThread | L2 解析（ether_type、vlan）+ 硬边界校验 |
| `ProtocolDecoder` | WorkerThread | L3/L4 解析（IPv4/IPv6/TCP/UDP/ICMP），输出 DecodeResult |
| `DetectionEngine` | WorkerThread | 规则匹配 + 行为检测（PortScan） |
| `EventEngine` | WorkerThread | 事件标准化、去重、限流、SOA 上报 |
| `HealthMonitor` | Main/Runtime | CPU/MEM 采样 + NIC 启停控制 |

---

## 2. 每个模块分析

### 2.1 PacketCapture（抓包）

**数据结构**

- `PacketSlot`：`data[]` 预分配缓冲区 + 元数据（l2_offset、l3_offset、l4_offset、l4_proto）
- `PacketPool`：预分配 Slot 池，支持 Small（256B）/Standard（2KB）/Large（16KB）三种规格
- `SPSC Queue`：单生产者单消费者无锁队列，连接 Frontend 与 Backend

**算法**

- **抓包后端**：libpcap（跨平台兼容保底），AF_PACKET（Linux 高性能）
- **BPF 过滤**：编译 BPF 字节码注入内核，减少无效报文
- **队列退避**（WorkerThread）：4阶段 CPU pause backoff，空载时 CPU 占用从 ~41% 降至接近 0

**与 Snort 对比**

| 维度 | Snort | 本 NIDS |
|------|-------|---------|
| 抓包后端 | libpcap / AFP | libpcap（保底）+ AF_PACKET（Linux）|
| 报文池 | 动态分配 | 预分配 SlotPool，三规格按包大小分桶 |
| 队列 | 无内置 | SPSC + 4阶段 backoff |

---

### 2.2 ProtocolDecoder（解码器）

**数据结构**

- `DecodeResult`：L3/L4 元数据（ether_type、ip_proto、src/dst IP:port、tcp_flags、ttl、dsize 等）
- `ParseStatus`：协议解析状态码（`Ok` / `Malformed` / `Truncated` / `ProtoError`）
- 子解码器注册表：`IPv4Decoder`、`IPv6Decoder`、`TcpDecoder`、`UdpDecoder`、`IcmpDecoder`、`ArpDecoder`

**算法**

- **分发**：DecoderCoordinator 根据 ether_type/ip_proto 路由到对应子解码器
- **校验**：IPv4 IHL 合法性、IPv6 固定头长度、L4 offset 合法性
- **TCP Options**：完整解析（kind/len/value），支持 MSS、WScale、SACK 等
- **扩展方式**：新增 decoder class + 注册，不修改主干分发逻辑（"注册即接入"）

**Slot 早期释放**

- Small/Standard slot 在 Decoder 完成后**立即归还 PacketPool**，payload 已 copy-by-value 到 DecodeResult
- Large slot 延迟到 DetectionEngine 完成后释放（content matching 需读 payload）

**与 Snort 对比**

| 维度 | Snort | 本 NIDS |
|------|-------|---------|
| 协议支持 | Ethernet/IPv4/IPv6/TCP/UDP/ICMP/ARP + 应用层 | 同左，首版聚焦 L3/L4 头，无 DPI |
| 扩展方式 | 插件链 | 注册即接入（新增 decoder + 注册）|
| TCP Options | 基础解析 | 完整解析（首版要求）|
| 分片重组 | stream5 preprocessor | **不支持**（范围外）|

---

### 2.3 DetectionEngine（检测引擎）

**数据结构**

- `PortGroupIndex`：按 `dst_port`（TCP/UDP）或 `icmp_type`（ICMP）精确索引，最多 32 组，每组最多 64 条规则
- `RuleOptionChain`：有序谓词链（proto/src_ip/dst_ip/src_port/dst_port/flags/ttl/dsize/fragbits/itype/icode），按编译期成本排序
- `DetectionFilterTable`：Open-Addressing 固定平面表，4096 slots，FNV-1a 哈希，线性探测最多 8 步
- `ContentMatcher`：封装 Aho-Corasick automaton + offset/depth 后处理

**算法**

1. **协议级快速拒绝**：`proto_mask_` 位掩码，无此协议规则则整批跳过
2. **PortGroupIndex 查找**：O(log N) 二分查找精确端口组，构建候选集（exact + any 去重）
3. **FastSelector 预筛**：协议/端口头匹配层，快速排除
4. **OptionChain 求值**：按成本排序顺序求值，任意节点失败即短路
5. **ContentMatcher**：AC automaton 一次遍历匹配所有 pattern，AND 语义（所有 patterns 都匹配才算命中）
6. **DetectionFilter**：阈值门控（type limit/both/threshold），按 `(gid, sid, track_ip)` 独立追踪
7. **PortScanInspector**：5 张独立哈希表 + Bloom filter 检测 SYN/UDP/ACK/FIN 扫描、主机扫、分布式扫描

**懒优化**

- `Lazy NowNs`：仅在 DetectionFilter 需要时才取时间戳
- `__builtin_prefetch`：FindOrInsert 第 0 步命中时预取下一 slot

**与 Snort 对比**

| 维度 | Snort 3 | 本 NIDS |
|------|---------|---------|
| 内容匹配 | hyperscan / AC | Aho-Corasick（Phase 2 升级 hyperscan）|
| 端口索引 | 端口组 | 精确索引（32 组 × 64 规则）|
| 检测过滤 | detection_filter + suppress | 同 detection_filter + alert_policy（未集成）|
| 行为检测 | 无内置 stream preproc | PortScanInspector（独立 C++ 引擎）|
| flowbits | 支持 | **不支持**（Phase 2）|
| PCRE | 支持 | **不支持**（Phase 2）|

---

### 2.4 RuleEngine（规则引擎）

**数据结构**

- `CompiledRule`：零动态分配运行态结构（固定字段数组）
- `RuleSet`：分发表（`tcp_rules`/`udp_rules`/`icmp_rules`/`ip_rules`/`any_rules` 向量）
- `RuleBank`：`shared_ptr<const RuleSet>`，快照语义
- `.nrb` 二进制格式：magic + version + rule_count + data_size + CRC32，固定 64B header per rule

**算法**

- **解析**：Snort3 文本格式子集（action/proto/src_ip/src_port/direction/dst_ip/dst_port）
- **编译**：`RuleSerializer::CompileAll()` 转换为 `CompiledRule`，固定字段无堆分配
- **热重载**：`Reload()` 原子替换 `bank_` 指针，检测路径继续用旧快照（shared_ptr 引用计数）
- **去重**：SID 重复时 last-write-wins

**与 Snort 对比**

| 维度 | Snort 3 | 本 NIDS |
|------|---------|---------|
| 规则格式 | Snort3 文本 + 光晕 | Snort3 文本子集 + 自研 `.nrb` 二进制 |
| 热重载 | 原子替换 | 同（shared_ptr 快照）|
| 内容匹配 | hyperscan | 编译到 AC automaton（Phase 2 升级）|
| flowbits | 完全支持 | **不支持** |
| pcre | 完全支持 | **不支持**（Phase 2）|
| 二进制序列化 | 无 | 自研 `.nrb` 格式（嵌入式友好）|

---

### 2.5 EventEngine（事件引擎）

**数据结构**

- `SecurityEvent`：`sig_id/proto/nic/src/dst/msg/action/hit_count/window_match_count/timestamp`
- `AlertPolicyTable`：4096 slot OA 表，支持 cooldown 跨窗口限速（独立于 detection_filter）

**算法**

- **来源**：规则命中（`ETH_IDS=12`） + HealthMonitor 健康状态
- **事件映射**：`event_type = SID / 1_000_000`（C++ 整数除法）
  - `1_xxx_xxx` → AttemptedDos（洪泛检测）
  - `2_xxx_xxx` → NetworkScan（端口/主机扫描）
  - `3_xxx_xxx` → AttemptedRecon（OS 指纹/工具指纹）
- **输出**：SOA `ReportCommonEventParam`（JSON event_info） + 本地 NLog
- **告警限速**：`alert_policy`（cooldown + threshold），尚未集成到生产路径

---

### 2.6 HealthMonitor（健康监控）

**数据结构**

- `SystemStatsTracker`：进程级 CPU/内存采样（SafeOS: libsys_insight；Linux: /proc）
- `HealthStateMachine`：Running ↔ Paused 状态机

**算法**

- **停止条件**（OR）：CPU ≥ stop_pct（默认 30%） 或 MEM ≥ stop_kb（默认 80MB）
- **恢复条件**（AND）：CPU < resume_pct（16%） 且 MEM < resume_kb（56MB）
- **防抖**：持续超阈值 5s 才触发停止（stop_confirm_sec 窗口）
- **冷却**：Paused → Running 至少等待 10s（cooldown_sec）
- **滞后区间**：`resume_threshold < x < stop_threshold` 时两者均为 false，防止频繁振荡
- **控制动作**：直接调用 `Capture::StopNic/StartNic`，同步返回后才更新状态

---

## 3. 规则语言

### 3.1 语法（Snort3 子集）

```
action protocol src_ip src_port direction dst_ip dst_port (options)
```

**支持的关键字**

| 类别 | 关键字 |
|------|--------|
| 头部 | `action`, `proto`（tcp/udp/icmp/ip/any）, `src_ip`, `src_port`, `direction`（`->`/`<>`）, `dst_ip`, `dst_port` |
| 选项 | `msg`, `sid`, `gid`, `rev`, `priority`, `classtype`, `reference`, `metadata` |
| 内容匹配 | `content`（含 `nocase`, `offset`, `depth`） |
| 协议字段 | `ttl`, `id`, `fragbits`, `fragoffset`, `flags`, `seq`, `ack`, `window`, `dsize`, `itype`, `icode` |
| 阈值 | `threshold`（type limit/both/threshold, track by_src/by_dst, count N, seconds M）|

### 3.2 与 Snort 规则的兼容度

| 兼容性维度 | 状态 |
|-----------|------|
| Snort3 文本格式解析 | ✅ 基本支持 |
| Snort3 `detection_filter:` 关键字 | ✅ 规范化为 `threshold` |
| `content` + `nocase` + `offset` + `depth` | ✅ |
| `threshold` 三种模式（limit/both/threshold）| ✅ |
| `flowbits`（状态机）| ❌ 不支持 |
| `pcre`（正则）| ❌ 不支持（Phase 2）|
| `uricontent` / URI 检测 | ❌ 不在范围内 |
| `byte_test` / `byte_jump` | ❌ 不支持 |
| `stream` / `preprocessor` 指令 | ❌ 不涉及 |
| Snort3 `enforce_body` | ❌ 不支持 |
| 规则注释 `#` | ✅ |
| 多规则同报（`result.hits[]`）| ✅ |

**结论**：规则语法兼容度约为 **Snort3 功能集的 60-70%**（以文本规则为主，content 匹配、阈值过滤为核心）。不支持 flowbits 状态机和 PCRE，行为检测依赖独立的 C++ PortScanInspector。

---

## 4. 数据流（完整路径）

```
Packet 到达 NIC
    │
    ▼
┌──────────────────────────────────────────────────────┐
│ CaptureThread                                         │
│ 1. PcapSource: libpcap 回调接收 raw bytes            │
│ 2. Preprocess: 写入 PacketSlot.data[]                │
│    - 最小帧长校验（<14B → Drop）                      │
│    - VLAN: 二次 EtherType 校验                        │
│    - 填充 l2_offset/ether_type                       │
│ 3. QueueWrite: PacketSlot* 入 SPSC 队列              │
└──────────────────────────────────────────────────────┘
    │ (slot 所有权转移至 Backend)
    ▼
┌──────────────────────────────────────────────────────┐
│ WorkerThread                                         │
│ 4. QueueRead: 从 SPSC 队列 取 PacketSlot*            │
│ 5. ProtocolDecoder.Decode(slot):                     │
│    - L3: IPv4/IPv6 解析，填充 src/dst IP            │
│    - L4: TCP/UDP/ICMP 解析，填充 port/flags/ttl     │
│    - 标注 parse_status                               │
│    - [Early Release] Small/Std slot → 归还 Pool     │
│ 6. DetectionEngine.Inspect(decode_result):           │
│    a. proto_mask_ 快速拒绝                           │
│    b. PortGroupIndex 查候选集（exact + any）         │
│    c. OptionChain 谓词求值（proto/ip/port/flags…）   │
│    d. ContentMatcher: AC 多模式 content 匹配         │
│    e. detection_filter: 阈值门控（limit/both/th）   │
│    f. PortScanInspector: 行为级扫描检测（独立路径） │
│ 7. EventEngine.Process(decode, detection_result):    │
│    - event_type 推导（event_type = SID / 1_000_000） │
│    - 去重/聚合/限流                                 │
│    - SOA SubmitReport (JSON event_info)              │
│    - NLog 本地写入                                   │
│ 8. [Release Large slot] → 归还 PacketPool            │
└──────────────────────────────────────────────────────┘
    │
    ▼
PacketPool（循环复用）
```

---

## 5. 当前能力矩阵

### 5.1 支持的协议

| 协议层 | 支持情况 |
|--------|---------|
| Ethernet | ✅ |
| VLAN（单/双标签）| ✅（Frontend L2 校验）|
| IPv4 | ✅ |
| IPv6 | ✅（固定头，不解析扩展头）|
| TCP | ✅（含完整 Options 解析）|
| UDP | ✅ |
| ICMP | ✅（含 type/code 匹配）|
| ARP | ✅ |
| SCTP / DCCP | ❌ |
| 应用层（HTTP/DNS 等）| ❌（无 DPI，不在范围内）|

### 5.2 检测类型

| 检测类型 | 实现方式 | 备注 |
|---------|---------|------|
| 洪泛检测（Flood）| `threshold: type both` 规则 | GID 103/104 |
| 端口扫描 | 文本规则（exact port）+ PortScanInspector | GID 107 + SID 2002001 |
| 主机扫描（sweep）| PortScanInspector | SID 2002003 |
| 分布式端口扫描 | PortScanInspector（反向 key）| SID 2002005 |
| ICMP 洪泛/扫描 | `itype` 精确匹配规则 | GID 104/106 |
| nmap 工具指纹 | 测试规则（GID 108/109/110）| 仅测试用 |
| 行为级扫描检测 | C++ PortScanInspector（Bloom filter）| 独立于文本规则 |
| 分片攻击（fragbits）| `fragbits` 关键字 | ✅ |
| TTL 范围检测 | `ttl` 关键字 | ✅ |

### 5.3 性能指标

| 指标 | 目标值 | 说明 |
|------|--------|------|
| 吞吐量 | 单网卡 100Mbps（百兆）| 文档设计目标 |
| P99 处理延迟 | < 100ms | 全链路 |
| CPU 占用 | < 10%（正常负载）| 进程级 |
| Health 停止阈值 | CPU ≥ 30% 或 RSS ≥ 80MB | 持续 5s 后触发 |
| 健康恢复阈值 | CPU < 16% 且 RSS < 56MB | cooldown 10s 后允许 |
| WorkerThread 空载 CPU | ≈ 0%（原 busy-loop 为 41%）| 4阶段 backoff |
| Queue 统计节流 | 每 64 包原子更新一次 | 降低 false sharing |
| DetectionFilter 表 | 4096 slot × 40B = 160KB | OA 平面表，FNV-1a |
| PortScan 表 | 5 张哈希表，每 entry 32B（压缩至 cache-line 优化）| Bloom filter 去重 |

---

## 6. 已知限制/不足

### 6.1 功能性限制

| 限制 | 严重度 | 说明 |
|------|--------|------|
| 无 flowbits 状态机 | 中 | 无法支持有状态检测场景，Snort 核心功能缺失 |
| 无 PCRE 正则 | 中 | 内容匹配仅支持固定字符串，Phase 2 计划 |
| 无应用层 DPI | 低 | 聚焦 L3/L4，边界清晰 |
| 无分片重组 | 中 | stream5 不在范围，无法检测分片逃避 |
| `alert_policy` 未集成 | 低 | 设计存在但未上生产路径 |
| 无规则版本管理/审计 | 中 | Phase 2 计划 |
| 无 Web UI / 规则编辑器 | 低 | Phase 2 计划 |

### 6.2 检测能力限制

| 限制 | 说明 |
|------|------|
| PortScanInspector vs 文本规则双层 | 行为扫描用 C++ 独立实现，与文本规则体系分离 |
| nmap OS 探针指纹 | 仅测试规则，无实际生产规则（GID 110）|
| 扫描工具指纹（GID 108）| 仅有测试规则（ipEye/myscan/synscan）|
| `pcre` 缺失 | 无法检测复杂模式（如 `content:"foo"; pcre:"/bar\\d+/i"`）|

### 6.3 架构层限制

| 限制 | 说明 |
|------|------|
| 目标仅 100Mbps | 非万兆设计，架构扩展有上限 |
| Runtime 三层边界未完全拆分 | Runtime 类承载全部编排职责（目标为 NicRuntimeUnit/PipelineDirector/DegradeController 拆分）|
| 双线程模型固定 | Frontend+Backend 双线程，未抽象为通用线程基类 |
| 无跨 NIC 全局降级协调 | 当前为 per-NIC 局部降级 |
| Aho-Corasick 非 hyperscan | 预计 Phase 2 升级到 Intel hyperscan 以支持更大规则集 |

### 6.4 规则体系限制

| 限制 | 说明 |
|------|------|
| SID 私有段（100k-999k）| 不对接 ET/VRT 公开规则库，需自维护 |
| 规则热重载后旧 bank 仍服务 | snapshot 语义，检测不中断但新规则不立即生效 |
| `.nrb` 格式为自研 | 与 Snort 不兼容，离线编译工具链较薄 |

---

## 7. 附录：配置摘要

**nids_conf.yaml 关键配置**

```yaml
global:
  log_level: 2
  rules:
    text_rule_files: ["/etc/nids/nids.rules"]
    max_rules: 0  # unlimited
  health:
    nids_cpu_stop_pct: 30.0
    nids_cpu_resume_pct: 16.0
    nids_mem_stop_kb: 81920       # 80MB
    nids_mem_resume_kb: 57344     # 56MB
    sample_interval_ms: 1000
    stop_confirm_sec: 5
    cooldown_sec: 10

nics:
  - name: "PFE.VLAN1"
    small_slots_numbers: 2048
    std_slots_numbers: 2048
    queue_size: 2048
    capture_backend: "af"
```

**生产规则集（nids.rules，11 条）**

- GID 103/104：洪泛检测（SYN Flood/ACK Flood/UDP Flood/ICMP Echo Flood/Large UDP Flood）
- GID 106：源速率异常（ICMP Echo Rate，by_src）
- GID 107：服务端口扫描（FTP/SSH exact port）
- SID 全部位于私有段 `1_xxx_xxx`（AttemptedDos）和 `2_xxx_xxx`（NetworkScan）
