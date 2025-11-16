---
name: sveltekit-pwa-guidelines
description: Comprehensive guidelines for building mobile-friendly Progressive Web Apps with SvelteKit, including offline support, push notifications, and native-like features
---

# SvelteKit PWA Development Guidelines

## Purpose

This skill provides production-ready patterns for building Progressive Web Apps with SvelteKit that work seamlessly across devices, with emphasis on mobile experience, offline capabilities, and native-like features.

## When to Use This Skill

**Auto-activation scenarios:**
- Creating new SvelteKit PWA projects
- Converting existing SvelteKit apps to PWAs
- Implementing offline capabilities
- Adding push notifications
- Optimizing for mobile devices
- Implementing installable web apps

**File patterns:**
- `src/service-worker.{js,ts}`
- `static/manifest.json`
- `**/*.svelte` (when implementing PWA features)
- `vite.config.{js,ts}` (PWA plugin configuration)

## Core Principles

### 1. Progressive Enhancement First
- Start with core HTML/CSS functionality
- Layer JavaScript enhancements
- Add PWA features as capabilities allow
- Ensure graceful degradation

### 2. Mobile-First Development
- Design for touch interfaces
- Optimize for constrained networks
- Consider battery and data usage
- Test on real devices

### 3. Offline-First Architecture
- Cache critical resources
- Implement background sync
- Provide meaningful offline experiences
- Handle network failures gracefully

### 4. Performance Obsession
- Minimize JavaScript bundles
- Leverage Svelte's compile-time optimizations
- Implement code splitting
- Use service worker caching strategically

## Quick Reference

### PWA Setup Checklist
```bash
# 1. Install PWA plugin
npm i -D @vite-pwa/sveltekit

# 2. Configure vite.config.js
# 3. Create manifest.json
# 4. Implement service worker
# 5. Add meta tags to app.html
# 6. Test with Lighthouse
```

### Essential Configuration

**vite.config.js:**
```javascript
import { sveltekit } from '@sveltejs/kit/vite';
import { SvelteKitPWA } from '@vite-pwa/sveltekit';

export default {
  plugins: [
    sveltekit(),
    SvelteKitPWA({
      registerType: 'autoUpdate',
      manifest: {
        name: 'App Name',
        short_name: 'App',
        theme_color: '#ffffff',
        display: 'standalone',
        icons: [/* ... */]
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,ico,png,svg,webp}']
      }
    })
  ]
};
```

### Service Worker Pattern
```javascript
// src/service-worker.js
import { build, files, version } from '$service-worker';

const CACHE = `cache-${version}`;
const ASSETS = [...build, ...files];

// Install and cache
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE)
      .then(cache => cache.addAll(ASSETS))
  );
});

// Activate and cleanup
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys => {
      return Promise.all(
        keys.filter(key => key !== CACHE)
          .map(key => caches.delete(key))
      );
    })
  );
});
```

## Architecture Patterns

### Component Structure
```
src/
├── lib/
│   ├── components/
│   │   ├── InstallPrompt.svelte     # PWA install UI
│   │   ├── OfflineIndicator.svelte  # Network status
│   │   └── UpdatePrompt.svelte      # SW update UI
│   ├── stores/
│   │   ├── network.js               # Online/offline state
│   │   ├── installation.js          # PWA install state
│   │   └── notifications.js         # Push notification state
│   └── utils/
│       ├── push-notifications.js    # Web Push utilities
│       └── storage.js               # IndexedDB/localStorage
├── routes/
│   └── +layout.svelte               # PWA initialization
├── service-worker.js                # Service worker logic
└── app.html                         # Meta tags, manifest link
```

### State Management for PWAs
```javascript
// lib/stores/network.js
import { writable, derived } from 'svelte/store';

function createNetworkStore() {
  const { subscribe, set } = writable(navigator.onLine);
  
  if (browser) {
    window.addEventListener('online', () => set(true));
    window.addEventListener('offline', () => set(false));
  }
  
  return { subscribe };
}

export const isOnline = createNetworkStore();
export const networkStatus = derived(
  isOnline,
  $isOnline => $isOnline ? 'online' : 'offline'
);
```

## Resource Files

### Setup & Configuration
- [pwa-setup.md](resources/pwa-setup.md) - Complete PWA setup with manifest, icons, and service worker
- [deployment-platforms.md](resources/deployment-platforms.md) - Adapter configuration for Vercel, Netlify, Cloudflare
- [deployment-cicd.md](resources/deployment-cicd.md) - CI/CD pipelines, monitoring, and operations

### Core PWA Features
- [offline-caching.md](resources/offline-caching.md) - Caching strategies and Workbox patterns
- [offline-sync.md](resources/offline-sync.md) - Background sync and IndexedDB integration
- [push-notifications-setup.md](resources/push-notifications-setup.md) - Push notification server setup and client implementation
- [push-notifications-advanced.md](resources/push-notifications-advanced.md) - Advanced patterns and service worker handlers

### Mobile Optimization
- [mobile-ux.md](resources/mobile-ux.md) - Touch gestures, viewport configuration, and responsive design
- [mobile-performance.md](resources/mobile-performance.md) - Mobile-specific performance optimizations
- [performance-optimization.md](resources/performance-optimization.md) - Build-time optimization and code splitting
- [performance-monitoring.md](resources/performance-monitoring.md) - Web Vitals and runtime performance monitoring

### Development Patterns
- [component-patterns.md](resources/component-patterns.md) - Component organization and composition patterns
- [state-management.md](resources/state-management.md) - Reactive state management for PWA features
- [accessibility-fundamentals.md](resources/accessibility-fundamentals.md) - ARIA patterns, keyboard navigation, screen readers
- [accessibility-testing.md](resources/accessibility-testing.md) - Automated and manual accessibility testing

### Testing & Operations
- [testing-pwa.md](resources/testing-pwa.md) - Unit, integration, and E2E testing strategies
- [debugging-pwa.md](resources/debugging-pwa.md) - Debugging tools and techniques
- [troubleshooting.md](resources/troubleshooting.md) - Common issues and solutions

## Common Patterns

### Install Prompt Component
```svelte
<!-- lib/components/InstallPrompt.svelte -->
<script>
  import { onMount } from 'svelte';
  
  let deferredPrompt;
  let showInstallPrompt = false;
  
  onMount(() => {
    window.addEventListener('beforeinstallprompt', (e) => {
      e.preventDefault();
      deferredPrompt = e;
      showInstallPrompt = true;
    });
  });
  
  async function install() {
    deferredPrompt.prompt();
    const { outcome } = await deferredPrompt.userChoice;
    console.log(`Install prompt outcome: ${outcome}`);
    showInstallPrompt = false;
    deferredPrompt = null;
  }
</script>

{#if showInstallPrompt}
  <div class="install-prompt">
    <button on:click={install}>Install App</button>
    <button on:click={() => showInstallPrompt = false}>Later</button>
  </div>
{/if}
```

### Update Notification
```svelte
<!-- lib/components/UpdatePrompt.svelte -->
<script>
  import { useRegisterSW } from 'virtual:pwa-register/svelte';
  
  const { needRefresh, updateServiceWorker } = useRegisterSW({
    onRegistered(r) {
      console.log('SW Registered:', r);
    }
  });
</script>

{#if $needRefresh}
  <div class="update-prompt">
    <p>New version available!</p>
    <button on:click={() => updateServiceWorker(true)}>
      Update
    </button>
  </div>
{/if}
```

## Anti-Patterns to Avoid

### ❌ DON'T: Cache Everything
```javascript
// Bad: Caches too much, wastes storage
cache.addAll([...allPossibleAssets]);
```

### ✅ DO: Cache Strategically
```javascript
// Good: Cache critical assets only
cache.addAll([...criticalAssets]);
// Use runtime caching for other resources
```

### ❌ DON'T: Ignore Network Status
```javascript
// Bad: Assumes always online
const data = await fetch('/api/data');
```

### ✅ DO: Handle Offline Gracefully
```javascript
// Good: Provides fallback
try {
  const data = await fetch('/api/data');
} catch (error) {
  const cached = await getFromCache('/api/data');
  if (cached) return cached;
  throw new Error('Offline: No cached data');
}
```

## Production Checklist

- [ ] Manifest.json configured with all required fields
- [ ] Service worker implements caching strategy
- [ ] Icons provided in multiple sizes (192px, 512px minimum)
- [ ] Meta tags added to app.html
- [ ] Install prompt implemented
- [ ] Update notification implemented
- [ ] Offline fallback page created
- [ ] Push notifications configured (if needed)
- [ ] Lighthouse PWA audit passing
- [ ] Tested on real mobile devices
- [ ] HTTPS configured for production
- [ ] Background sync implemented (if needed)

## Integration with SvelteKit Features

### SSR/SSG Compatibility
- Service workers only run client-side
- Use `browser` check for PWA features
- Prerender static pages for offline access
- Consider hydration with cached data

### Form Actions with Offline Support
```javascript
// +page.server.js
export const actions = {
  submit: async ({ request }) => {
    // Server-side handling
  }
};

// +page.svelte
import { enhance } from '$app/forms';
import { isOnline } from '$lib/stores/network';

function handleOfflineSubmit(form) {
  if (!$isOnline) {
    // Queue for background sync
    queueFormData(form);
    return { cancel: true };
  }
}
```

## Related Skills
- `backend-dev-guidelines` - For API design supporting PWAs
- `performance-optimization` - General performance patterns
- `accessibility-guidelines` - Comprehensive accessibility patterns

## References
- [Official SvelteKit Docs - Service Workers](https://kit.svelte.dev/docs/service-workers)
- [Vite PWA Plugin for SvelteKit](https://vite-pwa-org.netlify.app/frameworks/sveltekit)
- [MDN - Progressive Web Apps](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps)
- [web.dev PWA Guidelines](https://web.dev/progressive-web-apps/)
