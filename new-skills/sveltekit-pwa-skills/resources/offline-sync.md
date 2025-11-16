# Offline Data Sync - SvelteKit PWAs

**Background sync, IndexedDB, and offline state management**


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
  }
  
  async openDB() {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open('pwa-sync', 1);
      
      request.onerror = () => reject(request.error);
      request.onsuccess = () => resolve(request.result);
      
      request.onupgradeneeded = (event) => {
        const db = event.target.result;
        if (!db.objectStoreNames.contains('sync-queue')) {
          const store = db.createObjectStore('sync-queue', {
            keyPath: 'id',
            autoIncrement: true
          });
          store.createIndex('timestamp', 'timestamp');
        }
      };
    });
  }
}

export const syncManager = new SyncManager();
```

### Service Worker Background Sync Handler

```javascript
// In service-worker.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'data-sync') {
    event.waitUntil(syncData());
  }
});

async function syncData() {
  // Open IndexedDB and get queued requests
  const db = await openDB();
  const tx = db.transaction('sync-queue', 'readwrite');
  const requests = await tx.objectStore('sync-queue').getAll();
  
  const results = await Promise.allSettled(
    requests.map(async (request) => {
      const response = await fetch(request.url, {
        method: request.method,
        headers: request.headers,
        body: request.body
      });
      
      if (response.ok) {
        // Remove successful request from queue
        await tx.objectStore('sync-queue').delete(request.id);
      }
      
      return response;
    })
  );
  
  // Notify clients of sync completion
  const clients = await self.clients.matchAll();
  clients.forEach(client => {
    client.postMessage({
      type: 'SYNC_COMPLETE',
      successful: results.filter(r => r.status === 'fulfilled').length,
      failed: results.filter(r => r.status === 'rejected').length
    });
  });
}
```

## Data Persistence Strategies

### IndexedDB for Complex Data

```javascript
// lib/utils/indexed-db.js
class LocalDatabase {
  constructor(dbName = 'pwa-data', version = 1) {
    this.dbName = dbName;
    this.version = version;
    this.db = null;
  }
  
  async open() {
    if (this.db) return this.db;
    
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, this.version);
      
      request.onerror = () => reject(request.error);
      request.onsuccess = () => {
        this.db = request.result;
        resolve(this.db);
      };
      
      request.onupgradeneeded = (event) => {
        const db = event.target.result;
        
        // Create object stores
        if (!db.objectStoreNames.contains('users')) {
          const userStore = db.createObjectStore('users', {
            keyPath: 'id'
          });
          userStore.createIndex('email', 'email', { unique: true });
        }
        
        if (!db.objectStoreNames.contains('posts')) {
          const postStore = db.createObjectStore('posts', {
            keyPath: 'id'
          });
          postStore.createIndex('userId', 'userId');
          postStore.createIndex('timestamp', 'timestamp');
        }
      };
    });
  }
  
  async save(storeName, data) {
    const db = await this.open();
    const tx = db.transaction(storeName, 'readwrite');
    const store = tx.objectStore(storeName);
    
    if (Array.isArray(data)) {
      for (const item of data) {
        await store.put(item);
      }
    } else {
      await store.put(data);
    }
    
    return tx.complete;
  }
  
  async get(storeName, key) {
    const db = await this.open();
    const tx = db.transaction(storeName, 'readonly');
    return tx.objectStore(storeName).get(key);
  }
  
  async getAll(storeName, query) {
    const db = await this.open();
    const tx = db.transaction(storeName, 'readonly');
    const store = tx.objectStore(storeName);
    
    if (query?.index && query?.value) {
      const index = store.index(query.index);
      return index.getAll(query.value);
    }
    
    return store.getAll();
  }
  
  async delete(storeName, key) {
    const db = await this.open();
    const tx = db.transaction(storeName, 'readwrite');
    return tx.objectStore(storeName).delete(key);
  }
  
  async clear(storeName) {
    const db = await this.open();
    const tx = db.transaction(storeName, 'readwrite');
    return tx.objectStore(storeName).clear();
  }
}

export const db = new LocalDatabase();
```

### Store with Offline Sync

```javascript
// lib/stores/offline-store.js
import { writable, derived } from 'svelte/store';
import { db } from '$lib/utils/indexed-db';
import { syncManager } from '$lib/utils/background-sync';

export function createOfflineStore(name, fetcher) {
  const { subscribe, set, update } = writable({
    data: [],
    loading: true,
    error: null,
    lastSync: null,
    isOffline: !navigator.onLine
  });
  
  async function loadData() {
    update(s => ({ ...s, loading: true }));
    
    try {
      // Try network first
      const networkData = await fetcher();
      
      // Save to IndexedDB
      await db.save(name, networkData);
      
      set({
        data: networkData,
        loading: false,
        error: null,
        lastSync: new Date(),
        isOffline: false
      });
    } catch (error) {
      // Fall back to cached data
      const cachedData = await db.getAll(name);
      
      set({
        data: cachedData || [],
        loading: false,
        error: 'Using offline data',
        lastSync: null,
        isOffline: true
      });
    }
  }
  
  async function saveItem(item) {
    // Optimistic update
    update(s => ({
      ...s,
      data: [...s.data, { ...item, _pending: true }]
    }));
    
    try {
      // Try to save to server
      const response = await fetch(`/api/${name}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(item)
      });
      
      if (!response.ok) throw new Error('Save failed');
      
      const saved = await response.json();
      
      // Update with server response
      update(s => ({
        ...s,
        data: s.data.map(d => 
          d._pending && d.id === item.id ? saved : d
        )
      }));
    } catch (error) {
      // Queue for background sync
      await syncManager.queueRequest(
        new Request(`/api/${name}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(item)
        })
      );
    }
    
    // Always save to IndexedDB
    await db.save(name, item);
  }
  
  // Listen for online/offline events
  if (browser) {
    window.addEventListener('online', loadData);
    window.addEventListener('offline', () => {
      update(s => ({ ...s, isOffline: true }));
    });
  }
  
  return {
    subscribe,
    loadData,
    saveItem,
    refresh: loadData
  };
}
```

## Network Status Management

```svelte
<!-- lib/components/NetworkStatus.svelte -->
<script>
  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';
  
  const online = writable(true);
  const connection = writable(null);
  
  onMount(() => {
    // Initial status
    online.set(navigator.onLine);
    
    // Network status events
    window.addEventListener('online', () => online.set(true));
    window.addEventListener('offline', () => online.set(false));
    
    // Connection quality monitoring
    if ('connection' in navigator) {
      const conn = navigator.connection;
      
      connection.set({
        type: conn.effectiveType,
        downlink: conn.downlink,
        rtt: conn.rtt,
        saveData: conn.saveData
      });
      
      conn.addEventListener('change', () => {
        connection.set({
          type: conn.effectiveType,
          downlink: conn.downlink,
          rtt: conn.rtt,
          saveData: conn.saveData
        });
      });
    }
  });
</script>

{#if !$online}
  <div class="network-status offline">
    <span>📵 Offline - Some features may be limited</span>
  </div>
{:else if $connection?.saveData}
  <div class="network-status save-data">
    <span>📶 Data saver mode active</span>
  </div>
{:else if $connection?.type === 'slow-2g' || $connection?.type === '2g'}
  <div class="network-status slow">
    <span>🐌 Slow connection detected</span>
  </div>
{/if}

<style>
  .network-status {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    padding: 0.5rem;
    text-align: center;
    z-index: 1000;
    animation: slideDown 0.3s ease;
  }
  
  .offline {
    background: #ef4444;
    color: white;
  }
  
  .slow {
    background: #f59e0b;
    color: white;
  }
  
  .save-data {
    background: #3b82f6;
    color: white;
  }
  
  @keyframes slideDown {
    from {
      transform: translateY(-100%);
    }
    to {
      transform: translateY(0);
    }
  }
</style>
```

## Testing Offline Functionality

```javascript
// tests/offline.test.js
import { test, expect } from '@playwright/test';

test.describe('Offline functionality', () => {
  test('shows offline page when offline', async ({ page, context }) => {
    await page.goto('/');
    
    // Go offline
    await context.setOffline(true);
    
    // Try to navigate
    await page.goto('/about');
    
    // Should show offline page
    await expect(page.locator('h1')).toContainText('Offline');
  });
  
  test('caches visited pages', async ({ page, context }) => {
    // Visit pages while online
    await page.goto('/');
    await page.goto('/about');
    
    // Go offline
    await context.setOffline(true);
    
    // Should still access cached pages
    await page.goto('/about');
    await expect(page.locator('h1')).not.toContainText('Offline');
  });
  
  test('queues form submissions when offline', async ({ page, context }) => {
    await page.goto('/contact');
    
    // Go offline
    await context.setOffline(true);
    
    // Submit form
    await page.fill('input[name="name"]', 'Test User');
    await page.fill('input[name="email"]', 'test@example.com');
    await page.click('button[type="submit"]');
    
    // Check for queued message
    await expect(page.locator('.message')).toContainText('queued');
    
    // Go back online
    await context.setOffline(false);
    
    // Wait for sync
    await page.waitForTimeout(2000);
    
    // Check for success message
    await expect(page.locator('.message')).toContainText('sent');
  });
});
```

## Best Practices

1. **Cache strategically** - Don't cache everything, prioritize critical content
2. **Provide feedback** - Always indicate offline status and sync state
3. **Handle conflicts** - Plan for data conflicts when syncing
4. **Monitor storage** - Implement cache cleanup and quota management
5. **Test thoroughly** - Test offline scenarios on various devices and networks
6. **Progressive enhancement** - Ensure basic functionality without service workers

## References

- [Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [Cache API](https://developer.mozilla.org/en-US/docs/Web/API/Cache)
- [Background Sync API](https://developer.mozilla.org/en-US/docs/Web/API/Background_Synchronization_API)
- [IndexedDB API](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API)
