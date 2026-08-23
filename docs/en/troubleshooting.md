# Troubleshooting — Sovol SV08 Mainline migration

**Languages:** [Italiano](../troubleshooting.md) | **English**

Last review: **2026-08-23**.

## Purpose

This guide collects problems actually encountered during the Phoenix migration and turns them into a reusable diagnostic procedure.

The structure is:

- symptom;
- checks;
- probable cause;
- what not to touch;
- solution.

Not every historical Phoenix problem still exists in current software versions.

Obsolete workarounds are explicitly marked as such.

## Fundamental rule

Do not change several subsystems at the same time.

Before modifying:

- mechanics;
- Z;
- QGL;
- mesh;
- macros;
- slicer;

identify the layer where the problem actually originates.

## Diagnostic order

When the machine is not working correctly, verify in this order:

1. Linux/CB1;
2. USB/CAN;
3. MCU;
4. Klipper configuration;
5. endstops;
6. homing;
7. Eddy;
8. QGL;
9. mesh;
10. Phoenix Macros;
11. slicer;
12. first layer.

This prevents correcting a downstream symptom while the real cause is upstream.

# Linux / CB1

## The machine is not reachable over the network

### Check

- current IP address;
- NetworkManager state;
- Ethernet interface;
- Wi-Fi interface;
- default route;
- DNS;
- NTP.

### Phoenix case

Phoenix used both:

- direct local Ethernet;
- Wi-Fi for Internet access.

The Ethernet profile also installed a default route with better priority than Wi-Fi.

Internet and DNS traffic was therefore sent toward a local Ethernet network with no Internet access.

### Solution

Configure Ethernet as a local-only link without a default route.

Leave the Internet default route to Wi-Fi.

### What not to touch

Do not modify Klipper, Moonraker, or MCU firmware when:

- the machine responds locally;
- the problem concerns only DNS/Internet/NTP.

## Incorrect date and time

### Check

- Internet reachability;
- DNS;
- routes;
- NTP service.

### Phoenix case

NTP was active, but peers showed `reach 0`.

The cause was network configuration, not NTP itself.

### Solution

Restore routes and DNS first.

Then verify NTP again.

# MCU / USB

## Do not use `ttyACM0` / `ttyACM1`

Identifiers such as:

- `/dev/ttyACM0`
- `/dev/ttyACM1`

are not stable.

They may swap between boots.

### Solution

After flashing Mainline, use:

`/dev/serial/by-id/`

When stock MCUs expose the same generic serial, temporarily use:

`/dev/serial/by-path/`

to distinguish them physically before flashing.

## Original USB mainboard and toolboard appear swapped

### Possible symptoms

- nonexistent pins;
- wrong heater;
- wrong endstop;
- unreachable toolboard;
- apparently inconsistent errors.

### Check

In the historical configuration with the original USB toolboard:

- serial configured in `[mcu]`;
- serial configured in `[mcu extra_mcu]`;
- physical correspondence with each board.

With the current Sovol Zero, verify the CAN UUID and the state of the CAN toolhead MCU instead.

### Solution

Identify the two MCUs one at a time.

Do not rely on `ttyACM*` order.

## MCU does not start after flashing

### Check

- correct MCU target;
- STM32F103;
- crystal;
- USB pins;
- bootloader offset;
- Katapult;
- correct file for mainboard/toolboard.

### Phoenix verified

Phoenix used Katapult with:

- STM32F103;
- 8 MHz crystal;
- USB PA11/PA12;
- 8 KiB bootloader offset.

### What not to do

Do not erase the MCU again until firmware configuration has been verified.

Always preserve your personal original dump.

# Homing

## `Z axis must be homed before probing`

### Historical Phoenix context

This error appeared while using the old:

`probe_eddy_ng`

plugin on Klipper Mainline.

The path entered a circular dependency:

- Z homing required the probe;
- probing required Z to already be homed.

### Phoenix solution

The old plugin was abandoned.

Native Eddy support was used instead:

`[probe_eddy_current eddy]`

### What not to do

Do not permanently solve this with:

`set_position_z: 0`

That falsely declares Z homed.

Do not keep patching `probe_eddy_ng.py` when native Mainline support is available.

## Z movement before homing

Absolute or relative Z motion before Z is known can be dangerous.

### Solution

In the homing path, execute any Z lift only when:

`z` is already present in `printer.toolhead.homed_axes`.

# Eddy

## `Trigger analog error: RAW_RANGE`

### Check

- Eddy hardware actually installed;
- Eddy calibration;
- observed frequency;
- `max_sensor_hz`;
- I2C;
- power;
- wiring;
- any local changes to the LDC1612 driver.

### Current Phoenix — Sovol Zero

With the Eddy probe integrated into the Sovol Zero, the validated Phoenix configuration uses:

`max_sensor_hz: 7000000`

With the Mainline clock used on Phoenix, this gives LDC1612 divider 3.

During the Zero migration, calibration could complete with divider 2, but `G28 Z` produced:

`Trigger analog error: RAW_RANGE`

With divider 3 the problem disappeared.

The complete configuration is documented in:

[Sovol Zero toolhead, CAN and integrated Eddy](zero-toolhead-eddy-2026-08-17.md)

### Historical — previous Eddy NG

Before the Sovol Zero, the previous Phoenix Eddy NG configuration had shown frequencies up to about `8.523 MHz` and used:

`max_sensor_hz: 9000000`

That value belongs to the **old Eddy NG** and must not be transferred to the Zero.

### What not to do

Do not copy `7000000`, `9000000`, or any other value from a different configuration without verifying the hardware, driver, and frequency actually observed.

## Eddy unstable or noisy

### Check

- stable temperature;
- wiring;
- power;
- drive current;
- probe-bed distance;
- interference;
- frequency.

### Method

Compare multiple readings under the same conditions.

Do not diagnose thermal drift from a single measurement.

## `Tap not configured`

### Context

During the Phoenix migration, the installed DKEU version entered the TAP branch even without a real TAP calibration.

### Symptom

`Tap not configured`

### Historical cause

The logic checked for `tap_threshold` in a way that was incompatible with Mainline behavior at that time.

### Historical solution

A stricter condition was applied temporarily.

### Current state

That patch belongs to the historical DKEU phase and must not be applied automatically to other configurations.

Use:

- current Klipper Mainline;
- the DKEU version actually used, if analyzing a historical configuration or one independent from Phoenix;
- current TAP calibration if TAP is actually desired.

### What not to do

Do not import old values such as:

`tap_threshold: 300`

from legacy Eddy NG configurations.

# QGL

## QGL performs excessive corrections

### Check

- QGL points;
- gantry corners;
- `max_adjust`;
- probe coordinates;
- any Demon wrapper present in the historical configuration being analyzed;
- configuration actually loaded.

### Phoenix case

During a legacy stage, the Demon path could force behavior different from the native QGL defined in `printer.cfg`.

The solution was to return the Eddy path to the configured native QGL.

### Consolidated Phoenix configuration

- `horizontal_move_z: 3`
- `retry_tolerance: 0.05`
- `retries: 5`
- `max_adjust: 4`

### What not to do

Do not increase `max_adjust` dramatically just to make a problematic QGL finish.

A large limit can turn a configuration error into dangerous mechanical movement.

## QGL converges but the first layer is wrong

QGL measures the gantry relative to the bed.

It does not by itself guarantee:

- correct Z;
- correct mesh;
- correct compensations;
- correct macros.

### Verify after QGL

- probe;
- mesh;
- Z reference;
- any Demon path in the historical configuration;
- slicer.

# Mesh

## Mesh looks strangely flat but the first layer is strongly tilted

### Phoenix case

An old:

`axis_twist_compensation`

was found modifying probe results.

The compensation almost canceled a real difference of about:

`0.33 mm`

between bed sides.

The mesh looked artificially flat while the first layer was wrong.

### Solution

Neutralize the invalid legacy compensation and recreate the mesh.

### What not to do

Do not assume that an almost-flat mesh is automatically correct.

A mesh must represent the real surface.

## Mesh changes dramatically between similar tests

### Check

- bed temperature;
- soak;
- QGL;
- probe;
- residual compensations;
- any Demon wrapper present in the historical configuration being analyzed;
- `scan` / `rapid_scan` method;
- coordinates.

### Diagnosis

Compare tests performed under the same conditions.

Do not directly compare:

- cold bed;
- hot bed;
- different mesh geometries;
- different methods;

as if they were equivalent.

# Historical DKEU / Demon problems

The following sections describe problems actually encountered while Phoenix still used DKEU.

They are retained for diagnostic and historical value, but **DKEU is not part of the current Phoenix runtime baseline**.

For the current configuration, refer to Phoenix Macros.

## Emergency shutdown immediately at print start

### Check

- printer profile selected in OrcaSlicer;
- presence of `DEMON_START`;
- Machine G-code version;
- parameters required by DKEU;
- possible `_SPS GSTART=True`.

### Phoenix case

OrcaSlicer was still set to the Sovol machine instead of Phoenix.

The generated G-code did not contain the expected Demon Machine Start.

Demon therefore correctly performed an emergency shutdown.

### Solution

Select the correct printer preset and regenerate the G-code.

### What not to touch

Do not modify Eddy, QGL, or MCU firmware if the G-code does not even contain the start command required by Demon.

## `_PROBE_TAP` works but Z remains wrong

### Check

- `_APPLY_EDDY_Z_OFFSET` macro actually loaded;
- local overrides;
- `printer.probe.last_probe_position.z`;
- `SET_KINEMATIC_POSITION`;
- macro order.

### Phoenix case

An old override remained in `printer.cfg`:

`_APPLY_EDDY_Z_OFFSET`

which turned the macro into a no-op.

The probe measured contact correctly, but the Z coordinate was not realigned.

### Effect

The failed print showed a difference of about:

`0.061 mm`

which was enough to compromise a `0.20 mm` first layer.

### Solution

In the historical Phoenix configuration, the fix was to remove the legacy override and leave the DKEU macro expected at that time operational.

### What not to do

Do not correct this by changing:

- QGL;
- Z motors;
- screws;
- flow;
- mesh;

before verifying the macro that actually executed.

## Historical — DKEU behaved differently from manual tests

### Check

- wrapper macros;
- `_BASE` alias;
- effective mesh method;
- adaptive meshing;
- complete `DEMON_START` log.

### Method

Compare:

1. native Klipper command;
2. equivalent Demon macro;
3. complete start path.

This identifies the layer where behavior changes.

# OrcaSlicer

## Presets appear to have disappeared

### Check

- selected printer preset;
- user profile directory;
- versioned snapshot;
- OrcaSlicer log.

### Phoenix case

The presets were not lost.

They existed both in the Git snapshot and in the OrcaSlicer user directory.

The actual problem was the selected machine.

## Parameters look correct but the print behaves differently

### Check

Do not rely only on the UI.

Inspect real G-code for:

- temperatures;
- pressure advance;
- retraction;
- start macro;
- flow;
- overrides.

### Phoenix case

At different stages, different PA values existed in:

- `printer.cfg`;
- slicer profile;
- commands emitted in G-code.

The value actually executed is the one that matters.

# First layer

## First layer too squashed everywhere

### Check

- Z reference;
- Z homing;
- current Eddy probe;
- mesh actually active;
- any residual offset or babystep;
- Phoenix macros actually executed.

`_APPLY_EDDY_Z_OFFSET` belonged to the previous DKEU integration and is covered in the historical section, not in the current Phoenix Macros runtime.

### Do not start from

- flow ratio;
- pressure advance;
- Z motors.

## First layer too high everywhere

### Check

- Z reference;
- homing;
- probe;
- offset application;
- active mesh.

## First layer correct in some areas and badly wrong in others

### Check

- mesh;
- residual axis twist;
- QGL;
- bed temperature;
- contamination;
- real bed geometry.

### Phoenix case

A legacy axis twist compensation altered probe results and masked about `0.33 mm` of real difference.

After neutralizing it, the mesh began representing the bed correctly.

## Isolated local defect

If the rest of the bed is coherent, consider:

- fingerprints;
- grease;
- dust;
- contamination;
- local PEI defect.

Do not use a global correction for a local defect.

## Benchy bottom too squashed

A small model in the center can reveal a global Z problem but does not describe the complete bed geometry.

Use together:

- multi-zone test;
- mesh;
- real print.

# Historical suspicions not confirmed

## `_MESH_HANDLING`

During the Phoenix migration, a potentially questionable logical condition was observed in `_MESH_HANDLING`.

It was not demonstrated to be the cause of the failed print.

Do not apply historical patches without concrete reproduction on the configuration being analyzed.

## Detection of `probe_eddy_current eddy`

In a historical snapshot, one condition appeared to explicitly consider `probe_eddy_current btt_eddy` but not always `probe_eddy_current eddy`.

This also was not demonstrated as the real cause.

If DKEU is used independently from the Phoenix baseline, always verify the code of the version actually installed.

# Toolhead / wiring

## Toolhead wiring brushes the rear frame

### Check

- wiring of the toolhead actually installed;
- CAN cable / umbilical in the current Zero configuration;
- any chain in the historical configuration;
- PTFE;
- maximum Y position;
- free movement.

### Phoenix case

In the previous configuration with the original toolboard, the toolhead bundle could brush the rear frame and one chain link was removed to gain usable length.

With the Sovol Zero, verify the physical path of the current umbilical/CAN instead of automatically reusing geometric conclusions from the old toolhead.

### What not to do

Do not simply increase travel limits without physically checking the wiring.

# Quick recovery checklist

When the printer goes from "working" to "printing badly":

1. do not immediately modify mechanics;
2. save logs and active configuration;
3. check `git diff` or recent backups;
4. verify homing;
5. verify `PROBE`;
6. verify QGL;
7. verify mesh;
8. in a historical DKEU configuration, verify the macros actually loaded;
9. verify Orca presets;
10. verify real G-code;
11. reprint the same file after one correction only.

## If homing/probe/QGL/mesh are all coherent

Shift attention to:

- macros actually executed;
- Z reference/homing;
- slicer;
- start G-code.

Do not keep mechanically adjusting a machine that is already coherent in geometric tests.

## If mesh and first layer disagree

Verify:

- axis twist;
- probe transformations;
- offsets;
- wrapper macros;
- whether the intended mesh is actually active.

## If the problem appears after a software migration

Look first for:

- legacy overrides;
- duplicate macros;
- deprecated parameters;
- old plugins;
- copied stock configuration.

Apparent compatibility does not imply semantic compatibility.

# Confirmed Phoenix causes

Causes actually confirmed during the migration include:

- unsafe use of dynamic MCU identifiers;
- old Eddy NG plugin incompatible with the Mainline homing path;
- insufficient `max_sensor_hz` in the previous Eddy NG configuration;
- unsuitable LDC1612 divider/frequency during migration of the Eddy integrated into the Zero, resolved with the documented Zero configuration;
- incompatible TAP selection in the historical DKEU version;
- legacy axis twist compensation falsifying the mesh;
- wrong Orca printer profile;
- legacy `_APPLY_EDDY_Z_OFFSET` override preventing Z realignment;
- Ethernet route stealing Internet access from Wi-Fi.

These problems were observed and diagnosed with concrete evidence.

# Things not to do

During troubleshooting, avoid:

- using `ttyACM0/1` as permanent identifiers;
- using `set_position_z: 0` as a fake solution;
- patching old plugins when native Mainline support exists;
- copying legacy `tap_threshold` values;
- indiscriminately increasing `max_adjust`;
- chasing an artificially flat mesh;
- correcting Z through flow;
- correcting a macro problem through mechanics;
- applying historical workarounds to current software without verification.

# Escalation

If the problem remains unresolved, collect at least:

- Klipper version;
- Klipper commit;
- DKEU version, when analyzing an old Demon-based configuration;
- active configuration;
- `klippy.log`;
- `moonraker.log` where relevant;
- USB MCU identifiers and, where present, the toolhead CAN UUID;
- CAN bus state where relevant;
- homing output;
- probe output;
- QGL output;
- mesh;
- failed G-code;
- precise physical description of the symptom.

A reproducible diagnosis is worth more than many random adjustments.

---

## Navigation

- ← **Previous page:** [Validation and calibration](validation-and-calibration.md)
- → **Next page:** [README](../../README.en.md)
