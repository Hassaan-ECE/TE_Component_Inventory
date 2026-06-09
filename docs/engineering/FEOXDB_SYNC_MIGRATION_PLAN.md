# FeOxDB Shared Sync Notes

Last updated: 2026-06-09

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
```

The TE root is intentionally click-obvious for installers: keep the current NSIS installer `.exe` at the root and put signatures, checksums, updater metadata, old installers, and other support files under folders. Shared sync state belongs under `shared\inventory\`, not next to the installer.

## Sync Behavior

- Local edits write to FeOxDB and durable local outbox records before flush.
- A backend task publishes pending local operations to S-drive after saves.
- Sync applies the latest verified snapshot first when safe, then applies operation files newer than the snapshot watermarks.
- Snapshot application is skipped when local-only pending changes exist.
- Snapshot failures do not replace local FeOxDB data.
- Covered operation files are compacted only after a snapshot and manifest are written and verified.
- The latest three snapshots are retained.

## Conflict Behavior

Concurrent non-overlapping field edits merge when both edits started from the same base version.

Example:

- Machine A changes `location`
- Machine B changes `notes`

Result:

- both fields are kept
- no duplicate row is created
- overlapping edits still use newer-operation-wins behavior and record a stale conflict

## Validation Before First TE Release

- confirm both machines preserve their local `inventory.feox`
- confirm a clean profile hydrates from `manifest.json`, snapshot, and newer ops
- confirm create/update/delete/archive/verify converges both ways
- confirm different-field concurrent edits merge
- confirm same-field concurrent edits record a conflict and keep the newer value
- confirm the S-drive operation folder compacts after snapshot publication
- confirm TE never reads or writes the ME shared root
