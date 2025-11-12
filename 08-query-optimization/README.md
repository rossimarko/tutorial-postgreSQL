# 08 - Query Optimization

## Table of Contents
- [Join Methods](#join-methods)
- [CTEs vs Subqueries](#ctes-vs-subqueries)
- [Parallel Query Execution](#parallel-query-execution)
- [JIT Compilation](#jit-compilation)
- [Query Rewriting](#query-rewriting)
- [Optimization Techniques](#optimization-techniques)
- [Real-World Scenarios](#real-world-scenarios)

## Join Methods

### Nested Loop Join

**Best for:** Small datasets, indexed join columns

```sql
-- Example
EXPLAIN ANALYZE
SELECT u.username, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.id = 100;

-- Plan:
Nested Loop  (cost=0.86..16.91 rows=3 width=20)
  ->  Index Scan using users_pkey on users u
        Index Cond: (id = 100)
  ->  Index Scan using idx_orders_user_id on orders o
        Index Cond: (user_id = 100)
```

**Performance:** O(n × m)
- Outer: 1 row → Inner: 3 index lookups = 4 total lookups

### Hash Join

**Best for:** Equality joins, medium-large datasets, no suitable index

```sql
EXPLAIN ANALYZE
SELECT u.username, o.total
FROM users u
JOIN orders o ON u.id = o.user_id;

-- Plan:
Hash Join  (cost=1234.00..5678.00 rows=10000 width=20)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o
  ->  Hash
        ->  Seq Scan on users u
```

**Performance:** O(n + m)
1. Build hash table from smaller table (users)
2. Probe with larger table (orders)

**Tuning:**
```sql
-- Increase work_mem for larger hash tables
SET work_mem = '256MB';
```

### Merge Join

**Best for:** Pre-sorted data, large datasets, sorted indexes

```sql
EXPLAIN ANALYZE
SELECT u.username, o.total
FROM users u
JOIN orders o ON u.id = o.user_id
ORDER BY u.id;

-- Plan:
Merge Join  (cost=0.85..4567.00 rows=10000 width=20)
  Merge Cond: (u.id = o.user_id)
  ->  Index Scan using users_pkey on users u
  ->  Index Scan using idx_orders_user_id on orders o
```

**Performance:** O(n + m) with sorted data

## CTEs vs Subqueries

### Common Table Expressions (CTEs)

```sql
-- CTE (Optimization Fence in PostgreSQL < 12)
WITH active_users AS (
    SELECT id, username
    FROM users
    WHERE status = 'active'
)
SELECT * FROM active_users
WHERE username LIKE 'admin%';
```

**PostgreSQL < 12:** CTE materialized (optimization fence)
**PostgreSQL >= 12:** CTE inlined by default (optimizable)

### Materialized CTEs (PostgreSQL 12+)

```sql
-- Force materialization
WITH active_users AS MATERIALIZED (
    SELECT id, username
    FROM users
    WHERE status = 'active'
)
SELECT * FROM active_users
WHERE username LIKE 'admin%';

-- Prevent materialization (inline)
WITH active_users AS NOT MATERIALIZED (
    SELECT id, username
    FROM users
    WHERE status = 'active'
)
SELECT * FROM active_users
WHERE username LIKE 'admin%';
```

### Performance Comparison

```sql
-- Subquery (always inlined)
EXPLAIN ANALYZE
SELECT * FROM (
    SELECT id, username
    FROM users
    WHERE status = 'active'
) active_users
WHERE username LIKE 'admin%';

-- Result: Filters pushed down, efficient
Index Scan on users
  Filter: (status = 'active' AND username LIKE 'admin%')

-- Materialized CTE
WITH active_users AS MATERIALIZED (
    SELECT id, username FROM users WHERE status = 'active'
)
SELECT * FROM active_users WHERE username LIKE 'admin%';

-- Result: Two separate operations
CTE Scan on active_users
  Filter: (username LIKE 'admin%')
  CTE active_users
    ->  Seq Scan on users
          Filter: (status = 'active')
```

**When to materialize:**
- CTE referenced multiple times
- Expensive computation reused
- Prevent re-execution

## Parallel Query Execution

### Configuration

```sql
-- Max parallel workers per query
SET max_parallel_workers_per_gather = 4;

-- Total parallel workers system-wide
SET max_parallel_workers = 8;

-- Background worker processes
SET max_worker_processes = 8;

-- Minimum table size for parallel scan
SET min_parallel_table_scan_size = '8MB';

-- Minimum index size for parallel scan
SET min_parallel_index_scan_size = '512kB';
```

### Parallel Query Example

```sql
-- Sequential execution
EXPLAIN ANALYZE
SELECT count(*) FROM large_table;

-- Result:
Aggregate  (cost=169248.00..169248.01 rows=1 width=8)
  ->  Seq Scan on large_table
        (cost=0.00..144248.00 rows=10000000 width=0)
Planning Time: 0.123 ms
Execution Time: 3456.789 ms

-- Enable parallel
SET max_parallel_workers_per_gather = 4;

EXPLAIN ANALYZE
SELECT count(*) FROM large_table;

-- Result:
Finalize Aggregate  (cost=85432.12..85432.13 rows=1 width=8)
  ->  Gather  (cost=85432.00..85432.11 rows=4 width=8)
        Workers Planned: 4
        Workers Launched: 4
        ->  Partial Aggregate
              ->  Parallel Seq Scan on large_table
                    (cost=0.00..75432.00 rows=2500000 width=0)
Planning Time: 0.145 ms
Execution Time: 982.345 ms  (3.5x faster!)
```

### Operations Supporting Parallelism

- Sequential Scans
- Index Scans (PostgreSQL 10+)
- Index-Only Scans
- Bitmap Heap Scans
- Joins (Hash, Merge, Nested Loop)
- Aggregates
- CTEs (PostgreSQL 11+)

### Forcing Parallel Execution

```sql
-- Disable parallel safety check
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;
SET min_parallel_table_scan_size = 0;

-- Force parallel for testing
EXPLAIN ANALYZE
SELECT /*+ Parallel(users 4) */ count(*) FROM users;
```

## JIT Compilation

### What is JIT?

Just-In-Time compilation converts query execution into native machine code for:
- Expression evaluation
- Tuple deforming
- Aggregate functions

### Configuration

```sql
-- Enable JIT (default: on)
SET jit = on;

-- JIT cost threshold
SET jit_above_cost = 100000;  -- Use JIT for expensive queries

-- Inline functions
SET jit_inline_above_cost = 500000;

-- Optimize expressions
SET jit_optimize_above_cost = 500000;
```

### JIT Performance Example

```sql
-- Expensive aggregation query
EXPLAIN (ANALYZE, VERBOSE)
SELECT
    category,
    sum(price * quantity) as total,
    avg(price),
    count(*)
FROM sales
WHERE price > 10
GROUP BY category;

-- Without JIT:
Execution Time: 2345.678 ms

-- With JIT:
Execution Time: 1234.567 ms
JIT:
  Functions: 12
  Options: Inlining true, Optimization true, Expressions true
  Timing: Generation 15.234 ms, Inlining 8.567 ms, Optimization 45.123 ms, Emission 23.456 ms, Total 92.380 ms

-- Net speedup: ~47% (2345ms → 1234ms)
```

### When JIT Helps

✅ **Good candidates:**
- Complex expressions
- Large aggregations
- Many computed columns
- Long-running analytical queries

❌ **Poor candidates:**
- Simple queries
- Index lookups
- Small result sets
- OLTP workloads (overhead > benefit)

## Query Rewriting

### Use EXISTS Instead of IN

```sql
-- Slow: IN with subquery
SELECT * FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- Fast: EXISTS
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id AND o.total > 1000
);
```

### Avoid SELECT *

```sql
-- Inefficient
SELECT * FROM large_table WHERE id = 100;
-- Reads all columns (50+ fields, 500 bytes)

-- Efficient
SELECT id, name, email FROM large_table WHERE id = 100;
-- Reads only needed columns (3 fields, 100 bytes)
```

### Push Down Filters

```sql
-- Inefficient: Filter after join
SELECT *
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active' AND o.total > 1000;
-- Joins all users, then filters

-- Efficient: Filter before join (planner often does this automatically)
SELECT *
FROM (SELECT id FROM users WHERE status = 'active') u
JOIN (SELECT user_id, total FROM orders WHERE total > 1000) o
  ON u.id = o.user_id;
```

### Avoid Functions on Indexed Columns

```sql
-- Doesn't use index
SELECT * FROM users WHERE LOWER(email) = 'admin@example.com';

-- Solution 1: Don't transform
SELECT * FROM users WHERE email = 'admin@example.com';

-- Solution 2: Expression index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
```

## Optimization Techniques

### 1. Index Optimization

```sql
-- Check missing indexes
SELECT schemaname, tablename, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 1000 AND idx_scan < seq_scan
ORDER BY seq_tup_read DESC;

-- Create covering index
CREATE INDEX idx_users_cover ON users(status, email, created_at);

-- Now index-only scan
EXPLAIN SELECT email FROM users WHERE status = 'active';
-- Index Only Scan using idx_users_cover
```

### 2. LIMIT Optimization

```sql
-- Inefficient: Sort all, return 10
SELECT * FROM orders ORDER BY created_at DESC;
-- Sorts 1M rows

-- Efficient: Use index + limit
CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);

SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- Index Scan (top-N heap sort avoided)
```

### 3. Batch Operations

```sql
-- Slow: Row-by-row
DO $$
BEGIN
    FOR i IN 1..10000 LOOP
        INSERT INTO logs (message) VALUES ('Log ' || i);
    END LOOP;
END $$;
-- Time: 15 seconds

-- Fast: Batch insert
INSERT INTO logs (message)
SELECT 'Log ' || g FROM generate_series(1, 10000) g;
-- Time: 0.5 seconds (30x faster!)
```

### 4. Denormalization for Read Performance

```sql
-- Normalized: Join every query
SELECT u.username, count(o.id)
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id;

-- Denormalized: Pre-computed
ALTER TABLE users ADD COLUMN order_count INT DEFAULT 0;

-- Update with trigger
CREATE OR REPLACE FUNCTION update_order_count()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE users SET order_count = order_count + 1 WHERE id = NEW.user_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_order_count
AFTER INSERT ON orders
FOR EACH ROW EXECUTE FUNCTION update_order_count();

-- Fast query
SELECT username, order_count FROM users;
```

## Real-World Scenarios

### Scenario 1: Slow Reporting Query

**Problem:**
```sql
SELECT
    date_trunc('day', o.created_at) as day,
    count(*),
    sum(o.total)
FROM orders o
WHERE o.created_at >= '2024-01-01'
GROUP BY day
ORDER BY day;

-- Execution: 45 seconds
```

**Optimization:**
```sql
-- 1. Create index
CREATE INDEX idx_orders_created_day ON orders(date_trunc('day', created_at));

-- 2. Enable parallel
SET max_parallel_workers_per_gather = 4;

-- 3. Materialized view for frequent queries
CREATE MATERIALIZED VIEW daily_sales AS
SELECT
    date_trunc('day', created_at) as day,
    count(*) as order_count,
    sum(total) as total_sales
FROM orders
GROUP BY day;

CREATE INDEX idx_daily_sales_day ON daily_sales(day);

-- Refresh nightly
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;

-- Fast query
SELECT * FROM daily_sales WHERE day >= '2024-01-01';
-- Execution: 0.05 seconds (900x faster!)
```

### Scenario 2: N+1 Query Problem

**Problem:**
```python
# Application code
for user in User.all():
    orders = Order.where(user_id=user.id)
    print(f"{user.name}: {len(orders)}")

# Generates:
# SELECT * FROM users;  -- 1 query
# SELECT * FROM orders WHERE user_id = 1;  -- N queries
# SELECT * FROM orders WHERE user_id = 2;
# ...
# Total: 1 + N queries
```

**Solution:**
```python
# Eager loading with JOIN
users_with_orders = """
SELECT
    u.id,
    u.name,
    count(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name
"""
# 1 query total
```

---

## Quick Reference

```sql
-- Join hints
SET enable_nestloop = off;  -- Disable nested loop
SET enable_hashjoin = on;   -- Enable hash join
SET enable_mergejoin = on;  -- Enable merge join

-- Parallel execution
SET max_parallel_workers_per_gather = 4;

-- JIT
SET jit = on;
SET jit_above_cost = 100000;

-- Work memory (per operation)
SET work_mem = '256MB';
```

### Next Steps
- [← 07 - Query Plans](../07-query-plans/README.md)
- [09 - VACUUM and Autovacuum →](../09-vacuum-autovacuum/README.md)
- [Back to Main](../README.md)
