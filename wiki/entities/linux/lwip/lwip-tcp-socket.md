---
type: entity
tags: [linux, lwip, network, tcp]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP TCP Socket Flow

## 定义

TCP 是面向连接的协议，连接建立需要三次握手，连接关闭需要四次挥手。

## TCP 3次握手

```
Client                           Server
  │                                │
  │ ─────── SYN (seq=x) ────────→│  LISTEN
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

## TCP 4次挥手

```
Client                           Server
  │                                │
  │ ─────── FIN (seq=u) ────────→│  ESTABLISHED
  │    状态 → FIN_WAIT_1         │  收到 FIN
  │                                │  状态 → CLOSE_WAIT
  │                                │
  │ ←────── ACK (ack=u+1) ───────│  CLOSE_WAIT
  │    状态 → FIN_WAIT_2          │
  │                           ● close()
  │                                │
  │ ←────── FIN (seq=w) ────────│  LAST_ACK
  │                                │
  │ ─────── ACK (ack=w+1) ──────→│  CLOSED
  │    状态 → TIME_WAIT (2MSL)
  │    等待后 → CLOSED
```

## Server 端: LISTEN / ACCEPT

```c
// tcp_listen_with_backlog_and_err
lpcb = memp_malloc(MEMP_TCP_PCB_LISTEN);
lpcb->state = LISTEN;
lpcb->local_port = pcb->local_port;
TCP_REG(&tcp_listen_pcbs.pcbs, (struct tcp_pcb *)lpcb);
```

## Client 端: CONNECT

```c
// tcp_connect
err_t tcp_connect(struct tcp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port) {
    ip_addr_set(&pcb->remote_ip, ipaddr);
    pcb->remote_port = port;
    netif = ip_route(&pcb->local_ip, &pcb->remote_ip);
    iss = tcp_next_iss(pcb);
    pcb->snd_nxt = iss;
    pcb->lastack = iss - 1;
    pcb->cwnd = 1;  // 初始 cwnd = 1 MSS
    ret = tcp_enqueue_flags(pcb, TCP_SYN);
    pcb->state = SYN_SENT;
    TCP_REG_ACTIVE(pcb);
    return ret;
}
```

## CLOSE 流程

```c
// tcp_close
err_t tcp_close(struct tcp_pcb *pcb) {
    if (pcb->state != LISTEN) {
        tcp_set_flags(pcb, TF_RXCLOSED);
    }
    return tcp_close_shutdown(pcb, 1);  // 关闭读写
}

// tcp_close_shutdown — 根据状态发送 FIN
switch (pcb->state) {
    case ESTABLISHED: tcp_close_shutdown_fin(pcb); break;  // → FIN_WAIT_1
    case CLOSE_WAIT: tcp_close_shutdown_fin(pcb); break;     // → LAST_ACK
}
```

## Socket API 对应关系

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

## 相关概念

- [[entities/linux/lwip/lwip-tcp-input]] — tcp_input 中的状态机处理
- [[entities/linux/lwip/lwip-tcp-pcb]] — TCP PCB 结构
- [[entities/linux/lwip/lwip-tcp-recv-queue]] — backlog 机制
- [[entities/linux/lwip/lwip-tcp-output]] — tcp_write/tcp_output 发送
