# StateEstimatorTypes SDD
## 1. Overview
This document lists the custom structs, enums, and other types implemented by the ADCS software for used with the `AttitudeController` component.


## 2. Types

### 2.1. Structs

### 2.2. Enums

#### 2.2.1. `ControlTypes`
Defines the type of control modes that the `AttitudeController` can be commanded to. 

|Mnemonic|Value|Description
|-|-|-|
deactivate|Unspecified|Command the `AttitudeController` to be deactivate.
detumble|Unspecified|Command the `AttitudeController` to detumble the satellite.
point|Unspecified|Command the `AttitudeController` to point the satellite with a specific quaternion.
slew|Unspecified|Command the `AttitudeController` to slew the satellite with a specified angular velocity.

### 2.3. Misc

# 3. See Also
- `AttitudeController` Component