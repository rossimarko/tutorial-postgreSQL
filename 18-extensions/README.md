# 18 - PostgreSQL Extensions

## Essential Extensions

### pg_stat_statements

```sql
-- Enable in postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all

-- Restart PostgreSQL, then:
CREATE EXTENSION pg_stat_statements;

-- Top slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### pg_trgm (Trigram Similarity)

```sql
CREATE EXTENSION pg_trgm;

-- Fuzzy text search
CREATE INDEX idx_users_name_trgm ON users USING gin(name gin_trgm_ops);

-- Similar names
SELECT name FROM users WHERE name % 'John';

-- LIKE optimization
SELECT * FROM users WHERE name ILIKE '%smith%';
-- Uses idx_users_name_trgm
```

### PostGIS (Geospatial)

```sql
CREATE EXTENSION postgis;

-- Create table with geometry
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    geom GEOMETRY(Point, 4326)
);

-- Create spatial index
CREATE INDEX idx_locations_geom ON locations USING gist(geom);

-- Insert point
INSERT INTO locations (name, geom)
VALUES ('Office', ST_SetSRID(ST_MakePoint(-122.4194, 37.7749), 4326));

-- Find nearby points
SELECT name,
       ST_Distance(geom, ST_SetSRID(ST_MakePoint(-122.4, 37.7), 4326)) as distance
FROM locations
WHERE ST_DWithin(geom, ST_SetSRID(ST_MakePoint(-122.4, 37.7), 4326), 0.1)
ORDER BY distance;
```

### TimescaleDB (Time-Series)

```sql
CREATE EXTENSION timescaledb;

-- Create hypertable
CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    device_id INT,
    temperature NUMERIC,
    humidity NUMERIC
);

SELECT create_hypertable('metrics', 'time');

-- Automatic partitioning by time
-- Continuous aggregates
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) as bucket,
       device_id,
       avg(temperature) as avg_temp,
       avg(humidity) as avg_humidity
FROM metrics
GROUP BY bucket, device_id;
```

### uuid-ossp (UUID Generation)

```sql
CREATE EXTENSION "uuid-ossp";

-- Generate UUIDs
SELECT uuid_generate_v4();

-- Use as primary key
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    email TEXT
);
```

### pgcrypto (Encryption)

```sql
CREATE EXTENSION pgcrypto;

-- Hash password
INSERT INTO users (email, password_hash)
VALUES ('user@example.com', crypt('my_password', gen_salt('bf')));

-- Verify password
SELECT * FROM users
WHERE email = 'user@example.com'
  AND password_hash = crypt('my_password', password_hash);

-- Encrypt data
INSERT INTO sensitive_data (ssn)
VALUES (pgp_sym_encrypt('123-45-6789', 'encryption_key'));

-- Decrypt data
SELECT pgp_sym_decrypt(ssn::bytea, 'encryption_key') as ssn
FROM sensitive_data;
```

### hstore (Key-Value Store)

```sql
CREATE EXTENSION hstore;

-- Create table with hstore column
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    attributes hstore
);

-- Insert data
INSERT INTO products (name, attributes)
VALUES ('Laptop', 'brand=>Dell, ram=>16GB, cpu=>i7');

-- Query hstore
SELECT name, attributes->'brand' as brand
FROM products
WHERE attributes->'ram' = '16GB';

-- Create GIN index
CREATE INDEX idx_products_attributes ON products USING gin(attributes);
```

### pg_repack (Online Table Reorganization)

```bash
# Install
sudo apt install postgresql-17-repack

# Usage
pg_repack -d mydb -t users

# Remove bloat without locking table
```

### pgaudit (Audit Logging)

```sql
CREATE EXTENSION pgaudit;

-- Configure
ALTER SYSTEM SET pgaudit.log = 'read, write, ddl';
SELECT pg_reload_conf();

-- Audit logs written to PostgreSQL log file
```

### Foreign Data Wrappers

```sql
-- PostgreSQL FDW
CREATE EXTENSION postgres_fdw;

-- Connect to remote PostgreSQL
CREATE SERVER remote_db
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'remote-host', dbname 'remotedb', port '5432');

CREATE USER MAPPING FOR postgres
    SERVER remote_db
    OPTIONS (user 'postgres', password 'pwd');

-- Import schema
IMPORT FOREIGN SCHEMA public
    FROM SERVER remote_db
    INTO public;

-- Query remote tables
SELECT * FROM remote_table;
```

## Extension Management

```sql
-- List available extensions
SELECT * FROM pg_available_extensions;

-- List installed extensions
\dx
-- or
SELECT * FROM pg_extension;

-- Create extension
CREATE EXTENSION extension_name;

-- Drop extension
DROP EXTENSION extension_name CASCADE;

-- Update extension
ALTER EXTENSION extension_name UPDATE TO 'version';

-- Extension information
\dx+ extension_name
```

## Creating Custom Extensions

```sql
-- Simple extension example
CREATE EXTENSION plpgsql;  -- Ensure plpgsql is available

-- Create function
CREATE OR REPLACE FUNCTION my_function(x INT)
RETURNS INT AS $$
BEGIN
    RETURN x * 2;
END;
$$ LANGUAGE plpgsql;

-- Package as extension (requires filesystem access)
-- Create files:
-- my_extension--1.0.sql
-- my_extension.control

-- Install
CREATE EXTENSION my_extension;
```

---

**Next:** [19 - Migrations and Deployments →](../19-migrations-deployments/README.md)
