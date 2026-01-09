# Pool-RS User Guide

## Getting Started with Pool-RS

Welcome to the pool-rs user guide! This guide will help you get started with connection pooling in Rust.

### What is Pool-RS?

Pool-RS is a **generic connection pool** for Rust that makes managing database, HTTP, and other network connections simple and efficient.

**Key Features**:
- ✅ **Simple API**: Get started in 3 lines of code
- ✅ **Type-Safe**: Compile-time type checking prevents errors
- ✅ **Async-First**: Built on tokio for modern async Rust
- ✅ **Health Checking**: Automatic connection validation
- ✅ **Metrics**: Built-in observability
- ✅ **Adaptive**: Dynamic pool sizing based on load

---

## Quick Start (5 Minutes)

### Installation

Add to your `Cargo.toml`:

```toml
[dependencies]
pool = "0.1"
tokio = { version = "1", features = ["full"] }
```

### Hello World

```rust
use pool::Pool;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create a pool (3 lines!)
    let pool = Pool::new(|| async {
        MyConnection::connect("localhost:5432").await
    }).await?;

    // Acquire and use connection
    let conn = pool.get().await?;
    conn.query("SELECT * FROM users").await?;

    Ok(())
}

struct MyConnection;

impl MyConnection {
    async fn connect(_addr: &str) -> Result<Self, Box<dyn std::error::Error>> {
        Ok(MyConnection)
    }

    async fn query(&self, _sql: &str) -> Result<(), Box<dyn std::error::Error>> {
        Ok(())
    }
}
```

That's it! You now have a connection pool that:
- Manages up to 10 connections (default max_size)
- Automatically returns connections to the pool when done
- Handles connection creation and cleanup

---

## Core Concepts

### 1. Connections

A **connection** is any resource that:
- Can be created
- Can be used to perform operations
- Can be validated (health check)
- Can be closed/destroyed

Examples:
- Database connections (Postgres, MySQL, SQLite)
- HTTP clients
- gRPC connections
- Redis connections
- Custom connections

### 2. The Pool

The **pool** manages connections:
- **Idle connections**: Available for use
- **Active connections**: Currently in use by the application
- **Wait queue**: Tasks waiting for connections

### 3. Connection Lifecycle

```
Created ──> Idle ──> Active ──> Returned ──> Idle ──> ... ──> Destroyed
```

1. **Created**: Pool creates connection (if under max_size)
2. **Idle**: Connection available in pool
3. **Active**: Application acquires connection
4. **Returned**: Application finishes, connection returns to pool
5. **Destroyed**: Connection closed (idle_timeout, max_lifetime, unhealthy)

---

## Pool Configuration

### Default Configuration

```rust
Pool::new(|| async { /* ... */ }).await?
```

Defaults:
- `min_size: 0` - No minimum idle connections
- `max_size: 10` - Maximum 10 connections
- `acquire_timeout: 30s` - Wait up to 30s for connection
- `idle_timeout: 10min` - Close idle connections after 10min
- `max_lifetime: 30min` - Close connections after 30min
- `health_check_enabled: true` - Enable health checks
- `health_check_interval: 30s` - Check health every 30s
- `test_on_acquire: true` - Check health before acquire

---

### Custom Configuration

Use the builder for custom configuration:

```rust
use pool::Pool;
use std::time::Duration;

let pool = Pool::builder()
    .min_size(5)                           // Maintain 5 idle connections
    .max_size(100)                         // Maximum 100 connections
    .initial_size(10)                      // Pre-create 10 connections
    .acquire_timeout(Duration::from_secs(10))  // 10s acquire timeout
    .idle_timeout(Duration::from_secs(300))    // 5min idle timeout
    .max_lifetime(Duration::from_secs(1800))   // 30min max lifetime
    .health_check_interval(Duration::from_secs(60))  // Check every 60s
    .build(|| async {
        MyConnection::connect("localhost:5432").await
    }).await?;
```

---

### Pool Sizing

Choosing the right pool size is important:

**Formula**:
```
pool_size = core_count * (1 + async_tasks_per_core)
```

**Example** (4-core CPU, 100 async tasks):
```
pool_size = 4 * (1 + 100/4) = 4 * 26 = 104 connections
```

**Guidelines**:
- **Small apps** (1-10 requests/s): 5-10 connections
- **Medium apps** (10-100 requests/s): 10-50 connections
- **Large apps** (100+ requests/s): 50-200 connections
- **Microservices**: Start with 10, scale based on metrics

**Don't oversize!** Too many connections can:
- Overwhelm the database
- Increase memory usage
- Cause contention

---

## Connection Lifecycle

### Acquiring Connections

**Basic acquire**:
```rust
let conn = pool.get().await?;
```

**Try acquire without waiting**:
```rust
if let Some(conn) = pool.try_get().await? {
    // Use connection
} else {
    // No connections available
}
```

### Using Connections

```rust
let mut conn = pool.get().await?;

// Use connection
conn.query("SELECT * FROM users").await?;
conn.execute("INSERT INTO users VALUES (1, 'Alice')").await?;

// Connection automatically returned to pool when dropped
```

### Releasing Connections

Connections are automatically returned to the pool when dropped:

```rust
{
    let conn = pool.get().await?;
    conn.query("SELECT 1").await?;
} // Connection returned to pool here
```

---

## Health Checking

### Default Health Check

By default, connections are not health-checked (no-op). You should provide a health check:

```rust
let pool = Pool::builder()
    .health_check(|conn| async move {
        // Return Ok(()) if healthy
        conn.ping().await
            .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
    })
    .build(|| async {
        MyConnection::connect("localhost:5432").await
    }).await?;
```

### Database Health Checks

**PostgreSQL**:
```rust
health_check(|conn| async move {
    conn.simple_query("SELECT 1").await
        .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
})
```

**MySQL**:
```rust
health_check(|conn| async move {
    conn.query("SELECT 1").await
        .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
})
```

**Redis**:
```rust
health_check(|conn| async move {
    conn.ping().await
        .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
})
```

### Manual Health Check

Test all connections manually:

```rust
let result = pool.test().await?;
println!("Tested: {}, Unhealthy: {}, Replaced: {}",
    result.total_tested,
    result.unhealthy_count,
    result.replaced_count
);
```

---

## Pool Resizing

### Static Pool Size

By default, pool size is static:

```rust
let pool = Pool::builder()
    .min_size(5)
    .max_size(50)
    .build(/*...*/).await?;
// Pool stays between 5 and 50 connections
```

### Adaptive Pool Sizing

Enable adaptive sizing for dynamic pools:

```rust
let pool = Pool::builder()
    .min_size(5)
    .max_size(100)
    .adaptive_sizing(true)  // Enable adaptive sizing
    .build(/*...*/).await?;
```

**Behavior**:
- **Grows** when pool is 80% full
- **Shrinks** idle connections after 5 minutes
- Automatically adjusts to load

**Customize adaptive sizing**:
```rust
let pool = Pool::builder()
    .adaptive_sizing(true)
    .adaptive_grow_threshold(0.9)  // Grow when 90% full
    .adaptive_shrink_idle_secs(600)  // Shrink after 10min idle
    .adaptive_predictive(true)  // Use historical data
    .build(/*...*/).await?;
```

---

## Monitoring

### Pool Status

Check pool status:

```rust
let status = pool.status();
println!("Active: {}", status.active_connections);
println!("Idle: {}", status.idle_connections);
println!("Total: {}", status.total_connections);
println!("Wait queue: {}", status.wait_queue_length);
println!("Is shutting down: {}", status.is_shutting_down);
```

### Metrics

Enable metrics:

```rust
use pool::Metrics;

let pool = Pool::builder()
    .metrics(Metrics::prometheus())
    .build(/*...*/).await?;
```

**Export metrics**:
```rust
let prometheus_text = pool.metrics().export_prometheus();
println!("{}", prometheus_text);
```

**Metrics collected**:
- `pool_active_connections` (gauge)
- `pool_idle_connections` (gauge)
- `pool_wait_queue_length` (gauge)
- `pool_acquire_duration_seconds` (histogram)
- `pool_connections_created_total` (counter)
- `pool_connections_destroyed_total` (counter)
- `pool_health_check_failures_total` (counter)

---

## Common Patterns

### Pattern 1: Database Connection Pool

```rust
use pool::Pool;
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pool = Pool::builder()
        .min_size(5)
        .max_size(50)
        .acquire_timeout(Duration::from_secs(10))
        .health_check_interval(Duration::from_secs(30))
        .health_check(|conn| async move {
            conn.query("SELECT 1").await
                .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
        })
        .build(|| async {
            DbConnection::connect("postgres://user:pass@localhost/mydb").await
        }).await?;

    // Use pool
    let conn = pool.get().await?;
    let rows = conn.query("SELECT * FROM users").await?;

    Ok(())
}
```

---

### Pattern 2: HTTP Client Pool

```rust
use pool::Pool;

let pool = Pool::builder()
    .max_size(20)
    .idle_timeout(Duration::from_secs(300))
    .build(|| async {
        HttpClient::new("https://api.example.com").await
    }).await?;

// Use HTTP client
let client = pool.get().await?;
let response = client.get("/users").await?;
```

---

### Pattern 3: Connection Timeout Handling

```rust
use pool::Error;

match pool.get().await {
    Ok(conn) => {
        // Use connection
        conn.query("SELECT 1").await?;
    }
    Err(Error::AcquireTimeout) => {
        eprintln!("Timed out waiting for connection");
    }
    Err(Error::PoolShuttingDown) => {
        eprintln!("Pool is shutting down");
    }
    Err(e) => {
        eprintln!("Error: {}", e);
    }
}
```

---

### Pattern 4: Graceful Shutdown

```rust
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pool = Pool::new(/*...*/).await?;

    // Run application
    tokio::spawn(async move {
        run_server(pool).await
    });

    // Wait for shutdown signal
    signal::ctrl_c().await?;

    // Shutdown pool gracefully
    println!("Shutting down pool...");
    pool.shutdown()
        .timeout(Duration::from_secs(30))
        .await?;

    Ok(())
}
```

---

### Pattern 5: Multiple Pools

```rust
// Database pool
let db_pool: Pool<DbConnection> = Pool::new(|| async {
    DbConnection::connect("postgres://...").await
}).await?;

// Redis pool
let redis_pool: Pool<RedisConnection> = Pool::new(|| async {
    RedisConnection::connect("redis://localhost").await
}).await?;

// Use both pools
let db_conn = db_pool.get().await?;
let redis_conn = redis_pool.get().await?;
```

---

## Best Practices

### 1. Always Set Timeouts

```rust
Pool::builder()
    .acquire_timeout(Duration::from_secs(10))  // Don't wait forever
    .health_check_interval(Duration::from_secs(30))
    .build(/*...*/).await?
```

### 2. Use Health Checks

```rust
.health_check(|conn| async move {
    conn.ping().await
        .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
})
```

### 3. Monitor Your Pool

```rust
let status = pool.status();
if status.wait_queue_length > 10 {
    eprintln!("Warning: Pool wait queue is long!");
}
```

### 4. Enable Metrics in Production

```rust
Pool::builder()
    .metrics(Metrics::prometheus())
    .build(/*...*/).await?
```

### 5. Shutdown Gracefully

```rust
pool.shutdown()
    .timeout(Duration::from_secs(30))
    .await?;
```

### 6. Choose Right Pool Size

- Start small (10 connections)
- Monitor metrics
- Scale based on usage
- Don't oversize!

---

## Troubleshooting

### Problem: "Pool exhausted" or long wait times

**Cause**: Pool is too small for your load

**Solution**:
```rust
// Increase max_size
Pool::builder()
    .max_size(100)  // Increase from 10
    .build(/*...*/).await?;
```

**Or enable adaptive sizing**:
```rust
Pool::builder()
    .adaptive_sizing(true)
    .build(/*...*/).await?;
```

---

### Problem: Connections are stale/unhealthy

**Cause**: Health checks not enabled or configured

**Solution**:
```rust
Pool::builder()
    .health_check_interval(Duration::from_secs(30))  // Check every 30s
    .test_on_acquire(true)  // Check before acquire
    .health_check(|conn| async move {
        conn.ping().await
            .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
    })
    .build(/*...*/).await?;
```

---

### Problem: High memory usage

**Cause**: Pool too large or connections not being released

**Solution**:
```rust
// Reduce pool size
Pool::builder()
    .max_size(20)  // Reduce from 100
    .build(/*...*/).await?;

// Ensure connections are released (drop them)
{
    let conn = pool.get().await?;
    conn.query("SELECT 1").await?;
} // Connection released here
```

---

### Problem: Connection creation failures

**Cause**: Network issues, database down, or wrong connection string

**Solution**:
1. Check connection string
2. Test connectivity
3. Handle errors:

```rust
match pool.get().await {
    Ok(conn) => { /* use conn */ }
    Err(Error::ConnectionFailed(e)) => {
        eprintln!("Connection failed: {}", e);
        // Retry or fallback
    }
    Err(e) => {
        eprintln!("Error: {}", e);
    }
}
```

---

## Complete Tutorial: Protect a Database

Let's build a complete example of protecting a database with a connection pool.

### Step 1: Add Dependencies

```toml
[dependencies]
pool = "0.1"
tokio = { version = "1", features = ["full"] }
tokio-postgres = "0.7"
```

### Step 2: Create the Pool

```rust
use pool::Pool;
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let pool = Pool::builder()
        .min_size(5)
        .max_size(50)
        .initial_size(10)
        .acquire_timeout(Duration::from_secs(10))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .health_check_interval(Duration::from_secs(60))
        .health_check(|client| async move {
            client.simple_query("SELECT 1").await
                .map(|_| ())
                .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
        })
        .build(|| async {
            tokio_postgres::connect(
                "host=localhost user=postgres dbname=mydb",
                NoTls
            ).await
            .map(|(client, _)| client)
            .map_err(|e| pool::Error::ConnectionFailed(e.to_string()))
        }).await?;

    Ok(())
}
```

### Step 3: Use the Pool

```rust
async fn get_user(pool: &Pool<Client>, user_id: i32) -> Result<Option<User>, Box<dyn std::error::Error>> {
    let client = pool.get().await?;

    let row = client.query_one(
        "SELECT id, name, email FROM users WHERE id = $1",
        &[&user_id]
    ).await?;

    Ok(Some(User {
        id: row.get(0),
        name: row.get(1),
        email: row.get(2),
    }))
}

#[derive(Debug)]
struct User {
    id: i32,
    name: String,
    email: String,
}
```

### Step 4: Monitor the Pool

```rust
tokio::spawn(async move {
    loop {
        tokio::time::sleep(Duration::from_secs(10)).await;
        let status = pool.status();
        println!("Pool: {}/{} ({} active, {} idle, {} waiting)",
            status.total_connections,
            status.max_size,
            status.active_connections,
            status.idle_connections,
            status.wait_queue_length
        );
    }
});
```

### Step 5: Graceful Shutdown

```rust
tokio::select! {
    _ = tokio::signal::ctrl_c() => {
        println!("Shutting down...");
        pool.shutdown()
            .timeout(Duration::from_secs(30))
            .await?;
        println!("Shutdown complete");
    }
    result = run_server(&pool) => {
        result?;
    }
}
```

### Complete Code

```rust
use pool::Pool;
use std::time::Duration;
use tokio_postgres::{NoTls, Client};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create pool
    let pool = Pool::builder()
        .min_size(5)
        .max_size(50)
        .initial_size(10)
        .acquire_timeout(Duration::from_secs(10))
        .health_check_interval(Duration::from_secs(60))
        .health_check(|client| async move {
            client.simple_query("SELECT 1").await
                .map(|_| ())
                .map_err(|e| pool::Error::HealthCheckFailed(e.to_string()))
        })
        .build(|| async {
            tokio_postgres::connect(
                "host=localhost user=postgres dbname=mydb",
                NoTls
            ).await
            .map(|(client, _)| client)
            .map_err(|e| pool::Error::ConnectionFailed(e.to_string()))
        }).await?;

    // Start monitoring
    let pool_monitor = pool.clone();
    tokio::spawn(async move {
        loop {
            tokio::time::sleep(Duration::from_secs(10)).await;
            let status = pool_monitor.status();
            println!("Pool: {}/{} ({} active, {} idle)",
                status.total_connections,
                status.max_size,
                status.active_connections,
                status.idle_connections
            );
        }
    });

    // Run application
    run_app(&pool).await?;

    Ok(())
}

async fn run_app(pool: &Pool<Client>) -> Result<(), Box<dyn std::error::Error>> {
    // Use pool
    let client = pool.get().await?;
    let rows = client.query("SELECT * FROM users", &[]).await?;
    println!("Found {} users", rows.len());
    Ok(())
}
```

---

## Summary

You've learned:
- ✅ How to create a connection pool
- ✅ How to configure pool settings
- ✅ How to use health checks
- ✅ How to enable adaptive sizing
- ✅ How to monitor your pool
- ✅ How to shutdown gracefully

**Next**: [Developer Guide](05-dev-guide.md) - Contributing to pool-rs

---

**Document Status**: ✅ Complete
**Word Count**: ~3,000 words
