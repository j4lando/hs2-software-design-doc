# Camera2Manager SDD

## 1. Overview

`Camera2Manager` is the Layer 2 hardware manager for Camera 2 within the DataCollection subtopology. It owns all bus communication with Camera 2 — resetting, enabling, configuring, and capturing frames — and pushes raw image data to `DataCollectionApplication` via a typed output port on each successful capture.

Camera 2 connects via SPI (image frame transfer) and I2C (register configuration). `Camera2Manager` is an **Active** component with its own execution thread, needed because SPI frame readout can take tens of milliseconds and must not block the rate-group caller. It has no satellite mode awareness. It is powered on and off by `DataCollectionApplication` via direct command ports.

Camera 2 is the sensor for the FOUND (follow-up optical navigation) and SCOPE (star catalog calibration) algorithms, both invoked downstream by `ScienceInferenceApplication`. `Camera2Manager` is symmetric to `Camera1Manager` — the two components follow the same design pattern and differ only in which algorithms consume their output.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-CM2-001 | Camera2Manager shall reset Camera 2 to a known default state before any configuration is applied | Inspection |
| HS2-CM2-002 | Camera2Manager shall wait for Camera 2 reset stabilization before proceeding to enable | Inspection |
| HS2-CM2-003 | Camera2Manager shall enable Camera 2 and proceed to configuration on receipt of a `POWER_ON` command | Inspection |
| HS2-CM2-004 | Camera2Manager shall configure Camera 2 exposure, gain, and frame parameters from F' parameters before entering the ready state | Inspection |
| HS2-CM2-005 | Camera2Manager shall capture a single image frame on receipt of a `CAPTURE` command while in RUN | Inspection |
| HS2-CM2-006 | Camera2Manager shall push captured image data and a hardware timestamp to DataCollectionApplication via `imageOut` upon successful capture | Inspection |
| HS2-CM2-007 | Camera2Manager shall reject `CAPTURE` commands received outside RUN and respond with a failed command status | Inspection |
| HS2-CM2-008 | Camera2Manager shall transition to OFF and cease all bus operations on receipt of a `POWER_OFF` command from any state | Inspection |
| HS2-CM2-009 | Camera2Manager shall log a `WARNING_HI` event and transition to RESET on any bus error during startup or capture | Inspection |
| HS2-CM2-010 | Camera2Manager shall track consecutive failure count and report it as a telemetry channel | Inspection |
| HS2-CM2-011 | Camera2Manager shall apply updated F' parameter values by re-entering CONFIGURE without a full reset | Inspection |
| HS2-CM2-012 | Camera2Manager shall perform no bus operations while in OFF or WAIT_RESET | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal flat F' state machine (`Fw::Sm`). Has its own execution thread — required because SPI frame transfer during capture blocks for tens of milliseconds and must not stall the rate-group caller thread.

### 3.2 Commands

`Camera2Manager` accepts commands via `cmdIn` (see §3.4). `DataCollectionApplication` sends commands programmatically via its `cameraPowerOn[1]`, `cameraPowerOff[1]`, and `cameraCmd[1]` output ports, all of which connect to this single `cmdIn` in topology. The same commands are also accessible to ground through `CmdDispatcher` during testing and checkout.

| Mnemonic | Args | Description |
|----------|------|-------------|
| `POWER_ON` | — | Start initialization sequence from OFF |
| `POWER_OFF` | — | Shutdown from any state; enter OFF |
| `CONFIGURE` | `expId: U8` | Re-configure camera registers for the given experiment |
| `CAPTURE` | `expId: U8, seqId: U32` | Trigger a single frame capture (valid only in RUN) |

`POWER_ON` is idempotent if already out of OFF (ignored). `POWER_OFF` is valid from any state and interrupts any in-progress operation.

### 3.3 Parameters

Configuration is entirely parameter-driven. Parameters are loaded from `Svc::PrmDb` in the `CONFIGURE` state. A parameter update at runtime sends a `reconfigure` signal → direct transition `RUN → CONFIGURE` without a full reset.

| Parameter | Type | Description |
|-----------|------|-------------|
| `EXPOSURE_US` | `U32` | Exposure time in microseconds |
| `ANALOG_GAIN` | `U8` | Analog gain register value (device-specific) |
| `FRAME_WIDTH` | `U16` | Active pixel columns |
| `FRAME_HEIGHT` | `U16` | Active pixel rows |
| `RESET_WAIT_TICKS` | `U32` | Rate-group ticks to wait after issuing reset before proceeding |

> **Parameter independence:** Camera1Manager and Camera2Manager each hold their own independent parameter set. Exposure and gain values may differ between the two cameras, since FOUND (Camera 2) may require different optical settings than LOST (Camera 1). Ground can update each independently via `PrmDb`.

### 3.4 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | 1 Hz rate-group tick — drives SM startup progression |
| `cmdIn` | Input | `Fw.Cmd` | All commands — receives from DataCollectionApplication and CmdDispatcher |
| `imageOut` | Output | `Cam.ImagePort` | Push captured frame + hardware timestamp to DataCollectionApplication |
| `busWrite` | Output | `Drv.SpiWriteRead` | SPI frame data transfer (capture readout) |
| `busWriteRead` | Output | `Drv.I2cWriteRead` | I2C register configuration and readback |
| `prmGet` | Output | `Fw.PrmGet` | Load parameters from PrmDb during CONFIGURE |
| `logOut` | Output | `Fw.Log` | Event logging (state transitions, errors) |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (SM state, consecutive failure count, last capture timestamp, last seqId) |

> **Bus mapping:** `busWrite` (named for port symmetry with `busWriteRead`) uses `Drv.SpiWriteRead` for SPI frame readout. `busWriteRead` uses `Drv.I2cWriteRead` for I2C register write-then-readback. Both cameras are wired to separate SPI and I2C bus instances; they do not share a bus with Camera1Manager.

---

## 4. State Machine

`Camera2Manager` uses an identical flat F' state machine to `Camera1Manager`. The SM is extended from the standard hardware manager pattern to include an `OFF` state. Driven by `schedIn` (startup progression) and async commands via `cmdIn`.

```
OFF
  entry: (power rail management is external — see §5 Notes)
         log state transition event
  on tick: no action
  on POWER_ON command → RESET

RESET
  entry: write software reset register via busWriteRead (I2C)
         reset consecutive-failure counter
         reset wait-tick counter
  on tick → WAIT_RESET

WAIT_RESET
  on tick: increment wait-tick counter
           if counter >= RESET_WAIT_TICKS → ENABLE
           (no bus operations in this state)

ENABLE
  entry: write power management register to wake sensor (clear sleep bit via busWriteRead)
  on tick:
    if busWriteRead OK → CONFIGURE
    if error          → log WARNING_HI, increment failure count → RESET

CONFIGURE
  entry: load EXPOSURE_US, ANALOG_GAIN, FRAME_WIDTH, FRAME_HEIGHT from PrmDb
         write exposure register via busWriteRead (I2C)
         write gain register via busWriteRead (I2C)
         write frame size registers via busWriteRead (I2C)
  on tick:
    if all writes OK   → RUN
    if any write error → log WARNING_HI, increment failure count → RESET

RUN
  on CAPTURE command(expId, seqId):
    initiate frame transfer via busWrite (SPI)
    if busWrite OK:
      push imageOut(frameBuffer, seqId, timestamp)
      reset consecutive-failure counter
      return cmdResponse SUCCESS
    if busWrite error:
      log WARNING_HI (throttled), increment failure count
      return cmdResponse FAILED → RESET
  on CONFIGURE command(expId):
    → CONFIGURE    # re-configure for new experiment type

# Global transitions — reachable from any state:
on POWER_OFF command  → OFF         # commanded shutdown; any in-progress capture fails
on error signal       → RESET       # self-healing fallback
on reconfigure signal → CONFIGURE   # parameter update path; only effective from RUN
```

**POWER_ON / POWER_OFF:** `POWER_ON` received outside `OFF` is ignored (idempotent). `POWER_OFF` received from any state transitions immediately to `OFF` and returns `cmdResponse SUCCESS`. Any in-progress capture returns `cmdResponse FAILED` before shutdown.

**CAPTURE only in RUN:** `CAPTURE` received in any state other than `RUN` returns `cmdResponse FAILED` immediately without a state change. `DataCollectionApplication` treats this as an experiment failure (HS2-DCA-003).

**Error self-healing:** Bus errors in `ENABLE`, `CONFIGURE`, or `RUN` emit `WARNING_HI`, increment `consecutiveFailures`, and re-enter `RESET`. The SM retries the full startup sequence automatically on the next tick. If `consecutiveFailures` exceeds a threshold observable via telemetry, `DataCollectionApplication` can command `POWER_OFF`.

**Synchronized capture timing:** `DataCollectionApplication` issues `CAPTURE` commands to both `Camera1Manager` and `Camera2Manager` in close succession during the `CAPTURING` substate. Both cameras execute independently on their own threads. `DataCollectionApplication` collects both `imageOut` callbacks (keyed by `seqId`) along with StarTracker attitude and GNSS position within the 10 ms synchronized capture window (HS2-DCA-004).

**Parameter reconfiguration:** When `parameterUpdated()` is called, `Camera2Manager` sends a `reconfigure` signal. From `RUN` this goes directly to `CONFIGURE`. From any other state it is ignored.

Reference: [`fprime-community/fprime-sensors/ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager), [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc)

---

## 5. Notes

- `Camera2Manager` is instantiated inside the DataCollection subtopology. Its `busWrite` (SPI) and `busWriteRead` (I2C) ports connect to separate `LinuxSpiDriver` and `LinuxI2cDriver` instances from Camera1Manager at the top-level topology; bus assignment is fixed at integration time.
- **Camera power rail:** Physical power-on/off of the Camera 2 rail is managed externally — likely via `EPSApplication` or a dedicated GPIO line — before `DataCollectionApplication` issues the `POWER_ON` command. `Camera2Manager`'s `POWER_ON` triggers the software initialization sequence, not a hardware power switch. A GPIO power-enable port should be added here if the flight computer directly drives a camera enable line; deferred pending hardware confirmation.
- `Cam.ImagePort` carries: raw frame buffer, `seqId`, hardware-latched timestamp, and capture status. `DataCollectionApplication` matches Camera1 and Camera2 frames by `seqId` when assembling the synchronized capture result for storage and dispatch to `ScienceInferenceApplication`.
- Camera 2 images feed the FOUND (L&F experiments) and SCOPE (calibration experiments) algorithms in `ScienceInferenceApplication`. `Camera2Manager` is experiment-agnostic — experiment type is encoded in `expId` and stored with the frame by `DataCollectionApplication`.
- `Camera2Manager` is **excluded from health monitoring** (`Svc::Health`). Only `DataCollectionApplication` is health-checked in this subtopology.
- `Camera2Manager` has a single `cmdIn: Fw.Cmd` port following the F' standard (proven from `fprime-community/fprime-sensors` `ImuManager.fpp`: `command recv port CmdDisp`). At topology level, all three of `DataCollectionApplication`'s command output ports for camera 2 (`cameraPowerOn[1]`, `cameraPowerOff[1]`, `cameraCmd[1]`) connect to this single `cmdIn`. `Camera1Manager`'s `cmdIn` receives from index 0 of each array.
- If Camera 2 uses a single SPI bus for both register access and frame transfer, the `busWriteRead` (I2C) port is replaced by additional `busWrite` (SPI) usage. Bus type is resolved at topology assembly.
- `Camera2Manager` and `Camera1Manager` are symmetric designs. Any change to one's hardware manager pattern (e.g., adding a GPIO power-enable port) should be applied to both.
- Specific register addresses, power sequencing timing, and SPI frame protocol are implementation details for the component C++ source, not specified here.
- Deferred: exact `consecutiveFailures` threshold prompting `DataCollectionApplication` to command `POWER_OFF` is a system-level parameter defined at detailed design.
