# sys_net_sendto/recvfrom 分析 — T-102

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: 数据传输、共享内存优化、shm 机制

---

## 1. 概述

sys_net_sendto/recvfrom 是 NSv 中的数据传输接口，支持两种数据传输模式：

1. **直接数据传输**: 通过 seL4 IPC message registers (最多 512B)
2. **共享内存模式**: 通过 CMA 共享内存 (大块数据)

---

## 2. 数据结构

### 2.1 Socket 信息表

```c
typedef struct {
    pid_t owner;              // socket 所有者 PID
    int type;                 // socket 类型 (NETCONN_TCP/UDP/RAW/PACKET)
    int protocol;
    void *conn;              // lwIP netconn 指针
    // ...
} net_socket_info_t;
```

### 2.2 Data Cache

用于小数据块的临时缓存：

```c
#define DATA_CACHE_SIZE  512

static char net_local_data_tx[DATA_CACHE_SIZE];  // TX 缓存
static char net_local_data_rx[DATA_CACHE_SIZE];  // RX 缓存
```

---

## 3. sys_sendto_nb — 非阻塞发送

**文件**: `main.c:1274-1362`

### 3.1 函数签名

```c
static int sys_sendto_nb(sel4_msg_info_t info, sel4_word badge)
```

### 3.2 参数读取

```c
int socket = (int)sel4_get_mr(0);      // socket 描述符
int data_len = (int)sel4_get_mr(1);    // 数据长度
int flags = (int)sel4_get_mr(2);       // 标志
socklen_t socklen = (socklen_t)sel4_get_mr(3);  // 地址长度
int use_shm = (int)sel4_get_mr(4);     // 是否使用共享内存
unsigned long offset = sel4_get_mr(5);  // shm 偏移
```

### 3.3 两种传输模式

#### 模式 1: 共享内存模式 (use_shm = 1)

```c
if (use_shm) {
    // 从共享内存获取发送缓冲区
    send_buf = (char *)get_shm_va(pid, offset, data_len);
    if (!send_buf) {
        err = EINVAL;
        goto err_exit;
    }
}
```

#### 模式 2: IPC 直接传输 (use_shm = 0)

```c
else {
    // 使用 data cache (最大 512B)
    send_buf = (char *)alloc_data_cache();
    if (send_buf == NULL) {
        err = ENOMEM;
        goto err_exit;
    }

    // 从 seL4 IPC message registers 复制数据
    int cp_size = sys_unpack_data_from_mrs(5, (void *)send_buf, data_len, &next_mr);
    if (cp_size != data_len) {
        err = EINVAL;
        goto err_exit;
    }
}
```

### 3.4 调用 lwIP

```c
if (use_sockaddr) {
    // 带地址的发送
    sent = lwip_sendto(socket, send_buf, data_len, flags,
                       (const struct sockaddr *)&sockaddr, socklen);
} else {
    // 不带地址的发送
    sent = lwip_sendto(socket, send_buf, data_len, flags, 0, 0);
}
```

### 3.5 响应

```c
if (sent == -1) {
    err = errno;
    NSV_PERPID_STATS_INC(pid, net_socket_info[socket].type, sendto_fail_cnt);
    sys_reply_with_err_direct(err);
} else {
    sys_reply_with_one_direct(0, sent);  // 返回发送的字节数
    NSV_PERPID_STATS_INC(pid, net_socket_info[socket].type, sendto_success_cnt);
}
```

---

## 4. sys_recvfrom_nb — 非阻塞接收

**文件**: `main.c:1368-1455`

### 4.1 参数读取

```c
int socket = (int)sel4_get_mr(0);      // socket 描述符
int data_len = (int)sel4_get_mr(1);    // 接收缓冲区长度
int flags = (int)sel4_get_mr(2);       // 标志
socklen_t socklen = (socklen_t)sel4_get_mr(3);  // 地址长度
int use_shm = (int)sel4_get_mr(4);     // 是否使用共享内存
size_t offset = (int)sel4_get_mr(5);   // shm 偏移
```

### 4.2 两种接收模式

#### 共享内存模式 (use_shm = 1)

```c
if (use_shm) {
    vaddr_t shm_va = get_shm_va(pid, offset, data_len);
    recv_buf = (char *)shm_va;
}
```

#### IPC 直接接收 (use_shm = 0)

```c
else {
    recv_buf = (char *)alloc_data_cache();
}
```

### 4.3 调用 lwIP

```c
if (socklen) {
    recv = lwip_recvfrom(socket, recv_buf, data_len, flags,
                         (struct sockaddr *)&sockaddr, &socklen);
} else {
    recv = lwip_recvfrom(socket, recv_buf, data_len, flags, 0, 0);
}
```

### 4.4 响应封装

```c
if (recv == -1) {
    sys_reply_with_err_direct(errno);
} else {
    if (!use_shm) {
        // 将数据打包到 IPC message registers
        end_mr = sys_pack_data_to_mrs(0, recv_buf, recv);
    } else {
        sel4_set_mr(0, recv);  // 只发送接收长度
        end_mr = 1;
    }

    // 打包源地址
    if (socklen != 0) {
        end_mr = sys_pack_data_to_mrs(end_mr, &sockaddr, socklen);
    }

    // 发送响应
    info = sel4_msg_info_set_length(info, end_mr);
    sel4_reply(info);
}
```

---

## 5. 共享内存机制

### 5.1 CMA 共享内存

NSv 使用 96MB 的 CMA (Contiguous Memory Area) 与 NIC 驱动共享：

```c
#define CMA_SIZE  0x6000000  // 96MB

init_ds_ring():
    sys_mem_map(..., CMA_SIZE, PAGE_DMA);
```

### 5.2 get_shm_va — 获取共享内存虚拟地址

```c
static inline void *get_shm_va(pid_t pid, unsigned long offset, size_t size)
{
    // 查找进程的 shm 区域
    struct proc_shm_area *shm = find_shm_area(pid);
    if (!shm) return NULL;

    // 验证偏移和大小
    if (offset + size > shm->size) return NULL;

    return (void *)(shm->vaddr + offset);
}
```

### 5.3 数据传输流程

```
应用进程                           NSv 进程
    │                                 │
    │  seL4 IPC (badge + mr0-5)      │
    │───────────────────────────────► │
    │                                 │
    │  [use_shm=1]                   │
    │   - offset 指向共享内存         │
    │   - 直接写入共享内存           │
    │                                 │
    │  [use_shm=0]                   │
    │   - 数据在 message registers    │
    │   - sys_unpack_data_from_mrs()  │
    │                                 │
    ▼                                 ▼
lwip_sendto()                      lwip_sendto()
    │                                 │
    └─────────────────────────────────┘
              DMA → NIC
```

---

## 6. 性能优化

### 6.1 共享内存优势

| 模式 | 最大数据 | 复制次数 | 延迟 |
|------|----------|----------|------|
| **shm** | 受 CMA 大小限制 | 0 (零复制) | 低 |
| **IPC** | 512B | 1 (IPC → cache) | 中 |

### 6.2 何时使用共享内存

- 大块数据 (> 512B)
- 高性能场景
- 音频/视频流

### 6.3 何时使用 IPC

- 小数据 (< 512B)
- 控制消息
- 低频率操作

---

## 7. 错误处理

### 7.1 错误码

| 错误 | 原因 |
|------|------|
| EINVAL | 无效参数、偏移超出范围 |
| ENOMEM | 内存不足 (data cache 分配失败) |
| EFAULT | 共享内存地址无效 |

### 7.2 统计计数

```c
// per-process socket 统计
NSV_PERPID_STATS_INC(pid, type, sendto_fail_cnt);
NSV_PERPID_STATS_INC(pid, type, sendto_success_cnt);

// 全局网络统计
NET_PERF_STATS_INC(sendto_succ_total);
NET_PERF_STATS_INC(sendto_nomems);
```

---

## 8. 与 Linux 对比

| 特性 | SafeOS NSv | Linux |
|------|-------------|-------|
| **sendto** | seL4 IPC + shm | 系统调用 |
| **大块数据** | CMA 共享内存 | 用户态直接 DMA |
| **小数据** | message registers | iovec |
| **零复制** | 支持 (shm 模式) | 支持 (sendfile) |

---

## 9. 总结

### 9.1 数据流

```
TX:
  App → seL4 IPC (mr) → sys_sendto_nb → get_shm_va/alloc_data_cache
        → lwip_sendto() → tcp_output/udp_output() → NIC DMA

RX:
  NIC DMA → lwip_recvfrom() → sys_sendto_nb
        → sys_pack_data_to_mrs/shm_copy → seL4 IPC reply
        → App
```

### 9.2 关键优化

1. **零复制**: 共享内存模式避免数据复制
2. **小数据优化**: 使用 data cache 避免频繁 CMA 分配
3. **异步处理**: 非阻塞模式提高并发性
