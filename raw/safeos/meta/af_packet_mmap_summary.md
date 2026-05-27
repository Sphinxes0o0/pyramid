# SafeOS AF-PACKET + MMAP 实现总结与优化建议

> 文档版本: 1.0
> 更新日期: 2026/04/14
> 代码路径: `os-framework/servers/net/`, `external/net-cap/`, `libs/os_libs/libcore/`

---

## 一、当前实现架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    App (tcpdump / net-cap / nids)                   │
│                                                                      │
│  socket(AF_PACKET, SOCK_RAW, ETH_P_ALL)                            │
│       ↓                                                              │
│  packet_mmap_setup()                                                │
│       ├─ sys_dspace_create(4MB)         ← 创建 DSPACE 共享内存对象  │
│       ├─ grant_shm_to_net()             ← seL4 IPC 授权给 NSv      │
│       └─ setsockopt(PACKET_RX_RING)     ← 配置 ring 参数            │
│       ↓                                                              │
│  mmap() ◄──────────────────────────────── 映射同一块 DSPACE         │
│       ↓                                                              │
│  poll()/select() ◄─────────────────────── 等待数据就绪通知           │
│       ↓                                                              │
│  rb_read() / process_packet() ◄─────────── 从 ring buffer 读包      │
│       ↓                                                              │
│  pcap file / decoder pipeline                                        │
└─────────────────────────────────────────────────────────────────────┘
                              ↓ seL4 IPC
┌─────────────────────────────────────────────────────────────────────┐
│                    NSv (lwIP 用户态网络栈)                           │
│                                                                      │
│  packet_mmap_set_ring()          ← 映射 App 的 DSPACE 到 NSv 地址   │
│       ↓                                                               │
│  tpacket_recv callback            ← lwIP 收到包时调用                 │
│       ├─ rb_write_tpacket()      ← 写入 ring buffer (tp_status=USER)│
│       └─ API_EVENT(RCVPLUS)      ← 唤醒 select/poll                  │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                    seL4 Microkernel                                 │
│  ├─ DSPACE 对象管理 (创建、映射、授权、撤销)                         │
│  ├─ seL4 IPC (控制信息传递)                                          │
│  └─ seL4 Notification (RX/TX 事件通知)                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、核心组件关系

| 组件 | 位置 | 职责 |
|------|------|------|
| `packet_mmap.c` (NSv侧) | `servers/net/src/` | Ring设置、tpacket_recv回调、event callback |
| `packet_mmap.h` | `servers/net/include/nsv/` | packet_mmap_info 结构体、常量定义 |
| `dspace.c` | `libs/os_libs/libcore/src/` | DSPACE 系统调用实现 |
| `ds_ring.c` | `libs/os_libs/libcore/src/` | DS-RING 共享环 |
| `ringbuffer.c` | `libs/os_libs/libcore/src/` | ringbuf 读写实现 |
| `net_cap.c` / `packet_mmap.c` (App侧) | `external/net-cap/` | App侧抓包实现 |
| `tcpdump` | `os-framework/apps/tcpdump/` | tcpdump App |

---

## 三、数据结构

### DSPACE 布局 (4MB)

```
Offset 0x0:
├─ struct packet_mmap_info (32 bytes)  ← 元数据
├─ struct ringbuf (管理读写索引)        ← ring buffer 头部
└─ TPACKET 帧循环队列                   ← 数据区 (tp_frame_size × 1024)
```

### TPACKET 帧格式

```
┌────────────┬──────────────────────────┬─────────────────────────────┐
│ tpacket_hdr│      Ethernet Frame      │         Padding             │
│  (24 B)    │     (tp_len bytes)       │   (2048 - 24 - tp_len B)   │
└────────────┴──────────────────────────┴─────────────────────────────┘
```

### 关键常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `DEFAULT_TP_FRAME_SIZE` | 2048 | 每帧大小 |
| `DEFAULT_TP_FRAME_NR` | 1024 | 帧总数 |
| `DEFAULT_TP_BLOCK_SIZE` | 4096 | block 大小 |
| `DEFAULT_DSPACE_SIZE` | 0x400000 (4MB) | DSPACE 默认大小 |

---

## 四、关键流程

### 收包路径

```
NIC DMA → used_rx_buf_ring → nic_rx_thread → rx_callback → vnet_if.input
    → ethernet_input() → raw_afpacket_input() → tpacket_recv()
    → rb_write_tpacket() → API_EVENT(RCVPLUS) → poll()/select() 唤醒
    → App: rb_read() → pcap file
```

### 内存共享流程

```
App                              NSv
  │                               │
  ├─ sys_dspace_create(4MB) ──────┼─ PSv 创建 dspace_t
  │                               │
  ├─ grant_shm_to_net() ──────────┼─ seL4 IPC 授权
  │                               │
  │                               ├─ sys_dspace_map() 映射到NSv地址
  │                               │
  ├─ mmap() 映射同一 DSPACE ──────┼─ (通过 seL4 共享页)
```

---

## 五、可优化点

### 1. 架构边界问题（高优先级）

| 问题 | 当前状态 | 优化方向 |
|------|----------|----------|
| **内部头文件暴露** | `nsv/packet_mmap.h` 通过 CMakeLists.txt 直接暴露给 App | 创建 `include/nsv/packet_mmap_abi.h` 稳定ABI层 |
| **packet_mmap 代码重复** | `packet_mmap.c` 在 net-cap 和 tcpdump 中各有一份 | 抽取为 `libpacket_mmap` 共享库 |
| **App 依赖 NSv 内部实现** | tcpdump 直接 `#include servers/net/include` | 通过公共接口层隔离 |

### 2. 功能缺失（中等优先级）

| 问题 | 影响 | 优化方向 |
|------|------|----------|
| **仅支持 TPACKET_V1** | 缺乏 TPACKET_V3 的超时回收机制 | 实现 `PACKET_VERSION` setsockopt |
| **仅支持 RX Ring** | 无法做发送环 (`PACKET_TX_RING` 未实现) | 按需实现 TX Ring |
| **时间戳未填充** | `tp_sec`/`tp_usec` 固定为 0 | 使用实际收包时间戳 |
| **参数硬编码** | `tp_frame_size` 等不可自定义 | 支持运行时配置 |
| **无零拷贝** | pbuf → ring buffer → pcap 多次拷贝 | 考虑直接写入文件 |

### 3. API 设计问题（高优先级）

| 问题 | 当前 | 应该 |
|------|------|------|
| **函数命名空间** | `packet_mmap_*` 在全局 | `nsv_packet_mmap_*` 或在 `nsv::` 命名空间 |
| **错误码不一致** | 有的返回 errno，有的返回 -1 | 统一错误处理方式 |
| **DSPACE API 暴露** | App 需要知道 `dspace_info` 等内部结构 | 封装为更简单的接口 |

### 4. 文档和可维护性问题（中等优先级）

| 问题 | 优化方向 |
|------|----------|
| **缺乏 API 文档** | 补充函数注释、参数说明、返回值 |
| **架构文档分散** | 集中在 `docs/packet_mmap_design.md` 一处 |
| **缺少测试** | 添加 unit test 和 integration test |

### 5. 长期架构优化

```
应该分离为:
┌─────────────────────────────────────────┐
│  App (tcpdump / net-cap / nids)        │
│  只通过稳定 ABI/API 调用                │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  libpacket_mmap (公共库)                 │
│  - packet_mmap_setup()                  │
│  - process_packet()                      │
│  - 稳定的数据结构 (packet_mmap_info)     │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│  NSv 服务                               │
│  - packet_mmap_set_ring() (private)     │
│  - tpacket_recv callback (private)       │
│  - DSPACE 管理 (private)                 │
└─────────────────────────────────────────┘
```

---

## 六、nids 适配 SafeOS 可行性

### 功能层面

| 模块 | 功能 | SafeOS 兼容性 |
|------|------|--------------|
| **Decoder** (ipv4.cc, tcp.cc, udp.cc 等) | 纯 packet 数据解析 | ✅ 兼容 |
| **Preprocess** (packet_meta.cc) | packet 元数据提取 | ✅ 兼容 |
| **Flow hash** (flow_hash.cc) | 流哈希计算 | ✅ 兼容 |
| **Stats collector** | 统计收集 | ✅ 兼容 |
| **af_packet_mmap capture** | 从 ring buffer 读包 | ❌ 需重写 SafeOS 后端 |

### 障碍

1. **nids 当前期望标准 Linux AF-PACKET API**，而 SafeOS 使用自定义 DSPACE 方案
2. **nids CMakeLists.txt 没有 SafeOS os-framework 头文件路径**
3. **nids 无法链接 `core` 库**（包含 packet_mmap.c 实现）

### 结论

nids decoder 部分功能完全兼容，但 capture 层需要为 SafeOS 实现专用 backend，使用 `packet_mmap_setup()` + `rb_read()` API。

---

## 七、总结

| 方面 | 评价 |
|------|------|
| **架构合理性** | ⚠️ 中等 — 自定义 DSPACE 方案可行，但架构边界不清晰 |
| **API 稳定性** | ❌ 差 — 内部结构暴露给 App，无稳定 ABI 层 |
| **代码复用** | ❌ 差 — packet_mmap.c 代码重复 |
| **功能完整性** | ⚠️ 基本可用 — TPACKET_V1 RX Ring 可工作，但缺高级特性 |
| **可维护性** | ⚠️ 中等 — 文档分散，缺少测试 |
| **对外适配** | ❌ 差 — nids 等外部 App 无法直接使用 |

**核心建议**：优先创建稳定 ABI 层 (`include/nsv/`)，抽取 `libpacket_mmap` 共享库，解决头文件暴露和代码重复问题。
