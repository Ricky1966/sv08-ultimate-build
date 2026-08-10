# Day 1 — Inventario sistema attuale

Data: 2026-08-02

## Identità macchina

- Nome progetto macchina: Phoenix
- Modello: Sovol SV08
- Hostname: `SPI-XI`
- Utente SSH: `sovol`
- Indirizzo principale usato: `<PHOENIX_DIRECT_IP>`
- Secondo indirizzo rilevato sul sistema: `<SECOND_LOCAL_IP>`

## Sistema operativo stock

- Distribuzione: SPI-XI 2.3.3 Bullseye
- Base: Debian GNU/Linux 11 (Bullseye)
- Kernel: `5.16.17-sun50iw9`
- Architettura: `aarch64`
- Root filesystem: `/dev/mmcblk2p2`
- Spazio root: 6.8 GiB totali, 3.8 GiB usati, 2.9 GiB liberi
- Partizione boot: `/dev/mmcblk2p1`, 256 MiB totali
- Memoria rilevata: circa 986 MiB

## Stack applicativo presente

Nella home di `sovol` risultano installati:

- Klipper
- Moonraker
- Mainsail
- KlipperScreen
- Crowsnest
- KIAUH
- Moonraker Obico
- Moonraker Timelapse
- script e patch Sovol

## Configurazione stampante

Directory: `~/printer_data/config`

Elementi principali rilevati:

- `printer.cfg`
- `printer-backup.cfg`
- `old_Sovol_printer.cfg`
- `Macro.cfg`
- `My_Macros.cfg`
- `Demon_Klipper_Essentials_Unified/`
- `Demon_User_Files/`
- `KAMP_LiTE/`
- `RGB_LEDs.cfg`
- `saved_variables.cfg`
- `moonraker.conf`
- `moonraker-obico.cfg`
- `KlipperScreen.conf`
- `crowsnest.conf`
- `plr.cfg`
- `shell_command.cfg`

Sono inoltre presenti più archivi ZIP storici della configurazione, incluso un archivio da circa 11 MiB del 2026-07-26.

## Osservazioni operative

- La eMMC stock è attiva e monta il root filesystem come `/dev/mmcblk2p2`.
- L'installazione attuale contiene già Daemon Macro e personalizzazioni Eddy NG.
- Lo spazio libero è sufficiente per produrre un backup selettivo locale prima della migrazione.
- Non verrà eseguito alcun `apt upgrade` sulla piattaforma stock prima del backup.
- Il prossimo passo è creare un archivio verificabile della configurazione e dei file personalizzati, poi copiarlo sul PC nel repository locale senza includere credenziali o database sensibili.
