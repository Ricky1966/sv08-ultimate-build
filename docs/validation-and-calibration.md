# Validation and calibration — Sovol SV08 Mainline

Ultima revisione: **2026-08-23**.

## Scopo

Questa guida definisce l'ordine di validazione della Sovol SV08 Phoenix con:

- Klipper Mainline;
- MCU Mainline;
- Sovol Zero Extruder Kit via CAN;
- Eddy nativo integrato nella Zero;
- Phoenix Macros.

L'obiettivo è evitare di calibrare il materiale quando la macchina non è ancora geometricamente stabile, oppure di correggere meccanicamente problemi che appartengono a macro, Z o slicer.

## Ordine generale

La sequenza consigliata è:

1. sanity check hardware;
2. PID;
3. homing;
4. QGL;
5. probing;
6. bed mesh;
7. first layer;
8. temperatura materiale;
9. flow ratio;
10. pressure advance;
11. retraction;
12. max volumetric speed;
13. stampa reale di validazione.

Non invertire l'ordine senza una ragione precisa.

## Tre famiglie di calibrazione

È utile distinguere sempre:

### Machine calibration

Dipende principalmente dall'hardware della stampante.

Comprende:

- PID;
- direzioni stepper;
- endstop;
- limiti;
- cinematica;
- heater;
- ventole.

### Bed/Z calibration

Dipende dal sistema meccanico e dal piano.

Comprende:

- QGL;
- probe;
- Z reference;
- mesh;
- first layer.

### Filament calibration

Dipende principalmente da:

- materiale;
- hotend;
- nozzle;
- temperatura;
- velocità.

Comprende:

- temperatura;
- flow ratio;
- pressure advance;
- retraction;
- max volumetric speed.

## Regola fondamentale

Non usare una calibrazione materiale per nascondere un errore della macchina.

Esempi:

- non aumentare flow per compensare un nozzle troppo alto;
- non ridurre flow per compensare un nozzle troppo basso;
- non modificare pressure advance per correggere una mesh errata;
- non modificare Z per compensare un problema di extrusion multiplier.

Prima stabilizzare macchina e bed.

Poi calibrare il materiale.

## Sanity check iniziale

Prima di qualunque calibrazione verificare:

- nessun errore Klipper;
- mainboard collegata;
- toolhead Zero raggiungibile via CAN;
- temperature plausibili;
- ventole funzionanti;
- heater corretti;
- homing X/Y/Z funzionante;
- Eddy operativo;
- nessuna macro legacy che interferisca;
- nessuna collisione meccanica.

## PID

Il PID deve essere calibrato dopo modifiche che cambiano significativamente il comportamento termico.

Per l'hotend questo include:

- cambio hotend;
- cambio heater;
- cambio thermistor;
- modifica importante del blocco termico.

Per il bed questo include:

- cambio bed;
- cambio heater;
- modifica significativa della massa termica;
- modifica importante dell'isolamento.

## Storico Phoenix — PID hotend con MicroSwiss FlowTech

Durante una fase precedente della Phoenix era installato un MicroSwiss FlowTech.

Questa non è più la baseline hardware attuale con Sovol Zero.

Durante la migrazione Mainline sono stati salvati valori PID hotend a `220 °C`:

- `pid_Kp: 26.772`
- `pid_Ki: 2.052`
- `pid_Kd: 87.343`

Questi valori appartengono a quella specifica fase e configurazione Phoenix.

Non copiarli su un'altra macchina.

Effettuare sempre una propria calibrazione PID.

## Storico Phoenix — PID bed della configurazione precedente

Nella stessa fase Phoenix risultavano salvati:

- `pid_Kp: 74.106`
- `pid_Ki: 1.254`
- `pid_Kd: 1094.910`

Anche questi valori sono hardware-specifici.

Non usarli come preset universale.

## Cambio piatto

Un cambio piatto può modificare:

- massa termica;
- distribuzione temperatura;
- risposta PID;
- geometria;
- posizione Z;
- mesh;
- comportamento del first layer.

Dopo un cambio importante del piatto ripetere almeno:

- PID bed;
- QGL;
- probe/reference Z;
- mesh;
- first layer.

Non è invece necessario azzerare automaticamente tutte le calibrazioni del filamento.

## Homing

Dopo il PID verificare nuovamente:

`G28`

Controllare:

- X;
- Y;
- Z;
- posizione finale;
- assenza di urti;
- distanza nozzle-bed.

Non proseguire se l'homing non è ripetibile.

## QGL

Eseguire `QUAD_GANTRY_LEVEL` solo dopo che homing e probe sono affidabili.

Il QGL deve:

- convergere;
- non richiedere correzioni estreme;
- produrre risultati ripetibili.

Un QGL non deve essere usato per correggere una configurazione Z sbagliata.

## Storico Phoenix — QGL precedente alla Sovol Zero

Prima dell'installazione definitiva della Sovol Zero, un test Phoenix a caldo con la precedente configurazione Eddy nativa aveva prodotto:

- retries: `3/5`
- probed range: `0.018441`
- tolerance: `0.050000`

Configurazione utilizzata:

- `horizontal_move_z: 3`
- `retry_tolerance: 0.05`
- `retries: 5`
- `max_adjust: 4`

Il risultato non giustificava interventi su gantry o motori Z.

## Probe validation

Dopo QGL verificare il probe separatamente.

Controllare:

- valori coerenti;
- nessun errore RAW_RANGE;
- assenza di contatto indesiderato;
- comportamento coerente a freddo e a caldo.

Non assumere che una mesh riuscita dimostri automaticamente che tutto il percorso Z sia corretto.

## Mesh

Solo dopo homing, QGL e probing stabili creare la mesh.

Per Eddy Mainline è possibile utilizzare `rapid_scan` quando la configurazione è già stata validata.

Durante il debugging storico della configurazione DKEU fu utile verificare anche il comportamento del percorso nativo senza il wrapper Demon allora utilizzato.

## Storico Phoenix — mesh precedente alla Sovol Zero

Nella precedente configurazione Phoenix, la mesh 15 x 15 a caldo con:

`BED_MESH_CALIBRATE METHOD=rapid_scan`

ha prodotto circa:

- max: `+0.163 mm`
- min: `-0.178 mm`
- range: `0.341 mm`

Una sessione precedente aveva mostrato circa:

- max: `+0.194 mm`
- min: `-0.160 mm`
- range: `0.354 mm`

La coerenza fra le due misure ha indicato che la mesh stava rappresentando una geometria reale del piatto, non rumore casuale.

## Non inseguire una mesh piatta

Una mesh non deve essere necessariamente quasi zero.

Il suo compito è descrivere la superficie reale.

Una mesh artificialmente piatta può essere più pericolosa di una mesh con range maggiore ma fisicamente corretta.

Durante la storia Phoenix una compensazione precedente aveva quasi annullato una differenza reale di circa `0.33 mm`, producendo un first layer errato.

## First layer

Il first layer viene validato solo dopo:

- homing;
- QGL;
- probe;
- mesh.

Usare una geometria di test abbastanza ampia da mostrare differenze fra zone del piano.

Un piccolo quadrato centrale non è sufficiente per giudicare una SV08 da 350 mm.

## Test multi-zona

Durante il recupero Phoenix è stato utilizzato un first layer distribuito in più zone del piatto.

Questo permette di distinguere:

- errore Z globale;
- mesh errata;
- difetto locale del piatto;
- contaminazione;
- variazione di adesione.

## Diagnosi del first layer

Se il layer è troppo schiacciato ovunque:

verificare prima il riferimento Z.

Se è troppo alto ovunque:

verificare prima il riferimento Z.

Se varia sistematicamente da un lato all'altro:

verificare:

- mesh;
- QGL;
- probe;
- compensazioni residue.

Se il difetto è isolato:

considerare anche:

- sporco;
- contaminazione;
- difetto locale del piano.

## Non confondere Z e flow

Un first layer troppo pieno non significa automaticamente flow troppo alto.

Un first layer scarso non significa automaticamente flow troppo basso.

Flow ratio deve essere calibrato solo dopo aver dimostrato che il sistema Z/bed è corretto.


## Filament calibration

Solo quando:

- macchina;
- Z;
- QGL;
- probe;
- mesh;
- first layer

sono già affidabili, iniziare la calibrazione del filamento.

La sequenza consigliata è:

1. temperatura;
2. flow ratio;
3. pressure advance;
4. retraction;
5. max volumetric speed.

## Temperatura

Calibrare la temperatura prima del flow.

La temperatura influenza:

- viscosità;
- adesione fra layer;
- bridging;
- stringing;
- qualità superficiale;
- portata massima.

Non calibrare flow o pressure advance con una temperatura ancora incerta.

## Storico Phoenix — PolyTerra PLA con FlowTech

Per il profilo:

`Polymaker PolyTerra PLA @Phoenix 0.4`

la temperatura consolidata usata per le calibrazioni è stata:

`200 °C`

Questa era una scelta validata per quella specifica configurazione storica:

- Phoenix;
- MicroSwiss FlowTech;
- nozzle 0.4;
- quel materiale.

Non è un valore universale per ogni PLA.

## Flow ratio

Il flow ratio deve correggere la quantità reale di materiale estruso durante la stampa.

Non usarlo per compensare:

- Z errato;
- nozzle troppo vicino;
- nozzle troppo lontano;
- mesh errata;
- primo layer non calibrato.

## Storico Phoenix — Flow ratio con FlowTech

La calibrazione PolyTerra della precedente configurazione FlowTech aveva prodotto:

`Flow ratio = 1.0465`

Il valore deriva dalla calibrazione OrcaSlicer eseguita sulla Phoenix.

Non copiarlo direttamente su:

- altro hotend;
- altro nozzle;
- altro materiale;
- altra bobina senza verifica.

## Pressure Advance

Pressure Advance deve essere calibrato dopo il flow.

Serve a compensare il comportamento dinamico della pressione nel sistema di estrusione.

Valori errati possono produrre:

- angoli gonfi;
- angoli svuotati;
- variazioni di spessore durante accelerazioni;
- artefatti nelle transizioni di velocità.

## Storico Phoenix — Pressure Advance con FlowTech

Nella precedente configurazione FlowTech, la zona migliore osservata nella calibrazione PolyTerra era circa:

`0.034`

Valore consolidato:

`Pressure Advance = 0.034`

Durante fasi precedenti erano stati usati valori diversi, tra cui:

- `0.025` nel printer.cfg;
- `0.032` in profili/test slicer.

Questo dimostra perché va sempre verificato quale sorgente stia realmente impostando il PA durante la stampa.

## Chi controlla il Pressure Advance

Il PA può arrivare da:

- `printer.cfg`;
- OrcaSlicer;
- start G-code;
- macro;
- profilo filamento.

Nella baseline Phoenix attuale il Pressure Advance viene lasciato allo **slicer**, evitando una seconda sorgente concorrente nelle Phoenix Macros.

Prima di calibrare:

verificare chi lo sta realmente impostando.

Due valori corretti in due posti diversi possono comunque produrre un workflow incoerente.

## Retraction

Calibrare la retraction dopo:

- temperatura;
- flow;
- pressure advance.

La retraction deve essere aumentata solo se esiste un problema reale di stringing o ooze.

Non aumentarla automaticamente.

Valori eccessivi possono causare:

- ritardi di ripresa estrusione;
- vuoti;
- variazioni pressione;
- heat creep;
- intasamenti in alcuni hotend.

## Firmware retraction

La Phoenix ha avuto anche una configurazione:

- `retract_length: 0.8`
- `retract_speed: 30`
- `unretract_extra_length: 0.0`
- `unretract_speed: 30`

ma i G-code analizzati in quella fase non utilizzavano:

- `G10`;
- `G11`;
- `SET_RETRACTION`.

Quindi quella configurazione non stava necessariamente governando la retraction effettiva della stampa.

## Slicer retraction

Durante i test Phoenix sono comparsi valori slicer differenti.

Questo rende obbligatorio verificare il G-code reale.

Non basta leggere:

- printer.cfg;
- metadata;
- preset.

Quando necessario cercare direttamente i movimenti E e i comandi di retraction nel G-code prodotto.

## Storico Phoenix — Retraction con FlowTech

Durante la calibrazione PolyTerra della precedente configurazione FlowTech:

- le torri erano sostanzialmente pulite;
- non era presente stringing significativo;
- non è stato necessario aumentare ulteriormente la retraction.

La retraction utilizzata in quella configurazione è stata quindi mantenuta come valore validato per quella fase.

Il valore esatto non viene proposto come preset universale perché dipende dal percorso realmente usato dal profilo slicer.

## Max Volumetric Speed

Il Max Volumetric Speed definisce la portata massima di materiale richiesta all'hotend.

Deve essere calibrato dopo:

- temperatura;
- flow;
- PA;
- retraction.

Se impostato troppo alto può produrre:

- sottoestrusione;
- perdita di qualità;
- pareti deboli;
- inconsistenza alle alte velocità.

## Storico Phoenix — Max Volumetric Speed con FlowTech

Il profilo PolyTerra aveva inizialmente:

`22 mm³/s`

Dopo calibrazione è stato consolidato:

`24 mm³/s`

Questo valore era stato scelto come limite operativo validato per quella specifica combinazione Phoenix.

Non rappresenta la capacità universale del FlowTech né del PolyTerra PLA.

## Storico Phoenix — profilo consolidato con FlowTech

Valori finali validati:

- nozzle calibration: `200 °C`
- flow ratio: `1.0465`
- pressure advance: `0.034`
- retraction: corrente, validata dal test
- max volumetric speed: `24 mm³/s`

Questi valori appartengono alla precedente combinazione:

- Phoenix;
- MicroSwiss FlowTech;
- nozzle 0.4;
- PolyTerra PLA;
- profilo OrcaSlicer specifico.

Non rappresentano automaticamente la baseline Sovol Zero corrente.

## Cambio piatto e calibrazione filamento

Un cambio del piatto non obbliga automaticamente a rifare da zero:

- flow;
- PA;
- retraction;
- MVS.

Dopo un cambio piatto importante devono invece essere rivalidati prima:

- PID bed;
- QGL;
- Z reference;
- mesh;
- first layer.

Solo se una stampa successiva mostra problemi reali va riaperta la calibrazione del filamento.

## Cambio hotend o nozzle

Un cambio hotend o nozzle può invece richiedere una rivalidazione più ampia.

In particolare:

- PID hotend;
- temperatura;
- flow;
- PA;
- retraction;
- MVS.

Un nozzle di diametro diverso modifica direttamente la portata richiesta e il comportamento dell'estrusione.

## Stampa reale di validazione

Le torri di calibrazione non sono sufficienti da sole.

Dopo aver consolidato i valori eseguire una stampa reale conosciuta.

Controllare:

- first layer;
- pareti;
- top surface;
- angoli;
- stringing;
- bridging;
- dettagli piccoli;
- consistenza alle alte velocità.

## Confronto A/B

Quando si corregge un singolo problema, ristampare se possibile lo stesso G-code.

Questo permette di confrontare direttamente:

- prima;
- dopo.

Durante la precedente fase Phoenix basata su DKEU, questo metodo permise di distinguere chiaramente l'effetto della correzione storica `_APPLY_EDDY_Z_OFFSET` da eventuali cambiamenti di slicer o materiale.

Quella macro appartiene alla precedente integrazione DKEU e non fa parte del runtime Phoenix Macros corrente.

## Freeze della calibrazione

Quando una calibrazione è stata validata:

- documentarla;
- salvarla;
- versionarla;
- non modificarla senza una causa.

Una configurazione stabile è più utile di una configurazione continuamente ritoccata.

## Cosa rifare dopo modifiche hardware

### Cambio bed

Rifare almeno:

- PID bed;
- QGL;
- Z reference;
- mesh;
- first layer.

### Cambio hotend

Rifare almeno:

- PID hotend;
- temperatura materiale;
- flow;
- PA;
- retraction;
- MVS.

### Cambio nozzle

Rivalidare almeno:

- temperatura;
- flow;
- PA;
- retraction;
- MVS.

### Cambio probe/toolhead

Rifare almeno:

- offset X/Y;
- homing Z;
- calibrazione Eddy;
- QGL;
- mesh;
- first layer.

## Criterio di uscita

La macchina può essere considerata calibrata quando:

- PID stabile;
- homing ripetibile;
- QGL converge;
- probe coerente;
- mesh coerente;
- first layer uniforme;
- temperatura materiale definita;
- flow definito;
- PA definito;
- retraction verificata;
- MVS definito;
- stampa reale completata senza difetti sistematici evidenti.

---

## Navigazione

- ← **Pagina precedente:** [Phoenix Macros](phoenix-macros.md)
- → **Pagina successiva:** [Troubleshooting](troubleshooting.md)
