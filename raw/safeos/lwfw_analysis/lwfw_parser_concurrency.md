# YAML 解析器状态机与并发安全性分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw_parser.c`
> 头文件: `libs/util_libs/liblwfw/include/lwfw_parser.h`

---

## 1. 解析器架构概述

### 1.1 YAML 解析流程

```
lwfw_manifest_parse(filename)
  └─ read_lwfw_rules(filename, fw_cfg, rules, from_cpio)
        ├─ yaml_parser_init(&parser)
        ├─ 循环调用 yaml_parser_parse()
        │     └─ consume_event(&state, &event)  // 状态机处理每个事件
        └─ yaml_parser_delete(&parser)
```

### 1.2 状态机概览

```
STATE_START
  ├─ STATE_DOCUMENT_START
  │     └─ STATE_FW_CFG (解析策略配置)
  │           ├─ parse_spec_parameters()      → STATE_PARAMETERS
  │           ├─ parse_metadata()             → STATE_METADATA
  │           └─ STATE_FW_RULES (解析规则)
  │                 ├─ parse_spec_xgress_rule()        → 解析单条规则
  │                 │     ├─ parse_spec_xgress_rule_l2()      → L2 字段
  │                 │     ├─ parse_spec_xgress_rule_l3_ipblock() → L3 IP
  │                 │     ├─ parse_spec_xgress_rule_l4_range() → L4 端口范围
  │                 │     ├─ parse_spec_xgress_rule_l4_list()  → L4 端口列表
  │                 │     └─ parse_spec_xgress_rule_rate_limit() → 限速
  │                 └─ ...
```

---

## 2. 状态机状态定义

### 2.1 主状态枚举

```c
// lwfw_parser.h:50
enum state {
    STATE_START,
    STATE_DOCUMENT_START,
    STATE_FW_CFG,
    STATE_PARAMETERS,
    STATE_METADATA,
    STATE_FW_RULES,
    STATE_SPEC_XGRESS_INGRESS_RULES,  // 入口规则
    STATE_SPEC_XGRESS_EGRESS_RULES,    // 出口规则
    STATE_SPEC_XGRESS_INGRESS_DEFAULT_RULES,
    STATE_SPEC_XGRESS_EGRESS_DEFAULT_RULES,
    STATE_SPEC_XGRESS_RULE,
    STATE_SPEC_XGRESS_RULE_FROMTO,
    STATE_SPEC_XGRESS_RULE_FROMTO_L2,
    STATE_SPEC_XGRESS_RULE_FROMTO_L3,
    STATE_SPEC_XGRESS_RULE_FROMTO_L4,
    STATE_SPEC_XGRESS_RULE_FROMTO_L4_PORT,
    STATE_SPEC_XGRESS_RULE_FROMTO_L4_PORT_RANGE,
    STATE_SPEC_XGRESS_RULE_FROMTO_L4_PORT_LIST,
    STATE_SPEC_XGRESS_RULE_FROMTO_RATE_LIMIT,
    STATE_ERROR
};
```

### 2.2 子状态枚举

```c
// 入口/出口规则状态
enum spec_xgress_state {
    SPEC_XGRESS_STATE_INIT,
    SPEC_XGRESS_STATE_RULES,        // 规则列表开始
    SPEC_XGRESS_STATE_RULE_ENTRY,   // 单条规则
    SPEC_XGRESS_STATE_DEFAULT_ENTRY, // 默认规则
};

// L3 IP 块解析状态
enum spec_xgress_rule_fromto_l3_ipblock_state {
    SPEC_XGRESS_RULE_FROMTO_L3_IPBLOCK_STATE_INIT,
    SPEC_XGRESS_RULE_FROMTO_L3_IPBLOCK_STATE_SRCIPBLOCK,
    SPEC_XGRESS_RULE_FROMTO_L3_IPBLOCK_STATE_SRCIPMASK,
    SPEC_XGRESS_RULE_FROMTO_L3_IPBLOCK_STATE_DSTIPBLOCK,
    SPEC_XGRESS_RULE_FROMTO_L3_IPBLOCK_STATE_DSTIPMASK,
};

// L4 端口范围解析状态
enum spec_xgress_rule_fromto_l4_range_state {
    SPEC_XGRESS_RULE_FROMTO_L4_RANGE_STATE_INIT,
    SPEC_XGRESS_RULE_FROMTO_L4_RANGE_STATE_PORTBEGIN,
    SPEC_XGRESS_RULE_FROMTO_L4_RANGE_STATE_PORTEND,
};

// 限速解析状态
enum spec_xgress_rule_rate_limit_state {
    SPEC_XGRESS_RULE_RATE_LIMIT_STATE_INIT,
    SPEC_XGRESS_RULE_RATE_LIMIT_STATE_NAME,
    SPEC_XGRESS_RULE_RATE_LIMIT_STATE_BURST,
    SPEC_XGRESS_RULE_RATE_LIMIT_STATE_RATE,
    SPEC_XGRESS_RULE_RATE_LIMIT_STATE_EXPIRE,
    SPEC_XGRESS_RULE_RATE_LIMIT_STATE_EVENT_MODE,
};
```

---

## 3. 严重问题：静态状态变量

### 3.1 问题代码

**parse_spec_xgress_rule_l3_ipblock** (lwfw_parser.c:771):

```c
static lwfw_parser_err_t parse_spec_xgress_rule_l3_ipblock(struct parser_state *s, char *data) {
    uint32_t *ipaddr = NULL;
    uint32_t *ipmask = NULL;
    uint32_t *flags = NULL;
    static enum spec_xgress_rule_fromto_l3_ipblock_state
        spec_xgress_rule_fromto_l3_ipblock_state = SPEC_XGRESS_RULE_FROMTO_L3_IPBLOCK_STATE_INIT;
    // ...
}
```

**parse_spec_xgress_rule_l4_range** (lwfw_parser.c:895):

```c
static lwfw_parser_err_t parse_spec_xgress_rule_l4_range(struct parser_state *s, char *data) {
    static enum spec_xgress_rule_fromto_l4_range_state
        spec_xgress_rule_fromto_l4_range_state = SPEC_XGRESS_RULE_FROMTO_L4_RANGE_STATE_INIT;
    // ...
}
```

**consume_event** (lwfw_parser.c:1149):

```c
static lwfw_parser_err_t consume_event(struct parser_state *s, yaml_event_t *event) {
    static enum spec_xgress_rule_state spec_xgress_rule_state = SPEC_XGRESS_RULE_STATE_INIT;
    static enum spec_xgress_rule_fromto_state spec_xgress_rule_fromto_state = SPEC_XGRESS_RULE_FROMTO_STATE_INIT;
    static enum spec_xgress_rule_fromto_l3_state spec_xgress_rule_fromto_l3_state = SPEC_XGRESS_RULE_FROMTO_L3_STATE_INIT;
    static enum spec_xgress_rule_fromto_l4_state spec_xgress_rule_fromto_l4_state = SPEC_XGRESS_RULE_FROMTO_L4_STATE_INIT;
    // ...
}
```

### 3.2 全部静态状态变量列表

| 函数 | 静态变量 | 类型 |
|------|----------|------|
| `parse_spec_default_xgress` | `spec_default_xgress_state` | enum |
| `parse_spec_parameters` | `spec_default_parameters_state` | enum |
| `parse_spec_xgress_rule_l2` | `spec_xgress_rule_fromto_l2_state` | enum |
| `parse_spec_xgress_rule_l3_ipblock` | `spec_xgress_rule_fromto_l3_ipblock_state` | enum |
| `parse_spec_xgress_rule_l4_range` | `spec_xgress_rule_fromto_l4_range_state` | enum |
| `parse_spec_xgress_rule_rate_limit` | `spec_xgress_rule_rate_limit_state` | enum |
| `parse_metadata` | `metadata_state` | enum |
| `consume_event` | `spec_xgress_rule_state` | enum |
| `consume_event` | `spec_xgress_rule_fromto_state` | enum |
| `consume_event` | `spec_xgress_rule_fromto_l3_state` | enum |
| `consume_event` | `spec_xgress_rule_fromto_l4_state` | enum |

**总计**: 11 个静态状态变量

### 3.3 问题分析

| 项目 | 内容 |
|------|------|
| 问题 | 状态机使用 `static` 变量存储状态，不是线程安全的 |
| 影响 | 并发调用 `lwfw_config_init()` 或 `lwfw_manifest_parse()` 时状态会互相覆盖 |
| 触发条件 | 多个线程同时调用解析函数 |
| 当前调用链 | `lwfw_init()` → `lwfw_manifest_parse()` (单线程，暂无问题) |
| 风险 | 如果未来 lwfw_agent 启动时触发重载，或多进程同时初始化，可能出问题 |

---

## 4. 并发安全性分析

### 4.1 当前调用场景

```c
// lwfw_init() - 单线程调用
lwfw_init()
  └─ lwfw_manifest_parse(cfg_path)  // 单线程，暂无问题
        └─ read_lwfw_rules(filename, fw_cfg, rules, from_cpio)
              └─ consume_event()  // 静态状态机
```

```c
// lwfw_notif.c - 通知线程可能触发重载
lwfw_notification_timer_thread()
  └─ if (cfg_in_reloading)
        └─ lwfw_config_reload_manifest(cfg_path)  // 可能在另一线程
```

### 4.2 风险场景

**场景 1**: 热重载触发时通知线程与数据包处理线程并发

```c
// 线程 A: 通知线程
timer_thread() {
    if (cfg_in_reloading) {
        lwfw_config_reload_manifest(cfg_path);  // 调用解析器
    }
}

// 线程 B: 数据包处理线程
ip4_input() {
    // 正常数据包处理
}
```

**场景 2**: 未来多实例部署

```c
// 如果未来有多个 lwfw 实例，每个实例调用 lwfw_config_init()
// 会互相干扰对方的静态状态机
```

### 4.3 问题严重性评估

| 因素 | 评估 |
|------|------|
| 当前是否触发 | **否** - 初始化是单线程的 |
| 未来风险 | **中** - 如果增加并发重载或多实例支持会触发 |
| 数据损坏风险 | **高** - 状态机冲突可能导致规则解析错误 |
| 建议 | **高优先级修复** - 将状态移到 `parser_state` 结构体 |

---

## 5. 修复建议

### 5.1 方案：将静态状态移到 parser_state

**当前 `struct parser_state`** (lwfw_parser.h:243):

```c
struct parser_state {
    enum state state;                    // 主状态 ✓
    struct lwfw_policy_config *fw_cfg;
    struct lwfw_rule_config *rules;
    struct lwfw_rule_config *curr_rule;
    uint32_t rule_count;
    uint32_t src_port_index;
    uint32_t dst_port_index;
    bool process_egress;
    bool process_ingress;
    bool process_default_egress;
    bool process_default_ingress;
    // 缺少子状态字段！
};
```

**建议扩展**:

```c
struct parser_state {
    // 现有字段
    enum state state;
    struct lwfw_policy_config *fw_cfg;
    // ... 现有字段 ...

    // 新增子状态字段
    enum spec_xgress_state spec_xgress_state;
    enum spec_xgress_rule_fromto_l3_ipblock_state l3_ipblock_state;
    enum spec_xgress_rule_fromto_l4_range_state l4_range_state;
    enum spec_xgress_rule_rate_limit_state rate_limit_state;
    enum metadata_state metadata_state;
    // ... 其他子状态 ...
};
```

### 5.2 修复示例

**Before**:

```c
static lwfw_parser_err_t parse_spec_xgress_rule_l3_ipblock(struct parser_state *s, char *data) {
    static enum spec_xgress_rule_fromto_l3_ipblock_state state = ...;
    // 使用 state
}
```

**After**:

```c
static lwfw_parser_err_t parse_spec_xgress_rule_l3_ipblock(struct parser_state *s, char *data) {
    // 使用 s->l3_ipblock_state
    switch (s->l3_ipblock_state) {
        case SPEC_XGRESS_RULE_FROMTO_L3_IPBLOCK_STATE_INIT:
            // ...
            s->l3_ipblock_state = SPEC_XGRESS_RULE_FROMTO_L3_IPBLOCK_STATE_SRCIPBLOCK;
            break;
        // ...
    }
}
```

### 5.3 修复优先级

| 优先级 | 变量 | 影响 |
|--------|------|------|
| **P0** | `spec_xgress_rule_state` (consume_event) | 影响所有规则解析 |
| **P0** | `spec_xgress_rule_fromto_state` | 影响规则 from/to 解析 |
| **P1** | `spec_xgress_rule_fromto_l3_state` | 影响 L3 解析 |
| **P1** | `spec_xgress_rule_fromto_l4_state` | 影响 L4 解析 |
| **P2** | 其他子状态 | 影响局部解析 |

---

## 6. sscanf 格式串注入风险

### 6.1 问题代码 (lwfw_parser.c:827)

```c
static lwfw_parser_err_t parse_spec_xgress_rule_l3_ipblock(struct parser_state *s, char *data) {
    // ...
    if (strcmp(data, "srcIpBlock") == 0) {
        s->curr_rule->flags |= LWFW_RULE_FLAGS_SRC_IP_MASK;
        s->curr_rule->l3_info.src_ip_block = 1;
        s->curr_rule->l3_info.src_ip = ipaddr;
    } else if (strcmp(data, "srcIpPrefix") == 0) {
        s->curr_rule->flags |= LWFW_RULE_FLAGS_SRC_IP_MASK_LEN;
        s->curr_rule->l3_info.src_ip_block = 0;
        s->curr_rule->l3_info.src_prefix = prefix;
    }
    // ...
    } else {
        // 解析 IP 地址
        int result = sscanf(data, "%d.%d.%d.%d", &ip[0], &ip[1], &ip[2], &ip[3]);
        if (result != 4) {
            return LWFW_PARSER_ERR_INVALID_IP;
        }
        // 没有验证每个 octet 的范围 (0-255)！
    }
}
```

### 6.2 风险

- `sscanf` 不校验 octet 范围，负数或 >255 的值可能通过
- 可能的整数溢出或数组越界

### 6.3 建议修复

```c
// 方案 1: 使用 inet_pton
struct in_addr addr;
if (inet_pton(AF_INET, data, &addr) != 1) {
    return LWFW_PARSER_ERR_INVALID_IP;
}
ipaddr = ntohl(addr.s_addr);

// 方案 2: 显式校验
int result = sscanf(data, "%d.%d.%d.%d", &ip[0], &ip[1], &ip[2], &ip[3]);
if (result != 4) return LWFW_PARSER_ERR_INVALID_IP;
for (int i = 0; i < 4; i++) {
    if (ip[i] < 0 || ip[i] > 255) {
        return LWFW_PARSER_ERR_INVALID_IP;
    }
}
```

---

## 7. 优化建议汇总

| 优先级 | 问题 | 建议 | 复杂度 |
|--------|------|------|--------|
| **P1** | 静态状态变量 (11个) | 将状态移到 `parser_state` 结构体 | 高 |
| **P2** | sscanf 不校验 IP octet | 使用 `inet_pton` 或显式校验 | 低 |
| **P3** | 魔法数字 | `MAX_RULE_NAME_LEN` 等应可配置 | 低 |
| **P3** | 错误处理分散 | 重构为统一错误传播 | 中 |

---

## 8. 关键代码位置

| 文件 | 行号 | 描述 |
|------|------|------|
| `lwfw_parser.c` | 22 | `spec_default_xgress_state` 静态变量 |
| `lwfw_parser.c` | 108 | `spec_default_parameters_state` 静态变量 |
| `lwfw_parser.c` | 696 | `spec_xgress_rule_fromto_l2_state` 静态变量 |
| `lwfw_parser.c` | 775 | `spec_xgress_rule_fromto_l3_ipblock_state` 静态变量 |
| `lwfw_parser.c` | 899 | `spec_xgress_rule_fromto_l4_range_state` 静态变量 |
| `lwfw_parser.c` | 1012 | `spec_xgress_rule_rate_limit_state` 静态变量 |
| `lwfw_parser.c` | 1091 | `metadata_state` 静态变量 |
| `lwfw_parser.c` | 1153-1156 | consume_event 中的 4 个静态变量 |
| `lwfw_parser.h` | 243 | `struct parser_state` 定义 |
