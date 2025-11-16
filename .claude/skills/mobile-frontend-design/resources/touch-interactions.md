# Touch Interactions - Mobile-First PWA Design

**Related**: [SKILL.md](../SKILL.md) | [responsive-design.md](responsive-design.md) | [accessibility-mobile.md](accessibility-mobile.md)

A comprehensive guide to designing and implementing touch-optimized interactions for mobile-first Progressive Web Apps, covering WCAG 2.2 accessibility requirements, thumb zone ergonomics, gesture patterns, and mobile navigation UI.

---

## Table of Contents

1. [Touch Target Sizing](#touch-target-sizing)
2. [Thumb Zone Optimization](#thumb-zone-optimization)
3. [Gesture Patterns](#gesture-patterns)
4. [Mobile Navigation Patterns](#mobile-navigation-patterns)
5. [Visual Feedback](#visual-feedback)
6. [Avoiding Hover Dependencies](#avoiding-hover-dependencies)
7. [Input Type Switching](#input-type-switching)
8. [Testing Touch Interactions](#testing-touch-interactions)

---

## Touch Target Sizing

### WCAG 2.2 Success Criterion 2.5.8

**Target Size (Minimum)** - Level AA (October 2023)

The size of the target for pointer inputs must be at least **24×24 CSS pixels**, with exceptions:

- **Spacing**: Undersized targets have at least 24 CSS pixels spacing to every other target
- **Equivalent**: The function can be achieved through a different control meeting this criterion
- **Inline**: The target is in a sentence or block of text
- **User agent control**: The size is determined by the user agent (e.g., browser default buttons)
- **Essential**: A particular presentation is essential to the information conveyed

### Recommended Touch Target Size

While WCAG 2.2 mandates **24×24px minimum**, industry best practices recommend **44×44px** for optimal usability:

- **Apple iOS HIG**: 44×44pt minimum (approximately 44×44 CSS pixels at 1x scale)
- **Material Design 3**: 48×48dp minimum (48×48 CSS pixels)
- **Microsoft**: 44×44px minimum for touch targets

**Why 44×44px?**
- Average adult finger pad: 10-14mm (approximately 40-56 CSS pixels at typical DPI)
- 44×44px provides comfortable margin for error
- Reduces accidental taps on adjacent controls

### Implementation

```css
/* WCAG 2.2 Level AA compliant (24×24px minimum) */
.button-small {
  min-width: 24px;
  min-height: 24px;
  padding: 4px 8px;
}

/* Recommended best practice (44×44px) */
.button {
  min-width: 44px;
  min-height: 44px;
  padding: 12px 16px;
  border: none;
  border-radius: 8px;
  background: #007bff;
  color: white;
  font-size: 16px;
  cursor: pointer;
  touch-action: manipulation; /* Disable double-tap zoom */
}

/* Icon-only buttons */
.icon-button {
  min-width: 44px;
  min-height: 44px;
  padding: 10px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  background: transparent;
}

.icon-button svg {
  width: 24px;
  height: 24px;
}

/* Link touch targets */
a {
  min-height: 44px;
  display: inline-flex;
  align-items: center;
  padding: 8px 0;
  text-decoration: underline;
  color: #007bff;
}

/* Checkbox/radio touch targets */
.checkbox-wrapper {
  min-height: 44px;
  display: flex;
  align-items: center;
  cursor: pointer;
}

.checkbox-wrapper input[type="checkbox"] {
  width: 24px;
  height: 24px;
  margin-right: 12px;
  cursor: pointer;
}

.checkbox-wrapper label {
  flex: 1;
  cursor: pointer;
}
```

### Svelte Component Example

```svelte
<script lang="ts">
  export let variant: 'primary' | 'secondary' = 'primary';
  export let disabled = false;
  export let type: 'button' | 'submit' = 'button';
</script>

<button
  class="touch-button {variant}"
  {disabled}
  {type}
  on:click
>
  <slot />
</button>

<style>
  .touch-button {
    /* WCAG 2.2 compliant touch target */
    min-width: 44px;
    min-height: 44px;
    padding: 12px 24px;

    /* Typography */
    font-size: 16px;
    font-weight: 500;
    line-height: 1.5;

    /* Layout */
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 8px;

    /* Appearance */
    border: none;
    border-radius: 8px;
    cursor: pointer;

    /* Touch optimization */
    touch-action: manipulation;
    user-select: none;
    -webkit-tap-highlight-color: transparent;

    /* Transition */
    transition: all 150ms ease;
  }

  .touch-button.primary {
    background: #007bff;
    color: white;
  }

  .touch-button.secondary {
    background: transparent;
    color: #007bff;
    border: 2px solid #007bff;
  }

  .touch-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .touch-button:active:not(:disabled) {
    transform: scale(0.96);
  }
</style>
```

---

## Thumb Zone Optimization

### Understanding Thumb Zones

On smartphones, users interact primarily with their thumbs. The screen can be divided into three ergonomic zones:

```
┌─────────────────────────────┐
│                             │ ← Dead Zone (hard to reach)
│         DEAD ZONE           │   Avoid primary actions
│                             │
├─────────────────────────────┤
│                             │
│       STRETCH ZONE          │ ← Stretch Zone (requires effort)
│                             │   Secondary actions
│                             │
├─────────────────────────────┤
│                             │
│      EASY REACH ZONE        │ ← Easy Reach Zone (comfortable)
│    [Primary Actions Here]   │   Primary actions, navigation
│                             │
└─────────────────────────────┘
```

**Easy Reach Zone** (Bottom third):
- Most comfortable area for thumb interaction
- Place primary actions, navigation, frequently-used controls
- Bottom navigation bars, floating action buttons

**Stretch Zone** (Middle third):
- Requires slight thumb extension
- Secondary actions, scrollable content
- List items, cards, secondary buttons

**Dead Zone** (Top third, especially top corners):
- Difficult to reach with one-handed use
- Avoid critical interactive elements
- Safe for headers, status indicators, non-interactive content

### Thumb-Friendly Layout Patterns

```svelte
<!-- Mobile app layout with thumb zone optimization -->
<div class="app-layout">
  <!-- Dead zone: Non-interactive header -->
  <header class="app-header">
    <h1>App Title</h1>
    <div class="status-indicators">
      <span>🔔</span>
      <span>👤</span>
    </div>
  </header>

  <!-- Stretch zone: Scrollable content -->
  <main class="app-content">
    <!-- Content cards, lists, etc. -->
  </main>

  <!-- Easy reach zone: Primary navigation -->
  <nav class="bottom-nav">
    <button class="nav-item active">
      <svg><!-- Home icon --></svg>
      <span>Home</span>
    </button>
    <button class="nav-item">
      <svg><!-- Search icon --></svg>
      <span>Search</span>
    </button>
    <button class="nav-item">
      <svg><!-- Profile icon --></svg>
      <span>Profile</span>
    </button>
  </nav>
</div>

<style>
  .app-layout {
    display: flex;
    flex-direction: column;
    min-height: 100dvh; /* Dynamic viewport height */
  }

  /* Dead zone: Top header */
  .app-header {
    position: sticky;
    top: 0;
    padding: 1rem;
    padding-top: calc(1rem + env(safe-area-inset-top));
    background: white;
    border-bottom: 1px solid #e0e0e0;
    z-index: 100;

    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  /* Stretch zone: Scrollable content */
  .app-content {
    flex: 1;
    overflow-y: auto;
    padding: 1rem;
    padding-bottom: calc(80px + env(safe-area-inset-bottom));
  }

  /* Easy reach zone: Bottom navigation */
  .bottom-nav {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;

    display: flex;
    justify-content: space-around;

    background: white;
    border-top: 1px solid #e0e0e0;
    padding: 8px;
    padding-bottom: calc(8px + env(safe-area-inset-bottom));

    z-index: 100;
  }

  .nav-item {
    min-width: 64px;
    min-height: 56px; /* Material Design 3: 56dp for bottom nav */
    padding: 8px 12px;

    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 4px;

    background: transparent;
    border: none;
    border-radius: 16px;
    cursor: pointer;

    color: #666;
    font-size: 12px;
    transition: all 150ms ease;
  }

  .nav-item.active {
    background: rgba(0, 123, 255, 0.1);
    color: #007bff;
  }

  .nav-item svg {
    width: 24px;
    height: 24px;
  }
</style>
```

### Floating Action Button (FAB)

Place FABs in the **bottom-right corner** (or bottom-left for left-handed users) within the easy reach zone:

```svelte
<button class="fab" aria-label="Create new item">
  <svg viewBox="0 0 24 24">
    <path d="M19 13h-6v6h-2v-6H5v-2h6V5h2v6h6v2z"/>
  </svg>
</button>

<style>
  .fab {
    /* Position in easy reach zone */
    position: fixed;
    bottom: 80px; /* Above bottom navigation */
    right: 16px;

    /* Material Design 3: 56×56dp FAB */
    width: 56px;
    height: 56px;

    /* Appearance */
    background: #007bff;
    color: white;
    border: none;
    border-radius: 16px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);

    /* Layout */
    display: flex;
    align-items: center;
    justify-content: center;

    /* Touch optimization */
    cursor: pointer;
    touch-action: manipulation;

    /* Safe area insets */
    bottom: calc(80px + env(safe-area-inset-bottom));
    right: calc(16px + env(safe-area-inset-right));

    /* Animation */
    transition: all 200ms ease;
  }

  .fab:active {
    transform: scale(0.92);
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  }

  .fab svg {
    width: 24px;
    height: 24px;
    fill: currentColor;
  }
</style>
```

---

## Gesture Patterns

### Swipe Gestures

Swipe gestures are intuitive on mobile for actions like:
- Deleting list items
- Navigating carousels
- Dismissing modals
- Revealing actions

**Implementation with Touch Events**:

```svelte
<script lang="ts">
  let startX = 0;
  let currentX = 0;
  let isDragging = false;

  function handleTouchStart(e: TouchEvent) {
    startX = e.touches[0].clientX;
    currentX = 0;
    isDragging = true;
  }

  function handleTouchMove(e: TouchEvent) {
    if (!isDragging) return;
    currentX = e.touches[0].clientX - startX;
  }

  function handleTouchEnd() {
    if (!isDragging) return;

    const swipeThreshold = 100; // pixels

    if (currentX > swipeThreshold) {
      // Swiped right
      console.log('Swiped right');
    } else if (currentX < -swipeThreshold) {
      // Swiped left
      console.log('Swiped left');
    }

    // Reset
    isDragging = false;
    currentX = 0;
  }
</script>

<div
  class="swipeable-item"
  style="transform: translateX({currentX}px)"
  on:touchstart={handleTouchStart}
  on:touchmove={handleTouchMove}
  on:touchend={handleTouchEnd}
>
  <div class="item-content">
    <p>Swipe me left or right</p>
  </div>
</div>

<style>
  .swipeable-item {
    position: relative;
    background: white;
    border-radius: 8px;
    overflow: hidden;
    transition: transform 150ms ease-out;
    touch-action: pan-y; /* Allow vertical scrolling, control horizontal */
  }

  .item-content {
    padding: 1rem;
    min-height: 60px;
    display: flex;
    align-items: center;
  }
</style>
```

**Swipe-to-Delete Pattern**:

```svelte
<script lang="ts">
  export let item: { id: string; title: string };
  export let onDelete: (id: string) => void;

  let startX = 0;
  let currentX = 0;
  let isDragging = false;

  function handleTouchStart(e: TouchEvent) {
    startX = e.touches[0].clientX;
    isDragging = true;
  }

  function handleTouchMove(e: TouchEvent) {
    if (!isDragging) return;
    currentX = e.touches[0].clientX - startX;

    // Only allow left swipe (negative currentX)
    if (currentX > 0) currentX = 0;

    // Max swipe distance
    if (currentX < -80) currentX = -80;
  }

  function handleTouchEnd() {
    if (!isDragging) return;
    isDragging = false;

    if (currentX < -60) {
      // Threshold reached, delete item
      onDelete(item.id);
    } else {
      // Reset position
      currentX = 0;
    }
  }
</script>

<div class="swipe-container">
  <div class="delete-background">
    <svg viewBox="0 0 24 24">
      <path d="M6 19c0 1.1.9 2 2 2h8c1.1 0 2-.9 2-2V7H6v12zM19 4h-3.5l-1-1h-5l-1 1H5v2h14V4z"/>
    </svg>
  </div>

  <div
    class="item-foreground"
    class:dragging={isDragging}
    style="transform: translateX({currentX}px)"
    on:touchstart={handleTouchStart}
    on:touchmove={handleTouchMove}
    on:touchend={handleTouchEnd}
  >
    <p>{item.title}</p>
  </div>
</div>

<style>
  .swipe-container {
    position: relative;
    overflow: hidden;
    border-radius: 8px;
    margin-bottom: 8px;
  }

  .delete-background {
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    width: 80px;
    background: #dc3545;
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .delete-background svg {
    width: 24px;
    height: 24px;
    fill: white;
  }

  .item-foreground {
    position: relative;
    background: white;
    padding: 1rem;
    min-height: 60px;
    display: flex;
    align-items: center;
    touch-action: pan-y;
    transition: transform 150ms ease-out;
  }

  .item-foreground.dragging {
    transition: none; /* Disable transition during drag */
  }
</style>
```

### Pull-to-Refresh

Pull-to-refresh is a common mobile pattern for refreshing content:

```svelte
<script lang="ts">
  let startY = 0;
  let currentY = 0;
  let isPulling = false;
  let isRefreshing = false;

  async function handleTouchStart(e: TouchEvent) {
    // Only start pull if scrolled to top
    if (window.scrollY === 0) {
      startY = e.touches[0].clientY;
      isPulling = true;
    }
  }

  function handleTouchMove(e: TouchEvent) {
    if (!isPulling || isRefreshing) return;

    currentY = e.touches[0].clientY - startY;

    // Only allow downward pull
    if (currentY < 0) currentY = 0;

    // Dampen the pull (diminishing returns)
    if (currentY > 80) {
      currentY = 80 + (currentY - 80) * 0.3;
    }

    // Prevent default scroll when pulling
    if (currentY > 0) {
      e.preventDefault();
    }
  }

  async function handleTouchEnd() {
    if (!isPulling) return;
    isPulling = false;

    const refreshThreshold = 80;

    if (currentY > refreshThreshold && !isRefreshing) {
      isRefreshing = true;

      // Perform refresh action
      await refreshData();

      // Reset after refresh
      isRefreshing = false;
      currentY = 0;
    } else {
      // Reset position
      currentY = 0;
    }
  }

  async function refreshData() {
    // Simulate API call
    await new Promise(resolve => setTimeout(resolve, 1500));
  }
</script>

<div
  class="pull-to-refresh-container"
  on:touchstart={handleTouchStart}
  on:touchmove={handleTouchMove}
  on:touchend={handleTouchEnd}
>
  <div
    class="refresh-indicator"
    class:active={currentY > 0}
    class:refreshing={isRefreshing}
    style="height: {currentY}px"
  >
    <div class="spinner" style="opacity: {Math.min(currentY / 80, 1)}">
      {#if isRefreshing}
        <div class="loading-spinner"></div>
      {:else}
        <svg viewBox="0 0 24 24">
          <path d="M17.65 6.35C16.2 4.9 14.21 4 12 4c-4.42 0-7.99 3.58-7.99 8s3.57 8 7.99 8c3.73 0 6.84-2.55 7.73-6h-2.08c-.82 2.33-3.04 4-5.65 4-3.31 0-6-2.69-6-6s2.69-6 6-6c1.66 0 3.14.69 4.22 1.78L13 11h7V4l-2.35 2.35z"/>
        </svg>
      {/if}
    </div>
  </div>

  <div class="content" style="transform: translateY({currentY}px)">
    <slot />
  </div>
</div>

<style>
  .pull-to-refresh-container {
    position: relative;
    overflow: hidden;
  }

  .refresh-indicator {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    display: flex;
    align-items: flex-end;
    justify-content: center;
    overflow: hidden;
    transition: height 200ms ease;
  }

  .spinner {
    padding-bottom: 16px;
    transition: opacity 150ms ease;
  }

  .spinner svg {
    width: 24px;
    height: 24px;
    fill: #007bff;
  }

  .loading-spinner {
    width: 24px;
    height: 24px;
    border: 3px solid rgba(0, 123, 255, 0.2);
    border-top-color: #007bff;
    border-radius: 50%;
    animation: spin 800ms linear infinite;
  }

  @keyframes spin {
    to { transform: rotate(360deg); }
  }

  .content {
    transition: transform 200ms ease;
  }
</style>
```

### Long Press

Long press reveals contextual actions (like right-click on desktop):

```svelte
<script lang="ts">
  export let onLongPress: () => void;

  let pressTimer: number | undefined;
  let isPressing = false;

  function handleTouchStart(e: TouchEvent) {
    isPressing = true;

    pressTimer = window.setTimeout(() => {
      if (isPressing) {
        // Trigger haptic feedback if available
        if ('vibrate' in navigator) {
          navigator.vibrate(50);
        }
        onLongPress();
      }
    }, 500); // 500ms for long press
  }

  function handleTouchEnd() {
    isPressing = false;
    if (pressTimer) {
      clearTimeout(pressTimer);
    }
  }

  function handleTouchMove() {
    // Cancel long press if finger moves
    isPressing = false;
    if (pressTimer) {
      clearTimeout(pressTimer);
    }
  }
</script>

<div
  class="long-press-item"
  on:touchstart={handleTouchStart}
  on:touchend={handleTouchEnd}
  on:touchmove={handleTouchMove}
>
  <slot />
</div>

<style>
  .long-press-item {
    user-select: none;
    -webkit-user-select: none;
    touch-action: manipulation;
  }
</style>
```

---

## Mobile Navigation Patterns

### Bottom Navigation Bar

Bottom navigation provides easy thumb access to primary app sections (3-5 destinations):

```svelte
<script lang="ts">
  import { page } from '$app/stores';

  const navItems = [
    { label: 'Home', path: '/', icon: 'M10 20v-6h4v6h5v-8h3L12 3 2 12h3v8z' },
    { label: 'Search', path: '/search', icon: 'M15.5 14h-.79l-.28-.27C15.41 12.59 16 11.11 16 9.5 16 5.91 13.09 3 9.5 3S3 5.91 3 9.5 5.91 16 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5 4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5 14 7.01 14 9.5 11.99 14 9.5 14z' },
    { label: 'Notifications', path: '/notifications', icon: 'M12 22c1.1 0 2-.9 2-2h-4c0 1.1.89 2 2 2zm6-6v-5c0-3.07-1.64-5.64-4.5-6.32V4c0-.83-.67-1.5-1.5-1.5s-1.5.67-1.5 1.5v.68C7.63 5.36 6 7.92 6 11v5l-2 2v1h16v-1l-2-2z' },
    { label: 'Profile', path: '/profile', icon: 'M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 3c1.66 0 3 1.34 3 3s-1.34 3-3 3-3-1.34-3-3 1.34-3 3-3zm0 14.2c-2.5 0-4.71-1.28-6-3.22.03-1.99 4-3.08 6-3.08 1.99 0 5.97 1.09 6 3.08-1.29 1.94-3.5 3.22-6 3.22z' },
  ];
</script>

<nav class="bottom-nav" aria-label="Main navigation">
  {#each navItems as item}
    <a
      href={item.path}
      class="nav-item"
      class:active={$page.url.pathname === item.path}
      aria-label={item.label}
      aria-current={$page.url.pathname === item.path ? 'page' : undefined}
    >
      <svg viewBox="0 0 24 24" aria-hidden="true">
        <path d={item.icon} />
      </svg>
      <span class="nav-label">{item.label}</span>
    </a>
  {/each}
</nav>

<style>
  .bottom-nav {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;

    /* Layout */
    display: flex;
    justify-content: space-around;

    /* Material Design 3: Bottom nav height */
    height: 56px;
    padding-bottom: env(safe-area-inset-bottom);

    /* Appearance */
    background: white;
    border-top: 1px solid #e0e0e0;
    box-shadow: 0 -2px 8px rgba(0, 0, 0, 0.05);

    z-index: 100;
  }

  .nav-item {
    /* Touch target */
    min-width: 64px;
    flex: 1;
    max-width: 120px;

    /* Layout */
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 4px;
    padding: 4px 12px;

    /* Typography */
    color: #666;
    font-size: 12px;
    font-weight: 500;
    text-decoration: none;

    /* Touch optimization */
    touch-action: manipulation;
    -webkit-tap-highlight-color: transparent;
    user-select: none;

    /* Transition */
    transition: all 150ms ease;
  }

  .nav-item svg {
    width: 24px;
    height: 24px;
    fill: currentColor;
  }

  .nav-item.active {
    color: #007bff;
  }

  .nav-item:active {
    transform: scale(0.92);
  }

  .nav-label {
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    max-width: 100%;
  }

  /* Desktop: Convert to sidebar */
  @media (min-width: 768px) {
    .bottom-nav {
      position: fixed;
      top: 0;
      bottom: 0;
      left: 0;
      right: auto;

      width: 240px;
      height: auto;
      padding-bottom: 0;

      flex-direction: column;
      justify-content: flex-start;
      padding: 16px 8px;

      border-top: none;
      border-right: 1px solid #e0e0e0;
      box-shadow: 2px 0 8px rgba(0, 0, 0, 0.05);
    }

    .nav-item {
      flex-direction: row;
      justify-content: flex-start;
      gap: 16px;
      padding: 12px 16px;
      border-radius: 8px;
      max-width: none;
    }

    .nav-label {
      font-size: 14px;
    }
  }
</style>
```

### Bottom Sheet

Bottom sheets slide up from the bottom for contextual actions:

```svelte
<script lang="ts">
  export let open = false;
  export let onClose: () => void;

  let sheetElement: HTMLDivElement;
  let startY = 0;
  let currentY = 0;
  let isDragging = false;

  function handleTouchStart(e: TouchEvent) {
    startY = e.touches[0].clientY;
    isDragging = true;
  }

  function handleTouchMove(e: TouchEvent) {
    if (!isDragging) return;

    currentY = e.touches[0].clientY - startY;

    // Only allow downward drag
    if (currentY < 0) currentY = 0;
  }

  function handleTouchEnd() {
    if (!isDragging) return;
    isDragging = false;

    const closeThreshold = 100;

    if (currentY > closeThreshold) {
      onClose();
    }

    currentY = 0;
  }

  function handleBackdropClick() {
    onClose();
  }
</script>

{#if open}
  <div
    class="bottom-sheet-backdrop"
    on:click={handleBackdropClick}
    on:keydown={(e) => e.key === 'Escape' && onClose()}
    role="button"
    tabindex="0"
  >
    <div
      bind:this={sheetElement}
      class="bottom-sheet"
      class:dragging={isDragging}
      style="transform: translateY({currentY}px)"
      on:click|stopPropagation
      on:touchstart={handleTouchStart}
      on:touchmove={handleTouchMove}
      on:touchend={handleTouchEnd}
      role="dialog"
      aria-modal="true"
    >
      <div class="drag-handle"></div>

      <div class="bottom-sheet-content">
        <slot />
      </div>
    </div>
  </div>
{/if}

<style>
  .bottom-sheet-backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    z-index: 1000;

    display: flex;
    align-items: flex-end;

    animation: fadeIn 200ms ease;
  }

  @keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
  }

  .bottom-sheet {
    width: 100%;
    max-height: 80vh;
    background: white;
    border-radius: 16px 16px 0 0;

    display: flex;
    flex-direction: column;

    padding-bottom: env(safe-area-inset-bottom);

    animation: slideUp 300ms ease;
    transition: transform 200ms ease;
  }

  .bottom-sheet.dragging {
    transition: none;
  }

  @keyframes slideUp {
    from { transform: translateY(100%); }
    to { transform: translateY(0); }
  }

  .drag-handle {
    width: 32px;
    height: 4px;
    background: #ccc;
    border-radius: 2px;
    margin: 12px auto 8px;
    cursor: grab;
  }

  .bottom-sheet-content {
    flex: 1;
    overflow-y: auto;
    padding: 16px;
  }
</style>
```

### Hamburger Menu

For apps with many navigation options, a hamburger menu provides access:

```svelte
<script lang="ts">
  export let open = false;

  function toggleMenu() {
    open = !open;
  }

  function closeMenu() {
    open = false;
  }
</script>

<div class="hamburger-container">
  <!-- Hamburger button -->
  <button
    class="hamburger-button"
    aria-label={open ? 'Close menu' : 'Open menu'}
    aria-expanded={open}
    on:click={toggleMenu}
  >
    <div class="hamburger-icon" class:open>
      <span></span>
      <span></span>
      <span></span>
    </div>
  </button>

  <!-- Menu overlay -->
  {#if open}
    <div
      class="menu-backdrop"
      on:click={closeMenu}
      on:keydown={(e) => e.key === 'Escape' && closeMenu()}
      role="button"
      tabindex="0"
    >
      <nav
        class="menu-drawer"
        on:click|stopPropagation
        aria-label="Main navigation"
      >
        <slot />
      </nav>
    </div>
  {/if}
</div>

<style>
  .hamburger-button {
    /* Touch target */
    width: 44px;
    height: 44px;
    padding: 10px;

    /* Layout */
    display: flex;
    align-items: center;
    justify-content: center;

    /* Appearance */
    background: transparent;
    border: none;
    border-radius: 8px;
    cursor: pointer;

    /* Touch optimization */
    touch-action: manipulation;
    -webkit-tap-highlight-color: transparent;
  }

  .hamburger-icon {
    width: 24px;
    height: 18px;
    position: relative;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
  }

  .hamburger-icon span {
    display: block;
    height: 2px;
    background: currentColor;
    border-radius: 2px;
    transition: all 300ms ease;
  }

  /* Animated hamburger to X */
  .hamburger-icon.open span:nth-child(1) {
    transform: translateY(8px) rotate(45deg);
  }

  .hamburger-icon.open span:nth-child(2) {
    opacity: 0;
  }

  .hamburger-icon.open span:nth-child(3) {
    transform: translateY(-8px) rotate(-45deg);
  }

  .menu-backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    z-index: 1000;
    animation: fadeIn 200ms ease;
  }

  @keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
  }

  .menu-drawer {
    position: fixed;
    top: 0;
    left: 0;
    bottom: 0;
    width: 280px;
    max-width: 85vw;

    background: white;
    box-shadow: 2px 0 8px rgba(0, 0, 0, 0.1);

    overflow-y: auto;
    padding: 1rem;
    padding-top: calc(1rem + env(safe-area-inset-top));
    padding-left: calc(1rem + env(safe-area-inset-left));

    animation: slideInLeft 300ms ease;
  }

  @keyframes slideInLeft {
    from { transform: translateX(-100%); }
    to { transform: translateX(0); }
  }
</style>
```

---

## Visual Feedback

### Touch State Feedback

Provide immediate visual feedback for all touch interactions:

```css
/* Active state (while touching) */
.button:active {
  transform: scale(0.96);
  background: #0056b3;
}

/* Ripple effect (Material Design) */
.button {
  position: relative;
  overflow: hidden;
}

.button::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.3);
  transform: translate(-50%, -50%);
  transition: width 0.6s, height 0.6s;
}

.button:active::before {
  width: 300px;
  height: 300px;
}

/* Focus visible for keyboard/accessibility */
.button:focus-visible {
  outline: 2px solid #007bff;
  outline-offset: 2px;
}

/* Disable tap highlight color (use custom feedback) */
.button {
  -webkit-tap-highlight-color: transparent;
}
```

**Svelte Ripple Component**:

```svelte
<script lang="ts">
  export let disabled = false;

  let ripples: Array<{ x: number; y: number; id: number }> = [];
  let nextId = 0;

  function createRipple(e: MouseEvent | TouchEvent) {
    if (disabled) return;

    const button = e.currentTarget as HTMLElement;
    const rect = button.getBoundingClientRect();

    let x: number, y: number;

    if (e instanceof TouchEvent) {
      x = e.touches[0].clientX - rect.left;
      y = e.touches[0].clientY - rect.top;
    } else {
      x = e.clientX - rect.left;
      y = e.clientY - rect.top;
    }

    const id = nextId++;
    ripples = [...ripples, { x, y, id }];

    setTimeout(() => {
      ripples = ripples.filter(r => r.id !== id);
    }, 600);
  }
</script>

<button
  class="ripple-button"
  {disabled}
  on:click
  on:mousedown={createRipple}
  on:touchstart={createRipple}
>
  <span class="button-content">
    <slot />
  </span>

  {#each ripples as ripple (ripple.id)}
    <span
      class="ripple"
      style="left: {ripple.x}px; top: {ripple.y}px"
    ></span>
  {/each}
</button>

<style>
  .ripple-button {
    position: relative;
    overflow: hidden;

    min-width: 44px;
    min-height: 44px;
    padding: 12px 24px;

    background: #007bff;
    color: white;
    border: none;
    border-radius: 8px;

    font-size: 16px;
    font-weight: 500;

    cursor: pointer;
    touch-action: manipulation;
    -webkit-tap-highlight-color: transparent;

    transition: all 150ms ease;
  }

  .ripple-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .button-content {
    position: relative;
    z-index: 1;
  }

  .ripple {
    position: absolute;
    background: rgba(255, 255, 255, 0.4);
    border-radius: 50%;
    transform: translate(-50%, -50%);
    pointer-events: none;

    animation: ripple-animation 600ms ease-out;
  }

  @keyframes ripple-animation {
    from {
      width: 0;
      height: 0;
      opacity: 1;
    }
    to {
      width: 300px;
      height: 300px;
      opacity: 0;
    }
  }
</style>
```

### Loading States

Provide feedback during async operations:

```svelte
<script lang="ts">
  export let loading = false;
  export let disabled = false;
</script>

<button
  class="loading-button"
  disabled={disabled || loading}
  on:click
>
  {#if loading}
    <span class="spinner"></span>
  {/if}
  <span class="button-text" class:loading>
    <slot />
  </span>
</button>

<style>
  .loading-button {
    min-width: 44px;
    min-height: 44px;
    padding: 12px 24px;

    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 8px;

    background: #007bff;
    color: white;
    border: none;
    border-radius: 8px;

    font-size: 16px;
    cursor: pointer;
    touch-action: manipulation;

    transition: all 150ms ease;
  }

  .loading-button:disabled {
    opacity: 0.7;
    cursor: not-allowed;
  }

  .spinner {
    width: 16px;
    height: 16px;
    border: 2px solid rgba(255, 255, 255, 0.3);
    border-top-color: white;
    border-radius: 50%;
    animation: spin 600ms linear infinite;
  }

  @keyframes spin {
    to { transform: rotate(360deg); }
  }

  .button-text.loading {
    opacity: 0.7;
  }
</style>
```

---

## Avoiding Hover Dependencies

### The Problem

Hover states don't exist on touch devices. Users can't "hover" to reveal actions or tooltips.

**Bad pattern**:
```css
/* Hidden by default, shown on hover */
.action-menu {
  display: none;
}

.card:hover .action-menu {
  display: block; /* Touch users can't access this! */
}
```

### Solutions

**1. Always-Visible Actions**:
```css
/* Always show critical actions */
.action-menu {
  display: flex;
  gap: 8px;
}
```

**2. Touch-Activated Menus**:
```svelte
<script lang="ts">
  let menuOpen = false;
</script>

<div class="card">
  <p>Card content</p>

  <button
    class="menu-button"
    aria-label="More actions"
    on:click={() => menuOpen = !menuOpen}
  >
    ⋮
  </button>

  {#if menuOpen}
    <div class="action-menu">
      <button>Edit</button>
      <button>Delete</button>
    </div>
  {/if}
</div>
```

**3. Long Press for Context Menu** (shown earlier in Gesture Patterns)

**4. Feature Detection**:
```svelte
<script lang="ts">
  import { browser } from '$app/environment';

  let hasFinePointer = false;

  if (browser) {
    hasFinePointer = window.matchMedia('(pointer: fine)').matches;
  }
</script>

{#if hasFinePointer}
  <!-- Show hover-based UI for mouse users -->
{:else}
  <!-- Show touch-based UI for touch users -->
{/if}
```

---

## Input Type Switching

Detect input type and optimize UI accordingly:

```svelte
<script lang="ts">
  import { browser } from '$app/environment';
  import { writable } from 'svelte/store';

  const inputType = writable<'touch' | 'mouse'>('touch');

  if (browser) {
    // Detect fine pointer (mouse, trackpad)
    const hasFinePointer = window.matchMedia('(pointer: fine)').matches;
    inputType.set(hasFinePointer ? 'mouse' : 'touch');

    // Listen for pointer type changes (hybrid devices)
    window.addEventListener('touchstart', () => inputType.set('touch'), { passive: true });
    window.addEventListener('mousemove', () => inputType.set('mouse'), { once: true });
  }
</script>

<div class="app" class:touch-mode={$inputType === 'touch'}>
  <!-- UI adapts based on input type -->
</div>

<style>
  /* Larger touch targets for touch mode */
  .app.touch-mode .button {
    min-height: 44px;
    padding: 12px 16px;
  }

  /* Smaller, more compact for mouse mode */
  .app:not(.touch-mode) .button {
    min-height: 32px;
    padding: 8px 12px;
  }
</style>
```

**CSS Media Queries for Pointer Types**:

```css
/* Fine pointer (mouse, trackpad) */
@media (pointer: fine) {
  .button {
    min-height: 32px;
    padding: 8px 16px;
  }

  .button:hover {
    background: #0056b3;
  }
}

/* Coarse pointer (touch) */
@media (pointer: coarse) {
  .button {
    min-height: 44px;
    padding: 12px 16px;
  }

  /* No hover states for touch */
  .button:active {
    background: #0056b3;
  }
}

/* Any pointer (hybrid devices) */
@media (any-pointer: coarse) {
  /* Optimize for touch even if mouse is also available */
  .button {
    min-height: 44px;
  }
}
```

---

## Testing Touch Interactions

### Chrome DevTools

**Enable Device Mode**:
1. Open Chrome DevTools (F12 or Cmd+Opt+I)
2. Click "Toggle device toolbar" (Cmd+Shift+M)
3. Select a mobile device preset or set custom dimensions
4. Enable "Show device frame" to see notches/safe areas

**Touch Simulation**:
- DevTools automatically simulates touch events
- Click and drag to simulate swipes
- Cmd+Shift+M to toggle between mouse and touch

**Throttling**:
- Simulate slow networks (3G, 4G)
- CPU throttling for low-end devices

### Real Device Testing

**iOS Testing** (iPhone/iPad):
1. Enable Web Inspector:
   - Settings → Safari → Advanced → Web Inspector
2. Connect device via USB
3. Safari on Mac → Develop → [Your Device] → [Your Page]

**Android Testing**:
1. Enable USB Debugging:
   - Settings → About → Tap "Build number" 7 times
   - Settings → Developer Options → USB Debugging
2. Connect device via USB
3. Chrome on desktop → chrome://inspect → Select device

### Touch Event Testing Checklist

```markdown
## Touch Interaction Testing Checklist

### Touch Targets
- [ ] All interactive elements ≥44×44px
- [ ] Adequate spacing between targets (≥8px)
- [ ] No overlapping touch areas
- [ ] Icon buttons have sufficient padding

### Gestures
- [ ] Swipe gestures work smoothly
- [ ] Pull-to-refresh triggers correctly
- [ ] Long press reveals context menu
- [ ] Pinch-to-zoom disabled where appropriate

### Visual Feedback
- [ ] Touch feedback visible (<100ms delay)
- [ ] Loading states shown during async operations
- [ ] Active states for all buttons/links
- [ ] No reliance on hover states

### Navigation
- [ ] Bottom navigation accessible with thumb
- [ ] Primary actions in easy reach zone
- [ ] Hamburger menu slides smoothly
- [ ] Safe area insets respected

### Performance
- [ ] 60fps scrolling and animations
- [ ] No jank during touch interactions
- [ ] Touch events don't block scrolling
- [ ] Passive event listeners used

### Accessibility
- [ ] Screen reader announces touch targets
- [ ] Focus visible for keyboard navigation
- [ ] Touch targets labeled with aria-label
- [ ] Gestures have keyboard alternatives
```

### Automated Testing with Playwright

```typescript
// tests/touch-interactions.spec.ts
import { test, expect, devices } from '@playwright/test';

test.use({
  ...devices['iPhone 13'],
});

test('bottom navigation is touch-friendly', async ({ page }) => {
  await page.goto('/');

  // Check touch target size
  const navItem = page.locator('.nav-item').first();
  const box = await navItem.boundingBox();

  expect(box?.width).toBeGreaterThanOrEqual(44);
  expect(box?.height).toBeGreaterThanOrEqual(44);

  // Test tap
  await navItem.tap();
  await expect(page).toHaveURL('/home');
});

test('swipe to delete list item', async ({ page }) => {
  await page.goto('/list');

  const listItem = page.locator('.list-item').first();
  const box = await listItem.boundingBox();

  if (!box) throw new Error('List item not found');

  // Swipe left
  await page.touchscreen.tap(box.x + box.width / 2, box.y + box.height / 2);
  await page.touchscreen.swipe(
    { x: box.x + box.width - 10, y: box.y + box.height / 2 },
    { x: box.x + 10, y: box.y + box.height / 2 }
  );

  // Verify item deleted
  await expect(listItem).not.toBeVisible();
});

test('pull to refresh works', async ({ page }) => {
  await page.goto('/feed');

  // Pull down from top
  await page.touchscreen.swipe(
    { x: 200, y: 100 },
    { x: 200, y: 300 }
  );

  // Verify refresh indicator
  await expect(page.locator('.loading-spinner')).toBeVisible();
  await expect(page.locator('.loading-spinner')).not.toBeVisible({ timeout: 3000 });
});
```

---

## Summary

Touch interactions are fundamental to mobile-first PWA design. Follow these principles:

1. **WCAG 2.2 Compliance**: Minimum 24×24px touch targets, recommended 44×44px
2. **Thumb Zone Optimization**: Primary actions in easy reach zone (bottom third)
3. **Gesture Support**: Swipe, pull-to-refresh, long press for enhanced UX
4. **Mobile Navigation**: Bottom nav, bottom sheets, hamburger menus
5. **Visual Feedback**: Immediate touch feedback, ripple effects, loading states
6. **No Hover Dependencies**: Always-visible actions or touch-activated menus
7. **Input Detection**: Adapt UI based on pointer type (touch vs mouse)
8. **Comprehensive Testing**: DevTools, real devices, automated tests

**Related Resources**:
- [responsive-design.md](responsive-design.md) - Mobile-first CSS patterns
- [accessibility-mobile.md](accessibility-mobile.md) - WCAG 2.2 mobile criteria
- [pwa-fundamentals.md](pwa-fundamentals.md) - Service workers and offline functionality
