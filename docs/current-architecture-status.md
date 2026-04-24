# Current Architecture Status

**Date:** 2026-04-21  
**Purpose:** Capture what is already established in the F' rewrite and what still needs design work.

## Established Decisions

- **Three-layer architecture:** Flight software is organized as Layer 3 `*Application`, Layer 2 `*Manager`, and Layer 1 `*Driver`.
- **`SatStateMachine`:** Owns top-level satellite mode logic and sends typed mode commands to each Layer 3 application through dedicated ports.
- **Layer 3 hierarchical state machines:** `AdcsApplication`, `DataCollectionApplication`, `ScienceInferenceApplication`, and `CommsApplication` use hierarchical F' state machines where mode is the top-level state.
- **Layer 2 flat state machines:** Hardware managers such as `ImuManager`, `Camera1Manager`, and `StarTrackerManager` are not satellite-mode-aware. Their baseline pattern is `RESET -> WAIT_RESET -> ENABLE -> CONFIGURE -> RUN`, with reset-based recovery on error.
- **Current top-level mode model:** The repo currently defines `Safe` and `Standby`, with `Standby` submodes `Downlink`, `Science`, `Charge`, and `Eclipse`.
- **Current app set:** The active app-level design currently exists for ADCS, Data Collection, Science Inference, and Comms.

## Open Questions

- **EPS architecture:** Keep `EpsManager` as a standalone Layer 2 component, or introduce a Layer 3 `EpsApplication` for higher-level power and fault-management logic?
- **Thermal architecture:** Keep `ThermalManager` as a manager-only subsystem, or introduce a Layer 3 `ThermalApplication` if the control logic needs mode-aware orchestration?
- **CDH decomposition:** Map command handling, storage, parameters, health, and downlink behavior from the old PDF into F' pre-built components and any required custom pieces.
- **Ground software docs:** Translate the ground-software chapter from the PDF into markdown, or explicitly mark it out of scope for this repo revision.
- **Mode-switch behavior details:** Define what happens when an application receives a mode change mid-operation, especially for `DataCollectionApplication` and `AdcsApplication`.

## Immediate Next Step

- Decide EPS first. It is the smallest unresolved architecture question that can change whether more Layer 3 application docs are needed.
