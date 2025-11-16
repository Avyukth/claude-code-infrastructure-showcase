# Responsive Design - Mobile-First CSS Patterns

Complete guide to responsive design with mobile-first CSS, container queries, fluid typography, and adaptive layouts for all device sizes.

## Table of Contents

- [Mobile-First CSS Philosophy](#mobile-first-css-philosophy)
- [Breakpoint Strategy](#breakpoint-strategy)
- [Media Queries Deep Dive](#media-queries-deep-dive)
- [Container Queries](#container-queries)
- [Fluid Typography](#fluid-typography)
- [Responsive Images](#responsive-images)
- [Grid and Flexbox Layouts](#grid-and-flexbox-layouts)
- [Safe Area Insets](#safe-area-insets)
- [Viewport Units and CSS Variables](#viewport-units-and-css-variables)
- [Testing Across Devices](#testing-across-devices)

---

## Mobile-First CSS Philosophy

### Why Mobile-First?

**Traditional Desktop-First** (❌ Avoid):
```css
/* Base styles assume desktop */
.container {
  width: 1200px;
  padding: 2rem;
  display: grid;
  grid-template-columns: 250px 1fr 300px;
}

/* Override everything for mobile */
@media (max-width: 768px) {
  .container {
    width: 100% !important;
    padding: 1rem !important;
    display: block !important;
  }
}
```

**Problems**:
- Requires `!important` to override
- Downloads unnecessary CSS for mobile
- Mobile gets desktop code + overrides (bloat)
- Backwards approach (design for minority)

**Mobile-First** (✅ Recommended):
```css
/* Base: Mobile (default) */
.container {
  width: 100%;
  padding: 1rem;
}

/* Enhancement: Tablet */
@media (min-width: 768px) {
  .container {
    padding: 1.5rem;
    max-width: 720px;
    margin: 0 auto;
  }
}

/* Enhancement: Desktop */
@media (min-width: 1024px) {
  .container {
    padding: 2rem;
    max-width: 960px;
    display: grid;
    grid-template-columns: 250px 1fr 300px;
  }
}
```

**Benefits**:
- ✅ No `!important` needed
- ✅ Smaller CSS payload for mobile
- ✅ Progressive enhancement mindset
- ✅ Design for majority (60%+ mobile traffic)

### Progressive Enhancement Layers

**Layer 1: Mobile Base** (always loaded)
```css
/* Semantic, accessible HTML works without CSS */
.card {
  padding: 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
}

.card-title {
  font-size: 1.125rem;
  font-weight: 600;
  margin: 0 0 0.5rem;
}

.card-content {
  font-size: 0.875rem;
  line-height: 1.5;
}
```

**Layer 2: Tablet Enhancement** (>= 768px)
```css
@media (min-width: 768px) {
  .card {
    padding: 1.5rem;
  }

  .card-title {
    font-size: 1.25rem;
  }
}
```

**Layer 3: Desktop Enhancement** (>= 1024px)
```css
@media (min-width: 1024px) {
  .card {
    padding: 2rem;
    display: grid;
    grid-template-columns: 120px 1fr;
    gap: 1.5rem;
  }

  .card-image {
    grid-row: 1 / 3;
  }
}
```

---

## Breakpoint Strategy

### Standard Breakpoints (Mobile-First)

**Using em units** (respects user font-size preferences):

```css
/* Extra Small: Default mobile (0-575px) */
/* No media query needed - this is the base */

/* Small: Landscape phones (576px+) */
@media (min-width: 36em) { /* 576px / 16px */ }

/* Medium: Tablets (768px+) */
@media (min-width: 48em) { /* 768px / 16px */ }

/* Large: Desktops (992px+) */
@media (min-width: 62em) { /* 992px / 16px */ }

/* Extra Large: Large desktops (1200px+) */
@media (min-width: 75em) { /* 1200px / 16px */ }

/* 2X Large: Ultra-wide (1400px+) */
@media (min-width: 87.5em) { /* 1400px / 16px */ }
```

**Why em over px?**
- Respects user's browser font size settings
- Accessibility: Users who increase font size get appropriate layouts
- More flexible than fixed pixel breakpoints

### Content-Based Breakpoints (2024 Best Practice)

**❌ Device-specific** (outdated):
```css
@media (min-width: 375px) { /* iPhone 13 */ }
@media (min-width: 414px) { /* iPhone 13 Pro Max */ }
@media (min-width: 768px) { /* iPad */ }
```

**✅ Content-based** (recommended):
```css
/* When text becomes uncomfortable to read */
@media (min-width: 30em) { /* ~480px */
  .article-content {
    max-width: 65ch; /* Optimal reading width */
  }
}

/* When layout can benefit from two columns */
@media (min-width: 45em) { /* ~720px */
  .card-grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

/* When three columns fit comfortably */
@media (min-width: 64em) { /* ~1024px */
  .card-grid {
    grid-template-columns: repeat(3, 1fr);
  }
}
```

### CSS Custom Properties for Breakpoints

```css
:root {
  --breakpoint-sm: 36em;
  --breakpoint-md: 48em;
  --breakpoint-lg: 62em;
  --breakpoint-xl: 75em;
}

/* Use in media queries */
@media (min-width: 48em) { /* Can't use var() here */ }

/* Alternative: Define breakpoints in JavaScript */
```

**SvelteKit approach**:
```javascript
// lib/breakpoints.js
export const breakpoints = {
  sm: '36em',
  md: '48em',
  lg: '62em',
  xl: '75em'
};

// Use in components
import { breakpoints } from '$lib/breakpoints';
```

---

## Media Queries Deep Dive

### Basic Syntax (Mobile-First)

```css
/* Mobile base: No media query */
.element {
  font-size: 1rem;
  padding: 1rem;
}

/* Tablet: min-width */
@media (min-width: 48em) {
  .element {
    font-size: 1.125rem;
    padding: 1.5rem;
  }
}

/* Desktop: min-width */
@media (min-width: 62em) {
  .element {
    font-size: 1.25rem;
    padding: 2rem;
  }
}
```

### Range Syntax (Modern)

**Old syntax**:
```css
@media (min-width: 768px) and (max-width: 1023px) {
  /* Tablet only */
}
```

**New range syntax** (Chrome 104+, Safari 16.4+):
```css
@media (768px <= width < 1024px) {
  /* Tablet only - cleaner! */
}

@media (width >= 768px) {
  /* Same as min-width: 768px */
}
```

### Orientation Queries

```css
/* Portrait (height > width) */
@media (orientation: portrait) {
  .hero {
    height: 60vh;
  }
}

/* Landscape (width > height) */
@media (orientation: landscape) {
  .hero {
    height: 100vh;
    display: grid;
    grid-template-columns: 1fr 1fr;
  }
}

/* Common pattern: Landscape phones */
@media (min-width: 568px) and (max-height: 420px) and (orientation: landscape) {
  /* iPhone SE landscape, etc. */
  header {
    position: fixed;
    top: 0;
    height: 48px; /* Reduced for landscape */
  }
}
```

### Pointer and Hover Queries

```css
/* Touch devices (no hover) */
@media (hover: none) and (pointer: coarse) {
  .tooltip {
    /* Show on tap, not hover */
  }

  button {
    min-height: 44px; /* Larger touch targets */
  }
}

/* Devices with precise pointer and hover */
@media (hover: hover) and (pointer: fine) {
  button:hover {
    background: #3b82f6;
  }

  .card:hover {
    transform: translateY(-4px);
    box-shadow: 0 10px 20px rgba(0,0,0,0.1);
  }
}

/* Hybrid devices (touch + mouse) */
@media (hover: hover) and (pointer: coarse) {
  /* Surface with touch + trackpad */
}
```

### Prefers Color Scheme

```css
/* Light mode (default) */
:root {
  --bg: #ffffff;
  --text: #1f2937;
}

/* Dark mode */
@media (prefers-color-scheme: dark) {
  :root {
    --bg: #1f2937;
    --text: #f9fafb;
  }
}

/* Component using theme */
body {
  background: var(--bg);
  color: var(--text);
}
```

### Prefers Reduced Motion

```css
/* Default: Animations enabled */
.card {
  transition: transform 0.3s ease;
}

.card:hover {
  transform: scale(1.05);
}

/* Respect user preference for reduced motion */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

---

## Container Queries

### Why Container Queries?

**Problem with Media Queries**:
```css
/* Card adapts to viewport, not container */
@media (min-width: 600px) {
  .card {
    display: grid;
    grid-template-columns: 150px 1fr;
  }
}

/* Breaks in sidebar (narrow container, wide viewport) */
```

**Solution with Container Queries**:
```css
/* Card adapts to its container */
.card-container {
  container-type: inline-size;
  container-name: card;
}

@container card (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 150px 1fr;
  }
}
```

### Basic Setup

```css
/* Make parent a container */
.sidebar {
  container-type: inline-size;
  /* or 'size' for both inline and block */
}

/* Child responds to container width */
@container (min-width: 300px) {
  .widget {
    padding: 1.5rem;
  }
}

@container (min-width: 500px) {
  .widget {
    display: grid;
    grid-template-columns: 1fr 1fr;
  }
}
```

### Named Containers

```css
/* Multiple containers on same page */
.main-content {
  container-type: inline-size;
  container-name: main;
}

.sidebar {
  container-type: inline-size;
  container-name: sidebar;
}

/* Query specific container */
@container main (min-width: 800px) {
  .article {
    columns: 2;
  }
}

@container sidebar (min-width: 300px) {
  .widget {
    display: block;
  }
}
```

### Container Query Units

```css
.card-container {
  container-type: inline-size;
}

.card {
  /* cqw = 1% of container width */
  padding: 2cqw;

  /* cqh = 1% of container height */
  /* cqi = 1% of container inline size */
  /* cqb = 1% of container block size */
  /* cqmin = smaller of cqi or cqb */
  /* cqmax = larger of cqi or cqb */
}

.card-title {
  /* Fluid font size based on container */
  font-size: clamp(1rem, 2cqi + 0.5rem, 2rem);
}
```

### Complete Container Query Example

```svelte
<div class="product-grid">
  <div class="product-card-container">
    <article class="product-card">
      <img src="/product.jpg" alt="Product" class="product-image" />
      <div class="product-info">
        <h3 class="product-title">Product Name</h3>
        <p class="product-description">Description here...</p>
        <button class="product-cta">Add to Cart</button>
      </div>
    </article>
  </div>
</div>

<style>
  .product-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
    gap: 1rem;
  }

  .product-card-container {
    container-type: inline-size;
  }

  /* Narrow: Vertical stack */
  .product-card {
    display: flex;
    flex-direction: column;
    border: 1px solid #e5e7eb;
    border-radius: 8px;
    overflow: hidden;
  }

  .product-image {
    width: 100%;
    aspect-ratio: 1;
    object-fit: cover;
  }

  .product-info {
    padding: 1rem;
  }

  /* Medium: Horizontal layout */
  @container (min-width: 400px) {
    .product-card {
      flex-direction: row;
    }

    .product-image {
      width: 150px;
      height: 150px;
    }

    .product-info {
      padding: 1.5rem;
    }
  }

  /* Wide: Enhanced spacing */
  @container (min-width: 600px) {
    .product-image {
      width: 200px;
      height: 200px;
    }

    .product-info {
      padding: 2rem;
    }

    .product-title {
      font-size: 1.5rem;
    }
  }
</style>
```

---

## Fluid Typography

### The clamp() Function

**Syntax**: `clamp(min, preferred, max)`

```css
/* Font size scales smoothly between 1rem and 2rem */
h1 {
  font-size: clamp(1rem, 5vw, 2rem);
}

/* Breakdown:
   - min: 1rem (never smaller)
   - preferred: 5vw (scales with viewport)
   - max: 2rem (never larger)
*/
```

### Responsive Type Scale

```css
:root {
  /* Fluid font sizes */
  --font-size-xs:  clamp(0.75rem, 0.7rem + 0.25vw, 0.875rem);
  --font-size-sm:  clamp(0.875rem, 0.8rem + 0.375vw, 1rem);
  --font-size-base: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);
  --font-size-lg:  clamp(1.125rem, 1rem + 0.625vw, 1.25rem);
  --font-size-xl:  clamp(1.25rem, 1.1rem + 0.75vw, 1.5rem);
  --font-size-2xl: clamp(1.5rem, 1.3rem + 1vw, 2rem);
  --font-size-3xl: clamp(1.875rem, 1.6rem + 1.375vw, 2.5rem);
  --font-size-4xl: clamp(2.25rem, 1.9rem + 1.75vw, 3rem);
}

/* Usage */
.text-sm { font-size: var(--font-size-sm); }
.text-base { font-size: var(--font-size-base); }
.text-lg { font-size: var(--font-size-lg); }
h1 { font-size: var(--font-size-4xl); }
h2 { font-size: var(--font-size-3xl); }
```

### Fluid Space Scale

```css
:root {
  --space-xs:  clamp(0.25rem, 0.2rem + 0.25vw, 0.5rem);
  --space-sm:  clamp(0.5rem, 0.4rem + 0.5vw, 1rem);
  --space-md:  clamp(1rem, 0.8rem + 1vw, 1.5rem);
  --space-lg:  clamp(1.5rem, 1.2rem + 1.5vw, 2rem);
  --space-xl:  clamp(2rem, 1.6rem + 2vw, 3rem);
  --space-2xl: clamp(3rem, 2.4rem + 3vw, 4rem);
}

/* Usage */
.container {
  padding: var(--space-md);
  margin-bottom: var(--space-lg);
}

section {
  padding-block: var(--space-2xl);
}
```

### Line Height and Measure

```css
/* Optimal reading experience */
.article-content {
  font-size: clamp(1rem, 0.9rem + 0.5vw, 1.125rem);

  /* Line height: tighter for headings, looser for body */
  line-height: 1.6;

  /* Measure: 45-75 characters per line */
  max-width: 65ch; /* 'ch' = width of '0' character */
}

h1, h2, h3 {
  line-height: 1.2; /* Tighter for headings */
}
```

---

## Responsive Images

### Srcset and Sizes

```html
<!-- Basic responsive image -->
<img
  src="/image-800.jpg"
  srcset="
    /image-400.jpg 400w,
    /image-800.jpg 800w,
    /image-1200.jpg 1200w,
    /image-1600.jpg 1600w
  "
  sizes="
    (max-width: 400px) 400px,
    (max-width: 800px) 800px,
    (max-width: 1200px) 1200px,
    1600px
  "
  alt="Responsive image"
  loading="lazy"
  decoding="async"
/>
```

### Picture Element (Art Direction)

```html
<!-- Different crops for different screen sizes -->
<picture>
  <!-- Mobile: Square crop -->
  <source
    media="(max-width: 767px)"
    srcset="
      /hero-square-400.jpg 400w,
      /hero-square-800.jpg 800w
    "
    sizes="100vw"
  />

  <!-- Tablet: 16:9 crop -->
  <source
    media="(min-width: 768px) and (max-width: 1023px)"
    srcset="
      /hero-16x9-800.jpg 800w,
      /hero-16x9-1200.jpg 1200w
    "
    sizes="100vw"
  />

  <!-- Desktop: Ultra-wide crop -->
  <source
    media="(min-width: 1024px)"
    srcset="
      /hero-wide-1200.jpg 1200w,
      /hero-wide-1600.jpg 1600w,
      /hero-wide-2400.jpg 2400w
    "
    sizes="100vw"
  />

  <!-- Fallback -->
  <img
    src="/hero-16x9-800.jpg"
    alt="Hero image"
    loading="lazy"
  />
</picture>
```

### Modern Image Formats

```html
<!-- AVIF → WebP → JPEG fallback -->
<picture>
  <source
    type="image/avif"
    srcset="/image.avif 1x, /image@2x.avif 2x"
  />
  <source
    type="image/webp"
    srcset="/image.webp 1x, /image@2x.webp 2x"
  />
  <img
    src="/image.jpg"
    srcset="/image.jpg 1x, /image@2x.jpg 2x"
    alt="Modern image formats"
    width="800"
    height="600"
    loading="lazy"
    decoding="async"
  />
</picture>
```

### Background Images (CSS)

```css
/* Responsive background images */
.hero {
  background-image: url('/hero-small.jpg');
  background-size: cover;
  background-position: center;
}

@media (min-width: 768px) {
  .hero {
    background-image: url('/hero-medium.jpg');
  }
}

@media (min-width: 1024px) {
  .hero {
    background-image: url('/hero-large.jpg');
  }
}

/* High DPI displays */
@media (-webkit-min-device-pixel-ratio: 2),
       (min-resolution: 192dpi) {
  .hero {
    background-image: url('/hero-large@2x.jpg');
  }
}
```

---

## Grid and Flexbox Layouts

### Auto-Responsive Grid

```css
/* No media queries needed! */
.grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(280px, 1fr));
  gap: 1rem;
}

/* Breakdown:
   - auto-fit: Fits as many columns as possible
   - minmax(280px, 1fr): Min 280px, max 1fr
   - Automatically wraps to new rows
*/

/* Example: 1 column on mobile, 2-4 on desktop automatically */
```

### Mobile-First Grid

```css
/* Mobile: Single column */
.grid {
  display: grid;
  gap: 1rem;
}

/* Tablet: 2 columns */
@media (min-width: 768px) {
  .grid {
    grid-template-columns: repeat(2, 1fr);
  }
}

/* Desktop: 3 columns */
@media (min-width: 1024px) {
  .grid {
    grid-template-columns: repeat(3, 1fr);
  }
}

/* Large desktop: 4 columns */
@media (min-width: 1280px) {
  .grid {
    grid-template-columns: repeat(4, 1fr);
  }
}
```

### Responsive Flexbox

```css
/* Mobile: Stack vertically */
.flex-container {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

/* Tablet: Horizontal */
@media (min-width: 768px) {
  .flex-container {
    flex-direction: row;
    flex-wrap: wrap;
  }

  .flex-item {
    flex: 1 1 calc(50% - 0.5rem);
  }
}

/* Desktop: More columns */
@media (min-width: 1024px) {
  .flex-item {
    flex: 1 1 calc(33.333% - 0.667rem);
  }
}
```

### Holy Grail Layout

```css
/* Mobile: Stack */
.layout {
  display: grid;
  gap: 1rem;
  grid-template-areas:
    "header"
    "nav"
    "main"
    "aside"
    "footer";
}

/* Desktop: Classic 3-column */
@media (min-width: 1024px) {
  .layout {
    grid-template-areas:
      "header header header"
      "nav    main   aside"
      "footer footer footer";
    grid-template-columns: 200px 1fr 250px;
  }
}

.header { grid-area: header; }
.nav { grid-area: nav; }
.main { grid-area: main; }
.aside { grid-area: aside; }
.footer { grid-area: footer; }
```

---

## Safe Area Insets

### iPhone Notch and Home Indicator

```css
/* Support for notched devices */
:root {
  /* Fallback for non-notched */
  --safe-area-inset-top: 0px;
  --safe-area-inset-right: 0px;
  --safe-area-inset-bottom: 0px;
  --safe-area-inset-left: 0px;
}

/* Update viewport meta tag */
/* <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover"> */

/* Use safe area insets */
header {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  padding-top: env(safe-area-inset-top);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}

.bottom-nav {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}

/* Combine with existing padding */
.container {
  padding: 1rem;
  padding-top: calc(1rem + env(safe-area-inset-top));
  padding-bottom: calc(1rem + env(safe-area-inset-bottom));
}
```

### Fullscreen Content

```css
/* Full-bleed background with safe content */
.hero {
  /* Extend to edges */
  margin-left: calc(-1 * env(safe-area-inset-left));
  margin-right: calc(-1 * env(safe-area-inset-right));

  /* Keep content in safe area */
  padding-left: calc(1rem + env(safe-area-inset-left));
  padding-right: calc(1rem + env(safe-area-inset-right));
}
```

---

## Viewport Units and CSS Variables

### Viewport Units

```css
/* vw = 1% of viewport width */
.full-width {
  width: 100vw;
}

/* vh = 1% of viewport height */
.full-height {
  height: 100vh;
}

/* vmin = 1% of smaller dimension */
.square {
  width: 50vmin;
  height: 50vmin;
}

/* vmax = 1% of larger dimension */

/* New in 2023: Dynamic viewport units */
/* dvh = Dynamic viewport height (accounts for mobile browser chrome) */
.hero {
  height: 100dvh; /* Better than 100vh on mobile */
}

/* svh = Small viewport height (minimum) */
/* lvh = Large viewport height (maximum) */
```

### Responsive CSS Variables

```css
:root {
  /* Mobile values */
  --container-padding: 1rem;
  --heading-size: 2rem;
  --grid-columns: 1;
}

@media (min-width: 768px) {
  :root {
    --container-padding: 1.5rem;
    --heading-size: 2.5rem;
    --grid-columns: 2;
  }
}

@media (min-width: 1024px) {
  :root {
    --container-padding: 2rem;
    --heading-size: 3rem;
    --grid-columns: 3;
  }
}

/* Usage */
.container {
  padding: var(--container-padding);
}

h1 {
  font-size: var(--heading-size);
}

.grid {
  display: grid;
  grid-template-columns: repeat(var(--grid-columns), 1fr);
}
```

---

## Testing Across Devices

### Browser DevTools

**Chrome DevTools**:
1. F12 → Toggle Device Toolbar (Ctrl+Shift+M)
2. Select device presets or set custom dimensions
3. Test throttling (3G, 4G, offline)
4. Capture screenshots for documentation

**Responsive Design Mode (Firefox)**:
1. Ctrl+Shift+M
2. Custom dimensions or device presets
3. Touch simulation
4. Screenshot full page

### Real Device Testing

**Essential Devices**:
- iPhone SE (small phone: 375×667)
- iPhone 13/14 (standard phone: 390×844)
- iPhone 13 Pro Max (large phone: 428×926)
- iPad (tablet portrait: 768×1024)
- iPad Pro (large tablet: 1024×1366)

**Testing Checklist**:
- [ ] Text is readable without zooming
- [ ] Buttons are tappable (44×44px minimum)
- [ ] No horizontal scrolling
- [ ] Images load properly
- [ ] Forms are usable
- [ ] Navigation works on small screens
- [ ] Safe area insets respected

### Automated Testing

```javascript
// Playwright responsive testing
import { test, devices } from '@playwright/test';

test.describe('Responsive design', () => {
  test('renders correctly on mobile', async ({ page }) => {
    await page.setViewportSize(devices['iPhone 13'].viewport);
    await page.goto('/');
    await expect(page).toHaveScreenshot('mobile.png');
  });

  test('renders correctly on tablet', async ({ page }) => {
    await page.setViewportSize(devices['iPad Pro'].viewport);
    await page.goto('/');
    await expect(page).toHaveScreenshot('tablet.png');
  });

  test('renders correctly on desktop', async ({ page }) => {
    await page.setViewportSize({ width: 1920, height: 1080 });
    await page.goto('/');
    await expect(page).toHaveScreenshot('desktop.png');
  });
});
```

---

## Related Files

- [SKILL.md](../SKILL.md) - Main mobile PWA design overview
- [pwa-fundamentals.md](pwa-fundamentals.md) - Service workers and offline
- [touch-interactions.md](touch-interactions.md) - Touch targets and gestures
- [performance-optimization.md](performance-optimization.md) - Image optimization and Core Web Vitals

---

**Pattern Status**: Production-ready responsive design patterns ✅
