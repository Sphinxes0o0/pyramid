---
type: entity
tags: [linux, lwip, network, tcp]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP TCP PCB Structure

## 定义

TCP PCB (Protocol Control Block) 是 lwIP 维护每个 TCP 连接状态的核心数据结构。

## TCP PCB 类型

| PCB 类型 | 描述 | 所在链表 |
|---------|------|---------|
| **tcp_listen_pcbs** | LISTEN 状态的 PCB | 单独链表 |
| **tcp_bound_pcbs** | 已绑定但未连接的 PCB | 单独链表 |
| **tcp_active_pcbs** | 活跃的 PCB (ESTABLISHED等) | 活跃链表 |
| **tcp_tw_pcbs** | TIME-WAIT 状态的 PCB | 时间等待链表 |

## TCP PCB 结构 (关键字段)

```c
struct tcp_pcb {
    // 基础字段
    IP_PCB;                    // IP 地址 + port
    TCP_PCB_COMMON;           // state, next, errf

    // TCP Flags
    tcpflags_t flags;
#define TF_ACK_DELAY   0x01U   // 延迟 ACK
#define TF_ACK_NOW    0x02U   // 立即发送 ACK
#define TF_NODELAY    0x40U   // 禁用 Nagle 算法
#define TF_WND_SCALE  0x0100U // 窗口缩放

    // 接收方变量
    u32_t rcv_nxt;            // 期望接收的下一个 seq
    tcpwnd_size_t rcv_wnd;    // 接收窗口大小

    // 重传定时器
    s16_t rtime;              // RTO 计时器 (当前值)
    u16_t mss;                // 最大段大小
    u32_t rttest;             // RTT 估计
    s16_t sa, sv;             // Van Jacobson RTT 估计
    s16_t rto;                // RTO 值

    // 拥塞控制
    tcpwnd_size_t cwnd;       // 拥塞窗口
    tcpwnd_size_t ssthresh;   // 慢启动阈值

    // Segment 队列
    struct tcp_seg *unsent;   // 未发送的 segment
    struct tcp_seg *unacked;   // 已发送未确认的 segment
    struct tcp_seg *ooseq;    // 接收到的乱序 segment
};
```

## Timer 管理

### tcp_fasttmr (25ms)
- 发送延迟 ACK
- 处理 FIN
- 处理被拒绝数据

### tcp_slowtmr (50ms)
- 重传超时
- SYN/DATA 最大重传
- FIN-WAIT-2 超时
- Keepalive
- OOSEQ 超时

## RTT 估计 (Van Jacobson 算法)

```c
// RTO = (sa >> 3) + sv
// 每次 RTT 采样后:
new_srtt = (7/8) * old_srtt + (1/8) * sample
new_rttvar = (3/4) * old_rttvar + (1/4) * |sample - srtt|
RTO = srtt + 4 * rttvar
```

## Segment 队列

```
unsent (未发送)
    ├── seg[0]: len=1000, seq=1000
    └── seg[1]: len=800, seq=2000

unacked (已发送未确认)
    └── seg[0]: len=1000, seq=1000 (等待 ACK)

ooseq (乱序接收)
    └── seg[0]: len=1000, seq=3000 (比期望的 seq=2000 提前)
```

## 相关概念

- [[entities/linux/lwip/lwip-tcp-input]] — 使用 tcp_pcb 进行 demultiplex
- [[entities/linux/lwip/lwip-tcp-output]] — 使用 tcp_pcb 的 cwnd/ssthresh
- [[entities/linux/lwip/lwip-tcp-recv-queue]] — rcv_nxt/rcv_wnd/ooseq
- [[entities/linux/lwip/lwip-tcp-socket]] — listen/accept/connect 修改 state
