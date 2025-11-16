# Complete Examples - Full Working Code

Real-world examples showing complete implementation patterns from database to API.

## Table of Contents

- [Complete CRUD API](#complete-crud-api)
- [Authentication Flow](#authentication-flow)
- [File Upload Handling](#file-upload-handling)
- [Background Job Processing](#background-job-processing)
- [Complete Microservice Example](#complete-microservice-example)

---

## Complete CRUD API

### Complete User Management Feature

**1. Database Schema**

```sql
-- migrations/001_users.sql

CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(255) NOT NULL UNIQUE,
    pwd TEXT NOT NULL,
    pwd_salt UUID NOT NULL DEFAULT gen_random_uuid(),
    token_salt UUID NOT NULL DEFAULT gen_random_uuid(),

    -- Audit fields
    cid BIGINT NOT NULL,
    ctime TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    mid BIGINT NOT NULL,
    mtime TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

**2. Domain Models**

```rust
// lib-core/src/model/user.rs

use serde::{Deserialize, Serialize};
use sqlx::FromRow;
use modql::field::Fields;

#[derive(Debug, Clone, Fields, FromRow, Serialize)]
pub struct User {
    pub id: i64,
    pub username: String,
    pub email: String,

    #[serde(skip)]
    pub pwd: String,

    #[serde(skip)]
    pub pwd_salt: uuid::Uuid,

    #[serde(skip)]
    pub token_salt: uuid::Uuid,

    pub cid: i64,
    pub ctime: time::OffsetDateTime,
    pub mid: i64,
    pub mtime: time::OffsetDateTime,
}

#[derive(Fields, Deserialize)]
pub struct UserForCreate {
    pub username: String,
    pub email: String,
    pub pwd_clear: String,
}

#[derive(Fields, Deserialize, Default)]
pub struct UserForUpdate {
    pub username: Option<String>,
    pub email: Option<String>,
}
```

**3. Backend Model Controller (BMC)**

```rust
// lib-core/src/model/user.rs (continued)

use crate::ctx::Ctx;
use crate::model::{Error, Result, ModelManager, base};

pub struct UserBmc;

impl UserBmc {
    pub async fn create(
        ctx: &Ctx,
        mm: &ModelManager,
        user_c: UserForCreate,
    ) -> Result<i64> {
        // Validate
        if user_c.username.len() < 3 {
            return Err(Error::PropertyInvalid {
                property: "username",
                reason: "must be at least 3 characters".to_string(),
            });
        }

        // Check if username exists
        if let Ok(Some(_)) = Self::first_by_username(ctx, mm, &user_c.username).await {
            return Err(Error::UsernameAlreadyExists {
                username: user_c.username,
            });
        }

        // Check if email exists
        if let Ok(Some(_)) = Self::first_by_email(ctx, mm, &user_c.email).await {
            return Err(Error::EmailAlreadyExists {
                email: user_c.email,
            });
        }

        // Hash password
        let pwd = lib_auth::pwd::hash_pwd(&user_c.pwd_clear).await?;

        // Create user data
        let data = UserForCreate {
            username: user_c.username,
            email: user_c.email,
            pwd,
        };

        base::create::<Self, _>(ctx, mm, data).await
    }

    pub async fn get(ctx: &Ctx, mm: &ModelManager, id: i64) -> Result<User> {
        base::get::<Self, _>(ctx, mm, id).await
    }

    pub async fn first_by_username(
        ctx: &Ctx,
        mm: &ModelManager,
        username: &str,
    ) -> Result<Option<User>> {
        let user = sqlx::query_as!(
            User,
            r#"SELECT * FROM users WHERE username = $1"#,
            username
        )
        .fetch_optional(mm.db().pool())
        .await?;

        Ok(user)
    }

    pub async fn first_by_email(
        ctx: &Ctx,
        mm: &ModelManager,
        email: &str,
    ) -> Result<Option<User>> {
        let user = sqlx::query_as!(
            User,
            r#"SELECT * FROM users WHERE email = $1"#,
            email
        )
        .fetch_optional(mm.db().pool())
        .await?;

        Ok(user)
    }

    pub async fn list(
        ctx: &Ctx,
        mm: &ModelManager,
        limit: Option<u32>,
        offset: Option<u32>,
    ) -> Result<Vec<User>> {
        let limit = limit.unwrap_or(20);
        let offset = offset.unwrap_or(0);

        let users = sqlx::query_as!(
            User,
            r#"
            SELECT * FROM users
            ORDER BY id DESC
            LIMIT $1 OFFSET $2
            "#,
            limit as i64,
            offset as i64
        )
        .fetch_all(mm.db().pool())
        .await?;

        Ok(users)
    }

    pub async fn update(
        ctx: &Ctx,
        mm: &ModelManager,
        id: i64,
        user_u: UserForUpdate,
    ) -> Result<()> {
        base::update::<Self, _>(ctx, mm, id, user_u).await
    }

    pub async fn delete(ctx: &Ctx, mm: &ModelManager, id: i64) -> Result<()> {
        base::delete::<Self>(ctx, mm, id).await
    }
}
```

**4. API Types**

```rust
// web-server/src/web/api_types.rs

use serde::{Deserialize, Serialize};
use validator::Validate;

#[derive(Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(length(min = 3, max = 50))]
    pub username: String,

    #[validate(email)]
    pub email: String,

    #[validate(length(min = 8))]
    pub password: String,
}

#[derive(Deserialize, Validate)]
pub struct UpdateUserRequest {
    #[validate(length(min = 3, max = 50))]
    pub username: Option<String>,

    #[validate(email)]
    pub email: Option<String>,
}

#[derive(Serialize)]
pub struct UserResponse {
    pub id: i64,
    pub username: String,
    pub email: String,
    pub created_at: String,
}

impl From<User> for UserResponse {
    fn from(user: User) -> Self {
        Self {
            id: user.id,
            username: user.username,
            email: user.email,
            created_at: user.ctime.to_string(),
        }
    }
}
```

**5. API Handlers**

```rust
// web-server/src/web/routes_user.rs

use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::Json,
    routing::{get, post},
    Router,
    Extension,
};
use serde::Deserialize;

use lib_core::ctx::Ctx;
use lib_core::model::{ModelManager, UserBmc, UserForCreate, UserForUpdate};

use crate::web::api_types::*;
use crate::web::error::ApiError;
use crate::AppState;

pub fn routes(state: AppState) -> Router {
    Router::new()
        .route("/api/users", get(list_users).post(create_user))
        .route("/api/users/:id", get(get_user).put(update_user).delete(delete_user))
        .with_state(state)
}

// List Users
#[derive(Deserialize)]
struct ListQuery {
    #[serde(default)]
    limit: Option<u32>,

    #[serde(default)]
    offset: Option<u32>,
}

async fn list_users(
    State(state): State<AppState>,
    Extension(ctx): Extension<Ctx>,
    Query(query): Query<ListQuery>,
) -> Result<Json<Vec<UserResponse>>, ApiError> {
    let users = UserBmc::list(&ctx, &state.mm, query.limit, query.offset).await?;

    let response: Vec<UserResponse> = users.into_iter().map(UserResponse::from).collect();

    Ok(Json(response))
}

// Get User
async fn get_user(
    State(state): State<AppState>,
    Extension(ctx): Extension<Ctx>,
    Path(user_id): Path<i64>,
) -> Result<Json<UserResponse>, ApiError> {
    let user = UserBmc::get(&ctx, &state.mm, user_id).await?;

    Ok(Json(UserResponse::from(user)))
}

// Create User
async fn create_user(
    State(state): State<AppState>,
    Extension(ctx): Extension<Ctx>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<UserResponse>), ApiError> {
    // Validate
    payload.validate()?;

    // Create
    let user_id = UserBmc::create(
        &ctx,
        &state.mm,
        UserForCreate {
            username: payload.username,
            email: payload.email,
            pwd_clear: payload.password,
        },
    )
    .await?;

    // Fetch created user
    let user = UserBmc::get(&ctx, &state.mm, user_id).await?;

    Ok((StatusCode::CREATED, Json(UserResponse::from(user))))
}

// Update User
async fn update_user(
    State(state): State<AppState>,
    Extension(ctx): Extension<Ctx>,
    Path(user_id): Path<i64>,
    Json(payload): Json<UpdateUserRequest>,
) -> Result<Json<UserResponse>, ApiError> {
    // Validate
    payload.validate()?;

    // Update
    UserBmc::update(
        &ctx,
        &state.mm,
        user_id,
        UserForUpdate {
            username: payload.username,
            email: payload.email,
        },
    )
    .await?;

    // Fetch updated user
    let user = UserBmc::get(&ctx, &state.mm, user_id).await?;

    Ok(Json(UserResponse::from(user)))
}

// Delete User
async fn delete_user(
    State(state): State<AppState>,
    Extension(ctx): Extension<Ctx>,
    Path(user_id): Path<i64>,
) -> Result<StatusCode, ApiError> {
    UserBmc::delete(&ctx, &state.mm, user_id).await?;

    Ok(StatusCode::NO_CONTENT)
}
```

**6. Main Application**

```rust
// web-server/src/main.rs

use axum::Router;
use lib_core::model::ModelManager;
use std::sync::Arc;

mod web;

#[derive(Clone)]
pub struct AppState {
    pub mm: ModelManager,
}

#[tokio::main]
async fn main() -> Result<()> {
    // Initialize tracing
    tracing_subscriber::fmt::init();

    // Create model manager
    let mm = ModelManager::new().await?;

    // Create app state
    let state = AppState { mm };

    // Build router
    let app = Router::new()
        .merge(web::routes_user::routes(state.clone()))
        .merge(web::routes_login::routes(state.clone()))
        .layer(/* middleware layers */);

    // Start server
    let addr = "0.0.0.0:3000".parse().unwrap();
    tracing::info!("Listening on {}", addr);

    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await?;

    Ok(())
}
```

---

## Authentication Flow

### Complete Login Implementation

```rust
// web-server/src/web/routes_login.rs

use axum::{
    extract::State,
    http::StatusCode,
    response::Json,
    routing::post,
    Router,
};
use serde::{Deserialize, Serialize};
use tower_cookies::{Cookie, Cookies};

use lib_core::ctx::Ctx;
use lib_core::model::{ModelManager, UserBmc};
use lib_auth::{pwd, token};

#[derive(Deserialize)]
pub struct LoginPayload {
    pub username: String,
    pub password: String,
}

#[derive(Serialize)]
pub struct LoginResponse {
    pub success: bool,
    pub message: String,
}

pub fn routes(state: AppState) -> Router {
    Router::new()
        .route("/api/login", post(api_login))
        .route("/api/logout", post(api_logout))
        .with_state(state)
}

async fn api_login(
    State(state): State<AppState>,
    cookies: Cookies,
    Json(payload): Json<LoginPayload>,
) -> Result<Json<LoginResponse>, ApiError> {
    // Get user by username
    let user = UserBmc::first_by_username(&Ctx::root_ctx(), &state.mm, &payload.username)
        .await?
        .ok_or(ApiError::LoginFailUsernameNotFound)?;

    // Validate password
    let scheme_status = pwd::validate_pwd(&payload.password, &user.pwd).await?;

    // Upgrade password if using old scheme
    if matches!(scheme_status, pwd::SchemeStatus::Outdated) {
        let new_pwd = pwd::hash_pwd(&payload.password).await?;
        UserBmc::update_password(&Ctx::root_ctx(), &state.mm, user.id, new_pwd).await?;
    }

    // Generate token
    let token = token::generate_web_token(user.id, user.token_salt)?;

    // Set cookie
    let mut cookie = Cookie::new("auth-token", token);
    cookie.set_http_only(true);
    cookie.set_path("/");
    cookies.add(cookie);

    Ok(Json(LoginResponse {
        success: true,
        message: "Login successful".to_string(),
    }))
}

async fn api_logout(cookies: Cookies) -> Result<StatusCode, ApiError> {
    cookies.remove(Cookie::named("auth-token"));
    Ok(StatusCode::OK)
}
```

### Authentication Middleware

```rust
// lib-web/src/middleware/mw_auth.rs

use axum::{
    extract::{Request, State},
    middleware::Next,
    response::Response,
};
use tower_cookies::Cookies;

use lib_core::ctx::Ctx;
use lib_core::model::ModelManager;
use lib_auth::token;

pub async fn mw_ctx_resolver(
    State(mm): State<ModelManager>,
    cookies: Cookies,
    mut request: Request,
    next: Next,
) -> Result<Response, Error> {
    // Extract token from cookie
    let token = cookies
        .get("auth-token")
        .map(|c| c.value().to_string());

    // Resolve context
    let result_ctx = match token {
        Some(token) => {
            // Validate token and create context
            token::validate_web_token(&mm, &token).await
        }
        None => {
            // No token - anonymous context
            Err(Error::TokenNotInCookie)
        }
    };

    // Store context in extensions (even if error for logging)
    if let Ok(ctx) = result_ctx {
        request.extensions_mut().insert(ctx);
    }

    Ok(next.run(request).await)
}
```

---

## File Upload Handling

### Multipart File Upload

```rust
use axum::{
    extract::Multipart,
    http::StatusCode,
};
use tokio::fs;
use tokio::io::AsyncWriteExt;

#[derive(Serialize)]
pub struct UploadResponse {
    pub file_id: String,
    pub file_name: String,
    pub file_size: u64,
}

async fn upload_file(
    Extension(ctx): Extension<Ctx>,
    mut multipart: Multipart,
) -> Result<(StatusCode, Json<UploadResponse>), ApiError> {
    let mut file_data = Vec::new();
    let mut file_name = String::new();

    while let Some(field) = multipart.next_field().await? {
        let name = field.name().unwrap_or("").to_string();

        if name == "file" {
            file_name = field.file_name().unwrap_or("unknown").to_string();
            file_data = field.bytes().await?.to_vec();
        }
    }

    // Generate unique file ID
    let file_id = uuid::Uuid::new_v4().to_string();
    let file_path = format!("./uploads/{}", file_id);

    // Save file
    let mut file = fs::File::create(&file_path).await?;
    file.write_all(&file_data).await?;

    let response = UploadResponse {
        file_id,
        file_name,
        file_size: file_data.len() as u64,
    };

    Ok((StatusCode::CREATED, Json(response)))
}
```

---

## Background Job Processing

### Simple Job Queue with Tokio Channels

```rust
use tokio::sync::mpsc;
use std::sync::Arc;

#[derive(Clone, Debug)]
pub enum Job {
    SendEmail { user_id: i64, template: String },
    ProcessImage { image_id: String },
    GenerateReport { report_id: i64 },
}

pub struct JobProcessor {
    tx: mpsc::Sender<Job>,
}

impl JobProcessor {
    pub fn new(mm: ModelManager) -> Self {
        let (tx, mut rx) = mpsc::channel::<Job>(100);

        // Spawn worker
        tokio::spawn(async move {
            while let Some(job) = rx.recv().await {
                if let Err(e) = process_job(&mm, job).await {
                    tracing::error!("Job processing failed: {:?}", e);
                }
            }
        });

        Self { tx }
    }

    pub async fn enqueue(&self, job: Job) -> Result<()> {
        self.tx.send(job).await
            .map_err(|_| Error::JobQueueFull)?;
        Ok(())
    }
}

async fn process_job(mm: &ModelManager, job: Job) -> Result<()> {
    match job {
        Job::SendEmail { user_id, template } => {
            // Send email logic
            tracing::info!("Sending email to user {}", user_id);
        }
        Job::ProcessImage { image_id } => {
            // Image processing logic
            tracing::info!("Processing image {}", image_id);
        }
        Job::GenerateReport { report_id } => {
            // Report generation logic
            tracing::info!("Generating report {}", report_id);
        }
    }

    Ok(())
}
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [architecture-overview.md](architecture-overview.md)
- [api-design.md](api-design.md)
- [security-patterns.md](security-patterns.md)
- [testing-guide.md](testing-guide.md)
