---
type: entity
tags: [lwip, ipv4, source, network-layer]
created: 2026-05-25
sources: [safeos-lwip-core]
---

# ip4.c — IPv4 Input/Output

> IPv4 层实现：接收 (ip4_input) 和发送 (ip4_output/ip4_output_if)

## 文件概览

| 属性 | 值 |
|------|-----|
| 路径 | `src/core/ipv4/ip4.c` |
| 行数 | 1307 |
| 功能 | IPv4 包的输入处理、输出封装、路由查找、转发 |
| 依赖 | pbuf, netif, tcp, udp, icmp, igmp, raw |

## 函数索引

| 函数 | 行号 | 功能 |
|------|------|------|
| `ip4_route` | 166 | 查找目标地址对应的网络接口 (netif) |
| `ip4_route_src` | 143 | 带源地址的 IPv4 路由查找 (hook wrapper) |
| `ip4_canforward` | 252 | 判断 IP 地址是否可转发 (广播/组播/实验地址过滤) |
| `ip4_forward` | 295 | 执行 IP 包转发：TTL 递减、checksum 更新、output 调用 |
| `ip4_input_accept` | 396 | 判断收上的包是否匹配本 netif (单播/广播/loopback) |
| `ip4_input` | 468 | IPv4 输入主函数：header 解析 → checksum → netif 匹配 → 分片重组 → 协议分发 |
| `ip4_output_if` | 888 | 在指定 netif 上发送 IP 包 (构造 header + checksum) |
| `ip4_output_if_opt` | 903 | ip4_output_if + IP options 支持 |
| `ip4_output_if_src` | 928 | ip4_output_if 但保留 src 地址不替换 |
| `ip4_output_if_opt_src` | 941 | ip4_output_if_opt + src 保留 |
| `ip4_output` | 1194 | 通用 IPv4 输出：自动路由查找后调用 ip4_output_if |
| `ip4_output_hinted` | 1241 | ip4_output + netif_hint 支持 |
| `ip4_debug_print` | 1269 | 打印 IP header 调试信息 |
| `ip4_set_default_multicast_netif` | 131 | 设置默认组播 netif |

## 关键数据结构

### struct ip_hdr (lwip/ip.h)
```
IP header (20 bytes minimum):
  - _v_hl:  version(4) + header_len(4)
  - _tos:   type of service
  - _len:   total length
  - _id:    identification
  - _offset: flags(3) + fragment_offset(13)
  - _ttl:   time to live
  - _proto: protocol
  - _chksum: header checksum
  - src:    source IP address
  - dest:   destination IP address
```

## 调用链

### 输入 (ip4_input)
```
ethernet_input / ip_input
  → ip4_input
    → ip4_route (multicast/broadcast case)
    → ip4_forward (非本机包)
    → pbuf_remove_header
    → [LWFW ingress_filter hook]
    → tcp_input / udp_input / icmp_input / igmp_input / raw_input
```

### 输出 (ip4_output)
```
tcp_output / udp_send / raw_send
  → ip4_output / ip4_output_if
    → ip4_route
    → pbuf_add_header (IP header space)
    → [LWFW egress_filter hook]
    → netif->output (ethernet_output / netif_loop_output)
    → ip4_frag (if MTU exceeded)
```

## 交叉引用

### Analysis 层
- [[entities/linux/lwip/lwip-ip4-input]] — ip4_input 详细分析
- [[entities/linux/lwip/lwip-ip4-output]] — ip4_output 详细分析
- [[entities/linux/lwip/lwip-routing]] — 路由机制
- [[entities/linux/lwip/lwip-ip-fragmentation]] — 分片重组

### 上层协议
- [[entities/linux/lwip/lwip-tcp-input]] — TCP 输入
- [[entities/linux/lwip/lwip-udp-input]] — UDP 输入

### 下层接口
- [[entities/linux/lwip/lwip-netif]] — netif 结构
- [[entities/linux/lwip/lwip-pbuf]] — pbuf 管理

### 安全集成
- [[entities/linux/lwip/lwip-firewall]] — LWFW 防火墙
- [[entities/linux/lwip/lwip-lwfw-filter-hooks]] — Ingress/Egress hooks
