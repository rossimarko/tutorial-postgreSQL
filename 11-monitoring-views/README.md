# 11 - Monitoring Views

## pg_stat_activity

### Current Database Activity

```sql
-- All active queries
SELECT pid, usename, datname, state, query_start,
       now() - query_start as duration, query
FROM pg_stat_activity
WHERE state = 'active' AND pid != pg_backend_pid()
ORDER BY duration DESC;

-- Idle connections
SELECT count(*), state
FROM pg_stat_activity
GROUP BY state;

-- Long-running queries (> 5 minutes)
SELECT pid, usename, now() - query_start as duration, query
FROM pg_stat_activity
WHERE now() - query_start > interval '5 minutes'
  AND state = 'active';

-- Kill query
SELECT pg_cancel_backend(pid);   -- Graceful
SELECT pg_terminate_backend(pid); -- Force
```

## pg_stat_database

### Database Statistics

```sql
SELECT datname,
       numbackends as connections,
       xact_commit, xact_rollback,
       blks_read, blks_hit,
       round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) as cache_hit_ratio,
       tup_returned, tup_fetched, tup_inserted, tup_updated, tup_deleted,
       conflicts, deadlocks
FROM pg_stat_database
WHERE datname = current_database();
```

## pg_stat_user_tables

### Table Statistics

```sql
-- Table access patterns
SELECT schemaname, tablename,
       seq_scan, seq_tup_read,
       idx_scan, idx_tup_fetch,
       n_tup_ins, n_tup_upd, n_tup_del,
       n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum,
       last_analyze, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY seq_tup_read DESC;

-- Tables with most sequential scans (potential missing indexes)
SELECT tablename, seq_scan, seq_tup_read, idx_scan,
       seq_tup_read / NULLIF(seq_scan, 0) as avg_seq_read
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC;
```

## pg_stat_user_indexes

### Index Usage

```sql
-- Index usage statistics
SELECT schemaname, tablename, indexname,
       idx_scan, idx_tup_read, idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Unused indexes
SELECT schemaname, tablename, indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

## pg_stat_statements

### Query Performance

```sql
CREATE EXTENSION pg_stat_statements;

-- Slowest queries by average time
SELECT substring(query, 1, 100) as query_short,
       calls, mean_exec_time, total_exec_time,
       rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Most frequent queries
SELECT substring(query, 1, 100), calls, mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;

-- Queries with highest total time
SELECT substring(query, 1, 100), calls, total_exec_time, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

## Lock Monitoring

### Current Locks

```sql
SELECT
    l.locktype, l.database, l.relation::regclass,
    l.page, l.tuple, l.virtualxid, l.transactionid,
    l.mode, l.granted,
    a.usename, a.query, a.query_start
FROM pg_locks l
LEFT JOIN pg_stat_activity a ON l.pid = a.pid
WHERE NOT l.granted
ORDER BY l.relation;

-- Blocking queries
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
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

## Connection Monitoring

```sql
-- Connection counts by database
SELECT datname, count(*) as connections, max_conn
FROM pg_stat_activity,
     (SELECT setting::int as max_conn FROM pg_settings WHERE name = 'max_connections') s
GROUP BY datname, max_conn
ORDER BY connections DESC;

-- Connection age
SELECT pid, usename, application_name,
       now() - backend_start as connection_age,
       now() - query_start as query_age,
       state, query
FROM pg_stat_activity
ORDER BY backend_start;

-- Idle in transaction (potential problem)
SELECT pid, usename, now() - state_change as idle_duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY state_change;
```

## Replication Monitoring

```sql
-- Replication status (on primary)
SELECT client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) as send_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), write_lsn) as write_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), flush_lsn) as flush_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as replay_lag
FROM pg_stat_replication;

-- Replication lag (on standby)
SELECT now() - pg_last_xact_replay_timestamp() as replication_lag;
```

## System Monitoring

```sql
-- Table sizes
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                      pg_relation_size(schemaname||'.'||tablename)) as indexes_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- Database sizes
SELECT datname,
       pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- Cache hit ratio
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    round(100.0 * sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables;
```

## Monitoring Dashboard Query

```sql
-- Comprehensive monitoring dashboard
WITH connections AS (
    SELECT count(*) as total,
           count(*) FILTER (WHERE state = 'active') as active,
           count(*) FILTER (WHERE state = 'idle') as idle
    FROM pg_stat_activity
),
cache AS (
    SELECT round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit + blks_read), 0), 2) as hit_ratio
    FROM pg_stat_database
),
locks AS (
    SELECT count(*) as waiting FROM pg_locks WHERE NOT granted
)
SELECT
    (SELECT total FROM connections) as connections_total,
    (SELECT active FROM connections) as connections_active,
    (SELECT idle FROM connections) as connections_idle,
    (SELECT hit_ratio FROM cache) as cache_hit_ratio,
    (SELECT waiting FROM locks) as locks_waiting;
```

---

**Next:** [12 - Performance Tuning →](../12-performance-tuning/README.md)
