---
title: "Building a Corruption-Proof Write-Ahead Log in Go"
date: 2025-12-14
description: "Learn how to build a crash-safe Write-Ahead Log (WAL) in Go, and why CRC32 alone is not enough. We explore the durability layers UnisonDB uses to prevent corruption after crashes."
summary: "Learn how to build a crash-safe Write-Ahead Log (WAL) in Go, and why CRC32 alone is not enough. We explore the durability layers UnisonDB uses to prevent corruption after crashes."
keywords: ["Go", "Golang", "Write-Ahead Log", "WAL", "Database Engineering", "Data Reliability", "System Programming", "UnisonDB", "fsync", "mmap"]
images: ["/images/wal_fs.png"]
---

<img src="/images/wal_fs.svg" alt="Diagram of UnisonDB's corruption-proof WAL path" />

## The Problem

Every database promises durability. Write your data, get an acknowledgment, sleep well. But what happens between the [`write()`](https://man7.org/linux/man-pages/man2/write.2.html) syscall (or a memory-mapped store) and the moment electrons finally settle on persistent media?

There is a long, leaky pipeline — and every layer can betray you. A lot can go wrong:

1. **Power failure mid-write** — The system crashes while writing, so only part of your data reaches disk.
2. **Bit flips** — Hardware issues or random errors can silently change stored data.
3. **False success signals** — The operating system may report success even though data is still in memory.
4. **Filesystem limits** — Journaling keeps files intact, not your data’s meaning. WAL-on-WAL is not correctness — it’s wishful thinking.
5. **Torn writes** — A single 4KB write can span multiple sectors, and only some of them may commit.

>The purpose of a [Write-ahead logging](https://en.wikipedia.org/wiki/Write-ahead_logging) is not just to record writes in order, but to make correctness provable. After a crash, the database treats the log as evidence, replaying only records that can be proven complete and correct.


## The Stakes: Streaming Replication

In [UnisonDB](https://unisondb.io), the WAL serves as the **primary source for replication** in addition to its traditional role in recovery. Followers continuously read from the leader's WAL segments, applying records as they arrive.

```
┌─────────┐     WAL Segments      ┌──────────┐
│ Leader  │ ──────────────────▶   │ Follower │
│         │   (streaming read)    │          │
└─────────┘                       └──────────┘
```

This means:
- **Corruption propagates** - A bad record on the leader poisons followers

The WAL is our single source of truth. It had better be correct.

---

## The Record Format

Every record in our WAL has this structure:

```
┌─────────────────────────────────────────────────────────────────┐
│                         WAL Record                              │
├──────────┬──────────┬─────────────────────┬─────────────────────┤
│  CRC32   │  Length  │        Data         │       Trailer       │
│ 4 bytes  │ 4 bytes  │     N bytes         │      8 bytes        │
├──────────┴──────────┴─────────────────────┴─────────────────────┤
│                    Padded to 8-byte boundary                    │
└─────────────────────────────────────────────────────────────────┘
```

## Layer 1: CRC32 (Castagnoli)

```go
// CRC covers everything except itself
crc := crc32.Checksum(buf[4:], crc32.MakeTable(crc32.Castagnoli))
binary.LittleEndian.PutUint32(buf[0:4], crc)
```

CRC catches:
- Random bit flips
- Sector read errors
- Truncated data

CRC doesn't catch:
- **The "Wild Read" Problem** - If the length header is corrupt (e.g., claiming 1GB size), a CRC check forces you to read that garbage before failing.

---

## Layer 2: The Trailer Canary

This is where `0xDEADBEEFFEEDFACE` enters the story.

```go
const recordTrailer uint64 = 0xDEADBEEFFEEDFACE
```

Every record ends with this 8-byte magic value. During recovery, we verify it.

> A trailer marker makes correctness provable by separating “written” from “finished.”

```go
func (seg *Segment) Read(offset int64) ([]byte, RecordPosition, error) {
	...
	// validating  the trailer before reading data
	// we are ensuring no oob access even if length is corrupted.
	trailerOffset := offset + recordHeaderSize + dataSize
	end := trailerOffset + recordTrailerMarkerSize
	if end > seg.mmapSize {
		return nil, NilRecordPosition, ErrIncompleteChunk
	}

	...
	return data, next, nil
}

```

### Why the Trailer?

Consider this crash scenario:

```
Write sequence:
1. Write CRC (4 bytes)     - persisted
2. Write Length (4 bytes)  - persisted
3. Write Data (N bytes)    - CRASH - only 50% written
4. Write Trailer           - never reached
```

Without the trailer, recovery sees:
- Valid CRC (for partial data? unlikely but possible)
- Valid length field
- Garbage data

With the trailer, recovery sees:
- Trailer missing or wrong → **incomplete write, ignore this record**

This pattern was inspired by a real etcd bug ([#6191](https://github.com/etcd-io/etcd/issues/6191)) where torn writes corrupted the WAL. The trailer acts as a "commit marker" for each record.

### Why This Specific Value?

[`0xDEADBEEFFEEDFACE`](https://en.wikipedia.org/wiki/Deadbeef) is:
- Unlikely to appear in real data (it's a known debug pattern)
- Easy to spot in hex dumps
---

## Layer 3: WAL Alignment: What It Guarantees (and What It Doesn’t)

UnisonDB aligns every WAL record to an **8-byte boundary**. We do this specifically to ensure **WAL parsing and recovery are safe**, not to force atomic disk writes.

The primary danger we are avoiding is a **torn or corrupted header**. While CRCs catch torn payloads, a bad header is structural damage that can confuse the parser.

### The Real Problem: Torn Headers, Not Torn Payloads
```txt
[ CRC32 (4 bytes) | Length (4 bytes) ]
```

This 8-byte header controls how the rest of the log is parsed:
* how many bytes to read
* where the next record begins
* when recovery should stop

If this header is partially written or misinterpreted, recovery can:
* read past the end of the file
* allocate unbounded memory
* crash before corruption is detected

That is catastrophic failure. So the core question is:

> How do we ensure WAL control metadata is either read correctly or rejected—never misinterpreted?

The Invariant Enforced.

> Every WAL record starts at an offset divisible by 8.

This is implemented directly in the write path:

```go
func alignUp(n int64) int64 {
    return (n + 7) & ^7
}
```

Every record’s total size (header + payload + trailer) is rounded up using alignUp, and the write offset is advanced by that aligned size. As a result:

```go
writeOffset % 8 == 0
```

**What 8-Byte Alignment Guarantees**

1. **Headers Never Straddle Physical Boundaries**
   Since 512 and 4096 are multiples of 8, an 8-byte header starting at an 8-byte offset **cannot** cross a sector or page boundary. It effectively lives inside a single atomic write unit. This makes "torn headers" mathematically impossible.

2. **Corruption is Detectable, Not Fatal**
   Without alignment, a torn header introduces "phantom" valid records—random bytes interpreted as a massive length field, causing OOMs or wild seeks. With alignment, a header is either correct or obviously invalid (failed CRC). We never interpret garbage metadata as valid control instructions.

3. **Safe Termination**
   Recovery becomes a simple state machine: read until the first error, then stop. Because the *structure* of the log remains intact, the reader never crashes while trying to determine where the log ends. It makes recovery deterministic.

### What Alignment Does Not Guarantee

8-byte alignment does not:
* guarantee atomic disk writes
* make payload writes safe
* replace fsync / msync
* eliminate the need for trailers or CRCs

Alignment protects interpretation, not persistence. Without alignment, a corrupted length field can crash recovery before corruption is detected. With alignment, corruption is contained:
* recovery either proceeds correctly
* or stops safely

> Alignment doesn't prevent failure, but it downgrades a catastrophic crash into a handleable error.

## Each layer protects a different invariant

| Layer     | Protects Against                                          |
|-----------|-----------------------------------------------------------|
| Alignment | Partially written headers and invalid length fields       |
| CRC32    | Data corruption from bit flips or torn payload reads       |
| Trailer  | Records that were not fully written before a crash         |

Together, they ensure that:
* We never read beyond intended bounds
* We can detect corruption
* We can safely stop at the first incomplete record

---

## Layer 4: Directory Sync

### Why Directory Sync?

On Linux, calling [fsync()](https://man7.org/linux/man-pages/man2/fsync.2.html) on a file only guarantees that the file’s data and metadata are durable. It does not guarantee that the filesystem has persisted the directory entry that makes the file visible.

As the Linux manual explicitly warns:

> Calling fsync() does not necessarily ensure that the entry in the directory containing the file has also reached disk. For that, an explicit fsync() on a file descriptor for the directory is also needed.

If the system crashes at the wrong moment:
* The WAL segment was written
* The file itself was fsync’d
* But the directory update was never persisted

Consider this crash scenario without directory sync:

```
1. Create new segment file     - file exists in memory
2. Write segment header        - data in page cache
3. fsync segment file          - data on disk
4. [CRASH]
5. Recovery: "where's segment 3?" - directory entry never persisted!
```

After recovery, the file simply does not exist.

This is especially dangerous for systems with automatic WAL segment rotation, where new segment files are created continuously as the log grows. Losing a segment file during rotation means losing part of the log — even though all writes were “successful”

With directory sync:

```
1. Create new segment file     - file exists in memory
2. Write segment header        - data in page cache
3. fsync segment file          - data on disk
4. fsync directory             - directory entry on disk
5. [CRASH]
6. Recovery: segment 3 exists and is valid
```

```go
type DirectorySyncer struct {
    dirFd *os.File
}

func (d *DirectorySyncer) Sync() error {
    return d.dirFd.Sync()  // fsync on directory fd
}
```

By calling fsync() on the directory:

* The filesystem is forced to persist directory entries
* Newly created segment files are guaranteed to exist after a crash
* Segment rotation becomes crash-safe

When we use directory sync

We only need this at structural boundaries, such as:
* Creating a new WAL segment
* Finalizing segment rotation
* Renaming or deleting segment files

This keeps the hot write path fast while ensuring the WAL layout itself is durable.

---

## Layer 5: Conservative Recovery

Our recovery philosophy: **when in doubt, stop**.

```go
func (w *WALog) recoverSegments() error {
    segments := listSegmentFiles(w.dir)
    sort.Sort(segments)  // By segment ID

    for _, seg := range segments {
        if err := seg.recover(); err != nil {
            // Don't try to be clever - stop at first corruption
            w.logger.Error("segment corrupt, truncating",
                "segment", seg.ID,
                "error", err)
            return seg.truncateAtLastGoodRecord()
        }
    }
    return nil
}
```

We don't try to skip corrupted records and continue. Why?

1. **Ordering matters** - A gap in the log might hide a critical transaction
2. **Corruption often spreads** - If one record is bad, neighbors might be too
3. **Raft requires contiguity** - Log indices must be sequential

---

## Write

```go
func (seg *Segment) Write(data []byte, logIndex uint64) (RecordPosition, error) {
	if seg.closed.Load() || seg.state.Load() != StateOpen {
		return NilRecordPosition, ErrClosed
	}

	seg.writeMu.Lock()
	defer seg.writeMu.Unlock()

	flags := binary.LittleEndian.Uint32(seg.mmapData[40:44])
	if IsSealed(flags) {
		return NilRecordPosition, ErrSegmentSealed
	}

	offset := seg.writeOffset.Load()

	seg.writeFirstIndexEntry(logIndex)

	headerSize := int64(recordHeaderSize)
	dataSize := int64(len(data))
	trailerSize := int64(recordTrailerMarkerSize)
	rawSize := headerSize + dataSize + trailerSize
	entrySize := alignUp(rawSize)

	if offset+entrySize > seg.mmapSize {
		return NilRecordPosition, errors.New("write exceeds Segment size")
	}

	binary.LittleEndian.PutUint32(seg.header[4:8], uint32(len(data)))
	sum := crc32Checksum(seg.header[4:], data)
	binary.LittleEndian.PutUint32(seg.header[:4], sum)

	copy(seg.mmapData[offset:], seg.header[:])
	copy(seg.mmapData[offset+recordHeaderSize:], data)

	canaryOffset := offset + headerSize + dataSize
	copy(seg.mmapData[canaryOffset:], trailerMarker)

	paddingStart := offset + rawSize
	paddingEnd := offset + entrySize
	// ensuring alignment to 8 bytes
	for i := paddingStart; i < paddingEnd; i++ {
		seg.mmapData[i] = 0
	}

	newOffset := offset + entrySize
	seg.writeOffset.Store(newOffset)

	binary.LittleEndian.PutUint32(seg.mmapData[24:32], uint32(newOffset))
	prevCount := binary.LittleEndian.Uint64(seg.mmapData[32:40])
	binary.LittleEndian.PutUint64(seg.mmapData[32:40], prevCount+1)
	binary.LittleEndian.PutUint64(seg.mmapData[16:24], uint64(time.Now().UnixNano()))

	crc := crc32.Checksum(seg.mmapData[0:56], crcTable)
	binary.LittleEndian.PutUint32(seg.mmapData[56:60], crc)

	seg.appendIndexEntry(offset, uint32(len(data)))

	// MSync if option is set
	if seg.syncOption == MsyncOnWrite {
		if err := seg.mmapData.Flush(); err != nil {
			return NilRecordPosition, fmt.Errorf("mmap flush error after write: %w", err)
		}
	}

	return RecordPosition{
		SegmentID: seg.id,
		Offset:    offset,
	}, nil
}
```

## Read

```go
func (seg *Segment) Read(offset int64) ([]byte, RecordPosition, error) {
	if seg.closed.Load() {
		return nil, NilRecordPosition, ErrClosed
	}
	if offset+recordHeaderSize > seg.mmapSize {
		return nil, NilRecordPosition, io.EOF
	}

	header := seg.mmapData[offset : offset+recordHeaderSize]
	length := binary.LittleEndian.Uint32(header[4:8])
	dataSize := int64(length)

	rawSize := int64(recordHeaderSize) + dataSize + recordTrailerMarkerSize
	entrySize := alignUp(rawSize)

	if length > uint32(seg.WriteOffset()-offset-recordHeaderSize) {
		return nil, NilRecordPosition, ErrCorruptHeader
	}

	if offset+entrySize > seg.WriteOffset() {
		return nil, NilRecordPosition, io.EOF
	}

	// validating  the trailer before reading data
	// we are ensuring no oob access even if length is corrupted.
	trailerOffset := offset + recordHeaderSize + dataSize
	end := trailerOffset + recordTrailerMarkerSize
	if end > seg.mmapSize {
		return nil, NilRecordPosition, ErrIncompleteChunk
	}

	// previously we were doing byte which did show in pprof as runtime.memequal
	// switching to uint64 comparison removed it altogether.
	word := binary.LittleEndian.Uint64(seg.mmapData[trailerOffset:end])
	// validating trailer marker to detect torn/incomplete writes.
	if word != trailerWord {
		return nil, NilRecordPosition, ErrIncompleteChunk
	}

	data := seg.mmapData[offset+recordHeaderSize : offset+recordHeaderSize+dataSize]

	// sealed segments are immutable and may have been recovered
	// from disk after a crash or shutdown. CRC validation ensures that data
	// persisted to disk is still intact and wasn't partially written or corrupted.
	// for active segment, we do one validation at start if not sealed, else it's in the
	// same process memory, so having corruption of the same byte is very unlikely, until
	// done from some external forces.
	// doing this in the hot-path is CPU intensive and most of the read are towards the tail.
	if seg.isSealed.Load() && !seg.inMemorySealed.Load() {
		savedSum := binary.LittleEndian.Uint32(header[:4])
		computedSum := crc32Checksum(header[4:], data)
		if savedSum != computedSum {
			return nil, NilRecordPosition, ErrInvalidCRC
		}
	}

	next := RecordPosition{
		SegmentID: seg.id,
		Offset:    offset + entrySize,
	}

	return data, next, nil
}
```
---

## Lessons Learned

1. **CRC alone isn't enough** - You need to detect incomplete writes too
2. **fsync isn't enough** - You need directory sync for metadata
3. **mmap is tricky** - msync semantics vary by OS; always fsync the fd
4. **Alignment matters** - 8-byte alignment reduces torn write risk
5. **Be conservative in recovery** - Stop at first corruption, don't guess
6. **Test failure modes** - If you haven't tested it, it doesn't work

---

## Conclusion

A WAL is deceptively simple: append bytes, sync, done. The complexity hides in the failure modes:

- What if the write is torn?
- What if the sync lies?
- What if a bit flips?
- What if the directory entry isn't persisted?

Each layer of our design—CRC, trailer, alignment, header checksum, directory sync, conservative recovery—addresses a specific failure we've either experienced or studied in others' post-mortems.

The `0xDEADBEEFFEEDFACE` trailer is a perfect metaphor: it looks like a joke, but it's deadly serious. In distributed systems, the boundary between working and broken is measured in these small details.

Build your WAL like you don't trust anything—because you shouldn't.

Github: [https://github.com/ankur-anand/unisondb](https://github.com/ankur-anand/unisondb)

---
## Appendix: Some Discussions
1. https://stackoverflow.com/questions/2009063/are-disk-sector-writes-atomic
2. https://en.wikipedia.org/wiki/Solid-state_drive
3. [HN Discussion: Every HDD since the 1980s has guaranteed atomic sector writes](https://news.ycombinator.com/item?id=36570264) - LMDB Author has posted some interesting insights on this.
---

*The UnisonDB WAL is open source. Star us on GitHub if this saved you from learning these lessons the hard way.*
