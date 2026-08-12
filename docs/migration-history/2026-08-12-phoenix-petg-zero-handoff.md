# Phoenix — PETG, graphite bed and Sovol Zero toolhead handoff

Date: 2026-08-12

This note records only the Phoenix work relevant to the interrupted PETG calibration, the R3men graphite bed, and the preparation for the Sovol Zero toolhead migration.

## Current Phoenix state

- Printer: Sovol SV08 “Phoenix”.
- Host: BTT CB1, Klipper Mainline.
- Mainboard/CB1/camera/touchscreen were checked after the toolhead incident and appear healthy.
- The previous toolhead is out of service; the machine must not be treated as print-ready until the replacement Zero toolhead is installed and validated.
- The R3men graphite heated bed is mechanically installed, but the final mounting stack is still under review.
- Do not perform bed PID, QGL, Eddy/Z calibration, mesh or printing until the Zero toolhead is installed and the mechanical bed arrangement is confirmed.

## PETG calibration — last valid values

Filament: Sovol PETG, spool range 230–250 °C.

Last chosen/validated working values before the toolhead failure:

- nozzle temperature: 240 °C;
- bed: 80 °C first layer, 75 °C subsequent layers;
- Flow Ratio: 0.903;
- Pressure Advance: 0.030;
- Max Volumetric Speed: 23.5 mm³/s.

Cooling calibration was not completed. A 20–25% fan range was not enough for the bridge/cooling tower. A later 30–50% test was interrupted by the clog/toolhead problem.

These PETG values are a useful baseline only. After the Zero migration and graphite-bed commissioning, re-check at least hotend PID, retraction, cooling/bridging and first layer before considering the PETG profile final.

## R3men graphite heated bed

Product: R3men graphite heated bed for Sovol SV08.

Known electrical configuration:

- 1000 W AC heater;
- Omron G3NB-225B-1 25 A SSR;
- 230 V configuration;
- on this Phoenix, PSU red wire is Line (L), black wire is Neutral (N);
- one white heater lead goes to Neutral;
- the other white heater lead goes to one SSR LOAD terminal;
- Line goes to the other SSR LOAD terminal;
- yellow-green PE from the bed is bonded to chassis/earth;
- yellow 110 V heater lead is isolated and unused;
- original mainboard bed output controls the SSR DC input.

The active `printer.cfg` bed thermistor section was already changed to:

```ini
[heater_bed]
heater_pin: PA0
sensor_type: NTC 100K MGB18-104F39050L32
sensor_pin: PC5
max_power: 1.0
min_temp: 5
max_temp: 105
```

Old bed PID values were commented and must not be reused blindly.

### Mechanical mounting still to resolve

The graphite bed currently sits approximately 10 mm above the rear brush/wiper, which strongly suggests the current mechanical stack needs review before any brush modification.

Open question to resolve from R3men documentation/support/community:

- use metal standoffs only;
- use yellow springs only;
- or use standoffs and springs together.

Current working hypothesis is springs-only, but this is not yet confirmed and must not be treated as final.

Also verify whether the original SV08 plastic lower frame/cover/cornice is meant to remain or be removed. Do not modify the rear brush until the correct bed stack is known.

Once the mechanical stack is final and Zero is working, perform a fresh bed PID, QGL, hot mesh and first-layer validation.

## Old toolhead incident

The previous SV08 toolhead stopped working after a hardware incident. The two original toolhead boards are marked:

- `DLPT_TOP_V1.1`;
- `DLPT_BOTTOM_V1.1`;
- date marking `20240412`.

No obvious visible burn was found.

A specific mechanical/electrical risk was identified with the Sovol Eddy replacement cable: the short Dupont jumper feeding +5 V is connected to one of four exposed programming/flashing pins. The toolhead cover can press on or bend that pin. This is considered a plausible contributor to the failure and should be avoided on future layouts.

Host-side diagnostics after the failure showed Klipper, Moonraker, Crowsnest and nginx active. The main MCU remained visible over USB and no USB overcurrent was found. This supports the conclusion that CB1 and the mainboard are likely healthy.

## Replacement: Sovol Zero Extruder Kit

Replacement ordered: Sovol Zero Extruder Kit, approximately €69.99 from Sovol EU.

The Zero mechanically fits the SV08 carriage using the same three M3 mounting points according to the reference guides used during preparation.

The Zero has its own CAN-capable toolboard and integrates:

- LDC1612 Eddy sensor;
- LIS2DW accelerometer;
- extruder driver;
- heater/fan control.

No EBB36/EBB42 is planned.

## CAN architecture prepared

BTT U2C V2.1 USB-to-CAN adapter ordered. Intended architecture:

```text
CB1 -> USB -> BTT U2C -> CANH/CANL/GND -> Sovol Zero
                         + separate 24V/GND power to Zero
```

The 3-wire CAN connector on the Zero is officially labeled:

```text
CANL  GND  CANH
```

Do not trust cable colors alone. The existing SV08 cable may be reusable physically, but the original white/green conductors were USB D-/D+ in the old arrangement. Verify continuity and connector labels before applying power.

`can-utils` is installed and the required kernel CAN modules are available.

Prepared network configuration:

```ini
allow-hotplug can0
iface can0 can static
    bitrate 1000000
    up ip link set $IFACE txqueuelen 128
```

File:

```text
/etc/network/interfaces.d/can0
```

The 1 Mbit/s value is the current prepared setting; validate it against the actual U2C/Zero firmware behavior when hardware arrives rather than treating it as permanently certified.

## Official Zero reference data already collected

Official source used during preparation:

```text
https://github.com/Sovol3d/SOVOL-ZERO
```

Downloaded on Phoenix:

```text
/home/biqu/printer_data/config/Zero_Extra_Pin_definition.pdf
/home/biqu/printer_data/config/Zero_official_printer.cfg
```

Important official Zero pin data:

```text
Extruder STEP PA8
Extruder DIR  PA9
Extruder EN   PA11
Extruder UART PA12
Heater        PB7
Thermistor    PA5
Part fan      PB0
Hotend fan    PA6
Hotend tach   PA1
CAN RX/TX     PB8/PB9
```

Official Zero thermistor table:

```ini
[adc_temperature my_thermistor_e]
temperature1: 25
resistance1: 1268.60
temperature2: 180
resistance2: 1920.98
temperature3: 300
resistance3: 2398.52
```

Official Zero extruder baseline:

- `rotation_distance: 6.5`;
- `microsteps: 16`;
- TMC run current `0.8`;
- sense resistor `0.150`.

Do not copy the official example CAN UUID (`61755fe321ac`) into Phoenix. Query the actual hardware UUID after connection.

Do not copy the official example Eddy `reg_drive_current`, calibration curve, `z_offset`, `vir_contact_speed`, or example center/endstop coordinates. The new sensor must be calibrated on Phoenix.

## Phoenix-specific configuration strategy for Zero

A commented, non-active preparation file was created:

```text
/home/biqu/printer_data/config/phoenix-zero-toolhead.cfg
```

It is intentionally not included yet.

The draft preserves Phoenix behavior where appropriate and maps the Zero hardware to a future `extruder_mcu` CAN MCU.

Important Phoenix values to retain initially:

- machine reference center: X191 Y165;
- existing custom homing strategy;
- existing KAMP/Demon compatibility choices;
- Pressure Advance baseline `0.025` / smooth time `0.035` until PETG is recalibrated;
- `max_extrude_only_distance: 500` and `max_extrude_cross_section: 5` initially, rather than blindly importing the Zero example values.

Likely Zero physical Eddy XY offsets from the official config:

```text
x_offset: -19.8
y_offset: -0.75
```

At nozzle X191 Y165 this places the sensor at approximately X171.2 Y164.25, safely on the bed. There is no reason to import the official Zero example center `96,76.2` or `safe_z_home` into Phoenix.

For Eddy, preserve the Phoenix native-Eddy tuning initially where possible:

```text
descend_z: 0.5
max_sensor_hz: 9000000
```

but re-calibrate the actual Zero sensor completely.

## Native Eddy / previous Phoenix state

Before the toolhead failure Phoenix had already been migrated to Klipper Mainline and native Eddy. The previous active MCU arrangement used `extra_mcu` over USB; that arrangement will be replaced by the Zero CAN MCU.

The old saved Eddy calibration and Z relationship are invalid for the new toolhead/graphite combination and must not be reused as calibration truth.

Phoenix homing reference remains X191 Y165. Existing homing behavior should be preserved unless a concrete Zero hardware requirement proves otherwise.

## First power-up / migration sequence when hardware arrives

Proceed one step at a time.

1. Connect only the U2C to CB1 USB and inspect USB/CAN state.
2. Verify U2C mode, bitrate and termination.
3. Power everything off before wiring Zero CAN and 24 V.
4. Verify CANH, CANL and GND by connector labels/continuity, not color assumptions.
5. Supply Zero 24 V/GND separately as intended.
6. Bring up `can0`.
7. Query the real Zero UUID with Klipper's `canbus_query.py`.
8. Only then replace the old `extra_mcu` toolhead configuration with the actual `extruder_mcu` CAN UUID.
9. Validate toolhead MCU temperature, thermistor reading, LIS2DW, fans and Eddy communication before heating or moving.
10. Verify heater output remains off when expected; then perform controlled heater/fan tests.
11. Re-calibrate Eddy completely, including actual drive current/calibration data and Z relationship.
12. Re-run input shaper/resonance validation for the new toolhead.
13. Run hotend PID for the Zero hotend.
14. Finalize graphite-bed mechanical mounting, then run bed PID.
15. Run QGL and hot bed mesh.
16. Validate first layer.
17. Resume PETG calibration from the recorded baseline, especially retraction and cooling.

## Working rules for the next session

- Work in Italian.
- One machine action / one command block at a time, then wait for output.
- Do not use `M84`.
- Do not make unrelated motor-value changes.
- Do not reuse saved Z/Eddy values from the old toolhead as if valid for Zero.
- Do not modify the rear brush until the graphite-bed mounting height is resolved.
- Primary goal: install and validate Zero safely, then commission the graphite bed, then return Phoenix to reliable PETG printing.
