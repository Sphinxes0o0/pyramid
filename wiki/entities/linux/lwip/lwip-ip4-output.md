---
type: entity
tags: [linux, lwip, network, ip]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP ip4_output Analysis

## 定义

`ip4_output()` 是 lwIP 的 **IP 层出口函数**，负责通过路由查找选择合适的 netif、构造 IP Header、计算 IP checksum、LWFW egress filter、调用 netif->output。

## 调用链

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

## IP Header 结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Version(4) │  IHL(4)  │    TOS(8)    │         Total Length(16)           │
├─────────────────────────────────────────────────────────────────────────────┤
│        Identification(16)        │ Flags(3) │    Fragment Offset(13)        │
├─────────────────────────────────────────────────────────────────────────────┤
│      TTL(8)      │   Protocol(8)   │        Header Checksum(16)           │
├─────────────────────────────────────────────────────────────────────────────┤
│                         Source IP Address(32)                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                       Destination IP Address(32)                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

## LWFW Egress Filter Hook

```c
#ifdef NIO_LWIP_LWFW
if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_ENABLE) {
    err_t ret = lwfw_p->ops->egress_filter(p, netif);
    if (ret != ERR_OK) {
        IP_STATS_INC(ip.drop);
        return ERR_FW;  // 被防火墙丢弃
    }
}
#endif
```

## 路由查找 (ip4_route)

```c
struct netif *ip4_route(const ip4_addr_t *dest) {
    // 遍历 netif_list 找匹配
    NETIF_FOREACH(netif) {
        if (netif_is_up(netif) && netif_is_link_up(netif)) {
            // 检查目的地址是否在 netif 子网内
            if (ip4_addr_netcmp(dest, netif_ip4_addr(netif), netif_ip4_netmask(netif))) {
                return netif;  // 直接交付
            }
        }
    }
    // fallback 到 netif_default
    return netif_default;
}
```

## 相关概念

- [[entities/linux/lwip/lwip-ip4-input]] — 对应的 L3 入口
- [[entities/linux/lwip/lwip-routing]] — 路由查找详解
- [[entities/linux/lwip/lwip-ip-fragmentation]] — IP 分片重组
- [[entities/linux/lwip/lwip-tcp-output]] — TCP 发送使用 ip4_output
- [[entities/linux/lwip/lwip-udp-output]] — UDP 发送使用 ip4_output
