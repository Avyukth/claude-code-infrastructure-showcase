# GraphQL with async-graphql - Rust Backend Patterns

Complete guide to building type-safe GraphQL APIs in Rust using async-graphql with Rust10x patterns.

## Table of Contents

- [Why GraphQL in Rust](#why-graphql-in-rust)
- [async-graphql Overview](#async-graphql-overview)
- [Schema Definition](#schema-definition)
- [Queries and Mutations](#queries-and-mutations)
- [Subscriptions (WebSocket)](#subscriptions-websocket)
- [Integration with Axum](#integration-with-axum)
- [BMC Pattern Integration](#bmc-pattern-integration)
- [DataLoader for N+1 Prevention](#dataloader-for-n1-prevention)
- [Authentication and Authorization](#authentication-and-authorization)
- [Error Handling](#error-handling)
- [Testing GraphQL APIs](#testing-graphql-apis)
- [Complete Example](#complete-example)

---

## Why GraphQL in Rust

**Advantages of async-graphql:**
- ✅ **Compile-time type safety** - Schema and resolvers checked at compile time
- ✅ **Performance** - Zero-cost abstractions, native async/await
- ✅ **Code generation** - Automatic schema generation from Rust types
- ✅ **Feature-complete** - Queries, mutations, subscriptions, federation
- ✅ **Integration** - Works seamlessly with Axum, Actix-Web, Warp

**vs REST:**
- Client specifies exactly what data it needs
- Single endpoint reduces over-fetching
- Strong typing across client and server
- Built-in introspection and documentation

---

## async-graphql Overview

### Dependencies

```toml
[dependencies]
# GraphQL
async-graphql = "7.0"
async-graphql-axum = "7.0"

# Web framework
axum = "0.8"
tower = "0.5"
tower-http = { version = "0.6", features = ["cors"] }

# Database
sqlx = { version = "0.8", features = ["runtime-tokio", "postgres"] }

# Async runtime
tokio = { version = "1", features = ["full"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }

# Error handling
derive_more = { version = "1.0", features = ["from", "display"] }
```

### Key Concepts

**Schema First (Generated):**
```rust
use async_graphql::*;

// Types automatically become GraphQL types
#[derive(SimpleObject)]
struct User {
    id: ID,
    username: String,
    email: String,
}

// Queries
struct QueryRoot;

#[Object]
impl QueryRoot {
    async fn user(&self, id: ID) -> Result<User> {
        // Resolver logic
    }
}

// Build schema
let schema = Schema::build(QueryRoot, MutationRoot, SubscriptionRoot)
    .finish();
```

---

## Schema Definition

### Basic Types

```rust
use async_graphql::*;
use chrono::{DateTime, Utc};

// Simple Object (all fields exposed)
#[derive(SimpleObject)]
#[graphql(name = "User")]
pub struct UserGql {
    pub id: ID,
    pub username: String,
    pub email: String,
    pub created_at: DateTime<Utc>,
}

// Complex Object (custom resolvers)
#[derive(Clone)]
pub struct User {
    pub id: i64,
    pub username: String,
    pub email: String,
}

#[Object]
impl User {
    async fn id(&self) -> ID {
        ID(self.id.to_string())
    }

    async fn username(&self) -> &str {
        &self.username
    }

    async fn email(&self) -> &str {
        &self.email
    }

    // Computed field with context
    async fn posts(&self, ctx: &Context<'_>) -> Result<Vec<Post>> {
        let mm = ctx.data::<ModelManager>()?;
        PostBmc::list_by_user(&mm, self.id).await
    }
}
```

### Input Types

```rust
#[derive(InputObject)]
pub struct CreateUserInput {
    pub username: String,
    pub email: String,
    pub password: String,
}

#[derive(InputObject)]
pub struct UpdateUserInput {
    pub username: Option<String>,
    pub email: Option<String>,
}

#[derive(InputObject)]
pub struct UserFilter {
    pub username_contains: Option<String>,
    pub email_contains: Option<String>,
    pub created_after: Option<DateTime<Utc>>,
}
```

### Enums

```rust
#[derive(Enum, Copy, Clone, Eq, PartialEq)]
pub enum UserRole {
    Admin,
    User,
    Guest,
}

#[derive(Enum, Copy, Clone, Eq, PartialEq)]
pub enum SortOrder {
    Asc,
    Desc,
}
```

### Interfaces and Unions

```rust
#[derive(Interface)]
#[graphql(field(name = "id", type = "ID"))]
pub enum Node {
    User(User),
    Post(Post),
}

#[derive(Union)]
pub enum SearchResult {
    User(User),
    Post(Post),
    Comment(Comment),
}
```

---

## Queries and Mutations

### Query Root

```rust
use async_graphql::*;
use crate::ctx::Ctx;
use crate::model::{ModelManager, UserBmc, PostBmc};

pub struct QueryRoot;

#[Object]
impl QueryRoot {
    /// Get user by ID
    async fn user(&self, ctx: &Context<'_>, id: ID) -> Result<User> {
        let request_ctx = ctx.data::<Ctx>()?;
        let mm = ctx.data::<ModelManager>()?;

        let user_id = id.parse::<i64>()
            .map_err(|_| Error::new("Invalid user ID"))?;

        let user = UserBmc::get(request_ctx, mm, user_id).await?;
        Ok(user.into()) // Convert domain model to GraphQL type
    }

    /// List users with filtering and pagination
    async fn users(
        &self,
        ctx: &Context<'_>,
        filter: Option<UserFilter>,
        limit: Option<i32>,
        offset: Option<i32>,
    ) -> Result<Vec<User>> {
        let request_ctx = ctx.data::<Ctx>()?;
        let mm = ctx.data::<ModelManager>()?;

        let users = UserBmc::list_filtered(
            request_ctx,
            mm,
            filter,
            limit.unwrap_or(20),
            offset.unwrap_or(0),
        ).await?;

        Ok(users.into_iter().map(Into::into).collect())
    }

    /// Search across multiple types
    async fn search(&self, ctx: &Context<'_>, query: String) -> Result<Vec<SearchResult>> {
        let mm = ctx.data::<ModelManager>()?;

        let mut results = Vec::new();

        // Search users
        let users = UserBmc::search(mm, &query).await?;
        results.extend(users.into_iter().map(SearchResult::User));

        // Search posts
        let posts = PostBmc::search(mm, &query).await?;
        results.extend(posts.into_iter().map(SearchResult::Post));

        Ok(results)
    }
}
```

### Mutation Root

```rust
pub struct MutationRoot;

#[Object]
impl MutationRoot {
    /// Create a new user
    async fn create_user(
        &self,
        ctx: &Context<'_>,
        input: CreateUserInput,
    ) -> Result<User> {
        let request_ctx = ctx.data::<Ctx>()?;
        let mm = ctx.data::<ModelManager>()?;

        let user_id = UserBmc::create(
            request_ctx,
            mm,
            UserForCreate {
                username: input.username,
                email: input.email,
                password_clear: input.password,
            },
        ).await?;

        let user = UserBmc::get(request_ctx, mm, user_id).await?;
        Ok(user.into())
    }

    /// Update user
    async fn update_user(
        &self,
        ctx: &Context<'_>,
        id: ID,
        input: UpdateUserInput,
    ) -> Result<User> {
        let request_ctx = ctx.data::<Ctx>()?;
        let mm = ctx.data::<ModelManager>()?;

        let user_id = id.parse::<i64>()?;

        UserBmc::update(
            request_ctx,
            mm,
            user_id,
            UserForUpdate {
                username: input.username,
                email: input.email,
            },
        ).await?;

        let user = UserBmc::get(request_ctx, mm, user_id).await?;
        Ok(user.into())
    }

    /// Delete user
    async fn delete_user(&self, ctx: &Context<'_>, id: ID) -> Result<bool> {
        let request_ctx = ctx.data::<Ctx>()?;
        let mm = ctx.data::<ModelManager>()?;

        let user_id = id.parse::<i64>()?;
        UserBmc::delete(request_ctx, mm, user_id).await?;

        Ok(true)
    }
}
```

---

## Subscriptions (WebSocket)

### Subscription Root

```rust
use async_graphql::*;
use futures_util::{Stream, StreamExt};
use tokio_stream::wrappers::BroadcastStream;

pub struct SubscriptionRoot;

#[Subscription]
impl SubscriptionRoot {
    /// Subscribe to user updates
    async fn user_updated(
        &self,
        ctx: &Context<'_>,
        user_id: ID,
    ) -> Result<impl Stream<Item = User>> {
        let mm = ctx.data::<ModelManager>()?;
        let receiver = mm.subscribe_user_updates(user_id.parse()?);

        Ok(BroadcastStream::new(receiver).filter_map(|result| async move {
            result.ok()
        }))
    }

    /// Subscribe to new posts
    async fn post_created(&self, ctx: &Context<'_>) -> Result<impl Stream<Item = Post>> {
        let mm = ctx.data::<ModelManager>()?;
        let receiver = mm.subscribe_post_created();

        Ok(BroadcastStream::new(receiver).filter_map(|result| async move {
            result.ok()
        }))
    }

    /// Interval-based subscription
    async fn time_tick(&self) -> impl Stream<Item = DateTime<Utc>> {
        let mut interval = tokio::time::interval(std::time::Duration::from_secs(1));

        async_stream::stream! {
            loop {
                interval.tick().await;
                yield Utc::now();
            }
        }
    }
}
```

---

## Integration with Axum

### Axum Router Setup

```rust
use axum::{
    Router,
    routing::get,
    extract::State,
    response::IntoResponse,
};
use async_graphql::{
    http::{GraphQLPlaygroundConfig, playground_source},
    Schema,
};
use async_graphql_axum::{GraphQLRequest, GraphQLResponse, GraphQLSubscription};
use tower_http::cors::{CorsLayer, Any};

// Schema type alias
type AppSchema = Schema<QueryRoot, MutationRoot, SubscriptionRoot>;

// GraphQL handler
async fn graphql_handler(
    State(schema): State<AppSchema>,
    req: GraphQLRequest,
) -> GraphQLResponse {
    schema.execute(req.into_inner()).await.into()
}

// Playground handler
async fn graphql_playground() -> impl IntoResponse {
    axum::response::Html(playground_source(
        GraphQLPlaygroundConfig::new("/graphql").subscription_endpoint("/ws"),
    ))
}

// Build router
pub fn graphql_routes(schema: AppSchema) -> Router {
    Router::new()
        .route("/graphql", get(graphql_playground).post(graphql_handler))
        .route("/ws", get(GraphQLSubscription::new(schema.clone())))
        .layer(
            CorsLayer::new()
                .allow_origin(Any)
                .allow_headers(Any)
                .allow_methods(Any),
        )
        .with_state(schema)
}
```

### Main Application

```rust
use axum::Router;
use std::net::SocketAddr;

#[tokio::main]
async fn main() -> Result<()> {
    // Initialize database
    let mm = ModelManager::new().await?;

    // Build GraphQL schema
    let schema = Schema::build(QueryRoot, MutationRoot, SubscriptionRoot)
        .data(mm.clone())
        .finish();

    // Build router
    let app = Router::new()
        .merge(graphql_routes(schema));

    // Start server
    let addr = SocketAddr::from(([0, 0, 0, 0], 8080));
    println!("🚀 GraphQL server ready at http://{}/graphql", addr);

    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await?;

    Ok(())
}
```

---

## BMC Pattern Integration

### Domain Model to GraphQL Conversion

```rust
// Domain model (in lib-core)
#[derive(Clone, Debug, FromRow)]
pub struct User {
    pub id: i64,
    pub username: String,
    pub email: String,
    pub pwd: String, // Hashed password - never expose!
    pub pwd_salt: Uuid,
    pub token_salt: Uuid,
}

// GraphQL type (in lib-web or service)
#[derive(Clone)]
pub struct UserGql {
    pub id: i64,
    pub username: String,
    pub email: String,
}

#[Object]
impl UserGql {
    async fn id(&self) -> ID {
        ID(self.id.to_string())
    }

    async fn username(&self) -> &str {
        &self.username
    }

    async fn email(&self) -> &str {
        &self.email
    }

    // Relation: posts
    async fn posts(&self, ctx: &Context<'_>) -> Result<Vec<PostGql>> {
        let mm = ctx.data::<ModelManager>()?;
        let posts = PostBmc::list_by_user(mm, self.id).await?;
        Ok(posts.into_iter().map(Into::into).collect())
    }
}

// Conversion from domain to GraphQL
impl From<User> for UserGql {
    fn from(user: User) -> Self {
        Self {
            id: user.id,
            username: user.username,
            email: user.email,
            // pwd fields intentionally excluded
        }
    }
}
```

### Using BMC in Resolvers

```rust
#[Object]
impl QueryRoot {
    async fn user(&self, ctx: &Context<'_>, id: ID) -> Result<UserGql> {
        // Extract context
        let request_ctx = ctx.data::<Ctx>()?;
        let mm = ctx.data::<ModelManager>()?;

        // Parse ID
        let user_id = id.parse::<i64>()
            .map_err(|_| Error::new("Invalid user ID"))?;

        // Use BMC pattern
        let user = UserBmc::get(request_ctx, mm, user_id).await?;

        // Convert to GraphQL type
        Ok(user.into())
    }
}
```

---

## DataLoader for N+1 Prevention

### DataLoader Implementation

```rust
use async_graphql::dataloader::*;
use std::collections::HashMap;

// User loader
pub struct UserLoader {
    mm: ModelManager,
}

#[async_trait::async_trait]
impl Loader<i64> for UserLoader {
    type Value = User;
    type Error = Error;

    async fn load(&self, keys: &[i64]) -> Result<HashMap<i64, User>, Self::Error> {
        let users = UserBmc::get_many(&self.mm, keys).await?;

        Ok(users.into_iter().map(|user| (user.id, user)).collect())
    }
}

// Post loader
pub struct PostLoader {
    mm: ModelManager,
}

#[async_trait::async_trait]
impl Loader<i64> for PostLoader {
    type Value = Vec<Post>;
    type Error = Error;

    async fn load(&self, user_ids: &[i64]) -> Result<HashMap<i64, Vec<Post>>, Self::Error> {
        let posts = PostBmc::get_by_users(&self.mm, user_ids).await?;

        let mut result: HashMap<i64, Vec<Post>> = HashMap::new();
        for post in posts {
            result.entry(post.user_id).or_default().push(post);
        }

        Ok(result)
    }
}
```

### Using DataLoader

```rust
// Add to schema
let schema = Schema::build(QueryRoot, MutationRoot, SubscriptionRoot)
    .data(mm.clone())
    .data(DataLoader::new(
        UserLoader { mm: mm.clone() },
        tokio::spawn,
    ))
    .data(DataLoader::new(
        PostLoader { mm: mm.clone() },
        tokio::spawn,
    ))
    .finish();

// Use in resolver
#[Object]
impl PostGql {
    async fn author(&self, ctx: &Context<'_>) -> Result<UserGql> {
        let loader = ctx.data::<DataLoader<UserLoader>>()?;
        let user = loader.load_one(self.user_id).await?
            .ok_or_else(|| Error::new("User not found"))?;
        Ok(user.into())
    }
}
```

---

## Authentication and Authorization

### Context with Auth

```rust
use async_graphql::*;

// Request context
#[derive(Clone)]
pub struct Ctx {
    pub user_id: i64,
}

// Guard for authentication
pub struct AuthGuard;

impl Guard for AuthGuard {
    async fn check(&self, ctx: &Context<'_>) -> Result<()> {
        ctx.data::<Ctx>()
            .map(|_| ())
            .map_err(|_| Error::new("Unauthorized"))
    }
}

// Protected field
#[Object]
impl QueryRoot {
    #[graphql(guard = "AuthGuard")]
    async fn me(&self, ctx: &Context<'_>) -> Result<UserGql> {
        let request_ctx = ctx.data::<Ctx>()?;
        let mm = ctx.data::<ModelManager>()?;

        let user = UserBmc::get(request_ctx, mm, request_ctx.user_id).await?;
        Ok(user.into())
    }
}

// Role-based guard
pub struct RoleGuard {
    pub required_role: UserRole,
}

impl Guard for RoleGuard {
    async fn check(&self, ctx: &Context<'_>) -> Result<()> {
        let request_ctx = ctx.data::<Ctx>()?;
        let mm = ctx.data::<ModelManager>()?;

        let user = UserBmc::get(request_ctx, mm, request_ctx.user_id).await?;

        if user.role >= self.required_role {
            Ok(())
        } else {
            Err(Error::new("Insufficient permissions"))
        }
    }
}
```

### Axum Middleware for Context Injection

```rust
use axum::{
    extract::Request,
    middleware::Next,
    response::Response,
};

pub async fn auth_middleware(
    mut req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // Extract token from header
    let token = req.headers()
        .get("authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(StatusCode::UNAUTHORIZED)?;

    // Validate token and extract user_id
    let user_id = validate_token(token)
        .await
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    // Create context
    let ctx = Ctx { user_id };

    // Insert into request extensions
    req.extensions_mut().insert(ctx);

    Ok(next.run(req).await)
}

// GraphQL handler with context
async fn graphql_handler_with_auth(
    State(schema): State<AppSchema>,
    Extension(ctx): Extension<Ctx>,
    req: GraphQLRequest,
) -> GraphQLResponse {
    schema
        .execute(req.into_inner().data(ctx))
        .await
        .into()
}
```

---

## Error Handling

### Custom Error Types

```rust
use async_graphql::*;
use derive_more::{Display, From};

#[derive(Debug, Display, From)]
pub enum AppError {
    #[display(fmt = "Not found: {}", entity)]
    NotFound { entity: &'static str },

    #[display(fmt = "Unauthorized")]
    Unauthorized,

    #[display(fmt = "Validation error: {}", message)]
    Validation { message: String },

    #[from]
    Sqlx(sqlx::Error),
}

impl From<AppError> for Error {
    fn from(err: AppError) -> Self {
        match err {
            AppError::NotFound { entity } => {
                Error::new(format!("{} not found", entity))
                    .extend_with(|_, e| e.set("code", "NOT_FOUND"))
            }
            AppError::Unauthorized => {
                Error::new("Unauthorized")
                    .extend_with(|_, e| e.set("code", "UNAUTHORIZED"))
            }
            AppError::Validation { message } => {
                Error::new(message)
                    .extend_with(|_, e| e.set("code", "VALIDATION_ERROR"))
            }
            AppError::Sqlx(e) => {
                Error::new("Database error")
                    .extend_with(|_, e| e.set("code", "INTERNAL_ERROR"))
            }
        }
    }
}
```

### Error Extensions

```rust
#[Object]
impl QueryRoot {
    async fn user(&self, ctx: &Context<'_>, id: ID) -> Result<UserGql> {
        let mm = ctx.data::<ModelManager>()?;
        let user_id = id.parse::<i64>()?;

        let user = UserBmc::get(mm, user_id)
            .await
            .map_err(|_| {
                Error::new("User not found")
                    .extend_with(|_, e| {
                        e.set("code", "USER_NOT_FOUND");
                        e.set("user_id", user_id);
                    })
            })?;

        Ok(user.into())
    }
}
```

---

## Testing GraphQL APIs

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use async_graphql::*;

    #[tokio::test]
    async fn test_user_query() {
        let mm = create_test_model_manager().await;
        let schema = Schema::build(QueryRoot, EmptyMutation, EmptySubscription)
            .data(mm)
            .finish();

        let query = r#"
            query {
                user(id: "1") {
                    id
                    username
                    email
                }
            }
        "#;

        let result = schema.execute(query).await;
        assert!(result.errors.is_empty());

        let data = result.data.into_json().unwrap();
        assert_eq!(data["user"]["username"], "testuser");
    }

    #[tokio::test]
    async fn test_create_user_mutation() {
        let mm = create_test_model_manager().await;
        let schema = Schema::build(QueryRoot, MutationRoot, EmptySubscription)
            .data(mm)
            .finish();

        let query = r#"
            mutation {
                createUser(input: {
                    username: "newuser"
                    email: "new@example.com"
                    password: "password123"
                }) {
                    id
                    username
                }
            }
        "#;

        let result = schema.execute(query).await;
        assert!(result.errors.is_empty());
    }
}
```

---

## Complete Example

### Full Schema

```rust
// lib-web/src/graphql/mod.rs

use async_graphql::*;
use crate::ctx::Ctx;
use crate::model::ModelManager;

mod types;
mod queries;
mod mutations;
mod subscriptions;

pub use types::*;
use queries::QueryRoot;
use mutations::MutationRoot;
use subscriptions::SubscriptionRoot;

pub type AppSchema = Schema<QueryRoot, MutationRoot, SubscriptionRoot>;

pub fn build_schema(mm: ModelManager) -> AppSchema {
    Schema::build(QueryRoot, MutationRoot, SubscriptionRoot)
        .data(mm)
        .finish()
}
```

### Queries Module

```rust
// lib-web/src/graphql/queries.rs

use async_graphql::*;
use super::types::*;

pub struct QueryRoot;

#[Object]
impl QueryRoot {
    async fn user(&self, ctx: &Context<'_>, id: ID) -> Result<UserGql> {
        // Implementation
    }

    async fn users(&self, ctx: &Context<'_>) -> Result<Vec<UserGql>> {
        // Implementation
    }

    async fn post(&self, ctx: &Context<'_>, id: ID) -> Result<PostGql> {
        // Implementation
    }
}
```

---

## Related Files

- [SKILL.md](../SKILL.md) - Main Rust backend guidelines
- [api-design.md](api-design.md) - REST API patterns
- [grpc-tonic.md](grpc-tonic.md) - gRPC patterns
- [error-handling.md](error-handling.md) - Error handling strategies
- [testing-guide.md](testing-guide.md) - Testing approaches

---

**Pattern Status**: Production-ready GraphQL with Rust10x BMC integration ✅
