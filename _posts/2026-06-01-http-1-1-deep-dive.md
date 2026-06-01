---
layout: post
title: "HTTP/1.1 Deep Dive - Persistent Connections, Pipelining, Chunked Transfer và Caching"
date: 2026-06-01
categories: [networking]
tags: [http, http1.1, persistent-connection, keep-alive, caching, chunked-transfer, web]
---

## 1. Giới thiệu — Ngôn ngữ của Web

Hãy tưởng tượng bạn đến **quầy thông tin** ở trung tâm thương mại:

**HTTP/1.0 (cách cũ):**
1. Xếp hàng → đến lượt → hỏi "Tầng 3 bán gì?" → nhận trả lời → **ĐI RA**
2. Xếp hàng LẠI → "Giờ mở cửa?" → trả lời → **ĐI RA** 
3. Xếp hàng LẠI → "Có khuyến mãi không?" → trả lời → **ĐI RA**
4. Mỗi câu hỏi = **1 lần xếp hàng** (1 TCP connection)!

**HTTP/1.1 (cải tiến):**
1. Xếp hàng → đến lượt → **GIỮ CHỖ** (persistent connection)
2. "Tầng 3 bán gì?" → trả lời
3. "Giờ mở cửa?" → trả lời (CÙNG lượt, không xếp lại!)
4. "Có khuyến mãi?" → trả lời
5. "Tôi xong rồi" → ĐI RA

**HTTP/1.1** = "Giữ kết nối mở" để gửi nhiều request mà không cần tạo TCP connection mới mỗi lần.

### HTTP/1.1 trong cuộc sống hàng ngày

Khi bạn mở 1 trang web:
```
Trang web cần tải:
  - 1 file HTML
  - 15 images
  - 5 CSS files
  - 10 JavaScript files
  - 3 fonts
  = 34 resources total!

HTTP/1.0: 34 TCP connections × (handshake + request + close)
  = 34 × 3-way handshake = rất chậm!

HTTP/1.1: 1-6 TCP connections, mỗi connection gửi nhiều requests
  = Nhanh hơn nhiều!
```

### Lịch sử HTTP

| Version | Năm | RFC | Key Feature |
|---|---|---|---|
| HTTP/0.9 | 1991 | — | GET only, no headers |
| HTTP/1.0 | 1996 | RFC 1945 | Headers, methods, status codes |
| **HTTP/1.1** | **1997** | **RFC 2068 → 7230-7235 → 9110-9112** | **Persistent, Host header, chunked** |
| HTTP/2 | 2015 | RFC 7540 → 9113 | Binary, multiplexing, HPACK |
| HTTP/3 | 2022 | RFC 9114 | QUIC, no HoL blocking |

---

## 2. HTTP/1.1 Message Format — Request & Response

### Phép so sánh — Đơn đặt hàng và Hóa đơn

**HTTP Request** = Đơn đặt hàng:
- Dòng đầu: "Tôi muốn [hành động] [món gì] [phiên bản menu]"
- Headers: Thông tin thêm (kích cỡ, khẩu vị, dị ứng...)
- Body: Chi tiết đặc biệt (nếu có)

**HTTP Response** = Hóa đơn + Hàng:
- Dòng đầu: "Phiên bản menu [mã trạng thái] [mô tả]"
- Headers: Thông tin về hàng (kích cỡ, hạn sử dụng, loại...)
- Body: Hàng hóa thực tế (HTML, JSON, image...)

### Request Format

```
METHOD SP Request-URI SP HTTP-Version CRLF
Header-Field: Value CRLF
Header-Field: Value CRLF
CRLF
[Message Body]
```

```http
GET /api/users?page=1 HTTP/1.1
Host: api.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)
Accept: application/json
Accept-Language: vi-VN,vi;q=0.9,en;q=0.8
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
If-None-Match: "abc123"
Cache-Control: max-age=0

```

### Response Format

```
HTTP-Version SP Status-Code SP Reason-Phrase CRLF
Header-Field: Value CRLF
Header-Field: Value CRLF
CRLF
[Message Body]
```

```http
HTTP/1.1 200 OK
Date: Mon, 05 Jul 2026 10:30:00 GMT
Server: nginx/1.25.0
Content-Type: application/json; charset=utf-8
Content-Length: 256
Cache-Control: public, max-age=3600
ETag: "abc123"
X-Request-Id: req-12345
Connection: keep-alive
Keep-Alive: timeout=60, max=100

{"users": [{"id": 1, "name": "Nguyễn Văn A"}, ...]}
```

### HTTP Methods

| Method | Idempotent | Safe | Body | Use Case |
|---|---|---|---|---|
| **GET** | ✅ | ✅ | No | Lấy resource |
| **HEAD** | ✅ | ✅ | No | Lấy headers only (check existence) |
| **POST** | ❌ | ❌ | Yes | Tạo resource mới |
| **PUT** | ✅ | ❌ | Yes | Thay thế resource |
| **PATCH** | ❌ | ❌ | Yes | Cập nhật một phần |
| **DELETE** | ✅ | ❌ | Optional | Xóa resource |
| **OPTIONS** | ✅ | ✅ | No | Kiểm tra capabilities (CORS preflight) |
| **TRACE** | ✅ | ✅ | No | Loop-back diagnostic |
| **CONNECT** | ❌ | ❌ | — | Establish tunnel (HTTPS proxy) |

### Status Codes — Nhóm và ý nghĩa

```
1xx — Informational (tiếp tục)
  100 Continue         — "OK, gửi body tiếp đi"
  101 Switching Protocols — "Đổi sang WebSocket/HTTP2"

2xx — Success (thành công)
  200 OK               — Request thành công
  201 Created          — Resource mới được tạo (POST)
  204 No Content       — Thành công, không có body (DELETE)
  206 Partial Content  — Range request (download resume)

3xx — Redirection (chuyển hướng)
  301 Moved Permanently — URL đã đổi vĩnh viễn
  302 Found            — Tạm thời redirect
  304 Not Modified     — Cache vẫn valid (ETag match)
  307 Temporary Redirect — Redirect, giữ nguyên method
  308 Permanent Redirect — Redirect vĩnh viễn, giữ method

4xx — Client Error (lỗi phía client)
  400 Bad Request      — Request syntax sai
  401 Unauthorized     — Chưa authenticate
  403 Forbidden        — Không có quyền
  404 Not Found        — Resource không tồn tại
  405 Method Not Allowed — Method không support
  408 Request Timeout  — Client gửi quá chậm
  413 Payload Too Large — Body quá lớn
  429 Too Many Requests — Rate limited

5xx — Server Error (lỗi phía server)
  500 Internal Server Error — Bug/crash
  502 Bad Gateway      — Upstream server error
  503 Service Unavailable — Server overloaded/maintenance
  504 Gateway Timeout  — Upstream timeout
```

---

## 3. Persistent Connections — Keep-Alive

### Phép so sánh — Giữ điện thoại mở

**HTTP/1.0 (default: close):**
- Mỗi request = 1 cuộc gọi: dial → talk → hang up
- Request tiếp theo = dial AGAIN → talk → hang up AGAIN
- 10 requests = 10 cuộc gọi riêng biệt!

**HTTP/1.1 (default: keep-alive):**
- 1 cuộc gọi: dial → talk → talk → talk... → hang up
- 10 requests = 1 cuộc gọi, nói 10 lần!
- Tiết kiệm thời gian dial (TCP handshake) mỗi lần!

### Cách hoạt động

```
HTTP/1.0 (non-persistent):
  Client          Server
    │─── TCP SYN ──→│
    │←── SYN-ACK ──│     } TCP handshake (1 RTT)
    │─── ACK ──────→│
    │─── GET /a ───→│
    │←── Response ──│     } Request 1
    │─── FIN ──────→│     } Close!
    │                │
    │─── TCP SYN ──→│
    │←── SYN-ACK ──│     } TCP handshake AGAIN! (1 RTT wasted)
    │─── ACK ──────→│
    │─── GET /b ───→│
    │←── Response ──│     } Request 2
    │─── FIN ──────→│     } Close!
    
    Total: 2 handshakes + 2 requests = extra latency!

HTTP/1.1 (persistent — default):
  Client          Server
    │─── TCP SYN ──→│
    │←── SYN-ACK ──│     } TCP handshake (1 RTT) — ONCE!
    │─── ACK ──────→│
    │─── GET /a ───→│
    │←── Response ──│     } Request 1
    │─── GET /b ───→│
    │←── Response ──│     } Request 2 (same connection!)
    │─── GET /c ───→│
    │←── Response ──│     } Request 3 (same connection!)
    │─── FIN ──────→│     } Close (when done)
    
    Total: 1 handshake + 3 requests = much faster!
```

### Connection Header

```http
# HTTP/1.1 — persistent by default
# Explicit (redundant but common):
Connection: keep-alive
Keep-Alive: timeout=60, max=1000

# Close connection after this request:
Connection: close

# HTTP/1.0 — request persistence explicitly:
Connection: keep-alive
```

### Keep-Alive Parameters

```
Keep-Alive: timeout=60, max=1000

timeout=60:  Close idle connection after 60 seconds
max=1000:    Allow up to 1000 requests on this connection
             (then close and client opens new one)

Server-side configuration:
  nginx:
    keepalive_timeout 65;
    keepalive_requests 1000;
    
  Apache:
    KeepAlive On
    KeepAliveTimeout 5
    MaxKeepAliveRequests 100
```

### Lợi ích Persistent Connection

```
Performance gains:
  1. Eliminate TCP handshake: -1 RTT per request (20-100ms saved)
  2. TCP slow start: existing connection already has cwnd warmed up!
  3. TLS session reuse: don't repeat TLS handshake
  4. Fewer sockets: less memory, less TIME_WAIT
  5. Reduce server load: fewer connections to manage

Real-world impact:
  Page with 30 resources:
    Without keep-alive: 30 TCP handshakes × 50ms = 1500ms wasted!
    With keep-alive: 1 handshake × 50ms = 50ms (saved 97%!)
```

---

## 4. HTTP Pipelining — Gửi nhiều request không chờ

### Phép so sánh — Đặt nhiều món cùng lúc

**Không pipeline (sequential):**
- "Cho phở" → đợi phở → "Cho nước" → đợi nước → "Cho kem" → đợi kem
- Tổng: thời gian = nấu phở + nấu nước + nấu kem

**Có pipeline:**
- "Cho phở, nước, và kem luôn!" → đợi tất cả → nhận lần lượt
- Tổng: thời gian ≈ thời gian nấu lâu nhất (overlap cooking!)

### Pipelining hoạt động

```
Without Pipelining (Sequential):
  Client          Server
    │─── GET /a ───→│
    │              │ Processing...
    │←── Response A│
    │─── GET /b ───→│  ← Phải đợi response A xong!
    │              │ Processing...
    │←── Response B│
    │─── GET /c ───→│  ← Phải đợi response B xong!
    │              │ Processing...
    │←── Response C│
    
    Total time: RTT×3 + Processing×3 (sequential)

With Pipelining:
  Client          Server
    │─── GET /a ───→│
    │─── GET /b ───→│  ← Gửi NGAY, không đợi A!
    │─── GET /c ───→│  ← Gửi NGAY, không đợi B!
    │              │ Processing A, B, C...
    │←── Response A│  ← Must respond in ORDER!
    │←── Response B│
    │←── Response C│
    
    Total time: RTT×1 + Processing (overlapped!)
    Tiết kiệm 2 RTT!
```

### Head-of-Line Blocking Problem

```
⚠️ Pipelining có vấn đề nghiêm trọng: HEAD-OF-LINE BLOCKING!

Server PHẢI trả lời theo THỨ TỰ request!

  Client sends: GET /a, GET /b, GET /c
  
  Server:
    - /a takes 5 seconds (big file)
    - /b takes 10ms (small file)
    - /c takes 10ms (small file)
    
  Server MUST respond: A first, then B, then C
    - Response B ready in 10ms BUT MUST WAIT for A (5 seconds)!
    - Client receives: A (5s) → B (5.01s) → C (5.02s)
    
  Without pipeline: A(5s) + B(0.01s) + C(0.01s) = 5.02s
  With pipeline: same! Because B must wait for A!

→ This is why HTTP PIPELINING was NEVER widely adopted!
→ Most browsers DISABLE pipelining!
→ HTTP/2 solves this with MULTIPLEXING (independent streams)
```

### Browsers workaround — Multiple Connections

```
Since pipelining is broken, browsers use PARALLEL CONNECTIONS:

Browser opens 6 TCP connections per hostname (typical limit):
  Connection 1: GET /style.css → Response
  Connection 2: GET /script.js → Response
  Connection 3: GET /image1.png → Response
  Connection 4: GET /image2.png → Response
  Connection 5: GET /image3.png → Response
  Connection 6: GET /font.woff2 → Response

All 6 in parallel! Much faster than sequential.

But:
  - 6 TCP handshakes (latency for each)
  - 6 slow-start periods (suboptimal bandwidth)
  - 6 connections × memory = server resource usage
  - Domain sharding hack: img1.example.com, img2.example.com (more connections)
```

---

## 5. Chunked Transfer Encoding — Gửi khi chưa biết kích thước

### Phép so sánh — Phát sóng trực tiếp vs Pre-recorded

**Content-Length (pre-recorded):**
- Biết trước video dài 2GB → ghi vào header "Content-Length: 2147483648"
- Client biết trước cần download bao nhiêu → show progress bar

**Chunked (live stream):**
- KHÔNG biết trước dài bao nhiêu (đang livestream!)
- Gửi từng đoạn (chunk) → client đọc từng đoạn
- Khi hết → gửi "đoạn cuối" (0-length chunk)

### Khi nào dùng Chunked?

```
1. Dynamic content (chưa biết kích thước):
   - Server render HTML dynamically
   - Database query → stream results
   - Server-Sent Events

2. Large responses (muốn gửi dần):
   - Start sending ASAP (không buffer toàn bộ)
   - Client có thể render partial content
   
3. Keep-alive compatibility:
   - Persistent connection cần biết khi nào response kết thúc
   - Nếu không có Content-Length → dùng Chunked!
   - (Nếu close connection = end of response → chỉ 1 request/connection)
```

### Chunked Format

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Content-Type: text/html

1a\r\n                          ← Chunk size (hex) = 26 bytes
<html><head><title>Hi</title>\r\n   ← Chunk data (26 bytes)
2f\r\n                          ← Chunk size = 47 bytes
</head><body><h1>Hello World!</h1></body></html>\r\n  ← Data
0\r\n                           ← LAST chunk (size 0) = END!
\r\n                            ← End of chunked message
```

```
Chunk format:
  [Chunk Size in Hex]\r\n
  [Chunk Data]\r\n
  [Chunk Size in Hex]\r\n
  [Chunk Data]\r\n
  ...
  0\r\n                    ← Terminating chunk (size = 0)
  [Optional Trailers]\r\n  ← Trailer headers
  \r\n                     ← End
```

### Chunked + Trailers

```http
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Trailer: Content-MD5

... chunks ...
0\r\n
Content-MD5: Q2hlY2tzdW0gYWZ0ZXIgYm9keQ==\r\n
\r\n

Trailers = headers gửi SAU body!
Use case: Checksum, signature (chỉ biết sau khi generate toàn bộ body)
```

### Server-Side Chunked Example (Python)

```python
from flask import Flask, Response, stream_with_context
import time

app = Flask(__name__)

@app.route('/stream')
def stream():
    def generate():
        """Generator produces chunks over time"""
        yield "<html><body>"
        for i in range(10):
            yield f"<p>Item {i}: Loading...</p>\n"
            time.sleep(0.5)  # Simulate processing
        yield "</body></html>"
    
    return Response(
        stream_with_context(generate()),
        content_type='text/html'
    )
    # Flask automatically uses Transfer-Encoding: chunked!
```

---

## 6. HTTP Caching — Giảm tải, tăng tốc

### Phép so sánh — Tủ lạnh vs Đi chợ mỗi ngày

**Không cache** = Đi chợ mỗi bữa:
- Mỗi lần cần rau → đi chợ (request server)
- Tốn thời gian, công sức, tiền xăng

**Có cache** = Tủ lạnh:
- Mua rau 1 lần → bỏ tủ lạnh
- Lần sau cần → lấy từ tủ (no network request!)
- Hết hạn → đi chợ lại (revalidate)
- Kiểm tra: "Rau còn tươi không?" (conditional request)

### Cache-Control Header — Brain của HTTP Caching

```http
# === Response headers (server → client/proxy) ===

# Public cache — CDN/proxy có thể cache:
Cache-Control: public, max-age=3600
# "Cache 1 giờ, ai cũng có thể cache"

# Private cache — chỉ browser cache:
Cache-Control: private, max-age=600
# "Cache 10 phút, chỉ user này, proxy KHÔNG cache!"

# No caching at all:
Cache-Control: no-store
# "ĐỪNG lưu bất cứ đâu!" (sensitive data)

# Must revalidate before use:
Cache-Control: no-cache
# "Có thể cache, nhưng PHẢI kiểm tra với server trước khi dùng"

# Immutable (perfect for hashed filenames):
Cache-Control: public, max-age=31536000, immutable
# "Cache 1 năm, KHÔNG bao giờ thay đổi!" (style.abc123.css)

# stale-while-revalidate (modern):
Cache-Control: max-age=60, stale-while-revalidate=120
# "Fresh 60s, sau đó serve stale TRONG KHI revalidate background"
```

### ETag — "Fingerprint" của resource

```http
# Server response:
HTTP/1.1 200 OK
ETag: "abc123def456"
Content-Type: application/json

{"data": "..."}

# Client subsequent request (conditional):
GET /api/data HTTP/1.1
If-None-Match: "abc123def456"

# Server checks: Is current ETag still "abc123def456"?
# YES → resource unchanged:
HTTP/1.1 304 Not Modified
ETag: "abc123def456"
# (No body! Saves bandwidth!)

# NO → resource changed:
HTTP/1.1 200 OK
ETag: "xyz789"
Content-Type: application/json

{"data": "new data!"}
```

### Last-Modified / If-Modified-Since

```http
# Server response:
HTTP/1.1 200 OK
Last-Modified: Wed, 01 Jul 2026 10:00:00 GMT
Content-Type: text/html

<html>...</html>

# Client conditional request:
GET /page.html HTTP/1.1
If-Modified-Since: Wed, 01 Jul 2026 10:00:00 GMT

# Server: Was it modified after Jul 1?
# NO → 
HTTP/1.1 304 Not Modified
# (Saves bandwidth!)

# YES →
HTTP/1.1 200 OK
Last-Modified: Fri, 03 Jul 2026 15:00:00 GMT

<html>...updated...</html>
```

### Caching Strategy — Real World

```
Static assets (CSS, JS, images):
  Cache-Control: public, max-age=31536000, immutable
  Filename contains hash: style.a1b2c3.css
  → Cache forever! Deploy new version = new filename!

API responses (user data):
  Cache-Control: private, no-cache
  ETag: "user-v5-hash"
  → Always validate, but save bandwidth with 304

HTML pages:
  Cache-Control: public, max-age=0, must-revalidate
  ETag: "page-v3"
  → Always check freshness

Sensitive data (banking):
  Cache-Control: no-store, no-cache, must-revalidate
  Pragma: no-cache  (HTTP/1.0 compat)
  → NEVER cache!
```

---

## 7. Content Negotiation — Giao tiếp đa ngôn ngữ

### Accept Headers

```http
# Client tells server what it can handle:

Accept: text/html, application/json;q=0.9, */*;q=0.1
  "Prefer HTML, accept JSON (90% preference), accept anything else (10%)"

Accept-Language: vi-VN,vi;q=0.9,en-US;q=0.8,en;q=0.7
  "Prefer Vietnamese, then English (US), then any English"

Accept-Encoding: gzip, deflate, br, zstd
  "I support these compression algorithms"

Accept-Charset: utf-8, iso-8859-1;q=0.5
  "Prefer UTF-8, also accept ISO-8859-1"
```

### Content Encoding — Compression

```http
# Client advertises support:
Accept-Encoding: gzip, deflate, br, zstd

# Server responds with compressed content:
HTTP/1.1 200 OK
Content-Encoding: gzip
Content-Type: text/html
Content-Length: 5432  ← Compressed size

[compressed data]

# Compression savings:
Algorithm    Ratio    Speed    Browser Support
gzip         ~70%    Fast     Universal
deflate      ~70%    Fast     Universal
br (Brotli)  ~80%    Medium   Modern browsers
zstd         ~80%    Fast     Chrome 123+, Firefox 126+
```

```bash
# nginx compression:
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_types text/plain text/css application/json application/javascript;
gzip_comp_level 6;

# Brotli (nginx module):
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/json application/javascript;
```

---

## 8. HTTP/1.1 Security và Modern Headers

### Host Header — Bắt buộc trong HTTP/1.1!

```http
# HTTP/1.0: không có Host → server chỉ có thể host 1 website/IP
# HTTP/1.1: Host REQUIRED → multiple websites on same IP (virtual hosting)!

GET / HTTP/1.1
Host: www.example.com    ← Server biết website nào!

# 1 server (1 IP) có thể host hàng nghìn websites:
# - www.site-a.com → serve site A content
# - www.site-b.com → serve site B content
# Both on same IP! (Virtual hosting)
```

### Security Headers

```http
# Prevent clickjacking:
X-Frame-Options: DENY
# Or modern: Content-Security-Policy: frame-ancestors 'none'

# Prevent MIME sniffing:
X-Content-Type-Options: nosniff

# Enable XSS protection (legacy):
X-XSS-Protection: 1; mode=block

# HTTPS enforcement:
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
# "Always use HTTPS for next year, including subdomains"

# Content Security Policy:
Content-Security-Policy: default-src 'self'; script-src 'self' cdn.example.com

# Referrer control:
Referrer-Policy: strict-origin-when-cross-origin

# Permissions Policy (camera, mic, etc.):
Permissions-Policy: camera=(), microphone=(), geolocation=(self)
```

---

## 9. HTTP/1.1 Performance Optimization

### Request/Response Overhead

```
Typical HTTP/1.1 request overhead:
  Request line:     ~50 bytes
  Headers:          200-800 bytes (cookies can be huge!)
  Total overhead:   250-850 bytes PER REQUEST!

For a page with 50 resources:
  50 × 500 bytes headers = 25KB just in headers!
  (HTTP/2 HPACK compression reduces this by ~80%)
```

### Optimization Techniques

```
1. Minimize requests:
   - CSS/JS bundling (webpack)
   - Image sprites
   - Inline critical CSS
   - Data URIs for small images

2. Enable compression:
   - gzip/brotli for text resources
   - Saves 60-80% bandwidth

3. Caching strategy:
   - Long max-age for static assets (with hash filenames)
   - ETag/Last-Modified for dynamic content
   - CDN for global distribution

4. Connection reuse:
   - Keep-Alive enabled (default)
   - Connection pooling on server-side
   - Reasonable timeout (not too short)

5. Domain sharding (HTTP/1.1 specific):
   - Split resources across subdomains
   - More parallel connections (6 per domain × N domains)
   - Trade-off: more DNS lookups + TCP handshakes
   - OBSOLETE with HTTP/2 (multiplexing is better)

6. Reduce header size:
   - Shorter cookie names
   - Remove unnecessary headers
   - Use cookieless domain for static assets
```

### nginx Configuration for HTTP/1.1

```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name example.com;

    # Keep-Alive
    keepalive_timeout 65;
    keepalive_requests 1000;
    
    # Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json 
               application/javascript text/xml;
    
    # Static file caching
    location ~* \.(css|js|png|jpg|gif|ico|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
    
    # HTML caching
    location / {
        add_header Cache-Control "public, max-age=0, must-revalidate";
        try_files $uri $uri/ /index.html;
    }
    
    # Security headers
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Strict-Transport-Security "max-age=31536000" always;
    
    # Proxy to backend with connection pooling
    location /api/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # Enable keep-alive to upstream
    }
}

upstream backend {
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
    keepalive 64;  # Connection pool to backend
}
```

---

## 10. Troubleshooting HTTP/1.1 và Tools

### curl — Swiss Army Knife

```bash
# Basic request:
curl -v https://example.com

# Show response headers:
curl -I https://example.com

# Custom headers:
curl -H "Accept: application/json" -H "Authorization: Bearer token" \
  https://api.example.com/users

# POST with JSON body:
curl -X POST -H "Content-Type: application/json" \
  -d '{"name": "test"}' https://api.example.com/users

# Show timing:
curl -w "\nDNS: %{time_namelookup}s\nConnect: %{time_connect}s\nTLS: %{time_appconnect}s\nTTFB: %{time_starttransfer}s\nTotal: %{time_total}s\n" \
  -o /dev/null -s https://example.com

# Follow redirects:
curl -L https://example.com

# Show connection reuse:
curl -v --keepalive-time 60 https://example.com/a https://example.com/b
```

### Common Issues

```
Issue: "Connection: close" mỗi response
  → Check server config: KeepAlive On
  → Check proxy: proxy_http_version 1.1

Issue: Slow TTFB (Time to First Byte)
  → Server processing slow? (backend)
  → DNS resolution slow? (check DNS)
  → TLS handshake slow? (check cipher suites)

Issue: Large headers (Cookie bloat)
  → Use cookieless domain for static assets
  → Reduce cookie size
  → Consider HTTP/2 (HPACK compression)

Issue: Too many connections
  → Check browser limit (6/domain)
  → Use HTTP/2 (single connection, multiplexed)
  → Combine resources (bundling)
```

### Tổng kết

```
HTTP/1.1 Key Features:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Core Improvements over HTTP/1.0:                       │
│  1. Persistent connections (default keep-alive)         │
│  2. Host header mandatory (virtual hosting)             │
│  3. Chunked transfer encoding (streaming)               │
│  4. Pipelining (theoretical, not practical)             │
│  5. Content negotiation (Accept headers)                │
│  6. Range requests (resume downloads)                   │
│  7. Better caching (Cache-Control, ETag)               │
│                                                         │
│  Limitations (solved by HTTP/2):                        │
│  • Head-of-line blocking (no true multiplexing)        │
│  • Text-based headers (verbose, no compression)        │
│  • No server push                                       │
│  • Domain sharding needed (workaround)                 │
│                                                         │
│  Still Relevant:                                        │
│  • ~20% of web still HTTP/1.1                          │
│  • API backends often HTTP/1.1 (between services)      │
│  • Understanding HTTP/1.1 = understand HTTP/2&3 better │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

*Tài liệu tham khảo:*
- RFC 9110 — HTTP Semantics (replaces RFC 7231)
- RFC 9111 — HTTP Caching (replaces RFC 7234)
- RFC 9112 — HTTP/1.1 (replaces RFC 7230)
- MDN Web Docs — HTTP
- Google Web Fundamentals — HTTP Caching
- nginx HTTP Module Documentation
- Ilya Grigorik — High Performance Browser Networking (hpbn.co)
