# 01 - Installation and Configuration

## Table of Contents
- [Installation](#installation)
- [postgresql.conf](#postgresqlconf)
- [pg_hba.conf](#pg_hbaconf)
- [Connection Pooling](#connection-pooling)
- [Anti-Patterns](#anti-patterns)
- [Monitoring](#monitoring)
- [Real-World Scenarios](#real-world-scenarios)

## Installation

### Installation Methods

#### Ubuntu/Debian
```bash
# Add PostgreSQL repository
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null

# Install PostgreSQL 17
sudo apt update
sudo apt install postgresql-17 postgresql-contrib-17 -y

# Check status
sudo systemctl status postgresql
```

#### RHEL/CentOS/Rocky Linux
```bash
# Install repository
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Disable built-in PostgreSQL module
sudo dnf -qy module disable postgresql

# Install PostgreSQL 17
sudo dnf install -y postgresql17-server postgresql17-contrib

# Initialize and start
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
sudo systemctl enable postgresql-17 --now
```

#### Docker (Recommended for Testing)
```bash
# Run PostgreSQL 17
docker run -d \
  --name postgres17 \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_DB=testdb \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:17-alpine

# Connect
docker exec -it postgres17 psql -U postgres
```

### Directory Structure
```
/var/lib/postgresql/17/main/   # Data directory (Debian/Ubuntu)
├── postgresql.conf            # Main configuration
├── pg_hba.conf               # Client authentication
├── pg_ident.conf             # User name mapping
├── base/                      # Database files
├── global/                    # Cluster-wide tables
├── pg_wal/                   # Write-Ahead Log files
├── pg_xact/                  # Transaction commit status
├── pg_multixact/             # Multixact status
└── pg_stat_tmp/              # Temporary statistics files
```

## postgresql.conf

### Critical Parameters by Category

#### Memory Configuration
```ini
# Shared Buffers (25% of RAM, max 40%)
shared_buffers = 8GB                    # For 32GB RAM server

# Work Memory (per operation)
work_mem = 64MB                         # For sorting, hash joins
maintenance_work_mem = 1GB              # For VACUUM, CREATE INDEX
autovacuum_work_mem = 1GB               # Separate for autovacuum

# Effective Cache Size (50-75% of RAM)
effective_cache_size = 24GB             # OS + PostgreSQL cache
```

**Performance Comparison:**
```sql
-- Test with work_mem = 4MB (default)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table
ORDER BY column1
LIMIT 100;

-- Result: External merge sort, 500ms
-- Disk Sorts: 15
-- Peak Memory: 4MB

-- Test with work_mem = 64MB
SET work_mem = '64MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM large_table
ORDER BY column1
LIMIT 100;

-- Result: Quicksort in memory, 45ms (11x faster)
-- Disk Sorts: 0
-- Peak Memory: 62MB
```

#### WAL Configuration
```ini
# WAL Level (minimal, replica, logical)
wal_level = replica                     # Required for replication

# WAL Buffers (typically auto-tuned)
wal_buffers = 16MB                      # -1 = auto (1/32 of shared_buffers)

# Checkpoint Configuration
checkpoint_timeout = 15min              # Max time between checkpoints
max_wal_size = 4GB                      # Trigger checkpoint at this size
min_wal_size = 1GB                      # Recycle WAL files below this

# Checkpoint Completion Target
checkpoint_completion_target = 0.9      # Spread checkpoint I/O over 90% of interval

# WAL Compression (PostgreSQL 15+)
wal_compression = zstd                  # zstd, lz4, or on
```

#### Query Planner
```ini
# Cost-based optimizer parameters
random_page_cost = 1.1                  # SSD: 1.1, HDD: 4.0
effective_io_concurrency = 200          # For SSDs
seq_page_cost = 1.0                     # Sequential scan cost

# Parallel Query Execution
max_parallel_workers_per_gather = 4     # Per query
max_parallel_workers = 8                # Total parallel workers
max_worker_processes = 8                # Background workers

# JIT Compilation (PostgreSQL 11+)
jit = on                                # Enable JIT compilation
jit_above_cost = 100000                 # Use JIT for expensive queries
```

#### Connection and Resource Limits
```ini
max_connections = 100                   # Use connection pooling for more!
superuser_reserved_connections = 3      # Reserved for superuser

# Statement Timeout
statement_timeout = 30s                 # Kill queries after 30s
idle_in_transaction_session_timeout = 10min  # Kill idle transactions
```

#### Logging Configuration
```ini
# Where to Log
logging_collector = on                  # Enable log collector
log_directory = 'log'                   # Relative to data directory
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d                   # Rotate daily
log_rotation_size = 100MB               # Rotate at 100MB

# What to Log
log_min_duration_statement = 1000       # Log queries > 1s
log_checkpoints = on                    # Log checkpoint activity
log_connections = on                    # Log connections
log_disconnections = on                 # Log disconnections
log_lock_waits = on                     # Log lock waits
log_temp_files = 0                      # Log all temp files

# Log Line Prefix
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

### Configuration Best Practices

**Find your config file:**
```sql
-- Show all configuration files
SELECT name, setting
FROM pg_settings
WHERE name IN ('config_file', 'hba_file', 'ident_file', 'data_directory');
```

**Reload configuration:**
```sql
-- Reload without restart (for parameters that don't require restart)
SELECT pg_reload_conf();

-- Check if restart is needed
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;
```

## pg_hba.conf

### Authentication Methods

```ini
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# Local connections (Unix socket)
local   all             postgres                                peer
local   all             all                                     md5

# IPv4 local connections
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             10.0.0.0/8              scram-sha-256

# IPv6 local connections
host    all             all             ::1/128                 scram-sha-256

# Replication connections
host    replication     replicator      10.0.1.0/24             scram-sha-256

# SSL-only connections (PostgreSQL 16+)
hostssl all             all             0.0.0.0/0               scram-sha-256 clientcert=verify-full

# Specific database and user
host    production_db   app_user        10.0.2.100/32           scram-sha-256
```

### Authentication Methods Explained

| Method | Security | Use Case |
|--------|----------|----------|
| `trust` | None | Local development only |
| `peer` | OS-level | Unix socket, same username |
| `md5` | Moderate | Legacy systems (deprecated) |
| `scram-sha-256` | High | Modern default (PostgreSQL 10+) |
| `cert` | Very High | SSL certificate authentication |
| `ldap` | Varies | Enterprise directory integration |

### Security Best Practices

```ini
# ❌ NEVER DO THIS IN PRODUCTION
host    all             all             0.0.0.0/0               trust

# ✅ PRODUCTION CONFIGURATION
# Application connections
hostssl production_db   app_user        10.0.2.0/24             scram-sha-256

# Admin connections (specific IP)
hostssl all             postgres        203.0.113.10/32         scram-sha-256 clientcert=verify-full

# Monitoring connections
host    all             monitoring      127.0.0.1/32            scram-sha-256

# Replication connections (SSL required)
hostssl replication     replicator      10.0.1.0/24             scram-sha-256
```

**Reload pg_hba.conf:**
```sql
SELECT pg_reload_conf();
```

## Connection Pooling

### Why Connection Pooling?

**Problem: Direct Connections**
```
Application Servers (100)
        ↓
PostgreSQL (max_connections=100)
= Exhausted, slow context switching
```

**Solution: Connection Pooling**
```
Application Servers (100)
        ↓
PgBouncer (100 connections to apps)
        ↓
PostgreSQL (20 actual connections)
= Efficient, scalable
```

### PgBouncer Configuration

**Installation:**
```bash
# Ubuntu/Debian
sudo apt install pgbouncer -y

# RHEL/Rocky
sudo dnf install pgbouncer -y
```

**Configuration: /etc/pgbouncer/pgbouncer.ini**
```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
# Connection pool mode
pool_mode = transaction              # transaction, session, or statement

# Connection limits
max_client_conn = 1000               # Max client connections
default_pool_size = 25               # Connections per database/user
reserve_pool_size = 5                # Emergency connections
reserve_pool_timeout = 3             # Seconds to wait for free connection

# Timeouts
server_idle_timeout = 600            # Close idle server connections after 10min
query_timeout = 300                  # Kill queries after 5min

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1

# Authentication
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Listen
listen_addr = *
listen_port = 6432
```

**User Authentication: /etc/pgbouncer/userlist.txt**
```
"myuser" "SCRAM-SHA-256$4096:..."
```

**Generate password hash:**
```bash
# For PostgreSQL user
psql -U postgres -c "SELECT concat('\"', usename, '\" \"', passwd, '\"') FROM pg_shadow WHERE usename = 'myuser';"
```

### Pool Mode Comparison

| Mode | Transaction Lifespan | Use Case | Restrictions |
|------|---------------------|----------|--------------|
| **session** | Entire session | Default, safest | None |
| **transaction** | One transaction | Best performance | No prepared statements |
| **statement** | One query | Rare use | Very restrictive |

**Performance Comparison:**
```bash
# Without pooling: 100 concurrent connections
# Result: 500 TPS, high CPU, connection overhead

# With PgBouncer (transaction mode): 20 PostgreSQL connections
# Result: 2500 TPS (5x faster), low CPU
```

## Anti-Patterns

### ❌ Anti-Pattern 1: Excessive Connections
```python
# BAD: Creating new connection per query
def get_user(user_id):
    conn = psycopg2.connect("dbname=mydb")
    cur = conn.cursor()
    cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    result = cur.fetchone()
    conn.close()
    return result
```

**Impact:**
- High connection overhead (50-100ms per connection)
- Resource exhaustion
- Connection limit reached quickly

**Solution:**
```python
# GOOD: Use connection pool
from psycopg2 import pool

connection_pool = pool.SimpleConnectionPool(10, 50, "dbname=mydb")

def get_user(user_id):
    conn = connection_pool.getconn()
    try:
        cur = conn.cursor()
        cur.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        return cur.fetchone()
    finally:
        connection_pool.putconn(conn)
```

### ❌ Anti-Pattern 2: Default shared_buffers
```ini
# BAD: Using tiny default on 64GB RAM server
shared_buffers = 128MB
```

**Impact:**
- Poor cache hit ratio
- Excessive disk I/O
- Slow query performance

**Solution:**
```ini
# GOOD: Scale with available RAM
shared_buffers = 16GB  # 25% of 64GB RAM
```

### ❌ Anti-Pattern 3: Not using SSL
```ini
# BAD: Plaintext connections
host    all             all             0.0.0.0/0               scram-sha-256
```

**Solution:**
```ini
# GOOD: Require SSL
hostssl all             all             0.0.0.0/0               scram-sha-256
```

## Monitoring

### Configuration Monitoring Queries

**Current configuration values:**
```sql
-- All settings
SELECT name, setting, unit, context
FROM pg_settings
ORDER BY name;

-- Settings requiring restart
SELECT name, setting, pending_restart
FROM pg_settings
WHERE pending_restart = true;

-- Memory settings
SELECT name,
       setting,
       unit,
       pg_size_pretty(setting::bigint *
         CASE unit
           WHEN 'kB' THEN 1024
           WHEN 'MB' THEN 1024^2
           WHEN 'GB' THEN 1024^3
           ELSE 1
         END) as pretty_value
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'maintenance_work_mem',
               'effective_cache_size', 'wal_buffers')
ORDER BY name;
```

**Connection monitoring:**
```sql
-- Current connections by database
SELECT datname,
       count(*) as connections,
       max(max_connections::int) as max_connections,
       round(100.0 * count(*) / max(max_connections::int), 2) as pct_used
FROM pg_stat_activity,
     (SELECT setting::int as max_connections FROM pg_settings WHERE name = 'max_connections') s
GROUP BY datname
ORDER BY connections DESC;

-- Connection states
SELECT state,
       count(*),
       max(now() - state_change) as longest
FROM pg_stat_activity
WHERE pid != pg_backend_pid()
GROUP BY state;
```

**WAL monitoring:**
```sql
-- Current WAL write location
SELECT pg_current_wal_lsn();

-- WAL file size and count
SELECT
    count(*) as wal_files,
    pg_size_pretty(count(*) * 16777216) as total_size
FROM pg_ls_waldir();

-- WAL generation rate
SELECT
    pg_size_pretty(
        (pg_current_wal_lsn() - '0/0'::pg_lsn)
    ) as wal_generated;
```

## Real-World Scenarios

### Scenario 1: Application Connection Exhaustion

**Symptoms:**
```
FATAL: sorry, too many clients already
```

**Investigation:**
```sql
-- Check current connections
SELECT count(*), state
FROM pg_stat_activity
GROUP BY state;

-- Identify connection sources
SELECT client_addr,
       count(*),
       array_agg(DISTINCT application_name)
FROM pg_stat_activity
WHERE client_addr IS NOT NULL
GROUP BY client_addr
ORDER BY count(*) DESC;
```

**Solution:**
1. Implement PgBouncer with transaction pooling
2. Fix application connection leaks
3. Set connection timeout: `idle_in_transaction_session_timeout = 10min`

### Scenario 2: Slow Query Performance After Upgrade

**Investigation:**
```sql
-- Check if statistics are current
SELECT schemaname, tablename, last_analyze, last_autoanalyze
FROM pg_stat_user_tables
WHERE last_analyze IS NULL
  AND last_autoanalyze IS NULL;

-- Run ANALYZE
ANALYZE VERBOSE;
```

**Solution:**
```bash
# After major version upgrade, always run:
vacuumdb -U postgres -a -Z -v  # ANALYZE all databases
```

### Scenario 3: Checkpoint Warning Floods

**Symptoms in logs:**
```
LOG: checkpoints are occurring too frequently (24 seconds apart)
HINT: Consider increasing the configuration parameter "max_wal_size"
```

**Solution:**
```ini
# Increase WAL size
max_wal_size = 4GB  # Up from 1GB
checkpoint_timeout = 15min
checkpoint_completion_target = 0.9
```

**Verify:**
```sql
-- Monitor checkpoint statistics
SELECT * FROM pg_stat_bgwriter;
```

### Scenario 4: Memory Tuning for 64GB Server

**Initial Configuration:**
```ini
# Analyze current usage
SELECT pg_size_pretty(sum(heap_blks_hit) * 8192) as buffer_cache_hit,
       pg_size_pretty(sum(heap_blks_read) * 8192) as disk_read,
       round(100.0 * sum(heap_blks_hit) /
         NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0), 2) as cache_hit_ratio
FROM pg_statio_user_tables;
```

**Optimized Configuration for 64GB RAM:**
```ini
# Memory
shared_buffers = 16GB                   # 25% of RAM
effective_cache_size = 48GB             # 75% of RAM
work_mem = 64MB                         # 16GB / 250 connections
maintenance_work_mem = 2GB              # For VACUUM, CREATE INDEX

# WAL
wal_buffers = 16MB                      # Auto-tuned
max_wal_size = 4GB

# Parallelism
max_parallel_workers_per_gather = 4
max_parallel_workers = 8
max_worker_processes = 8
```

---

## Quick Reference

### Essential Commands
```bash
# Reload configuration
sudo systemctl reload postgresql

# Restart PostgreSQL
sudo systemctl restart postgresql

# Check configuration syntax
postgres -C config_file=/etc/postgresql/17/main/postgresql.conf -D /var/lib/postgresql/17/main --check

# View logs
sudo tail -f /var/log/postgresql/postgresql-17-main.log
```

### Next Steps
- [02 - Database Creation →](../02-database-creation/README.md)
- [Back to Main](../README.md)
