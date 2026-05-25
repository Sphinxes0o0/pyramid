# lwfw VLAN 拦截流程分析

> 分析日期: 2026/04/22
> 代码版本: release/vsel4.01.04.04

---

## 1. 整体流程

```
数据包进入 ip4_input()
         │
         ▼
┌─────────────────────────────────────┐
│  ip4.c:743-759                     │
│  lwfw_p->ops->ingress_filter(p, inp)│
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  lwfw.c:802                         │
│  ip4_filter_dispatch_incoming()     │
│  - 检查规则表 state                  │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  lwfw.c:724                         │
│  ip4_filter()                       │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  lwfw.c:361-366                     │
│  lwfw_pkt_l2_info_constructor()    │
│  - 从 Ethernet 头提取 VLAN Tag       │
└─────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  lwfw.c:383-437                     │
│  check_lwfw_l2_info()               │
│  - 匹配 rule->l2.vlan              │
│    与 packet->l2.vlan              │
└─────────────────────────────────────┘
```

---

## 2. 关键代码位置

| 位置 | 函数 | 说明 |
|------|------|------|
| `ip4.c:743-759` | `ip4_input()` | Hook 注入点 |
| `lwfw.c:802` | `ip4_filter_dispatch_incoming()` | Ingress 入口 |
| `lwfw.c:724` | `ip4_filter()` | 主过滤函数 |
| `lwfw.c:361-366` | `lwfw_pkt_l2_info_constructor()` | 调用 L2 解析 |
| `lwfw.c:232-259` | `lwfw_pkt_l2_info_constructor()` | **VLAN 提取位置** |
| `lwfw.c:383-437` | `check_lwfw_l2_info()` | L2 匹配（含 VLAN） |
| `lwfw.c:392` | `check_lwfw_l2_info()` | **VLAN 匹配条件** |

---

## 3. VLAN 提取（L2 解析）

```c
// lwfw.c:232-259
inline static void lwfw_pkt_l2_info_constructor(...) {
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
  }
}
```

**要点：**
- VLAN 提取发生在 Ingress 方向
- EtherType 为 `0x8100` 时表示存在 VLAN Tag
- VLAN ID 从 VLAN Tag 中提取（12bit，范围 1-65535）

---

## 4. VLAN 匹配

```c
// lwfw.c:392
if ((rule->flags & LWFW_RULE_FLAGS_VLAN) &&
    rule_l2_info->vlan != packet_info->vlan)
  goto out;  // 不匹配，跳出
```

**匹配逻辑：**
- 规则中设置 `vlan: 100` 时，`LWFW_RULE_FLAGS_VLAN` 标志位被设置
- 数据包 VLAN 与规则 VLAN 精确匹配
- 不匹配则返回 false，跳过该规则

---

## 5. Egress 方向限制

**Egress 路径不支持 VLAN 过滤**

原因：
- `lwfw_pkt_l2_info_constructor()` 仅在 `dir == LWFW_IN_TABLE` 时填充 L2 信息
- Egress 方向 L2 头在发送前可能已被修改或剥离

Egress 流程：
```
ip4_output_if() → lwfw_p->ops->egress_filter() → ip4_filter_dispatch_outgoing() → ip4_filter()
```

---

## 6. 编译选项要求

```c
#define LWFW_ADVANCED_FUNC_L2 1  // 启用 L2 (VLAN/MAC) 过滤
#define NIO_LWIP_LWFW 1           // 启用 lwfw 功能
```

---

## 7. 限制说明

| 限制项 | 说明 |
|--------|------|
| 仅 Ingress 方向 | Egress 方向无法使用 VLAN 条件过滤 |
| VLAN 精确匹配 | 只能匹配单个 VLAN ID，不支持范围 |
| 需编译选项 | 必须启用 `LWFW_ADVANCED_FUNC_L2` |

---

## 8. 相关文档

| 文档 | 说明 |
|------|------|
| [lwfw_architecture.md](lwfw_architecture.md) | lwfw 整体架构 |
| [lwfw_filter_flow.md](lwfw_filter_flow.md) | 规则匹配流程 |
| [lwfw_vlan_isolation_guide.md](lwfw_vlan_isolation_guide.md) | VLAN 隔离配置指南 |
| [lwfw_data_structure.md](lwfw_data_structure.md) | 数据结构设计 |
