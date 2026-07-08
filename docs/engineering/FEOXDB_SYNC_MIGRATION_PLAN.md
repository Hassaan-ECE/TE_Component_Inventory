# FeOxDB Shared Sync Notes

Last updated: 2026-07-08

Status: cloned from the ME Inventory sync architecture for the initial TE Lab Components Inventory setup. Use `README.md` for current release state and bug-fix handoff.

## Current Shape

- each machine owns one local `inventory.feox`
- the S-drive is sync transport, not a database file
- shared sync uses operation files, snapshots, a manifest, locks, and backups
- TE uses a separate app identifier and separate shared root from ME

Shared layout:

```text
S:\Engineering\Public\Syed_Hassaan_Shah\InventoryApps\TE\
  TE Lab Components Inventory_0.1.0_x64-setup.exe
  release-support\
    v0.1.0\
  shared\inventory\
    manifest.json
    ops\<client-id>\000000000001.op.json
    snapshots\snapshot-*.snapshot.json
    locks\snapshot.lock
    backups\
    media\<entry-uuid>\
      original.<ext>
      preview.jpg
```

Planned: picture files move from machine-local paths into `shared/inventory/media/{entryUuid}/` so every client can preview the same image. See `docs/engineering/PLANNED_IMPROVEMENTS.md`.

The TE root is intentionally click-obvious for installers: keep the current NSIS installer `.exe` at the root and put signatures, checksums, updater metadata, old installers, and other support files under folders. Shared sync state belongs under `shared\inventory\`, not next to the installer.

## Sync Behavior

- Local edits write to FeOxDB and durable local outbox records before flush.
- A backend task publishes pending local operations to S-drive after saves.
- Sync applies the latest verified snapshot first when safe, then applies operation files newer than the snapshot watermarks.
- Snapshot application is skipped when local-only pending changes exist.
- Snapshot failures do not replace local FeOxDB data.
- Covered operation files are compacted only after a snapshot and manifest are written and verified.
- The latest three snapshots are retained.

## Operation Compaction (Current Rules)

Old shared op files are **not** kept forever. Compaction is conservative by design:

| Constant | Current value | Meaning |
| --- | --- | --- |
| `SNAPSHOT_OP_COMPACTION_THRESHOLD` | `1_000` | Publish a snapshot once the share has at least this many op files |
| `SNAPSHOT_MAX_AGE` | `24 hours` | Publish a snapshot when a manifest exists, ops are present, and the manifest is older than this |
| `SNAPSHOT_KEEP_COUNT` | `3` | Keep only the three newest snapshot files |

Compaction sequence during `sync_inventory`:

1. Skip snapshot publish while the syncing machine still has pending local outbox operations.
2. Publish a verified snapshot + manifest when one of the thresholds above is met.
3. Delete op files where `local_seq <= watermark` for each `client_id` recorded in the manifest.
4. Leave ops newer than the watermark untouched.

Why the S-drive can still grow:

- Low traffic can retain ops for up to 24 hours between compactions.
- If no machine syncs, compaction does not run.
- A machine with unpublished local edits skips snapshot publish on that pass.

Planned improvements: lower lab-scale thresholds, show op count in shared status, and let read-only sync clients act as safe compaction publishers. See `docs/engineering/PLANNED_IMPROVEMENTS.md` section **Shared Operation Compaction**.

## Conflict Behavior

Concurrent non-overlapping field edits merge when both edits started from the same base version.

Example:

- Machine A changes `location`
- Machine B changes `notes`

Result:

- both fields are kept
- no duplicate row is created
- overlapping edits still use newer-operation-wins behavior and record a stale conflict

## Planned UX Improvements

- **Read-only mode by default** — sessions start in lookup-only mode unless the machine opted into always-allow editing; edit attempts show a three-choice dialog (cancel / this session / always on this machine).
- **Shared operation compaction** — tune snapshot thresholds and expose shared op health so `ops/` does not crowd the S-drive.
- **Conflict visibility** — expose existing `SyncConflictRecord` data in the UI when overlapping edits lose to a newer operation.
- **Shared media** — store pictures under `shared/inventory/media/{entryUuid}/` instead of local-only paths.

Full acceptance criteria and implementation notes: `docs/engineering/PLANNED_IMPROVEMENTS.md`.

## Validation Before First TE Release

- confirm both machines preserve their local `inventory.feox`
- confirm a clean profile hydrates from `manifest.json`, snapshot, and newer ops
- confirm create/update/delete/archive/verify converges both ways
- confirm different-field concurrent edits merge
- confirm same-field concurrent edits record a conflict and keep the newer value
- confirm the S-drive operation folder compacts after snapshot publication
- confirm TE never reads or writes the ME shared root
