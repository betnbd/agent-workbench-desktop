# AGENTS.md

## Purpose

Linux desktop app for managing local Codex workspaces and long-running agent sessions. Each workspace runs its own `codex app-server` child process; the Rust backend brokers JSON-RPC over that process's stdio and streams events to a React UI. Linux-only; ships as a `.deb`.

## Stack

- Frontend: React 19 + TypeScript + Vite 7 (`src/`).
- Backend: Tauri 2 / Rust, edition 2021 (`src-tauri/`).
- Tests: Vitest (frontend, jsdom), `cargo test --lib` (backend).

## Build / Run / Test

```bash
npm install                 # install JS deps (also resolves Cargo deps on first tauri run)
npm run tauri dev           # run the full desktop app (devUrl http://localhost:1420)
npm run dev                 # browser-only UI shell; no Tauri commands, folder picking, or app-server
npm run build               # tsc + vite build (frontend only)
npm run build:deb           # tauri build -> .deb under src-tauri/target/release/bundle/deb/
npm run release:checksums   # writes SHA256SUMS beside the built .deb
npm test                    # vitest run (src/**/*.test.ts only)
npm run smoke:app-server    # spawn `codex app-server` in cwd and exercise JSON-RPC handshake
```

Rust checks (run from `src-tauri/`):

```bash
cargo test --lib
cargo clippy --lib -- -D warnings
cargo audit --quiet         # exits 0 but reports advisories in transitive Tauri/GTK deps
```

Host deps for a Tauri build on Debian/Ubuntu:

```bash
sudo apt install libwebkit2gtk-4.1-dev libgtk-3-dev libsoup-3.0-dev \
  libayatana-appindicator3-dev librsvg2-dev patchelf
```

## Layout

- `src-tauri/src/lib.rs` — all backend logic: Tauri command handlers, per-workspace `codex app-server` process lifecycle, JSON-RPC request/response routing, and `app-server-event` emission. Almost everything backend lives here.
- `src-tauri/src/main.rs` — thin entrypoint; delegates to `agent_workbench_lib::run`.
- `src-tauri/tauri.conf.json` — window config, CSP, `bundle.targets: deb`, before-dev/build commands.
- `src-tauri/capabilities/default.json` — Tauri permission allow-list (dialog/opener plugins).
- `src/services/tauri.ts` — single bridge between UI and backend; every `invoke(...)` call lives here and no-ops outside the Tauri runtime.
- `src/hooks/` — workspace/thread/model/event state (`useAppServerEvents`, `useWorkspaces`, `useThreads`, ...).
- `src/utils/redact.ts` — best-effort redaction applied to debug/health output before display or copy.
- `scripts/codex-app-server-smoke.mjs` — standalone JSON-RPC smoke test against a real `codex app-server`.
- `scripts/release-checksums.mjs` — SHA-256 over `.deb` artifacts in the deb bundle dir.

## Gotchas

- Requires a local `codex` CLI supporting `codex app-server` at runtime: found via `PATH` or a per-workspace absolute custom binary path. Tests/build do not need it; `smoke:app-server` does.
- `npm run dev` (and the static `index.html` preview) cannot reach the backend — `services/tauri.ts` short-circuits when not in the Tauri runtime, so workspace/folder/app-server features only work under `npm run tauri dev`.
- Vite uses a fixed `strictPort` on 1420; the build fails if that port is taken.
- Vitest `include` is `src/**/*.test.ts` only — `.tsx` test files are not picked up.
- Workspaces persist to `workspaces.json` in the OS app-data dir (Tauri `app_data_dir()`); the backend deserializes camelCase setting keys (e.g. `defaultAccessMode`) via serde renames, so keep `WorkspaceSettings` field renames in sync with the TS `types.ts`.
- Each workspace spawns its own `codex app-server` with the workspace folder as the child cwd; threads are matched to a workspace by comparing `cwd`.
