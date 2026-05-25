---
type: entity
tags: [linux, lwip, network, routing, ip]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP Routing Analysis

## 定义

lwIP 的路由机制与 Linux 完全不同：**无独立路由表**，直接通过 netif 链表查找。

## 路由决策流程

```
ip4_output(src, dest)
    │
    ▼
ip4_route(dest)
    │
    ▼
遍历 netif_list 链表
    │
    ▼
netif_is_up() && netif_is_link_up() && has_ip_addr?
    │
    ├─► YES: 检查 netmask 匹配
    │       ip4_addr_netcmp(dest, netif_ip, mask)
    │           │
    │           ├─► YES: 返回此 netif
    │           └─► NO: 继续遍历
    │
    └─► NO: 继续遍历
    │
    ▼ (无匹配)
检查默认网关 (netif_default)
    │
    └─► 返回 netif_default (如果可用)
```

## 核心函数

### ip4_route

```c
struct netif *ip4_route(const ip4_addr_t *dest) {
    NETIF_FOREACH(netif) {
        if (netif_is_up(netif) && netif_is_link_up(netif)) {
            // 检查目的地址是否在本地网络
            if (ip4_addr_netcmp(dest, netif_ip4_addr(netif), netif_ip4_netmask(netif))) {
                return netif;  // 直接交付
            }
        }
    }
    return netif_default;  // 默认网关
}
```

### 网络匹配算法

```c
// ip4_addr_netcmp — 网络比较
// (dest.addr & mask.addr) == (netif.addr & mask.addr)

假设:
  netif IP:   192.168.1.100
  netmask:    255.255.255.0  (/24)
  dest:       192.168.1.50

计算:
  (dest.addr & mask.addr) = 192.168.1.0
  (netif.addr & mask.addr) = 192.168.1.0

结果: 匹配成功!
```

## lwIP vs Linux 路由对比

| 特性 | Linux | lwIP |
|------|-------|------|
| **路由表结构** | 独立路由表 (哈希/trie) | 无 (集成在 netif) |
| **查找算法** | 哈希 + trie | 线性遍历 O(n) |
| **多路由表** | 支持 | 不支持 |
| **默认路由** | `default` 条目 | `netif->netmask = 0` |
| **网关路由** | `via IP` | `netif->gw` |

## 为什么简化？

lwIP 设计目标是嵌入式微内核环境：
1. **资源受限**: 不需要复杂路由
2. **单宿主**: 通常只有一个网络接口
3. **Flat 网络**: 不需要复杂的多路由表
4. **性能优先**: O(n) 遍历，n 通常 ≤ 5

## 相关概念

- [[entities/linux/lwip/lwip-ip4-output]] — 使用路由查找
- [[entities/linux/lwip/lwip-ip4-input]] — 目的地址验证
- [[entities/linux/lwip/lwip-netif]] — netif 结构中的路由信息
- [[entities/linux/lwip/lwip-vlan-implementation]] — SafeOS VLAN 路由
