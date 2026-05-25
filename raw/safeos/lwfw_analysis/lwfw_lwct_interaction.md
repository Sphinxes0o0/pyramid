# lwfw 与 lwct 交互深度分析

> lwfw 代码: `libs/util_libs/liblwfw/src/lwfw.c`
> lwct 代码: `libs/util_libs/liblwfw/src/lwct/lwct_core.c`

---

## 1. 交互架构

```
数据包进入 lwIP
    │
    ▼
┌─────────────────────────────────────┐
│  lwct_main_hook(pbuf, dir)          │  ← lwct 主 hook
│    └─ lwct_in(pbuf, dir)            │
│          ├─ lwct_hash_tuple()        │  计算五元组哈希
│          ├─ lwct_lookup_conn()       │  查找连接
│          ├─ lwct_new_conn() [未找到] │  创建新连接
│          └─ pbuf->_lwct = conn|state │  绑定到 pbuf
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  ip4_filter(pbuf, inp)              │  ← lwfw 过滤
│    ├─ lwfw_pkt_info_constructor()    │
│    │     └─ pkt_info->ct_state =    │
│    │         p->_lwct & LWCT_STATE_MASK │  ← 提取连接状态
│    ├─ filter_engine->do_filter()     │  使用 ct_state 匹配规则
│    └─ lwfw_handle_filter_result()    │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  lwct_post_hook(pbuf, dir)          │  ← lwct post hook
│    ├─ lwct_confirm()                │  确认连接
│    └─ lwct_pbuf_free_cb()           │  释放连接引用
└─────────────────────────────────────┘
```

---

## 2. 连接信息绑定机制

### 2.1 pbuf 扩展字段

```c
// lwIP pbuf 结构扩展
struct pbuf {
    // ... 标准字段 ...
    uintptr_t _lwct;  // lwct 扩展: conn_ptr | state
};
```

**绑定时机**: `lwct_in()` 末尾

```c
// lwct_core.c:688
static inline int lwct_in(struct pbuf *pbuf, uint32_t dir) {
    // ... 查找/创建连接 ...

    // 绑定到 pbuf
    pbuf->_lwct = (uintptr_t)conn | lwct_state;

    return LWCT_ERR_OK;
}
```

### 2.2 连接状态编码

```c
// lwct_common.h
#define LWCT_STATE_MASK   0x7  // 低 3 位存储状态

typedef enum {
    LWCT_UNKNOWN          = 0,  // 未建立连接
    LWCT_ESTABLISHED      = 1,  // 已知连接
    LWCT_NEW              = 2,  // 新连接
    LWCT_IS_REPLY         = 3,  // 回复方向标志
    LWCT_ESTABLISHED_REPLY = 4, // ESTABLISHED + IS_REPLY
} lwct_state_t;
```

### 2.3 连接指针与状态分离

```c
// 提取连接指针
struct lwct_conn *conn = (struct lwct_conn *)(p->_lwct & LWCT_PTR_MASK);

// 提取连接状态
lwct_state_t state = p->_lwct & LWCT_STATE_MASK;
```

---

## 3. lwfw 如何使用连接状态

### 3.1 包信息构造

```c
// lwfw.c:345
void lwfw_pkt_info_constructor(...)
{
    // ... 解析 L2/L3/L4 字段 ...

#ifdef NIO_LWIP_LWCT
    // 提取连接状态
    pkt_info->ct_state = lwct_convert_reply_state(p->_lwct & LWCT_STATE_MASK);
#endif
}
```

### 3.2 连接状态转换

```c
// lwfw 使用前，lwct 将状态转换为 lwfw 格式
lwct_state_t lwct_convert_reply_state(lwct_state_t state)
{
    // lwct 状态中 IS_REPLY 位表示方向
    // lwfw 规则匹配需要知道是否是回复包
    switch (state) {
        case LWCT_NEW:              return LWFW_CT_NEW;
        case LWCT_ESTABLISHED:      return LWFW_CT_ESTABLISHED;
        case LWCT_IS_REPLY:         return LWFW_CT_RELATED;  // 关联连接
        case LWCT_ESTABLISHED_REPLY: return LWFW_CT_ESTABLISHED;
        default:                     return LWFW_CT_INVALID;
    }
}
```

### 3.3 规则匹配

```c
// lwfw.c:check_rule()
static bool check_rule(lwfw_rule_t *rule, lwfw_pkt_info_t *pkt_info, ...)
{
    // ... 其他匹配 ...

    // 连接状态匹配
    if (rule->flags & LWFW_RULE_FLAGS_CT_STATE) {
        if (rule->ct_state != pkt_info->ct_state) {
            return false;  // 状态不匹配，跳过此规则
        }
    }

    return true;
}
```

---

## 4. 关键交互点

### 4.1 lwct_enable 全局开关

```c
// lwct_core.c:69
int lwct_enable = 0;  // 默认关闭

// lwfw.c:739
if (lwct_enable == 1 && !p->_lwct) {
    // 未启用 lwct 或 pbuf 没有连接信息
    LWFW_STATICS_INC(g_lwfw_stats.ct_notrack);
}
```

### 4.2 未跟踪包的兜底机制

```c
// lwfw.c:739-754
#ifdef NIO_LWIP_LWCT
    if (lwct_enable == 1 && !p->_lwct) {
        // lwct 启用但包未跟踪（可能是连接表满）
        LWFW_STATICS_INC(g_lwfw_stats.ct_notrack);

        if (policy->params.ct_oot_action == LWFW_CT_OOT_ACTION_PASS) {
            return ERR_OK;  // 放行未跟踪的包
        }
    }
#endif
```

### 4.3 连接表满时的行为

| 场景 | lwct 行为 | lwfw 行为 |
|------|-----------|-----------|
| 连接表未满 | 正常创建连接 | 正常过滤 |
| 连接表满 (80% 水位) | 早期 GC，缩短超时 | 正常过滤 |
| 连接表完全满 | 新建连接失败，返回错误 | 使用 `ct_oot_action` 处理 |

---

## 5. 潜在问题分析

### 5.1 pbuf->_lwct 并发访问

```c
// lwct_in() - 在数据包处理上下文调用
pbuf->_lwct = (uintptr_t)conn | lwct_state;

// lwfw_pkt_info_constructor() - 稍后在同一上下文读取
pkt_info->ct_state = p->_lwct & LWCT_STATE_MASK;

// lwct_post_hook() - 在不同上下文（可能是 softirq）写入
pbuf->_lwct = 0;
```

**风险**: 如果同一 pbuf 被多个数据包引用，后面的包可能覆盖前面的连接信息。

### 5.2 位操作非原子

```c
// lwct_common.h:277-342
#define _set_bit(bit, addr)  (*(addr) |= BIT(bit))
#define _clear_bit(bit, addr)  (*(addr) &= ~BIT(bit))
```

这些位操作不是原子的，在多线程环境下可能丢失更新。

**但**: lwct 本身是单线程的（GC 线程），但 lwfw 调用 `lwct_in()` 是在数据包处理上下文中，可能与其他 lwct 操作并发。

### 5.3 连接超时与规则匹配时序

```
时间线:
  T1: 包A到达，lwct_in() 创建连接，设置 timeout=T1+300s
  T2: 包B到达，lwct 认为是同一连接，timeout 已过期但还未 GC
  T3: GC 线程扫描，删除该连接

问题: T2 时连接已过期但尚未删除，lwfw 会认为它是有效连接
```

---

## 6. 优化建议

### 6.1 位操作原子化

```c
// 当前 (非原子)
_set_bit(SEEN_REPLY_BIT, &conn->status);

// 建议 (原子)
__atomic_fetch_or(&conn->status, BIT(SEEN_REPLY_BIT), __ATOMIC_SEQ_CST);
```

### 6.2 添加连接有效性检查

```c
// 在 lwfw 中使用连接状态前，先检查是否已过期
bool lwct_conn_is_valid(struct lwct_conn *conn) {
    if (!conn) return false;
    uint32_t status = __atomic_load_n(&conn->status, __ATOMIC_SEQ_CST);
    if (status & BIT(DYING_BIT)) return false;
    if (conn->timeout < now) return false;
    return true;
}
```

### 6.3 连接表满时的告警

```c
// lwct_core.c:early_gc
if (conn_count >= water_level) {
    LWCT_PRINTF_WARN("Connection table at %d%% water level", water_level/100);
    // 触发告警，通知 lwfw_agent
}
```

---

## 7. 关键代码位置

| 文件 | 行号 | 描述 |
|------|------|------|
| `lwct_core.c` | 688 | `lwct_in()` 主 hook |
| `lwct_core.c` | 747 | `lwct_confirm()` post hook |
| `lwct_core.c` | 775 | `lwct_post_hook()` |
| `lwfw.c` | 345 | 提取连接状态 |
| `lwfw.c` | 739 | lwct 未启用/未跟踪时兜底 |
| `lwfw.c` | `check_rule()` | 使用 ct_state 匹配 |
