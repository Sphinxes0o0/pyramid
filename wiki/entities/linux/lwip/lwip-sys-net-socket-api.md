---
type: entity
tags: [linux, lwip, network, nsv, socket-api, bsd, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# BSD Socket API — sys_net_* Implementation

## 定义

SafeOS NSv 通过 seL4 IPC 提供 BSD 兼容的 Socket API，所有 socket 操作都通过 `event_loop()` 中的 syscall 分发处理，最终调用 lwIP 的 socket API。

## event_loop 中的 Syscall 分发

```c
switch (sel4_msg_info_get_label(info)) {
    case SYS_NET_SOCKET:      ret = sys_socket_nb(info, badge); break;
    case SYS_NET_BIND:        ret = sys_bind_nb(info, badge); break;
    case SYS_NET_LISTEN:      ret = sys_listen_nb(info, badge); break;
    case SYS_NET_ACCEPT:      ret = sys_accept(info, badge); break;
    case SYS_NET_CONNECT:     ret = sys_connect_nb(info, badge); break;
    case SYS_NET_SENDTO:      ret = sys_sendto_nb(info, badge); break;
    case SYS_NET_RECVFROM:    ret = sys_recvfrom_nb(info, badge); break;
    case SYS_NET_CLOSE:       ret = sys_close_nb(info, badge); break;
    case SYS_NET_SELECT:      ret = select_thread(info, badge); break;
    case SYS_NET_SETSOCKOPT:  ret = sys_setsockopt(info, badge); break;
    case SYS_NET_GETSOCKOPT:  ret = sys_getsockopt(info, badge); break;
}
```

## 支持的协议族

```c
static int is_supported_domain(int domain)
{
    return (domain == AF_INET ||
            domain == AF_UNIX ||
            domain == AF_PACKET);
}

static int is_supported_type(int type)
{
    return (type == SOCK_STREAM ||  // TCP
            type == SOCK_DGRAM ||   // UDP
            type == SOCK_RAW);     // RAW
}
```

## Socket 信息管理

```c
typedef struct {
    pid_t owner;     // socket 所有者 PID
    int type;       // socket 类型 (NETCONN_TCP/UDP/RAW)
    int protocol;    // 协议
    void *conn;     // lwIP netconn 指针
} net_socket_info_t;

#define NUM_SOCKETS 256
static net_socket_info_t net_socket_info[NUM_SOCKETS];
```

## TCP Socket 创建和连接流程

```
应用                           NSv                          lwIP
 │                              │                            │
 │  SYS_NET_SOCKET             │                            │
 │─────────────────────────────►│  lwip_socket()             │
 │                              │───────────────────────────►│
 │  SYS_NET_BIND               │                            │
 │─────────────────────────────►│  lwip_bind()              │
 │                              │───────────────────────────►│
 │  SYS_NET_LISTEN             │                            │
 │─────────────────────────────►│  lwip_listen()            │
 │                              │───────────────────────────►│
 │  SYS_NET_CONNECT            │                            │
 │─────────────────────────────►│  lwip_connect()          │
 │                              │───────────────────────────►│
```

## lwip_socket → netconn 映射

```
lwip_socket(domain, type, protocol)
    │
    ├─► netconn_new_with_proto()
    │     ├─ netconn_alloc()
    │     │     ├─ sys_mbox_new()  // 创建 tcpip 线程邮箱
    │     │     └─ sys_sem_new()  // 创建信号量
    │     │
    │     └─ set_pcb_new(conn, domain, type, protocol)
    │           ├─ NETCONN_TCP   → alloc_tcp_pcb()
    │           ├─ NETCONN_UDP   → alloc_udp_pcb()
    │           ├─ NETCONN_RAW   → raw_new()
    │           └─ NETCONN_PACKET→ packet_new()
    │
    └─► alloc_socket(conn) → sock->conn = conn
```

## 与 Linux 对比

| 操作 | SafeOS NSv | Linux |
|------|-------------|-------|
| **socket** | seL4 IPC | 系统调用 |
| **bind** | seL4 IPC | 系统调用 |
| **listen** | seL4 IPC | 系统调用 |
| **accept** | seL4 IPC (可能阻塞) | 系统调用 (可能阻塞) |
| **connect** | seL4 IPC (非阻塞) | 系统调用 |
| **send/recv** | seL4 IPC + shm | 系统调用 + DMA |

## 相关概念

- [[entities/linux/lwip/lwip-nsv-event-loop]] — NSv 事件循环
- [[entities/linux/lwip/lwip-sys-net-send-recv]] — send/recv 流程
- [[entities/linux/lwip/lwip-sel4-function]] — 整体 lwIP 调用链

## 来源详情

- [[sources/safeos-lwip-extensions]]
