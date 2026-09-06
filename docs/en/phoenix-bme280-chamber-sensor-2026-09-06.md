# Phoenix — BME280 chamber environment sensor

Validation performed on the Sovol SV08 **Phoenix** on September 6, 2026.

This note documents the real, hardware-validated installation of a 4-pin breakout marked `BME/BMP280` (`SDA`, `SCL`, `GND`, `VIN`) used as the chamber environment sensor.

> [!IMPORTANT]
> The wiring below describes the Phoenix machine that was actually verified. Before reproducing it on another SV08, always check the pinout, voltages, and configuration of your own board.

## Validated wiring

The sensor is connected to the mainboard **EXP2** header.

Final working mapping:

| BME280 signal | MCU pin | Phoenix Klipper alias |
|---|---|---|
| SCL | `PC6` | `EXP2_5` |
| SDA | `PC7` | `EXP2_3` |
| GND | EXP2 GND | — |
| VIN | EXP2 supply measured at about `3.26 V` | — |

On Phoenix, the voltage measured directly across the breakout `VIN`/`GND` pins was about **3.26 V**.

The Sovol `printer.cfg` historically labels `EXP2_10` as `<5V>`; that label must not be treated as an electrical measurement. On Phoenix the actual voltage was checked with a multimeter before connecting the sensor.

## Conflict with the stock display

In the stock configuration the `[display]` section uses:

```ini
encoder_pins: ^EXP2_5, ^EXP2_3
```

Those are the same two pins reused by the BME280 software I2C bus. Phoenix no longer uses the stock UC1701 display, so the `[display]` section was disabled before assigning `EXP2_5` and `EXP2_3` to the sensor.

The separate `[output_pin beeper]` section was left enabled.

## Validated Klipper configuration

Final configuration:

```ini
[temperature_sensor Chamber_Temp]
sensor_type: BME280
i2c_address: 118
i2c_mcu: mcu
i2c_software_scl_pin: EXP2_5
i2c_software_sda_pin: EXP2_3
i2c_speed: 100000
```

Details:

- I2C address: decimal `118` = `0x76`;
- SCL: `PC6` / `EXP2_5`;
- SDA: `PC7` / `EXP2_3`;
- I2C speed: `100000` Hz.

## Debugging note

During installation the initial SCL/SDA assignment was reversed and Klipper reported:

```text
MCU 'mcu' I2C request to addr 118 reports error START_NACK
```

The same error also appeared when trying address `0x77`. The configuration started working immediately after correcting the two signal lines to:

```text
SCL = PC6 = EXP2_5
SDA = PC7 = EXP2_3
```

This is an important diagnostic detail because `START_NACK` does not necessarily mean the I2C address is wrong; it can also mean the device is not seeing SDA/SCL correctly.

## Runtime validation

Moonraker correctly exposed the object:

```text
temperature_sensor Chamber_Temp
```

with an initial reading such as:

```text
temperature: 28.25 °C
```

After a few minutes Mainsail simultaneously displayed:

- chamber temperature: about **32–33 °C**;
- pressure: about **980.7 hPa**;
- relative humidity: about **33–36 %**.

The sensor was visible both in **Mainsail in Chrome** and in the Mainsail interface embedded in **OrcaSlicer**.

## Status

**VALIDATED ON REAL HARDWARE — Phoenix, 2026-09-06.**
