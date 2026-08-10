# Validation and calibration — Sovol SV08 Mainline

**Languages:** [Italiano](../validation-and-calibration.md) | **English**

Last review: **2026-08-10**.

## Purpose

This guide defines the validation order after migrating a Sovol SV08 to:

- Klipper Mainline;
- Mainline MCU firmware;
- native Eddy;
- Demon/DKEU.

The goal is to avoid calibrating filament while the machine is not yet geometrically stable, or mechanically correcting problems that actually belong to macros, Z, or the slicer.

## General order

Recommended sequence:

1. hardware sanity check;
2. PID;
3. homing;
4. QGL;
5. probing;
6. bed mesh;
7. first layer;
8. material temperature;
9. flow ratio;
10. pressure advance;
11. retraction;
12. max volumetric speed;
13. real validation print.

Do not change this order without a specific reason.

## Three calibration families

Always distinguish between:

### Machine calibration

Depends mainly on printer hardware.

Includes:

- PID;
- stepper directions;
- endstops;
- limits;
- kinematics;
- heaters;
- fans.

### Bed/Z calibration

Depends on the mechanical system and bed.

Includes:

- QGL;
- probe;
- Z reference;
- mesh;
- first layer.

### Filament calibration

Depends mainly on:

- material;
- hotend;
- nozzle;
- temperature;
- speed.

Includes:

- temperature;
- flow ratio;
- pressure advance;
- retraction;
- max volumetric speed.

## Fundamental rule

Do not use material calibration to hide a machine error.

Examples:

- do not increase flow to compensate for a nozzle that is too high;
- do not reduce flow to compensate for a nozzle that is too low;
- do not change pressure advance to correct a bad mesh;
- do not change Z to compensate for an extrusion-multiplier problem.

Stabilize the machine and bed first.

Then calibrate the material.

## Initial sanity check

Before any calibration, verify:

- no Klipper errors;
- mainboard connected;
- toolboard connected;
- plausible temperatures;
- working fans;
- correct heaters;
- working X/Y/Z homing;
- working Eddy;
- no interfering legacy macros;
- no mechanical collisions.

## PID

PID should be recalibrated after modifications that significantly change thermal behavior.

For the hotend this includes:

- hotend replacement;
- heater replacement;
- thermistor replacement;
- major thermal-block changes.

For the bed this includes:

- bed replacement;
- heater replacement;
- significant thermal-mass changes;
- major insulation changes.

## Hotend PID — Phoenix

Phoenix uses a MicroSwiss FlowTech.

During the Mainline migration, hotend PID values at `220 °C` were saved as:

- `pid_Kp: 26.772`
- `pid_Ki: 2.052`
- `pid_Kd: 87.343`

These values belong to that specific Phoenix stage and configuration.

Do not copy them to another machine.

Always run your own PID calibration.

## Bed PID — Phoenix

During the same Phoenix stage, saved values were:

- `pid_Kp: 74.106`
- `pid_Ki: 1.254`
- `pid_Kd: 1094.910`

These are also hardware-specific.

Do not use them as community presets.

## Bed replacement

Changing the bed can alter:

- thermal mass;
- temperature distribution;
- PID response;
- geometry;
- Z position;
- mesh;
- first-layer behavior.

After a major bed replacement, repeat at least:

- bed PID;
- QGL;
- probe/Z reference;
- mesh;
- first layer.

You do not automatically need to reset every filament calibration.

## Homing

After PID, verify again:

`G28`

Check:

- X;
- Y;
- Z;
- final position;
- absence of impacts;
- nozzle-bed clearance.

Do not proceed if homing is not repeatable.

## QGL

Run `QUAD_GANTRY_LEVEL` only after homing and probing are reliable.

QGL should:

- converge;
- not require extreme corrections;
- produce repeatable results.

QGL must not be used to correct an incorrect Z configuration.

## Phoenix verified — QGL

On Phoenix, with native Eddy and the consolidated configuration, a hot test produced:

- retries: `3/5`
- probed range: `0.018441`
- tolerance: `0.050000`

Configuration:

- `horizontal_move_z: 3`
- `retry_tolerance: 0.05`
- `retries: 5`
- `max_adjust: 4`

The result did not justify changes to the gantry or Z motors.

## Probe validation

After QGL, verify the probe separately.

Check:

- coherent values;
- no RAW_RANGE error;
- no unintended contact;
- coherent cold and hot behavior.

Do not assume a successful mesh automatically proves that the complete Z path is correct.

## Mesh

Only after stable homing, QGL, and probing should you create a mesh.

For Mainline Eddy, `rapid_scan` can be used once the configuration has already been validated.

During initial debugging it is also useful to verify native behavior without Demon wrappers.

## Phoenix verified — mesh

The hot 15 x 15 Phoenix mesh using:

`BED_MESH_CALIBRATE METHOD=rapid_scan`

produced approximately:

- max: `+0.163 mm`
- min: `-0.178 mm`
- range: `0.341 mm`

An earlier session had shown approximately:

- max: `+0.194 mm`
- min: `-0.160 mm`
- range: `0.354 mm`

The consistency between the two measurements indicated that the mesh represented real bed geometry rather than random noise.

## Do not chase a flat mesh

A mesh does not have to be nearly zero.

Its purpose is to describe the real surface.

An artificially flat mesh can be more dangerous than a larger-range mesh that is physically correct.

During Phoenix history, an old compensation almost erased a real difference of about `0.33 mm`, producing a bad first layer.

## First layer

Validate the first layer only after:

- homing;
- QGL;
- probe;
- mesh.

Use a test geometry large enough to reveal differences across the bed.

A small central square is not enough to judge a 350 mm SV08 bed.

## Multi-zone test

During Phoenix recovery, a first-layer test distributed across several bed zones was used.

This helps distinguish:

- global Z error;
- bad mesh;
- local bed defect;
- contamination;
- adhesion variation.

## First-layer diagnosis

If the layer is too squashed everywhere:

verify Z reference first.

If it is too high everywhere:

verify Z reference first.

If it varies systematically from one side to the other, verify:

- mesh;
- QGL;
- probe;
- residual compensations.

If the defect is isolated, also consider:

- dirt;
- contamination;
- local bed defects.

## Do not confuse Z and flow

An overfilled first layer does not automatically mean flow is too high.

A sparse first layer does not automatically mean flow is too low.

Flow ratio must be calibrated only after proving the Z/bed system is correct.

## Filament calibration

Only when:

- machine;
- Z;
- QGL;
- probe;
- mesh;
- first layer

are already reliable should filament calibration begin.

Recommended sequence:

1. temperature;
2. flow ratio;
3. pressure advance;
4. retraction;
5. max volumetric speed.

## Temperature

Calibrate temperature before flow.

Temperature affects:

- viscosity;
- interlayer adhesion;
- bridging;
- stringing;
- surface quality;
- maximum throughput.

Do not calibrate flow or pressure advance while temperature is still uncertain.

## Phoenix verified — PolyTerra PLA

For the profile:

`Polymaker PolyTerra PLA @Phoenix 0.4`

the consolidated calibration temperature was:

`200 °C`

This was validated for:

- Phoenix;
- MicroSwiss FlowTech;
- 0.4 nozzle;
- that material.

It is not a universal value for every PLA.

## Flow ratio

Flow ratio corrects the actual amount of material extruded during printing.

Do not use it to compensate for:

- incorrect Z;
- nozzle too close;
- nozzle too far;
- bad mesh;
- uncalibrated first layer.

## Phoenix verified — Flow ratio

PolyTerra calibration produced:

`Flow ratio = 1.0465`

The value comes from OrcaSlicer calibration performed on Phoenix.

Do not copy it directly to:

- another hotend;
- another nozzle;
- another material;
- another spool without verification.

## Pressure Advance

Calibrate Pressure Advance after flow.

It compensates for dynamic pressure behavior in the extrusion system.

Incorrect values can cause:

- swollen corners;
- hollowed corners;
- thickness variation during acceleration;
- artifacts during speed transitions.

## Phoenix verified — Pressure Advance

The best area observed in PolyTerra calibration was about:

`0.034`

Consolidated value:

`Pressure Advance = 0.034`

Earlier stages had used different values, including:

- `0.025` in `printer.cfg`;
- `0.032` in slicer profiles/tests.

This demonstrates why you must always verify which source is actually setting PA during printing.

## Who controls Pressure Advance

PA can come from:

- `printer.cfg`;
- Demon;
- OrcaSlicer;
- start G-code;
- macros;
- filament profile.

Before calibrating, verify who is actually setting it.

Two individually correct values in two different places can still create an inconsistent workflow.

## Retraction

Calibrate retraction after:

- temperature;
- flow;
- pressure advance.

Increase retraction only if there is a real stringing or ooze problem.

Do not increase it automatically.

Excessive values can cause:

- delayed extrusion restart;
- gaps;
- pressure variation;
- heat creep;
- clogs in some hotends.

## Firmware retraction

Phoenix also had a configuration with:

- `retract_length: 0.8`
- `retract_speed: 30`
- `unretract_extra_length: 0.0`
- `unretract_speed: 30`

but the G-code analyzed at that stage did not use:

- `G10`;
- `G11`;
- `SET_RETRACTION`.

Therefore that configuration was not necessarily controlling actual print retraction.

## Slicer retraction

Different slicer values appeared during Phoenix tests.

That makes verification of the real G-code mandatory.

It is not enough to read:

- `printer.cfg`;
- metadata;
- presets.

When necessary, inspect E moves and retraction commands directly in generated G-code.

## Phoenix verified — Retraction

During final PolyTerra calibration:

- the towers were essentially clean;
- there was no significant stringing;
- no further increase in retraction was required.

The current retraction was therefore kept as validated.

The exact value is not promoted as a community preset because it depends on the path actually used by the slicer profile.

## Max Volumetric Speed

Max Volumetric Speed defines the maximum material throughput requested from the hotend.

Calibrate it after:

- temperature;
- flow;
- PA;
- retraction.

If set too high it can cause:

- under-extrusion;
- quality loss;
- weak walls;
- inconsistency at high speed.

## Phoenix verified — Max Volumetric Speed

The PolyTerra profile initially used:

`22 mm³/s`

After calibration, the consolidated value became:

`24 mm³/s`

This is the validated operational limit for that Phoenix combination.

It does not represent the universal capability of the FlowTech or PolyTerra PLA.

## Consolidated Phoenix profile

Final validated values:

- nozzle calibration: `200 °C`
- flow ratio: `1.0465`
- pressure advance: `0.034`
- retraction: current value, validated by test
- max volumetric speed: `24 mm³/s`

These values belong to the combination:

- Phoenix;
- MicroSwiss FlowTech;
- 0.4 nozzle;
- PolyTerra PLA;
- specific OrcaSlicer profile.

## Bed replacement and filament calibration

Changing the bed does not automatically require redoing from zero:

- flow;
- PA;
- retraction;
- MVS.

After a major bed change, revalidate first:

- bed PID;
- QGL;
- Z reference;
- mesh;
- first layer.

Only reopen filament calibration if a later print shows a real problem.

## Hotend or nozzle replacement

A hotend or nozzle change may require broader revalidation.

In particular:

- hotend PID;
- material temperature;
- flow;
- PA;
- retraction;
- MVS.

A different nozzle diameter directly changes required throughput and extrusion behavior.

## Real validation print

Calibration towers are not enough by themselves.

After consolidating values, run a known real print.

Check:

- first layer;
- walls;
- top surface;
- corners;
- stringing;
- bridging;
- small details;
- consistency at high speed.

## A/B comparison

When correcting a single problem, reprint the same G-code when possible.

This allows a direct comparison of:

- before;
- after.

On Phoenix, this method clearly separated the effect of the `_APPLY_EDDY_Z_OFFSET` correction from slicer or material changes.

## Freeze validated calibration

Once a calibration has been validated:

- document it;
- save it;
- version it;
- do not change it without a cause.

A stable configuration is more useful than one that is constantly adjusted.

## What to repeat after hardware changes

### Bed change

Repeat at least:

- bed PID;
- QGL;
- Z reference;
- mesh;
- first layer.

### Hotend change

Repeat at least:

- hotend PID;
- material temperature;
- flow;
- PA;
- retraction;
- MVS.

### Nozzle change

Revalidate at least:

- temperature;
- flow;
- PA;
- retraction;
- MVS.

### Probe/toolhead change

Repeat at least:

- X/Y offsets;
- Z homing;
- Eddy calibration;
- QGL;
- mesh;
- first layer.

## Exit criteria

The machine can be considered calibrated when:

- PID is stable;
- homing is repeatable;
- QGL converges;
- probe is coherent;
- mesh is coherent;
- first layer is uniform;
- material temperature is defined;
- flow is defined;
- PA is defined;
- retraction is verified;
- MVS is defined;
- a real print completes without obvious systematic defects.

## Next step

With migration and calibration complete:

`docs/en/troubleshooting.md`
