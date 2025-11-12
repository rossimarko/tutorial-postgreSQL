# 19 - Migrations and Deployments

## Schema Migrations

### Manual Migrations

```sql
-- migrations/001_create_users.sql
BEGIN;

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_users_email ON users(email);

-- Migration metadata
CREATE TABLE IF NOT EXISTS schema_migrations (
    version INT PRIMARY KEY,
    applied_at TIMESTAMP DEFAULT now()
);

INSERT INTO schema_migrations (version) VALUES (1);

COMMIT;
```

### Flyway

```bash
# Install
wget https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.0.0/flyway-commandline-9.0.0-linux-x64.tar.gz
tar -xzf flyway-commandline-9.0.0-linux-x64.tar.gz

# Configure flyway.conf
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=postgres
flyway.password=password
flyway.locations=filesystem:./sql

# Migrations in sql/ directory
# V1__Create_users_table.sql
# V2__Add_orders_table.sql

# Run migrations
./flyway migrate

# Check status
./flyway info
```

### Liquibase

```xml
<!-- db-changelog.xml -->
<databaseChangeLog>
    <changeSet id="1" author="admin">
        <createTable tableName="users">
            <column name="id" type="SERIAL">
                <constraints primaryKey="true"/>
            </column>
            <column name="email" type="VARCHAR(255)">
                <constraints nullable="false" unique="true"/>
            </column>
        </createTable>
    </changeSet>
</databaseChangeLog>
```

```bash
# Run migrations
liquibase update \
    --url=jdbc:postgresql://localhost:5432/mydb \
    --username=postgres \
    --password=password \
    --changelog-file=db-changelog.xml
```

## Zero-Downtime Migrations

### Adding Columns

```sql
-- Step 1: Add column with default (fast in PostgreSQL 11+)
ALTER TABLE users ADD COLUMN status TEXT DEFAULT 'active';

-- Step 2: Backfill if needed (do in batches)
UPDATE users SET status = 'inactive'
WHERE last_login < now() - interval '1 year'
  AND id BETWEEN 1 AND 10000;

-- Step 3: Add constraint
ALTER TABLE users ADD CONSTRAINT check_status
    CHECK (status IN ('active', 'inactive', 'deleted'));
```

### Dropping Columns

```sql
-- Step 1: Deploy code ignoring column

-- Step 2: Drop column (after code deployment)
ALTER TABLE users DROP COLUMN old_column;
```

### Renaming Columns

```sql
-- Avoid renaming! Instead:

-- Step 1: Add new column
ALTER TABLE users ADD COLUMN new_name TEXT;

-- Step 2: Backfill
UPDATE users SET new_name = old_name;

-- Step 3: Deploy code using new_name

-- Step 4: Drop old_name
ALTER TABLE users DROP COLUMN old_name;
```

### Adding Indexes Concurrently

```sql
-- Blocking (locks table):
CREATE INDEX idx_users_email ON users(email);

-- Non-blocking (preferred):
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);

-- If fails, drop and retry:
DROP INDEX CONCURRENTLY IF EXISTS idx_users_email;
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

## Blue-Green Deployment

### Setup

```
┌─────────────┐         ┌─────────────┐
│   Blue DB   │ ◄─────  │  Load       │
│  (Active)   │         │  Balancer   │
└─────────────┘         └─────────────┘
                              │
                              │
┌─────────────┐               │
│  Green DB   │ ◄─────────────┘
│  (Standby)  │
└─────────────┘
```

### Process

```bash
# 1. Setup streaming replication (Blue → Green)
# 2. Apply migrations to Green (standby)
# 3. Monitor replication lag
# 4. Switch traffic to Green
# 5. Promote Green to primary
# 6. Blue becomes new standby
```

### Implementation

```sql
-- On Green (standby), promote to primary
SELECT pg_promote();

-- Update DNS or load balancer to point to Green

-- Configure Blue as new standby
-- pg_basebackup from Green to Blue
```

## Rolling Deployments

```sql
-- Compatible migrations only
-- Example: Adding optional column

-- Version N
SELECT id, email FROM users;

-- Deploy migration (add column)
ALTER TABLE users ADD COLUMN phone TEXT;

-- Version N and N+1 both work
-- Version N: SELECT id, email FROM users;
-- Version N+1: SELECT id, email, phone FROM users;
```

## Database Versioning

### Track Schema Version

```sql
CREATE TABLE schema_version (
    version INT PRIMARY KEY,
    description TEXT,
    applied_at TIMESTAMP DEFAULT now()
);

-- After each migration
INSERT INTO schema_version (version, description)
VALUES (1, 'Create users table');
```

### Git-Based Workflow

```bash
# Store migrations in git
migrations/
├── 001_create_users.sql
├── 002_add_orders.sql
└── 003_add_indexes.sql

# Tag releases
git tag -a v1.0.0 -m "Release 1.0.0"

# Deploy specific version
git checkout v1.0.0
./deploy.sh
```

## Rollback Strategies

### Safe Rollbacks

```sql
-- Down migration: 002_add_orders_down.sql
BEGIN;

DROP TABLE orders;

DELETE FROM schema_migrations WHERE version = 2;

COMMIT;
```

### Rollback-Safe Migrations

```sql
-- Add column with default (rollback: drop column)
ALTER TABLE users ADD COLUMN status TEXT DEFAULT 'active';

-- Add index concurrently (rollback: drop index concurrently)
CREATE INDEX CONCURRENTLY idx_users_status ON users(status);

-- Add constraint (rollback: drop constraint)
ALTER TABLE users ADD CONSTRAINT check_status
    CHECK (status IN ('active', 'inactive'));
```

## Testing Migrations

```bash
#!/bin/bash
# test_migration.sh

# Create test database
createdb test_migration

# Restore production backup
pg_restore -d test_migration production_backup.dump

# Apply migration
psql test_migration < migrations/003_add_column.sql

# Run tests
psql test_migration -c "SELECT count(*) FROM users;"

# Cleanup
dropdb test_migration
```

## CI/CD Integration

### GitHub Actions Example

```yaml
# .github/workflows/db-migration.yml
name: Database Migration

on:
  push:
    branches: [main]

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Install PostgreSQL client
        run: sudo apt-get install postgresql-client

      - name: Run migrations
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          for migration in migrations/*.sql; do
            psql $DATABASE_URL -f $migration
          done

      - name: Verify
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          psql $DATABASE_URL -c "SELECT * FROM schema_version;"
```

## Best Practices

1. **Always use transactions**
   ```sql
   BEGIN;
   ALTER TABLE users ADD COLUMN status TEXT;
   INSERT INTO schema_migrations (version) VALUES (5);
   COMMIT;
   ```

2. **Test migrations on copy of production**
   ```bash
   pg_dump production | psql test
   psql test < migration.sql
   ```

3. **Make migrations reversible**
   ```sql
   -- up.sql
   ALTER TABLE users ADD COLUMN phone TEXT;

   -- down.sql
   ALTER TABLE users DROP COLUMN phone;
   ```

4. **Use CONCURRENTLY for indexes**
   ```sql
   CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
   ```

5. **Avoid ALTER TYPE on large tables**
   ```sql
   -- Slow:
   ALTER TABLE users ALTER COLUMN id TYPE BIGINT;

   -- Fast: Create new column, migrate, swap
   ALTER TABLE users ADD COLUMN id_new BIGINT;
   UPDATE users SET id_new = id;
   ALTER TABLE users DROP COLUMN id;
   ALTER TABLE users RENAME COLUMN id_new TO id;
   ```

6. **Monitor migration progress**
   ```sql
   SELECT pid, now() - query_start as duration, query
   FROM pg_stat_activity
   WHERE query LIKE 'ALTER TABLE%';
   ```

---

**Complete!** You've finished all 19 modules.

**Next:** [Exercises →](../exercises/README.md) | [Back to Main](../README.md)
