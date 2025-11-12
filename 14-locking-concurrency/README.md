# 14 - Locking and Concurrency

## MVCC (Multi-Version Concurrency Control)

### How MVCC Works

```
Transaction 1 reads row (version 1)
Transaction 2 updates row (creates version 2)
Transaction 1 still sees version 1 (snapshot isolation)
Transaction 2 sees version 2
```

**Benefits:**
- Readers don't block writers
- Writers don't block readers
- High concurrency

## Lock Types

### Row-Level Locks

```sql
-- FOR UPDATE: Exclusive lock
SELECT * FROM users WHERE id = 100 FOR UPDATE;
-- Blocks other FOR UPDATE, allows SELECT

-- FOR SHARE: Shared lock
SELECT * FROM users WHERE id = 100 FOR SHARE;
-- Allows other FOR SHARE, blocks FOR UPDATE

-- FOR NO KEY UPDATE: For updates not changing key
SELECT * FROM users WHERE id = 100 FOR NO KEY UPDATE;

-- FOR KEY SHARE: Minimal lock
SELECT * FROM users WHERE id = 100 FOR KEY SHARE;
```

### Table-Level Locks

```sql
-- ACCESS SHARE: SELECT acquires this
SELECT * FROM users;

-- ROW EXCLUSIVE: INSERT, UPDATE, DELETE acquire this
INSERT INTO users VALUES (...);

-- SHARE: CREATE INDEX CONCURRENTLY acquires this
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- EXCLUSIVE: REFRESH MATERIALIZED VIEW acquires this
REFRESH MATERIALIZED VIEW user_stats;

-- ACCESS EXCLUSIVE: ALTER TABLE, DROP TABLE acquire this
ALTER TABLE users ADD COLUMN age INT;
```

### Lock Compatibility Matrix

| Current Lock | SELECT | INSERT/UPDATE | ALTER TABLE |
|--------------|--------|---------------|-------------|
| None         | ✅      | ✅             | ✅           |
| SELECT       | ✅      | ✅             | ❌           |
| INSERT/UPDATE| ✅      | ✅             | ❌           |
| ALTER TABLE  | ❌      | ❌             | ❌           |

## Deadlocks

### Detecting Deadlocks

```sql
-- Transaction 1
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

-- Transaction 2 (concurrent)
BEGIN;
UPDATE accounts SET balance = balance - 50 WHERE id = 2;  -- Waits for T1
UPDATE accounts SET balance = balance + 50 WHERE id = 1;  -- DEADLOCK!
COMMIT;

-- PostgreSQL detects and aborts one transaction:
-- ERROR: deadlock detected
```

### Deadlock Prevention

```sql
-- Solution: Lock in consistent order
-- Both transactions lock id=1 first, then id=2
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

### Monitor Deadlocks

```sql
-- Check deadlock count
SELECT datname, deadlocks
FROM pg_stat_database
WHERE datname = current_database();

-- Log deadlocks
-- In postgresql.conf:
-- log_lock_waits = on
-- deadlock_timeout = 1s
```

## Lock Monitoring

### Current Locks

```sql
SELECT
    l.pid,
    l.locktype,
    l.relation::regclass,
    l.mode,
    l.granted,
    a.query
FROM pg_locks l
LEFT JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.pid != pg_backend_pid()
ORDER BY l.pid;
```

### Blocking Queries

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON
    blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.database IS NOT DISTINCT FROM blocking_locks.database
    AND blocked_locks.relation IS NOT DISTINCT FROM blocking_locks.relation
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_stat_activity blocking ON blocking.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted
  AND blocking_locks.granted;
```

### Lock Waits

```sql
-- Queries waiting for locks
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

## Advisory Locks

### Application-Level Locking

```sql
-- Acquire advisory lock
SELECT pg_advisory_lock(12345);

-- Try lock (non-blocking)
SELECT pg_try_advisory_lock(12345);  -- Returns true/false

-- Release lock
SELECT pg_advisory_unlock(12345);

-- Use case: Prevent duplicate background jobs
DO $$
BEGIN
    IF NOT pg_try_advisory_lock(12345) THEN
        RAISE NOTICE 'Job already running';
        RETURN;
    END IF;

    -- Run job
    PERFORM long_running_task();

    -- Release lock
    PERFORM pg_advisory_unlock(12345);
END $$;
```

## Isolation Levels

```sql
-- READ COMMITTED (default)
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Sees committed changes from other transactions

-- REPEATABLE READ
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- Consistent snapshot, immune to non-repeatable reads

-- SERIALIZABLE
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Fully isolated, may cause serialization errors

-- Set default
ALTER DATABASE mydb SET default_transaction_isolation TO 'repeatable read';
```

### Isolation Level Comparison

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
|-------|------------|---------------------|--------------|----------------------|
| READ COMMITTED | ❌ | ✅ | ✅ | ✅ |
| REPEATABLE READ | ❌ | ❌ | ❌* | ✅ |
| SERIALIZABLE | ❌ | ❌ | ❌ | ❌ |

*PostgreSQL prevents phantom reads even in REPEATABLE READ

## Best Practices

```sql
-- 1. Keep transactions short
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;  -- Commit immediately

-- 2. Lock in consistent order (prevent deadlocks)
SELECT * FROM table1, table2 WHERE ... ORDER BY table1.id, table2.id FOR UPDATE;

-- 3. Use appropriate lock level
-- Don't use FOR UPDATE when FOR SHARE suffices

-- 4. Avoid long-running transactions
-- Set statement timeout
SET statement_timeout = '30s';

-- 5. Use NOWAIT to avoid blocking
SELECT * FROM users WHERE id = 100 FOR UPDATE NOWAIT;
-- Returns error immediately if locked

-- 6. Use SKIP LOCKED for queue processing
SELECT * FROM jobs WHERE status = 'pending'
ORDER BY id LIMIT 10 FOR UPDATE SKIP LOCKED;
-- Skips locked rows, processes available ones
```

---

**Next:** [15 - Connection Management →](../15-connection-management/README.md)
