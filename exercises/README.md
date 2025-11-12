# PostgreSQL DBA Exercises

## Exercise 1: Database Setup and Configuration

### Objective
Set up a PostgreSQL database with optimal configuration for a 16GB RAM server.

### Tasks
1. Install PostgreSQL 17
2. Configure memory parameters for 16GB RAM
3. Enable WAL archiving
4. Set up connection limits and timeouts

### Solution
```bash
# Install PostgreSQL 17
sudo apt install postgresql-17

# Edit postgresql.conf
sudo nano /etc/postgresql/17/main/postgresql.conf
```

```ini
# Memory (16GB server)
shared_buffers = 4GB
work_mem = 64MB
maintenance_work_mem = 1GB
effective_cache_size = 12GB

# WAL
wal_level = replica
max_wal_size = 2GB
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'

# Connections
max_connections = 100
idle_in_transaction_session_timeout = 10min
statement_timeout = 30s
```

## Exercise 2: Index Optimization

### Objective
Optimize a slow query using appropriate indexes.

### Setup
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    category TEXT,
    price NUMERIC(10,2),
    created_at TIMESTAMP DEFAULT now()
);

INSERT INTO products (name, category, price)
SELECT
    'Product ' || g,
    CASE (random() * 5)::INT
        WHEN 0 THEN 'Electronics'
        WHEN 1 THEN 'Clothing'
        WHEN 2 THEN 'Books'
        WHEN 3 THEN 'Home'
        ELSE 'Sports'
    END,
    (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 1000000) g;
```

### Problem Query (Slow)
```sql
EXPLAIN ANALYZE
SELECT * FROM products
WHERE category = 'Electronics'
  AND price BETWEEN 100 AND 500
ORDER BY created_at DESC
LIMIT 10;
```

### Tasks
1. Identify the performance issue
2. Create appropriate index(es)
3. Verify performance improvement
4. Measure speedup

### Solution
```sql
-- Before: Sequential scan (~200ms)
-- Create composite index
CREATE INDEX idx_products_category_price_created
ON products(category, price, created_at DESC);

-- Verify index usage
EXPLAIN ANALYZE
SELECT * FROM products
WHERE category = 'Electronics'
  AND price BETWEEN 100 AND 500
ORDER BY created_at DESC
LIMIT 10;

-- After: Index scan (~2ms, 100x faster)

-- Measure improvement
\timing on
-- Run query and compare times
```

## Exercise 3: Query Optimization

### Problem
```sql
-- Slow query with N+1 problem
SELECT u.username,
       (SELECT count(*) FROM orders WHERE user_id = u.id) as order_count,
       (SELECT sum(total) FROM orders WHERE user_id = u.id) as total_spent
FROM users u
WHERE u.status = 'active';
```

### Tasks
1. Identify the issue
2. Rewrite using JOIN
3. Compare EXPLAIN ANALYZE outputs

### Solution
```sql
-- Optimized query
SELECT u.username,
       count(o.id) as order_count,
       COALESCE(sum(o.total), 0) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.id, u.username;

-- Performance comparison:
-- Before: 2.5s (1000 users × 2 subqueries)
-- After: 45ms (single JOIN + GROUP BY)
```

## Exercise 4: Partitioning

### Objective
Convert a large time-series table to partitioned table.

### Setup
```sql
CREATE TABLE sensor_data (
    id BIGSERIAL,
    sensor_id INT,
    value NUMERIC,
    recorded_at TIMESTAMP
);

-- Insert 10M rows
INSERT INTO sensor_data (sensor_id, value, recorded_at)
SELECT
    (random() * 100)::INT,
    random() * 100,
    '2023-01-01'::TIMESTAMP + (g || ' seconds')::INTERVAL
FROM generate_series(1, 10000000) g;
```

### Tasks
1. Create partitioned table by month
2. Migrate existing data
3. Set up automatic partition creation
4. Verify query performance improvement

### Solution
```sql
-- 1. Create partitioned table
CREATE TABLE sensor_data_partitioned (
    id BIGSERIAL,
    sensor_id INT,
    value NUMERIC,
    recorded_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (recorded_at);

-- 2. Create partitions
CREATE TABLE sensor_data_2023_01 PARTITION OF sensor_data_partitioned
    FOR VALUES FROM ('2023-01-01') TO ('2023-02-01');

CREATE TABLE sensor_data_2023_02 PARTITION OF sensor_data_partitioned
    FOR VALUES FROM ('2023-02-01') TO ('2023-03-01');

-- Continue for all months...

-- 3. Migrate data
INSERT INTO sensor_data_partitioned SELECT * FROM sensor_data;

-- 4. Verify performance
EXPLAIN ANALYZE
SELECT * FROM sensor_data_partitioned
WHERE recorded_at BETWEEN '2023-01-15' AND '2023-01-20';

-- Result: Only sensor_data_2023_01 scanned (10x faster)
```

## Exercise 5: Vacuum and Bloat Management

### Objective
Detect and remove table bloat.

### Setup
```sql
CREATE TABLE test_bloat (
    id SERIAL PRIMARY KEY,
    data TEXT
);

INSERT INTO test_bloat (data)
SELECT md5(random()::TEXT) FROM generate_series(1, 1000000);

-- Create bloat
UPDATE test_bloat SET data = md5(random()::TEXT);
UPDATE test_bloat SET data = md5(random()::TEXT);
UPDATE test_bloat SET data = md5(random()::TEXT);
```

### Tasks
1. Measure table size and bloat
2. Run VACUUM and measure impact
3. Compare VACUUM vs VACUUM FULL
4. Set up autovacuum tuning

### Solution
```sql
-- 1. Check size and bloat
SELECT pg_size_pretty(pg_total_relation_size('test_bloat')) as total_size;
SELECT n_dead_tup, n_live_tup FROM pg_stat_user_tables WHERE tablename = 'test_bloat';

-- 2. VACUUM
VACUUM VERBOSE test_bloat;
SELECT pg_size_pretty(pg_total_relation_size('test_bloat')) as size_after_vacuum;

-- 3. VACUUM FULL (locks table!)
VACUUM FULL VERBOSE test_bloat;
SELECT pg_size_pretty(pg_total_relation_size('test_bloat')) as size_after_full;

-- 4. Tune autovacuum
ALTER TABLE test_bloat
SET (autovacuum_vacuum_scale_factor = 0.05);
```

## Exercise 6: Backup and Recovery

### Objective
Implement full backup and point-in-time recovery.

### Tasks
1. Configure WAL archiving
2. Take base backup
3. Simulate data loss
4. Perform PITR to before data loss

### Solution
```bash
# 1. Configure WAL archiving
# postgresql.conf:
# wal_level = replica
# archive_mode = on
# archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'

sudo systemctl restart postgresql

# 2. Take base backup
pg_basebackup -D /backup/base_$(date +%Y%m%d) -Ft -z -P -X stream

# 3. Note current time and create data
psql -c "CREATE TABLE important_data AS SELECT generate_series(1, 1000) as id;"
RECOVERY_TIME=$(date "+%Y-%m-%d %H:%M:%S")
sleep 5

# 4. Simulate data loss
psql -c "DROP TABLE important_data;"

# 5. Perform PITR
sudo systemctl stop postgresql
sudo mv /var/lib/postgresql/17/main /var/lib/postgresql/17/main.old
sudo mkdir -p /var/lib/postgresql/17/main
sudo tar -xzf /backup/base_*/base.tar.gz -C /var/lib/postgresql/17/main
sudo touch /var/lib/postgresql/17/main/recovery.signal
sudo bash -c "cat > /var/lib/postgresql/17/main/postgresql.auto.conf <<EOF
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
recovery_target_time = '$RECOVERY_TIME'
recovery_target_action = 'promote'
EOF"
sudo chown -R postgres:postgres /var/lib/postgresql/17/main
sudo systemctl start postgresql

# 6. Verify recovery
psql -c "SELECT count(*) FROM important_data;"
```

## Exercise 7: Monitoring Dashboard

### Objective
Create a comprehensive monitoring query.

### Solution
```sql
-- Create monitoring view
CREATE OR REPLACE VIEW db_health AS
WITH connection_stats AS (
    SELECT
        count(*) as total_connections,
        count(*) FILTER (WHERE state = 'active') as active,
        count(*) FILTER (WHERE state = 'idle') as idle,
        count(*) FILTER (WHERE state = 'idle in transaction') as idle_in_tx
    FROM pg_stat_activity
),
cache_stats AS (
    SELECT
        round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit + blks_read), 0), 2) as cache_hit_ratio
    FROM pg_stat_database
),
bloat_stats AS (
    SELECT
        count(*) FILTER (WHERE n_dead_tup::FLOAT / NULLIF(n_live_tup, 0) > 0.1) as bloated_tables
    FROM pg_stat_user_tables
    WHERE n_live_tup > 1000
)
SELECT
    (SELECT total_connections FROM connection_stats),
    (SELECT active FROM connection_stats),
    (SELECT idle FROM connection_stats),
    (SELECT idle_in_tx FROM connection_stats),
    (SELECT cache_hit_ratio FROM cache_stats),
    (SELECT bloated_tables FROM bloat_stats),
    pg_size_pretty(pg_database_size(current_database())) as db_size;

-- Query monitoring view
SELECT * FROM db_health;
```

---

## More Exercises

See individual module READMEs for specific exercises:
- [01 - Installation and Configuration](../01-installation-configuration/README.md#real-world-scenarios)
- [06 - Indexes](../06-indexes/README.md#real-world-scenarios)
- [13 - Table Partitioning](../13-table-partitioning/README.md)

**Next:** [Scripts →](../scripts/README.md) | [Back to Main](../README.md)
