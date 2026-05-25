---
type: entity
tags: [linux, lwip, network, nsv, send, recv, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# sys_net_sendto/recvfrom — Data Transmission

## 定义

`sys_net_sendto/recvfrom` 是 NSv 中的数据传输接口，支持两种模式：**直接数据传输**（seL4 IPC message registers，最多 512B）和**共享内存模式**（CMA，大块数据零拷贝）。

## 两种传输模式

### 模式 1: 共享内存模式 (use_shm = 1)
```c
if (use_shm) {
    send_buf = (char *)get_shm_va(pid, offset, data_len);
    if (!send_buf) { err = EINVAL; goto err_exit; }
}
```

### 模式 2: IPC 直接传输 (use_shm = 0)
```c
else {
    send_buf = (char *)alloc_data_cache();  // 最大 512B
    int cp_size = sys_unpack_data_from_mrs(5, send_buf, data_len, &next_mr);
}
```

## sys_sendto_nb — 非阻塞发送

```c
static int sys_sendto_nb(sel4_msg_info_t info, sel4_word badge)
{
    // 读取参数
    int socket = sel4_get_mr(0);
    int data_len = sel4_get_mr(1);
    int flags = sel4_get_mr(2);
    int use_shm = sel4_get_mr(4);
    unsigned long offset = sel4_get_mr(5);

    if (use_shm) {
        send_buf = get_shm_va(pid, offset, data_len);  // 共享内存
    } else {
        send_buf = alloc_data_cache();
        sys_unpack_data_from_mrs(5, send_buf, data_len, &next_mr);
    }

    sent = lwip_sendto(socket, send_buf, data_len, flags, addr, addrlen);

    if (sent == -1) {
        sys_reply_with_err_direct(errno);
    } else {
        sys_reply_with_one_direct(0, sent);  // 返回发送字节数
    }
}
```

## sys_recvfrom_nb — 非阻塞接收

```c
static int sys_recvfrom_nb(sel4_msg_info_t info, sel4_word badge)
{
    int socket = sel4_get_mr(0);
    int data_len = sel4_get_mr(1);
    int use_shm = sel4_get_mr(4);

    recv = lwip_recvfrom(socket, recv_buf, data_len, flags, &sockaddr, &socklen);

    if (!use_shm) {
        end_mr = sys_pack_data_to_mrs(0, recv_buf, recv);  // 数据放入 IPC mr
    } else {
        sel4_set_mr(0, recv);  // 只发送接收长度
        end_mr = 1;
    }
    sel4_reply(info);
}
```

## 数据传输流程

```
应用进程                           NSv 进程
    │                                 │
    │  seL4 IPC (badge + mr0-5)    │
    │───────────────────────────────► │
    │                                 │
    │  [use_shm=1]                   │
    │   - offset 指向共享内存         │
    │   - 直接写入共享内存           │
    │                                 │
    │  [use_shm=0]                   │
    │   - 数据在 message registers    │
    │   - sys_unpack_data_from_mrs() │
    │                                 │
    ▼                                 ▼
lwip_sendto()                      lwip_sendto()
    │                                 │
    └─────────────────────────────────┘
              DMA → NIC
```

## 性能对比

| 模式 | 最大数据 | 复制次数 | 延迟 |
|------|----------|----------|------|
| **shm** | 受 CMA 大小限制 | 0 (零复制) | 低 |
| **IPC** | 512B | 1 (IPC → cache) | 中 |

## 相关概念

- [[entities/linux/lwip/lwip-nsv-event-loop]] — NSv 事件循环
- [[entities/linux/lwip/lwip-cma-buffer]] — CMA 共享内存
- [[entities/linux/lwip/lwip-sel4-ipc]] — seL4 IPC 机制
- [[entities/linux/lwip/lwip-sys-net-socket-api]] — Socket API

## 来源详情

- [[sources/safeos-lwip-extensions]]
