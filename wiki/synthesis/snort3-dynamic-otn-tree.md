---
type: synthesis
tags: [snort3, otn, detection-engine, fast-pattern, rule-tree, data-structure]
created: 2026-05-25
sources: [snort3-claude-md]
---

# Snort3 Dynamic OTN Tree 技术深入分析

## 1. 概述

Snort3 的 Dynamic OTN Tree 是其检测引擎的核心数据结构，实现了运行时可变的规则树结构，相比 Snort2 的静态编译规则树具有显著的灵活性和性能优势。

### 核心概念

```
文本规则 → RTN (RuleTreeNode) + OTN (OptTreeNode) → PORT_RULE_MAP → RuleGroup → MPSE
```

- **RTN (RuleTreeNode)**: 规则头节点，存储地址/端口等匹配条件
- **OTN (OptTreeNode)**: 规则体节点，存储检测选项链表
- **PORT_RULE_MAP**: 三层索引结构，实现 O(1) 端口查找
- **RuleGroup**: 规则组，包含 MPSE (Multi-Pattern Search Engine) 和非 FP 规则
- **MPSE**: 多模式搜索引擎，执行 fast pattern 匹配

---

## 2. 核心数据结构定义

### 2.1 RuleTreeNode (规则头节点)

```cpp
// src/detection/treenodes.h:110
struct RuleTreeNode
{
    using Flag = uint8_t;
    static constexpr Flag ENABLED       = 0x01;  // 规则启用标志
    static constexpr Flag ANY_SRC_PORT  = 0x02;  // 任意源端口
    static constexpr Flag ANY_DST_PORT  = 0x04;  // 任意目的端口
    static constexpr Flag ANY_FLAGS     = 0x08;
    static constexpr Flag BIDIRECTIONAL = 0x10;   // 双向规则
    static constexpr Flag ANY_SRC_IP    = 0x20;   // 任意源IP
    static constexpr Flag ANY_DST_IP    = 0x40;   // 任意目的IP
    static constexpr Flag USER_MODE     = 0x80;

    // 匹配函数链表 (Bidirectional, Activates, etc.)
    RuleFpList* rule_func = nullptr;

    // 规则头信息 (action, proto, src_nets, src_ports, dir, dst_nets, dst_ports)
    RuleHeader* header = nullptr;

    // IP 地址变量
    sfip_var_t* sip = nullptr;
    sfip_var_t* dip = nullptr;

    // 端口对象
    PortObject* src_portobject = nullptr;
    PortObject* dst_portobject = nullptr;

    // 关联的输出函数列表
    struct ListHead* listhead = nullptr;

    SnortProtocolId snort_protocol_id = 0;

    // OTN 引用计数 (同一 policy 下多个 OTN 可引用同一 RTN)
    unsigned int otnRefCount = 0;

    snort::IpsAction::Type action = 0;
    uint8_t flags = 0;

    bool enabled() const { return (flags & ENABLED) != 0; }
    bool any_src_port() const { return (flags & ANY_SRC_PORT) != 0; }
    bool any_dst_port() const { return (flags & ANY_DST_PORT) != 0; }
    bool any_any_port() const { return any_src_port() and any_dst_port(); }
};
```

### 2.2 OptTreeNode (规则体节点)

```cpp
// src/detection/treenodes.h:167
struct OptTreeNode
{
    using Flag = uint8_t;
    static constexpr Flag WARNED_FP  = 0x01;  // 已警告 fast pattern 问题
    static constexpr Flag STATELESS  = 0x02;  // 无状态规则
    static constexpr Flag RULE_STATE = 0x04;   // 有规则状态
    static constexpr Flag META_MATCH = 0x08;   // 元数据已匹配
    static constexpr Flag TO_CLIENT  = 0x10;   // 方向: 到客户端
    static constexpr Flag TO_SERVER  = 0x20;  // 方向: 到服务器
    static constexpr Flag BIT_CHECK  = 0x40;   // 检查 flowbits
    static constexpr Flag SVC_ONLY   = 0x80;   // 仅限服务

    // === 元数据 ===
    SigInfo sigInfo;                          // gid, sid, rev, message, class, priority
    char* soid = nullptr;                     // SO rule ID

    // === 检测函数 ===
    OptFpList* opt_func = nullptr;            // 检测选项链表 (content, nocase, etc.)
    OutputSet* outputFuncs = nullptr;          // 输出函数
    snort::IpsOption* agent = nullptr;       // 代理选项 (for replace)
    const char** buffer_setters = nullptr;    // buffer 设置函数

    // Fast pattern 专用
    OptFpList* normal_fp_only = nullptr;      // 仅 normal MPSE 的 FP
    OptFpList* offload_fp_only = nullptr;     // 仅 offload MPSE 的 FP

    // 检测过滤器 (rate_filter)
    struct THD_NODE* detection_filter = nullptr;

    TagData* tag = nullptr;

    // === 多 policy 索引 ===
    // ptr to RTN 数组, indexed by policyId
    // 一个 OTN 可关联多个 policy 的 RTN
    RuleTreeNode** proto_nodes = nullptr;
    unsigned short proto_node_num = 0;

    // === 状态与统计 ===
    OtnState* state = nullptr;                // per-thread 性能统计

    // === 评估顺序 ===
    unsigned evalIndex = 0;                   // 在评估集中的位置
    unsigned ruleIndex = 0;                   // 唯一索引
    uint32_t num_detection_opts = 0;          // 检测选项数量

    SnortProtocolId snort_protocol_id = 0;

    // === Fast Pattern 相关 ===
    uint16_t longestPatternLen = 0;           // 最长模式长度 (用于排序)

    IpsPolicy::Enable enable = IpsPolicy::Enable::DISABLED;
    Flag flags = 0;

    // 方向信息
    enum SectionDir { SECT_TO_SRV = 0, SECT_TO_CLIENT, SECT_DIR__MAX };
    snort::section_flags sections[SECT_DIR__MAX];

    // 状态查询方法
    bool enabled_somewhere() const;            // 检查是否有任何 policy 启用
    bool checks_flowbits() const;              // 是否检查 flowbits
};
```

### 2.3 OptFpList (检测选项链表节点)

```cpp
// src/detection/treenodes.h:48
struct OptFpList
{
    snort::IpsOption* ips_opt;                // IPS 选项插件
    int (* OptTestFunc)(void* option_data, class Cursor&, snort::Packet*); // 检测函数
    OptFpList* next;                           // 链表下一节点
    unsigned char isRelative;                   // 是否为相对选项
    option_type_t type;                        // 选项类型 (CONTENT, PCRE, etc.)
};
```

### 2.4 PORT_RULE_MAP (三层索引结构)

```cpp
// src/detection/pcrm.h:37
struct PORT_RULE_MAP
{
    int prmNumDstRules;                        // 目的端口规则数
    int prmNumSrcRules;                        // 源端口规则数
    int prmNumGenericRules;                     // 任意端口规则数

    int prmNumDstGroups;                       // 目的端口组数
    int prmNumSrcGroups;                       // 源端口组数

    // 源端口索引表 (数组, 按端口号索引)
    RuleGroup* prmSrcPort[snort::MAX_PORTS];   // MAX_PORTS = 65536

    // 目的端口索引表 (数组, 按端口号索引)
    RuleGroup* prmDstPort[snort::MAX_PORTS];

    // 任意端口规则组
    RuleGroup* prmGeneric;
};
```

### 2.5 RuleGroup (规则组)

```cpp
// src/ports/port_group.h:67
struct RuleGroup
{
    // === 非 Fast Pattern 规则链表 ===
    RULE_NODE* nfp_head = nullptr;             // NFP 规则头
    RULE_NODE* nfp_tail = nullptr;             // NFP 规则尾

    // === Pattern Matcher 列表 ===
    // 按 PduSection 索引的 PatternMatcher 数组
    // PS_NONE, PS_PAYLOAD, PS_HEADER, PS_MAX
    using PmList = std::vector<PatternMatcher*>[snort::PS_MAX + 1];
    PmList pm_list;

    // 检测选项树 (用于 NFP 规则)
    void* nfp_tree = nullptr;

    unsigned rule_count = 0;                   // 规则总数
    unsigned nfp_rule_count = 0;               // NFP 规则数

    void add_rule();                           // 添加规则
    bool add_nfp_rule(void*);                  // 添加 NFP 规则
    void delete_nfp_rules();                   // 删除 NFP 规则

    PatternMatcher* get_pattern_matcher(       // 获取/创建 PatternMatcher
        PatternMatcher::Type, const char*, snort::PduSection sect);
};
```

### 2.6 PatternMatcher (模式匹配器)

```cpp
// src/ports/port_group.h:52
struct PatternMatcher
{
    enum Type { PMT_PKT, PMT_FILE, PMT_PDU };

    Type type;                                 // 类型
    const char* name;                          // buffer 名称 (pkt_data, http_uri, etc.)
    bool raw_data;                             // 是否原始数据

    snort::MpseGroup group;                    // MPSE 组 (normal + offload)
    snort::IpsOption* fp_opt = nullptr;        // Fast pattern 选项
};
```

### 2.7 MpseGroup (MPSE 组)

```cpp
// src/framework/mpse_batch.h:32
class MpseGroup
{
public:
    Mpse* get_normal_mpse() const { return normal_mpse; }
    Mpse* get_offload_mpse() const { return offload_mpse ? offload_mpse : normal_mpse; }

    bool create_normal_mpse(const SnortConfig*, const MpseAgent*);
    bool create_offload_mpse(const SnortConfig*, const MpseAgent*);

public:
    Mpse* normal_mpse;                         // 普通 MPSE
    Mpse* offload_mpse;                        // Offload MPSE (如 Hyperscan)
    bool normal_is_dup = false;
    bool offload_is_dup = false;
};
```

### 2.8 MpseBatch (批量搜索)

```cpp
// src/framework/mpse_batch.h:111
struct MpseBatch
{
    MpseMatch mf;                              // 匹配回调函数
    void* context;                             // 上下文 (IpsContext*)

    // 按 buffer key 索引的批量搜索项
    std::unordered_map<MpseBatchKey<>, MpseBatchItem, MpseBatchKeyHash> items;

    void search();                             // 执行搜索
    Mpse::MpseRespType receive_responses();    // 接收响应

    void offload_search();
    Mpse::MpseRespType receive_offload_responses();

    bool search_sync();                        // 同步搜索
    bool can_fallback() const;                 // 是否可回退到 normal
};
```

### 2.9 MpseStash (匹配结果暂存器)

```cpp
// src/detection/fp_detect.cc:729
class MpseStash
{
public:
    struct MatchData
    {
        void* user;    // PMX*
        void* tree;    // detection_option_tree_root_t*
        void* list;    // neg_list
        int index;
    };

    using MatchStore = std::vector<MatchData>;

public:
    // 构造函数 - 从 fast_pattern_config 初始化
    MpseStash(const FastPatternConfig& fp);

    // Offload 线程调用 - 暂存匹配结果
    bool push(void* user, void* tree, int index, void* context, void* list);

    // Packet 线程调用 - 处理所有暂存的匹配
    void process(IpsContext*);

private:
    void process(IpsContext*, MatchStore&);

private:
    static constexpr unsigned qmax = 128;     // 批量处理阈值
    unsigned inserts = 0;
    unsigned max;                              // 最大暂存数
    bool dedup;                                // 去重标志

    MatchStore queue;                          // 主队列
    MatchStore defer;                          // 延迟队列 (flowbit check)
};
```

### 2.10 IpsContext (检测上下文)

```cpp
// src/detection/ips_context.h:58
class IpsContext
{
public:
    // === 搜索相关 ===
    MpseBatch searches;                        // 批量搜索请求
    MpseStash* stash;                          // 匹配结果暂存器
    OtnxMatchData* otnx;                       // 匹配信息数组

    // === Packet 信息 ===
    Packet* packet;
    Packet* wire_packet = nullptr;
    Packet* encode_packet;
    DAQ_PktHdr_t* pkth;
    uint8_t* buf;

    // === 配置 ===
    const SnortConfig* conf = nullptr;

    // === 状态 ===
    uint64_t context_num;
    uint64_t packet_number = 0;
    ActiveRules active_rules;                   // NONE, NON_CONTENT, CONTENT
    State state;                               // IDLE, BUSY, SUSPENDED
};
```

---

## 3. PORT_RULE_MAP 三层索引结构

### 3.1 架构图

```
PORT_RULE_MAP
├── prmTcpRTNX ────────────────────────────── TCP 协议规则图
├── prmUdpRTNX ────────────────────────────── UDP 协议规则图
├── prmIpRTNX  ────────────────────────────── IP 协议规则图
└── prmIcmpRTNX ───────────────────────────── ICMP 协议规则图

每个 PORT_RULE_MAP 结构:
┌─────────────────────────────────────────────────────────┐
│ prmSrcPort[65536]  ─→ RuleGroup* (源端口索引)            │
│ prmDstPort[65536]  ─→ RuleGroup* (目的端口索引)          │
│ prmGeneric         ─→ RuleGroup* (任意端口)               │
└─────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
    RuleGroup            RuleGroup            RuleGroup
    ├── pm_list[]        ├── pm_list[]        ├── pm_list[]
    │   └── MPSE         │   └── MPSE         │   └── MPSE
    ├── nfp_tree         ├── nfp_tree         ├── nfp_tree
    └── rule_count       └── rule_count       └── rule_count
```

### 3.2 端口查找算法

```cpp
// src/detection/pcrm.cc:77
static int prmFindRuleGroup(
    PORT_RULE_MAP* p,
    int dport,
    int sport,
    RuleGroup** src,
    RuleGroup** dst,
    RuleGroup** gen)
{
    *src = *dst = *gen = nullptr;

    // O(1) 目的端口查找
    if (dport != ANYPORT and dport < MAX_PORTS)
        *dst = p->prmDstPort[dport];

    // O(1) 源端口查找
    if (sport != ANYPORT and sport < MAX_PORTS)
        *src = p->prmSrcPort[sport];

    // 如果没有特定端口规则，使用通用规则
    if (p->prmGeneric and p->prmGeneric->rule_count > 0)
    {
        if (split_any_any or (!*src and !*dst))
            *gen = p->prmGeneric;
    }

    return (*src or *dst or *gen) ? 1 : 0;
}
```

### 3.3 为什么是 O(1) 查找

传统的规则匹配需要遍历所有规则，时间复杂度 O(n)。PORT_RULE_MAP 通过端口数组索引实现 O(1) 查找：

1. **数组索引**: `prmDstPort[dport]` 直接访问，无需遍历
2. **空间换时间**: 使用 65536 × 8 = 512KB 空间（每个指针 8 字节）
3. **哈希表备用**: 相同端口的规则聚合成 RuleGroup，避免重复

---

## 4. "Dynamic" 体现在哪里

### 4.1 运行时规则插入/删除

Snort3 支持在不停止检测引擎的情况下动态加载/卸载规则：

```
旧配置 +→→→→→→→→→→┐
                        ├──→ 新配置 (原子替换)
新配置 +→→→→→→→→→→┘
```

**关键机制**:

1. **规则重载 (Reload)**:
   - `SnortConfig` 指针原子替换
   - 旧配置在所有 packet thread 完成后释放
   - 检测线程通过 `context->conf` 访问配置

2. **规则启用/禁用**:
   ```cpp
   // src/detection/treenodes.h:143
   void set_enabled() { flags |= ENABLED; }
   void clear_enabled() { flags &= (~ENABLED); }
   bool enabled() const { return (flags & ENABLED) != 0; }
   ```

### 4.2 OTN proto_nodes 多 Policy 索引

```cpp
// src/detection/treenodes.h:197
// ptr to list of RTNs (head part); indexed by policyId
RuleTreeNode** proto_nodes = nullptr;
```

**设计目的**: 一个 OTN 可对应多个 policy 的 RTN

```
OTN (同一条规则)
  ├── proto_nodes[policy_0] ─→ RTN_0 (Network Policy 0)
  ├── proto_nodes[policy_1] ─→ RTN_1 (Network Policy 1)
  └── proto_nodes[policy_2] ─→ RTN_2 (Network Policy 2)
```

**优势**:
- 规则解析一次，存储多份 RTN 指针
- 减少内存占用
- 便于规则状态统一管理

### 4.3 evalIndex 动态排序

```cpp
// src/detection/fp_detect.cc:491
static int sortOrderByContentLength(const void* e1, const void* e2)
{
    // longestPatternLen 长的优先
    if (otn1->longestPatternLen < otn2->longestPatternLen)
        return +1;  // 降序
    if (otn1->longestPatternLen > otn2->longestPatternLen)
        return -1;
    // 相等则按 sid 排序
    return (otn1->sigInfo.sid < otn2->sigInfo.sid) ? +1 : -1;
}
```

**排序策略**:
1. `longestPatternLen` 降序 - 最长模式优先（减少回退）
2. `sid` 升序 - 相同长度时稳定排序

### 4.4 RTN otnRefCount 引用计数

```cpp
// src/detection/treenodes.h:137
// reference count from otn.
// Multiple OTNs can reference this RTN with the same policy.
unsigned int otnRefCount = 0;
```

**用途**: 跟踪 RTN 被多少 OTN 引用，用于：
- 规则删除时判断是否可以释放 RTN
- 避免重复 RTN 内存分配

---

## 5. Fast Pattern 搜索流程

### 5.1 完整调用链

```
Packet 到达
    │
    ▼
DetectionEngine::detect()
    │
    ▼
fp_full() / fp_partial() / fp_complete()
    │
    ├─→ fpEvalPacket()  ─→ 按协议类型选择 eval 函数
    │                        │
    │                        ├─→ fpEvalHeaderTcp()  ─→ prmFindRuleGroupTcp()
    │                        ├─→ fpEvalHeaderUdp()  ─→ prmFindRuleGroupUdp()
    │                        ├─→ fpEvalHeaderIp()   ─→ prmFindRuleGroupIp()
    │                        └─→ fpEvalHeaderIcmp() ─→ prmFindRuleGroupIcmp()
    │
    ├─→ fpEvalHeaderSW() ─→ 遍历 RuleGroup
    │                        │
    │                        ├─→ eval_fp()     (Fast Pattern 评估)
    │                        │    │
    │                        │    └─→ fp_search()
    │                        │         │
    │                        │         ├─→ batch_search() → MpseBatch::search()
    │                        │         │                        │
    │                        │         │                        ▼
    │                        │         │                   rule_tree_queue()
    │                        │         │                        │
    │                        │         │                        ▼
    │                        │         │                   MpseStash::push()
    │                        │         │
    │                        │         └─→ MpseBatch::search_sync()
    │                        │              │
    │                        │              ▼
    │                        │         MpseStash::process()
    │                        │              │
    │                        │              ▼
    │                        │         rule_tree_match()
    │                        │              │
    │                        │              ├─→ detection_option_tree_evaluate()
    │                        │              │    │
    │                        │              │    └─→ 遍历检测选项树
    │                        │              │
    │                        │              └─→ fpEvalOption()  ─→ IpsOption::eval()
    │                        │
    │                        └─→ eval_nfp()  (Non-FP 评估)
    │                             │
    │                             └─→ detection_option_tree_evaluate()
    │                                  (遍历 nfp_tree)
    │
    └─→ fpFinalSelectEvent()
         │
         └─→ qsort()  ─→ 按 priority 或 longestPatternLen 排序
              │
              └─→ queue_event()  ─→ 事件入队
```

### 5.2 MpseStash 批处理机制

```cpp
// src/detection/fp_detect.cc:774
bool MpseStash::push(void* user, void* tree, int index, void* context, void* list)
{
    detection_option_tree_root_t* root = (detection_option_tree_root_t*)tree;

    // flowbit check 延迟处理
    bool checker = !root or root->otn->checks_flowbits();
    MatchStore& store = checker ? defer : queue;  // 分流

    // 去重
    if (dedup)
    {
        for (auto it = store.rbegin(); it != store.rend(); it++)
            if (tree == (*it).tree)
                return true;  // 已存在，跳过
    }

    // 队列满时批量处理
    if (qmax == queue.size() and is_packet_thread())
        process((IpsContext*)context, queue);

    store.push_back({ user, tree, list, index });
    return true;
}
```

**处理流程**:
1. **push()**: MPSE 匹配回调，每次匹配调用一次
2. **defer 队列**: flowbit 相关规则延迟处理
3. **queue 队列**: 正常规则，批量处理
4. **process()**: packet thread 空闲时处理所有暂存结果

### 5.3 为什么需要 Fast Pattern

```
without Fast Pattern:
┌────────────────────────────────────────────────────────┐
│ 遍历所有规则 OTN ─→ OTN ─→ OTN ─→ OTN ─→ ... (N条)    │
│   每条规则的所有检测选项都要执行                        │
└────────────────────────────────────────────────────────┘
   时间复杂度: O(N × M)  (N=规则数, M=平均选项数)

with Fast Pattern:
┌────────────────────────────────────────────────────────┐
│ 1. MPSE 快速模式匹配 (AC/Hyperscan)                    │
│    只匹配特定 content patterns                         │
├────────────────────────────────────────────────────────┤
│ 2. 匹配成功的 OTN 才执行完整检测选项树                  │
│    不匹配的规则直接跳过                                │
└────────────────────────────────────────────────────────┘
   时间复杂度: O(K × M)  (K=匹配数 << N, M=选项数)
```

**longestPatternLen 的作用**:
- 更长的模式更独特，优先匹配可减少回退
- 模式越长，假阳性越低

---

## 6. 规则编译到树的流程

### 6.1 文本规则 → OTN + RTN

```
文本规则:
alert tcp $HOME_NET any -> $EXTERNAL_NET 80 (msg:"SQL injection"; \
    content:"SELECT"; content:"FROM"; content:"WHERE"; ...)

         │
         ▼
┌─────────────────────────────────────────────────────┐
│                    解析阶段                          │
│  ├── ParseRule()                                     │
│  │   ├── 解析 action: alert                         │
│  │   ├── 解析 proto: tcp                            │
│  │   ├── 解析 src: $HOME_NET any                    │
│  │   ├── 解析 dst: $EXTERNAL_NET 80                 │
│  │   ├── 解析 direction: ->                         │
│  │   └── 解析 options: msg, content, etc.           │
│  │                                                   │
│  ├── 创建 RTN (RuleTreeNode)                        │
│  │   ├── sip = $HOME_NET                            │
│  │   ├── dip = $EXTERNAL_NET                        │
│  │   ├── src_portobject = any                        │
│  │   ├── dst_portobject = 80                        │
│  │   └── action = alert                             │
│  │                                                   │
│  └── 创建 OTN (OptTreeNode)                          │
│      ├── sigInfo.gid = 1                            │
│      ├── sigInfo.sid = 1000                         │
│      ├── sigInfo.message = "SQL injection"          │
│      ├── opt_func 链表                              │
│      │   ├── content:"SELECT"                       │
│      │   ├── content:"FROM"                         │
│      │   └── content:"WHERE"                        │
│      └── longestPatternLen = 最大 content 长度      │
└─────────────────────────────────────────────────────┘
```

### 6.2 fpCreateFastPacketDetection 流程

```cpp
// src/detection/fp_create.cc:1556
int fpCreateFastPacketDetection(SnortConfig* sc)
{
    // 1. 创建 Protocol Rule Maps
    fpCreateRuleMaps(sc, port_tables);
    // 结果: prmTcpRTNX, prmUdpRTNX, prmIpRTNX, prmIcmpRTNX

    // 2. 创建 Rule Groups (Port Table → RuleGroup)
    fpCreateRuleGroups(sc, port_tables);
    // 对每个 PortObject2 创建 RuleGroup
    // 调用 fpAddRuleGroupRule() 添加规则

    // 3. 创建 Service Rule Groups (基于 metadata service:)
    fpCreateServiceRuleGroups(sc);
    // 按服务名分组规则 (http, ftp, smtp, etc.)

    // 4. 编译 MPSE
    compile_mpses(sc, can_build_mt(fp));
    // 调用 Mpse::prep_patterns() 编译搜索自动机

    // 5. 打印统计信息
    fp_print_port_groups(port_tables);
    fp_print_service_groups(sc->spgmmTable, !label);
}
```

### 6.3 fpAddRuleGroupRule 流程

```cpp
// src/detection/fp_create.cc:489
static int fpAddRuleGroupRule(
    SnortConfig* sc, RuleGroup* pg, OptTreeNode* otn, FastPatternConfig* fp,
    const char* srvc, bool to_server)
{
    // 1. 获取 Fast Pattern 内容
    PatternMatchVector pmv = get_fp_content(otn, ofp, opt, ...);

    if (!pmv.empty())
    {
        // 2. 选择/创建 PatternMatcher
        PatternMatcher* pm = pg->get_pattern_matcher(pmt, s, sect);
        MpseGroup* mpg = &pm->group;

        // 3. 创建 MPSE (如果需要)
        if (!mpg->normal_mpse)
            mpg->create_normal_mpse(sc, &agent);

        // 4. 添加 pattern 到 MPSE
        PMX* pmx = snort_calloc(sizeof(PMX));
        pmx->rule_node.rnRuleData = otn;
        pmx->pmd = pmd;

        mpse->add_pattern(pattern, pattern_length, desc, pmx);

        // 5. 更新 longestPatternLen
        if (pmd->pattern_size > otn->longestPatternLen)
            otn->longestPatternLen = pmd->pattern_size;
    }
    else
    {
        // 6. 无 FP 的规则加入 NFP 链表
        pg->add_nfp_rule(otn);
    }
}
```

---

## 7. 与 Snort2 静态规则树的对比

### 7.1 Snort2 静态规则树

```
Snort2 编译时:
┌─────────────────────────────────────────────────────────┐
│ 静态规则树 (编译时构建)                                 │
│                                                          │
│   RTN ─→ OTN ─→ OTN ─→ OTN                             │
│    │       │       │                                    │
│    └──┬────┴───────┘                                    │
│       ▼                                                 │
│    (规则头相同，规则体不同)                              │
└─────────────────────────────────────────────────────────┘

问题:
1. 规则加载后不可更改 (需重启)
2. 内存布局固定
3. 难以支持多 policy 动态切换
```

### 7.2 Snort3 动态 OTN Tree

```
Snort3 运行时:
┌─────────────────────────────────────────────────────────┐
│ 动态 OTN Tree (运行时构建)                               │
│                                                          │
│  PORT_RULE_MAP (三层索引)                                │
│       │                                                  │
│       ├─→ prmDstPort[80]  ─→ RuleGroup ─→ MPSE + OTN    │
│       │                              └→ OTN ─→ OTN       │
│       │                                                 │
│       ├─→ prmDstPort[443] ─→ RuleGroup ─→ MPSE + OTN    │
│       │                                                 │
│       └─→ prmGeneric     ─→ RuleGroup ─→ MPSE + OTN     │
│                                                          │
│ 支持的操作:                                              │
│ 1. 原子替换整个配置                                      │
│ 2. 规则启用/禁用 (不改树结构)                            │
│ 3. 多 policy 共存                                        │
│ 4. evalIndex 动态排序                                    │
└─────────────────────────────────────────────────────────┘
```

### 7.3 性能对比

| 特性 | Snort2 | Snort3 |
|------|--------|--------|
| 端口查找 | O(n) 遍历 | O(1) 数组索引 |
| 规则重载 | 需重启 | 原子替换，无需锁 |
| 多 policy | 静态分区 | 动态 proto_nodes |
| FP 排序 | 固定顺序 | longestPatternLen 排序 |
| 内存布局 | 连续数组 | 分散 + 指针链接 |

---

## 8. 关键文件索引

| 文件 | 作用 |
|------|------|
| `src/detection/treenodes.h` | RTN, OTN, OptFpList 结构定义 |
| `src/detection/pcrm.h/cc` | PORT_RULE_MAP 索引管理 |
| `src/ports/port_group.h/cc` | RuleGroup, PatternMatcher, MpseGroup |
| `src/detection/fp_create.cc` | 规则编译到树 |
| `src/detection/fp_detect.cc` | Fast Pattern 检测流程 |
| `src/detection/fp_utils.cc` | FP 工具函数 |
| `src/framework/mpse.h` | Mpse 基类 |
| `src/framework/mpse_batch.h` | MpseBatch, MpseStash |
| `src/detection/ips_context.h` | IpsContext 检测上下文 |
| `src/detection/detection_options.h` | 检测选项树结构 |

---

## 9. 总结

Snort3 的 Dynamic OTN Tree 是其检测引擎的核心创新：

1. **三层索引结构** (PORT_RULE_MAP) 实现 O(1) 端口查找
2. **动态规则加载** 通过原子配置替换实现，无需停止检测
3. **MpseStash 批处理** 减少函数调用开销
4. **longestPatternLen 排序** 优化 fast pattern 匹配效率
5. **proto_nodes 多 policy 索引** 一条规则支持多个策略
6. **RTN/OTN 分离设计** 规则头体分离，便于共享和复用

这套设计使得 Snort3 在保持高性能的同时，获得了 Snort2 无法实现的运行时灵活性。
