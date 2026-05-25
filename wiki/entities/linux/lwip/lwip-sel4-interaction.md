---
type: entity
tags: [linux, lwip, network, sel4, safeos, integration]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# lwIP-seL4 Interaction — Physical NIC/VLAN/Hypervisor Integration

## 定义

lwIP 与 seL4 的交互涵盖三大层面：**物理网卡交互** (NIC Driver via seL4 IPC + CMA)、**VLAN 分发** (VID → netif 映射)、**Hypervisor 网桥** (VIRT_BRG/VNET_OVER_IPC)。

## 整体架构

```
App 进程
    │
    ├─ socket() / sendto() / recvfrom()
    └─ seL4 IPC (badge = pid)
           │
           ▼
┌─────────────────────────────────────────────────────────────┐
│  NSv (网络服务器进程)                                     │
│  ├─ event_loop() (App 请求处理)                          │
│  ├─ nic_rx_thread() (RX 包处理)                           │
│  ├─ tcpip_thread() (lwIP 协议栈)                        │
│  ├─ vnet_if (物理网卡 netif)                             │
│  ├─ vlan_if[i] (VLAN 网卡 netif)                         │
│  └─ CMA (96MB) + elem_ring x4                           │
└─────────────────────────────────────────────────────────────┘
    │ seL4 IPC + CMA PA
┌─────────────────────────────────────────────────────────────┐
│  NIC 驱动进程                                              │
│  nic_tx_ntfn / nic_rx_ntfn (seL4 Notification)            │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Hypervisor / VMM 层                                       │
│  #ifdef VIRT_BRG_SUPPORT: ethif_link_output_overload      │
│  #ifdef VNET_OVER_IPC_SUPPORT: ipcif_evt_loop            │
└─────────────────────────────────────────────────────────────┘
```

## VLAN netif 的特殊路径

| 步骤 | 物理网卡 (vnet_if) | VLAN 网卡 (vlan_if[i]) |
|------|---------------------|--------------------------|
| `netif->input` | `ethernet_input()` | `tcpip_input()` |
| `netif->linkoutput` | `ethif_link_output()` | `low_level_output()` |
| `netif->vlanid` | 0 (NO_VLANID) | 配置的 VID |
| `LWIP_HOOK_VLAN_SET` | 不插入 VLAN Tag | 插入 VLAN Tag |
| `LWIP_HOOK_VLAN_CHECK` | N/A | 检查 VID 匹配 |

## tcpip_thread — LWIP_TCPIP_CORE_LOCKING

```c
// tcpip_thread:136
LOCK_TCPIP_CORE();
while (1) {
  TCPIP_MBOX_FETCH(&tcpip_mbox, &msg);
  tcpip_thread_handle_msg(msg);
}

// nic_rx_thread 中:
LOCK_TCPIP_CORE();
rx_callback((struct pbuf *)va);
UNLOCK_TCPIP_CORE();
```

## 相关概念

- [[entities/linux/lwip/lwip-sel4-function]] — 函数级分析
- [[entities/linux/lwip/lwip-sel4-ipc]] — seL4 IPC 机制
- [[entities/linux/lwip/lwip-sel4-performance-boundary]] — 性能边界
- [[entities/linux/lwip/lwip-arp-filter-netif-fn]] — VLAN 分发
- [[entities/linux/lwip/lwip-virt-brg]] — VIRT_BRG 支持
- [[entities/linux/lwip/lwip-ipcif]] — VNET_OVER_IPC

## 来源详情

- [[sources/safeos-lwip-extensions]]
