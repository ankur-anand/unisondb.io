---
title: "Breaking Key-Value Size Limits: Linked List WALs for Atomic Large Writes"
date: 2026-01-23
description: "Key-Value size limits (512KB/1.5MB) are essential for cluster health. Learn how UnisonDB bypasses these constraints using a Linked List WAL without breaking atomicity."
summary: "In this post, we explore how UnisonDB uses standard size limits to protect itself, while providing a way to write large Key-Value, Wide-Column, and LOB data without sacrificing the atomicity promise."
keywords: ["Go", "Golang", "Write-Ahead Log", "WAL", "Database Engineering", "Data Reliability", "System Programming", "UnisonDB", "fsync", "mmap", "Multi-modal"]
slug: "breaking-kv-size-limits-linked-list-wal"
images: ["/images/wal_linked_list.png"]
---

<img src="/images/wal_linked_list.svg" alt="Diagram of UnisonDB's corruption-proof WAL path" />

## The "Hard Wall" of Distributed Systems

The majority of distributed Key-Value systems have some kind of limit. This limit exists for a purpose: it prevents a single request from overwhelming memory or stalling replication.

Whether it's the 512KB cap in [Consul](https://developer.hashicorp.com/consul/docs/automate/kv) or the 1.5MB default in [etcd](https://etcd.io/docs/v3.6/dev-guide/limit/), these boundaries are a survival mechanism. In a distributed cluster, every byte you write has to be replicated via protocols like Raft. If a single record is too large, it creates "head-of-line blocking"—the entire replication pipeline slows down just to move one massive object, potentially causing heartbeats to fail and nodes to drop out of the cluster.

At UnisonDB, we respect these same limits to protect our own system health. We need to be even more cautious about this, as we are not just doing Raft replication for writes. We also have high-fanout ISR (in-sync-replica) based edge replicas, meaning a single write can propagate to many more nodes. 

## Why ISR Edge Replication Changes the Stakes

In our environment, the "Hard Wall" isn't just about protecting memory or Raft heartbeats within a small core cluster. It is about protecting the replication integrity and lag across a vast edge network.

* **Heartbeat Fragility in Edge Environments**: Edge networks often have variable latency and less reliable connections. If replication takes too long because of oversized records, the system might falsely flag an edge node as "out of sync," triggering expensive and unnecessary full re-syncs, wasting bandwidth and compute.

* **Memory Pressure on Constrained Edge Nodes**: Unlike robust core cluster nodes, edge replicas frequently run on more resource-constrained hardware. Pushing a 20MB block in a single request could easily cause an Out Of Memory (OOM) event on a smaller edge instance, leading to outages at the edge.


But even with these constraints, we also understand that the need for large Key-Value storage hasn't gone away—it has actually intensified. As an Edge-replicated, general-purpose Multi-Modal database, we see this constantly. Whether it's a massive JSON configuration or high-dimensional vectors for AI use cases, modern data frequently pushes past those old boundaries.

This size pressure usually shows up in two ways:

1. **Batch Writes**: A single transaction involving multiple Key-Value pairs that, when grouped together, exceed the 1MB limit.
2. **Wide-Column/Row Updates**: A single row containing hundreds of columns where the aggregate size of the update blows past the ceiling.

In both cases, the user will expects the same ironclad KV guarantees they get with a tiny 1KB write. You shouldn't have to sacrifice Atomicity just because your data model is complex.

## Why Manual Chunking Fails

When engineers hit a size limit, the instinctive reaction is to "chunk" the data by splitting a 10MB write into ten separate 1MB requests. This is where things get dangerous. Without a specialized architecture, you lose the atomicity promise. If your connection drops after chunk seven, the database is left in a "zombie" state. You have a partial update that is neither the old version nor the new one. In a real database, the rule is absolute: it must be all or nothing.

## Unisondb Solution: A WAL That Remembers Its Past

To solve this at UnisonDB, **we stopped looking at the Write-Ahead Log (WAL) just as a flat, sequential file**. Instead, we treated it as a backward-linked list.

By adding a simple breadcrumb—a PrevTxnWalIndex—to every WAL record that are part of the same transaction, each chunk of data points back to the one that came before it. This allows us to stitch a single, massive transaction together across multiple physical writes without ever sending a request that exceeds the safety limit.

## The Flow: BEGIN, PREPARE, COMMIT

This logic is the backbone of how we handle large multi-modal data. The lifecycle of a transaction in our dbkernel looks like this:

1. **BEGIN**: We write an anchor record to the WAL. This initializes the transaction and generates a unique ID.

2. **PREPARE**: As you stream your data chunks, each one links back to the previous disk offset. Even if you send 50 chunks to stay under the limit, the database knows they belong to one chain.

3. **COMMIT**: This is the atomic switch. The final record acts as the seal.

> Nothing becomes visible to you the user until that COMMIT record is successfully flushed to disk. If the stream breaks halfway through, the database simply ignores those dangling fragments during the next recovery scan.


## Seeing it in the Code

The engine needs to know exactly where the previous piece of the puzzle lives on disk. We use physical disk offsets to create this chain.

```go
type LogRecord struct {
    LSN             uint64           
    TxnID           []byte
    // BEGIN, PREPARE, or COMMIT           
    TxnState        TransactionState
    // The link back to the previous record's offset
    PrevTxnWalIndex []byte
     // Payload: KV, Wide-Column, or LOB chunk          
    Data            []byte          
}
```

Here is how this looks inside the UnisonDB transaction engine. Notice how we track the prevOffset to build the link on the fly. In the AppendKVTxn function, we take the current prevOffset and bake it into the new log record before appending it to the WAL.

```go
// AppendKVTxn appends a key and value to the WAL as part of a transaction.
func (t *Txn) AppendKVTxn(key []byte, value []byte) error {
   // ..........
    record := logcodec.LogRecord{
        LSN:             index,
        TxnID:           t.txnID,
        TxnState:        logrecord.TransactionStatePrepare,
        // The Link: This points to the BEGIN record or the previous chunk
        PrevTxnWalIndex: t.prevOffset.Encode(), 
        Entries:         [][]byte{kvEncoded},
    }

    // Write to WAL and update the pointer for the next chunk
    encoded := record.FBEncode(len(kvEncoded) + 128)
    offset, err := t.engine.walIO.Append(encoded, index)
    if err != nil {
        t.err = err
        return err
    }

    t.prevOffset = offset
    t.valuesCount++
    return nil
}
```

When `commit` happens, the Commit function writes the final link. Only after the WAL confirms the commit do we flush the data to the in-memory MemTable.

```go
func (t *Txn) Commit() error {
    // ... ...
    
    // Write the final Commit record to the WAL
    record := logcodec.LogRecord{
        TxnState:        logrecord.TransactionStateCommit,
        TxnID:           t.txnID,
        PrevTxnWalIndex: t.prevOffset.Encode(),
        CRC32Checksum:   t.checksum,
    }

    offset, err := t.engine.walIO.Append(record.FBEncode(128), index)
    if err != nil {
        return err
    }

    // Now that the WAL is safe, make the data visible in memtable
    return t.memWriteFull()
}
```

## The Recovery Walk

When UnisonDB reboots after a crash, the recovery process starts from the last known checkpoint. As the engine scans the log forward from that point, it looks specifically for COMMIT records. Because our transaction chunks are chained together using physical disk offsets, the engine has a clear map to follow.

When a commit record is encountered, the engine uses the PrevTxnWalIndex to walk the chain of that specific transaction. It jumps from the commit record to the previous data chunk, and then to the one before that, continuing until it reaches the BEGIN record. This allows the engine to gather all the related pieces of a large Key-Value pair, Wide-Column row, or LOB without having to inspect unrelated transaction data that might be sitting in between those chunks.

### Reconstructing the Value

The GetTransactionRecords function is what performs this backward walk. It takes the offset of the commit record and follows the trail until the chain is complete.

```go
func (w *WalIO) GetTransactionRecords(startOffset *Offset) ([]*logrecord.LogRecord, error) {
    if startOffset == nil {
       return nil, nil
    }

    var records []*logrecord.LogRecord
    nextOffset := startOffset

    for {
       walEntry, err := w.Read(nextOffset)
       if err != nil {
          return nil, fmt.Errorf("failed to read WAL: %w", err)
       }

       payload, err := w.DecodeRecord(walEntry)
       if err != nil || len(payload) == 0 {
          break
       }

       record := logrecord.GetRootAsLogRecord(payload, 0)
       records = append(records, record)

       // If there is no previous pointer, we have reached the BEGIN record
       if record.PrevTxnWalIndexLength() == 0 {
          break
       }

       nextOffset = DecodeOffset(record.PrevTxnWalIndexBytes())
    }

    // replayed in the correct chronological order
    slices.Reverse(records)
    return records, nil
}
```

By chaining the records this way, we ensure that a large value is only rebuilt if the final commit record exists. If the engine finds chunks that don't lead to a commit, it simply ignores them. This keeps the data consistent and ensures that the all or nothing promise is maintained for every data model we support.

## Conclusion

Building a database is often a game of trade-offs. You want system stability, but you also want to support modern, heavy workloads like AI and complex multi-modal schemas.

By treating the Write-Ahead Log as a linked list, we found a way to have both. We keep our network requests small and safe, but we allow our data to be as large as it needs to be.

Whether it is a simple Key-Value pair, a massive Wide-Column row, or a Large Object, the atomicity promise remains unbroken.

---

If you found this article helpful, or if you're interested in the future of edge-replicated data, we’d love your support. 

You can check out the source code, contribute, or just give us a star on GitHub: [github.com/ankur-anand/unisondb](https://github.com/ankur-anand/unisondb)
