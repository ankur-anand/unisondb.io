---
title: "Configuration"
weight: 3
---

# Configuration Guide

UnisonDB uses TOML for configuration. This guide covers all available configuration options for both server and relayer modes.

## Table of Contents

- [Server Mode](#server-mode)
- [Relayer Mode](#relayer-mode)
- [Configuration Reference](#configuration-reference)

## Server Mode

Server mode runs UnisonDB as a primary instance that accepts writes and serves reads. Here's a complete example:

```toml
## Port of the http server
http_port = 4000
listen_ip = "0.0.0.0"

## grpc config for replication
[grpc_config]
listen_ip = "0.0.0.0"
port = 4001
# SSL/TLS certificate paths for gRPC server
cert_path = "../../certs/server.crt"
key_path = "../../certs/server.key"
ca_path = "../../certs/ca.crt"
# Allow insecure connections (no TLS) - ONLY for development!
allow_insecure = false

# StorageConfig stores all tunable parameters.
[storage_config]
base_dir = "/tmp/unisondb/server"   # Base directory for storage
namespaces = ["default", "tenant_1", "tenant_2", "tenant_3", "tenant_4"]
bytes_per_sync = "1MB"
segment_size = "16MB"
arena_size = "4MB"
wal_fsync_interval = "1s"

## WAL cleanup configuration
[storage_config.wal_cleanup_config]
enabled = false
interval = "5m"
max_age = "1h"
min_segments = 5
max_segments = 10

## Write notify config - coalesces notifications from WAL writers to readers
[write_notify_config]
enabled = true
max_delay = "20ms"

## ZeroMQ notifier configuration (per-namespace)
## Publishes change notifications for local application consumption
[notifier_config.default]
bind_port = 5555
high_water_mark = 1000
linger_time = 1000

[notifier_config.tenant_1]
bind_port = 5556
high_water_mark = 1000
linger_time = 1000

[pprof_config]
enabled = true
port = 6060

[log_config]
log_level = "info"
disable_timestamp = false

## This is for grpc logging only - controls sampling percentages per level
[log_config.min_level_percents]
debug = 100.0
info  = 50.0
warn  = 100.0
error = 100.0

## Fuzzer configuration (for testing)
[fuzz_config]
ops_per_namespace = 400
workers_per_namespace = 50
local_relayer_count = 1000
startup_delay = "10s"
enable_read_ops = false
```

## Relayer Mode

Relayer mode runs UnisonDB as a replica that streams changes from one or more upstream servers. This provides read scalability and data locality.

```toml
## Port of the http server
http_port = 6000

[grpc_config]
port = 6001
cert_path = "../../certs/server.crt"
key_path = "../../certs/server.key"
ca_path = "../../certs/ca.crt"

[storage_config]
base_dir = "/tmp/unisondb/relayer"
namespaces = ["default", "tenant_1", "tenant_2"]
bytes_per_sync = "1MB"
## IMPORTANT: segment_size must match upstream server!
segment_size = "16MB"
arena_size = "4MB"

## Relayer configuration - can have multiple upstreams
[relayer_config]

[relayer_config.relayer1]
namespaces = ["default", "tenant_1", "tenant_2"]
cert_path = "../../certs/client.crt"
key_path = "../../certs/client.key"
ca_path = "../../certs/ca.crt"
upstream_address = "localhost:4001"
segment_lag_threshold = 100
allow_insecure = false
# Optional: custom gRPC service config JSON
grpc_service_config = ""

## Optional: Add more relayers for different upstream sources
[relayer_config.relayer2]
namespaces = ["tenant_3"]
cert_path = "../../certs/client2.crt"
key_path = "../../certs/client2.key"
ca_path = "../../certs/ca.crt"
upstream_address = "remote-server:4001"
segment_lag_threshold = 100

[log_config]
log_level = "info"

[log_config.min_level_percents]
debug = 0.01
info = 1.0
warn = 1.0
error = 1.0
```

---

## Configuration Reference

### Server Configuration

#### HTTP Server

```toml
http_port = 4000
listen_ip = "0.0.0.0"
```

##### `http_port`
- **Type**: Integer
- **Default**: `4000`
- **Description**: Port for the HTTP API server

##### `listen_ip`
- **Type**: String
- **Default**: `"0.0.0.0"`
- **Description**: IP address to bind HTTP server to
- **Note**: Use `"127.0.0.1"` for localhost-only access

#### gRPC Configuration

```toml
[grpc_config]
listen_ip = "0.0.0.0"
port = 4001
cert_path = "/path/to/server.crt"
key_path = "/path/to/server.key"
ca_path = "/path/to/ca.crt"
allow_insecure = false
```

##### `listen_ip`
- **Type**: String
- **Default**: `"0.0.0.0"`
- **Description**: IP address to bind gRPC server to

##### `port`
- **Type**: Integer
- **Default**: `4001`
- **Description**: Port for the gRPC server (used for replication)

##### `cert_path`
- **Type**: String
- **Default**: `""`
- **Description**: Path to TLS certificate file (PEM format)
- **Required**: Yes (unless `allow_insecure = true`)

##### `key_path`
- **Type**: String
- **Default**: `""`
- **Description**: Path to TLS private key file (PEM format)
- **Required**: Yes (unless `allow_insecure = true`)

##### `ca_path`
- **Type**: String
- **Default**: `""`
- **Description**: Path to CA certificate file for mTLS
- **Required**: Yes (unless `allow_insecure = true`)

##### `allow_insecure`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Allow insecure connections without TLS
- **Warning**: ONLY use in development! Always enable TLS in production

---

### Storage Configuration

```toml
[storage_config]
base_dir = "./data"
namespaces = ["default", "app"]
bytes_per_sync = "1MB"
segment_size = "16MB"
arena_size = "4MB"
wal_fsync_interval = "1s"
disable_entry_type_check = false
```

#### `base_dir`
- **Type**: String
- **Default**: `"./data"`
- **Description**: Base directory for all data files (WAL segments, LMDB)
- **Note**: Must have write permissions

#### `namespaces`
- **Type**: Array of Strings
- **Default**: `["default"]`
- **Description**: List of namespaces to create on startup
- **Example**: `["default", "users", "metrics", "logs"]`
- **Note**: Each namespace is isolated with separate WAL and storage

#### `bytes_per_sync`
- **Type**: String (with unit)
- **Default**: `"1MB"`
- **Valid Units**: `KB`, `MB`, `GB`
- **Description**: Number of bytes to write before forcing fsync

#### `segment_size`
- **Type**: String (with unit)
- **Default**: `"16MB"`
- **Valid Units**: `KB`, `MB`, `GB`
- **Range**: `1MB` to `1GB`
- **Description**: Size of each WAL segment file
- **Important**: Must match across server and relayer!

#### `arena_size`
- **Type**: String (with unit)
- **Default**: `"4MB"`
- **Valid Units**: `KB`, `MB`, `GB`
- **Range**: `1MB` to `64MB`
- **Description**: Size of the write buffer (memtable)
- **Performance**: Larger = fewer flushes, more memory usage

#### `wal_fsync_interval`
- **Type**: String (duration)
- **Default**: `"1s"`
- **Valid Units**: `ms`, `s`, `m`
- **Description**: Interval for periodic WAL fsync
- **Trade-off**: Lower = better durability, higher = better performance

#### WAL Cleanup Configuration

```toml
[storage_config.wal_cleanup_config]
enabled = false
interval = "5m"
max_age = "1h"
min_segments = 5
max_segments = 10
```

##### `enabled`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Enable automatic WAL segment cleanup

##### `interval`
- **Type**: String (duration)
- **Default**: `"5m"`
- **Description**: How often to run cleanup

##### `max_age`
- **Type**: String (duration)
- **Default**: `"1h"`
- **Description**: Maximum age of segments before cleanup

##### `min_segments`
- **Type**: Integer
- **Default**: `5`
- **Description**: Minimum number of segments to keep

##### `max_segments`
- **Type**: Integer
- **Default**: `10`
- **Description**: Trigger cleanup when this many segments exist

---

### Write Notification Configuration

Write notifications coalesce updates from WAL writers to readers, reducing notification overhead.

```toml
[write_notify_config]
enabled = true
max_delay = "20ms"
```

#### `enabled`
- **Type**: Boolean
- **Default**: `true`
- **Description**: Enable write notification coalescing

#### `max_delay`
- **Type**: String (duration)
- **Default**: `"20ms"`
- **Valid Units**: `ms`, `s`
- **Description**: Maximum delay before notifying readers
- **Trade-off**: Higher = better batching, higher latency for reads

---

### ZeroMQ Notifier Configuration

UnisonDB can publish change notifications via ZeroMQ PUB/SUB sockets. This allows **local applications** to subscribe to real-time change notifications for specific namespaces.

**Use Case**: Applications running on the same machine can subscribe to a namespace's ZeroMQ socket and receive notifications whenever data changes, enabling reactive architectures.

```toml
## Each namespace can have its own ZeroMQ notifier
[notifier_config.default]
bind_port = 5555
high_water_mark = 1000
linger_time = 1000

[notifier_config.tenant_1]
bind_port = 5556
high_water_mark = 2000
linger_time = 500
```

#### `bind_port`
- **Type**: Integer
- **Required**: Yes
- **Description**: Port to bind ZeroMQ PUB socket to
- **Format**: Applications subscribe to `tcp://localhost:{bind_port}`

#### `high_water_mark`
- **Type**: Integer
- **Default**: `1000`
- **Description**: Maximum number of queued messages before blocking
- **Note**: Higher values use more memory but reduce message loss

#### `linger_time`
- **Type**: Integer (milliseconds)
- **Default**: `1000`
- **Description**: How long to wait for pending messages on shutdown
- **Range**: `0` to `5000` ms

**Example Application Subscription**:
```python
import zmq

context = zmq.Context()
socket = context.socket(zmq.SUB)
socket.connect("tcp://localhost:5555")  # Connect to default namespace
socket.setsockopt(zmq.SUBSCRIBE, b"")   # Subscribe to all messages

while True:
    message = socket.recv()
    print(f"Received change notification: {message}")
```

---

### Relayer Configuration

Relayer configuration allows a UnisonDB instance to stream WAL changes from one or more upstream servers. This is useful for:
- **Read scaling**: Run multiple read replicas
- **Data locality**: Keep data close to consumers in different regions
- **Backup**: Maintain hot standbys

```toml
[relayer_config]

[relayer_config.relayer1]
namespaces = ["default", "tenant_1"]
cert_path = "../../certs/client.crt"
key_path = "../../certs/client.key"
ca_path = "../../certs/ca.crt"
upstream_address = "primary-server:4001"
segment_lag_threshold = 100
allow_insecure = false
grpc_service_config = ""
```

#### Map Key (e.g., `relayer1`)
- **Type**: String
- **Description**: Unique identifier for this relayer connection
- **Note**: Multiple relayers can be configured with different keys

#### `namespaces`
- **Type**: Array of Strings
- **Required**: Yes
- **Description**: List of namespaces to replicate from this upstream
- **Note**: Namespaces must exist on both upstream and local instance

#### `cert_path`
- **Type**: String
- **Description**: Path to client TLS certificate for mTLS
- **Required**: Yes (unless `allow_insecure = true`)

#### `key_path`
- **Type**: String
- **Description**: Path to client private key for mTLS
- **Required**: Yes (unless `allow_insecure = true`)

#### `ca_path`
- **Type**: String
- **Description**: Path to CA certificate to verify upstream server
- **Required**: Yes (unless `allow_insecure = true`)

#### `upstream_address`
- **Type**: String
- **Required**: Yes
- **Description**: Address of upstream gRPC server
- **Format**: `host:port` (e.g., `"localhost:4001"`, `"10.0.1.5:4001"`)

#### `segment_lag_threshold`
- **Type**: Integer
- **Default**: `100`
- **Description**: Maximum segment lag before logging warnings
- **Note**: Helps monitor replication health

#### `allow_insecure`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Allow insecure connection to upstream (no TLS)
- **Warning**: Only for development!

#### `grpc_service_config`
- **Type**: String (JSON)
- **Default**: `""` (uses built-in defaults)
- **Description**: Custom gRPC service configuration JSON
- **Advanced**: See gRPC documentation for format

---

### Logging Configuration

```toml
[log_config]
log_level = "info"
disable_timestamp = false

[log_config.min_level_percents]
debug = 100.0
info  = 50.0
warn  = 100.0
error = 100.0
```

#### `log_level`
- **Type**: String
- **Valid Values**: `"debug"`, `"info"`, `"warn"`, `"error"`
- **Default**: `"info"`
- **Description**: Minimum log level to output

#### `disable_timestamp`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Disable timestamps in log output
- **Use Case**: When running under systemd/journal (timestamps added automatically)

#### `min_level_percents`
- **Type**: Map of String to Float
- **Description**: Sampling percentages for gRPC logging per level
- **Range**: `0.0` to `100.0`
- **Purpose**: Reduce log volume in high-traffic scenarios
- **Example**: `info = 1.0` means sample 1% of info logs

**Log Levels**:
- `debug`: `100.0` = log all debug messages
- `info`: `50.0` = log 50% of info messages (randomly sampled)
- `warn`: `100.0` = log all warnings
- `error`: `100.0` = log all errors

---

### PProf Configuration

```toml
[pprof_config]
enabled = true
port = 6060
```

#### `enabled`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Enable pprof HTTP server for profiling

#### `port`
- **Type**: Integer
- **Default**: `6060`
- **Description**: Port for pprof HTTP server
- **Access**: `http://localhost:6060/debug/pprof/`

**Available Profiles**:
- `/debug/pprof/heap` - Memory allocation
- `/debug/pprof/goroutine` - Goroutine stack traces
- `/debug/pprof/profile` - CPU profile
- `/debug/pprof/trace` - Execution trace

---

### Fuzzer Configuration

Built-in fuzzer for testing and stress testing UnisonDB.

```toml
[fuzz_config]
ops_per_namespace = 400
workers_per_namespace = 50
local_relayer_count = 1000
startup_delay = "10s"
enable_read_ops = false
```

#### `ops_per_namespace`
- **Type**: Integer
- **Default**: `400`
- **Description**: Number of operations to perform per namespace

#### `workers_per_namespace`
- **Type**: Integer
- **Default**: `50`
- **Description**: Number of concurrent workers per namespace

#### `local_relayer_count`
- **Type**: Integer
- **Default**: `1000`
- **Description**: Number of local relayer goroutines to simulate

#### `startup_delay`
- **Type**: String (duration)
- **Default**: `"10s"`
- **Description**: Delay before starting fuzzer
- **Purpose**: Allow infrastructure to fully initialize

#### `enable_read_ops`
- **Type**: Boolean
- **Default**: `false`
- **Description**: Include read operations in fuzzing
- **Note**: Generates mixed read/write workload when enabled

---
