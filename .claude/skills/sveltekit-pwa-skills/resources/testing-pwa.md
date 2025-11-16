# Testing and Debugging SvelteKit PWAs

## Overview

Comprehensive testing and debugging strategies for SvelteKit PWAs, covering unit tests, integration tests, E2E tests, PWA-specific testing, and debugging techniques.

## Testing Setup

### Test Environment Configuration

```javascript
// vitest.config.js
import { defineConfig } from 'vitest/config';
import { svelte } from '@sveltejs/vite-plugin-svelte';
import { resolve } from 'path';

export default defineConfig({
  plugins: [svelte({ hot: !process.env.VITEST })],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./tests/setup.js'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html', 'lcov'],
      exclude: [
        'node_modules/',
        'tests/',
        '*.config.js',
        '**/+*.js',
        '**/*.test.js'
      ]
    },
    alias: {
      '$lib': resolve('./src/lib'),
      '$app': resolve('./src/app')
    }
  }
});
```

### Test Setup File

```javascript
// tests/setup.js
import '@testing-library/jest-dom';
import { vi } from 'vitest';

// Mock SvelteKit modules
vi.mock('$app/environment', () => ({
  browser: true,
  dev: false,
  building: false,
  version: 'test'
}));

vi.mock('$app/navigation', () => ({
  goto: vi.fn(),
  replaceState: vi.fn(),
  pushState: vi.fn(),
  invalidate: vi.fn(),
  invalidateAll: vi.fn(),
  afterNavigate: vi.fn()
}));

vi.mock('$app/stores', () => {
  const writable = (initial) => {
    let value = initial;
    const subscribers = new Set();
    
    return {
      subscribe: (fn) => {
        subscribers.add(fn);
        fn(value);
        return () => subscribers.delete(fn);
      },
      set: (newValue) => {
        value = newValue;
        subscribers.forEach(fn => fn(value));
      }
    };
  };
  
  return {
    page: writable({ 
      url: new URL('http://localhost'),
      params: {},
      route: { id: '/' }
    }),
    navigating: writable(null),
    updated: writable(false)
  };
});

// Mock Service Worker API
global.navigator.serviceWorker = {
  register: vi.fn().mockResolvedValue({}),
  ready: Promise.resolve({
    pushManager: {
      subscribe: vi.fn(),
      getSubscription: vi.fn()
    }
  })
};

// Mock Notification API
global.Notification = {
  permission: 'default',
  requestPermission: vi.fn().mockResolvedValue('granted')
};
```

## Unit Testing

### Component Testing

```javascript
// tests/components/Button.test.js
import { render, fireEvent, screen } from '@testing-library/svelte';
import { describe, it, expect, vi } from 'vitest';
import Button from '$lib/components/Button.svelte';

describe('Button Component', () => {
  it('renders with default props', () => {
    render(Button);
    const button = screen.getByRole('button');
    expect(button).toBeInTheDocument();
    expect(button).not.toBeDisabled();
  });
  
  it('renders with custom text', () => {
    render(Button, { props: { text: 'Click me' } });
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });
  
  it('handles click events', async () => {
    const handleClick = vi.fn();
    const { component } = render(Button, { 
      props: { text: 'Click' } 
    });
    
    component.$on('click', handleClick);
    
    const button = screen.getByRole('button');
    await fireEvent.click(button);
    
    expect(handleClick).toHaveBeenCalledTimes(1);
  });
  
  it('applies disabled state', () => {
    render(Button, { props: { disabled: true } });
    expect(screen.getByRole('button')).toBeDisabled();
  });
  
  it('applies custom classes', () => {
    render(Button, { 
      props: { class: 'custom-class' } 
    });
    expect(screen.getByRole('button')).toHaveClass('custom-class');
  });
});
```

### Store Testing

```javascript
// tests/stores/auth.test.js
import { describe, it, expect, beforeEach } from 'vitest';
import { get } from 'svelte/store';
import { authStore } from '$lib/stores/auth';

describe('Auth Store', () => {
  beforeEach(() => {
    authStore.reset();
  });
  
  it('initializes with default state', () => {
    const state = get(authStore);
    expect(state).toEqual({
      user: null,
      isAuthenticated: false,
      loading: false,
      error: null
    });
  });
  
  it('handles login success', async () => {
    const user = { id: 1, name: 'Test User' };
    
    await authStore.login('test@example.com', 'password');
    
    const state = get(authStore);
    expect(state.isAuthenticated).toBe(true);
    expect(state.user).toEqual(user);
    expect(state.error).toBe(null);
  });
  
  it('handles login failure', async () => {
    await authStore.login('invalid', 'invalid');
    
    const state = get(authStore);
    expect(state.isAuthenticated).toBe(false);
    expect(state.user).toBe(null);
    expect(state.error).toBeTruthy();
  });
  
  it('handles logout', () => {
    authStore.logout();
    
    const state = get(authStore);
    expect(state.isAuthenticated).toBe(false);
    expect(state.user).toBe(null);
  });
});
```

## Integration Testing

### API Route Testing

```javascript
// tests/routes/api/users.test.js
import { describe, it, expect, beforeEach } from 'vitest';
import { GET, POST } from '$lib/server/api/users';

describe('Users API', () => {
  beforeEach(() => {
    // Reset database or mocks
  });
  
  describe('GET /api/users', () => {
    it('returns list of users', async () => {
      const response = await GET({
        url: new URL('http://localhost/api/users'),
        params: {},
        locals: { user: { role: 'admin' } }
      });
      
      const data = await response.json();
      
      expect(response.status).toBe(200);
      expect(Array.isArray(data.users)).toBe(true);
    });
    
    it('requires authentication', async () => {
      const response = await GET({
        url: new URL('http://localhost/api/users'),
        params: {},
        locals: {}
      });
      
      expect(response.status).toBe(401);
    });
    
    it('filters by query parameters', async () => {
      const url = new URL('http://localhost/api/users');
      url.searchParams.set('role', 'admin');
      
      const response = await GET({
        url,
        params: {},
        locals: { user: { role: 'admin' } }
      });
      
      const data = await response.json();
      
      expect(data.users.every(u => u.role === 'admin')).toBe(true);
    });
  });
  
  describe('POST /api/users', () => {
    it('creates new user', async () => {
      const request = new Request('http://localhost/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          name: 'New User',
          email: 'new@example.com'
        })
      });
      
      const response = await POST({
        request,
        locals: { user: { role: 'admin' } }
      });
      
      const data = await response.json();
      
      expect(response.status).toBe(201);
      expect(data.user.email).toBe('new@example.com');
    });
    
    it('validates input data', async () => {
      const request = new Request('http://localhost/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          name: '', // Invalid
          email: 'invalid' // Invalid
        })
      });
      
      const response = await POST({
        request,
        locals: { user: { role: 'admin' } }
      });
      
      expect(response.status).toBe(400);
    });
  });
});
```

## E2E Testing with Playwright

### PWA Installation Test

```javascript
// tests/e2e/pwa-install.test.js
import { test, expect } from '@playwright/test';

test.describe('PWA Installation', () => {
  test('shows install prompt', async ({ page, context }) => {
    // Grant permissions
    await context.grantPermissions(['notifications']);
    
    await page.goto('/');
    
    // Wait for install prompt
    const installButton = await page.waitForSelector(
      '[data-testid="install-prompt"]',
      { timeout: 10000 }
    );
    
    expect(installButton).toBeTruthy();
    
    // Click install
    await installButton.click();
    
    // Verify installation UI changes
    await expect(page.locator('[data-testid="install-success"]'))
      .toBeVisible();
  });
  
  test('works offline after caching', async ({ page, context }) => {
    // Visit pages while online
    await page.goto('/');
    await page.goto('/about');
    await page.goto('/contact');
    
    // Wait for service worker
    await page.waitForTimeout(2000);
    
    // Go offline
    await context.setOffline(true);
    
    // Navigate to cached page
    await page.goto('/about');
    
    // Should not show offline page
    await expect(page.locator('h1')).not.toContainText('Offline');
    await expect(page.locator('h1')).toContainText('About');
  });
  
  test('shows offline page for uncached routes', async ({ page, context }) => {
    await page.goto('/');
    
    // Go offline immediately
    await context.setOffline(true);
    
    // Try to visit uncached page
    await page.goto('/uncached-page');
    
    // Should show offline page
    await expect(page.locator('h1')).toContainText('Offline');
  });
});
```

### Mobile Testing

```javascript
// tests/e2e/mobile.test.js
import { test, devices, expect } from '@playwright/test';

// Test on mobile viewport
test.use({ ...devices['iPhone 13'] });

test.describe('Mobile Experience', () => {
  test('responsive navigation works', async ({ page }) => {
    await page.goto('/');
    
    // Mobile menu should be hidden
    await expect(page.locator('.desktop-nav')).not.toBeVisible();
    
    // Open mobile menu
    await page.click('[data-testid="mobile-menu-toggle"]');
    
    // Menu should be visible
    await expect(page.locator('.mobile-nav')).toBeVisible();
    
    // Navigate via mobile menu
    await page.click('.mobile-nav a[href="/about"]');
    await expect(page).toHaveURL('/about');
    
    // Menu should close after navigation
    await expect(page.locator('.mobile-nav')).not.toBeVisible();
