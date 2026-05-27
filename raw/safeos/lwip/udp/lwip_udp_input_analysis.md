# udp_input 分析 — T-040

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: udp_input 函数：UDP header 解析、socket 匹配、broadcast/multicast 处理

---

## 1. 概述

`udp_input()` 是 lwIP 的 **UDP 层入口函数**，负责：
1. 解析 UDP Header
2. Checksum 校验
3. Demultiplex — 找到匹配的 UDP PCB
4. 回调通知 application

### 1.1 调用链

```
ip4_input()
    │
    ▼
udp_input(p, inp)
    │
    ├─► 解析 UDP Header (src/dst port)
    │
    ├─► Demultiplex — 找 PCB
    │     - 完全匹配 (remote_ip + remote_port + local_port)
    │     - 部分匹配 (local_port + local_ip)
    │     - 广播/多播处理
    │
    ├─► Checksum 校验
    │
    └─► pcb->recv() 回调
          └─► 或发送 ICMP Port Unreachable
```

---

## 2. 函数源码分析

**文件**: `external/lwip_ds_mcu/src/core/udp.c:212`

### 2.1 udp_input — 主入口

```c
// udp.c:212-484
void
udp_input(struct pbuf *p, struct netif *inp)
{
    struct udp_hdr *udphdr;
    struct udp_pcb *pcb, *prev;
    struct udp_pcb *uncon_pcb;  // 未连接的匹配 PCB
    u16_t src, dest;
    u8_t broadcast;
    u8_t for_us = 0;

    // ============================================
    // Step 1: UDP Header 解析
    // ============================================
    if (p->len < UDP_HLEN) {
        goto dropped;
    }

    udphdr = (struct udp_hdr *)p->payload;
    src = lwip_ntohs(udphdr->src);
    dest = lwip_ntohs(udphdr->dest);

    // ============================================
    // Step 2: 判断是否广播
    // ============================================
    broadcast = ip_addr_isbroadcast(ip_current_dest_addr(), ip_current_netif());

    // ============================================
    // Step 3: Demultiplex — 找 PCB
    // ============================================
    for (pcb = udp_pcbs; pcb != NULL; pcb = pcb->next) {
        // 3.1 检查 local port + local IP 匹配
        if ((pcb->local_port == dest) &&
            (udp_input_local_match(pcb, inp, broadcast) != 0)) {

            // 3.2 记录第一个未连接的 PCB
            if ((pcb->flags & UDP_FLAGS_CONNECTED) == 0) {
                if (uncon_pcb == NULL) {
                    uncon_pcb = pcb;
                }
                // ... 广播特殊处理
            }

            // 3.3 检查 remote port + remote IP 完全匹配
            if ((pcb->remote_port == src) &&
                (ip_addr_isany_val(pcb->remote_ip) ||
                 ip_addr_cmp(&pcb->remote_ip, ip_current_src_addr()))) {
                // 移动到链表头部 (locality optimization)
                if (prev != NULL) {
                    prev->next = pcb->next;
                    pcb->next = udp_pcbs;
                    udp_pcbs = pcb;
                }
                break;  // 找到完全匹配
            }
        }
        prev = pcb;
    }

    // ============================================
    // Step 4: 无完全匹配则使用未连接 PCB
    // ============================================
    if (pcb == NULL) {
        pcb = uncon_pcb;
    }

    // ============================================
    // Step 5: Checksum 校验
    // ============================================
    #if CHECKSUM_CHECK_UDP
    IF__NETIF_CHECKSUM_ENABLED(inp, NETIF_CHECKSUM_CHECK_UDP) {
        if (udphdr->chksum != 0) {
            if (ip_chksum_pseudo(p, IP_PROTO_UDP, p->tot_len,
                                  ip_current_src_addr(), ip_current_dest_addr()) != 0) {
                goto chkerr;  // checksum 失败
            }
        }
    }
    #endif

    // ============================================
    // Step 6: 移除 UDP Header，传递给应用
    // ============================================
    pbuf_remove_header(p, UDP_HLEN);

    if (pcb != NULL) {
        // 回调通知应用
        if (pcb->recv != NULL) {
            pcb->recv(pcb->recv_arg, pcb, p, ip_current_src_addr(), src);
        } else {
            pbuf_free(p);  // 无回调，释放
        }
    } else {
        // 无匹配 PCB
        if (!broadcast && !ip_addr_ismulticast(ip_current_dest_addr())) {
            // 发送 ICMP Port Unreachable
            pbuf_header_force(p, (s16_t)(ip_current_header_tot_len() + UDP_HLEN));
            icmp_port_unreach(ip_current_is_v6(), p);
        }
        pbuf_free(p);
    }
}
```

### 2.2 udp_input_local_match — 地址匹配

**文件**: `udp.c:141-198`

```c
static u8_t
udp_input_local_match(struct udp_pcb *pcb, struct netif *inp, u8_t broadcast)
{
    // ============================================
    // Step 1: 检查 netif 绑定
    // ============================================
    if ((pcb->netif_idx != NETIF_NO_INDEX) &&
        (pcb->netif_idx != netif_get_index(ip_data.current_input_netif))) {
        return 0;
    }

    // ============================================
    // Step 2: ANY_TYPE — 双重栈，接收所有
    // ============================================
    if (IP_IS_ANY_TYPE_VAL(pcb->local_ip)) {
        #if LWIP_IPV4 && IP_SOF_BROADCAST_RECV
        if ((broadcast != 0) && !ip_get_option(pcb, SOF_BROADCAST)) {
            return 0;
        }
        #endif
        return 1;
    }

    // ============================================
    // Step 3: 匹配 local IP
    // ============================================
    if (IP_ADDR_PCB_VERSION_MATCH_EXACT(pcb, ip_current_dest_addr())) {
        // 广播处理
        if (broadcast != 0) {
            if (ip4_addr_isany(ip_2_ip4(&pcb->local_ip)) ||
                ip4_addr_netcmp(ip_2_ip4(&pcb->local_ip),
                                 ip4_current_dest_addr(),
                                 netif_ip4_netmask(inp))) {
                return 1;
            }
        }
        // 单播/多播匹配
        if (ip_addr_isany(&pcb->local_ip) ||
            ip_addr_cmp(&pcb->local_ip, ip_current_dest_addr()) ||
            // 多播 + 组成员检查
            (ip_addr_ismulticast(ip_current_dest_addr()) &&
             lookup_mcast_membership_in_pcb(pcb, ip_current_dest_addr()))) {
            return 1;
        }
    }

    return 0;
}
```

---

## 3. UDP PCB 匹配规则

### 3.1 匹配优先级

```
1. 完全匹配 (Connected)
   - local_port == dst_port
   - local_ip == dst_ip (或 ANY)
   - remote_port == src_port
   - remote_ip == src_ip (或 ANY)

2. 部分匹配 (Unconnected)
   - local_port == dst_port
   - local_ip == dst_ip (或 ANY)
   - remote_* 任意
```

### 3.2 广播/多播处理

| 场景 | 匹配条件 |
|------|----------|
| **单播** | local_ip == dst_ip |
| **广播** | 在子网内 + SO_BROADCAST 标志 |
| **多播** | 已加入多播组 (mcast_group) |

### 3.3 SO_REUSE 和 SO_REUSEADDR

```c
#if SO_REUSE && SO_REUSE_RXTOALL
if (ip_get_option(pcb, SOF_REUSEADDR) &&
    (broadcast || ip_addr_ismulticast(ip_current_dest_addr()))) {
    // 广播/多播复制到所有匹配 socket
    for (mpcb = udp_pcbs; mpcb != NULL; mpcb = mpcb->next) {
        if ((mpcb->local_port == dest) &&
            (udp_input_local_match(mpcb, inp, broadcast) != 0)) {
            // 复制 packet 给 mpcb
            pbuf_ref(p);
            mpcb->recv(mpcb->recv_arg, mpcb, p, ip_current_src_addr(), src);
        }
    }
}
#endif
```

---

## 4. 与 TCP 的对比

| 特性 | UDP | TCP |
|------|-----|-----|
| **状态** | 无状态 | 有状态 (PCB) |
| **连接** | 无连接 | 三次握手 |
| **可靠性** | 不可靠 | 可靠 |
| **排序** | 无序 | 有序 |
| **流量控制** | 无 | 有 (cwnd, rwnd) |
| **拥塞控制** | 无 | 有 |
| **PCB 查找** | O(n) | O(n) |

---

## 5. 与其他模块的关系

### 5.1 上游调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **ip4_input** | `udp_input()` | IP 层分发 |

### 5.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **icmp_port_unreach** | udp_input:458 | 无匹配时发送 ICMP |
| **ip_chksum_pseudo** | udp_input:383 | Checksum 计算 |

### 5.3 数据结构

| 结构 | 说明 |
|------|------|
| **udp_pcb** | UDP 协议控制块 |
| **udp_hdr** | UDP Header (8 bytes) |

---

## 6. UDP Header 结构

```
┌─────────────────────────────────────────────┐
│ 0                   1                   2  │
│ 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1│
├─────────────────────────────────────────────┤
│          Source Port (16)                   │
├─────────────────────────────────────────────┤
│        Destination Port (16)                │
├─────────────────────────────────────────────┤
│           Length (16)                      │
├─────────────────────────────────────────────┤
│          Checksum (16)                      │
└─────────────────────────────────────────────┘
```

---

## 7. 总结

### 7.1 udp_input 的核心作用

```
收到 UDP Datagram
    │
    ├─► 解析 Header (src/dst port)
    │
    ├─► Demultiplex — 找 PCB
    │     - 完全匹配优先 (connected)
    │     - 未连接 PCB 作为 fallback
    │     - 支持广播/多播
    │
    ├─► Checksum 校验
    │
    └─► 回调或 ICMP 错误
```

### 7.2 关键设计

1. **无连接特性**: UDP 无连接状态，完全靠 PCB 匹配
2. **广播/多播支持**: 需要 SO_BROADCAST 标志和 IGMP 组成员
3. **ICMP 错误**: 无匹配时返回 ICMP Port Unreachable (单播)
4. **locality optimization**: 匹配的 PCB 移到链表头部

### 7.3 SafeOS 特供

无明显特供修改，udp_input 保持标准 lwIP 实现。
