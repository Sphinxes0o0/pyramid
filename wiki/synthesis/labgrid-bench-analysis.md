---
type: synthesis
tags: [labgrid, bench-management, infrastructure, embedded]
created: 2026-05-29
sources: [github-labgrid]
---

# Labgrid Bench Management Replacement Analysis

## Background

Labgrid is an embedded board control library from Pengutronix (2016+) focused on testing, development, and automation. It provides remote client-exporter-coordinator infrastructure for controlling hardware from different computers on a network.

## Architecture Overview

### Core Components

| Component | Purpose |
|-----------|---------|
| **Target** | Python object representing a controllable embedded system |
| **Resource** | Passive information store for hardware access (e.g., RawSerialPort, NetworkPowerPort) |
| **Driver** | Active component using Resources to perform actions (e.g., SSHDriver, NetworkPowerDriver) |
| **Protocol** | Interface contracts (PowerProtocol, CommandProtocol, ConsoleProtocol) |
| **Strategy** | State machine for board boot sequences (BareboxStrategy, UBootStrategy) |

### Remote Infrastructure

```
┌─────────────┐     gRPC      ┌─────────────┐     gRPC      ┌─────────────┐
│   Client    │◄────────────►│ Coordinator │◄────────────►│  Exporter   │
│ (labgrid-   │               │ (labgrid-   │               │ (hardware   │
│  client)    │               │ coordinator)│               │  host)      │
└─────────────┘               └─────────────┘               └─────────────┘
                                                                  │
                                                    ┌─────────────┼─────────────┐
                                                    │             │             │
                                              ser2net         udev         direct
                                              (serial)      (USB)        (GPIO)
```

## What Labgrid Can Replace

### 1. Power Control

**Replaces:** relay + usbrelay for power switching

**Options:**

| Approach | Effort | Notes |
|----------|--------|-------|
| `ExternalPowerDriver` | Low | Wrap existing `usbrelay` CLI commands via `cmd_on`/`cmd_off`/`cmd_cycle` |
| Custom power backend | Medium | Follow `apc.py` pattern — Python module with `power_set`/`power_get` functions |
| Existing backends | Low | SNMP (APC), REST (various PDUs), Shelly, TPLink, etc. |

**ExternalPowerDriver config example:**
```python
@target_factory.reg_driver
@attr.s(eq=False)
class CCCPowerDriver(Driver, PowerResetMixin, PowerProtocol):
    cmd_on = "/usr/local/bin/ccc_relay on"
    cmd_off = "/usr/local/bin/ccc_relay off"
    cmd_cycle = "/usr/local/bin/ccc_relay cycle"
    delay = 2.0
```

**Custom backend example** (for `usbrelay`):
```python
# labgrid/driver/power/usbrelay.py
import subprocess

def power_set(host, port, index, value):
    state = "1" if value else "0"
    subprocess.run(["usbrelay", f"{index}_1={state}"])

def power_get(host, port, index):
    # parse usbrelay output
    ...
```

### 2. Serial Device Access

**Replaces:** `/dev/ccc-*` serial devices accessed directly

**Mechanism:** `SerialPortExport` runs `ser2net` to expose local serial ports via network (RFC2217/raw TCP).

```yaml
# exporter config.yaml
groups:
  nuc1:
    serial_a:
      cls: RawSerialPort
      port: /dev/ccc-serial-a
```

Client sees `NetworkSerialPort` with `host:port` — transparently proxies through exporter.

### 3. SSH Execution

**Replaces:** Direct SSH to NUCs

**Driver:** `SSHDriver` implements `CommandProtocol` and `FileTransferProtocol`

- Uses SSH ControlMaster for connection reuse
- Supports proxy jumping via exporter
- SCP/SFTP for file transfer
- Port forwarding helpers

```python
# Configuration via NetworkService resource
SSHDriver(
    networkservice=NetworkService(address="nuc1.local", username="root"),
    keyfile="/path/to/key"
)
```

### 4. Multi-User Scheduling / Reservation

**Replaces:** nuccli lock file reservation

**Labgrid Coordinator provides:**

| Feature | nuccli | Labgrid Coordinator |
|---------|--------|---------------------|
| Lock mechanism | File-based (`flock`) | gRPC with coordinator state |
| User tracking | Username in lockfile | Session tracking |
| Reservation | None | Token-based with expiry |
| Scheduling | None | Tag-based with priority queue |
| Place abstraction | Host-level | Resource group with match patterns |

**Coordinator reservation flow:**
1. Client creates reservation with filters (e.g., `arch=x86_64`, `gpu=nvidia`)
2. Coordinator schedules matching available places
3. Client polls reservation state until `allocated`
4. Client acquires place (all matching resources locked)
5. Resources only accessible to acquiring client

**Resource match patterns:**
```
*/nuc1/*           # All resources from group nuc1
*/nuc1/NetworkPowerPort  # Only power port from nuc1
exporter1/hub1/*  # All resources in hub1 group from exporter1
```

## What Labgrid Cannot Replace

### 1. NUC-Specific Flashing Workflows

Labgrid has loaders for:
- IMXUSBLoader (i.MX ROM loader)
- AndroidFastboot
- UBoot via serial

**Gap:** Custom flashing procedures (BIOS update, firmware flash, provisioning) require custom `Strategy` implementations. Labgrid provides the framework but not the NUC-specific knowledge.

**Workaround:** Keep flashing scripts separate; call from labgrid via `SSHDriver.run()` or `ExternalPowerDriver`-style wrappers.

### 2. Custom Relay Hardware

**Gap:** Any hardware without existing backend:
- Proprietary relay controllers
- Serial-based power switches
- GPIO-triggered power circuits

**Solution:** Write custom power backend module following the `apc.py` pattern — ~50 lines Python.

### 3. Hardware Feedback Signals

Labgrid's ManagedResource/udev integration handles device discovery, but:
- No built-in for voltage/current monitoring
- No watchdog timers
- No hardware fault detection beyond "available/unavailable"

## Custom Driver Requirements

### For usbrelay Hardware

```python
# labgrid/resource/usbrelay.py
@target_factory.reg_resource
@attr.s(eq=False)
class USBRelayPort(Resource):
    """USB relay port controlled via usbrelay CLI"""
    index = attr.ib(validator=attr.validators.instance_of(int))
    command_prefix = attr.ib(default=[], validator=attr.validators.instance_of(list))
```

```python
# labgrid/driver/usbrelay_power.py
@target_factory.reg_driver
@attr.s(eq=False)
class USBRelayPowerDriver(Driver, PowerResetMixin, PowerProtocol):
    bindings = {"port": "USBRelayPort"}

    def on_activate(self):
        self.tool = self.target.env.config.get_tool('usbrelay') or 'usbrelay'

    @Driver.check_active
    @step()
    def on(self):
        subprocess.run([self.tool, f"{self.port.index}_1=1"])

    @Driver.check_active
    @step()
    def off(self):
        subprocess.run([self.tool, f"{self.port.index}_1=0"])
```

### For ccc_relay (custom relay daemon)

```python
# labgrid/driver/power/ccc_relay.py
def power_set(host, port, index, value):
    subprocess.run(["/usr/local/bin/ccc_relay", "set", str(index), "on" if value else "off"])

def power_get(host, port, index):
    out = subprocess.check_output(["/usr/local/bin/ccc_relay", "get", str(index)])
    return b"on" in out
```

## Migration Effort Estimate

| Component | Effort | Complexity | Notes |
|-----------|--------|------------|-------|
| Power (usbrelay) | 2-3 days | Low | ExternalPowerDriver or custom backend |
| Power (custom relay) | 3-5 days | Medium | Custom backend module + testing |
| Serial via ser2net | 1-2 days | Low | Configure exporter, no code changes |
| SSH execution | 2-3 days | Medium | SSHDriver config, ControlMaster setup |
| Coordinator deployment | 3-5 days | High | gRPC, auth, HA considerations |
| nuccli → Reservation | 5-7 days | High | Rewrite reservation logic, client migration |
| Custom flashing strategies | 5-10 days | High | Per-NUC strategy implementation |
| Testing/integration | 5-10 days | High | Full bench regression testing |

**Total: 4-6 weeks for full migration**

## Recommended Migration Phases

### Phase 1: Power Control (Week 1-2)
1. Deploy `ExternalPowerDriver` wrapping existing `usbrelay`/`ccc_relay` CLI
2. Verify equivalent behavior to current relay control
3. No coordinator needed — local testing only

### Phase 2: Serial + SSH (Week 2-3)
1. Deploy exporter on NUCs with ser2net for `/dev/ccc-*`
2. Configure SSHDriver for NUC access
3. Test serial console over labgrid vs direct

### Phase 3: Coordinator + Reservation (Week 3-5)
1. Deploy coordinator (single instance, no HA initially)
2. Migrate places from nuccli lock files
3. Implement tag-based scheduling for NUC selection
4. Cut over clients to labgrid-client

### Phase 4: Strategy + Flashing (Week 5-7)
1. Implement per-NUC boot strategies
2. Migrate flashing scripts to Strategy classes
3. Full integration testing

## Alternative: Incremental Adoption

Instead of full migration:

1. **Use labgrid only for new NUCs** — no migration of existing
2. **Run coordinator alongside nuccli** — parallel reservation systems
3. **Labgrid for test automation only** — dev/debug stays with nuccli

This reduces risk but maintains two systems.

## Conclusion

Labgrid is a well-designed, production-tested framework that could replace the power/serial/SSH/reservation layer of the current bench setup. The main gaps are:

1. **Custom flashing workflows** — require strategy implementation but framework supports it
2. **Custom relay hardware** — custom driver needed (~3-5 days)
3. **Coordinator operational overhead** — gRPC service requiring care

The nuccli lock-file reservation is simpler but less capable than labgrid's coordinator. If multi-user scheduling with priorities/tags is needed, coordinator is worth the migration cost. If simple lock-based reservation suffices, ExternalPowerDriver + SSHDriver may be enough without full coordinator deployment.

**Recommendation:** Start with Phase 1 (ExternalPowerDriver) to validate labgrid behavior with existing hardware, then decide on full migration based on results.

---

## 扩展阅读

本文侧重 bench management 场景。关于车载多 ECU 场景的深度分析（Hierarchical Place、Resource Domain、VehicleStrategy），参见 [[wiki/sources/chatgpt-labgrid-analysis]] 和 [[wiki/synthesis/labgrid-automotive-scenario]]。
