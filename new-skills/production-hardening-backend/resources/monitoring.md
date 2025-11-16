# Monitoring and Observability - Rust Backend Production

**Comprehensive observability for production Rust services**

This resource covers structured logging, metrics collection, distributed tracing, health checks, and alerting for production-ready Rust backends.

---

## Table of Contents

- [Structured Logging](#structured-logging)
- [Metrics with Prometheus](#metrics-with-prometheus)
- [Distributed Tracing](#distributed-tracing)
- [Health Checks](#health-checks)
- [Audit Logging](#audit-logging)
- [Alerting Strategies](#alerting-strategies)

---

## Structured Logging

### Tracing Setup

```toml
[dependencies]
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
tracing-appender = "0.2"
```

```rust
use tracing::{info, warn, error, debug, instrument};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt, EnvFilter};

pub fn init_logging() {
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env().unwrap_or_else(|_| "info".into()))
        .with(tracing_subscriber::fmt::layer().json())
        .init();
}

#[instrument(skip(db))]
pub async fn get_user(db: &Database, user_id: &str) -> Result<User, Error> {
    info!(user_id = %user_id, "Fetching user");

    let user = db.query_user(user_id).await?;

    debug!(user_email = %user.email, "User retrieved");
    Ok(user)
}
```

### Correlation IDs

```rust
use tracing::Span;
use uuid::Uuid;

pub async fn handle_request(req: Request) -> Response {
    let correlation_id = req
        .headers()
        .get("x-correlation-id")
        .and_then(|v| v.to_str().ok())
        .unwrap_or_else(|| &Uuid::new_v4().to_string());

    let span = tracing::info_span!(
        "http_request",
        correlation_id = %correlation_id,
        method = %req.method(),
        path = %req.uri().path()
    );

    let _guard = span.enter();

    info!("Processing request");
    // All logs within this scope will include correlation_id
    process_request(req).await
}
```

### Log Levels by Environment

```rust
use tracing_subscriber::EnvFilter;

pub fn configure_log_level() -> EnvFilter {
    let default_level = if cfg!(debug_assertions) {
        "debug"
    } else {
        "info"
    };

    EnvFilter::try_from_default_env()
        .unwrap_or_else(|_| {
            EnvFilter::new(default_level)
                .add_directive("hyper=info".parse().unwrap())
                .add_directive("sqlx=warn".parse().unwrap())
        })
}
```

---

## Metrics with Prometheus

### Setup Prometheus Metrics

```toml
[dependencies]
prometheus = "0.13"
lazy_static = "1.4"
```

```rust
use prometheus::{
    register_counter_vec, register_histogram_vec, register_gauge,
    CounterVec, HistogramVec, Gauge, Encoder, TextEncoder,
};
use lazy_static::lazy_static;

lazy_static! {
    // HTTP request metrics
    pub static ref HTTP_REQUESTS: CounterVec = register_counter_vec!(
        "http_requests_total",
        "Total HTTP requests",
        &["method", "path", "status"]
    ).unwrap();

    pub static ref HTTP_DURATION: HistogramVec = register_histogram_vec!(
        "http_request_duration_seconds",
        "HTTP request duration in seconds",
        &["method", "path"],
        vec![0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
    ).unwrap();

    // Database metrics
    pub static ref DB_CONNECTIONS: Gauge = register_gauge!(
        "db_connections_active",
        "Active database connections"
    ).unwrap();

    pub static ref DB_QUERY_DURATION: HistogramVec = register_histogram_vec!(
        "db_query_duration_seconds",
        "Database query duration",
        &["query_type"]
    ).unwrap();
}

// Metrics middleware
pub async fn record_metrics<F, Fut>(
    method: &str,
    path: &str,
    handler: F,
) -> Response
where
    F: FnOnce() -> Fut,
    Fut: Future<Output = Response>,
{
    let timer = HTTP_DURATION
        .with_label_values(&[method, path])
        .start_timer();

    let response = handler().await;

    timer.observe_duration();
    HTTP_REQUESTS
        .with_label_values(&[method, path, &response.status().as_str()])
        .inc();

    response
}
```

### Expose Metrics Endpoint

```rust
use axum::{Router, routing::get};
use prometheus::{Encoder, TextEncoder};

pub fn metrics_routes() -> Router {
    Router::new().route("/metrics", get(metrics_handler))
}

async fn metrics_handler() -> Result<String, StatusCode> {
    let encoder = TextEncoder::new();
    let metric_families = prometheus::gather();

    let mut buffer = Vec::new();
    encoder.encode(&metric_families, &mut buffer)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;

    String::from_utf8(buffer)
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)
}
```

### Custom Business Metrics

```rust
use prometheus::{IntCounter, register_int_counter};

lazy_static! {
    pub static ref ORDERS_CREATED: IntCounter = register_int_counter!(
        "orders_created_total",
        "Total orders created"
    ).unwrap();

    pub static ref PAYMENT_FAILURES: CounterVec = register_counter_vec!(
        "payment_failures_total",
        "Payment failures by reason",
        &["reason"]
    ).unwrap();
}

pub async fn create_order(order: Order) -> Result<OrderId, Error> {
    let order_id = save_order(order).await?;
    ORDERS_CREATED.inc();
    Ok(order_id)
}
```

---

## Distributed Tracing

### OpenTelemetry Setup

```toml
[dependencies]
opentelemetry = { version = "0.21", features = ["rt-tokio"] }
opentelemetry-jaeger = { version = "0.20", features = ["rt-tokio"] }
tracing-opentelemetry = "0.22"
```

```rust
use opentelemetry::global;
use opentelemetry_jaeger::new_agent_pipeline;
use tracing_subscriber::layer::SubscriberExt;

pub fn init_tracing() -> Result<(), Box<dyn std::error::Error>> {
    global::set_text_map_propagator(opentelemetry_jaeger::Propagator::new());

    let tracer = new_agent_pipeline()
        .with_service_name("my-rust-service")
        .with_endpoint("localhost:6831")
        .install_simple()?;

    let telemetry = tracing_opentelemetry::layer().with_tracer(tracer);

    tracing_subscriber::registry()
        .with(telemetry)
        .with(tracing_subscriber::fmt::layer())
        .try_init()?;

    Ok(())
}
```

### Trace Context Propagation

```rust
use opentelemetry::global;
use opentelemetry::propagation::Extractor;
use std::collections::HashMap;

pub async fn handle_request_with_trace(req: Request) -> Response {
    // Extract trace context from headers
    let headers: HashMap<String, String> = req
        .headers()
        .iter()
        .map(|(k, v)| (k.to_string(), v.to_str().unwrap_or("").to_string()))
        .collect();

    let parent_cx = global::get_text_map_propagator(|propagator| {
        propagator.extract(&headers)
    });

    // Create span with parent context
    let span = tracing::info_span!("http_request", otel.kind = "server");
    span.set_parent(parent_cx);

    let _guard = span.enter();
    process_request(req).await
}
```

### Annotate Spans

```rust
use tracing::{info_span, Span};

#[instrument]
pub async fn complex_operation(user_id: &str) -> Result<(), Error> {
    let span = Span::current();

    // Add custom attributes
    span.record("user.id", user_id);

    let data = fetch_data(user_id).await?;
    span.record("data.size", data.len());

    process_data(data).await?;
    span.record("processing.complete", true);

    Ok(())
}
```

---

## Health Checks

### Liveness and Readiness Probes

```rust
use axum::{Router, routing::get, Json};
use serde_json::json;

pub fn health_routes() -> Router {
    Router::new()
        .route("/healthz", get(liveness))
        .route("/readyz", get(readiness))
}

// Liveness: Is the service running?
async fn liveness() -> Json<serde_json::Value> {
    Json(json!({
        "status": "ok",
        "timestamp": chrono::Utc::now().to_rfc3339()
    }))
}

// Readiness: Can the service handle requests?
async fn readiness(db: Extension<Database>) -> Result<Json<Value>, StatusCode> {
    let mut checks = HashMap::new();

    // Check database
    match db.ping().await {
        Ok(_) => checks.insert("database", "healthy"),
        Err(_) => checks.insert("database", "unhealthy"),
    };

    // Check external dependencies
    match check_external_api().await {
        Ok(_) => checks.insert("external_api", "healthy"),
        Err(_) => checks.insert("external_api", "degraded"),
    };

    let all_healthy = checks.values().all(|&v| v == "healthy");

    if all_healthy {
        Ok(Json(json!({
            "status": "ready",
            "checks": checks
        })))
    } else {
        Err(StatusCode::SERVICE_UNAVAILABLE)
    }
}
```

### Detailed Health Check

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize)]
pub struct HealthCheck {
    status: String,
    version: String,
    uptime_seconds: u64,
    checks: HashMap<String, ComponentHealth>,
}

#[derive(Serialize)]
pub struct ComponentHealth {
    status: String,
    message: Option<String>,
    latency_ms: Option<u64>,
}

pub async fn detailed_health(
    app_state: Extension<AppState>,
) -> Json<HealthCheck> {
    let mut checks = HashMap::new();

    // Database check with timing
    let db_start = Instant::now();
    let db_status = match app_state.db.ping().await {
        Ok(_) => ComponentHealth {
            status: "healthy".to_string(),
            message: None,
            latency_ms: Some(db_start.elapsed().as_millis() as u64),
        },
        Err(e) => ComponentHealth {
            status: "unhealthy".to_string(),
            message: Some(e.to_string()),
            latency_ms: None,
        },
    };
    checks.insert("database".to_string(), db_status);

    Json(HealthCheck {
        status: "ok".to_string(),
        version: env!("CARGO_PKG_VERSION").to_string(),
        uptime_seconds: app_state.start_time.elapsed().as_secs(),
        checks,
    })
}
```

---

## Audit Logging

### Security Event Logging

```rust
use serde::Serialize;
use tracing::{info, warn};

#[derive(Serialize)]
pub struct AuditLog {
    timestamp: String,
    event_type: String,
    user_id: Option<String>,
    resource: String,
    action: String,
    result: String,
    ip_address: Option<String>,
    metadata: serde_json::Value,
}

pub fn log_authentication_success(user_id: &str, ip: &str) {
    let audit = AuditLog {
        timestamp: chrono::Utc::now().to_rfc3339(),
        event_type: "authentication".to_string(),
        user_id: Some(user_id.to_string()),
        resource: "auth".to_string(),
        action: "login".to_string(),
        result: "success".to_string(),
        ip_address: Some(ip.to_string()),
        metadata: json!({}),
    };

    info!(
        audit_log = serde_json::to_string(&audit).unwrap(),
        "Authentication successful"
    );
}

pub fn log_authorization_failure(user_id: &str, resource: &str, action: &str) {
    let audit = AuditLog {
        timestamp: chrono::Utc::now().to_rfc3339(),
        event_type: "authorization".to_string(),
        user_id: Some(user_id.to_string()),
        resource: resource.to_string(),
        action: action.to_string(),
        result: "denied".to_string(),
        ip_address: None,
        metadata: json!({ "reason": "insufficient_permissions" }),
    };

    warn!(
        audit_log = serde_json::to_string(&audit).unwrap(),
        "Authorization denied"
    );
}
```

### Data Access Logging

```rust
pub async fn log_data_access(
    user_id: &str,
    table: &str,
    operation: &str,
    record_ids: &[String],
) {
    let audit = AuditLog {
        timestamp: chrono::Utc::now().to_rfc3339(),
        event_type: "data_access".to_string(),
        user_id: Some(user_id.to_string()),
        resource: table.to_string(),
        action: operation.to_string(),
        result: "success".to_string(),
        ip_address: None,
        metadata: json!({
            "record_count": record_ids.len(),
            "record_ids": record_ids,
        }),
    };

    info!(
        audit_log = serde_json::to_string(&audit).unwrap(),
        "Data access logged"
    );
}
```

---

## Alerting Strategies

### Define Alert Rules

```yaml
# prometheus-alerts.yml
groups:
  - name: backend_alerts
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m]) /
          rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # Slow response times
      - alert: SlowResponses
        expr: |
          histogram_quantile(0.95,
            rate(http_request_duration_seconds_bucket[5m])
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "95th percentile latency above 1s"

      # Database connection pool exhaustion
      - alert: DatabasePoolExhausted
        expr: db_connections_active / db_connections_max > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connection pool near capacity"
```

### Alert Integration

```rust
use reqwest::Client;

pub struct AlertManager {
    client: Client,
    webhook_url: String,
}

impl AlertManager {
    pub async fn send_alert(&self, alert: Alert) -> Result<(), Error> {
        self.client
            .post(&self.webhook_url)
            .json(&alert)
            .send()
            .await?;
        Ok(())
    }
}

#[derive(Serialize)]
pub struct Alert {
    severity: String,
    title: String,
    description: String,
    timestamp: String,
    metadata: HashMap<String, String>,
}

pub async fn trigger_circuit_breaker_alert(service: &str) {
    let alert = Alert {
        severity: "warning".to_string(),
        title: format!("Circuit breaker opened: {}", service),
        description: "Circuit breaker has opened due to high failure rate".to_string(),
        timestamp: chrono::Utc::now().to_rfc3339(),
        metadata: HashMap::from([
            ("service".to_string(), service.to_string()),
            ("action".to_string(), "investigate".to_string()),
        ]),
    };

    // Send to alerting system
    if let Err(e) = ALERT_MANAGER.send_alert(alert).await {
        error!("Failed to send alert: {}", e);
    }
}
```

---

## Observability Checklist

### Pre-Deployment
- [ ] Structured logging configured with JSON format
- [ ] Correlation IDs propagated across services
- [ ] Prometheus metrics exposed on `/metrics`
- [ ] Distributed tracing configured (Jaeger/Zipkin)
- [ ] Health check endpoints implemented
- [ ] Audit logging for security events

### Production Monitoring
- [ ] Log aggregation configured (ELK, Loki)
- [ ] Metrics dashboard created (Grafana)
- [ ] Alert rules defined and tested
- [ ] On-call runbooks documented
- [ ] SLIs and SLOs defined
- [ ] Regular log review process

---

**Related Resources:**
- [Fault Tolerance and Resilience](./fault-tolerance.md)
- [Performance and Scalability](./performance.md)
- [Deployment and Operations](./deployment.md)
- [Troubleshooting](./troubleshooting.md)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Compliance**: NIST SP 800-53 AU (Audit and Accountability), SI-4 (System Monitoring)
