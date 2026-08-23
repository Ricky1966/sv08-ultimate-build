# Getting started — Sovol SV08 to Klipper Mainline

Ultima verifica delle fonti: **2026-08-10**.

## Scopo

Questo repository documenta una migrazione reale e verificata di una **Sovol SV08** dal software Klipper modificato fornito da Sovol a **Klipper Mainline**, mantenendo un percorso di rollback e integrando successivamente:

- Moonraker;
- Mainsail;
- KlipperScreen, se utilizzato;
- Phoenix Macros;
- Sovol Zero Extruder Kit con probe Eddy integrato, gestito tramite supporto Eddy nativo di Klipper;
- OrcaSlicer.

L'obiettivo non è sostituire le guide upstream esistenti, ma fornire un mio percorso documentato, basato su una macchina realmente migrata fino alla stampa.

Sono documentati anche:

- tentativi che non hanno funzionato;
- configurazioni legacy incompatibili con Mainline;
- sintomi osservati;
- cause dei problemi effettivamente verificate;
- correzioni applicate;
- procedure di rollback.

## Fonte upstream principale

La guida primaria per la migrazione della SV08 resta:

[Rappetor/Sovol-SV08-Mainline](https://github.com/Rappetor/Sovol-SV08-Mainline)

Questo repository deve essere considerato complementare.

Quando una procedura upstream cambia, la documentazione corrente del progetto originale ha priorità rispetto a vecchi video, screenshot o copie locali.

## Hardware della macchina usata per la validazione

La migrazione descritta è stata verificata su una Sovol SV08 con:

- computer Linux Sovol/MKS;
- sistema di migrazione e validazione avviato da **MicroSD**;
- mainboard originale Sovol;
- **Sovol Zero Extruder Kit**;
- MCU STM32F103 sulla mainboard;
- **probe Eddy integrato nella Sovol Zero**, gestito tramite supporto Eddy nativo di Klipper;
- nozzle 0.4 mm.

Le modifiche hardware specifiche della macchina di test sono documentate separatamente sotto `examples/phoenix/`.

## Baseline software verificata

La macchina di validazione è attualmente operativa con:

- Klipper Mainline;
- Moonraker;
- Mainsail;
- KlipperScreen;
- Crowsnest;
- Phoenix Macros;
- Eddy integrato nella Sovol Zero e gestito tramite `[probe_eddy_current eddy]`.

Configurazione Klipper validata al termine della migrazione:

- versione: `v0.13.0-718-gd8659974-dirty`;
- commit: `d865997403cad36d105026f73a4b76dcacec4c76`.

Questi identificativi descrivono la configurazione realmente testata e **non significano che un nuovo utente debba installare esattamente quel commit**.

## Immagine CB1


Tuttavia la guida `Rappetor/Sovol-SV08-Mainline` continua a indicare esplicitamente **CB1 V2.3.4** come baseline per la SV08 e sconsiglia attualmente V3.0.0 o successive per questa procedura.

Per questo repository:

**CB1 V2.3.4 è la baseline documentata e verificata.**

Non interpretare "latest CB1 release" come "versione raccomandata per SV08 Mainline".

Una futura immagine verrà adottata solo dopo una nuova verifica end-to-end sulla SV08.

## Eddy: supporto nativo Klipper Mainline sulla Sovol Zero

La configurazione Phoenix utilizza il probe Eddy integrato nella **Sovol Zero Extruder Kit**, gestito direttamente tramite il supporto Eddy nativo di Klipper Mainline.

La configurazione utilizza:

`[probe_eddy_current eddy]`

Klipper Mainline supporta diversi metodi di probing Eddy, inclusi:

- probing standard;
- scan;
- rapid scan;
- tap.

La configurazione specifica della Sovol Zero e i parametri Eddy validati sulla Phoenix sono documentati nella pagina dedicata:

`docs/zero-toolhead-eddy-2026-08-17.md`

Questi valori descrivono **la configurazione Phoenix realmente testata** e non devono essere considerati preset universali.

Offset, calibrazione, curva Eddy e comportamento del probe devono essere verificati sulla propria macchina.

Durante una fase precedente dello sviluppo Phoenix era stato utilizzato un percorso esterno basato su `probe_eddy_ng.py`. Questo approccio è stato successivamente abbandonato dopo la migrazione al supporto Eddy nativo di Klipper Mainline.

La cronologia tecnica completa resta disponibile sotto:

`docs/migration-history/phoenix/`

## Phoenix Macros

La configurazione attuale utilizza un insieme autonomo di macro sviluppato durante la migrazione Phoenix.

Le **Phoenix Macros** sono state progressivamente derivate, riscritte e adattate durante il lavoro svolto su Klipper Mainline, Eddy nativo e sulla configurazione reale della macchina.

Demon Klipper Essentials Unified (DKEU) ha avuto un ruolo importante durante una fase dello sviluppo ed è riconosciuto come progetto upstream di riferimento, ma **non è più una dipendenza runtime della configurazione Phoenix attuale**.

La baseline corrente:

- non include DKEU;
- non dipende da macro DKEU durante il funzionamento;
- utilizza macro `PHOENIX_*` dedicate;
- utilizza il bed mesh nativo di Klipper con rapid scan e adaptive meshing;
- integra direttamente il flusso di stampa con OrcaSlicer.

Le Phoenix Macros comprendono funzioni per:

- avvio e fine stampa;
- pulizia nozzle;
- caricamento, scaricamento e cambio filamento;
- gestione runout;
- calibrazioni;
- diagnostica di probe, mesh, sensori e stepper;
- procedure di setup della macchina.

Le macro esposte e caricate da Klipper non devono essere confuse con quelle già fisicamente validate: la documentazione di validazione distingue esplicitamente i due stati.

Vedi:

`docs/phoenix-macros.md`

## Prima di iniziare

Non iniziare la migrazione senza avere:

1. backup completo di `printer_data/config`;
2. copia dei file e macro personalizzati;
3. backup o piano di ripristino della eMMC originale;
4. piano di rollback verificabile;
5. ST-Link disponibile per mainboard e toolboard;
6. possibilità di identificare senza ambiguità le due MCU;
7. accesso SSH;
8. una microSD o eMMC utilizzabile per il nuovo sistema;
9. tempo per verificare la macchina prima di eseguire homing o riscaldamenti.

## Regola fondamentale: preservare il sistema stock

Il percorso più sicuro è mantenere intatto un sistema stock avviabile fino a quando Mainline non è stato verificato end-to-end.

Non considerare un semplice backup dei file `.cfg` equivalente a un rollback completo.

Prima di modificare o flashare le MCU è fortemente raccomandato avere una copia verificata del firmware originale della propria mainboard e della propria toolboard.

I dump originali della macchina usata per questo progetto non vengono distribuiti nella baseline community.

Ogni utente deve salvare i propri backup.

## Ordine generale della migrazione

Il percorso documentato è:

1. inventario della macchina stock;
2. backup completo di configurazione, dati e possibilità di rollback;
3. preparazione della **MicroSD** con immagine CB1;
4. primo avvio del nuovo sistema Linux e verifica SSH/rete;
5. installazione di KIAUH, Klipper Mainline, Moonraker, Mainsail e componenti necessari;
6. preparazione della configurazione base SV08;
7. backup dei firmware MCU originali;
8. installazione di Katapult e flashing Mainline delle MCU interessate;
9. verifica delle MCU, temperature, endstop, ventole e uscite prima di qualsiasi movimento;
10. installazione e configurazione della **Sovol Zero Extruder Kit** e del relativo collegamento CAN;
11. configurazione del **probe Eddy integrato nella Zero** tramite supporto nativo Klipper;
12. verifica controllata degli assi e degli endstop;
13. homing controllato;
14. calibrazione e validazione Eddy;
15. QGL;
16. bed mesh / rapid scan e verifica della compensazione;
17. calibrazione/riferimento Z e validazione del primo layer;
18. installazione e configurazione delle **Phoenix Macros**;
19. integrazione con OrcaSlicer;
20. verifica completa del flusso `PHOENIX_START` → stampa → `PHOENIX_END`;
21. prima stampa reale;
22. calibrazioni del filamento e dello slicer;
23. validazione delle funzioni accessorie e diagnostiche.

Le fasi hardware, MCU, probing e sicurezza devono essere completate prima di considerare la macchina pronta alla stampa.

## Cosa NON copiare alla cieca dalla configurazione stock

Una configurazione funzionante sul Klipper Sovol non è automaticamente valida su Mainline.

Durante la migrazione reale sono stati trovati elementi legacy che causavano problemi, fra cui:

- vecchie integrazioni Eddy;
- macro scritte per API presenti nel Klipper Sovol ma non equivalenti in Mainline;
- configurazioni di axis twist non più affidabili dopo la migrazione;
- override locali e residui di configurazioni precedenti non più compatibili con la baseline Phoenix.

## Come leggere questa guida

Ogni passaggio dovrebbe essere interpretato secondo una delle seguenti categorie:

- **Upstream requirement** — richiesto o raccomandato dalla documentazione originale;
- **Phoenix verified** — verificato realmente sulla macchina usata per questo progetto;
- **Troubleshooting** — soluzione a un problema realmente incontrato;
- **Optional** — modifica non necessaria per una SV08 Mainline standard.

Questa distinzione serve a evitare che una personalizzazione della macchina di test venga scambiata per requisito generale.

---

## Navigazione

← **Pagina precedente:** [README](../README.md)
→ **Pagina successiva:** [Backup and rollback](backup-and-rollback.md)
