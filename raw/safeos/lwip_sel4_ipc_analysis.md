# seL4 IPC 机制分析 — T-003

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: seL4 微内核 IPC 机制：notification、endpoint 通信、badge 机制

---

## 1. 概述

### 1.1 seL4 IPC 类型

在 SafeOS NSv 中，主要使用以下 seL4 IPC 机制：

| IPC 类型 | 用途 | 延迟 |
|----------|------|------|
| **seL4_Signal** | 单向通知 (NIC → NSv) | ~50-200ns |
| **seL4_Recv** | 阻塞接收 (NSv → NIC) | ~100-500ns |
| **seL4_Call** | 同步调用 (NSv ↔ NIC) | ~200-1000ns |
| **seL4_Send** | 同步发送 | ~100-400ns |
| **seL4_Reply** | 回复 (配合 Call) | ~50-150ns |

### 1.2 NSv 中的 IPC 端点

```c
// main.c:443-477
sel4_cptr  nic_rx_ntfn = CPTR_NULL;    // RX 通知 (notification)
static sel4_cptr nic_tx_ntfn = CPTR_NULL;   // TX 通知 (notification)
static sel4_cptr nsv_nic_ep = CPTR_NULL;    // NIC endpoint (接收 NIC 消息)

sel4_cptr  net_pm_ep = CPTR_NULL;           // PM endpoint
static sel4_cptr net_nic_ep_suspend = CPTR_NULL;  // 暂停通知
```

---

## 2. Notification 机制

### 2.1 什么是 seL4 Notification

Notification 是 seL4 的**单向异步通知机制**，类似于事件标志或信号量。

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Notification 对象                            │
├─────────────────────────────────────────────────────────────────────┤
│  - 可以被信号 (signal)                                               │
│  - 可以被等待 (wait/recv)                                           │
│  - 支持 badge (32-bit 状态)                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 TX 通知流程

**main.c:3874-3891** (`ethif_link_output`):

```c
err_t ethif_link_output(struct netif *netif, struct pbuf *q)
{
    // 放入 pending_tx_buf_ring
    ret = elem_ring_put(pending_tx_buf_ring, e);
    if (ret) return ret;

    // 检查是否需要通知 NIC
    if (was_empty || is_full || (pending_read == pending_read_before + 1)) {
        // 发送 notification 给 NIC
        sel4_signal(nic_tx_ntfn);
    }

    return ERR_OK;
}
```

**关键点**: `sel4_signal` 是**异步**的，立即返回，不等待 NIC 处理。

### 2.3 RX 接收流程

**main.c:4961-4995** (`nic_rx_thread`):

```c
static void *nic_rx_thread(void *arg)
{
    // 绑定 notification 到线程
    int err = sys_bind_ntfn(nic_rx_ntfn);
    if (err) { ... }

    while (1) {
        // 阻塞等待 notification
        info = seL4_Recv(nsv_nic_ep, &badge);
        label = sel4_msg_info_get_label(info);

        if (badge == 0) {
            // 有 RX 数据可读
            while (1) {
                union elem e = elem_ring_get(used_rx_buf_ring);
                if (e.pa) {
                    LOCK_TCPIP_CORE();
                    rx_callback((struct pbuf *)cma_pa_to_va(&cma, e.pa));
                    UNLOCK_TCPIP_CORE();
                } else {
                    break;
                }
            }
        }
    }
}
```

---

## 3. Endpoint 机制

### 3.1 什么是 seL4 Endpoint

Endpoint 是 seL4 的**同步通信端点**，用于 send/recv 操作。

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Endpoint 对象                               │
├─────────────────────────────────────────────────────────────────────┤
│  - 线程可以在上面等待 (seL4_Recv)                                      │
│  - 线程可以发送 (seL4_Send)                                          │
│  - 支持 badge (接收时可识别发送者)                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Endpoint 注册

**main.c:5018** (`create_nic_thread`):

```c
static int create_nic_thread(void)
{
    // 分配 endpoint
    err = sys_svc_alloc_endpoint(&nsv_nic_ep);
    if (err || nsv_nic_ep == CPTR_NULL) { ... }

    // 注册为 "nsv-nic-if" 服务
    // 其他进程可以通过这个名字找到这个 endpoint
    err = sys_svc_reg(NSV_NIC_INTERFACE_NAME, nsv_nic_ep, 0);
    //            ↑ 服务名
    //                ↑ endpoint cap
    //                    ↑ extra caps (0)

    // 创建 RX 线程
    err = sys_thread_create(nic_rx_thread, NULL, SYS_PRIO_NETSVR, 1, &nic_tid);
    ...
}
```

### 3.3 NIC 初始化通知

**main.c:4531-4536** (`nic_evt_loop` 初始化):

```c
// 绑定 RX notification
sel4_set_cap_recv_path(&ntfn_cpath);
sel4_msg_info_t info = { 0 };
info = sel4_msg_info_set_label(info, NSV_PFE_NTFN_INIT);
sel4_set_cap(0, nic_rx_ntfn);  // notification 作为 extra cap
info = sel4_msg_info_set_extra_caps(info, 1);
info = sel4_msg_info_set_length(info, 0);
info = sel4_call(nic_ep, info);  // 同步调用 NIC
```

---

## 4. Badge 机制

### 4.1 Badge 的作用

Badge 是一个 32-bit 值，可以：
1. **发送时**: 携带 badge 到接收方
2. **接收时**: 接收方通过 badge 识别发送者

### 4.2 NSv 中的 Badge 使用

**main.c:4974-4976**:

```c
info = seL4_Recv(nsv_nic_ep, &badge);
// badge 携带了发送者的信息

if (unlikely((badge == NET_PM_NIC_RX_BADGE) && (label == NET_PM_SUSPEND))) {
    // NIC 收到暂停命令
    net_thread_suspend();
} else if (badge == 0) {
    // 正常的 RX 数据
    ...
} else if (label == SYS_DEV_DRV_RESTART) {
    // NIC 重启请求
    ...
}
```

### 4.3 Badge 的来源

Badge 在 endpoint/cap 层面设置，通常是：
- **Endpoint 的权限**: 发送时携带 endpoint 的 badge
- **Notification**: 信号时可以携带 badge

---

## 5. Syscall 接口

### 5.1 NSv 中的 Syscall 封装

```c
// 通知相关
sys_bind_ntfn(nic_rx_ntfn)      // 绑定 notification 到线程
sel4_signal(nic_tx_ntfn)          // 发送 notification

// Endpoint 相关
sys_svc_alloc_endpoint(&ep)      // 分配 endpoint
sys_svc_reg(name, ep, extra)     // 注册服务
sys_kobj_alloc(KOBJ_TYPE_NTFN)   // 分配 notification 对象
```

### 5.2 消息传递

```c
// 发送消息 (seL4_Send/Call)
sel4_set_mr(i, value);           // 设置消息寄存器 i
sel4_call(ep, info);             // 同步调用

// 接收消息 (seL4_Recv)
sel4_get_mr(i);                   // 读取消息寄存器 i
sel4_msg_info_get_label(info);   // 获取标签 (message info)

// 回复
sel4_reply(info);                 // 发送回复
```

### 5.3 MSG Info 结构

```c
typedef struct {
    uint64_t label: 12;      // 标签 (方法 ID)
    uint64_t length: 7;       // 消息长度 (words)
    uint64_t extra_caps: 7;  // 额外 cap 数量
    uint64_t unused: 38;
} sel4_msg_info_t;
```

---

## 6. NIC RX/TX 完整流程

### 6.1 RX 路径

```
NIC Driver                              NSv (lwIP)
─────────────                            ────────────
1. DMA 完成，写入 empty_rx_buf
2. elem_ring_put(used_rx_buf, e)
3. sel4_signal(nic_rx_ntfn)  ──────► seL4_Recv(nsv_nic_ep, &badge)
                                          │
                                          ▼
                                     badge == 0
                                          │
                                          ▼
                                     elem_ring_get(used_rx_buf)
                                          │
                                          ▼
                                     cma_pa_to_va(e.pa)
                                          │
                                          ▼
                                     rx_callback(pbuf)
                                          │
                                          ▼
                                     ethernet_input()
```

### 6.2 TX 路径

```
NSv (lwIP)                              NIC Driver
─────────────                            ────────────
1. ethernet_output(pbuf)                 (等待)
2. ethif_link_output()
3. elem_ring_put(pending_tx_buf, e)
4. sel4_signal(nic_tx_ntfn)  ─────────► 收到 notification
                                              │
                                              ▼
                                         elem_ring_get(pending_tx)
                                              │
                                              ▼
                                         DMA 读取 buffer
                                              │
                                              ▼
                                         elem_ring_put(used_tx_buf)
                                              │
                                              ▼
                                         (可选) TX 完成 notification
```

---

## 7. 性能特征

### 7.1 seL4 IPC 延迟参考

| 操作 | 延迟 (ARM Cortex-A57 @ 2GHz) |
|------|------------------------------|
| `sel4_signal` | ~50-200ns |
| `seL4_Recv` (无消息) | ~100-500ns (阻塞) |
| `seL4_Recv` (有消息) | ~50-100ns |
| `seL4_Call` | ~200-1000ns |
| `sel4_reply` | ~50-150ns |

### 7.2 单 packet IPC 开销

```
RX path:
sel4_signal + seL4_Recv ≈ 150-700ns per packet

TX path:
sel4_signal ≈ 50-200ns per packet (异步，立即返回)
```

### 7.3 优化方向

1. **批量处理**: 一次 notification 后处理多个 packet
2. **异步 TX**: TX 不等待完成，依赖 periodic reclaim
3. **Notification 合并**: 多次 RX 合并为一次 notification

---

## 8. 与其他模块的关系

### 8.1 上游调用者

| 模块 | 函数 | 调用目的 |
|------|------|----------|
| **ethif_link_output** | `sel4_signal` | 通知 NIC 有 TX 数据 |
| **nic_evt_loop** | `sys_bind_ntfn` | 绑定 RX notification |
| **create_nic_thread** | `sys_svc_reg` | 注册 NIC endpoint |

### 8.2 下游被调用者

| 模块 | 函数 | 说明 |
|------|------|------|
| **elem_ring** | `elem_ring_put/get` | 数据传递 |
| **CMA** | `cma_pa_to_va` | 地址转换 |

---

## 9. 总结

### 9.1 IPC 架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                              NSv                                    │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  nic_rx_thread:                                              │   │
│  │      seL4_Recv(nsv_nic_ep, &badge) ──► 等待 NIC 通知        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  ethif_link_output:                                          │   │
│  │      sel4_signal(nic_tx_ntfn) ──► 通知 NIC 有数据            │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                    │                              ▲
        sel4_signal │                              │ seL4_Recv
                    ▼                              │
┌─────────────────────────────────────────────────────────────────────┐
│                          NIC Driver                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.2 关键设计点

1. **异步通知**: TX 使用 `sel4_signal` 不阻塞
2. **同步接收**: RX 使用 `seL4_Recv` 阻塞等待
3. **Badge 机制**: 通过 badge 识别消息类型
4. **Endpoint 注册**: 通过 `sys_svc_reg` 暴露服务

### 9.3 性能影响

- **IPC 开销**: ~150-700ns per packet (RX)
- **主要瓶颈**: 单 tcpip_thread 处理 + seL4 IPC
- **优化方向**: 批处理、异步化
