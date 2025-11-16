# Performance Optimization - Mobile-First PWA Design

**Related**: [SKILL.md](../SKILL.md) | [pwa-fundamentals.md](pwa-fundamentals.md) | [responsive-design.md](responsive-design.md)

A comprehensive guide to optimizing Progressive Web App performance for mobile devices, covering Core Web Vitals, image optimization, code splitting, caching strategies, and mobile network considerations.

---

## Table of Contents

1. [Core Web Vitals](#core-web-vitals)
2. [Image Optimization](#image-optimization)
3. [Code Splitting and Lazy Loading](#code-splitting-and-lazy-loading)
4. [Critical CSS and Font Loading](#critical-css-and-font-loading)
5. [Service Worker Caching Performance](#service-worker-caching-performance)
6. [Network-Aware Loading](#network-aware-loading)
7. [Build Tool Optimization](#build-tool-optimization)
8. [Real User Monitoring](#real-user-monitoring)

---

## Core Web Vitals

### What are Core Web Vitals?

Google's **Core Web Vitals** are three key metrics measuring user experience:

1. **Largest Contentful Paint (LCP)** - Loading performance
2. **Interaction to Next Paint (INP)** - Interactivity (replaced FID in March 2024)
3. **Cumulative Layout Shift (CLS)** - Visual stability

**Target Thresholds** (for 75th percentile of page loads):

| Metric | Good | Needs Improvement | Poor |
|--------|------|-------------------|------|
| **LCP** | ≤2.5s | 2.5s - 4.0s | >4.0s |
| **INP** | ≤200ms | 200ms - 500ms | >500ms |
| **CLS** | ≤0.1 | 0.1 - 0.25 | >0.25 |

---

### 1. Largest Contentful Paint (LCP)

**What it measures**: Time until the largest content element becomes visible in the viewport.

**Common LCP elements**:
- `<img>` elements
- `<video>` elements (poster image)
- Block-level elements with background images
- Text blocks

**How to optimize for LCP ≤2.5s**:

#### a. Optimize Server Response Time (TTFB)

```javascript
// SvelteKit: Prerender static pages at build time
// src/routes/+page.server.ts
export const prerender = true;

export async function load() {
  // Fetch data at build time, not runtime
  return {
    posts: await fetchPosts()
  };
}
```

#### b. Preload Critical Resources

```html
<!-- src/app.html -->
<head>
  <!-- Preload LCP image -->
  <link rel="preload" as="image" href="/hero.avif" fetchpriority="high" />

  <!-- Preload critical fonts -->
  <link rel="preload" as="font" type="font/woff2" href="/fonts/inter-var.woff2" crossorigin />

  <!-- Preconnect to external domains -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
</head>
```

#### c. Optimize LCP Images

```svelte
<script lang="ts">
  export let data;
</script>

<!-- Hero image with priority loading -->
<picture>
  <source
    type="image/avif"
    srcset="
      /images/hero-400.avif 400w,
      /images/hero-800.avif 800w,
      /images/hero-1200.avif 1200w
    "
    sizes="100vw"
  />
  <source
    type="image/webp"
    srcset="
      /images/hero-400.webp 400w,
      /images/hero-800.webp 800w,
      /images/hero-1200.webp 1200w
    "
    sizes="100vw"
  />
  <img
    src="/images/hero-800.jpg"
    alt="Hero banner"
    width="1200"
    height="600"
    fetchpriority="high"
    decoding="async"
  />
</picture>

<style>
  img {
    width: 100%;
    height: auto;
    display: block;
  }
</style>
```

#### d. Eliminate Render-Blocking Resources

```html
<!-- Inline critical CSS -->
<style>
  /* Critical above-the-fold styles */
  body { margin: 0; font-family: system-ui; }
  .hero { min-height: 400px; background: #f0f0f0; }
</style>

<!-- Defer non-critical CSS -->
<link rel="preload" as="style" href="/styles/main.css" onload="this.onload=null;this.rel='stylesheet'" />
<noscript><link rel="stylesheet" href="/styles/main.css" /></noscript>
```

#### e. Use CDN and HTTP/2

```javascript
// vite.config.ts - Configure CDN for production
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [sveltekit()],
  build: {
    rollupOptions: {
      output: {
        // Use CDN for static assets in production
        assetFileNames: (assetInfo) => {
          return `assets/[name]-[hash][extname]`;
        }
      }
    }
  }
});
```

**Measuring LCP**:

```javascript
// Measure LCP in the browser
if (typeof window !== 'undefined' && 'PerformanceObserver' in window) {
  const observer = new PerformanceObserver((list) => {
    const entries = list.getEntries();
    const lastEntry = entries[entries.length - 1];

    console.log('LCP:', lastEntry.renderTime || lastEntry.loadTime);

    // Send to analytics
    // analytics.track('LCP', { value: lastEntry.renderTime || lastEntry.loadTime });
  });

  observer.observe({ type: 'largest-contentful-paint', buffered: true });
}
```

---

### 2. Interaction to Next Paint (INP)

**What it measures**: Responsiveness to user interactions throughout the page lifecycle.

**How to optimize for INP ≤200ms**:

#### a. Minimize JavaScript Execution

```javascript
// Break up long tasks with setTimeout
async function processLargeList(items) {
  const chunkSize = 50;

  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);

    // Process chunk
    chunk.forEach(processItem);

    // Yield to browser for user interactions
    await new Promise(resolve => setTimeout(resolve, 0));
  }
}
```

#### b. Use Web Workers for Heavy Computation

```javascript
// src/lib/workers/data-processor.ts
addEventListener('message', (event) => {
  const { data } = event;

  // Perform heavy computation off main thread
  const result = processData(data);

  postMessage(result);
});

function processData(data: unknown[]) {
  // CPU-intensive work here
  return data.map(item => /* transform */);
}
```

```svelte
<script lang="ts">
  import { browser } from '$app/environment';
  import { onMount } from 'svelte';

  let processedData = [];

  onMount(() => {
    if (!browser) return;

    const worker = new Worker('/workers/data-processor.js');

    worker.postMessage(rawData);

    worker.onmessage = (event) => {
      processedData = event.data;
    };

    return () => worker.terminate();
  });
</script>
```

#### c. Debounce and Throttle Event Handlers

```svelte
<script lang="ts">
  import { debounce } from '$lib/utils';

  let searchQuery = '';

  // Debounce search to avoid excessive API calls
  const debouncedSearch = debounce(async (query: string) => {
    const results = await fetch(`/api/search?q=${query}`);
    // Update UI with results
  }, 300);

  $: debouncedSearch(searchQuery);
</script>

<input
  type="search"
  bind:value={searchQuery}
  placeholder="Search..."
/>
```

```typescript
// src/lib/utils.ts
export function debounce<T extends (...args: any[]) => any>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timeoutId: number | undefined;

  return (...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    timeoutId = window.setTimeout(() => fn(...args), delay);
  };
}

export function throttle<T extends (...args: any[]) => any>(
  fn: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle = false;

  return (...args: Parameters<T>) => {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}
```

#### d. Optimize Third-Party Scripts

```html
<!-- Defer non-critical third-party scripts -->
<script src="https://analytics.example.com/script.js" defer></script>

<!-- Use facade pattern for heavy embeds (YouTube, maps) -->
```

```svelte
<!-- YouTube facade component -->
<script lang="ts">
  export let videoId: string;

  let loaded = false;

  function loadVideo() {
    loaded = true;
  }
</script>

{#if !loaded}
  <button class="video-facade" on:click={loadVideo}>
    <img
      src="https://img.youtube.com/vi/{videoId}/hqdefault.jpg"
      alt="Video thumbnail"
      loading="lazy"
    />
    <div class="play-button">▶</div>
  </button>
{:else}
  <iframe
    src="https://www.youtube.com/embed/{videoId}?autoplay=1"
    title="YouTube video"
    frameborder="0"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture"
    allowfullscreen
  ></iframe>
{/if}

<style>
  .video-facade {
    position: relative;
    width: 100%;
    aspect-ratio: 16 / 9;
    cursor: pointer;
    border: none;
    padding: 0;
  }

  .play-button {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 4rem;
    color: white;
    background: rgba(0, 0, 0, 0.7);
    border-radius: 50%;
    width: 80px;
    height: 80px;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  iframe {
    width: 100%;
    aspect-ratio: 16 / 9;
  }
</style>
```

**Measuring INP**:

```javascript
// Measure INP
if (typeof window !== 'undefined' && 'PerformanceObserver' in window) {
  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      // Log interaction delays
      console.log('Interaction delay:', entry.duration, entry.name);
    }
  });

  observer.observe({ type: 'event', buffered: true, durationThreshold: 16 });
}
```

---

### 3. Cumulative Layout Shift (CLS)

**What it measures**: Visual stability - unexpected layout shifts during page load.

**How to optimize for CLS ≤0.1**:

#### a. Always Specify Image Dimensions

```svelte
<!-- Bad: No dimensions, causes layout shift when image loads -->
<img src="/image.jpg" alt="Product" />

<!-- Good: Dimensions prevent layout shift -->
<img
  src="/image.jpg"
  alt="Product"
  width="800"
  height="600"
  loading="lazy"
/>

<style>
  img {
    width: 100%;
    height: auto;
  }
</style>
```

#### b. Reserve Space for Dynamic Content

```svelte
<script lang="ts">
  import { onMount } from 'svelte';

  let adContent = null;

  onMount(async () => {
    adContent = await fetchAd();
  });
</script>

<!-- Reserve space for ad before it loads -->
<div class="ad-container">
  {#if adContent}
    {@html adContent}
  {:else}
    <!-- Placeholder with same dimensions -->
    <div class="ad-placeholder"></div>
  {/if}
</div>

<style>
  .ad-container {
    min-height: 250px; /* Reserve space */
  }

  .ad-placeholder {
    width: 100%;
    height: 250px;
    background: #f0f0f0;
  }
</style>
```

#### c. Use CSS aspect-ratio for Responsive Elements

```css
/* Modern aspect ratio for responsive containers */
.video-container {
  width: 100%;
  aspect-ratio: 16 / 9;
  background: #000;
}

/* Fallback for older browsers */
@supports not (aspect-ratio: 16 / 9) {
  .video-container {
    position: relative;
    padding-bottom: 56.25%; /* 16:9 */
    height: 0;
  }

  .video-container iframe {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
  }
}
```

#### d. Avoid Inserting Content Above Existing Content

```svelte
<!-- Bad: Banner appears above content, causing shift -->
<script>
  let showBanner = false;
  onMount(() => {
    setTimeout(() => showBanner = true, 2000);
  });
</script>

{#if showBanner}
  <div class="banner">Special offer!</div>
{/if}

<main>Content shifts down when banner appears</main>

<!-- Good: Use fixed positioning or animate in from top -->
{#if showBanner}
  <div class="banner-fixed">Special offer!</div>
{/if}

<main>Content stays in place</main>

<style>
  .banner-fixed {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    z-index: 100;
    animation: slideDown 300ms ease;
  }

  @keyframes slideDown {
    from { transform: translateY(-100%); }
    to { transform: translateY(0); }
  }
</style>
```

#### e. Preload Fonts to Prevent FOUT/FOIT

```html
<!-- Preload critical fonts -->
<link rel="preload" as="font" type="font/woff2" href="/fonts/inter-var.woff2" crossorigin />
```

```css
/* Use font-display: swap to prevent invisible text */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2');
  font-weight: 100 900;
  font-display: swap; /* Show fallback font immediately, swap when custom font loads */
}
```

**Measuring CLS**:

```javascript
// Measure CLS
if (typeof window !== 'undefined' && 'PerformanceObserver' in window) {
  let clsScore = 0;

  const observer = new PerformanceObserver((list) => {
    for (const entry of list.getEntries()) {
      if (!entry.hadRecentInput) {
        clsScore += entry.value;
        console.log('CLS:', clsScore);
      }
    }
  });

  observer.observe({ type: 'layout-shift', buffered: true });
}
```

---

## Image Optimization

### Modern Image Formats

**Format Priority** (smallest to largest):
1. **AVIF** - 50% smaller than JPEG, excellent quality (90%+ browser support)
2. **WebP** - 30% smaller than JPEG (97%+ browser support)
3. **JPEG** - Universal fallback

### Responsive Images with srcset

```svelte
<script lang="ts">
  export let src: string;
  export let alt: string;
  export let width: number;
  export let height: number;
</script>

<picture>
  <!-- AVIF for modern browsers -->
  <source
    type="image/avif"
    srcset="
      {src}-400.avif 400w,
      {src}-800.avif 800w,
      {src}-1200.avif 1200w,
      {src}-1600.avif 1600w
    "
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  />

  <!-- WebP for wider compatibility -->
  <source
    type="image/webp"
    srcset="
      {src}-400.webp 400w,
      {src}-800.webp 800w,
      {src}-1200.webp 1200w,
      {src}-1600.webp 1600w
    "
    sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
  />

  <!-- JPEG fallback -->
  <img
    src="{src}-800.jpg"
    {alt}
    {width}
    {height}
    loading="lazy"
    decoding="async"
  />
</picture>

<style>
  img {
    width: 100%;
    height: auto;
    display: block;
  }
</style>
```

### Lazy Loading Images

```svelte
<!-- Browser-native lazy loading -->
<img
  src="/image.jpg"
  alt="Product"
  loading="lazy"
  decoding="async"
  width="800"
  height="600"
/>

<!-- Lazy load with Intersection Observer for more control -->
<script lang="ts">
  import { onMount } from 'svelte';

  export let src: string;
  export let alt: string;

  let imgElement: HTMLImageElement;
  let loaded = false;

  onMount(() => {
    const observer = new IntersectionObserver(
      (entries) => {
        entries.forEach((entry) => {
          if (entry.isIntersecting) {
            loaded = true;
            observer.disconnect();
          }
        });
      },
      { rootMargin: '50px' } // Start loading 50px before entering viewport
    );

    observer.observe(imgElement);

    return () => observer.disconnect();
  });
</script>

<img
  bind:this={imgElement}
  src={loaded ? src : 'data:image/svg+xml,%3Csvg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 800 600"%3E%3C/svg%3E'}
  {alt}
  class:loaded
/>

<style>
  img {
    width: 100%;
    height: auto;
    background: #f0f0f0;
    transition: opacity 300ms ease;
  }

  img:not(.loaded) {
    opacity: 0.5;
  }
</style>
```

### Image CDN and Automatic Optimization

```svelte
<script lang="ts">
  // Use image CDN for automatic format conversion and optimization
  function getOptimizedImage(src: string, width: number, format: 'avif' | 'webp' | 'jpg' = 'avif') {
    // Example with Cloudinary
    return `https://res.cloudinary.com/your-cloud/image/upload/f_${format},w_${width},q_auto/${src}`;

    // Example with imgix
    // return `https://your-domain.imgix.net/${src}?w=${width}&auto=format,compress`;
  }
</script>

<img
  src={getOptimizedImage('products/shoe.jpg', 800)}
  alt="Product"
  loading="lazy"
/>
```

---

## Code Splitting and Lazy Loading

### Route-Based Code Splitting (Automatic in SvelteKit)

```
src/routes/
├── +layout.svelte         # Loaded on all pages
├── +page.svelte           # Home page
├── about/+page.svelte     # About page (lazy loaded)
└── products/
    ├── +page.svelte       # Products list (lazy loaded)
    └── [id]/+page.svelte  # Product detail (lazy loaded)
```

SvelteKit automatically code-splits each route into separate bundles.

### Component-Level Code Splitting

```svelte
<script lang="ts">
  import { browser } from '$app/environment';

  // Lazy load heavy component only when needed
  let HeavyChart;

  async function loadChart() {
    if (!browser) return;

    const module = await import('$lib/components/HeavyChart.svelte');
    HeavyChart = module.default;
  }

  let showChart = false;

  $: if (showChart && !HeavyChart) {
    loadChart();
  }
</script>

<button on:click={() => showChart = true}>
  Show Chart
</button>

{#if showChart}
  {#if HeavyChart}
    <svelte:component this={HeavyChart} />
  {:else}
    <p>Loading chart...</p>
  {/if}
{/if}
```

### Lazy Load Below the Fold

```svelte
<script lang="ts">
  import { onMount } from 'svelte';

  let Footer;
  let Reviews;

  onMount(async () => {
    // Lazy load below-the-fold components after page load
    const [footerModule, reviewsModule] = await Promise.all([
      import('$lib/components/Footer.svelte'),
      import('$lib/components/Reviews.svelte')
    ]);

    Footer = footerModule.default;
    Reviews = reviewsModule.default;
  });
</script>

<!-- Above the fold: loaded immediately -->
<header>...</header>
<main>...</main>

<!-- Below the fold: lazy loaded -->
{#if Reviews}
  <svelte:component this={Reviews} />
{/if}

{#if Footer}
  <svelte:component this={Footer} />
{/if}
```

### Preload on Hover/Focus

```svelte
<script lang="ts">
  import { preloadCode } from '$app/navigation';

  function preloadRoute(path: string) {
    preloadCode(path);
  }
</script>

<a
  href="/products"
  on:mouseenter={() => preloadRoute('/products')}
  on:focus={() => preloadRoute('/products')}
>
  Products
</a>
```

---

## Critical CSS and Font Loading

### Inline Critical CSS

```html
<!-- src/app.html -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />

  <!-- Inline critical above-the-fold CSS -->
  <style>
    /* Reset and critical layout */
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui, sans-serif; line-height: 1.5; }

    /* Critical header styles */
    .header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 1rem;
      background: white;
      border-bottom: 1px solid #e0e0e0;
    }

    /* Critical hero styles */
    .hero {
      min-height: 400px;
      background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
      display: flex;
      align-items: center;
      justify-content: center;
      color: white;
    }
  </style>

  %sveltekit.head%

  <!-- Defer non-critical CSS -->
  <link rel="preload" as="style" href="%sveltekit.assets%/styles/main.css" onload="this.onload=null;this.rel='stylesheet'" />
  <noscript><link rel="stylesheet" href="%sveltekit.assets%/styles/main.css" /></noscript>
</head>
<body>
  <div>%sveltekit.body%</div>
</body>
</html>
```

### Optimal Font Loading Strategy

```html
<!-- Preload critical fonts -->
<link rel="preload" as="font" type="font/woff2" href="/fonts/inter-var.woff2" crossorigin />
```

```css
/* Font face with optimal font-display */
@font-face {
  font-family: 'Inter';
  src: url('/fonts/inter-var.woff2') format('woff2-variations');
  font-weight: 100 900;
  font-style: normal;
  font-display: swap; /* Show fallback immediately, swap when loaded */
}

/* Use system font as fallback */
body {
  font-family: 'Inter', system-ui, -apple-system, sans-serif;
}
```

**Font Loading API**:

```svelte
<script lang="ts">
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';

  let fontLoaded = false;

  onMount(async () => {
    if (!browser || !('fonts' in document)) return;

    try {
      await document.fonts.load('1rem Inter');
      fontLoaded = true;
    } catch (error) {
      console.error('Font loading failed:', error);
    }
  });
</script>

<div class:font-loaded={fontLoaded}>
  <!-- Content uses system font until custom font loads -->
</div>

<style>
  div {
    font-family: system-ui, sans-serif;
    transition: font-family 0s 100ms; /* Delay to prevent FOUT */
  }

  div.font-loaded {
    font-family: 'Inter', system-ui, sans-serif;
  }
</style>
```

---

## Service Worker Caching Performance

### Precache Critical Assets

```javascript
// src/service-worker.js
import { build, files, version } from '$service-worker';
import { precacheAndRoute } from 'workbox-precaching';

// Precache build files and static assets
const precache = [
  ...build, // App shell (JS, CSS)
  ...files  // Static files in /static
].map(file => ({
  url: file,
  revision: version
}));

precacheAndRoute(precache);
```

### Cache First for Static Assets

```javascript
import { registerRoute } from 'workbox-routing';
import { CacheFirst } from 'workbox-strategies';
import { CacheableResponsePlugin } from 'workbox-cacheable-response';
import { ExpirationPlugin } from 'workbox-expiration';

// Cache images with Cache First strategy
registerRoute(
  ({ request }) => request.destination === 'image',
  new CacheFirst({
    cacheName: 'images',
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      }),
      new ExpirationPlugin({
        maxEntries: 60,
        maxAgeSeconds: 30 * 24 * 60 * 60, // 30 days
        purgeOnQuotaError: true
      })
    ]
  })
);
```

### Network First for API Calls

```javascript
import { NetworkFirst } from 'workbox-strategies';

registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({
    cacheName: 'api-cache',
    plugins: [
      new CacheableResponsePlugin({
        statuses: [0, 200]
      }),
      new ExpirationPlugin({
        maxEntries: 50,
        maxAgeSeconds: 5 * 60 // 5 minutes
      })
    ],
    networkTimeoutSeconds: 3 // Fallback to cache after 3s
  })
);
```

---

## Network-Aware Loading

### Detect Connection Type

```svelte
<script lang="ts">
  import { browser } from '$app/environment';
  import { onMount } from 'svelte';

  let connectionType: 'slow' | 'fast' = 'fast';
  let saveData = false;

  onMount(() => {
    if (!browser) return;

    const connection = (navigator as any).connection || (navigator as any).mozConnection || (navigator as any).webkitConnection;

    if (connection) {
      // Detect effective connection type
      const effectiveType = connection.effectiveType;
      connectionType = ['slow-2g', '2g', '3g'].includes(effectiveType) ? 'slow' : 'fast';

      // Check if user has data saver enabled
      saveData = connection.saveData === true;

      // Listen for connection changes
      connection.addEventListener('change', () => {
        connectionType = ['slow-2g', '2g', '3g'].includes(connection.effectiveType) ? 'slow' : 'fast';
      });
    }
  });
</script>

{#if connectionType === 'slow' || saveData}
  <!-- Load low-quality images on slow connections -->
  <img src="/image-low-quality.jpg" alt="Product" />
{:else}
  <!-- Load high-quality images on fast connections -->
  <picture>
    <source type="image/avif" srcset="/image.avif" />
    <img src="/image.jpg" alt="Product" />
  </picture>
{/if}
```

### Adaptive Loading

```svelte
<script lang="ts">
  import { browser } from '$app/environment';

  let deviceMemory = 8; // Default to high-end
  let hardwareConcurrency = 8;

  if (browser) {
    deviceMemory = (navigator as any).deviceMemory || 8;
    hardwareConcurrency = navigator.hardwareConcurrency || 8;
  }

  const isLowEndDevice = deviceMemory < 4 || hardwareConcurrency < 4;
</script>

{#if isLowEndDevice}
  <!-- Simplified UI for low-end devices -->
  <div class="simple-ui">
    <!-- No animations, simpler layouts -->
  </div>
{:else}
  <!-- Full-featured UI for high-end devices -->
  <div class="rich-ui">
    <!-- Animations, complex layouts, WebGL, etc. -->
  </div>
{/if}
```

---

## Build Tool Optimization

### Vite Configuration for SvelteKit

```typescript
// vite.config.ts
import { sveltekit } from '@sveltejs/kit/vite';
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [sveltekit()],

  build: {
    // Increase chunk size warning limit if needed
    chunkSizeWarningLimit: 1000,

    rollupOptions: {
      output: {
        // Manual chunk splitting for better caching
        manualChunks: (id) => {
          // Vendor chunk for node_modules
          if (id.includes('node_modules')) {
            // Split large libraries into separate chunks
            if (id.includes('chart.js')) return 'charts';
            if (id.includes('date-fns')) return 'date-utils';

            return 'vendor';
          }

          // Component chunks
          if (id.includes('src/lib/components')) {
            return 'components';
          }
        }
      }
    },

    // Minification
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true, // Remove console.log in production
        drop_debugger: true
      }
    }
  },

  // Dependency optimization
  optimizeDeps: {
    include: ['svelte', '@sveltejs/kit'],
    exclude: ['large-optional-dependency']
  }
});
```

### Bundle Analysis

```bash
# Install bundle analyzer
npm install -D rollup-plugin-visualizer

# Generate bundle report
npx vite build --mode analyze
```

```typescript
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    sveltekit(),
    visualizer({
      filename: './stats.html',
      open: true,
      gzipSize: true,
      brotliSize: true
    })
  ]
});
```

---

## Real User Monitoring

### Web Vitals Library

```bash
npm install web-vitals
```

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import { browser } from '$app/environment';
  import { onMount } from 'svelte';

  onMount(async () => {
    if (!browser) return;

    const { onCLS, onINP, onLCP, onFCP, onTTFB } = await import('web-vitals');

    function sendToAnalytics(metric: any) {
      // Send to your analytics endpoint
      console.log(metric.name, metric.value);

      // Example: Send to Google Analytics
      // gtag('event', metric.name, {
      //   value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
      //   metric_id: metric.id,
      //   metric_value: metric.value,
      //   metric_delta: metric.delta
      // });
    }

    onCLS(sendToAnalytics);
    onINP(sendToAnalytics);
    onLCP(sendToAnalytics);
    onFCP(sendToAnalytics);
    onTTFB(sendToAnalytics);
  });
</script>

<slot />
```

### Performance Observer

```typescript
// src/lib/performance-monitoring.ts
export function initPerformanceMonitoring() {
  if (typeof window === 'undefined') return;

  // Monitor long tasks (>50ms)
  if ('PerformanceObserver' in window) {
    const longTaskObserver = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        console.warn('Long task detected:', entry.duration, entry);
        // Send to analytics
      }
    });

    longTaskObserver.observe({ type: 'longtask', buffered: true });
  }

  // Monitor resource loading
  window.addEventListener('load', () => {
    const resources = performance.getEntriesByType('resource');

    resources.forEach((resource: any) => {
      if (resource.duration > 1000) {
        console.warn('Slow resource:', resource.name, resource.duration);
      }
    });
  });
}
```

### Lighthouse CI

```yaml
# .github/workflows/lighthouse.yml
name: Lighthouse CI

on:
  pull_request:
    branches: [main]

jobs:
  lighthouse:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18

      - name: Install dependencies
        run: npm ci

      - name: Build
        run: npm run build

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v9
        with:
          urls: |
            http://localhost:4173
            http://localhost:4173/products
          uploadArtifacts: true
          temporaryPublicStorage: true
```

---

## Summary

Performance optimization for mobile PWAs requires attention to:

1. **Core Web Vitals**: Target LCP ≤2.5s, INP ≤200ms, CLS ≤0.1
2. **Image Optimization**: Use AVIF/WebP, responsive images, lazy loading
3. **Code Splitting**: Route-based and component-level lazy loading
4. **Critical Resources**: Inline critical CSS, optimize font loading
5. **Service Worker Caching**: Precache app shell, cache-first for assets
6. **Network-Aware Loading**: Adapt to connection speed and device capabilities
7. **Build Optimization**: Manual chunking, tree-shaking, minification
8. **Monitoring**: Web Vitals, Performance Observer, Lighthouse CI

**Related Resources**:
- [pwa-fundamentals.md](pwa-fundamentals.md) - Service worker caching strategies
- [responsive-design.md](responsive-design.md) - Responsive images and layouts
- [sveltekit-pwa-patterns.md](sveltekit-pwa-patterns.md) - SvelteKit-specific optimization
