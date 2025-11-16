# Testing Guide - Rust Backend Testing Strategies

Complete guide to testing Rust backend services with unit tests, integration tests, mocking, and test-driven development.

## Table of Contents

- [Test Organization](#test-organization)
- [Unit Testing](#unit-testing)
- [Integration Testing](#integration-testing)
- [Test Fixtures and Helpers](#test-fixtures-and-helpers)
- [Mocking](#mocking)
- [Property-Based Testing](#property-based-testing)
- [Test Coverage](#test-coverage)

---

## Test Organization

### File Structure

```
crates/libs/lib-core/
├── src/
│   ├── model/
│   │   ├── user.rs          # Implementation
│   │   └── user/
│   │       └── tests.rs     # Tests in submodule
│   └── lib.rs
└── tests/                   # Integration tests
    ├── common/
    │   └── mod.rs           # Shared test utilities
    └── user_tests.rs        # Integration tests
```

### Test Modules

```rust
// src/model/user.rs

pub struct UserBmc;

impl UserBmc {
    pub async fn create(ctx: &Ctx, mm: &ModelManager, data: UserForCreate) -> Result<i64> {
        // Implementation
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_create_user() {
        // Unit test
    }
}
```

---

## Unit Testing

### Basic Unit Test

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_user_validation() {
        let user = User {
            id: 1,
            username: "test".to_string(),
            email: "test@example.com".to_string(),
        };

        assert_eq!(user.username, "test");
        assert!(user.email.contains("@"));
    }
}
```

### Async Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_create_user() {
        // Setup
        let mm = ModelManager::new().await.unwrap();
        let ctx = Ctx::root_ctx();

        let user_data = UserForCreate {
            username: "testuser".to_string(),
            email: "test@example.com".to_string(),
        };

        // Execute
        let user_id = UserBmc::create(&ctx, &mm, user_data).await.unwrap();

        // Verify
        assert!(user_id > 0);

        // Cleanup
        UserBmc::delete(&ctx, &mm, user_id).await.unwrap();
    }
}
```

### Testing Error Cases

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_create_user_duplicate_username() {
        let mm = ModelManager::new().await.unwrap();
        let ctx = Ctx::root_ctx();

        let user_data = UserForCreate {
            username: "duplicate".to_string(),
            email: "test1@example.com".to_string(),
        };

        // Create first user
        let user_id = UserBmc::create(&ctx, &mm, user_data.clone()).await.unwrap();

        // Try to create duplicate
        let result = UserBmc::create(&ctx, &mm, UserForCreate {
            username: "duplicate".to_string(),
            email: "test2@example.com".to_string(),
        }).await;

        // Verify error
        assert!(result.is_err());
        match result.unwrap_err() {
            Error::UsernameAlreadyExists { username } => {
                assert_eq!(username, "duplicate");
            }
            _ => panic!("Expected UsernameAlreadyExists error"),
        }

        // Cleanup
        UserBmc::delete(&ctx, &mm, user_id).await.unwrap();
    }
}
```

### Parameterized Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_username_validation() {
        let test_cases = vec![
            ("ab", false, "too short"),
            ("abc", true, "minimum length"),
            ("a".repeat(50), true, "maximum length"),
            ("a".repeat(51), false, "too long"),
            ("user@123", false, "invalid characters"),
            ("valid_user", true, "valid username"),
        ];

        for (username, should_succeed, description) in test_cases {
            let result = validate_username(username);

            if should_succeed {
                assert!(result.is_ok(), "Failed: {} - {}", description, username);
            } else {
                assert!(result.is_err(), "Should have failed: {} - {}", description, username);
            }
        }
    }
}
```

---

## Integration Testing

### Integration Test Structure

```rust
// tests/user_integration_tests.rs

use lib_core::ctx::Ctx;
use lib_core::model::{ModelManager, UserBmc, UserForCreate};

mod common; // Shared test utilities

#[tokio::test]
async fn test_user_lifecycle() {
    // Setup
    let mm = common::setup_test_db().await;
    let ctx = Ctx::root_ctx();

    // Create
    let user_id = UserBmc::create(&ctx, &mm, UserForCreate {
        username: "integration_test".to_string(),
        email: "integration@example.com".to_string(),
    })
    .await
    .expect("Failed to create user");

    // Read
    let user = UserBmc::get(&ctx, &mm, user_id)
        .await
        .expect("Failed to get user");

    assert_eq!(user.username, "integration_test");

    // Update
    UserBmc::update(&ctx, &mm, user_id, UserForUpdate {
        username: Some("updated_user".to_string()),
        ..Default::default()
    })
    .await
    .expect("Failed to update user");

    let updated_user = UserBmc::get(&ctx, &mm, user_id)
        .await
        .expect("Failed to get updated user");

    assert_eq!(updated_user.username, "updated_user");

    // Delete
    UserBmc::delete(&ctx, &mm, user_id)
        .await
        .expect("Failed to delete user");

    let result = UserBmc::get(&ctx, &mm, user_id).await;
    assert!(result.is_err());

    // Cleanup
    common::cleanup_test_db(&mm).await;
}
```

### HTTP Integration Tests

```rust
// tests/api_integration_tests.rs

use axum::{
    body::Body,
    http::{Request, StatusCode},
};
use tower::ServiceExt; // For oneshot

#[tokio::test]
async fn test_create_user_endpoint() {
    // Setup
    let app = create_test_app().await;

    let payload = serde_json::json!({
        "username": "apitest",
        "email": "apitest@example.com",
        "password": "SecurePass123!"
    });

    // Make request
    let response = app
        .oneshot(
            Request::builder()
                .method("POST")
                .uri("/api/v1/users")
                .header("content-type", "application/json")
                .body(Body::from(serde_json::to_string(&payload).unwrap()))
                .unwrap(),
        )
        .await
        .unwrap();

    // Verify
    assert_eq!(response.status(), StatusCode::CREATED);

    let body = hyper::body::to_bytes(response.into_body())
        .await
        .unwrap();

    let user: UserResponse = serde_json::from_slice(&body).unwrap();
    assert_eq!(user.username, "apitest");
}
```

---

## Test Fixtures and Helpers

### Common Test Utilities

```rust
// tests/common/mod.rs

use lib_core::model::ModelManager;
use sqlx::PgPool;
use std::sync::Arc;

pub async fn setup_test_db() -> ModelManager {
    // Create test database connection
    let database_url = std::env::var("TEST_DATABASE_URL")
        .expect("TEST_DATABASE_URL must be set");

    let pool = PgPool::connect(&database_url)
        .await
        .expect("Failed to connect to test database");

    // Run migrations
    sqlx::migrate!("./migrations")
        .run(&pool)
        .await
        .expect("Failed to run migrations");

    ModelManager::new_from_pool(Arc::new(pool))
}

pub async fn cleanup_test_db(mm: &ModelManager) {
    // Clean up test data
    let pool = mm.db().pool();

    sqlx::query("TRUNCATE TABLE users, posts, subscriptions CASCADE")
        .execute(pool)
        .await
        .expect("Failed to clean up test database");
}

pub fn create_test_user_data() -> UserForCreate {
    UserForCreate {
        username: format!("test_{}", uuid::Uuid::new_v4()),
        email: format!("test_{}@example.com", uuid::Uuid::new_v4()),
    }
}
```

### Test Fixtures with Builder Pattern

```rust
// tests/common/fixtures.rs

pub struct UserFixture {
    username: String,
    email: String,
    is_admin: bool,
}

impl UserFixture {
    pub fn new() -> Self {
        Self {
            username: format!("user_{}", uuid::Uuid::new_v4()),
            email: format!("user_{}@example.com", uuid::Uuid::new_v4()),
            is_admin: false,
        }
    }

    pub fn username(mut self, username: &str) -> Self {
        self.username = username.to_string();
        self
    }

    pub fn email(mut self, email: &str) -> Self {
        self.email = email.to_string();
        self
    }

    pub fn admin(mut self) -> Self {
        self.is_admin = true;
        self
    }

    pub async fn create(self, ctx: &Ctx, mm: &ModelManager) -> i64 {
        UserBmc::create(ctx, mm, UserForCreate {
            username: self.username,
            email: self.email,
        })
        .await
        .expect("Failed to create test user")
    }
}

// Usage
#[tokio::test]
async fn test_with_fixture() {
    let mm = setup_test_db().await;
    let ctx = Ctx::root_ctx();

    let user_id = UserFixture::new()
        .username("admin")
        .admin()
        .create(&ctx, &mm)
        .await;

    // Test with user...
}
```

### Serial Test Execution

```rust
use serial_test::serial;

#[tokio::test]
#[serial] // Prevents parallel execution
async fn test_requires_exclusive_db_access() {
    let mm = setup_test_db().await;
    // Test that modifies global state
}

#[tokio::test]
#[serial] // Runs after previous test
async fn test_also_requires_exclusive_access() {
    let mm = setup_test_db().await;
    // Another test that needs isolation
}
```

---

## Mocking

### Manual Mocking with Traits

```rust
use async_trait::async_trait;

#[async_trait]
pub trait UserRepository: Send + Sync {
    async fn find_by_id(&self, id: i64) -> Result<Option<User>>;
    async fn save(&self, user: &User) -> Result<()>;
}

// Mock implementation
pub struct MockUserRepository {
    users: Arc<Mutex<HashMap<i64, User>>>,
}

#[async_trait]
impl UserRepository for MockUserRepository {
    async fn find_by_id(&self, id: i64) -> Result<Option<User>> {
        let users = self.users.lock().await;
        Ok(users.get(&id).cloned())
    }

    async fn save(&self, user: &User) -> Result<()> {
        let mut users = self.users.lock().await;
        users.insert(user.id, user.clone());
        Ok(())
    }
}

// Test usage
#[tokio::test]
async fn test_with_mock_repository() {
    let mock_repo = MockUserRepository {
        users: Arc::new(Mutex::new(HashMap::new())),
    };

    let service = UserService::new(Arc::new(mock_repo));

    // Test service with mock
}
```

### Using mockall Crate

```rust
use mockall::{automock, predicate::*};

#[automock]
#[async_trait]
pub trait EmailService: Send + Sync {
    async fn send_welcome_email(&self, user: &User) -> Result<()>;
    async fn send_password_reset(&self, user: &User, token: &str) -> Result<()>;
}

#[tokio::test]
async fn test_user_registration_sends_email() {
    // Create mock
    let mut mock_email = MockEmailService::new();

    // Set expectations
    mock_email
        .expect_send_welcome_email()
        .times(1)
        .returning(|_| Ok(()));

    // Use mock in service
    let service = UserService::new(Arc::new(mock_email));

    let user_id = service.register_user(UserForCreate {
        username: "test".to_string(),
        email: "test@example.com".to_string(),
    })
    .await
    .unwrap();

    // Mock verifies send_welcome_email was called
}
```

---

## Property-Based Testing

### Using proptest

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_username_validation_properties(
        username in "[a-zA-Z0-9_]{3,50}"
    ) {
        // Property: All usernames matching pattern should be valid
        let result = validate_username(&username);
        prop_assert!(result.is_ok());
    }

    #[test]
    fn test_username_too_short(
        username in "[a-zA-Z]{0,2}"
    ) {
        // Property: All usernames < 3 chars should be invalid
        let result = validate_username(&username);
        prop_assert!(result.is_err());
    }
}

// Custom strategies
fn arb_user() -> impl Strategy<Value = UserForCreate> {
    ("[a-zA-Z0-9_]{3,20}", "[a-z]+@[a-z]+\\.[a-z]+")
        .prop_map(|(username, email)| UserForCreate {
            username,
            email,
        })
}

proptest! {
    #[test]
    fn test_user_creation_roundtrip(
        user_data in arb_user()
    ) {
        // Property: Created users should be retrievable
        tokio_test::block_on(async {
            let mm = setup_test_db().await;
            let ctx = Ctx::root_ctx();

            let id = UserBmc::create(&ctx, &mm, user_data.clone())
                .await
                .unwrap();

            let retrieved = UserBmc::get(&ctx, &mm, id)
                .await
                .unwrap();

            prop_assert_eq!(retrieved.username, user_data.username);
        });
    }
}
```

---

## Test Coverage

### Running Tests with Coverage

```bash
# Install tarpaulin
cargo install cargo-tarpaulin

# Run tests with coverage
cargo tarpaulin --out Html --output-dir coverage

# View coverage report
open coverage/index.html
```

### Coverage Configuration

```toml
# .cargo/config.toml

[build]
rustflags = ["-C", "instrument-coverage"]

[test]
# Additional test configuration
```

### CI Coverage Integration

```yaml
# .github/workflows/ci.yml

- name: Run tests with coverage
  run: |
    cargo tarpaulin --out Xml --output-dir coverage

- name: Upload coverage to Codecov
  uses: codecov/codecov-action@v3
  with:
    files: ./coverage/cobertura.xml
```

### Coverage Targets

**Recommended Coverage:**
- **Unit Tests**: 80%+ for business logic
- **Integration Tests**: Critical paths covered
- **E2E Tests**: Happy paths and error cases

**Focus Areas:**
- ✅ Business logic (BMC methods)
- ✅ Error handling paths
- ✅ Input validation
- ✅ Authentication/authorization

**Less Critical:**
- ⚠️ Simple getters/setters
- ⚠️ Database boilerplate
- ⚠️ Generated code

---

## Testing Best Practices

### 1. Use Descriptive Test Names

```rust
#[tokio::test]
async fn test_create_user_with_duplicate_username_returns_conflict_error() {
    // Clear what is being tested
}
```

### 2. Follow AAA Pattern

```rust
#[tokio::test]
async fn test_update_user() {
    // Arrange
    let mm = setup_test_db().await;
    let ctx = Ctx::root_ctx();
    let user_id = create_test_user(&ctx, &mm).await;

    // Act
    let result = UserBmc::update(&ctx, &mm, user_id, UserForUpdate {
        username: Some("updated".to_string()),
        ..Default::default()
    }).await;

    // Assert
    assert!(result.is_ok());
    let user = UserBmc::get(&ctx, &mm, user_id).await.unwrap();
    assert_eq!(user.username, "updated");
}
```

### 3. Test One Thing Per Test

```rust
// ❌ BAD: Testing multiple things
#[tokio::test]
async fn test_user_operations() {
    let id = create_user().await;
    let user = get_user(id).await;
    update_user(id).await;
    delete_user(id).await;
}

// ✅ GOOD: Separate tests
#[tokio::test]
async fn test_create_user() { /* ... */ }

#[tokio::test]
async fn test_get_user() { /* ... */ }

#[tokio::test]
async fn test_update_user() { /* ... */ }

#[tokio::test]
async fn test_delete_user() { /* ... */ }
```

### 4. Clean Up Test Data

```rust
#[tokio::test]
async fn test_with_cleanup() {
    let mm = setup_test_db().await;
    let ctx = Ctx::root_ctx();

    let user_id = UserBmc::create(&ctx, &mm, test_data()).await.unwrap();

    // Test operations...

    // Always clean up
    UserBmc::delete(&ctx, &mm, user_id).await.unwrap();
}
```

---

**Related Files:**
- [SKILL.md](../SKILL.md)
- [architecture-overview.md](architecture-overview.md)
- [error-handling.md](error-handling.md)
- [api-design.md](api-design.md)
