# pulso-api (Go) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and publish `pulso-health-tracker/pulso-api`, a Go/Echo/GORM service that reproduces Django's 3 metrics endpoints byte-for-byte, as a standalone repo untouched by any other workstream.

**Architecture:** Echo HTTP server → handler layer (query param parsing/validation) → repository layer (GORM queries + in-memory aggregation, mirroring `apps/analytics/repositories.py` exactly) → Postgres (read-only, schema owned by `pulso-etl`). No migrations, no writes, no HTML rendering.

**Tech Stack:** Go 1.24, Echo v4, GORM (`gorm.io/gorm` + `gorm.io/driver/postgres`), `testify` for assertions, Postgres 17 (external, provided by `pulso-etl`).

## Global Constraints

- Repo: `pulso-health-tracker/pulso-api`, public, created fresh (not extracted — greenfield).
- This plan touches **only** the new `pulso-api` repo. Never edit anything under `/home/yagoazedias/github/pulso-dashboard` or `/home/yagoazedias/github/pulso-etl`.
- Migration philosophy: **mechanical 1:1 port** of `apps/analytics/repositories.py` and `apps/analytics/views.py` from the `pulso-dashboard` repo — same aggregation logic, same default windows (90 days / 12 weeks), same JSON contract, same date-validation error shape. Do not rewrite the aggregation as SQL `GROUP BY`.
- Env vars: `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` (same names as `pulso-etl`/`pulso-dashboard` use today), plus `CORS_ALLOWED_ORIGIN` (default `http://localhost:5173`) and `PORT` (default `8080`).
- GORM models are read-only: `AutoMigrate` is never called anywhere in this codebase. Schema ownership stays entirely with `pulso-etl`.
- Docker base images: `golang:1.24-alpine` (builder) + `alpine:latest` (runtime) — consistent with the `-alpine` convention already used in `pulso-etl`/`pulso-dashboard`.
- Testing: Go `testing` + `testify`, integration tests against a real Postgres, Given/When/Then structure in test names/comments (matching the project's existing test style).

---

### Task 1: Preflight

**Files:** none (verification only).

- [ ] **Step 1: Verify Go toolchain**

Run:
```bash
go version
```
Expected: Go 1.24 or newer. If missing/older, install Go 1.24+ before continuing (e.g. via your platform's package manager or https://go.dev/dl/).

- [ ] **Step 2: Verify GitHub org access**

Run:
```bash
gh api orgs/pulso-health-tracker --jq '.login'
```
Expected: `pulso-health-tracker`.

- [ ] **Step 3: Confirm the reference Django implementation is available to diff against later**

Run:
```bash
test -f /home/yagoazedias/github/pulso-dashboard/apps/analytics/repositories.py && echo "found"
```
Expected: `found`. This file is the source of truth for Task 12's contract-compatibility check — do not modify it.

---

### Task 2: Scaffold the Go module

**Files:**
- Create: `/tmp/pulso-api/go.mod`
- Create: `/tmp/pulso-api/internal/config/config.go`
- Create: `/tmp/pulso-api/internal/db/db.go`
- Create: `/tmp/pulso-api/.gitignore`

**Interfaces:**
- Produces: `config.Config` struct (fields: `DBHost, DBPort, DBName, DBUser, DBPassword, CORSAllowedOrigin, Port string`) and `config.Load() Config`.
- Produces: `db.Connect(cfg config.Config) (*gorm.DB, error)`.

- [ ] **Step 1: Initialize the module**

Run:
```bash
mkdir -p /tmp/pulso-api
cd /tmp/pulso-api
git init
go mod init github.com/pulso-health-tracker/pulso-api
```
Expected: `go.mod` created with `module github.com/pulso-health-tracker/pulso-api` and a `go 1.24` (or your installed version) directive.

- [ ] **Step 2: Add dependencies**

Run (from `/tmp/pulso-api`):
```bash
go get github.com/labstack/echo/v4@latest
go get gorm.io/gorm@latest
go get gorm.io/driver/postgres@latest
go get github.com/stretchr/testify@latest
```
Expected: `go.mod` now lists these as requirements; a `go.sum` file is created.

- [ ] **Step 3: Write `internal/config/config.go`**

```go
package config

import "os"

type Config struct {
	DBHost            string
	DBPort            string
	DBName            string
	DBUser            string
	DBPassword        string
	CORSAllowedOrigin string
	Port              string
}

func Load() Config {
	return Config{
		DBHost:            getEnv("DB_HOST", "localhost"),
		DBPort:            getEnv("DB_PORT", "5432"),
		DBName:            getEnv("DB_NAME", "pulso"),
		DBUser:            getEnv("DB_USER", "postgres"),
		DBPassword:        getEnv("DB_PASSWORD", "postgres"),
		CORSAllowedOrigin: getEnv("CORS_ALLOWED_ORIGIN", "http://localhost:5173"),
		Port:              getEnv("PORT", "8080"),
	}
}

func getEnv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}
```

- [ ] **Step 4: Write `internal/db/db.go`**

```go
package db

import (
	"fmt"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"

	"github.com/pulso-health-tracker/pulso-api/internal/config"
)

func Connect(cfg config.Config) (*gorm.DB, error) {
	dsn := fmt.Sprintf(
		"host=%s port=%s dbname=%s user=%s password=%s sslmode=disable",
		cfg.DBHost, cfg.DBPort, cfg.DBName, cfg.DBUser, cfg.DBPassword,
	)
	return gorm.Open(postgres.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Silent),
	})
}
```

- [ ] **Step 5: Write `.gitignore`**

```
/pulso-api
*.test
```

- [ ] **Step 6: Verify it compiles**

Run:
```bash
cd /tmp/pulso-api
go build ./...
```
Expected: exits 0, no output (nothing to build into a binary yet since there's no `main.go`, but `go build ./...` on packages with no `main` still type-checks successfully).

- [ ] **Step 7: Commit**

```bash
git add .
git commit -m "chore: scaffold Go module, config, and db connection"
git log --oneline -1
```
Expected: one commit.

---

### Task 3: GORM models and test harness

**Files:**
- Create: `/tmp/pulso-api/internal/models/models.go`
- Create: `/tmp/pulso-api/internal/testutil/schema.sql`
- Create: `/tmp/pulso-api/internal/testutil/testutil.go`

**Interfaces:**
- Consumes: `config.Config` (Task 2).
- Produces: `models.ActivitySummary`, `models.Workout`, `models.Record`, `models.RecordType` structs. `testutil.TestDB(t *testing.T) *gorm.DB` — connects to `pulso_test`, applies `schema.sql`, truncates all 4 tables, returns a ready-to-use `*gorm.DB`. Every later task's tests call this at the start of each test function.

- [ ] **Step 1: Write `internal/models/models.go`**

These map to the same tables `apps/analytics/models.py` in `pulso-dashboard` defines (`managed = False` there; here, `AutoMigrate` is simply never called — same effect). No schema prefix is needed on the table names: this service's Postgres connection has no `search_path` override, so it resolves against the default `public` schema like any plain connection.

```go
package models

import "time"

type ActivitySummary struct {
	ID                     uint      `gorm:"column:id;primaryKey"`
	DateComponents         time.Time `gorm:"column:date_components"`
	ActiveEnergyBurned     *float64  `gorm:"column:active_energy_burned"`
	ActiveEnergyBurnedGoal *float64  `gorm:"column:active_energy_burned_goal"`
}

func (ActivitySummary) TableName() string { return "activity_summary" }

type Workout struct {
	ID                uint      `gorm:"column:id;primaryKey"`
	ActivityType      string    `gorm:"column:activity_type"`
	Duration          *float64  `gorm:"column:duration"`
	TotalEnergyBurned *float64  `gorm:"column:total_energy_burned"`
	StartDate         time.Time `gorm:"column:start_date"`
	EndDate           time.Time `gorm:"column:end_date"`
}

func (Workout) TableName() string { return "workout" }

type RecordType struct {
	ID         uint   `gorm:"column:id;primaryKey"`
	Identifier string `gorm:"column:identifier"`
}

func (RecordType) TableName() string { return "record_type" }

type Record struct {
	ID           uint       `gorm:"column:id;primaryKey"`
	RecordTypeID uint       `gorm:"column:record_type_id"`
	RecordType   RecordType `gorm:"foreignKey:RecordTypeID"`
	StartDate    time.Time  `gorm:"column:start_date"`
	EndDate      time.Time  `gorm:"column:end_date"`
}

func (Record) TableName() string { return "record" }
```

- [ ] **Step 2: Write `internal/testutil/schema.sql`**

This is a **test-only** minimal schema (just the 4 tables and columns this service reads), copied from `pulso-etl`'s real migrations (`migrations/20260214160000-create-lookup-tables.up.sql`, `...160200-create-record-tables.up.sql`, `...160300-create-workout-tables.up.sql`, `...160500-create-activity-summary.up.sql`). In production, `pulso-etl` already owns and applies the full schema before this service ever runs — this file exists purely so CI/local tests have somewhere to write fixtures.

```sql
CREATE TABLE IF NOT EXISTS record_type (
    id         SERIAL PRIMARY KEY,
    identifier TEXT NOT NULL UNIQUE
);

CREATE TABLE IF NOT EXISTS workout (
    id                       BIGSERIAL PRIMARY KEY,
    activity_type            TEXT NOT NULL,
    duration                 DOUBLE PRECISION,
    total_energy_burned      DOUBLE PRECISION,
    start_date               TIMESTAMPTZ NOT NULL,
    end_date                 TIMESTAMPTZ NOT NULL
);

CREATE TABLE IF NOT EXISTS activity_summary (
    id                        BIGSERIAL PRIMARY KEY,
    date_components           DATE NOT NULL UNIQUE,
    active_energy_burned      DOUBLE PRECISION,
    active_energy_burned_goal DOUBLE PRECISION,
    active_energy_burned_unit TEXT
);

CREATE TABLE IF NOT EXISTS record (
    id             BIGSERIAL PRIMARY KEY,
    record_type_id INTEGER NOT NULL REFERENCES record_type(id),
    start_date     TIMESTAMPTZ NOT NULL,
    end_date       TIMESTAMPTZ NOT NULL
);
```

- [ ] **Step 3: Write `internal/testutil/testutil.go`**

```go
package testutil

import (
	_ "embed"
	"fmt"
	"os"
	"testing"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
)

//go:embed schema.sql
var schemaSQL string

// TestDB connects to the pulso_test database, applies the test-only schema
// (idempotent — CREATE TABLE IF NOT EXISTS), truncates all 4 tables, and
// returns a ready-to-use handle. Call this at the start of every test.
func TestDB(t *testing.T) *gorm.DB {
	t.Helper()

	dsn := fmt.Sprintf(
		"host=%s port=%s dbname=%s user=%s password=%s sslmode=disable",
		getEnv("DB_HOST", "localhost"),
		getEnv("DB_PORT", "5432"),
		getEnv("TEST_DB_NAME", "pulso_test"),
		getEnv("DB_USER", "postgres"),
		getEnv("DB_PASSWORD", "postgres"),
	)

	db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Silent),
	})
	if err != nil {
		t.Fatalf("failed to connect to test database: %v", err)
	}

	if err := db.Exec(schemaSQL).Error; err != nil {
		t.Fatalf("failed to apply test schema: %v", err)
	}

	if err := db.Exec("TRUNCATE record, record_type, workout, activity_summary RESTART IDENTITY CASCADE").Error; err != nil {
		t.Fatalf("failed to truncate tables: %v", err)
	}

	return db
}

func getEnv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}
```

- [ ] **Step 4: Verify it compiles**

Run:
```bash
cd /tmp/pulso-api
go build ./...
```
Expected: exits 0.

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat: add GORM models and test harness"
```

---

### Task 4: Repository — `GetEnergyVsGoal`

**Files:**
- Create: `/tmp/pulso-api/internal/repository/metrics.go`
- Create: `/tmp/pulso-api/internal/repository/metrics_test.go`
- Create: `/tmp/pulso-api/internal/repository/testhelpers_test.go`

**Interfaces:**
- Consumes: `models.ActivitySummary` (Task 3), `testutil.TestDB` (Task 3).
- Produces: `repository.Dataset{Label string; Data []interface{}}`, `repository.Meta{Unit, Window string; LastUpdated *string}`, `repository.MetricsResponse{Labels []string; Datasets []Dataset; Meta Meta}`, `repository.MetricsRepository` struct, `repository.NewMetricsRepository(db *gorm.DB) *MetricsRepository`, and `(*MetricsRepository) GetEnergyVsGoal(start, end *string) (MetricsResponse, error)`. Later tasks (5, 6) add methods to the same `MetricsRepository`; Task 7 calls all three.

Reference behavior being ported: `apps/analytics/repositories.py`'s `get_energy_vs_goal` in `pulso-dashboard` (read-only — do not modify that file).

- [ ] **Step 1: Write `internal/repository/metrics.go` (types + `GetEnergyVsGoal` only for now)**

```go
package repository

import (
	"time"

	"gorm.io/gorm"

	"github.com/pulso-health-tracker/pulso-api/internal/models"
)

type MetricsRepository struct {
	db *gorm.DB
}

func NewMetricsRepository(db *gorm.DB) *MetricsRepository {
	return &MetricsRepository{db: db}
}

type Dataset struct {
	Label string        `json:"label"`
	Data  []interface{} `json:"data"`
}

type Meta struct {
	Unit        string  `json:"unit"`
	Window      string  `json:"window"`
	LastUpdated *string `json:"last_updated"`
}

type MetricsResponse struct {
	Labels   []string  `json:"labels"`
	Datasets []Dataset `json:"datasets"`
	Meta     Meta      `json:"meta"`
}

const dateLayout = "2006-01-02"

func ptrToIface(f *float64) interface{} {
	if f == nil {
		return nil
	}
	return *f
}

// mondayOf returns the Monday (00:00, same location as t) of the ISO week
// containing t. Mirrors Python's `dt - timedelta(days=dt.weekday())`
// (Monday=0..Sunday=6). time.Weekday is Sunday=0..Saturday=6, so we convert.
func mondayOf(t time.Time) time.Time {
	daysSinceMonday := (int(t.Weekday()) + 6) % 7
	y, m, d := t.Date()
	day := time.Date(y, m, d, 0, 0, 0, 0, t.Location())
	return day.AddDate(0, 0, -daysSinceMonday)
}

func (r *MetricsRepository) GetEnergyVsGoal(start, end *string) (MetricsResponse, error) {
	isDefault := start == nil && end == nil
	today := time.Now().UTC()

	startVal := today.AddDate(0, 0, -90).Format(dateLayout)
	if start != nil {
		startVal = *start
	}
	endVal := today.Format(dateLayout)
	if end != nil {
		endVal = *end
	}

	var rows []models.ActivitySummary
	if err := r.db.
		Where("date_components >= ? AND date_components <= ?", startVal, endVal).
		Order("date_components ASC").
		Find(&rows).Error; err != nil {
		return MetricsResponse{}, err
	}

	labels := make([]string, 0, len(rows))
	energyData := make([]interface{}, 0, len(rows))
	goalData := make([]interface{}, 0, len(rows))
	var lastDate *string

	for _, row := range rows {
		dateStr := row.DateComponents.Format(dateLayout)
		labels = append(labels, dateStr)
		energyData = append(energyData, ptrToIface(row.ActiveEnergyBurned))
		goalData = append(goalData, ptrToIface(row.ActiveEnergyBurnedGoal))
		d := dateStr
		lastDate = &d
	}

	window := "custom"
	if isDefault {
		window = "90d"
	}

	return MetricsResponse{
		Labels: labels,
		Datasets: []Dataset{
			{Label: "Active Energy Burned", Data: energyData},
			{Label: "Goal", Data: goalData},
		},
		Meta: Meta{Unit: "kcal", Window: window, LastUpdated: lastDate},
	}, nil
}
```

- [ ] **Step 2: Write shared test helpers `internal/repository/testhelpers_test.go`** (used by this and Tasks 5/6's test files)

```go
package repository_test

import (
	"testing"
	"time"

	"github.com/stretchr/testify/require"
	"gorm.io/gorm"

	"github.com/pulso-health-tracker/pulso-api/internal/models"
)

func float64Ptr(f float64) *float64 { return &f }

func strPtr(s string) *string { return &s }

func mustDate(t *testing.T, s string) time.Time {
	t.Helper()
	d, err := time.Parse("2006-01-02", s)
	require.NoError(t, err)
	return d
}

func mustDateTime(t *testing.T, s string) time.Time {
	t.Helper()
	d, err := time.Parse(time.RFC3339, s)
	require.NoError(t, err)
	return d
}

func createRecordType(t *testing.T, db *gorm.DB, identifier string) models.RecordType {
	t.Helper()
	rt := models.RecordType{Identifier: identifier}
	require.NoError(t, db.Create(&rt).Error)
	return rt
}
```

- [ ] **Step 3: Write the failing tests in `internal/repository/metrics_test.go`**

```go
package repository_test

import (
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"

	"github.com/pulso-health-tracker/pulso-api/internal/models"
	"github.com/pulso-health-tracker/pulso-api/internal/repository"
	"github.com/pulso-health-tracker/pulso-api/internal/testutil"
)

func TestGetEnergyVsGoal_CustomRangeReturnsRowsInOrder(t *testing.T) {
	db := testutil.TestDB(t)
	repo := repository.NewMetricsRepository(db)

	// Given two activity_summary rows inside a custom range
	require.NoError(t, db.Create(&models.ActivitySummary{
		DateComponents:         mustDate(t, "2026-01-01"),
		ActiveEnergyBurned:     float64Ptr(500),
		ActiveEnergyBurnedGoal: float64Ptr(600),
	}).Error)
	require.NoError(t, db.Create(&models.ActivitySummary{
		DateComponents:         mustDate(t, "2026-01-02"),
		ActiveEnergyBurned:     float64Ptr(700),
		ActiveEnergyBurnedGoal: float64Ptr(600),
	}).Error)

	// When querying that exact range
	start, end := "2026-01-01", "2026-01-02"
	result, err := repo.GetEnergyVsGoal(&start, &end)

	// Then both rows come back in order, window is "custom"
	require.NoError(t, err)
	assert.Equal(t, []string{"2026-01-01", "2026-01-02"}, result.Labels)
	assert.Equal(t, []interface{}{500.0, 700.0}, result.Datasets[0].Data)
	assert.Equal(t, []interface{}{600.0, 600.0}, result.Datasets[1].Data)
	assert.Equal(t, "custom", result.Meta.Window)
	require.NotNil(t, result.Meta.LastUpdated)
	assert.Equal(t, "2026-01-02", *result.Meta.LastUpdated)
}

func TestGetEnergyVsGoal_DefaultWindowIs90Days(t *testing.T) {
	db := testutil.TestDB(t)
	repo := repository.NewMetricsRepository(db)

	// Given no rows at all
	// When called with no start/end
	result, err := repo.GetEnergyVsGoal(nil, nil)

	// Then window is "90d" and result is empty (no data in range)
	require.NoError(t, err)
	assert.Equal(t, "90d", result.Meta.Window)
	assert.Empty(t, result.Labels)
	assert.Nil(t, result.Meta.LastUpdated)
}

func TestGetEnergyVsGoal_NullGoalBecomesJSONNull(t *testing.T) {
	db := testutil.TestDB(t)
	repo := repository.NewMetricsRepository(db)

	// Given a row with no goal value set
	require.NoError(t, db.Create(&models.ActivitySummary{
		DateComponents:     mustDate(t, "2026-01-01"),
		ActiveEnergyBurned: float64Ptr(500),
	}).Error)

	start, end := "2026-01-01", "2026-01-01"
	result, err := repo.GetEnergyVsGoal(&start, &end)

	require.NoError(t, err)
	assert.Equal(t, []interface{}{nil}, result.Datasets[1].Data)
}
```

- [ ] **Step 4: Run against a real Postgres and verify it passes**

Start a local Postgres for tests if one isn't already running:
```bash
docker run -d --name pulso-api-test-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=pulso_test -p 5432:5432 postgres:17-alpine
sleep 3
```
Run:
```bash
cd /tmp/pulso-api
DB_HOST=localhost DB_USER=postgres DB_PASSWORD=postgres TEST_DB_NAME=pulso_test go test ./internal/repository/... -v -run TestGetEnergyVsGoal
```
Expected: `PASS` for all 3 tests, `ok` summary line.

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat: port GetEnergyVsGoal from Django's repositories.py"
```

---

### Task 5: Repository — `GetWorkoutVolume`

**Files:**
- Modify: `/tmp/pulso-api/internal/repository/metrics.go`
- Modify: `/tmp/pulso-api/internal/repository/metrics_test.go`

**Interfaces:**
- Consumes: `models.Workout` (Task 3), `mondayOf` (Task 4, same package).
- Produces: `(*MetricsRepository) GetWorkoutVolume(start, end *string) (MetricsResponse, error)`.

Reference behavior: `get_workout_volume` in `pulso-dashboard`'s `apps/analytics/repositories.py`.

- [ ] **Step 1: Add `GetWorkoutVolume` to `internal/repository/metrics.go`**

Append (needs `"sort"` added to the existing `import` block):

```go
func (r *MetricsRepository) GetWorkoutVolume(start, end *string) (MetricsResponse, error) {
	isDefault := start == nil && end == nil
	today := time.Now().UTC()

	startVal := today.AddDate(0, 0, -7*12).Format(dateLayout)
	if start != nil {
		startVal = *start
	}
	endVal := today.Format(dateLayout)
	if end != nil {
		endVal = *end
	}

	var workouts []models.Workout
	if err := r.db.
		Where("start_date >= ? AND start_date <= ?", startVal, endVal+"T23:59:59").
		Order("start_date ASC").
		Find(&workouts).Error; err != nil {
		return MetricsResponse{}, err
	}

	type weekAgg struct {
		count    int
		duration float64
		energy   float64
	}
	weeks := map[string]*weekAgg{}

	for _, w := range workouts {
		key := mondayOf(w.StartDate).Format(dateLayout)
		agg, ok := weeks[key]
		if !ok {
			agg = &weekAgg{}
			weeks[key] = agg
		}
		agg.count++
		if w.Duration != nil {
			agg.duration += *w.Duration / 60.0
		}
		if w.TotalEnergyBurned != nil {
			agg.energy += *w.TotalEnergyBurned
		}
	}

	sortedKeys := make([]string, 0, len(weeks))
	for k := range weeks {
		sortedKeys = append(sortedKeys, k)
	}
	sort.Strings(sortedKeys)

	counts := make([]interface{}, len(sortedKeys))
	durations := make([]interface{}, len(sortedKeys))
	energies := make([]interface{}, len(sortedKeys))
	for i, k := range sortedKeys {
		agg := weeks[k]
		counts[i] = agg.count
		durations[i] = agg.duration
		energies[i] = agg.energy
	}

	var lastDate *string
	if len(sortedKeys) > 0 {
		d := sortedKeys[len(sortedKeys)-1]
		lastDate = &d
	}

	window := "custom"
	if isDefault {
		window = "12w"
	}

	return MetricsResponse{
		Labels: sortedKeys,
		Datasets: []Dataset{
			{Label: "Workouts", Data: counts},
			{Label: "Duration (min)", Data: durations},
			{Label: "Energy Burned (kcal)", Data: energies},
		},
		Meta: Meta{Unit: "mixed", Window: window, LastUpdated: lastDate},
	}, nil
}
```

Update the `import` block at the top of the file to include `"sort"`:
```go
import (
	"sort"
	"time"

	"gorm.io/gorm"

	"github.com/pulso-health-tracker/pulso-api/internal/models"
)
```

- [ ] **Step 2: Append tests to `internal/repository/metrics_test.go`**

```go
func TestGetWorkoutVolume_GroupsByISOWeekMonday(t *testing.T) {
	db := testutil.TestDB(t)
	repo := repository.NewMetricsRepository(db)

	// Given two workouts in the same ISO week (Wed 2026-01-07, Fri 2026-01-09
	// — Monday of that week is 2026-01-05) and durations in seconds
	require.NoError(t, db.Create(&models.Workout{
		ActivityType:      "Running",
		Duration:          float64Ptr(1800), // 30 min
		TotalEnergyBurned: float64Ptr(300),
		StartDate:         mustDateTime(t, "2026-01-07T08:00:00Z"),
		EndDate:           mustDateTime(t, "2026-01-07T08:30:00Z"),
	}).Error)
	require.NoError(t, db.Create(&models.Workout{
		ActivityType:      "Cycling",
		Duration:          float64Ptr(3600), // 60 min
		TotalEnergyBurned: float64Ptr(400),
		StartDate:         mustDateTime(t, "2026-01-09T08:00:00Z"),
		EndDate:           mustDateTime(t, "2026-01-09T09:00:00Z"),
	}).Error)

	start, end := "2026-01-01", "2026-01-15"
	result, err := repo.GetWorkoutVolume(&start, &end)

	// Then they're grouped into one week, keyed by that week's Monday,
	// with duration summed in minutes (not seconds) and energy summed raw
	require.NoError(t, err)
	assert.Equal(t, []string{"2026-01-05"}, result.Labels)
	assert.Equal(t, []interface{}{2}, result.Datasets[0].Data)
	assert.Equal(t, []interface{}{90.0}, result.Datasets[1].Data)
	assert.Equal(t, []interface{}{700.0}, result.Datasets[2].Data)
}

func TestGetWorkoutVolume_SundayBelongsToPrecedingWeeksMonday(t *testing.T) {
	db := testutil.TestDB(t)
	repo := repository.NewMetricsRepository(db)

	// Given a workout on Sunday 2026-01-11 — the Monday of ISO week
	// containing a Sunday is 6 days earlier: 2026-01-05
	require.NoError(t, db.Create(&models.Workout{
		ActivityType:      "Swimming",
		Duration:          float64Ptr(1200),
		TotalEnergyBurned: float64Ptr(250),
		StartDate:         mustDateTime(t, "2026-01-11T09:00:00Z"),
		EndDate:           mustDateTime(t, "2026-01-11T09:20:00Z"),
	}).Error)

	start, end := "2026-01-01", "2026-01-15"
	result, err := repo.GetWorkoutVolume(&start, &end)

	require.NoError(t, err)
	assert.Equal(t, []string{"2026-01-05"}, result.Labels)
}

func TestGetWorkoutVolume_DefaultWindowIs12Weeks(t *testing.T) {
	db := testutil.TestDB(t)
	repo := repository.NewMetricsRepository(db)

	result, err := repo.GetWorkoutVolume(nil, nil)

	require.NoError(t, err)
	assert.Equal(t, "12w", result.Meta.Window)
}
```

- [ ] **Step 3: Run and verify**

Run:
```bash
cd /tmp/pulso-api
DB_HOST=localhost DB_USER=postgres DB_PASSWORD=postgres TEST_DB_NAME=pulso_test go test ./internal/repository/... -v -run TestGetWorkoutVolume
```
Expected: `PASS` for all 3 tests.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat: port GetWorkoutVolume from Django's repositories.py"
```

---

### Task 6: Repository — `GetTopRecordTypes`

**Files:**
- Modify: `/tmp/pulso-api/internal/repository/metrics.go`
- Modify: `/tmp/pulso-api/internal/repository/metrics_test.go`

**Interfaces:**
- Consumes: `models.Record`, `models.RecordType` (Task 3), `mondayOf` (Task 4, same package).
- Produces: `(*MetricsRepository) GetTopRecordTypes(start, end *string) (MetricsResponse, error)`.

Reference behavior: `get_top_record_types` in `pulso-dashboard`'s `apps/analytics/repositories.py`. **Read this carefully before implementing:** Python's `sorted(type_counts, key=type_counts.get, reverse=True)` sorts by count descending, and Python's sort is *stable* — on a tie, keys retain the order they were first inserted into `type_counts`, which happens while iterating `records` (already ordered by `start_date` ascending). So on a count tie between two record types, the one that occurred **chronologically first** wins the earlier slot in the top-5 list. Go's map iteration order is randomized, so a naive `for id := range counts` would break this — Step 1 below tracks first-seen order in a separate slice specifically to preserve it.

- [ ] **Step 1: Add `GetTopRecordTypes` to `internal/repository/metrics.go`**

```go
func (r *MetricsRepository) GetTopRecordTypes(start, end *string) (MetricsResponse, error) {
	isDefault := start == nil && end == nil
	today := time.Now().UTC()

	startVal := today.AddDate(0, 0, -7*12).Format(dateLayout)
	if start != nil {
		startVal = *start
	}
	endVal := today.Format(dateLayout)
	if end != nil {
		endVal = *end
	}

	window := "custom"
	if isDefault {
		window = "12w"
	}

	var records []models.Record
	if err := r.db.
		Preload("RecordType").
		Where("start_date >= ? AND start_date <= ?", startVal, endVal+"T23:59:59").
		Order("start_date ASC").
		Find(&records).Error; err != nil {
		return MetricsResponse{}, err
	}

	// Step 1: count total occurrences per type, tracking first-seen order
	// separately (see the note above this task's Step 1 in the plan) —
	// records are already ordered by start_date ascending, so "first-seen"
	// here means "chronologically first."
	counts := map[string]int{}
	var order []string
	seen := map[string]bool{}
	for _, rec := range records {
		id := rec.RecordType.Identifier
		if !seen[id] {
			seen[id] = true
			order = append(order, id)
		}
		counts[id]++
	}

	type typeCount struct {
		identifier string
		count      int
	}
	typeCounts := make([]typeCount, len(order))
	for i, id := range order {
		typeCounts[i] = typeCount{id, counts[id]}
	}
	sort.SliceStable(typeCounts, func(i, j int) bool {
		return typeCounts[i].count > typeCounts[j].count
	})

	topN := 5
	if len(typeCounts) < topN {
		topN = len(typeCounts)
	}
	topTypes := make([]string, topN)
	topTypesSet := map[string]bool{}
	for i := 0; i < topN; i++ {
		topTypes[i] = typeCounts[i].identifier
		topTypesSet[typeCounts[i].identifier] = true
	}

	if len(topTypes) == 0 {
		return MetricsResponse{
			Labels:   []string{},
			Datasets: []Dataset{},
			Meta:     Meta{Unit: "count", Window: window, LastUpdated: nil},
		}, nil
	}

	// Step 2: weekly counts per top type
	weekTypeCounts := map[string]map[string]int{}
	allWeeks := map[string]bool{}

	for _, rec := range records {
		id := rec.RecordType.Identifier
		if !topTypesSet[id] {
			continue
		}
		key := mondayOf(rec.StartDate).Format(dateLayout)
		allWeeks[key] = true
		if weekTypeCounts[key] == nil {
			weekTypeCounts[key] = map[string]int{}
		}
		weekTypeCounts[key][id]++
	}

	sortedWeeks := make([]string, 0, len(allWeeks))
	for w := range allWeeks {
		sortedWeeks = append(sortedWeeks, w)
	}
	sort.Strings(sortedWeeks)

	var lastDate *string
	if len(sortedWeeks) > 0 {
		d := sortedWeeks[len(sortedWeeks)-1]
		lastDate = &d
	}

	datasets := make([]Dataset, len(topTypes))
	for i, typeID := range topTypes {
		data := make([]interface{}, len(sortedWeeks))
		for j, w := range sortedWeeks {
			data[j] = weekTypeCounts[w][typeID]
		}
		datasets[i] = Dataset{Label: typeID, Data: data}
	}

	return MetricsResponse{
		Labels:   sortedWeeks,
		Datasets: datasets,
		Meta:     Meta{Unit: "count", Window: window, LastUpdated: lastDate},
	}, nil
}
```

- [ ] **Step 2: Append tests to `internal/repository/metrics_test.go`**

```go
func TestGetTopRecordTypes_TiesBreakByFirstChronologicalOccurrence(t *testing.T) {
	db := testutil.TestDB(t)
	repo := repository.NewMetricsRepository(db)

	// Given record types "B" and "A" each with exactly 1 record, where "B"
	// occurs chronologically first (see the note above this task's Step 1)
	typeB := createRecordType(t, db, "B")
	typeA := createRecordType(t, db, "A")
	require.NoError(t, db.Create(&models.Record{
		RecordTypeID: typeB.ID,
		StartDate:    mustDateTime(t, "2026-01-01T08:00:00Z"),
		EndDate:      mustDateTime(t, "2026-01-01T08:01:00Z"),
	}).Error)
	require.NoError(t, db.Create(&models.Record{
		RecordTypeID: typeA.ID,
		StartDate:    mustDateTime(t, "2026-01-02T08:00:00Z"),
		EndDate:      mustDateTime(t, "2026-01-02T08:01:00Z"),
	}).Error)

	start, end := "2026-01-01", "2026-01-15"
	result, err := repo.GetTopRecordTypes(&start, &end)

	// Then "B" (seen first) sorts before "A" on the count tie
	require.NoError(t, err)
	require.Len(t, result.Datasets, 2)
	assert.Equal(t, "B", result.Datasets[0].Label)
	assert.Equal(t, "A", result.Datasets[1].Label)
}

func TestGetTopRecordTypes_LimitsToTop5ByVolume(t *testing.T) {
	db := testutil.TestDB(t)
	repo := repository.NewMetricsRepository(db)

	// Given 6 distinct record types, with decreasing counts (6,5,4,3,2,1)
	identifiers := []string{"F", "E", "D", "C", "B", "A"}
	counts := []int{6, 5, 4, 3, 2, 1}
	for i, ident := range identifiers {
		rt := createRecordType(t, db, ident)
		for n := 0; n < counts[i]; n++ {
			require.NoError(t, db.Create(&models.Record{
				RecordTypeID: rt.ID,
				StartDate:    mustDateTime(t, "2026-01-01T08:00:00Z"),
				EndDate:      mustDateTime(t, "2026-01-01T08:01:00Z"),
			}).Error)
		}
	}

	start, end := "2026-01-01", "2026-01-15"
	result, err := repo.GetTopRecordTypes(&start, &end)

	// Then only the top 5 by volume appear (identifier "A", count 1, is excluded)
	require.NoError(t, err)
	require.Len(t, result.Datasets, 5)
	labels := make([]string, len(result.Datasets))
	for i, ds := range result.Datasets {
		labels[i] = ds.Label
	}
	assert.Equal(t, []string{"F", "E", "D", "C", "B"}, labels)
}

func TestGetTopRecordTypes_NoRecordsReturnsEmptyResponse(t *testing.T) {
	db := testutil.TestDB(t)
	repo := repository.NewMetricsRepository(db)

	start, end := "2026-01-01", "2026-01-15"
	result, err := repo.GetTopRecordTypes(&start, &end)

	require.NoError(t, err)
	assert.Equal(t, []string{}, result.Labels)
	assert.Equal(t, []repository.Dataset{}, result.Datasets)
	assert.Nil(t, result.Meta.LastUpdated)
}
```

- [ ] **Step 3: Run and verify — pay special attention to the tie-break test**

Run:
```bash
cd /tmp/pulso-api
DB_HOST=localhost DB_USER=postgres DB_PASSWORD=postgres TEST_DB_NAME=pulso_test go test ./internal/repository/... -v -run TestGetTopRecordTypes
```
Expected: `PASS` for all 3 tests, including `TestGetTopRecordTypes_TiesBreakByFirstChronologicalOccurrence`. If this specific test fails with datasets in the wrong order, the bug is almost always a map iterated directly instead of using the `order` slice from Step 1 — re-check that `topTypes`/`typeCounts` are built from `order`, not from ranging over `counts` directly.

- [ ] **Step 4: Run the full repository test suite**

Run:
```bash
cd /tmp/pulso-api
DB_HOST=localhost DB_USER=postgres DB_PASSWORD=postgres TEST_DB_NAME=pulso_test go test ./internal/repository/... -v
```
Expected: all tests from Tasks 4, 5, and 6 pass.

- [ ] **Step 5: Commit**

```bash
git add .
git commit -m "feat: port GetTopRecordTypes from Django's repositories.py"
```

---

### Task 7: HTTP layer — handlers, routing, CORS

**Files:**
- Create: `/tmp/pulso-api/internal/handlers/metrics.go`
- Create: `/tmp/pulso-api/internal/handlers/metrics_test.go`
- Create: `/tmp/pulso-api/main.go`

**Interfaces:**
- Consumes: `repository.MetricsRepository` and its 3 methods (Tasks 4–6), `config.Config` (Task 2), `db.Connect` (Task 2).
- Produces: `handlers.MetricsHandler`, `handlers.NewMetricsHandler(repo *repository.MetricsRepository) *MetricsHandler`, and 3 Echo handler methods (`EnergyVsGoal`, `WorkoutVolume`, `TopRecordTypes`) wired to routes `GET /api/metrics/energy-vs-goal`, `GET /api/metrics/workout-volume`, `GET /api/metrics/top-record-types` in `main.go`.

Reference behavior: `apps/analytics/views.py`'s `parse_date_params` and the 3 view functions in `pulso-dashboard`. The JSON error shape on invalid dates must match exactly: `{"error": "Invalid date format for '<name>': '<value>'. Expected YYYY-MM-DD."}` with HTTP 400 — written directly via `c.JSON`, not through Echo's default `HTTPError` wrapping (which would nest it under a `"message"` key and break the contract).

- [ ] **Step 1: Write `internal/handlers/metrics.go`**

```go
package handlers

import (
	"fmt"
	"net/http"
	"time"

	"github.com/labstack/echo/v4"

	"github.com/pulso-health-tracker/pulso-api/internal/repository"
)

type MetricsHandler struct {
	repo *repository.MetricsRepository
}

func NewMetricsHandler(repo *repository.MetricsRepository) *MetricsHandler {
	return &MetricsHandler{repo: repo}
}

// parseDateParams extracts and validates start/end query params, mirroring
// Django's parse_date_params: both optional, must be YYYY-MM-DD if present.
// On invalid input it writes the 400 response itself and returns ok=false —
// callers must return nil immediately in that case.
func parseDateParams(c echo.Context) (start, end *string, ok bool) {
	pairs := []struct {
		value string
		name  string
		out   **string
	}{
		{c.QueryParam("start"), "start", &start},
		{c.QueryParam("end"), "end", &end},
	}

	for _, p := range pairs {
		if p.value == "" {
			continue
		}
		if _, err := time.Parse("2006-01-02", p.value); err != nil {
			c.JSON(http.StatusBadRequest, map[string]string{
				"error": fmt.Sprintf("Invalid date format for '%s': '%s'. Expected YYYY-MM-DD.", p.name, p.value),
			})
			return nil, nil, false
		}
		v := p.value
		*p.out = &v
	}

	return start, end, true
}

func (h *MetricsHandler) EnergyVsGoal(c echo.Context) error {
	start, end, ok := parseDateParams(c)
	if !ok {
		return nil
	}
	data, err := h.repo.GetEnergyVsGoal(start, end)
	if err != nil {
		return c.JSON(http.StatusInternalServerError, map[string]string{"error": err.Error()})
	}
	return c.JSON(http.StatusOK, data)
}

func (h *MetricsHandler) WorkoutVolume(c echo.Context) error {
	start, end, ok := parseDateParams(c)
	if !ok {
		return nil
	}
	data, err := h.repo.GetWorkoutVolume(start, end)
	if err != nil {
		return c.JSON(http.StatusInternalServerError, map[string]string{"error": err.Error()})
	}
	return c.JSON(http.StatusOK, data)
}

func (h *MetricsHandler) TopRecordTypes(c echo.Context) error {
	start, end, ok := parseDateParams(c)
	if !ok {
		return nil
	}
	data, err := h.repo.GetTopRecordTypes(start, end)
	if err != nil {
		return c.JSON(http.StatusInternalServerError, map[string]string{"error": err.Error()})
	}
	return c.JSON(http.StatusOK, data)
}
```

- [ ] **Step 2: Write `internal/handlers/metrics_test.go`**

```go
package handlers_test

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/labstack/echo/v4"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"

	"github.com/pulso-health-tracker/pulso-api/internal/handlers"
	"github.com/pulso-health-tracker/pulso-api/internal/repository"
	"github.com/pulso-health-tracker/pulso-api/internal/testutil"
)

func TestEnergyVsGoal_InvalidStartDateReturns400WithExactErrorShape(t *testing.T) {
	db := testutil.TestDB(t)
	h := handlers.NewMetricsHandler(repository.NewMetricsRepository(db))

	e := echo.New()
	req := httptest.NewRequest(http.MethodGet, "/api/metrics/energy-vs-goal?start=not-a-date", nil)
	rec := httptest.NewRecorder()
	c := e.NewContext(req, rec)

	// When calling with an invalid start date
	err := h.EnergyVsGoal(c)

	// Then it returns 400 with the exact Django-compatible error shape
	require.NoError(t, err) // the handler writes the response itself; it doesn't return an error
	assert.Equal(t, http.StatusBadRequest, rec.Code)
	assert.JSONEq(t, `{"error": "Invalid date format for 'start': 'not-a-date'. Expected YYYY-MM-DD."}`, rec.Body.String())
}

func TestEnergyVsGoal_NoParamsReturns200WithDefaultWindow(t *testing.T) {
	db := testutil.TestDB(t)
	h := handlers.NewMetricsHandler(repository.NewMetricsRepository(db))

	e := echo.New()
	req := httptest.NewRequest(http.MethodGet, "/api/metrics/energy-vs-goal", nil)
	rec := httptest.NewRecorder()
	c := e.NewContext(req, rec)

	err := h.EnergyVsGoal(c)

	require.NoError(t, err)
	assert.Equal(t, http.StatusOK, rec.Code)
	assert.Contains(t, rec.Body.String(), `"window":"90d"`)
}
```

- [ ] **Step 3: Write `main.go`**

```go
package main

import (
	"log"
	"net/http"

	"github.com/labstack/echo/v4"
	"github.com/labstack/echo/v4/middleware"

	"github.com/pulso-health-tracker/pulso-api/internal/config"
	"github.com/pulso-health-tracker/pulso-api/internal/db"
	"github.com/pulso-health-tracker/pulso-api/internal/handlers"
	"github.com/pulso-health-tracker/pulso-api/internal/repository"
)

func main() {
	cfg := config.Load()

	gormDB, err := db.Connect(cfg)
	if err != nil {
		log.Fatalf("failed to connect to database: %v", err)
	}

	repo := repository.NewMetricsRepository(gormDB)
	h := handlers.NewMetricsHandler(repo)

	e := echo.New()
	e.Use(middleware.CORSWithConfig(middleware.CORSConfig{
		AllowOrigins: []string{cfg.CORSAllowedOrigin},
		AllowMethods: []string{http.MethodGet},
	}))

	e.GET("/api/metrics/energy-vs-goal", h.EnergyVsGoal)
	e.GET("/api/metrics/workout-volume", h.WorkoutVolume)
	e.GET("/api/metrics/top-record-types", h.TopRecordTypes)

	log.Printf("pulso-api listening on :%s", cfg.Port)
	e.Logger.Fatal(e.Start(":" + cfg.Port))
}
```

- [ ] **Step 4: Run all tests**

Run:
```bash
cd /tmp/pulso-api
DB_HOST=localhost DB_USER=postgres DB_PASSWORD=postgres TEST_DB_NAME=pulso_test go test ./... -v
```
Expected: all tests across `internal/repository` and `internal/handlers` pass.

- [ ] **Step 5: Build and smoke-test the binary**

Run:
```bash
cd /tmp/pulso-api
go build -o pulso-api .
DB_HOST=localhost DB_USER=postgres DB_PASSWORD=postgres ./pulso-api &
sleep 1
curl -s http://localhost:8080/api/metrics/energy-vs-goal
curl -s -w "\n%{http_code}\n" "http://localhost:8080/api/metrics/energy-vs-goal?start=bad"
kill %1
```
Expected: first `curl` returns `{"labels":[],"datasets":[...],"meta":{"unit":"kcal","window":"90d","last_updated":null}}` (empty since `pulso_test` was truncated by the last test run); second `curl` returns the 400 JSON error shape with status `400`.

- [ ] **Step 6: Commit**

```bash
git add .
git commit -m "feat: add HTTP handlers, routing, and CORS"
```

---

### Task 8: Dockerfile and docker-compose.yml

**Files:**
- Create: `/tmp/pulso-api/Dockerfile`
- Create: `/tmp/pulso-api/docker-compose.yml`

**Interfaces:** none new — packages what Task 7 built.

- [ ] **Step 1: Write `Dockerfile`**

```dockerfile
FROM golang:1.24-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o pulso-api .

FROM alpine:latest
WORKDIR /app
COPY --from=builder /app/pulso-api .
EXPOSE 8080
ENTRYPOINT ["./pulso-api"]
```

- [ ] **Step 2: Write `docker-compose.yml`**

No `db` service — this repo doesn't own Postgres, matching the established pattern (`pulso-etl` owns it). Connects to an already-running Postgres via env vars, defaulting to `host.docker.internal` so it can reach a host-published Postgres from `pulso-etl`'s compose.

```yaml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      DB_HOST: ${DB_HOST:-host.docker.internal}
      DB_PORT: ${DB_PORT:-5432}
      DB_NAME: ${DB_NAME:-pulso}
      DB_USER: ${DB_USER:-postgres}
      DB_PASSWORD: ${DB_PASSWORD:-postgres}
      CORS_ALLOWED_ORIGIN: ${CORS_ALLOWED_ORIGIN:-http://localhost:5173}
      PORT: "8080"
    ports:
      - "8080:8080"
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

- [ ] **Step 3: Verify the image builds**

Run:
```bash
cd /tmp/pulso-api
docker compose build
```
Expected: builds successfully, ending in a tagged image, no errors.

- [ ] **Step 4: Commit**

```bash
git add .
git commit -m "feat: add Dockerfile and docker-compose.yml"
```

---

### Task 9: CI workflows

**Files:**
- Create: `/tmp/pulso-api/.github/workflows/tests.yml`
- Create: `/tmp/pulso-api/.github/workflows/docker.yml`

**Interfaces:** none new.

- [ ] **Step 1: Write `.github/workflows/tests.yml`**

```yaml
name: Build and Test

on:
  push:
    branches: [ master, main, develop ]
  pull_request:
    branches: [ master, main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:17-alpine
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: pulso_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: "1.24"

      - name: Download dependencies
        run: go mod download

      - name: Run go vet
        run: go vet ./...

      - name: Run tests
        env:
          DB_HOST: localhost
          DB_USER: postgres
          DB_PASSWORD: postgres
          TEST_DB_NAME: pulso_test
        run: go test ./... -v
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
  docker-api:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build API Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: false
          tags: pulso-api:latest
          cache-from: type=gha,scope=api
          cache-to: type=gha,mode=max,scope=api
```

- [ ] **Step 3: Commit**

```bash
git add .
git commit -m "ci: add tests and docker build workflows"
```

---

### Task 10: README

**Files:**
- Create: `/tmp/pulso-api/README.md`

- [ ] **Step 1: Write `README.md`**

```markdown
# Pulso API

Go API serving the 3 metrics endpoints for the [Pulso](https://github.com/pulso-health-tracker) health dashboard — reads from the PostgreSQL database that [pulso-etl](https://github.com/pulso-health-tracker/pulso-etl) populates. Read-only: this service owns no schema and runs no migrations.

## Tech Stack

- **Go** 1.24, **Echo** v4 (HTTP), **GORM** + `gorm.io/driver/postgres` (DB access)
- **testify** for test assertions

## Prerequisites

- [pulso-etl](https://github.com/pulso-health-tracker/pulso-etl) running locally (`docker compose up db` at minimum) — this API reads from its `pulso` database.
- Go 1.24+, or Docker.

## Quick Start

### With Docker

```bash
# 1. Make sure pulso-etl's docker compose is already running (provides Postgres on localhost:5432)
# 2. Build and start the API, pointing at that Postgres:
DB_HOST=host.docker.internal docker compose up --build
```

API is then available at http://localhost:8080.

### Local Development (no Docker)

```bash
go mod download
DB_HOST=localhost go run .
```

## Configuration

| Variable               | Default                   | Description                          |
|-------------------------|----------------------------|---------------------------------------|
| `DB_HOST`                | `localhost`                | PostgreSQL host                       |
| `DB_PORT`                | `5432`                     | PostgreSQL port                       |
| `DB_NAME`                | `pulso`                    | Database name                         |
| `DB_USER`                | `postgres`                 | Database user                         |
| `DB_PASSWORD`             | `postgres`                 | Database password                     |
| `CORS_ALLOWED_ORIGIN`     | `http://localhost:5173`    | Allowed CORS origin                   |
| `PORT`                    | `8080`                     | HTTP listen port                      |

## Endpoints

- `GET /api/metrics/energy-vs-goal?start=YYYY-MM-DD&end=YYYY-MM-DD` — daily active energy vs goal, default last 90 days
- `GET /api/metrics/workout-volume?start=YYYY-MM-DD&end=YYYY-MM-DD` — weekly workout count/duration/energy, default last 12 weeks
- `GET /api/metrics/top-record-types?start=YYYY-MM-DD&end=YYYY-MM-DD` — top 5 record types by volume, weekly, default last 12 weeks

All three return `{"labels": [...], "datasets": [{"label": ..., "data": [...]}], "meta": {"unit": ..., "window": ..., "last_updated": ...}}`. Invalid `start`/`end` returns `400` with `{"error": "..."}`.

This is a byte-for-byte contract-compatible reimplementation of the Django API that used to live in `pulso-dashboard` — see `docs/superpowers/specs/2026-07-11-django-to-go-api-migration-design.md` in that repo for the full design rationale.

## Testing

```bash
# Start a local Postgres for tests
docker run -d --name pulso-api-test-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=pulso_test -p 5432:5432 postgres:17-alpine

# Run all tests
DB_HOST=localhost DB_USER=postgres DB_PASSWORD=postgres TEST_DB_NAME=pulso_test go test ./... -v
```

Tests apply their own minimal schema (`internal/testutil/schema.sql`) to `pulso_test` — production schema is always owned by `pulso-etl`.

## Continuous Integration

**Build and Test** (`.github/workflows/tests.yml`) — `go vet` + `go test ./...` against a Postgres service container, on every push/PR.

**Docker Build** (`.github/workflows/docker.yml`) — builds the API Docker image, on every push/PR.
```

- [ ] **Step 2: Commit**

```bash
git add .
git commit -m "docs: add README"
```

---

### Task 11: Create the GitHub repo and publish

**Files:** none new — pushes everything built so far.

- [ ] **Step 1: Create the repo and push**

Run (from `/tmp/pulso-api`):
```bash
gh repo create pulso-health-tracker/pulso-api --public --source=. --remote=origin --push
```
Expected: output ending with repo creation confirmation and a successful push.

- [ ] **Step 2: Verify the repo**

Run:
```bash
gh repo view pulso-health-tracker/pulso-api --json name,visibility,defaultBranchRef --jq '{name, visibility, branch: .defaultBranchRef.name}'
```
Expected: `{"name":"pulso-api","visibility":"PUBLIC","branch":"master"}`.

- [ ] **Step 3: Verify CI is green**

Run:
```bash
sleep 15
gh run list --repo pulso-health-tracker/pulso-api --limit 5
```
Expected: `Build and Test` and `Docker Build` both `completed`/`success`. If either failed, inspect with `gh run view --repo pulso-health-tracker/pulso-api --log-failed <run-id>` and fix before proceeding to Task 12.

---

### Task 12: Contract verification against the live Django API

**Files:** none — this is a manual verification task, not a code change.

**Purpose:** confirm `pulso-api` is genuinely contract-compatible with Django, per the design spec's §5 acceptance check — not just "returns some JSON," but byte-identical output for the same inputs.

- [ ] **Step 1: Load fixture data into pulso-etl's Postgres**

Run (mirrors the verification done earlier in this project for `pulso-etl` + `pulso-dashboard` — runs the ETL directly via `uv` rather than through `docker compose run`, since the latter's `-v` flag would collide with the `./data:/data` mount already defined in `pulso-etl`'s `docker-compose.yml` for the same target path):
```bash
cd /home/yagoazedias/github/pulso-etl
docker compose up -d db
uv sync
DB_HOST=localhost uv run python -m pulso.cli --file tests/fixtures/small-export.xml
```
Expected: exits 0, logs show records/workouts/etc. loaded.

- [ ] **Step 2: Start Django (pulso-dashboard) against that Postgres**

Run:
```bash
cd /home/yagoazedias/github/pulso-dashboard
pip install -r requirements.txt
DB_HOST=localhost python manage.py runserver 8000 &
sleep 2
```

- [ ] **Step 3: Start pulso-api against the same Postgres**

Run:
```bash
cd /tmp/pulso-api
DB_HOST=localhost PORT=8081 ./pulso-api &
sleep 1
```

- [ ] **Step 4: Diff the 3 endpoints**

Run:
```bash
for path in "energy-vs-goal" "workout-volume" "top-record-types"; do
  echo "=== $path ==="
  diff <(curl -s "http://localhost:8000/api/metrics/$path?start=2025-01-01&end=2025-02-01" | python3 -m json.tool --sort-keys) \
       <(curl -s "http://localhost:8081/api/metrics/$path?start=2025-01-01&end=2025-02-01" | python3 -m json.tool --sort-keys)
done
```
Expected: no diff output for any of the 3 endpoints (empty `diff` output means identical, modulo key order which `--sort-keys` normalizes).

- [ ] **Step 5: Clean up**

Run:
```bash
kill %1 %2 2>/dev/null
cd /home/yagoazedias/github/pulso-etl
docker compose down
```

- [ ] **Step 6: Record the result**

If Step 4 showed any diff, that's a real bug in the Go port — fix it in the relevant repository method (Tasks 4–6) and re-verify from Step 4. Once all 3 endpoints diff clean, `pulso-api` is done. No commit needed for this task (verification only).
