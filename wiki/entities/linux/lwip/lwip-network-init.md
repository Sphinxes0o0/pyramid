---
type: entity
tags: [linux, lwip, network, initialization]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP Network Init Analysis

## 定义

SafeOS NSv 网络栈的初始化流程：资源初始化 → DMA ring 初始化 → NIC 线程创建 → lwIP 协议栈初始化 → 物理网口注册 → VLAN 网口初始化 → QoS 配置 → DNS 初始化。

## 完整初始化调用链

```
main()
    │
    ├─► net_resources_init()
    │     ├─ net_perf_stats_init()
    │     ├─ net_socket_info_init()
    │     └─ net_config_init()
    │
    ├─► init_ds_ring()
    │     ├─ cma_alloc() → CMA 96MB
    │     ├─ elem_ring_create(TX_BUF_NR)
    │     └─ elem_ring_create(RX_BUF_NR)
    │
    ├─► create_nic_thread()
    │     ├─ sel4_thread_create(nic_rx_thread)
    │     └─ sel4_notification_create()
    │
    ├─► tcpip_init()
    │     ├─ lwip_init()
    │     │     ├─ mem_init()
    │     │     ├─ memp_init()
    │     │     ├─ pbuf_init()
    │     │     ├─ netif_init()
    │     │     ├─ ip_init()
    │     │     ├─ raw_init()
    │     │     ├─ udp_init()
    │     │     ├─ tcp_init()
    │     │     └─ igmp_init()
    │     └─ sys_thread_new(tcpip_thread)
    │
    ├─► netif_add(&vnet_if, ...)
    │     └─ init_ethif()
    │
    ├─► vlanif_setup()
    │     ├─ vlanif_conf_init()
    │     └─ netif_add(&vlan_if[i], ...)
    │           └─ vlanif_init()
    │
    ├─► qos_setup()
    │
    └─► dns_setup()
```

## tcpip_init — lwIP 协议栈初始化

```c
void tcpip_init(tcpip_init_fn init_done, void *arg) {
    lwip_init();  // 初始化各模块
    sys_thread_new("tcpip", tcpip_thread, NULL, TCPIP_THREAD_STACKSIZE, TCPIP_THREAD_PRIO);
    tcpip_mbox = sys_mbox_new(TCPIP_MBOX_SIZE);
}

void lwip_init(void) {
    mem_init();    // 内存堆初始化
    memp_init();   // 内存池初始化
    pbuf_init();   // pbuf 子系统初始化
    netif_init();  // netif 链表初始化
    ip_init();     // IP 层初始化
    raw_init();    // RAW PCB 初始化
    udp_init();    // UDP PCB 初始化
    tcp_init();    // TCP PCB 初始化
    igmp_init();   // IGMP 初始化
    dns_init();    // DNS 初始化
}
```

## netif 链表状态

```
netif_list:
  │
  ├── vnet_if (物理网口)
  │     name = "et0"
  │     ip_addr = 172.20.0.1
  │     vlanid = 0
  │     input = ethernet_input
  │     output = etharp_output
  │     linkoutput = ethif_link_output
  │
  ├── vlan_if[0] (VLAN 100)
  │     name = "vl0"
  │     ip_addr = 172.20.100.1
  │     vlanid = 100
  │     input = ethernet_input
  │     output = etharp_output
  │     linkoutput = low_level_output
  │
  └── vlan_if[1] (VLAN 200)
        name = "vl1"
        ip_addr = 172.20.200.1
        vlanid = 200
        ...
```

## 线程状态

| 线程 | 优先级 | 职责 |
|------|--------|------|
| nic_rx_thread | 高 | 等待 NIC 中断，处理 RX |
| tcpip_thread | 中 | lwIP 协议栈处理 |
| event_loop | 主 | seL4 IPC 事件分发 |

## 相关概念

- [[entities/linux/lwip/lwip-tcpip-thread]] — tcpip_thread 主循环
- [[entities/linux/lwip/lwip-netif-add]] — netif_add 注册流程
- [[entities/linux/lwip/lwip-vlan-dispatch]] — VLAN netif 创建
- [[entities/linux/lwip/lwip-malloc]] — mem_init/memp_init 内存初始化
