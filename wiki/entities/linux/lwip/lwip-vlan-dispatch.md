---
type: entity
tags: [linux, lwip, network, vlan]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP VLAN Dispatch Analysis

## 定义

VLAN packet 分发机制：当一个 VLAN-tagged packet 到达物理网卡时，lwIP 如何分发到正确的 VLAN interface。SafeOS 使用 `LWIP_ARP_FILTER_NETIF=1` 模式。

## 核心架构差异

| 维度 | lwIP (SafeOS) | Linux |
|------|---------------|-------|
| **VLAN netif 本质** | 独立的 `struct netif`，拥有独立 IP | 虚拟 net_device，依赖物理设备 |
| **VLAN 识别方式** | IP 地址匹配 + 可选 VLAN ID 检查 | VLAN net_device 堆叠在物理设备上 |
| **Packet 分发** | `ethernet_input` → `ip4_input` 遍历所有 netif | VLAN net_device 注册 `.real_dev` 接收回调 |

## RX 方向 — Packet 分发流程

```
物理网卡 DMA
    │
    ▼
rx_callback() → vnet_if.input(p, &vnet_if) = ethernet_input()
    │
    ├─► 解析 Ethernet Header
    ├─► if (ETHTYPE_VLAN) 解析 VLAN Header
    │
    ▼
LWIP_ARP_FILTER_NETIF_FN(p, netif, type)
    │
    ├─► ETHTYPE_VLAN: 通过 netif->vlanid == packet VID 匹配
    ├─► ETHTYPE_IP:   通过 IP 地址 + vlanid==0 匹配
    └─► 返回正确的 netif (vnet_if 或 vlan_if[i])
    │
    ▼
ip4_input(p, netif)  // netif 已是正确的 vlan_if[i]
```

## lwip_arp_filter_netif_fn — VLAN 分发核心

```c
struct netif *lwip_arp_filter_netif_fn(void *_p, void *_netifIn, u16_t type) {
    switch (type) {
        // VLAN-tagged Packet: 通过 VLAN ID 匹配
        case ETHTYPE_VLAN: {
            NETIF_FOREACH(netif) {
                if (netif_is_up(netif)) {
                    u16_t vid = netif->vlanid & VLAN_ID_MASK;
                    if (vid == (vlan_hdr->prio_vid & VLAN_ID_MASK)) {
                        return netif;  // 找到匹配的 VLAN netif!
                    }
                }
            }
            return NULL;  // 无匹配，丢弃
        }
        // 非 VLAN Packet: 通过 IP 地址 + vlanid==0 匹配
        case ETHTYPE_IP: {
            ip_addr_copy_from_ip4(dst, iphdr->dest);
            NETIF_FOREACH(netif) {
                if (netif_is_up(netif) &&
                    ip4_addr_cmp(&dst, &netif->ip_addr) &&
                    netif->vlanid == 0u) {
                    return netif;
                }
            }
            break;
        }
    }
    return netifIn;  // 默认返回输入的 netif
}
```

## 分发决策矩阵

| Packet 类型 | 分发条件 | 返回的 netif |
|------------|----------|--------------|
| **VLAN-tagged** (VID=100) | `netif->vlanid == 100` | `vlan_if[100]` |
| **VLAN-tagged** (VID=200) | `netif->vlanid == 200` | `vlan_if[200]` |
| **非 VLAN** (IP=172.20.0.1) | `netif->ip_addr == 172.20.0.1 && netif->vlanid == 0` | `vnet_if` |

## TX 方向 — VLAN Tag 插入

```
App Socket 发送
    │
    ▼
etharp_output() → ethernet_output()
    │
    ▼
LWIP_HOOK_VLAN_SET(netif, p, src, dst, eth_type)
    │
    ├─► netif->vlanid == NO_VLANID? → return -1 (不插入)
    └─► netif->vlanid == 100? → return (pbuf->priority << 13) | 100
    │
    ▼
pbuf_add_header(p, 18)  // 腾出 VLAN tag 空间
    │
    ▼
netif->linkoutput(netif, p)  // = low_level_output
    │
    ▼
physical_netif->linkoutput() → NIC DMA
```

## SafeOS 配置

| 网口类型 | vlanid | TX 行为 |
|----------|--------|---------|
| 物理网口 (vnet_if) | 0 (NO_VLANID) | 不插入 VLAN tag |
| VLAN 网口 (vlan_if[i]) | 配置值 (如 100, 200) | 插入 VLAN tag |

## 相关概念

- [[entities/linux/lwip/lwip-ethernet-input]] — ethernet_input 中的 VLAN 解析
- [[entities/linux/lwip/lwip-ethernet-output]] — ethernet_output 中的 VLAN 插入
- [[entities/linux/lwip/lwip-vlan-hook]] — LWIP_HOOK_VLAN_SET 和 LWIP_HOOK_VLAN_CHECK
- [[entities/linux/lwip/lwip-vlan-dispatch-deep]] — 深度分析：模块、函数、设计哲学
