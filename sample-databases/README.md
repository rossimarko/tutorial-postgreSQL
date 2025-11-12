# Sample Databases

Pre-configured sample databases for practicing PostgreSQL DBA concepts.

## Setup

```bash
# Run all setup scripts
./setup_all.sh

# Or individual setups
psql -U postgres -f ecommerce_setup.sql
psql -U postgres -f analytics_setup.sql
psql -U postgres -f timeseries_setup.sql
```

## 1. E-Commerce Database

### Schema
```sql
-- ecommerce_setup.sql
CREATE DATABASE ecommerce;
\c ecommerce

-- Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    status TEXT DEFAULT 'active',
    created_at TIMESTAMP DEFAULT now(),
    last_login TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status ON users(status);

-- Products table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    description TEXT,
    category TEXT NOT NULL,
    price NUMERIC(10,2) NOT NULL,
    stock INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_price ON products(price);

-- Orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    status TEXT DEFAULT 'pending',
    total NUMERIC(10,2),
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now()
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at);

-- Order items
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(id) ON DELETE CASCADE,
    product_id INT REFERENCES products(id),
    quantity INT NOT NULL,
    price NUMERIC(10,2) NOT NULL
);

CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Generate sample data
-- 100,000 users
INSERT INTO users (email, username, password_hash, status, created_at, last_login)
SELECT
    'user' || g || '@example.com',
    'user' || g,
    md5(random()::text),
    CASE (random() * 10)::INT
        WHEN 0 THEN 'inactive'
        WHEN 1 THEN 'deleted'
        ELSE 'active'
    END,
    '2020-01-01'::TIMESTAMP + (random() * (now() - '2020-01-01'::TIMESTAMP)),
    CASE WHEN random() > 0.3 THEN
        '2024-01-01'::TIMESTAMP + (random() * (now() - '2024-01-01'::TIMESTAMP))
    ELSE NULL END
FROM generate_series(1, 100000) g;

-- 10,000 products
INSERT INTO products (name, category, price, stock, created_at)
SELECT
    'Product ' || g,
    CASE (random() * 5)::INT
        WHEN 0 THEN 'Electronics'
        WHEN 1 THEN 'Clothing'
        WHEN 2 THEN 'Books'
        WHEN 3 THEN 'Home & Garden'
        ELSE 'Sports'
    END,
    (random() * 1000 + 10)::NUMERIC(10,2),
    (random() * 1000)::INT,
    '2020-01-01'::TIMESTAMP + (random() * (now() - '2020-01-01'::TIMESTAMP))
FROM generate_series(1, 10000) g;

-- 500,000 orders
INSERT INTO orders (user_id, status, total, created_at)
SELECT
    (random() * 99999 + 1)::INT,
    CASE (random() * 10)::INT
        WHEN 0..5 THEN 'completed'
        WHEN 6..7 THEN 'shipped'
        WHEN 8 THEN 'pending'
        ELSE 'cancelled'
    END,
    (random() * 1000 + 10)::NUMERIC(10,2),
    '2023-01-01'::TIMESTAMP + (random() * (now() - '2023-01-01'::TIMESTAMP))
FROM generate_series(1, 500000) g;

-- 1.5M order items
INSERT INTO order_items (order_id, product_id, quantity, price)
SELECT
    (random() * 499999 + 1)::INT,
    (random() * 9999 + 1)::INT,
    (random() * 5 + 1)::INT,
    (random() * 500 + 10)::NUMERIC(10,2)
FROM generate_series(1, 1500000) g;

-- Update order totals
UPDATE orders o
SET total = (
    SELECT COALESCE(sum(quantity * price), 0)
    FROM order_items
    WHERE order_id = o.id
);

-- Analyze tables
ANALYZE;
```

### Practice Queries

```sql
-- 1. Top selling products
SELECT p.name, count(oi.id) as order_count, sum(oi.quantity) as total_quantity
FROM products p
JOIN order_items oi ON p.id = oi.product_id
GROUP BY p.id, p.name
ORDER BY total_quantity DESC
LIMIT 20;

-- 2. Monthly sales
SELECT
    date_trunc('month', created_at) as month,
    count(*) as orders,
    sum(total) as revenue
FROM orders
WHERE status = 'completed'
GROUP BY month
ORDER BY month DESC;

-- 3. Customer lifetime value
SELECT u.username,
       count(o.id) as order_count,
       sum(o.total) as lifetime_value
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.status = 'completed'
GROUP BY u.id, u.username
ORDER BY lifetime_value DESC
LIMIT 100;
```

## 2. Analytics Database

```sql
-- analytics_setup.sql
CREATE DATABASE analytics;
\c analytics

-- Page views (partitioned by date)
CREATE TABLE page_views (
    id BIGSERIAL,
    user_id INT,
    page_url TEXT,
    session_id UUID,
    referrer TEXT,
    user_agent TEXT,
    ip_address INET,
    created_at TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions for each month
CREATE TABLE page_views_2024_01 PARTITION OF page_views
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE page_views_2024_02 PARTITION OF page_views
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Generate 10M page views
INSERT INTO page_views (user_id, page_url, session_id, referrer, ip_address, created_at)
SELECT
    (random() * 100000)::INT,
    '/page' || (random() * 1000)::INT,
    uuid_generate_v4(),
    CASE (random() * 3)::INT
        WHEN 0 THEN 'https://google.com'
        WHEN 1 THEN 'https://facebook.com'
        ELSE 'https://twitter.com'
    END,
    ('192.168.' || (random() * 255)::INT || '.' || (random() * 255)::INT)::INET,
    '2024-01-01'::TIMESTAMP + (random() * interval '60 days')
FROM generate_series(1, 10000000) g;

CREATE INDEX idx_page_views_user_id ON page_views(user_id);
CREATE INDEX idx_page_views_created ON page_views(created_at);

ANALYZE;
```

## 3. Time-Series Database

```sql
-- timeseries_setup.sql
CREATE DATABASE timeseries;
\c timeseries

CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Sensor data
CREATE TABLE sensor_metrics (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INT NOT NULL,
    temperature NUMERIC(5,2),
    humidity NUMERIC(5,2),
    pressure NUMERIC(6,2)
);

-- Convert to hypertable
SELECT create_hypertable('sensor_metrics', 'time');

-- Generate 50M sensor readings
INSERT INTO sensor_metrics (time, sensor_id, temperature, humidity, pressure)
SELECT
    '2024-01-01'::TIMESTAMPTZ + (g * interval '10 seconds'),
    (random() * 100)::INT,
    (random() * 40 - 10)::NUMERIC(5,2),
    (random() * 100)::NUMERIC(5,2),
    (random() * 200 + 900)::NUMERIC(6,2)
FROM generate_series(1, 50000000) g;

-- Continuous aggregate
CREATE MATERIALIZED VIEW sensor_metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) as bucket,
    sensor_id,
    avg(temperature) as avg_temp,
    max(temperature) as max_temp,
    min(temperature) as min_temp,
    avg(humidity) as avg_humidity,
    avg(pressure) as avg_pressure
FROM sensor_metrics
GROUP BY bucket, sensor_id;

ANALYZE;
```

## Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:17-alpine
    container_name: postgres-dba-tutorial
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_USER: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./sample-databases:/docker-entrypoint-initdb.d
    command:
      - "postgres"
      - "-c"
      - "shared_buffers=256MB"
      - "-c"
      - "max_connections=200"

  pgadmin:
    image: dpage/pgadmin4:latest
    container_name: pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@example.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"
    depends_on:
      - postgres

volumes:
  pgdata:
```

## Start Environment

```bash
# Start all services
docker-compose up -d

# Connect to PostgreSQL
psql -h localhost -U postgres

# Access pgAdmin
# Open browser: http://localhost:5050
# Email: admin@example.com
# Password: admin
```

---

**Next:** [Docker Setup →](../docker/) | [Back to Main](../README.md)
