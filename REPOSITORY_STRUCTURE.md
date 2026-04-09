# Repository Structure & Implementation Plan

**Date:** April 9, 2026

---

## Where Does the Code Go?

### This Repository (`instrokedata`)
**Purpose:** **Proposal and planning documents only** — not the actual implementation

**Contents:**
- Architecture proposal
- Requirements specification
- Technical reference
- Migration guides
- RFC for community discussion

**Think of this as:** The blueprint and specification documents before building starts.

---

## Implementation Repositories

Since we're recommending **Option A (Extend rownative/worker)**, the actual implementation will happen in the existing rownative repositories:

### 1. `rownative/worker` — Backend API
**Repository:** https://github.com/rownative/worker  
**Language:** TypeScript  
**Platform:** Cloudflare Workers

**New code location:**
```
worker/
├── src/
│   ├── instroke-ingest.ts      # NEW: FIT parsing & ingestion
│   ├── instroke-analysis.ts    # NEW: Analysis engine
│   ├── instroke-compare.ts     # NEW: Multi-analysis comparison
│   ├── instroke-share.ts       # NEW: Shareable link generation
│   ├── instroke-library.ts     # NEW: User's saved analyses
│   └── index.ts                # MODIFIED: Add /api/instroke/* routes
├── test/
│   └── instroke/               # NEW: Test files
│       ├── ingest.test.ts
│       ├── analysis.test.ts
│       └── compare.test.ts
└── wrangler.jsonc              # MODIFIED: Add D1 tables, KV bindings
```

**New API routes:**
- `POST /api/instroke/workouts/ingest` — Import FIT from intervals.icu
- `GET /api/instroke/workouts` — List user's workouts
- `POST /api/instroke/analyses` — Create analysis with filters
- `GET /api/instroke/analyses/:id` — Get analysis + chart data
- `POST /api/instroke/compare` — Compare multiple analyses
- `POST /api/instroke/share` — Generate shareable link
- `GET /api/instroke/library` — User's saved analyses

**Database migrations:**
```sql
-- migrations/0003_add_instroke_tables.sql
CREATE TABLE instroke_workouts (...);
CREATE TABLE instroke_curves (...);
CREATE TABLE instroke_analyses (...);
CREATE TABLE instroke_analysis_metrics (...);
CREATE TABLE instroke_shares (...);
```

---

### 2. `rownative/courses` — Frontend UI
**Repository:** https://github.com/rownative/courses  
**Language:** HTML5 + Vanilla JavaScript + D3.js  
**Platform:** GitHub Pages (Cloudflare Pages)

**New code location:**
```
courses/
├── site/
│   ├── instroke/                    # NEW: All instroke UI
│   │   ├── index.html               # Activity browser
│   │   ├── analyze.html             # Single workout analysis
│   │   ├── compare.html             # Multi-analysis comparison
│   │   ├── library.html             # Saved analyses
│   │   ├── shared.html              # Public shared view
│   │   ├── help.html                # Documentation
│   │   ├── js/
│   │   │   ├── charts/              # D3.js charts (from Rowsandall)
│   │   │   │   ├── instroke.js      # In-stroke analysis chart
│   │   │   │   ├── forcecurve.js    # Force curve with sliders
│   │   │   │   └── compare.js       # Multi-analysis comparison
│   │   │   ├── instroke-api.js      # API client
│   │   │   ├── instroke-utils.js    # Filters, normalization
│   │   │   └── instroke-fit.js      # Client-side FIT parsing (optional)
│   │   └── css/
│   │       └── instroke.css         # Styles
│   ├── courses.html                 # MODIFIED: Add "Analyze Force Curves" link
│   └── ...                          # Existing files unchanged
└── README.md                        # MODIFIED: Document instroke feature
```

**Navigation integration:**
```html
<!-- courses.html - add link to instroke -->
<nav>
  <a href="courses.html">Courses</a>
  <a href="challenges.html">Challenges</a>
  <a href="instroke/index.html">Force Curves</a>  <!-- NEW -->
</nav>
```

---

## Development Workflow

### Phase 1: Backend API (rownative/worker)

1. **Create feature branch:**
   ```bash
   cd ../worker
   git checkout -b feature/instroke-api
   ```

2. **Add D1 migrations:**
   ```bash
   # Create migration file
   npx wrangler d1 migrations create instroke-db add_instroke_tables
   
   # Edit migrations/0003_add_instroke_tables.sql
   # Add CREATE TABLE statements
   
   # Apply migration to local dev
   npx wrangler d1 migrations apply instroke-db --local
   ```

3. **Implement modules:**
   ```bash
   # Create new TypeScript files
   touch src/instroke-ingest.ts
   touch src/instroke-analysis.ts
   touch src/instroke-compare.ts
   touch src/instroke-share.ts
   touch src/instroke-library.ts
   ```

4. **Update router in `src/index.ts`:**
   ```typescript
   // Add instroke routes
   if (url.pathname.startsWith('/api/instroke/')) {
     if (url.pathname === '/api/instroke/workouts/ingest') {
       return handleInstrokeIngest(request, env);
     }
     if (url.pathname.startsWith('/api/instroke/analyses')) {
       return handleInstrokeAnalyses(request, env);
     }
     // ... more routes
   }
   ```

5. **Test locally:**
   ```bash
   npm test
   npm run dev
   curl http://localhost:8787/api/instroke/workouts
   ```

6. **Deploy to staging:**
   ```bash
   npx wrangler deploy --env staging
   ```

---

### Phase 2: Frontend UI (rownative/courses)

1. **Create feature branch:**
   ```bash
   cd ../courses
   git checkout -b feature/instroke-ui
   ```

2. **Extract D3.js templates from Rowsandall:**
   ```bash
   mkdir -p site/instroke/js/charts
   
   # Download Rowsandall templates
   curl https://git.wereldraadsel.tech/sander/rowsandall-charts/raw/branch/main/templates/instroke.js \
     -o site/instroke/js/charts/instroke-original.js
   
   curl https://git.wereldraadsel.tech/sander/rowsandall-charts/raw/branch/main/templates/forcecurve.js \
     -o site/instroke/js/charts/forcecurve-original.js
   ```

3. **Adapt D3.js templates:**
   ```bash
   # Manually edit to remove Go template syntax
   # See D3JS_MIGRATION_GUIDE.md for instructions
   ```

4. **Create HTML pages:**
   ```bash
   touch site/instroke/index.html
   touch site/instroke/analyze.html
   touch site/instroke/compare.html
   touch site/instroke/library.html
   ```

5. **Test locally:**
   ```bash
   # Serve locally
   python3 -m http.server 8000 --directory site
   
   # Open http://localhost:8000/instroke/
   ```

6. **Deploy to GitHub Pages:**
   ```bash
   git add site/instroke/
   git commit -m "Add in-stroke analysis UI"
   git push origin feature/instroke-ui
   
   # Create PR, review, merge to main
   # GitHub Actions will auto-deploy to rownative.icu
   ```

---

### Phase 3: Integration & Testing

1. **Cross-link from courses page:**
   ```html
   <!-- site/courses.html -->
   <div class="course-time-card">
     <h3>2k Row - 7:15.3</h3>
     <p>Date: 2026-03-15</p>
     <a href="instroke/analyze.html?activityId=12345">Analyze Force Curves →</a>
   </div>
   ```

2. **Test end-to-end flow:**
   - User logs in via intervals.icu OAuth (existing flow)
   - User navigates to "Force Curves" tab
   - User selects activity from intervals.icu
   - Worker fetches FIT file, parses, stores in D1
   - Frontend renders D3.js chart with data from API
   - User adjusts sliders, creates analysis, saves
   - User compares multiple analyses
   - User generates shareable link

3. **Beta testing:**
   - Deploy to staging environment
   - Invite 5-10 beta testers (Rowsandall users)
   - Collect feedback, fix bugs
   - Iterate

---

## File Synchronization

**Question:** If we work in separate repos, how do UI and API stay in sync?

**Answer:** They don't need to stay in perfect sync during development, but you need coordination at PR time:

### Development Process:

1. **Define API contract first** (OpenAPI spec or TypeScript types)
2. **Backend team** implements API in `rownative/worker`
3. **Frontend team** implements UI in `rownative/courses` (can use mock data initially)
4. **Integration testing** when both PRs are ready
5. **Deploy backend first**, then frontend (or simultaneously if using feature flags)

### API Versioning:

```typescript
// src/instroke-api.ts
const API_VERSION = 'v1';

// Frontend can check version
fetch('/api/instroke/version')
  .then(res => res.json())
  .then(data => {
    if (data.version !== 'v1') {
      console.warn('API version mismatch');
    }
  });
```

### TypeScript Type Sharing (Optional):

Create a shared types package:

```bash
# Option 1: Copy types manually
cp ../worker/src/types/instroke.ts ../courses/site/instroke/js/types.ts

# Option 2: Publish to npm (overkill for small project)
# Option 3: Git submodule (complex)
# Option 4: Just duplicate and keep in sync (simplest)
```

**Recommendation:** Keep it simple — duplicate types in both repos, coordinate at PR review time.

---

## Repository Organization Options

### Option 1: Keep Separate (Current Approach) ✅ RECOMMENDED

**Pros:**
- Existing structure, no migration needed
- Clear separation of concerns (backend vs frontend)
- Independent deployment (backend can update without frontend redeploy)
- Smaller PR scope (easier to review)

**Cons:**
- Need to coordinate PRs across repos
- Potential for API/UI mismatch during development

**Mitigation:**
- Define API contract upfront (OpenAPI spec)
- Feature flags to enable/disable instroke until both sides ready
- Staging environment for integration testing

---

### Option 2: Monorepo (Not Recommended for Now)

Move everything to one repo:

```bash
rownative/
├── packages/
│   ├── worker/      # Backend (existing rownative/worker)
│   ├── courses/     # Frontend (existing rownative/courses)
│   └── shared/      # Shared types
├── package.json     # Workspace root
└── turbo.json       # Turborepo config (if using)
```

**Pros:**
- Atomic commits (UI + API changes in one PR)
- Easy type sharing
- Unified CI/CD

**Cons:**
- Large migration effort (move existing repos)
- Larger PRs (harder to review)
- More complex deployment (need to selectively deploy changed packages)
- Not standard for Cloudflare Workers + GitHub Pages projects

**Verdict:** Not worth the migration cost for this project.

---

## This Proposal Repository

**What happens to the `instrokedata` folder?**

**Option A:** Archive after implementation starts
- Move to a `docs/` folder in `rownative/worker` or `rownative/courses`
- Keep as historical reference

**Option B:** Keep as separate "RFC" repository
- Useful if you want a central place for design docs
- Can reference from implementation repos

**Option C:** Copy relevant docs to implementation repos
- `ARCHITECTURE.md` → `rownative/worker/docs/INSTROKE_ARCHITECTURE.md`
- `D3JS_MIGRATION_GUIDE.md` → `rownative/courses/docs/D3JS_MIGRATION.md`
- `REQUIREMENTS.md` → `rownative/worker/docs/INSTROKE_REQUIREMENTS.md`

**Recommendation:** **Option C** — copy relevant docs to implementation repos so developers can find them easily.

---

## Summary

| Concern | Answer |
|---------|--------|
| **Where does backend code go?** | `rownative/worker` repository |
| **Where does frontend code go?** | `rownative/courses` repository |
| **What's this `instrokedata` folder?** | Proposal docs only (blueprint before building) |
| **How do we keep API/UI in sync?** | Define API contract upfront, coordinate PRs, use feature flags |
| **Should we use a monorepo?** | No — too much migration effort, not worth it |
| **Where do these docs end up?** | Copy to `docs/` folders in implementation repos |

**Bottom line:** The actual coding happens in the existing `rownative/worker` and `rownative/courses` repositories. This `instrokedata` folder is just the planning/proposal phase.

---

**Next step:** Once community approves the proposal, create implementation issues in the actual repositories:
- `rownative/worker` Issue: "Implement in-stroke API backend"
- `rownative/courses` Issue: "Implement in-stroke UI frontend"
