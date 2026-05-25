---
type: entity
tags: [safeos, nsv, lwip, network, cma, ds-ring, elem-ring, seL4, rx-tx]
created: 2026-05-25
sources: [safeos-architecture]
---

# SafeOS Network Implementation — 网络实现深度分析

## 定义

SafeOS 网络实现是基于 seL4 微内核的**全用户态网络栈**，通过 CMA (Contiguous Memory Area) + elem_ring 实现 NIC 驱动与 NSv 网络服务器之间的零拷贝 DMA 缓冲共享，所有网络协议 (TCP/UDP/IP/ARP/VLAN) 完全在用户态运行。

## 整体架构

```
┌──────────────────────────────────────────────────────────────────────────────┐
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

## CMA + DS-RING 内存架构

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

### 4 个 Ring 的关系

| Ring | 方向 | 生产者 | 消费者 | 说明 |
|------|------|--------|--------|------|
| `empty_rx_buf_ring` | RX空缓冲 | NSv | NIC | NIC 从中取空buffer用于收包 |
| `used_rx_buf_ring` | RX收包 | NIC | NSv | NIC 收包后放入，NSv 取走处理 |
| `pending_tx_buf_ring` | TX待发 | NSv | NIC | NSv 放入待发数据，NIC 取走发送 |
| `used_tx_buf_ring` | TX完成 | NIC | NSv | NIC 发完后放入，NSv 回收buffer |

### DS-RING 初始化

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

## elem_ring 无锁环形缓冲

### 结构

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

elem_ring 是**无锁单生产者/单消费者**环形缓冲区，使用 ARM `dmb(ish)` 内存屏障保证读写顺序。

---

## 收包路径 (RX)

```
NIC 驱动
   │
   │ DMA 收包到 DMA buffer (物理地址)
   │
   ▼
used_rx_buf_ring (elem_ring_put) ─────────────────┐
   │                                              │
   │ sel4_signal(nic_rx_ntfn)                      │
   │                                              │
▼                                              │
nic_rx_thread()                                    │
   │                                              │
   │ seL4_Recv(nsv_nic_ep)                       │
   │                                              │
▼                                              │
elem_ring_get(used_rx_buf_ring) → 获取 buffer PA   │
   │                                              │
   │ cma_pa_to_va() → 转换为虚拟地址             │
   │                                              │
▼                                              │
rx_callback(pbuf)                                  │
   │                                              │
   │ LOCK_TCPIP_CORE()                           │
   │                                              │
▼                                              │
vnet_if.input(p, &vnet_if) = ethernet_input()     │
   │                                              │
   ├── ETH_P_IP  → ip_input() → raw_input() ─────► raw_afpacket_input()
   │                    │                    │              │
   │                    │                    │         tpacket_recv()
   ├── ETH_P_ARP → etharp_input()            │              │
   └── 其他          → 未知协议              │              │
                                        │              │
                    API_EVENT(conn, RCVPLUS) ──────────┘
                          │
                          │ select()/poll() 唤醒 App
                          ▼
                       App recvfrom()
```

---

## 协议支持

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

## 与 Linux 网络栈的本质差异

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

## 设计特点与限制

**优点**:
- 极简内核 — seL4 只做 IPC 和内存管理，网络全在用户态
- 可预测性 — 网络延迟不依赖内核调度
- 隔离性 — NIC 驱动崩溃不影响网络栈

**限制**:
- 性能不如原生 Linux — 用户态复制、中断处理开销
- 协议支持有限 — 无 IPv6、无高级网络特性
- 实时性依赖 seL4 调度 — 中断→notification→线程唤醒链路长

---

## 关键文件清单

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

## 相关概念

- [[entities/linux/safeos/safeos-nsv]] — NSv Network Server 深度分析
- [[entities/linux/safeos/safeos-packet-mmap]] — AF-PACKET + TPACKET 抓包实现
- [[entities/linux/lwip/lwip-elem-ring]] — 无锁单生产者/单消费者环形缓冲区
- [[entities/linux/lwip/lwip-cma-buffer]] — CMA 缓冲区分配、pbuf 映射
- [[entities/linux/lwip/lwip-sel4-interaction]] — lwIP 与 seL4 物理网卡/VLAN/Hypervisor 交互
- [[entities/linux/lwip/lwip-ethernet-input]] — L2→L3 入口 ethernet_input
- [[entities/linux/lwip/lwip-ethernet-output]] — L3→L2 封装 ethernet_output
- [[lwip-index]] — lwIP 模块完整索引

## 来源详情

- [[sources/safeos-architecture]] — SafeOS Architecture & Design Documents
