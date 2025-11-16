# Deployment CI/CD and Operations - SvelteKit PWAs

**CI/CD pipelines, monitoring, and operational excellence**

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
  - npm ci --cache .npm --prefer-offline

test:
  stage: test
  image: node:${NODE_VERSION}
  script:
    - npm run test
    - npm run test:e2e
  artifacts:
    reports:
      junit: test-results/junit.xml
    when: always

build:
  stage: build
  image: node:${NODE_VERSION}
  script:
    - npm run build
  artifacts:
    paths:
      - build/
    expire_in: 1 week

deploy:
  stage: deploy
  image: alpine:latest
  dependencies:
    - build
  only:
    - main
  script:
    - apk add --no-cache curl
    - tar -czf build.tar.gz build/
    - 'curl -X POST -H "Authorization: Bearer $DEPLOY_TOKEN" -F "file=@build.tar.gz" https://deploy.example.com/upload'
```

## Environment Configuration

### Environment Variables Management

```javascript
// lib/env.js
import { dev } from '$app/environment';

const requiredEnvVars = [
  'DATABASE_URL',
  'REDIS_URL',
  'JWT_SECRET',
  'VAPID_PRIVATE_KEY',
  'VAPID_PUBLIC_KEY'
];

export function validateEnv() {
  if (dev) return; // Skip in development
  
  const missing = requiredEnvVars.filter(
    key => !process.env[key]
  );
  
  if (missing.length > 0) {
    throw new Error(
      `Missing required environment variables: ${missing.join(', ')}`
    );
  }
}

export function getEnv(key, defaultValue = '') {
  return process.env[key] || defaultValue;
}

// Validate on startup
validateEnv();
```

### Multi-Environment Configuration

```javascript
// config/environments.js
const environments = {
  development: {
    apiUrl: 'http://localhost:3000/api',
    wsUrl: 'ws://localhost:3000',
    features: {
      debug: true,
      analytics: false,
      pushNotifications: false
    }
  },
  staging: {
    apiUrl: 'https://staging-api.example.com',
    wsUrl: 'wss://staging-ws.example.com',
    features: {
      debug: true,
      analytics: true,
      pushNotifications: true
    }
  },
  production: {
    apiUrl: 'https://api.example.com',
    wsUrl: 'wss://ws.example.com',
    features: {
      debug: false,
      analytics: true,
      pushNotifications: true
    }
  }
};

export const config = environments[process.env.NODE_ENV || 'development'];
```

## Monitoring and Analytics

### Application Performance Monitoring

```javascript
// lib/monitoring.js
import * as Sentry from '@sentry/sveltekit';

// Initialize Sentry
Sentry.init({
  dsn: import.meta.env.VITE_SENTRY_DSN,
  environment: import.meta.env.MODE,
  integrations: [
    new Sentry.BrowserTracing(),
    new Sentry.Replay()
  ],
  tracesSampleRate: import.meta.env.PROD ? 0.1 : 1.0,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0
});

// Custom performance monitoring
export class PerformanceMonitor {
  constructor() {
    this.metrics = new Map();
  }
  
  mark(name) {
    performance.mark(name);
    this.metrics.set(name, performance.now());
  }
  
  measure(name, startMark, endMark) {
    performance.measure(name, startMark, endMark);
    
    const measure = performance.getEntriesByName(name)[0];
    
    // Send to analytics
    this.sendMetric({
      name,
      duration: measure.duration,
      timestamp: Date.now()
    });
  }
  
  async sendMetric(metric) {
    // Send to monitoring service
    await fetch('/api/metrics', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(metric)
    });
  }
  
  reportWebVitals() {
    if ('web-vital' in window) {
      ['CLS', 'FID', 'LCP', 'FCP', 'TTFB'].forEach(metric => {
        window['web-vital'][metric](({ value, name }) => {
          this.sendMetric({ name, value });
        });
      });
    }
  }
}
```

### Health Checks and Monitoring Endpoints

```javascript
// src/routes/api/health/+server.js
import { json } from '@sveltejs/kit';

export async function GET() {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    version: process.env.APP_VERSION || 'unknown'
  };
  
  // Check dependencies
  const checks = await Promise.allSettled([
    checkDatabase(),
    checkRedis(),
    checkExternalAPI()
  ]);
  
  health.checks = checks.map((check, index) => ({
    name: ['database', 'redis', 'api'][index],
    status: check.status === 'fulfilled' ? 'healthy' : 'unhealthy',
    ...(check.status === 'rejected' && { error: check.reason.message })
  }));
  
  const isHealthy = checks.every(c => c.status === 'fulfilled');
  
  return json(health, {
    status: isHealthy ? 200 : 503,
    headers: {
      'Cache-Control': 'no-cache, no-store, must-revalidate'
    }
  });
}

// Readiness check
export async function HEAD() {
  try {
    await checkDatabase();
    return new Response(null, { status: 200 });
  } catch {
    return new Response(null, { status: 503 });
  }
}
```

## CDN and Edge Configuration

### Cloudflare CDN Rules

```javascript
// cloudflare-rules.js
export const cacheRules = [
  {
    pattern: '*.js',
    cache: {
      ttl: 31536000, // 1 year
      cacheControl: 'public, immutable'
    }
  },
  {
    pattern: '*.css',
    cache: {
      ttl: 31536000,
      cacheControl: 'public, immutable'
    }
  },
  {
    pattern: '/service-worker.js',
    cache: {
      ttl: 0,
      cacheControl: 'no-cache, no-store, must-revalidate'
    }
  },
  {
    pattern: '/api/*',
    cache: {
      ttl: 0,
      bypass: true
    }
  },
  {
    pattern: '*.webp',
    cache: {
      ttl: 86400, // 1 day
      cacheControl: 'public, max-age=86400'
    }
  }
];
```

### Edge Function for Geolocation

```javascript
// edge-functions/geolocation.js
export default {
  async fetch(request, env) {
    const country = request.cf?.country || 'US';
    const city = request.cf?.city || 'Unknown';
    
    // Route to nearest server
    const servers = {
      'US': 'https://us.example.com',
      'EU': 'https://eu.example.com',
      'ASIA': 'https://asia.example.com'
    };
    
    const region = getRegion(country);
    const server = servers[region] || servers['US'];
    
    // Add geo headers
    const response = await fetch(server + request.url.pathname, {
      headers: {
        ...request.headers,
        'X-Country': country,
        'X-City': city,
        'X-Region': region
      }
    });
    
    return response;
  }
};

function getRegion(country) {
  const regions = {
    'US': ['US', 'CA', 'MX'],
    'EU': ['GB', 'DE', 'FR', 'IT', 'ES'],
    'ASIA': ['JP', 'CN', 'KR', 'IN', 'SG']
  };
  
  for (const [region, countries] of Object.entries(regions)) {
    if (countries.includes(country)) return region;
  }
  
  return 'US';
}
```

## Security Headers

```javascript
// nginx.conf
server {
  listen 443 ssl http2;
  server_name example.com;
  
  # SSL Configuration
  ssl_certificate /etc/nginx/certs/cert.pem;
  ssl_certificate_key /etc/nginx/certs/key.pem;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;
  
  # Security Headers
  add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
  add_header X-Frame-Options "DENY" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-XSS-Protection "1; mode=block" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  add_header Permissions-Policy "camera=(), microphone=(), geolocation=(self)" always;
  
  # CSP Header
  add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:; connect-src 'self' https://api.example.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self';" always;
  
  # Proxy to app
  location / {
    proxy_pass http://app:3000;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
  
  # Cache static assets
  location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
    expires 1y;
    add_header Cache-Control "public, immutable";
  }
  
  # Don't cache service worker
  location = /service-worker.js {
    expires off;
    add_header Cache-Control "no-cache, no-store, must-revalidate";
  }
}
```

## Rollback Strategy

```bash
#!/bin/bash
# rollback.sh

# Get current and previous versions
CURRENT_VERSION=$(curl -s https://api.example.com/version)
PREVIOUS_VERSION=$(git tag --sort=-version:refname | sed -n '2p')

echo "Rolling back from $CURRENT_VERSION to $PREVIOUS_VERSION"

# Backup current version
docker tag app:latest app:rollback-$CURRENT_VERSION

# Deploy previous version
git checkout $PREVIOUS_VERSION
docker build -t app:$PREVIOUS_VERSION .
docker tag app:$PREVIOUS_VERSION app:latest

# Update services
docker-compose up -d --no-deps app

# Verify rollback
sleep 10
NEW_VERSION=$(curl -s https://api.example.com/version)

if [ "$NEW_VERSION" = "$PREVIOUS_VERSION" ]; then
  echo "Rollback successful"
  
  # Clear CDN cache
  curl -X POST "https://api.cloudflare.com/zones/$CF_ZONE_ID/purge_cache" \
    -H "Authorization: Bearer $CF_TOKEN" \
    -d '{"purge_everything":true}'
else
  echo "Rollback failed"
  exit 1
fi
```

## Deployment Checklist

- [ ] Environment variables configured
- [ ] SSL certificates installed
- [ ] Security headers configured
- [ ] Service worker cache busting implemented
- [ ] Database migrations run
- [ ] CDN cache rules configured
- [ ] Health check endpoints working
- [ ] Monitoring and alerts configured
- [ ] Backup strategy in place
- [ ] Rollback procedure tested
- [ ] Load testing completed
- [ ] PWA manifest validated
- [ ] Lighthouse scores meet targets
- [ ] Error tracking configured
- [ ] Analytics implemented

## Best Practices

1. **Use progressive deployment** - Canary, blue-green, or rolling deployments
2. **Implement health checks** - Ensure services are ready before routing traffic
3. **Configure auto-scaling** - Handle traffic spikes automatically
4. **Use CDN for static assets** - Reduce server load and improve performance
5. **Implement cache busting** - Ensure users get latest updates
6. **Monitor everything** - Logs, metrics, errors, and user behavior
7. **Automate deployments** - Reduce human error
8. **Test in staging** - Mirror production environment
9. **Document procedures** - Deployment, rollback, and incident response
10. **Regular backups** - Database and user-generated content

## References

- [SvelteKit Adapters](https://kit.svelte.dev/docs/adapters)
- [Vercel Documentation](https://vercel.com/docs)
- [Netlify Documentation](https://docs.netlify.com/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [GitHub Actions](https://docs.github.com/en/actions)
