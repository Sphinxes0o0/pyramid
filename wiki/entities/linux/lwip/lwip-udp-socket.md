---
type: entity
tags: [linux, lwip, network, udp]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP UDP Socket Analysis

## 定义

UDP 是无连接的协议，但 lwIP 仍然使用 PCB (Protocol Control Block) 来管理 socket 状态。UDP PCB 保存 local/remote 地址和端口，通过 `udp_pcbs` 全局链表进行接收匹配。

## UDP PCB 结构

```c
struct udp_pcb {
    IP_PCB;                    // IP 地址 + port
    struct udp_pcb *next;     // 链表指针
    u8_t flags;               // UDP_FLAGS_*

    u16_t local_port;         // 本地端口
    u16_t remote_port;         // 远端端口

#if LWIP_IGMP
    ip4_addr_t mcast_group[LWIP_MAX_NUM_MCAST_GROUP];  // 加入的多播组
#endif

    udp_recv_fn recv;         // 接收回调
    void *recv_arg;          // 回调参数
};
```

### UDP Flags

```c
#define UDP_FLAGS_NONE           0x00
#define UDP_FLAGS_UDPLITE        0x02  // UDPLite 协议
#define UDP_FLAGS_NOCHKSUM      0x04  // 不计算 checksum
#define UDP_FLAGS_MULTICAST_LOOP 0x08  // 多播回环
#define UDP_FLAGS_CONNECTED      0x10  // 已连接 (connect() 调用过)
```

## Socket API 对应关系

| Socket API | Raw API | 说明 |
|-----------|---------|------|
| `socket()` | `udp_new()` | 创建 PCB，memp_malloc(MEMP_UDP_PCB) |
| `bind()` | `udp_bind()` | 绑定地址端口，加入 udp_pcbs 链表 |
| `connect()` | `udp_connect()` | 设置远端地址/端口，设置 CONNECTED 标志 |
| `sendto()` | `udp_sendto()` | 发送数据，自动查找路由 |
| `recvfrom()` | `recv` 回调 | 遍历 udp_pcbs 匹配 |

## UDP connect vs TCP connect

| 特性 | UDP connect | TCP connect |
|------|-------------|-------------|
| **含义** | 设置远端地址/端口 | 建立连接 (3次握手) |
| **状态** | 只设置 pcb->remote_ip/port | PCB 状态变为 SYN_SENT |
| **网络操作** | 无 | 发送 SYN |
| **CONNECTED 标志** | 设置 | 不设置 |

## UDP Socket 流程

```
socket()                    // udp_new()
    │                         pcb = memp_malloc(MEMP_UDP_PCB)
    ▼                         pcb->local_port = 0
bind(addr, port)             // udp_bind()
    │                         ├─ 分配端口 (如果 port=0)
    │                         └─ 加入 udp_pcbs 链表
    ▼
connect(remote, port)       // udp_connect()
    │                         ├─ 如果未 bind，先 bind
    │                         └─ 设置 remote_ip/port
    │                           └─ 设置 UDP_FLAGS_CONNECTED
    ▼
sendto(data)                // udp_sendto()
    │                         ├─ 查找路由 → netif
    │                         ├─ 添加 UDP Header
    │                         ├─ 计算 Checksum
    │                         └─ ip_output_if()
    ▼
recvfrom()                  // recv 回调
                              └─ 遍历 udp_pcbs 匹配
```

## 相关概念

- [[entities/linux/lwip/lwip-udp-input]] — udp_input 中遍历 udp_pcbs 匹配
- [[entities/linux/lwip/lwip-udp-output]] — udp_sendto 发送流程
- [[entities/linux/lwip/lwip-igmp]] — IGMP 多播组成管理
- [[entities/linux/lwip/lwip-tcp-socket]] — TCP socket 流程对比
