# Deployment and Operations - SvelteKit Production

**Production-grade deployment patterns for SvelteKit applications**

This resource provides comprehensive deployment and operational strategies for SvelteKit applications across various platforms, with emphasis on security, reliability, and observability.

---

## 1. SvelteKit Adapters Overview

SvelteKit uses **adapters** to deploy to different platforms. Choose the adapter that matches your deployment target.

### Adapter Selection Guide

| Platform | Adapter | Use Case | Features |
|----------|---------|----------|----------|
| **Vercel** | `@sveltejs/adapter-vercel` | Serverless, edge functions | Edge middleware, image optimization, ISR |
| **Netlify** | `@sveltejs/adapter-netlify` | Serverless, edge functions | Edge functions, split testing, forms |
| **Cloudflare** | `@sveltejs/adapter-cloudflare` | Edge workers | Global CDN, KV storage, Durable Objects |
| **Node.js** | `@sveltejs/adapter-node` | Traditional servers, VPS | Full control, long-running processes |
| **Static** | `@sveltejs/adapter-static` | Static hosting (S3, GitHub Pages) | Maximum performance, CDN-friendly |
| **Auto** | `@sveltejs/adapter-auto` | Auto-detect platform | Development, zero config |

---

## 2. Adapter-Specific Configuration

### Vercel Adapter

```bash
npm install -D @sveltejs/adapter-vercel
```

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-vercel';

export default {
  kit: {
    adapter: adapter({
      runtime: 'nodejs20.x', // or 'edge'
      regions: ['iad1'], // Deployment regions
      split: false, // Split into multiple functions
      memory: 1024, // MB
      maxDuration: 10 // seconds
    })
  }
};
```

**Edge Runtime (Recommended for Performance):**

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-vercel';

export default {
  kit: {
    adapter: adapter({
      runtime: 'edge',
      regions: 'all' // Deploy to all edge locations
    })
  }
};
```

**Per-Route Configuration:**

```typescript
// src/routes/api/fast/+server.ts
export const config = {
  runtime: 'edge'
};

export const GET = async () => {
  return new Response('Fast edge response');
};
```

### Netlify Adapter

```bash
npm install -D @sveltejs/adapter-netlify
```

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-netlify';

export default {
  kit: {
    adapter: adapter({
      edge: false, // Use edge functions
      split: false // Split into multiple functions
    })
  }
};
```

**Netlify Edge Functions:**

```javascript
import adapter from '@sveltejs/adapter-netlify';

export default {
  kit: {
    adapter: adapter({
      edge: true // Deploy to Netlify Edge
    })
  }
};
```

### Cloudflare Pages/Workers Adapter

```bash
npm install -D @sveltejs/adapter-cloudflare
```

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-cloudflare';

export default {
  kit: {
    adapter: adapter({
      routes: {
        include: ['/*'],
        exclude: ['<build>', '<files>']
      }
    })
  }
};
```

**Using Cloudflare KV:**

```typescript
// src/routes/api/cache/+server.ts
interface Env {
  CACHE: KVNamespace;
}

export const GET = async ({ platform }) => {
  const env = platform.env as Env;

  // Read from KV
  const cached = await env.CACHE.get('key');

  if (cached) {
    return new Response(cached);
  }

  // Write to KV
  const data = 'new data';
  await env.CACHE.put('key', data, { expirationTtl: 3600 });

  return new Response(data);
};
```

### Node.js Adapter (Self-Hosted)

```bash
npm install -D @sveltejs/adapter-node
```

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-node';

export default {
  kit: {
    adapter: adapter({
      out: 'build',
      precompress: true, // Generate .gz and .br files
      envPrefix: 'APP_' // Environment variable prefix
    })
  }
};
```

**Custom Server with Express:**

```typescript
// server.js
import express from 'express';
import compression from 'compression';
import helmet from 'helmet';
import { handler } from './build/handler.js';

const app = express();
const PORT = process.env.PORT || 3000;

// Security middleware
app.use(helmet({
  contentSecurityPolicy: false // SvelteKit handles CSP
}));

// Compression
app.use(compression());

// Rate limiting
import rateLimit from 'express-rate-limit';
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});
app.use('/api/', limiter);

// Health check
app.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

// SvelteKit handler
app.use(handler);

const server = app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM received, closing server...');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
```

**Docker Deployment:**

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build
RUN npm prune --production

# Production image
FROM node:20-alpine

WORKDIR /app

# Security: Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S sveltekit -u 1001

# Copy built application
COPY --from=builder --chown=sveltekit:nodejs /app/build ./build
COPY --from=builder --chown=sveltekit:nodejs /app/node_modules ./node_modules
COPY --from=builder --chown=sveltekit:nodejs /app/package.json ./

USER sveltekit

EXPOSE 3000

ENV NODE_ENV=production
ENV PORT=3000

CMD ["node", "build"]
```

**Docker Compose for Local Testing:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/mydb
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

### Static Adapter (Static Site Generation)

```bash
npm install -D @sveltejs/adapter-static
```

```javascript
// svelte.config.js
import adapter from '@sveltejs/adapter-static';

export default {
  kit: {
    adapter: adapter({
      pages: 'build',
      assets: 'build',
      fallback: '404.html', // SPA fallback
      precompress: true,
      strict: true // Fail build if dynamic route found
    })
  }
};
```

**Prerendering All Routes:**

```typescript
// src/routes/+layout.ts
export const prerender = true;
export const ssr = false; // Optional: disable SSR for SPA mode
```

---

## 3. Environment Variable Management

### Development vs Production

```bash
# .env.development (local development)
PUBLIC_API_URL=http://localhost:4000
DATABASE_URL=postgresql://localhost/mydb_dev
REDIS_URL=redis://localhost:6379

# .env.production (production - NOT committed)
PUBLIC_API_URL=https://api.example.com
DATABASE_URL=postgresql://prod-server/mydb
REDIS_URL=redis://prod-cache:6379
SECRET_KEY=super_secret_key
```

### Secure Environment Variables

```typescript
// src/lib/server/env.ts
import { z } from 'zod';

const envSchema = z.object({
  // Public variables (available in browser)
  PUBLIC_API_URL: z.string().url(),

  // Private variables (server-only)
  DATABASE_URL: z.string().min(1),
  SECRET_KEY: z.string().min(32),
  REDIS_URL: z.string().url()
});

// Validate on app start
const parsed = envSchema.safeParse(process.env);

if (!parsed.success) {
  console.error('❌ Invalid environment variables:', parsed.error.flatten().fieldErrors);
  throw new Error('Invalid environment variables');
}

export const env = parsed.data;
```

**Usage:**

```typescript
// src/routes/api/data/+server.ts
import { env } from '$lib/server/env';

export const GET = async () => {
  // ✅ Safe - validated on startup
  const response = await fetch(`${env.PUBLIC_API_URL}/data`, {
    headers: {
      'Authorization': `Bearer ${env.SECRET_KEY}`
    }
  });

  return response;
};
```

### Platform-Specific Environment Variables

**Vercel:**

```bash
# Via CLI
vercel env add SECRET_KEY production
vercel env add DATABASE_URL production

# Via dashboard: vercel.com/project/settings/environment-variables
```

**Netlify:**

```bash
# Via CLI
netlify env:set SECRET_KEY "value" --scope production

# Via dashboard: app.netlify.com/sites/[site]/settings/deploys#environment
```

**Docker:**

```yaml
# docker-compose.yml
services:
  web:
    build: .
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - SECRET_KEY=${SECRET_KEY}
    # Or use env_file
    env_file:
      - .env.production
```

**Kubernetes:**

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  SECRET_KEY: "super_secret_key"
  DATABASE_URL: "postgresql://..."
---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: web
          env:
            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: SECRET_KEY
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: DATABASE_URL
```

---

## 4. CI/CD Pipeline Setup

### GitHub Actions

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'

jobs:
  # Step 1: Test
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run type check
        run: npm run check

      - name: Run tests
        run: npm test

  # Step 2: Security scan
  security:
    runs-on: ubuntu-latest
    needs: test

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Install dependencies
        run: npm ci

      - name: Run security audit
        run: npm audit --audit-level=moderate

      - name: Run Snyk scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # Step 3: Build
  build:
    runs-on: ubuntu-latest
    needs: [test, security]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build
        env:
          PUBLIC_API_URL: ${{ secrets.PUBLIC_API_URL }}

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: build
          path: build/

  # Step 4: Deploy
  deploy:
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: build
          path: build/

      - name: Deploy to Vercel
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'

  # Step 5: Lighthouse performance audit
  lighthouse:
    runs-on: ubuntu-latest
    needs: deploy
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Run Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          urls: |
            https://example.com
            https://example.com/dashboard
          uploadArtifacts: true
          temporaryPublicStorage: true
```

### GitLab CI/CD

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  NODE_VERSION: '20'

# Caching
cache:
  paths:
    - node_modules/
    - .npm/

# Test stage
test:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run lint
    - npm run check
    - npm test
  artifacts:
    reports:
      junit: junit.xml

security:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm audit --audit-level=moderate
  allow_failure: false

# Build stage
build:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - build/
    expire_in: 1 day

# Deploy to production
deploy:production:
  stage: deploy
  image: node:${NODE_VERSION}
  script:
    - npm install -g vercel
    - vercel --token $VERCEL_TOKEN --prod
  only:
    - main
  environment:
    name: production
    url: https://example.com
```

---

## 5. CDN Configuration

### Cloudflare CDN

```javascript
// svelte.config.js for static sites
import adapter from '@sveltejs/adapter-static';

export default {
  kit: {
    adapter: adapter({
      precompress: true // Generate .br and .gz for Cloudflare
    })
  }
};
```

**Cache Rules (Cloudflare Dashboard):**

- **Static assets** (`/build/*`, `*.js`, `*.css`):
  - Cache Level: Standard
  - Edge TTL: 1 year
  - Browser TTL: 1 year

- **HTML pages** (`*.html`, `/`):
  - Cache Level: Standard
  - Edge TTL: 4 hours
  - Browser TTL: 0 (revalidate)

- **API routes** (`/api/*`):
  - Cache Level: Bypass
  - No caching

### AWS CloudFront

```json
{
  "Comment": "SvelteKit CDN Distribution",
  "Origins": [
    {
      "Id": "S3-sveltekit-app",
      "DomainName": "sveltekit-app.s3.amazonaws.com",
      "S3OriginConfig": {
        "OriginAccessIdentity": "origin-access-identity/cloudfront/EXAMPLE"
      }
    }
  ],
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-sveltekit-app",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "CacheBehaviors": [
    {
      "PathPattern": "/build/*",
      "TargetOriginId": "S3-sveltekit-app",
      "ViewerProtocolPolicy": "redirect-to-https",
      "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
      "Compress": true,
      "DefaultTTL": 31536000
    }
  ]
}
```

---

## 6. Monitoring and Logging

### Sentry Error Monitoring

```bash
npm install @sentry/sveltekit
npx @sentry/wizard@latest -i sveltekit
```

```typescript
// src/hooks.client.ts
import * as Sentry from '@sentry/sveltekit';

Sentry.init({
  dsn: 'https://example@sentry.io/123',
  environment: import.meta.env.MODE,
  tracesSampleRate: 0.1, // 10% of transactions
  replaysSessionSampleRate: 0.1, // 10% of sessions
  replaysOnErrorSampleRate: 1.0, // 100% of errors
  integrations: [
    new Sentry.BrowserTracing(),
    new Sentry.Replay({
      maskAllText: true,
      blockAllMedia: true
    })
  ]
});
```

```typescript
// src/hooks.server.ts
import * as Sentry from '@sentry/sveltekit';

Sentry.init({
  dsn: 'https://example@sentry.io/123',
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1
});

export const handleError = ({ error, event }) => {
  Sentry.captureException(error, { contexts: { sveltekit: { event } } });

  return {
    message: 'Internal server error',
    code: error.code ?? 'UNKNOWN'
  };
};
```

### Structured Logging

```typescript
// src/lib/server/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  formatters: {
    level: (label) => {
      return { level: label };
    }
  },
  timestamp: pino.stdTimeFunctions.isoTime,
  // Production: JSON format
  // Development: Pretty print
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty' }
    : undefined
});
```

**Usage in routes:**

```typescript
// src/routes/api/users/+server.ts
import { logger } from '$lib/server/logger';

export const GET = async ({ locals }) => {
  logger.info({ userId: locals.user?.id }, 'Fetching users');

  try {
    const users = await db.user.findMany();

    logger.info({ count: users.length }, 'Users fetched successfully');

    return json(users);
  } catch (error) {
    logger.error({ error }, 'Failed to fetch users');
    throw error;
  }
};
```

### Performance Monitoring with Vercel Analytics

```bash
npm install @vercel/analytics
```

```svelte
<!-- src/routes/+layout.svelte -->
<script>
  import { dev } from '$app/environment';
  import { inject } from '@vercel/analytics';

  inject({ mode: dev ? 'development' : 'production' });
</script>
```

---

## 7. Blue-Green Deployments

### Using Vercel Preview Deployments

```bash
# Automatic preview for every PR
git checkout -b feature/new-feature
git push origin feature/new-feature

# Vercel automatically creates preview: https://app-git-feature-new-feature.vercel.app
```

### Manual Blue-Green with Docker Swarm

```bash
# Deploy green version
docker service create \
  --name web-green \
  --replicas 3 \
  --publish 8080:3000 \
  myapp:v2

# Health check
curl http://localhost:8080/health

# Switch traffic (update load balancer)
# Route traffic from blue (v1) to green (v2)

# Remove blue version after successful deployment
docker service rm web-blue
```

### Kubernetes Blue-Green

```yaml
# deployment-green.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      version: green
  template:
    metadata:
      labels:
        app: web
        version: green
    spec:
      containers:
        - name: web
          image: myapp:v2
          ports:
            - containerPort: 3000
---
# service.yaml (switch selector to green)
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
    version: green # Switch from blue to green
  ports:
    - port: 80
      targetPort: 3000
```

---

## 8. Rollback Procedures

### Vercel Instant Rollback

```bash
# List deployments
vercel ls

# Rollback to previous deployment
vercel rollback [deployment-url]
```

### Docker Rollback

```bash
# List previous images
docker images myapp

# Rollback to previous tag
docker service update --image myapp:v1 web
```

### Kubernetes Rollback

```bash
# Check rollout history
kubectl rollout history deployment/web

# Rollback to previous version
kubectl rollout undo deployment/web

# Rollback to specific revision
kubectl rollout undo deployment/web --to-revision=2

# Monitor rollback
kubectl rollout status deployment/web
```

---

## 9. Health Checks and Readiness Probes

### SvelteKit Health Check Endpoint

```typescript
// src/routes/health/+server.ts
import { json } from '@sveltejs/kit';
import type { RequestHandler } from './$types';
import { db } from '$lib/server/db';

export const GET: RequestHandler = async () => {
  const checks = {
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    checks: {
      database: 'unknown',
      memory: 'unknown'
    }
  };

  // Database check
  try {
    await db.$queryRaw`SELECT 1`;
    checks.checks.database = 'healthy';
  } catch (error) {
    checks.checks.database = 'unhealthy';
    checks.status = 'degraded';
  }

  // Memory check
  const memUsage = process.memoryUsage();
  const memPercent = (memUsage.heapUsed / memUsage.heapTotal) * 100;

  if (memPercent > 90) {
    checks.checks.memory = 'warning';
    checks.status = 'degraded';
  } else {
    checks.checks.memory = 'healthy';
  }

  const statusCode = checks.status === 'ok' ? 200 : 503;

  return json(checks, { status: statusCode });
};
```

### Kubernetes Probes

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: web
          image: myapp:latest
          ports:
            - containerPort: 3000

          # Liveness probe - restart if unhealthy
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

          # Readiness probe - remove from service if not ready
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 2

          # Startup probe - allow slow startup
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 30
            periodSeconds: 10
```

---

## 10. Disaster Recovery

### Automated Backups

```yaml
# backup-cronjob.yaml (Kubernetes)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *" # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16
              command:
                - /bin/sh
                - -c
                - |
                  pg_dump $DATABASE_URL | gzip > /backups/db-$(date +%Y%m%d-%H%M%S).sql.gz
                  # Upload to S3
                  aws s3 cp /backups/ s3://my-backups/database/ --recursive
              envFrom:
                - secretRef:
                    name: app-secrets
          restartPolicy: OnFailure
```

### Multi-Region Deployment

```javascript
// vercel.json - Multi-region deployment
{
  "regions": ["iad1", "sfo1", "cdg1"], // US East, US West, Europe
  "github": {
    "silent": true
  }
}
```

---

**Related Resources:**
- [Security Hardening](./security-hardening.md)
- [Performance Optimization](./performance-optimization.md)
- [Accessibility and UX](./accessibility-ux.md)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Compliance**: Production deployment best practices
