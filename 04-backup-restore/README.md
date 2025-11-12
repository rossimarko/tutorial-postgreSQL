# 04 - Backup and Restore

## Table of Contents
- [Backup Strategies](#backup-strategies)
- [Logical Backups (pg_dump)](#logical-backups-pg_dump)
- [Physical Backups (pg_basebackup)](#physical-backups-pg_basebackup)
- [Point-in-Time Recovery (PITR)](#point-in-time-recovery-pitr)
- [Restore Strategies](#restore-strategies)
- [Third-Party Tools](#third-party-tools)
- [Anti-Patterns](#anti-patterns)
- [Monitoring](#monitoring)
- [Real-World Scenarios](#real-world-scenarios)

## Backup Strategies

### Backup Types Comparison

| Type | Method | Size | Speed | Granularity | Use Case |
|------|--------|------|-------|-------------|----------|
| **Logical** | pg_dump | Larger | Slower | Table/Database | Schema changes, upgrades |
| **Physical** | pg_basebackup | Smaller | Faster | Cluster | PITR, replication |
| **Continuous** | WAL archiving | Variable | N/A | Point-in-time | Production systems |
| **Snapshot** | Filesystem/LVM | Instant | Very fast | Cluster | Quick recovery |

### Backup Strategy Decision Tree

```
Do you need point-in-time recovery?
├─ YES → Physical backup + WAL archiving
│         (pg_basebackup + archive_mode)
└─ NO
    │
    ├─ Need to restore to different PG version?
    │  └─ YES → Logical backup (pg_dump)
    │
    ├─ Need to restore specific tables?
    │  └─ YES → Logical backup (pg_dump)
    │
    └─ Fastest backup/restore?
       └─ YES → Physical backup (pg_basebackup)
```

### 3-2-1 Backup Rule

```
3 copies of data
2 different media types
1 offsite copy

Example:
1. Primary database (production)
2. Local backup (same datacenter, different disk)
3. Offsite backup (cloud storage or different datacenter)
```

## Logical Backups (pg_dump)

### pg_dump Basics

```bash
# Dump single database
pg_dump mydb > mydb_backup.sql

# Dump with compression
pg_dump mydb | gzip > mydb_backup.sql.gz

# Custom format (recommended, supports parallel restore)
pg_dump -Fc mydb > mydb_backup.dump

# Directory format (supports parallel backup/restore)
pg_dump -Fd mydb -j 4 -f mydb_backup_dir

# Plain SQL format
pg_dump -Fp mydb > mydb_backup.sql
```

### pg_dump Formats

| Format | Flag | Compression | Parallel Restore | Selective Restore | Size |
|--------|------|-------------|------------------|-------------------|------|
| **Plain** | `-Fp` | External only | No | Manual editing | Largest |
| **Custom** | `-Fc` | Built-in | Yes | Yes | Medium |
| **Directory** | `-Fd` | Built-in | Yes | Yes | Medium |
| **Tar** | `-Ft` | External only | No | Limited | Medium |

### Comprehensive pg_dump Options

```bash
# Production-grade backup with all options
pg_dump \
  -h localhost \
  -p 5432 \
  -U postgres \
  -d mydb \
  -Fc \                              # Custom format
  --file=mydb_$(date +%Y%m%d_%H%M%S).dump \
  --verbose \                         # Show progress
  --compress=9 \                      # Max compression (0-9)
  --no-owner \                        # Don't restore ownership
  --no-acl \                          # Don't restore permissions
  --exclude-table=temp_* \            # Exclude temp tables
  --exclude-table-data=audit_log      # Exclude data but keep schema
```

### Selective Backups

```bash
# Backup specific tables
pg_dump -t users -t orders mydb > critical_tables.sql

# Backup specific schema
pg_dump -n public mydb > public_schema.sql

# Backup only schema (no data)
pg_dump --schema-only mydb > schema_only.sql

# Backup only data (no schema)
pg_dump --data-only mydb > data_only.sql

# Exclude specific tables
pg_dump --exclude-table=logs --exclude-table=temp_data mydb > backup.sql
```

### pg_dumpall (Cluster-wide Backup)

```bash
# Backup entire cluster (all databases, roles, tablespaces)
pg_dumpall > cluster_backup.sql

# Backup only globals (roles, tablespaces)
pg_dumpall --globals-only > globals.sql

# Backup only roles
pg_dumpall --roles-only > roles.sql

# Typical strategy: globals + individual databases
pg_dumpall --globals-only > globals.sql
pg_dump -Fc database1 > database1.dump
pg_dump -Fc database2 > database2.dump
```

### Performance Optimization

**Parallel Dump (PostgreSQL 9.3+):**
```bash
# Use 4 parallel jobs (directory format only)
pg_dump -Fd -j 4 mydb -f mydb_backup_dir

# Monitor progress
watch -n 1 'ls -lh mydb_backup_dir/ | wc -l'
```

**Performance Comparison:**
```bash
# Sequential dump (1 core)
time pg_dump -Fc large_db > backup.dump
# Time: 45 minutes

# Parallel dump (4 cores)
time pg_dump -Fd -j 4 large_db -f backup_dir
# Time: 14 minutes (3.2x faster)
```

**Large Object Support:**
```bash
# Include large objects (BLOBs)
pg_dump --blobs mydb > backup.sql

# Exclude large objects
pg_dump --no-blobs mydb > backup_no_blobs.sql
```

## Physical Backups (pg_basebackup)

### pg_basebackup Basics

```bash
# Basic physical backup
pg_basebackup -D /backup/base -Ft -z -P

# Detailed options
pg_basebackup \
  -h localhost \
  -p 5432 \
  -U replication_user \
  -D /backup/base_$(date +%Y%m%d_%H%M%S) \
  -Ft \                               # Tar format
  -z \                                # Compress
  -P \                                # Progress reporting
  -X stream \                         # Include WAL files
  -c fast                             # Fast checkpoint
```

### pg_basebackup Options

| Option | Description | Recommended |
|--------|-------------|-------------|
| `-D directory` | Target directory | Required |
| `-Ft` | Tar format (compressed) | Yes |
| `-Fp` | Plain format (directory tree) | For direct recovery |
| `-z` | Gzip compression | Yes |
| `-X stream` | Include WAL in backup | Yes (for standalone backup) |
| `-X fetch` | Fetch WAL at end | No (risky) |
| `-P` | Progress reporting | Yes |
| `-c fast` | Fast checkpoint | Yes |
| `-R` | Create recovery configuration | Yes (for standbys) |

### Parallel pg_basebackup (PostgreSQL 15+)

```bash
# Use multiple jobs for faster backup
pg_basebackup \
  -D /backup/base \
  --format=tar \
  --gzip \
  --compress=9 \
  --jobs=4 \
  -X stream \
  -P
```

### Creating Standby Server with pg_basebackup

```bash
# On standby server
pg_basebackup \
  -h primary-server \
  -p 5432 \
  -U replication_user \
  -D /var/lib/postgresql/17/main \
  -Fp \                               # Plain format for direct use
  -X stream \
  -P \
  -R                                  # Create standby.signal + config

# Automatically creates:
# - standby.signal file
# - primary_conninfo in postgresql.auto.conf

# Start standby
systemctl start postgresql
```

### Incremental Backups (PostgreSQL 17+)

```bash
# Full backup with manifest
pg_basebackup -D /backup/full -Fp --manifest-path=/backup/full.manifest

# Incremental backup
pg_combinebackup \
  --backup=/backup/full \
  --incremental=/backup/incr1 \
  --output=/backup/combined

# Restore from incremental
pg_combinebackup \
  /backup/full \
  /backup/incr1 \
  /backup/incr2 \
  -o /var/lib/postgresql/data
```

## Point-in-Time Recovery (PITR)

### PITR Prerequisites

1. **Base Backup:** Physical backup from pg_basebackup
2. **WAL Archiving:** Continuous WAL archive since backup
3. **Archive Location:** Accessible WAL archive directory

### PITR Configuration

**1. Enable WAL Archiving (postgresql.conf):**
```ini
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /mnt/wal_archive/%f && cp %p /mnt/wal_archive/%f'
```

**2. Take Base Backup:**
```bash
# Full backup with WAL
pg_basebackup \
  -D /backup/base_$(date +%Y%m%d_%H%M%S) \
  -Ft -z -P -X stream

# Record backup time
echo "Backup completed at: $(date)" > /backup/backup_time.txt
```

**3. Continuous WAL Archiving:**
```bash
# Verify WAL files are being archived
ls -lh /mnt/wal_archive/

# Check archiver status
psql -c "SELECT * FROM pg_stat_archiver;"
```

### Performing PITR

**Scenario: Recover to specific time**

```bash
# 1. Stop PostgreSQL
systemctl stop postgresql

# 2. Backup current data directory (safety)
mv /var/lib/postgresql/17/main /var/lib/postgresql/17/main.old

# 3. Restore base backup
mkdir -p /var/lib/postgresql/17/main
tar -xzf /backup/base.tar.gz -C /var/lib/postgresql/17/main

# 4. Create recovery.signal file
touch /var/lib/postgresql/17/main/recovery.signal

# 5. Configure recovery (postgresql.auto.conf or recovery.conf)
cat > /var/lib/postgresql/17/main/postgresql.auto.conf <<EOF
restore_command = 'cp /mnt/wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
recovery_target_action = 'promote'
EOF

# 6. Fix permissions
chown -R postgres:postgres /var/lib/postgresql/17/main

# 7. Start PostgreSQL and watch recovery
systemctl start postgresql
tail -f /var/log/postgresql/postgresql-17-main.log

# Wait for:
# LOG:  redo starts at 0/2000028
# LOG:  recovery stopping before commit of transaction 1234
# LOG:  recovery has paused
# LOG:  database system is ready to accept connections
```

### Recovery Targets

```ini
# Recover to specific timestamp
recovery_target_time = '2024-01-15 14:30:00'

# Recover to transaction ID
recovery_target_xid = '12345'

# Recover to named restore point
recovery_target_name = 'before_migration'

# Recover to specific WAL location
recovery_target_lsn = '0/3000000'

# Recover to end of available WAL
recovery_target = 'immediate'
```

**Create Named Restore Point:**
```sql
-- Must be superuser
SELECT pg_create_restore_point('before_major_update');
```

### Recovery Target Actions

```ini
# After reaching target, what to do?

# Promote to normal operation (default)
recovery_target_action = 'promote'

# Pause and wait for manual intervention
recovery_target_action = 'pause'

# Shutdown server
recovery_target_action = 'shutdown'
```

**Paused Recovery:**
```sql
-- While paused, examine database (read-only)
SELECT * FROM critical_table;

-- If correct, promote to read-write
SELECT pg_wal_replay_resume();

-- If incorrect, shutdown and adjust recovery target
SELECT pg_wal_replay_pause();
\q
systemctl stop postgresql
# Adjust recovery_target_time and restart
```

### Recovery Timeline

After PITR, database creates a new timeline:

```
Timeline 1: Original
├─ Base backup (10:00)
├─ Normal operation
├─ Accidental DELETE (14:35)
└─ Recovery to 14:30
    │
    └─ Timeline 2: After PITR recovery
       └─ New writes start here
```

```sql
-- View current timeline
SELECT timeline_id, redo_lsn FROM pg_control_checkpoint();

-- Timeline history
SELECT * FROM pg_walfile_name_offset(pg_current_wal_lsn());
```

## Restore Strategies

### Restoring from pg_dump

**Plain SQL Format:**
```bash
# Restore full database
psql mydb < backup.sql

# Restore with progress
pv backup.sql | psql mydb

# Restore and log errors
psql mydb < backup.sql 2> restore_errors.log

# Restore with single transaction (all-or-nothing)
psql mydb --single-transaction < backup.sql
```

**Custom Format:**
```bash
# Restore full database
pg_restore -d mydb backup.dump

# Parallel restore (4 jobs)
pg_restore -d mydb -j 4 backup.dump

# Restore specific tables
pg_restore -d mydb -t users -t orders backup.dump

# Restore specific schema
pg_restore -d mydb -n public backup.dump

# List contents without restoring
pg_restore -l backup.dump

# Create custom restore list
pg_restore -l backup.dump > restore.list
# Edit restore.list to exclude unwanted objects
pg_restore -d mydb -L restore.list backup.dump
```

**Directory Format:**
```bash
# Parallel restore from directory
pg_restore -d mydb -j 4 backup_dir/

# Restore specific tables
pg_restore -d mydb -t users backup_dir/
```

### Restore Best Practices

**1. Create database first:**
```bash
# For custom/directory format
psql -c "CREATE DATABASE mydb;"
pg_restore -d mydb backup.dump

# For plain SQL
psql -c "CREATE DATABASE mydb;"
psql mydb < backup.sql
```

**2. Disable triggers during restore:**
```bash
# Faster restore, but be careful with data integrity
pg_restore -d mydb --disable-triggers backup.dump
```

**3. Clean before restore:**
```bash
# Drop existing objects first
pg_restore -d mydb --clean backup.dump
# or
pg_restore -d mydb --clean --if-exists backup.dump  # No errors if objects don't exist
```

**4. Create + Clean (safest):**
```bash
# Drop and recreate database
pg_restore --create --clean -d postgres backup.dump
```

**5. Optimize for large restores:**
```bash
# Before restore: tune parameters
psql mydb <<EOF
ALTER SYSTEM SET maintenance_work_mem = '2GB';
ALTER SYSTEM SET max_wal_size = '10GB';
ALTER SYSTEM SET checkpoint_timeout = '30min';
ALTER SYSTEM SET synchronous_commit = off;
SELECT pg_reload_conf();
EOF

# Restore
pg_restore -d mydb -j 4 backup.dump

# After restore: reset parameters
psql mydb <<EOF
ALTER SYSTEM RESET maintenance_work_mem;
ALTER SYSTEM RESET max_wal_size;
ALTER SYSTEM RESET checkpoint_timeout;
ALTER SYSTEM RESET synchronous_commit;
SELECT pg_reload_conf();
ANALYZE;
EOF
```

### Restore Verification

```bash
# 1. Check row counts
psql mydb -c "
SELECT schemaname, tablename, n_live_tup
FROM pg_stat_user_tables
ORDER BY tablename;
"

# 2. Verify critical tables
psql mydb -c "SELECT count(*) FROM users;"
psql mydb -c "SELECT count(*) FROM orders;"

# 3. Check constraints and indexes
psql mydb -c "\d+ tablename"

# 4. Verify sequences (especially after data-only restore)
psql mydb -c "
SELECT schemaname, sequencename, last_value
FROM pg_sequences;
"

# 5. Update statistics
psql mydb -c "ANALYZE VERBOSE;"
```

## Third-Party Tools

### pgBackRest

**Features:**
- Parallel backup/restore
- Incremental and differential backups
- Built-in encryption and compression
- Automatic WAL archiving
- S3/Azure/GCS support

**Installation:**
```bash
# Ubuntu/Debian
sudo apt install pgbackrest -y

# Configuration: /etc/pgbackrest/pgbackrest.conf
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=4
start-fast=y
log-level-console=info

[main]
pg1-path=/var/lib/postgresql/17/main
pg1-port=5432
```

**Usage:**
```bash
# Full backup
pgbackrest --stanza=main --type=full backup

# Incremental backup
pgbackrest --stanza=main --type=incr backup

# Differential backup
pgbackrest --stanza=main --type=diff backup

# Restore
pgbackrest --stanza=main restore

# PITR
pgbackrest --stanza=main \
  --type=time \
  --target="2024-01-15 14:30:00" \
  restore
```

### Barman (Backup and Recovery Manager)

**Features:**
- Centralized backup management
- Multiple PostgreSQL servers
- WAL streaming
- Hook scripts for custom actions

**Installation:**
```bash
# Ubuntu/Debian
sudo apt install barman -y

# Configure /etc/barman.d/main.conf
[main]
description = "Main Production Server"
conninfo = host=db-server user=barman dbname=postgres
backup_method = postgres
streaming_conninfo = host=db-server user=streaming_barman dbname=postgres
streaming_archiver = on
slot_name = barman
```

**Usage:**
```bash
# Backup
barman backup main

# List backups
barman list-backup main

# Restore
barman recover main latest /var/lib/postgresql/17/main

# PITR
barman recover main latest /var/lib/postgresql/17/main \
  --target-time "2024-01-15 14:30:00"
```

### WAL-G

**Features:**
- S3/Azure/GCS/Swift support
- Delta backups
- Encryption and compression
- Continuous WAL archiving

**Installation:**
```bash
# Download from GitHub
wget https://github.com/wal-g/wal-g/releases/download/v2.0.1/wal-g-pg-ubuntu-20.04-amd64.tar.gz
tar -xzf wal-g-pg-ubuntu-20.04-amd64.tar.gz
sudo mv wal-g-pg-ubuntu-20.04-amd64 /usr/local/bin/wal-g
```

**Configuration:**
```bash
# Environment variables
export AWS_ACCESS_KEY_ID=your_key
export AWS_SECRET_ACCESS_KEY=your_secret
export WALG_S3_PREFIX=s3://backup-bucket/postgres
export PGDATA=/var/lib/postgresql/17/main
```

**Usage:**
```bash
# Backup
wal-g backup-push $PGDATA

# List backups
wal-g backup-list

# Restore
wal-g backup-fetch $PGDATA LATEST
```

## Anti-Patterns

### ❌ Anti-Pattern 1: No Backup Testing
```bash
# BAD: Taking backups but never testing restore
0 2 * * * pg_dump mydb > /backup/mydb.sql
```

**Impact:**
- Corrupted backups discovered during disaster
- Incorrect backup commands
- Missing permissions or dependencies

**Solution:**
```bash
# GOOD: Automated backup testing
0 2 * * * /usr/local/bin/backup_and_test.sh

# backup_and_test.sh
#!/bin/bash
pg_dump -Fc mydb > /backup/mydb_$(date +%Y%m%d).dump
createdb test_restore
pg_restore -d test_restore /backup/mydb_$(date +%Y%m%d).dump
psql test_restore -c "SELECT count(*) FROM critical_table;"
dropdb test_restore
```

### ❌ Anti-Pattern 2: Single Copy Backups
```bash
# BAD: Only one backup copy, same disk
pg_dump mydb > /var/lib/postgresql/backups/mydb.sql
```

**Impact:**
- Disk failure = data loss + backup loss
- Ransomware can encrypt backups

**Solution:**
```bash
# GOOD: 3-2-1 strategy
# 1. Local backup
pg_dump -Fc mydb > /local/backup/mydb.dump

# 2. Remote backup (different server)
scp /local/backup/mydb.dump backup-server:/backups/

# 3. Cloud backup
aws s3 cp /local/backup/mydb.dump s3://my-backups/
```

### ❌ Anti-Pattern 3: Not Testing PITR
```bash
# BAD: WAL archiving enabled but never tested PITR recovery
archive_command = 'cp %p /archive/%f'
```

**Impact:**
- Broken archive_command discovered during disaster
- Incorrect recovery procedures
- Data loss despite having backups

**Solution:**
```bash
# GOOD: Monthly PITR test
# 1. Take base backup
# 2. Create restore point
# 3. Make some changes
# 4. Perform PITR to restore point
# 5. Verify data
```

### ❌ Anti-Pattern 4: Using pg_dump for Large Databases
```bash
# BAD: 500GB database, pg_dump takes 8 hours
pg_dump -Fc huge_database > backup.dump  # 8 hours
pg_restore -d target huge_database backup.dump  # 12 hours
```

**Impact:**
- Long backup windows
- Long recovery time
- Database size grows faster than backup speed

**Solution:**
```bash
# GOOD: Use physical backups for large databases
pg_basebackup -D /backup/base -Ft -z -P -j 4  # 45 minutes
# Restore: extract tar, start PostgreSQL  # 10 minutes
```

## Monitoring

### Backup Monitoring Queries

```sql
-- Last successful backup (custom tracking table)
CREATE TABLE backup_log (
    backup_id SERIAL PRIMARY KEY,
    backup_type TEXT,
    backup_size BIGINT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    status TEXT,
    error_message TEXT
);

-- Log backup
INSERT INTO backup_log (backup_type, backup_size, started_at, completed_at, status)
VALUES ('pg_dump', 12345678, '2024-01-15 02:00:00', '2024-01-15 02:30:00', 'success');

-- Check recent backups
SELECT backup_type,
       pg_size_pretty(backup_size) as size,
       completed_at,
       completed_at - started_at as duration,
       status
FROM backup_log
ORDER BY completed_at DESC
LIMIT 10;

-- Alert: No backup in 24 hours
SELECT 'CRITICAL: No backup in 24 hours' as alert
WHERE NOT EXISTS (
    SELECT 1 FROM backup_log
    WHERE completed_at > now() - interval '24 hours'
      AND status = 'success'
);
```

### WAL Archive Monitoring

```sql
-- Archive status
SELECT archived_count,
       failed_count,
       last_archived_wal,
       last_archived_time,
       now() - last_archived_time as archive_lag
FROM pg_stat_archiver;

-- WAL files pending archive
SELECT count(*) as pending_wal_files
FROM pg_ls_archive_statusdir()
WHERE name LIKE '%.ready';

-- Alert: Archive lag > 10 minutes
SELECT 'WARNING: Archive lag exceeded' as alert,
       now() - last_archived_time as lag
FROM pg_stat_archiver
WHERE now() - last_archived_time > interval '10 minutes';
```

### Backup Size Trends

```sql
-- Backup size growth
SELECT date_trunc('week', completed_at) as week,
       avg(backup_size) as avg_size,
       pg_size_pretty(avg(backup_size)::bigint) as avg_size_pretty
FROM backup_log
WHERE backup_type = 'pg_dump'
  AND status = 'success'
GROUP BY week
ORDER BY week DESC;
```

## Real-World Scenarios

### Scenario 1: Daily Automated Backup Script

```bash
#!/bin/bash
# /usr/local/bin/postgres_backup.sh

set -e

# Configuration
DB_NAME="production"
BACKUP_DIR="/backups/postgresql"
RETENTION_DAYS=30
S3_BUCKET="s3://my-backups/postgresql"
LOG_FILE="/var/log/postgres_backup.log"

# Create backup directory
mkdir -p $BACKUP_DIR

# Log start
echo "$(date): Starting backup of $DB_NAME" >> $LOG_FILE

# Timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump"

# Perform backup
if pg_dump -Fc -Z9 -U postgres $DB_NAME > $BACKUP_FILE; then
    echo "$(date): Backup completed successfully" >> $LOG_FILE

    # Calculate size
    SIZE=$(stat -f%z "$BACKUP_FILE" 2>/dev/null || stat -c%s "$BACKUP_FILE")
    echo "$(date): Backup size: $(numfmt --to=iec $SIZE)" >> $LOG_FILE

    # Upload to S3
    if aws s3 cp $BACKUP_FILE $S3_BUCKET/; then
        echo "$(date): Uploaded to S3" >> $LOG_FILE
    else
        echo "$(date): ERROR: S3 upload failed" >> $LOG_FILE
        exit 1
    fi

    # Test backup integrity
    if pg_restore -l $BACKUP_FILE > /dev/null 2>&1; then
        echo "$(date): Backup integrity verified" >> $LOG_FILE
    else
        echo "$(date): ERROR: Backup is corrupted" >> $LOG_FILE
        exit 1
    fi

    # Clean old backups
    find $BACKUP_DIR -name "${DB_NAME}_*.dump" -mtime +$RETENTION_DAYS -delete
    echo "$(date): Cleaned backups older than $RETENTION_DAYS days" >> $LOG_FILE

else
    echo "$(date): ERROR: Backup failed" >> $LOG_FILE
    exit 1
fi

echo "$(date): Backup process completed" >> $LOG_FILE
```

**Cron:**
```bash
# Daily at 2 AM
0 2 * * * /usr/local/bin/postgres_backup.sh
```

### Scenario 2: Zero-Downtime Migration to New Server

```bash
# OLD SERVER (10.0.1.10)

# 1. Setup WAL archiving
# postgresql.conf:
wal_level = replica
archive_mode = on
archive_command = 'rsync -a %p new-server:/wal_archive/%f'

# Reload config
pg_ctl reload

# 2. Create replication user
psql -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_pwd';"

# 3. Configure pg_hba.conf
echo "host replication replicator 10.0.1.20/32 scram-sha-256" >> pg_hba.conf
pg_ctl reload

# NEW SERVER (10.0.1.20)

# 4. Create wal_archive directory
mkdir -p /wal_archive
chown postgres:postgres /wal_archive

# 5. Take base backup
pg_basebackup -h 10.0.1.10 -U replicator -D /var/lib/postgresql/17/main -Fp -Xs -P -R

# 6. Start as standby
systemctl start postgresql

# 7. Verify replication lag
psql -c "SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;"

# 8. When ready to switch, stop application writes to old server

# 9. Promote new server
pg_ctl promote

# 10. Update application connection string to new server

# 11. Verify
psql -c "SELECT pg_is_in_recovery();"  # Should return false
```

### Scenario 3: Recovering from Dropped Table

```sql
-- 10:45 AM: Someone accidentally drops critical table
DROP TABLE orders;

-- 10:46 AM: Realize mistake, need to recover
```

**Recovery Steps:**
```bash
# 1. Immediately stop accepting writes (if possible)
psql -c "ALTER SYSTEM SET default_transaction_read_only = on;"
psql -c "SELECT pg_reload_conf();"

# 2. Find when table was dropped
psql -c "
SELECT query, query_start
FROM pg_stat_statements
WHERE query LIKE '%DROP TABLE orders%';
"
# Result: 2024-01-15 10:45:23

# 3. Perform PITR to 10:45:00
systemctl stop postgresql
mv /var/lib/postgresql/17/main /var/lib/postgresql/17/main.dropped
tar -xzf /backup/base.tar.gz -C /var/lib/postgresql/17/
touch /var/lib/postgresql/17/main/recovery.signal

cat > /var/lib/postgresql/17/main/postgresql.auto.conf <<EOF
restore_command = 'cp /wal_archive/%f %p'
recovery_target_time = '2024-01-15 10:45:00'
recovery_target_action = 'pause'
EOF

systemctl start postgresql

# 4. Verify table exists
psql -c "SELECT count(*) FROM orders;"

# 5. Export table data
pg_dump -t orders -Fc postgres > orders_recovered.dump

# 6. Restore main database
systemctl stop postgresql
rm -rf /var/lib/postgresql/17/main
mv /var/lib/postgresql/17/main.dropped /var/lib/postgresql/17/main
systemctl start postgresql

# 7. Restore only orders table
pg_restore -t orders -d postgres orders_recovered.dump

# 8. Verify
psql -c "SELECT count(*) FROM orders;"

# 9. Re-enable writes
psql -c "ALTER SYSTEM SET default_transaction_read_only = off;"
psql -c "SELECT pg_reload_conf();"
```

---

## Quick Reference

### Backup Commands
```bash
# Logical backups
pg_dump -Fc mydb > backup.dump
pg_dumpall > cluster.sql

# Physical backups
pg_basebackup -D /backup -Ft -z -P -X stream

# Restore
pg_restore -d mydb backup.dump
psql mydb < backup.sql
```

### PITR Recovery
```bash
# 1. Restore base backup
# 2. Create recovery.signal
# 3. Set restore_command and recovery_target_time
# 4. Start PostgreSQL
```

### Next Steps
- [← 03 - WAL Architecture](../03-wal-architecture/README.md)
- [05 - Statistics →](../05-statistics/README.md)
- [Back to Main](../README.md)
