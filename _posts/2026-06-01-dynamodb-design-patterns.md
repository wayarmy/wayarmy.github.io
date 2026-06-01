---
layout: post
title: "DynamoDB Design Patterns - Thiết kế DynamoDB"
date: 2026-06-01
categories: [database]
tags: [dynamodb, single-table, gsi, lsi, nosql, access-patterns]
---

## Mục lục
1. [Góc nhìn tổng quan - Thư viện không có kệ](#overview)
2. [Partition Key Design - Chìa khóa phân phối](#partition-key)
3. [Single-Table Design - Một bảng cho tất cả](#single-table)
4. [GSI và LSI - Chỉ mục phụ](#gsi-lsi)
5. [Access Patterns - Thiết kế theo query](#access-patterns)
6. [DynamoDB Streams](#streams)
7. [TTL - Tự động xóa data cũ](#ttl)
8. [Transactions trong DynamoDB](#transactions)
9. [Best practices và Anti-patterns](#best-practices)
10. [Tổng kết](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

DynamoDB giống **tủ hồ sơ khổng lồ** nhưng KHÔNG có thứ tự alphabet:

- **Partition Key (PK)** = số ngăn tủ (quyết định data ở ngăn nào)
- **Sort Key (SK)** = thứ tự trong ngăn (sắp xếp items trong cùng partition)
- **GSI** = photocopy hồ sơ theo tiêu chí khác (index phụ)
- **Single-Table Design** = TOÀN BỘ data (users, orders, products) trong 1 tủ, phân biệt bằng prefix

### Khác biệt tư duy SQL vs DynamoDB

```
SQL (Relational):
- Thiết kế schema trước (normalization)
- Query flexible (JOIN bất kỳ)
- Optimize SAU khi biết query patterns
- Entity-per-table

DynamoDB (NoSQL):
- Xác định ACCESS PATTERNS trước
- Thiết kế keys/indexes PHÙ HỢP patterns
- Không có JOIN (denormalize)
- Có thể nhiều entities chung 1 table
```

---

## 2. Partition Key Design {#partition-key}

```
PK quyết định data được lưu ở partition nào:
- High cardinality = phân phối đều = performance tốt
- Low cardinality = hot partition = throttling!

✅ Good PK: user_id, order_id, device_id (millions of unique values)
❌ Bad PK: status ("active"/"inactive"), country (chỉ ~200 giá trị)
❌ Bad PK: date ("2024-01-15") → hot partition ngày hôm nay!

Hot partition solution:
- Write sharding: PK = "2024-01-15#" + random(1..10)
- Scatter-gather queries across shards
```

---

## 3. Single-Table Design {#single-table}

```
Ý tưởng: Tất cả entities trong 1 bảng, dùng PK/SK prefix phân biệt.

Bảng Orders system:
┌─────────────────┬──────────────────────┬─────────────────────┐
│ PK              │ SK                   │ Attributes           │
├─────────────────┼──────────────────────┼─────────────────────┤
│ USER#123        │ PROFILE              │ name, email, ...     │
│ USER#123        │ ORDER#2024-001       │ total, status, ...   │
│ USER#123        │ ORDER#2024-002       │ total, status, ...   │
│ ORDER#2024-001  │ ITEM#1               │ product, qty, price  │
│ ORDER#2024-001  │ ITEM#2               │ product, qty, price  │
│ PRODUCT#ABC     │ PRODUCT#ABC          │ name, price, stock   │
└─────────────────┴──────────────────────┴─────────────────────┘

Access patterns served:
- Get user profile: Query PK=USER#123, SK=PROFILE
- Get user orders:  Query PK=USER#123, SK begins_with("ORDER#")
- Get order items:  Query PK=ORDER#2024-001, SK begins_with("ITEM#")
- All in 1 query:   Query PK=USER#123 (get profile + all orders)
```

---

## 4. GSI và LSI {#gsi-lsi}

```
LSI (Local Secondary Index):
- SAME partition key, DIFFERENT sort key
- Must be created at table creation time
- Shares provisioned throughput with table
- Use case: Query same partition with different sort

GSI (Global Secondary Index):
- DIFFERENT partition key AND sort key
- Can be created/deleted anytime
- Has own provisioned throughput
- Use case: Completely different access pattern

Example - Order status lookup (GSI):
GSI1: PK = status, SK = created_at
→ Query: Get all "pending" orders sorted by date

GSI overloading (generic attribute names):
┌──────────┬──────────┬───────────┬───────────┐
│ PK       │ SK       │ GSI1PK    │ GSI1SK    │
├──────────┼──────────┼───────────┼───────────┤
│ USER#123 │ ORDER#1  │ pending   │ 2024-01-15│  ← Order by status
│ USER#123 │ PROFILE  │ john@mail │ USER#123  │  ← User by email
│ PROD#ABC │ PROD#ABC │ electronics│ PROD#ABC │  ← Product by category
└──────────┴──────────┴───────────┴───────────┘
```

---

## 5-10. Access Patterns, Streams, TTL, Transactions, Best Practices {#access-patterns}

### DynamoDB Streams
```
Streams capture item-level changes (INSERT/UPDATE/DELETE):
- Trigger Lambda on changes (event-driven)
- Cross-region replication (Global Tables)
- Materialized views / aggregations
- Audit log

Stream record: OLD image, NEW image, or BOTH
Retention: 24 hours
```

### TTL
```
# Automatically delete expired items (no cost!)
# Set a number attribute with Unix epoch timestamp
# DynamoDB deletes items after expiration (within 48h)

{
  "PK": "SESSION#abc123",
  "SK": "SESSION#abc123",
  "user_id": "user-456",
  "ttl": 1706140800    ← Unix timestamp (2024-01-25 00:00:00)
}
# After Jan 25: item automatically deleted!
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| AWS DynamoDB Developer Guide | Official documentation |
| The DynamoDB Book (Alex DeBrie) | Best single-table design resource |
| re:Invent talks on DynamoDB | Rick Houlihan advanced patterns |
| dynamodb-toolbox | TypeScript SDK for single-table |

---

*Bài viết tiếp theo: [Database Replication & HA](/2026/08/24/database-replication-ha/)*
