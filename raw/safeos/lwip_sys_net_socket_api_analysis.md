# SafeOS Socket API 分析

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: Socket 创建、bind、listen、accept、connect、close

---

## 1. 概述

SafeOS NSv 通过 seL4 IPC 提供 BSD 兼容的 Socket API，所有 socket 操作都通过 `event_loop()` 中的 syscall 分发处理。

---

## 2. Socket Syscall 分发

### 2.1 event_loop 中的分发

**文件**: `main.c`

```c
switch (sel4_msg_info_get_label(info)) {
    case SYS_NET_SOCKET:      ret = sys_socket_nb(info, badge); break;
    case SYS_NET_BIND:        ret = sys_bind_nb(info, badge); break;
    case SYS_NET_LISTEN:      ret = sys_listen_nb(info, badge); break;
    case SYS_NET_ACCEPT:      ret = sys_accept(info, badge); break;
    case SYS_NET_CONNECT:     ret = sys_connect_nb(info, badge); break;
    case SYS_NET_SENDTO:      ret = sys_sendto_nb(info, badge); break;
    case SYS_NET_RECVFROM:     ret = sys_recvfrom_nb(info, badge); break;
    case SYS_NET_CLOSE:       ret = sys_close_nb(info, badge); break;
    case SYS_NET_SELECT:      ret = select_thread(info, badge); break;
    case SYS_NET_SETSOCKOPT:  ret = sys_setsockopt(info, badge); break;
    case SYS_NET_GETSOCKOPT:  ret = sys_getsockopt(info, badge); break;
    // ...
}
```

---

## 3. sys_socket — 创建 Socket

### 3.1 sys_socket_nb

**文件**: `main.c:1000-1060`

```c
static int sys_socket_nb(sel4_msg_info_t UNUSED info, sel4_word badge)
{
    // 读取参数
    int domain = (int)sel4_get_mr(0);      // e.g., AF_INET, AF_UNIX
    int type = (int)sel4_get_mr(1);         // e.g., SOCK_STREAM, SOCK_DGRAM
    int protocol = (int)sel4_get_mr(2);    // e.g., IPPROTO_TCP

    pid_t pid = sys_get_pid_from_badge(badge);

    // 验证参数
    if (!is_supported_domain(domain)) {
        err = EAFNOSUPPORT;
    }
    if (!is_supported_proto(domain, protocol)) {
        err = EPROTONOSUPPORT;
    }
    if (!is_supported_type(type)) {
        err = EINVAL;
    }

    if (!err) {
        // 调用 lwIP 创建 socket
        socket = lwip_socket(domain, type, protocol);

        if (socket >= 0) {
            // 记录 socket 信息
            net_socket_info[socket].owner = pid;
            net_socket_info[socket].type = get_netconn_type(socket);
            netstat_add_info_by_badge(socket, badge);
            NET_PERF_STATS_INC(open_socket_success_cnts);
        } else {
            NET_PERF_STATS_INC(open_socket_fail_cnts);
        }
    }

    if (!err) {
        sys_reply_with_one_direct(0, socket);  // 返回 socket 描述符
    } else {
        sys_reply_with_err_direct(err);
    }
}
```

### 3.2 支持的协议族

```c
static int is_supported_domain(int domain)
{
    return (domain == AF_INET ||
            domain == AF_UNIX ||
            domain == AF_PACKET);
}
```

### 3.3 支持的类型

```c
static int is_supported_type(int type)
{
    return (type == SOCK_STREAM ||  // TCP
            type == SOCK_DGRAM ||   // UDP
            type == SOCK_RAW);      // RAW
}
```

---

## 4. sys_bind — 绑定地址

### 4.1 sys_bind_nb

**文件**: `main.c:1117-1138`

```c
static int sys_bind_nb(sel4_msg_info_t UNUSED info, sel4_word UNUSED badge)
{
    int socket = (int)sel4_get_mr(0);           // socket 描述符
    socklen_t socklen = (socklen_t)sel4_get_mr(1);  // 地址长度

    // 从 IPC message registers 获取 sockaddr
    struct sockaddr_storage sockaddr = {0};
    sys_unpack_data_from_mrs(2, &sockaddr, socklen, 0);

    // 调用 lwIP bind
    err = lwip_bind(socket, (struct sockaddr *)&sockaddr, socklen);

    if (err == -1) {
        sys_reply_with_err_direct(errno);
    } else {
        sys_reply_with_ok_direct();
    }
}
```

---

## 5. sys_listen — 监听连接

### 5.1 sys_listen_nb

**文件**: `main.c:1140-1153`

```c
static int sys_listen_nb(sel4_msg_info_t UNUSED info, sel4_word UNUSED badge)
{
    int socket = (int)sel4_get_mr(0);    // socket 描述符
    int backlog = (int)sel4_get_mr(1);  // backlog 大小

    err = lwip_listen(socket, backlog);

    if (err == -1) {
        sys_reply_with_err_direct(errno);
    } else {
        sys_reply_with_ok_direct();
    }
}
```

---

## 6. sys_accept — 接受连接

### 6.1 sys_accept

**文件**: `main.c:1155-1230`

```c
static int sys_accept(sel4_msg_info_t info, sel4_word badge)
{
    int socket = (int)sel4_get_mr(0);    // 监听 socket
    socklen_t socklen = sizeof(struct sockaddr_storage);
    int new_socket = -1;

    // 获取 socket flags
    int saved_flags = lwip_fcntl(socket, F_GETFL, 0);

    // 设置为阻塞模式
    lwip_fcntl(socket, F_SETFL, saved_flags & ~O_NONBLOCK);

    // 调用 lwIP accept
    new_socket = lwip_accept(socket, (struct sockaddr *)&client_addr, &socklen);

    if (new_socket < 0) {
        err = errno;
        goto err_exit;
    }

    // 恢复原始 flags
    lwip_fcntl(socket, F_SETFL, saved_flags);

    // 记录新 socket 信息
    net_socket_info[new_socket].owner = pid;
    net_socket_info[new_socket].type = get_netconn_type(new_socket);

    // 返回新 socket 描述符和地址
    sys_reply_with_two_direct(0, new_socket, client_len);
    return MSG_REPLIED;

err_exit:
    lwip_fcntl(socket, F_SETFL, saved_flags);
    sys_reply_with_err_direct(err);
}
```

---

## 7. sys_connect — 连接远端

### 7.1 sys_connect_nb

**文件**: `main.c`

```c
static int sys_connect_nb(sel4_msg_info_t info, sel4_word badge)
{
    int socket = (int)sel4_get_mr(0);
    socklen_t socklen = (socklen_t)sel4_get_mr(1);

    struct sockaddr_storage sockaddr = {0};
    sys_unpack_data_from_mrs(2, &sockaddr, socklen, 0);

    // 调用 lwIP connect
    err = lwip_connect(socket, (struct sockaddr *)&sockaddr, socklen);

    if (err == -1) {
        if (errno == EINPROGRESS) {
            // 非阻塞连接进行中
            sys_reply_with_ok_direct();
        } else {
            sys_reply_with_err_direct(errno);
        }
    } else {
        sys_reply_with_ok_direct();
    }
}
```

---

## 8. sys_close — 关闭 Socket

### 8.1 sys_close_nb

**文件**: `main.c`

```c
static int sys_close_nb(sel4_msg_info_t info, sel4_word badge)
{
    int socket = (int)sel4_get_mr(0);
    pid_t pid = sys_get_pid_from_badge(badge);

    // 检查 socket 是否属于此进程
    if (net_socket_info[socket].owner != pid) {
        err = EPERM;
        goto err_exit;
    }

    // 调用 lwIP close
    err = lwip_close(socket);

    if (err == -1) {
        sys_reply_with_err_direct(errno);
    } else {
        // 清理 socket 信息
        net_socket_info[socket].owner = 0;
        net_socket_info[socket].type = 0;
        sys_reply_with_ok_direct();
    }
}
```

---

## 9. Socket 信息管理

### 9.1 net_socket_info 结构

```c
typedef struct {
    pid_t owner;           // socket 所有者 PID
    int type;             // socket 类型 (NETCONN_TCP/UDP/RAW)
    int protocol;         // 协议
    void *conn;           // lwIP netconn 指针
    // ... 其他统计信息
} net_socket_info_t;

#define NUM_SOCKETS  256
static net_socket_info_t net_socket_info[NUM_SOCKETS];
```

### 9.2 获取 socket 类型

```c
static inline uint8_t get_netconn_type(int socket)
{
    struct lwip_sock *sock = get_socket(socket);
    if (sock == NULL) {
        return 0;
    }
    return NETCONNTYPE_GROUP(netconn_type(sock->conn));
}
```

---

## 10. Socket 选项

### 10.1 sys_setsockopt

**文件**: `main.c:2043**

```c
static int sys_setsockopt(sel4_msg_info_t info, sel4_word badge)
{
    int socket = (int)sel4_get_mr(0);
    int level = (int)sel4_get_mr(1);
    int optname = (int)sel4_get_mr(2);
    // ... optval 解析

    err = lwip_setsockopt(socket, level, optname, optval, optlen);

    if (err == -1) {
        sys_reply_with_err_direct(errno);
    } else {
        sys_reply_with_ok_direct();
    }
}
```

### 10.2 PACKET_RX_RING setsockopt

当 `optname == PACKET_RX_RING` 时，调用 `packet_mmap_set_ring()` 设置 AF_PACKET ring buffer。

---

## 11. 流程图

### 11.1 TCP Socket 创建和连接

```
应用                           NSv                          lwIP
 │                              │                            │
 │  SYS_NET_SOCKET             │                            │
 │─────────────────────────────►│                            │
 │                              │  lwip_socket()             │
 │                              │───────────────────────────►│
 │                              │◄───────────────────────────│
 │                              │                            │
 │  SYS_NET_BIND               │                            │
 │─────────────────────────────►│                            │
 │                              │  lwip_bind()               │
 │                              │───────────────────────────►│
 │                              │◄───────────────────────────│
 │                              │                            │
 │  SYS_NET_LISTEN             │                            │
 │─────────────────────────────►│                            │
 │                              │  lwip_listen()             │
 │                              │───────────────────────────►│
 │                              │◄───────────────────────────│
 │                              │                            │
 │  SYS_NET_CONNECT            │                            │
 │─────────────────────────────►│                            │
 │                              │  lwip_connect()            │
 │                              │───────────────────────────►│
 │                              │  (发送 SYN)                │
 │                              │◄───────────────────────────│
 │                              │                            │
```

### 11.2 TCP Accept

```
应用                           NSv                          lwIP
 │                              │                            │
 │  SYS_NET_ACCEPT             │                            │
 │─────────────────────────────►│                            │
 │                              │  lwip_accept()             │
 │                              │───────────────────────────►│
 │                              │◄───────────────────────────│
 │                              │  (3次握手完成，返回 new_socket) │
 │                              │                            │
 │  返回 new_socket             │                            │
 │◄─────────────────────────────│                            │
```

---

## 12. 与 Linux 对比

| 操作 | SafeOS NSv | Linux |
|------|-------------|-------|
| **socket** | seL4 IPC | 系统调用 |
| **bind** | seL4 IPC | 系统调用 |
| **listen** | seL4 IPC | 系统调用 |
| **accept** | seL4 IPC (可能阻塞) | 系统调用 (可能阻塞) |
| **connect** | seL4 IPC (非阻塞) | 系统调用 |
| **close** | seL4 IPC | 系统调用 |
| **send/recv** | seL4 IPC + shm | 系统调用 + DMA |

---

## 13. 总结

### 13.1 关键设计

1. **seL4 IPC**: 所有 socket 操作通过 IPC 消息分发
2. **lwIP 集成**: 使用 lwip_socket 等函数实现 BSD socket 到 netconn 的转换
3. **进程隔离**: 每个 socket 记录 owner PID，防止跨进程访问
4. **非阻塞支持**: 所有操作支持非阻塞模式

### 13.2 性能考虑

- **小数据**: 使用 message registers 直接传输
- **大数据**: 使用共享内存避免复制
- **批量操作**: select/poll 支持批量等待
