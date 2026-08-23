# Base configuration — Sovol SV08 on Klipper Mainline

**Languages:** [Italiano](../base-configuration.md) | **English**

Last review: **2026-08-23**.

## Purpose

This stage builds a minimal, controllable Klipper Mainline configuration before integrating:

- Sovol Zero Extruder Kit and CAN connection;
- native Eddy integrated into the Zero;
- mesh;
- Phoenix Macros;
- slicer calibration;
- hardware modifications that are not required for the migration.

The goal is not to copy the old Sovol `printer.cfg` wholesale.

The goal is to recover only the hardware parameters that are actually required and verify them one at a time.

## Prerequisites

The following must already be complete:

- [Backup and rollback](backup-and-rollback.md)
- [CB1 and Klipper Mainline installation](install-cb1-mainline.md)
- [MCU flashing and recovery](flash-mcus.md)

During this stock-migration phase, the mainboard and original toolboard must both be reachable from Klipper Mainline.

The Sovol Zero CAN toolhead is introduced in the next phase.

## Fundamental rule

A stock configuration that works on Sovol Klipper is not automatically a valid Mainline configuration.

Always separate:

- hardware parameters;
- macros;
- legacy workarounds;
- probe configuration;
- slicer configuration.

Hardware parameters can often be recovered.

Macros and workarounds must be reassessed.

## MCU

Use stable identifiers:

`/dev/serial/by-id/`

Avoid permanent configuration based on:

- `/dev/ttyACM0`
- `/dev/ttyACM1`

During the initial stock-hardware phase verify that:

- `[mcu]` corresponds to the mainboard;
- any second USB MCU really corresponds to the original toolboard.

Do not infer MCU identity from USB enumeration order.

The current Phoenix configuration instead uses the **Sovol Zero Extruder Kit over CAN**, with the toolhead MCU identified by CAN UUID.

## Stepper parameters

Recover the necessary motor parameters from the stock configuration, including:

- step pin;
- direction pin;
- enable pin;
- microsteps;
- rotation distance;
- gear ratio;
- endstop pin;
- endstop position;
- position min;
- position max;
- homing speed;
- homing direction.

Do not modify several mechanical parameters at the same time without a verified reason.

## Phoenix motor parameters that must not be changed casually

During the Phoenix migration, these were kept as protected mechanical values:

- `rotation_distance: 40` for the relevant axes;
- `gear_ratio: 80:12` where applicable;
- `microsteps: 16`.

These values belong to the verified Phoenix configuration.

Before using them on another SV08, compare them with that machine's stock configuration.

## Extruder

Recover from the stock configuration and the hardware actually installed:

- step pin;
- dir pin;
- enable pin;
- microsteps;
- rotation distance;
- full steps per rotation;
- nozzle diameter;
- filament diameter;
- heater pin;
- sensor type;
- sensor pin;
- min temp;
- max temp;
- pressure advance only as a provisional value.

If the hotend or extruder has been modified, do not treat Phoenix values as a universal baseline.

## Historical Phoenix — extruder before the Sovol Zero

During an earlier Phoenix calibration phase the configuration included:

- `rotation_distance: 6.5`
- `microsteps: 16`
- `full_steps_per_rotation: 200`
- `nozzle_diameter: 0.400`
- `filament_diameter: 1.75`
- `pressure_advance: 0.025`
- `pressure_advance_smooth_time: 0.035`

These values describe an earlier Phoenix hardware phase, before the current Sovol Zero baseline, and must not be copied automatically.

Final material calibration is covered separately.

## Historical Phoenix — firmware retraction

Phoenix also had:

- `retract_length: 0.8`
- `retract_speed: 30`
- `unretract_extra_length: 0.0`
- `unretract_speed: 30`

During the analyzed tests, the G-code did not use `G10`, `G11`, or `SET_RETRACTION`.

Therefore those values were not responsible for the behavior observed in those prints.

## Heaters and sensors

Before any heating, verify with the machine cold that:

- hotend temperature is plausible;
- bed temperature is plausible;
- any additional sensors are plausible.

An out-of-range temperature must be fixed before activating a heater.

Do not use heating as the first test of an unverified sensor.

## PID

PID values from the stock configuration may be retained only as an initial reference.

After:

- changing the hotend;
- changing the heater;
- changing the bed;
- significantly changing thermal mass;

run a new PID calibration.

Phoenix PID values are not a universal baseline.

## Endstops

Verify endstops before homing.

Check:

- idle state;
- triggered state;
- logical consistency;
- correct pins.

Do not use `G28` as the first test of an unverified endstop.

## First movement

The first movement should happen only after verifying:

- MCUs;
- temperatures;
- endstops;
- stepper configuration;
- machine limits.

Move one axis at a time using small distances.

Verify direction and travel distance.

If direction is wrong, correct the configuration before continuing.

## Do not use final Z homing yet

At this stage, do not define the final Z homing behavior.

Phoenix first used the old Eddy NG and later moved to the **Eddy integrated into the Sovol Zero** through native Mainline support.

The current Zero + Eddy path is documented in:

[Sovol Zero toolhead, CAN and integrated Eddy](zero-toolhead-eddy-2026-08-17.md)

Avoid temporary workarounds such as:

`set_position_z: 0`

which falsely declares Z homed.

## QGL

The SV08 uses four Z motors and `QUAD_GANTRY_LEVEL`.

QGL configuration must be verified against the real geometry of your machine.

Do not modify:

- gantry corners;
- probe points;
- max adjust;
- retry tolerance;

just to make QGL "pass".

A QGL run that requires excessive correction may indicate a mechanical problem or incorrect configuration.

## Historical Phoenix — QGL before the Sovol Zero

Before the final Sovol Zero installation, the Phoenix configuration used:

- gantry corners: `(-60,-10)` and `(410,420)`
- points:
  - `(36,10)`
  - `(36,320)`
  - `(346,320)`
  - `(346,10)`
- speed: `400`
- horizontal move Z: `3`
- retry tolerance: `0.05`
- retries: `5`
- max adjust: `4`

`max_adjust` had been reduced from `30` to `4` as a safety measure.

With the previous native Eddy configuration and a heated machine, this setup had completed QGL with:

- retries: `3/5`
- probed points range: about `0.018 mm`
- tolerance: `0.050 mm`

These values document that specific Phoenix phase and must not be interpreted as a QGL result for the current Zero configuration or as a universal requirement.

## X and Y homing

After verifying endstops, directions, and limits, test X and Y homing separately.

Check that:

- the axis moves in the expected direction;
- the endstop is reached without impact;
- motion stops correctly;
- the final position is coherent with machine geometry.

If X or Y moves in the wrong direction, fix the configuration before continuing.

Do not compensate for a direction error by changing machine limits or coordinates.

## Machine limits

Verify:

- `position_min`;
- `position_max`;
- endstop positions;
- actually reachable area;
- any margins required by probe, toolhead, or accessories.

Limits must describe the real physical machine.

Do not increase `position_max` just to reach a coordinate used by a macro.

Macros must respect real machine limits, not the other way around.

## Machine center and service positions

Phoenix uses this operational center reference:

- X `191`
- Y `165`

This point was used during homing and verification.

It is a verified Phoenix reference, not a universal SV08 geometry definition.

Any park, nozzle-cleaning, purge, homing, or probe-calibration positions must be checked against your own toolhead and offsets.

## Toolhead and clearances

Before using coordinates close to the edges, physically verify:

- toolhead dimensions;
- probe offset;
- cables;
- PTFE;
- any umbilical;
- nozzle cleaner;
- installed accessories;
- enclosure.

A coordinate that was valid in the stock configuration can become unsafe after a hardware modification.

## Fans

Verify each Klipper-controlled fan separately.

Identify at least:

- hotend fan;
- part cooling fan;
- electronics fan;
- any additional fans.

Check:

- pin;
- logical polarity;
- automatic behavior;
- useful minimum speed.

A misconfigured hotend fan can damage the thermal system even if Klipper does not immediately report an error.

## Heater safety

Before PID or printing, verify:

- the correct heater is associated with the correct sensor;
- initial temperature is plausible;
- temperature rises coherently with the heater that was activated;
- no unrelated sensor rises by mistake.

Perform initial thermal tests in a controlled way.

If heating the bed raises the hotend reading, or vice versa, stop and correct the configuration.

## Stock macros

Do not automatically import yet:

- start print;
- end print;
- clean nozzle;
- purge;
- heat soak;
- QGL wrapper;
- mesh wrapper;
- custom pause/resume;
- Eddy macros;
- old DKEU/Demon macros.

These may depend on:

- old Sovol Klipper;
- old APIs;
- previous MCU names;
- the old probe;
- Phoenix-specific coordinates;
- workarounds that are no longer required.

Bring them back only after the basic hardware configuration is stable.

## Axis twist

Do not automatically import an old `axis_twist_compensation`.

A compensation saved before a probe change, toolhead modification, firmware migration, kinematic change, or new Z calibration may no longer be valid.

On Phoenix the old compensation was neutralized during Mainline recovery and was not treated as something to restore by default.

## Mesh

Do not create a final bed mesh yet.

Mesh depends on:

- probe;
- offsets;
- Z homing;
- bed temperature;
- nozzle temperature;
- QGL;
- probing method.

Mesh configuration is handled after native Eddy integration.

## What must work before Zero and Eddy

Before installing the Zero and configuring its Eddy probe, verify:

- mainboard and original toolboard from the stock phase;
- temperatures;
- heaters;
- fans;
- X endstop;
- Y endstop;
- X movement;
- Y movement;
- controlled Z movement;
- machine limits;
- base geometry;
- QGL configured but not necessarily executed;
- no remaining Klipper errors related to base hardware.

## What is not final yet

At this point, these may still change:

- Z homing;
- Z offset;
- probe offset;
- mesh;
- rapid scan;
- tap;
- QGL workflow;
- start print;
- Phoenix Macros;
- purge;
- nozzle cleaning;
- material calibration.

That is normal.

The base configuration proves that the machine can be controlled safely before automation and compensation are added.

## Historical Phoenix — state before the final Zero and Eddy migration

Before the final Sovol Zero installation, Phoenix had already verified:

- Mainline MCUs;
- mainboard/original-toolboard communication;
- axes;
- temperatures;
- endstops;
- mechanical parameters;
- base QGL;
- stable Mainsail and SSH access.

The remaining problems were in the probe/Z-homing path, not the fundamental hardware configuration.

That distinction prevented random changes to motors, gantry, or kinematics during Eddy debugging.

## Exit criteria

Before moving to the Sovol Zero and its Eddy probe, all of the following must be true:

- working configuration without base hardware errors;
- mainboard correctly identified;
- original toolboard correctly identified;
- plausible temperatures;
- heaters verified;
- fans verified;
- X and Y homing working;
- stepper directions correct;
- machine limits coherent;
- no legacy macro required just to control the machine;
- backup still available.

---

## Navigation

- ← **Previous page:** [MCU flashing and recovery](flash-mcus.md)
- → **Next page:** [Sovol Zero, CAN and integrated Eddy](zero-toolhead-eddy-2026-08-17.md)
