---
type: entity
tags: [linux, lwip, network, raw-socket, af-packet, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# lwIP RAW Socket — AF-PACKET Implementation

## 定义

RAW socket 允许应用直接访问 L2/L3 协议数据。lwIP 的 **AF-PACKET** 实现通过 `raw_afpacket_input()` 在 `ethernet_input()` 中捕获 Ethernet 层数据包，配合 `packet_mmap` 实现零拷贝应用获取。

## RAW PCB 结构

```c
struct raw_pcb {
    IP_PCB;                    // IP 地址 + port

    struct raw_pcb *next;     // 链表指针
    u8_t domain;              // AF_INET / AF_INET6
    u16_t protocol;           // 协议号 (e.g., ETH_P_IP)
    u8_t flags;

    // 接收回调
    raw_recv_fn recv;
    void *recv_arg;

    // ==== SafeOS AF-PACKET 扩展 ====
    struct sockaddr_ll sockaddr;  // AF-PACKET 绑定信息
    void *conn;                   // packet_mmap 连接
    u8_t state;
};
```

## 两类 RAW PCB 链表

```c
// 普通 RAW PCB 链表
static struct raw_pcb *raw_pcbs;

// AF-PACKET PCB 链表 (SafeOS)
static struct raw_pcb *raw_afpacket_pcbs;
```

## AF-PACKET Input 处理

**文件**: `core/raw.c:281-390`

```c
raw_input_state_t raw_afpacket_input(struct pbuf *p, struct netif *inp, u16_t type)
{
    for (pcb = raw_afpacket_pcbs; pcb != NULL; pcb = pcb->next) {
        // 检查协议匹配
        if (pcb->protocol != type) continue;

        // 检查接口匹配 (ifindex)
        if (pcb->sockaddr.sll_ifindex != 0 &&
            netif_get_index(inp) != pcb->sockaddr.sll_ifindex) continue;

        // 调用接收回调
        if (pcb->recv(pcb->recv_arg, pcb, p, ...) == 1) {
            ret = RAW_INPUT_DELIVERED;  // 回调消费了数据包
            break;
        }
    }
    return ret;
}
```

## 与 Ethernet Input 集成

```c
ethernet_input(struct pbuf *p, struct netif *netif)
{
    ethhdr = (struct eth_hdr *)p->payload;
    type = ethhdr->type;

    // 发送到 RAW (AF-PACKET)
    if (raw_afpacket_input(p, netif, type) == RAW_INPUT_DELIVERED) {
        return ERR_OK;  // 被 RAW socket 消费
    }

    // 分发到上层协议
    switch (type) {
        case ETHTYPE_IP:   ip4_input(p, netif);  break;
        case ETHTYPE_ARP:  etharp_input(p, netif); break;
    }
}
```

## cBPF Socket 过滤

```
raw_afpacket_input()
    └─► lwip_run_socket_filter(conn, p, inp)
          └─► bpf_filter_run(sock->bpf_prog, p)
                └─► cbpf_execute_interpreter()
                    └─► bpf_filter_with_aux_data()
```

## AF-PACKET 架构

```
NIC 接收 packet
    │
    ▼
ethernet_input()
    │
    ├─► raw_afpacket_input()
    │     └─► 遍历 raw_afpacket_pcbs
    │           ├─► 检查 protocol (ETH_P_*)
    │           └─► 检查 ifindex
    │                 └─► 调用 recv 回调
    │
    └─► ip4_input() / etharp_input()
          (如果未被 RAW 消费)
```

## 与 Linux AF-PACKET 对比

| 特性 | SafeOS lwIP | Linux |
|------|-------------|-------|
| 实现位置 | lwIP raw.c | 内核 net/packet |
| 接收层次 | Ethernet (L2) | Ethernet (L2) |
| packet_mmap | 通过 conn 指针 | 原生支持 |
| 协议匹配 | ETH_P_* | ETH_P_* |

## 相关概念

- [[entities/linux/lwip/lwip-packet-mmap]] — AF-PACKET mmap 实现
- [[entities/linux/lwip/lwip-firewall]] — LWFW 防火墙 / cBPF 过滤
- [[entities/linux/lwip/lwip-ethernet-input]] — L2→L3 入口

## 来源详情

- [[sources/safeos-lwip-extensions]]
