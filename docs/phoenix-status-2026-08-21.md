# Phoenix — stato tecnico 2026-08-21

Questa nota fotografa lo stato reale della Sovol SV08 **Phoenix** al termine della sessione del 21 agosto 2026. È una nota macchina-specifica, utile per revisione tecnica e confronto con Sovol; non va interpretata come preset universale per altre SV08.

## Obiettivo della sessione

Portare Phoenix a una stampa PLA affidabile riducendo al minimo il debug non necessario, dopo la migrazione a Klipper Mainline e l'evoluzione hardware della toolhead / Eddy.

## Hardware corrente

- Sovol SV08 "Phoenix"
- BTT CB1
- Klipper Mainline `v0.13.0-718-gd8659974-dirty`
- Sovol Zero Extruder Kit
- Eddy Zero collegato a `extruder_mcu`
- BTT U2C V2.1
- R3men graphite bed + magnetic pad

## Eddy corrente

Configurazione verificata oggi:

```ini
[probe_eddy_current eddy]
sensor_type: ldc1612
i2c_mcu: extruder_mcu
i2c_address: 42
i2c_speed: 100000
x_offset: -19.8
y_offset: -0.75
descend_z: 0.5
max_sensor_hz: 7000000
reg_drive_current: 12
```

Questi valori sostituiscono, per **Phoenix attuale**, i vecchi valori storici documentati durante la prima migrazione Eddy nativa (`extra_mcu`, offset precedenti, `max_sensor_hz=9000000`, `reg_drive_current=22`). I vecchi dati restano utili solo come cronologia della precedente configurazione hardware.

## Patch Klipper attualmente necessarie sulla Phoenix

Sono presenti modifiche locali deliberate e **non vanno rimosse durante troubleshooting ordinario**:

- `probe_eddy_current.py`: distanza campioni calibrazione `0.040 -> 0.200`
- `ldc1612.py`: sample/data rate `400 -> 250 Hz`
- `bed_mesh.py`: `wait_moves()` prima di `save_profile`
- `bed_mesh.py`: compatibilità Eddy nel percorso scan:

```python
can_scan = "eddy" in probe_name  # eddy-ng
```

Queste patch descrivono lo stato reale della macchina testata il 2026-08-21.

## Bed mesh corrente

```ini
[bed_mesh]
speed: 200
horizontal_move_z: 1.5
algorithm: bicubic
mesh_min: 15,15
mesh_max: 330,335
probe_count: 15,15
fade_start: 0
fade_end: 10
scan_overshoot: 1
zero_reference_position: 175,175
```

### Validazione rapid scan

Il percorso `rapid_scan` è stato confrontato con misure point-by-point e con true scan.

Risultati significativi a freddo:

- automatic / descend point-by-point: circa `0.1715 mm`
- automatic / descend seconda prova: circa `0.1693 mm`
- rapid200 precedente: `0.098610 mm`
- rapid100: `0.099375 mm`
- true scan: `0.095043 mm`
- scan e rapid coincidono nell'ordine di `2–3 µm RMS`

Conclusione operativa: **rapid_scan è considerato validato per l'uso pratico sulla Phoenix attuale**.

## Bug DKEU trovato il 2026-08-21

File:

```text
/home/biqu/printer_data/config/Demon_Klipper_Essentials_Unified/demon_core_assets_v2.3.5.cfg
```

Il wrapper:

```ini
[gcode_macro BED_MESH_CALIBRATE]
rename_existing: BED_MESH_CALIBRATE_BASE
```

non inoltrava i parametri ricevuti. Di conseguenza un comando come:

```gcode
BED_MESH_CALIBRATE METHOD=rapid_scan SCAN_SPEED=200
```

poteva essere intercettato dal wrapper ma arrivare alla routine base senza `METHOD=rapid_scan`.

Backup creato prima della modifica:

```text
demon_core_assets_v2.3.5.cfg.before-phoenix-native-mesh-20260821
```

Correzione applicata: tutte le chiamate interessate a `BED_MESH_CALIBRATE_BASE` inoltrano ora `{rawparams}`.

La correzione è stata verificata facendo eseguire realmente il rapid scan attraverso il wrapper.

## Validazione termica del bed

Protocollo di riferimento usato oggi:

- nozzle heater off / nozzle freddo
- part cooling fan 100%
- bed `70 °C`
- 15 minuti di soak dopo il raggiungimento di 70 °C
- QGL due volte
- `BED_MESH_CALIBRATE METHOD=rapid_scan SCAN_SPEED=200`

### Primo giro a 70 °C

QGL:

- `Retries: 2/5`, range `0.015051`
- secondo QGL: `Retries: 0/5`, range `0.004103`

Mesh:

- max circa `+0.063 mm`
- min circa `-0.060 mm`
- range circa `0.123 mm`

### Secondo giro a 70 °C

QGL:

- range `0.004879 mm`
- secondo QGL: range `0.002560 mm`

Mesh:

- max `+0.063 mm`
- min `-0.061 mm`
- range `0.124–0.125 mm`

Le due mesh termicamente stabilizzate sono praticamente sovrapponibili. Non sono emersi spike o comportamento anomalo di Eddy.

Snapshot precedente conservato:

```text
/home/biqu/phoenix-mesh-cold-allminus18-rapid200-post-wrapper-fix-20260821.cfg
```

La seconda mesh termicamente stabilizzata è stata salvata come profilo `default` dopo backup del `printer.cfg`.

## Calibrazione Z Eddy rifatta

Comando:

```gcode
PROBE_EDDY_CURRENT_CALIBRATE CHIP=eddy
```

Paper / spessimetro test:

- partenza circa `10.045`
- arrivo `TESTZ -0.960`
- spessimetro `0.05 mm` con leggerissimo attrito
- `ACCEPT`

Output significativo:

```text
z: 3.051 # noise 0.000561mm, MAD_Hz=5.064
z: 2.049 # noise 0.001469mm, MAD_Hz=20.299
z: 1.050 # noise 0.000710mm, MAD_Hz=15.256
z: 0.651 # noise 0.000593mm, MAD_Hz=15.300
z: 0.450 # noise 0.000451mm, MAD_Hz=12.741
Total frequency range: 60035.251 Hz
probe_eddy_current: noise 0.001220mm, MAD_Hz=15.777 in 525 queries
```

La configurazione è stata salvata con `SAVE_CONFIG` e la stampante è tornata `Ready` senza errori.

## PHOENIX_START corrente

Il Machine Start usato da OrcaSlicer è:

```gcode
PHOENIX_START EXTRUDER=<temp> BED=<temp> LAYER=<layer> FILAMENT=<type>
```

Il percorso verificato oggi esegue:

1. bed alla temperatura richiesta;
2. nozzle temporaneamente a `150 °C`;
3. `G28`;
4. `CLEAN_NOZZLE`;
5. nozzle heater off;
6. part fan 100%;
7. soak termico gestito dallo stato Phoenix;
8. attesa esplicita nozzle `<= 50 °C`;
9. `QUAD_GANTRY_LEVEL`;
10. `G28 Z`;
11. adaptive mesh con:

```gcode
BED_MESH_CALIBRATE_BASE METHOD=rapid_scan ADAPTIVE=1 ADAPTIVE_MARGIN=10
```

12. fan off;
13. riscaldamento nozzle alla temperatura finale;
14. `LINE_PURGE`;
15. stampa.

Quindi il print-start **non usa la mesh `default` come mesh operativa della singola stampa**: genera una adaptive rapid mesh nuova nelle condizioni termiche di start. Il profilo `default` resta un riferimento stabile e verificato.

## PLA Polymaker PolyTerra — calibrazione 2026-08-21

### Temperature tower

Tower generata da `230 °C` a `190 °C`, step `5 °C`.

Il G-code è partito con:

```gcode
PHOENIX_START EXTRUDER=230 BED=70 LAYER=0.2 FILAMENT=PLA
```

Dal secondo layer Orca ha portato il bed a `65 °C`.

Osservazioni:

- 230–225 °C: troppo fluido / più stringing e meno definizione
- 220 °C: già sensibilmente migliore
- 215 °C: molto pulito
- 210 °C: miglior compromesso osservato
- 205 °C: ancora buono
- 200 °C: nessun vantaggio evidente rispetto a 205–210 °C
- a 195 °C la torre si è staccata

Valore base scelto:

```text
Nozzle PolyTerra PLA: 210 °C
```

Per le calibrazioni successive è stato scelto:

```text
Bed primo layer: 65 °C
Bed layer successivi: 60 °C
```

### Flow Ratio

Test Orca Flow Rate Pass 1 / Coarse.

Parametri di start verificati:

```gcode
PHOENIX_START EXTRUDER=210 BED=65 LAYER=0.2 FILAMENT=PLA
```

Bed poi a `60 °C`.

Il campione `0` è risultato visivamente il migliore / più equilibrato. Poiché `0` corrisponde al valore base del profilo, è stato mantenuto:

```text
Flow Ratio = 1.0465
```

Non è stato ritenuto necessario un Pass 2 fine.

### Pressure Advance

PA Tower DDE, range `0 -> 0.1`, step `0.002`.

Il valore precedente del profilo era `0.034`.

La torre ha confermato che la zona migliore è circa `0.030–0.040`; sopra questa fascia lo seam diventa progressivamente più marcato / seghettato.

Valore mantenuto:

```text
Pressure Advance = 0.034
```

Non è stata ritenuta necessaria una seconda torre fine.

### Adaptive mesh osservate durante le calibrazioni PLA

Temperature tower:

- QGL range `0.012174 mm`
- adaptive mesh 4x3
- max `+0.031 mm`
- min `-0.002 mm`
- range `0.033 mm`

Flow Ratio:

- QGL range `0.014237 mm`
- adaptive mesh 7x6
- max `+0.035 mm`
- min `-0.026 mm`
- range `0.061 mm`

Pressure Advance:

- QGL range `0.010185 mm`
- adaptive mesh 5x5
- max `+0.030 mm`
- min `-0.021 mm`
- range `0.051 mm`

In tutte e tre le stampe la compensazione è risultata regolare e la macchina ha stampato senza errori Eddy.

## Max Volumetric Speed — test in corso a fine sessione

Impostazione Orca:

- start `5 mm³/s`
- end `25 mm³/s`
- step `0.5 mm³/s`

G-code verificato:

```gcode
PHOENIX_START EXTRUDER=210 BED=65 LAYER=0.2 FILAMENT=PLA
SET_PRESSURE_ADVANCE ADVANCE=0.034
```

Bed poi a `60 °C`.

Nota: durante questo test Orca imposta internamente `filament_max_volumetric_speed = 200` per non limitare artificialmente la calibrazione; non è il valore operativo da usare nel profilo finale.

Start reale del test:

- soak residuo `0.2 min @ 65 °C`
- QGL range `0.022225 mm`
- adaptive mesh 10x6
- max `+0.036 mm`
- min `-0.044 mm`
- range `0.079 mm`

Durante il brim è stato percepito un leggero rumore di sfregamento e l'adesione del brim non è apparsa perfetta. La stampa, arrivata comunque al quinto layer e ancora in corso, è stata lasciata proseguire: non è stato aperto un nuovo ciclo di debug per un brim isolato a fine giornata.

Il valore finale di Max Volumetric Speed resta quindi **da determinare alla ripresa** in base al punto reale di degrado della parete, lasciando poi margine operativo.

## Stato sintetico a fine 2026-08-21

La parte macchina / Eddy oggi ha mostrato:

- QGL ripetibile;
- rapid scan validato;
- mesh termiche ripetibili;
- adaptive mesh correttamente generata dal `PHOENIX_START`;
- nessun errore Eddy durante le calibrazioni PLA;
- Z Eddy ricalibrato;
- DKEU wrapper mesh corretto;
- PolyTerra PLA già centrato per temperatura, flow ratio e pressure advance.

Valori PolyTerra attualmente scelti:

```text
Nozzle: 210 °C
Bed first layer: 65 °C
Bed other layers: 60 °C
Flow Ratio: 1.0465
Pressure Advance: 0.034
Max Volumetric Speed: da completare
```

## Cose da NON cambiare senza una nuova evidenza concreta

- curve Eddy
- `reg_drive_current`
- patch Klipper sopra elencate
- viti bed attuali
- parametri protetti extruder:

```text
rotation_distance = 40
gear_ratio = 80:12
microsteps = 16
```

Non usare `M84` durante il workflow Phoenix e non usare `Z_OFFSET_APPLY_PROBE`.

## Ripresa prevista

Alla prossima sessione:

1. valutare il Max Volumetric Speed test completato o fallito;
2. fissare il limite volumetrico operativo PolyTerra con margine;
3. verificare un primo layer / pezzo reale con il profilo PLA risultante;
4. evitare ulteriori campagne di debug su Eddy/DKEU in assenza di un errore concreto.
