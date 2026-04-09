# D3.js Code Reuse from Rowsandall Chart Server

> Guide for adapting Rowsandall's D3.js templates for client-side rendering

**Date:** April 9, 2026  
**Source:** https://git.wereldraadsel.tech/sander/rowsandall-charts (MIT licensed)

---

## Overview

The Rowsandall chart server uses Go templates to generate D3.js code server-side. We can **reuse these proven D3.js templates** by extracting them and adapting for client-side rendering, eliminating the need for a separate Go chart server.

**Benefits:**
- ✅ Proven UX (users familiar with Rowsandall interface)
- ✅ Interactive features already implemented (sliders, filtering, crosshairs)
- ✅ Smooth migration path (minimal relearning for users)
- ✅ No additional server infrastructure (client-side rendering)
- ✅ Less development time (adapt vs build from scratch)

---

## Rowsandall Architecture (Current)

```
┌──────────────────────────────────────────────────────────┐
│ Django App (Python)                                       │
│ - Prepares data from PostgreSQL                          │
│ - POST to chart server with JSON payload                 │
└────────────────┬─────────────────────────────────────────┘
                 │ POST https://charts.rowsandall.com/forcecurve
                 │ Body: { "data": [...], "title": "...", ... }
                 │ Authorization: Bearer <token>
┌────────────────▼─────────────────────────────────────────┐
│ Go Chart Server (Port 3000)                              │
│ - Receives JSON data                                     │
│ - Loads Go template: templates/instroke.js              │
│ - Injects data: {{ .Data }} → JSON                      │
│ - Minifies JavaScript with jsmin                        │
│ - Returns: { "script": "...", "div": "..." }            │
└────────────────┬─────────────────────────────────────────┘
                 │ Response: { "script": "d3.select(...)", "div": "<div id='...'/>" }
                 │
┌────────────────▼─────────────────────────────────────────┐
│ Browser                                                   │
│ - Django embeds script in page                           │
│ - D3.js v7 renders SVG chart                             │
│ - User interacts with sliders, crosshairs                │
└──────────────────────────────────────────────────────────┘
```

**Challenges:**
- Two services to deploy (Django + Go)
- Extra latency (HTTP round-trip to chart server)
- More operational complexity

---

## New Architecture (Client-Side D3.js)

```
┌──────────────────────────────────────────────────────────┐
│ Cloudflare Worker API                                    │
│ - GET /api/instroke/analyses/{id}                        │
│ - Returns: JSON { data: [...], metrics: {...}, ... }    │
└────────────────┬─────────────────────────────────────────┘
                 │ JSON response
                 │
┌────────────────▼─────────────────────────────────────────┐
│ Browser (rownative.icu/instroke/analyze.html)           │
│ 1. Fetch analysis data from API                          │
│ 2. Load D3.js chart code (instroke.js)                   │
│ 3. Inject data: window.CHART_DATA = apiResponse         │
│ 4. renderInstrokeChart('container', window.CHART_DATA)  │
│ 5. D3.js v7 renders SVG chart                            │
│ 6. User interacts with sliders, crosshairs               │
└──────────────────────────────────────────────────────────┘
```

**Benefits:**
- One service (Worker API)
- Lower latency (no extra HTTP hop)
- Simpler deployment (static files on GitHub Pages)

---

## Migration Steps

### Step 1: Extract D3.js Templates

**Source files from rowsandall-charts:**

| File | Purpose | New Location |
|------|---------|--------------|
| `templates/instroke.js` | In-stroke analysis (quartile shading, median curve) | `site/instroke/js/charts/instroke.js` |
| `templates/instroke_compare.js` | Multi-analysis comparison (overlay curves) | `site/instroke/js/charts/compare.js` |
| `templates/forcecurve.js` | Force curve analysis (interactive sliders) | `site/instroke/js/charts/forcecurve.js` |
| `templates/base_template.js` | Common D3.js utilities (if exists) | `site/instroke/js/charts/common.js` |

### Step 2: Remove Go Template Syntax

**Before (Go template):**
```javascript
{{ define "chart" }}
const data = {{ .Data }}
const title = {{ .Title }}
const ytitle = {{ .Ytitle }}
var individual_curves = {{ .IndividualCurves }}
{{ end }}
```

**After (Pure JavaScript):**
```javascript
/**
 * Render instroke analysis chart
 * @param {string} containerId - DOM element ID
 * @param {object} chartData - { data, title, ytitle, individualCurves }
 */
function renderInstrokeChart(containerId, chartData) {
  const data = chartData.data;
  const title = chartData.title;
  const ytitle = chartData.ytitle;
  const individualCurves = chartData.individualCurves;
  
  // Rest of D3.js code stays the same...
}
```

### Step 3: Data Injection Pattern

**HTML page structure:**
```html
<div id="instroke_viz"></div>

<script src="https://d3js.org/d3.v7.min.js"></script>
<script src="/instroke/js/charts/instroke.js"></script>
<script>
  // Fetch analysis data from API
  fetch('/api/instroke/analyses/456')
    .then(res => res.json())
    .then(analysis => {
      // Inject data and render
      renderInstrokeChart('instroke_viz', {
        data: analysis.metrics.HandleForceCurve.data,
        title: analysis.name,
        ytitle: 'Force (N)',
        individualCurves: false,
        spmMin: analysis.filters.spmMin,
        spmMax: analysis.filters.spmMax,
      });
    });
</script>
```

### Step 4: jQuery Dependency (Optional Refactor)

Rowsandall templates use jQuery for DOM manipulation:

```javascript
// Rowsandall (jQuery):
$("#minSPM").val();
$("#data_viz").width();
```

**Options:**
1. **Keep jQuery** (faster migration, 30KB overhead)
2. **Refactor to vanilla JS** (cleaner, no dependencies)

**Recommendation:** Keep jQuery for MVP, refactor in Phase 2.

```javascript
// Vanilla JS equivalent:
document.getElementById("minSPM").value;
document.getElementById("data_viz").offsetWidth;
```

---

## Code Reuse Checklist

### Force Curve (`forcecurve.js`)

**Features to migrate:**
- [x] Interactive sliders (SPM, distance, work per stroke)
- [x] Median force curve (red line)
- [x] Average force (blue dashed line)
- [x] Individual stroke curves (thin gray lines, toggle)
- [x] Peak force circles (toggle)
- [x] Oarlock scalar annotations (catch, finish, slip, wash)
- [x] Include/exclude rest strokes (toggle)
- [x] Export to PNG with logo watermark

**Adaptation required:**
- Replace `{{ .Data }}` with function parameter
- Replace `{{ .Title }}`, `{{ .ThresholdForce }}` with chartData properties
- Update logo path (Rowsandall → rownative.icu)

**Estimated effort:** 2-3 hours (mostly find/replace + testing)

### In-Stroke Analysis (`instroke.js`)

**Features to migrate:**
- [x] Quartile shading (Q1-Q3 band, light gray)
- [x] Median curve (black line)
- [x] Individual curves (optional overlay)
- [x] Crosshair tooltips (hover for exact values)
- [x] Annotations (SPM range, time range, analysis name)
- [x] Export to PNG

**Adaptation required:**
- Replace `{{ .Lines }}` with chartData.lines
- Replace `{{ .SpmMin }}`, `{{ .SpmMax }}`, etc. with chartData properties
- Remove or adapt save button (integrate with rownative.icu save flow)

**Estimated effort:** 2-3 hours

### In-Stroke Comparison (`instroke_compare.js`)

**File:** `templates/instroke_compare.js` (need to fetch from repo)

**Expected features:**
- Overlay multiple median curves
- Color-coded legend (from `{{ .Workouts }}`)
- Optional Y-axis normalization (scale different sensors)

**Adaptation required:**
- Replace `{{ .Workouts }}` with chartData.analyses array
- Map analysis IDs to colors

**Estimated effort:** 2-4 hours (might not exist as separate template; may need to extend instroke.js)

---

## Performance Comparison

| Approach | Server Load | Client Load | Latency | Maintenance |
|----------|-------------|-------------|---------|-------------|
| **Rowsandall (Go server)** | High (template rendering) | Low (just display) | +200ms | Two services |
| **Client-side D3.js** | Low (JSON only) | Medium (D3.js rendering) | 0ms extra | One service |

**Client-side is better for:**
- Lower latency (no extra HTTP hop)
- Scalability (offload rendering to client)
- Simpler deployment

**Server-side is better for:**
- Older browsers (pre-render charts)
- PDF/email generation (headless rendering)
- Consistent output (server controls exact rendering)

**Recommendation:** Client-side for MVP; add optional server-side rendering in Phase 2+ if needed (e.g., for email reports).

---

## Code Extraction Commands

```bash
# Clone rowsandall-charts repo (if you have access)
git clone https://git.wereldraadsel.tech/sander/rowsandall-charts.git

# Copy D3.js templates to new location
cd rowsandall-charts
cp templates/instroke.js ../courses/site/instroke/js/charts/instroke-original.js
cp templates/forcecurve.js ../courses/site/instroke/js/charts/forcecurve-original.js
cp templates/instroke_compare.js ../courses/site/instroke/js/charts/compare-original.js

# Now manually adapt: remove {{ }} syntax, wrap in functions
```

**Or:** Download directly from web:

```bash
curl https://git.wereldraadsel.tech/sander/rowsandall-charts/raw/branch/main/templates/instroke.js \
  -o site/instroke/js/charts/instroke-original.js
```

---

## Adaptation Example

### Before (Go Template)

```javascript
{{ define "chart" }}
var data = {{ .Data }}
var lines = {{ .Lines }}
var ytitle = {{ .Ytitle }}
var individual_curves = {{ .IndividualCurves }}
var spmmin = {{ .SpmMin }}
var spmmax = {{ .SpmMax }}

var SVG1 = d3.select("#interactive_viz")
  .append("svg")
  // ... D3.js code ...
{{ end }}
```

### After (Pure JavaScript Function)

```javascript
/**
 * Render in-stroke analysis chart
 * Adapted from Rowsandall templates/instroke.js
 * 
 * @param {string} containerId - DOM element ID (e.g., 'instroke_viz')
 * @param {object} chartData - Analysis data
 * @param {array} chartData.data - Summary data (x, median, high, low)
 * @param {array} chartData.lines - Individual stroke curves (optional)
 * @param {string} chartData.title - Chart title
 * @param {string} chartData.ytitle - Y-axis label
 * @param {boolean} chartData.individualCurves - Show individual strokes?
 * @param {number} chartData.spmMin - Filter: min SPM
 * @param {number} chartData.spmMax - Filter: max SPM
 */
function renderInstrokeChart(containerId, chartData) {
  const data = chartData.data;
  const lines = chartData.lines || [];
  const ytitle = chartData.ytitle;
  const individualCurves = chartData.individualCurves || false;
  const spmMin = chartData.spmMin;
  const spmMax = chartData.spmMax;
  const title = chartData.title;
  
  const margin = { top: 30, right: 60, bottom: 40, left: 60 };
  const containerWidth = document.getElementById(containerId).offsetWidth;
  const width = containerWidth - margin.left - margin.right;
  const height = width / 1.618 - margin.top - margin.bottom;
  
  var SVG1 = d3.select(`#${containerId}`)
    .append("svg")
    .attr("id", "my_svg1")
    .attr("width", width + margin.left + margin.right)
    .attr("height", height + margin.top + margin.bottom)
    .append("g")
    .attr("transform", `translate(${margin.left},${margin.top})`);
  
  // ... rest of D3.js code from Rowsandall template (mostly unchanged)
  
  // Quartile band
  const area = d3.area()
    .x(d => xScale(100 * d.x / xaxismax))
    .y0(d => yScale(d.low))
    .y1(d => yScale(d.high));
  
  SVG1.append("path")
    .datum(data)
    .attr("fill", "lightgray")
    .attr("d", area);
  
  // Median curve
  const lineMedian = d3.line()
    .x(d => xScale(100 * d.x / xaxismax))
    .y(d => yScale(d.median));
  
  SVG1.append("path")
    .datum(data)
    .attr("fill", "none")
    .attr("stroke", "black")
    .attr("stroke-width", 2.0)
    .attr("d", lineMedian);
  
  // Individual curves (optional)
  if (individualCurves) {
    lines.forEach(line => {
      const lineData = Object.keys(line).map(k => ({ x: +k, y: line[k] }));
      const lineFunc = d3.line()
        .x(d => xScale(100 * d.x / xaxismax))
        .y(d => yScale(d.y));
      
      SVG1.append("path")
        .datum(lineData)
        .attr("fill", "none")
        .attr("stroke", "gray")
        .attr("stroke-width", 0.5)
        .attr("d", lineFunc);
    });
  }
  
  // Crosshairs, axes, labels, etc. (reuse from Rowsandall)
}
```

---

## Files to Adapt

### 1. `instroke.js` (Single Workout Analysis)

**Source:** `rowsandall-charts/templates/instroke.js`

**Features:**
- Quartile shading (Q1-Q3 band)
- Median curve
- Individual stroke overlay (toggle)
- Crosshair tooltips
- Export to PNG

**Go Template Variables:**
```go
type InstrokeData struct {
    Data []InstrokeSummaryRecord `json:"data"`  // x, median, high, low
    Title string `json:"title"`
    Ytitle string `json:"ytitle"`
    Lines []map[int]float64 `json:"lines"`  // Individual curves
    IndividualCurves bool `json:"individual_curves"`
    TimeMin string `json:"timemax"`  // Note: swapped in Go code
    TimeMax string `json:"timemin"`
    SpmMin float64 `json:"spmmin"`
    SpmMax float64 `json:"spmmax"`
    AnalysisName string `json:"analysis_name"`
}
```

**Adaptation:**
- Wrap entire template in `function renderInstrokeChart(containerId, chartData)`
- Replace `{{ .Data }}` with `chartData.data`
- Replace `#interactive_viz` with `#${containerId}`
- Keep all D3.js logic intact

**Estimated LOC:** ~200 lines (mostly copy/paste)

### 2. `forcecurve.js` (Force Curve with Sliders)

**Source:** `rowsandall-charts/templates/forcecurve.js`

**Features:**
- Interactive sliders (SPM, distance, work per stroke)
- Median force curve (catch → slip → peak → wash → finish)
- Average force (horizontal dashed line)
- Individual stroke curves (thin lines, toggle)
- Peak force circles (toggle)
- Annotations (peak force, catch/finish angles, slip/wash)
- Real-time filtering (updateChart() on slider change)

**Go Template Variables:**
```go
type ForceData struct {
    Data []StrokeRecord `json:"data"`  // All stroke data
    Title string `json:"title"`
    ThresholdForce float64 `json:"thresholdforce"`  // 100N for sculling, 200N for sweep
}
```

**Adaptation:**
- Wrap in `function renderForceCurve(containerId, chartData, onFilterChange)`
- Replace `{{ .Data }}` with `chartData.data`
- Keep slider logic (already JavaScript, no Go templates in event handlers)
- Update `updateChart()` to accept callback for saving filter state

**Estimated LOC:** ~300 lines

### 3. `instroke_compare.js` (Multi-Analysis Comparison)

**Source:** Need to check if this exists as separate template, or if it's an extension of `instroke.js`

**Expected features:**
- Overlay multiple median curves
- Color-coded legend
- Toggle curves on/off

**If doesn't exist:** Extend `instroke.js` to accept multiple datasets (similar to Rowsandall's `forcecurve_compare.js`).

**Estimated LOC:** ~150 lines (if building from instroke.js) or ~250 lines (if exists separately)

---

## Integration with Worker API

### Analysis Endpoint Response Format

**Worker API:** `GET /api/instroke/analyses/456`

```json
{
  "id": 456,
  "name": "Race - 32 spm",
  "workoutId": 123,
  "filters": { "spmMin": 31, "spmMax": 33 },
  "strokeCount": 42,
  "chartData": {
    "data": [
      { "x": 0, "median": 0, "high": 0, "low": 0 },
      { "x": 1, "median": 12.5, "high": 18.2, "low": 8.1 },
      { "x": 2, "median": 45.3, "high": 62.1, "low": 32.4 },
      ...
    ],
    "lines": [
      { "0": 0, "1": 11.2, "2": 43.1, ... },  // Stroke 1
      { "0": 0, "1": 13.8, "2": 47.5, ... },  // Stroke 2
    ],
    "title": "2026-03-15 Morning Row - boat accelerator curve",
    "ytitle": "Boat acceleration (m/s^2)",
    "spmMin": 31,
    "spmMax": 33,
    "timeMin": "00:05:30",
    "timeMax": "00:12:45",
    "analysisName": "Race - 32 spm"
  }
}
```

**Frontend rendering:**

```html
<script>
  fetch('/api/instroke/analyses/456')
    .then(res => res.json())
    .then(analysis => {
      renderInstrokeChart('viz_container', analysis.chartData);
    });
</script>
```

---

## Slider HTML (from Rowsandall)

**HTML structure for interactive filters:**

```html
<div class="filters">
  <label>SPM Range:
    <input type="range" id="minSPM" class="slider" min="0" max="60" value="15">
    <span id="minSPMValue">15</span>
    -
    <input type="range" id="maxSPM" class="slider" min="0" max="60" value="60">
    <span id="maxSPMValue">60</span>
  </label>
  
  <label>Distance (m):
    <input type="range" id="minDist" class="slider" min="0" max="10000" value="0">
    <span id="minDistValue">0</span>
    -
    <input type="range" id="maxDist" class="slider" min="0" max="10000" value="5000">
    <span id="maxDistValue">5000</span>
  </label>
  
  <label>
    <input type="checkbox" id="plotlines"> Show individual strokes
  </label>
  
  <label>
    <input type="checkbox" id="includereststrokes"> Include rest strokes
  </label>
</div>

<div id="data_viz"></div>
<canvas id="canvas" style="display:none;"></canvas>  <!-- For PNG export -->
```

**JavaScript (from Rowsandall):**

```javascript
d3.selectAll(".slider").on("input", function() {
  minSPM = $("#minSPM").val();
  maxSPM = $("#maxSPM").val();
  minDist = $("#minDist").val();
  maxDist = $("#maxDist").val();
  
  // Filter data
  filterData = data.filter(d => 
    d.spm >= minSPM && d.spm <= maxSPM &&
    d.cumdist >= minDist && d.cumdist <= maxDist
  );
  
  // Update chart
  updateChart();
  
  // Update slider value displays
  $("#minSPMValue").text(minSPM);
  $("#maxSPMValue").text(maxSPM);
  $("#minDistValue").text(minDist);
  $("#maxDistValue").text(maxDist);
});
```

---

## PNG Export (from Rowsandall)

**Feature:** Export chart as PNG with logo watermark

```javascript
d3.select('#saveButton').on('click', function(){
  // Remove grid lines for cleaner export
  $("#x-grid").remove();
  $("#y-grid").remove();
  
  // Serialize SVG to string
  const svgNode = document.querySelector('#my_svg1');
  const svgString = (new XMLSerializer()).serializeToString(svgNode);
  const svgBlob = new Blob([svgString], { type: 'image/svg+xml;charset=utf-8' });
  
  // Create image from SVG
  const url = URL.createObjectURL(svgBlob);
  const image = new Image();
  image.width = containerWidth;
  image.height = containerHeight;
  image.src = url;
  
  image.onload = function() {
    // Draw to canvas
    const canvas = document.getElementById('canvas');
    canvas.width = image.width;
    canvas.height = image.height;
    const ctx = canvas.getContext('2d');
    
    // White background
    ctx.fillStyle = "white";
    ctx.fillRect(0, 0, canvas.width, canvas.height);
    
    // Add logo watermark
    const bgImg = new Image();
    bgImg.src = '/static/img/rownative-logo.png';  // Update path
    bgImg.onload = function() {
      ctx.drawImage(bgImg, 50, 50, 200, 200 * bgImg.height / bgImg.width);
      ctx.drawImage(image, 0, 0);
      URL.revokeObjectURL(url);
      
      // Trigger download
      const imgURI = canvas.toDataURL('image/png')
        .replace('image/png', 'image/octet-stream');
      const link = document.createElement('a');
      link.download = 'force-curve.png';
      link.href = imgURI;
      link.click();
    };
  };
  
  // Add grid lines back
  // ... (code to re-append grids)
});
```

---

## Testing Strategy

### Unit Tests (Vitest)

```typescript
// test/d3-charts.test.ts

import { describe, it, expect, beforeEach } from 'vitest';
import { JSDOM } from 'jsdom';
import * as d3 from 'd3';

describe('D3.js Chart Rendering', () => {
  let dom: JSDOM;
  let document: Document;
  
  beforeEach(() => {
    dom = new JSDOM('<!DOCTYPE html><body><div id="test_viz"></div></body>');
    document = dom.window.document;
    global.document = document;
    global.window = dom.window as any;
  });
  
  it('should render instroke chart with median curve', () => {
    const chartData = {
      data: [
        { x: 0, median: 0, high: 0, low: 0 },
        { x: 50, median: 100, high: 120, low: 80 },
        { x: 100, median: 0, high: 0, low: 0 },
      ],
      title: 'Test',
      ytitle: 'Force (N)',
      spmMin: 28,
      spmMax: 32,
    };
    
    renderInstrokeChart('test_viz', chartData);
    
    // Check SVG was created
    const svg = document.querySelector('#my_svg1');
    expect(svg).toBeDefined();
    
    // Check median path exists
    const medianPath = document.querySelector('path.median');
    expect(medianPath).toBeDefined();
  });
});
```

### Integration Tests

Test with actual FIT files from Rowsandall export:

1. Parse Empower FIT → extract curves
2. Compute quartiles
3. Render chart → verify SVG output
4. Simulate slider interaction → verify filtering

---

## Migration Timeline

| Phase | Duration | Task |
|-------|----------|------|
| **Extract** | 1 day | Download Rowsandall templates, copy to project |
| **Adapt** | 3-4 days | Remove Go syntax, wrap in functions, test locally |
| **Integrate** | 2-3 days | Connect to Worker API, test with real FIT data |
| **Polish** | 2-3 days | Update branding (logo, colors), accessibility, mobile |
| **Test** | 2 days | Cross-browser testing, performance testing |
| **Total** | **10-15 days** | Ready for alpha testing |

**vs Building from scratch with Chart.js:** 15-20 days (design, implement, test)

**Time saved:** ~5-7 days (33% faster)

---

## Open Questions

### 1. License Compatibility

**Rowsandall-charts license:** MIT (confirmed from repo)  
**rownative/worker license:** MIT  
**Compatibility:** ✅ Compatible — can reuse freely with attribution

**Attribution:** Add to README:
```markdown
## Credits
In-stroke analysis charts adapted from [Rowsandall](https://git.wereldraadsel.tech/sander/rowsandall-charts) (MIT License) by Sander Roosendaal.
```

### 2. Go Server Alternative

**Should we optionally deploy the Go chart server for advanced users?**

**Pros:**
- Zero adaptation work (use as-is)
- Server-side rendering for PDF exports

**Cons:**
- Separate deployment (not Cloudflare)
- Additional hosting cost (~$5/month for small VPS)
- More maintenance burden

**Recommendation:** Client-side only for MVP; consider Go server in Phase 2 if there's demand for server-side rendering.

### 3. Bokeh vs D3.js Confusion

**Clarification:** Rowsandall's `interactiveplots.py` imports Bokeh, but the **chart server uses D3.js**. Bokeh might be used for:
- Server-side static chart generation (not interactive)
- Holoviews integration (mentioned in imports)
- Legacy code (not used in chart server)

**For reuse:** Focus on D3.js templates, ignore Bokeh.

---

## Conclusion

**Recommendation:** **Reuse Rowsandall's D3.js templates** for all in-stroke charting.

**Rationale:**
1. ✅ **Proven UX** — users already familiar with interface
2. ✅ **Less work** — adapt vs build from scratch (save 5-7 days)
3. ✅ **Smooth migration** — minimize relearning for Rowsandall users
4. ✅ **No extra infrastructure** — client-side rendering (no Go server)
5. ✅ **Advanced features** — sliders, crosshairs, export already working

**Implementation order:**
1. Extract `instroke.js` (priority 1 — most important)
2. Extract `forcecurve.js` (priority 2 — nice to have)
3. Build comparison chart (priority 3 — extend instroke.js)

**Next steps:**
1. Download Rowsandall templates
2. Create `site/instroke/js/charts/` directory
3. Adapt first template (instroke.js)
4. Test with sample data
5. Integrate with Worker API

---

**Prepared by:** AI Assistant (Cursor)  
**For:** rownative/courses frontend development  
**Source:** Rowsandall chart server (MIT licensed)
