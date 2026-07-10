# Pulso Dashboard

Django-based analytics dashboard for [Pulso](https://github.com/pulso-health-tracker/pulso-etl) — renders health metrics (active energy vs goal, workout volume, top record types) from the PostgreSQL database that the [pulso-etl](https://github.com/pulso-health-tracker/pulso-etl) pipeline populates.

This repo only contains the read-side dashboard. It does **not** run its own Postgres for real data — it expects `pulso-etl`'s database to already be running and migrated. See Prerequisites below.

## Tech Stack

- **Django** 4.2, **psycopg2**, **django-vite**, **gunicorn**, **whitenoise**
- **React** 18 + **Chart.js** (via `react-chartjs-2`), bundled with **Vite**
- **pytest** + **pytest-django** for backend tests, **vitest** + **testing-library** for frontend tests

## Prerequisites

- [pulso-etl](https://github.com/pulso-health-tracker/pulso-etl) running locally (`docker compose up db` at minimum) — this dashboard reads from its `pulso` database and `dashboard` schema.
- Docker and Docker Compose, or Python 3.12+ and Node 20+ for local (non-Docker) development.

## Quick Start

### With Docker (recommended)

```bash
# 1. Make sure pulso-etl's docker compose is already running (provides Postgres on localhost:5432)
# 2. Build and start the dashboard, pointing at that Postgres:
DB_HOST=host.docker.internal docker compose up --build
```

Dashboard is then available at http://localhost:8000.

### Local Development (no Docker)

```bash
# 1. Install backend dependencies
pip install -r requirements.txt

# 2. Install frontend dependencies
npm ci

# 3. In one terminal, run the Vite dev server
npm run dev

# 4. In another terminal, run Django (pointing at pulso-etl's Postgres)
DB_HOST=localhost DEBUG=true python manage.py runserver
```

## Configuration

| Variable                 | Default                          | Description                                   |
|---------------------------|-----------------------------------|------------------------------------------------|
| `DB_HOST`                 | `localhost`                      | PostgreSQL host (point this at `pulso-etl`'s DB) |
| `DB_PORT`                 | `5432`                           | PostgreSQL port                                 |
| `DB_NAME`                 | `pulso`                          | Database name                                   |
| `DB_USER`                 | `postgres`                       | Database user                                   |
| `DB_PASSWORD`              | `postgres`                       | Database password                               |
| `SECRET_KEY`               | `django-insecure-dev-only-...`   | Django secret key — set a real value in production |
| `DEBUG`                    | `true`                           | Django debug mode                               |
| `ALLOWED_HOSTS`            | `localhost,127.0.0.1`            | Comma-separated allowed hosts                   |

Django reads from the Postgres `search_path` `dashboard,public` — `dashboard` (its own schema) falls back to `public` (where `pulso-etl`'s tables live). Both schemas are created by `pulso-etl`'s `postgres-init/` scripts.

## API Endpoints

- `GET /` — dashboard HTML page
- `GET /api/metrics/energy-vs-goal` — daily active energy vs goal, last 90 days
- `GET /api/metrics/workout-volume` — weekly workout count/duration/energy
- `GET /api/metrics/top-record-types` — top 5 record types by volume, weekly

## Testing

```bash
# Backend tests (pytest + pytest-django)
pip install -r requirements.txt
DB_HOST=localhost DB_USER=postgres DB_PASSWORD=postgres pytest

# Frontend tests (vitest)
npm ci
npm test
```

## Continuous Integration

**Build and Test** (`.github/workflows/tests.yml`) runs frontend tests (`npm ci && npm test`) on every push and pull request.

**Docker Build** (`.github/workflows/docker.yml`) builds the Dashboard Docker image and validates `docker-compose.yml`.

### Status Badges

```markdown
![Build and Test](https://github.com/pulso-health-tracker/pulso-dashboard/actions/workflows/tests.yml/badge.svg)
![Docker Build](https://github.com/pulso-health-tracker/pulso-dashboard/actions/workflows/docker.yml/badge.svg)
```

## Project Structure

```
pulso-dashboard/
├── manage.py
├── requirements.txt
├── package.json
├── vite.config.js
├── Dockerfile
├── docker-compose.yml
├── conftest.py
├── pytest.ini
├── dashboard_project/          # Django settings, urls, wsgi/asgi
├── apps/
│   └── analytics/
│       ├── models.py            # Unmanaged models over pulso-etl's tables
│       ├── repositories.py      # Query layer for the 3 metrics
│       ├── views.py             # index + 3 JSON metric endpoints
│       ├── urls.py
│       ├── templates/analytics/index.html
│       └── tests/
└── frontend/
    └── src/
        ├── components/          # Dashboard, ChartCard, StatCard, 3 chart components, DateRangeSelector
        ├── hooks/useChartData.js
        └── main.jsx
```
