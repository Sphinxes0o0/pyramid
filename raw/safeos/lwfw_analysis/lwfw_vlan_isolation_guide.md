# lwIP 防火墙 VLAN 间通信限制配置指南

> 文档版本: v1.0
> 更新日期: 2026/04/22
> 代码版本: release/vsel4.01.04.04

---

## 目录

1. [VLAN 间隔离架构](#1-vlan-间隔离架构)
2. [防火墙能力分析](#2-防火墙能力分析)
3. [配置方案](#3-配置方案)
4. [完整配置示例](#4-完整配置示例)
5. [部署与验证](#5-部署与验证)
6. [局限性说明](#6-局限性说明)

---

## 1. VLAN 间隔离架构

### 1.1 网络拓扑

```
┌─────────────────────────────────────────────────────────────────────┐
│                         VLAN 间通信架构                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    VLAN 100 (业务A)        ┌─────────┐       VLAN 200 (业务B)         │
│   192.168.100.0/24      │  Router │      192.168.200.0/24          │
│                           ─┤         ├─                            │
│   192.168.100.1 ──────────┤         ├───────── 192.168.200.1        │
│                           └─────────┘                                 │
│                                │                                     │
│                         ┌──────┴──────┐                              │
│                         │   Firewall  │ ← lwfw 在此拦截              │
│                         │  (lwIP)    │                              │
│                         └────────────┘                               │
│                                                                      │
│  防火墙职责:                                                          │
│  - 允许同 VLAN 内通信                                                 │
│  - 限制跨 VLAN 通信                                                   │
│  - 记录违规流量日志                                                   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 数据包处理流程

```
数据包进入 lwIP
       │
       ▼
┌─────────────────────────────┐
│   ip4_input() Hook         │
│   lwfw_p->ops->ingress_filter()
└─────────────────────────────┘
       │
       ▼
┌─────────────────────────────┐
│ lwfw_pkt_l2_info_constructor()│
│   - 解析 Ethernet 头         │
│   - 检测 VLAN Tag (0x8100)  │
│   - 提取 VLAN_ID (12bit)    │
└─────────────────────────────┘
       │
       ▼
┌─────────────────────────────┐
│ check_lwfw_l2_info()       │
│   if (flags & VLAN)        │
│     比较 rule->l2.vlan      │
│     与 packet->l2.vlan     │
└─────────────────────────────┘
       │
       ▼
┌─────────────────────────────┐
│ check_lwfw_l3_info()       │
│   - 比较 src_ip/dst_ip     │
│   - 支持 CIDR 前缀掩码     │
└─────────────────────────────┘
       │
       ▼
┌─────────────────────────────┐
│ check_lwfw_l4_info()       │
│   - 比较 src_port/dst_port │
│   - 支持端口列表            │
└─────────────────────────────┘
       │
       ▼
   匹配规则，执行 action
```

---

## 2. 防火墙能力分析

### 2.1 支持的功能

| 功能 | 支持 | 说明 |
|------|------|------|
| VLAN 精确匹配 | ✅ | `vlan: 100` |
| VLAN + 源 IP 组合 | ✅ | `vlan: 100` + `from.L3.ipBlock` |
| VLAN + 目标 IP 组合 | ✅ | `vlan: 100` + `to.L3.ipBlock` |
| VLAN + 协议组合 | ✅ | `vlan: 100` + `protocol: tcp` |
| VLAN + 端口组合 | ✅ | `vlan: 100` + `to.L4.list: [80,443]` |
| CIDR 前缀匹配 | ✅ | `prefix: 24` (支持 0-32) |
| 端口范围列表 | ✅ | `list: [80,443,8080]` |
| 速率限制 | ✅ | `ratelimit` 字段 |
| 事件上报 | ✅ | `event: true` |
| 日志记录 | ✅ | `log: true` |

### 2.2 不支持的功能

| 功能 | 状态 | 替代方案 |
|------|------|----------|
| VLAN 范围 | ❌ | 为每个 VLAN 写单独规则 |
| VLAN 掩码 | ❌ | 使用规则组 |
| 否定匹配 | ❌ | 使用默认拒绝策略 |
| 规则继承 | ❌ | 每条规则独立配置 |
| 时间条件 | ❌ | 依赖外部定时任务 |

### 2.3 编译要求

```c
// 在 lwip_ds_mcu 或 liblwfw 配置中定义:
#define LWFW_ADVANCED_FUNC_L2 1  // 启用 L2 (VLAN/MAC) 过滤
#define NIO_LWIP_LWFW 1           // 启用 lwfw 功能
```

---

## 3. 配置方案

### 3.1 策略模式选择

#### 方案 A：白名单模式（高安全）

```yaml
spec:
  default:
    ingress:
       action: deny    # 默认拒绝
    egress:
       action: deny
```

**适用场景**: 对安全性要求高，需要显式放行每个允许的通信。

#### 方案 B：黑名单模式（默认允许）

```yaml
spec:
  default:
    ingress:
       action: allow   # 默认允许
    egress:
       action: allow
```

**适用场景**: 需要限制特定通信，其余全部放行。

### 3.2 规则优先级

规则按 `index` 从小到大逐条匹配，**首次匹配即执行**。

```
匹配顺序: index 1 → index 2 → ... → default action
```

**设计原则**:
- 精确、具体的规则用小 index（优先匹配）
- 通用、宽泛的规则用大 index
- 阻断规则放在允许规则之后（白名单模式）

---

## 4. 完整配置示例

### 4.1 企业网络隔离配置

```yaml
apiVersion: nt3.networking.firewall/v1
kind: NetworkPolicy
metadata:
  name: enterprise-vlan-isolation
  namespace: default
  version: 1
  revision: 1

spec:
  # =============================================
  # 默认策略: 拒绝所有跨 VLAN 通信
  # =============================================
  default:
    ingress:
       action: deny
       event: true
       log: true
    egress:
       action: deny
       event: true
       log: true

  # =============================================
  # 入口规则配置
  # =============================================
  ingress:
    state: enable
    rules:

    # ---------- 同 VLAN 内通信 ----------

    # 规则 1: 允许 VLAN 100 内部通信
    - index: 1
      name: allow-vlan100-internal
      state: enable
      action: allow
      vlan: 100
      from:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 16   # 192.168.x.x 内部
      to:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 16

    # 规则 2: 允许 VLAN 200 内部通信
    - index: 2
      name: allow-vlan200-internal
      state: enable
      action: allow
      vlan: 200
      from:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 16
      to:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 16

    # 规则 3: 允许 VLAN 300 内部通信
    - index: 3
      name: allow-vlan300-internal
      state: enable
      action: allow
      vlan: 300
      from:
        L3:
          ipBlock:
            ip: 192.168.300.0
            prefix: 16
      to:
        L3:
          ipBlock:
            ip: 192.168.300.0
            prefix: 16

    # ---------- 跨 VLAN 特定访问 ----------

    # 规则 10: 允许 VLAN 100 访问 VLAN 100 的 Web 服务器
    - index: 10
      name: allow-vlan100-to-web
      state: enable
      action: allow
      vlan: 100
      protocol: tcp
      from:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 24
        L4:
          range:
            portBegin: 1024
            portEnd: 65535  # 客户端随机端口
      to:
        L3:
          ipBlock:
            ip: 192.168.100.10
            prefix: 32
        L4:
          list:
            - 80
            - 443

    # 规则 11: 允许 VLAN 100 访问本 VLAN 数据库
    - index: 11
      name: allow-vlan100-to-db
      state: enable
      action: allow
      vlan: 100
      protocol: tcp
      from:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 24
        L4:
          range:
            portBegin: 1024
            portEnd: 65535
      to:
        L3:
          ipBlock:
            ip: 192.168.100.20
            prefix: 32
        L4:
          list:
            - 3306
            - 5432
            - 6379

    # ---------- 跨 VLAN 显式拒绝 ----------

    # 规则 100: 拒绝 VLAN 100 → VLAN 200
    - index: 100
      name: block-vlan100-to-vlan200
      state: enable
      action: deny
      event: true
      log: true
      vlan: 100
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 24

    # 规则 101: 拒绝 VLAN 100 → VLAN 300
    - index: 101
      name: block-vlan100-to-vlan300
      state: enable
      action: deny
      event: true
      log: true
      vlan: 100
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.300.0
            prefix: 24

    # 规则 102: 拒绝 VLAN 200 → VLAN 100
    - index: 102
      name: block-vlan200-to-vlan100
      state: enable
      action: deny
      event: true
      log: true
      vlan: 200
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 24

    # 规则 103: 拒绝 VLAN 200 → VLAN 300
    - index: 103
      name: block-vlan200-to-vlan300
      state: enable
      action: deny
      event: true
      log: true
      vlan: 200
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.300.0
            prefix: 24

    # ---------- 保护核心网段 ----------

    # 规则 200: 拒绝所有 VLAN 访问管理网段 10.0.0.0/8
    - index: 200
      name: block-vlan100-to-mgmt
      state: enable
      action: deny
      event: true
      log: true
      vlan: 100
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 10.0.0.0
            prefix: 8

    - index: 201
      name: block-vlan200-to-mgmt
      state: enable
      action: deny
      event: true
      log: true
      vlan: 200
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 10.0.0.0
            prefix: 8

    - index: 202
      name: block-vlan300-to-mgmt
      state: enable
      action: deny
      event: true
      log: true
      vlan: 300
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 10.0.0.0
            prefix: 8

    # 规则 210: 拒绝所有 VLAN 访问互联网络 (0.0.0.0/0)
    - index: 210
      name: block-vlan100-to-external
      state: enable
      action: deny
      event: true
      log: true
      vlan: 100
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 0.0.0.0
            prefix: 0

  # =============================================
  # 出口规则配置 (类似入口)
  # =============================================
  egress:
    state: enable
    rules:

    # 允许同 VLAN 出口通信
    - index: 1
      name: allow-vlan100-internal-egress
      state: enable
      action: allow
      vlan: 100
      from:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 16
      to:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 16

    - index: 2
      name: allow-vlan200-internal-egress
      state: enable
      action: allow
      vlan: 200
      from:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 16
      to:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 16

    - index: 3
      name: allow-vlan300-internal-egress
      state: enable
      action: allow
      vlan: 300
      from:
        L3:
          ipBlock:
            ip: 192.168.300.0
            prefix: 16
      to:
        L3:
          ipBlock:
            ip: 192.168.300.0
            prefix: 16

    # 拒绝跨 VLAN
    - index: 100
      name: block-vlan100-to-vlan200-egress
      state: enable
      action: deny
      event: true
      log: true
      vlan: 100
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 24

    - index: 101
      name: block-vlan200-to-vlan100-egress
      state: enable
      action: deny
      event: true
      log: true
      vlan: 200
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 24
```

### 4.2 简化配置（仅关键隔离）

```yaml
apiVersion: nt3.networking.firewall/v1
kind: NetworkPolicy
metadata:
  name: simple-vlan-isolation
  namespace: default
  version: 1
  revision: 1

spec:
  default:
    ingress:
       action: allow
       event: false
       log: false
    egress:
       action: allow
       event: false
       log: false

  ingress:
    state: enable
    rules:

    # 规则 1: 拒绝 VLAN 100 → VLAN 200
    - index: 1
      name: block-vlan100-to-vlan200
      state: enable
      action: deny
      event: true
      log: true
      vlan: 100
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 24

    # 规则 2: 拒绝 VLAN 200 → VLAN 100
    - index: 2
      name: block-vlan200-to-vlan100
      state: enable
      action: deny
      event: true
      log: true
      vlan: 200
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 24

  egress:
    state: enable
    rules:

    - index: 1
      name: block-vlan100-to-vlan200-egress
      state: enable
      action: deny
      event: true
      log: true
      vlan: 100
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.200.0
            prefix: 24

    - index: 2
      name: block-vlan200-to-vlan100-egress
      state: enable
      action: deny
      event: true
      log: true
      vlan: 200
      protocol: ip
      to:
        L3:
          ipBlock:
            ip: 192.168.100.0
            prefix: 24
```

---

## 5. 部署与验证

### 5.1 部署步骤

#### 步骤 1: 确认编译配置

```bash
# 检查是否启用了 L2 过滤
grep -r "LWFW_ADVANCED_FUNC_L2" os-framework/build/ 2>/dev/null || \
grep -r "LWFW_ADVANCED_FUNC_L2" libs/util_libs/ 2>/dev/null
```

如果未启用，需要在编译配置中添加：

```bash
# 在 CMakeLists.txt 或配置文件中添加
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DLWFW_ADVANCED_FUNC_L2=1")
```

#### 步骤 2: 上传配置文件

```bash
# 复制到目标系统的配置目录
scp enterprise-vlan-isolation.yaml root@target:/etc/lwfw/vdf_firewall_policy.yaml

# 或者通过软链接更新
ln -sf /etc/lwfw/enterprise-vlan-isolation.yaml /tmp/etc/lwfw/vdf_firewall_policy.yaml
```

#### 步骤 3: 触发热重载

```bash
# 方式 1: 通过 IOCTL
lwfwcfg reload

# 方式 2: 等待定时器线程自动检测 (默认 60 秒)
```

### 5.2 验证方法

#### 验证 1: 确认防火墙状态

```bash
# 查看防火墙状态
lwfwcfg show

# 预期输出:
# Ingress: enable, rule_count: XX
# Egress: enable, rule_count: XX
```

#### 验证 2: 测试同 VLAN 通信

```bash
# 从 VLAN 100 主机测试
ping 192.168.100.1        # 应通过
ping 192.168.100.10       # 应通过 (Web 服务器)

# 查看日志
tail -f /var/log/lwfw/lwfw-event_*.log
```

#### 验证 3: 测试跨 VLAN 通信（应被拦截）

```bash
# 从 VLAN 100 主机测试跨 VLAN
ping 192.168.200.1        # 应被拒绝

# 查看事件日志
tail -f /var/log/lwfw/lwfw-event_*.log
# 预期看到 DENY 事件
```

#### 验证 4: 检查统计数据

```bash
# 查看防火墙统计
lwfwcfg stats

# 预期看到:
# - ct_notrack: 0
# - total_event_cnt: 增加
# - drop_events: 跨 VLAN 尝试被记录
```

### 5.3 日志格式说明

```json
{
  "event_id": 1,
  "rule_id": 100,
  "action": "DENY",
  "proto": "TCP",
  "src_ip": "192.168.100.50",
  "dst_ip": "192.168.200.10",
  "src_port": 12345,
  "dst_port": 80,
  "if_name": "eth0",
  "vlan": 100,
  "count": 1,
  "timestamp": 1712345678901
}
```

---

## 6. 局限性说明

### 6.1 当前限制

| 限制项 | 说明 | 影响 |
|--------|------|------|
| VLAN 精确匹配 | 只能匹配单个 VLAN ID | 无法用一条规则匹配多个连续 VLAN |
| 无规则继承 | 每条规则需独立配置所有字段 | 规则数量可能较多 |
| 仅 Ingress L2 | Egress 方向 L2 信息未填充 | Egress 规则中 VLAN 条件无效 |
| 无时间条件 | 无法设置时间段规则 | 需依赖外部定时任务 |
| 无否定匹配 | 无法使用 `vlan != 100` | 必须使用白名单模式 |

### 6.2 注意事项

1. **编译开关**: 必须启用 `LWFW_ADVANCED_FUNC_L2` 才能使用 VLAN 过滤。

2. **热切换**: 配置更新后自动热切换，无需重启服务。

3. **规则顺序**: 规则按 `index` 小到大逐条匹配，确保允许规则在阻断规则之前。

4. **双向配置**: 需要同时配置 Ingress 和 Egress 规则才能完全隔离。

5. **性能影响**: 规则数量过多（>100）时，建议启用 Tree 搜索模式。

### 6.3 未来优化方向

- [ ] 支持 VLAN 范围语法 `vlan: [100-200]`
- [ ] 支持 VLAN 组定义
- [ ] 支持规则模板/继承
- [ ] 支持 Egress 方向的 VLAN 检查
- [ ] 支持时间条件规则

---

## 附录 A: YAML 字段参考

### A.1 顶层字段

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `index` | integer | 是 | 规则索引 (1-65535) |
| `name` | string | 否 | 规则名称 |
| `state` | string | 是 | `enable` / `disable` |
| `action` | string | 是 | `allow` / `deny` |
| `vlan` | integer | 否 | VLAN ID (1-65535) |
| `protocol` | string | 否 | `ip` / `tcp` / `udp` / `icmp` |
| `event` | boolean | 否 | 是否上报事件 |
| `log` | boolean | 否 | 是否记录日志 |

### A.2 L3 嵌套字段

```yaml
from:
  L3:
    ipBlock:
      ip: 192.168.100.0   # IP 地址
      prefix: 24           # CIDR 前缀 (0-32)
to:
  L3:
    ipBlock:
      ip: 192.168.200.0
      prefix: 24
```

### A.3 L4 嵌套字段

```yaml
from:
  L4:
    range:
      portBegin: 1024
      portEnd: 65535
    # 或
    list:
      - 80
      - 443
to:
  L4:
    range:
      portBegin: 80
      portEnd: 80
    # 或
    list:
      - 3306
      - 5432
```

---

## 附录 B: 快速参考

### B.1 常用命令

```bash
# 查看防火墙状态
lwfwcfg show

# 查看统计信息
lwfwcfg stats

# 重载配置
lwfwcfg reload

# 查看日志
tail -f /var/log/lwfw/lwfw-event_*.log

# 刷新统计
lwfwcfg reset
```

### B.2 默认路径

| 文件 | 路径 |
|------|------|
| 配置文件 | `/etc/lwfw/vdf_firewall_policy.yaml` |
| 软链接 | `/tmp/etc/lwfw/vdf_firewall_policy.yaml` |
| 事件日志 | `/var/log/lwfw/lwfw-event_*.log` |
| CLI 工具 | `lwfwcfg` |

---

## 附录 C: 相关文档

| 文档 | 说明 |
|------|------|
| [lwfw_core_filtering.md](lwfw_core_filtering.md) | 核心过滤逻辑分析 |
| [lwfw_parser.md](lwfw_parser.md) | YAML 规则解析器 |
| [lwfw_data_structure.md](lwfw_data_structure.md) | 数据结构设计 |
| [lwfw_optimization.md](lwfw_optimization.md) | 优化建议汇总 |
| [lwfw_hook_injection.md](lwfw_hook_injection.md) | Hook 注入点 |
