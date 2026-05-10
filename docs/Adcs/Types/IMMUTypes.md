#IMMUTypes SDD

## 1. Overview
This document lists the custom structs, enums, and other types implemented by the ADCS software used with the `IMMUManager` Component. Types defined in this document depend only on other types defined in this document and integral FPrime types.

## 2. Types

### 2.1. Structs

#### 2.1.1. `RawIMUData`
Snapshot of the angular velocity and linear acceleration measured by the VN-100. Includes a timestamp of when the measurement was taken.

Field|Type|Description
|-|-|-|
linearAccel|f32x3|acceleration in m/s^2
angularVelocity|f32x3|angular velocity rad/s
temp|F32|sensor temperature in degrees C
timestamp|U32|measurement timestamp in milliseconds


#### 2.1.2. `RawBData`
Snapshot of the local magnetic field measured by the VN-100. Includes a timestamp of when the measurement was taken.

Field|Type|Description
|-|-|-|
B|f32x3|local magnetic field in Tesla
timestamp|U32|measurement timestamp in milliseconds


#### 2.1.3. `RawTempData`

Field|Type|Description
|-|-|-|
temp|F32|sensor temperature in degrees C
timestamp|U32|measurement timestamp in milliseconds


### 2.2. Enums
#### 2.2.1. - `VN100Errors`

|Mnemonic*|Value*|Description*
|-|-|-|

*See the VN-100 datasheet and ICD for a description of different IMMU errors. They match exactly to the datasheet and are omitted here to preserve single source of truth.

### 2.3. Misc
#### 2.3.1. `f32x3`
    array [3]F32

# 3. See Also
- `IMMUManager` Component
