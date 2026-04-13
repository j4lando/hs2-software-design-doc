# Custom Subtopology SDD Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Write Software Design Documents (SDDs) for each of the four custom HS2 subtopologies (Science, Adcs, Eps, Thermal) in the same format used by the F' prebuilt subtopologies.

**Architecture:** Each SDD follows the CdhCore/ComCcsds template: overview paragraph, requirements table, design & core functions (instance summary, wiring, required inputs, limitations), usage with FPP example, configuration, and traceability matrix. Documents live at `docs/<SubtopologyName>/sdd.md` mirroring the eventual fprime source tree structure.

**Tech Stack:** Markdown, FPP (F Prime Prime modeling language), F' framework conventions from `nasa/fprime@devel`

**Reference format:** `Svc/Subtopologies/CdhCore/docs/sdd.md` and `Svc/Subtopologies/ComCcsds/docs/sdd.md`

---

## File Structure

| File | Purpose |
|------|---------|
| `docs/Science/sdd.md` | Science subtopology SDD |
| `docs/Adcs/sdd.md` | ADCS subtopology SDD |
| `docs/Eps/sdd.md` | EPS subtopology SDD |
| `docs/Thermal/sdd.md` | Thermal subtopology SDD |

---

## Task 1: Science Subtopology SDD

**Files:**
- Create: `docs/Science/sdd.md`

- [ ] **Step 1: Create the Science SDD file**

Create `docs/Science/sdd.md` with the following content:

````markdown
# Science Subtopology — Software Design Document (SDD)

The **Science subtopology** manages the synchronized acquisition of camera images, star tracker attitude readings, and GNSS position/time data, then executes the LOST and FOUND optical navigation algorithms as low-priority background workers. The subtopology ensures the main satellite remains responsive during algorithm execution by separating the high-priority `ScienceManager` (which orchestrates capture and tracks state) from two low-priority workers (`LostWorker`, `FoundWorker`) that call the external algorithm libraries. Algorithm results are always recorded as F' data products. Raw images are stored to flash only when flagged by an algorithm result.

## 1. Requirements

| ID | Description | Validation |
|----|-------------|------------|
| HS2-SCI-001 | The subtopology shall simultaneously capture images from Camera 1 and Camera 2 upon a science trigger. | Inspection |
| HS2-SCI-002 | The subtopology shall acquire an attitude reading from `StarTrackerManager` at the time of image capture. | Inspection |
| HS2-SCI-003 | The subtopology shall acquire a position and time reading from `GnssManager` at the time of image capture. | Inspection |
| HS2-SCI-004 | The subtopology shall dispatch a data bundle (Camera 1 image + attitude + position/time) to `LostWorker` for LOST algorithm execution. | Inspection |
| HS2-SCI-005 | The subtopology shall dispatch a data bundle (Camera 2 image + attitude + position/time) to `FoundWorker` for FOUND algorithm execution. | Inspection |
| HS2-SCI-006 | The subtopology shall record algorithm results as F' data products via the `DataProducts` subtopology. | Inspection |
| HS2-SCI-007 | The subtopology shall store raw images to flash via the `FileHandling` subtopology when flagged by an algorithm result. | Inspection |
| HS2-SCI-008 | `ScienceManager` shall remain responsive to health pings during algorithm execution. | Inspection |
| HS2-SCI-009 | `ScienceManager` shall track busy state and reject new work requests while workers are active. | Inspection |
| HS2-SCI-010 | The subtopology shall support configurable instance properties (IDs, queue sizes, stack sizes, priorities). | Inspection |

## 2. Design & Core Functions

### 2.1 Instance Summary

| Instance | Type | Kind | Purpose |
|----------|------|------|---------|
| `scienceManager` | `HS2.ScienceManager` | Active (high priority) | Orchestrates synchronized data capture using callback port pattern; tracks busy state; dispatches data bundles to workers; records data products; flags images for storage |
| `lostWorker` | `HS2.LostWorker` | Active (low priority) | Calls LOST external C++ library with Camera 1 data bundle; reports results via `workDone` port |
| `foundWorker` | `HS2.FoundWorker` | Active (low priority) | Calls FOUND external C++ library with Camera 2 data bundle; reports results via `workDone` port |
| `camera1Manager` | `HS2.CameraManager` | Active (worker) | State machine: [Startup] → Idle → Capturing → Readout → Idle / Error. Drives Camera 1 hardware via `LinuxSpiDriver` or `LinuxI2cDriver`. |
| `camera2Manager` | `HS2.CameraManager` | Active (worker) | State machine: [Startup] → Idle → Capturing → Readout → Idle / Error. Drives Camera 2 hardware via `LinuxSpiDriver` or `LinuxI2cDriver`. |

### 2.2 Synchronized Capture — Callback Port Pattern

`ScienceManager` uses the F' callback port pattern to simultaneously request data from four sources before dispatching work:

1. Image from `camera1Manager` (async request → completion callback)
2. Image from `camera2Manager` (async request → completion callback)
3. Attitude from `StarTrackerManager` (top-level, sync get or async callback)
4. Position/time from `GnssManager` (top-level, sync get or async callback)

`ScienceManager` collects all four responses before dispatching. `LostWorker` and `FoundWorker` run in parallel once dispatched.

### 2.3 Internal Wiring

| Connection | Description |
|------------|-------------|
| `scienceManager.startLost → lostWorker.startWork` | Dispatch Camera 1 bundle to LOST worker |
| `scienceManager.cancelLost → lostWorker.cancelWork` | Synchronous cancel for LOST worker |
| `lostWorker.workDone → scienceManager.lostDoneRecv` | LOST completion callback |
| `scienceManager.startFound → foundWorker.startWork` | Dispatch Camera 2 bundle to FOUND worker |
| `scienceManager.cancelFound → foundWorker.cancelWork` | Synchronous cancel for FOUND worker |
| `foundWorker.workDone → scienceManager.foundDoneRecv` | FOUND completion callback |
| `scienceManager.startCapture1 → camera1Manager.captureIn` | Trigger Camera 1 image capture |
| `camera1Manager.captureOut → scienceManager.capture1Done` | Camera 1 image ready callback |
| `scienceManager.startCapture2 → camera2Manager.captureIn` | Trigger Camera 2 image capture |
| `camera2Manager.captureOut → scienceManager.capture2Done` | Camera 2 image ready callback |

### 2.4 Required Inputs for Operation

The Science subtopology is not stand-alone. It requires the following connections from the including deployment topology:

* **Rate Groups:** `scienceManager` requires scheduling via a 1 Hz rate group port for periodic availability checks.
* **Top-level hardware managers:**
  * `StarTrackerManager.attitudeOut → scienceManager.attitudeIn`
  * `GnssManager.positionOut → scienceManager.positionIn`
* **DataProducts subtopology:** `scienceManager` connects to `DpManager` for container allocation and `DpWriter` for storage.
* **FileHandling subtopology:** `scienceManager` connects to `fileDownlink` for flagged image storage.
* **SatStateMachine:** `scienceManager.modeIn` receives mode commands to enable/disable science operations.

### 2.5 Limitations

The Science subtopology does not provide:

* **ADCS control** — attitude pointing is handled by the `Adcs` subtopology.
* **Ground communications** — results are recorded locally; downlink is handled by `ComFprime` and `DataProducts`.
* **Startup protocol details for Camera 1 and Camera 2** — camera startup sequences are hardware-specific and to be defined during detailed design pending vendor datasheets.

## 3. Usage

```fpp
topology HS2Flight {
  import Science.Subtopology
  import DataProducts.Subtopology
  import FileHandling.Subtopology

  connections ScienceRateGroup {
    rg2.RateGroupMemberOut[N] -> Science.Subtopology.scienceManagerRun
  }

  connections ScienceHardware {
    starTrackerManager.attitudeOut -> Science.Subtopology.attitudeIn
    gnssManager.positionOut        -> Science.Subtopology.positionIn
  }

  connections ScienceDataProducts {
    Science.Subtopology.dpGetOut      -> DataProducts.Subtopology.productGetIn
    Science.Subtopology.dpSendOut     -> DataProducts.Subtopology.productSendIn
  }

  connections ScienceFileHandling {
    Science.Subtopology.imageFileOut  -> FileHandling.Subtopology.fileDownlinkIn
  }

  connections ScienceModeControl {
    satStateMachine.scienceModeOut -> Science.Subtopology.modeIn
  }
}
```

### 3.1 Integration Notes

* `LostWorker` and `FoundWorker` are **excluded from health monitoring** per the F' health checking pattern — they are low-priority background workers.
* `ScienceManager` **is** health-monitored and must have async `pingIn` / output `pingOut` ports wired to `$health`.
* The LOST and FOUND libraries are included as CMake dependencies and called synchronously within the worker's async port handler.
* Camera startup protocols must complete before `ScienceManager` begins issuing capture requests. This is coordinated via the camera state machine reaching its `Idle` state.

## 4. Configuration

**Module:** `Science/ScienceConfig/`

### 4.1 Component properties (`ScienceConfig.fpp`)

* **Base ID** — base ID for the subtopology; each instance is offset from this.
* **Queue sizes** — queue depths for `scienceManager`, `lostWorker`, `foundWorker`, `camera1Manager`, `camera2Manager`.
* **Stack sizes** — task stack allocation for all active components.
* **Priorities** — `scienceManager` runs at high priority; `lostWorker` and `foundWorker` run at low priority; camera managers run at medium priority.

## 5. Traceability Matrix

| Requirement ID | Satisfied by |
|----------------|-------------|
| HS2-SCI-001 | `camera1Manager`, `camera2Manager` — simultaneous capture trigger from `scienceManager` |
| HS2-SCI-002 | `scienceManager.attitudeIn` ← `StarTrackerManager.attitudeOut` |
| HS2-SCI-003 | `scienceManager.positionIn` ← `GnssManager.positionOut` |
| HS2-SCI-004 | `scienceManager.startLost → lostWorker.startWork` |
| HS2-SCI-005 | `scienceManager.startFound → foundWorker.startWork` |
| HS2-SCI-006 | `scienceManager.dpSendOut` → `DataProducts` subtopology |
| HS2-SCI-007 | `scienceManager.imageFileOut` → `FileHandling` subtopology |
| HS2-SCI-008 | `scienceManager` pingIn/pingOut wired to `$health` |
| HS2-SCI-009 | `scienceManager` busy-state tracking (Manager-Worker pattern) |
| HS2-SCI-010 | `ScienceConfig.fpp` |
````

- [ ] **Step 2: Review the document against the design spec**

Open `docs/superpowers/specs/2026-04-12-satellite-sdd-design.md` section 5.2 and verify:
- All five components are present in the Instance Summary table
- Synchronized capture requirement is documented
- All internal wiring connections are listed
- Required external inputs match the design spec's "Ports consumed from outside subtopology"

- [ ] **Step 3: Commit**

```bash
git add docs/Science/sdd.md
git commit -m "docs: add Science subtopology SDD"
```

---

## Task 2: ADCS Subtopology SDD

**Files:**
- Create: `docs/Adcs/sdd.md`

- [ ] **Step 1: Create the ADCS SDD file**

Create `docs/Adcs/sdd.md` with the following content:

````markdown
# Adcs Subtopology — Software Design Document (SDD)

The **Adcs subtopology** provides attitude determination and control (ADCS) for the HS2 satellite across three operating modes. In **Detumble** mode, it uses IMU and Sun Sensor data to damp initial spin after deployment. In **Nominal** mode, it maintains a stable attitude using the same sensors. In **Precision Pointing** mode, it incorporates Star Tracker data from the top-level `StarTrackerManager` to achieve the pointing accuracy required during science experiment windows. Magnetorquers are the sole actuator in all modes. The subtopology receives mode commands from `SatStateMachine` to switch between ADCS operating modes.

## 1. Requirements

| ID | Description | Validation |
|----|-------------|------------|
| HS2-ADCS-001 | The subtopology shall damp satellite angular rates using IMU and Sun Sensor data (Detumble mode). | Inspection |
| HS2-ADCS-002 | The subtopology shall maintain stable attitude using IMU and Sun Sensor data (Nominal mode). | Inspection |
| HS2-ADCS-003 | The subtopology shall achieve precision pointing using Star Tracker attitude data (Precision Pointing mode). | Inspection |
| HS2-ADCS-004 | The subtopology shall command magnetorquers to actuate attitude corrections. | Inspection |
| HS2-ADCS-005 | The subtopology shall switch ADCS operating mode in response to commands from `SatStateMachine`. | Inspection |
| HS2-ADCS-006 | The subtopology shall run the attitude control loop at 10 Hz. | Inspection |
| HS2-ADCS-007 | The subtopology shall support configurable instance properties (IDs, queue sizes, stack sizes, priorities). | Inspection |

## 2. Design & Core Functions

### 2.1 Instance Summary

| Instance | Type | Kind | Purpose |
|----------|------|------|---------|
| `adcsManager` | `HS2.AdcsManager` | Active (high priority, 10 Hz) | Owns attitude control loop; selects sensor inputs and control algorithm based on current ADCS mode; dispatches actuation commands to `magnetorquerManager` |
| `imuManager` | `HS2.ImuManager` | Active (worker) | State machine: [Startup] → Idle → Sampling → Idle / Error. Reads IMU angular rate and acceleration via `LinuxSpiDriver` or `LinuxI2cDriver`. |
| `sunSensorManager` | `HS2.SunSensorManager` | Active (worker) | State machine: [Startup] → Idle → Sampling → Idle / Error. Reads sun sensor data via `LinuxI2cDriver` or `LinuxGpioDriver`. |
| `magnetorquerManager` | `HS2.MagnetorquerManager` | Active (worker) | State machine: [Startup] → Idle → Actuating → Idle / Error. Commands magnetorquer coil currents via `LinuxGpioDriver` or `LinuxI2cDriver`. |

### 2.2 ADCS Modes (Internal to `adcsManager`)

| Mode | Active Sensors | Actuators | Entry Trigger |
|------|---------------|-----------|---------------|
| Detumble | IMU, Sun Sensors | Magnetorquers | `SatStateMachine` Detumble command or power-on default |
| Nominal | IMU, Sun Sensors | Magnetorquers | `SatStateMachine` Nominal command |
| Precision Pointing | IMU, Sun Sensors, Star Tracker | Magnetorquers | `SatStateMachine` Precision Pointing command |

### 2.3 Internal Wiring

| Connection | Description |
|------------|-------------|
| `adcsManager.imuRequest → imuManager.sampleIn` | Request IMU sample |
| `imuManager.sampleOut → adcsManager.imuData` | IMU data callback |
| `adcsManager.sunRequest → sunSensorManager.sampleIn` | Request sun sensor sample |
| `sunSensorManager.sampleOut → adcsManager.sunData` | Sun sensor data callback |
| `adcsManager.magnetorquerCmd → magnetorquerManager.actuateIn` | Magnetorquer actuation command |
| `magnetorquerManager.actuateOut → adcsManager.actuateDone` | Actuation completion callback |

### 2.4 Required Inputs for Operation

The Adcs subtopology is not stand-alone. It requires the following connections from the including deployment topology:

* **Rate Groups:** `adcsManager` requires scheduling via a 10 Hz rate group port.
* **Top-level hardware managers:**
  * `StarTrackerManager.attitudeOut → adcsManager.starTrackerData` (Precision Pointing mode only)
  * `GnssManager.positionOut → adcsManager.gnssData` (timing and position reference)
* **SatStateMachine:** `adcsManager.modeIn` receives ADCS mode commands.
* **F' native bus drivers:** `imuManager`, `sunSensorManager`, and `magnetorquerManager` each require a wired `LinuxI2cDriver`, `LinuxSpiDriver`, or `LinuxGpioDriver` instance.

### 2.5 Limitations

The Adcs subtopology does not provide:

* **Star Tracker hardware management** — `StarTrackerManager` is a top-level component shared with the Science subtopology.
* **GNSS hardware management** — `GnssManager` is a top-level component.
* **Startup protocol details for IMU, Sun Sensors, and Magnetorquers** — device-specific initialization sequences are to be defined during detailed design pending hardware datasheets.
* **Reaction wheels** — the HS2 satellite uses magnetorquers only.

## 3. Usage

```fpp
topology HS2Flight {
  import Adcs.Subtopology

  connections AdcsRateGroup {
    rg1.RateGroupMemberOut[0] -> Adcs.Subtopology.adcsManagerRun
  }

  connections AdcsHardware {
    starTrackerManager.attitudeOut -> Adcs.Subtopology.starTrackerDataIn
    gnssManager.positionOut        -> Adcs.Subtopology.gnssDataIn
  }

  connections AdcsModeControl {
    satStateMachine.adcsModeOut -> Adcs.Subtopology.modeIn
  }

  connections AdcsDrivers {
    Adcs.Subtopology.imuDriverOut          -> linuxI2cDriver0.portIn
    Adcs.Subtopology.sunSensorDriverOut    -> linuxI2cDriver1.portIn
    Adcs.Subtopology.magnetorquerDriverOut -> linuxGpioDriver0.portIn
  }
}
```

### 3.1 Integration Notes

* `adcsManager` **is** health-monitored. Hardware managers (`imuManager`, `sunSensorManager`, `magnetorquerManager`) are excluded.
* Precision Pointing mode is only available when `StarTrackerManager` has reached its `Tracking` state.
* Hardware manager startup protocols must complete before `adcsManager` begins issuing sample/actuate requests.
* The 10 Hz control loop must complete within 100 ms. A rate group slip produces a `WARNING_HI` event.

## 4. Configuration

**Module:** `Adcs/AdcsConfig/`

### 4.1 Component properties (`AdcsConfig.fpp`)

* **Base ID** — base ID for the subtopology; each instance is offset from this.
* **Queue sizes** — queue depths for `adcsManager`, `imuManager`, `sunSensorManager`, `magnetorquerManager`.
* **Stack sizes** — task stack allocation for all active components.
* **Priorities** — `adcsManager` runs at high priority (10 Hz control loop); hardware managers run at medium priority.

## 5. Traceability Matrix

| Requirement ID | Satisfied by |
|----------------|-------------|
| HS2-ADCS-001 | `adcsManager` Detumble mode using `imuManager` + `sunSensorManager` |
| HS2-ADCS-002 | `adcsManager` Nominal mode using `imuManager` + `sunSensorManager` |
| HS2-ADCS-003 | `adcsManager` Precision Pointing mode using `starTrackerDataIn` |
| HS2-ADCS-004 | `adcsManager.magnetorquerCmd → magnetorquerManager.actuateIn` |
| HS2-ADCS-005 | `adcsManager.modeIn` ← `SatStateMachine.adcsModeOut` |
| HS2-ADCS-006 | `adcsManagerRun` port connected to 10 Hz rate group |
| HS2-ADCS-007 | `AdcsConfig.fpp` |
````

- [ ] **Step 2: Review the document against the design spec**

Open `docs/superpowers/specs/2026-04-12-satellite-sdd-design.md` section 5.3 and verify:
- All four components are in the Instance Summary table
- Three ADCS modes are documented with correct sensor combinations
- StarTrackerManager and GnssManager listed as required top-level inputs
- SatStateMachine mode command port listed as required input

- [ ] **Step 3: Commit**

```bash
git add docs/Adcs/sdd.md
git commit -m "docs: add Adcs subtopology SDD"
```

---

## Task 3: EPS Subtopology SDD

**Files:**
- Create: `docs/Eps/sdd.md`

- [ ] **Step 1: Create the EPS SDD file**

Create `docs/Eps/sdd.md` with the following content:

````markdown
# Eps Subtopology — Software Design Document (SDD)

The **Eps subtopology** monitors the HS2 satellite's electrical power system (EPS) by reading battery state of charge from the EPS board over I2C or UART. It publishes the state of charge as a telemetry channel and exposes a `powerStateOut` port consumed by `SatStateMachine` to make autonomous Standby submode decisions (Downlink and Science activation). The subtopology raises `WARNING_HI` events as battery charge drops toward a configurable low threshold, and `FATAL` events when charge reaches a critical threshold, triggering a system transition to Safe mode.

## 1. Requirements

| ID | Description | Validation |
|----|-------------|------------|
| HS2-EPS-001 | The subtopology shall read battery state of charge from the EPS board over I2C or UART. | Inspection |
| HS2-EPS-002 | The subtopology shall publish battery state of charge as a telemetry channel. | Inspection |
| HS2-EPS-003 | The subtopology shall raise a `WARNING_HI` event when state of charge drops below the low power threshold. | Inspection |
| HS2-EPS-004 | The subtopology shall raise a `FATAL` event when state of charge drops below the critical power threshold. | Inspection |
| HS2-EPS-005 | The subtopology shall expose a `powerStateOut` port delivering current state of charge to `SatStateMachine`. | Inspection |
| HS2-EPS-006 | The subtopology shall support configurable instance properties (IDs, queue sizes, stack sizes, priorities). | Inspection |

## 2. Design & Core Functions

### 2.1 Instance Summary

| Instance | Type | Kind | Purpose |
|----------|------|------|---------|
| `epsManager` | `HS2.EpsManager` | Active (worker) | State machine: [Startup] → Idle → Reading → LowPower / Error. Reads EPS board SoC via driver; publishes telemetry; raises threshold events; exposes `powerStateOut`. |

### 2.2 EPS Manager State Machine

| State | Description |
|-------|-------------|
| [Startup] | Device-specific EPS board initialization sequence (to be defined in detailed design) |
| Idle | Awaiting next scheduled read |
| Reading | Requesting SoC value from EPS board driver |
| LowPower | SoC below `POWER_THRESHOLD`; `WARNING_HI` event emitted; `powerStateOut` signals insufficient power |
| Error | Driver communication failure; `WARNING_HI` event emitted |

### 2.3 Internal Wiring

`EpsManager` is the sole component in this subtopology. It communicates directly with the `LinuxI2cDriver` or `LinuxUartDriver` via the `ByteStreamDriverModel` interface.

### 2.4 Required Inputs for Operation

* **Rate Groups:** `epsManager` requires scheduling via a 1 Hz rate group port.
* **F' native bus driver:** `epsManager` requires a wired `LinuxI2cDriver` or `LinuxUartDriver` instance connected to the EPS board.
* **SatStateMachine:** `epsManager.powerStateOut → satStateMachine.powerStateIn` must be connected in the deployment topology.

### 2.5 Limitations

The Eps subtopology does not provide:

* **Solar panel pointing** — solar panel orientation is handled by the `Adcs` subtopology (Charge submode).
* **Power switch control** — enabling/disabling power rails is not in scope for this subtopology.
* **EPS board startup protocol details** — to be defined during detailed design pending EPS board datasheet.

## 3. Usage

```fpp
topology HS2Flight {
  import Eps.Subtopology

  connections EpsRateGroup {
    rg2.RateGroupMemberOut[N] -> Eps.Subtopology.epsManagerRun
  }

  connections EpsDriver {
    Eps.Subtopology.epsDriverOut -> linuxI2cDriver2.portIn
  }

  connections EpsModeControl {
    Eps.Subtopology.powerStateOut -> satStateMachine.powerStateIn
  }
}
```

### 3.1 Integration Notes

* `epsManager` **is** health-monitored and must have `pingIn`/`pingOut` ports wired to `$health`.
* The `POWER_THRESHOLD` and `CRITICAL_POWER_THRESHOLD` parameters are stored in `PrmDb` and loaded at startup.
* A `FATAL` event from `epsManager` triggers `SatStateMachine` to transition any mode → Safe.

## 4. Configuration

**Module:** `Eps/EpsConfig/`

### 4.1 Component properties (`EpsConfig.fpp`)

* **Base ID** — base ID for the subtopology.
* **Queue size** — queue depth for `epsManager`.
* **Stack size** — task stack allocation for `epsManager`.
* **Priority** — RTOS scheduling priority for `epsManager`.

### 4.2 Power threshold parameters (stored in `PrmDb`)

| Parameter | Description |
|-----------|-------------|
| `POWER_THRESHOLD` | Minimum SoC (%) for Downlink/Science submode activation; triggers `WARNING_HI` below this value |
| `CRITICAL_POWER_THRESHOLD` | SoC (%) at which `FATAL` event is raised and Safe mode is entered |

## 5. Traceability Matrix

| Requirement ID | Satisfied by |
|----------------|-------------|
| HS2-EPS-001 | `epsManager` reads via `LinuxI2cDriver` / `LinuxUartDriver` |
| HS2-EPS-002 | `epsManager` telemetry channel: `BatteryStateOfCharge` (U8, percentage) |
| HS2-EPS-003 | `epsManager` `WARNING_HI` event at `POWER_THRESHOLD` |
| HS2-EPS-004 | `epsManager` `FATAL` event at `CRITICAL_POWER_THRESHOLD` |
| HS2-EPS-005 | `Eps.Subtopology.powerStateOut → satStateMachine.powerStateIn` |
| HS2-EPS-006 | `EpsConfig.fpp` |
````

- [ ] **Step 2: Review the document against the design spec**

Open `docs/superpowers/specs/2026-04-12-satellite-sdd-design.md` sections 5.4 and 8 and verify:
- `powerStateOut` port is documented and matches wiring table in section 9
- `POWER_THRESHOLD` parameter matches `SatStateMachine` section
- `FATAL` event triggering Safe mode transition is consistent with mode table in section 3

- [ ] **Step 3: Commit**

```bash
git add docs/Eps/sdd.md
git commit -m "docs: add Eps subtopology SDD"
```

---

## Task 4: Thermal Subtopology SDD

**Files:**
- Create: `docs/Thermal/sdd.md`

- [ ] **Step 1: Create the Thermal SDD file**

Create `docs/Thermal/sdd.md` with the following content:

````markdown
# Thermal Subtopology — Software Design Document (SDD)

The **Thermal subtopology** monitors the HS2 satellite's temperature sensors and commands the heater to maintain internal operating temperatures within safe bounds. `ThermalManager` polls temperature sensors at 1 Hz via `TemperatureSensorManager` workers, compares readings against configurable F' parameter thresholds, and issues heater on/off commands to `HeaterManager`. Out-of-range temperature readings generate `WARNING_HI` or `FATAL` events visible in ground telemetry. Thermal thresholds are stored as F' parameters in `PrmDb`, allowing ground operators to adjust limits without a software update.

## 1. Requirements

| ID | Description | Validation |
|----|-------------|------------|
| HS2-THM-001 | The subtopology shall poll all temperature sensors at 1 Hz. | Inspection |
| HS2-THM-002 | The subtopology shall command the heater on when any temperature reading falls below the lower threshold. | Inspection |
| HS2-THM-003 | The subtopology shall command the heater off when all temperature readings are above the lower threshold. | Inspection |
| HS2-THM-004 | The subtopology shall raise a `WARNING_HI` event when any temperature reading exceeds the upper warning threshold. | Inspection |
| HS2-THM-005 | The subtopology shall raise a `FATAL` event when any temperature reading exceeds the upper critical threshold. | Inspection |
| HS2-THM-006 | Temperature thresholds shall be stored as configurable F' parameters persisted by `PrmDb`. | Inspection |
| HS2-THM-007 | The subtopology shall support configurable instance properties (IDs, queue sizes, stack sizes, priorities). | Inspection |

## 2. Design & Core Functions

### 2.1 Instance Summary

| Instance | Type | Kind | Purpose |
|----------|------|------|---------|
| `thermalManager` | `HS2.ThermalManager` | Active | Polls `temperatureSensorManager` at 1 Hz; evaluates readings against thresholds; issues heater commands to `heaterManager`; emits threshold events |
| `temperatureSensorManager` | `HS2.TemperatureSensorManager` | Active (worker) | State machine: [Startup] → Idle → Sampling → Idle / Error. Reads temperature sensor values over `LinuxI2cDriver`. |
| `heaterManager` | `HS2.HeaterManager` | Active (worker) | State machine: [Startup] → Off → HeatingUp → On → CoolingDown / Error. Commands heater on/off via `LinuxGpioDriver`. |

### 2.2 Thermal Control Logic (Internal to `thermalManager`)

| Condition | Action | Event |
|-----------|--------|-------|
| Any sensor < `TEMP_LOW_THRESHOLD` | Command heater ON | `HLTH_TEMP_LOW` (Activity HI) |
| All sensors ≥ `TEMP_LOW_THRESHOLD` | Command heater OFF | `HLTH_TEMP_NOMINAL` (Activity HI) |
| Any sensor > `TEMP_WARN_THRESHOLD` | No actuation | `HLTH_TEMP_HIGH_WARN` (Warning HI) |
| Any sensor > `TEMP_CRIT_THRESHOLD` | No actuation | `HLTH_TEMP_HIGH_CRIT` (FATAL) |

### 2.3 Internal Wiring

| Connection | Description |
|------------|-------------|
| `thermalManager.sampleRequest → temperatureSensorManager.sampleIn` | Request temperature sample |
| `temperatureSensorManager.sampleOut → thermalManager.tempData` | Temperature reading callback |
| `thermalManager.heaterCmd → heaterManager.cmdIn` | Heater on/off command |
| `heaterManager.cmdOut → thermalManager.heaterDone` | Heater command completion callback |

### 2.4 Required Inputs for Operation

* **Rate Groups:** `thermalManager` requires scheduling via a 1 Hz rate group port.
* **F' native bus drivers:**
  * `temperatureSensorManager` requires a wired `LinuxI2cDriver` instance.
  * `heaterManager` requires a wired `LinuxGpioDriver` instance.

### 2.5 Limitations

The Thermal subtopology does not provide:

* **Multi-zone independent thermal control** — all sensors share the same threshold parameters in this version.
* **Startup protocol details for temperature sensors and heater** — to be defined during detailed design pending hardware datasheets.

## 3. Usage

```fpp
topology HS2Flight {
  import Thermal.Subtopology

  connections ThermalRateGroup {
    rg2.RateGroupMemberOut[N] -> Thermal.Subtopology.thermalManagerRun
  }

  connections ThermalDrivers {
    Thermal.Subtopology.tempSensorDriverOut -> linuxI2cDriver3.portIn
    Thermal.Subtopology.heaterDriverOut     -> linuxGpioDriver1.portIn
  }
}
```

### 3.1 Integration Notes

* `thermalManager` **is** health-monitored. `temperatureSensorManager` and `heaterManager` are excluded.
* Thermal threshold parameters (`TEMP_LOW_THRESHOLD`, `TEMP_WARN_THRESHOLD`, `TEMP_CRIT_THRESHOLD`) are loaded from `PrmDb` at startup and can be updated by ground command.
* A `FATAL` event from `thermalManager` propagates through `EventManager` to `fatalHandler`, triggering a system reset.

## 4. Configuration

**Module:** `Thermal/ThermalConfig/`

### 4.1 Component properties (`ThermalConfig.fpp`)

* **Base ID** — base ID for the subtopology; each instance is offset from this.
* **Queue sizes** — queue depths for `thermalManager`, `temperatureSensorManager`, `heaterManager`.
* **Stack sizes** — task stack allocation for all active components.
* **Priority** — RTOS scheduling priority for all active components.

### 4.2 Thermal threshold parameters (stored in `PrmDb`)

| Parameter | Description |
|-----------|-------------|
| `TEMP_LOW_THRESHOLD` | Temperature (°C) below which heater turns ON |
| `TEMP_WARN_THRESHOLD` | Temperature (°C) above which `WARNING_HI` event is raised |
| `TEMP_CRIT_THRESHOLD` | Temperature (°C) above which `FATAL` event is raised |

## 5. Traceability Matrix

| Requirement ID | Satisfied by |
|----------------|-------------|
| HS2-THM-001 | `thermalManagerRun` port connected to 1 Hz rate group |
| HS2-THM-002 | `thermalManager.heaterCmd → heaterManager.cmdIn` on low threshold |
| HS2-THM-003 | `thermalManager` heater OFF command when all sensors nominal |
| HS2-THM-004 | `thermalManager` `WARNING_HI` event at `TEMP_WARN_THRESHOLD` |
| HS2-THM-005 | `thermalManager` `FATAL` event at `TEMP_CRIT_THRESHOLD` |
| HS2-THM-006 | `TEMP_LOW_THRESHOLD`, `TEMP_WARN_THRESHOLD`, `TEMP_CRIT_THRESHOLD` in `PrmDb` |
| HS2-THM-007 | `ThermalConfig.fpp` |
````

- [ ] **Step 2: Review the document against the design spec**

Open `docs/superpowers/specs/2026-04-12-satellite-sdd-design.md` section 5.5 and verify:
- All three components are in the Instance Summary table with correct state machines
- Threshold parameters match what is referenced in the design spec
- `FATAL` event propagation to Safe mode is consistent with section 3 mode table

- [ ] **Step 3: Commit**

```bash
git add docs/Thermal/sdd.md
git commit -m "docs: add Thermal subtopology SDD"
```
