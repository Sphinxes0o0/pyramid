---
type: synthesis
tags: [labgrid, automotive, central-compute, ccc, nio, embedded, testing]
created: 2026-05-29
sources: [chatgpt-labgrid-analysis, labgrid-bench-analysis]
---

# Labgrid 车载 CCC 测试场景分析

## NIO 中央 CCC 架构解析

### 计算平台布局

```
┌─────────────────────────────────────────────────────────────────────┐
│  中央计算集群 CCC (Central Compute Cluster)                          │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │
│  │     CDF       │  │     ADF      │  │     VDF      │  │     SAF      │             │
│  │  (座舱娱乐)   │  │  (智驾感知)  │  │  (车控)     │  │  (车联网)    │             │
│  │              │  │              │  │              │             │
│  │  Android VM  │  │  Linux VM    │  │  AUTOSAR AP  │             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘             │
│         │                 │                 │                      │
│         └─────────────────┼─────────────────┘                      │
│                           │                                         │
│                    Service Router (SOME/IP)                        │
│                           │                                         │
│  ┌────────────────────────┴───────────────────────────────────┐   │
│  │           Automotive Ethernet Spine (1Gbps + gPTP)          │   │
│  └────────────────────────┬───────────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┬───────────────┐
            │               │               │               │
     ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
     │ Zone Ctrl   │ │ Zone Ctrl   │ │ Zone Ctrl   │ │ Zone Ctrl   │
     │ Front-Left  │ │Front-Right  │ │ Rear-Left   │ │ Rear-Right  │
     │  (ZF_L)     │ │  (ZF_R)     │ │  (ZR_L)     │ │  (ZR_R)     │
     └──────┬──────┘ └──────┬──────┘ └──────┬──────┘ └──────┬──────┘
            │               │               │               │
      LIN/PWM/I2C    LIN/PWM/I2C     LIN/PWM/I2C     LIN/PWM/I2C
            │               │               │               │
      传感器/执行器    传感器/执行器    传感器/执行器    传感器/执行器
```

### 实时计算单元 RCU

安全关键功能运行在独立实时计算单元（RTU，AUTOSAR CP 或 RTOS），与中央 SoC 通过硬件安全信道通信：

- **动力域**：加速/刹车信号（ISO 26262 ASIL-D）
- **转向域**：EPS 控制（ASILD）
- **安全域**：气囊/约束系统（ASILD）

---

## 核心问题：为什么需要中央 CCC 建模

中央 CCC 架构带来新的测试挑战：

1. **软件分区隔离**：多个 Domain VM/容器共享同一 SoC，需要资源配额和故障隔离测试
2. **服务化通信**：ECU 间 CAN 消息 → SOME/IP 服务调用，测试点从帧级移到服务级
3. **时间敏感网络**：gPTP 时钟同步、TSN 整形器需要端到端时延验证
4. **OTA 原子性**：整车主更新需要分区回滚、事务一致性保证
5. **区域控制器**：Zonal 架构下的传感器注入和执行器反馈需要新的测试抽象

---

## Hierarchical Place 设计（CCC 版本）

### 概念

Place 是 labgrid 的核心抽象，代表一个**原子化 DUT 资源集合**。在中央 CCC 场景中，Place 层次对应架构边界：

```
VehicleCCC (顶层 Place)
│
├── Power Domain
│    └── 12V Bus, 5V Rail, 3.3V Rail, VBist
│
├── Network Domain
│    ├── Ethernet Spine (1000BASE-T1, gPTP)
│    ├── Service Bus (SOME/IP)
│    └── Zone Controller Backbone
│
├── CCC_Central (子 Place — 中央计算集群)
│    ├── CCC_CDF (Domain Place — 座舱)
│    │    ├── AndroidVM (QEMU/LibreKVM)
│    │    ├── DisplayPort (DP)输出
│    │    └── AudioCodec (I2S)
│    │
│    ├── CCC_ADF (Domain Place — 智驾)
│    │    ├── SensorFusion (Camera/Radar/Lidar)
│    │    ├── PerceptionModel (ONNX Runtime)
│    │    └── PlanningControl (ISO 21448 SOTIF)
│    │
│    └── CCC_VDF (Domain Place — 车控)
│         ├── BodyControl (车窗/灯光/雨刮)
│         └── ComfortControl (空调/座椅)
│
├── CCC_RTU (子 Place — 实时计算单元)
│    ├── RealTimeOS ( AUTOSAR CP 或 seL4)
│    ├── Safety岛 (动力/转向/约束)
│    └── HW Security Channel (SPI/TTL)
│
└── ZoneControllers (子 Place — 区域控制器)
     ├── ZC_FrontLeft (ZF_L)
     │    ├── LINGateway (传感器总线)
     │    ├── PWMGateway (执行器驱动)
     │    └── EthernetBridge (TSN 端口)
     │
     ├── ZC_FrontRight (ZF_R)
     ├── ZC_RearLeft (ZR_L)
     └── ZC_RearRight (ZR_R)
```

### 与传统 ECU Place 的本质区别

传统 ECU Place 代表**物理盒子**；中央 CCC Place 代表**软件域**。这意味着：

- 一个物理 CCC 可能运行 3 个 Domain Place，每个 Domain 内部是 VM/容器
- ZoneController Place 仍然是物理盒子，但功能是**桥接和汇聚**而非计算
- 测试资源按**服务边界**而非 ECU 边界划分

---

## Resource Domain 划分：Service-Oriented

### Central Domain（中央计算域）

中央 SoC 内的软件分区资源，被同域内多个 Driver 共享：

| Resource | 特点 |
|----------|------|
| CDF Service Bus | SOME/IP 订阅/发布，Android 域内 |
| ADF Sensor Bus | 原始传感器数据流（Camera/Radar/Lidar） |
| VDF Service Router | 车控服务（灯光/空调/座椅） |
| OTA Manager | 分区更新事务状态 |

### Real-Time Domain（实时计算域）

独立 RTU 内的安全关键资源，只被 RTU Driver 访问：

| Resource | 特点 |
|----------|------|
| Safety Channel | 与 CCC 通信的硬件安全信道 |
|动力 CAN | 动力域实时控制（加速/刹车）|
| 转向 PWM | EPS 角度指令 |
| 约束系统 | 气囊展开逻辑 |

### Zonal Domain（区域控制域）

Zone Controller 内的 I/O 聚合资源：

| Zone | Local Resources |
|------|-----------------|
| ZF_L | 大灯(LIN)、左后视镜(PWM)、左门锁(LIN) |
| ZF_R | 大灯(LIN)、右后视镜(PWM)、右门锁(LIN) |
| ZR_L | 尾灯(LIN)、左座椅( LIN/PWM)、左轮速(CAN) |
| ZR_R | 尾灯(LIN)、右座椅(LIN/PWM)、右轮速(CAN) |

---

## 嵌套锁机制

中央 CCC 场景的锁层级对应**架构分层**，而非物理 ECU 边界：

```
┌─────────────────────────────────────────────────────────────┐
│  Global Lock (VehicleCCC)                                    │
│  ├── Power Bus                                               │
│  ├── Ethernet Spine (gPTP)                                   │
│  │                                                            │
│  │  ┌─────────────────────────────────────────────────┐     │
│  │  │  Domain Lock (CCC_ADAS)                         │     │
│  │  │  ├── Sensor Bus Lock                            │     │
│  │  │  ├── Perception Model Lock                      │     │
│  │  │  └── Planning Control Lock                       │     │
│  │  └─────────────────────────────────────────────────┘     │
│  │                                                            │
│  │  ┌─────────────────────────────────────────────────┐     │
│  │  │  Domain Lock (CCC_RTU)                          │     │
│  │  │  ├── Safety Channel Lock                        │     │
│  │  │  └── Power Train Lock (CAN)                     │     │
│  │  └─────────────────────────────────────────────────┘     │
│  │                                                            │
│  │  ┌─────────────────────────────────────────────────┐     │
│  │  │  Zonal Lock (ZC_FrontLeft)                     │     │
│  │  │  ├── LIN Bus Lock                               │     │
│  │  │  └── PWM Channel Lock                           │     │
│  │  └─────────────────────────────────────────────────┘     │
└──┘
```

### 死锁避免规则

1. **固定获取顺序**：Global → Central Domain → RTU → Zonal（按架构层次）
2. **超时机制**：锁获取超时后自动释放已获取的锁
3. **可中断操作**：支持 Ctrl-C 中断测试并清理锁
4. **跨域安全**：CCC ↔ RTU 的 Safety Channel 锁独立于 Domain 锁

```python
@contextmanager
def nested_lock(lock_order):
    acquired = []
    try:
        for lock in lock_order:
            acquired.append(lock.acquire(timeout=30))
        yield
    finally:
        for lock in reversed(acquired):
            if lock.acquired:
                lock.release()
```

---

## 网络隔离设计（TSN + Service-Oriented）

### Central CCC 网络拓扑

| 层次 | 协议 | 带宽 | 确定性 |
|------|------|------|--------|
| Spine | 1000BASE-T1 (gPTP) | 1Gbps | 高（TSN 时间同步）|
| Service Bus | SOME/IP + DDS | - | 中（UDP）|
| Zone Backbone | 100BASE-T1 | 100Mbps | 中 |
| Sensor Local | CAN FD / LIN | 5Mbps / 20kbps | 低 |

### 建议架构

```
┌─────────────────────────────────────────────────────────────┐
│  Exporter Host (NUC)                                         │
│                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ CCC_SoC  │  │ CCC_RTU  │  │ ZC_FL   │  │ ZC_FR   │   │
│  │  eth0    │  │  eth1    │  │  eth2   │  │  eth3   │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │              │        │
│  ┌────┴──────────────┴──────────────┴──────────────┴────┐   │
│  │              bridge0 (Automotive Ethernet Spine)      │   │
│  │                    + gPTP Grandmaster                   │   │
│  └────────────────────────────────────────────────────────┘   │
│       │              │              │              │        │
│   eth0 (上行)    CAN Adapter   LIN Adapter   Service Bus    │
└─────────────────────────────────────────────────────────────┘
```

### 物理隔离 vs 虚拟化隔离

| 方案 | 适用场景 | 复杂度 |
|------|----------|--------|
| 物理 TSN Switch | 确定性要求最高（动力域） | 高 |
| vETH + Linux TSN | 中央 SoC 内域间隔离 | 中 |
| VLAN on spine | 同域内多 Zone 隔离 | 低 |
| VM 防火墙 | Domain 间服务隔离 | 中 |

---

## CCCStrategy 状态机设计

### 中央 CCC 协调状态机

```python
from enum import Enum, auto
from labgrid.strategy import Strategy

class CCCState(Enum):
    POWER_OFF = auto()
    RTU_INIT = auto()         # RTU 实时单元先启动（安全关键）
    CCC_BOOT = auto()         # 中央 SoC 启动
    NETWORK_SYNC = auto()     # gPTP 时钟同步
    SERVICE_DISCOVERY = auto() # SOME/IP 服务注册
    DOMAIN_READY = auto()     # 各 Domain 就绪
    OPERATIONAL = auto()      # 整车运行
    SHUTDOWN = auto()         # 优雅关机

class CCCStrategy(Strategy):
    """
    NIO 中央 CCC 协调策略
    """

    def __init__(self, target, places):
        super().__init__(target)
        self.places = places
        self.state = CCCState.POWER_OFF

    def transition(self, target_state):
        """状态转换，按正确顺序协调中央 CCC"""
        states = list(CCCState)
        current_idx = states.index(self.state)
        target_idx = states.index(target_state)

        if target_idx < current_idx:
            return self._shutdown_sequence(target_state)
        else:
            return self._boot_sequence(target_state)

    def _boot_sequence(self, target_state):
        with self.nested_lock(self._get_lock_order()):
            if target_state >= CCCState.RTU_INIT:
                self._init_rtu_power()
                self._init_safety_channel()

            if target_state >= CCCState.CCC_BOOT:
                self.places.CCC_Central.boot()

            if target_state >= CCCState.NETWORK_SYNC:
                self._sync_gptp_clock()
                self._configure_tsn_shaper()

            if target_state >= CCCState.SERVICE_DISCOVERY:
                self._wait_for_service_discovery()

            if target_state >= CCCState.DOMAIN_READY:
                self.places.CCC_Central.wait_domains_ready()

    def _get_lock_order(self):
        """返回全局锁顺序，避免死锁"""
        return [
            self.places.VehicleCCC.locks.global_power,
            self.places.VehicleCCC.locks.ethernet_spine,
            self.places.CCC_RTU.locks.local,
            self.places.CCC_Central.locks.local,
            self.places.ZoneControllers.locks.local,
        ]
```

### Domain 子状态机

每个软件域 Place 有自己的子状态机：

```python
class DomainStrategy(Strategy):
    """
    单个软件域的状态机基类
    """
    states = (OFF, BOOTING, WAITING_SERVICES, READY, RUNNING, FAULT)

    def transition(self, state):
        if state == self.OFF:
            self._power_off()
        elif state == self.BOOTING:
            self._init_vm_container()
            self._load_domain_services()
        elif state == self.WAITING_SERVICES:
            self._discover_dependencies()
            self._bind_services()
        elif state == self.READY:
            self._run_diagnostics()
        elif state == self.RUNNING:
            self._start_application()
```

---

## 事件总线建议（服务化架构）

中央 CCC 架构下，ECU 间消息 → 服务调用，事件总线需要服务级抽象。

### 选项对比

| 方案 | 适用规模 | 延迟 | 服务发现 | 生态 |
|------|----------|------|----------|------|
| **SOME/IP** | 中型 | <5ms | 是 | 汽车行业标准（AUTOSAR）|
| **gRPC** | 中型 | <10ms | 是 | 云原生，扩展方便 |
| **DDS** | 中型 | <1ms | 是 | 实时，机器人行业流行 |
| **MQTT** | 小型 | <50ms | 否 | IoT，不适合实时控制 |

### 建议：SOME/IP + DDS 混合

```
┌─────────────────────────────────────────────────────────────┐
│  Test Orchestrator                                          │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  SOME/IP Proxy (测试服务调用)                           │  │
│  │  - Service discovery (测试环境模拟)                    │  │
│  │  - Method calls (同步/异步)                            │  │
│  │  - Event notifications                                 │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  DDS (传感器数据流)                                     │  │
│  │  - Camera streams                                     │  │
│  │  - Radar point clouds                                 │  │
│  │  - Lidar scans                                        │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  gRPC (测试控制和配置)                                  │  │
│  │  - OTA test control                                   │  │
│  │  - Fault injection                                    │  │
│  │  - Telemetry collection                               │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### SOME/IP Service 设计

```
vehicle/ccc/CDF/DisplayControl/{instance}   # 显示屏控制服务
vehicle/ccc/ADF/SensorFusion/{instance}    # 传感器融合服务
vehicle/ccc/VDF/BodyControl/{instance}      # 车控服务
vehicle/ccc/RTU/SafetyChannel/{instance}   # 安全通道服务
vehicle/zonal/{zone}/IOMux/{instance}      # 区域 IO 复用服务
```

---

## Digital Twin 虚实结合

### HIL/SIL 分层（中央 CCC 版）

```
┌─────────────────────────────────────────────────────────────┐
│  Test Layer (pytest)                                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Digital Twin Simulator                               │  │
│  │  - Vehicle dynamics model                            │  │
│  │  - Sensor simulation (camera, radar, lidar)          │  │
│  │  - Service mock (SOME/IP stubs)                     │  │
│  │  - TSN network emulator                               │  │
│  └──────────────────────────────────────────────────────┘  │
│                         │                                    │
│                    Simulated │ Real                          │
│                         ▼                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Automotive Ethernet Spine (gPTP + TSN)              │  │
│  └──────────────────────────────────────────────────────┘  │
│              │                        │                      │
│       ┌──────┴──────┐          ┌──────┴──────┐            │
│       │ Central SoC  │          │   Simulated  │            │
│       │ (Real ADAS)  │          │   ADAS VM     │            │
│       └─────────────┘          └───────────────┘            │
└─────────────────────────────────────────────────────────────┘
```

### 仿真切换

labgrid 的 Resource 发现机制支持热切换：

```python
# 配置：优先使用真实硬件，硬件不可用时自动切换到仿真
class AdaptiveResourceProvider:
    def get_resource(self, resource_class, filters):
        # 1. 尝试获取真实 Resource
        real = self.exporter.get_resource(resource_class, filters)
        if real and real.is_available():
            return real

        # 2. 回退到 Digital Twin Resource
        return self.simulator.create_resource(resource_class, filters)
```

---

## 与 safeOS 的关联

[[wiki/entities/linux/safeos/safeos-lwip-sel4-performance-boundary]] 讨论了 safeOS 在 seL4 微内核上运行 lwIP 的性能边界，这与中央 CCC 架构有直接交叉：

- **seL4 作为 RTU OS**：seL4 的形式化验证对安全关键 RTU 极具吸引力
- **safeOS 网络栈**：中央 CCC 的 Automotive Ethernet 需要高性能确定性协议栈
- **RTOS + Linux 混合**：[[wiki/entities/linux/safeos/safeos-packet-mmap]] 中的零拷贝技术可用于中央 SoC 与 Zone Controller 之间的高速数据路径

---

## 结论

labgrid 的中央 CCC 扩展关键设计点：

1. **Hierarchical Place**：按架构层次建模（CCC → Domain → Zone），而非物理 ECU
2. **Service-Oriented Resource**：CAN/LIN 帧 → SOME/IP 服务调用
3. **嵌套锁**：按架构层次获取锁（Global → Domain → RTU → Zonal）
4. **TSN 网络隔离**：gPTP 时钟同步 + TSN 整形器支持确定性网络
5. **事件总线**：SOME/IP/DDS 混合处理服务调用和传感器数据流
6. **Digital Twin**：中央 SoC 软件分区虚拟化，虚实结合降低 HIL 成本

---

## 相关页面

- [[wiki/synthesis/labgrid-bench-analysis]] — Bench management 场景（分布式 ECU 参考）
- [[wiki/sources/chatgpt-labgrid-analysis]] — ChatGPT 原始分析
- [[wiki/entities/linux/safeos/safeos-lwip-sel4-performance-boundary]] — seL4 + lwIP 性能
- [[wiki/entities/linux/safeos/safeos-packet-mmap]] — safeOS 零拷贝网络
