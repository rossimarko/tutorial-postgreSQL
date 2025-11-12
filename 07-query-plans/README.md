# 07 - Query Plans

## Table of Contents
- [EXPLAIN Fundamentals](#explain-fundamentals)
- [EXPLAIN ANALYZE](#explain-analyze)
- [Plan Nodes](#plan-nodes)
- [Cost Estimation](#cost-estimation)
- [pg_stat_statements](#pg_stat_statements)
- [Reading Complex Plans](#reading-complex-plans)
- [Monitoring](#monitoring)

## EXPLAIN Fundamentals

```sql
-- Basic EXPLAIN
EXPLAIN SELECT * FROM users WHERE id = 100;

-- EXPLAIN with execution
EXPLAIN ANALYZE SELECT * FROM users WHERE id = 100;

-- Detailed output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, COSTS, TIMING)
SELECT * FROM users WHERE id = 100;
```

### Understanding Output

```sql
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

Seq Scan on users  (cost=0.00..18334.00 rows=1 width=45)
  Filter: (email = 'user@example.com'::text)

-- Breakdown:
-- Seq Scan: Sequential scan (reads every row)
-- cost=0.00..18334.00: Estimated startup cost..total cost
-- rows=1: Estimated rows returned
-- width=45: Average row width in bytes
```

## EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 100;

-- Output:
Index Scan using idx_orders_customer_id on orders
  (cost=0.43..8.45 rows=1 width=100)
  (actual time=0.023..0.025 rows=1 loops=1)
  Buffers: shared hit=4
Planning Time: 0.123 ms
Execution Time: 0.045 ms
```

### Key Metrics

| Metric | Meaning |
|--------|---------|
| `cost=0.43..8.45` | Startup..Total cost (abstract units) |
| `rows=1` | Estimated rows |
| `actual time=0.023..0.025` | Actual startup..total time (ms) |
| `rows=1 loops=1` | Actual rows returned |
| `Buffers: shared hit=4` | Pages read from cache |
| `Planning Time` | Time to create plan |
| `Execution Time` | Time to execute |

## Plan Nodes

### Scan Nodes

```sql
-- Sequential Scan: Read entire table
Seq Scan on users
  Filter: (status = 'active')
-- Use when: No index, small table, or reading most rows

-- Index Scan: Use index + fetch rows
Index Scan using idx_users_email on users
  Index Cond: (email = 'user@example.com')
-- Use when: Selective query, index available

-- Index Only Scan: Data entirely in index
Index Only Scan using idx_users_email_status on users
  Index Cond: (email = 'user@example.com')
-- Use when: All SELECT columns in index

-- Bitmap Index Scan: Multiple index lookups
Bitmap Heap Scan on orders
  Recheck Cond: (status = ANY ('{pending,shipped}'::text[]))
  -> Bitmap Index Scan on idx_orders_status
      Index Cond: (status = ANY ('{pending,shipped}'::text[]))
-- Use when: Multiple index conditions, OR queries
```

### Join Nodes

```sql
-- Nested Loop: Small inner table
Nested Loop
  -> Seq Scan on users
  -> Index Scan on orders using idx_orders_user_id
      Index Cond: (user_id = users.id)
-- Complexity: O(n * m)
-- Best for: Small tables, indexed join columns

-- Hash Join: Medium-sized tables
Hash Join
  Hash Cond: (orders.user_id = users.id)
  -> Seq Scan on orders
  -> Hash
      -> Seq Scan on users
-- Complexity: O(n + m)
-- Best for: Equality joins, medium-sized tables

-- Merge Join: Pre-sorted data
Merge Join
  Merge Cond: (orders.user_id = users.id)
  -> Index Scan using idx_orders_user_id on orders
  -> Index Scan using users_pkey on users
-- Complexity: O(n + m)
-- Best for: Large sorted datasets
```

### Aggregate Nodes

```sql
-- Aggregate
Aggregate
  -> Seq Scan on orders
-- Functions: count(*), sum(), avg(), etc.

-- GroupAggregate: Pre-sorted groups
GroupAggregate
  Group Key: status
  -> Index Scan using idx_orders_status on orders
-- Use when: Data already sorted by GROUP BY column

-- HashAggregate: Unsorted groups
HashAggregate
  Group Key: status
  -> Seq Scan on orders
-- Use when: Data not sorted, hash table fits in work_mem
```

## Cost Estimation

### Cost Parameters

```sql
-- View current cost parameters
SELECT name, setting, unit
FROM pg_settings
WHERE name IN (
    'seq_page_cost',
    'random_page_cost',
    'cpu_tuple_cost',
    'cpu_index_tuple_cost',
    'cpu_operator_cost'
);
```

**Default Values:**
- `seq_page_cost = 1.0` (sequential page read)
- `random_page_cost = 4.0` (random page read, HDD)
- `cpu_tuple_cost = 0.01` (processing one row)
- `cpu_index_tuple_cost = 0.005` (processing index entry)
- `cpu_operator_cost = 0.0025` (operator execution)

**For SSDs:**
```sql
ALTER SYSTEM SET random_page_cost = 1.1;
SELECT pg_reload_conf();
```

### Cost Calculation Example

```sql
-- Table: 10,000 pages, 100,000 rows
-- Query: SELECT * FROM users;

-- Sequential Scan Cost:
-- = seq_page_cost * pages + cpu_tuple_cost * rows
-- = 1.0 * 10000 + 0.01 * 100000
-- = 10000 + 1000 = 11000

-- Index Scan Cost (100 rows):
-- = random_page_cost * index_pages + cpu_index_tuple_cost * index_rows
--   + random_page_cost * heap_pages + cpu_tuple_cost * rows
-- ≈ 4.0 * 3 + 0.005 * 100 + 4.0 * 100 + 0.01 * 100
-- = 12 + 0.5 + 400 + 1 = 413.5
```

## pg_stat_statements

### Installation

```sql
-- Add to postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000

-- Restart PostgreSQL, then:
CREATE EXTENSION pg_stat_statements;
```

### Usage

```sql
-- Top 10 slowest queries
SELECT query,
       calls,
       total_exec_time,
       mean_exec_time,
       max_exec_time,
       stddev_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Most frequently executed
SELECT query,
       calls,
       mean_exec_time
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Highest total time
SELECT query,
       calls,
       total_exec_time,
       mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

## Reading Complex Plans

### Example: Multi-Join Query

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.username, o.total, p.name
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE u.status = 'active'
  AND o.created_at > '2024-01-01'
  AND p.category = 'electronics'
ORDER BY o.total DESC
LIMIT 10;
```

**Plan:**
```
Limit  (cost=1234.56..1234.58 rows=10 width=100)
  ->  Sort  (cost=1234.56..1234.78 rows=90 width=100)
        Sort Key: o.total DESC
        ->  Hash Join  (cost=456.78..1233.45 rows=90 width=100)
              Hash Cond: (oi.product_id = p.id)
              ->  Hash Join  (cost=123.45..789.01 rows=500 width=50)
                    Hash Cond: (o.id = oi.order_id)
                    ->  Hash Join  (cost=45.67..234.56 rows=100 width=40)
                          Hash Cond: (o.user_id = u.id)
                          ->  Index Scan using idx_orders_created on orders o
                                Index Cond: (created_at > '2024-01-01')
                                Buffers: shared hit=50
                          ->  Hash
                                ->  Index Scan using idx_users_status on users u
                                      Index Cond: (status = 'active')
                                      Buffers: shared hit=10
                    ->  Hash
                          ->  Seq Scan on order_items oi
                                Buffers: shared hit=200
              ->  Hash
                    ->  Index Scan using idx_products_category on products p
                          Index Cond: (category = 'electronics')
                          Buffers: shared hit=15
Planning Time: 1.234 ms
Execution Time: 5.678 ms
```

**Reading Bottom-Up:**
1. Three index scans (users, orders, products)
2. Sequential scan (order_items)
3. Hash joins combine results
4. Sort by total
5. Limit to 10 rows

## Monitoring

### Identify Slow Queries

```sql
-- Current running queries
SELECT pid,
       now() - query_start as duration,
       state,
       query
FROM pg_stat_activity
WHERE state = 'active'
  AND query NOT LIKE '%pg_stat_activity%'
ORDER BY duration DESC;

-- Kill long-running query
SELECT pg_terminate_backend(pid);
```

### Auto_explain Extension

```sql
-- Add to postgresql.conf
shared_preload_libraries = 'auto_explain'
auto_explain.log_min_duration = 1000  # Log queries > 1s
auto_explain.log_analyze = on
auto_explain.log_buffers = on

-- Restart PostgreSQL
-- Plans logged to PostgreSQL logs
```

---

## Quick Reference

```sql
-- Basic explain
EXPLAIN query;

-- With execution
EXPLAIN ANALYZE query;

-- Full details
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) query;

-- Query statistics
SELECT * FROM pg_stat_statements;

-- Current activity
SELECT * FROM pg_stat_activity;
```

### Next Steps
- [← 06 - Indexes](../06-indexes/README.md)
- [08 - Query Optimization →](../08-query-optimization/README.md)
- [Back to Main](../README.md)
