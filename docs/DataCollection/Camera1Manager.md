# Camera1Manager SDD

## 1. Overview

`Camera1Manager` is the Layer 2 hardware manager for Camera 1 within the DataCollection subtopology. It owns all bus communication with Camera 1 — resetting, enabling, configuring, and capturing frames — and pushes raw image data to `DataCollectionApplication` via a typed output port on each successful capture.

Camera 1 connects via SPI (image frame transfer) and I2C (register configuration). `Camera1Manager` is an **Active** component with its own execution thread, needed because SPI frame readout can take tens of milliseconds and must not block the rate-group caller. It has no satellite mode awareness. It is powered on and off by `DataCollectionApplication` via direct command ports.

Camera 1 is the sensor for the LOST (lost-in-space star identification) and SCOPE (star catalog calibration) algorithms, both invoked downstream by `ScienceInferenceApplication`.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|--------------|
| HS2-CM1-001 | Camera1Manager shall reset Camera 1 to a known default state before any configuration is applied | Inspection |
| HS2-CM1-002 | Camera1Manager shall wait for Camera 1 reset stabilization before proceeding to enable | Inspection |
| HS2-CM1-003 | Camera1Manager shall enable Camera 1 and proceed to configuration on receipt of a `POWER_ON` command | Inspection |
| HS2-CM1-004 | Camera1Manager shall configure Camera 1 exposure, gain, and frame parameters from F' parameters before entering the ready state | Inspection |
| HS2-CM1-005 | Camera1Manager shall capture a single image frame on receipt of a `CAPTURE` command while in RUN | Inspection |
| HS2-CM1-006 | Camera1Manager shall push captured image data and a hardware timestamp to DataCollectionApplication via `imageOut` upon successful capture | Inspection |
| HS2-CM1-007 | Camera1Manager shall reject `CAPTURE` commands received outside RUN and respond with a failed command status | Inspection |
| HS2-CM1-008 | Camera1Manager shall transition to OFF and cease all bus operations on receipt of a `POWER_OFF` command from any state | Inspection |
| HS2-CM1-009 | Camera1Manager shall log a `WARNING_HI` event and transition to RESET on any bus error during startup or capture | Inspection |
| HS2-CM1-010 | Camera1Manager shall track consecutive failure count and report it as a telemetry channel | Inspection |
| HS2-CM1-011 | Camera1Manager shall apply updated F' parameter values by re-entering CONFIGURE without a full reset | Inspection |
| HS2-CM1-012 | Camera1Manager shall perform no bus operations while in OFF or WAIT_RESET | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal flat F' state machine (`Fw::Sm`). Has its own execution thread — required because SPI frame transfer during capture blocks for tens of milliseconds and must not stall the rate-group caller thread.

### 3.2 Commands

Camera1Manager accepts commands via `cmdIn` (see §3.4). `DataCollectionApplication` sends commands programmatically via its `cameraPowerOn[0]`, `cameraPowerOff[0]`, and `cameraCmd[0]` output ports, all of which connect to this single `cmdIn` in topology. The same commands are also accessible to ground through `CmdDispatcher` during testing and checkout.

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

> **Bus mapping:** `busWrite` (named for port symmetry with `busWriteRead`) uses `Drv.SpiWriteRead` — the SPI bus is used for full-duplex frame readout, not register access. `busWriteRead` uses `Drv.I2cWriteRead` for register write-then-readback. This follows the reference pattern in `fprime-community/fprime-sensors` `ImuManager` (`busWrite: Drv.I2c`, `busWriteRead: Drv.I2cWriteRead`).

---

## 4. State Machine

`Camera1Manager` uses a single flat F' state machine extended from the standard hardware manager pattern to include an `OFF` state. The camera is explicitly powered on and off by `DataCollectionApplication` to manage power budget around experiments.

The SM is driven by two inputs: `schedIn` (for startup state progression) and async commands via `cmdIn`.

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
    if all writes OK  → RUN
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
    → CONFIGURE    # re-configure for new experiment type; re-enters CONFIGURE state

# Global transitions — reachable from any state:
on POWER_OFF command  → OFF         # commanded shutdown; any in-progress capture fails
on error signal       → RESET       # self-healing fallback
on reconfigure signal → CONFIGURE   # parameter update path; only effective from RUN
```

**POWER_ON / POWER_OFF:** `POWER_ON` received in any state other than `OFF` is ignored — the camera is already initializing or ready (idempotent). `POWER_OFF` received from any state transitions immediately to `OFF` and returns `cmdResponse SUCCESS`. Any in-progress capture returns `cmdResponse FAILED` before the shutdown.

**CAPTURE only in RUN:** If a `CAPTURE` command arrives in any state other than `RUN`, `Camera1Manager` returns `cmdResponse FAILED` immediately without a state transition. `DataCollectionApplication` is responsible for treating this as an experiment failure (see HS2-DCA-003).

**Error self-healing:** Bus errors in `ENABLE`, `CONFIGURE`, or `RUN` emit `WARNING_HI`, increment `consecutiveFailures`, and re-enter `RESET`. The SM retries the full startup sequence automatically on the next tick. `Camera1Manager` does not manage its own power rail — if `consecutiveFailures` exceeds a threshold observable via telemetry, `DataCollectionApplication` can command `POWER_OFF` and treat the camera as failed.

**Parameter reconfiguration:** When `parameterUpdated()` is called (F' framework callback), `Camera1Manager` sends a `reconfigure` signal. From `RUN` this goes directly to `CONFIGURE`. From any other state it is ignored; updated values apply on next natural `CONFIGURE` entry.

Reference: [`fprime-community/fprime-sensors/ImuManager`](https://github.com/fprime-community/fprime-sensors/tree/devel/fprime-sensors/MpuImu/Components/ImuManager), [FPP flat state machines](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc)

---

## 5. Notes

- `Camera1Manager` is instantiated inside the DataCollection subtopology. Its `busWrite` (SPI) and `busWriteRead` (I2C) ports connect to `LinuxSpiDriver` and `LinuxI2cDriver` instances at the top-level topology; bus assignment is fixed at integration time.
- **Camera power rail:** Physical power-on/off of the Camera 1 rail is managed externally — likely via `EPSApplication` or a dedicated GPIO line — before `DataCollectionApplication` issues the `POWER_ON` command. `Camera1Manager`'s `POWER_ON` triggers the software initialization sequence, not a hardware power switch. A GPIO power-enable port should be added here if the flight computer directly drives a camera enable line; this is deferred pending hardware confirmation.
- `Cam.ImagePort` carries: raw frame buffer, `seqId`, hardware-latched timestamp, and capture status. `DataCollectionApplication` matches Camera1 and Camera2 frames by `seqId` when assembling the synchronized capture result.
- Camera 1 images feed the LOST (L&F experiments) and SCOPE (calibration experiments) algorithms in `ScienceInferenceApplication`. `Camera1Manager` is experiment-agnostic — experiment type is encoded in `expId` and stored with the frame by `DataCollectionApplication`.
- `Camera1Manager` is **excluded from health monitoring** (`Svc::Health`). Only `DataCollectionApplication` is health-checked in this subtopology.
- `Camera1Manager` has a single `cmdIn: Fw.Cmd` port following the F' standard (proven from `fprime-community/fprime-sensors` `ImuManager.fpp`: `command recv port CmdDisp`). At topology level, all three of `DataCollectionApplication`'s command output ports for camera 1 (`cameraPowerOn[0]`, `cameraPowerOff[0]`, `cameraCmd[0]`) connect to this single `cmdIn`. Multiple sources connecting to one command-receive port is standard F' — it is how `CmdDispatcher` itself works. `Camera2Manager`'s `cmdIn` receives from index 1 of each array.
- If Camera 1 uses a single SPI bus for both register access and frame transfer, the `busWriteRead` (I2C) port is replaced by an additional `busWrite` (SPI) usage and the I2C driver connection is omitted. Bus type is resolved at topology assembly.
- Specific register addresses, power sequencing timing, and SPI frame protocol are implementation details for the component C++ source, not specified here.
- Deferred: exact `consecutiveFailures` threshold prompting `DataCollectionApplication` to command `POWER_OFF` is a system-level parameter defined at detailed design.
