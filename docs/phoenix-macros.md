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

Phoenix usa ora `save_variables` per preferenze e stato termico persistente del proprio sistema Automatic Soak, tramite un file Phoenix dedicato. Il vecchio `~/demon_vars.cfg` non viene riutilizzato.

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

### Automatic Soak persistente

Aggiunto il 25 agosto 2026 e attualmente in validazione funzionale.

Macro utente:

- `PHOENIX_SOAK_AUTOSTART ENABLE=0|1`
- `PHOENIX_SOAK_TEMPERATURE BED_TEMP=<temperatura>`
- `PHOENIX_SOAK_START [BED_TEMP=<temperatura>]`
- `PHOENIX_SOAK_STOP`
- `PHOENIX_SOAK_STATUS`

Il sistema usa quattro variabili persistenti:

```text
phoenix_soak_autostart
phoenix_soak_temperature
phoenix_soak_credit
phoenix_soak_timestamp
```

salvate tramite:

```ini
[save_variables]
filename: ~/printer_data/config/phoenix_variables.cfg
```

`phoenix_soak_credit` rappresenta i secondi di storia termica valida accumulati dal bed; `phoenix_soak_timestamp` contiene il timestamp Unix dell'ultimo salvataggio del credito.

Per rendere disponibile alle macro un clock assoluto attraverso restart e riavvii di Klipper, Phoenix usa inoltre il piccolo helper Klipper:

```text
phoenix_clock.py
[phoenix_clock]
```

che espone a Jinja:

```text
printer.phoenix_clock.epoch
```

Il file storico `~/demon_vars.cfg` non viene usato dal sistema Phoenix.

#### Avvio automatico

Se `phoenix_soak_autostart` è abilitato, pochi secondi dopo l'avvio di Klipper Phoenix:

1. legge temperatura, credito e timestamp persistenti;
2. calcola il tempo trascorso dall'ultimo timestamp;
3. sottrae quel tempo dal credito precedentemente registrato;
4. verifica la temperatura reale del bed;
5. invalida il credito storico se il bed è sceso oltre la soglia termica ammessa;
6. imposta il target del bed;
7. attende il ritorno nella finestra utile di temperatura;
8. applica, quando esiste credito recuperato, un guard di sicurezza di 60 secondi;
9. continua ad accumulare credito termico;
10. persiste periodicamente credito e timestamp.

Il tempo trascorso a macchina spenta **non genera credito**: viene sottratto dal credito precedente.

Esempio:

```text
credito salvato: 120 s
tempo trascorso dal timestamp: 15 s
credito recuperato: 105 s
```

Il 25 agosto questo caso è stato verificato realmente con un `FIRMWARE_RESTART`: Phoenix ha riportato `recovered 105 s after 15 s offline`.

#### Finestra termica

Il credito non cresce durante la rampa iniziale. Con target 70 °C il conteggio avanza quando il bed è almeno a circa 69 °C.

Se il bed scende oltre 6 °C sotto la temperatura di riferimento, la storia termica precedente non viene più considerata affidabile e il credito viene azzerato.

La baseline minima per considerare un soak completo è 600 secondi, cioè 10 minuti effettivi nella finestra termica prevista.

Il credito, però, **non viene limitato a 600 secondi**: continua a rappresentare la durata reale dello stato termico. Un bed mantenuto in soak per un'ora può quindi registrare circa 3600 secondi di credito. Questo consente di attraversare restart brevi senza perdere artificialmente una lunga stabilizzazione già avvenuta.

#### Recovery guard da 60 secondi

Quando un credito precedente viene recuperato attraverso un restart, Phoenix non lo dichiara immediatamente valido. Il bed viene prima riportato alla temperatura target e deve trascorrere almeno altri 60 secondi nella finestra termica utile.

Il guard non sostituisce il requisito minimo dei 600 secondi. Per esempio, un credito recuperato di 105 secondi resta insufficiente anche dopo il minuto di sicurezza: il sistema continua in stato `TRACKING` fino al raggiungimento della soglia minima richiesta.

Test reale del 25 agosto:

```text
recovered: 105 s
status intermedio: credit 145 s | recovery guard 20 s
status successivo: credit 195 s | recovery guard 0 s
```

Il comportamento osservato corrisponde al modello previsto.

#### Persistenza e frequenza di scrittura

Il watcher runtime opera a intervalli di 10 secondi, mentre `phoenix_soak_credit` e `phoenix_soak_timestamp` vengono salvati su disco ogni 60 secondi. Questa scelta evita scritture inutilmente frequenti sulla memoria mantenendo un errore massimo conservativo di circa un minuto in caso di spegnimento improvviso.

Il tracking continua anche durante preparazione e stampa, finché il bed resta nella condizione termica prevista. In questo modo credito e timestamp rappresentano la storia reale del piatto e non soltanto la fase precedente alla stampa.

#### Stato termico runtime

Il sistema mantiene compatibilità con lo stato Phoenix esistente:

```text
_PHOENIX_THERMAL_STATE
```

con variabili runtime quali:

```text
soak_valid
soak_temp
soak_total_seconds
thermal_credit_seconds
```

Lo stato operativo del nuovo motore è mantenuto anche in:

```text
_PHOENIX_SOAK_AUTOSTART_STATE
```

che tiene, tra gli altri:

```text
active
target
credit_seconds
recovery_guard_seconds
persist_counter
```

`PHOENIX_SOAK_STATUS` mostra:

- Automatic Soak abilitato o disabilitato;
- stato `INACTIVE`, `TRACKING` o `VALID`;
- temperatura configurata;
- temperatura reale e target del bed;
- credito termico corrente;
- eventuale recovery guard residuo.

`PHOENIX_SOAK_STOP` interrompe il soak corrente e spegne il bed senza cambiare la preferenza persistente di autostart; prima di fermarsi salva la storia termica corrente.

`PHOENIX_SOAK_START` permette l'avvio manuale usando la temperatura persistente oppure un override temporaneo tramite `BED_TEMP`.

#### Integrazione con `PHOENIX_START`

Il soak è ora una **fase autonoma della macchina**, separata dalla preparazione stampa.

`PHOENIX_START` esegue due fasi distinte:

**Fase 1 — stato termico del bed**

1. legge il credito termico compatibile con il target richiesto;
2. porta il bed alla temperatura richiesta;
3. completa solo l'eventuale tempo di soak ancora necessario;
4. soddisfa l'eventuale recovery guard;
5. registra lo stato termico come valido.

Durante questa fase il nozzle non viene usato per costruire il soak.

**Fase 2 — preparazione stampa**

Solo dopo che il bed ha soddisfatto lo stato termico richiesto:

1. porta il nozzle alla temperatura di preparazione;
2. esegue homing;
3. esegue `PHOENIX_CLEAN_NOZZLE`;
4. raffredda il nozzle prima di QGL/mesh;
5. esegue QGL;
6. esegue nuovamente homing Z;
7. genera la bed mesh adaptive tramite Klipper Mainline;
8. porta il nozzle alla temperatura finale;
9. esegue la purge line;
10. avvia la stampa.

Questa separazione evita che cleaner, riscaldamento o raffreddamento del nozzle facciano parte artificiosamente del tempo di soak.

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

Gestisce il workflow di avvio stampa Phoenix.

La sequenza corrente è ora esplicitamente separata tra stato termico del bed e preparazione stampa:

1. imposta il target del bed;
2. usa il credito termico compatibile già disponibile;
3. attende esclusivamente l'eventuale soak residuo;
4. marca il soak come valido;
5. solo a questo punto porta il nozzle alla temperatura di preparazione;
6. esegue homing;
7. esegue `PHOENIX_CLEAN_NOZZLE`;
8. spegne il riscaldamento nozzle e lo raffredda;
9. attende nozzle <= 50 °C;
10. esegue QGL;
11. esegue nuovamente homing Z;
12. genera la bed mesh tramite Klipper Mainline;
13. porta il nozzle alla temperatura finale di stampa;
14. esegue la purge line;
15. avvia la stampa.

La bed mesh usa il percorso nativo Klipper con rapid scan e adaptive meshing.

Non viene caricato alcun wrapper DKEU per `BED_MESH_CALIBRATE`.

## `PHOENIX_END`

Gestisce la conclusione della stampa secondo il workflow Phoenix.

La macro appartiene al layer Phoenix e sostituisce l'uso delle precedenti macro di fine stampa DKEU.

Non viene eseguito `M84`.

Il nuovo watcher persistente del soak non viene riavviato da `PHOENIX_END`: resta autonomo e continua a rappresentare la storia termica del bed quando applicabile.

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

La gestione termica Phoenix distingue esplicitamente:

- temperatura target del bed;
- credito termico persistente;
- timestamp assoluto dell'ultimo salvataggio valido;
- decadimento del credito tra restart;
- recovery guard da 60 secondi;
- temperatura di preparazione dell'ugello;
- raffreddamento dell'ugello prima del QGL;
- temperatura finale di stampa.

Il comportamento termico non dipende da variabili o framework DKEU. Persistenza e clock introdotti il 25 agosto appartengono esclusivamente al layer Phoenix.

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
- helper `phoenix_clock` caricato e timestamp Unix leggibile da Jinja;
- lettura corretta di `phoenix_soak_autostart`, `phoenix_soak_temperature`, `phoenix_soak_credit` e `phoenix_soak_timestamp`;
- avvio automatico del bed con target configurato a 70 °C;
- nessun credito accumulato durante la rampa termica;
- incremento del credito a passi di 10 secondi nella finestra utile;
- persistenza del credito e del timestamp ogni 60 secondi;
- recupero reale dopo `FIRMWARE_RESTART`: 120 s salvati, 15 s trascorsi, 105 s recuperati;
- applicazione reale del recovery guard da 60 secondi;
- decremento osservato del guard da 60 a 20 fino a 0 mentre il credito continuava a crescere;
- stato correttamente mantenuto in `TRACKING` con guard terminato ma credito ancora inferiore a 600 s;
- separazione del soak dalla preparazione nozzle in `PHOENIX_START` implementata e caricata.

Restano da completare prima di considerare il nuovo sistema definitivamente validato:

- osservare il nuovo motore persistente raggiungere `VALID` con credito >= 600 s;
- test fisico completo della nuova sequenza `PHOENIX_START` dopo la separazione bed-only/preparazione stampa;
- test operativo di `PHOENIX_SOAK_START` e `PHOENIX_SOAK_STOP` nel motore persistente definitivo.

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
