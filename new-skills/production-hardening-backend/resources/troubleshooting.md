# Troubleshooting - Rust Backend Production Issues

**Debugging and resolving common production problems in Rust services**

This resource provides diagnostic procedures and solutions for cargo audit failures, performance issues, deployment problems, and runtime errors in production Rust backends.

---

## Table of Contents

- [Dependency and Security Issues](#dependency-and-security-issues)
- [Performance Problems](#performance-problems)
- [Database Connection Issues](#database-connection-issues)
- [Container and Deployment Issues](#container-and-deployment-issues)
- [Runtime Errors](#runtime-errors)
- [Monitoring and Observability Issues](#monitoring-and-observability-issues)

---

## Dependency and Security Issues

### cargo audit Failures

**Symptom**: `cargo audit` reports vulnerabilities

**Diagnosis**:
```bash
# Check which crates have vulnerabilities
cargo audit

# Get detailed JSON output
cargo audit --json

# Check specific advisory
cargo audit --ignore RUSTSEC-2023-0001
```

**Solutions**:

**Issue 1: Outdated Dependencies**
```bash
# Update dependencies
cargo update

# Check for breaking updates
cargo outdated

# Update specific crate
cargo update -p <crate-name>
```

**Issue 2: Vulnerable Dependency with No Fix**
```toml
# .cargo/deny.toml - Temporarily ignore if no fix available
[advisories]
ignore = [
    "RUSTSEC-2023-0001",  # Document why ignored
]
```

**Issue 3: Transitive Dependency Vulnerability**
```bash
# Find which crate depends on vulnerable crate
cargo tree -i <vulnerable-crate>

# Update parent dependency or find alternative
```

---

## Performance Problems

### High Memory Usage

**Symptom**: Container OOMKilled or high memory alerts

**Diagnosis**:
```rust
// Add memory profiling
use sysinfo::{System, SystemExt};

pub fn log_memory_usage() {
    let mut sys = System::new_all();
    sys.refresh_memory();

    tracing::info!(
        used_mb = sys.used_memory() / 1024 / 1024,
        total_mb = sys.total_memory() / 1024 / 1024,
        "Memory usage"
    );
}
```

**Solutions**:

**Issue 1: Memory Leak in Connection Pool**
```rust
// ❌ Problem: Not closing connections
let conn = pool.acquire().await?;
// ... use connection
// Forgot to drop!

// ✅ Solution: Explicit scope
{
    let conn = pool.acquire().await?;
    // Use connection
}  // Connection dropped here
```

**Issue 2: Unbounded Cache Growth**
```rust
// ❌ Problem: No size limit
let cache: HashMap<String, Vec<u8>> = HashMap::new();

// ✅ Solution: Use LRU cache with size limit
use lru::LruCache;
let mut cache = LruCache::new(NonZeroUsize::new(1000).unwrap());
```

### Slow Response Times

**Symptom**: P95/P99 latency spikes

**Diagnosis**:
```bash
# Generate flamegraph
cargo flamegraph --bin my-service

# Profile with perf
perf record -g ./target/release/my-service
perf report
```

**Solutions**:

**Issue 1: Blocking Database Queries**
```rust
// ❌ Problem: Sequential queries
let user = get_user(id).await?;
let posts = get_posts(user.id).await?;

// ✅ Solution: Parallel queries
let (user, posts) = tokio::join!(
    get_user(id),
    get_posts_for_user(id)
);
```

**Issue 2: No Query Timeout**
```rust
use tokio::time::{timeout, Duration};

// Add timeout to prevent hanging
match timeout(Duration::from_secs(5), slow_query()).await {
    Ok(Ok(result)) => result,
    Ok(Err(e)) => return Err(e),
    Err(_) => return Err(Error::Timeout),
}
```

---

## Database Connection Issues

### "Too Many Connections" Error

**Symptom**: `FATAL: sorry, too many clients already`

**Diagnosis**:
```sql
-- Check current connections
SELECT count(*) FROM pg_stat_activity;

-- Check connection limit
SHOW max_connections;

-- See who's connected
SELECT usename, application_name, client_addr, state, query
FROM pg_stat_activity
WHERE state != 'idle';
```

**Solutions**:

**Issue 1: Connection Pool Too Large**
```rust
// ❌ Problem: Pool size exceeds database limit
let pool = PgPoolOptions::new()
    .max_connections(100)  // Database only allows 50!
    .connect(url).await?;

// ✅ Solution: Match database capacity
let pool = PgPoolOptions::new()
    .max_connections(20)  // Leave room for other services
    .connect(url).await?;
```

**Issue 2: Connection Leaks**
```rust
// Add connection pool metrics
use prometheus::Gauge;

lazy_static! {
    static ref DB_CONNECTIONS: Gauge = register_gauge!(
        "db_connections_active",
        "Active database connections"
    ).unwrap();
}

// Monitor in background task
tokio::spawn(async move {
    loop {
        DB_CONNECTIONS.set(pool.size() as f64);
        tokio::time::sleep(Duration::from_secs(10)).await;
    }
});
```

### Slow Query Performance

**Symptom**: Database queries taking seconds instead of milliseconds

**Diagnosis**:
```sql
-- Enable slow query logging (PostgreSQL)
ALTER DATABASE mydb SET log_min_duration_statement = 1000;  -- Log queries > 1s

-- Find slow queries
SELECT query, calls, mean_exec_time, max_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

**Solutions**:

**Issue 1: Missing Index**
```sql
-- Explain query plan
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';

-- Add index if missing
CREATE INDEX idx_users_email ON users(email);
```

**Issue 2: N+1 Query Problem**
```rust
// ❌ Problem: N+1 queries
for user in users {
    let posts = get_posts(user.id).await?;  // N queries!
}

// ✅ Solution: Single query with JOIN or batch fetch
let user_ids: Vec<_> = users.iter().map(|u| u.id).collect();
let all_posts = get_posts_batch(&user_ids).await?;
```

---

## Container and Deployment Issues

### Container Image Build Failures

**Symptom**: Docker build fails with compilation errors

**Diagnosis**:
```bash
# Build with verbose output
docker build --progress=plain .

# Check build context size
du -sh .
```

**Solutions**:

**Issue 1: Build Context Too Large**
```dockerfile
# .dockerignore
target/
.git/
*.log
node_modules/
```

**Issue 2: Missing System Dependencies**
```dockerfile
# Install required system packages
FROM rust:1.75-slim

RUN apt-get update && apt-get install -y \
    libpq-dev \
    libssl-dev \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*
```

### Kubernetes Pod CrashLoopBackOff

**Symptom**: Pod keeps restarting

**Diagnosis**:
```bash
# Check pod status
kubectl get pods

# View pod logs
kubectl logs <pod-name>

# Previous container logs
kubectl logs <pod-name> --previous

# Describe pod for events
kubectl describe pod <pod-name>
```

**Solutions**:

**Issue 1: Failed Health Check**
```rust
// Ensure /healthz endpoint responds quickly
pub async fn health_check() -> impl IntoResponse {
    // Don't do expensive checks in liveness probe
    Json(json!({"status": "ok"}))
}
```

**Issue 2: Missing Environment Variables**
```yaml
# Check secrets are mounted
env:
- name: DATABASE_URL
  valueFrom:
    secretKeyRef:
      name: app-secrets
      key: database-url
      optional: false  # Fail if missing
```

---

## Runtime Errors

### Panics in Production

**Symptom**: Service crashes with panic message

**Diagnosis**:
```bash
# Check logs for panic
kubectl logs <pod> | grep -i panic

# Check if panic handler is set
```

**Solutions**:

**Issue 1: `.unwrap()` in Production Code**
```rust
// ❌ Problem: Panic on None
let value = map.get(key).unwrap();

// ✅ Solution: Proper error handling
let value = map.get(key)
    .ok_or(Error::KeyNotFound)?;
```

**Issue 2: Division by Zero**
```rust
// ❌ Problem: Unchecked division
let result = total / count;

// ✅ Solution: Check for zero
let result = if count == 0 {
    0
} else {
    total / count
};
```

### Deadlocks

**Symptom**: Service hangs, requests timeout

**Diagnosis**:
```rust
// Add timeout to lock acquisition
use parking_lot::Mutex;

let mutex = Mutex::new(data);
if let Some(guard) = mutex.try_lock_for(Duration::from_secs(5)) {
    // Use guard
} else {
    error!("Failed to acquire lock - possible deadlock");
}
```

**Solutions**:

**Issue 1: Inconsistent Lock Ordering**
```rust
// ✅ Always acquire locks in same order
let (lock_a, lock_b) = if id_a < id_b {
    (mutex_a.lock(), mutex_b.lock())
} else {
    (mutex_b.lock(), mutex_a.lock())
};
```

---

## Monitoring and Observability Issues

### Missing Metrics

**Symptom**: Prometheus /metrics endpoint returns empty or missing metrics

**Diagnosis**:
```bash
# Check if metrics endpoint is accessible
curl http://localhost:8080/metrics

# Verify metrics are registered
```

**Solutions**:

**Issue 1: Metrics Not Initialized**
```rust
// Ensure lazy_static metrics are referenced at startup
pub fn init_metrics() {
    // Force initialization
    let _ = &*HTTP_REQUESTS;
    let _ = &*DB_CONNECTIONS;
}

#[tokio::main]
async fn main() {
    init_metrics();  // Call before starting server
    start_server().await;
}
```

### Distributed Tracing Not Working

**Symptom**: Traces not appearing in Jaeger/Zipkin

**Diagnosis**:
```bash
# Check if tracer is sending data
curl http://localhost:14268/api/traces

# Verify environment variables
echo $JAEGER_AGENT_HOST
```

**Solutions**:

**Issue 1: Trace Context Not Propagated**
```rust
// Extract and inject trace context in HTTP middleware
use opentelemetry::global;

pub async fn propagate_trace(req: Request, next: Next) -> Response {
    let parent_cx = global::get_text_map_propagator(|propagator| {
        propagator.extract(&HeaderExtractor(req.headers()))
    });

    let span = tracing::info_span!("http_request");
    span.set_parent(parent_cx);

    let _guard = span.enter();
    next.run(req).await
}
```

---

## Debugging Checklist

### When Service is Down
1. Check recent deployments (kubectl rollout history)
2. Review error logs (kubectl logs)
3. Check resource limits (kubectl top pods)
4. Verify health checks passing
5. Check database connectivity
6. Review recent config changes

### When Service is Slow
1. Check Prometheus metrics (latency, error rate)
2. Review distributed traces (find slow spans)
3. Check database query performance
4. Verify connection pool not exhausted
5. Look for memory pressure
6. Check for circuit breaker activations

### When Errors Spike
1. Check error logs for patterns
2. Review recent code changes
3. Check external dependencies (APIs, DB)
4. Verify rate limiting not triggered
5. Check for resource exhaustion
6. Review alerting rules

---

**Related Resources:**
- [Monitoring and Observability](./monitoring.md)
- [Fault Tolerance and Resilience](./fault-tolerance.md)
- [Deployment and Operations](./deployment.md)
- [Performance and Scalability](./performance.md)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Purpose**: Production debugging and issue resolution
