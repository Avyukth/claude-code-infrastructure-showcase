# Security Patterns - Authentication and Security Best Practices

Complete guide to implementing secure authentication, authorization, and security best practices in Rust backend services.

## Table of Contents

- [Multi-Scheme Password Hashing](#multi-scheme-password-hashing)
- [Token-Based Authentication](#token-based-authentication)
- [Authorization Patterns](#authorization-patterns)
- [SQL Injection Prevention](#sql-injection-prevention)
- [Input Validation and Sanitization](#input-validation-and-sanitization)
- [Security Headers and CORS](#security-headers-and-cors)

---

## Multi-Scheme Password Hashing

### Why Multi-Scheme?

Support multiple hashing algorithms with automatic upgrade path for security improvements without forcing password resets.

**Storage Format**: `#01#base64_hash` or `#02#base64_hash`

### Argon2 Implementation (Scheme 02)

```rust
// lib-auth/src/pwd/scheme_02.rs

use argon2::{
    password_hash::{PasswordHash, PasswordHasher, PasswordVerifier, SaltString},
    Argon2,
};
use rand::rngs::OsRng;

pub fn hash_password(password: &str) -> Result<String> {
    let salt = SaltString::generate(&mut OsRng);
    let argon2 = Argon2::default();

    let password_hash = argon2
        .hash_password(password.as_bytes(), &salt)
        .map_err(|e| Error::PasswordHashingFailed(e.to_string()))?
        .to_string();

    // Prefix with scheme identifier
    Ok(format!("#02#{}", password_hash))
}

pub fn verify_password(password: &str, hash: &str) -> Result<bool> {
    // Remove scheme prefix
    let hash = hash.strip_prefix("#02#")
        .ok_or(Error::InvalidPasswordFormat)?;

    let parsed_hash = PasswordHash::new(hash)
        .map_err(|e| Error::InvalidPasswordFormat)?;

    let argon2 = Argon2::default();

    match argon2.verify_password(password.as_bytes(), &parsed_hash) {
        Ok(()) => Ok(true),
        Err(_) => Ok(false),
    }
}
```

### Legacy HMAC-SHA512 (Scheme 01)

```rust
// lib-auth/src/pwd/scheme_01.rs

use hmac::{Hmac, Mac};
use sha2::Sha512;
use base64::{Engine as _, engine::general_purpose};

type HmacSha512 = Hmac<Sha512>;

const HMAC_KEY: &[u8] = b"your-secret-key-change-in-production";

pub fn hash_password(password: &str) -> Result<String> {
    let mut mac = HmacSha512::new_from_slice(HMAC_KEY)
        .map_err(|e| Error::HashingError(e.to_string()))?;

    mac.update(password.as_bytes());
    let result = mac.finalize();
    let hash_bytes = result.into_bytes();

    let hash_b64 = general_purpose::STANDARD.encode(hash_bytes);

    Ok(format!("#01#{}", hash_b64))
}

pub fn verify_password(password: &str, hash: &str) -> Result<bool> {
    let expected_hash = hash_password(password)?;
    Ok(expected_hash == hash)
}
```

### Multi-Scheme Validation with Upgrade

```rust
// lib-auth/src/pwd/mod.rs

pub enum SchemeStatus {
    Current,
    Outdated,
}

pub async fn validate_pwd(pwd_clear: &str, pwd_ref: &str) -> Result<SchemeStatus> {
    match pwd_ref.split('#').nth(1) {
        Some("01") => {
            // Legacy scheme - verify but signal upgrade
            if scheme_01::verify_password(pwd_clear, pwd_ref)? {
                Ok(SchemeStatus::Outdated)
            } else {
                Err(Error::PasswordMismatch)
            }
        }
        Some("02") => {
            // Current scheme
            if scheme_02::verify_password(pwd_clear, pwd_ref)? {
                Ok(SchemeStatus::Current)
            } else {
                Err(Error::PasswordMismatch)
            }
        }
        _ => Err(Error::InvalidPasswordFormat),
    }
}

// Upgrade password on login if outdated
pub async fn login_and_upgrade(
    username: &str,
    password: &str,
    mm: &ModelManager,
) -> Result<User> {
    let user = UserBmc::first_by_username(&ctx, mm, username).await?;

    let status = validate_pwd(password, &user.pwd).await?;

    match status {
        SchemeStatus::Outdated => {
            // Upgrade to current scheme
            let new_hash = scheme_02::hash_password(password)?;
            UserBmc::update_password(&ctx, mm, user.id, new_hash).await?;
        }
        SchemeStatus::Current => {
            // Already using current scheme
        }
    }

    Ok(user)
}
```

---

## Token-Based Authentication

### Custom Token Format (JWT Alternative)

**Format**: `{user_id}::{exp_duration}::{signature}`

**Benefits**:
- Simpler than JWT
- Faster validation
- No library dependencies
- Easy to revoke (change user salt)

### Token Generation

```rust
// lib-auth/src/token/mod.rs

use blake3;
use uuid::Uuid;

pub fn generate_web_token(user_id: i64, salt: Uuid) -> Result<String> {
    let duration = CoreConfig::get().token_duration(); // e.g., "1h"
    let signature = _token_signature_into_b64u(user_id, duration, salt)?;

    Ok(format!("{}::{}.{}", user_id, duration, signature))
}

fn _token_signature_into_b64u(
    user_id: i64,
    duration: &str,
    salt: Uuid,
) -> Result<String> {
    let content = format!("{user_id}::{duration}::{salt}");

    let hash = blake3::hash(content.as_bytes());
    let b64 = base64_url_encode(hash.as_bytes());

    Ok(b64)
}

fn base64_url_encode(data: &[u8]) -> String {
    use base64::{Engine as _, engine::general_purpose};
    general_purpose::URL_SAFE_NO_PAD.encode(data)
}
```

### Token Validation

```rust
pub async fn validate_web_token(
    mm: &ModelManager,
    token: &str,
) -> Result<Ctx> {
    // Parse token
    let parts: Vec<&str> = token.split("::").collect();
    if parts.len() != 2 {
        return Err(Error::TokenInvalidFormat);
    }

    let user_id: i64 = parts[0].parse()
        .map_err(|_| Error::TokenInvalidFormat)?;

    let duration_and_sig = parts[1];
    let sig_parts: Vec<&str> = duration_and_sig.split('.').collect();
    if sig_parts.len() != 2 {
        return Err(Error::TokenInvalidFormat)?;
    }

    let duration = sig_parts[0];
    let provided_sig = sig_parts[1];

    // Get user salt from database
    let user = UserBmc::get(&Ctx::root_ctx(), mm, user_id).await?;

    // Recompute signature
    let expected_sig = _token_signature_into_b64u(user_id, duration, user.token_salt)?;

    // Constant-time comparison
    if !constant_time_compare(provided_sig, &expected_sig) {
        return Err(Error::TokenSignatureInvalid);
    }

    // Check expiration
    if is_token_expired(duration)? {
        return Err(Error::TokenExpired);
    }

    Ok(Ctx::new(user_id)?)
}

fn constant_time_compare(a: &str, b: &str) -> bool {
    if a.len() != b.len() {
        return false;
    }

    a.bytes()
        .zip(b.bytes())
        .fold(0, |acc, (a, b)| acc | (a ^ b))
        == 0
}
```

### Token Revocation

```rust
// Revoke all tokens for a user by changing their salt
pub async fn revoke_all_tokens(
    ctx: &Ctx,
    mm: &ModelManager,
    user_id: i64,
) -> Result<()> {
    let new_salt = Uuid::new_v4();

    UserBmc::update(ctx, mm, user_id, UserForUpdate {
        token_salt: Some(new_salt),
        ..Default::default()
    })
    .await?;

    Ok(())
}
```

---

## Authorization Patterns

### Context-Based Authorization

```rust
#[derive(Clone, Debug)]
pub struct Ctx {
    user_id: i64,
    permissions: Vec<Permission>,
}

#[derive(Clone, Debug, PartialEq)]
pub enum Permission {
    ViewUsers,
    CreateUsers,
    UpdateUsers,
    DeleteUsers,
    ViewPosts,
    CreatePosts,
    UpdateOwnPosts,
    UpdateAnyPosts,
    DeleteOwnPosts,
    DeleteAnyPosts,
}

impl Ctx {
    pub fn has_permission(&self, permission: Permission) -> bool {
        self.permissions.contains(&permission)
    }

    pub fn require_permission(&self, permission: Permission) -> Result<()> {
        if self.has_permission(permission.clone()) {
            Ok(())
        } else {
            Err(Error::PermissionDenied {
                required: format!("{:?}", permission),
            })
        }
    }
}
```

### Resource-Level Authorization

```rust
pub async fn update_post(
    ctx: &Ctx,
    mm: &ModelManager,
    post_id: i64,
    data: PostForUpdate,
) -> Result<()> {
    // Get post
    let post = PostBmc::get(ctx, mm, post_id).await?;

    // Check ownership or admin permission
    if post.user_id == ctx.user_id() {
        // User owns the post
    } else if ctx.has_permission(Permission::UpdateAnyPosts) {
        // User has admin permission
    } else {
        return Err(Error::PermissionDenied {
            required: "post ownership or UpdateAnyPosts".to_string(),
        });
    }

    // Perform update
    base::update::<Self, _>(ctx, mm, post_id, data).await
}
```

### Middleware-Based Authorization

```rust
use axum::middleware::{self, Next};

async fn require_permission_middleware(
    Extension(ctx): Extension<Ctx>,
    Extension(required): Extension<Permission>,
    request: Request,
    next: Next,
) -> Result<Response, ApiError> {
    ctx.require_permission(required)?;
    Ok(next.run(request).await)
}

fn admin_routes() -> Router {
    Router::new()
        .route("/users", get(list_all_users))
        .layer(Extension(Permission::ViewUsers))
        .layer(middleware::from_fn(require_permission_middleware))
}
```

---

## SQL Injection Prevention

### Always Use Parameterized Queries

```rust
// ✅ SAFE: SQLx parameterized query
pub async fn get_user_by_username(
    pool: &PgPool,
    username: &str,
) -> Result<Option<User>> {
    let user = sqlx::query_as!(
        User,
        "SELECT * FROM users WHERE username = $1",
        username // Safely parameterized
    )
    .fetch_optional(pool)
    .await?;

    Ok(user)
}

// ❌ DANGEROUS: String concatenation (NEVER DO THIS)
pub async fn unsafe_get_user(
    pool: &PgPool,
    username: &str,
) -> Result<Option<User>> {
    let sql = format!("SELECT * FROM users WHERE username = '{}'", username);
    // SQL INJECTION VULNERABILITY!
    // username = "'; DROP TABLE users; --"
}
```

### Dynamic Queries with Sea-Query

```rust
use sea_query::{Expr, PostgresQueryBuilder, Query};

pub async fn search_users_safe(
    pool: &PgPool,
    filters: UserFilters,
) -> Result<Vec<User>> {
    let mut query = Query::select();
    query.from(User::Table).columns([User::Id, User::Username]);

    // Safe parameterization
    if let Some(username) = filters.username {
        query.and_where(Expr::col(User::Username).like(format!("%{}%", username)));
    }

    if let Some(email) = filters.email {
        query.and_where(Expr::col(User::Email).eq(email));
    }

    let (sql, values) = query.build_sqlx(PostgresQueryBuilder);

    let users = sqlx::query_as_with(&sql, values)
        .fetch_all(pool)
        .await?;

    Ok(users)
}
```

---

## Input Validation and Sanitization

### Input Validation with validator

```rust
use validator::{Validate, ValidationError};

#[derive(Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(length(min = 3, max = 50))]
    #[validate(regex = "USERNAME_REGEX")]
    username: String,

    #[validate(email)]
    email: String,

    #[validate(length(min = 8))]
    #[validate(custom = "validate_password_strength")]
    password: String,
}

lazy_static! {
    static ref USERNAME_REGEX: Regex = Regex::new(r"^[a-zA-Z0-9_-]+$").unwrap();
}

fn validate_password_strength(password: &str) -> Result<(), ValidationError> {
    let has_uppercase = password.chars().any(|c| c.is_uppercase());
    let has_lowercase = password.chars().any(|c| c.is_lowercase());
    let has_digit = password.chars().any(|c| c.is_numeric());
    let has_special = password.chars().any(|c| "!@#$%^&*()".contains(c));

    if has_uppercase && has_lowercase && has_digit && has_special {
        Ok(())
    } else {
        Err(ValidationError::new("weak_password"))
    }
}
```

### HTML Sanitization

```rust
use ammonia::clean;

pub fn sanitize_html(input: &str) -> String {
    clean(input)
}

pub async fn create_post(
    ctx: &Ctx,
    mm: &ModelManager,
    data: PostForCreate,
) -> Result<i64> {
    // Sanitize HTML content
    let sanitized_content = sanitize_html(&data.content);

    let post_data = PostForCreate {
        title: data.title,
        content: sanitized_content,
        user_id: ctx.user_id(),
    };

    base::create::<Self, _>(ctx, mm, post_data).await
}
```

---

## Security Headers and CORS

### Security Headers Middleware

```rust
use axum::{
    middleware::{self, Next},
    response::Response,
    http::Request,
};

async fn security_headers_middleware<B>(
    request: Request<B>,
    next: Next<B>,
) -> Response {
    let mut response = next.run(request).await;

    let headers = response.headers_mut();

    // Prevent clickjacking
    headers.insert("X-Frame-Options", "DENY".parse().unwrap());

    // Prevent MIME sniffing
    headers.insert("X-Content-Type-Options", "nosniff".parse().unwrap());

    // Enable XSS protection
    headers.insert("X-XSS-Protection", "1; mode=block".parse().unwrap());

    // Content Security Policy
    headers.insert(
        "Content-Security-Policy",
        "default-src 'self'; script-src 'self' 'unsafe-inline'".parse().unwrap(),
    );

    // HSTS
    headers.insert(
        "Strict-Transport-Security",
        "max-age=31536000; includeSubDomains".parse().unwrap(),
    );

    response
}
```

### CORS Configuration

```rust
use tower_http::cors::{CorsLayer, Any};

fn create_app() -> Router {
    let cors = CorsLayer::new()
        .allow_origin("https://yourdomain.com".parse::<HeaderValue>().unwrap())
        .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
        .allow_headers([AUTHORIZATION, CONTENT_TYPE])
        .allow_credentials(true);

    Router::new()
        .route("/api/users", get(list_users))
        .layer(cors)
}
```

### Rate Limiting

```rust
use tower_governor::{
    governor::GovernorConfigBuilder,
    GovernorLayer,
};

fn create_app() -> Router {
    // 10 requests per IP per minute
    let governor_conf = Box::new(
        GovernorConfigBuilder::default()
            .per_millisecond(60_000)
            .burst_size(10)
            .finish()
            .unwrap(),
    );

    Router::new()
        .route("/api/users", post(create_user))
        .layer(GovernorLayer {
            config: Box::leak(governor_conf),
        })
}
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [error-handling.md](error-handling.md)
- [api-design.md](api-design.md)
- [testing-guide.md](testing-guide.md)
