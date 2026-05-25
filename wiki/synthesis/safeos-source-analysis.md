---
type: synthesis
tags: [safeos, lwip, lwfw, network, analysis, source-code]
created: 2026-05-25
sources: [safeos-lwip-core, safeos-lwip-extensions, safeos-lwfw]
---

# SafeOS 网络栈源码分析报告

**分析日期**: 2026/05/25
**远程机器**: 10.88.2.77
**源码路径**:
- lwIP: `/home/shiyang/nio/nt35/safeos/external/lwip_ds_mcu/src/`
- NIO封装: `/home/shiyang/nio/nt35/safeos/libs/util_libs/liblwip/`
- LWFW防火墙: `/home/shiyang/nio/nt35/safeos/libs/util_libs/liblwfw/`
- 网络驱动: `/home/shiyang/nio/nt35/safeos/drivers/net/dwmac/`
- LWFW Agent: `/home/shiyang/nio/nt35/safeos/os-framework/servers/daemons/lwfw_agent/`

---

## 1. lwIP 核心数据结构

### 1.1 `struct netif` — 网络接口

**文件**: `lwip_ds_mcu/src/include/lwip/netif.h`

```c
struct netif {
  struct netif *next;                    // 链表下一个接口
#if LWIP_IPV4
  ip_addr_t ip_addr;                     // IPv4 地址
  ip_addr_t netmask;
  ip_addr_t gw;
#endif
#if LWIP_IPV6
  ip_addr_t ip6_addr[LWIP_IPV6_NUM_ADDRESSES];
  u8_t ip6_addr_state[LWIP_IPV6_NUM_ADDRESSES];
  u32_t ip6_addr_valid_life[...];
  u32_t ip6_addr_pref_life[...];
#endif
  netif_input_fn input;                  // 输入回调 (pbuf → TCPIP栈)
#if LWIP_IPV4
  netif_output_fn output;                // IPv4 输出回调 (通常是 etharp_output)
#endif
  netif_linkoutput_fn linkoutput;        // 链路层输出 (ethernet_output)
  netif_drv_op_fn drv_op;               // [扩展] 驱动操作 (promisc等)
#if LWIP_IPV6
  netif_output_ip6_fn output_ip6;
#endif
  netif_status_callback_fn status_callback;
  netif_status_callback_fn link_callback;
  void *state;                           // 驱动私有状态指针
  void* client_data[...];                 // 客户端数据扩展
  u16_t chksum_flags;                    // [扩展] Checksum控制
  u16_t mtu;
  u8_t hwaddr[NETIF_MAX_HWADDR_LEN];    // MAC地址
  u8_t hwaddr_len;
  u8_t flags;                            // NETIF_FLAG_*
  char name[2];                          // 接口名缩写 (如 "et")
  char *fullname;                        // [扩展] 完整接口名
  u8_t num;                              // 接口编号
#if MIB2_STATS
  struct stats_mib2_netif_ctrs mib2_counters;
#endif
#if ETHARP_SUPPORT_VLAN
  u16_t vlanid;                          // [扩展] VLAN ID
#endif
  struct netif_pkt_stats stats;          // [扩展] 收发包统计
};
```

**SafeOS 自定义扩展字段**:
- `drv_op` / `NETIF_DRV_OP_*` — 驱动层混杂/多播控制
- `chksum_flags` — per-netif checksum offload 控制 (`NETIF_CHECKSUM_GEN_*`)
- `fullname` — 超过2字符的接口全名
- `stats` (`struct netif_pkt_stats`) — 64位收发包计数器 (`rx_packets/rx_bytes/tx_packets/tx_bytes`)
- `vlanid` — VLAN 标签 (通过 `ETHARP_SUPPORT_VLAN` 配置)
- `_lwct` in `struct pbuf` — LWCT connection track 指针嵌入

**关键宏/标志**:
- `NETIF_FLAG_UP` — 接口启用
- `NETIF_FLAG_ETHARP` / `NETIF_FLAG_ETHERNET` — 以太网
- `NETIF_FLAG_LINK_UP` — 链路状态
- `NETIF_FLAG_FORWARD` — 支持IP转发
- `STATISTICS_ADD(packets, bytes, pbuf_len)` — 防溢出的64位统计

---

### 1.2 `struct pbuf` — 数据包缓冲区

**文件**: `lwip_ds_mcu/src/include/lwip/pbuf.h`

```c
struct pbuf {
  struct pbuf *next;          // 链表中下一个pbuf
  void *payload;              // 数据指针
#if LWIP_SO_TIMESTAMPING
  struct timespec timestamp;   // [扩展] 硬件时间戳
#endif
  u16_t tot_len;              // 本pbuf + 后续pbuf总长度
  u16_t len;                  // 本pbuf数据长度
  u8_t type_internal;         // pbuf类型 (RAM/ROM/REF/POOL) + 来源
  u8_t priority;              // [扩展] buffer priority
  u8_t flags;                 // PBUF_FLAG_* (PUSH/MCASTLOOP/TCP_FIN等)
  LWIP_PBUF_REF_T ref;        // 引用计数
  u8_t if_idx;                // 输入接口索引
#ifdef NIO_LWIP_LWCT
  u64_t _lwct;                // [扩展] 连接跟踪指针 + 状态 (低3位)
#endif
};
```

**SafeOS 扩展 (`NIO_LWIP_LWCT`)**:
- `_lwct` 字段将 `struct lwct_conn*` 指针和 `lwct_state_t` 状态合并存储:
  - 低3位 = lwct_state
  - 其余位 = conntrack conn 指针
- 访问函数: `pbuf_lwct()`, `lwct_conn_get_from_pbuf()`, `lwct_conn_set_to_pbuf()`

**pbuf 类型**:
- `PBUF_RAM` — 堆分配，用于TX
- `PBUF_ROM` — 只读，指向ROM
- `PBUF_REF` — 引用，可变数据
- `PBUF_POOL` — 内存池分配，用于RX

**flags**:
- `PBUF_FLAG_PUSH` — 立即递送应用
- `PBUF_FLAG_IS_CUSTOM` — 自定义释放回调
- `PBUF_FLAG_MCASTLOOP` — 组播环回
- `PBUF_FLAG_LLBCAST` / `LLMCAST` — 链路层广播/组播
- `PBUF_FLAG_TCP_FIN` — TCP FIN
- `PBUF_FLAG_TX_TSTAMP` — 请求TX时间戳

---

### 1.3 `struct tcp_pcb` — TCP协议控制块

**文件**: `lwip_ds_mcu/src/include/lwip/tcp.h`

```c
struct tcp_pcb {
  IP_PCB;                        // 宏: local_ip/remote_ip/netif_idx/so_options/tos/ttl

  u16_t remote_port;             // 远端端口
  tcpflags_t flags;              // TF_* 标志
  // flags: TF_ACK_DELAY/TF_ACK_NOW/TF_INFR/TF_CLOSEPEND/
  //         TF_RXCLOSED/TF_FIN/TF_NODELAY/TF_WND_SCALE/
  //         TF_TIMESTAMP/TF_SACK/TF_RTO/TCP_DELAY_ACK_ENHANCE...

  u8_t polltmr, pollinterval;
  u8_t last_timer;
  u32_t tmr;
  u32_t recv_tmr_start;

  // Receiver
  u32_t rcv_nxt;                 // 期望的下一个seq
  tcpwnd_size_t rcv_wnd;         // 接收窗口
  tcpwnd_size_t rcv_ann_wnd;     // 通告窗口
  u32_t rcv_ann_right_edge;

#if LWIP_TCP_SACK_OUT
  struct tcp_sack_range rcv_sacks[LWIP_TCP_MAX_SACK_NUM];
#endif

  s16_t rtime;                   // 重传定时器
  u16_t mss;                     // MSS

  // RTT估计
  u32_t rttest, rtseq;
  s16_t sa, sv;                  // Van Jacobson 估算
  s16_t rto;                     // 重传超时
  u8_t nrtx;                    // 重传次数

  // 快速重传/恢复
  u8_t dupacks;
  u32_t lastack;

  // 拥塞控制
  tcpwnd_size_t cwnd;            // 拥塞窗口
  tcpwnd_size_t ssthresh;

  u32_t rto_end;

  // Sender
  u32_t snd_nxt;
  u32_t snd_wl1, snd_wl2;
  u32_t snd_lbb;
  tcpwnd_size_t snd_wnd;
  tcpwnd_size_t snd_buf;         // 发送缓冲区大小
  u16_t snd_queuelen;
#if TCP_OVERSIZE
  u16_t unsent_oversize;
#endif
  tcpwnd_size_t bytes_acked;

  struct tcp_seg *unsent;
  struct tcp_seg *unacked;
  struct tcp_seg *ooseq;         // 无序队列

  struct pbuf *refused_data;

  // Callbacks
  tcp_sent_fn sent;
  tcp_recv_fn recv;
  tcp_connected_fn connected;
  tcp_poll_fn poll;
  tcp_err_fn errf;

#if LWIP_TCP_TIMESTAMPS
  u32_t ts_lastacksent, ts_recent;
#endif

  u32_t keep_idle;
#if LWIP_TCP_KEEPALIVE
  u32_t keep_intvl, keep_cnt;
#endif

  u8_t keep_cnt_sent;

#if LWIP_WND_SCALE
  u8_t snd_scale, rcv_scale;
#endif
};
```

**SafeOS/NIO 扩展**:
- `TCP_DELAY_ACK_ENHANCE` 标志: `TF_NO_QUICKACK`, `ack_num`, `ack_num_max`
- TCP扩展参数通过 `lwipopts.h` 配置:
  - `TCP_KEEPIDLE_DEFAULT = 10000ms`
  - `TCP_KEEPINTVL_DEFAULT = 3000ms`
  - `TCP_KEEPCNT_DEFAULT = 5`
  - `LWIP_TCP_SACK_OUT = 1`, `TCP_MAX_SACK_NUM = 8`
  - `LWIP_WND_SCALE = 1`, `TCP_RCV_SCALE = 10`
  - `TCP_MSS = 1460`
  - `TCP_SND_BUF = 32 * TCP_MSS`, `TCP_WND = 96 * TCP_MSS`

**struct tcp_pcb_listen**: 监听PCB，与 `tcp_pcb` 共用前导结构但补齐至相同大小。

---

### 1.4 `struct udp_pcb` — UDP协议控制块

**文件**: `lwip_ds_mcu/src/include/lwip/udp.h`

```c
struct udp_pcb {
  IP_PCB;                        // local_ip/remote_ip/netif_idx/so_options/tos/ttl
  struct udp_pcb *next;
  u8_t flags;                    // UDP_FLAGS_*
  u16_t local_port, remote_port;
  ip_addr_t netstat_remote_ip;   // [扩展] 最后收到包的远端IP
  u16_t netstat_remote_port;     // [扩展] 最后收到包的远端端口

#if LWIP_MULTICAST_TX_OPTIONS
  ip4_addr_t mcast_ip4;
  u8_t mcast_ifindex;
  u8_t mcast_ttl;
#endif

#if LWIP_IGMP
  ip4_addr_t mcast_group[LWIP_MAX_NUM_MCAST_GROUP];
#endif

#if LWIP_UDPLITE
  u16_t chksum_len_rx, chksum_len_tx;
#endif

  udp_recv_fn recv;
  void *recv_arg;
};
```

---

### 1.5 `struct ip_pcb` / `IP_PCB` 宏 — 通用PCB头部

**文件**: `lwip_ds_mcu/src/include/lwip/ip.h`

```c
#define IP_PCB                             \
  ip_addr_t local_ip;                      \
  ip_addr_t remote_ip;                     \
  u8_t netif_idx;                         \
  u8_t so_options;                        \
  u8_t tos;                               \
  u8_t ttl                                \
  IP_PCB_NETIFHINT
```

**Socket Options** (`so_options`):
- `SOF_REUSEADDR`, `SOF_KEEPALIVE`, `SOF_BROADCAST`
- `SOF_TSTAMP_TX_HW` / `SOF_TSTAMP_RX_HW` (时间戳)
- `SOF_PTP_EVENT_SOCK` (PTP事件)

**ip_globals** — 当前处理中的IP包上下文:
```c
struct ip_globals {
  struct netif *current_netif;      // 接收接口
  struct netif *current_input_netif;
  const struct ip_hdr *current_ip4_header;
  struct ip6_hdr *current_ip6_header;
  u16_t current_ip_header_tot_len;
  ip_addr_t current_iphdr_src;
  ip_addr_t current_iphdr_dest;
};
```

---

### 1.6 lwIP 配置选项差异 (与标准lwIP对比)

**文件**: `liblwip/default_opts/lwipopts.h`

| 选项 | SafeOS值 | 标准lwIP默认值 |
|------|----------|----------------|
| `NO_SYS` | 0 | 1 |
| `LWIP_TCPIP_CORE_LOCKING` | 1 | 0 |
| `LWIP_SOCKET` | 1 | 可选 |
| `LWIP_NETCONN` | 1 | 可选 |
| `MEM_SIZE` | `0x4000*64*4` (~4MB) | ~16KB |
| `MEMP_NUM_NETCONN` | 1024 | 4 |
| `TCP_SND_BUF` | `32*TCP_MSS` (46KB) | 8KB |
| `TCP_WND` | `96*TCP_MSS` (139KB) | 16KB |
| `LWIP_WND_SCALE` | 1 | 0 |
| `TCP_RCV_SCALE` | 10 | 0 |
| `LWIP_TCP_SACK_OUT` | 1 | 0 |
| `LWIP_SOCKET_NPOLL` | 1 | 0 |
| `LWIP_CACHE_MUTEX_SEM_CPTRS` | 1 | 0 |
| `IP_FORWARD` | 1 | 0 |
| `ETHARP_SUPPORT_STATIC_ENTRIES` | 1 | 可选 |
| `PBUF_POOL_SIZE` | 64 | 16 |
| `TCP_LISTEN_BACKLOG` | 1 | 0 |
| `ARP_TABLE_SIZE` | 100 | 10 |
| `TCP_MAXRTX` | 6 | 12 |
| `TCP_SYNMAXRTX` | 4 | 6 |
| `LWIP_TCP_KEEPALIVE` | (默认启用) | 0 |
| `TCP_KEEPIDLE_DEFAULT` | 10000ms | 2小时 |
| `ETH_PAD_SIZE` | 0 | 2 |
| `IPV6` | 0 | 可选 |

---

## 2. lwIP NIO 封装层 (liblwip)

### 2.1 目录结构

```
liblwip/
├── include/
│   ├── arch/
│   │   ├── cc.h           # 编译器/平台抽象 (typedefs, 字节序)
│   │   ├── sys_arch.h    # seL4 sys层 (信号量/互斥/邮箱)
│   │   ├── fast_select.h # fast_select API
│   │   └── npoll.h       # npoll (epoll风格) API
│   └── nio/
│       ├── lwip_interface.h   # hook接口
│       ├── nio_inet.h         # inet_*(BSD风格)
│       └── nio_sockets.h       # socket API扩展
├── src/
│   ├── lwip_interface.c   # hook实现 (memp_monitor)
│   ├── sys_arch_sel4.c   # seL4 sys_arch 实现
│   ├── npoll.c            # npoll 实现
│   └── fast_select.c      # fast_select 实现
├── default_opts/          # 平台默认配置
└── m57_opts/             # M57芯片配置
```

### 2.2 sys_arch_sel4.c — seL4 系统抽象层

**文件**: `liblwip/src/sys_arch_sel4.c` (509行)

**核心数据结构**:

```c
struct sys_mbox {
    u32_t head, tail;
    u16_t size, caps;
    sys_mutex_t mutex;
    sys_sem_t empty_slots;    // 信号量控制空槽
    int valid;
    sel4_cptr ntfn;          // seL4通知端点 (合并: 超时+唤醒+定时)
    sel4_cptr read_ntfn;     // 读取通知 (badge=WAKEUP)
    sel4_cptr timer_ntfn;    // 定时通知 (badge=TIMEOUT)
    void **msgs;             // 消息指针数组
};
```

**关键设计**:

1. **seL4 IPC 邮箱**: 使用 `seL4_NBSend` (非阻塞发送) + `seL4_Recv` 接收通知端点
2. **通知徽章机制** (`BADGE_WAKEUP`, `BADGE_TIMEOUT`) 区分唤醒源
3. **cptr缓存**: `LWIP_CACHE_MUTEX_SEM_CPTRS` 复用 seL4 端点/通知cap，避免每次分配开销
4. **互斥锁**: `sys_mutex_t` 基于 `sync_mutex_t` (seL4)
5. **TCPIP线程追踪**: `lwip_tcpip_thread_id`, `tcpip_thread_cnt`

**关键函数**:

| 函数 | 作用 |
|------|------|
| `sys_mbox_new()` | 创建邮箱 (含seL4 ntfn) |
| `lwip_hook_tcpip_mbox_new()` | 创建带seL4通知cap的tcpip专用邮箱 |
| `sys_mbox_trypost()` | 非阻塞放入消息，必要时发seL4_NBSend |
| `sys_mbox_fetch()` / `sys_arch_mbox_tryfetch()` | 获取消息，支持超时 |
| `sys_mutex_new()` / `sys_mutex_lock()` / `sys_mutex_unlock()` | seL4互斥锁 |
| `sys_sem_new()` | seL4信号量 |
| `sys_thread_new()` | 创建lwIP线程 |
| `sys_mark_tcpip_thread()` | 标记当前为tcpip线程 |
| `sys_lock_tcpip_core()` / `sys_unlock_tcpip_core()` | 核心锁 (LWIP_TCPIP_CORE_LOCKING) |
| `sys_check_core_locking()` | 断言核心锁持有者 |

**cptr缓存** (可选, `LWIP_CACHE_MUTEX_SEM_CPTRS`):
```c
// 复用 seL4 endpoint (sem) 和 notification (bin_sem)
// deque管理空闲cptr队列，避免频繁 sys_kobj_alloc
sync_sem_new_cached() / sync_sem_destroy_cached()
sync_bin_sem_new_cached() / sync_bin_sem_destroy_cached()
sync_mutex_new_cached() / sync_mutex_destroy_cached()
```

### 2.3 npoll.c — seL4 epoll风格多路复用

**文件**: `liblwip/src/npoll.c` (354行)

**数据结构**:

```c
// npoll callback对象 (挂载在socket fd上)
struct lwip_socket_npoll_node {
    struct lwip_socket_npoll_node *next, *prev;
    struct lwip_npoll_cb *npoll_cb;
};

// npoll control block (每个epoll fd一个)
struct lwip_npoll_cb {
    void *npoll_data;              // npoll mssn
    npoll_wakeup_fn_t npoll_wakeup_fn;
    npoll_update_fn_t npoll_update_fn;
    unsigned long ntfn;            // seL4通知端点
    int sem_signalled;            // 是否已发信号
};

// 全局fd→node映射 (按socket fd索引)
static struct lwip_socket_npoll_node *socket_npoll_list[NUM_SOCKETS];
```

**API**:

| 函数 | 作用 |
|------|------|
| `sel4_lwip_npoll_init()` | 初始化npoll scb |
| `sel4_lwip_npoll_commit()` | 提交epoll fd注册 |
| `sel4_lwip_npoll_decommit()` | 注销epoll fd |
| `sel4_lwip_npoll_del()` | 删除scb |
| `sel4_lwip_npollscb_activate()` / `deactivate()` | 启用/禁用sem信号 |
| `sel4_lwip_npollscan()` | 扫描socket事件 |
| `npoll_check_waiters()` | 检查并唤醒等待者 (tx/rx/err) |

**npoll扫描逻辑** (`sel4_lwip_npollscan`):
```c
// 遍历pollfd数组
// 对每个socket: 获取sock->lastdata/rcvevent/sendevent/errevent
// 按watch_evt (NPOLL_IN/NPOLL_OUT/NPOLL_ERR) 检查就绪状态
// 支持 LWIP_NPOLLSCAN_INC_WAIT/DEC_WAIT 增减select_waiting计数
```

**与seL4集成**: 通过 `seL4_Signal(ntfn)` 通知等待任务有I/O事件。

### 2.4 fast_select.c — 快速选择机制

**文件**: `liblwip/src/fast_select.c`

**数据结构**:
```c
struct lwip_sock_fselect {
    fselect_mbox_t *mb;
    void *user_data;
    int wait_evts;              // 等待的事件类型
    s8_t in_fifo_recvs, in_fifo_sends, in_fifo_errs; // FIFO中的事件计数
};

static struct lwip_sock_fselect sock_fselects[NUM_SOCKETS];
```

**回调**: `fselect_event_callback()` — socket事件触发时调用

### 2.5 NIO Socket API扩展

**文件**: `liblwip/include/nio/nio_sockets.h`

- 自定义 `FD_SET` / `FD_CLR` / `FD_ISSET` / `FD_ZERO` (与 `MEMP_NUM_NETCONN` 联动)
- TCP选项扩展: `TCP_NODELAY`, `TCP_MAXSEG`, `TCP_KEEPIDLE`, `TCP_KEEPINTVL`, `TCP_KEEPCNT`, `TCP_NO_QUICKACK`
- `struct pollfd` 和 `POLL*` 常量 (用于epoll风格)
- ioctl: `FIONREAD`, `FIONBIO`

### 2.6 lwip_interface.c — Hook和监控

**文件**: `liblwip/src/lwip_interface.c`

- `memp_monitor()`: 内存池水位监控 (7/8为警戒, 满为枯竭)
- `LWIP_HOOK_IP4_ROUTE`: 自定义路由hook
- `LWIP_HOOK_TCPIP_MBOX_NEW`: 自定义tcpip邮箱创建hook

---

## 3. LWFW 防火墙引擎 (liblwfw)

### 3.1 目录结构

```
liblwfw/
├── include/
│   ├── lwfw.h              # 核心API (lwfw_policy_t, lwfw_firewall_t, lwfw_rule_t)
│   ├── lwfw_common.h      # 通用定义 (事件/规则配置/FIFO)
│   ├── lwfw_external.h     # 外部接口 (日志/状态/IOCTL)
│   ├── lwfw_parser.h      # YAML解析器状态机
│   ├── lwfw_notif.h       # 通知线程
│   ├── lwfw_stats.h       # 统计
│   ├── param.h            # 树搜索维度参数 (HS_DIM=5, HS_DIM6=6)
│   ├── tree_rule.h        # 规则结构
│   ├── tree_hs.h          # Hyperscan树结构
│   ├── tree_hs_api.h      # 树API
│   ├── mem.h              # 内存管理
│   └── lwct/              # 连接跟踪模块
│       ├── lwct.h         # LWCT连接结构
│       ├── lwct_core.h    # LWCT核心API
│       ├── lwct_tuple.h   # 5元组
│       ├── lwct_common.h  # 公共定义
│       ├── lwct_hash.h    # 哈希
│       ├── lwct_proto.h   # 协议处理
│       └── lwct_proto_tcp/udp/icmp.c
└── src/
    ├── lwfw.c             # 主防火墙逻辑 (2307行)
    ├── lwfw_parser.c      # YAML解析
    ├── lwfw_notif.c       # 通知线程
    ├── tree_hs.c          # 树搜索 (861行)
    ├── tree_rule.c        # 规则管理
    ├── tree_mem.c         # 树内存/堆
    ├── tree_utils.c       # 工具函数
    ├── lwct/lwct_core.c   # LWCT核心 (1331行)
    └── lwct/lwct_proto_tcp/udp/icmp.c
```

### 3.2 防火墙规则结构

**文件**: `liblwfw/include/lwfw.h`

```c
// 规则: L2匹配 (MAC, VLAN, EtherType)
typedef struct lwfw_rule_l2_info {
    lwfw_mac_t src_mac;          // addr[6] + mask[6]
    lwfw_mac_t dst_mac;
    uint16_t ether_type;
    uint16_t vlan;
} lwfw_rule_l2_info_t;

// 规则: L3匹配 (IP, 协议)
typedef struct lwfw_rule_l3_info {
    lwfw_ip_addr_t src_ip;      // addr + masklen/mask
    lwfw_ip_addr_t dst_ip;
    uint8_t proto;              // 0xff=任意
} lwfw_rule_l3_info_t;

// 规则: L4匹配 (端口范围/列表)
typedef struct lwfw_rule_l4_info {
    lwfw_port_info_t src_ports;  // port_range[2] 或 port_list[4]
    lwfw_port_info_t dst_ports;
} lwfw_rule_l4_info_t;

// 完整防火墙规则
typedef struct __attribute__((aligned(CACHE_ALIGNMENT))) lwfw_rule {
    struct cdlist next;          // 链表节点
    uint16_t index;
    uint16_t priority;
    uint16_t state;              // 启用/禁用
    uint16_t ct_state;           // [扩展] 连接跟踪状态
    uint32_t flags;              // LWFW_RULE_FLAGS_*
    char rule_name[MAX_RULE_NAME_LEN];
    lwfw_netif_t interface;       // 接口名
    lwfw_rule_l2_info_t l2;
    lwfw_rule_l3_info_t l3;
    lwfw_rule_l4_info_t l4;
    lwfw_action_t action;         // DENY|EVENT|LOGGING
    rate_limit_t rlimit;          // 限速
    uint32_t hit_cnt;             // 命中计数
} lwfw_rule_t;
```

**规则标志** (`lwfw_rule_flag_t`):
- `LWFW_RULE_FLAGS_NETIF` — 接口名匹配
- `LWFW_RULE_FLAGS_SRC_MAC` / `DST_MAC` — MAC地址
- `LWFW_RULE_FLAGS_VLAN` — VLAN ID
- `LWFW_RULE_FLAGS_ETHER_TYPE` — 以太网类型
- `LWFW_RULE_FLAGS_PROTOCOL` — L3协议
- `LWFW_RULE_FLAGS_SRC_IP_MASK` / `*_MASK_LEN` — 源IP
- `LWFW_RULE_FLAGS_DST_IP_MASK` / `*_MASK_LEN` — 目的IP
- `LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE` / `LIST` — 源端口
- `LWFW_RULE_FLAGS_DST_L4_PORT_RANGE` / `LIST` — 目的端口
- `LWFW_RULE_FLAGS_CT_STATE` — 连接跟踪状态
- `LWFW_RULE_FLAGS_RATE_LIMIT` — 限速

### 3.3 规则表结构

```c
typedef struct __attribute__((aligned(CACHE_ALIGNMENT))) lwfw_rule_table {
    uint16_t rule_cnt;
    uint16_t state;
    lwfw_action_t def_action;     // 默认动作
    uint32_t def_hit_cnt;
    struct cdlist header;          // 链表头
    rule_set_t _ruleset;          // 线性规则集
    hs_tree_t _hs_tree;           // Hyperscan搜索树
} lwfw_rule_table_t;

// 两条过滤链
typedef enum lwfw_table_flag {
    LWFW_IN_TABLE = 0,            // 入口 (RX)
    LWFW_OUT_TABLE,               // 出口 (TX)
    LWFW_MAX_COUNT_TABLE,
} lwfw_table_flag_t;
```

### 3.4 策略结构

```c
typedef struct __attribute__((aligned(CACHE_ALIGNMENT))) lwfw_policy {
    char policy_name[MAX_POLICY_NAME_LEN];
    uint32_t version, revision;
    struct {
        uint32_t event_queue_size;
        uint32_t event_notify_interval;
        uint32_t logs_per_second;
        uint32_t tree_bucket_size;    // 树参数
        uint32_t tree_node_num;
        uint32_t ct_oot_action;       // 连接跟踪超出动作
        // ...
    } params;
    const lwfw_backend_engine_t *filter_engine;
    lwfw_rule_table_t rule_tables[LWFW_MAX_COUNT_TABLE];
    const int32_t memp_type;            // MEMP_LWFW_RULE or MEMP_LWFW_RULE_SWAP
} lwfw_policy_t;
```

**防火墙全局控制**:
```c
typedef struct lwfw_firewall {
    bool is_notify_thread_ready;
    lwfw_ctrl_t ctrl;                  // 版本/状态/规则数/限速参数
    sync_mutex_t policy_lock;          // 策略锁
    lwfw_policy_t *policy;             // 当前策略
    lwfw_policy_t *inactive_policy;    // 热备策略 (原子切换)
    const lwfw_firewall_ops_t *ops;
} lwfw_firewall_t;
```

### 3.5 树搜索算法 (Hyperscan风格)

**文件**: `liblwfw/src/tree_hs.c`

**维度定义** (`param.h`):
- `HS_DIM = 5` — IPv4: SIP, DIP, SPORT, DPORT, PROTOCOL
- `HS_DIM6 = 6` — IPv6增加

**树节点**:
```c
typedef struct hs_node_s {
    unsigned char d2s;          // 分割维度 (2bit足够)
    unsigned char depth;         // 树深度
    unsigned int thresh;         // 分割阈值
    unsigned int child_idx;       // 子节点索引
    rule_set_t ruleset;         // 本节点规则集
} hs_node_t;

typedef struct hs_tree_s {
    hs_build_aux_t *aux;         // 辅助结构 (ranges + heap)
    struct hs_node_vec node_vec; // 节点数组
    hs_node_t *root;
    tree_info_t tree_info;       // 统计信息
    rule_set_t ruleset;
    tree_param_t params;
} hs_tree_t;
```

**搜索过程**:
1. **规则构造**: 5个维度每条规则对应一个 `[low, high]` 区间
2. **分割维度选择(d2s)**: 在包含最多未分割规则的维度上分割
3. **阈值计算**: 对该维度所有规则的区间端点排序，取中位数
4. **节点递归**: 小于阈值的规则进入左子树，大于等于进入右子树
5. **叶子节点**: 规则数 ≤ bucketSize 时停止分裂，使用线性搜索

**线性搜索** (`linear_search`):
```c
// 遍历ruleset中所有规则
// 对每个规则的HS_DIM个维度，检查key是否在[range[0], range[1]]内
// 支持可选的接口名匹配 (TREE_SUPPORT_NETIF)
// 返回最高优先级(最小数值)的匹配规则
static inline int linear_search(rule_set_t *ruleset, hs_key4_t *key);
```

**命中计数**: 规则结构中的 `hit_cnt` 字段累计

### 3.6 连接跟踪 (LWCT)

**文件**: `liblwfw/src/lwct/`

**连接结构** (`lwct.h`):
```c
struct lwct_conn {
    int32_t refcnt;                    // 引用计数
    uint64_t timeout;                  // 过期时间戳 (ns)
    struct lwct_tuple_hash tuplehash[LWCT_DIR_MAX]; // 双向哈希 (原方向+回复方向)
    uint64_t status;                   // 状态位
    union {
        struct lwct_ext *ext;          // 扩展数据
        uint32_t stats[LWCT_DIR_MAX]; // 统计计数
    };
} ALIGN(8);                          // 8字节对齐
```

**连接状态位** (`lwct_common.h`):
- `LWCT_CONFIRMED_BIT` — 已确认 (加入哈希表)
- `LWCT_SEEN_REPLY_BIT` — 收到过回复
- `LWCT_ASSURED_BIT` — 确定连接
- `LWCT_DYING_BIT` — 正在删除

**五元组** (`lwct_tuple.h`):
```c
struct lwct_tuple {
    struct lwct_info src;         // 源IP + 端口/ICMP ID
    struct lwct_info dst;         // 目的IP + 端口/ICMP type,code
    uint8_t protonum;            // IP_PROTO_TCP/UDP/ICMP
    lwct_dir_t dir;              // LWCT_DIR_ORIGINAL / REPLY
};

struct lwct_tuple_hash {
    struct cdlist node;           // 链表节点 (挂载到桶)
    struct lwct_tuple tuple;
    uint32_t hash;               // 哈希值
};
```

**TCP状态机** (`lwct_proto_tcp.c`):
- `TCP_CT_UNREPLIED` — 未收到回复 (3分钟超时)
- `TCP_CT_REPLIED` — 已收到回复 (3分钟超时)
- `TCP_CT_ESTABLISHED` — 已确认连接 (3小时超时)
- 收到 RST → `conn->timeout = 0` (立即删除)

**哈希表结构**:
```c
struct lwct_conn_table {
    struct cdlist *conn_lists;   // 桶数组 (每桶一个cdlist)
    sys_mutex_t *bkt_locks;     // 每桶一把锁 (减少锁竞争)
};
```

**超时宏**:
```c
// TCP超时 (可配置)
tcp_timeouts[TCP_CT_UNREPLIED]  = 3 * 60 * NS_IN_MINUTE;
tcp_timeouts[TCP_CT_REPLIED]    = 3 * 60 * NS_IN_MINUTE;
tcp_timeouts[TCP_CT_ESTABLISHED]= 3 * 60 * NS_IN_MINUTE;

// UDP/ICMP超时 (通过lwct_parameters配置)
```

**GC线程** (`gc_thread_fn`):
- 周期性扫描所有桶
- 删除过期连接 (`lwct_should_gc()`)
- 支持早收 (`lwct_should_early_gc()`) — TCP/UDP超时前一定时间提前删除

**与pbuf绑定**:
- `pbuf->_lwct` 字段存储 `(uint64_t)conn | state`
- lwct_state: NEW/ESTABLISHED/RELATED/INVALID

### 3.7 Hook注入机制

**文件**: `liblwfw/src/lwfw.c`

**主过滤函数** (第300+行):

```c
// 入口过滤 (RX方向)
int lwfw_ingress_filter(const struct pbuf *p, const struct netif *inp);

// 出口过滤 (TX方向)
int lwfw_egress_filter(const struct pbuf *p, const struct netif *inp);
```

**过滤流程** (`lwfw.c`):
1. 解析数据包 L2/L3/L4 字段 → `lwfw_pkt_*_info_constructor()`
2. 调用 `lwct_in()` 进行连接跟踪 → `lwct_conn_get_from_pbuf()`
3. 调用树搜索 → `hs_linear_search_entry()` → `linear_search_entry()`
4. 匹配规则 → 执行动作 (DENY/EVENT/LOGGING)
5. 限速检查 → `rate_limit_t`

**通知线程** (`lwfw_notif.c`):
- 独立线程 `lwfw_notify_thread`
- 事件FIFO (`lwfw_event_fifo_t`) — 生产者(过滤引擎)/消费者(写日志)
- 支持节流 (`logs_per_second`) 和事件合并

### 3.8 VLAN隔离实现

**方式**: 通过 `ETHARP_SUPPORT_VLAN` 在 `netif->vlanid` 和规则中的 `lwfw_rule_l2_info_t.vlan` 匹配实现。

**VLAN解析** (`lwfw_pkt_l2_info_constructor`):
```c
if (eth_hdr->type == PP_HTONS(ETHTYPE_VLAN)) {
    eth_vlan_hdr = (struct eth_vlan_hdr *)((void*)eth_hdr + ETHER_HDR_LEN);
    l2->vlan = VLAN_ID(eth_vlan_hdr);
} else {
    l2->vlan = 0;
}
```

### 3.9 防火墙操作码

```c
typedef enum lwfw_ioctl_opcode {
    LWFW_IOCTL_OPCODE_RESET_FW = 0,
    LWFW_IOCTL_OPCODE_ENABLE_RULE,
    LWFW_IOCTL_OPCODE_DISABLE_RULE,
    LWFW_IOCTL_OPCODE_INSERT_RULE,
    LWFW_IOCTL_OPCODE_DELETE_RULE,
    LWFW_IOCTL_OPCODE_GET_INFO,
    LWFW_IOCTL_OPCODE_GET_STATUS,
    LWFW_IOCTL_OPCODE_GET_STATISTICS,
    LWFW_IOCTL_OPCODE_MAX,
} lwfw_ioctl_opcode_t;
```

---

## 4. 网络驱动 (dwmac)

### 4.1 目录结构

```
drivers/net/dwmac/
├── include/
│   ├── dwmac.h          # 主头文件
│   ├── dwmac_cfg.h      # 配置
│   ├── dwmac_cfg_int.h  # 内部配置
│   ├── dwmac_common.h   # 通用定义
│   ├── dwmac_core.h     # 核心API
│   ├── dwmac_ctrl.h     # 控制API
│   ├── dwmac_desc.h     # DMA描述符
│   ├── dwmac_dma.h      # DMA配置
│   ├── dwmac_regs.h     # 寄存器映射
│   ├── ndev_cfg.h / ndev.h  # 网络设备抽象
│   └── phy/             # PHY驱动 (RTL8211F, RTL9000BX)
└── src/
    ├── dwmac.c          # 主驱动 (入口/初始化)
    ├── dwmac_core.c     # 核心MAC操作
    ├── dwmac_ctrl.c     # 控制接口
    ├── dwmac_desc.c     # DMA描述符管理
    ├── dwmac_dma.c      # DMA配置
    ├── dwmac_mdio.c     # MDIO/PHY管理
    ├── dwmac_mem.c      # 内存管理
    ├── dwmac_mmc.c      # MMC计数器
    ├── dwmac_phy.c      # PHY配置
    ├── ndev_nsv.c       # NSv集成 (收发路径)
    └── ndev_hyp_bridge.c # 虚拟机桥接
```

### 4.2 驱动模型

**基础架构**: 统一驱动框架 (`struct drv`)

```c
struct drv {
    const char *name;
    size_t dev_size;
    const struct drv_ops {
        int (*drv_init)(struct drv*);
        int (*drv_deinit)(struct drv*);
        int (*dev_init)(struct dev*);
        int (*dev_deinit)(struct dev*);
        sel4_msg_info_t (*evt_hdlr)(sel4_word badge, sel4_msg_info_t info, struct sel4_svc_ctx *ctx);
        int (*irq_hdlr)(struct dev*, sel4_word badge);
    } drv_ops;
    const struct dev_ops *dev_ops;
    uint32_t options;  // DRV_OPTION_IRQ_AFFI | DRV_OPTION_PM_* (电源管理)
};
```

**dwmac驱动注册**:
```c
struct drv dwmac_drv = {
    .name = "axera,dwmac-5.40a",
    .drv_ops = {
        .drv_init = dwmac_drv_init,
        .drv_deinit = dwmac_drv_deinit,
        .dev_init = dwmac_dev_init,
        .evt_hdlr = dwmac_evt_hdlr,
        .irq_hdlr = dwmac_irq_hdlr
    }
};
```

### 4.3 网络设备抽象 (ndev)

**文件**: `drivers/net/dwmac/include/ndev.h`

```c
struct ndev {
    char svc_name[16];
    const struct ndev_ops *ops;
    struct ds_ring_user *ds_ring_info;  // 共享内存信息 (NSv)
    struct elem_ring *empty_rx_buf_ring; // 空RX缓冲区环
    struct elem_ring *used_rx_buf_ring;  // 已用RX缓冲区环
    struct elem_ring *used_tx_buf_ring;  // 已用TX缓冲区环
    struct elem_ring *pending_tx_buf_ring;// 待发送TX缓冲区环
    struct ndev_stats stats;
    int empty_rx_buf_size;
    tid_t rx_softirq_thread;
    sync_sem_t rx_softirq_sem;
    sel4_cptr rx_ntfn;
    tid_t tx_thread;
    sel4_cptr tx_ntfn;
    sel4_cptr server_ep;
    sel4_cptr endpoint;
    int cid;                             // 客户ID
};

struct ndev_ops {
    int (*init)(struct ndev*);
    int (*ioctl)(struct ndev*, uint64_t, void*, size_t);
    int (*rx)(struct ndev*, int limit, nbuf *buf);
    int (*tx)(struct ndev*, nbuf buf);
};
```

### 4.4 NSv集成 (共享内存 + IPC)

**文件**: `drivers/net/dwmac/src/ndev_nsv.c`

**共享内存环**:
- `NDEV_RX_RING_SIZE = 4096` — RX环大小
- `NDEV_TX_RING_SIZE = 2048` — TX环大小

**地址翻译**:
```c
// NSv VA ↔ NDEV VA ↔ PA 三角翻译
translate_va_to_pa()      // NDEV VA → PA
translate_pa_to_va()      // PA → NDEV VA
translate_nsv_va_to_ndev_va()  // NSv VA → NDEV VA
translate_ndev_va_to_nsv_va()  // NDEV VA → NSv VA
```

**环操作**:
- `ndev_add_empty_rx_bufs()` — 注册空RX缓冲区环
- `ndev_add_used_rx_bufs()` — 注册已用RX缓冲区环
- `ndev_add_pending_tx_bufs()` — 注册待发送TX缓冲区环
- `ndev_add_used_tx_bufs()` — 注册已用TX缓冲区环

**收发线程**:
- `ndev_rx_thread()` — 从共享内存RX环取包 → lwIP `tcpip_input()`
- `ndev_tx_thread()` — 从lwIP取包 → 共享内存TX环 → DMA发送

### 4.5 DMA描述符

**文件**: `drivers/net/dwmac/src/dwmac_desc.c`

DMA描述符管理 (环形缓冲区)，每个描述符包含:
- 缓冲区地址 (PA)
- 控制字 (长度、TBS位、OWN位)
- 链接下一个描述符

### 4.6 PHY驱动

**目录**: `drivers/net/dwmac/src/phy/`
- `dwmac_phy_rtl8211f.c` — Realtek RTL8211F 千兆 PHY
- `dwmac_phy_rtl9000bx.c` — Realtek RTL9000BX 2.5G PHY

### 4.7 收发包路径

**RX (硬件 → lwIP)**:
1. DMA接收完成中断 → `dwmac_irq_hdlr()`
2. `ndev_rx_thread()` 从RX环取pbuf
3. 通过NSv共享内存传递pbuf给lwIP VM
4. 调用 `tcpip_input()` / `netif->input()` 注入lwIP栈
5. **可选**: LWFW `lwfw_ingress_filter()` 在 `netif->input()` 之前过滤

**TX (lwIP → 硬件)**:
1. lwIP调用 `netif->linkoutput()` → `tcpip_output()`
2. `ndev_tx_thread()` 从lwIP获取pbuf
3. 通过NSv共享内存传递pbuf
4. DMA发送描述符填充
5. 硬件发送完成

---

## 5. seL4 集成

### 5.1 IPC 通道

**VM ↔ NSv 通信**:

1. **seL4通知端点 (Notification)** — 用于异步事件通知
   - `sel4_cptr ntfn` — 合并的通知端点 (多个badge)
   - `BADGE_WAKEUP` / `BADGE_TIMEOUT` — 邮箱线程唤醒/超时
   - `seL4_NBSend()` — 非阻塞发送通知
   - `seL4_Recv()` — 接收通知

2. **seL4 IPC调用** — 用于请求/响应
   - `sel4_svc_run()` — 服务循环
   - `sel4_msg_info_t` — 消息元数据 (长度/标签)
   - `sel4_get_mr()` — 获取消息寄存器

3. **邮箱结构** (`sys_mbox_t`):
   - `mb->ntfn` — 合并通知端点
   - `mb->read_ntfn` — 读取通知 (badge=WAKEUP)
   - `mb->timer_ntfn` — 定时通知 (badge=TIMEOUT)

### 5.2 共享内存管理 (CMA/DS)

**文件**: `drivers/net/dwmac/src/ndev_nsv.c`

```c
struct ds_ring_user {
    dspace_t dsi;       // dspace句柄 (pid/ds/va/size/attr)
    // ... 映射后得到虚拟地址
};

struct ds_ring {
    uint32_t idx;       // 共享环索引
    paddr_t start_pa;   // 物理地址起始
    vaddr_t start_va;   // 虚拟地址起始
    // ...
};
```

**映射流程**:
1. NSv发送 `dspace_t` (capability) 给VM
2. VM调用 `sys_dspace_map()` 映射到虚拟地址
3. 双方使用共享虚拟地址操作环形缓冲区

### 5.3 事件循环机制

**tcpip线程**:
- `tcpip_thread()` 运行在固定优先级 (`TCPIP_THREAD_PRIO=200`)
- 通过 `sys_mbox` 接收来自其他线程的消息
- `sys_arch_mbox_fetch()` 使用 `seL4_Recv(mb->ntfn)` 阻塞等待

**通知线程** (`lwfw_notify_thread`):
- 独立高优先级线程 (200)
- 从 `lwfw_event_fifo` 消费事件日志
- 通过 `sel4_svc_run()` 与NSv通信

**LWCT GC线程** (`lwct_gc_thread`):
- 独立线程 (优先级200)
- 周期性扫描连接跟踪哈希表
- 删除过期连接

### 5.4 cptr缓存机制

**文件**: `liblwip/src/sys_arch_sel4.c`

```c
// 每socket需要: 1个endpoint(sem) + 2个notification(读+定时)
// 1024个socket → 3072个cap
// 复用而非每次分配，降低seL4 cap管理开销
static struct deque free_sem_cptrs;      // endpoint caps
static struct deque free_mutex_cptrs;   // notification caps

// 分配: 从deque取，若空则 sys_kobj_alloc()
// 释放: 放回deque，若满则 sys_kobj_free()
```

---

## 6. LWFW Agent

**文件**: `os-framework/servers/daemons/lwfw_agent/`

### 6.1 目录结构

```
lwfw_agent/
├── include/
│   ├── cjson.h          # JSON解析 (日志格式)
│   └── event_handler.h  # 事件处理
└── src/
    ├── main.c           # 入口: sel4_svc_run()
    ├── event_handler.c  # 事件消费/写文件
    └── cjson.c          # JSON实现
```

### 6.2 事件处理

**事件消费**:
```c
static sel4_msg_info_t lwfw_agent_evt_consume(sel4_msg_info_t info, pid_t pid, struct sel4_svc_ctx *ctx);
// 从 lwfw_event_fifo 读取事件
// 写日志到 /var/log/lwfw/lwfw-event_<timestamp>.log
// 支持文件轮转 (按大小/时间)
```

**日志格式** (JSON):
```json
{
  "timestamp": 1733470543961376,
  "rule_version": "3.0",
  "domain_type": 0,
  "event_source": 10,
  "event_type": 1,
  "rule_id": 2400,
  "score": 0,
  "event_info": {
    "event_id": 3,
    "count": 10,
    "action": "deny",
    "net_if": "PFEVLAN.1",
    "protocol": "TCP",
    "src_ip": "172.20.1.123",
    "dest_ip": "172.20.1.1",
    "src_port": 12345,
    "dest_port": 13400
  }
}
```

### 6.3 配置文件

- 默认路径: `/etc/lwfw/vdf_firewall_policy.yaml`
- 通过 `lwfw_config_reset_manifest()` 热重载

---

## 7. YAML规则解析器

**文件**: `liblwfw/src/lwfw_parser.c`

**解析状态机** (约30+个状态):

```
STATE_START → STATE_STREAM → STATE_DOCUMENT → STATE_SPEC
  → STATE_SPEC_INGRESS / STATE_SPEC_EGRESS
  → STATE_SPEC_XGRESS_RULES → STATE_SPEC_XGRESS_RULE_ENTRY
  → STATE_SPEC_XGRESS_RULE_ITEM_FROM/TO
  → STATE_SPEC_XGRESS_RULE_ITEM_FROMTO_L2/L3/L4
```

**解析流程**:
1. 读取YAML文件 (`CONF_FILE` 或 CPIO打包路径)
2. 填充 `lwfw_policy_config_t`
3. 构建 `lwfw_policy_t` 和 `lwfw_rule_t` 规则链表
4. 构建Hyperscan树 (用于加速匹配)

---

## 8. 关键数据流总览

### 8.1 包接收流程

```
[硬件 DMA]
    ↓
[dwmac驱动 - ndev_rx_thread]
    ↓ (共享内存 NSv)
[lwIP tcpip_input()]
    ↓
[LWFW lwfw_ingress_filter()] ← 五元组匹配 + 规则过滤
    ↓
[lwct_in(pbuf)] ← 连接跟踪 + 状态绑定到pbuf->_lwct
    ↓
[etharp_input() / ip4_input()]
    ↓
[udp_input() / tcp_input()]
```

### 8.2 包发送流程

```
[应用 send()/write()]
    ↓
[lwIP tcp_write() / udp_send()]
    ↓
[LWFW lwfw_egress_filter()] ← 可选出口过滤
    ↓
[ip4_output() / ip6_output()]
    ↓
[netif->linkoutput() → ethernet_output()]
    ↓
[dwmac - ndev_tx_thread]
    ↓ (共享内存 NSv)
[硬件 DMA]
```

### 8.3 策略更新流程

```
[配置文件 YAML]
    ↓
[lwfw_parser.c] → lwfw_policy_config_t
    ↓
[lwfw_agent (用户空间)] → seL4 IPC
    ↓
[lwfw_firewall.ioctl(LWFW_IOCTL_OPCODE_INSERT_RULE)]
    ↓
[构建 lwfw_rule_t → 加入 rule_tables[]._ruleset]
    ↓
[重建 Hyperscan 树 _hs_tree]
    ↓ (原子切换)
[inactive_policy ↔ policy 指针交换]
```

---

## 9. 内存池配置

**lwIP内存池** (`MEMP_*` 类型, 定义在 `lwipopts.h`):

| 池类型 | 数量 (默认) | 说明 |
|--------|------------|------|
| `MEMP_NUM_PBUF` | 64 | pbuf池 |
| `PBUF_POOL_SIZE` | 64 | pbuf池别名 |
| `MEMP_NUM_NETCONN` | 1024 | 网络连接数 |
| `MEMP_NUM_NETBUF` | 16 | netbuf数 |
| `MEMP_NUM_TCP_SEG` | 256 | TCP段 |

**lwfw专用池**:
- `MEMP_LWFW_RULE` — 当前策略规则
- `MEMP_LWFW_RULE_SWAP` — 热备策略规则

---

## 10. 文件索引

### lwIP 核心
| 文件 | 作用 |
|------|------|
| `lwip_ds_mcu/src/include/lwip/netif.h` | netif结构、接口管理API |
| `lwip_ds_mcu/src/include/lwip/pbuf.h` | pbuf结构、pbuf管理API |
| `lwip_ds_mcu/src/include/lwip/tcp.h` | tcp_pcb、TCP API |
| `lwip_ds_mcu/src/include/lwip/udp.h` | udp_pcb、UDP API |
| `lwip_ds_mcu/src/include/lwip/ip.h` | ip_pcb、IP_PCB宏、ip_globals |
| `lwip_ds_mcu/src/include/lwip/tcpip.h` | tcpip_init、tcpip_input、tcpip_inpkt |
| `lwip_ds_mcu/src/include/lwip/arch.h` | 平台抽象、字节序、对齐宏 |

### liblwip
| 文件 | 作用 |
|------|------|
| `liblwip/src/sys_arch_sel4.c` | seL4系统抽象 (邮箱/信号量/互斥/线程) |
| `liblwip/src/npoll.c` | seL4 epoll风格多路复用 |
| `liblwip/src/fast_select.c` | fast_select机制 |
| `liblwip/include/nio/nio_sockets.h` | socket选项、FD_SET扩展 |
| `liblwip/include/nio/nio_inet.h` | inet_addr转换 |
| `liblwip/default_opts/lwipopts.h` | lwIP配置选项 |
| `liblwip/src/lwip_interface.c` | hook实现、内存池监控 |

### liblwfw
| 文件 | 作用 |
|------|------|
| `liblwfw/include/lwfw.h` | 防火墙主结构、lwfw_rule_t、lwfw_policy_t |
| `liblwfw/include/lwfw_common.h` | 通用定义、事件、FIFO、参数 |
| `liblwfw/include/lwfw_external.h` | 外部API、日志级别、lwfw_agent |
| `liblwfw/include/lwfw_parser.h` | YAML解析器状态机 |
| `liblwfw/include/lwfw_notif.h` | 通知线程API |
| `liblwfw/include/tree_rule.h` | rule_base_t/rule_t/rule6_t/rule_set_t |
| `liblwfw/include/tree_hs.h` | hs_node_t/hs_tree_t/hs_key4_t |
| `liblwfw/include/param.h` | HS_DIM=5, HS_DIM6=6 |
| `liblwfw/src/lwfw.c` | 主防火墙逻辑、规则构造、包解析、过滤函数 |
| `liblwfw/src/lwfw_parser.c` | YAML规则解析 |
| `liblwfw/src/tree_hs.c` | Hyperscan树构建和搜索 |
| `liblwfw/src/tree_rule.c` | 规则管理 |
| `liblwfw/src/tree_mem.c` | 树内存/堆实现 |
| `liblwfw/src/lwct/lwct_core.c` | LWCT初始化、连接跟踪、GC |
| `liblwfw/src/lwct/lwct_proto_tcp.c` | TCP状态机 |
| `liblwfw/src/lwct/lwct_proto_udp.c` | UDP处理 |
| `liblwfw/src/lwct/lwct_proto_icmp.c` | ICMP处理 |

### 网络驱动
| 文件 | 作用 |
|------|------|
| `drivers/net/dwmac/include/ndev.h` | ndev网络设备抽象 |
| `drivers/net/dwmac/src/dwmac.c` | 驱动入口、初始化 |
| `drivers/net/dwmac/src/ndev_nsv.c` | NSv共享内存收发路径 |
| `drivers/net/dwmac/src/dwmac_desc.c` | DMA描述符管理 |
| `drivers/net/dwmac/src/dwmac_dma.c` | DMA配置 |

### LWFW Agent
| 文件 | 作用 |
|------|------|
| `lwfw_agent/src/main.c` | seL4服务入口 |
| `lwfw_agent/src/event_handler.c` | 事件消费、日志写入、文件轮转 |

---

## 附录A: 规则维度映射

Hyperscan树使用的5个维度在包字段上的映射:

| 维度ID | 符号 | lwfw中的含义 | 对应包字段 |
|--------|------|-------------|-----------|
| 0 | RULE_KEY_SIP | 源IP | IP src |
| 1 | RULE_KEY_DIP | 目的IP | IP dst |
| 2 | RULE_KEY_SPORT | 源端口 | TCP/UDP src port |
| 3 | RULE_KEY_DPORT | 目的端口 | TCP/UDP dst port |
| 4 | RULE_KEY_PROTOCOL | 协议 | IP protocol |

---

## 附录B: TCP连接状态转移

```
NEW → (TCP SYN) → ESTABLISHED → (TCP FIN) → 
或者 NEW → (TCP SYN) → ESTABLISHED → (TCP RST) → 删除
```

lwct中对TCP的处理:
- TCP SYN (无回复): `TCP_CT_UNREPLIED` (3分钟超时)
- TCP SYN+ACK (收到回复): `TCP_CT_REPLIED` (3分钟超时)
- 连接确认: `LWCT_ASSURED_BIT` 设置 → `TCP_CT_ESTABLISHED` (3小时超时)
- TCP RST: `conn->timeout = 0` → 立即GC

---

## 附录C: 通知线程事件流

```
[防火墙过滤命中] → lwfw_event_push()
    ↓ (FIFO)
[lwfw_notify_thread] → lwfw_notification_thread_loop()
    ↓
[lwfw_agent_evt_consume] → JSON格式化
    ↓
[/var/log/lwfw/lwfw-event_<timestamp>.log]
```

---

*报告生成时间: 2026/05/25*
