---
title: "Architecture"
weight: 2
bookCollapseSection: false
---

# Architecture Overview

UnisonDB is a log-native database that replicates like a message bus, combining database semantics with streaming mechanics. It operates in two modes: **Server Mode** (primary/standalone) and **Relayer Mode** (replica/streaming).

## Operational Modes

### Server Mode
Primary instance that accepts writes and serves reads. This is the source of truth for data.

**Characteristics**:
- Accepts write operations via HTTP APIs
- Maintains the authoritative Write-Ahead Log (WAL)
- Streams WAL changes to relayers via gRPC
- Can publish local change notifications via ZeroMQ
- Stores data in LMDB B+Tree

### Relayer Mode
Replica instance that streams changes from one or more upstream servers and serves local reads.

**Characteristics**:
- Connects to upstream server(s) via gRPC
- Receives and applies WAL segment streams
- Read-only local access (no direct writes)
- Can relay to downstream instances
- Can publish local change notifications via ZeroMQ
- Provides data locality and read scalability

## Communication Architecture

UnisonDB uses two distinct communication channels for different purposes:

### 1. gRPC - Replication Channel (Node-to-Node)

**Purpose**: Stream WAL segments between UnisonDB nodes for replication.

**Use Cases**:
- Server → Relayer replication (primary to replica)
- Multi-hop replication (relayer → relayer)
- Cross-region data distribution
- Hub-and-spoke topologies

**Characteristics**:
- Bidirectional streaming RPC
- mTLS authentication.
- Segment-based streaming with flow control

**Security**:
- TLS/mTLS required in production
- Can run insecure mode (development only)

### 2. ZeroMQ - Notification Channel (Interprocess Communication)

**Purpose**: Publish real-time change notifications to local applications on the same machine.

**Use Cases**:
- React to data changes in real-time
- Trigger local application logic on writes
- Build event-driven architectures
- Cache invalidation signals
- Local microservices coordination

**Characteristics**:
- PUB/SUB pattern (one-to-many)
- Per-namespace sockets (isolation)
- Local IPC only (same machine)
- Fire-and-forget (no acknowledgments)

**Typical Flow**:
```
Application A writes to UnisonDB → UnisonDB writes to WAL →
ZeroMQ publishes notification → Applications B, C, D receive notification
```

## High-Level Architecture

```
+--------------------------------------------------------------------------------------+
|                                 UnisonDB Instance                                   |
+--------------------------------------------------------------------------------------+
|                              Client / Integration Layer                             |
|                                                                                      |
|   +------------------+   +------------------+   +------------------+                  |
|   |   HTTP / REST    |   | Transaction API  |   |  Admin / Stats   |                  |
|   +------------------+   +------------------+   +------------------+                  |
|              |                        |                       |                     |
+--------------+------------------------+-----------------------+----------------------+
|                              Multi-Model Data Interfaces                            |
|                                                                                      |
|   +-------------+    +----------------+    +------------------+                      |
|   | Key-Value   |    | Wide-Column    |    | LOB (Chunked TXN)|                      |
|   +-------------+    +----------------+    +------------------+                      |
|              \__________   _____________/                                            |
|                         \ /                                                         |
+--------------------------v-----------------------------------------------------------+
|                               Storage Engine (dbkernel)                             |
|                                                                                      |
|   Write Path:                                                                        |
|     +----------------+    +----------------+    +----------------+                   |
|     |   WALFS        | -> |   MemTable     | -> |   B+Tree Store |                   |
|     | (append-only,  |    | (in-memory)    |    | (LMDB / Bolt)  |                   |
|     |  mmap, ordered)|    +----------------+    +----------------+                   |
|     +----------------+                                                           *   |
|                                                                                 *    |
|   Read Path: B+Tree (+MemTable overlay) ---------------------------------------*---->|
|                                                                                      |
+--------------------------------------------------------------------------------------+
|                           Replication & Notifications                                |
|                                                                                      |
|   +-----------------------+         +-----------------------------+                  |
|   | gRPC Replicator       |         | Change Notifier             |                  |
|   | - Stream WAL          |         | - PUB/SUB (ZeroMQ / MQTT)   |                  |
|   | - Offset catch-up     |         | - Per-namespace topics      |                  |
|   | - TLS / mTLS          |         | - Fire-and-forget delivery  |                  |
|   +-----------------------+         +-----------------------------+                  |
|           |                                        |                                 |
|           v                                        v                                 |
|   +------------------+                    +------------------+                       |
|   |  Follower Node   |                    | Reactive Clients |                       |
|   |  (Edge Replica)  |                    | (Dashboards etc.)|                       |
|   +------------------+                    +------------------+                       |
+--------------------------------------------------------------------------------------+

Data Flow Summary:
  1. Client writes → WALFS (append-only)
  2. WALFS → MemTable → B+Tree (commit-ordered)
  3. gRPC Replicator streams WAL entries to other nodes
  4. Notifier publishes change events to local/remote subscribers
  5. Readers query locally from B+Tree + MemTable (no network call)
```

## Core Components

### [Write-Ahead Log (WAL)]
Transaction logging system that enables streaming replication and crash recovery.

**Responsibilities**:
- Sequential write log (append-only)
- Segment-based storage (configurable size)
- Checksum validation for integrity
- Source of truth for replication
- Crash recovery mechanism

**WAL Lifecycle**:
1. Client writes → WAL append
2. WAL fsync (durability)
3. MemTable update
4. Background flush to B+Tree
5. Segment cleanup (after checkpoint)

### [Storage Engine]
Built on B+Trees (LMDB/BOLT-DB).

**Responsibilities**:
- Multi-modal data storage (KV, Wide-Column, LOB)
- Crash recovery via WAL replay

### [Replication System]
gRPC-based streaming replication with WAL segment fan-out.

**Server Mode** (WAL Producer):
- Exposes gRPC streaming endpoint
- Streams WAL segments to connected relayers
- Monitors replication lag

**Relayer Mode** (WAL Consumer):
- Connects to upstream gRPC endpoints
- Receives WAL segments in order
- Applies segments to local storage
- Can relay to downstream instances
- Multiple upstream sources supported

**Replication Guarantees**:
- **Eventual Consistency**: All replicas converge to same state
- **Ordered Delivery**: Segments applied in correct order
- **At-Least-Once**: Retries ensure delivery
- **Gap Detection**: Missing segments are requested

### [Change Notifications]
ZeroMQ-based PUB/SUB system for local interprocess communication.

**Architecture**:
- One ZeroMQ PUB socket per namespace
- Local applications subscribe via SUB sockets
- Notifications triggered on WAL writes
- Non-blocking, fire-and-forget delivery

**Notification Flow**:
```
Write Request → WAL Append → [optional delay for coalescing] →
ZeroMQ Publish → Local Subscribers
```

**Configuration Options**:
- `bind_port`: TCP port for namespace socket
- `high_water_mark`: Max queued messages before dropping
- `linger_time`: Wait time for pending messages on shutdown

**Use Case Example**:
```
Namespace: "inventory"
- UnisonDB binds: tcp://*:5555
- Warehouse app subscribes: tcp://localhost:5555
- Web API app subscribes: tcp://localhost:5555
- Both apps receive real-time inventory updates
```

## Design Principles

### 1. Log-Native Design

The Write-Ahead Log is a first-class citizen, not just a recovery mechanism.

**Implications**:
- **Log is the source of truth**: All operations flow through WAL
- **Replication is log streaming**: No separate replication protocol needed
- **Recovery is log replay**: Database state reconstructed from WAL
- **Time-travel possible**: Historic state can be reconstructed from log

### 2. Multi-Modal Storage

Three storage models share the same underlying B+Tree and WAL.

**Key-Value Storage**:
```
Key: "user:123"
Value: {"name": "Alice", "email": "alice@example.com"}
```

**Wide-Column Storage** (like Cassandra/HBase):
```
RowKey: "user:123"
Columns: {
  "profile:name": "Alice",
  "profile:email": "alice@example.com",
  "settings:theme": "dark"
}
```

**Large Object (LOB) Storage**:
```
ObjectKey: "video:abc"
Chunks: [chunk0, chunk1, chunk2, ...] (streaming support)
```

### 3. Edge-First Architecture

Designed for edge computing, local-first applications, and distributed topologies.

**Characteristics**:
- Hub-and-Spoke: Central primary, many edge replicas
- Multi-hop: Relayers can relay to other relayers
- Eventual consistency: Replicas converge over time
- Offline operation: Edge nodes work independently
- Local reads: Read from nearby replica (data locality)

**Example Topology**:
```
          ┌──────────────┐
          │ Primary (US) │
          └──────┬───────┘
                 │ gRPC replication
        ┌────────┴────────┐
        ↓                 ↓
  ┌──────────┐      ┌──────────┐
  │ Relayer  │      │ Relayer  │
  │ (Europe) │      │  (Asia)  │
  └────┬─────┘      └────┬─────┘
       │                 │
   ZeroMQ            ZeroMQ
   (local)           (local)
       ↓                 ↓
  Local Apps        Local Apps
```

### 4. Separation of Concerns

**gRPC for Distribution** (between machines):
- Durable replication
- Network-tolerant
- Authenticated and encrypted
- Handles intermittent connectivity

**ZeroMQ for Local Reactivity** (within machine):
- Fast interprocess communication
- Event-driven architecture
- No network overhead
- Decoupled local services

## Data Flow

### Write Path (Server Mode)

```
Client Write Request
    ↓
HTTP/gRPC API Handler
    ↓
Transaction Context
    ↓
Storage Engine (dbkernel)
    ↓
┌───────────────────────────┐
│ 1. Append to WAL          │ ← Sequential write (durability)
│ 2. fsync (if configured)  │
└───────────┬───────────────┘
            ↓
┌───────────────────────────┐
│ 3. Update MemTable        │ ← In-memory write buffer
└───────────┬───────────────┘
            ↓
┌───────────────────────────┐
│ 4. Background flush to    │ ← Periodic flush to LMDB
│    B+Tree when full       │
└───────────┬───────────────┘
            ↓
Response to Client
            │
            ├──────────────────┐
            ↓                  ↓
   ┌──────────────────┐  ┌─────────────────┐
   │ gRPC Stream to   │  │ ZeroMQ Publish  │
   │ Relayers         │  │ to Local Apps   │
   └──────────────────┘  └─────────────────┘
```

### Read Path

```
Client Read Request
    ↓
HTTP/gRPC API Handler
    ↓
Storage Engine (dbkernel)
    ↓
┌───────────────────────────┐
│ 1. Check MemTable         │ ← Recent writes (hot data)
└───────────┬───────────────┘
            │ (if not found)
            ↓
┌───────────────────────────┐
│ 2. Query B+Tree (LMDB)    │ ← Persistent storage
└───────────┬───────────────┘
            ↓
Response to Client
```

### Replication Flow (gRPC)

```
Server Mode (Primary)
    │
    │ gRPC Streaming RPC
    │ (bidirectional)
    ↓
┌─────────────────────────┐
│ WAL Segment Streaming   │
│ - Segment metadata      │
│ - Binary WAL data       │
│ - Checksum validation   │
└──────────┬──────────────┘
           │ Network (TLS/mTLS)
           ↓
Relayer Mode (Replica)
    │
    ↓
┌─────────────────────────┐
│ 1. Receive WAL segment  │
│ 2. Validate checksum    │
│ 3. Write to local WAL   │
│ 4. Apply to MemTable    │
│ 5. Flush to B+Tree      │
└─────────────────────────┘
    │
    ├──────────────────┐
    ↓                  ↓
Can relay to      Can notify
downstream        local apps
relayers          via ZeroMQ
```

### Notification Flow (ZeroMQ)

```
WAL Write Event
    ↓
┌─────────────────────────────┐
│ Write Notify Coalescer      │ ← Optional delay (max_delay)
│ (reduces notification spam) │    to batch rapid writes
└───────────┬─────────────────┘
            ↓
┌─────────────────────────────┐
│ ZeroMQ PUB Socket           │ ← Per-namespace socket
│ tcp://*:{bind_port}         │
└───────────┬─────────────────┘
            │ Local IPC
    ┌───────┴───────┬──────────┐
    ↓               ↓          ↓
┌──────────┐  ┌──────────┐  ┌──────────┐
│  App A   │  │  App B   │  │  App C   │
│  (SUB)   │  │  (SUB)   │  │  (SUB)   │
└──────────┘  └──────────┘  └──────────┘
 tcp://localhost:{bind_port}
```

## Storage Layout

### Directory Structure

```
data/
├── default/                    # Namespace: default
│   ├── wal/                   # Write-Ahead Log directory
│   │   ├── segment-00000000   # WAL segment 0 (16MB)
│   │   ├── segment-00000001   # WAL segment 1
│   │   ├── segment-00000002   # WAL segment 2
│   │   └── ...
│   ├── db/                    # LMDB B+Tree database
│   │   ├── data.mdb           # LMDB data file
│   │   └── lock.mdb           # LMDB lock file
│   └── checkpoint             # Checkpoint metadata
│       └── last_applied       # Last applied WAL offset
├── users/                     # Namespace: users
│   ├── wal/
│   ├── db/
│   └── checkpoint
└── metrics/                   # Namespace: metrics
    ├── wal/
    ├── db/
    └── checkpoint
```

### Key Encoding

Internal key structure varies by storage model:

```
Key-Value:
  Internal Key: <user-provided-key>
  Example: "user:123"

Wide-Column (Row):
  Internal Key: <rowKey>:<columnName>
  Example: "user:123:profile:name"

Large Object (LOB):
  Internal Key: <objectKey>:chunk:<chunkIndex>
  Example: "video:abc:chunk:0000000042"
```

### WAL Segment Format

```
┌────────────────────────────────────────┐
│          WAL Segment Header            │
│  - Magic Number (validation)           │
│  - Segment Number                      │
│  - Timestamp                           │
└────────────────────────────────────────┘
│          WAL Entry 1                   │
│  - Entry Type (Put/Delete/Txn)         │
│  - Namespace                           │
│  - Key Length                          │
│  - Value Length                        │
│  - Key Data                            │
│  - Value Data                          │
│  - Checksum (CRC32)                    │
├────────────────────────────────────────┤
│          WAL Entry 2                   │
│  ...                                   │
├────────────────────────────────────────┤
│          WAL Entry N                   │
│  ...                                   │
└────────────────────────────────────────┘
```

## Concurrency Model

**Per-Namespace Isolation**:
- Different namespaces are independent
- Concurrent access to different namespaces

## Failure Modes and Recovery

### Crash Recovery (Both Modes)

**Server Mode Recovery**:
1. **Scan WAL**: Identify all uncommitted operations
2. **Replay WAL**: Re-apply operations to MemTable and B+Tree
3. **Validate Checkpoint**: Ensure consistency with last checkpoint
4. **Resume Operations**: Begin accepting requests

**Relayer Mode Recovery**:
1. **Scan Local WAL**: Determine last applied segment
2. **Resume Stream**: Reconnect to upstream at last offset
3. **Gap Detection**: Request missing segments if needed
4. **Catch-up**: Apply backlog before accepting reads

### Replication Failure

**Segment Gap Detection**:
- Relayers track segment sequence numbers
- Missing segments trigger explicit requests
- Prevents inconsistent state

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

## Summary

UnisonDB's architecture is built around two key insights:

1. **Dual Communication Channels**:
   - **gRPC** for durable, network-based replication between nodes
   - **ZeroMQ** for fast, local interprocess notifications

2. **Dual Operational Modes**:
   - **Server Mode** for writable primary instances
   - **Relayer Mode** for read-scalable replicas

This separation allows you to build sophisticated topologies:
- Hub-and-spoke for edge computing
- Multi-region for geographic distribution
- Local event-driven architectures via ZeroMQ
- Scalable read replicas via gRPC streaming

The log-native design ensures that replication, recovery, and notifications all flow naturally from the Write-Ahead Log, making the system simple to reason about while remaining powerful and flexible.
