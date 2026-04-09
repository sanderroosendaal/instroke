# In-Stroke Analysis Platform — High-Level Architecture Proposal

> Community-maintained, open-source replacement for Rowsandall.com in-stroke analysis functionality.
> Companion service to intervals.icu. Zero hosting cost, fully serverless.

**Version:** 1.0  
**Date:** April 9, 2026  
**Status:** Proposal

---

## 1. Executive Summary

### 1.1 Purpose

Replace Rowsandall.com's in-stroke analysis capabilities before its shutdown (end of 2026) with a community-maintained, open-source platform that:

- Analyzes force curves, seat curves, boat acceleration, and other in-stroke metrics
- Integrates seamlessly with intervals.icu for data access and authentication
- Provides tools for stroke-by-stroke analysis, comparison, and trend visualization
- Supports multiple sensors (Empower Oarlock, Quiske, RP3, smartphone apps)
- Enables sharing and collaborative analysis

### 1.2 Integration Strategy: Extend vs Standalone

Two viable options are considered:

**Option A: Extend rownative/worker and courses**
- Pros: Shared authentication, infrastructure, deployment pipeline, DNS, user base
- Cons: Increased complexity, potential performance impact on courses API, tighter coupling

**Option B: Standalone instroke app**
- Pros: Independent deployment, focused scope, easier to contribute, clean separation
- Cons: Duplicate OAuth setup, separate DNS/deployment, fragmented user experience

**Recommendation:** **Option A (Extend)** with modular design:
- Add `/api/instroke/*` routes to existing worker
- Reuse authentication, KV, and D1 bindings
- Add `site/instroke/` subdirectory for UI
- Keep core logic in separate TypeScript modules
- Minimal impact on existing courses/challenges functionality

Benefits:
- Single sign-on: users already authenticated for courses/challenges
- Shared intervals.icu OAuth credentials and token management
- Unified deployment and CI/CD
- Natural cross-linking (e.g., "Analyze force curves from this course time")
- Reduced maintenance burden (one worker, one domain, one database)

---

## 2. Architecture Overview

### 2.1 System Context

```
┌─────────────────────────────────────────────────────────────────────┐
│                         intervals.icu                                │
│  - OAuth identity provider                                           │
│  - Activity metadata API                                             │
│  - FIT file download API                                             │
│  - Raw streams (latlng, time, hr, power, cadence)                   │
└───────────────────┬─────────────────────────────────────────────────┘
                    │ OAuth + API
                    │
┌───────────────────▼─────────────────────────────────────────────────┐
│                    Cloudflare Infrastructure                         │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  rownative Worker (TypeScript)                                │   │
│  │  - Existing: /api/courses, /api/challenges, /oauth/*          │   │
│  │  - NEW: /api/instroke/*                                        │   │
│  └────┬─────────────────────────────────────┬─────────────────┬──┘   │
│       │                                      │                 │      │
│  ┌────▼─────┐  ┌──────────────────────┐  ┌──▼──────────┐  ┌──▼────┐ │
│  │ D1 (DB)  │  │ KV (Cache + State)    │  │ R2 (Future) │  │ Queues│ │
│  │ - courses│  │ - liked courses       │  │ - FIT cache │  │(Future│ │
│  │ - times  │  │ - OAuth state         │  │ - analyses  │  │ )     │ │
│  │ - NEW:   │  │ - NEW: sensor configs │  └─────────────┘  └───────┘ │
│  │   analyses│ │ - NEW: shared analyses│                             │
│  │   curves  │  └──────────────────────┘                             │
│  │   metrics │                                                        │
│  └──────────┘                                                         │
└───────────────────┬─────────────────────────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────────────────────────┐
│              GitHub Pages (rownative.icu)                            │
│  - Existing: site/*.html (map, challenges, my-times)                 │
│  - NEW: site/instroke/*.html                                         │
│    - instroke/index.html (activity browser)                          │
│    - instroke/analyze.html (single workout analysis)                 │
│    - instroke/compare.html (multi-workout comparison)                │
│    - instroke/library.html (saved analyses)                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Technology Stack

**Unchanged from rownative/worker:**

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | HTML5 + Vanilla JS + Leaflet | Zero build step; fast; existing pattern |
| Charting | **D3.js v7** (from Rowsandall) | Interactive force curves, comparisons; reuse proven templates |
| API Backend | Cloudflare Workers (TypeScript) | Serverless, globally distributed, free tier |
| Database | Cloudflare D1 (SQLite) | Relational data (analyses, metrics), SQL queries |
| Cache/State | Cloudflare KV | User preferences, shared analysis links |
| File Storage | Cloudflare R2 (future) | FIT file caching, large analysis datasets |
| Hosting | Cloudflare Pages | Static site hosting, GitHub integration |
| Auth | intervals.icu OAuth | Existing flow; no credential management |
| FIT Parsing | JavaScript FIT SDK or fit-parser | Client or server-side parsing |

**New dependencies:**

- **FIT parsing library:** `fit-file-parser` (npm) or equivalent for server-side parsing
- **Charting library:** **D3.js v7** (reuse from Rowsandall chart server)
- **Math/stats utilities:** For curve normalization, quartile calculations, correlation

**Charting Strategy — Reuse Rowsandall D3.js Code:**

The Rowsandall chart server (https://git.wereldraadsel.tech/sander/rowsandall-charts) uses D3.js v7 with proven, well-tested templates. **We can reuse this code** rather than building from scratch:

**Approach: Extract D3.js Templates (Client-Side Rendering)**

1. **Extract D3.js code** from `rowsandall-charts/templates/*.js` files
   - `forcecurve.js` — force curve analysis with interactive sliders
   - `instroke.js` — in-stroke analysis with quartile shading
   - `instroke_compare.js` — multi-analysis comparison overlay

2. **Remove Go template syntax**, replace with JavaScript data injection:
   ```javascript
   // Rowsandall (Go template):
   const data = {{ .Data }}
   
   // New approach (JavaScript):
   const data = window.CHART_DATA; // injected by API response
   ```

3. **Embed in static site** as `site/instroke/js/charts/*.js`

4. **No Go server needed** — all rendering happens client-side

**Benefits:**
- ✅ Proven UX (users familiar with Rowsandall interface)
- ✅ Interactive features already implemented (sliders, filtering, crosshairs, export)
- ✅ Smooth migration path (minimal relearning)
- ✅ No additional server infrastructure (pure client-side)
- ✅ Open source code available for reuse

**Alternative (Not Recommended):** Deploy the full Go chart server as a microservice alongside the Worker. This works but adds operational complexity (separate deployment, monitoring, scaling, hosting cost).

**Rationale:** D3.js gives us more power and customization than Chart.js, and since the code already exists and works, reusing it is significantly faster than reimplementing in Chart.js.

---

## 3. Functional Components

### 3.1 Data Pipeline

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ intervals.icu│────▶│ Worker:      │────▶│ FIT Parser   │────▶│ D1: Extract  │
│ FIT download │     │ Fetch + Auth │     │ Extract      │     │ curves +     │
│              │     │              │     │ in-stroke    │     │ scalars      │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                                        │
                                           ┌────────────────────────────▼──────┐
                                           │ Analysis Engine                    │
                                           │ - Filter by spm, interval, time    │
                                           │ - Compute quartiles, diff, maxpos  │
                                           │ - Normalize curves (0-1 abscissa)  │
                                           │ - Store analysis snapshot          │
                                           └────────────────────────────────────┘
                                                         │
                          ┌──────────────────────────────┼──────────────────────────────┐
                          │                              │                              │
                     ┌────▼─────┐                  ┌─────▼──────┐              ┌────────▼────────┐
                     │ D1:      │                  │ D1:        │              │ UI:             │
                     │ analyses │                  │ curves     │              │ D3.js           │
                     │ (metadata)│                 │ (timeseries)│             │ Interactive     │
                     └──────────┘                  └────────────┘              │ comparison      │
                                                                                └─────────────────┘
```

### 3.2 Core Modules (Worker)

#### 3.2.1 FIT Ingestion Module (`src/instroke-ingest.ts`)

**Responsibilities:**
- Fetch FIT file from intervals.icu given activity ID + bearer token
- Parse FIT using fit-file-parser library
- Extract developer fields per FIT_EXPORT.md spec:
  - Force curves (HandleForceCurve, BoatAcceleratorCurve, etc.)
  - Oarlock scalars (catch, finish, slip, wash, peak force angle, effective length)
  - Stroke drive/recovery time, drive length, drag factor
  - Abscissa metadata (InstrokeAbscissaType, InstrokeSampleInterval, InstrokePointCount)
- Handle `.instroke.json` companion files if present
- Store raw extracted data in D1 for analysis

**Inputs:** `activityId`, `athleteId` (from session), intervals.icu token  
**Outputs:** Parsed workout record with per-stroke curves and scalars

**Error handling:**
- Activity not found → 404
- FIT parsing failure → 422 with error details
- No in-stroke data detected → 200 with empty curves (allow user to still analyze standard metrics)

#### 3.2.2 Analysis Engine Module (`src/instroke-analysis.ts`)

**Responsibilities:**
- Filter strokes by criteria (spm range, interval, time window)
- Compute per-curve statistics:
  - Quartiles (Q1, Q2/median, Q3, Q4)
  - Min/max positions along drive
  - Diff (Q4 - Q1) for asymmetry detection
  - Mean curve (stroke-averaged)
- Normalize curves to common abscissa (e.g., 0-1 normalized drive)
- Save analysis snapshot to D1 with user-provided name

**Inputs:** `workoutId` (foreign key), filter criteria, analysis name  
**Outputs:** Analysis ID, computed metrics, filtered stroke indices

**Key algorithms:**
- Curve resampling (when abscissa types differ between workouts)
- Outlier detection (optional: exclude strokes >3 SD from mean)
- Temporal smoothing (optional: moving average for noisy sensors)

#### 3.2.3 Comparison Module (`src/instroke-compare.ts`)

**Responsibilities:**
- Load multiple analyses by ID
- Align curves to common abscissa (time, handle distance, or normalized 0-1)
- Compute deltas (absolute and percentage)
- Generate comparison chart data (JSON for D3.js)

**Inputs:** Array of analysis IDs  
**Outputs:** Comparison data structure with aligned curves + metadata

**Features:**
- Overlay up to 5 analyses on one chart
- Color coding per analysis
- Legend with analysis metadata (date, spm range, boat, athlete)

#### 3.2.4 Sharing Module (`src/instroke-share.ts`)

**Responsibilities:**
- Generate shareable UUID for analysis or comparison
- Store share link in KV with TTL (e.g., 90 days)
- Public access to shared analyses (no auth required for read)
- Privacy: athlete name hidden unless analysis is public

**Inputs:** Analysis ID(s), privacy setting (public/private)  
**Outputs:** Share URL (`https://rownative.icu/instroke/shared/{uuid}`)

### 3.3 Database Schema (D1)

#### 3.3.1 `instroke_workouts`

Stores ingested workout metadata (one row per intervals.icu activity with in-stroke data).

| Column | Type | Description |
|--------|------|-------------|
| id | INTEGER PRIMARY KEY | Auto-increment |
| athlete_id | TEXT NOT NULL | intervals.icu athlete ID (from token) |
| activity_id | TEXT NOT NULL | intervals.icu activity ID |
| activity_date | DATE | Workout date from activity metadata |
| sport | TEXT | 'Rowing' or 'IndoorRowing' |
| duration_s | INTEGER | Total duration (seconds) |
| distance_m | INTEGER | Total distance (meters) |
| avg_spm | REAL | Average stroke rate |
| stroke_count | INTEGER | Total strokes |
| sensor_type | TEXT | 'Empower', 'Quiske', 'RP3', 'Smartphone', 'Unknown' |
| has_force_curve | BOOLEAN | HandleForceCurve present? |
| has_boat_accel_curve | BOOLEAN | BoatAcceleratorCurve present? |
| has_seat_curve | BOOLEAN | SeatCurve present? |
| has_oarlock_data | BOOLEAN | Catch/finish/slip/wash present? |
| abscissa_type | TEXT | 'TIME_UNIFORM_MS', 'HANDLE_DISTANCE_UNIFORM_M', etc. |
| created_at | DATETIME | Ingestion timestamp |
| updated_at | DATETIME | Last modified |

**Indexes:**
- `athlete_id, activity_date DESC` (user's workout list)
- `activity_id` (unique constraint)

#### 3.3.2 `instroke_curves`

Stores per-stroke curve data (normalized for querying and comparison).

| Column | Type | Description |
|--------|------|-------------|
| id | INTEGER PRIMARY KEY | Auto-increment |
| workout_id | INTEGER NOT NULL | Foreign key to instroke_workouts |
| stroke_number | INTEGER NOT NULL | 1-indexed stroke number |
| spm | REAL | Stroke rate for this stroke |
| interval_idx | INTEGER | Lap/interval index (from FIT) |
| curve_type | TEXT | 'HandleForceCurve', 'BoatAcceleratorCurve', etc. |
| abscissa_type | TEXT | X-axis semantic (from FIT metadata) |
| point_count | INTEGER | Number of points in curve |
| curve_data | BLOB | JSON array of Y values (compressed) |
| catch_angle | REAL | Oarlock catch angle (degrees) |
| finish_angle | REAL | Oarlock finish angle (degrees) |
| drive_time_ms | INTEGER | Stroke drive time (milliseconds) |
| recovery_time_ms | INTEGER | Stroke recovery time (milliseconds) |
| drive_length_m | REAL | Handle travel distance (meters) |
| peak_force_n | REAL | Peak force (Newtons) |
| avg_force_n | REAL | Average drive force (Newtons) |

**Indexes:**
- `workout_id, stroke_number` (fetch strokes for a workout)
- `workout_id, curve_type` (fetch specific curve type)

**Storage note:** BLOB stores JSON for flexibility; alternative is separate columns per point (harder to query variable-length curves).

#### 3.3.3 `instroke_analyses`

Stores user-created analysis snapshots (filtered/aggregated data for later comparison).

| Column | Type | Description |
|--------|------|-------------|
| id | INTEGER PRIMARY KEY | Auto-increment |
| uuid | TEXT NOT NULL UNIQUE | Public identifier for sharing |
| athlete_id | TEXT NOT NULL | Owner |
| workout_id | INTEGER NOT NULL | Foreign key |
| name | TEXT | User-provided name (e.g., "Race - 32 spm") |
| description | TEXT | Optional notes |
| filter_spm_min | REAL | SPM filter lower bound |
| filter_spm_max | REAL | SPM filter upper bound |
| filter_interval_idx | INTEGER | Specific interval (NULL = all) |
| filter_time_start_s | INTEGER | Time window start (seconds from workout start) |
| filter_time_end_s | INTEGER | Time window end |
| stroke_count | INTEGER | Number of strokes included |
| curve_types | TEXT | Comma-separated list of curves analyzed |
| is_public | BOOLEAN | Public sharing enabled? |
| created_at | DATETIME | Snapshot creation time |

**Indexes:**
- `athlete_id, created_at DESC` (user's analysis library)
- `uuid` (shared link lookup)

#### 3.3.4 `instroke_analysis_metrics`

Stores computed metrics for each analysis (quartiles, mean curve, etc.).

| Column | Type | Description |
|--------|------|-------------|
| id | INTEGER PRIMARY KEY | Auto-increment |
| analysis_id | INTEGER NOT NULL | Foreign key |
| curve_type | TEXT | 'HandleForceCurve', etc. |
| metric_name | TEXT | 'q1', 'q2', 'q3', 'q4', 'mean', 'diff', 'maxpos', 'minpos' |
| metric_data | BLOB | JSON array of Y values (normalized abscissa) |

**Indexes:**
- `analysis_id, curve_type, metric_name` (fetch metrics for comparison)

#### 3.3.5 `instroke_shares` (KV alternative)

If using KV instead of D1 for ephemeral share links:

**KV Key:** `instroke:share:{uuid}`  
**Value:** JSON: `{ analysisIds: [...], createdAt: ISO8601, expiresAt: ISO8601 }`  
**TTL:** 90 days

---

## 4. API Specification

### 4.1 Ingestion and Listing

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/instroke/workouts` | Cookie | List user's workouts with in-stroke data (paginated) |
| POST | `/api/instroke/workouts/ingest` | Cookie | Ingest activity from intervals.icu. Body: `{ activityId }` |
| GET | `/api/instroke/workouts/{id}` | Cookie | Workout detail + available curve types |
| DELETE | `/api/instroke/workouts/{id}` | Cookie | Delete workout and associated curves/analyses |

**Response for `GET /workouts`:**

```json
{
  "workouts": [
    {
      "id": 123,
      "activityId": "i12345678",
      "activityDate": "2026-03-15",
      "sport": "Rowing",
      "durationS": 1800,
      "distanceM": 5000,
      "avgSpm": 28.5,
      "strokeCount": 855,
      "sensorType": "Empower",
      "curveTypes": ["HandleForceCurve", "BoatAcceleratorCurve"],
      "hasOarlockData": true,
      "createdAt": "2026-03-16T10:00:00Z"
    }
  ],
  "pagination": { "page": 1, "perPage": 20, "total": 45 }
}
```

### 4.2 Analysis

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/instroke/analyses` | Cookie | Create analysis. Body: `{ workoutId, name, filters }` |
| GET | `/api/instroke/analyses` | Cookie | List user's analyses |
| GET | `/api/instroke/analyses/{id}` | Cookie or public if shared | Analysis metadata + metrics |
| PATCH | `/api/instroke/analyses/{id}` | Cookie | Update name, description, sharing |
| DELETE | `/api/instroke/analyses/{id}` | Cookie | Delete analysis |

**Request body for `POST /analyses`:**

```json
{
  "workoutId": 123,
  "name": "Race - 32 spm",
  "description": "Force curve analysis of 2k final",
  "filters": {
    "spmMin": 31,
    "spmMax": 33,
    "intervalIdx": 2,
    "timeStartS": null,
    "timeEndS": null
  },
  "curveTypes": ["HandleForceCurve"]
}
```

**Response:**

```json
{
  "id": 456,
  "uuid": "a1b2c3d4-...",
  "workoutId": 123,
  "name": "Race - 32 spm",
  "filters": { "spmMin": 31, "spmMax": 33, "intervalIdx": 2 },
  "strokeCount": 42,
  "curveTypes": ["HandleForceCurve"],
  "metrics": {
    "HandleForceCurve": {
      "q1": [0, 12.3, 45.6, ...],
      "q2": [0, 18.2, 67.8, ...],
      "q3": [0, 23.1, 89.4, ...],
      "q4": [0, 28.5, 110.2, ...],
      "mean": [0, 20.5, 78.2, ...],
      "diff": [0, 16.2, 64.6, ...],
      "maxpos": 0.65,
      "minpos": 0.0
    }
  },
  "createdAt": "2026-03-16T12:30:00Z"
}
```

### 4.3 Comparison

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/instroke/compare` | Cookie | Generate comparison data. Body: `{ analysisIds: [...] }` |

**Response:**

```json
{
  "analyses": [
    { "id": 456, "name": "Race 1", "date": "2026-03-15", "color": "#FF6384" },
    { "id": 789, "name": "Race 2", "date": "2026-03-22", "color": "#36A2EB" }
  ],
  "curveType": "HandleForceCurve",
  "abscissa": [0, 0.01, 0.02, ..., 1.0],
  "curves": {
    "456": { "mean": [...], "q1": [...], "q3": [...] },
    "789": { "mean": [...], "q1": [...], "q3": [...] }
  }
}
```

### 4.4 Sharing

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/api/instroke/share` | Cookie | Create share link. Body: `{ analysisIds: [...], expiresInDays: 90 }` |
| GET | `/api/instroke/shared/{uuid}` | Public | Retrieve shared analysis or comparison |

**Response for `POST /share`:**

```json
{
  "uuid": "x7y8z9a0-...",
  "url": "https://rownative.icu/instroke/shared/x7y8z9a0-...",
  "expiresAt": "2026-06-14T12:30:00Z"
}
```

### 4.5 Sensor Configuration (Future)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/instroke/sensors` | Cookie | List sensor configs (custom calibrations, mappings) |
| POST | `/api/instroke/sensors` | Cookie | Save sensor config. Body: `{ sensorType, calibration }` |

---

## 5. Frontend Components

### 5.1 Page Structure

```
site/instroke/
├── index.html           # Activity browser (list OTW workouts from intervals.icu)
├── ingest.html          # Ingestion confirmation page (after selecting activity)
├── analyze.html         # Single workout analysis (filter + chart + save)
├── library.html         # User's saved analyses (list + manage)
├── compare.html         # Multi-analysis comparison (overlay charts)
├── shared.html          # Public shared analysis view (read-only)
├── help.html            # Documentation (FIT format, sensor types, curve interpretation)
├── js/
│   ├── instroke-api.js    # API client (fetch wrapper)
│   ├── charts/
│   │   ├── forcecurve.js  # D3.js force curve chart (adapted from Rowsandall)
│   │   ├── instroke.js    # D3.js in-stroke analysis (adapted from Rowsandall)
│   │   └── compare.js     # D3.js comparison overlay (adapted from Rowsandall)
│   ├── instroke-fit.js    # Client-side FIT parsing (optional, for preview)
│   └── instroke-utils.js  # Filters, normalization, stroke selection
└── css/
    └── instroke.css     # Styles (consistent with existing rownative.icu)
```

### 5.2 Key User Flows

#### Flow 1: First-time analysis

1. User navigates to `/instroke/index.html` (signed in via intervals.icu OAuth)
2. Page fetches `/api/me` to confirm authentication
3. Page fetches `/api/instroke/workouts` to show previously ingested workouts
4. User clicks "Import from intervals.icu" → fetches `/api/me/activities` (OTW rowing, last 3 months)
5. User selects activity → POST `/api/instroke/workouts/ingest` with `activityId`
6. Worker downloads FIT, parses, stores in D1 → returns workout ID
7. Page redirects to `/instroke/analyze.html?workout={id}`
8. Analysis UI loads workout metadata + curve types
9. User adjusts filters (spm sliders, interval dropdown)
10. Page fetches filtered strokes (client-side or via API), renders force curve
11. User names analysis, clicks "Save" → POST `/api/instroke/analyses`
12. Confirmation toast, link to analysis library

#### Flow 2: Comparison

1. User navigates to `/instroke/library.html`
2. Page lists saved analyses (table with checkboxes)
3. User selects 2+ analyses, clicks "Compare"
4. Page redirects to `/instroke/compare.html?ids={id1,id2}`
5. Page fetches `/api/instroke/compare` with selected IDs
6. D3.js renders overlaid force curves with legend (reusing Rowsandall template)
7. User clicks "Share" → POST `/api/instroke/share` → receives shareable URL
8. URL copied to clipboard

#### Flow 3: Shared analysis (public)

1. Recipient opens share link: `/instroke/shared/{uuid}`
2. Page fetches `/api/instroke/shared/{uuid}` (no auth required)
3. Analysis metadata + chart data rendered (read-only)
4. Athlete name shown only if analysis is public; otherwise "Anonymous"

### 5.3 UI Components

**Activity Browser:**
- Table: Date, Distance, Duration, SPM, Sensor, Curve Types, Actions (Analyze, Delete)
- Filters: Date range, sensor type, has force curves?
- "Import from intervals.icu" button

**Analysis Page:**
- Top: Workout metadata (date, distance, spm, sensor)
- Sidebar: Filters (spm sliders, interval dropdown, time range) — **reuse Rowsandall UX**
- Main: D3.js SVG canvas (force curve + quartile shading)
- Bottom: Analysis name input, description textarea, "Save Analysis" button

**Comparison Page:**
- Sidebar: Selected analyses (color legend, remove button)
- Main: Overlaid force curves (D3.js) — **reuse Rowsandall comparison template**
- Controls: Toggle Q1/Q3 shading, normalize abscissa, export as PNG
- Share button

**Chart Features (inherited from Rowsandall):**
- Interactive sliders (SPM, distance, work per stroke)
- Crosshair tooltips (hover for exact values)
- Toggle layers (mean curve, Q1/Q3 band, individual strokes)
- Median force curve (red line) + average force (blue dashed line)
- Export to PNG with custom watermark
- Responsive SVG rendering

---

## 6. Deployment and Operations

### 6.1 Infrastructure (Cloudflare)

**Shared with courses/challenges:**
- Worker: `rownative-worker` (existing)
- KV Namespace: `ROWING_COURSES` (existing)
- D1 Database: `rowing-courses-db` (existing; add new tables via migrations)
- DNS: `rownative.icu` (existing)

**New resources:**
- R2 Bucket (future): `rownative-instroke-fit-cache` (optional; for caching FIT files)

**Estimated costs (Cloudflare free tier sufficient for <100k requests/day):**
- Worker: 100k requests/day free
- D1: 5M reads/day, 100k writes/day free
- KV: 100k reads/day, 1k writes/day free
- R2: 10GB storage, 1M Class A ops/month free

### 6.2 CI/CD Pipeline

**GitHub Actions (extend existing `.github/workflows/`):**

1. `validate-instroke.yml` (new)
   - Trigger: PRs modifying `site/instroke/` or `src/instroke-*.ts`
   - Steps: TypeScript lint, unit tests (Vitest), integration tests

2. `deploy-worker.yml` (existing, extended)
   - Trigger: Push to `main`
   - Steps: Build worker, run migrations, deploy to Cloudflare
   - Add: Run D1 migrations for instroke tables

3. `deploy-pages.yml` (existing, extended)
   - Trigger: Push to `main`
   - Steps: Build static site, deploy to GitHub Pages
   - Add: Copy `site/instroke/` to `_site/instroke/`

### 6.3 Monitoring and Observability

**Cloudflare Analytics:**
- Worker metrics: request rate, latency, errors
- D1 metrics: query latency, storage growth
- KV metrics: cache hit rate

**Application logging:**
- Worker logs (console.log to Cloudflare dashboard)
- Error tracking: Log FIT parsing failures, intervals.icu API errors
- Usage metrics: ingestion rate, analysis creation rate, share link clicks

**Alerting (future):**
- D1 storage >75% of free tier → email admin
- Error rate >5% for 10 minutes → Slack notification

### 6.4 Scaling Considerations

**Current design targets:**
- 1,000 active users
- 10,000 ingested workouts
- 50,000 analyses

**Bottlenecks and mitigations:**

| Bottleneck | Mitigation |
|------------|-----------|
| D1 storage (10GB free tier) | Implement workout/analysis TTL (e.g., auto-delete after 1 year inactivity); add "archive" feature |
| FIT download bandwidth from intervals.icu | Cache FIT files in R2 (optional); rate limit ingestion to 1 workout per 10 seconds per user |
| Curve data blob size in D1 | Compress JSON with gzip; store only downsampled curves (e.g., 64 points max) |
| Analysis computation latency | Precompute metrics on save (not on-demand); use Web Workers in frontend for client-side filtering |

**Paid tier upgrade path (if needed):**
- D1: $5/month for 50GB storage
- R2: $0.015/GB/month for storage beyond 10GB
- Workers: $5/month for 10M requests (beyond 100k/day free tier)

---

## 7. Security and Privacy

### 7.1 Authentication

- **OAuth flow:** Reuse existing intervals.icu OAuth (no changes)
- **Session management:** Encrypted cookie (`rn_session`) with 90-day TTL
- **API authorization:** All `/api/instroke/*` endpoints require valid session except `/shared/{uuid}`

### 7.2 Data Privacy

**Principle:** Respect intervals.icu privacy settings; default to private analyses.

- **Workout ingestion:** Only accessible to workout owner (athlete_id from token)
- **Analyses:** Default `is_public=false`; only owner can view/edit
- **Shared analyses:** UUID-based links; no athlete name revealed unless `is_public=true`
- **Data retention:** User can delete workouts + analyses at any time; cascading delete in D1

### 7.3 Input Validation

**FIT file ingestion:**
- Max file size: 10MB (intervals.icu enforces smaller limits)
- Timeout: 30 seconds for FIT download + parse
- Error handling: Invalid FIT → 422 with human-readable error; log to Cloudflare dashboard

**API inputs:**
- Analysis name: Max 100 characters, no HTML tags (sanitize with DOMPurify or server-side escaping)
- Filters: Validate spm range (0-60), interval index (0-100), time window (0-86400s)

**CORS:**
- Allow `https://rownative.icu` and `http://localhost:*` (for dev)
- Credentials: Include cookies in CORS requests

### 7.4 Rate Limiting

| Endpoint | Limit | Window |
|----------|-------|--------|
| POST `/workouts/ingest` | 10 requests | 10 minutes |
| POST `/analyses` | 50 requests | 1 hour |
| POST `/share` | 20 requests | 1 hour |
| GET `/shared/{uuid}` | 100 requests | 1 minute (per UUID) |

**Implementation:** KV-based rate limiting (reuse existing pattern from courses/challenges).

### 7.5 GDPR Compliance

**Data subject rights:**
- **Access:** GET `/api/instroke/workouts` lists user's data
- **Deletion:** DELETE `/api/instroke/workouts/{id}` and DELETE `/api/instroke/analyses/{id}` (cascading)
- **Portability:** Export analyses as JSON (future feature)

**Legal basis:** Legitimate interest (providing in-stroke analysis service); user consent implied by OAuth authorization.

**Data retention:** No automatic deletion; user-initiated only. Consider adding "auto-archive after 1 year inactivity" in future.

---

## 8. Migration from Rowsandall

### 8.1 Data Export from Rowsandall

**Prerequisites (before Rowsandall shutdown):**
- Add "Export In-Stroke Analyses" button to Rowsandall user dashboard
- Generate ZIP containing:
  - `analyses.json`: Metadata (name, date, filters, curve type)
  - `workouts/{activity_id}.fit`: FIT files with in-stroke data
  - `curves/{analysis_id}.json`: Precomputed metrics (quartiles, mean)

**Rowsandall export format:**

```json
{
  "version": "1.0",
  "exportDate": "2026-04-01T00:00:00Z",
  "athleteId": "123456",
  "analyses": [
    {
      "id": "rowsandall-abc123",
      "name": "2k PR - 32 spm",
      "workoutDate": "2026-03-15",
      "activityId": "i12345678",
      "curveType": "HandleForceCurve",
      "filters": { "spmMin": 31, "spmMax": 33 },
      "strokeCount": 42
    }
  ],
  "workouts": [
    { "activityId": "i12345678", "filename": "workouts/i12345678.fit" }
  ]
}
```

### 8.2 Import to Instroke Platform

**New endpoint:** `POST /api/instroke/import-rowsandall`

**Process:**
1. User uploads ZIP via `/instroke/import.html`
2. Worker unzips, validates `analyses.json` schema
3. For each workout FIT:
   - Ingest via existing `ingest()` flow
   - Map `activity_id` to new `workout_id`
4. For each analysis:
   - Create analysis in D1 with mapped `workout_id`
   - Import precomputed metrics if available
5. Return import summary: `{ success: 10, failed: 2, errors: [...] }`

**UI:** Progress bar, error details, link to analysis library.

### 8.3 Migration Timeline

| Phase | Date | Milestone |
|-------|------|-----------|
| Alpha | May 2026 | Core ingestion + analysis working; internal testing |
| Beta | July 2026 | Public beta; Rowsandall export tool live |
| Migration | Sep-Nov 2026 | User migration period; parallel operation |
| GA | Dec 2026 | Full launch; Rowsandall shutdown |

---

## 9. Future Enhancements

### 9.1 Phase 1 (MVP) — Q2 2026

- [x] Ingestion from intervals.icu
- [x] Force curve analysis (filter by spm/interval)
- [x] Save analyses
- [x] Compare 2 analyses
- [x] Share links (public/private)

### 9.2 Phase 2 — Q3 2026

- [ ] Boat acceleration curves (Quiske)
- [ ] Seat curves (RP3)
- [ ] Oarlock symmetry analysis (dual EmPower)
- [ ] Export analyses to CSV/JSON
- [ ] Trend analysis (compare same workout type over time)

### 9.3 Phase 3 — Q4 2026

- [ ] Smartphone sensor integration (custom app uploads)
- [ ] Community curve library (opt-in sharing of anonymized curves)
- [ ] Machine learning insights (stroke efficiency scoring)
- [ ] Integration with CrewNerd (real-time analysis on water)

### 9.4 Phase 4 — 2027

- [ ] Video overlay (sync force curve with video recording)
- [ ] Coach tools (annotate curves, share with athletes)
- [ ] API for third-party apps (read-only access to public analyses)
- [ ] Mobile-optimized UI (responsive design for tablets)

---

## 10. Open Questions and Decisions Needed

### 10.1 Technical

1. **FIT parsing: Client or server-side?**
   - **Server-side (recommended):** Consistent parsing, validation, caching in D1
   - **Client-side:** Faster preview, no bandwidth cost, but harder to validate

2. **Curve storage: BLOB JSON or separate columns?**
   - **JSON BLOB (recommended):** Flexible, supports variable-length curves
   - **Separate columns:** Faster queries, but limited to fixed-length curves

3. **Abscissa normalization: Always to 0-1, or preserve original?**
   - **Always normalize (recommended):** Simplifies comparison across different sensors
   - **Preserve original:** More accurate, but requires complex alignment logic

4. **R2 for FIT caching: Immediate or Phase 2?**
   - **Phase 2 (recommended):** Start with D1 storage; add R2 if storage limits hit
   - **Immediate:** Better architecture, but adds complexity

### 10.2 Product

5. **Default privacy: Public or private analyses?**
   - **Private (recommended):** User must opt-in to sharing; respects privacy
   - **Public:** Encourage community learning, but may deter casual users

6. **Workout retention: Unlimited or TTL?**
   - **1-year TTL (recommended):** Keep storage under free tier; user can "archive" to extend
   - **Unlimited:** Better UX, but requires paid tier or aggressive cleanup

7. **Sensor calibration: Custom configs or rely on FIT metadata?**
   - **FIT metadata only (recommended):** Simpler; sensors should self-calibrate
   - **Custom configs:** Advanced users can override, but adds UI complexity

### 10.3 Community

8. **Governance: Rowsandall maintainers or new team?**
   - **Recommendation:** Invite Rowsandall community to GitHub; form maintainer team from contributors

9. **Branding: "Instroke by rownative.icu" or separate name?**
   - **Recommendation:** "Instroke Analysis" with rownative.icu branding; keep subdomain for clarity

10. **Monetization: Donations or premium features?**
    - **Recommendation:** Free tier on Cloudflare; optional "Buy me a coffee" for hosting costs; no premium features (keep community-first)

---

## 11. Success Metrics

**MVP Success (3 months post-launch):**
- 500 registered users (intervals.icu OAuth)
- 2,000 ingested workouts
- 5,000 saved analyses
- 500 shared links generated
- <100ms p50 latency for analysis creation
- <5% error rate on FIT ingestion

**Long-term Success (12 months):**
- 2,000 active users (1+ analysis per month)
- 20,000 ingested workouts
- 50,000 saved analyses
- 5,000 community shared analyses
- Integration with 3+ sensor platforms (Empower, Quiske, RP3)
- <1% Cloudflare free tier quota usage (sustainable scaling)

---

## 12. Conclusion and Next Steps

### 12.1 Recommendation

**Proceed with Option A (Extend rownative/worker and courses):**

- Leverage existing infrastructure, authentication, and community
- Add `/api/instroke/*` routes to worker
- Add `site/instroke/` UI to static site
- Minimize operational overhead (one deployment, one domain)
- Enable natural cross-linking (e.g., "Analyze force curves from this course time")

### 12.2 Immediate Next Steps

1. **Gather feedback:** Share this proposal with Rowsandall community (GitHub discussion)
2. **Prototype:** Build minimal FIT ingestion + force curve charting (1-2 weeks)
3. **Validate intervals.icu API:** Confirm FIT file download endpoint, permissions, rate limits
4. **D1 schema:** Write migration scripts for `instroke_*` tables
5. **UI wireframes:** Sketch analysis page, comparison page (Figma or paper)

### 12.3 Team and Contributions

**Looking for contributors:**
- **TypeScript developer:** Worker API + FIT parsing
- **Frontend developer:** D3.js template adaptation + UI polish (extracting from Rowsandall)
- **Designer:** UI/UX review, branding, help documentation
- **Data scientist:** Curve normalization algorithms, outlier detection
- **Tester:** Device testing (iOS/Android), sensor compatibility

**Communication:**
- GitHub Discussions for design decisions
- Discord or Slack for async coordination (optional)
- Monthly community calls for progress updates

---

**Prepared by:** AI Assistant (Cursor)  
**For:** Sander Roosendaal (rowsandall maintainer)  
**Feedback:** Open GitHub issue in the rownative repositories
