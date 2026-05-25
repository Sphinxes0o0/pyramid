# netif_add 与 netif_get_by_index 分析 — T-011

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: netif 注册到全局链表、netif_get_by_index 函数

---

## 1. 概述

`netif_add()` 是 lwIP 注册网络接口的核心函数，负责：
1. 初始化 netif 结构体字段
2. 分配唯一的 netif 编号 (num)
3. 将 netif 添加到全局链表 `netif_list`
4. 调用 driver 的 init 函数

`netif_get_by_index()` 通过索引查找 netif。

---

## 2. 数据结构

### 2.1 netif_list 链表

**文件**: `external/lwip_ds_mcu/src/core/netif.c:114`

```c
#if !LWIP_SINGLE_NETIF
struct netif *netif_list;  // 全局 netif 链表头
#endif /* !LWIP_SINGLE_NETIF */
struct netif *netif_default;  // 默认 netif 指针
```

### 2.2 NETIF_FOREACH 宏

**文件**: `netif.h:459-465`

```c
// 如果定义了 LWIP_NETIF_LOOPBACK，使用 netif_default 初始化
#if LWIP_NETIF_LOOPBACK
#define NETIF_FOREACH(netif) if (((netif) = netif_default) != NULL)

// 否则遍历 netif_list 链表
#else
#define NETIF_FOREACH(netif) for ((netif) = netif_list; (netif) != NULL; (netif) = (netif)->next)
#endif
```

### 2.3 netif 结构 (关键字段)

**文件**: `netif.h:297-400`

```c
struct netif {
    // 链表指针
    struct netif *next;  // 指向下一个 netif

    // IP 配置
    ip_addr_t ip_addr;   // IP 地址
    ip_addr_t netmask;   // 子网掩码
    ip_addr_t gw;        // 默认网关

    // 输入输出函数
    netif_input_fn input;   // packet 输入函数 (ethernet_input)
    netif_output_fn output; // IP 输出函数 (etharp_output)
    netif_linkoutput_fn linkoutput;  //链路层输出函数

    // 硬件信息
    u8_t hwaddr[NETIF_MAX_HWADDR_LEN];  // MAC 地址
    u8_t hwaddr_len;  // MAC 地址长度

    // 状态和标志
    u8_t flags;       // NETIF_FLAG_*
    u8_t num;         // netif 编号 (0-254)
    char name[2];     // 接口名称 (如 "en", "v0")
    char *fullname;   // 完整名称

    // 特定数据
    void *state;      // driver 特定状态
    void *client_data[...];  // 客户端数据

    u16_t mtu;        // 最大传输单元

    // ==== SafeOS 特供 ====
    u8_t vlanid;      // VLAN ID (SafeOS)
};
```

### 2.4 netif 编号与索引

```c
// netif 编号: 内部使用 (0-254)
netif->num

// netif 索引: 对外暴露 (1-255)
#define netif_get_index(netif) ((u8_t)((netif)->num + 1))

// idx = NETIF_NO_INDEX (0) 表示无效索引
```

---

## 3. netif_add 函数详解

**文件**: `netif.c:281-435`

```c
netif_add(struct netif *netif,
#if LWIP_IPV4
          const ip4_addr_t *ipaddr, const ip4_addr_t *netmask, const ip4_addr_t *gw,
#endif
          void *state, netif_init_fn init, netif_input_fn input)
{
    // ============================================
    // Step 1: 核心锁检查
    // ============================================
    LWIP_ASSERT_CORE_LOCKED();  // 必须在 tcpip_thread 上下文

    // ============================================
    // Step 2: 参数校验
    // ============================================
    LWIP_ERROR("netif_add: invalid netif", netif != NULL, return NULL);
    LWIP_ERROR("netif_add: No init function given", init != NULL, return NULL);

    // ============================================
    // Step 3: 初始化 IPv4 字段
    // ============================================
#if LWIP_IPV4
    ip_addr_set_zero_ip4(&netif->ip_addr);
    ip_addr_set_zero_ip4(&netif->netmask);
    ip_addr_set_zero_ip4(&netif->gw);
    netif->output = netif_null_output_ip4;  // 默认输出函数
#endif

    // ============================================
    // Step 4: 初始化 IPv6 字段
    // ============================================
#if LWIP_IPV6
    // ... 初始化 IPv6 地址数组
#endif

    // ============================================
    // Step 5: 设置校验和、MTU、标志
    // ============================================
    NETIF_SET_CHECKSUM_CTRL(netif, NETIF_CHECKSUM_ENABLE_ALL);
    netif->mtu = 0;
    netif->flags = 0;

    // ============================================
    // Step 6: 保存 state、编号、input 函数
    // ============================================
    netif->state = state;
    netif->num = netif_num;  // 分配编号
    netif->input = input;    // packet 输入函数

    // ============================================
    // Step 7: 设置 IP 地址
    // ============================================
#if LWIP_IPV4
    netif_set_addr(netif, ipaddr, netmask, gw);
#endif

    // ============================================
    // Step 8: 调用 driver 的 init 函数
    // ============================================
    if (init(netif) != ERR_OK) {
        return NULL;  // init 失败
    }

    // ============================================
    // Step 9: 分配唯一编号 (O(n²) 算法)
    // ============================================
#if !LWIP_SINGLE_NETIF
    {
        struct netif *netif2;
        int num_netifs;
        do {
            if (netif->num == 255) {
                netif->num = 0;  // 绕回
            }
            num_netifs = 0;
            // 遍历链表检查编号冲突
            for (netif2 = netif_list; netif2 != NULL; netif2 = netif2->next) {
                if (netif2->num == netif->num) {
                    netif->num++;  // 编号冲突，尝试下一个
                    break;
                }
                num_netifs++;
            }
        } while (netif2 != NULL);  // 直到没有冲突

        // 更新全局编号计数器
        if (netif->num == 254) {
            netif_num = 0;
        } else {
            netif_num = netif->num + 1;
        }
    }
#endif

    // ============================================
    // Step 10: 添加到链表头部
    // ============================================
#if !LWIP_SINGLE_NETIF
    netif->next = netif_list;  // 新 netif 指向当前链表头
    netif_list = netif;         // 链表头指向新 netif
#endif

    // ============================================
    // Step 11: IGMP/MLD 处理
    // ============================================
#if LWIP_IGMP
    if (netif->flags & NETIF_FLAG_IGMP) {
        igmp_start(netif);
    }
#endif

    return netif;  // 注册成功
}
```

---

## 4. netif_get_by_index 函数

**文件**: `netif.c:1744-1759`

```c
struct netif *
netif_get_by_index(u8_t idx)
{
    struct netif *netif;

    LWIP_ASSERT_CORE_LOCKED();

    if (idx != NETIF_NO_INDEX) {
        // 遍历 netif_list 链表
        NETIF_FOREACH(netif) {
            if (idx == netif_get_index(netif)) {  // idx = num + 1
                return netif;  // 找到
            }
        }
    }

    return NULL;  // 没找到
}
```

### 4.1 使用场景

| 场景 | 函数 | 说明 |
|------|------|------|
| **UDP recv** | `udp_input()` | 通过 `if_idx` 查找源 netif |
| **Raw PCB** | `raw_input()` | 检查 PCB 绑定的 netif |
| **LWFW** | `lwfw_ct` | connection tracking 使用 netif 索引 |
| **SNMP** | `snmp` | MIB-II ifTable 索引 |

---

## 5. SafeOS 中的 netif

### 5.1 SafeOS netif 初始化顺序

```
lwip_init()
    │
    ├─► netif_init()
    │     └─► netif_add(&loop_netif, ...)  // 添加 loopback
    │
    ├─► tcpip_init()
    │     └─► tcpip_thread 创建
    │
    └─► vnetif_setup()
          └─► netif_add(&vnet_if, ethif_init, ethernet_input)
                └─► vlanif_setup()
                      └─► netif_add(&vlan_if[i], ethif_init, ethernet_input)
```

### 5.2 SafeOS 特供字段

**文件**: `netif.h` (SafeOS 扩展)

```c
struct netif {
    // ... 标准字段 ...

    // ==== SafeOS 特供 ====
    u8_t vlanid;       // VLAN ID (0 表示不是 VLAN 接口)
    u8_t priority;     // 默认优先级
};
```

---

## 6. netif 链表遍历

### 6.1 NETIF_FOREACH 用法

```c
// 遍历所有 netif
NETIF_FOREACH(netif) {
    if (netif_is_up(netif) && netif_is_link_up(netif)) {
        // 处理 netif
    }
}

// 带提前退出
struct netif *find_netif_by_vlan(u16_t vlan_id) {
    struct netif *netif;
    NETIF_FOREACH(netif) {
        if (netif->vlanid == vlan_id) {
            return netif;
        }
    }
    return NULL;
}
```

### 6.2 SafeOS VLAN 分发中的遍历

**文件**: `ethernet.c:459-517` (lwip_arp_filter_netif_fn)

```c
case ETHTYPE_VLAN:
    NETIF_FOREACH(netif) {
        if (netif_is_up(netif)) {
            u16_t vid = PP_HTONS(netif->vlanid) & PP_HTONS(VLAN_ID_MASK);
            u16_t pkt_vid = vlan_hdr->prio_vid & PP_HTONS(VLAN_ID_MASK);
            if (vid == pkt_vid) {
                return netif;  // VID 匹配
            }
        }
    }
```

---

## 7. netif_num 编号分配

### 7.1 编号范围

```
netif->num: 0-254 (内部编号)
netif_get_index(): 1-255 (对外索引, = num + 1)
NETIF_NO_INDEX: 0 (无效索引)
```

### 7.2 分配算法

```c
// 全局计数器
static u8_t netif_num;  // 下个可用编号

// 分配流程
netif->num = netif_num;  // 先尝试当前编号

// 检查是否冲突
for (netif2 = netif_list; netif2 != NULL; netif2 = netif2->next) {
    if (netif2->num == netif->num) {
        netif->num++;  // 冲突，递增
        // 重新检查...
    }
}

// 更新全局计数器
if (netif->num == 254) {
    netif_num = 0;  // 绕回到 0
} else {
    netif_num = netif->num + 1;
}
```

### 7.3 链表添加位置

```c
// 新 netif 添加到链表头部 (LIFO)
netif->next = netif_list;
netif_list = netif;
```

这意味着**后添加的 netif 会在链表前面**，在 NETIF_FOREACH 中先被遍历到。

---

## 8. 性能特征

### 8.1 时间复杂度

| 操作 | 复杂度 | 说明 |
|------|--------|------|
| **netif_add** | O(n²) | 编号冲突检查，最坏遍历整个链表 |
| **netif_get_by_index** | O(n) | 线性遍历链表 |
| **NETIF_FOREACH** | O(n) | 线性遍历 |

### 8.2 netif_num 绕回

当 `netif->num == 254` 时，`netif_num` 绕回到 0。由于最大支持 255 个 netif，这个绕回是安全的。

---

## 9. 总结

### 9.1 netif_add 核心流程

```
netif_add()
    │
    ├─► 参数校验
    │
    ├─► 初始化字段 (IP、checksum、MTU)
    │
    ├─► 保存 state、input 函数
    │
    ├─► netif_set_addr() 设置 IP 地址
    │
    ├─► init() 调用 driver 初始化
    │
    ├─► 分配唯一编号 (num)
    │     └─► O(n²) 冲突检查
    │
    ├─► 添加到 netif_list 链表头部
    │
    └─► 返回 netif
```

### 9.2 netif_get_by_index 核心流程

```
netif_get_by_index(idx)
    │
    └─► NETIF_FOREACH 遍历链表
          └─► 比较 idx == netif_get_index(netif)
                └─► 找到返回 netif，否则返回 NULL
```

### 9.3 关键设计

1. **链表头插法**: 新 netif 在链表头部，遍历时先看到新的
2. **编号 vs 索引**: 内部用 num (0-254)，外部用 index (1-255)
3. **编号分配**: 避免冲突的 O(n²) 算法，最大支持 255 个 netif
4. **SafeOS 扩展**: vlanid 字段支持 VLAN 分发

### 9.4 SafeOS 特供

1. **vlanid**: VLAN 接口标识
2. **priority**: 默认优先级
3. **if_idx**: pbuf 中记录输入 netif 索引
