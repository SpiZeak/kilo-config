---
name: verification
description: Use when verifying a change before reporting done — running the project's tests, lint, typecheck, or builds, or when the project's check commands are unknown. Covers the manifest-detection recipe, narrowest-check-first ordering, and failure widening.
---

# Verification

A change is not done until it passes the project's own checks. This skill
holds the detection recipe for finding and running those checks. Do not
hardcode commands or guess — read what actually exists.

## 1. Read the manifest in parallel

Pick whichever exist:

- Node/JS: `package.json` `scripts`; respect `packageManager` field.
- Python: `pyproject.toml` `[project.optional-dependencies]`,
  `[tool.poetry.scripts]`, `[tool.ruff]`, `[tool.mypy]`, `[tool.pytest]`.
- Rust: `Cargo.toml`; respect `[workspace.metadata]` and `rust-toolchain.toml`.
- Go: `go.mod` and `Makefile` (most Go projects put commands there).
- Polyglot: read **all** manifests that exist.

## 2. Map scripts to verbs

For each manifest, label which entries are `dev`, `build`, `test`,
`typecheck`, `lint`, `format`. If a verb has no script, mark it "none" —
do not invent one.

## 3. Run the narrowest check first

Pick the check that touches the changed code:

- Single-file change → lint + typecheck of that file.
- Library/API change → tests of callers + the lib's own tests.
- Schema/migration change → tests + a dry-run of migration up/down.
- Frontend change → typecheck + `npm run build` (production build is the
  ground truth, not `dev`) + the `ui-e2e` skill loop.

## 4. Widen only on failure

If lint passes but tests fail, fix tests before claiming done; do not
declare partial success.

## 5. Cache the recipe

Once known, record the recipe in project `AGENTS.md` under
`## Verification Checklist` so future sessions skip discovery. If a
required command is genuinely unknown after the recipe, ask the user for
the command and offer to record it in project `AGENTS.md` next time.
