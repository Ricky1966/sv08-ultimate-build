# Phoenix documentation audit — August 25, 2026

This page is a review checkpoint, not an operational guide. Its purpose is to separate documentation that belongs to the current baseline from material that only preserves a historical snapshot. No candidate file is deleted or moved in this review: uncertain cases remain explicitly pending a decision.

## Classification

The following labels are used:

- **KEEP — baseline**: necessary guide or current reference;
- **KEEP — history**: useful content, but it must be read as history rather than current configuration;
- **REVIEW / MOVE**: technically useful archive material that probably belongs under `docs/migration-history/phoenix/`;
- **UPDATE**: necessary file containing sections superseded by the current baseline.

## KEEP — baseline

### `docs/en/getting-started.md`

Remains the entry point for the migration and is consistent with the separation between the reusable community baseline and Phoenix-specific customization.

### `docs/en/backup-and-rollback.md`

Remains essential. The project still requires verified backups and a rollback path before invasive changes.

### `docs/en/install-cb1-mainline.md`

Remains essential for the CB1/Mainline path.

### `docs/en/flash-mcus.md`

Remains essential for flashing and recovery.

### `docs/en/base-configuration.md`

Remains part of the Mainline baseline.

### `docs/en/native-eddy.md`

Remains the general reference for native Eddy. Phoenix-specific values must remain separate from the reusable baseline.

### `docs/en/zero-toolhead-eddy-2026-08-17.md`

Remains necessary as the machine-specific Sovol Zero/Eddy document for Phoenix, provided it continues to be presented as a tested machine configuration rather than a universal preset.

### `docs/en/phoenix-macros.md`

Remains the current reference for the Phoenix Macros layer.

Phoenix Automatic Soak is already documented here. The page still needs one final consolidation pass after the two remaining soak questions are closed: credit compatibility when `PHOENIX_START BED=...` differs from the persisted soak temperature, and the semantics of `soak_total_seconds` when `SOAK_SECONDS` differs from 600.

### `docs/en/validation-and-calibration.md`

Remains necessary as a methodology guide. The distinction between the current baseline and explicitly historical sections should be preserved.

### `docs/en/troubleshooting.md`

Remains necessary and matches the evidence-first approach used during the migration.

### `docs/en/remote-access-tailscale.md`

New guide validated on August 25, 2026. Keep it.

Validated state:

- Tailscale installed on the CB1 as a normal single node;
- no subnet routing;
- no exit node;
- `--accept-dns=false` so local DNS management remains untouched;
- SSH over Tailscale works;
- Mainsail/Moonraker over Tailscale works after adding `100.64.0.0/10` to Moonraker `trusted_clients`.

The explicit test with Phoenix and the client on two different Internet connections is still pending.

## KEEP — history / REVIEW

### `docs/en/demon-integration.md`

The file is already clearly marked as a historical development phase and states that DKEU is no longer a Phoenix runtime dependency. The content still has technical and attribution value.

**Decision for tomorrow:**

1. leave it under `docs/en/` as an easily reachable historical appendix; or
2. move it into migration history so it is even harder to confuse with the current baseline.

Preliminary recommendation: **MOVE**, keeping a small redirect/index entry if public links to the current path need compatibility.

## REVIEW / MOVE — dated snapshots still under `docs/`

### `docs/phoenix-print-recovery-2026-08-18.md`

This is a valuable snapshot of the Phoenix transition, but it contains explicitly superseded configuration: DKEU dependencies that were still present at the time, Demon `CLEAN_NOZZLE`, an older `PHOENIX_START` state, temporary mesh decisions, and old Orca hooks.

It must not be interpreted as a current guide.

Preliminary recommendation: **MOVE to `docs/migration-history/phoenix/`**.

### `docs/phoenix-status-2026-08-21.md`

This is a machine-specific snapshot and includes patches/configuration that precisely describe the August 21 state, including dependencies and wrappers later removed.

Preliminary recommendation: **MOVE to `docs/migration-history/phoenix/`**.

### `docs/phoenix-status-2026-08-22.md`

This is an important snapshot of the birth of thermal credit, but the August 25 Automatic Soak design has superseded it in fundamental areas: disk persistence, wall-clock timestamps, restart recovery, the 60-second recovery guard, and credit that can grow beyond 600 seconds.

Preliminary recommendation: **MOVE to `docs/migration-history/phoenix/`**.

### `docs/phoenix-macros-validation-2026-08-22.md`

Useful as evidence of progressive physical validation, but it records intermediate macro counts (`22`, then `21`) that predate the new Automatic Soak user macros.

Preliminary recommendation: **MOVE to `docs/migration-history/phoenix/`** and keep only the consolidated current state in `docs/phoenix-macros.md`.

### `docs/phoenix-session-closeout-2026-08-22.md`

By definition this is a session closeout. It contains pending items that were later closed and a runtime snapshot that subsequently evolved.

Preliminary recommendation: **MOVE to `docs/migration-history/phoenix/`**.

### `docs/phoenix-input-shaper-validation-2026-08-22.md`

The measurement remains useful and contains the valid reminder to repeat Input Shaper after reinstalling the insulated enclosure panels. However, it is a dated machine-specific validation, not a community guide.

Preliminary recommendation: **MOVE to `docs/migration-history/phoenix/`**, while keeping only the general recalibration principle in the calibration guide.

## UPDATE — inconsistencies to fix without deleting history

### Italian and English README files

The published README still describes the August 23 baseline with a fixed count of `21` `PHOENIX_*` macros and states that `save_variables` is not used in the Phoenix layer.

After Phoenix Automatic Soak was introduced, the fixed count no longer represents the baseline well, while `save_variables` is now intentionally used for persistent thermal state in `phoenix_variables.cfg`.

Recommendation: replace the rigid count with a functional description of the current set, and clarify that `save_variables` is used only by the Phoenix persistence subsystem rather than as a DKEU leftover.

The README should also add an explicit link to `docs/remote-access-tailscale.md` without turning Tailscale into a migration requirement: it is an optional remote-access feature.

### `docs/phoenix-macros.md`

The page is correct in structure, but some wording still calls Automatic Soak "under validation". Persistent recovery, wall-clock timestamps, offline decay, the 60-second guard, and autonomous return to `VALID` were actually validated on August 25.

The two technical cases listed above should remain open until they are explicitly tested.

## Recommended documentation structure

Target for the next cleanup:

```text
docs/
  getting-started.md
  backup-and-rollback.md
  install-cb1-mainline.md
  flash-mcus.md
  base-configuration.md
  native-eddy.md
  zero-toolhead-eddy-2026-08-17.md
  phoenix-macros.md
  validation-and-calibration.md
  troubleshooting.md
  remote-access-tailscale.md
  migration-history/
    phoenix/
      ...dated snapshots and session records...
```

The proposed rule is simple: **guides under `docs/` should say what to do today; snapshots should explain what happened then**.

## Deferred decisions

No moves or deletions are performed by this audit.

Tomorrow decide:

1. whether to move `demon-integration.md` into history;
2. which dated snapshots to move together under `docs/migration-history/phoenix/`;
3. whether old public paths need redirect/compatibility stubs;
4. update the IT/EN README files for Automatic Soak and Tailscale;
5. consolidate `phoenix-macros.md` after the last two thermal-consistency tests.
