---
type: synthesis
tags: [计算机网络, 课程, TCPIP, UDP, HTTP, DNS, 网络安全]
created: 2026-05-20
sources: [notes-overview-network_fundamentals]
---

# 计算机网络基础 (Network Fundamentals)

## 背景

计算机网络基础课程笔记，涵盖网络分层模型、TCP/UDP协议、IP协议、Socket编程、HTTP、DNS、网络安全等核心主题。

## 核心概念

### 网络分层模型 (TCP/IP五层)

| 层级 | 职责 | 协议示例 |
|------|------|----------|
| 应用层 | 为用户提供网络服务 | HTTP, DNS, FTP |
| 传输层 | 端到端通信，可靠性 | TCP, UDP |
| 网络层 | 主机到主机路由，寻址 | IP, ICMP |
| 数据链路层 | 链路级别传输，帧封装 | Ethernet, ARP |
| 物理层 | 二进制光电传输 | 光纤, 电缆 |

### 多路复用 (Multiplexing)

**传输层多路复用**: 多个TCP连接复用一条线路（如HTTP keep-alive）
**网络层多路复用**: 多个传输层连接共用底层的物理网络

优势: 提升吞吐量，信号间隔离不互相阻塞

### TCP协议

**三次握手**:
1. Client→Server: SYN (seq=x)
2. Server→Client: SYN-ACK (seq=y, ack=x+1)
3. Client→Server: ACK (ack=y+1)

**四次挥手**:
1. Client→Server: FIN
2. Server→Client: ACK
3. Server→Client: FIN
4. Client→Server: ACK

**为什么握手3次？**: 双方都需要确认对方的发送和接收能力都正常
**为什么挥手4次？**: TCP是全双工，服务端收到FIN后先发送ACK，但需要等上层应用确认后才能发送FIN

**滑动窗口**: 流量控制，接收方通告窗口大小，发送方控制发送速率
**拥塞控制**: 慢启动→拥塞避免→快速重传→快速恢复

**粘包/拆包**: TCP是流式协议，应用程序需定义消息边界（长度前缀/分隔符）

### UDP协议

**特点**: 无连接、低延迟、不保证可靠交付
**优势**: 无需握手，比TCP快；适合实时性要求高的场景（直播、游戏、语音）
**劣势**: 无拥塞控制、可能丢包乱序

### IP协议

**IPv4**: 32位地址，分四段 (如 192.168.1.1)
**IPv6**: 128位地址，简化头部，支持自动配置
**IPv6 Tunnel**: 6to4、Teredo等隧道技术穿越IPv4网络

**寻址 vs 路由**:
- 寻址 (Addressing): 类似导航，告诉下一个目的地方向
- 路由 (Routing): 根据目的地选择具体路径

**子网掩码**: 定义网络号和主机号边界（如 255.255.255.0）

### Socket编程

**Socket本质**: 双向管道文件，不是纯内存结构
- 服务端Socket文件存储客户端Socket的文件描述符
- 客户端Socket文件存储实际传输数据

**I/O多路复用模型**:
- **BIO (Blocking I/O)**: 阻塞等待
- **NIO (Non-blocking I/O)**: 主动轮询，CPU空转
- **AIO (Async I/O)**: 事件驱动回调

**epoll为什么用红黑树**:
- 插入/删除/查找: O(log n)
- 适合百万级并发连接管理
- select/poll是O(n)遍历

### HTTP协议

**强制缓存**: Cache-Control, Expires
**协商缓存**: Last-Modified/If-Modified-Since, ETag/If-None-Match

**缓存流程**:
1. 检查强制缓存是否有效 → 直接返回
2. 发送协商缓存请求 → 服务端判断是否变化
3. 变化返回新内容，否则304 Not Modified

### DNS

**记录类型**:
- A记录: 域名→IPv4
- AAAA: 域名→IPv6
- CNAME: 域名别名
- NS: 域名服务器

**解析过程**: 浏览器缓存→系统缓存→本地域名服务器→根服务器→顶级域服务器→权威服务器

### CDN (Content Delivery Network)

**回源**: 用户请求→CDN边缘节点→无缓存→回源站获取→缓存→返回用户

### 网络安全

**对称加密**: AES, DES — 加密解密用同一密钥，速度快
**非对称加密**: RSA, ECC — 公钥加密/私钥解密，慢但安全
**混合加密**: 非对称加密传输对称密钥，实际数据用对称加密

**中间人攻击**: 攻击者插入自己，在双方间转发并可能篡改数据
**TLS/SSL**: 非对称加密建立会话，对称加密传输数据，证书验证身份

**对称 vs 非对称**:
- 对称: 速度快，密钥分发难
- 非对称: 速度慢，密钥分发简单

### 流媒体技术

**直播架构**: 采集→编码→推流(CDN)→拉流→解码→播放
**协议**: RTMP (基于TCP), HLS (HTTP流), WebRTC (实时)

### 爬虫与反爬虫

**常用反爬**: IP限流、验证码、User-Agent检测、Cookie追踪、JavaScript渲染

## 关键引用

### TCP三次握手原因
> 双方都需要确认对方的发送能力和接收能力都正常。Client发SYN说"我要说话"，Server回SYN-ACK说"我能听到你也能说"，Client再ACK说"我知道你能听到我"。

### epoll vs select/poll
> select是主动模型，线程自己去操作系统查看哪些fd有I/O事件；epoll在操作系统内核中提供中间数据结构（红黑树），线程被动等待通知。高并发下epoll更快。

### UDP适用场景
> TCP像打电话（建立连接，确认收到），UDP像发快递（不保证按时送达）。实时性要求高、可容忍少量丢包的场景（游戏、直播、语音）适合UDP。

## 相关页面
- linux-kernel-net-core - Linux网络核心
- linux-kernel-netfilter - Linux防火墙/Netfilter
- [[synthesis/topic-os-fundamentals]] - 操作系统基础
