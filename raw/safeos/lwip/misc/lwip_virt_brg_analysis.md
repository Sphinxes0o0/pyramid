# VIRT_BRG_SUPPORT 分析 — T-093

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: VIRT_BRG_SUPPORT 与 hypervisor 交互、port_input/port_output

---

## 1. 概述

VIRT_BRG_SUPPORT 是 SafeOS 的虚拟网桥支持，允许 NSv 通过 hypervisor 与虚拟机进行网络通信。

**主要特性**：
- 作为 bridge port 集成到 lwIP bridgeif
- 通过 seL4 IPC 与 hypervisor 通信
- 支持 VM ↔ VM 和 VM ↔ NSv 的数据包转发

---

## 2. 核心数据结构

### 2.1 bridge_port_t

**文件**: `bridge.h`

```c
typedef struct bridge_port_s {
    char name[BRIDGE_NAME_LEN];          // 端口名称
    uint8_t enabled;                     // 是否启用
    struct dsm_info port_dsm;           // 共享内存信息
    uint32_t port_dsm_offset;           // DSM 偏移
    void (*port_input)(...);            // 输入回调
    void (*port_output)(...);           // 输出回调
    // ...
} bridge_port_t;
```

### 2.2 bridge_port_data_t

```c
typedef struct {
    uint32_t len;                        // 数据长度
    uint8_t payload[BRIDGE_SHM_SIZE];   // 数据负载
} bridge_port_data_t;
```

### 2.3 全局变量

```c
bridge_port_t vbridge_port = { 0 };  // 虚拟网桥端口
static tid_t vbridge_tid = 0;         // 网桥线程 TID
```

---

## 3. 核心流程

### 3.1 ethif_link_output_overload — TX 劫持

**文件**: `bridge.c:38-50`

当 VIRT_BRG_SUPPORT 启用时，物理网口的 `linkoutput` 被替换为此函数：

```c
err_t ethif_link_output_overload(struct netif *netif, struct pbuf *p)
{
    if (vbridge_port.port_input) {
        // 遍历 pbuf 链表，将每个 segment 发送到 bridge
        struct pbuf *q = p;
        while (q != NULL) {
            struct pbuf *next = q->next;
            q->tot_len = q->len;
            // 调用 bridge port 的 input
            vbridge_port.port_input(&vbridge_port, q->payload, q->tot_len);
            q = next;
        }
    }
    // 正常发送到物理网卡
    ethif_link_output(netif, p);
}
```

**关键设计**：
- 劫持物理网卡的发送
- 同时发送到 bridge port 和物理网卡
- 遍历 pbuf 链表处理分散的数据

### 3.2 vbridge_evt_loop — 网桥事件循环

**文件**: `bridge.c:70-96`

```c
static void *vbridge_evt_loop(void *arg)
{
    sel4_cptr ep = (sel4_cptr)arg;
    bridge_port_data_t data;
    sel4_msg_info_t info = { 0 };
    sel4_word badge = 0;

    while (1) {
        // 等待 hypervisor 消息
        info = seL4_Recv(ep, &badge);
        label = sel4_msg_info_get_label(info);

        switch (label) {
            case BRIDGE_PORT_OUTPUT:
                // 从共享内存读取数据
                idx = sel4_get_mr(0);
                memcpy(&data, port_dsm.va + idx * BRIDGE_SHM_SIZE,
                       sizeof(bridge_port_data_t));

                // 发送到网桥
                vbridge_port_output(&vbridge_port, data.payload, data.len);
                break;
        }
    }
}
```

### 3.3 vbridge_port_output — 发送到 VM

**文件**: `bridge.c:52-68`

```c
static int vbridge_port_output(bridge_port_t *port, void *payload, int len)
{
    // 1. 分配 pbuf
    struct pbuf *p = pbuf_alloc(PBUF_RAW, len, PBUF_POOL);
    if (p == NULL) {
        return -1;
    }
    p->next = NULL;
    p->flags = PBUF_FLAG_BRIDGE_OUTPUT;
    p->tot_len = p->len;

    // 2. 复制数据
    memcpy(p->payload, payload, p->tot_len);

    // 3. 发送到物理网卡 (不走 bridge port_input，避免循环)
    ethif_link_output(&vnet_if, p);

    // 4. 回调 rx_callback，让 lwIP 处理
    tcpip_callback(rx_callback, p);

    return 0;
}
```

---

## 4. 与 lwIP bridgeif 的集成

### 4.1 bridgeif_input — 接收处理

**文件**: `bridgeif.c`

```c
static err_t bridgeif_input(struct pbuf *p, struct netif *netif)
{
    // 获取 bridge 和 port 信息
    port = netif_get_client_data(netif, bridgeif_netif_client_id);
    br = port->bridge;

    // 保存接收端口索引
    p->if_idx = netif_get_index(netif);

    // 解析 Ethernet header
    dst = p->payload;
    src = p->payload + 6;

    // 学习源 MAC
    if ((src->addr[0] & 1) == 0) {
        bridgeif_fdb_update_src(br->fdbd, src, port->port_num);
    }

    // 组播/广播处理
    if (dst->addr[0] & 1) {
        dstports = bridgeif_find_dst_ports(br, dst);
        bridgeif_send_to_ports(br, p, dstports);
        if (dstports & (1 << BRIDGEIF_MAX_PORTS)) {
            br->netif->input(p, br->netif);  // 发送到 CPU
        }
        return ERR_OK;
    }

    // 单播处理
    if (bridgeif_is_local_mac(br, dst)) {
        return br->netif->input(p, br->netif);
    }

    // FDB 查找并发送
    dstports = bridgeif_find_dst_ports(br, dst);
    bridgeif_send_to_ports(br, p, dstports);
    pbuf_free(p);
    return ERR_OK;
}
```

### 4.2 bridgeif_send_to_ports — 发送到端口

```c
static void bridgeif_send_to_ports(bridgeif_private_t *br, struct pbuf *p,
                                   bridgeif_portmask_t ports)
{
    int i;
    struct pbuf *q;

    for (i = 0; i < br->num_ports; i++) {
        if (ports & (1 << i)) {
            // 复制 pbuf 并发送到对应端口
            q = pbuf_copy(p);
            br->ports[i]->portif->output(br->ports[i]->portif, q, dst);
        }
    }
}
```

---

## 5. 完整数据流

### 5.1 VM → NSv (RX)

```
VM                              Hypervisor                      NSv
 │                               │                               │
 │  DMA buffer 准备好            │                               │
 │──────────────────────────────►│                               │
 │  seL4 IPC (BRIDGE_PORT_OUTPUT)│                               │
 │                               │                               │
 │                               ▼                               │
 │                         vbridge_evt_loop()                  │
 │                               │                               │
 │                               ▼                               │
 │                         vbridge_port_output()               │
 │                               │                               │
 │                               ├─► ethif_link_output()        │
 │                               │     └─► NIC DMA              │
 │                               │                               │
 │                               ├─► tcpip_callback(rx_callback)│
 │                               │                               │
 │                               ▼                               │
 │                         rx_callback()                        │
 │                               │                               │
 │                               ▼                               │
 │                         vnet_if.input()                      │
 │                               │                               │
 │                               ▼                               │
 │                         ethernet_input()                     │
 │                               │                               │
 │                               ├─► VLAN 处理                  │
 │                               ├─► LWFW ingress_filter        │
 │                               │                               │
 │                               ▼                               │
 │                         ip4_input() / bridgeif_input()     │
```

### 5.2 NSv → VM (TX)

```
App                              NSv                          Hypervisor
 │                               │                               │
 │  send()                       │                               │
 │──────────────────────────────►│                               │
 │                               ▼                               │
 │                         lwIP 协议栈                          │
 │                               │                               │
 │                               ▼                               │
 │                         ethernet_output()                    │
 │                               │                               │
 │                               ▼                               │
 │                         ethif_link_output_overload()         │
 │                               │                               │
 │                               ├─► vbridge_port.port_input()  │
 │                               │     └─► seL4 IPC → Hypervisor│
 │                               │                               │
 │                               └─► ethif_link_output()        │
 │                                     └─► NIC DMA              │
 │                               │                               │
 ▼                               ▼                               ▼
```

---

## 6. 与 IPCIF 的对比

| 特性 | VIRT_BRG (VIRT_BRG_SUPPORT) | IPCIF (VNET_OVER_IPC) |
|------|------------------------------|------------------------|
| **用途** | VM 间桥接 + NSv 通信 | VM → NSv 通信 |
| **通信方向** | 双向 | 单向 (VM → NSv) |
| **协议栈集成** | bridgeif + port_input/output | netif + input() |
| **数据来源** | 物理网卡 + Hypervisor | Hypervisor only |
| **FDB 学习** | 支持 | 不支持 |

---

## 7. 安全性考虑

### 7.1 PBUF_FLAG_BRIDGE_OUTPUT

```c
p->flags = PBUF_FLAG_BRIDGE_OUTPUT;
```

用于标记来自 bridge 的 packet，避免在 `rx_callback` 中重复处理。

### 7.2 端口使能检查

```c
if (vbridge_port.port_input) {
    // 只有端口启用时才发送到 bridge
}
```

---

## 8. 总结

### 8.1 关键设计

1. **TX 劫持**：通过 `ethif_link_output_overload` 同时发送到 bridge 和物理网卡
2. **RX 分发**：通过 `vbridge_evt_loop` 从 hypervisor 接收并注入 lwIP
3. **双路并行**：bridge 转发和协议栈处理并行执行
4. **FDB 学习**：通过 `bridgeif_fdb_update_src` 学习 MAC 地址

### 8.2 数据流

```
TX: ethernet_output → ethif_link_output_overload → ethif_link_output + vbridge_port.port_input

RX: vbridge_evt_loop → vbridge_port_output → ethif_link_output + tcpip_callback → rx_callback → ethernet_input
```

### 8.3 与 Linux 对比

| 特性 | SafeOS VIRT_BRG | Linux Bridge |
|------|-----------------|--------------|
| **集成方式** | lwIP bridgeif + port_input | 纯内核 bridgeif |
| **hypervisor 通信** | seL4 IPC | vswitch/ovs |
| **FDB 学习** | 支持 | 支持 |
| **VLAN 感知** | 不支持 | 支持 |
