---
name: git-discipline
description: Use when performing git operations — inspecting status/diff/log, branching, committing, pushing, opening PRs, rebasing, merging, resolving conflicts, stashes, dependency/lockfile churn, or any commit/build hygiene question. Load this skill whenever a task touches a git repo so the git workflow rules apply.
---

# Git Discipline

These rules apply to every git operation. They were moved out of the global
`AGENTS.md` so they only load when git work is happening. Load this skill
whenever a task involves a git repo.

## 1. Inspect before changing

Run `git status`, `git diff`, `git log --oneline -10` before any edit so
you know the current state, the size of your own pending changes, and
the recent commit style.

## 2. Stage only what the user named

Never use `git add -A` or `git add .` unless explicitly asked. Prefer
explicit paths: `git add src/foo.ts tests/foo.test.ts`.

## 3. Match the repo's commit message style

Conventional Commits with scope prefixes, plain imperative, gitmoji,
etc. If unclear, mirror the most recent 10 messages.

## 4. Do not rewrite history without an explicit ask

No `git commit --amend`, `git push --force`, or skipping hooks on shared
or protected branches. Rebasing your own unmerged feature branch is
expected (§11); rewriting anything others build on is not. Once a
commit is on a shared branch it is someone else's problem to change.

## 5. Before opening a PR

Verify remote tracking, base branch, and that the pushed diff matches
the local diff. Re-read your own full diff before opening the PR and call
out the riskiest hunks in the description. Cite the PR URL in the handoff
once opened.

## 6. Dependency & lockfile discipline

- Adding a dependency is a **lockfile + install + retest** operation.
  Always run install and re-run the verification checklist (§4.4 of
  AGENTS.md) before claiming done.
- Pin versions deliberately: prefer the form the repo already uses
  (`^x.y.z` vs exact `x.y.z`). If unsure, mirror the dominant style.
- After any `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod`
  edit, diff the lockfile and summarize the changes (added X, removed
  Y, bumped Z major) in your handoff.
- Never edit a lockfile by hand. If the registry resolved wrong, change
  the manifest and let the package manager regenerate.
- Upgrades that change a **major** version require an explicit user
  sign-off and a migration note in commit/PR description.

## 7. Branch naming

Always use `feature/` with a slash for feature branch names.

## 8. Verify before you commit

Run the project's checks (§4.4 of AGENTS.md) and re-grep the staged diff
against the §6.2 done-state ledger (secrets, `console.log`, debuggers,
TODOs) *before* `git commit`, not after. Commit first, verify later is the
fastest way to ship a broken or leaking commit.

## 9. Atomic commits

One logical change per commit, explicitly staged via `git add <paths>`.
Each commit should be independently revertible and must not bundle
unrelated refactors, dependency bumps, or formatting churn.

## 10. Never commit artifacts or env files

Respect `.gitignore`. Before staging, confirm no `dist/`, `node_modules/`,
binaries, or `.env*` files leak into the diff. `.env` is the most common
secret vector — treat any staged dotenv as a blocker.

## 11. Fetch before branching or rebasing

`git fetch` first, branch feature branches off up-to-date `main`, and keep
them rebased so conflicts are resolved locally before the PR — never
discovered first in CI.

## 12. Check commit identity

Verify `user.name`/`user.email` match the repo's expectations before
committing. A misconfigured or borrowed identity silently pollutes
history and cannot be cleaned up cleanly afterward.

## 13. Read CONTRIBUTING.md / PR template

Before creating a PR, honor the repo's merge strategy (squash vs merge
commits), required checks, GPG signing, and issue-linking conventions. Do
not guess from the commit log alone. Add a `CHANGELOG.md` entry when the
repo maintains one.

## 14. Clean up after merge

Delete merged local branches (`git branch -d`) and update local `main` so
the next branch starts from a fresh base.

## 15. Never commit WIP or broken builds

If work is not shippable, commit it to a WIP branch instead of leaving
broken state on a feature branch others may pull. A commit should be code
you stand behind.

## 16. Never commit directly to the default branch

When the repo has a PR workflow, land changes via a feature branch and a
PR. Committing straight to `main`/`master` bypasses review and breaks CI
expectations.

## 17. Merge-conflict discipline

Never resolve conflicts with a blind ours/theirs sweep. Read each hunk,
take the semantically correct side, and re-run the project's tests after
resolving. If a hunk is unclear, `git merge --abort` (or `git rebase
--abort`) and ask — a wrong resolution is worse than a paused one.

## 18. No stash litter or detached HEAD

Do not leave behind unnamed stashes, orphaned scratch branches, or a
detached HEAD. Name stashes, pop them before finishing, and leave the
worktree on a real branch.
