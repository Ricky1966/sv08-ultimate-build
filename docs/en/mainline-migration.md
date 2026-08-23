# Klipper Mainline migration

**Languages:** [Italiano](../mainline-migration.md) | **English**

This page is kept as a compatibility entry point for older links.

The Sovol SV08 to Klipper Mainline migration procedure is now split across the following operational guides.

## Recommended path

1. [Getting started](getting-started.md)
2. [Backup and rollback](backup-and-rollback.md)
3. [CB1 and Mainline installation](install-cb1-mainline.md)
4. [MCU flashing and recovery](flash-mcus.md)
5. [Base Mainline configuration](base-configuration.md)
6. [Sovol Zero toolhead, CAN and integrated Eddy](zero-toolhead-eddy-2026-08-17.md)
7. [Phoenix Macros](phoenix-macros.md)
8. [Validation and calibration](validation-and-calibration.md)
9. [Troubleshooting](troubleshooting.md)

The pages dedicated to the old Eddy NG configuration and DKEU integration remain available as **historical** documentation of the Phoenix development path, but they are not part of the current operational baseline.

## Important

Do not start by flashing the MCUs.

Before changing firmware, eMMC, or configuration, complete at least:

- machine inventory;
- configuration backups;
- system backup/rollback plan;
- personal MCU dumps where appropriate;
- verification of the recovery procedure.

## Phoenix

The detailed Phoenix machine history is preserved separately under:

`docs/migration-history/phoenix/`

That history records the real development and debugging path, but it must not be used as a linear operational procedure. It is currently maintained in Italian.

---

## Navigation

- ← **Previous page:** [README](../../README.en.md)
- → **Next page:** [Getting started](getting-started.md)
