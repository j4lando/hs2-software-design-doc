# MpptIcManager SDD

## 1. Overview

`MpptIcManager` is the Layer 2 Active component responsible for all communication with the BQ25756 MPPT and battery charging IC. No other component reads or writes the BQ25756 directly.

On each rate group tick in `RUNNING` state, `MpptIcManager` triggers a one-shot ADC conversion, reads all relevant measurement and status registers over I2C (voltages, currents, charging status, fault flags), assembles the data into a state struct, and calls `EPSApplication`'s `batteryStateIn` port. When `EPSApplication` forwards a configuration command (charging enable/disable, MPPT enable/disable, threshold writes), `MpptIcManager` writes the appropriate BQ25756 registers over I2C and emits an event. When the BQ25756 INT interrupt fires, `MpptIcManager` reads the fault registers, emits a fault event, and reinitializes the IC.

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
| `chargingControl` | Input | `Fw.On` | Enable/disable charging — called by EPSApplication |
| `mpptControl` | Input | `Fw.On` | Enable/disable MPPT — called by EPSApplication |
| `setChargingCurrentLimit` | Input | Custom port (U32 mA) | Charging current limit write — called by EPSApplication |
| `setJeitaThreshold[4]` | Input | Custom port | JEITA T1/T2/T3/T5 threshold writes — called by EPSApplication |
| `setRechargeThreshold` | Input | Custom port | Recharge threshold write — called by EPSApplication |
| `watchdogControl` | Input | `Fw.On` | Enable/disable BQ25756 internal watchdog — called by EPSApplication |
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
             transition → RUNNING
             emit "transitioned to RUNNING" event

RUNNING
  └─ (on schedIn tick) trigger one-shot ADC
                       read measurement registers (vbatt, ibatt, vac, iac, vfb)
                       read configuration registers (vrechg, vbat_lowv, ichg)
                       read charging status register
                       read fault/flag registers
                       assemble batteryState struct
                       call batteryStateOut port
                       emit telemetry channels
     (on configuration port call) write register(s) over I2C
                                  emit configuration-changed event
     (on intPin interrupt) read fault status registers
                           emit WARNING_HI fault event
                           transition → UNINITIALIZED

# Fault recovery from any state:
on intPin interrupt → enter UNINITIALIZED
```

**UNINITIALIZED behavior for incoming port calls:** Configuration port calls received during `UNINITIALIZED` are either queued until `RUNNING` is entered or dropped — exact behavior TBD during detailed design.

Reference: [FPP substates](https://github.com/nasa/fpp/blob/main/docs/users-guide/Defining-State-Machines.adoc#substates)

---

## 5. Notes

- The current port interface uses one input port per BQ25756 configuration operation (e.g., `setJeitaThreshold[4]`, `setChargingCurrentLimit`, `setRechargeThreshold`). A preferred alternative is a single `setRegister` port taking a register address and value, reducing the port count substantially. To preserve readability, register addresses should be expressed as a named enum (e.g., `BQ25756Reg::JEITA_T1`, `BQ25756Reg::ICHG_LIMIT`) rather than raw hex values. This consolidation is deferred to detailed design.
- Whether a formal state machine is strictly necessary is TBD; the key requirement is a post-reset initialization delay with register verification before processing requests.
- `batteryStateOut` port type is a custom struct. Consider splitting into a measurements port and a flags port if the struct becomes unwieldy; see `EPSApplication` notes for the matching `batteryStateIn` discussion.
- INT interrupt port type depends on the LinuxGpioDriver version in use — exact port type to be confirmed during detailed design.
- BQ25756 registers relevant to initialization: internal watchdog enable (`wd_stat`), charging threshold registers whose power-on defaults are not acceptable. Full register list TBD pending hardware bring-up.
