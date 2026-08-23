# Getting started — Sovol SV08 to Klipper Mainline

**Languages:** [Italiano](../getting-started.md) | **English**

Last source review: **2026-08-10**.

## Purpose

This repository documents a real and verified migration of a **Sovol SV08** from Sovol's modified Klipper software to **Klipper Mainline**, while preserving a rollback path and later integrating:

- Moonraker;
- Mainsail;
- KlipperScreen, when used;
- Phoenix Macros;
- Sovol Zero Extruder Kit with integrated Eddy probe, managed through Klipper's native Eddy support;
- OrcaSlicer.

The goal is not to replace existing upstream guides, but to provide my own documented path based on a machine that was actually migrated all the way to successful printing.

The repository also records:

- attempts that did not work;
- legacy configurations incompatible with Mainline;
- observed symptoms;
- causes of problems that were actually verified;
- applied fixes;
- rollback procedures.

## Primary upstream source

The primary SV08 migration guide remains:

[Rappetor/Sovol-SV08-Mainline](https://github.com/Rappetor/Sovol-SV08-Mainline)

This repository should be considered complementary.

When an upstream procedure changes, the current documentation of the original project takes priority over old videos, screenshots, or local copies.

## Hardware used for validation

The migration was verified on a Sovol SV08 with:

- Sovol/MKS Linux computer;
- migration and validation system booted from **MicroSD**;
- original Sovol mainboard;
- **Sovol Zero Extruder Kit**;
- STM32F103 MCU on the mainboard;
- **Eddy probe integrated into the Sovol Zero**, managed through Klipper's native Eddy support;
- 0.4 mm nozzle.

Machine-specific hardware modifications are documented separately under `examples/phoenix/`.

## Verified software baseline

The validation machine is currently operational with:

- Klipper Mainline;
- Moonraker;
- Mainsail;
- KlipperScreen;
- Crowsnest;
- Phoenix Macros;
- Eddy integrated into the Sovol Zero and managed through `[probe_eddy_current eddy]`.

Klipper configuration verified at the end of the migration:

- version: `v0.13.0-718-gd8659974-dirty`;
- commit: `d865997403cad36d105026f73a4b76dcacec4c76`.

These identifiers describe the configuration that was actually tested and **do not mean that a new user must install exactly that commit**.

## CB1 image

The `Rappetor/Sovol-SV08-Mainline` guide still explicitly identifies **CB1 V2.3.4** as the SV08 baseline and currently advises against V3.0.0 or later for this procedure.

For this repository:

**CB1 V2.3.4 is the documented and verified baseline.**

Do not interpret "latest CB1 release" as "recommended version for SV08 Mainline".

A future image should only replace this baseline after a new end-to-end SV08 validation.

## Eddy: native Klipper Mainline support on the Sovol Zero

The Phoenix configuration uses the Eddy probe integrated into the **Sovol Zero Extruder Kit**, managed directly through Klipper Mainline's native Eddy support.

The configuration uses:

`[probe_eddy_current eddy]`

Klipper Mainline supports several Eddy probing methods, including:

- standard probing;
- scan;
- rapid scan;
- tap.

The Sovol Zero-specific configuration and the Eddy parameters validated on Phoenix are documented in the dedicated page:

[Sovol Zero toolhead, CAN and integrated Eddy](zero-toolhead-eddy-2026-08-17.md)

These values describe **the Phoenix configuration that was actually tested** and must not be treated as universal presets.

Offsets, calibration, Eddy curve, and probe behavior must be verified on each machine.

During an earlier phase of Phoenix development, an external path based on `probe_eddy_ng.py` was used. That approach was later abandoned after the migration to Klipper Mainline's native Eddy support.

The complete technical history remains available under:

`docs/migration-history/phoenix/`

## Phoenix Macros

The current configuration uses an autonomous macro set developed during the Phoenix migration.

The **Phoenix Macros** were progressively derived, rewritten, and adapted while working with Klipper Mainline, native Eddy support, and the actual machine configuration.

Demon Klipper Essentials Unified (DKEU) played an important role during one phase of development and is acknowledged as an upstream reference project, but **it is no longer a runtime dependency of the current Phoenix configuration**.

The current baseline:

- does not include DKEU;
- does not depend on DKEU macros at runtime;
- uses dedicated `PHOENIX_*` macros;
- uses Klipper's native bed mesh with rapid scan and adaptive meshing;
- integrates the print workflow directly with OrcaSlicer.

The Phoenix Macros currently include functions for:

- print start and end;
- nozzle cleaning;
- filament loading, unloading, and change;
- runout handling;
- calibrations;
- probe, mesh, sensor, and stepper diagnostics;
- machine setup procedures.

Macros that are exposed and loaded by Klipper must not be confused with macros that have already been physically validated: the validation documentation explicitly distinguishes the two states.

See:

[Phoenix Macros](phoenix-macros.md)

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
2. create a complete backup of configuration, data, and rollback capability;
3. prepare the **MicroSD** with the CB1 image;
4. boot the new Linux system for the first time and verify SSH/network access;
5. install KIAUH, Klipper Mainline, Moonraker, Mainsail, and the required components;
6. prepare the base SV08 configuration;
7. back up the original MCU firmware;
8. install Katapult and flash Mainline on the required MCUs;
9. verify MCUs, temperatures, endstops, fans, and outputs before any movement;
10. install and configure the **Sovol Zero Extruder Kit** and its CAN connection;
11. configure the **Eddy probe integrated into the Zero** through Klipper's native support;
12. verify axes and endstops under controlled conditions;
13. perform controlled homing;
14. calibrate and validate Eddy;
15. run QGL;
16. run bed mesh / rapid scan and verify compensation;
17. calibrate/reference Z and validate the first layer;
18. install and configure the **Phoenix Macros**;
19. integrate with OrcaSlicer;
20. verify the complete `PHOENIX_START` → print → `PHOENIX_END` workflow;
21. perform the first real print;
22. calibrate filament and slicer settings;
23. validate accessory and diagnostic functions.

Hardware, MCU, probing, and safety stages must be complete before the machine is considered ready to print.

## What NOT to copy blindly from the stock configuration

A configuration that works on Sovol Klipper is not automatically valid on Mainline.

During the real migration, several legacy elements caused problems, including:

- old Eddy integrations;
- macros written for APIs available in Sovol Klipper but not equivalent in Mainline;
- axis twist configuration no longer reliable after migration;
- local overrides and remnants of earlier configurations no longer compatible with the Phoenix baseline.

## How to read this guide

Each step should be interpreted as one of these categories:

- **Upstream requirement** — required or recommended by the original documentation;
- **Phoenix verified** — actually verified on the machine used for this project;
- **Troubleshooting** — solution to a problem that was actually encountered;
- **Optional** — change not required for a standard SV08 Mainline migration.

This distinction prevents a test-machine customization from being mistaken for a universal requirement.

---

## Navigation

- ← **Previous page:** [README](../../README.en.md)
- → **Next page:** [Backup and rollback](backup-and-rollback.md)
