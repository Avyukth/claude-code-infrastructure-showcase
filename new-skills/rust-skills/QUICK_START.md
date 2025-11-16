# Quick Start Guide - Rust Backend Development Skill

Get started with the Rust Backend Development Guidelines in 5 minutes.

---

## 📚 What is This?

A comprehensive skill package for building production-ready Rust backend services using:
- **Axum** for web framework
- **SQLx** for database operations
- **Tokio** for async runtime
- **Rust10x** architectural patterns

---

## 🚀 Installation

```bash
# Copy to your Claude Code project
cp -r rust-skill /path/to/your/project/.claude/skills/

# Or create a symbolic link
ln -s $(pwd)/rust-skill /path/to/your/project/.claude/skills/
```

---

## 📖 5-Minute Tour

### 1. Start with the Hub (2 min)

Read **SKILL.md** for:
- 10 core Rust principles
- Quick start checklist for new features
- HTTP status code reference
- Common imports cheat sheet

### 2. Check the Architecture (1 min)

Open **resources/architecture-overview.md** to understand:
- Multi-crate workspace structure
- Backend Model Controller (BMC) pattern
- Request lifecycle

### 3. Copy Your First Example (2 min)

From **resources/complete-examples.md**, copy the CRUD API example:

```rust
// Handler
async fn create_user(
    State(state): State<AppState>,
    Extension(ctx): Extension<Ctx>,
    Json(payload): Json<CreateUserRequest>,
) -> Result<(StatusCode, Json<UserResponse>), ApiError> {
    payload.validate()?;

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

    let user = UserBmc::get(&ctx, &state.mm, user_id).await?;

    Ok((StatusCode::CREATED, Json(UserResponse::from(user))))
}
```

---

## 🎯 Common Tasks

### Creating a New API Endpoint

1. Read: **resources/api-design.md** (Axum patterns)
2. Read: **resources/architecture-overview.md** (BMC pattern)
3. Copy: **resources/complete-examples.md** (CRUD example)

### Adding Authentication

1. Read: **resources/security-patterns.md** (Multi-scheme passwords)
2. Copy: Token generation and validation examples
3. Implement: Context middleware

### Writing Tests

1. Read: **resources/testing-guide.md** (Testing strategies)
2. Copy: Integration test example
3. Run: `cargo test`

### Deploying to Production

1. Read: **resources/deployment-guide.md** (Docker & CI/CD)
2. Copy: Multi-stage Dockerfile
3. Copy: GitHub Actions workflow

---

## 📁 File Reference

### Core Files
- **SKILL.md** - Main hub (start here)
- **README.md** - Overview and status

### Architecture
- **architecture-overview.md** - Workspace & BMC pattern
- **project-structure.md** - Crate organization

### Development
- **error-handling.md** - Custom error types
- **async-patterns.md** - Tokio & async/await
- **database-sqlx.md** - SQLx + Sea-Query
- **api-design.md** - Axum routing

### Quality & Security
- **testing-guide.md** - Unit & integration tests
- **security-patterns.md** - Auth & authorization

### Production
- **complete-examples.md** - Full working code
- **performance-optimization.md** - Profiling & benchmarks
- **deployment-guide.md** - Docker, CI/CD, monitoring

---

## 🔑 Key Patterns to Know

### 1. Backend Model Controller (BMC)

```rust
pub struct UserBmc;

impl UserBmc {
    pub async fn create(ctx: &Ctx, mm: &ModelManager, data: UserForCreate) -> Result<i64> {
        // Implementation
    }
}
```

**Why**: Zero-sized type, stateless, context-based authorization

### 2. Context Pattern

```rust
#[derive(Clone, Debug)]
pub struct Ctx {
    user_id: i64,
}
```

**Why**: Future ACL implementation, audit trails

### 3. Error Handling with derive_more

```rust
#[derive(Debug, From)]
pub enum Error {
    EntityNotFound { entity: &'static str, id: i64 },
    
    #[from]
    Sqlx(sqlx::Error),
}
```

**Why**: Selective auto-conversion, clean error types

### 4. Compile-Time SQL Verification

```rust
let user = sqlx::query_as!(
    User,
    "SELECT * FROM users WHERE id = $1",
    id
)
.fetch_optional(pool)
.await?;
```

**Why**: Catch SQL errors at compile time

---

## 💡 Quick Tips

### DO ✅
- Use newtypes: `UserId(i64)` not `i64`
- Use `?` operator for error propagation
- Validate input with `validator` crate
- Use `#[tokio::test]` for async tests
- Parameterize all SQL queries

### DON'T ❌
- Use `.unwrap()` in production code
- Use primitive types for domain IDs
- Concatenate SQL strings
- Hold locks across `.await` points
- Ignore clippy warnings

---

## 🛠️ Development Workflow

### 1. New Feature
```bash
# 1. Create domain model (lib-core)
# 2. Implement BMC (business logic)
# 3. Add routes (web-server)
# 4. Write tests
# 5. Add integration tests
```

### 2. Add Database Model
```bash
# 1. Create migration
# 2. Define struct with sqlx::FromRow
# 3. Implement BMC CRUD operations
# 4. Write tests
```

### 3. Deploy
```bash
# 1. Update Dockerfile
# 2. Push to GitHub
# 3. CI/CD runs tests
# 4. Build Docker image
# 5. Deploy to production
```

---

## 🔍 Troubleshooting

### SQLx Compile Errors
```bash
# Set DATABASE_URL
export DATABASE_URL=postgresql://user:pass@localhost/db

# Run migrations
sqlx migrate run

# Regenerate query metadata
cargo sqlx prepare
```

### Slow Compile Times
```bash
# Use cargo-chef in Dockerfile
# Enable incremental compilation
# Use mold/lld linker
```

### Type Errors
```bash
# Check trait bounds
# Ensure Send + Sync for async
# Use #[derive(Debug)] for better errors
```

---

## 📚 Learning Path

### Beginner (Week 1)
1. Read SKILL.md
2. Study architecture-overview.md
3. Copy complete-examples.md code
4. Modify examples to learn

### Intermediate (Week 2)
1. Read error-handling.md
2. Read async-patterns.md
3. Read database-sqlx.md
4. Build a simple API

### Advanced (Week 3)
1. Read testing-guide.md
2. Read security-patterns.md
3. Read performance-optimization.md
4. Optimize your API

### Production (Week 4)
1. Read deployment-guide.md
2. Set up CI/CD
3. Add monitoring
4. Deploy to production

---

## 🎓 Recommended Reading Order

**For New Rust Backend Developers:**
1. SKILL.md (10 core principles)
2. architecture-overview.md (understand the patterns)
3. complete-examples.md (see full code)
4. error-handling.md (master errors)
5. testing-guide.md (write tests)

**For Experienced Backend Developers New to Rust:**
1. SKILL.md (Rust-specific patterns)
2. async-patterns.md (Tokio vs Node.js)
3. error-handling.md (Result<T, E> vs try/catch)
4. database-sqlx.md (compile-time verification)
5. deployment-guide.md (Docker for Rust)

**For Rust Developers New to Web Development:**
1. architecture-overview.md (web architecture)
2. api-design.md (Axum framework)
3. database-sqlx.md (database operations)
4. security-patterns.md (authentication)
5. deployment-guide.md (production deployment)

---

## 🆘 Getting Help

### Check These Files First:
- **Error handling**: error-handling.md
- **Database issues**: database-sqlx.md
- **Async problems**: async-patterns.md
- **Test failures**: testing-guide.md
- **Performance**: performance-optimization.md

### Common Questions:

**Q: How do I add authentication?**
A: See security-patterns.md → Token-Based Authentication

**Q: How do I structure my project?**
A: See project-structure.md → Workspace Organization

**Q: How do I deploy?**
A: See deployment-guide.md → Docker Deployment

**Q: How do I test async code?**
A: See testing-guide.md → Async Testing

---

## ✨ Next Steps

1. ✅ Install the skill package
2. ✅ Read SKILL.md (5 minutes)
3. ✅ Try a complete example (15 minutes)
4. ✅ Build your first endpoint (30 minutes)
5. ✅ Add tests (15 minutes)
6. ✅ Deploy with Docker (30 minutes)

**Total time to first deployed endpoint**: ~2 hours

---

## 📞 Support

This skill package is based on:
- **Rust10x**: Jeremy Chone's production patterns
- **Axum**: Tokio's web framework
- **SQLx**: Compile-time SQL verification

For updates and contributions, refer to the main README.md.

---

**Version**: 1.0  
**Last Updated**: 2025-11-15  
**Status**: Production-Ready
