# MpptIcManager SDD

## 1. Overview

`MpptIcManager` is the Layer 2 Active component responsible for all communication with the BQ25756 MPPT and battery charging IC. No other component reads or writes the BQ25756 directly.

On each rate group tick in `RUNNING` state, `MpptIcManager` triggers a one-shot ADC conversion, reads all relevant measurement and status registers over I2C (voltages, currents, charging status, fault flags), assembles the data into a state struct, and calls `EPSApplication`'s `batteryStateIn` port. It also maintains a per-rail consecutive-fault counter: each tick where a rail voltage is out of range emits a `WARNING_HI` event; after N consecutive out-of-range readings on a rail, a `FATAL` event is emitted. The FATAL routes through `EventManager.FatalAnnounce → fatalHandler` in CdhCore.

When `EPSApplication` forwards a register-write command via the `setRegister` port, `MpptIcManager` writes the specified BQ25756 register over I2C and emits an event. When the BQ25756 INT interrupt fires, `MpptIcManager` reads the fault registers, emits a fault event, and reinitializes the IC.

---

## 2. Requirements

TODO

---

## 3. Design

### 3.1 Component Type

Active component with internal flat F' state machine (`Fw::Sm`).

### 3.2 Ports

| Port | Direction | Type | Purpose |
|------|-----------|------|---------|
| `schedIn` | Input | `Svc.Sched` | Rate group tick; triggers ADC read cycle in RUNNING state |
| `i2c` | In/Out | `Drv.I2c` | I2C bus port to LinuxI2cDriver; all BQ25756 register reads and writes |
| `intPin` | Input | GPIO interrupt port | Fires asynchronously when BQ25756 asserts INT due to a fault — exact port type TBD pending driver version |
| `icReset` | Output | `Drv.GpioWrite` | Assert high to reset BQ25756; deassert after reset sequence completes |
| `batteryStateOut` | Output | Custom struct port | Battery and IC state to EPSApplication (vbatt, ibatt, vac, iac, charging status, fault flags, MPPT state, temperature) — type TBD |
| `setRegister` | Input | Custom port (`BQ25756Reg`, `U32`) | Register write called by EPSApplication; writes specified register over I2C |
| `logOut` | Output | `Fw.Log` | Event logging |
| `tlmOut` | Output | `Fw.Tlm` | Telemetry (vbatt, ibatt, vac, iac, charging status) |

---

## 4. State Machine

`MpptIcManager` uses a flat two-state F' state machine. This deviates from the standard `RESET → WAIT_RESET → ENABLE → CONFIGURE → RUN` hardware manager pattern because initialization requires an explicit IC reset pulse followed by a register verification pass before normal operation can begin.

```
UNINITIALIZED
  └─ (entry) assert icReset GPIO high
             deassert icReset after reset delay
             read configuration registers; write any that differ from desired values
             reset all per-rail fault counters to zero
             transition → RUNNING
             emit "transitioned to RUNNING" event

RUNNING
  └─ (on schedIn tick) trigger one-shot ADC
                       read measurement registers (vbatt, ibatt, vac, iac, vfb)
                       read configuration registers (vrechg, vbat_lowv, ichg)
                       read charging status register
                       read fault/flag registers
                       for each monitored rail:
                           if voltage out of range:
                               increment rail fault counter
                               emit WARNING_HI (rail ID, measured value)
                               if fault counter >= FAULT_COUNT_THRESHOLD:
                                   emit FATAL (rail ID)
                           else:
                               reset rail fault counter to zero
                       assemble batteryState struct
                       call batteryStateOut port
                       emit telemetry channels
     (on setRegister call) write register over I2C
                           emit configuration-changed event
     (on intPin interrupt) read fault status registers
                           emit WARNING_HI fault event
                           transition → UNINITIALIZED
```

**UNINITIALIZED behavior for incoming port calls:** `setRegister` calls received during `UNINITIALIZED` are either queued until `RUNNING` is entered or dropped — exact behavior TBD during detailed design.

Reference: [FPP substates](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#substates)

---

## 5. Notes

- `FAULT_COUNT_THRESHOLD` (N consecutive out-of-range readings before FATAL) is a hardcoded constant. Value TBD pending hardware characterization; should account for transient noise without masking real faults.
- Per-rail fault counters reset to zero on each transition into `UNINITIALIZED` (i.e., on IC reset/reinitialization).
- The FATAL event emitted after N consecutive bad readings routes through CdhCore: `EventManager.FatalAnnounce → fatalHandler`, which resets the system. The satellite reboots into Safe mode.
- `batteryStateOut` port type is a custom struct. Consider splitting into a measurements port and a flags port if the struct becomes unwieldy; see `EPSApplication` notes for the matching `batteryStateIn` discussion.
- Whether a formal state machine is strictly necessary is TBD; the key requirement is a post-reset initialization delay with register verification before processing requests.
- INT interrupt port type depends on the LinuxGpioDriver version in use — exact port type to be confirmed during detailed design.
- BQ25756 registers relevant to initialization: internal watchdog enable (`wd_stat`), charging threshold registers whose power-on defaults are not acceptable. Full register list TBD pending hardware bring-up.
- `BQ25756Reg` enum definition is shared with `EPSApplication` and defines all addressable registers by name. Produced during detailed design.
