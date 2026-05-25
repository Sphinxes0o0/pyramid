---
type: entity
tags: [linux, lwip, network, tcp]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP tcp_output Analysis

## 定义

`tcp_output()` 是 lwIP 的 **TCP 发送主函数**，负责计算可用发送窗口 (snd_wnd, cwnd)、应用 Nagle 算法、发送窗口内的 segment、管理重传定时器。

## 调用链

```
tcp_write() — 应用层数据写入发送缓冲区
    │
    ▼
tcp_output() — 发送数据
    │
    ├─► tcp_route() — 路由查找
    ├─► Nagle 算法检查
    ├─► tcp_output_segment() — 发送单个 segment
    │     └─► ip_output_if() → ip4_output_if()
    │           └─► LWFW egress_filter
    │
    └─► 管理 unacked 队列
```

## 核心逻辑

```c
err_t tcp_output(struct tcp_pcb *pcb) {
    // 计算发送窗口
    wnd = LWIP_MIN(pcb->snd_wnd, pcb->cwnd);

    // 遍历 unsent 队列
    while (seg != NULL && seg_seq_in_window) {
        // Nagle 算法检查
        if (tcp_do_output_nagle(pcb) == 0) {
            break;  // Nagle 阻止发送
        }

        // 设置 ACK 标志
        TCPH_SET_FLAG(seg->tcphdr, TCP_ACK);

        // 发送 segment
        tcp_output_segment(seg, pcb, netif);

        // 移动到 unacked 队列
        pcb->unsent = seg->next;
        seg = pcb->unsent;
    }
    return ERR_OK;
}
```

## Nagle 算法

减少小包数量：
- 如果有未确认的数据 (unacked 不为空) 且有小数据 (< MSS)，则等待

## 拥塞控制

```c
// 初始拥塞窗口 (IW) — RFC 2581
cwnd = LWIP_MIN(4 * MSS, max(2 * MSS, 4380))

// 慢启动
if (cwnd < ssthresh) {
    cwnd += MSS;
}

// 拥塞避免
if (cwnd >= ssthresh) {
    cwnd += MSS * MSS / cwnd;
}
```

## 快速重传

当收到 **3 个重复 ACK** (dupacks >= 3)：
```c
if (pcb->dupacks >= 3) {
    tcp_rexmit_fast(pcb);  // 快速重传丢失的 segment
}
```

## 相关概念

- [[entities/linux/lwip/lwip-tcp-input]] — tcp_output 在 tcp_input 中被调用响应
- [[entities/linux/lwip/lwip-tcp-pcb]] — TCP PCB 结构中的 cwnd/ssthresh
- [[entities/linux/lwip/lwip-ip4-output]] — 使用 ip4_output_if 发送
- [[entities/linux/lwip/lwip-routing]] — tcp_route 路由查找
