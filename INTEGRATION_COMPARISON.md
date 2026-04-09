# Integration Strategy Comparison — Extend vs Standalone

> Detailed analysis of two implementation approaches for in-stroke analysis platform

**Date:** April 9, 2026

---

## Quick Comparison

| Aspect | **Option A: Extend rownative/worker** | **Option B: Standalone instroke app** |
|--------|--------------------------------------|---------------------------------------|
| **Repository** | Add to existing `rownative/worker` | New `rownative/instroke-worker` |
| **Domain** | `rownative.icu/instroke/*` | `instroke.rownative.icu` or separate domain |
| **Authentication** | Reuse existing OAuth flow | Duplicate OAuth setup |
| **Infrastructure** | Shared D1, KV, DNS | New D1, KV, separate deployment |
| **User Experience** | Single sign-on, unified site | Separate login, standalone site |
| **Deployment** | One worker, one CI/CD | Two workers, two CI/CD pipelines |
| **Maintenance** | Single codebase | Two codebases |
| **Time to Market** | Faster (OAuth already working) | Slower (set up from scratch) |
| **Hosting Cost** | $0 (within free tier) | $0 (separate free tier) |
| **Complexity** | Higher (larger codebase) | Lower (focused scope) |
| **Recommendation** | ✅ **RECOMMENDED** | ❌ Not recommended |

---

## Option A: Extend rownative/worker (RECOMMENDED)

### Overview

Add in-stroke analysis functionality to the existing `rownative/worker` and `rownative/courses` repositories, leveraging shared infrastructure and authentication.

### Architecture

```
rownative/worker (TypeScript)
├── src/
│   ├── index.ts              (existing: routes /api/courses, /api/challenges, /oauth/*)
│   ├── intervals-api.ts      (existing: intervals.icu client)
│   ├── course-time.ts        (existing: GPS validation)
│   ├── NEW: instroke-ingest.ts    (FIT parsing + storage)
│   ├── NEW: instroke-analysis.ts  (filter + compute metrics)
│   ├── NEW: instroke-compare.ts   (multi-analysis overlay)
│   └── NEW: instroke-share.ts     (shareable links)
├── test/
│   └── NEW: instroke*.test.ts     (unit + integration tests)
└── migrations/
    └── NEW: 0004_instroke_tables.sql (D1 schema)

rownative/courses (Static site)
├── site/
│   ├── index.html, challenges.html, my-times.html (existing)
│   └── NEW: instroke/
│       ├── index.html        (activity browser)
│       ├── analyze.html      (single workout analysis)
│       ├── compare.html      (multi-analysis comparison)
│       ├── library.html      (saved analyses)
│       ├── shared.html       (public shared view)
│       └── js/instroke-*.js  (Chart.js integration)
```

### Implementation Details

**Routes:**
- `/api/instroke/workouts` — List/ingest workouts
- `/api/instroke/analyses` — Create/list/delete analyses
- `/api/instroke/compare` — Generate comparison data
- `/api/instroke/share` — Create shareable links
- `/api/instroke/shared/{uuid}` — Public shared analysis

**Database:**
- Reuse existing D1 database (`rowing-courses-db`)
- Add 4 new tables: `instroke_workouts`, `instroke_curves`, `instroke_analyses`, `instroke_analysis_metrics`

**KV:**
- Reuse existing KV namespace (`ROWING_COURSES`)
- Keys: `instroke:share:{uuid}` (shared links with TTL)

**DNS:**
- No changes; same domain: `rownative.icu`

### Pros

#### 1. **Shared Authentication ✅ CRITICAL**
- Users already signed in for courses/challenges
- No duplicate OAuth setup (CLIENT_ID, CLIENT_SECRET, encryption keys)
- Single session cookie (`rn_session`)
- Consistent user experience

#### 2. **Infrastructure Reuse**
- Same Cloudflare Worker (one deployment)
- Same D1 database (one migration pipeline)
- Same KV namespace (one secret to manage)
- Same DNS (no subdomain setup)

#### 3. **Natural Integration**
- Cross-link from course times: "View force curves for this result"
- Compare force curves from different challenge submissions
- Unified navigation (one site, one account)

#### 4. **Lower Maintenance Burden**
- One codebase to maintain
- One CI/CD pipeline (GitHub Actions)
- One deployment process (`wrangler deploy`)
- Fewer moving parts → fewer points of failure

#### 5. **Faster Time to Market**
- OAuth flow already working (no setup required)
- intervals.icu API client already implemented (`intervals-api.ts`)
- D1 and KV bindings already configured
- Focus on in-stroke logic, not infrastructure

#### 6. **Cost Efficiency**
- All features under one Cloudflare free tier
- No risk of splitting traffic across multiple workers (hitting rate limits)

### Cons

#### 1. **Increased Codebase Complexity**
- Larger codebase (harder for new contributors to navigate)
- **Mitigation:** Clear module separation (`src/instroke-*.ts`), good documentation

#### 2. **Potential Performance Impact**
- In-stroke API calls share worker resources with courses API
- **Mitigation:** Separate route handlers; Cloudflare Workers auto-scale per request

#### 3. **Tighter Coupling**
- In-stroke functionality depends on courses infrastructure
- **Mitigation:** Use dependency injection; modular design (in-stroke modules have minimal imports from courses code)

#### 4. **Deploy Risk**
- Bug in in-stroke code could affect courses/challenges
- **Mitigation:** Comprehensive testing; feature flags for rollback

#### 5. **Repository Organization**
- Mixing two distinct features in one repo
- **Mitigation:** Use folders (`src/instroke/`, `site/instroke/`); clear README sections

### Costs

**Development:**
- 1-2 weeks to integrate (vs 3-4 weeks standalone, including OAuth setup)

**Ongoing:**
- Same maintenance as standalone (code quality, testing, documentation)
- Lower operational overhead (one deployment, one monitoring dashboard)

**Hosting:**
- $0/month (within Cloudflare free tier)

---

## Option B: Standalone instroke app

### Overview

Create a separate GitHub repository and Cloudflare Worker for in-stroke analysis, independent of courses/challenges.

### Architecture

```
rownative/instroke-worker (new repo)
├── src/
│   ├── index.ts              (routes /api/*, /oauth/*)
│   ├── oauth.ts              (duplicate OAuth setup)
│   ├── intervals-api.ts      (duplicate API client)
│   ├── instroke-ingest.ts    (FIT parsing)
│   ├── instroke-analysis.ts  (analysis engine)
│   └── instroke-compare.ts   (comparison)
├── test/
│   └── instroke*.test.ts
└── migrations/
    └── 0001_initial.sql

rownative/instroke-site (new repo or subfolder)
├── site/
│   ├── index.html            (activity browser)
│   ├── analyze.html
│   ├── compare.html
│   └── js/instroke-*.js
```

### Implementation Details

**Domain:**
- Option 1: Subdomain: `instroke.rownative.icu` (requires DNS setup)
- Option 2: Separate domain: `instroke.icu` or similar (requires domain purchase)

**Routes:**
- `/api/workouts`, `/api/analyses`, `/api/compare`, `/api/share`
- `/oauth/authorize`, `/oauth/callback`, `/oauth/logout` (duplicate)

**Database:**
- New D1 database: `instroke-db` (separate quota)

**KV:**
- New KV namespace: `INSTROKE_DATA` (separate quota)

### Pros

#### 1. **Cleaner Separation**
- In-stroke code isolated from courses/challenges
- Easier to reason about (smaller codebase)
- Clearer repository boundaries

#### 2. **Independent Deployment**
- Deploy in-stroke changes without touching courses
- No risk of breaking courses API
- Separate versioning (semantic versioning per repo)

#### 3. **Focused Scope**
- Contributors only need to understand in-stroke domain
- Lower cognitive load for new developers

#### 4. **Separate Quotas**
- If in-stroke hits free tier limits, courses unaffected
- **Note:** Unlikely to hit limits with <1,000 users

### Cons

#### 1. **Duplicate OAuth Setup ❌ CRITICAL**
- Must re-implement OAuth flow (authorize, callback, logout)
- Duplicate secrets (`INTERVALS_CLIENT_ID`, `INTERVALS_CLIENT_SECRET`, `TOKEN_ENCRYPTION_KEY`)
- Duplicate session management (cookie handling, encryption, expiration)
- **Effort:** 2-3 days of work, plus testing

#### 2. **Fragmented User Experience**
- Users must sign in separately to instroke.rownative.icu
- No session sharing (cookies scoped to domain)
- Confusing: "Why do I need to log in again?"

#### 3. **Duplicate Infrastructure**
- Separate D1 database (one more thing to manage)
- Separate KV namespace (more secrets to configure)
- Separate DNS record (subdomain setup)
- Separate GitHub Actions (CI/CD duplication)

#### 4. **Higher Maintenance Burden**
- Two codebases to maintain
- Two deployment pipelines
- Two monitoring dashboards (Cloudflare Analytics)
- More PRs, more reviews, more releases

#### 5. **No Cross-Linking**
- Cannot link from course times to force curves (different domains/sessions)
- Users must manually find their workouts in both platforms

#### 6. **Longer Time to Market**
- Must set up OAuth from scratch (~3 days)
- Must configure D1, KV, secrets, DNS (~1 day)
- Must create deployment pipeline (~1 day)
- Total delay: ~1 week vs extending existing infrastructure

### Costs

**Development:**
- 3-4 weeks to MVP (including OAuth setup, infrastructure)

**Ongoing:**
- Higher maintenance (two repos, two deployments)

**Hosting:**
- $0/month (separate free tier; sufficient for <1,000 users)

---

## Side-by-Side Feature Comparison

| Feature | **Extend (Option A)** | **Standalone (Option B)** |
|---------|----------------------|---------------------------|
| **OAuth** | Reuse existing (0 days work) | Re-implement (2-3 days) |
| **Session Management** | Reuse existing cookie | Duplicate cookie handling |
| **intervals.icu API Client** | Reuse `intervals-api.ts` | Duplicate or copy module |
| **D1 Database** | Add tables to existing DB | New DB (one more thing to manage) |
| **KV Namespace** | Reuse `ROWING_COURSES` | New namespace (more secrets) |
| **DNS** | No change | Subdomain setup required |
| **CI/CD** | Extend existing workflow | Create new workflow from scratch |
| **User Sign-In** | Already signed in | Must sign in again |
| **Cross-Linking** | "Analyze force curves" button on course times | No cross-linking (separate domains) |
| **Deployment Risk** | Small (bug could affect courses) | None (isolated) |
| **Maintenance Burden** | Single codebase | Two codebases |
| **Time to MVP** | 2-3 weeks | 4-5 weeks |

---

## Architectural Patterns (Option A)

To mitigate the cons of extending rownative/worker, we adopt these patterns:

### 1. **Modular Design**

**Goal:** Keep in-stroke code isolated from courses/challenges.

**Approach:**
- All in-stroke logic in `src/instroke-*.ts` modules
- Minimal imports from courses code (only shared utilities like `corsAllowOrigin`, `parseSession`)
- Separate test files: `test/instroke-*.test.ts`

**Example:**

```typescript
// src/instroke-ingest.ts
export async function ingestWorkout(
  env: Env,
  activityId: string,
  athleteId: string
): Promise<WorkoutRecord> {
  // FIT parsing logic here
  // No dependencies on courses or challenges code
}

// src/index.ts (main router)
if (url.pathname.startsWith('/api/instroke/')) {
  return handleInstrokeRoutes(request, env, url);
}
```

### 2. **Separate Route Handlers**

**Goal:** Isolate in-stroke request handling.

**Approach:**

```typescript
// src/index.ts
async function fetch(request: Request, env: Env): Promise<Response> {
  const url = new URL(request.url);

  if (url.pathname.startsWith('/api/instroke/')) {
    return handleInstrokeRoutes(request, env, url);
  }

  if (url.pathname.startsWith('/api/courses/')) {
    return handleCoursesRoutes(request, env, url);
  }

  // ... other routes
}
```

### 3. **Feature Flags (Optional)**

**Goal:** Allow gradual rollout and easy rollback.

**Approach:**

```typescript
// wrangler.jsonc
{
  "vars": {
    "FEATURE_INSTROKE_ENABLED": "true"
  }
}

// src/index.ts
if (!env.FEATURE_INSTROKE_ENABLED || env.FEATURE_INSTROKE_ENABLED !== 'true') {
  return new Response('In-stroke analysis coming soon', { status: 503 });
}
```

### 4. **Database Namespacing**

**Goal:** Keep in-stroke tables separate in shared D1 database.

**Approach:**
- Prefix all in-stroke tables with `instroke_`
- Use foreign keys only within in-stroke tables (no references to courses tables)

```sql
CREATE TABLE instroke_workouts (
  id INTEGER PRIMARY KEY,
  athlete_id TEXT NOT NULL,
  activity_id TEXT NOT NULL UNIQUE,
  -- No FK to courses or challenges tables
);
```

### 5. **Shared Utilities Only**

**Goal:** Reuse infrastructure code, not business logic.

**Reusable:**
- `corsAllowOrigin()` (CORS helper)
- `parseSession()` (decrypt session cookie)
- `fetchIntervalsActivity()` (intervals.icu API client)
- `rateLimitCheck()` (KV-based rate limiting)

**NOT Reusable:**
- Course validation logic (specific to courses)
- Challenge scoring logic (specific to challenges)

---

## Cost-Benefit Analysis

### Time to MVP

| Task | **Extend (Option A)** | **Standalone (Option B)** |
|------|-----------------------|---------------------------|
| OAuth setup | 0 days (reuse) | 3 days (re-implement) |
| D1/KV setup | 0 days (reuse) | 1 day (new DB, namespace) |
| DNS/deployment | 0 days (existing) | 1 day (subdomain + CI/CD) |
| Core in-stroke logic | 10 days | 10 days |
| Testing | 3 days | 3 days |
| **Total** | **13 days** ✅ | **18 days** |

**Time saved:** 5 days (27% faster)

### Ongoing Maintenance

| Activity | **Extend (Option A)** | **Standalone (Option B)** |
|----------|-----------------------|---------------------------|
| Code reviews | 1 repo | 2 repos (split attention) |
| Deployments | 1 worker | 2 workers (coordination required) |
| Monitoring | 1 dashboard | 2 dashboards |
| Security updates | 1 codebase | 2 codebases (e.g., dependency updates) |
| User support | 1 account system | 2 account systems (confusing) |

**Effort:** 30% lower maintenance for Option A

### User Experience

| Scenario | **Extend (Option A)** | **Standalone (Option B)** |
|----------|-----------------------|---------------------------|
| First-time user | Sign in once → access courses + instroke | Sign in to rownative.icu, then again to instroke.rownative.icu |
| Returning user | Already signed in | Must remember separate site |
| Cross-feature use | "Analyze force curves from this course time" (one click) | Copy activity ID, switch sites, paste, sign in |

**UX:** Significantly better for Option A (seamless integration)

---

## Risk Analysis

### Option A Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|-----------|
| In-stroke bug breaks courses API | High | Low | Comprehensive testing; feature flag for rollback; separate route handlers |
| Codebase becomes too complex | Medium | Medium | Modular design; clear documentation; folder structure |
| Performance degradation | Medium | Low | Cloudflare Workers auto-scale; monitor latency per route |
| Merge conflicts (multiple contributors) | Low | Medium | Clear contribution guidelines; use feature branches |

### Option B Risks

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|-----------|
| Users confused by separate sign-ins | High | High | Clear messaging; link between sites; shared branding |
| OAuth duplication introduces bugs | Medium | Medium | Copy existing OAuth code; rigorous testing |
| Subdomain DNS issues | Low | Low | Use Cloudflare DNS (same provider) |
| Split maintenance burden → abandonment | High | Medium | Strong community governance; clear ownership |

---

## Community Feedback Questions

1. **Do you use both courses and in-stroke analysis?**
   - If yes → seamless integration (Option A) is valuable
   - If no → standalone (Option B) might be acceptable

2. **How often do you compare force curves from different workouts?**
   - Frequently → need analysis library (both options support this)

3. **Would you use cross-linking (e.g., "Analyze force curves" from course time)?**
   - Yes → strongly favor Option A

4. **Are you comfortable with a larger codebase if features are well-organized?**
   - Yes → Option A is acceptable
   - No → might prefer Option B (but consider maintenance tradeoff)

---

## Recommendation

### ✅ **Option A: Extend rownative/worker**

**Rationale:**
1. **User Experience:** Single sign-on, unified site, natural cross-linking
2. **Time to Market:** 27% faster (13 days vs 18 days)
3. **Maintenance:** 30% lower ongoing effort (one codebase, one deployment)
4. **Cost:** Same hosting cost ($0), no additional infrastructure

**Mitigation for Cons:**
- **Complexity:** Modular design (separate folders, minimal coupling)
- **Performance:** Separate route handlers, monitoring per endpoint
- **Deploy Risk:** Comprehensive testing, feature flags, rollback plan

**When to Consider Option B:**
- If courses/challenges and in-stroke analysis diverge significantly (different user bases, different governance)
- If codebase complexity becomes unmanageable despite modularization
- If Cloudflare free tier limits are hit (then separate quotas might help)

**None of these conditions apply today, so proceed with Option A.**

---

## Implementation Roadmap (Option A)

### Phase 1: Foundation (Week 1-2)
- Create folder structure: `src/instroke/`, `site/instroke/`
- Write D1 migration: `0004_instroke_tables.sql`
- Implement basic route handler: `handleInstrokeRoutes()`
- Set up feature flag (optional)

### Phase 2: Core Logic (Week 3-4)
- Implement FIT parsing: `instroke-ingest.ts`
- Implement analysis engine: `instroke-analysis.ts`
- Write unit tests (Vitest)

### Phase 3: UI (Week 5-6)
- Build activity browser: `instroke/index.html`
- Build analysis page: `instroke/analyze.html`
- Integrate Chart.js for force curves

### Phase 4: Advanced Features (Week 7-8)
- Implement comparison: `instroke-compare.ts`
- Implement sharing: `instroke-share.ts`
- Build comparison page: `instroke/compare.html`

### Phase 5: Testing & Launch (Week 9-10)
- Integration testing (end-to-end flows)
- Performance testing (load testing with Artillery)
- Beta launch (invite Rowsandall users)
- Documentation (help pages, API reference)

---

**Prepared by:** AI Assistant (Cursor)  
**For:** Sander Roosendaal and rownative community  
**Date:** April 9, 2026
