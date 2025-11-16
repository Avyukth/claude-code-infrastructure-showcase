# Async Patterns - Tokio and Async/Await Best Practices

Complete guide to asynchronous programming in Rust backend services using Tokio.

## Table of Contents

- [Tokio Runtime Setup](#tokio-runtime-setup)
- [Async Function Patterns](#async-function-patterns)
- [Concurrent Operations](#concurrent-operations)
- [Blocking Operations](#blocking-operations)
- [Common Pitfalls](#common-pitfalls)
- [Performance Tips](#performance-tips)

---

## Tokio Runtime Setup

### Basic Runtime Configuration

```rust
// main.rs - Simple setup

#[tokio::main]
async fn main() -> Result<()> {
    let pool = PgPool::connect(&database_url).await?;

    let app = create_app(pool);

    axum::Server::bind(&"0.0.0.0:3000".parse()?)
        .serve(app.into_make_service())
        .await?;

    Ok(())
}
```

### Custom Runtime Configuration

```rust
// main.rs - Production setup

use tokio::runtime::Builder;

fn main() -> Result<()> {
    // Custom runtime with specific thread pool size
    let runtime = Builder::new_multi_thread()
        .worker_threads(4)
        .thread_name("api-worker")
        .enable_all()
        .build()?;

    runtime.block_on(async {
        run_server().await
    })
}

async fn run_server() -> Result<()> {
    let pool = PgPool::connect(&database_url).await?;
    // Start server
    Ok(())
}
```

---

## Async Function Patterns

### Always Use ? for Error Propagation

```rust
// ✅ CORRECT: Use ? for async operations
pub async fn get_user_with_posts(
    ctx: &Ctx,
    mm: &ModelManager,
    user_id: i64,
) -> Result<UserWithPosts> {
    let user = UserBmc::get(ctx, mm, user_id).await?;
    let posts = PostBmc::list_for_user(ctx, mm, user_id).await?;

    Ok(UserWithPosts { user, posts })
}

// ❌ WRONG: Don't use .await.unwrap()
pub async fn bad_example(ctx: &Ctx, mm: &ModelManager, id: i64) -> Result<User> {
    let user = UserBmc::get(ctx, mm, id).await.unwrap(); // PANIC if error!
    Ok(user)
}
```

### Async Traits

```rust
use async_trait::async_trait;

#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: i64) -> Result<Option<User>>;
    async fn find_by_email(&self, email: &str) -> Result<Option<User>>;
    async fn save(&self, user: &User) -> Result<()>;
}

// Implementation
pub struct PostgresUserRepository {
    pool: Arc<PgPool>,
}

#[async_trait]
impl UserRepository for PostgresUserRepository {
    async fn find_by_id(&self, id: i64) -> Result<Option<User>> {
        let user = sqlx::query_as!(
            User,
            "SELECT * FROM users WHERE id = $1",
            id
        )
        .fetch_optional(&*self.pool)
        .await?;

        Ok(user)
    }

    async fn save(&self, user: &User) -> Result<()> {
        sqlx::query!(
            "UPDATE users SET username = $1, email = $2 WHERE id = $3",
            user.username,
            user.email,
            user.id
        )
        .execute(&*self.pool)
        .await?;

        Ok(())
    }
}
```

---

## Concurrent Operations

### Parallel Execution with tokio::spawn

```rust
use tokio::task::JoinSet;

pub async fn process_batch_concurrent(
    ctx: &Ctx,
    mm: &ModelManager,
    user_ids: Vec<i64>,
) -> Result<Vec<User>> {
    let mut set = JoinSet::new();

    for user_id in user_ids {
        let ctx = ctx.clone();
        let mm = mm.clone();

        set.spawn(async move {
            UserBmc::get(&ctx, &mm, user_id).await
        });
    }

    let mut users = Vec::new();
    while let Some(res) = set.join_next().await {
        let user = res??; // Handle join error and inner error
        users.push(user);
    }

    Ok(users)
}
```

### Parallel with try_join!

```rust
use tokio::try_join;

pub async fn get_dashboard_data(
    ctx: &Ctx,
    mm: &ModelManager,
    user_id: i64,
) -> Result<DashboardData> {
    // Execute all queries in parallel
    let (user, posts, subscriptions) = try_join!(
        UserBmc::get(ctx, mm, user_id),
        PostBmc::list_for_user(ctx, mm, user_id),
        SubscriptionBmc::list_for_user(ctx, mm, user_id),
    )?;

    Ok(DashboardData {
        user,
        posts,
        subscriptions,
    })
}
```

### Handling Partial Failures

```rust
use futures::future::join_all;

pub async fn send_notifications(
    users: Vec<User>,
) -> Result<Vec<Result<(), NotificationError>>> {
    let futures = users.into_iter().map(|user| {
        async move {
            send_email(&user.email).await
        }
    });

    // Collect all results (successes and failures)
    let results = join_all(futures).await;

    Ok(results)
}
```

---

## Blocking Operations

### spawn_blocking for CPU-Intensive Work

```rust
use tokio::task;

pub async fn hash_password(password: String) -> Result<String> {
    // Argon2 is CPU-intensive, move to blocking thread
    let hashed = task::spawn_blocking(move || {
        let salt = SaltString::generate(&mut OsRng);
        let argon2 = Argon2::default();

        argon2
            .hash_password(password.as_bytes(), &salt)
            .map_err(|e| Error::PasswordHashingFailed(e.to_string()))
            .map(|hash| hash.to_string())
    })
    .await??; // Handle join error and inner error

    Ok(hashed)
}
```

### File I/O with tokio::fs

```rust
use tokio::fs;

// ✅ CORRECT: Use tokio::fs for async file operations
pub async fn read_config() -> Result<Config> {
    let contents = fs::read_to_string("config.toml").await?;
    let config: Config = toml::from_str(&contents)?;
    Ok(config)
}

// ❌ WRONG: Don't use std::fs in async code
pub async fn bad_read_config() -> Result<Config> {
    // This blocks the async runtime!
    let contents = std::fs::read_to_string("config.toml")?;
    let config: Config = toml::from_str(&contents)?;
    Ok(config)
}
```

---

## Common Pitfalls

### Pitfall 1: Holding Locks Across Await

```rust
use tokio::sync::Mutex;

// ❌ WRONG: Holding lock across await
pub async fn bad_mutex_usage(data: Arc<Mutex<Vec<String>>>) -> Result<()> {
    let mut guard = data.lock().await;
    guard.push("item".to_string());

    // Lock still held during async operation!
    some_async_operation().await?;

    Ok(())
}

// ✅ CORRECT: Release lock before await
pub async fn good_mutex_usage(data: Arc<Mutex<Vec<String>>>) -> Result<()> {
    {
        let mut guard = data.lock().await;
        guard.push("item".to_string());
    } // Lock released here

    some_async_operation().await?;

    Ok(())
}
```

### Pitfall 2: Not Handling Cancellation

```rust
// ✅ CORRECT: Handle cancellation gracefully
pub async fn graceful_shutdown(
    server: axum::Server<...>,
    shutdown_signal: impl Future<Output = ()>,
) -> Result<()> {
    server
        .with_graceful_shutdown(shutdown_signal)
        .await?;

    Ok(())
}

// Setup shutdown signal
async fn shutdown_signal() {
    tokio::signal::ctrl_c()
        .await
        .expect("failed to install CTRL+C signal handler");
}
```

### Pitfall 3: Spawning Without Bounds

```rust
// ❌ WRONG: Unbounded spawning
pub async fn process_requests(requests: Vec<Request>) -> Result<()> {
    for req in requests {
        tokio::spawn(async move {
            process_request(req).await
        });
    }
    Ok(())
}

// ✅ CORRECT: Use semaphore to limit concurrency
use tokio::sync::Semaphore;

pub async fn bounded_process_requests(
    requests: Vec<Request>,
    max_concurrent: usize,
) -> Result<()> {
    let semaphore = Arc::new(Semaphore::new(max_concurrent));
    let mut set = JoinSet::new();

    for req in requests {
        let permit = semaphore.clone().acquire_owned().await?;

        set.spawn(async move {
            let result = process_request(req).await;
            drop(permit); // Release semaphore
            result
        });
    }

    while let Some(res) = set.join_next().await {
        res??;
    }

    Ok(())
}
```

---

## Performance Tips

### Tip 1: Batch Database Operations

```rust
// ❌ SLOW: N+1 queries
pub async fn get_users_with_posts_slow(
    ctx: &Ctx,
    mm: &ModelManager,
    user_ids: Vec<i64>,
) -> Result<Vec<UserWithPosts>> {
    let mut results = Vec::new();

    for user_id in user_ids {
        let user = UserBmc::get(ctx, mm, user_id).await?;
        let posts = PostBmc::list_for_user(ctx, mm, user_id).await?;
        results.push(UserWithPosts { user, posts });
    }

    Ok(results)
}

// ✅ FAST: Batch queries
pub async fn get_users_with_posts_fast(
    ctx: &Ctx,
    mm: &ModelManager,
    user_ids: Vec<i64>,
) -> Result<Vec<UserWithPosts>> {
    // Single query for all users
    let users = UserBmc::list_by_ids(ctx, mm, &user_ids).await?;

    // Single query for all posts
    let all_posts = PostBmc::list_for_users(ctx, mm, &user_ids).await?;

    // Group posts by user_id
    let mut posts_by_user: HashMap<i64, Vec<Post>> = HashMap::new();
    for post in all_posts {
        posts_by_user
            .entry(post.user_id)
            .or_default()
            .push(post);
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

### Tip 2: Use Channels for Producer-Consumer

```rust
use tokio::sync::mpsc;

pub async fn process_stream(items: Vec<Item>) -> Result<()> {
    let (tx, mut rx) = mpsc::channel(100);

    // Spawn producer
    tokio::spawn(async move {
        for item in items {
            if tx.send(item).await.is_err() {
                break;
            }
        }
    });

    // Consumer
    while let Some(item) = rx.recv().await {
        process_item(item).await?;
    }

    Ok(())
}
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [error-handling.md](error-handling.md)
- [performance-optimization.md](performance-optimization.md)
