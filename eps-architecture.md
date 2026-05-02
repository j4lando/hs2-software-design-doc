# EPS Architecture — Component Descriptions and Port Definitions

---

## EPS Manager

### What it does

On each rate group tick, the EPS manager reads the latest battery and IC state it has received from the MPPT IC Manager and runs its autonomous protection checks against that data. The specific thresholds and actions are TBD pending testing, but the inputs available to it are ibatt, vbatt, fault flags, temperature, and charging status. Importantly, the EPS manager will not autonomously enable/disable MPPT or charging - only the satellite state machine and ground station are allowed to do that. It may autonomously do other stuff like change charging thresholds based on the information it receives (this may require a state machine within the EPS manager, TBD).

When a command or port call arrives to enable/disable charging, the EPS manager forwards it to the MPPT IC Manager via a typed output port. Same for enable/disable MPPT. When a deploy panels command arrives, it forwards it to the Deploy Panels component. There is no command refusal logic at this time.

On each rate group tick (or whenever new data arrives from the MPPT IC Manager, TBD), the EPS manager pushes the latest health/status data out through its port to the satellite state machine via an output port.

### Ports

Note: for some of these (in particular, the output operations that write to a register), we could compress the number of ports by using F Prime's type system. For example, we could combine `Enable MPPT` and `Disable MPPT` in to one port with type `Fw.On`.

| Port | Direction | Type | Notes |
|---|---|---|---|
| Rate group | Input | `Svc.Sched` | Standard fprime rate group port |
| Commands (Enable/disable charging/MPPT, deploy panels) | Input | `Fw.Cmd` | Ground command via command dispatcher |
| Respond to commands | Output | `Fw.CmdResponse` | Ground command via command dispatcher |
| Battery/IC state (from MPPT IC Manager) | Input | Type not set yet — likely a custom struct port carrying all BQ25756 and battery data (vbatt, ibatt, vac, iac, charging status, fault flags, MPPT enabled state, temp). Could be split into a measurements port and a status/flags port if the struct becomes unwieldy | Populated by MPPT IC Manager on each of its rate group ticks |
| EPS health/status (to state machine) | Output | Type not set yet — custom port carrying a struct with: vbatt (mV), ibatt (mA), mppt status (bool or enum), fault flags (bitmask or individual bools — see fault flag note below), temperature (see temperature note below), charging status (enum: NOT_CHARGING, TRICKLE_CHARGE, PRE_CHARGE, FAST_CHARGE, TAPER_CHARGE, TOP_OFF_TIMER_ACTIVE, CHARGE_TERMINATION_DONE). Could remain one port or be split — most likely one port since the state machine will consume all fields together | Pushed to the satellite state machine on each rate group tick or on data update |
| Charging enable/disable (to MPPT IC Manager) | Output | Type not set yet — could be a boolean port or two separate void ports | Forwarded from ground command or state machine action |
| MPPT enable/disable (to MPPT IC Manager) | Output | Type not set yet — same options as charging port | Forwarded from ground command or state machine action |
| Set charging current limit (to MPPT IC Manager) | Output | Type not set yet — likely a port taking a U32 in mA | Forwarded from ground command or autonomous logic |
| Set JEITA T1 threshold (to MPPT IC Manager) | Output | Type not set yet | -- |
| Set JEITA T2 threshold (to MPPT IC Manager) | Output | Type not set yet | -- |
| Set JEITA T3 threshold (to MPPT IC Manager) | Output | Type not set yet | -- |
| Set JEITA T5 threshold (to MPPT IC Manager) | Output | Type not set yet | -- |
| Set recharge threshold (to MPPT IC Manager) | Output | Type not set yet | -- |
| Enable/disable watchdog (to MPPT IC Manager) | Output | Type not set yet - bool port? | -- |
| Deploy (to Deploy Panels) | Output | Type not set yet - likely a void port? | Forwarded from deploy panels ground command |

---

## MPPT IC / Battery Charging Manager

### What it does

This component is the sole owner of all communication with the BQ25756 IC. Nothing else reads or writes the IC directly.

On each rate group tick while in `RUNNING` state, the component enables a one-shot ADC reading, reads all relevant ADC registers over I2C (vbatt, ibatt, vac, iac, vfb), reads the configuration registers needed for derived values (vrechg, vbat_lowv, ichg), reads the charging status register, and reads the fault/flag registers. It assembles this data into its output struct and calls the port to the EPS Manager. It also emits telemetry channels for vbatt, ibatt, vac, iac, and charging status.

When an incoming port call arrives from the EPS Manager (e.g., disable charging, change JEITA threshold), the component writes the appropriate register(s) over I2C and emits an event describing what changed.

When the INT GPIO interrupt fires, the component reads the fault status registers to identify the fault type, emits a fault event, then transitions to `UNINITIALIZED` and asserts the IC reset GPIO line.

### State Machine

Note: We may not actually need a state machine, this is a TBD. The main purpose of having a state machine would be that we wanted to wait after a power-on-reset - either for a set number of seconds, or until after we'd written some values (i.e. charging thresholds, etc) to the BQ, before processing other requests.

#### `UNINITIALIZED`

This is the entry state on boot. It is also re-entered whenever the INT interrupt triggers a fault.

On entry, the component asserts the IC reset GPIO line high (to reset the BQ25756), then deasserts it after a brief delay. It then reads the registers it cares about configuring — specifically the internal watchdog enable register (`wd_stat`) and any charging threshold registers whose defaults are not acceptable. It compares the read values to the desired values. For any that differ, it writes the desired value. After all reads and writes are complete, it transitions to `RUNNING` and emits a "transitioned to RUNNING" event.

During `UNINITIALIZED`, all incoming port calls from the EPS Manager (configure MPPT, configure charging, etc.) are either queued or dropped (TBD). 

#### `RUNNING`

Normal operation. Incoming configuration port calls are processed immediately and forwarded to the IC over I2C. 

Transition back to `UNINITIALIZED` is triggered by the INT interrupt or similar errors.

### Ports

Note: for some of these, we could compress the number of ports by using F Prime's type system. For example, we could combine `Enable MPPT` and `Disable MPPT` in to one port with type `Fw.On`.

| Port | Direction | Type | Notes |
|---|---|---|---|
| Rate group | Input | `Svc.Sched` | Standard fprime rate group port |
| I2C | Input/Output | `Drv.I2c` | fprime Linux I2C driver port; used for all register reads and writes to BQ25756 |
| INT pin (interrupt) | Input | Type not set yet — fprime Linux GPIO driver has an interrupt-capable input port; the exact port type depends on the driver version in use | Fires asynchronously when BQ25756 asserts INT due to a fault |
| IC reset line | Output | `Drv.GpioWrite` | Asserted high to reset the BQ25756; deasserted after reset sequence completes |
| Enable charging | Input | Type not set yet — void port or bool port | Called by EPS Manager |
| Disable charging | Input | Type not set yet — void port or bool port | Called by EPS Manager |
| Enable MPPT | Input | Type not set yet | -- |
| Disable MPPT | Input | Type not set yet | -- |
| Set charging current limit | Input | Type not set yet — likely a port taking a U32 in mA | -- |
| Set JEITA T1 threshold | Input | Type not set yet | -- |
| Set JEITA T2 threshold | Input | Type not set yet | -- |
| Set JEITA T3 threshold | Input | Type not set yet | -- |
| Set JEITA T5 threshold | Input | Type not set yet | -- |
| Set recharge threshold | Input | Type not set yet | -- |
| Enable/disable watchdog | Input | Type not set yet — bool port | -- |
| Battery/IC state | Output | Type not set yet — either custom struct or an array | Called on each rate group tick in RUNNING state; carries all measurement and status data |

---

## Hardware Component Reset Manager

### What it does

On each rate group tick, the component reads each monitored power line's current sense signal. For each line, if the reading exceeds the hardcoded threshold for that line, the component asserts the corresponding reset GPIO output and emits two events: one indicating overcurrent was detected on that line (including the measured value), and one indicating that a reset was asserted.

As of right now, this component doesn't yet talk with the EPS manager. 

**Open hardware question**: I believe of the current sense ICs uses I2C (check with Chase Rushing), but there might be other sensors that don't. We should either be reading i2c, gpio, or perhaps from an analog pin (in this last case, we would need an ADC driver)

### Ports

| Port | Direction | Type | Notes |
|---|---|---|---|
| Rate group | Input | `Svc.Sched` | |
| Current sense, line 1..N | Input | Type not set yet, possibly `Drv.GpioRead` | One port per monitored component |
| Reset line, component 1..N | Output | `Drv.GpioWrite` | One per monitored component; asserted when that component's current sense exceeds threshold |

---

## Hardware Watchdog Timer Pinger

### What it does

On each rate group tick, toggles the watchdog GPIO pin to produce a pulse that satisfies the hardware watchdog on the EPS board. Very simple component.

### Ports

| Port | Direction | Type | Notes |
|---|---|---|---|
| Rate group | Input | `Svc.Sched` | Rate must be within the watchdog timeout window; exact frequency TBD with hardware team |
| Watchdog ping | Output | `Drv.GpioWrite` | |

---

## Deploy Panels Component

### What it does

When the deploy command is received, the component asserts the burn wire GPIO output and emits a "deployment sequence initiated" event. After the burn wire has been active for the required duration, it deasserts the GPIO and emits a "deployment sequence complete" event. The burn wire duration is a constant defined in the component.

There is no automatic retry. If deployment needs to be attempted again, the command must be resent from ground or satellite state machine.

### Ports

| Port | Direction | Type | Notes |
|---|---|---|---|
| Deploy panels | Input | `Fw.Cmd` | Received from EPS Manager, which forwards it from the ground command dispatcher |
| Burn wire GPIO | Output | `Drv.GpioWrite` | Asserted for burn wire duration, then deasserted |
