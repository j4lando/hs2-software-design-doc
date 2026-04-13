# HS2 Satellite Flight Software — Software Design Document
**Date:** 2026-04-12
**Framework:** F' (F Prime) — `nasa/fprime@devel`
**Platform:** 3U CubeSat, single flight computer

---

## 1. Mission Overview

HS2 is a 3U CubeSat scientific mission validating two optical navigation algorithms:
- **LOST** — runs on Camera 1; included as an external C++ library via CMake
- **FOUND** — runs on Camera 2; included as an external C++ library via CMake

The satellite uses an EnduroSat S-band radio for ground communications using the F' native protocol. Science results are always recorded as F' data products. Raw images are stored to external flash only when flagged by the science algorithms. The flight software is implemented in F' and organized into custom and pre-built subtopologies.

---

## 2. Hardware Inventory

| Hardware | Interface | Primary User |
|----------|-----------|--------------|
| Camera 1 | SPI/I2C | Science (LOST algorithm) |
| Camera 2 | SPI/I2C | Science (FOUND algorithm) |
| Star Tracker | UART | Science (synchronized capture), ADCS (precision pointing) |
| aGNSS Receiver | UART | Science (synchronized capture), ADCS (position/timing) |
| IMU | SPI/I2C | ADCS |
| Sun Sensors | I2C/GPIO | ADCS |
| Magnetorquers | GPIO/I2C | ADCS |
| EPS Board | I2C/UART | EPS subtopology, SatStateMachine |
| EnduroSat S-band Radio | UART | ComFprime subtopology |
| External Flash | SPI | FileHandling subtopology |
| Temperature Sensors | I2C | Thermal subtopology |
| Heater | GPIO | Thermal subtopology |

---

## 3. Operational Modes

The satellite operates in three main modes managed by `SatStateMachine`. Uplink commands are accepted in all modes.

| Mode | Description |
|------|-------------|
| **Checkout** | Initial lifecycle phase. All subsystems verified operational. Submodes TBD. |
| **Standby** | Nominal operations. Runs Charge, Downlink, and Science submodes autonomously based on conditions. |
| **Safe** | Low-power survival mode. Minimal operations. Uplink always active. |

### Main Mode Transitions

| From | To | Trigger |
|------|----|---------|
| Checkout | Standby | Ground command: `CHECKOUT_COMPLETE` |
| Any | Safe | Ground command `SAFE_MODE` or EPS `FATAL` low-power event |
| Safe | Standby | Ground command: `SAFE_EXIT` |

### Standby Submodes

Standby has no separate Idle state — Idle is the base condition from which all submodes inherit. Submodes activate and deactivate autonomously based on real-time conditions.

| Submode | Activation Conditions | Description |
|---------|-----------------------|-------------|
| **Charge** | Sun present AND Downlink not active AND Science not active | Points solar panels at sun to maximize power generation |
| **Downlink** | Enough power AND passing over ground station | Downlinks telemetry, events, data products, and files to ground |
| **Science** | Downlink not active AND enough power AND `EXPERIMENT_ENABLED` parameter set by ground | Runs LOST and FOUND algorithms opportunistically |

**Submode priority:** Downlink takes priority over Science. Charge activates in the remaining windows when neither Downlink nor Science is active.

**Key parameters** (configurable, persisted via `PrmDb`):
- `POWER_THRESHOLD` — minimum EPS state of charge required for Downlink or Science
- `EXPERIMENT_ENABLED` — ground-set boolean enabling the Science submode

---

## 4. Architecture Overview

The flight software uses a **three-layer App-Manager-Driver architecture**:

```
Layer 3 — Mission Subtopologies (application logic)
    Science | Adcs | Eps | Thermal
    + pre-built: CdhCore | ComFprime | FileHandling | DataProducts

Layer 2 — Hardware Managers (active workers with internal state machines)
    Camera1Manager | Camera2Manager | StarTrackerManager | GnssManager
    ImuManager | SunSensorManager | MagnetorquerManager
    EpsManager | TemperatureSensorManager | HeaterManager | EnduroSatManager

Layer 1 — F' Native Bus Drivers (hardware-only)
    LinuxI2cDriver | LinuxSpiDriver | LinuxUartDriver | LinuxGpioDriver
```

All hardware managers are **Active components** with internal F' state machines (`Fw::Sm`). They act as workers in the manager-worker pattern — application-level components dispatch work to them and receive completion callbacks.

Each hardware manager state machine includes a **startup protocol phase** (PowerOn → device-specific initialization sequence → Idle) before entering operational states. The specific startup protocols for each device are to be defined during detailed design once hardware datasheets and vendor SDKs are available.

`StarTrackerManager`, `GnssManager`, and `EnduroSatManager` are instantiated at the **top-level topology** because they are shared across multiple subtopologies. All other hardware managers are instantiated inside their primary subtopology.

---

## 5. Subtopology Decomposition

### 5.1 Pre-Built Subtopologies (imported without modification)

| Subtopology | Source | Purpose |
|-------------|--------|---------|
| `CdhCore` | `Svc/Subtopologies/CdhCore` | CmdDispatcher, EventManager, Health, Version, AssertFatalAdapter |
| `ComFprime` | `Svc/Subtopologies/ComFprime` | F' native protocol framing, uplink/downlink pipeline |
| `FileHandling` | `Svc/Subtopologies/FileHandling` | FileUplink, FileDownlink, FileManager, PrmDb |
| `DataProducts` | `Svc/Subtopologies/DataProducts` | DpManager, DpWriter, DpCatalog for science results |

### 5.2 Science Subtopology

**Purpose:** Manages synchronized image capture and execution of the LOST and FOUND optical navigation algorithms.

**Components:**

| Component | Type | Purpose |
|-----------|------|---------|
| `ScienceManager` | Active (high priority) | Orchestrates synchronized data capture; tracks busy state; dispatches bundles to workers; records data products; flags images for storage |
| `LostWorker` | Active (low priority) | Calls LOST library with Camera 1 image + attitude + position bundle; reports results as data products |
| `FoundWorker` | Active (low priority) | Calls FOUND library with Camera 2 image + attitude + position bundle; reports results as data products |
| `Camera1Manager` | Active (worker) | State machine: [Startup] → Idle → Capturing → Readout → Idle / Error |
| `Camera2Manager` | Active (worker) | State machine: [Startup] → Idle → Capturing → Readout → Idle / Error |

**Synchronized Capture Requirement:**
`ScienceManager` must simultaneously acquire all of the following before dispatching work:
1. Image from `Camera1Manager` (for LOST)
2. Image from `Camera2Manager` (for FOUND)
3. Attitude reading from `StarTrackerManager` (top-level)
4. Position/time reading from `GnssManager` (top-level)

This uses the F' **callback port pattern** — `ScienceManager` sends requests to all four, collects responses, then dispatches complete data bundles to `LostWorker` and `FoundWorker` in parallel.

**Science data outputs:**
- Algorithm results → always stored via `DataProducts` subtopology
- Flagged raw images → stored to flash via `FileHandling` subtopology

**Health monitoring:** `ScienceManager` is health-monitored. Workers are excluded per F' health checking pattern.

**Ports consumed from outside subtopology:**
- `StarTrackerManager` attitude port (top-level)
- `GnssManager` position/time port (top-level)
- `DpManager` (from `DataProducts` subtopology)
- `DpWriter` (from `DataProducts` subtopology)
- `FileDownlink` (from `FileHandling` subtopology)

### 5.3 ADCS Subtopology

**Purpose:** Attitude determination and control across all three ADCS operating modes.

**Components:**

| Component | Type | Purpose |
|-----------|------|---------|
| `AdcsManager` | Active (high priority) | Owns attitude control loop; switches ADCS mode based on `SatStateMachine` commands |
| `ImuManager` | Active (worker) | State machine: [Startup] → Idle → Sampling → Idle / Error |
| `SunSensorManager` | Active (worker) | State machine: [Startup] → Idle → Sampling → Idle / Error |
| `MagnetorquerManager` | Active (worker) | State machine: [Startup] → Idle → Actuating → Idle / Error |

**ADCS Modes (internal to `AdcsManager`):**

| Mode | Sensors | Actuators |
|------|---------|-----------|
| Detumble | IMU, Sun Sensors | Magnetorquers |
| Nominal | IMU, Sun Sensors | Magnetorquers |
| Precision Pointing | IMU, Sun Sensors, Star Tracker | Magnetorquers |

**Ports consumed from top-level:**
- `StarTrackerManager` attitude port (precision pointing mode)
- `GnssManager` position/time port (timing reference)
- `SatStateMachine` mode command port

**Health monitoring:** `AdcsManager` is health-monitored. Hardware managers excluded.

### 5.4 EPS Subtopology

**Purpose:** Monitors battery state of charge and exposes power availability to `SatStateMachine`.

**Components:**

| Component | Type | Purpose |
|-----------|------|---------|
| `EpsManager` | Active (worker) | State machine: [Startup] → Idle → Reading → LowPower / Error. Publishes SoC as telemetry channel; raises `WARNING_HI` at low threshold, `FATAL` at critical threshold |

**Ports exposed out of subtopology:**
- `powerStateOut` → consumed by `SatStateMachine` for Standby submode activation decisions

**Health monitoring:** `EpsManager` is health-monitored.

### 5.5 Thermal Subtopology

**Purpose:** Monitors temperature sensors and commands heater to maintain operating temperature range.

**Components:**

| Component | Type | Purpose |
|-----------|------|---------|
| `ThermalManager` | Active | Polls temperature sensors; commands heater based on configurable F' parameter thresholds; raises events on out-of-range readings |
| `TemperatureSensorManager` | Active (worker) | State machine: [Startup] → Idle → Sampling → Idle / Error |
| `HeaterManager` | Active (worker) | State machine: [Startup] → Off → HeatingUp → On → CoolingDown / Error |

**Health monitoring:** `ThermalManager` is health-monitored. Hardware managers excluded.

---

## 6. Top-Level Standalone Components

These components are instantiated at the top-level topology and are not inside any subtopology.

| Component | Type | Purpose |
|-----------|------|---------|
| `SatStateMachine` | Active | Owns the 3-mode satellite state machine with Standby submodes (Charge, Downlink, Science); evaluates submode conditions each 1 Hz tick |
| `StarTrackerManager` | Active (worker) | State machine: [Startup] → Idle → Acquiring → Tracking → Idle / Error. Shared by Science and ADCS |
| `GnssManager` | Active (worker) | State machine: [Startup] → Idle → Acquiring → Tracking → Idle / Error. Shared by Science, ADCS, and time services |
| `EnduroSatManager` | Active (worker) | State machine: [Startup] → Idle → Transmitting / Receiving → Idle / Error. Bridges ComFprime to the EnduroSat S-band radio |
| `RateGroupDriver` | Passive | Divides hardware timer interrupt into multiple rate signals |
| `RateGroup1` | Active | 10 Hz scheduling |
| `RateGroup2` | Active | 1 Hz scheduling |
| `RateGroup3` | Active | 0.1 Hz scheduling |
| `LinuxI2cDriver` (×N) | Passive | One instance per I2C bus |
| `LinuxSpiDriver` (×N) | Passive | One instance per SPI bus |
| `LinuxUartDriver` (×N) | Passive | One instance per UART (star tracker, GNSS, radio) |
| `LinuxGpioDriver` (×N) | Passive | One instance per GPIO group (magnetorquers, heater) |

---

## 7. Rate Group Scheduling

| Rate Group | Frequency | Scheduled Components |
|------------|-----------|---------------------|
| `RateGroup1` | 10 Hz | `AdcsManager`, `ImuManager`, `SunSensorManager`, `MagnetorquerManager`, watchdog stroke |
| `RateGroup2` | 1 Hz | `SatStateMachine`, `EpsManager`, `ThermalManager`, `ScienceManager` (availability check), `Health` |
| `RateGroup3` | 0.1 Hz | `StarTrackerManager`, `GnssManager`, `SystemResources`, `FileDownlink` |

---

## 8. SatStateMachine Design

`SatStateMachine` is an Active component implementing the satellite state machine with `Fw::Sm`.

### Main Mode Transitions

| From | To | Trigger |
|------|----|---------|
| Checkout | Standby | Ground command: `CHECKOUT_COMPLETE` |
| Any | Safe | Ground command `SAFE_MODE` or `EpsManager` `FATAL` event |
| Safe | Standby | Ground command: `SAFE_EXIT` |

### Standby Submode Logic (evaluated each 1 Hz tick)

Idle is the base Standby state — not a named state, but the condition where no submode is active. All submodes inherit from it.

| Submode | Enter When | Exit When |
|---------|-----------|-----------|
| Downlink | Enough power AND over ground station | No longer over ground station OR power below threshold |
| Science | NOT Downlink AND enough power AND `EXPERIMENT_ENABLED` | Downlink activates OR power drops below threshold OR `EXPERIMENT_ENABLED` cleared |
| Charge | Sun present AND NOT Downlink AND NOT Science | Downlink or Science activates OR sun not present |

**Priority:** Downlink > Science > Charge.

### Parameters (persisted via `PrmDb`)

| Parameter | Description |
|-----------|-------------|
| `POWER_THRESHOLD` | Minimum EPS state of charge (%) for Downlink or Science activation |
| `EXPERIMENT_ENABLED` | Ground-set boolean enabling the Science submode |

**Events emitted:** main mode and submode entry/exit events for every transition.

**Health checked:** Yes.

---

## 9. Key Cross-Subtopology Wiring

| Source | Destination | Data |
|--------|-------------|------|
| `StarTrackerManager` | `ScienceManager` | Attitude reading (synchronized capture) |
| `StarTrackerManager` | `AdcsManager` | Attitude reading (precision pointing) |
| `GnssManager` | `ScienceManager` | Position + time (synchronized capture) |
| `GnssManager` | `AdcsManager` | Position + timing reference |
| `GnssManager` | Time services | PPS timing signal |
| `EpsManager.powerStateOut` | `SatStateMachine` | Battery state of charge |
| `SatStateMachine` | `AdcsManager` | Mode commands |
| `SatStateMachine` | `ScienceManager` | Mode commands |
| `EnduroSatManager` | `ComFprime` | Uplink/downlink byte stream |
| `Science` | `DataProducts` | Science result data products |
| `Science` | `FileHandling` | Flagged image files |

---

## 10. Health Monitoring Summary

Components enrolled in `Svc::Health` ping/pong monitoring:

| Component | Subtopology |
|-----------|-------------|
| `SatStateMachine` | Top-level |
| `ScienceManager` | Science |
| `AdcsManager` | ADCS |
| `EpsManager` | EPS |
| `ThermalManager` | Thermal |
| `cmdDisp` | CdhCore |
| `events` | CdhCore |

All hardware manager workers and science workers (`LostWorker`, `FoundWorker`) are excluded from health monitoring per the F' health checking pattern.

---

## 11. Design Pattern References

| Pattern | Applied To | F' Documentation |
|---------|-----------|-----------------|
| Manager-Worker | `ScienceManager` + `LostWorker` + `FoundWorker`; all hardware managers | `docs/user-manual/design-patterns/manager-worker.md` |
| App-Manager-Driver | All subsystems (Science, ADCS, EPS, Thermal) | `docs/user-manual/design-patterns/app-man-drv.md` |
| Subtopologies | Science, ADCS, EPS, Thermal, + 4 pre-built | `docs/user-manual/design-patterns/subtopologies.md` |
| Rate Groups | RateGroup1/2/3 driven by RateGroupDriver | `docs/user-manual/design-patterns/rate-group.md` |
| Health Checking | All application-level active components | `docs/user-manual/design-patterns/health-checking.md` |
| Callback Ports | Synchronized capture in ScienceManager | `docs/user-manual/design-patterns/common-port-patterns.md` |
| State Machines | SatStateMachine + all hardware managers | `docs/user-manual/framework/state-machines.md` |
| Data Products | Science algorithm results | `docs/user-manual/framework/data-products.md` |

---

## 12. External Library Integration

| Library | Algorithm | Camera | Integration |
|---------|-----------|--------|-------------|
| LOST | Optical navigation | Camera 1 | CMake; called from `LostWorker` |
| FOUND | Optical navigation | Camera 2 | CMake; called from `FoundWorker` |

Both libraries are C++ and included as CMake dependencies. Workers call the library entry points synchronously within their async port handlers, allowing the rest of the system to remain responsive during algorithm execution.
