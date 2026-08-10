# Getting started — Sovol SV08 to Klipper Mainline

**Languages:** [Italiano](../getting-started.md) | **English**

Last source review: **2026-08-10**.

## Purpose

This repository documents a real and verified migration of a **Sovol SV08** from Sovol's modified Klipper software to **Klipper Mainline**, while preserving a rollback path and later integrating:

- Moonraker;
- Mainsail;
- KlipperScreen, when used;
- Demon Klipper Essentials Unified (DKEU);
- Sovol Eddy NG hardware through Klipper's native Eddy support;
- OrcaSlicer.

The goal is not to replace existing upstream guides, but to provide a second documented path based on a machine that was actually migrated all the way to successful printing.

The repository also records:

- attempts that did not work;
- legacy configurations incompatible with Mainline;
- observed symptoms;
- verified root causes;
- applied fixes;
- rollback procedures.

## Primary upstream source

The primary SV08 migration guide remains:

`Rappetor/Sovol-SV08-Mainline`

This repository should be considered complementary.

When an upstream procedure changes, the current documentation of the original project takes priority over old videos, screenshots, or local copies.

## Hardware used for validation

The migration was verified on a Sovol SV08 with:

- original Sovol/MKS Linux computer and mainboard environment with eMMC;
- original Sovol mainboard;
- original Sovol toolboard;
- STM32F103 MCU on mainboard and toolboard;
- Sovol Eddy NG / LDC1612 probe hardware;
- MicroSwiss FlowTech hotend;
- 0.4 mm nozzle.

The FlowTech is **not a requirement** for migrating to Mainline.

Machine-specific hardware modifications are documented separately under `examples/phoenix/`.

## Verified software baseline

The validation machine is currently operational with:

- Klipper Mainline;
- Moonraker;
- Mainsail;
- KlipperScreen;
- Crowsnest;
- Demon Klipper Essentials Unified;
- Eddy managed through `[probe_eddy_current eddy]`.

Klipper configuration verified at the end of the migration:

- version: `v0.13.0-718-gd8659974-dirty`;
- commit: `d865997403cad36d105026f73a4b76dcacec4c76`.

These identifiers describe the configuration that was actually tested and **do not mean that a new user must install exactly that commit**.

## CB1 image

The current `Rappetor/Sovol-SV08-Mainline` guide still explicitly identifies **CB1 V2.3.4** as the SV08 baseline and currently advises against V3.0.0 or later for this procedure.

Therefore, for this repository:

**CB1 V2.3.4 is the documented and verified baseline.**

Do not interpret "latest CB1 release" as "recommended version for SV08 Mainline".

A future image should only replace this baseline after a new end-to-end SV08 validation.

## Eddy: use current Mainline support

Current Klipper Mainline provides native Eddy support with several probing methods, including:

- default probing;
- scan;
- rapid scan;
- tap.

The machine used for this migration runs Sovol Eddy NG / LDC1612 hardware through:

`[probe_eddy_current eddy]`

Parameters validated on the test machine:

- `sensor_type: ldc1612`
- `i2c_mcu: extra_mcu`
- `i2c_scl: PB6`
- `i2c_sda: PB7`
- `x_offset: -16.43`
- `y_offset: 10.22`
- `descend_z: 0.5`
- `max_sensor_hz: 9000000`
- `reg_drive_current: 22`

These values describe the machine that was actually tested.

Eddy offsets and calibration must be verified on each printer and must not be copied blindly.

The old path based on a local `probe_eddy_ng.py` patch is not the path recommended by this repository for current Klipper Mainline.

During the migration of the test machine, that path was initially used and later abandoned in favor of Mainline's native Eddy support.

The complete transition history is preserved under:

`docs/migration-history/phoenix/`

## Demon Klipper Essentials Unified

DKEU is an active and evolving project.

The verified migration uses Demon with Klipper Mainline and native Eddy, but this repository does not distribute a frozen copy of Demon macros as the primary source.

Install and consult the current official repository:

`3DPrintDemon/Demon_Klipper_Essentials_Unified`

At the time of verification, DKEU is in the DKEU3 generation and includes dedicated handling for differences between modern Klipper Mainline and legacy/factory versions.

The Orca configuration verified on the test machine uses Demon Machine G-code `v1.4`.

Phoenix-specific configurations are kept separately under:

`examples/phoenix/`

## Before you start

Do not begin the migration unless you have:

1. a complete backup of `printer_data/config`;
2. copies of custom files and macros;
3. a backup or recovery plan for the original eMMC;
4. a verifiable rollback plan;
5. an ST-Link available for mainboard and toolboard;
6. a way to identify the two MCUs without ambiguity;
7. SSH access;
8. a microSD or eMMC usable for the new system;
9. enough time to validate the machine before homing or heating.

## Fundamental rule: preserve the stock system

The safest path is to keep a bootable stock system intact until Mainline has been verified end-to-end.

Do not treat a simple backup of `.cfg` files as equivalent to a complete rollback.

Before modifying or flashing the MCUs, it is strongly recommended to keep a verified copy of the original firmware from your own mainboard and toolboard.

The original dumps from the machine used for this project are not distributed in the community baseline.

Each user must save their own backups.

## General migration order

The documented path is:

1. inventory the stock machine;
2. back up required configuration and data;
3. prepare the CB1 storage medium;
4. boot the new Linux system;
5. install Klipper, Moonraker and Mainsail;
6. prepare the SV08 configuration;
7. back up original MCU firmware;
8. install Katapult;
9. flash Klipper Mainline to mainboard and toolboard;
10. verify temperatures, MCUs, fans and safety signals;
11. verify axes and endstops;
12. perform controlled homing;
13. integrate native Eddy;
14. integrate Demon;
15. run QGL and mesh;
16. calibrate Z;
17. perform the first print;
18. calibrate slicer and material.

Hardware and probing stages must be complete before the machine is considered ready to print.

## What NOT to copy blindly from the stock configuration

A configuration that works on Sovol Klipper is not automatically valid on Mainline.

During the real migration, several legacy elements caused problems, including:

- old Eddy integrations;
- macros written for APIs available in Sovol Klipper but not equivalent in Mainline;
- axis twist configuration no longer reliable after migration;
- local overrides that neutralized updated Demon macros.

## Verified case: `_APPLY_EDDY_Z_OFFSET`

The most important post-migration problem involved an old local override:

`[gcode_macro _APPLY_EDDY_Z_OFFSET]`

The override had previously been created for Sovol Klipper 0.12, where the updated Demon path was not compatible.

After moving to Mainline, the override was still present in `printer.cfg`.

As a result, the correct Demon macro was neutralized and the Z position measured by the probe was not applied correctly.

The observed symptom was a severely wrong first layer even though QGL and mesh appeared valid.

Removing the legacy override restored the correct path:

`_PROBE_TAP -> _APPLY_EDDY_Z_OFFSET -> printer.probe.last_probe_position.z -> SET_KINEMATIC_POSITION`

Reprinting the same G-code showed a clear improvement and confirmed the root cause.

This case is preserved in the Phoenix technical history and is also covered in troubleshooting documentation.

## How to read this guide

Each step should be interpreted as one of these categories:

- **Upstream requirement** — required or recommended by original upstream documentation;
- **Phoenix verified** — actually verified on the machine used for this project;
- **Troubleshooting** — solution to a problem that was actually encountered;
- **Optional** — change not required for a standard SV08 Mainline migration.

This distinction prevents a test-machine customization from being mistaken for a universal requirement.

## Next step

Before installing software or flashing an MCU:

**create and verify the backup.**

Continue with:

`docs/en/backup-and-rollback.md`
