# Troubleshooting — Sovol SV08 Mainline migration

Ultima revisione: **2026-08-23**.

## Scopo

Questa guida raccoglie i problemi realmente incontrati durante la migrazione Phoenix e li trasforma in una procedura diagnostica riutilizzabile.

La struttura è:

- sintomo;
- verifiche;
- causa probabile;
- cosa non toccare;
- soluzione.

Non tutti i problemi storici Phoenix sono ancora presenti nelle versioni correnti.

Le workaround obsolete vengono esplicitamente marcate come tali.

## Regola fondamentale

Non cambiare più sottosistemi contemporaneamente.

Prima di modificare:

- meccanica;
- Z;
- QGL;
- mesh;
- macro;
- slicer;

identificare il livello in cui nasce realmente il problema.

## Ordine diagnostico

Quando la macchina non funziona correttamente, verificare nell'ordine:

1. Linux/CB1;
2. USB/CAN;
3. MCU;
4. configurazione Klipper;
5. endstop;
6. homing;
7. Eddy;
8. QGL;
9. mesh;
10. Phoenix Macros;
11. slicer;
12. first layer.

Questo evita di correggere un problema a valle mentre la causa è a monte.

# Linux / CB1

## La macchina non è raggiungibile in rete

### Verificare

- indirizzo IP corrente;
- stato NetworkManager;
- interfaccia Ethernet;
- interfaccia Wi-Fi;
- route di default;
- DNS;
- NTP.

### Caso Phoenix

La Phoenix utilizzava contemporaneamente:

- Ethernet diretta locale;
- Wi-Fi per Internet.

Il profilo Ethernet installava però anche una default route con priorità migliore del Wi-Fi.

Il traffico Internet e DNS veniva quindi inviato verso una rete Ethernet locale senza accesso Internet.

### Soluzione

Configurare Ethernet come collegamento locale senza default route.

Lasciare al Wi-Fi la default route verso Internet.

### Cosa non toccare

Non modificare Klipper, Moonraker o MCU quando:

- la macchina risponde localmente;
- il problema riguarda soltanto DNS/Internet/NTP.

## Data e ora non corrette

### Verificare

- raggiungibilità Internet;
- DNS;
- route;
- servizio NTP.

### Caso Phoenix

Il servizio NTP era attivo, ma i peer mostravano `reach 0`.

La causa era la configurazione di rete, non NTP stesso.

### Soluzione

Ripristinare prima route e DNS.

Poi verificare nuovamente NTP.

# MCU / USB

## Non usare `ttyACM0` / `ttyACM1`

Gli identificativi:

- `/dev/ttyACM0`
- `/dev/ttyACM1`

non sono stabili.

Possono invertirsi fra un boot e l'altro.

### Soluzione

Dopo il flash Mainline usare:

`/dev/serial/by-id/`

Quando le MCU stock espongono lo stesso seriale generico, usare temporaneamente:

`/dev/serial/by-path/`

per distinguerle fisicamente prima del flash.

## Mainboard e toolboard USB originale risultano scambiate

### Sintomi possibili

- pin inesistenti;
- heater errato;
- endstop errato;
- toolboard non raggiungibile;
- errori apparentemente incoerenti.

### Verificare

Nella configurazione storica con toolboard USB originale:

- seriale configurato in `[mcu]`;
- seriale configurato in `[mcu extra_mcu]`;
- corrispondenza fisica con la scheda.

Con la Sovol Zero attuale verificare invece UUID CAN e stato della MCU toolhead CAN.

### Soluzione

Identificare le due MCU una alla volta.

Non basarsi sull'ordine `ttyACM*`.

## MCU non parte dopo il flash

### Verificare

- target MCU corretto;
- STM32F103;
- crystal;
- USB pins;
- bootloader offset;
- Katapult;
- file corretto per mainboard/toolboard.

### Phoenix verified

Il percorso Phoenix ha utilizzato Katapult con:

- STM32F103;
- 8 MHz crystal;
- USB PA11/PA12;
- bootloader offset 8 KiB.

### Cosa non fare

Non cancellare nuovamente la MCU finché non è stata verificata la configurazione del firmware.

Conservare sempre il dump originale personale.

# Homing

## `Z axis must be homed before probing`

### Contesto storico Phoenix

Questo errore è comparso utilizzando il vecchio plugin:

`probe_eddy_ng`

su Klipper Mainline.

Il percorso entrava in una dipendenza circolare:

- homing Z richiedeva il probe;
- il probe richiedeva Z già homed.

### Soluzione Phoenix

Il vecchio plugin è stato abbandonato.

È stato utilizzato il supporto Eddy nativo:

`[probe_eddy_current eddy]`

### Cosa non fare

Non risolvere permanentemente con:

`set_position_z: 0`

Questo dichiara falsamente Z come homed.

Non continuare a patchare `probe_eddy_ng.py` se il supporto Mainline nativo è disponibile.

## Movimento Z prima dell'homing

Un movimento Z assoluto o relativo prima che Z sia noto può essere pericoloso.

### Soluzione

Nel percorso homing, eseguire eventuali sollevamenti Z solo quando:

`z` è già presente in `printer.toolhead.homed_axes`.

# Eddy

## `Trigger analog error: RAW_RANGE`

### Verificare

- hardware Eddy realmente installato;
- calibrazione Eddy;
- frequenza osservata;
- `max_sensor_hz`;
- I2C;
- alimentazione;
- cablaggio;
- eventuali modifiche locali del driver LDC1612.

### Phoenix corrente — Sovol Zero

Con la Eddy integrata nella Sovol Zero, la configurazione Phoenix validata utilizza:

`max_sensor_hz: 7000000`

Con il clock Mainline utilizzato sulla Phoenix questo porta il LDC1612 al divider 3.

Durante la migrazione Zero, con divider 2 la calibrazione poteva completare ma `G28 Z` produceva:

`Trigger analog error: RAW_RANGE`

Con divider 3 il problema è scomparso.

La configurazione completa è documentata nella pagina:

[Sovol Zero toolhead, CAN ed Eddy integrato](zero-toolhead-eddy-2026-08-17.md)

### Storico — precedente Eddy NG

Prima della Sovol Zero, la precedente configurazione Eddy NG Phoenix aveva mostrato frequenze fino a circa `8.523 MHz` ed era stato utilizzato:

`max_sensor_hz: 9000000`

Quel valore appartiene alla **vecchia Eddy NG** e non deve essere trasferito alla Zero.

### Cosa non fare

Non copiare `7000000`, `9000000` o altri valori da una configurazione diversa senza verificare hardware, driver e frequenza realmente osservata.

## Eddy instabile o rumoroso

### Verificare

- temperatura stabile;
- cablaggio;
- alimentazione;
- drive current;
- distanza probe-bed;
- interferenze;
- frequenza.

### Metodo

Confrontare più letture nelle stesse condizioni.

Non diagnosticare drift termico partendo da una singola misura.

## `Tap not configured`

### Contesto

Durante la migrazione Phoenix la versione DKEU installata entrava nel ramo TAP anche senza una vera calibrazione TAP.

### Sintomo

`Tap not configured`

### Causa storica

La logica verificava la presenza di `tap_threshold` in modo incompatibile con il comportamento Mainline di quella fase.

### Soluzione storica

Fu applicata temporaneamente una condizione più restrittiva.

### Stato corrente

Quella patch appartiene alla fase storica DKEU e non deve essere applicata automaticamente ad altre configurazioni.

Usare:

- Klipper Mainline corrente;
- versione DKEU utilizzata, se si sta analizzando una configurazione storica o indipendente da Phoenix;
- calibrazione TAP corrente se TAP è realmente desiderato.

### Cosa non fare

Non importare vecchi valori come:

`tap_threshold: 300`

da configurazioni Eddy NG legacy.

# QGL

## QGL esegue correzioni eccessive

### Verificare

- punti QGL;
- gantry corners;
- `max_adjust`;
- coordinate probe;
- eventuale wrapper Demon presente nella configurazione storica analizzata;
- configurazione realmente caricata.

### Caso Phoenix

Durante una fase legacy il ramo Demon poteva forzare un comportamento differente dal QGL nativo definito nel `printer.cfg`.

La soluzione fu riportare il percorso Eddy al QGL nativo configurato.

### Configurazione Phoenix consolidata

- `horizontal_move_z: 3`
- `retry_tolerance: 0.05`
- `retries: 5`
- `max_adjust: 4`

### Cosa non fare

Non aumentare enormemente `max_adjust` per far terminare un QGL problematico.

Un limite ampio può trasformare un errore di configurazione in un movimento meccanico pericoloso.

## QGL converge ma il first layer è sbagliato

QGL misura il rapporto del gantry rispetto al piano.

Non garantisce da solo:

- Z corretto;
- mesh corretta;
- compensazioni corrette;
- macro corrette.

### Verificare dopo QGL

- probe;
- mesh;
- Z reference;
- eventuale percorso Demon della configurazione storica;
- slicer.

# Mesh

## Mesh stranamente piatta ma first layer molto inclinato

### Caso Phoenix

È stata individuata una vecchia:

`axis_twist_compensation`

che modificava i risultati probe.

La compensazione quasi annullava una differenza reale di circa:

`0.33 mm`

fra i lati del piatto.

La mesh appariva artificialmente piatta mentre il first layer risultava errato.

### Soluzione

Neutralizzare la compensazione legacy non più valida e rifare la mesh.

### Cosa non fare

Non interpretare una mesh quasi piatta come automaticamente corretta.

Una mesh deve rappresentare la superficie reale.

## Mesh cambia drasticamente fra test simili

### Verificare

- temperatura bed;
- soak;
- QGL;
- probe;
- compensazioni residue;
- eventuale wrapper Demon presente nella configurazione storica analizzata;
- metodo `scan` / `rapid_scan`;
- coordinate.

### Diagnosi

Confrontare test eseguiti nelle stesse condizioni.

Non confrontare direttamente:

- bed freddo;
- bed caldo;
- geometrie mesh diverse;
- metodi diversi;

come se fossero equivalenti.


# Problemi storici DKEU / Demon

Le sezioni seguenti descrivono problemi realmente incontrati quando Phoenix utilizzava ancora DKEU.

Sono mantenute per valore diagnostico e storico, ma **DKEU non fa parte della baseline runtime Phoenix attuale**.

Per la configurazione corrente fare riferimento alle Phoenix Macros.

## Emergency shutdown subito all'avvio stampa

### Verificare

- profilo macchina selezionato in OrcaSlicer;
- presenza di `DEMON_START`;
- versione Machine G-code;
- parametri richiesti da DKEU;
- eventuale `_SPS GSTART=True`.

### Caso Phoenix

OrcaSlicer era rimasto selezionato sulla macchina Sovol invece che sulla macchina Phoenix.

Il G-code generato non conteneva il Machine Start Demon atteso.

Demon ha quindi eseguito correttamente un emergency shutdown.

### Soluzione

Selezionare il preset macchina corretto e rigenerare il G-code.

### Cosa non toccare

Non modificare Eddy, QGL o MCU se il G-code non contiene nemmeno lo start richiesto da Demon.

## `_PROBE_TAP` funziona ma Z resta errato

### Verificare

- macro `_APPLY_EDDY_Z_OFFSET` effettivamente caricata;
- eventuali override locali;
- `printer.probe.last_probe_position.z`;
- `SET_KINEMATIC_POSITION`;
- ordine delle macro.

### Caso Phoenix

Era rimasto nel `printer.cfg` un vecchio override:

`_APPLY_EDDY_Z_OFFSET`

che trasformava la macro in un no-op.

Il probe misurava correttamente il contatto, ma la coordinata Z non veniva riallineata.

### Effetto

Nel print fallito era stata osservata una differenza di circa:

`0.061 mm`

sufficiente a compromettere un first layer da `0.20 mm`.

### Soluzione

Nella configurazione storica Phoenix, la correzione consistette nel rimuovere l'override legacy e lasciare operativa la macro DKEU allora prevista.

### Cosa non fare

Non correggere la situazione modificando:

- QGL;
- motori Z;
- viti;
- flow;
- mesh;

prima di aver verificato la macro realmente eseguita.

## Storico — DKEU si comportava diversamente dai test manuali

### Verificare

- wrapper macro;
- alias `_BASE`;
- metodo mesh effettivo;
- adaptive meshing;
- log completo di `DEMON_START`.

### Metodo

Confrontare:

1. comando Klipper nativo;
2. macro Demon equivalente;
3. start completo.

Questo permette di localizzare il livello in cui cambia il comportamento.

# OrcaSlicer

## Preset apparentemente scomparsi

### Verificare

- preset macchina selezionato;
- directory profili utente;
- snapshot versionato;
- log OrcaSlicer.

### Caso Phoenix

I preset non erano stati persi.

Erano presenti sia nello snapshot Git sia nella directory utente OrcaSlicer.

Il problema reale era la macchina selezionata.

## Parametri apparentemente corretti ma stampa diversa

### Verificare

Non limitarsi alla UI.

Controllare il G-code reale per:

- temperature;
- pressure advance;
- retraction;
- start macro;
- flow;
- eventuali override.

### Caso Phoenix

In diverse fasi erano presenti valori PA differenti fra:

- `printer.cfg`;
- profilo slicer;
- comandi emessi nel G-code.

Il valore realmente eseguito è quello che conta.

# First layer

## First layer troppo schiacciato ovunque

### Verificare

- riferimento Z;
- homing Z;
- probe Eddy corrente;
- mesh realmente attiva;
- eventuale offset o babystep residuo;
- macro Phoenix effettivamente eseguite.

`_APPLY_EDDY_Z_OFFSET` apparteneva alla precedente integrazione DKEU ed è trattata nella sezione storica, non nel runtime Phoenix Macros corrente.

### Non iniziare da

- flow ratio;
- pressure advance;
- motori Z.

## First layer troppo alto ovunque

### Verificare

- riferimento Z;
- homing;
- probe;
- applicazione offset;
- mesh attiva.

## First layer corretto in alcune zone e molto errato in altre

### Verificare

- mesh;
- axis twist residuo;
- QGL;
- temperatura bed;
- contaminazione;
- geometria reale del piatto.

### Caso Phoenix

Una vecchia axis twist compensation alterava i risultati probe e mascherava circa `0.33 mm` di differenza reale.

Dopo la neutralizzazione la mesh ha iniziato a rappresentare correttamente il piatto.

## Difetto locale isolato

Se il resto del piano è coerente, considerare:

- impronta;
- grasso;
- polvere;
- contaminazione;
- difetto locale PEI.

Non usare una correzione globale per un difetto locale.

## Benchy con fondo troppo schiacciato

Un modello piccolo al centro può rivelare un problema Z globale ma non descrive l'intera geometria del bed.

Usare insieme:

- test multi-zona;
- mesh;
- stampa reale.

# Sospetti storici non confermati

## `_MESH_HANDLING`

Durante la migrazione Phoenix è stata osservata una possibile condizione logica discutibile nella macro `_MESH_HANDLING`.

Non è stata dimostrata come causa del print fallito.

Non applicare patch storiche senza una riproduzione concreta sulla configurazione che si sta analizzando.

## Rilevamento `probe_eddy_current eddy`

In uno snapshot storico una condizione sembrava considerare esplicitamente `probe_eddy_current btt_eddy` ma non sempre `probe_eddy_current eddy`.

Anche questo punto non è stato dimostrato come causa reale.

Se si utilizza DKEU indipendentemente dalla baseline Phoenix, verificare sempre il codice della versione realmente installata.

# Toolhead / cablaggio

## Cablaggio toolhead sfiora il telaio posteriore

### Verificare

- cablaggio della toolhead realmente installata;
- cavo CAN / umbilical nella configurazione Zero corrente;
- eventuale catena nella configurazione storica;
- PTFE;
- posizione massima Y;
- libertà di movimento.

### Caso Phoenix

Nella precedente configurazione con toolboard originale, il fascio toolhead poteva sfiorare il telaio posteriore e venne rimosso un elemento della catena per recuperare lunghezza utile.

Con la Sovol Zero va invece verificato il percorso fisico dell'umbilical/CAN attuale senza riutilizzare automaticamente le conclusioni geometriche della vecchia toolhead.

### Cosa non fare

Non aumentare semplicemente i limiti di corsa senza verificare fisicamente il cablaggio.

# Checklist rapida di recupero

Quando la stampante passa da "funzionante" a "stampa male":

1. non modificare subito la meccanica;
2. salvare log e configurazione attiva;
3. verificare `git diff` o backup recenti;
4. verificare homing;
5. verificare `PROBE`;
6. verificare QGL;
7. verificare mesh;
8. nella configurazione storica DKEU, verificare le macro realmente caricate;
9. verificare preset Orca;
10. verificare G-code reale;
11. ristampare lo stesso file dopo una sola correzione.

## Se homing/probe/QGL/mesh sono tutti coerenti

Spostare l'attenzione su:

- macro effettivamente eseguite;
- riferimento/homing Z;
- slicer;
- start G-code.

Non continuare a regolare meccanicamente una macchina che nei test geometrici è già coerente.

## Se mesh e first layer non concordano

Verificare:

- axis twist;
- trasformazioni probe;
- offset;
- wrapper macro;
- mesh realmente attiva.

## Se il problema compare dopo una migrazione software

Cercare prima:

- override legacy;
- macro duplicate;
- parametri deprecati;
- vecchi plugin;
- configurazioni stock copiate.

La compatibilità apparente non significa compatibilità semantica.

# Cause Phoenix dimostrate

Le cause realmente confermate durante la migrazione includono:

- uso non sicuro di identificativi MCU dinamici;
- vecchio plugin Eddy NG incompatibile con il percorso homing Mainline;
- limite `max_sensor_hz` insufficiente nella precedente configurazione Eddy NG;
- divider/frequenza LDC1612 non adatti durante la migrazione della Eddy integrata nella Zero, risolti con la configurazione Zero documentata;
- selezione TAP incompatibile nella versione DKEU storica;
- axis twist compensation legacy che falsava la mesh;
- profilo macchina Orca errato;
- override legacy `_APPLY_EDDY_Z_OFFSET` che impediva il riallineamento Z;
- route Ethernet che sottraeva Internet al Wi-Fi.

Questi problemi sono stati osservati e diagnosticati con evidenza concreta.

# Cose da non fare

Durante il troubleshooting evitare di:

- usare `ttyACM0/1` come identificativi permanenti;
- usare `set_position_z: 0` come falsa soluzione;
- patchare vecchi plugin quando esiste il supporto Mainline nativo;
- copiare valori `tap_threshold` legacy;
- aumentare indiscriminatamente `max_adjust`;
- inseguire una mesh artificialmente piatta;
- correggere Z tramite flow;
- correggere un problema macro tramite meccanica;
- applicare workaround storici a software corrente senza verifica.

# Escalation

Se il problema resta irrisolto, raccogliere almeno:

- versione Klipper;
- commit Klipper;
- versione DKEU, quando si analizza una vecchia configurazione basata su Demon;
- configurazione attiva;
- `klippy.log`;
- `moonraker.log` se pertinente;
- identificativi MCU USB e, quando presente, UUID CAN della toolhead;
- stato del bus CAN se pertinente;
- output homing;
- output probe;
- output QGL;
- mesh;
- G-code fallito;
- descrizione fisica precisa del sintomo.

Una diagnosi riproducibile vale più di molte regolazioni casuali.

---

## Navigazione

- ← **Pagina precedente:** [Validazione e calibrazione](validation-and-calibration.md)
- → **Pagina successiva:** [README](../README.md)
