# Mobile Accessibility - Mobile-First PWA Design

**Related**: [SKILL.md](../SKILL.md) | [touch-interactions.md](touch-interactions.md) | [responsive-design.md](responsive-design.md)

A comprehensive guide to creating accessible Progressive Web Apps for mobile devices, covering WCAG 2.2 success criteria, screen reader support, keyboard navigation, and testing strategies for assistive technologies.

---

## Table of Contents

1. [WCAG 2.2 Mobile Success Criteria](#wcag-22-mobile-success-criteria)
2. [Touch Target Sizing](#touch-target-sizing)
3. [Pointer Gestures](#pointer-gestures)
4. [Dragging Movements](#dragging-movements)
5. [Orientation Support](#orientation-support)
6. [Focus Management](#focus-management)
7. [Screen Reader Support](#screen-reader-support)
8. [Color Contrast](#color-contrast)
9. [Testing with Assistive Technologies](#testing-with-assistive-technologies)

---

## WCAG 2.2 Mobile Success Criteria

**WCAG 2.2** (October 2023) introduced new success criteria specifically addressing mobile accessibility:

| SC | Level | Name | Mobile Impact |
|-----|-------|------|---------------|
| **2.5.7** | AA | Dragging Movements | Provide alternatives to drag-and-drop |
| **2.5.8** | AA | Target Size (Minimum) | 24×24px minimum touch targets |
| **3.2.6** | A | Consistent Help | Help consistently placed |
| **3.3.7** | A | Redundant Entry | Don't ask for same info twice |
| **3.3.8** | AA | Accessible Authentication | No cognitive function tests |

**Existing criteria critical for mobile**:

| SC | Level | Name | Mobile Impact |
|-----|-------|------|---------------|
| **1.3.4** | AA | Orientation | Support both portrait and landscape |
| **2.5.1** | A | Pointer Gestures | Alternatives to complex gestures |
| **2.5.2** | A | Pointer Cancellation | Prevent accidental activation |
| **2.5.3** | A | Label in Name | Visible label matches accessible name |
| **1.4.10** | AA | Reflow | Content reflows at 320px width |
| **1.4.3** | AA | Contrast (Minimum) | 4.5:1 for text, 3:1 for UI |

---

## Touch Target Sizing

### SC 2.5.8: Target Size (Minimum) - Level AA

**Requirement**: Touch targets must be at least **24×24 CSS pixels**, with exceptions:

- **Spacing**: Undersized targets have ≥24px spacing to other targets
- **Equivalent**: Alternative control exists meeting size requirement
- **Inline**: Target is within a sentence or text block
- **User agent control**: Size determined by browser (e.g., default checkbox)
- **Essential**: Specific size essential to information

**Best Practice**: Use **44×44px** minimum (Apple HIG, Material Design 3)

### Implementation

```css
/* WCAG 2.2 Level AA compliant (24×24px minimum) */
.button-minimum {
  min-width: 24px;
  min-height: 24px;
}

/* Best practice (44×44px recommended) */
.button {
  min-width: 44px;
  min-height: 44px;
  padding: 12px 16px;

  /* Touch optimization */
  touch-action: manipulation;
  -webkit-tap-highlight-color: transparent;
}

/* Icon buttons with adequate padding */
.icon-button {
  min-width: 44px;
  min-height: 44px;
  padding: 10px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
}

.icon-button svg {
  width: 24px;
  height: 24px;
}

/* Spacing exception: Small targets with adequate spacing */
.tag {
  min-width: 20px;
  min-height: 20px;
  padding: 4px 8px;
  margin: 12px; /* 12px + 20px + 12px = 44px total hit area */
}
```

### Svelte Component with ARIA

```svelte
<script lang="ts">
  export let variant: 'primary' | 'secondary' = 'primary';
  export let disabled = false;
  export let ariaLabel: string | undefined = undefined;
</script>

<button
  class="accessible-button {variant}"
  {disabled}
  aria-label={ariaLabel}
  on:click
>
  <slot />
</button>

<style>
  .accessible-button {
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

    /* Accessibility */
    touch-action: manipulation;
    user-select: none;

    /* Transition */
    transition: all 150ms ease;
  }

  .accessible-button.primary {
    background: #007bff;
    color: white;
  }

  .accessible-button.secondary {
    background: transparent;
    color: #007bff;
    border: 2px solid #007bff;
  }

  /* Focus indicator for keyboard/screen reader users */
  .accessible-button:focus-visible {
    outline: 3px solid #007bff;
    outline-offset: 2px;
  }

  /* Active state for touch feedback */
  .accessible-button:active:not(:disabled) {
    transform: scale(0.96);
  }

  .accessible-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
</style>
```

---

## Pointer Gestures

### SC 2.5.1: Pointer Gestures - Level A

**Requirement**: All functionality using **multipoint** or **path-based** gestures must have a **single-pointer alternative**.

**Examples**:
- **Pinch to zoom**: Provide zoom buttons
- **Two-finger swipe**: Provide navigation buttons
- **Draw a shape**: Provide alternative input method

### Implementation

```svelte
<script lang="ts">
  let scale = 1;
  let isPinching = false;
  let startDistance = 0;

  // Touch-based pinch zoom
  function handleTouchStart(e: TouchEvent) {
    if (e.touches.length === 2) {
      isPinching = true;
      startDistance = getDistance(e.touches[0], e.touches[1]);
    }
  }

  function handleTouchMove(e: TouchEvent) {
    if (!isPinching || e.touches.length !== 2) return;

    const currentDistance = getDistance(e.touches[0], e.touches[1]);
    const scaleChange = currentDistance / startDistance;
    scale = Math.max(1, Math.min(3, scale * scaleChange));
    startDistance = currentDistance;
  }

  function handleTouchEnd() {
    isPinching = false;
  }

  function getDistance(touch1: Touch, touch2: Touch) {
    const dx = touch1.clientX - touch2.clientX;
    const dy = touch1.clientY - touch2.clientY;
    return Math.sqrt(dx * dx + dy * dy);
  }

  // Single-pointer alternatives (WCAG 2.5.1 compliant)
  function zoomIn() {
    scale = Math.min(3, scale + 0.25);
  }

  function zoomOut() {
    scale = Math.max(1, scale - 0.25);
  }

  function resetZoom() {
    scale = 1;
  }
</script>

<div class="zoom-container">
  <!-- Zoom controls (single-pointer alternative) -->
  <div class="zoom-controls" role="toolbar" aria-label="Zoom controls">
    <button on:click={zoomOut} aria-label="Zoom out">
      −
    </button>
    <button on:click={resetZoom} aria-label="Reset zoom">
      Reset
    </button>
    <button on:click={zoomIn} aria-label="Zoom in">
      +
    </button>
  </div>

  <!-- Zoomable content -->
  <div
    class="zoomable-content"
    style="transform: scale({scale})"
    on:touchstart={handleTouchStart}
    on:touchmove={handleTouchMove}
    on:touchend={handleTouchEnd}
    role="img"
    aria-label="Zoomable image"
  >
    <img src="/image.jpg" alt="Product" />
  </div>
</div>

<style>
  .zoom-container {
    position: relative;
    overflow: hidden;
  }

  .zoom-controls {
    position: absolute;
    bottom: 16px;
    right: 16px;
    z-index: 10;

    display: flex;
    gap: 8px;
    padding: 8px;
    background: rgba(255, 255, 255, 0.9);
    border-radius: 8px;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  }

  .zoom-controls button {
    min-width: 44px;
    min-height: 44px;
    padding: 8px;
    background: white;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-size: 18px;
    font-weight: bold;
    cursor: pointer;
  }

  .zoom-controls button:focus-visible {
    outline: 2px solid #007bff;
    outline-offset: 2px;
  }

  .zoomable-content {
    transform-origin: center;
    transition: transform 150ms ease;
  }
</style>
```

### Path-Based Gesture Alternative

```svelte
<script lang="ts">
  let items = ['Item 1', 'Item 2', 'Item 3'];

  // Swipe to delete (path-based gesture)
  let startX = 0;
  let currentX = 0;
  let swipingIndex: number | null = null;

  function handleSwipeStart(e: TouchEvent, index: number) {
    startX = e.touches[0].clientX;
    swipingIndex = index;
  }

  function handleSwipeMove(e: TouchEvent) {
    if (swipingIndex === null) return;
    currentX = e.touches[0].clientX - startX;
  }

  function handleSwipeEnd() {
    if (swipingIndex === null) return;

    if (currentX < -100) {
      deleteItem(swipingIndex);
    }

    swipingIndex = null;
    currentX = 0;
  }

  // Single-pointer alternative (WCAG 2.5.1 compliant)
  function deleteItem(index: number) {
    items = items.filter((_, i) => i !== index);
  }
</script>

<ul class="list">
  {#each items as item, index}
    <li class="list-item">
      <div
        class="item-content"
        style="transform: translateX({swipingIndex === index ? currentX : 0}px)"
        on:touchstart={(e) => handleSwipeStart(e, index)}
        on:touchmove={handleSwipeMove}
        on:touchend={handleSwipeEnd}
      >
        <span>{item}</span>

        <!-- Delete button (single-pointer alternative) -->
        <button
          class="delete-button"
          on:click={() => deleteItem(index)}
          aria-label="Delete {item}"
        >
          Delete
        </button>
      </div>
    </li>
  {/each}
</ul>

<style>
  .list-item {
    list-style: none;
    margin-bottom: 8px;
  }

  .item-content {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 16px;
    background: white;
    border-radius: 8px;
    transition: transform 150ms ease;
  }

  .delete-button {
    min-width: 44px;
    min-height: 44px;
    padding: 8px 16px;
    background: #dc3545;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
  }
</style>
```

---

## Dragging Movements

### SC 2.5.7: Dragging Movements - Level AA

**Requirement**: All functionality using **dragging movements** must have a **single-pointer alternative** (unless dragging is essential).

**Examples**:
- **Drag to reorder list**: Provide up/down buttons
- **Slider**: Provide increment/decrement buttons
- **Drag and drop**: Provide select and place buttons

### Accessible Slider

```svelte
<script lang="ts">
  export let value = 50;
  export let min = 0;
  export let max = 100;
  export let step = 1;
  export let label = 'Slider';

  function increment() {
    value = Math.min(max, value + step);
  }

  function decrement() {
    value = Math.max(min, value - step);
  }
</script>

<div class="slider-container">
  <label for="slider" class="slider-label">{label}</label>

  <div class="slider-controls">
    <!-- Single-pointer alternative buttons (WCAG 2.5.7 compliant) -->
    <button
      on:click={decrement}
      aria-label="Decrease {label}"
      disabled={value <= min}
    >
      −
    </button>

    <!-- Native range input (accessible) -->
    <input
      id="slider"
      type="range"
      {min}
      {max}
      {step}
      bind:value
      aria-valuemin={min}
      aria-valuemax={max}
      aria-valuenow={value}
      aria-label={label}
    />

    <button
      on:click={increment}
      aria-label="Increase {label}"
      disabled={value >= max}
    >
      +
    </button>
  </div>

  <output for="slider" class="slider-value">{value}</output>
</div>

<style>
  .slider-container {
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .slider-controls {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .slider-controls button {
    min-width: 44px;
    min-height: 44px;
    padding: 8px;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 4px;
    font-size: 18px;
    font-weight: bold;
    cursor: pointer;
  }

  .slider-controls button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .slider-controls button:focus-visible {
    outline: 2px solid #007bff;
    outline-offset: 2px;
  }

  input[type="range"] {
    flex: 1;
    min-height: 44px; /* Adequate touch target */
  }

  .slider-value {
    font-weight: bold;
    text-align: center;
  }
</style>
```

### Accessible Drag-and-Drop

```svelte
<script lang="ts">
  let items = ['Task 1', 'Task 2', 'Task 3'];
  let selectedIndex: number | null = null;

  // Keyboard/button-based reordering (WCAG 2.5.7 compliant)
  function moveUp(index: number) {
    if (index === 0) return;
    [items[index], items[index - 1]] = [items[index - 1], items[index]];
    items = items;
  }

  function moveDown(index: number) {
    if (index === items.length - 1) return;
    [items[index], items[index + 1]] = [items[index + 1], items[index]];
    items = items;
  }
</script>

<ul class="sortable-list" role="list">
  {#each items as item, index}
    <li class="sortable-item" role="listitem">
      <span class="item-text">{item}</span>

      <!-- Single-pointer alternatives (no dragging required) -->
      <div class="item-controls" role="toolbar" aria-label="Reorder {item}">
        <button
          on:click={() => moveUp(index)}
          disabled={index === 0}
          aria-label="Move {item} up"
        >
          ↑
        </button>
        <button
          on:click={() => moveDown(index)}
          disabled={index === items.length - 1}
          aria-label="Move {item} down"
        >
          ↓
        </button>
      </div>
    </li>
  {/each}
</ul>

<style>
  .sortable-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 12px 16px;
    background: white;
    border: 1px solid #e0e0e0;
    border-radius: 8px;
    margin-bottom: 8px;
  }

  .item-controls {
    display: flex;
    gap: 8px;
  }

  .item-controls button {
    min-width: 44px;
    min-height: 44px;
    padding: 8px;
    background: #007bff;
    color: white;
    border: none;
    border-radius: 4px;
    font-size: 18px;
    cursor: pointer;
  }

  .item-controls button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .item-controls button:focus-visible {
    outline: 2px solid #007bff;
    outline-offset: 2px;
  }
</style>
```

---

## Orientation Support

### SC 1.3.4: Orientation - Level AA

**Requirement**: Content must not restrict viewing to a single orientation (portrait or landscape) unless essential.

**Implementation**:

```css
/* Allow content to adapt to both orientations */
@media (orientation: portrait) {
  .content {
    /* Portrait-specific styles */
    grid-template-columns: 1fr;
  }
}

@media (orientation: landscape) {
  .content {
    /* Landscape-specific styles */
    grid-template-columns: 1fr 1fr;
  }
}

/* Never lock orientation with CSS */
/* DON'T DO THIS */
@media (orientation: landscape) {
  html {
    transform: rotate(-90deg);
    transform-origin: left top;
    /* This locks content to landscape */
  }
}
```

### Detecting Orientation Changes

```svelte
<script lang="ts">
  import { browser } from '$app/environment';
  import { onMount } from 'svelte';

  let orientation: 'portrait' | 'landscape' = 'portrait';

  onMount(() => {
    if (!browser) return;

    function updateOrientation() {
      orientation = window.innerHeight > window.innerWidth ? 'portrait' : 'landscape';
    }

    updateOrientation();

    window.addEventListener('resize', updateOrientation);
    window.addEventListener('orientationchange', updateOrientation);

    return () => {
      window.removeEventListener('resize', updateOrientation);
      window.removeEventListener('orientationchange', updateOrientation);
    };
  });
</script>

<div class="app" class:portrait={orientation === 'portrait'} class:landscape={orientation === 'landscape'}>
  <!-- Content adapts to orientation -->
</div>

<style>
  .app.portrait {
    /* Portrait layout */
  }

  .app.landscape {
    /* Landscape layout */
  }
</style>
```

---

## Focus Management

### Focus Order and Visibility

```svelte
<script lang="ts">
  import { onMount } from 'svelte';

  let modalOpen = false;
  let modalElement: HTMLDivElement;
  let previousFocus: HTMLElement | null = null;

  function openModal() {
    previousFocus = document.activeElement as HTMLElement;
    modalOpen = true;

    // Move focus to modal
    setTimeout(() => {
      const firstFocusable = modalElement.querySelector<HTMLElement>('button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])');
      firstFocusable?.focus();
    }, 0);
  }

  function closeModal() {
    modalOpen = false;

    // Restore focus to previous element
    previousFocus?.focus();
  }

  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Escape') {
      closeModal();
    }

    // Trap focus within modal
    if (e.key === 'Tab') {
      const focusableElements = modalElement.querySelectorAll<HTMLElement>('button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])');
      const firstElement = focusableElements[0];
      const lastElement = focusableElements[focusableElements.length - 1];

      if (e.shiftKey && document.activeElement === firstElement) {
        e.preventDefault();
        lastElement.focus();
      } else if (!e.shiftKey && document.activeElement === lastElement) {
        e.preventDefault();
        firstElement.focus();
      }
    }
  }
</script>

<button on:click={openModal}>Open Modal</button>

{#if modalOpen}
  <div
    class="modal-backdrop"
    on:click={closeModal}
    on:keydown={handleKeydown}
  >
    <div
      bind:this={modalElement}
      class="modal"
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      on:click|stopPropagation
    >
      <h2 id="modal-title">Modal Title</h2>
      <p>Modal content</p>

      <button on:click={closeModal}>Close</button>
    </div>
  </div>
{/if}

<style>
  /* Focus indicator visible on all interactive elements */
  :global(*:focus-visible) {
    outline: 3px solid #007bff;
    outline-offset: 2px;
  }

  .modal {
    background: white;
    padding: 2rem;
    border-radius: 8px;
    max-width: 500px;
  }

  .modal-backdrop {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.5);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
  }
</style>
```

### Skip Links

```svelte
<!-- src/routes/+layout.svelte -->
<a href="#main-content" class="skip-link">
  Skip to main content
</a>

<header>
  <!-- Navigation -->
</header>

<main id="main-content" tabindex="-1">
  <slot />
</main>

<style>
  .skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    background: #007bff;
    color: white;
    padding: 8px 16px;
    text-decoration: none;
    z-index: 100;
  }

  .skip-link:focus {
    top: 0;
  }
</style>
```

---

## Screen Reader Support

### VoiceOver (iOS) and TalkBack (Android)

**Best Practices**:
1. Use semantic HTML (`<button>`, `<nav>`, `<main>`, etc.)
2. Provide `aria-label` for icon-only buttons
3. Use `aria-describedby` for additional context
4. Announce dynamic content with `aria-live`
5. Mark decorative images with `alt=""`

### ARIA Labels and Descriptions

```svelte
<script lang="ts">
  export let loading = false;
  export let items = [];
</script>

<!-- Icon-only button needs aria-label -->
<button aria-label="Search">
  <svg aria-hidden="true"><!-- Search icon --></svg>
</button>

<!-- Form with proper labeling -->
<form>
  <label for="email">
    Email Address
  </label>
  <input
    id="email"
    type="email"
    aria-describedby="email-hint"
    required
  />
  <div id="email-hint" class="hint">
    We'll never share your email with anyone.
  </div>

  <button type="submit" aria-label="Submit form">
    Submit
  </button>
</form>

<!-- Live region for dynamic content -->
<div
  aria-live="polite"
  aria-atomic="true"
  class="sr-only"
>
  {#if loading}
    Loading items...
  {:else if items.length === 0}
    No items found
  {:else}
    {items.length} items loaded
  {/if}
</div>

<style>
  /* Screen reader only class */
  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border-width: 0;
  }
</style>
```

### Loading State Announcements

```svelte
<script lang="ts">
  export let loading = false;
</script>

<button
  on:click
  aria-busy={loading}
  aria-label={loading ? 'Loading...' : 'Submit'}
>
  {#if loading}
    <span class="spinner" aria-hidden="true"></span>
    <span class="sr-only">Loading...</span>
  {:else}
    Submit
  {/if}
</button>

<!-- Live region announces state changes -->
<div aria-live="assertive" aria-atomic="true" class="sr-only">
  {#if loading}
    Submitting form, please wait
  {/if}
</div>
```

### Navigation Landmarks

```svelte
<!-- Proper landmark structure -->
<header role="banner">
  <nav aria-label="Main navigation">
    <!-- Primary navigation -->
  </nav>
</header>

<main role="main">
  <article aria-labelledby="article-title">
    <h1 id="article-title">Article Title</h1>
    <!-- Content -->
  </article>

  <aside aria-label="Related articles">
    <!-- Sidebar -->
  </aside>
</main>

<footer role="contentinfo">
  <nav aria-label="Footer navigation">
    <!-- Footer links -->
  </nav>
</footer>
```

---

## Color Contrast

### SC 1.4.3: Contrast (Minimum) - Level AA

**Requirement**:
- **Text**: 4.5:1 minimum (large text 3:1)
- **UI Components**: 3:1 minimum (focus indicators, form borders, icons)

**Large text**: 18pt (24px) or 14pt (18.66px) bold

### High Contrast for Mobile

Mobile devices are often used outdoors in bright sunlight, requiring higher contrast:

```css
/* Standard contrast (4.5:1) */
.button {
  background: #007bff;
  color: white; /* 4.5:1 contrast */
}

/* Enhanced contrast for mobile (7:1+) */
.text {
  color: #111; /* 15.3:1 contrast on white */
  font-size: 16px;
}

.text-large {
  color: #333; /* 12.6:1 contrast on white */
  font-size: 24px; /* Large text allows 3:1 minimum */
}

/* UI component contrast */
.input {
  border: 2px solid #666; /* 5.7:1 contrast */
}

.input:focus {
  outline: 3px solid #007bff; /* 4.5:1 contrast */
  outline-offset: 2px;
}

/* Icon contrast */
.icon {
  fill: #333; /* 12.6:1 contrast */
}
```

### Dark Mode Contrast

```css
/* Dark mode with high contrast */
@media (prefers-color-scheme: dark) {
  :root {
    --bg-primary: #121212;
    --text-primary: #e0e0e0; /* 13.6:1 contrast */
    --text-secondary: #b0b0b0; /* 9.1:1 contrast */
    --border: #444; /* 5.5:1 contrast */
  }

  body {
    background: var(--bg-primary);
    color: var(--text-primary);
  }

  .button {
    background: #1e88e5; /* Adjusted for dark mode */
    color: white; /* 5.9:1 contrast */
  }
}

/* High contrast mode */
@media (prefers-contrast: high) {
  .text {
    color: #000;
    font-weight: 600; /* Increase weight for visibility */
  }

  .button {
    border: 3px solid currentColor;
  }
}
```

---

## Testing with Assistive Technologies

### iOS VoiceOver Testing

**Enable VoiceOver**:
1. Settings → Accessibility → VoiceOver → On
2. Or triple-click side button (after enabling in Accessibility Shortcut)

**VoiceOver Gestures**:
- **Swipe right/left**: Navigate elements
- **Double tap**: Activate element
- **Two-finger double tap**: Play/pause
- **Rotor**: Two-finger twist to change navigation mode

**Testing Checklist**:
```markdown
## VoiceOver Testing Checklist

### Navigation
- [ ] All interactive elements focusable
- [ ] Focus order logical (top to bottom, left to right)
- [ ] Skip links work correctly
- [ ] Landmarks announced (header, nav, main, footer)

### Content
- [ ] Headings announced with level (Heading Level 1, etc.)
- [ ] Images have alt text (decorative marked with alt="")
- [ ] Links describe destination ("Learn more about X", not "Click here")
- [ ] Forms have labels associated with inputs

### Interactive Elements
- [ ] Buttons announce role and state ("Button, Submit")
- [ ] Form errors announced and associated with fields
- [ ] Loading states announced
- [ ] Dynamic content changes announced (aria-live)

### Gestures
- [ ] All gestures have single-pointer alternatives
- [ ] Custom controls have appropriate ARIA roles
```

### Android TalkBack Testing

**Enable TalkBack**:
1. Settings → Accessibility → TalkBack → On
2. Or volume keys shortcut (after enabling)

**TalkBack Gestures**:
- **Swipe right/left**: Navigate elements
- **Double tap**: Activate element
- **Swipe down then right**: Read from top
- **Local context menu**: Swipe up then down

### Automated Testing with Playwright

```typescript
// tests/accessibility.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('home page should be accessible', async ({ page }) => {
  await page.goto('/');

  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa', 'wcag22aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});

test('form should have proper labels', async ({ page }) => {
  await page.goto('/contact');

  // Check all form inputs have labels
  const inputs = page.locator('input, textarea, select');
  const count = await inputs.count();

  for (let i = 0; i < count; i++) {
    const input = inputs.nth(i);
    const id = await input.getAttribute('id');
    const ariaLabel = await input.getAttribute('aria-label');
    const ariaLabelledby = await input.getAttribute('aria-labelledby');

    // Must have either id with label, aria-label, or aria-labelledby
    const hasLabel = id ? await page.locator(`label[for="${id}"]`).count() > 0 : false;

    expect(hasLabel || ariaLabel || ariaLabelledby).toBeTruthy();
  }
});

test('touch targets should be adequate size', async ({ page }) => {
  await page.goto('/');

  const buttons = page.locator('button, a');
  const count = await buttons.count();

  for (let i = 0; i < count; i++) {
    const button = buttons.nth(i);
    const box = await button.boundingBox();

    if (!box) continue;

    // WCAG 2.2 Level AA: 24×24px minimum
    expect(box.width).toBeGreaterThanOrEqual(24);
    expect(box.height).toBeGreaterThanOrEqual(24);
  }
});
```

### Manual Testing Checklist

```markdown
## Manual Accessibility Testing Checklist

### Keyboard Navigation (Tablets/Bluetooth Keyboards)
- [ ] Tab through all interactive elements
- [ ] Focus indicator visible on all elements
- [ ] No keyboard traps (can escape all widgets)
- [ ] Skip links work
- [ ] Modal focus trapped correctly

### Touch Interaction
- [ ] All touch targets ≥44×44px (best practice)
- [ ] Adequate spacing between targets
- [ ] Visual feedback on touch
- [ ] No hover-only interactions

### Screen Reader
- [ ] Test with VoiceOver (iOS) and TalkBack (Android)
- [ ] All content announced correctly
- [ ] Interactive elements have proper roles
- [ ] Dynamic content changes announced

### Visual
- [ ] Color contrast meets WCAG AA (4.5:1 text, 3:1 UI)
- [ ] Content reflows at 320px width (1.4.10)
- [ ] Text resizes up to 200% without loss of functionality
- [ ] No information conveyed by color alone

### Orientation
- [ ] Content works in both portrait and landscape
- [ ] No orientation locked unless essential

### Forms
- [ ] All inputs have labels
- [ ] Error messages associated with fields
- [ ] Required fields indicated
- [ ] Form submission feedback provided
```

---

## Summary

Mobile accessibility requires attention to:

1. **WCAG 2.2 Criteria**: 24×24px minimum touch targets (SC 2.5.8), alternatives to dragging (SC 2.5.7), orientation support (SC 1.3.4)
2. **Touch Targets**: 44×44px recommended, adequate spacing, focus indicators
3. **Gesture Alternatives**: Single-pointer alternatives for multipoint and path-based gestures
4. **Focus Management**: Logical order, visible indicators, focus trapping in modals
5. **Screen Readers**: Proper ARIA labels, live regions, semantic HTML, landmark regions
6. **Color Contrast**: 4.5:1 for text, 3:1 for UI components, enhanced for outdoor use
7. **Testing**: VoiceOver, TalkBack, automated axe-core tests, manual keyboard navigation

**Related Resources**:
- [touch-interactions.md](touch-interactions.md) - Touch target sizing and gestures
- [responsive-design.md](responsive-design.md) - Responsive layouts and reflow
- [SKILL.md](../SKILL.md) - Mobile-first PWA design principles
