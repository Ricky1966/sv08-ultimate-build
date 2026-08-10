# Native Eddy — Sovol Eddy NG hardware on Klipper Mainline

**Languages:** [Italiano](../native-eddy.md) | **English**

Last source review: **2026-08-10**.

## Purpose

This guide documents the migration of Sovol Eddy NG / LDC1612 hardware from the old Eddy-NG-specific plugin path to the native Eddy support available in Klipper Mainline.

Phoenix was actually migrated all the way to:

- working Z homing;
- static probing;
- QGL;
- rapid bed mesh;
- Demon integration;
- a verified first print.

## Prerequisites

The following must already be complete:

- `docs/en/backup-and-rollback.md`
- `docs/en/install-cb1-mainline.md`
- `docs/en/flash-mcus.md`
- `docs/en/base-configuration.md`

The MCUs, axes, temperatures, and X/Y endstops must already work.

## Hardware

Phoenix uses:

- Sovol Eddy NG sensor;
- LDC1612 controller;
- I2C connection through the toolboard;
- toolboard MCU identified as `extra_mcu`.

The hardware can continue to be used without retaining the old `probe_eddy_ng.py` plugin.

## Why the old plugin was abandoned

During the Phoenix migration, the old `probe_eddy_ng` path was initially ported to Klipper Mainline.

The main problem appeared in the Z-homing path.

The plugin could enter a circular dependency:

- `G28 Z` required the probe;
- the probe path required Z to already be homed.

The observed symptom was:

`Z axis must be homed before probing`

Temporary workarounds were tested, but the decision was made not to keep patching `probe_eddy_ng.py`.

The recommended community path is therefore Klipper Mainline's native Eddy support.

## Do not use fake Z homing

During debugging, this was temporarily used:

`set_position_z: 0`

It artificially declares Z homed without physically establishing the reference.

It was removed.

Do not use `set_position_z: 0` as a permanent fix for Eddy homing problems.

## Native Eddy section

The verified Phoenix configuration uses:

`[probe_eddy_current eddy]`

Parameters:

- `sensor_type: ldc1612`
- `i2c_mcu: extra_mcu`
- I2C clock on `PB6`
- I2C data on `PB7`
- `x_offset: -16.43`
- `y_offset: 10.22`
- `descend_z: 0.5`
- `max_sensor_hz: 9000000`

Calibration also saved:

`reg_drive_current: 22`

X/Y offsets belong to the Phoenix toolhead and must be verified on your own machine.

## Note on I2C syntax

Historical migration states included configuration forms with explicit software pins associated with `extra_mcu`.

Current Mainline configuration must follow the syntax supported by the Klipper version actually installed.

Do not copy old syntax only because it appears in Phoenix history.

## `max_sensor_hz`

The first native Z-homing attempt produced:

`Trigger analog error: RAW_RANGE`

Calibration showed frequencies up to about:

`8.523 MHz`

The initial native limit was too low for the Phoenix sensor.

It was therefore set to:

`max_sensor_hz: 9000000`

After this change, `G28 Z` completed without nozzle contact with the bed.

This value is verified on Phoenix.

Before using it elsewhere, check the actual frequency observed from your sensor.

## Drive current

Native calibration on the Phoenix sensor produced:

`reg_drive_current: 22`

The value was saved with `SAVE_CONFIG`.

Do not assume `22` is correct for every Eddy sensor.

Run the calibration required by your Klipper version.

## Eddy current calibration

Phoenix used:

`PROBE_EDDY_CURRENT_CALIBRATE CHIP=eddy`

The resulting curve covered approximately:

- Z: `0.050625 -> 2.090625 mm`
- frequency: `8.523 -> 8.451 MHz`

Calibration must be performed on your own machine.

Do not copy another printer's calibrated curve.

## `descend_z`

The Phoenix configuration keeps:

`descend_z: 0.5`

This parameter belongs to current native probing behavior.

Do not confuse it with old `z_offset` parameters used by previous implementations.

## Homing override

During migration, old calls to:

`PROBE_EDDY_NG_PROBE_STATIC HOME_Z=1`

were removed.

The Z path must use only the configured native probe.

The Phoenix full-homing path included an initial Z lift.

It was made safe by executing it only when Z was already homed.

Conceptually:

`if Z is already homed -> raise Z before XY movement`

This prevents issuing relative Z motion when the Z position is still unknown.

## Verified full homing

After removing legacy workarounds and configuring native Eddy, a complete `G28` starting with all axes unhomed completed successfully.

Observed final Phoenix position:

- X `191`
- Y `165`
- Z `10`

These coordinates belong to the Phoenix workflow.

## Cold validation

Before QGL and mesh, the following were verified:

- full `G28`;
- center `PROBE`;
- Z realignment from the probe;
- probing used by the Demon workflow;
- no unintended nozzle-bed contact.

Center `PROBE` values observed cold:

- `0.002734 mm`
- `0.006694 mm`

These values were coherent enough to proceed.

## Caution after `PROBE`

A simple `PROBE` leaves the toolhead near the trigger height.

Before another probing operation:

- raise Z;
- verify that the nozzle has safe clearance from the bed.

Do not repeat probing while assuming the toolhead automatically returned to a safe height.

## Hot validation

Phoenix test conditions:

- bed `65 °C`
- nozzle `170 °C`
- soak about `10 minutes`

Full homing completed without problems.

Observed center `PROBE`:

`0.007357 mm`

The difference from the cold readings was on the order of a few microns under those test conditions.

This does not universally prove the absence of thermal drift.

Each machine must be verified under its own operating conditions.

## QGL with native Eddy

After validating homing and probing, Phoenix ran `QUAD_GANTRY_LEVEL` without changing the consolidated QGL configuration.

Observed result:

- retries: `3/5`
- probed points range: `0.018441`
- tolerance: `0.050000`

QGL configuration remained:

- `horizontal_move_z: 3`
- `retry_tolerance: 0.05`
- `retries: 5`
- `max_adjust: 4`

The result did not indicate a need to modify:

- gantry;
- Z motors;
- screws;
- kinematics.

## Rapid bed mesh

After QGL, a mesh was run with:

`BED_MESH_CALIBRATE METHOD=rapid_scan`

The hot 15 x 15 Phoenix mesh produced approximately:

- max: `+0.163 mm`
- min: `-0.178 mm`
- range: `0.341 mm`

This was consistent with the geometry already observed on the original bed.

No isolated spikes were seen that would clearly indicate a probing error.

The test mesh was not made permanent during that stage.

## Intermediate Demon state — do NOT reproduce

During an intermediate migration stage, a local override was still present:

`[gcode_macro _APPLY_EDDY_Z_OFFSET]`

which turned the macro into a no-op.

It had been created for the old Sovol Klipper 0.12, where the field used by Demon was unavailable.

At that stage it was intentionally retained as a legacy workaround.

This state does not represent the correct final Mainline configuration.

## Root cause of the bad first layer

The following day, analysis of the failed first layer proved that this legacy override prevented Demon from correctly realigning the Z coordinate after `_PROBE_TAP`.

In the failed print the probe had estimated:

`z=-0.060846`

Failing to apply about `0.061 mm` of correction was enough to severely damage a `0.20 mm` first layer.

The primary cause was not:

- QGL;
- mesh;
- Z motors;
- mechanics;
- general under-extrusion.

It was the legacy override still present in `printer.cfg`.

## Final correction

The local `_APPLY_EDDY_Z_OFFSET` override was removed.

The correct Demon macro was then left active, using:

`printer.probe.last_probe_position.z`

to realign Z through:

`SET_KINEMATIC_POSITION`

The final verified path is:

`_PROBE_TAP -> _APPLY_EDDY_Z_OFFSET -> printer.probe.last_probe_position.z -> SET_KINEMATIC_POSITION`

After this correction, reprinting the same G-code radically improved the first layer.

## Manual Z-realignment test

After removing the legacy override, the following were run:

`G28`

and:

`_PROBE_TAP`

The system reported:

`Setting position to 0.457101`

and a contact estimate:

`z=0.065696`

This confirmed that Mainline + native Eddy + Demon was again applying Z realignment.

## Practical rule

On current Klipper Mainline:

do not retain overrides created only to compensate for limitations of old Sovol Klipper.

In particular, inspect and remove local blocks that redefine:

- `_APPLY_EDDY_Z_OFFSET`
- `_PROBE_TAP`
- Z homing
- legacy static probing
- old Eddy NG macros

before deciding the problem is mechanical.

## TAP

Native Mainline Eddy supports TAP with its own semantics.

During the Phoenix migration, a Demon logic issue was observed where the TAP branch was selected merely because the `tap_threshold` key existed with its default value `0.0`.

The result was:

`Tap not configured`

At that historical stage, the workaround was to treat TAP as active only when the configured value was greater than zero.

That behavior belongs to the historical Demon integration.

For new installations, use current DKEU logic and documentation.

Do not automatically add old values such as:

`tap_threshold: 300`

Current native TAP semantics are not the same as the old Eddy NG plugin.

## RAW_RANGE

If homing or calibration reports:

`Trigger analog error: RAW_RANGE`

check first:

- actual sensor frequency;
- `max_sensor_hz`;
- current calibration;
- I2C wiring;
- sensor power.

On Phoenix the problem was solved by increasing:

`max_sensor_hz`

to:

`9000000`

because the measured curve exceeded 8.5 MHz.

Do not treat `9000000` as a universal value.

## Problems that should NOT immediately trigger mechanical changes

If:

- homing works;
- `PROBE` is coherent;
- QGL converges;
- mesh is reasonable;

but the first layer is still wrong, verify first:

- legacy macros;
- Z realignment;
- local overrides;
- slicer start G-code;
- Demon;
- probe state;

before touching:

- the four Z drives;
- gantry;
- screws;
- axis twist;
- mechanical geometry.

This principle avoided unnecessary mechanical changes during Phoenix recovery.

## Slicer and a false Eddy diagnosis

During the first printing attempt, another error completely independent of the probe was also found:

OrcaSlicer was still set to the Sovol machine instead of Phoenix.

The G-code therefore did not contain the expected Demon Machine Start and Demon performed an emergency shutdown.

This shows that not every error appearing during the Eddy stage is caused by Eddy.

Always verify:

- active printer preset;
- start G-code;
- filament profile;
- process profile.

## Exit criteria

Before considering Eddy complete, all of the following must work:

- full homing;
- real Z homing;
- `PROBE`;
- current calibration;
- QGL;
- rapid scan;
- no RAW_RANGE;
- no old `PROBE_EDDY_NG_*` command still required;
- no `set_position_z: 0`;
- correct Z realignment after probing;
- coherent first layer in a real test.

## Next step

Integrate Demon into the already stable Mainline system:

`docs/en/demon-integration.md`
