# Sovol SV08 Phoenix — recupero Eddy/QGL, correzione mesh e rete

Data: 2026-08-07 / 2026-08-08

Branch di lavoro: `feature/mainline-migration`

## Obiettivo della sessione

Questa sessione ha consolidato il comportamento della Phoenix su Klipper Mainline dopo un incidente durante il QGL, ha individuato la causa principale della compensazione errata del primo layer, ha migliorato meccanicamente il piano originale e ha sistemato la connettività del CB1 per rendere affidabili data, ora, DNS e NTP.

Il risultato operativo è una macchina nuovamente stabile, con QGL affidabile, mesh coerente con la geometria reale del piatto, first layer molto più uniforme e rete configurata in modo da mantenere l'Ethernet diretta solo come collegamento locale mentre il Wi-Fi viene usato per Internet.

## Versione Klipper

Versione Mainline osservata durante la sessione:

```text
v0.13.0-718-gd8659974-dirty
```

## Configurazione Eddy NG attiva

```ini
[probe_eddy_ng eddy]
sensor_type: ldc1612
i2c_mcu: extra_mcu
i2c_software_scl_pin: extra_mcu:PB6
i2c_software_sda_pin: extra_mcu:PB7
x_offset: -16.43
y_offset: 10.22
reg_drive_current: 22
home_trigger_height: 1.8
tap_target_z: -0.400
```

Il probing statico resta il riferimento operativo principale.

## Incidente QGL e causa immediata

Durante un QGL gestito dal ramo Eddy delle macro Demon, veniva forzato un comando equivalente a:

```gcode
QUAD_GANTRY_LEVEL horizontal_move_z=8 retry_tolerance=1
```

Con Eddy NG questo portava il sensore a campionare troppo lontano dal piatto e poteva produrre correzioni Z sproporzionate. Durante l'evento il gantry si è inclinato in modo anomalo e la stampante è stata fermata togliendo alimentazione prima di ulteriori danni.

La correzione applicata al ramo Eddy della macro `_QGL` è stata riportare il comportamento al QGL nativo definito in `printer.cfg`:

```ini
{% if 'probe_eddy_ng btt_eddy' in printer or 'probe_eddy_ng eddy' in printer %}
RESPOND TYPE=COMMAND MSG="🚧 PHOENIX EDDY NG: native QGL using printer.cfg settings"
QUAD_GANTRY_LEVEL
```

## Parametri QGL consolidati

Configurazione attiva:

```ini
[quad_gantry_level]
gantry_corners:
 -60,-10
 410,420
points:
 36,10
 36,320
 346,320
 346,10
speed: 400
horizontal_move_z: 3
retry_tolerance: 0.05
retries: 5
max_adjust: 4
```

Il parametro di sicurezza `max_adjust` è stato ridotto da `30` a `4`.

Backup creato prima della modifica:

```text
~/printer_data/config/printer.cfg.before-qgl-safety-20260807.bak
```

Dopo il recupero e le regolazioni meccaniche il QGL nativo ha convergito nuovamente con risultati molto buoni. Fra i risultati osservati:

```text
retries 3/5 — spread 0.033154
retries 0/5 — spread 0.013373
retries 0/5 — spread 0.008005
```

Il QGL viene quindi considerato nuovamente validato.

## Recupero meccanico del gantry

Durante la diagnosi era stato applicato intenzionalmente uno spostamento netto di circa `+1 mm` al solo motore `z2` tramite `FORCE_MOVE` per una verifica controllata.

Lo spostamento è stato successivamente neutralizzato esattamente con `-1 mm` e la fase è stata chiusa.

Non sono richieste ulteriori correzioni individuali dei motori Z. La SV08 usa quattro motori Z a cinghia e non deve essere trattata come un piatto a viti manuali per l'allineamento del gantry.

## Homing Z e probing statico

L'homing override attivo usa il centro macchina e il probing statico Eddy per ristabilire il riferimento Z.

Sequenza Z rilevante:

```gcode
G90
G0 X191 Y165 F3600
SET_GCODE_OFFSET Z=0
G28 Z
M400
G0 Z2.0 F3600
G4 P1000
M400
PROBE_EDDY_NG_PROBE_STATIC HOME_Z=1
M400
G0 Z10 F600
```

### Nota importante su `PROBE_EDDY_NG_TAP HOME_Z=0`

L'analisi del comportamento del plugin Eddy NG ha mostrato che `PROBE_EDDY_NG_TAP HOME_Z=0` non è neutro rispetto al riferimento G-code Z: dopo il tap modifica comunque internamente `gcode_move.base_position[2]` e `homing_position[2]`.

Di conseguenza non deve essere usato per confronti laterali di quota senza eseguire successivamente un nuovo homing Z.

Per misure comparative non invasive va preferito:

```gcode
PROBE_EDDY_NG_PROBE_STATIC HOME_Z=0
```

Il probing statico con `HOME_Z=0` non altera il riferimento Z.

## Configurazione bed mesh

Configurazione attiva:

```ini
[bed_mesh]
speed: 200 #400
horizontal_move_z: 1.5
algorithm: bicubic
mesh_min: 13,15
mesh_max: 333,340
probe_count: 15,15
fade_start: 0
fade_end: 10
scan_overshoot: 4
zero_reference_position: 175,175
```

Per la calibrazione nativa non-rapid Eddy il riferimento corretto è `home_trigger_height=1.8`:

```gcode
BED_MESH_CALIBRATE_BASE METHOD=automatic HORIZONTAL_MOVE_Z=1.8
```

A `1.8 mm` la calibrazione procede senza gli avvisi che comparivano con `2.0 mm`.

Le macro Demon `BED_MESH_CALIBRATE` forzano invece il percorso rapid scan; quando serve verificare il comportamento nativo va quindi usato l'alias `BED_MESH_CALIBRATE_BASE`.

## Root cause principale: axis twist compensation

La causa principale della compensazione incoerente del first layer è stata individuata nella sezione:

```ini
[axis_twist_compensation]
```

I valori salvati erano:

```ini
#*# z_compensations = 0.151038, 0.041981, -0.193020
#*# compensation_start_x = 20.0
#*# compensation_end_x = 330.0
```

Il modulo `axis_twist_compensation.py` intercetta `probe:update_results` e aggiunge la compensazione interpolata al valore `bed_z` di ogni risultato di probing.

In pratica questi valori stavano quasi annullando una differenza reale di circa `0.33 mm` fra lato sinistro e lato destro del piatto. La mesh appariva quindi artificialmente molto più piatta di quanto fosse la geometria reale, e il lato destro risultava eccessivamente schiacciato durante il first layer.

### Evidenza sperimentale

Dopo un homing Z pulito, con Eddy statico e coordinate fisiche confrontabili, sono stati osservati circa:

```text
sinistra: 1.999 mm
destra:   1.667 mm
differenza: 0.332 mm
```

Con la vecchia axis twist compensation attiva, la mesh applicava invece una correzione laterale dell'ordine di soli `0.015 mm`.

### Neutralizzazione

Backup creato:

```text
~/printer_data/config/printer.cfg.before-axis-twist-disable-20260807.bak
```

I valori salvati sono stati neutralizzati:

```ini
#*# z_compensations = 0.0, 0.0, 0.0
```

Dopo `RESTART`, la mesh ha finalmente iniziato a rappresentare la reale differenza di quota del piatto.

Questa modifica viene considerata strutturale: i vecchi valori di axis twist compensation non devono essere ripristinati.

## Verifica della mesh dopo la neutralizzazione

Subito dopo la neutralizzazione, prima della regolazione delle viti del piatto, la mesh mostrava:

```text
range: 0.7386 mm
min:  -0.3961 mm
max:  +0.3425 mm
```

Un confronto della posizione toolhead a `G-code Z=1` mostrava circa:

```text
sinistra: 0.887267
destra:   1.204432
differenza applicata: 0.317165 mm
```

Questo valore era finalmente coerente con la differenza reale misurata da Eddy.

## Regolazione meccanica del piatto originale

Una volta resa affidabile la mesh, è stata effettuata una piccola regolazione delle quattro viti principali del piatto originale:

- lato destro: rotazione oraria per abbassare leggermente il lato destro;
- lato sinistro: rotazione antioraria per alzare leggermente il lato sinistro;
- entità indicativa: circa `1/8` di giro per vite.

Prima dell'intervento l'asse Z è stato alzato per evitare qualsiasi contatto nozzle/piatto.

Dopo la regolazione il QGL è stato ripetuto e ha convergito immediatamente con spread molto ridotto.

## Mesh finale del piatto originale

Dopo la regolazione meccanica la mesh è migliorata fino a circa:

```text
Total Variation: 0.354 mm
max: +0.194 mm
min: -0.160 mm
```

La forma residua è simile a una leggera sella/torsione, ma la variazione è ormai compatibile con la compensazione software per il piatto attuale.

Si è deciso di non intervenire ulteriormente sulle viti in questa fase.

La sola mesh da mantenere è `default`; i profili storici `default_65`, `default_70` e `default_100` sono stati eliminati.

Nota: dopo un futuro riavvio va comunque verificato che il profilo `default` finale da `0.354 mm` sia effettivamente persistito tramite SAVE_CONFIG.

## First layer test a cinque zone

File base:

```text
~/printer_data/gcodes/PHOENIX_FirstLayer_5Zone_PolyPLA.gcode
```

Test composto da cinque zone `40 x 40 mm` distribuite sul piatto.

Prima della neutralizzazione axis twist:

- front-left: buono;
- centro: buono;
- rear-left: buono;
- front-right: fortemente schiacciato e quasi trasparente;
- rear-right: evidente gradiente di schiacciamento.

Dopo la neutralizzazione axis twist e prima della correzione meccanica del piatto, le cinque zone sono diventate molto più coerenti fra loro.

### Test PLA a 210 °C

È stata creata una copia del G-code:

```text
~/printer_data/gcodes/PHOENIX_FirstLayer_5Zone_PLA_210.gcode
```

Le sole modifiche rispetto al test precedente riguardavano la temperatura nozzle, portata a `210 °C`.

Risultato qualitativo:

- front-left e front-right simili fra loro;
- centro buono;
- rear-right molto buono;
- rear-left buono con una piccola banda inferiore leggermente diversa;
- stringing ancora evidente anche a `210 °C`.

La geometria del primo layer viene quindi considerata sufficientemente stabilizzata per passare alla calibrazione del materiale.

## Parametri estrusore rilevati prima della calibrazione PLA

Configurazione estrusore rilevante:

```ini
[extruder]
rotation_distance: 6.5
microsteps: 16
full_steps_per_rotation: 200
nozzle_diameter: 0.400
filament_diameter: 1.75
pressure_advance: 0.025
pressure_advance_smooth_time: 0.035
```

Firmware retraction:

```ini
[firmware_retraction]
retract_length: 0.8
retract_speed: 30
unretract_extra_length: 0.0
unretract_speed: 30
```

I G-code già presenti non contenevano `G10`, `G11` o `SET_RETRACTION`, quindi la retraction firmware non era stata usata da quei file.

## OrcaSlicer: preparazione tower PLA

È stata preparata in OrcaSlicer una temperature tower PLA:

```text
220 -> 215 -> 210 -> 205 -> 200 -> 195 -> 190 °C
```

File caricato sulla Phoenix:

```text
temperature_tower_PLA_0.2_32m32s.gcode
```

Dal metadata del file sono emersi parametri importanti del profilo Orca:

```text
use_firmware_retraction = 0
retraction_speed = 30
deretraction_speed = 30
pressure_advance = 0.032
enable_pressure_advance = 1
```

Il G-code contiene quindi un override:

```gcode
SET_PRESSURE_ADVANCE ADVANCE=0.032
```

che prevale sul valore `pressure_advance: 0.025` presente nel `printer.cfg`.

Nei metadata compaiono sia `filament_retraction_length = 1.2` sia `retraction_length = 0.5`; prima di stampare la torre va quindi chiarito quale valore venga realmente emesso come movimenti E nel G-code.

La tower non è ancora stata avviata: la calibrazione PLA viene rimandata alla sessione successiva dopo la chiusura documentale.

## Rete Phoenix — situazione iniziale del 2026-08-08

All'avvio la Phoenix non era raggiungibile al vecchio indirizzo hotspot `<OLD_HOTSPOT_IP>` e l'AP Phoenix non risultava visibile nella scansione Wi-Fi del PC.

Il PC era collegato all'hotspot del telefono `<PHONE_HOTSPOT>`, rete:

```text
<HOTSPOT_SUBNET>
```

Una scansione della subnet ha individuato:

```text
<HOTSPOT_GATEWAY>   gateway/hotspot
<PHOENIX_WIFI_IP>  Phoenix
```

Il ping verso `<PHOENIX_WIFI_IP>` era regolare e l'SSH è stato ristabilito con:

```bash
ssh -o ServerAliveInterval=30 -o ServerAliveCountMax=3 biqu@<PHOENIX_WIFI_IP>
```

## Collegamento Ethernet diretto

Sul PC il collegamento diretto usa:

```text
Phoenix-direct
PC eno1: <PC_DIRECT_IP>/24
```

Sul CB1:

```text
eth0: <PHOENIX_DIRECT_IP>/24
```

Il problema rilevato era che il profilo Ethernet del CB1 installava anche un default gateway verso `<DIRECT_ETH_GATEWAY>`, con metrica migliore del Wi-Fi. Questo costringeva il traffico Internet e la risoluzione DNS verso la rete Ethernet locale invece che verso l'hotspot del telefono.

Profilo CB1 identificato:

```text
Wired connection 1
```

Correzione applicata:

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.never-default yes \
  ipv4.ignore-auto-dns yes

sudo nmcli device reapply eth0
```

Dopo la modifica la routing table corretta risultava:

```text
default via <HOTSPOT_GATEWAY> dev wlan0
<HOTSPOT_SUBNET> dev wlan0 src <PHOENIX_WIFI_IP>
<DIRECT_ETH_SUBNET> dev eth0 src <PHOENIX_DIRECT_IP>
```

L'Ethernet rimane quindi disponibile per il collegamento diretto locale, ma non deve essere usata come default route.

## DNS errato fornito via IPv6

Nonostante la correzione della route, ogni nome DNS veniva risolto in:

```text
<DIRECT_ETH_GATEWAY>
```

Il resolver anomalo era un DNS IPv6 dell'hotspot presente in `/etc/resolv.conf`:

```text
<HOTSPOT_IPV6_DNS>
```

Interrogandolo direttamente:

```bash
dig @<HOTSPOT_IPV6_DNS> google.com A +short
```

rispondeva:

```text
<DIRECT_ETH_GATEWAY>
```

Le query dirette a DNS pubblici funzionavano correttamente.

Il profilo Wi-Fi `<PHONE_HOTSPOT>` è stato configurato con DNS pubblici:

```bash
sudo nmcli connection modify "<PHONE_HOTSPOT>" \
  ipv4.ignore-auto-dns yes \
  ipv4.dns "1.1.1.1 8.8.8.8" \
  ipv6.ignore-auto-dns yes

sudo nmcli device reapply wlan0
```

Poiché il vecchio resolver IPv6 era rimasto temporaneamente stale in `/etc/resolv.conf`, è stato creato un backup:

```text
/etc/resolv.conf.before-phoenix-dns-fix
```

ed è stato riscritto temporaneamente:

```text
nameserver 1.1.1.1
nameserver 8.8.8.8
```

Dopo la correzione `getent` ha iniziato a restituire indirizzi reali per Google e per i pool NTP Debian.

### Verifica ancora da fare

La configurazione dei profili NetworkManager è persistente, ma la rigenerazione automatica di `/etc/resolv.conf` dopo un riavvio non è ancora stata verificata. Alla prossima accensione controllare che il resolver IPv6 errato non ricompaia.

## NTP e orologio di sistema

Situazione iniziale:

```text
Local time: Tue 2026-08-04 22:51 UTC
RTC time: Fri 1970-01-02 ...
Time zone: Etc/UTC
System clock synchronized: no
NTP service: n/a
```

Il CB1 dispone già di:

```text
/usr/sbin/ntpd
/usr/sbin/ntpdate
ntp.service enabled
```

Il servizio `ntp` era attivo ma tutti i peer avevano `reach 0` a causa del problema di route/DNS.

Dopo la correzione DNS:

```bash
sudo systemctl restart ntp
```

Il daemon ha contattato correttamente i server e ha corretto lo scarto di circa `+287526 s`.

La verifica `ntpq -pn` mostrava un peer selezionato con `*`:

```text
*193.204.114.232
```

La data è quindi tornata corretta:

```text
Sat 08 Aug 2026 06:51 UTC
```

È stato infine impostato il fuso orario:

```bash
sudo timedatectl set-timezone Europe/Rome
```

Risultato:

```text
Local time: Sat 2026-08-08 08:51 CEST
Time zone: Europe/Rome (CEST, +0200)
```

`timedatectl` continua a mostrare `NTP service: n/a` perché il sistema usa il servizio classico `ntpd` e non `systemd-timesyncd`; la sincronizzazione effettiva è stata verificata tramite `ntpq`.

L'RTC continua a riportare un valore del 1970 e non viene considerato affidabile; il riferimento temporale corretto viene ottenuto via NTP quando è disponibile la rete.

## Stato conclusivo della sessione

Al termine della sessione documentata:

- Phoenix operativa su Klipper Mainline;
- Eddy NG static probing stabile;
- QGL nativo nuovamente sicuro e validato;
- `max_adjust` QGL limitato a `4`;
- vecchia axis twist compensation neutralizzata;
- mesh coerente con la geometria reale del piatto;
- piatto originale regolato meccanicamente fino a circa `0.354 mm` di variazione;
- first layer a cinque zone molto più uniforme;
- PLA azzurro ancora soggetto a stringing;
- tower temperatura 220 -> 190 °C preparata ma non ancora stampata;
- Ethernet diretta mantenuta come rete locale senza default route;
- Wi-Fi usato come default route Internet;
- DNS pubblico configurato;
- NTP funzionante;
- fuso orario impostato su `Europe/Rome`.

## Prossima sessione: calibrazione PLA

Ordine operativo previsto, un parametro alla volta:

1. verificare il G-code della temperature tower Orca e identificare la retraction effettivamente emessa;
2. stampare la temperature tower `220 -> 190 °C`;
3. scegliere la temperatura migliore per il PLA azzurro;
4. calibrare flow ratio;
5. calibrare pressure advance;
6. calibrare retraction/stringing;
7. verificare velocità e volumetric flow;
8. consolidare i valori nel profilo Orca e nella configurazione Phoenix evitando override contraddittori.

## Regole di sicurezza da mantenere

- non usare `M84` durante le procedure di calibrazione meccanica;
- non applicare correzioni individuali ai quattro motori Z salvo diagnosi controllata e motivata;
- mantenere il QGL nativo con i parametri presenti in `printer.cfg`;
- non ripristinare i vecchi valori di `axis_twist_compensation`;
- usare `PROBE_EDDY_NG_PROBE_STATIC HOME_Z=0` per misure comparative che non devono alterare il riferimento Z;
- dopo il futuro montaggio del piatto in grafite ripetere PID bed, QGL, mesh, riferimento Z e first layer.
