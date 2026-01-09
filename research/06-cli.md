# Pool-RS CLI Design

## Command-Line Tool for Pool-RS

The pool-rs CLI provides operations and monitoring for connection pools without code changes.

---

## Installation

```bash
cargo install pool-cli
```

---

## Command Structure

```
pool <COMMAND> [OPTIONS]

Commands:
  status     Show pool status
  test       Test pool health
  drain      Close idle connections
  benchmark  Benchmark pool performance
  help       Show this help message

Options:
  -v, --verbose     Verbose output
  -q, --quiet       Quiet output
  -h, --help        Show help
  -V, --version     Show version
```

---

## Commands

### pool status

Show pool status and statistics.

#### Usage

```bash
pool status [OPTIONS] <POOL_NAME>
```

#### Options

- `-j, --json` - Output in JSON format
- `-w, --watch` - Watch mode (update every second)

#### Examples

**Basic status**:
```bash
$ pool status database-pool

Pool: database-pool
Status: Running
Configuration:
  Min size: 5
  Max size: 100
  Current size: 25

Connections:
  Active: 20
  Idle: 5
  Total: 25

Wait Queue:
  Length: 0

Metrics (last 5 minutes):
  Acquires: 1,234
  Releases: 1,234
  Acquire time (avg): 2.5ms
  Health check failures: 0
```

**JSON output**:
```bash
$ pool status database-pool --json

{
  "pool_name": "database-pool",
  "status": "running",
  "config": {
    "min_size": 5,
    "max_size": 100
  },
  "connections": {
    "active": 20,
    "idle": 5,
    "total": 25
  },
  "wait_queue": {
    "length": 0
  },
  "metrics": {
    "acquires_total": 1234,
    "releases_total": 1234,
    "avg_acquire_time_ms": 2.5
  }
}
```

**Watch mode**:
```bash
$ pool status database-pool --watch

Pool: database-pool (updating every 1s)
Active: 20, Idle: 5, Waiting: 0
^C
```

---

### pool test

Test pool health by checking all connections.

#### Usage

```bash
pool test [OPTIONS] <POOL_NAME>
```

#### Options

- `-v, --verbose` - Show detailed output
- `-f, --fix` - Replace unhealthy connections

#### Examples

**Basic test**:
```bash
$ pool test database-pool

Testing pool: database-pool
Connections tested: 25
Healthy: 25
Unhealthy: 0
Replaced: 0

Status: ✓ All connections healthy
```

**With unhealthy connections**:
```bash
$ pool test database-pool

Testing pool: database-pool
Connections tested: 25
Healthy: 23
Unhealthy: 2
Replaced: 2

Status: ⚠ Replaced 2 unhealthy connections
```

**Verbose output**:
```bash
$ pool test database-pool --verbose

Testing pool: database-pool
Connection 1: ✓ Healthy (1.2ms)
Connection 2: ✓ Healthy (0.8ms)
Connection 3: ✗ Unhealthy - Connection timeout
Connection 4: ✓ Healthy (1.1ms)
...
Connection 25: ✓ Healthy (0.9ms)

Summary: 24/25 healthy, 1 replaced
```

---

### pool drain

Close idle connections (useful for maintenance or freeing resources).

#### Usage

```bash
pool drain [OPTIONS] <POOL_NAME>
```

#### Options

- `-k, --keep <N>` - Keep N idle connections (default: min_size)
- `-f, --force` - Close all idle connections (ignore min_size)

#### Examples

**Drain to min_size**:
```bash
$ pool drain database-pool

Draining pool: database-pool
Idle connections before: 20
Idle connections after: 5
Closed: 15 connections
```

**Keep specific number**:
```bash
$ pool drain database-pool --keep 2

Draining pool: database-pool
Idle connections before: 20
Idle connections after: 2
Closed: 18 connections
```

**Force close all**:
```bash
$ pool drain database-pool --force

Draining pool: database-pool
Idle connections before: 20
Idle connections after: 0
Closed: 20 connections

Warning: Pool will create new connections as needed
```

---

### pool benchmark

Benchmark pool performance under load.

#### Usage

```bash
pool benchmark [OPTIONS] <POOL_NAME>
```

#### Options

- `-c, --connections <N>` - Number of connections to acquire (default: 1000)
- `-p, --parallel <N>` - Parallel tasks (default: 10)
- `-d, --duration <SECONDS>` - Benchmark duration (default: 10)
- `-w, --warmup <SECONDS>` - Warmup duration (default: 1)

#### Examples

**Basic benchmark**:
```bash
$ pool benchmark database-pool

Benchmarking pool: database-pool
Warmup: 1s
Duration: 10s
Connections: 1000
Parallel: 10

Results:
  Total acquires: 1000
  Successful: 1000
  Failed: 0
  Timeouts: 0

  Throughput: 100 acquires/sec
  Avg acquire time: 2.5ms
  P50: 2.1ms
  P95: 4.2ms
  P99: 8.1ms

  Max concurrent: 10
  Avg concurrent: 8.5
```

**Custom benchmark**:
```bash
$ pool benchmark database-pool \
  --connections 10000 \
  --parallel 50 \
  --duration 30

Benchmarking pool: database-pool
...
Results:
  Throughput: 333 acquires/sec
  Avg acquire time: 3.2ms
  P99: 12.4ms
```

---

## Configuration File

The CLI can read configuration from a file.

### File Location

- Linux/macOS: `~/.pool/config.toml`
- Windows: `%USERPROFILE%\.pool\config.toml`

### Configuration Format

```toml
# Default pool settings
[defaults]
min_size = 5
max_size = 100
acquire_timeout = "30s"
health_check_interval = "30s"

# Pool-specific settings
[pool.database-pool]
min_size = 10
max_size = 200
acquire_timeout = "10s"

[pool.redis-pool]
min_size = 2
max_size = 20
```

---

## Environment Variables

Configuration via environment variables:

```bash
export POOL_MIN_SIZE=5
export POOL_MAX_SIZE=100
export POOL_ACQUIRE_TIMEOUT=30s
export POOL_HEALTH_CHECK_INTERVAL=30s
```

---

## Advanced Features

### Remote Pool Management

Connect to remote pools via API server:

```bash
pool --api-url http://localhost:8080 status database-pool
```

### Batch Operations

Operate on multiple pools:

```bash
pool status --all

Pool: database-pool
  Status: Running
  Connections: 20 active, 5 idle

Pool: redis-pool
  Status: Running
  Connections: 5 active, 2 idle

Pool: http-pool
  Status: Running
  Connections: 10 active, 3 idle
```

### Integration with Monitoring

Export to monitoring systems:

```bash
# Prometheus
pool status database-pool --format prometheus > /tmp/metrics.prom

# StatsD
pool status database-pool --format statsd | nc -U /tmp/statsd.sock
```

---

## Examples

### Scenario 1: Debug Pool Exhaustion

```bash
# Check pool status
$ pool status database-pool

Pool: database-pool
Connections: 100 active, 0 idle
Wait queue: 25 tasks waiting

# Pool is exhausted! Increase max_size
# (via config file or application restart)
```

### Scenario 2: Test After Database Restart

```bash
# Database was restarted, test pool health
$ pool test database-pool

Testing pool: database-pool
Connections tested: 25
Healthy: 0
Unhealthy: 25
Replaced: 25

Status: ✓ Replaced all unhealthy connections
```

### Scenario 3: Performance Tuning

```bash
# Benchmark current configuration
$ pool benchmark database-pool

Throughput: 100 acquires/sec
P99: 8.1ms

# Try increasing pool size and re-benchmark
# (modify config, restart application)
$ pool benchmark database-pool

Throughput: 200 acquires/sec
P99: 3.2ms
```

---

## Summary

The pool-rs CLI provides:
- ✅ Pool status monitoring
- ✅ Health testing
- ✅ Connection draining
- ✅ Performance benchmarking
- ✅ Remote management
- ✅ Monitoring integration

**Next**: [Use Cases & Integration](07-use-cases.md) - Real-world scenarios

---

**Document Status**: ✅ Complete
**Word Count**: ~2,000 words
