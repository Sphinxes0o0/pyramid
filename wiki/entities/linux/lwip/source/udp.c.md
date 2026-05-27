---
type: entity
tags: [lwip, udp, source, transport-layer]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# udp.c — UDP Input/Output

> UDP：输入分发 (udp_input)、输出发送 (udp_send)、PCB 管理

## 文件概览

| 属性 | 值 |
|------|-----|
| 路径 | `src/core/udp.c` |
| 行数 | 1385 |
| 功能 | UDP datagram 分发、发送、PCB 管理 |
| 依赖 | pbuf, netif, ip, inet_chksum |

## 函数索引

### 输入
| 函数 | 行号 | 功能 |
|------|------|------|
| `udp_input` | 213 | UDP 输入主函数：header 解析 → PCB 匹配 → recv callback |
| `udp_input_local_match` | 142 | 判断 PCB 是否匹配输入包 (netif/broadcast/IP) |

### 输出
| 函数 | 行号 | 功能 |
|------|------|------|
| `udp_send` | 508 | 通过已连接 PCB 发送 UDP |
| `udp_sendto` | 561 | 发送到指定地址 (自动路由) |
| `udp_sendto_chksum` | 571 | udp_sendto + checksum |
| `udp_sendto_if` | 670 | 指定 netif 发送 |
| `udp_sendto_if_chksum` | 679 | udp_sendto_if + checksum |
| `udp_sendto_if_src` | 749 | 指定源 IP 发送 |
| `udp_sendto_if_src_chksum` | 758 | udp_sendto_if_src + checksum |

### PCB 管理
| 函数 | 行号 | 功能 |
|------|------|------|
| `udp_new` | 1287 | 创建 UDP PCB |
| `udp_new_ip_type` | 1326 | 创建指定 IP 类型的 UDP PCB |
| `udp_bind` | 996 | 绑定 PCB 到 local addr:port |
| `udp_bind_netif` | 1111 | 绑定到特定 netif |
| `udp_connect` | 1140 | 设置 remote addr:port (设 CONNECTED 标志) |
| `udp_disconnect` | 1195 | 清除 remote 关联 |
| `udp_recv` | 1227 | 注册 recv 回调 |
| `udp_remove` | 1248 | 移除并释放 PCB |
| `udp_init` | 87 | 模块初始化 |
| `udp_new_port` | 100 | 分配新本地端口 |
| `udp_netif_ip_addr_changed` | 1349 | netif 地址变更时更新 PCB |

### 调试
| 函数 | 行号 | 功能 |
|------|------|------|
| `udp_debug_print` | 1372 | 打印 UDP header |

## 关键数据结构

### struct udp_hdr (lwip/udp.h)
```
UDP header (8 bytes):
  - src:    source port (2 bytes, network order)
  - dest:   destination port (2 bytes)
  - len:    UDP length (2 bytes)
  - chksum: checksum (2 bytes, 0 = no checksum)
```

### udp_pcb (核心字段)
```c
struct udp_pcb {
  struct udp_pcb *next;     // 链表指针
  ip_addr_t local_ip;       // 绑定本地 IP (any = 0.0.0.0)
  u16_t local_port;          // 绑定本地端口
  ip_addr_t remote_ip;       // 远端 IP (connect 设置)
  u16_t remote_port;         // 远端端口
  u8_t flags;               // UDP_FLAGS_*
  udp_recv_fn recv;          // 接收回调
  void *recv_arg;           // 回调参数
  // ... netif_idx, mcast 相关
};
```

### 全局变量
```c
struct udp_pcb *udp_pcbs;   // 所有 UDP PCB 链表
static u16_t udp_port;      // 下一个可用本地端口 (49152-65535)
```

## 调用链

### UDP 输入 (udp_input)
```
ip4_input (IP_PROTO_UDP)
  → udp_input
    → 检查 UDP_HLEN
    → pbuf_remove_header (去掉 UDP header)
    → 遍历 udp_pcbs 链表匹配 (local_port + local_ip + netif_idx)
      → 优先完全匹配 (local+remote)
      → 其次 unconnected PCB
    → [匹配且有 recv] → pcb->recv()
    → [无匹配] → icmp_port_unreach()
    → pbuf_free()
```

### UDP 输出 (udp_sendto_if_src)
```
udp_send()
  → udp_sendto()
    → udp_sendto_if()
      → udp_sendto_if_src()
        → ip_route() / netif_get_by_index()  // 路由查找
        → pbuf_add_header(UDP_HLEN)         // 添加 UDP header
        → udphdr->src = pcb->local_port
        → udphdr->dest = dst_port
        → udphdr->chksum = (计算 pseudo-header checksum)
        → ip_output_if_src()                 // 调用 IP 层
```

## 交叉引用

### Analysis 层
- [[entities/linux/lwip/lwip-udp-input]] — UDP 输入详解
- [[entities/linux/lwip/lwip-udp-output]] — UDP 输出详解
- [[entities/linux/lwip/lwip-udp-socket]] — UDP Socket API

### 依赖层
- [[entities/linux/lwip/lwip-pbuf]] — pbuf 管理
- [[entities/linux/lwip/lwip-netif]] — netif/路由

### IP 层
- [[entities/linux/lwip/source/ip4.c]] — IP 输出 (ip_output_if_src)
