# UDP Socket 分析 — T-042

> 文档版本: 1.0
> 创建日期: 2026/04/22
> 分析目标: UDP PCB 管理、bind/connect 流程、socket 匹配

---

## 1. 概述

UDP 是无连接的协议，但 lwIP 仍然使用 PCB (Protocol Control Block) 来管理 socket 状态：

1. **udp_pcb**: 保存 local/remote 地址和端口
2. **udp_pcbs**: 全局 PCB 链表，用于接收匹配
3. **无连接状态**: UDP 没有真正的"连接"概念

---

## 2. UDP PCB 结构

**文件**: `include/lwip/udp.h:82-123`

```c
struct udp_pcb {
    IP_PCB;                    // IP 地址 + port

    struct udp_pcb *next;     // 链表指针
    u8_t flags;               // UDP_FLAGS_*

    // ports are in host byte order
    u16_t local_port;         // 本地端口
    u16_t remote_port;        // 远端端口

    // 用于 netstat
    ip_addr_t netstat_remote_ip;
    u16_t netstat_remote_port;

#if LWIP_MULTICAST_TX_OPTIONS
    ip4_addr_t mcast_ip4;     // 多播接口 IP
    u8_t mcast_ifindex;       // 多播接口索引
    u8_t mcast_ttl;           // 多播 TTL
#endif

#if LWIP_IGMP
    ip4_addr_t mcast_group[LWIP_MAX_NUM_MCAST_GROUP];  // 加入的多播组
#endif

    // 回调函数
    udp_recv_fn recv;         // 接收回调
    void *recv_arg;          // 回调参数
};
```

### 2.1 UDP Flags

```c
#define UDP_FLAGS_NONE           0x00
#define UDP_FLAGS_UDPLITE        0x02  // UDPLite 协议
#define UDP_FLAGS_NOCHKSUM      0x04  // 不计算 checksum
#define UDP_FLAGS_MULTICAST_LOOP 0x08  // 多播回环
#define UDP_FLAGS_CONNECTED      0x10  // 已连接 (connect() 调用过)
```

---

## 3. UDP PCB 链表

**文件**: `core/udp.c:81`

```c
struct udp_pcb *udp_pcbs;  // 全局 UDP PCB 链表
```

### 3.1 链表操作

```c
// 注册 PCB 到链表
pcb->next = udp_pcbs;
udp_pcbs = pcb;

// 从链表移除
TCP_RMV(&udp_pcbs, pcb);
```

---

## 4. udp_new — 创建 PCB

**文件**: `core/udp.c:1287`

```c
struct udp_pcb *udp_new(void)
{
    struct udp_pcb *pcb;

    pcb = (struct udp_pcb *)memp_malloc(MEMP_UDP_PCB);
    if (pcb != NULL) {
        memset(pcb, 0, sizeof(struct udp_pcb));
        pcb->local_port = 0;  // 未绑定
        pcb->remote_port = 0;
        ip_addr_set_any(IP_VERSION, &pcb->local_ip);
        ip_addr_set_any(IP_VERSION, &pcb->remote_ip);
    }
    return pcb;
}
```

---

## 5. udp_bind — 绑定地址端口

**文件**: `core/udp.c:996-1095`

```c
err_t udp_bind(struct udp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port)
{
    struct udp_pcb *ipcb;

    // ============================================
    // Step 1: 参数处理
    // ============================================
    if (ipaddr == NULL) {
        ipaddr = IP4_ADDR_ANY;  // IPv4 ANY
    }

    // ============================================
    // Step 2: 检查是否已绑定
    // ============================================
    if (pcb->local_port != 0) {
        return ERR_VAL;  // 已绑定，不能再次 bind
    }

    // ============================================
    // Step 3: 端口分配
    // ============================================
    if (port == 0) {
        port = udp_new_port();  // 自动分配端口
        if (port == 0) {
            return ERR_USE;  // 没有可用端口
        }
    }

    // ============================================
    // Step 4: 检查端口冲突
    // ============================================
    for (ipcb = udp_pcbs; ipcb != NULL; ipcb = ipcb->next) {
        if (ipcb->local_port == port &&
            (ip_addr_cmp(&ipcb->local_ip, ipaddr) ||
             ip_addr_isany(ipaddr) ||
             ip_addr_isany(&ipcb->local_ip))) {
            // 端口已被占用 (除非设置了 SO_REUSEADDR)
            return ERR_USE;
        }
    }

    // ============================================
    // Step 5: 设置地址和端口
    // ============================================
    ip_addr_copy(pcb->local_ip, *ipaddr);
    pcb->local_port = port;

    // ============================================
    // Step 6: 注册到链表
    // ============================================
    pcb->next = udp_pcbs;
    udp_pcbs = pcb;

    return ERR_OK;
}
```

---

## 6. udp_connect — 连接 (设置远端)

**文件**: `core/udp.c:1140-1185`

```c
err_t udp_connect(struct udp_pcb *pcb, const ip_addr_t *ipaddr, u16_t port)
{
    // ============================================
    // Step 1: 如果未绑定，先绑定
    // ============================================
    if (pcb->local_port == 0) {
        err_t err = udp_bind(pcb, &pcb->local_ip, pcb->local_port);
        if (err != ERR_OK) {
            return err;
        }
    }

    // ============================================
    // Step 2: 设置远端地址和端口
    // ============================================
    ip_addr_set(&pcb->remote_ip, ipaddr);
    pcb->remote_port = port;

    // ============================================
    // Step 3: 设置 CONNECTED 标志
    // ============================================
    pcb->flags |= UDP_FLAGS_CONNECTED;

    // ============================================
    // Step 4: 如果 PCB 不在链表中，加入
    // ============================================
    for (ipcb = udp_pcbs; ipcb != NULL; ipcb = ipcb->next) {
        if (pcb == ipcb) {
            return ERR_OK;  // 已在链表中
        }
    }
    pcb->next = udp_pcbs;
    udp_pcbs = pcb;

    return ERR_OK;
}
```

### 6.1 UDP connect vs TCP connect

| 特性 | UDP connect | TCP connect |
|------|-------------|-------------|
| **含义** | 设置远端地址/端口 | 建立连接 (3次握手) |
| **状态** | 只设置 pcb->remote_ip/port | PCB 状态变为 SYN_SENT |
| **网络操作** | 无 | 发送 SYN |
| **CONNECTED 标志** | 设置 | 不设置 |

---

## 7. udp_disconnect — 断开连接

**文件**: `core/udp.c:1195-1205`

```c
void udp_disconnect(struct udd_pcb *pcb)
{
    // 清除远端地址和 CONNECTED 标志
    ip_addr_set_any(IP_VERSION, &pcb->remote_ip);
    pcb->remote_port = 0;
    pcb->flags &= ~UDP_FLAGS_CONNECTED;

    // 注意: 不从链表中移除！
    // UDP socket 仍然可以接收数据
}
```

---

## 8. Socket 匹配 (udp_input)

**文件**: `core/udp.c:200-340`

```c
// udp_input 中遍历 udp_pcbs 链表
for (pcb = udp_pcbs; pcb != NULL; pcb = pcb->next) {
    // 检查本地端口
    if (pcb->local_port != dest) {
        continue;
    }

    // 检查本地 IP
    if (pcb->local_ip != IP_ANY &&
        ip_current_dest_addr() != pcb->local_ip) {
        continue;
    }

    // 检查广播/多播
    if (ip_addr_ismulticast(ip_current_dest_addr())) {
        // 多播匹配
    }

    // 检查远端 (如果已设置)
    if (pcb->flags & UDP_FLAGS_CONNECTED) {
        // 检查远端 IP 和端口
        if (pcb->remote_ip != src_ip ||
            pcb->remote_port != src_port) {
            continue;
        }
    }

    // 找到匹配的 PCB
    return pcb;
}
```

### 8.1 匹配优先级

```
1. local_port 精确匹配
2. local_ip 匹配 (IP_ANY 匹配任何)
3. 远端地址匹配 (仅 CONNECTED socket)
4. 广播/多播处理
```

---

## 9. udp_send — 发送数据

**文件**: `core/udp.c:508-519`

```c
err_t udp_send(struct udp_pcb *pcb, struct pbuf *p)
{
    // 如果未连接，使用 NULL 远端地址
    if (pcb->flags & UDP_FLAGS_CONNECTED) {
        return udp_sendto(pcb, p, &pcb->remote_ip, pcb->remote_port);
    }
    return ERR_VAL;  // 未连接不能直接 send
}
```

---

## 10. 总结

### 10.1 UDP Socket 流程

```
socket()                    // udp_new()
    │                         pcb = memp_malloc(MEMP_UDP_PCB)
    ▼                         pcb->local_port = 0
bind(addr, port)             // udp_bind()
    │                         ├─ 分配端口 (如果 port=0)
    │                         └─ 加入 udp_pcbs 链表
    ▼
connect(remote, port)       // udp_connect()
    │                         ├─ 如果未 bind，先 bind
    │                         └─ 设置 remote_ip/port
    │                           └─ 设置 UDP_FLAGS_CONNECTED
    ▼
sendto(data)                // udp_sendto()
    │                         ├─ 查找路由 → netif
    │                         ├─ 添加 UDP Header
    │                         ├─ 计算 Checksum
    │                         └─ ip_output_if()
    ▼
recvfrom()                  // recv 回调
                              └─ 遍历 udp_pcbs 匹配
```

### 10.2 关键设计

1. **无连接**: UDP 没有真正的连接状态
2. **PCB 链表**: `udp_pcbs` 用于接收匹配
3. **CONNECTED 标志**: 区分已连接和未连接的 socket
4. **端口自动分配**: `port=0` 时自动分配

### 10.3 与 TCP 对比

| 特性 | UDP | TCP |
|------|-----|-----|
| **PCB 类型** | `udp_pcb` | `tcp_pcb` |
| **连接状态** | 无 | 有 (SYN_SENT 等) |
| **bind** | 可选 | 可选 |
| **connect** | 设置远端地址 | 发起 3次握手 |
| **send** | 需要 connect 或 sendto | 需要连接建立 |
| **listen/accept** | 无 | 有 |
