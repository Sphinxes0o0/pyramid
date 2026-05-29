---
type: reference
tags: [labgrid, api, powerdriver, phase1]
created: 2026-05-29
source: labgrid/driver/powerdriver.py
---

# ExternalPowerDriver API 参考

> 从 labgrid 源码提取（powerdriver.py:116-147）

## 类定义

```python
@target_factory.reg_driver
@attr.s(eq=False)
class ExternalPowerDriver(Driver, PowerResetMixin, PowerProtocol):
```

## 参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `cmd_on` | `str` | **必填** | 上电命令，`shlex.split()` 后执行 |
| `cmd_off` | `str` | **必填** | 断电命令 |
| `cmd_cycle` | `Optional[str]` | `None` | 直接 cycle 命令。为空则 off→sleep→on |
| `delay` | `float` | `2.0` | cycle 时 off/on 之间的间隔秒数 |

## 方法

| 方法 | 实现 |
|------|------|
| `on()` | `processwrapper.check_output(shlex.split(cmd_on))` |
| `off()` | `processwrapper.check_output(shlex.split(cmd_off))` |
| `cycle()` | 有 `cmd_cycle` → 执行 cmd_cycle；无 → `off() → sleep(delay) → on()` |

Exit code != 0 抛出异常，命令超时 30s。

## PowerProtocol 接口

继承自 `PowerProtocol`（protocol.py）：
- `on()`, `off()`, `cycle()` 已实现
- `get()` — 可选，状态查询（ExternalPowerDriver 未重写，需自行实现）

PowerResetMixin: `reset()` → `self.cycle()`

## YAML 配置示例

```yaml
targets:
  main:
    drivers:
      - cls: ExternalPowerDriver
        name: relay-power
        cmd_on: "sudo relay 01 on && sudo relay 02 on"
        cmd_off: "sudo relay 01 off && sudo relay 02 off"
        delay: 5.0
    
      - cls: ExternalPowerDriver
        name: usbrelay-mode
        cmd_on: "usbrelay 6QMBS_1=1 && usbrelay 6QMBS_2=1"
        cmd_off: "usbrelay 6QMBS_1=0 && usbrelay 6QMBS_2=0"
```

## 注意事项

1. **sudo**: relay 需要 sudo。若未配 NOPASSWD，ExternalPowerDriver 会卡住等密码
2. **device ID**: usbrelay 设备 ID 每台 NUC 不同（6QMBS/HW341/...），需自动发现
3. **cycle 时序**: relay cycle 需要 `off → sleep 5 → usbrelay → sleep 5 → on`，不能简单用 cmd_cycle 字符串
4. **delay**: bench 电源 cycle 需 10s+（含电容放电），默认 2s 不够
