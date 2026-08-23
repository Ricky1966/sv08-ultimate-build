# Base configuration — Sovol SV08 on Klipper Mainline

Ultima revisione: **2026-08-23**.

## Scopo

Questa fase costruisce una configurazione Klipper Mainline minima e controllabile prima di integrare:

- Sovol Zero Extruder Kit e collegamento CAN;
- Eddy nativo integrato nella Zero;
- mesh;
- Phoenix Macros;
- calibrazioni slicer;
- modifiche hardware non necessarie alla migrazione.

L'obiettivo non è copiare integralmente il vecchio `printer.cfg` Sovol.

L'obiettivo è recuperare solo i parametri hardware realmente necessari e verificarli uno per volta.

## Prerequisiti

Devono essere già completati:

- [Backup and rollback](backup-and-rollback.md)
- [Installazione CB1 e Klipper Mainline](install-cb1-mainline.md)
- [Flashing e recovery MCU](flash-mcus.md)

In questa fase della migrazione stock, mainboard e toolboard originale devono risultare raggiungibili da Klipper Mainline.

La Sovol Zero CAN viene introdotta nella fase successiva.

## Regola fondamentale

Una configurazione stock funzionante sul Klipper Sovol non è automaticamente una configurazione Mainline valida.

Separare sempre:

- parametri hardware;
- macro;
- workaround legacy;
- configurazione probe;
- configurazione slicer.

I parametri hardware possono spesso essere recuperati.

Macro e workaround devono invece essere rivalutati.

## MCU

Usare identificativi stabili:

`/dev/serial/by-id/`

Evitare configurazioni permanenti basate su:

- `/dev/ttyACM0`
- `/dev/ttyACM1`

Durante la fase iniziale con hardware stock verificare che:

- `[mcu]` corrisponda alla mainboard;
- l'eventuale seconda MCU USB corrisponda realmente alla toolboard originale.

Non dedurre l'identità della MCU dall'ordine di enumerazione USB.

La configurazione Phoenix attuale utilizza invece la **Sovol Zero Extruder Kit via CAN**, con una MCU toolhead identificata tramite UUID CAN.

## Parametri stepper

Recuperare dalla configurazione stock i parametri motore necessari, fra cui:

- pin step;
- pin direction;
- pin enable;
- microsteps;
- rotation distance;
- gear ratio;
- endstop pin;
- position endstop;
- position min;
- position max;
- homing speed;
- homing direction.

Non modificare contemporaneamente più parametri meccanici senza una ragione verificata.

## Parametri motori Phoenix da non alterare casualmente

Durante la migrazione Phoenix sono stati mantenuti come valori meccanici protetti:

- `rotation_distance: 40` per gli assi interessati;
- `gear_ratio: 80:12` dove previsto;
- `microsteps: 16`.

Questi valori appartengono alla configurazione verificata della macchina Phoenix.

Prima di usarli su un'altra SV08, confrontarli con la propria configurazione stock.

## Estrusore

Recuperare dalla configurazione stock e dall'hardware effettivamente installato:

- step pin;
- dir pin;
- enable pin;
- microsteps;
- rotation distance;
- full steps per rotation;
- nozzle diameter;
- filament diameter;
- heater pin;
- sensor type;
- sensor pin;
- min temp;
- max temp;
- pressure advance solo come valore provvisorio.

Se l'hotend o l'estrusore sono stati modificati, non considerare i valori Phoenix come baseline universale.

## Storico Phoenix — estrusore precedente alla Sovol Zero

Durante la fase di calibrazione Phoenix risultavano:

- `rotation_distance: 6.5`
- `microsteps: 16`
- `full_steps_per_rotation: 200`
- `nozzle_diameter: 0.400`
- `filament_diameter: 1.75`
- `pressure_advance: 0.025`
- `pressure_advance_smooth_time: 0.035`

Questi valori descrivono una fase precedente della macchina Phoenix, prima della baseline Sovol Zero attuale, e non devono essere copiati automaticamente.

La calibrazione materiale definitiva viene trattata separatamente.

## Storico Phoenix — firmware retraction

Phoenix disponeva di:

- `retract_length: 0.8`
- `retract_speed: 30`
- `unretract_extra_length: 0.0`
- `unretract_speed: 30`

Durante i test analizzati, i G-code non utilizzavano `G10`, `G11` o `SET_RETRACTION`.

Quindi questi valori non erano responsabili del comportamento osservato nelle stampe analizzate.

## Heater e sensori

Prima di effettuare qualsiasi riscaldamento verificare a macchina fredda che:

- temperatura hotend sia plausibile;
- temperatura bed sia plausibile;
- eventuali sensori aggiuntivi siano plausibili.

Una temperatura fuori scala deve essere risolta prima di attivare un heater.

Non usare il riscaldamento come test iniziale di un sensore non verificato.

## PID

I valori PID presenti nella configurazione stock possono essere mantenuti solo come riferimento iniziale.

Dopo:

- cambio hotend;
- cambio heater;
- cambio bed;
- modifica significativa della massa termica;

eseguire una nuova calibrazione PID.

I PID Phoenix non sono baseline universale.

## Endstop

Verificare gli endstop prima dell'homing.

Controllare:

- stato a riposo;
- stato quando attivati;
- coerenza logica;
- pin corretti.

Non usare `G28` come primo test di un endstop non verificato.

## Primo movimento

Il primo movimento deve avvenire solo dopo aver verificato:

- MCU;
- temperature;
- endstop;
- configurazione stepper;
- limiti macchina.

Muovere un solo asse alla volta, con spostamenti piccoli.

Verificare direzione e distanza.

Se la direzione è errata, correggere la configurazione prima di continuare.

## Non usare ancora homing Z

In questa fase non definire ancora il comportamento definitivo di homing Z.

La Phoenix è passata prima dalla vecchia Eddy NG e successivamente alla **Eddy integrata nella Sovol Zero** tramite supporto nativo Mainline.

Il percorso attuale Zero + Eddy è documentato in:

[Sovol Zero toolhead, CAN ed Eddy integrato](zero-toolhead-eddy-2026-08-17.md)

Evitare workaround temporanei come:

`set_position_z: 0`

per dichiarare falsamente Z homed.

## QGL

La SV08 utilizza quattro motori Z e `QUAD_GANTRY_LEVEL`.

La configurazione QGL deve essere verificata sulla geometria reale della propria macchina.

Non modificare:

- gantry corners;
- probe points;
- max adjust;
- retry tolerance;

solo per "far passare" QGL.

Un QGL che richiede correzioni eccessive può indicare un problema meccanico o una configurazione errata.

## Storico Phoenix — QGL prima della Sovol Zero

Prima dell'installazione definitiva della Sovol Zero, la configurazione Phoenix utilizzava:

- gantry corners: `(-60,-10)` e `(410,420)`
- points:
  - `(36,10)`
  - `(36,320)`
  - `(346,320)`
  - `(346,10)`
- speed: `400`
- horizontal move Z: `3`
- retry tolerance: `0.05`
- retries: `5`
- max adjust: `4`

`max_adjust` era stato ridotto da `30` a `4` come misura di sicurezza.

Con la precedente configurazione Eddy NG già gestita dal supporto nativo Mainline e macchina a caldo, questo setup aveva completato QGL con:

- retries: `3/5`
- probed points range: circa `0.018 mm`
- tolerance: `0.050 mm`

Questi valori documentano quella specifica fase della Phoenix e non devono essere interpretati come risultato QGL della configurazione Zero corrente né come requisito universale.


## Homing X e Y

Dopo aver verificato endstop, direzioni e limiti, testare separatamente l'homing degli assi X e Y.

Controllare che:

- l'asse si muova nella direzione prevista;
- l'endstop venga raggiunto senza urti;
- il movimento si arresti correttamente;
- la posizione finale sia coerente con la geometria della macchina.

Se X o Y si muovono nella direzione sbagliata, correggere la configurazione prima di proseguire.

Non compensare un errore di direzione modificando limiti o coordinate macchina.

## Limiti macchina

Verificare:

- `position_min`;
- `position_max`;
- posizione degli endstop;
- area realmente raggiungibile;
- eventuali margini necessari per probe, toolhead o accessori.

I limiti devono descrivere la macchina fisica reale.

Non aumentare `position_max` soltanto per raggiungere una coordinata usata da una macro.

Le macro devono rispettare i limiti reali della macchina, non il contrario.

## Centro macchina e posizioni di servizio

La macchina Phoenix utilizza come riferimento operativo centrale:

- X `191`
- Y `165`

Questo punto è stato usato durante homing e verifiche.

È un riferimento Phoenix verificato, non una definizione universale della geometria SV08.

Eventuali posizioni di:

- park;
- nozzle cleaning;
- purge;
- homing;
- probe calibration;

devono essere controllate rispetto alla propria toolhead e ai propri offset.

## Toolhead e ingombri

Prima di utilizzare coordinate vicine ai bordi verificare fisicamente:

- dimensioni toolhead;
- offset probe;
- cavi;
- PTFE;
- eventuale umbilical;
- nozzle cleaner;
- accessori montati;
- enclosure.

Una coordinata valida nella configurazione stock può diventare non sicura dopo una modifica hardware.

## Ventole

Verificare separatamente ogni ventola controllata da Klipper.

Identificare almeno:

- hotend fan;
- part cooling fan;
- electronics fan;
- eventuali ventole aggiuntive.

Controllare:

- pin;
- polarità logica;
- comportamento automatico;
- velocità minima utile.

Una ventola hotend configurata male può danneggiare il sistema termico anche se Klipper non segnala errori immediati.

## Heater safety

Prima di effettuare PID o stampe verificare:

- heater corretto associato al sensore corretto;
- temperatura iniziale plausibile;
- incremento temperatura coerente con l'heater attivato;
- nessun altro sensore che aumenti per errore.

Effettuare i primi test termici in modo controllato.

Se attivando il bed aumenta la temperatura dell'hotend, o viceversa, fermarsi e correggere la configurazione.

## Macro stock

Non importare ancora automaticamente:

- start print;
- end print;
- clean nozzle;
- purge;
- heat soak;
- QGL wrapper;
- mesh wrapper;
- pause/resume personalizzati;
- macro Eddy;
- vecchie macro DKEU/Demon.

Queste macro possono contenere dipendenze da:

- vecchio Klipper Sovol;
- vecchie API;
- nomi MCU precedenti;
- vecchio probe;
- coordinate Phoenix specifiche;
- workaround non più necessari.

Portarle solo dopo che la configurazione hardware di base è stabile.

## Axis twist

Non importare automaticamente una vecchia compensazione `axis_twist_compensation`.

Una compensazione salvata prima di:

- cambio probe;
- modifica toolhead;
- migrazione firmware;
- modifica cinematica;
- nuova calibrazione Z;

può non essere più valida.

Sulla Phoenix la vecchia compensazione è stata neutralizzata durante il recupero Mainline e non è stata considerata una baseline da ripristinare.

## Mesh

Non creare ancora una bed mesh definitiva.

La mesh dipende da:

- probe;
- offset;
- homing Z;
- temperatura bed;
- temperatura nozzle;
- QGL;
- metodo di probing.

La configurazione mesh viene trattata dopo l'integrazione Eddy nativa.

## Cosa deve funzionare prima di Zero ed Eddy

Prima di passare all'installazione della Zero e alla configurazione del relativo probe Eddy devono essere verificati:

- mainboard e toolboard originale della fase stock;
- temperature;
- heater;
- ventole;
- endstop X;
- endstop Y;
- movimento X;
- movimento Y;
- movimento Z controllato;
- limiti macchina;
- geometria base;
- QGL configurato ma non necessariamente eseguito;
- nessun errore Klipper residuo relativo all'hardware base.

## Cosa NON deve ancora essere considerato definitivo

A questo punto possono ancora cambiare:

- homing Z;
- Z offset;
- probe offset;
- mesh;
- rapid scan;
- tap;
- QGL workflow;
- start print;
- Phoenix Macros;
- purge;
- nozzle cleaning;
- material calibration.

Questo è normale.

La configurazione base serve a dimostrare che la macchina è controllabile in sicurezza prima di aggiungere automazioni e compensazioni.

## Storico Phoenix — stato prima della migrazione definitiva a Zero ed Eddy

Sulla Phoenix, nella fase precedente all'installazione definitiva della Zero, erano già stati verificati:

- MCU Mainline;
- comunicazione mainboard/toolboard originale;
- assi;
- temperature;
- endstop;
- parametri meccanici;
- QGL base;
- accesso stabile via Mainsail e SSH.

I problemi residui riguardavano il percorso probe/homing Z, non la configurazione hardware fondamentale.

Questa distinzione ha permesso di evitare modifiche casuali a motori, gantry o cinematica durante il debugging Eddy.

## Criterio di uscita

Prima di passare alla Sovol Zero e alla relativa Eddy devono essere vere tutte queste condizioni:

- configurazione funzionante senza errori hardware di base;
- mainboard correttamente identificata;
- toolboard originale correttamente identificata;
- temperature plausibili;
- heater verificati;
- ventole verificate;
- X e Y homing funzionanti;
- direzioni stepper corrette;
- limiti macchina coerenti;
- nessuna macro legacy necessaria al semplice controllo della macchina;
- backup ancora disponibile.

---

## Navigazione

- ← **Pagina precedente:** [Flashing e recovery MCU](flash-mcus.md)
- → **Pagina successiva:** [Sovol Zero, CAN ed Eddy integrato](zero-toolhead-eddy-2026-08-17.md)
