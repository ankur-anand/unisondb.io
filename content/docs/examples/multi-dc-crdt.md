---
title: "How to Build Conflict-Free Multi-Datacenter Systems with CRDTs and UnisonDB"
linkTitle: "Multi-DC CRDT Replication"
description: "Learn how to build globally distributed, conflict-free applications using CRDTs, real-time replication, and ZeroMQ notifications in UnisonDB."
date: 2025-11-02
type: posts
author: "Ankur Anand"
weight: 20
draft: false
bookToC: true
images: ['/images/multi_dc_crdt.png']
keywords: ["UnisonDB", "CRDT", "Replication", "Edge Computing", "ZeroMQ", "Go"]
categories: ["Distributed Systems", "Tutorials", "CRDT"]
---

<img src="/images/multi_dc_crdt.svg" alt="Multi DC CRDT" />

## Introduction: The Challenge of Distributed State Management

Imagine you're building a globally distributed application where users across different continents need to see consistent data think user presence status, live dashboards, or real-time collaboration features. Traditional databases force you to choose between consistency and availability, but what if there was a better way?

[**Conflict-free Replicated Data Types (CRDTs)**](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) offer a mathematical approach to distributed state management where conflicts are resolved automatically through well-defined merge operations. When combined with **edge notifications**, you get a powerful pattern: write anywhere, replicate everywhere, and get notified of changes in real-time.

In this post, we'll build a multi-datacenter system using UnisonDB that demonstrates:
-   Concurrent writes to multiple datacenters
-   Automatic conflict resolution using [CRDTs](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)
-   Real-time change notifications via [ZeroMQ](https://en.wikipedia.org/wiki/ZeroMQ)
-   [Eventual consistency](https://en.wikipedia.org/wiki/Eventual_consistency) across all nodes

## Architecture Overview

Our demo system consists of three UnisonDB nodes:

```
+---------------------------------------------------------------+
|                    Multi-DC CRDT Architecture                 |
+---------------------------------------------------------------+

        Writes                                    Writes
          |                                         |
          v                                         v
   +----------------+                       +----------------+
   |  Datacenter 1  |                       |  Datacenter 2  |
   |   (Primary)    |                       |   (Primary)    |
   |                |                       |                |
   |  HTTP: 8001    |                       |  HTTP: 8002    |
   |  gRPC: 4001    |                       |  gRPC: 4002    |
   +--------+-------+                       +--------+-------+
            |                                         |
            |               gRPC Replication          |
            +---------------------+-------------------+
                                  |
                                  v
                        +---------------------+
                        |      Replica        |
                        |     (Read-Only)     |
                        |                     |
                        |  HTTP: 8003         |
                        |  ZMQ dc1: 5555 ---> |----+
                        |  ZMQ dc2: 5556 ---> |----+  Watch API
                        +---------------------+    |  Notifications
                                                   |
                                                   v
                                         +--------------------+
                                         |    CRDT Client     |
                                         |   (Go / Node.js)   |
                                         |                    |
                                         |   Converged State  |
                                         +--------------------+
```

### Component Roles

| Component | Role | Namespace | HTTP Port | gRPC Port | ZMQ Ports |
|-----------|------|-----------|-----------|-----------|-----------|
| **DC1** | Primary (accepts writes) | `ad-campaign-dc1` | 8001 | 4001 | - |
| **DC2** | Primary (accepts writes) | `ad-campaign-dc2` | 8002 | 4002 | - |
| **Replica** | Read-only replica | `ad-campaign-dc1`, `ad-campaign-dc2` | 8003 | - | 5555, 5556 |

## Building and Running UnisonDB

### Prerequisites

```bash
# Ensure you have Go 1.21+ and CGO enabled
go version
# go version go1.21.0 or higher
```

### Step 1: Build UnisonDB

This needs Zero MQ Installed Make Sure You've have it Installed.
[Install ZeroMQ dependency ](https://zeromq.org/download/)

```bash 
# Clone the repository
git clone https://github.com/ankur-anand/unisondb.git
cd unisondb

# Build the binary (CGO required for RocksDB)
CGO_ENABLED=1 go build -tags zeromq ./cmd/unisondb
```

### Step 2: Start the Multi-DC Cluster

Open **three separate terminal windows** and run:

**Terminal 1: Start Datacenter 1**
```bash
./unisondb server -config ./cmd/examples/crdt-multi-dc/configs/dc1.toml
```

**Terminal 2: Start Datacenter 2**
```bash
./unisondb server -config ./cmd/examples/crdt-multi-dc/configs/dc2.toml
```

**Terminal 3: Start Replica**
```bash
./unisondb replica -config ./cmd/examples/crdt-multi-dc/configs/relayer.toml
```

You should see output indicating each node is ready:
```
INFO: HTTP server listening on :8001
INFO: gRPC server listening on :4001
INFO: Namespace 'ad-campaign-dc1' initialized
```

### Step 3: Start the CRDT Client

Open a **fourth terminal** to run the client that will observe CRDT state:

```bash
cd cmd/examples/golang-crdt-client
go run main.go
```

Expected output:
```
  Waiting for change notifications...

  Connecting to ZeroMQ ad-campaign-dc1: tcp://localhost:5555
  Connecting to ZeroMQ ad-campaign-dc2: tcp://localhost:5556
  ZeroMQ listener started for namespace: ad-campaign-dc1
  ZeroMQ listener started for namespace: ad-campaign-dc2
```

Your system is now ready!

## Understanding CRDTs: Two Types in Action

### 1. LWW-Register (Last-Write-Wins Register)

**Use Cases:** User profiles, configuration settings, feature flags

**How it works:**
- Each write includes a **timestamp** and **replica ID**
- Conflicts are resolved by choosing the write with the **latest timestamp**
- If timestamps are equal, the **lexicographically higher replica ID** wins

**Data Format:**
```json
{
  "value": "actual data",
  "timestamp": 1698765432000,
  "replica": "ad-campaign-dc1"
}
```

### 2. G-Counter (Grow-Only Counter)

**Use Cases:** Page views, API calls, distributed metrics (monotonically increasing)

**How it works:**
- Each replica maintains its own counter
- Merging takes the **maximum** count per replica
- Total value is the **sum** of all replica counters
- Can only increase (never decrease)

**Data Format:**
```json
{
  "replica": "ad-campaign-dc1",
  "count": 5
}
```

## Demo Scenarios with curl Examples

### Scenario 1: Basic LWW-Register Update

Let's update a user's status across two datacenters:

**Write "online" to DC1 (timestamp: 1698765432000)**
```bash
curl -X PUT "http://localhost:8001/api/v1/ad-campaign-dc1/kv/lww:user-status" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"value":"online","timestamp":1698765432000,"replica":"ad-campaign-dc1"}' | base64)'"}'
```

**Client Output:**
```
  Change notification received
   Topic: ad-campaign-dc1.kv
   Key: lww:user-status
   Operation: put

  Processing update: lww:user-status
  LWW-Register updated: lww:user-status
   Value: online
   Timestamp: 1698765432000
   Replica: ad-campaign-dc1

  CURRENT CRDT STATE

  LWW-Registers:
  lww:user-status:
    Value: online
    Timestamp: 1698765432000
    Replica: ad-campaign-dc1
```

Now write "away" to DC2 with a **newer timestamp**:

**Write "away" to DC2 (timestamp: 1698765433000)**
```bash
curl -X PUT "http://localhost:8002/api/v1/ad-campaign-dc2/kv/lww:user-status" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"value":"away","timestamp":1698765433000,"replica":"ad-campaign-dc2"}' | base64)'"}'
```

**Client Output:**
```
  Change notification received
   Topic: ad-campaign-dc2.kv
   Key: lww:user-status
   Operation: put

  Processing update: lww:user-status
  LWW-Register updated: lww:user-status
   Value: away
   Timestamp: 1698765433000
   Replica: ad-campaign-dc2

  CURRENT CRDT STATE

  LWW-Registers:
  lww:user-status:
    Value: away
    Timestamp: 1698765433000
    Replica: ad-campaign-dc2
```

**What happened?** The client automatically resolved the conflict! DC2's write won because it had a **newer timestamp** (1698765433000 > 1698765432000).

### Scenario 2: Concurrent Writes with Same Timestamp

What happens when two datacenters write at the exact same millisecond?

**Write to DC1:**
```bash
TIMESTAMP=$(date +%s)000
curl -X PUT "http://localhost:8001/api/v1/ad-campaign-dc1/kv/lww:config" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n "{\"value\":\"DC1 wins?\",\"timestamp\":$TIMESTAMP,\"replica\":\"ad-campaign-dc1\"}" | base64)'"}'
```

**Write to DC2 (same timestamp):**
```bash
curl -X PUT "http://localhost:8002/api/v1/ad-campaign-dc2/kv/lww:config" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n "{\"value\":\"DC2 wins!\",\"timestamp\":$TIMESTAMP,\"replica\":\"ad-campaign-dc2\"}" | base64)'"}'
```

**Result:** `ad-campaign-dc2` wins because lexicographically `"ad-campaign-dc2" > "ad-campaign-dc1"`. This ensures **deterministic conflict resolution** across all replicas.

### Scenario 3: Distributed Counter (G-Counter)

Let's track page views across two datacenters:

**DC1 serves 5 requests:**
```bash
curl -X PUT "http://localhost:8001/api/v1/ad-campaign-dc1/kv/counter:page-views" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"replica":"ad-campaign-dc1","count":5}' | base64)'"}'
```

**DC2 serves 3 requests:**
```bash
curl -X PUT "http://localhost:8002/api/v1/ad-campaign-dc2/kv/counter:page-views" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"replica":"ad-campaign-dc2","count":3}' | base64)'"}'
```

**Client Output:**
```
  CURRENT CRDT STATE

  G-Counters:
  counter:page-views:
    Replica Counts: {"ad-campaign-dc1":5,"ad-campaign-dc2":3}
    Total: 8
```

**Result:** Total = **8** (5 from DC1 + 3 from DC2). The counters from both datacenters are automatically merged!

### Scenario 4: Out-of-Order Delivery (Stale Write)

What if network delays cause an old write to arrive after a newer one?

**Write NEW value to DC1:**
```bash
curl -X PUT "http://localhost:8001/api/v1/ad-campaign-dc1/kv/lww:feature-flag" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"value":true,"timestamp":2000,"replica":"ad-campaign-dc1"}' | base64)'"}'
```

**Write OLD value to DC2 (stale):**
```bash
curl -X PUT "http://localhost:8002/api/v1/ad-campaign-dc2/kv/lww:feature-flag" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"value":false,"timestamp":1000,"replica":"ad-campaign-dc2"}' | base64)'"}'
```

**Client Output:**
```
  Processing update: lww:feature-flag
    LWW-Register ignored (stale): lww:feature-flag
   Incoming timestamp: 1000
   Current timestamp: 2000
```

**Result:** The stale write is **automatically ignored**. The CRDT logic ensures we never regress to an older state!

### Scenario 5: Multiple Counters Operating Independently

```bash
# Track different metrics
curl -X PUT "http://localhost:8001/api/v1/ad-campaign-dc1/kv/counter:api-calls" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"replica":"ad-campaign-dc1","count":100}' | base64)'"}'

curl -X PUT "http://localhost:8002/api/v1/ad-campaign-dc2/kv/counter:api-calls" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"replica":"ad-campaign-dc2","count":75}' | base64)'"}'

curl -X PUT "http://localhost:8001/api/v1/ad-campaign-dc1/kv/counter:db-queries" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"replica":"ad-campaign-dc1","count":250}' | base64)'"}'
```

**Client Output:**
```
  CURRENT CRDT STATE

  G-Counters:
  counter:api-calls:
    Replica Counts: {"ad-campaign-dc1":100,"ad-campaign-dc2":75}
    Total: 175

  counter:db-queries:
    Replica Counts: {"ad-campaign-dc1":250}
    Total: 250
```

Each counter operates independently with its own convergence!

## Reading Data from the Relayer

The relayer provides read-only access to both datacenter namespaces:

**Read from DC1 namespace:**
```bash
curl "http://localhost:8003/api/v1/ad-campaign-dc1/kv/lww:user-status" | jq
```

**Response:**
```json
{
  "value": "eyJ2YWx1ZSI6ImF3YXkiLCJ0aW1lc3RhbXAiOjE2OTg3NjU0MzMwMDAsInJlcGxpY2EiOiJhZC1jYW1wYWlnbi1kYzIifQ==",
  "found": true
}
```

**Decode the base64 value:**
```bash
echo "eyJ2YWx1ZSI6ImF3YXkiLCJ0aW1lc3RhbXAiOjE2OTg3NjU0MzMwMDAsInJlcGxpY2EiOiJhZC1jYW1wYWlnbi1kYzIifQ==" | base64 -d | jq
```

**Output:**
```json
{
  "value": "away",
  "timestamp": 1698765433000,
  "replica": "ad-campaign-dc2"
}
```

## How Conflict Resolution Works Under the Hood

### LWW-Register Algorithm

The conflict resolution logic in `lww_register.go:30-39`:

```go
func (r *LWWRegister) Update(value interface{}, timestamp int64, replica string) bool {
    // Rule 1: Accept if timestamp is newer
    if timestamp > r.Timestamp {
        r.Value = value
        r.Timestamp = timestamp
        r.Replica = replica
        return true
    }

    // Rule 2: If timestamps equal, use replica ID as tiebreaker
    if timestamp == r.Timestamp && replica > r.Replica {
        r.Value = value
        r.Replica = replica
        return true
    }

    // Rule 3: Reject stale updates
    return false
}
```

**Key Properties:**
-   **Commutative**: Order of updates doesn't matter
-   **Associative**: Grouping of updates doesn't matter
-   **Idempotent**: Applying the same update multiple times is safe
-   **Deterministic**: All replicas converge to the same value

### G-Counter Merge Algorithm

The merge logic in `g_counter.go`:

```go
func (c *GCounter) Merge(replica string, count int64) bool {
    current := c.Counts[replica]

    // Only accept higher counts (monotonic)
    if count > current {
        c.Counts[replica] = count
        return true
    }

    return false
}

func (c *GCounter) GetValue() int64 {
    total := int64(0)
    for _, count := range c.Counts {
        total += count
    }
    return total
}
```

**Key Properties:**
-   **Monotonic**: Values only increase
-   **Convergent**: All replicas reach the same total
-   **Partition-tolerant**: Works across network splits

## Real-World Use Cases

### 1. User Presence System
```bash
# User goes online in US datacenter
curl -X PUT "http://localhost:8001/api/v1/ad-campaign-dc1/kv/lww:user:alice:status" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n "{\"value\":\"online\",\"timestamp\":$(date +%s)000,\"replica\":\"us-east-1\"}" | base64)'"}'

# User goes away in EU datacenter (newer timestamp wins)
curl -X PUT "http://localhost:8002/api/v1/ad-campaign-dc2/kv/lww:user:alice:status" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n "{\"value\":\"away\",\"timestamp\":$(($(date +%s)+5))000,\"replica\":\"eu-west-1\"}" | base64)'"}'
```

All clients worldwide see the latest status in real-time!

### 2. Distributed Analytics
```bash
# Track impressions across regions
curl -X PUT "http://localhost:8001/api/v1/ad-campaign-dc1/kv/counter:campaign-123:impressions" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"replica":"us-east-1","count":1500}' | base64)'"}'

curl -X PUT "http://localhost:8002/api/v1/ad-campaign-dc2/kv/counter:campaign-123:impressions" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n '{"replica":"eu-west-1","count":2300}' | base64)'"}'

# Global total: 3800 impressions
```

### 3. Feature Flags
```bash
# Enable feature in production
curl -X PUT "http://localhost:8001/api/v1/ad-campaign-dc1/kv/lww:feature:new-ui" \
  -H "Content-Type: application/json" \
  -d '{"value": "'$(echo -n "{\"value\":true,\"timestamp\":$(date +%s)000,\"replica\":\"control-plane\"}" | base64)'"}'
```

Feature flag changes propagate globally within milliseconds!

### Try It Yourself

```bash
# Clone and run the example
git clone https://github.com/ankur-anand/unisondb.git
cd unisondb
CGO_ENABLED=1 go build -tags zeromq -o unisondb ./cmd/unisondb

# Start the demo
cd cmd/examples/crdt-multi-dc
```

**Watch the magic happen** as conflicts resolve themselves and state converges across datacenters!

---

## Additional Resources

- [UnisonDB GitHub Repository](https://github.com/ankur-anand/unisondb)
- [CRDT Research Papers](https://crdt.tech/)
- [ZeroMQ Guide](https://zguide.zeromq.org/)

Have questions or want to contribute? Open an issue on GitHub or join our community discussions!

