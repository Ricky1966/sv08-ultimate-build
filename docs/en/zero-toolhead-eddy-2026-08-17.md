# Phoenix — Sovol Zero toolhead, native Eddy, and graphite bed

**Languages:** [Italiano](../zero-toolhead-eddy-2026-08-17.md) | **English**

Migration date documented here: **2026-08-17**

Last document review: **2026-09-02**.

## How to read this page

This page originates from the work performed on August 17, 2026 to migrate Phoenix to the **Sovol Zero Extruder Kit with CAN and integrated Eddy**.

It contains two kinds of information:

- **technical configuration that still forms the Phoenix Zero baseline**, such as CAN MCU data, pins, Eddy configuration, and the Mainline changes required for compatibility;
- **test results from the August 17 session**, retained as historical validation data rather than values every later run must reproduce exactly.

Later evolution of the print workflow and macros is documented in the dedicated Phoenix pages.

## Hardware / platform

- Sovol SV08 “Phoenix”
- BTT CB1
- Klipper Mainline `v0.13.0-718-gd8659974-dirty`
- Sovol Zero Extruder Kit
- BTT U2C V2.1 USB-CAN
- Zero toolhead CAN UUID: `eedecd99868b`
- Zero firmware: `v0.13.0-718-gd8659974-dirty-20260817_135054-BTT-CB1`
- R3men graphite bed
- 1000 W AC heater with Omron SSR

## Active Zero configuration

The toolhead file is:

`/home/biqu/printer_data/config/phoenix-zero-toolhead.cfg`

The CAN MCU is:

```ini
[mcu extruder_mcu]
canbus_uuid: eedecd99868b
```

Accelerometer:

```ini
[lis2dw]
cs_pin: extruder_mcu:PB12
spi_software_sclk_pin: extruder_mcu:PB13
spi_software_mosi_pin: extruder_mcu:PB15
spi_software_miso_pin: extruder_mcu:PB14
axes_map: x,z,y
```

## Zero hotend — PT1000 sensor and Sovol ADC curve

The **Sovol Zero hotend uses a PT1000 sensor**. However, the official Sovol Zero configuration does not use the generic `sensor_type: PT1000` profile: it defines a custom ADC curve matched to the actual Zero toolboard measurement circuit.

Phoenix intentionally keeps **the same curve used by the official `Sovol3d/SOVOL-ZERO` configuration**:

```ini
[adc_temperature sovol_zero_pt1000]
temperature1: 25
resistance1: 1268.60
temperature2: 180
resistance2: 1920.98
temperature3: 300
resistance3: 2398.52
```

The extruder configuration uses:

```ini
sensor_type: sovol_zero_pt1000
pullup_resistor: 11500
sensor_pin: extruder_mcu:PA5
```

The Phoenix name `sovol_zero_pt1000` replaces Sovol's generic `my_thermistor_e` name **without changing any curve value**. The rename exists only to make the sensor type explicit and to prevent confusion with a 100K NTC thermistor.

Do not automatically replace this definition with a generic `sensor_type: PT1000`. On the Zero, keep the Sovol-provided ADC curve together with `pullup_resistor: 11500` and `extruder_mcu:PA5`, unless the hardware is deliberately changed and revalidated.

### Hotend PID

After verifying the PT1000 configuration, hotend PID must be recalibrated on the actual machine. On Phoenix, the **2026-09-02** calibration was run at **220 °C** using:

```text
PID_CALIBRATE HEATER=extruder TARGET=220
```

Validated result:

```ini
pid_Kp: 37.717
pid_Ki: 6.133
pid_Kd: 57.990
```

These PID values describe the current Phoenix machine and **are not a universal baseline**. After replacing the hotend, heater, sensor, changing the measurement circuit, or making another significant thermal-system change, run `PID_CALIBRATE` again at an appropriate operating target and store the new values in the active configuration.

## Eddy integrated into the Zero

Verified configuration:

```ini
[probe_eddy_current eddy]
sensor_type: ldc1612
i2c_mcu: extruder_mcu
i2c_address: 42
i2c_speed: 100000
i2c_software_scl_pin: extruder_mcu:PB10
i2c_software_sda_pin: extruder_mcu:PB11
x_offset: -19.8
y_offset: -0.75
descend_z: 0.5
max_sensor_hz: 7000000
reg_drive_current: 12
```

`max_sensor_hz: 7000000` forces the LDC1612 divider to 3 with the current 12 MHz Mainline clock. With divider 2, calibration could complete but `G28 Z` produced `Trigger analog error: RAW_RANGE`. With divider 3 the problem disappeared.

## Mainline patch required for Zero calibration

File:

`/home/biqu/klipper/klippy/extras/probe_eddy_current.py`

Backup created:

`/home/biqu/klipper/klippy/extras/probe_eddy_current-before-zero-compat-20260817.py`

The original Mainline calibration sampled every 0.040 mm:

```python
samp_dist = 0.040
```

With the integrated Zero probe, noise between adjacent samples prevented a usable curve from being produced.

It was changed to:

```python
samp_dist = 0.200
```

Calibration then completed with usable data over roughly 4 mm.

## Real sensor response

A direct test through the Klipper `ldc1612/dump_ldc1612` socket confirmed that Eddy responds clearly to bed distance.

Example:

- initial position: average about `6065489.954 Hz`
- after Z -3 mm: average about `6123035.633 Hz`

Change: about 57.5 kHz over 3 mm.

## Valid current calibration

With divider 3 and `samp_dist = 0.200`:

- `z: 3.051`, noise `0.004459 mm`, MAD `40.501 Hz`
- `z: 2.049`, noise `0.002637 mm`, MAD `35.780 Hz`
- `z: 1.050`, noise `0.002293 mm`, MAD `46.813 Hz`
- `z: 0.651`, noise `0.001245 mm`, MAD `27.180 Hz`
- `z: 0.450`, noise `0.000500 mm`, MAD `11.319 Hz`
- total frequency range `54937.936 Hz`
- probe noise `0.004974 mm`

The final paper test was approximately `TESTZ 0.830` and was judged correct.

## Homing

The post-homing XY park was moved to `X348 Y348` to avoid contact with the rear area / brush / guide of the new bed.

The full `G28` then completed without errors or collisions.

After `G28 Z`, the verified final position was approximately:

- X `191`
- Y `165`
- Z `10.000`

## QGL

After the first valid Eddy calibration:

`QUAD_GANTRY_LEVEL` completed with a `0.006000 mm` range after 3 retries.

After a later restart, another valid QGL produced:

- retries `2/5`
- range `0.026399 mm`

## Rapid-mesh problem: overshoot

Previous configuration:

```ini
mesh_min: 13,15
mesh_max: 333,340
scan_overshoot: 4
```

With `x_offset: -19.8`, rapid scan attempted to move the nozzle beyond the X limit (`Move out of range: 355.013 ...`).

`scan_overshoot: 0` is not valid in Mainline: the minimum allowed value is `1.0`.

Correct configuration:

```ini
scan_overshoot: 1
```

Rapid scan completes with this value.

## QGL timeout and LDC1612 data rate

During a later QGL, the following appeared:

`Error during homing probe: Communication timeout during homing`

CAN remained fully `ERROR-ACTIVE`, with no errors, drops, bus-off events, or retries.

In `klippy.log`, immediately before the timeout, repeated messages appeared:

`eddy: Amplitude Low/High warning (...)`

The Mainline LDC1612 `data_rate` was therefore checked.

File:

`/home/biqu/klipper/klippy/extras/ldc1612.py`

Backup:

`/home/biqu/klipper/klippy/extras/ldc1612-before-zero-250hz-20260817.py`

Change:

```python
self.data_rate = 400
```

->

```python
self.data_rate = 250
```

After `RESTART`:

- `G28` completed without errors
- QGL completed without timeout
- QGL: retries `2/5`, range `0.030720 mm`
- rapid mesh completed normally

This configuration became the stable integrated-Zero baseline reached during the August 17 session and remained the starting point for later validation.

## August 17 results — mechanical adjustment of the graphite bed

The first mesh clearly showed all four corners depressed, consistent with corner fasteners being overtightened.

The four corners were progressively loosened and then selectively refined.

Cold results reached approximately:

- range `0.175 mm`
- later about `0.169 mm`

The result was considered much more regular than the initial bed state.

## August 17 results — thermal validation of the graphite bed

Selected condition:

- bed `70 °C`
- nozzle `130 °C`
- initial soak about 10 minutes

Hot QGL:

- retries `2/5`
- range `0.009759 mm`

The first hot mesh after soaking mainly showed two low rear corners, with minima of about:

- X ~350 Y ~350: `-0.298 mm`
- X ~13 Y ~340: `-0.271 mm`

The two rear fasteners were selectively loosened.

The next mesh dropped to about `0.224 mm` range.

After a further micro-adjustment of the X0 Y0 fastener, the best practical result of the session was:

- **range about `0.190 mm` at 70 °C**

The bed was not perfectly flat, but its shape was much more regular and usable than the initial strongly warped state. The decision was made not to keep chasing mechanical flatness beyond this point.

## Mesh saved during the August 17 session

The hot mesh with about `0.190 mm` range was saved using `SAVE_CONFIG`.

After restart, Phoenix returned to `Ready`.

## Main session backups

- `/home/biqu/klipper/klippy/extras/probe_eddy_current-before-zero-compat-20260817.py`
- `/home/biqu/klipper/klippy/extras/ldc1612-before-zero-250hz-20260817.py`
- `/home/biqu/printer_data/config/phoenix-zero-toolhead-before-divider3-20260817.cfg`
- `/home/biqu/printer_data/config/printer-before-zero-eddy-divider3-save-20260817.cfg`
- `/home/biqu/printer_data/config/printer-before-zero-graphite-mesh-save-20260817.cfg`

## State verified at the end of the August 17 session

Working:

- Zero CAN
- integrated Eddy over software I2C
- current calibration
- XY homing
- Z homing
- full G28
- QGL
- rapid bed mesh
- rapid mesh with corrected overshoot
- Eddy at 250 Hz without the QGL timeout observed at 400 Hz
- hot graphite-bed mesh

---

## Navigation

- ← **Previous page:** [Base Mainline configuration](base-configuration.md)
- → **Next page:** [Phoenix Macros](phoenix-macros.md)
