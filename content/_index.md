---
title: "UnisonDB"
type: docs
---

# UnisonDB

**A reactive, multi-modal database built on B+Trees and WAL-based streaming replication, designed for local-first, edge-scale applications.**

<img src="images/logo.svg" alt="UnisonDB" width="300" />


> [!NOTE]
> **What is UnisonDB?**
> UnisonDB is a log-native database that replicates like a message bus. It combines database semantics with streaming mechanics to provide a powerful foundation for reactive architectures and local-first systems.

## Key Features

- **Multi-Modal Storage**: Key-Value, Wide-Column, and Large Objects (LOB)
- **Streaming Replication**: WAL-based replication with sub-second fan-out to 100+ edge replicas
- **Real-Time Notifications**: ZeroMQ-based change notifications with sub-millisecond latency
- **Durable & Fast**: B+Tree storage with Write-Ahead Logging
- **Edge-First Design**: Optimized for edge computing and local-first architectures
- **Namespace Isolation**: Multi-tenancy support with namespace-based isolation

<style>
.architecture-section { margin: 1rem 0 2rem; }

.architecture-section .architecture-diagram {
  font-family: "JetBrains Mono","Fira Code", ui-monospace, Menlo, monospace;
  font-size: 14px;
  line-height: 1.35;
  letter-spacing: 0.2px;
  background: #0b1021;                
  color: #e6ecff;                     
  border: 1px solid #4c63ff;          
  border-radius: 10px;
  padding: 1.25rem 1.5rem;
  box-shadow: 0 4px 22px rgba(0,0,0,.18);
  overflow-x: auto;
  white-space: pre;
  opacity: 1 !important;               
  filter: none !important;           
}

@media (prefers-color-scheme: light) {
  .architecture-section .architecture-diagram {
    background: #0b1021;
    color: #f4f7ff;
    border-color: #5b72ff;
  }
}
</style>

<section class="architecture-section">
  <h2>Architecture Highlights</h2>
  <pre class="architecture-diagram">
                         ┌───────────────────────────────────────────────┐
                         │                   UnisonDB                    │
                         ├───────────────────────────────────────────────┤
Clients & Systems  ────▶ │           HTTP API (REST / Txn)               │
                         │       (Reads, Writes, Transactions)           │
                         ├───────────────────────────────────────────────┤
                         │                Storage Engine                 │
                         │   ┌──────────┐  ┌──────────┐  ┌────────┐      │
                         │   │   KV     │  │   Row    │  │  LOB   │      │
                         │   └──────────┘  └──────────┘  └────────┘      │
                         ├───────────────────────────────────────────────┤
                         │     Log-Native Core: WAL + MemTable +         │
                         │             B+Tree (LMDB/BoltDB)              │
                         ├───────────────────────────────────────────────┤
                         │             Streaming Replication             │
                         │         (Hub-and-Spoke / Peer-to-Peer)        │
                         └───────┬───────────────────────────┬───────────┘
                                 │                           │
                                 ▼                           ▼
                        ┌─────────────────┐         ┌─────────────────┐
                        │   Edge Replica  │  ...    │   Edge Replica  │
                        │  Local Queries  │         │  Local Queries  │
                        └────────┬────────┘         └────────┬────────┘
                                 │                           │
                                 │  (Post-Commit)            │  (Post-Commit)
                                 ▼                           ▼
                       ┌────────────────────────────┐   ┌────────────────────────────┐
                       │   ZeroMQ PUB/SUB Sidecar   │   │   ZeroMQ PUB/SUB Sidecar   │
                       │   • Local change feed      │   │   • Local change feed      │
                       │   • Runs on primary/edge   │   │   • Runs on primary/edge   │
                       └────────────────────────────┘   └────────────────────────────┘
  </pre>
</section>


## Quick Start

```bash
# Clone the repository
git clone https://github.com/ankur-anand/unisondb
cd unisondb

# Build
go build -o unisondb ./cmd/unisondb

# Run in replicator mode (primary)
./unisondb server --mode replicator --config config.toml

# Use the HTTP API
curl -X PUT http://localhost:4000/api/v1/default/kv/mykey \
  -H "Content-Type: application/json" \
  -d '{"value":"bXl2YWx1ZQ=="}'
```

## Deployment Topologies

### 1. Single Server

```
┌─────────────────┐
│  Primary Server │
│  (Server Mode)  │
│  - HTTP API     │
│  - Writes OK    │
│  - Reads OK     │
└─────────────────┘
```

Simple deployment for small workloads.

### 2. Primary + Replicas (Read Scaling)

```
       ┌─────────────────┐
       │  Primary Server │
       │  (Server Mode)  │
       └────────┬────────┘
                │ gRPC replication
       ┌────────┴────────┐
       ↓                 ↓
┌─────────────┐   ┌─────────────┐
│  Relayer 1  │   │  Relayer 2  │
│ (Read-only) │   │ (Read-only) │
└─────────────┘   └─────────────┘
```

Primary handles all writes, relayers provide read scalability.

### 3. Hub-and-Spoke (Edge Computing)

```
           ┌──────────────────┐
           │   Central Hub    │
           │  (Server Mode)   │
           └────────┬─────────┘
                    │ gRPC replication
         ┌──────────┼──────────┐
         ↓          ↓          ↓
    ┌────────┐ ┌────────┐ ┌────────┐
    │ Edge 1 │ │ Edge 2 │ │ Edge 3 │
    │Relayer │ │Relayer │ │Relayer │
    └───┬────┘ └───┬────┘ └───┬────┘
        │          │          │
     ZeroMQ     ZeroMQ     ZeroMQ
        ↓          ↓          ↓
    Local      Local      Local
    Apps       Apps       Apps
```

Central hub replicates to many edge nodes. Each edge serves local apps via ZeroMQ.

### 4. Multi-Hub (Geographic Distribution)

```
┌──────────────┐           ┌──────────────┐
│  Hub US-East │◄─────────►│  Hub Europe  │
│ (Server Mode)│   gRPC    │(Server Mode) │
└──────┬───────┘  bidirect.└──────┬───────┘
       │                          │
   gRPC│                          │gRPC
       ↓                          ↓
  ┌────────┐                 ┌────────┐
  │ East   │                 │ Europe │
  │Relayers│                 │Relayers│
  └────────┘                 └────────┘
```

Multiple hubs for geographic distribution. Each hub can be a server accepting local writes.

### 5. Multi-Hop Relay (Deep Edge)

```
    ┌─────────┐
    │ Primary │ (Server Mode)
    └────┬────┘
         │ gRPC
         ↓
    ┌─────────┐
    │ Relay L1│ (Relayer Mode)
    └────┬────┘
         │ gRPC
    ┌────┴─────┐
    ↓          ↓
┌─────────┐ ┌─────────┐
│Relay L2a│ │Relay L2b│ (Relayer Mode)
└────┬────┘ └────┬────┘
     │           │ ZeroMQ
     ↓           ↓
  Local       Local
  Apps        Apps
```

Multi-hop replication for deep edge deployments.

### 6. Hybrid: Replication + Local Notifications

```
┌─────────────────────────────────────┐
│         Primary Server              │
│         (Server Mode)               │
│  ┌──────────────┐                   │
│  │   Storage    │                   │
│  └──────┬───────┘                   │
│         │                           │
│    ┌────┴────┐                      │
│    ↓         ↓                      │
│ [gRPC]   [ZeroMQ]                   │
└───┬──────────┬──────────────────────┘
    │          │
    │          └──→ Local Apps (IPC)
    │              - Cache service
    │              - Analytics service
    │              - Audit logger
    ↓
  Relayers (other machines)
```

Same instance uses gRPC for replication AND ZeroMQ for local notifications.

## Use Cases

### Edge Computing

Deploy UnisonDB at the edge for low-latency data access with automatic replication to central hubs.

### Local-First Applications

Build responsive applications that work offline and sync when connected.

### Real-Time Analytics

Stream data changes to analytics systems with sub-second latency.

### Multi-Region Replication

Replicate data across geographic regions with configurable consistency.

## Documentation Structure

- **[Getting Started](/docs/getting-started/)** - Installation, configuration, and quick start
- **[Architecture](/docs/architecture/)** - Deep dive into UnisonDB internals
- **[HTTP API](/docs/api/http-api/)** - REST API reference with examples

## Community & Support

- **GitHub**: [github.com/ankur-anand/unisondb](https://github.com/ankur-anand/unisondb)
- **Issues**: [Report bugs or request features](https://github.com/ankur-anand/unisondb/issues)
- **Discussions**: [Join the conversation](https://github.com/ankur-anand/unisondb/discussions)

## License

UnisonDB is released under the Apache 2.0 License.
