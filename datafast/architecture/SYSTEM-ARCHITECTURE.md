# DataFa.st System Architecture
## Technical Architecture & System Design

**Version:** 1.0
**Date:** November 18, 2025
**Status:** Draft

---

## Table of Contents
1. [System Overview](#1-system-overview)
2. [C4 Architecture Diagrams](#2-c4-architecture-diagrams)
3. [Component Details](#3-component-details)
4. [Data Flow](#4-data-flow)
5. [Infrastructure](#5-infrastructure)
6. [Security Architecture](#6-security-architecture)
7. [Scalability Strategy](#7-scalability-strategy)

---

## 1. System Overview

### High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        A[User's Website<br/>tracking.js]
        B[Dashboard SPA<br/>Next.js]
        C[Mobile Browser]
    end

    subgraph "Edge Layer - CDN"
        D[Cloudflare CDN<br/>Static Assets]
        E[Vercel Edge<br/>Next.js SSR]
    end

    subgraph "Application Layer"
        F[Tracking API<br/>Event Ingestion]
        G[Dashboard API<br/>Query Service]
        H[Auth Service<br/>NextAuth.js]
        I[Billing Service<br/>Stripe Webhooks]
    end

    subgraph "Integration Layer"
        J[Stripe API]
        K[Lemon Squeezy API]
        L[Shopify API]
        M[Email Service<br/>Resend]
    end

    subgraph "Data Layer"
        N[ClickHouse<br/>Analytics Events]
        O[PostgreSQL<br/>Users & Config]
        P[Redis<br/>Cache & Sessions]
    end

    subgraph "Observability"
        Q[Sentry<br/>Error Tracking]
        R[Axiom<br/>Logs]
        S[Prometheus<br/>Metrics]
    end

    A -->|POST /track| F
    B -->|GraphQL/REST| G
    C -->|HTTPS| E

    D -->|Serve Script| A
    E -->|SSR| B

    F -->|Write Events| N
    G -->|Query Analytics| N
    G -->|Query Users| O
    H -->|Sessions| P
    H -->|User Data| O

    F -->|Errors| Q
    G -->|Errors| Q
    F -->|Logs| R
    G -->|Logs| R

    I -->|Webhooks| J
    I -->|Webhooks| K
    I -->|Webhooks| L
    H -->|Send Email| M

    G -->|Cache| P
```

### System Boundaries

**Internal Systems (DataFa.st owned):**
- Tracking script (JavaScript)
- Dashboard frontend (Next.js)
- Tracking API (event ingestion)
- Dashboard API (queries)
- Authentication service
- Billing service

**External Systems (Third-party):**
- Stripe, Lemon Squeezy, Polar, Shopify (payment processors)
- Cloudflare/BunnyCDN (script delivery)
- Vercel (frontend hosting)
- Fly.io/Railway (backend hosting)
- Sentry, Axiom, Prometheus (observability)

---

## 2. C4 Architecture Diagrams

### Level 1: System Context

```mermaid
C4Context
    title System Context - DataFa.st Analytics Platform

    Person(founder, "Indie Founder", "Website owner tracking revenue")
    Person(visitor, "Website Visitor", "End user being tracked")

    System(datafast, "DataFa.st Platform", "Revenue-first analytics")

    System_Ext(stripe, "Stripe", "Payment processing")
    System_Ext(shopify, "Shopify", "E-commerce platform")
    System_Ext(email, "Resend", "Email delivery")
    System_Ext(cdn, "Cloudflare CDN", "Script delivery")

    Rel(founder, datafast, "Views analytics, configures tracking")
    Rel(visitor, datafast, "Tracked via JavaScript")
    Rel(datafast, stripe, "Fetches purchase data, manages billing")
    Rel(datafast, shopify, "Fetches orders")
    Rel(datafast, email, "Sends notifications")
    Rel(cdn, visitor, "Delivers tracking script")
```

### Level 2: Container Diagram

```mermaid
C4Container
    title Container Diagram - DataFa.st Platform

    Person(user, "User", "Website owner")

    Container_Boundary(datafast, "DataFa.st Platform") {
        Container(web, "Dashboard", "Next.js", "Analytics dashboard, settings")
        Container(script, "Tracking Script", "JavaScript", "Collects events from websites")
        Container(ingest_api, "Ingestion API", "Node.js/Express", "Receives and processes events")
        Container(query_api, "Query API", "Node.js/Express", "Serves analytics data")
        Container(auth, "Auth Service", "NextAuth.js", "Authentication & sessions")
        Container(billing, "Billing Service", "Node.js", "Handles subscriptions")

        ContainerDb(clickhouse, "Analytics DB", "ClickHouse", "Stores events")
        ContainerDb(postgres, "Config DB", "PostgreSQL", "Users, websites, settings")
        ContainerDb(redis, "Cache", "Redis", "Sessions, rate limits")
    }

    System_Ext(stripe, "Stripe", "Payments")
    System_Ext(cdn, "CDN", "Script delivery")

    Rel(user, web, "Uses", "HTTPS")
    Rel(web, query_api, "Fetches data", "REST/GraphQL")
    Rel(web, auth, "Authenticates", "HTTP")
    Rel(script, ingest_api, "Sends events", "HTTPS POST")

    Rel(ingest_api, clickhouse, "Writes events", "Native protocol")
    Rel(query_api, clickhouse, "Queries data", "SQL")
    Rel(query_api, postgres, "Reads config", "SQL")
    Rel(auth, postgres, "Reads/writes users", "SQL")
    Rel(auth, redis, "Stores sessions", "TCP")
    Rel(billing, stripe, "Webhooks", "HTTPS")
    Rel(billing, postgres, "Updates subscriptions", "SQL")

    Rel(cdn, script, "Delivers", "HTTPS")
```

### Level 3: Component Diagram (Tracking API)

```mermaid
C4Component
    title Component Diagram - Tracking API (Event Ingestion)

    Container_Boundary(tracking_api, "Tracking API") {
        Component(router, "HTTP Router", "Express", "Routes requests")
        Component(validator, "Event Validator", "Zod", "Validates event payloads")
        Component(enricher, "Event Enricher", "Node.js", "Adds IP, user agent parsing")
        Component(batcher, "Event Batcher", "BullMQ", "Batches events for bulk insert")
        Component(writer, "DB Writer", "ClickHouse Client", "Writes to database")
        Component(deduper, "Deduplicator", "Redis", "Prevents duplicate events")
    }

    ContainerDb(clickhouse, "ClickHouse", "Analytics DB")
    ContainerDb(redis, "Redis", "Cache")

    Rel(router, validator, "Validates")
    Rel(validator, enricher, "Enriches")
    Rel(enricher, deduper, "Checks duplicates")
    Rel(deduper, batcher, "Queues")
    Rel(batcher, writer, "Batch writes")
    Rel(writer, clickhouse, "Bulk INSERT")
    Rel(deduper, redis, "Checks")
```

---

## 3. Component Details

### 3.1 Tracking Script (tracking.js)

**Purpose:** Lightweight JavaScript library embedded on customer websites to collect analytics events.

**Responsibilities:**
- Auto-track pageviews (including SPA route changes)
- Capture UTM parameters and referrer
- Generate visitor fingerprint (canvas, fonts, user agent)
- Send events to ingestion API (batched)
- Queue events offline, retry on failure
- Expose public API: `window.datafast.goal(name, value)`

**Technology:**
- Vanilla JavaScript (ES6+)
- Build: Rollup + Terser + Brotli
- Size: <5KB compressed

**Key Files:**
```
tracking-script/
├── src/
│   ├── index.js              # Entry point
│   ├── tracker.js            # Core tracking logic
│   ├── fingerprint.js        # Visitor ID generation
│   ├── queue.js              # Offline queue + retry
│   ├── spa-detector.js       # SPA auto-tracking
│   └── api.js                # Public API (goal, etc.)
├── rollup.config.js          # Build config
├── package.json
└── dist/
    ├── datafast.min.js       # Production build
    └── datafast.min.js.br    # Brotli compressed
```

**API Surface:**
```javascript
// Auto-initialized on load
window.datafast = {
    // Track custom goal
    goal: (name, value) => void,

    // Track custom event
    track: (eventName, properties) => void,

    // Enable debug mode
    debug: (enabled) => void
};
```

---

### 3.2 Ingestion API (POST /track)

**Purpose:** High-throughput API to receive and process analytics events from tracking scripts.

**Responsibilities:**
- Validate event payloads (Zod schema)
- Enrich events (parse user agent, resolve IP to country)
- Deduplicate events (prevent double-counting)
- Batch events for bulk insert to ClickHouse
- Handle retries and errors gracefully

**Technology:**
- Runtime: Node.js 20+ or Bun
- Framework: Hono (lightweight, fast)
- Queue: BullMQ (Redis-backed)
- Validation: Zod

**Endpoints:**
```
POST /track
  Body: { website_id, event_type, page_url, referrer, utm_*, ... }
  Response: 202 Accepted { id: "evt_abc123" }
```

**Event Schema (Zod):**
```typescript
const EventSchema = z.object({
    website_id: z.string().uuid(),
    event_type: z.enum(['pageview', 'goal', 'purchase']),
    visitor_id: z.string().uuid(),
    session_id: z.string().uuid(),
    timestamp: z.number(),
    page_url: z.string().url(),
    referrer: z.string().url().optional(),
    utm_source: z.string().max(100).optional(),
    utm_medium: z.string().max(100).optional(),
    utm_campaign: z.string().max(200).optional(),
    device_type: z.enum(['mobile', 'desktop', 'tablet']),
    browser: z.string().max(50),
    os: z.string().max(50),
    country: z.string().length(2),
    revenue: z.number().optional(),
    goal_name: z.string().max(100).optional(),
});
```

**Processing Pipeline:**
```
Request → Validate → Enrich → Dedupe → Queue → Batch Insert → Response (202)
```

**Performance Targets:**
- Latency: <50ms (p95)
- Throughput: 10,000 events/second
- Batch size: 1,000 events per INSERT
- Batch interval: 1 second

---

### 3.3 Query API (GET /api/metrics, etc.)

**Purpose:** Serve aggregated analytics data to dashboard and API users.

**Responsibilities:**
- Query ClickHouse for metrics (visitors, revenue, RPV)
- Apply filters (date range, source, device)
- Cache frequently accessed data (Redis)
- Enforce rate limits (per-user quotas)
- Generate API responses (JSON)

**Technology:**
- Runtime: Node.js 20+
- Framework: Next.js API Routes
- ORM: Raw SQL (ClickHouse client), Prisma (PostgreSQL)
- Cache: Redis (Upstash)
- Rate Limit: Upstash Rate Limit

**Key Endpoints:**
```
GET /api/metrics?start_date=...&end_date=...
GET /api/visitors?start_date=...&limit=100&offset=0
GET /api/revenue?start_date=...&source=google
GET /api/goals?goal_name=signup
GET /api/funnels/:id
```

**Example Query (ClickHouse):**
```sql
-- Get metrics by source
SELECT
    utm_source,
    count(DISTINCT visitor_id) AS visitors,
    countIf(event_type = 'purchase') AS purchases,
    sum(revenue) AS total_revenue,
    total_revenue / visitors AS rpv
FROM events
WHERE website_id = :website_id
    AND timestamp >= :start_date
    AND timestamp <= :end_date
GROUP BY utm_source
ORDER BY total_revenue DESC;
```

**Caching Strategy:**
- Cache key: `metrics:{website_id}:{start_date}:{end_date}:{filters}`
- TTL: 5 minutes (balance freshness vs. load)
- Invalidation: On new data (webhook or polling)

---

### 3.4 Authentication Service (NextAuth.js)

**Purpose:** Manage user authentication, sessions, and authorization.

**Responsibilities:**
- OAuth login (Google, X/Twitter)
- Email/password login (bcrypt hashing)
- Session management (database sessions)
- Token generation (API keys)
- Password reset, email verification

**Technology:**
- NextAuth.js (OAuth + credentials)
- Prisma (PostgreSQL adapter)
- bcrypt (password hashing, 12 rounds)
- Upstash Rate Limit (prevent brute force)

**Session Storage:**
- Database sessions (PostgreSQL `sessions` table)
- HTTP-only cookies (SameSite=Lax, Secure)
- Expiry: 30 days

**Security Measures:**
- Rate limiting: 5 login attempts per 15 min
- CAPTCHA: Cloudflare Turnstile (after 3 failures)
- Password requirements: 8+ chars, 1 number
- Email verification: Required within 7 days

---

### 3.5 Billing Service (Stripe Webhooks)

**Purpose:** Handle subscription lifecycle events from Stripe.

**Responsibilities:**
- Process Stripe webhooks (subscription created, updated, canceled)
- Update user subscriptions in PostgreSQL
- Handle prorated billing (automatic via Stripe)
- Trigger events (email on trial ending, upgrade success)

**Technology:**
- Next.js API Route (`/api/webhooks/stripe`)
- Stripe SDK (Node.js)
- Webhook signature verification (HMAC)

**Webhook Events:**
```
customer.subscription.created    → Start subscription
customer.subscription.updated    → Plan change (upgrade/downgrade)
customer.subscription.deleted    → Cancellation
invoice.paid                     → Renewal success
invoice.payment_failed           → Payment failed (retry logic)
```

**Processing Flow:**
```
Stripe → Webhook → Verify Signature → Parse Event → Update DB → Send Email → Respond 200
```

---

## 4. Data Flow

### 4.1 Event Ingestion Flow

```mermaid
sequenceDiagram
    participant V as Visitor
    participant S as Tracking Script
    participant I as Ingestion API
    participant Q as BullMQ Queue
    participant C as ClickHouse
    participant D as Dashboard

    V->>S: Visits page
    S->>S: Capture: URL, referrer, UTM
    S->>S: Fingerprint visitor
    S->>I: POST /track (batched)
    I->>I: Validate payload (Zod)
    I->>I: Enrich (user agent → device)
    I->>I: Dedupe (Redis check)
    I->>Q: Queue event
    I-->>S: 202 Accepted

    Note over Q: Every 1s or 1000 events
    Q->>C: Bulk INSERT (batch)
    C-->>Q: Success

    Note over D: User checks dashboard
    D->>C: SELECT metrics
    C-->>D: Return data
    D->>D: Render charts
```

### 4.2 Revenue Attribution Flow

```mermaid
sequenceDiagram
    participant V as Visitor
    participant S as Tracking Script
    participant C as ClickHouse
    participant ST as Stripe
    participant W as Webhook Handler
    participant P as PostgreSQL
    participant D as Dashboard

    V->>S: Visits site via Google
    S->>C: Track pageview (utm_source=google)
    C->>C: Store visitor_id + session

    V->>ST: Purchases ($99)
    ST->>W: Webhook: checkout.session.completed
    W->>W: Extract customer_email
    W->>C: Query: visitor_id WHERE email
    C-->>W: Return visitor_id + session
    W->>W: Attribute to utm_source=google
    W->>C: INSERT purchase event
    W->>P: Update user stats
    W-->>ST: 200 OK

    D->>C: Query revenue by source
    C-->>D: Google: $99
    D->>D: Display "Google: $99"
```

### 4.3 Real-Time Visitor Flow

```mermaid
sequenceDiagram
    participant V as Visitor
    participant S as Tracking Script
    participant I as Ingestion API
    participant R as Redis PubSub
    participant W as WebSocket Server
    participant D as Dashboard

    V->>S: Arrives on site
    S->>I: POST /track (pageview)
    I->>R: PUBLISH visitor:new (event data)
    R->>W: Forward to WebSocket clients
    W->>D: Send visitor data
    D->>D: Animate pin on map
    D->>D: Update visitor count
```

---

## 5. Infrastructure

### 5.1 Deployment Architecture

```
┌─────────────────────────────────────────────┐
│  Cloudflare CDN                             │
│  - tracking.js (edge cached, Brotli)        │
│  - Static assets (images, fonts)            │
└─────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  Vercel Edge Network                        │
│  - Next.js frontend (SSR + ISR)             │
│  - API routes (serverless functions)        │
│  - Auto-scaling, zero config                │
└─────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  Backend Services (Fly.io / Railway)        │
│  - Ingestion API (Node.js, 2+ instances)    │
│  - WebSocket Server (separate process)      │
│  - Worker: Batch event processor            │
└─────────────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│  Databases                                  │
│  - ClickHouse (single node → cluster)       │
│  - PostgreSQL (Supabase / RDS)              │
│  - Redis (Upstash, serverless)              │
└─────────────────────────────────────────────┘
```

### 5.2 Environment Setup

**Development:**
- Local Next.js dev server (`npm run dev`)
- Local ClickHouse (Docker)
- Local PostgreSQL (Docker)
- Local Redis (Docker)

**Staging:**
- Vercel preview deployments (PR-based)
- Shared ClickHouse instance
- Staging PostgreSQL database
- Staging Stripe (test mode)

**Production:**
- Vercel production deployment
- ClickHouse (managed or self-hosted)
- PostgreSQL (Supabase/RDS)
- Redis (Upstash)
- Stripe (live mode)

### 5.3 Scaling Strategy

**Phase 1: Single-Node (0-10K users)**
- Vercel: Auto-scales serverless functions
- ClickHouse: Single instance (8 CPU, 32 GB RAM)
- PostgreSQL: Small instance (2 CPU, 8 GB RAM)
- Cost: ~$200-300/month

**Phase 2: Vertical Scaling (10K-50K users)**
- ClickHouse: Upgrade to 16 CPU, 64 GB RAM
- PostgreSQL: Upgrade to 4 CPU, 16 GB RAM
- Add read replicas (PostgreSQL)
- Cost: ~$500-800/month

**Phase 3: Horizontal Scaling (50K+ users)**
- ClickHouse: Cluster with sharding + replication (3+ nodes)
- PostgreSQL: Primary + 2 read replicas
- Redis: Cluster mode
- Load balancer for ingestion API (multiple instances)
- Cost: ~$1,500-3,000/month

---

## 6. Security Architecture

### 6.1 Security Layers

**Application Security:**
- Input validation (Zod schemas)
- SQL injection prevention (parameterized queries)
- XSS prevention (React auto-escaping, CSP headers)
- CSRF protection (SameSite cookies)
- Rate limiting (per-IP, per-user)

**Authentication Security:**
- bcrypt password hashing (12 rounds)
- HTTP-only, Secure cookies
- OAuth 2.0 (Google, X)
- API token rotation (manual, future: automatic)
- Session expiry (30 days)

**Data Security:**
- TLS 1.3 (in transit)
- AES-256 (at rest, database encryption)
- IP anonymization (last octet hashed)
- No PII in ClickHouse (email hashed if needed)

**Infrastructure Security:**
- Firewall rules (allow only necessary ports)
- VPC/private networks (databases not public)
- Secrets management (Vercel env vars, AWS Secrets Manager)
- DDoS protection (Cloudflare)

### 6.2 Threat Model

| Threat | Mitigation |
|--------|------------|
| **DDoS on /track** | Cloudflare rate limiting, edge caching |
| **Credential stuffing** | Rate limiting, CAPTCHA, 2FA (future) |
| **SQL injection** | Parameterized queries, ORM (Prisma) |
| **XSS attacks** | React auto-escaping, CSP headers |
| **Data leaks** | Encryption at rest, access logs, RBAC |
| **API key theft** | HTTPS-only, key rotation, IP allowlist (optional) |

---

## 7. Scalability Strategy

### 7.1 Database Sharding (ClickHouse)

**When:** >100M events, >10K events/second

**Strategy:**
- Shard by `website_id` (each website's data on specific nodes)
- Replicate for high availability (3 replicas)
- Materialized views for common queries (pre-aggregation)

**Implementation:**
```sql
CREATE TABLE events_distributed AS events
ENGINE = Distributed(cluster, default, events, rand());

CREATE TABLE events ON CLUSTER cluster (
    event_id UUID,
    website_id UUID,
    ...
) ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(timestamp)
ORDER BY (website_id, timestamp);
```

### 7.2 API Caching

**Strategy:**
- CDN caching for public dashboards (Vercel Edge)
- Redis caching for authenticated APIs (5-minute TTL)
- Materialized views in ClickHouse (pre-aggregated metrics)

**Cache Hierarchy:**
```
Request → Vercel Edge Cache → Redis → ClickHouse (materialized view) → ClickHouse (raw)
```

### 7.3 Read Replicas

**When:** Dashboard queries slow down main database

**Strategy:**
- PostgreSQL: 1 primary + 2 read replicas (user queries)
- ClickHouse: Replicas for query distribution
- Load balancer distributes reads across replicas

---

## Summary

**Architecture Principles:**
- **Simplicity:** Use managed services where possible (Vercel, Upstash)
- **Performance:** Edge caching, batching, indexing
- **Scalability:** Vertical first, horizontal when needed
- **Reliability:** Replication, backups, monitoring
- **Security:** Defense in depth, least privilege

**Technology Choices:**
- **Frontend:** Next.js (React, TypeScript)
- **Backend:** Node.js/Bun, Hono, Express
- **Databases:** ClickHouse (events), PostgreSQL (config), Redis (cache)
- **Hosting:** Vercel (frontend), Fly.io/Railway (backend)
- **Observability:** Sentry, Axiom, Prometheus

**Next Steps:**
- Phase 0: Validate architecture with prototype
- Phase 1: Build MVP with single-node setup
- Phase 2: Scale vertically, add caching
- Phase 3: Horizontal scaling, distributed systems

---

**Document Status:** ✅ Ready for Review
**Last Updated:** November 18, 2025
