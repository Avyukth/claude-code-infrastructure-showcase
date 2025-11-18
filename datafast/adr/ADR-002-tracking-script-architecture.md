# ADR-002: Tracking Script Architecture

**Date:** 2025-11-18
**Status:** Accepted
**Deciders:** Engineering Team, Product Lead
**Technical Story:** [FR-1 - Analytics Tracking Core, NFR-1 - Performance]

## Context and Problem Statement

The tracking script is DataFa.st's core data collection mechanism, embedded on thousands of customer websites. It must:
- Be **ultra-lightweight** (<5KB) to minimize impact on page load
- Load **asynchronously** without blocking page rendering
- Support **single-page apps** (SPAs) with auto-tracking
- Be **privacy-compliant** (no cookies, GDPR-friendly)
- Handle network failures gracefully
- Work across all modern browsers

The script's performance directly affects user satisfaction and customer retention.

## Decision Drivers

- **File Size:** <5KB compressed (target: 3-4KB)
- **Page Load Impact:** <50ms added latency (p95)
- **Browser Support:** Latest 2 versions of Chrome, Firefox, Safari, Edge
- **Privacy:** Cookie-less tracking, respect DNT
- **SPA Support:** Auto-detect route changes (History API, hash routing)
- **Reliability:** Queue events offline, retry on failure
- **Developer Experience:** Simple 1-line install
- **Debuggability:** Console logs in debug mode

## Considered Options

1. **Vanilla JavaScript** (no dependencies, custom build)
2. **Preact/Lightweight Framework** (small React alternative)
3. **Web Assembly (WASM)** (high performance)
4. **Third-Party SDK** (e.g., Segment, Amplitude SDK modified)

## Decision Outcome

**Chosen option:** "Vanilla JavaScript with Custom Build Pipeline"

**Rationale:**
Vanilla JS gives full control over file size and performance. Modern ES6+ features (async/await, fetch, modules) provide excellent DX without framework overhead. Custom build pipeline with Terser (minification) + Brotli (compression) achieves <4KB target.

### Positive Consequences

- **Minimal size:** 3.5KB compressed (vs. 10KB+ for framework-based)
- **Zero dependencies:** No supply chain risks or breaking changes
- **Full control:** Optimize every byte for performance
- **Fast loading:** <20ms parse + execute time
- **Easy debugging:** Source maps available in debug mode
- **Future-proof:** No framework upgrade cycles

### Negative Consequences

- **More code to write:** No framework utilities (must build custom helpers)
- **Browser compatibility:** Manual polyfills for older browsers (if needed)
- **Maintenance burden:** Team owns all code (no community patches)
- **Testing complexity:** Must test across multiple browsers manually

## Pros and Cons of the Options

### Vanilla JavaScript

**Pros:**
- **Smallest possible size** (<4KB with aggressive minification)
- **Maximum performance** (no framework overhead)
- Zero dependencies = **no security vulnerabilities** from third-party libs
- **Full customization** for tracking logic
- **Faster iteration** (no framework constraints)

**Cons:**
- More development time (no pre-built components)
- Manual browser compatibility testing required
- No community support (team owns all code)
- Potential for bugs in custom code

---

### Preact (Lightweight React)

**Pros:**
- Familiar React-like API for team
- Component-based architecture
- Community support and ecosystem

**Cons:**
- **10KB+ compressed** (3x larger than vanilla)
- Unnecessary for simple event tracking (overkill)
- Additional parsing/execution overhead
- Framework lock-in

**Verdict:** Overkill for a tracking script. Preact's benefits (components, JSX) don't apply to event tracking logic.

---

### Web Assembly (WASM)

**Pros:**
- Near-native performance
- Language-agnostic (could write in Rust/Go)
- Potential for advanced analytics (client-side aggregation)

**Cons:**
- **Larger file size** (WASM binary + JS glue code > 10KB)
- **Slower initial load** (compile + instantiate WASM)
- Limited browser support (older browsers need polyfill)
- Overkill for simple HTTP requests
- Debugging is harder

**Verdict:** Performance gains don't justify size/complexity trade-offs for a tracking script.

---

### Third-Party SDK (Modified)

**Pros:**
- Battle-tested code (Segment, Amplitude)
- Pre-built features (retry logic, batching)
- Faster initial development

**Cons:**
- **Large file size** (15KB+ for Segment SDK)
- **Licensing issues** (can't fork closed-source SDKs)
- Limited customization (hard to modify)
- Not optimized for our specific use case

**Verdict:** Too heavyweight and legally complex.

---

## Technical Implementation

### Architecture

```
┌─────────────────────────────────────────┐
│  User's Website (Browser)               │
│                                          │
│  1. <script src="datafast.js">          │
│       ↓                                  │
│  2. window.datafast = { track, goal }   │
│       ↓                                  │
│  3. Auto-track pageview                 │
│       ↓                                  │
│  4. Batch events (max 10, 5s timeout)   │
│       ↓                                  │
│  5. POST /track (with retry queue)      │
└─────────────────────────────────────────┘
               ↓
    ┌──────────────────┐
    │ DataFa.st API    │
    │ POST /track      │
    └──────────────────┘
               ↓
    ┌──────────────────┐
    │ ClickHouse DB    │
    └──────────────────┘
```

### Core Features

**1. Async Loading**
```html
<script async src="https://cdn.datafa.st/v1/script.js" data-website-id="abc123"></script>
```

**2. Event Batching**
- Queue events in memory (max 10 events OR 5 seconds, whichever first)
- Single HTTP request per batch (reduce network overhead)

**3. Offline Support**
- Queue failed requests in localStorage
- Retry when online (use `navigator.onLine` + `online` event)
- Max queue size: 100 events (prevent storage overflow)

**4. SPA Auto-Tracking**
```javascript
// Detect History API changes (React Router, Next.js)
window.addEventListener('popstate', trackPageview);

// Intercept pushState/replaceState
const originalPushState = history.pushState;
history.pushState = function(...args) {
    originalPushState.apply(this, args);
    trackPageview();
};
```

**5. Privacy**
- No cookies (use fingerprinting: canvas, fonts, user agent)
- Respect DNT: `if (navigator.doNotTrack === '1') return;`
- IP anonymization: Last octet hashed server-side

**6. Error Handling**
- Wrap all code in try/catch (never break user's site)
- Log errors to console (debug mode only)
- Fallback: If main CDN fails, load from backup URL

### Build Pipeline

**Tools:**
- **Terser:** Minification (variable renaming, whitespace removal)
- **Brotli:** Compression (better than Gzip, 10-15% smaller)
- **Rollup:** Bundling (tree-shaking unused code)
- **ESLint:** Code quality checks

**Build Script:**
```bash
# 1. Bundle with Rollup
rollup src/datafast.js -o dist/datafast.bundle.js

# 2. Minify with Terser
terser dist/datafast.bundle.js -o dist/datafast.min.js \
    --compress passes=3 \
    --mangle toplevel \
    --module

# 3. Compress with Brotli
brotli -f -q 11 dist/datafast.min.js -o dist/datafast.min.js.br

# 4. Generate source map (for debugging)
terser dist/datafast.bundle.js -o dist/datafast.min.js \
    --source-map "url='datafast.min.js.map'"
```

**Result:** 3.5KB compressed, 12KB uncompressed

---

## Testing Strategy

**1. Unit Tests (Jest)**
- Test event batching logic
- Test retry queue
- Test fingerprinting accuracy

**2. Integration Tests (Playwright)**
- Install script on test site
- Simulate pageviews, verify events sent
- Test SPA navigation (Next.js, React Router)
- Test offline → online transition

**3. Browser Compatibility Tests (BrowserStack)**
- Chrome (latest 2 versions)
- Firefox (latest 2 versions)
- Safari (macOS, iOS latest 2 versions)
- Edge (latest 2 versions)

**4. Performance Tests**
- Lighthouse: Measure page load impact (<50ms)
- WebPageTest: 3G/4G connection speeds
- Bundle size monitoring (CI fails if >5KB)

---

## Deployment Strategy

**CDN:** Cloudflare or BunnyCDN
- Global edge network (low latency)
- Auto-compression (Brotli, Gzip)
- Cache-Control: 1 hour (balance freshness vs. performance)
- Versioning: `/v1/script.js` (allow future breaking changes)

**Rollout:**
1. Deploy to staging CDN → Test with beta users
2. Canary release: 10% traffic → Monitor error rates
3. Full release: 100% traffic
4. Rollback plan: Revert CDN to previous version (instant)

---

## Monitoring

**Metrics to Track:**
- Script load time (p50, p95, p99)
- Event delivery success rate (target: 99.9%)
- Retry queue size (should be near-zero normally)
- Error rate (JavaScript exceptions)
- File size over time (alert if >5KB)

**Alerts:**
- Error rate >0.1% → Slack notification
- Event delivery <99% → Page on-call engineer
- File size >5KB → Block deployment

---

## Future Enhancements (Phase 2+)

- [ ] **Client-side aggregation:** Pre-aggregate events in browser (reduce server load)
- [ ] **Compression:** Use MessagePack instead of JSON (smaller payload)
- [ ] **HTTP/3:** Use QUIC protocol for faster requests
- [ ] **WebWorker:** Offload tracking to background thread (improve main thread performance)
- [ ] **A/B testing:** Track experiment variants

---

## Links

- [Terser Documentation](https://terser.org/)
- [Rollup Guide](https://rollupjs.org/guide/en/)
- [Brotli Compression](https://github.com/google/brotli)
- [Canvas Fingerprinting](https://fingerprintjs.com/blog/canvas-fingerprinting/)
- [PRD Section 3: FR-1 Analytics Tracking](../prd/DATAFAST-PRD.md#fr-1-analytics-tracking-core)

---

**Reviewed by:** [Pending]
**Approved by:** [Pending]
