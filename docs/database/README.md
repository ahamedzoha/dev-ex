# Database Learning Path

> A comprehensive guide to mastering PostgreSQL and database concepts for senior fullstack engineers.

## Table of Contents

1. [Core Concepts](#core-concepts)
2. [Practical Exercises](#practical-exercises)
3. [Best Practices](#best-practices-and-guidelines)
4. [Common Problems](#common-problems-and-solutions)
5. [Resources](#resources-and-further-reading)

## Core Concepts

### Prerequisites

- Basic SQL knowledge
- PostgreSQL installed locally
- Docker for containerized database environments
- Basic understanding of database theory

### 1. PostgreSQL Fundamentals

#### Database Design Principles
- Entity Relationship Diagrams
- Primary/Foreign Keys
- Constraints and Validations
- Data Types and Best Practices

#### Normalization and Denormalization
- 1NF through 5NF
- When to Denormalize
- Performance Impact
- Real-world Trade-offs

#### Indexing Strategies
- B-Tree Indexes
- Hash Indexes
- GiST Indexes
- Partial Indexes
- Covering Indexes

#### Transaction Management
- ACID Properties
- Isolation Levels
- Deadlock Prevention
- Transaction Patterns

### 2. Query Optimization

#### Execution Plans
- EXPLAIN ANALYZE
- Reading Query Plans
- Cost Estimation
- Plan Types

#### Index Types and Usage
- When to Use Each Type
- Index Maintenance
- Index Only Scans
- Multi-Column Indexes

#### Query Performance Analysis
- Common Bottlenecks
- Query Rewriting
- Join Optimization
- Subquery Optimization

### 3. Advanced Topics

#### Partitioning
- Range Partitioning
- List Partitioning
- Hash Partitioning
- Partition Pruning

#### Replication
- Streaming Replication
- Logical Replication
- Conflict Resolution
- Failover Strategies

#### Concurrency Control
- MVCC
- Lock Types
- Row-Level Security
- Advisory Locks

## Practical Exercises

### Setup Instructions

```bash
# Start PostgreSQL with Docker
docker run --name pg-learning -e POSTGRES_PASSWORD=mysecretpassword -d -p 5432:5432 postgres:latest

# Connect to PostgreSQL
psql -h localhost -U postgres -d postgres

# Create learning database
CREATE DATABASE learning_playground;
```

### Exercise 1: Schema Design and Normalization

Design a schema for an e-commerce platform that handles:
- Products with variants (size, color, etc.)
- Customer orders and order history
- Inventory management
- Product categories (hierarchical)
- Customer reviews and ratings

<details>
<summary>View Solution</summary>

```sql
-- Base product information
CREATE TABLE products (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    name varchar(255) NOT NULL,
    description text,
    base_price numeric(10,2) NOT NULL,
    category_id uuid REFERENCES categories(id),
    created_at timestamptz DEFAULT CURRENT_TIMESTAMP
);

-- Product variants
CREATE TABLE product_variants (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id uuid REFERENCES products(id),
    sku varchar(50) UNIQUE NOT NULL,
    price_adjustment numeric(10,2) DEFAULT 0,
    inventory_count integer NOT NULL DEFAULT 0,
    attributes jsonb NOT NULL, -- {size: "M", color: "blue"}
    CONSTRAINT positive_inventory CHECK (inventory_count >= 0)
);

-- Hierarchical categories
CREATE TABLE categories (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    name varchar(100) NOT NULL,
    parent_id uuid REFERENCES categories(id),
    path ltree NOT NULL,
    UNIQUE (parent_id, name)
);

-- Example query for hierarchical categories
CREATE INDEX category_path_idx ON categories USING gist(path);

-- Get all subcategories of 'Electronics'
SELECT * FROM categories 
WHERE path <@ 'Electronics'::ltree;
```
</details>

### Exercise 2: Query Performance Optimization

Given a slow-performing analytics query, optimize it using appropriate indexes and query restructuring.

<details>
<summary>Problem</summary>

```sql
-- Original slow query for daily sales report
SELECT 
    DATE_TRUNC('day', o.created_at) as sale_date,
    c.name as category_name,
    COUNT(DISTINCT o.id) as num_orders,
    COUNT(DISTINCT o.user_id) as num_customers,
    SUM(oi.quantity) as items_sold,
    SUM(oi.quantity * oi.price) as total_revenue
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
JOIN categories c ON p.category_id = c.id
WHERE o.created_at >= NOW() - INTERVAL '30 days'
GROUP BY DATE_TRUNC('day', o.created_at), c.name
ORDER BY sale_date DESC, total_revenue DESC;
```
</details>

<details>
<summary>Solution</summary>

```sql
-- Create necessary indexes
CREATE INDEX idx_orders_created_at 
ON orders USING BRIN(created_at);

CREATE INDEX idx_order_items_composite 
ON order_items(order_id, product_id, quantity, price);

-- Create materialized view for daily aggregates
CREATE MATERIALIZED VIEW daily_sales_by_category AS
WITH daily_stats AS (
    SELECT 
        DATE_TRUNC('day', o.created_at) as sale_date,
        p.category_id,
        o.id as order_id,
        o.user_id,
        oi.quantity,
        oi.quantity * oi.price as revenue
    FROM orders o
    JOIN order_items oi ON o.id = oi.order_id
    JOIN products p ON oi.product_id = p.id
    WHERE o.created_at >= NOW() - INTERVAL '30 days'
)
SELECT 
    sale_date,
    c.name as category_name,
    COUNT(DISTINCT order_id) as num_orders,
    COUNT(DISTINCT user_id) as num_customers,
    SUM(quantity) as items_sold,
    SUM(revenue) as total_revenue
FROM daily_stats
JOIN categories c ON daily_stats.category_id = c.id
GROUP BY sale_date, c.name;

-- Create refresh function
CREATE OR REPLACE FUNCTION refresh_daily_sales()
RETURNS trigger AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_by_category;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Create trigger to refresh materialized view
CREATE TRIGGER refresh_daily_sales_trigger
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH STATEMENT
EXECUTE FUNCTION refresh_daily_sales();
```
</details>

### Exercise 3: Concurrency and Transactions

Implement a ticket booking system that handles concurrent bookings and prevents overselling.

<details>
<summary>View Solution</summary>

```sql
-- Schema
CREATE TABLE events (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    name varchar(255) NOT NULL,
    event_date timestamptz NOT NULL,
    total_seats integer NOT NULL,
    available_seats integer NOT NULL,
    CONSTRAINT valid_seats CHECK (available_seats >= 0 AND available_seats <= total_seats)
);

CREATE TABLE bookings (
    id uuid PRIMARY KEY DEFAULT uuid_generate_v4(),
    event_id uuid REFERENCES events(id),
    user_id uuid REFERENCES users(id),
    num_seats integer NOT NULL,
    status varchar(20) DEFAULT 'pending',
    created_at timestamptz DEFAULT CURRENT_TIMESTAMP
);

-- Booking function with concurrency handling
CREATE OR REPLACE FUNCTION book_tickets(
    p_event_id uuid,
    p_user_id uuid,
    p_num_seats integer
) RETURNS uuid AS $$
DECLARE
    v_booking_id uuid;
    v_available_seats integer;
BEGIN
    -- Lock the event row for update
    SELECT available_seats INTO v_available_seats
    FROM events
    WHERE id = p_event_id
    FOR UPDATE;

    IF v_available_seats >= p_num_seats THEN
        -- Create booking
        INSERT INTO bookings (event_id, user_id, num_seats, status)
        VALUES (p_event_id, p_user_id, p_num_seats, 'confirmed')
        RETURNING id INTO v_booking_id;

        -- Update available seats
        UPDATE events
        SET available_seats = available_seats - p_num_seats
        WHERE id = p_event_id;

        RETURN v_booking_id;
    ELSE
        RAISE EXCEPTION 'Not enough seats available';
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Usage example
BEGIN;
SELECT book_tickets('event-uuid', 'user-uuid', 2);
COMMIT;
```
</details>

### Exercise 4: Partitioning and Data Management

Implement a partitioning strategy for a large-scale logging system.

<details>
<summary>View Solution</summary>

```sql
-- Create partitioned table
CREATE TABLE system_logs (
    id bigserial,
    log_time timestamptz NOT NULL,
    level varchar(20),
    message text,
    metadata jsonb
) PARTITION BY RANGE (log_time);

-- Create partitions
CREATE TABLE system_logs_y2024m01 PARTITION OF system_logs
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE system_logs_y2024m02 PARTITION OF system_logs
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Create partition automatically
CREATE OR REPLACE FUNCTION create_log_partition()
RETURNS trigger AS $$
DECLARE
    partition_date date;
    partition_name text;
    start_date text;
    end_date text;
BEGIN
    partition_date := date_trunc('month', NEW.log_time);
    partition_name := 'system_logs_y' || to_char(partition_date, 'YYYY') || 'm' || to_char(partition_date, 'MM');
    
    IF NOT EXISTS (SELECT 1 FROM pg_class WHERE relname = partition_name) THEN
        start_date := to_char(partition_date, 'YYYY-MM-DD');
        end_date := to_char(partition_date + interval '1 month', 'YYYY-MM-DD');
        
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF system_logs FOR VALUES FROM (%L) TO (%L)',
            partition_name, start_date, end_date
        );
        
        -- Create indexes on the new partition
        EXECUTE format(
            'CREATE INDEX %I ON %I (log_time, level)',
            'idx_' || partition_name || '_time_level',
            partition_name
        );
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Create trigger for automatic partition creation
CREATE TRIGGER create_log_partition_trigger
    BEFORE INSERT ON system_logs
    FOR EACH ROW
    EXECUTE FUNCTION create_log_partition();
```
</details>

## Best Practices and Guidelines

### 1. Naming Conventions

```sql
-- Use snake_case for identifiers
-- Prefix indexes with idx_
-- Use plural for table names
-- Use singular for column names

CREATE TABLE order_items (
    order_id uuid REFERENCES orders(id),
    product_id uuid REFERENCES products(id)
);
```

### 2. Data Integrity

```sql
CREATE TABLE users (
    id uuid PRIMARY KEY,
    email varchar(255) NOT NULL CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    age integer CHECK (age >= 0 AND age <= 150),
    created_at timestamptz DEFAULT CURRENT_TIMESTAMP NOT NULL
);
```

## Common Problems and Solutions

### 1. Race Conditions

<details>
<summary>View Solutions</summary>

```sql
-- Solution 1: Using SERIALIZABLE isolation
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Solution 2: Using explicit locking
SELECT * FROM products 
WHERE id = 'some-uuid' 
FOR UPDATE;

-- Solution 3: Using advisory locks
SELECT pg_advisory_xact_lock(product_id::bigint);
```
</details>

### 2. Slow Aggregate Queries

<details>
<summary>View Solutions</summary>

```sql
-- Solution 1: Materialized view
CREATE MATERIALIZED VIEW product_stats AS
SELECT category_id, COUNT(*) as product_count
FROM products
GROUP BY category_id;

-- Solution 2: Counter table
CREATE TABLE category_counters (
    category_id uuid PRIMARY KEY,
    product_count integer DEFAULT 0
);
```
</details>

## Resources and Further Reading

### Books
- "PostgreSQL: Up and Running"
- "SQL Performance Explained"
- "The Art of PostgreSQL"

### Online Resources
- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/index.html)
- [PostgreSQL Tutorial](https://www.postgresqltutorial.com/)
- [Use The Index, Luke!](https://use-the-index-luke.com/)

### Tools
- [pgAdmin](https://www.pgadmin.org/)
- [DBeaver](https://dbeaver.io/)
- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html)

### Community
- [PostgreSQL Mailing Lists](https://www.postgresql.org/list/)
- [PostgreSQL Wiki](https://wiki.postgresql.org/)
- [Stack Overflow PostgreSQL Tag](https://stackoverflow.com/questions/tagged/postgresql) 