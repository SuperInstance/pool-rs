# Pool-RS Developer Guide

## Contributing to Pool-RS

This guide is for developers who want to contribute to pool-rs.

---

## Architecture Overview

### Crate Structure

```
pool-rs/
├── Cargo.toml
├── src/
│   ├── lib.rs              # Public API
│   ├── pool.rs             # Pool<T> implementation
│   ├── builder.rs          # PoolBuilder<T> implementation
│   ├── connection.rs       # PooledConnection<T> implementation
│   ├── manager.rs          # PoolManager (internal coordination)
│   ├── store.rs            # ConnectionStore (connection storage)
│   ├── health.rs           # HealthChecker
│   ├── adaptive.rs         # AdaptiveSizer
│   ├── metrics.rs          # MetricsCollector
│   ├── shutdown.rs         # ShutdownCoordinator
│   ├── config.rs           # PoolConfig
│   └── error.rs            # Error types
├── benches/
│   └── acquire.rs          # Benchmarks
├── tests/
│   ├── integration.rs      # Integration tests
│   └── health_check.rs     # Health check tests
└── examples/
    ├── hello_pool.rs       # Simple example
    ├── database_pool.rs    # Database example
    └── adaptive_sizing.rs  # Adaptive sizing example
```

---

## Development Setup

### Prerequisites

- Rust 1.75 or later
- tokio 1.x
- Clippy
- cargo-hack (for testing)

### Clone and Build

```bash
git clone https://github.com/SuperInstance/pool-rs.git
cd pool-rs
cargo build
cargo test
cargo clippy -- -D warnings
cargo fmt
```

### Run Examples

```bash
cargo run --example hello_pool
cargo run --example database_pool
cargo run --example adaptive_sizing
```

---

## Implementation Guide

### Custom Health Checks

Implement custom health checks for your connection type:

```rust
use pool::{Pool, Error};

let pool = Pool::builder()
    .health_check(|conn| async move {
        // Your health check logic here
        if conn.is_broken() {
            return Err(Error::HealthCheckFailed("Connection broken".to_string()));
        }

        // Test connection
        conn.ping().await
            .map_err(|e| Error::HealthCheckFailed(e.to_string()))?;

        Ok(())
    })
    .build(|| async {
        MyConnection::connect("localhost").await
    }).await?;
```

**Best practices**:
- Keep health checks fast (< 1 second)
- Use a simple query (SELECT 1, PING)
- Set health_check_interval appropriately
- Return specific error messages

---

### Custom Pool Strategies

#### Strategy 1: Conservative Pool

```rust
let pool = Pool::builder()
    .min_size(0)                      // Don't pre-create
    .max_size(5)                      // Small pool
    .idle_timeout(Duration::from_secs(60))   // Close quickly
    .test_on_acquire(true)            // Always test
    .build(factory).await?;
```

**Use case**: Low-traffic applications, database connections are expensive

---

#### Strategy 2: Aggressive Pool

```rust
let pool = Pool::builder()
    .min_size(10)                     // Pre-create many
    .max_size(100)                    // Large pool
    .idle_timeout(Duration::from_secs(3600))  // Keep connections
    .test_on_acquire(false)           // Skip test for speed
    .build(factory).await?;
```

**Use case**: High-traffic applications, need fast responses

---

#### Strategy 3: Adaptive Pool

```rust
let pool = Pool::builder()
    .min_size(5)
    .max_size(100)
    .adaptive_sizing(true)
    .adaptive_grow_threshold(0.8)
    .adaptive_shrink_idle_secs(300)
    .adaptive_predictive(true)
    .build(factory).await?;
```

**Use case**: Variable load, want automatic scaling

---

### Metrics Implementation

Pool-rs uses lock-free atomic operations for metrics:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

pub struct Metrics {
    // Counters (cumulative)
    connections_created: AtomicU64,
    connections_destroyed: AtomicU64,
    acquires_total: AtomicU64,
    releases_total: AtomicU64,
    health_check_failures: AtomicU64,

    // Gauges (current values)
    active_connections: AtomicU64,
    idle_connections: AtomicU64,
    wait_queue_length: AtomicU64,
}

impl Metrics {
    pub fn record_acquire(&self) {
        self.acquires_total.fetch_add(1, Ordering::Relaxed);
    }

    pub fn record_release(&self) {
        self.releases_total.fetch_add(1, Ordering::Relaxed);
    }

    pub fn connections_created(&self) -> u64 {
        self.connections_created.load(Ordering::Relaxed)
    }
}
```

**Why atomic operations**:
- Lock-free reads (fast)
- No contention on metrics collection
- Minimal overhead

---

### Testing Pools

#### Unit Tests

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::time::Duration;

    #[tokio::test]
    async fn test_pool_creation() {
        let pool = Pool::builder()
            .max_size(5)
            .build(|| async { Ok(TestConnection) })
            .await
            .unwrap();

        assert_eq!(pool.status().max_size, 5);
    }

    #[tokio::test]
    async fn test_connection_acquire() {
        let pool = Pool::new(|| async { Ok(TestConnection) }).await.unwrap();

        let conn = pool.get().await.unwrap();
        assert_eq!(pool.status().active_connections, 1);

        drop(conn);
        assert_eq!(pool.status().active_connections, 0);
    }

    #[tokio::test]
    async fn test_pool_exhaustion() {
        let pool = Pool::builder()
            .max_size(2)
            .acquire_timeout(Duration::from_millis(100))
            .build(|| async { Ok(TestConnection) })
            .await
            .unwrap();

        let conn1 = pool.get().await.unwrap();
        let conn2 = pool.get().await.unwrap();

        // Third connection should timeout
        let result = pool.get().await;
        assert!(matches!(result, Err(Error::AcquireTimeout)));
    }
}

struct TestConnection;
```

---

#### Integration Tests

```rust
// tests/integration.rs
use pool::Pool;
use std::time::Duration;

#[tokio::test]
async fn test_concurrent_acquires() {
    let pool = Pool::builder()
        .max_size(10)
        .build(|| async { Ok(TestConnection) })
        .await
        .unwrap();

    let handles: Vec<_> = (0..100)
        .map(|_| {
            let pool = pool.clone();
            tokio::spawn(async move {
                let conn = pool.get().await.unwrap();
                tokio::time::sleep(Duration::from_millis(10)).await;
                drop(conn);
            })
        })
        .collect();

    for handle in handles {
        handle.await.unwrap();
    }

    assert_eq!(pool.status().active_connections, 0);
}
```

---

#### Property-Based Tests

```rust
#[cfg(test)]
mod proptests {
    use proptest::prelude::*;

    proptest! {
        #[test]
        fn test_pool_invariants(max_size in 1usize..100) {
            let pool = Pool::builder()
                .max_size(max_size)
                .build(|| async { Ok(TestConnection) });

            // Invariant: total connections never exceed max_size
            tokio::runtime::Runtime::new().unwrap().block_on(async {
                let pool = pool.await.unwrap();
                for _ in 0..max_size * 2 {
                    let conn = pool.get().await.unwrap();
                    drop(conn);
                }

                let status = pool.status();
                assert!(status.total_connections <= max_size);
            });
        }
    }
}
```

---

### Error Handling

Define clear error types:

```rust
#[derive(Debug, thiserror::Error)]
pub enum Error {
    #[error("Connection failed: {0}")]
    ConnectionFailed(String),

    #[error("Health check failed: {0}")]
    HealthCheckFailed(String),

    #[error("Pool exhausted")]
    PoolExhausted,

    #[error("Acquire timeout")]
    AcquireTimeout,

    #[error("Pool shutting down")]
    PoolShuttingDown,

    #[error("Invalid config: {0}")]
    InvalidConfig(String),
}
```

**Error handling best practices**:
- Use `thiserror` for error definitions
- Provide context in error messages
- Make errors `Send + Sync`
- Use `Result<T, Error>` consistently

---

## Performance Optimization

### 1. Lock-Free Metrics

Use atomic operations for metrics:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

struct Metrics {
    acquires_total: AtomicU64,
}

// Fast (lock-free)
self.acquires_total.fetch_add(1, Ordering::Relaxed);

// Slow (mutex)
self.acquires_total.lock().push(1);
```

---

### 2. Lazy Health Checks

Skip health checks if not due:

```rust
if last_health_check.map_or(true, |t| t.elapsed() > interval) {
    // Run health check
} else {
    // Skip (fast path)
}
```

---

### 3. Connection Reuse

Reuse connections instead of closing:

```rust
// Bad: Close connection after use
drop(connection);

// Good: Return to pool
pool.release(connection);
```

---

### 4. Pre-Warm Pool

Create connections at startup:

```rust
Pool::builder()
    .initial_size(10)  // Pre-create 10 connections
    .build(factory).await?
```

---

### 5. Reduce Allocations

Pre-allocate with capacity:

```rust
let mut idle = VecDeque::with_capacity(max_size);
```

---

## Integration with Tripartite-RS

For distributed pool state coordination:

```rust
use tripartite::Council;
use pool::Pool;

let council = Council::new(/*...*/).await?;

// Use consensus for pool size decisions
let pool_size = council
    .decide("What should the pool size be?", vec![])
    .await?;

let pool = Pool::builder()
    .max_size(pool_size)
    .build(factory).await?;
```

---

## Contributing Guidelines

### Code Style

- Use `cargo fmt` for formatting
- Run `cargo clippy -- -D warnings` before committing
- Add tests for new features
- Document public APIs with doc comments

### Testing

- Add unit tests for new functionality
- Add integration tests for complex scenarios
- Ensure all tests pass: `cargo test`
- Run clippy: `cargo clippy -- -D warnings`

### Documentation

- Add doc comments to public APIs
- Update user guide for new features
- Add examples for new functionality
- Keep README up-to-date

### Pull Request Process

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests and documentation
5. Run `cargo test` and `cargo clippy`
6. Submit a pull request

### Code Review Checklist

- [ ] Tests pass
- [ ] No clippy warnings
- [ ] Code is formatted
- [ ] Documentation updated
- [ ] Examples added
- [ ] Performance considered

---

## Summary

You've learned:
- ✅ Pool-rs architecture
- ✅ Development setup
- ✅ Custom health checks
- ✅ Pool strategies
- ✅ Testing pools
- ✅ Performance optimization
- ✅ Contributing guidelines

**Next**: [CLI Design](06-cli.md) - Command-line tool for pool-rs

---

**Document Status**: ✅ Complete
**Word Count**: ~2,500 words
