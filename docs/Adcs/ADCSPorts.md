# ADCSPorts SDD

## 1. Overview
The ADCS subsystem implements custom port types, which are defined in their own dedicated files. Components may refer to ports specified here. All ports here refer to either integral FPrime types or to types defined by the ADCS subsystem.

## 2. Ports
|Name|Arg|Returns|Description
|-|-|-|-|
`satAttitude_p`|ref quat: `f32x4`|-|Used to exchange the quaternion attitude of the satellite.
`truthMeasureAttitude_p`|ref quat: `TimestampedOrientation`|-|Used to exchange timestamped truth measure readings.
`bState_p`|ref bState: `BState`|-|Used to exchange the state of the satellite's local magnetic field.
`attControl_p`|target_type: `ControlTypes`, target_info: `f32x4`|-|Used to control the attitude control system of the satellite. Note that the meaning of the target_info argument depends on the value of the target_type argument. If target_info is deactivate or detumble, target_info is meaningless; if target_type is point then target_info is the target quaternion orientation (w first); if target_type is slew then the __FIRST THREE INDICES__ are the desired x y and z slew in rad/s .
`mtDutycycle_p`|dc: `f32x3`|-|Used to control the dutycycle of three magnetorquers.
`imuDataRead_p`|ref data: `RawIMUData`|-|Used to get the raw accel/gyro data of the immu.
`bDataRead_p`|ref data: `RawBData`|-|Used to read the raw magnetometer data of the immu.
`tempDataRead_p`|ref data: `RawTempData`|-|Used to read the raw temperature data of the immu.
`mtToggle_p`|is_toggled: bool|-|Used to toggle the magnetorquer lock.
`sunVector_p`|ref sunVector: `f32x3`|-|Used to read the sun vector values.
`readAnalog_p`|pin: U16|F32|Used to read the specified analog pin.
