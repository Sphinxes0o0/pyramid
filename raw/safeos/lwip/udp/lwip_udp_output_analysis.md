# UDP 输出分析 — T-041

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: UDP 封装、checksum 计算、udp_output 函数

---

## 1. 概述

UDP 输出相比 TCP 简单得多，因为 UDP 是无连接的：
1. 分配 UDP Header 空间
2. 填充 UDP Header (src port, dest port, len, chksum)
3. 计算 UDP Checksum (pseudo-header)
4. 调用 IP 层发送

---

## 2. UDP PCB 结构

**文件**: `include/lwip/udp.h:82-123`

```c
struct udp_pcb {
    IP_PCB;                    // IP 地址 + port

    struct udp_pcb *next;     // 链表指针
    u8_t flags;               // UDP_FLAGS_*

    // ports are in host byte order
    u16_t local_port;         // 本地端口
    u16_t remote_port;        // 远端端口

    // for netstat
    ip_addr_t netstat_remote_ip;
    u16_t netstat_remote_port;

#if LWIP_MULTICAST_TX_OPTIONS
    ip4_addr_t mcast_ip4;     // 多播接口 IP
    u8_t mcast_ifindex;       // 多播接口索引
    u8_t mcast_ttl;           // 多播 TTL
#endif

#if LWIP_IGMP
    ip4_addr_t mcast_group[LWIP_MAX_NUM_MCAST_GROUP];  // 加入的多播组
#endif

#if LWIP_UDPLITE
    u16_t chksum_len_rx, chksum_len_tx;  // UDPLite checksum length
#endif

    // 回调函数
    udp_recv_fn recv;         // 接收回调
    void *recv_arg;           // 回调参数
};
```

### 2.1 UDP Flags

```c
#define UDP_FLAGS_NONE       0x00
#define UDP_FLAGS_UDPLITE    0x02  // UDPLite 协议
#define UDP_FLAGS_NOCHKSUM   0x04  // 不计算 checksum
#define UDP_FLAGS_MULTICAST_LOOP 0x08  // 多播回环
```

---

## 3. UDP 输出函数调用链

```
udp_send()
    │
    └─► udp_sendto()
          │
          └─► udp_sendto_if()
                │
                └─► udp_sendto_if_src_chksum()
                      │
                      ├─► pbuf_add_header() 添加 UDP Header
                      ├─► 填充 UDP Header
                      ├─► 计算 UDP Checksum
                      └─► ip_output_if_src() → IP 层
```

---

## 4. udp_sendto_if_src_chksum 详解

**文件**: `core/udp.c:758-973`

```c
err_t udp_sendto_if_src_chksum(struct udp_pcb *pcb, struct pbuf *p,
                                const ip_addr_t *dst_ip, u16_t dst_port,
                                struct netif *netif, u8_t have_chksum,
                                u16_t chksum, const ip_addr_t *src_ip)
{
    struct udp_hdr *udphdr;
    struct pbuf *q;  // 最终要发送的 pbuf

    // ============================================
    // Step 1: 检查 PCB 是否已绑定端口
    // ============================================
    if (pcb->local_port == 0) {
        // 自动分配端口
        err = udp_bind(pcb, &pcb->local_ip, pcb->local_port);
        if (err != ERR_OK) {
            return err;
        }
    }

    // ============================================
    // Step 2: 分配 UDP Header 空间
    // ============================================
    if (pbuf_add_header(p, UDP_HLEN)) {
        // Header 空间不足，分配新的 pbuf
        q = pbuf_alloc(PBUF_IP, UDP_HLEN, PBUF_RAM);
        if (q == NULL) {
            return ERR_MEM;
        }
        q->priority = p->priority;
        pbuf_chain(q, p);  // 链在一起
    } else {
        q = p;  // 直接使用原 pbuf
    }

    // ============================================
    // Step 3: 填充 UDP Header
    // ============================================
    udphdr = (struct udp_hdr *)q->payload;
    udphdr->src = lwip_htons(pcb->local_port);   // 源端口
    udphdr->dest = lwip_htons(dst_port);          // 目的端口
    udphdr->chksum = 0x0000;                      // 初始为 0

    // ============================================
    // Step 4: 计算 UDP Checksum
    // ============================================
    if (IP_IS_V6(dst_ip) || (pcb->flags & UDP_FLAGS_NOCHKSUM) == 0) {
        // UDP checksum 计算 (IPv6 必须，IPv4 可选)
        udpchksum = ip_chksum_pseudo(q, IP_PROTO_UDP, q->tot_len,
                                     src_ip, dst_ip);

        // chksum 为 0 时改为 0xffff (0 表示"无 checksum")
        if (udpchksum == 0x0000) {
            udpchksum = 0xffff;
        }
        udphdr->chksum = udpchksum;
    }

    // ============================================
    // Step 5: 发送到 IP 层
    // ============================================
    err = ip_output_if_src(q, src_ip, dst_ip,
                           ttl, pcb->tos, IP_PROTO_UDP, netif);

    // ============================================
    // Step 6: 释放临时 header pbuf
    // ============================================
    if (q != p) {
        pbuf_free(q);  // 释放临时的 header pbuf
    }

    return err;
}
```

---

## 5. UDP Checksum 计算

### 5.1 Pseudo Header

UDP checksum 使用 pseudo header 来验证端到端的地址有效性：

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Pseudo Header (12 bytes)                        │
├─────────────────────────────────────────────────────────────────────┤
│   Source IP Address (4 bytes)                                        │
├─────────────────────────────────────────────────────────────────────┤
│   Destination IP Address (4 bytes)                                    │
├─────────────────────────────────────────────────────────────────────┤
│   Zeros (1 byte)   │  Protocol (1 byte = 17)  │  UDP Length (2B)   │
└─────────────────────────────────────────────────────────────────────┘
                                        │
                                        ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    UDP Header + Data                                │
├─────────────────────────────────────────────────────────────────────┤
│   src port (2B)  │  dest port (2B)  │  Length (2B)  │  Chksum (2B) │
├─────────────────────────────────────────────────────────────────────┤
│   Data Payload ...                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 计算公式

```c
// ip_chksum_pseudo() 计算
checksum = ip_chksum_pseudo(p, IP_PROTO_UDP, udp_len, src_ip, dst_ip);

// checksum = ~(sum of all 16-bit words) & 0xFFFF
// 如果结果为 0，表示"no checksum"，改为 0xFFFF
```

---

## 6. UDP vs TCP 发送对比

| 特性 | UDP | TCP |
|------|-----|-----|
| **连接状态** | 无连接 | 面向连接 |
| **发送前** | 直接发送 | 建立连接/3次握手 |
| **Header** | 8 bytes | 20+ bytes (可选 options) |
| **Checksum** | 可选 (IPv4) / 必须 (IPv6) | 可选 |
| **Flow Control** | 无 | 有 (滑动窗口) |
| **Congestion Control** | 无 | 有 (慢启动/拥塞避免) |
| **Sequence Number** | 无 | 有 |
| **Retransmission** | 无 | 有 (RTO) |
| **发送函数** | `udp_sendto()` | `tcp_write()` → `tcp_output()` |

---

## 7. 广播/多播支持

### 7.1 广播检查

```c
#if LWIP_IPV4 && IP_SOF_BROADCAST
if (!ip_get_option(pcb, SOF_BROADCAST) &&
    ip_addr_isbroadcast(dst_ip, netif)) {
    return ERR_VAL;  // 不允许发送广播
}
#endif
```

### 7.2 多播 TTL

```c
#if LWIP_MULTICAST_TX_OPTIONS
ttl = ip_addr_ismulticast(dst_ip) ?
      udp_get_multicast_ttl(pcb) : pcb->ttl;
#else
ttl = pcb->ttl;
#endif
```

### 7.3 多播回环

```c
#if LWIP_MULTICAST_TX_OPTIONS
if ((pcb->flags & UDP_FLAGS_MULTICAST_LOOP) &&
    ip_addr_ismulticast(dst_ip)) {
    q->flags |= PBUF_FLAG_MCASTLOOP;  // 发送给本地
}
#endif
```

---

## 8. UDP Header 结构

**文件**: `include/lwip/prot/udp.h`

```c
struct udp_hdr {
    u16_t src;   // 源端口
    u16_t dest;  // 目的端口
    u16_t len;   // UDP 长度 (header + data)
    u16_t chksum; // Checksum (0 = 无 checksum)
};

#define UDP_HLEN 8  // UDP Header 长度
```

---

## 9. UDP 输入处理 (对比)

**文件**: `core/udp.c:142-484`

```
udp_input()
    │
    ├─► 遍历 udp_pcbs 链表，匹配 (local_ip, local_port)
    │
    ├─► 验证 UDP Checksum
    │
    ├─► 移除 UDP Header (pbuf_remove_header)
    │
    └─► 调用 recv 回调
          │
          └─► 如果没有匹配的 PCB:
                ├─► 发送 ICMP Port Unreachable
                └─► 丢弃 packet
```

---

## 10. 总结

### 10.1 UDP 输出流程

```
udp_send()
    │
    ├─► 路由查找 (ip_route)
    │     └─► 找到 netif
    │
    ├─► 分配 UDP Header 空间
    │     └─► pbuf_add_header() 或分配新 pbuf
    │
    ├─► 填充 UDP Header
    │     ├─► src port = local_port
    │     ├─► dest port = dst_port
    │     └─► len = UDP_HLEN + data_len
    │
    ├─► 计算 Checksum (可选)
    │     └─► ip_chksum_pseudo() over pseudo-header
    │
    └─► 调用 ip_output_if_src()
          └─► LWFW egress_filter → netif->output()
```

### 10.2 与 TCP 的关键区别

1. **无连接**: 不需要建立连接，直接发送
2. **无状态**: 每个 packet 独立处理
3. **无重传**: 丢包由应用处理
4. **无拥塞控制**: 网络过载直接丢包
5. **轻量级**: Header 仅 8 bytes

### 10.3 SafeOS 特供

无明显特供修改，UDP 保持标准 lwIP 实现。
