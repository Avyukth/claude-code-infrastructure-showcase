---
name: rust-backend-guidelines
description: Comprehensive Rust backend development guide for production-grade web services. Covers Axum web framework, SQLx database access, error handling, async patterns, type safety, security best practices, testing strategies, and deployment. Emphasizes the Rust ownership model, zero-cost abstractions, and compile-time guarantees for building performant, safe, and maintainable backend systems.
---

# Rust Backend Development Guidelines

## Purpose

Establish modern best practices for building production-grade Rust backend services using Axum, SQLx, and the Rust10x architectural patterns. This guide emphasizes **type safety**, **performance**, **security**, and **maintainability** through Rust's unique features.

## When to Use This Skill

Automatically activates when working on:
- Creating or modifying Rust web services (Axum, Actix-Web, Rocket)
- Database operations with SQLx or SeaORM
- API design and routing
- Error handling and custom error types
- Async programming with Tokio
- Type-safe patterns and trait design
- Authentication and authorization
- Testing and benchmarking
- Backend refactoring and optimization

---

## Quick Start

### New Rust Backend Feature Checklist

- [ ] **Project Structure**: Multi-crate workspace with clear layer separation
- [ ] **Error Handling**: Custom Error enum with `derive_more::From`
- [ ] **Database**: SQLx with compile-time query verification
- [ ] **API Layer**: Axum routes with extractors and middleware
- [ ] **Business Logic**: Backend Model Controller (BMC) pattern
- [ ] **Type Safety**: Strong newtypes, no primitive obsession
- [ ] **Async Runtime**: Tokio with proper error propagation
- [ ] **Testing**: Unit + integration tests with test fixtures
- [ ] **Security**: Input validation, SQL injection prevention, authentication
- [ ] **Documentation**: Inline docs + API documentation

### New Microservice Checklist

- [ ] Workspace structure with separate library crates
- [ ] Shared error types and utilities (lib-utils)
- [ ] Domain layer with business logic (lib-core)
- [ ] Infrastructure layer for external services (lib-infrastructure)
- [ ] Web layer with Axum handlers (lib-web)
- [ ] Configuration management (lib-config)
- [ ] Observability setup (tracing, metrics)
- [ ] CI/CD pipeline with cargo test and clippy

---

## Architecture Overview

### Layered Architecture

```
HTTP Request
    ↓
Routes (Axum routing + middleware)
    ↓
Handlers (extract, validate, delegate)
    ↓
BMC/Services (business logic)
    ↓
Repositories (optional data access abstraction)
    ↓
Database (SQLx + PostgreSQL)
```

**Key Principle:** Each layer has ONE responsibility, enforced by Rust's module system.

See [architecture-overview.md](resources/architecture-overview.md) for complete details.

---

## Core Principles (10 Key Rules)

### 1. Type Safety First
```rust
// ❌ NEVER: Primitive obsession
fn create_user(id: i64, role: String) -> Result<()> { }

// ✅ ALWAYS: Strong newtypes
#[derive(Debug, Clone, Copy)]
pub struct UserId(i64);

#[derive(Debug, Clone)]
pub enum UserRole {
    Admin,
    User,
    Guest,
}

fn create_user(id: UserId, role: UserRole) -> Result<()> { }
```

### 2. Explicit Error Handling
```rust
// ❌ NEVER: unwrap() or expect() in production code
let user = db.get_user(id).unwrap();

// ✅ ALWAYS: ? operator with custom Error types
let user = db.get_user(id)?;
```

### 3. Leverage the Type System
```rust
// ✅ Use enums for state machines
pub enum ConnectionState {
    Connecting,
    Connected { session_id: String },
    Disconnected { reason: String },
}

// ✅ Use Result<T, E> for fallible operations
pub fn parse_config() -> Result<Config, ConfigError> { }
```

### 4. Async-First with Tokio
```rust
// ✅ All I/O operations should be async
#[tokio::main]
async fn main() -> Result<()> {
    let pool = PgPool::connect(&database_url).await?;
    let server = axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await?;
    Ok(())
}
```

### 5. Compile-Time SQL Verification
```rust
// ✅ SQLx compile-time checked queries
let user = sqlx::query_as!(
    User,
    "SELECT id, username, email FROM users WHERE id = $1",
    id
)
.fetch_one(&pool)
.await?;
```

### 6. Zero-Cost Abstractions
```rust
// ✅ Traits for polymorphism without runtime cost
#[async_trait]
pub trait UserRepository {
    async fn find_by_id(&self, id: UserId) -> Result<User>;
    async fn save(&self, user: &User) -> Result<()>;
}
```

### 7. Ownership for Resource Management
```rust
// ✅ RAII pattern for automatic cleanup
pub struct Transaction<'a> {
    tx: PgTransaction<'a>,
}

impl<'a> Drop for Transaction<'a> {
    fn drop(&mut self) {
        // Auto-rollback if not committed
    }
}
```

### 8. Fearless Concurrency
```rust
// ✅ Send + Sync traits ensure thread safety
async fn process_batch<T: Send + Sync>(items: Vec<T>) -> Result<()> {
    let handles: Vec<_> = items
        .into_iter()
        .map(|item| tokio::spawn(async move { process_item(item).await }))
        .collect();

    for handle in handles {
        handle.await??;
    }
    Ok(())
}
```

### 9. Minimal Dependencies, Maximum Control
```rust
// ✅ Prefer std library when possible
// ✅ Choose libraries with minimal dependency trees
// ✅ Audit dependencies regularly with cargo-audit
```

### 10. Test Everything
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_user_creation() {
        let pool = create_test_pool().await;
        let user = UserBmc::create(&ctx, &mm, user_data).await.unwrap();
        assert_eq!(user.username, "testuser");
    }
}
```

---

## Directory Structure

```
rust-backend/
├── Cargo.toml              # Workspace configuration
├── crates/
│   ├── libs/               # Reusable library crates
│   │   ├── lib-auth/       # Authentication & authorization
│   │   ├── lib-core/       # Domain models & business logic
│   │   │   ├── src/
│   │   │   │   ├── ctx/    # Request context
│   │   │   │   ├── model/  # BMC pattern models
│   │   │   │   └── store/  # Database abstractions
│   │   ├── lib-utils/      # Shared utilities
│   │   ├── lib-web/        # Web layer (Axum middleware)
│   │   └── lib-config/     # Configuration management
│   └── services/
│       └── web-server/     # Main application entry
│           ├── src/
│           │   ├── web/    # Handlers and routes
│           │   │   ├── routes_*.rs
│           │   │   └── middleware/
│           │   └── main.rs
├── migrations/             # Database migrations (SQLx)
├── tests/                  # Integration tests
└── benches/                # Performance benchmarks
```

**Naming Conventions:**
- Crates: `lib-name`, `kebab-case`
- Modules: `snake_case`
- Types: `PascalCase`
- Functions: `snake_case`
- Constants: `SCREAMING_SNAKE_CASE`

---

## Common Imports

```rust
// Web Framework
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::{IntoResponse, Json},
    routing::{get, post},
    Router,
};

// Async Runtime
use tokio::time::Duration;

// Database
use sqlx::{PgPool, postgres::PgRow, FromRow};

// Error Handling
use derive_more::From;

// Serialization
use serde::{Deserialize, Serialize};

// Tracing
use tracing::{debug, error, info, warn};

// Config
use config::Config;
```

---

## Quick Reference

### HTTP Status Codes

| Code | Use Case | Example |
|------|----------|---------|
| 200 | Success (GET, PUT) | User retrieved, Updated |
| 201 | Created (POST) | User created |
| 204 | No Content (DELETE) | User deleted |
| 400 | Bad Request | Invalid input data |
| 401 | Unauthorized | Missing/invalid auth token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Duplicate resource |
| 422 | Unprocessable Entity | Validation failed |
| 500 | Internal Server Error | Unexpected error |

### Rust-Specific HTTP Responses

```rust
// 200 - Success with JSON
(StatusCode::OK, Json(data))

// 201 - Created
(StatusCode::CREATED, Json(user))

// 404 - Not Found
(StatusCode::NOT_FOUND, Json(ErrorResponse { message: "User not found" }))

// Custom error mapping
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound => (StatusCode::NOT_FOUND, self.to_string()),
            AppError::Unauthorized => (StatusCode::UNAUTHORIZED, self.to_string()),
            _ => (StatusCode::INTERNAL_SERVER_ERROR, "Internal error".to_string()),
        };

        (status, Json(json!({ "error": message }))).into_response()
    }
}
```

---

## Rust10x Patterns

### Backend Model Controller (BMC)
```rust
pub struct UserBmc;

impl UserBmc {
    pub async fn create(ctx: &Ctx, mm: &ModelManager, data: UserForCreate) -> Result<i64> {
        // Create user with context
    }

    pub async fn get(ctx: &Ctx, mm: &ModelManager, id: i64) -> Result<User> {
        // Get user by ID
    }

    pub async fn list(ctx: &Ctx, mm: &ModelManager) -> Result<Vec<User>> {
        // List all users
    }
}
```

**Benefits:**
- Explicit context passing for future ACL
- Stateless pattern
- Consistent API across entities
- Easy to test with mock ModelManager

---

## Anti-Patterns to Avoid

❌ Using `.unwrap()` or `.expect()` in production code
❌ Primitive obsession (using i64, String everywhere)
❌ Blocking operations in async code
❌ Direct SQL strings without parameterization
❌ Ignoring compiler warnings
❌ Not using `#[must_use]` on Result types
❌ Overly complex generic bounds
❌ Clone without understanding ownership

---

## Navigation Guide

| Need to... | Read this |
|------------|-----------|
| Understand architecture | [architecture-overview.md](resources/architecture-overview.md) |
| Handle errors properly | [error-handling.md](resources/error-handling.md) |
| Work with async/await | [async-patterns.md](resources/async-patterns.md) |
| Database operations | [database-sqlx.md](resources/database-sqlx.md) |
| Design APIs | [api-design.md](resources/api-design.md) |
| Build GraphQL APIs | [graphql-async-graphql.md](resources/graphql-async-graphql.md) |
| Implement gRPC services | [grpc-tonic.md](resources/grpc-tonic.md) |
| Use message queues | [message-queues.md](resources/message-queues.md) |
| Implement caching | [caching-patterns.md](resources/caching-patterns.md) |
| Write tests | [testing-guide.md](resources/testing-guide.md) |
| Implement security | [security-patterns.md](resources/security-patterns.md) |
| Optimize performance | [performance-optimization.md](resources/performance-optimization.md) |
| Structure projects | [project-structure.md](resources/project-structure.md) |
| Deploy services | [deployment-guide.md](resources/deployment-guide.md) |
| See complete examples | [complete-examples.md](resources/complete-examples.md) |

---

## Resource Files

### [architecture-overview.md](resources/architecture-overview.md)
Multi-crate workspace architecture, layered pattern, BMC pattern, hexagonal architecture

### [error-handling.md](resources/error-handling.md)
Custom error types, derive_more, error propagation, context preservation

### [async-patterns.md](resources/async-patterns.md)
Tokio runtime, async/await best practices, concurrent patterns, spawn blocking

### [database-sqlx.md](resources/database-sqlx.md)
SQLx setup, compile-time verification, migrations, connection pooling, Sea-Query

### [api-design.md](resources/api-design.md)
Axum routing, extractors, middleware, JSON-RPC vs REST, versioning strategies

### [testing-guide.md](resources/testing-guide.md)
Unit tests, integration tests, test fixtures, mocking, property-based testing

### [security-patterns.md](resources/security-patterns.md)
Authentication (JWT, tokens), authorization, password hashing, SQL injection prevention

### [performance-optimization.md](resources/performance-optimization.md)
Profiling, benchmarking, zero-copy patterns, memory optimization, caching

### [project-structure.md](resources/project-structure.md)
Workspace organization, module structure, feature flags, dependency management

### [deployment-guide.md](resources/deployment-guide.md)
Docker, CI/CD, monitoring, logging, blue-green deployment, release profiles

### [complete-examples.md](resources/complete-examples.md)
Full CRUD API, authentication flow, real-world patterns

### [graphql-async-graphql.md](resources/graphql-async-graphql.md)
GraphQL schema definition, queries, mutations, subscriptions, DataLoader, BMC integration

### [grpc-tonic.md](resources/grpc-tonic.md)
Protocol Buffers, gRPC services, streaming (unary, server, client, bidirectional), interceptors

### [message-queues.md](resources/message-queues.md)
RabbitMQ (lapin), Apache Kafka (rdkafka), event-driven architecture, async processing

### [caching-patterns.md](resources/caching-patterns.md)
In-memory caching (Moka), distributed caching (Redis), cache strategies, multi-level caching

---

## Related Skills

- **frontend-dev-guidelines** - SvelteKit/React frontend patterns
- **database-verification** - Schema verification and consistency
- **deployment-automation** - CI/CD pipeline patterns

---

**Skill Status**: PRODUCTION-READY ✅
**Line Count**: < 500 ✅
**Progressive Disclosure**: 15 resource files ✅
**Rust-Specific**: Ownership, type safety, zero-cost abstractions ✅
