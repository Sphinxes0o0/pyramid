---
type: entity
tags: [linux, lwip, network, udp]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP udp_input Analysis

## 定义

`udp_input()` 是 lwIP 的 **UDP 层入口函数**，负责解析 UDP Header、checksum 校验、demultiplex 找到匹配的 UDP PCB、回调节知 application。

## 函数源码

```c
void udp_input(struct pbuf *p, struct netif *inp) {
    struct udp_hdr *udphdr;
    struct udp_pcb *pcb, *uncon_pcb;
    u16_t src, dest;
    u8_t broadcast = ip_addr_isbroadcast(ip_current_dest_addr(), inp);

    // 解析 UDP Header
    udphdr = (struct udp_hdr *)p->payload;
    src = lwip_ntohs(udphdr->src);
    dest = lwip_ntohs(udphdr->dest);

    // Demultiplex — 遍历 udp_pcbs 链表
    for (pcb = udp_pcbs; pcb != NULL; pcb = pcb->next) {
        if (pcb->local_port == dest &&
            udp_input_local_match(pcb, inp, broadcast) != 0) {
            // 完全匹配优先 (connected)
            if ((pcb->flags & UDP_FLAGS_CONNECTED) == 0) {
                uncon_pcb = pcb;  // 记录未连接 PCB
            }
            if (pcb->remote_port == src &&
                (ip_addr_isany_val(pcb->remote_ip) ||
                 ip_addr_cmp(&pcb->remote_ip, ip_current_src_addr()))) {
                // 移到链表头部 (locality optimization)
                if (prev != NULL) { prev->next = pcb->next; pcb->next = udp_pcbs; udp_pcbs = pcb; }
                break;  // 完全匹配
            }
        }
        prev = pcb;
    }

    // 无完全匹配则使用未连接 PCB
    if (pcb == NULL) pcb = uncon_pcb;

    // Checksum 校验
    #if CHECKSUM_CHECK_UDP
    if (ip_chksum_pseudo(p, IP_PROTO_UDP, p->tot_len,
                          ip_current_src_addr(), ip_current_dest_addr()) != 0) {
        goto chkerr;
    }
    #endif

    // 移除 UDP Header，回调通知应用
    pbuf_remove_header(p, UDP_HLEN);
    if (pcb != NULL && pcb->recv != NULL) {
        pcb->recv(pcb->recv_arg, pcb, p, ip_current_src_addr(), src);
    } else {
        if (!broadcast && !ip_addr_ismulticast(ip_current_dest_addr())) {
            icmp_port_unreach(ip_current_is_v6(), p);  // ICMP 错误
        }
        pbuf_free(p);
    }
}
```

## UDP PCB 匹配规则

| 优先级 | 条件 | 说明 |
|--------|------|------|
| **完全匹配** | local_port + remote_port + remote_ip | Connected socket |
| **部分匹配** | local_port + local_ip | Unconnected socket fallback |
| **广播** | 子网内 + SO_BROADCAST | 需要 flag |
| **多播** | 已加入多播组 | 依赖 IGMP |

## UDP Header 结构

```
┌──────────────┬──────────────┬──────────────┬──────────────┐
│ src port(16)│ dst port(16)│  len (16)   │ chksum(16)  │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

## 与 TCP 对比

| 特性 | UDP | TCP |
|------|-----|-----|
| **状态** | 无状态 | 有状态 (PCB) |
| **连接** | 无连接 | 三次握手 |
| **可靠性** | 不可靠 | 可靠 |
| **PCB 查找** | O(n) | O(n) |

## 相关概念

- [[entities/linux/lwip/lwip-udp-socket]] — UDP PCB 管理、bind/connect
- [[entities/linux/lwip/lwip-udp-output]] — UDP 封装、checksum pseudo-header
- [[entities/linux/lwip/lwip-pbuf]] — mbox 传递的是 pbuf 指针
- [[entities/linux/lwip/lwip-ip4-input]] — ip4_input 分发到 udp_input
