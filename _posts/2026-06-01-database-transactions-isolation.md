---
layout: post
title: "Database Transactions & Isolation - Giao dịch và mức cô lập"
date: 2026-06-01
categories: [database]
tags: [transactions, acid, mvcc, isolation, deadlock, locking]
---

## Mục lục
1. [Góc nhìn tổng quan - Giao dịch ngân hàng](#goc-nhin)
2. [ACID Properties - 4 đảm bảo của transaction](#acid)
3. [Isolation Levels - Mức cô lập](#isolation)
4. [MVCC - Multi-Version Concurrency Control](#mvcc)
5. [Deadlocks - Bế tắc](#deadlocks)
6. [Optimistic vs Pessimistic Locking](#locking)
7. [Two-Phase Commit (2PC)](#2pc)
8. [Practical examples](#examples)
9. [PostgreSQL transaction internals](#postgres)
10. [Tổng kết](#tong-ket)

---

## 1. Góc nhìn tổng quan - Giao dịch ngân hàng {#goc-nhin}

### Ví dụ đời thường

Database transaction giống **giao dịch chuyển tiền ngân hàng**:

- Bạn chuyển 1 triệu từ TK A sang TK B
- Phải đảm bảo: trừ A VÀ cộng B cùng xảy ra, hoặc KHÔNG xảy ra gì
- Nếu hệ thống crash giữa chừng (đã trừ A, chưa cộng B) → rollback
- Hai người cùng chuyển tiền cùng lúc → không được conflict

```sql
-- Transaction chuyển tiền
BEGIN;
UPDATE accounts SET balance = balance - 1000000 WHERE id = 'A';
UPDATE accounts SET balance = balance + 1000000 WHERE id = 'B';
-- Nếu cả hai OK:
COMMIT;
-- Nếu lỗi:
ROLLBACK;
```

---

## 2. ACID Properties {#acid}

```
A - Atomicity (Nguyên tử):
    "Tất cả hoặc không gì"
    Transaction hoặc hoàn thành tất cả, hoặc rollback hoàn toàn.
    Ví dụ: Chuyển tiền - không thể chỉ trừ mà không cộng.

C - Consistency (Nhất quán):
    "Từ trạng thái hợp lệ → trạng thái hợp lệ khác"
    Constraints (foreign key, unique, check) luôn được duy trì.
    Ví dụ: Tổng tiền trong hệ thống luôn không đổi sau chuyển khoản.

I - Isolation (Cô lập):
    "Transactions không thấy nhau khi đang chạy"
    Concurrent transactions → kết quả như chạy sequential.
    Ví dụ: Hai người cùng check balance → không bị confuse.

D - Durability (Bền vững):
    "Committed = saved forever"
    Sau COMMIT, data tồn tại kể cả khi crash/mất điện.
    Ví dụ: Ngân hàng commit → tiền đã chuyển, không mất.
```

---

## 3. Isolation Levels {#isolation}

### Bảng Isolation Levels

```
┌───────────────────────┬──────────┬────────────────┬────────────────┬──────────────┐
│ Level                 │ Dirty    │ Non-Repeatable │ Phantom        │ Performance  │
│                       │ Read     │ Read           │ Read           │              │
├───────────────────────┼──────────┼────────────────┼────────────────┼──────────────┤
│ Read Uncommitted      │ Possible │ Possible       │ Possible       │ Fastest      │
│ Read Committed (*)    │ No       │ Possible       │ Possible       │ Fast         │
│ Repeatable Read       │ No       │ No             │ Possible(*)    │ Medium       │
│ Serializable          │ No       │ No             │ No             │ Slowest      │
└───────────────────────┴──────────┴────────────────┴────────────────┴──────────────┘

(*) PostgreSQL default = Read Committed
(*) PostgreSQL Repeatable Read thực tế ngăn cả Phantom (MVCC)
```

### Giải thích các anomaly

```sql
-- DIRTY READ: Đọc data chưa commit
-- TX1: UPDATE accounts SET balance = 0 WHERE id = 'A';
-- TX2: SELECT balance FROM accounts WHERE id = 'A'; → 0 (chưa commit!)
-- TX1: ROLLBACK;
-- → TX2 đọc giá trị KHÔNG TỒN TẠI!

-- NON-REPEATABLE READ: Đọc 2 lần khác kết quả
-- TX1: SELECT balance FROM accounts WHERE id = 'A'; → 1000
-- TX2: UPDATE accounts SET balance = 500 WHERE id = 'A'; COMMIT;
-- TX1: SELECT balance FROM accounts WHERE id = 'A'; → 500 (!)
-- → Cùng TX, cùng query, khác kết quả

-- PHANTOM READ: Rows mới xuất hiện giữa chừng
-- TX1: SELECT * FROM orders WHERE total > 1000; → 5 rows
-- TX2: INSERT INTO orders (total) VALUES (2000); COMMIT;
-- TX1: SELECT * FROM orders WHERE total > 1000; → 6 rows (!)
-- → Row mới "phantom" xuất hiện
```

---

## 4. MVCC - Multi-Version Concurrency Control {#mvcc}

```
MVCC = "Mỗi transaction thấy snapshot riêng"

Thay vì lock rows khi đọc, database giữ NHIỀU VERSIONS:
- Reader không block writer
- Writer không block reader
- Chỉ writer-writer mới conflict

PostgreSQL MVCC:
- Mỗi row có: xmin (TX tạo), xmax (TX xóa)
- UPDATE = INSERT new version + mark old version deleted
- Readers thấy version phù hợp với snapshot của mình
- VACUUM cleanup old versions

Ví dụ:
TX 100: INSERT (xmin=100, xmax=null)   → row visible
TX 200: UPDATE → new row (xmin=200), old row (xmax=200)
TX 150 (started before TX 200): vẫn thấy row cũ! (snapshot)
TX 250 (started after TX 200 committed): thấy row mới
```

---

## 5. Deadlocks {#deadlocks}

```sql
-- Deadlock = 2 transactions chờ nhau mãi mãi

-- TX1: UPDATE accounts SET balance=0 WHERE id='A';  -- Lock row A
-- TX2: UPDATE accounts SET balance=0 WHERE id='B';  -- Lock row B
-- TX1: UPDATE accounts SET balance=0 WHERE id='B';  -- WAIT for TX2...
-- TX2: UPDATE accounts SET balance=0 WHERE id='A';  -- WAIT for TX1...
-- → DEADLOCK! Cả hai chờ nhau mãi!

-- Database detects → kills one transaction (victim)
-- ERROR: deadlock detected

-- Prevention strategies:
-- 1. Access resources in consistent ORDER
-- (Always lock A before B, regardless of transaction)
-- 2. Use shorter transactions (less time holding locks)
-- 3. Use optimistic locking (no locks, detect conflicts)
-- 4. Set lock_timeout
SET lock_timeout = '5s';
```

---

## 6. Optimistic vs Pessimistic Locking {#locking}

```sql
-- PESSIMISTIC: Lock trước, sửa sau (bi quan, sợ conflict)
BEGIN;
SELECT * FROM products WHERE id = 1 FOR UPDATE;  -- LOCK row
-- ... process ...
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;  -- Release lock

-- OPTIMISTIC: Không lock, check version khi update
-- 1. Read with version
SELECT id, stock, version FROM products WHERE id = 1;
-- → stock=10, version=5

-- 2. Update with version check
UPDATE products 
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;
-- If 0 rows affected → someone else changed it → RETRY!

-- When to use:
-- Pessimistic: High contention (banking, inventory with low stock)
-- Optimistic: Low contention (read-heavy, rare conflicts)
```

---

## 7. Two-Phase Commit (2PC) {#2pc}

```
2PC = Distributed transaction across multiple databases

Phase 1 - PREPARE:
  Coordinator → All participants: "Can you commit?"
  Each participant: Write to WAL, respond YES/NO

Phase 2 - COMMIT/ABORT:
  If ALL said YES → Coordinator: "COMMIT"
  If ANY said NO  → Coordinator: "ABORT"

Problem: Coordinator crashes between phases → participants stuck!
→ Modern alternative: Saga pattern (compensating transactions)
```

---

## 8-10. Practical examples + PostgreSQL internals + Tổng kết {#examples}

```sql
-- Setting isolation level
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
-- ... your queries ...
COMMIT;

-- Check current level
SHOW transaction_isolation;

-- Advisory locks (application-level locks)
SELECT pg_advisory_lock(12345);      -- Acquire
-- ... critical section ...
SELECT pg_advisory_unlock(12345);    -- Release

-- Row-level locking options:
SELECT * FROM table FOR UPDATE;           -- Exclusive lock
SELECT * FROM table FOR SHARE;            -- Shared lock
SELECT * FROM table FOR UPDATE SKIP LOCKED; -- Skip locked rows
SELECT * FROM table FOR UPDATE NOWAIT;     -- Error if locked
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| PostgreSQL Transaction Documentation | Official isolation docs |
| Designing Data-Intensive Applications (Kleppmann) | Chapter 7: Transactions |
| SQL Standard ISO/IEC 9075 | Isolation level definitions |

---

*Bài viết tiếp theo: [DynamoDB Design Patterns](/2026/08/23/dynamodb-design-patterns/)*
