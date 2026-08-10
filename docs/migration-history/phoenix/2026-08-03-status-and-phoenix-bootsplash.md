# Sovol SV08 Phoenix — stato Mainline e bootsplash personalizzato

Data: 2026-08-03

Branch di lavoro: `feature/mainline-migration`

## Risultato della giornata

La migrazione della Sovol SV08, denominata **Phoenix**, è operativa su BTT CB1 con Klipper Mainline. L'interfaccia HDMI è funzionante con KlipperScreen, orientamento corretto e touch allineato. È stato inoltre installato e verificato un bootsplash personalizzato Phoenix.

## Stack attivo

- BTT CB1
- Klipper Mainline
- Moonraker
- Mainsail
- KlipperScreen su backend X11
- NetworkManager
- Crowsnest
- Nginx

I servizi `KlipperScreen`, `klipper`, `moonraker`, `crowsnest` e `nginx` risultavano attivi al termine delle verifiche.

## Eddy NG e homing Z

Configurazione Eddy NG attiva:

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
```

Decisioni validate:

- mantenere il probing statico;
- abbandonare il tap dinamico;
- sostituire nei due punti DKEU attivi la chiamata `PROBE_EDDY_NG_TAP` con:

```gcode
PROBE_EDDY_NG_PROBE_STATIC HOME_Z=1
```

L'homing Z risulta ripetibile e stabile con il probe statico.

## Quad Gantry Level

Per rendere affidabile il QGL è stato modificato il parametro attivo:

```ini
[quad_gantry_level]
horizontal_move_z: 3
```

Il valore precedente era `10`.

Dopo la modifica il QGL ha completato correttamente le procedure sia a temperatura ambiente sia a caldo, con range finali dell'ordine di circa `0.014 mm`.

## Riferimento Z e ripetibilità

Le prove statiche a `Z=3.0`, `Z=1.8` e sui punti QGL hanno mostrato ripetibilità molto elevata. Anche il comando standard `PROBE` è risultato stabile.

Verifica fisica al centro del piatto:

- a `Z=0.05` il foglio presenta un trascinamento appena percettibile;
- il risultato è stato ripetibile dopo un nuovo `G28 Z`;
- la regolazione finale verrà eseguita con first layer e babystep, senza forzare ulteriormente l'offset in questa fase.

## Mesh piatto attuale

Mesh salvata a `65 °C`:

- nome: `default_65`;
- griglia: `15 × 15`;
- range: circa `0.397 mm`.

Mesh precedente a `70 °C`:

- range: circa `0.375 mm`;
- forma generale coerente con la mesh a 65 °C.

Il piatto attuale è ancora supportato da molle e alcune regolazioni meccaniche erano state modificate in passato. La forma della mesh non va quindi interpretata automaticamente come inclinazione del gantry.

È previsto il montaggio del piatto in grafite R3men. Dopo la sostituzione dovranno essere ripetuti:

1. PID del bed;
2. QGL;
3. mesh;
4. riferimento Z e first layer.

## PID verificati e salvati

### Hotend MicroSwiss FlowTech a 220 °C

```ini
pid_Kp: 26.772
pid_Ki: 2.052
pid_Kd: 87.343
```

### Piatto attuale a 65 °C

```ini
pid_Kp: 74.106
pid_Ki: 1.254
pid_Kd: 1094.910
```

## KlipperScreen e orientamento HDMI

Sistema operativo CB1 verificato:

- BTT-CB1 2.3.4;
- Debian Bullseye.

Configurazione definitiva:

```ini
# /boot/system.cfg
ks_angle="inverted"
```

In `/boot/BoardEnv.txt` non deve essere presente un `extraargs` attivo per `fbcon=rotate:2`.

Il tentativo con `fbcon=rotate:2` ruotava il logo di boot nella direzione opposta e pertanto è stato rimosso. La configurazione finale mantiene:

- KlipperScreen dritto;
- touch corretto;
- bootsplash preparato appositamente nella giusta rotazione.

## Bootsplash Phoenix definitivo

Logo visualizzato:

- fenice stilizzata arancione e oro;
- testo `PHOENIX`;
- sottotitolo `Sovol SV08 · Mainline Klipper`;
- sfondo nero.

File sorgente utilizzato:

- formato: PNG;
- dimensioni sorgente: `1639 × 960`.

Dimensione finale validata per `logo.png`:

```text
540 × 316
```

Preparazione del logo prima del pack:

```bash
convert phoenix-logo-source.png \
  -fuzz 2% \
  -trim +repage \
  -resize 540x316 \
  -background black \
  -gravity center \
  -extent 540x316 \
  -rotate 180 \
  logo.png
```

La rotazione di `180°` nel file sorgente del bootsplash è necessaria per compensare l'orientamento fisico del display. Sul PC il file appare capovolto, mentre sul display della stampante appare dritto.

Generazione:

```bash
./create-bootsplash.sh
```

Bootsplash definitivo:

- dimensione approssimativa: `1008 KiB`;
- SHA256:

```text
d322680f203bb2c378e779ebeddc662a6915a31019906ff57e962b8cbff1164f
```

Installazione sul CB1:

```bash
sudo install -o root -g root -m 0664 \
  /home/biqu/bootsplash.armbian.phoenix.540 \
  /usr/lib/firmware/bootsplash.armbian

sudo update-initramfs -v -u
```

La verifica fotografica finale ha confermato:

- logo visibile;
- orientamento corretto;
- centratura corretta;
- nessun taglio;
- dimensione adeguata allo schermo;
- testo leggibile.

### Tentativi da non ripetere

- `200 × 200`: funzionante ma troppo piccolo;
- `900 × 527`: il logo non viene visualizzato dal bootsplash del CB1;
- `540 × 316`: dimensione definitiva validata.

## Backup bootsplash

Backup originale BTT conservato sul CB1:

```text
/home/biqu/bootsplash.armbian.original-20260803_172413
```

SHA256 backup originale:

```text
166aaf748b4d92e3099af562c99fb23c648ff0771da3cf24d82ce78281a54857
```

Sono stati inoltre creati backup intermedi prima delle installazioni successive del bootsplash Phoenix.

## Stato conclusivo

Al termine della sessione:

- Klipper era in stato Ready;
- QGL, homing Z e probing statico erano funzionanti;
- mesh a 65 °C salvata;
- PID hotend e bed salvati;
- KlipperScreen e touch corretti;
- bootsplash Phoenix definitivo installato e verificato;
- prima stampa Mainline ancora da eseguire.

## Prossimi passi

1. sincronizzare il repository locale con gli aggiornamenti remoti;
2. archiviare nel repository le configurazioni attive finali del CB1;
3. eseguire la prima stampa Mainline controllata;
4. completare la regolazione fine del primo layer;
5. ripetere le calibrazioni dopo il montaggio del piatto in grafite.
