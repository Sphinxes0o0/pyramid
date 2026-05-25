---
type: entity
tags: [safeos, nsv, af-packet, tpacket, dspace, ring-buffer, packet-mmap, seL4]
created: 2026-05-25
sources: [safeos-architecture]
---

# SafeOS AF-PACKET + TPACKET — 抓包实现

## 定义

SafeOS AF-PACKET 抓包实现是一个**自定义混合方案**：使用 TPACKET 数据格式作为 App 与 NSv 之间的 wire protocol，通过 DSPACE 共享内存实现跨进程通信，由 lwIP `raw_afpacket` 钩子触发数据包捕获。它并非 Linux 标准 AF-PACKET/TPACKET 的移植。

> **核心区别：** Linux 的 PACKET_MMAP 由内核态管理 ring buffer；SafeOS 由用户态 NSv (lwIP) 直接写入共享内存，绕过了内核。

---

## 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     APP (net-cap / tcpdump)                     │
│  socket(AF_PACKET, SOCK_RAW, ETH_P_ALL)                        │
│  packet_mmap_setup()                                            │
│    ├─ sys_dspace_create(4MB) 创建DSPACE对象                  │
│    └─ grant_shm_to_net() 通过seL4 IPC授权给NSv                │
│  setsockopt(PACKET_RX_RING, &tpacket_req3)                     │
│  mmap() ──── 映射同一块DSPACE                                   │
│  poll()/select() 等待数据                                       │
│  process_packet() ── rb_read() → pcap file                      │
└────────────────────────────┬────────────────────────────────────┘
                             │ seL4 IPC (sys_dspace_*)
┌────────────────────────────▼────────────────────────────────────┐
│                     NSv (Network Server — lwIP)                 │
│                                                                  │
│  PACKET_RX_RING setsockopt → packet_mmap_set_ring()             │
│    ├─ sys_dspace_map(app_ds) 映射到NSv虚拟地址空间               │
│    ├─ 初始化 packet_mmap_info (元数据区)                         │
│    ├─ 初始化 ringbuf (读写索引区)                               │
│    └─ 注册 callback:                                              │
│         sock->conn->pcb.raw_afpacket->recv = tpacket_recv        │
│         sock->conn->callback = packet_mmap_event_callback         │
│                                                                  │
│  tpacket_recv(arg, pcb, pbuf, addr) ←── lwIP收到数据包时调用     │
│    ├─ rb_write_avail(rx_buf_ring) > tp_frame_size?               │
│    ├─ rb_write_tpacket() 写入 ring buffer                        │
│    │    packet_hdr.tp_status = TP_STATUS_USER                     │
│    └─ API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0)                   │
│         → sock->rcvevent++ → select()/poll() 被唤醒             │
└─────────────────────────────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│                     seL4 Microkernel                            │
│  ├─ DSPACE 对象管理: 创建、映射、授权、撤销                      │
│  ├─ seL4 IPC 传递控制信息 (非数据包本身)                        │
│  └─ 不理解 TPACKET 语义，仅做内存页映射                          │
└─────────────────────────────────────────────────────────────────┘
```

### 收包数据流

```
网卡 → ethernet_input() → raw_afpacket_input(pcb, pbuf)
                              ↓
                        遍历 raw_afpacket_pcbs 链表
                              ↓
                        pcb->recv(pcb->recv_arg, pcb, pbuf, NULL)
                              ↓
                        tpacket_recv() 写入 ring buffer
                              ↓
                        API_EVENT(RCVPLUS) 唤醒 App
                              ↓
                        App: mmap 读取 → pcap 文件
```

---

## DSPACE 共享内存布局

默认 DSPACE 大小: **4MB (0x400000)**

```
DSPACE (mmap 到 App 和 NSv 各自虚拟地址空间)
┌──────────────────────────────────────────────────────────────────┐
│  Offset 0x0                                                       │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ struct packet_mmap_info (固定大小，约 32 字节)                │  │
│  │  ├─ pid              : pid_t                               │  │
│  │  ├─ socket           : int                                 │  │
│  │  ├─ tp_frame_size    : unsigned int                          │  │
│  │  ├─ rx_buf_offset    : uint16_t  (指向 ringbuf)            │  │
│  │  ├─ loop_recevent    : volatile uint16_t  (本轮已读帧数)    │  │
│  │  ├─ recved_packets   : volatile uint16_t  (总待读帧数)      │  │
│  │  └─ netconn          : void*    (lwIP netconn 指针)         │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Offset = RX_BUF_RING_OFFSET (sizeof(packet_mmap_info))          │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ struct ringbuf (管理读写索引)                               │  │
│  │  ├─ buf.offset : offset 到数据缓冲区                       │  │
│  │  ├─ size       : ring buffer 总大小                        │  │
│  │  ├─ ridx       : 读索引 (App 读取后移动)                    │  │
│  │  ├─ widx       : 写索引 (NSv 写入后移动)                   │  │
│  │  └─ use_offset : 使用 offset 模式而非直接指针               │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Offset = 数据区起始 (RX_BUF_RING_OFFSET + sizeof(ringbuf))      │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                    TPACKET 帧 循环队列                      │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │ struct tpacket_hdr (24 字节)                        │  │  │
│  │  │  ├─ tp_status  : TP_STATUS_USER(1) / TP_STATUS_KERNEL(0)  │  │
│  │  │  ├─ tp_len     : 实际包长                           │  │  │
│  │  │  ├─ tp_snaplen : 最大可抓长度                       │  │  │
│  │  │  ├─ tp_mac     : MAC 头偏移 (= 24)                   │  │  │
│  │  │  ├─ tp_net     : 网络层偏移 (= 24)                   │  │  │
│  │  │  ├─ tp_sec     : 秒 (固定 0，未使用)                 │  │  │
│  │  │  └─ tp_usec    : 微秒 (固定 0，未使用)               │  │  │
│  │  ├─────────────────────────────────────────────────────┤  │  │
│  │  │ Ethernet Header + IP Header + Payload               │  │  │
│  │  │ (tot_len bytes)                                     │  │  │
│  │  ├─────────────────────────────────────────────────────┤  │  │
│  │  │ Padding (填充到 tp_frame_size = 2048)               │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  │                         × 1024 帧 (循环)                   │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## TPACKET 帧格式

### struct tpacket_hdr (Linux 标准定义)

```c
struct tpacket_hdr {
    unsigned long  tp_status;   // TP_STATUS_USER(1) = 可读, TP_STATUS_KERNEL(0) = 已读待回收
    unsigned int   tp_len;     // 实际以太网帧长度
    unsigned int   tp_snaplen; // 最大可抓长度
    unsigned short tp_mac;     // MAC 头偏移 (固定 = sizeof(tpacket_hdr) = 24)
    unsigned short tp_net;     // 网络层偏移 (固定 = sizeof(tpacket_hdr) = 24)
    unsigned int   tp_sec;     // 秒 (未使用，固定 0)
    unsigned int   tp_usec;    // 微秒 (未使用，固定 0)
};
```

### 帧状态机

```
NSv 写入完成后:                      App 读取完成后:
tp_status = TP_STATUS_USER ────────→ tp_status = TP_STATUS_KERNEL
     (1)                                    (0)
     ↑                                      │
     └────────── poll()/select() 可读 ←──────┘
```

### 一帧完整内存布局 (tp_frame_size = 2048)

```
┌────────────┬──────────────────────────┬─────────────────────────────┐
│ tpacket_hdr│      Ethernet Frame       │         Padding              │
│  (24 B)    │     (tp_len bytes)        │   (2048 - 24 - tp_len B)     │
└────────────┴──────────────────────────┴─────────────────────────────┘
```

---

## 默认参数

| 常量 | 值 | 说明 |
|------|-----|------|
| `DEFAULT_TP_FRAME_SIZE` | 2048 | 每帧大小 (bytes) |
| `DEFAULT_TP_FRAME_NR` | 1024 | 帧总数 |
| `DEFAULT_TP_BLOCK_SIZE` | 4096 | block 大小 |
| `DEFAULT_TP_BLOCK_NR` | 512 | block 数量 |
| `DEFAULT_DSPACE_SIZE` | 0x400000 (4MB) | DSPACE 总大小 |

---

## 代码核心路径

### App 侧设置 (packet_mmap_setup)

```c
// external/net-cap/packet_mmap.c
int packet_mmap_setup(int socket, struct dspace_info *dspace_info,
                      struct packet_mmap_info **packet_info_app)
{
    dspace_attr_t attr = DSPACE_ATTR_CACHE | DSPACE_ATTR_WRITE;
    // 1. 创建 DSPACE 共享内存对象
    sys_dspace_create(DEFAULT_DSPACE_SIZE, attr, &(dspace_info->va), &(dspace_info->ds));

    // 2. 授权给 NSv (seL4 IPC)
    grant_shm_to_net(dspace_info->ds);

    // 3. 配置 TPACKET ring 参数
    struct packet_mmap_param param;
    param.req_u.req3.tp_frame_size = DEFAULT_TP_FRAME_SIZE;   // 2048
    param.req_u.req3.tp_frame_nr   = DEFAULT_TP_FRAME_NR;     // 1024
    // ... 4. 触发 NSv 侧的 ring 设置
    setsockopt(socket, SOL_PACKET, PACKET_RX_RING, &param, sizeof(param));
}
```

### NSv 侧 ring 设置 (packet_mmap_set_ring)

```c
// os-framework/servers/net/src/packet_mmap.c
int packet_mmap_set_ring(int socket, seL4_Word badge, ...)
{
    struct lwip_sock *sock = get_socket(socket);

    // 1. 将 App 的 dspace 映射到 NSv 地址空间
    sys_dspace_map(app_ds, &(dspace_info->va), ...);

    // 2. 填充 packet_mmap_info
    struct packet_mmap_info *packet_info = (struct packet_mmap_info *)(dspace_info->va);
    packet_info->rx_buf_offset = RX_BUF_RING_OFFSET;
    packet_info->pid           = pid;
    packet_info->socket        = socket;
    packet_info->tp_frame_size = req.tp_frame_size;
    packet_info->netconn       = sock->conn;

    // 3. 初始化 ringbuf
    struct ringbuf *rbuf = packet_mmap_rbuf(packet_info);
    init_rx_buf_ring(rbuf_size, rbuf);

    // 4. 注册数据包接收回调
    sock->conn->pcb.raw_afpacket->recv_arg = packet_info;
    sock->conn->pcb.raw_afpacket->recv     = tpacket_recv;
}
```

### NSv 侧收包 (tpacket_recv)

```c
static uint8_t tpacket_recv(void *arg, struct raw_pcb *pcb,
                            struct pbuf *p_buf, const ip_addr_t *addr)
{
    struct packet_mmap_info *packet_node = (struct packet_mmap_info *)arg;
    struct ringbuf *rx_buf_ring = packet_mmap_rbuf(packet_node);

    if (rb_write_avail(rx_buf_ring) > packet_node->tp_frame_size) {
        rb_write_tpacket(rx_buf_ring, p_buf, packet_node->tp_frame_size);
        API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0);
    } else {
        LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.rawafpacket_mbox_errs);
    }
    return 0;
}
```

---

## DSPACE 机制详解

DSPACE 是 seL4 提供的跨进程共享内存抽象：

```
App                               NSv
 │                                 │
 ├─ sys_dspace_create(4MB) ────────┼── PSv (进程服务器) 创建 dspace_t
 │                                 │
 ├─ sys_dspace_grant(ds, net_ep) ──┼── seL4 IPC 授权 DSPACE 页给 NSv
 │                                 │
 │                                 ├─ sys_dspace_map(app_ds, &va) 映射到NSv地址空间
 │                                 │
 │                                 ├─ 写入 ring buffer (直接内存写入)
 │                                 │
 ├─ mmap() 映射同一 DSPACE ─────────┤── (通过 seL4 共享页实现)
 │                                 │
 ├─ rb_read() 读取 ─────────────────┤
 │                                 │
 ├─ close(fd) ──────────────────────┼─ sys_dspace_revoke() 撤销授权
 └─ munmap() ───────────────────────┘
```

---

## 与 Linux PACKET_MMAP 的核心区别

| 方面 | Linux 标准实现 | SafeOS 实现 |
|------|---------------|-------------|
| **运行环境** | 内核态 ring buffer | 用户态 NSv 直接写入共享内存 |
| **内存来源** | `mmap(/dev/mem)` 或 `vmalloc` | seL4 DSPACE 对象 |
| **写入方** | 内核协议栈 | NSv (lwIP 用户态进程) |
| **状态同步** | 内核/用户态共享内存 | App/NSv 共享 DSPACE |
| **通知机制** | 内核 `poll()` 唤醒用户态 | lwIP `API_EVENT()` → `select()` |
| **零拷贝** | TPACKET_V3 + PACKET_FANOUT 可实现 | 无零拷贝，pbuf→ringbuf→pcap 两次拷贝 |
| **TPACKET_VERSION** | 支持 V1/V2/V3 | 无 (未调用 PACKET_VERSION) |
| **TPACKET_V3 特性** | 超时回收、private area | 不支持 |
| **PACKET_TX_RING** | 支持发送环 | 未实现 |

---

## 实现本质澄清

### 不是标准 TPACKET，是自定义混合方案

1. **TPACKET 结构体仅作为数据格式约定**
   - `struct tpacket_hdr` 的字段布局被 App 和 NSv 共同遵守
   - `tp_status` 的双状态语义 (USER/KERNEL) 被复用

2. **缺乏标准 TPACKET 的关键机制**
   - 无 `PACKET_VERSION` setsockopt 调用
   - `tp_retire_blk_tov` 字段未使用 (TPACKET_V3 超时回收依赖此字段)
   - `tpacket_req3` 仅填充 V1 所需字段，V3 特有字段均为 0

3. **灵魂是 DSPACE + lwIP raw_afpacket**
   - DSPACE 提供了跨进程的共享内存能力
   - lwIP 的 `raw_afpacket_input/output` 钩子实现了数据包的分发
   - TPACKET 只是一个"带内协议"，定义了在共享内存中交换的数据结构

---

## 当前限制

| 限制 | 说明 |
|------|------|
| **仅 RX Ring** | `PACKET_TX_RING` 未实现，无法做发送环 |
| **参数硬编码** | `tp_frame_size`、`tp_frame_nr` 等参数不可自定义 |
| **无 TPACKET_V3** | 无 `PACKET_VERSION` 设置，无超时回收机制 |
| **无零拷贝** | 数据在 pbuf → ring buffer → pcap 文件间多次拷贝 |
| **无流量控制** | 当 ring buffer 满时直接丢弃包 (`rawafpacket_mbox_errs` 统计) |
| **时间戳未填充** | `tp_sec`/`tp_usec` 固定为 0，未记录实际收包时间 |
| **内部头文件暴露** | `nsv/packet_mmap.h` 通过 CMakeLists.txt 直接暴露给 App |

---

## 相关概念

- [[entities/linux/safeos/safeos-nsv]] — NSv Network Server 完整分析
- [[entities/linux/safeos/safeos-network-implementation]] — SafeOS 网络实现完整分析
- [[entities/linux/lwip/lwip-packet-mmap]] — lwIP AF-PACKET mmap 实现
- [[entities/linux/lwip/lwip-raw-socket]] — RAW socket / AF-PACKET 绑定
- [[entities/linux/lwip/lwip-elem-ring]] — 无锁 ring buffer
- [[entities/linux/safeos/safeos-vdf-nids-relation]] — SafeOS 与 VDF nids 项目适配关系

## 来源详情

- [[sources/safeos-architecture]] — SafeOS Architecture & Design Documents
