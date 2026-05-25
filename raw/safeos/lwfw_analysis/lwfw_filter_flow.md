# lwfw 规则匹配流程深度分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw.c`
> 相关文件: `lwfw_parser.c`, `tree_entry.c`

---

## 1. 过滤入口

### 1.1 Ingress 入口

```c
// lwfw.c:802-834
static int ip4_filter_dispatch_incoming(const struct pbuf *p, const struct netif *inp)
{
  int ret = ERR_OK;

  // 检查 Ingress 规则表是否启用
  if (lwfw_p->policy->rule_tables[LWFW_IN_TABLE].state == LWFW_STATE_DISABLE)
    return ret;

  LWIP_ASSERT_CORE_LOCKED();
  LWIP_ASSERT("invalid pbuf *p", (p != NULL));
  LWIP_ASSERT("invalid netif *inp", (inp != NULL));

  ret = ip4_filter(lwfw_p, p, inp, LWFW_IN_TABLE);

  // DENY 动作返回 ERR_VAL，触发丢包
  if ((ret & LWFW_ACTION_CODE_DENY) == LWFW_ACTION_CODE_DENY) {
    LWFW_STATICS_INC(g_lwfw_stats.total_rx_drop);
    return ERR_VAL;
  }
  return ERR_OK;
}
```

### 1.2 Egress 入口

```c
// lwfw.c:847-880
static int ip4_filter_dispatch_outgoing(const struct pbuf *p, const struct netif *inp)
{
  int ret = ERR_OK;

  if (lwfw_p->policy->rule_tables[LWFW_OUT_TABLE].state == LWFW_STATE_DISABLE)
    return ret;

  // ... 类似 Ingress 处理
  ret = ip4_filter(lwfw_p, p, inp, LWFW_OUT_TABLE);

  if ((ret & LWFW_ACTION_CODE_DENY) == LWFW_ACTION_CODE_DENY) {
    LWFW_STATICS_INC(g_lwfw_stats.total_tx_drop);
    return ERR_VAL;
  }
  return ERR_OK;
}
```

---

## 2. ip4_filter 主函数

```c
// lwfw.c:724-789
static int ip4_filter(lwfw_firewall_t *fw, const struct pbuf *p,
                      const struct netif *inp, lwfw_table_flag_t dir)
{
  int ret = LWFW_ERR_OK;
  lwfw_pkt_info_t pkt_info = {0};
  match_result_t ret_rule = {0};
  lwfw_policy_t *policy = fw->policy;
  const lwfw_backend_engine_t *filter_engine = policy->filter_engine;

  // 1. lwct 未启用/未跟踪时的兜底处理
#ifdef NIO_LWIP_LWCT
  if (lwct_enable == 1 && !p->_lwct) {
    LWFW_STATICS_INC(g_lwfw_stats.ct_notrack);
    if (policy->params.ct_oot_action == LWFW_CT_OOT_ACTION_PASS) {
      return ERR_OK;  // 放行未跟踪的包
    }
  }
#endif

  // 2. 解析包信息 (L2/L3/L4 字段提取)
  lwfw_pkt_info_constructor(p, inp, &pkt_info, dir);

  // 3. 调用过滤器引擎进行规则匹配
  ret = filter_engine->do_filter((void *)policy, (void *)&pkt_info, (void *)&ret_rule);

  // 4. 如果匹配到规则需要上报事件，生成安全事件
  if (ret_rule.action & LWFW_ACTION_CODE_EVENT) {
    ret = lwfw_generate_secure_event(&ret_rule, p, &pkt_info,
                                      sizeof(pkt_info), ret_rule.hit_cnt);
  }

  return ret_rule.action;
}
```

---

## 3. 包信息解析

### 3.1 lwfw_pkt_info_constructor

```c
// lwfw.c:329-373
inline static void lwfw_pkt_info_constructor(...)
{
  // 1. 提取 L3/L4 头指针
  ip_hdr = (struct ip_hdr *)p->payload;
  trans_hdr = ((void *)p->payload) + IPH_HL_BYTES(ip_hdr);

  // 2. 提取连接状态 (从 pbuf->_lwct 扩展字段)
#ifdef NIO_LWIP_LWCT
  pkt_info->ct_state = lwct_convert_reply_state(p->_lwct & LWCT_STATE_MASK);
#endif

  // 3. 提取接口名
  if (inp->fullname) {
    memcpy(pkt_info->interface.if_name, inp->fullname, ...);
  } else {
    memcpy(pkt_info->interface.if_name, inp->name, ...);
  }

  // 4. 提取 L2 信息 (仅 Ingress)
  if (dir == LWFW_IN_TABLE) {
#ifdef LWFW_ADVANCED_FUNC_L2
    eth_hdr = (struct eth_hdr *)((uint8_t *)p + SIZEOF_STRUCT_PBUF);
    lwfw_pkt_l2_info_constructor(eth_hdr, dir, l2);
#endif
  }

  // 5. 提取 L3/L4 信息
  lwfw_pkt_l3_info_constructor(ip_hdr, l3);
  lwfw_pkt_l4_info_constructor(trans_hdr, l3->proto, l4);
}
```

### 3.2 L2 解析 (VLAN 支持)

```c
// lwfw.c:232-259
inline static void lwfw_pkt_l2_info_constructor(...)
{
  if (dir == LWFW_IN_TABLE) {
#if ETHARP_SUPPORT_VLAN
    struct eth_vlan_hdr *eth_vlan_hdr = NULL;
    if (eth_hdr->type == PP_HTONS(ETHTYPE_VLAN)) {
      eth_vlan_hdr = (struct eth_vlan_hdr *)((void*)eth_hdr + ETHER_HDR_LEN);
      l2->vlan = VLAN_ID(eth_vlan_hdr);  // 提取 VLAN ID
    } else {
      l2->vlan = 0;
    }
#endif
    l2->ether_type = eth_hdr->type;
    memcpy(l2->src_mac, eth_hdr->src.addr, ETH_HWADDR_LEN);
    memcpy(l2->dst_mac, eth_hdr->dest.addr, ETH_HWADDR_LEN);
  }
}
```

### 3.3 L3 解析

```c
// lwfw.c:269-279
inline static void lwfw_pkt_l3_info_constructor(...)
{
  l3->src_ip = lwip_ntohl(ip_hdr->src.addr);
  l3->dst_ip = lwip_ntohl(ip_hdr->dest.addr);
  l3->proto = ip_hdr->_proto;
}
```

### 3.4 L4 解析

```c
// lwfw.c:288-318
inline static void lwfw_pkt_l4_info_constructor(...)
{
  switch (proto) {
    case IP_PROTO_UDP:
    case IP_PROTO_UDPLITE:
      src_port = lwip_ntohs(((struct udp_hdr *)trans_hdr)->src);
      dst_port = lwip_ntohs(((struct udp_hdr *)trans_hdr)->dest);
      break;
    case IP_PROTO_TCP:
      src_port = lwip_ntohs(((struct tcp_hdr *)trans_hdr)->src);
      dst_port = lwip_ntohs(((struct tcp_hdr *)trans_hdr)->dest);
      break;
    default:
      src_port = 0;
      dst_port = 0;
  }
  l4->src_port = src_port;
  l4->dst_port = dst_port;
}
```

---

## 4. 规则匹配 check_rule

### 4.1 匹配顺序

```c
// lwfw.c:565-613
static bool check_rule(const lwfw_rule_t *rule, const lwfw_pkt_info_t *info,
                       lwfw_table_flag_t dir)
{
  bool matched = 1;

  // 1. CT_STATE 连接状态匹配 (可选)
#ifdef NIO_LWIP_LWCT
  if ((rule->flags & LWFW_RULE_FLAGS_CT_STATE) &&
      rule->ct_state != info->ct_state) {
    return false;
  }
#endif

  // 2. NETIF 接口匹配 (可选)
  if ((rule->flags & LWFW_RULE_FLAGS_NETIF) &&
      strncmp(rule->interface.if_name, info->interface.if_name, ...) != 0) {
    return false;
  }

  // 3. L2 信息匹配 (需要 LWFW_ADVANCED_FUNC_L2)
#ifdef LWFW_ADVANCED_FUNC_L2
  matched = check_lwfw_l2_info(rule, &info->l2);
  if (!matched) return false;
#endif

  // 4. L3 信息匹配
  matched = check_lwfw_l3_info(rule, &info->l3);
  if (!matched) return false;

  // 5. L4 信息匹配
  matched = check_lwfw_l4_info(rule, &info->l4);
  if (!matched) return false;

  return matched;
}
```

### 4.2 L2 匹配详情

```c
// lwfw.c:383-437
inline static bool check_lwfw_l2_info(...)
{
  // EtherType 匹配 (精确)
  if ((rule->flags & LWFW_RULE_FLAGS_ETHER_TYPE) &&
      rule_l2_info->ether_type != packet_info->ether_type)
    goto out;

  // VLAN 匹配 (精确)
  if ((rule->flags & LWFW_RULE_FLAGS_VLAN) &&
      rule_l2_info->vlan != packet_info->vlan)
    goto out;

  // Src MAC + Mask 匹配
  if (rule->flags & LWFW_RULE_FLAGS_SRC_MAC) {
    for (int i = 0; i < ETH_HWADDR_LEN; i++) {
      if ((rule_l2_info->src_mac.mask[i] & packet_info->src_mac[i]) !=
          (rule_l2_info->src_mac.addr[i] & rule_l2_info->src_mac.mask[i]))
        goto out;
    }
  }

  // Dst MAC + Mask 匹配 (同上)
  if (rule->flags & LWFW_RULE_FLAGS_DST_MAC) { ... }

  matched = 1;
out:
  return matched;
}
```

### 4.3 L3 匹配详情

```c
// lwfw.c:447-488
inline static bool check_lwfw_l3_info(...)
{
  // Protocol 匹配 (精确, 0xFF 表示任意协议)
  if ((rule->flags & LWFW_RULE_FLAGS_PROTOCOL) &&
      rule_l3_info->proto != LWFW_RULE_PROTO_IP &&
      rule_l3_info->proto != packet_info->proto)
    goto out;

  // Src IP + Mask 匹配
  if (rule->flags & LWFW_RULE_FLAGS_SRC_IP_MASK_LEN) {
    rule_mask = ~((1UL << (32 - rule_l3_info->src_ip.masklen)) - 1);
    if ((packet_info->src_ip & rule_mask) != (rule_l3_info->src_ip.addr & rule_mask))
      goto out;
  } else if (rule->flags & LWFW_RULE_FLAGS_SRC_IP_MASK) {
    if ((packet_info->src_ip & rule_l3_info->src_ip.mask) !=
        (rule_l3_info->src_ip.addr & rule_l3_info->src_ip.mask))
      goto out;
  }

  // Dst IP + Mask 匹配 (同上)
  if (rule->flags & LWFW_RULE_FLAGS_DST_IP_MASK_LEN) { ... }
  else if (rule->flags & LWFW_RULE_FLAGS_DST_IP_MASK) { ... }

  matched = 1;
out:
  return matched;
}
```

### 4.4 L4 匹配详情

```c
// lwfw.c:498-554
inline static bool check_lwfw_l4_info(...)
{
  // Src Port 匹配 (范围或列表)
  if (rule->flags & LWFW_RULE_FLAGS_SRC_L4_PORT_RANGE) {
    if ((packet_info->src_port >= rule_l4_info->src_ports.port_range[0]) &&
        (rule_l4_info->src_ports.port_range[1] >= packet_info->src_port))
      src_matched = 1;
  } else if (rule->flags & LWFW_RULE_FLAGS_SRC_L4_PORT_LIST) {
    for (int i = 0; i < LWFW_MAX_PORT_COUNT; i++) {
      if (rule_l4_info->src_ports.port_list[i] == packet_info->src_port) {
        src_matched = 1;
        break;
      }
    }
  } else {
    src_matched = 1;  // 无 Src Port 限制
  }

  // Dst Port 匹配 (同上)
  if (rule->flags & LWFW_RULE_FLAGS_DST_L4_PORT_RANGE) { ... }
  else if (rule->flags & LWFW_RULE_FLAGS_DST_L4_PORT_LIST) { ... }
  else dst_matched = 1;

  return (src_matched & dst_matched);
}
```

---

## 5. 匹配结果处理

### 5.1 动作编码

```c
// lwfw_common.h:38-42
typedef enum lwfw_action {
  LWFW_ACTION_CODE_DENY   = BIT(0),  // 丢弃包
  LWFW_ACTION_CODE_EVENT  = BIT(1),  // 上报事件
  LWFW_ACTION_CODE_LOGGING = BIT(2),  // 记录日志
} lwfw_action_t;
```

### 5.2 事件生成

```c
// lwfw.c:641-708
static int lwfw_generate_secure_event(match_result_t *ret_rule, ...)
{
  // 1. 限速检查
  if (lwfw_event_need_throttling()) return;

  // 2. FIFO 满检查
  if (lwfw_event_fifo_is_full()) {
    LWFW_STATICS_INC(g_lwfw_stats.drop_events);
    return LWFW_ERR_FIFO_FULL;
  }

  // 3. 构建事件
  event.hdr.event_type = LOG_KIND_EVENT;
  event.data.event_id = (rule->flags & LWFW_RULE_FLAGS_RATE_LIMIT) ?
                        LWFW_SEV_RATELIMIT : LWFW_SEV_GENERIC_FILTER;
  event.data.rule_id = ret_rule->rule_id;
  event.data.action = ret_rule->action;
  // ... 填充更多字段 ...

  // 4. 写入 FIFO
  ret = lwfw_event_push(&event);
}
```

---

## 6. 过滤流程图

```
数据包进入
    │
    ▼
ip4_filter_dispatch_incoming/outgoing()
    │ 检查规则表 state
    ▼
ip4_filter()
    │
    ├─ lwct 未跟踪兜底检查
    │
    ├─ lwfw_pkt_info_constructor()
    │     ├─ 提取接口名
    │     ├─ L2 解析 (VLAN/MAC/EtherType)
    │     ├─ L3 解析 (IP/Proto)
    │     └─ L4 解析 (Port)
    │
    ├─ filter_engine->do_filter()
    │     │
    │     └─ list_search_do_filter() 或 tree_search_do_filter()
    │           │
    │           └─ 遍历规则，调用 check_rule()
    │                 ├─ check_lwfw_l2_info()
    │                 ├─ check_lwfw_l3_info()
    │                 └─ check_lwfw_l4_info()
    │
    ├─ 如匹配且需事件 → lwfw_generate_secure_event()
    │
    └─ return action (ALLOW/DENY/EVENT)
```

---

## 7. 关键代码位置

| 函数 | 文件 | 行号 |
|------|------|------|
| `ip4_filter_dispatch_incoming` | lwfw.c | 802 |
| `ip4_filter_dispatch_outgoing` | lwfw.c | 847 |
| `ip4_filter` | lwfw.c | 724 |
| `lwfw_pkt_info_constructor` | lwfw.c | 329 |
| `lwfw_pkt_l2_info_constructor` | lwfw.c | 232 |
| `lwfw_pkt_l3_info_constructor` | lwfw.c | 269 |
| `lwfw_pkt_l4_info_constructor` | lwfw.c | 288 |
| `check_rule` | lwfw.c | 565 |
| `check_lwfw_l2_info` | lwfw.c | 383 |
| `check_lwfw_l3_info` | lwfw.c | 447 |
| `check_lwfw_l4_info` | lwfw.c | 498 |
| `lwfw_generate_secure_event` | lwfw.c | 641 |
