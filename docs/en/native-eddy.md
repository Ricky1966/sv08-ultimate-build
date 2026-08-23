# Native Eddy — historical note and current Phoenix path

**Languages:** [Italiano](../native-eddy.md) | **English**

Last structural review: **2026-08-23**.

## Status of this page

This page is kept to preserve old links and explain the evolution of the Eddy system on the Sovol SV08 “Phoenix”.

The current Phoenix configuration **no longer uses the previous Sovol Eddy NG hardware connected to the original toolboard**.

The current baseline uses:

- **Sovol Zero Extruder Kit**;
- toolhead connected over **CAN**;
- Eddy probe integrated into the Zero;
- native Klipper Mainline Eddy support;
- Phoenix Macros.

The current operational guide is:

[Sovol Zero toolhead, CAN and integrated Eddy](zero-toolhead-eddy-2026-08-17.md)

## Historical path

During an earlier phase of the Phoenix migration, the Sovol Eddy NG / LDC1612 probe was used on the original toolboard.

That configuration included:

- `extra_mcu` toolboard MCU;
- I2C connection on the old toolboard;
- initial use of the `probe_eddy_ng.py` plugin;
- later migration to `[probe_eddy_current eddy]`;
- calibration and workarounds specific to that configuration;
- DKEU integration.

These details remain useful for reconstructing the debugging process and the decisions that led to the current configuration, but **they must not be used as presets for the Sovol Zero**.

The technical history is preserved under:

[migration-history/phoenix](../migration-history/phoenix/)

## What not to copy from the old configuration

Do not automatically transfer to the Sovol Zero values belonging to the previous Eddy NG configuration, including:

- old toolboard MCU names;
- I2C pins;
- X/Y offsets;
- `max_sensor_hz`;
- `reg_drive_current`;
- calibration curves;
- workarounds for `probe_eddy_ng.py`;
- DKEU macros created for the old path.

The Zero uses its own hardware configuration and calibration.

## Current operational path

For the current Phoenix configuration:

1. verify the mainboard and Mainline system;
2. configure the Sovol Zero and CAN bus;
3. configure the integrated Eddy probe;
4. calibrate Eddy on your own machine;
5. verify homing;
6. verify QGL;
7. verify rapid bed mesh;
8. validate Z and the first layer;
9. integrate Phoenix Macros.

The parameters and results actually validated on the Zero are documented on the dedicated page.

---

## Navigation

- ← **Previous page:** [Base Mainline configuration](base-configuration.md)
- → **Next page:** [Sovol Zero, CAN and integrated Eddy](zero-toolhead-eddy-2026-08-17.md)
