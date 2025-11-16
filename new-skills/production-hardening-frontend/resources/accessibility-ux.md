# Accessibility and User Experience - SvelteKit Production

**WCAG 2.1 AA compliant accessibility patterns for SvelteKit applications**

This resource provides comprehensive accessibility and user experience patterns for SvelteKit applications, ensuring inclusive design and compliance with international accessibility standards.

---

## WCAG 2.1 Compliance Overview

### WCAG Conformance Levels

| Level | Description | Required For |
|-------|-------------|--------------|
| **A** | Minimum accessibility | Legal baseline in many countries |
| **AA** | Recommended standard | Most government/enterprise requirements |
| **AAA** | Enhanced accessibility | Specialized use cases |

**Target**: WCAG 2.1 AA compliance for all production applications.

---

## 1. Semantic HTML and Structure

### Proper Document Structure

```svelte
<!-- ✅ GOOD: Semantic HTML -->
<header>
  <nav aria-label="Main navigation">
    <ul>
      <li><a href="/">Home</a></li>
      <li><a href="/about">About</a></li>
      <li><a href="/contact">Contact</a></li>
    </ul>
  </nav>
</header>

<main id="main-content">
  <article>
    <h1>Page Title</h1>
    <section>
      <h2>Section Heading</h2>
      <p>Content...</p>
    </section>
  </article>
</main>

<footer>
  <p>&copy; 2025 Company Name</p>
</footer>
```

```svelte
<!-- ❌ BAD: Non-semantic divs -->
<div class="header">
  <div class="nav">
    <div class="nav-item">Home</div>
  </div>
</div>

<div class="content">
  <div class="title">Page Title</div>
  <div class="section">Content...</div>
</div>
```

### Heading Hierarchy

```svelte
<script lang="ts">
  // ✅ GOOD: Proper heading levels
</script>

<h1>Main Page Title</h1>

<section>
  <h2>Primary Section</h2>
  <p>Content...</p>

  <h3>Subsection</h3>
  <p>More content...</p>
</section>

<section>
  <h2>Another Primary Section</h2>
  <!-- Never skip heading levels (h2 → h4) -->
</section>
```

### Landmarks and Regions

```svelte
<!-- Main content landmark -->
<main id="main-content">
  <h1>Dashboard</h1>
</main>

<!-- Navigation landmark -->
<nav aria-label="Main navigation">
  <ul>...</ul>
</nav>

<!-- Search landmark -->
<search>
  <form role="search">
    <label for="search-input">Search</label>
    <input id="search-input" type="search" />
  </form>
</search>

<!-- Complementary content -->
<aside aria-label="Related articles">
  <h2>Related Content</h2>
</aside>

<!-- Form landmark -->
<form aria-label="Contact form">
  <label for="name">Name</label>
  <input id="name" type="text" />
</form>
```

---

## 2. Keyboard Navigation

### Focus Management

```svelte
<script lang="ts">
  import { onMount } from 'svelte';

  let dialogOpen = $state(false);
  let dialogElement: HTMLDialogElement;
  let previousFocus: HTMLElement | null = null;

  function openDialog() {
    previousFocus = document.activeElement as HTMLElement;
    dialogOpen = true;

    // Focus first focusable element in dialog
    setTimeout(() => {
      const firstInput = dialogElement.querySelector('input, button') as HTMLElement;
      firstInput?.focus();
    }, 0);
  }

  function closeDialog() {
    dialogOpen = false;

    // Return focus to trigger element
    previousFocus?.focus();
  }

  // Trap focus within dialog
  function handleKeydown(e: KeyboardEvent) {
    if (e.key === 'Escape') {
      closeDialog();
    }

    if (e.key === 'Tab') {
      const focusableElements = dialogElement.querySelectorAll(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      );

      const first = focusableElements[0] as HTMLElement;
      const last = focusableElements[focusableElements.length - 1] as HTMLElement;

      if (e.shiftKey && document.activeElement === first) {
        last.focus();
        e.preventDefault();
      } else if (!e.shiftKey && document.activeElement === last) {
        first.focus();
        e.preventDefault();
      }
    }
  }
</script>

<button onclick={openDialog}>Open Dialog</button>

{#if dialogOpen}
  <dialog
    bind:this={dialogElement}
    open
    onkeydown={handleKeydown}
    aria-labelledby="dialog-title"
    aria-modal="true"
  >
    <h2 id="dialog-title">Dialog Title</h2>
    <p>Dialog content...</p>

    <button onclick={closeDialog}>Close</button>
  </dialog>
{/if}
```

### Skip Links

```svelte
<!-- src/routes/+layout.svelte -->
<a href="#main-content" class="skip-link">
  Skip to main content
</a>

<nav>
  <!-- Navigation items -->
</nav>

<main id="main-content" tabindex="-1">
  <!-- Main content -->
</main>

<style>
  .skip-link {
    position: absolute;
    top: -40px;
    left: 0;
    background: #000;
    color: #fff;
    padding: 8px;
    text-decoration: none;
    z-index: 100;
  }

  .skip-link:focus {
    top: 0;
  }
</style>
```

### Custom Interactive Components

```svelte
<script lang="ts">
  // Accessible custom button
  let count = $state(0);

  function increment() {
    count++;
  }

  function handleKeydown(e: KeyboardEvent) {
    // Space or Enter should activate button
    if (e.key === ' ' || e.key === 'Enter') {
      e.preventDefault();
      increment();
    }
  }
</script>

<!-- ✅ GOOD: Native button (preferred) -->
<button onclick={increment}>
  Increment: {count}
</button>

<!-- ✅ ACCEPTABLE: Custom button with proper ARIA -->
<div
  role="button"
  tabindex="0"
  onclick={increment}
  onkeydown={handleKeydown}
  aria-label="Increment counter"
>
  Increment: {count}
</div>

<!-- ❌ BAD: Div without keyboard support -->
<div onclick={increment}>
  Increment: {count}
</div>
```

---

## 3. ARIA Roles, States, and Properties

### Form Validation with ARIA

```svelte
<script lang="ts">
  let email = $state('');
  let emailError = $state('');

  function validateEmail() {
    if (!email.includes('@')) {
      emailError = 'Please enter a valid email address';
    } else {
      emailError = '';
    }
  }
</script>

<form>
  <label for="email">
    Email Address
    <span aria-label="required">*</span>
  </label>

  <input
    id="email"
    type="email"
    bind:value={email}
    onblur={validateEmail}
    aria-required="true"
    aria-invalid={!!emailError}
    aria-describedby={emailError ? 'email-error' : undefined}
  />

  {#if emailError}
    <span id="email-error" role="alert" class="error">
      {emailError}
    </span>
  {/if}
</form>
```

### Loading States

```svelte
<script lang="ts">
  let loading = $state(false);
  let data = $state<any>(null);

  async function loadData() {
    loading = true;

    try {
      const response = await fetch('/api/data');
      data = await response.json();
    } finally {
      loading = false;
    }
  }
</script>

<button onclick={loadData} aria-busy={loading} disabled={loading}>
  {loading ? 'Loading...' : 'Load Data'}
</button>

{#if loading}
  <div role="status" aria-live="polite">
    Loading data, please wait...
  </div>
{/if}

{#if data}
  <div role="region" aria-label="Data results">
    <!-- Display data -->
  </div>
{/if}
```

### Accordion (Expandable Sections)

```svelte
<script lang="ts">
  let sections = $state([
    { id: 1, title: 'Section 1', content: 'Content 1', expanded: false },
    { id: 2, title: 'Section 2', content: 'Content 2', expanded: false },
    { id: 3, title: 'Section 3', content: 'Content 3', expanded: false }
  ]);

  function toggle(id: number) {
    sections = sections.map(s =>
      s.id === id ? { ...s, expanded: !s.expanded } : s
    );
  }
</script>

<div class="accordion">
  {#each sections as section (section.id)}
    <div class="accordion-item">
      <h3>
        <button
          onclick={() => toggle(section.id)}
          aria-expanded={section.expanded}
          aria-controls="section-{section.id}"
          id="accordion-header-{section.id}"
        >
          {section.title}
          <span aria-hidden="true">
            {section.expanded ? '−' : '+'}
          </span>
        </button>
      </h3>

      {#if section.expanded}
        <div
          id="section-{section.id}"
          role="region"
          aria-labelledby="accordion-header-{section.id}"
        >
          <p>{section.content}</p>
        </div>
      {/if}
    </div>
  {/each}
</div>
```

### Live Regions for Dynamic Content

```svelte
<script lang="ts">
  let notifications = $state<string[]>([]);

  function addNotification(message: string) {
    notifications = [...notifications, message];

    // Remove after 5 seconds
    setTimeout(() => {
      notifications = notifications.filter(n => n !== message);
    }, 5000);
  }
</script>

<!-- Polite announcements (don't interrupt) -->
<div aria-live="polite" aria-atomic="true" class="sr-only">
  {#if notifications.length > 0}
    {notifications[notifications.length - 1]}
  {/if}
</div>

<!-- Assertive announcements (immediate) -->
<div aria-live="assertive" role="alert" class="sr-only">
  <!-- Critical errors only -->
</div>

<style>
  /* Screen reader only */
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

---

## 4. Color Contrast and Readability

### WCAG AA Contrast Requirements

| Text Size | Normal Text | Large Text (18pt+) |
|-----------|-------------|-------------------|
| **Minimum Ratio** | 4.5:1 | 3:1 |

### Accessible Color Palette

```css
/* src/app.css */
:root {
  /* ✅ WCAG AA compliant colors */
  --color-primary: #0066cc; /* Blue - 4.54:1 on white */
  --color-primary-dark: #004a99; /* Darker blue - 7.52:1 on white */
  --color-success: #008000; /* Green - 4.69:1 on white */
  --color-error: #c00000; /* Red - 5.75:1 on white */
  --color-warning: #805000; /* Orange - 4.52:1 on white */

  --color-text: #1a1a1a; /* Near black - 15.8:1 on white */
  --color-text-secondary: #4d4d4d; /* Gray - 9.73:1 on white */

  --color-background: #ffffff;
  --color-surface: #f5f5f5;
}

/* ❌ BAD: Insufficient contrast */
.low-contrast {
  color: #999999; /* Only 2.85:1 - fails WCAG AA */
  background: #ffffff;
}

/* ✅ GOOD: Sufficient contrast */
.high-contrast {
  color: #4d4d4d; /* 9.73:1 - passes WCAG AAA */
  background: #ffffff;
}
```

### Don't Rely on Color Alone

```svelte
<!-- ❌ BAD: Color only -->
<span style="color: red;">Error</span>
<span style="color: green;">Success</span>

<!-- ✅ GOOD: Color + icon + text -->
<span class="error">
  <svg aria-hidden="true"><!-- Error icon --></svg>
  <span class="sr-only">Error:</span>
  Invalid input
</span>

<span class="success">
  <svg aria-hidden="true"><!-- Success icon --></svg>
  <span class="sr-only">Success:</span>
  Form submitted
</span>
```

### Focus Indicators

```css
/* ✅ GOOD: Visible focus indicator */
button:focus,
a:focus,
input:focus {
  outline: 3px solid #0066cc;
  outline-offset: 2px;
}

/* ❌ NEVER remove focus outline without replacement */
/* button:focus { outline: none; } */

/* ✅ Acceptable: Custom focus style */
button:focus {
  outline: none;
  box-shadow: 0 0 0 3px rgba(0, 102, 204, 0.5);
}
```

---

## 5. Screen Reader Support

### Descriptive Link Text

```svelte
<!-- ❌ BAD: Non-descriptive links -->
<a href="/article/123">Click here</a>
<a href="/download">Read more</a>

<!-- ✅ GOOD: Descriptive links -->
<a href="/article/123">Read the full article about accessibility</a>
<a href="/download">Download the accessibility guidelines PDF</a>

<!-- ✅ GOOD: Context provided via aria-label -->
<a href="/article/123" aria-label="Read more about WCAG 2.1 compliance">
  Read more
</a>
```

### Image Alternative Text

```svelte
<!-- Informative image -->
<img
  src="/product.jpg"
  alt="Red running shoes with white laces, size 10"
/>

<!-- Decorative image (no alt needed) -->
<img src="/decorative-border.png" alt="" role="presentation" />

<!-- Complex image with detailed description -->
<figure>
  <img
    src="/chart.png"
    alt="Bar chart showing sales increase"
    aria-describedby="chart-description"
  />
  <figcaption id="chart-description">
    Sales increased from $10k in January to $50k in December,
    with steady growth throughout the year.
  </figcaption>
</figure>

<!-- SVG icons -->
<svg aria-hidden="true" focusable="false">
  <!-- Decorative icon -->
</svg>

<svg role="img" aria-label="Close dialog">
  <!-- Functional icon -->
  <use href="#close-icon" />
</svg>
```

### Form Labels

```svelte
<!-- ✅ GOOD: Explicit label association -->
<label for="username">Username</label>
<input id="username" type="text" />

<!-- ✅ GOOD: Implicit label association -->
<label>
  Email
  <input type="email" />
</label>

<!-- ✅ GOOD: aria-label for icons-only buttons -->
<button aria-label="Close dialog">
  <svg aria-hidden="true"><!-- X icon --></svg>
</button>

<!-- ✅ GOOD: Grouping related inputs -->
<fieldset>
  <legend>Shipping Address</legend>

  <label for="street">Street</label>
  <input id="street" type="text" />

  <label for="city">City</label>
  <input id="city" type="text" />
</fieldset>
```

### Table Accessibility

```svelte
<table>
  <caption>Monthly Sales Report 2025</caption>

  <thead>
    <tr>
      <th scope="col">Month</th>
      <th scope="col">Revenue</th>
      <th scope="col">Growth</th>
    </tr>
  </thead>

  <tbody>
    <tr>
      <th scope="row">January</th>
      <td>$10,000</td>
      <td>+5%</td>
    </tr>
    <tr>
      <th scope="row">February</th>
      <td>$12,000</td>
      <td>+20%</td>
    </tr>
  </tbody>
</table>
```

---

## 6. Responsive Design Patterns

### Mobile-First Approach

```css
/* Mobile styles (default) */
.container {
  padding: 1rem;
  font-size: 1rem;
}

/* Tablet */
@media (min-width: 768px) {
  .container {
    padding: 2rem;
    font-size: 1.125rem;
  }
}

/* Desktop */
@media (min-width: 1024px) {
  .container {
    padding: 3rem;
    max-width: 1200px;
    margin: 0 auto;
  }
}
```

### Touch Target Sizes

```css
/* WCAG 2.1 AA: Minimum 44x44px touch targets */
button,
a,
input[type="checkbox"],
input[type="radio"] {
  min-width: 44px;
  min-height: 44px;
  /* Or use padding to achieve size */
}

/* ✅ GOOD: Adequate spacing between touch targets */
.button-group button {
  margin: 0.5rem; /* Prevents accidental taps */
}
```

### Responsive Images

```svelte
<script>
  import { Image } from '@sveltejs/enhanced-img';
  import heroImg from '$lib/assets/hero.jpg?enhanced';
</script>

<!-- Responsive with art direction -->
<picture>
  <source
    media="(min-width: 1024px)"
    srcset="/hero-desktop.jpg"
  />
  <source
    media="(min-width: 768px)"
    srcset="/hero-tablet.jpg"
  />
  <img
    src="/hero-mobile.jpg"
    alt="Hero image description"
  />
</picture>

<!-- Svelte 5 enhanced image -->
<Image
  src={heroImg}
  alt="Hero banner"
  sizes="(max-width: 768px) 100vw, 50vw"
/>
```

---

## 7. Progressive Enhancement

### Works Without JavaScript

```svelte
<!-- Form works with or without JS -->
<form action="/api/contact" method="POST" use:enhance>
  <label for="name">Name</label>
  <input id="name" name="name" required />

  <label for="email">Email</label>
  <input id="email" name="email" type="email" required />

  <button type="submit">Send</button>
</form>

<script lang="ts">
  import { enhance } from '$app/forms';

  // Enhanced with JS for better UX
</script>
```

### No-JS Fallbacks

```svelte
<noscript>
  <style>
    .requires-js {
      display: none;
    }
    .no-js-message {
      display: block;
      background: #fff3cd;
      padding: 1rem;
      border: 1px solid #ffc107;
    }
  </style>

  <div class="no-js-message">
    This site works best with JavaScript enabled.
    Some features may be limited.
  </div>
</noscript>
```

---

## 8. Motion and Animation Accessibility

### Respect Reduced Motion Preference

```css
/* Disable animations for users who prefer reduced motion */
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}

/* Normal animations */
@media (prefers-reduced-motion: no-preference) {
  .fade-in {
    animation: fadeIn 0.3s ease-in;
  }

  @keyframes fadeIn {
    from { opacity: 0; }
    to { opacity: 1; }
  }
}
```

```svelte
<script lang="ts">
  import { onMount } from 'svelte';

  let prefersReducedMotion = $state(false);

  onMount(() => {
    const mediaQuery = window.matchMedia('(prefers-reduced-motion: reduce)');
    prefersReducedMotion = mediaQuery.matches;

    mediaQuery.addEventListener('change', (e) => {
      prefersReducedMotion = e.matches;
    });
  });
</script>

<div class:animated={!prefersReducedMotion}>
  Content
</div>
```

---

## 9. Testing for Accessibility

### Automated Testing

```bash
# Install testing tools
npm install -D @axe-core/playwright vitest
```

```typescript
// tests/accessibility.test.ts
import { expect, test } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test.describe('accessibility tests', () => {
  test('homepage should not have accessibility violations', async ({ page }) => {
    await page.goto('/');

    const accessibilityScanResults = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .analyze();

    expect(accessibilityScanResults.violations).toEqual([]);
  });

  test('form should be accessible', async ({ page }) => {
    await page.goto('/contact');

    const accessibilityScanResults = await new AxeBuilder({ page })
      .include('#contact-form')
      .analyze();

    expect(accessibilityScanResults.violations).toEqual([]);
  });
});
```

### Manual Testing Checklist

- [ ] **Keyboard Navigation**
  - Tab through all interactive elements
  - Ensure logical focus order
  - Verify visible focus indicators
  - Test keyboard shortcuts (Enter, Space, Escape)

- [ ] **Screen Reader Testing**
  - Test with NVDA (Windows) or VoiceOver (macOS)
  - Verify all images have alt text
  - Check heading hierarchy
  - Confirm form labels are announced

- [ ] **Color and Contrast**
  - Use browser DevTools to check contrast ratios
  - Test with color blindness simulators
  - Verify information not conveyed by color alone

- [ ] **Responsive Design**
  - Test at 200% browser zoom
  - Verify mobile touch target sizes (44x44px minimum)
  - Test portrait and landscape orientations

- [ ] **Content**
  - Check that text can be resized to 200%
  - Verify all functionality works without mouse
  - Test with JavaScript disabled (progressive enhancement)

### Browser DevTools

```javascript
// Chrome DevTools Accessibility Panel
// 1. Open DevTools (F12)
// 2. Go to "Lighthouse" tab
// 3. Select "Accessibility" category
// 4. Run audit

// Or use axe DevTools extension
// https://www.deque.com/axe/devtools/
```

---

## 10. Accessibility Statement

```svelte
<!-- src/routes/accessibility/+page.svelte -->
<script lang="ts">
  const lastUpdated = '2025-11-15';
</script>

<h1>Accessibility Statement</h1>

<p>
  We are committed to ensuring digital accessibility for people with disabilities.
  We are continually improving the user experience for everyone and applying the
  relevant accessibility standards.
</p>

<h2>Conformance Status</h2>

<p>
  This website aims to conform to WCAG 2.1 Level AA standards.
  <strong>Conformance</strong> means the content has been tested and meets the
  accessibility requirements at that level.
</p>

<h2>Feedback</h2>

<p>
  We welcome feedback on the accessibility of this website.
  Please contact us at:
</p>

<ul>
  <li>Email: <a href="mailto:accessibility@example.com">accessibility@example.com</a></li>
  <li>Phone: 1-800-555-0100</li>
</ul>

<h2>Technical Specifications</h2>

<p>
  This website relies on the following technologies to work:
</p>

<ul>
  <li>HTML5</li>
  <li>WAI-ARIA</li>
  <li>CSS3</li>
  <li>JavaScript (with progressive enhancement)</li>
</ul>

<h2>Assessment Approach</h2>

<p>
  We assess the accessibility of this website using:
</p>

<ul>
  <li>Automated testing with axe-core</li>
  <li>Manual keyboard navigation testing</li>
  <li>Screen reader testing (NVDA, VoiceOver)</li>
  <li>User testing with people with disabilities</li>
</ul>

<p>
  <small>Last updated: {lastUpdated}</small>
</p>
```

---

## Quick Reference: Common ARIA Patterns

| Pattern | Role | Key Attributes |
|---------|------|----------------|
| **Button** | `button` | `aria-pressed`, `aria-expanded` |
| **Link** | `link` | `aria-current`, `aria-label` |
| **Dialog** | `dialog` | `aria-modal`, `aria-labelledby` |
| **Menu** | `menu` | `aria-haspopup`, `aria-expanded` |
| **Tabs** | `tablist`, `tab`, `tabpanel` | `aria-selected`, `aria-controls` |
| **Accordion** | `button` | `aria-expanded`, `aria-controls` |
| **Alert** | `alert` | `aria-live="assertive"` |
| **Status** | `status` | `aria-live="polite"` |
| **Checkbox** | `checkbox` | `aria-checked` |
| **Radio** | `radio` | `aria-checked` |

---

**Related Resources:**
- [Security Hardening](./security-hardening.md)
- [Performance Optimization](./performance-optimization.md)
- [Deployment and Operations](./deployment-operations.md)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Compliance**: WCAG 2.1 AA, Section 508, ADA
