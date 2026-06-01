---
layout: post
title: "Caching Strategies - Chiến lược bộ nhớ đệm"
date: 2026-06-01
categories: [database]
tags: [caching, redis, memcached, cache-aside, write-through, ttl]
---

## Mục lục
1. [Góc nhìn tổng quan - Quầy phục vụ nhanh](#overview)
2. [Cache-Aside (Lazy Loading)](#cache-aside)
3. [Write-Through](#write-through)
4. [Write-Behind (Write-Back)](#write-behind)
5. [TTL Strategies - Thời hạn cache](#ttl)
6. [Cache Invalidation - Xóa cache cũ](#invalidation)
7. [Redis vs Memcached](#redis-memcached)
8. [Hot Key Problem](#hot-key)
9. [Cache Stampede Prevention](#stampede)
10. [Tổng kết và decision guide](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

Caching giống **quầy phục vụ nhanh** ở nhà hàng:

- **Database** = bếp (nấu từ đầu = chậm nhưng tươi)
- **Cache** = quầy buffet (đồ sẵn = nhanh nhưng có thể không mới nhất)
- **Cache-aside** = "Khách hỏi món → check buffet → không có → gọi bếp → đặt lên buffet"
- **Write-through** = "Bếp nấu xong → ĐẶT LÊN BUFFET NGAY → phục vụ khách"
- **TTL** = "Đồ ở buffet quá 30 phút → bỏ, nấu lại"
- **Cache stampede** = "100 khách cùng hỏi 1 món hết hạn → 100 order xuống bếp!"
- **Hot key** = "Mọi người chỉ muốn 1 món → hết ngay!"

---

## 2. Cache-Aside (Lazy Loading) {#cache-aside}

```python
# Pseudocode: Cache-Aside pattern
def get_user(user_id):
    # 1. Check cache first
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)  # Cache HIT
    
    # 2. Cache miss → query database
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    
    # 3. Store in cache for next time
    redis.setex(f"user:{user_id}", 3600, json.dumps(user))  # TTL 1h
    
    return user

# Characteristics:
# ✅ Only caches what's actually requested (efficient memory)
# ✅ Cache failure = still works (fallback to DB)
# ❌ First request always slow (cache miss)
# ❌ Data can be stale (until TTL expires or invalidation)
```

---

## 3. Write-Through {#write-through}

```python
# Write-Through: Write to cache AND database simultaneously
def update_user(user_id, data):
    # 1. Write to database
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    
    # 2. Write to cache (immediately)
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
    
    return data

# ✅ Cache always consistent with DB
# ✅ No stale data on reads
# ❌ Write latency increased (write both)
# ❌ Caches data that may never be read (wasted memory)
```

---

## 4. Write-Behind (Write-Back) {#write-behind}

```python
# Write-Behind: Write to cache first, async flush to DB later
def update_user(user_id, data):
    # 1. Write to cache ONLY (fast!)
    redis.setex(f"user:{user_id}", 3600, json.dumps(data))
    
    # 2. Queue async write to database
    queue.enqueue("db_write", {"table": "users", "id": user_id, "data": data})
    
    return data  # Return immediately!

# Background worker:
def flush_to_db():
    while True:
        batch = queue.dequeue_batch(100)
        db.bulk_upsert(batch)  # Batch write = efficient

# ✅ Fastest writes (only cache, async DB)
# ✅ Batch DB writes (better throughput)
# ❌ Risk of data loss if cache crashes before flush
# ❌ Complex implementation
# ❌ DB temporarily inconsistent with cache
```

---

## 5-6. TTL và Invalidation {#ttl}

```python
# TTL Strategies:
# Short TTL (30s-5min): Frequently changing data (stock prices, sessions)
# Medium TTL (1h-24h): Semi-static data (user profiles, product info)
# Long TTL (days-weeks): Rarely changing (config, static content)

# Cache Invalidation patterns:
# 1. TTL expiry (simplest, eventual consistency)
# 2. Event-driven invalidation (on write → delete cache)
# 3. Pub/Sub invalidation (broadcast to all cache nodes)

def update_product(product_id, data):
    db.update(product_id, data)
    redis.delete(f"product:{product_id}")  # Invalidate!
    # OR
    redis.publish("cache_invalidation", f"product:{product_id}")
```

---

## 7. Redis vs Memcached {#redis-memcached}

```
┌──────────────┬───────────────────┬───────────────────┐
│ Feature      │ Redis             │ Memcached         │
├──────────────┼───────────────────┼───────────────────┤
│ Data types   │ String,Hash,List, │ String only       │
│              │ Set,Sorted Set    │                   │
│ Persistence  │ RDB + AOF        │ No (volatile)     │
│ Cluster      │ Redis Cluster    │ Client-side       │
│ Pub/Sub      │ Yes              │ No                │
│ Lua scripts  │ Yes              │ No                │
│ Memory       │ More overhead    │ More efficient    │
│ Multi-thread │ Single (6.x+MT) │ Multi-threaded    │
│ Use when     │ Complex caching, │ Simple K/V,       │
│              │ sessions, queues │ max memory efficiency│
└──────────────┴───────────────────┴───────────────────┘
```

---

## 8-9. Hot Key và Cache Stampede {#hot-key}

### Hot Key Problem
```
Problem: 1 key nhận 90% traffic → single Redis node overloaded

Solutions:
1. Local cache (in-process) + distributed cache (2-level)
2. Key replication across multiple nodes
3. Read-through with local cache (CDN approach)
```

### Cache Stampede Prevention
```python
# Problem: Key expires → 1000 concurrent requests → 1000 DB queries!

# Solution 1: Locking (only 1 request rebuilds cache)
def get_with_lock(key):
    value = redis.get(key)
    if value: return value
    
    # Try to acquire lock
    if redis.set(f"lock:{key}", "1", nx=True, ex=10):
        # Won the lock → rebuild cache
        value = db.query(...)
        redis.setex(key, 3600, value)
        redis.delete(f"lock:{key}")
        return value
    else:
        # Someone else is building → wait and retry
        time.sleep(0.1)
        return get_with_lock(key)

# Solution 2: Stale-while-revalidate
# Return stale value immediately, refresh in background

# Solution 3: Probabilistic early expiration
# Each read: if random() < probability(ttl_remaining), refresh
```

---

## 10. Tổng kết {#tong-ket}

| Strategy | Consistency | Write Speed | Read Speed | Complexity |
|----------|------------|-------------|------------|------------|
| Cache-Aside | Eventual | Normal | Fast (hit) | Low |
| Write-Through | Strong | Slower | Fast | Medium |
| Write-Behind | Eventual | Fastest | Fast | High |
| Read-Through | Eventual | Normal | Fast | Medium |

---
