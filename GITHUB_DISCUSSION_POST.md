# RFC: In-Stroke Analysis Platform for rownative.icu

**Type:** Proposal / Request for Comments  
**Status:** Draft — seeking community feedback  
**Date:** April 9, 2026  
**Related:** Rowsandall.com shutdown (end of 2026)

---

## TL;DR

**Context:** Rowsandall.com (2016-2026) is shutting down. After 10 years serving the rowing community, the creator is migrating users to intervals.icu and open-sourcing the D3.js charting code.

**Proposal:** Build a community-maintained, open-source in-stroke analysis platform as a companion to intervals.icu, leveraging lessons learned from Rowsandall's architecture.

**Approach:** Extend existing `rownative/worker` and `rownative/courses` repositories.

**Benefits:**
- ✅ Single sign-on (reuse intervals.icu OAuth)
- ✅ Zero hosting cost (Cloudflare free tier)
- ✅ Open source, community-maintained
- ✅ Natural integration with courses/challenges
- ✅ Faster time to market (27% vs standalone)

**Key Decision:** Extend existing repos (Option A) vs separate standalone app (Option B)?  
**Recommendation:** **Option A (extend)** — see comparison below.

---

## Problem Statement

**Rowsandall.com (2016-2026)** pioneered in-stroke analysis for rowing, serving thousands of users with force curve visualization, training plans, and performance tracking. Built as a Django monolith with a Go chart microservice, it proved the value of these tools but became complex to maintain solo.

**The shutdown:** The creator is migrating users to intervals.icu for general training logs (better multi-sport platform, team of developers) and encouraging the community to build open-source rowing-specific tools that won't be developed by intervals.icu for the small rowing market.

**What we're losing:**
- Force curve analysis (Empower Oarlock, Quiske, RP3 sensors)
- Stroke-by-stroke filtering and comparison
- In-stroke metrics (catch angle, finish angle, slip, wash)
- Training plan integration
- 10 years of proven UX and edge case handling

**What we need:** Community-maintained replacement that learns from Rowsandall's lessons (simpler architecture, reuse proven D3.js code) and integrates with intervals.icu.

---

## Proposed Solution

### Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│ intervals.icu (OAuth + FIT files + GPS data)             │
└───────────────────┬──────────────────────────────────────┘
                    │
┌───────────────────▼──────────────────────────────────────┐
│ Cloudflare Worker (rownative.icu/api/*)                  │
│ - Existing: /courses, /challenges, /oauth                │
│ - NEW: /instroke (ingestion, analysis, comparison)       │
├───────────────────────────────────────────────────────────┤
│ D1 Database (SQLite) + KV (Cache)                        │
│ - NEW: instroke_workouts, instroke_curves, analyses      │
└───────────────────┬──────────────────────────────────────┘
                    │
┌───────────────────▼──────────────────────────────────────┐
│ GitHub Pages (rownative.icu)                             │
│ - NEW: site/instroke/*.html (analyze, compare, library)  │
└──────────────────────────────────────────────────────────┘
```

**Technology Stack:**
- Backend: TypeScript (Cloudflare Workers) — reuse existing
- Frontend: HTML + Vanilla JS + **D3.js v7** (reuse Rowsandall templates) — no build step
- Database: D1 (SQLite) — add 4 new tables
- Auth: intervals.icu OAuth — reuse existing flow
- Cost: **$0/month** (Cloudflare free tier)

**Charting Strategy:** Reuse Rowsandall's proven D3.js templates (MIT licensed, being open-sourced) — familiar UX, 50% faster dev, advanced features already working. See [D3JS_MIGRATION_GUIDE.md](../D3JS_MIGRATION_GUIDE.md) for details.

**Funding:** 100% free platform, no subscriptions. Optional donations (GitHub Sponsors, Ko-fi) if hosting costs exceed Cloudflare free tier (~$0-20/month). See [FUNDING_MODEL.md](../FUNDING_MODEL.md) for details.

### Core Features (MVP)

1. **Workout Ingestion**
   - Import OTW rowing activities from intervals.icu
   - Parse FIT files per [rowingdata FIT_EXPORT spec](https://github.com/sanderroosendaal/rowingdata/blob/develop/docs/FIT_EXPORT.md)
   - Extract force curves, oarlock data, stroke metrics

2. **Single Workout Analysis**
   - Display force curve (mean + quartile shading) — **reuse Rowsandall D3.js UX**
   - Filter strokes by SPM range, interval, time window, work per stroke
   - Interactive D3.js chart (crosshair tooltips, toggle layers, zoom)
   - Statistical summary (avg force, peak force, catch/finish angles)
   - Save analysis with user-provided name
   - Export to PNG with watermark

3. **Multi-Analysis Comparison**
   - Overlay 2-5 force curves on one chart — **reuse Rowsandall comparison template**
   - Color-coded legend, toggle curves on/off
   - Normalize X-axis (0-1 drive phase)
   - Smooth migration for Rowsandall users

4. **Sharing**
   - Generate shareable UUID link (public/private)
   - Expiration: 30/90 days, or never

5. **Migration from Rowsandall**
   - Export tool in Rowsandall (ZIP with analyses + FIT files)
   - Import endpoint in new platform

---

## Key Decision: Integration Strategy

### Option A: Extend rownative/worker ✅ **RECOMMENDED**

**Add `/api/instroke/*` routes to existing worker; add `site/instroke/` to static site.**

**Pros:**
- ✅ Single sign-on (user already authenticated)
- ✅ Shared infrastructure (D1, KV, DNS, CI/CD)
- ✅ Natural cross-linking ("Analyze force curves from this course time")
- ✅ Lower maintenance (one codebase, one deployment)
- ✅ Faster time to market (13 days vs 18 days)

**Cons:**
- ❌ Larger codebase (mitigated by modular design)
- ❌ Potential performance impact (mitigated by separate route handlers)
- ❌ Deploy risk (bug could affect courses API; mitigated by testing + feature flags)

### Option B: Standalone instroke app

**Create separate GitHub repo and Cloudflare Worker.**

**Pros:**
- ✅ Cleaner separation (smaller, focused codebase)
- ✅ Independent deployment (no risk to courses API)

**Cons:**
- ❌ **Duplicate OAuth setup** (2-3 days of work)
- ❌ **Fragmented user experience** (separate login, no cross-linking)
- ❌ Higher maintenance burden (two codebases, two deployments)
- ❌ Longer time to market (18 days vs 13 days)

### Side-by-Side Comparison

| Aspect | **Extend (A)** | **Standalone (B)** |
|--------|----------------|---------------------|
| OAuth setup | 0 days (reuse) | 3 days (duplicate) |
| User sign-in | Already signed in | Sign in again |
| Cross-linking | "Analyze force curves" button | No linking (separate domains) |
| Maintenance | 1 codebase | 2 codebases |
| Time to MVP | 13 days | 18 days |

**Recommendation:** **Option A (extend)** — better UX, faster, lower maintenance.

**Full comparison:** See [INTEGRATION_COMPARISON.md](./INTEGRATION_COMPARISON.md)

---

## Sensor Compatibility

| Sensor | Force Curves | Boat Accel | Seat Curves | Oarlock Angles |
|--------|--------------|------------|-------------|----------------|
| **Empower Oarlock** | ✅ Yes | ❌ No | ❌ No | ✅ Yes |
| **Quiske** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes |
| **RP3 Erg** | ✅ Yes | ❌ No | ✅ Yes | ❌ No |
| **Smartphone** | 🟡 Future | 🟡 Future | ❌ No | ❌ No |

All sensors supported if FIT file follows rowingdata spec.

**MVP Focus:** Force curves (Empower, RP3) — most common use case.  
**Phase 2:** Boat acceleration (Quiske), seat curves.

---

## Timeline

| Phase | Dates | Deliverables |
|-------|-------|--------------|
| **Community Review** | April 2026 | Feedback on architecture + requirements |
| **Prototype** | May 2026 | FIT parsing + force curve chart |
| **Alpha** | June 2026 | Core ingestion + analysis working |
| **Beta** | July-Aug 2026 | Public beta; Rowsandall export tool |
| **Migration** | Sep-Nov 2026 | Parallel operation; user migration |
| **GA (v1.0)** | Dec 2026 | Full launch; Rowsandall shutdown |

---

## Documentation

Full proposal (200+ pages) available in this repo:

- 📋 [EXECUTIVE_SUMMARY.md](./EXECUTIVE_SUMMARY.md) — Quick overview (start here!)
- 🏗️ [ARCHITECTURE.md](./ARCHITECTURE.md) — Deep technical details
- 📝 [REQUIREMENTS.md](./REQUIREMENTS.md) — Functional + non-functional requirements
- ⚖️ [INTEGRATION_COMPARISON.md](./INTEGRATION_COMPARISON.md) — Extend vs standalone analysis
- 🔧 [TECHNICAL_REFERENCE.md](./TECHNICAL_REFERENCE.md) — Developer quick reference

---

## Questions for Community

### 1. Integration Strategy

**Do you prefer Option A (extend rownative) or Option B (standalone)?**

- 👍 Option A (extend) — better UX, natural integration
- 👎 Option B (standalone) — cleaner separation, but more work

**If Option A, are you concerned about codebase complexity?**

- Mitigation: Modular design (`src/instroke/` folder), clear documentation
- Example: How do courses + challenges currently coexist in one repo?

### 3. MVP Scope

**Should MVP include boat acceleration curves (Quiske), or defer to Phase 2?**

- 👍 Include in MVP (more sensors supported)
- 👎 Defer to Phase 2 (focus on force curves first — most common)

**Recommendation:** Force curves only for MVP (faster launch, 80% use case).

### 4. Privacy

**Should analyses default to Public or Private?**

- 👍 Public (encourage community learning, like Strava)
- 👎 Private (respect privacy, user opts in to sharing)

**Recommendation:** Private (respect privacy; Rowsandall was private by default).

### 5. Retention

**Should workouts be stored indefinitely, or auto-archive after 1 year inactivity?**

- 👍 Indefinite (long-term trend analysis)
- 👎 Auto-archive (keep storage under free tier limits)

**Recommendation:** Indefinite for MVP; add archive in Phase 2 if storage is an issue.

### 6. Cross-Linking

**Would you use "Analyze force curves" button from course times page?**

- Example: You row a 2k race on a measured course → save course time → click "Analyze Force Curves" → opens in-stroke analysis for same workout
- This feature only works with Option A (extend).

### 7. Sensor Expertise

**If you use Empower, Quiske, RP3, or other sensors:**
- Are you willing to share sample FIT files for testing?
- Any FIT parsing issues we should know about?

---

## How to Provide Feedback

**Preferred:** Comment on this GitHub Discussion.

**Specific questions:**
- Integration strategy (Option A vs B)?
- MVP scope (force curves only, or boat accel too)?
- Privacy defaults (public vs private)?
- Use case: Do you analyze force curves? How often?

**General feedback:**
- Architecture concerns?
- Missing features?
- Documentation clarity?

**GitHub:** Comment on this discussion or open an issue in the rownative repositories

---

## Next Steps

### Immediate (April 2026)
1. ✅ Share proposal with community (this post!)
2. 🔄 Gather feedback (integration strategy, MVP scope)
3. ⏳ Validate intervals.icu API (test FIT download, rate limits)

### Short-term (May 2026)
4. ⏳ Prototype FIT parsing (build minimal ingestion + chart)
5. ⏳ D1 schema design (write migration scripts)
6. ⏳ Recruit contributors (TypeScript dev, frontend dev, designer)

### Medium-term (June-Aug 2026)
7. ⏳ Alpha development (core API + UI working)
8. ⏳ Beta testing (invite Rowsandall users)
9. ⏳ Rowsandall export tool (implement ZIP export)

---

## Contributing

**We're looking for:**
- **TypeScript developer:** Worker API + FIT parsing
- **Frontend developer:** D3.js template adaptation + UI polish
- **Designer:** UI/UX review, branding, help docs
- **Tester:** Device testing (iOS/Android), sensor compatibility
- **Rower:** User feedback, feature priorities

**How to help:**
- Comment on this discussion with your thoughts
- Share sample FIT files (Empower, Quiske, RP3) for testing
- Volunteer to test alpha/beta versions
- Spread the word (intervals.icu forum, rowing communities)

---

## Related Links

**Existing Projects:**
- [rownative/courses](https://github.com/rownative/courses) — GPS course library
- [rownative/worker](https://github.com/rownative/worker) — Cloudflare Worker API
- [Rowsandall.com](https://rowsandall.com) — Current platform (shutting down)
- [Force Curves Guide](https://analytics.rowsandall.com/2024/06/20/how-to-work-with-force-curves-and-in-stroke-analysis/)

**Technical References:**
- [rowingdata FIT_EXPORT](https://github.com/sanderroosendaal/rowingdata/blob/develop/docs/FIT_EXPORT.md) — FIT parsing spec
- [intervals.icu API](https://intervals.icu/api/v1/docs/swagger-ui/index.html) — OAuth + activity API
- [intervals.icu Forum](https://forum.intervals.icu/t/intervals-icu-oauth-support/2759) — OAuth discussion

---

## Success Metrics (3 months post-launch)

**Users:**
- 500 registered users (intervals.icu OAuth)
- 2,000 ingested workouts
- 5,000 saved analyses

**Performance:**
- <2s page load time (3G)
- <5s analysis creation
- <100ms p50 API latency

**Community:**
- 5+ active contributors
- 20+ GitHub stars
- Active discussion forum

---

## Conclusion

**Recommendation:** Proceed with **Option A (extend rownative/worker)** for better UX, faster launch, and lower maintenance.

**Why now:** Rowsandall shuts down end of 2026; need MVP by June to allow migration period.

**Why open source:** Community maintenance ensures longevity; no single point of failure.

**Why intervals.icu:** De facto platform for rowing training data; seamless authentication.

**Your feedback is critical!** Please comment with your thoughts on integration strategy, MVP scope, and priorities.

---

**Prepared by:** Sander Roosendaal (with AI assistance)  
**GitHub:** [@sanderroosendaal](https://github.com/sanderroosendaal)  
**Full Proposal:** [instrokedata repo](.) (see README.md for all documents)
