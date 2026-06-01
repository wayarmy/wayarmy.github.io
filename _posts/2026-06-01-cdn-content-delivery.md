---
layout: post
title: "CDN & Content Delivery Deep Dive - Mạng Phân Phối Nội Dung"
date: 2026-06-01
categories: [networking]
tags: [cdn, cloudfront, caching, performance]
---

# CDN & Content Delivery Deep Dive - Mạng Phân Phối Nội Dung

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Hãy tưởng tượng bạn đang sống ở Hà Nội và muốn mua iPhone mới. Bạn có 2 lựa chọn:

**Không có CDN:** Đặt hàng trực tiếp từ nhà máy Apple ở California, Mỹ. Gói hàng phải bay hơn 13,000 km → mất 2-3 tuần.

**Có CDN:** Mua tại cửa hàng Apple Store ở Hà Nội. Hàng đã được vận chuyển sẵn từ Mỹ và trữ ở kho gần bạn → mua ngay, nhận ngay.

**CDN (Content Delivery Network — Mạng Phân Phối Nội Dung)** hoạt động y hệt nguyên lý "kho hàng gần nhà":
- Nội dung website (hình ảnh, video, CSS, JavaScript) được copy và lưu trữ sẵn tại nhiều "kho" (edge locations) trên khắp thế giới
- Khi bạn truy cập, bạn được phục vụ từ kho GẦN NHẤT thay vì phải "bay" đến server gốc ở xa

**Ví dụ thực tế khác:**

| Không CDN | Có CDN |
|-----------|--------|
| Chỉ có 1 thư viện trung tâm thành phố | Mỗi quận có thư viện chi nhánh |
| ATM chỉ ở trụ sở chính ngân hàng | ATM ở mọi góc phố |
| Netflix chỉ có 1 server ở Mỹ | Netflix đặt server cache ở mỗi ISP lớn |

**Kết quả khi dùng CDN:**
- Website load nhanh hơn 2-10x (giảm latency)
- Server gốc (origin) nhẹ gánh (giảm 60-90% traffic)
- Chống được DDoS tốt hơn (traffic phân tán ra nhiều nơi)
- Người dùng ở xa vẫn trải nghiệm nhanh

---

## 2. Kiến Thức Nền Tảng

### 2.1 Tại Sao Khoảng Cách Ảnh Hưởng Tốc Độ?

Ánh sáng truyền qua cáp quang với tốc độ ~200,000 km/s (2/3 tốc độ ánh sáng trong chân không). Nghe nhanh, nhưng:

```
Khoảng cách Hà Nội → California: ~13,000 km
Thời gian 1 chiều: 13,000 / 200,000 = 65 milliseconds
Round trip (đi + về): 130 milliseconds

Mỗi HTTP request cần:
- DNS lookup: 1 round trip
- TCP handshake: 1 round trip  
- TLS handshake: 2 round trips
- HTTP request/response: 1 round trip
Tổng minimum: 5 × 130ms = 650ms CHỈ cho 1 file!

Một trang web trung bình cần 50-100 files (images, CSS, JS)
→ Thời gian tải: NHIỀU GIÂY chỉ vì khoảng cách
```

**Với CDN (edge ở Hà Nội):**
```
Khoảng cách Hà Nội → Edge server Hà Nội: ~5 km
Round trip: < 5ms
5 round trips × 5ms = 25ms cho 1 file!
→ Nhanh hơn 26 lần so với không CDN!
```

### 2.2 Kiến Trúc CDN 3 Tầng

```
┌─────────────────────────────────────────────────┐
│                    INTERNET                       │
└──────────┬──────────────────────────┬───────────┘
           ↓                          ↓
   ┌──────────────┐          ┌──────────────┐
   │  Edge PoP 1  │          │  Edge PoP 2  │    ← Tầng 1: Edge
   │  (Hà Nội)   │          │  (TP.HCM)    │       (gần user nhất)
   └──────┬───────┘          └──────┬───────┘
          ↓                          ↓
   ┌─────────────────────────────────────────┐
   │         Origin Shield / Mid-Tier         │    ← Tầng 2: Shield
   │         (Singapore hoặc Tokyo)           │       (tầng trung gian)
   └──────────────────┬──────────────────────┘
                      ↓
   ┌─────────────────────────────────────────┐
   │              Origin Server               │    ← Tầng 3: Origin
   │           (Server gốc - US)              │       (nguồn dữ liệu)
   └─────────────────────────────────────────┘
```

- **Edge PoP (Point of Presence):** Điểm hiện diện gần user nhất, cache nội dung phổ biến
- **Origin Shield:** Tầng cache trung gian, bảo vệ origin khỏi bị request trực tiếp quá nhiều
- **Origin Server:** Server gốc chứa dữ liệu master

### 2.3 Thuật Ngữ Quan Trọng

| Thuật ngữ | Tiếng Việt | Giải thích |
|-----------|-----------|------------|
| Edge Location | Điểm biên | Server cache gần user nhất |
| Origin | Nguồn gốc | Server chính chứa dữ liệu gốc |
| Cache Hit | Trúng cache | Tìm thấy nội dung tại edge → không cần hỏi origin |
| Cache Miss | Trượt cache | Không có tại edge → phải lấy từ origin |
| TTL | Thời gian sống | Bao lâu content được giữ trong cache trước khi hết hạn |
| Invalidation | Xóa cache | Buộc edge xóa content cũ, lấy bản mới |
| Purge | Thanh lọc | Xóa nội dung cache ngay lập tức |
| PoP | Điểm hiện diện | Cụm server tại một vị trí địa lý |

---

## 3. Cache Key — Chìa Khóa Xác Định Nội Dung Cache

### 3.1 Cache Key Là Gì?

**Ví dụ đời thường:** Trong thư viện, mỗi sách có 1 mã số (ISBN). Khi bạn yêu cầu sách, thủ thư tra mã số để tìm sách trên kệ. **Cache key = mã số sách** — xác định duy nhất một nội dung trong cache.

**Cache Key mặc định của CDN:**
```
Cache Key = URL scheme + Host + Path + Query String

Ví dụ:
https://example.com/images/logo.png?v=2
→ Key: "https://example.com/images/logo.png?v=2"
```

### 3.2 Tùy Chỉnh Cache Key

**Vấn đề 1: Query String Parameters**
```
https://example.com/page?utm_source=facebook&utm_campaign=summer&id=123
https://example.com/page?utm_source=google&utm_campaign=winter&id=123

Cùng nội dung (id=123) nhưng khác query string
→ CDN cache 2 bản riêng biệt → lãng phí!

Giải pháp: Chỉ include "id" vào cache key, ignore utm_*
→ Cache key: "https://example.com/page?id=123"
→ Chỉ cache 1 bản, phục vụ cho cả 2 URL
```

**Vấn đề 2: Headers trong Cache Key**
```
Cùng URL nhưng khác Accept-Language:
GET /page HTTP/1.1
Accept-Language: vi → Trang tiếng Việt
Accept-Language: en → Trang tiếng Anh

Giải pháp: Include Accept-Language vào cache key
→ Cache 2 bản riêng cho 2 ngôn ngữ
```

**Vấn đề 3: Cookie trong Cache Key**
```
Cùng URL nhưng khác cookie "user_type":
Cookie: user_type=premium → Trang premium (có thêm features)
Cookie: user_type=free → Trang miễn phí

Giải pháp: Include cookie "user_type" vào cache key
⚠️ CẢNH BÁO: Đừng include session cookie → mỗi user = 1 cache entry → vô nghĩa
```

### 3.3 Cache Key Best Practices

```
✅ Include:
  - URL path (luôn luôn)
  - Query params ảnh hưởng nội dung (id, page, lang)
  - Accept-Encoding (gzip vs br vs none)
  - Device type header (mobile vs desktop, nếu serve khác)

❌ Exclude:
  - Tracking params (utm_source, utm_medium, fbclid, gclid)
  - Session cookies (JSESSIONID, PHPSESSID)
  - Random/unique params (timestamp, nonce)
  - Authorization header (nếu content public)

Nguyên tắc: Cache key càng ít thành phần → cache hit rate càng cao
             Nhưng phải đủ để phân biệt nội dung khác nhau
```

---

## 4. TTL (Time To Live) — Thời Gian Sống Của Cache

### 4.1 TTL Là Gì?

**Ví dụ đời thường:** Sữa tươi có hạn sử dụng 7 ngày. Sau 7 ngày, dù sữa trông vẫn bình thường, bạn nên vứt đi và mua hộp mới. **TTL = hạn sử dụng** của nội dung trong cache.

```
TTL = 3600 giây (1 giờ)
→ Sau 1 giờ, edge xem content là "stale" (cũ)
→ Request tiếp theo sẽ hỏi lại origin: "Content có mới hơn không?"
```

### 4.2 Cách Đặt TTL Qua HTTP Headers

**Cache-Control Header (Modern — RFC 7234):**
```http
Cache-Control: public, max-age=86400, s-maxage=3600

- public: CDN được phép cache (ngược lại: private = chỉ browser cache)
- max-age=86400: Browser cache 24 giờ
- s-maxage=3600: CDN (shared cache) cache 1 giờ
- s-maxage override max-age cho CDN
```

**Các directive phổ biến:**
```http
Cache-Control: no-store          → KHÔNG cache ở đâu cả (login pages)
Cache-Control: no-cache          → Cache nhưng PHẢI validate trước khi dùng
Cache-Control: public, max-age=31536000, immutable  → Cache 1 năm, không validate
Cache-Control: stale-while-revalidate=60  → Serve stale 60s trong khi fetch mới
Cache-Control: stale-if-error=300  → Nếu origin lỗi, serve stale tối đa 5 phút
```

### 4.3 Chiến Lược TTL Theo Loại Nội Dung

| Loại content | TTL khuyến nghị | Lý do |
|-------------|----------------|-------|
| Static assets (JS, CSS, images) | 1 năm (31536000s) | File tĩnh + versioned URL |
| HTML pages | 5 phút - 1 giờ | Có thể thay đổi thường xuyên |
| API responses | 0 - 60 giây | Data realtime |
| Video/Audio | 1 ngày - 1 tuần | Ít thay đổi sau publish |
| User-specific content | 0 (no-cache) | Khác nhau mỗi user |
| News articles | 5-15 phút | Cập nhật thường xuyên |
| Product images | 1 tuần | Hiếm khi thay đổi |

### 4.4 Cache Revalidation — Kiểm Tra Lại

Khi TTL hết hạn, edge không nhất thiết phải download lại toàn bộ content:

```
Lần đầu (Cold):
Edge → Origin: GET /image.jpg
Origin → Edge: 200 OK (5MB image)
  ETag: "abc123"
  Last-Modified: Mon, 01 Jun 2026 10:00:00 GMT

Sau TTL hết hạn (Revalidation):
Edge → Origin: GET /image.jpg
  If-None-Match: "abc123"
  If-Modified-Since: Mon, 01 Jun 2026 10:00:00 GMT

Nếu content KHÔNG thay đổi:
Origin → Edge: 304 Not Modified (0 bytes body!)
→ Edge tiếp tục dùng bản cache cũ → Tiết kiệm bandwidth!

Nếu content ĐÃ thay đổi:
Origin → Edge: 200 OK (5MB image mới)
  ETag: "def456"
→ Edge update cache
```

---

## 5. Cache Invalidation — Xóa Cache (Bài Toán Khó Nhất)

### 5.1 Tại Sao Invalidation Khó?

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

**Ví dụ đời thường:** Bạn in 10,000 brochure quảng cáo và phát đi khắp thành phố. Hôm sau phát hiện số điện thoại in sai. Bạn phải:
1. Tìm lại tất cả brochure đã phát (ở đâu? ai cầm?)
2. Thu hồi hoặc thông báo sai
3. In lại brochure mới và phát lại

CDN invalidation cũng vậy — content đã được copy đến 200+ edge locations trên thế giới!

### 5.2 Các Phương Pháp Invalidation

**Phương pháp 1: Path-based Invalidation (Xóa theo đường dẫn)**
```
Invalidate: /images/logo.png → Xóa 1 file
Invalidate: /images/* → Xóa tất cả trong /images/
Invalidate: /* → Xóa TOÀN BỘ cache (nuclear option!)

AWS CloudFront: Mỗi tháng 1,000 invalidation paths miễn phí
Sau đó: $0.005/path
```

**Phương pháp 2: Cache Tag / Surrogate Key**
```
Response từ origin kèm tag:
Surrogate-Key: product-123 homepage electronics

Khi product-123 thay đổi:
PURGE all objects tagged "product-123"
→ Xóa tất cả pages/images liên quan đến product này
→ Chính xác hơn wildcard
```

**Phương pháp 3: Versioned URLs (Tốt nhất cho static assets)**
```
Thay vì: /css/style.css (TTL 1 năm, invalidate khi thay đổi)
Dùng:   /css/style.v2.css hoặc /css/style.css?v=abc123

Khi code thay đổi:
HTML reference: /css/style.v3.css
→ URL mới = cache miss = CDN tự lấy bản mới
→ Không cần invalidation! URL cũ expire tự nhiên

Content-addressed: /css/style.a1b2c3d4.css (hash of content)
→ Content thay đổi → hash thay đổi → URL mới → auto cache bust
```

### 5.3 So Sánh Các Phương Pháp

| Phương pháp | Tốc độ | Chi phí | Chính xác | Use case |
|-------------|--------|---------|-----------|----------|
| Path invalidation | 1-15 phút | Có phí sau limit | Trung bình | Emergency fix |
| Cache tags | Instant | Provider-specific | Cao | Dynamic content |
| Versioned URLs | Instant (cache miss) | Miễn phí | 100% | Static assets |
| Short TTL | Tự động theo TTL | Miễn phí | N/A | Frequently changing |

### 5.4 Invalidation Best Practices

```
✅ Static assets: Versioned URLs (content hash) + TTL 1 năm
✅ HTML pages: Short TTL (5-15 phút) + stale-while-revalidate
✅ Emergency: Path invalidation (dùng khi phải fix ngay)
✅ Dynamic content: Cache tags + purge API

❌ Đừng invalidate /* thường xuyên (cache hit rate giảm xuống 0)
❌ Đừng dùng TTL=0 cho tất cả (vô nghĩa khi dùng CDN)
❌ Đừng rely vào invalidation cho workflow thường ngày (tốn tiền + chậm)
```

---

## 6. Origin Shield — Tầng Bảo Vệ Server Gốc

### 6.1 Vấn Đề Không Có Origin Shield

**Ví dụ đời thường:** Một ngôi sao nổi tiếng không có quản lý. 50 phóng viên từ 50 tòa soạn gọi trực tiếp cho ngôi sao → ngôi sao bị quấy rầy liên tục, không làm gì được.

```
Không có Origin Shield:
Edge Hà Nội   → Cache miss → Hỏi Origin
Edge TP.HCM   → Cache miss → Hỏi Origin
Edge Singapore → Cache miss → Hỏi Origin
Edge Tokyo     → Cache miss → Hỏi Origin
Edge Sydney    → Cache miss → Hỏi Origin
...50 edge locations cùng hỏi origin cùng lúc!
→ Origin nhận 50 requests cho CÙNG 1 content
→ Origin quá tải (thundering herd)
```

### 6.2 Origin Shield Giải Quyết

```
Có Origin Shield (đặt tại Singapore):
Edge Hà Nội   → Cache miss → Hỏi Shield Singapore
Edge TP.HCM   → Cache miss → Hỏi Shield Singapore
Edge Tokyo     → Cache miss → Hỏi Shield Singapore

Shield Singapore:
- Nhận request đầu tiên → hỏi Origin → cache kết quả
- Requests 2-50 → trả từ Shield cache
→ Origin chỉ nhận 1 request thay vì 50!
→ Origin nhẹ nhàng, bandwidth giảm 50x
```

### 6.3 Lợi Ích Origin Shield

| Metric | Không Shield | Có Shield | Cải thiện |
|--------|-------------|-----------|-----------|
| Requests đến Origin | 100% cache miss | ~5-10% | 90-95% giảm |
| Origin bandwidth | Cao | Rất thấp | Giảm chi phí egress |
| Cache hit rate tổng thể | 60-70% | 85-95% | +15-25% |
| Latency first request | Cao (origin xa) | Trung bình | Ít request xa |

### 6.4 Khi Nào Dùng Origin Shield

```
✅ Dùng khi:
  - Origin ở 1 region, users ở toàn cầu
  - Content phổ biến (ít loại, nhiều request) — video, images
  - Origin yếu hoặc tốn phí bandwidth cao
  - Muốn bảo vệ origin khỏi traffic spikes

❌ Không cần khi:
  - Content rất đa dạng (long-tail: mỗi URL chỉ request 1-2 lần)
  - Origin đã rất mạnh và gần users
  - TTL rất ngắn (content thay đổi liên tục)
  
AWS CloudFront Origin Shield:
  - Chọn region gần origin nhất
  - Phí: thêm $0.0090 - $0.012 / 10,000 requests (tùy region)
  - Có thể tiết kiệm nhiều hơn phí Shield nhờ giảm origin fetch
```

---

## 7. Push CDN vs Pull CDN

### 7.1 Pull CDN (CDN Kéo — Phổ biến nhất)

**Ví dụ đời thường:** Cửa hàng 7-Eleven đặt hàng từ nhà kho KHI nào hết hàng. Nhà kho không tự gửi hàng → cửa hàng chủ động "kéo" hàng về.

```
Cách hoạt động Pull CDN:
1. User request → Edge location
2. Edge kiểm tra cache:
   - Cache Hit → Trả ngay (nhanh!)
   - Cache Miss → Edge "pull" (kéo) content từ Origin
3. Edge cache content + trả cho user
4. Requests tiếp theo → Edge trả từ cache

Origin KHÔNG cần biết CDN tồn tại
CDN tự quyết định cache cái gì, bao lâu
```

**Ưu điểm Pull CDN:**
- Setup đơn giản: Chỉ cần trỏ CDN về origin
- Tự động: CDN tự cache nội dung phổ biến
- Storage efficient: Chỉ cache content CÓ người yêu cầu
- Origin không cần thay đổi gì

**Nhược điểm Pull CDN:**
- First request chậm (cache miss → phải lấy từ origin)
- Content ít phổ biến có thể bị evict (xóa) khỏi cache
- Không kiểm soát được chính xác cái gì được cache ở đâu

**Ví dụ:** AWS CloudFront, Cloudflare, Fastly, Akamai

### 7.2 Push CDN (CDN Đẩy)

**Ví dụ đời thường:** Nhà xuất bản in sách xong, chủ động phân phối đến TẤT CẢ hiệu sách trên toàn quốc TRƯỚC khi ngày ra mắt. Ngày ra mắt, khách đến hiệu sách nào cũng có sách ngay.

```
Cách hoạt động Push CDN:
1. Bạn UPLOAD content lên CDN storage (chủ động đẩy)
2. CDN phân phối content đến TẤT CẢ edge locations
3. User request → Edge luôn có sẵn → trả ngay
4. Bạn quản lý content: thêm, xóa, update thủ công

Origin có thể là CDN storage luôn (S3, GCS)
Bạn kiểm soát 100% content nào ở đâu
```

**Ưu điểm Push CDN:**
- KHÔNG có first-request latency (content sẵn sàng ngay)
- Kiểm soát hoàn toàn: biết chính xác cái gì cached ở đâu
- Tốt cho file lớn: push sẵn 1 lần, serve triệu lần
- Origin không bị request (content ở CDN storage)

**Nhược điểm Push CDN:**
- Tốn storage (content push đến TẤT CẢ edges dù chưa ai cần)
- Quản lý phức tạp: bạn phải tự upload/delete/update
- Không tự động adapt theo popularity
- Tốn bandwidth upload ban đầu

**Ví dụ:** S3 + CloudFront (origin = S3), Akamai NetStorage

### 7.3 So Sánh và Khi Nào Dùng

| Tiêu chí | Pull CDN | Push CDN |
|----------|----------|----------|
| Setup | Đơn giản (point to origin) | Phức tạp (upload content) |
| First request | Chậm (cache miss) | Nhanh (sẵn sàng) |
| Storage cost | Thấp (on-demand) | Cao (replicate everywhere) |
| Management | Tự động | Thủ công |
| Best for | Web apps, APIs, dynamic | Video VOD, software downloads, game patches |
| Content freshness | Tự update theo TTL | Phải push update thủ công |
| Traffic pattern | Unpredictable | Planned (launch, events) |

**Thực tế:** Hầu hết CDN hiện đại là **Hybrid** — Pull CDN với khả năng pre-warm (push content trước event lớn).

---

## 8. SSL/TLS Tại Edge — Bảo Mật Ở Biên

### 8.1 Vấn Đề TLS Handshake + Khoảng Cách

```
Không CDN — TLS handshake với origin xa:
Client (VN) ←→ Origin (US): mỗi round trip = 130ms
TLS 1.2 handshake: 2 round trips = 260ms
TLS 1.3 handshake: 1 round trip = 130ms

Có CDN — TLS handshake với edge gần:
Client (VN) ←→ Edge (VN): mỗi round trip = 5ms
TLS 1.2 handshake: 2 round trips = 10ms
TLS 1.3 handshake: 1 round trip = 5ms

Tiết kiệm: 120-255ms chỉ riêng TLS handshake!
```

### 8.2 Certificate Tại Edge

**Cách CDN quản lý SSL cert:**
```
1. Bạn upload certificate lên CDN (hoặc CDN auto-provision)
2. CDN deploy cert đến TẤT CẢ edge locations
3. Client kết nối HTTPS với edge gần nhất
4. TLS terminate tại edge → traffic decrypt
5. Edge → Origin: có thể HTTPS (re-encrypt) hoặc HTTP

Client ←HTTPS→ Edge PoP ←HTTPS/HTTP→ Origin
         ↑                    ↑
    Cert của bạn         Origin cert (nếu HTTPS)
```

**AWS CloudFront + ACM:**
- ACM (AWS Certificate Manager) cấp cert MIỄN PHÍ
- Auto-renew (không lo cert expire)
- Deploy đến tất cả CloudFront edge locations
- SNI (Server Name Indication) cho phép nhiều domains trên 1 IP

### 8.3 OCSP Stapling Tại Edge

```
Bình thường: Browser phải hỏi CA "cert này còn hợp lệ không?" (OCSP check)
→ Thêm 1 round trip đến CA server

OCSP Stapling: CDN edge tự hỏi CA định kỳ, "staple" (đính kèm) kết quả vào TLS handshake
→ Browser không cần hỏi CA → bớt 1 round trip
→ Privacy tốt hơn (CA không biết user truy cập site nào)
```

---

## 9. AWS CloudFront — CDN Của AWS

### 9.1 Kiến Trúc CloudFront

```
CloudFront Infrastructure:
├── 400+ PoPs (Points of Presence) globally
│   ├── Edge Locations (~90% PoPs) — gần users, cache phổ biến
│   └── Regional Edge Caches (~10% PoPs) — cache lớn hơn, ít location hơn
├── AWS Backbone Network — kết nối edge ↔ origin tốc độ cao
└── Origin Types:
    ├── S3 Bucket (static content)
    ├── ALB/NLB (dynamic applications)
    ├── EC2 instance
    ├── MediaStore/MediaPackage (video)
    └── Custom origin (any HTTP server)
```

### 9.2 CloudFront Behaviors

```
Distribution: cdn.example.com
├── Behavior 1: /api/*
│   ├── Origin: ALB (dynamic backend)
│   ├── Cache policy: No caching (forward all to origin)
│   ├── TTL: 0
│   └── Allowed methods: GET, POST, PUT, DELETE
│
├── Behavior 2: /static/*
│   ├── Origin: S3 bucket
│   ├── Cache policy: Cache everything
│   ├── TTL: 86400 (1 day)
│   └── Allowed methods: GET only
│
└── Default Behavior: *
    ├── Origin: ALB
    ├── Cache policy: TTL 300 (5 min)
    └── Allowed methods: GET, HEAD
```

### 9.3 CloudFront Functions vs Lambda@Edge

```
CloudFront Functions (lightweight):
- Chạy tại Edge Location (gần user nhất)
- Latency: < 1ms
- Memory: 2MB max
- Duration: < 10ms
- Use cases: URL redirect, header manipulation, cache key normalization
- Viewer Request/Response only

Lambda@Edge (powerful):
- Chạy tại Regional Edge Cache
- Latency: 5-50ms  
- Memory: 128-10240 MB
- Duration: 5-30 seconds
- Use cases: A/B testing, auth, image resize, dynamic HTML
- Viewer Request/Response + Origin Request/Response

Chọn CF Functions khi: Simple, fast, cheap (URL rewrite, header modify)
Chọn Lambda@Edge khi: Complex logic, external API calls, heavy computation
```

### 9.4 CloudFront Security

```
Protection layers:
├── AWS Shield Standard (miễn phí, built-in)
│   └── Layer 3/4 DDoS protection
├── AWS WAF integration
│   ├── Rate limiting
│   ├── SQL injection block
│   ├── XSS block
│   └── Geo-restriction
├── Origin Access Control (OAC)
│   └── S3 chỉ cho phép CloudFront truy cập (block direct S3 URL)
├── Signed URLs / Signed Cookies
│   └── Content chỉ accessible với valid signature (premium content)
└── Field-level Encryption
    └── Encrypt sensitive form fields at edge (credit card number)
```

### 9.5 CloudFront Pricing

```
Cấu thành chi phí:
1. Data Transfer Out (80-90% chi phí)
   - Đến Internet: $0.085/GB (US/EU) - $0.170/GB (India)
   - Đến Origin: $0.020/GB (in same region)
   
2. HTTP/S Requests
   - HTTP: $0.0075 / 10,000 requests (US/EU)
   - HTTPS: $0.0100 / 10,000 requests

3. Invalidation
   - 1,000 paths/tháng miễn phí
   - $0.005/path sau đó

4. Origin Shield (optional)
   - $0.0090 / 10,000 requests

Tiết kiệm:
- Price Class: Chỉ dùng edge ở regions cần → giảm giá
- Reserved Capacity: Cam kết 1 năm → discount 20-40%
- Origin in same region → data transfer miễn phí
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 CDN Decision Flowchart

```
Bạn có cần CDN không?

Users ở nhiều vùng địa lý? → CÓ, dùng CDN
Content chủ yếu là static (images, CSS, JS)? → CÓ, đặc biệt hiệu quả
Traffic spikes thường xuyên (Black Friday, viral)? → CÓ, CDN absorb spike
Cần bảo vệ origin? → CÓ, CDN che giấu origin IP
Budget rất thấp, traffic rất ít? → CÓ THỂ KHÔNG (CDN has minimum cost)
Content 100% personalized, real-time? → CDN ít giúp (nhưng vẫn giúp TLS)
```

### 10.2 Cache Strategy Summary

```
┌─────────────────────────────────────────┐
│         Content Type Decision Tree       │
├─────────────────────────────────────────┤
│                                         │
│  Static assets (CSS/JS/images):         │
│  → Versioned URL + TTL 1 year + immutable│
│                                         │
│  HTML pages:                            │
│  → TTL 5-15 min + stale-while-revalidate│
│                                         │
│  API responses (public):                │
│  → TTL 1-60 sec + s-maxage             │
│                                         │
│  User-specific content:                 │
│  → Cache-Control: private, no-store     │
│                                         │
│  Media (video/audio):                   │
│  → TTL 1 day - 1 week + Range support  │
│                                         │
└─────────────────────────────────────────┘
```

### 10.3 Key Metrics Để Monitor CDN

| Metric | Ý nghĩa | Target |
|--------|---------|--------|
| Cache Hit Ratio | % requests served from cache | > 85% |
| Origin Requests | Số requests đến origin | Càng thấp càng tốt |
| Edge Latency (TTFB) | Time to first byte từ edge | < 50ms |
| Error Rate (4xx/5xx) | Lỗi từ CDN hoặc origin | < 0.1% |
| Bandwidth Saved | Data served from cache vs origin | > 80% |

### 10.4 Tài Liệu Tham Khảo

- RFC 7234: HTTP Caching — Định nghĩa chính thức về HTTP caching
- RFC 7232: Conditional Requests — ETag, If-None-Match, 304 Not Modified
- RFC 9213: Targeted HTTP Cache Control — CDN-specific cache directives
- RFC 9111: HTTP Caching (update RFC 7234) — phiên bản mới nhất
- AWS CloudFront Developer Guide: https://docs.aws.amazon.com/AmazonCloudFront/
- Cloudflare Learning: "What is a CDN?" — giải thích trực quan
- Google Web Fundamentals: HTTP Caching
- Akamai Technical Publications: CDN architecture papers

---

*Bài viết tiếp theo: DDoS Attacks & Mitigation — Cách đối phó tấn công "làm nghẽn đường"*
