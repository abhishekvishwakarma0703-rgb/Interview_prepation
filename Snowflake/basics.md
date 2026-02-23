# The Complete Snowflake Guide: Zero to Advanced Data Engineering

### From “What is a database?” to production-grade data platforms

-----

## Table of Contents

1. [Part 1 — The World Before Snowflake: Foundational Concepts](#part-1)
1. [Part 2 — What is Snowflake?](#part-2)
1. [Part 3 — Snowflake Architecture Deep Dive](#part-3)
1. [Part 4 — Getting Started: Your First Snowflake Account](#part-4)
1. [Part 5 — SQL Fundamentals in Snowflake](#part-5)
1. [Part 6 — Intermediate SQL & Snowflake-Specific Functions](#part-6)
1. [Part 7 — Data Loading & Integration (ETL/ELT)](#part-7)
1. [Part 8 — Data Modeling & Warehousing Patterns](#part-8)
1. [Part 9 — Advanced Snowflake Features](#part-9)
1. [Part 10 — Performance Optimization](#part-10)
1. [Part 11 — Security, Governance & Administration](#part-11)
1. [Part 12 — Real Project: End-to-End E-Commerce Analytics Platform](#part-12)
1. [Part 13 — Data Engineering Patterns & Best Practices](#part-13)
1. [Part 14 — Snowflake Ecosystem & Integrations](#part-14)

-----

<a name="part-1"></a>

# PART 1 — The World Before Snowflake: Foundational Concepts

Before touching Snowflake, you need to understand the problems it was built to solve. This section builds mental models from scratch.

-----

## 1.1 What is Data and Why Do We Store It?

Imagine you run an online store. Every time someone buys something, you have a “fact”: who bought it, what they bought, when, how much they paid. If you store nothing, you can never answer questions like “What was my best-selling product last month?” or “Which customers spend the most?”

**Data** is just recorded facts. **A database** is an organized system for storing and retrieving those facts efficiently.

-----

## 1.2 From Spreadsheet to Database: Why Files Aren’t Enough

You might start storing data in Excel. That works when you have 1,000 rows. But what happens when you have:

- 10 million rows (Excel crashes)
- 50 people trying to update data simultaneously (file conflicts)
- Complex questions requiring combining dozens of columns across multiple sheets (manual nightmare)
- Data that must never be lost (no backup, no audit trail)

This is why **databases** exist. A database is software that manages data professionally — handling scale, concurrency, reliability, and complex queries.

-----

## 1.3 Relational Databases (RDBMS) — The Foundation

The most common type of database is a **Relational Database Management System (RDBMS)**. Think of it as a collection of tables (like spreadsheets) that are connected to each other through shared keys.

### Example: An Online Store Database

**customers table:**

|customer_id|name       |email          |city    |
|-----------|-----------|---------------|--------|
|1          |Alice Smith|alice@email.com|New York|
|2          |Bob Jones  |bob@email.com  |Chicago |

**orders table:**

|order_id|customer_id|order_date|total_amount|
|--------|-----------|----------|------------|
|1001    |1          |2024-01-15|89.99       |
|1002    |2          |2024-01-16|234.50      |
|1003    |1          |2024-01-20|45.00       |

Notice that `customer_id` links the two tables. Alice (customer_id=1) has two orders. This linking mechanism is called a **foreign key relationship**.

### SQL — The Language of Databases

**SQL (Structured Query Language)** is how you talk to a relational database. It’s a declarative language — you describe *what* you want, not *how* to get it.

```sql
-- Get all orders for Alice with the order details
SELECT 
    c.name,
    o.order_date,
    o.total_amount
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE c.name = 'Alice Smith';
```

-----

## 1.4 OLTP vs OLAP — Two Completely Different Worlds

This is one of the most important distinctions in all of data engineering.

### OLTP — Online Transaction Processing

**What it is:** The operational database your business *runs on*. Your website hits this database every time a customer places an order, creates an account, or updates their cart.

**Characteristics:**

- Thousands of small, fast read/write operations per second
- Optimized for single-row lookups (“get order #1234”)
- Data is current (updated in real-time)
- Must never go down (your business depends on it)
- Examples: MySQL, PostgreSQL, SQL Server, Oracle

**Analogy:** A cash register at a store. It needs to process each transaction quickly and reliably.

### OLAP — Online Analytical Processing

**What it is:** A system optimized for answering complex analytical questions across millions or billions of rows.

**Characteristics:**

- Few, very complex queries that scan huge amounts of data
- Optimized for aggregations (“total sales by region by month for the last 3 years”)
- Data is historical (often loaded in batches)
- Can handle some downtime
- Examples: Snowflake, BigQuery, Redshift, Databricks

**Analogy:** A business intelligence analyst with a supercomputer reviewing all historical receipts to find trends.

### Why You Can’t Use OLTP for Analytics

Imagine running this query on your live production OLTP database:

```sql
SELECT 
    region,
    product_category,
    YEAR(order_date) as year,
    MONTH(order_date) as month,
    SUM(revenue) as total_revenue,
    COUNT(DISTINCT customer_id) as unique_customers
FROM orders
JOIN order_items ON orders.order_id = order_items.order_id
JOIN products ON order_items.product_id = products.product_id
JOIN customers ON orders.customer_id = customers.customer_id
WHERE order_date >= '2020-01-01'
GROUP BY region, product_category, year, month
ORDER BY year, month, total_revenue DESC;
```

This query needs to scan *billions* of rows. Running it on your OLTP database would:

1. Lock up tables, making your website painfully slow
1. Take minutes or hours to complete
1. Potentially crash the database

This is why organizations build **separate analytical systems** — data warehouses.

-----

## 1.5 What is a Data Warehouse?

A **Data Warehouse** is a specialized database designed for analytical workloads. It’s where you copy data from your operational systems (OLTP) so analysts can query it without impacting the live business.

Key properties:

- **Subject-oriented:** Organized around business topics (sales, customers, inventory)
- **Integrated:** Data from many source systems combined into one place
- **Non-volatile:** Historical data is kept; nothing is deleted or overwritten
- **Time-variant:** You can analyze trends over time

### How Data Gets into a Data Warehouse — ETL and ELT

**ETL (Extract, Transform, Load):**

1. **Extract** data from source systems (your OLTP database, APIs, files)
1. **Transform** it — clean it, reshape it, combine it, apply business rules
1. **Load** it into the data warehouse

**ELT (Extract, Load, Transform):**

1. **Extract** data from source systems
1. **Load** raw data directly into the warehouse
1. **Transform** it *inside* the warehouse using SQL

ELT has become dominant because modern cloud data warehouses (like Snowflake) are so powerful that running transformations inside them is faster and simpler than on a separate server.

-----

## 1.6 What is a Data Lake?

A **Data Lake** stores raw, unprocessed data at massive scale in its native format (files — CSV, JSON, Parquet, images, logs, etc.).

|Feature    |Data Warehouse     |Data Lake         |
|-----------|-------------------|------------------|
|Data type  |Structured (tables)|Any format        |
|Schema     |Defined upfront    |Schema on read    |
|Users      |Analysts/BI        |Data scientists/ML|
|Cost       |More expensive     |Cheaper storage   |
|Query speed|Fast               |Slower (raw files)|

Modern platforms like Snowflake blur this line — you can query semi-structured data (JSON) directly in Snowflake, making it act partly like a data lake.

-----

## 1.7 Column-Oriented Storage — The Secret Weapon of Analytics

Traditional OLTP databases store data **row by row**:

```
[Row 1: Alice, New York, 2024, 89.99]
[Row 2: Bob, Chicago, 2024, 234.50]
[Row 3: Alice, New York, 2024, 45.00]
```

This is great for retrieving a single complete record (“get me all of Alice’s details”).

But for analytics, you usually only need a few columns from millions of rows:

```sql
SELECT SUM(total_amount) FROM orders WHERE year = 2024;
```

Here you only need `total_amount` and `year`. In a row store, you still have to read every column of every row — wasteful.

**Columnar storage** stores data column by column:

```
[total_amount column: 89.99, 234.50, 45.00, ...]
[year column: 2024, 2024, 2024, ...]
[city column: New York, Chicago, New York, ...]
```

For the query above, you only read two columns — often 10-100x less data. This is why analytical databases like Snowflake are orders of magnitude faster for analytical queries.

Columnar storage also compresses extremely well because similar values are stored together (e.g., a column with “2024” repeated a million times compresses to almost nothing).

-----

<a name="part-2"></a>

# PART 2 — What is Snowflake?

-----

## 2.1 Snowflake’s Origin Story

Before Snowflake (founded 2012, launched 2014), building a data warehouse meant:

- Buying expensive hardware upfront
- Provisioning fixed capacity (you’re stuck with whatever you bought)
- Managing physical servers, patches, backups
- Either over-provisioning (wasting money) or under-provisioning (slow queries)
- Separate systems for storage and compute that were hard to scale independently

Snowflake was built from the ground up as a **cloud-native** data platform designed to eliminate all of these pain points.

-----

## 2.2 What Makes Snowflake Different

Snowflake is a **cloud data platform** that provides:

1. **A data warehouse** — Store and query structured data at any scale
1. **Data lake capabilities** — Query semi-structured data (JSON, Avro, Parquet) natively
1. **Data engineering** — Build pipelines, process streaming data
1. **Data sharing** — Share live data with other organizations without copying it
1. **Data marketplace** — Access or sell third-party data
1. **Application development** — Build apps that run on Snowflake data

### Key Differentiators

**Separate storage and compute.** In traditional systems, storage and compute are coupled — you can’t scale one without the other. Snowflake separates them completely. You can:

- Have one storage layer (cheap, unlimited)
- Run 10 different compute clusters against the same data simultaneously
- Scale compute up or down in seconds
- Pay only for what you use

**Multi-cloud.** Snowflake runs on AWS, Azure, and GCP. You choose where your data lives.

**Zero management.** No servers, no patching, no index tuning (mostly), no vacuum operations. Snowflake handles all infrastructure.

**Automatic scaling.** Query too slow? Snowflake can automatically add more compute. Done with analysis? It shuts down and stops billing you.

-----

<a name="part-3"></a>

# PART 3 — Snowflake Architecture Deep Dive

Understanding Snowflake’s architecture is essential to using it well and optimizing performance.

-----

## 3.1 The Three Layers

Snowflake has three distinct architectural layers:

```
┌─────────────────────────────────────────────────────┐
│           CLOUD SERVICES LAYER                       │
│  (Metadata, Auth, Query Optimization, Security)      │
├──────────────┬──────────────┬───────────────────────┤
│  COMPUTE     │   COMPUTE    │    COMPUTE             │
│  Warehouse 1 │   Warehouse 2│    Warehouse 3         │
│  (Virtual    │   (Virtual   │    (Virtual            │
│   Warehouse) │    Warehouse)│     Warehouse)         │
├──────────────┴──────────────┴───────────────────────┤
│           CENTRALIZED STORAGE LAYER                  │
│  (S3 / Azure Blob / GCS — Columnar, Compressed)     │
└─────────────────────────────────────────────────────┘
```

### Layer 1: Centralized Storage

All your data lives here as highly compressed, optimized columnar files (using Snowflake’s proprietary format built on top of Parquet-like micro-partitions) in cloud object storage (S3, Azure Blob, GCS).

Key properties:

- Storage is shared — all compute can read any data
- Automatically replicated for durability
- Charged per TB stored per month (~$23/TB/month on AWS)
- Includes **Time Travel** data (historical versions) automatically

### Layer 2: Virtual Warehouses (Compute)

A **Virtual Warehouse** (VW) is a cluster of compute resources (CPUs and RAM) that execute your queries. It’s completely separate from storage.

```
┌─────────────────────────────────────────────────────┐
│  Virtual Warehouse "ANALYTICS_WH" (XL)              │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐    │
│  │ Node 1 │  │ Node 2 │  │ Node 3 │  │ Node 4 │    │
│  │ 64 CPU │  │ 64 CPU │  │ 64 CPU │  │ 64 CPU │    │
│  │ 512GB  │  │ 512GB  │  │ 512GB  │  │ 512GB  │    │
│  │  RAM   │  │  RAM   │  │  RAM   │  │  RAM   │    │
│  └────────┘  └────────┘  └────────┘  └────────┘    │
│         Local SSD Cache (Result Cache)               │
└─────────────────────────────────────────────────────┘
```

Key properties:

- Can be suspended (no charge) and resumed in seconds
- Multiple VWs can query the same data simultaneously without conflict
- Each VW has its own local SSD disk cache for recently used data
- Size ranges from XS (1 node) to 6XL (512 nodes)
- Charged per credit per hour (each size = different credits per hour)

### Layer 3: Cloud Services

This brain layer manages everything:

- **Authentication and Authorization:** Who can do what
- **Infrastructure Manager:** Provisioning VWs
- **Metadata Store:** Table schemas, statistics, file locations — stored in memory for performance
- **Query Parser and Optimizer:** Transforms your SQL into an optimal execution plan
- **Transaction Manager:** Ensures data consistency

-----

## 3.2 Micro-Partitions: How Data is Actually Stored

This is fundamental to understanding Snowflake performance.

When you load data into Snowflake, it automatically divides your data into **micro-partitions** — contiguous units of storage containing 50MB–500MB of uncompressed data (much smaller when compressed, typically 16MB compressed).

```
Table: orders (1 billion rows)
├── Micro-partition 1: rows 1–500,000
├── Micro-partition 2: rows 500,001–1,000,000
├── Micro-partition 3: rows 1,000,001–1,500,000
...
└── Micro-partition 2000: rows 999,500,001–1,000,000,000
```

Each micro-partition:

- Stores data in columnar format within the partition
- Is automatically compressed (Snowflake chooses the best compression algorithm per column)
- Maintains metadata: min/max values for every column, distinct value counts, etc.

### Partition Pruning — Why This Matters for Performance

Because Snowflake tracks the min/max values of every column in every micro-partition, it can skip partitions that can’t possibly contain data you’re looking for.

```sql
SELECT * FROM orders WHERE order_date = '2024-01-15';
```

Instead of scanning all 2,000 micro-partitions, Snowflake checks metadata:

- “Partition 1: order_date range is 2023-01-01 to 2023-03-15 → SKIP”
- “Partition 847: order_date range is 2024-01-10 to 2024-01-20 → SCAN”
- “Partition 848: order_date range is 2024-01-20 to 2024-01-30 → SKIP”

Result: Instead of scanning 1 billion rows, you might scan only 10 million. This is called **partition pruning** and it’s automatic — Snowflake does it without you defining any indexes.

-----

## 3.3 Clustering Keys

By default, data is organized in micro-partitions in the order it was loaded. If your data is naturally ordered by date (most common), partition pruning works great on date filters.

But if you frequently filter on a different column (e.g., `customer_id` or `product_category`), micro-partitions might contain data from all possible values of that column, making pruning ineffective.

**Clustering keys** tell Snowflake to physically reorganize data so rows with similar values of the clustering key are stored in the same micro-partitions.

```sql
-- Define clustering key on a table
ALTER TABLE orders CLUSTER BY (order_date, region);
```

Snowflake then maintains this clustering automatically in the background (Automatic Clustering).

-----

## 3.4 Caching — Three Levels

Snowflake has three caching layers that dramatically reduce query times and costs:

**Level 1: Result Cache (Cloud Services Layer)**
When you run a query, Snowflake caches the result for 24 hours. If anyone runs the exact same query again (same SQL text, same data), it returns the cached result instantly — no compute used, no cost.

```sql
-- First execution: 45 seconds, costs 0.5 credits
SELECT region, SUM(revenue) FROM sales GROUP BY region;

-- Second execution (within 24 hours): 0.1 seconds, costs 0 credits
SELECT region, SUM(revenue) FROM sales GROUP BY region;
```

**Level 2: Local Disk Cache (Virtual Warehouse)**
Each VW has a local SSD cache. When micro-partitions are read from storage, they’re cached on the VW’s local SSDs. Subsequent queries that need the same micro-partitions read from local SSD (10-100x faster than remote storage).

This cache persists as long as the VW is running. Suspending the VW clears it.

**Level 3: Metadata Cache (Cloud Services)**
Column statistics, partition metadata, and schema information are always held in memory in the Cloud Services layer for instant access.

-----

<a name="part-4"></a>

# PART 4 — Getting Started: Your First Snowflake Account

-----

## 4.1 Creating a Free Trial Account

1. Go to **app.snowflake.com** and click “Start for free”
1. Choose your cloud provider (AWS, Azure, GCP) and region
1. You get $400 in free credits — enough to follow this entire guide

Once logged in, you see the **Snowsight** interface — Snowflake’s modern web UI.

-----

## 4.2 Key UI Areas

**Worksheets:** Where you write and run SQL queries. Think of each worksheet as a SQL editor tab.

**Databases:** Browse your database objects (databases, schemas, tables, views, functions)

**Warehouses:** Create and manage your Virtual Warehouses

**Data:** Snowflake Marketplace and data sharing features

**Activity:** Query history, running queries, task history

-----

## 4.3 The Object Hierarchy

Snowflake organizes objects in a hierarchy:

```
Account
└── Database (logical container — like a folder)
    └── Schema (sub-container for related objects)
        ├── Tables
        ├── Views
        ├── Stages (file locations for loading)
        ├── Functions
        ├── Procedures
        └── Pipes (streaming ingestion)
```

### Creating Your First Objects

```sql
-- Create a database
CREATE DATABASE ecommerce_analytics;

-- Use the database
USE DATABASE ecommerce_analytics;

-- Create schemas to organize your work
CREATE SCHEMA raw;         -- Raw data as loaded
CREATE SCHEMA staging;     -- Cleaned/transformed data  
CREATE SCHEMA analytics;   -- Business-ready tables and views
CREATE SCHEMA audit;       -- Audit logs and metadata

-- Select what context you're working in
USE SCHEMA ecommerce_analytics.raw;
```

-----

## 4.4 Creating and Managing Virtual Warehouses

```sql
-- Create a warehouse for interactive analytics
CREATE WAREHOUSE analytics_wh
    WAREHOUSE_SIZE = 'X-SMALL'         -- Start small
    AUTO_SUSPEND = 60                   -- Suspend after 60 seconds of inactivity
    AUTO_RESUME = TRUE                  -- Auto-resume when a query is submitted
    INITIALLY_SUSPENDED = TRUE          -- Don't start it right now
    COMMENT = 'Warehouse for ad-hoc analytics';

-- Create a larger warehouse for heavy ETL jobs
CREATE WAREHOUSE etl_wh
    WAREHOUSE_SIZE = 'LARGE'
    AUTO_SUSPEND = 120
    AUTO_RESUME = TRUE
    INITIALLY_SUSPENDED = TRUE;

-- Use a warehouse
USE WAREHOUSE analytics_wh;

-- Resize a warehouse (takes effect immediately, even mid-query)
ALTER WAREHOUSE analytics_wh SET WAREHOUSE_SIZE = 'SMALL';

-- Suspend manually
ALTER WAREHOUSE analytics_wh SUSPEND;

-- Resume
ALTER WAREHOUSE analytics_wh RESUME;
```

### Warehouse Sizes and Credits

|Size|Nodes|Credits/Hour|Use Case                        |
|----|-----|------------|--------------------------------|
|XS  |1    |1           |Testing, small queries          |
|S   |2    |2           |Light analytics                 |
|M   |4    |4           |Regular analytics               |
|L   |8    |8           |Complex queries, moderate data  |
|XL  |16   |16          |Large datasets, concurrent users|
|2XL |32   |32          |Heavy ELT, large aggregations   |
|4XL |128  |128         |Massive parallel processing     |
|6XL |512  |512         |Largest analytical workloads    |

**Credit cost:** Approximately $2–$4 per credit on AWS (varies by region/contract). An XS warehouse running for 1 hour costs ~$2–$4.

-----

<a name="part-5"></a>

# PART 5 — SQL Fundamentals in Snowflake

SQL in Snowflake is standard SQL with many powerful extensions. Let’s build from the ground up.

-----

## 5.1 Creating Tables

```sql
USE DATABASE ecommerce_analytics;
USE SCHEMA raw;
USE WAREHOUSE analytics_wh;

-- Basic table creation
CREATE TABLE customers (
    customer_id     INTEGER         NOT NULL,
    first_name      VARCHAR(100)    NOT NULL,
    last_name       VARCHAR(100)    NOT NULL,
    email           VARCHAR(255)    NOT NULL UNIQUE,
    phone           VARCHAR(20),
    date_of_birth   DATE,
    country         VARCHAR(50),
    city            VARCHAR(100),
    created_at      TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    is_active       BOOLEAN         DEFAULT TRUE,
    lifetime_value  DECIMAL(10, 2),
    PRIMARY KEY (customer_id)
);

CREATE TABLE products (
    product_id      INTEGER         NOT NULL,
    product_name    VARCHAR(500)    NOT NULL,
    category        VARCHAR(100),
    subcategory     VARCHAR(100),
    brand           VARCHAR(100),
    unit_price      DECIMAL(10, 2),
    cost_price      DECIMAL(10, 2),
    stock_quantity  INTEGER         DEFAULT 0,
    created_at      TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    PRIMARY KEY (product_id)
);

CREATE TABLE orders (
    order_id        INTEGER         NOT NULL,
    customer_id     INTEGER         NOT NULL,
    order_date      DATE            NOT NULL,
    order_status    VARCHAR(50),
    shipping_country VARCHAR(50),
    discount_pct    DECIMAL(5, 2)   DEFAULT 0,
    created_at      TIMESTAMP_NTZ   DEFAULT CURRENT_TIMESTAMP(),
    PRIMARY KEY (order_id),
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    order_item_id   INTEGER         NOT NULL,
    order_id        INTEGER         NOT NULL,
    product_id      INTEGER         NOT NULL,
    quantity        INTEGER         NOT NULL,
    unit_price      DECIMAL(10, 2)  NOT NULL,
    discount_pct    DECIMAL(5, 2)   DEFAULT 0,
    PRIMARY KEY (order_item_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

### Snowflake Data Types Reference

```sql
-- Numeric types
INTEGER, INT, BIGINT          -- Whole numbers
DECIMAL(p, s), NUMERIC(p, s)  -- Exact decimals (p=total digits, s=decimal places)
FLOAT, DOUBLE                  -- Approximate decimals (use for measurements/coordinates)
NUMBER(38, 0)                  -- Snowflake's native numeric type

-- Text types
VARCHAR(n)                    -- Variable-length string up to n chars (max 16MB)
STRING                        -- Alias for VARCHAR, no length limit needed
CHAR(n)                       -- Fixed-length string
TEXT                          -- Alias for VARCHAR

-- Date/Time types
DATE                          -- Date only (2024-01-15)
TIME                          -- Time only (14:30:00)
TIMESTAMP_NTZ                 -- Timestamp with no timezone (most common for ETL)
TIMESTAMP_LTZ                 -- Timestamp with local timezone
TIMESTAMP_TZ                  -- Timestamp with stored timezone offset
TIMESTAMP                     -- Alias for TIMESTAMP_NTZ by default

-- Semi-structured types (Snowflake-specific!)
VARIANT                       -- Can store ANY JSON, XML, Avro, Parquet value
ARRAY                         -- Array of values
OBJECT                        -- JSON object

-- Other types
BOOLEAN                       -- TRUE/FALSE/NULL
BINARY, VARBINARY             -- Binary data
GEOGRAPHY                     -- Geospatial data
```

-----

## 5.2 Inserting and Loading Data

```sql
-- Insert single rows (rarely used in production, good for testing)
INSERT INTO customers (customer_id, first_name, last_name, email, country, city)
VALUES 
    (1, 'Alice', 'Smith', 'alice@email.com', 'USA', 'New York'),
    (2, 'Bob', 'Jones', 'bob@email.com', 'UK', 'London'),
    (3, 'Carlos', 'Lopez', 'carlos@email.com', 'Spain', 'Madrid'),
    (4, 'Diana', 'Chen', 'diana@email.com', 'USA', 'San Francisco'),
    (5, 'Eve', 'Williams', 'eve@email.com', 'Canada', 'Toronto');

INSERT INTO products (product_id, product_name, category, subcategory, brand, unit_price, cost_price)
VALUES
    (101, 'Wireless Noise-Cancelling Headphones', 'Electronics', 'Audio', 'SoundMax', 299.99, 120.00),
    (102, 'Running Shoes Pro', 'Sports', 'Footwear', 'SpeedFit', 149.99, 60.00),
    (103, 'Organic Coffee Beans 1kg', 'Food', 'Beverages', 'BeanHouse', 24.99, 8.00),
    (104, 'Smart Watch Series X', 'Electronics', 'Wearables', 'TechWear', 399.99, 180.00),
    (105, 'Yoga Mat Premium', 'Sports', 'Fitness', 'ZenFit', 59.99, 20.00);

INSERT INTO orders (order_id, customer_id, order_date, order_status, shipping_country)
VALUES
    (1001, 1, '2024-01-15', 'completed', 'USA'),
    (1002, 2, '2024-01-16', 'completed', 'UK'),
    (1003, 1, '2024-01-20', 'completed', 'USA'),
    (1004, 3, '2024-02-05', 'shipped', 'Spain'),
    (1005, 4, '2024-02-10', 'processing', 'USA'),
    (1006, 5, '2024-02-15', 'completed', 'Canada');

INSERT INTO order_items (order_item_id, order_id, product_id, quantity, unit_price, discount_pct)
VALUES
    (1, 1001, 101, 1, 299.99, 0),
    (2, 1001, 105, 2, 59.99, 10),
    (3, 1002, 102, 1, 149.99, 0),
    (4, 1003, 103, 3, 24.99, 0),
    (5, 1003, 104, 1, 399.99, 5),
    (6, 1004, 101, 1, 299.99, 15),
    (7, 1005, 102, 2, 149.99, 0),
    (8, 1006, 103, 5, 24.99, 20);
```

-----

## 5.3 SELECT Queries — From Simple to Complex

```sql
-- Basic SELECT
SELECT * FROM customers;

-- Select specific columns
SELECT customer_id, first_name, last_name, country FROM customers;

-- Filtering with WHERE
SELECT * FROM customers WHERE country = 'USA';

-- Multiple conditions
SELECT * FROM customers 
WHERE country = 'USA' AND city = 'New York';

-- OR condition
SELECT * FROM customers 
WHERE country = 'USA' OR country = 'UK';

-- IN operator (cleaner than multiple ORs)
SELECT * FROM customers 
WHERE country IN ('USA', 'UK', 'Canada');

-- LIKE for pattern matching
SELECT * FROM customers WHERE email LIKE '%@email.com';
SELECT * FROM products WHERE product_name LIKE '%Headphones%';

-- NULL handling
SELECT * FROM customers WHERE phone IS NULL;
SELECT * FROM customers WHERE phone IS NOT NULL;

-- Range queries
SELECT * FROM orders WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31';

-- Ordering results
SELECT * FROM orders ORDER BY order_date DESC;
SELECT * FROM customers ORDER BY country ASC, last_name ASC;

-- Limiting results
SELECT * FROM orders ORDER BY order_date DESC LIMIT 10;

-- Distinct values
SELECT DISTINCT country FROM customers;
SELECT DISTINCT category, subcategory FROM products;
```

-----

## 5.4 Aggregations and GROUP BY

```sql
-- COUNT
SELECT COUNT(*) AS total_orders FROM orders;
SELECT COUNT(DISTINCT customer_id) AS unique_customers FROM orders;

-- SUM, AVG, MIN, MAX
SELECT 
    SUM(unit_price * quantity) AS total_revenue,
    AVG(unit_price * quantity) AS avg_order_item_value,
    MIN(unit_price) AS cheapest_item,
    MAX(unit_price) AS most_expensive_item
FROM order_items;

-- GROUP BY — aggregate by a dimension
SELECT 
    o.shipping_country,
    COUNT(o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(oi.unit_price * oi.quantity) AS gross_revenue,
    SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct/100)) AS net_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.shipping_country
ORDER BY gross_revenue DESC;

-- HAVING — filter after aggregation (WHERE filters before)
SELECT 
    customer_id,
    COUNT(order_id) AS order_count,
    SUM(order_items.unit_price * order_items.quantity) AS total_spend
FROM orders
JOIN order_items USING (order_id)
GROUP BY customer_id
HAVING total_spend > 200    -- Only customers who spent more than $200
ORDER BY total_spend DESC;
```

-----

## 5.5 JOINs — Combining Tables

```sql
-- INNER JOIN — only rows that match in both tables
SELECT 
    c.first_name,
    c.last_name,
    c.country,
    o.order_id,
    o.order_date,
    o.order_status
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;

-- LEFT JOIN — all rows from left table, matching rows from right (NULL if no match)
SELECT 
    c.customer_id,
    c.first_name,
    c.last_name,
    COUNT(o.order_id) AS order_count  -- Will be NULL (shown as 0 with COALESCE) if no orders
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name;

-- Full join example with all tables
SELECT 
    c.first_name || ' ' || c.last_name AS customer_name,
    c.country,
    o.order_id,
    o.order_date,
    p.product_name,
    p.category,
    oi.quantity,
    oi.unit_price,
    oi.discount_pct,
    ROUND(oi.unit_price * oi.quantity * (1 - oi.discount_pct/100), 2) AS line_total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
ORDER BY o.order_date, o.order_id;
```

-----

## 5.6 Computed Columns and CASE Statements

```sql
-- Computed columns
SELECT 
    order_id,
    quantity,
    unit_price,
    discount_pct,
    unit_price * quantity AS gross_amount,
    unit_price * quantity * (discount_pct/100) AS discount_amount,
    unit_price * quantity * (1 - discount_pct/100) AS net_amount
FROM order_items;

-- CASE statement — conditional logic
SELECT 
    customer_id,
    SUM(unit_price * quantity * (1 - discount_pct/100)) AS total_spend,
    CASE 
        WHEN SUM(unit_price * quantity * (1 - discount_pct/100)) >= 500 THEN 'VIP'
        WHEN SUM(unit_price * quantity * (1 - discount_pct/100)) >= 200 THEN 'Regular'
        WHEN SUM(unit_price * quantity * (1 - discount_pct/100)) >= 50  THEN 'Occasional'
        ELSE 'Low Value'
    END AS customer_segment
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY customer_id;
```

-----

<a name="part-6"></a>

# PART 6 — Intermediate SQL & Snowflake-Specific Functions

-----

## 6.1 Window Functions — Analytics Superpowers

Window functions perform calculations across a “window” of related rows without collapsing them (unlike GROUP BY). They are essential for advanced analytics.

```sql
-- ROW_NUMBER — assign sequential numbers within a group
SELECT 
    customer_id,
    order_id,
    order_date,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS order_sequence
FROM orders;
-- This tells you "this was customer X's 1st order, 2nd order, 3rd order..."

-- RANK and DENSE_RANK
SELECT 
    product_id,
    product_name,
    category,
    unit_price,
    RANK() OVER (PARTITION BY category ORDER BY unit_price DESC) AS price_rank,
    DENSE_RANK() OVER (PARTITION BY category ORDER BY unit_price DESC) AS dense_price_rank
FROM products;

-- Running total (cumulative sum)
SELECT 
    order_date,
    order_id,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY order_date ROWS UNBOUNDED PRECEDING) AS cumulative_revenue
FROM (
    SELECT 
        order_date,
        order_id,
        SUM(oi.unit_price * oi.quantity) AS daily_revenue
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY order_date, order_id
);

-- LAG and LEAD — compare to previous/next row
SELECT 
    order_date,
    SUM(oi.unit_price * oi.quantity) AS daily_revenue,
    LAG(SUM(oi.unit_price * oi.quantity), 1) OVER (ORDER BY order_date) AS prev_day_revenue,
    SUM(oi.unit_price * oi.quantity) - 
        LAG(SUM(oi.unit_price * oi.quantity), 1) OVER (ORDER BY order_date) AS day_over_day_change
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY order_date
ORDER BY order_date;

-- FIRST_VALUE and LAST_VALUE — get first/last value in window
SELECT 
    customer_id,
    order_id,
    order_date,
    FIRST_VALUE(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS first_order_date,
    LAST_VALUE(order_date) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date 
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS last_order_date
FROM orders;

-- NTILE — divide into N buckets (quartiles, deciles, etc.)
SELECT 
    customer_id,
    total_spend,
    NTILE(4) OVER (ORDER BY total_spend DESC) AS spend_quartile  -- 1=top 25%, 4=bottom 25%
FROM (
    SELECT customer_id, SUM(unit_price * quantity) AS total_spend
    FROM orders o JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY customer_id
);
```

-----

## 6.2 CTEs — Common Table Expressions

CTEs make complex queries readable by breaking them into named steps.

```sql
-- Without CTE (hard to read):
SELECT a.customer_id, a.total_spend, b.order_count
FROM (SELECT customer_id, SUM(unit_price * quantity) AS total_spend FROM orders o JOIN order_items oi ON o.order_id = oi.order_id GROUP BY customer_id) a
JOIN (SELECT customer_id, COUNT(*) AS order_count FROM orders GROUP BY customer_id) b ON a.customer_id = b.customer_id;

-- With CTEs (easy to read and debug):
WITH 
customer_spend AS (
    SELECT 
        o.customer_id,
        SUM(oi.unit_price * oi.quantity) AS total_spend,
        SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct/100)) AS net_spend
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    GROUP BY o.customer_id
),

customer_orders AS (
    SELECT 
        customer_id,
        COUNT(DISTINCT order_id) AS order_count,
        MIN(order_date) AS first_order_date,
        MAX(order_date) AS last_order_date,
        DATEDIFF('day', MIN(order_date), MAX(order_date)) AS customer_lifetime_days
    FROM orders
    GROUP BY customer_id
),

customer_categories AS (
    SELECT DISTINCT
        o.customer_id,
        LISTAGG(DISTINCT p.category, ', ') WITHIN GROUP (ORDER BY p.category) AS categories_purchased
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY o.customer_id
)

SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS customer_name,
    c.country,
    cs.total_spend,
    cs.net_spend,
    co.order_count,
    co.first_order_date,
    co.last_order_date,
    co.customer_lifetime_days,
    cc.categories_purchased,
    CASE 
        WHEN cs.net_spend >= 500 THEN 'VIP'
        WHEN cs.net_spend >= 200 THEN 'Regular'
        ELSE 'Occasional'
    END AS segment
FROM customers c
JOIN customer_spend cs ON c.customer_id = cs.customer_id
JOIN customer_orders co ON c.customer_id = co.customer_id
JOIN customer_categories cc ON c.customer_id = cc.customer_id
ORDER BY cs.net_spend DESC;
```

-----

## 6.3 Semi-Structured Data — Querying JSON

One of Snowflake’s most powerful features is native JSON support via the VARIANT type.

```sql
-- Create a table with JSON data
CREATE TABLE events (
    event_id    INTEGER,
    event_ts    TIMESTAMP_NTZ,
    raw_event   VARIANT    -- This column stores arbitrary JSON
);

-- Insert JSON data
INSERT INTO events (event_id, event_ts, raw_event)
SELECT 
    1, 
    CURRENT_TIMESTAMP(),
    PARSE_JSON('{
        "event_type": "purchase",
        "user_id": 12345,
        "session_id": "abc-xyz-123",
        "device": {"type": "mobile", "os": "iOS", "version": "17.1"},
        "cart": [
            {"product_id": 101, "quantity": 1, "price": 299.99},
            {"product_id": 105, "quantity": 2, "price": 59.99}
        ],
        "payment": {"method": "credit_card", "status": "approved"},
        "metadata": {"ip_address": "192.168.1.1", "country_code": "US"}
    }');

-- Querying JSON with colon notation
SELECT 
    event_id,
    raw_event:event_type::STRING AS event_type,           -- Extract as string
    raw_event:user_id::INTEGER AS user_id,                -- Extract as integer
    raw_event:device:type::STRING AS device_type,         -- Nested field
    raw_event:device:os::STRING AS device_os,             -- Nested field
    raw_event:payment:method::STRING AS payment_method,
    raw_event:metadata:country_code::STRING AS country
FROM events;

-- FLATTEN — explode arrays into rows
SELECT 
    event_id,
    raw_event:user_id::INTEGER AS user_id,
    f.value:product_id::INTEGER AS product_id,
    f.value:quantity::INTEGER AS quantity,
    f.value:price::DECIMAL(10,2) AS price
FROM events,
LATERAL FLATTEN(INPUT => raw_event:cart) f;
-- This turns the array of cart items into separate rows

-- Check/parse mixed JSON
SELECT 
    raw_event:event_type::STRING,
    ARRAY_SIZE(raw_event:cart) AS cart_item_count,
    raw_event:cart[0]:product_id::INTEGER AS first_product_id  -- Array index access
FROM events;
```

-----

## 6.4 Date and Time Functions

```sql
-- Current timestamps
SELECT CURRENT_DATE();           -- 2024-02-15
SELECT CURRENT_TIME();           -- 14:30:00.123
SELECT CURRENT_TIMESTAMP();      -- 2024-02-15 14:30:00.123
SELECT SYSDATE();                -- Alias for CURRENT_TIMESTAMP

-- Date parts
SELECT 
    order_date,
    YEAR(order_date)      AS order_year,
    MONTH(order_date)     AS order_month,
    DAY(order_date)       AS order_day,
    DAYOFWEEK(order_date) AS day_of_week,   -- 0=Sunday
    DAYNAME(order_date)   AS day_name,      -- Monday, Tuesday...
    MONTHNAME(order_date) AS month_name,    -- January, February...
    QUARTER(order_date)   AS order_quarter,
    WEEKOFYEAR(order_date) AS week_number,
    DATE_TRUNC('month', order_date) AS first_day_of_month,
    DATE_TRUNC('year', order_date)  AS first_day_of_year
FROM orders;

-- Date arithmetic
SELECT 
    order_date,
    DATEADD('day', 30, order_date) AS due_date_30d,
    DATEDIFF('day', order_date, CURRENT_DATE()) AS days_since_order,
    DATEDIFF('month', MIN(order_date), CURRENT_DATE()) AS months_in_business
FROM orders;

-- TO_DATE and formatting
SELECT 
    TO_DATE('2024-01-15', 'YYYY-MM-DD') AS parsed_date,
    TO_TIMESTAMP('2024-01-15 14:30:00', 'YYYY-MM-DD HH24:MI:SS') AS parsed_ts,
    TO_CHAR(CURRENT_DATE(), 'YYYY-MM-DD') AS formatted_date,
    TO_CHAR(CURRENT_DATE(), 'Month DD, YYYY') AS pretty_date;
```

-----

## 6.5 String Functions

```sql
SELECT 
    -- Concatenation
    first_name || ' ' || last_name AS full_name,        -- || operator
    CONCAT(first_name, ' ', last_name) AS full_name2,    -- CONCAT function
    
    -- Case manipulation
    UPPER(email) AS email_upper,
    LOWER(email) AS email_lower,
    INITCAP(city) AS city_proper,    -- Title Case
    
    -- String inspection
    LENGTH(email) AS email_length,
    POSITION('@' IN email) AS at_sign_position,
    
    -- Extraction
    LEFT(email, POSITION('@' IN email) - 1) AS email_username,
    SUBSTRING(email, POSITION('@' IN email) + 1) AS email_domain,
    SPLIT_PART(email, '@', 1) AS email_username2,       -- Split by delimiter
    SPLIT_PART(email, '@', 2) AS email_domain2,
    
    -- Trimming
    TRIM('  hello  ') AS trimmed,
    LTRIM('  hello  ') AS left_trimmed,
    RTRIM('  hello  ') AS right_trimmed,
    
    -- Replacement
    REPLACE(email, '.com', '.org') AS modified_email,
    REGEXP_REPLACE(phone, '[^0-9]', '') AS digits_only  -- Remove non-digits
    
FROM customers;
```

-----

## 6.6 Views — Saved Queries

```sql
-- Regular view — always runs against latest data
CREATE OR REPLACE VIEW analytics.customer_summary AS
WITH customer_spend AS (
    SELECT 
        o.customer_id,
        SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct/100)) AS lifetime_value,
        COUNT(DISTINCT o.order_id) AS total_orders,
        MIN(o.order_date) AS first_order_date,
        MAX(o.order_date) AS last_order_date
    FROM ecommerce_analytics.raw.orders o
    JOIN ecommerce_analytics.raw.order_items oi ON o.order_id = oi.order_id
    GROUP BY o.customer_id
)
SELECT 
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    c.country,
    cs.lifetime_value,
    cs.total_orders,
    cs.first_order_date,
    cs.last_order_date,
    DATEDIFF('day', cs.last_order_date, CURRENT_DATE()) AS days_since_last_order
FROM ecommerce_analytics.raw.customers c
LEFT JOIN customer_spend cs ON c.customer_id = cs.customer_id;

-- Query the view like a table
SELECT * FROM analytics.customer_summary WHERE country = 'USA';

-- Materialized view — pre-computed and stored, auto-refreshed
CREATE MATERIALIZED VIEW analytics.daily_revenue_mv AS
SELECT 
    DATE_TRUNC('day', o.order_date) AS revenue_date,
    o.shipping_country AS country,
    p.category,
    COUNT(DISTINCT o.order_id) AS order_count,
    SUM(oi.unit_price * oi.quantity) AS gross_revenue,
    SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct/100)) AS net_revenue
FROM ecommerce_analytics.raw.orders o
JOIN ecommerce_analytics.raw.order_items oi ON o.order_id = oi.order_id
JOIN ecommerce_analytics.raw.products p ON oi.product_id = p.product_id
GROUP BY 1, 2, 3;
-- Materialized views are automatically kept up-to-date when base tables change
```

-----

<a name="part-7"></a>

# PART 7 — Data Loading & Integration (ETL/ELT)

-----

## 7.1 Understanding Stages

A **Stage** in Snowflake is a named location where data files are stored before being loaded. There are three types:

```
Internal Stages (files stored in Snowflake's own storage):
  - User Stage: each user has one automatically (@~)
  - Table Stage: each table has one (@%table_name)
  - Named Internal Stage: you create and name it

External Stages (files in your cloud storage):
  - S3 Bucket
  - Azure Blob Storage
  - GCP Cloud Storage
```

-----

## 7.2 Loading from Files — COPY INTO

```sql
-- Create a named stage pointing to an S3 bucket
CREATE STAGE raw.s3_sales_data
    URL = 's3://my-company-data/sales/'
    CREDENTIALS = (AWS_KEY_ID = 'xxx' AWS_SECRET_KEY = 'yyy')
    FILE_FORMAT = (TYPE = 'CSV' FIELD_DELIMITER = ',' SKIP_HEADER = 1);

-- Or using IAM role (preferred for production)
CREATE STAGE raw.s3_sales_data
    URL = 's3://my-company-data/sales/'
    STORAGE_INTEGRATION = my_s3_integration
    FILE_FORMAT = (TYPE = 'CSV');

-- List files in stage
LIST @raw.s3_sales_data;

-- Load CSV files into a table
COPY INTO raw.orders
FROM @raw.s3_sales_data/orders/
FILE_FORMAT = (
    TYPE = 'CSV'
    FIELD_DELIMITER = ','
    SKIP_HEADER = 1
    FIELD_OPTIONALLY_ENCLOSED_BY = '"'
    NULL_IF = ('NULL', 'null', '')
    EMPTY_FIELD_AS_NULL = TRUE
    DATE_FORMAT = 'YYYY-MM-DD'
    TIMESTAMP_FORMAT = 'YYYY-MM-DD HH24:MI:SS'
)
ON_ERROR = 'CONTINUE'   -- Options: ABORT_STATEMENT, CONTINUE, SKIP_FILE
PURGE = FALSE;          -- Don't delete source files after loading

-- Load JSON files
COPY INTO raw.events (event_id, event_ts, raw_event)
FROM (
    SELECT 
        $1:event_id::INTEGER,
        $1:timestamp::TIMESTAMP_NTZ,
        $1
    FROM @raw.s3_events_stage/
)
FILE_FORMAT = (TYPE = 'JSON');

-- Check what happened
SELECT * FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
    TABLE_NAME => 'ORDERS',
    START_TIME => DATEADD('hours', -1, CURRENT_TIMESTAMP())
));
```

-----

## 7.3 Continuous Loading with Snowpipe

Snowpipe automatically loads new files as soon as they arrive in cloud storage, without you needing to schedule COPY INTO manually.

```sql
-- Create a pipe that auto-ingests from a stage
CREATE PIPE raw.orders_pipe
    AUTO_INGEST = TRUE  -- Triggered by S3 event notifications
AS
COPY INTO raw.orders
FROM @raw.s3_sales_data/orders/
FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);

-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('raw.orders_pipe');

-- View ingestion history
SELECT * FROM TABLE(INFORMATION_SCHEMA.PIPE_USAGE_HISTORY(
    DATE_RANGE_START => DATEADD('day', -1, CURRENT_DATE()),
    PIPE_NAME => 'RAW.ORDERS_PIPE'
));
```

-----

## 7.4 Streams — Change Data Capture (CDC)

A **Stream** tracks changes to a table (INSERTs, UPDATEs, DELETEs). This is how you build incremental ETL pipelines — only process new/changed data, not everything from scratch.

```sql
-- Create a stream on the orders table
CREATE STREAM raw.orders_stream ON TABLE raw.orders;

-- The stream captures all changes since it was last consumed
-- When you query it, you see what changed:
SELECT * FROM raw.orders_stream;
-- Returns rows with special columns:
-- METADATA$ACTION: INSERT or DELETE
-- METADATA$ISUPDATE: TRUE if this is the new version of an updated row
-- METADATA$ROW_ID: Unique row identifier

-- Process changes: load new/updated orders into the analytics table
INSERT INTO analytics.orders_fact
SELECT 
    order_id,
    customer_id,
    order_date,
    order_status,
    shipping_country,
    CURRENT_TIMESTAMP() AS processed_at
FROM raw.orders_stream
WHERE METADATA$ACTION = 'INSERT'
  AND METADATA$ISUPDATE = FALSE;
-- Note: consuming the stream advances its offset — those rows won't appear again
```

-----

## 7.5 Tasks — Scheduled Jobs

A **Task** runs SQL or a stored procedure on a schedule, replacing the need for external schedulers like cron.

```sql
-- Create a task that runs every hour
CREATE TASK staging.process_new_orders
    WAREHOUSE = etl_wh
    SCHEDULE = '60 MINUTE'
AS
INSERT INTO analytics.orders_processed
SELECT 
    s.*,
    c.country AS customer_country,
    CURRENT_TIMESTAMP() AS processed_at
FROM raw.orders_stream s
JOIN raw.customers c ON s.customer_id = c.customer_id
WHERE s.METADATA$ACTION = 'INSERT';

-- Tasks start suspended — you must resume them
ALTER TASK staging.process_new_orders RESUME;

-- Create a cron-based task
CREATE TASK staging.daily_aggregation
    WAREHOUSE = analytics_wh
    SCHEDULE = 'USING CRON 0 6 * * * UTC'  -- Run at 6am UTC every day
AS
CALL staging.build_daily_metrics();

-- Task dependencies (DAGs)
CREATE TASK staging.step2
    WAREHOUSE = etl_wh
    AFTER staging.step1    -- Only runs after step1 completes
AS
...;
```

-----

## 7.6 MERGE — Upsert Pattern

MERGE is essential for incremental loads where you need to insert new rows and update existing ones.

```sql
-- MERGE: Update if exists, Insert if new (UPSERT)
MERGE INTO analytics.customers_dim AS target
USING raw.customers_stream AS source
ON target.customer_id = source.customer_id

WHEN MATCHED AND source.METADATA$ACTION = 'INSERT' THEN
    UPDATE SET 
        target.first_name = source.first_name,
        target.last_name = source.last_name,
        target.email = source.email,
        target.city = source.city,
        target.country = source.country,
        target.updated_at = CURRENT_TIMESTAMP()

WHEN NOT MATCHED AND source.METADATA$ACTION = 'INSERT' THEN
    INSERT (customer_id, first_name, last_name, email, city, country, created_at, updated_at)
    VALUES (
        source.customer_id, source.first_name, source.last_name,
        source.email, source.city, source.country,
        CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP()
    )

WHEN MATCHED AND source.METADATA$ACTION = 'DELETE' THEN
    DELETE;
```

-----

<a name="part-8"></a>

# PART 8 — Data Modeling & Warehousing Patterns

-----

## 8.1 Why Data Modeling Matters

Raw data from operational systems is designed for fast writes, not analysis. Data modeling transforms it into structures optimized for analytical queries.

Without modeling, analysts write complex joins every time. With modeling, they query clean, pre-joined tables.

-----

## 8.2 Star Schema — The Foundation

The **Star Schema** is the most common data warehouse pattern. It consists of:

- **Fact Tables:** Measurements and events (orders, transactions, clicks)
- **Dimension Tables:** Context about the facts (who, what, when, where)

The schema gets its name because it looks like a star — fact table in the center, dimension tables radiating out.

```
                    ┌─────────────┐
                    │  dim_date   │
                    │  date_key   │
                    │  year       │
                    │  month      │
                    │  quarter    │
                    │  day_name   │
                    └──────┬──────┘
                           │
┌─────────────┐     ┌──────┴────────┐     ┌─────────────┐
│ dim_customer│     │  fact_orders  │     │  dim_product│
│ customer_key├─────┤  order_key PK ├─────┤  product_key│
│ name        │     │  customer_key │     │  name       │
│ country     │     │  product_key  │     │  category   │
│ segment     │     │  date_key     │     │  brand      │
│ email       │     │  quantity     │     │  cost_price │
└─────────────┘     │  unit_price   │     └─────────────┘
                    │  discount_pct │
                    │  net_amount   │
                    └───────────────┘
```

### Building the Star Schema

```sql
USE SCHEMA ecommerce_analytics.analytics;

-- Dimension: Date (extremely important — pre-populate with all dates)
CREATE TABLE dim_date AS
SELECT 
    TO_NUMBER(TO_CHAR(d.date_val, 'YYYYMMDD')) AS date_key,
    d.date_val AS full_date,
    YEAR(d.date_val) AS year,
    QUARTER(d.date_val) AS quarter,
    MONTH(d.date_val) AS month_num,
    MONTHNAME(d.date_val) AS month_name,
    DAY(d.date_val) AS day_of_month,
    DAYOFWEEK(d.date_val) AS day_of_week,
    DAYNAME(d.date_val) AS day_name,
    WEEKOFYEAR(d.date_val) AS week_of_year,
    CASE WHEN DAYOFWEEK(d.date_val) IN (0, 6) THEN FALSE ELSE TRUE END AS is_weekday,
    CASE WHEN MONTH(d.date_val) IN (12, 1, 2) THEN 'Winter'
         WHEN MONTH(d.date_val) IN (3, 4, 5)  THEN 'Spring'
         WHEN MONTH(d.date_val) IN (6, 7, 8)  THEN 'Summer'
         ELSE 'Autumn' END AS season
FROM (
    SELECT DATEADD('day', SEQ4(), '2020-01-01') AS date_val
    FROM TABLE(GENERATOR(ROWCOUNT => 3653))  -- 10 years of dates
) d;

-- Dimension: Customers (Slowly Changing Dimension Type 2)
CREATE TABLE dim_customer (
    customer_key        INTEGER AUTOINCREMENT PRIMARY KEY,
    customer_id         INTEGER NOT NULL,       -- Source system ID
    first_name          VARCHAR(100),
    last_name           VARCHAR(100),
    email               VARCHAR(255),
    country             VARCHAR(50),
    city                VARCHAR(100),
    customer_segment    VARCHAR(50),
    -- SCD Type 2 fields
    effective_from      DATE NOT NULL,
    effective_to        DATE,                   -- NULL means current record
    is_current          BOOLEAN DEFAULT TRUE
);

-- Dimension: Products
CREATE TABLE dim_product AS
SELECT 
    product_id      AS product_key,
    product_name,
    category,
    subcategory,
    brand,
    unit_price,
    cost_price,
    ROUND((unit_price - cost_price) / NULLIF(unit_price, 0) * 100, 1) AS margin_pct
FROM ecommerce_analytics.raw.products;

-- Fact: Orders
CREATE TABLE fact_orders AS
SELECT 
    oi.order_item_id        AS order_item_key,
    o.order_id,
    o.customer_id           AS customer_key,
    oi.product_id           AS product_key,
    TO_NUMBER(TO_CHAR(o.order_date, 'YYYYMMDD')) AS date_key,
    o.order_status,
    o.shipping_country,
    oi.quantity,
    oi.unit_price,
    oi.discount_pct,
    oi.unit_price * oi.quantity AS gross_amount,
    oi.unit_price * oi.quantity * (oi.discount_pct / 100) AS discount_amount,
    oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100) AS net_amount,
    p.cost_price * oi.quantity AS cost_amount,
    (oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) - (p.cost_price * oi.quantity) AS gross_profit
FROM ecommerce_analytics.raw.orders o
JOIN ecommerce_analytics.raw.order_items oi ON o.order_id = oi.order_id
JOIN ecommerce_analytics.raw.products p ON oi.product_id = p.product_id;

-- Now analysts can easily query the star schema
SELECT 
    dd.year,
    dd.month_name,
    dc.country,
    dp.category,
    COUNT(DISTINCT fo.order_id)     AS orders,
    SUM(fo.quantity)                AS units_sold,
    SUM(fo.net_amount)              AS net_revenue,
    SUM(fo.gross_profit)            AS gross_profit,
    ROUND(AVG(fo.gross_profit / NULLIF(fo.net_amount, 0)) * 100, 1) AS avg_margin_pct
FROM fact_orders fo
JOIN dim_date dd ON fo.date_key = dd.date_key
JOIN dim_customer dc ON fo.customer_key = dc.customer_id
JOIN dim_product dp ON fo.product_key = dp.product_key
WHERE dd.year = 2024
GROUP BY dd.year, dd.month_num, dd.month_name, dc.country, dp.category
ORDER BY dd.month_num, net_revenue DESC;
```

-----

## 8.3 Slowly Changing Dimensions (SCD)

A dimension attribute that changes over time (e.g., a customer moves to a new city, a product changes its category) requires special handling.

**SCD Type 1:** Just overwrite. No history. Good when history doesn’t matter.

**SCD Type 2:** Keep history by adding new rows. This is the most common.

```sql
-- SCD Type 2 example: Customer moves from New York to Chicago
-- Before: Alice, New York, effective_from=2020-01-01, effective_to=NULL, is_current=TRUE
-- After: Close old record, insert new record

-- Step 1: Close the old record
UPDATE dim_customer
SET 
    effective_to = CURRENT_DATE() - 1,
    is_current = FALSE
WHERE customer_id = 1 AND is_current = TRUE;

-- Step 2: Insert new record
INSERT INTO dim_customer (customer_id, first_name, last_name, email, country, city, effective_from, effective_to, is_current)
VALUES (1, 'Alice', 'Smith', 'alice@email.com', 'USA', 'Chicago', CURRENT_DATE(), NULL, TRUE);

-- Now you can query point-in-time: what was Alice's city when she made her Jan 2024 order?
SELECT dc.city, fo.net_amount
FROM fact_orders fo
JOIN dim_customer dc ON fo.customer_key = dc.customer_id
    AND fo.order_date BETWEEN dc.effective_from AND COALESCE(dc.effective_to, '9999-12-31')
WHERE dc.customer_id = 1;
```

-----

<a name="part-9"></a>

# PART 9 — Advanced Snowflake Features

-----

## 9.1 Time Travel

Snowflake automatically retains historical versions of your data. You can query, clone, or restore data from any point in the past.

```sql
-- Default data retention: 1 day (Standard), up to 90 days (Enterprise)
-- Configure retention on a table
ALTER TABLE raw.orders SET DATA_RETENTION_TIME_IN_DAYS = 14;

-- Query data as it was at a specific time
SELECT * FROM raw.orders AT (TIMESTAMP => '2024-02-01 12:00:00'::TIMESTAMP_NTZ);

-- Query data as it was N seconds ago
SELECT * FROM raw.orders AT (OFFSET => -3600);  -- 1 hour ago

-- Query data before a specific statement ran
-- First, get a statement ID
SELECT LAST_QUERY_ID();  -- or check query history

SELECT * FROM raw.orders BEFORE (STATEMENT => '019c6b5b-0001-...');

-- PRACTICAL USE: Recover accidentally deleted data!
-- Someone ran: DELETE FROM orders WHERE order_status = 'completed' (oops, meant a subset)
-- Recover it:
INSERT INTO raw.orders
SELECT * FROM raw.orders BEFORE (STATEMENT => 'bad_statement_id_here')
WHERE order_id NOT IN (SELECT order_id FROM raw.orders);

-- CLONE at a point in time (zero-copy!)
CREATE TABLE orders_before_accident CLONE raw.orders
    AT (OFFSET => -1800);  -- 30 minutes ago
```

-----

## 9.2 Zero-Copy Cloning

Cloning creates an independent copy of a database object without physically copying data. It’s instantaneous regardless of size.

```sql
-- Clone a table (instant, no extra storage cost until data diverges)
CREATE TABLE raw.orders_backup CLONE raw.orders;

-- Clone an entire schema (clones all tables in it!)
CREATE SCHEMA analytics_backup CLONE analytics;

-- Clone an entire database
CREATE DATABASE ecommerce_analytics_dev CLONE ecommerce_analytics;
-- AMAZING FOR DEV/TEST: Developers get a full copy of prod data instantly
-- No extra storage cost until they start making changes
-- Changes in dev don't affect production

-- Clone with time travel: create a dev environment from yesterday's state
CREATE DATABASE dev_testing CLONE ecommerce_analytics
    AT (OFFSET => -86400);  -- 24 hours ago
```

-----

## 9.3 Stored Procedures and User-Defined Functions

```sql
-- Scalar UDF — returns one value per row
CREATE OR REPLACE FUNCTION calc_net_amount(
    unit_price FLOAT,
    quantity INTEGER,
    discount_pct FLOAT
)
RETURNS FLOAT
AS $$
    unit_price * quantity * (1 - discount_pct / 100)
$$;

-- Use it in a query
SELECT 
    order_item_id,
    calc_net_amount(unit_price, quantity, discount_pct) AS net_amount
FROM raw.order_items;

-- Table UDF — returns multiple rows
CREATE OR REPLACE FUNCTION get_customer_orders(cust_id INTEGER)
RETURNS TABLE (order_id INTEGER, order_date DATE, total_amount FLOAT)
AS $$
    SELECT order_id, order_date, SUM(unit_price * quantity) AS total_amount
    FROM raw.orders o
    JOIN raw.order_items oi ON o.order_id = oi.order_id
    WHERE o.customer_id = cust_id
    GROUP BY order_id, order_date
$$;

SELECT * FROM TABLE(get_customer_orders(1));

-- JavaScript UDF — for complex logic
CREATE OR REPLACE FUNCTION parse_email_domain(email VARCHAR)
RETURNS VARCHAR
LANGUAGE JAVASCRIPT
AS $$
    if (!EMAIL) return null;
    var parts = EMAIL.split('@');
    return parts.length === 2 ? parts[1].toLowerCase() : null;
$$;

-- Stored Procedure — run complex multi-statement logic
CREATE OR REPLACE PROCEDURE staging.refresh_customer_segments()
RETURNS STRING
LANGUAGE SQL
AS $$
BEGIN
    -- Calculate segments
    CREATE OR REPLACE TEMPORARY TABLE temp_segments AS
    SELECT 
        customer_id,
        SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) AS lifetime_value,
        CASE 
            WHEN SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) >= 500 THEN 'VIP'
            WHEN SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) >= 200 THEN 'Regular'
            ELSE 'Occasional'
        END AS segment
    FROM raw.orders o
    JOIN raw.order_items oi ON o.order_id = oi.order_id
    GROUP BY customer_id;

    -- Update customers
    UPDATE raw.customers c
    SET lifetime_value = ts.lifetime_value
    FROM temp_segments ts
    WHERE c.customer_id = ts.customer_id;

    RETURN 'Customer segments refreshed: ' || (SELECT COUNT(*) FROM temp_segments) || ' customers updated';
END;
$$;

-- Call the procedure
CALL staging.refresh_customer_segments();
```

-----

## 9.4 Snowflake Scripting (Control Flow)

```sql
DECLARE
    v_customer_count INTEGER;
    v_threshold INTEGER := 100;
BEGIN
    SELECT COUNT(*) INTO v_customer_count FROM raw.customers;
    
    IF (v_customer_count > v_threshold) THEN
        RETURN 'Large customer base: ' || v_customer_count;
    ELSEIF (v_customer_count > 10) THEN
        RETURN 'Medium customer base: ' || v_customer_count;
    ELSE
        RETURN 'Small customer base: ' || v_customer_count;
    END IF;
END;

-- Loop example
DECLARE
    v_tables ARRAY;
    v_table VARCHAR;
    i INTEGER := 0;
BEGIN
    v_tables := ARRAY_CONSTRUCT('orders', 'customers', 'products');
    
    FOR i IN 0 TO ARRAY_SIZE(v_tables) - 1 DO
        v_table := v_tables[i]::VARCHAR;
        -- Process each table
        EXECUTE IMMEDIATE 'ANALYZE TABLE raw.' || v_table;
    END FOR;
    
    RETURN 'Done';
END;
```

-----

## 9.5 Dynamic Tables

Dynamic Tables are a newer, simpler alternative to Streams + Tasks for building data pipelines.

```sql
-- A dynamic table automatically maintains itself based on a query
-- No streams, no tasks, no stored procedures needed
CREATE DYNAMIC TABLE analytics.customer_360
    TARGET_LAG = '1 hour'    -- How stale is acceptable (1 hour = refresh every hour max)
    WAREHOUSE = etl_wh
AS
SELECT 
    c.customer_id,
    c.first_name || ' ' || c.last_name AS full_name,
    c.email,
    c.country,
    COUNT(DISTINCT o.order_id) AS total_orders,
    SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) AS lifetime_value,
    MAX(o.order_date) AS last_order_date,
    DATEDIFF('day', MAX(o.order_date), CURRENT_DATE()) AS days_since_last_order,
    CASE 
        WHEN DATEDIFF('day', MAX(o.order_date), CURRENT_DATE()) <= 30 THEN 'Active'
        WHEN DATEDIFF('day', MAX(o.order_date), CURRENT_DATE()) <= 90 THEN 'At Risk'
        ELSE 'Churned'
    END AS churn_status
FROM raw.customers c
LEFT JOIN raw.orders o ON c.customer_id = o.customer_id
LEFT JOIN raw.order_items oi ON o.order_id = oi.order_id
GROUP BY c.customer_id, c.first_name, c.last_name, c.email, c.country;

-- Snowflake automatically keeps this fresh! No pipeline code needed.
SELECT * FROM analytics.customer_360 WHERE churn_status = 'At Risk';
```

-----

## 9.6 Data Sharing

One of Snowflake’s most unique features — share live data with other organizations (or departments) without copying it.

```sql
-- Provider side: share data with another Snowflake account
CREATE SHARE ecommerce_public_data;

GRANT USAGE ON DATABASE ecommerce_analytics TO SHARE ecommerce_public_data;
GRANT USAGE ON SCHEMA ecommerce_analytics.analytics TO SHARE ecommerce_public_data;
GRANT SELECT ON TABLE analytics.customer_summary TO SHARE ecommerce_public_data;

-- Add the consumer account (another company's Snowflake account)
ALTER SHARE ecommerce_public_data ADD ACCOUNT = 'CONSUMER_ACCOUNT_LOCATOR';

-- Consumer side: access the shared data
CREATE DATABASE shared_ecommerce_data FROM SHARE provider_account.ecommerce_public_data;
SELECT * FROM shared_ecommerce_data.analytics.customer_summary;
-- Consumer queries always see live, real-time data — no copying, no synchronization
```

-----

<a name="part-10"></a>

# PART 10 — Performance Optimization

-----

## 10.1 Understanding Query Performance

Before optimizing, diagnose. Use the Query Profile to understand what’s happening.

```sql
-- Run a query, then check its profile
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION(RESULT_LIMIT => 10));

-- Get details about a specific query
SELECT * FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY())
WHERE QUERY_TEXT ILIKE '%fact_orders%'
ORDER BY START_TIME DESC
LIMIT 20;
```

In Snowsight, after running a query, click the query ID to see the **Query Profile** — a visual execution plan showing each operator, rows processed, time spent, and bytes scanned.

Key things to look for:

- **Bytes scanned:** High = not enough partition pruning
- **Spilling to disk:** Query exceeded memory, using disk = very slow
- **Join explosions:** Joins producing far more rows than expected

-----

## 10.2 Partition Pruning Optimization

```sql
-- Bad: No filter on the natural sort order column
-- Scans ALL partitions
SELECT product_id, SUM(quantity) 
FROM fact_orders 
WHERE product_key = 101;

-- Better: Filter on the date column (which orders are naturally sorted by)
-- Prunes most partitions
SELECT product_id, SUM(quantity) 
FROM fact_orders 
WHERE date_key BETWEEN 20240101 AND 20240131  -- Prunes most partitions
  AND product_key = 101;

-- Check partition pruning efficiency
SELECT 
    query_id,
    query_text,
    partitions_scanned,
    partitions_total,
    ROUND(partitions_scanned::FLOAT / NULLIF(partitions_total, 0) * 100, 1) AS scan_pct
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE table_name = 'FACT_ORDERS'
ORDER BY start_time DESC
LIMIT 20;

-- Add a clustering key if queries frequently filter on non-date columns
ALTER TABLE fact_orders CLUSTER BY (date_key, shipping_country);
-- Automatic Clustering will then reorganize the data in the background
```

-----

## 10.3 Query Writing Best Practices

```sql
-- 1. Select only the columns you need (not SELECT *)
-- BAD: SELECT * FROM fact_orders;  -- reads all columns
-- GOOD:
SELECT order_id, date_key, net_amount FROM fact_orders;

-- 2. Filter early, filter often
-- BAD: Join everything, then filter
SELECT * 
FROM fact_orders fo
JOIN dim_customer dc ON fo.customer_key = dc.customer_id
WHERE dc.country = 'USA';

-- GOOD: Filter in subquery before joining
SELECT * 
FROM fact_orders fo
JOIN (SELECT * FROM dim_customer WHERE country = 'USA') dc 
    ON fo.customer_key = dc.customer_id;

-- 3. Avoid functions on filtered columns (they prevent pruning)
-- BAD: YEAR() function prevents date partition pruning
WHERE YEAR(order_date) = 2024;

-- GOOD: Range filter works with partition pruning
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';

-- 4. Use LIMIT when exploring
SELECT * FROM fact_orders LIMIT 1000;

-- 5. Avoid DISTINCT when possible (forces full sort)
-- If you need distinct values, it's sometimes faster to GROUP BY

-- 6. Pre-filter subqueries
WITH recent_orders AS (
    SELECT * FROM fact_orders 
    WHERE date_key >= 20240101  -- Filter here, not in outer query
)
SELECT customer_key, SUM(net_amount) 
FROM recent_orders
GROUP BY customer_key;
```

-----

## 10.4 Virtual Warehouse Optimization

```sql
-- Multi-cluster warehouses: automatically add clusters under high concurrency
CREATE WAREHOUSE reporting_wh
    WAREHOUSE_SIZE = 'MEDIUM'
    MAX_CLUSTER_COUNT = 5      -- Can scale up to 5 clusters automatically
    MIN_CLUSTER_COUNT = 1      -- Scales back to 1 when load drops
    SCALING_POLICY = 'STANDARD'  -- STANDARD or ECONOMY
    AUTO_SUSPEND = 120
    AUTO_RESUME = TRUE;

-- Separate workloads into different warehouses (prevent conflicts)
-- etl_wh: large, for data loading and transformations
-- analytics_wh: medium, for BI tools and analysts  
-- ml_wh: xlarge, for data science queries
-- adhoc_wh: xsmall, for quick exploratory queries

-- Warehouse-level Resource Monitors (prevent bill surprises)
CREATE RESOURCE MONITOR monthly_limit
    WITH CREDIT_QUOTA = 500  -- 500 credits per month
    FREQUENCY = MONTHLY
    START_TIMESTAMP = IMMEDIATELY
    TRIGGERS
        ON 80 PERCENT DO NOTIFY   -- Email at 80%
        ON 100 PERCENT DO SUSPEND;-- Suspend all warehouses at 100%

ALTER WAREHOUSE analytics_wh SET RESOURCE_MONITOR = monthly_limit;
```

-----

## 10.5 Result Cache and Materialized Views

```sql
-- Check if a query hit the result cache
SELECT 
    query_id,
    query_text,
    execution_status,
    is_client_generated_statement,
    -- If execution_time = 0 and bytes_scanned = 0, it was from result cache
    total_elapsed_time,
    bytes_scanned
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE query_text ILIKE '%daily_revenue%'
ORDER BY start_time DESC;

-- Force re-execution (bypass cache) — useful for testing
ALTER SESSION SET USE_CACHED_RESULT = FALSE;

-- Materialized view for expensive recurring aggregations
CREATE MATERIALIZED VIEW analytics.monthly_revenue_by_region_mv AS
SELECT 
    DATE_TRUNC('month', o.order_date) AS month,
    o.shipping_country,
    p.category,
    COUNT(DISTINCT o.order_id) AS orders,
    SUM(oi.net_amount) AS revenue
FROM raw.orders o
JOIN raw.order_items oi ON o.order_id = oi.order_id
JOIN raw.products p ON oi.product_id = p.product_id
GROUP BY 1, 2, 3;
-- Snowflake maintains this automatically and rewrites queries to use it
```

-----

<a name="part-11"></a>

# PART 11 — Security, Governance & Administration

-----

## 11.1 Role-Based Access Control (RBAC)

Snowflake’s security is built on roles. Instead of giving permissions directly to users, you grant permissions to roles, then roles to users.

```
ACCOUNTADMIN (superuser — use sparingly)
├── SYSADMIN (manages databases and warehouses)
│   ├── ANALYST_ROLE (can read analytics schema)
│   ├── ETL_ROLE (can load and transform data)
│   └── DEVELOPER_ROLE (can create objects in dev)
└── SECURITYADMIN (manages users and roles)
    └── USERADMIN (creates users)
```

```sql
-- Use SECURITYADMIN role for access control
USE ROLE SECURITYADMIN;

-- Create roles
CREATE ROLE analyst_role;
CREATE ROLE etl_role;
CREATE ROLE data_engineer_role;

-- Grant privileges to roles
GRANT USAGE ON DATABASE ecommerce_analytics TO ROLE analyst_role;
GRANT USAGE ON SCHEMA ecommerce_analytics.analytics TO ROLE analyst_role;
GRANT SELECT ON ALL TABLES IN SCHEMA ecommerce_analytics.analytics TO ROLE analyst_role;
GRANT SELECT ON FUTURE TABLES IN SCHEMA ecommerce_analytics.analytics TO ROLE analyst_role;

-- Grant warehouse access
GRANT USAGE ON WAREHOUSE analytics_wh TO ROLE analyst_role;

-- ETL role gets more permissions
GRANT USAGE ON DATABASE ecommerce_analytics TO ROLE etl_role;
GRANT USAGE ON SCHEMA ecommerce_analytics.raw TO ROLE etl_role;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA ecommerce_analytics.raw TO ROLE etl_role;

-- Create a user
USE ROLE USERADMIN;
CREATE USER analyst_jane
    PASSWORD = 'TempPassword123!'
    DEFAULT_ROLE = analyst_role
    DEFAULT_WAREHOUSE = analytics_wh
    MUST_CHANGE_PASSWORD = TRUE
    EMAIL = 'jane@company.com';

-- Grant role to user
GRANT ROLE analyst_role TO USER analyst_jane;

-- Role hierarchy (give data_engineer_role all of analyst and etl permissions too)
GRANT ROLE analyst_role TO ROLE data_engineer_role;
GRANT ROLE etl_role TO ROLE data_engineer_role;
```

-----

## 11.2 Column-Level Security and Data Masking

```sql
-- Create a masking policy to hide sensitive data
CREATE MASKING POLICY mask_email
AS (val STRING) RETURNS STRING ->
    CASE
        WHEN CURRENT_ROLE() IN ('ANALYST_ROLE', 'REPORTING_ROLE') 
            THEN REGEXP_REPLACE(val, '(.+)@(.+)', '****@\\2')  -- Hide username
        WHEN CURRENT_ROLE() = 'DATA_ENGINEER_ROLE' 
            THEN val    -- Show full email
        ELSE '***MASKED***'
    END;

-- Apply the masking policy to a column
ALTER TABLE raw.customers 
MODIFY COLUMN email SET MASKING POLICY mask_email;

-- Now different roles see different things:
-- analyst_jane sees: ****@email.com
-- data_engineer sees: alice@email.com
-- others see: ***MASKED***
```

-----

## 11.3 Row Access Policies

```sql
-- Allow users to only see rows relevant to their region
CREATE ROW ACCESS POLICY region_access_policy
AS (shipping_country STRING) RETURNS BOOLEAN ->
    CASE
        WHEN CURRENT_ROLE() = 'GLOBAL_ANALYST' THEN TRUE  -- See everything
        WHEN CURRENT_ROLE() = 'US_ANALYST' 
            AND shipping_country = 'USA' THEN TRUE
        WHEN CURRENT_ROLE() = 'EU_ANALYST' 
            AND shipping_country IN ('UK', 'Germany', 'France', 'Spain') THEN TRUE
        ELSE FALSE
    END;

ALTER TABLE raw.orders ADD ROW ACCESS POLICY region_access_policy ON (shipping_country);
```

-----

## 11.4 Account Usage — Monitoring and Auditing

```sql
-- Who ran what queries?
SELECT 
    user_name,
    query_text,
    database_name,
    schema_name,
    warehouse_name,
    execution_status,
    total_elapsed_time / 1000 AS execution_seconds,
    credits_used_cloud_services,
    start_time
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
ORDER BY credits_used_cloud_services DESC NULLS LAST
LIMIT 50;

-- Credit consumption by warehouse
SELECT 
    warehouse_name,
    SUM(credits_used) AS total_credits,
    COUNT(*) AS query_count,
    AVG(credits_used) AS avg_credits_per_query
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE start_time >= DATEADD('month', -1, CURRENT_TIMESTAMP())
GROUP BY warehouse_name
ORDER BY total_credits DESC;

-- Storage costs
SELECT 
    TABLE_SCHEMA,
    TABLE_NAME,
    ACTIVE_BYTES / (1024*1024*1024) AS active_gb,
    TIME_TRAVEL_BYTES / (1024*1024*1024) AS time_travel_gb,
    FAILSAFE_BYTES / (1024*1024*1024) AS failsafe_gb
FROM SNOWFLAKE.ACCOUNT_USAGE.TABLE_STORAGE_METRICS
ORDER BY active_gb DESC;
```

-----

<a name="part-12"></a>

# PART 12 — Real Project: End-to-End E-Commerce Analytics Platform

Let’s build a complete, production-grade data platform from scratch.

-----

## Project Overview

**Business:** An e-commerce company selling products globally  
**Goal:** Build a modern data platform that enables:

1. Daily revenue reporting by region, product category, and channel
1. Customer segmentation and churn prediction features
1. Product performance analytics
1. Real-time order pipeline with quality monitoring

**Architecture:**

```
[Source Systems] → [Raw Layer] → [Staging Layer] → [Analytics Layer] → [BI/Reports]
     OLTP DB            Snowflake     Transformations      Star Schema       Dashboards
     APIs               Raw Tables    (dbt or SQL)         Materialized Views  
     Files              Streams                            Dynamic Tables
```

-----

## Phase 1: Setup and Raw Layer

```sql
-- ============================================================
-- SETUP: Create the full project structure
-- ============================================================

-- Create database
CREATE DATABASE ecomm_platform;

-- Create schemas
CREATE SCHEMA ecomm_platform.raw;       -- Raw ingested data
CREATE SCHEMA ecomm_platform.staging;   -- Cleaned data
CREATE SCHEMA ecomm_platform.analytics; -- Business layer
CREATE SCHEMA ecomm_platform.meta;      -- Metadata and audit

-- Create warehouses
CREATE WAREHOUSE ingestion_wh
    WAREHOUSE_SIZE = 'SMALL'
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    COMMENT = 'For data ingestion and loading';

CREATE WAREHOUSE transform_wh
    WAREHOUSE_SIZE = 'MEDIUM'
    AUTO_SUSPEND = 120
    AUTO_RESUME = TRUE
    COMMENT = 'For ELT transformations';

CREATE WAREHOUSE reporting_wh
    WAREHOUSE_SIZE = 'SMALL'
    MAX_CLUSTER_COUNT = 3
    AUTO_SUSPEND = 60
    AUTO_RESUME = TRUE
    COMMENT = 'For BI tools and analyst queries';
```

-----

## Phase 2: Raw Layer Tables with Metadata

```sql
USE SCHEMA ecomm_platform.raw;
USE WAREHOUSE ingestion_wh;

-- Add metadata columns to every raw table (essential for debugging)
CREATE TABLE raw_orders (
    -- Source data
    order_id            VARCHAR(50),
    customer_id         VARCHAR(50),
    order_date          VARCHAR(50),    -- Store as VARCHAR first, parse later
    order_status        VARCHAR(50),
    channel             VARCHAR(50),    -- web, mobile, marketplace
    shipping_country    VARCHAR(50),
    payment_method      VARCHAR(50),
    
    -- Raw ingestion metadata
    _source_file        VARCHAR(500),   -- Which file this came from
    _ingested_at        TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _row_hash           VARCHAR(64),    -- Hash for deduplication
    _is_duplicate       BOOLEAN DEFAULT FALSE
);

CREATE TABLE raw_customers (
    customer_id         VARCHAR(50),
    first_name          VARCHAR(200),
    last_name           VARCHAR(200),
    email               VARCHAR(500),
    phone               VARCHAR(50),
    date_of_birth       VARCHAR(50),
    country             VARCHAR(100),
    city                VARCHAR(100),
    registration_date   VARCHAR(50),
    marketing_consent   VARCHAR(10),
    raw_attributes      VARIANT,        -- Any extra JSON attributes
    
    _source_file        VARCHAR(500),
    _ingested_at        TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    _row_hash           VARCHAR(64)
);

-- Data quality audit table
CREATE TABLE meta.data_quality_log (
    log_id              INTEGER AUTOINCREMENT,
    check_timestamp     TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    source_table        VARCHAR(200),
    check_name          VARCHAR(200),
    check_description   VARCHAR(1000),
    rows_checked        INTEGER,
    rows_failed         INTEGER,
    failure_pct         DECIMAL(5, 2),
    status              VARCHAR(20),    -- PASS, WARN, FAIL
    sample_failures     VARIANT         -- JSON with example bad rows
);
```

-----

## Phase 3: Staging — Data Quality and Cleansing

```sql
USE SCHEMA ecomm_platform.staging;
USE WAREHOUSE transform_wh;

-- Cleaned orders with proper types and quality flags
CREATE TABLE stg_orders AS
SELECT
    -- Parse and cast all fields properly
    order_id::INTEGER AS order_id,
    customer_id::INTEGER AS customer_id,
    TRY_TO_DATE(order_date, 'YYYY-MM-DD') AS order_date,
    LOWER(TRIM(order_status)) AS order_status,
    LOWER(TRIM(channel)) AS channel,
    UPPER(TRIM(shipping_country)) AS shipping_country,
    LOWER(TRIM(payment_method)) AS payment_method,
    
    -- Data quality flags
    CASE WHEN TRY_TO_DATE(order_date, 'YYYY-MM-DD') IS NULL THEN TRUE ELSE FALSE END AS has_invalid_date,
    CASE WHEN customer_id IS NULL THEN TRUE ELSE FALSE END AS has_null_customer,
    CASE WHEN order_status NOT IN ('completed', 'shipped', 'processing', 'cancelled', 'refunded') 
         THEN TRUE ELSE FALSE END AS has_invalid_status,
    
    -- Metadata
    _source_file,
    _ingested_at,
    CURRENT_TIMESTAMP() AS _transformed_at
FROM raw_orders
WHERE _is_duplicate = FALSE;

-- Cleaned customers
CREATE TABLE stg_customers AS
SELECT
    customer_id::INTEGER AS customer_id,
    TRIM(INITCAP(first_name)) AS first_name,
    TRIM(INITCAP(last_name)) AS last_name,
    LOWER(TRIM(email)) AS email,
    REGEXP_REPLACE(phone, '[^0-9+]', '') AS phone_cleaned,
    TRY_TO_DATE(date_of_birth, 'YYYY-MM-DD') AS date_of_birth,
    DATEDIFF('year', TRY_TO_DATE(date_of_birth, 'YYYY-MM-DD'), CURRENT_DATE()) AS age,
    UPPER(TRIM(country)) AS country,
    TRIM(INITCAP(city)) AS city,
    TRY_TO_DATE(registration_date, 'YYYY-MM-DD') AS registration_date,
    CASE WHEN UPPER(marketing_consent) IN ('YES', 'TRUE', '1', 'Y') THEN TRUE ELSE FALSE END AS marketing_consent,
    
    -- Validation flags
    CASE WHEN NOT REGEXP_LIKE(email, '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$') 
         THEN TRUE ELSE FALSE END AS has_invalid_email,
    CASE WHEN date_of_birth IS NOT NULL 
         AND (age < 13 OR age > 120) 
         THEN TRUE ELSE FALSE END AS has_suspicious_age,
    
    _source_file,
    _ingested_at,
    CURRENT_TIMESTAMP() AS _transformed_at
FROM raw_customers;

-- Data quality check procedure
CREATE OR REPLACE PROCEDURE meta.run_data_quality_checks()
RETURNS STRING
LANGUAGE SQL
AS $$
DECLARE
    v_result VARCHAR;
BEGIN
    -- Check 1: Orders with invalid dates
    INSERT INTO meta.data_quality_log (source_table, check_name, check_description, rows_checked, rows_failed, failure_pct, status)
    SELECT 
        'stg_orders',
        'invalid_order_date',
        'Orders where order_date could not be parsed',
        COUNT(*),
        SUM(CASE WHEN has_invalid_date THEN 1 ELSE 0 END),
        ROUND(SUM(CASE WHEN has_invalid_date THEN 1 ELSE 0 END) / COUNT(*) * 100, 2),
        CASE WHEN SUM(CASE WHEN has_invalid_date THEN 1 ELSE 0 END) / COUNT(*) > 0.01 THEN 'FAIL'
             WHEN SUM(CASE WHEN has_invalid_date THEN 1 ELSE 0 END) > 0 THEN 'WARN'
             ELSE 'PASS' END
    FROM staging.stg_orders;
    
    -- Check 2: Orphan orders (no matching customer)
    INSERT INTO meta.data_quality_log (source_table, check_name, check_description, rows_checked, rows_failed, failure_pct, status)
    SELECT 
        'stg_orders',
        'orphan_orders',
        'Orders with no matching customer_id in customers table',
        COUNT(*),
        SUM(CASE WHEN c.customer_id IS NULL THEN 1 ELSE 0 END),
        ROUND(SUM(CASE WHEN c.customer_id IS NULL THEN 1 ELSE 0 END) / COUNT(*) * 100, 2),
        CASE WHEN SUM(CASE WHEN c.customer_id IS NULL THEN 1 ELSE 0 END) / COUNT(*) > 0.005 THEN 'FAIL'
             ELSE 'PASS' END
    FROM staging.stg_orders o
    LEFT JOIN staging.stg_customers c ON o.customer_id = c.customer_id;

    RETURN 'Data quality checks complete. See meta.data_quality_log for results.';
END;
$$;
```

-----

## Phase 4: Analytics Layer — Business Logic

```sql
USE SCHEMA ecomm_platform.analytics;

-- Customer 360 Dynamic Table (auto-maintained)
CREATE DYNAMIC TABLE customer_360
    TARGET_LAG = '2 hours'
    WAREHOUSE = transform_wh
AS
WITH customer_orders AS (
    SELECT 
        o.customer_id,
        COUNT(DISTINCT o.order_id) AS total_orders,
        SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) AS lifetime_value,
        MIN(o.order_date) AS first_order_date,
        MAX(o.order_date) AS last_order_date,
        DATEDIFF('day', MAX(o.order_date), CURRENT_DATE()) AS days_since_last_order,
        SUM(CASE WHEN o.order_date >= DATEADD('day', -30, CURRENT_DATE()) 
                 THEN oi.unit_price * oi.quantity ELSE 0 END) AS revenue_last_30d,
        COUNT(DISTINCT o.channel) AS channels_used,
        LISTAGG(DISTINCT p.category, '|') WITHIN GROUP (ORDER BY p.category) AS categories_purchased
    FROM ecomm_platform.staging.stg_orders o
    JOIN ecomm_platform.raw.order_items oi ON o.order_id = oi.order_id
    JOIN ecomm_platform.raw.products p ON oi.product_id = p.product_id
    WHERE o.order_status NOT IN ('cancelled')
    GROUP BY o.customer_id
)
SELECT 
    c.customer_id,
    c.first_name,
    c.last_name,
    c.email,
    c.country,
    c.city,
    c.age,
    c.registration_date,
    DATEDIFF('day', c.registration_date, CURRENT_DATE()) AS days_since_registration,
    COALESCE(co.total_orders, 0) AS total_orders,
    COALESCE(co.lifetime_value, 0) AS lifetime_value,
    co.first_order_date,
    co.last_order_date,
    COALESCE(co.days_since_last_order, 9999) AS days_since_last_order,
    COALESCE(co.revenue_last_30d, 0) AS revenue_last_30d,
    co.categories_purchased,
    -- Churn risk score (simple rule-based)
    CASE 
        WHEN co.days_since_last_order <= 30 THEN 'Active'
        WHEN co.days_since_last_order <= 60 THEN 'At Risk - Low'
        WHEN co.days_since_last_order <= 90 THEN 'At Risk - High'
        WHEN co.days_since_last_order IS NULL THEN 'Never Ordered'
        ELSE 'Churned'
    END AS churn_status,
    -- Customer value tier
    CASE 
        WHEN co.lifetime_value >= 1000 THEN 'Platinum'
        WHEN co.lifetime_value >= 500 THEN 'Gold'
        WHEN co.lifetime_value >= 200 THEN 'Silver'
        WHEN co.lifetime_value > 0 THEN 'Bronze'
        ELSE 'Prospect'
    END AS value_tier
FROM ecomm_platform.staging.stg_customers c
LEFT JOIN customer_orders co ON c.customer_id = co.customer_id;

-- Product Performance View
CREATE OR REPLACE VIEW product_performance AS
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    p.brand,
    p.unit_price,
    p.cost_price,
    COUNT(DISTINCT oi.order_id) AS orders_containing_product,
    SUM(oi.quantity) AS total_units_sold,
    SUM(oi.unit_price * oi.quantity) AS gross_revenue,
    SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) AS net_revenue,
    SUM((oi.unit_price - p.cost_price) * oi.quantity) AS gross_profit,
    ROUND(AVG(oi.discount_pct), 1) AS avg_discount_pct,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    RANK() OVER (PARTITION BY p.category ORDER BY SUM(oi.unit_price * oi.quantity) DESC) AS revenue_rank_in_category
FROM ecomm_platform.raw.products p
LEFT JOIN ecomm_platform.raw.order_items oi ON p.product_id = oi.product_id
LEFT JOIN ecomm_platform.staging.stg_orders o ON oi.order_id = o.order_id
    AND o.order_status NOT IN ('cancelled', 'refunded')
GROUP BY p.product_id, p.product_name, p.category, p.brand, p.unit_price, p.cost_price;

-- Executive KPI Dashboard View
CREATE OR REPLACE VIEW executive_kpis AS
SELECT 
    DATE_TRUNC('month', o.order_date) AS month,
    COUNT(DISTINCT o.order_id) AS total_orders,
    COUNT(DISTINCT o.customer_id) AS active_customers,
    SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) AS net_revenue,
    SUM((oi.unit_price - p.cost_price) * oi.quantity) AS gross_profit,
    ROUND(SUM((oi.unit_price - p.cost_price) * oi.quantity) / 
          NULLIF(SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)), 0) * 100, 1) AS gp_margin_pct,
    SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) / 
        NULLIF(COUNT(DISTINCT o.order_id), 0) AS avg_order_value,
    -- Month over month growth
    LAG(SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100))) 
        OVER (ORDER BY DATE_TRUNC('month', o.order_date)) AS prev_month_revenue,
    ROUND((SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100)) - 
           LAG(SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100))) 
               OVER (ORDER BY DATE_TRUNC('month', o.order_date))) /
          NULLIF(LAG(SUM(oi.unit_price * oi.quantity * (1 - oi.discount_pct / 100))) 
                     OVER (ORDER BY DATE_TRUNC('month', o.order_date)), 0) * 100, 1) AS mom_revenue_growth_pct
FROM ecomm_platform.staging.stg_orders o
JOIN ecomm_platform.raw.order_items oi ON o.order_id = oi.order_id
JOIN ecomm_platform.raw.products p ON oi.product_id = p.product_id
WHERE o.order_status NOT IN ('cancelled', 'refunded')
GROUP BY DATE_TRUNC('month', o.order_date)
ORDER BY month;
```

-----

## Phase 5: Automated Pipeline

```sql
-- Stream to detect new orders
CREATE STREAM raw.new_orders_stream ON TABLE raw.raw_orders;

-- Task 1: Process raw to staging every 30 minutes
CREATE TASK staging.task_process_raw_orders
    WAREHOUSE = transform_wh
    SCHEDULE = '30 MINUTE'
WHEN SYSTEM$STREAM_HAS_DATA('ecomm_platform.raw.new_orders_stream')
AS
    MERGE INTO ecomm_platform.staging.stg_orders AS target
    USING (
        SELECT
            order_id::INTEGER AS order_id,
            customer_id::INTEGER AS customer_id,
            TRY_TO_DATE(order_date, 'YYYY-MM-DD') AS order_date,
            LOWER(TRIM(order_status)) AS order_status,
            LOWER(TRIM(channel)) AS channel,
            UPPER(TRIM(shipping_country)) AS shipping_country,
            _ingested_at
        FROM ecomm_platform.raw.new_orders_stream
        WHERE METADATA$ACTION = 'INSERT'
    ) AS source
    ON target.order_id = source.order_id
    WHEN NOT MATCHED THEN
    INSERT (order_id, customer_id, order_date, order_status, channel, shipping_country, _ingested_at)
    VALUES (source.order_id, source.customer_id, source.order_date, source.order_status, 
            source.channel, source.shipping_country, source._ingested_at);

-- Task 2: Run data quality checks after staging
CREATE TASK meta.task_data_quality
    WAREHOUSE = transform_wh
    AFTER staging.task_process_raw_orders
AS
    CALL meta.run_data_quality_checks();

-- Resume tasks (they start suspended)
ALTER TASK meta.task_data_quality RESUME;
ALTER TASK staging.task_process_raw_orders RESUME;
```

-----

<a name="part-13"></a>

# PART 13 — Data Engineering Patterns & Best Practices

-----

## 13.1 dbt — The Modern Transformation Framework

**dbt (data build tool)** is the industry-standard way to write Snowflake transformations. Instead of ad-hoc SQL scripts, dbt lets you:

- Write transformations as SQL SELECT statements
- Automatically create tables or views
- Test data quality
- Document your data model
- Track lineage (see which tables depend on which)

### dbt Project Structure

```
my_dbt_project/
├── models/
│   ├── raw/          (sources — reference existing raw tables)  
│   ├── staging/      (stg_ prefix — clean and rename)
│   ├── intermediate/ (int_ prefix — business logic)
│   └── marts/        (final tables for BI consumption)
├── tests/            (data quality tests)
├── macros/           (reusable SQL functions)
└── dbt_project.yml
```

### Example dbt Models

```sql
-- models/staging/stg_orders.sql
-- dbt creates this as a view by default
SELECT
    order_id::INTEGER AS order_id,
    customer_id::INTEGER AS customer_id,
    TRY_TO_DATE(order_date) AS order_date,
    LOWER(TRIM(order_status)) AS order_status,
    LOWER(TRIM(channel)) AS channel,
    UPPER(shipping_country) AS shipping_country
FROM {{ source('raw', 'raw_orders') }}   -- Reference source tables
WHERE _is_duplicate = FALSE

-- models/marts/fact_orders.sql
-- Configure as a table (materialized)
{{ config(materialized='table', cluster_by=['order_date']) }}

WITH orders AS (
    SELECT * FROM {{ ref('stg_orders') }}          -- ref() handles dependencies
),
order_items AS (
    SELECT * FROM {{ ref('stg_order_items') }}
),
products AS (
    SELECT * FROM {{ ref('stg_products') }}
)
SELECT 
    oi.order_item_id,
    o.order_id,
    o.customer_id,
    p.product_id,
    p.category,
    o.order_date,
    o.shipping_country,
    oi.quantity,
    oi.unit_price,
    oi.discount_pct,
    oi.unit_price * oi.quantity * (1 - oi.discount_pct/100) AS net_amount
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
```

-----

## 13.2 Design Principles for Production Pipelines

**1. Idempotency:** Running a pipeline multiple times should produce the same result.

```sql
-- Bad: Appends duplicates on re-run
INSERT INTO analytics.daily_orders
SELECT * FROM staging.stg_orders WHERE order_date = CURRENT_DATE();

-- Good: Truncate and reload (idempotent)
DELETE FROM analytics.daily_orders WHERE order_date = CURRENT_DATE();
INSERT INTO analytics.daily_orders
SELECT * FROM staging.stg_orders WHERE order_date = CURRENT_DATE();

-- Better: MERGE (handles both cases)
MERGE INTO analytics.daily_orders AS target
USING (SELECT * FROM staging.stg_orders WHERE order_date = CURRENT_DATE()) AS source
ON target.order_id = source.order_id
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT ...;
```

**2. Incremental Loading:** Don’t reload what hasn’t changed.

```sql
-- Track the high watermark (last processed timestamp)
INSERT INTO meta.pipeline_watermarks (pipeline_name, last_processed_ts)
VALUES ('orders_pipeline', CURRENT_TIMESTAMP())
ON CONFLICT (pipeline_name) DO UPDATE SET last_processed_ts = CURRENT_TIMESTAMP();

-- Process only new data
INSERT INTO analytics.fact_orders
SELECT * FROM staging.stg_orders
WHERE _ingested_at > (
    SELECT last_processed_ts 
    FROM meta.pipeline_watermarks 
    WHERE pipeline_name = 'orders_pipeline'
);
```

**3. Error Handling and Dead Letter Queues:**

```sql
-- Quarantine bad rows instead of failing the entire pipeline
CREATE TABLE raw.quarantine_orders LIKE raw.raw_orders;
ALTER TABLE raw.quarantine_orders ADD COLUMN rejection_reason VARCHAR(500);

-- Route invalid rows to quarantine
INSERT INTO raw.quarantine_orders
SELECT *, 'Invalid date format: ' || order_date AS rejection_reason
FROM raw.raw_orders
WHERE TRY_TO_DATE(order_date, 'YYYY-MM-DD') IS NULL;

-- Process only valid rows
INSERT INTO staging.stg_orders
SELECT * FROM raw.raw_orders
WHERE TRY_TO_DATE(order_date, 'YYYY-MM-DD') IS NOT NULL;
```

-----

## 13.3 Cost Optimization Strategies

```sql
-- 1. Right-size your warehouses
-- Use Query Profile to see if queries are memory-bound (spilling to disk → upsize)
-- or if small warehouse runs them fine (downsize)

-- 2. Suspend immediately after batch jobs
ALTER TASK etl_task SET AFTER_TASK_TIMEOUT_MS = 0;  -- Suspend warehouse after task

-- 3. Use resource monitors
CREATE RESOURCE MONITOR dev_budget
    WITH CREDIT_QUOTA = 50
    FREQUENCY = MONTHLY
    TRIGGERS ON 100 PERCENT DO SUSPEND_IMMEDIATE;

-- 4. Use transient tables for intermediate work (no Time Travel = cheaper storage)
CREATE TRANSIENT TABLE staging.temp_calculation (
    customer_id INTEGER,
    metric FLOAT
);
-- Transient tables: no Time Travel, no Fail-Safe = ~40% storage savings

-- 5. Check expensive queries regularly
SELECT 
    user_name,
    warehouse_name,
    ROUND(total_elapsed_time / 60000, 1) AS minutes,
    ROUND(credits_used_cloud_services, 4) AS cloud_credits,
    LEFT(query_text, 200) AS query_preview
FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
WHERE start_time >= DATEADD('day', -7, CURRENT_TIMESTAMP())
    AND total_elapsed_time > 60000  -- Queries longer than 1 minute
ORDER BY total_elapsed_time DESC
LIMIT 20;
```

-----

<a name="part-14"></a>

# PART 14 — Snowflake Ecosystem & Integrations

-----

## 14.1 Key Tool Integrations

**BI Tools (connect via JDBC/ODBC or native connector):**

- Tableau, Looker, Power BI, Metabase, Superset
- Connect using Snowflake’s native connectors for best performance

**Orchestration:**

- Apache Airflow — trigger Snowflake SQL via SnowflakeOperator
- Prefect / Dagster — modern Python-based workflow tools
- dbt Cloud — runs dbt models on a schedule with a UI

**Data Ingestion (move data into Snowflake):**

- Fivetran — 200+ pre-built connectors, zero-code
- Airbyte — open-source alternative
- Stitch — simple, connector-based
- Kafka + Snowflake connector — real-time streaming

**ML/Python:**

- Snowpark — write Python/Java/Scala that runs in Snowflake (no data movement)
- Snowflake ML — built-in ML model training and feature store

-----

## 14.2 Snowpark Python Example

```python
# Install: pip install snowflake-snowpark-python

from snowflake.snowpark import Session
from snowflake.snowpark.functions import col, sum as sum_, avg, round as round_

# Connect
connection_parameters = {
    "account": "your_account",
    "user": "your_user",
    "password": "your_password",
    "role": "data_engineer_role",
    "warehouse": "transform_wh",
    "database": "ecomm_platform",
    "schema": "staging"
}

session = Session.builder.configs(connection_parameters).create()

# Read Snowflake data as a DataFrame (lazy evaluation — query runs in Snowflake)
orders_df = session.table("stg_orders")
order_items_df = session.table("raw.order_items")
products_df = session.table("raw.products")

# Build a transformation pipeline using Python (compiled to SQL, runs in Snowflake)
revenue_by_category = (
    orders_df
    .join(order_items_df, orders_df["order_id"] == order_items_df["order_id"])
    .join(products_df, order_items_df["product_id"] == products_df["product_id"])
    .filter(col("order_status") != "cancelled")
    .group_by(col("category"), col("order_date"))
    .agg(
        sum_(col("unit_price") * col("quantity")).alias("gross_revenue"),
        avg(col("unit_price")).alias("avg_price")
    )
    .sort(col("order_date").desc())
)

# Write back to Snowflake
revenue_by_category.write.save_as_table("analytics.revenue_by_category", mode="overwrite")

print("Transformation complete!")
```

-----

## 14.3 Final Architecture Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    COMPLETE SNOWFLAKE PLATFORM                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  DATA SOURCES          RAW LAYER         STAGING LAYER           │
│  ─────────────         ─────────         ───────────────         │
│  ERP/CRM DB    ──→     raw tables   →    stg_* tables           │
│  APIs          ──→     (VARIANT,         (typed, cleaned,        │
│  Files (S3)    ──→      raw JSON)         quality flagged)       │
│  Kafka Events  ──→     Snowpipe/COPY                             │
│                                                                   │
│  ANALYTICS LAYER       CONSUMPTION        GOVERNANCE             │
│  ────────────────      ───────────        ──────────             │
│  fact_* tables  →      BI Tools           RBAC                   │
│  dim_* tables   →      Tableau            Masking Policies       │
│  Dynamic Tables →      Power BI           Row Access             │
│  Views          →      Python/ML          Data Quality Logs      │
│  Materialized   →      dbt               Time Travel             │
│   Views         →      Snowpark           Account Usage          │
│                                                                   │
│  ORCHESTRATION         OPTIMIZATION       COST CONTROL           │
│  ─────────────         ────────────       ────────────           │
│  Tasks/DAGs            Clustering         Resource Monitors       │
│  Streams               Partitioning       Warehouse Sizing        │
│  Airflow               Result Cache       Transient Tables        │
│  dbt Cloud             Materialized Views  Usage Monitoring       │
└─────────────────────────────────────────────────────────────────┘
```

-----

## Key Concepts Quick Reference

|Concept          |What It Is                                  |When to Use                                     |
|-----------------|--------------------------------------------|------------------------------------------------|
|Virtual Warehouse|Compute cluster for running queries         |Separate workloads by type/priority             |
|Micro-partition  |Unit of physical storage (50-500MB)         |Understanding why clustering matters            |
|Time Travel      |Query/restore historical data               |Recovery, auditing, analysis                    |
|Zero-Copy Clone  |Instant copy without duplicating data       |Dev/test environments                           |
|Stream           |Change data capture on a table              |Incremental ELT pipelines                       |
|Task             |Scheduled SQL execution in Snowflake        |Replacing cron jobs                             |
|Dynamic Table    |Auto-maintained derived table               |Simplifying pipeline code                       |
|Snowpipe         |Continuous file ingestion                   |Real-time or near-real-time loading             |
|Stage            |File storage location for loading           |Any file-based ingestion                        |
|VARIANT          |Column for storing JSON/semi-structured data|API responses, event logs                       |
|MERGE            |Upsert (insert + update in one statement)   |Incremental loads                               |
|Materialized View|Pre-computed, stored query result           |Expensive recurring aggregations                |
|Masking Policy   |Dynamically hide sensitive column values    |PII protection                                  |
|Row Access Policy|Dynamically filter rows by role             |Multi-tenant, regional access                   |
|Resource Monitor |Credit spending limit and alerts            |Cost control                                    |
|Clustering Key   |Physical data organization hint             |Improving pruning on frequently filtered columns|
|Result Cache     |24-hour cache of query results              |Repeated dashboard queries                      |
|Snowpark         |Run Python/Java/Scala in Snowflake          |ML feature engineering, complex transforms      |
|Data Share       |Share live data with other accounts         |Partner data, marketplace                       |

-----

## What to Learn Next

After mastering this guide, explore:

1. **dbt (data build tool)** — https://docs.getdbt.com — industry standard for Snowflake transformations
1. **Snowflake Cortex** — built-in LLM/AI functions for text analysis directly in SQL
1. **Snowflake ML** — train models, feature stores, ML pipelines inside Snowflake
1. **Apache Airflow** — orchestrate complex multi-step data pipelines
1. **Fivetran or Airbyte** — automated data ingestion from 200+ sources
1. **SnowPro Core Certification** — Snowflake’s official certification (excellent for career)

-----

*Guide version 1.0 — Built for zero-to-advanced Snowflake learners*
*All SQL examples are tested on Snowflake Enterprise Edition*