# Phoenix — Sovol Zero toolhead, Eddy nativo e piatto in grafite

Data della migrazione documentata: **2026-08-17**

Ultima revisione del documento: **2026-09-02**.

## Come leggere questa pagina

Questa pagina nasce dal lavoro di migrazione della Phoenix alla **Sovol Zero Extruder Kit con CAN ed Eddy integrato** eseguito il 17 agosto 2026.

Contiene due tipi di informazioni:

- **configurazione tecnica che costituisce ancora la baseline Zero della Phoenix**, come MCU CAN, pin, configurazione Eddy e modifiche Mainline necessarie;
- **risultati di test della sessione del 17 agosto**, conservati come dati storici di validazione e non come valori che ogni esecuzione successiva deve necessariamente riprodurre.

Le successive evoluzioni del workflow di stampa e delle macro sono documentate nelle pagine Phoenix dedicate.

## Hardware / piattaforma

- Sovol SV08 “Phoenix”
- BTT CB1
- Klipper Mainline `v0.13.0-718-gd8659974-dirty`
- Sovol Zero Extruder Kit
- BTT U2C V2.1 USB-CAN
- UUID CAN toolhead Zero: `eedecd99868b`
- firmware Zero: `v0.13.0-718-gd8659974-dirty-20260817_135054-BTT-CB1`
- piatto R3men in grafite
- heater AC 1000 W con SSR Omron

## Configurazione Zero attiva

Il file toolhead è:

`/home/biqu/printer_data/config/phoenix-zero-toolhead.cfg`

La MCU CAN è:

```ini
[mcu extruder_mcu]
canbus_uuid: eedecd99868b
```

Accelerometro:

```ini
[lis2dw]
cs_pin: extruder_mcu:PB12
spi_software_sclk_pin: extruder_mcu:PB13
spi_software_mosi_pin: extruder_mcu:PB15
spi_software_miso_pin: extruder_mcu:PB14
axes_map: x,z,y
```

## Hotend Zero — sensore PT1000 e curva ADC Sovol

L'hotend della **Sovol Zero utilizza un sensore PT1000**. La configurazione ufficiale Sovol Zero non usa però il profilo generico `sensor_type: PT1000`: definisce una curva ADC custom, associata al circuito reale della toolboard Zero.

Phoenix mantiene intenzionalmente **la stessa curva della configurazione ufficiale `Sovol3d/SOVOL-ZERO`**:

```ini
[adc_temperature sovol_zero_pt1000]
temperature1: 25
resistance1: 1268.60
temperature2: 180
resistance2: 1920.98
temperature3: 300
resistance3: 2398.52
```

La configurazione dell'extruder usa:

```ini
sensor_type: sovol_zero_pt1000
pullup_resistor: 11500
sensor_pin: extruder_mcu:PA5
```

Il nome Phoenix `sovol_zero_pt1000` sostituisce il nome generico Sovol `my_thermistor_e` **senza modificare alcun valore della curva**. La rinomina serve soltanto a rendere esplicito il tipo di sensore e a evitare che venga confuso con un NTC 100K.

Non sostituire automaticamente questa definizione con un generico `sensor_type: PT1000`: sulla Zero va conservata la curva ADC fornita da Sovol insieme al `pullup_resistor: 11500` e al pin `extruder_mcu:PA5`, salvo una modifica hardware consapevole e verificata.

### PID hotend

Dopo aver verificato la configurazione del PT1000, il PID dell'hotend deve essere ricalibrato sulla macchina reale. Sulla Phoenix la calibrazione del **2026-09-02** è stata eseguita a **220 °C** con:

```text
PID_CALIBRATE HEATER=extruder TARGET=220
```

Risultato validato:

```ini
pid_Kp: 37.717
pid_Ki: 6.133
pid_Kd: 57.990
```

Questi PID descrivono la Phoenix attuale e **non sono una baseline universale**. Dopo sostituzione di hotend, heater, sensore, modifica del circuito di lettura o altra variazione significativa del sistema termico, eseguire nuovamente `PID_CALIBRATE` al target operativo appropriato e salvare i nuovi valori nella configurazione attiva.

## Eddy integrato nella Zero

Configurazione verificata:

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

Il valore `max_sensor_hz: 7000000` forza il divider LDC1612 a 3 con il clock Mainline corrente da 12 MHz. Con divider 2 la calibrazione poteva riuscire ma `G28 Z` produceva `Trigger analog error: RAW_RANGE`. Con divider 3 il problema è scomparso.

## Patch Mainline necessaria per la calibrazione Zero

File:

`/home/biqu/klipper/klippy/extras/probe_eddy_current.py`

Backup creato:

`/home/biqu/klipper/klippy/extras/probe_eddy_current-before-zero-compat-20260817.py`

La calibrazione Mainline originale campionava ogni 0.040 mm:

```python
samp_dist = 0.040
```

Con la Zero integrata il rumore tra campioni adiacenti impediva di ottenere una curva utilizzabile.

È stato portato a:

```python
samp_dist = 0.200
```

La calibrazione ha quindi completato con dati utilizzabili su circa 4 mm.

## Risposta reale del sensore

Test diretto via socket Klipper `ldc1612/dump_ldc1612` ha confermato che Eddy reagisce nettamente alla distanza dal piatto.

Esempio:

- posizione iniziale: media circa `6065489.954 Hz`
- dopo Z -3 mm: media circa `6123035.633 Hz`

Variazione circa 57.5 kHz su 3 mm.

## Calibrazione current valida

Con divider 3 e `samp_dist = 0.200`:

- `z: 3.051`, noise `0.004459 mm`, MAD `40.501 Hz`
- `z: 2.049`, noise `0.002637 mm`, MAD `35.780 Hz`
- `z: 1.050`, noise `0.002293 mm`, MAD `46.813 Hz`
- `z: 0.651`, noise `0.001245 mm`, MAD `27.180 Hz`
- `z: 0.450`, noise `0.000500 mm`, MAD `11.319 Hz`
- total frequency range `54937.936 Hz`
- probe noise `0.004974 mm`

Il test carta finale era circa `TESTZ 0.830` ed è stato giudicato corretto.

## Homing

Il parcheggio post-homing XY è stato portato a `X348 Y348` per evitare contatti con zona posteriore / brush / guida del nuovo piatto.

Il full `G28` è stato poi completato senza errori e senza urti.

Dopo `G28 Z` la posizione finale verificata era circa:

- X `191`
- Y `165`
- Z `10.000`

## QGL

Dopo la prima calibrazione valida Eddy:

`QUAD_GANTRY_LEVEL` ha completato con range `0.006000 mm` dopo 3 retry.

Dopo un successivo restart, altro QGL valido:

- retries `2/5`
- range `0.026399 mm`

## Problema rapid mesh: overshoot

Configurazione precedente:

```ini
mesh_min: 13,15
mesh_max: 333,340
scan_overshoot: 4
```

Con `x_offset: -19.8`, il rapid scan tentava di portare il nozzle oltre il limite X (`Move out of range: 355.013 ...`).

`scan_overshoot: 0` non è valido in Mainline: minimo consentito `1.0`.

Configurazione corretta:

```ini
scan_overshoot: 1
```

Con questo valore il rapid scan completa.

## Timeout QGL e data rate LDC1612

Durante un QGL successivo è comparso:

`Error during homing probe: Communication timeout during homing`

Il CAN risultava perfettamente `ERROR-ACTIVE`, senza errori, dropped, bus-off o retry.

Nel `klippy.log`, immediatamente prima del timeout, comparivano ripetuti:

`eddy: Amplitude Low/High warning (...)`

È stato quindi verificato il `data_rate` Mainline del LDC1612.

File:

`/home/biqu/klipper/klippy/extras/ldc1612.py`

Backup:

`/home/biqu/klipper/klippy/extras/ldc1612-before-zero-250hz-20260817.py`

Modifica:

```python
self.data_rate = 400
```

->

```python
self.data_rate = 250
```

Dopo `RESTART`:

- `G28` completato senza errori
- QGL completato senza timeout
- QGL: retries `2/5`, range `0.030720 mm`
- rapid mesh completata normalmente

Questa configurazione costituì la baseline stabile della Zero integrata raggiunta nella sessione del 17 agosto ed è rimasta il punto di partenza delle validazioni successive.

## Risultati del 17 agosto — regolazione meccanica del piatto in grafite

La prima mesh mostrava chiaramente i quattro angoli depressi, compatibile con i fissaggi angolari troppo serrati.

Sono stati progressivamente allentati i quattro angoli, poi rifiniti in modo selettivo.

A freddo si è arrivati a circa:

- range `0.175 mm`
- successivamente circa `0.169 mm`

Il risultato è stato considerato molto più regolare del piatto iniziale.

## Risultati del 17 agosto — validazione termica del piatto in grafite

Condizione scelta:

- bed `70 °C`
- nozzle `130 °C`
- soak iniziale circa 10 minuti

QGL a caldo:

- retries `2/5`
- range `0.009759 mm`

La prima mesh a caldo dopo soak ha evidenziato soprattutto due angoli posteriori bassi, con minimi circa:

- X ~350 Y ~350: `-0.298 mm`
- X ~13 Y ~340: `-0.271 mm`

Sono stati allentati selettivamente i due fissaggi posteriori.

La mesh successiva è scesa a circa `0.224 mm` di range.

Dopo ulteriore micro-regolazione del fissaggio X0 Y0 il miglior risultato pratico della serata è stato:

- **range circa `0.190 mm` a 70 °C**

Il piatto non è perfettamente piatto, ma la forma è molto più regolare e utilizzabile rispetto allo stato iniziale “a taco”. Si è deciso di non continuare a inseguire la planarità meccanicamente oltre questo punto.

## Mesh salvata nella sessione del 17 agosto

La mesh a caldo da circa `0.190 mm` è stata salvata con `SAVE_CONFIG`.

Dopo il restart Phoenix è tornata `Ready`.

## Backup principali della sessione

- `/home/biqu/klipper/klippy/extras/probe_eddy_current-before-zero-compat-20260817.py`
- `/home/biqu/klipper/klippy/extras/ldc1612-before-zero-250hz-20260817.py`
- `/home/biqu/printer_data/config/phoenix-zero-toolhead-before-divider3-20260817.cfg`
- `/home/biqu/printer_data/config/printer-before-zero-eddy-divider3-save-20260817.cfg`
- `/home/biqu/printer_data/config/printer-before-mesh-overshoot-zero-20260817.cfg`
- `/home/biqu/printer_data/config/printer-before-zero-graphite-mesh-save-20260817.cfg`

## Stato verificato a fine sessione del 17 agosto

Funzionano:

- CAN Zero
- Eddy integrato via software I2C
- calibrazione current
- homing XY
- homing Z
- full G28
- QGL
- rapid bed mesh
- rapid mesh con overshoot corretto
- Eddy a 250 Hz senza il timeout QGL osservato a 400 Hz
- mesh a caldo del piatto in grafite

---

## Navigazione

- ← **Pagina precedente:** [Configurazione base Mainline](base-configuration.md)
- → **Pagina successiva:** [Phoenix Macros](phoenix-macros.md)
