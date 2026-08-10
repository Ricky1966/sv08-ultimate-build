# Flash SV08 MCUs — Katapult and Klipper Mainline

**Languages:** [Italiano](../flash-mcus.md) | **English**

Last source review: **2026-08-10**.

## Purpose

This stage moves the two original Sovol SV08 MCUs from stock firmware to:

1. Katapult as bootloader;
2. Klipper Mainline as the application.

The SV08 has two distinct MCUs that must be handled separately:

- mainboard MCU;
- toolboard MCU.

Do not proceed until you have completed:

- `docs/en/backup-and-rollback.md`
- `docs/en/install-cb1-mainline.md`

## Upstream sources

The path follows mainly:

`Rappetor/Sovol-SV08-Mainline`

and the official project:

`Arksine/katapult`

Current upstream instructions take priority over old screenshots, videos, or local copies.

## Fundamental rule

Never flash an MCU unless you know with certainty which board you are programming.

Mainboard and toolboard must be:

- identified separately;
- backed up separately;
- programmed separately;
- verified separately.

Swapping mainboard and toolboard can produce an unbootable machine or a configuration that looks valid but is associated with the wrong MCU.

## MCU hardware verified on Phoenix

The machine used to validate this migration uses, on both mainboard and toolboard:

- `STM32F103` MCU;
- 8 MHz crystal;
- USB communication on `PA11/PA12`.

These parameters were verified on Phoenix.

Before copying them to another machine, verify that its hardware revision is compatible.

## Katapult

Katapult is installed on both MCUs.

Katapult configuration validated on Phoenix:

- processor: `STM32F103`;
- clock reference: 8 MHz crystal;
- communication interface: USB;
- USB pins: `PA11/PA12`;
- application offset: 8 KiB;
- application start: `0x08002000`.

The critical value is the 8 KiB offset.

Klipper firmware compiled afterwards must use the matching bootloader offset.

## Why Katapult

After the initial programmer-based installation, Katapult allows Klipper updates without using ST-Link every time.

Katapult supports flashing over several transports, including USB, UART, and CAN.

The Phoenix configuration described here uses USB for both mainboard and toolboard.

## Installing Katapult from source

Upstream repository:

`Arksine/katapult`

The normal upstream process is:

- clone the repository;
- configure with `make menuconfig`;
- build with `make`.

Do not automatically reuse an old Katapult binary simply because it worked in the past.

For a new installation, prefer building from current verified source while preserving the correct hardware parameters.

## Phoenix verified — Katapult used in the original migration

During the Phoenix migration on 2026-08-02, the project used:

`Arksine/katapult`

at commit:

`ec59b9b`

The Katapult binaries produced for mainboard and toolboard were identical and had SHA256:

`6e8547d966271233cc7247569eeb1075deb724d02f7890e4b4611a43dfd941a0`

This is only a historical reference for the verified migration.

It is not a requirement for future installations.

## Klipper configuration — mainboard

Configuration verified on Phoenix:

- MCU `STM32F103`;
- bootloader offset 8 KiB;
- 8 MHz crystal;
- USB `PA11/PA12`;
- initial GPIOs `PA1,PA3`.

The firmware built during the Phoenix migration had SHA256:

`9182930d07326c7f9ca031ac03e735a044fe59eb4b5cb55a00167aeff0ccb8e8`

## Klipper configuration — toolboard

Configuration verified on Phoenix:

- MCU `STM32F103`;
- bootloader offset 8 KiB;
- 8 MHz crystal;
- USB `PA11/PA12`;
- initial GPIO `PA6`.

The firmware built during the Phoenix migration had SHA256:

`8db953d31271dd7999eb80f193637ff08f31f4e73f048290040d00dc6c71ee70`

The original Phoenix build also contained the Eddy NG support that was in use at that time.

That detail belongs to the migration history.

Current Eddy configuration is covered separately in the guide to native Mainline Eddy support.

## Back up original firmware BEFORE Katapult

Before erasing or programming an MCU, read and save the original firmware from your own machine.

Create two distinct files:

- original mainboard firmware;
- original toolboard firmware.

Do not use original firmware files from third-party repositories as your only backup.

A dump from another machine is not equivalent to firmware read from your own MCU.

## Phoenix backup sizes

On Phoenix, the verified original dumps were:

- mainboard: 524288 bytes;
- toolboard: 131072 bytes.

SHA256:

- mainboard:
  `911caf60ac216c6fbd8d9bd7211b77981a79c44775f9ad056b9446c7616393c8`

- toolboard:
  `3e5b75af609e972ea5c45f2d63a12412b683ee8db78accc0073814358beebf27`

These checksums identify only the Phoenix backups.

They must not be used to validate the original firmware of another SV08.

## Identify stock mainboard and toolboard

Before migration, the two Phoenix MCUs exposed the same generic USB identifier.

For that reason, `/dev/serial/by-id/` alone was not enough to distinguish them with certainty.

While still stock, the physical paths under:

`/dev/serial/by-path/`

were used to identify which connection belonged to the mainboard and which to the toolboard.

Record this association before any flashing.

## After new firmware

After Klipper Mainline installation, prefer stable identifiers under:

`/dev/serial/by-id/`

Then update `printer.cfg`, carefully verifying that:

- `[mcu]` points to the mainboard;
- `[mcu extra_mcu]`, or the equivalent name used by the configuration, points to the toolboard.

Do not infer the association from `ttyACM0`, `ttyACM1`, or similar ordering.

Those names can change between boots.

## ST-Link

The first Katapult installation on Phoenix was performed with ST-Link and STM32CubeProgrammer.

While connecting ST-Link to the SV08:

**the printer must be powered off.**

The upstream Rappetor procedure specifies that the MCU is powered by ST-Link during this operation.

Also verify that the ST-Link firmware is up to date before starting.

## Before the first erase

Before erasing the first MCU, all of the following must be true:

- MCU identified without ambiguity;
- original dump saved;
- dump checksum calculated;
- copy of the dump stored off the printer;
- Katapult configuration verified;
- correct Klipper firmware already prepared;
- mainboard and toolboard clearly labeled;
- printer powered off;
- ST-Link wiring verified.

Only then proceed with the first erase/programming operation.

## Program Katapult on the first MCU

Work on one MCU at a time.

Recommended sequence:

1. printer completely powered off;
2. connect ST-Link to the selected MCU;
3. read the MCU identity again;
4. verify the original dump has already been saved;
5. erase flash only after this verification;
6. program the correct Katapult binary;
7. verify programming result;
8. disconnect ST-Link;
9. restore the board's normal USB connection.

Do not move to the second MCU until the first one has been verified.

## Verify Katapult

After programming Katapult and rebooting the board, verify that the MCU is detected by Linux.

Check devices under:

`/dev/serial/by-id/`

and, where useful:

`/dev/serial/by-path/`

The presence of a new USB device alone is not sufficient.

Verify that the path really belongs to the MCU you just programmed.

## Flash Klipper through Katapult

Once Katapult is working, Klipper Mainline firmware can be loaded through the bootloader without reopening the printer to use ST-Link.

Use firmware compiled specifically for that MCU.

Mainboard and toolboard may require different parameters.

Do not use mainboard firmware on the toolboard or vice versa.

After flashing:

1. wait for the MCU to reboot;
2. verify it enumerates over USB again;
3. record the new stable identifier;
4. do not start movement or heating yet.

## Repeat on the second MCU

Only after the first board is complete and verified:

1. power the printer off again;
2. connect ST-Link to the second MCU;
3. verify the original backup;
4. program Katapult;
5. verify Katapult;
6. flash the Klipper firmware intended for the second MCU;
7. verify the new USB identifier.

The procedure must always preserve a clear association between:

- mainboard;
- toolboard.

## Verify the new serial identifiers

With both MCUs running Klipper Mainline, list:

`/dev/serial/by-id/`

Record the two complete paths.

Do not use permanently in configuration:

- `/dev/ttyACM0`;
- `/dev/ttyACM1`;
- other dynamically assigned kernel names.

`by-id` identifiers are preferred because they remain stable between reboots.

## Update printer.cfg

Update the configuration only after identifying both MCUs.

Carefully verify that:

- `[mcu]` corresponds to the mainboard;
- the toolboard MCU section corresponds to the toolboard.

A reversed association can produce errors that are difficult to interpret.

After the change, do not immediately run `G28`.

Start Klipper first and inspect configuration errors.

## First Klipper start with the new MCUs

At the first start, verify system state only.

Check:

- both MCUs connected;
- no protocol errors;
- no firmware mismatch;
- readable temperatures;
- no sensor faults;
- configuration loaded;
- no duplicate or invalid pins.

Do not run yet:

- homing;
- QGL;
- mesh;
- manual moves;
- prolonged heating.

Basic hardware validation must be completed first.

## Temperature verification

With the machine cold and stationary, verify that:

- bed temperature is plausible;
- hotend temperature is plausible;
- any additional sensors report realistic values.

An impossible or out-of-range reading must be fixed before energizing heaters or motors.

## Endstop and signal verification

Check endstop and input states without moving the axes.

Where possible and safe, verify that signals change coherently when activated manually.

Do not use homing as the first test of an unverified endstop.

## Fan and output verification

Outputs controlled by the mainboard and toolboard must be checked separately.

Pay particular attention to:

- hotend fan;
- part cooling;
- electronics fans;
- LEDs or accessory outputs.

Do not assume a stock configuration works unchanged on Mainline.

## Recovery — Katapult does not appear

If the MCU is not enumerated after flashing Katapult:

1. power off the printer;
2. recheck the USB connection;
3. recheck compiled Katapult parameters;
4. verify application offset;
5. retry through ST-Link;
6. if necessary, program Katapult directly again.

Do not start editing `printer.cfg`: an MCU that does not appear at USB level is not a Klipper configuration problem.

## Recovery — Klipper firmware does not start

If Katapult is reachable but Klipper does not start after flashing, verify:

- correct MCU;
- bootloader offset;
- clock;
- USB interface;
- firmware intended for the correct board;
- build actually produced from the desired Klipper checkout.

If the problem remains, Katapult can be used to reflash a correct build.

## Recovery — MCU not recoverable over USB

If neither Klipper nor Katapult is reachable:

- power off the printer;
- return to ST-Link;
- verify the MCU is still readable;
- reprogram Katapult;
- or restore your personal original firmware.

This is why the original backup is created before any erase.

## Recovery — complete MCU rollback

To return to stock firmware:

1. use the original dump from your own mainboard;
2. use the original dump from your own toolboard;
3. program each MCU separately through ST-Link;
4. verify file checksums before programming;
5. restore the stock Linux/configuration environment when required.

Stock MCU firmware alone does not necessarily recreate the complete original Sovol environment.

## What NOT to publish

Before making a repository or diagnostic archive public, check that it does not contain:

- personal MCU dumps;
- unsanitized complete eMMC images;
- SSH keys;
- passwords;
- Wi-Fi networks;
- tokens;
- databases;
- logs containing identifying data;
- hostnames or addresses used as personal identifiers.

Phoenix dumps remain local backups.

## Phoenix verified — migration result

The Phoenix migration successfully moved:

- original Sovol mainboard to Katapult + Klipper Mainline;
- original Sovol toolboard to Katapult + Klipper Mainline;
- both MCUs to stable identification;
- configuration to the new identifiers without swapping mainboard and toolboard;
- the printer to a successful Klipper Mainline boot.

This stage was completed before final Eddy integration and mechanical calibration.

## Exit criteria

Before moving to general configuration, all of the following must be true:

- both MCUs run Klipper Mainline;
- stable serial identifiers recorded;
- `[mcu]` associated with the correct mainboard;
- toolboard associated with the correct section;
- plausible temperatures;
- no MCU errors;
- original backups preserved;
- ST-Link recovery still available.

## Next step

With the MCUs operational:

`docs/en/base-configuration.md`
