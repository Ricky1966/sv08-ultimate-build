# Install CB1 Linux and Klipper Mainline on Sovol SV08

**Languages:** [Italiano](../install-cb1-mainline.md) | **English**

Last source review: **2026-08-23**.

## Purpose

This section documents the transition from the stock Sovol SV08 Linux system to a clean environment on which to install Klipper Mainline.

The main reference is:

[Rappetor/Sovol-SV08-Mainline](https://github.com/Rappetor/Sovol-SV08-Mainline)

Phoenix was migrated using this path as the foundation, with additional checks documented in this repository.

## Prerequisite

Before continuing, complete:

[Backup and rollback](backup-and-rollback.md)

Do not proceed if the stock system cannot be recovered.

## Selected Linux image

For the currently documented path use:

**BIGTREETECH CB1 V2.3.4 minimal**

The current Rappetor guide still advises against V3.0.0 or later for the SV08.

This does not mean V2.3.4 is the newest CB1 release.

It means that it is the currently documented and verified baseline for this SV08 path.

Prefer the **minimal** image, avoiding images with preinstalled and uncontrolled versions of Klipper, Moonraker, or Mainsail.

## Where to install the new system

The upstream guide allows several approaches:

- a new dedicated eMMC;
- the original eMMC after creating a complete backup;
- microSD;
- USB/FEL installation according to the upstream procedure.

For the Phoenix path documented in this repository, the **initial migration and validation are performed from MicroSD**, leaving the stock eMMC untouched.

This makes it easier to interrupt testing, replace the storage medium, or return to the original system without immediately rewriting the eMMC.

The MicroSD is not presented as inherently more reliable than eMMC for permanent use: it is chosen as the initial medium mainly for easier testing and rollback.

## Preparing the storage medium

After writing the CB1 V2.3.4 minimal image to the **MicroSD used for the initial Phoenix migration**, the `BOOT` partition must be accessible from your computer.

The same configuration changes apply when following another compatible upstream installation method.

Before editing any file:

- create a copy of `BoardEnv.txt`;
- do not delete sections that the upstream guide says must be preserved;
- prepare the SV08-specific DTB.

## SV08 DTB

The current upstream guide uses:

`sun50i-h616-sovol-emmc.dtb`

The file must be copied into:

`/dtb/allwinner/`

on the `BOOT` partition.

The configuration then uses:

`fdtfile=sun50i-h616-sovol-emmc`

This DTB allows the CB1 system to use the SV08 mainboard hardware correctly.

## BoardEnv.txt

The verified initial configuration includes:

- `bootlogo=false`
- `overlay_prefix=sun50i-h616`
- `fdtfile=sun50i-h616-sovol-emmc`
- `console=display`

and the overlays:

- `uart3`
- `ws2812`
- `spidev1_1`

Do not blindly replace the entire `BoardEnv.txt`.

Follow the structure of the file shipped with the image you are using and preserve the sections that upstream documentation says not to modify.

## system.cfg

If the printer will use Wi-Fi, configure credentials in:

`system.cfg`

A custom hostname can also be set.

Never publish in the repository:

- personal SSIDs;
- Wi-Fi passwords;
- hostnames containing personal data;
- private IP addresses used as persistent identifiers.

If the Wi-Fi password contains special characters, verify the escaping rules used by the CB1 script.

The upstream guide specifically notes that `$` may require escaping.

## First boot

After completing `BoardEnv.txt`, `system.cfg`, and the DTB:

1. safely eject the storage medium from the computer;
2. install it in the SV08;
3. power on the printer;
4. wait for Linux to boot;
5. identify the assigned IP address;
6. verify SSH access.

With the standard CB1 V2.3.4 image, the upstream guide initially uses:

- user: `biqu`
- password: `biqu`

Change the default credentials as soon as the system is under your control.

## Verify before installing Klipper

Before proceeding, verify at least:

- working SSH;
- working network;
- working DNS;
- plausible date and time;
- writable filesystem;
- correct boot device;
- no spontaneous reboots.

If microSD and eMMC are both present, use `lsblk` to verify which device is actually providing `/boot` and `/`.

This prevents configuring a different system from the one you think you booted.

## Software stack to install

With Linux booted and SSH working, install:

- KIAUH;
- Klipper;
- Moonraker;
- Mainsail;
- Crowsnest;
- optionally KlipperScreen.

## Update the base system

After the first SSH login, update package indexes and install available system updates.

Do this before installing Klipper and the other services.

If `apt` reports errors:

- do not automatically apply workarounds from old guides;
- verify network, DNS, date/time, and configured repositories first;
- distinguish a Linux image issue from a Klipper issue.

The Rappetor guide still contains historical notes about Python/Klipper incompatibilities encountered in the past.

Those are not part of the normal path documented here and should only be treated as troubleshooting if they are still reproducible.

## Install KIAUH

KIAUH is used as the installation manager for the Klipper stack.

Upstream repository:

[dw-0/kiauh](https://github.com/dw-0/kiauh)

Install KIAUH following the current instructions from the official project.

Do not use old locally saved copies of the script when a current upstream version compatible with the system is available.

## Components to install

For the SV08 Mainline baseline, install through KIAUH:

1. Klipper;
2. Moonraker;
3. Mainsail;
4. Mainsail-Config;
5. Crowsnest.

KlipperScreen is optional, but it was installed and verified on Phoenix.

Fluidd is not required if Mainsail is used.

## Klipper Mainline

The purpose of this procedure is to install the official Klipper Mainline repository, not the stock Sovol fork/build.

After installation verify:

- Klipper directory exists;
- Python environment was created correctly;
- Klipper service is installed;
- Moonraker can start;
- Mainsail is reachable.

At this stage it is normal for Klipper to be unable to communicate correctly with the MCUs if they still run stock firmware.

Do not try to solve an MCU error by randomly changing configuration before the planned flashing stage.

## Moonraker

Install Moonraker through KIAUH and verify the service starts without critical errors.

Check that:

- Mainsail can communicate with Moonraker;
- `printer_data` is created in the expected location;
- configuration files are accessible from the web interface.

## Mainsail

Install Mainsail and Mainsail-Config.

Verify the web interface is reachable at the printer address.

A working Mainsail interface does not mean the printer is ready to move.

Before MCU flashing and configuration validation, avoid unnecessary homing, movement, or heating commands.

## Crowsnest

Install Crowsnest if you intend to use a webcam.

The webcam is not required to complete the Mainline migration.

If Crowsnest reports errors, keep that diagnosis separate from Klipper: a webcam issue must not block MCU or printer-configuration work.

## KlipperScreen

KlipperScreen is optional.

On Phoenix it was installed after the base migration and proved compatible with the new environment.

It does not need to work before:

- flashing the MCUs;
- verifying Klipper;
- performing the first controlled homing.

## Record versions

After installation record:

- Klipper version;
- Klipper commit;
- Moonraker version;
- KIAUH version;
- any dirty state in the Klipper repository.

These details are essential for later troubleshooting.

A documented migration without version or commit information is much harder to compare with future problems.

## Do not import the full stock configuration yet

At this point, do not indiscriminately copy every file from the old Sovol installation into `printer_data/config`.

Instead, recover in a controlled way:

- hardware pins;
- motor parameters;
- geometry;
- MCU serial identifiers;
- heaters;
- sensors;
- fans;
- macros that are actually required.

Legacy macros, old Eddy integrations, and any DKEU components from the previous configuration must be evaluated separately and must not be imported automatically into the Phoenix baseline.

## Initial configuration

Before flashing the MCUs, prepare a minimal configuration that will later allow verification of:

- mainboard connection;
- connection to the original toolboard during the stock phase;
- temperatures;
- endstops;
- steppers;
- fans;
- probe.

The later installation of the **Sovol Zero Extruder Kit** introduces a CAN toolhead distinct from the original USB toolboard and is documented separately.

Do not consider this configuration final yet.

It will be refined in the sections dedicated to:

- MCUs;
- Sovol Zero and CAN;
- native Eddy integrated into the Zero;
- Phoenix Macros;
- calibration.

## Exit criteria

Before moving to MCU firmware, all of the following must be true:

- stable Linux;
- stable SSH;
- stable network;
- working KIAUH;
- Klipper Mainline installed;
- Moonraker installed;
- Mainsail reachable;
- stock configuration available as reference;
- backup and rollback already verified.

---

## Navigation

- ← **Previous page:** [Backup and rollback](backup-and-rollback.md)
- → **Next page:** [MCU flashing and recovery](flash-mcus.md)
