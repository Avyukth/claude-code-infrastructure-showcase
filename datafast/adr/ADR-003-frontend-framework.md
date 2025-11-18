# ADR-003: Frontend Framework Selection

**Date:** 2025-11-18
**Status:** Accepted
**Deciders:** Engineering Team, Design Lead
**Technical Story:** [FR-3 - Dashboard & Metrics, NFR-5 - Usability]

## Context and Problem Statement

DataFa.st needs a modern, performant frontend for the analytics dashboard. The framework must:
- Support **fast development** for an indie team (1-2 developers)
- Deliver **excellent performance** (First Contentful Paint <1.5s)
- Enable **real-time updates** (WebSocket or polling for live visitor map)
- Be **SEO-friendly** (for marketing pages: homepage, pricing, blog)
- Support **component reusability** (design system)
- Have a strong **ecosystem** (charting, authentication, deployment)

The framework choice affects development speed, performance, and long-term maintainability.

## Decision Drivers

- **Developer Productivity:** Indie team needs fast iterations
- **Performance:** Dashboard load <2s (p95), 90+ Lighthouse score
- **Real-Time:** Support WebSocket or Server-Sent Events (SSE)
- **SEO:** Marketing pages must rank on Google
- **TypeScript:** Strong typing for code quality
- **Ecosystem:** Charts (Recharts), auth (NextAuth), deployment (Vercel)
- **Learning Curve:** Team has React experience
- **Community:** Active community, frequent updates

## Considered Options

1. **Next.js 14+** (React meta-framework, App Router)
2. **SvelteKit** (Svelte meta-framework)
3. **Remix** (React meta-framework, web standards focus)
4. **Astro** (Multi-framework, content-focused)

## Decision Outcome

**Chosen option:** "Next.js 14+ (App Router)"

**Rationale:**
Next.js offers the best balance of performance, DX, and ecosystem for DataFa.st. Key advantages:
- **App Router:** React Server Components (RSC) for fast initial loads
- **Streaming:** Instant page loads with Suspense boundaries
- **Vercel deployment:** Zero-config, edge caching, CI/CD
- **Ecosystem:** Largest React ecosystem (shadcn/ui, Recharts, NextAuth)
- **TypeScript:** First-class support, excellent IntelliSense
- **Real-time:** Easy WebSocket integration or React Query polling

The team's existing React knowledge eliminates learning curve.

### Positive Consequences

- **Fast development:** Component libraries (shadcn/ui) accelerate UI building
- **Excellent performance:** RSC + Streaming = <1.5s FCP
- **SEO-friendly:** Built-in SSR for marketing pages
- **Vercel deployment:** Deploy in <2 minutes, auto-scaling, edge CDN
- **Strong typing:** TypeScript prevents bugs, improves refactoring
- **Active community:** Huge ecosystem, frequent updates, easy hiring

### Negative Consequences

- **Vendor lock-in:** Optimized for Vercel (can deploy elsewhere but less ideal)
- **Complexity:** App Router has a learning curve (vs. Pages Router)
- **React overhead:** Larger bundle size than Svelte (but acceptable)
- **Breaking changes:** Next.js updates sometimes require rewrites

## Pros and Cons of the Options

### Next.js 14+ (App Router)

**Pros:**
- **React Server Components:** Faster loads, smaller bundles
- **Streaming:** Instant page loads with Suspense
- **Largest ecosystem:** shadcn/ui, Recharts, TanStack Query, NextAuth
- **Vercel deployment:** Best DX, auto-scaling, analytics built-in
- **TypeScript:** Excellent support, zero config
- **SEO:** Built-in SSR, metadata API
- **Image optimization:** Automatic WebP conversion, lazy loading
- **Team familiarity:** React experience transfers directly

**Cons:**
- Vendor lock-in (Vercel-optimized)
- App Router complexity (mental model shift from Pages Router)
- Larger bundle than Svelte (~50KB base React)
- Occasional breaking changes

---

### SvelteKit

**Pros:**
- **Smallest bundles:** Svelte compiles to vanilla JS (no runtime overhead)
- **Best performance:** Fastest framework in benchmarks
- **Excellent DX:** Less boilerplate than React
- **Built-in animations:** Transition API
- **TypeScript:** First-class support

**Cons:**
- **Smaller ecosystem:** Fewer charting libraries (Chart.js vs. Recharts/Nivo)
- **Learning curve:** Team must learn Svelte syntax
- **Hiring:** Harder to find Svelte developers (vs. React)
- **Component libraries:** Limited options (vs. shadcn/ui, MUI)
- **Deployment:** Less optimized for Vercel (primarily SvelteKit Adapter)

**Verdict:** Best performance, but ecosystem/hiring limitations are blockers.

---

### Remix

**Pros:**
- **Web standards focus:** Uses native FormData, fetch, etc.
- **Excellent DX:** No client-side JS for forms (progressive enhancement)
- **Nested routing:** Clean data loading per route
- **Fast:** Server-side rendering, smart prefetching

**Cons:**
- **Smaller ecosystem:** Newer framework, fewer resources
- **Limited real-time:** No built-in WebSocket support (requires custom setup)
- **Deployment:** Requires Node server (Vercel supported but not ideal)
- **Learning curve:** Different mental model from Next.js

**Verdict:** Great framework, but Next.js has stronger ecosystem and real-time support.

---

### Astro

**Pros:**
- **Zero JS by default:** Ship only necessary JavaScript
- **Multi-framework:** Can mix React, Svelte, Vue components
- **Excellent for content:** Blog, marketing pages

**Cons:**
- **Not ideal for apps:** Designed for content sites, not dashboards
- **Limited interactivity:** Requires manual island hydration
- **Charting complexity:** Hard to integrate real-time charts
- **Ecosystem:** Smaller than Next.js

**Verdict:** Excellent for marketing site, but not for analytics dashboard.

---

## Technical Implementation

### Project Structure

```
datafast-app/
├── app/                    # Next.js App Router
│   ├── (auth)/            # Auth pages (layout group)
│   │   ├── login/
│   │   └── signup/
│   ├── (dashboard)/       # Dashboard pages (layout group)
│   │   ├── layout.tsx     # Dashboard layout
│   │   ├── page.tsx       # Overview dashboard
│   │   ├── goals/
│   │   ├── funnels/
│   │   └── settings/
│   ├── (marketing)/       # Marketing pages (layout group)
│   │   ├── page.tsx       # Homepage
│   │   ├── pricing/
│   │   └── blog/
│   ├── api/               # API routes
│   │   ├── auth/
│   │   └── metrics/
│   └── layout.tsx         # Root layout
├── components/            # Shared components
│   ├── ui/                # shadcn/ui components
│   ├── charts/            # Chart wrappers
│   └── dashboard/         # Dashboard-specific components
├── lib/                   # Utilities
│   ├── db.ts              # Database client (Prisma/Drizzle)
│   ├── auth.ts            # NextAuth config
│   └── api.ts             # API client
├── public/                # Static assets
└── styles/                # Global styles (Tailwind)
```

### Key Technologies

**UI Components:** [shadcn/ui](https://ui.shadcn.com/)
- Radix UI primitives (accessible)
- Tailwind CSS styling
- Copy/paste components (no npm install)
- Dark mode support

**Charting:** [Recharts](https://recharts.org/)
- React-native, composable charts
- Responsive by default
- Good performance for 10K+ data points
- Fallback: [Nivo](https://nivo.rocks/) for advanced visualizations

**Data Fetching:** [TanStack Query (React Query)](https://tanstack.com/query)
- Server state management
- Auto-caching, refetching
- Polling for real-time updates (fallback if WebSocket complex)
- Optimistic updates

**Auth:** [NextAuth.js](https://next-auth.js.org/)
- OAuth providers (Google, X/Twitter)
- Email/password with JWT
- Database sessions (PostgreSQL)

**Styling:** Tailwind CSS
- Utility-first, fast prototyping
- Smaller CSS bundle than component libraries
- Dark mode with `class` strategy

**Deployment:** Vercel
- Git-based workflow (auto-deploy on push)
- Edge network, automatic caching
- Built-in analytics (Web Vitals)
- Environment variables management

---

### Real-Time Updates Strategy

**Option 1: Polling with React Query**
```typescript
// Simplest approach for MVP
const { data } = useQuery({
    queryKey: ['live-visitors'],
    queryFn: fetchLiveVisitors,
    refetchInterval: 5000, // Poll every 5 seconds
});
```

**Pros:** Simple, no server infrastructure changes
**Cons:** Higher server load, 5-second latency

**Option 2: Server-Sent Events (SSE)**
```typescript
// Next.js API route
export async function GET() {
    const stream = new ReadableStream({
        start(controller) {
            setInterval(() => {
                const data = getLiveVisitors();
                controller.enqueue(`data: ${JSON.stringify(data)}\n\n`);
            }, 5000);
        },
    });
    return new Response(stream, {
        headers: { 'Content-Type': 'text/event-stream' },
    });
}
```

**Pros:** Lower latency, one-way server push
**Cons:** Requires server streaming (Vercel supports with caveats)

**Option 3: WebSocket (Phase 2)**
```typescript
// Separate WebSocket server (e.g., Fly.io)
const ws = new WebSocket('wss://ws.datafa.st');
ws.onmessage = (event) => {
    const visitor = JSON.parse(event.data);
    setLiveVisitors(prev => [...prev, visitor]);
};
```

**Pros:** True real-time, bi-directional
**Cons:** Requires separate server (Vercel doesn't support WebSockets)

**Decision for MVP:** Start with **Polling (React Query)**. Upgrade to SSE/WebSocket in Phase 2 if needed.

---

## Performance Targets

**Metrics (Lighthouse):**
- Performance: 90+
- Accessibility: 100
- Best Practices: 100
- SEO: 100

**Core Web Vitals:**
- First Contentful Paint (FCP): <1.5s
- Largest Contentful Paint (LCP): <2.5s
- Cumulative Layout Shift (CLS): <0.1
- Interaction to Next Paint (INP): <200ms

**Strategies:**
- React Server Components for initial page load
- Suspense boundaries for streaming
- Dynamic imports for code splitting
- Image optimization (next/image)
- Font optimization (next/font)
- Edge caching (Vercel CDN)

---

## Migration Path (if needed)

If Next.js proves unsuitable (e.g., Vercel costs too high):

**Alternative Deployment:**
1. Docker container → Fly.io or Railway
2. Static export → Cloudflare Pages (for marketing pages)
3. Hybrid: Marketing on Astro, Dashboard on Next.js

**Alternative Framework:**
1. Svelte Kit (better performance, smaller ecosystem)
2. Remix (better web standards, harder deployment)

**Risk:** Low (Next.js is industry-proven, Vercel scales to millions of users)

---

## Links

- [Next.js 14 Documentation](https://nextjs.org/docs)
- [App Router Guide](https://nextjs.org/docs/app)
- [shadcn/ui Components](https://ui.shadcn.com/)
- [TanStack Query Docs](https://tanstack.com/query/latest)
- [NextAuth.js Docs](https://next-auth.js.org/)
- [Vercel Deployment](https://vercel.com/docs)
- [PRD Section 5: Design & UX](../prd/DATAFAST-PRD.md#5-design--ux)

---

**Reviewed by:** [Pending]
**Approved by:** [Pending]
