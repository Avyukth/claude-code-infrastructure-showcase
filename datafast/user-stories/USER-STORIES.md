# DataFa.st User Stories
## Detailed User Stories with Acceptance Criteria

**Version:** 1.0
**Date:** November 18, 2025
**Status:** Draft

---

## Table of Contents
1. [Onboarding & Setup](#1-onboarding--setup)
2. [Analytics Tracking](#2-analytics-tracking)
3. [Revenue Attribution](#3-revenue-attribution)
4. [Dashboard & Visualization](#4-dashboard--visualization)
5. [Goals & Funnels](#5-goals--funnels)
6. [Real-Time Intelligence](#6-real-time-intelligence)
7. [Integrations](#7-integrations)
8. [API & Automation](#8-api--automation)
9. [Account & Billing](#9-account--billing)
10. [Privacy & Compliance](#10-privacy--compliance)

---

## 1. Onboarding & Setup

### US-1.1: Quick Sign-Up
**As a** potential user
**I want to** sign up in under 1 minute
**So that I can** start tracking my website immediately without friction

**Priority:** P0 (Must-Have)
**Epic:** Onboarding
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] User can sign up with email and password
- [ ] Password must be 8+ characters with at least 1 number
- [ ] Email verification sent within 30 seconds
- [ ] OAuth options available: Google, X (Twitter)
- [ ] OAuth completes without email verification step
- [ ] No credit card required for trial
- [ ] Sign-up form loads in <1 second
- [ ] Error messages are clear (e.g., "Email already registered")
- [ ] Success state redirects to "Add Website" step
- [ ] CAPTCHA only appears after 3 failed attempts (prevent spam)

**Dependencies:** None

**Test Scenarios:**
1. Valid email + strong password → Account created, verification email sent
2. Weak password → Error: "Password must be 8+ characters with 1 number"
3. Duplicate email → Error: "Email already registered. Sign in instead?"
4. OAuth Google → Redirect to Google, return with account created
5. Network error during signup → Retry button appears

---

### US-1.2: Add First Website
**As a** new user
**I want to** add my website domain and get a tracking ID
**So that I can** install the tracking script

**Priority:** P0 (Must-Have)
**Epic:** Onboarding
**Estimate:** 2 story points

**Acceptance Criteria:**
- [ ] User enters domain (e.g., example.com or https://example.com)
- [ ] System normalizes domain (strips protocol, trailing slash)
- [ ] Generates unique website_id (UUID)
- [ ] Validates domain format (regex check)
- [ ] Prevents duplicate domains for same user
- [ ] Allows same domain for different users (multi-tenancy)
- [ ] Shows preview of tracking script with website_id pre-filled
- [ ] "Test Mode" toggle available (sandbox data)
- [ ] Save button disabled until valid domain entered
- [ ] Success message: "Website added! Now install the script."

**Dependencies:** US-1.1 (user must be signed up)

**Test Scenarios:**
1. Enter "example.com" → Normalized to "example.com", script generated
2. Enter "https://example.com/" → Normalized to "example.com"
3. Enter invalid domain "not a domain" → Error: "Invalid domain format"
4. Add duplicate domain → Error: "You already added this domain"
5. Click "Test Mode" → Script changes to test endpoint

---

### US-1.3: Install Tracking Script
**As a** user (technical or non-technical)
**I want to** easily install the tracking script on my website
**So that I can** start collecting analytics data

**Priority:** P0 (Must-Have)
**Epic:** Onboarding
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] Script is <5KB compressed
- [ ] One-click copy button for script code
- [ ] Installation instructions for:
  - HTML (paste in `<head>` tag)
  - Next.js (add to `_app.js` or `layout.tsx`)
  - WordPress (paste in footer)
  - Google Tag Manager (custom HTML tag)
  - Webflow (embed code in site settings)
- [ ] Video tutorial embedded (30 seconds)
- [ ] "Verify Installation" button tests script
- [ ] Verification shows success/error state
- [ ] Debug mode: Console logs visible on user's site
- [ ] Script works on single-page apps (auto-tracks route changes)
- [ ] Script loads asynchronously (non-blocking)
- [ ] Fallback if CDN fails (load from primary domain)

**Dependencies:** US-1.2 (website must be added)

**Test Scenarios:**
1. Copy script → Paste in HTML → Visit site → Dashboard shows pageview
2. Install via GTM → Trigger on all pages → Pageview tracked
3. Install on Next.js SPA → Navigate between pages → Multiple pageviews tracked
4. CDN down → Script loads from fallback URL
5. Verify before installation → Error: "No events detected"
6. Verify after installation → Success: "Tracking verified!"

---

### US-1.4: Connect Payment Processor
**As a** user with an online business
**I want to** connect my Stripe account
**So that I can** see revenue attribution

**Priority:** P0 (Must-Have – MVP focuses on Stripe)
**Epic:** Onboarding
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] "Connect Stripe" button initiates OAuth flow
- [ ] OAuth requests restricted scopes: `read_customers`, `read_charges`
- [ ] Alternative: Manual API key input (restricted key)
- [ ] Validate API key with test call to Stripe
- [ ] Show connected account details (account name, currency)
- [ ] Option to disconnect and reconnect
- [ ] Webhook endpoint created: `/api/integrations/stripe/webhook`
- [ ] Webhook events subscribed: `checkout.session.completed`, `invoice.paid`, `charge.succeeded`
- [ ] Webhook signature verification (security)
- [ ] Test mode toggle (use Stripe test keys)
- [ ] Error handling: Invalid key, insufficient permissions, network errors

**Dependencies:** US-1.2 (website must be added)

**Test Scenarios:**
1. Click "Connect Stripe" → OAuth redirect → Return with account connected
2. Enter test API key → Validation passes → Account connected
3. Enter invalid key → Error: "Invalid Stripe API key"
4. Connect with insufficient scopes → Error: "Grant read_customers permission"
5. Receive webhook → Payment attributed to visitor session
6. Webhook signature invalid → Logged, user alerted

---

### US-1.5: Onboarding Checklist
**As a** new user
**I want to** see my progress through setup steps
**So that I can** quickly reach the "first insight" milestone

**Priority:** P1 (Should-Have)
**Epic:** Onboarding
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] Checklist visible in dashboard sidebar
- [ ] Steps:
  1. ✅ Create account
  2. ✅ Add website
  3. ⬜ Install script
  4. ⬜ Connect payment processor
  5. ⬜ View first data
- [ ] Progress bar shows completion percentage
- [ ] Clicking step navigates to relevant page
- [ ] Completed steps have checkmark icon
- [ ] Current step highlighted
- [ ] Checklist dismissible after completion
- [ ] Welcome email includes checklist
- [ ] In-app tooltips guide through each step

**Dependencies:** US-1.1, US-1.2, US-1.3, US-1.4

**Test Scenarios:**
1. New user logs in → Checklist shows 1/5 complete
2. Install script → Checklist updates to 3/5
3. Complete all steps → Checklist shows "🎉 Setup complete!"
4. Dismiss checklist → Hidden from sidebar

---

## 2. Analytics Tracking

### US-2.1: Automatic Pageview Tracking
**As a** website owner
**I want** pageviews tracked automatically
**So that I can** see visitor traffic without manual setup

**Priority:** P0 (Must-Have)
**Epic:** Core Analytics
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] Script tracks pageviews on load
- [ ] Captures: URL, referrer, timestamp, user agent
- [ ] Generates visitor_id (fingerprint or cookie)
- [ ] Groups events into sessions (30-minute timeout)
- [ ] Works on single-page apps (history API, hash routing)
- [ ] Respects Do Not Track (DNT) browser setting
- [ ] Does not track localhost or IP addresses (dev mode)
- [ ] Sends events to `/track` endpoint via POST
- [ ] Batches events (max 10 events, flush every 5 seconds)
- [ ] Queues events if offline, sends when online
- [ ] Event payload <1KB per pageview

**Dependencies:** US-1.3 (script installed)

**Test Scenarios:**
1. User visits page → Pageview event sent within 1 second
2. User navigates SPA → Each route change tracked
3. User enables DNT → Events not sent
4. User offline → Events queued, sent when online
5. 10 rapid pageviews → Batched into 1 request

---

### US-2.2: UTM Parameter Tracking
**As a** marketer
**I want** UTM parameters captured automatically
**So that I can** attribute traffic to campaigns

**Priority:** P0 (Must-Have)
**Epic:** Core Analytics
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] Extracts UTM params from URL: source, medium, campaign, content, term
- [ ] Persists UTM data for entire session (stored in localStorage)
- [ ] Associates UTM with all session events (pageviews, goals, purchases)
- [ ] Handles missing UTMs gracefully (null values)
- [ ] Dashboard shows UTM breakdown in referrer table
- [ ] Supports custom parameters (e.g., `ref=twitter`)
- [ ] Case-insensitive parameter matching
- [ ] URL decoding for special characters

**Dependencies:** US-2.1 (pageview tracking)

**Test Scenarios:**
1. Visit `example.com?utm_source=google&utm_medium=cpc` → Source: Google, Medium: CPC
2. Navigate to new page without UTM → Original UTM persists
3. New session after 30 min → UTM cleared
4. Visit with `ref=twitter` → Custom param tracked

---

### US-2.3: Device & Browser Detection
**As a** user
**I want to** see visitor breakdown by device and browser
**So that I can** optimize for my audience's platforms

**Priority:** P1 (Should-Have)
**Epic:** Core Analytics
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] Detects device type: mobile, tablet, desktop
- [ ] Identifies browser: Chrome, Firefox, Safari, Edge, Other
- [ ] Captures OS: Windows, macOS, Linux, iOS, Android
- [ ] Uses User-Agent header (server-side parsing)
- [ ] Fallback: Client Hints API for modern browsers
- [ ] Dashboard shows device/browser breakdown (pie chart)
- [ ] Filterable by device type (e.g., "Show mobile only")
- [ ] Handles unknown/obscure browsers gracefully

**Dependencies:** US-2.1 (pageview tracking)

**Test Scenarios:**
1. Visit from iPhone Safari → Device: Mobile, Browser: Safari, OS: iOS
2. Visit from Windows Chrome → Device: Desktop, Browser: Chrome, OS: Windows
3. Visit from obscure browser → Browser: "Other"

---

## 3. Revenue Attribution

### US-3.1: Stripe Purchase Attribution
**As a** SaaS founder
**I want** Stripe purchases linked to visitor sessions
**So that I can** see which traffic sources drive revenue

**Priority:** P0 (Must-Have)
**Epic:** Revenue Attribution
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] Webhook receives `checkout.session.completed` event
- [ ] Extracts: customer_email, amount, currency, timestamp
- [ ] Matches email to visitor_id (within 30-day attribution window)
- [ ] If multiple sessions, attributes to most recent
- [ ] Stores revenue in events table (event_type: 'purchase')
- [ ] Dashboard shows: Total revenue, Revenue by source, RPV
- [ ] Handles: Multiple purchases from same visitor, subscription vs. one-time
- [ ] Currency conversion (if user's currency ≠ Stripe's)
- [ ] Revenue appears in dashboard within 1 minute of webhook

**Dependencies:** US-1.4 (Stripe connected), US-2.1 (tracking)

**Test Scenarios:**
1. Visitor from Google → Purchases via Stripe → Revenue attributed to Google
2. Visitor from X → 2 purchases → Both attributed
3. Purchase outside attribution window → Not attributed (marked as "Direct")
4. Visitor with no email match → Fallback to IP/fingerprint matching

---

### US-3.2: Revenue Per Visitor (RPV) Calculation
**As a** user
**I want to** see RPV by traffic source
**So that I can** prioritize high-value channels

**Priority:** P0 (Must-Have)
**Epic:** Revenue Attribution
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] RPV = Total Revenue / Unique Visitors
- [ ] Calculated per traffic source (organic, social, paid, etc.)
- [ ] Dashboard displays RPV prominently (top metric card)
- [ ] Sortable table: Source | Visitors | Revenue | RPV
- [ ] Time filter applies to RPV (e.g., "Last 30 days")
- [ ] Handles edge case: 0 visitors → RPV = $0
- [ ] Currency formatting (e.g., $12.34, €10.50)
- [ ] Trend indicator (↑↓ vs. previous period)

**Dependencies:** US-3.1 (revenue attribution)

**Test Scenarios:**
1. Google: 100 visitors, $500 revenue → RPV: $5.00
2. X: 50 visitors, $0 revenue → RPV: $0.00
3. Filter to last 7 days → RPV recalculates for that period

---

### US-3.3: Subscription Revenue Tracking
**As a** SaaS founder
**I want** subscription revenue tracked separately
**So that I can** calculate MRR and churn

**Priority:** P1 (Should-Have)
**Epic:** Revenue Attribution
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] Webhook: `invoice.paid` for recurring subscriptions
- [ ] Identifies: New subscription, renewal, upgrade, downgrade
- [ ] Calculates MRR: Sum of all active subscription values
- [ ] Tracks churn: Cancelled subscriptions / Active subscriptions
- [ ] Dashboard shows: MRR, MRR growth, churn rate
- [ ] Attributes initial subscription to traffic source
- [ ] Renewals don't re-attribute (credit to original source)
- [ ] Handles: Trial conversions, annual plans (divide by 12 for MRR)

**Dependencies:** US-3.1 (Stripe integration)

**Test Scenarios:**
1. New $50/mo subscription → MRR increases by $50
2. Upgrade from $50 to $100 → MRR increases by $50
3. Cancellation → MRR decreases, churn rate updates
4. Annual $600 plan → MRR increases by $50 ($600/12)

---

### US-3.4: Multi-Integration Support
**As a** user with multiple payment processors
**I want to** connect Lemon Squeezy, Polar, and Shopify
**So that I can** track all revenue in one place

**Priority:** P1 (Should-Have – Phase 2)
**Epic:** Revenue Attribution
**Estimate:** 13 story points (per integration)

**Acceptance Criteria:**
**Lemon Squeezy:**
- [ ] OAuth or API key integration
- [ ] Webhook: `order_created`
- [ ] Revenue attribution identical to Stripe

**Polar:**
- [ ] API key integration
- [ ] Webhook: `checkout.completed`
- [ ] Revenue attribution identical to Stripe

**Shopify:**
- [ ] Shopify app installation
- [ ] Scopes: `read_orders`, `read_customers`
- [ ] Fetch orders via Shopify API (every 5 minutes)
- [ ] Match orders to sessions via customer email
- [ ] Handle: Multiple currencies, multi-location stores

**Dashboard:**
- [ ] Filter by payment processor (e.g., "Show Stripe only")
- [ ] Aggregated view: All revenue combined
- [ ] Processor breakdown (pie chart)

**Dependencies:** US-3.1 (Stripe pattern established)

**Test Scenarios:**
1. Connect Lemon Squeezy → Orders attributed
2. Connect Shopify → Orders fetched every 5 min, attributed
3. Revenue from multiple processors → Aggregated correctly

---

## 4. Dashboard & Visualization

### US-4.1: Overview Dashboard
**As a** user
**I want to** see key metrics at a glance
**So that I can** quickly understand business performance

**Priority:** P0 (Must-Have)
**Epic:** Dashboard
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] Metric cards: Visitors, Revenue, RPV, Conversion Rate
- [ ] Time period filter: Today, 7d, 30d, 90d, Custom date range
- [ ] Each metric shows: Current value, trend (↑↓), % change
- [ ] Line chart: Visitors + Revenue over time (dual-axis)
- [ ] Referrer breakdown table: Source, Visitors, Revenue, RPV
- [ ] Device breakdown: Mobile vs. Desktop (pie chart)
- [ ] Top pages table: URL, Pageviews, Bounce rate
- [ ] Data refreshes automatically (every 60 seconds)
- [ ] Loading states for slow queries (skeleton UI)
- [ ] Empty state: "No data yet. Install script to start."

**Dependencies:** US-2.1 (tracking), US-3.1 (revenue)

**Test Scenarios:**
1. User with no data → Empty state shown
2. User with 7 days data → All metrics populated
3. Change filter to "Last 30 days" → Charts/tables update
4. Metric increases 20% → Trend shows ↑ 20%

---

### US-4.2: Data Export
**As a** user
**I want to** export data to CSV
**So that I can** analyze it in Excel or share with my team

**Priority:** P1 (Should-Have)
**Epic:** Dashboard
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] "Export CSV" button on dashboard
- [ ] Exports current filtered view (respects date range, filters)
- [ ] CSV includes: Date, Visitors, Revenue, RPV, Source, Device
- [ ] File named: `datafast-export-YYYY-MM-DD.csv`
- [ ] Download starts immediately (no email)
- [ ] Handles large datasets: Max 100K rows (pagination if needed)
- [ ] Encoding: UTF-8 (supports special characters)

**Dependencies:** US-4.1 (dashboard)

**Test Scenarios:**
1. Click "Export CSV" → File downloads with current data
2. Filter to "Last 7 days" → CSV contains 7 days only
3. 200K rows → CSV limited to 100K, message: "Showing first 100K rows"

---

### US-4.3: Dark Mode Toggle
**As a** user who works late
**I want to** switch to dark mode
**So that I can** reduce eye strain

**Priority:** P2 (Nice-to-Have)
**Epic:** Dashboard
**Estimate:** 2 story points

**Acceptance Criteria:**
- [ ] Toggle in header: ☀️ Light / 🌙 Dark
- [ ] Preference saved to localStorage
- [ ] All UI elements support both modes
- [ ] WCAG AA contrast ratios maintained
- [ ] Smooth transition animation (200ms)
- [ ] Defaults to system preference (media query: `prefers-color-scheme`)

**Dependencies:** US-4.1 (dashboard)

**Test Scenarios:**
1. Click dark mode toggle → UI switches, preference saved
2. Reload page → Dark mode persists
3. System preference dark → Default to dark mode

---

### US-4.4: Mobile Responsive Dashboard
**As a** user on mobile
**I want** the dashboard to work on my phone
**So that I can** check metrics on the go

**Priority:** P0 (Must-Have)
**Epic:** Dashboard
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] Dashboard works on screens 320px+ width
- [ ] Charts scale to fit screen (responsive SVG)
- [ ] Tables scrollable horizontally if needed
- [ ] Touch-friendly: Buttons 44x44px min
- [ ] Date picker optimized for mobile (native input type)
- [ ] No horizontal scrolling on main view
- [ ] Fast Tap (no 300ms delay)
- [ ] Tested on: iPhone Safari, Android Chrome

**Dependencies:** US-4.1 (dashboard)

**Test Scenarios:**
1. Open on iPhone 12 → All elements visible, no horizontal scroll
2. Tap date filter → Native date picker appears
3. Swipe table → Scrolls smoothly

---

## 5. Goals & Funnels

### US-5.1: Create Custom Goals
**As a** user
**I want to** track custom events like signups or downloads
**So that I can** measure conversions beyond revenue

**Priority:** P1 (Should-Have)
**Epic:** Goals & Funnels
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] Create up to 10 goals per website
- [ ] Goal setup: Name (e.g., "Signup"), event trigger method
- [ ] Trigger methods:
  - Data attribute: `<button data-fast-goal="signup">Sign Up</button>`
  - JavaScript API: `window.datafast.goal('signup', { value: 100 })`
- [ ] Optional: Monetary value per goal (e.g., lead value)
- [ ] Goals appear in dashboard: Name, Completions, Conversion rate
- [ ] Filter dashboard by goal (e.g., "Show traffic that completed signup")
- [ ] Edit/delete goals
- [ ] Goal history: Track changes to goal config

**Dependencies:** US-2.1 (tracking)

**Test Scenarios:**
1. Create goal "Signup" → Install data attribute → Click triggers goal
2. Fire goal via JS API → Dashboard shows completion
3. Create 11th goal → Error: "Max 10 goals per site"
4. Delete goal → Removed from dashboard, historical data retained

---

### US-5.2: Funnel Visualization
**As a** marketer
**I want to** see conversion funnels (e.g., Homepage → Pricing → Signup)
**So that I can** identify where users drop off

**Priority:** P1 (Should-Have)
**Epic:** Goals & Funnels
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] Create funnel: Select 2-5 steps (goals or pages)
- [ ] Funnel shows: Step name, # of users, drop-off %
- [ ] Visualization: Horizontal bars with shrinking widths
- [ ] Time window: Users who completed funnel within 1 hour/day/week
- [ ] Breakdown by: Traffic source, device, country
- [ ] Click step to see users who dropped off
- [ ] Export funnel data to CSV
- [ ] Save funnels for future reference (up to 5 saved funnels)

**Dependencies:** US-5.1 (goals)

**Test Scenarios:**
1. Create funnel: Homepage → Pricing → Signup → 1000 → 500 → 100 (10% conversion)
2. Filter by mobile → Drop-off higher on step 2
3. Save funnel → Appears in "Saved Funnels" list

---

### US-5.3: Goal-Specific Revenue Attribution
**As a** user
**I want to** see revenue generated by users who completed a goal
**So that I can** calculate ROI per goal

**Priority:** P1 (Should-Have)
**Epic:** Goals & Funnels
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] Dashboard shows: Goal completions → Revenue
- [ ] Metric: Revenue per goal completion
- [ ] Time window: Revenue within 30 days of goal completion
- [ ] Filter: "Show revenue from users who completed X goal"
- [ ] Supports multiple goals: Revenue from users who completed A OR B

**Dependencies:** US-5.1 (goals), US-3.1 (revenue)

**Test Scenarios:**
1. 100 users complete "Signup" goal → 10 purchase → Revenue: $500 → Rev/Goal: $5
2. Filter dashboard by "Signup" goal → Shows only those 10 purchasing users

---

## 6. Real-Time Intelligence

### US-6.1: Real-Time Visitor Map
**As a** founder
**I want to** see live visitors on a map
**So that I can** watch my product going viral in real-time

**Priority:** P1 (Should-Have)
**Epic:** Real-Time
**Estimate:** 13 story points

**Acceptance Criteria:**
- [ ] Globe visualization (3D or 2D map)
- [ ] Visitor pins animate on arrival
- [ ] Update frequency: Every 5-10 seconds (WebSocket or polling)
- [ ] Click visitor pin → Show: Page, referrer, device, location (city/country)
- [ ] Active visitor count (real-time)
- [ ] Filter: Show only visitors who completed goal / generated revenue
- [ ] Map works on mobile (responsive, touch-friendly)
- [ ] Privacy: No PII displayed (no names/emails on map)
- [ ] Performance: Handles 100+ simultaneous visitors without lag

**Dependencies:** US-2.1 (tracking)

**Test Scenarios:**
1. New visitor arrives → Pin appears on map within 10 seconds
2. Click pin → Popup shows page URL, referrer
3. 100 visitors → Map renders smoothly
4. Filter "Revenue visitors" → Only purchasing visitors shown

---

### US-6.2: Visitor Journey Tracking
**As a** user
**I want to** see a visitor's page flow in current session
**So that I can** understand their behavior

**Priority:** P1 (Should-Have)
**Epic:** Real-Time
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] Click visitor on map → Show journey (list of pages visited)
- [ ] Journey shows: Page URL, timestamp, time on page
- [ ] Ordered chronologically (most recent first)
- [ ] Identify entry page, exit page
- [ ] Show goal completions in journey (highlighted)
- [ ] Session duration displayed
- [ ] Anonymized visitor ID (no PII)

**Dependencies:** US-6.1 (real-time map)

**Test Scenarios:**
1. Visitor visits: Homepage → Pricing → Signup → Journey shows 3 pages
2. Visitor completes goal → Goal highlighted in journey
3. Click "Entry page" → Shows referrer

---

### US-6.3: High-Value Visitor Alerts
**As a** growth marketer
**I want** real-time alerts when high-LTV visitors are on site
**So that I can** engage them proactively (e.g., chat)

**Priority:** P2 (Nice-to-Have – Phase 3)
**Epic:** Real-Time
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] Define "high-value": LTV > $X (user-configurable)
- [ ] Alert when high-value visitor arrives
- [ ] Alert methods: In-app notification, email, webhook
- [ ] Alert includes: Visitor details, current page, LTV
- [ ] Webhook for custom integrations (e.g., Slack, Intercom)
- [ ] Opt-in (not enabled by default)
- [ ] Rate limiting: Max 1 alert per visitor per day

**Dependencies:** US-6.1 (real-time), US-3.1 (revenue)

**Test Scenarios:**
1. Visitor with $500 LTV arrives → Alert sent
2. Same visitor returns → No duplicate alert (rate limit)
3. Webhook to Slack → Message posted in #sales channel

---

## 7. Integrations

### US-7.1: Google Tag Manager Integration
**As a** non-technical user
**I want to** install via Google Tag Manager
**So that I can** avoid editing my website's code

**Priority:** P1 (Should-Have)
**Epic:** Integrations
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] Documentation: Step-by-step GTM setup guide
- [ ] Tag type: Custom HTML
- [ ] Trigger: All Pages (pageview)
- [ ] Script auto-detects GTM environment (no conflicts)
- [ ] Verification: Test in GTM preview mode
- [ ] Video tutorial embedded in docs

**Dependencies:** US-1.3 (tracking script)

**Test Scenarios:**
1. Create GTM tag → Paste script → Publish → Pageviews tracked
2. Test in preview mode → Script fires on all pages

---

### US-7.2: X (Twitter) Attribution
**As an** indie maker active on X
**I want to** see which tweets drive revenue
**So that I can** optimize my X marketing strategy

**Priority:** P1 (Should-Have)
**Epic:** Integrations
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] Add X handles to track (e.g., @marclouvion)
- [ ] Auto-detect X referrers: `t.co`, `twitter.com`, `x.com`
- [ ] Show revenue per tweet (if URL in tweet tracked)
- [ ] Dashboard section: "X Attribution"
- [ ] Metrics: Visitors from X, Revenue, RPV, Tweets with revenue
- [ ] Compare: Organic X vs. X Ads
- [ ] Optional: Fetch tweet text via X API (requires API key)
- [ ] Handle multiple X accounts (personal, brand)

**Dependencies:** US-2.2 (UTM tracking), US-3.1 (revenue)

**Test Scenarios:**
1. Tweet with link → Visitors arrive → Revenue attributed to tweet
2. Add 2 X handles → Dashboard shows breakdown by handle
3. X API connected → Tweet text displayed in dashboard

---

### US-7.3: Webhook for Custom Events
**As a** developer
**I want** webhooks fired on events (new visitor, goal, purchase)
**So that I can** build custom automations

**Priority:** P1 (Should-Have – Phase 3)
**Epic:** Integrations
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] Configure webhook URL in settings
- [ ] Events: `visitor.created`, `goal.completed`, `purchase.created`
- [ ] Payload: JSON with event data (visitor_id, timestamp, etc.)
- [ ] Signature verification (HMAC-SHA256)
- [ ] Retry logic: 3 retries with exponential backoff
- [ ] Timeout: 5 seconds per request
- [ ] Logs: Success/failure status per webhook
- [ ] Test webhook button (sends sample payload)

**Dependencies:** US-5.1 (goals), US-3.1 (revenue)

**Test Scenarios:**
1. Configure webhook → Goal completed → Webhook fired within 10s
2. Webhook endpoint down → 3 retries, then marked failed
3. Invalid signature → Webhook rejected

---

## 8. API & Automation

### US-8.1: REST API for Metrics
**As a** developer
**I want** API access to analytics data
**So that I can** build custom dashboards

**Priority:** P1 (Should-Have – Phase 3)
**Epic:** API
**Estimate:** 13 story points

**Acceptance Criteria:**
- [ ] Authentication: Bearer token (generated in settings)
- [ ] Endpoints:
  - `GET /api/metrics` - Aggregated metrics (visitors, revenue, RPV)
  - `GET /api/visitors` - Visitor list with filters
  - `GET /api/revenue` - Revenue breakdown
  - `GET /api/goals` - Goal completions
- [ ] Query params: `start_date`, `end_date`, `source`, `device`, `goal`
- [ ] Pagination: `limit`, `offset` (max 1000 per request)
- [ ] Rate limiting: 100 req/min (configurable by plan)
- [ ] Response format: JSON
- [ ] Error handling: Standard HTTP codes (400, 401, 429, 500)
- [ ] API documentation: OpenAPI spec, examples

**Dependencies:** US-4.1 (dashboard), US-3.1 (revenue)

**Test Scenarios:**
1. `GET /api/metrics?start_date=2025-11-01` → Returns visitors, revenue
2. Exceed rate limit → 429 error: "Rate limit exceeded"
3. Invalid token → 401 error: "Unauthorized"

---

### US-8.2: API Token Management
**As a** user
**I want to** create and revoke API tokens
**So that I can** secure my data

**Priority:** P1 (Should-Have – Phase 3)
**Epic:** API
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] Generate API token in settings
- [ ] Token shown once (copy to clipboard)
- [ ] Token stored hashed (bcrypt)
- [ ] List active tokens: Name, created date, last used
- [ ] Revoke token (immediate invalidation)
- [ ] Max 5 tokens per user
- [ ] Token scopes (future): Read-only, read-write

**Dependencies:** US-8.1 (API)

**Test Scenarios:**
1. Create token → Shown once, copied
2. Use token in API call → Last used timestamp updates
3. Revoke token → API calls fail with 401

---

## 9. Account & Billing

### US-9.1: Plan Upgrade/Downgrade
**As a** user
**I want to** upgrade my plan when I hit event limits
**So that I can** continue tracking without interruption

**Priority:** P0 (Must-Have)
**Epic:** Billing
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] Dashboard shows: Current plan, event usage (e.g., 8,000 / 10,000)
- [ ] Alert at 80%, 90%, 100% of limit
- [ ] "Upgrade" button in dashboard
- [ ] Plan options: Starter ($99/mo), Growth ($199/mo)
- [ ] Upgrade: Immediate access, prorated charge
- [ ] Downgrade: Effective next billing cycle
- [ ] Overage handling: Pause tracking OR auto-upgrade (user choice)
- [ ] Stripe Customer Portal for self-service billing
- [ ] Cancel anytime, data export available

**Dependencies:** US-1.4 (Stripe)

**Test Scenarios:**
1. Hit 9,000 events → Alert: "80% of limit reached"
2. Click "Upgrade" → Redirected to Stripe, upgrade completed
3. Downgrade → Message: "Downgrade effective Dec 1"
4. Cancel plan → Data export offered

---

### US-9.2: 14-Day Free Trial
**As a** potential user
**I want** a free trial without credit card
**So that I can** evaluate the product risk-free

**Priority:** P0 (Must-Have)
**Epic:** Billing
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] No credit card required for trial
- [ ] Trial duration: 14 days
- [ ] Event limit during trial: 10,000 events
- [ ] Email reminders: Day 7, Day 12, Day 14 (expiring)
- [ ] After trial: Prompt to upgrade (no auto-charge)
- [ ] Trial data retained (not deleted) after expiry
- [ ] One trial per email (prevent abuse)

**Dependencies:** US-1.1 (signup)

**Test Scenarios:**
1. Sign up → Trial starts, 14-day countdown shown
2. Day 7 → Email: "7 days left in your trial"
3. Trial expires → Dashboard shows "Trial ended. Upgrade to continue."
4. Try second trial with same email → Error: "Trial already used"

---

### US-9.3: Invoice & Receipt Management
**As a** paying customer
**I want** access to my invoices
**So that I can** expense them or share with accounting

**Priority:** P1 (Should-Have)
**Epic:** Billing
**Estimate:** 3 story points

**Acceptance Criteria:**
- [ ] Billing section: List of invoices (date, amount, status)
- [ ] Download PDF invoice
- [ ] Invoices emailed automatically (Stripe feature)
- [ ] Update billing details: Address, VAT number
- [ ] VAT calculation for EU customers (Stripe Tax)

**Dependencies:** US-9.1 (billing)

**Test Scenarios:**
1. Navigate to Billing → List of past invoices shown
2. Click "Download" → PDF invoice downloads
3. Update VAT number → Next invoice includes VAT

---

## 10. Privacy & Compliance

### US-10.1: GDPR-Compliant Tracking
**As a** European business owner
**I want** GDPR-compliant analytics
**So that I can** avoid legal risks

**Priority:** P0 (Must-Have)
**Epic:** Privacy
**Estimate:** 8 story points

**Acceptance Criteria:**
- [ ] No cookies required (uses localStorage + fingerprinting)
- [ ] IP anonymization: Last octet hashed
- [ ] Data Processing Agreement (DPA) available for download
- [ ] Privacy policy and terms of service published
- [ ] Right to erasure: Delete visitor data on request (form in settings)
- [ ] Data retention: 2 years default (configurable: 1-5 years)
- [ ] Optional: Regional data storage (EU vs. US servers)
- [ ] Respects DNT (Do Not Track) header

**Dependencies:** US-2.1 (tracking)

**Test Scenarios:**
1. Enable DNT in browser → Events not tracked
2. Request data erasure → Visitor data deleted within 24 hours
3. Download DPA → PDF with legal terms

---

### US-10.2: Cookie Consent Integration
**As a** website owner
**I want to** integrate with my consent banner
**So that I can** comply with cookie laws

**Priority:** P1 (Should-Have)
**Epic:** Privacy
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] Script checks for consent before tracking
- [ ] Integration methods:
  - Check `window.datafast_consent` variable
  - Wait for custom event: `datafast:consent`
- [ ] Documentation for popular consent tools (CookieYes, Osano)
- [ ] If no consent, script does not fire
- [ ] Consent status visible in debug mode

**Dependencies:** US-2.1 (tracking)

**Test Scenarios:**
1. Set `window.datafast_consent = false` → Events not sent
2. Dispatch `datafast:consent` event → Tracking starts
3. Use CookieYes → Script waits for consent

---

### US-10.3: Data Export for Users
**As a** user exercising GDPR rights
**I want to** export all my tracked data
**So that I can** review what's collected

**Priority:** P1 (Should-Have)
**Epic:** Privacy
**Estimate:** 5 story points

**Acceptance Criteria:**
- [ ] Request data export in settings
- [ ] Export includes: All events, visitor IDs, revenue data
- [ ] Format: JSON or CSV (user choice)
- [ ] Export ready within 24 hours (async job)
- [ ] Email notification when ready
- [ ] Secure download link (expires in 7 days)
- [ ] Export file size limit: 100MB (split if larger)

**Dependencies:** US-4.2 (export)

**Test Scenarios:**
1. Request export → Email received within 24h with download link
2. Download → ZIP file with JSON/CSV
3. Link expires after 7 days → Error: "Link expired"

---

## Summary Statistics

**Total User Stories:** 43
**Priority Breakdown:**
- P0 (Must-Have): 16 stories
- P1 (Should-Have): 23 stories
- P2 (Nice-to-Have): 4 stories

**Epic Breakdown:**
- Onboarding: 5 stories
- Core Analytics: 3 stories
- Revenue Attribution: 4 stories
- Dashboard: 4 stories
- Goals & Funnels: 3 stories
- Real-Time: 3 stories
- Integrations: 3 stories
- API: 2 stories
- Billing: 3 stories
- Privacy: 3 stories

**Estimated Story Points:** ~240 points
**Estimated Sprints (2-week):** 8-10 sprints (assuming 25-30 points/sprint)

---

## Next Steps
1. Review user stories with stakeholders
2. Prioritize for Phase 1 MVP (focus on P0 stories)
3. Break down high-point stories (>8 points) into smaller tasks
4. Create technical tasks for each user story
5. Assign stories to sprints in project management tool

---

**Document Status:** ✅ Ready for Review
**Last Updated:** November 18, 2025
