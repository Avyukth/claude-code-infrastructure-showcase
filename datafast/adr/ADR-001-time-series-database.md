# ADR-001: Time-Series Database Selection

**Date:** 2025-11-18
**Status:** Accepted
**Deciders:** Engineering Team, Product Lead
**Technical Story:** [FR-1, FR-2, FR-3 - Analytics Tracking and Revenue Attribution]

## Context and Problem Statement

DataFa.st needs to store and query millions of analytics events (pageviews, goals, purchases) with high write throughput and fast read performance for time-range queries. The database must:
- Handle 10K+ events/second during peak traffic
- Efficiently query time-series data (e.g., "revenue in last 30 days")
- Support aggregations by multiple dimensions (source, device, country)
- Scale to 100M+ events with 2-year retention
- Keep costs reasonable for a bootstrapped SaaS

Traditional relational databases (PostgreSQL) struggle with high-volume time-series writes and complex analytical queries.

## Decision Drivers

- **Write Performance:** Must ingest 10K events/second without bottlenecks
- **Query Performance:** Dashboard queries must return in <500ms (p95)
- **Cost Efficiency:** Optimize for storage and compute costs
- **Developer Experience:** Team is familiar with SQL
- **Scalability:** Horizontal scaling for growth
- **Data Retention:** Efficiently manage 2-year retention with partitioning
- **Real-Time:** Support near real-time queries for live visitor map

## Considered Options

1. **ClickHouse** (columnar, OLAP database)
2. **TimescaleDB** (PostgreSQL extension for time-series)
3. **PostgreSQL** (with manual partitioning)
4. **MongoDB** (document database with time-series collections)

## Decision Outcome

**Chosen option:** "ClickHouse"

**Rationale:**
ClickHouse is purpose-built for OLAP workloads with exceptional write/read performance for time-series data. It offers:
- **Columnar storage:** Compresses events data by 10-20x, reducing storage costs
- **MergeTree engine:** Optimized for time-series with automatic partitioning
- **SQL interface:** Familiar syntax for the team
- **Horizontal scaling:** Distributed mode for future growth
- **Materialized views:** Pre-aggregate common queries (e.g., daily revenue by source)

Benchmarks show ClickHouse outperforms TimescaleDB by 3-5x for analytical queries on large datasets[1].

### Positive Consequences

- **Ultra-fast queries:** Dashboard metrics load in <200ms even with 50M+ events
- **Cost-effective:** Compression reduces storage by 90% vs. PostgreSQL
- **Scalability:** Can scale to billions of events with cluster setup
- **Real-time aggregations:** Materialized views enable instant metric updates
- **Active community:** Extensive documentation and ClickHouse Cloud managed service available

### Negative Consequences

- **Learning curve:** Team must learn ClickHouse-specific SQL syntax (minor differences)
- **No strong consistency:** Eventual consistency for distributed setups (acceptable for analytics)
- **Limited transactions:** Not suitable for user/billing data (use PostgreSQL alongside)
- **Operational complexity:** Self-hosting requires Kubernetes or Docker Compose knowledge
- **Vendor lock-in:** If using ClickHouse Cloud, harder to migrate later

## Pros and Cons of the Options

### ClickHouse

**Pros:**
- 10-100x faster than PostgreSQL for analytical queries[1]
- Columnar compression: 10-20x smaller storage footprint
- Built-in time-series optimizations (TTL, partitioning)
- SQL interface (familiar to developers)
- Horizontal scaling with replication/sharding
- Managed cloud option (ClickHouse Cloud) reduces ops burden

**Cons:**
- Eventual consistency in distributed mode
- No ACID transactions (not an issue for append-only analytics)
- Steeper learning curve vs. PostgreSQL
- Smaller ecosystem compared to PostgreSQL
- Update/delete operations are expensive (use rarely)

---

### TimescaleDB

**Pros:**
- PostgreSQL extension: Familiar for PostgreSQL users
- ACID transactions and strong consistency
- Supports JOINs with relational data
- Easier migration from existing PostgreSQL setup
- Good documentation and community support

**Cons:**
- **3-5x slower** than ClickHouse for large datasets[1]
- Higher storage costs (less compression)
- Partitioning requires more manual tuning
- Vertical scaling limits (single-node bottleneck)
- Paid features for compression and distributed setups (Timescale Cloud)

**Verdict:** Good for simpler use cases, but doesn't scale as well for 100M+ events.

---

### PostgreSQL (with Manual Partitioning)

**Pros:**
- Team is highly familiar with PostgreSQL
- ACID compliance and strong consistency
- Mature ecosystem (tools, extensions, monitoring)
- Can use same DB for analytics + user data

**Cons:**
- **Poor write performance** at high volume (10K events/s requires heavy optimization)
- **Slow analytical queries** on large tables (even with indexes)
- Manual partitioning is error-prone and maintenance-heavy
- High storage costs (limited compression)
- Vertical scaling limits

**Verdict:** Not suitable for high-volume analytics workloads.

---

### MongoDB (Time-Series Collections)

**Pros:**
- Flexible schema (easy to add new event fields)
- Time-series collections (introduced in MongoDB 5.0)
- Horizontal scaling via sharding
- Good for mixed workloads (documents + time-series)

**Cons:**
- **Slower aggregations** than ClickHouse/TimescaleDB
- Higher memory footprint
- Team lacks MongoDB expertise
- Less efficient for pure analytical queries
- Expensive at scale (storage + compute costs)

**Verdict:** Better suited for operational data, not analytics.

---

## Implementation Plan

### Phase 1: Single-Node ClickHouse (Months 1-6)
- Deploy ClickHouse on Fly.io or Railway (single instance)
- Use **MergeTree** engine with partitioning by month
- Create table schema for events (see below)
- Set up materialized views for common aggregations
- Estimated cost: $50-100/month for 10M events/month

**Schema:**
```sql
CREATE TABLE events (
    event_id UUID,
    website_id UUID,
    visitor_id UUID,
    session_id UUID,
    event_type Enum('pageview', 'goal', 'purchase'),
    timestamp DateTime,
    page_url String,
    referrer String,
    utm_source LowCardinality(Nullable(String)),
    utm_medium LowCardinality(Nullable(String)),
    utm_campaign Nullable(String),
    device_type Enum('mobile', 'desktop', 'tablet'),
    browser LowCardinality(String),
    os LowCardinality(String),
    country FixedString(2),
    revenue Nullable(Decimal(10, 2)),
    goal_name LowCardinality(Nullable(String))
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (website_id, timestamp, event_id)
TTL timestamp + INTERVAL 2 YEAR;
```

**Materialized View (Daily Revenue by Source):**
```sql
CREATE MATERIALIZED VIEW daily_revenue_by_source
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (website_id, date, utm_source)
AS SELECT
    website_id,
    toDate(timestamp) AS date,
    utm_source,
    count() AS events,
    uniq(visitor_id) AS visitors,
    sum(revenue) AS total_revenue
FROM events
WHERE event_type = 'purchase'
GROUP BY website_id, date, utm_source;
```

### Phase 2: Distributed ClickHouse (Month 7+, if needed)
- Migrate to ClickHouse Cloud (managed service) OR
- Deploy ClickHouse cluster (3+ nodes) with replication
- Enable sharding for horizontal scaling
- Estimated cost: $200-500/month for 100M+ events/month

### Fallback Plan
If ClickHouse proves too complex operationally, migrate to **TimescaleDB** (easier ops, acceptable performance for <50M events).

---

## Monitoring and Validation

**Metrics to Track:**
- Write throughput: Events ingested/second
- Query latency: p50, p95, p99 for dashboard queries
- Storage growth: GB per 1M events
- Cost per 1M events

**Success Criteria (3 months post-launch):**
- ✅ 10K events/second write throughput sustained
- ✅ Dashboard queries <500ms (p95)
- ✅ Storage cost <$0.10 per 1M events
- ✅ Zero data loss or corruption incidents

---

## Links

- [ClickHouse Documentation](https://clickhouse.com/docs)
- [ClickHouse vs. TimescaleDB Benchmark](https://altinity.com/blog/clickhouse-vs-timescaledb)
- [MergeTree Engine Guide](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree)
- [ClickHouse Cloud](https://clickhouse.com/cloud)
- [PRD Section 6: Technical Specifications](../prd/DATAFAST-PRD.md#6-technical-specifications)

---

**Reviewed by:** [Pending]
**Approved by:** [Pending]

---

## References

[1] Altinity. (2023). ClickHouse vs. TimescaleDB: Benchmark Comparison. https://altinity.com/blog/clickhouse-vs-timescaledb
