# DKEU / Demon integration — fase storica dello sviluppo Phoenix

Ultima verifica delle fonti storiche: **2026-08-10**.

> [!IMPORTANT]
> Questa pagina documenta una **fase precedente dello sviluppo Phoenix**.
>
> DKEU ha avuto un ruolo importante nel percorso di migrazione, ma **non è più una dipendenza runtime della configurazione Phoenix attuale**.
>
> La baseline corrente utilizza le [Phoenix Macros](phoenix-macros.md).

## Scopo storico

Questa guida conserva la documentazione dell'integrazione di Demon Klipper Essentials Unified utilizzata durante una fase della migrazione Phoenix, quando la macchina era già migrata a:

- Klipper Mainline;
- MCU Mainline;
- Eddy nativo;
- homing funzionante;
- QGL funzionante;
- probing verificato.

Demon deve essere aggiunto dopo che la configurazione hardware di base è stabile.

Non usare Demon come strumento per nascondere problemi ancora presenti in:

- MCU;
- stepper;
- endstop;
- heater;
- homing;
- probe.

## Fonte upstream

Repository ufficiale:

`3DPrintDemon/Demon_Klipper_Essentials_Unified`

Al momento della verifica il progetto è nella generazione:

`DKEU3`

DKEU è un progetto attivo.

Per chi desidera utilizzare DKEU indipendentemente da Phoenix, usare sempre la documentazione e i file correnti upstream.

Non utilizzare una vecchia copia Phoenix come sorgente primaria.

## DKEU corrente e Klipper Mainline

La versione DKEU corrente distingue automaticamente fra:

- Klipper Mainline moderno;
- versioni Klipper legacy/factory.

Questo rende obsolete alcune modifiche manuali che in passato erano necessarie sulle stampanti Sovol stock.

Non impostare variabili legacy sulla base di vecchie guide se la versione DKEU corrente non le richiede più.

## Prerequisiti

Prima di installare Demon devono funzionare:

- Mainsail;
- Moonraker;
- Klipper Mainline;
- mainboard MCU;
- toolboard MCU;
- homing X/Y/Z;
- Eddy nativo;
- QGL;
- probing;
- temperature;
- ventole.

La macchina deve essere controllabile anche senza Demon.

## Installazione

Installare DKEU seguendo il metodo corrente indicato dal progetto ufficiale.

Non copiare manualmente soltanto alcuni file da una vecchia installazione Phoenix.

DKEU utilizza un ecosistema di:

- core assets;
- user files;
- variabili;
- helper;
- eventuali shell script;
- configurazione slicer.

Una installazione parziale può lasciare macro apparentemente presenti ma internamente incoerenti.

## KIAUH

La documentazione DKEU corrente richiede particolare attenzione anche alla versione KIAUH e alla relativa shell script extension.

Se le operazioni File Handler di Demon non funzionano:

- verificare KIAUH;
- verificare la shell extension;
- non diagnosticare immediatamente un problema Klipper.

## Configurazione utente

DKEU separa i file core dai file utente.

Le personalizzazioni devono essere fatte nei file previsti dal progetto.

Non modificare direttamente i core assets salvo debugging temporaneo e controllato.

Una modifica locale ai core:

- può essere sovrascritta da un update;
- può impedire di capire se un problema appartiene a Demon o alla personalizzazione;
- può rendere più difficile confrontare la configurazione con upstream.

## Variabili Phoenix osservate

Durante la validazione Phoenix erano attivi valori equivalenti a:

- Orca integration disattivata/attivata secondo fase di test;
- adaptive meshing attivo;
- KAMP smart park attivo;
- slicer flow non usato da Demon in quella fase;
- slicer pressure advance non usato da Demon in quella fase;
- purge lines abilitate;
- nozzle cleaner presente;
- Klicky probe assente;
- bed fans assenti;
- chamber heater assente;
- Nevermore assente.

Questi valori rappresentano la configurazione Phoenix del momento.

Non sono una baseline universale DKEU.

## OrcaSlicer

DKEU corrente richiede una configurazione Machine G-code coerente con la versione installata.

La migrazione Phoenix è stata validata con Machine G-code:

`v1.4`

La stringa di start verificata conteneva:

`DEMON_START`

e:

`DMGCC="v1.4"`

insieme ai parametri forniti da OrcaSlicer.

## Importanza del profilo macchina corretto

Durante la migrazione Phoenix il primo avvio di stampa con Demon fallì perché OrcaSlicer era rimasto selezionato sulla macchina Sovol invece che sulla macchina Phoenix.

Il G-code prodotto non conteneva il Machine Start Demon atteso.

Demon eseguì quindi un emergency shutdown.

La causa non era:

- Eddy;
- Klipper;
- MCU;
- QGL.

Era il profilo macchina sbagliato nello slicer.

Prima di diagnosticare Demon verificare sempre:

- printer preset;
- filament preset;
- process preset;
- Machine Start G-code.

## `_APPLY_EDDY_Z_OFFSET`

DKEU corrente contiene una macro `_APPLY_EDDY_Z_OFFSET` che utilizza:

`printer.probe.last_probe_position.z`

per riallineare la coordinata Z tramite:

`SET_KINEMATIC_POSITION`

Questo è il comportamento corretto verificato sulla Phoenix con Klipper Mainline.

Non ridefinire localmente `_APPLY_EDDY_Z_OFFSET` senza una necessità dimostrata.

## Problema storico Phoenix — override legacy

Prima della migrazione Mainline, sulla Phoenix era stato creato un override locale per neutralizzare `_APPLY_EDDY_Z_OFFSET`.

Era necessario sul vecchio Klipper Sovol perché il campo usato da Demon non era disponibile.

Dopo il passaggio a Mainline quell'override era diventato dannoso.

Il blocco locale sostituiva la macro Demon corrente e impediva il corretto riallineamento Z dopo probing.

## Effetto osservato

Durante il primo print fallito `_PROBE_TAP` aveva stimato il contatto a circa:

`z=-0.060846`

ma il vecchio override non applicava la correzione.

L'errore risultante di circa:

`0.061 mm`

era sufficiente a compromettere fortemente un first layer da `0.20 mm`.

## Correzione

La soluzione non è stata modificare Demon.

È stato rimosso il vecchio override locale dal `printer.cfg`.

Dopo il restart, la macro DKEU attiva ha ripreso a usare correttamente:

`printer.probe.last_probe_position.z`

e:

`SET_KINEMATIC_POSITION`

La ristampa dello stesso G-code ha mostrato un miglioramento radicale.

## Regola di migrazione

Quando si passa da Klipper Sovol a Mainline:

controllare sempre se `printer.cfg` contiene macro che ridefiniscono macro DKEU.

In particolare cercare:

- `_APPLY_EDDY_Z_OFFSET`
- `_PROBE_TAP`
- `_MESH_HANDLING`
- macro homing;
- macro probe;
- wrapper QGL;
- wrapper bed mesh.

Una vecchia ridefinizione locale può avere priorità sulla versione aggiornata installata da Demon.

## TAP — stato corrente

DKEU corrente riconosce il supporto Eddy nativo e gestisce la presenza di `tap_threshold` attraverso la configurazione effettivamente caricata.

Durante la migrazione Phoenix era stata necessaria una correzione temporanea perché la versione DKEU allora installata interpretava male il valore di default del supporto Mainline.

Quella patch appartiene alla cronologia Phoenix.

Non deve essere applicata automaticamente alle versioni DKEU correnti.

## `tap_threshold`

La semantica del TAP Eddy Mainline è cambiata nel tempo.

Non recuperare vecchi valori come:

`tap_threshold: 300`

da configurazioni Eddy NG precedenti.

Se si vuole utilizzare TAP:

- seguire la documentazione Klipper corrente;
- eseguire la calibrazione prevista;
- verificare il valore prodotto sulla propria macchina.


## QGL e mesh sotto Demon

Demon può avvolgere e coordinare operazioni come:

- homing;
- QGL;
- mesh;
- heat soak;
- purge;
- start print.

Questo significa che un test manuale eseguito direttamente con un comando Klipper nativo non è necessariamente identico al percorso usato durante `DEMON_START`.

Durante il debugging è quindi importante distinguere:

- comportamento Klipper nativo;
- comportamento wrapper Demon;
- comportamento completo dello start print.

## Test nativo contro wrapper

Durante la migrazione Phoenix è stato utile eseguire anche comandi nativi per isolare il comportamento del probe e della mesh.

In particolare, le macro Demon potevano forzare un percorso `rapid_scan`.

Quando serve verificare Klipper senza l'intermediazione del wrapper, utilizzare il comando base/alias previsto dalla propria configurazione.

Non modificare una configurazione meccanica solo perché il comportamento dentro Demon differisce da un test manuale.

Prima ricostruire il percorso macro effettivamente eseguito.

## Adaptive meshing

DKEU può gestire adaptive meshing.

Sulla Phoenix questa funzione è stata utilizzata durante il workflow Demon.

Prima di abilitarla verificare che:

- probing nativo funzioni;
- mesh manuale funzioni;
- coordinate probe siano corrette;
- area mesh sia compatibile con gli offset;
- slicer trasmetta le informazioni richieste.

Non usare adaptive meshing come primo test del probe.

## Purge lines

Le purge lines devono essere abilitate solo dopo che:

- Z è affidabile;
- start G-code è corretto;
- nozzle cleaning è sicuro;
- coordinate di purge sono fisicamente raggiungibili.

Una purge line non deve diventare il primo movimento che rivela un errore di Z.

## Nozzle cleaner

La Phoenix utilizza un nozzle cleaner fisico.

Questo è hardware specifico e non un requisito DKEU.

Se presente, verificare:

- posizione;
- altezza;
- ingombro toolhead;
- percorso sicuro;
- temperatura di pulizia;
- assenza di collisioni.

Le coordinate Phoenix non devono essere copiate su una SV08 stock o su una toolhead diversa.

## Primo `DEMON_START`

Prima del primo start reale verificare separatamente:

- `G28`;
- probing;
- QGL;
- mesh;
- heater;
- ventole;
- nozzle cleaner, se presente;
- purge, se presente.

Solo dopo testare il percorso completo `DEMON_START`.

Questo riduce il numero di possibili cause quando qualcosa fallisce.

## Ricostruire lo start dai log

Se una stampa fallisce appena dopo `DEMON_START`, non modificare subito la macchina.

Ricostruire dal log:

1. homing eseguito;
2. probe eseguito;
3. Z risultante;
4. QGL;
5. mesh caricata o creata;
6. eventuale riallineamento Z;
7. heater;
8. purge;
9. inizio effettivo del primo layer.

Questo approccio ha permesso di identificare la root cause Phoenix senza intervenire inutilmente sulla meccanica.

## Phoenix verified — sequenza di diagnosi

Nel caso Phoenix:

- QGL manuale era stabile;
- mesh manuale era stabile;
- probing era coerente;
- il primo layer durante Demon risultava comunque errato.

Questo ha spostato correttamente l'attenzione dal sistema meccanico al percorso macro.

La verifica del log ha poi mostrato che `_PROBE_TAP` misurava correttamente il contatto ma l'override legacy impediva l'applicazione dell'offset.

## Osservazione non provata — `_MESH_HANDLING`

Durante la diagnosi Phoenix è stato notato un possibile problema logico nella condizione di rilevamento probe della macro `_MESH_HANDLING`.

La condizione osservata utilizzava `or` fra controlli di assenza.

Questo poteva teoricamente rendere vera la condizione anche quando uno dei probe supportati era presente.

Non è stata dimostrata come causa del problema di stampa.

Non è stata modificata.

Per installazioni nuove:

- verificare il comportamento della versione DKEU corrente;
- non applicare patch basate su questa osservazione storica senza una riproduzione concreta.

## Osservazione non provata — rilevamento Eddy

In uno snapshot DKEU utilizzato durante la migrazione Phoenix era stata osservata una condizione che considerava esplicitamente:

`probe_eddy_current btt_eddy`

ma non sempre:

`probe_eddy_current eddy`

Anche questo punto non è stato dimostrato come causa del print fallito.

Non deve essere presentato come bug certo.

La versione DKEU corrente va verificata direttamente prima di qualunque modifica.

## Aggiornamenti DKEU

DKEU evolve rapidamente.

Dopo un update:

- leggere le note upstream;
- verificare eventuali rinomini di variabili;
- verificare Machine G-code richiesto;
- verificare modifiche ai file user;
- verificare compatibilità Eddy/TAP;
- eseguire un test controllato prima di una stampa lunga.

Non assumere che una configurazione perfetta con una versione precedente resti automaticamente valida dopo un update.

## Non congelare i core assets nel repository community

Questo repository non deve diventare una copia statica di DKEU.

La baseline community deve:

- riferirsi al repository upstream;
- documentare i parametri SV08;
- documentare le incompatibilità incontrate;
- conservare esempi Phoenix separati.

I file core Demon appartengono al progetto Demon.

## Validazione Orca

Prima della prima stampa reale verificare che il G-code generato contenga i marker e i parametri richiesti dalla versione DKEU installata.

Per la configurazione Phoenix validata erano presenti:

- `DEMON_START`
- `DMGCC="v1.4"`
- `_SPS GSTART=True`

Questi dati descrivono il workflow verificato nel periodo della migrazione.

Per installazioni future controllare sempre la documentazione DKEU corrente.

## Prima stampa di validazione

Usare una stampa:

- breve;
- conosciuta;
- con first layer facilmente osservabile;
- senza modificare contemporaneamente materiale, meccanica e macro.

Se possibile, quando si corregge un problema ristampare lo stesso G-code.

Questo permette un confronto A/B significativo.

La Phoenix ha utilizzato proprio questo metodo per confermare la correzione del riallineamento Z.

## Non correggere più variabili insieme

Se il first layer è errato, evitare di cambiare contemporaneamente:

- Z offset;
- flow;
- QGL;
- mesh;
- motori Z;
- temperature;
- pressure advance;
- macro Demon.

Cambiare una variabile alla volta dopo aver identificato il percorso effettivo dell'errore.

## Criterio di uscita

L'integrazione Demon può essere considerata riuscita quando:

- DKEU corrente è installato senza errori;
- file user separati dai core;
- Klipper Mainline rilevato correttamente;
- Eddy nativo rilevato;
- `G28` funziona;
- `_PROBE_TAP` funziona;
- `_APPLY_EDDY_Z_OFFSET` applica il riallineamento corretto;
- QGL funziona;
- mesh funziona;
- Machine G-code Orca è coerente con DKEU;
- `DEMON_START` completa il proprio workflow;
- il primo layer reale è coerente;
- nessun override legacy Sovol neutralizza macro DKEU correnti.

---

## Navigazione

← **Pagina precedente:** [Sovol Zero, CAN ed Eddy integrato](zero-toolhead-eddy-2026-08-17.md)
→ **Pagina successiva:** [Phoenix Macros](phoenix-macros.md)
