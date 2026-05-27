# ip4_input 分析 — T-020

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: ip4_input 函数：IP header 解析、checksum 校验、netif 选择、socket 匹配

---

## 1. 概述

`ip4_input()` 是 lwIP 的 **IP 层入口函数**，负责：
1. 解析 IP Header
2. checksum 校验
3. 验证目标地址是否属于本机
4. 分发到上层协议 (TCP/UDP/RAW/ICMP)

### 1.1 调用链

```
ethernet_input()
    │
    ▼
ip4_input(p, netif)
    │
    ├─► 解析 IP Header
    ├─► checksum 校验
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

---

## 2. 函数源码分析

**文件**: `external/lwip_ds_mcu/src/core/ipv4/ip4.c:468`

```c
ip4_input(struct pbuf *p, struct netif *inp)
{
    const struct ip_hdr *iphdr;
    struct netif *netif;
    u16_t iphdr_hlen;
    u16_t iphdr_len;

    LWIP_ASSERT_CORE_LOCKED();
    IP_STATS_INC(ip.recv);

    // ============================================
    // Step 1: 解析 IP Header
    // ============================================
    iphdr = (struct ip_hdr *)p->payload;

    // 验证 IP 版本
    if (IPH_V(iphdr) != 4) {
        pbuf_free(p);
        return ERR_OK;
    }

    iphdr_hlen = IPH_HL_BYTES(iphdr);  // IP header 长度
    iphdr_len = lwip_ntohs(IPH_LEN(iphdr));  // IP packet 总长度

    // ============================================
    // Step 2: LWFW Ingress Filter Hook
    // ============================================
    #ifdef LWIP_HOOK_IP4_INPUT
    if (LWIP_HOOK_IP4_INPUT(p, inp)) {
        return ERR_OK;  // packet 被 filter 丢弃
    }
    #endif

    // ============================================
    // Step 3: 设置 ip_data
    // ============================================
    ip_addr_copy_from_ip4(ip_data.current_iphdr_dest, iphdr->dest);
    ip_addr_copy_from_ip4(ip_data.current_iphdr_src, iphdr->src);
    ip_data.current_input_netif = inp;  // ← 关键！设置当前 netif

    // ============================================
    // Step 4: 验证目标地址
    // ============================================
    if (ip4_addr_ismulticast(ip4_current_dest_addr())) {
        // 多播: 检查 inp 是否加入该组
        #if LWIP_IGMP
        if (igmp_lookfor_group(inp, ...)) {
            netif = inp;
        }
        #endif
    } else {
        // 单播: 先检查 inp
        if (ip4_input_accept(inp)) {
            netif = inp;
        } else {
            // inp 不匹配，遍历 netif_list
            #if !LWIP_SINGLE_NETIF
            NETIF_FOREACH(netif) {
                if (netif == inp) continue;
                if (ip4_input_accept(netif)) {
                    break;
                }
            }
            #endif
        }
    }

    // ============================================
    // Step 5: 发送到上层协议
    // ============================================
    if (netif != NULL) {
        // TCP/UDP/RAW/ICMP 处理
        switch (IPH_PROTO(iphdr)) {
            case IP_PROTO_TCP:
                tcp_input(p, netif);
                break;
            case IP_PROTO_UDP:
                udp_input(p, netif);
                break;
            case IP_PROTO_ICMP:
                icmp_input(p, netif);
                break;
            default:
                raw_input(p, netif);
        }
    }

    return ERR_OK;
}
```

---

## 3. ip4_input_accept — 地址验证

**文件**: `ip4.c:396-431`

```c
ip4_input_accept(struct netif *netif)
{
    // 检查 netif 是否 up 且有有效 IP
    if ((netif_is_up(netif)) && (!ip4_addr_isany_val(*netif_ip4_addr(netif)))) {
        // 检查目标 IP
        if (ip4_addr_cmp(ip4_current_dest_addr(), netif_ip4_addr(netif)) ||
            // 或广播地址
            ip4_addr_isbroadcast(ip4_current_dest_addr(), netif) ||
            // 或环回地址
            ip4_addr_get_u32(ip4_current_dest_addr()) == PP_HTONL(IPADDR_LOOPBACK)) {
            return 1;  // 接受
        }
    }
    return 0;  // 拒绝
}
```

---

## 4. 关键设计点

### 4.1 ip_data.current_input_netif

```c
ip_data.current_input_netif = inp;  // 在 ip4_input 中设置
```

这个值在 UDP/TCP socket 绑定检查中使用：

```c
// udp.c:151-153
if ((pcb->netif_idx != NETIF_NO_INDEX) &&
    (pcb->netif_idx != netif_get_index(ip_data.current_input_netif))) {
    return 0;  // socket 绑定的 netif 与接收的不匹配
}
```

### 4.2 NETIF_FOREACH 遍历

如果直接传入的 `inp` 不匹配，会遍历 `netif_list`：

```c
NETIF_FOREACH(netif) {
    if (netif == inp) continue;  // 跳过已检查的
    if (ip4_input_accept(netif)) {
        break;  // 找到匹配
    }
}
```

### 4.3 LWIP_HOOK_IP4_INPUT

```c
#ifdef LWIP_HOOK_IP4_INPUT
if (LWIP_HOOK_IP4_INPUT(p, inp)) {
    return ERR_OK;  // LWFW 可以在这里丢弃 packet
}
#endif
```

这是 **LWFW ingress filter** 的 hook 点！

---

## 5. 与其他模块的关系

### 5.1 上游调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **ethernet_input** | `ip4_input()` | L2→L3 入口 |

### 5.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **tcp_input** | TCP 入口 | TCP segment 处理 |
| **udp_input** | UDP 入口 | UDP datagram 处理 |
| **raw_input** | RAW 入口 | RAW socket 处理 |
| **icmp_input** | ICMP 入口 | ping 等 |

### 5.3 Hook 点

| Hook | 位置 | 说明 |
|------|------|------|
| `LWIP_HOOK_IP4_INPUT` | Step 2 | LWFW ingress filter |

---

## 6. 总结

### 6.1 ip4_input 的核心作用

```
收到 IP packet
    │
    ├─► 解析 IP Header
    │
    ├─► LWFW Ingress Filter (可选丢弃)
    │
    ├─► 验证目标地址
    │     - inp 直接匹配
    │     - 或遍历 netif_list 找匹配
    │
    ├─► 设置 ip_data.current_input_netif
    │
    └─► 分发到上层协议
          - TCP → tcp_input()
          - UDP → udp_input()
```

### 6.2 关键设计

1. **ip_data.current_input_netif**: 在 ip4_input 中设置，供 UDP/TCP socket 绑定检查使用
2. **NETIF_FOREACH**: 如果 inp 不匹配，会遍历所有 netif
3. **LWIP_HOOK_IP4_INPUT**: LWFW ingress filter 的 hook 点
