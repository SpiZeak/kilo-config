---
name: ui-e2e
description: End-to-end verification loop for frontend changes in a real browser via Chrome DevTools MCP. Covers viewports, realistic data seeds, full interaction paths, console/network cleanliness, keyboard access, and slow-network states. Use when the task touches a new route, screen, modal, drawer, tab, component, form, list, search, drag/drop, layout, styling, data binding, or rendering dependency. Skip for pure backend logic, scripts, build tooling, docs.
---

# UI & E2E Verification

"I added the screen" is not "the screen works." Every UI change gets a real-browser loop before done — mocked-only does not count.

## Scope

Trigger for: new route/screen/modal/drawer/tab; new interactive component (forms, lists, search, drag/drop, real-time updates); visual change (layout, spacing, color, typography, theme); new data binding; new rendering dependency (animation/chart/icon lib, CSS framework).

Skip: pure backend logic, scripts, build tooling, docs.

## Definition of verified (all must hold)

1. **Viewports shipped**: desktop first; mobile if responsive (`chrome-devtools_emulate viewport=390x844x2,mobile`); tablet only if layout branches.
2. **Realistic data**: load a real seed — `db:seed`, factory script, fixture loader, mock-server payload, MSW handlers, or stub JSON. ≥10 records plus edge cases (long text, missing fields, zero state) and the new entity itself. Never fresh-empty DB.
3. **Interactions end-to-end**: every clickable visited; every form submitted once valid, once invalid; observe results — do not stop after the first render.
4. **Console clean**: `chrome-devtools_list_console_messages` filtered to `error`/`warn` must be empty for the flow; unhandled promise rejections count as failures.
5. **Network clean**: `chrome-devtools_list_network_requests` shows no unexpected 4xx/5xx.
6. **Keyboard accessible**: tab order logical with visible focus; `Escape` closes overlays; modals trap focus; `aria-*` matches visual role.
7. **Auth/roles**: if role-gated, at least one forbidden role sees the denial path, not the data.
8. **Network state**: for user-visible loading paths, emulate `Slow 3G` once — skeletons/timeouts render, not a frozen spinner.

## Procedure

1. **Seed**: grep for `seed|fixture|factory|mock-server|msw`. If none exists, write a minimal seed/stub JSON and ask for review before depending on it.
2. **Dev server**: start via `background_process`; capture the `Local:` URL. Probe the port first (AGENTS.md §5.3): reuse a healthy server, kill a stale one, never run two dev servers.
3. **Browser**: `chrome-devtools_new_page` with `isolatedContext` (no cookie/cache leakage); `take_snapshot` for the a11y baseline; resize/emulate to target viewports.
4. **Drive the flow**: `click`, `fill_form`, `type_text`, `press_key`, `hover`, chained through every interaction; verify console and network; screenshot at start, after major interactions, and at end — save to `/tmp/kilo/` (outside the worktree, never assume the worktree root is writable), named `e2e-<feature>-<viewport>.<ext>`.
5. **Re-run on mobile viewport** before claiming done. Any failure: fix, restart the dev server, restart the loop. No partial success.

## No e2e runner?

In order of preference: drive the dev server manually via Chrome DevTools MCP; Vitest + Testing Library + jsdom; pure unit tests (last resort). If no runner exists, surface it in the handoff: "Project has no e2e runner; UI was verified manually via Chrome DevTools MCP against the dev server. Suggest bootstrapping Playwright."
