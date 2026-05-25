---
type: entity
tags: [safeos, nsv, lwip, network, seL4, event-loop, cma, packet-mmap]
created: 2026-05-25
sources: [safeos-architecture]
---

# SafeOS NSv — Network Server

## 定义

NSv (Network Server) 是运行在 seL4 微内核之上的**用户态网络栈**，基于 lwIP 实现。它完全在用户态处理所有网络协议，不依赖内核网络子系统，是 SafeOS 网络架构的核心组件。

## 核心属性

| 属性 | 值 |
|------|-----|
| 位置 | `os-framework/servers/net/` |
| 服务名 | `/net` (注册到 PSv) |
| 底层栈 | lwIP (用户态) |
| 线程数 | 3个基础线程 + 可选工作线程 |
| 共享内存 | CMA (96MB) + DSPACE |

---

## 初始化序列

```
main()
  │
  ├─ net_resources_init()         // 堆/互斥锁/哈希表
  │
  ├─ init_ds_ring()              // CMA 分配 + ds_ring 创建 + NIC 授权
  │
  ├─ create_nic_thread()          // nic_rx_thread 创建 + 通知机制
  │
  ├─ tcpip_init(0, 0)            // lwIP TCP/IP 核心初始化
  │
  ├─ netif_add(&vnet_if, ...)    // 注册虚拟网卡 vnet_if
  │
  ├─ netif_set_up(&vnet_if)      // 启用网卡
  │
  ├─ sys_svc_reg("/net", ...)    // 注册 /net 服务到 PSv
  │
  └─ event_loop()                 // 进入主事件循环
```

**关键初始化函数**:

| 函数 | 位置 | 职责 |
|------|------|------|
| `net_resources_init()` | main.c:6123 | 初始化堆、互斥锁、poll hashtable |
| `init_ds_ring()` | main.c:3610 | CMA 分配 + ds_ring 创建 + NIC 授权 |
| `create_nic_thread()` | main.c:4922 | 创建 nic_rx_thread 和通知机制 |
| `tcpip_init()` | lwIP | 初始化 lwIP 核心锁和 tcpip 线程 |

---

## 线程模型

### 3 个基础线程

| 线程 | 优先级 | 栈大小 | 职责 |
|------|--------|--------|------|
| `event_loop` | SYS_PRIO_NETSVR | NET_THRD_STACK_SZ | 处理 App socket 请求 |
| `nic_rx_thread` | SYS_PRIO_NETSVR | NET_THRD_STACK_SZ | 处理 NIC RX 事件 |
| `timer_thread` | - | - | 心跳监控 (可选) |

### 可选线程

| 线程 | 条件编译 | 职责 |
|------|----------|------|
| `nsv_smooth_thread` | `USE_SEND_SMOOTH_QAV` | QoS CBS 平滑发送 |
| `work_thread` (N个) | `NSV_WORKER_SELECT_THREADS_ON` | 处理 select/poll 请求 |
| `iperf` | `CONFIG_NSV_IPERF2_ENABLE` | iperf 性能测试服务器 |
| `sntp_service` | `CONFIG_SNTP_SVC_ENABLE` | SNTP 时间服务 |

### nic_rx_thread 主循环

```c
static void *nic_rx_thread(void *arg)
{
    sys_bind_ntfn(nic_rx_ntfn);
    while (1) {
        seL4_Recv(nsv_nic_ep, &badge);
        if (badge == 0) {
            while (1) {
                union elem e = elem_ring_get(used_rx_buf_ring);
                if (e.pa) {
                    LOCK_TCPIP_CORE();
                    rx_callback((struct pbuf *)cma_pa_to_va(&cma, e.pa));
                    UNLOCK_TCPIP_CORE();
                } else break;
            }
        }
    }
}
```

---

## 内存架构 — CMA (96MB)

### CMA 布局

```
CMA Region (96MB)
├── elem_ring (4个环形缓冲区)
│   ├── empty_rx_buf_ring   (RX 空缓冲，NSv→NIC)
│   ├── used_rx_buf_ring    (RX 已收包，NIC→NSv)
│   ├── pending_tx_buf_ring (TX 待发，NSv→NIC)
│   └── used_tx_buf_ring    (TX 已完成，NIC→NSv)
├── DMA Buffers (pbuf)
└── Reserved Memory
```

### init_ds_ring 流程

```c
static int init_ds_ring(void)
{
    // 1. 从 CMA 分配 DMA 缓冲区 (96MB)
    sys_mem_map(getpid(), &cma.pa, &cma.va, CMA_SIZE, PAGE_DMA);

    // 2. 基于 CMA 创建 ds_ring
    ds_ring = ds_ring_init(cma.va, cma.pa, ..., ds, getpid(), NSV_NIC_DESC_SIZE);

    // 3. 等待 NIC 驱动注册
    sys_svc_wait(NIC_STR, SVC_WAIT_EXACT, &nic_ep);

    // 4. 授权 CMA 给 NIC 驱动
    sys_dspace_grant(ds, nic_ep, DSPACE_GRANT_SVC_EP, attr);
    sys_ds_ring_share(nic_ep, ds);
}
```

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
   │ sel4_signal(nic_rx_ntfn)                     │
   │                                              │
▼                                              │
nic_rx_thread()                                   │
   │                                              │
   │ seL4_Recv(nsv_nic_ep)                       │
   │                                              │
▼                                              │
elem_ring_get(used_rx_buf_ring) → 获取 buffer PA   │
   │                                              │
   │ cma_pa_to_va() → 转换为虚拟地址             │
   │                                              │
▼                                              │
rx_callback(pbuf)                                 │
   │                                              │
   │ LOCK_TCPIP_CORE()                           │
   │                                              │
▼                                              │
vnet_if.input(p, &vnet_if) = ethernet_input()    │
   │                                              │
   ├── ETH_P_IP  → ip_input() → raw_input() ──► raw_afpacket_input()
   │                    │                   │              │
   │                    │                   │         tpacket_recv()
   ├── ETH_P_ARP → etharp_input()          │              │
   └── 其他          → 未知协议            │              │
                                         │              │
                     API_EVENT(conn, RCVPLUS) ──────────┘
                           │
                           │ select()/poll() 唤醒 App
                           ▼
                        App recvfrom()
```

### rx_callback

```c
void rx_callback(void *ctx)
{
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

---

## 发包路径 (TX)

### ethif_link_output

```c
err_t ethif_link_output(struct netif *netif, struct pbuf *p)
{
    if (!nic_ready) return ERR_OK;

    // 1. 检查是否有完成的 TX 包可以释放
    if (sync_mutex_trylock(&used_tx_buf_mutex) == 0) {
        free_complete_tx_packet_pbuf();
        sync_mutex_unlock(&used_tx_buf_mutex);
    }

    // 2. 检查 pending_tx_buf_ring 是否有空间
    int pending_read = elm_ring_avail_size(pending_tx_buf_ring);
    int is_full = (pending_read == (pending_tx_buf_ring->n - 1));

    if (!is_full) {
        // 3. 将 pbuf PA 放入 pending_tx_buf_ring
        union elem e = {.pa = cma_va_to_pa(&cma, (vaddr_t)p)};
        elem_ring_put(pending_tx_buf_ring, e);

        // 4. 通知 NIC
        sel4_signal(nic_tx_ntfn);
    }
    return ERR_OK;
}
```

### TX 流程

```
App sendto()
   │
   │ lwIP 处理
   ▼
ethernet_output()
   │
   ├── raw_afpacket_output()  // 发送抓包通知
   │
   └── netif->linkoutput() = ethif_link_output()
        │
        ▼
   elem_ring_put(pending_tx_buf_ring)
        │
        ▼
   sel4_signal(nic_tx_ntfn) → NIC 驱动
```

---

## Socket Syscall 处理

event_loop 中的 switch-case 分发:

| Syscall | Handler | lwIP 函数 |
|---------|---------|-----------|
| `SYS_NET_SOCKET` | `sys_socket_nb()` | `lwip_socket()` |
| `SYS_NET_CONNECT` | `sys_connect_nb()` | `lwip_connect()` |
| `SYS_NET_BIND` | `sys_bind_nb()` | `lwip_bind()` |
| `SYS_NET_LISTEN` | `sys_listen_nb()` | `lwip_listen()` |
| `SYS_NET_ACCEPT` | `sys_accept()` | `lwip_accept()` |
| `SYS_NET_SENDTO` | `sys_sendto_nb()` | `lwip_sendto()` |
| `SYS_NET_RECVFROM` | `sys_recvfrom_nb()` | `lwip_recvfrom()` |
| `SYS_NET_CLOSE` | `sys_close_nb()` | `lwip_close()` |
| `SYS_NET_SELECT` | `select_thread()` | `lwip_select()` |

---

## seL4 IPC 交互

### IPC 端点

| 端点 | 用途 |
|------|------|
| `svc_ep` | App 与 NSv 之间的 socket syscall |
| `nsv_nic_ep` | NIC 驱动与 NSv 之间的 RX 事件 |
| `net_pm_ep` | 电源管理暂停/恢复信号 |

### 通知机制

| 通知 | 触发者 | 接收者 | 用途 |
|------|--------|--------|------|
| `nic_rx_ntfn` | NIC 驱动 | nic_rx_thread | RX 数据可用 |
| `nic_tx_ntfn` | NSv | NIC 驱动 | TX 数据待发送 |

---

## 设计特点

**优点**:
- 极简内核 — seL4 只做 IPC 和内存管理，网络全在用户态
- 可预测性 — 网络延迟不依赖内核调度
- 隔离性 — NIC 驱动崩溃不影响网络栈

**限制**:
- 性能不如原生 Linux — 用户态复制、中断处理开销
- 协议支持有限 — 无 IPv6 (`LWIP_IPV6 = 0`)
- 实时性依赖 seL4 调度 — 中断→notification→线程唤醒链路长

---

## 关键文件清单

| 文件 | 职责 |
|------|------|
| `servers/net/src/main.c` | NSv 主入口、event_loop、socket 处理、TX/RX 路径 |
| `servers/net/src/packet_mmap.c` | AF-PACKET TPACKET ring 设置、tpacket_recv 回调 |
| `servers/net/include/nsv/nsv.h` | NSv 常量、宏定义 |
| `servers/net/include/nsv/packet_mmap.h` | packet_mmap_info 结构体、常量 |
| `libs/os_libs/libcore/src/ds_ring.c` | DS-RING 实现 |
| `libs/os_libs/libcore/include/core/elem_ring.h` | elem_ring 定义 |
| `external/lwip_ds_mcu/src/api/sockets.c` | BSD socket → lwIP 映射 |
| `external/lwip_ds_mcu/src/netif/ethernet.c` | ethernet_input/output |
| `external/lwip_ds_mcu/src/core/raw.c` | raw_afpacket_input/output |
| `libs/util_libs/liblwip/src/sys_arch_sel4.c` | seL4 sys_arch 适配层 |

---

## 相关概念

- [[entities/linux/lwip/lwip-sel4-function]] — lwIP 在 seL4 上运行的函数级深度分析
- [[entities/linux/lwip/lwip-sel4-ipc]] — seL4 notification/endpoint 通信机制
- [[entities/linux/lwip/lwip-sel4-performance-boundary]] — seL4 + lwIP 性能边界分析
- [[entities/linux/safeos/safeos-packet-mmap]] — AF-PACKET + TPACKET 抓包实现
- [[entities/linux/safeos/safeos-network-implementation]] — SafeOS 网络实现完整分析
- [[entities/linux/lwip/lwip-elem-ring]] — 无锁单生产者/单消费者环形缓冲区
- [[entities/linux/lwip/lwip-nsv-event-loop]] — NSv 事件循环、select/poll 实现

## 来源详情

- [[sources/safeos-architecture]] — SafeOS Architecture & Design Documents
