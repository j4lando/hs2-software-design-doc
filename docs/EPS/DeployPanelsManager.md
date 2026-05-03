# DeployPanelsManager SDD

## 1. Overview

`DeployPanelsManager` is a Layer 2 Active component that executes the solar panel deployment sequence when commanded by `EPSApplication`. On receipt of the deploy command it asserts the burn wire GPIO for a fixed duration defined as a component constant, then deasserts it and emits a completion event. There is no automatic retry — if deployment must be reattempted, a new command must be issued from ground or `SatStateMachine` via `EPSApplication`.

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
| `deploy` | Input | `Fw.Signal` | Deploy command forwarded from EPSApplication |
| `burnWire` | Output | `Drv.GpioWrite` | Burn wire GPIO; asserted for deployment duration then deasserted |
| `logOut` | Output | `Fw.Log` | Event logging |

### 3.3 Commands

None. `DeployPanelsManager` is driven entirely by the `deploy` port call from `EPSApplication`, which itself accepts the ground `DEPLOY_PANELS` command.

---

## 4. State Machine

`DeployPanelsManager` uses a flat three-state F' state machine:

```
IDLE
  └─ (on deploy signal) emit deployment-initiated event
                        assert burnWire GPIO high
                        transition → DEPLOYING

DEPLOYING
  └─ (after burn wire duration constant elapses)
                        deassert burnWire GPIO
                        emit deployment-complete event
                        transition → DEPLOYED

DEPLOYED
  └─ (on deploy signal) emit WARNING_HI "already deployed" event; ignore signal
                        (terminal state — no further deployment without reboot)
```

---

## 5. Notes

- TBD whether state machine is needed or not.
- Burn wire active duration is a hardcoded constant in the component. Value TBD pending hardware team specification.
- `DeployPanelsManager` transitions to `DEPLOYED` permanently on first successful deployment. Re-deployment after reboot is by design — the panel deployment mechanism is one-shot hardware.
- Timing the burn wire duration within an Active component (rather than using a rate group tick) is TBD — options include a timer port or a busy-wait with an OS sleep call. To be resolved during detailed design.
