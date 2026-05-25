---
type: entity
tags: [linux, lwfw, firewall, network, safeos]
created: 2026-05-25
sources: [safeos-lwfw]
---

# LWFW Packet Classification

## 定义

LWFW 使用 **5-tuple** (源IP, 目的IP, 协议, 源端口, 目的端口) 进行数据包分类，支持 L2/L3/L4 多层匹配规则。

## 分类入口

### ip4_filter — 主分类函数

```c
static int ip4_filter(lwfw_firewall_t *fw, const struct pbuf *p,
                      const struct netif *inp, lwfw_table_flag_t dir)
{
  lwfw_pkt_info_t pkt_info = {0};
  match_result_t ret_rule = {0};

  lwfw_pkt_info_constructor(p, inp, &pkt_info, dir);
  ret = filter_engine->do_filter((void *)policy, &pkt_info, &ret_rule);

  if (ret_rule.action & LWFW_ACTION_CODE_EVENT) {
    lwfw_generate_secure_event(&ret_rule, p, &pkt_info, ...);
  }
  return ret_rule.action;
}
```

## 包解析

### L2 信息 (Ingress)

| 字段 | 说明 |
|------|------|
| `ether_type` | Ethernet 类型 |
| `vlan` | VLAN ID |
| `src_mac` / `dst_mac` | MAC 地址 + 掩码 |

### L3 信息

| 字段 | 说明 |
|------|------|
| `src_ip` / `dst_ip` | IPv4 地址 (支持 CIDR 掩码) |
| `proto` | 协议号 (TCP/UDP/ICMP/IP) |

### L4 信息

| 字段 | 说明 |
|------|------|
| `src_port` / `dst_port` | 端口号 (支持范围或列表) |

## 规则匹配流程

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
    │     ├─► CT_STATE 匹配
    │     ├─► NETIF 匹配
    │     ├─► L2 匹配 (MAC, VLAN, Ethertype)
    │     ├─► L3 匹配 (IP, Protocol)
    │     └─► L4 匹配 (Ports)
    │
    ▼
找到匹配规则?
    ├─► 是 → 应用规则动作 (ALLOW/DENY/EVENT)
    └─► 否 → 应用默认动作
```

## 动作类型

| 动作 | 代码 | 说明 |
|------|------|------|
| ALLOW | 0x01 | 允许通过 |
| DENY | 0x02 | 拒绝 (丢弃) |
| EVENT | 0x04 | 生成事件 |
| LOGGING | 0x08 | 记录日志 |

## 相关概念

- [[entities/linux/lwip/lwip-tcpip-thread]] — 过滤执行上下文
- [[entities/linux/lwfw/lwfw-architecture]] — 引擎抽象架构
- [[entities/linux/lwfw/lwfw-core-filtering]] — 核心过滤逻辑
- [[entities/linux/lwfw/lwfw-stats]] — 统计计数
