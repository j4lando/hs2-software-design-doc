# AdcsManager SDD

> **STATUS: DRAFT — Requirements only. Ports, commands, and state machine to be defined in next session.**

## 1. Overview

`AdcsManager` is an Active component responsible for attitude determination and control on the HS2 satellite. It nominally detumbles the spacecraft using IMU and sun sensor data, commands magnetorquers for attitude corrections, and supports precise pointing via the star tracker when requested for experiment operations.

---

## 2. Requirements

| ID | Requirement | Verification |
|----|-------------|-------------|
| HS2-ADC-001 | AdcsManager shall detumble the spacecraft using IMU and sun sensor data | Inspection |
| HS2-ADC-002 | AdcsManager shall control magnetorquers to execute attitude corrections | Inspection |
| HS2-ADC-003 | AdcsManager shall command precise pointing using the star tracker when requested | Inspection |
| HS2-ADC-004 | AdcsManager shall provide current attitude estimate for telemetry | Inspection |
| HS2-ADC-005 | AdcsManager shall reject experiment pointing requests when attitude error exceeds threshold | Inspection |
| HS2-ADC-006 | AdcsManager shall assert WARNING_HI if IMU health check fails | Inspection |
| HS2-ADC-007 | AdcsManager shall assert WARNING_HI if magnetorquer actuation fails | Inspection |
| HS2-ADC-008 | AdcsManager shall respond to health ping within the required deadline | Inspection |

---

## 3. Design

> To be completed in next session.

