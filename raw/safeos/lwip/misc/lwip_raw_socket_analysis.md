# RAW Socket 分析 — T-050

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: RAW socket 实现、AF-PACKET 绑定、raw_pcb 管理和回调

---

## 1. 概述

RAW socket 允许应用直接访问 L2/L3 协议，是 lwIP 支持 **AF-PACKET** 的基础：

1. **协议捕获**: 接收任意协议类型的数据包
2. **AF-PACKET**: Linux 风格的 packet socket
3. **自定义协议**: 实现非标准协议

---

## 2. RAW PCB 结构

**文件**: `include/lwip/raw.h:88-123`

```c
struct raw_pcb {
    IP_PCB;                    // IP 地址 + port

    struct raw_pcb *next;     // 链表指针

    u8_t domain;              // AF_INET / AF_INET6
    u16_t protocol;           // 协议号 (e.g., ETH_P_IP)
    u8_t flags;              // 标志

#if LWIP_MULTICAST_TX_OPTIONS
    u8_t mcast_ifindex;      // 多播接口索引
    u8_t mcast_ttl;          // 多播 TTL
#endif

    // 接收回调
    raw_recv_fn recv;
    void *recv_arg;

#if LWIP_IPV6
    u16_t chksum_offset;     // Checksum 偏移
    u8_t chksum_reqd;       // 是否需要 checksum
#endif

    // ==== SafeOS AF-PACKET 扩展 ====
    struct sockaddr_ll sockaddr;  // AF-PACKET 绑定信息
    void *conn;                   // packet_mmap 连接
    u8_t state;                  // 状态
};
```

### 2.1 协议类型

```c
// Ethernet 协议类型 (ETH_P_*)
ETH_P_IP    = 0x0800   // IPv4
ETH_P_ARP   = 0x0806   // ARP
ETH_P_IPV6  = 0x86DD   // IPv6
ETH_P_VLAN  = 0x8100   // VLAN
```

---

## 3. RAW PCB 链表

### 3.1 两类链表

```c
// 普通 RAW PCB 链表
static struct raw_pcb *raw_pcbs;

// AF-PACKET PCB 链表 (SafeOS)
static struct raw_pcb *raw_afpacket_pcbs;
```

### 3.2 链表操作

```c
// 注册到 raw_pcbs
TCP_REG(&raw_pcbs, pcb);

// 注册到 raw_afpacket_pcbs
pcb->next = raw_afpacket_pcbs;
raw_afpacket_pcbs = pcb;
```

---

## 4. AF-PACKET 绑定

**文件**: `core/raw.c:694-810`

```c
err_t raw_afpacket_bind(struct raw_pcb *pcb,
                         const struct sockaddr *name,
                         socklen_t namelen)
{
    struct sockaddr_ll *addr = (struct sockaddr_ll *)name;

    // ============================================
    // Step 1: 解析地址
    // ============================================
    pcb->sockaddr.sll_ifindex = addr->sll_ifindex;  // 接口索引
    pcb->sockaddr.sll_protocol = addr->sll_protocol;  // 协议类型
    pcb->sockaddr.sll_family = AF_PACKET;

    // ============================================
    // Step 2: 设置协议
    // ============================================
    pcb->protocol = addr->sll_protocol;

    // ============================================
    // Step 3: 设置回调
    // ============================================
    pcb->recv = raw_afpacket_recv_callback;

    // ============================================
    // Step 4: 注册到 AF-PACKET 链表
    // ============================================
    for (p = raw_afpacket_pcbs; p != NULL; p = p->next) {
        if (p == pcb) {
            return ERR_OK;  // 已在链表中
        }
    }
    pcb->next = raw_afpacket_pcbs;
    raw_afpacket_pcbs = pcb;

    return ERR_OK;
}
```

---

## 5. AF-PACKET Input 处理

**文件**: `core/raw.c:281-390`

```c
raw_input_state_t
raw_afpacket_input(struct pbuf *p, struct netif *inp, u16_t type)
{
    struct raw_pcb *pcb;
    raw_input_state_t ret = RAW_INPUT_NONE;

    // 遍历 AF-PACKET PCB 链表
    for (pcb = raw_afpacket_pcbs; pcb != NULL; pcb = pcb->next) {
        // 检查协议匹配
        if (pcb->protocol != type) {
            continue;
        }

        // 检查接口匹配 (ifindex)
        if (pcb->sockaddr.sll_ifindex != 0 &&
            netif_get_index(inp) != pcb->sockaddr.sll_ifindex) {
            continue;
        }

        // 调用接收回调
        if (pcb->recv(pcb->recv_arg, pcb, p,
                      ip_current_src_addr(), ip_current_dest_addr()) == 1) {
            // 回调消费了数据包
            ret = RAW_INPUT_DELIVERED;
            break;
        }
    }

    return ret;
}
```

---

## 6. RAW Input 流程

**文件**: `core/raw.c:475-560`

```c
raw_input_state_t
raw_input(struct pbuf *p, struct netif *inp)
{
    struct raw_pcb *pcb;
    raw_input_state_t ret = RAW_INPUT_NONE;

    // ============================================
    // Step 1: 遍历 raw_pcbs 链表
    // ============================================
    for (pcb = raw_pcbs; pcb != NULL; pcb = pcb->next) {
        // 检查协议
        if (pcb->protocol != ip_current_header_proto()) {
            continue;
        }

        // 检查本地地址
        if (!ip_addr_ismanycast(&pcb->local_ip) &&
            !ip_addr_cmp(&pcb->local_ip, ip_current_dest_addr())) {
            continue;
        }

        // 调用回调
        if (pcb->recv(pcb->recv_arg, pcb, p, ...) == 1) {
            ret = RAW_INPUT_DELIVERED;
            break;
        }
    }

    return ret;
}
```

---

## 7. AF-PACKET Output 处理

**文件**: `core/raw.c:391-443`

```c
raw_input_state_t
raw_afpacket_output(struct pbuf *p, struct netif *inp)
{
    struct raw_pcb *pcb;
    raw_input_state_t ret = RAW_INPUT_NONE;

    for (pcb = raw_afpacket_pcbs; pcb != NULL; pcb = pcb->next) {
        // 检查协议
        if (pcb->protocol != eth_type) {
            continue;
        }

        // 检查接口
        if (pcb->sockaddr.sll_ifindex != 0 &&
            netif_get_index(inp) != pcb->sockaddr.sll_ifindex) {
            continue;
        }

        // 调用回调
        if (pcb->recv(pcb->recv_arg, pcb, p, ...) == 1) {
            ret = RAW_INPUT_DELIVERED;
            break;
        }
    }

    return ret;
}
```

---

## 8. AF-PACKET 与 Ethernet Input 集成

**文件**: `netif/ethernet.c`

```c
ethernet_input(struct pbuf *p, struct netif *netif)
{
    // ...

    // 解析 Ethernet Header
    ethhdr = (struct eth_hdr *)p->payload;
    type = ethhdr->type;

    // 发送到 RAW (AF-PACKET)
    if (raw_afpacket_input(p, netif, type) == RAW_INPUT_DELIVERED) {
        return ERR_OK;  // 被 RAW socket 消费
    }

    // 分发到上层协议
    switch (type) {
        case ETHTYPE_IP:
            ip4_input(p, netif);
            break;
        case ETHTYPE_ARP:
            etharp_input(p, netif);
            break;
    }
}
```

---

## 9. raw_sendto — 发送 RAW 数据包

**文件**: `core/raw.c`

```c
err_t raw_sendto(struct raw_pcb *pcb, struct pbuf *p,
                 const ip_addr_t *ipaddr)
{
    struct netif *netif;

    // 路由查找
    netif = ip_route(ipaddr);
    if (netif == NULL) {
        return ERR_RTE;
    }

    return raw_sendto_if_src(pcb, p, ipaddr, netif,
                              ip_netif_get_local_ip(netif, ipaddr));
}

err_t raw_sendto_if_src(struct raw_pcb *pcb, struct pbuf *p,
                        const ip_addr_t *dst_ip, struct netif *netif,
                        const ip_addr_t *src_ip)
{
    // 添加 IP Header
    pbuf_add_header(p, IP_HLEN);

    // 填充 IP Header
    // ...

    // 发送到 IP 层
    return ip_output_if(p, src_ip, dst_ip, ttl, tos, proto, netif);
}
```

---

## 10. 总结

### 10.1 AF-PACKET 架构

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

### 10.2 两类 RAW socket

| 类型 | 链表 | 用途 |
|------|------|------|
| **raw_pcbs** | 普通 RAW | 接收 IP 层数据 |
| **raw_afpacket_pcbs** | AF-PACKET | 接收 Ethernet 层数据 |

### 10.3 关键设计

1. **协议匹配**: `pcb->protocol` 匹配 Ethernet type 或 IP protocol
2. **接口匹配**: `sll_ifindex` 指定监听接口
3. **回调消费**: `recv` 返回 1 表示数据包已被消费
4. **分发放行**: 如果没有被 RAW 消费，继续分发到上层

### 10.4 与 Linux AF-PACKET 对比

| 特性 | SafeOS lwIP | Linux |
|------|-------------|-------|
| **实现位置** | lwIP raw.c | 内核 net/packet |
| **接收层次** | Ethernet (L2) | Ethernet (L2) |
| **packet_mmap** | 通过 conn 指针 | 原生支持 |
| **协议匹配** | ETH_P_* | ETH_P_* |
