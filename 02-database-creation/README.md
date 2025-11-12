# 02 - Database Creation

## Table of Contents
- [Database Creation](#database-creation)
- [Tablespaces](#tablespaces)
- [Schemas](#schemas)
- [Roles and Permissions](#roles-and-permissions)
- [Anti-Patterns](#anti-patterns)
- [Monitoring](#monitoring)
- [Real-World Scenarios](#real-world-scenarios)

## Database Creation

### Basic Database Creation

```sql
-- Simple database creation
CREATE DATABASE mydb;

-- With all options
CREATE DATABASE production_db
    WITH
    OWNER = app_owner
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8'
    TABLESPACE = pg_default
    TEMPLATE = template0
    CONNECTION LIMIT = 100
    IS_TEMPLATE = false;
```

### Template Databases

PostgreSQL uses template databases for creating new databases:

| Template | Purpose | Use Case |
|----------|---------|----------|
| `template0` | Clean slate | When you need specific encoding/collation |
| `template1` | Default template | General purpose, can be customized |
| Custom templates | Pre-configured DBs | Multi-tenant applications |

```sql
-- Create from clean template (for different encoding)
CREATE DATABASE utf8_db
    WITH
    TEMPLATE = template0
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.UTF-8';

-- Customize template1 for all new databases
\c template1
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE EXTENSION IF NOT EXISTS btree_gin;

-- New databases will now include these extensions
CREATE DATABASE mydb;  -- Automatically has extensions from template1
```

### Database Properties

```sql
-- List all databases
\l+

-- or SQL query
SELECT datname,
       pg_size_pretty(pg_database_size(datname)) as size,
       datcollate,
       datctype,
       encoding,
       datconnlimit
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- Connect to database
\c database_name

-- Show current database
SELECT current_database();
```

### Dropping Databases

```sql
-- Terminate all connections to database
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE pg_stat_activity.datname = 'target_db'
  AND pid <> pg_backend_pid();

-- Drop database
DROP DATABASE target_db;

-- Drop if exists (PostgreSQL 13+)
DROP DATABASE IF EXISTS target_db WITH (FORCE);
```

## Tablespaces

### Concept

Tablespaces allow you to define locations where database objects are stored. Useful for:
- Separating data across different disks
- Placing frequently accessed data on faster storage (SSD)
- Managing disk space constraints

### Architecture
```
┌─────────────────────────────────────┐
│         PostgreSQL Cluster          │
├─────────────────────────────────────┤
│  Tablespace: pg_default             │
│  Location: /var/lib/postgresql/data │
│    ├─ Database: postgres            │
│    └─ Database: mydb                │
├─────────────────────────────────────┤
│  Tablespace: fast_storage           │
│  Location: /mnt/ssd/pgdata          │
│    └─ Table: hot_data               │
├─────────────────────────────────────┤
│  Tablespace: archive_storage        │
│  Location: /mnt/hdd/pgarchive       │
│    └─ Table: historical_logs        │
└─────────────────────────────────────┘
```

### Creating Tablespaces

```sql
-- Create directory (must be owned by postgres user)
-- Run as root or postgres OS user:
-- sudo mkdir -p /mnt/ssd/pgdata
-- sudo chown postgres:postgres /mnt/ssd/pgdata

-- Create tablespace
CREATE TABLESPACE fast_storage
    LOCATION '/mnt/ssd/pgdata';

CREATE TABLESPACE archive_storage
    LOCATION '/mnt/hdd/pgarchive';

-- List tablespaces
\db+

-- Set default tablespace for database
ALTER DATABASE mydb SET default_tablespace = fast_storage;
```

### Using Tablespaces

```sql
-- Create table in specific tablespace
CREATE TABLE hot_data (
    id SERIAL PRIMARY KEY,
    data TEXT,
    created_at TIMESTAMP DEFAULT now()
) TABLESPACE fast_storage;

-- Create index in specific tablespace
CREATE INDEX idx_hot_data_created
    ON hot_data(created_at)
    TABLESPACE fast_storage;

-- Move existing table to different tablespace
ALTER TABLE historical_logs SET TABLESPACE archive_storage;

-- Move all tables in a tablespace
ALTER TABLESPACE fast_storage MOVE ALL TO archive_storage;
```

### Tablespace Monitoring

```sql
-- Tablespace sizes
SELECT spcname,
       pg_size_pretty(pg_tablespace_size(spcname)) as size
FROM pg_tablespace;

-- Objects per tablespace
SELECT t.spcname,
       count(*) as objects,
       pg_size_pretty(sum(pg_total_relation_size(c.oid))) as total_size
FROM pg_class c
JOIN pg_tablespace t ON c.reltablespace = t.oid
GROUP BY t.spcname;

-- Which tables are in which tablespace
SELECT schemaname,
       tablename,
       COALESCE(t.spcname, 'pg_default') as tablespace,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
LEFT JOIN pg_tablespace t ON pg_tables.tablespace = t.spcname
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

## Schemas

### Concept

Schemas are namespaces within a database. They provide:
- Logical organization of objects
- Multi-tenant isolation
- Permission boundaries
- Name collision avoidance

### Schema Hierarchy
```
Database: mycompany
├─ Schema: public (default)
│  ├─ Table: users
│  └─ Table: products
├─ Schema: hr
│  ├─ Table: employees
│  └─ Table: salaries
├─ Schema: finance
│  ├─ Table: transactions
│  └─ Table: ledger
└─ Schema: tenant_123
   ├─ Table: users
   └─ Table: orders
```

### Creating and Using Schemas

```sql
-- Create schema
CREATE SCHEMA hr;
CREATE SCHEMA finance AUTHORIZATION finance_team;

-- Create schema if not exists
CREATE SCHEMA IF NOT EXISTS analytics;

-- Create objects in schema
CREATE TABLE hr.employees (
    employee_id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    salary NUMERIC(10,2)
);

CREATE TABLE finance.transactions (
    transaction_id SERIAL PRIMARY KEY,
    amount NUMERIC(15,2),
    date DATE DEFAULT CURRENT_DATE
);

-- List all schemas
\dn+

-- Show current schema
SHOW search_path;
-- Result: "$user", public

-- Set search path (temporary)
SET search_path TO hr, finance, public;

-- Set search path (permanent for user)
ALTER USER myuser SET search_path = hr, finance, public;

-- Set search path (permanent for database)
ALTER DATABASE mydb SET search_path = hr, finance, public;
```

### Schema Search Path

The search path determines which schema is searched for unqualified object names:

```sql
-- Current search path
SELECT current_schemas(true);

-- Example with multiple schemas
SET search_path TO hr, finance, public;

-- This will search hr first, then finance, then public
SELECT * FROM employees;  -- Finds hr.employees

-- Explicitly specify schema
SELECT * FROM finance.transactions;
SELECT * FROM public.users;
```

### Schema Best Practices

```sql
-- Multi-tenant pattern
CREATE SCHEMA tenant_1;
CREATE SCHEMA tenant_2;
CREATE SCHEMA tenant_3;

-- Create same structure in each schema
CREATE TABLE tenant_1.orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    amount NUMERIC(10,2)
);

CREATE TABLE tenant_2.orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    amount NUMERIC(10,2)
);

-- Application sets search_path per tenant
SET search_path TO tenant_1, public;
SELECT * FROM orders;  -- Gets tenant_1.orders
```

### Dropping Schemas

```sql
-- Drop empty schema
DROP SCHEMA hr;

-- Drop schema with all objects
DROP SCHEMA hr CASCADE;

-- Drop if exists
DROP SCHEMA IF EXISTS hr CASCADE;
```

## Roles and Permissions

### Role Hierarchy

```
┌─────────────────────────────────────┐
│         superuser (postgres)         │
└───────────────┬─────────────────────┘
                │
        ┌───────┴────────┬──────────────┐
        │                │              │
┌───────▼─────┐  ┌──────▼──────┐  ┌───▼────────┐
│ db_admin    │  │ app_owner   │  │ monitoring │
└───────┬─────┘  └──────┬──────┘  └────────────┘
        │               │
    ┌───┴───┐      ┌────┴────┐
    │ hr_   │      │ app_    │
    │ admin │      │ readonly│
    └───────┘      └─────────┘
```

### Creating Roles

```sql
-- Create login role (user)
CREATE ROLE myuser WITH LOGIN PASSWORD 'secure_password';

-- Create role with specific attributes
CREATE ROLE app_admin WITH
    LOGIN
    PASSWORD 'strong_password'
    CREATEDB
    CREATEROLE
    CONNECTION LIMIT 10
    VALID UNTIL '2025-12-31';

-- Create group role (no login)
CREATE ROLE readonly;
CREATE ROLE readwrite;

-- List all roles
\du+

-- or SQL query
SELECT rolname,
       rolsuper,
       rolcreaterole,
       rolcreatedb,
       rolcanlogin,
       rolconnlimit,
       rolvaliduntil
FROM pg_roles
ORDER BY rolname;
```

### Role Attributes

| Attribute | Description | Example |
|-----------|-------------|---------|
| `LOGIN` | Can connect to database | `CREATE ROLE user1 WITH LOGIN` |
| `SUPERUSER` | Bypass all permission checks | `ALTER ROLE user1 SUPERUSER` |
| `CREATEDB` | Can create databases | `CREATE ROLE admin WITH CREATEDB` |
| `CREATEROLE` | Can create other roles | `CREATE ROLE admin WITH CREATEROLE` |
| `REPLICATION` | Can initiate replication | `CREATE ROLE replicator WITH REPLICATION` |
| `PASSWORD` | Set password | `CREATE ROLE user1 WITH PASSWORD 'pwd'` |
| `CONNECTION LIMIT` | Max concurrent connections | `ALTER ROLE user1 CONNECTION LIMIT 5` |

### Granting and Revoking Privileges

```sql
-- Grant database privileges
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT ALL PRIVILEGES ON DATABASE mydb TO app_owner;

-- Grant schema privileges
GRANT USAGE ON SCHEMA public TO app_user;
GRANT CREATE ON SCHEMA public TO app_owner;

-- Grant table privileges
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_owner;

-- Grant privileges on future tables (very important!)
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO readwrite;

-- Grant sequence privileges (for SERIAL columns)
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO readwrite;

-- Revoke privileges
REVOKE INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public FROM readonly;
```

### Role Membership (Group Roles)

```sql
-- Create group roles
CREATE ROLE readonly NOLOGIN;
CREATE ROLE readwrite NOLOGIN;
CREATE ROLE app_admin NOLOGIN;

-- Grant privileges to group roles
GRANT CONNECT ON DATABASE mydb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

GRANT readonly TO readwrite;  -- readwrite inherits readonly
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;

-- Grant roles to users
CREATE ROLE alice WITH LOGIN PASSWORD 'alice_pwd';
CREATE ROLE bob WITH LOGIN PASSWORD 'bob_pwd';

GRANT readonly TO alice;
GRANT readwrite TO bob;

-- Check role memberships
\du

-- or SQL query
SELECT r.rolname,
       array_agg(m.rolname) as member_of
FROM pg_roles r
LEFT JOIN pg_auth_members am ON r.oid = am.member
LEFT JOIN pg_roles m ON am.roleid = m.oid
WHERE r.rolcanlogin = true
GROUP BY r.rolname;
```

### Row Level Security (RLS)

```sql
-- Enable RLS on table
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title TEXT,
    owner TEXT,
    content TEXT
);

ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Create policy: users can only see their own documents
CREATE POLICY user_documents ON documents
    FOR ALL
    TO PUBLIC
    USING (owner = current_user);

-- Create policy: admins can see everything
CREATE POLICY admin_all ON documents
    FOR ALL
    TO app_admin
    USING (true);

-- Different policies for different operations
CREATE POLICY select_own ON documents
    FOR SELECT
    TO PUBLIC
    USING (owner = current_user);

CREATE POLICY insert_own ON documents
    FOR INSERT
    TO PUBLIC
    WITH CHECK (owner = current_user);

CREATE POLICY update_own ON documents
    FOR UPDATE
    TO PUBLIC
    USING (owner = current_user)
    WITH CHECK (owner = current_user);

-- View policies
\d+ documents

-- Test RLS
SET ROLE alice;
SELECT * FROM documents;  -- Only sees documents where owner='alice'

SET ROLE postgres;  -- Superuser bypasses RLS
SELECT * FROM documents;  -- Sees all documents
```

### Permission Queries

```sql
-- Check table privileges for a role
SELECT grantee,
       table_schema,
       table_name,
       privilege_type
FROM information_schema.table_privileges
WHERE grantee = 'app_user';

-- Check schema privileges
SELECT nspname,
       nspowner::regrole,
       nspacl
FROM pg_namespace
WHERE nspname NOT LIKE 'pg_%'
  AND nspname <> 'information_schema';

-- Check database privileges
SELECT datname,
       datacl
FROM pg_database
WHERE datname = current_database();

-- Human-readable privileges
SELECT grantee,
       privilege_type
FROM information_schema.role_table_grants
WHERE table_name = 'users';
```

## Anti-Patterns

### ❌ Anti-Pattern 1: Using public Schema for Everything
```sql
-- BAD: Everything in public schema
CREATE TABLE users (...);
CREATE TABLE products (...);
CREATE TABLE hr_employees (...);
CREATE TABLE finance_transactions (...);
```

**Impact:**
- No logical organization
- Difficult permission management
- Name collisions
- Poor multi-tenancy

**Solution:**
```sql
-- GOOD: Organize by domain
CREATE SCHEMA app;
CREATE SCHEMA hr;
CREATE SCHEMA finance;

CREATE TABLE app.users (...);
CREATE TABLE app.products (...);
CREATE TABLE hr.employees (...);
CREATE TABLE finance.transactions (...);
```

### ❌ Anti-Pattern 2: Granting Superuser Rights
```sql
-- BAD: Making application user superuser
ALTER ROLE app_user SUPERUSER;
```

**Impact:**
- Security risk
- Can bypass RLS and all permissions
- Can modify system catalogs
- No audit trail

**Solution:**
```sql
-- GOOD: Grant only necessary privileges
CREATE ROLE app_user WITH LOGIN PASSWORD 'secure_pwd';
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_user;
```

### ❌ Anti-Pattern 3: Not Using Default Privileges
```sql
-- BAD: Granting privileges on existing objects only
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- New tables created later won't have grants!
CREATE TABLE new_table (...);
-- readonly role can't access new_table
```

**Solution:**
```sql
-- GOOD: Set default privileges for future objects
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO readonly;
```

### ❌ Anti-Pattern 4: Shared User Accounts
```sql
-- BAD: All applications use same account
CREATE ROLE shared_app_user WITH LOGIN PASSWORD 'password';
```

**Impact:**
- No individual accountability
- Can't revoke access for specific application
- Difficult auditing
- Password sharing issues

**Solution:**
```sql
-- GOOD: Separate roles for each application/service
CREATE ROLE web_app WITH LOGIN PASSWORD 'web_pwd';
CREATE ROLE batch_processor WITH LOGIN PASSWORD 'batch_pwd';
CREATE ROLE analytics WITH LOGIN PASSWORD 'analytics_pwd';

-- Grant appropriate permissions to each
GRANT readwrite TO web_app;
GRANT readonly TO analytics;
```

## Monitoring

### Database Monitoring

```sql
-- Database sizes
SELECT datname,
       pg_size_pretty(pg_database_size(datname)) as size,
       numbackends as connections
FROM pg_stat_database
ORDER BY pg_database_size(datname) DESC;

-- Database activity
SELECT datname,
       numbackends as connections,
       xact_commit,
       xact_rollback,
       round(100.0 * xact_rollback / NULLIF(xact_commit + xact_rollback, 0), 2) as rollback_pct,
       blks_read,
       blks_hit,
       round(100.0 * blks_hit / NULLIF(blks_hit + blks_read, 0), 2) as cache_hit_ratio,
       tup_returned,
       tup_fetched,
       tup_inserted,
       tup_updated,
       tup_deleted
FROM pg_stat_database
WHERE datname = current_database();
```

### Schema Monitoring

```sql
-- Schema sizes
SELECT schemaname,
       count(*) as tables,
       pg_size_pretty(sum(pg_total_relation_size(schemaname||'.'||tablename))::bigint) as total_size
FROM pg_tables
GROUP BY schemaname
ORDER BY sum(pg_total_relation_size(schemaname||'.'||tablename)) DESC;

-- Largest objects per schema
SELECT schemaname,
       tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) -
                      pg_relation_size(schemaname||'.'||tablename)) as indexes_size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

### Role and Permission Monitoring

```sql
-- Active connections by role
SELECT usename,
       count(*),
       state
FROM pg_stat_activity
GROUP BY usename, state
ORDER BY count(*) DESC;

-- Role attributes
SELECT rolname,
       rolsuper,
       rolinherit,
       rolcreaterole,
       rolcreatedb,
       rolcanlogin,
       rolreplication,
       rolconnlimit,
       rolvaliduntil
FROM pg_roles
ORDER BY rolname;

-- Roles with superuser privileges (audit)
SELECT rolname, rolsuper, rolvaliduntil
FROM pg_roles
WHERE rolsuper = true;

-- Inactive roles (potential cleanup)
SELECT r.rolname,
       r.rolvaliduntil,
       max(s.backend_start) as last_connection
FROM pg_roles r
LEFT JOIN pg_stat_activity s ON r.rolname = s.usename
WHERE r.rolcanlogin = true
GROUP BY r.rolname, r.rolvaliduntil
ORDER BY max(s.backend_start) NULLS FIRST;
```

## Real-World Scenarios

### Scenario 1: Setting Up Multi-Tenant Database

**Requirement:** Isolate data for 1000+ tenants with identical schema structure.

**Solution:**
```sql
-- Create schema per tenant
DO $$
DECLARE
    tenant_id INT;
BEGIN
    FOR tenant_id IN 1..1000 LOOP
        EXECUTE format('CREATE SCHEMA tenant_%s', tenant_id);
    END LOOP;
END $$;

-- Function to create tenant tables
CREATE OR REPLACE FUNCTION setup_tenant_schema(tenant_schema TEXT)
RETURNS void AS $$
BEGIN
    EXECUTE format('CREATE TABLE %I.users (
        user_id SERIAL PRIMARY KEY,
        email TEXT UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT now()
    )', tenant_schema);

    EXECUTE format('CREATE TABLE %I.orders (
        order_id SERIAL PRIMARY KEY,
        user_id INT REFERENCES %I.users(user_id),
        amount NUMERIC(10,2),
        created_at TIMESTAMP DEFAULT now()
    )', tenant_schema, tenant_schema);
END;
$$ LANGUAGE plpgsql;

-- Apply to all tenants
DO $$
DECLARE
    tenant_id INT;
BEGIN
    FOR tenant_id IN 1..1000 LOOP
        PERFORM setup_tenant_schema('tenant_' || tenant_id);
    END LOOP;
END $$;

-- Application connection pattern
-- SET search_path TO tenant_123, public;
```

### Scenario 2: Migrating Data Between Tablespaces

**Requirement:** Move hot tables to SSD, archive tables to HDD.

```sql
-- Create tablespaces
CREATE TABLESPACE ssd_storage LOCATION '/mnt/ssd/pgdata';
CREATE TABLESPACE hdd_storage LOCATION '/mnt/hdd/pgdata';

-- Identify hot tables (frequently accessed)
SELECT schemaname,
       tablename,
       seq_scan + idx_scan as total_scans,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_stat_user_tables
ORDER BY seq_scan + idx_scan DESC
LIMIT 20;

-- Move hot tables to SSD
ALTER TABLE hot_table_1 SET TABLESPACE ssd_storage;
ALTER TABLE hot_table_2 SET TABLESPACE ssd_storage;

-- Move archive tables to HDD
ALTER TABLE logs_2020 SET TABLESPACE hdd_storage;
ALTER TABLE logs_2021 SET TABLESPACE hdd_storage;

-- Move indexes separately
ALTER INDEX idx_hot_table_1 SET TABLESPACE ssd_storage;
```

### Scenario 3: Implementing Least Privilege Access

**Requirement:** Developer needs read access, app needs read/write, admin needs all.

```sql
-- Create group roles
CREATE ROLE developers NOLOGIN;
CREATE ROLE applications NOLOGIN;
CREATE ROLE administrators NOLOGIN;

-- Grant privileges
-- Developers: read-only
GRANT CONNECT ON DATABASE mydb TO developers;
GRANT USAGE ON SCHEMA public TO developers;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO developers;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO developers;

-- Applications: read/write data
GRANT CONNECT ON DATABASE mydb TO applications;
GRANT USAGE ON SCHEMA public TO applications;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO applications;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO applications;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO applications;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO applications;

-- Administrators: DDL rights
GRANT CONNECT ON DATABASE mydb TO administrators;
GRANT ALL PRIVILEGES ON SCHEMA public TO administrators;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO administrators;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO administrators;

-- Create individual users
CREATE ROLE dev_alice WITH LOGIN PASSWORD 'alice_pwd';
CREATE ROLE app_service WITH LOGIN PASSWORD 'app_pwd';
CREATE ROLE admin_bob WITH LOGIN PASSWORD 'bob_pwd';

-- Assign to groups
GRANT developers TO dev_alice;
GRANT applications TO app_service;
GRANT administrators TO admin_bob;
```

### Scenario 4: Database Cloning for Testing

**Requirement:** Create exact copy of production database for testing.

```sql
-- Step 1: Terminate connections to source database
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE pg_stat_activity.datname = 'production_db'
  AND pid <> pg_backend_pid();

-- Step 2: Create database from template
CREATE DATABASE test_db WITH TEMPLATE production_db OWNER test_owner;

-- Step 3: Change all role passwords in test database
\c test_db
UPDATE pg_shadow SET passwd = 'test_password' WHERE usename = 'app_user';

-- Alternative: Use pg_dump for larger databases
-- pg_dump -U postgres production_db | psql -U postgres test_db
```

---

## Quick Reference

### Essential Commands
```sql
-- Database
CREATE DATABASE dbname;
DROP DATABASE dbname;
\l+                    -- List databases

-- Schema
CREATE SCHEMA schema_name;
DROP SCHEMA schema_name CASCADE;
\dn+                   -- List schemas

-- Roles
CREATE ROLE username WITH LOGIN PASSWORD 'pwd';
GRANT role_name TO username;
\du+                   -- List roles

-- Permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO role_name;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO role_name;
```

### Next Steps
- [← 01 - Installation and Configuration](../01-installation-configuration/README.md)
- [03 - WAL Architecture →](../03-wal-architecture/README.md)
- [Back to Main](../README.md)
