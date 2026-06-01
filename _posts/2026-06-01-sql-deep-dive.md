---
layout: post
title: "SQL Deep Dive - Truy vấn SQL chuyên sâu"
date: 2026-06-01
categories: [database]
tags: [sql, joins, window-functions, cte, optimization, explain]
---

## Mục lục
1. [Góc nhìn tổng quan - Ngôn ngữ hỏi đáp dữ liệu](#goc-nhin-tong-quan)
2. [All JOINs explained - Các kiểu ghép bảng](#joins)
3. [Window Functions - ROW_NUMBER, RANK](#window-functions)
4. [Window Functions - LAG, LEAD, SUM OVER](#window-advanced)
5. [CTEs - Common Table Expressions](#ctes)
6. [Recursive CTEs - Truy vấn đệ quy](#recursive-cte)
7. [EXPLAIN/ANALYZE - Đọc execution plan](#explain)
8. [Query Optimization Techniques](#optimization)
9. [Real-world query patterns](#patterns)
10. [Tổng kết và performance tips](#tong-ket)

---

## 1. Góc nhìn tổng quan {#goc-nhin-tong-quan}

### Ví dụ đời thường

SQL advanced features giống **công cụ phân tích** trong kế toán:

- **JOINs** = ghép sổ sách: sổ đơn hàng + sổ khách hàng → báo cáo ai mua gì
- **Window Functions** = nhìn cả bảng nhưng tính cho từng dòng: "doanh thu nhân viên này so với toàn team"
- **CTEs** = giấy nháp: tính kết quả trung gian, đặt tên, rồi dùng lại
- **Recursive CTE** = tìm cấu trúc cây: "ai là sếp của sếp của sếp?"
- **EXPLAIN** = xem "kế hoạch thi công" trước khi database thực hiện query

---

## 2. All JOINs explained {#joins}

```sql
-- Sample tables:
-- employees(id, name, dept_id)
-- departments(id, name)

-- INNER JOIN: Chỉ lấy rows có match CẢ HAI bên
SELECT e.name, d.name as dept
FROM employees e
INNER JOIN departments d ON e.dept_id = d.id;
-- Nhân viên không có dept → BỊ BỎ
-- Dept không có nhân viên → BỊ BỎ

-- LEFT JOIN: Tất cả bên TRÁI, match bên phải (NULL nếu không có)
SELECT e.name, d.name as dept
FROM employees e
LEFT JOIN departments d ON e.dept_id = d.id;
-- Nhân viên không có dept → vẫn hiện (dept = NULL)

-- RIGHT JOIN: Tất cả bên PHẢI
SELECT e.name, d.name as dept
FROM employees e
RIGHT JOIN departments d ON e.dept_id = d.id;
-- Dept không có nhân viên → vẫn hiện (name = NULL)

-- FULL OUTER JOIN: Tất cả CẢ HAI bên
SELECT e.name, d.name as dept
FROM employees e
FULL OUTER JOIN departments d ON e.dept_id = d.id;

-- CROSS JOIN: Mọi tổ hợp (Cartesian product)
SELECT e.name, d.name
FROM employees e
CROSS JOIN departments d;
-- 10 employees × 5 departments = 50 rows

-- SELF JOIN: Bảng join với chính nó
SELECT e.name as employee, m.name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

## 3. Window Functions - ROW_NUMBER, RANK {#window-functions}

### Window Functions là gì?

```
GROUP BY: collapse nhiều rows → 1 row per group
Window Function: giữ NGUYÊN mọi row, thêm cột tính toán

Ví dụ: "Xếp hạng doanh thu từng nhân viên TRONG department của họ"
→ Window function tính rank PER partition (department)
   nhưng vẫn giữ từng row nhân viên

Syntax: function() OVER (PARTITION BY col ORDER BY col)
```

```sql
-- ROW_NUMBER: Đánh số 1, 2, 3... (luôn unique)
SELECT 
  name, department, salary,
  ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as row_num
FROM employees;
-- Result: Mỗi department, salary cao nhất = 1, kế tiếp = 2...
-- Ties: arbitrary (ai lên trước tùy database)

-- RANK: Đánh hạng (có thể skip - 1,2,2,4)
SELECT 
  name, department, salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank
FROM employees;
-- Ties: cùng salary = cùng rank, skip số tiếp theo

-- DENSE_RANK: Đánh hạng liên tục (1,2,2,3 - không skip)
SELECT 
  name, department, salary,
  DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dense_rank
FROM employees;

-- Use case: Top 3 earners per department
WITH ranked AS (
  SELECT *, DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rnk
  FROM employees
)
SELECT * FROM ranked WHERE rnk <= 3;
```

---

## 4. Window Functions - LAG, LEAD, SUM OVER {#window-advanced}

```sql
-- LAG: Giá trị dòng TRƯỚC (look back)
-- LEAD: Giá trị dòng SAU (look ahead)
SELECT 
  date, revenue,
  LAG(revenue, 1) OVER (ORDER BY date) as prev_day_revenue,
  revenue - LAG(revenue, 1) OVER (ORDER BY date) as daily_change,
  LEAD(revenue, 1) OVER (ORDER BY date) as next_day_revenue
FROM daily_sales;

-- Month-over-month growth
SELECT
  month,
  revenue,
  LAG(revenue) OVER (ORDER BY month) as prev_month,
  ROUND((revenue - LAG(revenue) OVER (ORDER BY month)) * 100.0 / 
        LAG(revenue) OVER (ORDER BY month), 2) as growth_pct
FROM monthly_sales;

-- Running total (SUM OVER)
SELECT
  date, amount,
  SUM(amount) OVER (ORDER BY date) as running_total,
  SUM(amount) OVER (PARTITION BY customer_id ORDER BY date) as customer_running_total
FROM orders;

-- Moving average (window frame)
SELECT
  date, revenue,
  AVG(revenue) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) as avg_7day
FROM daily_sales;

-- NTILE: Chia thành N nhóm bằng nhau
SELECT
  name, salary,
  NTILE(4) OVER (ORDER BY salary) as quartile
FROM employees;
-- quartile 1 = bottom 25%, quartile 4 = top 25%
```

---

## 5. CTEs - Common Table Expressions {#ctes}

```sql
-- CTE = named temporary result set
-- Syntax: WITH cte_name AS (query) SELECT ... FROM cte_name

-- Ví dụ 1: Readable complex query
WITH high_earners AS (
  SELECT * FROM employees WHERE salary > 100000
),
dept_stats AS (
  SELECT department, AVG(salary) as avg_salary, COUNT(*) as count
  FROM employees GROUP BY department
)
SELECT h.name, h.salary, d.avg_salary, d.count
FROM high_earners h
JOIN dept_stats d ON h.department = d.department;

-- Ví dụ 2: Deduplication
WITH ranked AS (
  SELECT *,
    ROW_NUMBER() OVER (PARTITION BY email ORDER BY created_at DESC) as rn
  FROM users
)
SELECT * FROM ranked WHERE rn = 1;  -- Latest record per email
```

---

## 6. Recursive CTEs {#recursive-cte}

```sql
-- Recursive CTE: query tham chiếu chính nó
-- Use case: Hierarchical data (org chart, categories, paths)

-- Employee hierarchy (who reports to whom)
WITH RECURSIVE org_chart AS (
  -- Base case: CEO (no manager)
  SELECT id, name, manager_id, 1 as level, name as path
  FROM employees
  WHERE manager_id IS NULL
  
  UNION ALL
  
  -- Recursive case: employees with managers
  SELECT e.id, e.name, e.manager_id, oc.level + 1,
         oc.path || ' > ' || e.name
  FROM employees e
  INNER JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT * FROM org_chart ORDER BY path;

-- Output:
-- CEO (level 1)
-- CEO > VP Sales (level 2)
-- CEO > VP Sales > Regional Manager (level 3)

-- Category tree
WITH RECURSIVE category_tree AS (
  SELECT id, name, parent_id, 0 as depth
  FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.name, c.parent_id, ct.depth + 1
  FROM categories c
  JOIN category_tree ct ON c.parent_id = ct.id
)
SELECT REPEAT('  ', depth) || name as hierarchy FROM category_tree;
```

---

## 7. EXPLAIN/ANALYZE {#explain}

```sql
-- EXPLAIN: Show query plan (không chạy query)
EXPLAIN SELECT * FROM orders WHERE customer_id = 123;

-- EXPLAIN ANALYZE: Chạy query + show actual timing
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 123;

-- PostgreSQL output example:
-- Seq Scan on orders  (cost=0.00..1234.00 rows=50 width=100)
--   (actual time=0.012..45.678 rows=47 loops=1)
--   Filter: (customer_id = 123)
--   Rows Removed by Filter: 99953
-- Planning Time: 0.123 ms
-- Execution Time: 45.789 ms

-- Đọc hiểu:
-- Seq Scan = Full table scan (chậm! cần index?)
-- cost=start..total (estimated)
-- rows = estimated rows returned
-- actual time = real execution time
-- Rows Removed by Filter = wasted work

-- Sau khi thêm index:
-- Index Scan using idx_orders_customer on orders (cost=0.42..8.44 rows=50)
--   (actual time=0.025..0.089 rows=47 loops=1)
-- → 500x faster!

-- Common scan types (tốt → xấu):
-- Index Only Scan: tốt nhất (data từ index, không cần table)
-- Index Scan: tốt (index tìm rows, fetch from table)
-- Bitmap Index Scan: OK (nhiều rows, 2-step)
-- Seq Scan: chậm nếu bảng lớn (full scan)
```

---

## 8. Query Optimization {#optimization}

```sql
-- 1. Tránh SELECT * (chỉ lấy columns cần)
SELECT id, name, email FROM users WHERE active = true;

-- 2. Index cho WHERE, JOIN, ORDER BY columns
CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at);

-- 3. Tránh functions trên indexed columns
-- ❌ WHERE YEAR(created_at) = 2024  (không dùng index)
-- ✅ WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'

-- 4. EXISTS thay vì IN cho subqueries lớn
-- ❌ WHERE id IN (SELECT user_id FROM large_table)
-- ✅ WHERE EXISTS (SELECT 1 FROM large_table lt WHERE lt.user_id = users.id)

-- 5. LIMIT cho pagination (nhưng cẩn thận OFFSET lớn)
-- ❌ SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 100000  (scan 100020 rows!)
-- ✅ SELECT * FROM orders WHERE id > last_seen_id ORDER BY id LIMIT 20

-- 6. Avoid N+1 queries (batch instead)
-- ❌ For each user: SELECT * FROM orders WHERE user_id = ?
-- ✅ SELECT * FROM orders WHERE user_id IN (1,2,3,4,5)
```

---

## 9-10. Real-world patterns + Tổng kết {#patterns}

```sql
-- Gap analysis (find missing sequences)
WITH RECURSIVE seq AS (
  SELECT MIN(id) as n FROM orders
  UNION ALL
  SELECT n + 1 FROM seq WHERE n < (SELECT MAX(id) FROM orders)
)
SELECT n as missing_id FROM seq
WHERE n NOT IN (SELECT id FROM orders);

-- Sessionization (group events into sessions, 30min gap)
WITH events_with_gap AS (
  SELECT *,
    CASE WHEN timestamp - LAG(timestamp) OVER (PARTITION BY user_id ORDER BY timestamp) 
         > INTERVAL '30 minutes' THEN 1 ELSE 0 END as new_session
  FROM page_views
)
SELECT *, SUM(new_session) OVER (PARTITION BY user_id ORDER BY timestamp) as session_id
FROM events_with_gap;
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| PostgreSQL Documentation - SQL | Chuẩn SQL reference |
| Use The Index, Luke (use-the-index-luke.com) | Indexing guide |
| SQL Performance Explained (Markus Winand) | Sách optimization |
| Mode Analytics SQL Tutorial | Interactive learning |

---

*Bài viết tiếp theo: [Database Indexing](/2026/08/21/database-indexing/)*
