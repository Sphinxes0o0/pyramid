# 路由表分析 — T-022

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: 路由表结构、路由查找 (ip4_route)、默认网关、multicast routing

---

## 1. 概述

lwIP 的路由机制与 Linux 完全不同：
- **Linux**: 独立的路由表结构，支持多路由表、策略路由
- **lwIP**: 无独立路由表，直接在 netif 链表中查找

### 1.1 路由决策流程

```
ip4_output(src, dest)
    │
    ▼
ip4_route(dest) ─────────────────────────────────────┐
    │                                                 │
    ▼                                                 │
遍历 netif_list 链表                                  │
    │                                                 │
    ▼                                                 │
netif_is_up() && netif_is_link_up() && has_ip_addr?  │
    │                                                 │
    ├─► YES: 检查 netmask 匹配                        │
    │       ip4_addr_netcmp(dest, netif_ip, mask)     │
    │           │                                     │
    │           ├─► YES: 返回此 netif                 │
    │           │                                     │
    │           └─► NO: 继续遍历                      │
    │                                                 │
    └─► NO: 继续遍历                                   │
                                                         │
    ▼ (无匹配)                                         │
检查默认网关 (netif_default) ◄─────────────────────────┘
    │
    └─► 返回 netif_default (如果可用)
```

---

## 2. 核心函数

### 2.1 ip4_route — 路由查找

**文件**: `core/ipv4/ip4.c:166-242`

```c
struct netif *
ip4_route(const ip4_addr_t *dest)
{
#if !LWIP_SINGLE_NETIF
  struct netif *netif;

  LWIP_ASSERT_CORE_LOCKED();

  // ============================================
  // Step 1: Multicast 特殊处理
  // ============================================
#if LWIP_MULTICAST_TX_OPTIONS
  if (ip4_addr_ismulticast(dest) && ip4_default_multicast_netif) {
    return ip4_default_multicast_netif;
  }
#endif

  // ============================================
  // Step 2: 遍历 netif_list 链表
  // ============================================
  NETIF_FOREACH(netif) {
    // 检查 netif 是否可用
    if (netif_is_up(netif) &&
        netif_is_link_up(netif) &&
        !ip4_addr_isany_val(*netif_ip4_addr(netif))) {

      // 检查目的地址是否在本地网络
      if (ip4_addr_netcmp(dest, netif_ip4_addr(netif), netif_ip4_netmask(netif))) {
        return netif;  // 直接交付
      }

      // 检查是否发往网关 (point-to-point 接口)
      if (((netif->flags & NETIF_FLAG_BROADCAST) == 0) &&
          ip4_addr_cmp(dest, netif_ip4_gw(netif))) {
        return netif;
      }
    }
  }

  // ============================================
  // Step 3: Loopback 处理
  // ============================================
#if LWIP_NETIF_LOOPBACK && !LWIP_HAVE_LOOPIF
  if (ip4_addr_isloopback(dest)) {
    if (netif_default != NULL && netif_is_up(netif_default)) {
      return netif_default;
    }
    // ...
  }
#endif

  // ============================================
  // Step 4: Hook 扩展点
  // ============================================
#ifdef LWIP_HOOK_IP4_ROUTE
  netif = LWIP_HOOK_IP4_ROUTE(dest);
  if (netif != NULL) {
    return netif;
  }
#endif

  // ============================================
  // Step 5: 使用默认网关
  // ============================================
  if (netif_default == NULL ||
      !netif_is_up(netif_default) ||
      !netif_is_link_up(netif_default) ||
      ip4_addr_isany_val(*netif_ip4_addr(netif_default)) ||
      ip4_addr_isloopback(dest)) {
    // 无路由
    LWIP_DEBUGF(IP_DEBUG, ("ip4_route: No route to %p\n", dest));
    IP_STATS_INC(ip.rterr);
    MIB2_STATS_INC(mib2.ipoutnoroutes);
    return NULL;
  }

  return netif_default;
#endif /* !LWIP_SINGLE_NETIF */
}
```

### 2.2 ip4_route_src — 源地址路由 (可选)

**文件**: `ip4.c:143-154`

```c
#ifdef LWIP_HOOK_IP4_ROUTE_SRC
struct netif *
ip4_route_src(const ip4_addr_t *src, const ip4_addr_t *dest)
{
  if (src != NULL) {
    // 调用 hook 进行源地址路由
    struct netif *netif = LWIP_HOOK_IP4_ROUTE_SRC(src, dest);
    if (netif != NULL) {
      return netif;
    }
  }
  // fallback 到普通路由
  return ip4_route(dest);
}
#endif
```

---

## 3. 网络匹配算法

### 3.1 ip4_addr_netcmp — 网络比较

**文件**: `include/lwip/ip4_addr.h:141`

```c
#define ip4_addr_netcmp(addr1, addr2, mask) (((addr1)->addr & \
                                               (mask)->addr) == \
                                              ((addr2)->addr & \
                                               (mask)->addr))
```

### 3.2 匹配示例

```
假设:
  netif IP:     192.168.1.100
  netmask:      255.255.255.0  (即 /24)
  dest:         192.168.1.50

计算:
  (dest.addr & mask.addr) = (192.168.1.50 & 255.255.255.0) = 192.168.1.0
  (netif.addr & mask.addr) = (192.168.1.100 & 255.255.255.0) = 192.168.1.0

结果: 192.168.1.0 == 192.168.1.0 → 匹配成功!
```

### 3.3 默认路由匹配

```
假设:
  netif IP:     192.168.1.100
  netmask:      0.0.0.0  (即 /0，默认路由)
  dest:         8.8.8.8

计算:
  (dest.addr & mask.addr) = (8.8.8.8 & 0.0.0.0) = 0.0.0.0
  (netif.addr & mask.addr) = (192.168.1.100 & 0.0.0.0) = 0.0.0.0

结果: 0.0.0.0 == 0.0.0.0 → 匹配成功! (默认路由)
```

---

## 4. 与 Linux 的对比

### 4.1 Linux 路由表

```bash
$ ip route
default via 192.168.1.1 dev eth0 proto dhcp
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.100
10.0.0.0/8 via 192.168.1.254 dev eth0
```

Linux 支持：
- 多路由表 (rule-based routing)
- 策略路由 (source/destination based)
- 路由缓存 (历史遗留)
- Multipath routing

### 4.2 lwIP 路由机制

```
无独立路由表！
路由信息直接编码在 netif 结构中：

  netif->ip_addr   → 本地 IP
  netif->netmask   → 网络掩码
  netif->gw        → 默认网关

路由查找 = 遍历 netif_list + netmask 匹配
```

### 4.3 关键区别

| 特性 | Linux | lwIP |
|------|-------|------|
| **路由表结构** | 独立路由表 | 无 (集成在 netif) |
| **查找算法** | 哈希 + trie | 线性遍历 |
| **多路由表** | 支持 | 不支持 |
| **策略路由** | 支持 | 不支持 |
| **默认路由** | `default` 条目 | `netif->netmask = 0` |
| **网关路由** | `via IP` | `netif->gw` |

---

## 5. lwIP 路由的简化设计

### 5.1 为什么简化？

lwIP 设计目标是**嵌入式微内核环境**：
1. **资源受限**: 不需要复杂路由
2. **单宿主**: 通常只有一个网络接口
3. **Flat 网络**: 不需要复杂的多路由表
4. **性能优先**: 简单遍历 O(n) 足够 (n 通常 ≤ 5)

### 5.2 netif->gw 的实际用途

```c
// netif->gw 仅在以下情况使用:
if (ip4_addr_netcmp(dest, netif_ip4_addr(netif), netif_ip4_netmask(netif))) {
    // 目的在本地网络，直接 ARP
    return netif->output(netif, p, dest);
} else {
    // 目的不在本地网络，发往网关
    return netif->output(netif, p, netif_ip4_gw(netif));
}
```

实际上，**网关地址在 `etharp_output()` 中解析为 MAC 地址**，路由决策已经完成。

---

## 6. Multicast 路由

### 6.1 多播组配置

```c
// 设置默认多播 netif
#if LWIP_MULTICAST_TX_OPTIONS
void ip4_set_default_multicast_netif(struct netif *netif);
#endif
```

### 6.2 多播发送流程

```c
// ip4_route() 中
if (ip4_addr_ismulticast(dest) && ip4_default_multicast_netif) {
    return ip4_default_multicast_netif;  // 使用指定的 netif
}
```

### 6.3 IGMP 加入/离开

```c
// 加入多播组
igmp_joingroup(local_ip, multicast_ip);

// 离开多播组
igmp_leavegroup(local_ip, multicast_ip);
```

---

## 7. 路由 hook 扩展点

### 7.1 LWIP_HOOK_IP4_ROUTE

```c
// 在 ip4_route() 找不到时调用
#ifdef LWIP_HOOK_IP4_ROUTE
struct netif *my_hook(const ip4_addr_t *dest) {
    // 自定义路由逻辑
    if (some_condition) {
        return &my_netif;
    }
    return NULL;  // 使用默认
}
#endif
```

### 7.2 LWIP_HOOK_IP4_ROUTE_SRC

```c
// 源地址路由 (用于多宿主场景)
#ifdef LWIP_HOOK_IP4_ROUTE_SRC
struct netif *my_src_route(const ip4_addr_t *src, const ip4_addr_t *dest) {
    // 根据源地址选择 netif
    if (ip4_addr_cmp(src, &ip_192_168_1_100)) {
        return &netif0;
    }
    return &netif1;
}
#endif
```

### 7.3 SafeOS 中的使用

SafeOS 使用 `LWIP_HOOK_IP4_ROUTE` 实现**VLAN 感知的路由决策**：

```c
#ifdef LWIP_HOOK_IP4_ROUTE
struct netif *safeos_ip4_route(const ip4_addr_t *dest) {
    // 1. 先检查是否是 VLAN netif
    // 2. 根据 VID 选择正确的 netif
    // 3. 否则使用默认路由
}
#endif
```

---

## 8. ip4_frag — 分片处理

**文件**: `ip4.c:1165-1170`

```c
#if IP_FRAG
/* don't fragment if interface has mtu set to 0 [loopif] */
if (netif->mtu && (p->tot_len > netif->mtu)) {
    return ip4_frag(p, netif, dest);
}
#endif
```

### 8.1 分片触发条件

```
p->tot_len > netif->mtu
```

### 8.2 分片流程

```
ip4_frag(p, netif, dest)
    │
    ├─► 将大数据包分片为 MTU 大小
    │
    ├─► 每个分片:
    │     ├─► 设置 IP Header: ID, flags, fragment offset
    │     └─► 发送分片
    │
    └─► 返回
```

---

## 9. 总结

### 9.1 路由查找核心

```
ip4_route(dest)
    │
    ├─► Multicast? → 使用 ip4_default_multicast_netif
    │
    ├─► NETIF_FOREACH 遍历 netif_list
    │     │
    │     ├─► netif_is_up() && netif_is_link_up() && has_ip?
    │     │
    │     ├─► dest & netmask == netif_ip & netmask?
    │     │     └─► YES: 返回此 netif (直接交付)
    │     │
    │     └─► 发往网关? → 返回此 netif
    │
    ├─► Hook: LWIP_HOOK_IP4_ROUTE
    │
    └─► 返回 netif_default (默认网关)
```

### 9.2 关键设计

1. **无独立路由表**: 路由信息在 netif 结构中
2. **线性遍历**: O(n) 复杂度，n 通常很小
3. **netmask 匹配**: `ip4_addr_netcmp()` 进行网络比较
4. **默认路由**: `netmask = 0` 表示默认路由
5. **网关解析**: 在 `etharp_output()` 中完成

### 9.3 与 Linux 的差异

```
Linux: 路由表 → fib_table_lookup() → trie/哈希
lwIP:  netif_list → 线性遍历 → netmask 匹配

lwIP 适合: 单宿主、简单网络、嵌入式
Linux 适合: 多宿主、复杂网络、服务器
```

### 9.4 SafeOS 特供

1. **VLAN 路由**: 通过 `LWIP_HOOK_IP4_ROUTE` 实现
2. **多 netif**: vnet_if + vlan_if[i] 分开路由
3. **LWFW 集成**: egress_filter 在路由决策之后执行
