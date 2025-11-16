# Fault Tolerance and Resilience - Rust Backend Production

**Building resilient Rust backend services that gracefully handle failures**

This resource covers fault tolerance patterns, circuit breakers, retry strategies, and graceful degradation for production-ready Rust services.

---

## Table of Contents

- [Circuit Breakers](#circuit-breakers)
- [Retry Strategies](#retry-strategies)
- [Bulkhead Pattern](#bulkhead-pattern)
- [Timeout Configuration](#timeout-configuration)
- [Graceful Degradation](#graceful-degradation)
- [Panic Recovery](#panic-recovery)

---

## Circuit Breakers

### Principle

Circuit breakers prevent cascading failures by detecting failing services and temporarily blocking requests to allow recovery.

### Implementation with failsafe

```toml
#Cargo.toml
[dependencies]
failsafe = "1.3"
tokio = { version = "1", features = ["full"] }
```

```rust
use failsafe::{CircuitBreaker, Config, Error as FailsafeError};
use std::time::Duration;

// Configure circuit breaker
pub fn create_circuit_breaker() -> CircuitBreaker {
    Config::new()
        .failure_rate_threshold(0.5)  // Open at 50% failure rate
        .wait_duration_in_open_state(Duration::from_secs(30))  // Wait 30s before half-open
        .permitted_number_of_calls_in_half_open_state(3)  // Test with 3 calls
        .minimum_number_of_calls(10)  // Need 10 calls before calculating failure rate
        .build()
}

// Use circuit breaker with async function
pub async fn call_external_service_with_cb(
    cb: &CircuitBreaker,
    url: &str,
) -> Result<String, Box<dyn std::error::Error>> {
    let future = async {
        let response = reqwest::get(url).await?;
        response.text().await
    };

    match cb.call(future).await {
        Ok(data) => Ok(data),
        Err(FailsafeError::Rejected) => {
            // Circuit is open
            Err("Service temporarily unavailable (circuit open)".into())
        }
        Err(FailsafeError::Inner(e)) => Err(e),
    }
}
```

### Tower Circuit Breaker

```toml
[dependencies]
tower = { version = "0.4", features = ["limit", "timeout", "util"] }
```

```rust
use tower::{Service, ServiceBuilder, ServiceExt};
use tower::limit::ConcurrencyLimit;
use std::time::Duration;

// Build service with circuit breaker and limits
let service = ServiceBuilder::new()
    .concurrency_limit(100)  // Max 100 concurrent requests
    .timeout(Duration::from_secs(10))  // 10s timeout
    .service(my_service);
```

---

## Retry Strategies

### Exponential Backoff

```toml
[dependencies]
backoff = { version = "0.4", features = ["tokio"] }
```

```rust
use backoff::{ExponentialBackoff, future::retry};
use std::time::Duration;

pub async fn call_with_retry<F, Fut, T, E>(operation: F) -> Result<T, E>
where
    F: FnMut() -> Fut,
    Fut: Future<Output = Result<T, E>>,
    E: std::error::Error,
{
    let backoff = ExponentialBackoff {
        initial_interval: Duration::from_millis(100),
        max_interval: Duration::from_secs(10),
        max_elapsed_time: Some(Duration::from_secs(60)),
        multiplier: 2.0,
        ..Default::default()
    };

    retry(backoff, operation).await
}
```

### Conditional Retries

```rust
use backoff::Error as BackoffError;

pub async fn fetch_with_smart_retry(url: &str) -> Result<String, reqwest::Error> {
    let operation = || async {
        let response = reqwest::get(url).await?;

        match response.status() {
            status if status.is_success() => {
                let text = response.text().await?;
                Ok(text)
            }
            status if status.is_server_error() => {
                // Retry on 5xx errors
                Err(BackoffError::transient(response.error_for_status().unwrap_err()))
            }
            _ => {
                // Don't retry on 4xx errors
                Err(BackoffError::permanent(response.error_for_status().unwrap_err()))
            }
        }
    };

    backoff::future::retry(ExponentialBackoff::default(), operation).await
}
```

---

## Bulkhead Pattern

### Isolate Resource Pools

```rust
use tokio::sync::Semaphore;
use std::sync::Arc;

pub struct BulkheadService {
    // Separate semaphores for different resources
    database_permits: Arc<Semaphore>,
    api_permits: Arc<Semaphore>,
}

impl BulkheadService {
    pub fn new() -> Self {
        Self {
            database_permits: Arc::new(Semaphore::new(20)),  // Max 20 DB connections
            api_permits: Arc::new(Semaphore::new(50)),       // Max 50 API calls
        }
    }

    pub async fn execute_db_query<F, T>(&self, query: F) -> Result<T, Box<dyn std::error::Error>>
    where
        F: FnOnce() -> T,
    {
        let _permit = self.database_permits.acquire().await?;
        Ok(query())
    }

    pub async fn call_external_api<F, Fut, T>(&self, call: F) -> Result<T, Box<dyn std::error::Error>>
    where
        F: FnOnce() -> Fut,
        Fut: Future<Output = Result<T, Box<dyn std::error::Error>>>,
    {
        let _permit = self.api_permits.acquire().await?;
        call().await
    }
}
```

---

## Timeout Configuration

### Tokio Timeout

```rust
use tokio::time::{timeout, Duration};

pub async fn fetch_with_timeout(url: &str) -> Result<String, Box<dyn std::error::Error>> {
    let fetch_future = async {
        let response = reqwest::get(url).await?;
        response.text().await
    };

    match timeout(Duration::from_secs(5), fetch_future).await {
        Ok(Ok(data)) => Ok(data),
        Ok(Err(e)) => Err(e.into()),
        Err(_) => Err("Request timed out after 5 seconds".into()),
    }
}
```

### Tower Timeout Middleware

```rust
use tower::{ServiceBuilder, timeout::Timeout};
use std::time::Duration;

let service = ServiceBuilder::new()
    .timeout(Duration::from_secs(10))
    .service(my_service);
```

---

## Graceful Degradation

### Fallback Responses

```rust
pub async fn get_user_recommendations(user_id: &str) -> Result<Vec<String>, Box<dyn std::error::Error>> {
    // Try ML recommendation service
    match fetch_ml_recommendations(user_id).await {
        Ok(recommendations) => Ok(recommendations),
        Err(e) => {
            tracing::warn!("ML service failed: {}, using fallback", e);
            // Fallback to simple rule-based recommendations
            Ok(get_simple_recommendations(user_id))
        }
    }
}

fn get_simple_recommendations(user_id: &str) -> Vec<String> {
    // Simple fallback logic
    vec!["Popular Item 1".to_string(), "Popular Item 2".to_string()]
}
```

### Feature Toggles

```rust
use std::sync::Arc;
use std::collections::HashMap;

pub struct FeatureFlags {
    flags: Arc<HashMap<String, bool>>,
}

impl FeatureFlags {
    pub fn is_enabled(&self, feature: &str) -> bool {
        self.flags.get(feature).copied().unwrap_or(false)
    }
}

pub async fn handle_request(feature_flags: &FeatureFlags) -> Result<Response, Error> {
    if feature_flags.is_enabled("new_algorithm") {
        // Use new algorithm
        new_algorithm().await
    } else {
        // Fallback to stable algorithm
        stable_algorithm().await
    }
}
```

---

## Panic Recovery

### Global Panic Handler

```rust
use std::panic;

pub fn setup_panic_handler() {
    panic::set_hook(Box::new(|panic_info| {
        let payload = panic_info.payload();
        let message = if let Some(s) = payload.downcast_ref::<&str>() {
            s.to_string()
        } else if let Some(s) = payload.downcast_ref::<String>() {
            s.clone()
        } else {
            "Unknown panic payload".to_string()
        };

        let location = panic_info
            .location()
            .map(|l| format!("{}:{}:{}", l.file(), l.line(), l.column()))
            .unwrap_or_else(|| "unknown location".to_string());

        tracing::error!(
            panic_message = %message,
            panic_location = %location,
            "Panic occurred"
        );

        // Send alert to monitoring system
        // alert_system::send_panic_alert(&message, &location);
    }));
}
```

### Catch Unwind in Async

```rust
use std::panic::AssertUnwindSafe;
use futures::FutureExt;

pub async fn safe_execute<F, Fut, T>(operation: F) -> Result<T, Box<dyn std::error::Error>>
where
    F: FnOnce() -> Fut,
    Fut: Future<Output = T>,
{
    match AssertUnwindSafe(operation()).catch_unwind().await {
        Ok(result) => Ok(result),
        Err(panic) => {
            tracing::error!("Caught panic in async operation: {:?}", panic);
            Err("Operation panicked".into())
        }
    }
}
```

---

## Deadlock Prevention

### Consistent Lock Ordering

```rust
use std::sync::Mutex;

pub struct BankAccount {
    id: u64,
    balance: Mutex<u64>,
}

// ❌ BAD: Can cause deadlock
pub fn transfer_bad(from: &BankAccount, to: &BankAccount, amount: u64) {
    let mut from_balance = from.balance.lock().unwrap();
    let mut to_balance = to.balance.lock().unwrap();  // Deadlock risk!

    *from_balance -= amount;
    *to_balance += amount;
}

// ✅ GOOD: Consistent ordering prevents deadlock
pub fn transfer_good(from: &BankAccount, to: &BankAccount, amount: u64) {
    let (first, second) = if from.id < to.id {
        (from, to)
    } else {
        (to, from)
    };

    let mut first_balance = first.balance.lock().unwrap();
    let mut second_balance = second.balance.lock().unwrap();

    if from.id < to.id {
        *first_balance -= amount;
        *second_balance += amount;
    } else {
        *second_balance += amount;
        *first_balance -= amount;
    }
}
```

### Timeout on Lock Acquisition

```rust
use parking_lot::Mutex;
use std::time::Duration;

pub fn try_acquire_with_timeout<T>(
    mutex: &Mutex<T>,
    timeout: Duration,
) -> Option<parking_lot::MutexGuard<T>> {
    mutex.try_lock_for(timeout)
}
```

---

## Production Resilience Checklist

### Pre-Deployment
- [ ] Circuit breakers on all external dependencies
- [ ] Retry logic with exponential backoff
- [ ] Timeouts configured for all I/O operations
- [ ] Bulkheads isolate critical resource pools
- [ ] Panic handler logs and alerts configured
- [ ] Deadlock prevention verified (lock ordering)

### Runtime Monitoring
- [ ] Circuit breaker state metrics exposed
- [ ] Retry attempt counts tracked
- [ ] Timeout frequency monitored
- [ ] Panic events trigger alerts
- [ ] Resource pool saturation monitored

---

**Related Resources:**
- [Core Hardening Principles](./core-principles.md)
- [Security Hardening](./security-hardening.md)
- [Monitoring and Observability](./monitoring.md)
- [Performance and Scalability](./performance.md)

**Version**: 1.0
**Last Updated**: 2025-11-15
**Compliance**: NIST SP 800-53 SI-13 (Predictable Failure Prevention)
