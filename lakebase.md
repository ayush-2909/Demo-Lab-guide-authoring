# Lakebase Documentation

## 1. Overview

**Lakebase** is a fully managed, serverless, PostgreSQL-compatible OLTP database
designed for modern data applications and AI-driven workloads.  
It provides the reliability and SQL ecosystem of PostgreSQL with the elasticity,
automation, and operational simplicity of a cloud-native serverless database.

Lakebase removes the operational burden of provisioning, scaling, tuning,
patching, and maintaining database infrastructure — allowing teams to build and
ship features faster.

---

## 2. What Is Lakebase?

Lakebase is:

- A **serverless PostgreSQL database** (Postgres-compatible)
- Built for **high concurrency** and **low-latency OLTP workloads**
- Fully managed with **automatic scaling**
- Suitable for **traditional apps**, **microservices**, and **AI systems** as a state store
- Able to store structured, semi-structured (JSONB), and vector data (if supported)
- Consumption-based — pay-for-use model

In one line:

> **Lakebase = PostgreSQL compatibility + serverless autoscaling + managed operations.**

---

## 3. Key Features

- **Serverless autoscaling** (compute/throughput adjusts to load)
- **PostgreSQL compatibility** (same drivers, tools, extensions support)
- **High concurrency & low latency** for many small reads/writes
- **ACID transactions** and durability
- **JSONB support** (semi-structured data)
- **Vector/embedding support** where available (pgvector or similar)
- **Observability & metrics**: query latency, CPU/compute units, connections, IOPS
- **Automatic backups/snapshots** (depends on provider settings)
- **Zero-ops maintenance**: patches, replication, HA handled by the service

---

## 4. Why Lakebase Matters

- Removes DB infra complexity (no VM sizing, patching, replication to manage)
- Ideal for spiky traffic and unpredictable workloads
- Simplifies architectures that combine OLTP and AI state needs
- Increases developer velocity (faster provisioning, easier migration)
- Cost-efficient for bursty workloads (pay-for-use vs always-on VM)

---

## 5. Architecture (Conceptual)

Application(s) (APIs, Services, Agents)
|
v
Lakebase Layer (Postgres engine + serverless compute)
|
v
Cloud Storage (durable logs, backups, snapshots)


---

## 6. Primary Use Cases

- High-concurrency transactional backends (e-commerce, bookings)
- Microservices primary datastore
- AI agent state & memory store (user sessions, contexts)
- RAG augmentations where embeddings + relational data co-exist
- Real-time operational analytics & audit logs
- Hybrid structured + semi-structured feature storage (JSONB)

---

## 7. Demo Flow (Non-AI demo, 15–25 minutes)

1. **Provision lakebase instance** (via console/CLI/Terraform). Capture connection string.
2. **Connect** using `psql` or GUI (DBeaver, TablePlus).
3. **Create demo schema** (customers, orders, big_orders).
4. **Insert sample data** (small sample + efficient 100k row generator).
5. **Show measurable features**:
   - Query execution time (`\timing on`) for joins
   - Metrics dashboard: CPU/compute spikes during load
   - Concurrency test (3 tabs: load, read, write)
   - ACID demo (BEGIN / ROLLBACK)
   - JSONB queries
6. **Explain differences** vs Postgres on VM.
7. **(Optional)** Export data and delete instance (show how to safely back up).

---

## 8. Demo Scripts & Commands

> **Note:** Replace connection placeholders with your Lakebase values:
>
> `postgres://USERNAME:PASSWORD@HOST:PORT/DATABASE`

### 8.1 Connect using psql

    psql "postgres://username:password@host.lakebase.com:5432/defaultdb"

### 8.2 Create demo schema

    -- customers and orders
    CREATE TABLE customers (
        customer_id SERIAL PRIMARY KEY,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT NOW()
    );

    CREATE TABLE orders (
        order_id SERIAL PRIMARY KEY,
        customer_id INT REFERENCES customers(customer_id),
        amount DECIMAL(10,2),
        status TEXT,
        created_at TIMESTAMP DEFAULT NOW()
    );

    -- large table for performance testing
    CREATE TABLE big_orders (
        order_id SERIAL PRIMARY KEY,
        customer_id INT,
        amount DECIMAL(10,2),
        status TEXT,
        created_at TIMESTAMP DEFAULT NOW()
    );

### 8.3 Insert small sample data

    INSERT INTO customers (name, email)
    VALUES 
      ('John Doe', 'john@example.com'),
      ('Ayush Sharma', 'ayush@example.com'),
      ('Maria Lopez', 'maria@example.com');

    INSERT INTO orders (customer_id, amount, status)
    VALUES
      (1, 59.99, 'Completed'),
      (1, 120.50, 'Processing'),
      (2, 10.00, 'Completed'),
      (3, 220.00, 'Cancelled'),
      (3, 399.99, 'Completed');

### 8.4 Efficiently generate 100,000 rows (fixed script)

    INSERT INTO big_orders (customer_id, amount, status)
    SELECT
        (random() * 1000)::int,
        (random() * 500)::numeric(10,2),
        (ARRAY['Completed', 'Processing', 'Cancelled'])[ceil(random()*3)]
    FROM generate_series(1, 100000);

    -- verify
    SELECT COUNT(*) FROM big_orders;

### 8.5 Add index (optional) for query performance demo

    CREATE INDEX idx_big_orders_customer ON big_orders(customer_id);

### 8.6 Show timing / fast joins

    -- enable timing in psql:
    \timing on

    -- join query
    SELECT c.name, o.amount, o.status
    FROM customers c
    JOIN orders o ON c.customer_id = o.customer_id;

    -- with big table join/aggregation
    SELECT c.name, COUNT(o.order_id) AS orders_count, SUM(o.amount) AS total_amount
    FROM customers c
    JOIN big_orders o ON c.customer_id = o.customer_id
    GROUP BY c.name;

_Show `Time: X ms` reported by the client to demonstrate latency._

### 8.7 Concurrency test (open 3 tabs)

Tab A (load):

    DO $$
    BEGIN
        FOR i IN 1..20000 LOOP
            INSERT INTO orders (customer_id, amount, status)
            VALUES (1, i * 0.9, 'ConcurrentLoad');
        END LOOP;
    END $$;

Tab B (long read):

    SELECT pg_sleep(2), COUNT(*) FROM orders;

Tab C (quick write):

    INSERT INTO orders (customer_id, amount, status)
    VALUES (2, 49.99, 'WorksEvenUnderLoad');

_Show Tab C returns quickly and Tab B completes without blocking — proof of concurrency handling._

### 8.8 ACID transaction / rollback demo

    BEGIN;
    INSERT INTO orders (customer_id, amount, status)
    VALUES (1, 9999.99, 'TestTransaction');
    ROLLBACK;

    SELECT * FROM orders WHERE amount = 9999.99;  -- should return 0 rows

### 8.9 JSONB example

    ALTER TABLE customers ADD COLUMN preferences JSONB;

    UPDATE customers
    SET preferences = '{"theme": "dark", "language": "en"}'
    WHERE customer_id = 2;

    SELECT name, preferences->>'theme' AS theme
    FROM customers
    WHERE preferences->>'language' = 'en';

### 8.10 Export (pg_dump) before deletion

    -- full DB dump
    pg_dump "postgres://username:password@host:5432/defaultdb" -F c -f lakebase_backup.dump

    -- restore to another Postgres
    pg_restore -d targetdb lakebase_backup.dump

---

## 9. How To Visibly Demonstrate Key Features (what the audience actually sees)

### Fast joins / query latency
- Enable client timing (`\timing on`) and show `Time: X ms`.
- Compare query times before/after index creation or with/without large dataset.

### Autoscaling (serverless)
- Open Lakebase provider metrics dashboard (Console). Note baseline CPU/compute.
- Start heavy insert workload (e.g., 30k inserts).
- Refresh dashboard to show:
  - CPU/compute units spike
  - Connection/IOPS increase
- While load runs, run simple SELECT queries to show latency remains low.

> If the service exposes "compute units" or "provisioned capacity" graph, point that out — that’s the clearest evidence of autoscaling.

### Concurrency
- Run loads and simultaneous reads/writes in multiple client tabs.
- Team can see responses remain snappy.

### Reliability / ACID
- Demonstrate BEGIN/ROLLBACK and prove the rollback removed the inserted row.

---

## 10. Lakebase vs Postgres on VM — Short Comparison

| Aspect | Postgres on VM | Lakebase |
|---|---:|:---|
| Provisioning | VM + disk + DB install | Provision in minutes |
| Scaling | Manual, vertical/horizontal | Automatic, serverless |
| Maintenance | OS + DB patches | Managed by vendor |
| Cost model | Always-on instance | Pay-for-use (bursty savings) |
| Concurrency | Limited by VM size | High concurrency support |
| Backup & HA | Manual config | Built-in (service-dependent) |

---

## 11. Safety: Deleting the Instance

- **Deleting a Lakebase instance will delete all contained data** unless a final snapshot/backup is taken.
- Prior to deletion: perform `pg_dump` or create snapshots via the provider console.
- Confirm backups are restorable before deleting production or demo data.

---

## 12. Troubleshooting & Tips

- Use connection pooling (pgbouncer) for many short-lived connections if recommended by provider.
- Monitor connection limits; serverless often handles many connections but check provider docs.
- Use indexes for heavy read patterns (B-tree for equality, GIN for JSONB).
- For vector workloads, test `pgvector` support and measure similarity query performance.
- When demonstrating autoscaling, give the metrics dashboard a few seconds to refresh after starting load.

---

## 13. Appendix: Quick Checklist for Demo Day

- [ ] Provision Lakebase and confirm connection string
- [ ] Prepare SQL client (psql / DBeaver) with saved connection
- [ ] Create schema & run the 100k generator beforehand (or during demo)
- [ ] Open provider metrics dashboard in a visible browser tab
- [ ] Prepare 3 SQL client tabs for concurrency demo
- [ ] Have `pg_dump` command ready for backup demonstration
- [ ] Practice the script and talk track once end-to-end

---

## 14. Final Notes

Lakebase provides a modern, developer-friendly path from "traditional Postgres on a VM" to an elastic, managed Postgres experience that fits modern application, microservice, and AI-agent needs.  
For your internal demo, emphasize measurable proofs (timing, dashboards, concurrent behavior) rather than just describing benefits.



