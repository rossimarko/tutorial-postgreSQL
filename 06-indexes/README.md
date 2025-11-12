# 06 - Indexes

## Table of Contents
- [Index Fundamentals](#index-fundamentals)
- [B-tree Indexes](#b-tree-indexes)
- [Hash Indexes](#hash-indexes)
- [GiST Indexes](#gist-indexes)
- [GIN Indexes](#gin-indexes)
- [BRIN Indexes](#brin-indexes)
- [Partial Indexes](#partial-indexes)
- [Expression Indexes](#expression-indexes)
- [Multi-Column Indexes](#multi-column-indexes)
- [Index Maintenance](#index-maintenance)
- [Anti-Patterns](#anti-patterns)
- [Monitoring](#monitoring)
- [Real-World Scenarios](#real-world-scenarios)

## Index Fundamentals

### Why Indexes?

Without index:
```sql
-- Sequential scan: O(n)
SELECT * FROM users WHERE email = 'user@example.com';
-- Scans all 1,000,000 rows → ~200ms
```

With index:
```sql
CREATE INDEX idx_users_email ON users(email);
-- B-tree lookup: O(log n)
SELECT * FROM users WHERE email = 'user@example.com';
-- Scans ~4 index levels → ~2ms (100x faster!)
```

### Index Types Comparison

| Type | Use Case | Operations | Size | Speed |
|------|----------|------------|------|-------|
| **B-tree** | General purpose, ordered data | =, <, >, <=, >=, BETWEEN | Medium | Fast |
| **Hash** | Equality only | = | Small | Very Fast |
| **GiST** | Geometric, full-text | Overlaps, contains | Large | Medium |
| **GIN** | Arrays, JSONB, full-text | Contains, exists | Large | Medium-Fast |
| **BRIN** | Large sequential data | Range queries on ordered data | Tiny | Fast (large tables) |
| **SP-GiST** | Non-balanced data (phone numbers) | Prefix matching | Medium | Fast |

## B-tree Indexes

### Default and Most Common

```sql
-- Create B-tree index (default type)
CREATE INDEX idx_users_email ON users(email);

-- Explicitly specify B-tree
CREATE INDEX idx_users_created ON users USING btree(created_at);

-- Unique index
CREATE UNIQUE INDEX idx_users_username ON users(username);

-- Primary key creates unique B-tree index automatically
CREATE TABLE products (
    id SERIAL PRIMARY KEY,  -- Creates unique B-tree index
    name TEXT
);
```

### B-tree Structure

```
Root Node
├── Internal Node [1-100]
│   ├── Leaf [1-10]
│   ├── Leaf [11-25]
│   └── Leaf [26-100]
├── Internal Node [101-500]
│   ├── Leaf [101-200]
│   └── Leaf [201-500]
└── Internal Node [501-1000]
    └── Leaf [501-1000]

Each leaf points to actual table rows
```

### Supported Operations

```sql
-- Equality
SELECT * FROM users WHERE id = 100;

-- Range queries
SELECT * FROM orders WHERE created_at > '2024-01-01';
SELECT * FROM products WHERE price BETWEEN 10 AND 50;

-- Sorting (index order = sorted data)
SELECT * FROM users ORDER BY created_at DESC LIMIT 10;

-- Min/Max (index provides instant lookup)
SELECT max(created_at) FROM orders;

-- Pattern matching (left-anchored)
SELECT * FROM users WHERE email LIKE 'admin%';  -- Uses index
SELECT * FROM users WHERE email LIKE '%admin'; -- No index
```

### Performance Example

```sql
-- Create test table
CREATE TABLE large_table (
    id SERIAL PRIMARY KEY,
    value INT,
    data TEXT
);

INSERT INTO large_table (value, data)
SELECT (random() * 1000000)::int, md5(random()::text)
FROM generate_series(1, 1000000);

-- Without index
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table WHERE value = 50000;

-- Result:
-- Seq Scan on large_table  (cost=0.00..18334.00 rows=1 width=41) (actual time=45.123..156.789 rows=1 loops=1)
-- Planning Time: 0.123 ms
-- Execution Time: 156.825 ms

-- With index
CREATE INDEX idx_large_value ON large_table(value);

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table WHERE value = 50000;

-- Result:
-- Index Scan using idx_large_value on large_table  (cost=0.42..8.44 rows=1 width=41) (actual time=0.034..0.035 rows=1 loops=1)
-- Planning Time: 0.198 ms
-- Execution Time: 0.056 ms (2800x faster!)
```

## Hash Indexes

### When to Use

Hash indexes are optimized for **equality comparisons only** (PostgreSQL 10+ improved):

```sql
-- Create hash index
CREATE INDEX idx_users_api_key ON users USING hash(api_key);

-- Efficient for equality
EXPLAIN SELECT * FROM users WHERE api_key = 'abc123';
-- Index Scan using idx_users_api_key

-- NOT efficient for ranges
EXPLAIN SELECT * FROM users WHERE api_key > 'abc';
-- Seq Scan (hash doesn't support range queries)
```

### Hash vs B-tree Performance

```sql
-- Test table
CREATE TABLE hash_test (
    id SERIAL PRIMARY KEY,
    lookup_key TEXT
);

INSERT INTO hash_test (lookup_key)
SELECT md5(random()::text) FROM generate_series(1, 1000000);

-- B-tree index
CREATE INDEX idx_btree ON hash_test USING btree(lookup_key);

-- Hash index
CREATE INDEX idx_hash ON hash_test USING hash(lookup_key);

-- Compare sizes
SELECT pg_size_pretty(pg_relation_size('idx_btree')) as btree_size,
       pg_size_pretty(pg_relation_size('idx_hash')) as hash_size;
-- btree_size: 42 MB
-- hash_size: 37 MB (slightly smaller)

-- Compare equality lookup performance
-- Hash: ~0.02ms
-- B-tree: ~0.03ms
-- Difference negligible in most cases, B-tree more versatile
```

## GiST Indexes

### Generalized Search Tree

For geometric data, full-text search, range types:

```sql
-- Install PostGIS for geometric data
CREATE EXTENSION postgis;

-- Create table with geometric data
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    geom GEOMETRY(Point, 4326)
);

-- Insert sample data
INSERT INTO locations (name, geom)
VALUES
    ('Point A', ST_SetSRID(ST_MakePoint(-122.4194, 37.7749), 4326)),
    ('Point B', ST_SetSRID(ST_MakePoint(-118.2437, 34.0522), 4326));

-- Create GiST index
CREATE INDEX idx_locations_geom ON locations USING gist(geom);

-- Find points within distance
EXPLAIN ANALYZE
SELECT name, ST_Distance(geom, ST_SetSRID(ST_MakePoint(-122.4, 37.7), 4326)) as dist
FROM locations
WHERE ST_DWithin(geom, ST_SetSRID(ST_MakePoint(-122.4, 37.7), 4326), 0.1);
-- Uses GiST index for spatial search
```

### Full-Text Search with GiST

```sql
-- Create table with text
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    search_vector tsvector
);

-- Generate search vector
UPDATE documents
SET search_vector = to_tsvector('english', title || ' ' || content);

-- Create GiST index for full-text search
CREATE INDEX idx_documents_search ON documents USING gist(search_vector);

-- Full-text query
EXPLAIN ANALYZE
SELECT title
FROM documents
WHERE search_vector @@ to_tsquery('english', 'postgresql & performance');
-- Uses GiST index
```

## GIN Indexes

### Generalized Inverted Index

Optimal for **multi-value columns**: arrays, JSONB, full-text:

```sql
-- Array column
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    tags TEXT[]
);

INSERT INTO products (name, tags)
VALUES
    ('Product A', ARRAY['electronics', 'sale', 'featured']),
    ('Product B', ARRAY['clothing', 'new', 'sale']);

-- Create GIN index for array containment
CREATE INDEX idx_products_tags ON products USING gin(tags);

-- Query: find products with specific tag
EXPLAIN ANALYZE
SELECT * FROM products WHERE tags @> ARRAY['sale'];
-- Bitmap Heap Scan
--   Recheck Cond: (tags @> '{sale}'::text[])
--   ->  Bitmap Index Scan on idx_products_tags
-- Very fast!

-- JSONB with GIN
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    data JSONB
);

CREATE INDEX idx_events_data ON events USING gin(data);

-- Query JSON
SELECT * FROM events WHERE data @> '{"type": "purchase"}';
-- Uses GIN index
```

### GIN vs GiST for Full-Text

```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    content TEXT,
    search_vector tsvector
);

-- GIN index (larger, faster queries)
CREATE INDEX idx_gin ON articles USING gin(search_vector);

-- GiST index (smaller, slower queries, faster updates)
CREATE INDEX idx_gist ON articles USING gist(search_vector);
```

**Comparison:**
| Aspect | GIN | GiST |
|--------|-----|------|
| Size | Larger (3x) | Smaller |
| Query Speed | Faster | Slower |
| Update Speed | Slower | Faster |
| Use Case | Read-heavy | Write-heavy |

## BRIN Indexes

### Block Range Index

For **very large tables** with natural ordering:

```sql
-- Time-series data (naturally ordered by timestamp)
CREATE TABLE sensor_data (
    id SERIAL,
    sensor_id INT,
    value NUMERIC,
    recorded_at TIMESTAMP
);

-- Insert 100M rows in chronological order
INSERT INTO sensor_data (sensor_id, value, recorded_at)
SELECT
    (random() * 100)::int,
    random() * 100,
    '2020-01-01'::timestamp + (g || ' seconds')::interval
FROM generate_series(1, 100000000) g;

-- B-tree index: ~2GB
CREATE INDEX idx_btree ON sensor_data(recorded_at);

-- BRIN index: ~200KB (10,000x smaller!)
CREATE INDEX idx_brin ON sensor_data USING brin(recorded_at);

-- Query performance
EXPLAIN ANALYZE
SELECT * FROM sensor_data
WHERE recorded_at BETWEEN '2024-01-01' AND '2024-01-31';

-- BRIN scans only relevant blocks (10-50x fewer blocks than seq scan)
-- Good for large tables with natural order
```

**When to Use BRIN:**
- Table > 10GB
- Column has natural order (timestamp, auto-increment ID)
- Range queries
- Minimal updates/deletes

## Partial Indexes

### Index Only What You Need

```sql
-- Full index (large, includes inactive users)
CREATE INDEX idx_users_status ON users(status);

-- Partial index (small, only active users)
CREATE INDEX idx_users_active ON users(status)
WHERE status = 'active';

-- Size comparison
-- Full index: 100MB
-- Partial index: 20MB (80% savings!)

-- Query must match WHERE condition
EXPLAIN SELECT * FROM users WHERE status = 'active';
-- Uses idx_users_active

EXPLAIN SELECT * FROM users WHERE status = 'inactive';
-- Does NOT use idx_users_active (doesn't match WHERE clause)
```

### Real-World Examples

```sql
-- Index only recent orders
CREATE INDEX idx_recent_orders ON orders(created_at)
WHERE created_at > '2024-01-01';

-- Index only NULL values (sparse column)
CREATE INDEX idx_users_deleted ON users(deleted_at)
WHERE deleted_at IS NOT NULL;

-- Index only expensive products
CREATE INDEX idx_expensive_products ON products(price)
WHERE price > 1000;

-- Index only published articles
CREATE INDEX idx_published_articles ON articles(published_at)
WHERE status = 'published';
```

## Expression Indexes

### Index on Computed Values

```sql
-- Problem: Case-insensitive search
SELECT * FROM users WHERE lower(email) = 'admin@example.com';
-- Sequential scan (no index on lower(email))

-- Solution: Expression index
CREATE INDEX idx_users_lower_email ON users(lower(email));

-- Now uses index
EXPLAIN SELECT * FROM users WHERE lower(email) = 'admin@example.com';
-- Index Scan using idx_users_lower_email

-- Other examples
-- Extract from JSON
CREATE INDEX idx_events_type ON events((data->>'type'));

-- Date truncation
CREATE INDEX idx_orders_month ON orders(date_trunc('month', created_at));

-- Mathematical expression
CREATE INDEX idx_circle_area ON shapes((pi() * radius * radius));
```

## Multi-Column Indexes

### Composite Indexes

```sql
-- Single-column indexes
CREATE INDEX idx_users_last_name ON users(last_name);
CREATE INDEX idx_users_first_name ON users(first_name);

-- Multi-column index
CREATE INDEX idx_users_name ON users(last_name, first_name);
```

### Column Order Matters!

```sql
-- Index: (last_name, first_name)
CREATE INDEX idx_users_name ON users(last_name, first_name);

-- ✅ Uses index (left-most column)
SELECT * FROM users WHERE last_name = 'Smith';

-- ✅ Uses index (both columns)
SELECT * FROM users WHERE last_name = 'Smith' AND first_name = 'John';

-- ❌ Doesn't use index efficiently (right column only)
SELECT * FROM users WHERE first_name = 'John';
-- Consider separate index or reversed composite index
```

### Optimal Column Ordering

**Rule:** Most selective (unique) column first

```sql
-- Analyze selectivity
SELECT
    count(DISTINCT last_name) as distinct_last_names,
    count(DISTINCT first_name) as distinct_first_names,
    count(DISTINCT city) as distinct_cities,
    count(*) as total_rows
FROM users;

-- Results:
-- distinct_last_names: 50,000
-- distinct_first_names: 10,000
-- distinct_cities: 500
-- total_rows: 100,000

-- Best order: last_name (most selective), first_name, city
CREATE INDEX idx_users_composite ON users(last_name, first_name, city);
```

## Index Maintenance

### REINDEX

```sql
-- Rebuild index (removes bloat, updates statistics)
REINDEX INDEX idx_users_email;

-- Rebuild all indexes on table
REINDEX TABLE users;

-- Rebuild all indexes in database
REINDEX DATABASE mydb;

-- Concurrent reindex (PostgreSQL 12+, no locking)
REINDEX INDEX CONCURRENTLY idx_users_email;
```

### Index Bloat

```sql
-- Detect bloated indexes
SELECT schemaname,
       tablename,
       indexname,
       pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
       idx_scan,
       idx_tup_read,
       idx_tup_fetch
FROM pg_stat_user_indexes
JOIN pg_index USING (indexrelid)
ORDER BY pg_relation_size(indexrelid) DESC;

-- If idx_scan is low and size is large → consider dropping
-- If heavily used → REINDEX to remove bloat
```

## Anti-Patterns

### ❌ Anti-Pattern 1: Over-Indexing

```sql
-- BAD: Index every column
CREATE INDEX idx_users_id ON users(id);           -- Redundant (PK exists)
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_first ON users(first_name);
CREATE INDEX idx_users_last ON users(last_name);
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_users_status ON users(status);

-- Impact:
-- - Slow INSERT/UPDATE/DELETE
-- - Wasted disk space
-- - Index maintenance overhead
```

**Solution:** Index only frequently queried columns

### ❌ Anti-Pattern 2: Wrong Column Order

```sql
-- BAD: Low selectivity first
CREATE INDEX idx_users_status_email ON users(status, email);
-- status has 3 values (active, inactive, deleted)
-- email has 1M unique values

-- Query: WHERE email = 'user@example.com'
-- Doesn't use index efficiently
```

**Solution:**
```sql
-- GOOD: High selectivity first
CREATE INDEX idx_users_email_status ON users(email, status);
```

### ❌ Anti-Pattern 3: Not Using Partial Indexes

```sql
-- BAD: Full index for sparse data
CREATE INDEX idx_users_deleted_at ON users(deleted_at);
-- 1% of users deleted → 99% waste

-- GOOD: Partial index
CREATE INDEX idx_users_deleted ON users(deleted_at)
WHERE deleted_at IS NOT NULL;
```

## Monitoring

### Index Usage

```sql
-- Unused indexes
SELECT schemaname,
       tablename,
       indexname,
       idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Most used indexes
SELECT schemaname,
       tablename,
       indexname,
       idx_scan,
       idx_tup_read,
       idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC
LIMIT 20;
```

### Missing Indexes

```sql
-- Tables with sequential scans (potential index candidates)
SELECT schemaname,
       tablename,
       seq_scan,
       seq_tup_read,
       idx_scan,
       seq_tup_read / NULLIF(seq_scan, 0) as avg_seq_read,
       n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan > 100
  AND n_live_tup > 10000
  AND seq_tup_read / NULLIF(seq_scan, 0) > 10000
ORDER BY seq_tup_read DESC;
```

## Real-World Scenarios

### Scenario 1: Optimizing Email Lookup

```sql
-- Problem: Slow login query
EXPLAIN ANALYZE
SELECT * FROM users WHERE lower(email) = lower('User@Example.com');
-- Seq Scan, 350ms

-- Solution: Expression index
CREATE INDEX idx_users_lower_email ON users(lower(email));

-- Result:
EXPLAIN ANALYZE
SELECT * FROM users WHERE lower(email) = lower('User@Example.com');
-- Index Scan, 0.8ms (437x faster!)
```

---

## Quick Reference

```sql
-- Create indexes
CREATE INDEX idx_name ON table(column);
CREATE INDEX idx_name ON table USING gin(jsonb_column);
CREATE INDEX idx_name ON table(column) WHERE condition;

-- Monitor
SELECT * FROM pg_stat_user_indexes;
SELECT * FROM pg_indexes WHERE tablename = 'table';

-- Maintenance
REINDEX INDEX CONCURRENTLY idx_name;
DROP INDEX CONCURRENTLY idx_name;
```

### Next Steps
- [← 05 - Statistics](../05-statistics/README.md)
- [07 - Query Plans →](../07-query-plans/README.md)
- [Back to Main](../README.md)
