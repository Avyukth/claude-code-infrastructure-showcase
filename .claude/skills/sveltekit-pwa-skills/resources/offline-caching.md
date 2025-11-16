# Offline Capabilities for SvelteKit PWAs

## Overview

Implementing robust offline functionality is essential for PWAs. This guide covers caching strategies, offline fallbacks, background sync, and data persistence patterns.

## Caching Strategies

### Strategy Selection Matrix

| Content Type | Strategy | Rationale |
|-------------|----------|-----------|
| Static assets (JS, CSS) | Cache First | Rarely changes, performance critical |
| Images/Media | Cache First with expiry | Balance storage vs performance |
| API data | Network First | Fresh data preferred, cache fallback |
| User-generated content | Cache with Background Sync | Ensure data persistence |
| Third-party resources | Stale While Revalidate | Balance freshness and availability |

### Implementation with Workbox

```javascript
// src/service-worker.js with Workbox
import { precacheAndRoute, cleanupOutdatedCaches } from 'workbox-precaching';
import { registerRoute, NavigationRoute } from 'workbox-routing';
import { NetworkFirst, CacheFirst, StaleWhileRevalidate } from 'workbox-strategies';
import { CacheableResponsePlugin } from 'workbox-cacheable-response';
import { ExpirationPlugin } from 'workbox-expiration';
import { BackgroundSyncPlugin } from 'workbox-background-sync';

// Precache static assets
precacheAndRoute(self.__WB_MANIFEST);
cleanupOutdatedCaches();

// Cache First - Static assets
registerRoute(
  ({ request }) => 
    request.destination === 'script' ||
    request.destination === 'style' ||
    request.destination === 'font',
  new CacheFirst({
    cacheName: 'static-assets',
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      }),
      new ExpirationPlugin({
        maxAgeSeconds: 60 * 60 * 24 * 365, // 1 year
        maxEntries: 30,
        purgeOnQuotaError: true
      })
    ]
  })
);

// Cache First with Expiration - Images
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      }),
      new ExpirationPlugin({
        maxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
        maxEntries: 60,
        purgeOnQuotaError: true
      })
    ]
  })
);

// Network First - API calls
registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    networkTimeoutSeconds: 5,
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      }),
      new ExpirationPlugin({
        maxAgeSeconds: 60 * 60 * 24, // 1 day
        maxEntries: 50,
        purgeOnQuotaError: true
      })
    ]
  })
);

// Stale While Revalidate - Third-party resources
registerRoute(
  ({ url }) => url.origin !== self.location.origin,
  new StaleWhileRevalidate({
    cacheName: 'third-party',
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      }),
      new ExpirationPlugin({
        maxAgeSeconds: 60 * 60 * 24 * 7, // 1 week
        maxEntries: 30,
        purgeOnQuotaError: true
      })
    ]
  })
);

// Background sync for failed POST requests
const bgSyncPlugin = new BackgroundSyncPlugin('api-queue', {
  maxRetentionTime: 24 * 60 // Retry for up to 24 hours
});

registerRoute(
  ({ url }) => url.pathname.startsWith('/api/') && request.method === 'POST',
  new NetworkFirst({
    plugins: [bgSyncPlugin]
  }),
  'POST'
);
```

### Custom Service Worker Implementation

```javascript
// src/service-worker.js - Manual implementation
import { build, files, version } from '$service-worker';

const CACHE_VERSION = `v${version}`;
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const DYNAMIC_CACHE = `dynamic-${CACHE_VERSION}`;
const API_CACHE = `api-${CACHE_VERSION}`;

const STATIC_ASSETS = [...build, ...files];

// Install - cache static assets
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then((cache) => {
      return cache.addAll(STATIC_ASSETS);
    })
  );
  self.skipWaiting();
});

// Activate - cleanup old caches
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => {
            return name.startsWith('static-') ||
                   name.startsWith('dynamic-') ||
                   name.startsWith('api-');
          })
          .filter((name) => {
            return !name.includes(CACHE_VERSION);
          })
          .map((name) => caches.delete(name))
      );
    })
  );
  self.clients.claim();
});

// Fetch - implement caching strategies
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);
  
  // Skip non-GET requests
  if (request.method !== 'GET') {
    return;
  }
  
  // Handle different resource types
  if (STATIC_ASSETS.includes(url.pathname)) {
    // Cache First for static assets
    event.respondWith(cacheFirst(request, STATIC_CACHE));
  } else if (url.pathname.startsWith('/api/')) {
    // Network First for API calls
    event.respondWith(networkFirst(request, API_CACHE));
  } else if (request.destination === 'image') {
    // Cache First for images
    event.respondWith(cacheFirst(request, DYNAMIC_CACHE));
  } else {
    // Network First with offline fallback for pages
    event.respondWith(networkFirstWithOffline(request, DYNAMIC_CACHE));
  }
});

// Caching Strategies
async function cacheFirst(request, cacheName) {
  const cached = await caches.match(request);
  if (cached) return cached;
  
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(cacheName);
      cache.put(request, response.clone());
    }
    return response;
  } catch (error) {
    // Return offline fallback if available
    const offlinePage = await caches.match('/offline');
    return offlinePage || new Response('Offline', {
      status: 503,
      statusText: 'Service Unavailable'
    });
  }
}

async function networkFirst(request, cacheName) {
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(cacheName);
      cache.put(request, response.clone());
    }
    return response;
  } catch (error) {
    const cached = await caches.match(request);
    if (cached) return cached;
    
    return new Response(JSON.stringify({ 
      error: 'Offline',
      cached: false 
    }), {
      status: 503,
      headers: { 'Content-Type': 'application/json' }
    });
  }
}

async function networkFirstWithOffline(request, cacheName) {
  try {
    const response = await fetch(request);
    if (response.ok) {
      const cache = await caches.open(cacheName);
      cache.put(request, response.clone());
    }
    return response;
  } catch (error) {
    const cached = await caches.match(request);
    if (cached) return cached;
    
    // Return offline page for navigation requests
    if (request.mode === 'navigate') {
      return caches.match('/offline') || new Response('Offline');
    }
    
    return new Response('Network error', { status: 503 });
  }
}
```

## Offline Page Implementation

### Static Offline Page

```svelte
<!-- src/routes/offline/+page.svelte -->
<script>
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';
  
  let isOnline = false;
  let cachedPages = [];
  
  onMount(() => {
    if (!browser) return;
    
    // Check online status
    isOnline = navigator.onLine;
    
    // Listen for status changes
    window.addEventListener('online', () => {
      isOnline = true;
      // Optionally reload when back online
      setTimeout(() => window.location.reload(), 1000);
    });
    
    window.addEventListener('offline', () => {
      isOnline = false;
    });
    
    // List cached pages
    loadCachedPages();
  });
  
  async function loadCachedPages() {
    if (!browser || !('caches' in window)) return;
    
    const cacheNames = await caches.keys();
    const pages = [];
    
    for (const name of cacheNames) {
      const cache = await caches.open(name);
      const requests = await cache.keys();
      
      for (const request of requests) {
        const url = new URL(request.url);
        if (url.pathname.endsWith('.html') || 
            url.pathname.endsWith('/')) {
          pages.push({
            url: url.pathname,
            title: url.pathname.split('/').filter(Boolean).pop() || 'Home'
          });
        }
      }
    }
    
    cachedPages = [...new Set(pages.map(p => p.url))].map(url => 
      pages.find(p => p.url === url)
    );
  }
</script>

<div class="offline-container">
  <h1>You're Offline</h1>
  
  {#if isOnline}
    <p>Connection restored! Reloading...</p>
  {:else}
    <p>Please check your internet connection.</p>
    
    {#if cachedPages.length > 0}
      <h2>Available Offline Pages:</h2>
      <ul>
        {#each cachedPages as page}
          <li>
            <a href={page.url}>{page.title}</a>
          </li>
        {/each}
      </ul>
    {/if}
    
    <button on:click={() => window.location.reload()}>
      Try Again
    </button>
  {/if}
</div>

<style>
  .offline-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    padding: 2rem;
    text-align: center;
  }
</style>
```

### Prerender Offline Page

```javascript
// src/routes/offline/+page.js
export const prerender = true;
export const ssr = false; // Client-side only for cache access
```

## Background Sync

### Implementation with Background Sync API

```javascript
// lib/utils/background-sync.js
class SyncManager {
  constructor() {
    this.queue = [];
    this.isOnline = navigator.onLine;
    
    window.addEventListener('online', () => this.sync());
    window.addEventListener('offline', () => this.isOnline = false);
    
    // Register background sync
    if ('serviceWorker' in navigator && 'SyncManager' in window) {
      this.registerBackgroundSync();
    }
  }
  
  async registerBackgroundSync() {
    const registration = await navigator.serviceWorker.ready;
    
    try {
      await registration.sync.register('data-sync');
    } catch (error) {
      console.error('Background sync registration failed:', error);
    }
  }
  
  async queueRequest(request) {
    const serialized = {
      url: request.url,
      method: request.method,
      headers: Object.fromEntries(request.headers.entries()),
      body: await request.text(),
      timestamp: Date.now()
    };
    
    // Save to IndexedDB
    await this.saveToIndexedDB(serialized);
    
    // Try immediate sync if online
    if (this.isOnline) {
      this.sync();
    }
  }
  
  async saveToIndexedDB(request) {
    const db = await this.openDB();
    const tx = db.transaction('sync-queue', 'readwrite');
    await tx.objectStore('sync-queue').add(request);
  }
  
  async sync() {
    const db = await this.openDB();
    const tx = db.transaction('sync-queue', 'readwrite');
    const store = tx.objectStore('sync-queue');
    const requests = await store.getAll();
    
    for (const request of requests) {
      try {
        const response = await fetch(request.url, {
          method: request.method,
          headers: request.headers,
          body: request.body
        });
        
        if (response.ok) {
          // Remove from queue
          await store.delete(request.id);
        }
      } catch (error) {
        console.error('Sync failed for request:', error);
      }
    }
