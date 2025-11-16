# Performance Hardening for SvelteKit PWAs

## Overview

Performance optimization is critical for PWA success, especially on mobile devices with limited resources. This guide covers bundle optimization, code splitting, lazy loading, and runtime performance patterns.

## Build-Time Optimizations

### Vite Configuration for Performance

```javascript
// vite.config.js
import { sveltekit } from '@sveltejs/kit/vite';
import { visualizer } from 'rollup-plugin-visualizer';
import { compression } from 'vite-plugin-compression2';
import { imagetools } from 'vite-imagetools';

export default {
  plugins: [
    sveltekit(),
    
    // Image optimization
    imagetools({
      defaultDirectives: (id) => {
        if (id.searchParams.has('hero')) {
          return new URLSearchParams({
            format: 'avif;webp;jpg',
            w: '640;1280;1920',
            quality: '80'
          });
        }
        return new URLSearchParams();
      }
    }),
    
    // Compression
    compression({
      algorithm: 'brotli',
      exclude: [/\.(br|gz)$/],
      deleteOriginalAssets: false
    }),
    
    // Bundle analysis (dev only)
    process.env.ANALYZE && visualizer({
      filename: './stats.html',
      open: true,
      gzipSize: true,
      brotliSize: true
    })
  ],
  
  build: {
    // Modern browser target
    target: 'es2022',
    
    // Optimize chunks
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          // Vendor chunking strategy
          if (id.includes('node_modules')) {
            // Framework chunks
            if (id.includes('svelte')) return 'svelte';
            if (id.includes('@sveltejs')) return 'sveltekit';
            
            // Large libraries in separate chunks
            if (id.includes('three')) return 'three';
            if (id.includes('chart')) return 'charts';
            if (id.includes('editor')) return 'editor';
            
            // All other vendors
            return 'vendor';
          }
        },
        
        // Asset naming
        chunkFileNames: 'js/[name]-[hash].js',
        entryFileNames: 'js/[name]-[hash].js',
        assetFileNames: 'assets/[name]-[hash].[ext]'
      }
    },
    
    // Chunk size warnings
    chunkSizeWarningLimit: 500,
    
    // CSS code splitting
    cssCodeSplit: true,
    
    // Minification
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
        pure_funcs: ['console.log', 'console.debug'],
        passes: 2
      },
      mangle: {
        safari10: true
      },
      format: {
        comments: false
      }
    }
  },
  
  // Dependency optimization
  optimizeDeps: {
    include: ['svelte', '@sveltejs/kit'],
    exclude: ['@sveltejs/kit/node']
  }
};
```

### SvelteKit Configuration

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-auto';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

export default {
  preprocess: vitePreprocess(),
  
  kit: {
    adapter: adapter(),
    
    // Prerendering
    prerender: {
      crawl: true,
      enabled: true,
      entries: ['*'],
      handleHttpError: 'warn'
    },
    
    // Service worker
    serviceWorker: {
      register: true,
      files: (filepath) => !/\.DS_Store/.test(filepath)
    },
    
    // Inline critical CSS
    inlineStyleThreshold: 2048,
    
    // Output
    output: {
      preloadStrategy: 'modulepreload'
    },
    
    // CSP for performance
    csp: {
      mode: 'auto',
      directives: {
        'script-src': ['self'],
        'worker-src': ['self']
      }
    }
  },
  
  // Compiler options
  compilerOptions: {
    css: 'injected',
    dev: false,
    hydratable: true,
    immutable: true
  }
};
```

## Code Splitting Strategies

### Route-Based Splitting

```javascript
// +layout.js - Lazy load heavy components
export async function load({ route }) {
  // Only load admin components for admin routes
  if (route.id?.startsWith('/admin')) {
    const { AdminLayout } = await import('$lib/layouts/AdminLayout.svelte');
    return {
      component: AdminLayout
    };
  }
  
  return {};
}
```

### Component-Based Splitting

```svelte
<!-- LazyComponent.svelte -->
<script>
  import { onMount } from 'svelte';
  
  export let componentName;
  export let props = {};
  
  let Component;
  let loading = true;
  let error = null;
  
  onMount(async () => {
    try {
      const module = await import(
        /* @vite-ignore */
        `$lib/components/heavy/${componentName}.svelte`
      );
      Component = module.default;
      loading = false;
    } catch (e) {
      error = e;
      loading = false;
    }
  });
</script>

{#if loading}
  <div class="skeleton-loader" aria-busy="true">
    Loading...
  </div>
{:else if error}
  <div class="error">Failed to load component</div>
{:else if Component}
  <svelte:component this={Component} {...props} />
{/if}
```

### Dynamic Imports with Prefetching

```javascript
// lib/utils/dynamic-import.js
const moduleCache = new Map();

export async function dynamicImport(path, prefetch = false) {
  // Check cache
  if (moduleCache.has(path)) {
    return moduleCache.get(path);
  }
  
  if (prefetch) {
    // Prefetch but don't wait
    import(path).then(module => {
      moduleCache.set(path, module);
    });
    return null;
  }
  
  // Load and cache
  const module = await import(path);
  moduleCache.set(path, module);
  return module;
}

// Prefetch on idle
export function prefetchOnIdle(paths) {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(() => {
      paths.forEach(path => dynamicImport(path, true));
    });
  } else {
    setTimeout(() => {
      paths.forEach(path => dynamicImport(path, true));
    }, 2000);
  }
}
```

## Image Optimization

### Responsive Image Component

```svelte
<!-- lib/components/OptimizedImage.svelte -->
<script>
  import { onMount } from 'svelte';
  
  export let src;
  export let alt;
  export let sizes = '100vw';
  export let loading = 'lazy';
  export let aspectRatio = '16/9';
  
  let imgElement;
  let isIntersecting = false;
  let hasLoaded = false;
  
  const srcset = `
    ${src}?w=640 640w,
    ${src}?w=768 768w,
    ${src}?w=1024 1024w,
    ${src}?w=1280 1280w,
    ${src}?w=1920 1920w
  `;
  
  onMount(() => {
    if (loading === 'lazy') {
      const observer = new IntersectionObserver((entries) => {
        isIntersecting = entries[0].isIntersecting;
        if (isIntersecting) {
          observer.disconnect();
        }
      }, {
        rootMargin: '50px'
      });
      
      observer.observe(imgElement);
      
      return () => observer.disconnect();
    } else {
      isIntersecting = true;
    }
  });
</script>

<div 
  class="image-container"
  bind:this={imgElement}
  style="aspect-ratio: {aspectRatio}"
>
  {#if isIntersecting}
    <picture>
      <source 
        type="image/avif" 
        srcset={srcset.replace(/\?/g, '?format=avif&')}
      />
      <source 
        type="image/webp" 
        srcset={srcset.replace(/\?/g, '?format=webp&')}
      />
      <img
        {src}
        {srcset}
        {sizes}
        {alt}
        {loading}
        on:load={() => hasLoaded = true}
        class:loaded={hasLoaded}
      />
    </picture>
  {:else}
    <div class="placeholder" />
  {/if}
</div>

<style>
  .image-container {
    position: relative;
    overflow: hidden;
    background: #f0f0f0;
  }
  
  img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    opacity: 0;
    transition: opacity 0.3s;
  }
  
  img.loaded {
    opacity: 1;
  }
  
  .placeholder {
    width: 100%;
    height: 100%;
    background: linear-gradient(
      90deg,
      #f0f0f0 25%,
      #e0e0e0 50%,
      #f0f0f0 75%
    );
    animation: shimmer 2s infinite;
  }
  
  @keyframes shimmer {
    0% { transform: translateX(-100%); }
    100% { transform: translateX(100%); }
  }
</style>
```

