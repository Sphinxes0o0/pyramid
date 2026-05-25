---
type: entity
tags: [linux, lwip, network, netif]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# lwIP netif Structure

## 定义

`struct netif` 是 lwIP 的**网络接口抽象**，代表一个物理或虚拟网络设备。它串联在 `netif_list` 链表中，通过 `next` 指针组织。

## 关键字段

```c
struct netif {
    struct netif *next;           // 链表指针

    // IP 配置
    ip_addr_t ip_addr;            // IP 地址
    ip_addr_t netmask;            // 子网掩码
    ip_addr_t gw;                 // 默认网关

    // 回调函数 (核心!)
    netif_input_fn input;         // packet 输入函数
    netif_output_fn output;      // IP 输出函数 (etharp_output)
    netif_linkoutput_fn linkoutput;  // 链路层输出 (ethif_link_output)

    // 硬件信息
    u8_t hwaddr[NETIF_MAX_HWADDR_LEN];  // MAC 地址
    u8_t hwaddr_len;             // MAC 地址长度 (通常 6)
    u16_t mtu;                   // 最大传输单元 (通常 1500)

    // 标识
    char name[2];                // 接口名缩写 (如 "et", "vl")
    u8_t num;                    // 接口编号 (0-254)
    u8_t flags;                  // NETIF_FLAG_*

    // VLAN 支持 (SafeOS)
    u16_t vlanid;               // VLAN ID (0 = 无 VLAN)

    void *state;                // driver 特定状态
    void *client_data[];        // 客户端数据扩展
};
```

## SafeOS 中的 netif 实例

### vnet_if (物理网口)
- `name = "et0"`, `vlanid = 0`, `input = ethernet_input`
- `output = etharp_output`, `linkoutput = ethif_link_output`

### vlan_if[i] (VLAN 网口)
- `name = "vl0"`, `vlanid = 100/200`, `input = tcpip_input`
- `output = etharp_output`, `linkoutput = low_level_output` → ethif_link_output

## 链表结构

```
netif_list (链表头)
      │
      ▼
┌─────────────┐    next     ┌─────────────┐    next     ┌─────────────┐
│   vnet_if   │ ─────────► │  vlan_if[0] │ ─────────► │  vlan_if[1] │
│ (物理网口)   │            │  (VLAN 100) │            │  (VLAN 200) │
└─────────────┘            └─────────────┘            └─────────────┘
```

## 关键设计

1. **input 回调**: packet 进入协议栈的入口
2. **linkoutput 回调**: packet 发送到链路的出口
3. **vlanid 字段**: lwIP 的 VLAN 支持，通过此字段区分 VLAN
4. **链表头插法**: 新 netif 在链表头部，遍历时先被看到

## 相关概念

- [[entities/linux/lwip/lwip-netif-add]] — netif 注册流程
- [[entities/linux/lwip/lwip-ethernet-input]] — input 回调的调用方
- [[entities/linux/lwip/lwip-ethernet-output]] — linkoutput 回调的调用方
- [[entities/linux/lwip/lwip-vlan-implementation]] — vlanid 的使用方式
