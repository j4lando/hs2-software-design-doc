# AdcsApplication SDD

## 1. Overview

`AdcsApplication` is the Layer 3 Active component for the ADCS subsystem. It owns the attitude control loop and switches operating mode on command from `SatStateMachine`. It dispatches sensor requests and actuator commands to Layer 2 hardware managers (`ImuManager`, `SunSensorManager`, `MagnetorquerManager`) and consumes attitude from `StarTrackerManager` and timing from `GnssManager` (both top-level).

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|-------------|
| HS2-ADC-001 | AdcsApplication shall detumble the spacecraft using IMU and magnetometer data | Inspection |
| HS2-ADC-002 | AdcsApplication shall point solar panels at the sun using sun sensors and IMU | Inspection |
| HS2-ADC-003 | AdcsApplication shall point the antenna at the ground station using GNSS and IMU | Inspection |
| HS2-ADC-004 | AdcsApplication shall point the FOUND camera at the lit Earth limb while optimizing solar panel position | Inspection |
| HS2-ADC-005 | AdcsApplication shall hold attitude using IMU during eclipse when no active pointing is required | Inspection |
| HS2-ADC-006 | AdcsApplication shall switch operating mode on command from SatStateMachine without interrupting safe hardware state | Inspection |
| HS2-ADC-007 | AdcsApplication shall provide current attitude estimate as a telemetry channel | Inspection |
| HS2-ADC-008 | AdcsApplication shall assert WARNING_HI if IMU health check fails | Inspection |
| HS2-ADC-009 | AdcsApplication shall assert WARNING_HI if magnetorquer actuation fails | Inspection |
| HS2-ADC-010 | AdcsApplication shall respond to health ping within the required deadline | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal hierarchical F' state machine (`Fw::Sm`).

### 3.2 Mode Interface

`AdcsApplication` receives its operating mode from `SatStateMachine` via a typed port:

```fpp
sync input port modeIn: Sat.AdcsModePort   # carries Adcs.Mode
```

Mode enum (owned by this component's module):

```fpp
module Adcs {
    enum Mode { Off, Detumble, SunPointing, AntennaPointing, EarthLimbPointing, AttitudeHold }
}
```

If the incoming mode matches the current mode, the handler returns immediately (idempotent).

### 3.3 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `modeIn` | Input | `Sat.AdcsModePort` | Mode command from SatStateMachine |
| `schedIn` | Input | `Svc.Sched` | Rate group tick (10 Hz) |
| `imuDataIn` | Input | `Adcs.ImuDataPort` | Angular rate + linear acceleration sample pushed by `ImuManager` each tick |
| `sunVectorIn` | Input | `Adcs.SunVectorPort` | Body-frame sun unit vector + validity flag pushed by `SunSensorManager` each tick |
| `attitudeIn` | Input | `Adcs.AttitudePort` | Attitude estimate pushed by `StarTrackerManager` (top-level) |
| `positionIn` | Input | `Sat.PositionPort` | Position + timing pushed by `GnssManager` (top-level) |
| `dipoleCmdOut` | Output | `Adcs.DipoleCmdPort` | Target dipole command to `MagnetorquerManager` |
| `pingIn` / `pingOut` | In/Out | `Svc.Ping` | Health monitoring |
| `prmGet` | Output | `Fw.PrmGet` | Load control-loop parameters from PrmDb |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (attitude estimate, mode, error state) |

### 3.4 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `RESET` | — | Force transition back to current mode's initial substate |

---

## 4. State Machine

`AdcsApplication` uses a hierarchical F' state machine. Mode is the top-level state; operational substates are nested inside each mode. A single `switchMode: Adcs.Mode` signal defined at the top level is inherited by all leaf states, making mode switches valid from any nested substate.

Entry/exit actions follow the FPP Least Common Ancestor rule. Every mode re-entry starts from its `initial` substate — no history.

```
OFF

DETUMBLE
  └─ RUNNING         (on tick: run B-dot algorithm, command magnetorquers)

SUN_POINTING
  ├─ ACQUIRING       (on tick: search for sun vector via sun sensors)
  └─ TRACKING        (on tick: maintain sun-pointing, command magnetorquers)

ANTENNA_POINTING
  ├─ ACQUIRING       (on tick: compute ground station vector via GNSS ephemeris)
  └─ TRACKING        (on tick: maintain antenna pointing throughout pass)

EARTH_LIMB_POINTING
  ├─ ACQUIRING       (on tick: acquire lit Earth limb target + solar optimization)
  └─ TRACKING        (on tick: maintain Earth limb pointing, optimize panels)

ATTITUDE_HOLD
  └─ HOLDING         (on tick: hold current attitude using IMU, command magnetorquers)

# Inherited by all leaf states:
on switchMode(Adcs.Mode.Off)               enter OFF
on switchMode(Adcs.Mode.Detumble)          enter DETUMBLE
on switchMode(Adcs.Mode.SunPointing)       enter SUN_POINTING
on switchMode(Adcs.Mode.AntennaPointing)   enter ANTENNA_POINTING
on switchMode(Adcs.Mode.EarthLimbPointing) enter EARTH_LIMB_POINTING
on switchMode(Adcs.Mode.AttitudeHold)      enter ATTITUDE_HOLD
```

Reference: [FPP inherited transitions](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#inherited-transitions), [FPP substates](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#substates)

---

## 5. Notes

- `StarTrackerManager` and `GnssManager` are top-level components shared with `DataCollectionApplication`; connections are wired at the top-level topology.
- Hardware managers (`ImuManager`, `SunSensorManager`, `MagnetorquerManager`) are instantiated inside the ADCS subtopology.
- `EarthLimbPointing` uses star tracker for precision attitude knowledge — `StarTrackerManager` must be operational.
- Mid-operation mode switch behavior (e.g., mode switch arriving mid-maneuver) to be defined during detailed design.
