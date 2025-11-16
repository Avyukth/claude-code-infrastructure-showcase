# SvelteKit PWA Patterns - Mobile-First PWA Design

**Related**: [SKILL.md](../SKILL.md) | [pwa-fundamentals.md](pwa-fundamentals.md) | [performance-optimization.md](performance-optimization.md)

A comprehensive guide to implementing Progressive Web Apps with SvelteKit, covering service worker integration, SSR considerations, build-time precaching, deployment strategies, and testing patterns specific to the SvelteKit framework.

---

## Table of Contents

1. [SvelteKit Service Worker Integration](#sveltekit-service-worker-integration)
2. [Using $service-worker Module](#using-service-worker-module)
3. [SSR Considerations for PWAs](#ssr-considerations-for-pwas)
4. [Adapter Selection](#adapter-selection)
5. [Build-Time Precaching](#build-time-precaching)
6. [Dynamic Routes and Caching](#dynamic-routes-and-caching)
7. [Environment-Specific Service Workers](#environment-specific-service-workers)
8. [Testing SvelteKit PWAs](#testing-sveltekit-pwas)
9. [Deployment Strategies](#deployment-strategies)

---

## SvelteKit Service Worker Integration

### Service Worker File Location

SvelteKit automatically processes `src/service-worker.js` (or `.ts`) during build:

```
src/
├── routes/
│   ├── +layout.svelte
│   └── +page.svelte
├── service-worker.ts    ← Service worker file
└── app.html
```

**Key Features**:
- Automatically bundled during `vite build`
- Access to `$service-worker` module for build info
- TypeScript support with proper types
- Hot module replacement during development

### Basic Service Worker Setup

```typescript
// src/service-worker.ts
/// <reference types="@sveltejs/kit" />
/// <reference no-default-lib="true"/>
/// <reference lib="esnext" />
/// <reference lib="webworker" />

import { build, files, version } from '$service-worker';

const sw = self as unknown as ServiceWorkerGlobalScope;

const CACHE_NAME = `cache-${version}`;

// Files to cache on install
const ASSETS = [
  ...build, // App-generated files (JS, CSS)
  ...files  // Static files from /static
];

// Install event: Cache assets
sw.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(ASSETS);
    }).then(() => {
      sw.skipWaiting();
    })
  );
});

// Activate event: Clean up old caches
sw.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((keys) => {
      return Promise.all(
        keys
          .filter((key) => key !== CACHE_NAME)
          .map((key) => caches.delete(key))
      );
    }).then(() => {
      sw.clients.claim();
    })
  );
});

// Fetch event: Serve from cache, fallback to network
sw.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return;

  event.respondWith(
    caches.match(event.request).then((cached) => {
      return cached || fetch(event.request);
    })
  );
});
```

### Registering the Service Worker

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import { browser, dev } from '$app/environment';
  import { onMount } from 'svelte';

  onMount(async () => {
    // Only register service worker in production
    if (browser && 'serviceWorker' in navigator && !dev) {
      try {
        const registration = await navigator.serviceWorker.register('/service-worker.js');

        console.log('Service Worker registered:', registration);

        // Check for updates every hour
        setInterval(() => {
          registration.update();
        }, 60 * 60 * 1000);

        // Listen for new service worker
        registration.addEventListener('updatefound', () => {
          const newWorker = registration.installing;

          newWorker?.addEventListener('statechange', () => {
            if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
              // New service worker available, show update notification
              console.log('New version available! Refresh to update.');
            }
          });
        });
      } catch (error) {
        console.error('Service Worker registration failed:', error);
      }
    }
  });
</script>

<slot />
```

---

## Using $service-worker Module

The `$service-worker` module provides build-time information:

```typescript
import { build, files, version } from '$service-worker';

// build: Array of app-generated files
// Example: ['/_app/immutable/chunks/index.abc123.js', '/_app/immutable/assets/styles.def456.css']

// files: Array of static files from /static
// Example: ['/favicon.png', '/logo.svg', '/manifest.json']

// version: Build timestamp/hash for cache versioning
// Example: '1699876543210'
```

### Advanced Caching with Workbox

```bash
npm install workbox-precaching workbox-routing workbox-strategies
```

```typescript
// src/service-worker.ts
import { build, files, version } from '$service-worker';
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute, NavigationRoute } from 'workbox-routing';
import { NetworkFirst, CacheFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { CacheableResponsePlugin } from 'workbox-cacheable-response';
import { ExpirationPlugin } from 'workbox-expiration';

const sw = self as unknown as ServiceWorkerGlobalScope;

// Precache app shell and static assets
const precacheManifest = [
  ...build.map(file => ({ url: file, revision: version })),
  ...files.map(file => ({ url: file, revision: null }))
];

precacheAndRoute(precacheManifest);

// Cache images with CacheFirst strategy
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new CacheableResponsePlugin({ statuses: [0, 200] }),
      new ExpirationPlugin({
        maxEntries: 100,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
        purgeOnQuotaError: true
      })
    ]
  })
);

// Cache API calls with NetworkFirst strategy
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    plugins: [
      new CacheableResponsePlugin({ statuses: [0, 200] }),
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 5 * 60 // 5 minutes
      })
    ],
    networkTimeoutSeconds: 3
  })
);

// Cache Google Fonts with StaleWhileRevalidate
registerRoute(
  ({ url }) => url.origin === 'https://fonts.googleapis.com' || url.origin === 'https://fonts.gstatic.com',
  new StaleWhileRevalidate({
    cacheName: 'google-fonts',
    plugins: [
      new CacheableResponsePlugin({ statuses: [0, 200] }),
      new ExpirationPlugin({
        maxEntries: 20,
        maxAgeSeconds: 365 * 24 * 60 * 60 // 1 year
      })
    ]
  })
);

// App shell navigation fallback
const appShellRoute = new NavigationRoute(
  new NetworkFirst({
    cacheName: 'app-shell',
    plugins: [
      new CacheableResponsePlugin({ statuses: [0, 200] })
    ]
  })
);

registerRoute(appShellRoute);

// Offline fallback page
const OFFLINE_URL = '/offline';

sw.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('offline').then((cache) => {
      return cache.add(OFFLINE_URL);
    })
  );
});

sw.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => {
        return caches.match(OFFLINE_URL) as Promise<Response>;
      })
    );
  }
});
```

---

## SSR Considerations for PWAs

### Detecting SSR vs CSR

```svelte
<script lang="ts">
  import { browser } from '$app/environment';

  // This runs both server-side and client-side
  console.log('Runs everywhere');

  // Client-only code
  if (browser) {
    console.log('Client-side only');

    // Register service worker
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/service-worker.js');
    }
  }

  // Or use onMount for client-only
  import { onMount } from 'svelte';

  onMount(() => {
    // This only runs client-side
    console.log('Client-side only (onMount)');
  });
</script>
```

### Hydration-Friendly PWA Features

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import { browser } from '$app/environment';
  import { onMount } from 'svelte';

  let isOnline = true;
  let installPrompt: Event | null = null;

  // Client-only reactive state
  if (browser) {
    isOnline = navigator.onLine;
  }

  onMount(() => {
    // Online/offline detection
    function updateOnlineStatus() {
      isOnline = navigator.onLine;
    }

    window.addEventListener('online', updateOnlineStatus);
    window.addEventListener('offline', updateOnlineStatus);

    // Capture install prompt
    window.addEventListener('beforeinstallprompt', (e) => {
      e.preventDefault();
      installPrompt = e;
    });

    return () => {
      window.removeEventListener('online', updateOnlineStatus);
      window.removeEventListener('offline', updateOnlineStatus);
    };
  });

  async function promptInstall() {
    if (!installPrompt) return;

    (installPrompt as any).prompt();

    const { outcome } = await (installPrompt as any).userChoice;
    console.log(`User ${outcome === 'accepted' ? 'accepted' : 'dismissed'} the install prompt`);

    installPrompt = null;
  }
</script>

{#if !isOnline}
  <div class="offline-banner">
    You are currently offline. Some features may be unavailable.
  </div>
{/if}

{#if installPrompt}
  <button class="install-banner" on:click={promptInstall}>
    Install this app for a better experience
  </button>
{/if}

<slot />
```

### Prerendering for PWAs

```typescript
// src/routes/+page.server.ts
export const prerender = true; // Prerender at build time

export async function load() {
  // This runs at build time (SSR)
  return {
    posts: await fetchPosts()
  };
}
```

```typescript
// svelte.config.js
import adapter from '@sveltejs/adapter-static';

export default {
  kit: {
    adapter: adapter({
      pages: 'build',
      assets: 'build',
      fallback: '200.html', // SPA fallback for client-side routing
      precompress: true,
      strict: false
    }),
    prerender: {
      entries: [
        '/',
        '/about',
        '/contact',
        '/offline' // Include offline page
      ]
    }
  }
};
```

---

## Adapter Selection

### Static Adapter (Best for PWAs)

**Use when**: Your app can be fully prerendered (static site, JAMstack)

```bash
npm install -D @sveltejs/adapter-static
```

```typescript
// svelte.config.js
import adapter from '@sveltejs/adapter-static';

export default {
  kit: {
    adapter: adapter({
      pages: 'build',
      assets: 'build',
      fallback: 'index.html', // SPA mode
      precompress: true // Precompress files with gzip and brotli
    })
  }
};
```

**Deployment**: Netlify, Vercel, GitHub Pages, Cloudflare Pages

### Auto Adapter (SSR with PWA Support)

**Use when**: You need server-side rendering with PWA features

```bash
npm install -D @sveltejs/adapter-auto
```

```typescript
// svelte.config.js
import adapter from '@sveltejs/adapter-auto';

export default {
  kit: {
    adapter: adapter()
  }
};
```

**Deployment**: Vercel (automatic), Netlify (automatic)

### Node Adapter (Custom Server)

**Use when**: You need full control over the server

```bash
npm install -D @sveltejs/adapter-node
```

```typescript
// svelte.config.js
import adapter from '@sveltejs/adapter-node';

export default {
  kit: {
    adapter: adapter({
      out: 'build',
      precompress: true
    })
  }
};
```

**Deployment**: Custom Node.js server, Docker containers

---

## Build-Time Precaching

### Selective Precaching

```typescript
// src/service-worker.ts
import { build, files } from '$service-worker';

// Only precache critical assets
const criticalAssets = [
  ...build, // Always cache app bundle
  ...files.filter(file => {
    // Only precache essential static files
    return (
      file.includes('manifest.json') ||
      file.includes('favicon') ||
      file.includes('offline.html') ||
      file.endsWith('.woff2') // Fonts
    );
  })
];
```

### Lazy Caching for Large Assets

```typescript
// src/service-worker.ts
import { registerRoute } from 'workbox-routing';
import { CacheFirst } from 'workbox-strategies';

// Don't precache images, cache on demand
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images-runtime',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 7 * 24 * 60 * 60 // 7 days
      })
    ]
  })
);
```

---

## Dynamic Routes and Caching

### Caching Dynamic API Routes

```typescript
// src/routes/api/posts/+server.ts
import { json } from '@sveltejs/kit';
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async ({ setHeaders }) => {
  const posts = await fetchPosts();

  // Set cache headers for service worker
  setHeaders({
    'Cache-Control': 'public, max-age=300' // 5 minutes
  });

  return json(posts);
};
```

```typescript
// src/service-worker.ts
import { NetworkFirst } from 'workbox-strategies';

registerRoute(
  ({ url }) => url.pathname.startsWith('/api/posts'),
  new NetworkFirst({
    cacheName: 'posts-api',
    plugins: [
      new CacheableResponsePlugin({ statuses: [0, 200] }),
      new ExpirationPlugin({
        maxEntries: 20,
        maxAgeSeconds: 5 * 60 // 5 minutes
      })
    ],
    networkTimeoutSeconds: 3
  })
);
```

### Parameterized Routes

```typescript
// src/routes/products/[id]/+page.svelte
<script lang="ts">
  import { page } from '$app/stores';
  import { onMount } from 'svelte';

  export let data;

  onMount(() => {
    // Precache next/prev products for faster navigation
    const currentId = parseInt($page.params.id);

    if (currentId > 1) {
      fetch(`/api/products/${currentId - 1}`); // Cache previous
    }
    fetch(`/api/products/${currentId + 1}`); // Cache next
  });
</script>
```

```typescript
// src/service-worker.ts
registerRoute(
  ({ url }) => url.pathname.match(/^\/api\/products\/\d+$/),
  new NetworkFirst({
    cacheName: 'product-details',
    plugins: [
      new ExpirationPlugin({
        maxEntries: 30,
        maxAgeSeconds: 10 * 60 // 10 minutes
      })
    ]
  })
);
```

---

## Environment-Specific Service Workers

### Development vs Production

```typescript
// src/service-worker.ts
import { dev } from '$app/environment';

if (dev) {
  console.log('Service worker running in development mode');
  // Minimal caching in development
} else {
  console.log('Service worker running in production mode');
  // Aggressive caching in production
}
```

### Environment Variables

```typescript
// src/service-worker.ts
const API_URL = import.meta.env.VITE_API_URL || 'https://api.example.com';

registerRoute(
  ({ url }) => url.origin === API_URL,
  new NetworkFirst({
    cacheName: 'api-cache'
  })
);
```

```bash
# .env.production
VITE_API_URL=https://api.production.com

# .env.development
VITE_API_URL=http://localhost:3000
```

---

## Testing SvelteKit PWAs

### Unit Testing Service Worker Logic

```typescript
// tests/service-worker.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';

describe('Service Worker', () => {
  beforeEach(() => {
    // Mock global caches
    global.caches = {
      open: vi.fn().mockResolvedValue({
        addAll: vi.fn().mockResolvedValue(undefined),
        match: vi.fn().mockResolvedValue(undefined)
      }),
      keys: vi.fn().mockResolvedValue([]),
      delete: vi.fn().mockResolvedValue(true)
    } as any;
  });

  it('should cache assets on install', async () => {
    // Test service worker install logic
  });

  it('should serve from cache when offline', async () => {
    // Test fetch event handling
  });
});
```

### Integration Testing with Playwright

```typescript
// tests/pwa.spec.ts
import { test, expect } from '@playwright/test';

test('service worker registers successfully', async ({ page, context }) => {
  await page.goto('/');

  // Wait for service worker to register
  await page.waitForFunction(() => {
    return navigator.serviceWorker.controller !== null;
  });

  const swState = await page.evaluate(async () => {
    const registration = await navigator.serviceWorker.ready;
    return registration.active?.state;
  });

  expect(swState).toBe('activated');
});

test('app works offline', async ({ page, context }) => {
  await page.goto('/');

  // Wait for service worker
  await page.waitForFunction(() => navigator.serviceWorker.controller !== null);

  // Go offline
  await context.setOffline(true);

  // Navigate to cached page
  await page.goto('/about');

  // Should still load
  await expect(page.locator('h1')).toBeVisible();

  // Go back online
  await context.setOffline(false);
});

test('install prompt can be triggered', async ({ page }) => {
  await page.goto('/');

  // Trigger beforeinstallprompt event
  await page.evaluate(() => {
    window.dispatchEvent(new Event('beforeinstallprompt'));
  });

  // Check if install button appears
  await expect(page.locator('text=Install')).toBeVisible();
});
```

### Lighthouse PWA Audit

```bash
# Install Lighthouse
npm install -D @lhci/cli

# Run Lighthouse CI
npx lhci autorun --config=lighthouserc.json
```

```json
// lighthouserc.json
{
  "ci": {
    "collect": {
      "startServerCommand": "npm run preview",
      "url": ["http://localhost:4173/"],
      "numberOfRuns": 3
    },
    "assert": {
      "preset": "lighthouse:recommended",
      "assertions": {
        "installable-manifest": ["error", { "minScore": 1 }],
        "service-worker": ["error", { "minScore": 1 }],
        "splash-screen": ["error", { "minScore": 1 }],
        "themed-omnibox": ["error", { "minScore": 1 }],
        "viewport": ["error", { "minScore": 1 }],
        "categories:pwa": ["error", { "minScore": 0.9 }]
      }
    },
    "upload": {
      "target": "temporary-public-storage"
    }
  }
}
```

---

## Deployment Strategies

### Vercel (Auto Adapter)

```bash
# Install Vercel adapter
npm install -D @sveltejs/adapter-vercel
```

```typescript
// svelte.config.js
import adapter from '@sveltejs/adapter-vercel';

export default {
  kit: {
    adapter: adapter({
      runtime: 'nodejs18.x',
      split: true // Code splitting for optimal performance
    })
  }
};
```

**Deploy**:
```bash
npm install -g vercel
vercel
```

**vercel.json** (optional):
```json
{
  "headers": [
    {
      "source": "/service-worker.js",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "public, max-age=0, must-revalidate"
        },
        {
          "key": "Service-Worker-Allowed",
          "value": "/"
        }
      ]
    },
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        }
      ]
    }
  ]
}
```

### Netlify (Static Adapter)

```typescript
// svelte.config.js
import adapter from '@sveltejs/adapter-static';

export default {
  kit: {
    adapter: adapter({
      pages: 'build',
      assets: 'build',
      fallback: 'index.html'
    })
  }
};
```

**netlify.toml**:
```toml
[build]
  command = "npm run build"
  publish = "build"

[[headers]]
  for = "/service-worker.js"
  [headers.values]
    Cache-Control = "public, max-age=0, must-revalidate"

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

### Cloudflare Pages (Static)

```typescript
// svelte.config.js
import adapter from '@sveltejs/adapter-cloudflare';

export default {
  kit: {
    adapter: adapter()
  }
};
```

**_headers** (in static folder):
```
/service-worker.js
  Cache-Control: public, max-age=0, must-revalidate

/*
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
```

**Deploy**:
```bash
npm run build
npx wrangler pages publish build
```

### Docker (Node Adapter)

```dockerfile
# Dockerfile
FROM node:18-alpine AS build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM node:18-alpine

WORKDIR /app

COPY --from=build /app/build ./build
COPY --from=build /app/package*.json ./

RUN npm ci --production

EXPOSE 3000

CMD ["node", "build"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - ORIGIN=https://example.com
```

---

## Complete SvelteKit PWA Example

### Project Structure

```
sveltekit-pwa/
├── src/
│   ├── routes/
│   │   ├── +layout.svelte       # Root layout with SW registration
│   │   ├── +page.svelte         # Home page
│   │   ├── offline/+page.svelte # Offline fallback
│   │   └── api/
│   │       └── posts/+server.ts # API route
│   ├── service-worker.ts        # Service worker
│   ├── app.html                 # HTML template
│   └── lib/
│       └── components/
│           └── InstallPrompt.svelte
├── static/
│   ├── manifest.json
│   ├── icons/
│   │   ├── icon-192.png
│   │   ├── icon-512.png
│   │   └── maskable-icon.png
│   ├── offline.html
│   └── favicon.png
├── svelte.config.js
├── vite.config.ts
└── package.json
```

### manifest.json

```json
{
  "name": "SvelteKit PWA",
  "short_name": "SK-PWA",
  "description": "A mobile-first Progressive Web App built with SvelteKit",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#007bff",
  "orientation": "any",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    },
    {
      "src": "/icons/maskable-icon.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ],
  "categories": ["productivity", "utilities"],
  "screenshots": [
    {
      "src": "/screenshots/mobile.png",
      "sizes": "540x720",
      "type": "image/png",
      "form_factor": "narrow"
    },
    {
      "src": "/screenshots/desktop.png",
      "sizes": "1280x720",
      "type": "image/png",
      "form_factor": "wide"
    }
  ]
}
```

### app.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <meta name="theme-color" content="#007bff" />
  <meta name="description" content="A mobile-first Progressive Web App" />

  <link rel="manifest" href="/manifest.json" />
  <link rel="icon" href="/favicon.png" />
  <link rel="apple-touch-icon" href="/icons/icon-192.png" />

  %sveltekit.head%
</head>
<body data-sveltekit-preload-data="hover">
  <div style="display: contents">%sveltekit.body%</div>
</body>
</html>
```

---

## Summary

SvelteKit provides excellent PWA support with:

1. **Built-in Service Worker**: `src/service-worker.ts` with `$service-worker` module
2. **Flexible Adapters**: Static, auto, Node, Cloudflare for different deployment scenarios
3. **SSR/CSR Flexibility**: Prerendering, client-side navigation, hydration
4. **Build-Time Precaching**: Access to `build` and `files` arrays for intelligent caching
5. **Dynamic Route Caching**: Workbox integration for API and dynamic content
6. **Environment Awareness**: `dev` flag for development-specific behavior
7. **Testing Support**: Playwright for E2E PWA testing, Lighthouse for audits
8. **Easy Deployment**: One-command deploy to Vercel, Netlify, Cloudflare

**Related Resources**:
- [pwa-fundamentals.md](pwa-fundamentals.md) - Core PWA concepts and patterns
- [performance-optimization.md](performance-optimization.md) - Core Web Vitals and optimization
- [SKILL.md](../SKILL.md) - Mobile-first PWA design principles
