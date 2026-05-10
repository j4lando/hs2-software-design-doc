# StateEstimatorTypes SDD
## 1. Overview
This document lists the custom structs, enums, and other types implemented by the ADCS software used with the `StateEstimator` Component. Types defined in this document depend only on types defined in `IMMUTypes`, other types defined in this document, and integral FPrime types.

## 2 Types

### 2.1 Structs

#### 2.1.1 `TimestampedOrientation`

Field|Type|Description
|-|-|-|
quat|`f32x3`|the quaternion orientation with order w, i, j, k.
timestamp_ms|U32|the timestamp of the measurement in milliseconds. Note that this is when the measurement was taken, ie if an image is taken and then some nontrivial time is taken to process the image and calculate an attitude, this timestamp should be when the image was initially taken not when the final attitude was calculated.


#### 2.1.2 `BState`

Field|Type|Description
|-|-|-|
b|`f32x3`|The local magnetic field in Tesla.
bDot|`f32x3`|The change in local magnetic field in Tesla/sec.

### 2.2 Enums

### 2.3 Misc
#### 2.3.1 `f32x4`
    array [4]F32

# 3. See Also
- `StateEstimator` Component
- `IMMUTypes` Type List