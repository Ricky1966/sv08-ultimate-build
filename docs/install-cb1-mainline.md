# Install CB1 Linux and Klipper Mainline on Sovol SV08

Ultima verifica delle fonti: **2026-08-10**.

## Scopo

Questa sezione documenta il passaggio dal sistema Linux stock della Sovol SV08 a un ambiente pulito sul quale installare Klipper Mainline.

La procedura segue come riferimento principale:

`Rappetor/Sovol-SV08-Mainline`

La macchina Phoenix è stata migrata usando questo percorso come base, con verifiche aggiuntive documentate in questo repository.

## Prerequisito

Prima di continuare deve essere stato completato:

`docs/backup-and-rollback.md`

Non procedere se il sistema stock non è recuperabile.

## Immagine Linux scelta

Per il percorso attualmente documentato usare:

**BIGTREETECH CB1 V2.3.4 minimal**

La guida Rappetor corrente continua a sconsigliare V3.0.0 o successive per la SV08.

Questo non significa che V2.3.4 sia la release CB1 più recente.

Significa che è la baseline attualmente documentata e verificata per questo percorso SV08.

Preferire l'immagine **minimal**, evitando immagini con versioni preinstallate e non controllate di Klipper, Moonraker o Mainsail.

## Dove installare il nuovo sistema

Sono possibili diversi approcci:

- nuova eMMC dedicata;
- eMMC originale dopo averne creato un backup completo;
- microSD per test o uso permanente;
- installazione tramite USB/FEL secondo la procedura upstream.

Per un primo tentativo, mantenere fisicamente intatto un supporto stock avviabile è la soluzione con il rollback più semplice.

## Preparazione del supporto

Dopo aver scritto l'immagine CB1 V2.3.4 minimal sul supporto, la partizione `BOOT` deve essere accessibile dal computer.

Prima di modificare qualsiasi file:

- creare una copia di `BoardEnv.txt`;
- non cancellare la parte del file che la guida upstream indica di conservare;
- preparare il DTB specifico SV08.

## DTB SV08

La guida upstream corrente utilizza:

`sun50i-h616-sovol-emmc.dtb`

Il file deve essere copiato nella directory:

`/dtb/allwinner/`

della partizione `BOOT`.

La configurazione usa poi:

`fdtfile=sun50i-h616-sovol-emmc`

Questo DTB è il componente che permette al sistema CB1 di utilizzare correttamente l'hardware della mainboard SV08.

## BoardEnv.txt

La parte iniziale della configurazione verificata contiene:

- `bootlogo=false`
- `overlay_prefix=sun50i-h616`
- `fdtfile=sun50i-h616-sovol-emmc`
- `console=display`

e gli overlay:

- `uart3`
- `ws2812`
- `spidev1_1`

Non sostituire alla cieca l'intero `BoardEnv.txt`.

Seguire la struttura del file presente nell'immagine utilizzata e conservare le sezioni che la guida upstream indica di non modificare.

## system.cfg

Se la stampante utilizzerà Wi-Fi, configurare le credenziali in:

`system.cfg`

È possibile impostare anche un hostname personalizzato.

Non pubblicare mai nel repository:

- SSID personali;
- password Wi-Fi;
- hostname contenenti dati personali;
- indirizzi IP privati usati come identificativi permanenti.

Se la password Wi-Fi contiene caratteri speciali, verificare le regole di escaping previste dallo script CB1.

La guida upstream segnala in particolare il carattere `$`, che può richiedere escaping.

## Primo boot

Dopo aver completato `BoardEnv.txt`, `system.cfg` e il DTB:

1. espellere correttamente il supporto dal computer;
2. installarlo nella SV08;
3. avviare la stampante;
4. attendere il boot Linux;
5. identificare l'indirizzo IP assegnato;
6. verificare l'accesso SSH.

Con l'immagine CB1 V2.3.4 standard, la guida upstream utilizza inizialmente:

- utente: `biqu`
- password: `biqu`

Cambiare le credenziali predefinite appena il sistema è sotto controllo.

## Verifica prima di installare Klipper

Prima di proseguire verificare almeno:

- SSH funzionante;
- rete funzionante;
- DNS funzionante;
- data e ora plausibili;
- filesystem scrivibile;
- supporto di boot corretto;
- nessun riavvio spontaneo.

Se si usa una microSD insieme a una eMMC presente, verificare con `lsblk` quale dispositivo fornisce realmente `/boot` e `/`.

Questo evita di configurare accidentalmente un sistema diverso da quello che si crede di aver avviato.

## Passo successivo

Con Linux avviato e SSH funzionante si può procedere all'installazione di:

- KIAUH;
- Klipper;
- Moonraker;
- Mainsail;
- Crowsnest;
- opzionalmente KlipperScreen.

## Aggiornare il sistema base

Dopo il primo accesso SSH, aggiornare gli indici dei pacchetti e installare gli aggiornamenti di sistema disponibili.

Eseguire queste operazioni prima di installare Klipper e gli altri servizi.

Se `apt` restituisce errori:

- non applicare automaticamente workaround trovati in vecchie guide;
- verificare prima rete, DNS, data/ora e repository configurati;
- distinguere un problema dell'immagine Linux da un problema Klipper.

La guida Rappetor contiene ancora note storiche relative a incompatibilità Python/Klipper incontrate in passato.

Queste non fanno parte del percorso normale documentato qui e verranno trattate solo nel troubleshooting se ancora riproducibili.

## Installare KIAUH

KIAUH viene utilizzato come gestore di installazione per lo stack Klipper.

Repository upstream:

`dw-0/kiauh`

Installare KIAUH seguendo le istruzioni correnti del progetto ufficiale.

Non utilizzare copie vecchie dello script salvate localmente se è disponibile una versione upstream più recente compatibile con il sistema.

## Componenti da installare

Per la baseline Mainline della SV08 installare tramite KIAUH:

1. Klipper;
2. Moonraker;
3. Mainsail;
4. Mainsail-Config;
5. Crowsnest.

KlipperScreen è opzionale ma è stato installato e verificato sulla macchina Phoenix.

Fluidd non è necessario se si utilizza Mainsail.

## Klipper Mainline

L'obiettivo di questa procedura è installare il repository ufficiale Klipper Mainline, non il fork o la build Sovol stock.

Dopo l'installazione verificare:

- directory Klipper presente;
- ambiente Python creato correttamente;
- servizio Klipper installato;
- Moonraker avviabile;
- Mainsail raggiungibile.

In questa fase è normale che Klipper non possa ancora comunicare correttamente con le MCU se queste eseguono ancora il firmware stock.

Non tentare di risolvere un errore MCU modificando casualmente la configurazione prima del flashing previsto nelle fasi successive.

## Moonraker

Installare Moonraker tramite KIAUH e verificare che il servizio parta senza errori critici.

Controllare che:

- Mainsail riesca a comunicare con Moonraker;
- `printer_data` venga creato nella posizione prevista;
- i file di configurazione siano accessibili dalla web interface.

## Mainsail

Installare Mainsail e Mainsail-Config.

Verificare che l'interfaccia web sia raggiungibile all'indirizzo della stampante.

La presenza di Mainsail funzionante non significa ancora che la stampante sia pronta al movimento.

Prima del flash MCU e della validazione della configurazione, evitare comandi di homing, movimento o riscaldamento non necessari.

## Crowsnest

Installare Crowsnest se si intende utilizzare la webcam.

La webcam non è necessaria per completare la migrazione Mainline.

Se Crowsnest genera errori, separarli dalla diagnosi Klipper: un problema webcam non deve bloccare il lavoro sulle MCU o sulla configurazione della stampante.

## KlipperScreen

KlipperScreen è opzionale.

Sulla macchina Phoenix è stato installato dopo la migrazione base ed è risultato compatibile con il nuovo ambiente.

Non è necessario averlo funzionante prima di:

- flashare le MCU;
- verificare Klipper;
- effettuare il primo homing controllato.

## Verificare le versioni

Dopo l'installazione annotare:

- versione Klipper;
- commit Klipper;
- versione Moonraker;
- versione KIAUH;
- eventuale stato dirty del repository Klipper.

Queste informazioni sono fondamentali per il troubleshooting successivo.

Una migrazione documentata senza versione o commit rende molto più difficile confrontare problemi futuri.

## Non importare ancora tutta la configurazione stock

A questo punto non copiare indiscriminatamente tutti i file della vecchia installazione Sovol dentro `printer_data/config`.

Recuperare invece in modo controllato:

- pin hardware;
- parametri motori;
- geometria;
- seriali MCU;
- heater;
- sensori;
- ventole;
- macro realmente necessarie.

Le macro legacy e le integrazioni Eddy devono essere valutate separatamente.

## Configurazione iniziale

Prima del flashing delle MCU preparare una configurazione minima che permetta successivamente di verificare:

- connessione mainboard;
- connessione toolboard;
- temperature;
- endstop;
- stepper;
- ventole;
- probe.

Non considerare ancora questa configurazione definitiva.

La configurazione verrà raffinata nelle sezioni dedicate a:

- MCU;
- Eddy;
- Demon;
- calibrazione.

## Criterio di uscita

Prima di passare al firmware MCU devono essere vere tutte queste condizioni:

- Linux stabile;
- SSH stabile;
- rete stabile;
- KIAUH funzionante;
- Klipper Mainline installato;
- Moonraker installato;
- Mainsail raggiungibile;
- configurazioni stock disponibili come riferimento;
- backup e rollback già verificati.

## Passo successivo

La fase successiva è il backup e flashing controllato delle MCU:

`docs/flash-mcus.md`

