# ADR-004: Revenue Attribution Model

**Date:** 2025-11-18
**Status:** Accepted
**Deciders:** Product Lead, Engineering Team, Data Analyst
**Technical Story:** [FR-2 - Revenue Attribution, US-3.1 - Stripe Purchase Attribution]

## Context and Problem Statement

DataFa.st's core value proposition is **revenue attribution**: connecting website traffic to actual revenue. The attribution model must answer:
- "Which traffic source drove this purchase?"
- "What is the Revenue Per Visitor (RPV) by channel?"
- "Which marketing campaign has the best ROI?"

Multi-touch attribution is complex because users often:
- Visit from multiple sources before purchasing (e.g., Google → X → Direct)
- Have long decision cycles (30+ days for B2B SaaS)
- Use multiple devices (mobile discovery → desktop purchase)

We must choose an attribution model that balances **accuracy**, **simplicity**, and **fairness** to traffic sources.

## Decision Drivers

- **Accuracy:** Model should reflect reality of customer journeys
- **Simplicity:** Easy for users to understand (no black-box algorithms)
- **Fairness:** Credit valuable touchpoints (not just last click)
- **Performance:** Fast queries on 100M+ events
- **Attribution Window:** 30 days default (industry standard, configurable)
- **Multi-Touch:** Support for Phase 2 (linear, time-decay, U-shaped)
- **Privacy:** No cross-device tracking (respects GDPR)

## Considered Options

1. **Last-Click Attribution** (100% credit to final source)
2. **First-Click Attribution** (100% credit to initial source)
3. **Linear Attribution** (equal credit to all touchpoints)
4. **Time-Decay Attribution** (more credit to recent touchpoints)
5. **U-Shaped (Position-Based) Attribution** (40% first, 40% last, 20% middle)

## Decision Outcome

**Chosen option:** "Last-Click Attribution (MVP), Multi-Touch in Phase 2"

**Rationale:**
For MVP, **last-click** is the simplest and most understandable model. Users can quickly answer "What drove this sale?" without complex explanations. It aligns with industry standards (Google Analytics default) and is computationally cheap.

**Phase 2** will add multi-touch models (linear, time-decay) as premium features, giving power users more sophisticated attribution.

### Positive Consequences

- **Easy to understand:** "X drove 50% of revenue" is clear to founders
- **Fast queries:** Single JOIN between events and purchases
- **Industry standard:** Matches GA, Facebook Ads attribution
- **Simple debugging:** Easy to validate attribution accuracy
- **Low complexity:** Minimal code, fewer edge cases

### Negative Consequences

- **Undervalues discovery channels:** Organic search that introduces users gets no credit if they return via direct
- **Overvalues remarketing:** Retargeting ads get full credit even if user already knew about product
- **Ignores multi-touch journeys:** Doesn't reflect reality of complex decision-making
- **Gaming potential:** Users could manipulate last-click (e.g., self-referral links)

**Mitigation:** Educate users on limitations, add multi-touch in Phase 2.

## Pros and Cons of the Options

### Last-Click Attribution

**Pros:**
- Simplest to implement and explain
- Industry standard (GA, Facebook, most ad platforms)
- Fast queries (single session lookup)
- No complex data storage (just session → purchase link)
- Easy to validate (manual checking is straightforward)

**Cons:**
- Undervalues top-of-funnel (discovery) channels
- Ignores multi-touch customer journeys
- Biased toward retargeting and direct traffic
- Doesn't reflect reality for long sales cycles (B2B)

**Verdict:** Best for MVP, acceptable for 80% of users.

---

### First-Click Attribution

**Pros:**
- Credits discovery channels (organic, social)
- Useful for understanding top-of-funnel effectiveness
- Simple to implement (same as last-click, different logic)

**Cons:**
- Ignores nurturing and conversion efforts
- Overvalues awareness channels, undervalues conversion channels
- Less common (users expect last-click)

**Verdict:** Offer as alternative view in Phase 2.

---

### Linear Attribution

**Pros:**
- Fair: Every touchpoint gets equal credit
- Reflects multi-touch reality
- Encourages balanced marketing (awareness + conversion)

**Cons:**
- Complex to explain ("Why did organic get 25% credit for this sale?")
- Requires storing full journey (more data, slower queries)
- Harder to optimize (unclear which channel to invest in)
- Overvalues low-intent touchpoints (accidental visits)

**Verdict:** Good for advanced users, Phase 2 feature.

---

### Time-Decay Attribution

**Pros:**
- More credit to recent touchpoints (reflects recency bias)
- Balances discovery and conversion
- Configurable decay rate (e.g., 7-day half-life)

**Cons:**
- Most complex to implement (exponential decay formula)
- Hard to explain to non-technical users
- Arbitrary decay rate (no "correct" value)
- Requires full journey storage + custom queries

**Verdict:** Phase 3 (advanced analytics).

---

### U-Shaped (Position-Based)

**Pros:**
- Credits both discovery (40%) and conversion (40%)
- Middle touchpoints get 20% (nurturing credit)
- Balances first-click and last-click

**Cons:**
- Arbitrary percentages (why 40/40/20?)
- Complex to explain
- Requires journey storage
- Hard to validate accuracy

**Verdict:** Phase 2/3 (for sophisticated marketers).

---

## Technical Implementation

### MVP: Last-Click Attribution

**Schema (ClickHouse):**
```sql
-- Link purchase to most recent session before purchase
SELECT
    p.purchase_id,
    p.revenue,
    s.utm_source,
    s.utm_medium,
    s.utm_campaign,
    s.referrer
FROM purchases p
LEFT JOIN (
    SELECT
        visitor_id,
        utm_source,
        utm_medium,
        utm_campaign,
        referrer,
        timestamp,
        ROW_NUMBER() OVER (PARTITION BY visitor_id ORDER BY timestamp DESC) as rn
    FROM events
    WHERE event_type = 'pageview'
) s ON p.visitor_id = s.visitor_id
    AND s.timestamp <= p.purchase_timestamp
    AND s.timestamp >= p.purchase_timestamp - INTERVAL 30 DAY
    AND s.rn = 1;
```

**Logic:**
1. Stripe webhook receives purchase (customer_email, amount)
2. Match email → visitor_id (from events table)
3. Find most recent session within 30-day attribution window
4. Attribute purchase to that session's utm_source/referrer
5. If no session found → "Direct" (or "Unknown")

**Attribution Window:**
- Default: 30 days
- Configurable: 1, 7, 14, 30, 60, 90 days (in settings)
- Rationale: 30 days covers 90% of purchase decisions (research shows <30 days for B2C, <60 days for B2B)

**Edge Cases:**
- Multiple purchases from same visitor → Each attributed independently
- No session in window → Attributed to "Direct"
- Email mismatch → Fallback to IP + fingerprint (lower confidence)

---

### Phase 2: Multi-Touch Attribution

**Journey Storage:**
```sql
-- Store full journey per visitor
CREATE TABLE visitor_journeys (
    visitor_id UUID,
    touchpoints Array(Tuple(
        timestamp DateTime,
        utm_source String,
        utm_medium String,
        utm_campaign String,
        referrer String
    ))
) ENGINE = MergeTree()
ORDER BY visitor_id;
```

**Linear Attribution:**
```sql
-- Split revenue equally across touchpoints
SELECT
    unnest(touchpoints).utm_source as source,
    SUM(revenue / arrayLength(touchpoints)) as attributed_revenue
FROM purchases p
JOIN visitor_journeys j ON p.visitor_id = j.visitor_id
GROUP BY source;
```

**Time-Decay Attribution:**
```python
# Python function (API-side)
def time_decay_attribution(touchpoints, revenue, half_life_days=7):
    total_weight = 0
    weights = []
    now = touchpoints[-1].timestamp

    for touchpoint in touchpoints:
        days_ago = (now - touchpoint.timestamp).days
        weight = 0.5 ** (days_ago / half_life_days)
        weights.append(weight)
        total_weight += weight

    attributed = {}
    for touchpoint, weight in zip(touchpoints, weights):
        source = touchpoint.utm_source
        attributed[source] = attributed.get(source, 0) + (revenue * weight / total_weight)

    return attributed
```

---

## Attribution Report (Dashboard)

**Last-Click View (MVP):**
| Source | Visitors | Purchases | Revenue | RPV | Conv. Rate |
|--------|----------|-----------|---------|-----|------------|
| Google Organic | 5,000 | 50 | $2,500 | $0.50 | 1.0% |
| X (Twitter) | 1,000 | 20 | $1,000 | $1.00 | 2.0% |
| Direct | 2,000 | 30 | $1,500 | $0.75 | 1.5% |
| **Total** | **8,000** | **100** | **$5,000** | **$0.63** | **1.25%** |

**Multi-Touch View (Phase 2):**
- Dropdown: Select model (Last-Click, First-Click, Linear, Time-Decay)
- Compare button: Side-by-side comparison of models
- Export: Download attribution data (CSV) for custom analysis

---

## Validation & Testing

**Accuracy Testing:**
1. Manual test: Make purchase, verify correct source attributed
2. Cohort analysis: Compare self-reported source (survey) vs. attribution
3. A/B test: Run ads with unique UTMs, verify 100% attribution

**Edge Case Testing:**
- Purchase outside window → "Direct"
- Multiple sessions → Most recent wins
- Email change → Fallback to fingerprint

**Success Criteria:**
- 95%+ attribution accuracy (vs. manual validation)
- <1% "Unknown" attributions (max 5% acceptable)

---

## Future Enhancements (Phase 3)

- [ ] **Machine Learning Attribution:** Use ML to weight touchpoints (Google's data-driven attribution)
- [ ] **Cross-Device Tracking:** Link mobile → desktop (requires probabilistic matching, privacy concerns)
- [ ] **Offline Attribution:** Connect in-store purchases to online ads (via promo codes, phone numbers)
- [ ] **Incrementality Testing:** A/B test with holdout groups to measure true ad impact

---

## Links

- [Google Analytics Attribution Models](https://support.google.com/analytics/answer/1665189)
- [Facebook Attribution Window](https://www.facebook.com/business/help/458681590974355)
- [Multi-Touch Attribution Guide](https://www.bigcommerce.com/articles/ecommerce/multi-touch-attribution/)
- [ClickHouse Array Functions](https://clickhouse.com/docs/en/sql-reference/functions/array-functions)
- [PRD Section 3: FR-2 Revenue Attribution](../prd/DATAFAST-PRD.md#fr-2-revenue-attribution)

---

**Reviewed by:** [Pending]
**Approved by:** [Pending]
