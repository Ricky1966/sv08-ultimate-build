# Native Eddy — Sovol Eddy NG hardware on Klipper Mainline

Ultima verifica delle fonti: **2026-08-10**.

## Scopo

Questa guida documenta la migrazione dell'hardware Sovol Eddy NG / LDC1612 dal vecchio percorso plugin specifico Eddy-NG al supporto Eddy nativo disponibile in Klipper Mainline.

La macchina Phoenix è stata realmente migrata fino a:

- homing Z funzionante;
- probing statico;
- QGL;
- rapid bed mesh;
- integrazione Demon;
- prima stampa verificata.

## Prerequisiti

Devono essere già completati:

- `docs/backup-and-rollback.md`
- `docs/install-cb1-mainline.md`
- `docs/flash-mcus.md`
- `docs/base-configuration.md`

Le MCU, gli assi, le temperature e gli endstop X/Y devono già essere funzionanti.

## Hardware

La macchina Phoenix utilizza:

- sensore Sovol Eddy NG;
- controller LDC1612;
- collegamento I2C attraverso la toolboard;
- MCU toolboard identificata come `extra_mcu`.

L'hardware può continuare a essere utilizzato senza mantenere il vecchio plugin `probe_eddy_ng.py`.

## Perché è stato abbandonato il vecchio plugin

Durante la migrazione Phoenix il vecchio percorso `probe_eddy_ng` è stato inizialmente portato su Klipper Mainline.

Il problema principale è emerso nel percorso di homing Z.

Il plugin poteva entrare in una dipendenza circolare:

- `G28 Z` richiedeva il probe;
- il percorso probe richiedeva che Z risultasse già homed.

Il sintomo osservato era:

`Z axis must be homed before probing`

Sono stati sperimentati workaround temporanei, ma è stato deciso di non continuare a patchare `probe_eddy_ng.py`.

Il percorso community raccomandato è quindi il supporto Eddy nativo di Klipper Mainline.

## Non usare falsi homing Z

Durante il debugging era stato usato temporaneamente:

`set_position_z: 0`

Questo dichiara artificialmente Z come homed senza aver stabilito fisicamente il riferimento.

È stato rimosso.

Non usare `set_position_z: 0` come soluzione permanente a problemi di homing Eddy.

## Sezione Eddy nativa

La configurazione Phoenix verificata utilizza:

`[probe_eddy_current eddy]`

Parametri:

- `sensor_type: ldc1612`
- `i2c_mcu: extra_mcu`
- clock I2C su `PB6`
- data I2C su `PB7`
- `x_offset: -16.43`
- `y_offset: 10.22`
- `descend_z: 0.5`
- `max_sensor_hz: 9000000`

La calibrazione ha inoltre salvato:

`reg_drive_current: 22`

Gli offset X/Y appartengono alla toolhead Phoenix e devono essere verificati sulla propria macchina.

## Nota sulla sintassi I2C

Durante la migrazione storica erano presenti forme di configurazione con pin software espliciti associati a `extra_mcu`.

La configurazione Mainline corrente deve seguire la sintassi supportata dalla versione Klipper realmente installata.

Non copiare una vecchia sintassi solo perché compare nella cronologia Phoenix.

## `max_sensor_hz`

Il primo tentativo di homing Z con il supporto nativo aveva prodotto:

`Trigger analog error: RAW_RANGE`

La calibrazione mostrava frequenze fino a circa:

`8.523 MHz`

Il limite nativo iniziale risultava troppo basso per il sensore della Phoenix.

È stato quindi impostato:

`max_sensor_hz: 9000000`

Dopo questa modifica `G28 Z` ha completato senza contatto del nozzle con il piatto.

Questo valore è verificato sulla Phoenix.

Prima di applicarlo altrove, controllare la frequenza reale osservata dal proprio sensore.

## Drive current

La calibrazione nativa del sensore Phoenix ha prodotto:

`reg_drive_current: 22`

Il valore è stato salvato tramite `SAVE_CONFIG`.

Non assumere che `22` sia corretto per ogni sensore Eddy.

Eseguire la calibrazione prevista dalla propria versione Klipper.

## Calibrazione Eddy current

Sulla Phoenix è stata eseguita:

`PROBE_EDDY_CURRENT_CALIBRATE CHIP=eddy`

La curva risultante copriva circa:

- Z: `0.050625 -> 2.090625 mm`
- frequenza: `8.523 -> 8.451 MHz`

La calibrazione deve essere eseguita sulla propria macchina.

Non copiare una curva calibrata da un'altra stampante.

## `descend_z`

La configurazione Phoenix mantiene:

`descend_z: 0.5`

Questo parametro appartiene al comportamento del probing nativo corrente.

Non confonderlo con vecchi parametri `z_offset` usati da implementazioni precedenti.

## Homing override

Durante la migrazione sono state rimosse le vecchie chiamate:

`PROBE_EDDY_NG_PROBE_STATIC HOME_Z=1`

Il percorso Z deve utilizzare esclusivamente il probe nativo configurato.

Nel full homing Phoenix era presente un sollevamento iniziale Z.

È stato reso sicuro eseguendolo soltanto se Z risultava già homed.

Concettualmente:

`if Z is already homed -> raise Z before XY movement`

Questo evita di impartire un movimento Z relativo a una posizione non ancora conosciuta.

## Full homing verificato

Dopo la rimozione dei workaround legacy e la configurazione Eddy nativa, un `G28` completo partendo da assi totalmente non homed è stato completato con successo.

Posizione finale osservata sulla Phoenix:

- X `191`
- Y `165`
- Z `10`

Queste coordinate appartengono al workflow Phoenix.

## Validazione a freddo

Prima di procedere a QGL e mesh sono stati verificati:

- full `G28`;
- `PROBE` al centro;
- riallineamento Z dal probe;
- probing usato dal workflow Demon;
- nessun contatto indesiderato nozzle-bed.

Valori centrali `PROBE` osservati a freddo:

- `0.002734 mm`
- `0.006694 mm`

Questi valori sono risultati sufficientemente coerenti per proseguire.

## Attenzione dopo `PROBE`

Un semplice `PROBE` lascia la toolhead vicino alla quota di trigger.

Prima di ripetere una nuova operazione di probing:

- rialzare Z;
- verificare che il nozzle abbia distanza libera dal piatto.

Non eseguire probing ripetuti assumendo che la toolhead sia tornata automaticamente a una quota sicura.

## Validazione a caldo

Condizioni del test Phoenix:

- bed `65 °C`
- nozzle `170 °C`
- soak circa `10 minuti`

Il full homing a caldo è stato completato senza problemi.

`PROBE` centrale osservato:

`0.007357 mm`

La differenza rispetto ai valori a freddo era dell'ordine di pochi micron nelle condizioni di quel test.

Questo non dimostra universalmente assenza di drift termico.

Ogni macchina deve essere verificata nelle proprie condizioni operative.

## QGL con Eddy nativo

Dopo aver validato homing e probing, sulla Phoenix è stato eseguito `QUAD_GANTRY_LEVEL` senza modificare la configurazione QGL consolidata.

Risultato osservato:

- retries: `3/5`
- probed points range: `0.018441`
- tolerance: `0.050000`

Configurazione QGL rimasta invariata:

- `horizontal_move_z: 3`
- `retry_tolerance: 0.05`
- `retries: 5`
- `max_adjust: 4`

Il risultato non ha indicato la necessità di modificare:

- gantry;
- motori Z;
- viti;
- cinematica.

## Rapid bed mesh

Dopo QGL è stata eseguita una mesh con:

`BED_MESH_CALIBRATE METHOD=rapid_scan`

La mesh Phoenix 15 x 15 a caldo ha prodotto circa:

- max: `+0.163 mm`
- min: `-0.178 mm`
- range: `0.341 mm`

Questo valore era coerente con la geometria già osservata sul piatto originale.

Non sono stati osservati spike isolati tali da indicare un errore evidente di probing.

La mesh di test non è stata resa permanente durante quella fase.

## Stato intermedio Demon — da NON replicare

Durante una fase intermedia della migrazione era ancora presente un override locale:

`[gcode_macro _APPLY_EDDY_Z_OFFSET]`

che trasformava la macro in un no-op.

Era stato creato per il vecchio Klipper Sovol 0.12, dove il campo usato da Demon non era disponibile.

In quella fase il blocco appariva ancora intenzionalmente mantenuto come workaround legacy.

Questo stato non rappresenta la configurazione finale corretta su Mainline.

## Root cause del first layer errato

Il giorno successivo, analizzando il primo layer fallito, è stato dimostrato che proprio quell'override legacy impediva a Demon di riallineare correttamente la coordinata Z dopo `_PROBE_TAP`.

Nel print fallito il probe aveva stimato:

`z=-0.060846`

La mancata applicazione di circa `0.061 mm` di correzione è risultata sufficiente a compromettere fortemente un first layer da `0.20 mm`.

La causa primaria non era:

- QGL;
- mesh;
- motori Z;
- meccanica;
- sottoestrusione generalizzata.

Era l'override legacy rimasto nel `printer.cfg`.

## Correzione finale

L'override locale `_APPLY_EDDY_Z_OFFSET` è stato rimosso.

È stata quindi lasciata attiva la macro Demon corretta che usa:

`printer.probe.last_probe_position.z`

per riallineare la coordinata Z tramite:

`SET_KINEMATIC_POSITION`

Il percorso finale verificato è:

`_PROBE_TAP -> _APPLY_EDDY_Z_OFFSET -> printer.probe.last_probe_position.z -> SET_KINEMATIC_POSITION`

Dopo questa correzione, ristampando lo stesso G-code, il first layer è migliorato radicalmente.

## Test manuale del riallineamento Z

Dopo la rimozione dell'override legacy sono stati eseguiti:

`G28`

e:

`_PROBE_TAP`

Il sistema ha riportato:

`Setting position to 0.457101`

e una stima contatto:

`z=0.065696`

Questo ha confermato che la macro Mainline + Eddy nativo + Demon applicava nuovamente il riallineamento Z.

## Regola pratica

Su Klipper Mainline corrente:

non mantenere override creati esclusivamente per compensare limitazioni del vecchio Klipper Sovol.

In particolare, verificare e rimuovere eventuali blocchi locali che ridefiniscono:

- `_APPLY_EDDY_Z_OFFSET`
- `_PROBE_TAP`
- homing Z
- probe statico legacy
- vecchie macro Eddy NG

prima di concludere che il problema è meccanico.

## TAP

Il supporto Eddy nativo Mainline gestisce TAP con semantica propria.

Durante la migrazione Phoenix era stato rilevato un problema nella logica Demon che selezionava il ramo TAP solo perché la chiave `tap_threshold` esisteva con valore di default `0.0`.

Il risultato era:

`Tap not configured`

La correzione usata in quella fase consisteva nel considerare TAP attivo solo quando il valore configurato era maggiore di zero.

Questo comportamento appartiene alla cronologia dell'integrazione Demon.

Per installazioni nuove usare la logica e la documentazione DKEU corrente.

Non aggiungere automaticamente vecchi valori come:

`tap_threshold: 300`

La semantica del TAP nativo corrente non è quella del vecchio plugin Eddy NG.

## RAW_RANGE

Se durante homing o calibrazione compare:

`Trigger analog error: RAW_RANGE`

verificare prima:

- frequenza reale osservata dal sensore;
- `max_sensor_hz`;
- calibrazione current;
- cablaggio I2C;
- alimentazione sensore.

Sulla Phoenix il problema è stato risolto aumentando:

`max_sensor_hz`

a:

`9000000`

perché la curva misurata arrivava oltre 8.5 MHz.

Non trattare `9000000` come valore universale.

## Problemi che NON devono portare subito a modifiche meccaniche

Se:

- homing funziona;
- PROBE è coerente;
- QGL converge;
- mesh è ragionevole;

ma il first layer è ancora errato, verificare prima:

- macro legacy;
- Z realignment;
- override locali;
- slicer start G-code;
- Demon;
- stato del probe;

prima di toccare:

- quattro Z;
- gantry;
- viti;
- axis twist;
- geometria meccanica.

Questo principio ha evitato modifiche meccaniche inutili durante il recupero Phoenix.

## Slicer e falsa diagnosi Eddy

Durante il primo tentativo di stampa è stato osservato anche un errore completamente indipendente dal probe:

OrcaSlicer era rimasto selezionato sulla macchina Sovol invece che sulla macchina Phoenix.

Il G-code non conteneva quindi il Machine Start Demon atteso e Demon ha eseguito un emergency shutdown.

Questo episodio dimostra che non ogni errore apparso durante la fase Eddy è causato da Eddy.

Verificare sempre anche:

- profilo macchina attivo;
- start G-code;
- profilo materiale;
- profilo processo.

## Criterio di uscita

Prima di considerare Eddy completato devono funzionare:

- full homing;
- homing Z reale;
- `PROBE`;
- calibrazione current;
- QGL;
- rapid scan;
- nessun RAW_RANGE;
- nessun vecchio comando `PROBE_EDDY_NG_*` ancora necessario;
- nessun `set_position_z: 0`;
- riallineamento Z corretto dopo probing;
- first layer coerente su un test reale.

## Passo successivo

Integrare Demon sul sistema Mainline già stabile:

`docs/demon-integration.md`
