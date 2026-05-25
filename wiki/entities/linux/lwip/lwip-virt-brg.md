---
type: entity
tags: [linux, lwip, network, virtual-bridge, hypervisor, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# VIRT_BRG_SUPPORT — Virtual Bridge Hypervisor Integration

## 定义

VIRT_BRG_SUPPORT 是 SafeOS 的虚拟网桥支持，允许 NSv 通过 hypervisor 与虚拟机进行网络通信。作为 bridge port 集成到 lwIP bridgeif，支持 VM ↔ VM 和 VM ↔ NSv 的数据包转发。

## 核心数据结构

### bridge_port_t
```c
typedef struct bridge_port_s {
    char name[BRIDGE_NAME_LEN];
    uint8_t enabled;
    struct dsm_info port_dsm;           // 共享内存信息
    void (*port_input)(...);           // 输入回调
    void (*port_output)(...);          // 输出回调
} bridge_port_t;
```

### bridge_port_data_t
```c
typedef struct {
    uint32_t len;
    uint8_t payload[BRIDGE_SHM_SIZE];
} bridge_port_data_t;
```

## ethif_link_output_overload — TX 劫持

当 VIRT_BRG_SUPPORT 启用时，物理网卡的 `linkoutput` 被替换为：

```c
err_t ethif_link_output_overload(struct netif *netif, struct pbuf *p)
{
    if (vbridge_port.port_input) {
        struct pbuf *q = p;
        while (q != NULL) {
            q->tot_len = q->len;
            vbridge_port.port_input(&vbridge_port, q->payload, q->tot_len);
            q = q->next;
        }
    }
    // 正常发送到物理网卡
    ethif_link_output(netif, p);
}
```

**关键**: 劫持物理网卡的发送，同时发送到 bridge port 和物理网卡。

## vbridge_evt_loop — 网桥事件循环

```c
static void *vbridge_evt_loop(void *arg)
{
    while (1) {
        info = seL4_Recv(ep, &badge);
        switch (label) {
            case BRIDGE_PORT_OUTPUT:
                // 从共享内存读取数据
                idx = sel4_get_mr(0);
                memcpy(&data, port_dsm.va + idx * BRIDGE_SHM_SIZE, ...);
                vbridge_port_output(&vbridge_port, data.payload, data.len);
                break;
        }
    }
}
```

## vbridge_port_output — 发送到 VM

```c
static int vbridge_port_output(bridge_port_t *port, void *payload, int len)
{
    struct pbuf *p = pbuf_alloc(PBUF_RAW, len, PBUF_POOL);
    p->flags = PBUF_FLAG_BRIDGE_OUTPUT;  // 避免循环
    memcpy(p->payload, payload, p->tot_len);

    ethif_link_output(&vnet_if, p);     // 发送到物理网卡
    tcpip_callback(rx_callback, p);       // 注入到 RX 路径
    return 0;
}
```

## VM → NSv 数据流

```
VM                              Hypervisor                      NSv
 │                               │                               │
 │  DMA buffer 准备好            │                               │
 │──────────────────────────────►│                               │
 │  seL4 IPC (BRIDGE_PORT_OUTPUT)│                               │
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
 │                               ▼                               │
 │                         rx_callback()                        │
 │                               ▼                               │
 │                         vnet_if.input()                      │
```

## 与 IPCIF 对比

| 特性 | VIRT_BRG (VIRT_BRG_SUPPORT) | IPCIF (VNET_OVER_IPC) |
|------|------------------------------|------------------------|
| 用途 | VM 间桥接 + NSv 通信 | VM → NSv 通信 |
| 通信方向 | 双向 (VM ↔ VM, VM ↔ NSv) | 单向 (VM → NSv) |
| 协议栈集成 | bridgeif + port_input/output | netif + input() |
| 数据来源 | 物理网卡 + Hypervisor | Hypervisor only |
| FDB 学习 | 支持 | 不支持 |

## 相关概念

- [[entities/linux/lwip/lwip-bridgeif]] — 802.1D bridge 实现
- [[entities/linux/lwip/lwip-ipcif]] — VNET_OVER_IPC
- [[entities/linux/lwip/lwip-sel4-interaction]] — 整体交互架构

## 来源详情

- [[sources/safeos-lwip-extensions]]
