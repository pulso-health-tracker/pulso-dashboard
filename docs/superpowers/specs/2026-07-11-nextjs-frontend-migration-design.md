# Vite+React → Next.js (`pulso-web`) Migration — Design Spec

## 1. Objective

Stand up a new `pulso-web` repo (Next.js, App Router, TypeScript) that reproduces `pulso-dashboard`'s frontend — same 3 charts, same date-range interaction, same stat cards — as a **redesign**, not a mechanical port: data fetching moves from client-side `useEffect` calls to server-side rendering driven by URL search params. This repo is built in isolation and does not touch `pulso-dashboard`.

## 2. Motivation & Sequencing

This spec runs **concurrently** with `2026-07-11-django-to-go-api-migration-design.md` (Agent 1, building `pulso-api` in Go) as a separate, independent workstream (Agent 2):

- Agent 1 builds `pulso-api` from scratch, touching only that new repo.
- Agent 2 (this spec) builds `pulso-web` from scratch, touching only this new repo, developing against the **existing, still-running Django API** in `pulso-dashboard` — since both `pulso-api` and Django expose the identical `{labels, datasets, meta}` contract (guaranteed by Agent 1's spec §5 verification step), which backend `pulso-web` talks to during development makes no functional difference. Django is simply the one guaranteed to be running for the full duration, since Agent 1 might still be mid-build.

Neither agent touches `pulso-dashboard`. Once both `pulso-api` and `pulso-web` are independently verified, a small separate cutover step (see the Go spec's §6) repoints `pulso-web` at `pulso-api` and archives `pulso-dashboard`.

## 3. Why a Redesign, Not a Port

Unlike the ETL (Clojure→Python) and API (Django→Go) migrations, which used a mechanical 1:1 port to minimize risk, this migration embraces Next.js's core model instead of just relocating the existing client-fetch pattern into an App Router shell — a lift-and-shift would forfeit the actual reason to move frameworks. Concretely: today's `useChartData` hook (client-side `fetch` in `useEffect`, manual `loading`/`error` state) is replaced by a Server Component that fetches on the server and a URL-driven navigation model for date-range changes — see §5.

## 4. Target Repository

| Repo | Content |
|---|---|
| `pulso-health-tracker/pulso-web` (new) | Next.js (latest stable at implementation time), App Router, TypeScript |

## 5. Architecture

### 5.1 Data Flow

- `app/page.tsx` is a **Server Component**. It reads `start`/`end` from `searchParams` (falling back to the same defaults as today: 90 days for energy, 12 weeks for workouts/record-types — computed server-side at request time), and calls the 3 metrics endpoints directly via `fetch()` (server-to-server; the browser never talks to the API — no CORS needed for this repo's own requests).
- The 3 JSON responses are passed as props into a `<Dashboard>` component tree — no client-side data fetching anywhere.
- `DateRangeSelector` is a **Client Component** (`"use client"`). On change, it calls `router.push(\`/?start=${start}&end=${end}\`)` — this triggers a normal Next.js navigation, which re-runs `page.tsx` on the server with the new `searchParams` and re-renders with fresh data. No manual fetch/loading state management in this component; Next.js's navigation lifecycle handles it.
- `EnergyChart`, `WorkoutVolumeChart`, `TopRecordTypesChart` become **Client Components** (Chart.js requires a canvas/DOM), but only for rendering — they receive their dataset as a prop from the Server Component parent, dropping `useChartData` entirely.
- `StatCard`s' derived values (latest energy, this week's workout count, top record type) are computed server-side in `page.tsx` from the same 3 fetches, passed down as props — no separate client fetch for the stat cards (today's `Dashboard.jsx` has 3 duplicate inline fetches for this; the redesign fetches each endpoint exactly once).

### 5.2 Loading & Error States

- `app/loading.tsx` — Next.js's automatic Suspense boundary at the route level, shown during the server-side fetch on initial load and on every `searchParams` navigation. Replaces the manual `loading` boolean from `useChartData`.
- `app/error.tsx` — Next.js's route-level error boundary, shown if any of the 3 `fetch()` calls in `page.tsx` throws (network error, non-2xx status). Replaces the manual `error` string state from `useChartData`.

### 5.3 Component Reuse

`ChartCard`, `StatCard`, and the visual shell of `Dashboard` are presentational and carry over with minimal changes (props-driven already, no fetching logic in them today). `DateRangeSelector`, the 3 chart wrappers, and `Dashboard` itself are rewritten per §5.1. `useChartData.js` is deleted (superseded by server-side fetching).

## 6. Technology Decisions

| Concern | Choice |
|---|---|
| Framework | Next.js (latest stable at implementation time), App Router |
| Language | TypeScript |
| Charting | Chart.js + `react-chartjs-2` (unchanged from today, wrapped in Client Components) |
| Styling | Plain CSS, ported from today's `styles.css` (global stylesheet import in `app/layout.tsx`) — no framework switch (e.g. Tailwind) in scope |
| API config | `API_BASE_URL`, a server-only env var (no `NEXT_PUBLIC_` prefix — never sent to the browser, since only the Server Component fetches it) |
| Unit/component testing | Vitest + Testing Library, for the Client Components only (`DateRangeSelector`, the 3 chart wrappers, `ChartCard`, `StatCard`) — same tooling as `pulso-dashboard` today |
| End-to-end testing | Playwright, driving the full app (SSR page load, date-range navigation, chart rendering) against a real running instance of the API (Django during development, per §2) |
| Docker | Multi-stage: `node:20-alpine` builder (`next build` with `output: "standalone"`) → `node:20-alpine` runtime stage running the standalone server (`node server.js`) — SSR requires a Node runtime, not static hosting |
| CI | `tests.yml`: Vitest job (Client Components) + Playwright job (e2e, against Django's live API in the CI Postgres service — same `pulso_test` pattern already used by `pulso-etl`/`pulso-dashboard`). `docker.yml`: image build validation. |

## 7. Project Structure

```
pulso-web/
├── package.json
├── tsconfig.json
├── next.config.ts          # output: "standalone"
├── Dockerfile
├── docker-compose.yml
├── playwright.config.ts
├── app/
│   ├── layout.tsx           # imports global styles.css
│   ├── page.tsx             # Server Component: reads searchParams, fetches 3 endpoints
│   ├── loading.tsx
│   └── error.tsx
├── components/
│   ├── Dashboard.tsx         # presentational shell, receives all data as props
│   ├── ChartCard.tsx
│   ├── StatCard.tsx
│   ├── DateRangeSelector.tsx # "use client", navigates via router.push
│   ├── EnergyChart.tsx       # "use client", Chart.js, data via props
│   ├── WorkoutVolumeChart.tsx
│   └── TopRecordTypesChart.tsx
├── lib/
│   └── api.ts                # fetchEnergyVsGoal/fetchWorkoutVolume/fetchTopRecordTypes, called from page.tsx
├── tests/
│   ├── unit/                 # Vitest, one file per Client Component
│   └── e2e/                  # Playwright specs
└── styles.css
```

## 8. Local Dev Workflow

1. `pulso-etl`'s compose provides Postgres (+ schema/data).
2. `pulso-dashboard`'s Django server runs against that Postgres (unchanged, exactly as it works today).
3. `pulso-web` runs with `API_BASE_URL=http://localhost:8000` (Django's port) and calls it server-side on every request/navigation.

Once `pulso-api` is ready and verified (Agent 1's spec §5), swapping `API_BASE_URL` to point at it instead requires no code change in `pulso-web` — this is the cutover described in the Go spec's §6.

## 9. Out of Scope

- **Building `pulso-api`** — separate, concurrent spec (`2026-07-11-django-to-go-api-migration-design.md`). This spec only produces `pulso-web`.
- **Any changes to `pulso-dashboard`** — stays untouched until the cutover step in the Go spec's §6.
- **Authentication** — no auth today, none added here.
- **Styling redesign** — visual output matches today's dashboard; no new design system or component library.
- **Static export / edge deployment** — SSR via `searchParams` requires a Node server; static export is not viable given this architecture.
