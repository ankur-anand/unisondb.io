---
title: "UnisonDB Deployment Topologies"
weight: 3
bookCollapseSection: false
images: ["/images/deployment_topologies.png"]
description: "Learn how to deploy UnisonDB — a log-native, real-time database for AI and Edge Computing — across single-server, primary-replica, hub-and-spoke, multi-hop, and hybrid topologies for distributed systems."
keywords: ["UnisonDB deployment", "Edge Computing database", "real-time replication", "distributed database", "primary replica topology", "hub and spoke architecture", "multi-hop replication", "edge AI data sync", "UnisonDB relayer", "UnisonDB replicator"]
linkTitle: "Deployment Topologies"
---

# Deployment Topologies

> Scalable, resilient, and edge-ready — UnisonDB adapts to your deployment architecture.

<img src="/images/deployment_topologies.svg" alt="UnisonDB Deployment Topologies" style="max-width: 100%; height: auto;" />

Design your own [**data mesh**](https://en.wikipedia.org/wiki/Data_mesh) with UnisonDB’s log-native architecture.
Replicate, stream, and sync data effortlessly across edge and cloud environments.

---

## Overview

UnisonDB uses a **dual-mode architecture** — **Server (Replicator)** for writes and **Relayer** for reads — allowing flexible, distributed deployments.  

### Core Components
- **Replicator Mode** – Primary node responsible for writes and WAL-based streaming
- **Relayer Mode** – Read-only node replicating from one or more upstreams
- **Watch API** – Real-time notifications for local apps or edge devices
- **gRPC Replication** – Durable WAL streaming protocol for replication and recovery

---

## 1. Single Server

Simplest deployment for development, testing, or small standalone applications.

```
┌──────────────────────────┐
│   UnisonDB Server        │
│  (Replicator Mode)       │
│                          │
│   • HTTP API :8080       │
│   • gRPC API :9090       │
│   • Watch API :5555      │
│                          │
│   Capabilities:          │
│   * Reads & Writes       │
│   * Local watch events   │
│   * No replication       │
│   * No high availability │
└──────────────────────────┘
```

### Configuration

**Server config** (`server.toml`):
```toml
[server]
mode = "server"
data_dir = "./data"
http_addr = "0.0.0.0:8080"
grpc_addr = "0.0.0.0:9090"

[wal]
segment_size = "16MB"
fsync_enabled = true

[watch]
enabled = true
transport = "zeromq"
bind_addr = "tcp://*:5555"
buffer_size = 10000
```

### Monitoring

```bash
# Check server health
curl http://localhost:8080/health

# View server stats
curl http://localhost:8080/stats

# Monitor watch subscribers
curl http://localhost:8080/stats/watch
```

---

## 2. Replicator + Read Replicas

Replicator handles writes, replicas provide read scalability and geographic distribution.

```
       ┌─────────────────────┐
       │   Replicator Server │  (Replicator Mode)
       │   US-East           │
       │   • Writes          │
       │   • Reads           │
       └──────────┬──────────┘
                  │ gRPC replication (TLS)
         ┌────────┼────────┬────────┐
         ↓        ↓        ↓        ↓
    ┌────────┐┌────────┐┌────────┐┌────────┐
    │Relayer ││Relayer ││Relayer ││Relayer │
    │US-West ││Europe  ││Asia    ││Canada  │
    │        ││        ││        ││        │
    │Read-   ││Read-   ││Read-   ││Read-   │
    │only    ││only    ││only    ││only    │
    └────────┘└────────┘└────────┘└────────┘
```

### Configuration

**Replicator server** (`primary.toml`):
```toml
[server]
mode = "server"
data_dir = "/data/unisondb"
http_addr = "0.0.0.0:8080"
grpc_addr = "0.0.0.0:9090"

[replication]
enabled = true
tls_cert = "/etc/unisondb/tls/server.crt"
tls_key = "/etc/unisondb/tls/server.key"
tls_ca = "/etc/unisondb/tls/ca.crt"

[watch]
enabled = true
bind_addr = "tcp://*:5555"
buffer_size = 10000
```

**Relayer** (`relayer-us-west.toml`):
```toml
[server]
mode = "relayer"
data_dir = "/data/unisondb"
http_addr = "0.0.0.0:8080"

[relayer]
upstreams = [
  "primary.us-east.example.com:9090"
]
tls_cert = "/etc/unisondb/tls/client.crt"
tls_key = "/etc/unisondb/tls/client.key"
tls_ca = "/etc/unisondb/tls/ca.crt"

# Relayer can also publish local watch events
[watch]
enabled = true
bind_addr = "tcp://*:5555"
buffer_size = 10000
```
---

## 3. Hub-and-Spoke (Edge Computing)

Central hub replicates to many edge nodes, each serving local applications.

```
                 ┌──────────────────┐
                 │   Central Hub    │  (Replicator Mode)
                 │   (Cloud/DC)     │
                 │   • All writes   │
                 └────────┬─────────┘
                          │ gRPC replication
         ┌────────────────┼────────────────┐
         ↓                ↓                ↓
    ┌─────────┐      ┌─────────┐      ┌─────────┐
    │ Edge 1  │      │ Edge 2  │      │ Edge 3  │  (Relayers)
    │ Store A │      │ Store B │      │ Store C │
    └────┬────┘      └────┬────┘      └────┬────┘
         │ Watch API      │ Watch API      │ Watch API
         ↓                ↓                ↓
    ┌─────────┐      ┌─────────┐      ┌─────────┐
    │ POS     │      │ POS     │      │ POS     │  Local apps
    │ Inv Mgmt│      │ Inv Mgmt│      │ Inv Mgmt│  (subscribers)
    │ Display │      │ Display │      │ Display │
    └─────────┘      └─────────┘      └─────────┘
```

### Configuration

**Hub (central Replicator server)**:
```toml
[server]
mode = "server"
data_dir = "/data/unisondb"
http_addr = "0.0.0.0:8080"
grpc_addr = "0.0.0.0:9090"

[namespaces]
# Different namespaces for different data types
inventory = { wal_segment_size = "16MB" }
orders = { wal_segment_size = "8MB" }
analytics = { wal_segment_size = "32MB" }

[replication]
enabled = true
tls_enabled = true
max_connections = 1000  # Support many edge nodes
```

**Edge relayer** (`edge-store-001.toml`):
```toml
[server]
mode = "relayer"
data_dir = "/data/unisondb"
http_addr = "127.0.0.1:8080"  # Local only

[relayer]
upstreams = ["hub.central.example.com:9090"]
tls_enabled = true
reconnect_interval = "5s"
buffer_size = "100MB"  # Handle disconnections

# Edge nodes publish local watch events
[watch]
enabled = true
namespaces = ["inventory", "orders"]  # Only needed namespaces
bind_addr = "tcp://127.0.0.1:5555"    # Local IPC only
buffer_size = 5000
```

---

## 4. Multi-Hop Relay (Deep Edge)

Hierarchical replication for deep edge deployments or bandwidth-constrained networks.

```
         ┌──────────────┐
         │   Primary    │  (Replicator Mode - Cloud)
         │   (Cloud)    │
         └──────┬───────┘
                │ gRPC
                ↓
         ┌──────────────┐
         │  Tier 1      │  (Relayer - Regional DC)
         │  Regional    │
         └──────┬───────┘
                │ gRPC
        ┌───────┴────────┐
        ↓                ↓
   ┌─────────┐      ┌─────────┐
   │ Tier 2  │      │ Tier 2  │  (Relayer - Edge Cluster)
   │ West    │      │ East    │
   └────┬────┘      └────┬────┘
        │                │
    ┌───┴───┐        ┌───┴───┐
    ↓       ↓        ↓       ↓
  Tier 3  Tier 3   Tier 3  Tier 3  (Relayer - Leaf Nodes)
  Store1  Store2   Store3  Store4
    ↓       ↓        ↓       ↓
  Local   Local    Local   Local
  Apps    Apps     Apps    Apps
```

### Configuration

**Tier 1 (Regional relayer)**:
```toml
[server]
mode = "relayer"
data_dir = "/data/unisondb"
grpc_addr = "0.0.0.0:9090"  # Accept downstream connections

[relayer]
upstreams = ["primary.cloud.example.com:9090"]
# This relayer can also relay to downstream
enable_relay = true
tls_enabled = true
```

**Tier 2 (Edge cluster relayer)**:
```toml
[server]
mode = "relayer"
data_dir = "/data/unisondb"
grpc_addr = "0.0.0.0:9090"

[relayer]
upstreams = ["tier1-regional.example.com:9090"]
enable_relay = true  # Relay to Tier 3
tls_enabled = true
```

**Tier 3 (Leaf relayer)**:
```toml
[server]
mode = "relayer"
data_dir = "/data/unisondb"
http_addr = "127.0.0.1:8080"

[relayer]
upstreams = ["tier2-west.example.com:9090"]
enable_relay = false  # Leaf node, no downstream
tls_enabled = true

[watch]
enabled = true
bind_addr = "tcp://127.0.0.1:5555"
```

---

## 5. Hybrid: Replication + Local Events

Combines durable replication with local event-driven applications.

```
┌───────────────────────────────────┐
│        Replicator Server          │
│        (Replicator Mode)          │
│                                   │
│   ┌───────────────────┐           │
│   │   Storage Engine  │           │
│   └─────────┬─────────┘           │
│             │                     │
│     ┌───────┴────────┐            │
│     ↓                ↓            │
│  [gRPC]          [Watch API]      │
│  :9090           :5555            │
└────┬──────────────────┬───────────┘
     │                  │
     │                  └──────┐
     ↓                         ↓
┌─────────┐            ┌─────────────┐
│ Remote  │            │ Local Apps  │
│Relayers │            │             │
└─────────┘            │ • Cache     │
                       │ • Analytics │
                       │ • Audit Log │
                       │ • Dashboard │
                       └─────────────┘
```

### Configuration

**Primary with both channels**:
```toml
[server]
mode = "server"
data_dir = "/data/unisondb"
http_addr = "0.0.0.0:8080"
grpc_addr = "0.0.0.0:9090"

# gRPC replication for remote relayers
[replication]
enabled = true
tls_enabled = true

# Watch API for local applications
[watch]
enabled = true
transport = "zeromq"
namespaces = ["users", "sessions", "metrics"]

# Per-namespace watch configuration
[watch.users]
bind_addr = "tcp://*:5555"
buffer_size = 10000

[watch.sessions]
bind_addr = "tcp://*:5556"
buffer_size = 20000  # High-frequency updates

[watch.metrics]
bind_addr = "tcp://*:5557"
buffer_size = 50000
```

---

**Watch API security:**
- Bind to `127.0.0.1` for local-only access

---

## Next Steps

- [Configuration Reference](../getting-started/configurations/) - Detailed config options
- [HTTP API](../api/http-api/) - API documentation
- [Architecture Overview](../architecture/) - Architecture Overview

