# SvelteKit PWA Guidelines - Complete Skill Package

**Status**: ✅ **COMPLETE** - v1.1 Meta-Skill Compliant
**Created**: 2024
**Updated**: 2025-11-15
**Format**: Claude Code Skill

## 📁 Directory Structure

```
sveltekit-pwa-guidelines/
├── SKILL.md                              # Main entry point (337 lines)
├── README.md                              # This file
├── resources/                             # 18 resource files (~500 lines each)
│   ├── pwa-setup.md                      # Complete PWA setup guide (486 lines)
│   │
│   ├── Setup & Deployment
│   ├── deployment-platforms.md           # Vercel, Netlify, Cloudflare (450 lines)
│   └── deployment-cicd.md                # CI/CD pipelines, monitoring (449 lines)
│   │
│   ├── Core PWA Features
│   ├── offline-caching.md                # Caching strategies, Workbox (443 lines)
│   ├── offline-sync.md                   # Background sync, IndexedDB (443 lines)
│   ├── push-notifications-setup.md       # Server setup, client impl (478 lines)
│   └── push-notifications-advanced.md    # Advanced patterns, SW handlers (475 lines)
│   │
│   ├── Mobile Optimization
│   ├── mobile-ux.md                      # Touch, viewport, responsive (354 lines)
│   ├── mobile-performance.md             # Mobile-specific optimizations (354 lines)
│   ├── performance-optimization.md       # Build-time, code splitting (383 lines)
│   └── performance-monitoring.md         # Web Vitals, runtime monitoring (383 lines)
│   │
│   ├── Development Patterns
│   ├── component-patterns.md             # Component organization (392 lines)
│   ├── state-management.md               # Reactive state for PWA (392 lines)
│   ├── accessibility-fundamentals.md     # ARIA, keyboard, screen readers (385 lines)
│   └── accessibility-testing.md          # Automated & manual a11y tests (384 lines)
│   │
│   └── Testing & Operations
│       ├── testing-pwa.md                # Unit, integration, E2E (407 lines)
│       ├── debugging-pwa.md              # Debugging tools, techniques (407 lines)
│       └── troubleshooting.md            # Common issues, solutions (450 lines)
```

## 🚀 Quick Start

### 1. Integration with Claude

To use this skill in Claude's skill system:

1. Copy the entire `sveltekit-pwa-guidelines` folder to your project's `.claude/skills/` directory
2. The skill will auto-activate when working with PWA-related files or concepts
3. Reference specific resource files for detailed implementation guides

### 2. Manual Usage

Browse the skill documents directly:
- Start with `SKILL.md` for an overview
- Deep dive into specific topics via the `resources/` folder
- Each resource file contains production-ready code examples

## 📚 What's Included

### Core PWA Features
- **Service Worker Implementation**: Complete caching strategies with offline support
- **Web App Manifest**: Configuration for installable apps
- **Push Notifications**: Full Web Push API implementation with backend examples
- **Background Sync**: Queue and sync data when back online
- **Update Flow**: Seamless app updates with user notifications

### Mobile Optimization
- **Touch Gestures**: Swipe, pinch, drag implementations
- **Viewport Management**: Safe area handling for modern devices
- **Responsive Patterns**: Mobile-first component architecture
- **Virtual Keyboard**: Smart handling of input focus and layout adjustments

### Performance
- **Code Splitting**: Route-based and component-based strategies
- **Image Optimization**: Responsive images with AVIF/WebP support
- **Bundle Analysis**: Vite configuration for optimal builds
- **Runtime Performance**: Memory management and rendering optimizations
- **Web Vitals**: Monitoring and optimization strategies

### Production Readiness
- **Security Hardening**: CSP, XSS prevention, secure headers
- **Accessibility**: WCAG compliance, ARIA patterns, keyboard navigation
- **Testing Strategies**: Unit, integration, E2E, and PWA-specific tests
- **Deployment**: Platform-specific configurations (Vercel, Netlify, Docker, etc.)
- **CI/CD Pipelines**: GitHub Actions, GitLab CI examples

## 💡 Key Features

### Auto-Activation
The skill automatically activates when:
- Creating or editing `service-worker.js`, `manifest.json`
- Using PWA-related packages (`@vite-pwa/sveltekit`, `workbox`)
- Writing code with PWA patterns (caching, push notifications)
- Asking about offline functionality, installable apps, or mobile optimization

### Comprehensive Examples
Each resource file contains:
- ✅ Production-ready code snippets
- ✅ Configuration examples
- ✅ Best practices and anti-patterns
- ✅ Testing strategies
- ✅ Troubleshooting guides
- ✅ Performance optimization tips

### Framework Integration
- Built specifically for SvelteKit's architecture
- Leverages Svelte's reactivity and component model
- Integrates with SvelteKit's routing and SSR/SSG capabilities
- Compatible with all SvelteKit adapters

## 🎯 Use Cases

This skill helps you:
1. **Convert existing SvelteKit apps to PWAs**
2. **Build offline-first applications**
3. **Implement push notifications**
4. **Optimize for mobile devices**
5. **Meet Lighthouse PWA requirements**
6. **Deploy production-ready PWAs**

## 📖 How to Navigate

### For Beginners
1. Start with `resources/pwa-setup.md` for basic PWA setup
2. Follow `resources/mobile-optimization.md` for responsive design
3. Implement offline features using `resources/offline-capabilities.md`

### For Advanced Users
1. Dive into `resources/performance-hardening.md` for optimization
2. Implement complex patterns from `resources/component-architecture.md`
3. Set up complete CI/CD using `resources/deployment-strategies.md`

### For Specific Features
- **Push Notifications**: `resources/push-notifications.md`
- **Accessibility**: `resources/accessibility-patterns.md`
- **Testing**: `resources/testing-debugging.md`

## 🔧 Technology Stack

- **Framework**: SvelteKit 1.0+
- **Build Tool**: Vite 4.0+
- **PWA Plugin**: @vite-pwa/sveltekit
- **Service Worker**: Workbox or native implementation
- **Testing**: Vitest, Playwright
- **Deployment**: Vercel, Netlify, Docker, Static hosting

## 📝 Best Practices Highlighted

1. **Progressive Enhancement First**: Start with core functionality, layer PWA features
2. **Mobile-First Development**: Design for constraints, enhance for capabilities
3. **Offline-First Architecture**: Cache strategically, handle network failures
4. **Performance Obsession**: Every byte and millisecond matters
5. **Accessibility Always**: WCAG compliance from the start

## 🤝 Contributing

This skill package follows the Claude skill system format. To contribute or customize:
1. Maintain the directory structure
2. Keep main SKILL.md under 500 lines
3. Split detailed content into resource files
4. Include working code examples
5. Update skill-rules.json for auto-activation triggers

## 📄 License

This skill package is provided as educational material for building Progressive Web Apps with SvelteKit. Use freely in your projects.

## 🔗 References

- [SvelteKit Documentation](https://kit.svelte.dev)
- [PWA Documentation](https://web.dev/progressive-web-apps/)
- [Vite PWA Plugin](https://vite-pwa-org.netlify.app/)
- [MDN Web Docs](https://developer.mozilla.org/)

---

## 📋 Changelog

### v1.1 (2025-11-15) - Meta-Skill Compliance Update

**Major Restructuring:**
- ✅ Split 8 oversized files into 16 meta-skill compliant files (~400-500 lines each)
- ✅ Added comprehensive troubleshooting.md (450 lines)
- ✅ Integrated with .claude/skills/skill-rules.json for auto-activation
- ✅ Updated SKILL.md navigation with complete resource structure
- ✅ All files now comply with ~500-line meta-skill guideline

**New Files Created:**
1. `push-notifications-setup.md` + `push-notifications-advanced.md` (from 953-line original)
2. `deployment-platforms.md` + `deployment-cicd.md` (from 899-line original)
3. `offline-caching.md` + `offline-sync.md` (from 886-line original)
4. `testing-pwa.md` + `debugging-pwa.md` (from 814-line original)
5. `component-patterns.md` + `state-management.md` (from 784-line original)
6. `accessibility-fundamentals.md` + `accessibility-testing.md` (from 769-line original)
7. `performance-optimization.md` + `performance-monitoring.md` (from 766-line original)
8. `mobile-ux.md` + `mobile-performance.md` (from 708-line original)
9. `troubleshooting.md` (new, 450 lines)

**Total:** 19 files (SKILL.md + README.md + 17 resources) | ~8,337 lines

### v1.0 (2024)
- ✅ Initial release with 9 core resource files
- ✅ Comprehensive PWA patterns for SvelteKit
- ✅ Mobile optimization strategies
- ✅ Production deployment guides

---

Created following Claude's skill system architecture for optimal integration and usage.
