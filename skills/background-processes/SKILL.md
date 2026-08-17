---
name: background-processes
description: Use when starting or managing long-running processes — dev servers, file watchers, build loops, or any command that does not exit — including port detection, busy-port handling, watcher accumulation, and cleanup.
---

# Background Processes

How to start, reuse, and stop long-running processes correctly. Load this
skill whenever a task needs a dev server, watcher, or build loop.

## 1. Routing long-running work

Dev servers, watchers, build loops, and any process that does not exit go
through `background_process` — never `nohup`, `&`, `setsid`, or `disown`.
Use `background_process` with a `ready` probe when possible.

## 2. Always check before starting

A second dev server on the same port does not start — it fails silently or
shadows the first one — and two servers on different ports waste CPU and
confuse every later human and agent. Before any `background_process` call:

1. **Detect the port the task would bind to.** Read it from the project's
   config (`vite.config.ts` `server.port`, `next.config.*`, `astro.config.*`,
   `tauri.conf.json` `devUrl`, `Cargo.toml` `[package.metadata]`,
   `Makefile` `PORT`). If unknown, fall back to the framework default.
2. **Probe in parallel** with anything else the task needs you to read:

   ```
   ss -tlnp 2>/dev/null | grep -E ':<port>\b' || lsof -nP -iTCP:<port> -sTCP:LISTEN 2>/dev/null || echo FREE
   ```

   `FREE` → safe to start. Anything else → the port is busy.
3. **If the port is busy, do NOT start a second server.** Decide which:
   - Process is yours and healthy → reuse it (navigate to its URL).
   - Process is yours but stale (zombie, wrong commit, stuck on a crash
     screen) → kill it first (`kill <pid>` or `pkill -f <pattern>`),
     confirm the port is free, then start one fresh `background_process`.
   - Process belongs to the user / a teammate → ask via `question`
     whether to attach, kill, or use a different port.

Run `pgrep -af 'vite|next dev|tauri dev|cargo run|astro dev|remix dev|wrangler dev|storybook'`
in parallel as a coarse sanity check — it catches dev servers on any
port that you forgot to look at.

## 3. Cache ports per project

After you learn a project's dev port from a real run, record it under
`## Commands` in the project `AGENTS.md` so future sessions skip the
guessing.

**Common ports to probe**: Vite/Remix/SvelteKit 5173 · Next/Nuxt/Rails/
Express 3000 · Astro 4321 · Storybook 6006 · Gatsby/Django/Flask/FastAPI
8000 · Cloudflare Workers 8787 · Tauri 1420 · Cargo/Go 8080. Extend per
stack.

## 4. Watchers and tail-modes

`vitest --watch`, `tsc --watch`, `npm run dev` in watch mode accumulate in
the same way. `pkill` the previous one before starting another, or —
preferred — reuse it and let it pick up the new file changes on its own
(most watchers do).

## 5. Stop what you started

When the task ends, terminate background processes you launched
(`background_process` stop) unless the user asked to keep them running.
