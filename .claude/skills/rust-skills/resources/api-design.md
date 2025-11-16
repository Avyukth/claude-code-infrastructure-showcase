# API Design - Axum Routing and Handlers

Complete guide to designing REST and JSON-RPC APIs with Axum, including routing, extractors, middleware, and best practices.

## Table of Contents

- [Axum Basics](#axum-basics)
- [Routing Patterns](#routing-patterns)
- [Extractors](#extractors)
- [Middleware Composition](#middleware-composition)
- [JSON-RPC vs REST](#json-rpc-vs-rest)
- [API Versioning](#api-versioning)
- [Request Validation](#request-validation)

---

## Axum Basics

### Simple Handler

```rust
use axum::{
    response::Json,
    http::StatusCode,
};
use serde::Serialize;

#[derive(Serialize)]
struct HealthResponse {
    status: String,
    version: String,
}

async fn health_check() -> Json<HealthResponse> {
    Json(HealthResponse {
        status: "healthy".to_string(),
        version: env!("CARGO_PKG_VERSION").to_string(),
    })
}
```

### Basic Router

```rust
use axum::{
    routing::{get, post},
    Router,
};

fn create_router() -> Router {
    Router::new()
        .route("/health", get(health_check))
        .route("/api/users", get(list_users).post(create_user))
        .route("/api/users/:id", get(get_user).put(update_user).delete(delete_user))
}
```

---

## Routing Patterns

### Nested Routes

```rust
use axum::Router;

fn api_routes() -> Router {
    Router::new()
        .nest("/users", user_routes())
        .nest("/posts", post_routes())
        .nest("/admin", admin_routes())
}

fn user_routes() -> Router {
    Router::new()
        .route("/", get(list_users).post(create_user))
        .route("/:id", get(get_user).put(update_user).delete(delete_user))
        .route("/:id/posts", get(get_user_posts))
        .route("/:id/subscriptions", get(get_user_subscriptions))
}
```

### Route Groups with Shared State

```rust
use axum::{
    extract::State,
    Router,
};
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    db: Arc<PgPool>,
    config: Arc<Config>,
}

fn create_app(state: AppState) -> Router {
    Router::new()
        .route("/api/users", get(list_users))
        .route("/api/users/:id", get(get_user))
        .with_state(state)
}

async fn list_users(
    State(state): State<AppState>,
) -> Result<Json<Vec<User>>, ApiError> {
    let users = sqlx::query_as!(
        User,
        "SELECT * FROM users ORDER BY id"
    )
    .fetch_all(&*state.db)
    .await?;

    Ok(Json(users))
}
```

### Fallback Routes

```rust
use axum::http::StatusCode;

fn create_router() -> Router {
    Router::new()
        .route("/api/users", get(list_users))
        .fallback(handler_404)
}

async fn handler_404() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "Route not found")
}
```

---

## Extractors

### Path Parameters

```rust
use axum::extract::Path;

// Single parameter
async fn get_user(
    Path(user_id): Path<i64>,
) -> Result<Json<User>, ApiError> {
    let user = UserBmc::get(&ctx, &mm, user_id).await?;
    Ok(Json(user))
}

// Multiple parameters
async fn get_user_post(
    Path((user_id, post_id)): Path<(i64, i64)>,
) -> Result<Json<Post>, ApiError> {
    let post = PostBmc::get_for_user(&ctx, &mm, user_id, post_id).await?;
    Ok(Json(post))
}

// Named parameters with struct
#[derive(Deserialize)]
struct UserPostPath {
    user_id: i64,
    post_id: i64,
}

async fn get_user_post_named(
    Path(params): Path<UserPostPath>,
) -> Result<Json<Post>, ApiError> {
    let post = PostBmc::get_for_user(&ctx, &mm, params.user_id, params.post_id).await?;
    Ok(Json(post))
}
```

### Query Parameters

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct Pagination {
    #[serde(default = "default_page")]
    page: u32,
    #[serde(default = "default_per_page")]
    per_page: u32,
}

fn default_page() -> u32 { 1 }
fn default_per_page() -> u32 { 20 }

async fn list_users(
    Query(pagination): Query<Pagination>,
) -> Result<Json<Vec<User>>, ApiError> {
    let offset = (pagination.page - 1) * pagination.per_page;
    let limit = pagination.per_page;

    let users = UserBmc::list_paginated(&ctx, &mm, offset, limit).await?;
    Ok(Json(users))
}

// Optional query parameters
#[derive(Deserialize)]
struct UserFilters {
    username: Option<String>,
    email: Option<String>,
    is_active: Option<bool>,
}

async fn search_users(
    Query(filters): Query<UserFilters>,
) -> Result<Json<Vec<User>>, ApiError> {
    let users = UserBmc::search(&ctx, &mm, filters).await?;
    Ok(Json(users))
}
```

### JSON Body

```rust
use axum::extract::Json;
use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Deserialize, Validate)]
struct CreateUserRequest {
    #[validate(length(min = 3, max = 50))]
    username: String,

    #[validate(email)]
    email: String,

    #[validate(length(min = 8))]
    password: String,
}

#[derive(Serialize)]
struct UserResponse {
    id: i64,
    username: String,
    email: String,
    created_at: time::OffsetDateTime,
}

async fn create_user(
    State(state): State<AppState>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<UserResponse>), ApiError> {
    // Validate
    payload.validate()?;

    // Create user
    let user_id = UserBmc::create(
        &ctx,
        &mm,
        UserForCreate {
            username: payload.username,
            email: payload.email,
        },
    )
    .await?;

    let user = UserBmc::get(&ctx, &mm, user_id).await?;

    Ok((
        StatusCode::CREATED,
        Json(UserResponse {
            id: user.id,
            username: user.username,
            email: user.email,
            created_at: user.ctime,
        }),
    ))
}
```

### Headers

```rust
use axum::{
    extract::TypedHeader,
    headers::{Authorization, UserAgent},
};

async fn protected_route(
    TypedHeader(auth): TypedHeader<Authorization<Bearer>>,
    TypedHeader(user_agent): TypedHeader<UserAgent>,
) -> Result<Json<Data>, ApiError> {
    let token = auth.token();
    // Validate token...
    Ok(Json(data))
}

// Custom header
use axum::http::HeaderMap;

async fn custom_headers(
    headers: HeaderMap,
) -> Result<Json<Data>, ApiError> {
    let api_key = headers
        .get("X-API-Key")
        .and_then(|v| v.to_str().ok())
        .ok_or(ApiError::MissingApiKey)?;

    // Validate API key...
    Ok(Json(data))
}
```

### Request Extensions

```rust
use axum::Extension;

// Set in middleware
async fn auth_middleware(
    mut request: Request,
    next: Next,
) -> Result<Response, ApiError> {
    let ctx = resolve_ctx(&request).await?;
    request.extensions_mut().insert(ctx);
    Ok(next.run(request).await)
}

// Extract in handler
async fn get_current_user(
    Extension(ctx): Extension<Ctx>,
) -> Result<Json<User>, ApiError> {
    let user = UserBmc::get(&ctx, &mm, ctx.user_id()).await?;
    Ok(Json(user))
}
```

### Multiple Extractors

```rust
async fn complex_handler(
    State(state): State<AppState>,
    Path(user_id): Path<i64>,
    Query(filters): Query<Filters>,
    Extension(ctx): Extension<Ctx>,
    Json(payload): Json<UpdateRequest>,
) -> Result<Json<User>, ApiError> {
    // All extractors available
    let user = UserBmc::update(&ctx, &mm, user_id, payload).await?;
    Ok(Json(user))
}
```

---

## Middleware Composition

### Middleware Ordering

```rust
use tower::ServiceBuilder;
use tower_http::{
    cors::CorsLayer,
    trace::TraceLayer,
    compression::CompressionLayer,
};

fn create_app(state: AppState) -> Router {
    Router::new()
        .route("/api/users", get(list_users))
        .layer(
            ServiceBuilder::new()
                // Executed bottom-up (compression is innermost)
                .layer(TraceLayer::new_for_http())        // 3. Logging
                .layer(CorsLayer::permissive())           // 2. CORS
                .layer(CompressionLayer::new())           // 1. Compression
        )
        .with_state(state)
}
```

### Custom Middleware

```rust
use axum::{
    middleware::{self, Next},
    response::Response,
    http::Request,
};

async fn request_id_middleware<B>(
    mut request: Request<B>,
    next: Next<B>,
) -> Response {
    let request_id = uuid::Uuid::new_v4();
    request.extensions_mut().insert(request_id);

    let mut response = next.run(request).await;
    response.headers_mut().insert(
        "X-Request-ID",
        request_id.to_string().parse().unwrap(),
    );

    response
}

fn create_app() -> Router {
    Router::new()
        .route("/api/users", get(list_users))
        .layer(middleware::from_fn(request_id_middleware))
}
```

### Authentication Middleware

```rust
async fn auth_middleware(
    State(state): State<AppState>,
    mut request: Request,
    next: Next,
) -> Result<Response, ApiError> {
    // Extract token from header
    let token = request
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok())
        .and_then(|v| v.strip_prefix("Bearer "))
        .ok_or(ApiError::Unauthorized)?;

    // Validate token
    let ctx = validate_token(&state, token).await?;

    // Insert context into request
    request.extensions_mut().insert(ctx);

    Ok(next.run(request).await)
}

fn protected_routes() -> Router {
    Router::new()
        .route("/api/users", get(list_users))
        .layer(middleware::from_fn_with_state(state.clone(), auth_middleware))
}
```

### Response Mapping Middleware

```rust
async fn response_map_middleware(
    uri: Uri,
    method: Method,
    response: Response,
) -> Response {
    let request_id = response
        .extensions()
        .get::<Uuid>()
        .copied()
        .unwrap_or_else(Uuid::new_v4);

    // Log response
    tracing::info!(
        request_id = %request_id,
        method = %method,
        uri = %uri,
        status = %response.status(),
        "response"
    );

    // Map errors to client-safe format
    if response.status().is_server_error() {
        // Don't leak internal errors
        let body = Json(json!({
            "error": "Internal server error",
            "request_id": request_id.to_string(),
        }));

        (StatusCode::INTERNAL_SERVER_ERROR, body).into_response()
    } else {
        response
    }
}
```

---

## JSON-RPC vs REST

### REST API Pattern

```rust
// routes/user_routes.rs

use axum::{
    routing::{get, post, put, delete},
    Router,
};

pub fn routes() -> Router {
    Router::new()
        .route("/users", get(list_users).post(create_user))
        .route("/users/:id", get(get_user).put(update_user).delete(delete_user))
}

async fn list_users() -> Result<Json<Vec<User>>, ApiError> {
    // GET /users
}

async fn create_user(Json(payload): Json<CreateUserRequest>) -> Result<Json<User>, ApiError> {
    // POST /users
}

async fn get_user(Path(id): Path<i64>) -> Result<Json<User>, ApiError> {
    // GET /users/:id
}

async fn update_user(
    Path(id): Path<i64>,
    Json(payload): Json<UpdateUserRequest>,
) -> Result<Json<User>, ApiError> {
    // PUT /users/:id
}

async fn delete_user(Path(id): Path<i64>) -> Result<StatusCode, ApiError> {
    // DELETE /users/:id
}
```

### JSON-RPC 2.0 Pattern

```rust
// Using rpc-router crate

use rpc_router::{RpcRouter, RpcHandlerWrapperTrait, Request as RpcRequest};

#[derive(Deserialize)]
struct CreateUserParams {
    data: UserForCreate,
}

async fn rpc_create_user(
    ctx: Ctx,
    mm: ModelManager,
    params: CreateUserParams,
) -> Result<i64, Error> {
    let user_id = UserBmc::create(&ctx, &mm, params.data).await?;
    Ok(user_id)
}

pub fn rpc_router() -> RpcRouter {
    RpcRouter::new()
        .add("create_user", rpc_create_user.into_rpc_handler())
        .add("get_user", rpc_get_user.into_rpc_handler())
        .add("list_users", rpc_list_users.into_rpc_handler())
}

// Client request:
// POST /api/rpc
// {
//   "jsonrpc": "2.0",
//   "id": 1,
//   "method": "create_user",
//   "params": {
//     "data": {
//       "username": "john",
//       "email": "john@example.com"
//     }
//   }
// }
```

### Comparison

| Aspect | REST | JSON-RPC |
|--------|------|----------|
| **URL Structure** | Resource-based (/users/:id) | Single endpoint (/api/rpc) |
| **HTTP Methods** | GET, POST, PUT, DELETE | POST only |
| **Versioning** | URL (/v1/users) | Method name (get_user_v2) |
| **Caching** | Easy (HTTP caching) | Harder (single endpoint) |
| **Batching** | Not built-in | Native support |
| **Discoverability** | Self-documenting URLs | Requires documentation |
| **Type Safety** | Manual | rpc-router provides |

**Choose REST when:**
- ✅ Standard CRUD operations
- ✅ Public API
- ✅ HTTP caching important
- ✅ RESTful principles valued

**Choose JSON-RPC when:**
- ✅ Complex operations
- ✅ Batch requests needed
- ✅ Internal microservices
- ✅ Type-safe routing desired

---

## API Versioning

### URL Versioning

```rust
fn create_app() -> Router {
    Router::new()
        .nest("/api/v1", v1_routes())
        .nest("/api/v2", v2_routes())
}

fn v1_routes() -> Router {
    Router::new()
        .route("/users", get(v1::list_users))
        .route("/users/:id", get(v1::get_user))
}

fn v2_routes() -> Router {
    Router::new()
        .route("/users", get(v2::list_users))
        .route("/users/:id", get(v2::get_user))
}
```

### Header Versioning

```rust
async fn versioned_handler(
    headers: HeaderMap,
) -> Result<Json<Response>, ApiError> {
    let version = headers
        .get("API-Version")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("v1");

    match version {
        "v1" => handle_v1().await,
        "v2" => handle_v2().await,
        _ => Err(ApiError::UnsupportedVersion),
    }
}
```

---

## Request Validation

### Using validator Crate

```rust
use validator::{Validate, ValidationError};

#[derive(Deserialize, Validate)]
struct CreateUserRequest {
    #[validate(length(min = 3, max = 50))]
    username: String,

    #[validate(email)]
    email: String,

    #[validate(length(min = 8), custom = "validate_password_strength")]
    password: String,

    #[validate(range(min = 18, max = 120))]
    age: u8,
}

fn validate_password_strength(password: &str) -> Result<(), ValidationError> {
    let has_uppercase = password.chars().any(|c| c.is_uppercase());
    let has_lowercase = password.chars().any(|c| c.is_lowercase());
    let has_digit = password.chars().any(|c| c.is_numeric());

    if has_uppercase && has_lowercase && has_digit {
        Ok(())
    } else {
        Err(ValidationError::new("weak_password"))
    }
}

async fn create_user(
    Json(payload): Json<CreateUserRequest>,
) -> Result<Json<User>, ApiError> {
    // Validate
    payload.validate().map_err(ApiError::ValidationError)?;

    // Process...
    Ok(Json(user))
}
```

### Custom Extractor with Validation

```rust
use axum::{
    async_trait,
    extract::{FromRequest, Request},
};

pub struct ValidatedJson<T>(pub T);

#[async_trait]
impl<T, S> FromRequest<S> for ValidatedJson<T>
where
    T: DeserializeOwned + Validate,
    S: Send + Sync,
{
    type Rejection = ApiError;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(|_| ApiError::InvalidJson)?;

        value.validate().map_err(ApiError::ValidationError)?;

        Ok(ValidatedJson(value))
    }
}

// Usage
async fn create_user(
    ValidatedJson(payload): ValidatedJson<CreateUserRequest>,
) -> Result<Json<User>, ApiError> {
    // Already validated!
    Ok(Json(user))
}
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [architecture-overview.md](architecture-overview.md)
- [error-handling.md](error-handling.md)
- [async-patterns.md](async-patterns.md)
- [security-patterns.md](security-patterns.md)
