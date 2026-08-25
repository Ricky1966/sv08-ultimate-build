# Phoenix Macros

## Stato

Phoenix Macros costituisce il layer di macro attivo della Sovol SV08 “Phoenix”.

La baseline corrente è:

- Klipper Mainline;
- Sovol Zero Extruder Kit via CAN;
- Eddy integrato tramite supporto nativo Klipper;
- Phoenix Macros;
- componenti esterni esplicitamente attribuiti dove ancora utilizzati.

**DKEU non è più una dipendenza runtime della configurazione Phoenix.**

Il 22 agosto 2026 è stato completato l'audit strutturale del runtime attivo sulla macchina; la validazione funzionale principale è stata completata il 23 agosto 2026. Il 25 agosto è iniziata l'integrazione del nuovo sistema Phoenix Automatic Soak, attualmente in validazione funzionale sulla macchina.

La baseline Phoenix verificata comprende:

- nessun include DKEU attivo in `printer.cfg`;
- nessun riferimento operativo DKEU nei file `phoenix-*.cfg`;
- nessun uso di `force_move`;
- nessun uso di `M84`;
- 21 macro `PHOENIX_*` validate nella baseline del 23 agosto;
- 5 nuove macro utente Phoenix Automatic Soak aggiunte il 25 agosto e in validazione;
- Klipper riavviato senza errori di configurazione, Jinja o runtime durante la validazione strutturale dei pack.

Phoenix usa ora `save_variables` esclusivamente per preferenze utente persistenti del proprio sistema Automatic Soak, tramite un file Phoenix dedicato. Il vecchio `~/demon_vars.cfg` non viene riutilizzato.

Questa verifica certifica l'**autonomia strutturale/runtime** di Phoenix Macros rispetto a DKEU. La validazione fisica delle singole macro viene eseguita separatamente sulla macchina e non va confusa con il solo caricamento corretto da parte di Klipper.

## Principi

Phoenix Macros nasce per mantenere soltanto il comportamento realmente necessario e validato sulla Phoenix, evitando framework, wrapper e dipendenze non più utili.

Principi:

- usare direttamente le funzioni native di Klipper Mainline quando disponibili;
- mantenere le macro Phoenix piccole, leggibili e verificabili;
- separare chiaramente macro proprie e componenti esterni;
- evitare override inutili dei comandi Klipper;
- non usare `M84`;
- non nascondere comportamenti hardware dentro framework generici;
- mantenere la sequenza di stampa aderente ai test realmente eseguiti sulla macchina;
- distinguere sempre tra validazione strutturale e test fisico.

## File Phoenix attivi sulla macchina

La configurazione corrente include, in questo ordine:

```text
phoenix-print-start-end.cfg
phoenix-cleaner.cfg
phoenix-runout.cfg
phoenix-idle.cfg
phoenix-filament.cfg
phoenix-core.cfg
phoenix-calibration.cfg
phoenix-debug.cfg
phoenix-setup.cfg
```

I pack aggiunti durante la separazione definitiva da DKEU sono organizzati per responsabilità:

- **Core** — funzioni operative di base e compatibilità;
- **Calibration** — strumenti di calibrazione espliciti;
- **Debug** — diagnostica e test;
- **Setup** — helper di configurazione/verifica manuale.

## Macro Phoenix correnti

### Workflow di stampa

- `PHOENIX_START`
- `PHOENIX_END`
- `PHOENIX_CLEAN_NOZZLE`
- `PHOENIX_FILAMENT_RUNOUT`
- `PHOENIX_IDLE_TIMEOUT`

### Filamento e Core

- `PHOENIX_LOAD_FILAMENT`
- `PHOENIX_UNLOAD_FILAMENT`
- `PHOENIX_FILAMENT_CHANGE`
- `PHOENIX_READY_UP`
- `PHOENIX_PRESENT_TOOLHEAD`

Alias di compatibilità mantenuti intenzionalmente:

```text
LOAD_FILAMENT -> PHOENIX_LOAD_FILAMENT
UNLOAD_FILAMENT -> PHOENIX_UNLOAD_FILAMENT
M600 -> PHOENIX_FILAMENT_CHANGE
FILAMENT_CHANGE -> PHOENIX_FILAMENT_CHANGE
```

### Automatic Soak

Aggiunto il 25 agosto 2026 e attualmente in validazione funzionale.

Macro utente:

- `PHOENIX_SOAK_AUTOSTART ENABLE=0|1`
- `PHOENIX_SOAK_TEMPERATURE BED_TEMP=<temperatura>`
- `PHOENIX_SOAK_START [BED_TEMP=<temperatura>]`
- `PHOENIX_SOAK_STOP`
- `PHOENIX_SOAK_STATUS`

Il sistema usa due preferenze persistenti:

```text
phoenix_soak_autostart
phoenix_soak_temperature
```

salvate tramite:

```ini
[save_variables]
filename: ~/printer_data/config/phoenix_variables.cfg
```

Il file storico `~/demon_vars.cfg` non viene usato dal sistema Phoenix.

#### Avvio automatico

Se `phoenix_soak_autostart` è abilitato, pochi secondi dopo l'avvio di Klipper Phoenix:

1. legge la temperatura configurata;
2. imposta il target del bed;
3. comunica esplicitamente in console che l'Automatic Soak è stato avviato;
4. attende che il bed raggiunga la finestra utile di temperatura;
5. inizia ad accumulare credito termico;
6. registra il soak come valido quando il credito richiesto è completo.

Il timer **non** inizia durante la rampa di riscaldamento. Con target 70 °C il credito comincia quando il bed raggiunge circa 69 °C.

La baseline corrente del soak è 600 secondi, cioè 10 minuti effettivi nella finestra termica prevista.

#### Stato termico e credito

Il sistema riusa lo stato Phoenix esistente:

```text
_PHOENIX_THERMAL_STATE
```

con, tra le altre, le variabili runtime:

```text
soak_valid
soak_temp
soak_total_seconds
thermal_credit_seconds
```

La preferenza dell'utente è persistente; la validità termica del soak **non** viene salvata come stato permanente tra spegnimenti.

`PHOENIX_SOAK_STATUS` mostra:

- Automatic Soak abilitato o disabilitato;
- stato `INACTIVE`, `TRACKING` o `VALID`;
- temperatura configurata;
- temperatura reale e target del bed;
- credito termico accumulato.

`PHOENIX_SOAK_STOP` interrompe il soak corrente e spegne il bed senza cambiare la preferenza persistente di autostart.

`PHOENIX_SOAK_START` permette l'avvio manuale usando la temperatura persistente oppure un override temporaneo tramite `BED_TEMP`.

#### Integrazione con `PHOENIX_START`

La logica in integrazione permette a `PHOENIX_START` di utilizzare:

- un soak già completamente valido;
- oppure il credito parziale accumulato da un Automatic Soak ancora in corso.

In questo modo una stampa avviata dopo alcuni minuti di preriscaldamento non deve necessariamente ripetere da zero tutto il soak previsto.

### Calibration Pack

- `PHOENIX_PID_TUNE`
- `PHOENIX_SHAPER_CALIBRATE`
- `PHOENIX_RESONANCE_TEST_X`
- `PHOENIX_RESONANCE_TEST_Y`
- `PHOENIX_MACHINE_LEVEL`

Queste macro non eseguono `SAVE_CONFIG` automaticamente: il risultato deve essere verificato e accettato esplicitamente prima del salvataggio.

`PHOENIX_PRESSURE_ADVANCE_TEST` è stata rimossa il 23 agosto 2026 perché ridondante rispetto ai workflow di calibrazione Pressure Advance già disponibili nello slicer. La calibrazione PA viene quindi delegata allo slicer.

### Debug Pack

- `PHOENIX_PROBE_ACCURACY`
- `PHOENIX_PRINTER_STATUS`
- `PHOENIX_SYSTEM_SENSORS`
- `PHOENIX_REFERENCE_BED_MESH`
- `PHOENIX_STEPPER_BUZZ`

`PHOENIX_PROBE_ACCURACY` usa il percorso nativo Klipper compatibile con l'Eddy corrente e non reintroduce comandi `probe_eddy_ng`.

`PHOENIX_REFERENCE_BED_MESH` è uno strumento diagnostico per produrre una mesh completa non adaptive; non sostituisce il workflow adaptive usato da `PHOENIX_START`.

### Setup Pack

- `PHOENIX_CLEANER_SETUP`

È un helper manuale per portare la testina su coordinate candidate del cleaner e verificarne fisicamente la posizione. Non salva valori e non implementa il vecchio wizard DKEU.

## `PHOENIX_START`

Gestisce il workflow di avvio stampa validato sulla Phoenix.

Sequenza corrente:

1. imposta la temperatura target del bed;
2. recupera, quando compatibile, il credito termico già maturato dal sistema Phoenix;
3. porta l'ugello alla temperatura di preparazione;
4. esegue homing;
5. esegue `PHOENIX_CLEAN_NOZZLE`;
6. spegne il riscaldamento dell'ugello;
7. attiva il raffreddamento durante l'eventuale soak residuo;
8. completa soltanto il soak ancora necessario;
9. attende che l'ugello scenda alla temperatura richiesta prima del QGL;
10. esegue QGL;
11. esegue nuovamente homing Z;
12. genera la bed mesh tramite Klipper Mainline;
13. disattiva il raffreddamento usato durante il soak;
14. porta l'ugello alla temperatura finale di stampa;
15. esegue la purge line;
16. avvia la stampa.

La bed mesh usa il percorso nativo Klipper con rapid scan e adaptive meshing.

Non viene caricato alcun wrapper DKEU per `BED_MESH_CALIBRATE`.

## `PHOENIX_END`

Gestisce la conclusione della stampa secondo il workflow Phoenix.

La macro appartiene al layer Phoenix e sostituisce l'uso delle precedenti macro di fine stampa DKEU.

Non viene eseguito `M84`.

## `PHOENIX_CLEAN_NOZZLE`

Gestisce la pulizia meccanica dell'ugello.

Baseline validata sulla macchina:

- safe Z: 10 mm;
- posizione iniziale: X236 Y359 Z2.5;
- wipe: X236 -> X271;
- ritorno via Y360 -> X236;
- due cicli effettivi;
- velocità: 5 mm/s.

Quando viene eseguita manualmente fuori da una stampa:

1. preriscalda l'ugello a 170 °C;
2. esegue la pulizia;
3. parcheggia a X200 Y200 Z25;
4. spegne il riscaldamento dell'ugello.

Il test fisico manuale è stato completato con successo.

## `PHOENIX_FILAMENT_RUNOUT`

Gestisce il runout del filamento senza dipendere dalle macro DKEU.

Comportamento corrente:

```text
SAVE_GCODE_STATE
PAUSE X=100 Y=10 Z_MIN=50 RESTORE=0
RESTORE_GCODE_STATE
```

Validazione fisica completata il 23 agosto 2026:

- sensore runout aperto durante una stampa reale;
- rilevamento `filament not detected`;
- pausa automatica corretta;
- park a X100 Y10 Z50;
- stato di stampa preservato.

Nella stessa sessione sono state validate fisicamente anche:

- `PHOENIX_LOAD_FILAMENT`;
- `PHOENIX_UNLOAD_FILAMENT`;
- `PHOENIX_FILAMENT_CHANGE`, inclusa pausa, park X100 Y10 Z50 e ripresa corretta con `RESUME`.

## `PHOENIX_IDLE_TIMEOUT`

Implementa la politica idle specifica della Phoenix.

Se la stampante è in pausa:

- spegne la ventola con `M107`;
- spegne il riscaldamento dell'ugello;
- preserva lo stato della stampa.

Negli altri casi:

```text
TURN_OFF_HEATERS
```

Non viene eseguito `M84`.

## Gestione termica

La sequenza termica di avvio è gestita dal workflow Phoenix e dalla macro interna:

```text
_PHOENIX_THERMAL_STATE
```

La logica corrente distingue esplicitamente:

- temperatura target del bed;
- temperatura di preparazione dell'ugello;
- soak;
- credito termico già maturato;
- raffreddamento dell'ugello prima del QGL;
- temperatura finale di stampa.

Il comportamento termico non dipende da variabili o framework DKEU. La persistenza introdotta il 25 agosto riguarda soltanto preferenze Phoenix esplicite e utilizza un file dedicato.

## Funzioni lasciate a Klipper Mainline

Quando Klipper fornisce già direttamente una funzione adeguata, Phoenix Macros non introduce wrapper aggiuntivi.

In particolare:

- `BED_MESH_CALIBRATE` resta nel percorso nativo Klipper Mainline;
- QGL usa il supporto nativo Klipper;
- homing e gestione cinematica restano affidati a Klipper e alla configurazione macchina;
- `PROBE_ACCURACY` usa il supporto nativo disponibile con la configurazione Eddy corrente;
- le funzioni Eddy appartengono alla configurazione Klipper attiva e non a Phoenix Macros.

## Funzioni DKEU non recuperate

La separazione non è stata una copia indiscriminata delle macro DKEU. Diverse funzioni sono state deliberatamente eliminate perché obsolete, invasive, ridondanti o dipendenti dall'infrastruttura DKEU.

Tra queste rientrano, a titolo esemplificativo:

- wrapper DKEU di `BED_MESH_CALIBRATE`;
- `AUTO_BED_MESH_BUILDER`;
- `ADAPTIVE_MESHING_TOGGLE`;
- `HOT_MESH`;
- `ONE_CLICK_EDDY_NG_SETUP`;
- `EDDY_TEMP_COMP`;
- calibrazioni Z legacy basate sui vecchi percorsi probe;
- override `M84`;
- `Z_ASCENDER`;
- sistemi generici bed/chamber/nevermore fan non presenti nella configurazione hardware attiva;
- power-down DKEU;
- backup e shell helpers DKEU.

Dove un concetto DKEU restava utile, il comportamento è stato riscritto come macro Phoenix indipendente e attribuito correttamente.

## Componenti esterni

Phoenix continua a poter utilizzare componenti esterni separati dal layer Phoenix, purché siano chiaramente identificati e attribuiti.

Il file:

```text
Heat_Soak_Sovol_SV08.cfg
```

mantiene correttamente l'attribuzione a 3DPrintDemon e il relativo riferimento upstream.

Tali attribuzioni devono essere preservate.

Il nuovo Phoenix Automatic Soak è un sistema separato e non deriva la propria logica di stato o persistenza dal timer legacy contenuto in questo file.

## Stato storico di DKEU

DKEU ha costituito una fase importante dell'evoluzione della configurazione Phoenix ed è mantenuto nella documentazione storica e nelle attribuzioni dove necessario.

La configurazione runtime corrente non carica più:

```text
./Demon_Klipper_Essentials_Unified/*.cfg
./Demon_User_Files/*.cfg
Demon_User_Files_Handler_v*.cfg
```

Sono inoltre stati rimossi dalla configurazione attiva:

- `[save_variables]` collegato a `~/demon_vars.cfg`;
- `[force_move]`.

Il file `~/demon_vars.cfg` può ancora esistere sulla macchina come residuo storico, ma non è utilizzato dal runtime Phoenix.

Dal 25 agosto Phoenix dispone di un proprio `[save_variables]`, indipendente da DKEU, collegato a:

```text
~/printer_data/config/phoenix_variables.cfg
```

L'audit finale del 22 agosto 2026 ha prodotto esplicitamente:

```text
NESSUN INCLUDE DKEU ATTIVO
NESSUNA DIPENDENZA RUNTIME DKEU
```

Pertanto DKEU resta una **origine storica/upstream**, non una dipendenza operativa della baseline corrente.

## Stato di validazione

Baseline consolidata al 23 agosto 2026:

- tutti i pack Phoenix vengono caricati correttamente da Klipper;
- tutte le 21 macro `PHOENIX_*` della baseline risultano esposte;
- il runtime è strutturalmente indipendente da DKEU;
- `PHOENIX_START` e `PHOENIX_END` sono stati validati in stampa reale;
- `PHOENIX_CLEAN_NOZZLE` è stato validato fisicamente;
- `PHOENIX_IDLE_TIMEOUT` è stato validato anche nel ramo di stampa in pausa;
- `PHOENIX_PID_TUNE` è stato validato per BED ed EXTRUDER;
- `PHOENIX_RESONANCE_TEST_X`, `PHOENIX_RESONANCE_TEST_Y` e `PHOENIX_SHAPER_CALIBRATE` sono stati validati;
- `PHOENIX_LOAD_FILAMENT`, `PHOENIX_UNLOAD_FILAMENT`, `PHOENIX_FILAMENT_RUNOUT` e `PHOENIX_FILAMENT_CHANGE` sono stati validati fisicamente;
- `PHOENIX_PRESSURE_ADVANCE_TEST` è stata rimossa intenzionalmente e non fa più parte del runtime corrente.

### Automatic Soak — validazione del 25 agosto 2026

Verificato finora sulla macchina:

- `save_variables` Phoenix caricato correttamente;
- `phoenix_variables.cfg` creato e persistente;
- lettura corretta di `phoenix_soak_autostart` e `phoenix_soak_temperature`;
- abilitazione dell'autostart senza riscaldamento immediato;
- avvio automatico del bed dopo restart con target configurato a 70 °C;
- nessun credito accumulato durante la rampa termica;
- inizio del credito nella finestra termica prevista;
- incremento del credito a passi di 10 secondi.

Restano da completare prima di considerare la funzione validata:

- raggiungimento e registrazione di `600/600 s` come `VALID`;
- caricamento e test delle macro `PHOENIX_SOAK_START` e `PHOENIX_SOAK_STOP`;
- verifica del nuovo `PHOENIX_SOAK_STATUS` con stati `INACTIVE`, `TRACKING` e `VALID`;
- test del passaggio del credito parziale/valido a `PHOENIX_START`.

## Packaging

Phoenix Macros deve essere mantenuto nel repository come pacchetto separato e riconoscibile, contenente le macro Phoenix realmente attive e validate sulla macchina.

Il packaging deve:

- separare le macro Phoenix dai componenti esterni;
- mantenere intatte le attribuzioni upstream;
- evitare di importare file DKEU non più necessari;
- riflettere il runtime realmente utilizzato sulla stampante;
- permettere di individuare facilmente quali file appartengono al layer Phoenix;
- conservare la distinzione tra macro strutturalmente caricate e macro fisicamente validate.

I file `.cfg` effettivamente attivi sulla CB1 costituiscono la fonte runtime di riferimento e devono essere mantenuti sincronizzati con il packaging Phoenix pubblicato nel repository.

---

## Navigazione

- ← **Pagina precedente:** [Sovol Zero, CAN ed Eddy integrato](zero-toolhead-eddy-2026-08-17.md)
- → **Pagina successiva:** [Validazione e calibrazione](validation-and-calibration.md)
