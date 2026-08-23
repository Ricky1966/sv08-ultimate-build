<p align="center">
  <img src="docs/assets/phoenix-readme-logo.png" alt="Phoenix — Sovol SV08 Mainline Klipper" width="720">
</p>

# SV08 Ultimate Build



> [!WARNING]
> ## Disclaimer
>
> This is an **unofficial community project**. It is not affiliated with, sponsored by, or endorsed by Sovol or by any of the other projects or manufacturers referenced here.
>
> The procedures, configurations, and modifications documented in this repository come from a migration that was actually performed and tested on a specific Sovol SV08, but **they do not guarantee compatibility or correct operation on every machine**.
>
> Klipper, Katapult, Eddy, Phoenix Macros, and the other software components evolve over time. The material documented here represents a verified state of this project and is not necessarily the current state of the art.
>
> Firmware flashing, configuration changes, wiring, calibration, and machine movements can result in loss of the original configuration, malfunction, or hardware damage if performed incorrectly.
>
> **!!! You use this repository at your own risk. !!!** Before making any changes, keep verified backups and make sure you understand the operations you are performing.
>
**Languages:** [Italiano](README.md) | **English**

Community documentation for migrating a **Sovol SV08** from the factory software stack to **current Klipper Mainline**, including:

- CB1/Linux setup;
- MCU backup, flashing and recovery;
- replacement of the original toolhead with the Sovol Zero Extruder Kit;
- replacement of the original bed with an R3men graphite bed;
- Mainline printer configuration;
- native Klipper Eddy support;
- Phoenix Macros, developed and validated during the Phoenix migration;
- calibration;
- troubleshooting;
- rollback.

The guides are based on a real end-to-end migration of a Sovol SV08 nicknamed **Phoenix**, but the repository is structured so that Phoenix-specific values and history are kept separate from the reusable community baseline.

## Start here

If you are migrating a Sovol SV08, follow the guides in this order:

1. [Getting started](docs/en/getting-started.md)
2. [Backup and rollback](docs/en/backup-and-rollback.md)
3. [CB1 and Klipper Mainline installation](docs/en/install-cb1-mainline.md)
4. [MCU flashing and recovery](docs/en/flash-mcus.md)
5. [Sovol Zero toolhead, native Eddy and graphite bed](docs/zero-toolhead-eddy-2026-08-17.md)
6. [Base Mainline configuration](docs/en/base-configuration.md)
7. [Native Eddy](docs/en/native-eddy.md)
8. [Phoenix Macros](docs/phoenix-macros.md)
9. [Validation and calibration](docs/en/validation-and-calibration.md)
10. [Troubleshooting](docs/en/troubleshooting.md)

A compatibility index for older links is also available:

[Klipper Mainline migration](docs/en/mainline-migration.md)

## Read this before flashing anything

> [!WARNING]
> **Do not start by erasing or flashing the MCU boards.**
>
> For additional safety, the initial Phoenix migration work was performed from **MicroSD rather than directly from eMMC**. A MicroSD card does not necessarily provide the same reliability or lifespan as a good eMMC module, but it has an important practical advantage: if something goes wrong, it can be replaced quickly and cheaply with readily available media, without depending on the lead times or replacement cost of a compatible eMMC module. This makes MicroSD particularly useful as a test, recovery, and initial-migration medium.

Before changing firmware, complete at least:

- an inventory of the stock system;
- backup of `printer.cfg` and all included configuration;
- backup of Moonraker and custom macros;
- a system/eMMC rollback plan;
- personal dumps of the original MCU firmware when possible;
- verification that the recovery procedure is understood.

See:

[Backup and rollback](docs/en/backup-and-rollback.md)

## What this repository covers

The community path currently documents:

- migration away from the factory Sovol Klipper environment;
- CB1-based Linux environment used by the stock Sovol controller;
- Klipper Mainline;
- mainboard and toolboard firmware migration;
- Katapult-based recovery/update path;
- stable MCU identification using `/dev/serial/by-id/`;
- native `[probe_eddy_current ...]` support;
- homing Z with native Eddy;
- QGL and bed mesh;
- Phoenix Macros integration;
- OrcaSlicer Machine G-code integration;
- first-layer validation;
- filament calibration;
- troubleshooting based on real failure cases.

## Verified migration baseline

The Phoenix migration reached a fully operational Mainline system with:

- Klipper Mainline `v0.13.0-718-gd8659974-dirty`;
- Klipper commit `d865997403cad36d105026f73a4b76dcacec4c76`;
- Moonraker;
- Mainsail;
- KlipperScreen;
- mainboard and toolboard on Mainline-compatible firmware;
- native Eddy using `[probe_eddy_current eddy]`;
- full `G28`;
- native probing;
- QGL;
- bed mesh;
- Phoenix Macros;
- OrcaSlicer;
- successful real printing after migration.

These version numbers describe the tested Phoenix baseline. They are not a requirement to remain frozen on those exact commits.

For a new migration, always compare with the current upstream documentation before installing or patching software.

## Native Eddy

The old Phoenix migration initially used the external `probe_eddy_ng` path.

That path was eventually abandoned because the current Mainline environment already provides native Eddy support and the legacy plugin created problems in the Z-homing path.

The current community documentation therefore treats:

`[probe_eddy_current ...]`

as the baseline.

See [Native Eddy](docs/en/native-eddy.md).

Phoenix-specific values such as offsets, `reg_drive_current`, `max_sensor_hz`, and calibration curves are examples, not universal presets.

## Phoenix Macros

The current Phoenix configuration uses its own macro set on top of Klipper Mainline.

On August 22, 2026, the separation audit from DKEU was completed. The current runtime has:

- **21 `PHOENIX_*` macros correctly defined and exposed by Klipper**;
- Core Pack;
- Calibration Pack;
- Debug Pack;
- Setup Pack;
- no active DKEU include;
- no DKEU runtime dependency in Phoenix files;
- no `M84`, `force_move`, or `save_variables` in the Phoenix layer;
- native Klipper bed mesh with rapid scan and adaptive meshing in the print workflow;
- external components kept separate and attributed to their respective projects.

Current macros include `PHOENIX_START`, `PHOENIX_END`, `PHOENIX_CLEAN_NOZZLE`, filament management, PID/Input Shaper calibration, probe/mesh/sensor diagnostics, and setup helpers.

On August 23, 2026, the filament workflow was physically validated on the printer: `PHOENIX_LOAD_FILAMENT`, `PHOENIX_UNLOAD_FILAMENT`, `PHOENIX_FILAMENT_RUNOUT`, and `PHOENIX_FILAMENT_CHANGE` all completed their respective workflows correctly.

`PHOENIX_PRESSURE_ADVANCE_TEST` was intentionally removed: Pressure Advance calibration is delegated to the slicer, avoiding a second redundant tower procedure.

DKEU was used during an important stage of the Phoenix migration and remains a historical/upstream source that should be credited where appropriate, but **it is no longer an operational dependency of the current baseline**.

See [Phoenix Macros](docs/phoenix-macros.md).

## Troubleshooting philosophy

The migration intentionally follows an evidence-first approach.

If homing works, probing is repeatable, QGL converges, and the mesh is coherent, but printing is still wrong, inspect macro overrides, Z realignment, slicer profile, generated G-code, and software migration leftovers before changing the gantry, Z motors, bed screws, axis twist, or mechanical geometry.

Several Phoenix failures that initially looked mechanical were ultimately software/configuration problems.

See [Troubleshooting](docs/en/troubleshooting.md).

## Phoenix case study

The complete chronological migration history is preserved separately under:

`docs/migration-history/phoenix/`

It contains:

- intermediate states;
- failed experiments;
- temporary workarounds;
- measurements;
- root-cause investigations;
- final fixes.

This history is useful for debugging and technical archaeology. It is **not** the recommended step-by-step installation procedure.

## Phoenix examples

Phoenix-specific examples are stored under:

`examples/phoenix/`

These may include:

- hardware notes;
- OrcaSlicer profiles;
- project roadmap;
- machine-specific values.

Do not assume these files can be copied unchanged to another SV08.

## Calibration

The repository separates three classes of calibration:

### Machine

Examples:

- PID;
- stepper configuration;
- endstops;
- heaters.

### Bed / Z

Examples:

- Eddy calibration;
- QGL;
- Z reference;
- mesh;
- first layer.

### Filament

Examples:

- temperature;
- flow ratio;
- pressure advance;
- retraction;
- max volumetric speed.

See [Validation and calibration](docs/en/validation-and-calibration.md).

## Hardware modifications

The community Mainline baseline is intentionally kept separate from later Phoenix hardware development.

Future or machine-specific changes may include:

- enclosure modifications;
- insulation;
- umbilical redesign.

These modifications should not redefine the basic migration procedure.

## Upstream projects

This project builds on work from the Klipper and SV08 communities, including:

- Klipper;
- Moonraker;
- Mainsail;
- KIAUH;
- Katapult;
- Rappetor's Sovol SV08 Mainline work;
- Demon Klipper Essentials Unified;
- OrcaSlicer.

Always consult the relevant upstream project before applying old workarounds from migration history.

## Language policy

Italian is the primary language of this repository. English is maintained as the complete second-language community path.

Phoenix development history and machine-specific archival notes may remain Italian when they are not part of the reusable community procedure.

## License

Original material in this repository is distributed under the **GNU General Public License v3.0 or later**, to the extent that copyright in that material is held by the project authors.

See:

- [`LICENSE`](LICENSE) — full license text;
- [`NOTICE.md`](NOTICE.md) — attributions, third-party material, derived presets, and trademarks.

Referenced or used upstream projects retain their respective licenses.
