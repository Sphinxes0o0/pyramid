# LWFW 分类分析 — T-072

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: Packet classification、5-tuple 匹配、规则匹配流程

---

## 1. 概述

LWFW 使用 5-tuple (源IP, 目的IP, 协议, 源端口, 目的端口) 进行数据包分类，支持 L2/L3/L4 多层匹配：

1. **L2 匹配**: MAC 地址、VLAN、Ethertype
2. **L3 匹配**: IP 地址 (支持掩码)、协议号
3. **L4 匹配**: TCP/UDP 端口 (支持范围和列表)
4. **扩展匹配**: 接口名称、连接状态

---

## 2. 分类入口

### 2.1 ip4_filter — 主分类函数

**文件**: `lwfw.c:724-789`

```c
static int ip4_filter(lwfw_firewall_t *fw, const struct pbuf *p,
                      const struct netif *inp, lwfw_table_flag_t dir)
{
  lwfw_pkt_info_t pkt_info = {0};
  match_result_t ret_rule = {0};
  lwfw_policy_t *policy = fw->policy;
  const lwfw_backend_engine_t *filter_engine = policy->filter_engine;

  // 解析数据包信息
  lwfw_pkt_info_constructor(p, inp, &pkt_info, dir);

  // 调用后端过滤引擎
  ret = filter_engine->do_filter((void *)policy, &pkt_info, &ret_rule);

  // 生成安全事件 (如果需要)
  if (ret_rule.action & LWFW_ACTION_CODE_EVENT) {
    lwfw_generate_secure_event(&ret_rule, p, &pkt_info, ...);
  }

  return ret_rule.action;
}
```

### 2.2 两类过滤引擎

```c
// 列表搜索 (默认)
static const lwfw_backend_engine_t list_search_eng = {
  .name = "list search",
  .do_filter = list_search_do_filter,
};

// 树搜索 (可选，需要 LWFW_TREE_SEARCH_EN)
static const lwfw_backend_engine_t tree_search_eng;
```

---

## 3. 数据包解析

### 3.1 lwfw_pkt_info_constructor

**文件**: `lwfw.c:329-373`

```c
inline static void lwfw_pkt_info_constructor(const struct pbuf *p,
                                              const struct netif *inp,
                                              lwfw_pkt_info_t *pkt_info,
                                              lwfw_table_flag_t dir)
{
  const struct ip_hdr *ip_hdr;
  const void *trans_hdr;

  ip_hdr = (struct ip_hdr *)p->payload;
  trans_hdr = ((void *)p->payload) + IPH_HL_BYTES(ip_hdr);

  // L3 信息
  lwfw_pkt_l3_info_constructor(ip_hdr, l3);

  // L4 信息
  lwfw_pkt_l4_info_constructor(trans_hdr, l3->proto, l4);

  // 连接状态 (如果启用 LWCT)
  pkt_info->ct_state = lwct_convert_reply_state(p->_lwct & LWCT_STATE_MASK);

  // 接口信息
  pkt_info->interface.if_name = inp->fullname;

  // 方向
  pkt_info->dir = (dir == LWFW_IN_TABLE) ? LWFW_DIR_RX : LWFW_DIR_TX;
}
```

### 3.2 L3 信息解析

```c
inline static void lwfw_pkt_l3_info_constructor(const struct ip_hdr *ip_hdr,
                                                 lwfw_pkt_l3_info_t *l3)
{
  l3->src_ip = lwip_ntohl(ip_hdr->src.addr);    // 源 IP
  l3->dst_ip = lwip_ntohl(ip_hdr->dest.addr);    // 目的 IP
  l3->proto = ip_hdr->_proto;                    // 协议号
}
```

### 3.3 L4 信息解析

```c
inline static void lwfw_pkt_l4_info_constructor(const void *trans_hdr,
                                                 uint8_t proto,
                                                 lwfw_pkt_l4_info_t *l4)
{
  switch (proto) {
    case IP_PROTO_UDP:
    case IP_PROTO_UDPLITE:
      l4->src_port = lwip_ntohs(((struct udp_hdr *)trans_hdr)->src);
      l4->dst_port = lwip_ntohs(((struct udp_hdr *)trans_hdr)->dest);
      break;
    case IP_PROTO_TCP:
      l4->src_port = lwip_ntohs(((struct tcp_hdr *)trans_hdr)->src);
      l4->dst_port = lwip_ntohs(((struct tcp_hdr *)trans_hdr)->dest);
      break;
    default:
      l4->src_port = 0;
      l4->dst_port = 0;
  }
}
```

---

## 4. 规则匹配

### 4.1 check_rule — 规则匹配主函数

**文件**: `lwfw.c:565-613`

```c
static bool check_rule(const lwfw_rule_t *rule,
                       const lwfw_pkt_info_t *info,
                       lwfw_table_flag_t dir)
{
  // ============================================
  // Step 1: 连接状态匹配 (如果规则启用)
  // ============================================
  if ((rule->flags & LWFW_RULE_FLAGS_CT_STATE) &&
      rule->ct_state != info->ct_state) {
    return false;  // 不匹配
  }

  // ============================================
  // Step 2: 接口匹配
  // ============================================
  if ((rule->flags & LWFW_RULE_FLAGS_NETIF) &&
      strncmp(rule->interface.if_name, info->interface.if_name,
              sizeof(info->interface.if_name)) != 0) {
    return false;  // 不匹配
  }

  // ============================================
  // Step 3: L2 匹配
  // ============================================
#ifdef LWFW_ADVANCED_FUNC_L2
  if (!check_lwfw_l2_info(rule, &info->l2)) {
    return false;
  }
#endif

  // ============================================
  // Step 4: L3 匹配
  // ============================================
  if (!check_lwfw_l3_info(rule, &info->l3)) {
    return false;
  }

  // ============================================
  // Step 5: L4 匹配
  // ============================================
  if (!check_lwfw_l4_info(rule, &info->l4)) {
    return false;
  }

  return true;  // 所有条件都匹配
}
```

### 4.2 L3 匹配 — check_lwfw_l3_info

```c
inline static bool check_lwfw_l3_info(const lwfw_rule_t *rule,
                                      const lwfw_pkt_l3_info_t *packet_info)
{
  const lwfw_rule_l3_info_t *r = &rule->l3;

  // 协议匹配
  if ((rule->flags & LWFW_RULE_FLAGS_PROTOCOL) &&
      r->proto != LWFW_RULE_PROTO_IP &&
      r->proto != packet_info->proto) {
    return false;
  }

  // 源 IP 掩码匹配 (CIDR 格式)
  if (rule->flags & LWFW_RULE_FLAGS_SRC_IP_MASK_LEN) {
    uint32_t mask = ~((1UL << (32 - r->src_ip.masklen)) - 1);
    if ((packet_info->src_ip & mask) != (r->src_ip.addr & mask)) {
      return false;
    }
  }

  // 目的 IP 掩码匹配 (CIDR 格式)
  if (rule->flags & LWFW_RULE_FLAGS_DST_IP_MASK_LEN) {
    uint32_t mask = ~((1UL << (32 - r->dst_ip.masklen)) - 1);
    if ((packet_info->dst_ip & mask) != (r->dst_ip.addr & mask)) {
      return false;
    }
  }

  return true;
}
```

### 4.3 L4 匹配 — check_lwfw_l4_info

```c
inline static bool check_lwfw_l4_info(const lwfw_rule_t *rule,
                                      const lwfw_pkt_l4_info_t *packet_info)
{
  bool src_matched = false, dst_matched = false;

  // 源端口范围匹配
  if (rule->flags & LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE) {
    src_matched = (packet_info->src_port >= rule->l4.src_ports.port_range[0] &&
                   packet_info->src_port <= rule->l4.src_ports.port_range[1]);
  }
  // 源端口列表匹配
  else if (rule->flags & LWFW_RULE_FLAGS_SRC_L4_PORT_LIST) {
    for (int i = 0; i < LWFW_MAX_PORT_COUNT; i++) {
      if (rule->l4.src_ports.port_list[i] == packet_info->src_port) {
        src_matched = true;
        break;
      }
    }
  } else {
    src_matched = true;  // 未指定，匹配任意
  }

  // 目的端口范围匹配
  if (rule->flags & LWFW_RULE_FLAGS_DST_L4_PORT_RANGE) {
    dst_matched = (packet_info->dst_port >= rule->l4.dst_ports.port_range[0] &&
                   packet_info->dst_port <= rule->l4.dst_ports.port_range[1]);
  }
  // 目的端口列表匹配
  else if (rule->flags & LWFW_RULE_FLAGS_DST_L4_PORT_LIST) {
    for (int i = 0; i < LWFW_MAX_PORT_COUNT; i++) {
      if (rule->l4.dst_ports.port_list[i] == packet_info->dst_port) {
        dst_matched = true;
        break;
      }
    }
  } else {
    dst_matched = true;  // 未指定，匹配任意
  }

  return src_matched && dst_matched;
}
```

---

## 5. 规则链表遍历

### 5.1 list_search_do_filter

**文件**: `lwfw.c:1884-1978`

```c
static int list_search_do_filter(void *handle, void *data, void *result)
{
  lwfw_policy_t *policy = (lwfw_policy_t *)handle;
  lwfw_pkt_info_t *info = (lwfw_pkt_info_t *)data;
  match_result_t *ret_rule = (match_result_t *)result;

  curr_table = &policy->rule_tables[info->dir];

  // 按优先级遍历链表
  cdlist_iter_entry(curr_rule, header, next) {
    if (curr_rule->state == LWFW_STATE_DISABLE) {
      continue;  // 跳过禁用的规则
    }

    matched = check_rule(curr_rule, info, info->dir);
    if (matched) {
      break;  // 找到第一个匹配规则
    }
  }

  if (matched) {
    // 更新命中计数
    ret_rule->hit_cnt = ++curr_rule->hit_cnt;

    // 处理速率限制
    if (curr_rule->flags & LWFW_RULE_FLAGS_RATE_LIMIT) {
      // 速率限制检查...
    }

    ret_rule->action = curr_rule->action;
  } else {
    // 使用默认动作
    ret_rule->action = curr_table->def_action;
  }

  return LWFW_ERR_OK;
}
```

### 5.2 链表结构

```c
struct lwfw_rule_table {
  struct cdlist header;     // 链表头
  uint16_t rule_cnt;       // 规则数量
  uint16_t state;          // 表状态
  lwfw_action_t def_action; // 默认动作
  uint32_t def_hit_cnt;    // 默认命中计数
};

struct lwfw_rule {
  struct cdlist next;      // 链表指针
  uint16_t index;          // 规则索引
  uint16_t priority;      // 优先级 (用于排序)
  uint16_t state;         // 启用/禁用
  // ... match fields ...
};
```

---

## 6. 规则标志

### 6.1 匹配标志

```c
#define LWFW_RULE_FLAGS_CT_STATE          BIT(0)   // 连接状态匹配
#define LWFW_RULE_FLAGS_NETIF             BIT(1)   // 接口名匹配
#define LWFW_RULE_FLAGS_SRC_MAC           BIT(2)   // 源 MAC
#define LWFW_RULE_FLAGS_DST_MAC           BIT(3)   // 目的 MAC
#define LWFW_RULE_FLAGS_ETHER_TYPE        BIT(4)   // Ethertype
#define LWFW_RULE_FLAGS_VLAN              BIT(5)   // VLAN
#define LWFW_RULE_FLAGS_PROTOCOL          BIT(6)   // L3 协议
#define LWFW_RULE_FLAGS_SRC_IP_MASK       BIT(7)   // 源 IP 掩码
#define LWFW_RULE_FLAGS_SRC_IP_MASK_LEN   BIT(8)   // 源 IP CIDR
#define LWFW_RULE_FLAGS_DST_IP_MASK       BIT(9)   // 目的 IP 掩码
#define LWFW_RULE_FLAGS_DST_IP_MASK_LEN   BIT(10)  // 目的 IP CIDR
#define LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE BIT(11)  // 源端口范围
#define LWFW_RULE_FLAGS_SRC_L4_PORT_LIST  BIT(12)  // 源端口列表
#define LWFW_RULE_FLAGS_DST_L4_PORT_RANGE BIT(13)  // 目的端口范围
#define LWFW_RULE_FLAGS_DST_L4_PORT_LIST  BIT(14)  // 目的端口列表
#define LWFW_RULE_FLAGS_RATE_LIMIT        BIT(15)  // 速率限制
```

---

## 7. 分类流程图

```
数据包到达
    │
    ▼
lwfw_pkt_info_constructor()
    │
    ├─► 解析 L3: src_ip, dst_ip, proto
    ├─► 解析 L4: src_port, dst_port
    ├─► 解析接口名
    └─► 解析连接状态 (LWCT)
    │
    ▼
filter_engine->do_filter()
    │
    ▼
遍历规则链表 (按优先级)
    │
    ├─► check_rule()
    │     ├─► CT_STATE 匹配?
    │     ├─► NETIF 匹配?
    │     ├─► L2 匹配? (MAC, VLAN, Ethertype)
    │     ├─► L3 匹配? (IP, Protocol)
    │     └─► L4 匹配? (Ports)
    │
    ▼
找到匹配规则?
    │
    ├─► 是 → 应用规则动作 (ALLOW/DENY/EVENT)
    │         ├─► 增加 hit_cnt
    │         ├─► 检查速率限制
    │         └─► 生成事件 (如果需要)
    │
    └─► 否 → 应用默认动作
```

---

## 8. 动作类型

### 8.1 动作代码

```c
typedef enum {
  LWFW_ACTION_CODE_ALLOW   = 0x01,  // 允许通过
  LWFW_ACTION_CODE_DENY     = 0x02,  // 拒绝 (丢弃)
  LWFW_ACTION_CODE_EVENT    = 0x04,  // 生成事件
  LWFW_ACTION_CODE_LOGGING = 0x08,  // 记录日志
} lwfw_action_code_t;
```

### 8.2 速率限制动作

```c
typedef enum {
  LWFW_RLIMIT_STATE_NORMAL = 0,  // 正常状态
  LWFW_RLIMIT_STATE_LIMIT  = 1,  // 限速状态
} lwfw_rlimit_state_t;

// 限速触发后，DENY 动作被添加到规则动作
if (curr_rule->rlimit.state == LWFW_RLIMIT_STATE_LIMIT) {
  ret_rule->action |= LWFW_ACTION_CODE_DENY;
}
```

---

## 9. 与 Linux iptables 对比

| 特性 | LWFW | Linux iptables |
|------|------|----------------|
| **匹配层次** | L2/L3/L4 | L2/L3/L4/L5 |
| **IP 掩码** | CIDR 格式 | 完整掩码 |
| **端口匹配** | 范围 + 列表 | 范围 |
| **连接状态** | LWCT 集成 | conntrack |
| **规则组织** | 链表 (按优先级) | 链表 (按位置) |
| **默认动作** | 每表独立 | 每链独立 |

---

## 10. 总结

### 10.1 分类关键点

1. **解析**: 从 pbuf 提取 L2/L3/L4 字段
2. **匹配**: 按优先级遍历规则链表，依次检查各层匹配条件
3. **动作**: ALLOW/DENY/EVENT/LOGGING，可组合
4. **计数**: hit_cnt 统计规则命中次数

### 10.2 匹配顺序

```
1. CT_STATE (连接状态)
2. NETIF (接口名)
3. L2 (MAC, VLAN, Ethertype)
4. L3 (IP, Protocol)
5. L4 (Ports)
```

### 10.3 性能考虑

- 链表搜索：O(n) 线性遍历
- 规则数量大时考虑树搜索优化 (LWFW_TREE_SEARCH_EN)
- 支持数据预取 (LWFW_PREFETCH) 加速遍历
