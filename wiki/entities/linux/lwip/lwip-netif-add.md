---
type: entity
tags: [linux, lwip, network, netif]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP netif_add Analysis

## 定义

`netif_add()` 是 lwIP 注册网络接口的核心函数，完成初始化、编号分配、链表插入。

## 核心流程

```
netif_add()
    │
    ├─► 参数校验 (LWIP_ASSERT_CORE_LOCKED)
    ├─► 初始化字段 (IP、checksum、MTU)
    ├─► 保存 state、input 函数
    ├─► netif_set_addr() 设置 IP 地址
    ├─► init() 调用 driver 初始化
    ├─► 分配唯一编号 (num) — O(n²) 冲突检查
    ├─► 添加到 netif_list 链表头部
    └─► 返回 netif
```

## 编号分配机制

```c
// 编号范围
netif->num: 0-254 (内部编号)
netif_get_index(): 1-255 (对外索引, = num + 1)
NETIF_NO_INDEX: 0 (无效索引)

// 分配算法 — O(n²) 最坏情况
do {
    if (netif->num == 255) netif->num = 0;  // 绕回
    for (netif2 = netif_list; netif2 != NULL; netif2 = netif2->next) {
        if (netif2->num == netif->num) {
            netif->num++;  // 冲突，递增
            break;
        }
    }
} while (netif2 != NULL);  // 直到没有冲突
```

## netif_get_by_index

```c
struct netif *netif_get_by_index(u8_t idx) {
    NETIF_FOREACH(netif) {
        if (idx == netif_get_index(netif)) {
            return netif;  // O(n) 遍历
        }
    }
    return NULL;
}
```

## 使用场景

| 场景 | 函数 | 说明 |
|------|------|------|
| UDP recv | `udp_input()` | 通过 `if_idx` 查找源 netif |
| Raw PCB | `raw_input()` | 检查 PCB 绑定的 netif |
| LWFW | `lwfw_ct` | connection tracking 使用 netif 索引 |
| SNMP | `snmp` | MIB-II ifTable 索引 |

## SafeOS 初始化顺序

```
lwip_init()
    ├─► netif_init()
    │     └─► netif_add(&loop_netif, ...)  // loopback
    ├─► tcpip_init()
    │     └─► tcpip_thread 创建
    └─► vnetif_setup()
          └─► netif_add(&vnet_if, ethif_init, ethernet_input)
                └─► vlanif_setup()
                      └─► netif_add(&vlan_if[i], ethif_init, ethernet_input)
```

## 相关概念

- [[entities/linux/lwip/lwip-netif]] — netif 结构详解
- [[entities/linux/lwip/lwip-ethernet-input]] — 调用 netif->input 的入口
- [[entities/linux/lwip/lwip-udp-input]] — 使用 netif_get_by_index
- [[entities/linux/lwip/lwip-tcpip-thread]] — tcpip_thread 上下文要求
