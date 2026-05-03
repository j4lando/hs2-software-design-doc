# Current Architecture Status

**Date:** 2026-04-21  
**Purpose:** Capture what is already established in the F' rewrite and what still needs design work.

## Established Decisions

- **Three-layer architecture:** Flight software is organized as Layer 3 `*Application`, Layer 2 `*Manager`, and Layer 1 `*Driver`.
- **`SatStateMachine`:** Owns top-level satellite mode logic and sends typed mode commands to each Layer 3 application through dedicated ports.
- **Layer 3 hierarchical state machines:** `AdcsApplication`, `DataCollectionApplication`, `ScienceInferenceApplication`, and `CommsApplication` use hierarchical F' state machines where mode is the top-level state.
- **Layer 2 flat state machines:** Hardware managers such as `ImuManager`, `Camera1Manager`, and `StarTrackerManager` are not satellite-mode-aware. Their baseline pattern is `RESET -> WAIT_RESET -> ENABLE -> CONFIGURE -> RUN`, with reset-based recovery on error.
- **Current top-level mode model:** The repo currently defines `Safe` and `Standby`, with `Standby` submodes `Downlink`, `Science`, `Charge`, and `Eclipse`.
- **Current app set:** The active app-level design currently exists for ADCS, Data Collection, Science Inference, Comms, and EPS.
- **EPS architecture:** `EPSApplication` (Layer 3) is the EPS orchestrator — accepts charging, MPPT, threshold, and deploy commands from ground and `SatStateMachine`; publishes power state to `SatStateMachine`; has no mode interface and no hierarchical SM. Layer 2 hardware managers: `MpptIcManager` (BQ25756 over I2C), `HardwareResetManager` (overcurrent protection), `WatchdogPinger` (hardware watchdog GPIO), `DeployPanelsManager` (burn wire sequencer). See `docs/EPS/`.

## Open Questions

- **Thermal architecture:** Keep `ThermalManager` as a manager-only subsystem, or introduce a Layer 3 `ThermalApplication` if the control logic needs mode-aware orchestration?
- **CDH decomposition:** Map command handling, storage, parameters, health, and downlink behavior from the old PDF into F' pre-built components and any required custom pieces.
- **Ground software docs:** Translate the ground-software chapter from the PDF into markdown, or explicitly mark it out of scope for this repo revision.
- **Mode-switch behavior details:** Define what happens when an application receives a mode change mid-operation, especially for `DataCollectionApplication` and `AdcsApplication`.

## Immediate Next Step

- Decide Thermal next. Same question as EPS: manager-only or introduce a Layer 3 `ThermalApplication`? Thermal control logic is currently simple (poll sensors, command heater), but resolving this determines whether another application-level SDD is needed.
