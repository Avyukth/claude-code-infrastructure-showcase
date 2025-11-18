# DataFa.st Testing Strategy
## Quality Assurance & Testing Approach

**Version:** 1.0
**Date:** November 18, 2025
**Status:** Draft

---

## Table of Contents
1. [Testing Principles](#1-testing-principles)
2. [Testing Pyramid](#2-testing-pyramid)
3. [Unit Testing](#3-unit-testing)
4. [Integration Testing](#4-integration-testing)
5. [End-to-End Testing](#5-end-to-end-testing)
6. [Performance Testing](#6-performance-testing)
7. [Security Testing](#7-security-testing)
8. [Test Environments](#8-test-environments)
9. [CI/CD Integration](#9-cicd-integration)
10. [Quality Metrics](#10-quality-metrics)

---

## 1. Testing Principles

### Core Principles

1. **Test Early, Test Often:** Write tests during development, not after
2. **Automate Everything:** Manual testing only for exploratory/usability
3. **Fast Feedback:** Tests must run fast (<5 min for unit, <15 min for full suite)
4. **Realistic Data:** Use production-like data in tests
5. **Independent Tests:** Each test can run in isolation
6. **Descriptive Failures:** Clear error messages for debugging

### Quality Gates

All code must pass before merging:
- ✅ 80%+ code coverage (unit tests)
- ✅ All integration tests pass
- ✅ E2E smoke tests pass
- ✅ No critical security vulnerabilities
- ✅ Performance benchmarks within thresholds

---

## 2. Testing Pyramid

```
                 /\
                /  \
               / E2E \        10% - Critical user flows
              /______\
             /        \
            /Integration\     20% - API & service integration
           /__________\
          /            \
         /    Unit      \     70% - Component & function logic
        /________________\
```

### Test Distribution

| Level | Coverage | Run Time | Frequency |
|-------|----------|----------|-----------|
| **Unit** | 70% | <2 min | Every commit |
| **Integration** | 20% | <5 min | Every PR |
| **E2E** | 10% | <10 min | Pre-deploy |

---

## 3. Unit Testing

### Tools
- **Runner:** Vitest (fast, ESM-native)
- **Assertions:** Vitest built-in + Testing Library
- **Mocking:** Vitest mocks, MSW (Mock Service Worker)
- **Coverage:** c8 (V8 coverage)

### What to Unit Test

**Frontend (Next.js):**
- React components (rendering, props, state)
- Custom hooks (logic, side effects)
- Utility functions (formatters, validators)
- Zustand/React Query stores

**Backend (API):**
- Validation schemas (Zod)
- Business logic functions
- Data transformations
- Error handling

**Tracking Script:**
- Event batching logic
- Fingerprinting accuracy
- Offline queue behavior
- UTM parsing

### Example: Unit Test (Frontend)

```typescript
// MetricCard.test.tsx
import { render, screen } from '@testing-library/react';
import { MetricCard } from './MetricCard';

describe('MetricCard', () => {
    it('renders metric value with trend', () => {
        render(
            <MetricCard
                title="Revenue"
                value={2500}
                previousValue={2000}
                format="currency"
            />
        );

        expect(screen.getByText('Revenue')).toBeInTheDocument();
        expect(screen.getByText('$2,500')).toBeInTheDocument();
        expect(screen.getByText('+25%')).toBeInTheDocument();
        expect(screen.getByText('↑')).toHaveClass('text-green-500');
    });

    it('shows negative trend in red', () => {
        render(
            <MetricCard
                title="Visitors"
                value={800}
                previousValue={1000}
                format="number"
            />
        );

        expect(screen.getByText('-20%')).toBeInTheDocument();
        expect(screen.getByText('↓')).toHaveClass('text-red-500');
    });
});
```

### Example: Unit Test (Backend)

```typescript
// attribution.test.ts
import { describe, it, expect } from 'vitest';
import { attributeRevenue } from './attribution';

describe('attributeRevenue', () => {
    it('attributes to most recent session within window', () => {
        const sessions = [
            { timestamp: '2025-11-01T10:00:00Z', utm_source: 'google' },
            { timestamp: '2025-11-15T14:00:00Z', utm_source: 'twitter' },
        ];
        const purchaseTime = '2025-11-15T14:30:00Z';

        const result = attributeRevenue(sessions, purchaseTime, 30);

        expect(result.source).toBe('twitter');
    });

    it('returns "direct" when no sessions in window', () => {
        const sessions = [
            { timestamp: '2025-10-01T10:00:00Z', utm_source: 'google' },
        ];
        const purchaseTime = '2025-11-15T14:30:00Z';

        const result = attributeRevenue(sessions, purchaseTime, 30);

        expect(result.source).toBe('direct');
    });
});
```

### Example: Unit Test (Tracking Script)

```typescript
// queue.test.ts
import { describe, it, expect, vi } from 'vitest';
import { EventQueue } from './queue';

describe('EventQueue', () => {
    it('flushes when max batch size reached', async () => {
        const sendFn = vi.fn();
        const queue = new EventQueue({ maxBatch: 10, sendFn });

        // Add 10 events
        for (let i = 0; i < 10; i++) {
            queue.add({ type: 'pageview', page: `/page-${i}` });
        }

        expect(sendFn).toHaveBeenCalledTimes(1);
        expect(sendFn).toHaveBeenCalledWith(expect.arrayContaining([
            expect.objectContaining({ type: 'pageview' })
        ]));
    });

    it('flushes after timeout', async () => {
        vi.useFakeTimers();
        const sendFn = vi.fn();
        const queue = new EventQueue({ timeout: 5000, sendFn });

        queue.add({ type: 'pageview', page: '/' });
        expect(sendFn).not.toHaveBeenCalled();

        vi.advanceTimersByTime(5000);
        expect(sendFn).toHaveBeenCalledTimes(1);
    });
});
```

---

## 4. Integration Testing

### Tools
- **API Testing:** Vitest + Supertest
- **Database:** Testcontainers (Docker)
- **HTTP Mocking:** MSW (Mock Service Worker)

### What to Integration Test

- API endpoints (request → response)
- Database queries (ClickHouse, PostgreSQL)
- Authentication flows (NextAuth)
- Payment integration (Stripe webhooks)
- External API calls (Stripe, email)

### Example: API Integration Test

```typescript
// metrics.integration.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import request from 'supertest';
import { app } from '../app';
import { seedTestData, cleanupTestData } from './helpers';

describe('GET /api/metrics', () => {
    beforeAll(async () => {
        await seedTestData({
            events: 1000,
            website_id: 'test-website-123'
        });
    });

    afterAll(async () => {
        await cleanupTestData();
    });

    it('returns aggregated metrics for date range', async () => {
        const response = await request(app)
            .get('/api/metrics')
            .set('Authorization', 'Bearer test-token')
            .query({
                start_date: '2025-11-01',
                end_date: '2025-11-30',
                website_id: 'test-website-123'
            });

        expect(response.status).toBe(200);
        expect(response.body.data.summary).toHaveProperty('visitors');
        expect(response.body.data.summary).toHaveProperty('revenue');
        expect(response.body.data.summary).toHaveProperty('rpv');
    });

    it('returns 401 for invalid token', async () => {
        const response = await request(app)
            .get('/api/metrics')
            .set('Authorization', 'Bearer invalid-token')
            .query({ start_date: '2025-11-01', end_date: '2025-11-30' });

        expect(response.status).toBe(401);
    });

    it('returns 429 when rate limited', async () => {
        // Exceed rate limit
        for (let i = 0; i < 105; i++) {
            await request(app)
                .get('/api/metrics')
                .set('Authorization', 'Bearer test-token')
                .query({ start_date: '2025-11-01', end_date: '2025-11-30' });
        }

        const response = await request(app)
            .get('/api/metrics')
            .set('Authorization', 'Bearer test-token')
            .query({ start_date: '2025-11-01', end_date: '2025-11-30' });

        expect(response.status).toBe(429);
    });
});
```

### Example: Database Integration Test

```typescript
// clickhouse.integration.test.ts
import { describe, it, expect, beforeAll } from 'vitest';
import { ClickHouseClient } from './clickhouse';

describe('ClickHouse Queries', () => {
    let client: ClickHouseClient;

    beforeAll(async () => {
        client = new ClickHouseClient(process.env.CLICKHOUSE_TEST_URL);
        await client.seedTestData(10000); // 10K events
    });

    it('aggregates metrics by source within 500ms', async () => {
        const start = Date.now();

        const result = await client.query(`
            SELECT
                utm_source,
                count() AS visitors,
                sum(revenue) AS total_revenue
            FROM events
            WHERE timestamp >= '2025-11-01'
            GROUP BY utm_source
        `);

        const duration = Date.now() - start;

        expect(duration).toBeLessThan(500);
        expect(result.length).toBeGreaterThan(0);
    });
});
```

### Example: Stripe Webhook Test

```typescript
// stripe-webhook.integration.test.ts
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import Stripe from 'stripe';
import { app } from '../app';

describe('Stripe Webhooks', () => {
    const stripe = new Stripe(process.env.STRIPE_TEST_KEY!);

    it('attributes purchase on checkout.session.completed', async () => {
        // Create test event payload
        const event = {
            type: 'checkout.session.completed',
            data: {
                object: {
                    id: 'cs_test_123',
                    customer_email: 'test@example.com',
                    amount_total: 9900,
                    currency: 'usd'
                }
            }
        };

        const signature = stripe.webhooks.generateTestHeaderString({
            payload: JSON.stringify(event),
            secret: process.env.STRIPE_WEBHOOK_SECRET!
        });

        const response = await request(app)
            .post('/api/webhooks/stripe')
            .set('Stripe-Signature', signature)
            .send(event);

        expect(response.status).toBe(200);

        // Verify attribution in database
        const purchase = await getPurchaseByEmail('test@example.com');
        expect(purchase).toBeDefined();
        expect(purchase.revenue).toBe(99.00);
    });
});
```

---

## 5. End-to-End Testing

### Tools
- **Framework:** Playwright
- **Browser:** Chromium, Firefox, WebKit
- **Visual Regression:** Playwright Screenshots
- **Test Data:** Factory functions + API seeding

### What to E2E Test

**Critical User Flows:**
1. Sign up → Add website → Install script → See data
2. Connect Stripe → Make purchase → See attribution
3. Create goal → Trigger goal → View conversions
4. View dashboard → Filter → Export CSV
5. Upgrade plan → Billing portal → Confirm

### Example: E2E Test (Onboarding)

```typescript
// onboarding.e2e.test.ts
import { test, expect } from '@playwright/test';

test.describe('Onboarding Flow', () => {
    test('user can sign up and add first website', async ({ page }) => {
        // Go to homepage
        await page.goto('/');

        // Click CTA
        await page.click('text=Add my website');

        // Sign up
        await page.fill('input[name="email"]', 'test@example.com');
        await page.fill('input[name="password"]', 'SecurePass123');
        await page.click('button[type="submit"]');

        // Verify redirect to dashboard
        await expect(page).toHaveURL('/dashboard');

        // Add website
        await page.fill('input[name="domain"]', 'example.com');
        await page.click('text=Add Website');

        // Verify script generated
        await expect(page.locator('code')).toContainText('data-website-id');

        // Copy script
        await page.click('text=Copy Script');

        // Verify success message
        await expect(page.locator('.toast')).toContainText('Copied!');
    });

    test('user can connect Stripe', async ({ page }) => {
        // Login
        await page.goto('/login');
        await page.fill('input[name="email"]', 'test@example.com');
        await page.fill('input[name="password"]', 'SecurePass123');
        await page.click('button[type="submit"]');

        // Navigate to integrations
        await page.click('text=Settings');
        await page.click('text=Integrations');

        // Connect Stripe (mocked OAuth)
        await page.click('text=Connect Stripe');

        // Verify connected state
        await expect(page.locator('.stripe-status')).toContainText('Connected');
    });
});
```

### Example: E2E Test (Dashboard)

```typescript
// dashboard.e2e.test.ts
import { test, expect } from '@playwright/test';

test.describe('Dashboard', () => {
    test.beforeEach(async ({ page }) => {
        // Seed test data via API
        await page.request.post('/api/test/seed', {
            data: { events: 1000, purchases: 50 }
        });

        // Login
        await page.goto('/login');
        await page.fill('input[name="email"]', 'test@example.com');
        await page.fill('input[name="password"]', 'SecurePass123');
        await page.click('button[type="submit"]');
    });

    test('displays metrics correctly', async ({ page }) => {
        await page.goto('/dashboard');

        // Verify metric cards
        await expect(page.locator('[data-testid="visitors"]')).toBeVisible();
        await expect(page.locator('[data-testid="revenue"]')).toBeVisible();
        await expect(page.locator('[data-testid="rpv"]')).toBeVisible();
    });

    test('filters by date range', async ({ page }) => {
        await page.goto('/dashboard');

        // Select custom date range
        await page.click('text=Last 7 days');
        await page.click('text=Last 30 days');

        // Verify data updates
        await expect(page.locator('[data-testid="chart"]')).toBeVisible();
    });

    test('exports to CSV', async ({ page }) => {
        await page.goto('/dashboard');

        // Click export
        const [download] = await Promise.all([
            page.waitForEvent('download'),
            page.click('text=Export CSV')
        ]);

        // Verify file
        expect(download.suggestedFilename()).toMatch(/datafast-export-.*\.csv/);
    });
});
```

---

## 6. Performance Testing

### Tools
- **Load Testing:** k6
- **Benchmarking:** Apache Bench
- **Profiling:** Chrome DevTools, Lighthouse
- **APM:** Sentry Performance

### Performance Targets

| Metric | Target | Critical |
|--------|--------|----------|
| **Script Load** | <50ms | <100ms |
| **API Response (p95)** | <500ms | <1000ms |
| **Dashboard LCP** | <2s | <3s |
| **Event Ingestion** | 10K/sec | 5K/sec |

### Example: k6 Load Test

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '30s', target: 100 },  // Ramp up
        { duration: '1m', target: 100 },   // Steady state
        { duration: '30s', target: 0 },    // Ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'], // 95% under 500ms
        http_req_failed: ['rate<0.01'],   // <1% errors
    },
};

export default function () {
    // Test metrics endpoint
    const response = http.get(
        'https://api.datafa.st/v1/metrics?start_date=2025-11-01&end_date=2025-11-30',
        {
            headers: {
                Authorization: `Bearer ${__ENV.API_TOKEN}`,
            },
        }
    );

    check(response, {
        'status is 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
    });

    sleep(1);
}
```

### Example: Event Ingestion Test

```javascript
// ingestion-test.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
    vus: 100, // Virtual users
    duration: '10m',
    thresholds: {
        http_req_duration: ['p(95)<100'],
        http_reqs: ['rate>10000'], // 10K req/sec
    },
};

export default function () {
    const payload = JSON.stringify({
        website_id: 'test-123',
        event_type: 'pageview',
        visitor_id: `v_${__VU}_${__ITER}`,
        page_url: 'https://example.com/',
        timestamp: Date.now(),
    });

    const response = http.post(
        'https://api.datafa.st/track',
        payload,
        { headers: { 'Content-Type': 'application/json' } }
    );

    check(response, {
        'status is 202': (r) => r.status === 202,
    });
}
```

---

## 7. Security Testing

### Tools
- **SAST:** ESLint Security Plugin, Semgrep
- **Dependency Audit:** npm audit, Dependabot
- **Secrets Detection:** git-secrets, truffleHog
- **Penetration Testing:** OWASP ZAP (quarterly)

### Security Checklist

**Authentication:**
- [ ] Rate limiting on login (5 attempts/15 min)
- [ ] bcrypt with 12+ rounds
- [ ] Session expiry (30 days)
- [ ] CSRF protection (SameSite cookies)

**Input Validation:**
- [ ] All inputs validated (Zod schemas)
- [ ] SQL injection prevention (parameterized queries)
- [ ] XSS prevention (React auto-escaping)

**API Security:**
- [ ] Authentication required for all endpoints
- [ ] Rate limiting per user
- [ ] CORS configured correctly
- [ ] No sensitive data in logs

**Data Protection:**
- [ ] TLS 1.3 in transit
- [ ] Encryption at rest
- [ ] IP anonymization
- [ ] GDPR deletion support

### Example: Security Test

```typescript
// security.test.ts
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { app } from '../app';

describe('Security', () => {
    it('blocks SQL injection attempts', async () => {
        const response = await request(app)
            .get('/api/metrics')
            .set('Authorization', 'Bearer valid-token')
            .query({
                start_date: "2025-11-01'; DROP TABLE events; --",
                end_date: '2025-11-30'
            });

        expect(response.status).toBe(400); // Validation error
    });

    it('prevents XSS in user input', async () => {
        const response = await request(app)
            .post('/api/notes')
            .set('Authorization', 'Bearer valid-token')
            .send({
                content: '<script>alert("xss")</script>'
            });

        expect(response.body.data.content).not.toContain('<script>');
    });

    it('rate limits login attempts', async () => {
        // Attempt 6 logins
        for (let i = 0; i < 6; i++) {
            await request(app)
                .post('/api/auth/login')
                .send({ email: 'test@example.com', password: 'wrong' });
        }

        const response = await request(app)
            .post('/api/auth/login')
            .send({ email: 'test@example.com', password: 'wrong' });

        expect(response.status).toBe(429);
    });
});
```

---

## 8. Test Environments

### Local Development
```bash
# Start local databases
docker-compose -f docker-compose.test.yml up -d

# Run tests
npm test              # Unit tests
npm run test:int      # Integration tests
npm run test:e2e      # E2E tests
```

### CI Environment (GitHub Actions)
- Docker containers for databases
- Parallel test execution
- Artifact storage for screenshots/videos

### Staging Environment
- Mirror of production
- Synthetic data (anonymized)
- Used for E2E smoke tests before deploy

### Production
- No tests run in production
- Monitoring and alerting only
- Canary releases (10% traffic)

---

## 9. CI/CD Integration

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:coverage

      - uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info

  integration-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
      clickhouse:
        image: clickhouse/clickhouse-server:latest
      redis:
        image: redis:7

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm run test:int
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test
          CLICKHOUSE_URL: http://localhost:8123
          REDIS_URL: redis://localhost:6379

  e2e-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npx playwright install

      - run: npm run build
      - run: npm run test:e2e

      - uses: actions/upload-artifact@v3
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
      - uses: github/codeql-action/analyze@v2
```

---

## 10. Quality Metrics

### Coverage Requirements

| Area | Minimum | Target |
|------|---------|--------|
| **Unit Tests** | 80% | 90% |
| **Integration Tests** | 70% | 80% |
| **E2E Coverage** | Critical flows | All user flows |

### Quality Dashboard

Track in CI/monitoring:
- Test pass rate (target: 100%)
- Code coverage trend
- Test execution time
- Flaky test rate (<1%)
- Bug escape rate (bugs found in production)

### Test Maintenance

**Weekly:**
- Review flaky tests
- Update snapshots
- Check for outdated mocks

**Monthly:**
- Review coverage gaps
- Update E2E for new features
- Performance benchmark review

**Quarterly:**
- Security penetration testing
- Load testing review
- Test suite refactoring

---

**Document Status:** ✅ Ready for Review
**Last Updated:** November 18, 2025
