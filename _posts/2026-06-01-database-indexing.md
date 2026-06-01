---
layout: post
title: "Database Indexing - Đánh chỉ mục cơ sở dữ liệu"
date: 2026-06-01
categories: [database]
tags: [indexing, b-tree, gin, gist, postgresql, optimization]
---

## Mục lục
1. [Góc nhìn tổng quan - Mục lục sách](#goc-nhin-tong-quan)
2. [B-tree Index - Cây tìm kiếm cân bằng](#btree)
3. [Hash Index - Tra cứu trực tiếp](#hash)
4. [GIN Index - Inverted index cho arrays/JSON](#gin)
5. [GiST Index - Generalized search tree](#gist)
6. [Composite Index - Index nhiều cột](#composite)
7. [Covering và Partial Index](#covering-partial)
8. [Expression Index](#expression)
9. [Index-Only Scan và Maintenance](#index-only)
10. [Tổng kết và strategy](#tong-ket)

---

## 1. Góc nhìn tổng quan {#goc-nhin-tong-quan}

### Ví dụ đời thường

Database index giống **mục lục** cuối sách:

- **Không có index** = đọc từ trang 1 đến trang 500 để tìm từ "Kubernetes" → O(n) = chậm
- **Có index** = mở mục lục → "Kubernetes: trang 45, 123, 267" → O(log n) = nhanh
- **B-tree** = mục lục theo alphabet (tìm nhanh bằng binary search)
- **Hash** = mã màu sticky note (chính xác 1 từ → 1 vị trí)
- **GIN** = mục lục ngược: "từ X xuất hiện ở trang 5, 12, 89" (full-text search)
- **GiST** = mục lục cho bản đồ: "tìm tất cả thành phố trong bán kính 50km"

### Trade-offs của Index

```
Ưu điểm:
✅ SELECT nhanh hơn 100-10000x
✅ ORDER BY không cần sort
✅ JOIN nhanh hơn (index lookup)

Nhược điểm:
❌ INSERT/UPDATE/DELETE chậm hơn (phải update index)
❌ Tốn disk space (B-tree ≈ 1-3x data size)
❌ Maintenance overhead (VACUUM, REINDEX)

Nguyên tắc: Index cho columns hay query, KHÔNG index mọi thứ!
```

---

## 2. B-tree Index {#btree}

### B-tree là gì?

```
B-tree = Balanced tree (cây cân bằng):
- Default index type trong hầu hết databases
- Hỗ trợ: =, <, >, <=, >=, BETWEEN, IN, LIKE 'prefix%'
- Depth thường 3-4 levels (1M rows = 3 levels)
- Leaf nodes linked → range scan hiệu quả

         [50]
        /    \
    [20,35]   [70,85]
    / | \     / | \
 [..] [..] [..]  [..]  ← Leaf nodes (chứa data pointers)
```

```sql
-- Tạo B-tree index (default)
CREATE INDEX idx_users_email ON users(email);

-- Multi-column B-tree
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- B-tree hoạt động tốt cho:
WHERE email = 'john@example.com'     -- equality
WHERE created_at > '2024-01-01'      -- range
WHERE name LIKE 'John%'              -- prefix match (NOT %John%)
ORDER BY created_at DESC              -- sorted retrieval
WHERE id BETWEEN 100 AND 200          -- range scan
```

---

## 3. Hash Index {#hash}

```sql
-- Hash index: O(1) cho equality, KHÔNG hỗ trợ range
CREATE INDEX idx_sessions_token ON sessions USING hash(token);

-- Chỉ tốt cho: WHERE token = 'abc123'
-- KHÔNG dùng cho: WHERE token > 'abc', ORDER BY token

-- Khi nào dùng: 
-- Column chỉ dùng equality check (session tokens, UUIDs)
-- PostgreSQL 10+: WAL-logged, crash-safe
```

---

## 4. GIN Index {#gin}

```sql
-- GIN (Generalized Inverted Index): cho composite values
-- Arrays, JSONB, Full-text search, trigrams

-- Array contains
CREATE INDEX idx_tags ON articles USING gin(tags);
SELECT * FROM articles WHERE tags @> ARRAY['postgresql', 'performance'];

-- JSONB
CREATE INDEX idx_data ON events USING gin(data);
SELECT * FROM events WHERE data @> '{"type": "purchase"}';
SELECT * FROM events WHERE data ? 'email';  -- key exists

-- Full-text search
CREATE INDEX idx_fts ON documents USING gin(to_tsvector('english', content));
SELECT * FROM documents 
WHERE to_tsvector('english', content) @@ to_tsquery('kubernetes & networking');

-- Trigram (LIKE '%substring%')
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_name_trgm ON users USING gin(name gin_trgm_ops);
SELECT * FROM users WHERE name LIKE '%john%';  -- Uses index!
```

---

## 5. GiST Index {#gist}

```sql
-- GiST (Generalized Search Tree): spatial data, ranges, nearest-neighbor
-- PostGIS geometry, range types, full-text search

-- Spatial queries
CREATE INDEX idx_locations ON stores USING gist(location);
SELECT * FROM stores 
WHERE ST_DWithin(location, ST_MakePoint(106.7, 10.8)::geography, 5000);
-- Tìm stores trong 5km

-- Range overlap
CREATE INDEX idx_booking_dates ON bookings USING gist(daterange(check_in, check_out));
SELECT * FROM bookings 
WHERE daterange(check_in, check_out) && daterange('2024-03-01', '2024-03-15');

-- Exclusion constraint (no overlapping bookings for same room)
ALTER TABLE bookings ADD CONSTRAINT no_overlap 
  EXCLUDE USING gist (room_id WITH =, daterange(check_in, check_out) WITH &&);
```

---

## 6. Composite Index {#composite}

```sql
-- Column order MATTERS in composite index!
CREATE INDEX idx_orders_user_status_date ON orders(user_id, status, created_at);

-- This index helps:
WHERE user_id = 1                                    ✅ (leftmost prefix)
WHERE user_id = 1 AND status = 'active'             ✅ 
WHERE user_id = 1 AND status = 'active' AND created_at > '2024-01' ✅
WHERE user_id = 1 AND created_at > '2024-01'        ✅ (skip status, partial)

-- This index does NOT help:
WHERE status = 'active'                              ❌ (not leftmost)
WHERE created_at > '2024-01'                         ❌ (not leftmost)
WHERE status = 'active' AND created_at > '2024-01'   ❌

-- Rule: Index is useful for LEFTMOST PREFIX of columns
-- (user) → (user, status) → (user, status, date)
```

---

## 7. Covering và Partial Index {#covering-partial}

```sql
-- COVERING INDEX (Index-only scan): include extra columns
CREATE INDEX idx_users_email_covering ON users(email) INCLUDE (name, created_at);
-- Query satisfied entirely from index, no table fetch!
SELECT name, created_at FROM users WHERE email = 'john@example.com';

-- PARTIAL INDEX: Only index subset of rows
CREATE INDEX idx_orders_pending ON orders(created_at) 
  WHERE status = 'pending';
-- Smaller index! Only pending orders indexed.
-- Speeds up: SELECT * FROM orders WHERE status = 'pending' AND created_at > ...

-- Partial index for soft-delete
CREATE INDEX idx_users_active ON users(email) WHERE deleted_at IS NULL;
-- Only indexes non-deleted users (much smaller if many deleted)
```

---

## 8-10. Expression Index + Maintenance + Strategy {#expression}

```sql
-- Expression/Functional index
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
-- Now: WHERE LOWER(email) = 'john@example.com' uses index!

-- Index maintenance
REINDEX INDEX idx_users_email;           -- Rebuild bloated index
ANALYZE users;                           -- Update statistics
SELECT pg_size_pretty(pg_relation_size('idx_users_email'));  -- Check size

-- Monitoring unused indexes
SELECT indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%pkey%'
ORDER BY pg_relation_size(indexrelid) DESC;
-- idx_scan = 0 → unused index → candidate for DROP
```

### Indexing Strategy

```
1. Index columns in WHERE, JOIN ON, ORDER BY
2. Column order: high-cardinality first, range last
3. Use EXPLAIN ANALYZE to verify index usage
4. Monitor unused indexes (drop them)
5. Partial indexes for filtered queries
6. Covering indexes for frequently-accessed columns
7. GIN for JSONB/arrays/full-text
8. Don't over-index (slows writes)
```

---

*Bài viết tiếp theo: [Database Transactions & Isolation](/2026/08/22/database-transactions-isolation/)*
