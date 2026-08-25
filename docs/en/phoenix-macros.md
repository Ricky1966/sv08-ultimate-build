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

On August 22, 2026, the structural audit of the active runtime on the machine was completed; the main functional validation was completed on August 23, 2026. On August 25, integration of the new Phoenix Automatic Soak system began and is currently undergoing functional validation on the machine.

The verified Phoenix baseline includes:

- no active DKEU include in `printer.cfg`;
- no operational DKEU reference in `phoenix-*.cfg` files;
- no `force_move` use;
- no `M84` use;
- 21 `PHOENIX_*` macros validated in the August 23 baseline;
- 5 new user-facing Phoenix Automatic Soak macros added on August 25 and currently under validation;
- Klipper restarted without configuration, Jinja, or runtime errors during structural pack validation.

Phoenix now uses `save_variables` exclusively for persistent user preferences belonging to its Automatic Soak system, through a dedicated Phoenix file. The old `~/demon_vars.cfg` is not reused.

This confirms the **structural/runtime independence** of Phoenix Macros from DKEU. Physical validation of individual macros is performed separately on the machine and must not be confused with successful loading by Klipper alone.

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

### Automatic Soak

Added on August 25, 2026 and currently under functional validation.

User-facing macros:

- `PHOENIX_SOAK_AUTOSTART ENABLE=0|1`
- `PHOENIX_SOAK_TEMPERATURE BED_TEMP=<temperature>`
- `PHOENIX_SOAK_START [BED_TEMP=<temperature>]`
- `PHOENIX_SOAK_STOP`
- `PHOENIX_SOAK_STATUS`

The system uses two persistent preferences:

```text
phoenix_soak_autostart
phoenix_soak_temperature
```

stored through:

```ini
[save_variables]
filename: ~/printer_data/config/phoenix_variables.cfg
```

The historical `~/demon_vars.cfg` file is not used by the Phoenix system.

#### Automatic startup

When `phoenix_soak_autostart` is enabled, a few seconds after Klipper starts Phoenix:

1. reads the configured temperature;
2. sets the bed target;
3. explicitly reports in the console that Automatic Soak has started;
4. waits for the bed to enter the useful temperature window;
5. begins accumulating thermal credit;
6. records the soak as valid when the required credit is complete.

The timer does **not** start during the heating ramp. With a 70 °C target, credit begins when the bed reaches roughly 69 °C.

The current soak baseline is 600 seconds, meaning 10 effective minutes inside the expected thermal window.

#### Thermal state and credit

The system reuses the existing Phoenix state:

```text
_PHOENIX_THERMAL_STATE
```

including runtime variables such as:

```text
soak_valid
soak_temp
soak_total_seconds
thermal_credit_seconds
```

The user's preference is persistent; the thermal validity of the soak itself is **not** stored permanently across power cycles.

`PHOENIX_SOAK_STATUS` reports:

- Automatic Soak enabled or disabled;
- `INACTIVE`, `TRACKING`, or `VALID` status;
- configured temperature;
- actual and target bed temperature;
- accumulated thermal credit.

`PHOENIX_SOAK_STOP` stops the current soak and turns off the bed without changing the persistent autostart preference.

`PHOENIX_SOAK_START` allows manual startup using the persistent temperature or a temporary `BED_TEMP` override.

#### Integration with `PHOENIX_START`

The integration logic allows `PHOENIX_START` to consume:

- a soak that is already fully valid;
- or partial credit accumulated by an Automatic Soak that is still running.

This prevents a print started after several minutes of preheating from necessarily repeating the entire soak from zero.

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

1. set the bed target temperature;
2. recover compatible thermal credit already accumulated by the Phoenix system;
3. bring the nozzle to preparation temperature;
4. home;
5. run `PHOENIX_CLEAN_NOZZLE`;
6. turn off nozzle heating;
7. enable cooling during any remaining soak;
8. complete only the remaining soak time;
9. wait for the nozzle to cool to the required temperature before QGL;
10. run QGL;
11. home Z again;
12. generate the bed mesh through Klipper Mainline;
13. disable the cooling used during soak;
14. bring the nozzle to final print temperature;
15. run the purge line;
16. start the print.

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
- thermal credit already accumulated;
- nozzle cooling before QGL;
- final print temperature.

Thermal behavior does not depend on DKEU variables or frameworks. Persistence introduced on August 25 concerns only explicit Phoenix user preferences and uses a dedicated file.

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

The new Phoenix Automatic Soak is a separate system and does not derive its state or persistence logic from the legacy timer contained in that file.

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

Since August 25, Phoenix has its own `[save_variables]`, independent of DKEU, linked to:

```text
~/printer_data/config/phoenix_variables.cfg
```

The final audit on August 22, 2026 explicitly produced:

```text
NO ACTIVE DKEU INCLUDE
NO DKEU RUNTIME DEPENDENCY
```

DKEU therefore remains a **historical/upstream source**, not an operational dependency of the current baseline.

## Validation status

Consolidated baseline as of August 23, 2026:

- all Phoenix packs load correctly in Klipper;
- all 21 baseline `PHOENIX_*` macros are exposed;
- the runtime is structurally independent from DKEU;
- `PHOENIX_START` and `PHOENIX_END` were validated in real printing;
- `PHOENIX_CLEAN_NOZZLE` was physically validated;
- `PHOENIX_IDLE_TIMEOUT` was validated including the paused-print branch;
- `PHOENIX_PID_TUNE` was validated for BED and EXTRUDER;
- `PHOENIX_RESONANCE_TEST_X`, `PHOENIX_RESONANCE_TEST_Y`, and `PHOENIX_SHAPER_CALIBRATE` were validated;
- `PHOENIX_LOAD_FILAMENT`, `PHOENIX_UNLOAD_FILAMENT`, `PHOENIX_FILAMENT_RUNOUT`, and `PHOENIX_FILAMENT_CHANGE` were physically validated;
- `PHOENIX_PRESSURE_ADVANCE_TEST` was intentionally removed and is no longer part of the current runtime.

### Automatic Soak — August 25, 2026 validation

Verified so far on the machine:

- Phoenix `save_variables` loaded correctly;
- `phoenix_variables.cfg` created and persistent;
- correct reading of `phoenix_soak_autostart` and `phoenix_soak_temperature`;
- autostart preference can be enabled without immediate heating;
- automatic bed heating starts after restart with configured 70 °C target;
- no credit is accumulated during the thermal ramp;
- credit begins in the expected temperature window;
- credit increments in 10-second steps.

Still to be completed before considering the function validated:

- reaching and recording `600/600 s` as `VALID`;
- loading and testing `PHOENIX_SOAK_START` and `PHOENIX_SOAK_STOP`;
- validating the new `PHOENIX_SOAK_STATUS` states `INACTIVE`, `TRACKING`, and `VALID`;
- testing transfer of partial/valid credit into `PHOENIX_START`.

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
