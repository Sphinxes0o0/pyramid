# lwIP 在 seL4 上运行的函数级深度分析

> 文档版本: 1.0
> 更新日期: 2026/04/22
> 代码路径: `/home/shiyang/nio/nt35/safeos/`

---

## 1. 整体架构与关键组件

### 1.1 NSv 进程结构

SafeOS 中 lwIP 运行在 **NSv (Network Server)** 用户态进程中：

```
┌─────────────────────────────────────────────────────────────────────┐
│  NSv 进程 (pid = 网络服务器)                                         │
│                                                                      │
│  线程:                                                               │
│  ├─ event_loop()         ← 处理 App 的 BSD socket 请求 (seL4 IPC)  │
│  ├─ nic_rx_thread()      ← 从 NIC 驱动接收数据包                    │
│  ├─ tcpip_thread()       ← lwIP 内部协议处理线程                    │
│  └─ select_thread()      ← 阻塞 socket 的事件等待 (可选)           │
│                                                                      │
│  关键数据结构:                                                        │
│  ├─ vnet_if (struct netif)    ← lwIP 虚拟网卡                      │
│  ├─ CMA (96MB)                ← 与 NIC 共享的连续内存              │
│  └─ elem_ring x4               ← 无锁环形缓冲 (TX/RX 队列)         │
└─────────────────────────────────────────────────────────────────────┘
                              ↕ seL4 IPC + CMA 共享内存
┌─────────────────────────────────────────────────────────────────────┐
│  NIC 驱动进程 (独立用户态进程)                                        │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键文件清单

| 层次 | 文件 | 职责 |
|------|------|------|
| **NSv 入口** | `servers/net/src/main.c` | NSv 初始化、事件循环、socket 请求处理 |
| **seL4 适配** | `libs/util_libs/liblwip/src/sys_arch_sel4.c` | 信号量、互斥锁、mbox、时间接口 |
| **lwIP 核心** | `external/lwip_ds_mcu/src/core/` | TCP/UDP/IP/ARP 实现 |
| **Ethernet 层** | `external/lwip_ds_mcu/src/netif/ethernet.c` | `ethernet_input()`, `ethernet_output()` |
| **RAW Socket** | `external/lwip_ds_mcu/src/core/raw.c` | `raw_afpacket_input()`, `raw_afpacket_output()` |
| **Socket API** | `external/lwip_ds_mcu/src/api/sockets.c` | BSD socket → lwIP 转换 |
| **AF-PACKET** | `servers/net/src/packet_mmap.c` | TPACKET 实现 (`tpacket_recv()`) |
| **防火墙** | `libs/util_libs/liblwfw/src/lwfw.c` | 入方向/出方向过滤 (`ip4_filter_dispatch_*()`) |

---

## 2. 初始化路径

### 2.1 main() → NSv 完全初始化

```
main()                                              [main.c:6400]
  │
  ├─► net_resources_init()                         [main.c:6411]
  │     ├─ netbuf_init()                           // 初始化 netbuf 池
  │     ├─ net_socket_info_init()                   // 初始化 socket 信息表
  │     └─ data_cache_init()                       // 初始化数据缓存
  │
  ├─► init_ds_ring()                               [main.c:6416, 3666]
  │     ├─ sys_mem_map(..., CMA_SIZE, PAGE_DMA)    // 从 CMA 分配 96MB 连续内存
  │     ├─ ds_ring_init()                          // 初始化 ds_ring (含 elem_ring)
  │     └─ sys_dspace_grant() / sys_ds_ring_share() // 共享给 NIC 驱动
  │
  ├─► create_nic_thread()                          [main.c:6421, 5009]
  │     ├─ sys_svc_alloc_endpoint(&nsv_nic_ep)     // 分配 seL4 IPC endpoint
  │     ├─ sys_svc_reg(NSV_NIC_INTERFACE_NAME)     // 注册为 NIC 接口服务
  │     ├─ sys_kobj_alloc(KOBJ_TYPE_NTFN)          // 分配 seL4 notification
  │     └─ sys_thread_create(nic_rx_thread)         // 创建 RX 线程
  │
  ├─► tcpip_init(NULL, NULL)                       [main.c:6426]
  │     └─ 见下方 "lwIP tcpip_init() 内部"
  │
  ├─► netif_add(&vnet_if, &addr, &netmask, &gw, NULL, init_ethif, ethernet_input)
  │     [main.c:6441]                               // 添加虚拟网卡到 lwIP
  │     └─ init_ethif(netif)                       // 见下方 netif 初始化
  │
  ├─► netif_set_link_up() / netif_set_up()        // 启用网卡
  │
  ├─► ethif_update() / dhcp_start()               // 配置 IP (板级相关)
  │
  ├─► dns_server_init()                            // DNS 初始化
  │
  ├─► routing_table_init()                         // 路由表初始化
  │
  ├─► bridge_setup()                              // VirtIO 网桥 (可选)
  │
  ├─► sys_svc_reg("/net", svc_ep)                  [main.c:6581]
  │     // 注册 /net 服务，App 可通过此服务名访问 NSv
  │
  └─► event_loop()                                [main.c:6593]
        // 进入主事件循环，处理所有 App socket 请求
```

### 2.2 tcpip_init() 内部

```
tcpip_init(init_done_fn, init_done_arg)             [tcpip.c:247]
  │
  ├─► sys_mbox_new(&tcpip_mbox, TCPIP_MBOX_SIZE) // 创建 tcpip 线程邮箱
  │     └─ sys_arch_mbox_new() → sys_mbox_new()  // 最终调用 sys_arch_sel4.c
  │
  ├─► sys_thread_new(tcpip_thread, ...)           // 创建 tcpip_thread
  │     └─ tcpip_thread(arg)                      // 开始运行，见下方
  │
  └─► return

tcpip_thread(arg)                                 [tcpip.c:129]
  │
  ├─► LOCK_TCPIP_CORE()                           // 获取核心锁
  │
  ├─► tcpip_init_done(tcpip_init_done_arg)        // 通知初始化完成
  │
  └─► while (1) {                                 // 主循环
  │     TCPIP_MBOX_FETCH(&tcpip_mbox, &msg)       // 等待消息 (阻塞)
  │     tcpip_thread_handle_msg(msg)              // 处理消息
  │   }
  │
  └── tcpip_thread_handle_msg(msg)                [tcpip.c:158]
        ├─ TCPIP_MSG_API      → msg->msg.api_msg.function()     // Socket API 调用
        ├─ TCPIP_MSG_API_CALL → msg->msg.api_call.function()    // 带结果的 API 调用
        ├─ TCPIP_MSG_INPKT    → msg->msg.inp.input_fn()        // 数据包输入 (RX)
        ├─ TCPIP_MSG_CALLBACK → msg->msg.cb.function()         // 回调
        └─ TCPIP_MSG_TIMEOUT  → sys_timeout()                  // 定时器事件
```

### 2.3 netif 初始化 (init_ethif)

```
init_ethif(struct netif *netif)                   [main.c:4710]
  │
  ├─► eth_init(netif->hwaddr)                     // 读取 MAC 地址
  │
  ├─► netif->output = etharp_output               // IPv4 输出函数 (查 ARP 表)
  │
  ├─► #ifdef VIRT_BRG_SUPPORT
  │     netif->linkoutput = ethif_link_output_overload  // 网桥模式
  │   #else
  │     netif->linkoutput = ethif_link_output     // 普通模式 (见 TX 路径)
  │   #endif
  │
  └─► netif->flags = NETIF_FLAG_ETHARP | NETIF_FLAG_BROADCAST |
                     NETIF_FLAG_LINK_UP | NETIF_FLAG_IGMP | NETIF_FLAG_ETHERNET
```

---

## 3. 收包路径 (RX) — 从 NIC 到 App

### 3.1 完整调用链

```
NIC 驱动 (独立进程)
  │
  │ DMA 接收数据包到物理内存 (CMA 区域)
  │
  │ elem_ring_put(used_rx_buf_ring, e)           // 将 buffer 放入已收包队列
  │ sel4_signal(nic_rx_ntfn)                      // 通知 NSv 有数据包
  │
  ▼
nic_rx_thread()                                   [main.c:4961]
  │
  ├─► seL4_Recv(nsv_nic_ep, &badge)             // 等待 NIC notification
  │
  ├─► while (1) {
  │     elem_ring_get(used_rx_buf_ring)          // 获取数据包 buffer 物理地址
  │     if (e.pa == 0) break;                    // 无更多数据包
  │     cma_pa_to_va(&cma, e.pa)                 // 物理地址 → 虚拟地址
  │     LOCK_TCPIP_CORE()
  │     rx_callback((struct pbuf *)va)           // 处理数据包
  │     UNLOCK_TCPIP_CORE()
  │   }
  │
  ▼
rx_callback(void *ctx)                            [main.c:4781]
  │
  └─► vnet_if.input(p, &vnet_if)                 // 调用 netif 的 input 函数
        // 即 ethernet_input(p, &vnet_if)
        │
        ▼
ethernet_input(struct pbuf *p, struct netif *netif)  [ethernet.c:89]
  │
  ├─► p->len 检查 (必须 > SIZEOF_ETH_HDR)
  │
  ├─► ethhdr = (struct eth_hdr *)p->payload     // 解析 Ethernet 头
  │
  ├─► raw_afpacket_input(p, netif, type)        [raw.c:282] ← AF-PACKET 捕获点
  │     // 遍历 raw_afpacket_pcbs，调用每个 PCB 的 recv 回调
  │     // 例如 tpacket_recv() 被调用
  │
  ├─► switch (ethhdr->type) {
  │     case ETHTYPE_IP:   → ip4_input(p, netif)     // IPv4
  │     case ETHTYPE_ARP:  → etharp_input(p, netif)  // ARP
  │     case ETHTYPE_VLAN: → 处理 VLAN tag
  │   }
  │
  ▼
ip4_input(struct pbuf *p, struct netif *inp)     [ip4.c:743]
  │
  ├─► pbuf_remove_header(p, IP_HLEN)            // 去掉 IP 头
  │
  ├─► #ifdef NIO_LWIP_LWFW                       // ===== 防火墙入方向 =====
  │     if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
  │       pbuf_free(p); return ERR_OK;           // 丢弃包
  │     }
  │   #endif
  │
  ├─► switch (IPH_PROTO(iphdr)) {               // 根据协议分发
  │     case IP_PROTO_TCP:   → tcp_input(p, inp)     // TCP
  │     case IP_PROTO_UDP:   → udp_input(p, inp)     // UDP
  │     case IP_PROTO_ICMP: → icmp_input(p, inp)    // ICMP
  │     case IP_PROTO_IGMP: → igmp_input(p, inp)    // IGMP
  │   }
  │
  ▼
tcp_input(struct pbuf *p, struct netif *netif)   [tcp.c]
  │
  ├─► tcp_enqueue(p)                            // 将 pbuf 加入.recv 队列
  │
  └─► tcp_process() / tcp_receive()            // 触发 TCP 状态机
        // 最终通过 API_EVENT() 唤醒等待该 socket 的 App
```

### 3.2 AF-PACKET 捕获路径

```
raw_afpacket_input(p, netif, type)                [raw.c:282]
  │
  ├─► 遍历 raw_afpacket_pcbs 链表
  │
  ├─► 协议匹配检查:
  │     case ETH_P_ALL:  skip = 0              // 捕获所有
  │     case ETH_P_IP:   skip = (type != ETHTYPE_IP)
  │     default:         skip = 1
  │
  ├─► PCB 状态检查 (跳过 INIT 状态的 PCB)
  │
  ├─► 网卡绑定检查 (pcb->netif_idx)
  │
  ├─► lwip_run_socket_filter(conn, p, inp)     [sockets.c:580]
  │     └─ bpf_filter_run(sock->bpf_prog, p)  [bpf_filter.c:548]
  │         └─ cbpf_execute_interpreter()     [bpf_filter.c:68]
  │             // cBPF 解释器执行，返回 1=捕获, 0=跳过
  │
  └─► if (匹配) pcb->recv(pcb->recv_arg, pcb, p, NULL)
        // 调用 tpacket_recv() (对于 PACKET_MMAP socket)

tpacket_recv(void *arg, struct raw_pcb *pcb,
             struct pbuf *p_buf, const ip_addr_t *addr)  [packet_mmap.c:122]
  │
  ├─► rx_buf_ring = packet_mmap_rbuf(packet_node)
  │
  ├─► rb_write_avail(rx_buf_ring) > DEFAULT_TP_FRAME_SIZE ?
  │
  ├─► rb_write_tpacket(rx_buf_ring, p_buf, frame_size)  // 写入 ringbuf
  │
  └─► API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0)           // 唤醒 select/poll
```

---

## 4. 发包路径 (TX) — 从 App 到 NIC

### 4.1 完整调用链

```
App (用户进程)
  │
  │ sys_net_sendto(ep, socket, buf, len, ...)
  │   └─ seL4 IPC 发送请求到 NSv
  │
  ▼
event_loop()                                     [main.c:3400]
  │
  ├─► seL4_Recv(svc_ep, &badge)                 // 等待 App 请求
  │
  └─► switch (label) {
         case SYS_NET_SENDTO: sys_sendto_nb(info, badge)
       }

sys_sendto_nb(info, badge)                       [main.c:1274]
  │
  ├─► sel4_get_mr(0..5)                         // 从 seL4 message registers 获取参数
  │
  ├─► use_shm ? get_shm_va(pid, offset, data_len)
  │           : alloc_data_cache() + sys_unpack_data_from_mrs()
  │   // 获取发送缓冲区
  │
  └─► lwip_sendto(socket, send_buf, data_len, flags, addr, addrlen)
        │
        ▼
lwip_sendto(int s, const void *data, size_t len, ...)  [sockets.c]
  │
  ├─► get_socket(s)                             // 从 fd 找到 lwip_sock
  │
  ├─► netconn = sock->conn
  │
  └─► netconn_sendto(netconn, data, len, addr, addrlen)
        │
        ▼
netconn_sendto(struct netconn *conn, ...)        [api_msg.c]
  │
  ├─► netconn_send(conn, buf, len)              // 最终调用
  │     └─ #ifdef LWIP_NETCONN_SEND_PTON
  │          └─ netconn_send_data(conn, buf, len)  // 走数据路径
  │        #else
  │          └─ tcpip_outpu_tcp() / udp_output() / raw_output()
  │        #endif
  │
  └─► (对于 TCP) tcp_write() + tcp_output()
        (对于 UDP) udp_sendto() → ip4_output_if()
        (对于 RAW) raw_sendto() → ip4_output_if()

// 以下以 UDP 为例:

udp_sendto(struct udp_pcb *pcb, struct pbuf *p,
           const ip_addr_t *dst_ip, u16_t dst_port)   [udp.c]
  │
  └─► ip4_output_if(p, dst_ip, src_ip, dst_port, netif)
        │
        ▼
ip4_output_if(struct pbuf *p, const ip_addr_t *src_ip,
              const ip_addr_t *dst_ip, u16_t proto, struct netif *netif)  [ip4.c:1096]
  │
  ├─► #ifdef NIO_LWIP_LWFW                       // ===== 防火墙出方向 =====
  │     if (lwfw_p->ops->egress_filter(p, netif) != ERR_OK) {
  │       pbuf_free(p); return ERR_FW;           // 丢弃包
  │     }
  │   #endif
  │
  ├─► IP Header 填充: src_ip, dst_ip, proto, len, id, ...
  │
  └─► netif->linkoutput(netif, p)
        // 即 ethif_link_output(netif, p) (VIRT_BRG_SUPPORT 除外)
        │
        ▼
ethif_link_output(struct netif *netif, struct pbuf *q)  [main.c:3788]
  │
  ├─► if (!nic_ready) return ERR_OK              // NIC 未就绪则丢弃
  │
  ├─► LWIP_ASSERT_CORE_LOCKED()                  // 确保在 tcpip core 锁内
  │
  ├─► #ifdef USE_SEND_SMOOTH_QAV                // QoS 流量整形 (可选)
  │     if (q->flags & PBUF_FLAG_SM_Q5/Q6) {
  │       sm_post_element(sm_que5/6_buf_ring, q) // 放入整形队列
  │       return ERR_OK;
  │     }
  │   #endif
  │
  ├─► 分配新 pbuf 并复制数据:
  │     pbuf_init_alloced_pbuf(p, ...)
  │     memcpy(p->payload, q->payload, length)
  │
  ├─► free_complete_tx_packet_pbuf()             // 回收已完成的 TX buffer
  │
  ├─► elem_ring_put(pending_tx_buf_ring, e)     // 将 buffer PA 放入待发队列
  │
  ├─► sel4_signal(nic_tx_ntfn)                   // 通知 NIC 有数据包要发送
  │
  └─► return ERR_OK

// NIC 驱动端 (独立进程):
nic_tx_thread() / 事件处理
  │
  ├─► elem_ring_get(pending_tx_buf_ring)        // 获取待发 buffer PA
  │
  ├─► DMA 发送数据到网卡
  │
  └─► elem_ring_put(used_tx_buf_ring, e)        // 放入已完成队列
        sel4_signal(nic_rx_ntfn)                 // 通知 NSv 发送完成
```

---

## 5. Socket 请求处理路径

### 5.1 App 侧调用链

```
App                                        [用户进程]
  │
  ├─► socket(AF_INET, SOCK_STREAM, 0)         // POSIX socket()
  │     └─ sys_socket(ep, AF_INET, SOCK_STREAM, 0, &handle)
  │         └─ seL4 IPC: seL4_Call(svc_ep, SYS_NET_SOCKET)
  │
  ├─► bind(socket, &addr, addrlen)             // POSIX bind()
  │     └─ sys_bind(ep, handle, &addr, addrlen)
  │         └─ seL4 IPC: seL4_Call(svc_ep, SYS_NET_BIND)
  │
  ├─► listen(socket, backlog)                  // POSIX listen()
  │     └─ sys_listen(ep, handle, backlog)
  │         └─ seL4 IPC: seL4_Call(svc_ep, SYS_NET_LISTEN)
  │
  ├─► accept(socket, &client_addr, &addrlen)  // POSIX accept()
  │     └─ sys_accept(ep, handle, &accept_handle, ...)
  │         └─ seL4 IPC: seL4_Call(svc_ep, SYS_NET_ACCEPT)
  │
  ├─► connect(socket, &addr, addrlen)           // POSIX connect()
  │     └─ sys_connect(ep, handle, flags, &addr, addrlen)
  │         └─ seL4 IPC: seL4_Call(svc_ep, SYS_NET_CONNECT)
  │
  ├─► sendto(socket, buf, len, flags, addr, addrlen)
  │     └─ sys_sendto(ep, handle, buf, len, flags, use_shm, addr, addrlen, &sent)
  │         └─ seL4 IPC: seL4_Call(svc_ep, SYS_NET_SENDTO)
  │
  ├─► recvfrom(socket, buf, len, flags, &addr, &addrlen)
  │     └─ sys_recvfrom(ep, handle, buf, len, flags, use_shm, ...)
  │         └─ seL4 IPC: seL4_Call(svc_ep, SYS_NET_RECVFROM)
  │
  └─► close(socket)                           // POSIX close()
        └─ sys_close_nb(ep, handle)
            └─ seL4 IPC: seL4_Call(svc_ep, SYS_NET_CLOSE)
```

### 5.2 NSv event_loop 侧处理

```
event_loop()                                     [main.c:3400]
  │
  ├─► seL4_Recv(svc_ep, &badge)               // 等待 App 的 seL4 IPC 请求
  │
  └─► switch (label) {
  │     case SYS_NET_SOCKET:    → sys_socket_nb(info, badge)
  │     case SYS_NET_BIND:      → sys_bind_nb(info, badge)
  │     case SYS_NET_LISTEN:    → sys_listen_nb(info, badge)
  │     case SYS_NET_CONNECT:   → sys_connect_nb(info, badge)
  │     case SYS_NET_ACCEPT:    → sys_accept(info, badge)
  │     case SYS_NET_SENDTO:    → sys_sendto_nb(info, badge)
  │     case SYS_NET_RECVFROM:  → sys_recvfrom_nb(info, badge)
  │     case SYS_NET_CLOSE:     → sys_close_nb(info, badge)
  │   }

sys_socket_nb(info, badge)                       [main.c:1000]
  │
  ├─► sel4_get_mr(0..2)                        // 获取 domain, type, protocol
  │
  ├─► is_supported_domain(domain)             // 检查 AF_INET / AF_INET6 / AF_UNIX
  │
  ├─► is_supported_type(type)                 // 检查 SOCK_STREAM / SOCK_DGRAM / SOCK_RAW
  │
  ├─► lwip_socket(domain, type, protocol)     [sockets.c]
  │     ├─ netconn_new_with_proto()           // 创建 netconn
  │     │     └─ sys_mbox_new()              // 创建 tcpip 线程邮箱
  │     │     └─ sys_sem_new()               // 创建信号量
  │     │     └─ alloc_socket()               // 分配 lwip_sock
  │     └─ return socket fd
  │
  ├─► net_socket_info[socket].owner = pid     // 记录 socket 所有者
  │
  └─► netstat_add_info_by_badge(socket, badge)// 记录 badge 信息
        sel4_reply(info)                       // 回复 App
```

### 5.3 lwip_socket → netconn 映射

```
lwip_socket(int domain, int type, int protocol)   [sockets.c]
  │
  ├─► netconn_new_with_proto(domain, type, protocol, &conn)
  │     │
  │     ├─ netconn_alloc()                   // 分配 struct netconn
  │     │     └─ sys_mbox_new()              // 创建 mbox (tcpip 线程通信)
  │     │     └─ sys_sem_new()               // 创建信号量
  │     │
  │     └─ set_pcb_new(conn, domain, type, protocol)
  │           ├─ NETCONN_TCP   → alloc_tcp_pcb()     // 分配 tcp_pcb
  │           ├─ NETCONN_UDP   → alloc_udp_pcb()      // 分配 udp_pcb
  │           ├─ NETCONN_RAW   → raw_new()            // 分配 raw_pcb
  │           └─ NETCONN_PACKET→ packet_new()         // 分配 packet_pcb
  │
  └─► alloc_socket(conn)                      // 分配 socket fd
        └─ sock->conn = conn                  // socket 与 netconn 关联
```

---

## 6. seL4 适配层 (sys_arch_sel4.c)

### 6.1 信号量实现

```
sys_sem_new(struct sys_sem *sem, u8_t count)     [sys_arch_sel4.c]
  │
  ├─► sys_kobj_alloc(KOBJ_TYPE_NTFN, ...)      // 分配 seL4 notification
  │     // seL4 notification用作信号量底层
  │
  └─► sync_sem_init(sem, cptr, count)          // 初始化信号量

sys_sem_wait(struct sys_sem *sem) / sys_sem_signal(struct sys_sem *sem)
  │
  └─► seL4_Pend(sem->ep.cptr)                  // seL4 notification P 操作
      seL4_Signal(sem->ep.cptr)                // seL4 notification V 操作
```

### 6.2 互斥锁实现

```
sys_mutex_new(struct sys_mutex *mutex)          [sys_arch_sel4.c]
  │
  ├─► sys_kobj_alloc(KOBJ_TYPE_MUTEX, ...)    // 分配 seL4 mtx 对象
  │
  └─► sync_mutex_init(mutex, cptr)            // 初始化互斥锁

sys_mutex_lock(struct sys_mutex *mutex) / sys_mutex_unlock(mutex)
  │
  └─► seL4_MutexLock(seL4_CPtr) / seL4_MutexUnlock(seL4_CPtr)
```

### 6.3 邮箱 (mbox) 实现 — tcpip 线程通信

```
sys_mbox_new(struct sys_mbox *mbox, int size)   [sys_arch_sel4.c]
  │
  ├─► sys_mbox = mbox_alloc()                 // 分配 mbox 结构
  │
  └─► 内部使用 mbox 实现 (消息队列)
        // tcpip 线程和 event_loop/socket 线程间通信

sys_arch_mbox_fetch(struct sys_mbox *mbox, void **msg, u32_t timeout)
  │
  └─► 从 mbox 队列中获取消息，超时则返回 SYS_ARCH_TIMEOUT
```

### 6.4 内存分配

```
lwip_malloc(size)                              [sys_arch_sel4.c]
  │
  └─► ds_ring_mem_alloc(ds, size)             // 从 CMA 区域分配
        └─ 从预分配的 CMA 内存中按需分配

lwip_free(ptr)                                 [sys_arch_sel4.c]
  │
  └─► ds_ring_mem_free(ds, ptr)               // 释放到 CMA 区域
```

### 6.5 时间接口

```
sys_now(void)                                   [sys_arch_sel4.c:40]
  │
  └─► return (u32_t)(time_get_ns() / NS_IN_MS)  // 毫秒级时间
```

---

## 7. 防火墙 Hook 精确位置

### 7.1 Ingress (RX) Hook

```
ip4_input(p, inp)                              [ip4.c:743]
  │
  └─► #ifdef NIO_LWIP_LWFW
        if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
          pbuf_free(p); IP_STATS_INC(ip.drop);
          MIB2_STATS_INC(mib2.ipindiscards);
          return ERR_OK;                       // 丢弃
        }
      #endif
        │
        ▼
ip4_filter_dispatch_incoming(p, inp)          [lwfw.c:802]
  │
  └─► ip4_filter(lwfw_p, p, inp, LWFW_IN_TABLE)  [lwfw.c:724]
        │
        ├─► lwfw_pkt_info_constructor(p, inp, &pkt_info, dir)  [lwfw.c:329]
        │     ├─ 从 pbuf 解析 IP 头 → lwfw_pkt_l3_info_constructor()
        │     ├─ 从 pbuf 解析 TCP/UDP 头 → lwfw_pkt_l4_info_constructor()
        │     └─ 填充 lwfw_pkt_info_t (src_ip, dst_ip, proto, ports)
        │
        └─► filter_engine->do_filter(policy, &pkt_info, &ret_rule)
              └─ list_search_do_filter()           [lwfw.c:1884]
                    ├─ 遍历 cdlist 规则链表
                    └─ check_rule()                [lwfw.c:565]
                          ├─ check_lwfw_l2_info()   // EtherType, VLAN, MAC
                          ├─ check_lwfw_l3_info()   // IP, Protocol
                          └─ check_lwfw_l4_info()   // Ports
```

### 7.2 Egress (TX) Hook

```
ip4_output_if(p, src_ip, dst_ip, proto, netif)  [ip4.c:1096]
  │
  └─► #ifdef NIO_LWIP_LWFW
        if (lwfw_p->ops->egress_filter(p, netif) != ERR_OK) {
          MIB2_STATS_INC(mib2.ipoutdiscards);
          IP_STATS_INC(ip.drop);
          return ERR_FW;                       // 丢弃
        }
      #endif
        │
        ▼
ip4_filter_dispatch_outgoing(p, netif)        [lwfw.c:847]
  │
  └─► ip4_filter(lwfw_p, p, netif, LWFW_OUT_TABLE)
        // 调用链同上，区别在 dir = LWFW_OUT_TABLE
```

---

## 8. select/poll 事件机制

### 8.1 App 侧调用

```
select(nfds, &readfds, &writefds, &exceptfds, timeout)
  │
  └─► sys_select(ep, nfds, readfds, writefds, exceptfds, timeout)
        └─ seL4 IPC 到 NSv 的 select_thread
```

### 8.2 NSv 侧处理

```
select_thread()                                 [main.c:select_thread]
  │
  ├─► lwip_select(maxfdp, read_set, write_set, except_set, timeout)
  │     // 内部使用 select_wait() 等待 socket 事件
  │
  ├─► 当 socket 可读/可写时，select 返回
  │
  └─► 通知 App (通过之前建立的 notification)
```

### 8.3 lwip_select 内部

```
lwip_select(int maxfdp, fd_set *readfds, ...)  [sockets.c]
  │
  ├─► for each socket in readfds:
  │     if (sock->rcvevent > 0) set bit in return readfds
  │
  ├─► if (no bits set) {
  │     select_wait(sock, &read_set)           // 阻塞等待
  │   }
  │
  └─► return 准备好的 fd 数量

select_wait(struct lwip_sock *sock, fd_set *set)  [sockets.c]
  │
  └─► sys_arch_sem_wait(&sock->select_waiting_sem, 0)
        // 使用 seL4 notification 等待

API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0)         [api_event.c]
  │
  └─► sock->rcvevent++                         // 增加接收事件计数
      sys_sem_signal(&sock->select_waiting_sem) // 唤醒 select
```

---

## 9. 数据结构关键关系

### 9.1 Socket → Netconn → PCB 映射

```
lwip_sock (sockets_priv.h)
  ├─ socket fd (整数)
  ├─ struct netconn *conn
  ├─ s16_t rcvevent, u16_t sendevent, u16_t errevent
  ├─ SELWAIT_T select_waiting
  ├─ struct bpf_program *bpf_prog            // cBPF 程序
  └─ void *packet_info                       // PACKET_MMAP 元数据

netconn
  ├─ enum netconn_type (NETCONN_TCP/UDP/RAW/PACKET)
  ├─ union { tcp_pcb, udp_pcb, raw_pcb }
  ├─ sys_mbox_t mbox                         // tcpip 线程通信
  └─ sys_sem_t op_completed                  // 操作完成信号

tcp_pcb / udp_pcb / raw_pcb                   // 协议控制块
```

### 9.2 elem_ring — 无锁环形缓冲

```
struct elem {
    uint64_t pa;     // 物理地址
    uint32_t len;    // 长度
};

struct elem_ring {
    struct elem *elems;   // 数组
    int n;                // 元素数量
    int get_idx;          // 读索引
    int put_idx;          // 写索引
};

// 生产者调用:
elem_ring_put(ring, elem)   // 写入，失败返回 -ENOSPC

// 消费者调用:
elem_ring_get(ring)          // 读取，自动推进索引
```

### 9.3 pbuf — lwIP 数据包缓冲

```
struct pbuf {
    struct pbuf *next;      // 链表下一 pbuf
    void *payload;          // 数据指针
    u16_t len;              // 本 pbuf 数据长度
    u16_t tot_len;          // 链表中所有 pbuf 总长度
    u16_t type;             // PBUF_POOL / PBUF_RAM / PBUF_ROM
    u16_t flags;            // 标志 (如 PBUF_FLAG_SM_Q5)
    u8_t  if_idx;           // 网卡索引
};
```

---

## 10. 关键配置 (lwipopts.h)

```
LWIP_NETCONN           1   // 使用 Netconn API (内部)
LWIP_SOCKET           1   // BSD socket API
LWIP_RAW             1   // RAW socket (AF-PACKET 依赖)
LWIP_TCPIP_CORE_LOCKING 1 // tcpip 核心锁 (避免锁竞争)

LWIP_SOCKET_NPOLL    1   // poll() 支持

MEMP_NUM_NETCONN     1024 // 最大 netconn 数量

LWIP_IPV6            0   // IPv6 禁用

NIO_LWIP_LWFW        1   // 启用 lwfw 防火墙
NIO_LWIP_LWCT        1   // 启用连接跟踪

LWIP_NETIF_TX_SINGLE_PBUF 1  // TX 时尝试用单个 pbuf
```

---

## 11. 性能关键点

### 11.1 CMA + elem_ring 实现零拷贝

NSv 和 NIC 驱动共享 96MB CMA 内存，通过物理地址的 elem_ring 传递 buffer 指针，避免数据拷贝。

### 11.2 tcpip_core_locking 减少锁竞争

`LWIP_TCPIP_CORE_LOCKING=1` 使 tcpip_thread 持有一个全局锁，其他线程调用 lwIP 函数时需要获取此锁，减少了消息传递开销。

### 11.3 LOCK_TCPIP_CORE / UNLOCK_TCPIP_CORE

```c
#define LOCK_TCPIP_CORE()   sys_mutex_lock(&lock_tcpip_core)
#define UNLOCK_TCPIP_CORE() sys_mutex_unlock(&lock_tcpip_core)
```

RX 路径在 nic_rx_thread 中调用 `LOCK_TCPIP_CORE()` 后再调用 `vnet_if.input()`，确保在 tcpip 核心锁保护下执行协议栈代码。
