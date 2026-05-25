# SafeOS lwIP 与 seL4 物理网卡/VLAN/Hypervisor 交互深度分析

> 文档版本: 1.0
> 更新日期: 2026/04/22
> 代码路径: `/home/shiyang/nio/nt35/safeos/`

---

## 1. 整体架构 — 组件交互总览

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           用户空间 (User Space)                                 │
│                                                                                  │
│  App 进程                                                                    │
│     │                                                                           │
│     ├─ socket() / sendto() / recvfrom()                                       │
│     └─ seL4 IPC (badge = pid)                                                 │
│            │                                                                    │
│            ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │                    NSv (网络服务器进程)                                    │   │
│  │                                                                          │   │
│  │  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────────┐   │   │
│  │  │  event_loop()  │  │ nic_rx_thread() │  │  tcpip_thread()       │   │   │
│  │  │  (App请求处理)   │  │  (RX 包处理)    │  │  (lwIP 协议栈)        │   │   │
│  │  └────────┬────────┘  └────────┬────────┘  └───────────┬────────────┘   │   │
│  │           │                    │                       │                  │   │
│  │           │ seL4 IPC           │ NIC Notification       │ tcpip_mbox      │   │
│  │           │ (svc_ep)            │ (nsv_nic_ep)           │                 │   │
│  │           ▼                    ▼                       ▼                  │   │
│  │  ┌──────────────────────────────────────────────────────────────┐       │   │
│  │  │              vnet_if (物理网卡 netif)                          │       │   │
│  │  │  name="et", vlanid=0, linkoutput=ethif_link_output           │       │   │
│  │  └──────────────────────────────────────────────────────────────┘       │   │
│  │           ▲                                                          │   │
│  │  ┌────────┴──────────────────────────────────────────────────────┐       │   │
│  │  │           vlan_if[i] (VLAN 网卡 netif)                        │       │   │
│  │  │  name="vl0", vlanid=100, linkoutput=low_level_output          │       │   │
│  │  │  → 调用 physical_netif->linkoutput (即 ethif_link_output)     │       │   │
│  │  └──────────────────────────────────────────────────────────────┘       │   │
│  │                                                                          │   │
│  │  ┌──────────────────────────────────────────────────────────────┐       │   │
│  │  │  CMA 共享内存 (96MB) + elem_ring x4                          │       │   │
│  │  │  ├─ empty_rx_buf_ring  (NSv → NIC)                          │       │   │
│  │  │  ├─ used_rx_buf_ring  (NIC → NSv)                          │       │   │
│  │  │  ├─ pending_tx_buf_ring (NSv → NIC)                          │       │   │
│  │  │  └─ used_tx_buf_ring   (NIC → NSv)                          │       │   │
│  │  └──────────────────────────────────────────────────────────────┘       │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                      │ seL4 IPC + CMA PA                    │
└──────────────────────────────────────┼──────────────────────────────────────────┘
                                       │
┌──────────────────────────────────────┼──────────────────────────────────────────┐
│                           NIC 驱动进程 (独立进程)                               │
│                                      │                                         │
│  nic_tx_ntfn (seL4 Notification)    │  nic_rx_ntfn (seL4 Notification)      │
│                                      │                                         │
│  elem_ring_put(pending_tx_buf) ─────┼── elem_ring_get(empty_rx_buf)           │
│       ↑                              │       ↑                                 │
│       │ DMA 发送完成                   │       │ DMA 接收                          │
│       │ elem_ring_put(used_tx_buf)    │       │ elem_ring_put(used_rx_buf)      │
│       │ sel4_signal(nic_rx_ntfn)       │       │ sel4_signal(nic_tx_ntfn)          │
└──────────────────────────────────────┼──────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Hypervisor / VMM 层                                     │
│                                                                                  │
│  #ifdef VIRT_BRG_SUPPORT:                                                       │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  ethif_link_output_overload()  ──► ethif_link_output() + vbridge_port   │   │
│  │                                                                          │   │
│  │  vbridge_evt_loop()  ←── seL4 IPC (BRIDGE_PORT_OUTPUT)                  │   │
│  │       │                                                                    │   │
│  │       ▼                                                                    │   │
│  │  vbridge_port_output()  ──► ethif_link_output() + tcpip_callback(rx_cb)  │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  #ifdef VNET_OVER_IPC_SUPPORT:                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │  ipcif_evt_loop()  ←── seL4 IPC (IPCF_NSV_NOTIFY_RX)                    │   │
│  │       │                                                                    │   │
│  │       ▼                                                                    │   │
│  │  work_thread()  ──► vnet_if.input() via msg_box                         │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. NSv 初始化 — 完整调用链

### 2.1 main() 初始化序列

```c
main()                                                           [main.c:6400]
  │
  ├─► net_resources_init()                               // netbuf, socket 表, 缓存初始化
  │
  ├─► init_ds_ring()                                    // CMA + DS-Ring 初始化
  │     ├─ sys_mem_map(getpid(), &cma.pa, &cma.va, CMA_SIZE, PAGE_DMA)
  │     │     // 从 CMA 分配 96MB 连续物理内存
  │     ├─ sys_dspace_create(cma.size, attr, &cma.va, &ds)
  │     │     // 创建 dspace 用于共享内存管理
  │     ├─ ds_ring_init(cma.va, cma.pa, ..., ds, pid, NSV_NIC_DESC_SIZE)
  │     │     // 初始化 DS-Ring (包含 elem_ring)
  │     ├─ sys_svc_wait(NIC_STR, SVC_WAIT_EXACT, &nic_ep)
  │     │     // 等待 NIC 驱动服务注册
  │     ├─ sys_dspace_grant(ds, nic_ep, DSPACE_GRANT_SVC_EP, attr)
  │     │     // 将 dspace 授权给 NIC 驱动
  │     └─ sys_ds_ring_share(nic_ep, ds)
  │           // 共享 DS-Ring 给 NIC 驱动
  │
  ├─► create_nic_thread()                              // 创建 NIC RX 线程
  │     ├─ sys_svc_alloc_endpoint(&nsv_nic_ep)        // 分配 seL4 IPC endpoint
  │     ├─ sys_svc_reg(NSV_NIC_INTERFACE_NAME, nsv_nic_ep, 0)
  │     │     // 注册为 NIC 接口服务 (其他进程可通过此名称访问)
  │     ├─ sys_kobj_alloc(KOBJ_TYPE_NTFN, 0, &nic_rx_ntfn)
  │     │     // 分配 seL4 notification 用于 NIC RX 通知
  │     └─ sys_thread_create(nic_rx_thread, ...)
  │
  ├─► tcpip_init(NULL, NULL)                           // lwIP TCPIP 线程初始化
  │     ├─ sys_mbox_new(&tcpip_mbox, TCPIP_MBOX_SIZE)
  │     │     // 创建 tcpip 线程邮箱
  │     └─ sys_thread_new(tcpip_thread, TCPIP_THREAD_PRIO)
  │           // 创建 tcpip_thread
  │
  ├─► netif_add(&vnet_if, &addr, &netmask, &gw, NULL, init_ethif, ethernet_input)
  │     // 创建物理网卡 netif
  │     └─ init_ethif(netif)                         // 见下方
  │
  ├─► vlanif_conf_init(vlan_conf_cnt)                 // 分配 VLAN netif 内存
  │
  ├─► vlanif_setup()                                  // 创建 VLAN netif
  │     ├─ vlanif_find_next_conf()                    // 从 YAML 查找 VLAN 配置
  │     ├─ netif_add(&vlan_if[i], vlanif_init, tcpip_input)
  │     │     // 创建 VLAN netif，input = tcpip_input (非 ethernet_input!)
  │     └─ vlanif_update_arp_entry()                 // 添加静态 ARP 条目
  │
  ├─► bridge_setup()                                  // VIRT_BRG_SUPPORT: 启动网桥
  │     └─ start_vbridge_thread()
  │           ├─ sys_svc_reg(NSV_VBRIDGE_SVC_NAME, vbridge_ep, 0)
  │           └─ sys_thread_create(vbridge_evt_loop, ...)
  │
  ├─► ipcif_setup()                                   // VNET_OVER_IPC_SUPPORT: 启动 IPC 接口
  │     └─ start_ipcif_thread()
  │           ├─ sys_svc_reg(NSV_VNETIPC_SVC_NAME, ipcif_ep, 0)
  │           ├─ sys_thread_create(ipcif_evt_loop, ...)
  │           └─ sys_thread_create(work_thread, ...)
  │
  ├─► sys_svc_reg("/net", svc_ep, 0)                // 注册 /net 服务 (App 通过此名称访问)
  │
  └─► event_loop()                                   // 进入主事件循环
```

### 2.2 init_ethif() — 物理网卡 netif 初始化

```c
init_ethif(struct netif *netif)                           [main.c:4710]
  │
  ├─► eth_init(netif->hwaddr)                          // 从 NIC 驱动获取 MAC 地址
  │
  ├─► netif->output = etharp_output                  // IPv4 输出: 查 ARP 表 → ethernet_output
  │
  ├─► #ifdef VIRT_BRG_SUPPORT
  │     netif->linkoutput = ethif_link_output_overload  // 网桥模式
  │   #else
  │     netif->linkoutput = ethif_link_output            // 普通模式
  │   #endif
  │
  ├─► #ifdef DRV_NIC_PFE
  │     netif->drv_op = eth_drv_ioctl                   // PFE 驱动 ioctl
  │   #endif
  │
  └─► netif->flags = NETIF_FLAG_ETHARP | NETIF_FLAG_BROADCAST |
                       NETIF_FLAG_LINK_UP | NETIF_FLAG_IGMP | NETIF_FLAG_ETHERNET
```

### 2.3 vlanif_init() — VLAN 网卡 netif 初始化

```c
vlanif_init(struct netif *netif)                          [vlanif.c:93]
  │
  ├─► int conf_idx = *((int *)netif->state)           // 从 netif->state 获取配置索引
  │
  ├─► netif->name[0] = vlan_conf[conf_idx].ifName[0] // 如 "vl"
  │     netif->name[1] = vlan_conf[conf_idx].ifName[1] // 如 "0"
  │
  ├─► netif->output = etharp_output                   // 同物理网卡
  │
  ├─► netif->linkoutput = low_level_output             // ★ 关键: 指向 low_level_output
  │
  ├─► netif->vlanid = strtol(vlan_conf[conf_idx].vid, NULL, 10)
  │     // ★ 关键: 设置 VLAN ID (如 100, 200)
  │
  └─► netif->input = tcpip_input                      // ★ 关键: VLAN 用 tcpip_input
        // 而物理网卡用 ethernet_input
```

---

## 3. 收包路径 (RX) — 从 NIC 到 App

### 3.1 完整调用链

```
NIC 驱动进程
  │
  │ DMA 接收 → 物理地址 (CMA 区域)
  │
  │ elem_ring_put(used_rx_buf_ring, elem{pa=buffer_pa})
  │   // used_rx_buf_ring: NIC 生产, NSv 消费
  │
  │ sel4_signal(nic_rx_ntfn)
  │   // 通知 NSv 有新数据包
  │
  ▼
nic_rx_thread()                                       [main.c:4961]
  │
  ├─► seL4_Recv(nsv_nic_ep, &badge)               // 等待 NIC notification
  │     // 阻塞在 seL4 IPC，直到 nic_rx_ntfn 被 signal
  │
  ├─► if (badge == 0) {                           // 正常的 RX 通知
  │     while (1) {
  │       elem = elem_ring_get(used_rx_buf_ring)   // 获取 buffer PA
  │       if (elem.pa == 0) break;                // 无更多包
  │       va = cma_pa_to_va(&cma, elem.pa)        // PA → VA
  │       LOCK_TCPIP_CORE()
  │       rx_callback((struct pbuf *)va)            // 处理包
  │       UNLOCK_TCPIP_CORE()
  │     }
  │   }
  │
  ▼
rx_callback(void *ctx)                              [main.c:4781]
  │
  ├─► p = (struct pbuf *)ctx
  │     p->next = 0; p->tot_len = p->len          // 分离 chain
  │
  ├─► #ifdef VIRT_BRG_SUPPORT
  │     if (p->flags != PBUF_FLAG_BRIDGE_OUTPUT) {
  │       vbridge_port.port_input(&vbridge_port, p->payload, p->tot_len)
  │       // vbridge_evt_loop 处理网桥转发
  │     }
  │   #endif
  │
  └─► vnet_if.input(p, &vnet_if)                   // ★ 进入 lwIP
        // 对于物理网卡: input = ethernet_input
        // 对于 VLAN netif: input = tcpip_input
        ▼
=====================================================================
                        lwIP 协议栈入口
=====================================================================

// ========== 物理网卡 (ethernet_input) ==========

ethernet_input(struct pbuf *p, struct netif *netif)  [ethernet.c:89]
  │
  ├─► ethhdr = (struct eth_hdr *)p->payload
  │
  ├─► type = ethhdr->type
  │
  ├─► #if ETHARP_SUPPORT_VLAN
  │     if (type == ETHTYPE_VLAN (0x8100)) {
  │       vlan = (struct eth_vlan_hdr *)(p->payload + 14)
  │       next_hdr_offset = 18  // 14 + 4
  │       │
  │       ├─► #ifdef MAC_VLAN_FILTER
  │       │     if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
  │       │       pbuf_free(p); return ERR_OK;  // VLAN ID 不匹配，丢弃
  │       │     }
  │       │   #endif
  │       │
  │       ├─► type = vlan->tpid                  // 真正 EtherType (如 ETHTYPE_IP)
  │       │
  │       └─► p->priority = (vlan->prio_vid >> 13)  // 提取 PCP 优先级
  │     }
  │   #endif
  │
  ├─► raw_afpacket_input(p, netif, type)        // AF-PACKET 捕获
  │     // 遍历 raw_afpacket_pcbs
  │     // 调用 tpacket_recv() 写入 ringbuf，唤醒 select/poll
  │
  └─► switch (type) {
          ETHTYPE_IP:   ip4_input(p, netif)
          ETHTYPE_ARP:  etharp_input(p, netif)
      }

// ========== VLAN 网卡 (tcpip_input) ==========

tcpip_input(struct pbuf *p, struct netif *inp)      [tcpip.c:288]
  │
  └─► tcpip_inpkt(p, inp, ethernet_input)          // 通过 tcpip_mbox 传递
        │
        ▼
  tcpip_inpkt(struct pbuf *p, ...)                [tcpip.c:243]
    │
    ├─► #if LWIP_TCPIP_CORE_LOCKING_INPUT
    │     LOCK_TCPIP_CORE()
    │     ret = input_fn(p, inp)  // → ethernet_input(p, inp)
    │     UNLOCK_TCPIP_CORE()
    │     return ret;
    │   #else
    │     // 放入 tcpip_mbox，由 tcpip_thread 处理
    │     msg = memp_malloc(MEMP_TCPIP_MSG_INPKT)
    │     msg->type = TCPIP_MSG_INPKT
    │     msg->msg.inp.p = p
    │     msg->msg.inp.netif = inp
    │     msg->msg.inp.input_fn = ethernet_input
    │     sys_mbox_trypost(&tcpip_mbox, msg)
    │   #endif

// ========== IP 层 ==========

ip4_input(struct pbuf *p, struct netif *inp)      [ip4.c:743]
  │
  ├─► pbuf_remove_header(p, IP_HLEN)              // 去掉 IP 头
  │
  ├─► #ifdef NIO_LWIP_LWFW                        // ===== 防火墙入方向 =====
  │     if (lwfw_p->ops->ingress_filter(p, inp) != ERR_OK) {
  │       pbuf_free(p); IP_STATS_INC(ip.drop); return ERR_OK;
  │     }
  │   #endif
  │
  └─► switch (IPH_PROTO(iphdr)) {
          IP_PROTO_TCP:  tcp_input(p, inp)
          IP_PROTO_UDP:  udp_input(p, inp)
          IP_PROTO_ICMP: icmp_input(p, inp)
          IP_PROTO_IGMP: igmp_input(p, inp)
      }

// ========== TCP 层 ==========

tcp_input(struct pbuf *p, struct netif *netif)     [tcp.c]
  │
  ├─► tcp_enqueue(p)                             // 将 pbuf 加入 receive queue
  │
  └─► tcp_process() / tcp_receive()
        // 触发 TCP 状态机
        // 最终调用 API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0)
        ▼
=====================================================================
                        返回 App
=====================================================================

API_EVENT(conn, NETCONN_EVT_RCVPLUS, 0)            [api_event.c]
  │
  ├─► sock->rcvevent++                           // 增加接收事件计数
  │
  └─► sys_sem_signal(&sock->select_waiting_sem)  // 唤醒 select/poll 等待者

event_loop()                                         [main.c:3400]
  │
  └─► seL4_Recv(svc_ep, &badge)                 // 等待 App 请求
        ▼
sys_recvfrom_nb(info, badge)                       [main.c:1368]
  │
  ├─► lwip_recvfrom(socket, recv_buf, data_len, flags, addr, &addrlen)
  │     // 从 netconn socket 收取数据
  │
  └─► sys_reply_with_one_direct(recv) / sel4_reply()
        // 通过 seL4 IPC 返回数据给 App
```

### 3.2 elem_ring — 无锁生产者/消费者队列

```c
// 关键: 单生产者, 单消费者, 无锁设计
// NIC 驱动: 生产者 (写入 used_rx_buf_ring, pending_tx_buf_ring)
// NSv:     消费者 (读取 used_rx_buf_ring, pending_tx_buf_ring)

// NIC 驱动放入 (生产者):
elem_ring_put(used_rx_buf_ring, elem{pa=buffer_pa})
  ├─ next = (put_idx + 1) % n
  ├─ if (next == get_idx) return -ENOSPC  // 队列满
  ├─ elems[put_idx] = elem                // 写入数据
  ├─ dmb(ish)                              // 内存屏障
  └─ put_idx = next                        // 更新索引

// NSv 取出 (消费者):
elem_ring_get(used_rx_buf_ring)
  ├─ if (get_idx == put_idx) return elem{pa=0}  // 队列空
  ├─ elem = elems[get_idx]                      // 读取数据
  ├─ dmb(ish)                                   // 内存屏障
  └─ get_idx = (get_idx + 1) % n               // 更新索引
```

---

## 4. 发包路径 (TX) — 从 App 到 NIC

### 4.1 完整调用链

```
App (用户进程)
  │
  │ sys_net_sendto(ep, socket, buf, len, flags, use_shm, addr, addrlen, &sent)
  │   └─ seL4_Call(svc_ep, SYS_NET_SENDTO)
  │
  ▼
event_loop()                                     [main.c:3400]
  │
  └─► seL4_Recv(svc_ep, &badge)               // 等待 App 请求
        ▼
sys_sendto_nb(info, badge)                      [main.c:1274]
  │
  ├─► sel4_get_mr(0..5)                       // 从 seL4 message registers 获取参数
  │     // socket=mr0, data_len=mr1, flags=mr2, socklen=mr3, use_shm=mr4, offset=mr5
  │
  ├─► if (use_shm) {
  │     send_buf = get_shm_va(pid, offset, data_len)  // 共享内存
  │   } else {
  │     send_buf = alloc_data_cache()
  │     sys_unpack_data_from_mrs(5, send_buf, data_len, &next_mr)
  │   }
  │
  └─► lwip_sendto(socket, send_buf, data_len, flags, addr, addrlen)
        ▼
lwip_sendto(int s, const void *data, size_t len, ...)  [sockets.c]
  │
  ├─► sock = get_socket(s)                     // 从 fd 找到 lwip_sock
  │
  └─► netconn_sendto(sock->conn, data, len, addr, addrlen)
        ▼
netconn_sendto(struct netconn *conn, ...)     [api_msg.c]
  │
  ├─► #ifdef LWIP_NETCONN_SEND_PTON
  │     netconn_send_data(conn, buf, len)     // UDP/RAW 直接发数据
  │   #endif
  │
  └─► tcpip_callback_with_output(..., netconn_send, conn)
        // 对于 TCP: tcpip_outpu_tcp()
        // 对于 UDP: udp_sendto() → ip4_output_if()

// ========== UDP 为例 ==========

udp_sendto(struct udp_pcb *pcb, struct pbuf *p,     [udp.c]
             const ip_addr_t *dst_ip, u16_t dst_port)
  │
  └─► ip4_output_if(p, src_ip, dst_ip, proto, netif)
        ▼
ip4_output_if(p, src_ip, dst_ip, proto, netif)  [ip4.c:888]
  │
  ├─► ip4_output_if_opt_src() → ip4_route()   // 查找路由，确定 netif
  │
  ├─► #ifdef NIO_LWIP_LWFW                     // ===== 防火墙出方向 =====
  │     if (lwfw_p->ops->egress_filter(p, netif) != ERR_OK) {
  │       pbuf_free(p); return ERR_FW;           // 丢弃
  │     }
  │   #endif
  │
  └─► netif->output(netif, p, dest)           // ★ etharp_output
        ▼
etharp_output(struct netif *netif, struct pbuf *q, const ip4_addr_t *ipaddr)
                                                              [etharp.c:825]
  │
  ├─► if (ip4_addr_isbroadcast(ipaddr, netif)) {
  │     dest = &ethbroadcast
  │   } else if (ip4_addr_ismulticast(ipaddr)) {
  │     // 组播: IP → MAC 映射
  │     dest = &mcastaddr
  │   } else {
  │     // 单播: 查 ARP 表
  │     arp_idx = etharp_find_entry(ipaddr, ...)
  │     if (stable_entry) {
  │       return etharp_output_to_arp_index(netif, q, arp_idx)
  │     } else {
  │       return etharp_query(netif, ipaddr, q)  // 发送 ARP 请求
  │     }
  │   }
        ▼
etharp_output_to_arp_index(netif, q, arp_idx)   [etharp.c:782]
  │
  └─► ethernet_output(netif, q, netif->hwaddr, &arp_table[arp_idx].ethaddr, ETHTYPE_IP)
        ▼
ethernet_output(netif, p, src, dst, eth_type)  [ethernet.c:333]
  │
  ├─► #if ETHARP_SUPPORT_VLAN && defined(LWIP_HOOK_VLAN_SET)
  │     vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type)
  │     │   └─► lwip_hook_vlan_set_fn(netif, pbuf, ...)
  │     │         ├─ if (netif->vlanid == NO_VLANID) return -1  // 不带 VLAN
  │     │         └─ return (pbuf->priority << 13) | netif->vlanid
  │     │               // 组合 PCP (高3位) + VID (低12位)
  │     │
  │     if (vlan_prio_vid >= 0) {
  │       pbuf_add_header(p, SIZEOF_ETH_HDR + SIZEOF_VLAN_HDR)  // 增加 4 字节
  │       vlanhdr = (struct eth_vlan_hdr *)(p->payload + 14)
  │       vlanhdr->tpid = eth_type_be         // 保存原始 EtherType
  │       vlanhdr->prio_vid = lwip_htons(vlan_prio_vid)
  │       eth_type_be = ETHTYPE_VLAN (0x8100)  // 改为 VLAN EtherType
  │     }
  │   #endif
  │
  ├─► pbuf_add_header(p, SIZEOF_ETH_HDR)      // 增加 14 字节
  │
  ├─► ethhdr = (struct eth_hdr *)p->payload
  │     ethhdr->type = eth_type_be
  │     SMEMCPY(&ethhdr->dest, dst, ETH_HWADDR_LEN)
  │     SMEMCPY(&ethhdr->src, src, ETH_HWADDR_LEN)
  │
  ├─► raw_afpacket_output(p, netif)           // AF-PACKET 通知
  │
  └─► netif->linkoutput(netif, p)             // ★ 发送到网卡
        // 物理网卡: ethif_link_output()
        // VLAN 网卡: low_level_output() → ethif_link_output()
        ▼
=====================================================================
                      物理网卡 linkoutput
=====================================================================

low_level_output(struct netif *netif, struct pbuf *p)  [vlanif.c:52]
  │
  └─► physical_netif->linkoutput(physical_netif, p)
        // 直接调用物理网卡的 ethif_link_output()

ethif_link_output(struct netif *netif, struct pbuf *q)  [main.c:3788]
  │
  ├─► if (!nic_ready) return ERR_OK            // NIC 未就绪则丢弃
  │
  ├─► LWIP_ASSERT_CORE_LOCKED_SERIOUS()
  │
  ├─► #ifdef USE_SEND_SMOOTH_QAV
  │     if (q->flags & PBUF_FLAG_SM_Q5/Q6) {
  │       sm_post_element(sm_que5/6_buf_ring, q)  // QoS 整形队列
  │       return ERR_OK;
  │     }
  │   #endif
  │
  ├─► // 分配新 pbuf 并复制数据 (TX single pbuf)
  │     p = lwip_malloc(length + SIZEOF_STRUCT_PBUF + ...)
  │     pbuf_init_alloced_pbuf(p, payload_va, length, length, PBUF_RAM, 0)
  │     memcpy(p->payload, q->payload, length)
  │
  ├─► free_complete_tx_packet_pbuf()            // 回收已完成的 TX buffer
  │
  ├─► elem = {pa: cma_va_to_pa(&cma, p)}       // pbuf VA → CMA PA
  │
  ├─► elem_ring_put(pending_tx_buf_ring, elem) // 放入待发队列
  │     // pending_tx_buf_ring: NSv 生产, NIC 消费
  │
  ├─► if (was_empty || is_full) {
  │     sel4_signal(nic_tx_ntfn)                // ★ 通知 NIC 有数据包
  │   }
  │
  └─► return ERR_OK

// ========== NIC 驱动端 ==========

NIC 驱动进程
  │
  │ seL4_Wait(nic_tx_ep, ...)                  // 等待 TX 通知
  │
  ├─► elem = elem_ring_get(pending_tx_buf_ring)  // 获取 TX buffer PA
  │
  ├─► DMA 发送 (使用 buffer PA)
  │
  ├─► elem_ring_put(used_tx_buf_ring, elem)      // 放入完成队列
  │
  └─► sel4_signal(nic_rx_ntfn)                  // 通知 NSv 发送完成

// NSv 端 (在下次 rx_callback 时):
free_complete_tx_packet_pbuf()                    [main.c:3773]
  │
  └─► while (1) {
          elem = elem_ring_get(used_tx_buf_ring)  // 获取完成的 TX buffer
          if (elem.pa == 0) break
          p = cma_pa_to_va(&cma, elem.pa)
          lwip_free(p)                            // 释放 pbuf
        }
```

---

## 5. VLAN Tag 处理 — RX/TX 详解

### 5.1 RX: VLAN Tag 解析和 VID 检查

```
ethernet_input() 中的 VLAN 处理:                     [ethernet.c:133-169]

1. 检查 EtherType:
   if (type == ETHTYPE_VLAN (0x8100))

2. 解析 VLAN Tag:
   vlan = (struct eth_vlan_hdr *)(p->payload + 14)
   next_hdr_offset = 18  // 14 + 4 (ETH_HDR + VLAN_HDR)

3. VLAN ID 检查 (MAC_VLAN_FILTER):
   #ifdef MAC_VLAN_FILTER
     if (!LWIP_HOOK_VLAN_CHECK(netif, ethhdr, vlan)) {
       pbuf_free(p); return ERR_OK;  // VID 不匹配，丢弃
     }
   #endif

   lwip_hook_vlan_check_fn():
     vid_netif = netif->vlanid & 0x0FFF
     vid_pkt = vlan->prio_vid & 0x0FFF
     if (vid_netif == vid_pkt) return 1  // 匹配
     return 0                           // 不匹配

4. 提取 PCP 优先级:
   p->priority = (vlan->prio_vid >> 13)  // 取高 3 位

5. 替换 EtherType:
   type = vlan->tpid  // 如 ETHTYPE_IP (0x0800)
```

### 5.2 TX: VLAN Tag 插入

```
ethernet_output() 中的 VLAN 处理:                   [ethernet.c:339-355]

1. 调用 Hook 获取 VLAN ID:
   vlan_prio_vid = LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type)
     └─ lwip_hook_vlan_set_fn():
          if (netif->vlanid == NO_VLANID) return -1  // 不带 VLAN
          return (pbuf->priority << 13) | netif->vlanid
          // 组合: PCP (高 3 位) | VID (低 12 位)

2. 如果 vlan_prio_vid >= 0:
   pbuf_add_header(p, 18)  // 增加 18 字节空间
   vlanhdr = (struct eth_vlan_hdr *)(p->payload + 14)
   vlanhdr->tpid = eth_type_be     // 保存原始 EtherType
   vlanhdr->prio_vid = lwip_htons(vlan_prio_vid)
   eth_type_be = ETHTYPE_VLAN     // 改为 0x8100

3. 最终以太网头:
   DMAC(6) | SMAC(6) | TPID=0x8100 | TCI(PCP+DEI+VID) | EtherType
```

### 5.3 VLAN netif 的特殊路径

| 步骤 | 物理网卡 (vnet_if) | VLAN 网卡 (vlan_if[i]) |
|------|---------------------|--------------------------|
| `netif->input` | `ethernet_input()` | `tcpip_input()` |
| `netif->linkoutput` | `ethif_link_output()` | `low_level_output()` |
| `netif->vlanid` | 0 (NO_VLANID) | 配置的 VID |
| `LWIP_HOOK_VLAN_SET` | 不插入 VLAN Tag | 插入 VLAN Tag |
| `LWIP_HOOK_VLAN_CHECK` | N/A (type != VLAN) | 检查 VID 匹配 |

**VLAN → 物理网卡的桥接**:
```
VLAN netif (vlan_if[i]) 
    ↓ low_level_output()
physical_netif->linkoutput() = ethif_link_output()
    ↓
pending_tx_buf_ring → NIC
```

---

## 6. Hypervisor / VMM 网桥模式

### 6.1 VIRT_BRG_SUPPORT: ethif_link_output_overload

当定义了 `VIRT_BRG_SUPPORT` 时，物理网卡的 `linkoutput` 指向 `ethif_link_output_overload`:

```c
ethif_link_output_overload(struct netif *netif, struct pbuf *p)  [bridge.c:38]
  │
  ├─► if (vbridge_port.port_input) {
  │     // 遍历 pbuf chain
  │     pbuf *q = p;
  │     while (q != 0) {
  │       q->tot_len = q->len;
  │       vbridge_port.port_input(&vbridge_port, q->payload, q->tot_len)
  │       // 发送到 vbridge_evt_loop 处理
  │       q = q->next;
  │     }
  │   }
  │
  └─► ethif_link_output(netif, p)             // 同时正常发送
```

### 6.2 vbridge_evt_loop — 网桥事件循环

```c
vbridge_evt_loop(void *arg)                     [bridge.c:70]
  │
  └─► while (1) {
          seL4_Recv(vbridge_ep, &badge)
          switch (label) {
            case BRIDGE_PORT_OUTPUT:
              idx = sel4_get_mr(0)
              // 从共享内存读取数据
              memcpy(&data, shm_base + idx * BRIDGE_SHM_SIZE, sizeof(data))
              vbridge_port_output(&vbridge_port, data.payload, data.len)
          }
        }
```

### 6.3 vbridge_port_output — 发送数据包到网桥

```c
vbridge_port_output(bridge_port_t *port, void *payload, int len)  [bridge.c:52]
  │
  ├─► p = pbuf_alloc(PBUF_RAW, len, PBUF_POOL)
  │
  ├─► p->flags = PBUF_FLAG_BRIDGE_OUTPUT  // 标记为网桥输出
  │
  ├─► memcpy(p->payload, payload, len)
  │
  ├─► ethif_link_output(&vnet_if, p)      // 发送 (不走 vbridge_port_input)
  │
  └─► tcpip_callback(rx_callback, p)       // 同时注入到 RX 路径
        // 这样 VM 发送的包也会被 host 收到
```

---

## 7. IPCIF — VNET_OVER_IPC_SUPPORT

### 7.1 IPCIF 架构

IPCIF (IPC Network Interface) 允许 VM 通过 IPC 与 NSv 通信:

```
VM (虚拟网卡)
  │
  │ IPCF_NSV_NOTIFY_RX (seL4 IPC)
  │
  ▼
ipcif_evt_loop()                             [ipc-if.c:774]
  │
  ├─► seL4_Recv(ipcif_ep, &badge)
  │
  ├─► case IPCF_NSV_NOTIFY_RX:
  │     pa = sel4_get_mr(0)                  // DMA buffer 物理地址
  │     size = sel4_get_mr(1)
  │     work = {dma_buf: {pa, va, size}, cmd: VNET_PROCESS_RX}
  │     msg_box_post(&work_mbox, work)
  │
  ▼
work_thread()                                 [ipc-if.c:832]
  │
  └─► while (1) {
          work = msg_box_fetch(&work_mbox)
          switch (work->cmd) {
            case VNET_PROCESS_RX:
              p = cma_pa_to_va(work->dma_buf.pa)
              vnet_if.input(p, &vnet_if)     // 注入到 lwIP RX 路径
          }
        }
```

---

## 8. tcpip_thread — lwIP 核心锁机制

### 8.1 LWIP_TCPIP_CORE_LOCKING

```c
// tcpip.c:65-67
#if LWIP_TCPIP_CORE_LOCKING
sys_mutex_t lock_tcpip_core;  // 全局核心锁
#endif

// tcpip_thread:136
LOCK_TCPIP_CORE();             // 获取锁后进入主循环
while (1) {
  TCPIP_MBOX_FETCH(&tcpip_mbox, &msg);
  tcpip_thread_handle_msg(msg);
}

// tcpip_inpkt:245-250
#if LWIP_TCPIP_CORE_LOCKING_INPUT
  LOCK_TCPIP_CORE();
  ret = input_fn(p, inp);      // 直接调用 input 函数
  UNLOCK_TCPIP_CORE();
#else
  // 放入 tcpip_mbox，由 tcpip_thread 处理
#endif
```

### 8.2 NSv 中的锁使用

**RX 路径** (nic_rx_thread 中):
```c
LOCK_TCPIP_CORE();
rx_callback((struct pbuf *)va);    // 调用 vnet_if.input
UNLOCK_TCPIP_CORE();
```

**TX 路径** (event_loop/sys_sendto_nb 线程中):
```c
// 不需要显式加锁，因为:
// 1. lwIP socket API 内部会处理锁
// 2. 最终通过 tcpip_callback_with_output 传递
```

---

## 9. 关键数据结构关系

### 9.1 netif 与 vlanid

```c
// 物理网卡
struct netif vnet_if = {
    .name = {'e', 't'},
    .vlanid = 0,                    // NO_VLANID
    .input = ethernet_input,
    .linkoutput = ethif_link_output,  // 普通: ethif_link_output_overload
};

// VLAN 网卡
struct netif vlan_if[0] = {
    .name = {'v', 'l'},
    .vlanid = 100,                   // VID = 100
    .input = tcpip_input,            // ★ 不同: tcpip_input (非 ethernet_input)
    .linkoutput = low_level_output,  // ★ 不同: low_level_output
};
```

### 9.2 elem_ring 与 CMA 的关系

```
CMA 区域 (96MB)
├─────────────────────────────────────────────────────────┐
│  elem_ring x4                                         │
│  ├─ empty_rx_buf_ring   (NSv → NIC, 提供空 buffer)    │
│  ├─ used_rx_buf_ring   (NIC → NSv, 收到数据包)       │
│  ├─ pending_tx_buf_ring (NSv → NIC, 待发送数据包)     │
│  └─ used_tx_buf_ring   (NIC → NSv, 发送完成)          │
├─────────────────────────────────────────────────────────┤
│  DMA Buffer (pbuf)                                     │
│  ├─ Buffer 0: 1576 bytes (PA)                         │
│  ├─ Buffer 1: 1576 bytes (PA)                         │
│  └─ Buffer N: 1576 bytes (PA)                        │
└─────────────────────────────────────────────────────────┘
       ↑
       │ elem_ring 存的是 buffer PA
       │ elem_ring_get() → PA → cma_pa_to_va() → VA → pbuf*
```

---

## 10. 关键文件索引

| 文件 | 职责 |
|------|------|
| `servers/net/src/main.c` | NSv 入口、init_ethif、ethif_link_output、nic_rx_thread、event_loop |
| `servers/net/src/vlanif.c` | vlanif_init、low_level_output、vlanif_setup |
| `servers/net/src/bridge.c` | ethif_link_output_overload、vbridge_evt_loop、vbridge_port_output |
| `servers/net/src/ipc-if.c` | ipcif_evt_loop、work_thread、VNET_OVER_IPC_SUPPORT |
| `servers/net/src/conf_parser.c` | YAML 配置解析、vlan_conf、vlan_arp_conf |
| `external/lwip_ds_mcu/src/netif/ethernet.c` | ethernet_input、ethernet_output、lwip_hook_vlan_check_fn、lwip_hook_vlan_set_fn |
| `external/lwip_ds_mcu/src/core/ipv4/etharp.c` | etharp_output、etharp_output_to_arp_index |
| `external/lwip_ds_mcu/src/core/ipv4/ip4.c` | ip4_input、ip4_output_if (防火墙 hook 位置) |
| `external/lwip_ds_mcu/src/api/tcpip.c` | tcpip_init、tcpip_thread、tcpip_input、tcpip_inpkt |
| `libs/util_libs/liblwip/src/sys_arch_sel4.c` | seL4 适配层 (信号量、mutex、mbox) |
| `libs/os_libs/libcore/include/core/elem_ring.h` | elem_ring 无锁队列实现 |
