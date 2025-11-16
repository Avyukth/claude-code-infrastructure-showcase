# Troubleshooting SvelteKit PWAs

## Overview

Common issues and solutions for SvelteKit PWA development, covering service workers, caching, push notifications, and deployment problems.

## Service Worker Issues

### Service Worker Not Registering

**Symptom**: Console shows "Service worker registration failed"

**Solutions**:

1. **Check HTTPS requirement**
```javascript
// Service workers require HTTPS (except localhost)
if (location.protocol !== 'https:' && location.hostname !== 'localhost') {
  console.error('Service workers require HTTPS');
}
```

2. **Verify service worker path**
```javascript
// Service worker must be at root or above registered scope
navigator.serviceWorker.register('/service-worker.js', {
  scope: '/'  // Must be at or above this path
});
```

3. **Check for syntax errors**
```bash
# Build and check for errors
npm run build
```

### Service Worker Not Updating

**Symptom**: Changes to service worker not reflected in browser

**Solutions**:

1. **Force update in DevTools**
- Open Chrome DevTools → Application → Service Workers
- Check "Update on reload"
- Click "Unregister" then refresh

2. **Clear cache and hard refresh**
```bash
# Mac: Cmd + Shift + R
# Windows: Ctrl + Shift + R
```

3. **Implement update detection**
```javascript
navigator.serviceWorker.ready.then(registration => {
  registration.addEventListener('updatefound', () => {
    const newWorker = registration.installing;
    newWorker.addEventListener('statechange', () => {
      if (newWorker.state === 'installed' && navigator.serviceWorker.controller) {
        // New version available
        showUpdateNotification();
      }
    });
  });
});
```

## Cache Problems

### Stale Content Served

**Symptom**: Old version of site showing despite updates

**Solutions**:

1. **Version your cache**
```javascript
const CACHE_VERSION = 'v2';
const CACHE_NAME = `pwa-cache-${CACHE_VERSION}`;

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(keys => {
      return Promise.all(
        keys.filter(key => key !== CACHE_NAME)
          .map(key => caches.delete(key))
      );
    })
  );
});
```

2. **Implement cache busting**
```javascript
// Add timestamp or hash to requests
fetch(`/api/data?v=${Date.now()}`);
```

### Cache Quota Exceeded

**Symptom**: "QuotaExceededError" in console

**Solutions**:

1. **Implement cache limits**
```javascript
import { ExpirationPlugin } from 'workbox-expiration';

registerRoute(
  ({request}) => request.destination === 'image',
  new CacheFirst({
    plugins: [
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
        purgeOnQuotaError: true
      })
    ]
  })
);
```

2. **Check available storage**
```javascript
if ('storage' in navigator && 'estimate' in navigator.storage) {
  navigator.storage.estimate().then(estimate => {
    const percentUsed = (estimate.usage / estimate.quota) * 100;
    console.log(`Using ${percentUsed}% of storage`);
  });
}
```

## Push Notification Failures

### Notifications Not Showing

**Symptom**: Push events received but no notification displayed

**Solutions**:

1. **Check permission status**
```javascript
if (Notification.permission !== 'granted') {
  const permission = await Notification.requestPermission();
  if (permission !== 'granted') {
    console.error('Notification permission denied');
  }
}
```

2. **Ensure notification shown in push event**
```javascript
self.addEventListener('push', (event) => {
  const data = event.data.json();

  // MUST show notification in push event
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon-192.png'
    })
  );
});
```

### Subscription Fails

**Symptom**: `pushManager.subscribe()` rejects

**Solutions**:

1. **Check VAPID keys**
```javascript
// Ensure keys are correct format (base64)
const subscription = await registration.pushManager.subscribe({
  userVisibleOnly: true,
  applicationServerKey: urlBase64ToUint8Array(PUBLIC_VAPID_KEY)
});
```

2. **Verify service worker is active**
```javascript
const registration = await navigator.serviceWorker.ready;
// Now safe to subscribe
```

## Offline Functionality Issues

### Offline Page Not Showing

**Symptom**: Generic browser offline page instead of custom one

**Solutions**:

1. **Precache offline page**
```javascript
const OFFLINE_URL = '/offline';

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => {
      return cache.addAll([OFFLINE_URL]);
    })
  );
});
```

2. **Serve offline page on fetch failure**
```javascript
self.addEventListener('fetch', (event) => {
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => {
        return caches.match('/offline');
      })
    );
  }
});
```

### Background Sync Not Working

**Symptom**: Failed requests not retried when back online

**Solutions**:

1. **Check browser support**
```javascript
if ('SyncManager' in window) {
  // Background sync supported
  await registration.sync.register('sync-tag');
} else {
  // Fallback: manual retry
  window.addEventListener('online', retryFailedRequests);
}
```

2. **Handle sync event**
```javascript
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-tag') {
    event.waitUntil(syncData());
  }
});
```

## Build and Deployment Issues

### Build Fails with PWA Plugin

**Symptom**: Vite build error with @vite-pwa/sveltekit

**Solutions**:

1. **Check plugin configuration**
```javascript
// vite.config.js
import { SvelteKitPWA } from '@vite-pwa/sveltekit';

export default {
  plugins: [
    sveltekit(),
    SvelteKitPWA({
      registerType: 'autoUpdate',
      manifest: {
        name: 'App Name',
        short_name: 'App'
      }
    })
  ]
};
```

2. **Update dependencies**
```bash
npm update @vite-pwa/sveltekit vite
```

### Manifest.json Not Found

**Symptom**: 404 error for manifest.json

**Solutions**:

1. **Verify manifest location**
```html
<!-- In app.html or +layout.svelte -->
<link rel="manifest" href="/manifest.json">
```

2. **Add to static folder**
```bash
# Ensure manifest.json is in static/ directory
ls static/manifest.json
```

## Performance Issues

### Large Bundle Size

**Symptom**: Initial load is slow, bundle > 500KB

**Solutions**:

1. **Analyze bundle**
```bash
ANALYZE=true npm run build
```

2. **Implement code splitting**
```javascript
// Use dynamic imports
const HeavyComponent = () => import('./HeavyComponent.svelte');
```

3. **Remove unused dependencies**
```bash
# Check bundle composition
npx vite-bundle-visualizer
```

### Slow Lighthouse Scores

**Symptom**: PWA audit scores < 90

**Solutions**:

1. **Run Lighthouse audit**
```bash
npx lighthouse https://your-site.com --view
```

2. **Common fixes**:
- Add meta viewport tag
- Provide maskable icons
- Ensure service worker caches start_url
- Set theme_color in manifest

## Mobile-Specific Issues

### PWA Not Installable on iOS

**Symptom**: No install prompt on iPhone/iPad

**Solutions**:

1. **Add Apple meta tags**
```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<link rel="apple-touch-icon" href="/icon-192.png">
```

2. **Check requirements**
- HTTPS required
- Valid manifest.json
- Service worker registered

### Touch Events Not Working

**Symptom**: Swipes/gestures not detected on mobile

**Solutions**:

1. **Add touch event listeners**
```javascript
element.addEventListener('touchstart', handleTouch, { passive: false });
element.addEventListener('touchmove', handleTouch, { passive: false });
element.addEventListener('touchend', handleTouch, { passive: false });
```

2. **Prevent default carefully**
```javascript
function handleTouch(e) {
  // Only prevent if needed
  if (shouldPreventDefault) {
    e.preventDefault();
  }
}
```

## Debugging Tools

### Chrome DevTools

1. **Application Panel**
- View service workers
- Inspect cache storage
- Check manifest
- Test offline mode

2. **Network Panel**
- Filter by service worker
- Check cache hits/misses

3. **Console**
```javascript
// Check service worker status
navigator.serviceWorker.getRegistration().then(reg => {
  console.log('SW:', reg?.active?.state);
});

// Check cache contents
caches.keys().then(keys => console.log('Caches:', keys));
```

### Lighthouse CI

```yaml
# .github/workflows/lighthouse.yml
- name: Run Lighthouse
  uses: treosh/lighthouse-ci-action@v9
  with:
    urls: |
      https://your-site.com
    uploadArtifacts: true
```

## Common Errors and Fixes

| Error | Cause | Solution |
|-------|-------|----------|
| DOMException: Registration failed | Invalid SW path | Move SW to root or adjust scope |
| TypeError: Failed to fetch | Network error + no cache | Add offline fallback |
| QuotaExceededError | Cache too large | Implement cache expiration |
| SecurityError: The operation is insecure | HTTP (not HTTPS) | Deploy to HTTPS |
| NotAllowedError | User denied permission | Handle gracefully, don't re-prompt |

## Best Practices for Debugging

1. **Enable verbose logging in development**
```javascript
if (import.meta.env.DEV) {
  self.addEventListener('fetch', (event) => {
    console.log('Fetching:', event.request.url);
  });
}
```

2. **Use Chrome's DevTools → Application → Service Workers**
- "Update on reload" during development
- "Bypass for network" to test without SW

3. **Test offline mode**
```javascript
// Simulate offline in tests
await page.context().setOffline(true);
```

4. **Monitor production errors**
```javascript
// Send to error tracking
self.addEventListener('error', (event) => {
  fetch('/api/log-error', {
    method: 'POST',
    body: JSON.stringify({
      message: event.message,
      stack: event.error?.stack
    })
  });
});
```

## References

- [Service Worker Debugging](https://developer.chrome.com/docs/workbox/troubleshooting-and-logging/)
- [PWA Checklist](https://web.dev/pwa-checklist/)
- [Common PWA Issues](https://github.com/GoogleChrome/workbox/issues)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Lines**: ~450
