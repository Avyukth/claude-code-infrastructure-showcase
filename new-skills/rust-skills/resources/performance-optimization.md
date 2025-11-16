# Performance Optimization - Profiling and Optimization Guide

Complete guide to profiling, benchmarking, and optimizing Rust backend services for maximum performance.

## Table of Contents

- [Profiling Tools](#profiling-tools)
- [Benchmarking with Criterion](#benchmarking-with-criterion)
- [Memory Optimization](#memory-optimization)
- [Zero-Copy Patterns](#zero-copy-patterns)
- [Database Query Optimization](#database-query-optimization)
- [Caching Strategies](#caching-strategies)
- [Compilation Optimization](#compilation-optimization)

---

## Profiling Tools

### Flamegraph for CPU Profiling

```bash
# Install cargo-flamegraph
cargo install flamegraph

# Generate flamegraph (requires perf on Linux)
sudo cargo flamegraph --bin web-server

# Open flamegraph.svg in browser
firefox flamegraph.svg
```

### Using perf (Linux)

```bash
# Record performance data
perf record --call-graph=dwarf ./target/release/web-server

# Generate report
perf report

# Generate flamegraph from perf data
perf script | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg
```

### macOS Instruments

```bash
# Build with symbols
cargo build --release

# Run with Instruments
instruments -t "Time Profiler" ./target/release/web-server
```

### Memory Profiling with valgrind

```bash
# Install valgrind
sudo apt install valgrind  # Linux

# Profile memory usage
valgrind --tool=massif ./target/release/web-server

# Visualize with massif-visualizer
ms_print massif.out.*
```

### Heap Profiling with heaptrack

```bash
# Install heaptrack
sudo apt install heaptrack

# Profile heap allocations
heaptrack ./target/release/web-server

# Analyze results
heaptrack_gui heaptrack.web-server.*.gz
```

---

## Benchmarking with Criterion

### Setup

```toml
# Cargo.toml

[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "user_benchmarks"
harness = false
```

### Basic Benchmark

```rust
// benches/user_benchmarks.rs

use criterion::{black_box, criterion_group, criterion_main, Criterion};
use lib_core::model::{UserBmc, UserForCreate, ModelManager};

fn benchmark_user_creation(c: &mut Criterion) {
    let runtime = tokio::runtime::Runtime::new().unwrap();
    let mm = runtime.block_on(ModelManager::new()).unwrap();

    c.bench_function("create_user", |b| {
        b.to_async(&runtime).iter(|| async {
            let data = UserForCreate {
                username: format!("bench_{}", uuid::Uuid::new_v4()),
                email: format!("bench_{}@example.com", uuid::Uuid::new_v4()),
                pwd_clear: "BenchPass123!".to_string(),
            };

            let user_id = UserBmc::create(&ctx, &mm, data).await.unwrap();

            // Clean up
            UserBmc::delete(&ctx, &mm, user_id).await.unwrap();

            black_box(user_id)
        });
    });
}

criterion_group!(benches, benchmark_user_creation);
criterion_main!(benches);
```

### Parameterized Benchmarks

```rust
fn benchmark_list_users_pagination(c: &mut Criterion) {
    let runtime = tokio::runtime::Runtime::new().unwrap();
    let mm = runtime.block_on(ModelManager::new()).unwrap();

    let mut group = c.benchmark_group("list_users");

    for size in [10, 100, 1000, 10000].iter() {
        group.bench_with_input(
            BenchmarkId::from_parameter(size),
            size,
            |b, &size| {
                b.to_async(&runtime).iter(|| async {
                    let users = UserBmc::list(&ctx, &mm, Some(size), None)
                        .await
                        .unwrap();
                    black_box(users)
                });
            },
        );
    }

    group.finish();
}
```

### Running Benchmarks

```bash
# Run all benchmarks
cargo bench

# Run specific benchmark
cargo bench user_creation

# View HTML reports
open target/criterion/report/index.html
```

---

## Memory Optimization

### Avoid Unnecessary Clones

```rust
// ❌ BAD: Cloning large data
pub async fn process_users(users: Vec<User>) -> Result<()> {
    for user in users.clone() {  // Unnecessary clone!
        process_user(user).await?;
    }
    Ok(())
}

// ✅ GOOD: Use references or consume
pub async fn process_users(users: Vec<User>) -> Result<()> {
    for user in users {  // Consumes vector
        process_user(user).await?;
    }
    Ok(())
}

// ✅ GOOD: Use references if needed later
pub async fn process_users(users: &[User]) -> Result<()> {
    for user in users {  // Borrows
        process_user(user).await?;
    }
    Ok(())
}
```

### Use Cow for Conditional Ownership

```rust
use std::borrow::Cow;

pub fn normalize_email(email: &str) -> Cow<str> {
    let trimmed = email.trim();

    if trimmed.to_lowercase() == trimmed {
        // No allocation needed
        Cow::Borrowed(trimmed)
    } else {
        // Allocate new string
        Cow::Owned(trimmed.to_lowercase())
    }
}
```

### Preallocate Vectors

```rust
// ❌ SLOW: Multiple reallocations
pub fn process_items(n: usize) -> Vec<Item> {
    let mut items = Vec::new();

    for i in 0..n {
        items.push(Item { id: i });  // May reallocate
    }

    items
}

// ✅ FAST: Single allocation
pub fn process_items(n: usize) -> Vec<Item> {
    let mut items = Vec::with_capacity(n);

    for i in 0..n {
        items.push(Item { id: i });  // No reallocation
    }

    items
}
```

### Use SmallVec for Small Collections

```rust
use smallvec::SmallVec;

// Stores up to 8 items inline (no heap allocation)
pub fn get_user_roles(user: &User) -> SmallVec<[Role; 8]> {
    let mut roles = SmallVec::new();

    roles.push(Role::User);

    if user.is_admin {
        roles.push(Role::Admin);
    }

    roles
}
```

---

## Zero-Copy Patterns

### Using Bytes for Network I/O

```rust
use bytes::Bytes;
use axum::body::Body;

// ✅ Zero-copy body handling
pub async fn stream_file(file_path: &str) -> Result<Body> {
    let file = tokio::fs::File::open(file_path).await?;
    let stream = ReaderStream::new(file);
    let body = Body::from_stream(stream);
    Ok(body)
}
```

### Slice Instead of Vec

```rust
// ❌ ALLOCATES: Returns owned Vec
pub fn get_first_ten(items: &Vec<Item>) -> Vec<Item> {
    items.iter().take(10).cloned().collect()
}

// ✅ ZERO-COPY: Returns slice
pub fn get_first_ten(items: &[Item]) -> &[Item] {
    &items[..10.min(items.len())]
}
```

### Arc for Shared Data

```rust
use std::sync::Arc;

// ✅ Share data without cloning
#[derive(Clone)]
pub struct AppState {
    config: Arc<Config>,  // Shared, not cloned
    db: Arc<PgPool>,      // Shared, not cloned
}

impl AppState {
    pub fn new(config: Config, db: PgPool) -> Self {
        Self {
            config: Arc::new(config),
            db: Arc::new(db),
        }
    }
}
```

---

## Database Query Optimization

### Avoid N+1 Queries

```rust
// ❌ SLOW: N+1 queries (1 + N)
pub async fn get_users_with_posts_slow(
    ctx: &Ctx,
    mm: &ModelManager,
) -> Result<Vec<UserWithPosts>> {
    let users = UserBmc::list(ctx, mm, None, None).await?;

    let mut results = Vec::new();

    for user in users {
        // N additional queries!
        let posts = PostBmc::list_for_user(ctx, mm, user.id).await?;
        results.push(UserWithPosts { user, posts });
    }

    Ok(results)
}

// ✅ FAST: 2 queries total
pub async fn get_users_with_posts_fast(
    ctx: &Ctx,
    mm: &ModelManager,
) -> Result<Vec<UserWithPosts>> {
    let users = UserBmc::list(ctx, mm, None, None).await?;
    let user_ids: Vec<i64> = users.iter().map(|u| u.id).collect();

    // Single query for all posts
    let all_posts = PostBmc::list_for_users(ctx, mm, &user_ids).await?;

    // Group by user_id
    let mut posts_by_user: HashMap<i64, Vec<Post>> = HashMap::new();
    for post in all_posts {
        posts_by_user.entry(post.user_id).or_default().push(post);
    }

    // Combine
    let results = users
        .into_iter()
        .map(|user| UserWithPosts {
            posts: posts_by_user.remove(&user.id).unwrap_or_default(),
            user,
        })
        .collect();

    Ok(results)
}
```

### Use Database Indexes

```sql
-- Create indexes for frequently queried columns
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);

-- Composite index for common queries
CREATE INDEX idx_posts_user_created ON posts(user_id, created_at DESC);

-- Check index usage
EXPLAIN ANALYZE
SELECT * FROM posts WHERE user_id = 123 ORDER BY created_at DESC;
```

### Batch Inserts

```rust
// ❌ SLOW: Individual inserts
pub async fn create_posts_slow(
    ctx: &Ctx,
    mm: &ModelManager,
    posts: Vec<PostForCreate>,
) -> Result<Vec<i64>> {
    let mut ids = Vec::new();

    for post in posts {
        let id = PostBmc::create(ctx, mm, post).await?;
        ids.push(id);
    }

    Ok(ids)
}

// ✅ FAST: Batch insert
pub async fn create_posts_batch(
    ctx: &Ctx,
    mm: &ModelManager,
    posts: Vec<PostForCreate>,
) -> Result<Vec<i64>> {
    let mut tx = mm.dbx().pool().begin().await?;

    let mut ids = Vec::with_capacity(posts.len());

    for post in posts {
        let id = sqlx::query_scalar!(
            "INSERT INTO posts (title, content, user_id) VALUES ($1, $2, $3) RETURNING id",
            post.title,
            post.content,
            post.user_id
        )
        .fetch_one(&mut *tx)
        .await?;

        ids.push(id);
    }

    tx.commit().await?;

    Ok(ids)
}
```

### Connection Pooling Tuning

```rust
use sqlx::postgres::PgPoolOptions;

pub async fn create_optimized_pool(database_url: &str) -> Result<PgPool> {
    PgPoolOptions::new()
        .max_connections(20)              // Tune based on load
        .min_connections(5)                // Keep warm connections
        .acquire_timeout(Duration::from_secs(3))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .test_before_acquire(false)        // Skip health check for speed
        .connect(database_url)
        .await
        .map_err(Into::into)
}
```

---

## Caching Strategies

### In-Memory Cache with moka

```rust
use moka::future::Cache;
use std::sync::Arc;

pub struct CachedUserService {
    cache: Arc<Cache<i64, User>>,
    mm: ModelManager,
}

impl CachedUserService {
    pub fn new(mm: ModelManager) -> Self {
        let cache = Cache::builder()
            .max_capacity(10_000)
            .time_to_live(Duration::from_secs(300))  // 5 minutes
            .build();

        Self {
            cache: Arc::new(cache),
            mm,
        }
    }

    pub async fn get_user(&self, ctx: &Ctx, user_id: i64) -> Result<User> {
        // Try cache first
        if let Some(user) = self.cache.get(&user_id).await {
            return Ok(user);
        }

        // Cache miss - fetch from database
        let user = UserBmc::get(ctx, &self.mm, user_id).await?;

        // Store in cache
        self.cache.insert(user_id, user.clone()).await;

        Ok(user)
    }

    pub async fn invalidate(&self, user_id: i64) {
        self.cache.invalidate(&user_id).await;
    }
}
```

### Redis Caching

```rust
use redis::AsyncCommands;

pub async fn get_user_cached(
    redis: &mut redis::aio::Connection,
    ctx: &Ctx,
    mm: &ModelManager,
    user_id: i64,
) -> Result<User> {
    let cache_key = format!("user:{}", user_id);

    // Try cache
    let cached: Option<String> = redis.get(&cache_key).await?;

    if let Some(data) = cached {
        let user: User = serde_json::from_str(&data)?;
        return Ok(user);
    }

    // Cache miss
    let user = UserBmc::get(ctx, mm, user_id).await?;

    // Store in cache (5 minute TTL)
    let json = serde_json::to_string(&user)?;
    redis.set_ex(&cache_key, json, 300).await?;

    Ok(user)
}
```

---

## Compilation Optimization

### Release Profile Optimization

```toml
# Cargo.toml

[profile.release]
opt-level = 3              # Maximum optimization
lto = "fat"                # Link-time optimization
codegen-units = 1          # Better optimization, slower compile
strip = true               # Remove debug symbols
panic = "abort"            # Smaller binary

[profile.release-fast]
inherits = "release"
opt-level = 3
lto = "thin"               # Faster LTO
codegen-units = 16         # Faster compile
```

### Build for Production

```bash
# Build with full optimizations
cargo build --release --profile=release

# Cross-compile for different targets
rustup target add x86_64-unknown-linux-musl
cargo build --release --target=x86_64-unknown-linux-musl

# Check binary size
ls -lh target/release/web-server

# Strip symbols if not already done
strip target/release/web-server
```

### Dependency Optimization

```toml
# Cargo.toml

# Use cargo-deny to audit dependencies
[workspace.metadata.cargo-deny]

# Minimize dependency tree
[dependencies]
# Only include needed features
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }

# Replace heavy dependencies with lighter alternatives
# Instead of: serde_json
# Use: simd-json for performance-critical paths
```

### Compile-Time Feature Flags

```toml
[features]
default = ["postgres"]
postgres = ["sqlx/postgres"]
mysql = ["sqlx/mysql"]
sqlite = ["sqlx/sqlite"]

# Disable unused features
full = ["postgres", "redis", "s3"]
minimal = []
```

---

## Performance Monitoring

### Metrics with Prometheus

```rust
use prometheus::{IntCounter, HistogramVec, register_int_counter, register_histogram_vec};

lazy_static! {
    static ref HTTP_REQUESTS: IntCounter = register_int_counter!(
        "http_requests_total",
        "Total HTTP requests"
    ).unwrap();

    static ref HTTP_DURATION: HistogramVec = register_histogram_vec!(
        "http_request_duration_seconds",
        "HTTP request duration",
        &["method", "endpoint"]
    ).unwrap();
}

pub async fn metrics_middleware(
    request: Request,
    next: Next,
) -> Response {
    let method = request.method().to_string();
    let path = request.uri().path().to_string();

    HTTP_REQUESTS.inc();

    let timer = HTTP_DURATION
        .with_label_values(&[&method, &path])
        .start_timer();

    let response = next.run(request).await;

    timer.observe_duration();

    response
}
```

### Custom Metrics Endpoint

```rust
use prometheus::{Encoder, TextEncoder};

pub async fn metrics_handler() -> Result<String, ApiError> {
    let encoder = TextEncoder::new();
    let metric_families = prometheus::gather();

    let mut buffer = Vec::new();
    encoder.encode(&metric_families, &mut buffer)?;

    Ok(String::from_utf8(buffer)?)
}

// Add route
Router::new()
    .route("/metrics", get(metrics_handler))
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [async-patterns.md](async-patterns.md)
- [database-sqlx.md](database-sqlx.md)
- [testing-guide.md](testing-guide.md)
- [deployment-guide.md](deployment-guide.md)
