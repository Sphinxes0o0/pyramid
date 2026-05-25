---
type: entity
tags: [linux, lwip, network, bridge, fdb, safeos]
created: 2026-05-25
sources: [safeos-lwip-extensions]
---

# lwIP Bridgeif — 802.1D MAC Bridge

## 定义

lwIP 支持 IEEE 802.1D MAC Bridge，通过 Bridge netif + Port netifs 实现二层转发：Bridge netif 聚合多个 Port，Port 连接物理网卡或虚拟接口。

## 数据结构

### bridgeif_private_t
```c
typedef struct {
  struct netif *netif;           // Bridge netif
  struct eth_addr ethaddr;       // Bridge MAC 地址
  u16_t max_fdb_entries;         // 最大 FDB 条目数
  bridgeif_fdb_dynamic_entry_t *fdbd;  // 动态 FDB
  bridgeif_fdb_static_entry_t *fdbs;   // 静态 FDB
  bridgeif_port_t **ports;       // 端口数组
  int num_ports;                 // 端口数量
} bridgeif_private_t;
```

### bridgeif_port_t
```c
typedef struct bridgeif_port_s {
  struct netif *portif;     // 端口对应的 netif
  struct netif *bridge;      // 所属 bridge
  u8_t port_num;            // 端口号
  netif_input_fn old_input; // 原始 input 函数
} bridgeif_port_t;
```

## FDB (Forwarding Database)

### FDB 学习
```c
void bridgeif_fdb_update_src(void *fdb_ptr, struct eth_addr *src_addr, u8_t port_idx)
{
  // 查找现有条目，更新时间戳
  // 无则分配新条目
}
```

### FDB 查找
```c
bridgeif_portmask_t bridgeif_find_dst_ports(void *fdb_ptr, struct eth_addr *dst)
{
  // 查找 MAC → 端口映射
  // 未找到则泛洪
  return BRIDGEIF_FLOOD_PORTMASK;
}
```

### FDB 老化
```
BR_FDB_TIMEOUT_SEC = 60*5  // 5分钟超时
每 tick --ts, ts==0 时删除条目
```

## Bridge 输入 (bridgeif_input)

```
Port netif 收到帧
    │
    ▼
bridgeif_input(p, netif)
    │
    ├─► 更新源 MAC 学习 (bridgeif_fdb_update_src)
    │
    ├─► 组播/广播 → flood + CPU
    │
    └─► 单播
          ├─► 本地 MAC → 发送到 CPU
          └─► 查找 FDB → 发送到对应端口
```

## Bridge 输出

```c
bridgeif_output(netif, p, dst)
    → bridgeif_send_to_ports(br, p, BRIDGEIF_ALL_PORTMASK)
        → br->ports[i]->portif->output()  // 复制 pbuf 到各端口
```

## 端口管理

```c
bridgeif_add_port(bridgeif, portif)
    → port->old_input = portif->input    // 保存原始 input
    → portif->input = bridgeif_input    // 替换为 bridgeif_input
    → portif->flags &= ~NETIF_FLAG_ETHARP  // 清除 ETHARP flag

bridgeif_remove_port(bridgeif, portif)
    → portif->input = port->old_input   // 恢复原始 input
```

## 与 Linux Bridge 对比

| 特性 | lwIP bridgeif | Linux Bridge |
|------|----------------|--------------|
| 实现位置 | 用户空间 lwIP | 内核网络栈 |
| FDB 数量 | 可配置 | 可配置 |
| 学习机制 | 基于源 MAC | 基于源 MAC |
| 老化时间 | 5 分钟 | 可配置 |
| STP | 不支持 | 支持 |
| VLAN | 不支持 | 支持 |

## 相关概念

- [[entities/linux/lwip/lwip-virt-brg]] — VIRT_BRG 与 hypervisor 集成
- [[entities/linux/lwip/lwip-netif]] — netif 抽象

## 来源详情

- [[sources/safeos-lwip-extensions]]
