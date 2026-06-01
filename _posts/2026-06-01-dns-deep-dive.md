---
layout: post
title: "DNS Deep Dive - DNSSEC, DoH, DoT, Amplification Attacks, Split-Horizon & Resolver Internals"
date: 2026-06-01
categories: [networking]
tags: [dns, dnssec, doh, dot, security, networking]
---

# DNS Deep Dive — DNSSEC, DoH, DoT, Amplification Attacks, Split-Horizon & Resolver Internals

## Mục lục (Table of Contents)
1. [Giới thiệu bằng câu chuyện đời thường](#1-giới-thiệu-bằng-câu-chuyện-đời-thường)
2. [Tổng quan DNS và Resolver Internals](#2-tổng-quan-dns-và-resolver-internals)
3. [DNSSEC — Chữ ký số cho DNS](#3-dnssec--chữ-ký-số-cho-dns)
4. [RRSIG, DNSKEY, DS — Ba trụ cột của DNSSEC](#4-rrsig-dnskey-ds--ba-trụ-cột-của-dnssec)
5. [DNS over HTTPS (DoH) và DNS over TLS (DoT)](#5-dns-over-https-doh-và-dns-over-tls-dot)
6. [DNS Amplification Attacks — Tấn công khuếch đại DNS](#6-dns-amplification-attacks--tấn-công-khuếch-đại-dns)
7. [Split-Horizon DNS — Hai mặt của một tên miền](#7-split-horizon-dns--hai-mặt-của-một-tên-miền)
8. [Resolver Internals — Bên trong bộ phân giải DNS](#8-resolver-internals--bên-trong-bộ-phân-giải-dns)
9. [Hands-on Lab và Troubleshooting](#9-hands-on-lab-và-troubleshooting)
10. [Tổng kết và Best Practices](#10-tổng-kết-và-best-practices)

---

## 1. Giới thiệu bằng câu chuyện đời thường

### DNS như một danh bạ điện thoại khổng lồ

Hãy tưởng tượng bạn muốn gọi điện cho nhà hàng "Phở Hà Nội" nhưng không nhớ số điện thoại. Bạn sẽ mở danh bạ (phonebook) ra tra cứu. **DNS (Domain Name System)** chính là "danh bạ" của Internet — nó dịch tên miền dễ nhớ (như `google.com`) thành địa chỉ IP (như `142.250.80.46`) mà máy tính cần để liên lạc.

Nhưng danh bạ này có nhiều vấn đề an ninh:

| Vấn đề đời thường | Tương đương DNS |
|---|---|
| Ai đó sửa số điện thoại trong danh bạ, bạn gọi nhầm kẻ lừa đảo | **DNS Cache Poisoning** — kẻ tấn công chèn bản ghi giả |
| Ai đó nghe lén bạn đang tra cứu số nào | **DNS Eavesdropping** — ISP/hacker thấy bạn truy cập trang nào |
| Kẻ xấu mạo danh tổng đài danh bạ | **DNS Spoofing** — giả mạo server DNS |
| Kẻ xấu gọi hàng nghìn cuộc gọi tới nhà bạn | **DNS Amplification** — khuếch đại traffic tấn công |

Bài viết này sẽ đi sâu vào cách DNS giải quyết từng vấn đề trên, từ DNSSEC (chữ ký số xác thực), DoH/DoT (mã hóa), đến cách phòng chống tấn công.

### Tại sao DevOps/SRE cần hiểu sâu DNS?

- **DNS là single point of failure**: Nếu DNS chết, toàn bộ hệ thống "biến mất" khỏi Internet
- **DNS là vector tấn công phổ biến**: DDoS amplification, cache poisoning, DNS tunneling
- **DNS ảnh hưởng performance**: TTL sai = latency cao, resolver chậm = user chờ lâu
- **DNS là nền tảng của service discovery**: Kubernetes, AWS Route53, Consul đều dựa trên DNS

---

## 2. Tổng quan DNS và Resolver Internals

### 2.1 Kiến trúc phân cấp (Hierarchical Architecture)

DNS được thiết kế theo mô hình phân cấp (hierarchical), giống như hệ thống địa chỉ bưu chính: Quốc gia → Thành phố → Quận → Đường → Số nhà.

```
                    . (Root)
                   / | \
                  /  |  \
               .com .org .vn    ← Top-Level Domains (TLD)
              / |        |
         google amazon  example.vn  ← Second-Level Domains
           |
        mail.google.com  ← Subdomains (Tên miền phụ)
```

**Các thành phần chính:**

| Thành phần | Vai trò | Ví dụ đời thường |
|---|---|---|
| **Root Servers** (13 cụm, ký hiệu A-M) | Biết vị trí của TLD servers | Tổng đài quốc tế biết mã vùng các nước |
| **TLD Nameservers** (.com, .org, .vn) | Biết nameserver của domain | Tổng đài vùng biết số đường dây cục bộ |
| **Authoritative Nameservers** | Có câu trả lời cuối cùng cho domain | Nhân viên lễ tân biết số máy nhánh |
| **Recursive Resolver** (Resolver đệ quy) | Đi hỏi thay cho client | Nhân viên tổng đài tra cứu giúp bạn |
| **Stub Resolver** (trên máy client) | Gửi truy vấn tới recursive resolver | Bạn — người nhấc máy gọi tổng đài |

### 2.2 Quá trình phân giải DNS (Resolution Process)

Khi bạn gõ `www.example.com` trong trình duyệt:

```
Bước 1: Stub Resolver kiểm tra cache cục bộ
         ↓ (cache miss)
Bước 2: Hỏi Recursive Resolver (ISP hoặc 8.8.8.8)
         ↓ (cache miss)
Bước 3: Resolver hỏi Root Server → "Ai quản lý .com?"
         ↓ Trả lời: "Hỏi a.gtld-servers.net"
Bước 4: Resolver hỏi TLD Server (.com) → "Ai quản lý example.com?"
         ↓ Trả lời: "Hỏi ns1.example.com tại 93.184.216.34"
Bước 5: Resolver hỏi Authoritative Server → "IP của www.example.com?"
         ↓ Trả lời: "93.184.216.34, TTL=3600"
Bước 6: Resolver cache kết quả, trả về cho client
```

**Hai kiểu truy vấn:**

- **Recursive Query** (Truy vấn đệ quy): Client yêu cầu resolver phải trả lời đầy đủ (resolver đi hỏi thay)
- **Iterative Query** (Truy vấn lặp): Server trả lời "tôi không biết, nhưng hỏi server kia" (resolver tự đi hỏi tiếp)

```
[Client] ──recursive──→ [Recursive Resolver]
                              │
                    iterative │←→ [Root]
                    iterative │←→ [TLD]
                    iterative │←→ [Authoritative]
                              │
[Client] ←──answer────────────┘
```

### 2.3 DNS Record Types (Các loại bản ghi DNS)

| Record Type | Mục đích | Ví dụ |
|---|---|---|
| **A** | Tên → IPv4 | `example.com → 93.184.216.34` |
| **AAAA** | Tên → IPv6 | `example.com → 2606:2800:220:1::` |
| **CNAME** | Alias (bí danh) | `www → example.com` |
| **MX** | Mail server | `example.com → mail.example.com (priority 10)` |
| **NS** | Nameserver ủy quyền | `example.com → ns1.example.com` |
| **TXT** | Text tùy ý (SPF, DKIM, xác thực) | `"v=spf1 include:_spf.google.com ~all"` |
| **SOA** | Start of Authority — metadata zone | Serial, Refresh, Retry, Expire, Minimum TTL |
| **PTR** | Reverse DNS (IP → tên) | `34.216.184.93.in-addr.arpa → example.com` |
| **SRV** | Service discovery | `_http._tcp.example.com → server:8080` |
| **CAA** | CA Authorization | Chỉ CA nào được cấp cert cho domain |

### 2.4 DNS Caching và TTL

**TTL (Time To Live)** — thời gian cache sống — giống như "hạn sử dụng" trên hộp sữa:

```
example.com.    3600    IN    A    93.184.216.34
                ^^^^
                TTL = 3600 giây = 1 giờ
```

**Tầng cache:**

```
[Browser Cache] → [OS Cache] → [Router Cache] → [ISP Resolver Cache]
     5 phút         Theo TTL       Theo TTL          Theo TTL
```

**Trade-off của TTL:**

| TTL ngắn (60s) | TTL dài (86400s = 1 ngày) |
|---|---|
| ✅ Thay đổi IP lan nhanh | ✅ Giảm load lên DNS server |
| ✅ Failover nhanh | ✅ Giảm latency cho user |
| ❌ Nhiều query hơn = load cao | ❌ Thay đổi IP lan chậm |
| ❌ Latency cao hơn (phải query lại) | ❌ Failover chậm |

---

## 3. DNSSEC — Chữ ký số cho DNS

### 3.1 Vấn đề: DNS không có cơ chế xác thực

DNS nguyên bản (RFC 1035, năm 1987) được thiết kế khi Internet còn nhỏ và "tin tưởng lẫn nhau". Không có cơ chế nào để:
- Xác minh câu trả lời đến từ server đúng
- Phát hiện câu trả lời bị sửa đổi trên đường truyền

**Ví dụ đời thường**: Bạn gửi thư hỏi "địa chỉ ngân hàng ở đâu?" nhưng trên đường đi, kẻ xấu mở thư ra, thay nội dung thành địa chỉ giả, rồi dán lại. Bạn nhận thư tin tưởng vì phong bì trông bình thường.

### 3.2 DNSSEC giải quyết vấn đề gì?

**DNSSEC (DNS Security Extensions)** — định nghĩa trong RFC 4033, 4034, 4035 (2005) — thêm **chữ ký số (digital signatures)** vào DNS records. Nó đảm bảo:

| Thuộc tính | Ý nghĩa | DNSSEC cung cấp? |
|---|---|---|
| **Data Origin Authentication** (Xác thực nguồn gốc) | Biết câu trả lời đến từ server đúng | ✅ Có |
| **Data Integrity** (Toàn vẹn dữ liệu) | Biết dữ liệu không bị sửa đổi | ✅ Có |
| **Authenticated Denial of Existence** | Biết chắc domain không tồn tại (NXDOMAIN thật) | ✅ Có (NSEC/NSEC3) |
| **Confidentiality** (Bảo mật) | Mã hóa nội dung DNS | ❌ Không (cần DoH/DoT) |

**Quan trọng**: DNSSEC **KHÔNG mã hóa** DNS traffic. Nó chỉ **ký** (sign) để xác thực. Ai cũng có thể đọc nội dung, nhưng không thể giả mạo.

### 3.3 Chain of Trust (Chuỗi tin cậy)

DNSSEC hoạt động theo mô hình **chuỗi tin cậy từ trên xuống (top-down chain of trust)**, giống như hệ thống công chứng:

```
Ví dụ đời thường:
- Bộ Tư pháp (Root) → cấp phép cho Sở Tư pháp (.com TLD)
- Sở Tư pháp → cấp phép cho Văn phòng công chứng (example.com)
- Văn phòng công chứng → ký xác nhận tài liệu (A record)

Nếu bạn muốn kiểm tra tài liệu:
1. Kiểm tra chữ ký của Văn phòng công chứng
2. Kiểm tra Sở Tư pháp có ủy quyền cho Văn phòng này không
3. Kiểm tra Bộ Tư pháp có ủy quyền cho Sở này không
4. Trust anchor: Bạn tin tưởng Bộ Tư pháp (= root key)
```

```
[Root Zone]  ←── Trust Anchor (được cài sẵn trong resolver)
     │
     │ DS record (hash của DNSKEY .com)
     ↓
[.com TLD]   ←── Ký bằng DNSKEY của .com
     │
     │ DS record (hash của DNSKEY example.com)
     ↓
[example.com] ←── Ký bằng DNSKEY của example.com
     │
     │ RRSIG (chữ ký cho A record)
     ↓
[A Record: 93.184.216.34]  ←── Dữ liệu đã ký
```

### 3.4 ZSK và KSK — Hai loại khóa

DNSSEC sử dụng **hai cặp khóa** cho mỗi zone, giống như:
- **Khóa hàng ngày** (chìa khóa cửa phòng) = ZSK — dùng nhiều, thay thường xuyên
- **Khóa master** (chìa khóa két sắt) = KSK — dùng ít, bảo vệ cẩn thận hơn

| Loại khóa | Tên đầy đủ | Vai trò | Độ dài phổ biến | Chu kỳ thay |
|---|---|---|---|---|
| **ZSK** (Zone Signing Key) | Khóa ký zone | Ký tất cả RRsets trong zone | 1024-2048 bit RSA | 1-3 tháng |
| **KSK** (Key Signing Key) | Khóa ký khóa | Chỉ ký DNSKEY RRset | 2048-4096 bit RSA | 1-2 năm |

**Tại sao cần tách ZSK/KSK?**

Hãy tưởng tượng bạn có 1 triệu tài liệu cần ký. Nếu dùng 1 con dấu:
- Thay con dấu = phải ký lại 1 triệu tài liệu + thông báo cho tất cả mọi người

Nếu tách 2 con dấu:
- **Con dấu hàng ngày (ZSK)**: Ký 1 triệu tài liệu. Thay = ký lại tài liệu, nhưng không cần thông báo "trên" (parent zone)
- **Con dấu master (KSK)**: Chỉ ký xác nhận "con dấu hàng ngày là thật". Thay = chỉ cần cập nhật DS record ở parent

---

## 4. RRSIG, DNSKEY, DS — Ba trụ cột của DNSSEC

### 4.1 RRSIG Record (Resource Record Signature)

**RRSIG** là **chữ ký số** cho một tập bản ghi DNS (RRset). Mỗi RRset (tập các record cùng tên, lớp, type) đều có một RRSIG đi kèm.

**Ví dụ đời thường**: RRSIG giống như **chữ ký + con dấu** trên một văn bản pháp lý. Bạn đọc văn bản, rồi kiểm tra chữ ký để biết nó có hợp lệ không.

```dns
;; A record (dữ liệu gốc)
example.com.    3600    IN    A    93.184.216.34

;; RRSIG cho A record (chữ ký)
example.com.    3600    IN    RRSIG    A 13 2 3600 (
                    20260815000000   ; Signature Expiration (hết hạn)
                    20260716000000   ; Signature Inception (bắt đầu)
                    12345            ; Key Tag (ID của key đã ký)
                    example.com.     ; Signer's Name
                    base64-encoded-signature... )
```

**Các trường trong RRSIG:**

| Trường | Ý nghĩa | Ví dụ đời thường |
|---|---|---|
| Type Covered | Loại record được ký | "Tài liệu này là: Hợp đồng mua bán" |
| Algorithm | Thuật toán ký (13=ECDSA P-256) | "Con dấu loại: dấu tròn đỏ" |
| Labels | Số nhãn trong tên (bỏ wildcard) | Số cấp trong địa chỉ |
| Original TTL | TTL gốc (để verify) | Ngày gốc trên tài liệu |
| Expiration | Thời điểm chữ ký hết hạn | "Có hiệu lực đến: 15/08/2026" |
| Inception | Thời điểm chữ ký bắt đầu | "Có hiệu lực từ: 16/07/2026" |
| Key Tag | ID ngắn của khóa đã ký | Số serial con dấu |
| Signer's Name | Tên zone đã ký | "Đơn vị ký: Phòng công chứng X" |
| Signature | Chữ ký thực tế (Base64) | Mẫu chữ ký + con dấu |

**Quá trình ký:**

```
1. Lấy RRset gốc (tất cả A records của example.com)
2. Sắp xếp canonical (theo thứ tự chuẩn)
3. Hash toàn bộ RRset
4. Ký hash bằng private key (ZSK)
5. Đính kèm chữ ký thành RRSIG record
```

### 4.2 DNSKEY Record

**DNSKEY** chứa **khóa công khai (public key)** mà resolver dùng để xác minh chữ ký RRSIG.

**Ví dụ đời thường**: DNSKEY giống như **mẫu con dấu được công bố** (ai cũng xem được) để so sánh với con dấu trên tài liệu.

```dns
;; KSK (Key Signing Key) — flag 257
example.com.    3600    IN    DNSKEY    257 3 13 (
                    base64-encoded-public-key... )

;; ZSK (Zone Signing Key) — flag 256
example.com.    3600    IN    DNSKEY    256 3 13 (
                    base64-encoded-public-key... )
```

**Phân biệt KSK vs ZSK trong DNSKEY:**

| Flag | Giá trị bit | Loại khóa | Vai trò |
|---|---|---|---|
| 257 | Zone Key + SEP | KSK | Ký DNSKEY RRset; hash = DS record ở parent |
| 256 | Zone Key | ZSK | Ký tất cả RRsets khác trong zone |

**Thuật toán phổ biến (Algorithm field):**

| Số | Tên | Trạng thái |
|---|---|---|
| 5 | RSA/SHA-1 | ❌ Deprecated (không an toàn) |
| 8 | RSA/SHA-256 | ✅ Phổ biến (2048+ bit) |
| 13 | ECDSA P-256/SHA-256 | ✅ Khuyến khích (key nhỏ, nhanh) |
| 15 | Ed25519 | ✅ Mới nhất, hiệu quả nhất |

### 4.3 DS Record (Delegation Signer)

**DS record** là **cầu nối tin cậy** giữa parent zone và child zone. Nó chứa **hash** của KSK (DNSKEY 257) của child zone, được lưu ở parent zone.

**Ví dụ đời thường**: DS giống như **giấy ủy quyền** — Bộ Tư pháp giữ một bản sao "dấu vân tay con dấu" của Sở Tư pháp. Khi ai đó đưa tài liệu có dấu của Sở, bạn so vân tay con dấu với bản ở Bộ để xác nhận.

```dns
;; DS record tại .com (parent) cho example.com (child)
example.com.    86400    IN    DS    12345 13 2 (
                    a1b2c3d4e5f6...hash-hex... )
```

| Trường | Ý nghĩa |
|---|---|
| Key Tag (12345) | ID của KSK mà DS này tham chiếu |
| Algorithm (13) | Thuật toán của KSK (ECDSA P-256) |
| Digest Type (2) | Thuật toán hash (2=SHA-256) |
| Digest | Hash của DNSKEY record |

### 4.4 Quá trình xác thực đầy đủ (Validation Flow)

```
Resolver muốn xác thực: example.com A record

Bước 1: Nhận A record + RRSIG(A)
Bước 2: Nhận DNSKEY RRset của example.com (chứa ZSK + KSK)
Bước 3: Dùng ZSK (flag 256) để verify RRSIG(A) ← xác thực A record
Bước 4: Dùng KSK (flag 257) để verify RRSIG(DNSKEY) ← xác thực bộ DNSKEY
Bước 5: Nhận DS record từ .com zone
Bước 6: Hash KSK → so sánh với DS record ← liên kết lên parent
Bước 7: Dùng .com ZSK để verify RRSIG(DS) ← xác thực DS
Bước 8: Lặp lại... lên đến root
Bước 9: Trust anchor (root KSK được cài sẵn) → DONE ✅
```

### 4.5 NSEC và NSEC3 — Authenticated Denial of Existence

Khi domain không tồn tại, làm sao chứng minh? DNSSEC dùng **NSEC/NSEC3** để ký "khoảng trống":

**NSEC** (RFC 4034): Liệt kê record tiếp theo theo thứ tự alphabet
```dns
;; "Không có gì giữa alpha.example.com và beta.example.com"
alpha.example.com.  IN  NSEC  beta.example.com.  A AAAA RRSIG NSEC
```

**Vấn đề**: NSEC cho phép **zone walking** — kẻ tấn công lần lượt hỏi NSEC để liệt kê toàn bộ domain trong zone.

**NSEC3** (RFC 5155): Hash tên domain trước khi liệt kê
```dns
;; Thay vì "alpha" → "beta", dùng hash
ABC123.example.com.  IN  NSEC3  1 0 10 AABB (DEF456 A AAAA RRSIG)
;; Hash(alpha) = ABC123, Hash tiếp theo = DEF456
```

### 4.6 Key Rollover — Thay khóa an toàn

Khóa cần được thay định kỳ (giống đổi mật khẩu). Có 2 phương pháp chính:

**Pre-publish (cho ZSK):**
```
Giai đoạn 1: Công bố ZSK mới (chưa dùng ký)
Giai đoạn 2: Ký bằng ZSK mới + cũ (cả hai cùng tồn tại)
Giai đoạn 3: Gỡ ZSK cũ (sau khi cache cũ hết hạn)
```

**Double-DS (cho KSK):**
```
Giai đoạn 1: Tạo KSK mới, thêm DS mới ở parent
Giai đoạn 2: Ký DNSKEY RRset bằng KSK mới
Giai đoạn 3: Gỡ DS cũ ở parent
```

---

## 5. DNS over HTTPS (DoH) và DNS over TLS (DoT)

### 5.1 Vấn đề: DNS truyền thống không được mã hóa

DNS cổ điển gửi query/response dưới dạng **plaintext (bản rõ)** qua UDP port 53. Bất kỳ ai trên đường truyền (ISP, Wi-Fi quán cà phê, hacker) đều có thể:

1. **Đọc** bạn đang truy cập trang nào (eavesdropping)
2. **Chặn** và trả lời giả (man-in-the-middle)
3. **Ghi log** toàn bộ lịch sử duyệt web

**Ví dụ đời thường**: DNS plaintext giống như gửi bưu thiếp (ai cũng đọc được) thay vì thư trong phong bì kín (encrypted).

### 5.2 DNS over TLS (DoT) — RFC 7858

**DoT** mã hóa DNS bằng TLS, chạy trên **port 853 TCP**.

```
[Client] ───TLS 1.3 handshake───→ [DoT Resolver:853]
[Client] ←──Certificate + Key──── [DoT Resolver:853]
[Client] ───Encrypted DNS Query──→ [DoT Resolver:853]
[Client] ←──Encrypted DNS Answer── [DoT Resolver:853]
```

**Đặc điểm:**

| Thuộc tính | Chi tiết |
|---|---|
| Port | TCP 853 (dedicated) |
| Protocol | TLS 1.2/1.3 bọc DNS wire format |
| Nhận dạng | Dễ nhận ra (port 853 riêng) → dễ bị block |
| Latency | Thêm TLS handshake (1-2 RTT) |
| RFC | RFC 7858 (2016) |
| Ví dụ server | `dns.google:853`, `1.1.1.1:853` |

**Cấu hình DoT trên Linux (systemd-resolved):**
```ini
# /etc/systemd/resolved.conf
[Resolve]
DNS=1.1.1.1#cloudflare-dns.com
DNSOverTLS=yes
```

### 5.3 DNS over HTTPS (DoH) — RFC 8484

**DoH** gửi DNS queries qua **HTTPS (port 443)**, trông giống lưu lượng web bình thường.

```
[Client] ───HTTPS POST/GET──→ [DoH Resolver:443/dns-query]
         Content-Type: application/dns-message
         Body: <DNS wire format>
```

**Hai phương thức:**

```http
# GET (cacheable)
GET /dns-query?dns=AAABAAABAAAAAAAAA3d3dwdleGFtcGxlA2NvbQAAAQAB HTTP/2
Accept: application/dns-message

# POST (cho query lớn, không lộ qua URL)
POST /dns-query HTTP/2
Content-Type: application/dns-message
Content-Length: 33
<binary DNS message>
```

**Đặc điểm:**

| Thuộc tính | Chi tiết |
|---|---|
| Port | TCP 443 (chung với HTTPS) |
| Protocol | HTTP/2 + TLS 1.3, payload = DNS wire format |
| Nhận dạng | Rất khó phân biệt với traffic HTTPS thường |
| Latency | Có thể multiplexing nhiều query (HTTP/2) |
| RFC | RFC 8484 (2018) |
| Ví dụ URL | `https://dns.google/dns-query`, `https://cloudflare-dns.com/dns-query` |

### 5.4 So sánh DoH vs DoT vs DNS truyền thống

| Tiêu chí | DNS (UDP:53) | DoT (TCP:853) | DoH (TCP:443) |
|---|---|---|---|
| Mã hóa | ❌ Plaintext | ✅ TLS | ✅ TLS (HTTPS) |
| Xác thực server | ❌ | ✅ Certificate | ✅ Certificate |
| Dễ block | Rất dễ | Dễ (port 853) | Rất khó (lẫn với HTTPS) |
| Performance | Nhanh nhất | Chậm hơn (TLS overhead) | Tương tự DoT |
| Bypasses firewall | Không | Không (nếu block 853) | Có (port 443 luôn mở) |
| DNSSEC compatible | ✅ | ✅ | ✅ |
| Enterprise control | Dễ redirect | Trung bình | Khó kiểm soát |

### 5.5 Tranh cãi về DoH

**Ưu điểm (theo Mozilla, Cloudflare):**
- Bảo vệ privacy người dùng khỏi ISP theo dõi
- Chống censorship (firewall không block được)
- Chống man-in-the-middle attacks

**Nhược điểm (theo enterprise, ISP, quốc gia):**
- Bypass parental controls và enterprise policies
- Khó troubleshoot (DNS request lẫn trong HTTPS)
- Centralization — traffic DNS tập trung vào vài provider lớn (Google, Cloudflare)
- Malware có thể dùng DoH để bypass security monitoring

### 5.6 DNS over QUIC (DoQ) — RFC 9250

Giải pháp mới nhất (2022):
- Dùng QUIC (UDP + TLS 1.3)
- Nhanh hơn DoT (0-RTT handshake)
- Port 853 (UDP, không phải TCP)
- Tránh head-of-line blocking

---

## 6. DNS Amplification Attacks — Tấn công khuếch đại DNS

### 6.1 Nguyên lý: Phản xạ + Khuếch đại

**Ví dụ đời thường**: Hãy tưởng tượng bạn gửi một bưu thiếp nhỏ (vài byte) tới thư viện, xin photocopy toàn bộ cuốn bách khoa (hàng nghìn trang). Nhưng bạn ghi **địa chỉ người nhận là nhà hàng xóm**. Thư viện sẽ gửi hàng nghìn trang tới nhà hàng xóm. Bây giờ hãy tưởng tượng bạn gửi 1000 bưu thiếp như vậy — hàng xóm sẽ bị "ngập" trong bưu kiện.

**Ba yếu tố:**

1. **IP Spoofing** (Giả mạo IP nguồn): Attacker gửi DNS query với IP nguồn = IP nạn nhân
2. **Reflection** (Phản xạ): DNS resolver gửi response tới IP nạn nhân (vì tưởng nạn nhân hỏi)
3. **Amplification** (Khuếch đại): Response lớn hơn nhiều so với query (tỉ lệ 28x-54x)

```
                Query (60 bytes)
[Attacker] ─────────────────────→ [Open Resolver]
 IP src = victim                        │
                                        │ Response (3000+ bytes)
                                        ↓
                              [Victim Server]
                              (bị ngập traffic)
```

### 6.2 Tỉ lệ khuếch đại (Amplification Factor)

| Kỹ thuật | Query size | Response size | Tỉ lệ |
|---|---|---|---|
| ANY query (legacy) | ~60 bytes | ~3000 bytes | 50x |
| DNSSEC-signed domain | ~60 bytes | ~4000 bytes | 67x |
| TXT record lớn | ~40 bytes | ~4000 bytes | 100x |
| EDNS0 (4096 byte buffer) | ~60 bytes | ~4096 bytes | 68x |

**Ví dụ thực tế**: Attacker có 1 Gbps bandwidth:
- Gửi 1 Gbps queries giả mạo
- Open resolvers trả lại 50-70 Gbps tới victim
- Victim bị DDoS 50-70 Gbps chỉ với 1 Gbps attacker bandwidth

### 6.3 Tại sao DNS dễ bị khai thác?

1. **UDP protocol**: Không có handshake → dễ spoof IP
2. **Open resolvers**: Resolver chấp nhận query từ bất kỳ IP nào
3. **Response > Query**: Cấu trúc DNS cho phép response lớn hơn nhiều
4. **EDNS0**: Cho phép response tới 4096 bytes qua UDP
5. **DNSSEC**: Records có chữ ký → response càng lớn

### 6.4 Phòng chống (Mitigation)

**Phía DNS Operator (người quản lý DNS server):**

```bash
# 1. Không chạy open resolver — chỉ cho phép query từ IP nội bộ
# BIND9 configuration
options {
    allow-recursion { 10.0.0.0/8; 172.16.0.0/12; 192.168.0.0/16; };
    allow-query { 10.0.0.0/8; 172.16.0.0/12; 192.168.0.0/16; };
};

# 2. Response Rate Limiting (RRL) — giới hạn response/giây cho cùng IP
rate-limit {
    responses-per-second 5;    # Tối đa 5 response/giây cho cùng query
    window 5;                   # Cửa sổ 5 giây
    slip 2;                     # 1/2 response bị cắt thành TC (truncated)
};

# 3. Tắt ANY query type
minimal-any yes;    # Trả response tối thiểu cho ANY query
```

**Phía Network (ISP/Enterprise):**

```
# BCP38/BCP84 — Source Address Validation (SAV)
# ISP phải chặn packets có source IP không thuộc mạng của mình

# Ví dụ: ISP quản lý dải 203.0.113.0/24
# Chặn mọi packet đi ra với source IP KHÔNG phải 203.0.113.x
iptables -A FORWARD -s ! 203.0.113.0/24 -o eth0 -j DROP
```

**Phía Victim (người bị tấn công):**

| Biện pháp | Mô tả |
|---|---|
| DDoS protection service | Cloudflare, AWS Shield, Akamai — lọc traffic |
| Anycast routing | Phân tán traffic tới nhiều PoP |
| Rate limiting ingress | Giới hạn DNS response inbound |
| ACL block UDP:53 inbound | Nếu server không cần nhận DNS response |
| Upstream filtering | Nhờ ISP lọc trước khi tới server |

### 6.5 DNS Amplification vs. Các loại tấn công DDoS khác

| Loại | Protocol | Amplification | Phức tạp |
|---|---|---|---|
| DNS Amplification | UDP:53 | 28-54x | Thấp |
| NTP Amplification | UDP:123 | 556x | Thấp |
| Memcached Amplification | UDP:11211 | 51000x | Thấp |
| SSDP Amplification | UDP:1900 | 30x | Thấp |
| SYN Flood | TCP | 1x (không amplify) | Trung bình |
| HTTP Flood | TCP:80/443 | 1x | Cao |

---

## 7. Split-Horizon DNS — Hai mặt của một tên miền

### 7.1 Khái niệm

**Split-Horizon DNS** (còn gọi Split-View DNS, Split-Brain DNS) là kỹ thuật trả lời **khác nhau** cho cùng một tên miền, tùy thuộc vào **ai đang hỏi**.

**Ví dụ đời thường**: Khi khách hàng gọi đến công ty hỏi "Anh Minh ngồi đâu?", lễ tân trả lời "Tầng 2, phòng 201" (internal). Nhưng nếu ai đó gọi từ bên ngoài hỏi cùng câu, lễ tân chỉ nói "Anh Minh ở chi nhánh Quận 1" (external). Cùng câu hỏi, khác câu trả lời tùy nguồn.

### 7.2 Use Cases phổ biến

```
Scenario 1: Internal vs. External Access
─────────────────────────────────────────
                                    ┌─────────────────────┐
[Internet User]                     │   Internal Network   │
asks: app.company.com               │                     │
→ gets: 203.0.113.10 (Public IP)    │  [Employee]         │
                                    │  asks: app.company.com
                                    │  → gets: 10.0.1.50  │
                                    │    (Private IP)      │
                                    └─────────────────────┘
```

| Use Case | Internal Response | External Response |
|---|---|---|
| Web app | Private IP (10.x.x.x) | Public IP / Load Balancer |
| Mail server | Nội bộ (direct) | MX record (qua relay) |
| API endpoint | Internal service mesh | API Gateway |
| Dev/staging | staging.company.com → internal | Không tồn tại (NXDOMAIN) |

### 7.3 Cấu hình Split-Horizon với BIND9

```bash
# /etc/named.conf

# ACL định nghĩa "internal" network
acl "internal-nets" {
    10.0.0.0/8;
    172.16.0.0/12;
    192.168.0.0/16;
    localhost;
};

# View cho internal clients
view "internal" {
    match-clients { internal-nets; };
    
    zone "company.com" {
        type master;
        file "/etc/bind/zones/internal/company.com.zone";
    };
};

# View cho external clients (tất cả còn lại)
view "external" {
    match-clients { any; };
    
    zone "company.com" {
        type master;
        file "/etc/bind/zones/external/company.com.zone";
    };
};
```

**Internal zone file:**
```dns
; /etc/bind/zones/internal/company.com.zone
$TTL 300
@   IN  SOA  ns1.company.com. admin.company.com. (
            2026071601 3600 600 604800 300 )
@   IN  NS   ns1.company.com.
@   IN  A    10.0.1.50           ; Internal load balancer
www IN  A    10.0.1.50
api IN  A    10.0.2.100          ; Internal API
db  IN  A    10.0.3.200          ; Database (chỉ internal thấy)
```

**External zone file:**
```dns
; /etc/bind/zones/external/company.com.zone
$TTL 3600
@   IN  SOA  ns1.company.com. admin.company.com. (
            2026071601 3600 600 604800 300 )
@   IN  NS   ns1.company.com.
@   IN  A    203.0.113.10        ; Public IP
www IN  A    203.0.113.10
api IN  A    203.0.113.20        ; API Gateway public
; db KHÔNG CÓ ở đây → external query sẽ nhận NXDOMAIN
```

### 7.4 Split-Horizon trong AWS Route 53

AWS Route 53 hỗ trợ split-horizon thông qua **Private Hosted Zones**:

```
┌─────────────────────────────────────────┐
│            AWS Account                   │
│                                          │
│  ┌──────────────────┐                   │
│  │ VPC (10.0.0.0/16) │                   │
│  │                    │                   │
│  │  Private Hosted   │    Public Hosted  │
│  │  Zone:            │    Zone:          │
│  │  app.company.com  │    app.company.com│
│  │  → 10.0.1.50     │    → 54.x.x.x    │
│  │                    │                   │
│  │  [EC2 Instance]   │                   │
│  │  resolves app...  │                   │
│  │  → gets 10.0.1.50 │                   │
│  └──────────────────┘                   │
│                                          │
└─────────────────────────────────────────┘

[Internet User] resolves app.company.com → gets 54.x.x.x (Public)
```

### 7.5 Vấn đề và giải pháp

| Vấn đề | Giải pháp |
|---|---|
| Certificate mismatch (cert cho public IP, internal dùng private IP) | SAN certificate với cả hai IP, hoặc wildcard cert |
| DNS leak (internal record lộ ra ngoài) | Đảm bảo external view KHÔNG chứa internal records |
| Debugging khó (cùng domain, khác IP) | Luôn ghi log view nào được chọn; dùng `dig @server` chỉ định |
| DNSSEC phức tạp | Mỗi view cần DNSSEC riêng; hoặc chỉ sign external view |
| VPN split-tunnel | VPN client cần dùng internal DNS; split-tunnel có thể gây leak |

---

## 8. Resolver Internals — Bên trong bộ phân giải DNS

### 8.1 Kiến trúc của Recursive Resolver

**Ví dụ đời thường**: Resolver giống một **thám tử tư** — client thuê thám tử đi tìm thông tin, thám tử sẽ đi hỏi nhiều nguồn khác nhau và trả lại kết quả cuối cùng.

```
┌─────────────────────────────────────────────────────┐
│              Recursive Resolver                       │
│                                                       │
│  ┌───────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  Query    │  │   Cache      │  │  DNSSEC     │ │
│  │  Engine   │→ │   Manager    │→ │  Validator  │ │
│  │           │  │  (LRU/TTL)   │  │             │ │
│  └───────────┘  └──────────────┘  └──────────────┘ │
│       │                                    │         │
│       ↓                                    ↓         │
│  ┌───────────┐                    ┌──────────────┐ │
│  │  Iterator │                    │   Trust      │ │
│  │  (walks   │                    │   Anchors    │ │
│  │   the     │                    │   (root key) │ │
│  │   tree)   │                    └──────────────┘ │
│  └───────────┘                                      │
│       │                                              │
│       ↓ Sends iterative queries to:                 │
│  [Root] → [TLD] → [Authoritative]                  │
└─────────────────────────────────────────────────────┘
```

### 8.2 Cache Management

**Cấu trúc cache:**

```python
# Pseudocode — DNS cache entry
cache_entry = {
    "key": ("example.com", "A", "IN"),  # (name, type, class)
    "data": ["93.184.216.34"],
    "ttl_remaining": 2847,              # Giây còn lại
    "insertion_time": 1721100000,
    "original_ttl": 3600,
    "dnssec_status": "SECURE",          # SECURE / INSECURE / BOGUS
    "source": "authoritative",          # authoritative / additional / cache
}
```

**Negative caching** (cache kết quả "không tồn tại"):
- SOA record trong NXDOMAIN response chứa **Minimum TTL** = thời gian cache negative answer
- Giúp giảm load khi nhiều client hỏi cùng domain không tồn tại

```dns
;; NXDOMAIN response chứa SOA
example.com.  300  IN  SOA  ns1.example.com. admin.example.com. (
                   2026071601 3600 600 604800 300 )
                                                    ^^^
                                          Negative cache TTL = 300s
```

### 8.3 Query Processing Pipeline

```
1. RECEIVE query from client
   ↓
2. CHECK local cache
   → HIT: Return cached answer (nếu TTL chưa hết)
   → MISS: Continue to step 3
   ↓
3. DETERMINE starting point
   - Tìm delegation gần nhất đã biết trong cache
   - Ví dụ: biết NS của .com → bắt đầu từ .com (không cần hỏi root)
   ↓
4. ITERATE through DNS tree
   - Gửi iterative query tới authoritative server
   - Nhận referral (delegation) hoặc answer
   - Theo referral tới server tiếp theo
   ↓
5. VALIDATE (nếu DNSSEC enabled)
   - Verify RRSIG signatures
   - Walk chain of trust tới trust anchor
   - Nếu BOGUS → return SERVFAIL
   ↓
6. CACHE result (theo TTL)
   ↓
7. RETURN answer to client
```

### 8.4 Prefetching và Optimization

**Prefetch** (tải trước): Khi TTL gần hết (ví dụ < 10%), resolver tự động query lại ở background:

```
TTL = 3600s, Entry age = 3500s (còn 100s)
→ Resolver proactively queries authoritative server
→ Cache được refresh TRƯỚC KHI hết hạn
→ Client luôn nhận answer từ cache (no latency)
```

**QNAME Minimization** (RFC 7816): Gửi ít thông tin nhất có thể cho mỗi server:
```
Thay vì hỏi Root: "IP của www.secret.example.com là gì?"
Hỏi Root: "Ai quản lý .com?"          ← Root chỉ biết bạn hỏi .com
Hỏi .com: "Ai quản lý example.com?"   ← .com chỉ biết bạn hỏi example.com
Hỏi example.com NS: "IP của www.secret.example.com?"
```

### 8.5 Resolver phổ biến

| Resolver | License | Đặc điểm |
|---|---|---|
| **Unbound** | BSD | Nhẹ, bảo mật, DNSSEC validation tốt |
| **BIND9** (named) | MPL 2.0 | Full-featured, authoritative + recursive |
| **Knot Resolver** | GPL 3.0 | Modular, Lua scripting |
| **systemd-resolved** | LGPL | Tích hợp Linux, stub resolver |
| **dnsmasq** | GPL | Nhẹ, cho router/small networks |
| **PowerDNS Recursor** | GPL | High performance, scripting (Lua) |

### 8.6 DNS over TCP và Truncation

DNS query thường qua **UDP** (nhanh, nhẹ). Nhưng khi response > 512 bytes (hoặc EDNS0 buffer size):

```
1. Server gửi response với TC bit (Truncated) = 1
2. Client nhận TC=1 → thử lại qua TCP
3. TCP cho phép response không giới hạn kích thước
```

**EDNS0** (RFC 6891): Mở rộng UDP buffer size (mặc định 512 → tối đa 4096 bytes):
```dns
;; OPT record trong query
;; "Tôi chấp nhận response UDP tới 4096 bytes"
. 0 IN OPT 4096 ; UDP payload size = 4096
```

---

## 9. Hands-on Lab và Troubleshooting

### 9.1 Lab 1: Kiểm tra DNSSEC chain

```bash
# Kiểm tra domain có DNSSEC không
dig +dnssec example.com A

# Kiểm tra DNSKEY
dig example.com DNSKEY +short

# Kiểm tra DS record tại parent (.com)
dig example.com DS +short

# Trace toàn bộ chain of trust từ root
dig +trace +dnssec example.com A

# Validate DNSSEC (trả về "ad" flag = Authenticated Data)
dig @1.1.1.1 +dnssec example.com A
;; flags: qr rd ra ad;    ← "ad" = DNSSEC validated ✅

# Kiểm tra domain có DNSSEC broken không
dig @1.1.1.1 +dnssec dnssec-failed.org A
;; → SERVFAIL (validation failed) ❌
```

### 9.2 Lab 2: Test DoH/DoT

```bash
# Test DoH với curl
curl -s -H 'accept: application/dns-json' \
  'https://cloudflare-dns.com/dns-query?name=example.com&type=A' | jq .

# Test DoH POST (RFC 8484 wire format)
echo -n 'AAABAAABAAAAAAAAB2V4YW1wbGUDY29tAAABAAE=' | \
  base64 -d | \
  curl -s -H 'content-type: application/dns-message' \
    --data-binary @- \
    'https://dns.google/dns-query' | \
  python3 -c "import sys; import dns.message; print(dns.message.from_wire(sys.stdin.buffer.read()))"

# Test DoT với kdig (knot-dnsutils)
kdig +tls @1.1.1.1 example.com A

# Test DoT với openssl
echo -ne '\x00\x1d\xb2\x01\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00\x07example\x03com\x00\x00\x01\x00\x01' | \
  openssl s_client -connect 1.1.1.1:853 -quiet 2>/dev/null | xxd

# Kiểm tra DoT certificate
openssl s_client -connect 1.1.1.1:853 -showcerts </dev/null 2>/dev/null | \
  openssl x509 -text -noout | grep -A1 "Subject:"
```

### 9.3 Lab 3: Phát hiện Open Resolver

```bash
# Kiểm tra server có phải open resolver không
# (gửi query cho domain bên ngoài tới server)
dig @TARGET_IP google.com A +short

# Nếu trả lời IP → ĐÓ LÀ OPEN RESOLVER → cần fix!

# Kiểm tra bằng nmap
nmap -sU -p 53 --script dns-recursion TARGET_IP

# Output nếu open:
# PORT   STATE SERVICE
# 53/udp open  domain
# |_dns-recursion: Recursion appears to be enabled
```

### 9.4 Lab 4: Mô phỏng Split-Horizon

```bash
# Tạo Docker lab với BIND9 split-horizon
# docker-compose.yml
version: '3'
services:
  dns:
    image: ubuntu/bind9:latest
    ports:
      - "5353:53/udp"
      - "5353:53/tcp"
    volumes:
      - ./named.conf:/etc/bind/named.conf
      - ./zones:/etc/bind/zones

# Test từ "internal" network
dig @localhost -p 5353 app.company.com +short
# → 10.0.1.50

# Test từ "external" (nếu IP không match internal ACL)
dig @public-ip -p 5353 app.company.com +short
# → 203.0.113.10
```

### 9.5 Troubleshooting DNS phổ biến

| Triệu chứng | Nguyên nhân có thể | Cách kiểm tra |
|---|---|---|
| SERVFAIL | DNSSEC validation fail; server unreachable | `dig +cd` (disable validation) |
| NXDOMAIN | Domain không tồn tại; hoặc typo | `dig +trace` để xem break ở đâu |
| Timeout | Firewall block UDP:53; server down | `dig +tcp` (thử TCP); `nmap -sU -p 53` |
| Wrong IP | Cache cũ; DNS poisoning | `dig +trace` (bypass cache) |
| Slow resolution | Resolver chậm; high RTT tới authoritative | `dig +stats` (xem query time) |

```bash
# Debug toolkit
# 1. Bypass cache, trace từ root
dig +trace example.com A

# 2. Query specific nameserver
dig @ns1.example.com example.com A

# 3. Xem TTL remaining
dig example.com A | grep -A1 "ANSWER SECTION"

# 4. Reverse DNS check
dig -x 93.184.216.34

# 5. Kiểm tra tất cả record types
dig example.com ANY  # (một số server block ANY)

# 6. Xem EDNS0 options
dig +edns example.com A

# 7. Measure DNS latency
for i in $(seq 1 10); do dig @8.8.8.8 example.com +stats | grep "Query time"; done
```

---

## 10. Tổng kết và Best Practices

### 10.1 Tóm tắt kiến thức

```
┌─────────────────────────────────────────────────────────────────┐
│                    DNS Security Landscape                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────┐ │
│  │   DNSSEC    │    │  DoH / DoT  │    │  Split-Horizon      │ │
│  │             │    │             │    │                     │ │
│  │ Xác thực   │    │ Mã hóa      │    │ Kiểm soát truy cập │ │
│  │ (integrity)│    │ (privacy)   │    │ (access control)    │ │
│  │             │    │             │    │                     │ │
│  │ RRSIG      │    │ TLS 1.3     │    │ Internal/External   │ │
│  │ DNSKEY     │    │ HTTPS       │    │ views               │ │
│  │ DS         │    │ HTTP/2      │    │                     │ │
│  └─────────────┘    └─────────────┘    └─────────────────────┘ │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Threats                                    │ │
│  │  • Cache Poisoning  → DNSSEC prevents                       │ │
│  │  • Eavesdropping    → DoH/DoT prevents                     │ │
│  │  • Amplification    → RRL + BCP38 prevents                  │ │
│  │  • Data leak        → Split-Horizon prevents                │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Best Practices

**Cho DNS Server Operator:**

| # | Practice | Lý do |
|---|---|---|
| 1 | Bật DNSSEC signing cho authoritative zones | Ngăn cache poisoning |
| 2 | Không chạy open recursive resolver | Ngăn amplification attack |
| 3 | Bật Response Rate Limiting (RRL) | Giảm hiệu quả amplification |
| 4 | Dùng EDNS0 buffer size hợp lý (1232 bytes) | Tránh fragmentation |
| 5 | Automate key rollover (CDS/CDNSKEY) | Tránh key expire gây outage |
| 6 | Monitor DNSSEC expiry | RRSIG hết hạn = domain "chết" |
| 7 | Triển khai dual-stack (IPv4 + IPv6) | Đảm bảo reachability |

**Cho DevOps/SRE:**

| # | Practice | Lý do |
|---|---|---|
| 1 | Dùng TTL ngắn (60-300s) cho records cần failover nhanh | Faster recovery |
| 2 | TTL dài (3600s+) cho records ít thay đổi | Giảm latency, load |
| 3 | Luôn có fallback DNS resolver | Nếu primary resolver down |
| 4 | Monitor DNS resolution time | Phát hiện sớm vấn đề |
| 5 | Dùng health-check + DNS failover (Route53) | Tự động chuyển traffic |
| 6 | Cấu hình DoT/DoH cho internal resolvers | Bảo vệ privacy nội bộ |
| 7 | Log DNS queries cho security monitoring | Phát hiện DNS tunneling, malware C2 |

**Cho End Users:**

| # | Practice | Lý do |
|---|---|---|
| 1 | Dùng resolver hỗ trợ DNSSEC validation | Chống poisoning |
| 2 | Bật DoH/DoT trong browser/OS | Chống ISP snooping |
| 3 | Chọn resolver uy tín (1.1.1.1, 8.8.8.8, 9.9.9.9) | Performance + privacy |

### 10.3 Tài liệu tham khảo (References)

| Tài liệu | Nội dung |
|---|---|
| RFC 1035 | DNS Protocol Specification (1987) |
| RFC 4033-4035 | DNSSEC — Introduction, Records, Protocol |
| RFC 5155 | NSEC3 — Hashed Authenticated Denial of Existence |
| RFC 6891 | EDNS0 — Extension Mechanisms for DNS |
| RFC 7858 | DNS over TLS (DoT) |
| RFC 8484 | DNS over HTTPS (DoH) |
| RFC 9250 | DNS over QUIC (DoQ) |
| RFC 7816 | QNAME Minimisation |
| RFC 9704 | Split-Horizon in Validated Environments |
| BCP 38 (RFC 2827) | Network Ingress Filtering (anti-spoofing) |
| NIST SP 800-81-2 | Secure DNS Deployment Guide |

### 10.4 Câu hỏi ôn tập

1. DNSSEC bảo vệ khỏi những mối đe dọa nào? Không bảo vệ khỏi gì?
2. Sự khác biệt giữa ZSK và KSK? Tại sao cần tách?
3. DS record nằm ở đâu và vai trò của nó trong chain of trust?
4. DoH và DoT khác nhau thế nào? Khi nào dùng cái nào?
5. DNS amplification attack hoạt động ra sao? Ba biện pháp chống?
6. Split-horizon DNS giải quyết vấn đề gì? Rủi ro tiềm ẩn?
7. Recursive resolver khác iterative resolver thế nào?
8. Negative caching là gì? SOA Minimum TTL liên quan thế nào?
9. NSEC3 giải quyết vấn đề gì mà NSEC không giải quyết được?
10. EDNS0 là gì và tại sao DNSSEC cần nó?

---

*Bài viết được tham khảo từ RFC 4033-4035 (DNSSEC), RFC 7858 (DoT), RFC 8484 (DoH), RFC 9250 (DoQ), BCP 38, NIST SP 800-81-2, và tài liệu kỹ thuật của Cloudflare, Google Public DNS, ISC BIND.*
