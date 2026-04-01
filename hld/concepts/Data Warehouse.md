## 1. What is a Data Warehouse?

A data warehouse is a centralized repository optimized for analytical queries (OLAP) rather than transactional operations (OLTP).

```
OLTP (Transactional):
  Database: PostgreSQL, MySQL
  Workload: INSERT, UPDATE, DELETE
  Queries: Simple, fast (ms)
  Schema: Normalized
  Use: Production applications

OLAP (Analytical):
  Database: Redshift, BigQuery, Snowflake
  Workload: SELECT with aggregations
  Queries: Complex, slow (seconds to minutes)
  Schema: Denormalized (star/snowflake)
  Use: Analytics, reporting, BI
```

**Key principle:** Separate analytical workloads from transactional workloads.

---

## 2. Core Concepts

### OLTP vs OLAP

```
OLTP (Online Transaction Processing):
  - Row-oriented storage
  - Optimized for writes
  - Normalized schema
  - Real-time queries
  - Small result sets
  
  Example: E-commerce checkout
    INSERT INTO orders (user_id, total) VALUES (123, 99.99)

OLAP (Online Analytical Processing):
  - Column-oriented storage
  - Optimized for reads
  - Denormalized schema
  - Batch queries
  - Large result sets
  
  Example: Sales analysis
    SELECT product, SUM(revenue) FROM sales GROUP BY product
```

### Columnar Storage

```
Row-oriented (OLTP):
  Row 1: [id=1, name="Alice", age=30, city="NYC"]
  Row 2: [id=2, name="Bob", age=25, city="LA"]
  
  Read entire row even if only need one column
  Good for: SELECT * WHERE id = 1

Column-oriented (OLAP):
  id: [1, 2]
  name: ["Alice", "Bob"]
  age: [30, 25]
  city: ["NYC", "LA"]
  
  Read only needed columns
  Good for: SELECT AVG(age) FROM users
  
Benefits:
  - Better compression (similar values together)
  - Faster aggregations (scan one column)
  - Lower I/O (read only needed columns)
```

### Star Schema

```
Fact Table (center):
  sales: {sale_id, date_id, product_id, store_id, quantity, revenue}
  
Dimension Tables (points):
  dates: {date_id, date, month, quarter, year}
  products: {product_id, name, category, brand}
  stores: {store_id, name, city, state}

Query:
  SELECT p.category, SUM(s.revenue)
  FROM sales s
  JOIN products p ON s.product_id = p.product_id
  GROUP BY p.category

Denormalized for fast queries
```

### Snowflake Schema

```
Fact Table:
  sales: {sale_id, date_id, product_id, store_id, quantity, revenue}

Dimension Tables (normalized):
  products: {product_id, name, category_id, brand_id}
  categories: {category_id, name}
  brands: {brand_id, name}

More normalized than star schema
Saves storage but requires more joins
```

---

## 3. ETL Pipeline

### Extract, Transform, Load

```
1. Extract:
   Pull data from source systems (databases, APIs, files)
   
2. Transform:
   Clean, aggregate, join, enrich data
   
3. Load:
   Load into data warehouse

Example:
  Extract:
    - Orders from PostgreSQL
    - Users from MongoDB
    - Events from Kafka
  
  Transform:
    - Join orders with users
    - Aggregate daily sales
    - Calculate metrics
  
  Load:
    - Load into Redshift
    - Update dimension tables
    - Append to fact tables
```

### ELT (Modern Approach)

```
1. Extract:
   Pull data from sources
   
2. Load:
   Load raw data into warehouse
   
3. Transform:
   Transform inside warehouse (SQL)

Benefits:
  - Leverage warehouse compute power
  - Keep raw data (reprocess if needed)
  - Faster initial load

Example:
  Extract: Orders from PostgreSQL
  Load: Load raw orders into Redshift
  Transform: CREATE TABLE daily_sales AS SELECT ...
```

---

## 4. Data Warehouse Options

### Amazon Redshift

```
Columnar storage
MPP (massively parallel processing)
SQL-based

Pros:
  ✅ Mature, well-documented
  ✅ Integrates with AWS
  ✅ Good performance
  
Cons:
  ❌ Requires cluster management
  ❌ Expensive for small workloads

Use case: AWS-native, large datasets
```

### Google BigQuery

```
Serverless, auto-scaling
Columnar storage
SQL-based

Pros:
  ✅ Serverless (no cluster management)
  ✅ Scales automatically
  ✅ Pay per query
  
Cons:
  ❌ Expensive for frequent queries
  ❌ Vendor lock-in

Use case: GCP-native, variable workloads
```

### Snowflake

```
Cloud-native, multi-cloud
Separates storage and compute
SQL-based

Pros:
  ✅ Multi-cloud (AWS, Azure, GCP)
  ✅ Scales compute independently
  ✅ Easy to use
  
Cons:
  ❌ Expensive
  ❌ Vendor lock-in

Use case: Multi-cloud, flexible scaling
```

### ClickHouse

```
Open-source, columnar
Very fast for aggregations
SQL-based

Pros:
  ✅ Open-source
  ✅ Very fast
  ✅ Self-hosted
  
Cons:
  ❌ Requires ops
  ❌ Limited ecosystem

Use case: Self-hosted, real-time analytics
```

---

## 5. Implementation Example

### Schema Design (Star Schema)

```sql
-- Fact table
CREATE TABLE sales (
    sale_id BIGINT,
    date_id INT,
    product_id INT,
    store_id INT,
    quantity INT,
    revenue DECIMAL(10, 2),
    PRIMARY KEY (sale_id)
);

-- Dimension tables
CREATE TABLE dates (
    date_id INT PRIMARY KEY,
    date DATE,
    day_of_week VARCHAR(10),
    month INT,
    quarter INT,
    year INT
);

CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(255),
    category VARCHAR(100),
    brand VARCHAR(100)
);

CREATE TABLE stores (
    store_id INT PRIMARY KEY,
    name VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(50)
);
```

### ETL Pipeline (Python)

```python
import psycopg2
from datetime import datetime

# Extract from OLTP database
def extract_orders():
    conn = psycopg2.connect("postgresql://oltp-db")
    cursor = conn.cursor()
    cursor.execute("""
        SELECT id, user_id, product_id, quantity, price, created_at
        FROM orders
        WHERE created_at >= CURRENT_DATE - INTERVAL '1 day'
    """)
    return cursor.fetchall()

# Transform
def transform_orders(orders):
    transformed = []
    for order in orders:
        sale_id, user_id, product_id, quantity, price, created_at = order
        transformed.append({
            'sale_id': sale_id,
            'date_id': int(created_at.strftime('%Y%m%d')),
            'product_id': product_id,
            'store_id': 1,  # Lookup from user
            'quantity': quantity,
            'revenue': quantity * price
        })
    return transformed

# Load into data warehouse
def load_sales(sales):
    conn = psycopg2.connect("postgresql://redshift")
    cursor = conn.cursor()
    for sale in sales:
        cursor.execute("""
            INSERT INTO sales (sale_id, date_id, product_id, store_id, quantity, revenue)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, sale.values())
    conn.commit()

# Run ETL
orders = extract_orders()
sales = transform_orders(orders)
load_sales(sales)
```

### Analytical Queries

```sql
-- Total revenue by category
SELECT p.category, SUM(s.revenue) AS total_revenue
FROM sales s
JOIN products p ON s.product_id = p.product_id
GROUP BY p.category
ORDER BY total_revenue DESC;

-- Monthly sales trend
SELECT d.year, d.month, SUM(s.revenue) AS monthly_revenue
FROM sales s
JOIN dates d ON s.date_id = d.date_id
GROUP BY d.year, d.month
ORDER BY d.year, d.month;

-- Top 10 products by revenue
SELECT p.name, SUM(s.revenue) AS total_revenue
FROM sales s
JOIN products p ON s.product_id = p.product_id
GROUP BY p.name
ORDER BY total_revenue DESC
LIMIT 10;

-- Year-over-year growth
SELECT 
    d.year,
    SUM(s.revenue) AS revenue,
    LAG(SUM(s.revenue)) OVER (ORDER BY d.year) AS prev_year_revenue,
    (SUM(s.revenue) - LAG(SUM(s.revenue)) OVER (ORDER BY d.year)) / 
        LAG(SUM(s.revenue)) OVER (ORDER BY d.year) * 100 AS growth_pct
FROM sales s
JOIN dates d ON s.date_id = d.date_id
GROUP BY d.year
ORDER BY d.year;
```

---

## 6. Optimization Techniques

### Partitioning

```sql
-- Partition by date
CREATE TABLE sales (
    sale_id BIGINT,
    date_id INT,
    product_id INT,
    revenue DECIMAL(10, 2)
)
PARTITION BY RANGE (date_id);

CREATE TABLE sales_2024_01 PARTITION OF sales
    FOR VALUES FROM (20240101) TO (20240201);

CREATE TABLE sales_2024_02 PARTITION OF sales
    FOR VALUES FROM (20240201) TO (20240301);

Benefits:
  - Query only relevant partitions
  - Faster queries (less data scanned)
  - Easier data management (drop old partitions)
```

### Materialized Views

```sql
-- Pre-compute aggregations
CREATE MATERIALIZED VIEW daily_sales AS
SELECT 
    date_id,
    product_id,
    SUM(quantity) AS total_quantity,
    SUM(revenue) AS total_revenue
FROM sales
GROUP BY date_id, product_id;

-- Refresh periodically
REFRESH MATERIALIZED VIEW daily_sales;

Benefits:
  - Faster queries (pre-computed)
  - Reduce compute cost
```

### Distribution Keys (Redshift)

```sql
-- Distribute by join key
CREATE TABLE sales (
    sale_id BIGINT,
    product_id INT,
    revenue DECIMAL(10, 2)
)
DISTKEY(product_id);

CREATE TABLE products (
    product_id INT,
    name VARCHAR(255)
)
DISTKEY(product_id);

-- Join is local (no data movement)
SELECT p.name, SUM(s.revenue)
FROM sales s
JOIN products p ON s.product_id = p.product_id
GROUP BY p.name;
```

### Sort Keys (Redshift)

```sql
-- Sort by frequently filtered column
CREATE TABLE sales (
    sale_id BIGINT,
    date_id INT,
    product_id INT,
    revenue DECIMAL(10, 2)
)
SORTKEY(date_id);

-- Query uses sort key (fast)
SELECT * FROM sales
WHERE date_id >= 20240101 AND date_id < 20240201;
```

---

## 7. Data Warehouse vs Data Lake

### Data Warehouse

```
Structured data
Schema-on-write
SQL queries
Expensive storage
Fast queries

Use case: Business intelligence, reporting
```

### Data Lake

```
Raw, unstructured data
Schema-on-read
Any processing (SQL, Spark, ML)
Cheap storage (S3)
Slower queries

Use case: Data science, ML, exploratory analysis
```

### Data Lakehouse (Hybrid)

```
Combines warehouse and lake
Structured + unstructured data
ACID transactions on data lake
Examples: Databricks, Delta Lake

Benefits:
  - Cheap storage (data lake)
  - Fast queries (warehouse)
  - Flexible (any data type)
```

---

## 8. Challenges

### Data freshness

```
Problem:
  ETL runs daily
  Data is stale (up to 24 hours old)

Solutions:
  - Real-time ETL (streaming)
  - Micro-batches (every 5 minutes)
  - Incremental loads (only new data)
```

### Query performance

```
Problem:
  Queries are slow (minutes)

Solutions:
  - Partitioning (scan less data)
  - Materialized views (pre-compute)
  - Indexing (if supported)
  - Distribution keys (co-locate data)
  - Query optimization (rewrite queries)
```

### Cost

```
Problem:
  Data warehouse is expensive

Solutions:
  - Partition and archive old data
  - Use serverless (pay per query)
  - Optimize queries (scan less data)
  - Use data lake for cold data
```

---

## 9. When to Use a Data Warehouse

### Good fit

```
✅ Analytical queries (aggregations, joins)
✅ Business intelligence and reporting
✅ Historical data analysis
✅ Structured data
✅ SQL-based queries

Examples:
  - Sales dashboards
  - Financial reporting
  - Customer analytics
  - Marketing attribution
```

### Poor fit

```
❌ Transactional workloads (use OLTP)
❌ Real-time queries (use cache or stream processing)
❌ Unstructured data (use data lake)
❌ Machine learning (use data lake or lakehouse)

Examples:
  - E-commerce checkout (use PostgreSQL)
  - Real-time dashboards (use Redis + stream processing)
  - Log analysis (use Elasticsearch)
```

---

## 10. Common Interview Questions + Answers

### Q: What is a data warehouse and when would you use it?

> "A data warehouse is a centralized repository optimized for analytical queries rather than transactional operations. It uses columnar storage and denormalized schemas like star or snowflake for fast aggregations. You'd use it for business intelligence, reporting, and historical analysis where you need to run complex queries with joins and aggregations across large datasets. It's separate from your production database to avoid impacting transactional performance."

### Q: What's the difference between OLTP and OLAP?

> "OLTP is for transactional workloads — lots of small, fast writes and reads, normalized schema, row-oriented storage. Think e-commerce checkout. OLAP is for analytical workloads — complex queries with aggregations, denormalized schema, columnar storage. Think sales dashboards. You use OLTP databases like PostgreSQL for production and OLAP databases like Redshift for analytics. Separating them prevents analytical queries from slowing down your application."

### Q: What's the difference between ETL and ELT?

> "ETL extracts data from sources, transforms it outside the warehouse, then loads it. ELT extracts data, loads it raw into the warehouse, then transforms it inside using SQL. ELT is the modern approach because data warehouses are powerful enough to handle transformations, you can leverage their compute, and you keep raw data for reprocessing. ETL is still used when you need to transform data before loading, like for data quality or privacy reasons."

### Q: How do you optimize data warehouse queries?

> "First, partition tables by frequently filtered columns like date to scan less data. Second, use materialized views to pre-compute common aggregations. Third, use distribution keys to co-locate joined tables on the same nodes. Fourth, use sort keys for range queries. Fifth, optimize queries to scan fewer columns and rows. Finally, monitor query plans to identify bottlenecks and adjust accordingly."

---

## 11. Quick Reference

```
What is a Data Warehouse?
  Centralized repository for analytical queries
  Optimized for OLAP (not OLTP)
  Columnar storage, denormalized schema

OLTP vs OLAP:
  OLTP: Transactional, row-oriented, normalized
  OLAP: Analytical, columnar, denormalized

Columnar storage:
  Store by column (not row)
  Better compression, faster aggregations
  Read only needed columns

Schema:
  Star: Fact table + dimension tables (denormalized)
  Snowflake: Normalized dimension tables

ETL vs ELT:
  ETL: Extract → Transform → Load
  ELT: Extract → Load → Transform (modern)

Popular options:
  - Redshift (AWS, mature)
  - BigQuery (GCP, serverless)
  - Snowflake (multi-cloud, flexible)
  - ClickHouse (open-source, fast)

Optimization:
  - Partitioning (scan less data)
  - Materialized views (pre-compute)
  - Distribution keys (co-locate data)
  - Sort keys (range queries)

Data Warehouse vs Data Lake:
  Warehouse: Structured, schema-on-write, fast queries
  Lake: Raw, schema-on-read, cheap storage
  Lakehouse: Hybrid (best of both)

When to use:
  ✅ Analytical queries
  ✅ Business intelligence
  ✅ Historical analysis
  ✅ Structured data
  
  ❌ Transactional workloads
  ❌ Real-time queries
  ❌ Unstructured data

Best practices:
  - Separate from OLTP database
  - Use star or snowflake schema
  - Partition by date
  - Use materialized views
  - Monitor query performance
  - Archive old data
```
