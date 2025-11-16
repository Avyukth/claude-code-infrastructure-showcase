# PWA Fundamentals - Progressive Web App Implementation

Complete guide to building installable, offline-capable Progressive Web Apps for mobile-first experiences.

## Table of Contents

- [What Makes a Progressive Web App](#what-makes-a-progressive-web-app)
- [Web App Manifest Configuration](#web-app-manifest-configuration)
- [Service Workers Fundamentals](#service-workers-fundamentals)
- [Caching Strategies](#caching-strategies)
- [Offline Functionality Patterns](#offline-functionality-patterns)
- [Install Prompts and Add to Home Screen](#install-prompts-and-add-to-home-screen)
- [Push Notifications](#push-notifications)
- [Background Sync](#background-sync)
- [Testing PWA Features](#testing-pwa-features)
- [Complete SvelteKit PWA Example](#complete-sveltekit-pwa-example)

---

## What Makes a Progressive Web App

### Core Characteristics

A Progressive Web App is a web application that uses modern web capabilities to deliver an app-like experience:

**Essential Features:**
- ✅ **Installable** - Can be added to home screen
- ✅ **Offline-capable** - Works without network connection
- ✅ **Fast** - Responds quickly, even on slow networks
- ✅ **Responsive** - Works on any screen size
- ✅ **Discoverable** - Identifiable as an "application" by search engines
- ✅ **Re-engageable** - Push notifications keep users engaged
- ✅ **Safe** - Served over HTTPS

**Why PWA?**
- **54% of mobile sites** use service workers (2024)
- **No app store** distribution required
- **Instant updates** without user action
- **Lower development cost** than native apps
- **Cross-platform** - One codebase for all devices

### PWA Checklist (web.dev)

**Baseline Requirements:**
- [ ] Served over HTTPS
- [ ] Responsive on all devices
- [ ] Fast load time (LCP < 2.5s)
- [ ] Works offline or on flaky networks
- [ ] Can be installed to home screen
- [ ] Provides custom splash screen
- [ ] Sets custom theme color
- [ ] Content sized correctly for viewport

**Enhanced Features:**
- [ ] Push notifications
- [ ] Background sync
- [ ] Add to home screen prompt
- [ ] Offline fallback page
- [ ] Share target API
- [ ] Shortcuts in manifest

---

## Web App Manifest Configuration

### Basic Manifest

Create `static/manifest.json` (or `public/manifest.json`):

```json
{
  "name": "My Mobile PWA",
  "short_name": "PWA",
  "description": "A mobile-first Progressive Web App",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#6366f1",
  "orientation": "portrait-primary",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ]
}
```

### Manifest Properties Explained

**Display Modes:**
```json
{
  "display": "standalone"  // Recommended for mobile apps
}
```

Options:
- `"standalone"` - Removes browser UI, feels like native app (recommended)
- `"fullscreen"` - Fills entire screen, no browser UI
- `"minimal-ui"` - Minimal browser controls (back, reload)
- `"browser"` - Normal browser tab (not recommended for PWAs)

**Orientation:**
```json
{
  "orientation": "portrait-primary"  // Mobile apps
  // or "any" for tablets/desktops
  // or "landscape-primary" for games
}
```

**Theme Color:**
```json
{
  "theme_color": "#6366f1",        // Status bar color on Android
  "background_color": "#ffffff"    // Splash screen background
}
```

**Icons (Critical):**
```json
{
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"    // Supports Android adaptive icons
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ]
}
```

**Icon Requirements:**
- Minimum: 192×192 and 512×512 PNG
- Use `"purpose": "any maskable"` for Android adaptive icons
- Maskable icons safe area: 80% center circle
- Test with maskable.app

### Enhanced Manifest (2023+ Features)

```json
{
  "name": "My Mobile PWA",
  "short_name": "PWA",
  "description": "A comprehensive mobile PWA",
  "start_url": "/?source=pwa",
  "scope": "/",
  "display": "standalone",
  "display_override": ["window-controls-overlay", "standalone"],
  "background_color": "#ffffff",
  "theme_color": "#6366f1",
  "orientation": "portrait-primary",

  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icon-monochrome.svg",
      "sizes": "any",
      "type": "image/svg+xml",
      "purpose": "monochrome"
    }
  ],

  "screenshots": [
    {
      "src": "/screenshots/home-mobile.png",
      "sizes": "750x1334",
      "type": "image/png",
      "form_factor": "narrow",
      "label": "Home screen"
    },
    {
      "src": "/screenshots/dashboard-tablet.png",
      "sizes": "1536x2048",
      "type": "image/png",
      "form_factor": "wide",
      "label": "Dashboard view"
    }
  ],

  "shortcuts": [
    {
      "name": "New Task",
      "short_name": "New",
      "description": "Create a new task",
      "url": "/tasks/new?source=shortcut",
      "icons": [{ "src": "/icons/new-task.png", "sizes": "96x96" }]
    },
    {
      "name": "Dashboard",
      "short_name": "Dashboard",
      "description": "View your dashboard",
      "url": "/dashboard?source=shortcut",
      "icons": [{ "src": "/icons/dashboard.png", "sizes": "96x96" }]
    }
  ],

  "categories": ["productivity", "utilities"],
  "iarc_rating_id": "e84b072d-71b3-4d3e-86ae-31a8ce4e53b7",
  "related_applications": [],
  "prefer_related_applications": false
}
```

**New Features:**
- `screenshots` - Richer install prompts (added 2023)
- `shortcuts` - Quick actions from home screen icon
- `display_override` - Fallback display modes
- `form_factor` - Separate screenshots for narrow (mobile) vs wide (tablet)

### Linking Manifest in HTML

```html
<!-- src/app.html for SvelteKit -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <link rel="manifest" href="/manifest.json" />
    <link rel="icon" href="/favicon.png" />
    <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
    <meta name="theme-color" content="#6366f1" />
    <meta name="description" content="My mobile-first PWA" />
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
    <meta name="apple-mobile-web-app-title" content="PWA" />
    <link rel="apple-touch-icon" href="/icon-192.png" />
    %sveltekit.head%
  </head>
  <body data-sveltekit-preload-data="hover">
    <div style="display: contents">%sveltekit.body%</div>
  </body>
</html>
```

**iOS-Specific Meta Tags:**
- `apple-mobile-web-app-capable` - Enable standalone mode
- `apple-mobile-web-app-status-bar-style` - Status bar appearance
- `apple-touch-icon` - Home screen icon (iOS doesn't use manifest icons)

---

## Service Workers Fundamentals

### What is a Service Worker?

A service worker is a JavaScript file that runs in the background, separate from web pages, enabling:
- **Offline functionality** - Cache resources for offline access
- **Background sync** - Queue requests when offline, sync when online
- **Push notifications** - Re-engage users
- **Performance** - Cache-first strategies for instant loading

**Key Characteristics:**
- Runs on separate thread (doesn't block main thread)
- Can't directly access DOM
- Event-driven (install, activate, fetch, message)
- Requires HTTPS (except localhost)

### Basic Service Worker Lifecycle

```javascript
// service-worker.js

// 1. INSTALL - Cache critical resources
self.addEventListener('install', (event) => {
  console.log('[Service Worker] Installing...');

  event.waitUntil(
    caches.open('static-v1').then((cache) => {
      return cache.addAll([
        '/',
        '/offline.html',
        '/styles.css',
        '/app.js',
        '/icon-192.png'
      ]);
    })
  );

  // Force waiting service worker to become active
  self.skipWaiting();
});

// 2. ACTIVATE - Clean up old caches
self.addEventListener('activate', (event) => {
  console.log('[Service Worker] Activating...');

  const currentCaches = ['static-v1', 'dynamic-v1'];

  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((cacheName) => !currentCaches.includes(cacheName))
          .map((cacheName) => caches.delete(cacheName))
      );
    })
  );

  // Take control of all clients immediately
  self.clients.claim();
});

// 3. FETCH - Intercept network requests
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then((response) => {
      // Cache hit - return cached response
      if (response) {
        return response;
      }

      // Clone request for fetching
      const fetchRequest = event.request.clone();

      return fetch(fetchRequest).then((response) => {
        // Don't cache non-successful responses
        if (!response || response.status !== 200 || response.type !== 'basic') {
          return response;
        }

        // Clone response for caching
        const responseToCache = response.clone();

        caches.open('dynamic-v1').then((cache) => {
          cache.put(event.request, responseToCache);
        });

        return response;
      });
    }).catch(() => {
      // Network failed, return offline page
      return caches.match('/offline.html');
    })
  );
});
```

### Registering Service Worker

```javascript
// In main app or +layout.svelte
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker
      .register('/service-worker.js')
      .then((registration) => {
        console.log('SW registered:', registration.scope);

        // Check for updates every hour
        setInterval(() => {
          registration.update();
        }, 60 * 60 * 1000);
      })
      .catch((error) => {
        console.log('SW registration failed:', error);
      });
  });
}
```

---

## Caching Strategies

### 1. Cache First (Static Assets)

Best for: Images, fonts, CSS, JavaScript bundles

```javascript
// Cache First - Check cache, then network
self.addEventListener('fetch', (event) => {
  if (event.request.destination === 'image' ||
      event.request.destination === 'font') {
    event.respondWith(
      caches.match(event.request).then((cachedResponse) => {
        if (cachedResponse) {
          return cachedResponse;
        }

        return fetch(event.request).then((response) => {
          return caches.open('static-v1').then((cache) => {
            cache.put(event.request, response.clone());
            return response;
          });
        });
      })
    );
  }
});
```

**Pros:**
- ✅ Fastest response time
- ✅ Works offline immediately
- ✅ Reduces bandwidth usage

**Cons:**
- ❌ May serve stale content
- ❌ Requires manual cache updates

### 2. Network First (API Calls)

Best for: API requests, dynamic data

```javascript
// Network First - Try network, fallback to cache
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/api/')) {
    event.respondWith(
      fetch(event.request)
        .then((response) => {
          // Cache successful response
          return caches.open('api-v1').then((cache) => {
            cache.put(event.request, response.clone());
            return response;
          });
        })
        .catch(() => {
          // Network failed, return cache
          return caches.match(event.request);
        })
    );
  }
});
```

**With Timeout:**
```javascript
// Network First with 3s timeout
function networkFirstWithTimeout(request, timeout = 3000) {
  return new Promise((resolve, reject) => {
    const timeoutId = setTimeout(reject, timeout);

    fetch(request).then((response) => {
      clearTimeout(timeoutId);
      resolve(response);
    }, reject);
  }).catch(() => {
    return caches.match(request);
  });
}
```

**Pros:**
- ✅ Always tries to get fresh data
- ✅ Falls back to cache when offline

**Cons:**
- ❌ Slower when network is slow
- ❌ Requires network attempt even with cache

### 3. Stale-While-Revalidate (Best Balance)

Best for: CSS, JavaScript, frequently updated content

```javascript
// Stale-While-Revalidate - Return cache immediately, update in background
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.open('dynamic-v1').then((cache) => {
      return cache.match(event.request).then((cachedResponse) => {
        const fetchPromise = fetch(event.request).then((networkResponse) => {
          cache.put(event.request, networkResponse.clone());
          return networkResponse;
        });

        // Return cached version immediately, update cache in background
        return cachedResponse || fetchPromise;
      });
    })
  );
});
```

**Pros:**
- ✅ Fast response (returns cache immediately)
- ✅ Automatically updates cache
- ✅ Works offline

**Cons:**
- ❌ May briefly show stale content
- ❌ Double bandwidth (cache + network)

### 4. Network Only (Sensitive Data)

Best for: Authentication, payment, sensitive operations

```javascript
// Network Only - Never cache
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/auth/') ||
      event.request.url.includes('/payment/')) {
    event.respondWith(fetch(event.request));
  }
});
```

### 5. Cache Only (Precached Content)

Best for: App shell, critical resources

```javascript
// Cache Only - Serve only from cache
self.addEventListener('fetch', (event) => {
  if (event.request.url.includes('/app-shell/')) {
    event.respondWith(caches.match(event.request));
  }
});
```

---

## Offline Functionality Patterns

### Offline Page Fallback

```javascript
// service-worker.js
const OFFLINE_PAGE = '/offline.html';

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('offline-v1').then((cache) => {
      return cache.add(OFFLINE_PAGE);
    })
  );
});

self.addEventListener('fetch', (event) => {
  // Only handle navigation requests
  if (event.request.mode === 'navigate') {
    event.respondWith(
      fetch(event.request).catch(() => {
        return caches.match(OFFLINE_PAGE);
      })
    );
  }
});
```

**Offline Page (`static/offline.html`):**
```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Offline</title>
    <style>
      body {
        font-family: system-ui, sans-serif;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        min-height: 100vh;
        margin: 0;
        padding: 1rem;
        text-align: center;
        background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
        color: white;
      }
      h1 { margin: 0 0 1rem; }
      p { max-width: 400px; }
    </style>
  </head>
  <body>
    <h1>You're Offline</h1>
    <p>No internet connection found. This page requires an internet connection.</p>
    <button onclick="location.reload()">Try Again</button>
  </body>
</html>
```

### Offline-First Data Storage

```javascript
// lib/offline-storage.js
const DB_NAME = 'pwa-offline-db';
const STORE_NAME = 'pending-requests';

export async function savePendingRequest(request) {
  const db = await openDB();
  const tx = db.transaction(STORE_NAME, 'readwrite');
  await tx.store.add({
    url: request.url,
    method: request.method,
    headers: Object.fromEntries(request.headers),
    body: await request.text(),
    timestamp: Date.now()
  });
}

export async function getPendingRequests() {
  const db = await openDB();
  return db.getAll(STORE_NAME);
}

function openDB() {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(DB_NAME, 1);

    request.onerror = () => reject(request.error);
    request.onsuccess = () => resolve(request.result);

    request.onupgradeneeded = (event) => {
      const db = event.target.result;
      if (!db.objectStoreNames.contains(STORE_NAME)) {
        db.createObjectStore(STORE_NAME, { keyPath: 'id', autoIncrement: true });
      }
    };
  });
}
```

---

## Install Prompts and Add to Home Screen

### Capturing Install Prompt

```svelte
<!-- +layout.svelte -->
<script>
  import { onMount } from 'svelte';

  let deferredPrompt = null;
  let showInstallPrompt = false;

  onMount(() => {
    window.addEventListener('beforeinstallprompt', (e) => {
      // Prevent default install prompt
      e.preventDefault();

      // Store event for later
      deferredPrompt = e;

      // Show custom install UI
      showInstallPrompt = true;
    });

    // Track if app was installed
    window.addEventListener('appinstalled', () => {
      console.log('PWA installed');
      showInstallPrompt = false;
      deferredPrompt = null;
    });
  });

  async function handleInstall() {
    if (!deferredPrompt) return;

    // Show browser install prompt
    deferredPrompt.prompt();

    // Wait for user choice
    const { outcome } = await deferredPrompt.userChoice;

    console.log(`User ${outcome} the install prompt`);

    deferredPrompt = null;
    showInstallPrompt = false;
  }
</script>

{#if showInstallPrompt}
  <div class="install-banner">
    <p>Install this app for a better experience!</p>
    <button on:click={handleInstall}>Install</button>
    <button on:click={() => showInstallPrompt = false}>Not Now</button>
  </div>
{/if}

<style>
  .install-banner {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    background: #6366f1;
    color: white;
    padding: 1rem;
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 1rem;
    z-index: 1000;
  }

  button {
    padding: 0.5rem 1rem;
    border: 1px solid white;
    background: transparent;
    color: white;
    border-radius: 4px;
    cursor: pointer;
  }
</style>
```

---

## Push Notifications

### Server Push Setup

```javascript
// service-worker.js
self.addEventListener('push', (event) => {
  const data = event.data.json();

  const options = {
    body: data.body,
    icon: '/icon-192.png',
    badge: '/badge-72.png',
    vibrate: [200, 100, 200],
    data: {
      url: data.url
    },
    actions: [
      { action: 'open', title: 'Open' },
      { action: 'close', title: 'Close' }
    ]
  };

  event.waitUntil(
    self.registration.showNotification(data.title, options)
  );
});

self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'open' || !event.action) {
    event.waitUntil(
      clients.openWindow(event.notification.data.url)
    );
  }
});
```

### Client-Side Subscription

```javascript
// lib/push-notifications.js
export async function subscribeToPush() {
  if (!('serviceWorker' in navigator) || !('PushManager' in window)) {
    console.log('Push notifications not supported');
    return null;
  }

  const registration = await navigator.serviceWorker.ready;

  // Request notification permission
  const permission = await Notification.requestPermission();

  if (permission !== 'granted') {
    console.log('Notification permission denied');
    return null;
  }

  // Subscribe to push notifications
  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY)
  });

  // Send subscription to server
  await fetch('/api/push/subscribe', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(subscription)
  });

  return subscription;
}

function urlBase64ToUint8Array(base64String) {
  const padding = '='.repeat((4 - base64String.length % 4) % 4);
  const base64 = (base64String + padding)
    .replace(/-/g, '+')
    .replace(/_/g, '/');

  const rawData = window.atob(base64);
  const outputArray = new Uint8Array(rawData.length);

  for (let i = 0; i < rawData.length; ++i) {
    outputArray[i] = rawData.charCodeAt(i);
  }

  return outputArray;
}
```

---

## Background Sync

### Queuing Failed Requests

```javascript
// service-worker.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-posts') {
    event.waitUntil(syncPosts());
  }
});

async function syncPosts() {
  const db = await openDB();
  const pending = await db.getAll('pending-posts');

  for (const post of pending) {
    try {
      await fetch('/api/posts', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(post.data)
      });

      // Remove from pending queue
      await db.delete('pending-posts', post.id);
    } catch (error) {
      console.log('Sync failed, will retry:', error);
    }
  }
}
```

### Registering Background Sync

```javascript
// In component
async function submitPost(data) {
  try {
    await fetch('/api/posts', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
  } catch (error) {
    // Save to IndexedDB
    await savePendingPost(data);

    // Register sync
    const registration = await navigator.serviceWorker.ready;
    await registration.sync.register('sync-posts');

    console.log('Post will sync when online');
  }
}
```

---

## Testing PWA Features

### Lighthouse Audit

```bash
# Install Lighthouse
npm install -g lighthouse

# Run PWA audit
lighthouse https://your-app.com --view --preset=desktop

# Or mobile (default)
lighthouse https://your-app.com --view
```

**PWA Checklist:**
- [ ] Fast and reliable (PWA score 90+)
- [ ] Installable (manifest configured)
- [ ] PWA optimized (service worker registered)
- [ ] Accessible (WCAG compliance)
- [ ] Best practices (HTTPS, viewport)

### Chrome DevTools

**Application Tab:**
1. Manifest - Verify manifest.json properties
2. Service Workers - Test offline, update on reload
3. Storage - View caches, IndexedDB
4. Background Services - Test push, sync

**Test Offline:**
1. Open DevTools → Network tab
2. Select "Offline" from throttling dropdown
3. Reload page - should load from cache

### Real Device Testing

**Android:**
1. Chrome → Menu → Install app
2. Test installability and standalone mode
3. Verify theme color in status bar
4. Test offline functionality

**iOS:**
1. Safari → Share → Add to Home Screen
2. Note: iOS doesn't support manifest icons (use `apple-touch-icon`)
3. Test standalone mode

---

## Complete SvelteKit PWA Example

### Project Structure

```
my-pwa/
├── src/
│   ├── app.html
│   ├── service-worker.js
│   ├── routes/
│   │   ├── +layout.svelte
│   │   └── +page.svelte
│   └── lib/
├── static/
│   ├── manifest.json
│   ├── offline.html
│   ├── icon-192.png
│   └── icon-512.png
└── svelte.config.js
```

### service-worker.js

```javascript
/// <reference types="@sveltejs/kit" />
import { build, files, version } from '$service-worker';

const CACHE = `cache-${version}`;
const ASSETS = [...build, ...files];

// Install - precache assets
self.addEventListener('install', (event) => {
  async function addFilesToCache() {
    const cache = await caches.open(CACHE);
    await cache.addAll(ASSETS);
  }

  event.waitUntil(addFilesToCache());
  self.skipWaiting();
});

// Activate - clean old caches
self.addEventListener('activate', (event) => {
  async function deleteOldCaches() {
    for (const key of await caches.keys()) {
      if (key !== CACHE) await caches.delete(key);
    }
  }

  event.waitUntil(deleteOldCaches());
  self.clients.claim();
});

// Fetch - serve from cache, fallback to network
self.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return;

  async function respond() {
    const cache = await caches.open(CACHE);

    // Try cache first
    const cached = await cache.match(event.request);
    if (cached) return cached;

    // Try network
    try {
      const response = await fetch(event.request);
      if (response.status === 200) {
        cache.put(event.request, response.clone());
      }
      return response;
    } catch {
      // Offline fallback
      return cache.match('/offline.html');
    }
  }

  event.respondWith(respond());
});
```

### +layout.svelte

```svelte
<script>
  import { onMount } from 'svelte';

  let showInstall = false;
  let deferredPrompt;

  onMount(() => {
    // Install prompt
    window.addEventListener('beforeinstallprompt', (e) => {
      e.preventDefault();
      deferredPrompt = e;
      showInstall = true;
    });

    // Service worker updates
    if ('serviceWorker' in navigator) {
      navigator.serviceWorker.register('/service-worker.js');
    }
  });

  async function install() {
    deferredPrompt?.prompt();
    const { outcome } = await deferredPrompt?.userChoice ?? {};
    deferredPrompt = null;
    showInstall = false;
  }
</script>

{#if showInstall}
  <div class="install-prompt">
    <p>Install app for better experience</p>
    <button on:click={install}>Install</button>
  </div>
{/if}

<main>
  <slot />
</main>
```

---

## Related Files

- [SKILL.md](../SKILL.md) - Main mobile PWA design overview
- [responsive-design.md](responsive-design.md) - Mobile-first CSS patterns
- [performance-optimization.md](performance-optimization.md) - Core Web Vitals
- [sveltekit-pwa-patterns.md](sveltekit-pwa-patterns.md) - Framework integration

---

**Pattern Status**: Production-ready PWA implementation ✅
