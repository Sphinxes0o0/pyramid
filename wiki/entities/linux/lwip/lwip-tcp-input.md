---
type: entity
tags: [linux, lwip, network, tcp]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP tcp_input Analysis

## 定义

`tcp_input()` 是 lwIP 的 **TCP 层入口函数**，负责 TCP header 解析、checksum 校验、demultiplex (找 PCB)、TCP 状态机处理 (tcp_process)、数据接收和 ACK 处理 (tcp_receive)。

## 调用链

```
ip4_input()
    │
    ▼
tcp_input(p, inp)
    │
    ├─► TCP Header 解析 (checksum, src/dst port, seqno, ackno, flags)
    ├─► Demultiplex — 找 PCB
    │     - tcp_active_pcbs (已连接)
    │     - tcp_tw_pcbs (TIME_WAIT)
    │     - tcp_listen_pcbs (监听中)
    ├─► tcp_process() — 状态机
    │     ├─► RST/SYN 处理
    │     ├─► tcp_receive() — 数据接收
    │     └─► tcp_output() — 发送响应
    └─► 回调通知 application
```

## TCP Demultiplex

```c
// 4.1 先查找 active_pcbs (已连接)
for (pcb = tcp_active_pcbs; pcb != NULL; pcb = pcb->next) {
    // 检查 netif 绑定
    if ((pcb->netif_idx != NETIF_NO_INDEX) &&
        (pcb->netif_idx != netif_get_index(ip_data.current_input_netif))) {
        continue;
    }
    // 匹配 4-tuple
    if (pcb->remote_port == tcphdr->src &&
        pcb->local_port == tcphdr->dest &&
        ip_addr_cmp(&pcb->remote_ip, ip_current_src_addr()) &&
        ip_addr_cmp(&pcb->local_ip, ip_current_dest_addr())) {
        // 移动到链表头部 (locality optimization)
        if (prev != NULL) {
            prev->next = pcb->next;
            pcb->next = tcp_active_pcbs;
            tcp_active_pcbs = pcb;
        }
        break;
    }
}
```

## TCP 状态转换

```
CLOSED → LISTEN → SYN_RCVD → ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED
              ↓           ↓
         SYN_SENT ───────┘
              ↓
         FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED
```

## 拥塞控制

```c
// 慢启动 vs 拥塞避免
cwnd < ssthresh: cwnd += min(acked, MSS)   // 每个 ACK cwnd 增加一个 MSS
cwnd >= ssthresh: cwnd += MSS * MSS / cwnd  // 每个 ACK cwnd 增加一小部分
```

## 与 Linux 对比

| 特性 | SafeOS lwIP | Linux |
|------|-------------|-------|
| **PCB 查找** | 链表遍历 O(n) | 哈希表 O(1) |
| **ooseq 队列** | 链表 | 红黑树 |
| **SACK** | 支持 | 支持 |

## 相关概念

- [[entities/linux/lwip/lwip-tcp-output]] — tcp_output 调用 tcp_input 响应
- [[entities/linux/lwip/lwip-tcp-pcb]] — TCP PCB 结构
- [[entities/linux/lwip/lwip-tcp-recv-queue]] — 接收队列管理
- [[entities/linux/lwip/lwip-tcp-socket]] — listen/accept/connect 流程
- [[entities/linux/lwip/lwip-ip4-input]] — 上游调用者
