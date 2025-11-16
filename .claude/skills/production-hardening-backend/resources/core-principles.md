# Core Hardening Principles - Defense in Depth for Rust Backends

Foundational principles for building production-grade Rust backend systems with kernel-level reliability and security.

## Table of Contents

- [Least Privilege](#least-privilege)
- [Defense in Depth](#defense-in-depth)
- [Fail Secure](#fail-secure)
- [Zero Trust Architecture](#zero-trust-architecture)
- [Minimal Attack Surface](#minimal-attack-surface)
- [Immutable Infrastructure](#immutable-infrastructure)
- [Security by Design](#security-by-design)

---

## Least Privilege

### Principle

Grant the minimum permissions necessary for operation at every level: OS, application, network, and data access.

### Application-Level Implementation

#### Process User and Permissions

```dockerfile
# Dockerfile - Never run as root
FROM rust:1.75-slim as builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN useradd -m -u 1001 -s /bin/bash appuser
USER appuser
WORKDIR /home/appuser

COPY --from=builder /app/target/release/web-server /usr/local/bin/
CMD ["web-server"]
```

#### File System Permissions

```rust
use std::fs::OpenOptions;
use std::os::unix::fs::OpenOptionsExt;

// Create files with restricted permissions (0600 = owner read/write only)
pub fn write_secure_file(path: &Path, data: &[u8]) -> Result<()> {
    OpenOptions::new()
        .create(true)
        .write(true)
        .mode(0o600)  // Owner read/write only
        .open(path)?
        .write_all(data)?;

    Ok(())
}
```

#### Database Access with Minimal Grants

```rust
// Use read-only connections for queries
pub struct DatabaseConnections {
    read_pool: PgPool,   // SELECT only
    write_pool: PgPool,  // INSERT, UPDATE, DELETE
}

impl DatabaseConnections {
    pub async fn new(config: &DbConfig) -> Result<Self> {
        let read_pool = PgPoolOptions::new()
            .after_connect(|conn, _meta| Box::pin(async move {
                // Set session to read-only
                sqlx::query("SET SESSION CHARACTERISTICS AS TRANSACTION READ ONLY")
                    .execute(conn)
                    .await?;
                Ok(())
            }))
            .connect(&config.read_url)
            .await?;

        let write_pool = PgPoolOptions::new()
            .connect(&config.write_url)
            .await?;

        Ok(Self { read_pool, write_pool })
    }
}
```

### Network-Level Implementation

#### Kubernetes RBAC

```yaml
# service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-server
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind:

 Role
metadata:
  name: web-server-role
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get"]  # Read-only for pod info
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: web-server-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: web-server-role
subjects:
  - kind: ServiceAccount
    name: web-server
```

#### Network Policies (Zero Trust Networking)

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-server-policy
spec:
  podSelector:
    matchLabels:
      app: web-server
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: nginx-ingress
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:  # Allow DNS
        - namespaceSelector:
            matchLabels:
              name: kube-system
      ports:
        - protocol: UDP
          port: 53
```

---

## Defense in Depth

### Principle

Multiple layers of security controls so that if one layer fails, others provide protection.

### Layered Security Architecture

```
┌─────────────────────────────────────────┐
│  Layer 1: Network Perimeter             │
│  - WAF, DDoS protection, Firewall       │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│  Layer 2: TLS/Authentication            │
│  - mTLS, JWT validation, Rate limiting  │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│  Layer 3: Application Authorization     │
│  - RBAC, Resource-level permissions     │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│  Layer 4: Input Validation              │
│  - Schema validation, Sanitization      │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│  Layer 5: Business Logic                │
│  - Type-safe operations, Audit logging  │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│  Layer 6: Data Access                   │
│  - Parameterized queries, Row-level     │
└─────────────────────────────────────────┘
           ↓
┌─────────────────────────────────────────┐
│  Layer 7: Data Storage                  │
│  - Encryption at rest, Backups          │
└─────────────────────────────────────────┘
```

### Implementation Example

```rust
// Layer 1-2: TLS and Authentication (middleware)
async fn secure_handler(
    // Layer 2: Verify JWT
    Extension(ctx): Extension<Ctx>,
    // Layer 3: Check permissions
    Extension(permissions): Extension<Vec<Permission>>,
    // Layer 4: Validate input
    ValidatedJson(payload): ValidatedJson<CreateUserRequest>,
    State(state): State<AppState>,
) -> Result<Json<User>, ApiError> {
    // Layer 3: Authorization check
    ctx.require_permission(Permission::CreateUsers)?;

    // Layer 5: Business logic with audit
    tracing::info!(
        user_id = %ctx.user_id(),
        action = "create_user",
        "User creation requested"
    );

    // Layer 6: Data access with parameterized query
    let user_id = UserBmc::create(&ctx, &state.mm, UserForCreate {
        username: payload.username,
        email: payload.email,
        pwd_clear: payload.password,  // Will be hashed (Layer 7)
    })
    .await?;

    // Layer 7: Audit log
    AuditLog::log_action(&ctx, "user.create", user_id).await?;

    let user = UserBmc::get(&ctx, &state.mm, user_id).await?;
    Ok(Json(user))
}
```

---

## Fail Secure

### Principle

When errors occur, fail to a secure state rather than bypassing security controls.

### Error Handling Patterns

```rust
// ❌ INSECURE: Defaults to granting access on error
pub async fn check_permission(user_id: i64, resource: &str) -> bool {
    match get_user_permissions(user_id).await {
        Ok(perms) => perms.contains(resource),
        Err(_) => true,  // DANGEROUS: Grants access on error!
    }
}

// ✅ SECURE: Defaults to denying access
pub async fn check_permission(user_id: i64, resource: &str) -> Result<bool> {
    let perms = get_user_permissions(user_id).await?;  // Propagate error
    Ok(perms.contains(resource))
}

// Usage
match check_permission(user_id, "admin").await {
    Ok(true) => {
        // Grant access
    }
    Ok(false) | Err(_) => {
        // Deny access (fail secure)
        return Err(Error::PermissionDenied);
    }
}
```

### Circuit Breaker Fail-Secure

```rust
use failsafe::{CircuitBreaker, Config, failure_policy};

pub struct SecureService {
    client: HttpClient,
    circuit_breaker: CircuitBreaker,
}

impl SecureService {
    pub async fn call_external_api(&self, request: Request) -> Result<Response> {
        // Circuit breaker prevents cascading failures
        self.circuit_breaker
            .call(|| async {
                self.client.send(request).await
            })
            .await
            .map_err(|e| match e {
                // Fail secure: Log and return error, don't bypass
                failsafe::Error::Rejected => {
                    tracing::error!("Circuit breaker open, request rejected");
                    Error::ServiceUnavailable
                }
                failsafe::Error::Inner(err) => {
                    tracing::error!("External API error: {}", err);
                    Error::ExternalServiceError
                }
            })
    }
}
```

### Database Transaction Fail-Secure

```rust
pub async fn transfer_funds(
    from_account: i64,
    to_account: i64,
    amount: Decimal,
    pool: &PgPool,
) -> Result<()> {
    let mut tx = pool.begin().await?;

    // Debit source account
    let debit_result = sqlx::query!(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2 AND balance >= $1",
        amount,
        from_account
    )
    .execute(&mut *tx)
    .await?;

    // Fail secure: If debit didn't affect any rows, insufficient funds
    if debit_result.rows_affected() == 0 {
        tx.rollback().await?;  // Explicit rollback
        return Err(Error::InsufficientFunds);
    }

    // Credit destination account
    sqlx::query!(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount,
        to_account
    )
    .execute(&mut *tx)
    .await?;

    // Only commit if all operations succeed
    tx.commit().await?;

    Ok(())
}
```

---

## Zero Trust Architecture

### Principle

Never trust, always verify. Assume breach and verify every request.

### Implementation

#### Mutual TLS (mTLS)

```rust
use rustls::{ServerConfig, Certificate, PrivateKey};
use rustls::server::AllowAnyAuthenticatedClient;

pub fn create_mtls_config(
    cert_path: &Path,
    key_path: &Path,
    ca_cert_path: &Path,
) -> Result<ServerConfig> {
    let certs = load_certs(cert_path)?;
    let key = load_private_key(key_path)?;
    let ca_cert = load_certs(ca_cert_path)?;

    let mut root_store = rustls::RootCertStore::empty();
    for cert in ca_cert {
        root_store.add(&cert)?;
    }

    let client_auth = AllowAnyAuthenticatedClient::new(root_store);

    let config = ServerConfig::builder()
        .with_safe_defaults()
        .with_client_cert_verifier(client_auth)
        .with_single_cert(certs, key)?;

    Ok(config)
}
```

#### Request Verification

```rust
// Verify every request, even from internal services
async fn verify_request_middleware(
    request: Request,
    next: Next,
) -> Result<Response, ApiError> {
    // 1. Verify TLS client certificate
    let cert = request
        .extensions()
        .get::<ClientCertificate>()
        .ok_or(Error::MissingClientCert)?;

    // 2. Verify JWT token
    let auth_header = request
        .headers()
        .get(AUTHORIZATION)
        .ok_or(Error::MissingAuth)?;

    let token = extract_bearer_token(auth_header)?;
    let ctx = validate_jwt(token).await?;

    // 3. Verify source IP (allowlist)
    let source_ip = request
        .extensions()
        .get::<SourceIp>()
        .ok_or(Error::MissingSourceIp)?;

    if !is_ip_allowed(source_ip) {
        return Err(Error::IPNotAllowed);
    }

    // 4. Verify request signature (HMAC)
    verify_request_signature(&request)?;

    // 5. Add context to request
    let mut request = request;
    request.extensions_mut().insert(ctx);

    Ok(next.run(request).await)
}
```

---

## Minimal Attack Surface

### Principle

Remove all unnecessary code, dependencies, services, and ports to reduce potential vulnerabilities.

### Dependency Minimization

```toml
# Cargo.toml - Only include necessary features

[dependencies]
# ❌ BAD: Pull in everything
tokio = { version = "1", features = ["full"] }

# ✅ GOOD: Minimal features
tokio = { version = "1", features = ["rt-multi-thread", "macros", "time", "net"] }

# ❌ BAD: Unused dependency
uuid = { version = "1", features = ["v4", "v5", "serde", "macro-diagnostics"] }

# ✅ GOOD: Only what you use
uuid = { version = "1", features = ["v4", "serde"] }
```

### Audit Dependencies

```bash
# Install cargo-audit
cargo install cargo-audit

# Check for known vulnerabilities
cargo audit

# Check for unmaintained dependencies
cargo audit --deny unmaintained

# Use cargo-deny for policy enforcement
cargo install cargo-deny

# Check licenses, advisories, and bans
cargo deny check
```

### Remove Unused Code

```rust
// Use #[cfg] to exclude debug/dev code from production

#[cfg(debug_assertions)]
pub mod debug_endpoints {
    // Only compiled in debug builds
    pub fn debug_info() -> String {
        format!("Debug info: {:?}", std::env::vars())
    }
}

#[cfg(not(debug_assertions))]
pub mod debug_endpoints {
    // Empty in release builds
}
```

### Minimal Docker Image

```dockerfile
# Multi-stage build for minimal attack surface
FROM rust:1.75-slim as builder
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

# Distroless image (no shell, no package manager)
FROM gcr.io/distroless/static-debian12
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/web-server /
USER nobody
EXPOSE 8080
CMD ["/web-server"]
```

---

## Immutable Infrastructure

### Principle

Never modify running systems. Deploy new versions and replace old ones.

### Implementation

```rust
// Version every deployment
#[derive(Serialize)]
pub struct AppInfo {
    version: &'static str,
    commit_sha: &'static str,
    build_time: &'static str,
}

impl AppInfo {
    pub fn new() -> Self {
        Self {
            version: env!("CARGO_PKG_VERSION"),
            commit_sha: env!("GIT_COMMIT_SHA"),  // Set in build.rs
            build_time: env!("BUILD_TIME"),
        }
    }
}

// Health endpoint includes version
pub async fn health_check() -> Json<HealthResponse> {
    Json(HealthResponse {
        status: "healthy",
        version: AppInfo::new(),
    })
}
```

### Kubernetes Deployment Strategy

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web-server
        version: "v1.2.3"  # Immutable version label
    spec:
      containers:
        - name: web-server
          image: myregistry/web-server:v1.2.3-abc123  # Immutable tag
          imagePullPolicy: Always
          # No ssh, no shell access
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1001
```

---

## Security by Design

### Principle

Security requirements from the start, not bolted on later.

### Threat Modeling

```rust
// Document threats in code
/// # Security Considerations
///
/// This endpoint handles sensitive user data and must:
/// 1. Validate JWT token (authentication)
/// 2. Check user permissions (authorization)
/// 3. Validate all input (SQL injection prevention)
/// 4. Rate limit requests (DoS prevention)
/// 5. Audit log all actions (compliance)
///
/// Threat Model:
/// - Attacker tries SQL injection → Mitigated by parameterized queries
/// - Attacker tries privilege escalation → Mitigated by RBAC
/// - Attacker tries DoS → Mitigated by rate limiting
pub async fn update_user(
    Extension(ctx): Extension<Ctx>,
    Path(user_id): Path<i64>,
    ValidatedJson(payload): ValidatedJson<UpdateUserRequest>,
    State(state): State<AppState>,
) -> Result<Json<User>, ApiError> {
    // Implementation...
}
```

### Secure Defaults

```rust
// Use secure defaults in configuration
#[derive(Deserialize)]
pub struct ServerConfig {
    #[serde(default = "default_host")]
    pub host: String,

    #[serde(default = "default_port")]
    pub port: u16,

    #[serde(default = "default_tls_enabled")]
    pub tls_enabled: bool,

    #[serde(default = "default_max_connections")]
    pub max_connections: u32,
}

fn default_host() -> String {
    "127.0.0.1".to_string()  // Localhost by default, not 0.0.0.0
}

fn default_tls_enabled() -> bool {
    true  // TLS enabled by default
}

fn default_max_connections() -> u32 {
    100  // Conservative default
}
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [Security Hardening](security-hardening.md)
- [Fault Tolerance](fault-tolerance.md)
- [Monitoring](monitoring.md)
