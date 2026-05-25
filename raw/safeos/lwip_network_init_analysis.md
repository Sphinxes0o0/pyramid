# 网络初始化流程分析 — T-114

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: lwip_init → tcpip_init → netif_add → vlanif_setup 完整初始化流程

---

## 1. 概述

SafeOS NSv 网络栈的初始化遵循以下顺序：
1. 资源初始化 (net_resources_init)
2. DMA ring 初始化 (init_ds_ring)
3. NIC 线程创建 (create_nic_thread)
4. lwIP 协议栈初始化 (tcpip_init)
5. 物理网口注册 (netif_add)
6. VLAN 网口初始化 (vlanif_setup)
7. QoS 配置 (qos_setup)
8. DNS 初始化 (dns_setup)

---

## 2. 初始化入口

**文件**: `main.c:6395`

```c
int main(int argc, char *argv[])
{
    LOG_ALWAYS("Network server! %d %s pid is %d\n", argc, argv[0], getpid());
    int err = 0;
    nsv_opstate = NSV_NOT_READY;

    signal(SIGINT, sig_handler);
    signal(SIGTERM, sig_handler);

    // Step 1: 资源初始化
    err = net_resources_init();
    if (err != 0) {
        return err;
    }

    // Step 2: DMA ring 初始化
    err = init_ds_ring();
    if (err != 0) {
        goto err_init_ds_ring;
    }

    // Step 3: NIC 接收线程创建
    err = create_nic_thread();
    if (err != 0) {
        goto err_create_nic_thread;
    }

    // Step 4: lwIP 协议栈初始化
    tcpip_init(0, 0);
    ZF_LOGV("LWIP inited\n");
    // ...
}
```

---

## 3. Step 1: net_resources_init — 资源初始化

**文件**: `main.c`

```c
static int net_resources_init(void)
{
    // 1. 初始化网络统计
    net_perf_stats_init();

    // 2. 初始化 socket 信息表
    net_socket_info_init();

    // 3. 初始化进程 PID 表
    net_process_info_init();

    // 4. 初始化网络配置
    net_config_init();

    return 0;
}
```

### 3.1 网络统计初始化

```c
static void net_perf_stats_init(void)
{
    memset(&net_perf_stats, 0, sizeof(net_perf_stats));
    // 初始化 TX/RX 计数、错误计数等
}
```

### 3.2 Socket 信息表初始化

```c
#define NUM_SOCKETS 256
static net_socket_info_t net_socket_info[NUM_SOCKETS];

static void net_socket_info_init(void)
{
    memset(net_socket_info, 0, sizeof(net_socket_info));
    // 初始化 socket owner 为 0 (未使用)
}
```

---

## 4. Step 2: init_ds_ring — DMA Ring 初始化

**文件**: `main.c`

```c
static int init_ds_ring(void)
{
    // 1. 分配 CMA (Contiguous Memory Area) 区域
    cma = cma_alloc(CMA_SIZE);
    if (!cma) {
        ZF_LOGE("Failed to allocate CMA memory\n");
        return -1;
    }

    // 2. 初始化 TX/RX 环形缓冲区
    pending_tx_buf_ring = elem_ring_create(TX_BUF_NR);
    used_rx_buf_ring = elem_ring_create(RX_BUF_NR);

    // 3. 映射 DMA 缓冲区
    dma_buf_alloc(&dma_pbuf_pool, DMA_BUF_NR);
    // ...
}
```

### 4.1 CMA 区域分配

```c
cma = cma_alloc(CMA_SIZE);  // CMA_SIZE = 0x6000000 (96MB)

typedef struct {
    void *vaddr;           // 虚拟地址
    uint64_t dev_paddr;    // 物理地址 (NIC DMA 使用)
    size_t size;           // 大小
} cma_t;
```

### 4.2 环形缓冲区创建

```c
// TX buffer ring - NIC 发送队列
pending_tx_buf_ring = elem_ring_create(TX_BUF_NR);

// RX buffer ring - NIC 接收队列
used_rx_buf_ring = elem_ring_create(RX_BUF_NR);
```

---

## 5. Step 3: create_nic_thread — NIC 线程创建

**文件**: `main.c:4961`

```c
static int create_nic_thread(void)
{
    // 创建 seL4 线程
    sel4_thread_create(nic_rx_thread, stack, priority);

    // 创建通知机制用于 NIC → NSv 通信
    nic_tx_ntfn = sel4_notification_create();
    nic_rx_ntfn = sel4_notification_create();

    return 0;
}
```

### 5.1 NIC 接收线程

```c
static void nic_rx_thread(void *arg)
{
    while (1) {
        // 等待 NIC 中断通知
        seL4_Recv(nic_rx_ep, &badge);

        // 处理接收到的数据包
        while (elem_ring_get(used_rx_buf_ring, &e)) {
            // 物理地址 → pbuf 指针
            pbuf *p = cma_pa_to_va(&cma, e.pa);

            // 调用 rx_callback
            rx_callback(p);
        }
    }
}
```

---

## 6. Step 4: tcpip_init — lwIP 协议栈初始化

**文件**: `external/lwip_ds_mcu/src/core/tcpip.c`

```c
void tcpip_init(tcpip_init_done_fn init_done, void *arg)
{
    // 1. 初始化 lwIP 内存系统
    lwip_init();

    // 2. 创建 tcpip 线程
    sys_thread_new("tcpip", tcpip_thread, NULL, TCPIP_THREAD_STACKSIZE, TCPIP_THREAD_PRIO);

    // 3. 初始化mbox
    tcpip_mbox = sys_mbox_new(TCPIP_MBOX_SIZE);
}

void lwip_init(void)
{
    // 初始化各模块
    mem_init();        // 内存堆初始化
    memp_init();       // 内存池初始化
    pbuf_init();       // pbuf 子系统初始化
    netif_init();      // netif 链表初始化
    ip_init();         // IP 层初始化
    raw_init();        // RAW PCB 初始化
    udp_init();        // UDP PCB 初始化
    tcp_init();        // TCP PCB 初始化
    igmp_init();       // IGMP 初始化
    dns_init();        // DNS 初始化
}
```

---

## 7. Step 5: netif_add — 物理网口注册

**文件**: `main.c:6437-6447`

```c
// 配置 IP 地址
IP4_ADDR(&addr, 172, 20, 0, 1);
IP4_ADDR(&netmask, 255, 255, 255, 0);
IP4_ADDR(&gw, 172, 20, 0, 12);

// 设置网口名称
vnet_if.name[0] = PHYSICAL_IFNAME[0];  // 'e' 或 'v'
vnet_if.name[1] = PHYSICAL_IFNAME[1];  // 'n0', 't0' 等

// 注册网口
struct netif *netif = netif_add(
    &vnet_if,                    // netif 结构
    &addr, &netmask, &gw,       // IP 配置
    NULL,                         // state (VLAN 使用)
    init_ethif,                  // 初始化函数
    ethernet_input                // 输入函数
);

// 设置网口状态
netif_set_link_up(&vnet_if);   // 链路 UP
netif_set_up(&vnet_if);         // 网口 UP
netif_set_status_callback(&vnet_if, netif_status_callback);
```

### 7.1 init_ethif — 网口初始化函数

```c
static err_t init_ethif(struct netif *netif)
{
    // 设置输出函数
    netif->output = etharp_output;      // IP → ARP → Ethernet
    netif->output_ip6 = ethip6_output;

    // 设置 link output 函数
    netif->linkoutput = ethif_link_output;  // 实际发送函数

    // 设置网口标志
    netif->flags = NETIF_FLAG_BROADCAST |
                   NETIF_FLAG_ETHARP |
                   NETIF_FLAG_LINK_UP |
                   NETIF_FLAG_IGMP;

    // 设置 MTU
    netif->mtu = ETH_MTU;

    // 设置 MAC 地址
    netif->hwaddr_len = ETH_HWADDR_LEN;
    netif_set_hwaddr(netif, ethaddr, ETH_HWADDR_LEN);

    return ERR_OK;
}
```

---

## 8. Step 6: vlanif_setup — VLAN 网口初始化

**文件**: `vlanif.c`

```c
#if ETHARP_SUPPORT_VLAN
static int vlanif_setup(void)
{
    // 1. 获取 VLAN 配置数量
    const unsigned int vlan_conf_cnt = get_conf_entry_count(VLAN_CONF_ENTRIES_COUNT);

    // 2. 初始化 VLAN 配置
    err = vlanif_conf_init(vlan_conf_cnt);
    if (err != 0) {
        return err;
    }

    // 3. 为每个 VLAN 创建 netif
    for (int i = 0; i < vlan_conf_cnt; i++) {
        // 分配 netif 结构
        struct netif *vlan_netif = &vlan_if[i];

        // 设置名称 (如 "vl0", "vl1")
        vlan_netif->name[0] = 'v';
        vlan_netif->name[1] = 'l';

        // 设置 VLAN ID
        vlan_netif->vlanid = atoi(vlan_conf[i].vid);

        // 设置 IP 地址
        ip4_addr_t addr, netmask;
        IP4_ADDR(&addr, 172, 20, vlanid, 1);

        // 注册 netif
        netif_add(vlan_netif, &addr, &netmask, &gw, NULL, vlanif_init, ethernet_input);

        // 启用网口
        netif_set_up(vlan_netif);
        netif_set_link_up(vlan_netif);
    }

    return 0;
}
#endif
```

### 8.1 vlanif_init — VLAN 网口初始化函数

```c
static err_t vlanif_init(struct netif *netif)
{
    // VLAN netif 的特点：
    // 1. 使用物理网口的 linkoutput
    netif->linkoutput = low_level_output;  // → ethif_link_output

    // 2. 设置 VLAN ID
    netif->vlanid = strtol(vlan_conf[conf_idx].vid, NULL, 10);

    // 3. 清除 ETHARP flag (VLAN 网口不需要 ARP)
    netif->flags &= ~NETIF_FLAG_ETHARP;

    return ERR_OK;
}
```

---

## 9. Step 7: qos_setup — QoS 配置

**文件**: `main.c`

```c
#if ETH_SUPPORT_QOS
static int qos_setup(void)
{
    // 1. 初始化 QoS 队列
    qos_queue_init();

    // 2. 配置 QoS 规则
    qos_config_parse();

    return 0;
}
#endif
```

---

## 10. Step 8: dns_setup — DNS 初始化

**文件**: `main.c`

```c
#if LWIP_DNS
static int dns_setup(void)
{
    // 1. 添加默认 DNS 服务器
    ip_addr_t dns_server;
    IP4_ADDR(&dns_server, 8, 8, 8, 8);
    dns_setserver(0, &dns_server);

    return 0;
}
#endif
```

---

## 11. 初始化完成后的状态

### 11.1 netif 链表状态

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
  │     linkoutput = low_level_output → ethif_link_output
  │
  └── vlan_if[1] (VLAN 200)
        name = "vl1"
        ip_addr = 172.20.200.1
        vlanid = 200
        input = ethernet_input
        output = etharp_output
        linkoutput = low_level_output → ethif_link_output
```

### 11.2 线程状态

| 线程 | 优先级 | 职责 |
|------|--------|------|
| nic_rx_thread | 高 | 等待 NIC 中断，处理 RX |
| tcpip_thread | 中 | lwIP 协议栈处理 |
| vbridge_evt_loop | 中 | VIRT_BRG 网桥事件 (如果启用) |
| ipcif_evt_loop | 中 | IPCIF 事件 (如果启用) |
| event_loop | 主 | seL4 IPC 事件分发 |

---

## 12. 完整初始化调用链

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

---

## 13. 总结

### 13.1 初始化关键点

1. **资源优先**：先初始化 CMA、ring buffer 等物理资源
2. **协议栈次之**：lwip_init 初始化内存和协议数据结构
3. **网口最后**：注册 netif，确保协议栈就绪

### 13.2 VLAN 与物理网口的关系

| 属性 | 物理网口 (vnet_if) | VLAN 网口 (vlan_if[i]) |
|------|-------------------|------------------------|
| **input** | ethernet_input | ethernet_input |
| **output** | etharp_output | etharp_output |
| **linkoutput** | ethif_link_output | low_level_output |
| **vlanid** | 0 | 配置值 (如 100, 200) |
| **ETHARP flag** | 有 | 无 |

### 13.3 安全考虑

- VLAN 网口的 linkoutput 指向 low_level_output，最终调用物理网口的 linkoutput
- VLAN ID 在 RX 分发时用于匹配正确的 vlan_if
- TX 时通过 LWIP_HOOK_VLAN_SET 添加 VLAN tag
