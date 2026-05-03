# EPSApplication SDD

## 1. Overview

`EPSApplication` is the Layer 3 Active component for the EPS subtopology. It monitors battery and power system health by consuming state data published by `MpptIcManager` on each rate group tick and pushing a health/status struct to `SatStateMachine` so submode decisions can be made. It accepts commands from both ground and `SatStateMachine` to configure the power system (charging, MPPT, JEITA thresholds) and forwards them to `MpptIcManager`, and accepts the panel deployment command and forwards it to `DeployPanelsManager`.

Unlike other Layer 3 components, `EPSApplication` has no mode interface — it operates continuously regardless of satellite mode.

---

## 2. Requirements

TODO

---

## 3. Design

### 3.1 Component Type

Active component. No hierarchical state machine — `EPSApplication` operates continuously and is fully driven by rate group ticks and incoming commands with no satellite-mode-driven state transitions.

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | Rate group tick (1 Hz) |
| `cmdIn` | Input | `Fw.Cmd` | Ground commands via CmdDispatcher |
| `cmdResponseOut` | Output | `Fw.CmdResponse` | Command completion status |
| `batteryStateIn` | Input | Custom struct port | Battery and IC state from MpptIcManager (vbatt, ibatt, vac, iac, charging status, fault flags, MPPT state, temperature) — type TBD |
| `powerStateOut` | Output | Custom struct port | EPS health/status to SatStateMachine (vbatt, ibatt, MPPT status, fault flags, charging status, temperature) — type TBD |
| `chargingControl` | Output | `Fw.On` | Enable/disable charging on MpptIcManager |
| `mpptControl` | Output | `Fw.On` | Enable/disable MPPT on MpptIcManager |
| `setChargingCurrentLimit` | Output | Custom port (U32 mA) | Forwarded to MpptIcManager — type TBD |
| `setJeitaThreshold[4]` | Output | Custom port | JEITA T1/T2/T3/T5 threshold writes to MpptIcManager — type TBD |
| `setRechargeThreshold` | Output | Custom port | Recharge threshold write to MpptIcManager — type TBD |
| `watchdogControl` | Output | `Fw.On` | Enable/disable BQ25756 internal watchdog on MpptIcManager |
| `deploy` | Output | `Fw.Signal` | Trigger deployment sequence on DeployPanelsManager |
| `pingIn` / `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (vbatt, ibatt, charging status, fault flags) |
| `prmGet` | Output | `Fw.PrmGet` | Load power threshold parameters from PrmDb |

### 3.3 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `ENABLE_CHARGING` | — | Forward charging enable to MpptIcManager |
| `DISABLE_CHARGING` | — | Forward charging disable to MpptIcManager |
| `ENABLE_MPPT` | — | Forward MPPT enable to MpptIcManager |
| `DISABLE_MPPT` | — | Forward MPPT disable to MpptIcManager |
| `SET_CHARGING_CURRENT_LIMIT` | `limit: U32` (mA) | Forward charging current limit write to MpptIcManager |
| `SET_JEITA_T1` | `threshold: U32` | Forward JEITA T1 threshold write to MpptIcManager |
| `SET_JEITA_T2` | `threshold: U32` | Forward JEITA T2 threshold write to MpptIcManager |
| `SET_JEITA_T3` | `threshold: U32` | Forward JEITA T3 threshold write to MpptIcManager |
| `SET_JEITA_T5` | `threshold: U32` | Forward JEITA T5 threshold write to MpptIcManager |
| `SET_RECHARGE_THRESHOLD` | `threshold: U32` | Forward recharge threshold write to MpptIcManager |
| `ENABLE_WATCHDOG` | — | Enable BQ25756 internal watchdog via MpptIcManager |
| `DISABLE_WATCHDOG` | — | Disable BQ25756 internal watchdog via MpptIcManager |
| `DEPLOY_PANELS` | — | Trigger panel deployment sequence via DeployPanelsManager |

---

## 4. Operational Behavior

`EPSApplication` does not use a hierarchical state machine. Each 1 Hz rate group tick it reads the latest battery state received from `MpptIcManager`, runs protection threshold checks, emits any required fault events, and pushes the assembled health struct to `SatStateMachine`. All command handlers are thin forwarders — they validate the request, call the appropriate output port, and return a command response.

**Rate group tick flow:**

```
schedIn fires (1 Hz)
  → read latest batteryState from MpptIcManager
  → check vbatt against POWER_THRESHOLD parameter
      if below WARNING threshold → log WARNING_HI
      if below CRITICAL threshold → log FATAL
  → assemble powerState struct
  → call powerStateOut to SatStateMachine
  → emit telemetry channels
```

**Command flow (all commands):**

```
cmdIn (ground or SatStateMachine)
  → validate arguments
  → call corresponding output port to MpptIcManager or DeployPanelsManager
  → emit activity event
  → send cmdResponse (OK or EXECUTION_ERROR)
```

---

## 5. Notes

- `EPSApplication` does not autonomously enable or disable MPPT or charging. All such changes require an explicit command from ground or `SatStateMachine`. However, it may in the future autonomously adjust charging thresholds (e.g. JEITA limits) based on received battery state data — this could require an internal state machine and is TBD pending further design.
- There is no command refusal logic at this time — `EPSApplication` forwards all commands to `MpptIcManager` or `DeployPanelsManager` unconditionally without checking current IC state.
- `batteryStateIn` port type is a custom struct carrying all BQ25756 measurement and status data; final type to be resolved during detailed design. Consider splitting into a measurements port and a flags port if the struct becomes unwieldy.
- `powerStateOut` port type is a custom struct; must carry at minimum vbatt, ibatt, MPPT status, fault flags, and charging status for `SatStateMachine` submode evaluation.
- The current command set uses one command per BQ25756 register write (e.g., `SET_JEITA_T1`, `SET_JEITA_T2`, `SET_CHARGING_CURRENT_LIMIT`, etc.), which produces a large number of commands and forwarding ports. A preferred alternative is a single `SET_IC_REGISTER` command taking a register address and value, which would significantly reduce the port count. To preserve readability, register addresses should be expressed as a named enum (e.g., `BQ25756Reg::JEITA_T1`, `BQ25756Reg::ICHG_LIMIT`) rather than raw hex values. This consolidation is deferred to detailed design.
- `MpptIcManager`, `HardwareResetManager`, `WatchdogPinger`, and `DeployPanelsManager` are all instantiated within the EPS subtopology. `EPSApplication` is health-monitored; the hardware managers are not.
- Power threshold parameters (`POWER_THRESHOLD`, `CRITICAL_THRESHOLD`) persisted via `PrmDb`. Specific threshold values and actions TBD pending battery characterization testing.
