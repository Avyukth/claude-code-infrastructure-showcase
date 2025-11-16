# Debugging SvelteKit PWAs

**Debugging tools, techniques, and troubleshooting**

## Debugging Techniques

### Service Worker Debugging

```javascript
// lib/utils/sw-debug.js
export class ServiceWorkerDebugger {
  constructor() {
    this.logs = [];
    this.maxLogs = 100;
  }
  
  log(type, message, data = {}) {
    const entry = {
      type,
      message,
      data,
      timestamp: new Date().toISOString()
    };
    
    this.logs.push(entry);
    
    if (this.logs.length > this.maxLogs) {
      this.logs.shift();
    }
    
    // Send to clients
    this.broadcast({
      type: 'SW_DEBUG',
      payload: entry
    });
    
    // Console log in dev
    if (process.env.NODE_ENV === 'development') {
      console.log(`[SW ${type}]`, message, data);
    }
  }
  
  async broadcast(message) {
    const clients = await self.clients.matchAll();
    clients.forEach(client => client.postMessage(message));
  }
  
  async getLogs() {
    return this.logs;
  }
  
  clear() {
    this.logs = [];
  }
  
  // Debug helpers
  async cacheStatus() {
    const cacheNames = await caches.keys();
    const status = {};
    
    for (const name of cacheNames) {
      const cache = await caches.open(name);
      const keys = await cache.keys();
      status[name] = {
        count: keys.length,
        urls: keys.map(k => k.url)
      };
    }
    
    return status;
  }
  
  async networkStatus() {
    return {
      online: self.navigator.onLine,
      connection: self.navigator.connection
    };
  }
}

// Usage in service worker
const debug = new ServiceWorkerDebugger();

self.addEventListener('install', (event) => {
  debug.log('install', 'Service worker installing');
});

self.addEventListener('fetch', (event) => {
  debug.log('fetch', `Fetching: ${event.request.url}`, {
    method: event.request.method,
    mode: event.request.mode,
    cache: event.request.cache
  });
});
```

### Chrome DevTools for PWAs

```javascript
// lib/utils/devtools-pwa.js
export function initPWADevTools() {
  if (!import.meta.env.DEV) return;
  
  // Service Worker status
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.ready.then(registration => {
      console.group('Service Worker Status');
      console.log('Scope:', registration.scope);
      console.log('Active:', registration.active?.state);
      console.log('Waiting:', registration.waiting?.state);
      console.log('Installing:', registration.installing?.state);
      console.groupEnd();
    });
  }
  
  // Cache status
  if ('caches' in window) {
    caches.keys().then(async names => {
      console.group('Cache Storage');
      for (const name of names) {
        const cache = await caches.open(name);
        const keys = await cache.keys();
        console.log(`${name}: ${keys.length} entries`);
      }
      console.groupEnd();
    });
  }
  
  // PWA features
  console.group('PWA Features');
  console.log('Manifest:', document.querySelector('link[rel="manifest"]')?.href);
  console.log('Notification Permission:', Notification.permission);
  console.log('Push Support:', 'PushManager' in window);
  console.log('Background Sync:', 'SyncManager' in window);
  console.log('Share API:', 'share' in navigator);
  console.groupEnd();
}
```

### Remote Debugging

```javascript
// lib/utils/remote-debug.js
export class RemoteDebugger {
  constructor(endpoint) {
    this.endpoint = endpoint;
    this.buffer = [];
    this.maxBuffer = 50;
  }
  
  log(level, message, data = {}) {
    const entry = {
      level,
      message,
      data,
      timestamp: Date.now(),
      url: window.location.href,
      userAgent: navigator.userAgent
    };
    
    this.buffer.push(entry);
    
    if (this.buffer.length >= this.maxBuffer) {
      this.flush();
    }
  }
  
  async flush() {
    if (this.buffer.length === 0) return;
    
    const logs = [...this.buffer];
    this.buffer = [];
    
    try {
      await fetch(this.endpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ logs })
      });
    } catch (error) {
      // Re-add to buffer if failed
      this.buffer.unshift(...logs);
    }
  }
  
  captureError(error) {
    this.log('error', error.message, {
      stack: error.stack,
      name: error.name
    });
    this.flush();
  }
}

// Initialize
const debug = new RemoteDebugger('/api/debug');

// Capture errors
window.addEventListener('error', (e) => {
  debug.captureError(e.error);
});

window.addEventListener('unhandledrejection', (e) => {
  debug.captureError(new Error(e.reason));
});
```

## Testing Checklist

### Unit Tests
- [ ] Components render correctly
- [ ] Props are handled properly
- [ ] Events are fired and handled
- [ ] Store actions work as expected
- [ ] Utility functions return correct values
- [ ] Error cases are handled

### Integration Tests
- [ ] API endpoints return correct data
- [ ] Authentication works correctly
- [ ] Database operations succeed
- [ ] Form submissions work
- [ ] Navigation works properly

### E2E Tests
- [ ] User flows work end-to-end
- [ ] PWA installs correctly
- [ ] Offline mode works
- [ ] Push notifications work
- [ ] Mobile experience is smooth
- [ ] Performance meets targets

### PWA-Specific Tests
- [ ] Manifest is valid
- [ ] Service worker registers
- [ ] Caching strategies work
- [ ] Offline fallback works
- [ ] Update flow works
- [ ] Icons are correct

## Best Practices

1. **Test behavior, not implementation** - Focus on what users experience
2. **Use proper test isolation** - Each test should be independent
3. **Mock external dependencies** - Don't rely on network/APIs in unit tests
4. **Test edge cases** - Empty states, errors, boundaries
5. **Maintain test coverage** - Aim for 80%+ coverage
6. **Use meaningful test names** - Describe what is being tested
7. **Keep tests fast** - Parallelize when possible
8. **Test on real devices** - Emulators miss real-world issues
9. **Automate regression testing** - Prevent breaking changes
10. **Document test utilities** - Make tests maintainable

## References

- [Vitest Documentation](https://vitest.dev/)
- [Playwright Documentation](https://playwright.dev/)
- [Testing Library](https://testing-library.com/docs/svelte-testing-library/intro/)
- [Chrome DevTools PWA](https://developer.chrome.com/docs/devtools/progressive-web-apps/)
