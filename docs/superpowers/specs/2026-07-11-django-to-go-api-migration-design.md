# Django API → Go (`pulso-api`) Migration — Design Spec

## 1. Objective

Stand up a new standalone Go API repo (`pulso-api`) that reproduces the 3 metrics endpoints Django currently serves, as a **contract-compatible drop-in replacement**. This repo is created in isolation — it does not touch `pulso-dashboard` (Django+Vite) at all.

## 2. Motivation & Sequencing

The user wants to move off Python/Django for the dashboard's backend (Go) and, separately, off Vite+React for the frontend (Next.js). These two migrations now run **concurrently**, each as an independent agent/workstream, to avoid one waiting on the other:

- **Agent 1 (this spec):** builds `pulso-api` from scratch. Touches only the new repo.
- **Agent 2** (separate spec, `2026-07-11-nextjs-frontend-migration-design.md`): builds a new `pulso-web` (Next.js) repo from scratch, developing against the *existing, still-running* Django API (same contract `pulso-api` will expose) so it has something real to call without waiting on Agent 1.

Neither agent modifies `pulso-dashboard` — it keeps running exactly as-is (Django + Vite/React, own Postgres connection) as the live reference implementation and the thing both new repos are measured against, until both `pulso-api` and `pulso-web` are done and verified. Only then is `pulso-dashboard` retired (archived, same treatment as the original monorepo).

## 3. Target Repositories

| Repo | Change |
|---|---|
| `pulso-health-tracker/pulso-api` (new) | Go + Echo + GORM, hosts the 3 metrics endpoints |
| `pulso-health-tracker/pulso-dashboard` (existing) | **Untouched by this spec.** Stays as Django + Vite/React, fully functional, until both `pulso-api` and `pulso-web` are ready — see §7. |

No changes to `pulso-health-tracker/pulso-etl` — it remains the sole owner of the Postgres schema and migrations. Both `pulso-api` and the ETL read/write the same database; `pulso-api` is read-only.

## 4. `pulso-api`

### 4.1 Technology Decisions

| Concern | Choice |
|---|---|
| HTTP framework | Echo |
| DB access | GORM (ORM, chosen over raw SQL/pgx for familiarity coming from Django's ORM) |
| Migration philosophy | **Mechanical 1:1 port** — same in-application aggregation logic as the Python repository layer (fetch rows, group in memory), not a SQL-side rewrite. Consistent with the ETL Clojure→Python migration's approach: lower risk, directly traceable to the existing behavior. |
| Testing | Go `testing` + `testify`, integration tests against a real `pulso_test` Postgres database, Given/When/Then structure (matching the project's existing test style) |
| CORS | Echo CORS middleware, allowed origin configurable via `CORS_ALLOWED_ORIGIN` env var (default `http://localhost:5173` for local dev against the Vite dev server) |
| Config | Env vars, same names as today: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` |
| Docker | Multi-stage: `golang:1.24-alpine` builder stage (`CGO_ENABLED=0 go build`) + `alpine:latest` runtime stage running the static binary — consistent with the `-alpine` base images already used elsewhere in the project (`postgres:17-alpine`, `nginx:alpine`) |
| CI | Full CI from the first commit: `tests.yml` (Postgres service container, `go test ./...`) and `docker.yml` (image build validation) — no historical gap to inherit since this is a new repo |

### 4.2 Data Models (GORM)

Four GORM structs map to the existing tables the ETL owns. All use explicit `TableName()` methods pointing at `public.*`, and GORM's `AutoMigrate` is never called — schema ownership stays with `pulso-etl`'s migration runner, mirroring today's Django `managed = False`:

- `ActivitySummary` → `public.activity_summary` (`DateComponents`, `ActiveEnergyBurned`, `ActiveEnergyBurnedGoal`)
- `Workout` → `public.workout` (`ActivityType`, `Duration`, `TotalEnergyBurned`, `StartDate`, `EndDate`)
- `Record` → `public.record` (`RecordTypeID` FK, `StartDate`)
- `RecordType` → `public.record_type` (`Identifier`)

### 4.3 Endpoints

Same 3 routes, same query params (`start`, `end`, `YYYY-MM-DD`, both optional), same JSON response contract (`{labels, datasets, meta}`), same 400 response shape (`{"error": "..."}`) on invalid date format, ported line-for-line from `apps/analytics/repositories.py` and `apps/analytics/views.py`:

- `GET /api/metrics/energy-vs-goal` — default window 90 days, groups `activity_summary` by day
- `GET /api/metrics/workout-volume` — default window 12 weeks, groups `workout` by ISO week (Monday-keyed), sums duration (minutes) and energy
- `GET /api/metrics/top-record-types` — default window 12 weeks, top 5 `record_type` by volume, weekly counts per type

No `index`/HTML route — `pulso-api` is API-only, no template rendering.

## 5. Local Dev Workflow (Verifying `pulso-api` Standalone)

1. `pulso-etl`'s compose provides Postgres (+ schema/data).
2. `pulso-api` connects to that Postgres (same `DB_*` env var convention already established) and serves the 3 endpoints on its own port (e.g. `:8080`).
3. Verification: hit `pulso-api`'s 3 endpoints directly (`curl`) and diff the JSON against the same request made to the still-running `pulso-dashboard` Django API, for the same date range — the two must return byte-identical JSON (modulo key order). This is the acceptance check for "contract-compatible," not just "returns some JSON."

## 6. Cutover — When Both Parallel Migrations Are Done

Once `pulso-api` (this spec) and `pulso-web` (the Next.js spec) are both built and independently verified:

1. Point `pulso-web` at `pulso-api` instead of the Django API it was developed against (a config/env var change only — the contract is identical, per §5's verification step).
2. Confirm `pulso-web` + `pulso-api` together reproduce what `pulso-dashboard` does today (same manual check as the two-repo verification done for `pulso-etl` + `pulso-dashboard` earlier in this project).
3. Archive `pulso-dashboard` (Django + Vite/React), same treatment as the original monorepo: README replaced with a pointer notice to `pulso-api` and `pulso-web`, repo archived.

This step is out of scope for both individual specs — it's a separate, small integration task once both land.

## 7. Out of Scope

- **Building `pulso-web` (Next.js)** — separate, concurrent spec (`2026-07-11-nextjs-frontend-migration-design.md`). This spec only produces `pulso-api`.
- **Any changes to `pulso-dashboard`** — stays untouched until §6's cutover.
- **AWS deployment architecture** — the existing AWS spec (`docs/superpowers/specs/2026-07-09-aws-native-architecture-design.md` in the former monorepo) assumed a single Django container on ECS; once `pulso-api` + `pulso-web` replace it, that shape is invalidated (now two independently-deployable pieces: a Go API and a Next.js app). Revising that spec is not part of this plan — flagged here as a known follow-up.
- **Authentication** — no auth today, none added here.
- **Database schema changes** — `pulso-api` is strictly read-only against the existing schema; no migrations, no new tables.
- **SQL-side aggregation rewrite** — deferred; the mechanical port preserves today's in-application grouping behavior exactly.
