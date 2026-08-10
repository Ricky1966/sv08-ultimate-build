# SV08 Ultimate Build



> [!WARNING]
> ## Disclaimer
>
> Questo è un progetto **community e non ufficiale**. Non è affiliato, sponsorizzato o approvato da Sovol o dagli altri progetti e produttori citati.
>
> Le procedure, configurazioni e modifiche documentate in questo repository derivano da una migrazione realmente eseguita e testata su una specifica Sovol SV08, ma **non costituiscono una garanzia di compatibilità o corretto funzionamento su ogni macchina**.
>
> Klipper, Katapult, Eddy, DKEU e gli altri componenti software evolvono nel tempo: quanto documentato qui rappresenta uno stato verificato del progetto, non necessariamente lo stato dell'arte corrente.
>
> Flash del firmware, modifiche alla configurazione, cablaggi, calibrazioni e movimenti della macchina possono causare perdita della configurazione originale, malfunzionamenti o danni se eseguiti in modo errato.
>
> **Chi utilizza questo repository lo fa sotto la propria responsabilità.** Prima di qualsiasi modifica, conserva backup verificati e assicurati di comprendere le operazioni che stai eseguendo.
>
**Lingue:** **Italiano** | [English](README.en.md)

Documentazione community per migrare una **Sovol SV08** dallo stack software di fabbrica a **Klipper Mainline corrente**, includendo:

- configurazione CB1/Linux;
- backup, flashing e recovery delle MCU;
- configurazione della stampante su Mainline;
- supporto Eddy nativo di Klipper;
- Demon Klipper Essentials Unified / DKEU3;
- calibrazione;
- troubleshooting;
- rollback.

Le guide derivano da una migrazione reale end-to-end di una Sovol SV08 soprannominata **Phoenix**, ma il repository è strutturato in modo da tenere separati i valori e la cronologia specifici della Phoenix dalla baseline community riutilizzabile.

## Da dove iniziare

Se stai migrando una Sovol SV08, segui le guide in questo ordine:

1. [Getting started](docs/getting-started.md)
2. [Backup e rollback](docs/backup-and-rollback.md)
3. [Installazione CB1 e Klipper Mainline](docs/install-cb1-mainline.md)
4. [Flashing e recovery delle MCU](docs/flash-mcus.md)
5. [Configurazione base Mainline](docs/base-configuration.md)
6. [Eddy nativo](docs/native-eddy.md)
7. [Integrazione Demon / DKEU3](docs/demon-integration.md)
8. [Validazione e calibrazione](docs/validation-and-calibration.md)
9. [Troubleshooting](docs/troubleshooting.md)

È disponibile anche un indice di compatibilità per i vecchi link:

[Migrazione Klipper Mainline](docs/mainline-migration.md)

## Leggi prima di flashare qualsiasi cosa

**Non** iniziare cancellando o flashando le MCU.

Prima di modificare il firmware, completa almeno:

- inventario del sistema stock;
- backup di `printer.cfg` e di tutti i file inclusi;
- backup di Moonraker e delle macro personalizzate;
- piano di rollback del sistema/eMMC;
- dump personali del firmware MCU originale quando possibile;
- verifica di aver compreso la procedura di recovery.

Vedi:

[Backup e rollback](docs/backup-and-rollback.md)

## Cosa copre questo repository

Il percorso community documenta attualmente:

- migrazione dall'ambiente Klipper factory Sovol;
- ambiente Linux basato su CB1 usato dal controller Sovol stock;
- Klipper Mainline;
- migrazione firmware mainboard e toolboard;
- percorso di recovery/update basato su Katapult;
- identificazione stabile delle MCU tramite `/dev/serial/by-id/`;
- supporto nativo `[probe_eddy_current ...]`;
- homing Z con Eddy nativo;
- QGL e bed mesh;
- integrazione DKEU3;
- integrazione Machine G-code di OrcaSlicer;
- validazione del first layer;
- calibrazione filamento;
- troubleshooting basato su casi di guasto reali.

## Baseline di migrazione verificata

La migrazione Phoenix ha raggiunto un sistema Mainline pienamente operativo con:

- Klipper Mainline `v0.13.0-718-gd8659974-dirty`;
- commit Klipper `d865997403cad36d105026f73a4b76dcacec4c76`;
- Moonraker;
- Mainsail;
- KlipperScreen;
- mainboard e toolboard con firmware compatibile Mainline;
- Eddy nativo tramite `[probe_eddy_current eddy]`;
- `G28` completo;
- probing nativo;
- QGL;
- bed mesh;
- DKEU;
- OrcaSlicer;
- stampa reale completata con successo dopo la migrazione.

Questi numeri di versione descrivono la baseline Phoenix realmente testata. Non sono un requisito per restare bloccati su quegli esatti commit.

Per una nuova migrazione, confronta sempre la documentazione upstream corrente prima di installare o patchare software.

## Eddy nativo

La migrazione Phoenix inizialmente utilizzava il percorso esterno `probe_eddy_ng`.

Quel percorso è stato successivamente abbandonato perché l'ambiente Mainline corrente include già il supporto Eddy nativo e il plugin legacy creava problemi nel percorso di homing Z.

La documentazione community considera quindi:

`[probe_eddy_current ...]`

come baseline.

Vedi [Eddy nativo](docs/native-eddy.md).

Valori specifici Phoenix come offset, `reg_drive_current`, `max_sensor_hz` e curve di calibrazione sono esempi, non preset universali.

## Demon / DKEU3

Demon Klipper Essentials Unified viene integrato solo dopo che la macchina Mainline di base è già funzionante.

Non usare Demon come sostituto della validazione di comunicazione MCU, direzioni stepper, heater, endstop, homing, Eddy o QGL.

La guida documenta anche una lezione importante emersa dalla migrazione reale: un vecchio override locale di una macro può sostituire silenziosamente una macro DKEU più recente e creare un problema di stampa anche quando probing, QGL e mesh sembrano corretti.

Vedi [Integrazione Demon / DKEU3](docs/demon-integration.md).

## Filosofia di troubleshooting

La migrazione segue intenzionalmente un approccio basato sulle evidenze.

Se homing funziona, il probing è ripetibile, QGL converge e la mesh è coerente, ma la stampa è ancora errata, controlla override delle macro, riallineamento Z, profilo slicer, G-code generato e residui della migrazione software prima di modificare gantry, motori Z, viti del bed, axis twist o geometria meccanica.

Diversi problemi Phoenix che inizialmente sembravano meccanici si sono rivelati problemi software/configurazione.

Vedi [Troubleshooting](docs/troubleshooting.md).

## Caso di studio Phoenix

La cronologia completa e cronologica della migrazione è conservata separatamente sotto:

`docs/migration-history/phoenix/`

Contiene:

- stati intermedi;
- esperimenti falliti;
- workaround temporanei;
- misure;
- indagini sulle root cause;
- correzioni finali.

Questa cronologia è utile per debugging e archeologia tecnica. **Non** è la procedura di installazione step-by-step raccomandata.

## Esempi Phoenix

Gli esempi specifici Phoenix sono conservati sotto:

`examples/phoenix/`

Possono includere:

- note hardware;
- profili OrcaSlicer;
- roadmap del progetto;
- valori specifici della macchina.

Non assumere che questi file possano essere copiati invariati su un'altra SV08.

## Calibrazione

Il repository separa tre classi di calibrazione:

### Macchina

Esempi:

- PID;
- configurazione stepper;
- endstop;
- heater.

### Bed / Z

Esempi:

- calibrazione Eddy;
- QGL;
- riferimento Z;
- mesh;
- first layer.

### Filamento

Esempi:

- temperatura;
- flow ratio;
- pressure advance;
- retraction;
- max volumetric speed.

Vedi [Validazione e calibrazione](docs/validation-and-calibration.md).

## Modifiche hardware

La baseline community Mainline è intenzionalmente separata dal successivo sviluppo hardware Phoenix.

Modifiche future o specifiche della macchina possono includere:

- piatto in grafite;
- redesign toolhead;
- bus CAN;
- EBB36 / EBB42;
- modifiche enclosure;
- SSR;
- isolamento;
- redesign umbilical.

Queste modifiche non devono ridefinire la procedura di migrazione base.

## Regole del repository

Non committare:

- password;
- credenziali Wi-Fi;
- token;
- chiavi private;
- host key SSH;
- dump MCU personali destinati solo al rollback;
- immagini eMMC private complete;
- segreti specifici della macchina.

Prima di rendere pubblico un branch o una release, esegui un audit privacy di:

- file correnti;
- storia Git;
- branch;
- tag;
- asset delle release.

## Firmware e artefatti binari

I dump originali personali del firmware devono restare nei backup locali, non nella storia pubblica del repository.

Il firmware preparato per la migrazione deve essere riproducibile tramite parametri di build documentati quando possibile.

Se in futuro verranno distribuite immagini binarie grandi o immagini di sistema sanitizzate, le GitHub Releases sono preferibili alla normale storia Git.

## Progetti upstream

Questo progetto si basa sul lavoro delle community Klipper e SV08, inclusi:

- Klipper;
- Moonraker;
- Mainsail;
- KIAUH;
- Katapult;
- il lavoro Sovol SV08 Mainline di Rappetor;
- Demon Klipper Essentials Unified;
- OrcaSlicer.

Consulta sempre il progetto upstream pertinente prima di applicare vecchi workaround provenienti dalla cronologia di migrazione.

## Politica linguistica

L'italiano è la lingua principale del repository. L'inglese viene mantenuto come secondo percorso community completo.

La cronologia di sviluppo Phoenix e le note archivistiche specifiche della macchina possono restare in italiano quando non fanno parte della procedura community riutilizzabile.

## Licenza

Il materiale originale di questo repository è distribuito secondo i termini della **GNU General Public License v3.0 or later**, nella misura in cui il relativo copyright sia detenuto dagli autori del progetto.

Vedi:

- [`LICENSE`](LICENSE) — testo completo della licenza;
- [`NOTICE.md`](NOTICE.md) — attribuzioni, materiale di terze parti, preset derivati e marchi.

I progetti upstream citati o utilizzati mantengono le rispettive licenze.
