# Getting started — Sovol SV08 to Klipper Mainline

Ultima verifica delle fonti: **2026-08-10**.

## Scopo

Questo repository documenta una migrazione reale e verificata di una **Sovol SV08** dal software Klipper modificato fornito da Sovol a **Klipper Mainline**, mantenendo un percorso di rollback e integrando successivamente:

- Moonraker;
- Mainsail;
- KlipperScreen, se utilizzato;
- Demon Klipper Essentials Unified (DKEU);
- hardware Sovol Eddy NG tramite supporto Eddy nativo di Klipper;
- OrcaSlicer.

L'obiettivo non è sostituire le guide upstream esistenti, ma fornire un secondo percorso documentato, basato su una macchina realmente migrata fino alla stampa.

Sono documentati anche:

- tentativi che non hanno funzionato;
- configurazioni legacy incompatibili con Mainline;
- sintomi osservati;
- root cause verificate;
- correzioni applicate;
- procedure di rollback.

## Fonte upstream principale

La guida primaria per la migrazione della SV08 resta:

`Rappetor/Sovol-SV08-Mainline`

Questo repository deve essere considerato complementare.

Quando una procedura upstream cambia, la documentazione corrente del progetto originale ha priorità rispetto a vecchi video, screenshot o copie locali.

## Hardware della macchina usata per la validazione

La migrazione descritta è stata verificata su una Sovol SV08 con:

- computer Linux e mainboard originali Sovol/MKS con eMMC;
- mainboard originale Sovol;
- toolboard originale Sovol;
- MCU STM32F103 su mainboard e toolboard;
- probe hardware Sovol Eddy NG / LDC1612;
- hotend MicroSwiss FlowTech;
- nozzle 0.4 mm.

Il FlowTech **non è un requisito** per la migrazione a Mainline.

Le modifiche hardware specifiche della macchina di test sono documentate separatamente sotto `examples/phoenix/`.

## Baseline software verificata

La macchina di validazione è attualmente operativa con:

- Klipper Mainline;
- Moonraker;
- Mainsail;
- KlipperScreen;
- Crowsnest;
- Demon Klipper Essentials Unified;
- Eddy gestito tramite `[probe_eddy_current eddy]`.

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

## Eddy: usare il supporto Mainline corrente

Klipper Mainline corrente dispone di supporto Eddy nativo con diversi metodi di probing, inclusi:

- default probing;
- scan;
- rapid scan;
- tap.

La macchina usata per questa migrazione utilizza il probe hardware Sovol Eddy NG / LDC1612 attraverso:

`[probe_eddy_current eddy]`

Parametri validati sulla macchina di test:

- `sensor_type: ldc1612`
- `i2c_mcu: extra_mcu`
- `i2c_scl: PB6`
- `i2c_sda: PB7`
- `x_offset: -16.43`
- `y_offset: 10.22`
- `descend_z: 0.5`
- `max_sensor_hz: 9000000`
- `reg_drive_current: 22`

Questi valori descrivono la macchina realmente testata.

Gli offset e le calibrazioni Eddy devono essere verificati sulla propria stampante e non copiati alla cieca.

Il vecchio percorso basato su una patch locale `probe_eddy_ng.py` non è il percorso raccomandato da questo repository per Klipper Mainline corrente.

Durante la migrazione della macchina di test quel percorso è stato inizialmente utilizzato e successivamente abbandonato in favore del supporto Eddy nativo Mainline.

La cronologia completa del passaggio è conservata sotto:

`docs/migration-history/phoenix/`

## Demon Klipper Essentials Unified

DKEU è un progetto attivo e in evoluzione.

La migrazione verificata utilizza Demon con Klipper Mainline ed Eddy nativo, ma questo repository non distribuisce una copia congelata delle macro Demon come sorgente primaria.

Installare e consultare la versione corrente del repository ufficiale:

`3DPrintDemon/Demon_Klipper_Essentials_Unified`

Al momento della verifica, DKEU è nella generazione DKEU3 e include gestione dedicata delle differenze fra Klipper Mainline moderno e versioni factory/legacy.

La configurazione Orca verificata sulla macchina di test utilizza il Machine G-code Demon `v1.4`.

Le configurazioni Phoenix specifiche sono conservate separatamente sotto:

`examples/phoenix/`

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
2. backup configurazioni e dati necessari;
3. preparazione del supporto CB1;
4. avvio del nuovo sistema Linux;
5. installazione Klipper, Moonraker e Mainsail;
6. preparazione della configurazione SV08;
7. backup firmware MCU originali;
8. installazione Katapult;
9. flash Klipper Mainline su mainboard e toolboard;
10. verifica temperature, MCU, ventole e segnali di sicurezza;
11. verifica assi ed endstop;
12. homing controllato;
13. integrazione Eddy nativo;
14. integrazione Demon;
15. QGL e mesh;
16. calibrazione Z;
17. prima stampa;
18. calibrazione slicer e materiale.

Le fasi hardware e di probing devono essere completate prima di considerare la macchina pronta alla stampa.

## Cosa NON copiare alla cieca dalla configurazione stock

Una configurazione funzionante sul Klipper Sovol non è automaticamente valida su Mainline.

Durante la migrazione reale sono stati trovati elementi legacy che causavano problemi, fra cui:

- vecchie integrazioni Eddy;
- macro scritte per API presenti nel Klipper Sovol ma non equivalenti in Mainline;
- configurazioni di axis twist non più affidabili dopo la migrazione;
- override locali che neutralizzavano macro Demon aggiornate.

## Caso verificato: `_APPLY_EDDY_Z_OFFSET`

Il problema più importante incontrato dopo la migrazione riguardava un vecchio override locale:

`[gcode_macro _APPLY_EDDY_Z_OFFSET]`

L'override era stato creato in precedenza per il Klipper Sovol 0.12, dove il percorso Demon aggiornato non era compatibile.

Dopo il passaggio a Mainline l'override era rimasto nel `printer.cfg`.

Il risultato era che la macro Demon corretta veniva neutralizzata e la posizione Z misurata dal probe non veniva applicata correttamente.

Il sintomo osservato era un first layer fortemente errato nonostante QGL e mesh apparentemente validi.

Rimuovendo l'override legacy, il percorso corretto è tornato operativo:

`_PROBE_TAP -> _APPLY_EDDY_Z_OFFSET -> printer.probe.last_probe_position.z -> SET_KINEMATIC_POSITION`

La ristampa dello stesso G-code ha mostrato un miglioramento netto e ha confermato la root cause.

Questo caso è conservato nella cronologia tecnica Phoenix e verrà riportato anche nella documentazione troubleshooting.

## Come leggere questa guida

Ogni passaggio dovrebbe essere interpretato secondo una delle seguenti categorie:

- **Upstream requirement** — richiesto o raccomandato dalla documentazione originale;
- **Phoenix verified** — verificato realmente sulla macchina usata per questo progetto;
- **Troubleshooting** — soluzione a un problema realmente incontrato;
- **Optional** — modifica non necessaria per una SV08 Mainline standard.

Questa distinzione serve a evitare che una personalizzazione della macchina di test venga scambiata per requisito generale.

## Passo successivo

Prima di installare software o flashare una MCU:

**creare e verificare il backup.**

Continua con:

`docs/backup-and-rollback.md`
