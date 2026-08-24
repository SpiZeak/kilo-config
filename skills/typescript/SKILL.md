---
name: typescript
description: Use when working on TypeScript/JavaScript projects — Bun-first package management, lockfile-aware installs, scripts-to-verb mapping, monorepo workspace filters, typecheck/lint/format read-only invocations, single-test runs, module-system pitfalls, and framework detection (Next/Vite/Remix/Astro/Nuxt/SvelteKit).
---

# TypeScript / JavaScript

## 0. Package manager policy (hard rule)

**Bun is the default.** Every command goes through Bun unless a foreign
lockfile plus an explicit pin says otherwise.

Two tiers:

1. **Runner — always Bun, no exceptions.** `bun run <script>` and
   `bunx <bin>` are lockfile-neutral (they execute node_modules/.bin), so
   they are correct even in repos authored with npm/pnpm/yarn. Never reach
   for `npm run` or `npx`.
2. **Installer — Bun unless the repo is foreign-pinned.** `bun install` by
   default. Only gate when ALL of: a non-Bun lockfile exists
   (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`), AND a pin enforces
   it (`packageManager` field, CI lockfile-integrity step, or project
   AGENTS.md). Then run that PM's install (`npm ci`, `pnpm install
   --frozen-lockfile`) and record "repo pinned to <PM>" under gotchas.
   Migrating the repo to Bun is a user decision, never a side effect.

### Lockfile reference

| File                     | Meaning                              |
| ------------------------ | ------------------------------------ |
| `bun.lock`               | Bun ≥1.2, text — the good case       |
| `bun.lockb`              | Legacy Bun binary lockfile           |
| `package-lock.json` etc. | Foreign; installer gate may apply     |
| none                     | Greenfield: `bun install`, commit `bun.lock` |

`bun install --frozen-lockfile` is the CI-safe equivalent of `npm ci`.

### Bun footguns

- `bun test` runs Bun's NATIVE test runner (bun:test) — it does NOT run the
  package's `test` script. Package scripts always go through
  `bun run test`. Likewise prefer `bunx vitest run <file>` over any bare
  `test` verb when the repo uses vitest/jest.
- Bare `bunx vitest` / watchers hang non-interactive sessions — always the
  `run` form.
- PM-specific constructs inside scripts (`pnpm --filter`, `npm exec`,
  `postinstall` shelling to a PM) still execute as written; don't rewrite
  them drive-by.

## 1. Scripts → verbs (with fallbacks)

Read `package.json` `scripts` and map to verbs. When a verb is missing:

| Verb          | Common script names | Read-only fallback (via Bun)                    |
| ------------- | ------------------- | ----------------------------------------------- |
| typecheck     | `typecheck`         | `bunx tsc --noEmit` (`tsc -b` if references)    |
| lint          | `lint`              | `bunx eslint .` or `bunx biome check`           |
| format check  | `format:check`      | `bunx prettier --check .` / `biome format`      |
| test (single) | `test`              | `bunx vitest run <file>` / `bunx jest <file>`   |

Verification is read-only: never `--write`, `--fix`, or `--format` when
checking. Formatting whole directories requires the global §6.1 pre-flight
block.

## 2. Toolchain pins

Check `.nvmrc`, `.tool-versions`, `mise.toml`, `volta` in `package.json`,
and `engines`. Bun's runtime is only used when the repo already targets it
(`bunfig.toml`, Bun in scripts/CI) — otherwise Bun is the vehicle, Node is
the runtime.

## 3. Monorepos

Detect: `workspaces` in root `package.json` (Bun-native), `turbo.json`,
`nx.json`, `pnpm-workspace.yaml`, `lerna.json`.

- Bun workspaces: `bun run --filter=<pkg> <script>` (Bun ≥1.2); installs
  stay at root and cover all workspaces.
- turbo/nx repos keep their orchestrator (`turbo run <task>
  --filter=<pkg>`), invoked through `bun run` where a wrapper exists.
- A foreign `pnpm-workspace.yaml` triggers the §0 installer gate.
- Root `package.json` often has no source — real scripts live in
  `packages/*/package.json`.

## 4. Framework detection

`next.config.*`, `vite.config.*`, `remix.config.*`, `astro.config.*`,
`nuxt.config.*`, `svelte.config.*`. Affects dev-server port for probing
(Next/Remix/Nuxt 3000, Vite/SvelteKit 5173, Astro 4321) and build commands.
For Next: `src/app` (App Router, RSC constraints) vs `src/pages`. For any UI
change, the `ui-e2e` skill + chrome-devtools MCP govern browser verification.

## 5. Module system

`"type": "module"` → ESM defaults; `.cjs` is the escape hatch. In ESM:
`import`, no `require`; `__dirname` only via
`fileURLToPath(import.meta.url)`. Respect `tsconfig.json`
`moduleResolution` (`bundler` vs `node`). Match the import style of
unchanged neighbors before writing new ones.

## 6. Testing

Detect runner from `vitest.config.*` / `jest.config.*` / `node:test`.
Single affected file first, then the package suite (§0 footgun: via
`bunx vitest run` / `bun run test`, never bare `bun test` unless the repo
actually uses bun:test). `.only`/`.skip` left in test files is a done-state
violation (global §6.2). Skip coverage runs unless asked.

## 7. Gotchas

- `tsc -b` writes `.tsbuildinfo` — build output, never commit.
- Env vars: `NEXT_PUBLIC_*` / `VITE_*` are public by design; every other
  `process.env` reference is a potential secret — enumerate, never echo.
- Two lockfiles in one repo means someone broke §0 — surface it in the
  report; fixing it is the user's call.
