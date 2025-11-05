---
title: "UnisonDB – Reactive, Log-Native, Multi-Model Database for the Edge"
description: "UnisonDB is a reactive, log-native database built in Go — combining B+Tree storage and WAL-based streaming replication for real-time edge apps."
summary: "Reactive, log-native, multi-model database built in Go — with WAL-based streaming replication and B+Tree storage powering real-time, local-first systems at the edge."
image: "/images/unison_overview.png"
---

# UnisonDB

> **Replicates like a message bus. Acts like a database.**

<img src="images/logo.svg" alt="UnisonDB" width="300" />

# What is Unisondb ?

**UnisonDB** is a **reactive, log-native, multi-model database** built for **real-time and edge-scale applications**.  
It combines **B+Tree storage** with **WAL-based streaming replication**, enabling **near-instant fan-out** across hundreds of replicas — all while preserving **strong consistency and durability**.  


## Key Features
- **Multi-Modal Storage**: Key-Value, Wide-Column, and Large Objects (LOB)
- **Streaming Replication**: WAL-based replication with sub-second fan-out to 100+ edge replicas
- **Real-Time Notifications**: ZeroMQ-based change notifications with sub-millisecond latency
- **Durable & Fast**: B+Tree storage with Write-Ahead Logging
- **Edge-First Design**: Optimized for edge computing and local-first architectures
- **Namespace Isolation**: Multi-tenancy support with namespace-based isolation

<section class="architecture-section">
  <h2>Architecture Highlights</h2>
</section>

<img src="images/unison_overview.png"  alt="UnisonDB Overview" />


> A **reactive, log-native database** that unifies **storage, streaming, and sync** —  
> powering the next generation of **edge and real-time systems**.

## Namespace

A single UnisonDB instance can host many namespaces — each namespace is a fully isolated database with its own:

* Write-Ahead Log (WAL)
* MemTable (in-memory write buffer)
* B+Tree store (persistent)
* Each namespace emits its own WAL stream, enabling selective replication and low coupling.

All namespaces are exposed through one unified API and one process — making UnisonDB both multi-tenant and operationally simple.

<img src="images/unisondb_overview_namespace.png"  alt="UnisonDB namespace Overview" />

## Data Model

UnisonDB’s multi-model architecture lets you design data the way your application thinks.
Within a single instance, you can mix Key-Value, Wide-Column, and Large Object (LOB) storage models — all backed by the same WAL and B+Tree engine — without managing multiple systems.

* **Key-Value**: Stores and retrieves data by a single unique key — simple, fast, and ideal for lookups.
<img src="images/kv.png"  alt="kv" width="300"/>
* **Wide-Column**: Organizes data into rows with multiple named columns — great for structured, evolving entities.
<img src="images/wide_column.png"  alt="Wide Column" />
* **Large Object (LOB)**: Manages large binary or text data in chunks — perfect for files, media, or backups.
<img src="images/chunk_kv.png"  alt="lob" />

## Use Cases

UnisonDB is designed for systems where data and computation must live close together — reducing network hops, minimizing latency, and enabling real-time responsiveness at scale.
By co-locating data with the services that use it, UnisonDB eliminates the traditional separation between database and stream processor, allowing applications to react instantly to local changes while staying in sync globally.

### Event-Driven Microservices

Use UnisonDB as a reactive state store that behaves like both a database and a message bus. Services can subscribe to change streams to react instantly to updates without an external queue.

### Globaly Synced Cached

Deploy UnisonDB near your application servers as a fast, persistent local cache. Unlike Redis or Memcached, UnisonDB provides WAL-backed durability and asynchronous replication, ensuring that cached state can survive restarts and sync globally.


### Real-Time Personalization

Keep user context and recommendation data local to the edge where requests occur. Updates replicate asynchronously to regional hubs — enabling fast, context-aware responses without waiting for a central database.

### Multi-Region Replication

Replicate data across geographic regions with configurable consistency.

### Ad Delivery & Bidding Systems

Run bidding logic or campaign evaluation next to local user data. Each region holds its own copy of targeting indexes, ensuring low-latency ad decisions while maintaining eventual global consistency.

### Real-Time Analytics at the Edge

Perform in-situ aggregation, filtering, and anomaly detection close to where data is produced. UnisonDB’s WAL-based replication lets you stream computed insights upward to central clusters without heavy data movement.


> In essence: UnisonDB is ideal for systems that demand reactivity, locality, and reliability — anywhere you need your data and compute to move together instead of apart.

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

## Documentation

- **[Getting Started](/docs/getting-started/)** - Installation, configuration, and quick start
- **[Architecture](/docs/architecture/)** - Deep dive into UnisonDB internals
- **[HTTP API](/docs/api/http-api/)** - REST API reference with examples
- **[Examples](/docs/examples/)** - Various Use Case Examples

## Community & Support

- **GitHub**: [github.com/ankur-anand/unisondb](https://github.com/ankur-anand/unisondb)
- **Issues**: [Report bugs or request features](https://github.com/ankur-anand/unisondb/issues)
- **Discussions**: [Join the conversation](https://github.com/ankur-anand/unisondb/discussions)

## License

UnisonDB is released under the Apache 2.0 License.
