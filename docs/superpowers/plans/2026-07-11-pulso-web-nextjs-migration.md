# pulso-web (Next.js) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and publish `pulso-health-tracker/pulso-web`, a Next.js (App Router, TypeScript) redesign of `pulso-dashboard`'s frontend — same 3 charts, same date-range interaction, same stat cards — with data fetching moved from client-side `useEffect` to server-side rendering driven by URL search params.

**Architecture:** `app/page.tsx` is a Server Component that reads `start`/`end` from `searchParams`, fetches all 3 metrics endpoints server-side, and passes the data down as props. `DateRangeSelector` is a Client Component that only navigates (`router.push`) — it never fetches. The 3 chart components are Client Components (Chart.js needs a canvas) that render data received via props, with no fetching logic of their own.

**Tech Stack:** Next.js (latest stable), App Router, TypeScript, Chart.js + `react-chartjs-2`, Vitest + Testing Library (Client Components), Playwright (end-to-end).

## Global Constraints

- Repo: `pulso-health-tracker/pulso-web`, public, created fresh (not extracted — greenfield).
- This plan touches **only** the new `pulso-web` repo. Never edit anything under `/home/yagoazedias/github/pulso-dashboard`, `/home/yagoazedias/github/pulso-etl`, or `/home/yagoazedias/github/pulso-api` (or `/tmp/pulso-api` if that plan hasn't been published yet).
- This is a **redesign**, not a mechanical port: no client-side `fetch`/`useEffect` for data — see Architecture above. `ChartCard`, `StatCard` carry over near-verbatim (already presentational); `DateRangeSelector`, the 3 chart components, and `Dashboard` are rewritten per the new data flow.
- During development, `pulso-web` talks to the **existing Django API** in `pulso-dashboard` (still running, unmodified) via `API_BASE_URL` — not `pulso-api`, which may not exist yet (parallel workstream). Swapping to `pulso-api` later is a config-only change, out of scope for this plan.
- `API_BASE_URL` is a **server-only** env var (no `NEXT_PUBLIC_` prefix) — it must never reach the browser bundle, since only the Server Component fetches it.
- Styling: plain CSS, ported verbatim from `pulso-dashboard/frontend/src/styles.css`. No Tailwind, no CSS-in-JS.
- Charting: Chart.js + `react-chartjs-2`, unchanged from `pulso-dashboard`. No library swap.

---

### Task 1: Preflight

**Files:** none (verification only).

- [ ] **Step 1: Verify Node toolchain**

Run:
```bash
node --version
npm --version
```
Expected: Node 20+ (matches `pulso-dashboard`'s CI, which uses Node 20).

- [ ] **Step 2: Verify GitHub org access**

Run:
```bash
gh api orgs/pulso-health-tracker --jq '.login'
```
Expected: `pulso-health-tracker`.

- [ ] **Step 3: Confirm the reference implementation is available to port from**

Run:
```bash
test -d /home/yagoazedias/github/pulso-dashboard/frontend/src/components && echo "found"
```
Expected: `found`. This directory is the source of truth for Tasks 5–8 — do not modify it.

---

### Task 2: Scaffold the Next.js app

**Files:**
- Create: `/tmp/pulso-web/` (full `create-next-app` scaffold)

- [ ] **Step 1: Run `create-next-app` non-interactively**

Run:
```bash
cd /tmp
npx create-next-app@latest pulso-web \
  --typescript --eslint --app --no-tailwind --no-src-dir \
  --import-alias "@/*" --use-npm
cd /tmp/pulso-web
```
Expected: scaffold created with `app/`, `package.json`, `tsconfig.json`, `next.config.ts`, `.eslintrc` (or `eslint.config.*`), and a git repo already initialized (create-next-app runs `git init` + an initial commit itself).

- [ ] **Step 2: Set `output: "standalone"` in `next.config.ts`**

Required for the Docker runtime image in Task 11 — SSR needs a Node server, this trims the production `node_modules` footprint.

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
};

export default nextConfig;
```

- [ ] **Step 3: Add test dependencies**

Run:
```bash
npm install --save-dev vitest @testing-library/react @testing-library/jest-dom jsdom @vitejs/plugin-react @playwright/test
```
Expected: `package.json` `devDependencies` updated.

- [ ] **Step 4: Add Chart.js dependencies**

Run:
```bash
npm install chart.js react-chartjs-2
```
Expected: `package.json` `dependencies` updated.

- [ ] **Step 5: Add `vitest.config.ts`**

```typescript
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    setupFiles: ["./tests/unit/test-setup.tsx"],
    globals: true,
    exclude: ["**/node_modules/**", "**/tests/e2e/**"],
  },
});
```

- [ ] **Step 6: Add test scripts to `package.json`**

Add to the `"scripts"` block:
```json
"test": "vitest run",
"test:watch": "vitest",
"test:e2e": "playwright test"
```

- [ ] **Step 7: Verify the scaffold builds and runs**

Run:
```bash
npm run build
```
Expected: exits 0, `.next/` created including `.next/standalone/`.

- [ ] **Step 8: Commit**

```bash
git add .
git commit -m "chore: add test tooling, chart.js, and standalone output config"
```

---

### Task 3: Global styles and layout

**Files:**
- Create: `/tmp/pulso-web/app/globals.css` (replaces the scaffold's default)
- Modify: `/tmp/pulso-web/app/layout.tsx`

**Interfaces:** none new.

- [ ] **Step 1: Copy the existing stylesheet verbatim**

This file's content is unchanged from `pulso-dashboard` — copy it directly rather than retyping it:
```bash
cp /home/yagoazedias/github/pulso-dashboard/frontend/src/styles.css /tmp/pulso-web/app/globals.css
```
Expected: `app/globals.css` now contains the exact CSS from `pulso-dashboard` (CSS custom properties for colors, `.sidebar`, `.stat-cards`, `.chart-card`, `.date-range`, responsive breakpoints, etc.)

- [ ] **Step 2: Rewrite `app/layout.tsx`**

Replaces the scaffold's default (which references Geist fonts and a different global stylesheet) with the sidebar shell ported from `pulso-dashboard/frontend/src/components/App.jsx`:

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
  title: "Pulso Dashboard",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en">
      <body>
        <div className="app">
          <aside className="sidebar">
            <div className="sidebar-brand">
              <div className="sidebar-brand-icon">P</div>
              <span className="sidebar-brand-name">Pulso</span>
            </div>
            <ul className="sidebar-nav">
              <li className="sidebar-nav-item active">Dashboard</li>
            </ul>
          </aside>
          <main className="main">{children}</main>
        </div>
      </body>
    </html>
  );
}
```

- [ ] **Step 3: Delete unused scaffold assets**

Run:
```bash
cd /tmp/pulso-web
rm -f app/page.module.css app/favicon.ico public/*.svg
```
Expected: removes the default Next.js starter's demo styles/assets (not used by this app). `app/page.tsx` still exists — it gets fully rewritten in Task 9, so leave it as-is for now.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat: port global styles and app shell layout"
```

---

### Task 4: API client (`lib/api.ts`)

**Files:**
- Create: `/tmp/pulso-web/lib/api.ts`

**Interfaces:**
- Produces: `MetricsResponse` type (`{labels: string[]; datasets: {label: string; data: (number | null)[]}[]; meta: {unit: string; window: string; last_updated: string | null}}`), and `fetchEnergyVsGoal`, `fetchWorkoutVolume`, `fetchTopRecordTypes` — each `(start?: string, end?: string) => Promise<MetricsResponse>`. Consumed by `app/page.tsx` (Task 9).

- [ ] **Step 1: Write `lib/api.ts`**

```typescript
export type Dataset = {
  label: string;
  data: (number | null)[];
};

export type MetricsResponse = {
  labels: string[];
  datasets: Dataset[];
  meta: {
    unit: string;
    window: string;
    last_updated: string | null;
  };
};

const API_BASE_URL = process.env.API_BASE_URL ?? "http://localhost:8000";

async function fetchMetrics(
  path: string,
  start?: string,
  end?: string
): Promise<MetricsResponse> {
  const params = new URLSearchParams();
  if (start) params.set("start", start);
  if (end) params.set("end", end);
  const query = params.toString();
  const url = `${API_BASE_URL}${path}${query ? `?${query}` : ""}`;

  const res = await fetch(url, { cache: "no-store" });
  if (!res.ok) {
    throw new Error(`HTTP ${res.status} fetching ${path}`);
  }
  return res.json();
}

export function fetchEnergyVsGoal(start?: string, end?: string) {
  return fetchMetrics("/api/metrics/energy-vs-goal", start, end);
}

export function fetchWorkoutVolume(start?: string, end?: string) {
  return fetchMetrics("/api/metrics/workout-volume", start, end);
}

export function fetchTopRecordTypes(start?: string, end?: string) {
  return fetchMetrics("/api/metrics/top-record-types", start, end);
}
```

- [ ] **Step 2: Verify it type-checks**

Run:
```bash
cd /tmp/pulso-web
npx tsc --noEmit
```
Expected: exits 0, no type errors.

- [ ] **Step 3: Commit**

```bash
git add .
git commit -m "feat: add typed API client for the 3 metrics endpoints"
```

---

### Task 5: Presentational components — `ChartCard`, `StatCard`

**Files:**
- Create: `/tmp/pulso-web/components/ChartCard.tsx`
- Create: `/tmp/pulso-web/components/StatCard.tsx`
- Create: `/tmp/pulso-web/tests/unit/test-setup.tsx`
- Create: `/tmp/pulso-web/tests/unit/ChartCard.test.tsx`

**Interfaces:**
- Produces: `ChartCard` (props: `title: string; meta?: string; loading?: boolean; error?: string; empty?: boolean; children?: React.ReactNode`), `StatCard` (props: `label: string; value: string; sub?: string`). Consumed by the 3 chart components (Task 7) and `Dashboard` (Task 8).

These are unchanged in behavior from `pulso-dashboard/frontend/src/components/ChartCard.jsx` and `StatCard.jsx` — ported to TSX with prop types, no logic changes.

- [ ] **Step 1: Write `components/ChartCard.tsx`**

```tsx
export type ChartCardProps = {
  title: string;
  meta?: string | null;
  loading?: boolean;
  error?: string | null;
  empty?: boolean;
  children?: React.ReactNode;
};

export default function ChartCard({
  title,
  meta,
  loading,
  error,
  empty,
  children,
}: ChartCardProps) {
  return (
    <div className="chart-card">
      <div className="chart-card-header">
        <h3 className="chart-card-title">{title}</h3>
        {meta && <span className="chart-card-meta">{meta}</span>}
      </div>
      <div className="chart-card-body">
        {loading ? (
          <div className="chart-state">
            <div className="spinner" />
          </div>
        ) : error ? (
          <div className="chart-state error">{error}</div>
        ) : empty ? (
          <div className="chart-state">No data available</div>
        ) : (
          children
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Write `components/StatCard.tsx`**

```tsx
export type StatCardProps = {
  label: string;
  value: string;
  sub?: string;
};

export default function StatCard({ label, value, sub }: StatCardProps) {
  return (
    <div className="stat-card">
      <div className="stat-card-label">{label}</div>
      <div className="stat-card-value">{value}</div>
      {sub && <div className="stat-card-sub">{sub}</div>}
    </div>
  );
}
```

- [ ] **Step 3: Write the shared Vitest setup `tests/unit/test-setup.tsx`**

Mocks `react-chartjs-2`'s `Line` component the same way `pulso-dashboard`'s tests do, so chart components (Task 7) can be tested without a real canvas:

```tsx
import "@testing-library/jest-dom";
import { vi } from "vitest";

vi.mock("react-chartjs-2", () => ({
  Line: (props: Record<string, unknown>) => (
    <canvas data-testid="chart" data-props={JSON.stringify(props)} />
  ),
}));
```

- [ ] **Step 4: Write the failing test `tests/unit/ChartCard.test.tsx`**

Ported from `pulso-dashboard/frontend/src/components/ChartCard.test.jsx`:

```tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import ChartCard from "@/components/ChartCard";

describe("ChartCard", () => {
  it("shows spinner when loading", () => {
    const { container } = render(<ChartCard title="Test" loading />);
    expect(container.querySelector(".spinner")).toBeInTheDocument();
  });

  it("shows error message when error is set", () => {
    render(<ChartCard title="Test" error="Something broke" />);
    expect(screen.getByText("Something broke")).toBeInTheDocument();
  });

  it("shows no data message when empty", () => {
    render(<ChartCard title="Test" empty />);
    expect(screen.getByText("No data available")).toBeInTheDocument();
  });

  it("renders children when data is present", () => {
    render(
      <ChartCard title="Test">
        <p>Chart content</p>
      </ChartCard>
    );
    expect(screen.getByText("Chart content")).toBeInTheDocument();
  });

  it("displays meta when provided", () => {
    render(<ChartCard title="Test" meta="Unit: kcal" />);
    expect(screen.getByText("Unit: kcal")).toBeInTheDocument();
  });
});
```

- [ ] **Step 5: Run and verify**

Run:
```bash
cd /tmp/pulso-web
npm test -- ChartCard
```
Expected: 5 tests pass.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: port ChartCard and StatCard presentational components"
```

---

### Task 6: `DateRangeSelector` (Client Component, navigation-only)

**Files:**
- Create: `/tmp/pulso-web/components/DateRangeSelector.tsx`
- Create: `/tmp/pulso-web/tests/unit/DateRangeSelector.test.tsx`

**Interfaces:**
- Produces: `DateRangeSelector` (props: `startDate: string; endDate: string`). Consumed by `Dashboard` (Task 8). Unlike the original (which took an `onChange` callback prop), this version owns its own navigation — it reads/writes the URL directly via `next/navigation`, since that's now the single source of truth for the selected range (see Task 9).

This is the one component whose *external* behavior genuinely changes (per the design's redesign, not port, decision) — today's `onChange` callback becomes a direct `router.push`.

- [ ] **Step 1: Write `components/DateRangeSelector.tsx`**

```tsx
"use client";

import { useRouter, useSearchParams } from "next/navigation";

export type DateRangeSelectorProps = {
  startDate: string;
  endDate: string;
};

export default function DateRangeSelector({
  startDate,
  endDate,
}: DateRangeSelectorProps) {
  const router = useRouter();
  const searchParams = useSearchParams();

  function navigate(nextStart: string, nextEnd: string) {
    const params = new URLSearchParams(searchParams.toString());
    params.set("start", nextStart);
    params.set("end", nextEnd);
    router.push(`/?${params.toString()}`);
  }

  return (
    <div className="date-range">
      <label>From</label>
      <input
        type="date"
        value={startDate}
        onChange={(e) => navigate(e.target.value, endDate)}
      />
      <span className="date-range-separator">&mdash;</span>
      <label>To</label>
      <input
        type="date"
        value={endDate}
        onChange={(e) => navigate(startDate, e.target.value)}
      />
    </div>
  );
}
```

- [ ] **Step 2: Write the failing test `tests/unit/DateRangeSelector.test.tsx`**

Ported from `pulso-dashboard/frontend/src/components/DateRangeSelector.test.jsx`, updated to assert on `router.push` instead of an `onChange` callback:

```tsx
import { describe, it, expect, vi } from "vitest";
import { render, screen, fireEvent } from "@testing-library/react";

const push = vi.fn();
vi.mock("next/navigation", () => ({
  useRouter: () => ({ push }),
  useSearchParams: () => new URLSearchParams(),
}));

import DateRangeSelector from "@/components/DateRangeSelector";

describe("DateRangeSelector", () => {
  it("renders with provided dates", () => {
    render(<DateRangeSelector startDate="2026-01-01" endDate="2026-03-31" />);
    expect(screen.getByDisplayValue("2026-01-01")).toBeInTheDocument();
    expect(screen.getByDisplayValue("2026-03-31")).toBeInTheDocument();
  });

  it("navigates with updated startDate", () => {
    render(<DateRangeSelector startDate="2026-01-01" endDate="2026-03-31" />);
    fireEvent.change(screen.getByDisplayValue("2026-01-01"), {
      target: { value: "2026-02-01" },
    });
    expect(push).toHaveBeenCalledWith("/?start=2026-02-01&end=2026-03-31");
  });

  it("navigates with updated endDate", () => {
    render(<DateRangeSelector startDate="2026-01-01" endDate="2026-03-31" />);
    fireEvent.change(screen.getByDisplayValue("2026-03-31"), {
      target: { value: "2026-04-30" },
    });
    expect(push).toHaveBeenCalledWith("/?start=2026-01-01&end=2026-04-30");
  });
});
```

- [ ] **Step 3: Run and verify**

Run:
```bash
cd /tmp/pulso-web
npm test -- DateRangeSelector
```
Expected: 3 tests pass.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat: add DateRangeSelector as a navigation-only Client Component"
```

---

### Task 7: Chart components (Client Components, data via props)

**Files:**
- Create: `/tmp/pulso-web/lib/chartSetup.ts`
- Create: `/tmp/pulso-web/components/EnergyChart.tsx`
- Create: `/tmp/pulso-web/components/WorkoutVolumeChart.tsx`
- Create: `/tmp/pulso-web/components/TopRecordTypesChart.tsx`
- Create: `/tmp/pulso-web/tests/unit/EnergyChart.test.tsx`

**Interfaces:**
- Consumes: `MetricsResponse` type (Task 4), `ChartCard` (Task 5).
- Produces: `EnergyChart` (props: `data: MetricsResponse`), `WorkoutVolumeChart` (props: `data: MetricsResponse`), `TopRecordTypesChart` (props: `data: MetricsResponse`). Consumed by `Dashboard` (Task 8). Unlike the original (which fetched via `useChartData` given `startDate`/`endDate` props), these now receive already-fetched data directly — no fetching, no `loading`/`error` state of their own (the parent Server Component's fetch either succeeds before rendering or throws into `app/error.tsx`, so these components never see a loading/error state; `loading`/`error` props on `ChartCard` are simply not passed here, only `empty`).

- [ ] **Step 1: Write `lib/chartSetup.ts`** — Chart.js scale/element registration, ported from `pulso-dashboard/frontend/src/main.jsx`'s `ChartJS.register(...)` call. Imported once by each chart component (Chart.js registration is idempotent, so importing it from multiple files is safe).

```typescript
"use client";

import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler,
} from "chart.js";

ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler
);
```

- [ ] **Step 2: Write `components/EnergyChart.tsx`**

Ported from `pulso-dashboard/frontend/src/components/EnergyChart.jsx`, dropping the `useChartData` call — `data` arrives as a prop instead:

```tsx
"use client";

import { Line } from "react-chartjs-2";
import ChartCard from "./ChartCard";
import type { MetricsResponse } from "@/lib/api";
import "@/lib/chartSetup";

export default function EnergyChart({ data }: { data: MetricsResponse }) {
  const empty = data.datasets.every((ds) => ds.data.length === 0);

  const chartData = {
    labels: data.labels,
    datasets: [
      {
        label: data.datasets[0]?.label || "Active Energy Burned",
        data: data.datasets[0]?.data || [],
        borderColor: "#ff6384",
        backgroundColor: "rgba(255, 99, 132, 0.08)",
        fill: true,
        tension: 0.3,
        pointRadius: 1,
      },
      {
        label: data.datasets[1]?.label || "Goal",
        data: data.datasets[1]?.data || [],
        borderColor: "#4bc0c0",
        backgroundColor: "rgba(75, 192, 192, 0.08)",
        borderDash: [5, 5],
        tension: 0.3,
        pointRadius: 0,
      },
    ],
  };

  const options = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: { position: "top" as const },
    },
    scales: {
      y: {
        beginAtZero: true,
        title: { display: true, text: data.meta.unit || "kcal" },
      },
    },
  };

  const meta = data.meta.unit ? `Unit: ${data.meta.unit}` : undefined;

  return (
    <ChartCard title="Daily Active Energy vs Goal" meta={meta} empty={empty}>
      <div style={{ height: 300 }}>
        <Line data={chartData} options={options} />
      </div>
    </ChartCard>
  );
}
```

- [ ] **Step 3: Write `components/WorkoutVolumeChart.tsx`**

Ported from `pulso-dashboard/frontend/src/components/WorkoutVolumeChart.jsx`:

```tsx
"use client";

import { Line } from "react-chartjs-2";
import ChartCard from "./ChartCard";
import type { MetricsResponse } from "@/lib/api";
import "@/lib/chartSetup";

const COLORS = [
  { border: "#36a2eb", bg: "rgba(54, 162, 235, 0.08)" },
  { border: "#ff9f40", bg: "rgba(255, 159, 64, 0.08)" },
  { border: "#ff6384", bg: "rgba(255, 99, 132, 0.08)" },
];

export default function WorkoutVolumeChart({ data }: { data: MetricsResponse }) {
  const empty = data.datasets.every((ds) => ds.data.length === 0);

  const chartData = {
    labels: data.labels,
    datasets: data.datasets.map((ds, i) => ({
      label: ds.label,
      data: ds.data,
      borderColor: COLORS[i % COLORS.length].border,
      backgroundColor: COLORS[i % COLORS.length].bg,
      fill: false,
      tension: 0.3,
      pointRadius: 3,
    })),
  };

  const options = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: { position: "top" as const },
    },
    scales: {
      y: { beginAtZero: true },
    },
  };

  return (
    <ChartCard title="Workout Volume Trend" meta="Weekly" empty={empty}>
      <div style={{ height: 280 }}>
        <Line data={chartData} options={options} />
      </div>
    </ChartCard>
  );
}
```

- [ ] **Step 4: Write `components/TopRecordTypesChart.tsx`**

Ported from `pulso-dashboard/frontend/src/components/TopRecordTypesChart.jsx`:

```tsx
"use client";

import { Line } from "react-chartjs-2";
import ChartCard from "./ChartCard";
import type { MetricsResponse } from "@/lib/api";
import "@/lib/chartSetup";

const PALETTE = ["#36a2eb", "#ff6384", "#ff9f40", "#4bc0c0", "#9966ff"];

export default function TopRecordTypesChart({ data }: { data: MetricsResponse }) {
  const empty = data.datasets.length === 0;

  const chartData = {
    labels: data.labels,
    datasets: data.datasets.map((ds, i) => ({
      label: ds.label.replace("HKQuantityTypeIdentifier", ""),
      data: ds.data,
      borderColor: PALETTE[i % PALETTE.length],
      backgroundColor: "transparent",
      tension: 0.3,
      pointRadius: 3,
    })),
  };

  const options = {
    responsive: true,
    maintainAspectRatio: false,
    plugins: {
      legend: { position: "top" as const },
    },
    scales: {
      y: {
        beginAtZero: true,
        title: { display: true, text: "count" },
      },
    },
  };

  return (
    <ChartCard title="Top Record Types Over Time" meta="Weekly, top 5" empty={empty}>
      <div style={{ height: 280 }}>
        <Line data={chartData} options={options} />
      </div>
    </ChartCard>
  );
}
```

- [ ] **Step 5: Write the failing test `tests/unit/EnergyChart.test.tsx`**

Rewritten from `pulso-dashboard/frontend/src/components/EnergyChart.test.jsx` — the original tested the `useChartData` call, which no longer exists; this version tests the data-transformation logic directly against a `data` prop:

```tsx
import { describe, it, expect } from "vitest";
import { render, screen } from "@testing-library/react";
import EnergyChart from "@/components/EnergyChart";
import type { MetricsResponse } from "@/lib/api";

const mockData: MetricsResponse = {
  labels: ["2026-01-01", "2026-01-02"],
  datasets: [
    { label: "Active Energy Burned", data: [300, 400] },
    { label: "Goal", data: [500, 500] },
  ],
  meta: { unit: "kcal", window: "90d", last_updated: "2026-01-02" },
};

describe("EnergyChart", () => {
  it("renders chart with correct data transformation", () => {
    render(<EnergyChart data={mockData} />);
    const chart = screen.getByTestId("chart");
    const props = JSON.parse(chart.getAttribute("data-props")!);
    expect(props.data.labels).toEqual(["2026-01-01", "2026-01-02"]);
    expect(props.data.datasets[0].borderColor).toBe("#ff6384");
    expect(props.data.datasets[0].fill).toBe(true);
    expect(props.data.datasets[1].borderColor).toBe("#4bc0c0");
    expect(props.data.datasets[1].borderDash).toEqual([5, 5]);
  });

  it("shows empty state when all datasets are empty", () => {
    const empty: MetricsResponse = {
      labels: [],
      datasets: [
        { label: "Active Energy Burned", data: [] },
        { label: "Goal", data: [] },
      ],
      meta: { unit: "kcal", window: "90d", last_updated: null },
    };
    render(<EnergyChart data={empty} />);
    expect(screen.getByText("No data available")).toBeInTheDocument();
  });

  it("displays the unit in chart meta", () => {
    render(<EnergyChart data={mockData} />);
    expect(screen.getByText("Unit: kcal")).toBeInTheDocument();
  });
});
```

- [ ] **Step 6: Run and verify**

Run:
```bash
cd /tmp/pulso-web
npm test -- EnergyChart
```
Expected: 3 tests pass.

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "feat: port chart components as Client Components receiving data via props"
```

---

### Task 8: `Dashboard` (presentational shell)

**Files:**
- Create: `/tmp/pulso-web/components/Dashboard.tsx`

**Interfaces:**
- Consumes: `DateRangeSelector` (Task 6), `StatCard` (Task 5), `EnergyChart`/`WorkoutVolumeChart`/`TopRecordTypesChart` (Task 7), `MetricsResponse` (Task 4).
- Produces: `Dashboard` (props: `startDate: string; endDate: string; energy: MetricsResponse; workouts: MetricsResponse; topRecordTypes: MetricsResponse`). Consumed by `app/page.tsx` (Task 9).

Unlike `pulso-dashboard/frontend/src/components/Dashboard.jsx` (which does its own 3 inline `fetch` calls to derive stat card values, duplicating the same 3 requests the chart components separately make via `useChartData`), this version computes the stat values from the *same* 3 already-fetched responses passed in as props — each endpoint is now fetched exactly once, by `app/page.tsx`.

- [ ] **Step 1: Write `components/Dashboard.tsx`**

```tsx
import DateRangeSelector from "./DateRangeSelector";
import StatCard from "./StatCard";
import EnergyChart from "./EnergyChart";
import WorkoutVolumeChart from "./WorkoutVolumeChart";
import TopRecordTypesChart from "./TopRecordTypesChart";
import type { MetricsResponse } from "@/lib/api";

export type DashboardProps = {
  startDate: string;
  endDate: string;
  energy: MetricsResponse;
  workouts: MetricsResponse;
  topRecordTypes: MetricsResponse;
};

function latestEnergyStat(energy: MetricsResponse): string {
  const vals = (energy.datasets[0]?.data ?? []).filter(
    (v): v is number => v != null
  );
  const latest = vals.length > 0 ? vals[vals.length - 1] : null;
  return latest != null ? `${Math.round(latest)} kcal` : "--";
}

function latestWorkoutsStat(workouts: MetricsResponse): string {
  const counts = workouts.datasets[0]?.data ?? [];
  const latest = counts.length > 0 ? counts[counts.length - 1] : null;
  return latest != null ? String(latest) : "--";
}

function topRecordTypeStat(topRecordTypes: MetricsResponse): string {
  const label = topRecordTypes.datasets[0]?.label;
  return label ? label.replace("HKQuantityTypeIdentifier", "") : "--";
}

export default function Dashboard({
  startDate,
  endDate,
  energy,
  workouts,
  topRecordTypes,
}: DashboardProps) {
  return (
    <>
      <div className="main-header">
        <h1 className="main-title">Dashboard</h1>
        <DateRangeSelector startDate={startDate} endDate={endDate} />
      </div>

      <div className="stat-cards">
        <StatCard
          label="Latest Active Energy"
          value={latestEnergyStat(energy)}
          sub="Most recent day"
        />
        <StatCard
          label="Workouts This Week"
          value={latestWorkoutsStat(workouts)}
          sub="Latest week"
        />
        <StatCard
          label="Top Record Type"
          value={topRecordTypeStat(topRecordTypes)}
          sub="By volume"
        />
      </div>

      <div className="charts-grid">
        <EnergyChart data={energy} />
        <WorkoutVolumeChart data={workouts} />
        <TopRecordTypesChart data={topRecordTypes} />
      </div>
    </>
  );
}
```

- [ ] **Step 2: Verify it type-checks**

Run:
```bash
cd /tmp/pulso-web
npx tsc --noEmit
```
Expected: exits 0.

- [ ] **Step 3: Commit**

```bash
git add .
git commit -m "feat: add Dashboard shell computing stat cards from shared fetch results"
```

---

### Task 9: The page — Server Component, `loading.tsx`, `error.tsx`

**Files:**
- Modify: `/tmp/pulso-web/app/page.tsx` (full rewrite of the scaffold default)
- Create: `/tmp/pulso-web/app/loading.tsx`
- Create: `/tmp/pulso-web/app/error.tsx`

**Interfaces:**
- Consumes: `fetchEnergyVsGoal`/`fetchWorkoutVolume`/`fetchTopRecordTypes` (Task 4), `Dashboard` (Task 8).

This is the architectural core of the redesign: `page.tsx` computes the default date range (90 days for energy, 12 weeks for workouts/record-types — same defaults as Django, per the design spec) only when `searchParams` doesn't supply one, fetches all 3 endpoints, and renders `Dashboard`. Next.js's `searchParams` prop on a page is a `Promise` that must be awaited (current stable App Router API).

- [ ] **Step 1: Rewrite `app/page.tsx`**

```tsx
import Dashboard from "@/components/Dashboard";
import {
  fetchEnergyVsGoal,
  fetchWorkoutVolume,
  fetchTopRecordTypes,
} from "@/lib/api";

function defaultRange(): { startDate: string; endDate: string } {
  const end = new Date();
  const start = new Date();
  start.setDate(end.getDate() - 90);
  return {
    startDate: start.toISOString().split("T")[0],
    endDate: end.toISOString().split("T")[0],
  };
}

export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{ start?: string; end?: string }>;
}) {
  const params = await searchParams;
  const fallback = defaultRange();
  const startDate = params.start ?? fallback.startDate;
  const endDate = params.end ?? fallback.endDate;

  const [energy, workouts, topRecordTypes] = await Promise.all([
    fetchEnergyVsGoal(params.start, params.end),
    fetchWorkoutVolume(params.start, params.end),
    fetchTopRecordTypes(params.start, params.end),
  ]);

  return (
    <Dashboard
      startDate={startDate}
      endDate={endDate}
      energy={energy}
      workouts={workouts}
      topRecordTypes={topRecordTypes}
    />
  );
}
```

Note: `startDate`/`endDate` passed to `Dashboard` (and from there to `DateRangeSelector`) are what the *inputs* display — they fall back to the client-computed 90-day default when no `searchParams` are present, matching `pulso-dashboard/frontend/src/components/Dashboard.jsx`'s `defaultRange()` behavior. The actual data fetches (`fetchEnergyVsGoal` etc.) pass `params.start`/`params.end` through as-is (possibly `undefined`), letting each endpoint apply its own default window server-side (90d for energy, 12w for the other two) — exactly like Django/`pulso-api` already do. These two defaults intentionally differ in scope (energy vs. the two 12-week endpoints) but are display-only for the date inputs; the actual query windows are always authoritative from the API.

- [ ] **Step 2: Write `app/loading.tsx`**

```tsx
export default function Loading() {
  return (
    <div className="chart-state">
      <div className="spinner" />
    </div>
  );
}
```

- [ ] **Step 3: Write `app/error.tsx`**

```tsx
"use client";

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="chart-state error">
      <p>Failed to load dashboard: {error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

- [ ] **Step 4: Build and verify it compiles**

Run:
```bash
cd /tmp/pulso-web
npm run build
```
Expected: exits 0.

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat: add Server Component page with searchParams-driven data fetching"
```

---

### Task 10: End-to-end verification against the live Django API

**Files:**
- Create: `/tmp/pulso-web/playwright.config.ts`
- Create: `/tmp/pulso-web/tests/e2e/dashboard.spec.ts`

**Interfaces:** none new — this task verifies Tasks 1–9 work together against a real backend.

Per the design spec, `pulso-web` is developed against the still-running Django API in `pulso-dashboard` (same contract `pulso-api` will expose later). This task starts that real API, loads it with the same small fixture used throughout this project's other verifications, and drives the actual rendered page.

- [ ] **Step 1: Write `playwright.config.ts`**

```typescript
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  timeout: 30_000,
  use: {
    baseURL: "http://localhost:3000",
  },
  webServer: {
    command: "npm run build && npm run start",
    url: "http://localhost:3000",
    reuseExistingServer: false,
    timeout: 60_000,
    env: {
      API_BASE_URL: "http://localhost:8000",
    },
  },
});
```

- [ ] **Step 2: Install Playwright browsers**

Run:
```bash
cd /tmp/pulso-web
npx playwright install --with-deps chromium
```
Expected: Chromium downloaded successfully.

- [ ] **Step 3: Load fixture data and start the real Django API**

Run (runs the ETL directly via `uv` rather than through `docker compose run`, since the latter's `-v` flag would collide with the `./data:/data` mount already defined in `pulso-etl`'s `docker-compose.yml` for the same target path):
```bash
cd /home/yagoazedias/github/pulso-etl
docker compose up -d db
uv sync
DB_HOST=localhost uv run python -m pulso.cli --file tests/fixtures/small-export.xml

cd /home/yagoazedias/github/pulso-dashboard
pip install -r requirements.txt
DB_HOST=localhost python manage.py runserver 8000 &
sleep 2
```
Expected: Django running on `:8000`, serving real (if sparse) data from the fixture.

- [ ] **Step 4: Write the failing e2e test `tests/e2e/dashboard.spec.ts`**

```typescript
import { test, expect } from "@playwright/test";

test("dashboard loads, shows charts, and date-range navigation updates the URL", async ({ page }) => {
  // Given the dashboard is opened with no query params (default range)
  await page.goto("/");

  // Then the page title and all 3 chart cards render
  await expect(page.getByRole("heading", { name: "Dashboard" })).toBeVisible();
  await expect(page.getByText("Daily Active Energy vs Goal")).toBeVisible();
  await expect(page.getByText("Workout Volume Trend")).toBeVisible();
  await expect(page.getByText("Top Record Types Over Time")).toBeVisible();

  // When the start date is changed
  const startInput = page.locator('input[type="date"]').first();
  await startInput.fill("2025-01-01");

  // Then the URL reflects the new searchParams and the page re-renders server-side
  await expect(page).toHaveURL(/start=2025-01-01/);
  await expect(page.getByRole("heading", { name: "Dashboard" })).toBeVisible();
});
```

- [ ] **Step 5: Run and verify**

Run:
```bash
cd /tmp/pulso-web
npm run test:e2e
```
Expected: 1 test passes. Playwright's `webServer` config builds and starts `pulso-web` automatically against `API_BASE_URL=http://localhost:8000` (the Django instance from Step 3).

- [ ] **Step 6: Clean up**

Run:
```bash
kill %1 2>/dev/null
cd /home/yagoazedias/github/pulso-etl
docker compose down
```

- [ ] **Step 7: Commit**

```bash
cd /tmp/pulso-web
git add .
git commit -m "test: add Playwright e2e test against the live Django API"
```

---

### Task 11: Dockerfile and docker-compose.yml

**Files:**
- Create: `/tmp/pulso-web/Dockerfile`
- Create: `/tmp/pulso-web/docker-compose.yml`

- [ ] **Step 1: Write `Dockerfile`**

Multi-stage, using the `output: "standalone"` build from Task 2 for a minimal runtime image:

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
ARG API_BASE_URL
ENV API_BASE_URL=${API_BASE_URL}
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

- [ ] **Step 2: Write `docker-compose.yml`**

```yaml
services:
  web:
    build:
      context: .
      dockerfile: Dockerfile
      args:
        API_BASE_URL: ${API_BASE_URL:-http://host.docker.internal:8000}
    environment:
      API_BASE_URL: ${API_BASE_URL:-http://host.docker.internal:8000}
    ports:
      - "3000:3000"
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

- [ ] **Step 3: Verify the image builds**

Run:
```bash
cd /tmp/pulso-web
docker compose build
```
Expected: builds successfully.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat: add Dockerfile and docker-compose.yml"
```

---

### Task 12: CI workflows

**Files:**
- Create: `/tmp/pulso-web/.github/workflows/tests.yml`
- Create: `/tmp/pulso-web/.github/workflows/docker.yml`

- [ ] **Step 1: Write `.github/workflows/tests.yml`**

Two jobs: Vitest for Client Components (fast, no backend needed), and Playwright e2e against a real Django instance running from `pulso-dashboard`'s source — checked out as a second repo in the same job, mirroring the fixture-based verification used elsewhere in this project.

```yaml
name: Build and Test

on:
  push:
    branches: [ master, main, develop ]
  pull_request:
    branches: [ master, main, develop ]

jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Run unit tests
        run: npm test

  e2e-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: pulso
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - name: Checkout pulso-web
        uses: actions/checkout@v4
        with:
          path: pulso-web

      - name: Checkout pulso-dashboard (Django reference API)
        uses: actions/checkout@v4
        with:
          repository: pulso-health-tracker/pulso-dashboard
          path: pulso-dashboard

      - name: Checkout pulso-etl (fixture loader)
        uses: actions/checkout@v4
        with:
          repository: pulso-health-tracker/pulso-etl
          path: pulso-etl

      - name: Set up uv
        uses: astral-sh/setup-uv@v3

      - name: Load fixture data
        working-directory: ./pulso-etl
        run: |
          uv run python -m pulso.cli --file tests/fixtures/small-export.xml
        env:
          DB_HOST: localhost
          DB_USER: postgres
          DB_PASSWORD: postgres

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install Django dependencies
        working-directory: ./pulso-dashboard
        run: pip install -r requirements.txt

      - name: Start Django API
        working-directory: ./pulso-dashboard
        run: DB_HOST=localhost nohup python manage.py runserver 8000 &
        env:
          DB_HOST: localhost

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
          cache-dependency-path: pulso-web/package-lock.json

      - name: Install pulso-web dependencies
        working-directory: ./pulso-web
        run: npm ci

      - name: Install Playwright browsers
        working-directory: ./pulso-web
        run: npx playwright install --with-deps chromium

      - name: Run e2e tests
        working-directory: ./pulso-web
        run: npm run test:e2e
        env:
          API_BASE_URL: http://localhost:8000
```

- [ ] **Step 2: Write `.github/workflows/docker.yml`**

```yaml
name: Docker Build

on:
  push:
    branches: [ master, main ]
  pull_request:
    branches: [ master, main ]

jobs:
  docker-web:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Web Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: pulso-web:latest
          cache-from: type=gha,scope=web
          cache-to: type=gha,mode=max,scope=web
```

- [ ] **Step 3: Commit**

```bash
cd /tmp/pulso-web
git add .
git commit -m "ci: add unit test, e2e test, and docker build workflows"
```

---

### Task 13: README

**Files:**
- Create: `/tmp/pulso-web/README.md`

- [ ] **Step 1: Write `README.md`**

```markdown
# Pulso Web

Next.js (App Router, TypeScript) frontend for the [Pulso](https://github.com/pulso-health-tracker) health dashboard — same 3 charts as the original Django+Vite dashboard, redesigned around server-side rendering: `app/page.tsx` fetches data server-side based on URL search params, instead of client-side `useEffect` calls.

## Tech Stack

- **Next.js** (App Router), **TypeScript**
- **Chart.js** + `react-chartjs-2`
- **Vitest** + **Testing Library** (Client Component unit tests)
- **Playwright** (end-to-end)

## Prerequisites

- A running metrics API exposing `/api/metrics/{energy-vs-goal,workout-volume,top-record-types}` with the contract described below — either the Django API in [pulso-dashboard](https://github.com/pulso-health-tracker/pulso-dashboard) or [pulso-api](https://github.com/pulso-health-tracker/pulso-api).
- Node 20+, or Docker.

## Quick Start

### With Docker

```bash
API_BASE_URL=http://host.docker.internal:8000 docker compose up --build
```

App is then available at http://localhost:3000.

### Local Development

```bash
npm install
API_BASE_URL=http://localhost:8000 npm run dev
```

## Configuration

| Variable        | Default                  | Description                                             |
|------------------|----------------------------|-----------------------------------------------------------|
| `API_BASE_URL`    | `http://localhost:8000`    | Base URL of the metrics API — server-only, never sent to the browser |

## Architecture

`app/page.tsx` is a Server Component: it reads `start`/`end` from `searchParams`, fetches all 3 metrics endpoints server-side via `lib/api.ts`, and passes the results as props into `Dashboard`. `DateRangeSelector` is the only interactive piece — it's a Client Component that updates the URL via `router.push`, which triggers Next.js to re-run `page.tsx` on the server with the new params. The 3 chart components are Client Components (Chart.js needs a canvas) that render whatever data they're given via props — they never fetch anything themselves.

See `docs/superpowers/specs/2026-07-11-nextjs-frontend-migration-design.md` in `pulso-dashboard` for the full design rationale.

## Testing

```bash
# Unit tests (Client Components)
npm test

# End-to-end (requires a real metrics API running — see tests/e2e/dashboard.spec.ts)
npm run test:e2e
```

## Continuous Integration

**Build and Test** (`.github/workflows/tests.yml`) — Vitest unit tests, plus a Playwright e2e job that checks out `pulso-etl` and `pulso-dashboard`, loads fixture data, starts the real Django API, and drives the full rendered app.

**Docker Build** (`.github/workflows/docker.yml`) — builds the production Docker image.
```

- [ ] **Step 2: Commit**

```bash
cd /tmp/pulso-web
git add .
git commit -m "docs: add README"
```

---

### Task 14: Create the GitHub repo and publish

**Files:** none new — pushes everything built so far.

- [ ] **Step 1: Create the repo and push**

Run (from `/tmp/pulso-web`):
```bash
gh repo create pulso-health-tracker/pulso-web --public --source=. --remote=origin --push
```
Expected: output ending with repo creation confirmation and a successful push.

- [ ] **Step 2: Verify the repo**

Run:
```bash
gh repo view pulso-health-tracker/pulso-web --json name,visibility,defaultBranchRef --jq '{name, visibility, branch: .defaultBranchRef.name}'
```
Expected: `{"name":"pulso-web","visibility":"PUBLIC","branch":"master"}` (or `"main"`, depending on `create-next-app`'s/`git`'s default branch name — record whichever it actually is).

- [ ] **Step 3: Verify CI is green**

Run:
```bash
sleep 15
gh run list --repo pulso-health-tracker/pulso-web --limit 5
```
Expected: `Build and Test` (both `unit-test` and `e2e-test` jobs) and `Docker Build` all `completed`/`success`. If any failed, inspect with `gh run view --repo pulso-health-tracker/pulso-web --log-failed <run-id>` and fix.

Note: the `e2e-test` job in CI checks out `pulso-health-tracker/pulso-dashboard` and `pulso-health-tracker/pulso-etl` directly from GitHub (Task 12, Step 1) — this only works once this repo is actually pushed and those two repos are public and accessible, which they are as of this task.
