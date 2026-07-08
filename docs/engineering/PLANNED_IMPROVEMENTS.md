# Planned Improvements

Last updated: 2026-07-08

This document records prioritized UX and sync improvements for TE Lab Components Inventory. Current release state and smoke guidance remain in `README.md`.

## Priority Order

1. **Read-only mode by default** — prevent accidental edits for lookup-only users, with an opt-in "always allow editing" preference per machine
2. **Shared operation compaction** — keep S-drive `ops/` growth under control without weakening safety
3. **Conflict visibility** — surface when another machine won an overlapping edit
4. **Shared media storage** — make picture paths work across machines

Lower-priority open work (release smoke, TE icon, custom fields, performance baselines) stays in `README.md` under **Open Work**.

---

## 1. Read-Only Mode (Default at Startup)

### Problem

Most lab users only look up parts. Today the app opens in full edit mode, so accidental clicks can create, edit, archive, or delete entries.

### Goal

Protect lookup-only users from accidental edits while still making it easy for editors to opt in — either for one session or permanently on that machine.

### Proposed Behavior

**Launch default**

- On each launch, the app starts in **read-only mode** unless this machine has the persisted preference **Always allow editing**.
- Show a clear header state, for example **Editing off**, with a short explanation: *"Lookup mode is on to prevent accidental changes."*
- Machines without the persisted preference return to read-only on every launch.

**Edit attempt prompt**

When a user tries to mutate data while read-only — add entry, save edits, verify toggle, archive/restore, delete, pick/change picture, or any equivalent context-menu action — show a confirmation dialog instead of failing silently:

| Action | Result |
| --- | --- |
| **Cancel** | Close the dialog; remain read-only; do not open the underlying edit flow |
| **Enable editing for this session** | Unlock mutations for the current app session only; proceed with the requested action if still valid |
| **Always allow editing on this machine** | Persist an opt-in preference and unlock editing immediately; future launches on this machine open in edit mode until the preference is cleared |

The same three choices should be available from an explicit header control such as **Enable Editing**, not only from blocked edit attempts.

**Persisted preference**

- Store per-machine preference locally, for example `teComponentInventory.alwaysAllowEditing = true`.
- This is not synced across machines. A lookup kiosk can stay read-only while an editor laptop opts in permanently.
- Add a settings or header affordance to return to the safe default: **Require edit confirmation on startup** (clears the persisted preference).

**What still works in read-only mode**

- Load inventory and archive views
- Search, filter, sort, and column visibility
- Open saved links, online search, and local picture paths when they exist on the machine
- Excel export
- Shared sync **pull** (apply snapshots and remote operations) so data stays current for lookups
- Updater check/download/install

**What is blocked in read-only mode**

- Add entry
- Save entry edits
- Toggle verified
- Archive / restore
- Delete
- Pick or change picture path
- Any backend mutation command

**Sync behavior while read-only**

- Block new local mutations, so read-only clients do not enqueue fresh outbox operations.
- Still run shared sync **pull** on the normal interval so lookup data stays current.
- Still allow snapshot publish and op compaction during sync when the machine has **no pending local outbox** — read-only lookup machines become safe compaction helpers on the S-drive.
- If a machine still has stale pending outbox data from an earlier edit session, finish publishing that backlog only after editing is re-enabled, or surface a status warning if backlog compaction is needed before new edits.

### Implementation Notes

- Add `editMode: "read-only" | "editing"` in the frontend shell, derived from launch default plus session overrides.
- Add `EditModeDialog` for the three-choice prompt; reuse it from header **Enable Editing** and from mutation entry points.
- Gate `useInventoryEntryMutations`, context-menu destructive actions, verify toggles, and the entry dialog save path behind an `ensureEditingEnabled()` helper that opens the dialog when needed.
- Persist `alwaysAllowEditing` in local browser/Tauri storage using the existing `teComponentInventory.*` preference namespace.
- Add a backend guard on mutation commands (`create_entry`, `update_entry`, `toggle_verified_entry`, `set_archived_entry`, `delete_entry`, `pick_picture_path`) that rejects work when the app is in read-only mode.
- Pass edit mode from frontend to backend per session via a lightweight `set_edit_mode` command on startup and whenever the user changes mode.
- Reuse the status strip to announce mode changes.

### Acceptance Criteria

- Fresh launch on a machine without the preference cannot mutate inventory until the user opts in through the dialog.
- **Cancel** leaves the app read-only and does not open the pending edit flow.
- **Enable editing for this session** unlocks edits until the app is closed.
- **Always allow editing on this machine** persists across restarts on that machine only.
- Lookup workflows (search, open link, export) work without enabling editing.
- A read-only machine still receives remote changes from shared sync.
- Clearing the persisted preference returns the machine to read-only on next launch.

---

## 2. Shared Operation Compaction

### Problem

Operation files under `shared/inventory/ops/` are the safe replication log, but they consume S-drive space if they are not pruned aggressively enough. The design already deletes covered ops — we need to understand the current rules and tune them for a small lab with many read-only clients.

### Design Principle: Ops Are a Tail, Not a History

Operation files are **temporary transport**, not the source of truth. Once an op's changes are reflected in the latest verified snapshot **and** every client can catch up without that file, it should not remain on the S-drive.

In practice the share should only need:

```text
manifest.json          → points at the latest snapshot
snapshots/             → compacted DB state + per-client watermarks
ops/{client_id}/       → only the uncompacted tail since those watermarks
```

Successful merges do not need to be stored forever. The snapshot is the compacted truth; ops are just the delta queue between snapshot generations.

The one rule that prevents naive "delete on success": **a single machine applying an op locally does not mean every other machine has seen it yet**. Compaction must be based on shared watermarks recorded in a verified manifest, not on one client's local apply callback.

### How It Works Today

Old operation files **are** deleted, but only through a controlled compaction path:

1. Each local edit is written to FeOxDB and pushed to `ops/{client_id}/{local_seq}.op.json`.
2. During `sync_inventory`, `maybe_publish_snapshot()` may publish a new snapshot plus manifest when:
   - no manifest exists yet, or
   - total op file count is at least `SNAPSHOT_OP_COMPACTION_THRESHOLD` (**1,000**), or
   - there is at least one op file and the current manifest is older than `SNAPSHOT_MAX_AGE` (**24 hours**).
3. Snapshot publish is skipped while the syncing machine still has **pending local outbox operations**.
4. After a verified snapshot and manifest write, `compact_covered_operations()` deletes op files where `local_seq <= watermark` for each `client_id` listed in the manifest watermarks.
5. `prune_old_snapshots()` keeps only the latest **3** snapshot files.

Relevant code:

- `backend/src/sync/snapshot/publish.rs` — publish decision and compaction trigger
- `backend/src/sync/snapshot/io.rs` — `compact_covered_operations`, `prune_old_snapshots`
- `backend/src/sync/snapshot/types.rs` — `SNAPSHOT_OP_COMPACTION_THRESHOLD`, `SNAPSHOT_MAX_AGE`, `SNAPSHOT_KEEP_COUNT`
- `backend/src/sync/apply.rs` — watermark advancement while pulling remote ops

This is intentionally conservative: covered ops are removed only after a verified snapshot proves they are safe to drop.

### Why S-Drive Can Still Fill Up

| Scenario | Effect |
| --- | --- |
| Low edit volume | Ops can remain on the share for up to **24 hours** even when a manifest already exists |
| Pending local outbox on a syncing machine | That machine skips snapshot publish for that sync pass, delaying compaction |
| No machine syncs regularly | Snapshots and compaction do not run; ops accumulate |
| Client not present in snapshot watermarks | That client's covered op files are not deleted during compaction |
| Many machines / long history | Each uncovered op is a small JSON file; volume adds up when compaction lags |

The safety model is sound. The main issue is **retention duration and publish frequency**, not missing deletion logic.

### Proposed Improvements

**Treat compaction as "keep only the tail"**

- After each verified snapshot publish, delete every op file at or below the manifest watermark for that `client_id` — this is already the intended behavior.
- Make the uncompacted tail as short as possible by publishing snapshots more often instead of inventing a second long-term op archive.
- Do **not** delete an op the moment one machine merges it locally; wait until the manifest watermark proves it is safe for all clients to hydrate via snapshot + remaining tail.
- When a lagging client starts up, it should either apply the latest snapshot or consume only ops newer than that snapshot's watermarks. Covered ops should already be gone.

**Tune lab-scale compaction constants**

- Lower `SNAPSHOT_OP_COMPACTION_THRESHOLD` from `1_000` to a lab-appropriate value, for example **50–100** ops.
- Shorten `SNAPSHOT_MAX_AGE` from **24 hours** to something like **1–4 hours** for TE traffic patterns.
- Revisit `SNAPSHOT_KEEP_COUNT = 3` only if snapshot size becomes an issue; op files are the primary crowding concern.

**Use read-only clients as natural compactors**

- Read-only machines still pull sync and should have **zero pending local outbox operations**, so they are good snapshot publishers.
- After read-only mode ships, document that lookup machines help shared hygiene simply by staying open and syncing.
- Optionally prefer snapshot publish from clients with no pending local ops and the freshest watermark view.

**Expose shared op health in the UI**

- Extend shared status to include:
  - current op file count on the share
  - last snapshot time / id
  - last compaction count
  - whether this machine skipped publish because of pending local ops
- Show a subtle status-strip message when op count stays high, for example *"Shared sync log is large; open the app on an editor machine or wait for automatic compaction."*

**Manual maintenance affordance (optional)**

- Add an admin-only **Compact shared sync now** action on machines with editing enabled and no pending local ops.
- This should call the same safe snapshot + compaction path, not a separate delete sweep.

**Housekeeping**

- Remove empty `ops/{client_id}/` directories after compaction (already attempted today).
- Add smoke coverage that creates > threshold ops, publishes a snapshot, and asserts op count drops while newer-than-watermark ops remain.

### Acceptance Criteria

- After normal editing activity, covered op files are removed without manual intervention.
- Uncovered ops newer than the manifest watermark always remain.
- With tuned thresholds, typical TE daily usage does not keep more than one compaction window of op files on the share.
- Read-only clients can participate in compaction without publishing their own edits.
- Shared status makes op growth visible before the S-drive becomes a problem.

---

## 3. Conflict Visibility

### Problem

The backend already records stale-operation conflicts when an incoming remote change loses to newer local state, but the UI does not show them. Users cannot tell that another machine's edit was ignored or which values won.

### Existing Backend Support

Conflict records are already persisted in FeOxDB under the sync conflict keyspace via `SyncConflictRecord`:

- `entry_uuid`
- incoming vs current operation ids, client ids, local sequences, and mutation timestamps
- `reason` (currently `stale_incoming_operation`)
- `detected_at_utc`

Entry state also tracks `changed_fields` per operation, which is the basis for field-level merge and overlap detection.

Relevant code today:

- `backend/src/sync/conflicts.rs` — `record_stale_operation_conflict`
- `backend/src/storage/sync_state.rs` — conflict record CRUD and scan
- `backend/src/domain/entry_changes.rs` — changed-field projection for edits

### Goal

Make conflicts visible at the row and entry level, with enough detail to answer:

- Was this entry affected by a conflict?
- Which machine/time won?
- Which fields kept the winning value?

### Proposed Behavior

**Data exposure**

- Add a Tauri query command, for example `list_entry_conflicts` or include unresolved conflicts in `load_inventory` / `sync_inventory` payloads.
- Return unresolved conflicts keyed by `entryUuid`, newest first.
- Extend conflict records if needed to store overlapping field names and the winning field values at detection time. If we can derive that from existing operation payloads and entry state, prefer derivation over schema churn.

**Inventory table**

- Show a subtle conflict indicator on affected rows (badge, icon, or row accent distinct from verify/archive/lifecycle colors).
- Tooltip or status text example: *"A newer edit on another machine kept the current location; a remote notes change was not applied."*

**Entry dialog**

- When opening an entry with unresolved conflicts, show a banner above the form.
- List each conflict with:
  - detection time
  - remote client/source label when available
  - fields that overlapped
  - fields that won locally vs remotely
- Provide **Dismiss** or **Mark reviewed** so acknowledged conflicts do not keep alerting forever.

**After sync**

- When sync applies remote changes that overwrite local values the user had open, show a one-time status message naming the entry and changed fields.

### Implementation Notes

- Start with conflicts already stored as `SyncConflictRecord`; avoid recomputing history from operation logs in v1.
- Keep conflict UI read-only; resolution policy remains last-write-wins in the backend.
- Add frontend tests for row indicators and dialog banner rendering.
- Add backend tests for conflict listing and dismiss persistence.

### Acceptance Criteria

- A forced same-field concurrent edit on two machines produces a visible conflict on the losing machine.
- Different-field concurrent edits still merge silently with no conflict badge.
- Users can see which fields kept the winning value.
- Dismissed conflicts stay dismissed across restart unless the same entry conflicts again.

---

## 4. Shared Media Storage

### Problem

`picturePath` stores a local filesystem path. After sync, another machine often has metadata pointing at a path that does not exist on that computer.

### Goal

Store component pictures in the shared inventory tree so every machine can preview and open the same image when the shared workspace is available.

### Proposed Layout

```text
<shared root>\shared\inventory\
  manifest.json
  ops\
  snapshots\
  locks\
  backups\
  media\
    {entryUuid}\
      original.{ext}
      preview.jpg          # optional normalized preview generated on write
```

Example:

```text
S:\Engineering\Public\Syed_Hassaan_Shah\InventoryApps\TE\shared\inventory\media\8b1f3f7a-...\original.jpg
```

### Proposed Behavior

**On picture pick or save**

- When editing is enabled and shared storage is available, copy the selected file into `media/{entryUuid}/` using a stable name such as `original.{ext}`.
- Generate or refresh a cached preview file in that shared folder when the native preview pipeline runs.
- Persist the shared relative path or absolute shared path in `picturePath` consistently across clients.

**On preview/open**

- Resolve `picturePath` in this order:
  1. shared media path under `shared/inventory/media/{entryUuid}/`
  2. legacy local path on the current machine
- If neither exists, show the existing empty-preview state.

**Legacy entries**

- Keep supporting old local-only paths without forced migration.
- Optional later task: background migration tool that copies existing local pictures into shared media when the file is still reachable.

**Sync interaction**

- Picture copy happens on the editing machine before the entry mutation is flushed and published.
- Operation payloads continue to carry `picturePath`; remote machines resolve through the shared media folder rather than the writer's local disk.
- Read-only clients can load previews from shared media but never copy or replace files.

**Failure handling**

- If shared media is unavailable during save, fall back to current local-path behavior and surface a status message that the picture is local-only until shared storage is reachable.

### Implementation Notes

- Add shared media path helpers beside `backend/src/sync/shared_paths.rs`.
- Extend native picture preview/picker integration to read from shared media.
- Consider size limits and supported extensions (`jpg`, `jpeg`, `png`, `webp`, `gif`).
- Do not store binaries inside FeOxDB; only paths and shared files.
- Add smoke coverage for two-machine picture visibility.

### Acceptance Criteria

- Machine A assigns a picture to an entry; Machine B can preview it without re-picking the file.
- Legacy local-path entries still preview on the original machine.
- Read-only clients can view shared pictures but cannot replace them.
- Shared media lives only under `shared/inventory/media/`, not beside the installer at the TE root.

---

## Suggested Implementation Order

| Phase | Work | Why this order |
| --- | --- | --- |
| 1 | Read-only mode + edit prompt | Smallest user-risk win; clarifies which machines publish edits |
| 2 | Shared op compaction tuning + status | Protects S-drive soon; mostly constants, status, and smoke coverage |
| 3 | Conflict visibility | Builds on existing `SyncConflictRecord` storage; no sync transport change |
| 4 | Shared media | Depends on stable edit gating and clear status when shared storage is unavailable |

## Related Docs

- `README.md` — runtime behavior, shared layout, release checklist
- `docs/engineering/FEOXDB_SYNC_MIGRATION_PLAN.md` — operation-log sync architecture
- `docs/engineering/SYNC_RECOVERY_INVARIANTS.md` — local recovery rules
- `docs/engineering/PROJECT_FOLDER_BREAKDOWN.md` — file map
- `docs/engineering/AGENT_RUNBOOK.md` — agent validation and project traps