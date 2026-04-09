# In-Stroke Analysis Platform — Executive Summary

> Quick decision guide for replacing Rowsandall.com in-stroke functionality

**Date:** April 9, 2026  
**Status:** Proposal for community review

---

## Purpose

Replace Rowsandall.com's force curve and in-stroke analysis tools before shutdown (end of 2026) with a community-maintained, open-source platform integrated with intervals.icu.

---

## Key Decision: Integration Strategy

### Option A: Extend rownative/worker ✅ **RECOMMENDED**

**Extend the existing rownative.icu courses/challenges platform**

**Pros:**
- Shared authentication (single sign-on via intervals.icu OAuth)
- Reuse infrastructure (D1, KV, DNS, deployment pipeline)
- Natural cross-linking ("Analyze force curves from this course time")
- Unified user experience (one account, one site)
- Lower maintenance burden (one codebase, one deployment)
- Faster time to market (OAuth already working)

**Cons:**
- Increased codebase complexity
- Potential performance impact on courses API (mitigated by separate routes)
- Tighter coupling (mitigated by modular design)

**Implementation:**
- Add `/api/instroke/*` routes to existing worker
- Add `site/instroke/` subdirectory for UI
- Reuse D1 database (new tables via migrations)
- Keep in-stroke logic in separate TypeScript modules

### Option B: Standalone instroke app

**Create separate GitHub repo and deployment**

**Pros:**
- Independent deployment and versioning
- Cleaner separation of concerns
- Easier for new contributors (smaller codebase)

**Cons:**
- Duplicate OAuth setup and session management
- Separate DNS/domain or subdomain
- Fragmented user experience (multiple logins)
- Higher maintenance burden (two workers, two deployments)

---

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│ intervals.icu (OAuth + FIT files + GPS data)                │
└───────────────────────┬─────────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────────┐
│ Cloudflare Worker (rownative.icu/api/*)                     │
│ - Existing: /courses, /challenges, /oauth                   │
│ - NEW: /instroke (ingestion, analysis, comparison, sharing) │
├──────────────────────────────────────────────────────────────┤
│ D1 Database (SQLite)                                         │
│ - Existing: courses, course_times, challenges                │
│ - NEW: instroke_workouts, instroke_curves, instroke_analyses│
├──────────────────────────────────────────────────────────────┤
│ KV (Cache + State)                                           │
│ - Existing: liked courses, OAuth state                       │
│ - NEW: shared analysis links                                 │
└──────────────────────────────────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────────┐
│ GitHub Pages (rownative.icu)                                 │
│ - Existing: site/*.html (map, challenges)                    │
│ - NEW: site/instroke/*.html (analyze, compare, library)      │
└──────────────────────────────────────────────────────────────┘
```

**Technology Stack:**
- Frontend: HTML5 + Vanilla JS + **D3.js v7** (reuse Rowsandall templates; no build step)
- Backend: TypeScript (Cloudflare Workers)
- Database: D1 (SQLite), KV (key-value store)
- Hosting: Cloudflare (free tier)
- Auth: intervals.icu OAuth (existing flow)

**Cost:** $0/month (Cloudflare free tier sufficient for 1,000 users)

**Charting Strategy:** Reuse proven D3.js templates from Rowsandall chart server (https://git.wereldraadsel.tech/sander/rowsandall-charts) — familiar UX for users, less development work, interactive features already implemented

---

## Core Features (MVP)

### 1. Workout Ingestion
- Import OTW rowing activities from intervals.icu (last 3 months)
- Download and parse FIT files per [rowingdata FIT_EXPORT spec](https://github.com/sanderroosendaal/rowingdata/blob/develop/docs/FIT_EXPORT.md)
- Extract in-stroke data:
  - Force curves (Empower Oarlock, RP3)
  - Boat acceleration curves (Quiske)
  - Seat position curves (RP3)
  - Oarlock scalars (catch/finish angles, slip, wash)
- Store in D1 for analysis

### 2. Single Workout Analysis
- Display force curve (mean + quartile shading) — **reuse Rowsandall D3.js template**
- Filter strokes by:
  - SPM range (interactive sliders)
  - Interval (dropdown)
  - Time window (start/end)
  - Work per stroke (drive energy)
- Interactive chart (crosshair tooltips, zoom, toggle layers)
- Statistical summary (avg force, peak force, drive time, catch/finish angles)
- Save analysis with user-provided name
- Export to PNG with watermark

### 3. Analysis Library
- List all saved analyses (table view)
- Search and filter by curve type, date
- Actions: View, Compare, Share, Rename, Delete

### 4. Multi-Analysis Comparison
- Overlay 2-5 force curves on one chart — **reuse Rowsandall comparison template**
- Distinct colors per analysis with legend
- Toggle individual curves on/off
- Normalize X-axis (0-1 drive phase) for comparability
- Smooth migration for Rowsandall users (familiar interface)

### 5. Sharing
- Generate shareable link (UUID-based)
- Privacy: Public (show athlete name) or Private (anonymous)
- Expiration: 30/90 days, or never
- Read-only view for recipients (no login required)

### 6. Migration from Rowsandall
- Export tool in Rowsandall (ZIP with analyses.json + FIT files)
- Import endpoint in new platform
- Progress tracking + error reporting

---

## Key Capabilities by Sensor Type

| Sensor | Force Curves | Boat Accel | Seat Curves | Oarlock Angles | Dual (Port/Starboard) |
|--------|--------------|------------|-------------|----------------|------------------------|
| **Empower Oarlock** | ✅ Yes | ❌ No | ❌ No | ✅ Yes | ✅ Yes (dual oarlocks) |
| **Quiske** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | 🟡 TBD |
| **RP3 Erg** | ✅ Yes | ❌ No | ✅ Yes | ❌ No | ❌ No |
| **Smartphone** | 🟡 Future | 🟡 Future | ❌ No | ❌ No | ❌ No |

All sensors supported if FIT file follows rowingdata spec.

---

## Comparison with Rowsandall

| Feature | Rowsandall | New Platform |
|---------|------------|--------------|
| Authentication | Django accounts | intervals.icu OAuth (no passwords) |
| Hosting | VPS (paid) | Cloudflare (free) |
| Database | PostgreSQL | D1 (SQLite) |
| Force curve analysis | ✅ Yes | ✅ Yes (MVP) |
| Boat acceleration | ✅ Yes | 🟡 Phase 2 |
| Comparison tool | ✅ Yes | ✅ Yes (MVP) |
| Trend analysis | ✅ Yes | 🟡 Phase 2 |
| Video overlay | ❌ No | 🟡 Phase 3+ |
| Maintenance | Single maintainer | Community (open source) |
| Cost | $100/month | $0/month (free tier) |

---

## Timeline

| Phase | Dates | Deliverables |
|-------|-------|--------------|
| **Community Review** | April 2026 | Feedback on architecture + requirements |
| **Prototype** | May 2026 | FIT parsing + force curve chart (proof of concept) |
| **Alpha** | June 2026 | Core ingestion + analysis working (internal testing) |
| **Beta** | July-Aug 2026 | Public beta; Rowsandall export tool ready |
| **Migration Period** | Sep-Nov 2026 | Parallel operation; user migration |
| **GA (v1.0)** | Dec 2026 | Full launch; Rowsandall shutdown |

---

## Success Criteria (3 months post-launch)

**Users:**
- 500 registered users (intervals.icu OAuth)
- 2,000 ingested workouts
- 5,000 saved analyses

**Performance:**
- <2s page load time (3G connection)
- <5s analysis creation time
- <100ms p50 API latency

**Reliability:**
- 99.5% uptime
- <5% FIT parsing error rate
- Zero data loss incidents

**Community:**
- 5+ active contributors
- 20+ GitHub stars
- Active discussion forum

---

## Open Questions for Community

1. **MVP Scope:**
   - Include boat acceleration curves (Quiske) in MVP, or defer to Phase 2?
   - Include trend analysis (historical comparison) in MVP, or Phase 2?
   - **Recommendation:** Force curves only for MVP; boat accel in Phase 2.

2. **Default Privacy:**
   - Should analyses default to Public (encourage sharing) or Private (respect privacy)?
   - **Recommendation:** Private (user opts in to sharing).

3. **Workout Retention:**
   - Unlimited storage, or auto-archive after 1 year inactivity?
   - **Recommendation:** Unlimited for MVP; add archive in Phase 2 if storage is an issue.

4. **Branding:**
   - "Instroke Analysis by rownative.icu" or separate name?
   - **Recommendation:** Use rownative.icu branding; route: `rownative.icu/instroke/`

5. **Monetization:**
   - Accept donations? Premium features?
   - **Recommendation:** Free + optional "Buy me a coffee" link; no premium features.

---

## Risks and Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|-----------|
| intervals.icu API changes | High | Low | Monitor forum; maintain API client with version checks |
| FIT spec divergence | Medium | Medium | Coordinate with rowingdata maintainers; test with all sensors |
| Cloudflare free tier exceeded | Medium | Low | Monitor usage; accept donations or upgrade to paid tier ($5/month) |
| Low community engagement | High | Medium | Clear contribution guidelines; welcoming culture; regular updates |
| Rowsandall export tool delayed | Medium | Low | Work closely with Rowsandall maintainers; provide export spec early |

---

## Next Steps

### Immediate (April 2026)
1. **Share proposal with community:** Post to Rowsandall forum, Reddit r/Rowing, GitHub discussions
2. **Gather feedback:** Architecture, requirements, MVP scope
3. **Validate intervals.icu API:** Test FIT download, OAuth flow, rate limits
4. **Recruit contributors:** TypeScript dev, frontend dev, designer

### Short-term (May 2026)
5. **Prototype FIT parsing:** Build minimal ingestion + force curve chart (proof of concept)
6. **D1 schema design:** Write migration scripts for `instroke_*` tables
7. **UI wireframes:** Sketch analysis page, comparison page (Figma or paper)
8. **Set up CI/CD:** Extend existing workflows for in-stroke code

### Medium-term (June-Aug 2026)
9. **Alpha development:** Core API + UI working; internal testing
10. **Beta testing:** Invite Rowsandall users to test and provide feedback
11. **Rowsandall export tool:** Implement ZIP export in Rowsandall
12. **Migration testing:** Test import flow with real Rowsandall data

---

## How to Contribute

**Code:**
- GitHub repo: `rownative/worker` (extend existing)
- Frontend: `rownative/courses` (add `site/instroke/`)
- Open issues for feature requests or bugs
- Submit PRs for review

**Testing:**
- Try beta version with your FIT files (Empower, Quiske, RP3)
- Report parsing errors or UI issues

**Documentation:**
- Improve help pages (curve interpretation, sensor setup)
- Write blog posts or tutorials

**Design:**
- UI/UX review (accessibility, mobile responsiveness)
- Create icons, branding assets

**Discussion:**
- GitHub Discussions for design decisions
- Reddit r/Rowing for community outreach

---

## Key Contacts

**Project Lead:** Sander Roosendaal ([@sanderroosendaal](https://github.com/sanderroosendaal))  
**GitHub:** rownative/worker, rownative/courses  
**Community:** TBD (GitHub Discussions, Discord, or Slack)

---

**Recommendation:** **Proceed with Option A (Extend rownative/worker)** and begin prototype development in May 2026. Full proposal in [ARCHITECTURE.md](./ARCHITECTURE.md) and [REQUIREMENTS.md](./REQUIREMENTS.md).
