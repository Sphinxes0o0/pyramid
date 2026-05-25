---
type: entity
tags: [linux, lwip, network, tcpip, threading]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP tcpip_thread Analysis

## 定义

`tcpip_thread` 是 lwIP 的 **主协议栈线程**，负责从 mbox 接收消息 (API calls, packets, callbacks)、处理 timeout 事件、协调所有协议栈操作。

## SafeOS 中的使用

在 SafeOS 中，使用 **LWIP_TCPIP_CORE_LOCKING** 模式：
- RX path 直接在 `nic_rx_thread` 中调用 `LOCK_TCPIP_CORE()` 然后处理 packet
- tcpip_thread 主要处理 API calls、callbacks、timeouts

## 函数源码

```c
tcpip_thread(void *arg) {
    LOCK_TCPIP_CORE();
    if (tcpip_init_done != NULL) {
        tcpip_init_done(tcpip_init_done_arg);
    }

    while (1) {
        TCPIP_MBOX_FETCH(&tcpip_mbox, (void **)&msg);
        if (msg == NULL) continue;
        tcpip_thread_handle_msg(msg);
    }
}
```

## 消息类型

```c
typedef enum {
    TCPIP_MSG_API,           // BSD socket API 调用
    TCPIP_MSG_API_CALL,      // API 调用 (等待返回值)
    TCPIP_MSG_INPUT,         // packet 输入 (不使用 CORE_LOCKING 时)
    TCPIP_MSG_INPUT_ACK,     // packet 输入 (带确认)
    TCPIP_MSG_TIMEOUT,       // 注册 timeout
    TCPIP_MSG_UNTIMEOUT,    // 取消 timeout
    TCPIP_MSG_CALLBACK,      // 回调
    TCPIP_MSG_CALLBACK_STATIC // 静态回调
} tcpip_msg_type;
```

## LWIP_TCPIP_CORE_LOCKING 模式

### SafeOS 中的 RX 处理

```c
// nic_rx_thread 中
while (1) {
    union elem e = elem_ring_get(used_rx_buf_ring);
    if (e.pa) {
        LOCK_TCPIP_CORE();                    // ← 获取锁
        rx_callback((struct pbuf *)cma_pa_to_va(&cma, e.pa));
        UNLOCK_TCPIP_CORE();                // ← 释放锁
    }
}
```

### 模式对比

| 模式 | RX 处理位置 | 锁粒度 |
|------|-----------|--------|
| **LWIP_TCPIP_CORE_LOCKING=1** | nic_rx_thread | 整个协议栈处理持有锁 |
| **LWIP_TCPIP_CORE_LOCKING=0** | tcpip_thread (通过 TCPIP_MSG_INPUT) | packet 处理在 tcpip_thread 中 |

## 性能瓶颈分析

```
tcpip_thread 是单线程，所有 packet 处理串行化：

nic_rx_thread:
    LOCK_TCPIP_CORE()
    → rx_callback()
       → ethernet_input()
          → ip4_input()
             → udp_input() / tcp_input()
                → socket 接收
    UNLOCK_TCPIP_CORE()
```

在 4 核系统上：
- Core 0: tcpip_thread + nic_rx_thread (瓶颈)
- Core 1-3: 空闲或处理其他任务

## 相关概念

- [[entities/linux/lwip/lwip-network-init]] — tcpip_init 创建线程
- [[entities/linux/lwip/lwip-netif-add]] — 需要 CORE_LOCKED
- [[entities/linux/lwip/lwip-ethernet-input]] — nic_rx_thread 调用
- [[entities/linux/lwip/lwip-pbuf]] — mbox 传递的是 pbuf 指针
