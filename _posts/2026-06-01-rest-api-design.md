---
layout: post
title: "REST API Design - Resources, Methods, Status Codes, Versioning, Pagination, HATEOAS & Idempotency"
date: 2026-06-01
categories: [networking]
tags: [rest, api, http, design-patterns, web]
---

# REST API Design — Resources, Methods, Status Codes, Versioning, Pagination, HATEOAS & Idempotency

## Mục lục (Table of Contents)
1. [Giới thiệu bằng câu chuyện đời thường](#1-giới-thiệu-bằng-câu-chuyện-đời-thường)
2. [REST Fundamentals — Nguyên lý nền tảng](#2-rest-fundamentals--nguyên-lý-nền-tảng)
3. [Resources và URI Design](#3-resources-và-uri-design)
4. [HTTP Methods — Ngữ nghĩa của hành động](#4-http-methods--ngữ-nghĩa-của-hành-động)
5. [Status Codes — Ngôn ngữ phản hồi](#5-status-codes--ngôn-ngữ-phản-hồi)
6. [Versioning — Quản lý phiên bản API](#6-versioning--quản-lý-phiên-bản-api)
7. [Pagination — Phân trang dữ liệu](#7-pagination--phân-trang-dữ-liệu)
8. [HATEOAS — Hypermedia as the Engine of Application State](#8-hateoas--hypermedia-as-the-engine-of-application-state)
9. [Idempotency — Tính bất biến khi lặp lại](#9-idempotency--tính-bất-biến-khi-lặp-lại)
10. [Tổng kết và Best Practices](#10-tổng-kết-và-best-practices)

---

## 1. Giới thiệu bằng câu chuyện đời thường

### API như một thực đơn nhà hàng

Hãy tưởng tượng bạn đến một nhà hàng. Bạn không cần biết đầu bếp nấu ăn thế nào, bếp hoạt động ra sao, nguyên liệu lấy từ đâu. Bạn chỉ cần:

1. **Đọc thực đơn** (documentation) — biết có những món gì
2. **Gọi món** (request) — chọn món và nói cho bồi bàn
3. **Nhận món** (response) — nhận kết quả

**API (Application Programming Interface)** hoạt động y hệt vậy — nó là "thực đơn" giữa ứng dụng của bạn và server. **REST API** là kiểu thực đơn phổ biến nhất trên Internet.

| Nhà hàng | REST API |
|---|---|
| Thực đơn | API Documentation |
| Bồi bàn | HTTP Protocol |
| Gọi món "Phở bò" | `GET /api/dishes/pho-bo` |
| Thêm món mới vào thực đơn | `POST /api/dishes` |
| Đổi giá món | `PATCH /api/dishes/pho-bo` |
| Xóa món khỏi thực đơn | `DELETE /api/dishes/pho-bo` |
| Hóa đơn (status) | HTTP Status Code (200 OK, 404 Not Found) |
| "Hết món này rồi" | `404 Not Found` hoặc `410 Gone` |
| "Nhà hàng đông quá, xin chờ" | `429 Too Many Requests` |

### Tại sao REST API quan trọng?

- **95%+ web services** dùng REST (hoặc REST-like) API
- Mọi ứng dụng mobile, web, IoT đều gọi REST API
- Cloud services (AWS, Azure, GCP) expose REST API
- Hiểu REST giúp bạn **thiết kế API tốt** và **debug nhanh hơn**

---

## 2. REST Fundamentals — Nguyên lý nền tảng

### 2.1 REST là gì?

**REST (Representational State Transfer)** là một kiểu kiến trúc (architectural style) cho hệ thống phân tán, được Roy Fielding định nghĩa trong luận văn tiến sĩ năm 2000.

**Quan trọng**: REST **KHÔNG phải protocol**. Nó là tập hợp **ràng buộc kiến trúc (architectural constraints)**. Một API "RESTful" là API tuân thủ các ràng buộc này.

### 2.2 Sáu ràng buộc của REST

| # | Ràng buộc | Ý nghĩa | Ví dụ đời thường |
|---|---|---|---|
| 1 | **Client-Server** | Client và Server tách biệt | Khách hàng và nhà hàng là 2 thực thể riêng |
| 2 | **Stateless** | Mỗi request chứa ĐỦ thông tin, server không nhớ state | Mỗi lần gọi bồi bàn, bạn phải nhắc lại số bàn |
| 3 | **Cacheable** | Response có thể cache được | Thực đơn (ít thay đổi) có thể photocopy để bàn nào cũng có |
| 4 | **Uniform Interface** | Giao diện thống nhất cho mọi resource | Mọi món đều gọi cùng cách: "Cho tôi [tên món]" |
| 5 | **Layered System** | Client không biết đang nói với server nào | Bạn không cần biết có bao nhiêu bếp phó đang nấu |
| 6 | **Code on Demand** (optional) | Server có thể gửi code cho client chạy | Nhà hàng gửi hướng dẫn pha chế cocktail để bạn tự làm |

### 2.3 Stateless — Nguyên tắc quan trọng nhất

**Stateless** nghĩa là server **KHÔNG lưu trạng thái** giữa các request. Mỗi request phải tự túc (self-contained).

```
❌ Stateful (server nhớ):
Request 1: "Tôi là user ABC" → Server lưu: "à, user ABC đang ở"
Request 2: "Cho tôi xem orders" → Server: "user ABC muốn xem orders"

✅ Stateless (mỗi request đầy đủ):
Request 1: "Token: xyz123, GET /orders" → Server verify token, trả orders
Request 2: "Token: xyz123, GET /orders/5" → Server verify token, trả order #5
```

**Lợi ích của Stateless:**
- **Scalability**: Bất kỳ server nào cũng xử lý được (không cần sticky sessions)
- **Reliability**: Server restart không mất state
- **Simplicity**: Dễ debug, dễ test

### 2.4 Richardson Maturity Model — Đo mức "RESTful"

Leonard Richardson chia REST API thành 4 mức trưởng thành:

```
Level 3: Hypermedia Controls (HATEOAS)     ← "True REST"
         Links trong response hướng dẫn client
         
Level 2: HTTP Verbs                         ← Hầu hết API hiện tại
         Dùng đúng GET/POST/PUT/DELETE + status codes
         
Level 1: Resources                          ← Tốt hơn
         URL đại diện cho resources (/users/123)
         
Level 0: The Swamp of POX                   ← RPC-style
         Một URL, mọi thứ qua POST
         POST /api?action=getUser&id=123
```

---

## 3. Resources và URI Design

### 3.1 Resource là gì?

**Resource** là bất kỳ "thứ gì" mà API expose cho client. Nó có thể là:
- Một đối tượng cụ thể: user, order, product
- Một collection: danh sách users, danh sách orders
- Một khái niệm trừu tượng: search results, statistics

**Ví dụ đời thường**: Resource giống **danh mục trong thư viện** — mỗi cuốn sách là một resource, mỗi kệ sách là một collection.

### 3.2 URI Design Rules — Quy tắc thiết kế đường dẫn

| Quy tắc | Đúng ✅ | Sai ❌ | Lý do |
|---|---|---|---|
| Dùng **danh từ số nhiều** | `/users` | `/getUser`, `/user` | URI là "tên", không phải "hành động" |
| Dùng **kebab-case** | `/order-items` | `/orderItems`, `/order_items` | Chuẩn URL, dễ đọc |
| **Hierarchy** bằng nesting | `/users/123/orders` | `/getUserOrders?id=123` | Thể hiện quan hệ sở hữu |
| Không có **trailing slash** | `/users` | `/users/` | Consistency |
| Không có **file extension** | `/users/123` | `/users/123.json` | Dùng Accept header thay thế |
| Dùng **query params** cho filter | `/users?status=active` | `/active-users` | Linh hoạt, composable |

### 3.3 Resource Naming Patterns

```
# Collection (tập hợp)
GET /api/v1/users              → Danh sách tất cả users

# Singleton (một item cụ thể)
GET /api/v1/users/123          → User có ID 123

# Sub-resource (resource lồng nhau)
GET /api/v1/users/123/orders   → Tất cả orders của user 123
GET /api/v1/users/123/orders/456 → Order 456 của user 123

# Action resource (khi CRUD không đủ)
POST /api/v1/users/123/activate    → Kích hoạt user 123
POST /api/v1/orders/456/cancel     → Hủy order 456

# Search/Filter
GET /api/v1/users?role=admin&status=active
GET /api/v1/products?category=electronics&min_price=100&sort=-created_at
```

### 3.4 Singular vs. Plural — Khi nào dùng số ít

```
# Plural (số nhiều) — cho collections
GET  /users          → list
POST /users          → create
GET  /users/123      → read one
PUT  /users/123      → replace one
DELETE /users/123    → delete one

# Singular (số ít) — cho resource duy nhất gắn với context
GET  /users/123/profile    → Profile CỦA user 123 (chỉ có 1)
GET  /me                   → Current authenticated user
GET  /settings             → Application settings (singleton)
```

### 3.5 Anti-patterns — Những cách thiết kế tệ

```
❌ RPC-style (hành động trong URL):
POST /api/getUsers
POST /api/createUser
POST /api/deleteUser?id=123

❌ Quá deep nesting (>3 levels):
GET /api/countries/vn/cities/hcm/districts/1/wards/5/streets

✅ Flatten khi cần:
GET /api/streets?ward=5&district=1&city=hcm

❌ Dùng verbs trong URL:
GET /api/fetchAllProducts
POST /api/addToCart

✅ Dùng HTTP methods thay verbs:
GET /api/products
POST /api/cart/items
```

---

## 4. HTTP Methods — Ngữ nghĩa của hành động

### 4.1 Bảng HTTP Methods đầy đủ

| Method | Ý nghĩa | CRUD tương đương | Ví dụ đời thường |
|---|---|---|---|
| **GET** | Đọc resource | Read | Xem thực đơn |
| **POST** | Tạo resource mới | Create | Gọi món mới |
| **PUT** | Thay thế toàn bộ resource | Replace/Update | Đổi toàn bộ đơn hàng |
| **PATCH** | Cập nhật một phần resource | Partial Update | Sửa số lượng 1 món |
| **DELETE** | Xóa resource | Delete | Hủy món |
| **HEAD** | Giống GET nhưng không có body | Check existence | Kiểm tra món còn không (chỉ xem status) |
| **OPTIONS** | Xem methods nào được hỗ trợ | Discovery | Hỏi "bạn phục vụ gì?" |

### 4.2 Safe, Idempotent, và Cacheable

| Method | Safe? | Idempotent? | Cacheable? | Request Body? |
|---|---|---|---|---|
| GET | ✅ | ✅ | ✅ | ❌ (thường không) |
| HEAD | ✅ | ✅ | ✅ | ❌ |
| OPTIONS | ✅ | ✅ | ❌ | ❌ |
| POST | ❌ | ❌ | ❌ (thường) | ✅ |
| PUT | ❌ | ✅ | ❌ | ✅ |
| PATCH | ❌ | ❌* | ❌ | ✅ |
| DELETE | ❌ | ✅ | ❌ | ❌ (thường) |

> **Safe** (An toàn): Không thay đổi state trên server — chỉ đọc
> **Idempotent** (Bất biến): Gọi 1 lần hay N lần → kết quả giống nhau
> *PATCH có thể idempotent nếu thiết kế đúng, nhưng spec không bắt buộc

### 4.3 GET — Đọc dữ liệu

```http
# Lấy danh sách users
GET /api/v1/users?page=1&limit=20 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...
Accept: application/json

# Response
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: max-age=60
ETag: "abc123"

{
  "data": [
    {"id": 1, "name": "Nguyễn Văn A", "email": "a@example.com"},
    {"id": 2, "name": "Trần Thị B", "email": "b@example.com"}
  ],
  "meta": {"total": 150, "page": 1, "limit": 20}
}
```

**Rules cho GET:**
- KHÔNG thay đổi state (safe)
- Có thể cache (cacheable)
- Không có request body (theo chuẩn)
- Idempotent — gọi 100 lần cũng không đổi gì

### 4.4 POST — Tạo resource mới

```http
# Tạo user mới
POST /api/v1/users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGci...

{
  "name": "Lê Văn C",
  "email": "c@example.com",
  "role": "member"
}

# Response
HTTP/1.1 201 Created
Location: /api/v1/users/3
Content-Type: application/json

{
  "id": 3,
  "name": "Lê Văn C",
  "email": "c@example.com",
  "role": "member",
  "created_at": "2026-07-17T10:30:00Z"
}
```

**Rules cho POST:**
- KHÔNG idempotent — gọi 2 lần có thể tạo 2 resources
- Response nên có `201 Created` + `Location` header
- Body chứa resource vừa tạo (bao gồm server-generated fields)

### 4.5 PUT vs. PATCH — Replace vs. Partial Update

```http
# PUT — Thay thế TOÀN BỘ resource
# Giống như "xé bỏ form cũ, viết lại form mới"
PUT /api/v1/users/3 HTTP/1.1
Content-Type: application/json

{
  "name": "Lê Văn C",
  "email": "new-email@example.com",
  "role": "admin"
  // Phải gửi TẤT CẢ fields — field thiếu = bị xóa/set null
}

# Response: 200 OK (hoặc 204 No Content)
```

```http
# PATCH — Cập nhật MỘT PHẦN resource
# Giống như "dùng tẩy xóa 1 chỗ, viết lại"
PATCH /api/v1/users/3 HTTP/1.1
Content-Type: application/json

{
  "email": "new-email@example.com"
  // Chỉ gửi fields cần thay đổi — fields khác giữ nguyên
}

# Response: 200 OK
```

**So sánh PUT vs PATCH:**

| Tiêu chí | PUT | PATCH |
|---|---|---|
| Gửi gì? | Toàn bộ resource | Chỉ fields thay đổi |
| Field thiếu? | Bị xóa/null | Giữ nguyên |
| Idempotent? | ✅ Luôn | ❌ Không chắc |
| Khi nào dùng? | Replace toàn bộ | Sửa 1-2 fields |
| Bandwidth | Cao (gửi tất cả) | Thấp (chỉ delta) |

### 4.6 DELETE — Xóa resource

```http
# Xóa user
DELETE /api/v1/users/3 HTTP/1.1
Authorization: Bearer eyJhbGci...

# Response option 1: 204 No Content (xóa thành công, không body)
HTTP/1.1 204 No Content

# Response option 2: 200 OK (trả lại resource vừa xóa)
HTTP/1.1 200 OK
{"id": 3, "name": "Lê Văn C", "deleted_at": "2026-07-17T11:00:00Z"}
```

**Idempotent?** Gọi DELETE lần 2:
- Option A: Return `404 Not Found` (resource đã xóa rồi)
- Option B: Return `204 No Content` (operation thành công — resource không tồn tại = mục tiêu đạt được)
- Cả hai đều chấp nhận được, option B "thuần" idempotent hơn

---

## 5. Status Codes — Ngôn ngữ phản hồi

### 5.1 Phân loại Status Codes

| Nhóm | Ý nghĩa | Ví dụ đời thường |
|---|---|---|
| **1xx** Informational | Đã nhận request, đang xử lý | "Đơn hàng của bạn đang được xử lý" |
| **2xx** Success | Thành công | "Đây là món bạn đặt" |
| **3xx** Redirection | Client cần đi nơi khác | "Nhà hàng chuyển địa chỉ, xin đến XYZ" |
| **4xx** Client Error | Client sai | "Bạn gọi món không có trong menu" |
| **5xx** Server Error | Server sai | "Bếp cháy, xin lỗi!" |

### 5.2 Status Codes quan trọng cho API

**2xx — Thành công:**

| Code | Tên | Khi nào dùng | HTTP Method thường dùng |
|---|---|---|---|
| **200** | OK | Request thành công, có response body | GET, PUT, PATCH |
| **201** | Created | Resource mới được tạo | POST |
| **202** | Accepted | Request được nhận, đang xử lý async | POST (long-running) |
| **204** | No Content | Thành công, không có body | DELETE, PUT |

**3xx — Redirect:**

| Code | Tên | Khi nào dùng |
|---|---|---|
| **301** | Moved Permanently | Resource đã chuyển vĩnh viễn |
| **304** | Not Modified | Cache còn hợp lệ (ETag match) |
| **307** | Temporary Redirect | Chuyển tạm, giữ nguyên method |
| **308** | Permanent Redirect | Chuyển vĩnh viễn, giữ nguyên method |

**4xx — Client Error:**

| Code | Tên | Khi nào dùng | Ví dụ |
|---|---|---|---|
| **400** | Bad Request | Request sai cú pháp | JSON không hợp lệ |
| **401** | Unauthorized | Chưa xác thực | Thiếu token |
| **403** | Forbidden | Đã xác thực nhưng không có quyền | User thường xem admin page |
| **404** | Not Found | Resource không tồn tại | `/users/999` (user 999 không có) |
| **405** | Method Not Allowed | Method không được hỗ trợ | DELETE trên resource read-only |
| **409** | Conflict | Xung đột state | Tạo user với email đã tồn tại |
| **422** | Unprocessable Entity | Cú pháp đúng, semantics sai | Email format sai |
| **429** | Too Many Requests | Rate limit exceeded | Gọi quá 100 req/phút |

**5xx — Server Error:**

| Code | Tên | Khi nào dùng |
|---|---|---|
| **500** | Internal Server Error | Lỗi không xác định (bug) |
| **502** | Bad Gateway | Upstream server lỗi |
| **503** | Service Unavailable | Server đang maintenance |
| **504** | Gateway Timeout | Upstream server timeout |

### 5.3 Error Response Format — Chuẩn hóa lỗi

**RFC 7807 — Problem Details for HTTP APIs:**

```json
{
  "type": "https://api.example.com/errors/validation-failed",
  "title": "Validation Failed",
  "status": 422,
  "detail": "Email field is required and must be a valid email address",
  "instance": "/api/v1/users",
  "errors": [
    {
      "field": "email",
      "message": "must be a valid email address",
      "value": "not-an-email"
    },
    {
      "field": "name",
      "message": "must be at least 2 characters",
      "value": "A"
    }
  ],
  "trace_id": "req-abc123-def456"
}
```

**Best practices cho error response:**
1. Luôn dùng JSON (không plain text)
2. Có `trace_id` để correlate với server logs
3. Có `detail` đủ rõ để developer hiểu
4. KHÔNG leak internal details (stack trace, SQL query) ra production

### 5.4 Phân biệt 401 vs 403

| | 401 Unauthorized | 403 Forbidden |
|---|---|---|
| Ý nghĩa | "Tôi không biết bạn là ai" | "Tôi biết bạn là ai, nhưng bạn không được phép" |
| Authentication | ❌ Chưa xác thực | ✅ Đã xác thực |
| Authorization | N/A | ❌ Không có quyền |
| Client nên làm gì? | Gửi lại với credentials | Không gửi lại (vẫn bị từ chối) |
| Header thường kèm | `WWW-Authenticate` | Không |
| Ví dụ | Gọi API không có Bearer token | User "member" gọi endpoint admin |

---

## 6. Versioning — Quản lý phiên bản API

### 6.1 Tại sao cần versioning?

**Ví dụ đời thường**: Bạn vận hành nhà hàng với 1000 khách quen. Nếu bạn đổi thực đơn đột ngột (bỏ món cũ, thêm món mới), khách quen sẽ bối rối. Bạn cần giữ thực đơn cũ (version 1) cho khách quen, đồng thời giới thiệu thực đơn mới (version 2) cho khách mới.

**API cũng vậy**: Khi bạn thay đổi response format, bỏ field, đổi logic → client cũ sẽ break. Versioning cho phép client cũ tiếp tục hoạt động.

### 6.2 Breaking Changes vs. Non-breaking Changes

| Breaking Change (cần version mới) | Non-breaking (an toàn) |
|---|---|
| Xóa field khỏi response | Thêm field mới vào response |
| Đổi kiểu dữ liệu (string → number) | Thêm endpoint mới |
| Đổi ý nghĩa field | Thêm optional query parameter |
| Xóa endpoint | Thêm optional request field |
| Đổi tên field | Deprecate (đánh dấu sắp xóa) |
| Bắt buộc field mới trong request | Mở rộng enum values |

### 6.3 Các chiến lược versioning

**Strategy 1: URI Path Versioning (Phổ biến nhất)**
```
GET /api/v1/users/123
GET /api/v2/users/123
```

| Ưu điểm | Nhược điểm |
|---|---|
| Rõ ràng, dễ hiểu | URL thay đổi → cache bị invalidate |
| Dễ route (load balancer/gateway) | Không "RESTful" thuần (URI thay đổi cho cùng resource) |
| Dễ document | Phải maintain nhiều code paths |

**Strategy 2: Query Parameter**
```
GET /api/users/123?version=2
GET /api/users/123?v=2
```

| Ưu điểm | Nhược điểm |
|---|---|
| URI giữ nguyên | Dễ quên, dễ sai |
| Có default version | Khó route ở infrastructure level |

**Strategy 3: Custom Header**
```http
GET /api/users/123 HTTP/1.1
X-API-Version: 2
# hoặc
Api-Version: 2
```

| Ưu điểm | Nhược điểm |
|---|---|
| URI sạch | Không thấy version trong URL |
| Linh hoạt | Khó test (cần tool để set header) |

**Strategy 4: Accept Header (Content Negotiation)**
```http
GET /api/users/123 HTTP/1.1
Accept: application/vnd.example.v2+json
```

| Ưu điểm | Nhược điểm |
|---|---|
| Chuẩn HTTP nhất | Phức tạp |
| Có thể version từng resource | Ít tooling hỗ trợ |

### 6.4 Khuyến nghị trong thực tế

```
# Public API (external consumers): URI Path Versioning
# → Rõ ràng nhất, dễ communicate
GET /api/v1/users

# Internal API (microservices): Header hoặc không version
# → Dễ update đồng bộ

# Deprecation Policy:
# v1 maintained 12 months after v2 launch
# Sunset header (RFC 8594):
HTTP/1.1 200 OK
Sunset: Sat, 01 Jan 2027 00:00:00 GMT
Deprecation: true
Link: <https://api.example.com/docs/migration-v2>; rel="successor-version"
```

---

## 7. Pagination — Phân trang dữ liệu

### 7.1 Tại sao cần phân trang?

**Ví dụ đời thường**: Nếu bạn hỏi thư viện "cho tôi xem TẤT CẢ sách", thủ thư sẽ sụp đổ. Thay vào đó, thủ thư đưa bạn xem từng kệ (page) một.

Tương tự, nếu `GET /users` trả về 1 triệu users cùng lúc:
- Server: RAM explode, query chậm
- Network: Transfer hàng GB data
- Client: Parse timeout, UI lag

**Pagination** chia kết quả thành "trang" nhỏ, trả về từng trang.

### 7.2 Offset-based Pagination (Phân trang theo offset)

**Ý tưởng**: "Bỏ qua N mục đầu, lấy M mục tiếp theo" — giống đọc sách: "đi tới trang 5, đọc 20 dòng".

```http
# Trang 1: Bỏ qua 0, lấy 20
GET /api/users?offset=0&limit=20

# Trang 2: Bỏ qua 20, lấy 20
GET /api/users?offset=20&limit=20

# Trang 5: Bỏ qua 80, lấy 20
GET /api/users?offset=80&limit=20
```

**SQL tương đương:**
```sql
SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 80;
```

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "total": 1500,
    "limit": 20,
    "offset": 80,
    "has_more": true
  }
}
```

**Ưu điểm:**
- ✅ Đơn giản, dễ hiểu
- ✅ Có thể nhảy tới bất kỳ trang nào (random access)
- ✅ Biết tổng số trang (`total / limit`)
- ✅ Phổ biến, mọi database đều hỗ trợ

**Nhược điểm:**
- ❌ **Performance kém ở offset lớn**: `OFFSET 1000000` vẫn phải scan 1 triệu rows
- ❌ **Data inconsistency**: Nếu có INSERT/DELETE giữa 2 request → bị lặp hoặc bỏ sót item
- ❌ **COUNT(*) expensive**: Đếm total tốn kém với dataset lớn

```
Vấn đề inconsistency:
Trang 1: items [1,2,3,4,5,...,20]
         ↓ Giữa 2 request, item #3 bị xóa
Trang 2: items [22,23,...,41] ← Item #21 bị "bỏ qua" vì offset shift
```

### 7.3 Cursor-based Pagination (Phân trang theo con trỏ)

**Ý tưởng**: "Cho tôi 20 mục SAU mục X" — giống bookmark trong sách: "đọc tiếp từ chỗ này".

**Cursor** là một giá trị opaque (mã hóa) đại diện cho vị trí trong dataset. Client không cần biết cursor chứa gì — chỉ cần gửi lại cho server.

```http
# Request đầu tiên (không có cursor)
GET /api/users?limit=20

# Response
{
  "data": [...20 items...],
  "pagination": {
    "next_cursor": "eyJpZCI6MjB9",   // Base64 encode of {"id": 20}
    "has_more": true
  }
}

# Request tiếp theo (gửi cursor)
GET /api/users?limit=20&after=eyJpZCI6MjB9

# Response
{
  "data": [...next 20 items...],
  "pagination": {
    "next_cursor": "eyJpZCI6NDB9",
    "prev_cursor": "eyJpZCI6MjF9",
    "has_more": true
  }
}
```

**SQL tương đương (Keyset pagination):**
```sql
-- Thay vì OFFSET (chậm):
SELECT * FROM users WHERE id > 20 ORDER BY id LIMIT 20;
-- ← Dùng index trên `id`, cực nhanh bất kể vị trí
```

**Cursor encoding:**
```python
import base64, json

# Encode cursor
cursor_data = {"id": 20, "created_at": "2026-07-17T10:00:00Z"}
cursor = base64.b64encode(json.dumps(cursor_data).encode()).decode()
# → "eyJpZCI6IDIwLCAiY3JlYXRlZF9hdCI6ICIyMDI2LTA3LTE3VDEwOjAwOjAwWiJ9"

# Decode cursor (server-side)
cursor_data = json.loads(base64.b64decode(cursor))
# → {"id": 20, "created_at": "2026-07-17T10:00:00Z"}
```

**Ưu điểm:**
- ✅ **Performance nhất quán** (O(1) bất kể position)
- ✅ **Consistent**: Không bị trùng/mất item khi data thay đổi
- ✅ **Real-time friendly**: Hoạt động tốt với live data feeds
- ✅ Không cần COUNT(*) tốn kém

**Nhược điểm:**
- ❌ **Không random access**: Không thể nhảy tới "trang 50"
- ❌ **Phức tạp hơn**: Client phải lưu cursor
- ❌ **Khó hiển thị page numbers**: UI kiểu "trang 1 2 3 4 5" khó làm
- ❌ **Cursor có thể expire**: Nếu item tham chiếu bị xóa

### 7.4 So sánh Offset vs Cursor

| Tiêu chí | Offset | Cursor |
|---|---|---|
| Performance (dataset lớn) | ❌ Chậm dần | ✅ Nhất quán |
| Random access (nhảy trang) | ✅ Dễ dàng | ❌ Không thể |
| Data consistency | ❌ Có thể trùng/mất | ✅ Ổn định |
| Đơn giản | ✅ Rất đơn giản | ❌ Phức tạp hơn |
| UI page numbers | ✅ Dễ | ❌ Khó |
| Real-time feeds | ❌ Dễ lỗi | ✅ Phù hợp |
| Use case tốt nhất | Admin dashboard, search results | Social feeds, logs, infinite scroll |

### 7.5 Page-based Pagination (biến thể của Offset)

```http
# Thay vì offset/limit, dùng page/per_page
GET /api/users?page=3&per_page=20

# Response
{
  "data": [...],
  "meta": {
    "current_page": 3,
    "per_page": 20,
    "total_pages": 75,
    "total_count": 1500
  },
  "links": {
    "first": "/api/users?page=1&per_page=20",
    "prev": "/api/users?page=2&per_page=20",
    "next": "/api/users?page=4&per_page=20",
    "last": "/api/users?page=75&per_page=20"
  }
}
```

### 7.6 Pagination Best Practices

1. **Luôn có max limit** — Client request `limit=1000000` → server cap ở 100
2. **Default limit hợp lý** — 20-50 cho UI, 100-1000 cho internal
3. **Luôn sort ổn định** — ORDER BY phải unique (thêm `id` nếu cần)
4. **Trả metadata** — total count (nếu rẻ), has_more flag, links
5. **Document rõ ràng** — Cursor format, max limit, sort options

---

## 8. HATEOAS — Hypermedia as the Engine of Application State

### 8.1 HATEOAS là gì?

**HATEOAS** (phát âm: "hate-ee-oh-ass") là ràng buộc cuối cùng và "thuần túy" nhất của REST. Ý tưởng: **Response nên chứa links hướng dẫn client biết có thể làm gì tiếp theo.**

**Ví dụ đời thường**: Khi bạn vào một website, bạn không cần biết trước tất cả URLs. Bạn bắt đầu từ trang chủ, click link để đi tiếp. HATEOAS áp dụng nguyên tắc tương tự cho API.

**Giống đi siêu thị:**
- Vào cửa → thấy bảng chỉ dẫn "Rau quả: lối 1, Thịt: lối 2"
- Đến lối 1 → thấy "Rau xanh: kệ A, Hoa quả: kệ B"
- Bạn KHÔNG CẦN biết trước layout siêu thị — bảng chỉ dẫn hướng dẫn bạn

### 8.2 API không có HATEOAS vs. có HATEOAS

**Không có HATEOAS (hầu hết API hiện tại):**
```json
{
  "id": 123,
  "name": "Nguyễn Văn A",
  "status": "active",
  "order_count": 5
}
// Client phải "biết trước":
// - URL xem orders: /api/users/123/orders
// - URL deactivate: POST /api/users/123/deactivate
// - URL update: PUT /api/users/123
// → Hardcode logic trong client
```

**Có HATEOAS:**
```json
{
  "id": 123,
  "name": "Nguyễn Văn A",
  "status": "active",
  "order_count": 5,
  "_links": {
    "self": {"href": "/api/users/123", "method": "GET"},
    "update": {"href": "/api/users/123", "method": "PUT"},
    "delete": {"href": "/api/users/123", "method": "DELETE"},
    "orders": {"href": "/api/users/123/orders", "method": "GET"},
    "deactivate": {"href": "/api/users/123/deactivate", "method": "POST"}
  }
}
```

### 8.3 Lợi ích thực tế của HATEOAS

| Lợi ích | Giải thích |
|---|---|
| **Discoverability** | Client tìm được actions từ response, không cần hardcode URLs |
| **Decoupling** | Server đổi URL → client tự cập nhật (follow links) |
| **Self-documenting** | Response tự giải thích "bạn có thể làm gì" |
| **Conditional actions** | Chỉ show "deactivate" link nếu user đang active |

**Conditional links — Ẩn/hiện action theo state:**
```json
// User đang active → show deactivate, KHÔNG show activate
{
  "status": "active",
  "_links": {
    "deactivate": {"href": "/api/users/123/deactivate"}
    // KHÔNG có "activate" vì user đã active
  }
}

// User đang inactive → show activate, KHÔNG show deactivate
{
  "status": "inactive",
  "_links": {
    "activate": {"href": "/api/users/123/activate"}
    // KHÔNG có "deactivate" vì user đã inactive
  }
}
```

### 8.4 HAL (Hypertext Application Language) Format

**HAL** (RFC draft) là format phổ biến nhất cho HATEOAS:

```json
{
  "_links": {
    "self": {"href": "/api/orders/523"},
    "warehouse": {"href": "/api/warehouses/56"},
    "invoice": {"href": "/api/orders/523/invoice"}
  },
  "_embedded": {
    "items": [
      {
        "_links": {"self": {"href": "/api/products/1"}},
        "name": "Widget A",
        "quantity": 2,
        "price": 9.99
      }
    ]
  },
  "id": 523,
  "currency": "USD",
  "status": "processing",
  "total": 19.98
}
```

### 8.5 HATEOAS trong thực tế

**Thực tế**: Rất ít API thực sự implement HATEOAS đầy đủ. Hầu hết API dừng ở Level 2 (Richardson).

**Ai dùng HATEOAS?**
- GitHub API (partial — có pagination links)
- PayPal API (full HATEOAS)
- Spring HATEOAS (framework support)
- AWS API Gateway (partial)

**Tại sao ít dùng?**
- Overhead: Mỗi response lớn hơn (thêm _links)
- Complexity: Client phải parse links thay vì hardcode
- Trong thực tế: Client và Server thường develop cùng team → documentation đủ

---

## 9. Idempotency — Tính bất biến khi lặp lại

### 9.1 Idempotency là gì?

**Idempotent** (bất biến lũy đẳng): Một operation gọi 1 lần hay N lần đều cho **cùng kết quả** và **cùng side effects**.

**Ví dụ đời thường:**
- ✅ Idempotent: **Bật đèn** — bấm công tắc 1 lần: đèn sáng. Bấm 10 lần: vẫn sáng. Kết quả cuối cùng giống nhau.
- ❌ Không idempotent: **Rót nước** — rót 1 lần: 1 ly. Rót 10 lần: 10 ly. Kết quả khác nhau mỗi lần.

**Trong API:**
- ✅ `DELETE /users/123` — Gọi 1 lần: user bị xóa. Gọi 10 lần: user vẫn bị xóa (chỉ 1 lần thực sự xóa, 9 lần sau là no-op hoặc 404).
- ❌ `POST /orders` — Gọi 1 lần: tạo 1 order. Gọi 10 lần: tạo 10 orders!

### 9.2 Tại sao Idempotency quan trọng?

**Scenario thực tế — Network retry:**

```
Client ──POST /payments──→ [Network timeout]
                           Server NHẬN request, xử lý, 
                           trừ tiền thành công
                           Nhưng response bị mất!
                           
Client: "Hmm, timeout, retry?"
Client ──POST /payments──→ Server
                           Trừ tiền LẦN 2?? 💀
```

**Nếu không idempotent**: User bị trừ tiền 2 lần!
**Nếu có idempotency**: Server nhận ra đây là retry, trả lại kết quả cũ (không trừ tiền lần 2).

### 9.3 Idempotency Key Pattern

**Implementation**: Client gửi một **unique key** với mỗi request. Server lưu key + result. Nếu nhận request với key đã xử lý → trả lại result cũ.

```http
# Request lần 1
POST /api/payments HTTP/1.1
Idempotency-Key: pay_req_abc123
Content-Type: application/json

{"amount": 100, "currency": "USD", "recipient": "user_456"}

# Response lần 1
HTTP/1.1 201 Created
{"id": "pay_789", "status": "completed", "amount": 100}
```

```http
# Request lần 2 (retry với cùng key)
POST /api/payments HTTP/1.1
Idempotency-Key: pay_req_abc123
Content-Type: application/json

{"amount": 100, "currency": "USD", "recipient": "user_456"}

# Response lần 2 — CÙNG kết quả, KHÔNG tạo payment mới
HTTP/1.1 200 OK
{"id": "pay_789", "status": "completed", "amount": 100}
```

### 9.4 Server-side Implementation

```python
# Pseudocode — Idempotency middleware

class IdempotencyMiddleware:
    def process_request(self, request):
        key = request.headers.get("Idempotency-Key")
        
        if not key:
            return None  # No idempotency key → process normally
        
        # Check if we've seen this key before
        cached = redis.get(f"idempotency:{key}")
        
        if cached:
            # Already processed — return cached response
            return cached_response(cached)
        
        # First time — acquire lock to prevent race condition
        lock = redis.lock(f"idempotency_lock:{key}", timeout=30)
        
        if not lock.acquire(blocking_timeout=5):
            return Response(status=409, body="Request in progress")
        
        try:
            # Process request
            response = process_actual_request(request)
            
            # Cache response (TTL = 24 hours)
            redis.setex(f"idempotency:{key}", 86400, serialize(response))
            
            return response
        finally:
            lock.release()
```

### 9.5 Stripe's Idempotency Pattern (Thực tế industry)

Stripe (payment processor) là ví dụ mẫu mực:

```bash
# Stripe API
curl https://api.stripe.com/v1/charges \
  -u sk_test_xxx: \
  -H "Idempotency-Key: $(uuidgen)" \
  -d amount=2000 \
  -d currency=usd \
  -d source=tok_visa

# Rules:
# 1. Key valid 24 hours
# 2. Cùng key + KHÁC body → 400 error (misuse detection)
# 3. Cùng key + cùng body → return cached result
# 4. Key hết hạn → request được xử lý lại
```

### 9.6 Idempotency Matrix

| Method | Idempotent theo spec? | Cần Idempotency Key? | Lý do |
|---|---|---|---|
| GET | ✅ Always | ❌ Không | Chỉ đọc |
| PUT | ✅ Always | ❌ Không cần (inherent) | "Set state = X" idempotent by design |
| DELETE | ✅ Always | ❌ Không cần | "Resource không tồn tại" = mục tiêu đạt |
| POST | ❌ Never | ✅ CẦN | "Tạo mới" → mỗi lần tạo 1 resource |
| PATCH | ❌ Depends | ⚠️ Nên có | Depends on patch operation |

### 9.7 Idempotency vs. Safety vs. Nullipotent

| Thuộc tính | Định nghĩa | Ví dụ |
|---|---|---|
| **Safe** | Không thay đổi server state | GET, HEAD, OPTIONS |
| **Idempotent** | N lần = 1 lần (same result) | GET, PUT, DELETE |
| **Nullipotent** | Không có side effects nào | Tính toán thuần (2+2=4) |

**Quan hệ**: Safe ⊂ Idempotent (Mọi safe method đều idempotent, nhưng không ngược lại)

---

## 10. Tổng kết và Best Practices

### 10.1 Checklist thiết kế REST API

```
□ Resources: Danh từ số nhiều, kebab-case
□ Methods: Đúng ngữ nghĩa (GET=read, POST=create, PUT=replace, PATCH=partial, DELETE=remove)
□ Status Codes: Chính xác (201 Created, 404 Not Found, 409 Conflict...)
□ Versioning: Chọn 1 strategy, áp dụng nhất quán
□ Pagination: Cursor cho feeds, Offset cho search/admin
□ Error format: Chuẩn hóa (RFC 7807 hoặc custom schema)
□ Idempotency: Idempotency-Key cho POST endpoints quan trọng
□ Authentication: Bearer token (OAuth2/JWT)
□ Rate Limiting: 429 status + Retry-After header
□ CORS: Preflight OPTIONS handling
□ Content Negotiation: Accept/Content-Type headers
□ Documentation: OpenAPI/Swagger spec
```

### 10.2 Request/Response Conventions

```http
# Standard Request Headers
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
Idempotency-Key: <uuid>              # Cho POST
If-None-Match: "etag-value"          # Conditional GET (caching)
If-Match: "etag-value"               # Conditional PUT (optimistic lock)

# Standard Response Headers
Content-Type: application/json
ETag: "abc123"
Cache-Control: max-age=60, private
X-Request-Id: req-uuid-123            # Tracing
X-RateLimit-Limit: 100               # Rate limit info
X-RateLimit-Remaining: 45
X-RateLimit-Reset: 1721234567
Retry-After: 30                       # Seconds to wait (429/503)
```

### 10.3 URL Design Cheat Sheet

```
# CRUD operations
GET    /api/v1/users              # List users (with pagination)
POST   /api/v1/users              # Create user
GET    /api/v1/users/:id          # Get single user
PUT    /api/v1/users/:id          # Replace user
PATCH  /api/v1/users/:id          # Partial update
DELETE /api/v1/users/:id          # Delete user

# Sub-resources
GET    /api/v1/users/:id/orders   # User's orders
POST   /api/v1/users/:id/orders   # Create order for user

# Actions (when CRUD doesn't fit)
POST   /api/v1/users/:id/verify-email
POST   /api/v1/orders/:id/cancel
POST   /api/v1/payments/:id/refund

# Search/Filter
GET    /api/v1/users?role=admin&status=active
GET    /api/v1/products?q=phone&category=electronics&sort=-price

# Batch operations
POST   /api/v1/users/batch-delete
PATCH  /api/v1/users/batch-update
```

### 10.4 Common Anti-Patterns to Avoid

| Anti-pattern | Problem | Better approach |
|---|---|---|
| Chatty API (N+1 calls) | 100 users = 101 requests | Include related data, sparse fieldsets |
| God endpoints (`/api/do-everything`) | Unmaintainable | Small, focused resources |
| Inconsistent naming | `/getUsers`, `/user_list`, `/fetch-accounts` | Pick one convention, stick to it |
| Leaking internal IDs | Auto-increment IDs expose business info | UUIDs or hashids |
| No pagination | Returns all data | Always paginate collections |
| 200 OK with error in body | Client thinks success | Use proper 4xx/5xx status codes |
| Ignoring Accept headers | Always returns JSON | Content negotiation |

### 10.5 Tài liệu tham khảo (References)

| Tài liệu | Nội dung |
|---|---|
| Roy Fielding's Dissertation (2000) | REST architectural style definition |
| RFC 7231 | HTTP/1.1 Semantics and Content |
| RFC 7232 | HTTP/1.1 Conditional Requests |
| RFC 7807 | Problem Details for HTTP APIs |
| RFC 8594 | Sunset Header |
| RFC 5988 | Web Linking (for HATEOAS) |
| Microsoft REST API Guidelines | Industry best practices |
| Google API Design Guide | Resource-oriented design |
| Stripe API Reference | Industry-leading API design |
| JSON:API Specification | Standardized JSON response format |

### 10.6 Câu hỏi ôn tập

1. REST có phải là protocol không? Nó là gì?
2. Sáu ràng buộc (constraints) của REST là gì?
3. PUT và PATCH khác nhau thế nào? Cho ví dụ.
4. Khi nào dùng 401 vs 403? 400 vs 422?
5. Offset pagination và Cursor pagination: ưu/nhược điểm mỗi loại?
6. HATEOAS giải quyết vấn đề gì? Tại sao ít API dùng?
7. Idempotency key pattern hoạt động thế nào? Tại sao POST cần nó?
8. Nêu 3 breaking changes và 3 non-breaking changes trong API.
9. Richardson Maturity Model có mấy level? API bạn đang dùng ở level nào?
10. Thiết kế endpoints cho hệ thống quản lý thư viện (books, authors, borrowing).

---

*Bài viết được tham khảo từ Roy Fielding's Dissertation (2000), RFC 7231 (HTTP Semantics), RFC 7807 (Problem Details), Microsoft REST API Guidelines, Google API Design Guide, và Stripe API documentation.*
