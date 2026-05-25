---
type: entity
tags: [linux, lwip, network, memory]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP Memory Management

## 定义

lwIP 提供两种内存管理机制：**memp** (固定大小内存池) 和 **mem** (动态堆内存)。

## memp — 内存池架构

### 内存池列表

| 池名称 | 说明 | 结构大小 |
|--------|------|----------|
| `PBUF` | pbuf 引用 | sizeof(struct pbuf) |
| `PBUF_POOL` | pbuf 数据池 | sizeof(pbuf) + PBUF_POOL_BUFSIZE |
| `TCP_PCB` | TCP PCB | sizeof(struct tcp_pcb) |
| `TCP_PCB_LISTEN` | TCP Listen PCB | sizeof(struct tcp_pcb_listen) |
| `TCP_SEG` | TCP Segment | sizeof(struct tcp_seg) |
| `UDP_PCB` | UDP PCB | sizeof(struct udp_pcb) |
| `RAW_PCB` | RAW PCB | sizeof(struct raw_pcb) |
| `NETCONN` | netconn | sizeof(struct netconn) |
| `LWFW_RULE` | LWFW 规则 | sizeof(struct lwfw_rule) |
| `LWCT_CONN` | LWCT 连接追踪 | sizeof(struct lwct_conn) |

### 分配特性
- **O(1)** 分配时间
- 无碎片化
- 固定大小，可能浪费

## mem — 堆内存架构

### 分配算法
`mem_malloc` 使用**首次适应 (First Fit)** 算法：

```c
void *mem_malloc(mem_size_t size_in) {
    // 扫描空闲链表查找足够大的块
    for (ptr = mem_to_ptr(lfree); ptr < MEM_SIZE_ALIGNED - size; ptr = ptr_to_mem(ptr)->next) {
        mem = ptr_to_mem(ptr);
        if ((!mem->used) && (mem->next - (ptr + SIZEOF_STRUCT_MEM)) >= size) {
            // 找到足够的空间，可能分割
            if (mem->next - (ptr + SIZEOF_STRUCT_MEM) >= size + SIZEOF_STRUCT_MEM + MIN_SIZE_ALIGNED) {
                // 分割大块
                ptr2 = ptr + SIZEOF_STRUCT_MEM + size;
                mem2 = ptr_to_mem(ptr2);
                mem2->used = 0;
                mem2->next = mem->next;
                mem->next = ptr2;
            }
            mem->used = 1;
            return (u8_t *)mem + SIZEOF_STRUCT_MEM;
        }
    }
    return NULL;  // 分配失败
}
```

### 分配特性
- **O(n)** 分配时间
- 动态大小，灵活但可能碎片
- 按需分配，无浪费

## 分配策略对比

| 特性 | memp (内存池) | mem (堆) |
|------|---------------|----------|
| **分配时间** | O(1) | O(n) 首次适应 |
| **碎片化** | 无 | 可能 |
| **适用场景** | 协议栈内部结构 | 应用数据缓冲 |

## 线程安全

- **内存池**: `SYS_ARCH_PROTECT(old_level)` — 禁用中断
- **堆**: `sys_mutex_lock(&mem_mutex)` — 互斥锁

## 相关概念

- [[entities/linux/lwip/lwip-pbuf]] — pbuf 使用 memp 和 mem 分配
- [[entities/linux/lwip/lwip-tcp-pcb]] — TCP PCB 使用 memp
- [[entities/linux/lwip/lwip-udp-socket]] — UDP PCB 使用 memp
- [[entities/linux/lwip/lwip-network-init]] — mem_init 和 memp_init 在 lwip_init 中调用
