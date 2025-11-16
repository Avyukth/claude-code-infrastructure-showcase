# Project Structure - Workspace Organization

Complete guide to organizing Rust backend projects with multi-crate workspaces, feature flags, and dependency management.

## Table of Contents

- [Workspace Organization](#workspace-organization)
- [Crate Structure Best Practices](#crate-structure-best-practices)
- [Module Naming Conventions](#module-naming-conventions)
- [Feature Flags](#feature-flags)
- [Dependency Management](#dependency-management)
- [Build Configuration](#build-configuration)
- [Clippy Lints](#clippy-lints)

---

## Workspace Organization

### Recommended Workspace Layout

```
rust-backend/
├── Cargo.toml                  # Workspace root
├── Cargo.lock                  # Lock file (commit this!)
├── .cargo/
│   └── config.toml            # Cargo configuration
├── crates/
│   ├── libs/                  # Reusable libraries
│   │   ├── lib-auth/          # Authentication
│   │   │   ├── Cargo.toml
│   │   │   ├── src/
│   │   │   │   ├── lib.rs
│   │   │   │   ├── pwd/       # Password hashing
│   │   │   │   └── token/     # Token management
│   │   │   └── tests/
│   │   ├── lib-core/          # Domain models
│   │   │   ├── Cargo.toml
│   │   │   ├── src/
│   │   │   │   ├── lib.rs
│   │   │   │   ├── ctx/       # Context
│   │   │   │   ├── model/     # BMC pattern
│   │   │   │   └── store/     # Database
│   │   │   └── tests/
│   │   ├── lib-web/           # Web layer
│   │   ├── lib-utils/         # Utilities
│   │   └── lib-config/        # Configuration
│   ├── services/              # Executable services
│   │   ├── web-server/        # Main API server
│   │   │   ├── Cargo.toml
│   │   │   ├── src/
│   │   │   │   ├── main.rs
│   │   │   │   └── web/
│   │   │   │       ├── routes_*.rs
│   │   │   │       └── middleware/
│   │   │   ├── examples/
│   │   │   └── tests/
│   │   └── worker/            # Background worker
│   └── tools/                 # Development tools
│       └── gen-key/           # Key generator
├── migrations/                # Database migrations
│   ├── 001_users.sql
│   └── 002_posts.sql
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── scripts/
│   ├── setup.sh
│   └── deploy.sh
├── .github/
│   └── workflows/
│       └── ci.yml
├── docs/
│   ├── API.md
│   └── DEPLOYMENT.md
├── .gitignore
├── .env.example
├── README.md
└── LICENSE
```

### Workspace Cargo.toml

```toml
[workspace]
resolver = "2"
members = [
    "crates/libs/lib-auth",
    "crates/libs/lib-core",
    "crates/libs/lib-web",
    "crates/libs/lib-utils",
    "crates/libs/lib-config",
    "crates/services/web-server",
    "crates/services/worker",
    "crates/tools/gen-key",
]

# Shared dependencies across all crates
[workspace.dependencies]
# Web
axum = "0.8"
tower = "0.5"
tower-http = "0.6"
tower-cookies = "0.11"

# Async
tokio = { version = "1", features = ["full"] }
futures = "0.3"

# Database
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres", "macros", "migrate", "uuid", "time"] }
sea-query = "0.32"
sea-query-binder = { version = "0.7", features = ["sqlx-postgres"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Error handling
derive_more = { version = "1.0.0", features = ["from", "display"] }
anyhow = "1"
thiserror = "1"

# Logging/Tracing
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Utils
uuid = { version = "1", features = ["v4", "serde"] }
time = { version = "0.3", features = ["serde", "parsing", "formatting"] }
config = "0.14"

# Security
argon2 = "0.5"
blake3 = "1"
hmac = "0.12"
sha2 = "0.10"

# Validation
validator = { version = "0.18", features = ["derive"] }

# Testing
mockall = "0.13"

# Shared lints
[workspace.lints.rust]
unsafe_code = "forbid"
missing_docs = "warn"

[workspace.lints.clippy]
# Pedantic lints
pedantic = { level = "warn", priority = -1 }

# Specific denies
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
```

---

## Crate Structure Best Practices

### Library Crate (lib-core)

```
lib-core/
├── Cargo.toml
├── src/
│   ├── lib.rs                 # Public API
│   ├── ctx/
│   │   ├── mod.rs             # Context types
│   │   └── error.rs
│   ├── model/
│   │   ├── mod.rs             # Re-exports
│   │   ├── user.rs            # User BMC
│   │   ├── post.rs            # Post BMC
│   │   ├── base/              # Shared CRUD
│   │   │   ├── mod.rs
│   │   │   ├── crud_fns.rs
│   │   │   └── db_bmc.rs
│   │   └── error.rs
│   ├── store/
│   │   ├── mod.rs
│   │   ├── dbx.rs             # Database abstraction
│   │   └── error.rs
│   └── _dev_utils/            # Test utilities
│       └── mod.rs
├── tests/                     # Integration tests
│   ├── common/
│   │   └── mod.rs
│   └── user_tests.rs
└── benches/                   # Benchmarks
    └── user_benchmarks.rs
```

### lib.rs Structure

```rust
// lib-core/src/lib.rs

#![forbid(unsafe_code)]
#![warn(missing_docs)]

//! Core domain models and business logic for the application.
//!
//! This crate provides the Backend Model Controller (BMC) pattern
//! implementation for all entities.

// Public modules
pub mod ctx;
pub mod model;
pub mod store;

// Re-export commonly used types
pub use ctx::Ctx;
pub use model::ModelManager;
pub use store::Db;

// Internal utilities (only for tests)
#[cfg(test)]
pub mod _dev_utils;

// Prelude for convenient imports
pub mod prelude {
    pub use crate::ctx::Ctx;
    pub use crate::model::{ModelManager, Error, Result};
}
```

### Binary Crate (web-server)

```
web-server/
├── Cargo.toml
├── src/
│   ├── main.rs                # Entry point
│   ├── config.rs              # Configuration
│   ├── web/
│   │   ├── mod.rs
│   │   ├── routes_login.rs
│   │   ├── routes_user.rs
│   │   ├── routes_post.rs
│   │   ├── api_types.rs       # Request/Response DTOs
│   │   ├── error.rs           # HTTP error mapping
│   │   └── middleware/
│   │       ├── mod.rs
│   │       ├── mw_auth.rs
│   │       └── mw_logging.rs
│   └── utils.rs
├── examples/
│   └── quick_dev.rs           # Development testing
└── tests/
    └── api_tests.rs
```

### main.rs Structure

```rust
// web-server/src/main.rs

use axum::Router;
use lib_core::model::ModelManager;
use std::net::SocketAddr;
use tracing_subscriber::EnvFilter;

mod config;
mod web;

#[derive(Clone)]
struct AppState {
    mm: ModelManager,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Initialize tracing
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| EnvFilter::new("info"))
        )
        .init();

    // Load configuration
    let config = config::load()?;

    // Initialize model manager
    let mm = ModelManager::new().await?;

    // Create app state
    let state = AppState { mm };

    // Build router
    let app = Router::new()
        .merge(web::routes_login::routes(state.clone()))
        .merge(web::routes_user::routes(state.clone()))
        .merge(web::routes_post::routes(state))
        .layer(/* middleware */);

    // Start server
    let addr: SocketAddr = format!("{}:{}", config.host, config.port).parse()?;
    tracing::info!("Listening on {}", addr);

    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await?;

    Ok(())
}
```

---

## Module Naming Conventions

### File Naming

```
✅ GOOD
- user.rs              # Single module
- routes_user.rs       # Prefixed for grouping
- mw_auth.rs          # Prefixed middleware
- db_bmc.rs           # Prefixed trait

❌ BAD
- User.rs             # Not PascalCase
- user-routes.rs      # No hyphens
- auth_middleware.rs  # Too verbose (use mw_auth)
```

### Module Structure

```rust
// Flat structure for simple features
src/
├── lib.rs
├── user.rs
├── post.rs
└── error.rs

// Nested structure for complex features
src/
├── lib.rs
├── user/
│   ├── mod.rs        # Public interface
│   ├── types.rs      # Type definitions
│   ├── bmc.rs        # BMC implementation
│   └── tests.rs      # Unit tests
└── error.rs
```

### Visibility Guidelines

```rust
// lib.rs or mod.rs

// Public API (exported)
pub mod user;
pub mod post;

// Public but internal (doc(hidden))
#[doc(hidden)]
pub mod _dev_utils;

// Private (module-only)
mod internal;

// Re-exports for convenience
pub use user::{User, UserBmc, UserForCreate};
pub use post::{Post, PostBmc, PostForCreate};
```

---

## Feature Flags

### Defining Features

```toml
# lib-core/Cargo.toml

[features]
default = ["postgres"]

# Database backends
postgres = ["sqlx/postgres", "sea-query-binder/sqlx-postgres"]
mysql = ["sqlx/mysql", "sea-query-binder/sqlx-mysql"]
sqlite = ["sqlx/sqlite", "sea-query-binder/sqlx-sqlite"]

# Optional integrations
redis = ["dep:redis"]
s3 = ["dep:aws-sdk-s3"]

# Development features
with-test-utils = []
```

### Conditional Compilation

```rust
// Conditional struct fields
pub struct Config {
    pub database_url: String,

    #[cfg(feature = "redis")]
    pub redis_url: String,

    #[cfg(feature = "s3")]
    pub s3_bucket: String,
}

// Conditional implementations
#[cfg(feature = "redis")]
pub mod redis {
    pub async fn connect() -> Result<Connection> {
        // Redis-specific code
    }
}

// Conditional tests
#[cfg(all(test, feature = "with-test-utils"))]
mod tests {
    use super::*;

    #[test]
    fn test_with_utils() {
        // Test code
    }
}
```

---

## Dependency Management

### Workspace vs Crate Dependencies

```toml
# Workspace-level (Cargo.toml at root)
[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }

# Crate-level (crates/libs/lib-core/Cargo.toml)
[dependencies]
serde.workspace = true          # Use workspace version
tokio.workspace = true

# Crate-specific dependency
modql = "0.4"                   # Not in workspace

# Optional dependencies
redis = { version = "0.24", optional = true }
```

### Dependency Auditing

```toml
# .cargo/deny.toml

[advisories]
db-path = "~/.cargo/advisory-db"
db-urls = ["https://github.com/rustsec/advisory-db"]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"
notice = "warn"

[licenses]
unlicensed = "deny"
allow = [
    "MIT",
    "Apache-2.0",
    "BSD-3-Clause",
]
deny = [
    "GPL-3.0",
]

[bans]
multiple-versions = "warn"
wildcards = "deny"
```

### Using cargo-deny

```bash
# Install
cargo install cargo-deny

# Check dependencies
cargo deny check

# In CI
cargo deny check advisories
cargo deny check licenses
cargo deny check bans
```

---

## Build Configuration

### Custom Build Scripts

```rust
// build.rs

fn main() {
    // Set build-time environment variables
    println!("cargo:rerun-if-changed=migrations/");

    // Generate code at build time
    println!("cargo:rustc-env=BUILD_TIMESTAMP={}", chrono::Utc::now());

    // Conditional compilation
    if cfg!(feature = "postgres") {
        println!("cargo:rustc-cfg=database_postgres");
    }
}
```

### Cargo Configuration

```toml
# .cargo/config.toml

[build]
# Number of parallel jobs (0 = # of CPUs)
jobs = 4

# Incremental compilation for faster rebuilds
incremental = true

[target.x86_64-unknown-linux-gnu]
# Use lld linker for faster linking
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=lld"]

[target.x86_64-apple-darwin]
rustflags = ["-C", "link-arg=-fuse-ld=/usr/local/bin/zld"]

# Alias for common commands
[alias]
b = "build"
br = "build --release"
t = "test"
c = "check"
r = "run"
```

---

## Clippy Lints

### Workspace-Level Lints

```toml
# Cargo.toml (workspace root)

[workspace.lints.rust]
unsafe_code = "forbid"
missing_docs = "warn"
unused_import_braces = "warn"
unused_qualifications = "warn"

[workspace.lints.clippy]
# Groups
pedantic = { level = "warn", priority = -1 }
nursery = { level = "warn", priority = -1 }

# Specific denies (catch bugs)
unwrap_used = "deny"
expect_used = "deny"
panic = "deny"
todo = "warn"
unimplemented = "warn"

# Performance
large_enum_variant = "warn"
large_stack_arrays = "warn"

# Style
module_name_repetitions = "allow"  # Override pedantic
```

### Crate-Level Overrides

```rust
// lib.rs

#![forbid(unsafe_code)]
#![warn(missing_docs)]
#![warn(clippy::all)]
#![warn(clippy::pedantic)]
#![allow(clippy::module_name_repetitions)]

// Function-level allows
#[allow(clippy::similar_names)]
pub fn process_data(data_x: i32, data_y: i32) -> i32 {
    data_x + data_y
}
```

### Running Clippy

```bash
# Check all lints
cargo clippy --all-targets --all-features

# Fix automatically
cargo clippy --fix

# Deny warnings in CI
cargo clippy --all-targets --all-features -- -D warnings
```

---

## Documentation

### Crate-Level Documentation

```rust
// lib.rs

#![doc = include_str!("../README.md")]

//! # lib-core
//!
//! Core domain models and business logic.
//!
//! ## Features
//!
//! - **Backend Model Controller (BMC)** pattern
//! - Type-safe database operations
//! - Context-based authorization
//!
//! ## Example
//!
//! ```rust
//! use lib_core::prelude::*;
//!
//! # async fn example() -> Result<()> {
//! let mm = ModelManager::new().await?;
//! let ctx = Ctx::root_ctx();
//!
//! let user = UserBmc::get(&ctx, &mm, 1).await?;
//! # Ok(())
//! # }
//! ```
```

### Module Documentation

```rust
/// User management module.
///
/// Provides CRUD operations for users using the BMC pattern.
pub mod user {
    /// Represents a user in the system.
    ///
    /// # Fields
    ///
    /// - `id`: Unique identifier
    /// - `username`: Unique username
    /// - `email`: User's email address
    pub struct User {
        pub id: i64,
        pub username: String,
        pub email: String,
    }
}
```

### Generating Documentation

```bash
# Generate documentation
cargo doc --no-deps --open

# Include private items
cargo doc --no-deps --document-private-items

# Check documentation
cargo test --doc
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [architecture-overview.md](architecture-overview.md)
- [testing-guide.md](testing-guide.md)
- [deployment-guide.md](deployment-guide.md)
