# Mobile Performance - SvelteKit PWAs

**Mobile-specific performance optimizations and patterns**

## Performance Optimizations

### Optimize Touch Responsiveness

```css
/* Improve touch response */
.touchable {
  /* Remove 300ms tap delay */
  touch-action: manipulation;
  
  /* Hardware acceleration */
  transform: translateZ(0);
  will-change: transform;
  
  /* Prevent text selection */
  user-select: none;
  -webkit-user-select: none;
}

/* Active state feedback */
.touchable:active {
  transform: scale(0.98);
  opacity: 0.8;
}
```

### Lazy Loading Images

```svelte
<!-- lib/components/LazyImage.svelte -->
<script>
  import { onMount } from 'svelte';
  
  export let src;
  export let alt;
  export let placeholder = '/placeholder.webp';
  
  let imgElement;
  let isLoaded = false;
  
  onMount(() => {
    const observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          const img = new Image();
          img.src = src;
          img.onload = () => {
            isLoaded = true;
            observer.disconnect();
          };
        }
      });
    }, {
      rootMargin: '50px'
    });
    
    if (imgElement) {
      observer.observe(imgElement);
    }
    
    return () => observer.disconnect();
  });
</script>

<div class="lazy-image" bind:this={imgElement}>
  {#if isLoaded}
    <img {src} {alt} />
  {:else}
    <img src={placeholder} {alt} />
  {/if}
</div>

<style>
  .lazy-image img {
    width: 100%;
    height: auto;
  }
</style>
```

### Virtualized Lists

```svelte
<!-- lib/components/VirtualList.svelte -->
<script>
  import { onMount } from 'svelte';
  
  export let items = [];
  export let itemHeight = 60;
  export let visibleItems = 10;
  
  let scrollTop = 0;
  let containerHeight = 0;
  let container;
  
  $: startIndex = Math.floor(scrollTop / itemHeight);
  $: endIndex = Math.min(
    startIndex + visibleItems + 2,
    items.length
  );
  $: visibleRange = items.slice(startIndex, endIndex);
  $: offsetY = startIndex * itemHeight;
  
  onMount(() => {
    containerHeight = container.clientHeight;
    visibleItems = Math.ceil(containerHeight / itemHeight);
  });
  
  function handleScroll(e) {
    scrollTop = e.target.scrollTop;
  }
</script>

<div
  class="virtual-list"
  bind:this={container}
  on:scroll={handleScroll}
>
  <div style="height: {items.length * itemHeight}px">
    <div style="transform: translateY({offsetY}px)">
      {#each visibleRange as item (item.id)}
        <div class="list-item" style="height: {itemHeight}px">
          <slot {item} />
        </div>
      {/each}
    </div>
  </div>
</div>

<style>
  .virtual-list {
    overflow-y: auto;
    height: 100%;
    -webkit-overflow-scrolling: touch;
  }
</style>
```

## Mobile Navigation Patterns

### Bottom Navigation Bar

```svelte
<!-- lib/components/BottomNav.svelte -->
<script>
  import { page } from '$app/stores';
  
  const navItems = [
    { href: '/', icon: '🏠', label: 'Home' },
    { href: '/search', icon: '🔍', label: 'Search' },
    { href: '/profile', icon: '👤', label: 'Profile' }
  ];
</script>

<nav class="bottom-nav">
  {#each navItems as item}
    <a
      href={item.href}
      class:active={$page.url.pathname === item.href}
      class="nav-item"
    >
      <span class="icon">{item.icon}</span>
      <span class="label">{item.label}</span>
    </a>
  {/each}
</nav>

<style>
  .bottom-nav {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    display: flex;
    background: white;
    border-top: 1px solid #eee;
    padding-bottom: env(safe-area-inset-bottom);
  }
  
  .nav-item {
    flex: 1;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 0.5rem;
    text-decoration: none;
    color: #666;
  }
  
  .nav-item.active {
    color: var(--primary-color);
  }
</style>
```

### Swipeable Tabs

```svelte
<!-- lib/components/SwipeableTabs.svelte -->
<script>
  import { spring } from 'svelte/motion';
  
  export let tabs = [];
  export let activeTab = 0;
  
  let containerWidth = 0;
  let touchStart = 0;
  let offset = spring(0);
  
  $: offset.set(-activeTab * containerWidth);
  
  function handleTouchStart(e) {
    touchStart = e.touches[0].clientX;
  }
  
  function handleTouchEnd(e) {
    const touchEnd = e.changedTouches[0].clientX;
    const diff = touchStart - touchEnd;
    
    if (Math.abs(diff) > 50) {
      if (diff > 0 && activeTab < tabs.length - 1) {
        activeTab++;
      } else if (diff < 0 && activeTab > 0) {
        activeTab--;
      }
    }
  }
</script>

<div class="tabs-container" bind:clientWidth={containerWidth}>
  <div class="tab-headers">
    {#each tabs as tab, i}
      <button
        class:active={activeTab === i}
        on:click={() => activeTab = i}
      >
        {tab.label}
      </button>
    {/each}
  </div>
  
  <div
    class="tab-content"
    on:touchstart={handleTouchStart}
    on:touchend={handleTouchEnd}
  >
    <div
      class="tab-panels"
      style="transform: translateX({$offset}px)"
    >
      {#each tabs as tab, i}
        <div class="tab-panel" style="width: {containerWidth}px">
          <svelte:component this={tab.component} />
        </div>
      {/each}
    </div>
  </div>
</div>
```

## Device Feature Detection

```javascript
// lib/utils/device-features.js
export function detectFeatures() {
  return {
    touch: 'ontouchstart' in window,
    geolocation: 'geolocation' in navigator,
    camera: 'mediaDevices' in navigator,
    vibration: 'vibrate' in navigator,
    battery: 'getBattery' in navigator,
    network: 'connection' in navigator,
    orientation: 'orientation' in screen,
    motion: 'DeviceMotionEvent' in window,
    storage: 'storage' in navigator,
    share: 'share' in navigator,
    bluetooth: 'bluetooth' in navigator,
    nfc: 'NDEFReader' in window
  };
}

// Usage
import { detectFeatures } from '$lib/utils/device-features';

const features = detectFeatures();
if (features.touch) {
  // Enable touch-specific features
}
```

## Testing Mobile Features

### Device Emulation Setup

```javascript
// playwright.config.js
export default {
  projects: [
    {
      name: 'Mobile Chrome',
      use: {
        ...devices['Pixel 5']
      }
    },
    {
      name: 'Mobile Safari',
      use: {
        ...devices['iPhone 13']
      }
    }
  ]
};
```

### Touch Event Testing

```javascript
// tests/mobile.test.js
import { test, expect } from '@playwright/test';

test('swipe navigation works', async ({ page }) => {
  await page.goto('/');
  
  // Simulate swipe
  await page.locator('.swipeable').dispatchEvent('touchstart', {
    touches: [{ clientX: 300, clientY: 400 }]
  });
  
  await page.locator('.swipeable').dispatchEvent('touchend', {
    changedTouches: [{ clientX: 100, clientY: 400 }]
  });
  
  // Verify navigation occurred
  await expect(page).toHaveURL('/next-page');
});
```

## Best Practices

1. **Test on real devices** - Emulators miss hardware-specific issues
2. **Optimize for thumb reach** - Place interactive elements in thumb zone
3. **Provide visual feedback** - Use active states for all touchable elements
4. **Handle offline gracefully** - Show meaningful offline states
5. **Minimize input requirements** - Use appropriate input types and autocomplete
6. **Respect user preferences** - Honor reduced motion, dark mode, font size settings

## References

- [Touch Events API](https://developer.mozilla.org/en-US/docs/Web/API/Touch_events)
- [Viewport Meta Tag](https://developer.mozilla.org/en-US/docs/Web/HTML/Viewport_meta_tag)
- [Safe Area Insets](https://webkit.org/blog/7929/designing-websites-for-iphone-x/)
