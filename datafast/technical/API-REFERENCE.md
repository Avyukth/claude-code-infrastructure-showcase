# DataFa.st API Reference
## Complete REST API Documentation

**Version:** 1.0
**Base URL:** `https://api.datafa.st/v1`
**Date:** November 18, 2025

---

## Table of Contents
1. [Authentication](#authentication)
2. [Rate Limiting](#rate-limiting)
3. [Error Handling](#error-handling)
4. [Endpoints](#endpoints)
   - [Metrics](#metrics)
   - [Visitors](#visitors)
   - [Revenue](#revenue)
   - [Goals](#goals)
   - [Funnels](#funnels)
   - [Websites](#websites)
   - [Webhooks](#webhooks)
5. [Webhooks (Outbound)](#webhooks-outbound)

---

## Authentication

All API requests require authentication via Bearer token.

### Getting an API Token

1. Log in to DataFa.st dashboard
2. Navigate to **Settings → API Keys**
3. Click **Create API Token**
4. Copy the token (shown only once)

### Using the Token

Include the token in the `Authorization` header:

```bash
curl -H "Authorization: Bearer df_live_your_api_token_here" \
     https://api.datafa.st/v1/metrics
```

### Token Format

- **Prefix:** `df_live_` (production) or `df_test_` (sandbox)
- **Length:** 32 characters after prefix
- **Example:** `df_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`

### Security Best Practices

- Store tokens in environment variables, not code
- Use separate tokens for different applications
- Rotate tokens periodically
- Revoke unused tokens

---

## Rate Limiting

API requests are rate-limited per user account.

### Limits by Plan

| Plan | Requests/Minute | Burst Allowance |
|------|-----------------|-----------------|
| **Starter** | 100 | 2x for 10 seconds |
| **Growth** | 500 | 2x for 10 seconds |
| **Enterprise** | Custom | Custom |

### Response Headers

Every response includes rate limit information:

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1699564800
```

### Rate Limit Exceeded (429)

```json
{
    "error": {
        "code": "RATE_LIMIT_EXCEEDED",
        "message": "You have exceeded 100 requests per minute. Retry after 42 seconds.",
        "retry_after": 42
    }
}
```

**Best Practice:** Implement exponential backoff when receiving 429 errors.

---

## Error Handling

### HTTP Status Codes

| Code | Description |
|------|-------------|
| `200` | Success |
| `201` | Created |
| `400` | Bad Request (invalid parameters) |
| `401` | Unauthorized (invalid or missing token) |
| `403` | Forbidden (insufficient permissions) |
| `404` | Not Found |
| `429` | Rate Limit Exceeded |
| `500` | Internal Server Error |

### Error Response Format

```json
{
    "error": {
        "code": "INVALID_DATE_RANGE",
        "message": "start_date must be before end_date",
        "docs": "https://docs.datafa.st/api/errors#INVALID_DATE_RANGE",
        "request_id": "req_abc123"
    }
}
```

### Common Error Codes

| Code | Description | Resolution |
|------|-------------|------------|
| `INVALID_TOKEN` | API token is invalid or expired | Check token, generate new one |
| `INVALID_DATE_RANGE` | Date range parameters invalid | Ensure start_date < end_date |
| `WEBSITE_NOT_FOUND` | Website ID doesn't exist | Check website_id parameter |
| `RATE_LIMIT_EXCEEDED` | Too many requests | Wait and retry with backoff |
| `INSUFFICIENT_PERMISSIONS` | Token lacks required scope | Use token with correct scopes |

---

## Endpoints

### Metrics

#### GET /metrics

Get aggregated metrics for a website.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `start_date` | string (ISO 8601) | Yes | Start of date range |
| `end_date` | string (ISO 8601) | Yes | End of date range |
| `website_id` | UUID | No | Filter by website (default: all) |
| `source` | string | No | Filter by traffic source |
| `medium` | string | No | Filter by medium |
| `campaign` | string | No | Filter by campaign |
| `device` | enum | No | mobile, desktop, tablet |
| `country` | string (ISO 3166-1) | No | 2-letter country code |
| `interval` | enum | No | day, week, month (default: day) |

**Request:**

```bash
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/metrics?start_date=2025-11-01&end_date=2025-11-30&source=google"
```

**Response:**

```json
{
    "data": {
        "summary": {
            "visitors": 5000,
            "sessions": 6200,
            "pageviews": 15000,
            "purchases": 100,
            "revenue": 2500.00,
            "rpv": 0.50,
            "conversion_rate": 0.02,
            "bounce_rate": 0.45,
            "avg_session_duration": 180
        },
        "breakdown": [
            {
                "date": "2025-11-01",
                "visitors": 160,
                "sessions": 200,
                "pageviews": 500,
                "purchases": 3,
                "revenue": 75.00
            },
            {
                "date": "2025-11-02",
                "visitors": 175,
                "sessions": 220,
                "pageviews": 550,
                "purchases": 4,
                "revenue": 100.00
            }
        ]
    },
    "meta": {
        "start_date": "2025-11-01",
        "end_date": "2025-11-30",
        "website_id": "abc123...",
        "filters": {
            "source": "google"
        },
        "interval": "day",
        "currency": "USD"
    }
}
```

---

### Visitors

#### GET /visitors

Get list of visitors with details.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `start_date` | string | Yes | Start of date range |
| `end_date` | string | Yes | End of date range |
| `website_id` | UUID | No | Filter by website |
| `limit` | integer | No | Results per page (default: 100, max: 1000) |
| `offset` | integer | No | Pagination offset |
| `sort` | enum | No | timestamp, revenue, pageviews |
| `order` | enum | No | asc, desc (default: desc) |

**Request:**

```bash
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/visitors?start_date=2025-11-01&end_date=2025-11-30&limit=50"
```

**Response:**

```json
{
    "data": [
        {
            "visitor_id": "v_abc123...",
            "first_seen": "2025-11-01T10:30:00Z",
            "last_seen": "2025-11-15T14:22:00Z",
            "sessions": 5,
            "pageviews": 23,
            "goals_completed": 2,
            "revenue": 99.00,
            "first_source": "google",
            "last_source": "direct",
            "country": "US",
            "device": "desktop",
            "browser": "Chrome"
        }
    ],
    "pagination": {
        "limit": 50,
        "offset": 0,
        "total": 5000,
        "has_more": true
    }
}
```

---

#### GET /visitors/:id

Get detailed visitor profile.

**Request:**

```bash
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/visitors/v_abc123"
```

**Response:**

```json
{
    "data": {
        "visitor_id": "v_abc123...",
        "first_seen": "2025-11-01T10:30:00Z",
        "last_seen": "2025-11-15T14:22:00Z",
        "total_sessions": 5,
        "total_pageviews": 23,
        "total_revenue": 99.00,
        "goals_completed": ["signup", "purchase"],
        "sessions": [
            {
                "session_id": "s_xyz789...",
                "started_at": "2025-11-15T14:00:00Z",
                "duration": 300,
                "pageviews": 5,
                "source": "direct",
                "device": "desktop",
                "country": "US",
                "pages": [
                    {"url": "/", "time": "14:00:00"},
                    {"url": "/pricing", "time": "14:02:00"},
                    {"url": "/checkout", "time": "14:05:00"}
                ]
            }
        ]
    }
}
```

---

### Revenue

#### GET /revenue

Get revenue breakdown by source.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `start_date` | string | Yes | Start of date range |
| `end_date` | string | Yes | End of date range |
| `website_id` | UUID | No | Filter by website |
| `group_by` | enum | No | source, medium, campaign, device, country |

**Request:**

```bash
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/revenue?start_date=2025-11-01&end_date=2025-11-30&group_by=source"
```

**Response:**

```json
{
    "data": {
        "total_revenue": 5000.00,
        "total_purchases": 100,
        "avg_order_value": 50.00,
        "breakdown": [
            {
                "source": "google",
                "visitors": 2000,
                "purchases": 40,
                "revenue": 2000.00,
                "rpv": 1.00,
                "conversion_rate": 0.02
            },
            {
                "source": "twitter",
                "visitors": 1000,
                "purchases": 30,
                "revenue": 1500.00,
                "rpv": 1.50,
                "conversion_rate": 0.03
            },
            {
                "source": "direct",
                "visitors": 1500,
                "purchases": 30,
                "revenue": 1500.00,
                "rpv": 1.00,
                "conversion_rate": 0.02
            }
        ]
    },
    "meta": {
        "currency": "USD",
        "attribution_model": "last_click",
        "attribution_window_days": 30
    }
}
```

---

### Goals

#### GET /goals

Get goal completion data.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `start_date` | string | Yes | Start of date range |
| `end_date` | string | Yes | End of date range |
| `website_id` | UUID | No | Filter by website |
| `goal_name` | string | No | Filter by specific goal |

**Request:**

```bash
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/goals?start_date=2025-11-01&end_date=2025-11-30"
```

**Response:**

```json
{
    "data": {
        "goals": [
            {
                "name": "signup",
                "display_name": "Newsletter Signup",
                "completions": 450,
                "unique_visitors": 420,
                "conversion_rate": 0.084,
                "attributed_revenue": 2250.00,
                "avg_value": 5.00
            },
            {
                "name": "demo_request",
                "display_name": "Demo Request",
                "completions": 50,
                "unique_visitors": 48,
                "conversion_rate": 0.0096,
                "attributed_revenue": 1500.00,
                "avg_value": 30.00
            }
        ],
        "breakdown": [
            {
                "date": "2025-11-01",
                "signup": 15,
                "demo_request": 2
            }
        ]
    }
}
```

---

### Funnels

#### GET /funnels

List all funnels.

**Request:**

```bash
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/funnels?website_id=abc123"
```

**Response:**

```json
{
    "data": [
        {
            "id": "f_abc123",
            "name": "Signup Funnel",
            "steps": [
                {"type": "page", "pattern": "/"},
                {"type": "page", "pattern": "/pricing"},
                {"type": "goal", "name": "signup"}
            ],
            "created_at": "2025-11-01T00:00:00Z"
        }
    ]
}
```

---

#### GET /funnels/:id

Get funnel analysis.

**Parameters:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| `start_date` | string | Yes | Start of date range |
| `end_date` | string | Yes | End of date range |
| `source` | string | No | Filter by traffic source |
| `device` | enum | No | Filter by device type |

**Request:**

```bash
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/funnels/f_abc123?start_date=2025-11-01&end_date=2025-11-30"
```

**Response:**

```json
{
    "data": {
        "funnel_id": "f_abc123",
        "name": "Signup Funnel",
        "steps": [
            {
                "name": "Homepage",
                "type": "page",
                "pattern": "/",
                "visitors": 1000,
                "drop_off_rate": 0
            },
            {
                "name": "Pricing",
                "type": "page",
                "pattern": "/pricing",
                "visitors": 500,
                "drop_off_rate": 0.50
            },
            {
                "name": "Signup",
                "type": "goal",
                "name": "signup",
                "visitors": 100,
                "drop_off_rate": 0.80
            }
        ],
        "overall_conversion": 0.10,
        "time_window_hours": 24
    }
}
```

---

### Websites

#### GET /websites

List all websites.

**Request:**

```bash
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/websites"
```

**Response:**

```json
{
    "data": [
        {
            "id": "w_abc123",
            "domain": "example.com",
            "name": "My Website",
            "tracking_id": "df_xyz789",
            "is_active": true,
            "created_at": "2025-10-01T00:00:00Z",
            "events_this_month": 5000
        }
    ]
}
```

---

#### GET /websites/:id

Get website details.

**Request:**

```bash
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/websites/w_abc123"
```

**Response:**

```json
{
    "data": {
        "id": "w_abc123",
        "domain": "example.com",
        "name": "My Website",
        "tracking_id": "df_xyz789",
        "is_active": true,
        "public_dashboard": false,
        "attribution_window": 30,
        "integrations": [
            {
                "provider": "stripe",
                "is_active": true,
                "last_sync": "2025-11-18T10:00:00Z"
            }
        ],
        "goals": [
            {
                "name": "signup",
                "display_name": "Newsletter Signup"
            }
        ],
        "created_at": "2025-10-01T00:00:00Z"
    }
}
```

---

### Webhooks

#### POST /webhooks

Create a webhook.

**Request:**

```bash
curl -X POST \
     -H "Authorization: Bearer df_live_your_token" \
     -H "Content-Type: application/json" \
     -d '{
         "website_id": "w_abc123",
         "url": "https://example.com/webhook",
         "events": ["visitor.created", "purchase.created"]
     }' \
     "https://api.datafa.st/v1/webhooks"
```

**Response:**

```json
{
    "data": {
        "id": "wh_abc123",
        "url": "https://example.com/webhook",
        "events": ["visitor.created", "purchase.created"],
        "secret": "whsec_a1b2c3d4...",
        "is_active": true,
        "created_at": "2025-11-18T12:00:00Z"
    }
}
```

---

#### DELETE /webhooks/:id

Delete a webhook.

**Request:**

```bash
curl -X DELETE \
     -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/webhooks/wh_abc123"
```

**Response:**

```json
{
    "data": {
        "deleted": true
    }
}
```

---

## Webhooks (Outbound)

DataFa.st sends webhooks to your configured URLs when events occur.

### Event Types

| Event | Description |
|-------|-------------|
| `visitor.created` | New visitor arrives |
| `goal.completed` | Goal conversion |
| `purchase.created` | Purchase attributed |

### Payload Format

```json
{
    "id": "evt_abc123",
    "type": "purchase.created",
    "created": 1699564800,
    "data": {
        "visitor_id": "v_xyz789",
        "session_id": "s_abc123",
        "revenue": 99.00,
        "currency": "USD",
        "source": "google",
        "website_id": "w_abc123"
    }
}
```

### Signature Verification

Webhooks include an `X-DataFast-Signature` header for verification:

```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
    const computed = crypto
        .createHmac('sha256', secret)
        .update(payload)
        .digest('hex');
    return crypto.timingSafeEqual(
        Buffer.from(computed),
        Buffer.from(signature)
    );
}
```

### Retry Policy

- **Retries:** 3 attempts
- **Backoff:** 1 min, 5 min, 30 min
- **Timeout:** 5 seconds per request
- **Failure:** Disabled after 100 consecutive failures

---

## Code Examples

### Python

```python
import requests

API_KEY = "df_live_your_token"
BASE_URL = "https://api.datafa.st/v1"

def get_metrics(start_date, end_date):
    response = requests.get(
        f"{BASE_URL}/metrics",
        headers={"Authorization": f"Bearer {API_KEY}"},
        params={
            "start_date": start_date,
            "end_date": end_date
        }
    )
    return response.json()

# Example usage
metrics = get_metrics("2025-11-01", "2025-11-30")
print(f"Revenue: ${metrics['data']['summary']['revenue']}")
```

### JavaScript/Node.js

```javascript
const axios = require('axios');

const API_KEY = 'df_live_your_token';
const BASE_URL = 'https://api.datafa.st/v1';

async function getMetrics(startDate, endDate) {
    const response = await axios.get(`${BASE_URL}/metrics`, {
        headers: { Authorization: `Bearer ${API_KEY}` },
        params: {
            start_date: startDate,
            end_date: endDate
        }
    });
    return response.data;
}

// Example usage
getMetrics('2025-11-01', '2025-11-30')
    .then(data => console.log(`Revenue: $${data.data.summary.revenue}`));
```

### cURL

```bash
# Get metrics
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/metrics?start_date=2025-11-01&end_date=2025-11-30"

# Get revenue by source
curl -H "Authorization: Bearer df_live_your_token" \
     "https://api.datafa.st/v1/revenue?start_date=2025-11-01&end_date=2025-11-30&group_by=source"
```

---

## OpenAPI Specification

Full OpenAPI 3.0 spec available at:
- **YAML:** https://api.datafa.st/openapi.yaml
- **JSON:** https://api.datafa.st/openapi.json
- **Interactive Docs:** https://docs.datafa.st/api

---

**Document Status:** ✅ Ready for Review
**Last Updated:** November 18, 2025
