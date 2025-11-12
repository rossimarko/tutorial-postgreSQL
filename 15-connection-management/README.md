# 15 - Connection Management

## Connection Limits

```sql
-- View max connections
SHOW max_connections;  -- Default: 100

-- Current connections
SELECT count(*) FROM pg_stat_activity;

-- Connections by database
SELECT datname, count(*)
FROM pg_stat_activity
GROUP BY datname;

-- Increase max_connections
ALTER SYSTEM SET max_connections = 200;
-- Requires restart
```

## PgBouncer (Connection Pooling)

### Installation

```bash
sudo apt install pgbouncer -y
```

### Configuration

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
listen_addr = *
listen_port = 6432
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3

# Authentication
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt

# Logging
log_connections = 1
log_disconnections = 1
```

### Pool Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `session` | One connection per session | Default, safest |
| `transaction` | Release after transaction | Best performance |
| `statement` | Release after statement | Rarely used |

### Application Connection

```python
# Instead of connecting to PostgreSQL directly:
# conn = psycopg2.connect("host=localhost port=5432 dbname=mydb")

# Connect through PgBouncer:
conn = psycopg2.connect("host=localhost port=6432 dbname=mydb")
```

## Connection Pooling in Application

### Python (psycopg2)

```python
from psycopg2 import pool

# Create connection pool
connection_pool = pool.ThreadedConnectionPool(
    minconn=10,
    maxconn=50,
    host="localhost",
    database="mydb",
    user="postgres"
)

# Get connection
conn = connection_pool.getconn()

try:
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users LIMIT 10")
    results = cursor.fetchall()
finally:
    # Return connection to pool
    connection_pool.putconn(conn)
```

## Connection Timeouts

```sql
-- Idle connection timeout
ALTER SYSTEM SET idle_in_transaction_session_timeout = '10min';

-- Statement timeout
ALTER SYSTEM SET statement_timeout = '30s';

-- Lock timeout
ALTER SYSTEM SET lock_timeout = '5s';

-- TCP keepalive
ALTER SYSTEM SET tcp_keepalives_idle = 60;
ALTER SYSTEM SET tcp_keepalives_interval = 10;
ALTER SYSTEM SET tcp_keepalives_count = 3;

SELECT pg_reload_conf();
```

## Prepared Statements

```sql
-- Prepare statement
PREPARE get_user (INT) AS
    SELECT * FROM users WHERE id = $1;

-- Execute (faster on repeated calls)
EXECUTE get_user(100);
EXECUTE get_user(200);

-- Deallocate
DEALLOCATE get_user;
```

### Application Example

```python
# Without prepared statement (plan every time)
cursor.execute("SELECT * FROM users WHERE id = %s", (100,))

# With prepared statement (plan once)
cursor.execute("PREPARE get_user (INT) AS SELECT * FROM users WHERE id = $1")
cursor.execute("EXECUTE get_user(100)")
cursor.execute("EXECUTE get_user(200)")
```

## Monitoring Connections

```sql
-- Active connections
SELECT pid, usename, application_name, client_addr, state,
       backend_start, query_start, state_change, query
FROM pg_stat_activity
WHERE state = 'active';

-- Connection age
SELECT pid, usename,
       now() - backend_start as connection_age,
       state, query
FROM pg_stat_activity
ORDER BY backend_start;

-- Kill connections
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE pid = 12345;

-- Kill all connections to database
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE datname = 'mydb' AND pid <> pg_backend_pid();
```

## Reserved Connections

```sql
-- Reserve connections for superuser
ALTER SYSTEM SET superuser_reserved_connections = 3;

-- Total available = max_connections - superuser_reserved_connections
-- Example: 100 - 3 = 97 connections for regular users
```

---

**Next:** [16 - Security and Permissions →](../16-security-permissions/README.md)
