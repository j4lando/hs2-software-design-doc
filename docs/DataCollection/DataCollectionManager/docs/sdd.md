# DataCollectionManager SDD

## 1. Overview

`DataCollectionManager` is an Active component responsible for executing data collection experiments on the HS2 satellite. It powers on cameras, configures them with experiment parameters loaded from F' PrmDb, simultaneously acquires images and navigation data, stores results to flash, and powers off all hardware upon completion. It reports success or failure to the SatStateMachine.

DataCollectionManager is the manager in the DataCollection subtopology, coordinating CameraManager workers (Camera1, Camera2) as well as external attitude and position data sources (StarTrackerManager, GnssManager).

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|-------------|
| HS2-DCM-001 | DataCollectionManager shall capture data as defined by experiment parameters | Inspection |
| HS2-DCM-002 | DataCollectionManager shall power on all cameras prior to an experiment | Inspection |
| HS2-DCM-003 | DataCollectionManager shall perform a health check on all cameras before running an experiment and assert WARNING_HI if any check fails | Inspection |
| HS2-DCM-004 | DataCollectionManager shall simultaneously acquire images, attitude, and position within 10ms | Inspection |
| HS2-DCM-005 | DataCollectionManager shall store all captured images to flash with experiment ID and timestamp | Inspection |
| HS2-DCM-006 | DataCollectionManager shall not perform an experiment if flash data storage is full | Inspection |
| HS2-DCM-007 | DataCollectionManager shall power off all cameras upon experiment completion | Inspection |
| HS2-DCM-008 | DataCollectionManager shall report experiment success or failure upon completion | Inspection |
| HS2-DCM-009 | DataCollectionManager shall respond to health ping within the required deadline | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal F' state machine including a `[Startup]` phase.

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc::Sched` | Rate group tick |
| `runExperiment` | Input | `Fw::Cmd` | Trigger experiment from SatStateMachine or ground |
| `cameraPowerOn[2]` | Output | `Fw::Cmd` | Power on Camera1/Camera2 Manager |
| `cameraPowerOff[2]` | Output | `Fw::Cmd` | Power off Camera1/Camera2 Manager |
| `cameraCmd[2]` | Output | `Fw::Cmd` | Configure and capture commands to CameraManagers |
| `attitudeGet` | Output | `Fw::Dp` | Request attitude snapshot from StarTrackerManager |
| `positionGet` | Output | `Fw::Dp` | Request position snapshot from GnssManager |
| `storageQuery` | Output | `Fw::Cmd` | Query flash storage availability |
| `imageWrite` | Output | `Fw::Dp` | Write captured image and metadata to flash |
| `experimentResult` | Output | `Fw::Cmd` | Report success or failure to SatStateMachine |
| `pingIn` / `pingOut` | In/Out | `Svc::Ping` | Health monitoring via Svc::Health |
| `prmGet` | Output | `Fw::PrmGet` | Load experiment parameters from PrmDb |
| `logOut` | Output | `Fw::Log` | Event logging |
| `tlmOut` | Output | `Fw::Tlm` | Telemetry (experiment state, storage status) |

### 3.3 Commands

| Mnemonic | Args | Description |
|----------|------|-------------|
| `RUN_EXPERIMENT` | `expId: U8` | Trigger a data collection run |

### 3.4 Execution Flow

1. On `RUN_EXPERIMENT`: load experiment parameters from PrmDb, query storage — abort and report failure if full
2. Power on both cameras via `cameraPowerOn`
3. Camera health check is performed internally by each CameraManager during startup
4. Configure cameras with experiment parameters via `cameraCmd`
5. Simultaneously fire capture + request attitude + request position within 10ms window
6. Write images and metadata to flash via `imageWrite`
7. Power off all cameras via `cameraPowerOff`
8. Report success or failure via `experimentResult`

---

## 4. State Machine

DataCollectionManager uses an F' state machine (`Fw::Sm`) with the following phases:

- `[Startup]` — hardware initialization and self-check
- `IDLE` — waiting for experiment command
- `POWERING_ON` — cameras powering up
- `CAPTURING` — simultaneous image and navigation data acquisition
- `STORING` — writing results to flash
- `POWERING_OFF` — cameras powering down
- `REPORTING` — sending result to SatStateMachine

---

## 5. Notes

- Experiment types (L&F and Calibration) are encoded in the experiment parameters loaded from PrmDb; DataCollectionManager does not need to distinguish them explicitly beyond passing parameters to CameraManagers.
- StarTrackerManager and GnssManager are top-level components shared across subtopologies; connections are wired at the top-level topology.
- Storage full check is performed before any hardware is powered on to avoid unnecessary power draw.
