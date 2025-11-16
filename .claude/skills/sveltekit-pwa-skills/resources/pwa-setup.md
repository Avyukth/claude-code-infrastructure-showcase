# PWA Setup and Configuration

## Overview

This guide covers the complete setup process for converting a SvelteKit application into a Progressive Web App, including manifest configuration, service worker implementation, and installation flow.

## Installation Methods

### Method 1: Using @vite-pwa/sveltekit (Recommended)

The Vite PWA plugin provides zero-config PWA functionality with advanced features.

```bash
npm i -D @vite-pwa/sveltekit
```

**Configuration:**
```javascript
// vite.config.js
import { sveltekit } from '@sveltejs/kit/vite';
import { SvelteKitPWA } from '@vite-pwa/sveltekit';

export default {
  plugins: [
    sveltekit(),
    SvelteKitPWA({
      registerType: 'autoUpdate',
      devOptions: {
        enabled: true,
        type: 'module'
      },
      manifest: {
        name: 'My SvelteKit PWA',
        short_name: 'MySKPWA',
        description: 'My awesome SvelteKit Progressive Web App',
        theme_color: '#ffffff',
        background_color: '#ffffff',
        display: 'standalone',
        orientation: 'portrait',
        scope: '/',
        start_url: '/',
        icons: [
          {
            src: '/icon-192.png',
            sizes: '192x192',
            type: 'image/png',
            purpose: 'any maskable'
          },
          {
            src: '/icon-512.png',
            sizes: '512x512',
            type: 'image/png',
            purpose: 'any maskable'
          }
        ]
      },
      workbox: {
        globPatterns: ['client/**/*.{js,css,ico,png,svg,webp,woff,woff2}'],
        cleanupOutdatedCaches: true,
        clientsClaim: true,
        skipWaiting: true
      }
    })
  ]
};
```

### Method 2: Manual Setup (More Control)

For complete control over PWA behavior, implement manually.

#### Step 1: Create Manifest
```json
// static/manifest.json
{
  "name": "My SvelteKit Progressive Web App",
  "short_name": "MySKPWA",
  "description": "A mobile-friendly PWA built with SvelteKit",
  "categories": ["productivity", "utilities"],
  "lang": "en-US",
  "dir": "ltr",
  "theme_color": "#4F46E5",
  "background_color": "#ffffff",
  "display": "standalone",
  "orientation": "any",
  "scope": "/",
  "start_url": "/?source=pwa",
  "id": "/?source=pwa",
  "icons": [
    {
      "src": "/icon-72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "/icon-96.png",
      "sizes": "96x96",
      "type": "image/png"
    },
    {
      "src": "/icon-128.png",
      "sizes": "128x128",
      "type": "image/png"
    },
    {
      "src": "/icon-144.png",
      "sizes": "144x144",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "/icon-152.png",
      "sizes": "152x152",
      "type": "image/png"
    },
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icon-384.png",
      "sizes": "384x384",
      "type": "image/png"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ],
  "shortcuts": [
    {
      "name": "Dashboard",
      "short_name": "Dashboard",
      "description": "Open the dashboard",
      "url": "/dashboard",
      "icons": [
        {
          "src": "/icon-96.png",
          "sizes": "96x96"
        }
      ]
    }
  ],
  "screenshots": [
    {
      "src": "/screenshot-narrow.png",
      "sizes": "540x720",
      "type": "image/png",
      "form_factor": "narrow",
      "label": "Home screen"
    },
    {
      "src": "/screenshot-wide.png",
      "sizes": "1024x593",
      "type": "image/png",
      "form_factor": "wide",
      "label": "Home screen"
    }
  ],
  "related_applications": [],
  "prefer_related_applications": false,
  "share_target": {
    "action": "/share",
    "method": "POST",
    "enctype": "multipart/form-data",
    "params": {
      "title": "title",
      "text": "text",
      "url": "url",
      "files": [
        {
          "name": "media",
          "accept": ["image/*", "video/*"]
        }
      ]
    }
  }
}
```

#### Step 2: Service Worker Implementation
```javascript
// src/service-worker.js
import { build, files, version, prerendered } from '$service-worker';

const CACHE = `cache-${version}`;
const ASSETS = [
  ...build,       // JS/CSS from Vite build
  ...files,       // Static files
  ...prerendered  // Prerendered pages
];

// Install event - cache all assets
self.addEventListener('install', (event) => {
  console.log('[Service Worker] Installing');
  
  async function addFilesToCache() {
    const cache = await caches.open(CACHE);
    await cache.addAll(ASSETS);
  }
  
  event.waitUntil(addFilesToCache());
  self.skipWaiting(); // Activate immediately
});

// Activate event - clean old caches
self.addEventListener('activate', (event) => {
  console.log('[Service Worker] Activating');
  
  async function deleteOldCaches() {
    const keys = await caches.keys();
    
    await Promise.all(
      keys
        .filter(key => key !== CACHE)
        .map(key => caches.delete(key))
    );
  }
  
  event.waitUntil(deleteOldCaches());
  self.clients.claim(); // Take control immediately
});

// Fetch event - serve from cache, fallback to network
self.addEventListener('fetch', (event) => {
  if (event.request.method !== 'GET') return;
  
  async function respond() {
    const url = new URL(event.request.url);
    const cache = await caches.open(CACHE);
    
    // Try cache first for assets
    if (ASSETS.includes(url.pathname)) {
      const cached = await cache.match(url.pathname);
      if (cached) return cached;
    }
    
    // Network-first for API calls
    if (url.pathname.startsWith('/api/')) {
      try {
        const response = await fetch(event.request);
        if (response.ok) {
          cache.put(event.request, response.clone());
        }
        return response;
      } catch {
        const cached = await cache.match(event.request);
        if (cached) return cached;
        throw new Error('Offline');
      }
    }
    
    // Cache-first for everything else
    const cached = await cache.match(event.request);
    if (cached) return cached;
    
    try {
      const response = await fetch(event.request);
      if (response.ok && response.status === 200) {
        cache.put(event.request, response.clone());
      }
      return response;
    } catch {
      // Return offline page
      return cache.match('/offline');
    }
  }
  
  event.respondWith(respond());
});
```

#### Step 3: Update app.html
```html
<!-- src/app.html -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
    
    <!-- PWA Meta Tags -->
    <meta name="theme-color" content="#4F46E5" />
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
    <meta name="apple-mobile-web-app-title" content="MySKPWA" />
    
    <!-- Manifest -->
    <link rel="manifest" href="%sveltekit.assets%/manifest.json" />
    
    <!-- iOS Icons -->
    <link rel="apple-touch-icon" href="%sveltekit.assets%/icon-180.png" />
    
    <!-- Favicon -->
    <link rel="icon" href="%sveltekit.assets%/favicon.png" />
    
    %sveltekit.head%
  </head>
  <body data-sveltekit-preload-data="hover">
    <div style="display: contents">%sveltekit.body%</div>
  </body>
</html>
```

## Icon Generation

### Using PWA Asset Generator

```bash
npx pwa-asset-generator logo.svg ./static \
  --background "#fff" \
  --padding "10%" \
  --type png \
  --manifest ./static/manifest.json \
  --index ./src/app.html
```

### Manual Icon Requirements

| Size | Purpose | Required |
|------|---------|----------|
| 72x72 | Legacy Android | Optional |
| 96x96 | Legacy Chrome | Optional |
| 128x128 | Chrome Web Store | Optional |
| 144x144 | Legacy Microsoft | Yes |
| 152x152 | Legacy iOS | Optional |
| 180x180 | iOS (iPhone) | Yes |
| 192x192 | Chrome/Android | Yes |
| 384x384 | Android Splash | Recommended |
| 512x512 | Android Splash | Yes |

### Maskable Icons

Ensure 10% safe zone around content for maskable icons:

```svg
<!-- icon-maskable.svg -->
<svg viewBox="0 0 512 512">
  <!-- 10% padding = 51px on each side -->
  <rect x="51" y="51" width="410" height="410" fill="#4F46E5"/>
  <!-- Your logo content within safe zone -->
</svg>
```

## Installation UI Component

```svelte
<!-- lib/components/PWAInstaller.svelte -->
<script>
  import { onMount } from 'svelte';
  import { browser } from '$app/environment';
  
  let deferredPrompt = null;
  let showInstallButton = false;
  let isInstalled = false;
  let platform = '';
  
  onMount(() => {
    if (!browser) return;
    
    // Check if already installed
    if (window.matchMedia('(display-mode: standalone)').matches) {
      isInstalled = true;
      return;
    }
    
    // Detect platform
    const userAgent = navigator.userAgent.toLowerCase();
    if (/iphone|ipad|ipod/.test(userAgent)) {
      platform = 'ios';
    } else if (/android/.test(userAgent)) {
      platform = 'android';
    } else if (/windows/.test(userAgent)) {
      platform = 'windows';
    } else if (/macintosh/.test(userAgent)) {
      platform = 'macos';
    }
    
    // Listen for install prompt
    window.addEventListener('beforeinstallprompt', (e) => {
      e.preventDefault();
      deferredPrompt = e;
      showInstallButton = true;
    });
    
    // Check if installed after prompt
    window.addEventListener('appinstalled', () => {
      console.log('PWA installed');
      isInstalled = true;
      showInstallButton = false;
    });
  });
  
  async function installPWA() {
    if (!deferredPrompt) return;
    
    deferredPrompt.prompt();
    const { outcome } = await deferredPrompt.userChoice;
    
    console.log(`User response: ${outcome}`);
    deferredPrompt = null;
    showInstallButton = false;
  }
  
  function showIOSInstructions() {
    // Show iOS installation instructions
  }
</script>

{#if !isInstalled}
  {#if showInstallButton}
    <div class="install-prompt">
      <h3>Install App</h3>
      <p>Install our app for a better experience!</p>
      <button on:click={installPWA}>
        Install Now
      </button>
      <button on:click={() => showInstallButton = false}>
        Maybe Later
      </button>
    </div>
  {:else if platform === 'ios'}
    <button on:click={showIOSInstructions}>
      Install on iOS
    </button>
  {/if}
{/if}
```

## Testing PWA Installation

### Desktop Chrome
1. Open DevTools → Application tab
2. Check Manifest section
3. Check Service Workers section
4. Test "Add to Home Screen" flow

### Mobile Testing
1. Serve with HTTPS (or localhost)
2. Open in mobile browser
3. Check for install prompt
4. Test installation flow
5. Verify app opens standalone

### Lighthouse Audit
```bash
# Build and preview production
npm run build
npm run preview

# Run Lighthouse
# Chrome DevTools → Lighthouse → PWA category
```

## Common Issues and Solutions

### Issue: Service Worker Not Registering
**Solution:** Ensure HTTPS or localhost, check console errors

### Issue: Install Prompt Not Showing
**Solution:** Check manifest validity, ensure icons present

### Issue: iOS Not Installing
**Solution:** Add apple-specific meta tags, provide instructions

### Issue: Updates Not Working
**Solution:** Implement proper update flow with skipWaiting

## Best Practices

1. **Always test on real devices** - Emulators miss edge cases
2. **Provide offline fallbacks** - Don't leave users hanging
3. **Implement update notifications** - Keep users on latest version
4. **Use adaptive icons** - Support all platform styles
5. **Monitor storage usage** - Clean old caches regularly
6. **Track installation metrics** - Measure PWA adoption

## References

- [Web App Manifest Spec](https://www.w3.org/TR/appmanifest/)
- [Service Worker API](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API)
- [PWA Installation Criteria](https://web.dev/install-criteria/)
