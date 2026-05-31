---
layout: post
title: "Networking Fundamentals - Phần 5: HTTP, TCP/UDP và Ports"
subtitle: "Giao thức giao tiếp trên Internet — cách dữ liệu được truyền đi đáng tin cậy"
gh-repo: wayarmy/wayarmy.github.io
tags: [networking, aws, learning-path]
comments: true
date: 2026-06-01
categories: AWS-Learning-Path
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Phần 5).
>
> **Đối tượng:** Người mới hoàn toàn — không cần kiến thức IT trước.
>
> **Nguồn tham khảo:**
> - RFC 793 (1981) — Transmission Control Protocol (TCP)
> - RFC 768 (1980) — User Datagram Protocol (UDP)
> - RFC 9110 (2022) — HTTP Semantics (thay thế RFC 2616)
> - RFC 9112 (2022) — HTTP/1.1
> - Stevens, W.R. "TCP/IP Illustrated, Volume 1" — Chapters 17-20 (TCP), Chapter 11 (UDP)
> - AWS Documentation: [Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/), [Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)

---

## 1. Mở đầu — Tại sao cần giao thức?

### Ví dụ đời thường:

Hãy tưởng tượng bạn và người bạn ở Pháp muốn trao đổi thư. Để giao tiếp thành công, hai bên cần thống nhất **quy tắc**:
- Viết bằng ngôn ngữ nào? (Tiếng Anh? Tiếng Pháp?)
- Địa chỉ ghi ở đâu trên phong bì? (Góc trái trên? Góc phải?)
- Dán tem bao nhiêu? (Nội địa? Quốc tế?)
- Gửi qua đường nào? (Hàng không? Đường biển?)

Nếu không có quy tắc chung, thư sẽ **thất lạc** hoặc **không đọc được**.

**Giao thức (Protocol)** trong mạng máy tính chính là tập hợp quy tắc mà hai bên thống nhất để giao tiếp. Bài này sẽ đi sâu vào 3 giao thức quan trọng nhất: **TCP**, **UDP**, và **HTTP**.

---

## 2. TCP — "Thư bảo đảm" của Internet

### Ví dụ đời thường:

**TCP (Transmission Control Protocol)** giống như gửi **thư bảo đảm** qua bưu điện:
- Trước khi gửi, bạn **xác nhận** người nhận có ở nhà không (handshake)
- Mỗi bức thư được **đánh số** để người nhận sắp xếp đúng thứ tự
- Người nhận **báo lại** đã nhận mỗi bức thư (acknowledgment)
- Nếu thư bị mất, bưu điện **gửi lại** (retransmission)
- Khi xong, hai bên **chào tạm biệt** lịch sự (connection teardown)

### TCP Three-Way Handshake — "Bắt tay 3 bước" (RFC 793, Section 3.4):

Trước khi truyền dữ liệu, TCP phải thiết lập kết nối. Giống như gọi điện:

```
Client                          Server
  │                               │
  │──── SYN (seq=100) ──────────→│  "Alô, có nghe không?"
  │                               │
  │←─── SYN-ACK (seq=300,        │  "Nghe rồi! Nói đi!"
  │      ack=101) ────────────────│
  │                               │
  │──── ACK (seq=101,            │  "OK, bắt đầu nói nhé!"
  │      ack=301) ───────────────→│
  │                               │
  │═══════ Connection Established ═══════│
```

**Giải thích:**
1. **SYN** (Synchronize): Client gửi yêu cầu kết nối + sequence number ban đầu
2. **SYN-ACK**: Server đồng ý + gửi sequence number của mình + xác nhận đã nhận SYN
3. **ACK** (Acknowledge): Client xác nhận đã nhận SYN-ACK → kết nối thiết lập xong

**Tại sao 3 bước?** Để cả hai bên đều xác nhận được rằng đối phương có khả năng **gửi** VÀ **nhận**. Nếu chỉ 2 bước, Server không chắc Client có nhận được response hay không.

### Cấu trúc TCP Segment (RFC 793, Section 3.1):

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Source Port          │       Destination Port        │
├─────────────────────────────────────────────────────────────────┤
│                        Sequence Number                         │
├─────────────────────────────────────────────────────────────────┤
│                     Acknowledgment Number                      │
├───────┼───┼─┼─┼─┼─┼─┼─┼───────────────────────────────────────┤
│Offset │Res│U│A│P│R│S│F│           Window Size                 │
├─────────────────────────────────────────────────────────────────┤
│          Checksum             │       Urgent Pointer          │
├─────────────────────────────────────────────────────────────────┤
│                    Options (if any)                            │
├─────────────────────────────────────────────────────────────────┤
│                          Data                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Các trường quan trọng:**
- **Source/Destination Port:** Định danh ứng dụng gửi/nhận (16-bit → 0-65535)
- **Sequence Number:** Đánh số byte đầu tiên trong segment → giúp sắp xếp đúng thứ tự
- **Acknowledgment Number:** "Tôi đã nhận đến byte X, gửi byte X+1 tiếp đi"
- **Flags:** SYN, ACK, FIN, RST, PSH, URG — điều khiển trạng thái kết nối
- **Window Size:** Flow control — "Tôi còn chỗ nhận X bytes" (tránh gửi quá nhanh)
- **Checksum:** Phát hiện lỗi dữ liệu

### TCP đảm bảo độ tin cậy như thế nào?

| Tính năng | Cơ chế | Ví dụ đời thường |
|-----------|--------|-----------------|
| **Ordered delivery** | Sequence numbers | Đánh số trang sách để đọc đúng thứ tự |
| **Reliable delivery** | ACK + retransmission | Gửi thư bảo đảm: nhận biên lai, mất thì gửi lại |
| **Flow control** | Window size | "Đợi tôi đọc xong rồi hãy gửi tiếp!" |
| **Congestion control** | Slow start, AIMD | Đường đông → đi chậm lại để tránh kẹt thêm |
| **Error detection** | Checksum | Kiểm tra xem thư có bị rách/mất chữ không |

### TCP Connection Termination — "Tạm biệt 4 bước" (RFC 793, Section 3.5):

```
Client                          Server
  │                               │
  │──── FIN ─────────────────────→│  "Tôi nói xong rồi"
  │←─── ACK ─────────────────────│  "OK, tôi biết rồi"
  │                               │
  │←─── FIN ─────────────────────│  "Tôi cũng nói xong"
  │──── ACK ─────────────────────→│  "OK, tạm biệt!"
  │                               │
  │═══════ Connection Closed ════════│
```

**Tại sao 4 bước (không phải 3)?** Vì mỗi bên đóng **độc lập** — Client có thể nói xong nhưng Server vẫn đang gửi dữ liệu. Đây gọi là **half-close**.

---

## 3. UDP — "Bưu thiếp" nhanh gọn

### Ví dụ đời thường:

**UDP (User Datagram Protocol)** giống gửi **bưu thiếp**:
- KHÔNG cần xác nhận người nhận có ở nhà
- KHÔNG đánh số thứ tự
- KHÔNG biết bưu thiếp đến chưa
- Gửi rồi QUÊN — nhanh, gọn, nhẹ

**Khi nào dùng bưu thiếp?** Khi thông tin KHÔNG quan trọng đến mức phải bảo đảm, hoặc khi tốc độ quan trọng hơn độ chính xác.

### Cấu trúc UDP Datagram (RFC 768):

```
 0      7 8     15 16    23 24    31
+--------+--------+--------+--------+
|   Source Port   |  Dest Port      |
+--------+--------+--------+--------+
|     Length      |   Checksum      |
+--------+--------+--------+--------+
|          Data octets ...           |
+-----------------------------------+
```

**Chỉ 8 bytes header!** So với TCP (20-60 bytes header). Đơn giản hơn rất nhiều.

### So sánh TCP vs UDP:

| Đặc điểm | TCP | UDP |
|-----------|-----|-----|
| Kết nối | Connection-oriented (phải handshake) | Connectionless (gửi ngay) |
| Tin cậy | Đảm bảo giao hàng | Không đảm bảo |
| Thứ tự | Đảm bảo đúng thứ tự | Không đảm bảo |
| Tốc độ | Chậm hơn (do overhead) | Nhanh hơn |
| Header | 20-60 bytes | 8 bytes |
| Flow control | Có (Window) | Không |
| Use case | Web, email, file transfer | Video call, game, DNS query |

### Khi nào dùng TCP vs UDP?

| Ứng dụng | Protocol | Lý do |
|-----------|----------|-------|
| Web browsing (HTTP/HTTPS) | TCP | Cần load đầy đủ trang, đúng thứ tự |
| Email (SMTP, IMAP) | TCP | Không thể mất email |
| File transfer (FTP/SCP) | TCP | File phải nguyên vẹn |
| Video call (Zoom, Meet) | UDP | Mất 1-2 frame OK, delay mới là vấn đề |
| Online gaming | UDP | Tốc độ > chính xác tuyệt đối |
| DNS query | UDP (chủ yếu) | Query nhỏ, cần nhanh; fallback TCP nếu > 512 bytes |
| Streaming (Netflix) | TCP (HTTP-based) | Dùng buffer để bù delay |
| VoIP | UDP (RTP) | Real-time, không chờ retransmit |

---

## 4. Port Numbers — "Số phòng" trong tòa nhà

### Ví dụ đời thường:

IP address giống **địa chỉ tòa nhà** (số 123 đường Lê Lợi). Nhưng trong tòa nhà có nhiều **phòng ban**:
- Phòng 80: Lễ tân (Web Server)
- Phòng 443: Lễ tân VIP (Web Server encrypted)
- Phòng 22: Phòng IT (SSH)
- Phòng 25: Phòng thư (Email SMTP)
- Phòng 53: Tổng đài (DNS)

**Port number** cho phép một máy tính (một IP) chạy **nhiều dịch vụ** cùng lúc. Khi packet đến, OS dùng port number để biết **giao cho ứng dụng nào**.

### Phân loại Port (IANA - Internet Assigned Numbers Authority):

| Khoảng | Tên | Mô tả |
|--------|-----|--------|
| 0 - 1023 | **Well-Known Ports** | Dành cho dịch vụ chuẩn (cần quyền root) |
| 1024 - 49151 | **Registered Ports** | Đăng ký bởi ứng dụng/công ty |
| 49152 - 65535 | **Dynamic/Ephemeral Ports** | OS tự chọn cho kết nối outgoing |

### Các port quan trọng cần nhớ:

| Port | Protocol | Dịch vụ | Mô tả |
|------|----------|---------|--------|
| 20, 21 | TCP | FTP | File Transfer Protocol (data/control) |
| 22 | TCP | SSH | Secure Shell — remote login an toàn |
| 23 | TCP | Telnet | Remote login (KHÔNG mã hóa — tránh dùng!) |
| 25 | TCP | SMTP | Gửi email |
| 53 | TCP/UDP | DNS | Domain Name System |
| 80 | TCP | HTTP | Web không mã hóa |
| 110 | TCP | POP3 | Nhận email (download) |
| 143 | TCP | IMAP | Nhận email (sync) |
| 443 | TCP | HTTPS | Web có mã hóa TLS/SSL |
| 3306 | TCP | MySQL | Database MySQL |
| 5432 | TCP | PostgreSQL | Database PostgreSQL |
| 3389 | TCP | RDP | Remote Desktop (Windows) |
| 6379 | TCP | Redis | In-memory cache |
| 27017 | TCP | MongoDB | Database MongoDB |

### Socket — Kết hợp IP + Port:

**Socket** = IP:Port — định danh duy nhất một endpoint kết nối.

Một kết nối TCP được xác định bởi **4-tuple**:
```
(Source IP, Source Port, Destination IP, Destination Port)
```

Ví dụ:
```
Laptop bạn: 192.168.1.100:54321 → Google Server: 142.250.80.46:443
```

Đây là lý do một server có thể phục vụ **hàng nghìn client** cùng lúc trên cùng port 443 — vì mỗi kết nối có Source IP:Port khác nhau.

---

## 5. HTTP — "Ngôn ngữ" của Web

### Ví dụ đời thường:

Bạn đến quán cà phê:
1. Bạn **yêu cầu** (request): "Cho tôi một ly cà phê sữa đá"
2. Nhân viên **đáp lại** (response): "Đây, ly cà phê sữa đá của anh/chị"

HTTP (Hypertext Transfer Protocol) hoạt động theo mô hình **Request-Response**:
- Client (trình duyệt) gửi **HTTP Request**
- Server gửi lại **HTTP Response**

### HTTP Request — Cấu trúc (RFC 9112, Section 3):

```http
GET /search?q=hello HTTP/1.1
Host: www.google.com
User-Agent: Mozilla/5.0
Accept: text/html
Accept-Language: vi-VN,vi;q=0.9,en;q=0.8
Connection: keep-alive

```

**Các thành phần:**
1. **Request Line:** `METHOD /path HTTP/version`
2. **Headers:** Metadata bổ sung (ai gửi, chấp nhận gì, cookie, etc.)
3. **Blank line:** Phân tách header và body
4. **Body** (tùy chọn): Dữ liệu gửi kèm (form data, JSON, etc.)

### HTTP Methods — "Các loại yêu cầu" (RFC 9110, Section 9):

| Method | Ý nghĩa | Ví dụ đời thường | Ví dụ kỹ thuật |
|--------|----------|------------------|----------------|
| **GET** | Lấy/đọc dữ liệu | "Cho tôi xem menu" | Mở trang web, tìm kiếm |
| **POST** | Gửi/tạo dữ liệu mới | "Tôi muốn đặt món này" | Submit form, upload file |
| **PUT** | Thay thế toàn bộ | "Đổi toàn bộ đơn hàng thành..." | Cập nhật user profile hoàn toàn |
| **PATCH** | Sửa một phần | "Đổi thêm đá thành ít đá" | Sửa 1 field trong record |
| **DELETE** | Xóa | "Hủy đơn hàng" | Xóa bài viết |
| **HEAD** | Như GET nhưng chỉ lấy header | "Quán còn mở không?" (không cần menu) | Kiểm tra file có tồn tại không |
| **OPTIONS** | Hỏi server hỗ trợ gì | "Quán phục vụ những gì?" | CORS preflight request |

**Idempotency (RFC 9110, Section 9.2.2):** GET, PUT, DELETE là **idempotent** — gọi nhiều lần kết quả giống gọi 1 lần. POST KHÔNG idempotent — gọi 2 lần có thể tạo 2 đơn hàng.

### HTTP Response — Cấu trúc:

```http
HTTP/1.1 200 OK
Content-Type: text/html; charset=UTF-8
Content-Length: 1234
Date: Thu, 04 Jun 2026 10:30:00 GMT
Server: gws
Set-Cookie: session=abc123; Path=/; HttpOnly

<!DOCTYPE html>
<html>
<head><title>Google</title></head>
...
</html>
```

### HTTP Status Codes — "Mã phản hồi" (RFC 9110, Section 15):

| Nhóm | Ý nghĩa | Ví dụ đời thường |
|------|----------|-----------------|
| **1xx** | Informational | "Đang xử lý, đợi chút..." |
| **2xx** | Success | "OK, đây là món bạn gọi!" |
| **3xx** | Redirection | "Quán chuyển sang địa chỉ mới, đi theo đường này" |
| **4xx** | Client Error | "Bạn gọi sai! / Không có quyền" |
| **5xx** | Server Error | "Bếp bị cháy! Chúng tôi đang sửa" |

**Các status code PHẢI nhớ:**

| Code | Tên | Mô tả |
|------|-----|--------|
| 200 | OK | Thành công |
| 201 | Created | Tạo mới thành công (POST) |
| 204 | No Content | Thành công nhưng không có body (DELETE) |
| 301 | Moved Permanently | URL đã đổi vĩnh viễn → redirect |
| 302 | Found | Redirect tạm thời |
| 304 | Not Modified | Dữ liệu chưa thay đổi → dùng cache |
| 400 | Bad Request | Request sai format |
| 401 | Unauthorized | Chưa đăng nhập/xác thực |
| 403 | Forbidden | Đã đăng nhập nhưng không có quyền |
| 404 | Not Found | Resource không tồn tại |
| 405 | Method Not Allowed | Dùng sai method (PUT khi chỉ cho phép GET) |
| 429 | Too Many Requests | Rate limited — gọi quá nhiều |
| 500 | Internal Server Error | Server bị lỗi (bug) |
| 502 | Bad Gateway | Proxy/LB không liên lạc được backend |
| 503 | Service Unavailable | Server quá tải hoặc bảo trì |
| 504 | Gateway Timeout | Backend không respond kịp |

### HTTP Headers quan trọng:

**Request Headers:**

| Header | Mục đích | Ví dụ |
|--------|----------|-------|
| `Host` | Domain đích (bắt buộc HTTP/1.1) | `Host: www.google.com` |
| `User-Agent` | Trình duyệt/client nào | `Mozilla/5.0 (Windows...)` |
| `Accept` | Client chấp nhận content-type nào | `text/html, application/json` |
| `Authorization` | Thông tin xác thực | `Bearer eyJhbGci...` |
| `Cookie` | Gửi cookie đã lưu | `session=abc123` |
| `Content-Type` | Loại dữ liệu trong body | `application/json` |

**Response Headers:**

| Header | Mục đích | Ví dụ |
|--------|----------|-------|
| `Content-Type` | Loại dữ liệu trả về | `text/html; charset=UTF-8` |
| `Content-Length` | Kích thước body (bytes) | `1234` |
| `Set-Cookie` | Server gửi cookie cho client lưu | `session=abc123; HttpOnly` |
| `Cache-Control` | Hướng dẫn caching | `max-age=3600, public` |
| `Location` | URL redirect (với 3xx) | `https://new-url.com` |

### HTTP/1.1 vs HTTP/2 vs HTTP/3:

| Đặc điểm | HTTP/1.1 (RFC 9112) | HTTP/2 (RFC 9113) | HTTP/3 (RFC 9114) |
|-----------|---------------------|--------------------|--------------------|
| Year | 1997 (revised 2022) | 2015 | 2022 |
| Transport | TCP | TCP | QUIC (over UDP) |
| Multiplexing | Không (1 request/connection) | Có (nhiều stream/connection) | Có |
| Header compression | Không | HPACK | QPACK |
| Server Push | Không | Có | Có |
| Head-of-line blocking | Có | TCP level | Không (stream-level) |

---

## 6. HTTPS — HTTP + TLS

### Ví dụ đời thường:

HTTP giống gửi **bưu thiếp** — ai cũng đọc được nội dung. HTTPS giống gửi thư trong **phong bì niêm phong** — chỉ người nhận mở được.

**HTTPS = HTTP over TLS (Transport Layer Security)**

```
HTTP:   Browser ←── plaintext ──→ Server     (ai bắt giữa đường đọc được hết)
HTTPS:  Browser ←── encrypted ──→ Server     (dữ liệu bị mã hóa, không đọc được)
```

HTTPS sử dụng **port 443** (thay vì port 80 của HTTP).

**Tại sao HTTPS quan trọng?**
- Mã hóa dữ liệu (mật khẩu, thẻ tín dụng, thông tin cá nhân)
- Xác thực server (chứng minh đây đúng là google.com, không phải trang giả)
- Toàn vẹn dữ liệu (phát hiện nếu ai sửa đổi giữa đường)

---

## 7. AWS Mapping — Load Balancer và Security Groups

### Elastic Load Balancing (ELB):

Load Balancer phân phối traffic đến nhiều server — giống **nhân viên lễ tân** hướng dẫn khách đến các quầy khác nhau:

```
                    ┌─────────────┐
Users ──────────────│ Load Balancer│
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
          Server 1     Server 2     Server 3
          (EC2)        (EC2)        (EC2)
```

**3 loại Load Balancer trên AWS:**

| Loại | Layer | Protocol | Use case |
|------|-------|----------|----------|
| **ALB** (Application LB) | Layer 7 | HTTP/HTTPS | Web apps, API, microservices |
| **NLB** (Network LB) | Layer 4 | TCP/UDP/TLS | Gaming, IoT, extreme performance |
| **GWLB** (Gateway LB) | Layer 3 | IP | Network appliances (firewall, IDS) |

#### ALB (Application Load Balancer) — Layer 7:

ALB hiểu HTTP/HTTPS — có thể route dựa trên:
- **Path:** `/api/*` → API servers, `/images/*` → Image servers
- **Host header:** `api.mysite.com` → backend A, `www.mysite.com` → backend B
- **HTTP method:** GET → read replicas, POST → write server
- **Query string:** `?platform=mobile` → mobile backend

```
                         ALB
                          │
            ┌─────────────┼──────────────┐
            │             │              │
    /api/* route    /web/* route   /static/* route
            │             │              │
         API EC2       Web EC2       S3 Bucket
```

#### NLB (Network Load Balancer) — Layer 4:

NLB hoạt động ở tầng TCP/UDP — KHÔNG đọc HTTP content:
- **Cực nhanh:** Hàng triệu requests/giây, latency < 1ms
- **Static IP:** Mỗi NLB có IP cố định (hoặc Elastic IP)
- **Preserve source IP:** Backend thấy IP thật của client
- Use case: Game server, VoIP, MQTT (IoT)

### Security Groups — "Bảo vệ" mức instance:

Security Group giống **bảo vệ tòa nhà** — kiểm soát ai được vào/ra:

```
Inbound Rules (ai được VÀO):
┌──────────┬──────────┬─────────────────┬───────────────┐
│ Type     │ Protocol │ Port Range      │ Source        │
├──────────┼──────────┼─────────────────┼───────────────┤
│ HTTP     │ TCP      │ 80              │ 0.0.0.0/0    │ ← Ai cũng vào được
│ HTTPS    │ TCP      │ 443             │ 0.0.0.0/0    │ ← Web traffic
│ SSH      │ TCP      │ 22              │ 203.0.113.5/32│ ← Chỉ IP của tôi
│ Custom   │ TCP      │ 3306            │ sg-webserver │ ← Chỉ web servers
└──────────┴──────────┴─────────────────┴───────────────┘

Outbound Rules (ai được RA):
┌──────────┬──────────┬─────────────────┬───────────────┐
│ Type     │ Protocol │ Port Range      │ Destination   │
├──────────┼──────────┼─────────────────┼───────────────┤
│ All      │ All      │ All             │ 0.0.0.0/0    │ ← Mặc định cho ra hết
└──────────┴──────────┴─────────────────┴───────────────┘
```

**Đặc điểm Security Group:**
- **Stateful:** Nếu cho traffic VÀO, response tự động được cho RA (không cần rule outbound)
- **Allow only:** Chỉ có rule ALLOW, không có DENY. Mọi thứ không được allow = bị deny
- **Instance level:** Áp dụng cho từng EC2 instance (hoặc ENI)
- **Có thể reference SG khác:** "Cho phép traffic từ SG của web server" → không cần hardcode IP

### Mô hình thực tế kết hợp:

```
Internet
    │
    ▼
┌─────────┐
│   ALB   │ ← Security Group: Allow 80, 443 from 0.0.0.0/0
└────┬────┘
     │
     ▼
┌─────────┐
│  EC2    │ ← Security Group: Allow 80 from ALB's SG only
│  (Web)  │
└────┬────┘
     │
     ▼
┌─────────┐
│   RDS   │ ← Security Group: Allow 3306 from EC2's SG only
│  (DB)   │
└─────────┘
```

**Nguyên tắc:** Mỗi tầng chỉ cho phép traffic từ tầng trước nó — **defense in depth** (phòng thủ nhiều lớp).

---

## 8. Thực hành: Lab tự làm

### Lab 1: Phân tích TCP handshake

```bash
# Dùng tcpdump bắt TCP handshake
sudo tcpdump -i any -c 10 'tcp[tcpflags] & (tcp-syn|tcp-ack) != 0' -nn host google.com

# Hoặc dùng Wireshark (GUI) - filter: tcp.flags.syn == 1
```

### Lab 2: Xem HTTP request/response

```bash
# Dùng curl xem chi tiết HTTP
curl -v https://httpbin.org/get

# Chỉ xem headers
curl -I https://www.google.com

# Gửi POST request
curl -X POST https://httpbin.org/post \
  -H "Content-Type: application/json" \
  -d '{"name": "test", "value": 123}'

# Xem HTTP/2
curl --http2 -v https://www.google.com 2>&1 | head -30
```

### Lab 3: Kiểm tra port đang mở

```bash
# Xem tất cả port đang listen trên máy
ss -tlnp    # Linux
netstat -an | grep LISTEN    # macOS

# Kiểm tra port cụ thể trên remote host
nc -zv google.com 443
nc -zv google.com 80

# Quét nhiều port
nmap -p 22,80,443,3306 your-server-ip
```

### Lab 4: So sánh TCP vs UDP

```bash
# TCP - reliable (download file nhỏ)
time curl -o /dev/null https://www.google.com

# UDP - dùng dig (DNS query dùng UDP)
dig @8.8.8.8 google.com    # Nhanh, UDP
dig @8.8.8.8 +tcp google.com    # Chậm hơn, TCP
```

### Lab 5: AWS - Tạo ALB + Security Groups

1. Tạo 2 EC2 instances (web servers) trong 2 AZs khác nhau
2. Cài web server đơn giản: `echo "Server $(hostname)" > /var/www/html/index.html`
3. Tạo **Target Group** chứa 2 EC2 instances
4. Tạo **ALB** với listener port 80 → forward đến Target Group
5. Tạo **Security Group cho ALB:** Allow 80 from `0.0.0.0/0`
6. Tạo **Security Group cho EC2:** Allow 80 from ALB's SG only
7. Truy cập ALB DNS name → thấy traffic phân bổ đến 2 servers

---

## 9. Kiến thức bổ sung: WebSocket và gRPC

### WebSocket (RFC 6455):

HTTP là **request-response** — client phải hỏi trước, server mới trả lời. Giống gọi điện: bạn phải gọi trước.

WebSocket cho phép **bi-directional, full-duplex** — cả hai bên gửi bất kỳ lúc nào. Giống **chat video** — ai cũng nói được bất kỳ lúc nào.

Use case: Chat real-time, game multiplayer, stock ticker, notifications.

### gRPC:

gRPC dùng HTTP/2 + Protocol Buffers — nhanh hơn REST/JSON cho giao tiếp giữa microservices.

---

## 10. Tổng kết

| Khái niệm | Ví dụ đời thường | Kỹ thuật |
|-----------|-----------------|----------|
| TCP | Thư bảo đảm | Connection-oriented, reliable |
| UDP | Bưu thiếp | Connectionless, fast |
| Port | Số phòng trong tòa nhà | Định danh ứng dụng (0-65535) |
| HTTP | Đặt hàng ở quán cafe | Request-Response protocol |
| HTTPS | Đặt hàng trong phòng kín | HTTP + TLS encryption |
| Status Code | Phản hồi từ nhân viên | 2xx OK, 4xx lỗi client, 5xx lỗi server |
| ALB | Lễ tân phân phối khách | Layer 7 Load Balancer |
| NLB | Ống dẫn tốc độ cao | Layer 4 Load Balancer |
| Security Group | Bảo vệ tòa nhà | Stateful firewall per instance |

---

## Tài liệu tham khảo

1. **RFC 793** — Postel, J. (1981). "Transmission Control Protocol". [https://www.rfc-editor.org/rfc/rfc793](https://www.rfc-editor.org/rfc/rfc793)
2. **RFC 768** — Postel, J. (1980). "User Datagram Protocol". [https://www.rfc-editor.org/rfc/rfc768](https://www.rfc-editor.org/rfc/rfc768)
3. **RFC 9110** — Fielding, R. et al. (2022). "HTTP Semantics". [https://www.rfc-editor.org/rfc/rfc9110](https://www.rfc-editor.org/rfc/rfc9110)
4. **RFC 9112** — Fielding, R. et al. (2022). "HTTP/1.1". [https://www.rfc-editor.org/rfc/rfc9112](https://www.rfc-editor.org/rfc/rfc9112)
5. **Stevens, W.R.** "TCP/IP Illustrated, Volume 1: The Protocols", Addison-Wesley.
6. **AWS ELB Documentation** — [https://docs.aws.amazon.com/elasticloadbalancing/](https://docs.aws.amazon.com/elasticloadbalancing/)
7. **AWS Security Groups** — [https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
8. **IANA Service Name and Port Registry** — [https://www.iana.org/assignments/service-names-port-numbers/](https://www.iana.org/assignments/service-names-port-numbers/)

---

**Bài tiếp theo:** [Phần 6: NAT, Firewalls và Load Balancing — Bảo vệ và phân phối traffic](/2026-06-01-nat-firewalls-load-balancing/)
