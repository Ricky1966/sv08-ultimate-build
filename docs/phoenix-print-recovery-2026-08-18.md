# Phoenix — recovery stampa, mesh Eddy e macro custom

Data sessione: **2026-08-18**

Questo documento prosegue `docs/zero-toolhead-eddy-2026-08-17.md` e registra il lavoro svolto il 18 agosto per riportare Phoenix a una stampa reale affidabile con Klipper Mainline, Sovol Zero, Eddy current, piatto in grafite e macro Phoenix custom.

## Obiettivo operativo

Portare Phoenix a stampa affidabile con:

- Klipper Mainline;
- Sovol Zero toolhead;
- Eddy current integrato nella Zero;
- macro `PHOENIX_START` / `PHOENIX_END` al posto di `DEMON_START` / `DEMON_END`;
- mesh point-by-point/adaptive;
- primo layer stabile;
- eliminazione progressiva, non distruttiva, delle dipendenze Demon.

Principio seguito: privilegiare una macchina che stampa bene rispetto all'inseguimento di perfezionismi numerici non necessari.

## Hardware / piattaforma

- Sovol SV08 “Phoenix”
- BTT CB1
- Klipper Mainline `v0.13.0-718-gd8659974-dirty`
- Sovol Zero Extruder Kit
- BTT U2C V2.1
- Zero CAN UUID: `eedecd99868b`
- piatto R3men in grafite
- heater AC 1000 W + SSR Omron
- centro macchina usato: `X191 Y165`

## Eddy current — configurazione stabile

File:

`/home/biqu/printer_data/config/phoenix-zero-toolhead.cfg`

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

Stato confermato:

- `7 MHz` / divider 3 stabile;
- divider 2/default aveva causato `RAW_RANGE` durante `G28 Z`;
- nessuna modifica ulteriore a questi valori è stata ritenuta necessaria.

Patch Mainline già attive dal 17 agosto:

`/home/biqu/klipper/klippy/extras/probe_eddy_current.py`

```python
samp_dist = 0.200
```

al posto di `0.040`.

Backup:

`/home/biqu/klipper/klippy/extras/probe_eddy_current-before-zero-compat-20260817.py`

`/home/biqu/klipper/klippy/extras/ldc1612.py`

```python
self.data_rate = 250
```

al posto di `400` Hz.

Backup:

`/home/biqu/klipper/klippy/extras/ldc1612-before-zero-250hz-20260817.py`

Il passaggio a 250 Hz ha eliminato i timeout di comunicazione osservati durante probing/QGL con Eddy sulla Zero.

## Curva Eddy e offset Z

La curva Eddy valida è stata ripristinata da un backup sicuro:

`/home/biqu/printer_data/config/printer-20260818_130714.cfg`

Decisione importante:

- non usare `Z_OFFSET_APPLY_PROBE`;
- non incorporare l'offset operativo nella curva Eddy;
- mantenere la curva separata dall'eventuale regolazione di primo layer.

Offset operativo scelto e applicato dalla macro:

```gcode
SET_GCODE_OFFSET Z=-0.45 MOVE=1
```

Con `Z=-0.45` il primo layer reale è riuscito ed è risultato globalmente uniforme. Le osservazioni fotografiche indicano soltanto un possibile leggero eccesso di schiacciamento in alcune zone.

Eventuali prove future previste: `-0.43` e, solo se necessario, `-0.42`, senza cambiare contemporaneamente il flow.

## Bed mesh — abbandono del rapid scan

Il rapid scan con Eddy/Zero ha mostrato risultati non affidabili e warning `Amplitude Low/High`.

Decisione operativa del 18 agosto:

- non usare più `rapid_scan` per la validazione/stampa corrente;
- usare probing point-by-point;
- mantenere adaptive mesh per la stampa.

Configurazione `[bed_mesh]` attuale:

```ini
speed: 200
horizontal_move_z: 1.5
algorithm: bicubic
mesh_min: 13,15
mesh_max: 328,340
probe_count: 15,15
fade_start: 0
fade_end: 10
scan_overshoot: 1
zero_reference_position: 175,175
```

### Correzione `mesh_max X`

Il precedente `mesh_max X=333`, combinato con `x_offset=-19.8`, portava la toolhead a circa `X352.8`, troppo vicina/oltre il limite utile.

È stato quindi ridotto a:

```ini
mesh_max: 328,340
```

che porta la toolhead a circa `X347.8`.

Backup:

`/home/biqu/printer_data/config/printer-before-bedmesh-xmax328-20260818.cfg`

## QGL e lettura della planarità

QGL recente valido:

- range `0.011627 mm`;
- tolerance `0.050000`.

Il risultato è considerato molto buono e non richiede ulteriori regolazioni meccaniche del gantry.

Mesh centrale point-by-point di riferimento:

```gcode
BED_MESH_CALIBRATE_BASE MESH_MIN=90,90 MESH_MAX=260,260 PROBE_COUNT=8,8
```

Range osservato circa `0.136 mm`, con forma regolare e plausibile.

Le mesh full-bed hanno invece mostrato forti effetti bordo. Decisione: non usare le full-bed come motivo per inseguire ulteriormente le viti del piatto o modificare QGL.

## Primo layer reale

La prima stampa di validazione con il nuovo percorso Phoenix è partita correttamente.

Risultato:

- `PHOENIX_START` eseguito;
- nessuno shutdown Demon dopo la disattivazione del watcher;
- QGL + adaptive mesh + `Z=-0.45` coerenti;
- primo layer panoramico molto uniforme;
- dettagli casuali mostrano linee fuse e continue;
- nessuna grande zona manifestamente troppo alta o troppo bassa;
- lieve sospetto di Z un po' basso, da verificare solo con test comparativo dedicato.

La combinazione operativa considerata valida a fine sessione è quindi:

**QGL + adaptive mesh point-by-point + runtime Z offset -0.45**.

## Difetto locale superficie PEI

Era presente una piccola punta/danno fisico dovuto a precedenti crash del nozzle.

La zona è stata leggermente carteggiata con carta molto fine. Nel test successivo il difetto evidente non era più visibile nel primo layer.

## Macro Phoenix custom

File:

`/home/biqu/printer_data/config/phoenix-print-start-end.cfg`

Inclusione in `printer.cfg`:

```ini
[include ./phoenix-print-start-end.cfg]
```

### `PHOENIX_START`

Configurazione a fine sessione:

```ini
[gcode_macro PHOENIX_START]
gcode:
    {% set bed = params.BED|default(70)|float %}
    {% set extruder = params.EXTRUDER|default(200)|float %}

    G90
    M83

    SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET={bed}
    SET_HEATER_TEMPERATURE HEATER=extruder TARGET=150

    TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM={bed - 1}
    TEMPERATURE_WAIT SENSOR=extruder MINIMUM=145

    G28
    QUAD_GANTRY_LEVEL

    CLEAN_NOZZLE
    G28 Z

    BED_MESH_CALIBRATE_BASE ADAPTIVE=1 ADAPTIVE_MARGIN=10

    SET_GCODE_OFFSET Z=-0.45 MOVE=1

    TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM={bed - 1}
    SET_HEATER_TEMPERATURE HEATER=extruder TARGET={extruder}
    TEMPERATURE_WAIT SENSOR=extruder MINIMUM={extruder - 2}

    LINE_PURGE

    G90
    M83
    G92 E0
```

`LINE_PURGE` è la purge adaptive già usata indirettamente dal percorso Demon tramite `_PURGE_LINES`. È stata aggiunta durante l'ultima stampa e resa effettivamente attiva dopo il successivo `FIRMWARE_RESTART`.

### `PHOENIX_END`

```ini
[gcode_macro PHOENIX_END]
gcode:
    M400
    TURN_OFF_HEATERS
    M107

    G91
    G0 Z5 F600
    G90
    G0 X175 Y340 F6000

    BED_MESH_CLEAR

    M220 S100
    M221 S100
    G92 E0
```

Scelta esplicita: `PHOENIX_END` **non usa `M84`**.

Il vecchio Demon END conteneva `M84` e altri reset nascosti; non viene più usato per Phoenix.

## OrcaSlicer

Machine Start G-code impostato sui profili Phoenix:

```gcode
PHOENIX_START EXTRUDER=[nozzle_temperature_initial_layer] BED=[bed_temperature_initial_layer_single] LAYER=[layer_height] FILAMENT=[filament_type]
```

Machine End G-code:

```gcode
; printing object ENDGCODE
PHOENIX_END
```

Aggiornato sui profili nozzle:

- 0.2 mm;
- 0.4 mm;
- 0.6 mm;
- 0.8 mm.

Hook Demon ancora presenti volutamente:

Before layer:

```gcode
G92 E0
```

After layer:

```gcode
SET_PRINT_STATS_INFO CURRENT_LAYER={layer_num + 1}
M117 Layer ...
```

Change filament:

```gcode
_FIL_CHANGE_PARK
```

Change extrusion role:

```gcode
_DEMON_ADAPTIVE_PA TYPE=[extrusion_role]
```

Pause:

```gcode
PAUSE
```

Decisione: non rimuovere tutti gli hook Demon contemporaneamente. La migrazione deve essere progressiva.

## Disattivazione Demon START watcher

Problema osservato:

`DKEU` effettuava emergency shutdown con il messaggio che il Machine Start G-code Demon non era funzionale.

Causa:

`PHOENIX_START` non valorizza `core_vars.slicer_gcode`, quindi `_DEMON_START_WATCHER` interpretava correttamente l'assenza di `DEMON_START` come un errore, pur essendo ormai intenzionale.

File modificato:

`/home/biqu/printer_data/config/Demon_Klipper_Essentials_Unified/demon_core_assets_v2.3.5.cfg`

Backup:

`/home/biqu/printer_data/config/Demon_Klipper_Essentials_Unified/demon_core_assets_v2.3.5-before-disable-start-watcher-20260818.cfg`

Macro resa innocua:

```ini
[delayed_gcode _DEMON_START_WATCHER]
gcode:
    # Disabled for Phoenix custom PHOENIX_START / PHOENIX_END
```

Demon resta installato perché alcune macro sono ancora dipendenze attive, in particolare `CLEAN_NOZZLE`.

## `CLEAN_NOZZLE`

La macro Demon continua a essere usata ed è stata validata fisicamente.

Zona brush osservata:

- Y circa `359`;
- Z circa `2.5`;
- X circa `236 -> 271`;
- ritorno circa Y `360`;
- circa quattro passate.

## Interferenza blocco cleaner durante homing — risolta

Problema:

Durante `G28`, dopo l'homing Y, la toolhead restava troppo arretrata. Il successivo `G28 X` portava il nozzle sopra la spazzolina e poi sulla plastica del blocco cleaner, causando sollevamento e un evidente “salto” della testina.

Causa:

Il vecchio homing eseguiva `G28 X` immediatamente dopo `G28 Y` restando praticamente sul limite posteriore.

Soluzione adottata:

```gcode
G28 Y
G0 Y330 F1200
G4 P2000
M400
G28 X
G0 X330 F1200
```

poi spostamento al centro macchina e homing Z.

Backup:

`/home/biqu/printer_data/config/printer-before-homing-y330-x330-20260818.cfg`

### `homing_override` rilevante

Ramo `G28 X Y`:

```gcode
G28 Y
G0 Y330 F1200
G4 P2000
M400
G28 X
G0 X330 F1200
```

Ramo full `G28`:

```gcode
G28 Y
G0 Y330 F1200
G4 P2000
M400
G28 X
G0 X330 F1200
G90
G0 X191 Y165 F3600
G28 Z
M400
G0 Z2.0 F3600
G4 P1000
M400
M400
G0 Z10 F600
```

Durante la prima patch erano stati rimossi accidentalmente due `{% endif %}`. Sono stati immediatamente ripristinati e verificati prima del riavvio; la configurazione finale ha caricato correttamente.

Dopo `FIRMWARE_RESTART` il full `G28` è stato testato fisicamente ed è risultato **perfetto**, senza più salto sul blocco cleaner.

## Stato finale 18/08/2026 sera

Phoenix è `READY`.

Validato:

- Zero toolhead CAN;
- Eddy current integrato;
- `7 MHz`, divider 3;
- data rate LDC1612 `250 Hz`;
- curva Eddy valida ripristinata;
- runtime Z offset `-0.45`;
- QGL molto buono (`0.011627 mm` recente);
- point-by-point bed mesh;
- adaptive mesh in `PHOENIX_START`;
- primo layer reale uniforme;
- `PHOENIX_START` / `PHOENIX_END` al posto di Demon start/end;
- Demon start watcher disattivato;
- `CLEAN_NOZZLE` mantenuto;
- nuovo homing Y330/X330 validato fisicamente;
- `LINE_PURGE` caricata dopo l'ultimo `FIRMWARE_RESTART`;
- nessun uso di `M84` nel nuovo end macro.

## Prossime verifiche

1. stampa reale breve (Benchy scelta come test);
2. verificare `LINE_PURGE`, adaptive mesh e nuovo homing nello start reale;
3. giudicare il primo layer e la leggibilità della scritta sul fondo della Benchy;
4. tenere conto del soak termico del piatto in grafite prima di giudicare Z/mesh;
5. solo se confermato eccesso di schiacciamento, confrontare `Z=-0.45` con `-0.43` e poi eventualmente `-0.42`;
6. rimuovere ulteriori dipendenze Demon una alla volta, non in blocco;
7. documentare eventuali ulteriori modifiche e mantenere allineato il repository locale.

## Regole di sicurezza / manutenzione consolidate

- mai usare `M84` nel percorso Phoenix;
- preferire `FIRMWARE_RESTART` a `RESTART`;
- backup prima di ogni modifica a file esistente;
- non usare `Z_OFFSET_APPLY_PROBE`;
- non cambiare la curva Eddy senza analisi precisa;
- non riattivare `rapid_scan` nello stato attuale;
- non modificare i valori protetti degli stepper: `rotation_distance=40`, `gear_ratio=80:12`, `microsteps=16`.
