---
name: production-hygiene
description: Production-readiness rules that prevent bugs slipping past local review into staging or production. Covers schema/contract regeneration, caller sweeps, env-var coverage, feature-flag parity, rollback plans, observability, auth negative tests, clock/RNG injection, error layering, backwards compatibility, idempotency. Use when touching DB schemas, API contracts, protos, message formats, env vars, public signatures, deploy/migrations/secrets, auth/tenancy, cron/webhooks/queues, or observability.
---

# Production Hygiene

Patterns that prevent the kinds of bugs that only surface in staging or production:

- **Schema-first**: when changing a DB schema, API contract, proto, or message format, regenerate derived bindings (types, clients, mocks, fixtures) in the same commit. Never hand-update one without the other.
- **Caller sweep**: when changing a public signature, grep all call sites across the repo AND referenced packages/workers; fix them in the same diff; document breaking changes.
- **Env-var coverage**: a new env var is recorded in `.env.example`, README, and any cloud-secret manifest. Never let required config live only in your head.
- **Feature-flag parity**: both branches of a gated feature must build, type-check, and have ≥1 test each; flag removal is a separate PR.
- **Rollback plan**: changes touching deploy, migrations, secrets, infra, or scheduling end with `## Rollback` in the commit description — the exact commands to undo safely.
- **Idempotency**: migrations, webhooks, retryable jobs, and side-effecting endpoints must be safely re-runnable; test the re-run case.
- **Observability**: a new service path, cron, queue, or webhook emits metrics/logs/traces; if not, surface it as a follow-up — do not ship invisible code.
- **Auth & tenancy**: changes touching auth, sessions, multi-tenant data, RLS, RBAC, or CORS get ≥1 negative test — the forbidden case must fail loudly.
- **Time & clock**: never trust `Date.now()` / `time.time()` for business logic; use the project's clock abstraction (clock injection, freezegun, tokio::time::pause). Same for random IDs — use the project's seeded RNG or library.
- **Error layering**: a boundary `catch`/rescue/`?` logs once with full context, then returns a typed/structured error. No silent swallowing, no double logging in nested catches.
- **Backwards compatibility**: deleting a public function, exported type, or REST/GraphQL field is breaking — deprecate first, remove in a later release, ship a CHANGELOG entry.
