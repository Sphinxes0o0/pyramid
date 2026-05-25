# 防火墙策略数据结构设计分析

> 代码路径: `libs/util_libs/liblwfw/include/lwfw.h`, `lwfw_common.h`

---

## 1. 核心数据结构总览

```
lwfw_policy_t (策略)
  ├─ rule_tables[2]          ← IN_TABLE + OUT_TABLE
  │     ├─ rule_cnt          ← 规则数量
  │     ├─ state             ← ENABLE / DISABLE
  │     ├─ def_action        ← 默认动作
  │     ├─ header            ← cdlist 链表头
  │     ├─ _ruleset          ← 树搜索规则集
  │     └─ _hs_tree          ← 树索引
  ├─ filter_engine          ← 函数指针 (list / tree)
  ├─ params                 ← 策略参数
  └─ policy_name            ← 策略名称

lwfw_rule_t (规则)
  ├─ next                   ← cdlist 链表指针
  ├─ index                  ← 规则索引
  ├─ priority               ← 优先级
  ├─ state                  ← ENABLE / DISABLE
  ├─ flags                  ← 匹配标志位图
  ├─ interface              ← 网卡接口名
  ├─ l2 (MAC/VLAN/EtherType)
  ├─ l3 (IP/Protocol)
  ├─ l4 (Port)
  ├─ action                 ← DENY / EVENT / LOGGING
  ├─ rlimit                 ← 速率限制
  └─ hit_cnt                 ← 命中计数

lwfw_pkt_info_t (数据包)
  ├─ interface              ← 接口名
  ├─ l2 / l3 / l4          ← 包解析的字段
  ├─ dir                    ← RX / TX
  └─ ct_state               ← 连接跟踪状态
```

---

## 2. 内存布局与缓存对齐

### 2.1 规则结构

```c
struct __attribute__((aligned(CACHE_ALIGNMENT))) lwfw_rule {
  struct cdlist next;         // 链表节点 (通常 8-16 字节)
  uint16_t index;            // 规则索引
  uint16_t priority;         // 优先级 (当前未使用)
  uint16_t state;            // 启用/禁用
  uint16_t ct_state;         // 连接跟踪状态
  uint32_t flags;            // 匹配标志位图
  char rule_name[32];       // 规则名称
  lwfw_netif_t interface;   // 接口名 (32 字节)
  lwfw_rule_l2_info_t l2;   // L2 匹配字段
  lwfw_rule_l3_info_t l3;   // L3 匹配字段
  lwfw_rule_l4_info_t l4;   // L4 匹配字段
  lwfw_action_t action;     // 动作
  rate_limit_t rlimit;      // 速率限制
  uint32_t hit_cnt;         // 命中计数
};
```

### 2.2 CACHE_ALIGNMENT

```c
#define CACHE_ALIGNMENT (1U << CONFIG_L1_CACHE_LINE_SIZE_BITS)
// ARM Cortex-A 系列通常 L1_CACHE_LINE_SIZE_BITS = 6 (64 字节)
```

**目的**: 确保每条规则起始地址在缓存行边界，减少跨缓存行访问，提升遍历性能。

### 2.3 问题

`lwfw_rule_t` 包含 `char rule_name[32]` 和 `lwfw_netif_t interface` (也是 `char[32]`)，共 64 字节的字符数组。如果后续字段没有 8 字节对齐，可能导致 `l2` 起始地址不在缓存行边界。

---

## 3. 热切换机制

### 3.1 双缓冲策略

```c
lwfw_policy_t lwfw_policy = { .memp_type = MEMP_LWFW_RULE };
lwfw_policy_t lwfw_policy_swap = { .memp_type = MEMP_LWFW_RULE_SWAP };

// 运行时
lwfw_policy_t *policy;           // 当前活跃策略
lwfw_policy_t *inactive_policy;   // 备份策略
```

### 3.2 切换流程

```c
lwfw_config_reset_state()
  ├─ sync_mutex_lock(&policy_lock)
  ├─ lwfw_policy_clean(inactive_policy);  // 清理旧备份
  ├─ lwfw_copy_policy(inactive_policy, policy);  // 深拷贝规则
  ├─ inactive_policy->filter_engine->init();  // 重建索引
  ├─ tmp = policy
  ├─ policy = inactive_policy           // 原子交换
  ├─ inactive_policy = tmp
  └─ sync_mutex_unlock(&policy_lock)
```

### 3.3 深拷贝实现

```c
static int lwfw_copy_policy(dst, src)
{
  // 1. 拷贝元数据
  memcpy(dst, src, __builtin_offsetof(lwfw_policy_t, __lwfw_policy_copy_offset));

  // 2. 逐规则表拷贝
  for (i = 0; i < MAX_COUNT_TABLE; i++) {
    lwfw_copy_rule_table(&dst->rule_tables[i], &src->rule_tables[i]);
  }
}

static int lwfw_copy_rule_table(dst, src)
{
  cdlist_iter_entry(curr_rule, &src->header, next) {
    new_rule = memp_malloc(dst->memp_type);
    memcpy(new_rule, curr_rule, sizeof(lwfw_rule_t));
    cdlist_add_tail(&dst->header, &new_rule->next);
  }
}
```

---

## 4. 匹配标志位图

### 4.1 位定义

```c
// L2
BIT(0)  LWFW_RULE_FLAGS_NETIF
BIT(1)  LWFW_RULE_FLAGS_SRC_MAC
BIT(2)  LWFW_RULE_FLAGS_DST_MAC
BIT(3)  LWFW_RULE_FLAGS_VLAN
BIT(4)  LWFW_RULE_FLAGS_ETHER_TYPE

// L3
BIT(5)  LWFW_RULE_FLAGS_PROTOCOL
BIT(6)  LWFW_RULE_FLAGS_SRC_IP_MASK / SRC_IP_MASK_LEN
BIT(8)  LWFW_RULE_FLAGS_DST_IP_MASK / DST_IP_MASK_LEN

// L4
BIT(10) LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE
BIT(11) LWFW_RULE_FLAGS_SRC_L4_PORT_LIST
BIT(12) LWFW_RULE_FLAGS_DST_L4_PORT_RANGE
BIT(13) LWFW_RULE_FLAGS_DST_L4_PORT_LIST

// 其他
BIT(14) LWFW_RULE_FLAGS_CT_STATE
BIT(15) LWFW_RULE_FLAGS_RATE_LIMIT
```

### 4.2 IP 掩码标志冲突

```c
// lwfw_common.h:263-264
#define LWFW_RULE_FLAGS_SRC_IP_MASK_LEN        BIT(7)
#define LWFW_RULE_FLAGS_SRC_IP_MASK             BIT(6)
```

`BIT(6)` 和 `BIT(7)` 分别用于两种不同的掩码格式（直接掩码 vs CIDR 前缀），但 `BIT(7)` 已被 `SRC_L4_PORT_RANGE` 使用！实际代码中：

```c
// lwfw_parser.c
if (strcmp(data, "mask") == 0)    → 设置 SRC_IP_MASK (BIT 6)
if (strcmp(data, "prefix") == 0)   → 设置 SRC_IP_MASK_LEN (BIT 7)  ← 冲突！
```

### 4.3 位标志与字段不匹配

lwfw_common.h 定义了 `LWFW_RULE_FLAGS_*` 枚举，但 lwfw.h 又定义了同名宏（部分冲突）。建议统一使用一套定义。

---

## 5. 事件 FIFO

```c
struct lwfw_event_fifo {
  sync_mutex_t prod_lock;         // 多生产者锁
  lwfw_agent_parameters_t params;  // 配置参数
  lwfw_agent_data_t data;        // 统计信息
  volatile uint32_t get_idx;      // 消费者索引
  volatile uint32_t put_idx;    // 生产者索引
  uint32_t queue_size;           // 事件槽数量
  lwfw_event events[1];          // 变长数组
};
```

**变长数组**: `events[1]` 允许运行时根据 `queue_size` 分配实际大小。计算公式:

```c
fifo_size = sizeof(lwfw_event_fifo) + (queue_size - 1) * sizeof(lwfw_event);
fifo_size = ALIGN_UP(fifo_size, 4096);  // 4KB 对齐
```

---

## 6. 速率限制结构

```c
typedef struct rate_limit {
  char name[32];
  uint32_t burst;       // 桶容量
  uint32_t rate;        // 速率 (pps)
  uint32_t expire;      // 限速持续时间 (秒)
  uint32_t event_mode;  // EDGE / LEVEL
  uint32_t rx_pps;      // 当前速率 (原子更新)
  uint32_t time;        // 已限速时长
  uint32_t drops;       // 累计丢包
  uint16_t occurs;      // 进入限速次数
  uint16_t interval;
  lwfw_rlimit_state_t state;  // NORMAL / LIMIT
} rate_limit_t;
```

---

## 7. 数据结构问题汇总

### 7.1 位标志冲突

| 宏 | 位 | 冲突 |
|----|----|------|
| `SRC_IP_MASK` | BIT(6) |  |
| `SRC_IP_MASK_LEN` | BIT(7) | ❌ 与 `SRC_L4_PORT_RANGE` 冲突 |
| `SRC_L4_PORT_RANGE` | BIT(10) |  |
| `SRC_L4_PORT_LIST` | BIT(11) |  |

### 7.2 copy_offset 技巧

```c
struct {
  // ... many fields ...
  struct { } __lwfw_policy_copy_offset;  // 空结构体，仅用于计算偏移量
} lwfw_policy_t;
```

使用 `__builtin_offsetof(lwfw_policy_t, __lwfw_policy_copy_offset)` 获取需要拷贝的元数据长度。这是合法的 GCC 扩展，但缺乏文档说明。

### 7.3 规则名称长度

```c
#define MAX_RULE_NAME_LEN  32
#define MAX_IF_FULLNAME_LEN 32
```

两者相同，但 `lwfw_netif_t` 用的是 `MAX_IF_FULLNAME_LEN`，而 `lwfw_rule_config_t` 用的是 `MAX_RULE_NAME_LEN`。建议统一。

### 7.4 Tree 模式重复分配

```c
lwfw_policy_t lwfw_policy = { .memp_type = MEMP_LWFW_RULE };
lwfw_policy_t lwfw_policy_swap = { .memp_type = MEMP_LWFW_RULE_SWAP };
```

两套 `memp_type`，意味着需要两个独立的内存池。如果规则数量很多，内存占用翻倍。
