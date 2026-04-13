# ScienceInferenceManager SDD

## 1. Overview

`ScienceInferenceManager` is an Active component responsible for processing raw images stored on flash by running them through the HS2 science algorithms (LOST, FOUND, SCOPE). It operates on a scheduled polling cycle, finds unprocessed images, invokes the appropriate algorithm based on experiment metadata, stores results for downlink, compresses flagged images for downlink, and marks each image as processed.

ScienceInferenceManager is the manager in the ScienceInference subtopology. It is fully schedule-driven and receives no ground commands.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|-------------|
| HS2-SIM-001 | ScienceInferenceManager shall poll flash storage for unprocessed images on a scheduled interval | Inspection |
| HS2-SIM-002 | ScienceInferenceManager shall process each unprocessed image with its designated algorithm (LOST, FOUND, or SCOPE) based on experiment type | Inspection |
| HS2-SIM-003 | ScienceInferenceManager shall store algorithm outputs for downlink with matching experiment ID and timestamp | Inspection |
| HS2-SIM-004 | ScienceInferenceManager shall compress flagged images for downlink | Inspection |
| HS2-SIM-005 | ScienceInferenceManager shall mark each image as processed upon successful algorithm completion | Inspection |
| HS2-SIM-006 | ScienceInferenceManager shall assert WARNING_HI if algorithm processing fails for an image | Inspection |
| HS2-SIM-007 | ScienceInferenceManager shall respond to health ping within the required deadline | Inspection |

---

## 3. Design

### 3.1 Component Type

Active component with internal F' state machine including a `[Startup]` phase.

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc::Sched` | Rate group tick to poll for unprocessed images |
| `storageQuery` | Output | `Fw::Cmd` | Query flash for unprocessed images |
| `imageRead` | Output | `Fw::Dp` | Read image data from flash |
| `resultWrite` | Output | `Fw::Dp` | Write algorithm outputs to flash for downlink |
| `imageWrite` | Output | `Fw::Dp` | Write compressed flagged images to flash for downlink |
| `pingIn` / `pingOut` | In/Out | `Svc::Ping` | Health monitoring via Svc::Health |
| `logOut` | Output | `Fw::Log` | Event logging |
| `tlmOut` | Output | `Fw::Tlm` | Telemetry (images processed, failures) |

### 3.3 Commands

None. ScienceInferenceManager is fully schedule-driven.

### 3.4 Execution Flow

1. On `schedIn`: query flash for unprocessed images
2. For each unprocessed image:
   a. Read experiment type from image metadata
   b. Invoke LOST, FOUND, or SCOPE accordingly
   c. Store algorithm outputs for downlink with experiment ID and timestamp
   d. If image is flagged: compress and write to flash for downlink
   e. Mark image as processed
   f. On algorithm failure: assert WARNING_HI and continue to next image

---

## 4. State Machine

ScienceInferenceManager uses an F' state machine (`Fw::Sm`) with the following phases:

- `[Startup]` — initialization
- `IDLE` — waiting for next schedule tick
- `QUERYING` — polling flash for unprocessed images
- `PROCESSING` — running algorithm on current image
- `STORING` — writing results and compressed images to flash

---

## 5. External Libraries

| Library | Algorithm | Camera | Experiment Type |
|---------|-----------|--------|----------------|
| LOST | Lost-in-space star identification | Camera 1 | L&F |
| FOUND | Follow-up optical navigation | Camera 2 | L&F |
| SCOPE | Star catalog optical processing (runs LOST internally) | Camera 1 + 2 | Calibration |

Libraries are included via CMake and invoked directly from ScienceInferenceManager C++ implementation. No F' worker components wrap them at this stage.

---

## 6. Notes

- ScienceInferenceManager does not delete images. Image purge is handled separately.
- Flash polling rate is determined by rate group assignment (nominally 0.1Hz or lower to avoid compute contention).
- SCOPE runs LOST as an internal preprocessing stage; ScienceInferenceManager passes calibration images to SCOPE only.
