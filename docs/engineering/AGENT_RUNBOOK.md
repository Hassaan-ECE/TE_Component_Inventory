# Agent Runbook

Last updated: 2026-06-09

Use this file before release, cleanup, or bug-fix work in this repo. It records project-specific traps and repeatable validation commands. Current release state lives in `README.md`.

## Agent Rules

- Run `git status --short` before editing.
- Treat `README.md` as the current entry point.
- Preserve user-authored changes. Do not revert files outside your assigned scope.
- Keep importing files and extracted folders together. If `InventoryShell.tsx` imports `frontend/src/features/inventory/components/shell/*`, that folder must be included when staging or committing.
- Do not copy generated ME release artifacts, ME shared sync data, ME local `inventory.feox`, or ME archive evidence into this repo.
- The manager owns checklist updates, integration, validation, and final review.

## TE Identity

- Workspace: `C:\Projects\Active\Inventory_Apps\TE\TE_Parts_Inventory`
- GitHub repo: `https://github.com/Hassaan-ECE/TE_Component_Inventory`
- Product name: `TE Lab Components Inventory`
- Tauri identifier: `com.te.lab.components.inventory`
- Shared root: `S:\Engineering\Public\Syed_Hassaan_Shah\InventoryApps\TE`
- Shared-root override env var: `TE_LAB_COMPONENTS_SHARED_ROOT`
- Sync HMAC env var: `TE_COMPONENT_INVENTORY_SYNC_HMAC_KEY`
- Updater key path: `%USERPROFILE%\.tauri\te-component-inventory-updater.key`

## Known Hiccups And Pivots

### Bun-Only JavaScript Runtime

- This workspace is maintained with Bun as the JavaScript runtime.
- Do not require Node for normal project commands.
- Use Bun directly from the repo root:

```powershell
bun install
bun run <script>
```

- Package scripts intentionally call direct `bun node_modules/...` tool entry points for Vite, ESLint, TypeScript, and Vitest. On the current Windows/Bun setup, the package-bin shim path can crash Bun before the tool starts.

Current machine version verified with these scripts: `1.3.14`.

### NSIS User-Mapped Section Lock

- Symptom: `bun tauri build --bundles nsis` compiles the release executable, then fails during the NSIS bundle step with `The requested operation cannot be performed on a file with a user-mapped section open. (os error 1224)`.
- Pivot: confirm no app/build processes are running, remove only the generated installer for the target version under `backend\target\release\bundle\nsis\`, then rerun the bundle command.
- Treat the installer from a failed bundle command as untrusted even if an `.exe` file exists.

### Signed Tauri Updater Key

- Public updater config lives in `backend\tauri.conf.json`.
- Private updater signing key lives outside the repo at `%USERPROFILE%\.tauri\te-component-inventory-updater.key`.
- Do not commit private key material, passwords, generated `.sig` files, release bundles, or `latest.json` drafts unless the user explicitly asks for release asset staging.
- The current key was generated without a password. Rotate it before broad distribution if policy requires password-protected signing keys.
- For the current Tauri CLI, signing expects `TAURI_SIGNING_PRIVATE_KEY` to contain the private key text. Because the current key is unpassworded, set `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` to an explicit empty string for non-interactive builds.

### GitHub Release Upload

- Stage installer, `.sig`, SHA-256 sums, and `latest.json` locally under an ignored `release\vX.Y.Z\` folder.
- Upload those assets to a non-draft, non-prerelease GitHub Release tagged `vX.Y.Z`.
- Validate the app updater against the real GitHub metadata URL:

```text
https://github.com/Hassaan-ECE/TE_Component_Inventory/releases/latest/download/latest.json
```

### Shared Drive Staging

- Current shared root: `S:\Engineering\Public\Syed_Hassaan_Shah\InventoryApps\TE`.
- Keep the shared root obvious for installers: put only the current NSIS installer `.exe` at the root.
- Put updater metadata, `.sig` files, SHA-256 sums, previous installers, and other support material under folders such as `release-support\vX.Y.Z\` or `archive\`.
- Shared sync state belongs under `shared\inventory\`, with manifest, operation logs, snapshots, locks, and backups below that folder.
- The app fallback shared root is hardcoded in `backend/src/sync/types.rs`; update that constant if the shared root moves.

## Validation Commands

### Frontend

```powershell
bun run lint
bun run test
bun run build
bun audit
```

### Git

```powershell
git diff --check
git status --short
```

### Rust

```powershell
Push-Location backend
cargo fmt -- --check
cargo check
cargo test
cargo clippy --all-targets -- -D warnings
cargo audit
Pop-Location
```

`cargo clippy` and `cargo audit` require local tooling to be installed before they can be treated as release gates.
