---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW LWCT Analysis (Root-Level)

## 定义

LWCT (Light Weight Connection Tracking) 是 LWFW 的**状态追踪模块**，为防火墙提供**状态ful 过滤**能力：追踪 TCP/UDP/ICMP 连接状态、在 pbuf 中绑定连接信息 (`pbuf->_lwct`)、支持规则基于连接状态匹配 (NEW/ESTABLISHED/RELATED)。

## 核心数据结构

### lwct_tuple

```c
struct lwct_tuple {
    struct lwct_info src;  // 源 IP + Port
    struct lwct_info dst;  // 目的 IP + Port
    uint8_t protonum;      // L4 协议 (TCP/UDP/ICMP)
    lwct_dir_t dir;        // 方向 (ORIGINAL/REPLY)
};

struct lwct_info {
    uint32_t ip;           // IPv4 地址
    union {
        struct { uint16_t port; } tcp;
        struct { uint16_t port; } udp;
        struct { uint8_t type, code; } icmp;
    } u;
};
```

### lwct_conn

```c
struct lwct_conn {
    struct lwct_tuple_hash tuple_hash[2];  // 哈希链表 (原始+回复)
    uint32_t status;                     // 状态标志
    uint64_t timeout;                   // 超时时间
    struct lwct_ext *ext;               // 扩展数据
};
```

## 连接追踪流程

```c
// lwct_in() 主 Hook
static inline int lwct_in(struct pbuf *pbuf, uint32_t dir)
{
    conn = lwct_find_conn(pbuf, dir);
    if (conn == NULL) {
        conn = lwct_create_conn(pbuf, dir);
        state = LWCT_NEW;
    } else {
        state = lwct_update_state(conn, dir);
    }
    pbuf->_lwct = (uint64_t)conn | state;
    return ERR_OK;
}
```

## 哈希表规模

| 参数 | 值 |
|------|-----|
| 桶数 | 8192 |
| 锁数 | 256 |
| 每锁桶数 | 32 |
| 最大连接数 | 8192 |

## 状态机

| 协议 | 状态转换 |
|------|---------|
| TCP | NEW → SYN_SENT → SYNACK → ESTABLISHED → FIN → CLOSED |
| UDP | First Packet (NEW) → Reply → REPLIED → TIMEOUT → EXPIRED |

## 超时配置

| 协议 | unreplied_tmo | replied_tmo | established_tmo |
|------|---------------|-------------|-----------------|
| TCP | 180s | 180s | 10800s (3h) |
| UDP | 60s | 60s | 10800s (3h) |
| ICMP | 10s | 10s | - |

## 与 LWFW 规则集成

```c
// 规则匹配时检查连接状态
if (rule->flags & LWFW_RULE_FLAG_CT_STATE) {
    lwct_state_t ct_state = pbuf->_lwct & LWCT_STATE_MASK;
    switch (rule->ct_state) {
        case LWFW_CT_NEW:        if (ct_state != LWCT_NEW) return false; break;
        case LWFW_CT_ESTABLISHED: if (ct_state != LWCT_ESTABLISHED) return false; break;
    }
}
```

## 相关概念

- [[entities/linux/lwfw/lwfw-lwct-gc-analysis]] — GC 线程与内存管理
- [[entities/linux/lwfw/lwfw-lwct-interaction]] — LWFW 与 LWCT 交互
- [[entities/linux/lwfw/lwfw-classification]] — CT_STATE 匹配
- [[entities/linux/lwfw/lwfw-stats]] — CT 统计
