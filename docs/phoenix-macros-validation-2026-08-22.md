# Phoenix Macros — validazione funzionale del 22 agosto 2026

## Scopo

Questa pagina registra la validazione fisica progressiva delle Phoenix Macros sulla Sovol SV08 “Phoenix”.

La validazione funzionale è distinta dall'audit strutturale: una macro caricata ed esposta correttamente da Klipper non viene considerata fisicamente validata finché non viene eseguita sulla macchina e verificata nel comportamento reale.

## Stato iniziale

Prima dei test funzionali risultavano già verificati:

- nessun include DKEU attivo;
- nessuna dipendenza runtime DKEU nei file `phoenix-*.cfg`;
- Klipper attivo senza errori Config/Jinja/runtime al restart;
- 22 macro `PHOENIX_*` definite ed esposte;
- `PHOENIX_CLEAN_NOZZLE` già validata fisicamente.

## Test completati

### `PHOENIX_PRINTER_STATUS` — PASS fisico

Eseguita a macchina idle e non homed.

Output verificato:

- `Print state: standby`;
- `Idle state: Idle`;
- assi homed vuoti;
- posizione riportata correttamente;
- temperature extruder e bed lette correttamente;
- target heater a 0 °C;
- Virtual SD non attiva.

Nessun movimento, nessuna modifica di stato, nessun errore Jinja/runtime.

### `PHOENIX_SYSTEM_SENSORS` — PASS fisico

Sensori esposti correttamente:

- `temperature_sensor Toolhead_Temp`;
- `temperature_sensor mcu_temp`;
- `temperature_sensor Host_temp`;
- `heater_bed`;
- `extruder`.

Heater esposti correttamente:

- `heater_bed`;
- `extruder`.

Nessun movimento e nessun errore.

### `PHOENIX_PROBE_ACCURACY` — PASS fisico

Test eseguito con i parametri default, quindi senza riscaldamento del bed e senza soak.

La macro ha completato homing, posizionamento centrale, 20 campioni nativi `PROBE_ACCURACY` e park finale.

Risultato:

```text
maximum: -0.012036 mm
minimum: -0.018744 mm
range: 0.006708 mm
average: -0.014307 mm
median: -0.013696 mm
standard deviation: 0.001932 mm
samples: 20
```

Coordinate probe effettive riportate da Klipper: `X155.200 Y174.250`, coerenti con gli offset del sensore rispetto al toolhead posizionato a `X175 Y175`.

Nessun errore.

### `PHOENIX_STEPPER_BUZZ` — PASS fisico completo

Testati singolarmente e verificati visivamente tutti i sei stepper previsti dalla macro:

- `STEPPER=X` — PASS;
- `STEPPER=Y` — PASS;
- `STEPPER=Z` — PASS;
- `STEPPER=Z1` — PASS;
- `STEPPER=Z2` — PASS;
- `STEPPER=Z3` — PASS.

Per ogni selezione è stato osservato fisicamente il movimento previsto ed è stato verificato il completamento della macro senza errori.

### `PHOENIX_MACHINE_LEVEL` — PASS fisico dopo correzione del raffreddamento

Durante il primo test è emerso un difetto funzionale nella sequenza termica: dopo `PHOENIX_CLEAN_NOZZLE`, la macro spegneva l'extruder e attendeva `<= 50C`, ma non attivava il part fan. L'attesa risultava quindi inutilmente lunga.

La macro è stata corretta in `phoenix-calibration.cfg` aggiungendo il raffreddamento forzato:

```gcode
M106 S255
TEMPERATURE_WAIT SENSOR=extruder MAXIMUM=50
M107
```

La prima validazione era stata eseguita con `M106 S155`; successivamente il valore è stato portato a `S255` per ridurre il tempo di raffreddamento. Il comando fan era già stato verificato fisicamente e `M107` spegne correttamente la ventola.

Dopo il reload della configurazione, `PHOENIX_MACHINE_LEVEL` ha completato l'intero workflow:

1. `BED_MESH_CLEAR`;
2. homing;
3. `PHOENIX_CLEAN_NOZZLE`;
4. extruder target 0;
5. raffreddamento forzato fino a `<= 50C`;
6. spegnimento part fan;
7. `QUAD_GANTRY_LEVEL`;
8. `G28 Z`;
9. park `X175 Y175 Z25`;
10. messaggio `Phoenix Machine Level: complete`.

QGL finale:

```text
Retries: 2/5
Probed points range: 0.037489 mm
tolerance: 0.050000 mm
```

Il primo ciclo QGL aveva mostrato un range elevato (`0.643945 mm`), ma la procedura nativa ha corretto il gantry e ha convergito entro la tolleranza prevista al secondo retry. Questo non è stato classificato come errore della macro.

La macro è quindi classificata **TESTATA FISICAMENTE — PASS**.

### `PHOENIX_CLEANER_SETUP` — PASS fisico

Test eseguito con i parametri default:

```text
X236 Y359 Z2.5
SAFE_Z=10
```

La macro ha raggiunto correttamente la posizione del nozzle cleaner. La posizione è stata verificata visivamente sulla macchina come corretta, senza urti o anomalie. La macro ha inoltre confermato esplicitamente che nessun valore è stato salvato.

Output finale verificato:

```text
Phoenix Cleaner Setup: target X236.0 Y359.0 Z2.5
Phoenix Cleaner Setup: inspect position manually - no values were saved
```

La macro è quindi classificata **TESTATA FISICAMENTE — PASS**.

### `PHOENIX_READY_UP` — PASS fisico dopo correzione temperatura cleaner

Il primo test è stato eseguito con:

```text
PHOENIX_READY_UP BED=0 NOZZLE=160
```

Durante il test è emersa una incoerenza funzionale: `PHOENIX_READY_UP` richiedeva `NOZZLE=160`, ma `PHOENIX_CLEAN_NOZZLE` in uso standalone portava sempre l'ugello a 170 °C prima di restituire il controllo alla macro chiamante.

La correzione ha reso parametrica la temperatura standalone del cleaner mantenendo 170 °C come default:

```text
PHOENIX_CLEAN_NOZZLE TEMP=<temperatura>
```

`PHOENIX_READY_UP` passa ora esplicitamente il proprio target:

```gcode
PHOENIX_CLEAN_NOZZLE TEMP={nozzle}
```

Il retest con `BED=0 NOZZLE=160` ha mostrato:

```text
Phoenix Ready Up: bed 0C, nozzle 160C
Phoenix Cleaner: heating nozzle to 160C
Phoenix Cleaner: cleaning nozzle
```

La pulizia è stata eseguita correttamente e la testina è tornata al centro come previsto. Il target richiesto non viene più superato dal cleaner.

La macro è quindi classificata **TESTATA FISICAMENTE — PASS**.

### `PHOENIX_PRESENT_TOOLHEAD` — PASS fisico

La macro è stata eseguita in standby e ha portato correttamente la testina nella posizione di manutenzione calcolata dalla corsa configurata.

Posizione finale verificata fisicamente:

```text
X175
Y25
Z173.5
```

La posizione è risultata corretta e comodamente accessibile per manutenzione. Nessun urto, nessun heater coinvolto, nessuna modifica persistente.

La macro è quindi classificata **TESTATA FISICAMENTE — PASS**.

### `PHOENIX_PID_TUNE HEATER=BED TARGET=60` — PASS fisico

La calibrazione PID del bed ha completato correttamente il ciclo a 60 °C.

Risultato proposto da Klipper:

```text
pid_Kp=41.094
pid_Ki=0.719
pid_Kd=587.127
```

La macro ha emesso correttamente:

```text
Phoenix PID complete - review result, then SAVE_CONFIG manually if accepted
```

Non è stato eseguito `SAVE_CONFIG`; i valori non sono stati persistiti.

Il ramo BED della macro è quindi classificato **TESTATO FISICAMENTE — PASS**. Il ramo EXTRUDER resta intenzionalmente rinviato alla terzultima posizione della sequenza di test.

### `PHOENIX_IDLE_TIMEOUT` — ramo standby PASS fisico

Per verificare realmente lo spegnimento degli heater è stato impostato il bed a target 40 °C e poi eseguita la macro in stato `standby`.

Output:

```text
Phoenix Idle: heaters off
```

Il target del bed è tornato a 0 °C. Nessun `M84` viene utilizzato.

Il ramo `standby` è quindi **TESTATO FISICAMENTE — PASS**. Il ramo `paused` resta da verificare durante un successivo test che porti realmente la stampante in pausa.

### `PHOENIX_REFERENCE_BED_MESH` — PASS fisico dopo due correzioni

Il primo tentativo ha evidenziato l'assenza di raffreddamento forzato dopo `PHOENIX_CLEAN_NOZZLE`, come già osservato in `PHOENIX_MACHINE_LEVEL`. È stato aggiunto:

```gcode
M106 S255
TEMPERATURE_WAIT SENSOR=extruder MAXIMUM=50
M107
```

Il secondo tentativo ha evidenziato un residuo del vecchio wrapper DKEU:

```text
Unknown command:"BED_MESH_CALIBRATE_BASE"
```

La macro non aveva quindi eseguito alcuna mesh e il successivo salvataggio del profilo falliva.

L'audit immediato ha mostrato due residue chiamate `_BASE` nelle Phoenix Macros:

```text
phoenix-debug.cfg
phoenix-print-start-end.cfg
```

Entrambe sono state corrette per usare il comando nativo Klipper Mainline:

```gcode
BED_MESH_CALIBRATE METHOD=rapid_scan
BED_MESH_CALIBRATE METHOD=rapid_scan ADAPTIVE=1 ADAPTIVE_MARGIN=10
```

Il retest finale di `PHOENIX_REFERENCE_BED_MESH` ha completato correttamente:

- pulizia nozzle;
- raffreddamento forzato a ventola piena fino a `<=50 °C`;
- QGL;
- `G28 Z`;
- full rapid scan;
- creazione della mesh;
- salvataggio del profilo `phoenix_reference` nella sessione.

QGL finale:

```text
Retries: 2/5
Probed points range: 0.015904 mm
tolerance: 0.050000 mm
```

Output finale:

```text
Mesh Bed Leveling Complete
Bed Mesh state has been saved to profile [phoenix_reference]
Phoenix Reference Mesh 'phoenix_reference' complete - SAVE_CONFIG manually to persist
```

Mainsail ha mostrato il profilo `phoenix_reference` con mesh 15x15 e range di circa 0.133 mm. È comparso anche il profilo `default` della mesh corrente nella sessione; non è stato eseguito `SAVE_CONFIG`, quindi nessun profilo è stato persistito nel file di configurazione.

La macro è quindi classificata **TESTATA FISICAMENTE — PASS**.

La stessa correzione del comando nativo ha eliminato preventivamente un errore che avrebbe colpito anche `PHOENIX_START` durante la fase di bed mesh adaptive.

## Stato della validazione dopo questa sessione

Macro fisicamente validate:

- `PHOENIX_CLEAN_NOZZLE`;
- `PHOENIX_PRINTER_STATUS`;
- `PHOENIX_SYSTEM_SENSORS`;
- `PHOENIX_PROBE_ACCURACY`;
- `PHOENIX_STEPPER_BUZZ` — X, Y, Z, Z1, Z2, Z3;
- `PHOENIX_MACHINE_LEVEL`;
- `PHOENIX_CLEANER_SETUP`;
- `PHOENIX_READY_UP`;
- `PHOENIX_PRESENT_TOOLHEAD`;
- `PHOENIX_PID_TUNE` — ramo BED;
- `PHOENIX_IDLE_TIMEOUT` — ramo standby;
- `PHOENIX_REFERENCE_BED_MESH`.

Le altre Phoenix Macros restano strutturalmente caricate/esposte ma non devono essere dichiarate fisicamente validate finché non vengono provate singolarmente sulla macchina.

Ordine finale riservato della sequenza corrente:

- terzultimo: `PHOENIX_PID_TUNE HEATER=EXTRUDER`;
- penultimo: `PHOENIX_LOAD_FILAMENT`;
- ultimo: `PHOENIX_UNLOAD_FILAMENT`.

## Nota operativa

Il repository locale non è stato riallineato in questa fase, in conformità alla procedura di lavoro corrente.

## Aggiornamento del 23 agosto 2026

Questa sezione integra la fotografia storica del 22 agosto senza modificarne i risultati originali.

Il conteggio di **22 macro `PHOENIX_*`** riportato sopra è corretto per lo stato del runtime del 22 agosto 2026.

Il 23 agosto 2026:

- `PHOENIX_PRESSURE_ADVANCE_TEST` è stata rimossa intenzionalmente perché ridondante rispetto ai workflow di calibrazione Pressure Advance disponibili nello slicer;
- il runtime Phoenix corrente è quindi composto da **21 macro `PHOENIX_*`**;
- `PHOENIX_UNLOAD_FILAMENT` è stata validata fisicamente con esito PASS;
- `PHOENIX_LOAD_FILAMENT` è stata validata fisicamente con esito PASS;
- `PHOENIX_FILAMENT_RUNOUT` è stata validata durante una stampa reale: il sensore ha riportato `filament not detected`, la stampa è entrata correttamente in pausa e la testina è stata parcheggiata a X100 Y10 Z50;
- `PHOENIX_FILAMENT_CHANGE` è stata validata durante una stampa reale: fuori dallo stato `printing` la macro ha correttamente rifiutato l'esecuzione; durante la stampa ha richiesto il cambio filamento, eseguito pausa e park a X100 Y10 Z50, e la stampa è poi ripresa correttamente con `RESUME`.

La fase di validazione funzionale del blocco filamento può quindi essere considerata completata.

La nota operativa precedente sul repository locale descrive esclusivamente lo stato della sessione del 22 agosto. Il 23 agosto il branch locale `develop` è stato successivamente riallineato e aggiornato in modo controllato con il repository di sviluppo.
