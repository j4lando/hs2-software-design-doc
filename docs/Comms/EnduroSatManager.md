# EnduroSatManager SDD

## 1. Overview
`EnduroSatManager` is the layer 4 active component for the Comms subtopology. It directly interacts with the Endurosat S-band radio using a UART interface. The `CommsApplication` component will forward commands and data to the `EnduroSatManager` to be fed into the Endurosat S-band radio.

---

## 2. Requirements
| ID | Requirement | Verification |
|----|-------------|-------------|

---

## 3. Design

## 3.1 Component Type
Passive component that implements the F' `Communication Adapter Interface` that specifies both the ports and protocols used to operate with the standard F´ uplink and downlink components.

## 3.2 Communication Adapter Interface

## 3.3 Ports
| Port | Direction | Type | Purpose |
|------|-----------|------|---------|

- `EnduroSatManager` is instantiated at the top-level topology (shared with `ComCcsds` subtopology); `CommsApplication` connects to it via the top-level topology wiring.
- Detailed high-gain link configuration and `EnduroSatManager` interface to be defined during detailed design.
