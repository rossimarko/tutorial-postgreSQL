# 17 - High Availability

## Streaming Replication

### Primary Server Setup

```ini
# postgresql.conf (primary)
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
```

```sql
-- Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_pwd';
```

```ini
# pg_hba.conf (primary)
host replication replicator 10.0.1.0/24 scram-sha-256
```

### Standby Server Setup

```bash
# On standby server
pg_basebackup -h primary_server -D /var/lib/postgresql/17/main \
    -U replicator -Fp -Xs -P -R

# -R creates standby.signal and configures recovery

# Start standby
systemctl start postgresql
```

### Monitor Replication

```sql
-- On primary
SELECT client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) as send_lag,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as replay_lag
FROM pg_stat_replication;

-- On standby
SELECT now() - pg_last_xact_replay_timestamp() as replication_lag;
```

## Logical Replication

### Publisher Setup

```sql
-- Create publication (primary)
CREATE PUBLICATION my_publication FOR ALL TABLES;

-- Or specific tables
CREATE PUBLICATION my_publication FOR TABLE users, orders;
```

### Subscriber Setup

```sql
-- Create subscription (standby)
CREATE SUBSCRIPTION my_subscription
    CONNECTION 'host=primary_server dbname=mydb user=replicator password=pwd'
    PUBLICATION my_publication;
```

### Monitor Logical Replication

```sql
-- Publisher
SELECT * FROM pg_stat_replication;

-- Subscriber
SELECT * FROM pg_stat_subscription;
```

## Patroni (Automated Failover)

### Installation

```bash
pip3 install patroni[etcd]
sudo apt install etcd
```

### Patroni Configuration

```yaml
# /etc/patroni/patroni.yml
scope: postgres-cluster
name: node1

restapi:
  listen: 0.0.0.0:8008
  connect_address: 10.0.1.10:8008

etcd:
  hosts: 10.0.1.20:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 100
        shared_buffers: 2GB

postgresql:
  listen: 0.0.0.0:5432
  connect_address: 10.0.1.10:5432
  data_dir: /var/lib/postgresql/17/main
  authentication:
    replication:
      username: replicator
      password: secure_pwd
    superuser:
      username: postgres
      password: secure_pwd
```

### Start Patroni

```bash
patroni /etc/patroni/patroni.yml
```

### Patroni Operations

```bash
# Cluster status
patronictl -c /etc/patroni/patroni.yml list

# Switchover (planned)
patronictl -c /etc/patroni/patroni.yml switchover

# Failover (unplanned)
patronictl -c /etc/patroni/patroni.yml failover
```

## pgBackRest (Backup Solution)

### Installation

```bash
sudo apt install pgbackrest
```

### Configuration

```ini
# /etc/pgbackrest/pgbackrest.conf
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=4

[main]
pg1-path=/var/lib/postgresql/17/main
pg1-port=5432
```

### Usage

```bash
# Full backup
pgbackrest --stanza=main --type=full backup

# Incremental backup
pgbackrest --stanza=main --type=incr backup

# Restore
pgbackrest --stanza=main restore

# PITR
pgbackrest --stanza=main --type=time \
    --target="2024-01-15 14:30:00" restore
```

## Load Balancing (HAProxy)

### Configuration

```
# /etc/haproxy/haproxy.cfg
global
    maxconn 100

defaults
    mode tcp
    timeout connect 5s
    timeout client 30s
    timeout server 30s

# PostgreSQL primary (read-write)
listen postgres_primary
    bind *:5432
    option httpchk
    server pg1 10.0.1.10:5432 check port 8008
    server pg2 10.0.1.11:5432 check port 8008 backup

# PostgreSQL replicas (read-only)
listen postgres_replicas
    bind *:5433
    option httpchk
    balance roundrobin
    server pg2 10.0.1.11:5432 check port 8008
    server pg3 10.0.1.12:5432 check port 8008
```

## Monitoring HA Setup

```sql
-- Replication lag
SELECT client_addr,
       state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) as lag_bytes,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)) as lag
FROM pg_stat_replication;

-- Standby status
SELECT pg_is_in_recovery();  -- true on standby, false on primary

-- Last WAL received
SELECT pg_last_wal_receive_lsn();

-- Last WAL replayed
SELECT pg_last_wal_replay_lsn();
```

---

**Next:** [18 - Extensions →](../18-extensions/README.md)
