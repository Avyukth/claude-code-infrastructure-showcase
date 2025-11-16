# Architecture Overview - Rust Backend Services

Complete guide to the layered architecture pattern and multi-crate workspace structure for Rust backend services.

## Table of Contents

- [Multi-Crate Workspace Architecture](#multi-crate-workspace-architecture)
- [Layered Architecture Pattern](#layered-architecture-pattern)
- [Backend Model Controller (BMC) Pattern](#backend-model-controller-bmc-pattern)
- [Request Lifecycle](#request-lifecycle)
- [Module Organization](#module-organization)
- [Dependency Flow](#dependency-flow)

---

## Multi-Crate Workspace Architecture

### Why Multi-Crate?

**Decision**: Organize code into a Cargo workspace with separate crates for different concerns.

**Benefits:**
- **Modularity**: Clear boundaries prevent tight coupling
- **Reusability**: Libraries can be used by multiple services
- **Compilation Speed**: Incremental compilation only rebuilds changed crates
- **Dependency Management**: Prevents circular dependencies through crate graph
- **Team Scalability**: Teams can own specific crates
- **Testing Isolation**: Each crate can have its own test suite

### Workspace Structure

```
rust-web-app/
├── Cargo.toml              # Workspace configuration
├── crates/
│   ├── libs/               # Reusable library crates
│   │   ├── lib-auth/       # Authentication primitives
│   │   │   ├── src/
│   │   │   │   ├── pwd/    # Password hashing (multi-scheme)
│   │   │   │   └── token/  # Token generation/validation
│   │   │   └── Cargo.toml
│   │   ├── lib-core/       # Domain models & business logic
│   │   │   ├── src/
│   │   │   │   ├── ctx/    # Request context (Ctx)
│   │   │   │   ├── model/  # BMC pattern (UserBmc, AgentBmc)
│   │   │   │   │   ├── user.rs
│   │   │   │   │   ├── agent.rs
│   │   │   │   │   └── base/  # Shared CRUD functions
│   │   │   │   └── store/  # Database abstraction (Dbx)
│   │   │   └── Cargo.toml
│   │   ├── lib-rpc-core/   # RPC abstractions
│   │   │   └── src/
│   │   │       └── router/ # JSON-RPC routing
│   │   ├── lib-utils/      # Shared utilities
│   │   │   └── src/
│   │   │       ├── time.rs # Time utilities
│   │   │       └── b64.rs  # Base64 utilities
│   │   └── lib-web/        # Web layer (Axum)
│   │       ├── src/
│   │       │   └── middleware/
│   │       │       ├── mw_auth.rs      # Authentication
│   │       │       ├── mw_req_stamp.rs # Request UUID
│   │       │       └── mw_res_map.rs   # Error mapping
│   │       └── Cargo.toml
│   ├── services/
│   │   └── web-server/     # Application entry point
│   │       ├── src/
│   │       │   ├── main.rs
│   │       │   └── web/
│   │       │       ├── routes_login.rs
│   │       │       └── rpcs/  # RPC handlers
│   │       └── Cargo.toml
│   └── tools/
│       └── gen-key/        # Development utilities
```

### Workspace Cargo.toml

```toml
[workspace]
resolver = "2"
members = [
    "crates/libs/lib-auth",
    "crates/libs/lib-core",
    "crates/libs/lib-rpc-core",
    "crates/libs/lib-utils",
    "crates/libs/lib-web",
    "crates/services/web-server",
    "crates/tools/gen-key",
]

[workspace.dependencies]
# Web Framework
axum = "0.8"
tower = "0.5"
tower-cookies = "0.11"

# Async Runtime
tokio = { version = "1", features = ["full"] }

# Database
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres"] }
sea-query = "0.32"
sea-query-binder = { version = "0.7", features = ["sqlx-postgres"] }

# Serialization
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Error Handling
derive_more = { version = "1.0.0", features = ["from", "display"] }

# Tracing
tracing = "0.1"
tracing-subscriber = "0.3"

[workspace.lints.rust]
unsafe_code = "forbid"

[workspace.lints.clippy]
# Deny common mistakes
unwrap_used = "warn"
expect_used = "warn"
panic = "warn"
```

**Key Points:**
- `resolver = "2"` for modern dependency resolution
- Shared dependencies in `[workspace.dependencies]`
- Workspace-level lints for consistency
- `unsafe_code = "forbid"` for memory safety guarantee

---

## Layered Architecture Pattern

### Four-Layer Architecture

```
┌─────────────────────────────────────┐
│   Web Layer (lib-web)               │  HTTP concerns, middleware
│   - Axum routes and handlers        │
│   - Request/Response mapping        │
│   - Middleware (auth, logging)      │
├─────────────────────────────────────┤
│   RPC Layer (lib-rpc-core)          │  JSON-RPC routing & validation
│   - RPC router                      │  (optional - can use REST)
│   - Method dispatch                 │
│   - Request validation              │
├─────────────────────────────────────┤
│   Domain Layer (lib-core)           │  Business logic, BMC pattern
│   - Backend Model Controllers       │
│   - Context (Ctx) management        │
│   - Business rules                  │
├─────────────────────────────────────┤
│   Infrastructure (lib-core/store)   │  Database, transactions
│   - ModelManager                    │
│   - Database abstraction (Dbx)      │
│   - Connection pooling              │
└─────────────────────────────────────┘
```

**Dependency Direction**: Each layer only depends on layers below it.

### Layer Responsibilities

**Web Layer (lib-web)**:
- ✅ HTTP request/response handling
- ✅ Middleware composition (auth, logging, CORS)
- ✅ Cookie management
- ✅ Request stamping (UUID, timestamp)
- ❌ Business logic
- ❌ Database operations

**RPC Layer (lib-rpc-core)** (Optional):
- ✅ JSON-RPC method routing
- ✅ Request deserialization
- ✅ Response serialization
- ❌ Business logic
- ❌ Database access

**Domain Layer (lib-core)**:
- ✅ Business logic and rules
- ✅ Entity management (BMC pattern)
- ✅ Context propagation
- ✅ Domain-specific validation
- ❌ HTTP concerns
- ❌ Direct database calls (use abstractions)

**Infrastructure Layer (lib-core/store)**:
- ✅ Database connection management
- ✅ Transaction handling
- ✅ Query execution
- ✅ Connection pooling
- ❌ Business logic
- ❌ HTTP concerns

---

## Backend Model Controller (BMC) Pattern

### What is BMC?

The Backend Model Controller pattern is an alternative to traditional Repository or Active Record patterns, designed for Rust's ownership model and async ecosystem.

**Key Characteristics:**
- **Zero-Sized Types**: `struct UserBmc;` has no fields
- **Explicit Context**: Every operation requires `&Ctx` for future ACL
- **Stateless**: Pure functions, no instance state
- **Type-Safe**: Strong typing for create/update variants
- **Consistent API**: Standard CRUD interface across entities

### BMC Structure

```rust
// crates/libs/lib-core/src/model/user.rs

use crate::ctx::Ctx;
use crate::model::base::{self, DbBmc};
use crate::model::{Error, Result};
use crate::model::ModelManager;
use modql::field::Fields;
use modql::filter::FilterNodes;
use sea_query::{Iden, PostgresQueryBuilder};
use sea_query_binder::SqlxBinder;
use serde::{Deserialize, Serialize};
use sqlx::FromRow;

// -- Types

#[derive(Debug, Clone, Fields, FromRow, Serialize)]
pub struct User {
    pub id: i64,
    pub username: String,

    // Timestamps
    pub cid: i64,      // Creator user ID
    pub ctime: time::OffsetDateTime,
    pub mid: i64,      // Modifier user ID
    pub mtime: time::OffsetDateTime,
}

#[derive(Fields, Deserialize)]
pub struct UserForCreate {
    pub username: String,
}

#[derive(Fields, Deserialize)]
pub struct UserForUpdate {
    pub username: Option<String>,
}

// -- BMC

pub struct UserBmc;

impl DbBmc for UserBmc {
    const TABLE: &'static str = "user";
}

impl UserBmc {
    pub async fn create(
        ctx: &Ctx,
        mm: &ModelManager,
        user_c: UserForCreate,
    ) -> Result<i64> {
        base::create::<Self, _>(ctx, mm, user_c).await
    }

    pub async fn get(
        ctx: &Ctx,
        mm: &ModelManager,
        id: i64,
    ) -> Result<User> {
        base::get::<Self, _>(ctx, mm, id).await
    }

    pub async fn list(
        ctx: &Ctx,
        mm: &ModelManager,
        filters: Option<Vec<FilterNodes>>,
        list_options: Option<ListOptions>,
    ) -> Result<Vec<User>> {
        base::list::<Self, _>(ctx, mm, filters, list_options).await
    }

    pub async fn update(
        ctx: &Ctx,
        mm: &ModelManager,
        id: i64,
        user_u: UserForUpdate,
    ) -> Result<()> {
        base::update::<Self, _>(ctx, mm, id, user_u).await
    }

    pub async fn delete(
        ctx: &Ctx,
        mm: &ModelManager,
        id: i64,
    ) -> Result<()> {
        base::delete::<Self>(ctx, mm, id).await
    }
}
```

### Context (Ctx) Pattern

```rust
// crates/libs/lib-core/src/ctx/mod.rs

#[derive(Clone, Debug)]
pub struct Ctx {
    user_id: i64,
}

impl Ctx {
    pub fn root_ctx() -> Self {
        Ctx { user_id: 0 }
    }

    pub fn new(user_id: i64) -> Result<Self> {
        if user_id == 0 {
            Err(Error::CtxCannotNewRootCtx)
        } else {
            Ok(Self { user_id })
        }
    }

    pub fn user_id(&self) -> i64 {
        self.user_id
    }
}
```

**Benefits of Ctx:**
- **Explicit**: Every operation shows who is performing it
- **Future-Proof**: Easy to add permissions/roles later
- **Audit Trail**: Natural place to track who did what
- **Type-Safe**: Can't forget to pass context

---

## Request Lifecycle

### Complete Flow Example

```
1. HTTP POST /api/rpc
   ↓
2. Axum matches route
   ↓
3. Middleware chain (outside → inside):
   - mw_req_stamp_resolver (add UUID + timestamp)
   - CookieManager (parse cookies)
   - mw_ctx_resolver (resolve user context from token)
   - mw_response_map (transform errors to JSON)
   ↓
4. RPC Router dispatches to handler:
   rpc_handler::create_agent(ctx, mm, params)
   ↓
5. Handler extracts and validates:
   - Extract Ctx from request extensions
   - Deserialize params
   - Call AgentBmc::create()
   ↓
6. BMC executes business logic:
   - Validate data
   - Build SQL query (Sea-Query)
   - Execute via ModelManager.dbx()
   - Return entity ID
   ↓
7. Response flows back:
   BMC → Handler → RPC Router → Middleware → Client
```

### Middleware Execution Order

**Critical:** Middleware executes in registration order (Tower layer stack).

```rust
// crates/services/web-server/src/main.rs

use tower_cookies::CookieManagerLayer;
use lib_web::middleware::{
    mw_req_stamp_resolver,
    mw_ctx_resolver,
    mw_response_map,
};

let routes_all = Router::new()
    .merge(routes_login::routes(mm.clone()))
    .nest("/api", routes_rpc)
    .layer(middleware::map_response(mw_response_map))  // 4. Error mapping (LAST)
    .layer(middleware::from_fn_with_state(mm.clone(), mw_ctx_resolver))  // 3. Context
    .layer(CookieManagerLayer::new())  // 2. Cookies
    .layer(middleware::map_request(mw_req_stamp_resolver));  // 1. UUID (FIRST)
```

**Execution Order** (request → response):
```
Request → 1 → 2 → 3 → 4 → Handler → 4 → 3 → 2 → 1 → Response
```

**Why This Order?**
1. **Request Stamping First**: Every request gets UUID for tracing
2. **Cookie Parsing Early**: Required by auth layer
3. **Context Resolution**: Creates `Ctx` from token, optional for some routes
4. **Error Mapping Last**: Catches all errors, transforms to client-safe JSON

---

## Module Organization

### Flat vs. Nested Modules

**Flat Organization** (simple features):
```
lib-core/src/
├── ctx.rs
├── model/
│   ├── mod.rs
│   ├── user.rs
│   ├── agent.rs
│   └── base/
│       ├── mod.rs
│       └── crud_fns.rs
└── store/
    └── mod.rs
```

**Nested Organization** (complex features):
```
lib-core/src/
├── ctx/
│   ├── mod.rs
│   └── error.rs
├── model/
│   ├── mod.rs
│   ├── user/
│   │   ├── mod.rs
│   │   ├── types.rs
│   │   ├── bmc.rs
│   │   └── tests.rs
│   ├── agent/
│   │   ├── mod.rs
│   │   └── bmc.rs
│   └── base/
│       ├── mod.rs
│       ├── crud_fns.rs
│       └── db_bmc.rs
└── store/
    ├── mod.rs
    └── dbx/
        ├── mod.rs
        └── txn.rs
```

**When to Nest:**
- Feature has 5+ files
- Clear sub-modules exist
- Improves discoverability

---

## Dependency Flow

### Allowed Dependencies

```
lib-web       →  lib-core, lib-auth, lib-utils
lib-rpc-core  →  lib-core, lib-utils
lib-core      →  lib-utils
lib-auth      →  lib-utils
lib-utils     →  (no internal dependencies)
```

### Cargo.toml Example

```toml
# crates/services/web-server/Cargo.toml

[package]
name = "web-server"
version = "0.1.0"
edition = "2021"

[dependencies]
# Internal dependencies
lib-auth.workspace = true
lib-core.workspace = true
lib-rpc-core.workspace = true
lib-utils.workspace = true
lib-web.workspace = true

# External dependencies
axum.workspace = true
tokio.workspace = true
tower.workspace = true
tower-cookies.workspace = true
serde.workspace = true
serde_json.workspace = true
tracing.workspace = true
tracing-subscriber.workspace = true
```

**Benefits:**
- `workspace = true` uses workspace-defined versions
- Easy to update all services at once
- Consistent dependencies across crates

---

## Strengths of This Architecture

### 1. Type Safety Excellence
- **Compile-Time Guarantees**: SQL queries checked at compile time
- **Strong Newtypes**: Prevent primitive obsession
- **Exhaustive Enums**: Pattern matching catches all cases

**Impact**: Bugs caught at compile time, not production.

### 2. Modularity
- **Clear Boundaries**: Crate boundaries enforce separation
- **No Circular Dependencies**: Enforced by Cargo
- **Reusable Components**: Libraries can be extracted

**Impact**: Easy to maintain, test, and extend.

### 3. Performance
- **Zero-Cost Abstractions**: Trait objects compiled away
- **Async I/O**: Tokio runtime for concurrent requests
- **Efficient Memory Usage**: Ownership system prevents allocations

**Impact**: Production-grade performance out of the box.

### 4. Testing
- **Unit Tests**: Each BMC independently testable
- **Integration Tests**: Test entire request flow
- **Mock-Friendly**: Traits enable easy mocking

**Impact**: High confidence in correctness.

---

## Comparison to TypeScript/Node.js Architecture

| Aspect | Rust (this pattern) | TypeScript/Node.js |
|--------|---------------------|-------------------|
| **Type Safety** | Compile-time guarantees | Runtime type checks (optional) |
| **Error Handling** | Result<T, E> enforced | try/catch (easily forgotten) |
| **Concurrency** | Fearless (Send/Sync) | Single-threaded event loop |
| **Performance** | Native speed | V8 JIT overhead |
| **Memory Safety** | Guaranteed by compiler | GC + potential memory leaks |
| **Null Safety** | Option<T> explicit | undefined/null bugs common |
| **Dependency Mgmt** | Cargo with versioning | npm/yarn (can have conflicts) |
| **Compilation** | Slower, catches errors | Fast, errors at runtime |

**When to Choose Rust:**
- ✅ Performance-critical services
- ✅ Long-running processes
- ✅ Resource-constrained environments
- ✅ High-security requirements

**When to Choose TypeScript:**
- ✅ Rapid prototyping
- ✅ Large existing npm ecosystem needed
- ✅ Frontend/backend code sharing
- ✅ Team familiar with JS/TS

---

**Related Files:**
- [SKILL.md](../SKILL.md) - Main guide
- [error-handling.md](error-handling.md) - Error handling patterns
- [database-sqlx.md](database-sqlx.md) - Database layer details
- [api-design.md](api-design.md) - Web layer and routing
