# lwfw 核心过滤逻辑分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw.c`
> 版本: 1.9.0

---

## 1. 全局变量与初始化

### 1.1 全局状态

```c
lwfw_firewall_t g_lwfw_firewall, *lwfw_p;          // [lwfw.c:36] 防火墙全局句柄
lwfw_policy_t lwfw_policy = { .memp_type = MEMP_LWFW_RULE };           // 活跃策略
lwfw_policy_t lwfw_policy_swap = { .memp_type = MEMP_LWFW_RULE_SWAP }; // 热切换备份
uint32_t g_lwfw_curr_log_cnt = 0;                                      // 当前秒日志计数
struct stats_filter g_lwfw_stats = {0};                                // 防火墙统计
lwfw_log_level_t lwfw_log_level = LWFW_DEF_LOG_LEVEL;                 // 日志级别
```

### 1.2 初始化流程 (`lwfw_init`)

```
lwfw_init()
  ├─ memset(g_lwfw_firewall, 0)
  ├─ sync_mutex_new(&lwfw_p->policy_lock)
  ├─ lwfw_policies_setup()          → 初始化 policy / inactive_policy 指针
  ├─ lwfw_manifest_parse()         → 解析 YAML 配置文件
  ├─ lwfw_init_policy(policy, cfg) → 初始化规则表、过滤引擎
  ├─ lwfw_init_policy(inactive_policy, NULL)  → 初始化备份策略
  ├─ lwct_init()                    → 连接跟踪初始化
  ├─ sys_thread_new(lwfw_notification_thread) → 启动事件通知线程
  └─ fw_cur_status = LWFW_STATUS_READY
```

**关键设计**: 使用 `policy` / `inactive_policy` 双缓冲实现策略热切换，切换时原子交换指针。

---

## 2. 过滤引擎架构

### 2.1 后端引擎抽象

```c
typedef struct lwfw_backend_engine {
  char name[16];
  int (*init)(void *handle, void *data);
  int (*deinit)(void *handle, void *data);
  int (*do_filter)(void *handle, void *data, void* result);  // 核心过滤函数
  int (*dump)(void *handle, void *data);
} lwfw_backend_engine_t;
```

### 2.2 两套引擎

| 引擎 | 名称 | 适用场景 |
|------|------|----------|
| `list_search_eng` | "list search" | 默认引擎，规则少时使用 |
| `tree_search_eng` | "tree search" | 规则多时启用 (`LWFW_FILTER_TREE`) |

引擎在 `lwfw_init_policy()` 中根据 `filter_mode` 参数选择。

---

## 3. Ingress 过滤 (`ip4_filter_dispatch_incoming`)

### 3.1 函数签名与路径

```c
static int ip4_filter_dispatch_incoming(const struct pbuf *p, const struct netif *inp)
// 调用路径: ip4_input() → lwfw_p->ops->ingress_filter(p, inp)
```

### 3.2 决策逻辑

```
ip4_filter_dispatch_incoming(p, inp)
  │
  ├─ [IN_TABLE state == DISABLE?] ──是──► return ERR_OK (放行)
  │
  ├─ ip4_filter(lwfw_p, p, inp, LWFW_IN_TABLE)
  │     │
  │     ├─ lwct 兜底检查
  │     ├─ lwfw_pkt_info_constructor()  ←── 从 pbuf 解析包信息
  │     └─ filter_engine->do_filter()  ←── 调用过滤引擎
  │
  ├─ [ret & DENY] ──是──► total_rx_drop++ ; return ERR_VAL (丢弃)
  └─ return ERR_OK (放行)
```

---

## 4. Egress 过滤 (`ip4_filter_dispatch_outgoing`)

```c
static int ip4_filter_dispatch_outgoing(const struct pbuf *p, const struct netif *inp)
// 调用路径: ip4_output_if() → lwfw_p->ops->egress_filter(p, netif)
```

流程与 Ingress 基本一致，区别在于 `dir = LWFW_OUT_TABLE`，且 Egress 方向 L2 字段不填充（Ethernet 头尚未构建）。

---

## 5. 数据包解析 (`lwfw_pkt_info_constructor`)

**位置**: `lwfw.c:329-373`

```c
inline static void lwfw_pkt_info_constructor(p, inp, pkt_info, dir)
{
  // 1. 解析 IP 头 + 传输层头
  ip_hdr = p->payload;
  trans_hdr = p->payload + IPH_HL_BYTES(ip_hdr) * 4;  // IP头长度可变

  // 2. 提取 L3 信息 (src_ip, dst_ip, proto)
  l3->src_ip = lwip_ntohl(ip_hdr->src.addr);
  l3->dst_ip = lwip_ntohl(ip_hdr->dest.addr);
  l3->proto  = ip_hdr->_proto;

  // 3. 提取 L4 信息 (src_port, dst_port)
  switch (proto) {
    case TCP:  src = TCP_HDR.src;  dst = TCP_HDR.dst;  break;
    case UDP:  src = UDP_HDR.src;  dst = UDP_HDR.dst;  break;
    default:   src = 0;            dst = 0;              break;
  }

  // 4. Ingress 方向额外解析 L2 (Ethernet + VLAN)
  if (dir == LWFW_IN_TABLE) {
    eth_hdr = p->payload + SIZEOF_STRUCT_PBUF;  // pbuf 头部有私有数据
    l2->ether_type = eth_hdr->type;
    l2->vlan = VLAN_ID(vlan_hdr);
    memcpy(l2->src_mac, eth_hdr->src.addr, 6);
    memcpy(l2->dst_mac, eth_hdr->dest.addr, 6);
  }

  // 5. 连接跟踪状态 (lwct)
  pkt_info->ct_state = lwct_convert_reply_state(p->_lwct & LWCT_STATE_MASK);
}
```

**注意**: L2 解析只在 `LWFW_ADVANCED_FUNC_L2` 开启时有效，默认关闭。

---

## 6. 链表过滤引擎 (`list_search_do_filter`)

**位置**: `lwfw.c:1884-1978`

### 6.1 核心算法 (First-Match)

```c
static int list_search_do_filter(void *handle, void *data, void *result)
{
  curr_table = &policy->rule_tables[info->dir];
  if (curr_table->rule_cnt == 0)
    goto default_action;

  // 遍历规则链表 (按插入顺序)
  cdlist_iter_entry(curr_rule, header, next) {
    if (curr_rule->state == LWFW_STATE_DISABLE)
      continue;

    matched = check_rule(curr_rule, info, info->dir);
    if (matched)
      break;  // 首次匹配即停止
  }

  if (matched && curr_rule != NULL) {
    ret_rule->match_rule = curr_rule;
    ret_rule->rule_id = curr_rule->index;
    ret_rule->action = curr_rule->action;
    ret_rule->hit_cnt = ++curr_rule->hit_cnt;

    // 速率限制检查
    if (flags & RATE_LIMIT) {
      __atomic_fetch_add(&rlimit.rx_pps, 1, __ATOMIC_RELAXED);
      if (rlimit.state == LIMIT && rx_pps > rate)
        action |= DENY;  // 超速拒绝
    }
  } else {
default_action:
    action = curr_table->def_action;  // 无匹配使用默认动作
  }
}
```

### 6.2 时间复杂度

- **最佳**: O(1)（首条规则匹配）
- **最差**: O(n)（无匹配或末条匹配，n = 规则数）
- **平均**: O(n/2)

---

## 7. 规则匹配 (`check_rule`)

**位置**: `lwfw.c:565-613`

### 7.1 匹配顺序

```c
static bool check_rule(rule, info, dir)
{
  // 1. 连接跟踪状态匹配 (lwct)
  if (flags & CT_STATE && rule->ct_state != info->ct_state)
    return false;

  // 2. 网卡接口匹配
  if (flags & NETIF && strncmp(rule->if_name, info->if_name, ...) != 0)
    return false;

  // 3. L2 字段匹配 (需要 LWFW_ADVANCED_FUNC_L2)
  if (!check_lwfw_l2_info(rule, &info->l2))
    return false;

  // 4. L3 字段匹配
  if (!check_lwfw_l3_info(rule, &info->l3))
    return false;

  // 5. L4 字段匹配
  if (!check_lwfw_l4_info(rule, &info->l4))
    return false;

  return true;
}
```

### 7.2 L2 匹配 (`check_lwfw_l2_info`)

支持字段:
- `ETHER_TYPE`: 精确匹配以太网类型
- `VLAN`: 精确匹配 VLAN ID
- `SRC_MAC`: 源 MAC + 掩码匹配
- `DST_MAC`: 目标 MAC + 掩码匹配

**掩码匹配算法**:
```c
// 问: (mask[i] & packet[i]) == (addr[i] & mask[i]) ?
// 例: MAC 00:11:22:33:44:55, mask FF:FF:FF:00:00:00
//     只比较前3字节
```

### 7.3 L3 匹配 (`check_lwfw_l3_info`)

支持字段:
- `PROTOCOL`: 精确匹配 (TCP/UDP/ICMP/IP)
- `SRC_IP_MASK`: 源 IP + 任意掩码
- `SRC_IP_MASK_LEN`: 源 IP + CIDR 前缀长度
- `DST_IP_MASK` / `DST_IP_MASK_LEN`: 同上

**CIDR 掩码计算**:
```c
// prefix=24 → mask = ~((1 << (32-24)) - 1) = 0xFFFFFF00
rule_mask = (uint32_t)(~((1UL << (32 - masklen)) - 1));
```

### 7.4 L4 匹配 (`check_lwfw_l4_info`)

支持两种模式:
- **端口范围**: `portBegin` / `portEnd` 定义区间 `[begin, end]`
- **端口列表**: 最多 4 个端口，逐一比较

```c
// 范围匹配
src_matched = (src_port >= range[0] && src_port <= range[1]);

// 列表匹配
for (i=0; i<LWFW_MAX_PORT_COUNT; i++) {
  if (port_list[i] == 0) break;
  if (port_list[i] == src_port) { src_matched = 1; break; }
}
```

---

## 8. 速率限制 (`rate_limit`)

### 8.1 状态机

```
状态转换:
NORMAL ──(rx_pps >= burst)──► LIMIT ──(time >= expire)──► NORMAL

限速逻辑 (在 list_search_do_filter 中):
1. rx_pps++ (原子)
2. LIMIT 状态下:
   - 如果 action 不是 DENY 且 rx_pps > rate → 拒绝
   - event 只在 EDGE 模式上报一次
3. NORMAL 状态下: 不上报 event (静音)
```

### 8.2 配置参数

```c
rate_limit_t {
  char name[32];
  uint32_t burst;      // 桶容量 (pps)
  uint32_t rate;      // 速率上限 (pps)
  uint32_t expire;    // 限速持续时间 (秒)，0=永久
  uint32_t event_mode; // LEVEL=1 / EDGE=0
  uint32_t rx_pps;    // 当前速率 (运行时)
  uint32_t time;      // 已限速时长
  uint32_t drops;     // 累计丢包数
  uint16_t occurs;    // 进入限速状态次数
  uint16_t interval;
};
```

---

## 9. 热切换机制 (`lwfw_config_reset_state`)

### 9.1 原子切换流程

```c
// 适用于 TREE 模式
sync_mutex_lock(&lwfw_p->policy_lock);
lwfw_policy_clean(inactive_policy);
lwfw_copy_policy(inactive_policy, policy);      // 深拷贝规则
inactive_policy->filter_engine->init(...);        // 重建树索引
tmp = lwfw_p->policy;
lwfw_p->policy = lwfw_p->inactive_policy;         // 原子切换指针
lwfw_p->inactive_policy = tmp;
sync_mutex_unlock(&lwfw_p->policy_lock);
```

### 9.2 风险点

1. `lwfw_copy_policy` 深拷贝时，如果规则数量很大，会阻塞数据包处理
2. `inactive_policy` 内存池 (`MEMP_LWFW_RULE_SWAP`) 需要预先分配足够大
3. 切换期间 `policy_lock` 持锁，可能影响实时性

---

## 10. 事件生成 (`lwfw_generate_secure_event`)

### 10.1 触发条件

```c
if (ret_rule.action & LWFW_ACTION_CODE_EVENT) {
  lwfw_generate_secure_event(&ret_rule, p, &pkt_info, ...);
}
```

### 10.2 限速检查

```c
// 事件 PPS 限速
if (event_rlimit_rate != 0 && event_pps >= event_rlimit_rate)
  return;  // 丢弃

// FIFO 满检查
if (lwfw_event_fifo_is_full()) {
  drop_events++;  // 统计丢事件
  return;
}
```

### 10.3 事件数据结构

```c
lwfw_event_t {
  lwfw_event_hdr_t hdr;    // version, flag, event_type, count, timestamp
  lwfw_event_data_t data;  // rule_id, action, proto, ip, port, mac, vlan...
}
```

---

## 11. 优化建议

### 11.1 性能优化

| 问题 | 位置 | 建议 |
|------|------|------|
| 规则链表无索引 | `list_search_do_filter` | 规则>20条时启用 tree 模式 |
| 深拷贝阻塞持锁 | `lwfw_copy_policy` | 改为 RCU 无锁切换 |
| pbuf 头解析偏移 | `lwfw_pkt_info_constructor` | `SIZEOF_STRUCT_PBUF` 依赖编译期常量 |
| L2 解析默认关闭 | `LWFW_ADVANCED_FUNC_L2` | 考虑默认启用，减少条件编译 |

### 11.2 稳定性优化

| 问题 | 位置 | 建议 |
|------|------|------|
| 静态 parser state | `lwfw_parser.c:22` | 多线程 YAML 解析时状态机会冲突 |
| 深拷贝无错误恢复 | `lwfw_copy_policy` | 拷贝失败时 active_policy 已被污染 |
| rlimit interval 硬编码 | `lwfw_notification_timer_thread` | `pkt_rlimit_interval` 为0时行为不确定 |

### 11.3 代码质量

| 问题 | 位置 | 建议 |
|------|------|------|
| 大量 goto 错误处理 | 多处 | 重构为统一错误传播 |
| 魔法数字 | `lwfw.c:39-40` | `LWFW_TREE_BUCKET_SIZE=8`, `LWFW_TREE_NODE_NUM=64` 应可配置 |
| 日志宏重复定义 | `lwfw_common.h` | 与 `LWCT_PRINTF` 大量重复 |
