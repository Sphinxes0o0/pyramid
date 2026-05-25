# lwfw 规则解析器分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw_parser.c`
> 依赖: `libyaml`

---

## 1. 解析器架构

### 1.1 核心数据结构

```c
// 解析状态机
typedef struct parser_state {
  lwfw_policy_config_t *fw_cfg;      // 解析结果
  lwfw_rule_config_t *curr_rule;      // 当前解析的规则
  lwfw_rule_config_t *in_rules;       // ingress 规则数组
  lwfw_rule_config_t *out_rules;      // egress 规则数组
  int in_rule_idx;                     // ingress 规则索引
  int out_rule_idx;                    // egress 规则索引
  int state;                           // 解析状态机当前状态
  bool process_default_ingress;         // 是否解析 default ingress
  bool process_default_egress;          // 是否解析 default egress
  bool process_from;                    // 当前处理 src 还是 dst
  int src_port_index;                  // 源端口列表索引
  int dst_port_index;                   // 目标端口列表索引
} parser_state_t;
```

### 1.2 YAML 文件结构

```yaml
version: 1.0
revision: 1
name: "default_policy"
filter_mode: 1          # 0=DEFAULT(LIST), 1=TREE
default_ingress:         # Ingress 默认动作
  action: deny           # deny / allow
  event: false
default_egress:          # Egress 默认动作
  action: allow
  event: false
parameters:              # 系统参数
  lwfw_event_queue_size: 512
  lwfw_logs_per_second: 100
  ...
ingress_rules:           # Ingress 规则列表
  - index: 1
    priority: 1
    action: allow
    from:
      interface: "eth0"
      ip:
        source: "192.168.1.0/24"
      l4:
        protocol: tcp
        port_list: [80, 443]
    to:
      ip:
        destination: "10.0.0.1"
egress_rules:
  - ...
```

---

## 2. 解析状态机

### 2.1 顶层状态枚举

```c
enum parser_state {
  PARSER_STATE_START,
  PARSER_STATE_VERSION,
  PARSER_STATE_REVISION,
  PARSER_STATE_NAME,
  PARSER_STATE_FILTER_MODE,
  PARSER_STATE_DEFAULT_INGRESS,    // 进入 default ingress 块
  PARSER_STATE_DEFAULT_EGRESS,     // 进入 default egress 块
  PARSER_STATE_PARAMETERS,         // 进入 parameters 块
  PARSER_STATE_INGRESS_RULES,     // 进入 ingress_rules 块
  PARSER_STATE_EGRESS_RULES,       // 进入 egress_rules 块
  // 规则内部状态...
};
```

### 2.2 解析流程

```c
lwfw_config_init(filename)
  ├─ yaml_parser_init(&parser)
  ├─ parse_document()              ← 主解析循环
  │     └─ parse_content()
  │           ├─ "version:"         → 读版本号
  │           ├─ "revision:"        → 读修订号
  │           ├─ "filter_mode:"    → 读过滤模式
  │           ├─ "default_ingress:" → PARSER_STATE_DEFAULT_INGRESS
  │           ├─ "default_egress:" → PARSER_STATE_DEFAULT_EGRESS
  │           ├─ "parameters:"    → PARSER_STATE_PARAMETERS
  │           ├─ "ingress_rules:"   → PARSER_STATE_INGRESS_RULES
  │           └─ "egress_rules:"   → PARSER_STATE_EGRESS_RULES
  └─ yaml_parser_delete(&parser)
```

---

## 3. 关键解析函数

### 3.1 default ingress/egress 解析

```c
static lwfw_parser_err_t parse_spec_default_xgress(s, data)
{
  // data 是小写化后的值
  switch (spec_default_xgress_state) {
    case INIT:
      if (strcmp(data, "action") == 0)     → 下一状态 ACTION
      if (strcmp(data, "event") == 0)      → 下一状态 EVENT
      if (strcmp(data, "log") == 0)        → 下一状态 LOG
    case ACTION:
      if (strcmp(data, "deny") == 0)    → def_action |= DENY
      if (strcmp(data, "allow") == 0)   → def_action &= ~DENY
    case EVENT:
      if (strcmp(data, "true") == 0)   → def_action |= EVENT
      if (strcmp(data, "false") == 0)  → def_action &= ~EVENT
    case LOG:
      if (strcmp(data, "true") == 0)  → def_action |= LOGGING
      if (strcmp(data, "false") == 0) → def_action &= ~LOGGING
  }
}
```

**注意**: `action` 默认为 `deny`，其他字段默认为 `false`。

### 3.2 L3 IP 解析

```c
static lwfw_parser_err_t parse_spec_xgress_rule_l3_ipblock(s, data)
{
  switch (ipblock_state) {
    case INIT:
      if (strcmp(data, "ip") == 0)        → 读 IP 地址
      if (strcmp(data, "mask") == 0)     → 读点分十进制掩码
      if (strcmp(data, "prefix") == 0)    → 读 CIDR 前缀
    case IP:
      sscanf("192.168.1.1", "%d.%d.%d.%d", &ip[0]...)
      → addr = ip[0]<<24 | ip[1]<<16 | ip[2]<<8 | ip[3]
    case MASK:
      sscanf("255.255.255.0", "%d.%d.%d.%d", ...)
      → mask = 同上
    case PREFIX:
      sscanf("24", "%d", &prefix)
      → mask = (uint32_t)prefix  // 保存前缀值，运行时计算
  }
}
```

### 3.3 L4 端口解析

```c
// 支持两种格式: 范围 和 列表
static lwfw_parser_err_t parse_spec_xgress_rule_l4_range(s, data)
{
  // 范围格式
  switch (range_state) {
    case PORTBEGIN: port_begin = atoi(data); break;
    case PORTEND:   port_end   = atoi(data); break;
  }

static lwfw_parser_err_t parse_spec_xgress_rule_l4_list(s, data)
{
  // 列表格式: 最多 LWFW_MAX_PORT_COUNT=4 个端口
  port_list[port_index++] = atoi(data);
}
```

**问题**: 范围格式和列表格式是互斥的，通过不同的状态机处理。列表不支持 `portbegin/portend`，只能逐个端口指定。

---

## 4. 参数验证

### 4.1 范围检查

```c
// IP 地址
if (octet > 255) return INVALID;

// 端口
if (port < 0 || port > 65535) return INVALID;

// 时间参数
if (parsed_value < 1 || parsed_value > 1000000) return INVALID;

// 日志级别
if (parsed_value > LWFW_LOG_OFF) return INVALID;
```

---

## 5. 已知问题与优化建议

### 5.1 线程安全问题

**问题**: 解析器使用大量 `static` 局部变量保存解析状态:

```c
// lwfw_parser.c:22
static enum spec_default_xgress_state spec_default_xgress_state = SPEC_DEFAULT_XGRESS_STATE_INIT;
static enum spec_xgress_rule_fromto_l3_ipblock_state ...;
```

**影响**: 多线程并发调用 `lwfw_config_init()` 时状态会相互覆盖。

**建议**: 状态应保存在 `parser_state` 结构体中，而非 static。

### 5.2 sscanf 安全问题

```c
// lwfw_parser.c:827
if (sscanf(data, "%d.%d.%d.%d", &ip[0], &ip[1], &ip[2], &ip[3]) != 4)
```

**问题**: `sscanf` 不校验每个 octet 范围，可能导致越界写入。

**建议**: 改用 `inet_pton()` 或显式范围校验。

### 5.3 IP 前缀解析问题

```c
// lwfw_parser.c:882
*ipmask = (uint32_t)prefix;  // 保存的是前缀数字，不是掩码
```

调用侧需要根据 `LWFW_RULE_FLAGS_SRC_IP_MASK_LEN` 标志判断是用前缀还是直接用掩码，增加了运行时判断开销。

### 5.4 深拷贝无错误恢复

```c
static int lwfw_copy_policy(dst_policy, src_policy) {
  // 如果中途失败，dst_policy 已部分修改，无法回滚
  memcpy(dst_policy, src_policy, ...);
  for (i = 0; i < ...; i++) {
    err = lwfw_copy_rule_table(...);
    if (err != OK) {
      // 此时 dst_policy 已损坏
      lwfw_policy_clean(dst_policy);  // 只能清理，无法恢复
    }
  }
}
```

---

## 6. 配置文件加载

### 6.1 双配置源

```c
lwfw_policy_config_t* lwfw_manifest_parse()
{
#ifndef LWFW_UNIT_TEST
  // 优先级1: 指定路径
  // 优先级2: 编译期默认路径 (CPIO_LWFW_CONF_FILE)
  fw_cfg = lwfw_config_init(NULL, LWFW_CONF_FILE);
#else
  fw_cfg = TEST_lwfw_manifest_parse();  // 单元测试用
#endif
}
```

### 6.2 热重载

```c
lwfw_config_reload_manifest(path)
  ├─ stat(path)           → 检查文件存在
  ├─ lwfw_config_init()   → 解析新配置
  ├─ lwfw_init_policy(inactive_policy, new_cfg)  → 初始化备份策略
  ├─ atomic swap(policy, inactive_policy)       → 原子切换
  └─ lwfw_manifest_deinit(cfg)                   → 释放旧配置
```
