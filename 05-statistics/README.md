# 05 - Statistics

## Table of Contents
- [PostgreSQL Statistics System](#postgresql-statistics-system)
- [ANALYZE Command](#analyze-command)
- [pg_statistic Internals](#pg_statistic-internals)
- [Histograms and MCVs](#histograms-and-mcvs)
- [Extended Statistics](#extended-statistics)
- [Auto-Analyze](#auto-analyze)
- [Anti-Patterns](#anti-patterns)
- [Monitoring](#monitoring)
- [Real-World Scenarios](#real-world-scenarios)

## PostgreSQL Statistics System

### Why Statistics Matter

The query planner uses statistics to estimate:
- Number of rows matching WHERE conditions
- Cost of different join methods
- Whether to use indexes or sequential scans

```
Poor Statistics → Bad Query Plans → Slow Queries
Good Statistics → Optimal Query Plans → Fast Queries
```

### Statistics Architecture

```
┌─────────────────────────────────────────────┐
│          Table: users (1M rows)             │
└────────────────┬────────────────────────────┘
                 │
          ANALYZE command
                 │
                 ▼
┌─────────────────────────────────────────────┐
│        pg_statistic (system catalog)        │
├─────────────────────────────────────────────┤
│ - n_distinct (unique values)                │
│ - null_frac (NULL percentage)               │
│ - avg_width (average column width)          │
│ - Most Common Values (MCV)                  │
│ - Histogram bounds                          │
│ - Correlation                               │
└────────────────┬────────────────────────────┘
                 │
          Query Planning
                 │
                 ▼
┌─────────────────────────────────────────────┐
│     EXPLAIN: cost=1.50..25.70 rows=100      │
└─────────────────────────────────────────────┘
```

## ANALYZE Command

### Basic Usage

```sql
-- Analyze single table
ANALYZE users;

-- Analyze specific column
ANALYZE users (email);

-- Analyze all tables in database
ANALYZE;

-- Analyze with verbose output
ANALYZE VERBOSE users;
```

**Output:**
```
INFO:  analyzing "public.users"
INFO:  "users": scanned 30000 of 30000 pages, containing 1000000 live rows
                and 0 dead rows; 30000 rows in sample, 1000000 estimated total rows
```

### When to Run ANALYZE

1. **After bulk data loads**
2. **After significant data changes** (>10% of rows)
3. **Before critical queries** (manual intervention)
4. **After index creation**
5. **After upgrades** (version changes)

```sql
-- Bulk load scenario
BEGIN;
COPY users FROM '/data/users.csv' CSV;  -- Load 1M rows
ANALYZE users;  -- Update statistics
COMMIT;
```

### Statistics Target

Controls sample size and accuracy:

```sql
-- Default statistics target (100 = 100 histogram bins)
SHOW default_statistics_target;
-- Result: 100

-- Increase for specific column (more accurate, slower ANALYZE)
ALTER TABLE users ALTER COLUMN email SET STATISTICS 1000;

-- Decrease for large, unimportant columns
ALTER TABLE logs ALTER COLUMN message SET STATISTICS 10;

-- Set default for all new columns
ALTER DATABASE mydb SET default_statistics_target = 200;
```

**Impact:**
- Higher target = more accurate estimates, slower ANALYZE
- Lower target = faster ANALYZE, less accurate estimates

**Benchmark:**
```sql
-- Default (100)
ANALYZE users;
-- Time: 500ms
-- Histogram buckets: 100

-- High target (1000)
ALTER TABLE users ALTER COLUMN id SET STATISTICS 1000;
ANALYZE users;
-- Time: 3200ms
-- Histogram buckets: 1000
-- Better estimates for range queries
```

## pg_statistic Internals

### Viewing Statistics

```sql
-- Human-readable view
SELECT schemaname,
       tablename,
       attname,
       null_frac,
       avg_width,
       n_distinct,
       correlation
FROM pg_stats
WHERE tablename = 'users'
ORDER BY attname;
```

**Key Metrics:**

| Metric | Description | Example |
|--------|-------------|---------|
| `null_frac` | Fraction of NULL values | 0.05 (5% NULLs) |
| `avg_width` | Average column width in bytes | 25 (email column) |
| `n_distinct` | Estimated unique values | 950000 (distinct emails) |
| `most_common_vals` | Top frequent values | {admin@example.com, ...} |
| `most_common_freqs` | Frequencies of MCVs | {0.001, 0.0008, ...} |
| `histogram_bounds` | Distribution histogram | {a, h, m, s, z} |
| `correlation` | Physical vs logical ordering | 0.95 (well clustered) |

### Example: Analyzing Statistics

```sql
-- Create test table
CREATE TABLE test_stats (
    id SERIAL PRIMARY KEY,
    category TEXT,
    value INT,
    created_at TIMESTAMP DEFAULT now()
);

-- Insert data with patterns
INSERT INTO test_stats (category, value)
SELECT
    CASE (random() * 3)::int
        WHEN 0 THEN 'A'
        WHEN 1 THEN 'B'
        ELSE 'C'
    END,
    (random() * 1000)::int
FROM generate_series(1, 100000);

-- Analyze
ANALYZE test_stats;

-- View statistics
SELECT attname,
       null_frac,
       n_distinct,
       most_common_vals,
       most_common_freqs,
       histogram_bounds
FROM pg_stats
WHERE tablename = 'test_stats'
  AND attname IN ('category', 'value');
```

**Result:**
```
attname  | null_frac | n_distinct | most_common_vals | most_common_freqs | histogram_bounds
---------+-----------+------------+------------------+-------------------+------------------
category | 0         | 3          | {A,B,C}          | {0.333,0.333,0.333}| NULL
value    | 0         | -0.99      | NULL             | NULL              | {0,10,20,...,990}
```

### Interpreting n_distinct

- **Positive:** Actual count of distinct values (e.g., 3 for categories)
- **Negative:** Ratio of distinct to total rows (e.g., -0.99 = 99% unique)

```sql
-- Check n_distinct interpretation
SELECT attname,
       n_distinct,
       CASE
           WHEN n_distinct > 0 THEN n_distinct::text || ' distinct values'
           WHEN n_distinct < 0 THEN (abs(n_distinct) * 100)::text || '% unique'
       END as interpretation
FROM pg_stats
WHERE tablename = 'users';
```

## Histograms and MCVs

### Most Common Values (MCV)

Top N most frequent values stored separately for better estimates.

```sql
-- View MCVs
SELECT attname,
       most_common_vals as mcv,
       most_common_freqs as freq
FROM pg_stats
WHERE tablename = 'orders'
  AND attname = 'status';
```

**Example:**
```
mcv                          | freq
-----------------------------+----------------------
{completed,pending,shipped}  | {0.65,0.25,0.08}
```

**Query Planner Usage:**
```sql
EXPLAIN SELECT * FROM orders WHERE status = 'completed';

-- Planner knows: 65% of rows match → 650,000 rows from 1M total
-- Seq Scan cost calculation uses this estimate
```

### Histograms

For non-MCV values, PostgreSQL uses histogram buckets:

```sql
-- View histogram
SELECT attname,
       histogram_bounds
FROM pg_stats
WHERE tablename = 'sales'
  AND attname = 'amount';
```

**Example:**
```
histogram_bounds
{0.00, 25.50, 50.00, 75.25, 100.00, 250.00, 500.00, 1000.00, 5000.00, 10000.00}
```

**Interpretation:**
- 10% of values fall in each bucket
- Bucket 1: 0.00 - 25.50 (10% of rows)
- Bucket 2: 25.50 - 50.00 (10% of rows)
- ...

**Range Query Estimation:**
```sql
EXPLAIN SELECT * FROM sales WHERE amount BETWEEN 100 AND 500;

-- Planner calculates:
-- Buckets 5-7 partially covered
-- Estimates ~25% of rows
```

### Correlation

Correlation measures alignment between physical and logical order:

```sql
SELECT attname,
       correlation
FROM pg_stats
WHERE tablename = 'logs'
  AND attname IN ('id', 'created_at', 'random_value');
```

**Results:**
```
attname      | correlation
-------------+-------------
id           | 0.99        -- Perfectly ordered
created_at   | 0.85        -- Well ordered
random_value | 0.02        -- Random
```

**Impact on Query Plans:**
- High correlation (>0.9): Index scan is efficient (sequential disk reads)
- Low correlation (<0.1): Index scan is inefficient (random disk reads)

```sql
-- High correlation: index scan preferred
EXPLAIN SELECT * FROM logs WHERE id BETWEEN 1000 AND 2000;
-- Index Scan (sequential disk I/O)

-- Low correlation: seq scan may be preferred
EXPLAIN SELECT * FROM logs WHERE random_value BETWEEN 1000 AND 2000;
-- Seq Scan (avoid random disk I/O)
```

## Extended Statistics

### Multi-Column Statistics (PostgreSQL 10+)

Handle correlated columns that standard statistics miss.

**Problem: Correlated Columns**
```sql
CREATE TABLE addresses (
    id SERIAL PRIMARY KEY,
    city TEXT,
    state TEXT,
    country TEXT
);

-- Insert data: San Francisco is always in CA, USA
INSERT INTO addresses (city, state, country)
SELECT 'San Francisco', 'CA', 'USA'
FROM generate_series(1, 50000)
UNION ALL
SELECT 'New York', 'NY', 'USA'
FROM generate_series(1, 50000);

ANALYZE addresses;

-- Query with correlated filters
EXPLAIN ANALYZE
SELECT * FROM addresses
WHERE city = 'San Francisco'
  AND state = 'CA'
  AND country = 'USA';

-- Planner assumes independence: 50000 * 0.5 * 0.5 * 1.0 = 12500 rows
-- Reality: 50000 rows (correlation not detected)
```

**Solution: Extended Statistics**
```sql
-- Create multi-column statistics
CREATE STATISTICS addr_stats (dependencies)
ON city, state, country
FROM addresses;

-- Re-analyze to populate extended stats
ANALYZE addresses;

-- Now query estimates are accurate
EXPLAIN ANALYZE
SELECT * FROM addresses
WHERE city = 'San Francisco'
  AND state = 'CA'
  AND country = 'USA';

-- Planner now estimates: ~50000 rows (correct!)
```

### Types of Extended Statistics

**1. Dependencies (n_distinct):**
```sql
CREATE STATISTICS product_stats (dependencies)
ON category, subcategory
FROM products;
```

**2. N-Distinct:**
```sql
CREATE STATISTICS order_stats (ndistinct)
ON user_id, product_id
FROM orders;
```

**3. MCV (Most Common Values for column combinations):**
```sql
CREATE STATISTICS user_stats (mcv)
ON city, state
FROM users;
```

**4. All types:**
```sql
CREATE STATISTICS comprehensive_stats (dependencies, ndistinct, mcv)
ON col1, col2, col3
FROM my_table;
```

### Viewing Extended Statistics

```sql
-- List extended statistics
SELECT stxname,
       stxnamespace::regnamespace,
       stxrelid::regclass,
       stxkeys,
       stxkind
FROM pg_statistic_ext;

-- View extended stats details
SELECT * FROM pg_stats_ext
WHERE tablename = 'addresses';
```

### Expression Statistics (PostgreSQL 11+)

Statistics on expressions, not just columns:

```sql
-- Create index on expression
CREATE INDEX idx_users_lower_email ON users(lower(email));

-- Create statistics on expression
CREATE STATISTICS users_lower_email_stats
ON lower(email)
FROM users;

ANALYZE users;

-- Better estimates for expression queries
EXPLAIN SELECT * FROM users WHERE lower(email) = 'admin@example.com';
```

## Auto-Analyze

### How Auto-Analyze Works

```ini
# postgresql.conf
autovacuum = on  # Must be enabled

# Auto-analyze triggers
autovacuum_analyze_threshold = 50      # Min rows changed
autovacuum_analyze_scale_factor = 0.1  # 10% of table size

# Trigger condition:
# If (changes > threshold + scale_factor * table_size):
#     Run ANALYZE
```

**Example:**
- Table: 1,000,000 rows
- Threshold: 50
- Scale factor: 0.1
- Trigger: 50 + (0.1 * 1,000,000) = 100,050 changes

```sql
-- Monitor auto-analyze
SELECT schemaname,
       tablename,
       last_analyze,
       last_autoanalyze,
       n_mod_since_analyze,
       n_ins_since_vacuum
FROM pg_stat_user_tables
ORDER BY n_mod_since_analyze DESC;
```

### Tuning Auto-Analyze

**For volatile tables (frequent changes):**
```sql
ALTER TABLE high_churn_table
SET (autovacuum_analyze_scale_factor = 0.02);  -- Analyze at 2% changes

ALTER TABLE high_churn_table
SET (autovacuum_analyze_threshold = 100);
```

**For large, stable tables:**
```sql
ALTER TABLE large_stable_table
SET (autovacuum_analyze_scale_factor = 0.2);  -- Analyze at 20% changes
```

**Disable auto-analyze (manual control):**
```sql
ALTER TABLE manual_stats_table
SET (autovacuum_enabled = off);

-- Manually analyze as needed
ANALYZE manual_stats_table;
```

## Anti-Patterns

### ❌ Anti-Pattern 1: Never Running ANALYZE

```sql
-- BAD: Load data and immediately query
COPY users FROM '/data/users.csv';
SELECT * FROM users WHERE email = 'user@example.com';
-- Query uses default estimates, may choose wrong plan
```

**Solution:**
```sql
-- GOOD: Analyze after data load
COPY users FROM '/data/users.csv';
ANALYZE users;
SELECT * FROM users WHERE email = 'user@example.com';
```

### ❌ Anti-Pattern 2: Disabling Auto-Analyze Globally

```ini
# BAD
autovacuum = off
```

**Impact:**
- Stale statistics
- Suboptimal query plans
- Performance degradation

**Solution:**
```ini
# GOOD: Keep enabled, tune per table if needed
autovacuum = on
```

### ❌ Anti-Pattern 3: Ignoring Correlated Columns

```sql
-- BAD: Query with correlated filters, no extended stats
SELECT * FROM events
WHERE event_type = 'purchase'
  AND user_tier = 'premium';
-- Planner assumes independence, wrong row estimate
```

**Solution:**
```sql
-- GOOD: Create extended statistics
CREATE STATISTICS event_stats (dependencies, mcv)
ON event_type, user_tier
FROM events;

ANALYZE events;
```

### ❌ Anti-Pattern 4: Default Statistics Target for Critical Columns

```sql
-- BAD: Critical range query column with default stats
CREATE INDEX idx_orders_created ON orders(created_at);
-- default_statistics_target = 100

SELECT * FROM orders
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';
-- May have inaccurate estimates
```

**Solution:**
```sql
-- GOOD: Increase statistics target for critical columns
ALTER TABLE orders
ALTER COLUMN created_at SET STATISTICS 1000;

ANALYZE orders;
```

## Monitoring

### Statistics Freshness

```sql
-- Tables needing ANALYZE
SELECT schemaname,
       tablename,
       n_live_tup as rows,
       n_mod_since_analyze as modifications,
       last_analyze,
       last_autoanalyze,
       CASE
           WHEN last_analyze IS NULL AND last_autoanalyze IS NULL THEN 'NEVER'
           WHEN n_mod_since_analyze > n_live_tup * 0.1 THEN 'STALE'
           ELSE 'OK'
       END as status
FROM pg_stat_user_tables
WHERE n_live_tup > 1000  -- Only tables with >1000 rows
ORDER BY n_mod_since_analyze DESC;
```

### Statistics Age

```sql
-- How long since last ANALYZE
SELECT schemaname,
       tablename,
       now() - last_analyze as analyze_age,
       now() - last_autoanalyze as autoanalyze_age,
       n_mod_since_analyze
FROM pg_stat_user_tables
WHERE last_analyze IS NOT NULL
   OR last_autoanalyze IS NOT NULL
ORDER BY
    CASE
        WHEN last_analyze IS NULL THEN last_autoanalyze
        WHEN last_autoanalyze IS NULL THEN last_analyze
        ELSE GREATEST(last_analyze, last_autoanalyze)
    END ASC;
```

### Query Plan Statistics

```sql
-- Enable pg_stat_statements
CREATE EXTENSION pg_stat_statements;

-- Queries with high planning time (may need better stats)
SELECT query,
       calls,
       mean_plan_time,
       mean_exec_time,
       mean_plan_time / NULLIF(mean_exec_time, 0) as plan_to_exec_ratio
FROM pg_stat_statements
WHERE mean_plan_time > 1  -- Planning takes >1ms
ORDER BY mean_plan_time DESC
LIMIT 20;
```

## Real-World Scenarios

### Scenario 1: Bulk Data Load Performance

**Problem:**
```sql
-- Load 10M rows
COPY orders FROM '/data/orders.csv';

-- Immediate query is slow
SELECT * FROM orders WHERE status = 'pending';
-- Seq Scan (cost=0.00..169248.00 rows=1000000 width=100)
-- Wrong estimate! Only 50,000 pending orders
```

**Solution:**
```sql
-- Proper bulk load procedure
BEGIN;

-- 1. Load data
COPY orders FROM '/data/orders.csv';

-- 2. Create indexes
CREATE INDEX idx_orders_status ON orders(status);

-- 3. Analyze to update statistics
ANALYZE orders;

COMMIT;

-- 4. Verify
EXPLAIN SELECT * FROM orders WHERE status = 'pending';
-- Index Scan (cost=0.43..2548.67 rows=50123 width=100)
-- Correct estimate!
```

### Scenario 2: Geographic Query Optimization

**Setup:**
```sql
CREATE TABLE stores (
    id SERIAL PRIMARY KEY,
    name TEXT,
    city TEXT,
    state TEXT,
    country TEXT
);

-- Data: Most stores in specific cities per state
INSERT INTO stores (name, city, state, country)
SELECT
    'Store ' || g,
    CASE (random() * 10)::int
        WHEN 0..4 THEN 'San Francisco'
        WHEN 5..7 THEN 'Los Angeles'
        ELSE 'San Diego'
    END,
    'CA',
    'USA'
FROM generate_series(1, 100000) g;
```

**Problem:**
```sql
-- Correlated geographic data
EXPLAIN ANALYZE
SELECT * FROM stores
WHERE city = 'San Francisco'
  AND state = 'CA'
  AND country = 'USA';

-- Estimate: 12500 rows (assumes independence)
-- Actual: 50000 rows
-- Wrong plan chosen!
```

**Solution:**
```sql
-- Create extended statistics
CREATE STATISTICS geo_stats (dependencies, mcv)
ON city, state, country
FROM stores;

ANALYZE stores;

-- Re-run query
EXPLAIN ANALYZE
SELECT * FROM stores
WHERE city = 'San Francisco'
  AND state = 'CA'
  AND country = 'USA';

-- Estimate: 49876 rows (accurate!)
-- Correct plan chosen
```

### Scenario 3: Seasonal Data Patterns

**Problem:**
```sql
CREATE TABLE sales (
    id SERIAL PRIMARY KEY,
    amount NUMERIC(10,2),
    created_at DATE
);

-- Load 1 year of data (holiday spike in December)
-- Most sales: $10-100
-- December sales: $500-5000

-- Query after December
SELECT * FROM sales
WHERE created_at >= '2024-12-01'
  AND created_at < '2025-01-01'
  AND amount > 1000;

-- Uses amount histogram from whole year
-- Underestimates December results
```

**Solution 1: Partial Statistics (PostgreSQL 11+)**
```sql
-- Create statistics on subset
CREATE STATISTICS sales_december_stats
ON amount
FROM sales
WHERE created_at >= '2024-12-01' AND created_at < '2025-01-01';
-- Note: This feature is limited; better to use partitioning
```

**Solution 2: Table Partitioning**
```sql
-- Partition by month
CREATE TABLE sales (
    id SERIAL,
    amount NUMERIC(10,2),
    created_at DATE
) PARTITION BY RANGE (created_at);

CREATE TABLE sales_2024_12 PARTITION OF sales
FOR VALUES FROM ('2024-12-01') TO ('2025-01-01');

-- Each partition has its own statistics
ANALYZE sales_2024_12;

-- Queries now use partition-specific statistics
```

---

## Quick Reference

### Essential Commands
```sql
-- Analyze table
ANALYZE tablename;

-- Analyze specific column
ANALYZE tablename (columnname);

-- View statistics
SELECT * FROM pg_stats WHERE tablename = 'tablename';

-- Extended statistics
CREATE STATISTICS stat_name (dependencies, ndistinct, mcv)
ON col1, col2 FROM tablename;

-- Set statistics target
ALTER TABLE tablename ALTER COLUMN colname SET STATISTICS 1000;

-- Check statistics freshness
SELECT * FROM pg_stat_user_tables WHERE tablename = 'tablename';
```

### Next Steps
- [← 04 - Backup and Restore](../04-backup-restore/README.md)
- [06 - Indexes →](../06-indexes/README.md)
- [Back to Main](../README.md)
