# lwIP 内存管理分析 — T-113

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: lwip_malloc 内存池初始化、分配策略

---

## 1. 概述

lwIP 提供两种内存管理机制：
1. **memp** — 固定大小内存池，用于协议栈内部结构 (PCB、pbuf、segment 等)
2. **mem** — 动态堆内存，用于应用程序数据缓冲

**主要特性**：
- 零碎片化设计
- 确定性分配时间 (O(1) for pools)
- 线程安全 (通过 mutex 或 sys_arch_protect)
- 可选的 overflow 检测

---

## 2. 内存池 (memp) 架构

### 2.1 内存池列表

**文件**: `src/include/lwip/priv/memp_std.h`

lwIP 使用预定义的固定大小内存池：

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
| `NETBUF` | netbuf | sizeof(struct netbuf) |
| `ARP_QUEUE` | ARP 队列 | sizeof(struct etharp_q_entry) |
| `IGMP_GROUP` | IGMP 组 | sizeof(struct igmp_group) |
| `REASSDATA` | IP 分片重组数据 | sizeof(struct ip_reassdata) |
| `SYS_TIMEOUT` | 超时结构 | sizeof(struct sys_timeo) |
| `LWFW_RULE` | LWFW 规则 | sizeof(struct lwfw_rule) |
| `LWCT_CONN` | LWCT 连接追踪 | sizeof(struct lwct_conn) |

### 2.2 数据结构

**文件**: `src/core/memp.c`

```c
struct memp {
    struct memp *next;  // 链表指针
#if MEMP_OVERFLOW_CHECK
    const char *file;   // 分配文件名
    int line;           // 分配行号
#endif
};

struct memp_desc {
    struct memp **tab;   // 指向池链表头的指针
    const char *desc;    // 池描述
    mem_size_t size;     // 每个元素大小
    u16_t num;           // 元素数量
    union {
        struct {
            const void *base;
        } heap;
        struct {
            int dummy;
        } pool;
    } memory;
#if MEMP_STATS
    struct stats_mem *stats;
#endif
};
```

### 2.3 内存布局

```
memp_memory 区域:
┌─────────────────────────────────────────────────────────────┐
│  Pool 0          │  Pool 1          │  Pool 2          │...
│  ┌────────────┐  │  ┌────────────┐  │  ┌────────────┐  │
│  │ elem[0]    │  │  │ elem[0]    │  │  │ elem[0]    │  │
│  │ elem[1]    │  │  │ elem[1]    │  │  │ elem[1]    │  │
│  │ ...        │  │  │ ...        │  │  │ ...        │  │
│  │ elem[n-1]  │  │  │ elem[n-1]  │  │  │ elem[n-1]  │  │
│  └────────────┘  │  └────────────┘  │  └────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

每个池维护一个空闲链表，分配时从链表头部取元素。

---

## 3. 内存堆 (mem) 架构

### 3.1 堆结构

**文件**: `src/core/mem.c`

```c
struct mem {
    mem_size_t next;    // 下一个块的索引
    mem_size_t prev;    // 前一个块的索引
    u8_t used;          // 0=空闲, 1=已用
#if MEM_OVERFLOW_CHECK
    mem_size_t user_size;  // 用户分配大小 (用于 guard 检测)
#endif
};
```

### 3.2 堆布局

```
┌─────┬───────────────┬───────────────┬─────────────┬─────┐
│start│   used mem    │    free mem   │  used mem   │ end │
│     │  ┌─────────┐  │  ┌─────────┐  │             │     │
│     │  │ struct  │  │  │ struct  │  │             │     │
│     │  │   mem   │  │  │   mem   │  │             │     │
│     │  └─────────┘  │  └─────────┘  │             │     │
└─────┴───────────────┴───────────────┴─────────────┴─────┘
       ▲               ▲               ▲
       │               │               │
     used=1          used=0         used=1
```

### 3.3 分配算法

mem_malloc 使用**首次适应 (First Fit)** 算法：

```c
void *mem_malloc(mem_size_t size_in)
{
    // 1. 对齐和最小大小检查
    size = LWIP_MEM_ALIGN_SIZE(size_in);
    if (size < MIN_SIZE_ALIGNED) {
        size = MIN_SIZE_ALIGNED;
    }

    // 2. 扫描空闲链表查找足够大的块
    for (ptr = mem_to_ptr(lfree); ptr < MEM_SIZE_ALIGNED - size;
         ptr = ptr_to_mem(ptr)->next) {
        mem = ptr_to_mem(ptr);

        if ((!mem->used) &&
            (mem->next - (ptr + SIZEOF_STRUCT_MEM)) >= size) {

            // 找到足够的空间
            if (mem->next - (ptr + SIZEOF_STRUCT_MEM) >=
                (size + SIZEOF_STRUCT_MEM + MIN_SIZE_ALIGNED)) {
                // 分割大块，创建剩余空闲块
                ptr2 = ptr + SIZEOF_STRUCT_MEM + size;
                mem2 = ptr_to_mem(ptr2);
                mem2->used = 0;
                mem2->next = mem->next;
                mem2->prev = ptr;
                mem->next = ptr2;
                mem->used = 1;
            } else {
                // 恰好适合，不分割
                mem->used = 1;
            }
            return (u8_t *)mem + SIZEOF_STRUCT_MEM;
        }
    }
    return NULL;  // 分配失败
}
```

### 3.4 释放与碎片合并

```c
void mem_free(void *rmem)
{
    // 1. 获取 struct mem 指针
    mem = (struct mem *)((u8_t *)rmem - SIZEOF_STRUCT_MEM);

    // 2. 标记为未使用
    mem->used = 0;

    // 3. 更新 lfree 指针
    if (mem < lfree) {
        lfree = mem;
    }

    // 4. 合并相邻的空闲块 (plug_holes)
    plug_holes(mem);
}
```

plug_holes 前后合并相邻的空闲块，减少碎片。

---

## 4. 初始化流程

### 4.1 lwip_init — 主初始化函数

**文件**: `src/core/init.c:337-397`

```c
void lwip_init(void)
{
    stats_init();        // 统计初始化
    sys_init();          // 系统初始化
    mem_init();          // 堆内存初始化
    memp_init();         // 内存池初始化
    pbuf_init();         // pbuf 初始化
    netif_init();        // netif 初始化

#ifdef NIO_LWIP_LWFW
    lwfw_init();         // LWFW 初始化
#endif

    ip_init();           // IP 层初始化
    etharp_init();       // ARP 初始化
    raw_init();          // RAW socket 初始化
    udp_init();          // UDP 初始化
    tcp_init();          // TCP 初始化
    igmp_init();         // IGMP 初始化
    dns_init();          // DNS 初始化

    sys_timeouts_init(); // 超时处理初始化
}
```

### 4.2 mem_init — 堆初始化

**文件**: `src/core/mem.c:512-542`

```c
void mem_init(void)
{
    struct mem *mem;

    /* 对齐堆起始地址 */
    ram = (u8_t *)LWIP_MEM_ALIGN(LWIP_RAM_HEAP_POINTER);

    /* 初始化堆起始块 */
    mem = (struct mem *)ram;
    mem->next = MEM_SIZE_ALIGNED;
    mem->prev = 0;
    mem->used = 0;

    /* 初始化堆结束块 */
    ram_end = ptr_to_mem(MEM_SIZE_ALIGNED);
    ram_end->used = 1;
    ram_end->next = MEM_SIZE_ALIGNED;
    ram_end->prev = MEM_SIZE_ALIGNED;

    /* 初始化最低空闲指针 */
    lfree = (struct mem *)ram;

    /* 创建互斥锁 */
    sys_mutex_new(&mem_mutex);
}
```

### 4.3 memp_init — 内存池初始化

**文件**: `src/core/memp.c:231-249`

```c
void memp_init(void)
{
    u16_t i;

    /* 遍历所有内存池 */
    for (i = 0; i < LWIP_ARRAYSIZE(memp_pools); i++) {
        memp_init_pool(memp_pools[i]);
    }
}

void memp_init_pool(const struct memp_desc *desc)
{
    int i;
    struct memp *memp;

    *desc->tab = NULL;
    memp = (struct memp *)LWIP_MEM_ALIGN(desc->base);

    /* 创建空闲链表 */
    for (i = 0; i < desc->num; ++i) {
        memp->next = *desc->tab;  // 头插法
        *desc->tab = memp;
        memp = (struct memp *)((u8_t *)memp + MEMP_SIZE + desc->size);
    }
}
```

---

## 5. 分配策略对比

### 5.1 memp vs mem

| 特性 | memp (内存池) | mem (堆) |
|------|---------------|----------|
| **分配时间** | O(1) | O(n) 首次适应 |
| **碎片化** | 无 | 可能 (长期运行后) |
| **内存浪费** | 固定大小，可能浪费 | 按需分配，无浪费 |
| **适用场景** | 协议栈内部结构 | 应用数据缓冲 |
| **线程安全** | 是 (sys_arch_protect) | 是 (mutex) |

### 5.2 SafeOS 中的选择

SafeOS 中 lwIP 主要使用：

1. **memp** — 用于所有协议栈内部结构
   - TCP/UDP/RAW PCB
   - pbuf
   - LWFW 规则和连接追踪

2. **mem** — 用于应用程序数据
   - Socket 发送/接收缓冲
   - DNS 查询缓冲

---

## 6. 线程安全

### 6.1 内存池并发控制

```c
static void *do_memp_malloc_pool(const struct memp_desc *desc)
{
    struct memp *memp;
    SYS_ARCH_DECL_PROTECT(old_level);

    SYS_ARCH_PROTECT(old_level);    // 进入临界区
    memp = *desc->tab;              // 从链表头部取

    if (memp != NULL) {
        *desc->tab = memp->next;    // 更新链表头
    }
    SYS_ARCH_UNPROTECT(old_level);  // 离开临界区

    return (u8_t *)memp + MEMP_SIZE;
}
```

### 6.2 堆并发控制

```c
void *mem_malloc(mem_size_t size_in)
{
    sys_mutex_lock(&mem_mutex);     // 获取互斥锁
    LWIP_MEM_ALLOC_PROTECT();

    // ... 分配逻辑 ...

    sys_mutex_unlock(&mem_mutex);
}
```

---

## 7. 调试与检测

### 7.1 MEMP_OVERFLOW_CHECK

```c
// 分配时在元素后设置 guard pattern
memp_overflow_init_element(memp, desc);

// 释放时检测 guard 是否被破坏
memp_overflow_check_element(memp, desc);
```

### 7.2 MEM_OVERFLOW_CHECK

```c
// 堆块结构中的 user_size 记录
struct mem {
    // ...
#if MEM_OVERFLOW_CHECK
    mem_size_t user_size;
#endif
};

// mem_trim 重新设置大小
mem_overflow_init_element(mem, new_size);
```

---

## 8. 关键配置选项

**文件**: `lwipopts.h`

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `MEM_SIZE` | 堆大小 (bytes) | 1600 |
| `MEMP_NUM_*` | 各内存池元素数量 | 视配置而定 |
| `MEMP_MEM_MALLOC` | 使用 mem_malloc 分配池 | 0 |
| `MEM_LIBC_MALLOC` | 使用标准 C 库 malloc | 0 |
| `MEM_USE_POOLS` | 使用池而非堆 | 0 |
| `MEM_OVERFLOW_CHECK` | 堆 overflow 检测 | 0 |
| `MEMP_OVERFLOW_CHECK` | 池 overflow 检测 | 0 |
| `MEMP_SANITY_CHECK` | 池链表环检测 | 0 |
| `MEM_SANITY_CHECK` | 堆完整性检测 | 0 |

---

## 9. 总结

### 9.1 架构

```
lwIP 内存管理
    │
    ├── memp (内存池)
    │     ├── 固定大小，消除碎片
    │     ├── O(1) 分配时间
    │     └── 用于协议栈内部结构
    │
    └── mem (堆)
          ├── 动态大小，灵活但可能碎片
          ├── O(n) 分配时间
          └── 用于应用数据
```

### 9.2 初始化顺序

```
lwip_init()
    │
    ├─ mem_init()      → 初始化堆
    ├─ memp_init()     → 初始化所有内存池
    ├─ pbuf_init()     → 初始化 pbuf 池
    └─ 协议层 init()   → tcp_init(), udp_init() 等
```

### 9.3 分配策略

| 场景 | 分配方式 | 原因 |
|------|----------|------|
| TCP/UDP PCB | memp | 固定大小，需要 O(1) |
| pbuf | memp (PBUF_POOL) | 固定大小，高频使用 |
| 应用数据 | mem | 大小不确定 |
| LWFW 规则 | memp | 固定结构大小 |

### 9.4 SafeOS 特定配置

SafeOS 在 lwipopts.h 中配置了：
- `NIO_LWIP_LWFW` — 启用 LWFW 内存池
- `NIO_LWIP_LWCT` — 启用 LWCT 连接追踪内存池
- 适当的 `MEMP_NUM_*` 值以支持多连接场景
