# Backup and rollback — Sovol SV08 before Mainline migration

**Languages:** [Italiano](../backup-and-rollback.md) | **English**

Last source review: **2026-08-10**.

## Goal

Before modifying the operating system, eMMC, or MCU firmware, create a verifiable return path.

For a **stock SV08** migration to Klipper Mainline, consider at least four separate backup levels:

1. Klipper and service configuration;
2. Linux system / eMMC;
3. original mainboard MCU firmware;
4. original stock toolboard MCU firmware.

These four levels describe the stock machine before migration.

If your SV08 already has a different toolhead installed, such as the **Sovol Zero Extruder Kit**, also save the current toolhead firmware, configuration, and identifiers separately.

These backups are not equivalent.

Having only a copy of `printer.cfg` does not mean you have a complete rollback.

## Level 1 — Configuration

Save at least:

- `printer.cfg`;
- all included `.cfg` files;
- custom macros;
- `saved_variables.cfg`;
- Eddy configuration;
- Phoenix Macros and any other custom macros/configuration;
- `moonraker.conf`;
- `mainsail.cfg`;
- `KlipperScreen.conf`, if used;
- `crowsnest.conf`;
- any custom shell scripts;
- custom network configuration only when actually needed;
- PID values;
- input shaper;
- pressure advance;
- Z offset and useful calibration data;
- mesh and QGL results useful as historical reference.

## Level 2 — Linux system / eMMC

The safest method is to keep a bootable stock storage medium available.

Possible approaches include:

- keeping the original eMMC untouched;
- creating a complete eMMC image;
- installing Mainline on microSD first;
- using a second eMMC dedicated to the new system.

The best option is the one that lets you return quickly to a working stock system without rebuilding the machine from scratch.

## Verification rule

A backup is not considered valid until it has been:

1. copied off the printer;
2. verified as readable;
3. identified with date and source;
4. accompanied by checksums when appropriate.

Use SHA256 for binary files and images.

## Level 3 — Original mainboard MCU firmware

Before installing Katapult or Klipper Mainline on the mainboard, create a dump of the original firmware from your own machine.

Do not distribute another SV08's dump as a universal backup.

Original firmware saved from a different machine may not match your hardware revision or state.

Identify the backup with at least:

- machine;
- date;
- MCU;
- file size;
- SHA256 checksum.

## Level 4 — Original toolboard MCU firmware

This section refers to the **original Sovol toolboard present during the initial migration of the stock machine**.

Apply the same principle to the original toolboard before replacing it or modifying its firmware.

Mainboard and original toolboard must be treated as two different MCUs.

Do not assume that one dump covers both.

The toolboard backup should also include:

- machine;
- date;
- MCU;
- file size;
- SHA256 checksum.

## ST-Link

For this procedure, it is strongly recommended to have an **ST-Link** available before flashing anything.

ST-Link is the recovery path when:

- the MCU is no longer reachable over USB;
- Katapult does not start;
- firmware was written with incorrect parameters;
- original firmware must be restored;
- the MCU must be reflashed directly.

Do not begin flashing without knowing where to connect:

- SWDIO;
- SWCLK;
- GND;
- the appropriate power connection.

Always verify the pinout for your hardware revision.

## Checksums

After each binary dump, calculate SHA256.

Example:

`sha256sum filename.bin`

Keep the checksum with the original file.

Use the same principle for complete eMMC images:

`sha256sum image.img`

The checksum verifies that the file used for rollback is identical to the file originally saved.

## Recommended names

Use names that make source and date obvious.

Examples:

- `SV08-mainboard-original-YYYYMMDD.bin`
- `SV08-toolboard-original-YYYYMMDD.bin`
- `sv08-stock-emmc-YYYYMMDD.img`

Avoid generic names such as:

- `backup.bin`
- `firmware.bin`
- `old.img`

## Minimum rollback plan

Before migration, already know how to perform at least one of the following rollback paths.

### Linux rollback

Restore:

- the original stock eMMC;
- or a verified complete image;
- or a separately preserved stock storage medium.

### MCU rollback

Restore through ST-Link:

- original mainboard firmware;
- original toolboard firmware.

### Configuration rollback

Restore the original files under:

`printer_data/config/`

A complete rollback may require all three operations.

## What not to do

Do not:

- overwrite the only stock eMMC copy without a backup;
- keep original dumps only on the printer;
- rely on a single `printer.cfg` file;
- use third-party original firmware as your only recovery plan;
- begin flashing without verifying the newly created files;
- share **personal dumps** or **eMMC images** without first removing credentials and sensitive data.

## Backup verified on Phoenix

During the real migration used to validate this guide, the following were saved and verified separately:

- original mainboard firmware;
- original toolboard firmware;
- system backup;
- Klipper configuration;
- SHA256 checksums.

The current Phoenix configuration uses a **Sovol Zero Extruder Kit connected over CAN**.

The Zero backup is therefore an additional element beyond the four stock-machine levels: firmware, configuration, and CAN identifiers must be saved separately when working on the current toolhead.

The historical Phoenix backups are retained only to document that a real rollback path existed during migration.

## Exit criteria

Do not proceed to Mainline installation until all of these are true:

- configuration saved;
- stock system recoverable;
- mainboard MCU recoverable;
- original toolboard MCU recoverable;
- checksums available;
- backups stored off the printer;
- ST-Link available and usable.

---

## Navigation

- ← **Previous page:** [Getting started](getting-started.md)
- → **Next page:** [CB1 and Klipper Mainline installation](install-cb1-mainline.md)
