# Sovol SV08 Phoenix — migrazione a Eddy nativo e primo test di stampa

Data: 2026-08-08

Branch di lavoro: `feature/mainline-migration`

## Obiettivo della sessione

La sessione ha chiuso il blocco principale della migrazione della Phoenix da `probe_eddy_ng` al supporto Eddy nativo di Klipper Mainline, ha validato homing, probing, QGL e rapid mesh a caldo e ha portato la macchina fino al primo vero avvio di stampa con Demon Macro.

Il primo layer reale su area molto ampia non è risultato accettabile: la zona anteriore/destra è risultata fortemente schiacciata e trascinata. La stampa è stata fermata e la diagnosi del percorso `DEMON_START`/Z/mesh viene rimandata alla prossima sessione. Non vengono effettuate correzioni meccaniche o modifiche QGL sulla base di questo singolo test.

## Versione Klipper

Versione Mainline osservata:

```text
v0.13.0-718-gd8659974-dirty
```

Commit Klipper in uso:

```text
d865997403cad36d105026f73a4b76dcacec4c76
```

## Perché è stato abbandonato `probe_eddy_ng`

Il vecchio plugin Eddy-NG presentava una dipendenza circolare nel percorso di homing Z con il Klipper Mainline corrente: `G28 Z` entrava nella sessione di probe mentre il plugin richiedeva che Z risultasse già homed.

Il precedente workaround tramite `set_position_z: 0` nell'`[homing_override]` è stato rimosso perché dichiarava falsamente Z come homed.

Dopo la rimozione il problema reale è emerso chiaramente con l'errore:

```text
Z axis must be homed before probing
```

È stato quindi deciso di non continuare a patchare `probe_eddy_ng.py` e di migrare al supporto Eddy nativo incluso nel Klipper installato.

## Configurazione Eddy nativa

Configurazione manuale attiva:

```ini
[probe_eddy_current eddy]
sensor_type: ldc1612
i2c_mcu: extra_mcu
i2c_software_scl_pin: extra_mcu:PB6
i2c_software_sda_pin: extra_mcu:PB7
x_offset: -16.43
y_offset: 10.22
#reg_drive_current: 21
descend_z: 0.5
max_sensor_hz: 9000000
```

La calibrazione drive current nativa ha prodotto:

```text
reg_drive_current: 22
```

salvato tramite `SAVE_CONFIG`.

Il valore `max_sensor_hz: 9000000` è stato aggiunto dopo che il primo `G28 Z` nativo aveva restituito:

```text
Trigger analog error: RAW_RANGE
```

Il log indicava che la frequenza massima osservata durante la calibrazione superava il default nativo di 5 MHz; la curva calibrata arrivava a circa 8.523 MHz.

## Calibrazione Z nativa

È stata completata con successo la calibrazione:

```gcode
PROBE_EDDY_CURRENT_CALIBRATE CHIP=eddy
```

La curva risultante copre circa:

```text
Z: 0.050625 -> 2.090625 mm
frequenza: circa 8.523 -> 8.451 MHz
```

Dopo l'aggiunta di `max_sensor_hz: 9000000`, `G28 Z` è riuscito senza contatto del nozzle con il piatto. Il controllo con carta ha confermato che il nozzle restava libero.

Il parametro `descend_z: 0.5` viene mantenuto come configurazione corrente.

## Homing override

Sono state rimosse dal percorso Z le vecchie chiamate Eddy-NG:

```text
PROBE_EDDY_NG_PROBE_STATIC HOME_Z=1
```

È stato inoltre rimosso:

```ini
set_position_z: 0
```

Nel ramo full-G28 il movimento iniziale `G0 Z5` è stato protetto in modo da essere eseguito soltanto se Z è già homed:

```jinja
{% if 'z' in printer.toolhead.homed_axes %}
  G0 Z5 F300
{% endif %}
```

Un `G28` completo partendo da assi totalmente non homed è stato testato con successo. Posizione finale osservata:

```text
X191 Y165 Z10
```

Backup rilevante:

```text
/home/biqu/printer_data/config/printer.cfg.before-safe-full-g28-20260808-173646.bak
```

## Demon Macro — integrazione Eddy nativa

### `_APPLY_EDDY_Z_OFFSET`

Resta intenzionalmente un no-op:

```ini
[gcode_macro _APPLY_EDDY_Z_OFFSET]
gcode:
    RESPOND TYPE=COMMAND MSG="Eddy Z offset saltato (Klipper Sovol)"
```

### `_PROBE_TAP`

Demon verificava la sola presenza della chiave `tap_threshold`. Il supporto Eddy nativo espone però il valore di default `tap_threshold = 0.0`, quindi il ramo TAP veniva selezionato anche senza una vera calibrazione TAP e falliva con:

```text
Tap not configured
```

La condizione è stata corretta per usare TAP soltanto quando il valore configurato è maggiore di zero:

```jinja
{% if printer.configfile.settings.get('probe_eddy_current btt_eddy', {}).get('tap_threshold', 0)|float > 0 or printer.configfile.settings.get('probe_eddy_current eddy', {}).get('tap_threshold', 0)|float > 0 %}
```

Backup:

```text
/home/biqu/printer_data/config/Demon_Klipper_Essentials_Unified/demon_core_assets_v2.3.5.cfg.before-mainline-tap-threshold-fix-20260808-175823.bak
```

Dopo `FIRMWARE_RESTART`, `_PROBE_TAP` ha completato correttamente usando il probing nativo automatico.

Non è stata eseguita una calibrazione TAP e non va aggiunto un vecchio `tap_threshold: 300`: la semantica del TAP nativo corrente è diversa da quella del vecchio plugin.

## Validazione a freddo

Sono stati verificati:

- `G28` completo;
- `PROBE` al centro;
- `SET_Z_FROM_PROBE`;
- `_PROBE_TAP` nel ramo automatico non-TAP.

Valori `PROBE` centrali osservati a freddo:

```text
0.002734 mm
0.006694 mm
```

Nota operativa: un `PROBE` semplice lascia la toolhead vicino alla quota di trigger, quindi prima di ripetere un probing è opportuno rialzare Z.

## Validazione a caldo PLA

Condizioni di prova:

```text
bed: 65 °C
nozzle: 170 °C
soak: circa 10 minuti
```

Il full homing a caldo è stato completato fisicamente senza problemi.

`PROBE` centrale a caldo:

```text
0.007357 mm
```

La differenza osservata rispetto alle misure a freddo è stata di pochi micron in queste specifiche condizioni. Questo test è sufficiente per proseguire, ma non viene interpretato come prova universale di assenza di drift termico.

## QGL a caldo

Il QGL nativo è stato eseguito senza modificare la configurazione consolidata.

Risultato finale:

```text
Retries: 3/5
Probed points range: 0.018441
tolerance: 0.050000
```

La configurazione QGL resta quindi invariata:

```ini
horizontal_move_z: 3
retry_tolerance: 0.05
retries: 5
max_adjust: 4
```

Non sono richiesti interventi su gantry, motori Z o viti sulla base dei test di questa sessione.

## Rapid bed mesh a caldo

Comando:

```gcode
BED_MESH_CALIBRATE METHOD=rapid_scan
```

Mesh 15 x 15 completata con:

```text
max:   +0.163 mm
min:   -0.178 mm
range:  0.341 mm
```

Il risultato è coerente con la geometria già osservata sul piatto originale nella sessione precedente, dove la mesh finale era circa:

```text
max:   +0.194 mm
min:   -0.160 mm
range:  0.354 mm
```

Non sono emersi spike isolati evidenti. Non è stato eseguito `SAVE_CONFIG` per rendere permanente la mesh di test.

## OrcaSlicer — recupero preset Phoenix

Durante il primo tentativo di stampa Demon ha eseguito un emergency shutdown perché il G-code generato non conteneva il Machine Start atteso.

L'errore non era dovuto a Eddy: OrcaSlicer era rimasto selezionato sulla macchina Sovol invece che sulla macchina Phoenix.

I preset Phoenix non erano stati persi. Sono presenti sia nello snapshot Git sia nella directory utente OrcaSlicer Flatpak.

Snapshot versionato:

```text
config/orcaslicer/machine/Phoenix 0.4 nozzle.json
config/orcaslicer/filament/Polymaker PolyTerra PLA @Phoenix 0.4.json
config/orcaslicer/process/Phoenix 0.20 Strong 30% Gyroid + Support.json
```

Directory utente attiva OrcaSlicer:

```text
~/.var/app/com.orcaslicer.OrcaSlicer/config/OrcaSlicer/user/default/
```

Il log Orca 2.4.2 ha confermato che i preset Phoenix venivano caricati correttamente.

Il profilo macchina `Phoenix 0.4 nozzle` contiene il Machine Start Demon corretto:

```gcode
SET_PRINT_STATS_INFO TOTAL_LAYER=[total_layer_count]
M104 S0
M140 S0
DEMON_START EXTRUDER=[nozzle_temperature_initial_layer] TOOL={initial_tool} BED=[bed_temperature_initial_layer_single] LAYER=[layer_height] FILAMENT=[filament_type] SEQUENCE="[print_sequence]" EXCLUDE=[exclude_object] SURFACE="[curr_bed_type]" OAPA=[adaptive_pressure_advance] DMGCC="v1.4"
_SPS GSTART=True
```

Demon corrente richiede `DMGCC="v1.4"` e `_SPS GSTART=True`; entrambi sono presenti nel preset salvato.

Una volta selezionata `Phoenix 0.4 nozzle` in Orca, i profili Phoenix sono tornati visibili e utilizzabili.

## Profilo PLA

Lo snapshot del profilo operativo contiene:

```text
Polymaker PolyTerra PLA @Phoenix 0.4
flow ratio: 1.00
pressure advance: 0.032
nozzle: 210 °C
bed: 65 °C
first layer bed: 70 °C
max volumetric speed: 22 mm³/s ereditato dalla base
```

Per la prossima sessione è stato deciso di portare la temperatura nozzle del PolyTerra a `200 °C`. Lo snapshot non viene modificato retroattivamente: la variazione va applicata e poi nuovamente versionata quando effettivamente salvata in OrcaSlicer.

## Primo test reale di stampa

È stato preparato in Orca un primo layer molto esteso:

```text
315 x 315 x 0.20 mm
```

senza brim/skirt, con l'obiettivo di osservare la compensazione su gran parte del piatto.

Dopo il recupero del preset Phoenix il percorso di start è riuscito a superare il precedente controllo Demon e la stampa è partita.

Il risultato del primo layer non è stato accettabile. In particolare la zona anteriore/destra ha mostrato:

- filamento estremamente schiacciato;
- materiale trascinato/spalmato dal nozzle;
- forte differenza qualitativa rispetto ad altre zone del perimetro;
- accumulo e strisciate incompatibili con un test da lasciare completare.

La macchina è stata fermata e spenta.

La fotografia del test è stata conservata nella conversazione di lavoro ma non è stata ancora aggiunta al repository.

## Interpretazione provvisoria del fallimento

Il test reale dimostra che la catena completa di stampa non è ancora validata end-to-end.

Non viene attribuita automaticamente la causa a:

- QGL;
- meccanica del gantry;
- viti del piatto;
- cinghie;
- motori Z;
- semplice temperatura PLA 210 °C.

QGL e mesh manuali eseguiti poco prima erano coerenti e stabili. Prima di modificare la meccanica va quindi verificato il comportamento effettivo del percorso `DEMON_START`, con particolare attenzione a:

- homing e riferimento Z realmente usati durante lo start;
- generazione della mesh durante lo start;
- mesh effettivamente attiva al primo layer;
- eventuali offset Z applicati o azzerati dalle macro;
- eventuali differenze tra test manuali e sequenza Demon.

## Stato a fine sessione

### Validato

- Klipper Mainline operativo;
- Eddy nativo configurato e calibrato;
- drive current nativo `22`;
- `max_sensor_hz: 9000000`;
- homing Z nativo;
- full `G28` da assi non homed;
- probing automatico nativo;
- integrazione Demon `_PROBE_TAP` corretta per `tap_threshold = 0`;
- QGL a caldo;
- rapid mesh a caldo;
- preset Orca Phoenix ritrovati e caricati;
- Machine Start Demon v1.4 confermato nel preset Phoenix.

### Non ancora validato

- primo layer reale su area ampia;
- corretta applicazione end-to-end di Z/mesh dentro `DEMON_START`;
- temperatura definitiva PolyTerra a 200 °C;
- calibrazione PLA completa.

## Prossimo passo

Alla ripresa non effettuare nuove calibrazioni o regolazioni meccaniche alla cieca.

Partire dal log della stampa fallita, dal momento di `DEMON_START` fino all'inizio del primo layer, e ricostruire esattamente:

1. homing eseguito;
2. QGL eseguito;
3. probing/riferimento Z;
4. mesh generata e caricata;
5. offset Z applicati;
6. stato immediatamente precedente al primo movimento di estrusione.

Solo dopo questa verifica decidere se sia necessaria una modifica software, una nuova calibrazione Eddy o un intervento meccanico.

## Regole consolidate dopo la sessione

- non tornare al vecchio `probe_eddy_ng` come percorso operativo;
- non continuare a patchare il vecchio plugin per risolvere l'homing Mainline;
- non riaggiungere `set_position_z: 0` nell'homing override;
- non configurare TAP nativo con i vecchi valori Eddy-NG;
- non modificare QGL senza nuove evidenze;
- non modificare motori Z, axis twist o meccanica per inseguire il primo layer fallito prima di analizzare il log di start;
- mantenere il profilo macchina Orca selezionato su `Phoenix 0.4 nozzle` quando si stampa con Demon.
