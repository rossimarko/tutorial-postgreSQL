# 10 - Routine Maintenance

## REINDEX

### When to REINDEX

- Index bloat from many updates/deletes
- Corrupted indexes
- After major data changes
- Performance degradation

```sql
-- Rebuild single index
REINDEX INDEX idx_users_email;

-- Rebuild all indexes on table
REINDEX TABLE users;

-- Rebuild all indexes in schema
REINDEX SCHEMA public;

-- Rebuild all indexes in database
REINDEX DATABASE mydb;

-- Concurrent rebuild (PostgreSQL 12+, no blocking)
REINDEX INDEX CONCURRENTLY idx_users_email;
```

### Index Bloat Detection

```sql
SELECT
    schemaname, tablename, indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size,
    idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- If large size + low scans → consider dropping
-- If large size + high scans → REINDEX
```

## CLUSTER

### Table Clustering

Physically reorder table rows to match index order:

```sql
-- Cluster table by index
CLUSTER users USING users_pkey;

-- Subsequent clustering uses same index
CLUSTER users;

-- Cluster all tables
CLUSTER;
```

**Benefits:**
- Improved cache efficiency
- Faster range scans
- Better compression

**Drawbacks:**
- Exclusive lock (table unavailable)
- Slow on large tables
- One-time benefit (degrades with updates)

```sql
-- Check clustering correlation
SELECT tablename, attname, correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'created_at';

-- correlation near 1.0 = well clustered
-- correlation near 0.0 = random
```

## Table Bloat Management

### Detecting Bloat

```sql
SELECT
    schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) as dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY n_dead_tup DESC;
```

### Removing Bloat

```sql
-- Option 1: VACUUM (online, partial)
VACUUM users;

-- Option 2: VACUUM FULL (offline, complete)
VACUUM FULL users;  -- Locks table!

-- Option 3: pg_repack (online, complete)
-- Install extension first
CREATE EXTENSION pg_repack;
-- Run from command line
pg_repack -t users mydb

-- Option 4: Create new table + switch
CREATE TABLE users_new AS SELECT * FROM users;
ALTER TABLE users RENAME TO users_old;
ALTER TABLE users_new RENAME TO users;
-- Recreate indexes, constraints, etc.
DROP TABLE users_old;
```

## TOAST Management

### What is TOAST?

TOAST (The Oversized-Attribute Storage Technique) stores large values outside main table.

```sql
-- Find TOAST usage
SELECT
    schemaname, tablename,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                   pg_relation_size(schemaname||'.'||tablename)) as toast_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- VACUUM TOAST separately
VACUUM pg_toast.pg_toast_12345;
```

## Statistics Maintenance

```sql
-- Update statistics
ANALYZE users;

-- Update all statistics
ANALYZE;

-- Increase statistics target for critical columns
ALTER TABLE users ALTER COLUMN email SET STATISTICS 1000;
ANALYZE users;
```

## Maintenance Scripts

### Daily Maintenance

```sql
-- Daily routine
DO $$
BEGIN
    -- Update statistics
    ANALYZE;

    -- Log maintenance
    RAISE NOTICE 'Daily maintenance completed at %', now();
END $$;
```

### Weekly Maintenance

```bash
#!/bin/bash
# weekly_maintenance.sh

# VACUUM all databases
vacuumdb -U postgres -a -z -v

# REINDEX critical tables
psql -U postgres mydb -c "REINDEX TABLE CONCURRENTLY users;"
psql -U postgres mydb -c "REINDEX TABLE CONCURRENTLY orders;"

# Check for bloat
psql -U postgres mydb -c "
SELECT tablename, n_dead_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > 100000;
"
```

## Monitoring Maintenance

```sql
-- Last maintenance times
SELECT
    schemaname, tablename,
    last_vacuum, last_autovacuum,
    last_analyze, last_autoanalyze,
    n_mod_since_analyze
FROM pg_stat_user_tables
ORDER BY n_mod_since_analyze DESC;

-- Tables never analyzed
SELECT tablename
FROM pg_stat_user_tables
WHERE last_analyze IS NULL
  AND last_autoanalyze IS NULL
  AND n_live_tup > 1000;
```

---

**Next:** [11 - Monitoring Views →](../11-monitoring-views/README.md)
