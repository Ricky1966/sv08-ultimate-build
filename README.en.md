# SV08 Ultimate Build

**Languages:** [Italiano](README.md) | **English**

Community documentation for migrating a **Sovol SV08** from the factory software stack to **current Klipper Mainline**, including:

- CB1/Linux setup;
- MCU backup, flashing and recovery;
- Mainline printer configuration;
- native Klipper Eddy support;
- Demon Klipper Essentials Unified / DKEU3;
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
5. [Base Mainline configuration](docs/en/base-configuration.md)
6. [Native Eddy](docs/en/native-eddy.md)
7. [Demon / DKEU3 integration](docs/en/demon-integration.md)
8. [Validation and calibration](docs/en/validation-and-calibration.md)
9. [Troubleshooting](docs/en/troubleshooting.md)

A compatibility index for older links is also available:

[Klipper Mainline migration](docs/en/mainline-migration.md)

## Read this before flashing anything

Do **not** start by erasing or flashing the MCU boards.

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
- DKEU3 integration;
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
- DKEU;
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

## Demon / DKEU3

Demon Klipper Essentials Unified is integrated only after the basic Mainline machine is already working.

Do not use Demon as a substitute for validating MCU communication, stepper directions, heaters, endstops, homing, Eddy, or QGL.

The guide also documents an important real migration lesson: a legacy local macro override can silently replace a newer DKEU macro and create a printing problem even when probing, QGL and mesh all appear correct.

See [Demon / DKEU3 integration](docs/en/demon-integration.md).

## Troubleshooting philosophy

The migration intentionally follows an evidence-first approach.

If homing works, probing is repeatable, QGL converges, and the mesh is coherent, but printing is still wrong, inspect macro overrides, Z realignment, slicer profile, generated G-code, and software migration leftovers before changing the gantry, Z motors, bed screws, axis twist, or mechanical geometry.

Several Phoenix failures that initially looked mechanical were ultimately software/configuration problems.

See [Troubleshooting](docs/en/troubleshooting.md).

## Phoenix case study

The complete chronological migration history is preserved separately under:

`docs/migration-history/phoenix/`

It contains intermediate states, failed experiments, temporary workarounds, measurements, root-cause investigations, and final fixes.

This history is useful for debugging and technical archaeology. It is **not** the recommended step-by-step installation guide and is currently maintained in Italian.

## Phoenix examples

Phoenix-specific examples are stored under:

`examples/phoenix/`

These may include hardware notes, OrcaSlicer profiles, project roadmap, and machine-specific values.

Do not assume these files can be copied unchanged to another SV08.

## Calibration

The repository separates three classes of calibration:

### Machine

Examples: PID, stepper configuration, endstops, heaters.

### Bed / Z

Examples: Eddy calibration, QGL, Z reference, mesh, first layer.

### Filament

Examples: temperature, flow ratio, pressure advance, retraction, max volumetric speed.

See [Validation and calibration](docs/en/validation-and-calibration.md).

## Hardware modifications

The community Mainline baseline is intentionally kept separate from later Phoenix hardware development.

Future or machine-specific changes may include graphite bed, toolhead redesign, CAN bus, EBB36 / EBB42, enclosure modifications, SSR, insulation, and umbilical redesign.

These modifications should not redefine the basic migration procedure.

## Repository rules

Do not commit passwords, Wi-Fi credentials, tokens, private keys, SSH host keys, personal MCU dumps intended only for rollback, full private eMMC images, or machine-specific secrets.

Before making a branch or release public, perform a privacy audit of current files, Git history, branches, tags, and release assets.

## Firmware and binary artifacts

Personal original firmware dumps belong in local backup storage, not in the public repository history.

Prepared migration firmware should be reproducible from documented build settings whenever possible.

If large binary images or sanitized system images are distributed in the future, GitHub Releases are preferred over normal Git history.

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
