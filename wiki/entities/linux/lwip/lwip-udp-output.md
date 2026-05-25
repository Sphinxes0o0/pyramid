---
type: entity
tags: [linux, lwip, network, udp]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP udp_output Analysis

## 定义

`udp_output()` 是 lwIP 的 **UDP 输出函数**，负责分配 UDP Header 空间、填充 Header、计算 checksum pseudo-header、调用 IP 层发送。

## 调用链

```
udp_send()
    │
    └─► udp_sendto()
          │
          └─► udp_sendto_if_src_chksum()
                │
                ├─► pbuf_add_header() 添加 UDP Header
                ├─► 填充 UDP Header
                ├─► 计算 UDP Checksum
                └─► ip_output_if_src() → IP 层
```

## udp_sendto_if_src_chksum 源码

```c
err_t udp_sendto_if_src_chksum(struct udp_pcb *pcb, struct pbuf *p,
                                const ip_addr_t *dst_ip, u16_t dst_port,
                                struct netif *netif, u8_t have_chksum,
                                u16_t chksum, const ip_addr_t *src_ip) {
    // 检查 PCB 是否已绑定端口
    if (pcb->local_port == 0) {
        err = udp_bind(pcb, &pcb->local_ip, pcb->local_port);
    }

    // 分配 UDP Header 空间
    if (pbuf_add_header(p, UDP_HLEN)) {
        q = pbuf_alloc(PBUF_IP, UDP_HLEN, PBUF_RAM);
        pbuf_chain(q, p);
    }

    // 填充 UDP Header
    udphdr = (struct udp_hdr *)q->payload;
    udphdr->src = lwip_htons(pcb->local_port);
    udphdr->dest = lwip_htons(dst_port);
    udphdr->chksum = 0x0000;

    // 计算 UDP Checksum (pseudo-header)
    udpchksum = ip_chksum_pseudo(q, IP_PROTO_UDP, q->tot_len, src_ip, dst_ip);
    if (udpchksum == 0x0000) udpchksum = 0xffff;  // 0 表示"无 checksum"
    udphdr->chksum = udpchksum;

    // 发送到 IP 层
    err = ip_output_if_src(q, src_ip, dst_ip, ttl, pcb->tos, IP_PROTO_UDP, netif);
}
```

## UDP Checksum 计算

UDP checksum 使用 pseudo-header 来验证端到端的地址有效性：

```
┌────────────────────────────────────────────────┐
│   Source IP Address (4 bytes)                   │
├────────────────────────────────────────────────┤
│   Destination IP Address (4 bytes)              │
├────────────────────────────────────────────────┤
│   Zeros (1B) │ Protocol (1B=17) │ UDP len(2B) │
└────────────────────────────────────────────────┘
                     │
                     ▼
┌────────────────────────────────────────────────┐
│   UDP Header + Data                            │
│   src port│dst port│ len │ chksum │ payload    │
└────────────────────────────────────────────────┘
```

## 与 TCP 发送对比

| 特性 | UDP | TCP |
|------|-----|-----|
| **Header** | 8 bytes | 20+ bytes |
| **Checksum** | 可选 (IPv4) / 必须 (IPv6) | 可选 |
| **Flow Control** | 无 | 有 (滑动窗口) |
| **Congestion Control** | 无 | 有 |
| **Sequence Number** | 无 | 有 |
| **Retransmission** | 无 | 有 (RTO) |

## 相关概念

- [[entities/linux/lwip/lwip-udp-input]] — UDP 输入 demultiplex
- [[entities/linux/lwip/lwip-udp-socket]] — UDP PCB 管理和 connect 流程
- [[entities/linux/lwip/lwip-ip4-output]] — ip_output_if_src IP 层发送
- [[entities/linux/lwip/lwip-pbuf]] — pbuf_add_header 分配 Header 空间
