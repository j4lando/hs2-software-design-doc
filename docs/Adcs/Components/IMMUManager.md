# IMMUManager SDD

## 1. Overview


## 2. Requirements
ID|Requirement|Verification
-|-|-
ADCS-SW-IMMU-001|[TBD]|[TBD]
ADCS-SW-IMMU-002|[TBD]|[TBD]

## 3. Design
### 3.1. Component Type
Queued component with internal FPrime state machine.
### 3.2. Mode Interface
The IMMU manager does not rely on other components for its operation.

## 4. State Machine

initial enter RESET

signal tick
signal success
signal error

action doReset
action checkReset
action doConfigure
action doActive
action doRead

state RESET{
on success enter WAIT_RESET 
on tick do { doReset } 
}
state WAIT_RESET{
on success enter CONFIGURE 
on error enter RESET
on tick do { checkReset }
}
state CONFIGURE{
on success enter ACTIVE
on error enter RESET
on tick do { doConfigure }
}
state ACTIVE {
on error enter RESET
on tick do { doRead }
}

## 5. FPrime Interfaces
### 5.1. Commands
Command|Args|Description
-|-|-
RESET|-|Resets the IMMU to factory settings then re-initializes from the state machine's RESET state,

### 5.2. Events
Event|Args|Severity|Description
-|-|-|-
`immuStTrans`|st: `IMMUStateMachine_t`|activity low|records state transitions of the immu
`immuError`|code: `VN100Errors`|warning high|records immu errors
`uartError`|code: `Drv.ByteStreamStatus`|warning high|records uart communication errors        

### 5.3. Ports
Port|Direction|Type|Description
-|-|-|-
`getIMUData`|sync input|`imuDataRead_p`|port used to retrieve imu data
`getTemp`|sync input|`tempDataRead_p`|port used to retrieve temperature data
`getMagnetometer`|sync input|`getMagnetometer`|ort used to retrieve magnetometer data
`toggleMagnetorquers`|output|`mtToggle_p`|port used to request locking and unlocking of the 
`busWriteRead`|output|`Drv.ByteStreamData`|TODO
`busWrite`|output|`Drv.ByteStreamData`|TODO

### 5.4. Telemetry
Telemetry|Type|Description
-|-|-
`immuFailCtr`|U32|number of IMMU initialization fails
`immuStTransCtr`|U32|number of IMMU state transitions
`immuTempCtr`|U32|number of IMMU temperature readings
`rawIMU`|`RawIMUData`|most recent raw accel, gyro, and temperature readings from the IMMU
`rawTemp`|`RawTempData`|most recent temp readings from the IMMU
`magnetometerCtr`|U32|number of magnetometer readings
`rawB`|`RawBData`|most recent B reading from the IMMU

## 6. Notes

## 7. See Also
- `IMMUTypes` 
