# Bridgeif 分析 — T-064/T-065

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: 802.1D bridge 实现、FDB 学习/老化、port_input/port_output

---

## 1. 概述

lwIP 支持 IEEE 802.1D MAC Bridge，通过多层 netif 实现：
1. **Bridge netif**: 桥设备虚拟网络接口
2. **Port netifs**: 桥的各个端口 (连接物理网卡)

---

## 2. 文件结构

| 文件 | 描述 |
|------|------|
| `netif/bridgeif.c` | Bridge 主逻辑 |
| `netif/bridgeif_fdb.c` | FDB (Forwarding Database) 实现 |
| `include/netif/bridgeif.h` | Bridge 头文件 |
| `include/netif/bridgeif_opts.h` | Bridge 配置选项 |

---

## 3. 数据结构

### 3.1 bridgeif_initdata_t

```c
typedef struct {
  u8_t max_ports;           // 最大端口数
  u16_t max_fdb_entries;   // 最大 FDB 条目数
  u16_t max_static_entries;  // 最大静态 MAC 条目数
  struct eth_addr ethaddr;  // Bridge MAC 地址
} bridgeif_initdata_t;
```

### 3.2 bridgeif_private_t

```c
typedef struct {
  struct netif *netif;           // 指向 bridge netif
  struct eth_addr ethaddr;      // Bridge MAC 地址
  u16_t max_fdb_entries;        // 最大 FDB 条目数
  u16_t max_static_entries;      // 最大静态条目数
  bridgeif_fdb_dynamic_entry_t *fdbd;  // 动态 FDB
  bridgeif_fdb_static_entry_t *fdbs;   // 静态 FDB
  int used_fdb_entries;          // 已用 FDB 条目
  int used_fdbs_entries;         // 已用静态条目
  bridgeif_port_t **ports;      // 端口数组
  int num_ports;                // 端口数量
} bridgeif_private_t;
```

### 3.3 bridgeif_port_t

```c
typedef struct bridgeif_port_s {
  struct netif *portif;     // 端口对应的 netif
  struct netif *bridge;      // 所属 bridge
  u8_t port_num;            // 端口号
  u8_t old_flags;           // 原始 netif flags
  netif_input_fn old_input; // 原始 input 函数
} bridgeif_port_t;
```

---

## 4. FDB (Forwarding Database)

### 4.1 FDB 结构

**文件**: `bridgeif_fdb.c`

```c
#define BR_FDB_TIMEOUT_SEC  (60*5)  // 5 分钟超时

typedef struct bridgeif_dfdb_entry_s {
  u8_t used;              // 是否使用
  u8_t port;             // 出口端口
  u32_t ts;              // 时间戳 (用于老化)
  struct eth_addr addr;  // MAC 地址
} bridgeif_dfdb_entry_t;
```

### 4.2 FDB 学习

**bridgeif_fdb_update_src()**

```c
void bridgeif_fdb_update_src(void *fdb_ptr, struct eth_addr *src_addr, u8_t port_idx)
{
  // 1. 查找现有条目
  for (i = 0; i < fdb->max_fdb_entries; i++) {
    if (e->used && e->ts &&
        !memcmp(&e->addr, src_addr, sizeof(struct eth_addr))) {
      // 更新已有条目
      e->ts = BR_FDB_TIMEOUT_SEC;
      e->port = port_idx;
      return;
    }
  }

  // 2. 分配新条目
  for (i = 0; i < fdb->max_fdb_entries; i++) {
    if (!fdb->fdb[i].used) {
      fdb->fdb[i].used = 1;
      fdb->fdb[i].port = port_idx;
      fdb->fdb[i].ts = BR_FDB_TIMEOUT_SEC;
      memcpy(&fdb->fdb[i].addr, src_addr, sizeof(struct eth_addr));
      return;
    }
  }
}
```

### 4.3 FDB 查找

**bridgeif_find_dst_ports()**

```c
bridgeif_portmask_t bridgeif_find_dst_ports(void *fdb_ptr, struct eth_addr *dst)
{
  // 查找目标 MAC 地址对应的端口
  for (i = 0; i < fdb->max_fdb_entries; i++) {
    if (e->used && !memcmp(&e->addr, dst, sizeof(struct eth_addr))) {
      return 1 << e->port;  // 返回端口掩码
    }
  }
  return BRIDGEIF_FLOOD_PORTMASK;  // 未找到，泛洪
}
```

### 4.4 FDB 老化

```c
void bridgeif_fdb_age(void *fdb_ptr)
{
  for (i = 0; i < fdb->max_fdb_entries; i++) {
    if (e->used && e->ts > 0) {
      e->ts--;
      if (e->ts == 0) {
        e->used = 0;  // 删除过期条目
      }
    }
  }
}
```

---

## 5. Bridge 输入 (bridgeif_input)

### 5.1 接收流程

```
Port netif 收到帧
    │
    ▼
portif->input = bridgeif_input  // 替换原始 input
    │
    ▼
bridgeif_input(p, netif)
    │
    ├─► 获取 bridge 和 port 信息
    │
    ├─► 更新源 MAC 学习 (bridgeif_fdb_update_src)
    │
    ├─► 目标地址类型判断:
    │     │
    │     ├─► 组播/广播 → flood + CPU
    │     │
    │     └─► 单播
    │           │
    │           ├─► 本地 MAC → 发送到 CPU
    │           │
    │           └─► 查找 FDB → 发送到对应端口
    │
    ▼
pbuf_free(p)
```

### 5.2 bridgeif_input 代码

```c
static err_t bridgeif_input(struct pbuf *p, struct netif *netif)
{
  // 获取端口和 bridge 信息
  port = netif_get_client_data(netif, bridgeif_netif_client_id);
  br = port->bridge;

  // 保存接收端口索引
  p->if_idx = netif_get_index(netif);

  // 解析 Ethernet header
  dst = (struct eth_addr *)p->payload;
  src = (struct eth_addr *)(((u8_t *)p->payload) + sizeof(struct eth_addr));

  // 学习源 MAC
  if ((src->addr[0] & 1) == 0) {
    bridgeif_fdb_update_src(br->fdbd, src, port->port_num);
  }

  // 组播/广播处理
  if (dst->addr[0] & 1) {
    dstports = bridgeif_find_dst_ports(br, dst);  // 查找组播端口
    bridgeif_send_to_ports(br, p, dstports);        // 发送到所有端口
    if (dstports & (1 << BRIDGEIF_MAX_PORTS)) {
      br->netif->input(p, br->netif);  // 发送到 CPU
    }
    return ERR_OK;
  }

  // 单播处理
  if (bridgeif_is_local_mac(br, dst)) {
    // 目标在 bridge 本地，直接发送到 CPU
    return br->netif->input(p, br->netif);
  }

  // 查找目标端口并发送
  dstports = bridgeif_find_dst_ports(br, dst);
  bridgeif_send_to_ports(br, p, dstports);
  pbuf_free(p);
  return ERR_OK;
}
```

---

## 6. Bridge 输出

### 6.1 bridgeif_output

```c
err_t bridgeif_output(struct netif *netif, struct pbuf *p,
                      const struct eth_addr *dst)
{
  // 发送到所有端口 (除了源端口)
  bridgeif_send_to_ports(br, p, BRIDGEIF_ALL_PORTMASK);
}
```

### 6.2 bridgeif_send_to_ports

```c
static void bridgeif_send_to_ports(bridgeif_private_t *br, struct pbuf *p,
                                   bridgeif_portmask_t ports)
{
  int i;
  struct pbuf *q;

  for (i = 0; i < br->num_ports; i++) {
    if (ports & (1 << i)) {
      // 复制 pbuf 并发送到对应端口
      q = pbuf_copy(p);
      br->ports[i]->portif->output(br->ports[i]->portif, q, dst);
    }
  }
}
```

---

## 7. 端口管理

### 7.1 添加桥端口

```c
err_t bridgeif_add_port(struct netif *bridgeif, struct netif *portif)
{
  // 1. 保存原始 input 函数
  port->old_input = portif->input;
  port->old_flags = portif->flags;

  // 2. 替换为 bridgeif_input
  portif->input = bridgeif_input;

  // 3. 清除端口的 ETHARP flag (桥只有一个 IP)
  portif->flags &= ~NETIF_FLAG_ETHARP;
}
```

### 7.2 移除桥端口

```c
err_t bridgeif_remove_port(struct netif *bridgeif, struct netif *portif)
{
  // 恢复原始 input 函数
  portif->input = port->old_input;
  portif->flags = port->old_flags;
}
```

---

## 8. 与 Linux Bridge 对比

| 特性 | lwIP bridgeif | Linux Bridge |
|------|----------------|--------------|
| **实现位置** | 用户空间 lwIP | 内核网络栈 |
| **FDB 数量** | 可配置 | 可配置 |
| **学习机制** | 基于源 MAC | 基于源 MAC |
| **老化时间** | 5 分钟 | 可配置 |
| **STP** | 不支持 | 支持 (生成树协议) |
| **VLAN** | 不支持 | 支持 |

---

## 9. 总结

### 9.1 关键设计

1. **多层 netif**: Bridge + Ports 结构
2. **FDB 学习**: 基于源 MAC 的动态学习
3. **泛洪**: 未知单播帧泛洪到所有端口
4. **组播**: 组播帧泛洪 + CPU

### 9.2 数据流

```
RX:
  Port netif → bridgeif_input() → FDB lookup → Port(s) or CPU

TX:
  Bridge netif → bridgeif_output() → FDB lookup → All Ports
```

### 9.3 SafeOS 中的使用

SafeOS 可能使用 VIRT_BRG 来实现 VM 间桥接，这需要与 hypervisor 集成。
