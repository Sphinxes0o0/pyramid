---
type: synthesis
tags: [safeos, lwip, network, analysis, architecture]
created: 2026-05-25
sources: [safeos-lwip-core, safeos-lwip-extensions]
---

# SafeOS lwIP 深度逆向分析

> 文档版本: 1.0
> 分析日期: 2026/05/25
> 目标: 对 SafeOS 中 lwIP 协议栈进行函数级逆向分析
> 源文档: `~/workspace/wiki/pyramid/raw/safeos/` 下 19 个分析文档
> Wiki 参考: `~/workspace/wiki/pyramid/wiki/entities/linux/lwip/` 下 27 个 entity 页面

---

## 1. 整体架构

### 1.1 NSv 网络服务虚拟化架构图（文字描述）

SafeOS 运行在 **seL4 微内核**之上，网络栈完全在**用户态**实现（NSv 进程），不依赖内核网络子系统。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Application Layer                                  │
│              (iperf, ping, lwfwcfg, udpecho, net-cap, tcpdump)          │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ seL4 IPC (badge = pid)
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NSv — Network Server (lwIP)                            │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Thread: event_loop()                                                 │  │
│  │  Role: 处理 App 的 BSD socket 请求 (SYS_NET_SOCKET/BIND/SENDTO等)   │  │
│  │  Wait: seL4_Recv(svc_ep, &badge)  — 阻塞等待 App IPC 请求           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Thread: nic_rx_thread()                                              │  │
│  │  Role: 从 NIC 驱动接收数据包                                         │  │
│  │  Wait: seL4_Recv(nsv_nic_ep, &badge) — 等待 NIC RX notification    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Thread: tcpip_thread()                                               │  │
│  │  Role: lwIP 协议栈核心处理 (TCP/UDP/IP/ARP)                          │  │
│  │  Lock: LWIP_TCPIP_CORE_LOCKING — 全局核心锁保护                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Thread: select_thread() (可选)                                      │  │
│  │  Role: 监控 socket 事件 (READ/WRITE/ACCEPT)                        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  LWFW (Lightweight Firewall)                                          │  │
│  │  - Ingress filter hook: ip4_input 中的 LWIP_HOOK_IP4_INPUT          │  │
│  │  - Egress filter hook: ip4_output_if 中的 LWIP_HOOK_IP4_OUTPUT      │  │
│  │  - Connection tracking: lwfw_ct (lwfw_lwct.c)                       │  │
│  │  - Classification: 5-tuple 匹配规则                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  lwIP Protocol Stack (external/lwip_ds_mcu/src/)                    │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐   │  │
│  │  │   TCP    │ │   UDP    │ │  Raw     │ │  IGMP   │ │  DNS   │   │  │
│  │  │  tcp_in  │ │  udp_in  │ │  pcb    │ │ igmp_in │ │        │   │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └────────┘   │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │              IP Layer (ipv4)                                   │ │  │
│  │  │   ip4_input ───► LWFW ingress_filter                        │ │  │
│  │  │   ip4_output_if ◄── LWFW egress_filter                      │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────────────────────┐ │  │
│  │  │              L2 Layer (ethernet/arp/vlan)                     │ │  │
│  │  │   ethernet_input ───► lwip_arp_filter_netif_fn (VLAN 分发)    │ │  │
│  │  │   ethernet_output ◄── lwip_hook_vlan_set (VLAN tag 插入)      │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  NSv Network Virtualization                                          │  │
│  │  - packet_mmap.c: AF-PACKET TPACKET ring                            │  │
│  │  - bridge.c: VIRT_BRG_SUPPORT (hypervisor bridge)                  │  │
│  │  - ipc-if.c: VNET_OVER_IPC_SUPPORT (VM 通信)                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  DMA / elem_ring Layer (libs/os_libs/libcore/)                      │  │
│  │  - CMA (Contiguous Memory Area): 96MB 共享内存区域                 │  │
│  │  - elem_ring x4: 无锁单生产者/单消费者环形缓冲区                   │  │
│  │    ├─ empty_rx_buf_ring    (NSv → NIC, 提供空 buffer)             │  │
│  │    ├─ used_rx_buf_ring     (NIC → NSv, 已收包)                     │  │
│  │    ├─ pending_tx_buf_ring  (NSv → NIC, 待发送)                     │  │
│  │    └─ used_tx_buf_ring     (NIC → NSv, 已完成)                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ seL4 IPC + CMA 共享内存
┌────────────────────────────────▼────────────────────────────────────────────┐
│                    seL4 Microkernel                                        │
│                    (IPC, memory, threads)                                 │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│                    NIC Driver (PFE/VIRTIO)                                 │
│                    (独立用户态进程, DMA, interrupts, RX/TX)                │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 组件拓扑: NSv → lwIP → seL4 IPC → 硬件

| 层次 | 组件 | 关键文件 | 职责 |
|------|------|----------|------|
| **应用层** | App 进程 | `external/net-cap/`, `os-framework/apps/` | socket 调用、抓包 |
| **syscall 桥接** | seL4 IPC | `libs/os_libs/libcore/src/sys_net.c` | App → NSv 的 BSD syscall 转发 |
| **NSv Socket API** | event_loop | `servers/net/src/main.c:3400` | socket 请求分发处理 |
| **LWFW** | 防火墙 | `libs/util_libs/liblwfw/src/lwfw.c` | ingress/egress filter hooks |
| **lwIP Core** | 协议栈 | `external/lwip_ds_mcu/src/core/` | TCP/UDP/IP/ARP/IGMP |
| **lwIP netif** | 网卡抽象 | `external/lwip_ds_mcu/src/netif/ethernet.c` | ethernet_input/output, VLAN |
| **NSv 虚拟化** | packet_mmap | `servers/net/src/packet_mmap.c` | AF-PACKET ring buffer |
| **NSv 虚拟化** | VIRT_BRG | `servers/net/src/bridge.c` | hypervisor 网桥 |
| **NSv 虚拟化** | IPCIF | `servers/net/src/ipc-if.c` | VM 通信 |
| **内存层** | CMA/elem_ring | `libs/os_libs/libcore/include/core/elem_ring.h` | 零拷贝 DMA buffer |
| **seL4** | IPC + DSpace | kernel | 通知机制、共享内存授权 |
| **硬件** | NIC Driver | 独立进程 | DMA 收包、TX 发送 |

### 1.3 数据流向

#### 收包路径 (RX)

```
NIC 驱动进程
   │
   │ DMA 收包到 CMA buffer (物理地址)
   │
   │ elem_ring_put(used_rx_buf_ring, elem{pa=buffer_pa})
   │ sel4_signal(nic_rx_ntfn)
   │
   ▼
nic_rx_thread()  [main.c:4961]
   │
   │ seL4_Recv(nsv_nic_ep, &badge)
   │
   ▼
elem_ring_get(used_rx_buf_ring) → e.pa
   │
   │ cma_pa_to_va(&cma, e.pa) → 虚拟地址
   │
   ▼
LOCK_TCPIP_CORE()
   │
   ▼
rx_callback((struct pbuf *)va)  [main.c:4781]
   │
   ▼
vnet_if.input(p, &vnet_if) = ethernet_input()  [ethernet.c:89]
   │
   ├─► ETH_P_IP → ip4_input(p, netif)
   │     │
   │     ├─► #ifdef NIO_LWIP_LWFW → lwfw_p->ops->ingress_filter(p, inp)
   │     │
   │     └─► tcp_input() / udp_input() / icmp_input() / igmp_input()
   │
   ├─► ETH_P_ARP → etharp_input(p, netif)
   │
   └─► ETH_P_VLAN → VLAN tag 解析 → 分发到 vlan_if[i]
         │
         ▼
   raw_afpacket_input(p, netif, type)  [raw.c:282]
         │
         └─► 遍历 raw_afpacket_pcbs → tpacket_recv() → ring buffer
               │
               ▼
         API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0)
               │
               ▼
         select()/poll() 唤醒 App
               │
               ▼
         App recvfrom()
```

#### 发包路径 (TX)

```
App sendto()
   │
   │ seL4 IPC → event_loop
   │
   ▼
sys_sendto_nb()  [main.c:1274]
   │
   │ lwip_sendto(socket, buf, len, flags, addr, addrlen)
   │
   ▼
netconn_sendto() → tcp_output() / udp_sendto() / raw_sendto()
   │
   ▼
ip4_output_if(p, src_ip, dst_ip, proto, netif)  [ip4.c:888]
   │
   ├─► ip4_route() → 路由查找
   │
   ├─► #ifdef NIO_LWIP_LWFW → lwfw_p->ops->egress_filter(p, netif)
   │
   ├─► 添加 IP Header, 计算 checksum
   │
   └─► netif->output(netif, p, dest) = etharp_output()
         │
         ▼
   etharp_output() → ethernet_output(netif, p, ...)
         │
         ├─► [可选] LWIP_HOOK_VLAN_SET → 插入 VLAN Tag
         │
         ├─► 添加 Ethernet Header
         │
         ├─► raw_afpacket_output() → AF-PACKET 通知
         │
         └─► netif->linkoutput(netif, p) = ethif_link_output()
               │
               ▼
         ethif_link_output()  [main.c:3788]
               │
               ├─► 检查 nic_ready
               ├─► 分配新 pbuf 并 memcpy
               ├─► elem_ring_put(pending_tx_buf_ring, elem{pa})
               │
               └─► sel4_signal(nic_tx_ntfn)
                     │
                     ▼
                   NIC 驱动 DMA 发送
```

### 1.4 与 Standard lwIP 的差异点总结

| 方面 | Standard lwIP | SafeOS lwIP |
|------|---------------|-------------|
| **运行环境** | 嵌入式裸机或 OS | seL4 微内核用户态进程 (NSv) |
| **sys_arch** | 依赖具体 OS (FreeRTOS/Zephyr) | seL4 sys_arch_sel4.c 适配层 |
| **内存分配** | 静态内存池或 heap | CMA + ds_ring_mem_alloc |
| **进程隔离** | 无 | NSv 与 NIC 驱动在不同进程 |
| **VLAN 分发** | 无 (标准 lwIP 软 VLAN) | lwip_arp_filter_netif_fn 硬分发 |
| **VLAN netif** | 通过 netif->input 回调 | 独立 vlan_if[i]，input=tcpip_input |
| **AF-PACKET** | 无 | raw_afpacket_input/output + tpacket_recv |
| **packet_mmap** | 无 | DSPACE + ringbuf 自定义实现 |
| **防火墙** | 无 | LWFW ingress/egress filter hooks |
| **网桥** | bridgeif (802.1D) | VIRT_BRG_SUPPORT hypervisor 桥接 |
| **VM 通信** | 无 | VNET_OVER_IPC_SUPPORT ipcif |
| **多核利用** | 可配置多 tcpip_thread | 单 tcpip_thread (LWIP_TCPIP_CORE_LOCKING) |
| **IPv6** | 可启用 | LWIP_IPV6=0 禁用 |
| **线程模型** | 可选单线程或多线程 | 单 tcpip_thread + LWIP_TCPIP_CORE_LOCKING=1 |

---

## 2. 模块分析

### 2.1 pbuf / 内存管理

#### 核心数据结构

**struct pbuf** (`include/lwip/pbuf.h`):

```c
struct pbuf {
    struct pbuf *next;      // 链表下一 pbuf
    void *payload;          // 数据指针
    u16_t len;              // 本 pbuf 数据长度
    u16_t tot_len;          // 链表中所有 pbuf 总长度
    u16_t type;             // PBUF_POOL/PBUF_RAM/PBUF_ROM/PBUF_REF
    u16_t flags;            // 标志 (PBUF_FLAG_SM_Q5/Q6 for QoS)
    u8_t  if_idx;          // 网卡索引
    // DMA buffer 相关
    u16_t flags_internal;   // DMA buffer 标志
};
```

**CMA 结构** (`core/cma.h`):

```c
struct cma {
    vaddr_t va;             // 虚拟地址起始
    paddr_t pa;             // 物理地址起始
    size_t size;             // 区域大小 (96MB)
    kr_malloc_t mem;        // 内核内存分配器
};
```

#### 内存池设计

SafeOS lwIP 使用三种分配器:

| 分配器 | 用途 | 位置 | 特点 |
|--------|------|------|------|
| **CMA (dma_pbuf)** | NIC RX/TX DMA buffer | main.c:4155 `alloc_dma_buf` | 物理连续，96MB CMA 区域，slab cache 管理 |
| **lwip_malloc (ds_ring)** | 通用内存分配 | sys_arch_sel4.c `lwip_malloc` | 基于 ds_ring_mem_alloc，适合 lwIP 内部 |
| ** memp (内存池)** | 固定大小对象 | lwIP memp 子系统 | TCP PCB/UDP PCB/raw PCB/netconn 等 |

**DMA Buffer 分配路径**:
```c
alloc_dma_buf(size)  [main.c:4155]
  → dma_pbuf_alloc(DMA_BUF_SIZE)  [main.c:4121]
  → mem_cache_alloc()  // slab 分配
  → pbuf_init_alloced_pbuf()
  → cma_va_to_pa()  // 返回物理地址给 NIC DMA
```

#### malloc / CMA / elem_ring 分配器对比

| 维度 | lwip_malloc (ds_ring) | CMA (dma_pbuf) | elem_ring |
|------|------------------------|-----------------|-----------|
| **用途** | lwIP 内部通用内存 | NIC DMA buffer | NIC ↔ NSv buffer 指针传递 |
| **物理连续性** | 不保证 | 保证 (CMA) | N/A (只传指针) |
| **跨进程共享** | 否 | 是 (与 NIC 驱动共享 PA) | 是 (PA 在共享 CMA 中) |
| **分配粒度** | 任意大小 | 固定 DMA_BUF_SIZE (1576B) | 固定 elem 大小 |
| **零拷贝** | 否 | 支持 DMA 零拷贝 | 传递 PA 无拷贝 |

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `alloc_dma_buf` | main.c:4155 | 从 CMA 分配 DMA buffer PA |
| `free_dma_buf` | main.c:4171 | 释放 DMA buffer PA → VA → pbuf_free |
| `dma_pbuf_alloc` | main.c:4121 | slab cache 分配 pbuf |
| `refill_dma_buf_if_needed` | main.c:4738 | 预填充 empty_rx_buf_ring |
| `lwip_malloc` | sys_arch_sel4.c | ds_ring 内存分配 |
| `lwip_free` | sys_arch_sel4.c | ds_ring 内存释放 |
| `pbuf_alloc` | pbuf.c | 分配 pbuf chain |
| `pbuf_free` | pbuf.c | 释放 pbuf (引用计数) |
| `pbuf_copy` | pbuf.c | 拷贝 pbuf chain |
| `cma_va_to_pa` | cma.h:43 | 虚拟地址 → 物理地址 |
| `cma_pa_to_va` | cma.h:50 | 物理地址 → 虚拟地址 |

#### 与 Standard lwIP 差异

- Standard lwIP 的 pbuf 分配依赖 `MEMP` 内存池或 `mem_malloc`，SafeOS 使用 `ds_ring_mem_alloc` 替代
- DMA pbuf 使用专用 slab cache (`dma_pbuf_alloc_raw`)，与通用内存池分离
- pbuf 必须与 CMA 物理地址绑定 (VA ↔ PA 双向转换)

---

### 2.2 netif 管理

#### 核心数据结构

**struct netif** (`include/lwip/netif.h`):

```c
struct netif {
    struct netif *next;         // 链表指针
    char name[2];               // 网卡名，如 {'e','t'} 或 {'v','l'}
    u8_t num;                   // 网卡编号

    ip4_addr_t ip_addr;        // IP 地址
    ip4_addr_t netmask;        // 子网掩码
    ip4_addr_t gw;             // 默认网关

    netif_input_fn input;       // 输入函数指针
    netif_output_fn output;     // IP 层输出 (如 etharp_output)
    netif_linkoutput_fn linkoutput;  // L2 层输出 (如 ethif_link_output)

    u8_t hwaddr[NETIF_MAX_HWADDR_LEN];  // MAC 地址
    u8_t hwaddr_len;           // MAC 地址长度

    u16_t mtu;                 // 最大传输单元
    u16_t vlanid;              // ★ SafeOS 扩展: VLAN ID (0=物理网口)
    u32_t flags;               // 网卡标志

    void *state;               // 驱动特定状态
    void *client_data[LWIP_NETIF_CLIENT_DATA_INDEX_MAX];  // 客户端数据
};
```

#### netif_add 流程

```c
netif_add(&vnet_if, &addr, &netmask, &gw,
          NULL,                // state
          init_ethif,         // 初始化函数
          ethernet_input)      // 输入函数
```

**init_ethif** (`main.c:4710`):
```c
init_ethif(netif)
  → eth_init(netif->hwaddr)           // 读取 MAC 地址
  → netif->output = etharp_output     // IP → Ethernet
  → #ifdef VIRT_BRG_SUPPORT
      netif->linkoutput = ethif_link_output_overload  // 网桥模式
    #else
      netif->linkoutput = ethif_link_output           // 普通模式
  → netif->flags = NETIF_FLAG_ETHARP | NETIF_FLAG_BROADCAST |
                   NETIF_FLAG_LINK_UP | NETIF_FLAG_IGMP | NETIF_FLAG_ETHERNET
```

#### VLAN netif vs 物理 netif

| 属性 | 物理 netif (vnet_if) | VLAN netif (vlan_if[i]) |
|------|----------------------|--------------------------|
| **name** | {'e','t'} 或 {'v','i'} | {'v','l'} |
| **vlanid** | 0 (NO_VLANID) | 配置的 VID (如 100, 200) |
| **input** | `ethernet_input()` | `tcpip_input()` |
| **linkoutput** | `ethif_link_output()` | `low_level_output()` → 调用物理网卡的 linkoutput |
| **IP 地址** | 172.20.0.1 | 172.20.100.1 / 172.20.200.1 |

**vlanif_init** (`vlanif.c:93`):
```c
vlanif_init(netif)
  → netif->output = etharp_output
  → netif->linkoutput = low_level_output    // ★ 关键: 调用物理网卡的 linkoutput
  → netif->vlanid = strtol(vlan_conf[conf_idx].vid, NULL, 10)  // VID
  → netif->input = tcpip_input              // ★ 关键: 不走 ethernet_input
```

#### bridgeif 支持

lwIP 802.1D bridge (`netif/bridgeif.c`, `netif/bridgeif_fdb.c`):
- Bridge netif + Port netifs 多层结构
- FDB (Forwarding Database) 动态学习，5 分钟超时老化
- 端口添加时替换 `portif->input = bridgeif_input`

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `netif_add` | netif.c | 注册 netif 到全局链表 netif_list |
| `netif_set_up` | netif.c | 启用网卡 |
| `netif_set_link_up` | netif.c | 标志链路 UP |
| `netif_set_addr` | netif.c | 设置 IP/网关/掩码 |
| `netif_get_by_index` | netif.c | 通过编号查找 netif |
| `init_ethif` | main.c:4710 | 物理网卡初始化 |
| `vlanif_init` | vlanif.c:93 | VLAN 网卡初始化 |
| `vlanif_setup` | vlanif.c | 创建 VLAN netif |
| `bridgeif_input` | bridgeif.c | Bridge RX 处理 + FDB 学习 |
| `bridgeif_fdb_update_src` | bridgeif_fdb.c | FDB 源 MAC 学习 |

---

### 2.3 IPv4 输入/输出

#### ip4_input 核心流程

**文件**: `external/lwip_ds_mcu/src/core/ipv4/ip4.c:743`

```c
ip4_input(struct pbuf *p, struct netif *inp)
  → pbuf_remove_header(p, IP_HLEN)       // 去掉 IP 头
  → #ifdef NIO_LWIP_LWFW
      if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
        pbuf_free(p); return ERR_OK;    // ★ 防火墙 Ingress Hook
      }
    #endif
  → #if LWIP_IPV4_FORWARD
      ip4_forward(p, iphdr, inp)         // 路由转发
    #endif
  → switch (IPH_PROTO(iphdr)) {
        IP_PROTO_TCP:   tcp_input(p, inp)
        IP_PROTO_UDP:   udp_input(p, inp)
        IP_PROTO_ICMP: icmp_input(p, inp)
        IP_PROTO_IGMP: igmp_input(p, inp)
    }
```

#### ip4_output_if 核心流程

**文件**: `ip4.c:888`

```c
ip4_output_if(pbuf *p, src_ip, dst_ip, proto, netif)
  → ip4_output_if_opt_src()
      → ip4_route()                     // 路由查找，确定 netif
  → #ifdef NIO_LWIP_LWFW
      if (lwfw_p->ops->egress_filter(p, netif) != ERR_OK) {
        pbuf_free(p); return ERR_FW;   // ★ 防火墙 Egress Hook
      }
    #endif
  → IP Header 填充: src_ip, dst_ip, proto, len, id, ttl, tos
  → ip4_chksum()                        // 计算 IP checksum
  → netif->output(netif, p, dest)      // → etharp_output()
```

#### 分片重组算法

**分片输出** (`ip4_frag.c`):
- 当 `p->tot_len > netif->mtu` 时触发
- 按 MTU 大小切分，保留原始 IP 头但修改: `More Fragments=1`, `Fragment Offset` 递增
- 最后一个分片: `More Fragments=0`

**重组输入** (`ip4_frag.c`):
- 维护分片队列 `frag_table` (哈希表)
- 按 `(src_ip, dst_ip, identification, protocol)` 索引
- 超时 `IP_REASS_MAXAGE` (60 秒) 丢弃不完整分片
- 重组后替换 pbuf chain

#### 路由查找算法

**ip4_route** (`ip4.c`):
```c
struct netif *ip4_route(const ip4_addr_t *dest)
  → #if LWIP_IPV4_FORWARD
      ip4_route_src(NULL, dest)        // 支持源地址路由
    #else
      ip4_route_destonly(dest)
    #endif
```

路由表: `struct routing_table` (静态数组)
- `routing_table_init()` 初始化
- 默认网关: `gw.addr != 0` 时作为最后 fallback
- 匹配: `dest & netmask == netif->ip_addr & netmask` (直接交付)

#### ARP 集成

**etharp_output** (`etharp.c:825`):
```c
etharp_output(netif, p, ipaddr)
  → if ip4_addr_isbroadcast(ipaddr, netif) → ETHBROSCAST addr
  → else if ip4_addr_ismulticast(ipaddr) → 组播 MAC 映射
  → else arp_find_entry(ipaddr) → 查 ARP 表
      → 找到: etharp_output_to_arp_index()
      → 未找到: etharp_query() → 发送 ARP 请求
```

**ARP 表** (`struct etharp_entry` 数组, `ETHARP_TABLE_SIZE`):
- 静态条目: 手动添加不过期
- 动态条目: 学习得来，5 分钟老化
- ARP 过滤: `LWIP_ARP_FILTER_NETIF`

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `ip4_input` | ip4.c:743 | IP 输入处理、分发到 L4 |
| `ip4_output` | ip4.c | 通用 IP 输出 |
| `ip4_output_if` | ip4.c:888 | 指定 netif 的 IP 输出 |
| `ip4_route` | ip4.c | 路由查找 |
| `ip4_frag` | ip4_frag.c | IP 分片重组 |
| `ip4_forward` | ip4.c | IP 转发 (需 LWIP_IPV4_FORWARD) |
| `ip4_chksum` | ip4.c | IP checksum 计算 |
| `etharp_output` | etharp.c:825 | IP → Ethernet 封装，查 ARP |
| `etharp_input` | etharp.c | ARP 输入处理 |
| `etharp_query` | etharp.c | 发送 ARP 请求 |
| `etharp_find_entry` | etharp.c | 查找 ARP 表 |
| `lwip_arp_filter_netif_fn` | ethernet.c:459 | ★ SafeOS VLAN-aware netif 选择 |

---

### 2.4 TCP

#### 核心数据结构

**struct tcp_pcb** (`include/lwip/tcp.h`):

```c
struct tcp_pcb {
    IP_PCB;                      // local_ip, remote_ip, local_port, remote_port

    struct tcp_pcb *next;        // 链表指针

    enum tcp_state state;        // TCP 状态机
    u8_t prio;                   // PCB 优先级

    // 接收相关
    struct pbuf *recvmem;        // 接收缓冲区
    u16_t rcv_ann_wnd;          // 通告窗口
    u32_t rcv_nxt;              // 期望的下一个 seq
    struct tcp_seg *unsent;      // 未发送段
    struct tcp_seg *unacked;     // 已发送未确认段
    struct tcp_seg *oq;          // out-of-order 段队列

    // 发送相关
    u32_t snd_nxt;              // 下一个要发送的 seq
    u32_t snd_max;              // 发送窗口上界
    u32_t snd_wnd;              // 对方窗口
    u32_t snd_wl1, snd_wl2;    // 窗口更新 ack
    u32_t cwnd;                  // 拥塞窗口
    u32_t ssthresh;             // 慢启动阈值

    // RTT 相关
    u32_t rttest;               // RTT 采样
    u32_t rtseq;                // 采样时的 seq
    int16_t sa, sv;             // RTT 平滑估计

    // 重传队列
    struct tcp_seg *retransmit;  // 重传段
    u16_t saacksent;            // SACK 发送数 (扩展)

    // Timer
    u16_t polltmr, pollinterval; // 轮询定时器
    u8_t flags;

    // Callbacks
    tcp_accept_fn accept;
    tcp_recv_fn recv;
    tcp_sent_fn sent;
    tcp_poll_fn poll;
    tcp_err_fn err;
    void *callback_arg;
};
```

#### TCP 状态机

```
CLOSED → SYN_SENT → ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
                ↑                                                                              ↓
                └─────────────────── LISTEN ◄─────────────────── ◄──┘
                                          ↓
                                    SYN_RCVD → ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
```

#### PCB 管理

| 链表 | 用途 | 操作宏 |
|------|------|--------|
| `tcp_active_pcbs` | 活跃连接 | `TCP_REG` / `TCP_RMV` |
| `tcp_tw_pcbs` | TIME_WAIT | `TCP_REG` / `TCP_RMV` |
| `tcp_listen_pcbs` | 监听 sockets | `TCP_REG` / `TCP_RMV` |

#### 接收队列设计

```
tcp_input(p, inp)
  → tcp_process()              // 状态机处理
  → tcp_receive()              // 数据接收
      → if (pcb->rcv_ann_wnd > 0) → 调用 pcb->recv()
      → tcp_ack()              // 发送 ACK
  → tcp_enqueue(p)             // 将 pbuf 加入接收队列
      → pbuf_copy_partial()   // 复制数据到 recvmem
      → 更新 rcv_nxt
```

**Backlog**: `listen()` 时指定 backlog 队列长度，`accept()` 从队列取连接。

**Zero Window**: 当接收方窗口为 0，发送方进入 probe 模式，定期发送 zero-window probe。

#### 重传机制

| 机制 | 触发条件 | 行为 |
|------|----------|------|
| **RTO 重传** | 超时 (`tcp_slowtmr`) | 重传 oldest unacked segment |
| **Fast Retransmit** | 3 次重复 ACK | 不等 RTO 直接重传 |
| **SACK** | 扩展启用 | 记录非连续块避免无效重传 |

**RTT 估计** (Jacobson 算法):
```c
srtt += g * (rtt - srtt);      // g = 1/8
rttvar += h * (|rtt - srtt| - rttvar);  // h = 1/4
rto = srtt + 4 * rttvar;
```

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `tcp_input` | tcp.c | TCP 输入处理、状态机 |
| `tcp_process` | tcp.c | 状态转换、SYN/FIN/ACK 处理 |
| `tcp_receive` | tcp.c | 接收数据、触发 recv callback |
| `tcp_enqueue` | tcp.c | 数据入接收队列 |
| `tcp_output` | tcp.c | TCP 输出、发送数据 |
| `tcp_write` | tcp.c | 写数据到发送队列 |
| `tcp_slowtmr` | tcp.c | 慢速定时器 (重传/RTO) |
| `tcp_fasttmr` | tcp.c | 快速定时器 (延迟 ACK) |
| `tcp_accept` | tcp.c | accept 回调 |
| `tcp_connect` | tcp.c | 发起连接 (SYN) |
| `tcp_close` | tcp.c | 关闭连接 (FIN) |
| `tcp_abandon` | tcp.c | 放弃连接 |

---

### 2.5 UDP

#### 核心数据结构

**struct udp_pcb** (`include/lwip/udp.h`):

```c
struct udp_pcb {
    IP_PCB;                      // local_ip, remote_ip, local_port, remote_port

    struct udp_pcb *next;       // 链表指针
    u8_t flags;                 // UDP_FLAGS_* (MULTICAST_LOOP, LWIP_HOOK)

    // 接收回调
    udp_recv_fn recv;
    void *recv_arg;

#if LWIP_IGMP
    ip4_addr_t multicast_ip;     // 组播地址
    u8_t multicast_ttl;         // 组播 TTL
#endif
};
```

#### PCB 绑定

```c
udp_bind(pcb, local_ip, local_port)
  → 检查 port 是否被占用
  → pcb->local_ip = local_ip
  → pcb->local_port = port
  → 注册到 udp_pcbs 链表
```

**Broadcast 处理**: `UDP_FLAGS_BROADCAST` 标志时，绑定 `IP_ADDR_ANY` 的 socket 可收广播。

**Multicast 处理**: `igmp_joingroup()` 加入多播组，`ip_in_multicast_group()` 判断目的地址是否为多播。

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `udp_input` | udp.c | UDP 输入处理、socket 匹配 |
| `udp_output` | udp.c | UDP 输出封装 |
| `udp_sendto` | udp.c | 指定地址发送 |
| `udp_bind` | udp.c | 绑定本地地址端口 |
| `udp_connect` | udp.c | 设置远端地址端口 |
| `udp_recv` | udp.c | 设置接收回调 |

#### Checksum

UDP checksum 是可选的 (checksum 为 0 表示未使用)。计算范围: pseudo header + UDP header + data。

---

### 2.6 VLAN (802.1Q)

#### VLAN Tag 结构

**struct eth_vlan_hdr**:
```
  16-bit TPID = 0x8100 (Tag Protocol Identifier)
  16-bit TCI = PCP(3bit) + DEI(1bit) + VID(12bit)
```

#### VLAN 解析 (RX)

**ethernet_input** (`ethernet.c:133-169`):
```c
if (type == ETHTYPE_VLAN (0x8100)) {
    vlan = (struct eth_vlan_hdr *)(p->payload + 14);
    next_hdr_offset = 18;  // ETH_HDR(14) + VLAN_HDR(4)

    // MAC_VLAN_FILTER hook
    #ifdef MAC_VLAN_FILTER
      if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
        pbuf_free(p); return ERR_OK;  // VID 不匹配，丢弃
      }
    #endif

    type = vlan->tpid;     // 提取真正 EtherType
    p->priority = (vlan->prio_vid >> 13);  // PCP 优先级
}
```

**lwip_arp_filter_netif_fn** (`ethernet.c:459-517`) — VLAN VID → netif 分发:
```c
case ETHTYPE_VLAN:
    NETIF_FOREACH(netif) {
        u16_t vid = PP_HTONS(netif->vlanid) & VLAN_ID_MASK;
        if (vid == (vlan_hdr->prio_vid & VLAN_ID_MASK)) {
            return netif;  // VID 匹配!
        }
    }
    return NULL;  // VLAN packet 但无匹配 → 丢弃
```

#### VLAN Tag 插入 (TX)

**ethernet_output** (`ethernet.c:339-355`):
```c
#if ETHARP_SUPPORT_VLAN && defined(LWIP_HOOK_VLAN_SET)
    vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type);
    if (vlan_prio_vid >= 0) {
        pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR);
        vlanhdr = (struct eth_vlan_hdr *)(p->payload + 14);
        vlanhdr->tpid = eth_type_be;         // 保存原始 EtherType
        vlanhdr->prio_vid = lwip_htons(vlan_prio_vid);
        eth_type_be = ETHTYPE_VLAN;           // 改为 0x8100
    }
#endif
```

**lwip_hook_vlan_set_fn** (main.c):
```c
lwip_hook_vlan_set_fn(netif, pbuf, ...)
  → if (netif->vlanid == NO_VLANID) return -1;  // 不带 VLAN
  → return (pbuf->priority << 13) | netif->vlanid;
```

#### VLAN netif 的特殊路径

VLAN netif (`vlan_if[i]`) 的 `linkoutput = low_level_output`:
```c
low_level_output(netif, p)
  → physical_netif->linkoutput(physical_netif, p)
  // 直接调用物理网卡的 ethif_link_output()
```

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `ethernet_input` | ethernet.c:89 | Ethernet 输入、VLAN 解析 |
| `ethernet_output` | ethernet.c:333 | Ethernet 输出、VLAN 插入 |
| `lwip_arp_filter_netif_fn` | ethernet.c:459 | VLAN VID → netif 软分发 |
| `lwip_hook_vlan_check_fn` | ethernet.c:230 | MAC_VLAN_FILTER: VID 检查 |
| `lwip_hook_vlan_set_fn` | main.c | TX VLAN tag 插入 |
| `vlanif_init` | vlanif.c:93 | VLAN netif 初始化 |
| `vlanif_setup` | vlanif.c | VLAN netif 创建 |

---

### 2.7 seL4 集成

#### IPC 端点

| 端点 | 用途 | 类型 |
|------|------|------|
| `svc_ep` | App ↔ NSv socket syscall | endpoint |
| `nsv_nic_ep` | NIC → NSv RX 通知 | endpoint |
| `nic_rx_ntfn` | NSv → NIC TX 通知 | notification |
| `nic_tx_ntfn` | NIC → NSv TX 完成通知 | notification |
| `net_pm_ep` | 电源管理 | endpoint |

#### seL4 适配层

**sys_arch_sel4.c** 实现:
- `sys_mutex_new/lock/unlock` → seL4 mtx 对象
- `sys_sem_new/wait/signal` → seL4 notification
- `sys_mbox_new/post/fetch` → mbox 实现 (tcpip 线程通信)
- `lwip_malloc/free` → ds_ring_mem_alloc/free

#### 通知机制

**TX 通知** (`ethif_link_output`):
```c
elem_ring_put(pending_tx_buf_ring, elem);
sel4_signal(nic_tx_ntfn);  // 异步，不阻塞
```

**RX 接收** (`nic_rx_thread`):
```c
seL4_Recv(nsv_nic_ep, &badge);  // 阻塞等待
while (1) {
    elem = elem_ring_get(used_rx_buf_ring);
    if (elem.pa == 0) break;
    rx_callback(...);
}
```

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `sel4_signal` | seL4 API | 发送 notification (异步) |
| `seL4_Recv` | seL4 API | 阻塞接收 |
| `seL4_Call` | seL4 API | 同步调用 |
| `sys_bind_ntfn` | main.c | 绑定 notification 到线程 |
| `sys_mutex_new` | sys_arch_sel4.c | 创建互斥锁 |
| `sys_sem_new` | sys_arch_sel4.c | 创建信号量 |
| `sys_mbox_new` | sys_arch_sel4.c | 创建邮箱 |
| `lwip_malloc` | sys_arch_sel4.c | 内存分配 |

---

### 2.8 sys_net API

#### Socket → Netconn → PCB 映射

```
App socket fd (整数)
    ↓ get_socket(fd)
lwip_sock (sockets_priv.h)
    ├─ struct netconn *conn
    └─ packet_info (PACKET_MMAP)
          ↓
    netconn (netbuf.h)
          ├─ enum netconn_type (NETCONN_TCP/UDP/RAW/PACKET)
          └─ union { tcp_pcb, udp_pcb, raw_pcb }
```

#### 三层 API 设计

| 层次 | API | 文件 | 功能 |
|------|-----|------|------|
| **App syscall** | `sys_socket()`, `sys_sendto()` 等 | `libs/os_libs/libcore/src/sys_net.c` | seL4 IPC 封装 |
| **NSv handler** | `sys_socket_nb()`, `sys_sendto_nb()` 等 | `main.c:979-1455` | 解析 seL4 mr, 调用 lwIP |
| **lwIP 函数** | `lwip_socket()`, `lwip_sendto()` | `external/lwip_ds_mcu/src/api/sockets.c` | BSD → netconn 转换 |

#### send/recv 数据传输模式

| 模式 | 条件 | 机制 |
|------|------|------|
| **共享内存** | `use_shm=1` | App 与 NSv 共享 CMA，大块数据零拷贝 |
| **IPC 直接传输** | `use_shm=0` | 通过 seL4 message registers，小于 512B |

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `sys_socket_nb` | main.c:979 | socket 创建 |
| `sys_bind_nb` | main.c:1117 | bind 地址端口 |
| `sys_listen_nb` | main.c:1140 | listen 监听 |
| `sys_accept` | main.c:1155 | accept 接受连接 |
| `sys_connect_nb` | main.c:1230 | connect 连接 |
| `sys_sendto_nb` | main.c:1274 | 发送数据 |
| `sys_recvfrom_nb` | main.c:1368 | 接收数据 |
| `sys_close_nb` | main.c:2530 | 关闭 socket |
| `select_thread` | main.c:5560 | select/poll 事件监控 |
| `sys_net_ctl` | main.c:2140 | 控制命令 (ifconfig/netstat/lwfwcfg) |

---

### 2.9 packet_mmap

#### AF-PACKET 实现

SafeOS 使用自定义 DSPACE 方案，非 Linux 标准 PACKET_MMAP:

| 组件 | 说明 |
|------|------|
| **DSPACE** | seL4 跨进程共享内存抽象 (替代 mmap) |
| **ringbuf** | 单生产者/单消费者无锁环形缓冲区 |
| **TPACKET_V1** | 仅作为数据格式约定 |
| **raw_afpacket** | lwIP raw socket 钩子 |

#### DSPACE 布局 (4MB)

```
Offset 0x0:
├─ struct packet_mmap_info (32 bytes)  ← 元数据
├─ struct ringbuf (管理读写索引)         ← ring buffer 头部
└─ TPACKET 帧循环队列                   ← 数据区 (2048 × 1024 bytes)
```

#### TPACKET 帧格式

```
┌────────────┬─────────────────────────────────┬────────────────────────────┐
│ tpacket_hdr│      Ethernet Frame             │         Padding            │
│  (24 B)    │     (tp_len bytes)             │ (2048-24-tp_len bytes)    │
└────────────┴─────────────────────────────────┴────────────────────────────┘
```

#### 收包回调链

```
ethernet_input()
  → raw_afpacket_input(p, netif, type)
      → 遍历 raw_afpacket_pcbs
          → pcb->recv(pcb->recv_arg, pcb, p, NULL)
              → tpacket_recv()
                  → rb_write_tpacket()
                  → API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0)
```

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `packet_mmap_set_ring` | packet_mmap.c:265 | Ring 初始化，DSPACE 映射 |
| `tpacket_recv` | packet_mmap.c:122 | 收包回调，写入 ring buffer |
| `rb_write_tpacket` | packet_mmap.c:104 | 写入 TPACKET 帧 |
| `packet_mmap_event_callback` | packet_mmap.c:172 | 事件回调，select/poll 唤醒 |
| `packet_mmap_free_resources` | packet_mmap.c:409 | 释放资源 |

---

### 2.10 raw socket

#### raw_pcb 结构

```c
struct raw_pcb {
    IP_PCB;
    struct raw_pcb *next;
    u8_t domain;              // AF_INET / AF_INET6
    u16_t protocol;           // ETH_P_* 或 IP_PROTO_*
    u8_t flags;
    raw_recv_fn recv;         // 接收回调
    void *recv_arg;
    // ★ SafeOS 扩展
    struct sockaddr_ll sockaddr;  // AF-PACKET 绑定信息
    void *conn;                  // packet_mmap 连接
    u8_t state;
};
```

#### 两类 RAW socket

| 链表 | 用途 | 协议层次 |
|------|------|----------|
| `raw_pcbs` | 普通 RAW | IP 层 (IP_PROTO_*) |
| `raw_afpacket_pcbs` | AF-PACKET | Ethernet 层 (ETH_P_*) |

#### AF-PACKET 绑定

```c
raw_afpacket_bind(pcb, name, namelen)
  → pcb->sockaddr.sll_ifindex = addr->sll_ifindex;
  → pcb->sockaddr.sll_protocol = addr->sll_protocol;
  → pcb->protocol = addr->sll_protocol;
  → pcb->recv = raw_afpacket_recv_callback;
  → 注册到 raw_afpacket_pcbs 链表
```

#### 核心函数列表

| 函数 | 文件:行号 | 功能 |
|------|----------|------|
| `raw_afpacket_input` | raw.c:282 | AF-PACKET 接收，遍历 raw_afpacket_pcbs |
| `raw_afpacket_output` | raw.c:391 | AF-PACKET 发送通知 |
| `raw_afpacket_bind` | raw.c:694 | AF-PACKET 绑定 |
| `raw_input` | raw.c:475 | 普通 RAW IP input |
| `raw_sendto` | raw.c | RAW 发送 |

---

## 3. 数据流分析

### 3.1 完整收包路径

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ NIC 驱动进程                                                                 │
│   DMA 收包到物理地址 (CMA 区域)                                              │
│   elem_ring_put(used_rx_buf_ring, elem{pa})                                 │
│   sel4_signal(nic_rx_ntfn)                                                 │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ seL4 IPC (notification)
┌────────────────────────────────▼────────────────────────────────────────────┐
│ nic_rx_thread()  [main.c:4961]                                             │
│   seL4_Recv(nsv_nic_ep, &badge)                                            │
│   while (1) {                                                               │
│     elem = elem_ring_get(used_rx_buf_ring)                                  │
│     if (elem.pa == 0) break;                                               │
│     va = cma_pa_to_va(&cma, elem.pa)                                       │
│     LOCK_TCPIP_CORE()                                                       │
│     rx_callback((struct pbuf *)va)  [main.c:4781]                          │
│     UNLOCK_TCPIP_CORE()                                                     │
│   }                                                                         │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ pbuf (VA)
┌────────────────────────────────▼────────────────────────────────────────────┐
│ vnet_if.input(p, &vnet_if) = ethernet_input()  [ethernet.c:89]            │
│                                                                              │
│   Step 1: 解析 Ethernet Header                                              │
│     ethhdr = p->payload                                                     │
│     type = ethhdr->type                                                     │
│                                                                              │
│   Step 2: VLAN 解析 (type == ETHTYPE_VLAN)                                  │
│     vlan = p->payload + 14                                                 │
│     next_hdr_offset = 18                                                    │
│     #ifdef MAC_VLAN_FILTER                                                  │
│       if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) { 丢弃 }             │
│     #endif                                                                  │
│     type = vlan->tpid                                                       │
│                                                                              │
│   Step 3: VLAN 分发 — lwip_arp_filter_netif_fn  [ethernet.c:459]           │
│     case ETHTYPE_VLAN:                                                      │
│       NETIF_FOREACH → netif->vlanid == vid → 返回该 netif                 │
│     case ETHTYPE_IP / ETHTYPE_ARP:                                          │
│       从 header 提取 dest IP → NETIF_FOREACH → ip_addr 匹配 + vlanid==0   │
│                                                                              │
│   Step 4: AF-PACKET 捕获  [raw.c:282]                                      │
│     raw_afpacket_input(p, netif, type)                                     │
│       → 遍历 raw_afpacket_pcbs                                              │
│       → pcb->recv(pcb->recv_arg, pcb, p, NULL)                            │
│         → tpacket_recv() → ring buffer 写入 → API_EVENT(RCVPLUS)           │
│                                                                              │
│   Step 5: 协议分发                                                          │
│     switch (type) {                                                         │
│       ETHTYPE_IP:   ip4_input(p, netif)                                    │
│       ETHTYPE_ARP:  etharp_input(p, netif)                                  │
│     }                                                                       │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│ ip4_input(p, inp)  [ip4.c:743]                                            │
│                                                                              │
│   pbuf_remove_header(p, IP_HLEN)  // 去掉 IP 头                            │
│                                                                              │
│   #ifdef NIO_LWIP_LWFW                                                     │
│     if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) { 丢弃 }            │
│   #endif                                                                    │
│                                                                              │
│   switch (IPH_PROTO(iphdr)) {                                              │
│     IP_PROTO_TCP:   tcp_input(p, inp)                                      │
│     IP_PROTO_UDP:   udp_input(p, inp)                                      │
│     IP_PROTO_ICMP:  icmp_input(p, inp)                                    │
│     IP_PROTO_IGMP:  igmp_input(p, inp)                                    │
│   }                                                                       │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│ tcp_input / udp_input 等 L4 处理                                            │
│                                                                              │
│   → tcp_enqueue(p) → 放入接收队列                                          │
│   → tcp_receive() → 触发状态机                                            │
│   → API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0)                               │
│     → sock->rcvevent++                                                    │
│     → sys_sem_signal(&sock->select_waiting_sem)  // 唤醒 select/poll       │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│ App 侧                                                                        │
│   select()/poll() 返回 POLLIN                                              │
│   recvfrom() → NSv event_loop → sys_recvfrom_nb → lwip_recvfrom           │
│   → 数据通过 seL4 IPC reply 返回 App                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 完整发包路径

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ App 侧                                                                        │
│   sendto(socket, buf, len, flags, addr, addrlen)                          │
│   → seL4 IPC syscall → event_loop                                          │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│ sys_sendto_nb()  [main.c:1274]                                             │
│   → lwip_sendto(socket, send_buf, data_len, flags, addr, addrlen)        │
│     → lwIP socket API                                                      │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│ UDP: udp_sendto() / TCP: tcp_output() / RAW: raw_sendto()                 │
│                                                                              │
│   → ip4_output_if(p, src_ip, dst_ip, proto, netif)  [ip4.c:888]           │
│                                                                              │
│   Step 1: 路由查找                                                          │
│     ip4_route(dst_ip) → 确定输出 netif                                     │
│                                                                              │
│   Step 2: 防火墙 Egress Hook                                                │
│     #ifdef NIO_LWIP_LWFW                                                   │
│       if (lwfw_p->ops->egress_filter(p, netif) != ERR_OK) { 丢弃 }        │
│     #endif                                                                  │
│                                                                              │
│   Step 3: IP Header 填充                                                    │
│     src_ip, dst_ip, proto, len, id, ttl, tos                              │
│     ip4_chksum() → 计算 IP checksum                                        │
│                                                                              │
│   Step 4: netif->output = etharp_output()  [etharp.c:825]                │
│                                                                              │
│   Step 5: ARP 查找                                                          │
│     if broadcast → ETHBROSCAST                                             │
│     else if multicast → 组播 MAC 映射                                       │
│     else → arp_find_entry() → 查 ARP 表                                     │
│       未找到 → etharp_query() 发送 ARP 请求                                 │
│       找到 → etharp_output_to_arp_index()                                  │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│ ethernet_output()  [ethernet.c:333]                                        │
│                                                                              │
│   Step 1: VLAN Tag 插入 (可选)                                              │
│     #ifdef LWIP_HOOK_VLAN_SET                                               │
│       vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, ...)                   │
│       if (vlan_prio_vid >= 0) {                                            │
│         pbuf_add_header(p, 18);  // ETH(14) + VLAN(4)                    │
│         填充 VLAN header, eth_type = ETHTYPE_VLAN                          │
│       }                                                                     │
│     #endif                                                                  │
│                                                                              │
│   Step 2: Ethernet Header                                                  │
│     SMEMCPY(ethhdr->dest, dst, ETH_HWADDR_LEN)                            │
│     SMEMCPY(ethhdr->src, src, ETH_HWADDR_LEN)                             │
│     ethhdr->type = eth_type_be                                              │
│                                                                              │
│   Step 3: AF-PACKET 通知                                                   │
│     raw_afpacket_output(p, netif)  [raw.c:391]                           │
│       → 遍历 raw_afpacket_pcbs → 触发 recv 回调                           │
│                                                                              │
│   Step 4: 实际发送                                                          │
│     netif->linkoutput(netif, p) = ethif_link_output()  [main.c:3788]     │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│ ethif_link_output()  [main.c:3788]                                         │
│                                                                              │
│   if (!nic_ready) return ERR_OK;  // NIC 未就绪丢弃                        │
│                                                                              │
│   #ifdef USE_SEND_SMOOTH_QAV                                                │
│     if (q->flags & PBUF_FLAG_SM_Q5/Q6) {                                  │
│       sm_post_element(sm_que5/6_buf_ring, q); return ERR_OK;             │
│     }                                                                       │
│   #endif                                                                    │
│                                                                              │
│   // 分配新 pbuf 并 memcpy (TX single pbuf)                                │
│   p = lwip_malloc(length + SIZEOF_STRUCT_PBUF + ...);                     │
│   pbuf_init_alloced_pbuf(p, payload_va, length, length, PBUF_RAM, 0);     │
│   memcpy(p->payload, q->payload, length);                                 │
│                                                                              │
│   free_complete_tx_packet_pbuf();  // 回收已完成的 TX buffer               │
│                                                                              │
│   elem = {pa: cma_va_to_pa(&cma, p)}                                      │
│   elem_ring_put(pending_tx_buf_ring, elem);                                │
│                                                                              │
│   sel4_signal(nic_tx_ntfn);  // 通知 NIC 有数据包                          │
│   return ERR_OK;                                                            │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │ elem {pa}
┌────────────────────────────────▼────────────────────────────────────────────┐
│ NIC 驱动进程                                                                 │
│   seL4_Wait(nic_tx_ep, ...)                                                │
│   elem = elem_ring_get(pending_tx_buf_ring);                               │
│   DMA 发送 (使用 buffer PA)                                                 │
│   elem_ring_put(used_tx_buf_ring, elem);                                   │
│   sel4_signal(nic_rx_ntfn);  // 通知 NSv TX 完成                           │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│ NSv: 下次 rx_callback 时                                                   │
│   free_complete_tx_packet_pbuf()  [main.c:3773]                            │
│     elem = elem_ring_get(used_tx_buf_ring);                                │
│     p = cma_pa_to_va(&cma, elem.pa);                                     │
│     lwip_free(p);  // 释放 pbuf                                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 VLAN 分发路径

```
含 VLAN Tag 的包进入 ethernet_input()
   │
   ▼
type == ETHTYPE_VLAN (0x8100)?
   │
   ├─ YES:
   │    解析 VLAN Tag (TPID=0x8100, TCI=VID+PCP+DEI)
   │    p->priority = PCP (高 3 位)
   │    从 VLAN header 提取真正 EtherType (tpid 字段)
   │    调用 LWIP_HOOK_VLAN_CHECK() → VID 匹配检查
   │
   └─ NO:
        type = ethhdr->type (IP 或 ARP)
   │
   ▼
lwip_arp_filter_netif_fn() — 软分发决策
   │
   ├─ ETHTYPE_VLAN → 遍历 netif_list → netif->vlanid == packet vid?
   │     ├─ 匹配 → 返回该 vlan_if[i]
   │     └─ 无匹配 → 返回 NULL (丢弃)
   │
   ├─ ETHTYPE_IP → 提取 dest IP → 遍历 netif_list
   │     ├─ ip_addr 匹配 && vlanid == 0 → 返回 vnet_if
   │     └─ 多播 → ip_in_multicast_group() 检查
   │
   └─ ETHTYPE_ARP → 提取 dest IP → 遍历 netif_list
         ├─ ip_addr 匹配 && vlanid == 0 → 返回 vnet_if
         └─ 无匹配 → 返回 netifIn (fallback)
   │
   ▼
ethernet_input 继续处理
   └─ 使用分发后的 netif 继续走 raw_afpacket_input / ip4_input
```

### 3.4 每个阶段的 Buffer 传递方式

| 阶段 | 传递方式 | 说明 |
|------|----------|------|
| NIC DMA → used_rx_buf_ring | **PA 指针** | 物理地址，零拷贝 |
| elem_ring_get → cma_pa_to_va | **VA 指针** | 转换为虚拟地址 |
| rx_callback → ethernet_input | **pbuf 引用** | pbuf* 指针传递 |
| raw_afpacket_input → tpacket_recv | **pbuf 引用** | pbuf* 指针传递 |
| tpacket_recv → ring buffer | **memcpy** | pbuf → ring buffer 拷贝一次 |
| ring buffer → App | **零拷贝** | mmap 直接读取 |
| App → lwip_sendto | **shm 或 memcpy** | shm 零拷贝或 IPC memcpy |
| lwip_sendto → ip4_output | **pbuf chain** | pbuf 链表引用 |
| ip4_output → ethernet_output | **pbuf chain** | pbuf 链表引用 |
| ethernet_output → ethif_link_output | **pbuf chain** | pbuf 链表引用 |
| ethif_link_output → pending_tx_buf_ring | **PA 指针** | pbuf VA → CMA PA |

---

## 4. 能力评估

### 4.1 支持的网络协议及功能矩阵

| 协议/功能 | 支持状态 | 配置宏 | 说明 |
|-----------|----------|--------|------|
| **IPv4** | ✅ 完整 | 默认启用 | TCP/UDP/ICMP/IGMP/ARP |
| **IPv6** | ❌ 禁用 | `LWIP_IPV6=0` | 未实现 |
| **TCP** | ✅ 完整 | 默认启用 | 状态机/重传/拥塞控制/SACK |
| **UDP** | ✅ 完整 | 默认启用 | 广播/组播/checksum |
| **RAW IP** | ✅ 完整 | `LWIP_RAW=1` | SOCK_RAW |
| **ARP** | ✅ 完整 | `LWIP_ARP=1` | 动态学习/静态条目 |
| **IGMP** | ✅ 完整 | `LWIP_IGMP=1` | v1/v2/v3 |
| **VLAN (802.1Q)** | ✅ 完整 | `ETHARP_SUPPORT_VLAN=1` | VID 分发/插入 |
| **Bridge (802.1D)** | ✅ 基础 | `LWIP_BRIDGE=1` | FDB 学习/老化，无 STP |
| **DHCP** | ✅ 客户端 | `LWIP_DHCP=1` | 发现/请求/续约 |
| **DNS** | ✅ 客户端 | `LWIP_DNS=1` | 递归查询 |
| **AF_PACKET** | ✅ 扩展 | SafeOS 定制 | raw_afpacket |
| **packet_mmap** | ✅ 部分 | SafeOS 定制 | TPACKET_V1 RX only |
| **LWFW 防火墙** | ✅ 扩展 | `NIO_LWIP_LWFW=1` | Ingress/Egress hooks |
| **IP Fragmentation** | ✅ 完整 | `LWIP_IPV4_FRAG=1` | 分片重组 |
| **IP Forwarding** | ⚠️ 可选 | `LWIP_IPV4_FORWARD=0` | 默认禁用 |
| **SNTP** | ✅ 可选 | `CONFIG_SNTP_SVC_ENABLE` | 时间服务 |
| **VIRT_BRG** | ✅ 可选 | `VIRT_BRG_SUPPORT` | hypervisor 网桥 |
| **VNET_OVER_IPC** | ✅ 可选 | `VNET_OVER_IPC_SUPPORT` | VM 通信 |

### 4.2 性能特征

#### 零拷贝 vs 拷贝路径

| 数据流阶段 | 零拷贝? | 拷贝次数 | 说明 |
|-----------|---------|----------|------|
| NIC DMA → pbuf (RX) | ✅ | 0 | DMA 直接写入 CMA，VA/PA 转换 |
| pbuf → packet_mmap ring | ❌ | 1 | tpacket_recv 中 memcpy |
| ring → App (packet_mmap) | ✅ | 0 | mmap 直接读取 |
| App → lwip_sendto (shm) | ✅ | 0 | 共享内存 |
| App → lwip_sendto (IPC) | ❌ | 1 | sys_unpack_data_from_mrs |
| lwIP 内部 pbuf chain | ✅ | 0 | 引用传递 |
| ethif_link_output TX | ❌ | 1 | pbuf copy 到 DMA buffer |

#### 内存使用模型

| 区域 | 大小 | 说明 |
|------|------|------|
| CMA 总大小 | 96MB | NSv 与 NIC 共享 |
| elem_ring x4 | ~32KB | 每个 ring 约 8KB (n=4096, 每个 elem=8B) |
| DMA Buffers | ~95MB | 约 60000 个 × 1576B |
| lwIP 内存池 | 动态 | ds_ring_mem_alloc |

#### 性能边界

基于 `lwip_sel4_performance_boundary.md` 分析:

| 指标 | 单核极限 | 4核极限 | 限制因素 |
|------|----------|---------|----------|
| **Max PPS** | ~156K | ~200K | tcpip_thread 单线程 |
| **Max Throughput (1500B)** | ~1.87 Gbps | ~2.4 Gbps | 单核心 + IPC 开销 |
| **单包延迟** | ~6.4μs | N/A | seL4_Recv + 协议栈处理 |
| **IPC 开销 (RX)** | ~150-710ns/packet | — | sel4_signal + seL4_Recv |

### 4.3 VLAN/bridge 隔离能力

| 能力 | 支持 | 说明 |
|------|------|------|
| **VLAN VID 分发** | ✅ | 通过 lwip_arp_filter_netif_fn 精确匹配 |
| **VLAN PCP 优先级** | ✅ | 从 VLAN TCI 提取，存入 pbuf->priority |
| **MAC_VLAN_FILTER** | ✅ | LWIP_HOOK_VLAN_CHECK 实现 VID 检查 |
| **VLAN 隔离** | ✅ | 无 VID 匹配则丢弃 |
| **Bridge FDB 学习** | ✅ | 源 MAC 学习，5 分钟老化 |
| **Bridge 泛洪** | ✅ | 未知单播泛洪到所有端口 |
| **Bridge STP** | ❌ | 不支持生成树协议 |
| **Bridge VLAN** | ❌ | bridgeif 不支持 802.1Q VLAN |

### 4.4 并发/线程安全模型

| 方面 | 实现 | 说明 |
|------|------|------|
| **tcpip_core_lock** | `LWIP_TCPIP_CORE_LOCKING=1` | 全局锁保护 lwIP 核心 |
| **RX 处理** | nic_rx_thread 获取锁后调用 rx_callback | 串行处理所有包 |
| **TX 处理** | event_loop 调用 lwIP socket API | 内部处理锁 |
| **select/poll** | select_thread 独立线程 | 与 event_loop 通过 mbox 通信 |
| **多核利用** | 差 | 单 tcpip_thread，多核 scaling 约 1.2x |
| **LWFW filter** | 在 LOCK_TCPIP_CORE 内执行 | 串行过滤 |

---

## 5. 优势 & 缺陷

### 5.1 相比 Standard lwIP 的增强点

| 增强 | 说明 |
|------|------|
| **硬 VLAN 分发** | lwip_arp_filter_netif_fn 实现精确 VID → netif 映射，超越标准 lwIP 的软 VLAN |
| **AF-PACKET 支持** | 通过 raw_afpacket_input/output 实现 Ethernet 层 packet socket |
| **packet_mmap** | 自定义 DSPACE + ringbuf 实现零拷贝抓包 (虽然 pbuf→ring 仍有一次拷贝) |
| **LWFW 防火墙** | Ingress/Egress filter hooks 支持 5-tuple 连接追踪 |
| **VIRT_BRG** | hypervisor 网桥支持 VM 间和 VM ↔ NSv 通信 |
| **VNET_OVER_IPC** | VM 通过 IPC 直接通信 |
| **用户态隔离** | seL4 微内核下，NIC 驱动崩溃不影响网络栈 |

### 5.2 潜在的性能瓶颈

| 瓶颈 | 影响程度 | 原因 |
|------|----------|------|
| **tcpip_thread 单线程** | **极高** | 所有 RX/TX 包串行处理，多核利用约 25% |
| **seL4 IPC 延迟** | **高** | 每次 RX 包约 150-710ns 开销 |
| **TCPIP_CORE_LOCK 竞争** | **中** | RX/TX 竞争同一锁 |
| **pbuf 分配/释放** | **中** | slab cache 仍有一定开销 |
| **TX memcpy** | **中** | ethif_link_output 中 pbuf copy |
| **packet_mmap 拷贝** | **低** | pbuf → ring buffer 一次拷贝 |
| **elem_ring 操作** | **低** | ~10-20ns，已最优 |

### 5.3 代码复杂度/维护性评估

| 方面 | 评分 | 说明 |
|------|------|------|
| **架构清晰度** | ⚠️ 中等 | NSv + lwIP + seL4 三层，跨进程共享复杂 |
| **代码复用** | ❌ 差 | packet_mmap.c 在 net-cap 和 tcpdump 中重复 |
| **API 稳定性** | ❌ 差 | `nsv/packet_mmap.h` 内部结构暴露给 App |
| **文档完整性** | ⚠️ 中等 | 分析文档齐全，但代码注释较少 |
| **测试覆盖** | 未知 | 未见单元测试文档 |
| **ABI 稳定性** | ❌ 差 | 无稳定 ABI 层，内部头文件直接暴露 |

**架构边界问题**:
- `nsv/packet_mmap.h` 同时作为内部实现和 App API
- `packet_mmap.c` 代码重复 (net-cap 和 tcpdump 各一份)
- App (tcpdump) 通过 CMakeLists.txt 显式依赖 NSv 内部

### 5.4 缺失的功能

| 功能 | 状态 | 说明 |
|------|------|------|
| **IPv6** | ❌ | `LWIP_IPV6=0` 禁用 |
| **TCP SACK** | ⚠️ 部分 | 代码中有 `saacksent`，但非完整实现 |
| **TCP BBR** | ❌ | 不支持 BBR 拥塞控制 |
| **TCP Fast Open** | ❌ | 不支持 TFO |
| **VLAN-aware Bridge** | ❌ | bridgeif 不支持 802.1Q VLAN |
| **STP (生成树)** | ❌ | bridgeif 不支持 |
| **VLAN Trunk** | ❌ | 只能单个 VID 分发 |
| **PACKET_TX_RING** | ❌ | 仅支持 RX Ring |
| **TPACKET_V3** | ❌ | 仅 TPACKET_V1 |
| **零拷贝 TX** | ❌ | pbuf → DMA buffer 仍需拷贝 |
| **802.1X** | ❌ | 不支持 |
| **DCB** | ❌ | 不支持 |

---

## 6. 提升方向

### 6.1 短期 (稳定性/性能优化)

| 优先级 | 方向 | 具体措施 | 预期收益 |
|--------|------|----------|----------|
| P0 | **批处理 RX** | nic_rx_thread 一次 seL4_Recv 后批量处理多个 elem_ring_get | ~20-30% 提升，减少 IPC 次数 |
| P0 | **预分配 pbuf 池** | 启动时预分配 1024 个 dma_pbuf，避免运行时分配 | ~10-15% 提升，消除分配开销 |
| P1 | **减小锁临界区** | 在 rx_callback 中尽早释放 TCPIP_CORE_LOCK | ~5-10% 提升，减少竞争 |
| P1 | **TX 异步化** | ethif_link_output 不等待 TX 完成，异步完成 | 降低 TX 延迟 |
| P2 | **修复假共享** | elem_ring 的 get_idx/put_idx 放入不同 cache line | 多核场景提升 |
| P2 | **packet_mmap 时间戳** | tpacket_hdr.tp_sec/usec 使用实际时间 | 支持抓包时间分析 |

### 6.2 中期 (功能增强)

| 方向 | 具体措施 | 前提条件 |
|------|----------|----------|
| **多 tcpip_thread** | 为每个 NIC queue 创建独立 tcpip_thread | 需要 lwIP 核心修改 |
| **PACKET_TX_RING** | 实现发送环，支持零拷贝 TX | 较大改动 |
| **TPACKET_V3** | 实现超时回收、超长 snaplen | 需参考 Linux 内核实现 |
| **VLAN Trunk** | 支持多个 VID 的 trunk 接口 | 需要 netif 扩展 |
| **VLAN-aware Bridge** | bridgeif 支持 802.1Q VLAN | 较大架构改动 |
| **创建稳定 ABI 层** | 分离 `include/nsv/packet_mmap_abi.h` | 公共接口层 |
| **抽取 libpacket_mmap** | 消除代码重复 | 重构 |

### 6.3 长期 (架构演进)

| 方向 | 说明 | 风险 |
|------|------|------|
| **并行 lwIP** | 解除 LWIP_TCPIP_CORE_LOCKING，多 worker 处理 | 需要大量 lwIP 核心修改 |
| **RDMA 支持** | 在 CNA 场景下引入 RDMA 绕过 CPU | 硬件依赖 |
| **XDP 近似** | 在 RX 路径早期做过滤/转发决策 | 需要新 hook 点 |
| **WireProtocol offload** | TSO/GSO/CHECKSUM_OFFLOAD 到 NIC | 驱动支持 |

---

## 附录 A: 核心文件索引

### NSv 核心

| 文件 | 职责 |
|------|------|
| `servers/net/src/main.c` | NSv 主入口、init_ethif、ethif_link_output、nic_rx_thread、event_loop、sys_net_* handlers |
| `servers/net/src/vlanif.c` | vlanif_init、low_level_output、vlanif_setup |
| `servers/net/src/bridge.c` | ethif_link_output_overload、vbridge_evt_loop、vbridge_port_output |
| `servers/net/src/ipc-if.c` | ipcif_evt_loop、work_thread、ipc_if_rx_proc |
| `servers/net/src/packet_mmap.c` | packet_mmap_set_ring、tpacket_recv、rb_write_tpacket |

### lwIP 核心

| 文件 | 职责 |
|------|------|
| `external/lwip_ds_mcu/src/netif/ethernet.c` | ethernet_input/output、lwip_arp_filter_netif_fn、lwip_hook_vlan_check_fn |
| `external/lwip_ds_mcu/src/core/ipv4/ip4.c` | ip4_input、ip4_output_if、ip_route |
| `external/lwip_ds_mcu/src/core/ipv4/etharp.c` | etharp_output、etharp_input、arp_find_entry |
| `external/lwip_ds_mcu/src/core/tcp.c` | tcp_input、tcp_output、tcp_connect、tcp_close |
| `external/lwip_ds_mcu/src/core/udp.c` | udp_input、udp_output、udp_bind |
| `external/lwip_ds_mcu/src/core/raw.c` | raw_afpacket_input/output、raw_input、raw_sendto |
| `external/lwip_ds_mcu/src/netif/bridgeif.c` | bridgeif_input、bridgeif_output、bridgeif_send_to_ports |
| `external/lwip_ds_mcu/src/netif/bridgeif_fdb.c` | FDB 学习/查找/老化 |

### seL4 适配层

| 文件 | 职责 |
|------|------|
| `libs/util_libs/liblwip/src/sys_arch_sel4.c` | sys_mutex/sem/mbox、lwip_malloc、sys_now |
| `libs/os_libs/libcore/include/core/elem_ring.h` | elem_ring 无锁队列实现 |
| `libs/os_libs/libcore/include/core/cma.h` | CMA 结构体和地址转换 |
| `libs/os_libs/libcore/src/ringbuffer.c` | ringbuf 读写实现 |

### LWFW 防火墙

| 文件 | 职责 |
|------|------|
| `libs/util_libs/liblwfw/src/lwfw.c` | lwfw ingress/egress filter、lwfw_ct 连接追踪 |
| `libs/util_libs/liblwfw/src/lwfw_lwct.c` | 连接跟踪表管理 |
| `libs/util_libs/liblwfw/src/lwfw_classify.c` | 5-tuple 规则匹配 |

---

## 附录 B: 配置宏参考

| 宏 | 值 | 说明 |
|----|---|------|
| `LWIP_NETCONN` | 1 | Netconn API |
| `LWIP_SOCKET` | 1 | BSD socket API |
| `LWIP_RAW` | 1 | RAW socket |
| `LWIP_TCPIP_CORE_LOCKING` | 1 | tcpip 核心锁 |
| `LWIP_SOCKET_NPOLL` | 1 | poll() 支持 |
| `MEMP_NUM_NETCONN` | 1024 | 最大 netconn 数 |
| `LWIP_IPV6` | 0 | IPv6 禁用 |
| `NIO_LWIP_LWFW` | 1 | 启用 LWFW |
| `NIO_LWIP_LWCT` | 1 | 启用连接追踪 |
| `ETHARP_SUPPORT_VLAN` | 1 | VLAN 支持 |
| `LWIP_ARP_FILTER_NETIF` | 1 | VLAN-aware netif 选择 |
| `VIRT_BRG_SUPPORT` | 可选 | hypervisor 网桥 |
| `VNET_OVER_IPC_SUPPORT` | 可选 | VM IPC 通信 |
| `USE_SEND_SMOOTH_QAV` | 可选 | QoS 流量整形 |
