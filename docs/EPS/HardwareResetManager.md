# HardwareResetManager SDD

## 1. Overview

`HardwareResetManager` is a Layer 2 Passive component. It receives a fault notification from a fault-counting component in CdhCore after `MpptIcManager` has emitted a threshold number of consecutive `WARNING_HI` events about a specific power rail or voltage line. On receipt it emits a hardware-reset event and calls `SatStateMachine` to report the fault so the satellite can take appropriate action.

`HardwareResetManager` has no satellite mode awareness and no state machine.

---

## 2. Requirements

TODO

---

## 3. Design

### 3.1 Component Type

Passive component. No thread, no queue. Called synchronously by the CdhCore fault-counter on its thread.

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `hardwareFaultIn` | Input | Custom port (line/component ID) | Called by CdhCore fault-counter after N consecutive warnings about a rail; carries identifier of the affected line |
| `faultNotifyOut` | Output | Custom port (line/component ID) | Notifies `SatStateMachine` of the hardware fault; carries same identifier |
| `logOut` | Output | `Fw.Log` | Event logging |

### 3.3 Commands

None.

---

## 4. Operational Behavior

On `hardwareFaultIn` call: emit `WARNING_HI` hardware-fault event (including line/component ID), then call `faultNotifyOut` to notify `SatStateMachine`.

---

## 5. Notes

- The fault-counting mechanism in CdhCore — tracking N consecutive `WARNING_HI` events from `MpptIcManager` about a specific rail — requires a component with that logic. No standard F' service component provides event-count-based triggering; a custom fault-counter component may be needed, or the count can be tracked inside `MpptIcManager` directly. To be resolved during detailed design.
- Whether `HardwareResetManager` also asserts a hardware GPIO reset line for the affected component (before or after notifying `SatStateMachine`) is TBD pending hardware team confirmation of which rails have software-controllable reset pins.
- `SatStateMachine` is responsible for deciding the response to the fault notification — e.g., entering Safe mode or issuing a component recovery command.
- Number and identity of monitored rails (N) TBD pending hardware finalization.
