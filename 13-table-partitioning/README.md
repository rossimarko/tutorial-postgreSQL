# 13 - Table Partitioning

## Partitioning Fundamentals

### Why Partition?

- **Performance:** Query only relevant partitions
- **Maintenance:** VACUUM/ANALYZE smaller chunks
- **Data Management:** Easy archival/deletion
- **Parallel:** Better parallel query execution

## Partition Types

### Range Partitioning

```sql
-- Create partitioned table
CREATE TABLE sales (
    id BIGSERIAL,
    sale_date DATE NOT NULL,
    amount NUMERIC(10,2)
) PARTITION BY RANGE (sale_date);

-- Create partitions
CREATE TABLE sales_2023_q1 PARTITION OF sales
    FOR VALUES FROM ('2023-01-01') TO ('2023-04-01');

CREATE TABLE sales_2023_q2 PARTITION OF sales
    FOR VALUES FROM ('2023-04-01') TO ('2023-07-01');

CREATE TABLE sales_2024 PARTITION OF sales
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Insert routes to correct partition
INSERT INTO sales (sale_date, amount)
VALUES ('2023-02-15', 100.00);  -- Goes to sales_2023_q1
```

### List Partitioning

```sql
CREATE TABLE customers (
    id BIGSERIAL,
    name TEXT,
    country TEXT NOT NULL
) PARTITION BY LIST (country);

CREATE TABLE customers_us PARTITION OF customers
    FOR VALUES IN ('USA', 'US');

CREATE TABLE customers_eu PARTITION OF customers
    FOR VALUES IN ('UK', 'FR', 'DE', 'IT', 'ES');

CREATE TABLE customers_asia PARTITION OF customers
    FOR VALUES IN ('JP', 'CN', 'IN', 'KR');
```

### Hash Partitioning

```sql
CREATE TABLE orders (
    id BIGSERIAL,
    user_id BIGINT NOT NULL,
    total NUMERIC(10,2)
) PARTITION BY HASH (user_id);

-- Create 4 hash partitions
CREATE TABLE orders_p0 PARTITION OF orders
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE orders_p1 PARTITION OF orders
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE orders_p2 PARTITION OF orders
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE orders_p3 PARTITION OF orders
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

## Partition Pruning

### Automatic Pruning

```sql
-- Query automatically scans only relevant partition
EXPLAIN SELECT * FROM sales
WHERE sale_date BETWEEN '2023-01-01' AND '2023-03-31';

-- Plan shows only sales_2023_q1 scanned:
Seq Scan on sales_2023_q1 sales
  Filter: (sale_date >= '2023-01-01' AND sale_date <= '2023-03-31')
```

### Enable Partition Pruning

```sql
-- Ensure enabled (default: on)
SET enable_partition_pruning = on;
```

## Partition Management

### Attach/Detach Partitions

```sql
-- Create new partition
CREATE TABLE sales_2025_q1 (LIKE sales INCLUDING ALL);

-- Attach to partitioned table
ALTER TABLE sales ATTACH PARTITION sales_2025_q1
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');

-- Detach partition (for archival or deletion)
ALTER TABLE sales DETACH PARTITION sales_2023_q1;

-- Now can drop or archive
DROP TABLE sales_2023_q1;
-- or
pg_dump -t sales_2023_q1 mydb > sales_2023_q1_archive.sql
```

### Default Partition

```sql
-- Catch-all for values not in other partitions
CREATE TABLE sales_default PARTITION OF sales DEFAULT;
```

## Indexes on Partitioned Tables

```sql
-- Index on partitioned table creates index on all partitions
CREATE INDEX idx_sales_date ON sales(sale_date);

-- Automatically creates:
-- idx_sales_date_sales_2023_q1
-- idx_sales_date_sales_2023_q2
-- etc.

-- Per-partition index
CREATE INDEX idx_sales_2024_amount ON sales_2024(amount);
```

## Partitioning Existing Tables

```sql
-- 1. Create new partitioned table
CREATE TABLE sales_new (
    id BIGSERIAL,
    sale_date DATE NOT NULL,
    amount NUMERIC(10,2)
) PARTITION BY RANGE (sale_date);

-- 2. Create partitions
CREATE TABLE sales_new_2023 PARTITION OF sales_new
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

-- 3. Copy data
INSERT INTO sales_new SELECT * FROM sales;

-- 4. Rename tables
BEGIN;
ALTER TABLE sales RENAME TO sales_old;
ALTER TABLE sales_new RENAME TO sales;
COMMIT;

-- 5. Verify and drop old
SELECT count(*) FROM sales;
SELECT count(*) FROM sales_old;
DROP TABLE sales_old;
```

## Best Practices

```sql
-- 1. Partition key in queries
-- Good: Partition pruning works
SELECT * FROM sales WHERE sale_date = '2024-01-15';

-- Bad: Full scan (no partition key)
SELECT * FROM sales WHERE amount > 1000;

-- 2. Use constraints for validation
ALTER TABLE sales_2024 ADD CONSTRAINT check_2024
    CHECK (sale_date >= '2024-01-01' AND sale_date < '2025-01-01');

-- 3. Regular partition maintenance
-- Create next quarter's partition in advance
CREATE TABLE sales_2025_q2 PARTITION OF sales
    FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');
```

## Monitoring

```sql
-- List all partitions
SELECT
    nmsp_parent.nspname AS parent_schema,
    parent.relname AS parent_table,
    nmsp_child.nspname AS child_schema,
    child.relname AS child_table,
    pg_size_pretty(pg_total_relation_size(child.oid)) AS size
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
JOIN pg_namespace nmsp_parent ON nmsp_parent.oid = parent.relnamespace
JOIN pg_namespace nmsp_child ON nmsp_child.oid = child.relnamespace
ORDER BY parent.relname, child.relname;

-- Partition sizes
SELECT tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
FROM pg_tables
WHERE tablename LIKE 'sales_%'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

**Next:** [14 - Locking and Concurrency →](../14-locking-concurrency/README.md)
