# In-Stroke Analysis Platform — Requirements Document

> Functional and non-functional requirements for the Rowsandall.com in-stroke analysis replacement.

**Version:** 1.0  
**Date:** April 9, 2026  
**Status:** Draft for Community Review

---

## 1. Introduction

### 1.1 Document Purpose

This document specifies the requirements for the in-stroke analysis companion app to intervals.icu, replacing Rowsandall.com's force curve and in-stroke analysis functionality before its shutdown (end of 2026).

### 1.2 Scope

**In Scope:**
- Import rowing workouts with in-stroke data from intervals.icu
- Parse FIT files per the rowingdata FIT_EXPORT specification
- Analyze force curves, seat curves, boat acceleration, and oarlock metrics
- Filter strokes by rate, interval, or time window
- Compute quartiles, mean curves, and statistical summaries
- Save analyses with user-provided names
- Compare multiple analyses with overlaid charts
- Share analyses via public links
- Migrate existing Rowsandall analyses

**Out of Scope (Future):**
- Video synchronization with force curves
- Real-time analysis during rowing (CrewNerd integration in Phase 3+)
- Coaching tools (annotations, athlete management)
- Mobile native apps (responsive web UI only)
- Automatic workout import (user must manually trigger ingestion)

### 1.3 Target Audience

**Primary Users:**
- Competitive rowers (club, masters, elite) analyzing technique
- Coaches reviewing athlete data
- Rowing scientists conducting research
- Indoor rowing enthusiasts (RP3, Concept2 with sensors)

**Supported Sensors/Platforms:**
- Empower Oarlock (NK)
- Quiske sensors
- RowPerfect/RP3 ergs
- Smartphone apps (future; custom in-stroke FIT export)
- Any FIT file compliant with rowingdata FIT_EXPORT spec

### 1.4 References

- [Rowsandall Force Curves Guide](https://analytics.rowsandall.com/2024/06/20/how-to-work-with-force-curves-and-in-stroke-analysis/)
- [rowingdata FIT_EXPORT.md](https://github.com/sanderroosendaal/rowingdata/blob/develop/docs/FIT_EXPORT.md)
- [intervals.icu API Documentation](https://intervals.icu/api/v1/docs/swagger-ui/index.html)
- [rownative/courses Architecture](https://github.com/rownative/courses/blob/main/SPECIFICATION.md)

---

## 2. Functional Requirements

### 2.1 Authentication and Authorization

#### FR-AUTH-01: OAuth Login
- **Description:** Users must authenticate via intervals.icu OAuth.
- **Rationale:** No credential management; leverage existing intervals.icu trust.
- **Acceptance Criteria:**
  - Clicking "Sign In" redirects to intervals.icu OAuth consent page
  - After consent, user is redirected back with access token
  - Session cookie (`rn_session`) is set with 90-day TTL
  - Session cookie is encrypted (AES-256-GCM)
  - User remains signed in across page navigation

#### FR-AUTH-02: Session Management
- **Description:** User sessions persist until logout or expiration.
- **Acceptance Criteria:**
  - Session expires after 90 days of inactivity
  - "Logout" button clears session cookie and redirects to landing page
  - `/api/me` endpoint returns user's athlete ID and authentication state

#### FR-AUTH-03: Authorization
- **Description:** Users can only access their own workouts and analyses.
- **Acceptance Criteria:**
  - GET `/api/instroke/workouts` returns only workouts owned by authenticated user
  - POST/PATCH/DELETE to analyses require ownership (403 if not owner)
  - Shared analyses (public) are accessible without authentication

### 2.2 Workout Ingestion

#### FR-INGEST-01: Activity Browser
- **Description:** Users can browse their OTW rowing activities from intervals.icu.
- **Acceptance Criteria:**
  - Page fetches OTW rowing activities from last 3 months via intervals.icu API
  - Activities are displayed in a table: Date, Name, Distance, Duration, SPM
  - User can filter by date range
  - Activities already ingested show "Analyzed" badge
  - Click "Analyze" button to ingest activity

#### FR-INGEST-02: FIT File Download
- **Description:** System downloads FIT file from intervals.icu for selected activity.
- **Acceptance Criteria:**
  - POST `/api/instroke/workouts/ingest` accepts `{ activityId }`
  - Worker fetches FIT file from intervals.icu using bearer token
  - Download timeout: 30 seconds
  - Max file size: 10MB
  - Error handling: 404 if activity not found, 422 if FIT is malformed

#### FR-INGEST-03: FIT Parsing
- **Description:** System parses FIT file and extracts in-stroke data.
- **Acceptance Criteria:**
  - Parser extracts developer fields per rowingdata FIT_EXPORT spec:
    - Force curves (HandleForceCurve, BoatAcceleratorCurve, OarAngleVelocityCurve, SeatCurve)
    - Oarlock scalars (Catch, Finish, Slip, Wash, PeakForceAngle, EffectiveLength)
    - Stroke timing (StrokeDriveTime, StrokeRecoveryTime)
    - Drive metrics (DriveLength, AverageDriveForceN, PeakDriveForceN)
    - Abscissa metadata (InstrokeAbscissaType, InstrokeSampleInterval, InstrokePointCount)
  - Parser handles `.instroke.json` companion files if present
  - Parser infers sensor type from developer field application_id (e.g., "Empower", "Quiske")
  - Parser validates curve data (point count matches metadata, values in expected range)
  - Parsing failures return 422 with human-readable error message

#### FR-INGEST-04: Workout Storage
- **Description:** Extracted workout data is stored in D1 database.
- **Acceptance Criteria:**
  - Workout metadata stored in `instroke_workouts` table
  - Per-stroke curves stored in `instroke_curves` table
  - Duplicate ingestion (same activityId) returns existing workout ID (idempotent)
  - POST response includes workout ID, curve types, stroke count
  - Storage operation completes within 10 seconds for typical 30-minute workout

#### FR-INGEST-05: Ingestion Confirmation
- **Description:** User receives confirmation after successful ingestion.
- **Acceptance Criteria:**
  - Success toast notification: "Workout ingested: 855 strokes, 3 curve types"
  - Page redirects to analysis page: `/instroke/analyze.html?workout={id}`
  - Error notification if ingestion fails (with reason)

### 2.3 Single Workout Analysis

#### FR-ANALYZE-01: Workout Metadata Display
- **Description:** Analysis page displays workout summary.
- **Acceptance Criteria:**
  - Header shows: Date, Distance, Duration, Avg SPM, Sensor Type
  - Available curve types listed (e.g., "Force Curve, Boat Acceleration")
  - Link to intervals.icu activity (opens in new tab)

#### FR-ANALYZE-02: Stroke Filtering
- **Description:** User can filter strokes by criteria.
- **Acceptance Criteria:**
  - SPM range sliders (min/max, range 0-60, default: all strokes)
  - Interval dropdown (e.g., "All", "Interval 1", "Interval 2", ...)
  - Time window input (start/end in MM:SS format)
  - Stroke count updates dynamically as filters change
  - "Reset Filters" button clears all selections

#### FR-ANALYZE-03: Force Curve Visualization
- **Description:** System displays force curve for selected strokes.
- **Acceptance Criteria:**
  - Chart displays mean force curve (averaged across selected strokes)
  - Quartile shading (Q1-Q3) shown as semi-transparent band
  - X-axis: Normalized drive phase (0-1) or time (milliseconds) based on abscissa type
  - Y-axis: Force (Newtons) or raw units from FIT
  - Hover tooltip shows exact force value at point
  - Chart is interactive (zoom, pan via Chart.js plugins)
  - "Show individual strokes" toggle overlays all selected strokes (max 50 for performance)

#### FR-ANALYZE-04: Curve Type Selection
- **Description:** User can switch between curve types if multiple are available.
- **Acceptance Criteria:**
  - Tabs or dropdown to select curve type (e.g., "Force Curve", "Boat Acceleration", "Seat Curve")
  - Chart updates immediately on selection
  - Selected curve type persisted in URL query parameter (shareable link)

#### FR-ANALYZE-05: Statistical Summary
- **Description:** System displays statistical metrics for selected strokes.
- **Acceptance Criteria:**
  - Table shows: Stroke count, Avg SPM, Avg force, Peak force, Drive time, Recovery time
  - Metrics computed from filtered strokes only
  - Metrics updated dynamically as filters change

#### FR-ANALYZE-06: Save Analysis
- **Description:** User can save analysis snapshot for later comparison.
- **Acceptance Criteria:**
  - "Save Analysis" button opens modal
  - Modal prompts for: Name (required, max 100 chars), Description (optional, max 500 chars)
  - POST `/api/instroke/analyses` with filter criteria and curve type
  - Success notification: "Analysis saved: {name}"
  - Link to analysis library

#### FR-ANALYZE-07: Analysis Persistence
- **Description:** Saved analyses include all filter criteria and computed metrics.
- **Acceptance Criteria:**
  - Analysis record stored in D1 with: workoutId, name, filters, curveType, strokeCount
  - Computed metrics (quartiles, mean curve) stored in `instroke_analysis_metrics`
  - Analysis is immutable (cannot edit filters; user must create new analysis)
  - User can update name and description via PATCH endpoint

### 2.4 Analysis Library

#### FR-LIBRARY-01: List User Analyses
- **Description:** Page displays all analyses created by user.
- **Acceptance Criteria:**
  - Table shows: Name, Date created, Workout date, Curve type, Strokes, Actions
  - Sorted by creation date (newest first)
  - Pagination (20 analyses per page)
  - Search by name (client-side filter)
  - Filter by curve type (dropdown)

#### FR-LIBRARY-02: Analysis Actions
- **Description:** User can manage analyses from library page.
- **Acceptance Criteria:**
  - "View" button opens analysis detail (read-only chart + metadata)
  - "Compare" checkbox (select 2-5 analyses, then click "Compare" button)
  - "Share" button generates shareable link (see FR-SHARE-01)
  - "Rename" button opens modal to edit name/description
  - "Delete" button prompts confirmation, then DELETE `/api/instroke/analyses/{id}`

#### FR-LIBRARY-03: Empty State
- **Description:** Library page shows helpful message if no analyses exist.
- **Acceptance Criteria:**
  - Message: "No analyses yet. Go to Activity Browser to import your first workout."
  - Button: "Browse Activities" links to `/instroke/index.html`

### 2.5 Multi-Analysis Comparison

#### FR-COMPARE-01: Comparison Page
- **Description:** Page displays overlaid force curves from multiple analyses.
- **Acceptance Criteria:**
  - URL format: `/instroke/compare.html?ids={id1,id2,id3}`
  - Fetches comparison data via POST `/api/instroke/compare`
  - Chart displays mean curves for each analysis (distinct colors)
  - Legend shows: Analysis name, Date, Color swatch, Strokes
  - Curves are aligned to common abscissa (0-1 normalized)

#### FR-COMPARE-02: Comparison Interactivity
- **Description:** User can customize comparison view.
- **Acceptance Criteria:**
  - Toggle individual curves on/off (click legend)
  - Toggle Q1/Q3 shading for each curve (checkbox per analysis)
  - Normalize Y-axis (force) to 0-1 (optional, for comparing different sensors)
  - Zoom/pan synchronized across all curves

#### FR-COMPARE-03: Remove Analysis from Comparison
- **Description:** User can remove an analysis from comparison.
- **Acceptance Criteria:**
  - "Remove" button next to each analysis in legend
  - Curve immediately disappears from chart
  - URL updates to reflect remaining analysis IDs

#### FR-COMPARE-04: Comparison Limits
- **Description:** System enforces reasonable limits on comparison complexity.
- **Acceptance Criteria:**
  - Max 5 analyses in one comparison (UI prevents selecting more)
  - Error message if >5 IDs in URL: "Maximum 5 analyses per comparison"

### 2.6 Sharing

#### FR-SHARE-01: Generate Share Link
- **Description:** User can create shareable link for analysis or comparison.
- **Acceptance Criteria:**
  - "Share" button on analysis detail or comparison page
  - Modal prompts: Privacy (Public/Private), Expiration (30/90 days, Never)
  - POST `/api/instroke/share` with analysisIds and settings
  - Response includes shareable URL
  - URL copied to clipboard automatically
  - Success notification: "Share link copied to clipboard"

#### FR-SHARE-02: Public Shared Analysis
- **Description:** Anyone with share link can view analysis (no login required).
- **Acceptance Criteria:**
  - GET `/instroke/shared/{uuid}` renders analysis or comparison
  - Page is read-only (no save, delete, or edit buttons)
  - Athlete name shown only if analysis is marked public
  - Private analyses show "Anonymous" instead of athlete name
  - Page includes "Create your own analyses at rownative.icu" CTA

#### FR-SHARE-03: Share Link Expiration
- **Description:** Share links expire after configured TTL.
- **Acceptance Criteria:**
  - Expired links return 410 Gone: "This share link has expired"
  - User can regenerate share link from library page

#### FR-SHARE-04: Share Link Revocation
- **Description:** User can revoke share link.
- **Acceptance Criteria:**
  - "Revoke Share Link" button in analysis detail (if link exists)
  - DELETE `/api/instroke/shares/{uuid}` removes from KV
  - Existing links return 404: "This share link has been revoked"

### 2.7 Data Management

#### FR-DATA-01: Workout Deletion
- **Description:** User can delete ingested workouts.
- **Acceptance Criteria:**
  - DELETE `/api/instroke/workouts/{id}` removes workout and associated curves
  - Confirmation modal: "Delete workout from {date}? This will also delete {N} analyses."
  - Cascading delete: All analyses referencing workout are deleted
  - Deletion is permanent (no undo)

#### FR-DATA-02: Analysis Deletion
- **Description:** User can delete individual analyses without deleting workout.
- **Acceptance Criteria:**
  - DELETE `/api/instroke/analyses/{id}` removes analysis and metrics
  - Confirmation modal: "Delete analysis '{name}'? This cannot be undone."
  - Workout and curves remain in database

#### FR-DATA-03: Bulk Operations
- **Description:** User can perform bulk actions from library page.
- **Acceptance Criteria:**
  - "Select All" checkbox in library table header
  - "Delete Selected" button (prompts confirmation)
  - Bulk delete processes up to 50 analyses (UI enforces limit)

### 2.8 Migration from Rowsandall

#### FR-MIGRATE-01: Export from Rowsandall
- **Description:** Rowsandall users can export their analyses.
- **Acceptance Criteria:**
  - Rowsandall adds "Export In-Stroke Analyses" button (implemented in Rowsandall)
  - Export generates ZIP with: `analyses.json`, FIT files, precomputed metrics
  - ZIP download size limit: 100MB
  - Export includes analyses created in last 2 years (configurable)

#### FR-MIGRATE-02: Import to Instroke Platform
- **Description:** Users can import Rowsandall export ZIP.
- **Acceptance Criteria:**
  - `/instroke/import.html` page with file upload widget
  - POST `/api/instroke/import-rowsandall` accepts multipart form
  - Worker unzips, validates `analyses.json` schema
  - Workouts ingested via existing FIT parsing flow
  - Analyses recreated with mapped workout IDs
  - Progress indicator during import (e.g., "Importing 5 of 12 workouts...")
  - Import summary displayed: "Success: 10 analyses, Failed: 2 (errors: ...)"

#### FR-MIGRATE-03: Error Handling
- **Description:** Import process reports errors clearly.
- **Acceptance Criteria:**
  - Validation errors: "Invalid ZIP structure", "Missing analyses.json"
  - Partial success: "8 of 10 workouts imported successfully"
  - Detailed errors: "Workout {activityId} failed: FIT parsing error at stroke 42"
  - Link to successfully imported analyses

### 2.9 Help and Documentation

#### FR-HELP-01: Help Page
- **Description:** Comprehensive documentation for users.
- **Acceptance Criteria:**
  - `/instroke/help.html` page with sections:
    - Getting Started (OAuth, ingestion, first analysis)
    - Sensor Compatibility (list of supported devices)
    - Curve Interpretation (what force curves mean)
    - FIT Format (link to rowingdata spec)
    - FAQ (common issues, troubleshooting)
  - Search functionality (client-side, via Ctrl+F)

#### FR-HELP-02: Tooltips and Contextual Help
- **Description:** In-app tooltips for key features.
- **Acceptance Criteria:**
  - Question mark icons next to technical terms (e.g., "Quartiles", "Abscissa Type")
  - Hover/click opens tooltip with brief explanation
  - Tooltips link to relevant help page section

---

## 3. Non-Functional Requirements

### 3.1 Performance

#### NFR-PERF-01: Page Load Time
- **Requirement:** All pages load in <2 seconds on 3G connection.
- **Measurement:** Lighthouse performance score >90.

#### NFR-PERF-02: Analysis Creation Time
- **Requirement:** POST `/api/instroke/analyses` completes in <5 seconds for typical 30-minute workout.
- **Measurement:** p95 latency <5s in production monitoring.

#### NFR-PERF-03: Chart Rendering
- **Requirement:** Force curve chart renders in <500ms for up to 100 strokes.
- **Measurement:** Frame rate >30fps during zoom/pan (Chrome DevTools Performance tab).

#### NFR-PERF-04: Comparison Loading
- **Requirement:** Comparison page loads 5 analyses in <3 seconds.
- **Measurement:** Total time from URL navigation to chart fully rendered.

### 3.2 Scalability

#### NFR-SCALE-01: User Capacity
- **Requirement:** Support 1,000 concurrent users without degradation.
- **Measurement:** Load testing with Cloudflare Workers' concurrency limits.

#### NFR-SCALE-02: Data Volume
- **Requirement:** Handle 10,000 ingested workouts (≈100,000 analyses) within free tier limits.
- **Measurement:** D1 storage <10GB, KV operations <100k reads/day.

#### NFR-SCALE-03: FIT Parsing Throughput
- **Requirement:** Parse and store 1 FIT file per 10 seconds per user (rate limit).
- **Measurement:** Monitor ingestion queue latency.

### 3.3 Reliability

#### NFR-REL-01: Uptime
- **Requirement:** 99.5% uptime (excluding planned maintenance).
- **Measurement:** Cloudflare Workers dashboard (uptime percentage).

#### NFR-REL-02: Data Durability
- **Requirement:** No data loss in D1 or KV (Cloudflare guarantees durability).
- **Measurement:** Zero reported data loss incidents.

#### NFR-REL-03: Error Recovery
- **Requirement:** FIT parsing failures do not crash worker; errors logged and returned as 422.
- **Measurement:** Worker error rate <5% under normal operation.

### 3.4 Security

#### NFR-SEC-01: Authentication
- **Requirement:** All authenticated endpoints require valid session cookie.
- **Measurement:** Penetration testing confirms no auth bypass.

#### NFR-SEC-02: Authorization
- **Requirement:** Users cannot access other users' workouts or analyses.
- **Measurement:** OWASP ZAP scan finds no authorization vulnerabilities.

#### NFR-SEC-03: Input Validation
- **Requirement:** All user inputs sanitized (XSS prevention).
- **Measurement:** Content Security Policy (CSP) enforced; no XSS in security audit.

#### NFR-SEC-04: Rate Limiting
- **Requirement:** Ingestion limited to 10 requests per 10 minutes per user.
- **Measurement:** Rate limiter blocks excess requests with 429 response.

### 3.5 Usability

#### NFR-USAB-01: Accessibility
- **Requirement:** WCAG 2.1 Level AA compliance.
- **Measurement:** Automated testing (axe-core) reports 0 critical issues.

#### NFR-USAB-02: Mobile Responsiveness
- **Requirement:** All pages usable on mobile (portrait and landscape).
- **Measurement:** Manual testing on iOS Safari and Chrome Android; Lighthouse mobile score >80.

#### NFR-USAB-03: Browser Compatibility
- **Requirement:** Support Chrome, Firefox, Safari, Edge (last 2 versions).
- **Measurement:** Manual testing confirms no critical bugs.

#### NFR-USAB-04: Internationalization (Future)
- **Requirement:** UI text externalized for translation (Phase 2+).
- **Measurement:** i18n keys used for all UI strings (no hardcoded English).

### 3.6 Maintainability

#### NFR-MAINT-01: Code Quality
- **Requirement:** TypeScript strict mode enabled; ESLint warnings resolved.
- **Measurement:** ESLint CI check passes; no TypeScript errors.

#### NFR-MAINT-02: Test Coverage
- **Requirement:** >80% test coverage for worker API (unit + integration).
- **Measurement:** Vitest coverage report.

#### NFR-MAINT-03: Documentation
- **Requirement:** All API endpoints documented (JSDoc or OpenAPI).
- **Measurement:** README includes API reference; code comments present.

### 3.7 Interoperability

#### NFR-INTEROP-01: FIT Spec Compliance
- **Requirement:** Parser handles all FIT files compliant with rowingdata FIT_EXPORT spec.
- **Measurement:** Test suite includes FIT files from Empower, Quiske, RP3.

#### NFR-INTEROP-02: intervals.icu API
- **Requirement:** OAuth and activity API calls follow intervals.icu documentation.
- **Measurement:** Manual testing with intervals.icu sandbox/production.

#### NFR-INTEROP-03: Browser APIs
- **Requirement:** Use standard Web APIs (no vendor-specific features).
- **Measurement:** No console warnings for deprecated APIs.

---

## 4. Data Requirements

### 4.1 Data Model

#### DR-MODEL-01: Workout Metadata
- **Description:** Store workout-level information.
- **Entities:** `instroke_workouts` table (D1)
- **Attributes:** athlete_id, activity_id, activity_date, sport, duration_s, distance_m, avg_spm, stroke_count, sensor_type, curve_types, abscissa_type, created_at
- **Constraints:** activity_id UNIQUE per athlete_id

#### DR-MODEL-02: Per-Stroke Curves
- **Description:** Store raw curve data for each stroke.
- **Entities:** `instroke_curves` table (D1)
- **Attributes:** workout_id (FK), stroke_number, spm, interval_idx, curve_type, abscissa_type, point_count, curve_data (BLOB), oarlock_scalars, drive_metrics
- **Constraints:** (workout_id, stroke_number, curve_type) UNIQUE

#### DR-MODEL-03: Analyses
- **Description:** Store user-created analysis snapshots.
- **Entities:** `instroke_analyses` table (D1)
- **Attributes:** uuid, athlete_id, workout_id (FK), name, description, filters, stroke_count, curve_types, is_public, created_at
- **Constraints:** uuid UNIQUE

#### DR-MODEL-04: Analysis Metrics
- **Description:** Store precomputed metrics (quartiles, mean).
- **Entities:** `instroke_analysis_metrics` table (D1)
- **Attributes:** analysis_id (FK), curve_type, metric_name, metric_data (BLOB)
- **Constraints:** (analysis_id, curve_type, metric_name) UNIQUE

#### DR-MODEL-05: Shared Links
- **Description:** Store shareable analysis links.
- **Entities:** KV namespace (key: `instroke:share:{uuid}`)
- **Attributes:** analysisIds (array), createdAt, expiresAt
- **Constraints:** TTL enforced by KV

### 4.2 Data Retention

#### DR-RETAIN-01: Workouts
- **Requirement:** User-initiated deletion only (no automatic expiration).
- **Rationale:** Long-term trend analysis requires historical data.
- **Future:** Consider "archive" feature for old workouts (move to cheaper storage).

#### DR-RETAIN-02: Analyses
- **Requirement:** User-initiated deletion only.
- **Rationale:** Analyses are small (metadata + metrics); storage cost negligible.

#### DR-RETAIN-03: Share Links
- **Requirement:** Expire after user-configured TTL (30/90 days, or never).
- **Rationale:** Balance usability (long-lived links) with privacy (expiring sensitive data).

### 4.3 Data Privacy

#### DR-PRIVACY-01: Personal Data
- **Description:** Platform stores minimal personal data.
- **Data Points:** athlete_id (from intervals.icu), activity_date, workout metrics
- **Exclusions:** No names, emails, or photos stored (fetched from intervals.icu API on demand)

#### DR-PRIVACY-02: Shared Analyses
- **Description:** Private by default; opt-in to public sharing.
- **Default:** is_public=false (athlete name hidden in shared views)
- **Opt-in:** User sets is_public=true (athlete name shown)

#### DR-PRIVACY-03: GDPR Compliance
- **Description:** Support data subject rights.
- **Rights:** Access (GET workouts/analyses), Deletion (DELETE endpoints), Portability (future: export as JSON)

---

## 5. Interface Requirements

### 5.1 User Interface

#### IR-UI-01: Design Consistency
- **Requirement:** Match rownative.icu visual style (colors, fonts, layout).
- **Rationale:** Unified brand experience; reduce cognitive load.

#### IR-UI-02: Navigation
- **Requirement:** Global navigation bar with links: Home, Activities, Library, Help, [User Menu].
- **Rationale:** Easy access to core features.

#### IR-UI-03: User Menu
- **Requirement:** Dropdown menu (click user avatar) with: My Profile (intervals.icu), Settings (future), Logout.
- **Rationale:** Standard web app pattern.

#### IR-UI-04: Loading States
- **Requirement:** Spinner or skeleton loaders during async operations.
- **Rationale:** User feedback; perceived performance.

#### IR-UI-05: Error States
- **Requirement:** Friendly error messages with actionable guidance (e.g., "Try again", "Contact support").
- **Rationale:** Reduce user frustration.

### 5.2 API Interface

#### IR-API-01: RESTful Design
- **Requirement:** Use standard HTTP methods (GET, POST, PATCH, DELETE).
- **Rationale:** Predictable, cacheable, widely understood.

#### IR-API-02: JSON Responses
- **Requirement:** All API responses use `Content-Type: application/json`.
- **Rationale:** Easy parsing in JavaScript; standard format.

#### IR-API-03: Error Responses
- **Requirement:** Errors include `{ error: "message", code: "ERROR_CODE" }`.
- **Rationale:** Client can display user-friendly messages or log structured errors.

#### IR-API-04: Pagination
- **Requirement:** List endpoints support `?page=1&perPage=20`.
- **Rationale:** Handle large datasets; reduce response size.

#### IR-API-05: Versioning
- **Requirement:** API routes include version prefix (e.g., `/api/v1/instroke/*`).
- **Rationale:** Allow future breaking changes without disrupting clients.

### 5.3 External Interfaces

#### IR-EXT-01: intervals.icu OAuth
- **Requirement:** Implement OAuth 2.0 authorization code flow.
- **Endpoints:** `GET /oauth/authorize`, `GET /oauth/callback`, `GET /oauth/logout` (reuse existing).

#### IR-EXT-02: intervals.icu Activity API
- **Requirement:** Fetch activities via `GET /api/v1/athlete/{id}/activities`.
- **Parameters:** `oldest`, `newest` (date range), `type` (filter to rowing).

#### IR-EXT-03: intervals.icu FIT Download
- **Requirement:** Download FIT file via `GET /api/v1/activity/{id}/fit`.
- **Authorization:** Bearer token from OAuth.

#### IR-EXT-04: intervals.icu Streams API
- **Requirement:** Fetch GPS streams via `GET /api/v1/activity/{id}/streams`.
- **Streams:** latlng, time (used for GPS validation, if needed).

---

## 6. Constraints and Assumptions

### 6.1 Technical Constraints

#### CON-TECH-01: Cloudflare Free Tier
- **Constraint:** Must operate within free tier limits (100k requests/day, 10GB D1 storage).
- **Mitigation:** Rate limiting, user quotas, auto-archive old data.

#### CON-TECH-02: Browser Compatibility
- **Constraint:** No native app; web-only (progressive web app in future).
- **Mitigation:** Responsive design, mobile-optimized UI.

#### CON-TECH-03: FIT Parsing Library
- **Constraint:** JavaScript FIT parser may be slower than native parsers (C++, Rust).
- **Mitigation:** Parse server-side (worker); cache results in D1.

### 6.2 Business Constraints

#### CON-BIZ-01: Open Source
- **Constraint:** All code must be open-source (MIT license).
- **Rationale:** Community maintenance, no vendor lock-in.

#### CON-BIZ-02: Zero Hosting Cost (Goal)
- **Constraint:** Aim for $0/month hosting cost (Cloudflare free tier).
- **Mitigation:** If free tier exceeded, accept donations or upgrade to paid tier ($5/month).

#### CON-BIZ-03: No Monetization
- **Constraint:** No paid features, subscriptions, or ads.
- **Rationale:** Community-first; avoid conflicts of interest.

### 6.3 Assumptions

#### ASSUME-01: intervals.icu API Stability
- **Assumption:** intervals.icu OAuth and activity API remain stable.
- **Risk:** If API changes, app breaks. Mitigation: Monitor intervals.icu forum, maintain API client with version checks.

#### ASSUME-02: FIT File Availability
- **Assumption:** intervals.icu allows downloading FIT files via API.
- **Risk:** If rate-limited or blocked, ingestion fails. Mitigation: Implement caching, respect rate limits.

#### ASSUME-03: Rowsandall Export Format
- **Assumption:** Rowsandall exports ZIP with documented schema.
- **Risk:** If schema differs, import fails. Mitigation: Work with Rowsandall maintainers to finalize schema before shutdown.

#### ASSUME-04: Community Contributions
- **Assumption:** Rowsandall users will contribute to development and testing.
- **Risk:** Low engagement → slower development. Mitigation: Clear contribution guidelines, welcoming community culture.

---

## 7. Acceptance Criteria (MVP)

### 7.1 MVP Definition

**Minimum Viable Product (MVP)** includes core functionality for single-workout analysis and basic comparison:

- [x] OAuth login via intervals.icu
- [x] Activity browser (last 3 months, OTW rowing)
- [x] FIT ingestion (parse force curves, oarlock data)
- [x] Single workout analysis (filter by spm, chart force curve)
- [x] Save analysis (with name)
- [x] Analysis library (list, delete)
- [x] Compare 2 analyses (overlay force curves)
- [x] Share analysis (public link)

**Out of MVP (Phase 2):**
- Boat acceleration, seat curves (Quiske, RP3)
- Trend analysis (historical comparison)
- Rowsandall import tool
- Mobile-optimized UI
- Video overlay

### 7.2 MVP Acceptance Tests

#### TEST-MVP-01: End-to-End User Flow
1. User signs in via intervals.icu OAuth → Redirected to activity browser
2. User selects OTW rowing activity → POST `/workouts/ingest` → Success toast
3. User redirected to analysis page → Force curve displayed
4. User adjusts spm slider (28-32) → Chart updates
5. User saves analysis ("Race 1") → Success notification
6. User navigates to library → "Race 1" listed
7. User ingests second workout, saves analysis ("Race 2")
8. User selects both analyses, clicks "Compare" → Comparison page shows 2 overlaid curves
9. User clicks "Share" → Share link copied to clipboard
10. User opens share link in incognito tab → Read-only analysis displayed

**Expected Result:** All steps complete without errors; charts render correctly; data persists across sessions.

#### TEST-MVP-02: Error Handling
1. User selects activity with no FIT file → 404 error, message: "FIT file not found"
2. User selects activity with malformed FIT → 422 error, message: "FIT parsing failed at stroke 42"
3. User tries to ingest 11th workout in 10 minutes → 429 error, message: "Rate limit exceeded. Try again in 5 minutes."

**Expected Result:** Errors displayed clearly; app does not crash.

#### TEST-MVP-03: Performance
1. User ingests 30-minute workout (855 strokes) → Completes in <10 seconds
2. User filters to 32 spm (42 strokes) → Chart renders in <500ms
3. User compares 3 analyses → Page loads in <3 seconds

**Expected Result:** All operations meet performance requirements.

---

## 8. Traceability Matrix

Map requirements to architecture components and test cases.

| Requirement | Architecture Component | Test Case |
|-------------|------------------------|-----------|
| FR-AUTH-01 | OAuth flow (index.ts) | TEST-MVP-01 step 1 |
| FR-INGEST-02 | `fetchIntervalsActivity()` | TEST-MVP-02 step 1 |
| FR-INGEST-03 | `instroke-ingest.ts` | TEST-MVP-02 step 2 |
| FR-ANALYZE-03 | `instroke-chart.js` | TEST-MVP-01 step 4 |
| FR-ANALYZE-06 | POST `/analyses` | TEST-MVP-01 step 5 |
| FR-COMPARE-01 | `instroke-compare.ts` | TEST-MVP-01 step 8 |
| FR-SHARE-01 | POST `/share` | TEST-MVP-01 step 9 |
| NFR-PERF-02 | Worker performance | TEST-MVP-03 step 1 |

---

## 9. Review and Approval

### 9.1 Stakeholders

| Role | Name | Responsibility |
|------|------|----------------|
| Product Owner | Sander Roosendaal | Approve requirements, prioritize features |
| Technical Lead | TBD (community) | Architecture review, technical decisions |
| UX Designer | TBD (community) | UI/UX review, accessibility |
| QA Lead | TBD (community) | Test plan, acceptance testing |
| Community | Rowsandall users | Feedback, user acceptance |

### 9.2 Review Process

1. **Draft Review (April 2026):** Share this document in GitHub discussion; gather feedback
2. **Refinement (May 2026):** Incorporate feedback; finalize MVP scope
3. **Approval (May 2026):** Product owner approves requirements; development begins
4. **Change Management:** Requirements changes require GitHub issue + maintainer approval

### 9.3 Open Questions

1. **Should we support video overlay in MVP or defer to Phase 2?**
   - **Recommendation:** Defer to Phase 2 (high complexity, low MVP priority).

2. **Should analyses be editable (update filters) or immutable (create new)?**
   - **Recommendation:** Immutable (simpler; user creates new analysis to iterate).

3. **Should we auto-import workouts from intervals.icu (webhook) or manual only?**
   - **Recommendation:** Manual only for MVP (webhooks require intervals.icu integration).

4. **Default privacy: Public or private analyses?**
   - **Recommendation:** Private (user opts in to sharing; respects privacy).

5. **Should we support dual oarlock (port/starboard) in MVP?**
   - **Recommendation:** Yes (FIT spec already supports it; minor parser extension).

---

## 10. Glossary

| Term | Definition |
|------|------------|
| **Abscissa** | X-axis of force curve (time, handle distance, or normalized 0-1) |
| **Analysis** | User-created snapshot of filtered strokes with computed metrics |
| **Curve Type** | Type of in-stroke data (force, boat acceleration, seat position, oar angle velocity) |
| **Developer Field** | Custom FIT field defined by app (e.g., rowingdata) |
| **FIT File** | Garmin Flexible and Interoperable Data Transfer format (binary workout file) |
| **Force Curve** | Handle force vs. drive phase (in-stroke metric) |
| **Ingestion** | Process of downloading, parsing, and storing FIT file |
| **Oarlock** | Smart sensor on oar that measures force, angle, and timing |
| **OTW** | On-the-water (rowing on rivers/lakes, not indoor erg) |
| **Quartiles** | Q1 (25th percentile), Q2 (median), Q3 (75th percentile) of curve values |
| **SPM** | Strokes per minute (rowing cadence) |
| **UUID** | Universally Unique Identifier (used for share links) |

---

**Prepared by:** AI Assistant (Cursor)  
**Approved by:** [Pending community review]  
**Next Review:** May 2026 (post-MVP)
