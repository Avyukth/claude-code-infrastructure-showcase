# DataFa.st Phase 0 Validation Plan
## Pre-Development Validation & Technical Proof of Concept

**Version:** 1.0
**Date:** November 18, 2025
**Status:** Draft
**Duration:** 2 weeks
**Team:** 1-2 developers

---

## Executive Summary

Phase 0 validates DataFa.st's core technical assumptions and product-market fit before committing to full development. This 2-week sprint focuses on:
- **Technical feasibility:** Can we achieve <5min setup and <500ms queries?
- **User validation:** Will indie makers pay $99/mo for revenue analytics?
- **Architecture validation:** Does ClickHouse + Next.js meet performance targets?

**Success Criteria:** 80%+ of validation tests pass → Proceed to Phase 1 MVP development

---

## Table of Contents
1. [Objectives](#1-objectives)
2. [Technical Validation](#2-technical-validation)
3. [User Validation](#3-user-validation)
4. [Market Validation](#4-market-validation)
5. [Success Metrics](#5-success-metrics)
6. [Timeline](#6-timeline)
7. [Deliverables](#7-deliverables)
8. [Go/No-Go Decision](#8-gono-go-decision)

---

## 1. Objectives

### Primary Objectives

1. **Validate Technical Feasibility**
   - Prove ClickHouse can handle 10K events/second
   - Prove dashboard queries return in <500ms (p95)
   - Prove tracking script is <5KB compressed
   - Prove Stripe attribution accuracy >90%

2. **Validate User Demand**
   - Interview 20 target users (indie makers, bootstrappers)
   - Achieve 60%+ "would pay $99/mo" response rate
   - Identify top 3 must-have features

3. **Validate Market Positioning**
   - Confirm "revenue-first" resonates vs. "privacy-first" (Plausible)
   - Identify differentiation gaps vs. competitors

4. **Validate Architecture Decisions**
   - Confirm Next.js + ClickHouse + Vercel tech stack
   - Identify potential bottlenecks or risks

### Secondary Objectives

- Create minimal design mockups (Figma)
- Draft API documentation (OpenAPI spec)
- Estimate development timeline and costs
- Recruit 10 beta testers for Phase 1

---

## 2. Technical Validation

### Test 1: ClickHouse Performance Benchmark

**Hypothesis:** ClickHouse can ingest 10K events/second and query 100M events in <500ms.

**Method:**
1. Set up local ClickHouse instance (Docker)
2. Generate synthetic event data (10M events)
3. Benchmark:
   - **Write performance:** Bulk insert 10K events/second for 10 minutes
   - **Query performance:** Run typical dashboard queries (metrics by source)
   - **Storage efficiency:** Measure compression ratio (target: 10x)

**Tools:**
- ClickHouse Docker image
- k6 or Apache Bench (load testing)
- Custom script to generate events

**Synthetic Event Schema:**
```sql
CREATE TABLE events (
    event_id UUID,
    website_id UUID,
    visitor_id UUID,
    timestamp DateTime,
    utm_source LowCardinality(String),
    device_type Enum('mobile', 'desktop', 'tablet'),
    revenue Nullable(Decimal(10, 2))
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (website_id, timestamp);
```

**Test Queries:**
```sql
-- Query 1: Metrics by source (30 days)
SELECT
    utm_source,
    count() AS visitors,
    sum(revenue) AS total_revenue
FROM events
WHERE timestamp >= today() - INTERVAL 30 DAY
GROUP BY utm_source;

-- Query 2: Time-series (daily visitors, 30 days)
SELECT
    toDate(timestamp) AS date,
    count() AS visitors
FROM events
WHERE timestamp >= today() - INTERVAL 30 DAY
GROUP BY date
ORDER BY date;
```

**Success Criteria:**
- ✅ Ingest rate: >10K events/second sustained for 10 minutes
- ✅ Query latency: <500ms (p95) for 10M events
- ✅ Storage: <100 MB per 1M events (10x compression)

**Expected Results:**
- ClickHouse is known to handle 100K+ events/second, so 10K should be trivial
- Queries on 10M events should be <200ms with proper indexing

**Risks & Mitigations:**
- **Risk:** Queries >500ms on complex aggregations
- **Mitigation:** Use materialized views for pre-aggregation

---

### Test 2: Tracking Script Size & Performance

**Hypothesis:** Tracking script can be <5KB compressed without sacrificing features.

**Method:**
1. Build minimal tracking script (vanilla JS)
2. Features:
   - Auto-track pageviews
   - Capture UTM parameters
   - Generate visitor fingerprint (canvas + user agent)
   - Batch events (max 10 events, 5-second timeout)
   - Offline queue with retry
3. Build pipeline: Rollup + Terser + Brotli
4. Measure:
   - Compressed size (Brotli)
   - Parse + execute time (Chrome DevTools)
   - Network latency (send event to API)

**Build Command:**
```bash
rollup src/index.js -o dist/datafast.js
terser dist/datafast.js -o dist/datafast.min.js --compress --mangle
brotli -f -q 11 dist/datafast.min.js
```

**Performance Test:**
- Host script on CDN (Cloudflare)
- Create test HTML page with Lighthouse audit
- Measure impact on page load (First Contentful Paint, Total Blocking Time)

**Success Criteria:**
- ✅ Compressed size: <5KB (target: 3-4KB)
- ✅ Parse + execute: <20ms
- ✅ FCP impact: <50ms (Lighthouse)
- ✅ Network request: <100ms (to API)

**Expected Results:**
- Vanilla JS with aggressive minification should hit 3.5-4KB
- Modern browsers parse JS very fast (<10ms for 4KB)

**Risks & Mitigations:**
- **Risk:** Fingerprinting code adds too much size
- **Mitigation:** Use simpler fingerprinting (user agent + screen resolution only)

---

### Test 3: Stripe Revenue Attribution Accuracy

**Hypothesis:** >90% of Stripe purchases can be attributed to correct traffic source.

**Method:**
1. Create test Stripe account (test mode)
2. Simulate visitor journeys:
   - Visitor A: Google → Homepage → Purchase
   - Visitor B: X → Pricing → Purchase
   - Visitor C: Direct → Purchase
3. Send test pageview events to ingestion API
4. Trigger Stripe test webhooks (checkout.session.completed)
5. Query attribution: Verify correct source (google, x, direct)

**Test Cases:**
| Visitor | Journey | Expected Attribution | Actual | Pass? |
|---------|---------|----------------------|--------|-------|
| A | Google → Home → Purchase | Google | TBD | TBD |
| B | X → Pricing → Purchase | X | TBD | TBD |
| C | Direct → Purchase | Direct | TBD | TBD |
| D | Google → (30 days later) → Direct → Purchase | Google | TBD | TBD |
| E | Google → (31 days later) → Direct → Purchase | Direct | TBD | TBD |

**Success Criteria:**
- ✅ Accuracy: >90% (at least 9/10 correct attributions)
- ✅ Attribution window: Correctly applies 30-day default
- ✅ Fallback: "Direct" when no session found

**Expected Results:**
- Email matching is straightforward (100% accuracy if email matches)
- Edge cases (multiple sessions) handled by "most recent" logic

**Risks & Mitigations:**
- **Risk:** Email mismatch (user uses different email in Stripe)
- **Mitigation:** Fallback to IP + fingerprint (lower confidence, but better than nothing)

---

### Test 4: Dashboard Query Performance (Next.js + ClickHouse)

**Hypothesis:** Dashboard loads in <2 seconds with 1M events.

**Method:**
1. Build minimal Next.js dashboard
2. Components:
   - Metric cards (visitors, revenue, RPV)
   - Line chart (revenue over time)
   - Referrer table (top 10 sources)
3. Populate ClickHouse with 1M test events
4. Measure:
   - Time to First Byte (TTFB)
   - Time to Interactive (TTI)
   - Largest Contentful Paint (LCP)

**Performance Targets:**
- TTFB: <300ms
- LCP: <2s
- TTI: <3s

**Optimization Techniques:**
- React Server Components (fetch data on server)
- Streaming with Suspense (instant page load, data streams in)
- Redis caching (5-minute TTL)

**Success Criteria:**
- ✅ LCP: <2s (Lighthouse)
- ✅ ClickHouse queries: <500ms (p95)
- ✅ Redis cache hit rate: >80%

**Expected Results:**
- Next.js 14 App Router with RSC should easily hit <2s LCP
- ClickHouse queries on 1M events should be <200ms

**Risks & Mitigations:**
- **Risk:** Complex queries (multi-table JOINs) slow down
- **Mitigation:** Denormalize data, use materialized views

---

## 3. User Validation

### Interview Plan: 20 Target Users

**Target Segments:**
1. **Indie SaaS Founders** (10 interviews)
   - Revenue: <$10K MRR
   - Current tool: Google Analytics or none
   - Pain: Can't see revenue by traffic source

2. **E-commerce Owners** (5 interviews)
   - Platform: Shopify
   - Revenue: $20K-50K/month
   - Pain: Shopify analytics lacks attribution

3. **Growth Marketers** (5 interviews)
   - Company: 5-10 person teams
   - Budget: $500-2K/month on ads
   - Pain: GA4 attribution models don't match reality

**Recruitment:**
- X (Twitter): DM indie makers with >1K followers
- Reddit: Post in r/SaaS, r/Entrepreneur, r/indiebiz
- Indie Hackers: Forum post + DMs
- Referrals: Ask existing network

**Interview Script (15 minutes):**

1. **Discovery (5 min)**
   - What analytics tools do you currently use?
   - What do you like/dislike about them?
   - How do you track revenue by traffic source today?

2. **Problem Validation (5 min)**
   - On a scale of 1-10, how painful is this problem?
   - What workarounds do you currently use?
   - How much time do you spend on analytics weekly?

3. **Solution Validation (3 min)**
   - Show DataFa.st concept (mockups, landing page)
   - Would you use this? Why or why not?
   - What features are must-haves vs. nice-to-haves?

4. **Pricing Validation (2 min)**
   - Would you pay $99/month for this? $199/month?
   - What would make it worth $199 vs. $99?

**Interview Goals:**
- 60%+ say "I would pay $99/mo" → Product-market fit signal
- Identify top 3 must-have features for MVP
- Validate "revenue-first" positioning
- Recruit 10 beta testers (offer lifetime deal: $500 one-time)

**Success Criteria:**
- ✅ 12+ of 20 say "would pay $99/mo" (60%+ conversion)
- ✅ 3 must-have features identified (consensus across 70%+ interviews)
- ✅ 10 beta testers recruited

---

### Landing Page Test

**Hypothesis:** "Revenue-first analytics" messaging converts at 10%+ (visitors → email signups).

**Method:**
1. Build simple landing page (Next.js, Vercel)
2. Sections:
   - Hero: "Stop tracking vanity metrics. Track revenue."
   - Problem: GA is complex, doesn't show revenue by source
   - Solution: 4-minute setup, Stripe integration, revenue attribution
   - Pricing: $99/mo Starter, $199/mo Growth
   - CTA: "Join Beta (14-day trial)"
3. Run micro-ads campaign:
   - X Ads: $200 budget, target indie makers
   - Reddit Ads: $100 budget, r/SaaS, r/Entrepreneur
4. Track:
   - Landing page visits
   - Email signups (beta waitlist)
   - Conversion rate

**Success Criteria:**
- ✅ Conversion rate: >10% (visits → signups)
- ✅ 100+ beta waitlist signups
- ✅ Qualitative feedback: "This is exactly what I need!"

**Expected Results:**
- 10% conversion is strong for B2B SaaS (industry avg: 2-5%)
- If <10%, iterate on messaging

**Risks & Mitigations:**
- **Risk:** Low conversion (<5%)
- **Mitigation:** A/B test headlines, try different pain points

---

## 4. Market Validation

### Competitive Analysis

**Goal:** Confirm DataFa.st has defensible differentiation.

**Competitors to Analyze:**
1. **Plausible** (privacy-first, simple)
2. **Cometly** (marketing attribution)
3. **Google Analytics 4** (free, complex)
4. **Mixpanel** (product analytics)

**Comparison Matrix:**

| Feature | DataFa.st | Plausible | Cometly | GA4 |
|---------|-----------|-----------|---------|-----|
| **Revenue Attribution** | ✅ Core | ❌ No | ✅ Yes | ⚠️ Partial |
| **Setup Time** | 4-5 min | 5-10 min | 10+ min | 15+ min |
| **Stripe Integration** | ✅ Native | ❌ No | ⚠️ Via Zapier | ❌ Manual |
| **Real-Time Map** | ✅ Yes | ❌ No | ⚠️ Partial | ✅ Yes |
| **Pricing** | $99-199/mo | $9/10k views | Custom | Free |
| **Target Audience** | Indie makers | Privacy-focused | Agencies | Enterprise |

**Differentiation:**
- **vs. Plausible:** Revenue attribution (Plausible is privacy-only)
- **vs. Cometly:** Simpler, indie-focused (Cometly is enterprise)
- **vs. GA4:** Much simpler, revenue-first (GA4 is complex, vanity metrics)
- **vs. Mixpanel:** Revenue, not product events (Mixpanel is product analytics)

**Success Criteria:**
- ✅ Unique positioning identified (revenue-first for indie makers)
- ✅ Pricing validated as competitive ($99 vs. Plausible's $9)
- ✅ No direct competitor offering same feature set

---

## 5. Success Metrics

### Phase 0 Scorecard

| Validation Area | Metric | Target | Weight | Status |
|-----------------|--------|--------|--------|--------|
| **Technical Feasibility** | | | 40% | TBD |
| ClickHouse Performance | Ingest >10K events/sec | Pass | 10% | TBD |
| Query Performance | <500ms (p95) | Pass | 10% | TBD |
| Script Size | <5KB compressed | Pass | 10% | TBD |
| Attribution Accuracy | >90% correct | Pass | 10% | TBD |
| **User Validation** | | | 40% | TBD |
| User Interviews | 60%+ would pay $99/mo | Pass | 20% | TBD |
| Beta Signups | 100+ waitlist | Pass | 10% | TBD |
| Landing Page CVR | >10% conversion | Pass | 10% | TBD |
| **Market Validation** | | | 20% | TBD |
| Competitive Differentiation | Unique positioning | Pass | 10% | TBD |
| Pricing Validation | $99 acceptable | Pass | 10% | TBD |

**Overall Score:**
- **80%+ Pass → GREEN (Proceed to Phase 1)**
- **60-79% Pass → YELLOW (Iterate, re-test)**
- **<60% Pass → RED (Pivot or stop)**

---

## 6. Timeline

### Week 1: Technical Validation

| Day | Tasks | Owner | Deliverables |
|-----|-------|-------|--------------|
| Mon | Set up ClickHouse local instance | Dev 1 | Docker Compose file |
| Tue | Generate synthetic data (10M events) | Dev 1 | Event generator script |
| Wed | Run performance benchmarks | Dev 1 | Benchmark report |
| Thu | Build tracking script (v0.1) | Dev 2 | datafast.js (MVP) |
| Fri | Test script size & performance | Dev 2 | Lighthouse report |

### Week 2: User & Market Validation

| Day | Tasks | Owner | Deliverables |
|-----|-------|-------|--------------|
| Mon | Conduct 10 user interviews | Product | Interview notes |
| Tue | Build landing page | Dev 2 | Deployed site (Vercel) |
| Wed | Launch micro-ads campaign | Marketing | Ad campaigns live |
| Thu | Conduct 10 more interviews | Product | Interview notes |
| Fri | Compile validation report | Product | Phase 0 report |

**Total Duration:** 10 business days (2 weeks)

---

## 7. Deliverables

### Technical Deliverables

1. **ClickHouse Benchmark Report**
   - Write performance: Events/second, latency
   - Query performance: p50, p95, p99
   - Storage efficiency: Compression ratio

2. **Tracking Script (v0.1)**
   - Compressed size (Brotli)
   - Lighthouse score
   - Parse/execute time

3. **Stripe Attribution Test Results**
   - Attribution accuracy matrix
   - Edge case handling

4. **Dashboard Performance Report**
   - Next.js load times (TTFB, LCP, TTI)
   - ClickHouse query times

### User Validation Deliverables

1. **User Interview Summary**
   - 20 interview transcripts
   - Key insights & quotes
   - Must-have features (ranked)
   - Willingness to pay analysis

2. **Landing Page Analytics**
   - Visitors, signups, conversion rate
   - Heatmap, user recordings (Hotjar)
   - Feedback form responses

3. **Beta Waitlist**
   - 100+ email addresses
   - Segmented by persona (founder, marketer, etc.)

### Market Validation Deliverables

1. **Competitive Analysis**
   - Comparison matrix
   - SWOT analysis
   - Positioning statement

2. **Pricing Research**
   - Willingness to pay (WTP) analysis
   - Competitor pricing comparison
   - Recommended pricing strategy

### Final Deliverable: Phase 0 Validation Report

**Sections:**
1. Executive Summary (1 page)
2. Technical Validation Results (5 pages)
3. User Validation Results (5 pages)
4. Market Validation Results (3 pages)
5. Recommendation: Go / No-Go / Iterate (1 page)
6. Appendices: Raw data, interview transcripts, benchmarks

**Format:** PDF, shared with stakeholders

---

## 8. Go/No-Go Decision

### Decision Framework

**GREEN (Proceed to Phase 1):**
- Technical: All 4 tests pass
- User: 60%+ would pay, 100+ signups
- Market: Clear differentiation
- **Action:** Start Phase 1 MVP development (8-10 weeks)

**YELLOW (Iterate):**
- Technical: 3/4 tests pass (identify bottleneck)
- User: 40-59% would pay (refine positioning)
- Market: Some differentiation (sharpen messaging)
- **Action:** 2-week iteration sprint, re-test

**RED (Pivot or Stop):**
- Technical: <3/4 tests pass (fundamental issues)
- User: <40% would pay (weak demand)
- Market: No clear differentiation (crowded space)
- **Action:** Pivot to different problem or stop

### Decision Makers

- **Product Lead:** Final go/no-go authority
- **Engineering Lead:** Technical feasibility input
- **Advisor/Mentor:** Market validation input

### Decision Deadline

**End of Week 2 (November 32, 2025)**
- Present Phase 0 report to stakeholders
- Discuss results, address concerns
- Make final decision within 48 hours

---

## Budget

| Item | Cost | Notes |
|------|------|-------|
| **Infrastructure** | $50 | ClickHouse Cloud trial, Vercel hobby |
| **Ads (Landing Page Test)** | $300 | X Ads ($200), Reddit Ads ($100) |
| **Tools** | $100 | Hotjar (heatmaps), Loom (video interviews) |
| **User Incentives** | $500 | $50 Amazon gift card for 10 beta testers |
| **Total** | **$950** | ~$1,000 budget for 2-week validation |

---

## Risks & Contingencies

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Can't recruit 20 users | Medium | High | Lower target to 15, offer higher incentive ($100 gift card) |
| ClickHouse too slow | Low | Critical | Fallback to TimescaleDB (Phase 1 decision) |
| Landing page <10% CVR | Medium | Medium | A/B test messaging, try different channels (Product Hunt) |
| Low willingness to pay | Medium | High | Test lower price ($49/mo), add features to justify $99 |

---

## Success Indicators (Leading Metrics)

**By End of Week 1:**
- ClickHouse benchmarks complete
- Tracking script built (<5KB)
- 10 user interviews scheduled

**By End of Week 2:**
- All 20 interviews complete
- 100+ landing page signups
- Technical validation report done

**Final Outcome:**
- Phase 0 report presented to stakeholders
- Go/no-go decision made
- If GO: Phase 1 kickoff scheduled (Week 3)

---

**Document Status:** ✅ Ready for Execution
**Next Step:** Get stakeholder approval, begin Week 1 tasks
**Last Updated:** November 18, 2025
