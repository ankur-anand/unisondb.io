---
title: "Getting Started with UnisonDB"
linkTitle: "Getting Started"
weight: 1
bookCollapseSection: false
images: ["/images/getting_started.jpg"]
description: "Step-by-step guide to install, configure, and run UnisonDB in under 5 minutes. Learn how to set up replicator and relayer modes for real-time data replication and edge deployment."
keywords: [
  "UnisonDB quick start",
  "install UnisonDB",
  "UnisonDB tutorial",
  "edge computing database setup",
  "real-time replication database",
  "WAL-based database",
  "distributed database getting started",
  "UnisonDB replicator mode",
  "UnisonDB relayer mode",
  "UnisonDB configuration guide"
]
---


# Getting Started with UnisonDB

<img src="/images/getting_started.svg" alt="Getting Started with UnisonDB" style="max-width: 100%; height: auto;" />

This guide will walk you through installing UnisonDB, configuring it, and running it in both Replicator and Relayer modes.

```
          ┌────────────────┐
          │  Replicator    │
          │  (Primary)     │
          │  Writes → WAL  │
          │  Streams gRPC  │
          └──────┬─────────┘
                 │
        WAL Stream (gRPC)
                 │
   ┌─────────────┴──────────────┐
   ↓                            ↓
┌───────────┐              ┌───────────┐
│ Relayer 1 │              │ Relayer 2 │
│ (Replica) │              │ (Replica) │
│ Local DB  │              │ Local DB  │
│ Watch API │              │ Watch API │
└───────────┘              └───────────┘
```

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Running in Server Mode](#running-in-server-mode)
- [Running in Relayer Mode](#running-in-relayer-mode)

## Prerequisites

### System Requirements

- **Operating System**: Linux or macOS
- **Go**: Version 1.24 or higher

## Installation

### Building from Source

UnisonDB requires CGO to be enabled for LMDB bindings.

```bash
# Clone the repository
git clone https://github.com/ankur-anand/unisondb.git
cd unisondb

# Enable CGO (required for LMDB)
export CGO_ENABLED=1

# Build the binary
go build -o unisondb ./cmd/unisondb

# Verify installation
./unisondb --help
```

Expected output:
```
  _   _          _                     ___    ___
 | | | |  _ _   (_)  ___  ___   _ _   |   \  | _ )
 | |_| | | ' \  | | (_-< / _ \ | ' \  | |) | | _ \
  \___/  |_||_| |_| /__/ \___/ |_||_| |___/  |___/

     Database + Message Bus. Built for Edge.
               https://unisondb.io

NAME:
   unisondb - Run UnisonDB

USAGE:
   unisondb [global options] command [command options]

COMMANDS:
   replicator  Run in replicator mode
   relayer     Run in relayer mode
   fuzzer      This is a testing-only feature (disabled in production builds)
   help, h     Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --config value, -c value  Path to TOML config file (default: "./config.toml") [$UNISON_CONFIG]
   --env value, -e value     Environment: dev, staging, prod (default: "dev") [$UNISON_ENV]
   --grpc, -G                Enable gRPC server in Relayer Mode (default: false) [$UNISON_GRPC_ENABLED]
   --help, -h                show help
```

### Building with Fuzzer Support (Optional)

For testing and development, you can build with fuzzer support:

```bash
CGO_ENABLED=1 go build -tags fuzz -o unisondb ./cmd/unisondb
```

**Note**: When built with `-tags fuzz`:
- `fuzzer` command is available
- `replicator` command is **disabled** for safety

To run fuzzer mode:
```bash
./unisondb --config config.toml fuzzer
```

### Installation to System Path (Optional)

```bash
# Move to system path
sudo mv unisondb /usr/local/bin/

# Verify
unisondb --help
```

## Running in Replicator Mode

Replicator Mode runs UnisonDB as a **primary instance** that accepts writes and serves reads.

### 1. Generate TLS Certificates (Recommended)

For production or multi-node setups, generate TLS certificates for gRPC:

**Using OpenSSL**:
```bash
mkdir -p certs && cd certs

# Generate CA
openssl genrsa -out ca.key 4096
openssl req -new -x509 -key ca.key -sha256 -subj "/CN=UnisonDB CA" -days 365 -out ca.crt

# Generate server certificate
openssl genrsa -out server.key 4096
openssl req -new -key server.key -out server.csr -subj "/CN=localhost"
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -sha256

# Generate client certificate
openssl genrsa -out client.key 4096
openssl req -new -key client.key -out client.csr -subj "/CN=client"
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 365 -sha256

cd ..
```

### 2. Create Server Configuration

Create a `server.toml` configuration file:

```toml
## HTTP API port
http_port = 4000
listen_ip = "0.0.0.0"

## gRPC configuration (for replication)
[grpc_config]
listen_ip = "0.0.0.0"
port = 4001
cert_path = "./certs/server.crt"
key_path = "./certs/server.key"
ca_path = "./certs/ca.crt"
# For development only - allows insecure connections
allow_insecure = false

## Storage configuration
[storage_config]
base_dir = "./data/server"
namespaces = ["default", "users", "products"]
bytes_per_sync = "1MB"
segment_size = "16MB"
arena_size = "4MB"
wal_fsync_interval = "1s"

## WAL cleanup (prevents disk exhaustion)
[storage_config.wal_cleanup_config]
enabled = true
interval = "5m"
max_age = "1h"
min_segments = 5
max_segments = 100

## Write notification coalescing
[write_notify_config]
enabled = true
max_delay = "20ms"

## ZeroMQ notifications (optional - for local apps)
[notifier_config.default]
bind_port = 5555
high_water_mark = 1000
linger_time = 1000

[notifier_config.users]
bind_port = 5556
high_water_mark = 1000
linger_time = 1000

## Profiling endpoint
[pprof_config]
enabled = true
port = 6060

## Logging
[log_config]
log_level = "info"

[log_config.min_level_percents]
debug = 100.0
info  = 100.0
warn  = 100.0
error = 100.0
```

### 3. Start the Replicator Server

**Replicator Mode** (using `replicator` command):
```bash
./unisondb --config server.toml replicator
```

### 4. Verify Server is Running

**Check HTTP health endpoint**:
```bash
curl http://localhost:4000/health
```

### Development Mode (Insecure)

For quick local testing without TLS:

**server-dev.toml**:
```toml
http_port = 4000

[grpc_config]
port = 4001
allow_insecure = true  # WARNING: Development only!

[storage_config]
base_dir = "./data/server"
namespaces = ["default"]

[log_config]
log_level = "debug"
```

```bash
./unisondb --config server-dev.toml replicator
```

## Running in Relayer Mode

Relayer Mode runs UnisonDB as a **replica** that streams changes from an upstream server.

### 1. Create Relayer Configuration

Create a `relayer.toml` configuration file:

```toml
## HTTP API port (different from server)
http_port = 5000
listen_ip = "0.0.0.0"

## gRPC config (can accept downstream relayers)
[grpc_config]
listen_ip = "0.0.0.0"
port = 5001
cert_path = "./certs/server.crt"
key_path = "./certs/server.key"
ca_path = "./certs/ca.crt"

## Storage configuration
[storage_config]
base_dir = "./data/relayer"
namespaces = ["default", "users", "products"]
bytes_per_sync = "1MB"
# IMPORTANT: segment_size MUST match upstream server!
segment_size = "16MB"
arena_size = "4MB"

## Relayer configuration - connects to upstream
[relayer_config.primary]
namespaces = ["default", "users", "products"]
cert_path = "./certs/client.crt"
key_path = "./certs/client.key"
ca_path = "./certs/ca.crt"
upstream_address = "localhost:4001"
segment_lag_threshold = 100
allow_insecure = false

## Optional: Connect to multiple upstreams
# [relayer_config.secondary]
# namespaces = ["products"]
# upstream_address = "other-server:4001"
# cert_path = "./certs/client.crt"
# key_path = "./certs/client.key"
# ca_path = "./certs/ca.crt"

## ZeroMQ notifications (optional)
[notifier_config.default]
bind_port = 6555
high_water_mark = 1000
linger_time = 1000

## Logging
[log_config]
log_level = "info"

[log_config.min_level_percents]
debug = 1.0   # Sample 1% of debug logs
info  = 10.0  # Sample 10% of info logs
warn  = 100.0
error = 100.0
```

### 2. Start the Relayer Server

**Start relayer**:
```bash
./unisondb --config relayer.toml relayer
```

### 3. Enable gRPC Server on Relayer (Multi-Hop)

To allow downstream relayers to connect to this relayer:

```bash
./unisondb --config relayer.toml --grpc relayer
```

This enables the relayer to act as both a **consumer** (from upstream) and a **producer** (to downstream).

### Development Mode (Insecure)

**relayer-dev.toml**:
```toml
http_port = 5000

[grpc_config]
port = 5001

[storage_config]
base_dir = "./data/relayer"
namespaces = ["default"]
segment_size = "16MB"  # Must match server!

[relayer_config.primary]
namespaces = ["default"]
upstream_address = "localhost:4001"
allow_insecure = true  # WARNING: Development only!
segment_lag_threshold = 100

[log_config]
log_level = "debug"
```

```bash
./unisondb --config relayer-dev.toml relayer
```

## Basic Operations

### Writing Data

**Key-Value Write** (via HTTP):
```bash
# Put a key-value pair
curl -X POST http://localhost:4000/api/v1/namespaces/default/kv \
  -H "Content-Type: application/json" \
  -d '{
    "key": "user:123",
    "value": "eyJuYW1lIjoiQWxpY2UiLCJlbWFpbCI6ImFsaWNlQGV4YW1wbGUuY29tIn0="
  }'
```

**Batch Write**:
```bash
curl -X POST http://localhost:4000/api/v1/namespaces/default/kv/batch \
  -H "Content-Type: application/json" \
  -d '{
    "operations": [
      {"key": "user:1", "value": "..."},
      {"key": "user:2", "value": "..."},
      {"key": "user:3", "value": "..."}
    ]
  }'
```

### Reading Data

**Read from Server**:
```bash
curl http://localhost:4000/api/v1/namespaces/default/kv/user:123
```

**Read from Relayer** (same API):
```bash
curl http://localhost:5000/api/v1/namespaces/default/kv/user:123
```

### Subscribing to Changes (ZeroMQ)

* Build UnisonDB with ZeroMQ Lib

This needs Zero MQ Installed Make Sure You've have it Installed.
[Install ZeroMQ dependency ](https://zeromq.org/download/)

```bash 
# Clone the repository
git clone https://github.com/ankur-anand/unisondb.git
cd unisondb

# Build the binary (CGO required for RocksDB)
CGO_ENABLED=1 go build -tags zeromq ./cmd/unisondb
```


**Python example** (install `pyzmq` first):
```python
import zmq

context = zmq.Context()
socket = context.socket(zmq.SUB)

# Subscribe to 'default' namespace
socket.connect("tcp://localhost:5555")
socket.setsockopt(zmq.SUBSCRIBE, b"")  # Subscribe to all messages

print("Listening for changes on namespace 'default'...")

while True:
    message = socket.recv()
    print(f"Change notification: {message}")
```

**Run the subscriber**:
```bash
python subscriber.py
```

Now any writes to the `default` namespace will trigger notifications!

## Common Deployment Patterns

### 1. Single Server (Development)

```bash
# Terminal 1: Start server
./unisondb --config server-dev.toml replicator
```

### 2. Server + Single Relayer (Read Scaling)

```bash
# Terminal 1: Start server
./unisondb --config server.toml replicator

# Terminal 2: Start relayer
./unisondb --config relayer.toml relayer
```

### 3. Server + Multiple Relayers (Edge Computing)

```bash
# Terminal 1: Start server
./unisondb --config server.toml replicator

# Terminal 2: Start relayer 1
./unisondb --config relayer1.toml relayer

# Terminal 3: Start relayer 2
./unisondb --config relayer2.toml relayer

# Terminal 4: Start relayer 3
./unisondb --config relayer3.toml relayer
```

### 4. Multi-Hop (Relayer → Relayer)

```bash
# Terminal 1: Primary server
./unisondb --config server.toml replicator

# Terminal 2: L1 relayer (with gRPC enabled for downstream)
./unisondb --config relayer-l1.toml --grpc relayer

# Terminal 3: L2 relayer (connects to L1)
# Update relayer-l2.toml upstream_address to point to L1 (localhost:5001)
./unisondb --config relayer-l2.toml relayer
```

Happy building with UnisonDB!
