# In-Stroke Analysis Platform — Proposal Documentation

> High-level architecture and requirements for replacing Rowsandall.com in-stroke analysis functionality

**Status:** Proposal — awaiting community feedback  
**Date:** April 9, 2026  
**Version:** 1.0

---

## Overview

This folder contains **proposal and planning documents only** for a community-maintained, open-source in-stroke analysis platform to replace Rowsandall.com's force curve and stroke analysis tools before its shutdown (end of 2026).

**Important:** These are architectural proposals and requirements. The actual implementation will happen in:
- **Backend API:** `rownative/worker` repository (Cloudflare Workers)
- **Frontend UI:** `rownative/courses` repository (GitHub Pages)

See [REPOSITORY_STRUCTURE.md](./REPOSITORY_STRUCTURE.md) for where code actually goes.

The platform will:
- Integrate seamlessly with intervals.icu for data access and authentication
- Parse FIT files per the [rowingdata FIT_EXPORT specification](https://github.com/sanderroosendaal/rowingdata/blob/develop/docs/FIT_EXPORT.md)
- Analyze force curves, seat curves, boat acceleration, and oarlock metrics
- Enable stroke-by-stroke filtering, comparison, and visualization
- Support multiple sensors (Empower Oarlock, Quiske, RP3, smartphone apps)
- Run on Cloudflare's free tier (zero hosting cost)

---

## Documentation Structure

### 📂 [REPOSITORY_STRUCTURE.md](./REPOSITORY_STRUCTURE.md)
**Start here if you're implementing!** Clarifies where the actual code goes (this is just proposal docs).

**Contents:**
- Where does backend code go? (`rownative/worker`)
- Where does frontend code go? (`rownative/courses`)
- What is this `instrokedata` folder? (proposal docs only)
- How do we keep API/UI in sync?
- Development workflow (feature branches, testing, deployment)

**Read this first** if you're ready to start coding.

---

### 📋 [EXECUTIVE_SUMMARY.md](./EXECUTIVE_SUMMARY.md)
**Start here if you're deciding!** Quick decision guide with key recommendations, timeline, and success criteria.

**Contents:**
- Key decision: Extend rownative/worker vs standalone app
- Architecture at a glance
- Core features (MVP)
- Comparison with Rowsandall
- Timeline and success metrics
- Open questions for community

**Read this first** if you want the high-level overview and key recommendations.

---

### 🏗️ [ARCHITECTURE.md](./ARCHITECTURE.md)
Comprehensive technical architecture proposal.

**Contents:**
1. Executive Summary
2. Architecture Overview (system context, technology stack)
3. Functional Components (data pipeline, core modules)
4. Database Schema (D1 tables)
5. API Specification (endpoints, request/response formats)
6. Frontend Components (page structure, UI flows, charting)
7. Deployment and Operations (Cloudflare infrastructure, CI/CD, monitoring)
8. Security and Privacy (auth, GDPR, rate limiting)
9. Migration from Rowsandall (export/import process)
10. Future Enhancements (roadmap)
11. Open Questions and Decisions

**Read this** for deep technical details and implementation guidance.

---

### 📝 [REQUIREMENTS.md](./REQUIREMENTS.md)
Detailed functional and non-functional requirements.

**Contents:**
1. Introduction (scope, audience, references)
2. Functional Requirements (FR-* series)
   - Authentication and authorization
   - Workout ingestion (FIT parsing)
   - Single workout analysis
   - Analysis library
   - Multi-analysis comparison
   - Sharing and data management
   - Migration from Rowsandall
   - Help and documentation
3. Non-Functional Requirements (NFR-* series)
   - Performance, scalability, reliability
   - Security, usability, maintainability
   - Interoperability
4. Data Requirements (schema, retention, privacy)
5. Interface Requirements (UI, API, external interfaces)
6. Constraints and Assumptions
7. Acceptance Criteria (MVP definition, test scenarios)
8. Traceability Matrix (map requirements to architecture)
9. Review and Approval process
10. Glossary

**Read this** for complete requirements suitable for implementation and testing.

---

### ⚖️ [INTEGRATION_COMPARISON.md](./INTEGRATION_COMPARISON.md)
Detailed comparison of two implementation approaches.

**Contents:**
1. Quick Comparison table
2. Option A: Extend rownative/worker (RECOMMENDED)
   - Architecture, pros/cons, costs
3. Option B: Standalone instroke app
   - Architecture, pros/cons, costs
4. Side-by-Side Feature Comparison
5. Architectural Patterns (modular design, route handlers, feature flags)
6. Cost-Benefit Analysis (time to MVP, ongoing maintenance, UX)
7. Risk Analysis (both options)
8. Community Feedback Questions
9. Recommendation and Implementation Roadmap

**Read this** if you're unsure about the integration strategy or want to compare alternatives.

---

### 🎨 [D3JS_MIGRATION_GUIDE.md](./D3JS_MIGRATION_GUIDE.md) **NEW**
Guide for reusing Rowsandall's D3.js charting code.

**Contents:**
1. Overview (why reuse vs build from scratch)
2. Architecture comparison (Go server vs client-side)
3. Migration steps (extract templates, remove Go syntax)
4. Code examples (force curve, in-stroke analysis, comparison)
5. Slider HTML and JavaScript patterns
6. PNG export functionality
7. Testing strategy
8. Timeline (10-15 days vs 15-20 days from scratch)

**Read this** for implementation guidance on adapting Rowsandall's proven D3.js charts for client-side rendering.

---

## Key Recommendation

### ✅ **Extend rownative/worker (Option A)**

**Rationale:**
- **Single sign-on:** Users already authenticated via intervals.icu OAuth
- **Shared infrastructure:** Reuse D1, KV, DNS, deployment pipeline
- **Faster time to market:** 27% faster (13 days vs 18 days)
- **Lower maintenance:** 30% less effort (one codebase, one deployment)
- **Natural integration:** Cross-link from course times to force curve analysis

**Implementation:**
- Add `/api/instroke/*` routes to existing `rownative/worker`
- Add `site/instroke/` subdirectory to existing `rownative/courses`
- Reuse D1 database (new tables via migrations)
- Modular design (separate `src/instroke-*.ts` modules)

See [INTEGRATION_COMPARISON.md](./INTEGRATION_COMPARISON.md) for full analysis.

---

## Technical Stack

**Unchanged from rownative/worker:**

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | HTML5 + Vanilla JS | Zero build step; fast; existing pattern |
| Charting | **D3.js v7** (from Rowsandall) | Interactive force curves; reuse proven templates |
| API Backend | Cloudflare Workers (TypeScript) | Serverless, globally distributed, free tier |
| Database | Cloudflare D1 (SQLite) | Relational data, SQL queries |
| Cache/State | Cloudflare KV | User preferences, shared links |
| File Storage | Cloudflare R2 (future) | FIT caching, large datasets |
| Hosting | Cloudflare Pages | Static site hosting |
| Auth | intervals.icu OAuth | No credential management |
| FIT Parsing | fit-file-parser (npm) | Server-side parsing |

**Cost:** $0/month (Cloudflare free tier sufficient for 1,000 users)

**Key Strategy:** Reuse D3.js charting code from Rowsandall's MIT-licensed [chart server](https://git.wereldraadsel.tech/sander/rowsandall-charts) — familiar UX, less development work, interactive features already working.

---

## Core Features (MVP)

### 1. Workout Ingestion
- Import OTW rowing activities from intervals.icu
- Parse FIT files (force curves, oarlock data, stroke metrics)
- Store in D1 for analysis

### 2. Single Workout Analysis
- Display force curve (mean + quartile shading)
- Filter strokes by SPM range, interval, time window
- Interactive chart (zoom, pan, hover tooltips)
- Save analysis with name

### 3. Analysis Library
- List all saved analyses
- Search, filter, manage (view, rename, delete)

### 4. Multi-Analysis Comparison
- Overlay 2-5 force curves on one chart
- Distinct colors, legend, toggle curves on/off

### 5. Sharing
- Generate shareable UUID link
- Public (show athlete name) or Private (anonymous)
- Expiration: 30/90 days, or never

### 6. Migration from Rowsandall
- Export tool in Rowsandall (ZIP with analyses + FIT files)
- Import endpoint in new platform

---

## Timeline

| Phase | Dates | Deliverables |
|-------|-------|--------------|
| **Community Review** | April 2026 | Feedback on architecture + requirements |
| **Prototype** | May 2026 | FIT parsing + force curve chart (proof of concept) |
| **Alpha** | June 2026 | Core ingestion + analysis working |
| **Beta** | July-Aug 2026 | Public beta; Rowsandall export tool ready |
| **Migration** | Sep-Nov 2026 | Parallel operation; user migration |
| **GA (v1.0)** | Dec 2026 | Full launch; Rowsandall shutdown |

---

## Sensor Compatibility

| Sensor | Force Curves | Boat Accel | Seat Curves | Oarlock Angles | Dual Oarlocks |
|--------|--------------|------------|-------------|----------------|---------------|
| **Empower Oarlock** | ✅ Yes | ❌ No | ❌ No | ✅ Yes | ✅ Yes |
| **Quiske** | ✅ Yes | ✅ Yes | ✅ Yes | ✅ Yes | 🟡 TBD |
| **RP3 Erg** | ✅ Yes | ❌ No | ✅ Yes | ❌ No | ❌ No |
| **Smartphone** | 🟡 Future | 🟡 Future | ❌ No | ❌ No | ❌ No |

All sensors supported if FIT file follows [rowingdata FIT_EXPORT spec](https://github.com/sanderroosendaal/rowingdata/blob/develop/docs/FIT_EXPORT.md).

---

## Success Metrics (3 months post-launch)

**Users:**
- 500 registered users
- 2,000 ingested workouts
- 5,000 saved analyses

**Performance:**
- <2s page load time (3G)
- <5s analysis creation
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
   - **Recommendation:** Force curves only for MVP.

2. **Default Privacy:**
   - Should analyses default to Public or Private?
   - **Recommendation:** Private (user opts in to sharing).

3. **Workout Retention:**
   - Unlimited storage, or auto-archive after 1 year inactivity?
   - **Recommendation:** Unlimited for MVP.

4. **Branding:**
   - "Instroke Analysis by rownative.icu" or separate name?
   - **Recommendation:** Use rownative.icu branding.

5. **Monetization:**
   - Accept donations? Premium features?
   - **Recommendation:** Free + optional "Buy me a coffee"; no premium features.

---

## How to Provide Feedback

### GitHub Discussions (Recommended)
- [rownative/courses Discussions](https://github.com/rownative/courses/discussions) (existing repo)
- Create a new discussion: "In-Stroke Analysis Platform — Feedback"
- Tag with `enhancement`, `architecture`, `requirements`

### Email
- Project lead: Sander Roosendaal ([@sanderroosendaal](https://github.com/sanderroosendaal))
- Subject: "In-Stroke Analysis — Feedback on [ARCHITECTURE/REQUIREMENTS]"

### Reddit
- Post to [r/Rowing](https://www.reddit.com/r/Rowing/)
- Title: "RFC: Open-source in-stroke analysis platform (Rowsandall replacement)"

### Specific Questions
If you have expertise in one of these areas, we'd especially value your input:

**For Rowers/Coaches:**
- Which curve types are most important for technique analysis?
- How do you currently use Rowsandall's in-stroke tools?
- What features would you add/remove?

**For Developers:**
- Is the modular design (Option A) sufficient to manage complexity?
- Are there security/privacy concerns we've missed?
- Suggestions for FIT parsing library (JavaScript)?

**For Sensor Manufacturers/Developers:**
- Is the FIT_EXPORT spec sufficient for your sensors?
- Would you contribute test FIT files for validation?

**For intervals.icu Users:**
- Does the OAuth flow match your expectations?
- Should we request additional scopes (e.g., write access for annotations)?

---

## Next Steps

### Immediate (April 2026)
1. ✅ **Complete proposal documentation** (ARCHITECTURE, REQUIREMENTS, COMPARISON)
2. 🔄 **Share with community** (GitHub, Reddit, forums)
3. 🔄 **Gather feedback** (prioritize features, validate assumptions)
4. 🔄 **Validate intervals.icu API** (test FIT download, rate limits)

### Short-term (May 2026)
5. ⏳ **Prototype FIT parsing** (build minimal ingestion + chart)
6. ⏳ **D1 schema design** (write migration scripts)
7. ⏳ **UI wireframes** (Figma or paper sketches)
8. ⏳ **Recruit contributors** (TypeScript dev, frontend dev, designer)

### Medium-term (June-Aug 2026)
9. ⏳ **Alpha development** (core API + UI working)
10. ⏳ **Beta testing** (invite Rowsandall users)
11. ⏳ **Rowsandall export tool** (implement ZIP export)
12. ⏳ **Migration testing** (test import flow)

---

## How to Contribute

**Code:**
- GitHub repo: [rownative/worker](https://github.com/rownative/worker) (API)
- GitHub repo: [rownative/courses](https://github.com/rownative/courses) (frontend)
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
- Participate in GitHub Discussions
- Share on Reddit r/Rowing, rowing forums

---

## Related Resources

### Existing Projects
- **rownative/courses:** [GitHub](https://github.com/rownative/courses) | [Website](https://rownative.icu)
- **rownative/worker:** [GitHub](https://github.com/rownative/worker)
- **Rowsandall.com:** [Website](https://rowsandall.com) | [Force Curves Guide](https://analytics.rowsandall.com/2024/06/20/how-to-work-with-force-curves-and-in-stroke-analysis/)

### Technical References
- **rowingdata FIT_EXPORT:** [Specification](https://github.com/sanderroosendaal/rowingdata/blob/develop/docs/FIT_EXPORT.md)
- **intervals.icu API:** [Documentation](https://intervals.icu/api/v1/docs/swagger-ui/index.html)
- **Garmin FIT SDK:** [Documentation](https://developer.garmin.com/fit/overview/)

### Community
- **intervals.icu Forum:** [OAuth Discussion](https://forum.intervals.icu/t/intervals-icu-oauth-support/2759)
- **Reddit r/Rowing:** [Community](https://www.reddit.com/r/Rowing/)
- **Rowsandall Forum:** [Analytics Blog](https://analytics.rowsandall.com)

---

## Project Contacts

**Project Lead:** Sander Roosendaal  
**GitHub:** [@sanderroosendaal](https://github.com/sanderroosendaal)

**Repository:** [rownative/courses](https://github.com/rownative/courses) (planned extension)  
**Discussion:** [GitHub Discussions](https://github.com/rownative/courses/discussions) (pending)

---

## License

This proposal documentation is licensed under **MIT License** (same as rownative/worker).

Future code contributions will follow the same license as the target repository (MIT).

Course data (if any) will follow **ODbL 1.0** license (same as rownative/courses).

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | April 9, 2026 | AI Assistant (Cursor) | Initial proposal (ARCHITECTURE, REQUIREMENTS, COMPARISON, SUMMARY) |

---

**Status:** 📋 Proposal — awaiting community review  
**Next Milestone:** Prototype (May 2026)

---

## Quick Links

- 📋 [Executive Summary](./EXECUTIVE_SUMMARY.md) — Start here!
- 🏗️ [Architecture Proposal](./ARCHITECTURE.md) — Deep technical details
- 📝 [Requirements Document](./REQUIREMENTS.md) — Functional + non-functional requirements
- ⚖️ [Integration Comparison](./INTEGRATION_COMPARISON.md) — Extend vs standalone analysis

---

**Feedback welcome!** Open a GitHub issue in the rownative repositories
