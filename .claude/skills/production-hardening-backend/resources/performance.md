# Performance and Scalability - Rust Backend Production

**Optimizing Rust backend services for high performance and scalability**

This resource covers connection pooling, caching strategies, async optimization, load testing, and database query optimization for production Rust services.

---

## Table of Contents

- [Connection Pooling](#connection-pooling)
- [Caching Strategies](#caching-strategies)
- [Async Optimization](#async-optimization)
- [Resource Limits](#resource-limits)
- [Load Testing](#load-testing)
- [Database Optimization](#database-optimization)

---

## Connection Pooling

### Database Connection Pool (SQLx)

```toml
[dependencies]
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres"] }
```

```rust
use sqlx::postgres::{PgPool, PgPoolOptions};
use std::time::Duration;

pub async fn create_pool(database_url: &str) -> Result<PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(20)  // Max pool size
        .min_connections(5)   // Min idle connections
        .acquire_timeout(Duration::from_secs(3))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .connect(database_url)
        .await
}
```

### HTTP Connection Pool (reqwest)

```rust
use reqwest::Client;
use std::time::Duration;

pub fn create_http_client() -> Client {
    Client::builder()
        .pool_max_idle_per_host(10)
        .pool_idle_timeout(Duration::from_secs(90))
        .timeout(Duration::from_secs(10))
        .build()
        .expect("Failed to build HTTP client")
}
```

---

## Caching Strategies

### In-Memory Cache (moka)

```toml
[dependencies]
moka = { version = "0.12", features = ["future"] }
```

```rust
use moka::future::Cache;
use std::time::Duration;

pub struct CacheLayer {
    cache: Cache<String, String>,
}

impl CacheLayer {
    pub fn new() -> Self {
        let cache = Cache::builder()
            .max_capacity(10_000)
            .time_to_live(Duration::from_secs(300))
            .time_to_idle(Duration::from_secs(60))
            .build();

        Self { cache }
    }

    pub async fn get_or_fetch<F, Fut>(
        &self,
        key: String,
        fetch: F,
    ) -> Result<String, Error>
    where
        F: FnOnce() -> Fut,
        Fut: Future<Output = Result<String, Error>>,
    {
        self.cache
            .try_get_with(key, async move { fetch().await })
            .await
            .map_err(|e| Error::CacheFetch(e.to_string()))
    }
}
```

### Redis Cache

```toml
[dependencies]
redis = { version = "0.24", features = ["tokio-comp", "connection-manager"] }
```

```rust
use redis::AsyncCommands;
use redis::aio::ConnectionManager;

pub struct RedisCache {
    conn: ConnectionManager,
}

impl RedisCache {
    pub async fn get_or_set<F, Fut>(
        &mut self,
        key: &str,
        ttl: usize,
        fetch: F,
    ) -> Result<String, Error>
    where
        F: FnOnce() -> Fut,
        Fut: Future<Output = Result<String, Error>>,
    {
        // Try cache first
        if let Ok(Some(cached)) = self.conn.get::<_, Option<String>>(key).await {
            return Ok(cached);
        }

        // Fetch fresh data
        let data = fetch().await?;

        // Set in cache
        let _: () = self.conn.set_ex(key, &data, ttl).await?;

        Ok(data)
    }
}
```

---

## Async Optimization

### Tokio Runtime Tuning

```rust
use tokio::runtime::{Builder, Runtime};

pub fn create_runtime() -> Runtime {
    Builder::new_multi_thread()
        .worker_threads(num_cpus::get())
        .thread_name("tokio-worker")
        .thread_stack_size(3 * 1024 * 1024)  // 3MB stack
        .enable_all()
        .build()
        .expect("Failed to create Tokio runtime")
}
```

### Parallel Processing

```rust
use futures::stream::{self, StreamExt};

pub async fn process_batch(items: Vec<Item>) -> Vec<Result<Output, Error>> {
    stream::iter(items)
        .map(|item| async move { process_item(item).await })
        .buffer_unordered(10)  // Process 10 concurrently
        .collect()
        .await
}
```

### Avoid Blocking

```rust
// ❌ BAD: Blocking in async context
pub async fn bad_example() {
    std::thread::sleep(Duration::from_secs(1));  // Blocks entire thread!
}

// ✅ GOOD: Use tokio::time::sleep
pub async fn good_example() {
    tokio::time::sleep(Duration::from_secs(1)).await;  // Non-blocking
}

// ✅ GOOD: Spawn blocking work
pub async fn compute_heavy_work() -> u64 {
    tokio::task::spawn_blocking(|| {
        // CPU-intensive work here
        expensive_computation()
    })
    .await
    .expect("Blocking task failed")
}
```

---

## Resource Limits

### Backpressure with Semaphores

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

pub struct RateLimiter {
    semaphore: Arc<Semaphore>,
}

impl RateLimiter {
    pub fn new(max_concurrent: usize) -> Self {
        Self {
            semaphore: Arc::new(Semaphore::new(max_concurrent)),
        }
    }

    pub async fn execute<F, Fut, T>(&self, f: F) -> Result<T, Error>
    where
        F: FnOnce() -> Fut,
        Fut: Future<Output = Result<T, Error>>,
    {
        let _permit = self.semaphore.acquire().await?;
        f().await
    }
}
```

### Memory Limits

```rust
// Monitor memory usage
use sysinfo::{System, SystemExt};

pub fn check_memory_pressure() -> bool {
    let mut sys = System::new_all();
    sys.refresh_memory();

    let used_percent = (sys.used_memory() as f64 / sys.total_memory() as f64) * 100.0;

    used_percent > 90.0  // Alert if > 90% memory used
}
```

---

## Load Testing

### k6 Load Test

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },  // Ramp up to 100 users
    { duration: '5m', target: 100 },  // Stay at 100 users
    { duration: '2m', target: 200 },  // Ramp up to 200
    { duration: '5m', target: 200 },  // Stay at 200
    { duration: '2m', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95% requests < 500ms
    http_req_failed: ['rate<0.01'],    // Error rate < 1%
  },
};

export default function () {
  let res = http.get('http://localhost:8080/api/users');

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

### Run Load Test

```bash
k6 run --vus 100 --duration 30s load-test.js
```

---

## Database Optimization

### Query Optimization

```rust
// ❌ BAD: N+1 query problem
pub async fn get_users_with_posts_bad(pool: &PgPool) -> Result<Vec<User>, Error> {
    let users = sqlx::query_as::<_, User>("SELECT * FROM users")
        .fetch_all(pool)
        .await?;

    for user in &mut users {
        user.posts = sqlx::query_as::<_, Post>(
            "SELECT * FROM posts WHERE user_id = $1"
        )
        .bind(user.id)
        .fetch_all(pool)
        .await?;
    }

    Ok(users)
}

// ✅ GOOD: Single query with JOIN
pub async fn get_users_with_posts_good(pool: &PgPool) -> Result<Vec<User>, Error> {
    sqlx::query_as::<_, UserWithPosts>(
        "SELECT u.*, p.* FROM users u LEFT JOIN posts p ON u.id = p.user_id"
    )
    .fetch_all(pool)
    .await
}
```

### Indexing

```sql
-- Create indexes for frequently queried columns
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);

-- Composite index for complex queries
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);
```

### Connection Pooling Best Practices

```rust
// Configure pool based on workload
pub fn calculate_pool_size() -> u32 {
    let cpu_count = num_cpus::get() as u32;

    // Formula: connections = (core_count * 2) + effective_spindle_count
    // For cloud databases with SSDs, spindle count ≈ 0
    cpu_count * 2
}
```

---

## Performance Checklist

### Development
- [ ] Use connection pooling for database and HTTP
- [ ] Implement caching for frequently accessed data
- [ ] Avoid blocking operations in async code
- [ ] Use `spawn_blocking` for CPU-intensive work
- [ ] Add database indexes for common queries

### Pre-Production
- [ ] Run load tests to find bottlenecks
- [ ] Profile application with flamegraph
- [ ] Monitor memory usage under load
- [ ] Test with production-like data volumes
- [ ] Validate cache hit rates

### Production
- [ ] Monitor query performance (p95, p99)
- [ ] Track connection pool saturation
- [ ] Alert on high memory usage (>85%)
- [ ] Review slow query logs weekly
- [ ] Capacity plan for growth

---

**Related Resources:**
- [Fault Tolerance and Resilience](./fault-tolerance.md)
- [Monitoring and Observability](./monitoring.md)
- [Deployment and Operations](./deployment.md)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Compliance**: Efficient resource utilization, scalability best practices
