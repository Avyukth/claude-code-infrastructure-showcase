# Deployment Guide - Production Deployment Best Practices

Complete guide to deploying Rust backend services to production with Docker, CI/CD, monitoring, and operational best practices.

## Table of Contents

- [Docker Deployment](#docker-deployment)
- [CI/CD with GitHub Actions](#cicd-with-github-actions)
- [Monitoring and Observability](#monitoring-and-observability)
- [Logging with tracing-subscriber](#logging-with-tracing-subscriber)
- [Health Checks and Graceful Shutdown](#health-checks-and-graceful-shutdown)
- [Database Migrations](#database-migrations)
- [Environment Configuration](#environment-configuration)

---

## Docker Deployment

### Multi-Stage Dockerfile

```dockerfile
# Stage 1: Build dependencies (cached layer)
FROM rust:1.75-slim as chef
WORKDIR /app
RUN cargo install cargo-chef

# Stage 2: Prepare recipe
FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

# Stage 3: Build dependencies
FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json

# Copy source and build application
COPY . .
RUN cargo build --release --bin web-server

# Stage 4: Runtime
FROM debian:bookworm-slim AS runtime
WORKDIR /app

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl3 \
    && rm -rf /var/lib/apt/lists/*

# Copy binary from builder
COPY --from=builder /app/target/release/web-server /usr/local/bin/web-server

# Create non-root user
RUN useradd -m -u 1001 appuser && \
    chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["web-server"]
```

### Optimized for Size

```dockerfile
# Use musl for static binary
FROM rust:1.75-alpine as builder
WORKDIR /app

# Install musl-dev for static linking
RUN apk add --no-cache musl-dev

COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

# Minimal runtime image
FROM scratch
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/web-server /web-server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080
ENTRYPOINT ["/web-server"]
```

### Docker Compose for Development

```yaml
# docker-compose.yml

version: '3.8'

services:
  web-server:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/myapp
      - REDIS_URL=redis://redis:6379
      - RUST_LOG=info,web_server=debug
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    volumes:
      - ./migrations:/app/migrations

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

---

## CI/CD with GitHub Actions

### Complete CI/CD Pipeline

```yaml
# .github/workflows/ci.yml

name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Test Suite
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - uses: Swatinem/rust-cache@v2

      - name: Run migrations
        run: |
          cargo install sqlx-cli --no-default-features --features postgres
          sqlx database create
          sqlx migrate run
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db

      - name: Run tests
        run: cargo test --all-features --workspace
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db

      - name: Check formatting
        run: cargo fmt --all -- --check

      - name: Run clippy
        run: cargo clippy --all-targets --all-features -- -D warnings

  coverage:
    name: Code Coverage
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - name: Install tarpaulin
        run: cargo install cargo-tarpaulin

      - name: Generate coverage
        run: cargo tarpaulin --out Xml --workspace
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db

      - name: Upload to codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./cobertura.xml

  security:
    name: Security Audit
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - name: Install cargo-audit
        run: cargo install cargo-audit

      - name: Run security audit
        run: cargo audit

      - name: Install cargo-deny
        run: cargo install cargo-deny

      - name: Check licenses
        run: cargo deny check licenses

      - name: Check advisories
        run: cargo deny check advisories

  build:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest
    needs: [test, coverage, security]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Deploy to production
        run: |
          # Add deployment steps here
          echo "Deploying to production..."
          # Example: kubectl rollout restart deployment/web-server
```

### Release Workflow

```yaml
# .github/workflows/release.yml

name: Release

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  create-release:
    name: Create Release
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          draft: false
          prerelease: false

  build-binaries:
    name: Build Release Binaries
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            target: x86_64-unknown-linux-gnu
          - os: ubuntu-latest
            target: x86_64-unknown-linux-musl
          - os: macos-latest
            target: x86_64-apple-darwin
          - os: macos-latest
            target: aarch64-apple-darwin

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Build
        run: cargo build --release --target ${{ matrix.target }}

      - name: Package
        run: |
          cd target/${{ matrix.target }}/release
          tar czf web-server-${{ matrix.target }}.tar.gz web-server

      - name: Upload Release Asset
        uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ steps.create_release.outputs.upload_url }}
          asset_path: ./target/${{ matrix.target }}/release/web-server-${{ matrix.target }}.tar.gz
          asset_name: web-server-${{ matrix.target }}.tar.gz
          asset_content_type: application/gzip
```

---

## Monitoring and Observability

### Prometheus Metrics

```rust
// lib-web/src/metrics.rs

use prometheus::{
    register_histogram_vec, register_int_counter_vec, register_int_gauge,
    Encoder, HistogramVec, IntCounterVec, IntGauge, TextEncoder,
};
use once_cell::sync::Lazy;

// HTTP metrics
pub static HTTP_REQUESTS_TOTAL: Lazy<IntCounterVec> = Lazy::new(|| {
    register_int_counter_vec!(
        "http_requests_total",
        "Total number of HTTP requests",
        &["method", "endpoint", "status"]
    )
    .unwrap()
});

pub static HTTP_REQUEST_DURATION: Lazy<HistogramVec> = Lazy::new(|| {
    register_histogram_vec!(
        "http_request_duration_seconds",
        "HTTP request duration in seconds",
        &["method", "endpoint"],
        vec![0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
    )
    .unwrap()
});

// Database metrics
pub static DB_CONNECTIONS_ACTIVE: Lazy<IntGauge> = Lazy::new(|| {
    register_int_gauge!("db_connections_active", "Active database connections").unwrap()
});

pub static DB_QUERY_DURATION: Lazy<HistogramVec> = Lazy::new(|| {
    register_histogram_vec!(
        "db_query_duration_seconds",
        "Database query duration in seconds",
        &["query_type"],
        vec![0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5]
    )
    .unwrap()
});

// Metrics endpoint handler
pub async fn metrics_handler() -> Result<String, ApiError> {
    let encoder = TextEncoder::new();
    let metric_families = prometheus::gather();

    let mut buffer = Vec::new();
    encoder.encode(&metric_families, &mut buffer)?;

    Ok(String::from_utf8(buffer)?)
}
```

### Metrics Middleware

```rust
// lib-web/src/middleware/mw_metrics.rs

use axum::{
    middleware::Next,
    response::Response,
    extract::Request,
};
use std::time::Instant;

pub async fn metrics_middleware(
    request: Request,
    next: Next,
) -> Response {
    let method = request.method().to_string();
    let path = request.uri().path().to_string();

    let start = Instant::now();
    let response = next.run(request).await;
    let duration = start.elapsed();

    let status = response.status().as_u16().to_string();

    // Record metrics
    crate::metrics::HTTP_REQUESTS_TOTAL
        .with_label_values(&[&method, &path, &status])
        .inc();

    crate::metrics::HTTP_REQUEST_DURATION
        .with_label_values(&[&method, &path])
        .observe(duration.as_secs_f64());

    response
}
```

### Prometheus Configuration

```yaml
# prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'web-server'
    static_configs:
      - targets: ['web-server:8080']
    metrics_path: '/metrics'
    scrape_interval: 5s
```

### Grafana Dashboard

```json
{
  "dashboard": {
    "title": "Web Server Metrics",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])"
          }
        ]
      },
      {
        "title": "Request Duration (p95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Database Connections",
        "targets": [
          {
            "expr": "db_connections_active"
          }
        ]
      }
    ]
  }
}
```

---

## Logging with tracing-subscriber

### Setup

```rust
// web-server/src/main.rs

use tracing_subscriber::{
    fmt,
    layer::SubscriberExt,
    util::SubscriberInitExt,
    EnvFilter,
};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Initialize tracing
    tracing_subscriber::registry()
        .with(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info,web_server=debug,lib_core=debug"))
        )
        .with(
            fmt::layer()
                .with_target(true)
                .with_level(true)
                .with_thread_ids(true)
                .json()  // JSON output for production
        )
        .init();

    tracing::info!("Starting web server");

    // Application code...

    Ok(())
}
```

### Structured Logging

```rust
use tracing::{info, warn, error, debug, instrument};

#[instrument(skip(mm), fields(user_id = %user_id))]
pub async fn get_user(
    ctx: &Ctx,
    mm: &ModelManager,
    user_id: i64,
) -> Result<User> {
    debug!("Fetching user from database");

    let user = base::get::<Self, _>(ctx, mm, user_id).await?;

    info!(
        username = %user.username,
        email = %user.email,
        "User retrieved successfully"
    );

    Ok(user)
}

// Error logging
pub async fn create_user(
    ctx: &Ctx,
    mm: &ModelManager,
    data: UserForCreate,
) -> Result<i64> {
    match base::create::<Self, _>(ctx, mm, data.clone()).await {
        Ok(id) => {
            info!(user_id = %id, "User created");
            Ok(id)
        }
        Err(e) => {
            error!(
                error = %e,
                username = %data.username,
                "Failed to create user"
            );
            Err(e)
        }
    }
}
```

### Request Logging Middleware

```rust
// lib-web/src/middleware/mw_logging.rs

use axum::{
    middleware::Next,
    response::Response,
    extract::Request,
};
use tracing::info;
use std::time::Instant;

pub async fn logging_middleware(
    request: Request,
    next: Next,
) -> Response {
    let method = request.method().clone();
    let uri = request.uri().clone();
    let start = Instant::now();

    info!(
        method = %method,
        uri = %uri,
        "Request started"
    );

    let response = next.run(request).await;

    let duration = start.elapsed();
    let status = response.status();

    info!(
        method = %method,
        uri = %uri,
        status = %status,
        duration_ms = %duration.as_millis(),
        "Request completed"
    );

    response
}
```

---

## Health Checks and Graceful Shutdown

### Health Check Endpoint

```rust
// web-server/src/web/routes_health.rs

use axum::{
    routing::get,
    Router,
    Json,
};
use serde::Serialize;

#[derive(Serialize)]
pub struct HealthResponse {
    status: String,
    database: String,
    redis: Option<String>,
}

pub fn routes(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health_check))
        .route("/health/ready", get(readiness_check))
        .route("/health/live", get(liveness_check))
        .with_state(state)
}

async fn health_check(
    State(state): State<AppState>,
) -> Result<Json<HealthResponse>, ApiError> {
    // Check database
    let db_status = match check_database(&state.mm).await {
        Ok(_) => "healthy",
        Err(_) => "unhealthy",
    };

    // Check redis (optional)
    let redis_status = if let Some(ref redis) = state.redis {
        match check_redis(redis).await {
            Ok(_) => Some("healthy".to_string()),
            Err(_) => Some("unhealthy".to_string()),
        }
    } else {
        None
    };

    Ok(Json(HealthResponse {
        status: if db_status == "healthy" { "healthy" } else { "unhealthy" }.to_string(),
        database: db_status.to_string(),
        redis: redis_status,
    }))
}

async fn check_database(mm: &ModelManager) -> Result<()> {
    sqlx::query("SELECT 1")
        .execute(mm.dbx().pool())
        .await?;
    Ok(())
}

// Liveness probe - is the app running?
async fn liveness_check() -> &'static str {
    "OK"
}

// Readiness probe - is the app ready to serve traffic?
async fn readiness_check(
    State(state): State<AppState>,
) -> Result<&'static str, ApiError> {
    check_database(&state.mm).await?;
    Ok("OK")
}
```

### Graceful Shutdown

```rust
// web-server/src/main.rs

use tokio::signal;
use axum::Server;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Initialize...

    let addr = SocketAddr::from(([0, 0, 0, 0], 8080));
    tracing::info!("Listening on {}", addr);

    // Create server with graceful shutdown
    Server::bind(&addr)
        .serve(app.into_make_service())
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    tracing::info!("Server shutdown complete");

    Ok(())
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c()
            .await
            .expect("Failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("Failed to install SIGTERM handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {
            tracing::info!("Received Ctrl+C, shutting down");
        },
        _ = terminate => {
            tracing::info!("Received SIGTERM, shutting down");
        },
    }
}
```

---

## Database Migrations

### SQLx Migrations

```sql
-- migrations/001_initial_schema.sql

CREATE TABLE IF NOT EXISTS "user" (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    pwd VARCHAR(255) NOT NULL,
    token_salt UUID NOT NULL DEFAULT gen_random_uuid(),

    cid BIGINT NOT NULL DEFAULT 0,
    ctime TIMESTAMPTZ NOT NULL DEFAULT now(),
    mid BIGINT NOT NULL DEFAULT 0,
    mtime TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_user_username ON "user"(username);
CREATE INDEX idx_user_email ON "user"(email);
```

### Migration in Code

```rust
// lib-core/src/store/mod.rs

use sqlx::migrate::MigrateDatabase;

pub async fn run_migrations(db_url: &str) -> Result<()> {
    // Create database if it doesn't exist
    if !sqlx::Postgres::database_exists(db_url).await? {
        sqlx::Postgres::create_database(db_url).await?;
        tracing::info!("Database created");
    }

    // Connect to database
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(db_url)
        .await?;

    // Run migrations
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await?;

    tracing::info!("Migrations completed");

    Ok(())
}
```

---

## Environment Configuration

### Configuration Management

```rust
// lib-config/src/lib.rs

use serde::Deserialize;
use config::{Config, ConfigError, Environment, File};

#[derive(Debug, Deserialize)]
pub struct AppConfig {
    pub server: ServerConfig,
    pub database: DatabaseConfig,
    pub redis: Option<RedisConfig>,
    pub auth: AuthConfig,
}

#[derive(Debug, Deserialize)]
pub struct ServerConfig {
    pub host: String,
    pub port: u16,
}

#[derive(Debug, Deserialize)]
pub struct DatabaseConfig {
    pub url: String,
    pub max_connections: u32,
}

#[derive(Debug, Deserialize)]
pub struct RedisConfig {
    pub url: String,
}

#[derive(Debug, Deserialize)]
pub struct AuthConfig {
    pub token_duration: String,
}

impl AppConfig {
    pub fn load() -> Result<Self, ConfigError> {
        let env = std::env::var("RUN_ENV").unwrap_or_else(|_| "development".into());

        let config = Config::builder()
            // Start with default configuration
            .add_source(File::with_name("config/default"))
            // Layer on environment-specific configuration
            .add_source(File::with_name(&format!("config/{}", env)).required(false))
            // Layer on local configuration (gitignored)
            .add_source(File::with_name("config/local").required(false))
            // Override with environment variables (with prefix APP_)
            .add_source(Environment::with_prefix("APP").separator("__"))
            .build()?;

        config.try_deserialize()
    }
}
```

### Configuration Files

```yaml
# config/default.yml

server:
  host: "0.0.0.0"
  port: 8080

database:
  url: "postgresql://postgres:postgres@localhost:5432/myapp"
  max_connections: 20

auth:
  token_duration: "1h"
```

```yaml
# config/production.yml

server:
  host: "0.0.0.0"
  port: 8080

database:
  max_connections: 50

auth:
  token_duration: "24h"
```

### Environment Variables

```bash
# .env.example

# Server
APP_SERVER__HOST=0.0.0.0
APP_SERVER__PORT=8080

# Database
APP_DATABASE__URL=postgresql://postgres:postgres@localhost:5432/myapp
APP_DATABASE__MAX_CONNECTIONS=20

# Redis (optional)
APP_REDIS__URL=redis://localhost:6379

# Auth
APP_AUTH__TOKEN_DURATION=1h

# Logging
RUST_LOG=info,web_server=debug
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [architecture-overview.md](architecture-overview.md)
- [project-structure.md](project-structure.md)
- [security-patterns.md](security-patterns.md)
