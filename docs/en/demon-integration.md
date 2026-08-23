# DKEU / Demon integration — historical Phoenix development phase

**Languages:** [Italiano](../demon-integration.md) | **English**

Last review of historical sources: **2026-08-10**.

> [!IMPORTANT]
> This page documents an **earlier phase of Phoenix development**.
>
> DKEU played an important role in the migration path, but **it is no longer a runtime dependency of the current Phoenix configuration**.
>
> The current baseline uses [Phoenix Macros](phoenix-macros.md).

## Historical purpose

This guide preserves documentation of the Demon Klipper Essentials Unified integration used during one phase of the Phoenix migration, when the machine had already been migrated to:

- Klipper Mainline;
- Mainline MCU firmware;
- native Eddy;
- working homing;
- working QGL;
- verified probing.

Demon must be added only after the basic hardware configuration is stable.

Do not use Demon to hide problems that still exist in:

- MCU communication;
- steppers;
- endstops;
- heaters;
- homing;
- probe.

## Upstream source

Official repository:

`3DPrintDemon/Demon_Klipper_Essentials_Unified`

At the time of verification, the project is in the:

`DKEU3`

generation.

DKEU is an active project.

If you want to use DKEU independently from Phoenix, always use current upstream documentation and files.

Do not use an old Phoenix copy as the primary source.

## Current DKEU and Klipper Mainline

Current DKEU automatically distinguishes between:

- modern Klipper Mainline;
- legacy/factory Klipper versions.

This makes some manual changes that were historically necessary on stock Sovol systems obsolete.

Do not set legacy variables just because an old guide did so when the current DKEU version no longer requires them.

## Prerequisites

Before installing Demon, all of the following must work:

- Mainsail;
- Moonraker;
- Klipper Mainline;
- mainboard MCU;
- toolboard MCU;
- X/Y/Z homing;
- native Eddy;
- QGL;
- probing;
- temperatures;
- fans.

The machine must remain controllable without Demon.

## Installation

Install DKEU following the current method documented by the official project.

Do not manually copy only a few files from an old Phoenix installation.

DKEU uses an ecosystem of:

- core assets;
- user files;
- variables;
- helpers;
- optional shell scripts;
- slicer configuration.

A partial installation can leave macros that appear present but are internally inconsistent.

## KIAUH

Current DKEU documentation also requires attention to the KIAUH version and its shell-script extension.

If Demon File Handler operations do not work:

- verify KIAUH;
- verify the shell extension;
- do not immediately diagnose a Klipper problem.

## User configuration

DKEU separates core files from user files.

Customizations should be made in the locations intended by the project.

Do not directly modify core assets except for temporary, controlled debugging.

A local core modification:

- can be overwritten by an update;
- can make it unclear whether a problem belongs to Demon or to the customization;
- makes comparison with upstream more difficult.

## Phoenix variables observed

During Phoenix validation, the configuration included values equivalent to:

- Orca integration enabled/disabled depending on the test stage;
- adaptive meshing enabled;
- KAMP smart park enabled;
- slicer flow not used by Demon in that stage;
- slicer pressure advance not used by Demon in that stage;
- purge lines enabled;
- nozzle cleaner present;
- Klicky probe absent;
- bed fans absent;
- chamber heater absent;
- Nevermore absent.

These values describe the Phoenix configuration at that time.

They are not a universal DKEU baseline.

## OrcaSlicer

Current DKEU requires Machine G-code consistent with the installed version.

The Phoenix migration was validated with Machine G-code:

`v1.4`

The verified start string contained:

`DEMON_START`

and:

`DMGCC="v1.4"`

along with parameters supplied by OrcaSlicer.

## Importance of selecting the correct printer profile

During the Phoenix migration, the first print start with Demon failed because OrcaSlicer was still set to the Sovol machine instead of the Phoenix machine.

The generated G-code did not contain the expected Demon Machine Start.

Demon therefore performed an emergency shutdown.

The cause was not:

- Eddy;
- Klipper;
- MCU firmware;
- QGL.

It was the wrong printer profile in the slicer.

Before diagnosing Demon, always verify:

- printer preset;
- filament preset;
- process preset;
- Machine Start G-code.

## `_APPLY_EDDY_Z_OFFSET`

Current DKEU contains an `_APPLY_EDDY_Z_OFFSET` macro that uses:

`printer.probe.last_probe_position.z`

to realign the Z coordinate through:

`SET_KINEMATIC_POSITION`

This is the correct behavior verified on Phoenix with Klipper Mainline.

Do not locally redefine `_APPLY_EDDY_Z_OFFSET` without a demonstrated need.

## Historical Phoenix problem — legacy override

Before the Mainline migration, Phoenix had a local override that neutralized `_APPLY_EDDY_Z_OFFSET`.

It had been needed on old Sovol Klipper because the field used by Demon was unavailable.

After moving to Mainline, that override became harmful.

The local block replaced the current Demon macro and prevented correct Z realignment after probing.

## Observed effect

During the failed first print, `_PROBE_TAP` estimated contact at approximately:

`z=-0.060846`

but the old override did not apply the correction.

The resulting error of about:

`0.061 mm`

was enough to severely compromise a `0.20 mm` first layer.

## Correction

The solution was not to modify Demon.

The old local override was removed from `printer.cfg`.

After restart, the active DKEU macro again correctly used:

`printer.probe.last_probe_position.z`

and:

`SET_KINEMATIC_POSITION`

Reprinting the same G-code showed a radical improvement.

## Migration rule

When moving from Sovol Klipper to Mainline, always inspect `printer.cfg` for macros that redefine DKEU macros.

In particular, look for:

- `_APPLY_EDDY_Z_OFFSET`
- `_PROBE_TAP`
- `_MESH_HANDLING`
- homing macros;
- probe macros;
- QGL wrappers;
- bed-mesh wrappers.

An old local redefinition can take precedence over the updated version installed by Demon.

## TAP — current state

Current DKEU recognizes native Eddy support and handles the presence of `tap_threshold` through the configuration that was actually loaded.

During the Phoenix migration, a temporary correction was required because the DKEU version installed at that time misinterpreted the Mainline default value.

That patch belongs to Phoenix history.

Do not automatically apply it to current DKEU versions.

## `tap_threshold`

Mainline Eddy TAP semantics have changed over time.

Do not recover old values such as:

`tap_threshold: 300`

from previous Eddy NG configurations.

If TAP is desired:

- follow current Klipper documentation;
- run the required calibration;
- verify the resulting value on your own machine.

## QGL and mesh under Demon

Demon can wrap and coordinate operations such as:

- homing;
- QGL;
- mesh;
- heat soak;
- purge;
- print start.

That means a manual test using a native Klipper command is not necessarily identical to the path used during `DEMON_START`.

During debugging it is important to distinguish:

- native Klipper behavior;
- Demon wrapper behavior;
- complete print-start behavior.

## Native test versus wrapper

During the Phoenix migration, native commands were also useful to isolate probe and mesh behavior.

In particular, Demon macros could force a `rapid_scan` path.

When you need to verify Klipper without the wrapper, use the base command or alias provided by your configuration.

Do not change mechanical configuration just because behavior inside Demon differs from a manual test.

First reconstruct the macro path that actually ran.

## Adaptive meshing

DKEU can manage adaptive meshing.

Phoenix used this feature during the Demon workflow.

Before enabling it, verify that:

- native probing works;
- manual mesh works;
- probe coordinates are correct;
- mesh area is compatible with offsets;
- the slicer supplies the required information.

Do not use adaptive meshing as the first probe test.

## Purge lines

Enable purge lines only after:

- Z is reliable;
- start G-code is correct;
- nozzle cleaning is safe;
- purge coordinates are physically reachable.

A purge line should not be the first movement that reveals a Z error.

## Nozzle cleaner

Phoenix uses a physical nozzle cleaner.

This is machine-specific hardware and not a DKEU requirement.

If present, verify:

- position;
- height;
- toolhead clearance;
- safe path;
- cleaning temperature;
- absence of collisions.

Phoenix coordinates must not be copied to a stock SV08 or a different toolhead.

## First `DEMON_START`

Before the first real start, separately verify:

- `G28`;
- probing;
- QGL;
- mesh;
- heaters;
- fans;
- nozzle cleaner, if present;
- purge, if present.

Only then test the complete `DEMON_START` path.

This reduces the number of possible causes when something fails.

## Reconstruct the start from logs

If a print fails immediately after `DEMON_START`, do not modify the machine immediately.

Reconstruct from the log:

1. homing performed;
2. probing performed;
3. resulting Z;
4. QGL;
5. mesh loaded or created;
6. any Z realignment;
7. heaters;
8. purge;
9. actual first-layer start.

This approach identified the Phoenix root cause without unnecessary mechanical intervention.

## Phoenix verified — diagnostic sequence

In the Phoenix case:

- manual QGL was stable;
- manual mesh was stable;
- probing was coherent;
- the first layer during Demon was still wrong.

That correctly shifted attention from mechanics to the macro path.

Log verification then showed that `_PROBE_TAP` measured contact correctly but the legacy override prevented application of the offset.

## Unproven observation — `_MESH_HANDLING`

During Phoenix diagnosis, a possible logic issue was noticed in the probe-detection condition of `_MESH_HANDLING`.

The observed condition used `or` between absence checks.

In theory, this could make the condition true even when one supported probe was present.

It was not demonstrated to be the cause of the printing problem.

It was not modified.

For new installations:

- verify behavior in the current DKEU version;
- do not apply a patch based on this historical observation without concrete reproduction.

## Unproven observation — Eddy detection

In one DKEU snapshot used during the Phoenix migration, a condition explicitly considered:

`probe_eddy_current btt_eddy`

but did not always consider:

`probe_eddy_current eddy`

This was also not demonstrated as the cause of the failed print.

It must not be presented as a confirmed bug.

Verify the current DKEU version directly before making changes.

## DKEU updates

DKEU evolves quickly.

After an update:

- read upstream notes;
- verify variable renames;
- verify required Machine G-code;
- verify user-file changes;
- verify Eddy/TAP compatibility;
- run a controlled test before a long print.

Do not assume a configuration that was perfect on a previous version remains automatically valid after an update.

## Relationship with upstream DKEU

DKEU remains an independent upstream project.

Historical Phoenix configurations only document the parameters and incompatibilities encountered on the SV08; Demon core files should be obtained and updated from the upstream project.

## Orca validation

Before the first real print, verify that generated G-code contains the markers and parameters required by the installed DKEU version.

For the validated Phoenix configuration, these were present:

- `DEMON_START`
- `DMGCC="v1.4"`
- `_SPS GSTART=True`

These values describe the workflow verified during the migration period.

For future installations, always check current DKEU documentation.

## First validation print

Use a print that is:

- short;
- known;
- easy to inspect on the first layer;
- not combined with simultaneous changes to material, mechanics, and macros.

When correcting a problem, reprint the same G-code when possible.

That gives a meaningful A/B comparison.

Phoenix used exactly this method to confirm the Z-realignment fix.

## Do not correct several variables at once

If the first layer is wrong, avoid changing all of these together:

- Z offset;
- flow;
- QGL;
- mesh;
- Z motors;
- temperatures;
- pressure advance;
- Demon macros.

Change one variable at a time after identifying the actual path of the error.

## Exit criteria

Demon integration can be considered successful when:

- current DKEU is installed without errors;
- user files remain separate from core files;
- Klipper Mainline is detected correctly;
- native Eddy is detected;
- `G28` works;
- `_PROBE_TAP` works;
- `_APPLY_EDDY_Z_OFFSET` applies the correct realignment;
- QGL works;
- mesh works;
- Orca Machine G-code is consistent with DKEU;
- `DEMON_START` completes its workflow;
- the real first layer is coherent;
- no legacy Sovol override neutralizes current DKEU macros.

---

## Navigation

- ← **Previous page:** [Sovol Zero, CAN and integrated Eddy](zero-toolhead-eddy-2026-08-17.md)
- → **Next page:** [Phoenix Macros](phoenix-macros.md)
