# Caching Patterns - Moka and Redis

Complete guide to implementing high-performance caching in Rust backend services using in-memory (Moka) and distributed (Redis) caching strategies.

## Table of Contents

- [Caching Overview](#caching-overview)
- [In-Memory Caching with Moka](#in-memory-caching-with-moka)
- [Distributed Caching with Redis](#distributed-caching-with-redis)
- [Cache Strategies](#cache-strategies)
- [BMC Pattern Integration](#bmc-pattern-integration)
- [Cache Invalidation](#cache-invalidation)
- [Cache-Aside Pattern](#cache-aside-pattern)
- [Write-Through and Write-Behind](#write-through-and-write-behind)
- [Multi-Level Caching](#multi-level-caching)
- [Monitoring and Metrics](#monitoring-and-metrics)
- [Testing Caching Logic](#testing-caching-logic)
- [Complete Examples](#complete-examples)

---

## Caching Overview

### When to Use Caching

**Use Cases:**
- ✅ **Expensive computations** - Cache results of complex calculations
- ✅ **Database queries** - Reduce database load
- ✅ **External API calls** - Avoid rate limits, reduce latency
- ✅ **Session data** - Fast user session lookup
- ✅ **Static content** - Cache rarely-changing data

**Moka vs Redis:**

| Feature | Moka (In-Memory) | Redis (Distributed) |
|---------|------------------|---------------------|
| **Latency** | ~10-100 ns | ~1-5 ms (network) |
| **Throughput** | Millions ops/s | 100K-500K ops/s |
| **Persistence** | None (process-local) | Optional |
| **Sharing** | Single process | Multi-process/server |
| **Eviction** | LRU, LFU, TTL | LRU, LFU, TTL |
| **Use Case** | Hot data, per-instance | Shared state, sessions |

---

## In-Memory Caching with Moka

### Dependencies

```toml
[dependencies]
# In-memory cache
moka = { version = "0.12", features = ["future"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Hashing (for custom keys)
ahash = "0.8"
```

### Basic Setup

```rust
// lib-infrastructure/src/cache/moka.rs

use moka::future::Cache;
use std::time::Duration;
use std::sync::Arc;

#[derive(Clone)]
pub struct MokaCache<K, V> {
    cache: Cache<K, V>,
}

impl<K, V> MokaCache<K, V>
where
    K: std::hash::Hash + Eq + Send + Sync + 'static,
    V: Clone + Send + Sync + 'static,
{
    pub fn new(max_capacity: u64, ttl: Duration) -> Self {
        let cache = Cache::builder()
            .max_capacity(max_capacity)
            .time_to_live(ttl)
            .build();

        Self { cache }
    }

    pub async fn get(&self, key: &K) -> Option<V> {
        self.cache.get(key).await
    }

    pub async fn insert(&self, key: K, value: V) {
        self.cache.insert(key, value).await;
    }

    pub async fn invalidate(&self, key: &K) {
        self.cache.invalidate(key).await;
    }

    pub async fn get_with<F, Fut>(&self, key: K, init: F) -> V
    where
        F: FnOnce() -> Fut,
        Fut: std::future::Future<Output = V>,
    {
        self.cache
            .get_with(key, async move { init().await })
            .await
    }
}

// Specialized user cache
pub type UserCache = MokaCache<i64, Arc<User>>;

impl UserCache {
    pub fn new_default() -> Self {
        Self::new(
            10_000,                      // 10K users
            Duration::from_secs(3600),   // 1 hour TTL
        )
    }
}
```

### Advanced Configuration

```rust
use moka::future::Cache;

pub fn create_advanced_cache<K, V>() -> Cache<K, V>
where
    K: std::hash::Hash + Eq + Send + Sync + 'static,
    V: Clone + Send + Sync + 'static,
{
    Cache::builder()
        // Capacity
        .max_capacity(10_000)

        // Time-based expiration
        .time_to_live(Duration::from_secs(300))      // 5 min TTL
        .time_to_idle(Duration::from_secs(60))       // Evict if not accessed for 1 min

        // Eviction listener
        .eviction_listener(|key, value, cause| {
            tracing::debug!("Evicted {:?}: {:?} (cause: {:?})", key, value, cause);
        })

        // Initial capacity
        .initial_capacity(1000)

        .build()
}
```

### Weighted Cache (Memory-Aware)

```rust
use moka::future::Cache;

pub fn create_weighted_cache<K, V>() -> Cache<K, V>
where
    K: std::hash::Hash + Eq + Send + Sync + 'static,
    V: Clone + Send + Sync + 'static,
{
    Cache::builder()
        .max_capacity(100_000_000) // 100 MB
        .weigher(|key: &K, value: &V| -> u32 {
            // Estimate size in bytes
            std::mem::size_of_val(key) as u32 + std::mem::size_of_val(value) as u32
        })
        .build()
}
```

---

## Distributed Caching with Redis

### Dependencies

```toml
[dependencies]
# Redis client
redis = { version = "0.25", features = ["tokio-comp", "connection-manager"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

### Connection Setup

```rust
// lib-infrastructure/src/cache/redis.rs

use redis::{
    Client, AsyncCommands, aio::ConnectionManager,
    RedisError,
};
use serde::{Serialize, de::DeserializeOwned};
use std::time::Duration;

#[derive(Clone)]
pub struct RedisCache {
    conn: ConnectionManager,
}

impl RedisCache {
    pub async fn new(redis_url: &str) -> Result<Self> {
        let client = Client::open(redis_url)?;
        let conn = ConnectionManager::new(client).await?;

        Ok(Self { conn })
    }

    pub async fn get<T: DeserializeOwned>(&self, key: &str) -> Result<Option<T>> {
        let mut conn = self.conn.clone();
        let value: Option<String> = conn.get(key).await?;

        match value {
            Some(json) => Ok(Some(serde_json::from_str(&json)?)),
            None => Ok(None),
        }
    }

    pub async fn set<T: Serialize>(
        &self,
        key: &str,
        value: &T,
        ttl: Option<Duration>,
    ) -> Result<()> {
        let mut conn = self.conn.clone();
        let json = serde_json::to_string(value)?;

        match ttl {
            Some(duration) => {
                conn.set_ex(key, json, duration.as_secs()).await?;
            }
            None => {
                conn.set(key, json).await?;
            }
        }

        Ok(())
    }

    pub async fn delete(&self, key: &str) -> Result<()> {
        let mut conn = self.conn.clone();
        conn.del(key).await?;
        Ok(())
    }

    pub async fn exists(&self, key: &str) -> Result<bool> {
        let mut conn = self.conn.clone();
        Ok(conn.exists(key).await?)
    }

    pub async fn expire(&self, key: &str, ttl: Duration) -> Result<()> {
        let mut conn = self.conn.clone();
        conn.expire(key, ttl.as_secs() as i64).await?;
        Ok(())
    }
}
```

### Advanced Redis Operations

```rust
impl RedisCache {
    // Hash operations (for nested data)
    pub async fn hset<T: Serialize>(
        &self,
        key: &str,
        field: &str,
        value: &T,
    ) -> Result<()> {
        let mut conn = self.conn.clone();
        let json = serde_json::to_string(value)?;
        conn.hset(key, field, json).await?;
        Ok(())
    }

    pub async fn hget<T: DeserializeOwned>(
        &self,
        key: &str,
        field: &str,
    ) -> Result<Option<T>> {
        let mut conn = self.conn.clone();
        let value: Option<String> = conn.hget(key, field).await?;

        match value {
            Some(json) => Ok(Some(serde_json::from_str(&json)?)),
            None => Ok(None),
        }
    }

    // List operations (for queues)
    pub async fn lpush<T: Serialize>(&self, key: &str, value: &T) -> Result<()> {
        let mut conn = self.conn.clone();
        let json = serde_json::to_string(value)?;
        conn.lpush(key, json).await?;
        Ok(())
    }

    pub async fn rpop<T: DeserializeOwned>(&self, key: &str) -> Result<Option<T>> {
        let mut conn = self.conn.clone();
        let value: Option<String> = conn.rpop(key, None).await?;

        match value {
            Some(json) => Ok(Some(serde_json::from_str(&json)?)),
            None => Ok(None),
        }
    }

    // Atomic increment
    pub async fn incr(&self, key: &str) -> Result<i64> {
        let mut conn = self.conn.clone();
        Ok(conn.incr(key, 1).await?)
    }

    // Pattern-based deletion
    pub async fn delete_pattern(&self, pattern: &str) -> Result<()> {
        let mut conn = self.conn.clone();
        let keys: Vec<String> = redis::cmd("KEYS")
            .arg(pattern)
            .query_async(&mut conn)
            .await?;

        if !keys.is_empty() {
            conn.del(keys).await?;
        }

        Ok(())
    }
}
```

---

## Cache Strategies

### Cache-Aside (Lazy Loading)

```rust
use std::sync::Arc;

pub async fn get_user_cached(
    cache: &MokaCache<i64, Arc<User>>,
    mm: &ModelManager,
    user_id: i64,
) -> Result<Arc<User>> {
    // Try cache first
    if let Some(user) = cache.get(&user_id).await {
        return Ok(user);
    }

    // Load from database
    let user = UserBmc::get(mm, user_id).await?;
    let user_arc = Arc::new(user);

    // Store in cache
    cache.insert(user_id, user_arc.clone()).await;

    Ok(user_arc)
}

// With get_with helper (atomic)
pub async fn get_user_cached_v2(
    cache: &MokaCache<i64, Arc<User>>,
    mm: &ModelManager,
    user_id: i64,
) -> Result<Arc<User>> {
    let mm_clone = mm.clone();

    let user = cache
        .get_with(user_id, async move {
            let user = UserBmc::get(&mm_clone, user_id).await.unwrap();
            Arc::new(user)
        })
        .await;

    Ok(user)
}
```

### Read-Through Pattern

```rust
pub struct CachedRepository<T> {
    cache: MokaCache<i64, Arc<T>>,
    mm: ModelManager,
}

impl CachedRepository<User> {
    pub async fn get(&self, id: i64) -> Result<Arc<User>> {
        self.cache
            .get_with(id, async {
                let user = UserBmc::get(&self.mm, id).await.unwrap();
                Arc::new(user)
            })
            .await;

        Ok(user)
    }

    pub async fn list(&self, ids: Vec<i64>) -> Result<Vec<Arc<User>>> {
        let mut users = Vec::new();

        for id in ids {
            users.push(self.get(id).await?);
        }

        Ok(users)
    }
}
```

### Write-Through Pattern

```rust
pub async fn update_user_write_through(
    cache: &MokaCache<i64, Arc<User>>,
    mm: &ModelManager,
    user_id: i64,
    data: UserForUpdate,
) -> Result<()> {
    // Update database
    UserBmc::update(mm, user_id, data).await?;

    // Update cache
    let user = UserBmc::get(mm, user_id).await?;
    cache.insert(user_id, Arc::new(user)).await;

    Ok(())
}
```

### Write-Behind (Async) Pattern

```rust
use tokio::sync::mpsc;

pub struct WriteBehindCache {
    cache: MokaCache<i64, Arc<User>>,
    write_queue: mpsc::UnboundedSender<(i64, User)>,
}

impl WriteBehindCache {
    pub fn new(mm: ModelManager) -> Self {
        let cache = MokaCache::new_default();
        let (tx, mut rx) = mpsc::unbounded_channel();

        // Background writer
        tokio::spawn(async move {
            while let Some((user_id, user)) = rx.recv().await {
                if let Err(e) = UserBmc::update_from_cache(&mm, user_id, &user).await {
                    tracing::error!("Failed to write to database: {}", e);
                }
            }
        });

        Self {
            cache,
            write_queue: tx,
        }
    }

    pub async fn update(&self, user_id: i64, user: User) {
        // Update cache immediately
        self.cache.insert(user_id, Arc::new(user.clone())).await;

        // Queue database write
        let _ = self.write_queue.send((user_id, user));
    }
}
```

---

## BMC Pattern Integration

### Cached BMC Wrapper

```rust
// lib-core/src/model/cached_user_bmc.rs

use crate::model::{UserBmc, ModelManager};
use lib_infrastructure::cache::MokaCache;
use std::sync::Arc;

pub struct CachedUserBmc {
    cache: MokaCache<i64, Arc<User>>,
    mm: ModelManager,
}

impl CachedUserBmc {
    pub fn new(mm: ModelManager) -> Self {
        Self {
            cache: MokaCache::new(10_000, Duration::from_secs(3600)),
            mm,
        }
    }

    pub async fn get(&self, ctx: &Ctx, user_id: i64) -> Result<Arc<User>> {
        let ctx_clone = ctx.clone();
        let mm_clone = self.mm.clone();

        let user = self
            .cache
            .get_with(user_id, async move {
                let user = UserBmc::get(&ctx_clone, &mm_clone, user_id)
                    .await
                    .unwrap();
                Arc::new(user)
            })
            .await;

        Ok(user)
    }

    pub async fn create(
        &self,
        ctx: &Ctx,
        data: UserForCreate,
    ) -> Result<i64> {
        let user_id = UserBmc::create(ctx, &self.mm, data).await?;

        // Invalidate cache if needed (or pre-populate)
        // self.cache.invalidate(&user_id).await;

        Ok(user_id)
    }

    pub async fn update(
        &self,
        ctx: &Ctx,
        user_id: i64,
        data: UserForUpdate,
    ) -> Result<()> {
        UserBmc::update(ctx, &self.mm, user_id, data).await?;

        // Invalidate cache
        self.cache.invalidate(&user_id).await;

        Ok(())
    }

    pub async fn delete(&self, ctx: &Ctx, user_id: i64) -> Result<()> {
        UserBmc::delete(ctx, &self.mm, user_id).await?;

        // Invalidate cache
        self.cache.invalidate(&user_id).await;

        Ok(())
    }
}
```

### Multi-Level Cache (Moka + Redis)

```rust
pub struct MultiLevelCache {
    l1: MokaCache<i64, Arc<User>>,  // In-memory (fast)
    l2: RedisCache,                  // Distributed (shared)
}

impl MultiLevelCache {
    pub async fn get(&self, user_id: i64) -> Result<Option<Arc<User>>> {
        // Try L1 cache
        if let Some(user) = self.l1.get(&user_id).await {
            return Ok(Some(user));
        }

        // Try L2 cache
        if let Some(user) = self.l2.get::<User>(&format!("user:{}", user_id)).await? {
            let user_arc = Arc::new(user);

            // Populate L1 cache
            self.l1.insert(user_id, user_arc.clone()).await;

            return Ok(Some(user_arc));
        }

        Ok(None)
    }

    pub async fn set(&self, user_id: i64, user: Arc<User>) -> Result<()> {
        // Set in both caches
        self.l1.insert(user_id, user.clone()).await;
        self.l2.set(
            &format!("user:{}", user_id),
            &*user,
            Some(Duration::from_secs(3600)),
        ).await?;

        Ok(())
    }

    pub async fn invalidate(&self, user_id: i64) -> Result<()> {
        self.l1.invalidate(&user_id).await;
        self.l2.delete(&format!("user:{}", user_id)).await?;
        Ok(())
    }
}
```

---

## Cache Invalidation

### Event-Based Invalidation

```rust
use tokio::sync::broadcast;

pub struct CacheInvalidator {
    tx: broadcast::Sender<CacheInvalidationEvent>,
}

pub enum CacheInvalidationEvent {
    User(i64),
    Post(i64),
    Pattern(String),
}

impl CacheInvalidator {
    pub fn new() -> (Self, broadcast::Receiver<CacheInvalidationEvent>) {
        let (tx, rx) = broadcast::channel(100);
        (Self { tx }, rx)
    }

    pub fn invalidate_user(&self, user_id: i64) {
        let _ = self.tx.send(CacheInvalidationEvent::User(user_id));
    }

    pub fn invalidate_pattern(&self, pattern: String) {
        let _ = self.tx.send(CacheInvalidationEvent::Pattern(pattern));
    }
}

// Consumer
pub async fn cache_invalidation_worker(
    mut rx: broadcast::Receiver<CacheInvalidationEvent>,
    cache: MokaCache<i64, Arc<User>>,
    redis: RedisCache,
) {
    while let Ok(event) = rx.recv().await {
        match event {
            CacheInvalidationEvent::User(user_id) => {
                cache.invalidate(&user_id).await;
                let _ = redis.delete(&format!("user:{}", user_id)).await;
            }
            CacheInvalidationEvent::Pattern(pattern) => {
                let _ = redis.delete_pattern(&pattern).await;
            }
            _ => {}
        }
    }
}
```

### TTL-Based Invalidation

Already handled by Moka and Redis TTL settings.

### Manual Invalidation

```rust
// Invalidate on update
impl UserBmc {
    pub async fn update_with_cache_invalidation(
        ctx: &Ctx,
        mm: &ModelManager,
        cache: &MokaCache<i64, Arc<User>>,
        user_id: i64,
        data: UserForUpdate,
    ) -> Result<()> {
        Self::update(ctx, mm, user_id, data).await?;

        // Invalidate cache
        cache.invalidate(&user_id).await;

        Ok(())
    }
}
```

---

## Monitoring and Metrics

### Moka Metrics

```rust
use prometheus::{IntCounter, Histogram};

lazy_static! {
    static ref CACHE_HITS: IntCounter =
        IntCounter::new("cache_hits_total", "Cache hits").unwrap();

    static ref CACHE_MISSES: IntCounter =
        IntCounter::new("cache_misses_total", "Cache misses").unwrap();

    static ref CACHE_GET_DURATION: Histogram =
        Histogram::new("cache_get_duration_seconds", "Cache get duration").unwrap();
}

pub async fn get_with_metrics<K, V>(
    cache: &MokaCache<K, V>,
    key: &K,
) -> Option<V>
where
    K: std::hash::Hash + Eq,
    V: Clone,
{
    let timer = CACHE_GET_DURATION.start_timer();

    let result = cache.get(key).await;

    timer.observe_duration();

    match &result {
        Some(_) => CACHE_HITS.inc(),
        None => CACHE_MISSES.inc(),
    }

    result
}
```

### Cache Statistics

```rust
impl MokaCache<K, V> {
    pub fn stats(&self) -> CacheStats {
        CacheStats {
            entry_count: self.cache.entry_count(),
            weighted_size: self.cache.weighted_size(),
            // ... more stats
        }
    }
}
```

---

## Testing Caching Logic

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_cache_hit() {
        let cache = MokaCache::new(100, Duration::from_secs(60));

        cache.insert(1, "value".to_string()).await;

        let result = cache.get(&1).await;
        assert_eq!(result, Some("value".to_string()));
    }

    #[tokio::test]
    async fn test_cache_miss() {
        let cache = MokaCache::new(100, Duration::from_secs(60));

        let result = cache.get(&999).await;
        assert_eq!(result, None);
    }

    #[tokio::test]
    async fn test_ttl_expiration() {
        let cache = MokaCache::new(100, Duration::from_millis(100));

        cache.insert(1, "value".to_string()).await;

        tokio::time::sleep(Duration::from_millis(200)).await;

        let result = cache.get(&1).await;
        assert_eq!(result, None);
    }
}
```

---

## Complete Examples

### Full Cached Service

```rust
// Service with multi-level caching
pub struct UserService {
    cache: MultiLevelCache,
    mm: ModelManager,
}

impl UserService {
    pub async fn get_user(&self, ctx: &Ctx, user_id: i64) -> Result<Arc<User>> {
        // Try cache
        if let Some(user) = self.cache.get(user_id).await? {
            return Ok(user);
        }

        // Load from database
        let user = UserBmc::get(ctx, &self.mm, user_id).await?;
        let user_arc = Arc::new(user);

        // Populate cache
        self.cache.set(user_id, user_arc.clone()).await?;

        Ok(user_arc)
    }
}
```

---

## Related Files

- [SKILL.md](../SKILL.md) - Main Rust backend guidelines
- [database-sqlx.md](database-sqlx.md) - Database patterns
- [performance-optimization.md](performance-optimization.md) - Performance tuning

---

**Pattern Status**: Production-ready caching with Rust10x integration ✅
