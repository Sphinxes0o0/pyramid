---
type: entity
tags: [lwip, tcp, source, transport-layer]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# tcp.c — TCP Core (Setup, Timers, Close)

> TCP 核心：连接建立、状态机、定时器、close/abort

## 文件概览

| 属性 | 值 |
|------|-----|
| 路径 | `src/core/tcp.c` |
| 行数 | 2768 |
| 功能 | TCP PCB 管理、连接 setup/teardown、定时器、listen/backlog |
| 依赖 | pbuf, ip, tcp_in, tcp_out |

## 函数索引

### PCB 管理
| 函数 | 行号 | 功能 |
|------|------|------|
| `tcp_new` | — | 创建新 TCP PCB (ALLOC) |
| `tcp_bind` | 663 | 绑定 PCB 到本地 addr:port |
| `tcp_listen` | — | 进入 LISTEN 状态 |
| `tcp_listen_with_backlog` | — | 带 backlog 的 listen |
| `tcp_connect` | — | 发起主动连接 (SYN_SENT) |
| `tcp_close` | 486 | 关闭连接 (发送 FIN) |
| `tcp_shutdown` | 517 | 部分关闭 (rx/tx) |
| `tcp_abort` | 640 | 强制 abort (发 RST) |
| `tcp_free` | 210 | 释放 TCP PCB |
| `tcp_free_listen` | 221 | 释放 listen PCB |

### 定时器
| 函数 | 行号 | 功能 |
|------|------|------|
| `tcp_tmr` | 234 | 主定时器入口 (25ms fast / 50ms slow) |
| `tcp_fasttmr` | — | Fast timer (25ms, 探测零窗口) |
| `tcp_slowtmr` | — | Slow timer (50ms, 重传/保持/关闭) |

### Listen/Backlog
| 函数 | 行号 | 功能 |
|------|------|------|
| `tcp_backlog_delayed` | 294 | 延迟接受 (backlog pending) |
| `tcp_backlog_accepted` | 317 | backlog accepted |
| `tcp_remove_listener` | 251 | 从 active pcb 移除 listener 引用 |
| `tcp_listen_closed` | 269 | listen pcb 关闭时清理 |

### 内部
| 函数 | 行号 | 功能 |
|------|------|------|
| `tcp_close_shutdown` | 348 | close 的核心逻辑 (RST on unacked) |
| `tcp_close_shutdown_fin` | 410 | 发送 FIN 并转换状态 |
| `tcp_abandon` | 565 | abandon 连接 (可选发 RST) |
| `tcp_new_port` | — | 分配新本地端口 |
| `tcp_init` | 201 | 模块初始化 |

## 关键数据结构

### TCP PCB Lists (全局)
```c
struct tcp_pcb *tcp_bound_pcbs;    // 绑定但未连接/listen
union tcp_listen_pcbs_t tcp_listen_pcbs;  // LISTEN 状态
struct tcp_pcb *tcp_active_pcbs; // 活动连接
struct tcp_pcb *tcp_tw_pcbs;     // TIME_WAIT 状态
```

### TCP State (enum)
```
CLOSED → LISTEN → SYN_SENT → SYN_RCVD → ESTABLISHED
                                         ↓
         FIN_WAIT_1 ← ← ← ← ← ← ← ← ← ← ← ← ←
              ↓                ↑
         FIN_WAIT_2         CLOSE_WAIT
              ↓                ↓
         CLOSING ← ← ← ← ← ← LAST_ACK
              ↓
         TIME_WAIT
```

## 调用链

### 连接建立 (主动)
```
tcp_connect()
  → tcp_new()
  → tcp_bind()
  → tcp_output()  // 发送 SYN
  → TCP 状态: SYN_SENT
```

### 连接建立 (被动)
```
tcp_listen()
  → tcp_new()
  → 插入 tcp_listen_pcbs
  → TCP 状态: LISTEN
```

### 关闭连接
```
tcp_close()
  → tcp_set_flags(TF_RXCLOSED)
  → tcp_close_shutdown(pcb, 1)
    → [有未读数据] → tcp_rst() → tcp_free()
    → [ESTABLISHED] → tcp_send_fin() → FIN_WAIT_1
    → [CLOSE_WAIT] → LAST_ACK
```

### 定时器调度
```
tcp_tmr() (由 sys_timer 调用)
  → tcp_fasttmr()  // 每 25ms
  → tcp_slowtmr()  // 每 50ms (++tcp_timer & 1)
```

## 交叉引用

### Analysis 层
- [[entities/linux/lwip/lwip-tcp-pcb]] — TCP PCB 结构详解
- [[entities/linux/lwip/lwip-tcp-input]] — TCP 输入/状态机
- [[entities/linux/lwip/lwip-tcp-output]] — TCP 输出/拥塞窗口
- [[entities/linux/lwip/lwip-tcp-socket]] — Socket API 封装
- [[entities/linux/lwip/lwip-tcp-recv-queue]] — 接收队列/窗口

### 定时器
- [[entities/linux/lwip/lwip-tcpip-thread]] — tcpip_thread 调度

### 安全
- [[entities/linux/lwip/lwip-firewall]] — LWFW
