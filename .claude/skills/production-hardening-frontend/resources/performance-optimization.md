# Performance Optimization - SvelteKit Production

**Enterprise-grade performance patterns for SvelteKit applications**

This resource provides comprehensive performance optimization strategies for SvelteKit applications, aligned with Core Web Vitals standards and modern web performance best practices.

---

## Core Web Vitals Targets

### Performance Metrics

| Metric | Target | Good | Needs Improvement | Poor |
|--------|--------|------|-------------------|------|
| **LCP** (Largest Contentful Paint) | < 2.5s | < 2.5s | 2.5s - 4s | > 4s |
| **FID** (First Input Delay) | < 100ms | < 100ms | 100ms - 300ms | > 300ms |
| **INP** (Interaction to Next Paint) | < 200ms | < 200ms | 200ms - 500ms | > 500ms |
| **CLS** (Cumulative Layout Shift) | < 0.1 | < 0.1 | 0.1 - 0.25 | > 0.25 |
| **TTFB** (Time to First Byte) | < 600ms | < 600ms | 600ms - 1.8s | > 1.8s |
| **FCP** (First Contentful Paint) | < 1.8s | < 1.8s | 1.8s - 3s | > 3s |

---

## 1. Code Splitting Strategies

### Route-Based Code Splitting

SvelteKit automatically code-splits at route boundaries. Optimize this further:

```typescript
// src/routes/+layout.ts
export const load = async ({ fetch }) => {
  // Only load critical data in layout
  const config = await fetch('/api/config').then(r => r.json());

  return {
    config
    // Don't load heavy data here - do it in page load functions
  };
};
```

```typescript
// src/routes/dashboard/+page.ts
export const load = async ({ fetch, parent }) => {
  const { config } = await parent(); // Get layout data

  // Load page-specific data - creates separate chunk
  const [stats, activities] = await Promise.all([
    fetch('/api/stats').then(r => r.json()),
    fetch('/api/activities').then(r => r.json())
  ]);

  return { stats, activities };
};
```

### Dynamic Imports for Heavy Components

```svelte
<script lang="ts">
  import { onMount } from 'svelte';

  let ChartComponent: any;
  let chartData = $state<any>(null);

  onMount(async () => {
    // Lazy load heavy chart library only when needed
    const module = await import('$lib/components/charts/AdvancedChart.svelte');
    ChartComponent = module.default;

    // Load data after component
    chartData = await fetch('/api/chart-data').then(r => r.json());
  });
</script>

{#if ChartComponent && chartData}
  <svelte:component this={ChartComponent} data={chartData} />
{:else}
  <div class="skeleton-loader">Loading chart...</div>
{/if}
```

### Conditional Loading Based on Device

```typescript
// src/lib/utils/device.ts
export function isMobile(): boolean {
  return window.matchMedia('(max-width: 768px)').matches;
}

export async function loadMobileOptimized<T>(
  mobileImport: () => Promise<T>,
  desktopImport: () => Promise<T>
): Promise<T> {
  const module = isMobile()
    ? await mobileImport()
    : await desktopImport();

  return module;
}
```

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import { loadMobileOptimized } from '$lib/utils/device';

  let MapComponent: any;

  onMount(async () => {
    MapComponent = await loadMobileOptimized(
      () => import('$lib/components/maps/SimpleMobileMap.svelte'),
      () => import('$lib/components/maps/FullFeaturedMap.svelte')
    );
  });
</script>
```

---

## 2. Image Optimization

### Using @sveltejs/enhanced-img (Svelte 5+)

```svelte
<script>
  import { Image } from '@sveltejs/enhanced-img';
  import heroImage from '$lib/assets/hero.jpg?enhanced';
</script>

<!-- Automatic WebP/AVIF generation, responsive srcset -->
<Image
  src={heroImage}
  alt="Hero banner"
  sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
/>
```

**Features:**
- Automatic format conversion (WebP, AVIF)
- Responsive image generation
- Lazy loading by default
- Optimized delivery

### Manual Image Optimization with vite-imagetools

```javascript
// vite.config.js
import { sveltekit } from '@sveltejs/kit/vite';
import { imagetools } from 'vite-imagetools';

export default {
  plugins: [
    sveltekit(),
    imagetools({
      defaultDirectives: new URLSearchParams({
        format: 'webp;avif;jpg',
        quality: '80',
        w: '400;800;1200'
      })
    })
  ]
};
```

```svelte
<script>
  import image from '$lib/assets/photo.jpg?w=400;800;1200&format=webp;avif;jpg';
</script>

<picture>
  <source
    type="image/avif"
    srcset={image.avif.map((img, i) =>
      `${img.src} ${[400, 800, 1200][i]}w`
    ).join(', ')}
  />
  <source
    type="image/webp"
    srcset={image.webp.map((img, i) =>
      `${img.src} ${[400, 800, 1200][i]}w`
    ).join(', ')}
  />
  <img
    src={image.jpg[0].src}
    alt="Description"
    loading="lazy"
    decoding="async"
  />
</picture>
```

### Lazy Loading Images Below Fold

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import { inview } from 'svelte-inview';

  let isInView = $state(false);
</script>

<div
  use:inview
  on:inview_change={(event) => {
    isInView = event.detail.inView;
  }}
>
  {#if isInView}
    <img src="/images/heavy-image.jpg" alt="Below fold content" />
  {:else}
    <div class="placeholder" style="aspect-ratio: 16/9; background: #eee;"></div>
  {/if}
</div>
```

### Blur-Up Placeholder Technique

```svelte
<script lang="ts">
  import { onMount } from 'svelte';

  let loaded = $state(false);

  const blurDataUrl = "data:image/jpeg;base64,/9j/4AAQSkZJRg..."; // Tiny base64
</script>

<div class="image-container">
  <img
    src={blurDataUrl}
    alt="Product"
    class="blur"
    class:hidden={loaded}
  />
  <img
    src="/images/product-full.jpg"
    alt="Product"
    onload={() => loaded = true}
    class:visible={loaded}
  />
</div>

<style>
  .image-container {
    position: relative;
  }

  img {
    position: absolute;
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .blur {
    filter: blur(20px);
    transition: opacity 0.3s;
  }

  .blur.hidden {
    opacity: 0;
  }

  .visible {
    opacity: 1;
    transition: opacity 0.3s;
  }
</style>
```

---

## 3. SSR vs CSR vs Prerendering Trade-offs

### When to Use Each Strategy

#### Server-Side Rendering (SSR) - Default
✅ **Use for:**
- Dynamic content (user-specific data)
- SEO-critical pages with changing content
- Pages requiring authentication
- Real-time data

```typescript
// src/routes/dashboard/+page.server.ts
export const load: PageServerLoad = async ({ locals }) => {
  // Runs on server for every request
  const user = locals.user;
  const data = await db.getUserDashboard(user.id);

  return { data };
};
```

**Performance characteristics:**
- TTFB: Higher (server processing time)
- FCP: Fast (HTML sent immediately)
- LCP: Fast (content rendered on server)

#### Client-Side Rendering (CSR)
✅ **Use for:**
- Highly interactive apps
- Admin panels
- Authenticated-only areas
- When SEO not critical

```typescript
// src/routes/admin/+page.ts
export const load: PageLoad = async ({ fetch }) => {
  // Runs in browser
  const data = await fetch('/api/admin/stats').then(r => r.json());
  return { data };
};
```

**Performance characteristics:**
- TTFB: Low (static HTML)
- FCP: Slower (wait for JS)
- LCP: Slower (wait for data fetch + render)

#### Prerendering (Static Generation)
✅ **Use for:**
- Marketing pages
- Documentation
- Blog posts
- Any content that doesn't change per-user

```typescript
// src/routes/blog/[slug]/+page.ts
export const prerender = true;

export const load: PageLoad = async ({ params, fetch }) => {
  const post = await fetch(`/api/posts/${params.slug}`).then(r => r.json());
  return { post };
};
```

**Performance characteristics:**
- TTFB: Fastest (CDN edge)
- FCP: Fastest (no server processing)
- LCP: Fastest (complete HTML)

### Hybrid Approach - Best Practice

```typescript
// src/routes/blog/+page.ts
export const prerender = true; // Static list

// src/routes/blog/[slug]/+page.server.ts
export const prerender = true; // Static posts

// src/routes/dashboard/+page.server.ts
// SSR by default - dynamic user data

// src/routes/admin/+page.ts
export const ssr = false; // CSR for admin
```

---

## 4. Bundle Size Optimization

### Analyze Bundle Size

```bash
# Install analyzer
npm install -D vite-bundle-visualizer

# Build and analyze
npm run build
npx vite-bundle-visualizer
```

### Configure Manual Chunks

```javascript
// vite.config.js
import { sveltekit } from '@sveltejs/kit/vite';

export default {
  plugins: [sveltekit()],
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          // Vendor chunk (< 150KB)
          if (id.includes('node_modules')) {
            // Split large vendors
            if (id.includes('chart.js')) {
              return 'vendor-charts';
            }
            if (id.includes('@tanstack')) {
              return 'vendor-query';
            }
            return 'vendor';
          }
        }
      }
    },
    chunkSizeWarningLimit: 200 // KB - warn if exceeded
  }
};
```

### Tree Shaking - Remove Unused Code

```typescript
// ❌ BAD - Imports entire library
import _ from 'lodash';
const result = _.debounce(fn, 300);

// ✅ GOOD - Import only what you need
import { debounce } from 'lodash-es';
const result = debounce(fn, 300);

// ✅ BETTER - Use native or smaller alternatives
function debounce(fn: Function, ms: number) {
  let timer: ReturnType<typeof setTimeout>;
  return (...args: any[]) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), ms);
  };
}
```

### Remove Development-Only Code

```typescript
// vite.config.js
export default {
  define: {
    __DEV__: JSON.stringify(process.env.NODE_ENV !== 'production')
  }
};
```

```typescript
// Usage in code
if (__DEV__) {
  console.log('Debug info'); // Removed in production build
}
```

---

## 5. Font Loading Optimization

### Self-Hosted Fonts (Recommended)

```css
/* src/app.css */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-weight: 100 900;
  font-display: swap; /* Show fallback immediately */
  font-style: normal;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}
```

### Preload Critical Fonts

```svelte
<!-- src/app.html -->
<head>
  <!-- Preload critical font -->
  <link
    rel="preload"
    href="/fonts/inter-var.woff2"
    as="font"
    type="font/woff2"
    crossorigin
  />
</head>
```

### Variable Fonts for Performance

```css
/* Single file for all weights instead of multiple files */
@font-face {
  font-family: 'Inter Variable';
  src: url('/fonts/inter-var.woff2') format('woff2-variations');
  font-weight: 100 900; /* All weights in one file */
  font-display: swap;
}

.heading {
  font-weight: 700; /* Uses same file */
}

.body {
  font-weight: 400; /* Uses same file */
}
```

### Subset Fonts (Advanced)

```bash
# Install pyftsubset
pip install fonttools brotli

# Subset to Latin characters only (reduces size by ~70%)
pyftsubset Inter-Regular.ttf \
  --output-file=Inter-Regular-subset.woff2 \
  --flavor=woff2 \
  --layout-features='*' \
  --unicodes=U+0000-00FF,U+0131,U+0152-0153,U+02BB-02BC,U+02C6,U+02DA,U+02DC,U+2000-206F,U+2074,U+20AC,U+2122,U+2191,U+2193,U+2212,U+2215,U+FEFF,U+FFFD
```

---

## 6. Critical Rendering Path Optimization

### Inline Critical CSS

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import '$lib/styles/app.css'; // Non-critical CSS
</script>

<svelte:head>
  <!-- Inline critical above-fold CSS -->
  <style>
    /* Critical CSS extracted and inlined */
    body { margin: 0; font-family: system-ui; }
    .hero { min-height: 100vh; background: #000; color: #fff; }
  </style>
</svelte:head>
```

### Defer Non-Critical JavaScript

```svelte
<svelte:head>
  <!-- Analytics - not critical, load async -->
  <script async src="https://www.googletagmanager.com/gtag/js?id=GA_ID"></script>

  <!-- Third-party widgets - defer -->
  <script defer src="https://widget.example.com/embed.js"></script>
</svelte:head>
```

### Preconnect to Required Origins

```svelte
<svelte:head>
  <!-- Preconnect to API server -->
  <link rel="preconnect" href="https://api.example.com" />

  <!-- DNS prefetch for CDN -->
  <link rel="dns-prefetch" href="https://cdn.example.com" />
</svelte:head>
```

---

## 7. Service Worker Caching Strategies

### Install Workbox

```bash
npm install -D @sveltejs/adapter-auto workbox-precaching workbox-routing workbox-strategies
```

### Configure Service Worker

```typescript
// src/service-worker.ts
/// <reference types="@sveltejs/kit" />
import { build, files, version } from '$service-worker';
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute } from 'workbox-routing';
import { NetworkFirst, CacheFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { ExpirationPlugin } from 'workbox-expiration';

// Precache build assets
const precacheList = [
  ...build.map(file => ({ url: file, revision: version })),
  ...files.map(file => ({ url: file, revision: version }))
];

precacheAndRoute(precacheList);

// API responses - Network First (fresh data, fallback to cache)
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 5 * 60 // 5 minutes
      })
    ]
  })
);

// Images - Cache First (static, long-lived)
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'image-cache',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 100,
        maxAgeSeconds: 30 * 24 * 60 * 60 // 30 days
      })
    ]
  })
);

// Fonts - Cache First with long expiration
registerRoute(
  ({ request }) => request.destination === 'font',
  new CacheFirst({
    cacheName: 'font-cache',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 30,
        maxAgeSeconds: 365 * 24 * 60 * 60 // 1 year
      })
    ]
  })
);

// HTML pages - Stale While Revalidate
registerRoute(
  ({ request }) => request.mode === 'navigate',
  new StaleWhileRevalidate({
    cacheName: 'pages-cache'
  })
);
```

---

## 8. Compression

### Vite Compression Plugin

```bash
npm install -D vite-plugin-compression
```

```javascript
// vite.config.js
import { sveltekit } from '@sveltejs/kit/vite';
import compression from 'vite-plugin-compression';

export default {
  plugins: [
    sveltekit(),

    // Brotli compression (better than Gzip)
    compression({
      algorithm: 'brotliCompress',
      ext: '.br',
      threshold: 1024, // Only compress files > 1KB
      deleteOriginFile: false
    }),

    // Gzip fallback for older browsers
    compression({
      algorithm: 'gzip',
      ext: '.gz',
      threshold: 1024
    })
  ]
};
```

### Server-Side Compression (Node Adapter)

```typescript
// server.js (custom server with Node adapter)
import express from 'express';
import compression from 'compression';
import { handler } from './build/handler.js';

const app = express();

// Enable compression
app.use(compression({
  level: 9, // Maximum compression
  threshold: 1024,
  filter: (req, res) => {
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  }
}));

app.use(handler);

app.listen(3000);
```

---

## 9. Database Query Optimization

### N+1 Query Prevention

```typescript
// ❌ BAD - N+1 queries
export const load: PageServerLoad = async () => {
  const users = await db.user.findMany();

  // This runs a query for EACH user!
  const usersWithPosts = await Promise.all(
    users.map(async (user) => ({
      ...user,
      posts: await db.post.findMany({ where: { authorId: user.id } })
    }))
  );

  return { users: usersWithPosts };
};

// ✅ GOOD - Single query with include
export const load: PageServerLoad = async () => {
  const users = await db.user.findMany({
    include: {
      posts: true // Single query with JOIN
    }
  });

  return { users };
};
```

### Pagination for Large Datasets

```typescript
export const load: PageServerLoad = async ({ url }) => {
  const page = parseInt(url.searchParams.get('page') || '1');
  const pageSize = 20;

  const [items, total] = await Promise.all([
    db.item.findMany({
      skip: (page - 1) * pageSize,
      take: pageSize,
      orderBy: { createdAt: 'desc' }
    }),
    db.item.count()
  ]);

  return {
    items,
    pagination: {
      page,
      pageSize,
      totalPages: Math.ceil(total / pageSize),
      total
    }
  };
};
```

### Database Connection Pooling

```typescript
// src/lib/server/db.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as {
  prisma: PrismaClient | undefined;
};

export const db = globalForPrisma.prisma ?? new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
  // Connection pool configuration
  datasources: {
    db: {
      url: process.env.DATABASE_URL
    }
  }
});

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = db;
}

// Close connections gracefully
process.on('beforeExit', async () => {
  await db.$disconnect();
});
```

---

## 10. Performance Monitoring

### Web Vitals Measurement

```typescript
// src/lib/analytics/web-vitals.ts
import { onCLS, onFID, onLCP, onFCP, onTTFB, onINP } from 'web-vitals';

function sendToAnalytics(metric: any) {
  const body = JSON.stringify(metric);

  // Use `navigator.sendBeacon()` if available, falling back to `fetch()`
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/analytics', body);
  } else {
    fetch('/api/analytics', { body, method: 'POST', keepalive: true });
  }
}

export function initWebVitals() {
  onCLS(sendToAnalytics);
  onFID(sendToAnalytics);
  onLCP(sendToAnalytics);
  onFCP(sendToAnalytics);
  onTTFB(sendToAnalytics);
  onINP(sendToAnalytics);
}
```

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import { onMount } from 'svelte';
  import { initWebVitals } from '$lib/analytics/web-vitals';

  onMount(() => {
    initWebVitals();
  });
</script>
```

### Performance Observer API

```typescript
// src/lib/performance/observer.ts
export function observeLongTasks() {
  if (!('PerformanceObserver' in window)) return;

  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (entry.duration > 50) { // Long task threshold
        console.warn('Long task detected:', {
          duration: entry.duration,
          startTime: entry.startTime
        });
      }
    }
  });

  observer.observe({ entryTypes: ['longtask'] });
}
```

---

## Performance Budget Enforcement

### Lighthouse CI Configuration

```yaml
# .lighthouserc.yml
ci:
  collect:
    numberOfRuns: 3
    url:
      - http://localhost:3000
      - http://localhost:3000/dashboard
  assert:
    preset: 'lighthouse:recommended'
    assertions:
      # Core Web Vitals
      'largest-contentful-paint': ['error', { maxNumericValue: 2500 }]
      'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }]
      'total-blocking-time': ['error', { maxNumericValue: 300 }]

      # Performance score
      'categories:performance': ['error', { minScore: 0.9 }]

      # Bundle size
      'total-byte-weight': ['warn', { maxNumericValue: 500000 }]
      'dom-size': ['warn', { maxNumericValue: 1500 }]
  upload:
    target: 'temporary-public-storage'
```

### Bundle Size Limits in package.json

```json
{
  "scripts": {
    "build": "vite build",
    "check-size": "bundlewatch"
  },
  "bundlewatch": {
    "files": [
      {
        "path": "./build/**/*.js",
        "maxSize": "200kb"
      },
      {
        "path": "./build/**/*.css",
        "maxSize": "50kb"
      }
    ]
  }
}
```

---

**Related Resources:**
- [Security Hardening](./security-hardening.md)
- [Deployment and Operations](./deployment-operations.md)
- [Accessibility and UX](./accessibility-ux.md)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Compliance**: Core Web Vitals, Lighthouse Best Practices
