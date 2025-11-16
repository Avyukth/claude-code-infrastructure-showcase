# Database Patterns - SQLx and Sea-Query

Complete guide to database operations using SQLx with compile-time verification and Sea-Query for type-safe query building.

## Table of Contents

- [SQLx Setup](#sqlx-setup)
- [Compile-Time Query Verification](#compile-time-query-verification)
- [Sea-Query Integration](#sea-query-integration)
- [Connection Pooling](#connection-pooling)
- [Transactions](#transactions)
- [Migrations](#migrations)
- [Performance Optimization](#performance-optimization)

---

## SQLx Setup

### Dependencies

```toml
[dependencies]
sqlx = { version = "0.8", features = [
    "runtime-tokio",
    "postgres",
    "macros",
    "migrate",
    "uuid",
    "time",
] }
sea-query = "0.32"
sea-query-binder = { version = "0.7", features = ["sqlx-postgres"] }
```

### Database Connection

```rust
use sqlx::postgres::{PgPool, PgPoolOptions};

pub async fn create_pool(database_url: &str) -> Result<PgPool> {
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(database_url)
        .await?;

    Ok(pool)
}
```

---

## Compile-Time Query Verification

### query! Macro (Type-Safe)

```rust
// SQLx verifies this query at compile time against your database!
pub async fn get_user_by_id(pool: &PgPool, id: i64) -> Result<Option<User>> {
    let user = sqlx::query_as!(
        User,
        r#"
        SELECT id, username, email, cid, ctime, mid, mtime
        FROM "user"
        WHERE id = $1
        "#,
        id
    )
    .fetch_optional(pool)
    .await?;

    Ok(user)
}
```

### Handling NULL Columns

```rust
#[derive(Debug, FromRow)]
pub struct User {
    pub id: i64,
    pub username: String,
    pub email: String,
    pub bio: Option<String>,      // NULL column
    pub avatar_url: Option<String>, // NULL column
}

let user = sqlx::query_as!(
    User,
    r#"
    SELECT id, username, email, bio, avatar_url
    FROM users
    WHERE id = $1
    "#,
    id
)
.fetch_one(pool)
.await?;
```

### Type Overrides

```rust
// Override SQLx's type inference
let user = sqlx::query_as!(
    User,
    r#"
    SELECT
        id,
        username,
        created_at as "created_at: time::OffsetDateTime"
    FROM users
    WHERE id = $1
    "#,
    id
)
.fetch_one(pool)
.await?;
```

---

## Sea-Query Integration

### Why Sea-Query?

- ✅ Type-safe query builder
- ✅ Database-agnostic (can switch PostgreSQL → MySQL)
- ✅ Composable queries
- ✅ No string interpolation

### Basic Query Building

```rust
use sea_query::{Expr, PostgresQueryBuilder, Query};
use sea_query_binder::SqlxBinder;

#[derive(Iden)]
enum User {
    Table,
    Id,
    Username,
    Email,
}

pub async fn list_users(pool: &PgPool) -> Result<Vec<UserRow>> {
    let mut query = Query::select();
    query
        .from(User::Table)
        .columns([User::Id, User::Username, User::Email])
        .order_by(User::Id, Order::Asc);

    // Convert to SQL
    let (sql, values) = query.build_sqlx(PostgresQueryBuilder);

    let users = sqlx::query_as_with::<_, UserRow, _>(&sql, values)
        .fetch_all(pool)
        .await?;

    Ok(users)
}
```

### Conditional Queries

```rust
pub async fn search_users(
    pool: &PgPool,
    username_filter: Option<&str>,
    email_filter: Option<&str>,
) -> Result<Vec<User>> {
    let mut query = Query::select();
    query
        .from(User::Table)
        .columns([User::Id, User::Username, User::Email]);

    // Add conditions dynamically
    if let Some(username) = username_filter {
        query.and_where(Expr::col(User::Username).like(format!("%{username}%")));
    }

    if let Some(email) = email_filter {
        query.and_where(Expr::col(User::Email).eq(email));
    }

    let (sql, values) = query.build_sqlx(PostgresQueryBuilder);

    let users = sqlx::query_as_with(&sql, values)
        .fetch_all(pool)
        .await?;

    Ok(users)
}
```

### Joins

```rust
#[derive(Iden)]
enum Post {
    Table,
    Id,
    Title,
    UserId,
}

pub async fn get_users_with_post_count(pool: &PgPool) -> Result<Vec<UserWithCount>> {
    let query = Query::select()
        .from(User::Table)
        .columns([
            (User::Table, User::Id),
            (User::Table, User::Username),
        ])
        .expr_as(
            Expr::col((Post::Table, Post::Id)).count(),
            Alias::new("post_count"),
        )
        .left_join(
            Post::Table,
            Expr::col((User::Table, User::Id)).equals((Post::Table, Post::UserId)),
        )
        .group_by_col((User::Table, User::Id))
        .to_owned();

    let (sql, values) = query.build_sqlx(PostgresQueryBuilder);

    let results = sqlx::query_as_with(&sql, values)
        .fetch_all(pool)
        .await?;

    Ok(results)
}
```

---

## Connection Pooling

### Pool Configuration

```rust
use sqlx::postgres::PgPoolOptions;
use std::time::Duration;

pub async fn create_optimized_pool(database_url: &str) -> Result<PgPool> {
    PgPoolOptions::new()
        .max_connections(20)              // Maximum connections
        .min_connections(5)                // Keep alive minimum
        .acquire_timeout(Duration::from_secs(3))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .connect(database_url)
        .await
        .map_err(Into::into)
}
```

### ModelManager Pattern

```rust
#[derive(Clone)]
pub struct ModelManager {
    db: Arc<Db>,
}

impl ModelManager {
    pub async fn new() -> Result<Self> {
        let db = Db::new().await?;
        Ok(ModelManager { db: Arc::new(db) })
    }

    pub fn db(&self) -> &Db {
        &self.db
    }

    pub async fn new_with_txn(&self) -> Result<Self> {
        let db = self.db.new_with_txn().await?;
        Ok(ModelManager { db: Arc::new(db) })
    }
}

pub struct Db {
    pool: PgPool,
    tx: Mutex<Option<Tx>>,
}

impl Db {
    pub async fn new() -> Result<Self> {
        let pool = create_pool(&database_url).await?;
        Ok(Db {
            pool,
            tx: Mutex::new(None),
        })
    }
}
```

---

## Transactions

### Basic Transaction

```rust
pub async fn transfer_funds(
    pool: &PgPool,
    from_account: i64,
    to_account: i64,
    amount: Decimal,
) -> Result<()> {
    let mut tx = pool.begin().await?;

    // Debit from account
    sqlx::query!(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        amount,
        from_account
    )
    .execute(&mut *tx)
    .await?;

    // Credit to account
    sqlx::query!(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount,
        to_account
    )
    .execute(&mut *tx)
    .await?;

    // Commit transaction
    tx.commit().await?;

    Ok(())
}
```

### Transaction with Rollback

```rust
pub async fn create_user_with_profile(
    pool: &PgPool,
    user_data: UserForCreate,
    profile_data: ProfileForCreate,
) -> Result<i64> {
    let mut tx = pool.begin().await?;

    // Create user
    let user_id = sqlx::query_scalar!(
        "INSERT INTO users (username, email) VALUES ($1, $2) RETURNING id",
        user_data.username,
        user_data.email
    )
    .fetch_one(&mut *tx)
    .await?;

    // Create profile (if fails, user creation also rolls back)
    sqlx::query!(
        "INSERT INTO profiles (user_id, first_name, last_name) VALUES ($1, $2, $3)",
        user_id,
        profile_data.first_name,
        profile_data.last_name
    )
    .execute(&mut *tx)
    .await?;

    // Commit both operations
    tx.commit().await?;

    Ok(user_id)
}
```

---

## Migrations

### Migration Files

```sql
-- migrations/001_create_users.up.sql
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,

    cid BIGINT NOT NULL,
    ctime TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    mid BIGINT NOT NULL,
    mtime TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

```sql
-- migrations/001_create_users.down.sql
DROP TABLE IF EXISTS users;
```

### Running Migrations

```rust
use sqlx::migrate::Migrator;

pub static MIGRATOR: Migrator = sqlx::migrate!("./migrations");

pub async fn run_migrations(pool: &PgPool) -> Result<()> {
    MIGRATOR.run(pool).await?;
    Ok(())
}

// In main.rs
#[tokio::main]
async fn main() -> Result<()> {
    let pool = create_pool(&database_url).await?;

    // Run migrations on startup
    run_migrations(&pool).await?;

    // Start server
    Ok(())
}
```

---

## Performance Optimization

### Batch Inserts

```rust
// ❌ SLOW: Individual inserts
pub async fn create_posts_slow(pool: &PgPool, posts: Vec<PostForCreate>) -> Result<()> {
    for post in posts {
        sqlx::query!(
            "INSERT INTO posts (title, content, user_id) VALUES ($1, $2, $3)",
            post.title,
            post.content,
            post.user_id
        )
        .execute(pool)
        .await?;
    }
    Ok(())
}

// ✅ FAST: Batch insert
pub async fn create_posts_batch(pool: &PgPool, posts: Vec<PostForCreate>) -> Result<()> {
    let mut tx = pool.begin().await?;

    for post in posts {
        sqlx::query!(
            "INSERT INTO posts (title, content, user_id) VALUES ($1, $2, $3)",
            post.title,
            post.content,
            post.user_id
        )
        .execute(&mut *tx)
        .await?;
    }

    tx.commit().await?;
    Ok(())
}
```

### Prepared Statements

```rust
// SQLx automatically prepares and caches statements
pub async fn get_users_by_ids(pool: &PgPool, ids: &[i64]) -> Result<Vec<User>> {
    let mut users = Vec::new();

    for id in ids {
        // This query is prepared once and reused
        let user = sqlx::query_as!(
            User,
            "SELECT * FROM users WHERE id = $1",
            id
        )
        .fetch_one(pool)
        .await?;

        users.push(user);
    }

    Ok(users)
}
```

### Query Result Streaming

```rust
use futures::TryStreamExt;

pub async fn process_large_dataset(pool: &PgPool) -> Result<()> {
    let mut stream = sqlx::query_as!(
        User,
        "SELECT * FROM users ORDER BY id"
    )
    .fetch(pool);

    while let Some(user) = stream.try_next().await? {
        // Process each user without loading all into memory
        process_user(user).await?;
    }

    Ok(())
}
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [architecture-overview.md](architecture-overview.md)
- [async-patterns.md](async-patterns.md)
- [performance-optimization.md](performance-optimization.md)
