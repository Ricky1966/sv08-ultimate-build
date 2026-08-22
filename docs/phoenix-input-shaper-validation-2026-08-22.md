# Phoenix — Input Shaper validation 2026-08-22

## Physical validation

`PHOENIX_RESONANCE_TEST_X` and `PHOENIX_RESONANCE_TEST_Y` completed successfully with native Klipper resonance sweeps and CSV output under `/tmp`.

`PHOENIX_SHAPER_CALIBRATE` then completed successfully and produced the following recommended values:

```text
X: shaper_type = ei
X: shaper_freq = 40.6 Hz

Y: shaper_type = mzv
Y: shaper_freq = 40.2 Hz
```

The calibration was accepted and saved with `SAVE_CONFIG`. Klipper restarted normally and a subsequent `G28` completed successfully.

## Mandatory future recalibration

The current calibration was performed with the printer in its present mechanical configuration. When the insulated enclosure panels are reinstalled, `PHOENIX_SHAPER_CALIBRATE` must be run again and the resulting values reviewed/saved, because the added panel mass and structural coupling can alter the machine resonance response.

This recalibration is mandatory before declaring the final closed-machine tuning complete.
