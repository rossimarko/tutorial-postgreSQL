# 09 - VACUUM and Autovacuum

## Core Concepts

### Why VACUUM?

PostgreSQL uses MVCC (Multi-Version Concurrency Control):
- UPDATEs create new row versions
- DELETEs mark rows as dead
- Old versions remain until VACUUM reclaims space

```sql
-- Table bloat example
CREATE TABLE test (id INT, data TEXT);
INSERT INTO test SELECT g, 'data' FROM generate_series(1, 100000) g;
-- Size: 7 MB

UPDATE test SET data = 'updated';
-- Size: 14 MB (doubled! Old rows still exist)

VACUUM test;
-- Size: 7 MB (dead rows reclaimed)
```

## VACUUM Commands

```sql
-- Standard VACUUM (reclaims space, doesn't shrink file)
VACUUM users;

-- VACUUM FULL (locks table, rewrites entire table)
VACUUM FULL users;

-- VACUUM with ANALYZE
VACUUM ANALYZE users;

-- VACUUM VERBOSE (shows progress)
VACUUM VERBOSE users;

-- VACUUM all database
VACUUM;
```

### VACUUM vs VACUUM FULL

| Aspect | VACUUM | VACUUM FULL |
|--------|--------|-------------|
| Lock | Allows reads/writes | Exclusive lock (no access) |
| Space | Marks for reuse | Returns to OS |
| Speed | Fast | Slow |
| Downtime | None | Yes |
| Use | Regular maintenance | Emergency bloat removal |

## Autovacuum Configuration

```sql
-- Enable autovacuum (default: on)
autovacuum = on

-- Workers
autovacuum_max_workers = 3

-- Trigger thresholds
autovacuum_vacuum_threshold = 50          -- Min changes
autovacuum_vacuum_scale_factor = 0.2      -- 20% of table

-- Formula: vacuum if (dead_tuples > threshold + scale_factor * table_size)

-- Costs and delays
autovacuum_vacuum_cost_delay = 2ms        -- Throttle to reduce I/O impact
autovacuum_vacuum_cost_limit = 200

-- Anti-wraparound
autovacuum_freeze_max_age = 200000000
```

### Per-Table Tuning

```sql
-- High-churn table: vacuum more frequently
ALTER TABLE orders
SET (autovacuum_vacuum_scale_factor = 0.05);  -- Vacuum at 5% changes

-- Large table: more aggressive autovacuum
ALTER TABLE logs
SET (autovacuum_vacuum_cost_limit = 1000);

-- Disable autovacuum (manual control)
ALTER TABLE staging_table
SET (autovacuum_enabled = false);
```

## Monitoring

```sql
-- Last vacuum/autovacuum times
SELECT schemaname, tablename,
       last_vacuum, last_autovacuum,
       n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) as dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- Tables needing vacuum
SELECT tablename, n_dead_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- Autovacuum currently running
SELECT pid, query_start, query
FROM pg_stat_activity
WHERE query LIKE '%autovacuum%' AND pid != pg_backend_pid();
```

## Bloat Detection

```sql
-- Table bloat estimate
SELECT
    schemaname, tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
    n_dead_tup,
    round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) as bloat_pct
FROM pg_stat_user_tables
WHERE n_live_tup + n_dead_tup > 0
ORDER BY bloat_pct DESC;
```

## Anti-Wraparound VACUUM

```sql
-- Check age
SELECT datname, age(datfrozenxid), datfrozenxid
FROM pg_database
ORDER BY age(datfrozenxid) DESC;

-- Warning if age > 200M
-- Emergency if age > 2B (autovacuum_freeze_max_age)

-- Manual anti-wraparound vacuum
VACUUM FREEZE users;
```

## Best Practices

```sql
-- 1. Monitor autovacuum regularly
SELECT * FROM pg_stat_user_tables WHERE last_autovacuum < now() - interval '1 day';

-- 2. Tune aggressive tables
ALTER TABLE high_churn SET (autovacuum_vacuum_scale_factor = 0.05);

-- 3. Schedule manual VACUUM during low-traffic
VACUUM ANALYZE;

-- 4. Use VACUUM FULL only when necessary (offline maintenance)
VACUUM FULL users;  -- Caution: locks table!
```

---

**Next:** [10 - Routine Maintenance →](../10-routine-maintenance/README.md)
