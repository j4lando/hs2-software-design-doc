# MagnetorquerManager SDD

## 1. Overview

`MagnetorquerManager` is the Layer 2 hardware manager for the magnetorquer coils within the ADCS subtopology. It is an **actuator manager** — unlike the sensor managers, its primary data flow is *inbound* from `AdcsApplication`. It receives a target magnetic dipole command, and on each rate-group tick drives the corresponding current through the X, Y, and Z magnetorquer coils.

`MagnetorquerManager` is a **Queued** component driven by `RateGroup1` (10 Hz). It has no satellite mode awareness. It stores the most recently received dipole command and re-applies it on every tick while in `RUN`, ensuring continuous actuation without requiring `AdcsApplication` to retransmit each cycle. `AdcsApplication` (Layer 3) is responsible for computing dipole commands and, if `MagnetorquerManager` reports repeated failures, commanding a power cut via EPS.

The magnetorquer coils are driven through a GPIO or I2C H-bridge current driver (bus assignment fixed at integration time). All bus access goes through the Layer 1 `LinuxGpioDriver` or `LinuxI2cDriver`.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-MTQ-001 | MagnetorquerManager shall reset the coil driver to a known safe state (zero current on all axes) before any configuration is applied | Inspection |
| HS2-MTQ-002 | MagnetorquerManager shall wait for the driver reset stabilization period before proceeding to enable | Inspection |
| HS2-MTQ-003 | MagnetorquerManager shall enable the H-bridge driver for all three coil axes before entering the run loop | Inspection |
| HS2-MTQ-004 | MagnetorquerManager shall configure maximum dipole moment limits and PWM frequency from F' parameters before entering the run loop | Inspection |
| HS2-MTQ-005 | MagnetorquerManager shall zero all coil currents on entry to RESET to ensure safe actuator state during recovery | Inspection |
| HS2-MTQ-006 | MagnetorquerManager shall accept dipole commands from AdcsApplication via an async input port while in RUN | Inspection |
| HS2-MTQ-007 | MagnetorquerManager shall apply the most recently received dipole command to the coil driver each rate-group tick while in RUN | Inspection |
| HS2-MTQ-008 | MagnetorquerManager shall clamp commanded dipole moment to the MAX_DIPOLE_MOMENT parameter before writing to the driver | Inspection |
| HS2-MTQ-009 | MagnetorquerManager shall log a WARNING_HI event and transition to RESET on any bus write failure | Inspection |
| HS2-MTQ-010 | MagnetorquerManager shall track consecutive failure count and report it as a telemetry channel | Inspection |
| HS2-MTQ-011 | MagnetorquerManager shall apply updated F' parameter values by transitioning from RUN to CONFIGURE without a full reset | Inspection |
| HS2-MTQ-012 | MagnetorquerManager shall not perform any bus operations while in WAIT_RESET | Inspection |

---

## 3. Design

### 3.1 Component Type

Queued component with internal flat F' state machine (`Fw::Sm`). Has a message queue but no dedicated thread — executes on the `RateGroup1` caller thread each 10 Hz tick. The `dipoleCmdIn` async input port queues incoming commands for dispatch on the component's next execution window.

### 3.2 Parameters

`MagnetorquerManager` is not satellite-mode-aware and receives no mode commands from `SatStateMachine`. All coil driver configuration is parameter-driven. Parameters are loaded from `Svc::PrmDb` in the `CONFIGURE` state. A parameter update at runtime sends a `reconfigure` signal, causing `MagnetorquerManager` to re-enter `CONFIGURE` from `RUN` without a full reset; coil current is zeroed during the transition.

| Parameter | Type | Description |
|-----------|------|-------------|
| `MAX_DIPOLE_MOMENT` | `F32` | Maximum allowable dipole moment magnitude per axis (A·m²) — commands are clamped to this |
| `PWM_FREQUENCY` | `U32` | PWM carrier frequency for the H-bridge driver (Hz) |
| `AXIS_POLARITY[3]` | `I8[3]` | Per-axis polarity sign (+1 or -1) to account for physical coil winding direction |
| `RESET_WAIT_TICKS` | `U32` | Number of rate-group ticks to wait after reset before proceeding |

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | 10 Hz rate group tick — drives the SM step and applies stored dipole command |
| `dipoleCmdIn` | Input (async) | `Adcs.DipoleCmdPort` | Receives target dipole vector (X, Y, Z in A·m²) from AdcsApplication |
| `busWrite` | Output | `Drv.GpioWrite` | Write PWM duty cycle and direction signals to H-bridge driver |
| `busWriteI2c` | Output | `Drv.I2cWrite` | Write to I2C current driver (if coil driver uses I2C rather than GPIO) |
| `prmGet` | Output | `Fw.PrmGet` | Load parameters from PrmDb during CONFIGURE |
| `logOut` | Output | `Fw.Log` | Event logging (state transitions, clamping events, errors) |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (SM state, consecutive failure count, applied dipole per axis) |

> **Bus type:** If the H-bridge uses GPIO PWM signals, `busWrite` maps to `LinuxGpioDriver`. If the driver uses an I2C current controller, `busWriteI2c` maps to `LinuxI2cDriver`. Only one of the two output ports is connected at topology assembly depending on the physical interface. The unused port is left unconnected.

### 3.4 Commands

`MagnetorquerManager` accepts no ground commands. It is not health-monitored. All recovery is handled autonomously by the self-healing SM or escalated via telemetry to `AdcsApplication`.

---

## 4. State Machine

`MagnetorquerManager` uses a single flat F' state machine following the `fprime-community/fprime-sensors` `ImuManager` reference pattern. All states are peers — no nesting. The SM is stepped by `schedIn` each tick. Dipole commands may arrive via `dipoleCmdIn` at any time and are buffered in the component's internal queue; they are processed and stored during the next `schedIn` dispatch.

```
RESET
  entry: write zero current to all three coil axes (safe actuator state)
         reset consecutive-failure counter
         reset wait-tick counter
         clear stored dipole command (zero vector)
  on tick → WAIT_RESET

WAIT_RESET
  on tick: increment wait-tick counter
           if counter >= RESET_WAIT_TICKS → ENABLE
           (no bus operations performed in this state)
           (dipole commands received in this state are accepted into queue
            but not applied — stored for use once RUN is reached)

ENABLE
  entry: write enable command to H-bridge driver (enable all axes, set PWM frequency)
  on tick:
    if busWrite OK → CONFIGURE
    if busWrite error → log WARNING_HI, increment failure count → RESET

CONFIGURE
  entry: zero coil currents
         load MAX_DIPOLE_MOMENT, PWM_FREQUENCY, AXIS_POLARITY from PrmDb
         write PWM frequency configuration register to driver
         write axis polarity configuration
  on tick:
    if all writes OK → RUN
    if any write error → log WARNING_HI, increment failure count → RESET

RUN
  on dipoleCmdIn: clamp each axis to ± MAX_DIPOLE_MOMENT
                  apply AXIS_POLARITY sign per axis
                  store as current command
  on tick: write stored dipole command to coil driver (all three axes)
           if busWrite OK:
             emit applied dipole as telemetry
             reset consecutive-failure counter
           if busWrite error:
             log WARNING_HI (throttled), increment failure count → RESET

# From any state:
on error signal → RESET        # self-healing fallback
on reconfigure signal → CONFIGURE   # parameter update path from RUN
```

**Safe zero-current on RESET:** Every entry to `RESET` immediately zeroes all coil axes before doing anything else. This ensures the magnetorquers are not left producing uncontrolled torque during the recovery cycle, even if the reset was triggered mid-actuation.

**Dipole command buffering:** `dipoleCmdIn` is an async input port. Commands may arrive between ticks and are queued. During `RUN`, each `dipoleCmdIn` dispatch updates the stored command; the stored value is applied to hardware on the next `schedIn` tick. If multiple commands arrive between ticks, only the most recently dispatched value is applied (prior queued commands are processed and overwrite each other in order).

**Clamping:** `AdcsApplication` is the authority on dipole magnitude, but `MagnetorquerManager` enforces the hardware-safe `MAX_DIPOLE_MOMENT` clamp as a protective backstop. Clamping events are logged at `DIAGNOSTIC` severity.

**Error self-healing:** Any bus write failure from `ENABLE`, `CONFIGURE`, or `RUN` emits a throttled `WARNING_HI` event, increments `consecutiveFailures`, zeros coil currents, and re-enters `RESET`. The SM retries the full startup sequence automatically.

**Repeated failure escalation:** `MagnetorquerManager` does not cut power to itself. If `consecutiveFailures` exceeds a threshold observable by `AdcsApplication` via telemetry, `AdcsApplication` is responsible for commanding an EPS power rail cut. `MagnetorquerManager` continues attempting self-healing until then.

**Parameter reconfiguration:** When `parameterUpdated()` is called, `MagnetorquerManager` sends a `reconfigure` signal. If currently in `RUN`, this transitions directly to `CONFIGURE` — coil currents are zeroed first, new parameters loaded, then `RUN` is re-entered. From any other state the signal is ignored.

Reference: [`fprime-community/fprime-sensors/ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager), [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc)

---

## 5. Notes

- `MagnetorquerManager` is instantiated inside the ADCS subtopology. Its `busWrite` / `busWriteI2c` port connects to a `LinuxGpioDriver` or `LinuxI2cDriver` instance at the top-level topology, depending on the H-bridge interface.
- `dipoleCmdIn` is connected from `AdcsApplication` within the ADCS subtopology. `AdcsApplication` sends commands at its own control loop rate (10 Hz); `MagnetorquerManager` re-applies the stored command each tick regardless of whether a new one arrived.
- `MagnetorquerManager` is **excluded from health monitoring** (`Svc::Health`). Only `AdcsApplication` is health-checked.
- The `Adcs.DipoleCmdPort` type (carrying a three-axis dipole vector in A·m² and a timestamp) is defined in the ADCS module and shared with `AdcsApplication`.
- Physical coil drive details (PWM duty cycle mapping to current, H-bridge register layout) are implementation details for the component's C++ source, not specified here.
- `MagnetorquerManager` produces torque-related telemetry (applied dipole per axis) but does **not** measure actual coil current. Current sensing, if available on the hardware, would be a separate driver concern.
- Deferred: exact `consecutiveFailures` threshold that triggers `AdcsApplication` to cut magnetorquer power is a system-level parameter to be defined during detailed design.
