---
name: production-hardening-frontend
description: Enterprise-grade production hardening for SvelteKit frontend applications. Use when securing SvelteKit apps with CSP/XSS prevention, optimizing Core Web Vitals (LCP < 2.5s), ensuring WCAG 2.1 AA accessibility, deploying to Vercel/Netlify/Cloudflare, implementing service workers, or hardening browser-level security. Covers SvelteKit security patterns, performance budgets, PWA, client-side monitoring, production deployment strategies, security headers, bundle optimization, and OWASP Top 10 mitigations.
---

# Production Hardening - SvelteKit Frontend Applications

**Enterprise-grade hardening for SvelteKit frontend with browser-level security**

This skill guideline provides comprehensive production hardening practices for SvelteKit applications, emphasizing security, performance, accessibility, and operational excellence. Drawing from OWASP guidelines, browser security standards, and performance best practices, these patterns ensure production-ready frontend applications.

---

## Core Frontend Hardening Principles

### 1. **Security First** 🔒
Client-side security with CSP, XSS prevention, and secure authentication

### 2. **Performance by Default** ⚡
Optimized bundles, lazy loading, and efficient rendering strategies

### 3. **Progressive Enhancement** 📱
Works without JavaScript, enhanced with it

### 4. **Accessibility** ♿
WCAG 2.1 AA compliance, keyboard navigation, screen reader support

### 5. **Resilient Error Handling** 🛡️
Graceful degradation, fallback UI, client-side monitoring

### 6. **Zero Trust Client** 🔐
Never trust client-side data, validate everything server-side

### 7. **Minimal Bundle Size** 📦
Tree shaking, code splitting, dynamic imports

### 8. **Observability** 📊
Client-side monitoring, error tracking, performance metrics

### 9. **Secure by Default** 🎯
Security headers, SRI, HTTPS enforcement

### 10. **Fast User Experience** 🚀
< 3s LCP, < 100ms FID, < 0.1 CLS

---

## Quick Reference: SvelteKit Production Checklist

### Pre-Deployment Security

- [ ] **CSP Headers**: Strict Content Security Policy configured
- [ ] **Security Headers**: X-Frame-Options, X-Content-Type-Options, HSTS
- [ ] **XSS Prevention**: All user input sanitized with DOMPurify
- [ ] **Dependency Audit**: Run `npm audit` and fix vulnerabilities
- [ ] **SRI Hashes**: Subresource Integrity for CDN resources
- [ ] **Secure Auth**: HttpOnly cookies, short-lived tokens
- [ ] **HTTPS Only**: Force HTTPS in production
- [ ] **Environment Secrets**: No secrets in client bundle
- [ ] **CORS Policy**: Strict origin validation
- [ ] **Input Validation**: Client AND server-side validation

### Performance Optimization

- [ ] **Code Splitting**: Dynamic imports for routes
- [ ] **Image Optimization**: WebP/AVIF with fallbacks
- [ ] **Prerendering**: Static pages prerendered at build time
- [ ] **Bundle Analysis**: Check bundle size < 200KB initial
- [ ] **Font Loading**: Font-display: swap, preload critical fonts
- [ ] **Lazy Loading**: Below-fold images lazy loaded
- [ ] **Service Worker**: Offline support with Workbox
- [ ] **Compression**: Brotli/Gzip enabled
- [ ] **Tree Shaking**: Unused code eliminated
- [ ] **Critical CSS**: Above-fold styles inlined

### Runtime Resilience

- [ ] **Error Boundaries**: Global error handler configured
- [ ] **Error Monitoring**: Sentry or similar integrated
- [ ] **Fallback UI**: Loading states and error states
- [ ] **Offline Support**: Service worker for offline functionality
- [ ] **Rate Limiting**: Client-side throttling for API calls
- [ ] **Graceful Degradation**: Works without JavaScript
- [ ] **Progressive Web App**: Manifest and app icons
- [ ] **Analytics**: Privacy-respecting analytics configured
- [ ] **Performance Monitoring**: Core Web Vitals tracked
- [ ] **A11y Testing**: Automated accessibility tests passing

---

## Resource Files

This skill is organized into focused resource files:

### 1. [Security Hardening](resources/security-hardening.md)
- Content Security Policy (CSP) configuration
- XSS and injection attack prevention
- Secure authentication patterns
- Dependency vulnerability management
- Security headers (HSTS, X-Frame-Options, etc.)
- API security (CORS, CSRF protection)
- Secure data storage (avoiding localStorage for sensitive data)

### 2. [Performance Optimization](resources/performance-optimization.md)
- Code splitting strategies
- Image optimization (WebP, lazy loading)
- SSR vs CSR vs prerendering trade-offs
- Bundle size optimization
- Vite configuration for production
- Font loading strategies
- Critical rendering path optimization
- Service worker caching strategies

### 3. [Deployment and Operations](resources/deployment-operations.md)
- Adapter configuration (Vercel, Netlify, Node)
- Environment variable management
- CI/CD pipeline setup
- Build optimization
- CDN configuration
- Monitoring and logging
- Rollback strategies
- Blue-green deployments

### 4. [Accessibility and UX](resources/accessibility-ux.md)
- WCAG 2.1 AA compliance
- Semantic HTML and ARIA
- Keyboard navigation
- Screen reader support
- Focus management
- Color contrast and readability
- Responsive design patterns
- Progressive enhancement

---

## SvelteKit-Specific Hardening Advantages

### Compile-Time Optimizations ✅
- **Reactive by default**: Minimal runtime overhead
- **No virtual DOM**: Direct DOM manipulation
- **Tree shaking**: Unused code eliminated automatically
- **Scoped CSS**: No class name collisions

### Server-Side Rendering ✅
- **SEO friendly**: Content rendered on server
- **Fast initial load**: HTML delivered immediately
- **Progressive enhancement**: Works without JS
- **Secure by default**: Sensitive logic stays server-side

### Type Safety ✅
- **TypeScript integration**: Type-safe at compile time
- **Form actions**: Type-safe server actions
- **Load functions**: Type-safe data loading

---

## OWASP Top 10 for Frontend

### A01: Broken Access Control

```svelte
<!-- ❌ BAD: Client-side only authorization -->
<script>
  import { user } from '$lib/stores/user';
</script>

{#if $user.role === 'admin'}
  <button>Delete User</button>
{/if}

<!-- ✅ GOOD: Server-side authorization -->
<script>
  // src/routes/admin/+page.server.ts
  export const load = async ({ locals }) => {
    if (!locals.user || locals.user.role !== 'admin') {
      throw redirect(302, '/login');
    }
    return {};
  };
</script>
```

### A02: Cryptographic Failures

```typescript
// ❌ BAD: Storing tokens in localStorage
localStorage.setItem('token', authToken);

// ✅ GOOD: HttpOnly cookies (set server-side)
// src/routes/login/+page.server.ts
export const actions = {
  default: async ({ request, cookies }) => {
    const data = await request.formData();
    const token = await authenticateUser(data);

    cookies.set('session', token, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      maxAge: 60 * 60 * 24 * 7, // 1 week
      path: '/'
    });
  }
};
```

### A03: Injection (XSS)

```svelte
<script>
  import DOMPurify from 'isomorphic-dompurify';

  export let userContent: string;

  // ❌ BAD: Direct HTML rendering
  // {@html userContent}

  // ✅ GOOD: Sanitize before rendering
  $: sanitized = DOMPurify.sanitize(userContent, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p'],
    ALLOWED_ATTR: []
  });
</script>

{@html sanitized}
```

---

## Production Hardening Workflow

### 1. Development Phase
```
Security Linting → Type Checking → Component Testing → Accessibility Audit
```

### 2. Build Phase
```
Bundle Analysis → Tree Shaking → Minification → SRI Hash Generation
```

### 3. Pre-Deployment Phase
```
Security Scan → Performance Audit → A11y Testing → E2E Tests
```

### 4. Deployment Phase
```
Staged Rollout → Canary Testing → Monitor Metrics → Rollback Ready
```

### 5. Production Phase
```
Error Monitoring → Performance Tracking → Security Audits → User Feedback
```

---

## Essential Packages for Production

### Security
- **isomorphic-dompurify** - XSS prevention
- **helmet** - Security headers (for SvelteKit node adapter)
- **@sentry/sveltekit** - Error monitoring
- **csrf** - CSRF token generation

### Performance
- **@sveltejs/adapter-auto** - Auto-detect deployment platform
- **vite-plugin-compression** - Gzip/Brotli compression
- **vite-imagetools** - Image optimization
- **workbox-precaching** - Service worker caching

### Development
- **eslint-plugin-security** - Security linting
- **@axe-core/playwright** - Accessibility testing
- **lighthouse** - Performance auditing
- **svelte-check** - Type checking

---

## Core Web Vitals Targets

| Metric | Target | Description |
|--------|--------|-------------|
| **LCP** | < 2.5s | Largest Contentful Paint |
| **FID** | < 100ms | First Input Delay |
| **CLS** | < 0.1 | Cumulative Layout Shift |
| **TTFB** | < 600ms | Time to First Byte |
| **FCP** | < 1.8s | First Contentful Paint |

---

## Security Headers Configuration

```typescript
// src/hooks.server.ts
export const handle = async ({ event, resolve }) => {
  const response = await resolve(event);

  // Content Security Policy
  response.headers.set(
    'Content-Security-Policy',
    [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline'",  // Svelte needs inline
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self'",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'"
    ].join('; ')
  );

  // Security Headers
  response.headers.set('X-Frame-Options', 'DENY');
  response.headers.set('X-Content-Type-Options', 'nosniff');
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
  response.headers.set('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');

  // HSTS (only in production with HTTPS)
  if (import.meta.env.PROD) {
    response.headers.set(
      'Strict-Transport-Security',
      'max-age=31536000; includeSubDomains; preload'
    );
  }

  return response;
};
```

---

## Common Frontend Anti-Patterns

### ❌ DON'T: Store Sensitive Data Client-Side

```typescript
// ❌ BAD
localStorage.setItem('apiKey', 'sk_live_...');
localStorage.setItem('password', userPassword);

// ✅ GOOD: Keep sensitive data server-side only
// Use HttpOnly cookies for session management
```

### ❌ DON'T: Trust Client-Side Validation

```svelte
<!-- ❌ BAD: Client-side only -->
<script>
  function handleSubmit(data) {
    if (data.email.includes('@')) {
      submitToServer(data);  // Can be bypassed!
    }
  }
</script>

<!-- ✅ GOOD: Client AND server validation -->
<script>
  // Client-side for UX
  function validateEmail(email) {
    return email.includes('@');
  }
</script>

<!-- Server-side for security -->
<!-- src/routes/api/submit/+server.ts -->
<script lang="ts">
  export const POST = async ({ request }) => {
    const data = await request.formData();
    const email = data.get('email');

    // Always validate server-side
    if (!email || !validateEmail(email)) {
      throw error(400, 'Invalid email');
    }

    // Process...
  };
</script>
```

### ❌ DON'T: Expose API Keys in Client Bundle

```typescript
// ❌ BAD: API key in client code
const apiKey = 'pk_live_abc123';  // Visible in bundle!

fetch(`/api/data?key=${apiKey}`);

// ✅ GOOD: Use environment variables and server-side proxy
// .env (not committed)
PRIVATE_API_KEY=sk_live_abc123

// src/routes/api/proxy/+server.ts
import { PRIVATE_API_KEY } from '$env/static/private';

export const GET = async () => {
  const response = await fetch('https://api.example.com/data', {
    headers: { 'Authorization': `Bearer ${PRIVATE_API_KEY}` }
  });
  return response;
};
```

---

## Performance Budget

Set and enforce performance budgets:

```javascript
// vite.config.js
export default {
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          // Vendor bundle budget: < 150KB
          if (id.includes('node_modules')) {
            return 'vendor';
          }
        }
      }
    },
    // Warn if bundle exceeds limits
    chunkSizeWarningLimit: 200 // KB
  }
};
```

---

## Getting Started

### 1. Security Audit (10 minutes)

```bash
# Check for vulnerabilities
npm audit

# Fix automatically
npm audit fix

# Security linting
npm install -D eslint-plugin-security
```

### 2. Configure CSP (15 minutes)

```typescript
// src/hooks.server.ts
// Add security headers (see example above)
```

### 3. Enable Error Monitoring (20 minutes)

```bash
# Install Sentry
npm install @sentry/sveltekit

# Configure in hooks.client.ts and hooks.server.ts
```

### 4. Optimize Performance (30 minutes)

```bash
# Analyze bundle
npm run build
npx vite-bundle-visualizer

# Add image optimization
npm install -D vite-imagetools
```

---

## Compliance and Standards

This skill aligns with:
- ✅ **OWASP Top 10** (Web application security)
- ✅ **WCAG 2.1 AA** (Accessibility)
- ✅ **Core Web Vitals** (Performance)
- ✅ **CSP Level 3** (Content Security Policy)
- ✅ **PCI DSS** (Payment card industry - if applicable)
- ✅ **GDPR** (Privacy and data protection)

---

**Related Skills:**
- [Rust Backend Development](../../SKILL.md)
- [Production Hardening - Backend](../../production-hardening/SKILL.md)

---

**Version**: 1.0
**Last Updated**: 2025-11-15
**Status**: Production-Ready
**Compliance**: OWASP, WCAG 2.1, Core Web Vitals
