# SV08 Mainline migration checkpoint — 2026-08-02

## Stato raggiunto

Migrazione in corso su BTT CB1 2.3.4 (Debian Bullseye, kernel 5.16.17-sun50iw9), avvio da microSD, hostname `BTT-CB1`, utente `biqu`.

Componenti installati tramite KIAUH:

- Klipper Mainline
- Moonraker
- Mainsail
- Crowsnest
- Mainsail-Config
- G-Code Shell Command

Servizi base verificati: Klipper, Moonraker, Crowsnest e nginx.

## Backup

Backup Phoenix trasferito sul CB1 ed estratto in:

`/home/biqu/phoenix-backup-extracted/phoenix-mainline-backup-20260802-070459`

Archivio originale:

`phoenix-mainline-backup-20260802-070459.tar.gz`

SHA256:

`458d44e8a4c2b5244812131f1a1493a2ae5f21d8c937fa9b70ddd86372f99407`

## Configurazione migrata

Configurazione copiata in `/home/biqu/printer_data/config` preservando i file Mainline di Mainsail, Moonraker e Crowsnest.

Adattamenti già applicati:

- disabilitati temporaneamente `timelapse.cfg` e `moonraker_obico_macros.cfg` perché assenti;
- disabilitati gli include stock `get_ip.cfg` e `plr.cfg`;
- aggiornato `virtual_sdcard` a `/home/biqu/printer_data/gcodes/`;
- ripristinato `/home/biqu/demon_vars.cfg`;
- mantenuti i percorsi MCU stabili `by-path`;
- sostituito `max_accel_to_decel` con `minimum_cruise_ratio: 0.5`;
- installato Eddy NG ufficiale `vvuk/eddy-ng`, commit `1ed056b`;
- convertito il probe in `[probe_eddy_ng eddy]` con configurazione Sovol:
  - `sensor_type: ldc1612`
  - `i2c_mcu: extra_mcu`
  - `i2c_software_scl_pin: extra_mcu:PB6`
  - `i2c_software_sda_pin: extra_mcu:PB7`
  - `x_offset: -16.43`
  - `y_offset: 10.22`
  - `reg_drive_current: 22`
  - `home_trigger_height: 1.8`
- rimosse le sezioni obsolete `[probe_eddy_current eddy]` e `[z_offset_calibration]` e la calibrazione Eddy legacy nel `SAVE_CONFIG`.

Demon Klipper Essentials già contiene il supporto esplicito a `probe_eddy_ng eddy`.

## Firmware preparati

Katapult installato da `Arksine/katapult`, commit `ec59b9b`.

Profilo Katapult usato per entrambe le MCU:

- STM32F103
- cristallo 8 MHz
- USB PA11/PA12
- applicazione a offset 8 KiB (`0x08002000`)

Firmware disponibili sul CB1 in `/home/biqu/firmware-prepared`:

| File | SHA256 |
| --- | --- |
| `katapult-sv08-mainboard.bin` | `6e8547d966271233cc7247569eeb1075deb724d02f7890e4b4611a43dfd941a0` |
| `katapult-sv08-toolboard.bin` | `6e8547d966271233cc7247569eeb1075deb724d02f7890e4b4611a43dfd941a0` |
| `klipper-sv08-mainboard.bin` | `9182930d07326c7f9ca031ac03e735a044fe59eb4b5cb55a00167aeff0ccb8e8` |
| `klipper-sv08-toolboard.bin` | `8db953d31271dd7999eb80f193637ff08f31f4e73f048290040d00dc6c71ee70` |

Archivio firmware completo:

`/home/biqu/sv08-mainline-firmware-20260802.tar.gz`

SHA256 archivio:

`005fd63fdbebb166e4deb1459ef418ac14108f8e923588bddad014f6475ae7f2`

Klipper mainboard:

- STM32F103
- bootloader 8 KiB
- cristallo 8 MHz
- USB PA11/PA12
- pin iniziali `PA1,PA3`

Klipper toolboard:

- STM32F103
- bootloader 8 KiB
- cristallo 8 MHz
- USB PA11/PA12
- pin iniziale `PA6`
- sorgente MCU Eddy NG incluso (`sensor_ldc1612_ng.c`)

## Stato operativo e sicurezza

- Klipper è attualmente fermo.
- Nessuna MCU è stata ancora flashata.
- Non eseguire homing, movimenti o riscaldamenti.
- Il primo caricamento di Katapult richiede ST-Link e STM32CubeProgrammer, con stampante spenta.
- I file `mainboard_original_firmware.bin` e `toolhead_original_firmware.bin` presenti nella repository `Rappetor/Sovol-SV08-Mainline` sono dump generici forniti dalla guida e non backup personali letti dalle MCU di Phoenix.
- Prima di cancellare o programmare le schede, leggere e salvare separatamente il firmware originale della mainboard e della toolboard di Phoenix tramite ST-Link.
- Le due MCU stock espongono lo stesso seriale USB generico; per distinguerle prima del nuovo firmware si usano i percorsi fisici `by-path`.

## Passo successivo

1. Copiare in sicurezza i quattro firmware fuori dal CB1.
2. Leggere e salvare i firmware originali delle due MCU di Phoenix con ST-Link.
3. Flashare Katapult con ST-Link separatamente su mainboard e toolboard.
4. Avviare le schede in Katapult e flashare i rispettivi firmware Klipper.
5. Verificare i nuovi seriali, aggiornare `printer.cfg` senza scambiare `mcu` ed `extra_mcu`.
6. Avviare Klipper e risolvere eventuali errori di configurazione residui prima di qualunque movimento o riscaldamento.
