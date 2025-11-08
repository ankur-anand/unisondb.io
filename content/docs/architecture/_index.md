---
title: "Architecture Overview – Log-Native Database for Edge and AI Systems"
description: "Learn how UnisonDB’s log-native architecture unifies database durability and message-bus replication — designed for AI agents, edge computing, and real-time data streaming."
linkTitle: "Architecture Overview"
images: ['/images/unisondb_architecture.png']
weight: 2
bookCollapseSection: false
keywords: [
  "UnisonDB architecture",
  "log-native database",
  "edge AI database",
  "edge computing database",
  "AI agents database",
  "real-time replication",
  "write-ahead log",
  "WAL streaming replication",
  "multi-model storage engine",
  "local-first database",
  "distributed database design",
  "reactive database architecture",
  "gRPC replication",
  "B+Tree storage engine",
  "database for AI and edge computing"
]
---

# Architecture Overview

<img src="/images/unisondb_architecture.svg" alt="UnisonDB Architecture Diagram" style="max-width: 100%; height: auto;" />

UnisonDB is a **log-native, real-time database** that replicates like a message bus — merging transactional consistency with streaming replication. It’s purpose-built for [**Edge AI**](https://www.ibm.com/think/topics/edge-ai), [**Edge Computing**](https://en.wikipedia.org/wiki/Edge_computing), and local-first distributed systems that demand low-latency synchronization across nodes.

## Key Characteristics

| Aspect | Design Choice |
|--------|---------------|
| **Storage Model** | Multi-modal: Key-Value, Wide-Column, Large Objects (LOB) |
| **Storage Engine** | B+Tree (LMDB/BoltDB) with in-memory MemTable overlay |
| **Replication** | Log streaming (gRPC) with eventual consistency |
| **Watch API** | Best-effort change notifications (per-namespace) |
| **Consistency** | Single-primary writes, eventual consistency for replicas |
| **Durability** | WAL-first with configurable fsync |
| **Deployment Modes** | Replicator (writable primary) & Relayer (read-only replica) |

## Core Concepts

### 1. Log-Native Design

The Write-Ahead Log (WAL) is a **first-class citizen**, not just a recovery mechanism.

- **Replication = Log streaming**: No separate replication protocol
- **Recovery = Log replay**: State reconstructed from WAL
- **Time-travel enabled**: Historic snapshots from log
- **Single source of truth**: All operations flow through WAL

### 2. Operational Modes

UnisonDB instances run in one of two modes:

#### Replicator Mode (Primary)
Writable instance that maintains the authoritative WAL.

```
┌─────────────────────────────────────┐
│     Replicator Mode Instance        │
│                                     │
│  • Accepts writes (HTTP API)        │
│  • Maintains authoritative WAL      │
│  • Streams to relayers (gRPC)       │
│  • Publishes watch events (local)   │
│  • Serves reads from local storage  │
└─────────────────────────────────────┘
```

#### Relayer Mode (Replica)
Read-only instance that streams changes from upstream replicators.

```
┌─────────────────────────────────────┐
│        Relayer Mode Instance        │
│                                     │
│  • Connects to upstream (gRPC)      │
│  • Receives WAL segment streams     │
│  • Applies to local storage (RO)    │
│  • Can relay to downstream nodes    │
│  • Publishes watch events (local)   │
│  • Serves reads (data locality)     │
└─────────────────────────────────────┘
```

### 3. Dual Communication Channels

UnisonDB separates **distribution** from **local reactivity**:

| Channel | Purpose | Use Case | Scope |
|---------|---------|----------|-------|
| **gRPC** | Durable replication | Node-to-node WAL streaming | Network (cross-machine) |
| **Watch API** | Best-effort notifications | Local application reactivity | IPC (same machine) |

**Design Rationale**:
- **gRPC Replication**: Network-tolerant, authenticated, durable, at-least-once delivery
- **Watch API**: Lightweight, fire-and-forget, at-most-once, local-only

## System Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        UnisonDB Instance                           │
├────────────────────────────────────────────────────────────────────┤
│  Client APIs                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  HTTP REST   │  │ Transactions │  │ Admin/Stats  │              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         └─────────────────┴─────────────────┘                      │
├────────────────────────────────────────────────────────────────────┤
│  Storage Models                                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Key-Value   │  │ Wide-Column  │  │ Large Object │               │
│  └──────┬──────┘  └──────┬───────┘  └──────┬───────┘               │
│         └────────────────┴─────────────────┘                       │
├────────────────────────────────────────────────────────────────────┤
│  Storage Engine (dbkernel)                                         │
│                                                                    │
│  Write:  WAL (append) → MemTable → B+Tree (LMDB)                   │
│  Read:   MemTable + B+Tree → Response                              │
├────────────────────────────────────────────────────────────────────┤
│  Distribution Layer                                                │
│  ┌──────────────────────┐     ┌──────────────────────┐             │
│  │  gRPC Replicator     │     │  Watch API           │             │
│  │  • WAL streaming     │     │  • Change events     │             │
│  │  • TLS/mTLS          │     │  • Per-namespace     │             │
│  │  • At-least-once     │     │  • At-most-once      │             │
│  └──────────┬───────────┘     └──────────┬───────────┘             │
│             v                            v                         │
│      Remote Relayers              Local Applications               │
└────────────────────────────────────────────────────────────────────┘
```

## Core Components

### Write-Ahead Log (WAL)

Append-only transaction log that serves as the source of truth.

| Property | Implementation |
|----------|----------------|
| **Structure** | Segmented files (configurable size, default 16MB) |
| **Format** | Binary with CRC32 checksums per entry |
| **Lifecycle** | Write → fsync → MemTable → B+Tree → Segment cleanup |
| **Purpose** | Durability, replication streaming, crash recovery |

**Segment Format**:
```
┌─────────────────────────────────┐
│  Header (magic, segment#, ts)   │
├─────────────────────────────────┤
│  Entry: flatbuffer-encoded      │
│  Entry: flatbuffer-encoded      │
│  ...                            │
└─────────────────────────────────┘
```

### Storage Engine

Multi-modal storage built on B+Trees with MemTable overlay.

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **MemTable** | In-memory skip list | Write buffer, recent reads |
| **B+Tree** | LMDB/BoltDB | Persistent sorted storage |
| **Encoding** | Model-specific key schemas | Key Space isolation |

**Storage Models**:
```
Key-Value:      <key> -> <value>
Wide-Column:    <rowKey>:<colName> -> <colValue>
Large Object:   <objectKey>:chunk:<N> -> <chunk_data>
```

### Replication System

gRPC-based WAL streaming with bidirectional flow control.

**Replicator Role** (Primary):
- Streams WAL segments to connected relayers

**Relayer Role** (Replica):
- Consumes from one or more upstream servers
- Applies segments in order to local storage
- Can fan-out to downstream relayers (multi-hop)

**Guarantees**:
- **Consistency Model**: Eventual (all replicas converge)
- **Delivery**: At-least-once with gap detection
- **Ordering**: Strict segment sequence enforcement

### Namespace Watch API

Lightweight, best-effort notification system for real-time change awareness.

**Event Structure**:
```json
{
  "namespace": "users",
  "key": "user:123",
  "operation": "put",
  "entry_type": "kv",
  "seq_num": 42
}
```

| Field | Description |
|-------|-------------|
| `namespace` | Namespace where change occurred |
| `key` | Exact key that changed |
| `operation` | Operation: `put`, `delete`, `delete_row` |
| `entry_type` | Operation: `kv`, `lob`, `row` |
| `seq` | Monotonic sequence number (per-namespace ordering of events)  |

**Characteristics**:
- **Per-namespace streams**: Independent event streams per namespace
- **Ordered delivery**: Events delivered in sequence order (if received)
- **Fire-and-forget**: No acknowledgments, best-effort delivery
- **Ring buffer**: Configurable size-based event retention
- **Backpressure handling**: Events dropped if subscribers can't keep up

**Delivery Guarantees**:
- Events may be dropped if consumer can't keep up.
- Cannot replay from a specific sequence
- Subscribers not notified of missed events

**Use Cases**:
- Cache invalidation across microservices
- Trigger event-driven workflows
- Local application coordination
-  NOT for critical workflows requiring guaranteed delivery (use gRPC replication instead)

**Example Flow**:
```
Application A: PUT user:123 → UnisonDB appends to WAL →
Watch stream publishes: {namespace:"users", key:"user:123", entry_type:"PUT", seq:42} →
Applications B, C, D receive notification and react
```

## Design Principles

### Multi-Modal Storage Examples

All storage models share the same WAL and B+Tree, differing only in key encoding:

| Model | Example | Key Encoding |
|-------|---------|--------------|
| **Key-Value** | `user:123` → `{"name":"Alice"}` | `<key>` |
| **Wide-Column** | `user:123` with columns `name`, `email` | `<rowKey>:<colName>` |
| **Large Object** | `video:abc` as streamable chunks | `<objectKey>:chunk:<N>` |

### Edge-First Topology

Designed for hub-and-spoke, multi-hop replication with data locality:

```
          ┌──────────────┐
          │   Primary    │  (Repliactor Mode - accepts writes)
          │   (US-East)  │
          └──────┬───────┘
                 │ gRPC (durable replication)
        ┌────────┼────────┐
        ↓        ↓        ↓
    ┌───────┐┌───────┐┌───────┐
    │Europe ││ Asia  ││US-West│  (Relayer Mode - read replicas)
    │Relayer││Relayer││Relayer│
    └───┬───┘└───┬───┘└───┬───┘
        │        │        │ Watch API (local events)
        ↓        ↓        ↓
    Local    Local    Local
    Apps     Apps     Apps
```

## Data Flow

### Write Path (Server Mode)

```
Write Request → API Handler → Storage Engine
                                    ↓
                    ┌───────────────┴───────────────┐
                    ↓                               ↓
            1. WAL Append                   2. MemTable Update
               (+ fsync)                       (in-memory)
                    ↓                               ↓
            3. Background Flush ────────────────→ B+Tree (LMDB)
                    ↓
            ┌───────┴────────┐
            ↓                ↓
    gRPC Stream        Watch Event
    (to relayers)      (to local apps)
```

### Read Path

```
Read Request → API Handler → Storage Engine
                                    ↓
                            ┌───────┴───────┐
                            ↓               ↓
                      MemTable          B+Tree
                      (check first)  (if not found)
                            └───────┬───────┘
                                    ↓
                            Merge & Response
```

### Replication Flow (gRPC)

```
Replicator (Primary)                     Relayer (Replica)
      │                                      │
      │ ─── WAL Segment (gRPC stream) ────→  |
      │     [metadata + binary + CRC]        │
      │                                      ↓
      │                              1. Validate checksum
      │                              2. Append to local WAL
      │                              3. Apply to MemTable
      │                              4. Flush to B+Tree
      │                                      ↓
      │                              Can relay downstream
      │                              Can notify local apps
```

### Watch Event Flow

```
WAL Write → Watch Event Builder → Ring Buffer → Transport Layer
                                                      │
                                       ┌──────────────┼──────────────┐
                                       ↓              ↓              ↓
                                    App A          App B          App C
                                 (subscriber)   (subscriber)   (subscriber)
```

**Event Details**:
- Triggered on every WAL append (PUT/DELETE/UPDATE)
- Buffered in ring buffer (configurable size)
- Events dropped if buffer full or subscriber slow

## Storage Layout

```
data/
├── <namespace>/
│   ├── wal/
│   │   ├── segment-00000000     # 16MB segments (configurable)
│   │   ├── segment-00000001
│   │   └── ...
│   ├── db/
│   │   ├── data.mdb             # LMDB data file
│   │   └── lock.mdb             # LMDB lock file
│   └── checkpoint/
│       └── last_applied         # Recovery point
```

**Per-Namespace Isolation**: Each namespace has independent WAL, DB, and checkpoint state.

## System Characteristics

### Consistency Model

| Aspect | Guarantee |
|--------|-----------|
| **Writes** | Single primary (Replicator Mode) for linearizability |
| **Reads** | Eventually consistent across relayers |
| **Replication** | At-least-once delivery, ordered segments |
| **Isolation** | Per-namespace (independent namespaces) |

### Durability & Recovery

**Crash Recovery**:
1. Scan WAL for uncommitted operations
2. Replay WAL to rebuild MemTable and B+Tree
3. Validate checkpoint consistency
4. Resume operations

**Replication Recovery** (Relayer Mode):
1. Determine last applied segment from checkpoint
2. Reconnect to upstream at last offset
3. Request missing segments (gap detection)
4. Apply backlog before serving reads

**Data Durability**:
- WAL with optional fsync (configurable)
- CRC32 checksums on all WAL entries
- Segment-level validation during replication

## Deployment Topologies

UnisonDB supports various deployment patterns. See the **[Deployment Guide](../deployment/)** for detailed configurations and examples.

### Example: Hub-and-Spoke

```
           ┌──────────┐
           │   Hub    │ (Replicator Mode)
           └────┬─────┘
                │ gRPC (durable replication)
     ┌──────────┼──────────┐
     ↓          ↓          ↓
  ┌──────┐  ┌──────┐  ┌──────┐
  │Edge 1│  │Edge 2│  │Edge 3│ (Relayers)
  └──┬───┘  └──┬───┘  └──┬───┘
     │         │         │ Watch API (local events)
     ↓         ↓         ↓
  Local     Local     Local
  Apps      Apps      Apps
```

**Key characteristics:**
- Central hub accepts all writes
- Edge relayers provide data locality
- Local applications subscribe to watch events

For detailed configurations, monitoring, and operational guidance, see the **[Deployment Topologies Guide](../deployment/)**.

## Tradeoffs & Limitations

### Design Tradeoffs

| Aspect | Tradeoff | Rationale |
|--------|----------|-----------|
| **Write Scalability** | Single primary per namespace | Ensures linearizable writes, simplifies conflict resolution |
| **Read Consistency** | Eventual consistency on relayers | Enables high read scalability and data locality |
| **Replication Model** | At-least-once delivery | Prioritizes availability over exactly-once semantics |
| **Watch Events** | At-most-once, best-effort | Minimizes latency, lightweight notification for non-critical use cases |

### Current Limitations

1. **Write Scaling**: Single primary per namespace (no multi-master)
2. **Consistency**: No strong consistency guarantees for relayer reads
3. **Transactions**: Limited to single-namespace operations
4. **Query Model**: No complex queries (no SQL, joins, aggregations)
5. **Schema**: Schema-less (application-managed structure)

### When to Use UnisonDB

**Good Fit**:
- Edge computing with hub-and-spoke topology
- Read-heavy workloads requiring data locality
- Event-driven architectures (via Watch API)
- Applications tolerating eventual consistency
- Key-value or wide-column access patterns
- Large object storage with streaming

**Not a Good Fit**:
- Strong consistency requirements across replicas
- Complex relational queries (joins, aggregations)
- Multi-region active-active writes
- Workloads requiring ACID transactions across namespaces

## Summary

UnisonDB combines **database semantics** with **streaming mechanics** through:

1. **Log-Native Design**: WAL as first-class citizen (replication = log streaming)
2. **Dual Communication**: gRPC for distribution, Watch API for local reactivity
3. **Dual Modes**: Server (writable primary) and Relayer (read replicas)
4. **Multi-Modal Storage**: Key-Value, Wide-Column, Large Objects on shared B+Tree

**Architecture Strengths**:
- Data locality through edge replicas
- Event-driven integration via Watch API
- Simple operational model (log streaming)
- Flexible deployment topologies (hub-and-spoke, multi-hop)

**Best For**: Edge computing, local-first applications, and read-scalable systems with eventual consistency tolerance.
