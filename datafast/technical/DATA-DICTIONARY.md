# DataFa.st Data Dictionary
## Database Schemas & Data Models

**Version:** 1.0
**Date:** November 18, 2025
**Status:** Draft

---

## Table of Contents
1. [Overview](#overview)
2. [ClickHouse Schemas (Analytics)](#clickhouse-schemas-analytics)
3. [PostgreSQL Schemas (Config)](#postgresql-schemas-config)
4. [Redis Data Structures](#redis-data-structures)
5. [Data Flows](#data-flows)
6. [Data Retention](#data-retention)

---

## Overview

DataFa.st uses a polyglot persistence strategy:

| Database | Purpose | Data Types |
|----------|---------|------------|
| **ClickHouse** | Analytics events | Pageviews, goals, purchases, sessions |
| **PostgreSQL** | Configuration | Users, websites, integrations, billing |
| **Redis** | Cache & Sessions | Session tokens, rate limits, real-time data |

---

## ClickHouse Schemas (Analytics)

### events

**Purpose:** Store all analytics events (pageviews, goals, purchases)

**Engine:** MergeTree with monthly partitioning

```sql
CREATE TABLE events (
    -- Identifiers
    event_id UUID DEFAULT generateUUIDv4(),
    website_id UUID NOT NULL,
    visitor_id UUID NOT NULL,
    session_id UUID NOT NULL,

    -- Event Info
    event_type Enum8('pageview' = 1, 'goal' = 2, 'purchase' = 3),
    timestamp DateTime NOT NULL,

    -- Page Data
    page_url String NOT NULL,
    page_title Nullable(String),
    referrer String DEFAULT '',

    -- UTM Parameters
    utm_source LowCardinality(Nullable(String)),
    utm_medium LowCardinality(Nullable(String)),
    utm_campaign Nullable(String),
    utm_content Nullable(String),
    utm_term Nullable(String),

    -- Device Info
    device_type Enum8('desktop' = 1, 'mobile' = 2, 'tablet' = 3),
    browser LowCardinality(String),
    browser_version String,
    os LowCardinality(String),
    os_version String,
    screen_width UInt16,
    screen_height UInt16,

    -- Location
    country FixedString(2),
    region LowCardinality(Nullable(String)),
    city Nullable(String),

    -- Revenue (for purchase events)
    revenue Nullable(Decimal64(2)),
    currency LowCardinality(FixedString(3)) DEFAULT 'USD',

    -- Goals
    goal_name LowCardinality(Nullable(String)),
    goal_value Nullable(Decimal64(2)),

    -- Metadata
    is_bounce UInt8 DEFAULT 0,
    session_duration UInt32 DEFAULT 0,
    page_load_time UInt16 DEFAULT 0
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (website_id, timestamp, event_id)
TTL timestamp + INTERVAL 2 YEAR
SETTINGS index_granularity = 8192;
```

**Indexes:**
```sql
-- Optimize common queries
CREATE INDEX idx_visitor ON events (visitor_id) TYPE bloom_filter GRANULARITY 1;
CREATE INDEX idx_utm_source ON events (utm_source) TYPE set(100) GRANULARITY 1;
CREATE INDEX idx_goal_name ON events (goal_name) TYPE set(50) GRANULARITY 1;
```

**Column Descriptions:**

| Column | Type | Description |
|--------|------|-------------|
| `event_id` | UUID | Unique event identifier |
| `website_id` | UUID | Foreign key to websites table |
| `visitor_id` | UUID | Fingerprint-based visitor identifier |
| `session_id` | UUID | Groups events within 30-min window |
| `event_type` | Enum | pageview, goal, or purchase |
| `timestamp` | DateTime | Event occurrence time (UTC) |
| `page_url` | String | Full URL of the page |
| `referrer` | String | HTTP referrer header |
| `utm_source` | String | UTM source parameter |
| `utm_medium` | String | UTM medium parameter |
| `utm_campaign` | String | UTM campaign parameter |
| `device_type` | Enum | Device category |
| `browser` | String | Browser name (Chrome, Firefox, etc.) |
| `os` | String | Operating system |
| `country` | FixedString(2) | ISO 3166-1 alpha-2 code |
| `revenue` | Decimal | Purchase amount (null if not purchase) |
| `goal_name` | String | Custom goal identifier |
| `is_bounce` | UInt8 | 1 if single-page session |

---

### sessions

**Purpose:** Aggregated session data for faster queries

**Engine:** SummingMergeTree (auto-aggregates on insert)

```sql
CREATE TABLE sessions (
    -- Keys
    website_id UUID NOT NULL,
    visitor_id UUID NOT NULL,
    session_id UUID NOT NULL,
    session_start DateTime NOT NULL,

    -- Aggregated Metrics
    pageviews UInt32,
    duration UInt32,
    is_bounce UInt8,

    -- First Touch Attribution
    entry_page String,
    referrer String,
    utm_source LowCardinality(Nullable(String)),
    utm_medium LowCardinality(Nullable(String)),
    utm_campaign Nullable(String),

    -- Device/Location (first event)
    device_type Enum8('desktop' = 1, 'mobile' = 2, 'tablet' = 3),
    browser LowCardinality(String),
    country FixedString(2),

    -- Goals & Revenue
    goals_completed UInt8,
    revenue Decimal64(2)
) ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(session_start)
ORDER BY (website_id, session_start, session_id);
```

---

### daily_metrics

**Purpose:** Pre-aggregated daily metrics (materialized view)

```sql
CREATE MATERIALIZED VIEW daily_metrics
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (website_id, date, utm_source, device_type)
POPULATE
AS SELECT
    website_id,
    toDate(timestamp) AS date,
    utm_source,
    device_type,

    -- Counts
    count() AS events,
    uniq(visitor_id) AS visitors,
    uniq(session_id) AS sessions,

    -- Revenue
    sumIf(revenue, event_type = 'purchase') AS total_revenue,
    countIf(event_type = 'purchase') AS purchases,

    -- Goals
    countIf(event_type = 'goal') AS goal_completions,

    -- Calculated
    sum(is_bounce) AS bounces
FROM events
GROUP BY website_id, date, utm_source, device_type;
```

---

### goal_conversions

**Purpose:** Track goal completion funnels

```sql
CREATE TABLE goal_conversions (
    website_id UUID NOT NULL,
    visitor_id UUID NOT NULL,
    goal_name LowCardinality(String) NOT NULL,
    converted_at DateTime NOT NULL,

    -- Attribution
    session_id UUID NOT NULL,
    utm_source LowCardinality(Nullable(String)),

    -- Value
    goal_value Nullable(Decimal64(2)),

    -- Revenue (within attribution window)
    attributed_revenue Nullable(Decimal64(2))
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(converted_at)
ORDER BY (website_id, goal_name, converted_at);
```

---

## PostgreSQL Schemas (Config)

### users

**Purpose:** User accounts and authentication

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Auth
    email VARCHAR(255) UNIQUE NOT NULL,
    email_verified TIMESTAMP,
    password_hash VARCHAR(255),

    -- OAuth
    oauth_provider VARCHAR(50),
    oauth_provider_id VARCHAR(255),

    -- Profile
    name VARCHAR(255),
    avatar_url VARCHAR(500),

    -- Billing
    stripe_customer_id VARCHAR(255) UNIQUE,
    subscription_plan VARCHAR(50) DEFAULT 'trial',
    subscription_status VARCHAR(50) DEFAULT 'trialing',
    trial_ends_at TIMESTAMP DEFAULT NOW() + INTERVAL '14 days',
    event_limit INTEGER DEFAULT 10000,
    events_used INTEGER DEFAULT 0,
    events_reset_at TIMESTAMP DEFAULT NOW() + INTERVAL '1 month',

    -- Preferences
    timezone VARCHAR(50) DEFAULT 'UTC',
    currency CHAR(3) DEFAULT 'USD',

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    last_login_at TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_stripe ON users(stripe_customer_id);
```

**Column Descriptions:**

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `email` | VARCHAR | User's email address |
| `email_verified` | TIMESTAMP | When email was verified |
| `password_hash` | VARCHAR | bcrypt hash (null if OAuth) |
| `oauth_provider` | VARCHAR | 'google', 'twitter', or null |
| `stripe_customer_id` | VARCHAR | Stripe customer ID |
| `subscription_plan` | VARCHAR | 'trial', 'starter', 'growth', 'enterprise' |
| `subscription_status` | VARCHAR | 'trialing', 'active', 'past_due', 'canceled' |
| `event_limit` | INTEGER | Monthly event quota |
| `events_used` | INTEGER | Events tracked this month |

---

### websites

**Purpose:** Tracked website configurations

```sql
CREATE TABLE websites (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Site Info
    domain VARCHAR(255) NOT NULL,
    name VARCHAR(255),

    -- Tracking
    tracking_id VARCHAR(50) UNIQUE NOT NULL,
    is_active BOOLEAN DEFAULT true,

    -- Settings
    public_dashboard BOOLEAN DEFAULT false,
    public_dashboard_url VARCHAR(100) UNIQUE,
    attribution_window INTEGER DEFAULT 30,

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_websites_user ON websites(user_id);
CREATE INDEX idx_websites_domain ON websites(domain);
CREATE INDEX idx_websites_tracking ON websites(tracking_id);

-- Constraint: max sites per user plan
CREATE OR REPLACE FUNCTION check_website_limit() RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT COUNT(*) FROM websites WHERE user_id = NEW.user_id) >=
       (SELECT CASE subscription_plan
            WHEN 'starter' THEN 1
            WHEN 'growth' THEN 30
            ELSE 100 END
        FROM users WHERE id = NEW.user_id) THEN
        RAISE EXCEPTION 'Website limit reached for your plan';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER website_limit_trigger
    BEFORE INSERT ON websites
    FOR EACH ROW EXECUTE FUNCTION check_website_limit();
```

---

### integrations

**Purpose:** Payment processor integrations

```sql
CREATE TABLE integrations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    website_id UUID NOT NULL REFERENCES websites(id) ON DELETE CASCADE,

    -- Provider
    provider VARCHAR(50) NOT NULL, -- 'stripe', 'lemon_squeezy', 'polar', 'shopify'

    -- Credentials (encrypted)
    access_token TEXT, -- Encrypted
    refresh_token TEXT, -- Encrypted
    api_key TEXT, -- Encrypted

    -- Metadata
    account_id VARCHAR(255),
    account_name VARCHAR(255),
    webhook_secret TEXT,

    -- Status
    is_active BOOLEAN DEFAULT true,
    last_sync_at TIMESTAMP,
    sync_error TEXT,

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_integrations_website ON integrations(website_id);
CREATE UNIQUE INDEX idx_integrations_unique ON integrations(website_id, provider);
```

---

### goals

**Purpose:** Custom goal definitions

```sql
CREATE TABLE goals (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    website_id UUID NOT NULL REFERENCES websites(id) ON DELETE CASCADE,

    -- Goal Info
    name VARCHAR(100) NOT NULL,
    display_name VARCHAR(255),

    -- Tracking Method
    track_method VARCHAR(50) NOT NULL, -- 'data_attribute', 'javascript', 'page_url'
    match_pattern VARCHAR(500), -- URL pattern or selector

    -- Value
    monetary_value DECIMAL(10, 2),

    -- Status
    is_active BOOLEAN DEFAULT true,

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_goals_website ON goals(website_id);
CREATE UNIQUE INDEX idx_goals_unique ON goals(website_id, name);
```

---

### funnels

**Purpose:** Conversion funnel definitions

```sql
CREATE TABLE funnels (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    website_id UUID NOT NULL REFERENCES websites(id) ON DELETE CASCADE,

    -- Funnel Info
    name VARCHAR(255) NOT NULL,

    -- Steps (ordered JSON array)
    steps JSONB NOT NULL,
    -- Example: [
    --   {"type": "page", "pattern": "/"},
    --   {"type": "page", "pattern": "/pricing"},
    --   {"type": "goal", "name": "signup"}
    -- ]

    -- Settings
    time_window_hours INTEGER DEFAULT 24,

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_funnels_website ON funnels(website_id);
```

---

### sessions (Auth)

**Purpose:** NextAuth.js session storage

```sql
CREATE TABLE auth_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    session_token VARCHAR(255) UNIQUE NOT NULL,
    expires TIMESTAMP NOT NULL,

    -- Metadata
    ip_address INET,
    user_agent TEXT,

    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_auth_sessions_token ON auth_sessions(session_token);
CREATE INDEX idx_auth_sessions_user ON auth_sessions(user_id);
```

---

### api_tokens

**Purpose:** API access tokens

```sql
CREATE TABLE api_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,

    -- Token
    name VARCHAR(255) NOT NULL,
    token_hash VARCHAR(255) NOT NULL, -- bcrypt hash
    token_prefix CHAR(8) NOT NULL, -- First 8 chars for identification

    -- Permissions
    scopes VARCHAR(255)[] DEFAULT ARRAY['read'], -- 'read', 'write'

    -- Usage
    last_used_at TIMESTAMP,
    request_count INTEGER DEFAULT 0,

    -- Status
    is_active BOOLEAN DEFAULT true,
    expires_at TIMESTAMP,

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_api_tokens_user ON api_tokens(user_id);
CREATE INDEX idx_api_tokens_prefix ON api_tokens(token_prefix);
```

---

### webhooks

**Purpose:** User-configured webhooks

```sql
CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    website_id UUID NOT NULL REFERENCES websites(id) ON DELETE CASCADE,

    -- Config
    url VARCHAR(500) NOT NULL,
    secret VARCHAR(255) NOT NULL, -- For HMAC signature
    events VARCHAR(100)[] NOT NULL, -- ['visitor.created', 'purchase.created']

    -- Status
    is_active BOOLEAN DEFAULT true,
    last_triggered_at TIMESTAMP,
    failure_count INTEGER DEFAULT 0,

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_webhooks_website ON webhooks(website_id);
```

---

### notes

**Purpose:** Timeline annotations

```sql
CREATE TABLE notes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    website_id UUID NOT NULL REFERENCES websites(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),

    -- Content
    content TEXT NOT NULL,
    note_date DATE NOT NULL,

    -- Tags
    tags VARCHAR(50)[],

    -- Timestamps
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_notes_website_date ON notes(website_id, note_date);
```

---

## Redis Data Structures

### Session Tokens
```
Key: session:{token}
Type: String
Value: JSON {userId, expires}
TTL: 30 days
```

### Rate Limiting
```
Key: ratelimit:{userId}:{endpoint}
Type: String (counter)
Value: Request count
TTL: 60 seconds
```

### Real-Time Visitors
```
Key: visitors:{websiteId}:active
Type: Sorted Set
Score: Timestamp
Value: JSON {visitorId, page, country, device}
TTL: None (cleaned by cron)
```

### Cache
```
Key: cache:metrics:{websiteId}:{dateRange}:{filters}
Type: String
Value: JSON metrics data
TTL: 5 minutes
```

### Event Queue (BullMQ)
```
Queue: event-ingestion
Job: {eventData}
```

---

## Data Flows

### Event Ingestion
```
Script → API → Validate → Enrich → Queue → Batch Insert → ClickHouse
```

### Dashboard Query
```
Dashboard → API → Cache Check → ClickHouse Query → Cache Store → Response
```

### Revenue Attribution
```
Stripe Webhook → API → Match Visitor → Query Session → Attribute → ClickHouse
```

---

## Data Retention

| Data Type | Retention | Notes |
|-----------|-----------|-------|
| **Events** | 2 years | Configurable (1-5 years) |
| **Sessions** | 2 years | Same as events |
| **Daily Metrics** | 5 years | Aggregated, smaller |
| **Users** | Forever | Until account deletion |
| **Integrations** | Forever | Until disconnected |
| **API Tokens** | Forever | Until revoked |
| **Cache** | 5 minutes | Redis TTL |
| **Auth Sessions** | 30 days | NextAuth default |

### GDPR Data Deletion

```sql
-- Delete user and all associated data
BEGIN;

-- ClickHouse (run separately)
-- ALTER TABLE events DELETE WHERE website_id IN (SELECT id FROM websites WHERE user_id = :userId);

-- PostgreSQL (cascades)
DELETE FROM users WHERE id = :userId;

COMMIT;
```

---

## Schema Migrations

Use Prisma for PostgreSQL migrations:

```bash
# Generate migration
npx prisma migrate dev --name add_webhooks

# Apply to production
npx prisma migrate deploy
```

ClickHouse migrations managed manually (version-controlled SQL files).

---

**Document Status:** ✅ Ready for Review
**Last Updated:** November 18, 2025
