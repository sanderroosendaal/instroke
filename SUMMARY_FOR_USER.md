# Project Summary — High-Level Architecture & Requirements

**For:** Sander Roosendaal  
**Date:** April 9, 2026  
**Status:** Complete — ready for community review

---

## What I've Created

I've prepared a comprehensive proposal for an **in-stroke analysis platform** to replace Rowsandall.com functionality before its shutdown. This includes:

### 📚 Seven Complete Documents

1. **[README.md](./README.md)** — Overview and navigation guide
2. **[EXECUTIVE_SUMMARY.md](./EXECUTIVE_SUMMARY.md)** — Quick decision guide (15 pages)
3. **[ARCHITECTURE.md](./ARCHITECTURE.md)** — Complete technical architecture (70+ pages)
4. **[REQUIREMENTS.md](./REQUIREMENTS.md)** — Functional + non-functional requirements (80+ pages)
5. **[INTEGRATION_COMPARISON.md](./INTEGRATION_COMPARISON.md)** — Extend vs standalone analysis (35+ pages)
6. **[TECHNICAL_REFERENCE.md](./TECHNICAL_REFERENCE.md)** — Developer quick reference (20+ pages)
7. **[GITHUB_DISCUSSION_POST.md](./GITHUB_DISCUSSION_POST.md)** — Ready-to-post RFC for community

**Total:** ~220 pages of comprehensive documentation (no implementation code, as requested).

---

## Key Recommendation

### ✅ **Extend rownative/worker (Option A)**

**Add in-stroke analysis to existing rownative.icu platform rather than creating a standalone app.**

**Why:**
1. **Better UX:** Single sign-on via intervals.icu OAuth (users already authenticated)
2. **Faster:** 27% faster time to MVP (13 days vs 18 days)
3. **Lower maintenance:** One codebase, one deployment pipeline
4. **Natural integration:** Cross-link from course times to force curve analysis
5. **Lower cost:** Same $0/month hosting (within Cloudflare free tier)

**How to implement:**
- Add `/api/instroke/*` routes to existing `rownative/worker` (TypeScript)
- Add `site/instroke/` subdirectory to existing `rownative/courses` (HTML/JS)
- Add 4 new D1 tables via migration (instroke_workouts, instroke_curves, instroke_analyses, instroke_analysis_metrics)
- Modular design: all in-stroke code in separate `src/instroke-*.ts` modules

**Alternative (Option B):** Standalone app — cleaner separation but duplicate OAuth, fragmented UX, higher maintenance. Not recommended.

---

## What the Platform Does

### Core Features (MVP)

1. **Workout Ingestion**
   - Import OTW rowing activities from intervals.icu
   - Download and parse FIT files per rowingdata FIT_EXPORT spec
   - Extract force curves, oarlock data (catch/finish angles), stroke metrics
   - Store in Cloudflare D1 database

2. **Single Workout Analysis**
   - Display force curve (mean + quartile shading)
   - Filter strokes by SPM range, interval, time window
   - Interactive Chart.js chart (zoom, pan, hover tooltips)
   - Statistical summary (avg force, peak force, drive time)
   - Save analysis with user-provided name

3. **Analysis Library**
   - List all saved analyses (table view)
   - Search and filter by curve type, date
   - Manage: view, rename, delete

4. **Multi-Analysis Comparison**
   - Overlay 2-5 force curves on one chart
   - Distinct colors per analysis with legend
   - Toggle individual curves on/off
   - Normalize X-axis (0-1 drive phase) for comparability

5. **Sharing**
   - Generate shareable UUID link
   - Privacy: Public (show athlete name) or Private (anonymous)
   - Expiration: 30/90 days, or never
   - Read-only view for recipients (no login required)

6. **Migration from Rowsandall**
   - Export tool in Rowsandall (ZIP with analyses.json + FIT files)
   - Import endpoint in new platform
   - Progress tracking + error reporting

---

## Sensor Compatibility

| Sensor | Force Curves | Boat Accel | Seat Curves | Oarlock Angles | Status |
|--------|--------------|------------|-------------|----------------|--------|
| **Empower Oarlock** | ✅ Yes | ❌ No | ❌ No | ✅ Yes | **MVP** |
| **RP3 Erg** | ✅ Yes | ❌ No | ✅ Yes | ❌ No | **MVP** |
| **Quiske** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | Phase 2 |
| **Smartphone** | 🟡 Future | 🟡 Future | ❌ No | ❌ No | Phase 3+ |

**MVP Focus:** Force curves (Empower, RP3) — 80% of use cases.  
**Phase 2:** Boat acceleration (Quiske), seat curves, trend analysis.

---

## Technology Stack

**No changes to existing stack — leverage what's already working:**

| Layer | Technology | Cost |
|-------|-----------|------|
| **Backend** | TypeScript (Cloudflare Workers) | Free tier |
| **Database** | Cloudflare D1 (SQLite) | Free tier (10GB) |
| **Cache** | Cloudflare KV | Free tier |
| **Frontend** | HTML5 + Vanilla JS + Chart.js | Free (no build step) |
| **Auth** | intervals.icu OAuth (reuse existing) | Free |
| **Hosting** | Cloudflare Pages (GitHub Actions) | Free |
| **FIT Parsing** | fit-file-parser (npm) | Free |

**Total cost:** $0/month for <1,000 users (sufficient for Rowsandall migration).

---

## Timeline

| Phase | Dates | Deliverables |
|-------|-------|--------------|
| **Community Review** | April 2026 | Feedback on architecture (this proposal) |
| **Prototype** | May 2026 | FIT parsing + force curve chart (proof of concept) |
| **Alpha** | June 2026 | Core ingestion + analysis working (internal testing) |
| **Beta** | July-Aug 2026 | Public beta; Rowsandall export tool ready |
| **Migration** | Sep-Nov 2026 | Parallel operation; user migration from Rowsandall |
| **GA (v1.0)** | Dec 2026 | Full launch; Rowsandall shutdown |

**Critical path:** Need Rowsandall export tool by July 2026 (allow 5 months for user migration).

---

## Architecture Summary

### Data Flow

```
User signs in (intervals.icu OAuth)
  ↓
Selects OTW rowing activity from intervals.icu
  ↓
Worker downloads FIT file (intervals.icu API)
  ↓
Parse FIT: extract curves + oarlock scalars + stroke metrics
  ↓
Store in D1: instroke_workouts + instroke_curves tables
  ↓
User filters strokes (spm sliders, interval dropdown)
  ↓
Compute metrics (quartiles, mean curve, diff)
  ↓
Render Chart.js force curve
  ↓
Save analysis → D1: instroke_analyses + instroke_analysis_metrics
  ↓
Compare analyses → overlay multiple curves
  ↓
Share link → KV: UUID → analysis IDs (public/private)
```

### Database Schema (4 new tables in D1)

1. **instroke_workouts** — Workout metadata (athlete_id, activity_id, date, sport, sensor_type, curve_types)
2. **instroke_curves** — Per-stroke curve data (workout_id, stroke_number, curve_type, curve_data BLOB, oarlock scalars)
3. **instroke_analyses** — Saved analysis snapshots (uuid, workout_id, name, filters, stroke_count, is_public)
4. **instroke_analysis_metrics** — Precomputed metrics (analysis_id, curve_type, metric_name, metric_data BLOB)

**Storage estimate:** 30-minute workout (~855 strokes) ≈ 5MB (curves + metadata). 10GB D1 free tier = ~2,000 workouts.

### API Endpoints (new routes in worker)

```
GET  /api/instroke/workouts           # List user's workouts
POST /api/instroke/workouts/ingest    # Ingest from intervals.icu
GET  /api/instroke/workouts/{id}      # Workout detail

POST   /api/instroke/analyses         # Create analysis
GET    /api/instroke/analyses         # List analyses
GET    /api/instroke/analyses/{id}    # Analysis detail
PATCH  /api/instroke/analyses/{id}    # Update name/description
DELETE /api/instroke/analyses/{id}    # Delete analysis

POST /api/instroke/compare            # Generate comparison data
POST /api/instroke/share              # Create share link
GET  /api/instroke/shared/{uuid}      # Public shared view (no auth)
```

### Frontend Pages (new in site/instroke/)

```
site/instroke/
├── index.html       # Activity browser (list OTW workouts from intervals.icu)
├── analyze.html     # Single workout analysis (filter + chart + save)
├── compare.html     # Multi-analysis comparison (overlay charts)
├── library.html     # User's saved analyses (list + manage)
├── shared.html      # Public shared analysis view (read-only)
├── help.html        # Documentation (FIT format, sensor setup, curve interpretation)
└── js/
    ├── instroke-api.js     # API client (fetch wrapper)
    ├── instroke-chart.js   # Chart.js helpers (render force curves)
    └── instroke-utils.js   # Filters, normalization, stroke selection
```

---

## Open Questions for Community

1. **Integration Strategy:** Extend rownative/worker (Option A) vs standalone app (Option B)?
   - **Recommendation:** Option A (extend)

2. **MVP Scope:** Force curves only, or include boat acceleration (Quiske)?
   - **Recommendation:** Force curves only (faster launch, 80% use case)

3. **Default Privacy:** Public or private analyses?
   - **Recommendation:** Private (user opts in to sharing)

4. **Workout Retention:** Unlimited storage, or auto-archive after 1 year?
   - **Recommendation:** Unlimited for MVP; add archive if needed

5. **Branding:** "Instroke Analysis by rownative.icu" or separate name?
   - **Recommendation:** Use rownative.icu branding

---

## Next Steps

### Immediate (April 2026)
1. ✅ **Complete proposal** (done: 7 documents, 220+ pages)
2. 🔄 **Share with community** (post GITHUB_DISCUSSION_POST.md to GitHub Discussions, Reddit r/Rowing)
3. 🔄 **Gather feedback** (integration strategy, MVP scope, priorities)
4. 🔄 **Validate intervals.icu API** (test FIT download endpoint, rate limits, OAuth scopes)

### Short-term (May 2026)
5. ⏳ **Prototype FIT parsing** (build minimal ingestion + Chart.js force curve)
6. ⏳ **D1 schema design** (write migration script 0004_instroke_tables.sql)
7. ⏳ **UI wireframes** (Figma or paper sketches for analyze.html, compare.html)
8. ⏳ **Recruit contributors** (TypeScript dev, frontend dev, designer, testers)

### Medium-term (June-Aug 2026)
9. ⏳ **Alpha development** (core API + UI working; internal testing)
10. ⏳ **Beta testing** (invite Rowsandall users; gather feedback)
11. ⏳ **Rowsandall export tool** (implement ZIP export in Rowsandall codebase)
12. ⏳ **Migration testing** (test import flow with real Rowsandall data)

---

## How to Use These Documents

### For Community Review

**Share:** [GITHUB_DISCUSSION_POST.md](./GITHUB_DISCUSSION_POST.md)
- Post to GitHub Discussions (rownative/courses)
- Post to Reddit r/Rowing
- Share on rowing forums

**Focus questions:**
- Option A (extend) vs Option B (standalone)?
- MVP scope: force curves only, or boat accel too?
- Privacy defaults: public or private?

### For Developers

**Start with:** [TECHNICAL_REFERENCE.md](./TECHNICAL_REFERENCE.md)
- API endpoints, database schema, code examples
- Quick reference for implementation

**Deep dive:** [ARCHITECTURE.md](./ARCHITECTURE.md)
- Full system design, component details, security, deployment

### For Stakeholders

**Start with:** [EXECUTIVE_SUMMARY.md](./EXECUTIVE_SUMMARY.md)
- 15-page overview with key decisions, timeline, success metrics

**Decision analysis:** [INTEGRATION_COMPARISON.md](./INTEGRATION_COMPARISON.md)
- Detailed pros/cons of extend vs standalone
- Cost-benefit analysis, risk assessment

### For Contributors

**Start with:** [REQUIREMENTS.md](./REQUIREMENTS.md)
- Functional requirements (FR-*), acceptance criteria
- Non-functional requirements (NFR-*), test scenarios

**Implementation:** [TECHNICAL_REFERENCE.md](./TECHNICAL_REFERENCE.md)
- Code examples, testing patterns, contribution workflow

---

## Risks and Mitigations

### Technical Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| FIT parsing errors (sensor incompatibility) | Medium | Test with real FIT files from all sensors; graceful error handling |
| D1 storage limits (10GB free tier) | Low | Monitor usage; implement user export/archive; solicit donations if >10GB needed (~$5/month) |
| intervals.icu API changes | High | Monitor forum; maintain API client with version checks; coordinate with intervals.icu team |
| Chart.js performance (100+ strokes) | Low | Use decimation plugin; lazy-load charts; debounce filter changes |

### Product Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Low user adoption (Rowsandall users don't migrate) | High | Early beta testing; clear migration path; Rowsandall export tool ready by July |
| Community contributor attrition | Medium | Clear contribution guidelines; welcoming culture; regular updates |
| Feature scope creep | Medium | Focus on MVP (force curves); defer boat accel, trend analysis to Phase 2 |

### Operational Risks

| Risk | Impact | Mitigation |
|------|--------|-----------|
| Deploy breaks courses/challenges | High | Comprehensive testing; feature flags for rollback; separate route handlers |
| Cloudflare outage | Medium | No mitigation (accept risk); Cloudflare SLA 99.99% uptime |
| Security vulnerability (XSS, SQLi) | High | Input validation; OWASP best practices; security audit pre-launch |

---

## Success Metrics

**3 months post-launch (March 2027):**

- **Users:** 500 registered (intervals.icu OAuth)
- **Workouts:** 2,000 ingested
- **Analyses:** 5,000 saved
- **Shares:** 500 links generated
- **Performance:** <2s page load, <5s analysis creation, <100ms p50 API latency
- **Reliability:** 99.5% uptime, <5% FIT parsing error rate, zero data loss
- **Community:** 5+ active contributors, 20+ GitHub stars

**12 months post-launch (Dec 2027):**

- **Users:** 2,000 active (1+ analysis per month)
- **Workouts:** 20,000 ingested
- **Analyses:** 50,000 saved
- **Community:** 10+ contributors, 100+ GitHub stars, active Discord/Slack

---

## Cost Estimate

**Development (one-time):**
- Prototype: 1 week (volunteer)
- Alpha: 4 weeks (volunteer)
- Beta: 4 weeks (volunteer)
- **Total:** ~10 weeks volunteer effort (or ~$20k if paid at $50/hour)

**Hosting (ongoing):**
- **$0/month** for <1,000 users (Cloudflare free tier)
- **$5/month** if >1,000 users (D1 paid tier: 50GB storage)

**Maintenance (ongoing):**
- ~5 hours/week for bug fixes, dependency updates, community support
- Volunteer effort (or ~$1,250/month if paid)

**Total cost over 3 years:** $0 (volunteer-run) or ~$45k (paid maintainer + hosting).

**Recommendation:** Volunteer-run, accept donations for hosting if needed ("Buy me a coffee" link).

---

## Comparison with Rowsandall

| Feature | Rowsandall | New Platform |
|---------|------------|--------------|
| Authentication | Django accounts (username/password) | intervals.icu OAuth (no passwords) |
| Hosting | VPS (DigitalOcean, $100/month) | Cloudflare (free tier, $0/month) |
| Database | PostgreSQL (self-hosted) | D1 (SQLite, managed) |
| Force curve analysis | ✅ Yes | ✅ Yes (MVP) |
| Boat acceleration | ✅ Yes | 🟡 Phase 2 |
| Comparison tool | ✅ Yes | ✅ Yes (MVP) |
| Trend analysis | ✅ Yes | 🟡 Phase 2 |
| Video overlay | ❌ No | 🟡 Phase 3+ |
| Maintenance | Single maintainer | Community (open source) |
| Longevity | Shutting down (end 2026) | Community-maintained (indefinite) |

**Key advantages:**
1. **Zero hosting cost** (vs $100/month)
2. **Community-maintained** (vs single maintainer)
3. **intervals.icu integration** (OAuth, activity import)
4. **Open source** (MIT license; anyone can fork/contribute)

---

## What's NOT Included (Future Phases)

**Phase 2 (Q3-Q4 2026):**
- Boat acceleration curves (Quiske)
- Seat position curves (RP3)
- Oarlock symmetry analysis (dual EmPower)
- Trend analysis (compare same workout type over time)
- Export analyses to CSV/JSON

**Phase 3 (2027+):**
- Video overlay (sync force curve with video recording)
- Smartphone sensor integration (custom app uploads)
- Machine learning insights (stroke efficiency scoring)
- CrewNerd integration (real-time analysis on water)
- Coach tools (annotate curves, share with athletes)
- Mobile-optimized UI (responsive design for tablets)

---

## Final Thoughts

This proposal is **comprehensive but not exhaustive**. It's designed to:

1. **Get community feedback** on key decisions (integration strategy, MVP scope)
2. **Provide implementation guidance** for developers (architecture, requirements, technical reference)
3. **Set expectations** for timeline, costs, and success metrics

**This is NOT a specification to be followed blindly.** It's a starting point for discussion. Expect changes based on community feedback, technical constraints, and user needs.

**What I recommend you do next:**
1. Review EXECUTIVE_SUMMARY.md (15 pages, quick read)
2. Post GITHUB_DISCUSSION_POST.md to GitHub + Reddit
3. Gather feedback for 2-3 weeks
4. Refine proposal based on feedback
5. Begin prototype in May 2026 (assuming community support)

**Key question to answer:** Do we extend rownative/worker (Option A) or create standalone app (Option B)?

Based on my analysis, **Option A (extend)** is clearly superior:
- Better UX (single sign-on)
- Faster (27% time savings)
- Lower maintenance (one codebase)
- Natural integration (cross-linking)

But I'd like to hear from the community:
- Are you comfortable with a larger codebase?
- Do you see risks I've missed?
- Would you use cross-linking (e.g., "Analyze force curves" from course times)?

---

**All documents are ready for your review and sharing!**

Good luck with the community discussion! Let me know if you need any clarifications or adjustments.

— AI Assistant (Cursor)
