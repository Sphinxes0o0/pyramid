---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW and LWCT Interaction

## 定义

LWFW 与 LWCT 通过 **pbuf->_lwct 扩展字段**紧密集成：LWCT 在包处理早期创建/查找连接并绑定状态，LWFW 在规则匹配时读取该状态进行连接感知过滤。

## 交互架构

```
数据包进入 lwIP
    │
    ▼
┌─────────────────────────────────────┐
│  lwct_main_hook(pbuf, dir)          │  ← lwct 主 hook
│    └─ lwct_in(pbuf, dir)            │
│          ├─ lwct_hash_tuple()        │
│          ├─ lwct_lookup_conn()      │
│          └─ pbuf->_lwct = conn|state │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  ip4_filter(pbuf, inp)              │  ← lwfw 过滤
│    ├─ lwfw_pkt_info_constructor()  │
│    │     └─ pkt_info->ct_state =  │
│    │         p->_lwct & LWCT_STATE_MASK │
│    └─ filter_engine->do_filter()    │
└─────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────┐
│  lwct_post_hook(pbuf, dir)          │  ← lwct post hook
│    ├─ lwct_confirm()               │
│    └─ lwct_pbuf_free_cb()          │
└─────────────────────────────────────┘
```

## pbuf 扩展字段

```c
struct pbuf {
    uintptr_t _lwct;  // lwct 扩展: conn_ptr | state
};

// 编码: [高位: 连接指针] [低3位: 状态]
// ptr = _lwct & LWCT_PTR_MASK (~0x7)
// state = _lwct & LWCT_STATE_MASK (0x7)
```

## 连接状态转换

```c
lwct_state_t lwct_convert_reply_state(lwct_state_t state)
{
  switch (state) {
    case LWCT_NEW:              return LWFW_CT_NEW;
    case LWCT_ESTABLISHED:      return LWFW_CT_ESTABLISHED;
    case LWCT_IS_REPLY:         return LWFW_CT_RELATED;
    case LWCT_ESTABLISHED_REPLY: return LWFW_CT_ESTABLISHED;
    default:                     return LWFW_CT_INVALID;
  }
}
```

## lwfw 中的使用

```c
// lwfw.c - check_rule()
if (rule->flags & LWFW_RULE_FLAGS_CT_STATE) {
  if (rule->ct_state != pkt_info->ct_state) {
    return false;
  }
}
```

## 未跟踪包的兜底机制

```c
// lwfw.c
if (lwct_enable == 1 && !p->_lwct) {
  LWFW_STATICS_INC(g_lwfw_stats.ct_notrack);
  if (policy->params.ct_oot_action == LWFW_CT_OOT_ACTION_PASS) {
    return ERR_OK;  // 放行未跟踪的包
  }
}
```

## 潜在并发问题

| 问题 | 说明 |
|------|------|
| pbuf->_lwct 并发访问 | lwct_in() 写入，lwfw 读取，lwct_post_hook() 清除 |
| 位操作非原子 | `_set_bit` / `_clear_bit` 在多线程下可能丢失更新 |

## 相关概念

- [[entities/linux/lwfw/lwfw-lwct]] — LWCT 模块整体
- [[entities/linux/lwfw/lwfw-lwct-gc-analysis]] — GC 线程分析
- [[entities/linux/lwfw/lwfw-classification]] — 规则匹配
- [[entities/linux/lwip/lwip-pbuf]] — pbuf 结构
