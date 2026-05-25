---
type: synthesis
tags: [nids, snort3, packet-capture, afpacket, daq, comparison]
created: 2026-05-25
sources: [nids-current-architecture, safeos-packet-mmap, snort3-claude]
---

# NIDS vs Snort3 Packet Capture 模块对比

---

## 1. 架构总览

| 维度 | 本 NIDS | Snort3 |
|------|---------|--------|
| **抽象层** | `capture_backend: "af"` 硬编码切换 | DAQ (Data Acquisition) 框架，插件化 |
| **抓包后端** | libpcap（保底）+ AF_PACKET（Linux） | 动态加载 DAQ 模块（pcap/afpacket 等），bundled 仅 `daq_file` + `daq_hext` |
| **配置语言** | YAML (`nids_conf.yaml`) | Lua (`snort.lua`) |
| **运行模式** | Frontend(CaptureThread) + Backend(WorkerThread) 双线程 | Main thread + Packet threads ("Pigs") + DAQ instance per thread |
| **多实例支持** | per-NIC 独立进程 | DAQ multi-instance ( `--DAQ-total-instances` ) |

---

## 2. 核心架构对比

### 本 NIDS — AF-PACKET + TPACKET 混合方案

```
App (net-cap / tcpdump)
  socket(AF_PACKET, SOCK_RAW, ETH_P_ALL)
  sys_dspace_create(4MB) → seL4 DSPACE 共享内存
  setsockopt(PACKET_RX_RING)
  mmap() ──── 映射同一块 DSPACE
  poll()/select() 等待
  rb_read() → pcap file
         ↑
    seL4 IPC (sys_dspace_*)
         ↑
  NSv (lwIP) ──── PACKET_RX_RING setsockopt
    ├─ packet_mmap_set_ring()
    ├─ tpacket_recv() callback
    └─ rb_write_tpacket() → TP_STATUS_USER(1)
```

**关键特性**：
- TPACKET 结构体仅作为 App↔NSv 间的 wire protocol 数据格式
- seL4 DSPACE 替代 Linux `vmalloc`/`mmap(/dev/mem)` 提供跨进程共享内存
- NSv (lwIP) 用户态进程直接写入共享内存，绕过 Linux 内核协议栈
- Ring buffer 状态机复用 TP_STATUS_USER/KERNEL 双状态语义

### Snort3 — DAQ 框架

```
Snort Main Thread
  │
  ├─ SFDAQ::init() → daq_load_dynamic_modules()
  │    └─ 动态加载 $DAQ_DIR/*.so (pcap, afpacket, etc.)
  │
  ├─ SFDAQInstance::init() → daq_instance_instantiate()
  │    └─ 每 thread 一个 instance (thread-local)
  │
  └─ PacketThread (Pig)
       ├─ receive_messages(batch_size=64)
       │    └─ daq_instance_msg_receive() → DAQ_Msg_h[]
       ├─ Codec decode → Inspector eval → Detection
       └─ finalize_message(verdict) → daq_instance_msg_finalize()
```

**关键特性**：
- `SFDAQInstance` 是 thread-local 的，每个 packet thread 独立实例
- Batch receive：`daq_instance_msg_receive()` 批量获取 (default 64)
- DAQ message pool：`daq_config_set_msg_pool_size()` 预分配消息对象
- BPF filter：`daq_instance_set_filter()` 编译后注入内核

---

## 3. 内存布局与零拷贝

### 本 NIDS DSPACE 布局 (4MB)

```
Offset 0x0:
  struct packet_mmap_info (32B)
    pid, socket, tp_frame_size, rx_buf_offset, loop_recevent, recved_packets, netconn

Offset = RX_BUF_RING_OFFSET:
  struct ringbuf (读写索引: ridx/widx)

Offset = 数据区:
  struct tpacket_hdr (24B) + Ethernet + IP + Payload
  Padding → tp_frame_size = 2048
  × 1024 帧 (循环)
```

| 参数 | 值 |
|------|-----|
| `tp_frame_size` | 2048 (固定) |
| `tp_frame_nr` | 1024 |
| `tp_block_size` | 4096 |
| `tp_block_nr` | 512 |
| 总 DSPACE | 4MB |

**零拷贝状态**：**无零拷贝**。pbuf → ring buffer → pcap 文件，2次拷贝。

### Snort3 DAQ 内存模型

Snort3 本身不管理 ring buffer，内存模型由各 DAQ 模块自行实现：

- **pcap DAQ**：libpcap 内部管理，recvfrom() → 用户态 buffer
- **afpacket DAQ**（外部）：TPACKET_V3 ring buffer，mmap 可实现零拷贝
- **Message pool**：`DAQ_Msg_h` 批量预分配 (batch_size × 4)

```
DAQ 内部 buffer (DAQ 模块管理)
        ↓  (一次拷贝)
Snort 内部 DAQ_Msg_h message
        ↓  (Snort copy-on-handle)
Packet 解析 → Detection
```

**零拷贝状态**：取决于 DAQ 模块。afpacket DAQ 可达零拷贝；pcap DAQ 至少有 1 次 socket→用户态拷贝。

---

## 4. VLAN 支持

### 本 NIDS

- **VLAN 检测**：Frontend `Preprocess` 模块二次 EtherType 校验（单/双标签）
- **VLAN 数据结构**：`PacketSlot` 元数据中记录 `l2_offset`，VLAN 头在 `data[]` 中保留
- **VLAN 配置**：`nids_conf.yaml` 中 NIC 名 `PFE.VLAN1` 标识 VLAN 接口

```cpp
// Preprocess 中 VLAN 硬边界校验
if (eth_type == ETH_P_8021Q) {
    vlan_offset = l2_offset + 2;
    inner_eth_type = *(uint16_t*)(data + vlan_offset + 2);
    // 校验 inner_eth_type 合法性
}
```

### Snort3

- **VLAN 处理**：`SFDAQInstance::add_expected()` 中 `key.vlan_id` 传递
- **隧道协议支持**：DAQ 层声明 `DAQ_CAPA_DECODE_GTP/TEREDO/VXLAN/GRE/MPLS/GENEVE` 等
- **VLAN Bypass**：`get_tunnel_bypass(uint16_t proto)` 查询能力
- **Codec 层**：`src/protocols/vlan.h` 有专门 VLAN protocol decoder

| 能力 | 本 NIDS | Snort3 |
|------|---------|--------|
| 802.1Q 单标签 | ✅ | ✅ |
| 802.1ad 双标签 | ✅ | ✅ |
| VLAN 传递到检测引擎 | ✅ (l2_offset) | ✅ (layer::get_vlan_layer) |
| 隧道协议 (VXLAN/GRE/GTP) | ❌ | ✅ (DAQ 层) |

---

## 5. 硬件加速与性能

### 本 NIDS

| 特性 | 状态 | 说明 |
|------|------|------|
| AF_PACKET | ✅ 已实现 | `capture_backend: "af"` |
| TPACKET_V3 | ❌ | 未实现 PACKET_VERSION，无超时回收 |
| PACKET_TX_RING | ❌ | 未实现发送环 |
| 硬中断 coalescing | ❌ | 依赖 NIC 驱动 |
| RSS / 多队列 | ❌ | 非多线程抓包设计 |
| 零拷贝 | ❌ | pbuf→ringbuf→pcap 两次拷贝 |

**性能目标**：单网卡 100Mbps（百兆），非万兆设计。

### Snort3

| 特性 | 状态 | 说明 |
|------|------|------|
| DAQ 模块化 | ✅ | 动态加载，支持 afpacket/pcap/dump/等 |
| Batch receive | ✅ | `batch_size` default 64 |
| Multi-instance | ✅ | `DAQ_CAPA_MULTI_INSTANCE` |
| Hardware offload | ✅ via DAQ | 取决于 DAQ 模块（afpacket 可用 tpacket_v3） |
| Zero-copy | ✅ via DAQ | afpacket DAQ + mmap 可达零拷贝 |
| Inline mode | ✅ | `DAQ_MODE_INLINE`，支持 drop/block |

**Snort3 DAQ 统计指标**（`DAQStats`）：
`received`, `analyzed`, `dropped`, `filtered`, `outstanding`, `injected`, `verdicts[6]`, `rx_bytes`, `expected_flows`, `retries_*`, `sof/eof_messages`

---

## 6. 配置方式

### 本 NIDS (`nids_conf.yaml`)

```yaml
nics:
  - name: "PFE.VLAN1"
    small_slots_numbers: 2048   # 256B slot 数量
    std_slots_numbers: 2048     # 2KB slot 数量
    queue_size: 2048            # SPSC 队列深度
    capture_backend: "af"       # AF_PACKET 后端
```

**参数局限性**：
- `tp_frame_size` 硬编码 2048，不可自定义
- `tp_frame_nr` 硬编码 1024，不可自定义
- `capture_backend` 仅支持 `"af"` / `"pcap"` 字符串硬切换

### Snort3 (`snort.lua`)

```lua
daq = {
    module_dirs = { "/usr/local/lib/snort/daq" },
    inputs = { "eth0" },
    snaplen = 1518,
    batch_size = 64,
    modules = {
        { name = "afpacket", mode = "passive", variables = { ... } },
        { name = "pcap", mode = "read-file" }
    }
}
```

**参数灵活性**：
- `snaplen`：0-65535 可配置
- `batch_size`：1 以上可配置
- `module_dirs`：可指定多个 DAQ 搜索路径
- `variables`：DAQ 模块自定义 key=value 对

---

## 7. BPF 过滤

### 本 NIDS

- **BPF 支持**：编译 BPF 字节码注入内核（通过 libpcap 或 AF_PACKET setsockopt）
- **注入时机**：CaptureThread 初始化时
- **BPF 后端**：依赖 libpcap 的 `pcap_setfilter()` 或 AF_PACKET 的 `SO_ATTACH_FILTER`

### Snort3

- **BPF 支持**：通过 DAQ 模块的 `daq_instance_set_filter()`
- **注入时机**：`SFDAQInstance::init()` 中实例化时
- **BPF 后端**：由各 DAQ 模块实现（pcap DAQ → libpcap；afpacket DAQ → kernel）

---

## 8. 关键限制对比

| 限制项 | 本 NIDS | Snort3 |
|--------|---------|--------|
| TPACKET_V3 超时回收 | ❌ 未实现 | ✅ via afpacket DAQ |
| PACKET_TX_RING 发送环 | ❌ 未实现 | ✅ via afpacket DAQ |
| 零拷贝 | ❌ 无 | ✅ via afpacket DAQ + mmap |
| 多线程抓包 | ❌ 双线程仅 1 capture | ✅ DAQ multi-instance |
| 隧道协议卸载 | ❌ 无 | ✅ DAQ 层 GTP/VXLAN/GRE |
| 硬件 RSS 协同 | ❌ 无 | ✅ via DAQ 模块 |
| 参数可调性 | 差（硬编码多）| 好（Lua 全局配置）|
| 非 Linux 支持 | ✅ (seL4/DSPACE) | ❌ (DAQ 模块多依赖 Linux) |

---

## 9. 核心架构差异总结

```
本 NIDS                          Snort3
──────────────────────────────────────────────────────────────
用户态 NSv (lwIP) 直接写 ring   DAQ 模块抽象，内核/用户态皆有
seL4 DSPACE 跨进程共享内存       mmap(/dev/mem) 或 socket recvfrom
TPACKET 仅作数据格式约定         DAQ API 作抽象接口
无零拷贝（seL4 无 DMA）         afpacket DAQ 可零拷贝
参数硬编码多                     Lua 可配置，模块化
嵌入式 / seL4 专精              通用 IPS，支持多平台
```

### 设计哲学

- **本 NIDS**：深度定制，针对 SafeOS/seL4 环境优化，以 DSPACE 替代内核，用 TPACKET 格式做 wire protocol，代价是灵活性低、参数硬化。
- **Snort3**：通用框架，通过 DAQ 抽象层支持多种数据源，每种 DAQ 模块自行选择最优抓包方式（pcap 通用，afpacket 高性能），Lua 配置层提供完整可调性。

---

## 10. 关键文件索引

| 文件 | 作用 |
|------|------|
| `nids_conf.yaml` | NIDS YAML 配置 |
| `wiki/synthesis/nids-current-architecture.md` | NIDS 整体架构 |
| `wiki/entities/linux/safeos/safeos-packet-mmap.md` | SafeOS TPACKET 实现细节 |
| `github/snort3/src/packet_io/sfdaq.cc` | Snort3 DAQ 入口 |
| `github/snort3/src/packet_io/sfdaq_instance.cc` | Snort3 DAQ 实例封装 |
| `github/snort3/src/packet_io/sfdaq_module.cc` | Snort3 DAQ Lua 配置模块 |
| `github/snort3/daqs/daq_file.c` | Snort3 Bundled 文件 DAQ |
| `github/snort3/CLAUDE.md` | Snort3 构建/架构总览 |
