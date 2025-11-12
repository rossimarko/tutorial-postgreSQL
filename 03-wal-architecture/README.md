# 03 - WAL Architecture

## Table of Contents
- [WAL Fundamentals](#wal-fundamentals)
- [WAL Internals](#wal-internals)
- [Checkpoints](#checkpoints)
- [fsync and Durability](#fsync-and-durability)
- [WAL Archiving](#wal-archiving)
- [Anti-Patterns](#anti-patterns)
- [Monitoring](#monitoring)
- [Real-World Scenarios](#real-world-scenarios)

## WAL Fundamentals

### What is WAL?

**Write-Ahead Logging (WAL)** is the foundation of PostgreSQL's crash recovery and replication mechanisms.

**Core Principle:** Changes must be logged to WAL **before** being written to data files.

```
┌─────────────────────────────────────────────┐
│          Transaction Workflow               │
└─────────────────────────────────────────────┘
                    │
        1. BEGIN    ▼
    ┌──────────────────────┐
    │ Modify Shared Buffers│ (in memory)
    └──────────┬───────────┘
               │
        2.     ▼
    ┌──────────────────────┐
    │  Write to WAL Buffer │ (in memory)
    └──────────┬───────────┘
               │
        3.     ▼
    ┌──────────────────────┐
    │  Flush WAL to Disk   │ (on COMMIT)
    └──────────┬───────────┘
               │
        4.     ▼
    ┌──────────────────────┐
    │   Return SUCCESS     │
    └──────────┬───────────┘
               │
Later: Checkpoint writes dirty buffers to data files
```

### Why WAL?

1. **Crash Recovery:** Replay WAL to restore database to consistent state
2. **Performance:** Write sequential WAL (fast) instead of random data file changes (slow)
3. **Replication:** Stream WAL to standby servers
4. **Point-in-Time Recovery (PITR):** Restore to any point by replaying archived WAL

### WAL vs Traditional Logging

| Aspect | Traditional (UNDO Logging) | PostgreSQL WAL (REDO Logging) |
|--------|---------------------------|-------------------------------|
| **What's Logged** | Before and after images | Changes to make |
| **Recovery** | Undo incomplete transactions | Redo committed transactions |
| **Complexity** | Higher (undo/redo) | Lower (redo only) |
| **MVCC Integration** | Difficult | Natural fit |

## WAL Internals

### WAL File Structure

```
/var/lib/postgresql/data/pg_wal/
├── 000000010000000000000001  (16MB each)
├── 000000010000000000000002
├── 000000010000000000000003
├── ...
└── archive_status/
    ├── 000000010000000000000001.done
    └── 000000010000000000000001.ready
```

**WAL File Naming:**
```
000000010000000000000001
│      ││              │
│      ││              └─ Segment number
│      │└──────────────── Timeline ID (high 32 bits)
│      └───────────────── Timeline ID (low 32 bits)
└──────────────────────── Timeline ID
```

### WAL Records

Each WAL record contains:
- **Header:** Transaction ID, record length, CRC
- **Resource Manager ID:** Table, index, transaction, etc.
- **Block Reference:** Which page was modified
- **Record Data:** The actual change

```sql
-- View WAL record details (requires pg_walinspect extension)
CREATE EXTENSION pg_walinspect;

-- Current WAL insert location
SELECT pg_current_wal_lsn();
-- Result: 0/3000060

-- Examine WAL records
SELECT lsn,
       xid,
       resource_manager,
       length,
       description
FROM pg_get_wal_records_info('0/3000000', '0/3001000')
LIMIT 10;
```

### LSN (Log Sequence Number)

LSN is a 64-bit value representing a position in the WAL stream:

```
Format: segment_file_id/offset_within_file
Example: 0/3000060
         │ │
         │ └─ Offset: 3000060 bytes
         └─── Segment: 0
```

```sql
-- Current WAL positions
SELECT pg_current_wal_lsn() as current_wal,
       pg_current_wal_insert_lsn() as insert_lsn,
       pg_current_wal_flush_lsn() as flush_lsn;

-- Calculate WAL distance
SELECT pg_size_pretty(
    pg_wal_lsn_diff(
        pg_current_wal_lsn(),
        '0/0'
    )
) as total_wal_generated;

-- WAL generation rate
WITH wal_stats AS (
    SELECT pg_current_wal_lsn() as lsn,
           now() as time
)
SELECT pg_size_pretty(
    pg_wal_lsn_diff(w2.lsn, w1.lsn)
) as wal_generated,
extract(epoch FROM (w2.time - w1.time)) as seconds
FROM wal_stats w1, wal_stats w2;
-- Run this twice with delay to measure rate
```

### WAL Buffers

```ini
# postgresql.conf
wal_buffers = 16MB        # -1 = auto (1/32 of shared_buffers)
```

WAL flow through buffers:
```
1. Transaction modifies data
   ↓
2. Write to WAL buffers (in shared memory)
   ↓
3. WAL writer process (or COMMIT) flushes to OS
   ↓
4. fsync() forces to physical disk
   ↓
5. Transaction committed safely
```

**Performance Test:**
```sql
-- Generate WAL to test buffer size
CREATE TABLE wal_test (id INT, data TEXT);

-- Monitor WAL buffer usage
SELECT * FROM pg_stat_wal;

-- Look for wal_buffers_full (should be 0 or very low)
```

## Checkpoints

### What is a Checkpoint?

A checkpoint flushes all dirty shared buffers to disk, creating a known recovery point.

**Checkpoint Process:**
```
1. Checkpoint starts
   ↓
2. Mark checkpoint start in WAL
   ↓
3. Identify all dirty buffers
   ↓
4. Write dirty buffers to data files (spread over checkpoint_completion_target)
   ↓
5. fsync all data files
   ↓
6. Update pg_control file with checkpoint LSN
   ↓
7. Checkpoint complete
```

### Checkpoint Configuration

```ini
# postgresql.conf

# Time-based checkpoint
checkpoint_timeout = 15min              # Max time between checkpoints

# Size-based checkpoint
max_wal_size = 4GB                      # Trigger checkpoint at this WAL size
min_wal_size = 1GB                      # Recycle WAL files below this

# Checkpoint spreading
checkpoint_completion_target = 0.9      # Spread over 90% of interval

# Logging
log_checkpoints = on                    # Log checkpoint statistics
```

### Checkpoint Tuning

**Problem: Too Frequent Checkpoints**
```
# In logs:
LOG: checkpoints are occurring too frequently (24 seconds apart)
HINT: Consider increasing max_wal_size
```

**Impact:**
- High I/O load
- Performance degradation
- Increased write amplification

**Solution:**
```ini
# Increase max_wal_size
max_wal_size = 8GB          # Up from 1GB
checkpoint_timeout = 30min   # Up from 5min
```

**Performance Comparison:**
```sql
-- Before: checkpoint every 60 seconds
-- TPS: 500
-- I/O wait: 25%

-- After: checkpoint every 15 minutes
-- TPS: 1200 (2.4x improvement)
-- I/O wait: 8%
```

### Checkpoint Statistics

```sql
-- View checkpoint stats
SELECT checkpoints_timed,
       checkpoints_req,
       checkpoint_write_time,
       checkpoint_sync_time,
       buffers_checkpoint,
       buffers_clean,
       buffers_backend,
       buffers_backend_fsync
FROM pg_stat_bgwriter;
```

**Interpreting Results:**
- `checkpoints_timed`: Good (scheduled)
- `checkpoints_req`: Should be low (forced by max_wal_size)
- Ratio: `checkpoints_req / checkpoints_timed` should be < 0.1

```sql
-- Checkpoint frequency analysis
WITH checkpoint_stats AS (
    SELECT checkpoints_timed + checkpoints_req as total_checkpoints,
           checkpoints_req::float / NULLIF(checkpoints_timed + checkpoints_req, 0) as req_ratio
    FROM pg_stat_bgwriter
)
SELECT total_checkpoints,
       round(req_ratio * 100, 2) as pct_forced_checkpoints
FROM checkpoint_stats;

-- If pct_forced_checkpoints > 10%, increase max_wal_size
```

### Manual Checkpoints

```sql
-- Force immediate checkpoint
CHECKPOINT;

-- Before maintenance operations (recommended)
-- 1. Checkpoint to reduce recovery time
CHECKPOINT;

-- 2. Perform maintenance
VACUUM FULL large_table;

-- 3. Another checkpoint
CHECKPOINT;
```

## fsync and Durability

### Durability Guarantees

PostgreSQL guarantees **ACID compliance** through fsync:

```
Transaction COMMIT
   ↓
Write to WAL buffer
   ↓
Flush WAL buffer to OS
   ↓
fsync() WAL file to disk ← COMMIT doesn't return until this completes
   ↓
Return success to client
```

### fsync Configuration

```ini
# postgresql.conf

# fsync method
fsync = on                              # NEVER turn off in production!
synchronous_commit = on                 # on, remote_apply, remote_write, local, off

# fsync implementation
wal_sync_method = fdatasync             # open_datasync, fdatasync, fsync, open_sync

# Group commit optimization
commit_delay = 0                        # Microseconds (0 = disabled)
commit_siblings = 5                     # Minimum concurrent transactions
```

### synchronous_commit Levels

| Level | Durability | Performance | Use Case |
|-------|------------|-------------|----------|
| `off` | WAL not guaranteed on crash | Fastest | Non-critical data, bulk loads |
| `local` | WAL on primary disk | Fast | Single server |
| `remote_write` | WAL sent to standby OS | Medium | Replication, not yet fsynced |
| `on` | WAL fsynced on primary | Standard | Default |
| `remote_apply` | Applied on standby | Slowest | Zero data loss replication |

**Performance Comparison:**
```sql
-- Test with synchronous_commit = on
BEGIN;
INSERT INTO test_table SELECT generate_series(1, 10000);
COMMIT;
-- Time: 850ms

-- Test with synchronous_commit = off
SET LOCAL synchronous_commit = off;
BEGIN;
INSERT INTO test_table SELECT generate_series(1, 10000);
COMMIT;
-- Time: 45ms (19x faster!)

-- Risk: If server crashes before WAL is flushed, last few commits are lost
```

**Use Case: Bulk Loading**
```sql
-- Temporarily disable for bulk load
SET synchronous_commit = off;

COPY large_table FROM '/data/bulk_data.csv' WITH CSV;
-- 1M rows: 15 seconds vs 180 seconds

-- Re-enable
SET synchronous_commit = on;

-- Force WAL flush
SELECT pg_wal_replay_pause();
CHECKPOINT;
```

### wal_sync_method

Different operating systems and file systems optimize differently:

```ini
# Linux with ext4/xfs
wal_sync_method = fdatasync              # Fastest on Linux

# FreeBSD
wal_sync_method = fsync

# macOS
wal_sync_method = fsync_writethrough
```

**Benchmark sync methods:**
```bash
# Use pg_test_fsync utility
pg_test_fsync -f /var/lib/postgresql/data

# Sample output:
# fdatasync:            1234.567 ops/sec
# fsync:                1123.456 ops/sec
# open_datasync:        1098.765 ops/sec
```

## WAL Archiving

### Why Archive WAL?

1. **Point-in-Time Recovery (PITR):** Restore to exact moment
2. **Disaster Recovery:** Rebuild from base backup + WAL archives
3. **Delayed Standby:** Standby server lags behind by time interval

### Enabling WAL Archiving

```ini
# postgresql.conf

# Enable archiving
wal_level = replica                     # minimal, replica, or logical
archive_mode = on                       # Enable archiving
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
                                        # Command to archive WAL files
archive_timeout = 300                   # Force WAL switch every 5 minutes

# Archive library (PostgreSQL 15+, alternative to archive_command)
archive_library = 'basic_archive'
basic_archive.archive_directory = '/mnt/wal_archive'
```

**Placeholder Variables:**
- `%p`: Full path of file to archive
- `%f`: File name only
- `%r`: Last restartpoint file name

### Archive Command Examples

**Local directory:**
```bash
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
```

**Remote server (rsync):**
```bash
archive_command = 'rsync -a %p backup-server:/wal_archive/%f'
```

**Amazon S3:**
```bash
archive_command = 'aws s3 cp %p s3://my-bucket/wal_archive/%f'
```

**With compression:**
```bash
archive_command = 'gzip < %p > /mnt/wal_archive/%f.gz'
```

**Production-grade with error handling:**
```bash
#!/bin/bash
# /usr/local/bin/archive_wal.sh
WAL_FILE=$1
WAL_PATH=$2
ARCHIVE_DIR="/mnt/wal_archive"

# Check if already archived
if [ -f "$ARCHIVE_DIR/$WAL_FILE" ]; then
    exit 0
fi

# Copy with atomic write
cp "$WAL_PATH" "$ARCHIVE_DIR/${WAL_FILE}.tmp" || exit 1
mv "$ARCHIVE_DIR/${WAL_FILE}.tmp" "$ARCHIVE_DIR/$WAL_FILE" || exit 1

exit 0
```

```ini
archive_command = '/usr/local/bin/archive_wal.sh %f %p'
```

### Monitoring WAL Archiving

```sql
-- Check archiving status
SELECT archived_count,
       last_archived_wal,
       last_archived_time,
       failed_count,
       last_failed_wal,
       last_failed_time
FROM pg_stat_archiver;

-- WAL files waiting to be archived
SELECT count(*)
FROM pg_ls_waldir()
WHERE name NOT IN (
    SELECT last_archived_wal FROM pg_stat_archiver
);

-- Archive lag
SELECT now() - last_archived_time as archive_lag
FROM pg_stat_archiver;
```

**Alert if archiving fails:**
```sql
-- Set up monitoring query
SELECT failed_count,
       last_failed_wal,
       last_failed_time
FROM pg_stat_archiver
WHERE failed_count > 0;

-- If failed_count increases, investigate immediately!
```

### WAL File Management

```sql
-- List WAL files
SELECT name,
       size,
       modification
FROM pg_ls_waldir()
ORDER BY modification DESC;

-- Total WAL directory size
SELECT pg_size_pretty(sum(size)) as total_wal_size,
       count(*) as wal_file_count
FROM pg_ls_waldir();

-- Switch WAL file manually (force archiving)
SELECT pg_switch_wal();

-- Archive status
SELECT * FROM pg_ls_archive_statusdir();
```

### Cleaning Up Archived WAL

```bash
# Find WAL files older than 30 days
find /mnt/wal_archive -type f -name "*.wal" -mtime +30

# Delete old archives (after verifying backups!)
find /mnt/wal_archive -type f -name "*.wal" -mtime +30 -delete

# Better: Use pg_archivecleanup
pg_archivecleanup /mnt/wal_archive 000000010000000000000100
# Removes all WAL files before the specified file
```

## Anti-Patterns

### ❌ Anti-Pattern 1: Disabling fsync in Production
```ini
# NEVER DO THIS IN PRODUCTION
fsync = off
```

**Impact:**
- Database corruption on crash
- Data loss
- Unrecoverable state

**Only acceptable:**
- Initial bulk data load on new database
- Immediately followed by base backup

### ❌ Anti-Pattern 2: Insufficient max_wal_size
```ini
# BAD: Default 1GB on high-volume OLTP system
max_wal_size = 1GB
checkpoint_timeout = 5min
```

**Impact:**
- Checkpoint every 60 seconds
- High I/O, slow queries
- Increased write amplification

**Solution:**
```ini
# GOOD: Scale with workload
max_wal_size = 8GB
checkpoint_timeout = 30min
checkpoint_completion_target = 0.9
```

### ❌ Anti-Pattern 3: Not Monitoring Archive Failures
```bash
# BAD: Archive command that silently fails
archive_command = 'cp %p /mnt/wal_archive/%f'
# If /mnt is full, archiving fails but database continues
```

**Impact:**
- WAL accumulates, fills disk
- Database shutdown when pg_wal fills
- Unable to perform PITR

**Solution:**
```bash
# GOOD: Monitor and alert
SELECT failed_count FROM pg_stat_archiver WHERE failed_count > 0;
# Set up alerting (Prometheus, Nagios, etc.)
```

### ❌ Anti-Pattern 4: archive_command Overwrites Existing Files
```bash
# DANGEROUS: Can corrupt archive
archive_command = 'cp %p /mnt/wal_archive/%f'
```

**Risk:**
- Partial writes if process interrupted
- Corrupted archive

**Solution:**
```bash
# SAFE: Check before overwriting
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
```

## Monitoring

### Comprehensive WAL Monitoring

```sql
-- WAL Statistics Dashboard
WITH wal_stats AS (
    SELECT pg_current_wal_lsn() as current_lsn,
           pg_walfile_name(pg_current_wal_lsn()) as current_wal_file,
           (SELECT count(*) FROM pg_ls_waldir()) as wal_file_count,
           (SELECT pg_size_pretty(sum(size)) FROM pg_ls_waldir()) as total_wal_size
),
archive_stats AS (
    SELECT archived_count,
           failed_count,
           last_archived_wal,
           last_archived_time,
           now() - last_archived_time as archive_lag
    FROM pg_stat_archiver
),
checkpoint_stats AS (
    SELECT checkpoints_timed,
           checkpoints_req,
           round(100.0 * checkpoints_req / NULLIF(checkpoints_timed + checkpoints_req, 0), 2) as pct_forced,
           pg_size_pretty(buffers_checkpoint * 8192) as checkpoint_write_volume,
           round(checkpoint_write_time::numeric / 1000, 2) as checkpoint_write_sec,
           round(checkpoint_sync_time::numeric / 1000, 2) as checkpoint_sync_sec
    FROM pg_stat_bgwriter
)
SELECT 'WAL' as category, * FROM wal_stats
UNION ALL
SELECT 'Archive', * FROM archive_stats
UNION ALL
SELECT 'Checkpoint', * FROM checkpoint_stats;
```

### WAL Generation Rate

```sql
-- Create function to track WAL generation
CREATE TABLE wal_tracking (
    measured_at TIMESTAMP PRIMARY KEY,
    wal_lsn pg_lsn,
    wal_files INT
);

-- Collect sample
INSERT INTO wal_tracking
SELECT now(),
       pg_current_wal_lsn(),
       (SELECT count(*) FROM pg_ls_waldir());

-- Calculate generation rate (after 1 hour)
SELECT t2.measured_at - t1.measured_at as duration,
       pg_size_pretty(pg_wal_lsn_diff(t2.wal_lsn, t1.wal_lsn)) as wal_generated,
       pg_size_pretty(
           pg_wal_lsn_diff(t2.wal_lsn, t1.wal_lsn) /
           EXTRACT(epoch FROM (t2.measured_at - t1.measured_at))
       ) as wal_per_second
FROM wal_tracking t1, wal_tracking t2
WHERE t1.measured_at < t2.measured_at
ORDER BY t2.measured_at DESC
LIMIT 1;
```

### Alerts and Thresholds

```sql
-- Alert: Archive lag > 5 minutes
SELECT 'CRITICAL: Archive lag exceeded' as alert,
       now() - last_archived_time as lag
FROM pg_stat_archiver
WHERE now() - last_archived_time > interval '5 minutes';

-- Alert: Forced checkpoints > 10%
SELECT 'WARNING: Too many forced checkpoints' as alert,
       checkpoints_req,
       checkpoints_timed,
       round(100.0 * checkpoints_req / (checkpoints_timed + checkpoints_req), 2) as pct_forced
FROM pg_stat_bgwriter
WHERE checkpoints_req::float / (checkpoints_timed + checkpoints_req) > 0.1;

-- Alert: WAL directory > 80% of max_wal_size
WITH wal_usage AS (
    SELECT sum(size) as total_size,
           current_setting('max_wal_size')::text as max_wal
    FROM pg_ls_waldir()
)
SELECT 'WARNING: WAL directory filling up' as alert,
       pg_size_pretty(total_size) as current,
       max_wal as maximum
FROM wal_usage
WHERE total_size > pg_size_bytes(max_wal) * 0.8;
```

## Real-World Scenarios

### Scenario 1: Recovering from Checkpoint Saturation

**Symptoms:**
- Database slow during peak hours
- Logs show checkpoints every 30 seconds
- High I/O wait

**Investigation:**
```sql
SELECT checkpoints_req,
       checkpoints_timed,
       round(100.0 * checkpoints_req / (checkpoints_timed + checkpoints_req), 2) as pct_forced
FROM pg_stat_bgwriter;
-- Result: 90% forced checkpoints
```

**Solution:**
```ini
# Before
max_wal_size = 1GB
checkpoint_timeout = 5min

# After
max_wal_size = 8GB
checkpoint_timeout = 30min
checkpoint_completion_target = 0.9

# Restart required
```

**Result:**
- Checkpoints now every 15-20 minutes
- I/O wait dropped from 30% to 5%
- Query latency improved by 40%

### Scenario 2: WAL Archive Disk Full

**Symptoms:**
```
PANIC: could not write to file "pg_wal/xlogtemp.12345": No space left on device
```

**Investigation:**
```bash
df -h /var/lib/postgresql/data
# Result: 100% full

ls -lh /var/lib/postgresql/data/pg_wal/ | wc -l
# Result: 500 WAL files (8GB)
```

**Root Cause:**
```sql
SELECT failed_count, last_failed_time
FROM pg_stat_archiver;
-- Archive has been failing for 2 hours
```

**Resolution:**
```bash
# 1. Fix archive destination (e.g., mount failed)
mount /mnt/wal_archive

# 2. Manually archive WAL files
for wal in /var/lib/postgresql/data/pg_wal/0000*; do
    cp "$wal" /mnt/wal_archive/
done

# 3. Switch WAL file to trigger archive
psql -c "SELECT pg_switch_wal();"

# 4. Verify archiving resumed
psql -c "SELECT * FROM pg_stat_archiver;"
```

### Scenario 3: Point-in-Time Recovery Setup

**Requirement:** Recover to 10:30 AM today after accidental DELETE at 10:35 AM.

**Prerequisites:**
```bash
# 1. Base backup taken at 2:00 AM
/backups/base_2024_01_15_0200.tar.gz

# 2. WAL archives available
ls /mnt/wal_archive/
# 000000010000000000000020 through 000000010000000000000045
```

**Recovery:**
```bash
# 1. Stop PostgreSQL
systemctl stop postgresql

# 2. Backup current data (safety)
mv /var/lib/postgresql/data /var/lib/postgresql/data.old

# 3. Restore base backup
mkdir -p /var/lib/postgresql/data
tar -xzf /backups/base_2024_01_15_0200.tar.gz -C /var/lib/postgresql/data
chown -R postgres:postgres /var/lib/postgresql/data

# 4. Create recovery.signal file
touch /var/lib/postgresql/data/recovery.signal

# 5. Configure recovery
cat > /var/lib/postgresql/data/postgresql.auto.conf <<EOF
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2024-01-15 10:30:00'
recovery_target_action = 'promote'
EOF

# 6. Start PostgreSQL
systemctl start postgresql

# 7. Monitor recovery
tail -f /var/log/postgresql/postgresql-17-main.log
# Wait for: "database system is ready to accept connections"

# 8. Verify data
psql -c "SELECT count(*) FROM deleted_table;"
# Should show data as of 10:30 AM
```

### Scenario 4: Optimizing WAL for High-Throughput Workload

**Workload:** 50,000 transactions/second, mostly INSERT/UPDATE

**Initial Configuration:**
```ini
wal_level = replica
wal_buffers = 16MB
max_wal_size = 1GB
checkpoint_timeout = 5min
checkpoint_completion_target = 0.5
wal_compression = off
```

**Issues:**
- Checkpoints every 2 minutes
- WAL generation: 500 MB/minute
- TPS drops during checkpoints

**Optimized Configuration:**
```ini
# Increase checkpoint interval
max_wal_size = 16GB
checkpoint_timeout = 30min
checkpoint_completion_target = 0.9

# Enable WAL compression (PostgreSQL 15+)
wal_compression = zstd

# Tune WAL buffers
wal_buffers = 64MB

# Consider dedicated WAL disk
# Move pg_wal to separate SSD
```

**Implementation:**
```bash
# Move pg_wal to fast SSD
systemctl stop postgresql
mv /var/lib/postgresql/data/pg_wal /mnt/fast_ssd/pg_wal
ln -s /mnt/fast_ssd/pg_wal /var/lib/postgresql/data/pg_wal
systemctl start postgresql
```

**Results:**
- WAL compression reduced size by 40%
- Checkpoints now every 20 minutes
- No TPS drops during checkpoints
- Overall throughput increased 35%

---

## Quick Reference

### Essential Commands
```sql
-- WAL location
SELECT pg_current_wal_lsn();
SELECT pg_walfile_name(pg_current_wal_lsn());

-- Switch WAL file
SELECT pg_switch_wal();

-- WAL stats
SELECT * FROM pg_stat_archiver;
SELECT * FROM pg_stat_bgwriter;

-- Force checkpoint
CHECKPOINT;

-- WAL files
SELECT * FROM pg_ls_waldir();
```

### Configuration Quick Guide
```ini
# Basic WAL
wal_level = replica
wal_buffers = 16MB

# Checkpoints
max_wal_size = 4GB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9

# Archiving
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
```

### Next Steps
- [← 02 - Database Creation](../02-database-creation/README.md)
- [04 - Backup and Restore →](../04-backup-restore/README.md)
- [Back to Main](../README.md)
