# Pool-RS API Design

## Overview

This document defines the complete public API for pool-rs. The API is designed to be:

- **Simple**: 3-line API for basic use cases
- **Powerful**: Builder pattern for advanced configuration
- **Type-Safe**: Generic pool with compile-time type checking
- **Async-First**: Built on tokio from the ground up
- **Observable**: Metrics and tracing built-in

---

## Core Types

### Pool<T>

The main connection pool type.

```rust
/// Generic connection pool for type `T`
///
/// # Example
///
/// ```rust
/// use pool::Pool;
/// use tokio::time::{timeout, Duration};
///
/// # #[tokio::main]
/// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
/// let pool = Pool::builder()
///     .min_size(5)
///     .max_size(20)
///     .build(|| async {
///         // Create your connection here
///         MyConnection::connect("localhost:5432").await
///     }).await?;
///
/// // Acquire connection
/// let conn = pool.get().await?;
/// // Use connection
/// conn.query("SELECT * FROM users").await?;
/// // Connection automatically returned to pool when dropped
/// # Ok(())
/// # }
///
/// struct MyConnection;
///
/// impl MyConnection {
///     async fn connect(_addr: &str) -> Result<Self, Box<dyn std::error::Error>> {
///         Ok(MyConnection)
///     }
///     async fn query(&self, _sql: &str) -> Result<(), Box<dyn std::error::Error>> {
///         Ok(())
///     }
/// }
/// ```
pub struct Pool<T> {
    inner: Arc<PoolInner<T>>,
}
```

---

### PoolBuilder<T>

Builder for constructing pools with custom configuration.

```rust
/// Builder for creating a [`Pool`] with custom configuration.
///
/// # Example
///
/// ```rust
/// use pool::Pool;
/// use std::time::Duration;
///
/// # #[tokio::main]
/// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
/// let pool = Pool::builder()
///     .min_size(5)
///     .max_size(20)
///     .initial_size(10)
///     .acquire_timeout(Duration::from_secs(10))
///     .idle_timeout(Duration::from_secs(300))
///     .max_lifetime(Duration::from_secs(1800))
///     .health_check_interval(Duration::from_secs(60))
///     .health_check(|conn| async move {
///         conn.ping().await.map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
///     })
///     .build(|| async {
///         MyConnection::connect("localhost:5432").await
///     }).await?;
/// # Ok(())
/// # }
///
/// struct MyConnection;
/// impl MyConnection {
///     async fn connect(_addr: &str) -> Result<Self, Box<dyn std::error::Error>> {
///         Ok(MyConnection)
///     }
///     async fn ping(&self) -> Result<(), Box<dyn std::error::Error>> {
///         Ok(())
///     }
/// }
/// ```
pub struct PoolBuilder<T> {
    config: PoolConfig,
    health_check_fn: Option<HealthCheckFn<T>>,
    metrics: Option<Metrics>,
}
```

---

### PooledConnection<T>

A wrapper around a connection that automatically returns it to the pool when dropped.

```rust
/// A connection managed by the pool.
///
/// When this wrapper is dropped, the connection is returned to the pool.
/// If the pool is shutting down, the connection is closed instead.
///
/// # Example
///
/// ```rust
/// # use pool::{Pool, PooledConnection};
/// # #[tokio::main]
/// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
/// # let pool: Pool<MyConnection> = unimplemented!();
/// // Acquire connection
/// let mut conn = pool.get().await?;
///
/// // Use connection
/// conn.execute("SELECT * FROM users").await?;
///
/// // Connection automatically returned to pool when `conn` is dropped
/// # Ok(())
/// # }
/// # struct MyConnection;
/// # impl MyConnection {
/// #     async fn execute(&self, _sql: &str) -> Result<(), Box<dyn std::error::Error>> {
/// #         Ok(())
/// #     }
/// # }
/// ```
pub struct PooledConnection<T> {
    pool: Arc<PoolInner<T>>,
    conn: Option<T>,
    created_at: Instant,
}
```

---

## Configuration API

### Pool::new() - Quick Start

Create a pool with default configuration.

```rust
impl<T> Pool<T>
where
    T: Send + 'static,
{
    /// Create a new pool with default configuration.
    ///
    /// # Default Configuration
    ///
    /// - `min_size`: 0
    /// - `max_size`: 10
    /// - `initial_size`: 0
    /// - `acquire_timeout`: 30s
    /// - `idle_timeout`: 600s (10 minutes)
    /// - `max_lifetime`: 1800s (30 minutes)
    /// - `health_check_enabled`: true
    /// - `health_check_interval`: 30s
    /// - `test_on_acquire`: true
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::new(|| async {
    ///     MyConnection::connect("localhost:5432").await
    /// }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// impl MyConnection {
    ///     async fn connect(_addr: &str) -> Result<Self, Box<dyn std::error::Error>> {
    ///         Ok(MyConnection)
    ///     }
    /// }
    /// ```
    pub async fn new<F, Fut>(factory: F) -> Result<Self, Error>
    where
        F: Fn() -> Fut + Send + Sync + 'static,
        Fut: Future<Output = Result<T, Error>> + Send + 'static,
    {
        Self::builder().build(factory).await
    }
}
```

---

### Pool::builder() - Custom Configuration

Create a builder for custom configuration.

```rust
impl<T> Pool<T>
where
    T: Send + 'static,
{
    /// Create a [`PoolBuilder`] to configure the pool.
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    /// use std::time::Duration;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .min_size(5)
    ///     .max_size(20)
    ///     .build(|| async {
    ///         MyConnection::connect("localhost:5432").await
    ///     }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// impl MyConnection {
    ///     async fn connect(_addr: &str) -> Result<Self, Box<dyn std::error::Error>> {
    ///         Ok(MyConnection)
    ///     }
    /// }
    /// ```
    pub fn builder() -> PoolBuilder<T> {
        PoolBuilder::new()
    }
}
```

---

## Builder API

### PoolBuilder Methods

#### min_size()

```rust
impl<T> PoolBuilder<T> {
    /// Set the minimum number of idle connections to maintain.
    ///
    /// # Default
    ///
    /// `0`
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .min_size(5)  // Maintain at least 5 idle connections
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn min_size(mut self, min: usize) -> Self {
        self.config.min_size = min;
        self
    }
}
```

---

#### max_size()

```rust
impl<T> PoolBuilder<T> {
    /// Set the maximum number of connections to create.
    ///
    /// # Default
    ///
    /// `10`
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .max_size(100)  // Maximum 100 connections
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn max_size(mut self, max: usize) -> Self {
        self.config.max_size = max;
        self
    }
}
```

---

#### initial_size()

```rust
impl<T> PoolBuilder<T> {
    /// Set the number of connections to create at startup.
    ///
    /// # Default
    ///
    /// `min_size`
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .initial_size(10)  // Pre-create 10 connections
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn initial_size(mut self, size: usize) -> Self {
        self.config.initial_size = size;
        self
    }
}
```

---

#### acquire_timeout()

```rust
impl<T> PoolBuilder<T> {
    /// Set the maximum time to wait for a connection.
    ///
    /// # Default
    ///
    /// `30 seconds`
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    /// use std::time::Duration;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .acquire_timeout(Duration::from_secs(10))  // 10 second timeout
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn acquire_timeout(mut self, timeout: Duration) -> Self {
        self.config.acquire_timeout = timeout;
        self
    }
}
```

---

#### idle_timeout()

```rust
impl<T> PoolBuilder<T> {
    /// Set the maximum time a connection can be idle before being closed.
    ///
    /// # Default
    ///
    /// `600 seconds (10 minutes)`
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    /// use std::time::Duration;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .idle_timeout(Duration::from_secs(300))  // 5 minutes
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn idle_timeout(mut self, timeout: Duration) -> Self {
        self.config.idle_timeout = timeout;
        self
    }
}
```

---

#### max_lifetime()

```rust
impl<T> PoolBuilder<T> {
    /// Set the maximum lifetime of a connection.
    ///
    /// Connections older than this will be closed and replaced.
    ///
    /// # Default
    ///
    /// `1800 seconds (30 minutes)`
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    /// use std::time::Duration;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .max_lifetime(Duration::from_secs(3600))  // 1 hour
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn max_lifetime(mut self, lifetime: Duration) -> Self {
        self.config.max_lifetime = lifetime;
        self
    }
}
```

---

#### health_check_interval()

```rust
impl<T> PoolBuilder<T> {
    /// Set the interval between automatic health checks.
    ///
    /// # Default
    ///
    /// `30 seconds`
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    /// use std::time::Duration;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .health_check_interval(Duration::from_secs(60))  // Check every minute
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn health_check_interval(mut self, interval: Duration) -> Self {
        self.config.health_check_interval = interval;
        self
    }
}
```

---

#### health_check()

```rust
impl<T> PoolBuilder<T> {
    /// Set a custom health check function.
    ///
    /// The health check function is called:
    /// - Before acquiring a connection (if `test_on_acquire` is true)
    /// - Periodically by the background health check task
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .health_check(|conn| async move {
    ///         // Return Ok(()) if healthy, Err if unhealthy
    ///         conn.ping().await
    ///             .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
    ///     })
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// impl MyConnection {
    ///     async fn ping(&self) -> Result<(), Box<dyn std::error::Error>> {
    ///         Ok(())
    ///     }
    /// }
    /// ```
    pub fn health_check<F>(mut self, f: F) -> Self
    where
        F: Fn(&T) -> Pin<Box<dyn Future<Output = Result<(), Error>> + Send>> + Send + Sync + 'static,
    {
        self.health_check_fn = Some(Arc::new(f));
        self
    }
}
```

---

#### test_on_acquire()

```rust
impl<T> PoolBuilder<T> {
    /// Whether to test connections before giving them to the application.
    ///
    /// # Default
    ///
    /// `true`
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .test_on_acquire(false)  // Skip health check on acquire
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn test_on_acquire(mut self, test: bool) -> Self {
        self.config.test_on_acquire = test;
        self
    }
}
```

---

#### adaptive_sizing()

```rust
impl<T> PoolBuilder<T> {
    /// Enable adaptive pool sizing.
    ///
    /// When enabled, the pool will automatically grow and shrink based on load.
    ///
    /// # Default
    ///
    /// `false`
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .adaptive_sizing(true)  // Enable adaptive sizing
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn adaptive_sizing(mut self, enabled: bool) -> Self {
        self.config.adaptive_sizing_enabled = enabled;
        self
    }
}
```

---

#### metrics()

```rust
impl<T> PoolBuilder<T> {
    /// Set metrics collection.
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::{Pool, Metrics};
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .metrics(Metrics::prometheus())
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn metrics(mut self, metrics: Metrics) -> Self {
        self.metrics = Some(metrics);
        self
    }
}
```

---

#### build()

```rust
impl<T> PoolBuilder<T> {
    /// Build the pool with the configured settings.
    ///
    /// # Example
    ///
    /// ```rust
    /// use pool::Pool;
    ///
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let pool = Pool::builder()
    ///     .min_size(5)
    ///     .max_size(20)
    ///     .build(|| async {
    ///         MyConnection::connect("localhost:5432").await
    ///     }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// impl MyConnection {
    ///     async fn connect(_addr: &str) -> Result<Self, Box<dyn std::error::Error>> {
    ///         Ok(MyConnection)
    ///     }
    /// }
    /// ```
    pub async fn build<F, Fut>(self, factory: F) -> Result<Pool<T>, Error>
    where
        T: Send + 'static,
        F: Fn() -> Fut + Send + Sync + 'static,
        Fut: Future<Output = Result<T, Error>> + Send + 'static,
    {
        // Validate configuration
        self.config.validate()?;

        // Create pool inner
        let inner = Arc::new(PoolInner::new(self.config, factory, self.health_check_fn, self.metrics).await?);

        Ok(Pool { inner })
    }
}
```

---

## Connection Acquisition API

### pool.get()

Acquire a connection from the pool.

```rust
impl<T> Pool<T>
where
    T: Send + 'static,
{
    /// Acquire a connection from the pool.
    ///
    /// This will:
    /// 1. Try to get an idle connection from the pool
    /// 2. Create a new connection if under max_size
    /// 3. Wait in the queue if the pool is full
    ///
    /// Returns an error if:
    /// - The acquire timeout is exceeded
    /// - The pool is shutting down
    /// - Connection creation fails
    ///
    /// # Example
    ///
    /// ```rust
    /// # use pool::Pool;
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// # let pool: Pool<MyConnection> = unimplemented!();
    /// let conn = pool.get().await?;
    /// // Use connection
    /// conn.execute("SELECT * FROM users").await?;
    /// // Connection returned to pool when `conn` is dropped
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// # impl MyConnection {
    /// #     async fn execute(&self, _sql: &str) -> Result<(), Box<dyn std::error::Error>> {
    /// #         Ok(())
    /// #     }
    /// # }
    /// ```
    pub async fn get(&self) -> Result<PooledConnection<T>, Error> {
        self.inner.acquire().await
    }
}
```

---

### pool.try_get()

Try to acquire a connection without waiting.

```rust
impl<T> Pool<T>
where
    T: Send + 'static,
{
    /// Try to acquire a connection without waiting.
    ///
    /// Returns `None` immediately if no connections are available.
    ///
    /// # Example
    ///
    /// ```rust
    /// # use pool::Pool;
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// # let pool: Pool<MyConnection> = unimplemented!();
    /// if let Some(conn) = pool.try_get().await? {
    ///     conn.execute("SELECT * FROM users").await?;
    /// } else {
    ///     println!("No connections available");
    /// }
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// # impl MyConnection {
    /// #     async fn execute(&self, _sql: &str) -> Result<(), Box<dyn std::error::Error>> {
    /// #         Ok(())
    /// #     }
    /// # }
    /// ```
    pub async fn try_get(&self) -> Result<Option<PooledConnection<T>>, Error> {
        self.inner.try_acquire().await
    }
}
```

---

## Pool Status API

### pool.status()

Get the current status of the pool.

```rust
impl<T> Pool<T> {
    /// Get the current status of the pool.
    ///
    /// # Example
    ///
    /// ```rust
    /// # use pool::Pool;
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// # let pool: Pool<MyConnection> = unimplemented!();
    /// let status = pool.status();
    /// println!("Active: {}", status.active_connections);
    /// println!("Idle: {}", status.idle_connections);
    /// println!("Total: {}", status.total_connections);
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn status(&self) -> PoolStatus {
        self.inner.status()
    }
}

/// Pool status information.
///
/// # Example
///
/// ```rust
/// # use pool::PoolStatus;
/// let status = PoolStatus {
///     active_connections: 5,
///     idle_connections: 3,
///     total_connections: 8,
///     wait_queue_length: 0,
///     min_size: 5,
///     max_size: 20,
///     is_shutting_down: false,
/// };
/// println!("Pool: {}/{} connections ({} active)",
///     status.total_connections,
///     status.max_size,
///     status.active_connections
/// );
/// ```
#[derive(Debug, Clone)]
pub struct PoolStatus {
    /// Currently active (in-use) connections
    pub active_connections: usize,

    /// Currently idle (available) connections
    pub idle_connections: usize,

    /// Total connections (active + idle)
    pub total_connections: usize,

    /// Number of tasks waiting for a connection
    pub wait_queue_length: usize,

    /// Minimum pool size
    pub min_size: usize,

    /// Maximum pool size
    pub max_size: usize,

    /// Is the pool shutting down?
    pub is_shutting_down: bool,
}
```

---

## Health Check API

### pool.test()

Run health checks on all idle connections.

```rust
impl<T> Pool<T> {
    /// Run health checks on all idle connections.
    ///
    /// Unhealthy connections will be removed from the pool.
    ///
    /// # Example
    ///
    /// ```rust
    /// # use pool::Pool;
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// # let pool: Pool<MyConnection> = unimplemented!();
    /// let result = pool.test().await?;
    /// println!("Tested {} connections, {} unhealthy",
    ///     result.total_tested,
    ///     result.unhealthy_count
    /// );
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub async fn test(&self) -> Result<HealthCheckResult, Error> {
        self.inner.test_all().await
    }
}

/// Result of a health check operation.
#[derive(Debug)]
pub struct HealthCheckResult {
    /// Total connections tested
    pub total_tested: usize,

    /// Unhealthy connections found
    pub unhealthy_count: usize,

    /// Unhealthy connections replaced
    pub replaced_count: usize,
}
```

---

## Graceful Shutdown API

### pool.shutdown()

Shutdown the pool gracefully.

```rust
impl<T> Pool<T> {
    /// Shutdown the pool gracefully.
    ///
    /// This will:
    /// 1. Stop accepting new acquire() calls
    /// 2. Wait for in-flight connections (up to timeout)
    /// 3. Close all connections
    ///
    /// # Example
    ///
    /// ```rust
    /// # use pool::Pool;
    /// # use std::time::Duration;
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// # let pool: Pool<MyConnection> = unimplemented!();
    /// pool.shutdown()
    ///     .timeout(Duration::from_secs(30))
    ///     .await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn shutdown(&self) -> ShutdownHandle<T> {
        ShutdownHandle::new(self.inner.clone())
    }
}

/// Handle for gracefully shutting down a pool.
pub struct ShutdownHandle<T> {
    inner: Arc<PoolInner<T>>,
}

impl<T> ShutdownHandle<T> {
    /// Set the shutdown timeout.
    ///
    /// # Example
    ///
    /// ```rust
    /// # use pool::Pool;
    /// # use std::time::Duration;
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// # let pool: Pool<MyConnection> = unimplemented!();
    /// pool.shutdown()
    ///     .timeout(Duration::from_secs(60))
    ///     .await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn timeout(mut self, timeout: Duration) -> Self {
        self.inner.set_shutdown_timeout(timeout);
        self
    }

    /// Execute the shutdown.
    ///
    /// # Example
    ///
    /// ```rust
    /// # use pool::Pool;
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// # let pool: Pool<MyConnection> = unimplemented!();
    /// pool.shutdown().await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub async fn shutdown(self) -> Result<(), Error> {
        self.inner.shutdown().await
    }
}
```

---

## Metrics API

### Metrics::prometheus()

Create a Prometheus metrics collector.

```rust
impl Metrics {
    /// Create a Prometheus metrics collector.
    ///
    /// # Example
    ///
    /// ```rust
    /// # use pool::{Pool, Metrics};
    /// # #[tokio::main]
    /// # async fn main() -> Result<(), Box<dyn std::error::Error>> {
    /// let metrics = Metrics::prometheus();
    /// let pool = Pool::builder()
    ///     .metrics(metrics)
    ///     .build(|| async { Ok(MyConnection) }).await?;
    /// # Ok(())
    /// # }
    /// # struct MyConnection;
    /// ```
    pub fn prometheus() -> Self {
        Self::new(Collector::Prometheus)
    }

    /// Export metrics in Prometheus format.
    ///
    /// # Example
    ///
    /// ```rust
    /// # use pool::Metrics;
    /// let metrics = Metrics::prometheus();
    /// let prometheus_text = metrics.export_prometheus();
    /// println!("{}", prometheus_text);
    /// ```
    pub fn export_prometheus(&self) -> String {
        // Export metrics in Prometheus text format
    }
}
```

---

## Error Handling API

### Error Type

```rust
/// Pool error types.
#[derive(Debug, thiserror::Error)]
pub enum Error {
    /// Connection creation failed.
    #[error("Connection failed: {0}")]
    ConnectionFailed(String),

    /// Health check failed.
    #[error("Health check failed: {0}")]
    HealthCheckFailed(String),

    /// Pool is exhausted (max_size reached).
    #[error("Pool exhausted")]
    PoolExhausted,

    /// Acquire timeout exceeded.
    #[error("Acquire timeout exceeded")]
    AcquireTimeout,

    /// Pool is shutting down.
    #[error("Pool is shutting down")]
    PoolShuttingDown,

    /// Invalid configuration.
    #[error("Invalid configuration: {0}")]
    InvalidConfig(String),

    /// Internal error.
    #[error("Internal error: {0}")]
    Internal(String),
}
```

---

## Type Safety API

### Type-Safe Pool Usage

The pool is generic over the connection type, providing compile-time type safety.

```rust
// Database connection pool
let db_pool: Pool<DbConnection> = Pool::new(|| async {
    DbConnection::connect("postgres://...").await
}).await?;

// HTTP client pool
let http_pool: Pool<HttpClient> = Pool::new(|| async {
    HttpClient::new("https://api.example.com").await
}).await?;

// Compile error: Can't use DbConnection pool for HttpClient
let db_conn: DbConnection = db_pool.get().await?;  // ✅ OK
let http_conn: HttpClient = http_pool.get().await?;  // ✅ OK
let wrong: HttpClient = db_pool.get().await?;  // ❌ Compile error!
```

---

## Complete API Example

```rust
use pool::{Pool, Metrics};
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create pool with custom configuration
    let pool = Pool::builder()
        .min_size(5)
        .max_size(100)
        .initial_size(10)
        .acquire_timeout(Duration::from_secs(10))
        .idle_timeout(Duration::from_secs(300))
        .max_lifetime(Duration::from_secs(1800))
        .health_check_interval(Duration::from_secs(60))
        .health_check(|conn| async move {
            conn.ping().await
                .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
        })
        .test_on_acquire(true)
        .adaptive_sizing(true)
        .metrics(Metrics::prometheus())
        .build(|| async {
            MyConnection::connect("localhost:5432").await
        }).await?;

    // Use pool
    let conn = pool.get().await?;
    conn.execute("SELECT * FROM users").await?;
    // Connection automatically returned to pool

    // Check pool status
    let status = pool.status();
    println!("Pool: {}/{} connections",
        status.total_connections,
        status.max_size
    );

    // Run health checks
    let health_result = pool.test().await?;
    println!("Health: {} tested, {} unhealthy",
        health_result.total_tested,
        health_result.unhealthy_count
    );

    // Shutdown gracefully
    pool.shutdown()
        .timeout(Duration::from_secs(30))
        .await?;

    Ok(())
}

struct MyConnection;

impl MyConnection {
    async fn connect(addr: &str) -> Result<Self, Box<dyn std::error::Error>> {
        Ok(MyConnection)
    }

    async fn ping(&self) -> Result<(), Box<dyn std::error::Error>> {
        Ok(())
    }

    async fn execute(&self, sql: &str) -> Result<(), Box<dyn std::error::Error>> {
        Ok(())
    }
}
```

---

## Summary

The pool-rs API is designed to be:

- **Simple**: `Pool::new()` for basic use cases
- **Powerful**: Builder pattern for advanced configuration
- **Type-Safe**: Generic pool with compile-time type checking
- **Async-First**: All operations are async
- **Observable**: Built-in metrics and tracing
- **Resilient**: Health checking and graceful shutdown

---

**Next**: [User Guide](04-user-guide.md) - Getting started with pool-rs

---

**Document Status**: ✅ Complete
**Word Count**: ~3,500 words
**API Complete**: Yes
