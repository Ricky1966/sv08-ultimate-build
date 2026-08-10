# Ricerca Mainline — 2026-08-02

## Obiettivo

Portare Phoenix, Sovol SV08 con hardware di controllo originale, FlowTech, Eddy NG e Daemon Macro, a Klipper Mainline mantenendo una procedura reversibile e documentata.

## Fonte principale

La guida di riferimento scelta è `Rappetor/Sovol-SV08-Mainline`.

Motivi:

- è specifica per SV08 con CB1;
- descrive avvio da eMMC oppure microSD;
- avverte esplicitamente che i video possono essere obsoleti;
- include configurazioni, addon Sovol aggiornati e procedura di flash delle due MCU;
- documenta il rollback e il backup del firmware stock.

## Risultati della verifica

### Immagine CB1

La guida attuale raccomanda l'immagine BTT CB1 `v2.3.4` e avverte di non usare, per ora, immagini `v3.0.0` o superiori finché non sono dichiarate stabili per questa procedura.

### Supporto di avvio

Mainline può essere installato ed eseguito da:

- eMMC;
- microSD.

Per Phoenix si preferisce inizialmente la microSD, lasciando la eMMC stock intatta come rollback fisico rapido.

### Componenti software

La procedura installa tramite KIAUH, nell'ordine:

1. Klipper;
2. Moonraker;
3. Mainsail;
4. Crowsnest;
5. KlipperScreen, solo se necessario.

### MCU

La migrazione completa richiede la compilazione e il flash del firmware Klipper Mainline su:

- mainboard MCU;
- toolboard MCU.

La guida raccomanda Katapult con offset bootloader da 8 KiB e l'uso esclusivo degli identificativi stabili `/dev/serial/by-id/`.

È vietato configurare le MCU usando `ttyACM0` o `ttyACM1`, perché possono invertirsi fra gli avvii.

### Addon Sovol

La configurazione di riferimento ripristina gli addon Sovol necessari, fra cui:

- `probe_pressure.py`;
- `z_offset_calibration.py`.

Questi file devono essere verificati rispetto alla configurazione Eddy NG già installata su Phoenix prima di copiarli.

### Eddy NG

Klipper Mainline supporta oggi le funzioni Eddy necessarie. La configurazione Sovol Eddy NG non va però trattata automaticamente come equivalente a BTT Eddy: devono essere conservati e confrontati i file specifici forniti da Sovol e le macro personalizzate già validate sulla macchina.

### Daemon Macro

Le macro vengono portate solo dopo che la configurazione base Mainline, le MCU, gli assi, le temperature e il probe sono funzionanti. Non devono essere usate per il primo avvio diagnostico.

## Strategia scelta per Phoenix

1. Salvare configurazione e dati stock.
2. Conservare la eMMC originale invariata.
3. Installare l'immagine CB1 raccomandata sulla microSD.
4. Avviare CB1 dalla microSD.
5. Installare lo stack Mainline con KIAUH.
6. Preparare la configurazione base SV08.
7. Identificare con certezza mainboard e toolboard tramite `/dev/serial/by-id/`.
8. Eseguire backup firmware stock.
9. Installare Katapult e Klipper sulle MCU seguendo la guida corrente.
10. Verificare hardware base.
11. Integrare Eddy NG.
12. Integrare le macro personalizzate.
13. Integrare Daemon Macro.
14. Eseguire QGL, mesh e primo layer.

## Criteri di successo del Day 1

- sistema avviato dalla microSD;
- Klipper Mainline, Moonraker e Mainsail attivi;
- entrambe le MCU riconosciute con seriali stabili;
- temperature plausibili a freddo;
- assi e ventole verificati;
- Eddy NG operativo;
- homing, QGL e bed mesh funzionanti;
- nessuna modifica al piatto R3men, SSR, toolhead o cablaggio durante la migrazione.

## Fonti

- https://github.com/Rappetor/Sovol-SV08-Mainline
- https://github.com/bigtreetech/CB1
- https://github.com/bigtreetech/Eddy
- https://www.klipper3d.org/Eddy_Probe.html
