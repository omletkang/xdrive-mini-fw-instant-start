# xdrive-mini-fw-instant-start

Modified ODrive v0.5.1 firmware for the **XDrive MINI** (an ODrive v3.6 compatible board by Makerbase/MKS) with the onboard **AS5047P** absolute encoder.

With this firmware the drive:

- **skips encoder offset calibration at power-on** and uses the saved encoder offset, and
- **starts from any rotor position** and goes straight into closed-loop control, without jumping.

It is based on the MKS firmware from [makerbase-mks/ODrive-MKS](https://github.com/makerbase-mks/ODrive-MKS) (`Firmware/MKS ODrive_S/ODrive_S-fw-v0.5.1`).

<p>
  <img src="images/xdrive-mini-mad5010-setup.jpg" width="49%" alt="XDrive MINI with a MAD 5010 110KV motor mounted on the back, powered from a 24 V bench supply">
  <img src="images/xdrive-mini-board.jpg" width="49%" alt="XDrive MINI board: motor phase terminals A/B/C, DC input, CAN and I/O connectors">
</p>

## Repository layout

```
xdrive-mini-fw-instant-start/
├── ODrive_S-fw-v0.5.1/   firmware source (MKS firmware + this modification)
├── xdrive_config.txt     odrivetool configuration used with this firmware
├── images/               photos of the test setup
├── README.md
└── LICENSE               GPL-3.0
```

## The problem

With the stock MKS firmware:

1. **Calibration every boot.** Setting `encoder.config.pre_calibrated = True` for the absolute encoder was cleared again by the firmware unless offset calibration had already run in the same power cycle. So the drive had to calibrate (spin the motor) on every power-on.
2. **Could only start at position zero.** When calibration was skipped, the position estimator (PLL) started at 0 instead of the real encoder angle. If the rotor was anywhere else at power-on, the estimate had to slew to the real position, causing a velocity spike and a jump or error when entering closed-loop control.

## What changed

All changes are in [`ODrive_S-fw-v0.5.1/Firmware/MotorControl/encoder.cpp`](ODrive_S-fw-v0.5.1/Firmware/MotorControl/encoder.cpp). The full diff is a single commit on top of the unmodified MKS import. See the commit history.

| Function | Change |
|---|---|
| `check_pre_calibrated()` | Absolute (SPI) encoders keep `pre_calibrated = True` without re-running offset calibration. Checks `config_.mode` instead of `mode_` (which is only refreshed at boot). |
| `abs_spi_cb()` | Until the first `update()`, seeds the PLL with the absolute SPI position and zero velocity. Re-seeds on every SPI read until then, so a bad first read from the AS5047P after power-on is overwritten. |
| `abs_spi_start_transaction()` | If SPI is still busy, skips that cycle instead of raising `ERROR_ABS_SPI_NOT_READY`. Sustained failures still raise `ERROR_ABS_SPI_COM_FAIL` via `spi_error_rate_`. |

## Usage

1. **Flash the firmware.** Download the `.hex` / `.elf` from the [Releases](../../releases) page, or build and flash it yourself (see [Building and flashing](#building-and-flashing)).
2. **Configure with odrivetool.** The full configuration is in [`xdrive_config.txt`](xdrive_config.txt). Adjust the board, motor and controller values to your setup. The encoder settings for the onboard AS5047P are:

   ```python
   odrv0.axis0.encoder.config.mode = ENCODER_MODE_SPI_ABS_AMS
   odrv0.axis0.encoder.config.abs_spi_cs_gpio_pin = 7
   odrv0.axis0.encoder.config.cpr = 16384
   ```

3. **Calibrate once and mark as pre-calibrated:**

   ```python
   odrv0.axis0.requested_state = AXIS_STATE_MOTOR_CALIBRATION
   # wait until it finishes
   odrv0.axis0.motor.config.pre_calibrated = True
   odrv0.axis0.requested_state = AXIS_STATE_ENCODER_OFFSET_CALIBRATION
   # wait until the motor stops turning
   odrv0.axis0.encoder.config.pre_calibrated = True
   odrv0.save_configuration()
   odrv0.reboot()
   ```

4. **Use it.** After a reboot, at any rotor position, you can go straight into closed loop:

   ```python
   odrv0.axis0.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL
   odrv0.axis0.controller.input_pos = odrv0.axis0.encoder.pos_estimate + 0.2
   ```

   To enter closed loop automatically at power-on, also set `odrv0.axis0.config.startup_closed_loop_control = True`.

> **Note:** the saved offset is only valid while the encoder magnet's position relative to the motor stays the same. If you remount the magnet or the motor, run the calibration again.

## Building and flashing

**Requirements** (Linux / WSL): `arm-none-eabi-gcc`, `tup`, `python3`, and `openocd` with an **ST-Link V2** for flashing. The board version is set in [`ODrive_S-fw-v0.5.1/Firmware/tup.config`](ODrive_S-fw-v0.5.1/Firmware/tup.config) (`CONFIG_BOARD_VERSION=v3.6-56V`).

Connect the ST-Link V2 to the board's SWD pins (SWDIO, SWCLK, GND, 3.3V), as shown in the MKS ODrive instruction video: <https://youtu.be/k9cBYpxeytk>

```bash
cd ODrive_S-fw-v0.5.1/Firmware

# one-time: the build scripts call `python`, so point it to python3
# (on Ubuntu, `sudo apt install python-is-python3` does the same)
sudo ln -sf $(which python3) /usr/bin/python

# clean build (also needed after moving or re-cloning the folder)
rm -rf .tup build autogen
tup init

make          # build only: output in build/ODriveFirmware.hex / .elf / .bin
make flash    # build and flash through the ST-Link V2
```

## Hardware tested

- XDrive MINI (ODrive v3.6 compatible, MKS)
- Onboard AS5047P absolute magnetic encoder (SPI, 14-bit)
- Motors (both 14 pole pairs):
  - [MAD M6C12 150KV](https://store.mad-motor.com/products/mad-m6c12-eee-brushless-motor-for-the-long-flight-time-multirotor-hexacopter-octopter)
  - [MAD 5010 110KV](https://mad-motor.com/products/mad-components-5010-eee?VariantsId=11308)

Motor and controller values for both are in [`xdrive_config.txt`](xdrive_config.txt). The main values are for the M6C12; comments marked `5010` give the values for the 5010.

## Credits and license

- Original firmware: [ODrive Robotics](https://github.com/odriverobotics/ODrive), MIT License, see [`ODrive_S-fw-v0.5.1/LICENSE.md`](ODrive_S-fw-v0.5.1/LICENSE.md). The original ODrive README is at [`ODrive_S-fw-v0.5.1/README.md`](ODrive_S-fw-v0.5.1/README.md).
- MKS board port: [Makerbase (MKS)](https://github.com/makerbase-mks/ODrive-MKS), distributed under GPL-3.0.
- Instant-start modification: Seunghoon Kang, 2026.

This repository is distributed under the **GNU GPL v3.0** (see [`LICENSE`](LICENSE)), following the MKS repository. The ODrive-derived code also remains under its original MIT notice.

This firmware is provided as-is, without warranty. Motors can move unexpectedly when entering closed-loop control, so make sure your mechanism is safe to move before testing.
