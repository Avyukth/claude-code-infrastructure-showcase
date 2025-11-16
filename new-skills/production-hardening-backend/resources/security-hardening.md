# Security Hardening - OWASP and Vulnerability Prevention

Comprehensive security hardening for Rust backend services, covering OWASP Top 10, dependency management, secret handling, and attack prevention.

## Table of Contents

- [Dependency Management and Auditing](#dependency-management-and-auditing)
- [Secret Management](#secret-management)
- [Input Validation and Sanitization](#input-validation-and-sanitization)
- [Authentication and Authorization](#authentication-and-authorization)
- [TLS Configuration](#tls-configuration)
- [OWASP Top 10 Mitigations](#owasp-top-10-mitigations)
- [API Security](#api-security)

---

## Dependency Management and Auditing

### Automated Vulnerability Scanning

```toml
# .cargo/deny.toml
[advisories]
db-path = "~/.cargo/advisory-db"
db-urls = ["https://github.com/rustsec/advisory-db"]
vulnerability = "deny"
unmaintained = "warn"
yanked = "deny"
notice = "warn"
ignore = []

[licenses]
unlicensed = "deny"
allow = [
    "MIT",
    "Apache-2.0",
    "BSD-3-Clause",
]
deny = [
    "GPL-3.0",
    "AGPL-3.0",
]

[bans]
multiple-versions = "warn"
wildcards = "deny"
highlight = "all"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
```

### CI/CD Security Scanning

```yaml
# .github/workflows/security.yml
name: Security Audit

on:
  push:
    branches: [main, develop]
  pull_request:
  schedule:
    - cron: '0 0 * * *'  # Daily at midnight

jobs:
  security-audit:
    name: Security Audit
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable

      - name: Install cargo-audit
        run: cargo install cargo-audit

      - name: Run cargo audit
        run: cargo audit --deny warnings

      - name: Install cargo-deny
        run: cargo install cargo-deny

      - name: Run cargo deny
        run: cargo deny check

      - name: Check for outdated dependencies
        run: |
          cargo install cargo-outdated
          cargo outdated --exit-code 1

  dependency-review:
    name: Dependency Review
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request'

    steps:
      - uses: actions/checkout@v4

      - name: Dependency Review
        uses: actions/dependency-review-action@v3
        with:
          fail-on-severity: moderate
```

### Dependency Pinning

```toml
# Cargo.toml - Pin critical dependencies

[dependencies]
# Pin exact versions for security-critical crates
tokio = "=1.35.1"
axum = "=0.7.4"
sqlx = "=0.7.3"

# Use tilde requirement for patch updates only
serde = "~1.0.195"
tracing = "~0.1.40"

# Workspace dependencies for consistency
[workspace.dependencies]
tokio = { version = "1.35", features = ["full"] }
```

### Supply Chain Security

```rust
// build.rs - Verify build-time dependencies

use std::process::Command;

fn main() {
    // Verify cargo-audit is installed
    let audit_check = Command::new("cargo")
        .args(&["audit", "--version"])
        .output()
        .expect("cargo-audit not installed");

    if !audit_check.status.success() {
        panic!("cargo-audit must be installed: cargo install cargo-audit");
    }

    // Run audit during build
    let audit_result = Command::new("cargo")
        .args(&["audit", "--json"])
        .output()
        .expect("Failed to run cargo audit");

    if !audit_result.status.success() {
        panic!("Security vulnerabilities found! Run `cargo audit` to see details.");
    }

    println!("cargo:rerun-if-changed=Cargo.lock");
}
```

---

## Secret Management

### Environment-Based Secrets

```rust
use secrecy::{Secret, ExposeSecret};
use std::env;

#[derive(Clone)]
pub struct Secrets {
    database_url: Secret<String>,
    jwt_secret: Secret<Vec<u8>>,
    api_key: Secret<String>,
}

impl Secrets {
    pub fn from_env() -> Result<Self> {
        Ok(Self {
            database_url: Secret::new(
                env::var("DATABASE_URL")
                    .map_err(|_| Error::MissingSecret("DATABASE_URL"))?
            ),
            jwt_secret: Secret::new(
                env::var("JWT_SECRET")
                    .map_err(|_| Error::MissingSecret("JWT_SECRET"))?
                    .into_bytes()
            ),
            api_key: Secret::new(
                env::var("API_KEY")
                    .map_err(|_| Error::MissingSecret("API_KEY"))?
            ),
        })
    }

    // Only expose when needed, never log
    pub fn database_url(&self) -> &str {
        self.database_url.expose_secret()
    }

    pub fn jwt_secret(&self) -> &[u8] {
        self.jwt_secret.expose_secret()
    }
}

// Implement Drop to zero memory
impl Drop for Secrets {
    fn drop(&mut self) {
        // secrecy crate handles zeroing on drop
    }
}
```

### HashiCorp Vault Integration

```rust
use vaultrs::{client::VaultClient, kv2};

pub struct VaultSecrets {
    client: VaultClient,
    mount: String,
}

impl VaultSecrets {
    pub async fn new(vault_addr: &str, token: Secret<String>) -> Result<Self> {
        let client = VaultClient::new(
            vault_addr,
            token.expose_secret(),
        )?;

        Ok(Self {
            client,
            mount: "secret".to_string(),
        })
    }

    pub async fn get_database_credentials(&self) -> Result<DatabaseCredentials> {
        let secret: HashMap<String, String> = kv2::read(
            &self.client,
            &self.mount,
            "database/postgres",
        )
        .await?;

        Ok(DatabaseCredentials {
            username: secret.get("username")
                .ok_or(Error::MissingSecret("username"))?
                .clone(),
            password: Secret::new(
                secret.get("password")
                    .ok_or(Error::MissingSecret("password"))?
                    .clone()
            ),
        })
    }

    pub async fn rotate_secret(&self, path: &str, new_value: Secret<String>) -> Result<()> {
        kv2::set(
            &self.client,
            &self.mount,
            path,
            &[("value", new_value.expose_secret())],
        )
        .await?;

        tracing::info!(path = %path, "Secret rotated successfully");
        Ok(())
    }
}
```

### Kubernetes Secrets

```yaml
# secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-server-secrets
  namespace: production
type: Opaque
stringData:
  database-url: "postgresql://user:pass@postgres:5432/db"
  jwt-secret: "your-secret-key-min-32-chars"
  api-key: "sk_live_..."
---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  template:
    spec:
      containers:
        - name: web-server
          envFrom:
            - secretRef:
                name: web-server-secrets
          # Mount secrets as files for higher security
          volumeMounts:
            - name: secrets
              mountPath: "/secrets"
              readOnly: true
      volumes:
        - name: secrets
          secret:
            secretName: web-server-secrets
            defaultMode: 0400  # Read-only for owner
```

### Secure Secret Loading from Files

```rust
use std::fs;
use std::path::Path;

pub fn load_secret_from_file(path: &Path) -> Result<Secret<String>> {
    let content = fs::read_to_string(path)
        .map_err(|e| Error::SecretLoadError(path.display().to_string(), e))?;

    // Trim whitespace and newlines
    let secret = content.trim().to_string();

    if secret.is_empty() {
        return Err(Error::EmptySecret(path.display().to_string()));
    }

    Ok(Secret::new(secret))
}

// Load all secrets from directory
pub fn load_secrets_from_dir(dir: &Path) -> Result<HashMap<String, Secret<String>>> {
    let mut secrets = HashMap::new();

    for entry in fs::read_dir(dir)? {
        let entry = entry?;
        let path = entry.path();

        if path.is_file() {
            let key = path
                .file_name()
                .and_then(|n| n.to_str())
                .ok_or_else(|| Error::InvalidSecretPath(path.display().to_string()))?
                .to_string();

            secrets.insert(key, load_secret_from_file(&path)?);
        }
    }

    Ok(secrets)
}
```

---

## Input Validation and Sanitization

### Request Validation with validator

```rust
use validator::{Validate, ValidationError};
use serde::Deserialize;

#[derive(Debug, Deserialize, Validate)]
pub struct CreateUserRequest {
    #[validate(length(min = 3, max = 50))]
    #[validate(regex = "USERNAME_REGEX")]
    username: String,

    #[validate(email)]
    email: String,

    #[validate(length(min = 12))]
    #[validate(custom = "validate_password_strength")]
    password: String,

    #[validate(url)]
    #[validate(custom = "validate_safe_url")]
    website: Option<String>,
}

lazy_static! {
    static ref USERNAME_REGEX: Regex = Regex::new(r"^[a-zA-Z0-9_-]+$").unwrap();
}

fn validate_password_strength(password: &str) -> Result<(), ValidationError> {
    let has_uppercase = password.chars().any(|c| c.is_uppercase());
    let has_lowercase = password.chars().any(|c| c.is_lowercase());
    let has_digit = password.chars().any(|c| c.is_numeric());
    let has_special = password.chars().any(|c| "!@#$%^&*()_+-=[]{}|;:,.<>?".contains(c));

    if has_uppercase && has_lowercase && has_digit && has_special {
        Ok(())
    } else {
        Err(ValidationError::new("weak_password"))
    }
}

fn validate_safe_url(url: &str) -> Result<(), ValidationError> {
    let parsed = url::Url::parse(url)
        .map_err(|_| ValidationError::new("invalid_url"))?;

    // Block private IP ranges and localhost
    if let Some(host) = parsed.host_str() {
        if host == "localhost"
            || host.starts_with("127.")
            || host.starts_with("192.168.")
            || host.starts_with("10.")
            || host.starts_with("172.16.") {
            return Err(ValidationError::new("private_url"));
        }
    }

    // Only allow HTTPS
    if parsed.scheme() != "https" {
        return Err(ValidationError::new("insecure_scheme"));
    }

    Ok(())
}
```

### SQL Injection Prevention

```rust
// ✅ ALWAYS use parameterized queries

// ❌ NEVER EVER DO THIS
pub async fn unsafe_query(pool: &PgPool, username: &str) -> Result<User> {
    let query = format!("SELECT * FROM users WHERE username = '{}'", username);
    // SQL INJECTION VULNERABILITY!
    sqlx::query_as::<_, User>(&query)
        .fetch_one(pool)
        .await
        .map_err(Into::into)
}

// ✅ CORRECT: Parameterized query with SQLx
pub async fn safe_query(pool: &PgPool, username: &str) -> Result<User> {
    sqlx::query_as!(
        User,
        "SELECT * FROM users WHERE username = $1",
        username
    )
    .fetch_one(pool)
    .await
    .map_err(Into::into)
}

// ✅ CORRECT: Type-safe with Sea-Query
pub async fn safe_query_builder(pool: &PgPool, username: &str) -> Result<User> {
    let (sql, values) = Query::select()
        .from(Users::Table)
        .columns([Users::Id, Users::Username, Users::Email])
        .and_where(Expr::col(Users::Username).eq(username))
        .build_sqlx(PostgresQueryBuilder);

    sqlx::query_as_with(&sql, values)
        .fetch_one(pool)
        .await
        .map_err(Into::into)
}
```

### XSS Prevention

```rust
use ammonia::clean;

pub fn sanitize_html(input: &str) -> String {
    // Strip all HTML tags and attributes
    clean(input)
}

pub fn sanitize_html_limited(input: &str) -> String {
    // Allow only safe tags
    let mut builder = ammonia::Builder::default();
    builder
        .allowed_tags(hashset!["p", "br", "strong", "em", "ul", "ol", "li"])
        .allowed_attributes(hashmap![]);

    builder.clean(input).to_string()
}

// Use in handlers
pub async fn create_post(
    ctx: &Ctx,
    mm: &ModelManager,
    data: PostForCreate,
) -> Result<i64> {
    let sanitized_content = sanitize_html_limited(&data.content);

    PostBmc::create(ctx, mm, PostForCreate {
        title: data.title,
        content: sanitized_content,
        user_id: ctx.user_id(),
    })
    .await
}
```

### Path Traversal Prevention

```rust
use std::path::{Path, PathBuf};

pub fn safe_path_join(base: &Path, user_input: &str) -> Result<PathBuf> {
    // Normalize and validate path
    let requested_path = Path::new(user_input);

    // Reject absolute paths
    if requested_path.is_absolute() {
        return Err(Error::InvalidPath("Absolute paths not allowed"));
    }

    // Reject paths with ".."
    for component in requested_path.components() {
        if matches!(component, std::path::Component::ParentDir) {
            return Err(Error::InvalidPath("Parent directory traversal not allowed"));
        }
    }

    // Join and canonicalize
    let full_path = base.join(requested_path);
    let canonical = full_path.canonicalize()
        .map_err(|e| Error::PathError(e))?;

    // Ensure result is still under base directory
    if !canonical.starts_with(base) {
        return Err(Error::InvalidPath("Path escapes base directory"));
    }

    Ok(canonical)
}

// Usage
pub async fn serve_file(
    Path(filename): Path<String>,
) -> Result<Response, ApiError> {
    let base_dir = Path::new("/var/www/uploads");
    let safe_path = safe_path_join(base_dir, &filename)?;

    let file = tokio::fs::read(safe_path).await?;
    Ok(Response::new(file.into()))
}
```

---

## Authentication and Authorization

### JWT Authentication

```rust
use jsonwebtoken::{encode, decode, Header, Validation, EncodingKey, DecodingKey};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    sub: i64,          // Subject (user ID)
    exp: i64,          // Expiry
    iat: i64,          // Issued at
    roles: Vec<String>, // User roles
    permissions: Vec<String>,
}

pub fn generate_jwt(user_id: i64, roles: Vec<String>, secret: &[u8]) -> Result<String> {
    let now = chrono::Utc::now().timestamp();
    let expiration = now + 3600;  // 1 hour

    let claims = Claims {
        sub: user_id,
        exp: expiration,
        iat: now,
        roles,
        permissions: vec![],  // Loaded based on roles
    };

    encode(
        &Header::default(),
        &claims,
        &EncodingKey::from_secret(secret),
    )
    .map_err(|e| Error::JwtError(e))
}

pub fn validate_jwt(token: &str, secret: &[u8]) -> Result<Claims> {
    let mut validation = Validation::default();
    validation.leeway = 60;  // 1 minute clock skew tolerance

    decode::<Claims>(
        token,
        &DecodingKey::from_secret(secret),
        &validation,
    )
    .map(|data| data.claims)
    .map_err(|e| Error::JwtError(e))
}

// Middleware
pub async fn jwt_auth_middleware(
    mut request: Request,
    next: Next,
    State(state): State<AppState>,
) -> Result<Response, ApiError> {
    let auth_header = request
        .headers()
        .get(AUTHORIZATION)
        .and_then(|h| h.to_str().ok())
        .ok_or(Error::MissingAuth)?;

    let token = auth_header
        .strip_prefix("Bearer ")
        .ok_or(Error::InvalidAuthFormat)?;

    let claims = validate_jwt(token, &state.secrets.jwt_secret())?;

    // Check expiration
    let now = chrono::Utc::now().timestamp();
    if claims.exp < now {
        return Err(Error::TokenExpired);
    }

    // Create context
    let ctx = Ctx::new(claims.sub, claims.roles, claims.permissions)?;
    request.extensions_mut().insert(ctx);

    Ok(next.run(request).await)
}
```

### Role-Based Access Control (RBAC)

```rust
#[derive(Clone, Debug, PartialEq)]
pub enum Role {
    Admin,
    User,
    ReadOnly,
}

#[derive(Clone, Debug, PartialEq)]
pub enum Permission {
    // User permissions
    ViewUsers,
    CreateUsers,
    UpdateUsers,
    DeleteUsers,

    // Post permissions
    ViewPosts,
    CreatePosts,
    UpdateOwnPosts,
    UpdateAnyPosts,
    DeleteOwnPosts,
    DeleteAnyPosts,

    // Admin permissions
    ManageRoles,
    ViewAuditLogs,
}

impl Role {
    pub fn permissions(&self) -> Vec<Permission> {
        match self {
            Role::Admin => vec![
                Permission::ViewUsers,
                Permission::CreateUsers,
                Permission::UpdateUsers,
                Permission::DeleteUsers,
                Permission::ViewPosts,
                Permission::CreatePosts,
                Permission::UpdateAnyPosts,
                Permission::DeleteAnyPosts,
                Permission::ManageRoles,
                Permission::ViewAuditLogs,
            ],
            Role::User => vec![
                Permission::ViewUsers,
                Permission::ViewPosts,
                Permission::CreatePosts,
                Permission::UpdateOwnPosts,
                Permission::DeleteOwnPosts,
            ],
            Role::ReadOnly => vec![
                Permission::ViewUsers,
                Permission::ViewPosts,
            ],
        }
    }
}

#[derive(Clone, Debug)]
pub struct Ctx {
    user_id: i64,
    roles: Vec<Role>,
    permissions: Vec<Permission>,
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

    pub fn has_role(&self, role: Role) -> bool {
        self.roles.contains(&role)
    }
}

// Usage in handlers
pub async fn delete_user(
    Extension(ctx): Extension<Ctx>,
    Path(user_id): Path<i64>,
    State(state): State<AppState>,
) -> Result<StatusCode, ApiError> {
    // Authorization check
    ctx.require_permission(Permission::DeleteUsers)?;

    UserBmc::delete(&ctx, &state.mm, user_id).await?;

    Ok(StatusCode::NO_CONTENT)
}
```

---

## TLS Configuration

### Secure TLS Server

```rust
use rustls::{ServerConfig, Certificate, PrivateKey};
use rustls::server::NoClientAuth;

pub fn create_tls_config(
    cert_path: &Path,
    key_path: &Path,
) -> Result<ServerConfig> {
    let certs = load_certs(cert_path)?;
    let key = load_private_key(key_path)?;

    let config = ServerConfig::builder()
        .with_safe_default_cipher_suites()
        .with_safe_default_kx_groups()
        .with_protocol_versions(&[&rustls::version::TLS13])  // TLS 1.3 only
        .map_err(|e| Error::TlsConfigError(e))?
        .with_no_client_auth()
        .with_single_cert(certs, key)
        .map_err(|e| Error::TlsConfigError(e))?;

    Ok(config)
}

fn load_certs(path: &Path) -> Result<Vec<Certificate>> {
    let file = std::fs::File::open(path)?;
    let mut reader = std::io::BufReader::new(file);

    rustls_pemfile::certs(&mut reader)
        .map(|certs| certs.into_iter().map(Certificate).collect())
        .map_err(|_| Error::TlsCertLoadError)
}

fn load_private_key(path: &Path) -> Result<PrivateKey> {
    let file = std::fs::File::open(path)?;
    let mut reader = std::io::BufReader::new(file);

    let keys = rustls_pemfile::pkcs8_private_keys(&mut reader)
        .map_err(|_| Error::TlsKeyLoadError)?;

    keys.into_iter()
        .next()
        .map(PrivateKey)
        .ok_or(Error::TlsKeyLoadError)
}
```

### Enforce HTTPS

```rust
// Middleware to redirect HTTP to HTTPS
pub async fn enforce_https_middleware(
    request: Request,
    next: Next,
) -> Result<Response, ApiError> {
    let scheme = request
        .uri()
        .scheme_str()
        .unwrap_or("http");

    if scheme != "https" {
        // Redirect to HTTPS
        let host = request
            .headers()
            .get(HOST)
            .and_then(|h| h.to_str().ok())
            .ok_or(Error::MissingHost)?;

        let redirect_url = format!(
            "https://{}{}",
            host,
            request.uri().path_and_query()
                .map(|p| p.as_str())
                .unwrap_or("/")
        );

        return Ok(Response::builder()
            .status(StatusCode::MOVED_PERMANENTLY)
            .header(LOCATION, redirect_url)
            .body(Body::empty())?);
    }

    Ok(next.run(request).await)
}
```

---

## OWASP Top 10 Mitigations

### A01: Broken Access Control

```rust
// Resource-level authorization
pub async fn update_post(
    Extension(ctx): Extension<Ctx>,
    Path(post_id): Path<i64>,
    Json(data): Json<UpdatePostRequest>,
    State(state): State<AppState>,
) -> Result<Json<Post>, ApiError> {
    // Get post
    let post = PostBmc::get(&ctx, &state.mm, post_id).await?;

    // Check ownership OR admin permission
    if post.user_id != ctx.user_id()
        && !ctx.has_permission(Permission::UpdateAnyPosts) {
        return Err(Error::PermissionDenied {
            required: "post ownership or UpdateAnyPosts".to_string(),
        });
    }

    // Perform update
    let updated = PostBmc::update(&ctx, &state.mm, post_id, data).await?;

    Ok(Json(updated))
}
```

### A02: Cryptographic Failures

```rust
// Encrypt sensitive data at rest
use aes_gcm::{Aes256Gcm, Key, Nonce};
use aes_gcm::aead::{Aead, NewAead};

pub fn encrypt_data(plaintext: &[u8], key: &[u8]) -> Result<Vec<u8>> {
    let key = Key::from_slice(key);
    let cipher = Aes256Gcm::new(key);

    let nonce = Nonce::from_slice(&[0u8; 12]);  // Use random nonce in production
    cipher.encrypt(nonce, plaintext)
        .map_err(|e| Error::EncryptionError(e))
}

pub fn decrypt_data(ciphertext: &[u8], key: &[u8]) -> Result<Vec<u8>> {
    let key = Key::from_slice(key);
    let cipher = Aes256Gcm::new(key);

    let nonce = Nonce::from_slice(&[0u8; 12]);
    cipher.decrypt(nonce, ciphertext)
        .map_err(|e| Error::DecryptionError(e))
}
```

### A03: Injection (SQL, Command, LDAP)

```rust
// Already covered in SQL Injection Prevention section

// Command Injection Prevention
pub async fn process_file(filename: &str) -> Result<Output> {
    // ❌ NEVER use shell commands with user input
    // let output = Command::new("sh")
    //     .arg("-c")
    //     .arg(format!("cat {}", filename))  // COMMAND INJECTION!
    //     .output()?;

    // ✅ Use direct file I/O
    let content = tokio::fs::read_to_string(filename).await?;

    Ok(Output { content })
}
```

### A04: Insecure Design

```rust
// Rate limiting per user
use tower::limit::RateLimit;
use tower_governor::{GovernorLayer, GovernorConfig};

pub fn rate_limit_layer() -> GovernorLayer {
    let config = Box::new(
        GovernorConfig::default()
            .per_second(10)     // 10 requests per second
            .burst_size(20)     // Burst of 20
            .finish()
            .unwrap()
    );

    GovernorLayer {
        config: Box::leak(config),
    }
}
```

### A05: Security Misconfiguration

```rust
// Secure defaults and configuration validation
#[derive(Deserialize)]
pub struct SecurityConfig {
    #[serde(default = "default_tls_enabled")]
    pub tls_enabled: bool,

    #[serde(default = "default_min_tls_version")]
    pub min_tls_version: String,

    #[serde(default = "default_allowed_origins")]
    pub allowed_origins: Vec<String>,

    #[serde(default = "default_max_request_size")]
    pub max_request_size: usize,
}

fn default_tls_enabled() -> bool { true }
fn default_min_tls_version() -> String { "1.3".to_string() }
fn default_allowed_origins() -> Vec<String> { vec![] }  // Deny all by default
fn default_max_request_size() -> usize { 1024 * 1024 }  // 1MB

impl SecurityConfig {
    pub fn validate(&self) -> Result<()> {
        if !self.tls_enabled {
            return Err(Error::InsecureConfig("TLS must be enabled in production"));
        }

        if self.allowed_origins.is_empty() {
            return Err(Error::InsecureConfig("CORS origins must be configured"));
        }

        Ok(())
    }
}
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [Core Principles](core-principles.md)
- [Fault Tolerance](fault-tolerance.md)
- [Monitoring](monitoring.md)
