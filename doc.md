# Calibrate IMU Example

The `calibrate_imu` overlay prints the fused yaw, pitch and roll angles from
`DroneState` so you can confirm that the IMU axis mappings match the airframe.
The values are refreshed every 100 ms on both the AT7456E OSD and the
`UART_DEBUG` console. The build focuses purely on the IMU and accelerometer, so
you can tilt the board and immediately check whether each axis lines up with
the body frame definition.

## Building and running

1. Copy the `aspid` directory from this example into the CubeIDE project,
   replacing the existing folder. Only the files provided by the overlay are
   required.
2. Build and flash the firmware. The debug LED pulses twice when the example
   starts.
3. Connect the video output that carries the AT7456E overlay and open a serial
   terminal on `UART_DEBUG` (115200 8N1 by default).
4. Place the aircraft or IMU module on a flat surface and wait a few seconds
   for the angles to stabilise.

The OSD shows the fused attitude together with the live accelerometer vector in
g so you can confirm which body axis currently points downward (≈ −1 g) or
upward (≈ +1 g). The right-hand column runs a short gyroscope bias calibration
routine and prints the averaged offsets in rad/s:

```
Yaw:   XXX.XX deg
Pitch: XXX.XX deg
Roll:  XXX.XX deg
AccelX:  +0.01 g
AccelY:  -0.02 g
AccelZ:  -1.00 g
Tip: Tilt to map xyz
GyCal RUN
Elapsed  2.3s
Samples   023
BiasX +0.0012
BiasY -0.0008
BiasZ +0.0005
Copy to board_hw
```

The serial console mirrors the same numbers and adds the accelerometer values in
g:

```
IMU angles Yaw 12.34 Pitch -1.20 Roll 0.55
Accel g   X +0.01 Y -0.02 Z -1.00
```

## Verifying axis directions

Follow the sequence below to ensure the sensor axes are aligned with the body
frame:

1. **Level check** – With the frame perfectly level all three angles should sit
   close to 0 deg. `AccelZ` should hover near −1 g while `AccelX/Y` remain close
   to 0 g. Small offsets are normal and can be trimmed later with sensor bias
   calibration.
2. **Pitch test** – Tilt the nose upward about the pitch axis. The reported
   pitch should increase (positive values). Tilting the nose downward should
   drive the pitch negative. If the sign is reversed, flip `IMU_AXIS_SIGN_Y`
   after verifying the accelerometer mapping in the sections below. If the axis
   does not respond, revisit `IMU_AXIS_MAP_Y`.
3. **Roll test** – Drop the right wing (roll clockwise when looking forward).
   Roll should increase positively. Lifting the right wing should make the roll
   negative. Adjust `IMU_AXIS_SIGN_X` or `IMU_AXIS_MAP_X` if the behaviour is
   incorrect.
4. **Yaw test** – Rotate the nose clockwise when viewed from above. Yaw should
   increase. Rotating counter-clockwise should decrease the angle. Yaw comes
   from integrating the gyro, so it may drift slowly but it should still
   respond to deliberate rotations during the test. Use `IMU_AXIS_SIGN_Z` and
   `IMU_AXIS_MAP_Z` when corrections are required.

Repeat the sequence after each modification until all three axes respond with
the expected sign and magnitude.

Once the `GyCal` line switches to `DONE`, record the three bias values and paste
them into the `IMU_GYRO_CAL_OFFSET_*` macros located in
`Core/Inc/board_hw.h`. Rebuild the firmware so the offsets are subtracted from
every gyro sample during normal ASPID operation. The reported biases already
include any offsets currently configured in `board_hw.h`, so you can copy them
directly without clearing the existing macros first.

## Selecting the orientation macro

Axis orientation is configured in `Core/Inc/board_hw.h`. The IMU drivers already
remap the raw accelerometer and gyroscope channels into the standard chip frame
listed in the integration guide. Choose the sensor pose relative to the airframe
with the orientation helper and rebuild the firmware:

```c
#define IMU_ORIENTATION IMU_AXIS_DOWN
#include "aspid/config/imu_orientation.h"
```

`IMU_AXIS_DOWN*` keeps the sensor Z axis pointing downward and rotates the X/Y
axes clockwise by 0°, 90°, 180° or 270°. `IMU_AXIS_UP*` mirrors the sensor so
its Z axis points upward while automatically flipping the signs to stay aligned
with the FRD body frame. The helper expands to the traditional
`IMU_AXIS_MAP_*` and `IMU_AXIS_SIGN_*` macros, so you can still fine-tune the
mapping if none of the presets match your board exactly.

After adjusting the orientation rebuild the firmware, flash the board and repeat
the checks above. Once pitch, roll and yaw move in the expected directions the
IMU axes are calibrated correctly.

## Using the accelerometer readout

The extra OSD and UART lines display the raw accelerometer sample in g. Place
the airframe on a flat surface and confirm that the axis mapped to body +Z
reports approximately −1 g while the others sit near 0 g. Adjust the
`IMU_ORIENTATION` selection (or the expanded `IMU_AXIS_*` macros) until the
reported values match the expected orientation for each test in the checklist
above. The live readout makes it easy to double-check the mapping before
reflashing the configuration.

## Mapping and signing the accelerometer

Use the live accelerometer readout to assign each sensor channel to the body
axes and choose the correct sign macros:

1. **Body +Z (down)** – Keep the frame level so the belly points down. The
   channel hovering near −1 g is the sensor axis that should feed
   `IMU_AXIS_MAP_Z`. If it reads around +1 g flip `IMU_AXIS_SIGN_Z` to `-1`.
2. **Body +X (forward)** – Stand the airframe on its tail or otherwise point the
   nose straight up. The channel that now reports approximately ±1 g corresponds
   to the forward axis; set `IMU_AXIS_MAP_X` accordingly and flip
   `IMU_AXIS_SIGN_X` if the value is negative so that nose-up tilt produces a
   positive accelerometer reading.
3. **Body +Y (right)** – Roll the frame onto its left side so the right wing is
   pointing upward. The channel near +1 g identifies the rightward axis. Map it
   to `IMU_AXIS_MAP_Y` and invert `IMU_AXIS_SIGN_Y` when the reading is
   negative so that raising the right wing yields a positive accelerometer
   response.

Repeat the same checks for gyroscope responses after confirming the
accelerometer. Tilting or rotating the frame should cause the corresponding
`IMU_AXIS_*` channel to respond with the correct sign because both sensors share
the board-level mapping.

After updating the macros rebuild and flash the firmware, then repeat the pitch,
roll and yaw tests to confirm the angles follow the expected directions.
