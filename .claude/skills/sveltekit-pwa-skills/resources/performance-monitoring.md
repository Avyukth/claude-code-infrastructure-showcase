# Performance Monitoring - SvelteKit PWAs

**Web Vitals, bundle analysis, and runtime performance monitoring**


### Web Vitals Monitoring

```javascript
// lib/utils/web-vitals.js
import { onCLS, onFCP, onFID, onLCP, onTTFB } from 'web-vitals';

export function initWebVitals() {
  function sendToAnalytics({ name, value, delta, id }) {
    // Send to analytics service
    if ('sendBeacon' in navigator) {
      const data = JSON.stringify({
        metric: name,
        value: Math.round(name === 'CLS' ? value * 1000 : value),
        delta: Math.round(name === 'CLS' ? delta * 1000 : delta),
        id,
        url: window.location.href
      });
      
      navigator.sendBeacon('/api/analytics/vitals', data);
    }
    
    // Log to console in dev
    if (import.meta.env.DEV) {
      console.log(`${name}: ${value}ms (Δ ${delta}ms)`);
    }
  }
  
  onCLS(sendToAnalytics);
  onFCP(sendToAnalytics);
  onFID(sendToAnalytics);
  onLCP(sendToAnalytics);
  onTTFB(sendToAnalytics);
}

// Initialize in app.html or root layout
if (browser) {
  initWebVitals();
}
```

### Resource Hints

```svelte
<!-- +layout.svelte - Add resource hints -->
<script>
  import { page } from '$app/stores';
  
  // Prefetch likely next pages
  $: {
    if ($page.url.pathname === '/') {
      prefetchRoute('/about');
      prefetchRoute('/products');
    }
  }
  
  function prefetchRoute(href) {
    const link = document.createElement('link');
    link.rel = 'prefetch';
    link.href = href;
    document.head.appendChild(link);
  }
</script>

<svelte:head>
  <!-- DNS prefetch for external domains -->
  <link rel="dns-prefetch" href="https://api.example.com" />
  <link rel="dns-prefetch" href="https://cdn.example.com" />
  
  <!-- Preconnect for critical resources -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
  
  <!-- Preload critical resources -->
  <link 
    rel="preload" 
    as="font" 
    type="font/woff2" 
    href="/fonts/inter-var.woff2" 
    crossorigin
  />
</svelte:head>
```

### Memory Management

```javascript
// lib/utils/memory-management.js
class MemoryManager {
  constructor() {
    this.observers = new Set();
    this.cache = new Map();
    this.maxCacheSize = 50;
  }
  
  // Observer cleanup
  registerObserver(observer) {
    this.observers.add(observer);
  }
  
  cleanupObservers() {
    this.observers.forEach(observer => {
      if (observer.disconnect) {
        observer.disconnect();
      }
    });
    this.observers.clear();
  }
  
  // Cache management
  addToCache(key, value) {
    // LRU eviction
    if (this.cache.size >= this.maxCacheSize) {
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    this.cache.set(key, value);
  }
  
  clearCache() {
    this.cache.clear();
  }
  
  // Monitor memory usage
  getMemoryUsage() {
    if ('memory' in performance) {
      return {
        used: performance.memory.usedJSHeapSize,
        total: performance.memory.totalJSHeapSize,
        limit: performance.memory.jsHeapSizeLimit,
        percentage: (performance.memory.usedJSHeapSize / 
                    performance.memory.jsHeapSizeLimit) * 100
      };
    }
    return null;
  }
  
  // Auto cleanup when memory is high
  startMemoryMonitoring() {
    if (!('memory' in performance)) return;
    
    setInterval(() => {
      const usage = this.getMemoryUsage();
      if (usage && usage.percentage > 80) {
        console.warn('High memory usage:', usage.percentage + '%');
        this.clearCache();
        this.cleanupObservers();
        
        // Trigger garbage collection if available
        if (window.gc) {
          window.gc();
        }
      }
    }, 30000); // Check every 30 seconds
  }
}

export const memoryManager = new MemoryManager();
```

## Rendering Performance

### Virtual Scrolling

```svelte
<!-- lib/components/VirtualScroll.svelte -->
<script>
  export let items = [];
  export let itemHeight = 50;
  export let buffer = 5;
  
  let scrollTop = 0;
  let clientHeight = 0;
  
  $: visibleStart = Math.floor(scrollTop / itemHeight);
  $: visibleEnd = Math.ceil((scrollTop + clientHeight) / itemHeight);
  $: visibleItems = items.slice(
    Math.max(0, visibleStart - buffer),
    Math.min(items.length, visibleEnd + buffer)
  );
  $: offsetY = Math.max(0, visibleStart - buffer) * itemHeight;
</script>

<div 
  class="container"
  bind:scrollTop
  bind:clientHeight
>
  <div 
    class="spacer"
    style="height: {items.length * itemHeight}px"
  >
    <div 
      class="items"
      style="transform: translateY({offsetY}px)"
    >
      {#each visibleItems as item (item.id)}
        <div class="item" style="height: {itemHeight}px">
          <slot {item} />
        </div>
      {/each}
    </div>
  </div>
</div>

<style>
  .container {
    overflow-y: auto;
    height: 100%;
  }
  
  .spacer {
    position: relative;
  }
  
  .items {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
  }
</style>
```

### Debouncing and Throttling

```javascript
// lib/utils/performance.js
export function debounce(func, wait, immediate = false) {
  let timeout;
  
  return function executedFunction(...args) {
    const later = () => {
      timeout = null;
      if (!immediate) func(...args);
    };
    
    const callNow = immediate && !timeout;
    clearTimeout(timeout);
    timeout = setTimeout(later, wait);
    
    if (callNow) func(...args);
  };
}

export function throttle(func, limit) {
  let inThrottle;
  
  return function(...args) {
    if (!inThrottle) {
      func.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// Usage in component
import { throttle } from '$lib/utils/performance';

const handleScroll = throttle(() => {
  // Handle scroll event
}, 16); // ~60fps
```

## Bundle Analysis and Optimization

### Bundle Size Monitoring

```json
// package.json
{
  "scripts": {
    "build": "vite build",
    "analyze": "ANALYZE=true vite build",
    "size": "size-limit",
    "test:size": "npm run build && size-limit"
  },
  "size-limit": [
    {
      "path": "build/**/*.js",
      "limit": "200 KB",
      "gzip": true
    },
    {
      "path": "build/**/*.css",
      "limit": "50 KB"
    }
  ]
}
```

### Tree Shaking Optimization

```javascript
// lib/utils/index.js - Named exports for better tree shaking
export { formatDate } from './date';
export { formatCurrency } from './currency';
export { debounce, throttle } from './performance';

// Bad: Default export of object
// export default { formatDate, formatCurrency, debounce, throttle };

// Usage - specific imports
import { formatDate } from '$lib/utils';
// Not: import utils from '$lib/utils';
```

## Performance Checklist

- [ ] Bundle size < 200KB (gzipped)
- [ ] First Contentful Paint < 1.8s
- [ ] Largest Contentful Paint < 2.5s
- [ ] Time to Interactive < 3.8s
- [ ] Cumulative Layout Shift < 0.1
- [ ] First Input Delay < 100ms
- [ ] Images optimized (WebP/AVIF)
- [ ] Critical CSS inlined
- [ ] JavaScript code split
- [ ] Service Worker caching implemented
- [ ] Resource hints added
- [ ] Web fonts optimized
- [ ] Third-party scripts deferred
- [ ] Render-blocking resources eliminated
- [ ] Compression enabled (Brotli/Gzip)

## References

- [Web Vitals](https://web.dev/vitals/)
- [Chrome DevTools Performance](https://developer.chrome.com/docs/devtools/performance/)
- [Vite Performance Guide](https://vitejs.dev/guide/performance.html)
- [SvelteKit Performance](https://kit.svelte.dev/docs/performance)
