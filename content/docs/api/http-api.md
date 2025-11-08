---
title: "UnisonDB HTTP API Reference – REST Endpoints and Usage Guide"
description: "Comprehensive UnisonDB HTTP API reference — learn how to interact with UnisonDB’s REST endpoints for data operations, replication control, and namespace management in edge and AI-driven deployments."
images: ['/images/http_api_overview.png']
linkTitle: "HTTP API Reference"
keywords: [
  "UnisonDB API",
  "UnisonDB HTTP API",
  "UnisonDB REST API",
  "log-native database API",
  "edge computing API",
  "AI database API",
  "database replication API",
  "namespace management API",
  "real-time database API",
  "Go database REST API"
]
---

# HTTP API Reference

<img src="/images/http_api_overview.svg" alt="UnisonDB HTTP API" style="max-width: 100%; height: auto;" />

Complete reference for UnisonDB's [HTTP](https://en.wikipedia.org/wiki/HTTP) [REST](https://en.wikipedia.org/wiki/REST) API.

## Base URL

```
http://localhost:4000/api/v1/{namespace}
```

## Data Encoding

All binary values must be [base64-encoded](https://en.wikipedia.org/wiki/Base64):

```bash
# Encode a value
echo -n "hello world" | base64
# Output: aGVsbG8gd29ybGQ=

# Use in request
curl -X PUT http://localhost:4000/api/v1/default/kv/greeting \
  -d '{"value":"aGVsbG8gd29ybGQ="}'
```

## Key-Value Operations

### Put KV

Store a key-value pair.

**Request**:
```bash
PUT /api/v1/{namespace}/kv/{key}
Content-Type: application/json

{
  "value": "base64-encoded-value"
}
```

**Example**:
```bash
curl -X PUT http://localhost:4000/api/v1/default/kv/user:123 \
  -H "Content-Type: application/json" \
  -d '{
    "value": "eyJuYW1lIjoiSm9obiIsImFnZSI6MzB9"
  }'
```

**Response** (200 OK):
```json
{
  "success": true
}
```

**Errors**:
- `400 Bad Request`: Invalid base64 encoding
- `404 Not Found`: Namespace not found
- `500 Internal Server Error`: Engine error

---

### Get KV

Retrieve a value by key.

**Request**:
```bash
GET /api/v1/{namespace}/kv/{key}
```

**Example**:
```bash
curl http://localhost:4000/api/v1/default/kv/user:123
```

**Response** (200 OK):
```json
{
  "value": "eyJuYW1lIjoiSm9obiIsImFnZSI6MzB9",
  "found": true
}
```

**Response** (404 Not Found):
```json
{
  "value": "",
  "found": false
}
```

---

### Delete KV

Delete a key.

**Request**:
```bash
DELETE /api/v1/{namespace}/kv/{key}
```

**Example**:
```bash
curl -X DELETE http://localhost:4000/api/v1/default/kv/user:123
```

**Response** (200 OK):
```json
{
  "success": true
}
```

---

### Batch KV Operations

Perform multiple operations in one request.

#### Batch Put

**Request**:
```bash
POST /api/v1/{namespace}/kv/batch
Content-Type: application/json

{
  "operation": "put",
  "items": [
    {"key": "key1", "value": "dmFsdWUx"},
    {"key": "key2", "value": "dmFsdWUy"}
  ]
}
```

**Example**:
```bash
curl -X POST http://localhost:4000/api/v1/default/kv/batch \
  -d '{
    "operation": "put",
    "items": [
      {"key": "user:1", "value": "dXNlcjE="},
      {"key": "user:2", "value": "dXNlcjI="}
    ]
  }'
```

**Response** (200 OK):
```json
{
  "success": true,
  "processed": 2
}
```

#### Batch Delete

**Request**:
```bash
POST /api/v1/{namespace}/kv/batch
Content-Type: application/json

{
  "operation": "delete",
  "keys": ["key1", "key2"]
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "processed": 2
}
```

---

## Wide-Column Operations

### Put Row

Store a row with multiple columns.

**Request**:
```bash
PUT /api/v1/{namespace}/row/{rowKey}
Content-Type: application/json

{
  "columns": {
    "column1": "base64-value1",
    "column2": "base64-value2"
  }
}
```

**Example**:
```bash
curl -X PUT http://localhost:4000/api/v1/default/row/user:john \
  -d '{
    "columns": {
      "name": "Sm9obiBEb2U=",
      "email": "am9obkBleGFtcGxlLmNvbQ==",
      "age": "MzA="
    }
  }'
```

**Response** (200 OK):
```json
{
  "success": true
}
```

---

### Get Row

Retrieve all columns for a row.

**Request**:
```bash
GET /api/v1/{namespace}/row/{rowKey}
```

**Example**:
```bash
curl http://localhost:4000/api/v1/default/row/user:john
```

**Response** (200 OK):
```json
{
  "rowKey": "user:john",
  "columns": {
    "name": "Sm9obiBEb2U=",
    "email": "am9obkBleGFtcGxlLmNvbQ==",
    "age": "MzA="
  },
  "found": true
}
```

---

### Get Row Columns

Retrieve specific columns only.

**Request**:
```bash
GET /api/v1/{namespace}/row/{rowKey}?columns=col1,col2
```

**Example**:
```bash
curl "http://localhost:4000/api/v1/default/row/user:john?columns=name,email"
```

**Response** (200 OK):
```json
{
  "rowKey": "user:john",
  "columns": {
    "name": "Sm9obiBEb2U=",
    "email": "am9obkBleGFtcGxlLmNvbQ=="
  },
  "found": true
}
```

---

### Delete Row

Delete an entire row.

**Request**:
```bash
DELETE /api/v1/{namespace}/row/{rowKey}
```

**Example**:
```bash
curl -X DELETE http://localhost:4000/api/v1/default/row/user:john
```

**Response** (200 OK):
```json
{
  "success": true
}
```

---

### Delete Row Columns

Delete specific columns from a row.

**Request**:
```bash
DELETE /api/v1/{namespace}/row/{rowKey}/columns?columns=col1,col2
```

**Example**:
```bash
curl -X DELETE "http://localhost:4000/api/v1/default/row/user:john/columns?columns=age,city"
```

**Response** (200 OK):
```json
{
  "success": true
}
```

---

### Batch Row Operations

#### Batch Put Rows

**Request**:
```bash
POST /api/v1/{namespace}/row/batch
Content-Type: application/json

{
  "operation": "put",
  "rows": [
    {
      "rowKey": "user:1",
      "columns": {
        "name": "QWxpY2U=",
        "email": "YWxpY2VAZXhhbXBsZS5jb20="
      }
    }
  ]
}
```

**Response** (200 OK):
```json
{
  "success": true,
  "processed": 1
}
```

---

## Large Object (LOB) Operations

### Put LOB

Upload a large binary object. Check Transaction API.

---

### Get LOB

Download a large binary object.

**Request**:
```bash
GET /api/v1/{namespace}/lob?key={key}
```

**Example**:
```bash
curl "http://localhost:4000/api/v1/default/lob?key=file:doc.pdf" \
  --output document.pdf
```

**Response**: Binary data stream

---

## Transaction Operations

Transactions allow atomic operations across multiple keys.

### Transaction Lifecycle

```
1. BEGIN    → Get transaction ID
2. APPEND   → Add operations (multiple times)
3. COMMIT   → Apply atomically
   OR
3. ABORT    → Cancel transaction
```

### Begin Transaction

Start a new transaction.

**Request**:
```bash
POST /api/v1/{namespace}/tx/begin
Content-Type: application/json

{
  "operation": "put",
  "entryType": "kv"
}
```

**Parameters**:
- `operation`: `"put"`, `"update"`, or `"delete"`
- `entryType`: `"kv"`, `"row"`, or `"lob"`

**Example**:
```bash
curl -X POST http://localhost:4000/api/v1/default/tx/begin \
  -d '{
    "operation": "put",
    "entryType": "kv"
  }'
```

**Response** (200 OK):
```json
{
  "txnId": "2a3b4c5d6e7f8g9h0i1j2k3l4m5n6o7p",
  "success": true
}
```

**Save the `txnId`** - you'll need it for subsequent requests!

---

### Append KV to Transaction

Add a key-value operation to the transaction.

**Request**:
```bash
POST /api/v1/{namespace}/tx/{txnId}/kv
Content-Type: application/json

{
  "key": "mykey",
  "value": "bXl2YWx1ZQ=="
}
```

**Example**:
```bash
# Use the txnId from BEGIN response
curl -X POST http://localhost:4000/api/v1/default/tx/2a3b4c.../kv \
  -d '{
    "key": "account:alice",
    "value": "MTAwMA=="
  }'
```

**Response** (200 OK):
```json
{
  "success": true
}
```

**Call this endpoint multiple times** to add multiple operations to the same transaction.

---

### Append Row to Transaction

Add a row operation to the transaction.

**Request**:
```bash
POST /api/v1/{namespace}/tx/{txnId}/row
Content-Type: application/json

{
  "rowKey": "user:1",
  "columns": {
    "name": "QWxpY2U=",
    "status": "YWN0aXZl"
  }
}
```

**Example**:
```bash
curl -X POST http://localhost:4000/api/v1/default/tx/{txnId}/row \
  -d '{
    "rowKey": "user:charlie",
    "columns": {
      "name": "Q2hhcmxpZQ==",
      "email": "Y2hhcmxpZUBleGFtcGxlLmNvbQ=="
    }
  }'
```

**Response** (200 OK):
```json
{
  "success": true
}
```

---

### Append LOB to Transaction

Add a large object to the transaction.

**Request**:
```bash
POST /api/v1/{namespace}/tx/{txnId}/lob?key={key}
Content-Type: application/octet-stream

<binary data>
```

**Example**:
```bash
curl -X POST "http://localhost:4000/api/v1/default/tx/{txnId}/lob?key=file:backup.tar.gz" \
  --data-binary @backup.tar.gz
```

**Response** (200 OK):
```json
{
  "success": true
}
```

---

### Commit Transaction

Apply all operations atomically.

**Request**:
```bash
POST /api/v1/{namespace}/tx/{txnId}/commit
```

**Example**:
```bash
curl -X POST http://localhost:4000/api/v1/default/tx/2a3b4c.../commit
```

**Response** (200 OK):
```json
{
  "success": true
}
```

After commit:
- All operations are applied atomically
- Transaction ID is no longer valid
- Data is durable and replicated

---

### Abort Transaction

Cancel the transaction without applying changes.

**Request**:
```bash
POST /api/v1/{namespace}/tx/{txnId}/abort
```

**Example**:
```bash
curl -X POST http://localhost:4000/api/v1/default/tx/2a3b4c.../abort
```

**Response** (200 OK):
```json
{
  "success": true
}
```

After abort:
- No operations are applied
- Transaction ID is no longer valid

---

## Metadata Operations

### Get Current Offset

Get the current WAL position.

**Request**:
```bash
GET /api/v1/{namespace}/offset
```

**Example**:
```bash
curl http://localhost:4000/api/v1/default/offset
```

**Response** (200 OK):
```json
{
  "namespace": "default",
  "segmentId": 5,
  "offset": 12345
}
```

---

### Get Engine Statistics

Get engine performance statistics.

**Request**:
```bash
GET /api/v1/{namespace}/stats
```

**Example**:
```bash
curl http://localhost:4000/api/v1/default/stats
```

**Response** (200 OK):
```json
{
  "namespace": "default",
  "opsReceived": 15234,
  "opsFlushed": 15100,
  "currentSegment": 5,
  "currentOffset": 12345,
  "lastFlushTime": "2024-01-15T10:30:45Z"
}
```

---

### Get Checkpoint

Get the last checkpoint position.

**Request**:
```bash
GET /api/v1/{namespace}/checkpoint
```

**Example**:
```bash
curl http://localhost:4000/api/v1/default/checkpoint
```

**Response** (200 OK):
```json
{
  "namespace": "default",
  "recordProcessed": 15000,
  "segmentId": 5,
  "offset": 12000
}
```

---

## Backup Operations

UnisonDB provides APIs for creating durable backups of both WAL segments and B-Tree snapshots. All backup paths must be **relative** to the server's backup root (`<dataDir>/backups/{namespace}`).

### WAL Segment Backup

Create an incremental backup by copying sealed WAL segments.

**Request**:
```bash
POST /api/v1/{namespace}/wal/backup
Content-Type: application/json

{
  "afterSegmentId": 42,
  "backupDir": "wal/customer-a"
}
```

**Parameters**:
- `afterSegmentId` (optional): Only copy segments with IDs greater than this value. Omit or set to `0` to copy all sealed segments.
- `backupDir` (required): **Relative path** within the backup root. Absolute paths or `..` traversal are rejected.

**Example**:
```bash
# Backup all segments after segment 100
curl -X POST http://localhost:4000/api/v1/default/wal/backup \
  -H "Content-Type: application/json" \
  -d '{
    "afterSegmentId": 100,
    "backupDir": "wal/daily"
  }'
```

**Response** (200 OK):
```json
{
  "backups": [
    {
      "segmentId": 101,
      "path": "/var/unison/data/backups/default/wal/daily/000000101.wal"
    },
    {
      "segmentId": 102,
      "path": "/var/unison/data/backups/default/wal/daily/000000102.wal"
    }
  ]
}
```

**Use Cases**:
- Incremental backups for point-in-time recovery
- Compliance archival of transaction logs
- Shipping WAL segments to remote storage

**Errors**:
- `400 Bad Request`: Invalid path (absolute or contains `..`)
- `404 Not Found`: Namespace not found
- `500 Internal Server Error`: Filesystem error, permission denied

---

### B-Tree Snapshot Backup

Create a full snapshot of the B-Tree store.

**Request**:
```bash
POST /api/v1/{namespace}/btree/backup
Content-Type: application/json

{
  "path": "snapshots/users-20250108.snapshot"
}
```

**Parameters**:
- `path` (required): **Relative path** within the backup root. The server writes to `{path}.tmp`, fsyncs, then atomically renames.

**Example**:
```bash
# Create a snapshot with today's date
curl -X POST http://localhost:4000/api/v1/default/btree/backup \
  -H "Content-Type: application/json" \
  -d '{
    "path": "snapshots/backup-'$(date +%Y%m%d)'.db"
  }'
```

**Response** (200 OK):
```json
{
  "path": "/var/unison/data/backups/default/snapshots/backup-20250108.db",
  "bytes": 73400320
}
```

**Use Cases**:
- Full database backups for disaster recovery
- Creating read-only replicas for analytics
- Migrating data to new servers

**Errors**:
- `400 Bad Request`: Invalid path (absolute or contains `..`)
- `404 Not Found`: Namespace not found
- `500 Internal Server Error`: Filesystem error, disk full

---

### Backup Workflow Example

Complete backup automation script:

```bash
#!/bin/bash
# Daily backup script

NAMESPACE="users"
DATE=$(date +%Y%m%d)
STATE_FILE="/var/lib/unison/wal-state.json"

# 1. B-Tree snapshot (daily)
echo "Creating B-Tree snapshot..."
curl -X POST http://localhost:4000/api/v1/$NAMESPACE/btree/backup \
  -H "Content-Type: application/json" \
  -d "{\"path\": \"snapshots/btree-$DATE.db\"}"

# 2. WAL incremental
echo "Backing up WAL segments..."
LAST_SEGMENT=$(jq -r '.lastSegmentId // 0' "$STATE_FILE" 2>/dev/null || echo 0)

RESPONSE=$(curl -s -X POST http://localhost:4000/api/v1/$NAMESPACE/wal/backup \
  -H "Content-Type: application/json" \
  -d "{\"afterSegmentId\": $LAST_SEGMENT, \"backupDir\": \"wal/$DATE\"}")

# 3. Update state
NEW_LAST=$(echo "$RESPONSE" | jq -r '.backups[-1].segmentId // 0')
if [ "$NEW_LAST" != "0" ]; then
  echo "{\"lastSegmentId\": $NEW_LAST, \"timestamp\": \"$(date -Iseconds)\"}" > "$STATE_FILE"
fi

# 4. Compress and upload to S3 (optional)
BACKUP_ROOT="/var/unison/data/backups/$NAMESPACE"
tar -czf "/tmp/backup-$DATE.tar.gz" -C "$BACKUP_ROOT" .
aws s3 cp "/tmp/backup-$DATE.tar.gz" "s3://backups/unison/$NAMESPACE/"

echo "Backup completed successfully"
```

For detailed backup strategies, automation, and restore procedures, see the [Backup and Restore Guide](../../operations/backup-restore/).

---

## Error Responses

All errors follow this format:

```json
{
  "error": "error message description"
}
```

### HTTP Status Codes

| Code | Meaning | Example |
|------|---------|---------|
| 200 | Success | Operation completed |
| 400 | Bad Request | Invalid base64, malformed JSON |
| 404 | Not Found | Namespace not found, key not found, transaction not found |
| 500 | Internal Server Error | Engine error, disk full, WAL error |

### Common Errors

**Namespace not found**:
```json
{
  "error": "namespace not found: invalid-ns"
}
```
Status: `404 Not Found`

**Transaction not found**:
```json
{
  "error": "transaction not found: 2a3b4c5d..."
}
```
Status: `404 Not Found`

**Invalid base64**:
```json
{
  "error": "invalid base64 encoding"
}
```
Status: `400 Bad Request`

**Engine error**:
```json
{
  "error": "failed to write: disk full"
}
```
Status: `500 Internal Server Error`

---

## Complete Transaction Example

```bash
#!/bin/bash

# 1. Begin transaction
RESPONSE=$(curl -s -X POST http://localhost:4000/api/v1/default/tx/begin \
  -d '{"operation":"put","entryType":"kv"}')

TXN_ID=$(echo $RESPONSE | jq -r '.txnId')
echo "Transaction ID: $TXN_ID"

# 2. Append multiple operations
curl -X POST http://localhost:4000/api/v1/default/tx/$TXN_ID/kv \
  -d '{"key":"account:alice:balance","value":"MTAwMA=="}'

curl -X POST http://localhost:4000/api/v1/default/tx/$TXN_ID/kv \
  -d '{"key":"account:bob:balance","value":"MjAwMA=="}'

curl -X POST http://localhost:4000/api/v1/default/tx/$TXN_ID/kv \
  -d '{"key":"transfer:log:123","value":"YWxpY2UgLT4gYm9iOiAxMDA="}'

# 3. Commit
curl -X POST http://localhost:4000/api/v1/default/tx/$TXN_ID/commit

echo "Transaction committed!"

# 4. Verify
curl http://localhost:4000/api/v1/default/kv/account:alice:balance
```

---

