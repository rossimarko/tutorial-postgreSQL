# PostgreSQL DBA Tutorial Repository

Comprehensive PostgreSQL Database Administration tutorial covering fundamentals to advanced topics for PostgreSQL 15, 16, and 17.

## Overview

This repository provides in-depth, hands-on learning materials for mastering PostgreSQL database administration. Each module includes theory, practical examples, performance comparisons, anti-patterns, monitoring queries, and real-world scenarios.

## Quick Start

### Docker Setup (Recommended)

```bash
# Clone repository
git clone https://github.com/yourusername/tutorial-postgreSQL.git
cd tutorial-postgreSQL

# Start PostgreSQL environment
docker-compose up -d

# Connect to PostgreSQL
psql -h localhost -U postgres -d tutorial

# Access pgAdmin
# Open http://localhost:5050
# Email: admin@example.com
# Password: admin
```

### Local Installation

```bash
# Ubuntu/Debian
sudo apt install postgresql-17 postgresql-contrib-17

# RHEL/Rocky Linux
sudo dnf install postgresql17-server postgresql17-contrib

# Start PostgreSQL
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

## Repository Structure

```
tutorial-postgreSQL/
├── 01-installation-configuration/    # Installation, postgresql.conf, pg_hba.conf
├── 02-database-creation/             # Databases, tablespaces, schemas, roles
├── 03-wal-architecture/              # WAL internals, checkpoints, archiving
├── 04-backup-restore/                # pg_dump, pg_basebackup, PITR
├── 05-statistics/                    # ANALYZE, pg_statistic, extended stats
├── 06-indexes/                       # B-tree, Hash, GiST, GIN, BRIN, partial
├── 07-query-plans/                   # EXPLAIN, cost estimation, pg_stat_statements
├── 08-query-optimization/            # Joins, CTEs, parallel queries, JIT
├── 09-vacuum-autovacuum/             # VACUUM, bloat management, anti-wraparound
├── 10-routine-maintenance/           # REINDEX, CLUSTER, TOAST management
├── 11-monitoring-views/              # pg_stat_*, lock monitoring, replication
├── 12-performance-tuning/            # Memory, WAL, connection pooling, benchmarking
├── 13-table-partitioning/            # Range, list, hash partitioning
├── 14-locking-concurrency/           # MVCC, locks, deadlocks, isolation levels
├── 15-connection-management/         # PgBouncer, connection limits, prepared statements
├── 16-security-permissions/          # RLS, SSL/TLS, roles, encryption, audit
├── 17-high-availability/             # Replication, Patroni, pgBackRest, load balancing
├── 18-extensions/                    # pg_stat_statements, PostGIS, TimescaleDB
├── 19-migrations-deployments/        # Schema migrations, zero-downtime, blue-green
├── exercises/                        # Hands-on labs with solutions
├── scripts/                          # Monitoring and automation scripts
├── sample-databases/                 # Pre-configured sample databases
└── docker-compose.yml                # Docker environment setup
```

## Learning Path

### Section 1: Fundamentals (1-4 weeks)

Master the basics of PostgreSQL installation, configuration, and core architecture.

| Module | Topic | Key Concepts |
|--------|-------|--------------|
| [01](./01-installation-configuration) | Installation & Configuration | postgresql.conf, pg_hba.conf, connection pooling |
| [02](./02-database-creation) | Database Creation | Tablespaces, schemas, roles, permissions |
| [03](./03-wal-architecture) | WAL Architecture | WAL internals, checkpoints, fsync, archiving |
| [04](./04-backup-restore) | Backup & Restore | pg_dump, pg_basebackup, PITR strategies |

**Exercises:** [Database Setup](./exercises#exercise-1-database-setup-and-configuration) | [Backup and Recovery](./exercises#exercise-6-backup-and-recovery)

### Section 2: Performance Foundations (2-3 weeks)

Learn query optimization, indexing strategies, and performance analysis.

| Module | Topic | Key Concepts |
|--------|-------|--------------|
| [05](./05-statistics) | Statistics | ANALYZE, pg_statistic, histograms, extended stats |
| [06](./06-indexes) | Indexes | B-tree, Hash, GiST, GIN, BRIN, partial, expression |
| [07](./07-query-plans) | Query Plans | EXPLAIN ANALYZE, cost estimation, plan nodes |
| [08](./08-query-optimization) | Query Optimization | Join methods, CTEs, parallel queries, JIT |

**Exercises:** [Index Optimization](./exercises#exercise-2-index-optimization) | [Query Optimization](./exercises#exercise-3-query-optimization)

### Section 3: Maintenance & Monitoring (2-3 weeks)

Understand database maintenance, monitoring, and performance tuning.

| Module | Topic | Key Concepts |
|--------|-------|--------------|
| [09](./09-vacuum-autovacuum) | VACUUM & Autovacuum | Bloat management, autovacuum tuning, anti-wraparound |
| [10](./10-routine-maintenance) | Routine Maintenance | REINDEX, CLUSTER, table bloat, TOAST |
| [11](./11-monitoring-views) | Monitoring Views | pg_stat_*, pg_stat_activity, locks, connections |
| [12](./12-performance-tuning) | Performance Tuning | Shared_buffers, work_mem, WAL tuning, benchmarking |

**Exercises:** [Vacuum and Bloat](./exercises#exercise-5-vacuum-and-bloat-management) | [Monitoring Dashboard](./exercises#exercise-7-monitoring-dashboard)

### Section 4: Advanced Topics (3-4 weeks)

Dive into advanced features: partitioning, HA, security, and production deployments.

| Module | Topic | Key Concepts |
|--------|-------|--------------|
| [13](./13-table-partitioning) | Table Partitioning | Range, list, hash, partition pruning, management |
| [14](./14-locking-concurrency) | Locking & Concurrency | MVCC, lock types, deadlocks, advisory locks |
| [15](./15-connection-management) | Connection Management | PgBouncer, connection limits, prepared statements |
| [16](./16-security-permissions) | Security & Permissions | RLS, SSL/TLS, roles, encryption, pg_hba.conf |
| [17](./17-high-availability) | High Availability | Streaming replication, Patroni, pgBackRest |
| [18](./18-extensions) | Extensions | pg_stat_statements, pg_trgm, PostGIS, TimescaleDB |
| [19](./19-migrations-deployments) | Migrations & Deployments | Flyway, Liquibase, zero-downtime, blue-green |

**Exercises:** [Partitioning](./exercises#exercise-4-partitioning)

## Hands-On Practice

### Sample Databases

Three pre-configured databases for practicing concepts:

```bash
# Start Docker environment
docker-compose up -d

# Databases auto-created:
# - ecommerce: 100K users, 500K orders, 1.5M order items
# - analytics: 10M page views (partitioned)
# - timeseries: 50M sensor readings (TimescaleDB)
```

See [Sample Databases](./sample-databases) for details.

### Exercises

Progressive difficulty exercises with step-by-step solutions:

1. **Database Setup and Configuration** - Configure PostgreSQL for 16GB server
2. **Index Optimization** - Optimize slow query with proper indexes
3. **Query Optimization** - Rewrite queries to eliminate N+1 problem
4. **Partitioning** - Convert large table to partitioned table
5. **Vacuum and Bloat Management** - Detect and remove table bloat
6. **Backup and Recovery** - Implement PITR recovery
7. **Monitoring Dashboard** - Create comprehensive health check

See [Exercises](./exercises) for all labs.

### Scripts

Production-ready monitoring and automation scripts:

- **Monitoring:** health_check.sql, slow_queries.sql, blocking.sql
- **Maintenance:** backup.sh, vacuum_maintenance.sh, reindex_maintenance.sh
- **Automation:** create_partitions.sql, update_stats.sh

See [Scripts](./scripts) for all tools.

## Key Features

### Modern PostgreSQL Focus

- PostgreSQL 15, 16, and 17 best practices
- Latest features: JIT compilation, parallel query, partitioning improvements
- JSON/JSONB operations, generated columns, table access methods

### Comprehensive Coverage

- **Theory:** Concise explanations with visual diagrams
- **Examples:** Real SQL code with EXPLAIN ANALYZE outputs
- **Performance:** Before/after comparisons with metrics
- **Anti-Patterns:** What to avoid and why
- **Monitoring:** pg_stat_* queries for each topic
- **Scenarios:** Real-world production challenges

### Practical Tools

- Docker environment with PostgreSQL 17, pgAdmin, PgBouncer
- Sample databases with millions of rows
- Production-ready monitoring and automation scripts
- Progressive exercises with detailed solutions

## Quick Reference

### Essential Commands

```sql
-- Configuration
SHOW shared_buffers;
ALTER SYSTEM SET work_mem = '64MB';
SELECT pg_reload_conf();

-- Monitoring
SELECT * FROM pg_stat_activity;
SELECT * FROM pg_stat_database;
SELECT * FROM pg_stat_statements;

-- Maintenance
VACUUM ANALYZE table_name;
REINDEX INDEX CONCURRENTLY idx_name;

-- Backup
pg_dump -Fc mydb > backup.dump
pg_basebackup -D /backup -Ft -z -P

-- Performance
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

### Critical Queries

```sql
-- Cache hit ratio (should be >99%)
SELECT round(100.0 * sum(blks_hit) / NULLIF(sum(blks_hit + blks_read), 0), 2)
FROM pg_stat_database;

-- Bloated tables
SELECT tablename, n_dead_tup, n_live_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000;

-- Unused indexes
SELECT indexname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE idx_scan = 0;

-- Long-running queries
SELECT pid, now() - query_start as duration, query
FROM pg_stat_activity
WHERE state = 'active' AND now() - query_start > interval '5 minutes';
```

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add new exercise'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

## Resources

### Official Documentation

- [PostgreSQL 17 Documentation](https://www.postgresql.org/docs/17/)
- [PostgreSQL Wiki](https://wiki.postgresql.org/)
- [PostgreSQL Release Notes](https://www.postgresql.org/docs/release/)

### Tools

- [pgAdmin](https://www.pgadmin.org/) - Web-based administration
- [PgBouncer](https://www.pgbouncer.org/) - Connection pooling
- [pgBackRest](https://pgbackrest.org/) - Backup and restore
- [Patroni](https://github.com/zalando/patroni) - High availability

### Extensions

- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [PostGIS](https://postgis.net/) - Geospatial database
- [TimescaleDB](https://www.timescale.com/) - Time-series data
- [pg_trgm](https://www.postgresql.org/docs/current/pgtrgm.html) - Fuzzy text search

## License

This tutorial is provided as-is for educational purposes.

## Support

- **Issues:** Report bugs or request features via [GitHub Issues](../../issues)
- **Discussions:** Ask questions in [GitHub Discussions](../../discussions)

---

## Progress Tracking

- [ ] Complete Section 1: Fundamentals
- [ ] Complete Section 2: Performance Foundations
- [ ] Complete Section 3: Maintenance & Monitoring
- [ ] Complete Section 4: Advanced Topics
- [ ] Complete all exercises
- [ ] Set up Docker environment
- [ ] Practice with sample databases
- [ ] Build monitoring dashboard
- [ ] Implement automated backups
- [ ] Configure high availability

**Start your PostgreSQL DBA journey today!**

[Begin with Module 01 →](./01-installation-configuration)
