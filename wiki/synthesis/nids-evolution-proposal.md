---
type: synthesis
tags: [nids, roadmap, strategy, review]
created: 2026-05-25
sources: [nids-current-architecture, nids-gap-analysis-roadmap, dps-market-research]
---

# NIDS 演进路线图 — 内部评审稿

> 状态：待评审 | 日期：2026/05/25 | 版本：v0.1

---

## 1. 执行摘要

**我们要做什么**：将轻量级 L3/L4 IDS 演进为支持应用层 DPI、规则兼容性 >90%、吞吐量 10Gbps+ 的嵌入式 NIDS/NDR 平台，分三阶段交付。

**为什么**：当前产品规则兼容性仅 60-70%，缺少 flowbits/PCRE/EVE JSON 等核心能力，在应用层攻击检测和 SIEM 对接上存在显著差距，无法进入主流市场。

**预期收益**：Phase 1 结束后解锁 SSH 暴力破解多步检测、Web 攻击检测、ELK/SIEM 标准对接；Phase 2 达成 10Gbps+ 吞吐，进入数据中心/云原生市场；Phase 3 演进为 AI 驱动的 NDR 平台，形成差异化竞争壁垒。

| | 当前定位 | 目标定位 |
|---|---|---|
| **能力聚焦** | L3/L4 边界检测 | L3-L7 深度检测 + NDR |
| **规则兼容** | Snort3 ~60% | Snort3 >90% → ET Open 对接 |
| **性能** | 单网卡 100Mbps | 10Gbps+（Phase 2）|
| **日志** | SOA JSON + NLog（私有）| EVE JSON（标准）+ SIEM 直连 |
| **差异化** | 嵌入式友好/零堆分配 | 嵌入式 + 车规安全网关 |

---

## 2. 市场与竞品格局

### 2.1 开源三强矩阵

| 维度 | Snort 3 | Suricata | Zeek |
|------|---------|----------|------|
| **规则生态** | Talos（50000+规则，订阅）| ET Open（35000+，免费）+ OISF | 社区 ID（无官方规则）|
| **协议解析** | L3-L7 + Lua 扩展 | L3-L7 + Lua 扩展 | 事件驱动，30+ 分析器 |
| **规则语言** | Snort3 + LuaJIT + flowbits + PCRE | Snort 兼容 + flowbits + PCRE | Zeek 脚本（图灵完备）|
| **模式匹配** | AC + Hyperscan | AC + Hyperscan | AC（插件）|
| **分片重组** | stream5 | stream/reassembly | 会话追踪 |
| **TLS 检测** | 需 Lua | JA3/JA4 原生 | JA3/JA4 原生 |
| **日志格式** | Unified2 + JSON | **EVE JSON（行业标准）**| ASCII/JSON/TSV |
| **集群/HA** | Active-Passive（基础）| 外部（Redis/Kafka）| 原生 Manager/Worker/Proxy |
| **云原生** | 手动 | 手动 | 手动 |
| **ML 能力** | libml（HTTP 参数）| 无 | 外部集成 |
| **许可证** | GPLv2 | GPLv2 | BSD |

### 2.2 我们的差异化空间

```
竞品空白：
├── Suricata/Zeek：通用高性能，但嵌入式不友好（动态分配、内存占用大）
├── Snort3：规则最全，但 Lua/插件体系复杂，嵌入式成本高
└── 商业 NGFW（Palo Alto/Fortinet）：车规安全网关场景价格极高

我们的机会窗口：
① 嵌入式实时：零堆分配运行态、预分配 PacketPool、SPSC 无锁队列
② 车规安全网关：耐高温/抗振动/低功耗定制，与安全网关深度集成
③ 轻量级边界检测：相比 Suricata 10x 内存占用，聚焦网关边界
```

**核心差异化定位**：嵌入式 + 车规场景的高可靠 NIDS，与 Suricata/Snort 的通用市场形成互补。

---

## 3. 演进路线图

### Phase 1 — 快速补缺（0-3 个月）

> **目标**：消除最高优先级功能差距，解锁有状态检测 + 标准 SIEM 对接

#### 功能明细

| 功能 | 目标 | 输入 | 输出 | 验收标准 |
|------|------|------|------|---------|
| **flowbits 状态机** | 解锁有状态检测规则 | Snort `flowbits:set,foo; flowbits:isset,foo` 语法 | 位图状态表 + 规则条件求值 | SSH 暴力破解多步检测规则可运行 |
| **PCRE 正则引擎** | 规则覆盖率 60% → 90% | Snort `pcre:"/pattern/flags"` 关键字 | PCRE2/Oniguruma 匹配结果 | ET Open HTTP 攻击规则集通过率 >90% |
| **HTTP 基础解析** | Web 攻击检测 | HTTP/1.1 request line + header | `http_method`/`http_uri`/`http_stat_code` 字段 | SQLi/XSS 工具指纹规则可运行 |
| **JA3 TLS 指纹** | 恶意加密流量检测 | TLS ClientHello | `ja3_hash`（MD5）写入事件字段 | Suricata EVE JA3 规则可迁移 |
| **EVE JSON 日志** | 标准 SIEM 对接 | SecurityEvent | `eve.json`（alert/fileinfo/stats/dns/log）| ELK Stack 原生接入，无需解析器 |

#### Phase 1 里程碑

| 指标 | 当前 | Phase 1 目标 |
|------|------|-------------|
| Snort 规则兼容度 | ~60% | **>90%** |
| 有状态规则 | 0 | **SSH 暴力破解检测** |
| SIEM 对接 | 私有 SOA JSON | **EVE JSON（标准）** |
| TLS 检测 | 无 | **JA3 指纹** |

---

### Phase 2 — 架构升级（4-9 个月）

> **目标**：吞吐量 10Gbps+，支持水平扩展，协议覆盖扩展至 DNS/SMB/HTTP

#### 功能明细

| 功能 | 目标 | 输入 | 输出 | 验收标准 |
|------|------|------|------|---------|
| **Hyperscan 模式匹配** | 规则集容量 10x+，PCRE 硬件加速 | Snort 规则集（50000+）| 编译后 Hyperscan 数据库 | 10K 规则 AC vs HS 性能对比：HS 3-5x 提升 |
| **AF_XDP / DPDK 捕获** | 10Gbps → 100Gbps | AF_XDP 零拷贝路径（Linux 5.4+）| DPDK PMD 选项（数据中心）| 10Gbps 64B 小包线速（<10% CPU 空闲）|
| **IP 分片重组** | 检测分片逃避攻击 | IPv4/IPv6 分片队列 | 重组后完整报文 | Teardrop/fraggle 攻击检测规则可运行 |
| **HA 集群方案** | 多节点水平扩展 | Manager/Worker 架构设计 | 主动-主动或主动-被动 | 故障切换 <1s，支持 2+ 节点 |
| **DNS 深度解析** | DNS 隧道/放大攻击检测 | DNS query/response | `dns_query`/`dns_response` 字段 | DNS 隧道检测规则可运行 |
| **SMB/FTP 解析** | 勒索软件传播路径检测 | SMB1/2/3 + FTP 命令 | 协议事件 + 告警 | 勒索软件 SMB 传播规则可运行 |
| **二进制规则格式增强** | `.nrb` 支持 flowbits/PCRE 元数据 | 扩展 `.nrb` schema | 编译时完整性校验 | Snort `.rules` → `.nrb` 转换工具可用 |

#### Phase 2 里程碑

| 指标 | Phase 1 | Phase 2 目标 |
|------|---------|-------------|
| 吞吐量 | 100Mbps | **10Gbps+** |
| 规则兼容度 | >90% | **>95%** |
| 协议覆盖 | L3/L4 | **DNS/SMB/FTP/HTTP** |
| 集群 | 无 | **HA 支持（2+ 节点）** |
| 分片重组 | 无 | **IPv4/IPv6 重组** |

---

### Phase 3 — 差异化能力（10-21 个月）

> **目标**：从 IDS 演进为 AI 驱动的 NDR 平台，建立云原生 + 威胁情报生态

#### 功能明细

| 功能 | 目标 | 输入 | 输出 | 验收标准 |
|------|------|------|------|---------|
| **ML 异常检测** | 检测 0-day 攻击 | HTTP 参数/DNS 查询序列 | 异常分数 + 告警 | 参考 Snort libml，HTTP 参数统计分类上线 |
| **文件提取 + SHA256** | 恶意软件检测 | 文件 carve + magic 识别 | SHA256 hash + 文件存储 | Suricata 文件提取规则可迁移 |
| **沙箱集成** | 动态恶意软件分析 | 文件上传 API（WildFire/FortiSandbox 兼容）| 沙箱报告 + verdict | 文件上传 + 报告拉取完整链路 |
| **威胁情报框架** | 已知恶意 IP/域名/URL 检测 | STIX 2.0/OTX IOC 输入 | 内存 IOC 表 + 命中事件 | STIX/OTX 导入 + 命中检测 |
| **Kubernetes DaemonSet** | 容器网络东西向流量检测 | K8s DaemonSet + 服务发现 | Pod 级别告警 | K8s 部署 Chart + 服务发现 |
| **服务网格集成** | Istio 扩展 | wasm filter 镜像流量 | 微服务间流量告警 | Istio 扩展镜像 + 配置 |
| **Web UI + REST API** | 产品化可用性 | Angular/React + OpenAPI | 可视化规则管理 + REST | 管理界面基础功能上线 |

#### Phase 3 里程碑

| 指标 | Phase 2 | Phase 3 目标 |
|------|---------|-------------|
| 检测模式 | 规则匹配 | **规则 + ML 异常检测** |
| 恶意软件 | 无 | **文件提取 + 沙箱集成** |
| 威胁情报 | 无 | **STIX/OTX 原生对接** |
| 部署模式 | 嵌入式/物理机 | **K8s DaemonSet + Istio** |
| 产品形态 | IDS 引擎 | **NDR 平台** |

---

## 4. 资源需求

### 4.1 人力估算

| 阶段 | 核心功能 | 预估人月 | 依赖 |
|------|---------|---------|------|
| **Phase 1** | flowbits + PCRE + HTTP + JA3 + EVE JSON | **4-6 人月** | PCRE2/Oniguruma 库、HTTP parser |
| **Phase 2** | Hyperscan + DPDK + 分片重组 + HA + DNS/SMB | **12-16 人月** | Intel Hyperscan、DPDK 库 |
| **Phase 3** | ML + 文件提取 + 沙箱 + TI + K8s + WebUI | **16-20 人月** | ML 框架、K8s SDK、沙箱 API |

### 4.2 关键外部依赖

| 依赖 | 阶段 | 许可证 | 备注 |
|------|------|--------|------|
| **PCRE2 / Oniguruma** | Phase 1 | BSD/GPL | 正则表达式引擎 |
| **Intel Hyperscan** | Phase 2 | BSD | 模式匹配（需 Intel CPU 或兼容库）|
| **DPDK** | Phase 2 | BSD | 高性能数据包捕获（数据中心场景）|
| **ET Open 规则** | Phase 1+ | GPLv2 | 规则来源（35000+）|
| **Snort VRT** | Phase 2 | 专有（订阅）| 补充规则来源 |
| **STIX/OTX 库** | Phase 3 | BSD | 威胁情报输入 |

### 4.3 硬件需求

| 阶段 | 需求 | 用途 |
|------|------|------|
| Phase 1 | 标准 x86 开发机 | 功能开发 + 规则兼容性测试 |
| Phase 2 | **Intel VT-d 服务器**（10Gbps NIC×2）| DPDK/AF_XDP 基准测试 |
| Phase 3 | K8s 集群（3+ 节点）| DaemonSet + Istio 集成测试 |

---

## 5. 风险与缓解

### 5.1 技术风险

| 风险 | 概率 | 影响 | 缓解策略 |
|------|------|------|---------|
| **Hyperscan 在非 Intel 平台性能差** | 中 | 高 | Phase 2 保留 AC 作为 fallback；嵌入式场景评估 RIE |
| **PCRE 正则回溯导致 DoS** | 低 | 高 | 启用 PCRE2 `PCRE2_MATCH_OK` 模式，限制匹配步数/时间 |
| **DPDK 与容器环境冲突** | 中 | 中 | Phase 2 提供 DPDK 选项，嵌入式保留 AF_PACKET |
| **flowbits 状态机内存泄漏** | 低 | 高 | 引入状态超时机制 + 内存上限保护 |

### 5.2 资源风险

| 风险 | 概率 | 影响 | 缓解策略 |
|------|------|------|---------|
| **Phase 2 HA 集群设计复杂** | 中 | 中 | 参考 Zeek/Consul 架构，分阶段交付（先主动-被动）|
| **ML 模块数据标注成本高** | 高 | 中 | 优先使用 Snort libml 方法论，结合开源数据集 |
| **规则兼容性测试人力不足** | 中 | 中 | Phase 1 末建立自动化回归测试套件 |

### 5.3 时间风险

| 风险 | 概率 | 影响 | 缓解策略 |
|------|------|------|---------|
| **Phase 2 DPDK 集成延期** | 中 | 高 | Phase 1 提前完成 AF_PACKET 优化，Phase 2 可分批交付 |
| **Phase 3 K8s/Istio 生态集成超出范围** | 中 | 中 | Phase 3 末期评审，明确 MVP 范围（DaemonSet 优先，Istio 降级）|

---

## 6. 决策点

> 以下 4 个决策需委员会评审确认，决策结果将影响架构方向和资源分配。

---

### 决策 1：规则兼容策略 — Snort 子集 vs 自研 DSL

| 选项 | 描述 | 推荐 |
|------|------|------|
| **A. Snort 子集扩展（推荐）** | 继续扩展 Snort3 子集兼容，支持 flowbits/PCRE/http_method 等关键字，对接 ET Open/VRT 规则生态 | ✅ |
| **B. 自研 DSL** | 完全自研规则语言，掌控权最大化，但规则社区割裂，迁移成本极高 | ❌ |

**推荐理由**：自研 DSL 需重建规则生态（35000+ ET Open 规则不可用），与差异化定位（嵌入式 + 车规网关）不符。

---

### 决策 2：Phase 2 架构升级优先级 — DPDK vs Hyperscan

| 选项 | 描述 | 推荐 |
|------|------|------|
| **A. Hyperscan 优先** | 先升级模式匹配引擎（规则集容量 10x），再评估 DPDK | ✅ |
| **B. DPDK 优先** | 先升级捕获引擎（吞吐量 100Gbps），Hyperscan 作为后续迭代 | ⚠️ |

**推荐理由**：当前瓶颈是规则集容量和 PCRE 性能，不是捕获速度；100Mbps → 10Gbps 的核心瓶颈在匹配层，不在捕获层。

---

### 决策 3：开源策略 — 开源 vs 闭源

| 选项 | 描述 | 推荐 |
|------|------|------|
| **A. 核心引擎开源** | 将 NIDS 核心引擎开源（GitHub），吸引社区贡献，对接 ET Open 规则生态 | ✅ |
| **B. 完全闭源** | 保持完全闭源，依赖商业规则订阅和服务 | ⚠️ |

**推荐理由**：开源三强（Snort/Suricata/Zeek）均为开源，闭源难以进入安全社区；开源核心 + 商业增值（规则订阅/支持/定制）是更可持续的模式。

---

### 决策 4：商业模式 — 内嵌 vs 独立产品

| 选项 | 描述 | 推荐 |
|------|------|------|
| **A. 内嵌模式（推荐）** | NIDS 作为安全网关的内嵌组件，按网关出货量计费，聚焦嵌入式/车规场景 | ✅ |
| **B. 独立产品模式** | NIDS 作为独立产品销售（硬件/VM/容器），与安全网关解耦 | ⚠️ |

**推荐理由**：当前差异化定位是嵌入式 + 车规安全网关，独立产品需与 Suricata/Snort 正面竞争，资源消耗大；内嵌模式可借力安全网关渠道，快速放量。

---

## 附录：关键指标汇总

| 指标 | 当前 | Phase 1 目标 | Phase 2 目标 | Phase 3 目标 |
|------|------|-------------|-------------|-------------|
| **Snort 规则兼容** | ~60% | >90% | >95% | >98% |
| **吞吐量** | 100Mbps | 100Mbps | 10Gbps+ | 10Gbps+ |
| **协议层** | L3/L4 | L3/L4 + HTTP | +DNS/SMB/FTP | L3-L7 |
| **TLS 检测** | 无 | JA3 | JA3/JA4 | JA3/JA4 |
| **日志格式** | SOA JSON | EVE JSON | EVE JSON | EVE JSON + 扩展 |
| **分片重组** | 无 | 无 | 支持 | 支持 |
| **集群/HA** | 无 | 无 | HA 2+ 节点 | 多节点扩展 |
| **ML 检测** | 无 | 无 | 无 | HTTP 参数分类 |
| **云原生** | 无 | 无 | 无 | K8s DaemonSet |

---

*文档用于内部评审，可直接导出为 slides。关键决策点需在评审会前确认。*
