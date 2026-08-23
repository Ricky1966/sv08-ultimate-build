# Backup and rollback — Sovol SV08 before Mainline migration

Ultima verifica delle fonti: **2026-08-10**.

## Obiettivo

Prima di modificare il sistema operativo, la eMMC o il firmware delle MCU, creare un percorso di ritorno verificabile.

Per una migrazione della SV08 stock verso Klipper Mainline considerare almeno quattro livelli distinti di backup:

1. configurazione Klipper e servizi;
2. sistema Linux / eMMC;
3. firmware originale mainboard MCU;
4. firmware originale toolboard MCU stock.

Questi quattro livelli descrivono la macchina stock prima della migrazione.

Se sulla propria SV08 è già installata una toolhead diversa, come la **Sovol Zero Extruder Kit**, salvare separatamente anche firmware, configurazione e identificativi della toolhead corrente.

Questi backup non sono equivalenti fra loro.

Avere soltanto una copia di `printer.cfg` non significa avere un rollback completo.

## Livello 1 — Configurazione

Salvare almeno:

- `printer.cfg`;
- tutti i file `.cfg` inclusi;
- macro personalizzate;
- `saved_variables.cfg`;
- configurazioni Eddy;
- Phoenix Macros e altre macro/configurazioni personalizzate;
- `moonraker.conf`;
- `mainsail.cfg`;
- `KlipperScreen.conf`, se usato;
- `crowsnest.conf`;
- eventuali script shell personalizzati;
- configurazioni di rete personalizzate solo se realmente necessarie;
- valori PID;
- input shaper;
- pressure advance;
- Z offset e calibrazioni utili come riferimento;
- mesh e risultati QGL utili come confronto storico.

Non copiare nel repository pubblico:

- password;
- token;
- chiavi SSH;
- credenziali Wi-Fi;
- database personali;
- file con segreti;
- log non verificati.

## Livello 2 — Sistema Linux / eMMC

Il metodo più sicuro è mantenere disponibile un supporto stock ancora avviabile.

Le possibilità includono:

- conservare la eMMC originale senza modificarla;
- creare un'immagine completa della eMMC;
- installare Mainline inizialmente su microSD;
- utilizzare una seconda eMMC dedicata al nuovo sistema.

La scelta migliore è quella che permette di tornare rapidamente a un sistema stock funzionante senza dover ricostruire la macchina da zero.

## Regola di verifica

Un backup non è considerato valido finché non è stato:

1. copiato fuori dalla stampante;
2. verificato come leggibile;
3. identificato con data e origine;
4. accompagnato da checksum quando opportuno.

Per file binari e immagini usare SHA256.

## Livello 3 — Firmware originale mainboard MCU

Prima di installare Katapult o Klipper Mainline sulla mainboard, creare un dump del firmware originale della propria macchina.

Non distribuire come backup universale il dump di un'altra SV08.

Il firmware originale salvato da una macchina diversa può non essere adatto alla propria revisione hardware o al proprio stato.

Il backup deve essere identificato almeno con:

- macchina;
- data;
- MCU;
- dimensione del file;
- checksum SHA256.

## Livello 4 — Firmware originale toolboard MCU

Questa sezione si riferisce alla **toolboard originale Sovol presente durante la migrazione iniziale della macchina stock**.

Applicare lo stesso principio alla toolboard originale prima di sostituirla o modificarne il firmware.

Mainboard e toolboard originale devono essere trattate come due MCU distinte.

Non assumere che un solo dump copra entrambe.

Anche il backup toolboard deve riportare:

- macchina;
- data;
- MCU;
- dimensione del file;
- checksum SHA256.

## ST-Link

Per il percorso documentato in questo repository è fortemente raccomandato avere uno ST-Link disponibile prima di iniziare il flashing.

Lo ST-Link serve come percorso di recupero quando:

- la MCU non è più raggiungibile via USB;
- Katapult non parte;
- il firmware è stato scritto con parametri errati;
- è necessario ripristinare il firmware originale;
- è necessario riflashare direttamente la MCU.

Non iniziare una procedura di flashing senza sapere dove collegare:

- SWDIO;
- SWCLK;
- GND;
- alimentazione appropriata.

Verificare sempre il pinout della propria revisione hardware.

## Checksum

Dopo ogni dump binario calcolare SHA256.

Esempio:

`sha256sum nome-file.bin`

Conservare il checksum insieme al file originale.

Per immagini complete della eMMC usare lo stesso principio:

`sha256sum immagine.img`

Il checksum serve a verificare che la copia usata per il rollback sia identica a quella salvata.

## Nomi consigliati

Usare nomi che rendano evidente origine e data.

Esempi:

- `SV08-mainboard-original-YYYYMMDD.bin`
- `SV08-toolboard-original-YYYYMMDD.bin`
- `sv08-stock-emmc-YYYYMMDD.img`

Evitare nomi generici come:

- `backup.bin`
- `firmware.bin`
- `old.img`

## Rollback minimo

Prima della migrazione sapere già come eseguire almeno uno dei seguenti rollback:

### Rollback del sistema Linux

Ripristinare:

- eMMC stock originale;
- oppure immagine completa verificata;
- oppure supporto stock conservato separatamente.

### Rollback delle MCU

Ripristinare tramite ST-Link:

- firmware originale mainboard;
- firmware originale toolboard.

### Rollback della configurazione

Ripristinare i file originali sotto:

`printer_data/config/`

Il rollback completo può richiedere tutte e tre le operazioni.

## Cosa non fare

Non:

- sovrascrivere l'unica copia della eMMC stock senza backup;
- salvare i dump originali soltanto sulla stampante;
- affidarsi a un singolo file `printer.cfg`;
- usare firmware originali scaricati da terzi come unico piano di recupero;
- iniziare il flashing senza aver verificato i file appena creati;
- pubblicare dump personali o immagini eMMC senza una revisione privacy e sicurezza.

## Backup verificato sulla macchina Phoenix

Durante la migrazione reale usata per validare questa guida sono stati salvati e verificati separatamente:

- firmware originale mainboard;
- firmware originale toolboard;
- backup del sistema;
- configurazioni Klipper;
- checksum SHA256.

I file originali specifici della macchina Phoenix sono conservati localmente e non fanno parte della baseline pubblica del repository.

La configurazione Phoenix attuale utilizza una **Sovol Zero Extruder Kit collegata via CAN**.

Il backup della Zero è quindi un elemento aggiuntivo rispetto ai quattro livelli riferiti alla SV08 stock: firmware, configurazione e identificativi CAN devono essere salvati separatamente quando si interviene sulla toolhead corrente.

La presenza dei backup storici della macchina Phoenix serve solo a documentare che durante la migrazione era disponibile un rollback reale.

## Criterio di uscita

Non procedere alla fase di installazione Mainline finché non sono vere tutte queste condizioni:

- configurazioni salvate;
- sistema stock recuperabile;
- mainboard MCU recuperabile;
- toolboard originale MCU recuperabile;
- checksum disponibili;
- backup conservati fuori dalla stampante;
- ST-Link disponibile e utilizzabile.

---

## Navigazione

- ← **Pagina precedente:** [Getting started](getting-started.md)
- → **Pagina successiva:** [Installazione CB1 e Klipper Mainline](install-cb1-mainline.md)
