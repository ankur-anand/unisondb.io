---
title: "Backup and Restore"
weight: 10
description: "Complete guide to backing up and restoring UnisonDB data using WAL segments and B-Tree snapshots"
keywords: ["UnisonDB backup", "database backup", "WAL backup", "B-Tree snapshot", "disaster recovery", "point-in-time recovery"]
---

# Backup and Restore

UnisonDB provides HTTP APIs for creating durable backups of both Write-Ahead Log (WAL) segments and B-Tree snapshots.

## Overview

UnisonDB's backup system is designed around two complementary artifacts:

| Component | Purpose | Recovery Capability | Storage Size |
|-----------|---------|---------------------|--------------|
| **WAL Segments** | Incremental transaction logs | Point-in-time recovery | Small (append-only) |
| **B-Tree Snapshots** | Full database state | Fast baseline restore | Larger (full state) |

**Key Principles:**
- **Namespace isolation**: Each namespace has its own backup root (`<dataDir>/backups/{namespace}`)
- **Crash-consistent**: All backups are atomic and immediately usable
- **BYO tooling**: UnisonDB emits raw files; you control compression, encryption, and storage
- **Security**: Relative paths only—no directory traversal or cross-namespace access

## Backup APIs

### WAL Segment Backup

Copies sealed WAL segments to a backup directory for incremental archival.

**Endpoint:**
```
POST /api/v1/{namespace}/wal/backup
```

**Request Body:**
```json
{
  "afterSegmentId": 42,
  "backupDir": "wal/customer-a"
}
```

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `afterSegmentId` | integer | No | Only copy segments with IDs greater than this value. Omit or set to `0` to copy all sealed segments. |
| `backupDir` | string | Yes | **Relative path** within `<dataDir>/backups/{namespace}`. Absolute paths or `..` traversal are rejected. |

**Response:**
```json
{
  "backups": [
    {
      "segmentId": 43,
      "path": "/var/unison/data/backups/users/wal/customer-a/000000043.wal"
    },
    {
      "segmentId": 44,
      "path": "/var/unison/data/backups/users/wal/customer-a/000000044.wal"
    }
  ]
}
```

**Example:**
```bash
curl -X POST http://localhost:8080/api/v1/users/wal/backup \
  -H "Content-Type: application/json" \
  -d '{
    "afterSegmentId": 100,
    "backupDir": "wal/daily"
  }'
```

---

### B-Tree Snapshot Backup

Creates a consistent snapshot of the entire B-Tree store.

**Endpoint:**
```
POST /api/v1/{namespace}/btree/backup
```

**Request Body:**
```json
{
  "path": "snapshots/users-20250108.snapshot"
}
```

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | string | Yes | **Relative path** within `<dataDir>/backups/{namespace}`. UnisonDB writes to `{path}.tmp`, fsyncs, then atomically renames. |

**Response:**
```json
{
  "path": "/var/unison/data/backups/users/snapshots/users-20250108.snapshot",
  "bytes": 73400320
}
```

**Example:**
```bash
curl -X POST http://localhost:8080/api/v1/users/btree/backup \
  -H "Content-Type: application/json" \
  -d '{
    "path": "snapshots/users-'$(date +%Y%m%d)'.snapshot"
  }'
```

---

## Backup Strategies

### 1. Full Backup (Snapshot Only)

Simple strategy for small databases or infrequent backups.

**Workflow:**
```bash
# Daily full snapshot
curl -X POST http://localhost:8080/api/v1/users/btree/backup \
  -H "Content-Type: application/json" \
  -d '{"path": "daily/snapshot-'$(date +%Y%m%d)'.db"}'
```

---

### 2. Incremental Backup (WAL + Periodic Snapshots)

**Workflow:**

**Daily baseline:**
```bash
# Full B-Tree snapshot
curl -X POST http://localhost:8080/api/v1/users/btree/backup \
  -d '{"path": "weekly/snapshot-'$(date +%Y%m%d)'.db"}'
```

**Hourly incremental:**
```bash
# Every hour: Copy new WAL segments
LAST_SEGMENT=$(cat /var/backups/last_wal_id.txt || echo 0)

curl -X POST http://localhost:8080/api/v1/users/wal/backup \
  -d "{\"afterSegmentId\": $LAST_SEGMENT, \"backupDir\": \"wal/$(date +%Y%m%d-%H)\"}" \
  | jq -r '.backups[-1].segmentId' > /var/backups/last_wal_id.txt
```

---

### 3. Continuous WAL Archival

**Workflow:**

Use a cron job or systemd timer to continuously ship WAL segments:

```bash
#!/bin/bash
# /usr/local/bin/unison-wal-archive.sh

NAMESPACE="users"
STATE_FILE="/var/lib/unison/wal-archive-state.json"

# Read last archived segment
LAST_SEGMENT=$(jq -r '.lastSegmentId // 0' "$STATE_FILE" 2>/dev/null || echo 0)

# Backup new segments
RESPONSE=$(curl -s -X POST http://localhost:8080/api/v1/$NAMESPACE/wal/backup \
  -H "Content-Type: application/json" \
  -d "{\"afterSegmentId\": $LAST_SEGMENT, \"backupDir\": \"wal/archive\"}")

# Extract latest segment ID
NEW_LAST=$(echo "$RESPONSE" | jq -r '.backups[-1].segmentId // 0')

if [ "$NEW_LAST" != "0" ]; then
  # Update state
  echo "{\"lastSegmentId\": $NEW_LAST, \"timestamp\": \"$(date -Iseconds)\"}" > "$STATE_FILE"

  # Compress and upload to S3
  find /var/unison/data/backups/$NAMESPACE/wal/archive -name "*.wal" \
    -mmin -15 -exec gzip {} \; \
    -exec aws s3 cp {}.gz s3://backups/unison/$NAMESPACE/wal/ \;
fi
```

**Cron schedule:**
```cron
*/15 * * * * /usr/local/bin/unison-wal-archive.sh
```

---

## Backup Automation

### Systemd Timer Example

**Service unit** (`/etc/systemd/system/unison-backup.service`):
```ini
[Unit]
Description=UnisonDB Backup Service
After=network.target

[Service]
Type=oneshot
User=unison
ExecStart=/usr/local/bin/unison-backup.sh
StandardOutput=journal
StandardError=journal
```

**Timer unit** (`/etc/systemd/system/unison-backup.timer`):
```ini
[Unit]
Description=UnisonDB Backup Timer

[Timer]
OnCalendar=daily
OnCalendar=*:0/6  # Every 6 hours
Persistent=true

[Install]
WantedBy=timers.target
```

**Backup script** (`/usr/local/bin/unison-backup.sh`):
```bash
#!/bin/bash
set -euo pipefail

NAMESPACE="users"
BACKUP_ROOT="/var/unison/data/backups/$NAMESPACE"
DATE=$(date +%Y%m%d-%H%M)

# 1. B-Tree snapshot (daily at midnight)
if [ "$(date +%H%M)" = "0000" ]; then
  echo "Creating B-Tree snapshot..."
  curl -X POST http://localhost:8080/api/v1/$NAMESPACE/btree/backup \
    -H "Content-Type: application/json" \
    -d "{\"path\": \"snapshots/btree-$DATE.db\"}"

  # Compress snapshot
  zstd -q --rm "$BACKUP_ROOT/snapshots/btree-$DATE.db"

  # Upload to S3
  aws s3 cp "$BACKUP_ROOT/snapshots/btree-$DATE.db.zst" \
    s3://backups/unison/$NAMESPACE/snapshots/
fi

# 2. WAL incremental (every run)
echo "Backing up WAL segments..."
STATE_FILE="/var/lib/unison/wal-state-$NAMESPACE.json"
LAST_SEGMENT=$(jq -r '.lastSegmentId // 0' "$STATE_FILE" 2>/dev/null || echo 0)

RESPONSE=$(curl -s -X POST http://localhost:8080/api/v1/$NAMESPACE/wal/backup \
  -H "Content-Type: application/json" \
  -d "{\"afterSegmentId\": $LAST_SEGMENT, \"backupDir\": \"wal/$DATE\"}")

# Update state
NEW_LAST=$(echo "$RESPONSE" | jq -r '.backups[-1].segmentId // 0')
if [ "$NEW_LAST" != "0" ]; then
  echo "{\"lastSegmentId\": $NEW_LAST, \"timestamp\": \"$(date -Iseconds)\"}" > "$STATE_FILE"

  # Compress and upload WAL segments
  tar -czf "$BACKUP_ROOT/wal-$DATE.tar.gz" -C "$BACKUP_ROOT" "wal/$DATE"
  aws s3 cp "$BACKUP_ROOT/wal-$DATE.tar.gz" s3://backups/unison/$NAMESPACE/wal/

  # Cleanup local WAL backups older than 7 days
  find "$BACKUP_ROOT/wal" -type d -mtime +7 -exec rm -rf {} +
fi

echo "Backup completed successfully"
```

**Enable the timer:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now unison-backup.timer
sudo systemctl status unison-backup.timer
```

---

## Storage Integration Examples

### Amazon S3

```bash
#!/bin/bash
# Backup to S3 with encryption

NAMESPACE="users"
BACKUP_DIR="/var/unison/data/backups/$NAMESPACE"
S3_BUCKET="s3://company-backups/unison/$NAMESPACE"
DATE=$(date +%Y%m%d)

# 1. Create B-Tree snapshot
curl -X POST http://localhost:8080/api/v1/$NAMESPACE/btree/backup \
  -d "{\"path\": \"snapshots/btree-$DATE.db\"}"

# 2. Compress and encrypt
zstd -q "$BACKUP_DIR/snapshots/btree-$DATE.db"
gpg --encrypt --recipient backup@company.com \
  "$BACKUP_DIR/snapshots/btree-$DATE.db.zst"

# 3. Upload to S3 with server-side encryption
aws s3 cp "$BACKUP_DIR/snapshots/btree-$DATE.db.zst.gpg" \
  "$S3_BUCKET/snapshots/" \
  --storage-class STANDARD_IA \
  --server-side-encryption AES256

# 4. Cleanup local files
rm -f "$BACKUP_DIR/snapshots/btree-$DATE.db"*
```

---

## Restore Procedures

### Full Restore from B-Tree Snapshot

**Scenario:** Complete data loss, restore from latest snapshot.

```bash
# 1. Stop UnisonDB
sudo systemctl stop unisondb

# 2. Download snapshot from S3
aws s3 cp s3://backups/unison/users/snapshots/btree-20250108.db.zst \
  /tmp/restore.db.zst

# 3. Decompress
zstd -d /tmp/restore.db.zst -o /tmp/restore.db

# 4. Replace existing database
NAMESPACE="users"
DATA_DIR="/var/unison/data/$NAMESPACE"

rm -rf "$DATA_DIR/db"/*
mkdir -p "$DATA_DIR/db"

# Copy snapshot to data directory (engine-specific)
# For LMDB:
cp /tmp/restore.db "$DATA_DIR/db/data.mdb"

# 5. Restart UnisonDB
sudo systemctl start unisondb

# 6. Verify
curl http://localhost:8080/api/v1/users/kv/test-key
```

---

### Point-in-Time Recovery (PITR)

**Scenario:** Restore to a specific point in time using snapshot + WAL replay.

```bash
#!/bin/bash
# Restore to 2025-01-08 14:30:00 UTC

TARGET_TIME="2025-01-08T14:30:00Z"
NAMESPACE="users"

# 1. Find baseline snapshot before target time
SNAPSHOT=$(aws s3 ls s3://backups/unison/$NAMESPACE/snapshots/ | \
  awk '{print $4}' | \
  grep -E 'btree-[0-9]{8}' | \
  sort | \
  awk -v target="$(date -d "$TARGET_TIME" +%Y%m%d)" '$0 <= target' | \
  tail -1)

echo "Using baseline snapshot: $SNAPSHOT"

# 2. Download and restore snapshot
aws s3 cp "s3://backups/unison/$NAMESPACE/snapshots/$SNAPSHOT" /tmp/
zstd -d "/tmp/$SNAPSHOT" -o /tmp/restore.db

# 3. Download WAL segments after snapshot
SNAPSHOT_DATE=$(echo $SNAPSHOT | grep -oE '[0-9]{8}')
mkdir -p /tmp/wal-restore

aws s3 sync "s3://backups/unison/$NAMESPACE/wal/" /tmp/wal-restore/ \
  --exclude "*" \
  --include "wal-${SNAPSHOT_DATE}*.tar.gz"

# 4. Extract WAL segments
for archive in /tmp/wal-restore/*.tar.gz; do
  tar -xzf "$archive" -C /tmp/wal-restore/
done

# 5. Stop UnisonDB and restore
sudo systemctl stop unisondb

DATA_DIR="/var/unison/data/$NAMESPACE"
rm -rf "$DATA_DIR/db"/* "$DATA_DIR/wal"/*
cp /tmp/restore.db "$DATA_DIR/db/data.mdb"

# 6. Copy WAL segments up to target time
# (UnisonDB will replay WAL on startup)
find /tmp/wal-restore -name "*.wal" | sort | while read wal; do
  # Check WAL timestamp (implementation-specific)
  # Copy only if before target time
  cp "$wal" "$DATA_DIR/wal/"
done

# 7. Restart and verify
sudo systemctl start unisondb
```

---

## Best Practices

### Backup Schedule

| Frequency | Component | Retention |
|-----------|-----------|-----------|
| **Hourly** | WAL segments | 7 days local, 30 days remote |
| **Daily** | B-Tree snapshot | 7 days local, 90 days remote |
| **Weekly** | Full backup (WAL + snapshot) | 1 year |
| **Monthly** | Compliance archive | 7 years (if required) |

### Security

**1. Encrypt backups:**
```bash
# GPG encryption before upload
gpg --encrypt --recipient backup-key@company.com backup.db
```

**2. Access control:**
```bash
# Restrict backup directory permissions
chmod 700 /var/unison/data/backups
chown unison:unison /var/unison/data/backups
```

**3. Audit logging:**
```bash
# Log all backup API calls
curl -X POST http://localhost:8080/api/v1/users/wal/backup \
  -d '{"backupDir": "wal/archive"}' \
  | tee -a /var/log/unison-backups.log
```

### Testing Restores

**Monthly restore drill:**
```bash
#!/bin/bash
# Test restore in isolated environment

NAMESPACE="users"
TEST_DIR="/tmp/restore-test-$(date +%s)"

# 1. Create test environment
mkdir -p "$TEST_DIR"/{db,wal}

# 2. Download latest snapshot
aws s3 cp s3://backups/unison/$NAMESPACE/snapshots/latest.db.zst "$TEST_DIR/"
zstd -d "$TEST_DIR/latest.db.zst" -o "$TEST_DIR/db/data.mdb"

# 3. Start UnisonDB in test mode
unisondb --data-dir "$TEST_DIR" --http-addr localhost:9999 &
PID=$!

# 4. Verify data integrity
sleep 5
curl http://localhost:9999/health
curl http://localhost:9999/api/v1/$NAMESPACE/kv/test-key

# 5. Cleanup
kill $PID
rm -rf "$TEST_DIR"

echo "Restore test completed successfully"
```

### Monitoring

**Key metrics to track:**

```bash
# Backup success rate
curl http://localhost:8080/metrics | grep backup_success_total

# Last backup timestamp
curl http://localhost:8080/metrics | grep backup_last_timestamp_seconds

# Backup size
curl http://localhost:8080/metrics | grep backup_bytes_total
```

**Alert on:**
- Backup failures (2+ consecutive failures)
- Backup age > 24 hours
- Backup size anomalies (>50% change)
- WAL segment gap detection

---

## Troubleshooting

### Backup Fails with Permission Denied

**Symptom:**
```json
{"error": "permission denied: /var/unison/data/backups/users/wal"}
```

**Solution:**
```bash
# Ensure backup directory is writable
sudo chown -R unison:unison /var/unison/data/backups
sudo chmod -R 755 /var/unison/data/backups
```

---

### Backup Directory Full

**Symptom:**
```json
{"error": "no space left on device"}
```

**Solution:**
```bash
# Clean up old local backups
find /var/unison/data/backups -type f -mtime +7 -delete

# Or mount separate volume
sudo mount /dev/sdb1 /var/unison/data/backups
```

---

### Restore Fails with Corrupted Snapshot

**Symptom:**
```
ERROR: database file is corrupted
```

**Solution:**
```bash
# 1. Verify checksum (if available)
sha256sum backup.db
# Compare with original checksum

# 2. Try previous snapshot
aws s3 ls s3://backups/unison/users/snapshots/ | sort | tail -2
```
---

## Next Steps

- [Deployment Topologies](../deployment/) - High availability with replicas
- [Monitoring](../monitoring/) - Tracking backup health
- [HTTP API Reference](../api/http-api/) - Complete API documentation
