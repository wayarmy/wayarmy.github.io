---
layout: post
title: "Data Modeling Patterns - Mẫu thiết kế dữ liệu"
date: 2026-06-01
categories: [database]
tags: [data-modeling, normalization, star-schema, event-sourcing, cqrs]
---

## Mục lục
1. [Góc nhìn tổng quan - Kiến trúc sư dữ liệu](#overview)
2. [ER Diagrams - Sơ đồ thực thể quan hệ](#er-diagrams)
3. [Normalization (1NF → 3NF → BCNF)](#normalization)
4. [Denormalization - Khi nào phá chuẩn](#denormalization)
5. [Star Schema - Sơ đồ sao](#star-schema)
6. [Snowflake Schema](#snowflake)
7. [Event Sourcing - Lưu sự kiện thay vì trạng thái](#event-sourcing)
8. [CQRS - Tách đọc và ghi](#cqrs)
9. [Time-series Data Modeling](#time-series)
10. [Tổng kết và decision framework](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

Data modeling giống **thiết kế bố trí kho hàng**:
- **Normalization** = mỗi sản phẩm 1 vị trí duy nhất, không trùng lặp → tiết kiệm diện tích nhưng lấy hàng tốn thời gian
- **Denormalization** = đặt sẵn combo ở nhiều kệ → tốn diện tích nhưng lấy nhanh
- **Star Schema** = kệ trung tâm (fact) + các kệ phụ xung quanh (dimensions)
- **Event Sourcing** = sổ nhật ký giao dịch: không ghi "kho còn 50", mà ghi "nhập 100, xuất 30, xuất 20"
- **CQRS** = 2 quầy riêng: quầy nhập hàng (write) và quầy xuất hàng (read)

---

## 2. ER Diagrams {#er-diagrams}

```
ER Diagram = bản vẽ relationship giữa các entities

Entities: User, Order, Product, Category
Relationships:
  User 1───∞ Order      (1 user có nhiều orders)
  Order ∞───∞ Product   (via OrderItems junction table)
  Category 1───∞ Product (1 category có nhiều products)

Cardinality:
  1:1  (User ─── Profile)
  1:N  (User ─── Orders)
  M:N  (Students ─── Courses via Enrollment)
```

---

## 3. Normalization {#normalization}

```sql
-- 1NF: No repeating groups, atomic values
-- ❌ orders(id, items: "phone,laptop,tablet")
-- ✅ order_items(order_id, product_name, qty)

-- 2NF: No partial dependencies (all non-key depends on FULL PK)
-- ❌ order_items(order_id, product_id, product_name, qty)
--    product_name depends only on product_id, not full PK
-- ✅ Separate: products(id, name), order_items(order_id, product_id, qty)

-- 3NF: No transitive dependencies
-- ❌ employees(id, dept_id, dept_name, dept_manager)
--    dept_name depends on dept_id (transitive through id → dept_id → dept_name)
-- ✅ Separate: departments(id, name, manager_id)

-- BCNF: Every determinant is a candidate key
-- Stricter than 3NF, rarely needed in practice
```

---

## 4. Denormalization {#denormalization}

```sql
-- When to denormalize:
-- 1. Read-heavy workloads (90% reads)
-- 2. Complex JOINs hurting performance
-- 3. Reporting/Analytics queries
-- 4. Caching frequently computed values

-- Example: Store computed total in orders
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2);
-- Updated via trigger or application on item change

-- Example: Materialized view for dashboard
CREATE MATERIALIZED VIEW daily_sales AS
SELECT date_trunc('day', created_at) as day,
       COUNT(*) as order_count,
       SUM(total_amount) as revenue
FROM orders
GROUP BY 1;
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales;
```

---

## 5-6. Star Schema & Snowflake {#star-schema}

```
STAR SCHEMA (Data Warehouse):
                    ┌────────────┐
                    │ dim_date   │
                    │ date_key   │
                    │ day, month │
                    └─────┬──────┘
                          │
┌────────────┐    ┌───────┴───────┐    ┌────────────┐
│ dim_product│────│  fact_sales   │────│ dim_store  │
│ product_key│    │ date_key      │    │ store_key  │
│ name, cat  │    │ product_key   │    │ name, city │
└────────────┘    │ store_key     │    └────────────┘
                  │ customer_key  │
                  │ quantity      │
                  │ amount        │
                  └───────┬───────┘
                          │
                    ┌─────┴──────┐
                    │dim_customer│
                    │customer_key│
                    │ name, age  │
                    └────────────┘

Fact table: Events/measurements (billions of rows)
Dimension tables: Descriptive attributes (thousands of rows)
```

---

## 7. Event Sourcing {#event-sourcing}

```
Traditional (State-based):
  Account: { id: 1, balance: 750 }
  → Only current state, history lost

Event Sourcing:
  Events:
    { type: "AccountCreated", balance: 0, timestamp: T1 }
    { type: "Deposited", amount: 1000, timestamp: T2 }
    { type: "Withdrawn", amount: 250, timestamp: T3 }
  → Replay events → current state (balance = 750)
  → Full audit trail!
  → Can rebuild state at ANY point in time

Benefits:
✅ Complete audit trail
✅ Time-travel debugging
✅ Can derive new views from events
✅ Natural fit for event-driven architectures

Challenges:
❌ Storage grows forever
❌ Complex queries (need projections/snapshots)
❌ Schema evolution of events
❌ Eventually consistent projections
```

---

## 8. CQRS {#cqrs}

```
CQRS = Command Query Responsibility Segregation

Traditional: Same model for reads AND writes
CQRS: Separate models optimized for each

┌─────────────┐     ┌─────────────┐
│  Write Model│     │  Read Model │
│  (Commands) │     │  (Queries)  │
│  Normalized │     │ Denormalized│
│  Consistent │     │  Eventually │
└──────┬──────┘     └──────┬──────┘
       │                    │
       ▼                    ▼
┌─────────────┐     ┌─────────────┐
│  Write DB   │────▶│   Read DB   │
│ (PostgreSQL)│event│(Elasticsearch│
│             │sync │  / Redis)   │
└─────────────┘     └─────────────┘

Use when:
- Read patterns very different from write patterns
- Need different scaling for reads vs writes
- Complex query requirements (search, analytics)
- Event sourcing (events → projections)
```

---

## 9-10. Time-series + Tổng kết {#time-series}

```sql
-- Time-series data: Metrics, IoT, logs, financial data
-- Characteristics: Write-heavy, append-only, time-ordered, aggregation queries

-- PostgreSQL with TimescaleDB:
CREATE TABLE metrics (
  time TIMESTAMPTZ NOT NULL,
  device_id TEXT NOT NULL,
  temperature DOUBLE PRECISION,
  humidity DOUBLE PRECISION
);
SELECT create_hypertable('metrics', 'time');

-- Efficient queries:
SELECT time_bucket('1 hour', time) as hour,
       device_id,
       AVG(temperature),
       MAX(temperature)
FROM metrics
WHERE time > NOW() - INTERVAL '24 hours'
GROUP BY hour, device_id;
```

| Pattern | Best For |
|---------|----------|
| Normalized (3NF) | OLTP, transactional systems |
| Star Schema | Data warehouse, analytics |
| Event Sourcing | Audit trail, event-driven systems |
| CQRS | Different read/write requirements |
| Time-series | IoT, metrics, monitoring |
| Document (NoSQL) | Flexible schema, rapid iteration |

---
