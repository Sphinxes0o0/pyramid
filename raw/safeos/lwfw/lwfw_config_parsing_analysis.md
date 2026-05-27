# LWFW 配置解析分析 — T-083

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: YAML 配置解析、rule 加载、策略初始化

---

## 1. 概述

LWFW 使用 YAML 格式的配置文件来定义防火墙策略和规则：

1. **配置文件路径**: `/etc/lwfw/*.yaml`
2. **解析库**: libyaml (C YAML 解析器)
3. **配置结构**: apiVersion, kind, metadata, spec (default, parameters, ingress, egress)

---

## 2. 配置结构

### 2.1 YAML 配置示例

```yaml
apiVersion: nt3.networking.firewall/v1
kind: NetworkPolicy
metadata:
  name: vdf-a-network-firewall-policy
  namespace: default
  version: 0
  revision: 1

spec:
  default:
    ingress:
       action: allow
       event: false
       log: false
    egress:
       action: allow
       event: false
       log: false
  ingress:
    state: disable
  egress:
    state: disable
```

### 2.2 带规则的配置

```yaml
spec:
  default:
    ingress:
       action: allow
       event: false
       log: false
    egress:
       action: allow
       event: false
       log: false

  ingress:
    state: enable
    rules:
    - index: 1
      name: sample-rule
      state: enable
      action: deny
      event: false
      log: false
      protocol: ip
      from:
        L2:
          macAddress: ff:ff:ff:ff:ff:ff
          macMask: 00:00:00:00:00:00
        L3:
          ipBlock:
            ip: 172.20.0.0
            prefix: 0
        L4:
          list:
            - 65527
            - 65528
      to:
        L2:
          macAddress: ff:ff:ff:ff:ff:ff
          macMask: 00:00:00:00:00:00
        L3:
          ipBlock:
            ip: 172.20.0.0
            prefix: 0
        L4:
          list:
            - 65527
            - 65528
```

---

## 3. 解析入口

### 3.1 lwfw_config_init

**文件**: `lwfw_parser.c:2442-2530`

```c
struct lwfw_policy_config* lwfw_config_init(const char *cpio_filename,
                                             const char *filename)
{
  // 1. 分配配置结构
  fw_cfg = malloc(sizeof(struct lwfw_policy_config));

  // 2. 设置默认参数
  lwfw_policy_set_default_parameters(fw_cfg);

  // 3. 分配规则内存
  fw_rules = malloc(sizeof(struct lwfw_rule_config) * MAX_RULE_COUNT);

  // 4. 等待文件系统就绪
  sys_svc_wait("/", SVC_WAIT_EXACT, 0);

  // 5. 解析 YAML 文件
  err = read_lwfw_rules(filename, fw_cfg, fw_rules, false);
  if (err != LWFW_PARSER_SUCCESS && cpio_filename) {
    // 尝试从 cpio 归档中读取
    err = read_lwfw_rules(cpio_filename, fw_cfg, fw_rules, true);
  }

  // 6. 验证配置
  verify_lwfw_policy_config(fw_cfg);
  for (i = 0; i < fw_cfg->in_rules_num; i++) {
    verify_lwfw_rule_config(&fw_cfg->in_rules[i]);
  }
}
```

---

## 4. YAML 解析器

### 4.1 read_lwfw_rules

**文件**: `lwfw_parser.c:2257-2338`

```c
static lwfw_parser_err_t read_lwfw_rules(const char *filename,
                                          struct lwfw_policy_config *fw_cfg,
                                          struct lwfw_rule_config *rules,
                                          bool from_cpio)
{
  yaml_parser_t parser;

  // 初始化 YAML 解析器
  yaml_parser_initialize(&parser);

  if (from_cpio) {
    // 从 CPIO 归档中获取文件内容
    lwfw_conf = cpio_get_file(_cpio_archive, cpio_len, filename, &size);
    yaml_parser_set_input_string(&parser, lwfw_conf, size);
  } else {
    // 从文件系统读取
    file = fopen(filename, "rb");
    yaml_parser_set_input_file(&parser, file);
  }

  // 事件循环
  do {
    yaml_event_t event;
    if (!yaml_parser_parse(&parser, &event)) {
      return LWFW_PARSER_INVALID_YAML_FORMAT;
    }
    ret = consume_event(&state, &event);
    yaml_event_delete(&event);
  } while (state.state != STATE_STOP);

  yaml_parser_delete(&parser);
}
```

### 4.2 状态机

解析器使用状态机来处理 YAML 事件：

```c
enum parser_state {
  STATE_START,           // 初始状态
  STATE_STREAM,          // YAML 流
  STATE_DOCUMENT,        // 文档
  STATE_SECTION,         // 顶层字段 (apiVersion, kind, metadata, spec)
  STATE_API_VERSION,
  STATE_KIND,
  STATE_METADATA,
  STATE_SPEC,
  STATE_SPEC_DEFAULT,     // default: {...}
  STATE_SPEC_PARAMETERS, // parameters: {...}
  STATE_SPEC_MODE,       // mode: {...}
  STATE_INGRESS,         // ingress: {...}
  STATE_EGRESS,          // egress: {...}
  STATE_RULES,           // rules: [...]
  STATE_RULE,            // 单条规则
  STATE_FROM,            // from: {...}
  STATE_TO,              // to: {...}
  STATE_L2,              // L2: {...}
  STATE_L3,              // L3: {...}
  STATE_L4,              // L4: {...}
  STATE_STOP,            // 完成
};
```

### 4.3 consume_event 状态转换

**文件**: `lwfw_parser.c:1149-1400+`

```c
static lwfw_parser_err_t consume_event(struct parser_state *s,
                                        yaml_event_t *event)
{
  switch (s->state) {
  case STATE_START:
    if (event->type == YAML_STREAM_START_EVENT) {
      s->state = STATE_STREAM;
    }
    break;

  case STATE_STREAM:
    if (event->type == YAML_DOCUMENT_START_EVENT) {
      s->state = STATE_DOCUMENT;
    } else if (event->type == YAML_STREAM_END_EVENT) {
      s->state = STATE_STOP;  // 完成
    }
    break;

  case STATE_DOCUMENT:
    if (event->type == YAML_MAPPING_START_EVENT) {
      s->state = STATE_SECTION;
    }
    break;

  case STATE_SECTION:
    if (event->type == YAML_SCALAR_EVENT) {
      value = (char *)event->data.scalar.value;
      if (strcmp(value, "apiversion") == 0) {
        s->state = STATE_API_VERSION;
      } else if (strcmp(value, "kind") == 0) {
        s->state = STATE_KIND;
      } else if (strcmp(value, "metadata") == 0) {
        s->state = STATE_METADATA;
      } else if (strcmp(value, "spec") == 0) {
        s->state = STATE_SPEC;
      }
    }
    break;

  case STATE_SPEC:
    if (event->type == YAML_SCALAR_EVENT) {
      if (strcmp(value, "default") == 0) {
        s->state = STATE_SPEC_DEFAULT;
      } else if (strcmp(value, "parameters") == 0) {
        s->state = STATE_SPEC_PARAMETERS;
      } else if (strcmp(value, "ingress") == 0) {
        s->state = STATE_INGRESS;
      } else if (strcmp(value, "egress") == 0) {
        s->state = STATE_EGRESS;
      }
    }
    break;

  // ... 更多状态处理
  }
}
```

---

## 5. 配置数据结构

### 5.1 lwfw_policy_config

```c
struct lwfw_policy_config {
  // 元数据
  char policy_name[MAX_POLICY_NAME_LEN];
  uint32_t version;
  uint32_t revision;

  // 默认动作
  lwfw_action_t def_in_action;
  lwfw_action_t def_out_action;

  // 状态
  lwfw_state_t ingress_state;
  lwfw_state_t egress_state;

  // 规则
  uint16_t in_rules_num;
  uint16_t out_rules_num;
  struct lwfw_rule_config *in_rules;
  struct lwfw_rule_config *out_rules;

  // 参数
  lwfw_policy_parameter_t parameters;

  // 过滤器模式
  uint32_t filter_mode;

  // 是否包含状态规则
  bool contains_stateful_rule;

  // 树搜索参数
  struct {
    uint16_t bucket_size;
    uint16_t node_num;
  } params;
};
```

### 5.2 lwfw_rule_config

```c
struct lwfw_rule_config {
  uint16_t index;           // 规则索引
  uint16_t priority;       // 优先级
  uint16_t state;          // 启用/禁用

  char rule_name[MAX_RULE_NAME_LEN];

  // 动作
  lwfw_action_t action;

  // 匹配标志
  uint32_t flags;

  // 接口名
  lwfw_netif_t interface;

  // L2 匹配
  uint8_t src_hwaddr[ETH_HWADDR_LEN];
  uint8_t src_hwmask[ETH_HWADDR_LEN];
  uint8_t dst_hwaddr[ETH_HWADDR_LEN];
  uint8_t dst_hwmask[ETH_HWADDR_LEN];
  uint16_t ether_type;
  uint16_t vlan;

  // L3 匹配
  uint8_t proto;
  uint32_t src_ip;
  uint32_t src_ip_mask;
  uint32_t dst_ip;
  uint32_t dst_ip_mask;

  // L4 匹配
  uint16_t src_port_list[LWFW_MAX_PORT_COUNT];
  uint16_t dst_port_list[LWFW_MAX_PORT_COUNT];

  // 速率限制
  rate_limit_config_t rlimit;

#ifdef NIO_LWIP_LWCT
  // 连接状态
  uint16_t ct_state;
#endif
};
```

---

## 6. 默认参数

### 6.1 lwfw_policy_set_default_parameters

```c
int lwfw_policy_set_default_parameters(struct lwfw_policy_config *fw_cfg)
{
  lwfw_policy_parameter_t *p = &fw_cfg->parameters;

  // LWFW 参数
  p->lwfw_event_queue_size = LWFW_EVENT_NUM;              // 事件队列大小
  p->lwfw_event_notify_interval = EVENT_NOTIFY_INTERVAL;   // 通知间隔 (ms)
  p->lwfw_event_notify_threshold = LWFW_EVENT_ALMOST_FULL_THRESHOLD;  // 阈值
  p->lwfw_log_pdu_num = LWFW_LOG_PDU_NUM;                 // 日志 PDU 数量
  p->lwfw_log_pdu_len = LWFW_LOG_PDU_LEN;                 // 日志 PDU 长度
  p->lwfw_l2_filter_en = LWFW_L2_FILTER_OFF;              // L2 过滤开关
  p->lwfw_logs_per_second = LWFW_DEF_LOGS_PER_SECOND;      // 日志限速
  p->lwfw_log_level = LWFW_DEF_LOG_LEVEL;                  // 日志级别
  p->lwfw_event_file_count = CONFIG_NUM_EVENT_FILES;       // 事件文件数
  p->lwfw_max_event_file_size = CONFIG_MAX_EVENT_FILE_SIZE; // 文件大小限制
  p->lwfw_event_file_rotate_time = LWFW_FILE_ROTATE_TIME;  // 轮换时间
  p->lwfw_ct_oot_action = LWFW_CT_OOT_ACTION_CONT;         // CT 超时动作
  p->lwfw_pkt_rlimit_interval = LWFW_PKT_RLIMIT_INTERVAL_DEF;  // 限速间隔
  p->lwfw_event_rlimit_rate = LWFW_EVENT_RLIMIT_RATE_DEF;   // 事件限速

  // LWCT 参数
#ifdef NIO_LWIP_LWCT
  p->lwct_gc_scan_interval = LWCT_GC_INTERVAL;
  p->lwct_gc_entry_per_scan = LWCT_GC_SCAN_MAX;
  p->lwct_conn_confirm_tmo = LWCT_CONFIRM_CONN_TIMEOUT_SECS;
  p->lwct_bucket_count = LWCT_BUCKET_COUNT;
  p->lwct_lock_count = LWCT_LOCK_COUNT;
  p->lwct_conn_count = LWCT_MAX_CONN_COUNT;
  p->lwct_tcp_unreplied_tmo = LWCT_TCP_UNREPLIED_TMO;
  p->lwct_tcp_replied_tmo = LWCT_TCP_REPLIED_TMO;
  p->lwct_tcp_established_tmo = LWCT_TCP_ESTABLISHED_TMO;
  p->lwct_udp_unreplied_tmo = LWCT_UDP_UNREPLIED_TMO;
  p->lwct_udp_replied_tmo = LWCT_UDP_REPLIED_TMO;
  p->lwct_udp_established_tmo = LWCT_UDP_ESTABLISHED_TMO;
  p->lwct_icmp_unreplied_tmo = LWCT_ICMP_UNREPLIED_TMO;
  p->lwct_icmp_replied_tmo = LWCT_ICMP_REPLIED_TMO;
#endif

  return LWFW_PARSER_SUCCESS;
}
```

---

## 7. 策略初始化

### 7.1 lwfw_init 中的调用流程

```c
void lwfw_init()
{
  // 1. 解析配置文件
  lwfw_cfg = lwfw_manifest_parse();  // 返回 lwfw_policy_config*

  // 2. 初始化策略
  lwfw_init_policy(lwfw_p, lwfw_p->policy, lwfw_cfg);

  // 3. 初始化连接追踪 (如果启用)
#ifdef NIO_LWIP_LWCT
  lwct_init(&lwfw_cfg->parameters);
#endif

  // 4. 创建通知线程
  sys_thread_new(LWFW_NOTIFICATION_THREAD_NAME,
                 lwfw_notification_thread,
                 NULL,
                 LWFW_NOTIFICATION_THREAD_STACKSIZE,
                 LWFW_NOTIFICATION_THREAD_PRIO);
}
```

### 7.2 lwfw_init_policy

```c
static int lwfw_init_policy(lwfw_firewall_t *fw,
                             lwfw_policy_t *policy,
                             lwfw_policy_config_t *fw_cfg)
{
  // 设置版本和名称
  policy->version = fw_cfg->version;
  policy->revision = fw_cfg->revision;
  memcpy(policy->policy_name, fw_cfg->policy_name, MAX_POLICY_NAME_LEN);

  // 检查规则数量
  if ((fw_cfg->in_rules_num + fw_cfg->out_rules_num) >
      fw->ctrl.max_support_rule_num) {
    return LWFW_ERR_RULE_NUM_EXCEED;
  }

  // 清理旧规则
  lwfw_policy_clean(policy);

  // 初始化规则表
  lwfw_init_rule_tables(fw_cfg, policy);

  // 选择过滤引擎
  if (policy->params.filter_mode == LWFW_FILTER_TREE) {
    policy->filter_engine = &tree_search_eng;
  } else if (policy->params.filter_mode == LWFW_FILTER_HARDWARE) {
    // 硬件加速 (暂不支持)
    policy->filter_engine = &list_search_eng;
  } else {
    policy->filter_engine = &list_search_eng;  // 默认列表搜索
  }

  policy->filter_engine->init(policy, 0);
}
```

---

## 8. 规则表初始化

### 8.1 lwfw_init_rule_tables

```c
static int lwfw_init_rule_tables(lwfw_policy_config_t *fw_cfg,
                                  lwfw_policy_t *policy)
{
  for (int dir = 0; dir < LWFW_MAX_COUNT_TABLE; dir++) {
    rule_table = &policy->rule_tables[dir];

    // 设置默认动作
    if (dir == LWFW_IN_TABLE) {
      rule_table->def_action = fw_cfg->def_in_action;
      rule_table->state = fw_cfg->ingress_state;
      rule_cfg_list = fw_cfg->in_rules;
      rule_num = fw_cfg->in_rules_num;
    } else {
      rule_table->def_action = fw_cfg->def_out_action;
      rule_table->state = fw_cfg->egress_state;
      rule_cfg_list = fw_cfg->out_rules;
      rule_num = fw_cfg->out_rules_num;
    }

    // 遍历配置中的规则，按优先级插入链表
    for (idx = 0; idx < rule_num; idx++) {
      // 分配新规则
      new_rule = memp_malloc(policy->memp_type);

      // 填充规则数据
      new_rule->index = rule_cfg_list[idx].index;
      new_rule->priority = rule_cfg_list[idx].priority;
      new_rule->state = rule_cfg_list[idx].state;
      new_rule->action = rule_cfg_list[idx].action;

      // 构造 L2/L3/L4 信息
      lwfw_rule_l2_info_constructor(&rule_cfg_list[idx], new_rule);
      lwfw_rule_l3_info_constructor(&rule_cfg_list[idx], new_rule);
      lwfw_rule_l4_info_constructor(&rule_cfg_list[idx], new_rule);

      // 按优先级插入链表
      cdlist_insert_before(header, &cur_rule->next, &new_rule->next);
    }
  }
}
```

---

## 9. 运行时重载

### 9.1 lwfw_config_reload_manifest

支持在不重启的情况下重新加载配置：

```c
int lwfw_config_reload_manifest(char *path)
{
  // 1. 检查文件是否存在
  if (stat(path, &buffer) != 0) {
    return ENOENT;
  }

  // 2. 解析新配置
  fw_cfg = lwfw_config_init(NULL, path);

  // 3. 加锁保护
  sync_mutex_lock(&lwfw_p->policy_lock);

  // 4. 初始化到备用策略
  err = lwfw_init_policy(lwfw_p, lwfw_p->inactive_policy, fw_cfg);

  // 5. 交换策略
  temp = lwfw_p->policy;
  lwfw_p->policy = lwfw_p->inactive_policy;
  lwfw_p->inactive_policy = temp;

  sync_mutex_unlock(&lwfw_p->policy_lock);

  // 6. 释放旧配置
  lwfw_manifest_deinit(fw_cfg);
}
```

---

## 10. 配置验证

### 10.1 verify_lwfw_policy_config

```c
static lwfw_parser_err_t verify_lwfw_policy_config(
    struct lwfw_policy_config *fw_cfg)
{
  if (fw_cfg->version == 0 && fw_cfg->revision == 0) {
    LWFW_PARSER_ERRF("Invalid policy version");
    return LWFW_PARSER_INVALID_VALUE;
  }

  if ((fw_cfg->in_rules_num + fw_cfg->out_rules_num) > MAX_RULE_COUNT) {
    LWFW_PARSER_ERRF("Rule count exceeds maximum");
    return LWFW_PARSER_INVALID_VALUE;
  }

  return LWFW_PARSER_SUCCESS;
}
```

---

## 11. 总结

### 11.1 配置解析流程

```
YAML 文件
    │
    ▼
yaml_parser_initialize()
    │
    ▼
yaml_parser_set_input_file/string()
    │
    ▼
事件循环:
  yaml_parser_parse()
    │
    ▼
consume_event()  // 状态机处理
    │
    ├─► STATE_SECTION: 识别顶层字段
    ├─► STATE_SPEC: 解析 spec
    ├─► STATE_INGRESS/EGRESS: 解析规则
    └─► STATE_RULE: 解析单条规则
    │
    ▼
verify_lwfw_policy_config()  // 验证
    │
    ▼
lwfw_init_rule_tables()  // 构建内存链表
```

### 11.2 关键设计

1. **YAML 解析**: 使用 libyaml 事件驱动解析
2. **状态机**: 管理解析上下文，支持嵌套结构
3. **双策略**: active/inactive 策略支持原子切换
4. **内存池**: 规则使用 MEMP_LWFW_RULE 内存池分配
5. **优先级排序**: 规则按 priority 字段排序插入链表

### 11.3 与 Linux iptables 对比

| 特性 | LWFW | Linux iptables |
|------|------|----------------|
| **配置格式** | YAML | iptables-save/iptables-restore |
| **规则存储** | 链表 (内存) | 链表 (内核) |
| **运行时重载** | 支持 (原子切换) | 支持 (iptables-restore) |
| **验证** | 解析时验证 | 加载时验证 |
