# ip4_output 分析 — T-021

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: ip4_output 函数：IP header 构造、路由查找、checksum 计算、LWFW egress filter

---

## 1. 概述

`ip4_output()` 是 lwIP 的 **IP 层出口函数**，负责：
1. 通过路由查找选择合适的 netif
2. 构造 IP Header
3. 计算 IP checksum
4. 调用 netif->output 发送到下一层

### 1.1 调用链

```
TCP/UDP output
    │
    ▼
ip4_output(p, src, dest, ttl, tos, proto)
    │
    ├─► ip4_route_src(src, dest) — 路由查找，找 netif
    │
    ▼
ip4_output_if(p, src, dest, ttl, tos, proto, netif)
    │
    ├─► [LWIP_HOOK_IP4_OUTPUT] LWFW egress filter
    ├─► 添加 IP Header
    ├─► 计算 IP Checksum
    ├─► [IP_FRAG] 分片处理
    └─► netif->output(netif, p, dest)
          └─► etharp_output() → ethernet_output()
```

---

## 2. 函数源码分析

**文件**: `external/lwip_ds_mcu/src/core/ipv4/ip4.c:888-1174`

### 2.1 ip4_output — 路由查找入口

```c
// ip4.c:1193-1219
err_t
ip4_output(struct pbuf *p, const ip4_addr_t *src, const ip4_addr_t *dest,
           u8_t ttl, u8_t tos, u8_t proto)
{
    struct netif *netif;

    // Step 1: 路由查找 — 找到输出 netif
    if ((netif = ip4_route_src(src, dest)) == NULL) {
        LWIP_DEBUGF(IP_DEBUG, ("ip4_output: No route to %"U16_F".%"U16_F".%"U16_F".%"U16_F"\n",
                               ip4_addr1_16(dest), ip4_addr2_16(dest),
                               ip4_addr3_16(dest), ip4_addr4_16(dest)));
        IP_STATS_INC(ip.rterr);
        return ERR_RTE;  // 无路由
    }

    // Step 2: 调用 ip4_output_if
    return ip4_output_if(p, src, dest, ttl, tos, proto, netif);
}
```

### 2.2 ip4_route_src — 路由选择

**文件**: `ip4.c:142-154`

```c
struct netif *
ip4_route_src(const ip4_addr_t *src, const ip4_addr_t *dest)
{
    if (src != NULL) {
        // 有源地址，使用源地址路由
        struct netif *netif = LWIP_HOOK_IP4_ROUTE_SRC(src, dest);
        if (netif != NULL) {
            return netif;
        }
    }
    return ip4_route(dest);  // fallback 到目的地址路由
}
```

### 2.3 ip4_route — 目的地址路由

**文件**: `ip4.c:165-242`

```c
struct netif *
ip4_route(const ip4_addr_t *dest)
{
    struct netif *netif;

    // ============================================
    // Step 1: 多播处理
    // ============================================
    #if LWIP_MULTICAST_TX_OPTIONS
    if (ip4_addr_ismulticast(dest) && ip4_default_multicast_netif) {
        return ip4_default_multicast_netif;
    }
    #endif

    // ============================================
    // Step 2: 遍历 netif_list 找匹配
    // ============================================
    NETIF_FOREACH(netif) {
        if (netif_is_up(netif) && netif_is_link_up(netif) &&
            !ip4_addr_isany_val(*netif_ip4_addr(netif))) {

            // 检查目的地址是否在 netif 子网内
            if (ip4_addr_netcmp(dest, netif_ip4_addr(netif), netif_ip4_netmask(netif))) {
                return netif;  // 直接交付
            }

            // 检查是否点对点链路 (peer 地址匹配)
            if (((netif->flags & NETIF_FLAG_BROADCAST) == 0) &&
                ip4_addr_cmp(dest, netif_ip4_gw(netif))) {
                return netif;
            }
        }
    }

    // ============================================
    // Step 3: Hook 扩展
    // ============================================
    #ifdef LWIP_HOOK_IP4_ROUTE_SRC
    netif = LWIP_HOOK_IP4_ROUTE_SRC(NULL, dest);
    if (netif != NULL) return netif;
    #elif defined(LWIP_HOOK_IP4_ROUTE)
    netif = LWIP_HOOK_IP4_ROUTE(dest);
    if (netif != NULL) return netif;
    #endif

    // ============================================
    // Step 4: 使用 default netif
    // ============================================
    if ((netif_default == NULL) || !netif_is_up(netif_default) ||
        !netif_is_link_up(netif_default) || ip4_addr_isloopback(dest)) {
        LWIP_DEBUGF(IP_DEBUG | LWIP_DBG_LEVEL_SERIOUS, ("ip4_route: No route to %"U16_F".%"U16_F".%"U16_F".%"U16_F"\n", ...));
        IP_STATS_INC(ip.rterr);
        MIB2_STATS_INC(mib2.ipoutnoroutes);
        return NULL;
    }

    return netif_default;  // 通过默认网关发送
}
```

### 2.4 ip4_output_if — IP Header 构造

**文件**: `ip4.c:940-1174`

```c
err_t
ip4_output_if_opt_src(struct pbuf *p, const ip4_addr_t *src, const ip4_addr_t *dest,
                      u8_t ttl, u8_t tos, u8_t proto, struct netif *netif,
                      void *ip_options, u16_t optlen)
{
    struct ip_hdr *iphdr;

    // ============================================
    // Step 1: 添加 IP Header 空间
    // ============================================
    if (dest != LWIP_IP_HDRINCL) {
        if (pbuf_add_header(p, IP_HLEN)) {
            // 空间不足
            return ERR_BUF;
        }

        iphdr = (struct ip_hdr *)p->payload;

        // ============================================
        // Step 2: 填充 IP Header 字段
        // ============================================
        IPH_VHL_SET(iphdr, 4, IP_HLEN / 4);     // Version=4, HeaderLen=5
        IPH_TOS_SET(iphdr, tos);                 // Type of Service
        IPH_LEN_SET(iphdr, lwip_htons(p->tot_len));  // Total Length
        IPH_OFFSET_SET(iphdr, 0);                // Fragment Offset
        IPH_ID_SET(iphdr, lwip_htons(ip_id));   // Identification
        ++ip_id;                                 // 递增 IP ID
        IPH_TTL_SET(iphdr, ttl);                // Time to Live
        IPH_PROTO_SET(iphdr, proto);             // Protocol (TCP/UDP/ICMP)

        // 源/目的地址
        ip4_addr_copy(iphdr->dest, *dest);
        if (src == NULL) {
            ip4_addr_copy(iphdr->src, *IP4_ADDR_ANY4);
        } else {
            ip4_addr_copy(iphdr->src, *src);
        }

        // ============================================
        // Step 3: 计算 IP Checksum
        // ============================================
        #if CHECKSUM_GEN_IP_INLINE
        // 快速 inline checksum 计算
        chk_sum = ...;
        iphdr->_chksum = (u16_t)chk_sum;
        #else
        IPH_CHKSUM_SET(iphdr, 0);
        #if CHECKSUM_GEN_IP
        IF__NETIF_CHECKSUM_ENABLED(netif, NETIF_CHECKSUM_GEN_IP) {
            IPH_CHKSUM_SET(iphdr, inet_chksum(iphdr, ip_hlen));
        }
        #endif
        #endif
    }

    // ============================================
    // Step 4: LWFW Egress Filter (SafeOS 特供)
    // ============================================
    #ifdef NIO_LWIP_LWFW
    if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
        err_t ret = lwfw_p->ops->egress_filter(p, netif);
        if (ret != ERR_OK) {
            IP_STATS_INC(ip.drop);
            return ERR_FW;  // 被防火墙丢弃
        }
    }
    #endif

    // ============================================
    // Step 5: 分片处理 (如果 packet > MTU)
    // ============================================
    #if IP_FRAG
    if (netif->mtu && (p->tot_len > netif->mtu)) {
        return ip4_frag(p, netif, dest);
    }
    #endif

    // ============================================
    // Step 6: 发送到下层 (ethernet_output)
    // ============================================
    return netif->output(netif, p, dest);
}
```

---

## 3. IP Header 结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 0                   1                   2                   3              │
│ 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1            │
├─────────────────────────────────────────────────────────────────────────────┤
│ Version(4) │  IHL(4)  │    TOS(8)    │         Total Length(16)            │
├─────────────────────────────────────────────────────────────────────────────┤
│        Identification(16)        │ Flags(3) │    Fragment Offset(13)       │
├─────────────────────────────────────────────────────────────────────────────┤
│      TTL(8)      │   Protocol(8)   │        Header Checksum(16)           │
├─────────────────────────────────────────────────────────────────────────────┤
│                         Source IP Address(32)                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                       Destination IP Address(32)                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. LWFW Egress Filter Hook

### 4.1 Hook 位置

```c
// ip4_output_if 中，IP header 构造之后，netif->output 之前
#ifdef NIO_LWIP_LWFW
if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
    err_t ret = lwfw_p->ops->egress_filter(p, netif);
    if (ret != ERR_OK) {
        IP_STATS_INC(ip.drop);
        return ERR_FW;  // ← 包被丢弃，不进入网络
    }
}
#endif
```

### 4.2 与 Ingress Filter 的对比

| 特性 | Ingress (ip4_input) | Egress (ip4_output) |
|------|---------------------|---------------------|
| **Hook 位置** | ip4_input:507 | ip4_output_if:1096 |
| **Hook 宏** | `LWIP_HOOK_IP4_INPUT` | 无 (直接调用 `lwfw_p->ops->egress_filter`) |
| **方向** | 入方向 (刚收到包) | 出方向 (准备发送前) |
| **使用 netif** | `inp` (接收接口) | `netif` (发送接口) |

---

## 5. 与其他模块的关系

### 5.1 上游调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **tcp_output** | `ip4_output()` | TCP 发送 |
| **udp_output** | `ip4_output()` | UDP 发送 |
| **icmp_output** | `ip4_output()` | ICMP 发送 |
| **ip4_forward** | `netif->output()` | 转发 |

### 5.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **ip4_route** | 路由查找 | 找到输出 netif |
| **ip4_frag** | 分片处理 | packet > MTU 时 |
| **etharp_output** | netif->output | Ethernet 输出 |

### 5.3 Hook 点

| Hook | 位置 | 说明 |
|------|------|------|
| `LWIP_HOOK_IP4_ROUTE_SRC` | ip4_route_src:147 | 源地址路由扩展 |
| `LWIP_HOOK_IP4_ROUTE` | ip4_route:223 | 目的地址路由扩展 |
| `LWIP_HOOK_IP4_OUTPUT` | ip4_output_if:1172 | Egress filter (LWFW) |

---

## 6. 性能特征

### 6.1 路由查找复杂度

```
O(n) — 遍历 netif_list 直到找到匹配的子网
```

### 6.2 瓶颈分析

```
ip4_output 瓶颈:
1. ip4_route() 线性遍历 netif_list — O(n)
2. 单个 netif 输出 — 无并行化
3. IP checksum 计算 — O(IP_HLEN) = 20 bytes
```

### 6.3 与 Linux 对比

| 特性 | SafeOS lwIP | Linux |
|------|-------------|-------|
| **路由表** | netif_list 遍历 | 哈希/红黑树 |
| **多路径** | 不支持 | 支持 (RPF) |
| **路由缓存** | 无 | 有 (rt_cache) |

---

## 7. 总结

### 7.1 ip4_output 的核心作用

```
发送 IP packet
    │
    ├─► ip4_route_src() — 路由查找，找 netif
    │     - 匹配子网 → 直接交付
    │     - 匹配 gateway → 通过默认网关
    │     - default netif → 最后 fallback
    │
    ├─► ip4_output_if() — 封装
    │     - 添加 IP Header (20 bytes)
    │     - 计算 IP Checksum
    │     - [可选] LWFW Egress Filter
    │     - [可选] 分片处理
    │
    └─► netif->output() — 发送到下层
          └─► etharp_output() → ethernet_output()
```

### 7.2 关键设计

1. **无路由缓存**: 每次 output 都重新遍历 netif_list
2. **IP ID 递增**: 每个 packet 递增，唯一标识分片
3. **Checksum offload**: 如果 netif 支持，硬件计算 checksum
4. **LWFW egress 在 IP 层**: 在分片之前过滤，符合 RFC 1812

### 7.3 SafeOS 特供

1. **LWFW egress_filter**: 在 ip4_output_if 中直接调用，而非 hook
2. **LWFW_TEST_LATENCY**: 性能测量代码存在于 ip4_output_if 中
