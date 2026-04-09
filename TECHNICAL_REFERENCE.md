# In-Stroke Analysis Platform — Technical Reference

> Quick reference for developers implementing the platform

**Date:** April 9, 2026  
**For:** Contributors to rownative/worker and rownative/courses

---

## Technology Stack

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| **Runtime** | Cloudflare Workers | Latest | Serverless API backend |
| **Language** | TypeScript | 5.5+ | Type-safe server code |
| **Database** | Cloudflare D1 | Latest | SQLite for relational data |
| **Cache** | Cloudflare KV | Latest | Key-value store for sessions, shares |
| **Storage** | Cloudflare R2 | Latest | (Future) FIT file caching |
| **Frontend** | HTML5 + Vanilla JS | ES2020+ | Zero build step |
| **Charting** | **D3.js** | 7.x | Force curve visualization (reuse Rowsandall templates) |
| **FIT Parsing** | fit-file-parser | Latest | Parse Garmin FIT files |
| **Testing** | Vitest | 3.x | Unit + integration tests |
| **Deployment** | Wrangler | 4.x | Cloudflare CLI |

**Note on D3.js:** We reuse the proven D3.js charting code from [Rowsandall chart server](https://git.wereldraadsel.tech/sander/rowsandall-charts), extracting the templates and adapting them for client-side rendering (no Go server needed).

---

## Repository Structure (Extended)

```
rownative/worker/
├── src/
│   ├── index.ts                    # Main router (existing)
│   ├── intervals-api.ts            # intervals.icu client (existing)
│   ├── instroke-ingest.ts          # NEW: FIT parsing + storage
│   ├── instroke-analysis.ts        # NEW: Analysis engine
│   ├── instroke-compare.ts         # NEW: Comparison logic
│   ├── instroke-share.ts           # NEW: Shareable links
│   └── env.d.ts                    # TypeScript environment types
├── test/
│   ├── instroke-ingest.test.ts     # NEW: Ingestion tests
│   ├── instroke-analysis.test.ts   # NEW: Analysis tests
│   └── fixtures/
│       └── sample-empower.fit      # NEW: Test FIT files
├── migrations/
│   └── 0004_instroke_tables.sql    # NEW: D1 schema
└── package.json

rownative/courses/
└── site/
    ├── instroke/                   # NEW: In-stroke UI
    │   ├── index.html              # Activity browser
    │   ├── analyze.html            # Single workout analysis
    │   ├── compare.html            # Multi-analysis comparison
    │   ├── library.html            # Saved analyses
    │   ├── shared.html             # Public shared view
    │   ├── help.html               # Documentation
    │   ├── js/
    │   │   ├── instroke-api.js     # API client
    │   │   ├── charts/             # D3.js charts (adapted from Rowsandall)
    │   │   │   ├── forcecurve.js   # Force curve with sliders
    │   │   │   ├── instroke.js     # In-stroke analysis
    │   │   │   └── compare.js      # Multi-analysis comparison
    │   │   └── instroke-utils.js   # Filters, normalization
    │   └── css/
    │       └── instroke.css        # Styles
    └── index.html, challenges.html, ... (existing)
```

---

## API Endpoints

### Ingestion

```
GET  /api/instroke/workouts
POST /api/instroke/workouts/ingest
GET  /api/instroke/workouts/{id}
DELETE /api/instroke/workouts/{id}
```

**Example: Ingest workout**

```bash
curl -X POST https://rownative.icu/api/instroke/workouts/ingest \
  -H "Cookie: rn_session=..." \
  -H "Content-Type: application/json" \
  -d '{"activityId": "i12345678"}'
```

**Response:**

```json
{
  "id": 123,
  "activityId": "i12345678",
  "strokeCount": 855,
  "curveTypes": ["HandleForceCurve"],
  "sensorType": "Empower"
}
```

### Analysis

```
POST   /api/instroke/analyses
GET    /api/instroke/analyses
GET    /api/instroke/analyses/{id}
PATCH  /api/instroke/analyses/{id}
DELETE /api/instroke/analyses/{id}
```

**Example: Create analysis**

```bash
curl -X POST https://rownative.icu/api/instroke/analyses \
  -H "Cookie: rn_session=..." \
  -H "Content-Type: application/json" \
  -d '{
    "workoutId": 123,
    "name": "Race - 32 spm",
    "filters": {
      "spmMin": 31,
      "spmMax": 33
    },
    "curveTypes": ["HandleForceCurve"]
  }'
```

**Response:**

```json
{
  "id": 456,
  "uuid": "a1b2c3d4-...",
  "name": "Race - 32 spm",
  "strokeCount": 42,
  "metrics": {
    "HandleForceCurve": {
      "mean": [0, 12.5, 45.3, ...],
      "q1": [0, 8.2, 32.1, ...],
      "q3": [0, 16.8, 58.4, ...]
    }
  }
}
```

### Comparison

```
POST /api/instroke/compare
```

**Example: Compare analyses**

```bash
curl -X POST https://rownative.icu/api/instroke/compare \
  -H "Cookie: rn_session=..." \
  -H "Content-Type: application/json" \
  -d '{"analysisIds": [456, 789]}'
```

**Response:**

```json
{
  "analyses": [
    { "id": 456, "name": "Race 1", "color": "#FF6384" },
    { "id": 789, "name": "Race 2", "color": "#36A2EB" }
  ],
  "curveType": "HandleForceCurve",
  "abscissa": [0, 0.01, 0.02, ..., 1.0],
  "curves": {
    "456": { "mean": [...], "q1": [...], "q3": [...] },
    "789": { "mean": [...], "q1": [...], "q3": [...] }
  }
}
```

### Sharing

```
POST /api/instroke/share
GET  /api/instroke/shared/{uuid}
```

**Example: Create share link**

```bash
curl -X POST https://rownative.icu/api/instroke/share \
  -H "Cookie: rn_session=..." \
  -H "Content-Type: application/json" \
  -d '{
    "analysisIds": [456],
    "expiresInDays": 90
  }'
```

**Response:**

```json
{
  "uuid": "x7y8z9a0-...",
  "url": "https://rownative.icu/instroke/shared/x7y8z9a0-...",
  "expiresAt": "2026-06-14T12:30:00Z"
}
```

---

## Database Schema (D1)

### `instroke_workouts`

```sql
CREATE TABLE instroke_workouts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  athlete_id TEXT NOT NULL,
  activity_id TEXT NOT NULL UNIQUE,
  activity_date DATE NOT NULL,
  sport TEXT NOT NULL,  -- 'Rowing' or 'IndoorRowing'
  duration_s INTEGER NOT NULL,
  distance_m INTEGER NOT NULL,
  avg_spm REAL NOT NULL,
  stroke_count INTEGER NOT NULL,
  sensor_type TEXT,  -- 'Empower', 'Quiske', 'RP3', 'Unknown'
  has_force_curve BOOLEAN DEFAULT FALSE,
  has_boat_accel_curve BOOLEAN DEFAULT FALSE,
  has_seat_curve BOOLEAN DEFAULT FALSE,
  has_oarlock_data BOOLEAN DEFAULT FALSE,
  abscissa_type TEXT,  -- 'TIME_UNIFORM_MS', 'HANDLE_DISTANCE_UNIFORM_M', etc.
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_workouts_athlete_date ON instroke_workouts(athlete_id, activity_date DESC);
```

### `instroke_curves`

```sql
CREATE TABLE instroke_curves (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  workout_id INTEGER NOT NULL,
  stroke_number INTEGER NOT NULL,  -- 1-indexed
  spm REAL NOT NULL,
  interval_idx INTEGER,
  curve_type TEXT NOT NULL,  -- 'HandleForceCurve', 'BoatAcceleratorCurve', etc.
  abscissa_type TEXT,
  point_count INTEGER NOT NULL,
  curve_data BLOB NOT NULL,  -- JSON array: [0, 12.5, 45.3, ...]
  
  -- Oarlock scalars
  catch_angle REAL,
  finish_angle REAL,
  
  -- Stroke metrics
  drive_time_ms INTEGER,
  recovery_time_ms INTEGER,
  drive_length_m REAL,
  peak_force_n REAL,
  avg_force_n REAL,
  
  FOREIGN KEY (workout_id) REFERENCES instroke_workouts(id) ON DELETE CASCADE
);

CREATE INDEX idx_curves_workout ON instroke_curves(workout_id, stroke_number);
CREATE INDEX idx_curves_type ON instroke_curves(workout_id, curve_type);
```

### `instroke_analyses`

```sql
CREATE TABLE instroke_analyses (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  uuid TEXT NOT NULL UNIQUE,
  athlete_id TEXT NOT NULL,
  workout_id INTEGER NOT NULL,
  name TEXT NOT NULL,
  description TEXT,
  
  -- Filters
  filter_spm_min REAL,
  filter_spm_max REAL,
  filter_interval_idx INTEGER,
  filter_time_start_s INTEGER,
  filter_time_end_s INTEGER,
  
  stroke_count INTEGER NOT NULL,
  curve_types TEXT NOT NULL,  -- Comma-separated: 'HandleForceCurve,BoatAcceleratorCurve'
  is_public BOOLEAN DEFAULT FALSE,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  
  FOREIGN KEY (workout_id) REFERENCES instroke_workouts(id) ON DELETE CASCADE
);

CREATE INDEX idx_analyses_athlete ON instroke_analyses(athlete_id, created_at DESC);
CREATE UNIQUE INDEX idx_analyses_uuid ON instroke_analyses(uuid);
```

### `instroke_analysis_metrics`

```sql
CREATE TABLE instroke_analysis_metrics (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  analysis_id INTEGER NOT NULL,
  curve_type TEXT NOT NULL,
  metric_name TEXT NOT NULL,  -- 'q1', 'q2', 'q3', 'q4', 'mean', 'diff', 'maxpos', 'minpos'
  metric_data BLOB NOT NULL,  -- JSON array: [0, 12.5, 45.3, ...]
  
  FOREIGN KEY (analysis_id) REFERENCES instroke_analyses(id) ON DELETE CASCADE
);

CREATE UNIQUE INDEX idx_metrics_analysis_curve_metric 
  ON instroke_analysis_metrics(analysis_id, curve_type, metric_name);
```

---

## FIT Parsing

### Developer Fields (rowingdata spec)

| Field ID | FIT Name | Type | Scale | Units | Source Column |
|----------|----------|------|-------|-------|---------------|
| 0 | DriveLength | UINT16 | 100 | m | DriveLength (meters) |
| 1 | StrokeDriveTime | UINT16 | 1 | ms | DriveTime (ms) |
| 2 | DragFactor | UINT16 | 1 | — | DragFactor |
| 3 | StrokeRecoveryTime | UINT16 | 1 | ms | StrokeRecoveryTime (ms) |
| 6 | AverageDriveForceN | UINT16 | 10 | N | AverageDriveForce (N) |
| 7 | PeakDriveForceN | UINT16 | 10 | N | PeakDriveForce (N) |
| 11 | Catch | SINT16 | 10 | deg | catch, catchAngle |
| 12 | Finish | SINT16 | 10 | deg | finish, finishAngle |
| 13 | Slip | SINT16 | 10 | deg | slip |
| 14 | Wash | SINT16 | 10 | deg | wash |

**In-stroke curves (dynamic IDs, default start at 60):**

- HandleForceCurve
- BoatAcceleratorCurve
- OarAngleVelocityCurve
- SeatCurve

**Abscissa metadata (IDs 90-92):**

- InstrokeAbscissaType (UINT8): 0=UNKNOWN, 1=TIME_UNIFORM_MS, 2=HANDLE_DISTANCE_UNIFORM_M, etc.
- InstrokeSampleInterval (UINT16): Spacing between samples (interpretation depends on type)
- InstrokePointCount (UINT8): Number of points in each curve array

### Parsing Example (TypeScript)

```typescript
import FitParser from 'fit-file-parser';

interface WorkoutRecord {
  activityId: string;
  sport: string;
  strokes: StrokeRecord[];
}

interface StrokeRecord {
  strokeNumber: number;
  spm: number;
  intervalIdx: number;
  curves: {
    HandleForceCurve?: number[];
    BoatAcceleratorCurve?: number[];
  };
  oarlock: {
    catchAngle?: number;
    finishAngle?: number;
  };
  metrics: {
    driveTimeMs?: number;
    recoveryTimeMs?: number;
    driveLengthM?: number;
    peakForceN?: number;
  };
}

async function parseFIT(fitBuffer: ArrayBuffer): Promise<WorkoutRecord> {
  const parser = new FitParser();
  const fit = parser.parse(fitBuffer);
  
  const strokes: StrokeRecord[] = [];
  for (const record of fit.records) {
    const stroke: StrokeRecord = {
      strokeNumber: record.total_cycles || strokes.length + 1,
      spm: record.cadence || 0,
      intervalIdx: record.lap_index || 0,
      curves: {},
      oarlock: {},
      metrics: {},
    };
    
    // Extract developer fields
    if (record.developer_fields) {
      for (const field of record.developer_fields) {
        if (field.field_name === 'HandleForceCurve') {
          stroke.curves.HandleForceCurve = field.value;  // Array
        } else if (field.field_name === 'Catch') {
          stroke.oarlock.catchAngle = field.value / 10;  // Scale 10
        } else if (field.field_name === 'StrokeDriveTime') {
          stroke.metrics.driveTimeMs = field.value;
        }
        // ... more fields
      }
    }
    
    strokes.push(stroke);
  }
  
  return {
    activityId: fit.activity.id,
    sport: fit.activity.sport,
    strokes,
  };
}
```

---

## Frontend Components

### D3.js Charts (Adapted from Rowsandall)

**Source:** [rowsandall-charts](https://git.wereldraadsel.tech/sander/rowsandall-charts) — MIT licensed, reusable

```javascript
// js/charts/instroke.js (adapted from Rowsandall template)

/**
 * Render in-stroke analysis with quartile shading and interactive filtering
 * Adapted from Rowsandall's instroke.js template
 */
function renderInstrokeChart(containerId, analysisData) {
  const margin = { top: 30, right: 60, bottom: 40, left: 60 };
  const containerWidth = document.getElementById(containerId).offsetWidth;
  const width = containerWidth - margin.left - margin.right;
  const height = width / 1.618 - margin.top - margin.bottom;  // Golden ratio
  
  // Create SVG
  const svg = d3.select(`#${containerId}`)
    .append("svg")
    .attr("id", "instroke_svg")
    .attr("width", width + margin.left + margin.right)
    .attr("height", height + margin.top + margin.bottom)
    .append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);
  
  // Scales
  const xScale = d3.scaleLinear()
    .domain([0, 100])  // Normalized 0-100%
    .range([0, width]);
  
  const yScale = d3.scaleLinear()
    .domain([
      d3.min(analysisData.data, d => d.low),
      d3.max(analysisData.data, d => d.high)
    ])
    .range([height, 0]);
  
  // Axes
  svg.append("g")
    .attr("transform", `translate(0,${height})`)
    .call(d3.axisBottom(xScale));
  
  svg.append("g")
    .call(d3.axisLeft(yScale));
  
  // Quartile shading (Q1-Q3 band)
  const area = d3.area()
    .x(d => xScale(100 * d.x / d3.max(analysisData.data, dd => dd.x)))
    .y0(d => yScale(d.low))   // Q1
    .y1(d => yScale(d.high));  // Q3
  
  svg.append("path")
    .datum(analysisData.data)
    .attr("fill", "lightgray")
    .attr("d", area);
  
  // Median curve (black line)
  const lineMedian = d3.line()
    .x(d => xScale(100 * d.x / d3.max(analysisData.data, dd => dd.x)))
    .y(d => yScale(d.median));
  
  svg.append("path")
    .datum(analysisData.data)
    .attr("fill", "none")
    .attr("stroke", "black")
    .attr("stroke-width", 2.0)
    .attr("d", lineMedian);
  
  // Axis labels
  svg.append("text")
    .attr("transform", "rotate(-90)")
    .attr("y", 0 - margin.left)
    .attr("x", 0 - (height / 2))
    .attr("dy", "1em")
    .style("text-anchor", "middle")
    .text(analysisData.ytitle || "Force (N)");
  
  svg.append("text")
    .attr("transform", `translate(${width / 2},${height + margin.top})`)
    .style("text-anchor", "middle")
    .text("Normalized Drive Phase (%)");
  
  // Add crosshairs (hover tooltips)
  addCrosshairs(svg, xScale, yScale, width, height);
  
  return svg;
}

/**
 * Add interactive crosshairs (from Rowsandall)
 */
function addCrosshairs(svg, xScale, yScale, width, height) {
  const focus = svg.append("g")
    .attr("class", "focus")
    .style("display", "none");
  
  focus.append("line")
    .attr("class", "x-crosshair")
    .attr("y1", 0)
    .attr("y2", height);
  
  focus.append("line")
    .attr("class", "y-crosshair")
    .attr("x1", 0)
    .attr("x2", width);
  
  focus.append("text")
    .attr("class", "x-label")
    .attr("text-anchor", "middle")
    .attr("dy", "-0.5em");
  
  focus.append("text")
    .attr("class", "y-label")
    .attr("text-anchor", "middle")
    .attr("dy", "1em");
  
  svg.append("rect")
    .attr("width", width)
    .attr("height", height)
    .style("fill", "none")
    .style("pointer-events", "all")
    .on("mouseover", () => focus.style("display", null))
    .on("mouseout", () => focus.style("display", "none"))
    .on("mousemove", function(event) {
      const [mx, my] = d3.pointer(event);
      const x0 = xScale.invert(mx);
      const y0 = yScale.invert(my);
      
      focus.select(".x-crosshair").attr("transform", `translate(${xScale(x0)},0)`);
      focus.select(".x-label").attr("transform", `translate(${xScale(x0)},${height})`).text(x0.toFixed(2));
      focus.select(".y-crosshair").attr("transform", `translate(0,${yScale(y0)})`);
      focus.select(".y-label").attr("transform", `translate(20,${yScale(y0)})`).text(y0.toFixed(3));
    });
}

/**
 * Interactive filtering with sliders (from Rowsandall force curve)
 */
function setupFilters(data, updateCallback) {
  let filterData = data;
  let minSPM = 15, maxSPM = 60;
  let minDist = 0, maxDist = d3.max(data, d => d.cumdist);
  let minWork = 0, maxWork = 1500;
  
  d3.selectAll(".slider").on("input", function() {
    minSPM = +d3.select("#minSPM").property("value");
    maxSPM = +d3.select("#maxSPM").property("value");
    minDist = +d3.select("#minDist").property("value");
    maxDist = +d3.select("#maxDist").property("value");
    minWork = +d3.select("#minWork").property("value");
    maxWork = +d3.select("#maxWork").property("value");
    
    // Filter strokes
    filterData = data.filter(d => 
      d.spm >= minSPM && d.spm <= maxSPM &&
      d.driveenergy >= minWork && d.driveenergy <= maxWork &&
      d.cumdist >= minDist && d.cumdist <= maxDist
    );
    
    // Update chart with filtered data
    updateCallback(filterData);
    
    // Update slider value displays
    d3.select("#minSPMValue").text(minSPM);
    d3.select("#maxSPMValue").text(maxSPM);
    d3.select("#minDistValue").text(minDist);
    d3.select("#maxDistValue").text(maxDist);
  });
}
```

### API Client

```javascript
// instroke-api.js

const API_BASE = '/api/instroke';

async function ingestWorkout(activityId) {
  const response = await fetch(`${API_BASE}/workouts/ingest`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',  // Include session cookie
    body: JSON.stringify({ activityId }),
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Ingestion failed');
  }
  
  return response.json();
}

async function createAnalysis(workoutId, name, filters) {
  const response = await fetch(`${API_BASE}/analyses`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',
    body: JSON.stringify({ workoutId, name, filters, curveTypes: ['HandleForceCurve'] }),
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Analysis creation failed');
  }
  
  return response.json();
}

async function compareAnalyses(analysisIds) {
  const response = await fetch(`${API_BASE}/compare`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    credentials: 'include',
    body: JSON.stringify({ analysisIds }),
  });
  
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error || 'Comparison failed');
  }
  
  return response.json();
}
```

---

## Testing

### Unit Tests (Vitest)

```typescript
// test/instroke-ingest.test.ts

import { describe, it, expect, beforeAll } from 'vitest';
import { ingestWorkout } from '../src/instroke-ingest';
import * as fs from 'fs';

describe('FIT Ingestion', () => {
  let sampleFIT: ArrayBuffer;
  
  beforeAll(() => {
    sampleFIT = fs.readFileSync('./test/fixtures/sample-empower.fit');
  });
  
  it('should parse Empower FIT file', async () => {
    const result = await parseFIT(sampleFIT);
    
    expect(result.strokes.length).toBeGreaterThan(0);
    expect(result.strokes[0].curves.HandleForceCurve).toBeDefined();
    expect(result.strokes[0].oarlock.catchAngle).toBeTypeOf('number');
  });
  
  it('should handle missing in-stroke data gracefully', async () => {
    const minimalFIT = fs.readFileSync('./test/fixtures/minimal.fit');
    const result = await parseFIT(minimalFIT);
    
    expect(result.strokes.length).toBeGreaterThan(0);
    expect(result.strokes[0].curves.HandleForceCurve).toBeUndefined();
  });
});
```

### Integration Tests

```typescript
// test/instroke-integration.test.ts

import { describe, it, expect } from 'vitest';
import { unstable_dev } from 'wrangler';

describe('Instroke API (integration)', () => {
  let worker;
  
  beforeAll(async () => {
    worker = await unstable_dev('src/index.ts', {
      experimental: { disableExperimentalWarning: true },
    });
  });
  
  afterAll(async () => {
    await worker.stop();
  });
  
  it('GET /api/instroke/workouts requires auth', async () => {
    const response = await worker.fetch('/api/instroke/workouts');
    expect(response.status).toBe(401);
  });
  
  it('POST /api/instroke/workouts/ingest with valid session', async () => {
    const sessionCookie = 'rn_session=...';  // Mock session
    const response = await worker.fetch('/api/instroke/workouts/ingest', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Cookie': sessionCookie,
      },
      body: JSON.stringify({ activityId: 'i12345678' }),
    });
    
    expect(response.status).toBe(200);
    const data = await response.json();
    expect(data.id).toBeDefined();
  });
});
```

---

## Deployment

### Local Development

```bash
# Clone repos
git clone https://github.com/rownative/worker.git
git clone https://github.com/rownative/courses.git

# Install dependencies
cd worker
npm install

# Set up secrets (.dev.vars)
cp .dev.vars.example .dev.vars
# Edit .dev.vars: INTERVALS_CLIENT_ID, INTERVALS_CLIENT_SECRET, TOKEN_ENCRYPTION_KEY

# Run migrations (D1 local)
npx wrangler d1 migrations apply rowing-courses-db --local

# Start worker (with watch mode)
npm run dev

# In another terminal, serve static site
cd ../courses
python3 -m http.server 8000  # or: npm run dev
```

**Access:**
- Worker: `http://localhost:8787`
- Static site: `http://localhost:8000?debug=1&api=http://localhost:8787/api`

### Production Deployment

```bash
# Deploy worker
cd worker
npm run deploy  # Runs: wrangler deploy

# Deploy static site (GitHub Pages)
cd ../courses
git push origin main  # GitHub Actions auto-deploys to Pages
```

**Migrations (production):**

```bash
npx wrangler d1 migrations apply rowing-courses-db --remote
```

---

## Performance Guidelines

### Worker

- **Cold start:** <50ms (TypeScript compiled to JS by Wrangler)
- **FIT parsing:** <2s for 30-minute workout (855 strokes)
- **Analysis creation:** <5s (filter + compute quartiles)
- **Comparison:** <1s (load 5 analyses from D1)

**Optimization tips:**
- Cache parsed FIT data in D1 (avoid re-parsing)
- Use D1 indexes on `athlete_id`, `workout_id`
- Limit curve resolution (max 127 points per curve in FIT)

### Frontend

- **Initial load:** <2s on 3G (target Lighthouse score >90)
- **Chart rendering:** <500ms for 100 strokes
- **Comparison page:** <3s for 5 analyses

**Optimization tips:**
- Downsample large curves for display (D3.js handles this natively)
- Lazy-load charts (only render when visible)
- Debounce filter changes (wait 300ms before re-rendering)

---

## Security Checklist

- [ ] All `/api/instroke/*` endpoints require valid session (except `/shared/{uuid}`)
- [ ] Rate limiting: 10 ingestions per 10 minutes per user
- [ ] Input validation: Sanitize analysis names (max 100 chars, no HTML)
- [ ] CORS: Allow `https://rownative.icu` and `http://localhost:*` only
- [ ] SQL injection: Use parameterized queries (D1 bindings)
- [ ] XSS: Escape user-provided names in HTML (use `textContent`, not `innerHTML`)
- [ ] CSRF: State parameter in OAuth flow (already implemented)

---

## Common Issues and Solutions

### Issue: FIT parsing fails with "Unknown developer field"

**Cause:** FIT file uses developer fields not in rowingdata spec.

**Solution:**
1. Log unknown fields for debugging
2. Gracefully skip unknown fields (don't fail entire parse)
3. File issue in rowingdata repo to add new field

### Issue: Chart renders blank

**Cause:** Curve data format mismatch (expecting array, got object).

**Solution:**
1. Validate `metric_data` is JSON array in D1
2. Check D3.js console errors (invalid data binding)

### Issue: Session cookie not set after OAuth

**Cause:** Cookie SameSite attribute or domain mismatch.

**Solution:**
1. Verify `SameSite=Lax` and `Secure` flags
2. Check `Set-Cookie` header in browser DevTools
3. Ensure OAuth callback URL matches worker origin

---

## Contribution Workflow

1. **Fork** rownative/worker and rownative/courses
2. **Branch:** `git checkout -b feature/instroke-comparison`
3. **Develop:** Implement feature with tests
4. **Test locally:** `npm test && npm run dev`
5. **Commit:** `git commit -m "feat: add multi-analysis comparison"`
6. **Push:** `git push origin feature/instroke-comparison`
7. **PR:** Open pull request to `main` branch
8. **Review:** Maintainer reviews and merges

**Commit conventions:**
- `feat:` New feature
- `fix:` Bug fix
- `docs:` Documentation only
- `test:` Add or update tests
- `refactor:` Code refactor (no behavior change)

---

## Key Dependencies

```json
{
  "dependencies": {
    "fit-file-parser": "^2.x",
    "fflate": "^0.8.2"
  },
  "devDependencies": {
    "@cloudflare/vitest-pool-workers": "^0.12.4",
    "vitest": "^3.2.0",
    "wrangler": "^4.74.0",
    "typescript": "^5.5.2"
  }
}
```

**Frontend (CDN):**
- D3.js: `https://d3js.org/d3.v7.min.js`
- jQuery (for Rowsandall template compatibility): `https://code.jquery.com/jquery-3.7.1.min.js`

**Note:** Rowsandall templates use jQuery for DOM manipulation and event handling. We can optionally refactor to vanilla JS in Phase 2.

---

## Monitoring and Observability

### Cloudflare Dashboard

**Metrics to monitor:**
- Request rate: `/api/instroke/*` (target <100k/day)
- Latency: p50, p95, p99 (target p95 <1s)
- Error rate: 4xx, 5xx (target <5%)
- D1 storage: (target <10GB)

**Alerts:**
- Error rate >10% for 5 minutes
- D1 storage >8GB (80% of free tier)

### Application Logging

```typescript
// src/instroke-ingest.ts

export async function ingestWorkout(env: Env, activityId: string, athleteId: string) {
  console.log(`[Ingestion] Start: activityId=${activityId}, athleteId=${athleteId}`);
  
  try {
    const fit = await downloadFIT(env, activityId);
    const workout = await parseFIT(fit);
    await storeWorkout(env.DB, workout);
    
    console.log(`[Ingestion] Success: workoutId=${workout.id}, strokes=${workout.strokeCount}`);
    return workout;
  } catch (error) {
    console.error(`[Ingestion] Error: activityId=${activityId}`, error);
    throw error;
  }
}
```

---

## Resources

- **Cloudflare Workers Docs:** https://developers.cloudflare.com/workers/
- **D1 Documentation:** https://developers.cloudflare.com/d1/
- **D3.js Docs:** https://d3js.org/
- **Rowsandall Charts (source code):** https://git.wereldraadsel.tech/sander/rowsandall-charts
- **rowingdata FIT_EXPORT:** https://github.com/sanderroosendaal/rowingdata/blob/develop/docs/FIT_EXPORT.md
- **intervals.icu API:** https://intervals.icu/api/v1/docs/swagger-ui/index.html

---

**Prepared by:** AI Assistant (Cursor)  
**For:** Contributors to rownative/worker and rownative/courses  
**Last Updated:** April 9, 2026
