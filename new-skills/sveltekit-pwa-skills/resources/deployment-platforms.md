# Deployment Strategies for SvelteKit PWAs

## Overview

This guide covers deployment strategies, adapter configurations, CI/CD pipelines, monitoring, and production best practices for SvelteKit PWAs across various platforms.

## Adapter Selection Guide

### Platform Comparison Matrix

| Platform | Adapter | Best For | Limitations |
|----------|---------|----------|-------------|
| Vercel | `@sveltejs/adapter-vercel` | Edge functions, automatic scaling | Vendor lock-in, cost at scale |
| Netlify | `@sveltejs/adapter-netlify` | Static sites with functions | Build time limits |
| Cloudflare | `@sveltejs/adapter-cloudflare` | Global edge deployment | Workers limitations |
| Node.js | `@sveltejs/adapter-node` | Full control, containers | Manual scaling needed |
| Static | `@sveltejs/adapter-static` | CDN deployment, GitHub Pages | No SSR |
| Auto | `@sveltejs/adapter-auto` | Platform detection | Limited customization |

## Platform-Specific Deployments

### Vercel Deployment

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-vercel';

export default {
  kit: {
    adapter: adapter({
      runtime: 'edge', // or 'nodejs20.x'
      regions: ['iad1'], // deployment regions
      split: true, // split functions for better cold starts
      external: [], // external packages
      includeFiles: ['./data/**/*'] // additional files
    })
  }
};
```

```json
// vercel.json
{
  "functions": {
    "src/routes/api/heavy/+server.js": {
      "maxDuration": 60,
      "memory": 3008
    }
  },
  "crons": [
    {
      "path": "/api/cron/daily",
      "schedule": "0 0 * * *"
    }
  ],
  "headers": [
    {
      "source": "/service-worker.js",
      "headers": [
        {
          "key": "Cache-Control",
          "value": "no-cache, no-store, must-revalidate"
        }
      ]
    },
    {
      "source": "/(.*)",
      "headers": [
        {
          "key": "X-Frame-Options",
          "value": "DENY"
        },
        {
          "key": "X-Content-Type-Options",
          "value": "nosniff"
        },
        {
          "key": "Referrer-Policy",
          "value": "strict-origin-when-cross-origin"
        }
      ]
    }
  ],
  "rewrites": [
    {
      "source": "/api/(.*)",
      "destination": "/api/$1"
    }
  ]
}
```

### Netlify Deployment

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-netlify';

export default {
  kit: {
    adapter: adapter({
      edge: true, // Use edge functions
      split: true // Split routes into separate functions
    })
  }
};
```

```toml
# netlify.toml
[build]
  command = "npm run build"
  publish = "build"

[build.environment]
  NODE_VERSION = "20"

[[headers]]
  for = "/service-worker.js"
  [headers.values]
    Cache-Control = "no-cache, no-store, must-revalidate"

[[headers]]
  for = "/*"
  [headers.values]
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
    Permissions-Policy = "camera=(), microphone=(), geolocation=()"

[[redirects]]
  from = "/api/*"
  to = "/.netlify/functions/:splat"
  status = 200

[[plugins]]
  package = "@netlify/plugin-lighthouse"
  
  [plugins.inputs]
    audit_url = "/"
    thresholds = { performance = 0.9, accessibility = 0.9, pwa = 0.9 }
```

### Cloudflare Pages/Workers

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-cloudflare';

export default {
  kit: {
    adapter: adapter({
      routes: {
        include: ['/*'],
        exclude: ['<all>']
      }
    })
  }
};
```

```javascript
// wrangler.toml
name = "sveltekit-pwa"
compatibility_date = "2024-01-01"
main = "./.cloudflare/worker.js"
site.bucket = "./.cloudflare/public"

[build]
command = "npm run build"

[env.production]
vars = { ENVIRONMENT = "production" }
kv_namespaces = [
  { binding = "CACHE", id = "cache_namespace_id" }
]
durable_objects.bindings = [
  { name = "COUNTER", class_name = "Counter", script_name = "counter" }
]

[[r2_buckets]]
binding = "STORAGE"
bucket_name = "pwa-storage"

[observability]
enabled = true
```

### Docker Deployment

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder

# Install dependencies
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Build app
COPY . .
RUN npm run build

# Runtime stage
FROM node:20-alpine

WORKDIR /app

# Copy built app
COPY --from=builder /app/build build/
COPY --from=builder /app/package*.json ./

# Install production dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "fetch('http://localhost:3000/api/health').then(r => process.exit(r.ok ? 0 : 1))"

EXPOSE 3000
ENV NODE_ENV=production

USER node

CMD ["node", "build"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
      - NODE_ENV=production
    depends_on:
      - redis
      - postgres
    restart: unless-stopped
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data
    restart: unless-stopped

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=pwa_db
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - app
    restart: unless-stopped

volumes:
  redis_data:
  postgres_data:
```

### Static Site Deployment

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-static';

export default {
  kit: {
    adapter: adapter({
      pages: 'build',
      assets: 'build',
      fallback: 'index.html',
      precompress: true,
      strict: true
    }),
    prerender: {
      entries: ['*'],
      handleHttpError: ({ path, referrer, message }) => {
        // Log errors but don't fail build
        console.warn(`Failed to prerender ${path}`, message);
      }
    }
  }
};
```

## CI/CD Pipelines

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy PWA

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Run E2E tests
        run: |
          npx playwright install
          npm run test:e2e
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results
          path: test-results/

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
        env:
          PUBLIC_API_URL: ${{ secrets.API_URL }}
          PUBLIC_VAPID_KEY: ${{ secrets.VAPID_PUBLIC_KEY }}
      
      - name: Run Lighthouse CI
        run: |
          npm install -g @lhci/cli
          lhci autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
      
      - name: Upload build artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build
          path: build/

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      
      - name: Download build artifacts
        uses: actions/download-artifact@v3
        with:
          name: build
          path: build/
      
      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v20
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
      
      - name: Purge CDN Cache
        run: |
          curl -X POST "https://api.cloudflare.com/client/v4/zones/${{ secrets.CF_ZONE_ID }}/purge_cache" \
            -H "X-Auth-Email: ${{ secrets.CF_EMAIL }}" \
            -H "X-Auth-Key: ${{ secrets.CF_API_KEY }}" \
            -H "Content-Type: application/json" \
            --data '{"purge_everything":true}'
      
      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'PWA deployed to production'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### GitLab CI

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  NODE_VERSION: "20"

cache:
  paths:
    - node_modules/
    - .npm/

before_script:
