# TCP Socket 流程分析 — T-032

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: TCP listen/accept/connect/close 流程

---

## 1. 概述

TCP 是面向连接的协议，连接建立需要三次握手，连接关闭需要四次挥手。本文档分析 lwIP 中 socket 操作的完整流程。

### 1.1 TCP 状态机回顾

```
CLOSED → SYN_SENT (client)        LISTEN → SYN_RCVD (server)
    │                                   │
    │ ←────────────── ACK ──────────────│
    │                                   │
    ↓                                   ↓
 ESTABLISHED ←──────────────────── ESTABLISHED
    │                                   │
    │           FIN_WAIT_1              │  ← close()
    │               │                   │
    │     ┌─────────┴─────────┐        │
    │     ↓                   ↓        │  ← close()
    │  FIN_WAIT_2         CLOSING      │
    │     │                   │        │
    │     └─────────┬─────────┘        │
    │               ↓                   │
    │           TIME_WAIT              │
    │               │                   │
    └──────→ CLOSE_WAIT ←──────────────┘
                │
                └──────────→ LAST_ACK
                                │
                                └──────────→ CLOSED
```

---

## 2. Server 端: LISTEN / ACCEPT

### 2.1 tcp_listen_with_backlog_and_err — 监听

**文件**: `core/tcp.c:855-930`

```c
struct tcp_pcb *tcp_listen_with_backlog_and_err(struct tcp_pcb *pcb,
                                                u8_t backlog, err_t *err)
{
    struct tcp_pcb_listen *lpcb;

    // ============================================
    // Step 1: 分配 listen PCB
    // ============================================
    lpcb = (struct tcp_pcb_listen *)memp_malloc(MEMP_TCP_PCB_LISTEN);
    if (lpcb == NULL) {
        return NULL;  // ERR_MEM
    }

    // ============================================
    // Step 2: 复制 PCB 信息到 listen PCB
    // ============================================
    lpcb->callback_arg = pcb->callback_arg;
    lpcb->priority = pcb->priority;
    lpcb->local_port = pcb->local_port;
    lpcb->state = LISTEN;
    ip_addr_copy(lpcb->local_ip, pcb->local_ip);

    // ============================================
    // Step 3: 释放原 PCB
    // ============================================
    tcp_free(pcb);

    // ============================================
    // Step 4: 设置 backlog
    // ============================================
#if TCP_LISTEN_BACKLOG
    lpcb->accepts_pending = 0;
    tcp_backlog_set(lpcb, backlog);
#endif

    // ============================================
    // Step 5: 注册到 listen PCB 链表
    // ============================================
    TCP_REG(&tcp_listen_pcbs.pcbs, (struct tcp_pcb *)lpcb);

    return (struct tcp_pcb *)lpcb;
}
```

### 2.2 TCP 3次握手 (Server 侧)

```
Client                           Server
  │                                │
  │ ──────── SYN (seq=x) ────────→│  LISTEN
  │                                │
  │         ● 创建新 PCB           │
  │         ● 状态 → SYN_RCVD     │
  │                                │
  │ ←────── SYN+ACK (seq=y,ack=x+1)│  SYN_RCVD
  │                                │
  │ ─────── ACK (ack=y+1) ──────→│  ESTABLISHED
  │                                │
  │                           ● 调用 accept 回调
```

### 2.3 tcp_input 中的 SYN_RCVD 处理

**文件**: `core/tcp_in.c:924-959`

```c
case SYN_RCVD:
    if (flags & TCP_ACK) {
        if (TCP_SEQ_BETWEEN(ackno, pcb->lastack + 1, pcb->snd_nxt)) {
            pcb->state = ESTABLISHED;

            // 调用 backlog accepted
            tcp_backlog_accepted(pcb);

            // 调用 accept 回调
            TCP_EVENT_ACCEPT(pcb->listener, pcb, pcb->callback_arg, ERR_OK, err);

            if (err != ERR_OK) {
                tcp_abort(pcb);  // 拒绝连接
                return ERR_ABRT;
            }
        }
    }
```

---

## 3. Client 端: CONNECT

### 3.1 tcp_connect — 发起连接

**文件**: `core/tcp.c:1076-1200`

```c
err_t tcp_connect(struct tcp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port,
                  tcp_connected_fn connected)
{
    struct netif *netif;
    u32_t iss;

    // ============================================
    // Step 1: 设置远端地址
    // ============================================
    ip_addr_set(&pcb->remote_ip, ipaddr);
    pcb->remote_port = port;

    // ============================================
    // Step 2: 路由查找
    // ============================================
    netif = ip_route(&pcb->local_ip, &pcb->remote_ip);
    if (netif == NULL) {
        return ERR_RTE;  // 无路由
    }

    // ============================================
    // Step 3: 选择源地址
    // ============================================
    if (ip_addr_isany(&pcb->local_ip)) {
        const ip_addr_t *local_ip = ip_netif_get_local_ip(netif, ipaddr);
        ip_addr_copy(pcb->local_ip, *local_ip);
    }

    // ============================================
    // Step 4: 分配本地端口
    // ============================================
    if (pcb->local_port == 0) {
        pcb->local_port = tcp_new_port();
    }

    // ============================================
    // Step 5: 初始化序列号
    // ============================================
    iss = tcp_next_iss(pcb);
    pcb->rcv_nxt = 0;
    pcb->snd_nxt = iss;
    pcb->lastack = iss - 1;

    // ============================================
    // Step 6: 初始化拥塞窗口
    // ============================================
    pcb->cwnd = 1;  // 初始 cwnd = 1 MSS

    // ============================================
    // Step 7: 设置 connected 回调
    // ============================================
    pcb->connected = connected;

    // ============================================
    // Step 8: 发送 SYN
    // ============================================
    ret = tcp_enqueue_flags(pcb, TCP_SYN);
    if (ret == ERR_OK) {
        pcb->state = SYN_SENT;
        TCP_REG_ACTIVE(pcb);  // 加入活跃 PCB 链表
    }

    return ret;
}
```

### 3.2 TCP 3次握手 (Client 侧)

```
Client                           Server
  │                                │
  │ ─────── SYN (seq=x) ────────→│  LISTEN
  │                                │
  │    状态 → SYN_SENT            │
  │                                │
  │ ←────── SYN+ACK (seq=y,ack=x+1)│  SYN_RCVD
  │                                │
  │ ─────── ACK (ack=y+1) ──────→│  ESTABLISHED
  │    状态 → ESTABLISHED         │
  │                           ● 调用 connected 回调
  │                                │
  │ ←─────── Data ───────────────→│  ESTABLISHED
  │    ● 调用 recv 回调            │
```

---

## 4. ACCEPT 回调

### 4.1 tcp_accept — 设置 accept 回调

**文件**: `core/tcp.c:2158-2166`

```c
void tcp_accept(struct tcp_pcb *pcb, tcp_accept_fn accept)
{
    if (pcb->state == LISTEN) {
        struct tcp_pcb_listen *lpcb = (struct tcp_pcb_listen *)pcb;
        lpcb->accept = accept;  // 设置回调
    }
}
```

### 4.2 Backlog 机制

```c
// TCP_LISTEN_BACKLOG 时使用
tcp_backlog_set(lpcb, backlog);  // 设置半连接队列长度
tcp_backlog_delayed(pcb);  // 延迟接受
tcp_backlog_accepted(pcb);  // 完成接受
```

---

## 5. CLOSE 流程

### 5.1 tcp_close — 关闭连接

**文件**: `core/tcp.c:486-501`

```c
err_t tcp_close(struct tcp_pcb *pcb)
{
    if (pcb->state != LISTEN) {
        tcp_set_flags(pcb, TF_RXCLOSED);  // 不再接收数据
    }
    return tcp_close_shutdown(pcb, 1);  // 关闭读写
}
```

### 5.2 tcp_close_shutdown — 实际关闭

```c
err_t tcp_close_shutdown(struct tcp_pcb *pcb, u8_t rst_on_unacked_data)
{
    // ...

    // 根据状态决定如何关闭
    switch (pcb->state) {
        case CLOSED:
            tcp_free(pcb);
            break;

        case LISTEN:
            tcp_listen_closed(pcb);
            tcp_free(pcb);
            break;

        case SYN_SENT:
            tcp_free(pcb);
            break;

        case SYN_RCVD:
            // 发送 FIN，进入 FIN_WAIT_1
            tcp_close_shutdown_fin(pcb);
            break;

        case ESTABLISHED:
            // 发送 FIN，进入 FIN_WAIT_1
            tcp_close_shutdown_fin(pcb);
            break;

        case CLOSE_WAIT:
            // 发送 FIN，进入 LAST_ACK
            tcp_close_shutdown_fin(pcb);
            break;

        // ...
    }

    return ERR_OK;
}
```

### 5.3 四次挥手

```
Client                           Server
  │                                │
  │ ─────── FIN (seq=u) ────────→│  ESTABLISHED
  │    状态 → FIN_WAIT_1         │  收到 FIN
  │                                │  状态 → CLOSE_WAIT
  │                                │  ● 调用 recv 回调 (返回 0)
  │                                │
  │ ←────── ACK (ack=u+1) ───────│  CLOSE_WAIT
  │    状态 → FIN_WAIT_2          │
  │                           ● 调用 close()
  │                                │
  │                           ─────┘
  │                           │ 状态 → LAST_ACK
  │ ←────── FIN (seq=w) ───────│  LAST_ACK
  │                                │
  │ ─────── ACK (ack=w+1) ──────→│  CLOSED
  │    状态 → TIME_WAIT (2MSL)
  │    等待后 → CLOSED
```

---

## 6. Socket API 对应关系

### 6.1 lwIP Raw API vs Socket API

| Socket API | Raw API | 说明 |
|-----------|---------|------|
| `socket()` | `tcp_new()` | 创建 PCB |
| `bind()` | `tcp_bind()` | 绑定地址端口 |
| `listen()` | `tcp_listen_with_backlog()` | 开始监听 |
| `accept()` | callback `accept` | 接受连接 |
| `connect()` | `tcp_connect()` | 发起连接 |
| `write()` | `tcp_write()` | 发送数据 |
| `read()` | callback `recv` | 接收数据 |
| `close()` | `tcp_close()` | 关闭连接 |

### 6.2 lwIP Socket API 实现

**文件**: `api/sockets.c`

```c
// listen 实现
int lwip_listen(int s, int backlog)
{
    // ...
    ndb->pcb.tcp = tcp_listen_with_backlog_and_err(ndb->pcb.tcp, backlog, &err);
    // ...
}

// connect 实现
int lwip_connect(int s, const struct sockaddr *name, socklen_t namelen)
{
    // ...
    err = tcp_connect(ndb->pcb.tcp, addr, port, connected_callback);
    // ...
}
```

---

## 7. 连接状态转换总结

### 7.1 Server 侧

```
tcp_new() → tcp_bind() → tcp_listen()
    │                           │
    │                      LISTEN (监听)
    │                           │
    │                    tcp_input()
    │                           │
    │              ┌───────────┴───────────┐
    │              ↓                       ↓
    │         SYN_RCVD               (忽略其他)
    │              │
    │      tcp_backlog_accepted()
    │              │
    │      TCP_EVENT_ACCEPT()
    │              │
    │              ↓
    │         ESTABLISHED
    │              │
    │         tcp_close()
    │              │
    │      tcp_close_shutdown_fin()
    │              │
    │              ↓
    │         FIN_WAIT_1
    │              │
    │      ACK ←───┘
    │              │
    │              ↓
    │         FIN_WAIT_2
    │              │
    │      FIN ←───┘
    │              │
    │              ↓
    │         TIME_WAIT
    │              │
    │      (2MSL 超时)
    │              │
    └──────────→ CLOSED
```

### 7.2 Client 侧

```
tcp_new() → tcp_bind() → tcp_connect()
    │                           │
    │                      SYN_SENT
    │                           │
    │              ┌───────────┴───────────┐
    │              ↓                       ↓
    │         ESTABLISHED              (超时/失败)
    │              │                       │
    │      TCP_EVENT_CONNECTED()            │
    │              │                   → CLOSED
    │              │
    │         tcp_close()
    │              │
    │      tcp_close_shutdown_fin()
    │              │
    │              ↓
    │         FIN_WAIT_1
    │              │
    │      ACK ←───┐
    │              │
    │      ┌───────┴───────┐
    │      ↓               ↓
    │  FIN_WAIT_2      CLOSING
    │      │               │
    │      └───────┬───────┘
    │              ↓
    │         TIME_WAIT
    │              │
    │      (2MSL 超时)
    │              │
    └──────────→ CLOSED
```

---

## 8. 总结

### 8.1 连接建立 (3次握手)

```
Client                          Server
  │                               │
  │ ─── SYN (seq=x) ────────────→│
  │                               │
  │ ←── SYN+ACK (seq=y,ack=x+1) ─│
  │                               │
  │ ─── ACK (ack=y+1) ──────────→│
  │                               │
  ● connected 回调                 ● accept 回调
```

### 8.2 连接关闭 (4次挥手)

```
Client                          Server
  │                               │
  │ ─── FIN (seq=u) ────────────→│
  │   FIN_WAIT_1                  │  CLOSE_WAIT
  │                               │
  │ ←── ACK (ack=u+1) ──────────│
  │   FIN_WAIT_2                  │
  │                               │
  │                           ● close()
  │                               │
  │ ←── FIN (seq=w) ─────────────│  LAST_ACK
  │   TIME_WAIT                   │
  │                               │
  │ ─── ACK (ack=w+1) ──────────→│  CLOSED
  │   (等待 2MSL)                 │
  │   CLOSED                      │
```

### 8.3 关键设计

1. **listen PCB vs connection PCB**: listen PCB 是特殊的 `tcp_pcb_listen`，连接使用 `tcp_pcb`
2. **backlog**: 半连接队列长度限制
3. **状态转换**: TCP 状态机是连接管理的核心
4. **回调机制**: lwIP 使用 callback 而不是同步返回
