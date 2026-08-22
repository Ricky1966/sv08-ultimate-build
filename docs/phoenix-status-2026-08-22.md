# Phoenix — stato tecnico 2026-08-22

Questa nota aggiorna lo stato reale della Sovol SV08 **Phoenix** alla sessione del 22 agosto 2026, in continuità con `docs/phoenix-status-2026-08-21.md`.

## Obiettivo della sessione

Chiudere la calibrazione PLA PolyTerra, verificare il primo layer su zone diverse del piatto e migliorare la logica di heat soak senza riaprire debug già chiusi su Eddy, QGL, mesh o viti.

## PolyTerra PLA — Max Volumetric Speed chiuso

Test Orca:

- start `5 mm³/s`
- end `25 mm³/s`
- step `0.5 mm³/s`
- nozzle `210 °C`
- bed `65 °C` primo layer / `60 °C` successivi
- PA `0.034`

Il pezzo è arrivato fino a fine test senza un punto evidente di collasso della portata: nessuna sottoestrusione netta crescente, pareti ancora regolari nella zona alta.

Conclusione: il limite reale del sistema PolyTerra + Zero Extruder + hotend attuale è almeno nell'ordine di `25 mm³/s` nelle condizioni testate.

Valore operativo scelto con margine:

```text
Max Volumetric Speed = 22 mm³/s
```

Il profilo precedente era `24 mm³/s`; è stato abbassato a `22 mm³/s` per privilegiare affidabilità.

## Baseline PolyTerra PLA corrente

```text
Nozzle: 210 °C
Bed first layer: 65 °C
Bed other layers: 60 °C
Flow Ratio: 1.0465
Pressure Advance: 0.034
Max Volumetric Speed: 22 mm³/s
```

Resta da eseguire un test retrazione dedicato: durante alcune stampe utili sono stati osservati piccoli fili negli spostamenti.

## Verifica primo layer a 5 zone

È stato preparato un test con cinque aree: centro + quattro angoli, un solo layer da `0.20 mm`, con brim esterno `5 mm`, gap `0`.

Start G-code verificato:

```gcode
PHOENIX_START EXTRUDER=210 BED=65 LAYER=0.2 FILAMENT=PLA
SET_PRESSURE_ADVANCE ADVANCE=0.034
```

Stato termico prima del test: macchina accesa da oltre un'ora.

QGL:

```text
Retries: 2/5
Probed points range: 0.017527 mm
tolerance: 0.050000
```

Adaptive mesh:

```text
size: 15x15
max: +0.061 mm
min: -0.044 mm
range: 0.105 mm
```

La mesh è coerente con le precedenti misure termiche; max e min non sono localizzati sulle zone viti, quindi non è stato aperto alcun intervento sulle viti.

### Lettura visiva del primo layer

Dalle foto:

- **FR**: troppo schiacciato, con creste/materiale trascinato;
- **FL**: troppo schiacciato, ancora più evidente;
- **centro**: buono;
- **RR**: quasi corretto;
- **RL**: meno schiacciato / linee più separate.

Conclusione: il problema non appare compatibile con un semplice Z offset globale. Alzare o abbassare globalmente lo Z migliorerebbe alcune zone peggiorandone altre.

## Ipotesi superficie stampata sul PEI

Le zone anteriori FL/FR includono grafica/serigrafie bianche stampate sul foglio PEI. È stata formulata un'ipotesi concreta: Eddy misura il substrato conduttivo sottostante, mentre un eventuale spessore superficiale della grafica non viene letto come quota reale della superficie di stampa.

Questo potrebbe spiegare un primo layer localmente più schiacciato pur con mesh formalmente corretta.

L'ipotesi resta da verificare; non è stata fatta alcuna modifica alla mesh o allo Z per inseguirla.

## Conferma zona centrale su stampe reali

Una prima stampa utile, centrata sul piatto, ha mostrato una faccia inferiore / primo layer visivamente uniforme e ben aderente.

Una seconda stampa utile, più larga ma sempre centrata, era ancora in corso al momento di questa nota; la faccia inferiore sarà valutata a stampa terminata capovolgendo il pezzo.

Queste osservazioni rafforzano l'idea che il problema di primo layer sia locale e non un errore globale di Z.

## Compensazione mesh — verifica

Durante una stampa centrale con adaptive mesh `5x5`:

```text
QGL range: 0.028827 mm
mesh max: +0.031 mm
mesh min: -0.020 mm
mesh range: 0.051 mm
```

`GET_POSITION` ha mostrato:

```text
toolhead Z: 1.500000
gcode Z:    1.515444
```

Quindi la compensazione mesh è effettivamente attiva; non siamo nel caso di una mesh generata ma non applicata.

## Revisione logica heat soak

La logica precedente in `phoenix-print-start-end.cfg` usava:

```text
soak_valid
soak_temp
soak_minutes
cooldown_seconds
```

Il credito soak veniva invalidato dopo `600 s` di bed spento, indipendentemente dalla temperatura reale raggiunta.

### Analisi `klippy.log`

Il log registra normalmente gli `Stats` con:

```text
heater_bed: target=<...> temp=<...> pwm=<...>
```

Sequenza reale osservata nella sessione:

```text
216.9   target=70 temp=71.4
3689.9  target=0  temp=69.9
5878.9  target=65 temp=38.3
7233.8  target=0  temp=65.0
7250.9  target=65 temp=64.3
8346.4  target=60 temp=65.1
```

Questa sequenza dimostra che due riaccensioni non sono equivalenti:

- bed spento ~17 s e ancora a `64.3 °C`: nessun motivo per rifare un soak completo;
- bed raffreddato fino a `38.3 °C`: il credito termico è realmente perso.

È stato verificato che `klippy.log` è quasi continuo durante l'uso attivo, ma può avere buchi significativi durante periodi idle/cooldown. Un caso reale:

```text
prima: Stats 4590.6, target=0, temp=50.0
poi:   Stats 5878.9, target=65, temp=38.3
buco:  ~1288.3 s
```

Pertanto il log non è stato scelto come unica fonte runtime del credito termico.

## Nuova logica thermal credit — file già patchato, non ancora caricato

Backup creato prima della modifica:

```text
/home/biqu/printer_data/config/phoenix-print-start-end.cfg.before-thermal-history-20260822
```

La patch sostituisce `cooldown_seconds` con:

```text
thermal_credit_seconds
```

Principio:

- dopo un soak completo: credito `600 s`;
- riferimento: `soak_temp`;
- soglia termica: `soak_temp - 6 °C`;
- se la temperatura reale scende sotto la soglia: credito azzerato;
- se il bed è acceso, è tornato sopra soglia e il credito è <600: il watcher riaccumula credito a blocchi di 10 s;
- `PHOENIX_START` aspetta `600 - thermal_credit_seconds`.

Esempi attesi per PLA `65 °C`:

```text
65 -> OFF -> 64.3 °C per pochi secondi
=> credito resta praticamente pieno

65 -> OFF -> temperatura sotto 59 °C
=> credito azzerato

riaccensione e permanenza sopra soglia per 7 minuti
=> credito ~420 s, soak residuo ~180 s
```

La normale transizione del profilo PolyTerra `65 -> 60 °C` resta sopra la soglia `59 °C`, quindi non annulla il credito.

### Parte nozzle lasciata invariata

Resta separata e intatta la protezione già validata:

```gcode
RESPOND TYPE=echo MSG="Phoenix Thermal State: attendo nozzle <= 50C prima di QGL/mesh"
TEMPERATURE_WAIT SENSOR=extruder MAXIMUM=50
```

Nessuna modifica a:

- Eddy;
- QGL;
- adaptive rapid mesh;
- CLEAN_NOZZLE;
- temperature finali;
- LINE_PURGE.

### Stato patch al momento della nota

La modifica è presente su disco ma **non è ancora caricata da Klipper**, perché una stampa è in corso. È stato deliberatamente evitato qualsiasi `RESTART` durante la stampa.

Dopo la fine della stampa:

1. `RESTART` da Mainsail;
2. verificare ritorno `Ready`;
3. test controllato della nuova logica thermal credit.

## Prossimi passi

1. terminare la stampa utile attualmente in corso;
2. capovolgere il pezzo e valutare la faccia inferiore / primo layer;
3. fare `RESTART` da Mainsail per caricare la nuova logica heat soak;
4. testare il thermal credit in modo controllato;
5. eseguire calibrazione retrazione PolyTerra PLA per ridurre i piccoli fili osservati;
6. riprendere l'analisi locale del primo layer solo se ancora necessario, senza modificare Z globale, viti o Eddy in assenza di nuova evidenza.

## Regole operative ancora valide

Non modificare senza nuova evidenza concreta:

- curve Eddy;
- `reg_drive_current`;
- patch Klipper correnti;
- viti bed;
- parametri protetti extruder:

```text
rotation_distance = 40
gear_ratio = 80:12
microsteps = 16
```

Non usare `M84` e non usare `Z_OFFSET_APPLY_PROBE`.
