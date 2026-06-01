---
layout: post
title: "Data - Phần 11: Databases & Data Architecture"
subtitle: "Hiểu cơ sở dữ liệu từ SQL đến NoSQL, OLTP đến Data Lake"
gh-repo: wayarmy/wayarmy.github.io
tags: [databases, aws, learning-path]
comments: true
date: 2026-06-01
categories: [database]
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Phần 11).
>
> **Đối tượng:** Người mới hoàn toàn — biết coding cơ bản.
>
> **Nguồn tham khảo:**
> - Codd, E.F. (1970). "A Relational Model of Data for Large Shared Data Banks". Communications of the ACM.
> - Date, C.J. "An Introduction to Database Systems" — 8th Edition
> - AWS Documentation: [RDS](https://docs.aws.amazon.com/rds/), [DynamoDB](https://docs.aws.amazon.com/dynamodb/), [Redshift](https://docs.aws.amazon.com/redshift/), [Glue](https://docs.aws.amazon.com/glue/)
> - PostgreSQL Official Documentation — [https://www.postgresql.org/docs/](https://www.postgresql.org/docs/)
> - MongoDB Manual — [https://www.mongodb.com/docs/manual/](https://www.mongodb.com/docs/manual/)

---

## 1. Database là gì? — "Kho lưu trữ có tổ chức"

### Ví dụ đời thường:

Hãy nghĩ về **thư viện**:
- **Sách** = dữ liệu (data)
- **Kệ sách có phân loại** (Văn học, Khoa học, Lịch sử) = bảng (tables)
- **Thẻ thư viện + hệ thống danh mục** = database management system (DBMS)
- **Thủ thư** = query engine (tìm sách theo yêu cầu)

Nếu bạn chỉ đổ sách vào phòng KHÔNG có tổ chức → tìm 1 cuốn mất hàng giờ.
Nếu có hệ thống (phân loại, đánh index, danh mục) → tìm trong vài giây.

**Database** = Nơi lưu trữ dữ liệu có cấu trúc, cho phép truy vấn (query) nhanh chóng.

---

## 2. Relational Database (RDBMS) — Cơ sở dữ liệu quan hệ

### Ví dụ đời thường:

RDBMS giống **bảng tính Excel** nhưng mạnh hơn nhiều:
- Mỗi **sheet** = 1 **table** (bảng)
- Mỗi **cột** = 1 **column/field** (trường)
- Mỗi **hàng** = 1 **row/record** (bản ghi)
- Các sheet có thể **liên kết** với nhau (relationship)

### Lịch sử — Mô hình quan hệ (Codd, 1970):

Edgar F. Codd tại IBM đề xuất **Relational Model** trong paper lịch sử *"A Relational Model of Data for Large Shared Data Banks"* (1970). Ý tưởng cốt lõi:
- Dữ liệu được tổ chức trong **relations** (bảng)
- Truy vấn bằng **relational algebra** (nền tảng của SQL)
- Tách biệt **logical view** (user thấy gì) khỏi **physical storage** (lưu như thế nào)

### Ví dụ cụ thể:

```
Table: customers (Khách hàng)
┌──────┬──────────┬──────────────────┬─────────────┐
│ id   │ name     │ email            │ city        │
├──────┼──────────┼──────────────────┼─────────────┤
│ 1    │ Minh     │ minh@email.com   │ Hà Nội     │
│ 2    │ Lan      │ lan@email.com    │ HCM        │
│ 3    │ Hùng     │ hung@email.com   │ Đà Nẵng    │
└──────┴──────────┴──────────────────┴─────────────┘

Table: orders (Đơn hàng)
┌──────┬─────────────┬──────────┬──────────┐
│ id   │ customer_id │ product  │ amount   │
├──────┼─────────────┼──────────┼──────────┤
│ 101  │ 1           │ Laptop   │ 25000000 │
│ 102  │ 1           │ Mouse    │ 500000   │
│ 103  │ 2           │ Keyboard │ 1200000  │
└──────┴─────────────┴──────────┴──────────┘

Relationship: orders.customer_id → customers.id
(Mỗi đơn hàng THUỘC VỀ 1 khách hàng)
```

### Key concepts:

| Khái niệm | Mô tả | Ví dụ |
|-----------|--------|-------|
| **Primary Key (PK)** | Định danh duy nhất mỗi row | `customers.id` |
| **Foreign Key (FK)** | Liên kết đến PK ở bảng khác | `orders.customer_id` → `customers.id` |
| **Index** | "Mục lục" giúp tìm nhanh | Index trên `email` → tìm user theo email nhanh |
| **Constraint** | Ràng buộc dữ liệu | NOT NULL, UNIQUE, CHECK |
| **Schema** | Cấu trúc database (tables, relationships) | Blueprint của database |

---

## 3. SQL — Ngôn ngữ truy vấn dữ liệu

### SQL (Structured Query Language) là gì?

SQL giống **ngôn ngữ để "nói chuyện" với database** — bạn yêu cầu gì, database trả lời.

### CRUD Operations:

| Operation | SQL Command | Ví dụ đời thường |
|-----------|-------------|-----------------|
| **C**reate | INSERT | Thêm sách mới vào thư viện |
| **R**ead | SELECT | Tìm sách theo tên |
| **U**pdate | UPDATE | Sửa thông tin sách |
| **D**elete | DELETE | Xóa sách khỏi hệ thống |

### Ví dụ SQL cơ bản:

```sql
-- CREATE: Tạo bảng
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    city VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- INSERT: Thêm dữ liệu
INSERT INTO customers (name, email, city)
VALUES ('Minh', 'minh@email.com', 'Hà Nội');

INSERT INTO customers (name, email, city)
VALUES ('Lan', 'lan@email.com', 'HCM'),
       ('Hùng', 'hung@email.com', 'Đà Nẵng');

-- SELECT: Truy vấn dữ liệu
SELECT * FROM customers;                          -- Tất cả
SELECT name, email FROM customers WHERE city = 'Hà Nội';  -- Lọc
SELECT city, COUNT(*) FROM customers GROUP BY city;        -- Nhóm + đếm

-- UPDATE: Cập nhật
UPDATE customers SET city = 'HCM' WHERE id = 3;

-- DELETE: Xóa
DELETE FROM customers WHERE id = 3;
```

### JOIN — Kết hợp nhiều bảng:

```sql
-- INNER JOIN: Chỉ lấy rows có match ở CẢ HAI bảng
SELECT c.name, o.product, o.amount
FROM customers c
INNER JOIN orders o ON c.id = o.customer_id;

-- Result:
-- Minh  | Laptop   | 25000000
-- Minh  | Mouse    | 500000
-- Lan   | Keyboard | 1200000
-- (Hùng không có đơn → không xuất hiện)

-- LEFT JOIN: Lấy TẤT CẢ rows từ bảng trái, match hoặc NULL
SELECT c.name, o.product
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id;

-- Result:
-- Minh  | Laptop
-- Minh  | Mouse
-- Lan   | Keyboard
-- Hùng  | NULL      ← Không có đơn nhưng vẫn xuất hiện
```

---

## 4. ACID — Đảm bảo tính toàn vẹn dữ liệu

### Ví dụ đời thường — Chuyển tiền ngân hàng:

Bạn chuyển **5 triệu** từ tài khoản A sang tài khoản B:
1. Trừ 5 triệu từ A
2. Cộng 5 triệu vào B

Điều gì xảy ra nếu hệ thống crash SAU bước 1 nhưng TRƯỚC bước 2? → Tiền biến mất!

**ACID** đảm bảo điều này KHÔNG xảy ra:

| Tính chất | Ý nghĩa | Ví dụ |
|-----------|---------|-------|
| **A**tomicity | All or nothing — hoặc thành công HẾT, hoặc rollback HẾT | Cả 2 bước thành công, hoặc cả 2 rollback |
| **C**onsistency | Database luôn ở trạng thái hợp lệ | Tổng tiền 2 tài khoản luôn = const |
| **I**solation | Transactions không ảnh hưởng lẫn nhau | 2 người chuyển tiền cùng lúc → không conflict |
| **D**urability | Committed data không bao giờ mất | Sau khi confirm → dù crash, data vẫn còn |

### Transaction example:

```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 5000000 WHERE id = 'A';
UPDATE accounts SET balance = balance + 5000000 WHERE id = 'B';

-- Nếu cả 2 OK:
COMMIT;

-- Nếu lỗi bất kỳ đâu:
ROLLBACK;   -- Quay lại trạng thái trước
```

---

## 5. Normalization — Tổ chức dữ liệu tối ưu

### Ví dụ đời thường:

**Không normalized (lặp lại dữ liệu):**

| Order ID | Customer | Customer Email | Product | Price |
|----------|----------|----------------|---------|-------|
| 101 | Minh | minh@email.com | Laptop | 25M |
| 102 | Minh | minh@email.com | Mouse | 500K |
| 103 | Minh | minh@email.com | Monitor | 8M |

**Vấn đề:**
- Email "minh@email.com" lặp 3 lần → tốn storage
- Nếu Minh đổi email → phải sửa 3 chỗ → dễ sai sót
- Xóa order cuối → mất luôn thông tin customer

**Normalized (tách thành bảng riêng):**

```
customers: {id: 1, name: "Minh", email: "minh@email.com"}
orders: {id: 101, customer_id: 1, product: "Laptop", price: 25M}
        {id: 102, customer_id: 1, product: "Mouse", price: 500K}
```

### Các dạng chuẩn (Normal Forms):

| Form | Quy tắc | Ý nghĩa đơn giản |
|------|---------|-------------------|
| **1NF** | Mỗi ô chứa 1 giá trị atomic | Không có "danh sách" trong 1 ô |
| **2NF** | 1NF + mọi non-key column phụ thuộc TOÀN BỘ PK | Không phụ thuộc một phần của composite key |
| **3NF** | 2NF + không có transitive dependency | Column không phụ thuộc qua column khác (ngoài PK) |

**Thực tế:** Hầu hết ứng dụng normalize đến 3NF là đủ. Over-normalizing gây quá nhiều JOINs → chậm.

---

## 6. NoSQL — Khi Relational không đủ

### Tại sao cần NoSQL?

RDBMS tuyệt vời nhưng có hạn chế:
- **Schema cứng nhắc:** Mỗi lần thêm column phải ALTER TABLE (downtime)
- **Scaling khó:** Scale vertically (máy to hơn) dễ, horizontally (nhiều máy) khó
- **Dữ liệu phi cấu trúc:** JSON lồng nhau, document phức tạp → ép vào tables không tự nhiên

### Các loại NoSQL:

#### 1. Document Database (MongoDB, DynamoDB):

Lưu dữ liệu dạng **document** (JSON/BSON) — mỗi document có thể có cấu trúc khác nhau.

```json
// MongoDB document
{
  "_id": "user_001",
  "name": "Minh",
  "email": "minh@email.com",
  "addresses": [
    {"type": "home", "city": "Hà Nội", "street": "123 Lê Lợi"},
    {"type": "work", "city": "Hà Nội", "street": "456 Trần Hưng Đạo"}
  ],
  "orders": [
    {"product": "Laptop", "amount": 25000000, "date": "2026-06-01"}
  ]
}
```

**Ưu điểm:** Flexible schema, nested data tự nhiên, scale dễ
**Use case:** CMS, user profiles, product catalogs, real-time analytics

#### 2. Key-Value Store (Redis, DynamoDB):

Đơn giản nhất — giống **dictionary/hashmap**: key → value.

```
Key: "session:abc123"  →  Value: {"user": "minh", "expires": "2026-06-12"}
Key: "cart:user001"    →  Value: [{"item": "laptop", "qty": 1}]
Key: "rate:usd-vnd"    →  Value: "25000"
```

**Ưu điểm:** Cực nhanh (O(1) lookup), đơn giản
**Use case:** Caching, sessions, rate limiting, leaderboards

#### 3. Column-Family Store (Cassandra, HBase):

Lưu theo **column families** — tối ưu cho write-heavy workloads với dữ liệu phân tán.

**Use case:** Time-series data, IoT sensor data, messaging systems

#### 4. Graph Database (Neo4j, Neptune):

Lưu **nodes** (thực thể) và **edges** (quan hệ) — tối ưu cho truy vấn quan hệ phức tạp.

```
(Minh)-[:FRIENDS_WITH]->(Lan)
(Minh)-[:WORKS_AT]->(CompanyA)
(Lan)-[:WORKS_AT]->(CompanyA)
(Minh)-[:PURCHASED]->(Laptop)
```

**Use case:** Social networks, recommendation engines, fraud detection

### SQL vs NoSQL — Khi nào dùng gì?

| Tiêu chí | SQL (RDBMS) | NoSQL |
|----------|-------------|-------|
| Schema | Fixed (phải define trước) | Flexible (thay đổi thoải mái) |
| Relationships | Mạnh (JOINs) | Yếu hoặc denormalized |
| Transactions | ACID đầy đủ | Thường eventual consistency |
| Scaling | Vertical (scale up) | Horizontal (scale out) |
| Query language | SQL (standardized) | API-specific |
| Use case | Banking, ERP, dữ liệu quan hệ phức tạp | Real-time apps, big data, flexible schemas |

---

## 7. OLTP vs OLAP — Hai "kiểu đọc" dữ liệu

### Ví dụ đời thường:

| | OLTP | OLAP |
|-|------|------|
| **Tên** | Online Transaction Processing | Online Analytical Processing |
| **Ví dụ** | Nhân viên bán hàng **ghi hóa đơn** | Giám đốc **xem báo cáo doanh thu quý** |
| **Đặc điểm** | Nhiều giao dịch nhỏ, nhanh | Ít query nhưng scan dữ liệu khổng lồ |
| **Data** | Current (hiện tại) | Historical (lịch sử) |
| **Query** | INSERT 1 row, UPDATE 1 row | SELECT SUM(), GROUP BY, spanning millions of rows |
| **Latency** | Milliseconds | Seconds to minutes |
| **Users** | Hàng nghìn (end users) | Ít (analysts, managers) |

### So sánh kỹ thuật:

| Đặc điểm | OLTP | OLAP (Data Warehouse) |
|-----------|------|----------------------|
| Design | Normalized (3NF) | Denormalized (Star/Snowflake schema) |
| Storage | Row-oriented | Column-oriented |
| Optimize for | Write performance | Read/scan performance |
| Example | PostgreSQL, MySQL | Redshift, BigQuery, Snowflake |

### Star Schema — Thiết kế cho Data Warehouse:

```
                    ┌──────────────┐
                    │  dim_date    │
                    │ (Ngày tháng) │
                    └──────┬───────┘
                           │
┌──────────────┐    ┌──────┴───────┐    ┌──────────────┐
│ dim_product  ├────┤  fact_sales  ├────┤ dim_customer │
│ (Sản phẩm)  │    │ (Doanh thu)  │    │ (Khách hàng) │
└──────────────┘    └──────┬───────┘    └──────────────┘
                           │
                    ┌──────┴───────┐
                    │  dim_store   │
                    │ (Cửa hàng)  │
                    └──────────────┘
```

**Fact table** (trung tâm): Chứa metrics/measures (doanh thu, số lượng)
**Dimension tables** (xung quanh): Chứa attributes để filter/group (thời gian, sản phẩm, khách hàng)

---

## 8. ETL/ELT và Data Lake

### ETL (Extract, Transform, Load):

**Ví dụ đời thường:** Nhập khẩu trái cây:
1. **Extract:** Thu hoạch trái cây từ vườn (lấy data từ nguồn)
2. **Transform:** Rửa sạch, phân loại, đóng gói (clean, format, aggregate data)
3. **Load:** Đưa vào siêu thị (load vào Data Warehouse)

```
Data Sources          ETL Pipeline          Data Warehouse
┌──────────┐                                ┌──────────────┐
│ Database ├──┐                             │              │
└──────────┘  │    ┌───────────────────┐    │   Redshift   │
┌──────────┐  ├───→│  E → T → L       │───→│   (OLAP)     │
│   API    ├──┤    │  Extract          │    │              │
└──────────┘  │    │  Transform        │    │  Analytics   │
┌──────────┐  │    │  Load             │    │  Dashboards  │
│ CSV/Logs ├──┘    └───────────────────┘    └──────────────┘
└──────────┘
```

### ELT (Extract, Load, Transform):

Xu hướng hiện đại — load RAW data vào trước, transform SAU (trong Data Warehouse/Lake):

```
Sources → Extract → Load (raw) → Data Lake/Warehouse → Transform (in-place)
```

**Tại sao ELT?** Data Warehouse hiện đại (Redshift, BigQuery) đủ mạnh để transform; lưu raw data cho phép transform lại nếu cần.

### Data Lake — "Hồ dữ liệu":

**Ví dụ:** Data Warehouse giống **siêu thị** (dữ liệu đã phân loại, sạch sẽ, sẵn sàng dùng). Data Lake giống **hồ nước** — chứa MỌI THỨ ở dạng thô (raw), chưa xử lý.

| Đặc điểm | Data Warehouse | Data Lake |
|-----------|---------------|-----------|
| Data | Structured, cleaned | Raw (structured + unstructured) |
| Schema | Schema-on-write (define trước) | Schema-on-read (define khi đọc) |
| Users | Business analysts | Data scientists, ML engineers |
| Cost | Đắt (optimized storage) | Rẻ (S3, object storage) |
| Format | Proprietary | Open (Parquet, ORC, JSON, CSV) |

### Data Lake Architecture trên AWS:

```
┌─── Data Sources ───┐          ┌──── Data Lake (S3) ────────────┐
│ RDS, DynamoDB      │          │                                 │
│ API logs           │──ETL/───→│  s3://datalake/raw/            │
│ IoT devices        │  ELT     │  s3://datalake/processed/      │
│ SaaS (Salesforce)  │          │  s3://datalake/curated/        │
└────────────────────┘          └─────────────┬───────────────────┘
                                              │
                                   ┌──────────┼──────────────┐
                                   │          │              │
                                   ▼          ▼              ▼
                              Athena      Redshift      SageMaker
                            (Ad-hoc SQL) (BI/Reports)  (ML Training)
```

---

## 9. AWS Database Services

### Amazon RDS (Relational Database Service):

**RDS** = Managed RDBMS — AWS lo: patching, backups, replication, failover. Bạn chỉ lo: schema + queries.

**Engines hỗ trợ:** MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, **Aurora** (AWS custom)

```
                ┌─────── RDS ─────────────────────────┐
                │                                      │
                │  ┌─── Primary (Write) ───┐          │
                │  │  PostgreSQL 15         │          │
                │  │  db.r6g.xlarge         │          │
                │  └──────────┬────────────┘          │
                │             │ Synchronous            │
                │             │ Replication            │
                │  ┌──────────▼────────────┐          │
                │  │  Standby (Failover)    │  (AZ-b) │
                │  └───────────────────────┘          │
                │                                      │
                │  ┌─── Read Replica ──────┐          │
                │  │  (Async replication)   │          │
                │  │  For read-heavy        │          │
                │  └───────────────────────┘          │
                │                                      │
                └──────────────────────────────────────┘
```

**Aurora** — AWS-engineered database:
- Compatible với MySQL/PostgreSQL
- **5x** performance so với MySQL, **3x** PostgreSQL
- Storage tự auto-scale (10GB → 128TB)
- 6 copies of data across 3 AZs
- Serverless option (auto-scale compute)

### Amazon DynamoDB:

**DynamoDB** = Managed NoSQL (key-value + document) — serverless, millisecond latency at any scale.

```
Table: Orders
┌───────────────────┬─────────────────┬───────────────────────────────────┐
│ Partition Key     │ Sort Key        │ Attributes                        │
│ (customer_id)     │ (order_date)    │                                   │
├───────────────────┼─────────────────┼───────────────────────────────────┤
│ CUST001           │ 2026-06-01      │ {product: "Laptop", amount: 25M} │
│ CUST001           │ 2026-06-05      │ {product: "Mouse", amount: 500K} │
│ CUST002           │ 2026-06-03      │ {product: "Monitor", amount: 8M} │
└───────────────────┴─────────────────┴───────────────────────────────────┘
```

**Đặc điểm:**
- **Serverless:** Không quản lý server
- **Auto-scaling:** Tự điều chỉnh capacity
- **Single-digit millisecond:** Latency ổn định dù table có hàng tỷ items
- **Global Tables:** Replicate across regions
- **DynamoDB Streams:** Event-driven (trigger Lambda khi data thay đổi)

**Pricing model:**
- **On-Demand:** Trả theo request (good for unpredictable workloads)
- **Provisioned:** Đặt trước RCU/WCU (cheaper for stable workloads)

### Amazon Redshift — Data Warehouse:

**Redshift** = Managed OLAP data warehouse — columnar storage, massively parallel processing (MPP).

**Use case:** Business Intelligence, reporting trên petabytes of data.

```sql
-- Ví dụ query trên Redshift (billions of rows, returns in seconds)
SELECT
    d.year,
    d.quarter,
    p.category,
    SUM(f.revenue) as total_revenue,
    COUNT(DISTINCT f.customer_id) as unique_customers
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
WHERE d.year = 2026
GROUP BY d.year, d.quarter, p.category
ORDER BY total_revenue DESC;
```

### AWS Glue — ETL Service:

**Glue** = Managed ETL — crawl data sources, build data catalog, run ETL jobs.

```
┌───────────────┐     ┌──────────────┐     ┌─────────────────┐
│  Data Sources │     │   AWS Glue   │     │  Data Lake/DW   │
│               │     │              │     │                 │
│ S3 (CSV/JSON) ├────→│ Crawler      │     │ S3 (Parquet)    │
│ RDS           │     │ → Data Catalog│────→│ Redshift        │
│ DynamoDB      │     │ → ETL Jobs   │     │ Athena          │
└───────────────┘     └──────────────┘     └─────────────────┘
```

### Chọn database nào trên AWS?

| Use case | AWS Service | Lý do |
|----------|-------------|-------|
| Web app, e-commerce (OLTP) | **RDS** (PostgreSQL/MySQL) | ACID, relationships, mature |
| High-traffic, flexible schema | **DynamoDB** | Serverless, milliseconds, scale |
| Analytics, BI reports (OLAP) | **Redshift** | Columnar, petabyte scale |
| Caching, sessions | **ElastiCache** (Redis) | Sub-millisecond, in-memory |
| Graph queries (social, fraud) | **Neptune** | Graph model, traversal queries |
| Time-series (IoT, metrics) | **Timestream** | Optimized for time-series |
| Document search (full-text) | **OpenSearch** | Elasticsearch-compatible |
| Data Lake catalog + ETL | **Glue** + **Athena** | Serverless, schema-on-read |

---

## 10. Database Indexing — Tối ưu tốc độ truy vấn

### Ví dụ đời thường:

**Index** giống **mục lục** cuối sách giáo khoa:
- Không có index: Tìm từ "photosynthesis" → lật từng trang (full table scan)
- Có index: Tra mục lục → "photosynthesis: trang 45, 67, 120" → đi thẳng đến

### B-Tree Index (phổ biến nhất):

```
                    ┌───────────────────┐
                    │   [M]             │   ← Root
                    └─────┬──────┬──────┘
                          │      │
              ┌───────────┘      └───────────┐
              ▼                              ▼
    ┌─────────────────┐            ┌─────────────────┐
    │ [D] [H]         │            │ [R] [W]         │   ← Internal
    └──┬───┬────┬─────┘            └──┬───┬────┬─────┘
       │   │    │                     │   │    │
       ▼   ▼    ▼                     ▼   ▼    ▼
     [A-C][D-G][H-L]               [M-Q][R-V][W-Z]      ← Leaf (data pointers)
```

**Cách hoạt động:** Tìm "Minh" → Root: M ≥ M → go right → Internal: M < R → go left → Leaf: [M-Q] → found!

**Khi nào tạo index:**
- Column thường dùng trong WHERE, JOIN, ORDER BY
- Column có high cardinality (nhiều giá trị unique)

**Khi nào KHÔNG nên:**
- Table nhỏ (full scan nhanh hơn)
- Column ít giá trị unique (gender: M/F)
- Table write-heavy (index phải update mỗi lần INSERT/UPDATE)

```sql
-- Tạo index
CREATE INDEX idx_customer_email ON customers(email);
CREATE INDEX idx_orders_date ON orders(order_date);

-- Composite index (nhiều columns)
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Kiểm tra query có dùng index không
EXPLAIN ANALYZE SELECT * FROM customers WHERE email = 'minh@email.com';
```

---

## 11. Backup và High Availability

### Backup strategies:

| Strategy | Mô tả | RPO | RTO |
|----------|--------|-----|-----|
| **Full backup** | Copy toàn bộ DB | Point-in-time | Giờ |
| **Incremental** | Chỉ copy changes since last backup | Phút | Phút |
| **Continuous (WAL)** | Stream write-ahead logs realtime | Giây | Phút |
| **Snapshot** | Point-in-time snapshot (EBS/RDS) | Giờ (scheduled) | Phút |

**RPO (Recovery Point Objective):** Mất tối đa bao nhiêu data
**RTO (Recovery Time Objective):** Thời gian khôi phục

### RDS Backup trên AWS:

```
Automated Backups:
- Daily snapshot (retention: 1-35 days)
- Transaction logs every 5 minutes
- Point-in-time recovery (đến bất kỳ giây nào trong retention period)

Manual Snapshots:
- User-triggered, persist indefinitely
- Can share across accounts, copy across regions
```

### Multi-AZ Deployment:

```
┌─── AZ-a ────────┐          ┌─── AZ-b ────────┐
│                  │          │                  │
│  RDS Primary     │ ──sync──→│  RDS Standby    │
│  (read + write)  │ replication│ (not accessible)│
│                  │          │                  │
└──────────────────┘          └──────────────────┘
         │
    If Primary fails → automatic failover → Standby becomes Primary
         (typically 60-120 seconds)
```

---

## 12. Thực hành: Lab tự làm

### Lab 1: SQL cơ bản với PostgreSQL

```bash
# Chạy PostgreSQL via Docker
docker run -d --name mydb \
    -e POSTGRES_PASSWORD=[REDACTED_PASSWORD] \
    -e POSTGRES_DB=myapp \
    -p 5432:5432 \
    postgres:15

# Connect
docker exec -it mydb psql -U postgres -d myapp

# Trong psql:
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2),
    category VARCHAR(50),
    stock INTEGER DEFAULT 0
);

INSERT INTO products (name, price, category, stock) VALUES
    ('Laptop Dell', 25000000, 'Electronics', 50),
    ('iPhone 15', 30000000, 'Electronics', 100),
    ('Sách SQL', 250000, 'Books', 200);

SELECT * FROM products WHERE category = 'Electronics' ORDER BY price DESC;
```

### Lab 2: DynamoDB Local

```bash
# Chạy DynamoDB local
docker run -d -p 8000:8000 amazon/dynamodb-local

# Tạo table (AWS CLI)
aws dynamodb create-table \
    --table-name Users \
    --attribute-definitions AttributeName=userId,AttributeType=S \
    --key-schema AttributeName=userId,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --endpoint-url http://localhost:8000
```

### Lab 3: RDS trên AWS

1. Tạo RDS PostgreSQL (Free Tier: db.t3.micro)
2. Configure Security Group (allow 5432 from your IP)
3. Connect từ local: `psql -h <endpoint> -U postgres -d myapp`
4. Tạo tables, insert data, query
5. Tạo manual snapshot
6. Enable Multi-AZ (chuyển sang paid tier)

### Lab 4: Redshift Serverless

1. Tạo Redshift Serverless workspace
2. Load sample data (TPC-H benchmark)
3. Chạy analytical queries
4. So sánh performance với RDS PostgreSQL trên cùng data

---

## 13. Tổng kết

| Khái niệm | Ví dụ đời thường | AWS Service |
|-----------|-----------------|-------------|
| RDBMS | Bảng tính Excel có liên kết | RDS, Aurora |
| NoSQL | Tủ hồ sơ linh hoạt | DynamoDB |
| ACID | Chuyển tiền an toàn | RDS (transactions) |
| Index | Mục lục sách | CREATE INDEX |
| OLTP | Ghi hóa đơn bán hàng | RDS, DynamoDB |
| OLAP | Báo cáo doanh thu quý | Redshift |
| ETL | Thu hoạch → phân loại → siêu thị | Glue |
| Data Lake | Hồ chứa mọi thứ (thô) | S3 + Athena + Glue |

---

## Tài liệu tham khảo

1. **Codd, E.F. (1970)** — "A Relational Model of Data for Large Shared Data Banks". Communications of the ACM, 13(6), pp. 377-387.
2. **Date, C.J.** — "An Introduction to Database Systems", 8th Edition, Pearson.
3. **PostgreSQL Documentation** — [https://www.postgresql.org/docs/15/](https://www.postgresql.org/docs/15/)
4. **AWS RDS User Guide** — [https://docs.aws.amazon.com/rds/](https://docs.aws.amazon.com/rds/)
5. **AWS DynamoDB Developer Guide** — [https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/)
6. **AWS Redshift** — [https://docs.aws.amazon.com/redshift/](https://docs.aws.amazon.com/redshift/)
7. **AWS Glue** — [https://docs.aws.amazon.com/glue/](https://docs.aws.amazon.com/glue/)
8. **MongoDB Manual** — [https://www.mongodb.com/docs/manual/](https://www.mongodb.com/docs/manual/)

---

**Bài tiếp theo:** [Phần 12: Security Fundamentals — Bảo mật cơ bản cho Cloud](/2026-06-01-security-fundamentals/)
