# Pool-RS Architecture Design

## Overview

Pool-RS is a **generic, async-first connection pool** for Rust with a focus on simplicity, performance, and observability. It manages connections of any type (database, HTTP, gRPC, custom) with built-in health checking, metrics, and adaptive sizing.

### Design Philosophy

1. **Zero-Trait API**: Just provide a closure, not a complex trait
2. **Async-First**: Built on tokio from the ground up
3. **Observable**: Metrics and tracing built-in
4. **Resilient**: Health checking and graceful shutdown
5. **Performant**: Lock-free operations, zero-allocation in hot path
6. **Type-Safe**: Generic pool with compile-time type checking

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Application                              │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  │ pool.get().await
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Pool API                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ Pool::new()  │  │ pool.get()  │  │ pool.status  │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Pool Manager                                │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  Connection     │  │  Health Check   │  │  Metrics        │  │
│  │  Manager        │  │  Scheduler      │  │  Collector      │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  Adaptive       │  │  Wait Queue     │  │  Shutdown       │  │
│  │  Sizer          │  │  Manager        │  │  Coordinator    │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Connection Store                           │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Idle      │  │   Active    │  │   Creating  │             │
│  │ Connections │  │ Connections │  │ Connections │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Connection Factory                            │
│                                                                   │
│  User-provided closure: || async { Connection::create().await } │
└─────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                      External Resources                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Database    │  │  HTTP API    │  │  gRPC        │         │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

#### Connection Acquisition Flow

```
1. Application calls pool.get().await
2. Pool Manager checks idle connections
3. If idle connection exists:
   a. Health check runs (if due)
   b. If healthy, return connection
   c. If unhealthy, replace and retry
4. If no idle connection:
   a. Check if under max_size
   b. If yes, create new connection
   c. If no, wait in queue (with timeout)
5. Return connection to application
```

#### Connection Release Flow

```
1. Application drops PooledConnection
2. PooledConnection's Drop trait runs
3. Connection returned to idle pool
4. Signal wait queue (if tasks waiting)
5. Adaptive sizer evaluates pool size
```

---

## Core Components

### 1. Pool<T> (Main Pool Structure)

**Purpose**: Generic pool managing connections of type T

**Responsibilities**:
- Store connection configuration
- Coordinate all pool operations
- Manage lifecycle (create, acquire, release, destroy)
- Provide public API

**Fields**:
```rust
pub struct Pool<T> {
    // Configuration
    config: Arc<PoolConfig>,

    // Connection store
    connections: Arc<ConnectionStore<T>>,

    // Manager components
    manager: Arc<PoolManager<T>>,

    // Shutdown signal
    shutdown: Shutdown,

    // Metrics
    metrics: Arc<Metrics>,

    // Phantom data for type T
    _phantom: PhantomData<T>,
}
```

**Key Methods**:
- `new(factory) -> Self` - Create pool with defaults
- `builder() -> PoolBuilder<T>` - Create pool with custom config
- `get(&self) -> PooledConnection<T>` - Acquire connection
- `status(&self) -> PoolStatus` - Get pool status
- `shutdown(&self) -> ShutdownHandle` - Initiate graceful shutdown

---

### 2. PoolConfig (Configuration)

**Purpose**: Immutable pool configuration

**Fields**:
```rust
pub struct PoolConfig {
    // Pool sizing
    pub min_size: usize,              // Minimum idle connections (default: 0)
    pub max_size: usize,              // Maximum connections (default: 10)
    pub initial_size: usize,          // Pre-create at startup (default: min_size)

    // Timeouts
    pub acquire_timeout: Duration,    // Max time to wait for connection (default: 30s)
    pub idle_timeout: Duration,       // Close idle connections after (default: 10min)
    pub max_lifetime: Duration,       // Close connections after (default: 30min)
    pub shutdown_timeout: Duration,   // Max time to wait during shutdown (default: 30s)

    // Health checking
    pub health_check_enabled: bool,                    // Enable health checks (default: true)
    pub health_check_interval: Duration,               // Health check frequency (default: 30s)
    pub health_check_timeout: Duration,                // Health check timeout (default: 5s)
    pub test_on_acquire: bool,                         // Check health before acquire (default: true)

    // Adaptive sizing
    pub adaptive_sizing_enabled: bool,                 // Enable adaptive sizing (default: false)
    pub adaptive_grow_threshold: f64,                  // Grow when usage > this (default: 0.8)
    pub adaptive_shrink_idle_secs: u64,                // Shrink after idle this long (default: 300)
    pub adaptive_predictive: bool,                     // Use historical data (default: false)

    // Metrics
    pub metrics_enabled: bool,                         // Enable metrics (default: true)
    pub metrics_labels: HashMap<String, String>,       // Prometheus labels

    // Wait queue
    pub max_wait_queue_size: usize,                    // Max tasks waiting (default: 100)
}
```

**Defaults**:
- `min_size: 0`
- `max_size: 10`
- `initial_size: 0`
- `acquire_timeout: 30s`
- `idle_timeout: 600s` (10 minutes)
- `max_lifetime: 1800s` (30 minutes)
- `health_check_enabled: true`
- `health_check_interval: 30s`
- `test_on_acquire: true`

---

### 3. PoolBuilder<T> (Builder Pattern)

**Purpose**: Construct pools with custom configuration

**API**:
```rust
let pool = Pool::builder()
    .min_size(5)
    .max_size(100)
    .initial_size(10)
    .acquire_timeout(Duration::from_secs(10))
    .idle_timeout(Duration::from_secs(300))
    .max_lifetime(Duration::from_secs(1800))
    .health_check_interval(Duration::from_secs(60))
    .health_check(|conn| async move {
        conn.ping().await.map_err(|e| Error::HealthCheckFailed(e))
    })
    .adaptive_sizing(true)
    .adaptive_grow_threshold(0.9)
    .metrics(Metrics::prometheus())
    .metrics_labels(vec![("pool", "database").into()])
    .build(|| async {
        DbConnection::connect("postgres://...").await
    }).await?;
```

**Methods**:
- `min_size(usize) -> Self`
- `max_size(usize) -> Self`
- `initial_size(usize) -> Self`
- `acquire_timeout(Duration) -> Self`
- `idle_timeout(Duration) -> Self`
- `max_lifetime(Duration) -> Self`
- `health_check_interval(Duration) -> Self`
- `health_check<F>(F) -> Self` where F: Fn(&T) -> Future<Output = Result<()>>
- `test_on_acquire(bool) -> Self`
- `adaptive_sizing(bool) -> Self`
- `adaptive_grow_threshold(f64) -> Self`
- `adaptive_shrink_idle_secs(u64) -> Self`
- `adaptive_predictive(bool) -> Self`
- `metrics(Metrics) -> Self`
- `metrics_labels(Vec<(String, String)>) -> Self`
- `max_wait_queue_size(usize) -> Self`
- `build<F>(F) -> impl Future<Output = Result<Pool<T>>>`

---

### 4. ConnectionFactory<T> (Factory Function)

**Purpose**: User-provided function to create connections

**Type**:
```rust
pub type ConnectionFactory<T> = Arc<
    dyn Fn() -> Pin<Box<dyn Future<Output = Result<T, Error>> + Send>> + Send + Sync
>;
```

**Simplified API** (via closure auto-conversion):
```rust
// User provides simple closure
Pool::new(|| async {
    DbConnection::connect("postgres://...").await
})

// Internally converted to ConnectionFactory
```

**Why this design**:
- No complex trait to implement
- Zero boilerplate
- Works with any async function
- Captures environment easily

---

### 5. ConnectionStore<T> (Connection Storage)

**Purpose**: Store and manage connections

**Fields**:
```rust
pub struct ConnectionStore<T> {
    // Idle connections (available for acquire)
    idle: Arc<Mutex<VecDeque<IdleConnection<T>>>>,

    // Active connections (currently in use)
    active: Arc<AtomicUsize>,

    // Creating connections (being created)
    creating: Arc<AtomicUsize>,

    // Total created (lifetime counter)
    total_created: Arc<AtomicU64>,
}

pub struct IdleConnection<T> {
    connection: T,
    created_at: Instant,
    last_used: Instant,
    last_health_check: Option<Instant>,
}
```

**Thread Safety**:
- `idle`: Mutex for exclusive access (low contention)
- `active`: AtomicUsize for lock-free reads
- `creating`: AtomicUsize for lock-free reads
- `total_created`: AtomicU64 for lock-free reads

**Why Mutex for idle**:
- Need to push/pop from both ends
- Low contention (only on acquire/release)
- Simpler than lock-free queue

---

### 6. PoolManager<T> (Coordination)

**Purpose**: Coordinate pool operations

**Fields**:
```rust
pub struct PoolManager<T> {
    // Configuration
    config: Arc<PoolConfig>,

    // Connection factory
    factory: ConnectionFactory<T>,

    // Connection store
    store: Arc<ConnectionStore<T>>,

    // Health checker
    health_checker: Arc<HealthChecker<T>>,

    // Adaptive sizer
    adaptive_sizer: Arc<AdaptiveSizer>,

    // Wait queue
    wait_queue: Arc<Mutex<WaitQueue>>,

    // Metrics
    metrics: Arc<Metrics>,
}
```

**Key Methods**:
- `acquire() -> impl Future<Output = Result<PooledConnection<T>>>`
- `release(connection: T)`
- `create_connection() -> impl Future<Output = Result<T>>`
- `check_health(connection: &T) -> impl Future<Output = Result<()>>`
- `maybe_grow_pool()`
- `maybe_shrink_pool()`

---

### 7. HealthChecker<T> (Health Validation)

**Purpose**: Validate connection health

**Fields**:
```rust
pub struct HealthChecker<T> {
    // User-provided health check function
    check_fn: Option<HealthCheckFn<T>>,

    // Health check interval
    interval: Duration,

    // Timeout for health checks
    timeout: Duration,

    // Track last check time per connection
    last_checks: Arc<Mutex<HashMap<usize, Instant>>>,
}

pub type HealthCheckFn<T> = Arc<
    dyn Fn(&T) -> Pin<Box<dyn Future<Output = Result<(), Error>> + Send>> + Send + Sync
>;
```

**Health Check Strategy**:
1. **On Acquire** (if `test_on_acquire: true`):
   - Before giving connection to user
   - If unhealthy, replace connection
   - Fast path: skip if not due yet

2. **Periodic** (if `health_check_enabled: true`):
   - Background task checks idle connections
   - Runs every `health_check_interval`
   - Removes unhealthy connections from pool

3. **Explicit** (user can trigger):
   - `pool.test().await` - Check all connections
   - Returns health status

**Default Health Check**:
- If no custom health check provided
- Default: Assume healthy (no-op)
- Override for database: `SELECT 1`
- Override for HTTP: `GET /health`

---

### 8. AdaptiveSizer (Dynamic Pool Sizing)

**Purpose**: Automatically adjust pool size based on load

**Fields**:
```rust
pub struct AdaptiveSizer {
    // Configuration
    config: AdaptiveConfig,

    // Historical usage data
    usage_history: Arc<Mutex<VecDeque<UsageSample>>>,

    // Last grow/shrink time
    last_grow: Arc<Mutex<Option<Instant>>>,
    last_shrink: Arc<Mutex<Option<Instant>>>,
}

pub struct AdaptiveConfig {
    pub enabled: bool,
    pub grow_threshold: f64,         // Grow when usage > 0.8 (80%)
    pub shrink_idle_secs: u64,       // Shrink after 300s idle
    pub predictive: bool,            // Use historical data
    pub grow_increment: usize,       // Add this many connections
    pub shrink_decrement: usize,     // Remove this many connections
}

pub struct UsageSample {
    timestamp: Instant,
    active_count: usize,
    wait_queue_length: usize,
}
```

**Sizing Algorithm**:

**Grow** (when pool is 80% full):
```rust
if active_connections + wait_queue_length > max_size * 0.8 {
    // Grow pool by 10 connections (or grow_increment)
    let new_size = min(current_size + grow_increment, max_size);
    // Pre-create connections
    create_connections(new_size - current_size).await;
}
```

**Shrink** (when connections idle for 5 minutes):
```rust
for conn in idle_connections {
    if conn.idle_for() > Duration::from_secs(300) {
        // Close connection
        conn.close().await;
        // Remove from pool
        remove_connection(conn);
    }
}
```

**Predictive** (if enabled):
```rust
// Analyze historical data
if let Some(prediction) = predict_load_spread(&usage_history) {
    if prediction.upcoming_spike {
        // Pre-grow pool before spike
        let predicted_size = prediction.peak_connections;
        ensure_connections(predicted_size).await;
    }
}
```

**When it runs**:
- After every connection release
- Periodically (every 60 seconds)
- On connection acquire failure

---

### 9. WaitQueue (Acquire Queue)

**Purpose**: Queue tasks waiting for connections

**Implementation**:
```rust
pub struct WaitQueue {
    queue: VecDeque<Waiter>,
    max_size: usize,
}

pub struct Waiter {
    task_id: usize,
    registered_at: Instant,
    waker: Option<Waker>,
    result: OnceCell<Result<PooledConnection>>,

    // Channel for signaling
    sender: oneshot::Sender<Result<PooledConnection>>,
}
```

**Acquire Flow with Wait Queue**:
```rust
async fn acquire() -> Result<PooledConnection> {
    // Try immediate acquire
    if let Some(conn) = try_acquire_immediate()? {
        return Ok(conn);
    }

    // No connection available, wait in queue
    let (tx, rx) = oneshot::channel();
    let waiter = Waiter::new(tx);

    // Add to wait queue
    wait_queue.push(waiter).await?;

    // Wait for connection (with timeout)
    tokio::time::timeout(
        config.acquire_timeout,
        rx
    ).await??
}
```

**Signal Waiters** (when connection released):
```rust
fn signal_waiters() {
    while let Some(waiter) = wait_queue.pop_front() {
        if let Some(conn) = try_acquire_immediate() {
            // Send connection to waiter
            let _ = waiter.send(Ok(conn));
        } else {
            // No connections available, stop signaling
            wait_queue.push_front(waiter);
            break;
        }
    }
}
```

---

### 10. MetricsCollector (Observability)

**Purpose**: Collect and export pool metrics

**Metrics**:

**Gauges** (current values):
- `pool_active_connections` - Currently in-use connections
- `pool_idle_connections` - Currently idle connections
- `pool_wait_queue_length` - Tasks waiting for connections
- `pool_creating_connections` - Currently being created

**Counters** (cumulative values):
- `pool_connections_created_total` - Total connections created
- `pool_connections_destroyed_total` - Total connections destroyed
- `pool_acquires_total` - Total acquire calls
- `pool_releases_total` - Total release calls
- `pool_health_check_failures_total` - Failed health checks
- `pool_timeouts_total` - Acquire timeouts

**Histograms** (distributions):
- `pool_acquire_duration_seconds` - Time to acquire connection
- `pool_connection_lifetime_seconds` - Connection lifetime
- `pool_idle_time_seconds` - How long connections are idle

**Export Formats**:
- Prometheus (text-based)
- StatsD (UDP)
- OpenTelemetry (if feature enabled)

---

### 11. ShutdownCoordinator (Graceful Shutdown)

**Purpose**: Coordinate clean pool shutdown

**Fields**:
```rust
pub struct ShutdownCoordinator {
    // Shutdown signal
    signal: Arc<AtomicBool>,

    // Wait for in-flight connections
    in_flight: Arc<AtomicUsize>,

    // Shutdown timeout
    timeout: Duration,
}
```

**Shutdown Flow**:
```rust
async fn shutdown() -> Result<()> {
    // 1. Set shutdown flag (stop accepting new acquires)
    signal.store(true, Ordering::SeqCst);

    // 2. Wait for in-flight connections (with timeout)
    tokio::time::timeout(timeout, async {
        while in_flight.load(Ordering::Acquire) > 0 {
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
    }).await?;

    // 3. Close all idle connections
    for conn in idle_connections.drain() {
        conn.close().await;
    }

    // 4. Cancel wait queue (return errors to waiters)
    for waiter in wait_queue.drain() {
        let _ = waiter.send(Err(Error::PoolShuttingDown));
    }

    Ok(())
}
```

**PooledConnection Drop**:
```rust
impl<T> Drop for PooledConnection<T> {
    fn drop(&mut self) {
        // If pool is shutting down, close connection
        if self.pool.is_shutting_down() {
            // Don't return to pool, close it
            return;
        }

        // Otherwise, return to pool
        self.pool.release(self.conn);
    }
}
```

---

## Component Diagrams

### Pool Creation Flow

```
Pool::builder()
    │
    ├─> Build PoolConfig with defaults
    │   - Set default values
    │   - Validate configuration
    │
    ├─> Create ConnectionFactory from closure
    │   - Box the closure
    │   - Wrap in Arc
    │
    ├─> Create ConnectionStore
    │   - Initialize idle queue
    │   - Initialize atomics
    │
    ├─> Create HealthChecker
    │   - Set health check function
    │   - Start background health check task
    │
    ├─> Create AdaptiveSizer
    │   - Initialize usage tracking
    │   - Start background sizing task
    │
    ├─> Create MetricsCollector
    │   - Initialize metrics registry
    │
    ├─> Pre-create initial connections
    │   - Call factory function
    │   - Add to idle pool
    │
    └─> Return Pool<T>
```

### Connection Acquisition Flow

```
pool.get().await
    │
    ├─> Check if pool is shutting down
    │   └─> If yes, return Error::PoolShuttingDown
    │
    ├─> Try to acquire from idle pool
    │   ├─> Pop from idle queue
    │   ├─> If found:
    │   │   ├─> Health check (if due)
    │   │   │   ├─> If healthy → return connection
    │   │   │   └─> If unhealthy → replace connection
    │   │   └─> Increment active count
    │   └─> If not found, continue
    │
    ├─> Check if we can create new connection
    │   ├─> If total < max_size:
    │   │   ├─> Increment creating count
    │   │   ├─> Call factory function
    │   │   ├─> If success:
    │   │   │   ├─> Increment active count
    │   │   │   ├─> Record metrics
    │   │   │   └─> Return connection
    │   │   └─> If failure:
    │   │   │   ├─> Decrement creating count
    │   │   │   └─> Return error
    │   └─> If total >= max_size, continue
    │
    ├─> Wait in wait queue
    │   ├─> Create oneshot channel
    │   ├─> Add to wait queue
    │   ├─> Wait for signal (with timeout)
    │   ├─> If timeout → return Error::AcquireTimeout
    │   ├─> If signal → return connection or error
    │   └─> If pool shutdown → return Error::PoolShuttingDown
    │
    └─> Return PooledConnection<T>
```

### Connection Release Flow

```
drop(pooled_connection)
    │
    ├─> Decrement active count
    │
    ├─> Check if pool is shutting down
    │   ├─> If yes → close connection
    │   └─> If no → return to pool
    │
    ├─> Return to pool
    │   ├─> Update last_used timestamp
    │   ├─> Push to idle queue
    │   ├─> Record metrics
    │   │
    │   ├─> Signal wait queue
    │   │   └─> Wake up next waiter
    │   │
    │   └─> Trigger adaptive sizing
    │       ├─> Should we grow?
    │       ├─> Should we shrink?
    │       └─> Evaluate pool size
    │
    └─> Done
```

### Health Check Flow

```
Background task (every health_check_interval)
    │
    ├─> Lock idle queue
    │
    ├─> For each idle connection:
    │   ├─> Check if health check is due
    │   │   ├─> If not due → skip
    │   │   └─> If due → continue
    │   │
    │   ├─> Run health check function
    │   │   ├─> With timeout
    │   │   ├─> If healthy:
    │   │   │   ├─> Update last_health_check timestamp
    │   │   │   └─> Keep in pool
    │   │   └─> If unhealthy:
    │   │   │   ├─> Close connection
    │   │   │   ├─> Remove from pool
    │   │   │   ├─> Record health_check_failure metric
    │   │   │   └─> Create replacement (if below min_size)
    │   │
    │   └─> Continue to next connection
    │
    ├─> Ensure min_size maintained
    │   └─> Create connections if needed
    │
    └─> Sleep until next interval
```

---

## Data Structures

### Connection Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                      Connection States                          │
└─────────────────────────────────────────────────────────────────┘

    Creating ──> Idle ──> Active ──> Idle ──> Destroyed
        │           │          │
        │           │          └─> Active (re-acquired)
        │           │
        │           └─> Health Check ──> Idle (healthy) or Destroyed (unhealthy)
        │
        └─> Failed ──> Destroyed

State Transitions:
- Creating → Idle: Connection created successfully
- Idle → Active: Connection acquired by application
- Active → Idle: Connection released back to pool
- Idle → Health Check: Periodic health check or before acquire
- Health Check → Idle: Connection is healthy
- Health Check → Destroyed: Connection is unhealthy
- Any → Destroyed: Connection closed/destroyed
```

### Idle Connection Metadata

```rust
pub struct IdleConnection<T> {
    // The actual connection
    pub connection: T,

    // When this connection was created
    pub created_at: Instant,

    // Last time this connection was used (acquired)
    pub last_used: Instant,

    // Last time health check ran
    pub last_health_check: Option<Instant>,

    // Number of times this connection has been acquired
    pub acquire_count: u64,

    // Connection ID (for tracking)
    pub id: usize,
}
```

### Pool Status

```rust
pub struct PoolStatus {
    // Current counts
    pub active_connections: usize,
    pub idle_connections: usize,
    pub creating_connections: usize,
    pub total_connections: usize,

    // Wait queue
    pub wait_queue_length: usize,

    // Pool configuration
    pub min_size: usize,
    pub max_size: usize,

    // Pool state
    pub is_shutting_down: bool,

    // Metrics (recent samples)
    pub avg_acquire_time: Duration,
    pub avg_idle_time: Duration,
    pub health_check_failures_last_hour: u64,
}
```

---

## Performance Considerations

### 1. Lock-Free Operations

**Atomic counters** for hot path:
```rust
pub struct ConnectionStore<T> {
    // Lock-free reads for performance
    active: Arc<AtomicUsize>,
    creating: Arc<AtomicUsize>,
    total_created: Arc<AtomicU64>,

    // Mutex only for idle queue (low contention)
    idle: Arc<Mutex<VecDeque<IdleConnection<T>>>>,
}
```

**Why**:
- Reading `active` count is on hot path (every acquire)
- Atomic operations are faster than mutex
- Idle queue has low contention (only on acquire/release)

---

### 2. Zero-Allocation in Hot Path

**Avoid allocations** during acquire:
```rust
// Pre-allocate with capacity
let mut idle = VecDeque::with_capacity(max_size);

// Reuse connections (don't destroy unless necessary)
// - Don't close idle connections
// - Reuse after health check
// - Only close on max_lifetime or idle_timeout
```

**Why**:
- Allocations are expensive
- Reduce memory fragmentation
- Better cache locality

---

### 3. Lazy Health Checks

**Don't health check every time**:
```rust
// Only health check if interval has passed
if last_health_check.map_or(true, |t| t.elapsed() > interval) {
    // Run health check
} else {
    // Skip health check (fast path)
}
```

**Why**:
- Health checks can be expensive (network round-trip)
- Most connections are healthy
- Skip unless due

---

### 4. Connection Pre-Warming

**Pre-create connections at startup**:
```rust
Pool::builder()
    .initial_size(10)  // Create 10 connections at startup
    .build(factory)
```

**Why**:
- First requests are fast (no connection creation)
- Amortize connection creation cost
- Better cold-start performance

---

### 5. Adaptive Sizing

**Grow pool before it's exhausted**:
```rust
if active_connections + wait_queue_length > max_size * 0.8 {
    // Grow pool now, before it's fully exhausted
    create_connections(10).await;
}
```

**Why**:
- Avoid wait queue buildup
- Reduce acquire latency
- Better performance under load

---

## Security Considerations

### 1. Connection String Redaction

**Don't log connection strings**:
```rust
// Use privox for redaction
let safe_string = redactor.redact(&connection_string)?;
```

### 2. Health Check Privilege Escalation

**Health check shouldn't have elevated privileges**:
```rust
// Use read-only user for health checks
health_check(|conn| async move {
    conn.query("SELECT 1").await  // Read-only query
});
```

### 3. Connection Timeout Enforcement

**Always enforce timeouts**:
```rust
.acquire_timeout(Duration::from_secs(30))
.health_check_timeout(Duration::from_secs(5))
```

**Why**:
- Prevent indefinite hangs
- Protect against slow/laggy connections
- DoS protection

---

## Testing Architecture

### Unit Tests

- Test pool state transitions
- Test health check logic
- Test adaptive sizing algorithm
- Test metrics collection
- Test error handling

### Integration Tests

- Test with real database (Postgres, MySQL, Redis)
- Test concurrent access
- Test graceful shutdown
- Test pool resizing
- Test health checks

### Load Tests

- Test under high concurrency
- Test connection exhaustion
- Test wait queue behavior
- Test pool performance
- Benchmark against alternatives

### Property-Based Tests

- Test pool invariants (e.g., total <= max_size)
- Test connection lifecycle (no leaks)
- Test health check properties
- Test adaptive sizing properties

---

## Error Handling

### Error Types

```rust
pub enum Error {
    // Connection errors
    ConnectionFailed(String),
    HealthCheckFailed(String),

    // Pool errors
    PoolExhausted,
    AcquireTimeout,
    PoolShuttingDown,

    // Configuration errors
    InvalidConfig(String),

    // Timeout errors
    Timeout(Duration),

    // Generic error
    Internal(String),
}
```

### Error Recovery

**Connection creation failed**:
- Return error to caller
- Decrement creating count
- Log metric

**Health check failed**:
- Replace connection (if idle)
- Create new connection
- Log metric

**Acquire timeout**:
- Return error to caller
- Remove from wait queue
- Log metric

---

## Next Steps

Now that we have the architecture designed, we'll move to:

**Next**: [API Design](03-api.md) - Define the public API

---

**Document Status**: ✅ Complete
**Word Count**: ~4,000 words
**Architecture Complete**: Yes
