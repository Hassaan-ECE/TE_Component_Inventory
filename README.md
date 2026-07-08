# TE Lab Components Inventory

Last consolidated: 2026-06-09

TE Lab Components Inventory is a Windows desktop inventory app built with Tauri 2, React 19, TypeScript, Vite, Tailwind CSS v4, Bun, Rust, and FeOxDB.

Built by Syed Hassaan Shah.

This README is the current project entry point for setup, runtime behavior, shared storage, and release flow.

## Current Source Truth

- Active workspace: `C:\Projects\Active\Inventory_Apps\TE\TE_Parts_Inventory`
- App name: `TE Lab Components Inventory`
- Display name: `TE Lab Components Inventory v0.1.0`
- Version source: `package.json`, `backend\Cargo.toml`, and `backend\tauri.conf.json`
- Tauri identifier: `com.te.lab.components.inventory`
- Install mode: current-user NSIS install
- Updater: signed Tauri updater with GitHub Releases static metadata
- GitHub repo: `https://github.com/Hassaan-ECE/TE_Component_Inventory`
- Runtime database: local FeOxDB file named `inventory.feox`
- Local app-data path: `%APPDATA%\com.te.lab.components.inventory\inventory.feox`
- Shared drive root: `S:\Engineering\Public\Syed_Hassaan_Shah\InventoryApps\TE`
- Shared sync: S-drive FeOx operation logs plus manifest/snapshot bootstrap under `shared\inventory`

The TE app intentionally uses a different Tauri identifier, shared root, updater key, updater endpoint, HMAC environment variable, and frontend preference keys from the ME Inventory app. Do not point TE clients at the ME shared root or copy ME shared sync data into the TE root.

## Project Layout

```text
frontend/     React/Vite UI, frontend tests, UI assets, and Tauri bridge code
backend/      Tauri/Rust app, commands, storage, sync, export, and native helpers
docs/         Current engineering notes and performance baselines
scripts/      Smoke/manual automation scripts
```

Generated or machine-local folders are intentionally not source truth:

```text
node_modules/
frontend/dist/
backend/target/
backend/gen/
release/
.tmp/
coverage/
```

## Runtime Behavior

### Desktop App

- Tauri 2 shell with one main window.
- Current-user NSIS packaging.
- App data is stored under the Tauri app data folder for `com.te.lab.components.inventory`.
- Startup opens `inventory.feox` directly.
- Normal commands read and write FeOxDB only.

### Inventory UI

- Inventory and Archive views.
- Global search.
- Column filters for asset number, manufacturer, model, description, and location.
- Sorting and column visibility.
- Color Rows toggle.
- Theme persistence using `teComponentInventory.*` browser preference keys.
- Virtualized table rendering for larger result sets.
- Add, edit, verify, archive, restore, and delete flows.
- Full entry dialog with picture path support.
- Right-click context menu with open, saved-link, online-search, archive/restore, and delete actions.

### Entry Fields

The current `InventoryEntry` projection supports:

- `id`, `databaseId`, and `entryUuid`
- asset number and serial number
- quantity
- manufacturer, model, description, project, location, and assigned user
- links and notes
- lifecycle status
- working status
- condition
- verified state
- archived state
- manual-entry marker
- picture path
- created and updated timestamps

This is the same compatibility projection as the ME app, with TE-specific identity and storage. It fits the current TE lab components workflow without a schema fork.

### Excel Export

- `Export > Excel` uses a native save dialog.
- Default filename: `TE_Parts_Inventory_Export.xlsx`.
- Export includes all entries, not only the visible filtered rows.
- The workbook has exactly two sheets:
  - `Inventory` for active entries
  - `Archive` for archived entries

## Shared Sync

TE sync uses local FeOxDB on each machine plus shared operation logs, snapshots, and a manifest under the TE S-drive root. Clients do not mutate one shared FeOxDB file.

Shared root resolution:

- `TE_LAB_COMPONENTS_SHARED_ROOT`
- fallback: `S:\Engineering\Public\Syed_Hassaan_Shah\InventoryApps\TE`

Shared-drive layout:

```text
S:\Engineering\Public\Syed_Hassaan_Shah\InventoryApps\TE\
  TE Lab Components Inventory_0.1.0_x64-setup.exe
  release-support\
    v0.1.0\
      latest.json
      TE Lab Components Inventory_0.1.0_x64-setup.exe.sig
      SHA256SUMS.txt
  shared\
    inventory\
      manifest.json
      ops\
      snapshots\
      locks\
      backups\
      media\          # planned: shared entry pictures
        {entryUuid}\
```

Keep the shared root obvious for users: place only the current installer executable at the TE root, and put updater metadata, signatures, checksums, previous installers, and support material under `release-support\` or `archive\`.

Shared sync data belongs under `shared\inventory\`, not beside the installer.

Operation files are written under:

```text
<shared root>\shared\inventory\ops\{client_id}\000000000001.op.json
```

Snapshot files are written under:

```text
<shared root>\shared\inventory\snapshots\snapshot-*.snapshot.json
```

The latest snapshot is advertised by:

```text
<shared root>\shared\inventory\manifest.json
```

Optional HMAC hardening is controlled by `TE_COMPONENT_INVENTORY_SYNC_HMAC_KEY`. Set the same 16+ byte secret on every trusted TE client when S-drive write access alone is not enough.

## Signed Tauri Updater

The app uses the official signed Tauri updater. Update metadata is expected at:

```text
https://github.com/Hassaan-ECE/TE_Component_Inventory/releases/latest/download/latest.json
```

- `backend\tauri.conf.json` stores the updater public key and endpoint.
- The private signing key is generated outside the repo at `%USERPROFILE%\.tauri\te-component-inventory-updater.key`.
- The private key and password must never be committed.
- `bundle.createUpdaterArtifacts` is enabled so release builds produce updater artifacts and signatures.

The generated TE updater key currently has no password. Rotate it before broad distribution if release policy requires a password-protected private key.

## Setup

Install dependencies:

```powershell
bun install
```

Run the web UI:

```powershell
bun run dev
```

Run the Tauri desktop app:

```powershell
bun run dev:desktop
```

Build the frontend:

```powershell
bun run build
```

Run frontend tests:

```powershell
bun run test
```

Run the one-machine shared-sync smoke:

```powershell
powershell -ExecutionPolicy Bypass -File scripts\smoke-sync-one-machine.ps1
```

Build the Windows NSIS installer:

```powershell
bun run build:desktop
```

Installer output:

```powershell
backend\target\release\bundle\nsis\
```

## Release Checklist

Before building a release candidate:

```powershell
bun run lint
bun run test
bun run build
bun audit

Push-Location backend
cargo fmt -- --check
cargo check
cargo test
Pop-Location
```

`cargo audit` requires `cargo install cargo-audit`. Clippy is also a release gate once installed with `rustup component add clippy`:

```powershell
Push-Location backend
cargo clippy --all-targets -- -D warnings
cargo audit
Pop-Location
```

For signed updater releases, build with the updater private key available outside the repo:

```powershell
$env:PATH = "$env:USERPROFILE\.bun\bin;$env:PATH"
$env:TAURI_SIGNING_PRIVATE_KEY = (Get-Content -LiteralPath "$env:USERPROFILE\.tauri\te-component-inventory-updater.key" -Raw).Trim()
$env:TAURI_SIGNING_PRIVATE_KEY_PASSWORD = ""
bun run build:desktop
Remove-Item Env:\TAURI_SIGNING_PRIVATE_KEY
Remove-Item Env:\TAURI_SIGNING_PRIVATE_KEY_PASSWORD
```

Publish the generated NSIS installer, its `.sig` file, SHA-256 sums, and a GitHub Release asset named `latest.json`. Static updater metadata must include the Tauri updater fields for the Windows platform:

```json
{
  "version": "X.Y.Z",
  "notes": "Release notes",
  "pub_date": "YYYY-MM-DDTHH:MM:SSZ",
  "platforms": {
    "windows-x86_64": {
      "signature": "contents of the generated .sig file",
      "url": "https://github.com/Hassaan-ECE/TE_Component_Inventory/releases/download/vX.Y.Z/TE.Lab.Components.Inventory_X.Y.Z_x64-setup.exe"
    }
  }
}
```

Manual smoke for a new release:

- Confirm `package.json`, `backend\Cargo.toml`, and `backend\tauri.conf.json` versions match.
- Confirm the app identifier is `com.te.lab.components.inventory`.
- Confirm the visible name and version match the release.
- On clean app data, confirm startup hydrates from the TE S-drive snapshot and newer operation files.
- Add, edit, verify, archive, restore, and delete a disposable smoke entry.
- Save and open a safe `https://` link.
- Select a local picture path, confirm preview, then open it.
- Export Excel and confirm exactly `Inventory` and `Archive` sheets.
- Confirm GitHub Release assets and updater metadata.
- Run a real shared-drive multi-machine smoke and confirm create/update/delete convergence.
- Confirm the shared root has one obvious installer `.exe` at `S:\Engineering\Public\Syed_Hassaan_Shah\InventoryApps\TE\` and sync data under `shared\inventory\`.

## Bug-Fix Handoff

When debugging a future bug, capture the app version, Windows user, whether the installed app or dev app is running, and the local database path:

```powershell
whoami
Get-Item "$env:APPDATA\com.te.lab.components.inventory\inventory.feox" -ErrorAction SilentlyContinue
Get-ChildItem Env:TE_LAB_COMPONENTS_SHARED_ROOT,Env:TE_COMPONENT_INVENTORY_SYNC_HMAC_KEY -ErrorAction SilentlyContinue
Test-Path "S:\Engineering\Public\Syed_Hassaan_Shah\InventoryApps\TE\shared\inventory"
```

Expected production state:

- installed app version is `0.1.0` or newer
- TE shared inventory path returns `True`
- `TE_LAB_COMPONENTS_SHARED_ROOT` is unset unless intentionally testing a non-production root
- each user has their own local `%APPDATA%\com.te.lab.components.inventory\inventory.feox`

Only rename a user's local `inventory.feox` during stale-data testing after the app is closed and after taking note of the original timestamp/size. Normal fixes should preserve the local DB and let sync converge.

## Open Work

### Prioritized Improvements

Detailed specs live in `docs/engineering/PLANNED_IMPROVEMENTS.md`.

1. **Read-only mode by default** — each session starts in lookup-only mode unless the machine opted into **Always allow editing**. Any edit attempt shows a dialog: **Cancel**, **Enable editing for this session**, or **Always allow editing on this machine**. Read-only clients block mutations but still pull shared sync and can help compact old op files when they have no local outbox backlog.
2. **Shared operation compaction** — old op files are already deleted after verified snapshots, but current thresholds retain ops for up to 24 hours or 1,000 files. Tune for lab scale, expose op count in shared status, and use read-only sync clients as safe compaction publishers.
3. **Conflict visibility** — surface backend `SyncConflictRecord` data in the UI so users can see when another machine won an overlapping edit and which fields kept the winning value. Different-field merges stay silent.
4. **Shared media storage** — store pictures under `shared/inventory/media/{entryUuid}/` instead of machine-local paths only, so previews work on every client when the shared workspace is available.

### Release And Maintenance

- Run full two-machine create/update/archive/delete sync smoke before the first TE release.
- Decide whether TE needs custom fields beyond the current compatibility projection.
- Replace the cloned icon with a TE-specific icon if desired.
- Add locked-file smoke coverage for shared snapshot publish when needed.
- Benchmark real TE inventory size for search, sort, startup, sync, and table rendering.
