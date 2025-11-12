# 16 - Security and Permissions

## Authentication Configuration

### pg_hba.conf

```ini
# TYPE  DATABASE   USER       ADDRESS          METHOD

# Local connections
local   all        postgres                    peer
local   all        all                         scram-sha-256

# TCP/IP connections
host    all        all        127.0.0.1/32     scram-sha-256
host    all        all        ::1/128          scram-sha-256

# SSL connections only
hostssl all        all        0.0.0.0/0        scram-sha-256

# Specific user from specific IP
host    mydb       app_user   10.0.1.50/32     scram-sha-256

# Replication
host    replication replicator 10.0.1.0/24     scram-sha-256
```

## SSL/TLS Configuration

### Enable SSL

```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/path/to/server.crt'
ssl_key_file = '/path/to/server.key'
ssl_ca_file = '/path/to/root.crt'
```

### Generate Self-Signed Certificate

```bash
# Generate private key
openssl genrsa -out server.key 2048
chmod 600 server.key
chown postgres:postgres server.key

# Generate certificate
openssl req -new -x509 -key server.key -out server.crt -days 365

# Restart PostgreSQL
sudo systemctl restart postgresql
```

### Require SSL

```ini
# pg_hba.conf
hostssl all all 0.0.0.0/0 scram-sha-256

# Client connection
psql "host=localhost dbname=mydb sslmode=require"
```

### SSL Modes

| Mode | Security | Use Case |
|------|----------|----------|
| `disable` | None | Local development |
| `allow` | Opportunistic | Legacy |
| `prefer` | Opportunistic (default) | General |
| `require` | Encrypted | Production |
| `verify-ca` | Verify server cert | High security |
| `verify-full` | Verify server + hostname | Maximum security |

## Row Level Security (RLS)

### Enable RLS

```sql
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    user_id INT,
    title TEXT,
    content TEXT
);

-- Enable RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Policy: Users see only their own documents
CREATE POLICY user_documents ON documents
    FOR ALL
    TO PUBLIC
    USING (user_id = current_setting('app.user_id')::INT);

-- Set user context
SET app.user_id = 100;

-- Query only returns documents where user_id = 100
SELECT * FROM documents;
```

### RLS Policies

```sql
-- SELECT policy
CREATE POLICY select_own ON documents
    FOR SELECT
    TO PUBLIC
    USING (user_id = current_user::INT);

-- INSERT policy
CREATE POLICY insert_own ON documents
    FOR INSERT
    TO PUBLIC
    WITH CHECK (user_id = current_user::INT);

-- UPDATE policy
CREATE POLICY update_own ON documents
    FOR UPDATE
    TO PUBLIC
    USING (user_id = current_user::INT)
    WITH CHECK (user_id = current_user::INT);

-- DELETE policy
CREATE POLICY delete_own ON documents
    FOR DELETE
    TO PUBLIC
    USING (user_id = current_user::INT);

-- Admin policy (see all)
CREATE POLICY admin_all ON documents
    TO admin_role
    USING (true);
```

## Role-Based Access Control

### Create Roles

```sql
-- Group roles
CREATE ROLE readonly NOLOGIN;
CREATE ROLE readwrite NOLOGIN;
CREATE ROLE admin NOLOGIN;

-- Grant privileges to group roles
GRANT CONNECT ON DATABASE mydb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO readonly;

GRANT readonly TO readwrite;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT INSERT, UPDATE, DELETE ON TABLES TO readwrite;

-- User roles
CREATE ROLE alice WITH LOGIN PASSWORD 'secure_pwd';
CREATE ROLE bob WITH LOGIN PASSWORD 'secure_pwd';

GRANT readonly TO alice;
GRANT readwrite TO bob;
```

## Encryption

### Data at Rest

```bash
# Encrypt entire filesystem
# Use LUKS, dm-crypt, or cloud provider encryption

# Encrypt backup files
pg_dump mydb | gpg --encrypt --recipient admin@example.com > backup.sql.gpg

# Decrypt
gpg --decrypt backup.sql.gpg | psql mydb
```

### Data in Transit

```sql
-- Force SSL
ALTER SYSTEM SET ssl = on;
```

### Column-Level Encryption (pgcrypto)

```sql
CREATE EXTENSION pgcrypto;

-- Encrypt data
INSERT INTO users (email, ssn)
VALUES ('user@example.com', pgp_sym_encrypt('123-45-6789', 'secret_key'));

-- Decrypt data
SELECT email, pgp_sym_decrypt(ssn::bytea, 'secret_key') as ssn
FROM users;
```

## Password Policies

```sql
-- Set password
ALTER USER alice WITH PASSWORD 'strong_password_123!';

-- Password expiration
ALTER USER alice VALID UNTIL '2025-12-31';

-- Require password change
ALTER USER alice PASSWORD 'temp_password' VALID UNTIL 'tomorrow';

-- No password expiration
ALTER USER alice VALID UNTIL 'infinity';
```

## Audit Logging

### Enable Query Logging

```ini
# postgresql.conf
log_statement = 'all'              # none, ddl, mod, all
log_duration = on
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
```

### pgAudit Extension

```sql
CREATE EXTENSION pgaudit;

-- Audit all DDL
ALTER SYSTEM SET pgaudit.log = 'ddl';

-- Audit specific operations
ALTER SYSTEM SET pgaudit.log = 'read, write';

-- Audit specific role
ALTER SYSTEM SET pgaudit.role = 'auditor';

SELECT pg_reload_conf();
```

## Security Best Practices

```sql
-- 1. Use scram-sha-256 authentication
-- pg_hba.conf: scram-sha-256

-- 2. Least privilege principle
GRANT SELECT ON users TO app_readonly;
-- Not: GRANT ALL

-- 3. Separate admin from application users
CREATE ROLE app_user WITH LOGIN;
GRANT readwrite TO app_user;
-- Don't use postgres superuser

-- 4. Enable SSL
ALTER SYSTEM SET ssl = on;

-- 5. Row Level Security for multi-tenant
ALTER TABLE data ENABLE ROW LEVEL SECURITY;

-- 6. Regular password rotation
ALTER USER alice PASSWORD 'new_password' VALID UNTIL '2025-12-31';

-- 7. Audit critical tables
CREATE TRIGGER audit_users AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION log_audit();

-- 8. Network restrictions in pg_hba.conf
host all all 10.0.1.0/24 scram-sha-256  -- Only internal network
```

## Security Checklist

- [ ] scram-sha-256 authentication
- [ ] SSL/TLS enabled
- [ ] Firewall configured
- [ ] Least privilege roles
- [ ] RLS for multi-tenant data
- [ ] pgaudit enabled
- [ ] Regular password rotation
- [ ] Backup encryption
- [ ] No superuser for applications
- [ ] pg_hba.conf restricted

---

**Next:** [17 - High Availability →](../17-high-availability/README.md)
