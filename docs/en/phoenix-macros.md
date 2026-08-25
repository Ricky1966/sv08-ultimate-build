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

Phoenix now uses `save_variables` for persistent user preferences and persistent thermal state belonging to its Automatic Soak system, through a dedicated Phoenix file. The old `~/demon_vars.cfg` is not reused.

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

### Persistent Automatic Soak

Added on August 25, 2026 and currently under functional validation.

User-facing macros:

- `PHOENIX_SOAK_AUTOSTART ENABLE=0|1`
- `PHOENIX_SOAK_TEMPERATURE BED_TEMP=<temperature>`
- `PHOENIX_SOAK_START [BED_TEMP=<temperature>]`
- `PHOENIX_SOAK_STOP`
- `PHOENIX_SOAK_STATUS`

The system uses four persistent variables:

```text
phoenix_soak_autostart
phoenix_soak_temperature
phoenix_soak_credit
phoenix_soak_timestamp
```

stored through:

```ini
[save_variables]
filename: ~/printer_data/config/phoenix_variables.cfg
```

`phoenix_soak_credit` represents the number of seconds of valid thermal history accumulated by the bed; `phoenix_soak_timestamp` stores the Unix timestamp of the latest persisted credit update.

To make an absolute clock available to macros across Klipper restarts, Phoenix also uses the small Klipper helper:

```text
phoenix_clock.py
[phoenix_clock]
```

which exposes to Jinja:

```text
printer.phoenix_clock.epoch
```

The historical `~/demon_vars.cfg` file is not used by the Phoenix system.

#### Automatic startup

When `phoenix_soak_autostart` is enabled, a few seconds after Klipper starts Phoenix:

1. reads the persistent temperature, credit, and timestamp;
2. calculates elapsed time since the latest timestamp;
3. subtracts that elapsed time from previously recorded credit;
4. checks the actual bed temperature;
5. invalidates historical credit if the bed has dropped beyond the allowed thermal threshold;
6. sets the bed target;
7. waits for the bed to return to the useful temperature window;
8. applies a 60-second safety guard whenever recovered credit exists;
9. continues accumulating thermal credit;
10. periodically persists credit and timestamp.

Time spent while the machine is off **does not create credit**: it is subtracted from the previously stored credit.

Example:

```text
saved credit: 120 s
elapsed time since timestamp: 15 s
recovered credit: 105 s
```

On August 25 this exact case was verified with a real `FIRMWARE_RESTART`: Phoenix reported `recovered 105 s after 15 s offline`.

#### Thermal window

Credit does not increase during the initial heating ramp. With a 70 °C target, counting advances when the bed is at least roughly 69 °C.

If the bed drops more than 6 °C below the reference temperature, the previous thermal history is no longer considered trustworthy and credit is reset.

The minimum baseline for a complete soak is 600 seconds, meaning 10 effective minutes inside the expected thermal window.

Credit, however, is **not capped at 600 seconds**: it continues to represent the real duration of the thermal state. A bed held at soak temperature for one hour may therefore record roughly 3600 seconds of credit. This allows short restarts to preserve a long stabilization that has already occurred.

#### 60-second recovery guard

When previous credit is recovered across a restart, Phoenix does not immediately consider it valid. The bed is first returned to target temperature and must then spend at least another 60 seconds inside the useful thermal window.

The guard does not replace the 600-second minimum requirement. For example, 105 seconds of recovered credit is still insufficient after the safety minute: the system remains in `TRACKING` until the minimum required threshold is reached.

Real August 25 test:

```text
recovered: 105 s
intermediate status: credit 145 s | recovery guard 20 s
next status: credit 195 s | recovery guard 0 s
```

The observed behavior matches the intended model.

#### Persistence and write frequency

The runtime watcher operates at 10-second intervals, while `phoenix_soak_credit` and `phoenix_soak_timestamp` are written to disk every 60 seconds. This avoids unnecessarily frequent storage writes while keeping the worst-case loss after an abrupt shutdown conservatively around one minute.

Tracking continues during print preparation and printing while the bed remains in the expected thermal condition. Credit and timestamp therefore represent the actual thermal history of the bed, not just the pre-print stage.

#### Runtime thermal state

The system keeps compatibility with the existing Phoenix state:

```text
_PHOENIX_THERMAL_STATE
```

with runtime variables such as:

```text
soak_valid
soak_temp
soak_total_seconds
thermal_credit_seconds
```

The new engine also maintains its operational state in:

```text
_PHOENIX_SOAK_AUTOSTART_STATE
```

including:

```text
active
target
credit_seconds
recovery_guard_seconds
persist_counter
```

`PHOENIX_SOAK_STATUS` reports:

- Automatic Soak enabled or disabled;
- `INACTIVE`, `TRACKING`, or `VALID` status;
- configured temperature;
- actual and target bed temperature;
- current thermal credit;
- any remaining recovery guard.

`PHOENIX_SOAK_STOP` stops the current soak and turns off the bed without changing the persistent autostart preference; before stopping, it saves the current thermal history.

`PHOENIX_SOAK_START` allows manual startup using the persistent temperature or a temporary `BED_TEMP` override.

#### Integration with `PHOENIX_START`

Soak is now an **autonomous machine phase**, separate from print preparation.

`PHOENIX_START` executes two distinct phases:

**Phase 1 — bed thermal state**

1. reads thermal credit compatible with the requested target;
2. brings the bed to the requested temperature;
3. completes only any remaining soak time;
4. satisfies any recovery guard;
5. records the thermal state as valid.

The nozzle is not used to build soak time during this phase.

**Phase 2 — print preparation**

Only after the bed has satisfied the requested thermal state:

1. brings the nozzle to preparation temperature;
2. homes;
3. runs `PHOENIX_CLEAN_NOZZLE`;
4. cools the nozzle before QGL/mesh;
5. runs QGL;
6. homes Z again;
7. generates the adaptive bed mesh through Klipper Mainline;
8. brings the nozzle to final print temperature;
9. runs the purge line;
10. starts the print.

This separation prevents cleaner activity, nozzle heating, or nozzle cooling from being artificially included in soak time.

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

Manages the Phoenix print-start workflow.

The current sequence is now explicitly split between bed thermal state and print preparation:

1. set the bed target;
2. use any compatible thermal credit already available;
3. wait only for any remaining soak requirement;
4. mark the soak as valid;
5. only then bring the nozzle to preparation temperature;
6. home;
7. run `PHOENIX_CLEAN_NOZZLE`;
8. turn off nozzle heating and cool it;
9. wait for nozzle <= 50 °C;
10. run QGL;
11. home Z again;
12. generate the bed mesh through Klipper Mainline;
13. bring the nozzle to final print temperature;
14. run the purge line;
15. start the print.

The bed mesh uses the native Klipper path with rapid scan and adaptive meshing.

No DKEU wrapper is loaded for `BED_MESH_CALIBRATE`.

## `PHOENIX_END`

Handles print completion according to the Phoenix workflow.

The macro belongs to the Phoenix layer and replaces the earlier DKEU end-print macros.

`M84` is not executed.

The new persistent soak watcher is not restarted by `PHOENIX_END`: it remains autonomous and continues to represent bed thermal history when applicable.

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

Phoenix thermal management now explicitly distinguishes:

- bed target temperature;
- persistent thermal credit;
- absolute timestamp of the latest valid persistence update;
- credit decay across restarts;
- 60-second recovery guard;
- nozzle preparation temperature;
- nozzle cooling before QGL;
- final print temperature.

Thermal behavior does not depend on DKEU variables or frameworks. Persistence and clock support introduced on August 25 belong exclusively to the Phoenix layer.

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
- `phoenix_clock` helper loaded and Unix epoch readable from Jinja;
- correct reading of `phoenix_soak_autostart`, `phoenix_soak_temperature`, `phoenix_soak_credit`, and `phoenix_soak_timestamp`;
- automatic bed startup with configured 70 °C target;
- no credit accumulated during the thermal ramp;
- credit increments in 10-second steps inside the useful thermal window;
- credit and timestamp persisted every 60 seconds;
- real recovery after `FIRMWARE_RESTART`: 120 s saved, 15 s elapsed, 105 s recovered;
- real application of the 60-second recovery guard;
- observed guard countdown from 60 to 20 to 0 while credit continued to increase;
- state correctly remained `TRACKING` after guard completion while credit was still below 600 s;
- separation of soak from nozzle preparation in `PHOENIX_START` implemented and loaded.

Still to be completed before considering the new system fully validated:

- observe the new persistent engine reach `VALID` with credit >= 600 s;
- complete a full physical test of the new `PHOENIX_START` sequence after the bed-only/print-preparation separation;
- operationally test `PHOENIX_SOAK_START` and `PHOENIX_SOAK_STOP` with the final persistent engine.

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
