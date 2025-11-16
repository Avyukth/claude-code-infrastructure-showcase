---
name: mobile-pwa-design
description: Comprehensive mobile-first Progressive Web App (PWA) design guidelines for cross-device responsive frontends. Covers PWA fundamentals (manifest, service workers, offline), responsive design for phones/tablets/desktops, touch interactions, performance optimization, and accessibility. Emphasizes SvelteKit patterns with app-like mobile experiences.
---

# Mobile-First PWA Design - Cross-Device Frontend Excellence

Progressive Web App development for mobile-first experiences that work seamlessly across phones, tablets, and desktops.

## Purpose

This skill guides developers in creating **production-ready Progressive Web Apps** with mobile-first design principles that deliver:

- **App-like experiences** - Installable, offline-capable, fast-loading web applications
- **Cross-device responsiveness** - Adaptive layouts for phones (320-480px), tablets (768-1024px), and desktops (1200px+)
- **Touch-optimized interactions** - 44×44px minimum touch targets, gesture support, thumb-zone optimization
- **Performance excellence** - Core Web Vitals compliance (LCP ≤2.5s, INP ≤200ms, CLS ≤0.1)
- **Accessibility compliance** - WCAG 2.2 Level AA with mobile screen reader support
- **SvelteKit integration** - Framework-specific patterns for service workers, routing, and SSR

## When to Use This Skill

Activate this skill when:
- ✅ Building PWAs or mobile-optimized web applications
- ✅ Creating responsive interfaces for phones, tablets, and desktops
- ✅ Implementing service workers, offline functionality, or installability
- ✅ Optimizing for Core Web Vitals and mobile performance
- ✅ Designing touch-friendly interactions and navigation patterns
- ✅ Ensuring mobile accessibility with screen readers
- ✅ Working with SvelteKit for mobile-first development

## Core Principles

### 1. **Mobile-First Progressive Enhancement**
Start with mobile constraints (small screens, touch input, slow networks), then enhance for larger devices. This forces content prioritization and performance optimization.

**Why**: Mobile accounts for 60%+ of web traffic; mobile-first reduces code bloat and improves all device experiences.

```css
/* ✅ Mobile-first: Base styles for mobile, enhance for desktop */
.button {
  width: 100%;           /* Full width on mobile */
  padding: 1rem;
  font-size: 1rem;
}

@media (min-width: 768px) {
  .button {
    width: auto;         /* Natural width on desktop */
    padding: 0.75rem 2rem;
  }
}

/* ❌ Desktop-first: Requires overrides for mobile */
.button {
  width: auto;
  padding: 0.75rem 2rem;
}

@media (max-width: 767px) {
  .button {
    width: 100% !important;  /* Override with !important */
    padding: 1rem !important;
  }
}
```

### 2. **PWA-First Architecture**
Design for installability, offline capability, and app-like behavior from the start.

**Components**:
- **Manifest**: `manifest.json` for installability
- **Service Worker**: Caching strategies for offline support
- **App Shell**: Fast-loading UI skeleton

```json
{
  "name": "My Mobile PWA",
  "short_name": "PWA",
  "display": "standalone",
  "orientation": "portrait-primary",
  "theme_color": "#6366f1",
  "background_color": "#ffffff",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ]
}
```

### 3. **Touch-Optimized Interaction Design**
Design for fingers, not cursors. **Minimum 44×44px touch targets** (WCAG 2.2).

**Thumb Zones**:
- **Easy reach**: Bottom third of screen (primary actions)
- **Stretch zone**: Top corners (secondary actions)
- **Dead zone**: Top center (avoid critical actions)

```css
/* Touch-friendly button */
.touch-button {
  min-height: 44px;
  min-width: 44px;
  padding: 12px 24px;
  /* Visual feedback */
  transition: transform 0.1s, box-shadow 0.1s;
}

.touch-button:active {
  transform: scale(0.98);
  box-shadow: inset 0 2px 8px rgba(0,0,0,0.2);
}
```

### 4. **Performance-First Loading**
Target Core Web Vitals for mobile networks (often 3G/4G):
- **LCP ≤ 2.5s**: Largest Contentful Paint
- **INP ≤ 200ms**: Interaction to Next Paint
- **CLS ≤ 0.1**: Cumulative Layout Shift

**Techniques**:
- Image optimization (AVIF/WebP, responsive images)
- Code splitting and lazy loading
- Efficient caching strategies
- Critical CSS inlining

### 5. **Adaptive Responsive Layouts**
Respond to device characteristics beyond screen width:
- **Container queries**: Component-based responsiveness
- **Device capabilities**: Touch vs. pointer, network quality
- **Orientation**: Portrait vs. landscape
- **Viewport**: Safe areas (notches, home indicators)

```css
/* Container queries for component responsiveness */
.card-container {
  container-type: inline-size;
}

@container (min-width: 400px) {
  .card {
    display: grid;
    grid-template-columns: 150px 1fr;
  }
}

/* Safe area support for notches */
.header {
  padding-top: env(safe-area-inset-top);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}
```

### 6. **Accessibility Across Devices**
Mobile accessibility is not optional (WCAG 2.2 Level AA):
- Screen reader support (VoiceOver, TalkBack)
- Keyboard navigation on tablets
- Focus management for modals/navigation
- Alternative input methods for gestures

## Quick Start Checklist

### PWA Essentials
- [ ] Create `manifest.json` with 192×192 and 512×512 icons
- [ ] Implement service worker with caching strategy
- [ ] Test installability on mobile browsers
- [ ] Configure `display: "standalone"` for app-like UI
- [ ] Set `theme_color` for status bar customization

### Responsive Design
- [ ] Use mobile-first CSS with `min-width` media queries
- [ ] Implement responsive images with `srcset` and `sizes`
- [ ] Test on real devices (phones, tablets) or emulators
- [ ] Ensure text is readable without zoom (16px+ base font)
- [ ] Avoid horizontal scrolling on any viewport

### Touch Interactions
- [ ] All interactive elements ≥44×44px
- [ ] Provide visual feedback on touch (`:active` states)
- [ ] Support swipe gestures where appropriate
- [ ] Avoid hover-dependent interactions
- [ ] Test with thumb navigation zones

### Performance
- [ ] Optimize images (AVIF/WebP, compression, lazy loading)
- [ ] Achieve Core Web Vitals targets (LCP, INP, CLS)
- [ ] Minimize JavaScript bundle size (<200KB initial)
- [ ] Implement code splitting for routes
- [ ] Test on throttled 3G network

### Accessibility
- [ ] Minimum 44×44px touch targets (WCAG 2.2 SC 2.5.8)
- [ ] Provide keyboard alternatives for gestures
- [ ] Test with VoiceOver (iOS) and TalkBack (Android)
- [ ] Support portrait and landscape orientations
- [ ] Ensure 4.5:1 text contrast ratio

## Navigation Guide

| Need to... | Read this |
|------------|-----------|
| Understand PWA fundamentals | [pwa-fundamentals.md](resources/pwa-fundamentals.md) |
| Implement responsive layouts | [responsive-design.md](resources/responsive-design.md) |
| Design touch interactions | [touch-interactions.md](resources/touch-interactions.md) |
| Optimize mobile performance | [performance-optimization.md](resources/performance-optimization.md) |
| Ensure mobile accessibility | [accessibility-mobile.md](resources/accessibility-mobile.md) |
| Use SvelteKit PWA patterns | [sveltekit-pwa-patterns.md](resources/sveltekit-pwa-patterns.md) |

## Resource Files

### [pwa-fundamentals.md](resources/pwa-fundamentals.md)
Manifest configuration, service workers, caching strategies, offline functionality, installability, push notifications

### [responsive-design.md](resources/responsive-design.md)
Mobile-first CSS, media queries, container queries, fluid typography, adaptive layouts for phones/tablets/desktops

### [touch-interactions.md](resources/touch-interactions.md)
Touch target sizing, gesture patterns, thumb zones, navigation patterns (tab bars, bottom sheets), visual feedback

### [performance-optimization.md](resources/performance-optimization.md)
Core Web Vitals, image optimization (AVIF/WebP), lazy loading, code splitting, caching, mobile network considerations

### [accessibility-mobile.md](resources/accessibility-mobile.md)
WCAG 2.2 mobile criteria, screen readers (VoiceOver/TalkBack), focus management, keyboard navigation, orientation support

### [sveltekit-pwa-patterns.md](resources/sveltekit-pwa-patterns.md)
SvelteKit-specific PWA implementation, service worker integration, SSR considerations, adapter configuration, build optimization

---

## Anti-Patterns to Avoid

❌ Desktop-first design requiring mobile overrides
❌ Touch targets smaller than 44×44px
❌ Hover-dependent interactions without touch alternatives
❌ Fixed viewport widths preventing zoom
❌ Large unoptimized images blocking LCP
❌ Gestures without keyboard alternatives
❌ Missing service worker or offline support
❌ Ignoring safe area insets on notched devices

---

## Related Skills

- **frontend-dev-guidelines** - General frontend patterns and best practices
- **production-hardening-frontend** - Security, monitoring, deployment for production
- **rust-backend-guidelines** - Backend API design for PWA consumption

---

**Skill Status**: PRODUCTION-READY ✅
**Line Count**: < 500 ✅
**Progressive Disclosure**: 6 resource files ✅
**Mobile-Specific**: PWA, touch, cross-device, performance ✅
