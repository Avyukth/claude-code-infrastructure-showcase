# Error Handling - Rust Best Practices

Complete guide to error handling in Rust backend services using modern patterns and the `derive_more` crate.

## Table of Contents

- [The Two Modes of Error Handling](#the-two-modes-of-error-handling)
- [Production Error Pattern](#production-error-pattern)
- [Custom Error Types](#custom-error-types)
- [Error Propagation](#error-propagation)
- [Error Context](#error-context)
- [HTTP Error Mapping](#http-error-mapping)
- [Common Patterns](#common-patterns)

---

## The Two Modes of Error Handling

### Test/Example Code vs Production Code

There are two sides of the spectrum for error handling in Rust:

**Test Code** - For tests, examples, and early development:
- Want something **loose**, **simple**, and **transparent**
- `Box<dyn Error>` provides flexibility
- Context via `String` or `&'static str` is sufficient

**Production Code** - For production apps and libraries:
- Benefits of **strict** type-safe errors
- Full control over error variants
- Explicit error handling at boundaries

**Golden Rule**: Use the `?` operator consistently in both modes. Avoid `.unwrap()`, `.expect()`, and even `.context()` from `anyhow` in favor of explicit error types.

---

## Test/Example Error Pattern

### Basic Pattern

```rust
// For tests, examples, early development

pub type Result<T> = core::result::Result<T, Error>;
pub type Error = Box<dyn std::error::Error>; // Boxed trait object

fn hello_world() -> Result<String> {
    Ok(format!("Hello, World!"))
}

fn main() -> Result<()> {
    let message = hello_world()?;
    println!("{message}");

    // Works because std::io::Error implements std::error::Error
    let contents = std::fs::read_to_string("config.toml")?;

    Ok(())
}
```

**Benefits:**
- ✅ Very flexible - any error converts automatically
- ✅ Minimal boilerplate
- ✅ Good for prototyping
- ✅ Similar to `anyhow` but using std types

**Limitations:**
- ⚠️ Lose type information
- ⚠️ Can't pattern match on error variants
- ⚠️ Not suitable for library public APIs

### Example: Quick Dev Script

```rust
// examples/quick_dev.rs

type Result<T> = core::result::Result<T, Error>;
type Error = Box<dyn std::error::Error>;

#[tokio::main]
async fn main() -> Result<()> {
    // Database connection
    let pool = PgPool::connect(&env::var("DATABASE_URL")?).await?;

    // Any error type works
    let user = create_test_user(&pool).await?;
    println!("Created user: {user:?}");

    Ok(())
}

async fn create_test_user(pool: &PgPool) -> Result<User> {
    // SQLx error, validation error, business logic error - all work
    let user = sqlx::query_as!(
        User,
        "INSERT INTO users (username, email) VALUES ($1, $2) RETURNING *",
        "testuser",
        "test@example.com"
    )
    .fetch_one(pool)
    .await?;

    Ok(user)
}
```

---

## Production Error Pattern

### Custom Error Enum with derive_more

```rust
// lib-core/src/model/error.rs

use derive_more::From;

pub type Result<T> = core::result::Result<T, Error>;

#[derive(Debug, From)]
pub enum Error {
    // -- Entity Not Found
    EntityNotFound { entity: &'static str, id: i64 },

    // -- Validation Errors
    #[from]
    ValidationError(ValidationError),

    // -- Property Errors
    PropertyRequired { property: &'static str },
    PropertyInvalid { property: &'static str, reason: String },

    // -- Database Errors
    #[from]
    Sqlx(sqlx::Error),

    #[from]
    SeaQuery(sea_query::error::Error),

    // -- Authentication Errors
    #[from]
    AuthError(lib_auth::pwd::Error),

    TokenInvalid,
    CannotDecodeToken,

    // -- Authorization Errors
    PermissionDenied { required: &'static str },

    // -- Business Logic Errors
    EmailAlreadyExists { email: String },
    UsernameAlreadyExists { username: String },
    InvalidState { expected: String, actual: String },

    // -- External Service Errors
    #[from]
    HttpClient(reqwest::Error),

    // -- Internal Errors (should not leak to client)
    ConfigMissing { config_key: &'static str },
    InternalServerError,
}

// Implement Display (required for std::error::Error)
impl core::fmt::Display for Error {
    fn fmt(&self, fmt: &mut core::fmt::Formatter) -> core::result::Result<(), core::fmt::Error> {
        write!(fmt, "{self:?}")
    }
}

// Implement std::error::Error
impl std::error::Error for Error {}
```

**Key Decisions:**
- `derive_more::From` instead of `thiserror`
- Selective `#[from]` for automatic conversions
- Structured error variants with data
- Display = Debug for simplicity (will convert to JSON anyway)

### Why derive_more Over thiserror?

**derive_more**:
- ✅ Selective derives - only use what you need
- ✅ More flexible
- ✅ Can use `From` without `Display` if not needed
- ✅ Lighter weight

**thiserror**:
- ⚠️ More opinionated
- ⚠️ Requires implementing Display for all variants
- ✅ Better error messages out of the box
- ✅ More familiar to many Rust developers

**Choose derive_more** when you want full control and will map errors to JSON at the API boundary anyway.

---

## Custom Error Types

### Validation Error

```rust
// lib-core/src/model/validation.rs

use derive_more::From;

#[derive(Debug, Clone)]
pub struct ValidationError {
    pub field: String,
    pub reason: String,
}

impl ValidationError {
    pub fn new(field: impl Into<String>, reason: impl Into<String>) -> Self {
        Self {
            field: field.into(),
            reason: reason.into(),
        }
    }
}

impl std::fmt::Display for ValidationError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Validation error on field '{}': {}", self.field, self.reason)
    }
}

impl std::error::Error for ValidationError {}
```

### Usage in Business Logic

```rust
pub async fn create_user(
    ctx: &Ctx,
    mm: &ModelManager,
    data: UserForCreate,
) -> Result<i64> {
    // Validate username
    if data.username.len() < 3 {
        return Err(Error::PropertyInvalid {
            property: "username",
            reason: "must be at least 3 characters".to_string(),
        });
    }

    // Check if username exists
    let existing = UserBmc::first_by_username(ctx, mm, &data.username).await;
    if existing.is_ok() {
        return Err(Error::UsernameAlreadyExists {
            username: data.username,
        });
    }

    // Create user
    base::create::<Self, _>(ctx, mm, data).await
}
```

---

## Error Propagation

### The ? Operator

```rust
// ✅ ALWAYS use ? for error propagation
async fn get_user_with_posts(
    ctx: &Ctx,
    mm: &ModelManager,
    user_id: i64,
) -> Result<UserWithPosts> {
    // Automatically converts errors
    let user = UserBmc::get(ctx, mm, user_id).await?;
    let posts = PostBmc::list_for_user(ctx, mm, user_id).await?;

    Ok(UserWithPosts { user, posts })
}

// ❌ NEVER use unwrap() in production code
async fn bad_example(ctx: &Ctx, mm: &ModelManager, id: i64) -> Result<User> {
    let user = UserBmc::get(ctx, mm, id).await.unwrap(); // PANIC!
    Ok(user)
}

// ❌ NEVER use expect() in production code
async fn also_bad(ctx: &Ctx, mm: &ModelManager, id: i64) -> Result<User> {
    let user = UserBmc::get(ctx, mm, id).await.expect("user should exist"); // PANIC!
    Ok(user)
}
```

### Converting Between Error Types

```rust
// Automatic conversion with #[from]
#[derive(Debug, From)]
pub enum Error {
    #[from]
    Sqlx(sqlx::Error),

    #[from]
    SeaQuery(sea_query::error::Error),
}

// Usage: automatic conversion via ?
async fn query_user(pool: &PgPool, id: i64) -> Result<User> {
    let user = sqlx::query_as!(
        User,
        "SELECT * FROM users WHERE id = $1",
        id
    )
    .fetch_one(pool)
    .await?; // sqlx::Error automatically converts to Error

    Ok(user)
}

// Manual conversion when needed
impl From<MyLibraryError> for Error {
    fn from(err: MyLibraryError) -> Self {
        match err {
            MyLibraryError::NotFound => Error::EntityNotFound {
                entity: "resource",
                id: 0,
            },
            MyLibraryError::Validation(msg) => Error::PropertyInvalid {
                property: "unknown",
                reason: msg,
            },
            _ => Error::InternalServerError,
        }
    }
}
```

---

## Error Context

### Adding Context to Errors

```rust
// ✅ Add context at the error creation site
pub async fn update_user(
    ctx: &Ctx,
    mm: &ModelManager,
    id: i64,
    data: UserForUpdate,
) -> Result<()> {
    // Check user exists first
    let _user = UserBmc::get(ctx, mm, id).await.map_err(|_| {
        Error::EntityNotFound {
            entity: "User",
            id,
        }
    })?;

    // Perform update
    base::update::<Self, _>(ctx, mm, id, data).await?;

    Ok(())
}

// ✅ Use custom error variants for business logic
pub async fn delete_user(
    ctx: &Ctx,
    mm: &ModelManager,
    id: i64,
) -> Result<()> {
    // Check if user has active subscriptions
    let has_active = SubscriptionBmc::has_active_for_user(ctx, mm, id).await?;
    if has_active {
        return Err(Error::InvalidState {
            expected: "User must cancel all subscriptions before deletion".to_string(),
            actual: "User has active subscriptions".to_string(),
        });
    }

    base::delete::<Self>(ctx, mm, id).await
}
```

### Pattern: Result Extensions

```rust
// Create extension trait for adding context
pub trait ResultExt<T> {
    fn map_entity_not_found(self, entity: &'static str, id: i64) -> Result<T>;
}

impl<T> ResultExt<T> for Result<T> {
    fn map_entity_not_found(self, entity: &'static str, id: i64) -> Result<T> {
        self.map_err(|_| Error::EntityNotFound { entity, id })
    }
}

// Usage
pub async fn get_user(ctx: &Ctx, mm: &ModelManager, id: i64) -> Result<User> {
    UserBmc::get(ctx, mm, id)
        .await
        .map_entity_not_found("User", id)
}
```

---

## HTTP Error Mapping

### IntoResponse Implementation

```rust
// lib-web/src/error.rs

use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde::Serialize;
use serde_json::json;

#[derive(Debug, Serialize)]
struct ErrorResponse {
    error: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    details: Option<serde_json::Value>,
}

impl IntoResponse for lib_core::model::Error {
    fn into_response(self) -> Response {
        let (status, message, details) = match self {
            // -- 400 Bad Request
            Error::PropertyRequired { property } => (
                StatusCode::BAD_REQUEST,
                format!("Missing required property: {property}"),
                None,
            ),

            Error::PropertyInvalid { property, reason } => (
                StatusCode::BAD_REQUEST,
                format!("Invalid property: {property}"),
                Some(json!({ "reason": reason })),
            ),

            Error::ValidationError(err) => (
                StatusCode::BAD_REQUEST,
                "Validation failed".to_string(),
                Some(json!({ "field": err.field, "reason": err.reason })),
            ),

            // -- 401 Unauthorized
            Error::TokenInvalid | Error::CannotDecodeToken => (
                StatusCode::UNAUTHORIZED,
                "Invalid or expired authentication token".to_string(),
                None,
            ),

            // -- 403 Forbidden
            Error::PermissionDenied { required } => (
                StatusCode::FORBIDDEN,
                format!("Permission denied. Required: {required}"),
                None,
            ),

            // -- 404 Not Found
            Error::EntityNotFound { entity, id } => (
                StatusCode::NOT_FOUND,
                format!("{entity} with id {id} not found"),
                None,
            ),

            // -- 409 Conflict
            Error::EmailAlreadyExists { email } => (
                StatusCode::CONFLICT,
                format!("Email already exists: {email}"),
                None,
            ),

            Error::UsernameAlreadyExists { username } => (
                StatusCode::CONFLICT,
                format!("Username already exists: {username}"),
                None,
            ),

            // -- 422 Unprocessable Entity
            Error::InvalidState { expected, actual } => (
                StatusCode::UNPROCESSABLE_ENTITY,
                "Invalid state".to_string(),
                Some(json!({ "expected": expected, "actual": actual })),
            ),

            // -- 500 Internal Server Error (never leak internal details)
            Error::Sqlx(_) | Error::SeaQuery(_) | Error::InternalServerError => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "Internal server error".to_string(),
                None,
            ),

            // Catch-all for unexpected errors
            _ => (
                StatusCode::INTERNAL_SERVER_ERROR,
                "An unexpected error occurred".to_string(),
                None,
            ),
        };

        let body = Json(ErrorResponse {
            error: message,
            details,
        });

        (status, body).into_response()
    }
}
```

### Response Mapper Middleware

```rust
// lib-web/src/middleware/mw_res_map.rs

use axum::{
    http::{Method, Uri},
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;
use tracing::debug;
use uuid::Uuid;

pub async fn mw_response_map(
    uri: Uri,
    req_method: Method,
    res: Response,
) -> Response {
    let uuid = Uuid::new_v4();

    // Get the response error if any
    let service_error = res.extensions().get::<Error>();
    let client_status_error = service_error.map(|se| se.client_status_and_error());

    // If client error, build new response
    let error_response = client_status_error
        .as_ref()
        .map(|(status_code, client_error)| {
            let client_error = to_json_string(client_error);
            debug!("    ->> client_error_body: {client_error}");

            // Build new response from error
            (*status_code, Json(client_error)).into_response()
        });

    // Log server error
    if let Some(service_error) = service_error {
        debug!("  ->> server_error: {service_error:?}");
    }

    // Return error response or original
    error_response.unwrap_or(res)
}
```

---

## Common Patterns

### Pattern 1: Early Return on Validation

```rust
pub async fn create_post(
    ctx: &Ctx,
    mm: &ModelManager,
    data: PostForCreate,
) -> Result<i64> {
    // Validate title length
    if data.title.is_empty() {
        return Err(Error::PropertyRequired { property: "title" });
    }

    if data.title.len() > 200 {
        return Err(Error::PropertyInvalid {
            property: "title",
            reason: "must be less than 200 characters".to_string(),
        });
    }

    // Validate user has permission
    let user = UserBmc::get(ctx, mm, ctx.user_id()).await?;
    if !user.can_create_posts() {
        return Err(Error::PermissionDenied {
            required: "create_posts",
        });
    }

    // Create post
    base::create::<Self, _>(ctx, mm, data).await
}
```

### Pattern 2: Option to Result Conversion

```rust
// Convert Option to Result with custom error
pub async fn get_active_subscription(
    ctx: &Ctx,
    mm: &ModelManager,
    user_id: i64,
) -> Result<Subscription> {
    SubscriptionBmc::find_active(ctx, mm, user_id)
        .await?
        .ok_or(Error::EntityNotFound {
            entity: "Active Subscription",
            id: user_id,
        })
}

// Alternative: custom extension trait
pub trait OptionExt<T> {
    fn ok_or_not_found(self, entity: &'static str, id: i64) -> Result<T>;
}

impl<T> OptionExt<T> for Option<T> {
    fn ok_or_not_found(self, entity: &'static str, id: i64) -> Result<T> {
        self.ok_or(Error::EntityNotFound { entity, id })
    }
}

// Usage
pub async fn get_user(ctx: &Ctx, mm: &ModelManager, id: i64) -> Result<User> {
    UserBmc::find_by_id(ctx, mm, id)
        .await?
        .ok_or_not_found("User", id)
}
```

### Pattern 3: Transaction Error Handling

```rust
pub async fn transfer_subscription(
    ctx: &Ctx,
    mm: &ModelManager,
    from_user_id: i64,
    to_user_id: i64,
    subscription_id: i64,
) -> Result<()> {
    // Start transaction
    let mm = mm.new_with_txn().await?;

    // Perform operations (any error rolls back automatically)
    let subscription = SubscriptionBmc::get(ctx, &mm, subscription_id).await?;

    // Validate ownership
    if subscription.user_id != from_user_id {
        return Err(Error::PermissionDenied {
            required: "subscription ownership",
        });
    }

    // Update ownership
    SubscriptionBmc::update(
        ctx,
        &mm,
        subscription_id,
        SubscriptionForUpdate {
            user_id: Some(to_user_id),
            ..Default::default()
        },
    )
    .await?;

    // Create audit log
    AuditLogBmc::create(
        ctx,
        &mm,
        AuditLogForCreate {
            action: "transfer_subscription".to_string(),
            entity_type: "Subscription".to_string(),
            entity_id: subscription_id,
        },
    )
    .await?;

    // Commit transaction
    mm.dbx().commit_txn().await?;

    Ok(())
}
```

### Pattern 4: Error Aggregation

```rust
pub async fn validate_batch(
    ctx: &Ctx,
    mm: &ModelManager,
    items: Vec<ItemForCreate>,
) -> Result<Vec<ValidationError>> {
    let mut errors = Vec::new();

    for (index, item) in items.iter().enumerate() {
        // Validate each item
        if item.name.is_empty() {
            errors.push(ValidationError::new(
                format!("items[{index}].name"),
                "is required",
            ));
        }

        if item.price < 0.0 {
            errors.push(ValidationError::new(
                format!("items[{index}].price"),
                "must be non-negative",
            ));
        }
    }

    if !errors.is_empty() {
        return Err(Error::BatchValidationFailed { errors });
    }

    Ok(())
}
```

---

## Testing Error Handling

### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_username_validation() {
        let ctx = Ctx::root_ctx();
        let mm = ModelManager::new().await.unwrap();

        // Test short username
        let result = UserBmc::create(
            &ctx,
            &mm,
            UserForCreate {
                username: "ab".to_string(),
                email: "test@example.com".to_string(),
            },
        )
        .await;

        assert!(result.is_err());
        match result.unwrap_err() {
            Error::PropertyInvalid { property, .. } => {
                assert_eq!(property, "username");
            }
            _ => panic!("Expected PropertyInvalid error"),
        }
    }

    #[tokio::test]
    async fn test_duplicate_username() {
        let ctx = Ctx::root_ctx();
        let mm = ModelManager::new().await.unwrap();

        // Create first user
        UserBmc::create(&ctx, &mm, UserForCreate {
            username: "testuser".to_string(),
            email: "test1@example.com".to_string(),
        })
        .await
        .unwrap();

        // Try to create duplicate
        let result = UserBmc::create(&ctx, &mm, UserForCreate {
            username: "testuser".to_string(),
            email: "test2@example.com".to_string(),
        })
        .await;

        assert!(matches!(result, Err(Error::UsernameAlreadyExists { .. })));
    }
}
```

---

**Related Files:**
- [SKILL.md](../SKILL.md) - Main guide
- [architecture-overview.md](architecture-overview.md) - Architecture patterns
- [api-design.md](api-design.md) - HTTP error mapping
- [testing-guide.md](testing-guide.md) - Testing error scenarios
