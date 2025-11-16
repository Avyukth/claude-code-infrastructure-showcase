# Deployment and Operations - Rust Backend Production

**Secure deployment strategies and operational excellence for Rust services**

This resource covers secure Docker images, CI/CD pipelines, database migrations, deployment strategies, and incident response for production Rust backends.

---

## Table of Contents

- [Secure Docker Images](#secure-docker-images)
- [CI/CD Pipeline Security](#cicd-pipeline-security)
- [Environment Configuration](#environment-configuration)
- [Database Migrations](#database-migrations)
- [Deployment Strategies](#deployment-strategies)
- [Rollback Procedures](#rollback-procedures)
- [Incident Response](#incident-response)

---

## Secure Docker Images

### Multi-Stage Builds with Distroless

```dockerfile
# Dockerfile - Secure multi-stage build
FROM rust:1.75-slim as builder

WORKDIR /app

# Cache dependencies
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && \
    echo "fn main() {}" > src/main.rs && \
    cargo build --release && \
    rm -rf src

# Build application
COPY src ./src
RUN cargo build --release

# Minimal runtime image (distroless)
FROM gcr.io/distroless/cc-debian12

# Copy binary
COPY --from=builder /app/target/release/my-service /usr/local/bin/my-service

# Non-root user (distroless uses UID 65532)
USER 65532:65532

EXPOSE 8080

CMD ["/usr/local/bin/my-service"]
```

### Security Scanning

```dockerfile
# .dockerignore
target/
.git/
.env
*.md
Dockerfile
.dockerignore
```

```yaml
# .github/workflows/docker-scan.yml
name: Docker Security Scan

on:
  push:
    branches: [main]
  pull_request:

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t my-service:${{ github.sha }} .

      - name: Run Trivy scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: my-service:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

---

## CI/CD Pipeline Security

### GitHub Actions Secure Pipeline

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:

env:
  RUST_VERSION: "1.75"

jobs:
  # Security audit
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - name: Install cargo-audit
        run: cargo install cargo-audit

      - name: Security audit
        run: cargo audit --deny warnings

      - name: Install cargo-deny
        run: cargo install cargo-deny

      - name: Check dependencies
        run: cargo deny check

  # Test
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - uses: Swatinem/rust-cache@v2

      - name: Run tests
        run: cargo test --all-features

      - name: Run clippy
        run: cargo clippy -- -D warnings

  # Build
  build:
    needs: [audit, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - name: Build release
        run: cargo build --release

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: binary
          path: target/release/my-service

  # Deploy
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}

      - name: Build and push Docker image
        run: |
          docker build -t ${{ secrets.ECR_REGISTRY }}/my-service:${{ github.sha }} .
          docker push ${{ secrets.ECR_REGISTRY }}/my-service:${{ github.sha }}

      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster production \
            --service my-service \
            --force-new-deployment
```

---

## Environment Configuration

### Environment Variable Management

```rust
use std::env;
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct Config {
    pub database_url: String,
    pub redis_url: String,
    pub jwt_secret: String,
    pub port: u16,
    pub log_level: String,
}

impl Config {
    pub fn from_env() -> Result<Self, config::ConfigError> {
        Ok(Config {
            database_url: env::var("DATABASE_URL")
                .expect("DATABASE_URL must be set"),
            redis_url: env::var("REDIS_URL")
                .unwrap_or_else(|_| "redis://localhost:6379".to_string()),
            jwt_secret: env::var("JWT_SECRET")
                .expect("JWT_SECRET must be set"),
            port: env::var("PORT")
                .unwrap_or_else(|_| "8080".to_string())
                .parse()
                .expect("PORT must be a number"),
            log_level: env::var("LOG_LEVEL")
                .unwrap_or_else(|_| "info".to_string()),
        })
    }
}
```

### Kubernetes Secrets

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-service-secrets
type: Opaque
stringData:
  database-url: "postgresql://user:pass@db:5432/mydb"
  jwt-secret: "super-secret-key"
  redis-url: "redis://redis:6379"
---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
    spec:
      serviceAccountName: my-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
        fsGroup: 65532
      containers:
      - name: my-service
        image: my-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: my-service-secrets
              key: database-url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: my-service-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

---

## Database Migrations

### SQLx Migrations

```bash
# Create migration
sqlx migrate add create_users_table

# Run migrations
sqlx migrate run

# Revert last migration
sqlx migrate revert
```

```sql
-- migrations/20231115000001_create_users_table.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
```

### Run Migrations in Application

```rust
use sqlx::postgres::PgPool;

pub async fn run_migrations(pool: &PgPool) -> Result<(), sqlx::Error> {
    sqlx::migrate!("./migrations")
        .run(pool)
        .await?;
    Ok(())
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pool = PgPool::connect(&env::var("DATABASE_URL")?).await?;

    // Run migrations on startup
    run_migrations(&pool).await?;

    // Start server
    start_server(pool).await?;
    Ok(())
}
```

---

## Deployment Strategies

### Blue-Green Deployment

```yaml
# service.yaml (routes to active deployment)
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-service
    version: blue  # Switch to 'green' for cutover
  ports:
  - port: 80
    targetPort: 8080
---
# deployment-blue.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-service
      version: blue
  template:
    metadata:
      labels:
        app: my-service
        version: blue
    spec:
      containers:
      - name: my-service
        image: my-service:v1.0.0
---
# deployment-green.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-service
      version: green
  template:
    metadata:
      labels:
        app: my-service
        version: green
    spec:
      containers:
      - name: my-service
        image: my-service:v1.1.0  # New version
```

### Canary Deployment

```yaml
# Istio VirtualService for canary
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-service
spec:
  hosts:
  - my-service
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: my-service
        subset: v2
  - route:
    - destination:
        host: my-service
        subset: v1
      weight: 90
    - destination:
        host: my-service
        subset: v2
      weight: 10  # 10% traffic to canary
```

---

## Rollback Procedures

### Kubernetes Rollback

```bash
# Check rollout history
kubectl rollout history deployment/my-service

# Rollback to previous version
kubectl rollout undo deployment/my-service

# Rollback to specific revision
kubectl rollout undo deployment/my-service --to-revision=2

# Monitor rollback
kubectl rollout status deployment/my-service
```

### Automated Rollback on Failure

```yaml
# deployment.yaml with progressive rollout
apiVersion: apps/v1
kind: Deployment
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  minReadySeconds: 30
  progressDeadlineSeconds: 600
```

---

## Incident Response

### Incident Response Runbook

```markdown
# Incident Response Runbook

## Severity Levels
- **P1 (Critical)**: Service down, data loss
- **P2 (High)**: Major feature broken, degraded performance
- **P3 (Medium)**: Minor feature broken
- **P4 (Low)**: Cosmetic issues

## Response Steps

### 1. Detect (Automated Alerts)
- Monitor dashboards show anomalies
- Alerts trigger pager duty
- Users report issues

### 2. Assess
- Determine severity
- Check recent deployments
- Review error logs and metrics

### 3. Mitigate
- If recent deployment: rollback immediately
- If external dependency: enable circuit breaker
- If database issue: failover to replica

### 4. Resolve
- Apply permanent fix
- Run tests
- Deploy fix

### 5. Document
- Create postmortem
- Update runbook
- Implement preventive measures
```

### Graceful Shutdown

```rust
use tokio::signal;
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};

pub async fn run_server() {
    let shutdown = Arc::new(AtomicBool::new(false));
    let shutdown_clone = shutdown.clone();

    // Spawn signal handler
    tokio::spawn(async move {
        signal::ctrl_c().await.expect("Failed to listen for Ctrl+C");
        info!("Shutdown signal received");
        shutdown_clone.store(true, Ordering::SeqCst);
    });

    // Server loop
    while !shutdown.load(Ordering::SeqCst) {
        // Handle requests
        tokio::select! {
            _ = handle_requests() => {}
            _ = tokio::time::sleep(Duration::from_millis(100)) => {}
        }
    }

    info!("Shutting down gracefully...");

    // Drain in-flight requests (30 second grace period)
    tokio::time::sleep(Duration::from_secs(30)).await;

    info!("Shutdown complete");
}
```

---

## Deployment Checklist

### Pre-Deployment
- [ ] Security audit passed (cargo audit)
- [ ] All tests passing
- [ ] Database migrations tested
- [ ] Environment variables configured
- [ ] Docker image scanned for vulnerabilities
- [ ] Load testing completed
- [ ] Rollback plan documented

### Deployment
- [ ] Backup database before migration
- [ ] Run migrations in transaction
- [ ] Deploy to canary first (10% traffic)
- [ ] Monitor error rates and latency
- [ ] Gradually increase traffic
- [ ] Full deployment after 1 hour stable

### Post-Deployment
- [ ] Verify health checks passing
- [ ] Check error logs
- [ ] Monitor key metrics (latency, errors, saturation)
- [ ] Test critical user flows
- [ ] Document deployment in changelog

---

**Related Resources:**
- [Security Hardening](./security-hardening.md)
- [Monitoring and Observability](./monitoring.md)
- [Performance and Scalability](./performance.md)
- [Troubleshooting](./troubleshooting.md)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Compliance**: NIST SP 800-53 CM (Configuration Management), SA-10 (Developer Configuration Management)
