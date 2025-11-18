# ADR-006: API Design Approach

**Date:** 2025-11-18
**Status:** Accepted
**Deciders:** Engineering Team, Product Lead
**Technical Story:** [FR-8 - Custom API, US-8.1 - REST API for Metrics]

## Context and Problem Statement

DataFa.st needs a public API for power users to:
- Query analytics data programmatically
- Build custom dashboards
- Automate workflows (e.g., Slack alerts when revenue > $X)
- Integrate with tools (Zapier, Make, n8n)

The API must be:
- **Simple** for indie developers to use
- **Fast** (<500ms response times)
- **Secure** (authentication, rate limiting)
- **Scalable** (handle 1000+ req/min across all users)
- **Well-documented** (OpenAPI spec, examples)

We must choose between REST, GraphQL, or tRPC.

## Decision Drivers

- **Developer Experience:** Easy to understand, test with curl/Postman
- **Performance:** <500ms (p95) for standard queries
- **Simplicity:** No over-engineering for indie use cases
- **Documentation:** Auto-generated docs (Swagger/OpenAPI)
- **Rate Limiting:** Per-user quotas (100 req/min)
- **Versioning:** Support /v1, /v2 without breaking changes
- **Ecosystem:** Works with Zapier, Make, n8n (webhook platforms)
- **Team Familiarity:** Next.js/TypeScript experience

## Considered Options

1. **REST API** (JSON over HTTP)
2. **GraphQL** (Query language for APIs)
3. **tRPC** (Type-safe RPC for TypeScript)
4. **gRPC** (High-performance RPC)

## Decision Outcome

**Chosen option:** "REST API (JSON over HTTP)"

**Rationale:**
REST is the **industry standard** for public APIs, offering:
- **Universal compatibility:** Works with every language/tool
- **Simple testing:** curl, Postman, HTTPie
- **Zapier/Make support:** Native HTTP request actions
- **Documentation:** OpenAPI/Swagger auto-generates beautiful docs
- **Caching:** HTTP caching headers (CDN-friendly)
- **Familiarity:** Team knows REST, no learning curve

GraphQL/tRPC add complexity without significant benefits for DataFa.st's use case (simple metrics queries).

### Positive Consequences

- **Easy to use:** Developers understand REST (vs. GraphQL schemas)
- **Great tooling:** Postman, Insomnia, OpenAPI generators
- **Zapier-ready:** HTTP requests work out-of-the-box
- **Caching:** Use Vercel Edge Cache for GET requests
- **Simple versioning:** /v1, /v2 in URL path
- **No vendor lock-in:** REST is language-agnostic

### Negative Consequences

- **Over-fetching:** Clients get full objects (vs. GraphQL's precise fields)
- **Multiple requests:** No batching (vs. GraphQL mutations)
- **Schema drift:** Manual docs can get out of sync (mitigated by OpenAPI)
- **Less type-safe:** No built-in types (vs. tRPC)

**Mitigation:** Use OpenAPI/TypeScript to generate client SDKs (Phase 2).

## Pros and Cons of the Options

### REST API

**Pros:**
- **Universal:** Every language has HTTP libraries
- **Simple:** curl works instantly, no setup
- **Tooling:** Postman, Insomnia, Swagger, Redoc
- **Caching:** HTTP headers (ETag, Cache-Control)
- **Zapier/Make:** Native support for HTTP requests
- **Scalable:** CDN caching for GET endpoints

**Cons:**
- Over-fetching data (send full objects)
- No built-in batching (multiple requests)
- Schema validation requires manual OpenAPI maintenance

---

### GraphQL

**Pros:**
- **Precise queries:** Fetch only needed fields
- **Single endpoint:** /graphql for all queries
- **Batching:** Multiple queries in one request
- **Type-safe:** Schema enforces types
- **Introspection:** Self-documenting

**Cons:**
- **Complex:** Learning curve for clients (vs. REST)
- **Tooling overhead:** Requires GraphQL client (Apollo, urql)
- **Zapier/Make:** Harder to integrate (need GraphQL plugin)
- **Caching:** More complex than HTTP caching
- **N+1 queries:** DataLoader required (adds complexity)
- **Overkill:** Most users just need `GET /metrics?start_date=...`

**Verdict:** Too complex for DataFa.st's simple use case.

---

### tRPC

**Pros:**
- **Type-safe:** End-to-end TypeScript types
- **No code generation:** Types inferred automatically
- **Great DX:** Autocomplete in VS Code
- **Fast:** RPC is faster than REST for internal calls

**Cons:**
- **TypeScript-only:** Not useful for Python, Go, Ruby users
- **No Zapier/Make support:** Requires HTTP wrapper
- **Internal-only:** Best for monorepos, not public APIs
- **Limited ecosystem:** No Postman, no OpenAPI

**Verdict:** Excellent for internal Next.js → backend calls, but not for public API.

---

### gRPC

**Pros:**
- **High performance:** Protocol Buffers are faster than JSON
- **Type-safe:** `.proto` schema files
- **Streaming:** Bi-directional streaming

**Cons:**
- **Binary protocol:** Can't test with curl
- **Tooling:** Requires gRPC clients (no Postman)
- **HTTP/2 required:** Not all clients support
- **Overkill:** Performance gains don't justify complexity for analytics API

**Verdict:** Great for microservices, not public APIs.

---

## Technical Implementation

### API Structure

**Base URL:** `https://api.datafa.st/v1`

**Endpoints:**

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/metrics` | Aggregated metrics (visitors, revenue, RPV) |
| GET | `/visitors` | Visitor list with filters |
| GET | `/revenue` | Revenue breakdown by source |
| GET | `/goals` | Goal completions |
| GET | `/funnels/:id` | Funnel analysis |
| GET | `/websites` | List user's websites |
| GET | `/websites/:id` | Website details |
| POST | `/webhooks` | Register webhook URL |
| DELETE | `/webhooks/:id` | Delete webhook |

---

### Authentication

**Method:** Bearer token (API keys)

**Request:**
```bash
curl -H "Authorization: Bearer sk_live_abc123..." \
     https://api.datafa.st/v1/metrics?start_date=2025-11-01
```

**Token Generation:**
- User creates token in dashboard: Settings → API Keys
- Token shown once, stored hashed (bcrypt) in database
- Token scopes (future): read-only, read-write

**Security:**
- HTTPS-only (reject HTTP requests)
- Rate limiting: 100 req/min per token
- IP allowlisting (optional, enterprise feature)

---

### Response Format

**Success (200 OK):**
```json
{
    "data": {
        "visitors": 5000,
        "revenue": 2500.00,
        "rpv": 0.50,
        "conversion_rate": 0.02
    },
    "meta": {
        "start_date": "2025-11-01",
        "end_date": "2025-11-30",
        "website_id": "abc123"
    }
}
```

**Error (4xx, 5xx):**
```json
{
    "error": {
        "code": "INVALID_DATE_RANGE",
        "message": "start_date must be before end_date",
        "docs": "https://docs.datafa.st/api/errors#INVALID_DATE_RANGE"
    }
}
```

**HTTP Status Codes:**
- 200: Success
- 400: Bad request (invalid parameters)
- 401: Unauthorized (invalid or missing token)
- 403: Forbidden (valid token, insufficient permissions)
- 429: Rate limit exceeded
- 500: Internal server error

---

### Query Parameters

**Example: GET /metrics**

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `start_date` | string (ISO 8601) | Yes | Start of date range | 2025-11-01 |
| `end_date` | string (ISO 8601) | Yes | End of date range | 2025-11-30 |
| `website_id` | UUID | No | Filter by website (default: all) | abc123... |
| `source` | string | No | Filter by traffic source | google |
| `device` | enum | No | Filter by device (mobile, desktop, tablet) | mobile |
| `goal` | string | No | Filter by goal name | signup |

**Example Request:**
```bash
GET /v1/metrics?start_date=2025-11-01&end_date=2025-11-30&source=google&device=mobile
```

**Example Response:**
```json
{
    "data": {
        "visitors": 1200,
        "revenue": 600.00,
        "rpv": 0.50,
        "conversion_rate": 0.025,
        "breakdown": [
            {
                "date": "2025-11-01",
                "visitors": 40,
                "revenue": 20.00
            },
            ...
        ]
    }
}
```

---

### Pagination

**Large datasets:** Use limit + offset

**Example:**
```bash
GET /v1/visitors?start_date=2025-11-01&limit=100&offset=0
```

**Response:**
```json
{
    "data": [ /* 100 visitors */ ],
    "pagination": {
        "limit": 100,
        "offset": 0,
        "total": 5000,
        "has_more": true
    }
}
```

**Limits:**
- Default limit: 100
- Max limit: 1000 per request
- Offset-based (simple, but slow for large offsets)
- Future: Cursor-based pagination (faster for 1M+ records)

---

### Rate Limiting

**Per-User Limits:**
- Starter plan: 100 req/min
- Growth plan: 500 req/min
- Enterprise plan: Custom (e.g., 5000 req/min)

**Headers (in response):**
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1699564800
```

**429 Response:**
```json
{
    "error": {
        "code": "RATE_LIMIT_EXCEEDED",
        "message": "You have exceeded the rate limit of 100 requests per minute. Upgrade your plan for higher limits.",
        "retry_after": 42
    }
}
```

**Implementation:** Upstash Rate Limit (Redis-based)

---

### Documentation

**Tool:** OpenAPI 3.0 + Redoc

**Auto-generation:**
```typescript
// Use ts-to-zod to generate Zod schemas from TypeScript
import { z } from "zod";

export const MetricsQuerySchema = z.object({
    start_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
    end_date: z.string().regex(/^\d{4}-\d{2}-\d{2}$/),
    website_id: z.string().uuid().optional(),
    source: z.string().optional(),
    device: z.enum(["mobile", "desktop", "tablet"]).optional(),
});

export type MetricsQuery = z.infer<typeof MetricsQuerySchema>;
```

**Generate OpenAPI spec:**
```bash
npx openapi-generator-cli generate \
    -i openapi.yaml \
    -g typescript-fetch \
    -o ./sdk/typescript
```

**Docs URL:** https://docs.datafa.st/api

**Features:**
- Interactive playground (try API in browser)
- Code examples (curl, JavaScript, Python, Go)
- Error code reference
- Rate limit details

---

### Webhooks (Future: Phase 3)

**Register Webhook:**
```bash
POST /v1/webhooks
{
    "url": "https://example.com/datafast-webhook",
    "events": ["visitor.created", "purchase.created"],
    "secret": "whsec_abc123" // For signature verification
}
```

**Webhook Payload:**
```json
{
    "id": "evt_abc123",
    "type": "purchase.created",
    "created": 1699564800,
    "data": {
        "visitor_id": "xyz789",
        "revenue": 99.00,
        "source": "google",
        "website_id": "abc123"
    }
}
```

**Signature Verification:**
```python
import hmac
import hashlib

def verify_webhook(payload, signature, secret):
    computed = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(computed, signature)
```

---

## Monitoring & Observability

**Metrics to Track:**
- API response times (p50, p95, p99)
- Error rates by endpoint
- Rate limit hits (per user)
- Most popular endpoints
- Slowest queries

**Tools:**
- Vercel Analytics (built-in)
- Sentry (error tracking)
- Axiom (structured logging)

**Alerts:**
- p95 latency >1s → Slack notification
- Error rate >1% → Page on-call
- Rate limit hit rate >10% of users → Email product team

---

## Future Enhancements (Phase 2/3)

- [ ] **Client SDKs:** TypeScript, Python, Go (auto-generated from OpenAPI)
- [ ] **GraphQL layer:** Optional for advanced users (uses REST under the hood)
- [ ] **Streaming:** Server-Sent Events (SSE) for real-time metrics
- [ ] **Batch endpoints:** `/batch` to execute multiple queries in one request
- [ ] **Field selection:** `?fields=visitors,revenue` to reduce payload
- [ ] **API playground:** In-dashboard API explorer (like Stripe)

---

## Links

- [OpenAPI 3.0 Specification](https://swagger.io/specification/)
- [Redoc Documentation](https://redoc.ly/)
- [Upstash Rate Limit](https://upstash.com/docs/redis/features/ratelimiting)
- [REST API Best Practices](https://restfulapi.net/)
- [PRD Section 3: FR-8 Custom API](../prd/DATAFAST-PRD.md#fr-8-custom-api)

---

**Reviewed by:** [Pending]
**Approved by:** [Pending]
