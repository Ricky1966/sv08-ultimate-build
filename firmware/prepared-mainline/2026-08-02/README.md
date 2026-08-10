# Prepared Mainline firmware — 2026-08-02

Questa directory documenta i firmware utilizzati durante la migrazione originale della Sovol SV08 Phoenix a Klipper Mainline.

I binari precompilati non vengono distribuiti direttamente nella storia Git pubblica.

## Parametri di build validati

Per entrambe le MCU originali SV08:

- MCU: STM32F103
- clock reference: 8 MHz crystal
- communication: USB
- USB pins: PA11 / PA12
- Katapult application offset: 8 KiB
- Klipper bootloader offset: 8 KiB
- application start: 0x08002000

La procedura completa di build, flashing e recovery è documentata in:

`docs/flash-mcus.md`

## Checksum della build Phoenix originale

Il file `SHA256SUMS` conserva gli hash dei quattro binari prodotti e utilizzati il 2026-08-02:

- Katapult mainboard
- Katapult toolboard
- Klipper mainboard
- Klipper toolboard

Questi checksum servono come riferimento storico e di verifica.

Non implicano che quelle build debbano essere riutilizzate su nuove migrazioni.

Per una nuova installazione è preferibile ricompilare Katapult e Klipper dai sorgenti correnti usando i parametri documentati e verificando eventuali cambiamenti upstream.
