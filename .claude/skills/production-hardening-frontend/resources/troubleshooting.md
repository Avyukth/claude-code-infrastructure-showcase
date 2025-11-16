# Troubleshooting - SvelteKit Production Issues

**Debugging and resolving common production problems**

This resource provides diagnostic procedures and solutions for production issues in SvelteKit applications, covering security violations, performance regressions, accessibility failures, deployment problems, and runtime errors.

---

## 1. Content Security Policy (CSP) Violations

### Symptoms
- Console errors: `Refused to load...violates CSP directive`
- Resources failing to load (scripts, styles, images)
- Third-party integrations broken

### Diagnostic Steps

**Step 1: Check CSP Report Console**

```typescript
// src/routes/api/csp-violations/+server.ts
import type { RequestHandler } from './$types';

export const POST: RequestHandler = async ({ request }) => {
  const report = await request.json();

  console.error('CSP Violation:', {
    blockedUri: report['blocked-uri'],
    violatedDirective: report['violated-directive'],
    documentUri: report['document-uri'],
    sourceFile: report['source-file'],
    lineNumber: report['line-number']
  });

  return new Response(null, { status: 204 });
};
```

**Step 2: Enable Report-Only Mode**

```javascript
// svelte.config.js
export default {
  kit: {
    csp: {
      reportOnly: {
        'default-src': ['self'],
        'script-src': ['self', 'unsafe-inline'],
        'report-uri': ['/api/csp-violations']
      }
    }
  }
};
```

**Step 3: Identify Violating Resources**

Check browser console for:
```
Refused to load the script 'https://example.com/script.js'
because it violates the following CSP directive: "script-src 'self'"
```

### Common Solutions

**Issue 1: Inline Scripts Blocked**

```javascript
// ❌ Problem: Inline script without nonce
// src/app.html
<script>
  console.log('Analytics initialized');
</script>

// ✅ Solution: Use nonce
<script nonce="%sveltekit.nonce%">
  console.log('Analytics initialized');
</script>
```

**Issue 2: Third-Party CDN Resources**

```javascript
// svelte.config.js
export default {
  kit: {
    csp: {
      directives: {
        'script-src': [
          'self',
          'https://cdn.example.com',  // Add trusted CDN
          'https://analytics.google.com'
        ],
        'connect-src': [
          'self',
          'https://api.example.com'
        ]
      }
    }
  }
};
```

**Issue 3: Svelte Transitions Requiring `unsafe-inline`**

```javascript
// svelte.config.js
export default {
  kit: {
    csp: {
      directives: {
        'style-src': ['self', 'unsafe-inline']  // Required for transitions
      }
    }
  }
};
```

---

## 2. Performance Regressions

### Symptoms
- Lighthouse score dropped below 90
- LCP > 2.5s
- CLS > 0.1
- Bundle size increased significantly

### Diagnostic Steps

**Step 1: Run Lighthouse CI**

```bash
npm install -D @lhci/cli

# Run audit
npx lhci autorun --collect.url=http://localhost:4173
```

**Step 2: Analyze Bundle Size**

```bash
npm install -D rollup-plugin-visualizer

npm run build
# Open build/client/stats.html
```

**Step 3: Check Core Web Vitals**

```typescript
// src/routes/+layout.svelte
<script>
  import { onCLS, onFCP, onLCP, onTTFB, onINP } from 'web-vitals';

  if (import.meta.env.PROD) {
    onCLS(console.log);
    onFCP(console.log);
    onLCP(console.log);
    onTTFB(console.log);
    onINP(console.log);
  }
</script>
```

### Common Solutions

**Issue 1: Large Bundle Size**

```bash
# Check what's bloating the bundle
npx vite-bundle-visualizer

# Solution: Code splitting
```

```svelte
<!-- Before: Import directly -->
<script>
  import HeavyChart from '$lib/HeavyChart.svelte';
</script>

<!-- After: Lazy load -->
<script>
  let showChart = false;
</script>

{#if showChart}
  {#await import('$lib/HeavyChart.svelte') then Module}
    <Module.default />
  {/await}
{/if}
```

**Issue 2: Slow LCP (Images)**

```svelte
<!-- ❌ Problem: Large unoptimized image -->
<img src="/hero.jpg" alt="Hero" />

<!-- ✅ Solution: Optimized with @sveltejs/enhanced-img -->
<script>
  import { Image } from '@sveltejs/enhanced-img';
  import hero from '$lib/assets/hero.jpg?enhanced';
</script>

<Image
  src={hero}
  alt="Hero"
  loading="eager"
  fetchpriority="high"
/>
```

**Issue 3: Layout Shift (CLS > 0.1)**

```svelte
<!-- ❌ Problem: No dimensions specified -->
<img src="/image.jpg" alt="Content" />

<!-- ✅ Solution: Reserve space -->
<img
  src="/image.jpg"
  alt="Content"
  width="800"
  height="600"
/>

<!-- Or use aspect ratio -->
<div style="aspect-ratio: 16/9;">
  <img src="/image.jpg" alt="Content" />
</div>
```

---

## 3. Hydration Errors

### Symptoms
- Console warning: `Hydration failed because the initial UI does not match what was rendered on the server`
- Flash of incorrect content
- Interactive elements not working

### Diagnostic Steps

**Check Server/Client Mismatch**

```typescript
// Add debugging
// src/routes/+page.svelte
<script>
  import { browser } from '$app/environment';

  let value = browser ? 'client' : 'server';
  console.log('Rendered on:', value);
</script>
```

### Common Solutions

**Issue 1: Time-Dependent Rendering**

```svelte
<!-- ❌ Problem: Different server/client output -->
<script>
  const now = new Date();
</script>
<p>Time: {now.toLocaleTimeString()}</p>

<!-- ✅ Solution: Client-only rendering -->
<script>
  import { browser } from '$app/environment';
  import { onMount } from 'svelte';

  let now = new Date();

  onMount(() => {
    const interval = setInterval(() => {
      now = new Date();
    }, 1000);

    return () => clearInterval(interval);
  });
</script>

{#if browser}
  <p>Time: {now.toLocaleTimeString()}</p>
{/if}
```

**Issue 2: localStorage on Server**

```svelte
<!-- ❌ Problem: localStorage undefined on server -->
<script>
  const theme = localStorage.getItem('theme') || 'light';
</script>

<!-- ✅ Solution: Guard with browser check -->
<script>
  import { browser } from '$app/environment';

  const theme = browser
    ? localStorage.getItem('theme') || 'light'
    : 'light';
</script>
```

**Issue 3: Random Values**

```svelte
<!-- ❌ Problem: Different IDs server/client -->
<script>
  const id = Math.random().toString(36);
</script>

<!-- ✅ Solution: Generate on client only -->
<script>
  import { onMount } from 'svelte';

  let id = 'placeholder';

  onMount(() => {
    id = Math.random().toString(36);
  });
</script>
```

---

## 4. Authentication Issues

### Symptoms
- Users logged out unexpectedly
- Session cookies not persisting
- CSRF validation failures

### Diagnostic Steps

**Step 1: Check Cookie Settings**

```typescript
// src/routes/auth/login/+page.server.ts
export const actions = {
  default: async ({ cookies }) => {
    cookies.set('session', token, {
      httpOnly: true,
      secure: true,  // ⚠️ Must be true in production
      sameSite: 'strict',
      path: '/',
      maxAge: 60 * 60 * 24 * 7
    });
  }
};
```

**Step 2: Verify HTTPS in Production**

```bash
# Check if site is served over HTTPS
curl -I https://yourdomain.com

# Should return:
# Strict-Transport-Security: max-age=31536000
```

### Common Solutions

**Issue 1: Cookies Not Set (HTTPS Required)**

```typescript
// ❌ Problem: secure: true but using HTTP
cookies.set('session', token, {
  secure: true  // Fails on HTTP
});

// ✅ Solution: Check environment
import { dev } from '$app/environment';

cookies.set('session', token, {
  httpOnly: true,
  secure: !dev,  // false in dev, true in production
  sameSite: 'strict',
  path: '/'
});
```

**Issue 2: CSRF Validation Failing**

```typescript
// Check that CSRF is enabled
// svelte.config.js
export default {
  kit: {
    csrf: {
      checkOrigin: true  // Default: true
    }
  }
};

// In production, Origin header must match domain
// Check request headers in browser DevTools
```

**Issue 3: Session Expired**

```typescript
// src/hooks.server.ts
export const handle = async ({ event, resolve }) => {
  const sessionToken = event.cookies.get('session');

  if (sessionToken) {
    const session = await verifySession(sessionToken);

    if (!session) {
      // Session expired or invalid - clear cookie
      event.cookies.delete('session', { path: '/' });
    } else {
      event.locals.user = session.user;

      // Refresh session expiry (rolling sessions)
      event.cookies.set('session', sessionToken, {
        httpOnly: true,
        secure: true,
        sameSite: 'strict',
        path: '/',
        maxAge: 60 * 60 * 24 * 7  // Reset to 7 days
      });
    }
  }

  return await resolve(event);
};
```

---

## 5. Deployment Issues

### Adapter-Specific Problems

#### Vercel Deployment

**Issue: Build Fails with "Memory Limit Exceeded"**

```javascript
// vercel.json
{
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/node",
      "config": {
        "maxLambdaSize": "50mb"
      }
    }
  ]
}
```

Or increase memory in adapter config:

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-vercel';

export default {
  kit: {
    adapter: adapter({
      memory: 3008,  // MB (max 3008 for Pro)
      maxDuration: 60
    })
  }
};
```

**Issue: Environment Variables Not Available**

```bash
# Set via CLI
vercel env add DATABASE_URL production

# Or via vercel.com dashboard
# Project > Settings > Environment Variables
```

#### Netlify Deployment

**Issue: Functions Timing Out**

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-netlify';

export default {
  kit: {
    adapter: adapter({
      edge: true  // Use edge functions (faster)
    })
  }
};
```

**Issue: Redirects Not Working**

```toml
# netlify.toml
[[redirects]]
  from = "/old-path"
  to = "/new-path"
  status = 301

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

#### Cloudflare Pages

**Issue: Node.js Built-ins Not Available**

```typescript
// ❌ Problem: crypto module not available
import crypto from 'crypto';

// ✅ Solution: Use Web Crypto API
const hash = await crypto.subtle.digest(
  'SHA-256',
  new TextEncoder().encode(data)
);
```

**Issue: KV Namespace Not Found**

```typescript
// wrangler.toml
[[kv_namespaces]]
binding = "CACHE"
id = "your-namespace-id"
```

```typescript
// src/routes/+page.server.ts
export const load = async ({ platform }) => {
  const cache = platform?.env?.CACHE;

  if (!cache) {
    console.warn('KV namespace not found');
    return { data: null };
  }

  const data = await cache.get('key');
  return { data };
};
```

#### Docker Deployment

**Issue: Build Fails "ENOENT: no such file or directory"**

```dockerfile
# ❌ Problem: Missing dependencies
FROM node:20-alpine
COPY build ./build
CMD ["node", "build"]

# ✅ Solution: Copy node_modules
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
RUN npm prune --production

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/build ./build
COPY --from=builder /app/node_modules ./node_modules  # ← Important
COPY --from=builder /app/package.json ./
EXPOSE 3000
CMD ["node", "build"]
```

---

## 6. Accessibility Test Failures

### Symptoms
- axe DevTools reports violations
- Screen reader navigation broken
- Keyboard navigation not working

### Diagnostic Steps

**Step 1: Run Automated Tests**

```bash
npm install -D @axe-core/playwright

# Test with Playwright
npx playwright test --headed
```

```typescript
// tests/accessibility.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('should not have accessibility violations', async ({ page }) => {
  await page.goto('http://localhost:4173');

  const results = await new AxeBuilder({ page }).analyze();

  expect(results.violations).toEqual([]);
});
```

### Common Solutions

**Issue 1: Missing Form Labels**

```svelte
<!-- ❌ Problem -->
<input type="email" placeholder="Email" />

<!-- ✅ Solution -->
<label for="email">Email</label>
<input id="email" type="email" />

<!-- Or visually hidden label -->
<label for="email" class="sr-only">Email</label>
<input id="email" type="email" placeholder="Email" />
```

**Issue 2: Low Color Contrast**

```css
/* ❌ Problem: 2.5:1 (fails WCAG AA) */
.text {
  color: #999;
  background: #fff;
}

/* ✅ Solution: 4.5:1 (passes WCAG AA) */
.text {
  color: #666;
  background: #fff;
}

/* Check contrast at: https://webaim.org/resources/contrastchecker/ */
```

**Issue 3: Missing ARIA Labels**

```svelte
<!-- ❌ Problem: Button with icon only -->
<button on:click={close}>
  <XIcon />
</button>

<!-- ✅ Solution: Add aria-label -->
<button on:click={close} aria-label="Close dialog">
  <XIcon />
</button>
```

---

## 7. Error Monitoring Issues

### Sentry Not Capturing Errors

**Issue: No Errors Appearing in Sentry Dashboard**

```typescript
// Step 1: Verify DSN
// src/hooks.client.ts
import * as Sentry from '@sentry/sveltekit';

Sentry.init({
  dsn: 'https://example@sentry.io/123',  // ← Check this is correct
  environment: import.meta.env.MODE,
  tracesSampleRate: 1.0,  // Capture 100% in testing
  debug: true  // Enable debug logs
});
```

**Issue: Source Maps Not Uploading**

```bash
# Install Sentry CLI
npm install -D @sentry/vite-plugin

# Add to vite.config.js
import { sentryVitePlugin } from '@sentry/vite-plugin';

export default {
  plugins: [
    sveltekit(),
    sentryVitePlugin({
      org: 'your-org',
      project: 'your-project',
      authToken: process.env.SENTRY_AUTH_TOKEN
    })
  ],
  build: {
    sourcemap: true  // Required for source maps
  }
};
```

---

## 8. Service Worker Issues

### Symptoms
- Offline mode not working
- Stale content served
- Service worker not updating

### Diagnostic Steps

**Check Service Worker Status**

```javascript
// In browser console
navigator.serviceWorker.getRegistrations().then(registrations => {
  console.log('Active service workers:', registrations);
});
```

### Common Solutions

**Issue 1: Service Worker Not Updating**

```typescript
// src/service-worker.ts
import { version } from '$service-worker';

const CACHE = `cache-${version}`;  // ← Version changes trigger update

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(async (keys) => {
      // Delete old caches
      for (const key of keys) {
        if (key !== CACHE) {
          await caches.delete(key);
        }
      }
    })
  );
});
```

**Issue 2: Cached Assets Not Updating**

Force refresh in development:

```bash
# Chrome DevTools
# Application > Service Workers > Update on reload ✓
# Application > Service Workers > Bypass for network ✓
```

Programmatic update:

```typescript
// src/routes/+layout.svelte
<script>
  import { onMount } from 'svelte';

  onMount(() => {
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/service-worker.js');

      // Check for updates every hour
      setInterval(() => {
        navigator.serviceWorker.getRegistration().then(reg => {
          reg?.update();
        });
      }, 60 * 60 * 1000);
    }
  });
</script>
```

---

## 9. Database Connection Issues

**Issue: "Too Many Connections" Error**

```typescript
// ❌ Problem: Creating new Prisma client per request
import { PrismaClient } from '@prisma/client';

export const load = async () => {
  const prisma = new PrismaClient();  // Don't do this!
  const data = await prisma.user.findMany();
  return { data };
};

// ✅ Solution: Singleton pattern
// src/lib/server/db.ts
import { PrismaClient } from '@prisma/client';

const globalForPrisma = global as unknown as { prisma: PrismaClient };

export const prisma = globalForPrisma.prisma || new PrismaClient({
  log: ['query', 'error', 'warn']
});

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

---

## 10. Common Error Messages

### "Cannot access 'x' before initialization"

```typescript
// ❌ Problem: Circular dependency
// userService.ts
import { authService } from './authService';

// authService.ts
import { userService } from './userService';

// ✅ Solution: Extract shared logic
// shared.ts
export const validateToken = () => { /* ... */ };

// userService.ts
import { validateToken } from './shared';

// authService.ts
import { validateToken } from './shared';
```

### "window is not defined"

```svelte
<!-- ❌ Problem: Using window on server -->
<script>
  const width = window.innerWidth;
</script>

<!-- ✅ Solution: Check if browser -->
<script>
  import { browser } from '$app/environment';

  let width = browser ? window.innerWidth : 0;
</script>
```

### "Cannot read property of undefined"

```typescript
// ❌ Problem: No null checks
export const load = async ({ locals }) => {
  const userId = locals.user.id;  // locals.user might be null
};

// ✅ Solution: Optional chaining
export const load = async ({ locals }) => {
  const userId = locals.user?.id;

  if (!userId) {
    throw redirect(303, '/login');
  }

  return { userId };
};
```

---

## Debugging Tools

### Browser DevTools

**Network Tab**
- Check failed requests
- Inspect response headers
- Monitor API calls

**Console Tab**
- CSP violations
- JavaScript errors
- Hydration warnings

**Application Tab**
- Service worker status
- Cache storage
- Cookies and localStorage

**Lighthouse Tab**
- Performance audit
- Accessibility scan
- SEO analysis

### CLI Tools

```bash
# Bundle analysis
npx vite-bundle-visualizer

# Security audit
npm audit
npx snyk test

# Performance testing
npx @lhci/cli autorun

# TypeScript check
npm run check

# Accessibility testing
npx @axe-core/cli http://localhost:4173
```

---

**Related Resources:**
- [Security Hardening](./security-hardening.md)
- [Performance Optimization](./performance-optimization.md)
- [Deployment Operations](./deployment-operations.md)
- [Accessibility and UX](./accessibility-ux.md)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Maintenance**: Update with new common issues as they arise
