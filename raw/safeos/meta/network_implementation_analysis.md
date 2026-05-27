# SafeOS 网络实现深度分析

> 文档版本: 1.0
> 更新日期: 2026/04/13
> 代码路径: `/home/shiyang/nio/nt35/safeos/`

---

## 1. 整体架构

SafeOS 运行在 **seL4 微内核**上，网络栈完全在**用户态**实现，没有传统意义上的内核网络子系统。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Applications                                       │
│         net-cap / tcpdump / iperf / udpecho / user apps                   │
│              ↓ socket() / sendto() / recvfrom()                           │
└──────────────────────────────┬──────────────────────────────────────────────┘
                               │ seL4 IPC (sys_net_*)
┌──────────────────────────────▼──────────────────────────────────────────────┐
│                    NSv — Network Server (lwIP)                              │
│                                                                              │
│  ┌─────────────────┐   ┌─────────────────┐   ┌────────────────────────┐   │
│  │  App Event Loop │   │  NIC RX Thread  │   │    lwIP TCPIP Thread   │   │
│  │  (BSD syscalls) │   │ (RX path from   │   │                        │   │
│  │                 │   │  NIC driver)    │   │  TCP / UDP / RAW / IP  │   │
│  └────────┬────────┘   └────────┬────────┘   └────────────┬─────────────┘   │
│           │                     │                         │                 │
│  ┌────────▼────────────────────▼─────────────────────────▼─────────────┐   │
│  │                    vnet_if (struct netif)                            │   │
│  │              netif->output = etharp_output                            │   │
│  │              netif->linkoutput = ethif_link_output                   │   │
│  └──────────────────────────────┬────────────────────────────────────────┘   │
└─────────────────────────────────┼───────────────────────────────────────────┘
                                  │ elem_ring (shared memory)
┌─────────────────────────────────▼───────────────────────────────────────────┐
│                     NIC Driver (separate process)                            │
│            Reads from pending_tx_buf_ring, Writes to used_rx_buf_ring      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 核心组件

### 2.1 NSv (Network Server)

| 属性 | 值 |
|------|-----|
| 位置 | `os-framework/servers/net/` |
| 服务名 | `/net` (注册到 PSv) |
| 底层栈 | lwIP (用户态) |
| 线程数 | 3个: App Event Loop + NIC RX Thread + lwIP TCPIP Thread |

**初始化序列** (`main.c:6322`):

```c
net_resources_init()      // 初始化统计/缓存/互斥锁
    ↓
init_ds_ring()            // 创建 CMA 共享内存 + ds_ring
    ↓
create_nic_thread()       // 创建 nic_rx_thread
    ↓
netif_add(&vnet_if)      // 注册 lwIP netif
    ↓
sys_svc_reg("/net", ...)  // 注册 /net 服务
    ↓
event_loop()              // 进入主事件循环
```

### 2.2 lwIP 集成 (`external/lwip_ds_mcu/`)

**sys_arch_sel4.c** — seL4 适配层:

- `sys_mutex_new()` / `sys_sem_new()` 基于 seL4 notification
- `sys_mbox_new()` — tcpip 线程通信
- `lwip_malloc()` / `lwip_free()` 基于 `ds_ring_mem_alloc()` / `ds_ring_mem_free()`

**lwIP 配置** (`libs/util_libs/liblwip/default_opts/lwipopts.h`):

```c
LWIP_NETCONN           1   // Netconn API
LWIP_SOCKET            1   // BSD socket API
LWIP_RAW               1   // RAW socket (AF_PACKET依赖)
LWIP_TCPIP_CORE_LOCKING 1  // 核心锁
MEMP_NUM_NETCONN       1024
LWIP_SOCKET_NPOLL      1   // poll() 支持
LWIP_IPV6              0   // IPv6 禁用
```

---

## 3. 内存架构 — CMA + DS-RING

NSv 与 NIC 驱动之间通过 **CMA (Contiguous Memory Area) + elem_ring** 实现零拷贝 DMA 缓冲共享。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CMA Region (96MB) — NSv 与 NIC 共享                    │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    elem_ring (4个)                                    │  │
│  │  ┌───────────────┬───────────────┬───────────────┬───────────────┐  │  │
│  │  │empty_rx_buf   │used_rx_buf    │pending_tx_buf │used_tx_buf    │  │  │
│  │  │  (RX空缓冲)    │  (RX已收包)   │  (TX待发)      │  (TX已完成)   │  │  │
│  │  └───────────────┴───────────────┴───────────────┴───────────────┘  │  │
│  ├─────────────────────────────────────────────────────────────────────┤  │
│  │                         DMA Buffers (pbuf)                           │  │
│  │   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                   │  │
│  │   │  DMA Buf 0  │ │  DMA Buf 1  │ │  DMA Buf N  │  (每块 = 1576B)  │  │
│  │   └─────────────┘ └─────────────┘ └─────────────┘                   │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**DS-RING 初始化** (`main.c:3610`):

```c
static int init_ds_ring(void) {
    // 1. 从 CMA 分配 DMA 缓冲区 (96MB)
    sys_mem_map(getpid(), &cma.pa, &cma.va, CMA_SIZE, PAGE_DMA);

    // 2. 基于 CMA 创建 ds_ring (包含 desc/sring/cring)
    ds_ring = ds_ring_init(cma.va, cma.pa, CMA_SIZE, ds, pid, ...);

    // 3. 将 ds_ring 授权给 NIC 驱动
    sys_dspace_grant(ds, nic_ep, DSPACE_GRANT_SVC_EP, attr);
    sys_ds_ring_share(nic_ep, ds);
}
```

---

## 4. 收包路径 (RX)

```
NIC 驱动
   │
   │ DMA 收包到 DMA buffer (物理地址)
   │
   ▼
used_rx_buf_ring (elem_ring_put) ─────────────────┐
   │                                               │
   │ sel4_signal(nic_rx_ntfn)                      │
   │                                               │
▼                                               │
nic_rx_thread()                                    │
   │                                               │
   │ seL4_Recv(nsv_nic_ep)                        │
   │                                               │
▼                                               │
elem_ring_get(used_rx_buf_ring) → 获取 buffer PA    │
   │                                               │
   │ cma_pa_to_va() → 转换为虚拟地址               │
   │                                               │
▼                                               │
rx_callback(pbuf)                                  │
   │                                               │
   │ LOCK_TCPIP_CORE()                            │
   │                                               │
▼                                               │
vnet_if.input(p, &vnet_if) = ethernet_input()       │
   │                                               │
   ├── ETH_P_IP  → ip_input() → raw_input() ──────► raw_afpacket_input()
   │                    │                    │              │
   │                    │                    │         tpacket_recv()
   │                    │                    │              │
   ├── ETH_P_ARP → etharp_input()            │              │
   └── 其他          → 未知协议              │              │
                                            │              │
                        API_EVENT(conn, RCVPLUS) ──────────┘
                              │
                              │ select()/poll() 唤醒 App
                              ▼
                           App recvfrom()
```

**核心函数** (`main.c:4694`):

```c
void rx_callback(void *ctx) {
    struct pbuf *p = (struct pbuf *)ctx;
    while (p != 0) {
        struct pbuf *next = p->next;
        p->next = 0;
        p->tot_len = p->len;
        vnet_if.input(p, &vnet_if);  // → ethernet_input()
        p = next;
    }
}
```

**ethif_link_output (TX输出)** (`main.c:3728`):

```c
err_t ethif_link_output(struct netif *netif, struct pbuf *p) {
    if (!nic_ready) return ERR_OK;

    // 1. 从 CMA 分配 DMA buffer
    // 2. 复制 pbuf 到 DMA buffer
    // 3. elem_ring_put(pending_tx_buf_ring, pa)
    // 4. sel4_signal(nic_tx_ntfn) 通知 NIC
    return ERR_OK;
}
```

---

## 5. Socket API 映射

App 的 BSD socket 调用经 seL4 IPC 到达 NSv，映射到 lwIP:

| App syscall | NSv handler | lwIP 函数 |
|-------------|-------------|-----------|
| `socket()` | `sys_socket_nb()` | `lwip_socket()` |
| `bind()` | `sys_bind_nb()` | `lwip_bind()` |
| `listen()` | `sys_listen_nb()` | `lwip_listen()` |
| `accept()` | `sys_accept()` | `lwip_accept()` |
| `connect()` | `sys_connect_nb()` | `lwip_connect()` |
| `sendto()` | `sys_sendto_nb()` | `lwip_sendto()` |
| `recvfrom()` | `sys_recvfrom_nb()` | `lwip_recvfrom()` |
| `close()` | `sys_close_nb()` | `lwip_close()` |
| `select()` | `select_thread()` | `lwip_select()` |

**socket → netconn → pcb 映射**:

```
struct lwip_sock (sockets_priv.h)
    ├─ socket fd
    ├─ struct netconn *conn
    └─ packet_info (PACKET_MMAP 相关)

struct netconn
    ├─ enum netconn_type (NETCONN_TCP/UDP/RAW/PACKET)
    └─ union { tcp_pcb, udp_pcb, raw_pcb }
```

---

## 6. AF-PACKET 实现

### 6.1 raw_afpacket_input (RX)

`external/lwip_ds_mcu/src/core/raw.c:282`:

```c
void raw_afpacket_input(struct pbuf *p, struct netif *netif, uint16_t type) {
    // 遍历 raw_afpacket_pcbs 链表
    // 对每个匹配的 PCB:
    //   if (pcb->recv) pcb->recv(pcb->recv_arg, pcb, p, NULL);
}
```

### 6.2 raw_afpacket_output (TX)

`external/lwip_ds_mcu/src/netif/ethernet.c:380`:

```c
int ethernet_output(struct pbuf *p, struct netif *netif) {
    raw_afpacket_output(p, netif);  // 发送抓包通知
    return netif->linkoutput(netif, p);  // 实际发送
}
```

### 6.3 tpacket_recv (PACKET_MMAP 核心)

`os-framework/servers/net/src/packet_mmap.c:122`:

```c
static uint8_t tpacket_recv(void *arg, struct raw_pcb *pcb,
                            struct pbuf *p_buf, const ip_addr_t *addr)
{
    struct packet_mmap_info *packet_node = (struct packet_mmap_info *)arg;
    struct ringbuf *rx_buf_ring = packet_mmap_rbuf(packet_node);

    if (rb_write_avail(rx_buf_ring) > DEFAULT_TP_FRAME_SIZE) {
        // 写入 ringbuf: tpacket_hdr + payload + padding
        rb_write_tpacket(rx_buf_ring, p_buf, packet_node->tp_frame_size);
        // 唤醒 select/poll
        API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0);
    }
    return 0;
}
```

---

## 7. 网络相关的 Ring 结构

| Ring | 方向 | 生产者 | 消费者 | 说明 |
|------|------|--------|--------|------|
| `empty_rx_buf_ring` | RX空缓冲 | NSv | NIC | NIC 从中取空buffer用于收包 |
| `used_rx_buf_ring` | RX收包 | NIC | NSv | NIC 收包后放入，NSv 取走处理 |
| `pending_tx_buf_ring` | TX待发 | NSv | NIC | NSv 放入待发数据，NIC 取走发送 |
| `used_tx_buf_ring` | TX完成 | NIC | NSv | NIC 发完后放入，NSv 回收buffer |

**elem_ring** (`core/elem_ring.h`) — 无锁环形缓冲:

```c
struct elem {
    uint64_t pa;    // 物理地址
    uint32_t len;   // 长度
};

static inline union elem elem_ring_get(struct elem_ring *ring) {
    // 使用 dmb(ish) 内存屏障保证顺序
    // 返回 ring[get_idx++ % size]
}
```

---

## 8. 协议支持

| 协议 | 状态 | 说明 |
|------|------|------|
| IPv4 | ✅ | 完整 TCP/UDP/ICMP/IGMP/ARP |
| IPv6 | ❌ | `LWIP_IPV6 = 0` |
| TCP | ✅ | SOCK_STREAM |
| UDP | ✅ | SOCK_DGRAM |
| RAW IP | ✅ | SOCK_RAW (AF_INET) |
| AF_PACKET | ✅ | SOCK_RAW (ETH_P_ALL) |
| ARP | ✅ | 地址解析 |
| DHCP | ✅ | 动态IP配置 |
| DNS | ✅ | 域名解析 |
| VLAN | ✅ | `ETHARP_SUPPORT_VLAN` |

---

## 9. 关键文件清单

```
os-framework/servers/net/src/main.c          NSv 主入口 + event loop
os-framework/servers/net/include/nsv/nsv.h   NSv 常量/宏定义
os-framework/servers/net/src/packet_mmap.c   AF-PACKET TPACKET 实现
os-framework/servers/net/include/nsv/packet_mmap.h
os-framework/servers/net/src/bridge.c       VirtIO Bridge 实现
os-framework/libs/os_libs/libcore/src/ds_ring.c     DS-RING 实现
os-framework/libs/os_libs/libcore/src/ringbuffer.c  ringbuf 实现
os-framework/libs/os_libs/libcore/include/core/elem_ring.h
external/lwip_ds_mcu/src/core/raw.c         raw_afpacket_input/output
external/lwip_ds_mcu/src/netif/ethernet.c   ethernet_input/output
external/lwip_ds_mcu/src/api/sockets.c       BSD socket → lwIP
external/lwip_ds_mcu/src/api/api_msg.c      Netconn 消息处理
external/lwip_ds_mcu/src/core/tcp.c
external/lwip_ds_mcu/src/core/udp.c
external/lwip_ds_mcu/src/core/ipv4/ip4.c
external/lwip_ds_mcu/src/core/ipv4/etharp.c
libs/util_libs/liblwip/src/sys_arch_sel4.c  seL4 sys_arch 适配层
libs/util_libs/liblwip/default_opts/lwipopts.h  lwIP 配置
libs/project_libs/libringbuffer/src/ringbuffer.c
external/net-cap/net_cap.c                  抓包应用
os-framework/libs/os_libs/libcore/include/core/sys_net.h  App侧socket syscalls
```

---

## 10. 与 Linux 网络栈的本质差异

| 方面 | Linux | SafeOS |
|------|-------|--------|
| 网络栈位置 | **内核态** | **用户态** (NSv进程) |
| NIC驱动 | 内核模块 | **独立用户态进程** |
| 内存共享 | kernel/User共享内存 | **DSPACE + CMA** |
| 调度 | 内核网络子系统和调度器 | **lwIP + seL4调度** |
| socket实现 | 内核文件描述符 | **lwIP netconn → 用户态模拟** |
| 零拷贝 | sendfile/AF_XDP | **NIC→CMA→pbuf→App，两次拷贝** |
| 中断处理 | 硬件中断 | **seL4 notification** |
| 包分发 | 内核协议栈 → socket | **lwIP raw_afpacket → callback** |

---

## 11. 设计特点与限制

**优点**:

- 极简内核 — seL4只做IPC和内存管理，网络全在用户态
- 可预测性 — 网络延迟不依赖内核调度
- 隔离性 — NIC驱动崩溃不影响网络栈

**限制**:

- 性能不如原生Linux — 用户态复制、中断处理开销
- 协议支持有限 — 无IPv6、无高级网络特性
- 实时性依赖seL4调度 — 中断→notification→线程唤醒链路长
