# Funding Model — Community-Supported, Always Free

**Date:** April 9, 2026  
**Principle:** No payment processors, no subscriptions, no refunds, no compliance burden on maintainers

---

## Core Philosophy

**The platform is 100% free for all users, forever.**

- No paid tiers
- No premium features
- No paywalls
- No ads
- No subscription management
- No payment processor integration
- No refund handling

**Why?** Community maintainers should focus on code, not payment compliance, accounting, refunds, and tax filings.

---

## Hosting Cost Reality

### Cloudflare Free Tier Limits

| Resource | Free Tier | Estimated Support |
|----------|-----------|-------------------|
| Workers requests | 100,000/day | ~500 users @ 200 req/day |
| D1 storage | 10 GB | ~5,000 analyses @ 2MB each |
| D1 reads | 5M/day | ~500 users @ 10k reads/day |
| KV operations | 100,000/day | More than sufficient |
| R2 storage | 10 GB | Future FIT caching |

### When Free Tier Is Exceeded

**Estimated costs if scaling beyond free tier:**

- 1,000 active users: **$0-5/month** (likely still free)
- 3,000 active users: **$10-20/month** (D1 storage + extra requests)
- 10,000 active users: **$50-80/month** (D1 + R2 + Workers paid tier)

**For context:** Rowsandall.com served ~3,000 users on a VPS costing ~$20/month.

---

## Donation Model

### Platform

Use **GitHub Sponsors** as primary donation platform:

**Benefits:**
- ✅ Zero fees (GitHub doesn't take a cut)
- ✅ Integrated with repository (users see "Sponsor" button)
- ✅ Monthly or one-time donations
- ✅ Public transparency (sponsor list visible)
- ✅ Tax handling (GitHub provides receipts)
- ✅ No maintainer burden (GitHub manages payments)

**Alternative platforms:**
- **Ko-fi:** "Buy me a coffee" one-time donations, 0% platform fee
- **Open Collective:** Transparent community fund, fiscal host handles taxes (10% fee)

### Suggested Donation Tiers

**GitHub Sponsors tiers (suggested):**

| Tier | Amount | Description |
|------|--------|-------------|
| Supporter | $2/month | Help cover server costs |
| Enthusiast | $5/month | Keep the platform running |
| Benefactor | $10/month | Support ongoing development |
| One-time | $5-50 | Buy us a coffee |

**No rewards, no exclusive features.** Donations are purely voluntary support for infrastructure.

---

## Transparency

### Monthly Cost Reports

Publish transparent monthly reports in GitHub Discussions:

**Example report format:**

```markdown
## Hosting Costs — March 2027

**Cloudflare Usage:**
- Workers requests: 2.1M (free tier: 3M/month)
- D1 storage: 8.2 GB (free tier: 10 GB)
- D1 reads: 42M (free tier: 150M/month)

**Cost this month:** $0.00 (within free tier ✅)

**Donations received:** $15 (3 sponsors @ $5/month)
**Cumulative fund:** $142

**Status:** ✅ Healthy — no cost pressure
```

### Annual Transparency Report

Once per year, publish:
- Total donations received
- Total hosting costs paid
- Fund balance
- Major expenses (if any: domains, SSL certs, etc.)
- Contributor recognition

**Example:**
```markdown
## 2027 Annual Report

**Donations:** $180 (12 months × $15/month average)
**Costs:** $0 (Cloudflare free tier sufficient)
**Fund balance:** $180

**Use of funds:**
- Reserved for future scaling or domain renewal
- No payments to maintainers (all volunteer)

**Thank you to our 12 sponsors!**
```

---

## Scaling Strategy

### If Free Tier Exceeded

**Step 1: Optimize**
- Implement database archiving (export old analyses to user's local storage)
- Cache aggressively (reduce D1 reads)
- Compress data (reduce storage)

**Step 2: Community Appeal**
- Post transparent cost report: "We need $20/month to cover hosting"
- Update README with donation links
- Email active users (via intervals.icu integration) with optional appeal

**Step 3: Accept Donations**
- Enable GitHub Sponsors
- Add Ko-fi "Buy me a coffee" link to footer
- No pressure, just visibility

**Step 4: If Donations Insufficient**
- Continue optimizing
- Consider more aggressive archiving (auto-delete analyses >2 years old)
- As last resort: Cloudflare paid tier ($5-20/month) covered by maintainer, continue accepting donations

---

## What We Will NOT Do

❌ **No payment processors** (Stripe, PayPal, Braintree)
- Requires business registration
- Tax compliance (VAT, sales tax)
- PCI compliance for card data
- Refund management
- Chargebacks
- Customer support burden

❌ **No subscriptions**
- Recurring billing complexity
- Cancellation flows
- Failed payment handling
- Pro-rating refunds
- Billing disputes

❌ **No "premium features"**
- Creates two-tier community
- Maintenance burden (feature flags, access control)
- Moral hazard (temptation to paywall useful features)

❌ **No ads**
- Degrades UX
- Privacy concerns (tracking)
- Low revenue for niche community
- Not aligned with open-source values

---

## Comparisons

### Rowsandall.com Model

- Paid subscriptions ($25/year for Pro)
- Payment processing via PayPal/Braintree
- Refund management
- Tax compliance
- Single maintainer burden

**Problem:** Solo developer had to handle billing support, refunds, tax filings on top of development.

### Our Model

- 100% free
- Optional donations (GitHub Sponsors)
- Zero payment processing
- Zero refunds
- Zero tax compliance for maintainers (GitHub/Ko-fi handles it)

**Benefit:** Maintainers focus on code, community handles infrastructure support voluntarily.

---

## Governance

### How Donation Funds Are Used

**Priority order:**
1. Cloudflare paid tier (if free tier exceeded)
2. Domain renewal ($12/year for rownative.icu)
3. Development tools (if needed: testing services, etc.)
4. ❌ **NOT for paying maintainers** (keeps things simple, avoids conflicts)

**Who decides?** Core maintainer team (3-5 people) with transparent voting in GitHub Discussions.

---

## FAQ

### Q: What if no one donates?

**A:** Platform stays free, we optimize to stay within free tier, or maintainer covers small overage ($5-20/month) as personal contribution.

### Q: What if we get too many donations?

**A:** Unlikely for a niche rowing app, but if it happens:
- Save for future scaling
- Donate excess to related open-source projects (intervals.icu, rowingdata)
- Use for development tools (hosting for staging environment, etc.)

### Q: Can I donate to specific features?

**A:** Donations are for infrastructure only, not feature bounties. Features are decided by community consensus, implemented by volunteers.

### Q: Do I get anything for donating?

**A:** Just our gratitude and your name on the sponsors list (if you opt in). No exclusive features, no special access.

### Q: Is this sustainable long-term?

**A:** Rowsandall served 3,000 users for 10 years on ~$20/month VPS. Cloudflare's free tier is more generous. With 1% of users donating $5/month, we'd cover costs easily. If not, costs are low enough for maintainers to cover as personal contribution.

---

## Implementation

### Phase 1: MVP (Pre-Funding)

- Launch on Cloudflare free tier
- No donation links yet
- Monitor usage for 3-6 months
- Establish baseline costs (likely $0)

### Phase 2: Add Donation Links (If Needed)

**Trigger:** If usage exceeds 75% of free tier, or approaching $5/month cost

**Actions:**
1. Set up GitHub Sponsors for the `rownative` organization
2. Add "Sponsor" button to repository
3. Add small footer link: "☕ Support hosting costs"
4. Publish first transparency report

### Phase 3: Community Fund (If Successful)

**Trigger:** If regular donations exceed $20/month

**Actions:**
1. Set up Open Collective for transparent fund management
2. Publish quarterly reports
3. Consider using funds for related projects (e.g., sponsor rowingdata development)

---

## Summary

**The platform will always be free.**

If you want to support hosting costs, donations are appreciated but never required. Maintainers commit to transparent reporting and efficient use of resources to minimize costs.

**We explicitly reject payment processors, subscriptions, and premium features** to keep maintenance burden low and community-first values strong.

**Expected reality:** Most users will never donate, and that's perfectly fine. The small minority who do will easily cover the $0-20/month infrastructure costs.

---

**Questions?** Open a GitHub Discussion.

**Want to donate?** Check the "Sponsor" button on the repository (available after launch).
