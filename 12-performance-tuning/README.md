# 12 - Performance Tuning

## Memory Configuration

### Shared Buffers

```ini
# postgresql.conf
# 25% of RAM for dedicated DB server
# 10-15% for shared server
shared_buffers = 8GB  # For 32GB RAM server
```

### Work Memory

```ini
# Per-operation memory (sort, hash)
work_mem = 64MB

# Formula: (Total RAM * 0.25) / max_connections
# Example: (32GB * 0.25) / 100 = 80MB
```

### Maintenance Work Memory

```ini
# For VACUUM, CREATE INDEX, ALTER TABLE
maintenance_work_mem = 2GB

# Larger = faster maintenance operations
```

### Effective Cache Size

```ini
# OS + PostgreSQL cache (50-75% of RAM)
effective_cache_size = 24GB  # For 32GB RAM

# Influences query planner decisions
```

## WAL Configuration

```ini
# WAL settings for performance
wal_buffers = 16MB           # Auto-tuned (1/32 of shared_buffers)
checkpoint_timeout = 15min
max_wal_size = 4GB
min_wal_size = 1GB
checkpoint_completion_target = 0.9

# WAL compression (PostgreSQL 15+)
wal_compression = zstd
```

## Query Planner

```ini
# For SSDs
random_page_cost = 1.1       # Default: 4.0 (HDD)
effective_io_concurrency = 200  # SSD capability

# Parallelism
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_worker_processes = 8
```

## Connection Pooling

### PgBouncer Configuration

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
```

## Performance Benchmarking

### pgbench

```bash
# Initialize test database
pgbench -i -s 100 testdb

# Run benchmark (10 clients, 2 threads, 60 seconds)
pgbench -c 10 -j 2 -T 60 testdb

# Results:
# tps = 1234.567 (including connections establishing)
# tps = 1245.678 (excluding connections establishing)
```

## Operating System Tuning

### Linux Kernel Parameters

```bash
# /etc/sysctl.conf

# Shared memory
kernel.shmmax = 17179869184  # 16GB
kernel.shmall = 4194304      # 16GB / page_size

# Swap
vm.swappiness = 1            # Minimize swap usage

# Network
net.ipv4.tcp_keepalive_time = 200
net.ipv4.tcp_keepalive_intvl = 200
net.ipv4.tcp_keepalive_probes = 5

# Apply changes
sudo sysctl -p
```

### File System

```bash
# Mount options for PostgreSQL data directory
# /etc/fstab
/dev/sdb1  /var/lib/postgresql  ext4  noatime,nodiratime  0  2

# noatime = don't update access time (faster)
```

## Monitoring Performance

```sql
-- Cache hit ratio (should be >99%)
SELECT
    sum(heap_blks_hit) / NULLIF(sum(heap_blks_hit + heap_blks_read), 0) * 100 as cache_hit_ratio
FROM pg_statio_user_tables;

-- Query performance
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 10;

-- Table bloat
SELECT tablename, n_dead_tup, n_live_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup, 0), 2) as bloat_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 0
ORDER BY bloat_pct DESC;
```

## Quick Wins

```sql
-- 1. Add missing indexes
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- 2. Update statistics
ANALYZE;

-- 3. Remove unused indexes
DROP INDEX unused_idx;

-- 4. Increase work_mem for specific queries
SET work_mem = '256MB';

-- 5. Enable pg_stat_statements
CREATE EXTENSION pg_stat_statements;
```

## Performance Checklist

- [ ] Shared buffers = 25% of RAM
- [ ] Work_mem tuned for workload
- [ ] random_page_cost = 1.1 for SSDs
- [ ] Indexes on foreign keys
- [ ] Regular VACUUM and ANALYZE
- [ ] Connection pooling configured
- [ ] pg_stat_statements enabled
- [ ] Checkpoint settings optimized
- [ ] Parallel query enabled

---

**Next:** [13 - Table Partitioning →](../13-table-partitioning/README.md)
