# WatchdogPinger SDD

## 1. Overview

`WatchdogPinger` is a Layer 2 Passive component that satisfies the hardware watchdog on the EPS board by toggling a dedicated GPIO pin on each rate group tick. It has no satellite mode awareness and no state machine.

---

## 2. Requirements

TODO

---

## 3. Design

### 3.1 Component Type

Passive component. No queue, no thread, no state machine.

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | Rate group tick; triggers watchdog GPIO toggle |
| `watchdogPing` | Output | `Drv.GpioWrite` | GPIO pin connected to EPS board hardware watchdog |

### 3.3 Commands

None.

---

## 4. Operational Behavior

On each `schedIn` tick, `WatchdogPinger` asserts then deasserts `watchdogPing` to produce a pulse satisfying the EPS hardware watchdog timer. If this component stops being scheduled — due to a software hang or rate group slip — the hardware watchdog will expire and reset the system.

---

## 5. Notes

- Tick rate and GPIO pulse timing to be confirmed with the hardware team against the specific watchdog IC's timeout and minimum pulse width specifications.
- `WatchdogPinger` is scheduled via the standard F' rate group mechanism. The rate group must be selected such that its period is comfortably within the watchdog timeout window including any rate group slip margin.
