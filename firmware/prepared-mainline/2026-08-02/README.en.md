# Prepared Mainline firmware — 2026-08-02

**Languages:** [Italiano](README.md) | **English**

This directory documents the firmware used during the original Sovol SV08 Phoenix migration to Klipper Mainline.

Precompiled binaries are not distributed directly in the public Git history.

## Validated build parameters

For both original SV08 MCUs:

- MCU: STM32F103
- clock reference: 8 MHz crystal
- communication: USB
- USB pins: PA11 / PA12
- Katapult application offset: 8 KiB
- Klipper bootloader offset: 8 KiB
- application start: 0x08002000

The complete build, flashing, and recovery procedure is documented in:

`docs/en/flash-mcus.md`

## Original Phoenix build checksums

The `SHA256SUMS` file preserves the hashes of the four binaries produced and used on 2026-08-02:

- Katapult mainboard
- Katapult toolboard
- Klipper mainboard
- Klipper toolboard

These checksums are kept as a historical and verification reference.

They do not imply that those builds should be reused for new migrations.

For a new installation, prefer rebuilding Katapult and Klipper from current source using the documented parameters and checking for upstream changes.
