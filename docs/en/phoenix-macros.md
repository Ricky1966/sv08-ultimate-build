# Phoenix Macros

**Languages:** [Italiano](../phoenix-macros.md) | **English**

## Status

Phoenix Macros is the active macro layer of the Sovol SV08 “Phoenix”.

The current baseline is:

- Klipper Mainline;
- Sovol Zero Extruder Kit over CAN;
- Eddy integrated through native Klipper support;
- Phoenix Macros;
- external components explicitly attributed where they are still used.

**DKEU is no longer a runtime dependency of the Phoenix configuration.**

On August 22, 2026, the structural audit of the active runtime on the machine was completed; functional validation was completed on August 23, 2026:

- no active DKEU include in `printer.cfg`;
- no operational DKEU reference in `phoenix-*.cfg` files;
- no `save_variables` use by Phoenix Macros;
- no `force_move` use;
- no `M84` use;
- 21 `PHOENIX_*` macros defined;
- all 21 macros correctly exposed by Klipper;
- Klipper restarted without configuration, Jinja, or runtime errors during structural pack validation.

This check confirms the **structural/runtime independence** of Phoenix Macros from DKEU. Physical validation of individual macros is performed separately on the machine and must not be confused with successful loading by Klipper alone.

## Principles

Phoenix Macros is designed to keep only behavior that is actually needed and validated on Phoenix, avoiding frameworks, wrappers, and dependencies that are no longer useful.

Principles:

- use native Klipper Mainline functions directly when available;
- keep Phoenix macros small, readable, and verifiable;
- clearly separate Phoenix-owned macros from external components;
- avoid unnecessary overrides of Klipper commands;
- do not use `M84`;
- do not hide hardware behavior inside generic frameworks;
- keep the print sequence aligned with tests actually performed on the machine;
- always distinguish structural validation from physical testing.

## Active Phoenix files on the machine

The current configuration includes, in this order:

```text
phoenix-print-start-end.cfg
phoenix-cleaner.cfg
phoenix-runout.cfg
phoenix-idle.cfg
phoenix-filament.cfg
phoenix-core.cfg
phoenix-calibration.cfg
phoenix-debug.cfg
phoenix-setup.cfg
```

The packs added during the final separation from DKEU are organized by responsibility:

- **Core** — base operational functions and compatibility;
- **Calibration** — explicit calibration tools;
- **Debug** — diagnostics and tests;
- **Setup** — manual configuration/verification helpers.

## Current Phoenix macros

### Print workflow

- `PHOENIX_START`
- `PHOENIX_END`
- `PHOENIX_CLEAN_NOZZLE`
- `PHOENIX_FILAMENT_RUNOUT`
- `PHOENIX_IDLE_TIMEOUT`

### Filament and Core

- `PHOENIX_LOAD_FILAMENT`
- `PHOENIX_UNLOAD_FILAMENT`
- `PHOENIX_FILAMENT_CHANGE`
- `PHOENIX_READY_UP`
- `PHOENIX_PRESENT_TOOLHEAD`

Compatibility aliases intentionally retained:

```text
LOAD_FILAMENT -> PHOENIX_LOAD_FILAMENT
UNLOAD_FILAMENT -> PHOENIX_UNLOAD_FILAMENT
M600 -> PHOENIX_FILAMENT_CHANGE
FILAMENT_CHANGE -> PHOENIX_FILAMENT_CHANGE
```

### Calibration Pack

- `PHOENIX_PID_TUNE`
- `PHOENIX_SHAPER_CALIBRATE`
- `PHOENIX_RESONANCE_TEST_X`
- `PHOENIX_RESONANCE_TEST_Y`
- `PHOENIX_MACHINE_LEVEL`

These macros do not run `SAVE_CONFIG` automatically: the result must be checked and explicitly accepted before saving.

`PHOENIX_PRESSURE_ADVANCE_TEST` was removed on August 23, 2026 because it was redundant with the Pressure Advance calibration workflows already available in the slicer. PA calibration is therefore delegated to the slicer.

### Debug Pack

- `PHOENIX_PROBE_ACCURACY`
- `PHOENIX_PRINTER_STATUS`
- `PHOENIX_SYSTEM_SENSORS`
- `PHOENIX_REFERENCE_BED_MESH`
- `PHOENIX_STEPPER_BUZZ`

`PHOENIX_PROBE_ACCURACY` uses the native Klipper path compatible with the current Eddy configuration and does not reintroduce `probe_eddy_ng` commands.

`PHOENIX_REFERENCE_BED_MESH` is a diagnostic tool for producing a full, non-adaptive mesh; it does not replace the adaptive workflow used by `PHOENIX_START`.

### Setup Pack

- `PHOENIX_CLEANER_SETUP`

This is a manual helper that moves the toolhead to candidate cleaner coordinates for physical verification. It does not save values and does not implement the old DKEU wizard.

## `PHOENIX_START`

Manages the print-start workflow validated on Phoenix.

Current sequence:

1. set bed target temperature;
2. bring the nozzle to preparation temperature;
3. home;
4. run `PHOENIX_CLEAN_NOZZLE`;
5. turn off nozzle heating;
6. enable cooling during soak;
7. perform thermal soak;
8. wait for the nozzle to cool to the required temperature before QGL;
9. run QGL;
10. home Z again;
11. generate the bed mesh through Klipper Mainline;
12. disable the cooling used during soak;
13. bring the nozzle to final print temperature;
14. run the purge line;
15. start the print.

The bed mesh uses the native Klipper path with rapid scan and adaptive meshing.

No DKEU wrapper is loaded for `BED_MESH_CALIBRATE`.

## `PHOENIX_END`

Handles print completion according to the Phoenix workflow.

The macro belongs to the Phoenix layer and replaces the earlier DKEU end-print macros.

`M84` is not executed.

## `PHOENIX_CLEAN_NOZZLE`

Handles mechanical nozzle cleaning.

Baseline validated on the machine:

- safe Z: 10 mm;
- initial position: X236 Y359 Z2.5;
- wipe: X236 -> X271;
- return via Y360 -> X236;
- two effective cycles;
- speed: 5 mm/s.

When run manually outside a print:

1. preheat the nozzle to 170 °C;
2. perform cleaning;
3. park at X200 Y200 Z25;
4. turn off nozzle heating.

The manual physical test completed successfully.

## `PHOENIX_FILAMENT_RUNOUT`

Handles filament runout without depending on DKEU macros.

Current behavior:

```text
SAVE_GCODE_STATE
PAUSE X=100 Y=10 Z_MIN=50 RESTORE=0
RESTORE_GCODE_STATE
```

Physical validation completed on August 23, 2026:

- runout sensor opened during a real print;
- `filament not detected` detected;
- automatic pause worked correctly;
- park at X100 Y10 Z50;
- print state preserved.

During the same session the following were also physically validated:

- `PHOENIX_LOAD_FILAMENT`;
- `PHOENIX_UNLOAD_FILAMENT`;
- `PHOENIX_FILAMENT_CHANGE`, including pause, park X100 Y10 Z50, and correct continuation with `RESUME`.

## `PHOENIX_IDLE_TIMEOUT`

Implements Phoenix-specific idle policy.

If the printer is paused:

- turn off the fan with `M107`;
- turn off nozzle heating;
- preserve print state.

Otherwise:

```text
TURN_OFF_HEATERS
```

`M84` is not executed.

## Thermal management

The print-start thermal sequence is managed by the Phoenix workflow and the internal macro:

```text
_PHOENIX_THERMAL_STATE
```

The current logic explicitly distinguishes:

- bed target temperature;
- nozzle preparation temperature;
- soak;
- nozzle cooling before QGL;
- final print temperature.

Thermal behavior no longer depends on DKEU variables or frameworks.

## Functions left to Klipper Mainline

Where Klipper already provides an appropriate native function, Phoenix Macros does not add another wrapper.

In particular:

- `BED_MESH_CALIBRATE` remains on the native Klipper Mainline path;
- QGL uses native Klipper support;
- homing and kinematics remain managed by Klipper and machine configuration;
- `PROBE_ACCURACY` uses the native support available with the current Eddy configuration;
- Eddy functions belong to the active Klipper configuration, not Phoenix Macros.

## DKEU functions not carried forward

The separation was not a blind copy of DKEU macros. Several functions were deliberately removed because they were obsolete, invasive, redundant, or dependent on DKEU infrastructure.

Examples include:

- DKEU wrapper for `BED_MESH_CALIBRATE`;
- `AUTO_BED_MESH_BUILDER`;
- `ADAPTIVE_MESHING_TOGGLE`;
- `HOT_MESH`;
- `ONE_CLICK_EDDY_NG_SETUP`;
- `EDDY_TEMP_COMP`;
- legacy Z calibrations based on old probe paths;
- `M84` override;
- `Z_ASCENDER`;
- generic bed/chamber/nevermore fan systems not present in the active hardware configuration;
- DKEU power-down;
- DKEU backup and shell helpers.

Where a DKEU concept remained useful, the behavior was rewritten as an independent Phoenix macro and correctly attributed.

## External components

Phoenix can continue to use external components separate from the Phoenix layer, provided they are clearly identified and attributed.

The file:

```text
Heat_Soak_Sovol_SV08.cfg
```

correctly retains attribution to 3DPrintDemon and the corresponding upstream reference.

Those attributions must be preserved.

## Historical DKEU status

DKEU was an important phase in the evolution of the Phoenix configuration and remains documented historically and in attributions where required.

The current runtime configuration no longer loads:

```text
./Demon_Klipper_Essentials_Unified/*.cfg
./Demon_User_Files/*.cfg
Demon_User_Files_Handler_v*.cfg
```

The following were also removed from the active configuration:

- `[save_variables]` linked to `~/demon_vars.cfg`;
- `[force_move]`.

`~/demon_vars.cfg` may still exist on the machine as a historical residue, but it is not used by the Phoenix runtime.

The final audit on August 22, 2026 explicitly produced:

```text
NO ACTIVE DKEU INCLUDE
NO DKEU RUNTIME DEPENDENCY
```

DKEU therefore remains a **historical/upstream source**, not an operational dependency of the current baseline.

## Validation status

As of August 23, 2026:

- all Phoenix packs load correctly in Klipper;
- all 21 current `PHOENIX_*` macros are exposed;
- the runtime is structurally independent from DKEU;
- `PHOENIX_START` and `PHOENIX_END` were validated in real printing;
- `PHOENIX_CLEAN_NOZZLE` was physically validated;
- `PHOENIX_IDLE_TIMEOUT` was validated including the paused-print branch;
- `PHOENIX_PID_TUNE` was validated for BED and EXTRUDER;
- `PHOENIX_RESONANCE_TEST_X`, `PHOENIX_RESONANCE_TEST_Y`, and `PHOENIX_SHAPER_CALIBRATE` were validated;
- `PHOENIX_LOAD_FILAMENT`, `PHOENIX_UNLOAD_FILAMENT`, `PHOENIX_FILAMENT_RUNOUT`, and `PHOENIX_FILAMENT_CHANGE` were physically validated;
- `PHOENIX_PRESSURE_ADVANCE_TEST` was intentionally removed and is no longer part of the current runtime.

Functional validation of Phoenix Macros can therefore be considered substantially complete; future checks concern regressions, hardware changes, or new functions.

## Packaging

Phoenix Macros should be maintained in the repository as a separate, recognizable package containing the Phoenix macros that are actually active and validated on the machine.

Packaging should:

- separate Phoenix macros from external components;
- preserve upstream attribution;
- avoid importing DKEU files that are no longer needed;
- reflect the runtime actually used on the printer;
- make it easy to identify which files belong to the Phoenix layer;
- preserve the distinction between structurally loaded macros and physically validated macros.

The `.cfg` files actually active on the CB1 are the runtime source of truth and must remain synchronized with the Phoenix package published in the repository.

---

## Navigation

- ← **Previous page:** [Sovol Zero, CAN and integrated Eddy](zero-toolhead-eddy-2026-08-17.md)
- → **Next page:** [Validation and calibration](validation-and-calibration.md)
