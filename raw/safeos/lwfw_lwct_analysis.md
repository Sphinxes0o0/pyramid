# LWFW 连接追踪分析 — T-071

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: Connection tracking 表、状态机 (NEW/ESTABLISHED)、lwct 模块

---

## 1. 概述

LWCT (Light Weight Connection Tracking) 是 LWFW 的**状态追踪模块**，为防火墙提供**状态ful 过滤**能力：

1. 追踪 TCP/UDP/ICMP 连接状态
2. 在 pbuf 中绑定连接信息 (`pbuf->_lwct`)
3. 支持规则基于连接状态匹配 (NEW/ESTABLISHED/RELATED)

---

## 2. 数据结构

### 2.1 连接元组 (lwct_tuple)

**文件**: `lwct/lwct_tuple.h`

```c
struct lwct_tuple {
    struct lwct_info src;  // 源 IP + Port
    struct lwct_info dst;  // 目的 IP + Port
    uint8_t protonum;      // L4 协议 (TCP/UDP/ICMP)
    lwct_dir_t dir;        // 方向 (ORIGINAL/REPLY)
};

struct lwct_info {
    uint32_t ip;           // IPv4 地址
    union {
        struct { uint16_t port; } tcp;
        struct { uint16_t port; } udp;
        struct { uint8_t type, code; } icmp;
        struct { uint16_t id; } icmp_o;
    } u;
};
```

### 2.2 连接结构 (lwct_conn)

```c
struct lwct_conn {
    struct lwct_tuple_hash tuple_hash;  // 哈希链表
    uint32_t status;                    // 状态标志
    uint64_t timeout;                   // 超时时间
    struct lwct_ext *ext;              // 扩展数据
};
```

### 2.3 状态标志

**文件**: `lwct/lwct_common.h:69-86`

```c
typedef enum {
    LWCT_SEEN_REPLY_BIT = 0,   // 收到回复
    LWCT_SEEN_REPLY = (1 << 0),

    LWCT_ASSURED_BIT = 1,      // 连接已确认，不能删除
    LWCT_ASSURED = (1 << 1),

    LWCT_CONFIRMED_BIT = 2,     // 发起包已离开
    LWCT_CONFIRMED = (1 << 2),

    LWCT_DYING_BIT = 3,         // 连接正在删除
    LWCT_DYING = (1 << 3),
} lwct_status_t;
```

### 2.4 连接状态 (per-packet)

```c
typedef enum {
    LWCT_UNKNOWN = 0,           // 未知
    LWCT_ESTABLISHED,           // 已建立 (任一方向)
    LWCT_NEW,                   // 新连接 (仅 ORIGINAL)
    LWCT_IS_REPLY,              // 回复方向标记
    LWCT_ESTABLISHED_REPLY = LWCT_ESTABLISHED + LWCT_IS_REPLY,
} lwct_state_t;

// 状态存储在 pbuf->_lwct 的低 3 位
#define LWCT_STATE_MASK 7UL
#define LWCT_PTR_MASK ~LWCT_STATE_MASK
```

---

## 3. 连接追踪流程

### 3.1 lwct_in — 主 Hook

**文件**: `lwct/lwct_core.h:80`

```c
// ip4_input/ip4_output 中调用
static inline int lwct_in(struct pbuf *pbuf, uint32_t dir)
{
    // 1. 查找连接
    conn = lwct_find_conn(pbuf, dir);
    if (conn == NULL) {
        // 2. 无连接，创建新连接
        conn = lwct_create_conn(pbuf, dir);
        state = LWCT_NEW;
    } else {
        // 3. 有连接，更新状态
        state = lwct_update_state(conn, dir);
    }

    // 4. 绑定到 pbuf
    pbuf->_lwct = (uint64_t)conn | state;

    return ERR_OK;
}
```

### 3.2 lwct_confirm — 确认连接

```c
// 当发起包离开时调用
int lwct_confirm(struct pbuf *pbuf, uint32_t dir)
{
    // 将连接从未确认列表移到确认列表
    lwct_move_to_confirmed_list(conn);
    set_bit(LWCT_CONFIRMED, conn->status);
}
```

---

## 4. 哈希表结构

### 4.1 连接表

```c
struct lwct_conn_table {
    struct cdlist *conn_lists;  // 哈希桶数组
    sys_mutex_t *bkt_locks;    // 桶锁数组
};

// 桶和锁配置
#define LWCT_LOCK_COUNT           256
#define LWCT_BUCKETS_PER_LOCK     32
#define LWCT_BUCKET_COUNT         (LWCT_LOCK_COUNT * LWWCT_BUCKETS_PER_LOCK)  // 8192 桶
#define LWCT_MAX_CONN_COUNT       8192
```

### 4.2 哈希计算

```c
// djb2 哈希算法
static uint32_t hash_lwct_tuple(const struct lwct_tuple *tuple) {
    return djb2((const char*)tuple,
                OFFSETOF(struct lwct_tuple, __lwct_hash_offsetend));
}

// 映射到桶
#define LWCT_HASH_TO_BKT_INDEX(hash) reciprocal_scale(hash, lwct_get_bucket_size())
```

---

## 5. 状态机

### 5.1 TCP 状态转换

```
                   SYN (NEW)
                       │
                       ▼
                   TCP_SYN_SENT
                       │
           SYN+ACK ◄───┘
               │
               ▼
        ESTABLISHED
               │
   ┌───────────┴───────────┐
   │                       │
FIN (NEW)               FIN (ESTABLISHED_REPLY)
   │                       │
   ▼                       ▼
  FIN_WAIT_1           CLOSE_WAIT
   │                       │
   │   ┌───────────────────┘
   │   │
   │   ▼
   │  LAST_ACK
   │   │
   │   ▼
   │  CLOSED
   │
   ▼
TIME_WAIT
   │
   ▼
CLOSED
```

### 5.2 UDP 状态转换

```
                First Packet (NEW)
                       │
                       ▼
                   UDP_UNREPLIED
                       │
           Reply ◄─────┘
               │
               ▼
          UDP_REPLIED
               │
    Timeout ◄──┴──► Activity
               │
               ▼
           UDP_EXPIRED
```

### 5.3 超时配置

**文件**: `lwct/lwct_common.h:138-145`

```c
#define LWCT_TCP_UNREPLIED_TMO    (3 * 60)   // 3 分钟
#define LWCT_TCP_REPLIED_TMO      (3 * 60)   // 3 分钟
#define LWCT_TCP_ESTABLISHED_TMO (3 * 60 * 60)  // 3 小时
#define LWCT_UDP_UNREPLIED_TMO    (1 * 60)   // 1 分钟
#define LWCT_UDP_REPLIED_TMO      (1 * 60)   // 1 分钟
#define LWCT_UDP_ESTABLISHED_TMO  (3 * 60 * 60)  // 3 小时
#define LWCT_ICMP_UNREPLIED_TMO   10          // 10 秒
#define LWCT_ICMP_REPLIED_TMO     10          // 10 秒
```

---

## 6. GC (垃圾回收) 线程

### 6.1 GC 参数

```c
#define LWCT_GC_THREAD_NAME     "lwct-gc"
#define LWCT_GC_THREAD_PRIO     200
#define LWCT_GC_SCAN_MAX        1024        // 每次扫描最大连接数
#define LWCT_GC_INTERVAL        3000       // 扫描间隔 (ms)
```

### 6.2 GC 流程

```c
void gc_thread_fn(void *__tbd)
{
    while (1) {
        sleep(LWCT_GC_INTERVAL);

        // 扫描连接表
        for (i = 0; i < LWCT_GC_SCAN_MAX; i++) {
            // 检查连接是否超时
            if (conn->timeout < now) {
                // 删除过期连接
                lwct_delete_conn(conn);
            }
        }
    }
}
```

---

## 7. 与 LWFW 规则集成

### 7.1 规则中的 CT_STATE 匹配

**文件**: `lwfw/lwfw.c`

```c
// 规则匹配时检查连接状态
if (rule->flags & LWFW_RULE_FLAG_CT_STATE) {
    lwct_state_t ct_state = pbuf->_lwct & LWCT_STATE_MASK;

    switch (rule->ct_state) {
        case LWFW_CT_NEW:
            if (ct_state != LWCT_NEW) match = false;
            break;
        case LWFW_CT_ESTABLISHED:
            if (ct_state != LWCT_ESTABLISHED &&
                ct_state != LWCT_ESTABLISHED_REPLY) match = false;
            break;
        case LWFW_CT_RELATED:
            // ICMP error 等关联连接
            break;
    }
}
```

### 7.2 状态映射

| lwct 状态 | lwfw 状态 | 说明 |
|-----------|---------|------|
| `LWCT_NEW` | `LWFW_CT_NEW` | 新连接 |
| `LWCT_ESTABLISHED` | `LWFW_CT_ESTABLISHED` | 已建立 (原始方向) |
| `LWCT_ESTABLISHED_REPLY` | `LWFW_CT_ESTABLISHED` | 已建立 (回复方向) |

---

## 8. pbuf->_lwct 字段

### 8.1 字段结构

```c
struct pbuf {
    // ... 其他字段 ...

    #ifdef NIO_LWIP_LWCT
    u64_t _lwct;  // 连接追踪状态
    #endif
};

// _lwct 结构:
// [高位: 连接指针] [低3位: 状态]
// ptr = _lwct & LWCT_PTR_MASK
// state = _lwct & LWCT_STATE_MASK
```

### 8.2 访问宏

```c
#define LWCT_STATE2DIR(lwct_state) ((lwct_state) >= LWCT_IS_REPLY ? \
                                     LWCT_DIR_REPLY : LWCT_DIR_ORIGINAL)
```

---

## 9. 性能特征

### 9.1 哈希表规模

| 参数 | 值 |
|------|-----|
| **桶数** | 8192 |
| **锁数** | 256 |
| **每锁桶数** | 32 |
| **最大连接数** | 8192 |

### 9.2 时间复杂度

| 操作 | 复杂度 |
|------|--------|
| **查找连接** | O(1) 平均 |
| **创建连接** | O(1) |
| **删除连接** | O(1) |
| **GC 扫描** | O(n) |

---

## 10. 总结

### 10.1 LWCT 核心功能

```
lwct
    │
    ├─► 连接追踪
    │     ├─► 创建/查找/删除连接
    │     ├─► 状态管理 (NEW/ESTABLISHED/RELATED)
    │     └─► 超时管理
    │
    ├─► 哈希表
    │     ├─► 8192 桶
    │     ├─► 256 锁 (减少竞争)
    │     └─► O(1) 查找
    │
    └─► GC 线程
          ├─► 定期扫描过期连接
          └─► 释放资源
```

### 10.2 与 LWFW 集成

```
Packet 到达
    │
    ├─► lwct_in() → 创建/查找连接
    │     └─► pbuf->_lwct = conn | state
    │
    └─► lwfw_rule_match()
          └─► 检查 ct_state 匹配
```

### 10.3 关键设计

1. **无锁哈希**: 每个桶有独立锁，减少竞争
2. **状态存储在 pbuf**: 无需额外查找，直接获取连接状态
3. **GC 线程**: 定期清理过期连接
4. **双缓冲**: 确认/未确认连接分离管理
