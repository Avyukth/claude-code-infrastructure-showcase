# Mobile Optimization for SvelteKit PWAs

## Overview

Mobile optimization is critical for PWA success. This guide covers touch interactions, viewport management, responsive design patterns, and mobile-specific performance optimizations.

## Touch Gesture Implementation

### Using svelte-gesture Library

```bash
npm i svelte-gesture
```

**Basic Touch Gestures:**
```svelte
<script>
  import { drag, pinch, swipe } from 'svelte-gesture';
  import { spring } from 'svelte/motion';
  
  let coords = spring({ x: 0, y: 0 });
  let scale = spring(1);
  
  function handleDrag({ detail }) {
    const { movement: [mx, my], active } = detail;
    coords.set({
      x: active ? mx : 0,
      y: active ? my : 0
    });
  }
  
  function handlePinch({ detail }) {
    scale.set(detail.offset[0]);
  }
  
  function handleSwipe({ detail }) {
    const { direction, velocity } = detail;
    if (direction[0] > 0 && velocity > 0.5) {
      // Swiped right
      navigateBack();
    }
  }
</script>

<div
  use:drag
  on:drag={handleDrag}
  use:pinch
  on:pinch={handlePinch}
  use:swipe={{ threshold: 10 }}
  on:swipe={handleSwipe}
  style="transform: translate({$coords.x}px, {$coords.y}px) scale({$scale})"
>
  <!-- Content -->
</div>
```

### Custom Touch Handling

```svelte
<script>
  let touchStart = { x: 0, y: 0 };
  let touchCurrent = { x: 0, y: 0 };
  let isDragging = false;
  
  function handleTouchStart(e) {
    const touch = e.touches[0];
    touchStart = { x: touch.clientX, y: touch.clientY };
    touchCurrent = { ...touchStart };
    isDragging = true;
  }
  
  function handleTouchMove(e) {
    if (!isDragging) return;
    
    const touch = e.touches[0];
    touchCurrent = { x: touch.clientX, y: touch.clientY };
    
    // Prevent scrolling while dragging
    e.preventDefault();
  }
  
  function handleTouchEnd(e) {
    const deltaX = touchCurrent.x - touchStart.x;
    const deltaY = touchCurrent.y - touchStart.y;
    
    // Determine gesture type
    if (Math.abs(deltaX) > 50 && Math.abs(deltaX) > Math.abs(deltaY)) {
      // Horizontal swipe
      if (deltaX > 0) {
        handleSwipeRight();
      } else {
        handleSwipeLeft();
      }
    }
    
    isDragging = false;
  }
</script>

<div
  on:touchstart={handleTouchStart}
  on:touchmove={handleTouchMove}
  on:touchend={handleTouchEnd}
  style="touch-action: none;"
>
  <!-- Draggable content -->
</div>
```

## Viewport Configuration

### Advanced Viewport Meta Tags

```html
<!-- src/app.html -->
<meta name="viewport" content="
  width=device-width,
  initial-scale=1,
  minimum-scale=1,
  maximum-scale=5,
  user-scalable=yes,
  viewport-fit=cover
">

<!-- iOS specific -->
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">

<!-- Disable tap highlight -->
<style>
  * { -webkit-tap-highlight-color: transparent; }
</style>
```

### Safe Area Handling (iPhone Notch)

```css
/* global.css */
.app-container {
  /* Account for safe areas */
  padding-top: env(safe-area-inset-top);
  padding-right: env(safe-area-inset-right);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  
  /* Or use CSS variables */
  --safe-top: env(safe-area-inset-top, 0);
  --safe-right: env(safe-area-inset-right, 0);
  --safe-bottom: env(safe-area-inset-bottom, 0);
  --safe-left: env(safe-area-inset-left, 0);
}

/* Fixed header accounting for notch */
.header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: calc(60px + env(safe-area-inset-top));
  padding-top: env(safe-area-inset-top);
}

/* Bottom navigation with home indicator space */
.bottom-nav {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  padding-bottom: env(safe-area-inset-bottom);
  /* Extra padding for home indicator */
  padding-bottom: calc(env(safe-area-inset-bottom) + 8px);
}
```

## Responsive Design Patterns

### Mobile-First Breakpoints

```css
/* lib/styles/breakpoints.css */
:root {
  /* Mobile-first breakpoints */
  --screen-sm: 640px;   /* Small tablets */
  --screen-md: 768px;   /* Tablets */
  --screen-lg: 1024px;  /* Desktop */
  --screen-xl: 1280px;  /* Large desktop */
}

/* Mobile-first approach */
.container {
  /* Mobile styles (default) */
  padding: 1rem;
  font-size: 14px;
}

@media (min-width: 640px) {
  .container {
    /* Tablet styles */
    padding: 1.5rem;
    font-size: 16px;
  }
}

@media (min-width: 1024px) {
  .container {
    /* Desktop styles */
    padding: 2rem;
    max-width: 1200px;
    margin: 0 auto;
  }
}
```

### Responsive Component Pattern

```svelte
<!-- lib/components/ResponsiveLayout.svelte -->
<script>
  import { onMount } from 'svelte';
  import { writable } from 'svelte/store';
  
  export let breakpoint = 768;
  
  const isMobile = writable(true);
  
  onMount(() => {
    const checkMobile = () => {
      isMobile.set(window.innerWidth < breakpoint);
    };
    
    checkMobile();
    window.addEventListener('resize', checkMobile);
    
    return () => {
      window.removeEventListener('resize', checkMobile);
    };
  });
</script>

{#if $isMobile}
  <div class="mobile-layout">
    <slot name="mobile" />
  </div>
{:else}
  <div class="desktop-layout">
    <slot name="desktop" />
  </div>
{/if}

<!-- Usage -->
<ResponsiveLayout>
  <MobileNav slot="mobile" />
  <DesktopNav slot="desktop" />
</ResponsiveLayout>
```

### Container Queries (Modern Approach)

```css
/* Container queries for component-level responsiveness */
.card-container {
  container-type: inline-size;
  container-name: card;
}

.card {
  /* Default mobile layout */
  display: flex;
  flex-direction: column;
}

@container card (min-width: 400px) {
  .card {
    /* Wider container layout */
    flex-direction: row;
    gap: 1rem;
  }
}
```

## Virtual Keyboard Handling

### Dynamic Viewport Height

```svelte
<script>
  import { onMount } from 'svelte';
  
  onMount(() => {
    // Set CSS variable for true viewport height
    function setViewportHeight() {
      const vh = window.innerHeight * 0.01;
      document.documentElement.style.setProperty('--vh', `${vh}px`);
    }
    
    setViewportHeight();
    window.addEventListener('resize', setViewportHeight);
    
    // Detect virtual keyboard
    let lastHeight = window.innerHeight;
    window.addEventListener('resize', () => {
      const currentHeight = window.innerHeight;
      const diff = lastHeight - currentHeight;
      
      if (diff > 100) {
        // Keyboard likely opened
        document.body.classList.add('keyboard-open');
      } else if (diff < -100) {
        // Keyboard likely closed
        document.body.classList.remove('keyboard-open');
      }
      
      lastHeight = currentHeight;
    });
  });
</script>

<style>
  /* Use dynamic viewport height */
  .fullscreen {
    height: calc(var(--vh, 1vh) * 100);
  }
  
  /* Adjust layout when keyboard is open */
  :global(body.keyboard-open) .bottom-bar {
    display: none;
  }
</style>
```

### Input Focus Management

```svelte
<script>
  let inputElement;
  
  function scrollIntoViewOnFocus() {
    // Ensure input is visible when keyboard opens
    setTimeout(() => {
      inputElement?.scrollIntoView({
        behavior: 'smooth',
        block: 'center'
      });
    }, 300); // Delay for keyboard animation
  }
</script>

<input
  bind:this={inputElement}
  on:focus={scrollIntoViewOnFocus}
  type="text"
  inputmode="numeric"
  pattern="[0-9]*"
