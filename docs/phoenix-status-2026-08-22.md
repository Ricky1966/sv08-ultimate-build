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
Retraction length: 0.4 mm
Retraction speed: 35 mm/s
```

## Calibrazione retrazione PolyTerra PLA

È stata eseguita una torre retrazione Orca con:

```text
Start: 0.2 mm
End:   1.2 mm
Step:  0.1 mm
Speed: 35 mm/s
```

Il G-code è stato verificato prima della stampa: la progressione copriva correttamente `0.2 -> 1.2 mm` con incrementi di `0.1 mm`.

La torre è risultata molto pulita già nella parte bassa, senza una zona di miglioramento netto che giustificasse retrazioni elevate da `1.0-1.2 mm`. Sono comparsi solo pochi fili sporadici, non una ragnatela sistematica tra i pilastri.

Valore operativo scelto e salvato nel profilo PolyTerra PLA:

```text
Retraction length = 0.4 mm
Retraction speed  = 35 mm/s
```

Non sono stati modificati contemporaneamente temperatura, flow, PA o MVS.

## Verifica primo layer a 5 zone

È stato preparato un test con cinque aree: centro + quattro angoli, un solo layer da `0.20 mm`, con brim esterno `5 mm`, gap `0`.

Start G-code verificato all'epoca del test:

```gcode
PHOENIX_START EXTRUDER=210 BED=65 LAYER=0.2 FILAMENT=PLA
SET_PRESSURE_ADVANCE ADVANCE=0.034
```

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

La prima stampa utile, piccola e centrata, aveva già mostrato una faccia inferiore / primo layer visivamente uniforme e ben aderente.

La seconda stampa utile, molto più larga ma ancora nella zona centrale, è stata poi terminata e fotografata sia sopra sia sulla faccia inferiore. Il vero primo layer è risultato molto buono sull'intera impronta del pezzo:

- texture PEI uniforme;
- linee ben fuse;
- nessun gap evidente;
- nessuna zona strappata dal nozzle;
- nessuna variazione significativa lungo l'estensione del pezzo.

Questo conferma che **non va corretto lo Z globale**. La compensazione centrale funziona bene anche su un oggetto di impronta maggiore; le anomalie FL/FR restano quindi da trattare come fenomeno locale, non come errore generale di offset o mesh.

Le irregolarità viste sulla parte superiore della seconda stampa non sono state usate per correggere Z, flow o PA: sono un problema distinto e richiedono lettura dello slicing/G-code prima di qualsiasi modifica.

## Compensazione mesh — verifica

Durante una stampa centrale con adaptive mesh `5x5`:

```text
QGL range: 0.028827 mm
mesh max: +0.031 mm
mesh min: -0.020 mm
mesh range: 0.051 mm
```

`GET_POSITION` aveva mostrato:

```text
toolhead Z: 1.500000
gcode Z:    1.515444
```

Un'ulteriore verifica del 22 agosto, dopo adaptive mesh `15x15`, ha mostrato:

```text
toolhead Z: 1.620762
gcode Z:    1.600000
```

Anche qui la compensazione è effettivamente attiva. Non siamo nel caso di una mesh generata ma non applicata.

## Mesh/QGL dopo soak completo

Dopo il primo caricamento runtime della nuova logica thermal credit è stato eseguito un soak completo a `65 °C`.

QGL:

```text
Retries: 2/5
Probed points range: 0.015889 mm
tolerance: 0.050000
```

Adaptive rapid mesh:

```text
size: 15x15
max: +0.059 mm
min: -0.047 mm
range: 0.106 mm
```

Il risultato è praticamente sovrapponibile al precedente range `0.105 mm`, quindi la geometria termica del bed resta coerente.

## Thermal credit — analisi e test runtime

La prima revisione sostituiva il vecchio `cooldown_seconds` con `thermal_credit_seconds` e usava una soglia `soak_temp - 6 °C`.

Il primo test runtime ha confermato:

1. dopo `RESTART`, stato iniziale senza credito;
2. `PHOENIX_START BED=65 EXTRUDER=210` -> `soak 10.0 min @ 65.0C`;
3. al termine del soak:

```text
soak_valid: 1
soak_temp: 65.0
soak_minutes: 10
thermal_credit_seconds: 600
```

4. passaggio bed `65 -> 60 °C`: credito rimasto `600`;
5. nella prima implementazione, discesa a `58.6-58.8 °C` (< floor `59.0 °C`) -> credito azzerato;
6. preriscaldamento manuale a `65 °C` -> riaccumulo verificato, per esempio `thermal_credit_seconds: 40` dopo circa 40 s utili;
7. un successivo `PHOENIX_START` ha realmente usato il credito parziale, mostrando `soak 9.0 min @ 65.0C`.

Questi test hanno confermato che il meccanismo di credito era funzionante, ma hanno anche mostrato un difetto concettuale: l'azzeramento istantaneo appena sotto `ref - 6 °C` era troppo aggressivo.

## Thermal credit — logica finale con decadimento graduale

È stato creato il backup:

```text
/home/biqu/printer_data/config/phoenix-print-start-end.cfg.before-thermal-credit-decay-20260822
```

Il watcher è stato modificato in modo che sotto la soglia termica il credito **non venga più azzerato istantaneamente**.

Logica finale:

- watcher ogni `10 s`;
- se `temp >= soak_temp - 6 °C` e heater target > 0, il credito cresce di `10 s` ogni `10 s`, fino al massimo configurato;
- se `temp >= soak_temp - 6 °C` e il credito è già pieno, resta pieno;
- se `temp < soak_temp - 6 °C`, il credito cala di `10 s` ogni `10 s`;
- minimo `0`;
- nessun precipizio artificiale passando, per esempio, da `59.0` a `58.8 °C` con riferimento `65 °C`.

Esempio concettuale:

```text
credito 600 s
bed scende sotto 59 °C per 30 s
=> credito circa 570 s
=> prossima stampa richiede circa 30 s di soak residuo
```

Un raffreddamento prolungato porta invece naturalmente il credito a zero.

## Soak configurabile

Durante la stessa sessione è stato deciso di eliminare il limite fisso di `600 s` dalla logica operativa, mantenendolo solo come default.

Backup creato prima della parametrizzazione:

```text
/home/biqu/printer_data/config/phoenix-print-start-end.cfg.before-configurable-soak-20260822
```

Lo stato runtime contiene ora:

```text
variable_soak_total_seconds: 600
```

`PHOENIX_START` accetta:

```text
SOAK_SECONDS=<secondi>
```

Esempi:

```gcode
PHOENIX_START BED=65 EXTRUDER=210 SOAK_SECONDS=300
PHOENIX_START BED=65 EXTRUDER=210 SOAK_SECONDS=600
PHOENIX_START BED=65 EXTRUDER=210 SOAK_SECONDS=900
```

Il valore scelto governa contemporaneamente:

- durata soak totale;
- credito massimo;
- clamp del credito precedente;
- credito impostato al termine di un soak completato;
- riaccumulo del watcher.

Dopo `RESTART` la configurazione è stata accettata da Klipper e lo stato iniziale verificato:

```text
soak_valid: 0
soak_temp: 0.0
soak_minutes: 0
soak_total_seconds: 600
thermal_credit_seconds: 0
```

Test override reale:

```gcode
PHOENIX_START BED=65 EXTRUDER=210 SOAK_SECONDS=300
```

Risultato console:

```text
Phoenix Thermal State: soak 5.0 min @ 65.0C
```

Quindi l'override configurabile è funzionante.

## Orca — passaggio SOAK_SECONDS

Il Machine start G-code di Orca è stato aggiornato a:

```gcode
PHOENIX_START EXTRUDER=[nozzle_temperature_initial_layer] BED=[bed_temperature_initial_layer_single] LAYER=[layer_height] FILAMENT=[filament_type] SOAK_SECONDS=600
```

Il G-code rigenerato è stato verificato e contiene realmente:

```gcode
PHOENIX_START EXTRUDER=210 BED=65 LAYER=0.2 FILAMENT=PLA SOAK_SECONDS=600
```

In questo modo la durata può essere modificata a `300`, `600`, `900`, ecc. dal profilo Orca senza intervenire sul file Klipper.

## Protezione nozzle invariata

Resta separata e intatta la protezione già validata:

```gcode
RESPOND TYPE=echo MSG="Phoenix Thermal State: attendo nozzle <= 50C prima di QGL/mesh"
TEMPERATURE_WAIT SENSOR=extruder MAXIMUM=50
```

Nessuna modifica a:

- Eddy;
- curve Eddy;
- `reg_drive_current`;
- QGL;
- adaptive rapid mesh;
- CLEAN_NOZZLE;
- temperature finali;
- LINE_PURGE;
- viti bed;
- parametri protetti extruder.

## Nota sui test manuali PHOENIX_START

Durante i test è stato osservato un blob di PLA nel punto finale della purge line. La causa non era mesh o primo layer: `PHOENIX_START` era stato lanciato manualmente senza una stampa successiva, quindi dopo `LINE_PURGE` il nozzle era rimasto caldo e fermo, continuando a colare.

Questo artefatto non rappresenta il normale flusso di stampa. Per test futuri della sola logica soak va evitato di lasciare completare inutilmente un `PHOENIX_START` manuale fino alla purge se non seguirà una stampa.

## Stato operativo a fine aggiornamento

- PolyTerra PLA: MVS `22 mm³/s`;
- Flow Ratio `1.0465`;
- PA `0.034`;
- retrazione `0.4 mm @ 35 mm/s`;
- primo layer centrale confermato buono anche su impronta più ampia;
- Z globale invariato;
- viti bed non toccate;
- rapid scan mantenuto con `METHOD=rapid_scan`;
- thermal credit con decadimento/riaccumulo graduale;
- soak configurabile con default `600 s` e override `SOAK_SECONDS`;
- Orca passa esplicitamente `SOAK_SECONDS=600`.

## Prossimi passi

1. fare un lavoro di fino sulla logica/macros senza riaprire componenti già validati;
2. eventualmente verificare in uso reale una seconda stampa avviata poco dopo la prima, senza `RESTART`, per osservare il credito residuo nel caso reale;
3. tornare al problema locale FL/FR solo se necessario, includendo verifica fisica della serigrafia PEI a piatto freddo;
4. non correggere lo Z globale finché le stampe centrali continuano a mostrare un primo layer corretto.

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
