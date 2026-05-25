---
type: entity
tags: [linux, lwip, network, ip]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP ip4_input Analysis

## 定义

`ip4_input()` 是 lwIP 的 **IP 层入口函数**，负责解析 IP Header、checksum 校验、验证目标地址、LWFW ingress filter、分发到上层协议 (TCP/UDP/RAW/ICMP)。

## 调用链

```
ethernet_input()
    │
    ▼
ip4_input(p, netif)
    │
    ├─► 解析 IP Header (版本、长度、checksum)
    ├─► LWIP_HOOK_IP4_INPUT (LWFW ingress hook)
    ├─► ip4_input_accept() — 验证目标地址
    ├─► NETIF_FOREACH — 遍历 netif 找匹配
    ├─► ip4_forward() — 如果需要转发
    └─► 分发到上层
          - TCP → tcp_input()
          - UDP → udp_input()
          - RAW → raw_input()
          - ICMP → icmp_input()
```

## 核心逻辑

```c
ip4_input(struct pbuf *p, struct netif *inp) {
    iphdr = (struct ip_hdr *)p->payload;

    // Step 1: LWFW Ingress Filter Hook
    #ifdef LWIP_HOOK_IP4_INPUT
    if (LWIP_HOOK_IP4_INPUT(p, inp)) {
        return ERR_OK;  // packet 被 filter 丢弃
    }
    #endif

    // Step 2: 设置 ip_data
    ip_data.current_input_netif = inp;  // ← 关键！

    // Step 3: 验证目标地址
    if (ip4_addr_ismulticast(ip4_current_dest_addr())) {
        // 多播: 检查 inp 是否加入该组
        #if LWIP_IGMP
        if (igmp_lookfor_group(inp, ...)) { netif = inp; }
        #endif
    } else {
        // 单播: 先检查 inp，否则遍历 netif_list
        if (ip4_input_accept(inp)) {
            netif = inp;
        } else {
            NETIF_FOREACH(netif) { ... }
        }
    }

    // Step 4: 分发到上层协议
    switch (IPH_PROTO(iphdr)) {
        case IP_PROTO_TCP: tcp_input(p, netif); break;
        case IP_PROTO_UDP: udp_input(p, netif); break;
        case IP_PROTO_ICMP: icmp_input(p, netif); break;
        default: raw_input(p, netif);
    }
}
```

## ip4_input_accept — 地址验证

```c
ip4_input_accept(struct netif *netif) {
    if ((netif_is_up(netif)) && (!ip4_addr_isany_val(*netif_ip4_addr(netif)))) {
        if (ip4_addr_cmp(ip4_current_dest_addr(), netif_ip4_addr(netif)) ||
            ip4_addr_isbroadcast(ip4_current_dest_addr(), netif) ||
            ip4_addr_get_u32(ip4_current_dest_addr()) == IPADDR_LOOPBACK) {
            return 1;  // 接受
        }
    }
    return 0;  // 拒绝
}
```

## 关键设计点

1. **ip_data.current_input_netif**: 在 ip4_input 中设置，供 UDP/TCP socket 绑定检查使用
2. **NETIF_FOREACH 遍历**: 如果 inp 不匹配，会遍历所有 netif
3. **LWIP_HOOK_IP4_INPUT**: LWFW ingress filter 的 hook 点

## 相关概念

- [[entities/linux/lwip/lwip-ethernet-input]] — 上游调用者
- [[entities/linux/lwip/lwip-tcp-input]] — TCP 分发目标
- [[entities/linux/lwip/lwip-udp-input]] — UDP 分发目标
- [[entities/linux/lwip/lwip-routing]] — 路由查找
- [[entities/linux/lwip/lwip-ip-fragmentation]] — IP 分片重组
