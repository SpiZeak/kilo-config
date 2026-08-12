---
description: Bootstrap a project - generate project AGENTS.md from detected stack
agent: code
---

Bootstrap the project at `$1` (default: current working directory) so future
agent sessions have accurate, project-scoped rules to follow.

## Phase 1 - Discover (parallel, no edits)

Issue all of these in a single message:

1. Check if the directory is a git repo (`!`ls -d .git``). If it is, run
   !`git ls-files | head -200` and !`git status && git log --oneline -10` to
   understand state; if not, list the top 2 directory levels instead and note
   "not a git repo" under `## Gotchas`.
2. Read whichever manifest files exist: `package.json`, `package-lock.json`
   (head), `pnpm-workspace.yaml`, `yarn.lock`, `bun.lock` (head), `deno.json`,
   `pyproject.toml`, `pdm.lock`, `requirements.txt`, `setup.py`, `Cargo.toml`,
   `rust-toolchain.toml`, `go.mod`, `composer.json`, `build.gradle`,
   `pom.xml`, `mix.exs`.
3. Read the formatter/linter config if present: `biome.json`, `.eslintrc*`,
   `prettier.config.*`, `.editorconfig`, `ruff.toml`, `.clippy.toml`,
   `golangci.yml`, `.rubocop.yml`.
4. Read toolchain pins: `.nvmrc`, `.tool-versions`, `Makefile` (a first-class
   manifest for Go and polyglot projects, per global AGENTS.md §4.4).
5. Read the test runner config: `vitest.config.*`, `jest.config.*`,
   `playwright.config.*`, `pytest.ini`, `conftest.py`.
6. Detect the e2e runner (if any): grep the source tree AND `.github/workflows/*`,
   `.gitlab-ci.yml`, `Makefile` for `playwright`, `cypress`, `puppeteer`,
   `webdriverio`, `@playwright/test`. Read its config.
7. Detect seed/fixture scripts: `db:seed`, `prisma/seed.ts`, `seeds/`,
   `factories/`, `fixtures/`, `mock-server`, `msw` handlers, faker
   factories. A "verified" UI change requires populated data — the
   absence of a seed is a project-shaped finding, not an oversight.
8. Read existing repo docs if present: `README.md`, `CONTRIBUTING.md`,
   `docs/index.md`, `ARCHITECTURE.md`.
9. Glob `**/AGENTS.md` to detect existing project-level rule files.

Use Grep for any symbol that looks load-bearing (the project name, the main
entry point, the deploy script).

## Phase 2 - Classify

From the manifests, derive:

- Stack (language, runtime, framework, DB, infra).
- Package manager.
- Commands: exact `scripts` entries that map to dev/build/test/typecheck/lint/format.
- Toolchain pins: node/python/rust/go versions from manifest fields or
  `.nvmrc` / `.tool-versions`.
- Test topology: unit vs integration vs e2e, which directories hold which,
  how to run a single test file.

## Phase 3 - Draft project AGENTS.md

Compose a project-level `AGENTS.md` at the repo root with these sections,
in this order, only when you have real content for them:

```
# <Project name>

One-paragraph summary lifted or distilled from README.md.

## Tech Stack
Bullet list: language, runtime, package manager, framework, key libs,
infra/deploy target.

## Commands
Markdown table copied verbatim from package.json / pyproject.toml /
Cargo.toml scripts/aliases. Include: dev, build, test, typecheck, lint,
format.

## Architecture
ASCII tree of the top 2 levels under `src/` (or equivalent), with one-line
annotations for non-obvious directories.

## Conventions
- Naming, file layout, import order: derived from existing neighbors.
- Error handling style: derive from 3+ existing call sites.
- Testing patterns: derive from existing test files.

## Env / Config
List ALL env vars referenced by the codebase (Grep for `process.env`,
`os.environ`, `std::env`, etc.). Note which have defaults, which are
required, and which are secrets. Confirm `.env.example` covers them; if
not, propose one but do not write it without asking.

## Gotchas
Anything you discovered that would burn future agents: platform-specific
behavior, build-time fetches, native deps, long install steps, slow tests,
required services.

## Verification Checklist
Exact commands a future agent MUST run before claiming done: typecheck,
lint, unit tests, build, and (when relevant) integration/e2e.

## E2E / UI Verification (if applicable)
Following the global AGENTS.md §14 rules:
- Run dev server: <command>
- E2E runner and config: <runner> / <config path>
- Seed/fixture command: <command> (or "none — write minimal seed first")
- Viewports required: desktop, mobile (and tablet if responsive)
- Browser tool: chrome-devtools MCP
- Screenshots saved under: `screenshots/e2e-<feature>-<viewport>.png`
```

If `AGENTS.md` already exists at the repo root, do NOT overwrite. Produce a
diff proposal only in your reply: list each section you would add or
rewrite, and the proposed text, then stop and let the user decide.

If subdirectories hold their own domain (e.g. `admin/`, `app/`, `workers/`,
`packages/*`), suggest one `AGENTS.md` per top-level subdirectory with only
the domain-specific sections it needs. Make proposals additive, never
destructive.

## Phase 4 - Verify the proposed commands

For each command listed under `## Commands` and `## Verification Checklist`:

- Run the typecheck command if it exists; report PASS/FAIL with the last
  20 lines of output.
- Run the lint command READ-ONLY (never `--write` or `--fix`); report
  PASS/FAIL.
- Run the unit test command in CI-equivalent mode; report PASS/FAIL and
  the test count.

This phase is best-effort, not a gate. Set a 5-minute timeout per command;
if a suite needs services/network or does not finish in time, stop, report
`skipped (slow | requires <service>)`, and record why under `## Gotchas`.
If any check fails in a way unrelated to your draft (e.g. pre-existing
flaky test), document it under `## Gotchas`.

## Phase 5 - Handoff

End your reply with this single short block — use `Wrote:` when a file was
created, `Proposed:` with a path list when the existing-file branch was
taken, and per-path `Stack`/`Commands`/`Verified` lines in monorepos:

```
Wrote (or Proposed): <path[, path...]> (new | updated | existing → diff)
Stack: <one line per project>
Commands: <list per project>
Verified: typecheck <pass/fail/skipped>, lint <pass/fail/skipped>, test <pass/fail/skipped>
E2E: <runner or "none"> | Seed: <command or "missing">
Still hand-curate: <one sentence on what needs human eyes>
```

Do not narrate the process. Do not offer to "do more". The user decides next.
