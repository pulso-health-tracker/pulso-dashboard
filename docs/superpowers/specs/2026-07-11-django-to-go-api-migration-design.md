# Django API → Go (`pulso-api`) Migration — Design Spec

## 1. Objective

Replace the Django backend in `pulso-dashboard` with a new standalone Go API repo (`pulso-api`), and reduce `pulso-dashboard` to a static frontend that talks to it over HTTP. This is the first of two sequential migrations toward the target stack (Go API + Next.js frontend) — the Next.js swap is a separate, later plan that will consume `pulso-api` once it exists.

## 2. Motivation & Sequencing

The user wants to move off Python/Django for the dashboard's backend, replacing it with Go, and separately wants to move the frontend from Vite+React to Next.js. The two are sequenced: Go API first, then Next.js — because the Next.js plan will build against a stable `pulso-api` rather than Django's soon-to-be-retired endpoints. This spec covers only the Go API migration and the minimal frontend changes needed to keep the current React app working against it in the interim.

## 3. Target Repositories

| Repo | Change |
|---|---|
| `pulso-health-tracker/pulso-api` (new) | Go + Echo + GORM, hosts the 3 metrics endpoints |
| `pulso-health-tracker/pulso-dashboard` (existing) | Django removed entirely; becomes a static Vite+React build served by nginx, calling `pulso-api` cross-origin |

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

## 5. `pulso-dashboard`

### 5.1 Removed

`dashboard_project/`, `apps/`, `manage.py`, `requirements.txt`, `conftest.py`, `pytest.ini`, the Django `Dockerfile`, `django-vite` (package.json dependency and `apps/analytics/templates/`).

### 5.2 Frontend Build Changes

- `vite.config.js`: drop `root: "frontend"` + manifest-based django-vite output (`outDir: "static"`, `manifest: true`). Standard Vite SPA build instead (`outDir: "dist"`), with a new `frontend/index.html` entry point (replacing the Django template `apps/analytics/templates/analytics/index.html`, which used `{% vite_asset %}` tags) — the `<div id="chart-root">` and a `<script type="module" src="/src/main.jsx">` move into this new static HTML file.
- New `frontend/src/config.js` exporting `API_BASE_URL` read from `import.meta.env.VITE_API_BASE_URL` (Vite build-time env var).
- Six existing relative-path `fetch` call sites are updated to prefix with `API_BASE_URL`:
  - `Dashboard.jsx`: 3 inline `fetch("/api/metrics/...")` calls (stat cards)
  - `useChartData.js`: the shared hook used by `EnergyChart`, `WorkoutVolumeChart`, `TopRecordTypesChart`

### 5.3 Docker & Compose

- New Dockerfile: `node:20-alpine` build stage (`npm ci && npm run build`) → `nginx:alpine` runtime stage serving `dist/`.
- New `docker-compose.yml`: single `dashboard` service (nginx), build arg `VITE_API_BASE_URL` pointing at `pulso-api`. No more `DB_*`, `SECRET_KEY`, `DJANGO_SETTINGS_MODULE`, `DEBUG` — the frontend no longer touches Postgres or Django settings at all.

### 5.4 CI

- `tests.yml`: unchanged — still just the `frontend-test` job (`npm ci && npm test`, vitest). The pre-existing gap (no backend test job) becomes moot since there's no backend left in this repo.
- `docker.yml`: updated to build the new nginx-based Dockerfile instead of the old Django one; drop the `docker-compose` validation job's Postgres/Django-specific assumptions (compose now only has one static-serving service).

### 5.5 README

Rewritten to describe the new architecture: static frontend, no local database of its own, depends on `pulso-api` running separately (with its own Postgres, provided by `pulso-etl`). Quick Start becomes: run `pulso-etl`'s compose, run `pulso-api`, then `docker compose up --build` here with `VITE_API_BASE_URL` pointing at it.

## 6. Local Dev Workflow (Three Repos Together)

1. `pulso-etl`'s compose provides Postgres (+ schema/data).
2. `pulso-api` connects to that Postgres (same `DB_*` env var convention already established) and serves the 3 endpoints on its own port (e.g. `:8080`).
3. `pulso-dashboard`'s compose builds with `VITE_API_BASE_URL=http://localhost:8080` and serves the SPA, which calls `pulso-api` cross-origin (CORS-enabled for the dashboard's origin).

## 7. Out of Scope

- **Next.js migration** — separate, later plan. Will replace `pulso-dashboard`'s Vite+React frontend with Next.js, consuming this same `pulso-api` unchanged.
- **AWS deployment architecture** — the existing AWS spec (`docs/superpowers/specs/2026-07-09-aws-native-architecture-design.md` in the former monorepo) assumed a single Django container on ECS; this migration invalidates that shape (now two independently-deployable pieces: a Go API and a static site). Revising that spec is not part of this plan — flagged here as a known follow-up.
- **Authentication** — no auth today, none added here.
- **Database schema changes** — `pulso-api` is strictly read-only against the existing schema; no migrations, no new tables.
- **SQL-side aggregation rewrite** — deferred; the mechanical port preserves today's in-application grouping behavior exactly.
