# lwfw 防火墙整体架构深度分析

> 代码路径: `libs/util_libs/liblwfw/src/lwfw.c`
> 头文件: `libs/util_libs/liblwfw/include/`

---

## 1. 架构概览

### 1.1 系统分层

```
┌─────────────────────────────────────────────────────────────┐
│                    lwfw_agent (用户态)                       │
│  - 事件处理与日志写入                                         │
│  - JSON 格式化                                               │
│  - 配置文件热重载                                             │
└──────────────────────────┬──────────────────────────────────┘
                           │ seL4 IPC + 共享内存
┌──────────────────────────┴──────────────────────────────────┐
│                    lwfw (内核态 lwIP)                        │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │              ip4_filter 过滤入口                          │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  lwfw_pkt_info_constructor()  包信息提取                  │ │
│  │    ├─ L2 解析 (VLAN, MAC, EtherType)                    │ │
│  │    ├─ L3 解析 (IP, Protocol)                            │ │
│  │    └─ L4 解析 (Port)                                    │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  过滤器引擎抽象层                                         │ │
│  │    ├─ list_search_engine (规则<20)                       │ │
│  │    └─ tree_search_engine (规则≥20)                       │ │
│  ├─────────────────────────────────────────────────────────┤ │
│  │  lwct 连接追踪 (可选)                                     │ │
│  │    ├─ lwct_main_hook()                                  │ │
│  │    ├─ 连接哈希表                                          │ │
│  │    └─ GC 线程                                            │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 编译选项

| 选项 | 默认 | 说明 |
|------|------|------|
| `NIO_LWIP_LWFW` | 需启用 | 全局开关，启用防火墙 |
| `NIO_LWIP_LWCT` | 需启用 | 连接追踪支持 |
| `LWFW_ADVANCED_FUNC_L2` | 关闭 | L2 过滤 (VLAN/MAC) |
| `LWFW_TREE_SEARCH_EN` | 开启 | 树搜索引擎 |
| `LWFW_LATENCY_TEST` | 关闭 | 延迟测试 |
| `LWFW_PREFETCH` | 关闭 | 规则预取优化 |

### 1.3 核心数据结构

```
lwfw_firewall_t (全局控制结构)
  ├─ ctrl           : lwfw_ctrl_t (版本、状态、配置路径)
  ├─ policy         : lwfw_policy_t* (当前策略)
  ├─ inactive_policy: lwfw_policy_t* (热切换缓冲)
  └─ policy_lock    : sync_mutex_t

lwfw_policy_t (策略结构)
  ├─ rule_tables[2] : 规则表 (Ingress/Egress)
  │     ├─ state        : 启用/禁用
  │     ├─ def_action   : 默认动作
  │     ├─ header       : cdlist 链表头
  │     ├─ _ruleset     : 树搜索规则集
  │     └─ _hs_tree     : 决策树
  ├─ filter_engine   : const lwfw_backend_engine_t*
  └─ params          : 策略参数

lwfw_rule_t (单条规则)
  ├─ index          : 规则索引
  ├─ priority       : 优先级
  ├─ state          : 启用/禁用
  ├─ flags          : 匹配标志位图
  ├─ action         : 动作 (ALLOW/DENY/EVENT)
  ├─ interface      : 接口名
  ├─ l2             : L2 匹配 (MAC/VLAN/EtherType)
  ├─ l3             : L3 匹配 (IP/Prefix)
  ├─ l4             : L4 匹配 (Port/PortRange)
  ├─ ct_state       : 连接状态 (可选)
  └─ rlimit         : 限速参数
```

---

## 2. 过滤器引擎抽象

### 2.1 引擎接口

```c
// lwfw_common.h:204-211
typedef struct lwfw_backend_engine {
  char name[16];
  int (*init)(void *handle, void *data);
  int (*deinit)(void *handle, void *data);
  int (*do_filter)(void *handle, void *data, void* result);
  int (*dump)(void *handle, void *data);
} lwfw_backend_engine_t;
```

### 2.2 两套引擎

| 引擎 | 触发条件 | 数据结构 | 搜索复杂度 |
|------|---------|---------|-----------|
| `list_search_eng` | 规则数 < 20 | cdlist 链表 | O(n) |
| `tree_search_eng` | 规则数 ≥ 20 | hyperscan 树 | O(log n) |

### 2.3 引擎初始化

```c
// lwfw.c:lwfw_engine_init()
if (rule_cnt >= 20 && LWFW_TREE_SEARCH_EN) {
    policy->filter_engine = &tree_search_eng;
    filter_engine->init(policy, (void *)(uintptr_t)dir);
} else {
    policy->filter_engine = &list_search_eng;
    filter_engine->init(policy, 0);
}
```

---

## 3. 包过滤流程

### 3.1 数据流

```
数据包 (pbuf)
    │
    ▼
ip4_filter_dispatch_incoming() / ip4_filter_dispatch_outgoing()
    │ 检查规则表 state
    ▼
ip4_filter()
    │
    ├─ lwfw_pkt_info_constructor()  ← 解析包信息
    │     ├─ lwfw_pkt_l2_info_constructor()  [需LWFW_ADVANCED_FUNC_L2]
    │     ├─ lwfw_pkt_l3_info_constructor()
    │     └─ lwfw_pkt_l4_info_constructor()
    │
    ├─ lwct 未跟踪兜底检查 (NIO_LWIP_LWCT)
    │
    ├─ filter_engine->do_filter()  ← 引擎匹配
    │
    ├─ lwfw_generate_secure_event() ← 事件上报
    │
    └─ return action
```

### 3.2 规则匹配 check_rule()

```c
// 匹配顺序 (按字段优先级)
1. CT_STATE 匹配 (可选)
2. NETIF 接口匹配 (可选)
3. L2 信息匹配 (需 LWFW_ADVANCED_FUNC_L2)
   ├─ EtherType
   ├─ VLAN
   ├─ Src MAC + Mask
   └─ Dst MAC + Mask
4. L3 信息匹配
   ├─ Protocol
   ├─ Src IP + Mask
   └─ Dst IP + Mask
5. L4 信息匹配
   ├─ Src Port / PortRange
   └─ Dst Port / PortRange
```

### 3.3 动作编码

```c
// lwfw_common.h
LWFW_ACTION_CODE_ALLOW   = 0x00  // 允许
LWFW_ACTION_CODE_DENY    = 0x01  // 拒绝
LWFW_ACTION_CODE_EVENT   = 0x02  // 上报事件
LWFW_ACTION_LOGGING_MASK = 0x10  // 日志
```

---

## 4. 策略热切换

### 4.1 双缓冲机制

```
lwfw_firewall_t
  ├─ policy          ← 当前生效策略
  └─ inactive_policy ← 正在更新的策略

切换时:
  1. lwfw_copy_policy() 复制到 inactive_policy
  2. inactive_policy->filter_engine->init() 初始化新树
  3. 指针交换: policy <-> inactive_policy
```

### 4.2 热切换触发点

```c
// lwfw_config_reset_state() 行 1283-1298
if (lwfw_p->policy->params.filter_mode == LWFW_FILTER_TREE) {
    sync_mutex_lock(&lwfw_p->policy_lock);
    lwfw_policy_clean(lwfw_p->inactive_policy);
    ret = lwfw_copy_policy(lwfw_p->inactive_policy, lwfw_p->policy);
    // ... 初始化新树 ...
    lwfw_p->policy = lwfw_p->inactive_policy;
    lwfw_p->inactive_policy = tmp;
    sync_mutex_unlock(&lwfw_p->policy_lock);
}
```

---

## 5. lwct 连接追踪集成

### 5.1 集成架构

```
lwIP 数据包
    │
    ▼
┌─────────────────────────────────────┐
│  lwct_main_hook()                    │
│    └─ lwct_in() → 创建/查找连接       │
│         └─ pbuf->_lwct = conn | state │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  ip4_filter()                        │
│    └─ lwfw_pkt_info_constructor()    │
│         └─ pkt_info->ct_state =     │
│             p->_lwct & LWCT_STATE_MASK │
└─────────────────────────────────────┘
    │
    └─ check_rule() 中使用 ct_state 匹配
```

### 5.2 连接状态映射

| lwct 状态 | lwfw 状态 | 说明 |
|-----------|---------|------|
| LWCT_NEW | LWFW_CT_NEW | 新连接 |
| LWCT_ESTABLISHED | LWFW_CT_ESTABLISHED | 已建立 |
| LWCT_IS_REPLY | LWFW_CT_RELATED | 关联连接 |
| LWCT_ESTABLISHED_REPLY | LWFW_CT_ESTABLISHED | 已建立回复 |

---

## 6. 事件通知机制

### 6.1 事件上报流程

```
ip4_filter()
    │
    └─ lwfw_generate_secure_event()
          │
          ├─ 检查限速 (event_rlimit_rate)
          ├─ 检查 FIFO 满
          │
          └─ lwfw_event_push() → 共享内存 FIFO
                │
                ▼
          seL4 IPC 通知 lwfw_agent
                │
                ▼
          lwfw_agent 消费事件 → JSON 日志
```

### 6.2 FIFO 结构

```c
// lwfw_event_fifo_t (共享内存)
├─ params        : lwfw_agent_parameters_t
├─ data          : lwfw_agent_data_t (统计)
├─ get_idx       : 消费者索引
├─ put_idx       : 生产者索引
└─ events[]      : 事件数组 (512个)
```

---

## 7. 全局变量

```c
// lwfw.c:35-46
lwfw_log_level_t lwfw_log_level = LWFW_DEF_LOG_LEVEL;
lwfw_firewall_t g_lwfw_firewall, *lwfw_p;   // 全局防火墙句柄
lwfw_policy_t lwfw_policy = { .memp_type = MEMP_LWFW_RULE };
lwfw_policy_t lwfw_policy_swap = { .memp_type = MEMP_LWFW_RULE_SWAP };
uint32_t g_lwfw_curr_log_cnt = 0;
struct stats_filter g_lwfw_stats = {0};
```

---

## 8. 关键代码位置

| 文件 | 行号 | 描述 |
|------|------|------|
| `lwfw.c` | 724 | `ip4_filter()` 主入口 |
| `lwfw.c` | 329 | `lwfw_pkt_info_constructor()` 包解析 |
| `lwfw.c` | 565 | `check_rule()` 规则匹配 |
| `lwfw.c` | 1884 | `list_search_do_filter()` 线性扫描 |
| `lwfw.c` | 1815 | `lwfw_policies_setup()` 策略初始化 |
| `lwfw.c` | 1283 | 热切换逻辑 |
| `lwfw.c` | 1321 | `lwfw_dump_fw_info()` 信息导出 |
| `lwfw.c` | 1836 | `lwfw_rule_rlimit_worker()` 限速worker |

---

## 9. 编译配置

```cmake
# os-framework/settings/ 下根据平台启用
set(NIO_LWIP_LWFW ON)           # 防火墙全局开关
set(NIO_LWIP_LWCT ON)           # 连接追踪开关
set(LWFW_ADVANCED_FUNC_L2 OFF)  # L2 高级功能 (VLAN/MAC)
```

---

## 10. 架构设计亮点

1. **引擎抽象**: 通过 `lwfw_backend_engine_t` 统一接口，支持多引擎切换
2. **双缓冲热切换**: 规则更新时不影响当前策略
3. **连接追踪集成**: 通过 pbuf->_lwct 扩展字段传递连接状态
4. **共享内存 FIFO**: 内核与用户态高效事件传递

## 11. 架构设计局限

1. **全局锁竞争**: `policy_lock` 在热切换时持有，可能阻塞包处理
2. **lwct 单实例**: GC 线程退出无重启机制
3. **事件合并 O(n²)**: `lwfw_agent` 中事件合并算法可优化
