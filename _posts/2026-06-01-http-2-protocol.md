---
layout: post
title: "HTTP/2 Protocol - Binary Framing, Multiplexing, HPACK Header Compression và Server Push"
date: 2026-06-01
categories: [networking]
tags: [http2, binary-framing, multiplexing, hpack, server-push, web-performance]
---

## 1. Giới thiệu — Nâng cấp đường truyền

Hãy tưởng tượng **hệ thống bưu chính**:

**HTTP/1.1 (cách cũ):**
- 6 đường ống riêng biệt từ bạn đến bưu điện (6 TCP connections)
- Mỗi ống: gửi thư → đợi trả lời → gửi tiếp (sequential)
- Phong bì viết tay (text-based headers)
- Mỗi lá thư ghi lại ĐẦY ĐỦ thông tin (địa chỉ, họ tên... lặp lại!)

**HTTP/2 (cách mới):**
- **1 đường ống to** duy nhất (1 TCP connection)
- Chia thành nhiều **"luồng" song song** bên trong (multiplexed streams)
- Đóng gói bằng **mã vạch** thay vì viết tay (binary framing)
- Chỉ gửi **khác biệt** so với lần trước (HPACK compression)
- Bưu điện có thể **gửi trước** thứ bạn chưa yêu cầu! (server push)

### HTTP/2 mang lại gì?

| Cải tiến | HTTP/1.1 | HTTP/2 |
|---|---|---|
| Encoding | Text-based | **Binary** |
| Connections | 6 per domain | **1 per domain** |
| Requests | Sequential (per conn) | **Multiplexed (parallel)** |
| Headers | Full repeat mỗi lần | **HPACK compressed (delta)** |
| Server initiative | Không | **Server Push** |
| Priority | Không | **Stream Priority** |
| Flow control | TCP-level only | **Per-stream + connection** |

### Real-world Performance

```
Page load metrics (Google research):
  HTTP/1.1 → HTTP/2:
  - Page load time: 15-50% improvement
  - Time to first paint: 20-40% improvement
  - Header overhead: 80% reduction
  - Connections per page: 6+ → 1
```

---

## 2. Binary Framing Layer — Nền tảng của HTTP/2

### Phép so sánh — Ngôn ngữ mới

**HTTP/1.1 (text)** = Viết thư bằng tiếng Việt:
```
GET /index.html HTTP/1.1\r\n
Host: www.example.com\r\n
Accept: text/html\r\n
\r\n
```
- Dễ đọc bằng mắt thường
- Parser phải scan từng ký tự, tìm `\r\n`
- Không rõ ràng khi nào kết thúc header vs body

**HTTP/2 (binary)** = Đóng gói theo mã vạch:
```
[Length: 38][Type: HEADERS][Flags: END_HEADERS]
[Stream ID: 1]
[Compressed header block]
```
- Máy đọc cực nhanh (biết ngay size, type, boundaries)
- Không ambiguity — mỗi frame có length rõ ràng
- Nhưng KHÔNG đọc được bằng mắt thường (cần tools)

### Frame Structure

```
HTTP/2 Frame Format (9-byte header + payload):

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                   Length (24 bits)                                 │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│   Type (8)  │  Flags (8)  │R│       Stream Identifier (31)       │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                   Frame Payload (0 to 2^24-1 bytes)               │
└─────────────────────────────────────────────────────────────────────┘

Length:     Payload size (NOT including 9-byte header)
            Max: 16,384 (default) to 16,777,215 (max setting)
Type:       Frame type (DATA, HEADERS, PRIORITY, etc.)
Flags:      Type-specific flags
R:          Reserved bit (0)
Stream ID:  0 = connection-level, 1+ = specific stream
```

### Frame Types

| Type | Code | Mô tả |
|---|---|---|
| **DATA** | 0x00 | Request/Response body |
| **HEADERS** | 0x01 | HTTP headers (compressed) |
| **PRIORITY** | 0x02 | Stream priority (deprecated in RFC 9113) |
| **RST_STREAM** | 0x03 | Cancel a stream |
| **SETTINGS** | 0x04 | Connection configuration |
| **PUSH_PROMISE** | 0x05 | Server Push announcement |
| **PING** | 0x06 | Connectivity check + RTT measurement |
| **GOAWAY** | 0x07 | Graceful connection shutdown |
| **WINDOW_UPDATE** | 0x08 | Flow control update |
| **CONTINUATION** | 0x09 | Header block continuation |

### Connection Preface

```
HTTP/2 connection starts with a "magic" string:

Client sends (24 bytes):
  PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n

Followed by SETTINGS frame:
  [Length=0][Type=SETTINGS][Flags=0][StreamID=0]

Server responds with:
  SETTINGS frame (its own settings)
  SETTINGS ACK (acknowledging client settings)

Then: ready for request/response!
```

---

## 3. Multiplexing — Song song thực sự

### Phép so sánh — 1 đường cao tốc, nhiều làn

**HTTP/1.1** = 6 con đường nhỏ riêng biệt:
- Mỗi đường 1 xe đi/1 lúc
- Xe to (slow request) chặn cả đường

**HTTP/2** = 1 đường cao tốc siêu rộng:
- Chia thành nhiều "làn" (streams)
- Mỗi làn chạy độc lập
- Xe to chạy làn riêng, không chặn xe nhỏ
- TẤT CẢ trong 1 connection duy nhất!

### Streams, Messages, Frames

```
Connection (1 TCP connection):
├── Stream 1 (Request 1: GET /index.html)
│   ├── Frame: HEADERS (request headers)
│   └── Frame: DATA (response body, part 1)
│   └── Frame: DATA (response body, part 2)
│
├── Stream 3 (Request 2: GET /style.css)
│   ├── Frame: HEADERS (request headers)
│   └── Frame: DATA (response body)
│
├── Stream 5 (Request 3: GET /app.js)
│   ├── Frame: HEADERS (request headers)
│   └── Frame: DATA (response body, part 1)
│   └── Frame: DATA (response body, part 2)
│   └── Frame: DATA (response body, part 3)
│
└── Stream 0 (Connection-level control)
    ├── SETTINGS
    ├── WINDOW_UPDATE
    └── PING
```

### Interleaving on the Wire

```
Actual byte sequence on TCP connection (interleaved!):

[HEADERS Stream 1] [HEADERS Stream 3] [HEADERS Stream 5]
[DATA Stream 1, chunk 1] [DATA Stream 3, chunk 1] 
[DATA Stream 5, chunk 1] [DATA Stream 1, chunk 2]
[DATA Stream 5, chunk 2] [DATA Stream 3, chunk 2]
[DATA Stream 5, chunk 3]

Streams 1, 3, 5 all progress SIMULTANEOUSLY!
No waiting! No head-of-line blocking (at HTTP level)!

(TCP-level HoL blocking still exists → solved by HTTP/3/QUIC)
```

### Stream Rules

```
- Stream IDs are odd (client-initiated) or even (server-initiated)
  Client: 1, 3, 5, 7, 9...
  Server: 2, 4, 6, 8, 10... (server push)
  
- Stream 0: Connection control (SETTINGS, PING, GOAWAY)
- Streams are NOT reused — once closed, ID is "spent"
- Concurrency limited by SETTINGS_MAX_CONCURRENT_STREAMS
  Default: implementation-specific (often 100-256)
  
- Stream states:
  idle → open → half-closed (local/remote) → closed
```

---

## 4. HPACK — Header Compression

### Phép so sánh — Tốc ký vs Viết đầy đủ

**HTTP/1.1** — viết đầy đủ mỗi lần:
```
Request 1: "Nguyễn Văn An, 123 Trần Hưng Đạo, Quận 1, TP.HCM, Việt Nam"
Request 2: "Nguyễn Văn An, 123 Trần Hưng Đạo, Quận 1, TP.HCM, Việt Nam" (LẶP!)
Request 3: "Nguyễn Văn An, 123 Trần Hưng Đạo, Quận 1, TP.HCM, Việt Nam" (LẶP!)
```

**HPACK** — dùng tốc ký + dynamic table:
```
Request 1: "Nguyễn Văn An, 123 Trần Hưng Đạo, Q1, HCM, VN" (full, save as #62)
Request 2: "#62" (chỉ 3 bytes thay vì 60 bytes!)
Request 3: "#62" (chỉ 3 bytes!)
```

### HPACK hoạt động

```
HPACK sử dụng 2 tables:

1. Static Table (61 entries, pre-defined):
   Index 1: :authority = ""
   Index 2: :method = GET
   Index 3: :method = POST
   Index 4: :path = /
   Index 5: :path = /index.html
   ...
   Index 61: www-authenticate = ""

2. Dynamic Table (built during connection):
   Index 62: :authority = www.example.com (added from first request)
   Index 63: accept-encoding = gzip, deflate, br
   Index 64: cookie = session=abc123
   ...

Encoding:
  - Indexed: Just send table index (1 byte!)
  - Literal with indexing: Send value + add to dynamic table
  - Literal without indexing: Send value, don't store
  - Literal never indexed: Send value, NEVER store (sensitive!)
```

### HPACK Compression Example

```
First request:
  :method: GET           → Index 2 (static table) → 1 byte
  :path: /api/users      → Literal + index → ~12 bytes (add to DT as #62)
  :authority: api.ex.com → Literal + index → ~14 bytes (add to DT as #63)
  accept: application/json → Literal + index → ~20 bytes (add to DT as #64)
  cookie: token=abc...   → Literal never indexed → full size
  Total: ~50 bytes (vs ~200 bytes uncompressed)

Second request (same server):
  :method: GET           → Index 2 → 1 byte
  :path: /api/users/1    → Literal + index → ~15 bytes (different path)
  :authority: api.ex.com → Index 63 (dynamic!) → 1 byte!
  accept: application/json → Index 64 → 1 byte!
  cookie: token=abc...   → Literal → full size
  Total: ~25 bytes (vs ~200 bytes = 87% compression!)

Headers that DON'T change between requests → nearly FREE!
```

### HPACK vs HPACK Bomb (CRIME attack mitigation)

```
CRIME attack (2012): Exploited gzip compression on headers
  - Attacker injects data into request
  - Observes compressed size
  - Smaller = data matches secret (cookie/token)
  - Gradually leaks secrets!

HPACK mitigation:
  - NO general-purpose compression (no gzip on headers)
  - Huffman coding (fixed, no adaptive dictionary)
  - Static + Dynamic table approach (not compressing across origins)
  - "Never indexed" flag for sensitive headers (cookies, auth)
```

---

## 5. Server Push — Gửi trước khi được hỏi

### Phép so sánh — Phục vụ proactive

Bạn vào nhà hàng gọi "phở":
- **Không push:** Mang phở → "Cho nước mắm" → mang nước mắm → "Cho tương ớt" → mang
- **Server push:** Mang phở + nước mắm + tương ớt **CÙNG LÚC** (server biết bạn cần!)

### Server Push hoạt động

```
Client requests: GET /index.html

Server knows: index.html needs style.css and app.js
Server sends:
  1. PUSH_PROMISE (stream 2): "I'll push /style.css"
  2. PUSH_PROMISE (stream 4): "I'll push /app.js"
  3. HEADERS + DATA (stream 1): /index.html response
  4. HEADERS + DATA (stream 2): /style.css (pushed!)
  5. HEADERS + DATA (stream 4): /app.js (pushed!)

Client receives style.css and app.js BEFORE parsing HTML!
→ No extra round-trip for these resources!
```

```
Timeline comparison:

Without Server Push:
  Client: GET /index.html ────→ Server
  Server: ←──── index.html response
  Client parses HTML, discovers needs CSS/JS
  Client: GET /style.css ────→ Server    } Extra RTT!
  Client: GET /app.js ──────→ Server    }
  Server: ←──── style.css
  Server: ←──── app.js
  Total: 2 RTTs

With Server Push:
  Client: GET /index.html ────→ Server
  Server: ←──── index.html + PUSH(style.css) + PUSH(app.js)
  Total: 1 RTT! (saved 1 RTT)
```

### Server Push Configuration — nginx

```nginx
# nginx with HTTP/2 push:
server {
    listen 443 ssl http2;
    
    location = /index.html {
        http2_push /css/style.css;
        http2_push /js/app.js;
        http2_push /img/logo.png;
    }
    
    # Or auto-detect from Link header:
    location / {
        http2_push_preload on;
        # Backend sends: Link: </style.css>; rel=preload; as=style
        # nginx converts to HTTP/2 push
    }
}
```

### Server Push Status (2026)

```
⚠️ Server Push is being DEPRECATED!

Why push failed in practice:
1. Cache awareness: Server doesn't know if client already has file cached!
   → Pushes file client already has → waste bandwidth!
   
2. Priority inversion: Pushed resources compete with critical content
   → Can delay the main HTML response!

3. 103 Early Hints replaces push:
   Server sends: HTTP/1.1 103 Early Hints
                 Link: </style.css>; rel=preload; as=style
   → Client starts fetching preloads WHILE server prepares response
   → Better than push (respects cache, no priority issues)

4. Chrome removed push support (Chrome 106, 2022)
5. RFC 9113 deprecates push in HTTP/2 specification

Alternative: Use 103 Early Hints + preload hints!
```

---

## 6. Flow Control — Kiểm soát luồng per-stream

### HTTP/2 Flow Control

```
HTTP/2 has DUAL flow control:
  1. Connection-level: total bytes across all streams
  2. Per-stream: bytes for individual stream

WINDOW_UPDATE frame:
  [Length: 4][Type: WINDOW_UPDATE][Flags: 0][Stream: X]
  [Window Size Increment (31 bits)]

Initial window: 65,535 bytes (per stream and connection)
Can be increased via SETTINGS_INITIAL_WINDOW_SIZE

Rules:
  - Receiver controls (similar to TCP rwnd)
  - Sender cannot send more than window allows
  - WINDOW_UPDATE increases window (as receiver consumes data)
  - Window can reach 0 (sender must pause)
  - Only DATA frames are flow-controlled (not HEADERS, etc.)
```

### Stream Priority (Original — Deprecated)

```
Original HTTP/2 (RFC 7540):
  - Tree-based dependency model
  - Streams depend on parent streams
  - Weight: 1-256 (relative priority within siblings)
  
  Stream 1 (HTML) → weight 256 (highest)
    ├── Stream 3 (CSS) → weight 200
    ├── Stream 5 (JS) → weight 100
    └── Stream 7 (Image) → weight 50

RFC 9218 — Extensible Prioritization (replaces):
  Priority: u=0, i     (urgency=0 highest, incremental)
  Priority: u=3        (urgency=3 normal, non-incremental)
  Priority: u=6, i     (urgency=6 low, incremental — images)
  
  Urgency: 0 (highest) to 7 (lowest)
  Incremental: i (can render partially, like images)
```

---

## 7. HTTP/2 Connection Lifecycle

### ALPN Negotiation (Application-Layer Protocol Negotiation)

```
HTTP/2 is negotiated during TLS handshake:

TLS ClientHello:
  Extension: ALPN
    Protocol: h2        ← HTTP/2 over TLS
    Protocol: http/1.1  ← Fallback

TLS ServerHello:
  Extension: ALPN
    Protocol: h2        ← Server accepts HTTP/2!

After TLS: Both sides know to speak HTTP/2.

For cleartext HTTP/2 (h2c — rarely used):
  Client sends: Upgrade: h2c
  Server: HTTP/1.1 101 Switching Protocols
  → Switch to HTTP/2
  (Not supported by browsers — they require TLS for h2)
```

### SETTINGS Frame

```
Client and Server exchange SETTINGS at connection start:

Key settings:
  SETTINGS_HEADER_TABLE_SIZE (0x1)     = 4096 (HPACK table size)
  SETTINGS_ENABLE_PUSH (0x2)           = 1 (server push allowed)
  SETTINGS_MAX_CONCURRENT_STREAMS (0x3) = 100 (parallel streams)
  SETTINGS_INITIAL_WINDOW_SIZE (0x4)    = 65535 (flow control)
  SETTINGS_MAX_FRAME_SIZE (0x5)        = 16384 (max frame payload)
  SETTINGS_MAX_HEADER_LIST_SIZE (0x6)  = 8192 (max header size)
```

### Graceful Shutdown — GOAWAY

```
Server wants to shut down (maintenance, deploy):

1. Server sends GOAWAY:
   [Last-Stream-ID: 7]
   "I will not process any stream > 7"
   
2. Existing streams (≤7): continue to completion
3. New streams (>7): client must open new connection
4. Client: finishes current requests, opens new connection for next

→ Zero-downtime deploy!
→ No interrupted requests!
```

---

## 8. HTTP/2 Deployment — nginx, Cloud, CDN

### nginx HTTP/2 Configuration

```nginx
# /etc/nginx/conf.d/http2.conf

server {
    listen 443 ssl http2;       # Enable HTTP/2 (requires TLS)
    server_name example.com;
    
    ssl_certificate /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    
    # HTTP/2 specific settings
    http2_max_concurrent_streams 128;
    http2_max_field_size 8k;
    http2_max_header_size 32k;
    
    # Keep-alive to upstream (HTTP/1.1)
    # Many backends still speak HTTP/1.1!
    location /api/ {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}

# Redirect HTTP to HTTPS (HTTP/2 requires TLS in browsers)
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

### gRPC over HTTP/2

```
gRPC uses HTTP/2 as transport:
  - Binary Protocol Buffers (efficient serialization)
  - HTTP/2 streams for request/response
  - HTTP/2 multiplexing for parallel RPCs
  - HTTP/2 flow control
  - Bidirectional streaming (client + server streams)

gRPC Frame:
  [Compressed Flag (1 byte)] [Message Length (4 bytes)] [Message]
  
  Carried as DATA frames in HTTP/2 stream
```

---

## 9. HTTP/2 vs HTTP/1.1 — Performance Analysis

### When HTTP/2 Helps Most

```
✅ HTTP/2 excels when:
  - Many small resources (modern web pages: 50-100 resources)
  - High latency connections (mobile, satellite)
  - TLS required (amortize handshake over 1 connection)
  - Header-heavy requests (large cookies, auth tokens)
  - Mixed-priority resources (critical CSS vs lazy images)

⚠️ HTTP/2 might NOT help when:
  - Single large download (already 1 stream)
  - Very low latency LAN (handshake overhead minimal)
  - High packet loss (TCP HoL blocking worse with 1 connection)
  - Small number of resources (minimal multiplexing benefit)
```

### HTTP/2 Rapid Reset Attack (CVE-2023-44487)

```
Discovery: October 2023 (Cloudflare, Google, AWS)

Attack:
  1. Client opens stream → RST_STREAM immediately (cancel)
  2. Repeat millions of times → server allocates/deallocates per stream
  3. Server CPU exhausted processing rapid stream open/close!
  
  Unlike traditional DDoS (bandwidth), this attacks PROCESSING!
  
Mitigation:
  - Rate limit stream creation per connection
  - Count RST_STREAM frames → disconnect abusive clients
  - nginx: http2_max_concurrent_streams + rate limits
  - Cloud: WAF rules detecting rapid reset patterns
```

---

## 10. Tổng kết và Migration Guide

### Migration Checklist: HTTP/1.1 → HTTP/2

```
1. Enable TLS (required for browsers):
   □ Valid SSL certificate (Let's Encrypt free)
   □ TLS 1.2+ (TLS 1.3 recommended)
   □ Strong cipher suites

2. Server configuration:
   □ Enable HTTP/2 (nginx: listen 443 ssl http2)
   □ Set appropriate concurrent stream limit
   □ Configure HPACK table size

3. REMOVE HTTP/1.1 optimizations (now anti-patterns!):
   □ Remove domain sharding (1 connection better!)
   □ Remove image sprites (multiplexing handles it)
   □ Remove CSS/JS concatenation (individual files = better caching)
   □ Remove inline resources (can be pushed/preloaded)

4. Keep these optimizations:
   □ Compression (gzip/brotli)
   □ Caching headers (Cache-Control, ETag)
   □ Image optimization
   □ Critical CSS (still useful for first paint)

5. Testing:
   □ Chrome DevTools → Protocol column shows "h2"
   □ curl --http2 -v
   □ Lighthouse audit
```

### Summary

```
HTTP/2 Key Innovations:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Binary Framing:                                        │
│  • 9-byte frame header (Length + Type + Flags + Stream) │
│  • Efficient parsing, no ambiguity                      │
│  • Foundation for multiplexing                          │
│                                                         │
│  Multiplexing:                                          │
│  • Multiple streams on 1 TCP connection                 │
│  • No head-of-line blocking (at HTTP level)            │
│  • Eliminates need for domain sharding                  │
│                                                         │
│  HPACK:                                                 │
│  • Static table (61 common headers)                    │
│  • Dynamic table (connection-specific)                  │
│  • Huffman coding for values                           │
│  • 80%+ header compression                             │
│                                                         │
│  Server Push (deprecated):                              │
│  • Replaced by 103 Early Hints                         │
│                                                         │
│  Flow Control:                                          │
│  • Per-stream + connection level                       │
│  • WINDOW_UPDATE mechanism                              │
│                                                         │
│  Remaining Issue:                                        │
│  • TCP head-of-line blocking → solved by HTTP/3 (QUIC) │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

*Tài liệu tham khảo:*
- RFC 9113 — HTTP/2 (replaces RFC 7540)
- RFC 7541 — HPACK: Header Compression for HTTP/2
- RFC 9218 — Extensible Prioritization Scheme for HTTP
- RFC 8297 — An HTTP Status Code for Indicating Hints (103)
- Google — HTTP/2 Performance Analysis
- nginx HTTP/2 Module Documentation
- Ilya Grigorik — High Performance Browser Networking, HTTP/2 Chapter
- CVE-2023-44487 — HTTP/2 Rapid Reset Attack
