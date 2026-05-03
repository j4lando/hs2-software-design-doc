# HardwareResetManager SDD

## 1. Overview

`HardwareResetManager` is a Layer 2 Queued component that monitors the current-sense signals for each protected power rail. On each rate group tick it reads the current sense value for each monitored component; if any rail exceeds its hardcoded overcurrent threshold, it asserts the corresponding reset GPIO and emits overcurrent and reset events.

`HardwareResetManager` operates independently of `EPSApplication` and has no satellite mode awareness.

---

## 2. Requirements

TODO

---

## 3. Design

### 3.1 Component Type

Active component. No state machine — behavior is fully determined by rate group ticks with no initialization sequence or recovery states.

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | Rate group tick; triggers current-sense read cycle |
| `currentSenseIn[N]` | Input | TBD — `Drv.GpioRead`, I2C, or ADC port | One port per monitored power rail; exact interface TBD pending hardware confirmation |
| `resetOut[N]` | Output | `Drv.GpioWrite` | One per monitored component; asserted when that rail exceeds its overcurrent threshold |
| `logOut` | Output | `Fw.Log` | Event logging |

### 3.3 Commands

None.

---

## 4. Operational Behavior

On each `schedIn` tick, `HardwareResetManager` reads the current-sense input for each monitored power rail. For any rail exceeding its hardcoded overcurrent threshold, it asserts the corresponding `resetOut` GPIO, emits a `WARNING_HI` overcurrent event (including rail ID and measured value), and emits a reset-asserted event.

---

## 5. Notes

- Overcurrent thresholds are currently hardcoded constants in the component implementation. However, it might make sense to have these as parameters stored in a parameter database, not entirely sure.
- Current-sense interface is hardware-dependent and unconfirmed. At least one current-sense IC may use I2C; others may use GPIO or analog (requiring an ADC driver). Interface type to be confirmed with the hardware team.
- `HardwareResetManager` should probably report reset events to `EPSApplication`, but this isn't currently in the specification. Whether a notification port is needed is deferred to detailed design. to have a component that is not the sat state machine with the power to turn off components. We may want a dedicated `OffManager` that takes requests/complaints from many different sources, and tries to reset components in various different ways (i.e. sending an off command, then pulling a reset pin high, then shutting off power. This last one may not be a possibility for many components due to hardware limitations.)
- Number of monitored rails (N) TBD pending hardware finalization.
