# SafeOS NSv (Network Server) 深度分析

> 文档版本: 1.0
> 更新日期: 2026/04/13
> 代码路径: `/home/shiyang/nio/nt35/safeos/os-framework/servers/net/`

---

## 1. 概述

NSv (Network Server) 是运行在 seL4 微内核之上的**用户态网络栈**，基于 lwIP 实现。它完全在用户态处理所有网络协议，不依赖内核网络子系统。

| 属性 | 值 |
|------|-----|
| 位置 | `os-framework/servers/net/` |
| 服务名 | `/net` (注册到 PSv) |
| 底层栈 | lwIP (用户态) |
| 线程数 | 3个基础线程 + 可选工作线程 |
| 共享内存 | CMA (96MB) + DSPACE |

---

## 2. 初始化序列

**main.c:6308** — NSv 主入口:

```c
int main(int argc, char *argv[])
{
    net_resources_init();      // 初始化堆、互斥锁、哈希表
        ↓
    init_ds_ring();            // 创建 CMA 共享内存 + ds_ring
        ↓
    create_nic_thread();       // 创建 nic_rx_thread
        ↓
    tcpip_init(0, 0);          // 初始化 lwIP TCP/IP 核心
        ↓
    netif_add(&vnet_if, ...); // 注册虚拟网卡
        ↓
    netif_set_up(&vnet_if);    // 启用网卡
        ↓
    sys_svc_reg("/net", ...); // 注册 /net 服务到 PSv
        ↓
    event_loop();              // 进入主事件循环
}
```

**关键初始化函数**:

| 函数 | 位置 | 职责 |
|------|------|------|
| `net_resources_init()` | main.c:6123 | 初始化堆、互斥锁、poll hashtable |
| `init_ds_ring()` | main.c:3610 | CMA 分配 + ds_ring 创建 + NIC 授权 |
| `create_nic_thread()` | main.c:4922 | 创建 nic_rx_thread 和通知机制 |
| `tcpip_init()` | lwIP | 初始化 lwIP 核心锁和 tcpip 线程 |

---

## 3. 线程模型

### 3.1 基础线程

| 线程 | 优先级 | 栈大小 | 职责 |
|------|--------|--------|------|
| `event_loop` | SYS_PRIO_NETSVR | NET_THRD_STACK_SZ | 处理 App socket 请求 |
| `nic_rx_thread` | SYS_PRIO_NETSVR | NET_THRD_STACK_SZ | 处理 NIC RX 事件 |
| `timer_thread` | - | - | 心跳监控 (可选) |

### 3.2 可选线程

| 线程 | 条件编译 | 职责 |
|------|----------|------|
| `nsv_smooth_thread` | `USE_SEND_SMOOTH_QAV` | QoS CBS 平滑发送 |
| `work_thread` (N个) | `NSV_WORKER_SELECT_THREADS_ON` | 处理 select/poll 请求 |
| `iperf` | `CONFIG_NSV_IPERF2_ENABLE` | iperf 性能测试服务器 |
| `sntp_service` | `CONFIG_SNTP_SVC_ENABLE` | SNTP 时间服务 |
| `echoserver` | `CONFIG_NSV_ECHO_ENABLE` | Echo 服务器 |

### 3.3 NIC RX Thread

**main.c:4874** — nic_rx_thread 主循环:

```c
static void *nic_rx_thread(void *arg)
{
    sys_bind_ntfn(nic_rx_ntfn);  // 绑定 RX 通知

    while (1) {
        seL4_Recv(nsv_nic_ep, &badge);  // 等待 NIC 事件

        if (badge == 0) {
            // 处理收到的包
            while (1) {
                union elem e = elem_ring_get(used_rx_buf_ring);
                if (e.pa) {
                    LOCK_TCPIP_CORE();
                    rx_callback((struct pbuf *)cma_pa_to_va(&cma, e.pa));
                    UNLOCK_TCPIP_CORE();
                } else {
                    break;
                }
            }
        }
    }
}
```

---

## 4. 内存架构

### 4.1 CMA (Contiguous Memory Area)

**main.c:281**:
```c
#define CMA_SIZE    0x6000000ul  // 96MB
```

CMA 由 NSv 分配，用于:
1. **DMA 缓冲区** — NIC 驱动和 NSv 共享的包缓冲区
2. **DS Ring** — 描述符环，存储 NIC 和 NSv 之间的缓冲区描述符

### 4.2 CMA 布局

```
CMA Region (96MB)
├── elem_ring (4个环形缓冲区)
│   ├── empty_rx_buf_ring   (RX 空缓冲)
│   ├── used_rx_buf_ring    (RX 已收包)
│   ├── pending_tx_buf_ring (TX 待发)
│   └── used_tx_buf_ring    (TX 已完成)
├── DMA Buffers (pbuf)
└── Reserved Memory (NIC 初始化参数)
```

### 4.3 init_ds_ring

**main.c:3610**:

```c
static int init_ds_ring(void)
{
    // 1. 从 CMA 分配 DMA 缓冲区 (96MB)
    sys_mem_map(getpid(), &cma.pa, &cma.va, CMA_SIZE, PAGE_DMA);

    // 2. 基于 CMA 创建 ds_ring
    ds_ring = ds_ring_init(cma.va, cma.pa, (cma.size - CMA_RESERVED_MEM_SIZE),
                           ds, getpid(), NSV_NIC_DESC_SIZE);

    // 3. 等待 NIC 驱动注册
    sys_svc_wait(NIC_STR, SVC_WAIT_EXACT, &nic_ep);

    // 4. 授权 CMA 给 NIC 驱动
    sys_dspace_grant(ds, nic_ep, DSPACE_GRANT_SVC_EP, attr);
    sys_ds_ring_share(nic_ep, ds);
}
```

---

## 5. Socket Syscall 处理

event_loop 中的 switch-case 分发:

| Syscall | Handler | lwIP 函数 |
|---------|---------|-----------|
| `SYS_NET_SOCKET` | `sys_socket_nb()` (main.c:979) | `lwip_socket()` |
| `SYS_NET_SOCKETPAIR` | `sys_socketpair_nb()` (main.c:1036) | `lwip_socketpair()` |
| `SYS_NET_CONNECT` | `sys_connect_nb()` | `lwip_connect()` |
| `SYS_NET_BIND` | `sys_bind_nb()` | `lwip_bind()` |
| `SYS_NET_LISTEN` | `sys_listen_nb()` | `lwip_listen()` |
| `SYS_NET_ACCEPT` | `sys_accept()` | `lwip_accept()` |
| `SYS_NET_SENDTO` | `sys_sendto_nb()` | `lwip_sendto()` |
| `SYS_NET_RECVFROM` | `sys_recvfrom_nb()` | `lwip_recvfrom()` |
| `SYS_NET_CLOSE` | `sys_close_nb()` | `lwip_close()` |
| `SYS_NET_SETSOCKOPT` | `sys_setsockopt()` (main.c:2043) | `lwip_setsockopt()` |
| `SYS_NET_GETSOCKOPT` | `sys_getsockopt()` | `lwip_getsockopt()` |
| `SYS_NET_SELECT` | `select_thread()` | `lwip_select()` |
| `SYS_NET_POLL` | `sys_poll()` | `lwip_poll()` |

### 5.1 sys_socket_nb

**main.c:979**:

```c
static int sys_socket_nb(sel4_msg_info_t info, sel4_word badge)
{
    int domain = sel4_get_mr(0);    // e.g., AF_PACKET
    int type = sel4_get_mr(1);      // e.g., SOCK_RAW
    int protocol = sel4_get_mr(2);  // e.g., ETH_P_ALL
    pid_t pid = sys_get_pid_from_badge(badge);

    socket = lwip_socket(domain, type, protocol);

    net_socket_info[socket].owner = pid;
    netstat_add_info_by_badge(socket, badge);

    sys_reply_with_one_direct(0, socket);
    return MSG_REPLIED;
}
```

### 5.2 PACKET_RX_RING setsockopt

**main.c:2073**:

```c
if (optname == PACKET_RX_RING) {
    struct lwip_sock *sock = get_socket(socket);

    if (NETCONNTYPE_GROUP(sock->conn->type) == NETCONN_PACKET
        && optlen == sizeof(struct packet_mmap_param)) {
        err = packet_mmap_set_ring(socket, badge, (void *)opt,
                                   optlen, &packet_mmap_list);
    }
}
```

---

## 6. lwIP 集成

### 6.1 虚拟网卡初始化

**main.c:6352**:

```c
vnet_if.name[0] = PHYSICAL_IFNAME[0];  // 'e' 或 'v'
vnet_if.name[1] = PHYSICAL_IFNAME[1];  // 'n' 或 'i'

netif = netif_add(&vnet_if,
                  &addr, &netmask, &gw,
                  NULL,
                  init_ethif,        // 初始化函数
                  ethernet_input);   // 输入函数

netif_set_link_up(&vnet_if);
netif_set_up(&vnet_if);
```

### 6.2 网卡参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| IP 地址 | 172.20.0.1 | |
| 子网掩码 | 255.255.255.0 | |
| 网关 | 172.20.0.12 | |
| MTU | 1500 | |

### 6.3 IP 地址配置

```c
IP4_ADDR(&addr, 172, 20, 0, 1);
IP4_ADDR(&netmask, 255, 255, 255, 0);
IP4_ADDR(&gw, 172, 20, 0, 12);
```

---

## 7. 收包路径 (RX)

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
   ├── ETH_P_ARP → etharp_input()            │              │
   └── 其他          → 未知协议              │              │
                                            │              │
                        API_EVENT(conn, RCVPLUS) ──────────┘
                              │
                              │ select()/poll() 唤醒 App
                              ▼
                           App recvfrom()
```

### 7.1 rx_callback

**main.c:4694**:

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

## 8. 发包路径 (TX)

### 8.1 ethif_link_output

**main.c:3728** — TX 输出到 NIC:

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

### 8.2 TX 流程

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

## 9. AF-PACKET + TPACKET 实现

### 9.1 packet_mmap_set_ring

**packet_mmap.c:265** — NSv 侧 ring 设置:

```c
int packet_mmap_set_ring(int socket, seL4_Word badge,
                         const void *packet_mmap_param, unsigned int len,
                         struct packet_mmap_dspace_node *packet_mmap_list)
{
    struct packet_mmap_param *param = (struct packet_mmap_param *)packet_mmap_param;
    pid_t pid = sys_get_pid_from_badge(badge);

    // 1. 将 App 的 dspace 映射到 NSv 地址空间
    struct packet_mmap_dspace_node *ret = packet_mmap_add_node(app_ds, pid, socket, packet_mmap_list);
    sys_dspace_map(app_ds, &(dspace_info->va), ...);

    // 2. 填充 packet_mmap_info
    struct packet_mmap_info *packet_info = (struct packet_mmap_info *)(dspace_info->va);
    packet_info->rx_buf_offset    = RX_BUF_RING_OFFSET;
    packet_info->pid             = pid;
    packet_info->socket          = socket;
    packet_info->tp_frame_size   = req.tp_frame_size;
    packet_info->netconn         = sock->conn;

    // 3. 初始化 ringbuf
    struct ringbuf *rbuf = packet_mmap_rbuf(packet_info);
    init_rx_buf_ring(rbuf_size, rbuf);

    // 4. 注册数据包接收回调
    sock->conn->pcb.raw_afpacket->recv_arg = packet_info;
    sock->conn->pcb.raw_afpacket->recv     = tpacket_recv;
    sock->conn->callback                    = packet_mmap_event_callback;
}
```

### 9.2 tpacket_recv 回调

**packet_mmap.c:122**:

```c
static uint8_t tpacket_recv(void *arg, struct raw_pcb *pcb,
                            struct pbuf *p_buf, const ip_addr_t *addr)
{
    struct packet_mmap_info *packet_node = (struct packet_mmap_info *)arg;
    struct ringbuf *rx_buf_ring = packet_mmap_rbuf(packet_node);

    int write_avail = rb_write_avail(rx_buf_ring);

    if (write_avail > DEFAULT_TP_FRAME_SIZE) {
        // 写入 tpacket_hdr + payload + padding
        rb_write_tpacket(rx_buf_ring, p_buf, packet_node->tp_frame_size);
        // 唤醒 select/poll
        API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0);
    } else {
        LWIP_DIAG_EXTEND_STATS_INC(stats_diag_ext.rawafpacket_mbox_errs);
    }
    return 0;
}
```

### 9.3 DSPACE 布局

```
DSPACE (4MB, App 与 NSv 共享)
├── packet_mmap_info (固定大小，约 32 字节)
├── ringbuf (读写索引)
└── TPACKET 帧循环队列 (tp_frame_size × tp_frame_nr)
```

---

## 10. 电源管理

### 10.1 状态

| 状态 | 值 | 说明 |
|------|-----|------|
| `NET_PM_STATE_RUNNING` | 0 | 运行中 |
| `NET_PM_STATE_SUSPEND` | 1 | 暂停 |
| `NET_PM_STATE_RESUME` | 2 | 恢复 |

### 10.2 暂停/恢复

| 函数 | 位置 | 职责 |
|------|------|------|
| `net_thread_suspend()` | main.c | 暂停所有网络线程 |
| `net_thread_resume()` | main.c | 恢复所有网络线程 |
| `net_suspend()` | main.c | 处理暂停请求 |
| `net_resume()` | main.c | 处理恢复请求 |

---

## 11. 服务注册

### 11.1 NSv 服务端点

**main.c:6492**:
```c
err = sys_svc_reg(NSV_SVC_NAME, svc_ep, 0);  // 注册 /net 服务
```

### 11.2 NIC 接口

**main.c:4931**:
```c
err = sys_svc_reg(NSV_NIC_INTERFACE_NAME, nsv_nic_ep, 0);
```

---

## 12. 关键常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `CMA_SIZE` | 0x6000000 (96MB) | CMA 总大小 |
| `MTU_SIZE` | 1500 | 最大传输单元 |
| `DEFAULT_TP_FRAME_SIZE` | 2048 | TPACKET 每帧大小 |
| `DEFAULT_TP_FRAME_NR` | 1024 | TPACKET 帧总数 |
| `DMA_BUF_CACHE_MAX_NUM` | 4096*4 | DMA 缓冲区缓存最大数量 |
| `NSV_SVC_NAME` | "/net" | NSv 服务名 |
| `NSV_NIC_INTERFACE_NAME` | "nic_interface" | NIC 接口名 |

---

## 13. 性能统计

event_loop 维护的统计数据 (`NET_PERF_STATS_INC`):

| 统计项 | 说明 |
|--------|------|
| `evl_socket_cnts` | socket 调用次数 |
| `evl_bind_cnts` | bind 调用次数 |
| `evl_connect_cnts` | connect 调用次数 |
| `evl_accept_cnts` | accept 调用次数 |
| `evl_sendto_cnts` | sendto 调用次数 |
| `open_socket_success_cnts` | 成功打开的 socket 数 |
| `open_socket_fail_cnts` | 打开失败的 socket 数 |
| `lwip_mem_bufs_alllocated` | lwIP 内存分配成功次数 |
| `lwip_mem_bufs_failures` | lwIP 内存分配失败次数 |

---

## 14. 关键文件清单

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

## 15. 与 seL4 的交互

### 15.1 IPC 端点

| 端点 | 用途 |
|------|------|
| `svc_ep` | App 与 NSv 之间的 socket syscall |
| `nsv_nic_ep` | NIC 驱动与 NSv 之间的 RX 事件 |
| `net_pm_ep` | 电源管理暂停/恢复信号 |

### 15.2 通知机制

| 通知 | 触发者 | 接收者 | 用途 |
|------|--------|--------|------|
| `nic_rx_ntfn` | NIC 驱动 | nic_rx_thread | RX 数据可用 |
| `nic_tx_ntfn` | NSv | NIC 驱动 | TX 数据待发送 |

---

## 16. 设计特点

**优点**:
- 极简内核 — seL4 只做 IPC 和内存管理，网络全在用户态
- 可预测性 — 网络延迟不依赖内核调度
- 隔离性 — NIC 驱动崩溃不影响网络栈

**限制**:
- 性能不如原生 Linux — 用户态复制、中断处理开销
- 协议支持有限 — 无 IPv6 (`LWIP_IPV6 = 0`)
- 实时性依赖 seL4 调度 — 中断→notification→线程唤醒链路长
