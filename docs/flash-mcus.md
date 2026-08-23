# Flash SV08 MCUs — Katapult and Klipper Mainline

Ultima verifica delle fonti: **2026-08-23**.

> [!IMPORTANT]
> Questa pagina documenta il flashing delle **MCU originali presenti durante la migrazione iniziale della SV08 stock**.
>
> La configurazione Phoenix attuale utilizza successivamente una **Sovol Zero Extruder Kit collegata via CAN**. La Zero non deve essere confusa con la toolboard USB originale descritta in questa pagina.
>
> **Convenzione di questa pagina:** salvo dove specificato diversamente, ogni riferimento a “toolboard” indica la **toolboard originale Sovol USB della macchina stock**, non la MCU CAN della Sovol Zero.

## Scopo

Questa fase porta le due MCU originali della Sovol SV08 dal firmware stock a:

1. Katapult come bootloader;
2. Klipper Mainline come applicazione.

Nella configurazione stock descritta in questa fase, la SV08 possiede due MCU distinte da trattare separatamente:

- mainboard MCU;
- toolboard originale MCU.

Non procedere finché non sono stati completati:

- [Backup and rollback](backup-and-rollback.md)
- [Installazione CB1 e Klipper Mainline](install-cb1-mainline.md)

## Fonti upstream

Il percorso segue principalmente:

[Rappetor/Sovol-SV08-Mainline](https://github.com/Rappetor/Sovol-SV08-Mainline)

e il progetto ufficiale:

[Arksine/katapult](https://github.com/Arksine/katapult)

Le istruzioni upstream correnti devono avere priorità su vecchi screenshot, video o copie locali.

## Regola fondamentale

Non flashare mai una MCU se non sai con certezza quale scheda stai programmando.

Mainboard e toolboard devono essere:

- identificate separatamente;
- salvate separatamente;
- programmate separatamente;
- verificate separatamente.

Scambiare mainboard e toolboard può produrre una macchina non avviabile o una configurazione apparentemente corretta ma associata alla MCU sbagliata.

## Hardware MCU verificato sulla Phoenix

Durante la migrazione iniziale, la macchina Phoenix utilizzava, sia sulla mainboard sia sulla toolboard originale:

- MCU `STM32F103`;
- cristallo 8 MHz;
- comunicazione USB su `PA11/PA12`.

Questi parametri sono stati verificati sulla macchina Phoenix.

Prima di copiarli su una macchina diversa, verificare che la revisione hardware sia compatibile.

## Katapult

In questa fase stock, Katapult viene installato su entrambe le MCU originali.

Configurazione Katapult validata sulla Phoenix:

- processor: `STM32F103`;
- clock reference: 8 MHz crystal;
- communication interface: USB;
- USB pins: `PA11/PA12`;
- application offset: 8 KiB;
- application start: `0x08002000`.

Il valore fondamentale è l'offset da 8 KiB.

Il firmware Klipper compilato successivamente deve utilizzare un bootloader offset corrispondente.

## Perché usare Katapult

Dopo il primo caricamento tramite programmatore, Katapult permette di aggiornare Klipper senza dover utilizzare ogni volta ST-Link.

Katapult supporta flashing tramite diversi trasporti, inclusi USB, UART e CAN.

Durante la migrazione originale documentata in questa pagina, Phoenix utilizzava USB per mainboard e toolboard originale.

## Installazione sorgente Katapult

Repository upstream:

[Arksine/katapult](https://github.com/Arksine/katapult)

Il normale processo upstream consiste in:

- clone del repository;
- configurazione tramite `make menuconfig`;
- compilazione tramite `make`.

Non utilizzare automaticamente un vecchio binario Katapult solo perché ha funzionato in passato.

Per una nuova installazione è preferibile compilare dal sorgente corrente verificato, conservando i parametri hardware corretti.

## Phoenix verified — Katapult usato nella migrazione originale

Durante la migrazione Phoenix del 2026-08-02 venne utilizzato:

`Arksine/katapult`

al commit:

`ec59b9b`

I due binari Katapult prodotti per mainboard e toolboard risultavano identici e avevano SHA256:

`6e8547d966271233cc7247569eeb1075deb724d02f7890e4b4611a43dfd941a0`

Questo dato serve esclusivamente come riferimento storico della migrazione verificata.

Non costituisce un requisito per installazioni future.

## Configurazione Klipper — mainboard

Configurazione verificata sulla Phoenix:

- MCU `STM32F103`;
- bootloader offset 8 KiB;
- cristallo 8 MHz;
- USB `PA11/PA12`;
- GPIO iniziali `PA1,PA3`.

Il firmware prodotto durante la migrazione Phoenix aveva SHA256:

`9182930d07326c7f9ca031ac03e735a044fe59eb4b5cb55a00167aeff0ccb8e8`

## Configurazione Klipper — toolboard originale (fase storica)

Configurazione verificata sulla Phoenix:

- MCU `STM32F103`;
- bootloader offset 8 KiB;
- cristallo 8 MHz;
- USB `PA11/PA12`;
- GPIO iniziale `PA6`.

Il firmware prodotto durante la migrazione Phoenix aveva SHA256:

`8db953d31271dd7999eb80f193637ff08f31f4e73f048290040d00dc6c71ee70`

La build originale Phoenix conteneva inoltre il supporto Eddy NG allora utilizzato.

Questo dettaglio appartiene alla cronologia della migrazione e **non descrive la toolhead Phoenix attuale**.

La configurazione corrente utilizza la Sovol Zero via CAN con Eddy integrato ed è documentata in:

[Sovol Zero toolhead, CAN ed Eddy integrato](zero-toolhead-eddy-2026-08-17.md)

## Backup firmware originale PRIMA di Katapult

Prima di cancellare o programmare una MCU leggere e salvare il firmware originale della propria macchina.

Creare due file distinti:

- firmware originale mainboard;
- firmware originale toolboard stock.

Non usare come unico backup i file original firmware presenti in repository di terzi.

Un dump fornito da un'altra macchina non è equivalente al firmware letto dalla propria MCU.

## Dimensioni dei backup Phoenix

Sulla macchina Phoenix i dump originali verificati avevano dimensione:

- mainboard: 524288 byte;
- toolboard: 131072 byte.

SHA256:

- mainboard:
  `911caf60ac216c6fbd8d9bd7211b77981a79c44775f9ad056b9446c7616393c8`

- toolboard:
  `3e5b75af609e972ea5c45f2d63a12412b683ee8db78accc0073814358beebf27`

Questi checksum identificano esclusivamente i backup della macchina Phoenix.

Non devono essere usati per validare il firmware originale di un'altra SV08.

## Identificare mainboard e toolboard stock

Prima della migrazione le due MCU Phoenix esponevano lo stesso identificativo USB generico.

Per questo motivo il semplice `/dev/serial/by-id/` non era sufficiente a distinguerle con certezza.

Nella fase stock sono stati usati i percorsi fisici:

`/dev/serial/by-path/`

per identificare quale collegamento corrispondesse alla mainboard e quale alla toolboard.

Annotare questa associazione prima di qualsiasi flashing.

## Dopo il nuovo firmware

Dopo l'installazione Klipper Mainline, utilizzare preferibilmente gli identificativi stabili:

`/dev/serial/by-id/`

Aggiornare quindi `printer.cfg` verificando attentamente che:

- `[mcu]` punti alla mainboard;
- `[mcu extra_mcu]`, o il nome equivalente utilizzato in quella fase, punti alla toolboard originale.

Dopo l'installazione della Sovol Zero questa associazione non rappresenta più la toolhead corrente: la Zero utilizza la propria MCU CAN.

Non dedurre l'associazione dall'ordine `ttyACM0`, `ttyACM1` o simili.

Questi nomi possono cambiare fra un boot e l'altro.

## ST-Link

Il primo caricamento Katapult sulla Phoenix è stato eseguito tramite ST-Link e STM32CubeProgrammer.

Durante il collegamento ST-Link alla SV08:

**la stampante deve essere spenta.**

La procedura upstream Rappetor specifica che la MCU viene alimentata dallo ST-Link durante questa operazione.

Verificare inoltre che lo ST-Link abbia firmware aggiornato prima di iniziare.

## Prima del primo erase

Prima di cancellare la flash della prima MCU devono essere vere tutte queste condizioni:

- MCU identificata senza ambiguità;
- dump originale salvato;
- checksum del dump calcolato;
- copia del dump conservata fuori dalla stampante;
- configurazione Katapult verificata;
- firmware Klipper corretto già preparato;
- mainboard e toolboard chiaramente etichettate;
- stampante spenta;
- collegamenti ST-Link verificati.

Solo a questo punto si può procedere al primo erase/programming.

## Programmare Katapult sulla prima MCU

Lavorare su una MCU alla volta.

Sequenza raccomandata:

1. stampante completamente spenta;
2. collegare ST-Link alla MCU scelta;
3. leggere nuovamente l'identità della MCU;
4. verificare che il dump originale sia già stato salvato;
5. cancellare la flash solo dopo questa verifica;
6. programmare il binario Katapult corretto;
7. verificare il risultato della programmazione;
8. scollegare ST-Link;
9. ripristinare il normale collegamento USB della scheda.

Non passare alla seconda MCU finché la prima non è stata verificata.

## Verificare Katapult

Dopo aver programmato Katapult e riavviato la scheda, verificare che la MCU venga rilevata dal sistema Linux.

Controllare i dispositivi disponibili sotto:

`/dev/serial/by-id/`

e, quando utile:

`/dev/serial/by-path/`

La presenza di un nuovo dispositivo USB non è da sola sufficiente.

Verificare che il percorso appartenga davvero alla MCU appena programmata.

## Flash Klipper tramite Katapult

Una volta che Katapult è operativo, il firmware Klipper Mainline può essere caricato tramite il bootloader senza riaprire la stampante per usare ST-Link.

Usare il firmware compilato specificamente per quella MCU.

Mainboard e toolboard possono richiedere parametri diversi.

Non utilizzare il firmware mainboard sulla toolboard o viceversa.

Dopo il flashing:

1. attendere il riavvio della MCU;
2. verificare che venga nuovamente enumerata via USB;
3. annotare il nuovo identificativo stabile;
4. non avviare ancora movimenti o riscaldamenti.

## Ripetere sulla seconda MCU

Solo dopo aver completato e verificato la prima scheda:

1. spegnere nuovamente la stampante;
2. collegare ST-Link alla seconda MCU;
3. verificare il backup originale;
4. programmare Katapult;
5. verificare Katapult;
6. flashare il firmware Klipper destinato alla seconda MCU;
7. verificare il nuovo identificativo USB.

La procedura deve mantenere sempre chiara l'associazione:

- mainboard;
- toolboard.

## Verificare i nuovi seriali

Con entrambe le MCU su Klipper Mainline, elencare:

`/dev/serial/by-id/`

Annotare i due percorsi completi.

Non usare in configurazione permanente:

- `/dev/ttyACM0`;
- `/dev/ttyACM1`;
- altri nomi assegnati dinamicamente dal kernel.

Gli identificativi `by-id` sono preferibili perché stabili fra i reboot.

## Aggiornare printer.cfg

Aggiornare la configurazione solo dopo aver identificato entrambe le MCU.

Verificare attentamente che:

- `[mcu]` corrisponda alla mainboard;
- la sezione MCU toolboard corrisponda alla toolboard.

Un'associazione invertita può generare errori difficili da interpretare.

Dopo la modifica non eseguire subito `G28`.

Prima avviare Klipper e controllare gli errori di configurazione.

## Primo avvio Klipper con le nuove MCU

Al primo avvio verificare esclusivamente lo stato del sistema.

Controllare:

- entrambe le MCU connesse;
- nessun errore di protocollo;
- nessun mismatch firmware;
- temperature leggibili;
- nessun sensore in fault;
- configurazione caricata;
- nessun pin duplicato o non valido.

Non eseguire ancora:

- homing;
- QGL;
- mesh;
- movimenti manuali;
- riscaldamenti prolungati.

Prima deve essere completata la validazione hardware di base.

## Verifica temperature

Con macchina ferma e fredda, controllare che:

- temperatura bed sia plausibile;
- temperatura hotend sia plausibile;
- eventuali altri sensori riportino valori realistici.

Una lettura impossibile o fuori scala deve essere risolta prima di energizzare heater o motori.

## Verifica endstop e segnali

Controllare lo stato degli endstop e degli ingressi senza muovere gli assi.

Verificare che i segnali cambino in modo coerente quando vengono azionati manualmente, dove possibile e sicuro.

Non usare un homing come primo test di un endstop non ancora verificato.

## Verifica ventole e uscite

In questa fase, le uscite controllate dalla mainboard e dalla toolboard originale devono essere verificate separatamente.

Prestare particolare attenzione a:

- ventola hotend;
- part cooling;
- eventuali ventole elettronica;
- LED o uscite accessorie.

Non assumere che una configurazione stock funzioni senza modifiche su Mainline.

## Recovery — Katapult non compare

Se dopo il flashing Katapult la MCU non viene enumerata:

1. spegnere la stampante;
2. ricontrollare il collegamento USB;
3. ricontrollare i parametri Katapult compilati;
4. verificare offset applicazione;
5. riprovare tramite ST-Link;
6. se necessario riprogrammare Katapult direttamente.

Non iniziare a modificare `printer.cfg`: una MCU che non compare a livello USB non è un problema di configurazione Klipper.

## Recovery — firmware Klipper non parte

Se Katapult è raggiungibile ma Klipper non parte dopo il flashing, verificare:

- MCU corretta;
- bootloader offset;
- clock;
- interfaccia USB;
- firmware destinato alla scheda corretta;
- build realmente prodotta dal checkout Klipper desiderato.

Se il problema persiste, Katapult può essere usato per riflashare una build corretta.

## Recovery — MCU non recuperabile via USB

Se né Klipper né Katapult risultano raggiungibili:

- spegnere la stampante;
- tornare a ST-Link;
- verificare che la MCU sia ancora leggibile;
- riprogrammare Katapult;
- oppure ripristinare il firmware originale personale.

Questo è il motivo per cui il backup originale viene eseguito prima di qualsiasi erase.

## Recovery — rollback completo MCU

Per tornare al firmware stock:

1. usare il dump originale della propria mainboard;
2. usare il dump originale della propria toolboard;
3. programmare ogni MCU separatamente tramite ST-Link;
4. verificare i checksum dei file prima della programmazione;
5. ripristinare anche il sistema Linux/configurazione stock quando necessario.

Il firmware MCU stock da solo non ricrea necessariamente l'intero ambiente Sovol originale.

## Phoenix verified — risultato della migrazione

La migrazione Phoenix ha portato con successo:

- mainboard originale Sovol su Katapult + Klipper Mainline;
- toolboard originale Sovol su Katapult + Klipper Mainline;
- entrambe le MCU riconosciute stabilmente;
- configurazione aggiornata senza scambiare mainboard e toolboard;
- successivo avvio della stampante su Klipper Mainline.

Questa fase è stata completata prima dell'installazione definitiva della Sovol Zero, dell'Eddy integrato e delle successive calibrazioni.

## Criterio di uscita

Prima di passare alla configurazione generale devono essere vere tutte queste condizioni:

- entrambe le MCU originali eseguono Klipper Mainline;
- seriali stabili annotati;
- `[mcu]` associata alla mainboard corretta;
- toolboard originale associata alla sezione corretta;
- temperature plausibili;
- nessun errore MCU;
- backup originali conservati;
- recovery ST-Link ancora disponibile.

---

## Navigazione

- ← **Pagina precedente:** [Installazione CB1 e Klipper Mainline](install-cb1-mainline.md)
- → **Pagina successiva:** [Configurazione base Mainline](base-configuration.md)
