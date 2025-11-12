# PostgreSQL DBA Scripts

Collection of monitoring, maintenance, and automation scripts for PostgreSQL administration.

## Monitoring Scripts

### 1. Health Check (`health_check.sql`)

```sql
-- health_check.sql
-- Comprehensive database health check

\echo '=== PostgreSQL Health Check ==='
\echo ''

\echo '1. Connection Stats'
SELECT
    count(*) as total_connections,
    count(*) FILTER (WHERE state = 'active') as active,
    count(*) FILTER (WHERE state = 'idle') as idle,
    max_conn as max_connections,
    round(100.0 * count(*) / max_conn, 2) as pct_used
FROM pg_stat_activity,
     (SELECT setting::int as max_conn FROM pg_settings WHERE name = 'max_connections') s;

\echo ''
\echo '2. Cache Hit Ratio (should be > 99%)'
SELECT
    round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit + blks_read), 0), 2) as cache_hit_ratio
FROM pg_stat_database;

\echo ''
\echo '3. Database Size'
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
WHERE datname = current_database();

\echo ''
\echo '4. Top 10 Largest Tables'
SELECT
    schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;

\echo ''
\echo '5. Tables with Bloat (>10% dead tuples)'
SELECT
    schemaname, tablename,
    n_dead_tup, n_live_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) as bloat_pct
FROM pg_stat_user_tables
WHERE n_dead_tup::FLOAT / NULLIF(n_live_tup, 0) > 0.1
  AND n_live_tup > 1000
ORDER BY bloat_pct DESC;

\echo ''
\echo '6. Unused Indexes'
SELECT
    schemaname, tablename, indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 10;

\echo ''
\echo '7. Long Running Queries'
SELECT
    pid, usename,
    now() - query_start as duration,
    state, query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '1 minute'
ORDER BY duration DESC;

\echo ''
\echo '=== Health Check Complete ==='
```

### 2. Slow Query Monitor (`slow_queries.sql`)

```sql
-- slow_queries.sql
-- Requires pg_stat_statements extension

SELECT
    substring(query, 1, 100) as query_short,
    calls,
    round(mean_exec_time::numeric, 2) as avg_time_ms,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) as pct_total_time,
    rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### 3. Blocking Queries (`blocking.sql`)

```sql
-- blocking.sql
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query,
    blocked_activity.application_name AS blocked_app,
    blocking_activity.application_name AS blocking_app
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

## Maintenance Scripts

### 4. Backup Script (`backup.sh`)

```bash
#!/bin/bash
# backup.sh - Daily PostgreSQL backup script

set -e

# Configuration
DB_NAME="mydb"
BACKUP_DIR="/backups/postgresql"
RETENTION_DAYS=30
S3_BUCKET="s3://my-backups/postgresql"
LOG_FILE="/var/log/postgres_backup.log"

# Create backup directory
mkdir -p $BACKUP_DIR

# Timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${TIMESTAMP}.dump"

# Log function
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

log "Starting backup of $DB_NAME"

# Perform backup
if pg_dump -Fc -Z9 -U postgres $DB_NAME > $BACKUP_FILE; then
    SIZE=$(stat -f%z "$BACKUP_FILE" 2>/dev/null || stat -c%s "$BACKUP_FILE")
    log "Backup completed: $(numfmt --to=iec $SIZE)"

    # Upload to S3
    if command -v aws &> /dev/null; then
        log "Uploading to S3..."
        aws s3 cp $BACKUP_FILE $S3_BUCKET/
        log "Upload complete"
    fi

    # Test backup
    log "Testing backup integrity..."
    if pg_restore -l $BACKUP_FILE > /dev/null 2>&1; then
        log "Backup integrity verified"
    else
        log "ERROR: Backup is corrupted!"
        exit 1
    fi

    # Clean old backups
    log "Cleaning backups older than $RETENTION_DAYS days..."
    find $BACKUP_DIR -name "${DB_NAME}_*.dump" -mtime +$RETENTION_DAYS -delete
    log "Cleanup complete"

else
    log "ERROR: Backup failed!"
    exit 1
fi

log "Backup process completed successfully"
```

### 5. Vacuum Script (`vacuum_maintenance.sh`)

```bash
#!/bin/bash
# vacuum_maintenance.sh - Regular VACUUM maintenance

set -e

DB_NAME="mydb"
LOG_FILE="/var/log/vacuum_maintenance.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

log "Starting VACUUM maintenance"

# VACUUM ANALYZE all tables
log "Running VACUUM ANALYZE..."
vacuumdb -U postgres -d $DB_NAME -z -v >> $LOG_FILE 2>&1

# Check for bloated tables
log "Checking for bloated tables..."
psql -U postgres $DB_NAME -c "
SELECT tablename, n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) as bloat_pct
FROM pg_stat_user_tables
WHERE n_dead_tup::FLOAT / NULLIF(n_live_tup, 0) > 0.1
ORDER BY bloat_pct DESC;
" | tee -a $LOG_FILE

log "VACUUM maintenance completed"
```

### 6. Reindex Script (`reindex_maintenance.sh`)

```bash
#!/bin/bash
# reindex_maintenance.sh - Rebuild indexes concurrently

set -e

DB_NAME="mydb"
LOG_FILE="/var/log/reindex_maintenance.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

log "Starting REINDEX maintenance"

# Get list of indexes
INDEXES=$(psql -U postgres $DB_NAME -t -c "
SELECT indexname
FROM pg_indexes
WHERE schemaname = 'public'
  AND indexname NOT LIKE '%_pkey';
")

# Reindex each index concurrently
for index in $INDEXES; do
    log "Reindexing $index..."
    psql -U postgres $DB_NAME -c "REINDEX INDEX CONCURRENTLY $index;" >> $LOG_FILE 2>&1
    log "Completed $index"
done

log "REINDEX maintenance completed"
```

## Automation Scripts

### 7. Auto-Partition Creator (`create_partitions.sql`)

```sql
-- create_partitions.sql
-- Automatically create next month's partition

DO $$
DECLARE
    next_month DATE;
    next_month_end DATE;
    partition_name TEXT;
BEGIN
    -- Calculate next month
    next_month := date_trunc('month', CURRENT_DATE + interval '1 month');
    next_month_end := next_month + interval '1 month';

    -- Partition name
    partition_name := 'sales_' || to_char(next_month, 'YYYY_MM');

    -- Check if partition exists
    IF NOT EXISTS (
        SELECT 1 FROM pg_tables WHERE tablename = partition_name
    ) THEN
        -- Create partition
        EXECUTE format('
            CREATE TABLE %I PARTITION OF sales
            FOR VALUES FROM (%L) TO (%L)
        ', partition_name, next_month, next_month_end);

        RAISE NOTICE 'Created partition: %', partition_name;
    ELSE
        RAISE NOTICE 'Partition already exists: %', partition_name;
    END IF;
END $$;
```

### 8. Stats Update Script (`update_stats.sh`)

```bash
#!/bin/bash
# update_stats.sh - Update statistics for stale tables

set -e

DB_NAME="mydb"
LOG_FILE="/var/log/update_stats.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

log "Checking for stale statistics..."

# Find tables with stale stats
STALE_TABLES=$(psql -U postgres $DB_NAME -t -c "
SELECT tablename
FROM pg_stat_user_tables
WHERE n_mod_since_analyze > 10000
   OR (last_analyze IS NULL AND last_autoanalyze IS NULL AND n_live_tup > 1000);
")

if [ -z "$STALE_TABLES" ]; then
    log "No stale statistics found"
    exit 0
fi

# Analyze stale tables
for table in $STALE_TABLES; do
    log "Analyzing $table..."
    psql -U postgres $DB_NAME -c "ANALYZE VERBOSE $table;" >> $LOG_FILE 2>&1
done

log "Statistics update completed"
```

## Usage

```bash
# Health check
psql -U postgres mydb -f scripts/health_check.sql

# Slow queries
psql -U postgres mydb -f scripts/slow_queries.sql

# Daily backup
./scripts/backup.sh

# Weekly vacuum
./scripts/vacuum_maintenance.sh

# Monthly reindex
./scripts/reindex_maintenance.sh
```

## Cron Schedule

```cron
# Daily backup at 2 AM
0 2 * * * /path/to/scripts/backup.sh

# Weekly VACUUM on Sunday at 3 AM
0 3 * * 0 /path/to/scripts/vacuum_maintenance.sh

# Monthly REINDEX on 1st at 4 AM
0 4 1 * * /path/to/scripts/reindex_maintenance.sh

# Daily stats update at 6 AM
0 6 * * * /path/to/scripts/update_stats.sh

# Create next month's partitions on 25th
0 0 25 * * psql -U postgres mydb -f /path/to/scripts/create_partitions.sql
```

---

**Next:** [Sample Databases →](../sample-databases/README.md) | [Back to Main](../README.md)
