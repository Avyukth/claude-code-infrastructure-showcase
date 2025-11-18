# DataFa.st Product Documentation
## Revenue-First Web Analytics Platform

**Project Status:** Phase 0 (Validation)
**Last Updated:** November 18, 2025
**Version:** 1.0

---

## Quick Links

| Document | Description | Status |
|----------|-------------|--------|
| [PRD](prd/DATAFAST-PRD.md) | Complete Product Requirements Document | ✅ Ready |
| [User Stories](user-stories/USER-STORIES.md) | 43 detailed user stories with acceptance criteria | ✅ Ready |
| [ADRs](adr/README.md) | 6 Architecture Decision Records | ✅ Ready |
| [System Architecture](architecture/SYSTEM-ARCHITECTURE.md) | Technical architecture & C4 diagrams | ✅ Ready |
| [User Flows](architecture/USER-FLOWS.md) | 7 comprehensive user journey diagrams | ✅ Ready |
| [Phase 0 Plan](validation/PHASE-0-PLAN.md) | 2-week validation sprint plan | ✅ Ready |
| [Data Dictionary](technical/DATA-DICTIONARY.md) | Database schemas & data models | ✅ Ready |
| [API Reference](technical/API-REFERENCE.md) | Complete API documentation | ✅ Ready |
| [Testing Strategy](technical/TESTING-STRATEGY.md) | QA and testing approach | ✅ Ready |

---

## Project Overview

### What is DataFa.st?

DataFa.st is a **revenue-first web analytics platform** designed for indie makers and small businesses. Unlike traditional analytics tools that focus on vanity metrics (pageviews, bounce rates), DataFa.st connects website traffic directly to revenue through native integrations with payment processors.

### Key Value Propositions

1. **Fast Setup:** 4-5 minutes from signup to first data
2. **Revenue Attribution:** See which traffic sources drive actual revenue
3. **Simplicity:** Focus on RPV (Revenue Per Visitor), not 100 metrics
4. **Privacy-First:** GDPR compliant, no cookies, lightweight 4KB script
5. **Real-Time:** Live visitor intelligence with globe visualization

### Target Users

- **Primary:** Solo founders and indie makers (bootstrapped SaaS)
- **Secondary:** Small e-commerce businesses (Shopify stores)
- **Tertiary:** Growth marketers at small teams (5-10 people)

### Business Goals

| Metric | Current | Target (6 Months) |
|--------|---------|-------------------|
| Active Users | 9,619 | 15,000+ |
| MRR | ~$6K | $50K |
| Trial → Paid | TBD | 18%+ |
| Monthly Retention | TBD | 85%+ |

---

## Documentation Structure

```
datafast/
├── README.md                    # This file - Project overview & navigation
├── prd/
│   └── DATAFAST-PRD.md         # Complete PRD (25+ pages)
├── user-stories/
│   └── USER-STORIES.md         # 43 user stories with acceptance criteria
├── adr/
│   ├── README.md               # ADR index
│   ├── ADR-001-time-series-database.md
│   ├── ADR-002-tracking-script-architecture.md
│   ├── ADR-003-frontend-framework.md
│   ├── ADR-004-revenue-attribution-model.md
│   ├── ADR-005-authentication-strategy.md
│   └── ADR-006-api-design.md
├── architecture/
│   ├── SYSTEM-ARCHITECTURE.md  # C4 diagrams & technical architecture
│   └── USER-FLOWS.md           # Mermaid diagrams for user journeys
├── technical/
│   ├── DATA-DICTIONARY.md      # Database schemas & data models
│   ├── API-REFERENCE.md        # Complete API documentation
│   └── TESTING-STRATEGY.md     # Testing approach & strategy
└── validation/
    └── PHASE-0-PLAN.md         # 2-week validation sprint plan
```

---

## Technology Stack

### Frontend
- **Framework:** Next.js 14+ (App Router)
- **UI Components:** shadcn/ui + Tailwind CSS
- **Charts:** Recharts
- **State:** TanStack Query (React Query)
- **Auth:** NextAuth.js

### Backend
- **Runtime:** Node.js 20+ / Bun
- **Framework:** Hono (lightweight) or Express
- **Validation:** Zod
- **Queue:** BullMQ (Redis-backed)

### Databases
- **Analytics Events:** ClickHouse (time-series optimized)
- **User/Config Data:** PostgreSQL (Supabase/RDS)
- **Cache/Sessions:** Redis (Upstash)

### Infrastructure
- **Frontend Hosting:** Vercel
- **Backend Hosting:** Fly.io or Railway
- **CDN:** Cloudflare (tracking script)
- **Monitoring:** Sentry, Axiom, Prometheus

### Third-Party Integrations
- **Payments:** Stripe, Lemon Squeezy, Polar, Shopify
- **Email:** Resend
- **Auth:** Google OAuth, X/Twitter OAuth

---

## Development Phases

### Phase 0: Validation (Current - 2 Weeks)
**Goal:** Validate technical feasibility and user demand

- [ ] ClickHouse performance benchmarks
- [ ] Tracking script prototype (<5KB)
- [ ] Stripe attribution accuracy tests
- [ ] 20 user interviews
- [ ] Landing page test (100+ signups)

**Success Criteria:** 80%+ validation tests pass → Proceed to Phase 1

### Phase 1: MVP Launch (Weeks 3-10)
**Goal:** Launch publicly with core revenue attribution

**Features:**
- ✅ Analytics tracking core
- ✅ Revenue attribution (Stripe)
- ✅ Dashboard & metrics
- ✅ Quick setup flow
- ✅ Event limits & billing
- ✅ Privacy & compliance

**Target:** 500 signups, 18%+ trial conversion

### Phase 2: Growth Features (Weeks 11-18)
**Goal:** Add differentiating features

**Features:**
- ✅ Goals & funnels
- ✅ Real-time visitor intelligence
- ✅ X (Twitter) attribution
- ✅ Multi-site management
- ✅ Additional payment integrations (LS, Polar, Shopify)

**Target:** 2,000+ users, NPS 50+

### Phase 3: API & Advanced (Weeks 19-24)
**Goal:** Enable power users and custom integrations

**Features:**
- ✅ Custom REST API
- ✅ Notes & annotations
- ✅ Team collaboration
- ✅ Advanced filtering

**Target:** 10%+ API adoption, Enterprise plan revenue

---

## Key Documents by Role

### For Product Managers
1. [PRD](prd/DATAFAST-PRD.md) - Complete requirements and success metrics
2. [User Stories](user-stories/USER-STORIES.md) - Backlog with acceptance criteria
3. [Phase 0 Plan](validation/PHASE-0-PLAN.md) - Validation sprint details

### For Engineers
1. [System Architecture](architecture/SYSTEM-ARCHITECTURE.md) - Technical design
2. [ADRs](adr/README.md) - Technology decisions and rationale
3. [Data Dictionary](technical/DATA-DICTIONARY.md) - Database schemas
4. [API Reference](technical/API-REFERENCE.md) - API endpoints
5. [Testing Strategy](technical/TESTING-STRATEGY.md) - QA approach

### For Designers
1. [PRD Section 5](prd/DATAFAST-PRD.md#5-design--ux) - Design principles
2. [User Flows](architecture/USER-FLOWS.md) - User journey diagrams

### For Stakeholders
1. [PRD Executive Summary](prd/DATAFAST-PRD.md#1-executive-summary) - Business overview
2. [Phase 0 Plan](validation/PHASE-0-PLAN.md) - Validation approach
3. [PRD Section 8](prd/DATAFAST-PRD.md#8-release-plan--roadmap) - Roadmap

---

## Quick Reference

### Pricing
- **Starter:** $99/month - 10K events, 1 site
- **Growth:** $199/month - 30K events, 30 sites
- **Enterprise:** $499/month - Custom limits, priority support

### Core Metrics
- **RPV (Revenue Per Visitor):** Total Revenue / Unique Visitors
- **Attribution Window:** 30 days (default, configurable 1-90 days)
- **Event:** User action (pageview, goal, purchase)
- **Session:** Group of events within 30 minutes

### Performance Targets
- **Script Size:** <5KB compressed
- **Page Load Impact:** <50ms
- **Dashboard Load:** <2s (p95)
- **API Response:** <500ms (p95)
- **Uptime:** 99.9% SLA

---

## Getting Started (Development)

### Prerequisites
- Node.js 20+ or Bun
- Docker (for local databases)
- Vercel CLI (for deployment)

### Local Setup
```bash
# Clone the repository
git clone https://github.com/your-org/datafast.git
cd datafast

# Install dependencies
npm install

# Start local databases (Docker)
docker-compose up -d

# Set up environment variables
cp .env.example .env.local

# Run development server
npm run dev
```

### Key Environment Variables
```bash
# Database
CLICKHOUSE_URL=http://localhost:8123
DATABASE_URL=postgresql://localhost:5432/datafast
REDIS_URL=redis://localhost:6379

# Auth
NEXTAUTH_SECRET=your-secret
GOOGLE_CLIENT_ID=xxx
GOOGLE_CLIENT_SECRET=xxx
TWITTER_CLIENT_ID=xxx
TWITTER_CLIENT_SECRET=xxx

# Stripe
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# Other
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

---

## Contributing

### Updating Documentation

1. All documents are in Markdown format
2. Use Mermaid for diagrams
3. Follow existing formatting conventions
4. Update the README index when adding new documents
5. Commit with descriptive messages

### Document Naming Conventions
- Use UPPERCASE for main documents (PRD, README)
- Use kebab-case for ADRs (ADR-001-title.md)
- Include version and date in document header

### Review Process
1. Create PR with documentation changes
2. Request review from relevant stakeholders
3. Update version and date after approval
4. Merge to main branch

---

## Links & Resources

### External Documentation
- [ClickHouse Docs](https://clickhouse.com/docs)
- [Next.js Docs](https://nextjs.org/docs)
- [Stripe API](https://stripe.com/docs/api)
- [NextAuth.js](https://next-auth.js.org/)

### Competitor Research
- [Plausible](https://plausible.io/) - Privacy-first analytics
- [Cometly](https://www.cometly.com/) - Marketing attribution
- [Google Analytics](https://analytics.google.com/) - Free, complex

### Design Resources
- [shadcn/ui](https://ui.shadcn.com/) - Component library
- [Tailwind CSS](https://tailwindcss.com/) - Styling
- [Recharts](https://recharts.org/) - Charts

---

## Contact & Support

**Product Lead:** TBD
**Engineering Lead:** TBD
**Design Lead:** TBD

For questions about this documentation, please contact the product team.

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Nov 18, 2025 | Product Team | Initial documentation suite |

---

**Document Status:** ✅ Ready for Phase 0 Validation
**Next Milestone:** Phase 0 kickoff (Week 1)
