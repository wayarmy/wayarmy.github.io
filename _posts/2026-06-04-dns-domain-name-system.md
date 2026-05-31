---
layout: post
title: "Networking Fundamentals - Phần 4: DNS — Hệ thống phân giải tên miền"
subtitle: "Hiểu cách trình duyệt tìm ra địa chỉ IP khi bạn gõ google.com"
gh-repo: wayarmy/wayarmy.github.io
tags: [networking, aws, learning-path]
comments: true
date: 2026-06-04
categories: AWS-Learning-Path
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Phần 4).
>
> **Đối tượng:** Người mới hoàn toàn — không cần kiến thức IT trước.
>
> **Nguồn tham khảo:**
> - RFC 1034 (1987) — Domain Names: Concepts and Facilities
> - RFC 1035 (1987) — Domain Names: Implementation and Specification
> - RFC 2181 — Clarifications to the DNS Specification
> - AWS Documentation: [Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/)
> - Tanenbaum, "Computer Networks" 5th Edition — Chapter 7: Application Layer (DNS)

---

## 1. DNS là gì? — "Danh bạ điện thoại" của Internet

### Ví dụ đời thường:

Hãy tưởng tượng bạn muốn gọi điện cho **Nhà hàng Phở Thìn** ở Hà Nội. Bạn KHÔNG nhớ số điện thoại (ví dụ: 024-3828-9069), nhưng bạn nhớ **tên** nhà hàng.

Bạn sẽ làm gì?
1. Mở **danh bạ** (hoặc Google) → gõ "Phở Thìn"
2. Danh bạ trả về số điện thoại: `024-3828-9069`
3. Bạn dùng số đó để gọi

**DNS (Domain Name System)** chính xác là "danh bạ" của Internet:
- **Tên miền** (domain name) = tên bạn nhớ → `google.com`, `facebook.com`
- **Địa chỉ IP** = "số điện thoại" thực sự → `142.250.80.46`

Con người nhớ **tên** dễ hơn số. Máy tính làm việc với **số** (IP). DNS là cầu nối giữa hai thế giới đó.

### Tại sao cần DNS?

Hãy tưởng tượng nếu mỗi lần muốn vào Facebook, bạn phải nhớ và gõ `157.240.1.35` thay vì `facebook.com`. Với hàng tỷ website, không ai có thể nhớ hết được. DNS giải quyết vấn đề này — bạn chỉ cần nhớ tên, DNS sẽ "dịch" thành số.

### Lịch sử ngắn gọn:

Trước khi có DNS (trước năm 1983), tất cả các máy tính trên Internet dùng một file duy nhất gọi là `HOSTS.TXT` — do SRI-NIC quản lý. Mỗi khi có máy tính mới, ai đó phải cập nhật file này và mọi người phải tải lại. Khi Internet phát triển lên hàng nghìn máy, cách này trở nên bất khả thi.

Paul Mockapetris đề xuất DNS trong **RFC 1034** và **RFC 1035** (tháng 11/1987), tạo ra hệ thống phân cấp, phân tán mà ta dùng đến ngày nay.

---

## 2. Cấu trúc phân cấp của DNS — "Cây gia phả" của tên miền

### Ví dụ đời thường:

Hãy nghĩ về **địa chỉ nhà** của bạn:
```
Phòng 502, Tòa A, Chung cư Vinhomes, Quận Cầu Giấy, Hà Nội, Việt Nam
```

Địa chỉ này có **cấu trúc phân cấp** — từ lớn đến nhỏ:
```
Việt Nam → Hà Nội → Quận Cầu Giấy → Chung cư Vinhomes → Tòa A → Phòng 502
```

DNS cũng tương tự, nhưng đọc **từ phải sang trái**:

```
www.mail.google.com.
│    │     │      │ │
│    │     │      │ └─ Root (gốc) — dấu chấm cuối
│    │     │      └─── TLD (Top-Level Domain): .com
│    │     └────────── SLD (Second-Level Domain): google
│    └──────────────── Subdomain: mail
└───────────────────── Subdomain: www
```

### Các tầng của DNS (theo RFC 1034, Section 3.1):

| Tầng | Tên | Ví dụ | Vai trò |
|------|-----|-------|---------|
| 0 | Root | `.` (dấu chấm) | Gốc của toàn bộ cây DNS |
| 1 | TLD (Top-Level Domain) | `.com`, `.vn`, `.org` | Phân loại theo mục đích/quốc gia |
| 2 | SLD (Second-Level Domain) | `google`, `facebook` | Tên tổ chức/dịch vụ đăng ký |
| 3+ | Subdomain | `www`, `mail`, `api` | Phân chia bên trong tổ chức |

### Root servers — "Thủ thư trưởng" của Internet:

Toàn bộ hệ thống DNS bắt đầu từ **13 cụm root server** (đánh tên từ A đến M). Chúng không lưu mọi tên miền, mà chỉ biết: *"Hỏi về `.com` thì đi hỏi server nào, hỏi về `.vn` thì đi đâu"*.

Con số 13 là giới hạn kỹ thuật: một DNS response phải vừa trong 512 bytes (UDP) theo RFC 1035 Section 2.3.4, và 13 bản ghi NS vừa đủ vừa trong giới hạn đó. Tuy nhiên, mỗi "root server" thực ra là **hàng trăm máy** phân bố khắp thế giới nhờ kỹ thuật Anycast.

---

## 3. Quá trình phân giải DNS — Từ tên miền đến IP

### Ví dụ đời thường:

Bạn là sinh viên mới, muốn tìm **phòng của Thầy Nguyễn Văn A** trong trường đại học:

1. **Hỏi bạn cùng phòng** (local cache): "Ê, phòng thầy A ở đâu?"
   - Nếu bạn biết → xong! (cache hit)
   - Nếu không biết → bước tiếp

2. **Đến phòng thường trực** (Recursive Resolver): "Cho em hỏi phòng thầy A?"
   - Phòng thường trực KHÔNG biết ngay, nhưng sẽ **đi hỏi hộ bạn**

3. Phòng thường trực hỏi **Ban giám hiệu** (Root server): "Thầy A thuộc khoa nào?"
   - Ban giám hiệu: "Thầy A thuộc **Khoa CNTT**. Đi hỏi văn phòng khoa."

4. Phòng thường trực hỏi **Văn phòng Khoa CNTT** (TLD server): "Thầy A ở phòng nào?"
   - Văn phòng khoa: "Thầy A thuộc **Bộ môn Mạng**. Đi hỏi trưởng bộ môn."

5. Phòng thường trực hỏi **Trưởng bộ môn Mạng** (Authoritative server): "Phòng thầy A?"
   - Trưởng bộ môn: "Phòng **B305**!" ← đây là câu trả lời cuối cùng

6. Phòng thường trực quay lại nói cho bạn: "**Phòng B305** nhé!"

### Kỹ thuật — Hai phương thức truy vấn (RFC 1034, Section 4.3.1-4.3.2):

#### Recursive Query (Truy vấn đệ quy):

```
Client → Recursive Resolver: "IP của www.google.com là gì?"
         (Resolver đi hỏi hộ từ đầu đến cuối)
Client ← Recursive Resolver: "142.250.80.46"
```

**Resolver làm hết việc cho bạn.** Bạn chỉ hỏi 1 lần và nhận kết quả cuối cùng. Giống như nhờ phòng thường trực đi tìm hộ.

#### Iterative Query (Truy vấn lặp):

```
Resolver → Root Server: "www.google.com?"
Resolver ← Root Server: "Tôi không biết, nhưng hỏi .com server ở 192.5.6.30"

Resolver → .com TLD Server: "www.google.com?"
Resolver ← .com TLD: "Hỏi Google NS ở ns1.google.com (216.239.32.10)"

Resolver → Google NS: "www.google.com?"
Resolver ← Google NS: "142.250.80.46" ✓
```

**Mỗi server chỉ "chỉ đường" đến server tiếp theo.** Resolver phải tự đi hỏi từng nơi.

### Toàn bộ quy trình thực tế:

```
Trình duyệt gõ: www.example.com

1. Browser cache → kiểm tra đã lưu chưa?
2. OS cache (file /etc/hosts trên Linux) → có sẵn không?
3. Stub Resolver gửi recursive query → Recursive Resolver (thường là ISP hoặc 8.8.8.8)
4. Recursive Resolver kiểm tra cache → nếu không có:
   4a. Hỏi Root Server → nhận referral đến .com TLD
   4b. Hỏi .com TLD → nhận referral đến NS của example.com
   4c. Hỏi Authoritative NS của example.com → nhận IP cuối cùng
5. Recursive Resolver cache kết quả (theo TTL)
6. Trả IP về cho client
7. Browser dùng IP để kết nối HTTP/HTTPS
```

---

## 4. Các loại DNS Record — "Các loại thông tin trong danh bạ"

### Ví dụ đời thường:

Quay lại danh bạ điện thoại. Khi tra "Nguyễn Văn A", bạn có thể tìm được nhiều loại thông tin:
- **Số điện thoại** chính → Giống record **A**
- **Số điện thoại 2** (VoIP/SIP) → Giống record **AAAA**
- **"Nếu gọi Anh A, hãy gọi số của anh B"** (chuyển tiếp) → Giống **CNAME**
- **"Gửi thư cho A qua địa chỉ này"** → Giống **MX**
- **"Quản lý mọi thứ của A là ông B"** → Giống **NS**
- **Ghi chú bổ sung** → Giống **TXT**

### Chi tiết từng loại record (RFC 1035, Section 3.2.2 - 3.3):

#### A Record (Address Record) — IPv4

**Mục đích:** Map tên miền thành địa chỉ IPv4 (32-bit).

```dns
example.com.    IN    A    93.184.216.34
www.google.com. IN    A    142.250.80.46
```

Đây là loại record phổ biến nhất — khi bạn gõ tên miền vào trình duyệt, cuối cùng trình duyệt cần một A record để biết IP đích.

#### AAAA Record — IPv6

**Mục đích:** Map tên miền thành địa chỉ IPv6 (128-bit). Gọi là "AAAA" (quad-A) vì IPv6 dài gấp 4 lần IPv4.

```dns
google.com.    IN    AAAA    2607:f8b0:4004:800::200e
```

#### CNAME Record (Canonical Name) — Alias

**Mục đích:** Tạo "bí danh" — trỏ tên này sang tên khác. Giống như "Nếu hỏi số của Tèo, thì tra số của Nguyễn Văn A" (Tèo là biệt danh).

```dns
www.example.com.    IN    CNAME    example.com.
blog.mysite.com.    IN    CNAME    mysite.github.io.
```

**Quy tắc quan trọng (RFC 1034, Section 3.6.2):** CNAME KHÔNG được tồn tại cùng record khác cho cùng tên. Nếu `blog.mysite.com` có CNAME, bạn không thể đặt thêm A record hay MX record cho nó.

#### MX Record (Mail Exchange)

**Mục đích:** Chỉ định mail server nhận email cho domain. Khi ai đó gửi email đến `user@example.com`, hệ thống sẽ tra MX record của `example.com`.

```dns
example.com.    IN    MX    10    mail1.example.com.
example.com.    IN    MX    20    mail2.example.com.
```

Số **10**, **20** là **priority** (ưu tiên) — số nhỏ hơn = ưu tiên cao hơn. Nếu `mail1` chết, email sẽ được gửi đến `mail2`.

**Ví dụ đời thường:** Giống như bạn có địa chỉ chính và địa chỉ phụ. Bưu điện gửi đến địa chỉ chính trước; nếu không giao được thì gửi đến địa chỉ phụ.

#### NS Record (Name Server)

**Mục đích:** Chỉ định DNS server nào chịu trách nhiệm (authoritative) cho domain.

```dns
example.com.    IN    NS    ns1.example.com.
example.com.    IN    NS    ns2.example.com.
```

**Ví dụ:** Giống như tổng đài 1080 nói: "Để biết thông tin về công ty ABC, hãy gọi đến số hotline riêng của họ là 1900-xxxx."

#### TXT Record

**Mục đích:** Lưu trữ text tự do — thường dùng để xác minh quyền sở hữu domain, cấu hình email security (SPF, DKIM, DMARC).

```dns
example.com.    IN    TXT    "v=spf1 include:_spf.google.com ~all"
example.com.    IN    TXT    "google-site-verification=abc123..."
```

**Ví dụ đời thường:** Giống như ghi chú dán trên cửa nhà: "Thư gửi cho gia đình này phải qua bưu điện X" hoặc "Xác nhận: đây đúng là nhà của ông A".

#### SOA Record (Start of Authority)

**Mục đích (RFC 1035, Section 3.3.13):** Chứa thông tin quản trị cho zone — ai là admin, serial number, thời gian refresh/retry/expire.

```dns
example.com.    IN    SOA    ns1.example.com. admin.example.com. (
                            2024010101  ; Serial
                            3600        ; Refresh (1 giờ)
                            900         ; Retry (15 phút)
                            604800      ; Expire (7 ngày)
                            86400       ; Minimum TTL (1 ngày)
                            )
```

#### PTR Record (Pointer) — Reverse DNS

**Mục đích:** Ngược lại với A record — tra từ IP ra tên miền. Dùng trong reverse DNS lookup.

```dns
34.216.184.93.in-addr.arpa.    IN    PTR    example.com.
```

---

## 5. TTL — "Hạn sử dụng" của thông tin DNS

### Ví dụ đời thường:

Bạn hỏi bạn cùng phòng: "Quán cơm gần trường mở cửa lúc mấy giờ?" Bạn ấy trả lời: "11h, nhưng thông tin này **chỉ đúng trong tuần này** — tuần sau họ có thể đổi giờ."

**TTL (Time To Live)** là "hạn sử dụng" của DNS record trong cache:
- TTL = 3600 → thông tin này có thể dùng lại trong 3600 giây (1 giờ)
- Sau 1 giờ, phải hỏi lại authoritative server để lấy thông tin mới

### Kỹ thuật (RFC 1035, Section 3.2.1):

TTL là trường 32-bit unsigned integer, đơn vị giây:
- **TTL cao** (86400 = 24h): Record ít thay đổi (ví dụ: MX record). Giảm tải cho DNS server.
- **TTL thấp** (60 = 1 phút): Record thay đổi thường xuyên (ví dụ: khi migration, failover). Tăng tải nhưng cập nhật nhanh.
- **TTL = 0**: Không cache, luôn hỏi mới. Dùng khi cần cập nhật tức thì.

### Chiến lược TTL thực tế:

| Tình huống | TTL khuyến nghị | Lý do |
|------------|-----------------|-------|
| Record bình thường | 3600 - 86400 | Cân bằng giữa performance và cập nhật |
| Chuẩn bị migration | 300 (5 phút) | Để khi đổi IP, mọi người cập nhật nhanh |
| Đang migration | 60 (1 phút) | Nhanh nhất có thể |
| Sau migration ổn định | 86400 (24h) | Giảm tải, tăng tốc |

---

## 6. DNS Zone và Zone Transfer

### Ví dụ đời thường:

Hãy nghĩ về **hệ thống hành chính Việt Nam**:
- Chính phủ quản lý cấp quốc gia
- UBND Tỉnh quản lý cấp tỉnh
- UBND Huyện quản lý cấp huyện

Mỗi cấp quản lý một "vùng" (zone) riêng. Chính phủ không trực tiếp quản lý đến từng hộ dân — họ **ủy quyền** (delegate) cho tỉnh, tỉnh ủy quyền cho huyện.

### Kỹ thuật:

**DNS Zone** = một phần của namespace DNS mà một tổ chức/server chịu trách nhiệm quản lý.

Ví dụ: `google.com` là một zone. Bên trong có thể có:
- `www.google.com` — A record
- `mail.google.com` — A record
- `cloud.google.com` — có thể delegate sang zone riêng

**Zone Transfer (AXFR/IXFR — RFC 5936):** Cơ chế sao chép dữ liệu zone từ Primary DNS server sang Secondary DNS server để đảm bảo tính sẵn sàng (availability).

```
Primary NS ──── Zone Transfer (AXFR) ────→ Secondary NS
  (master)                                    (slave)
```

- **AXFR (Full Transfer):** Copy toàn bộ zone — dùng khi Secondary mới setup
- **IXFR (Incremental Transfer):** Copy chỉ những thay đổi — tiết kiệm bandwidth

---

## 7. DNS Security — Những mối nguy hiểm

### DNS Cache Poisoning (Đầu độc cache)

**Ví dụ đời thường:** Ai đó lẻn vào phòng thường trực của trường và **sửa danh bạ**: thay vì ghi "Phòng tài vụ = Tầng 3", kẻ xấu sửa thành "Phòng tài vụ = Tầng 1 (phòng giả)". Sinh viên đến nộp tiền sẽ bị lừa.

**Kỹ thuật:** Attacker gửi DNS response giả mạo vào cache của Recursive Resolver, khiến resolver trả về IP sai cho user → user bị redirect đến website giả.

### DNSSEC (DNS Security Extensions — RFC 4033-4035):

Giải pháp: thêm **chữ ký số** vào DNS record. Giống như giấy tờ có **con dấu đỏ + chữ ký công chứng** — bạn biết chắc nó là thật.

DNSSEC thêm các record mới:
- **RRSIG**: Chữ ký số cho record
- **DNSKEY**: Public key để xác minh chữ ký
- **DS**: Delegation Signer — liên kết trust giữa parent zone và child zone

### DNS over HTTPS (DoH) và DNS over TLS (DoT):

DNS truyền thống gửi query dạng **plaintext** qua UDP port 53 — ai bắt gói tin giữa đường đều đọc được bạn đang truy cập trang nào.

- **DoH (RFC 8484):** Đóng gói DNS query trong HTTPS → port 443 → trông giống traffic web thường
- **DoT (RFC 7858):** DNS qua TLS → port 853 → mã hóa nhưng dùng port riêng

---

## 8. Công cụ DNS — Thực hành tra cứu

### nslookup (có sẵn trên mọi OS):

```bash
# Tra A record
$ nslookup google.com
Server:    8.8.8.8
Address:   8.8.8.8#53

Non-authoritative answer:
Name:    google.com
Address: 142.250.80.46

# Tra MX record
$ nslookup -type=MX google.com
google.com    mail exchanger = 10 smtp.google.com.
```

### dig (DNS lookup utility — phổ biến trên Linux/Mac):

```bash
# Tra chi tiết A record
$ dig www.example.com A

;; ANSWER SECTION:
www.example.com.    86400    IN    A    93.184.216.34

;; Query time: 24 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)

# Trace toàn bộ quá trình phân giải (giống iterative)
$ dig +trace www.google.com

# Tra NS record
$ dig NS example.com

# Tra bất kỳ record nào
$ dig ANY example.com
```

### host (đơn giản nhất):

```bash
$ host google.com
google.com has address 142.250.80.46
google.com has IPv6 address 2607:f8b0:4004:800::200e
google.com mail is handled by 10 smtp.google.com.
```

---

## 9. AWS Mapping — Amazon Route 53

### Tại sao tên "Route 53"?

DNS sử dụng **port 53** (cả TCP và UDP theo RFC 1035). AWS đặt tên dịch vụ DNS của họ là Route 53 — kết hợp "Route" (định tuyến) và "53" (port DNS). Đây cũng là cách chơi chữ với "Route 66" — con đường huyền thoại nước Mỹ.

### Route 53 là gì?

Amazon Route 53 là dịch vụ **Authoritative DNS** + **Domain Registration** + **Health Checking** của AWS:

| Chức năng | Mô tả |
|-----------|--------|
| DNS hosting | Lưu trữ DNS records cho domain của bạn |
| Domain registration | Đăng ký mua tên miền |
| Health check | Kiểm tra server có sống không, tự động chuyển traffic nếu chết |
| Traffic routing | Định tuyến traffic thông minh dựa trên nhiều tiêu chí |

### Hosted Zone:

**Hosted Zone** = DNS Zone trên Route 53. Chứa tất cả records cho một domain.

- **Public Hosted Zone:** Phân giải tên miền cho traffic từ Internet
- **Private Hosted Zone:** Phân giải chỉ trong VPC (mạng riêng AWS) — ví dụ: `database.internal.mycompany.com` chỉ hoạt động bên trong VPC

### Routing Policies (Chính sách định tuyến):

Route 53 không chỉ là DNS thông thường — nó hỗ trợ nhiều thuật toán routing thông minh:

| Policy | Mô tả | Ví dụ |
|--------|--------|-------|
| **Simple** | Trả về IP cố định | Website nhỏ, 1 server |
| **Weighted** | Phân bổ traffic theo tỷ lệ % | 90% đến v2, 10% đến v3 (canary deployment) |
| **Latency-based** | Trả về IP có latency thấp nhất với user | User VN → server Singapore, User US → server Virginia |
| **Failover** | Tự chuyển sang backup khi primary chết | Primary ở us-east-1, backup ở eu-west-1 |
| **Geolocation** | Route dựa trên vị trí địa lý | User VN → trang tiếng Việt, User JP → trang tiếng Nhật |
| **Geoproximity** | Route dựa trên khoảng cách + bias | Mở rộng/thu hẹp vùng phục vụ |
| **Multivalue Answer** | Trả về nhiều IP (có health check) | Kiểu load balancing đơn giản |

### Alias Record — Đặc biệt của Route 53:

Route 53 có loại record đặc biệt gọi **Alias** — giống CNAME nhưng:
- Hoạt động ở **zone apex** (root domain) — CNAME không được phép ở apex theo RFC
- **Miễn phí** query (CNAME bị tính tiền mỗi lần resolve)
- Tự động trả về IP của AWS resource (ALB, CloudFront, S3, etc.)

```
# CNAME - KHÔNG thể dùng cho root domain
example.com    CNAME    my-alb-1234.elb.amazonaws.com   ← KHÔNG HỢP LỆ!

# Alias - CÓ THỂ dùng cho root domain
example.com    A (Alias)    my-alb-1234.elb.amazonaws.com   ← OK!
```

### Kết hợp Route 53 với Health Check:

```
                    ┌──────────────┐
User → Route 53 ──→│ Health Check │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         Server A      Server B    Server C
         (healthy)     (unhealthy)  (healthy)
              ▲                        ▲
              └────── Traffic ──────────┘
                   (chỉ đến healthy servers)
```

---

## 10. Thực hành: Lab tự làm

### Lab 1: Tra cứu DNS records

```bash
# 1. Tra A record của facebook.com
dig facebook.com A +short

# 2. Tra MX record (mail server) của gmail.com
dig gmail.com MX

# 3. Tra NS record (name server) của amazon.com
dig amazon.com NS

# 4. Trace toàn bộ quá trình phân giải
dig +trace www.cloudflare.com

# 5. Tra reverse DNS (từ IP ra tên)
dig -x 8.8.8.8
```

### Lab 2: Đọc file /etc/hosts (local DNS override)

```bash
# Xem nội dung file hosts
cat /etc/hosts

# Thêm entry thử nghiệm (cần sudo)
echo "127.0.0.1    mytest.local" | sudo tee -a /etc/hosts

# Bây giờ ping mytest.local sẽ trỏ đến máy bạn
ping mytest.local
```

### Lab 3: So sánh thời gian phân giải (cache effect)

```bash
# Lần 1 - query đi ra Internet (chậm)
time dig @8.8.8.8 www.wikipedia.org

# Lần 2 - cache hit (nhanh hơn nhiều)
time dig @8.8.8.8 www.wikipedia.org

# Quan sát TTL giảm dần
watch -n 5 'dig @8.8.8.8 www.wikipedia.org | grep -A1 "ANSWER SECTION"'
```

### Lab 4: Sử dụng Route 53 (AWS Free Tier)

1. Đăng nhập AWS Console → Route 53
2. Tạo **Hosted Zone** (nếu có domain, hoặc dùng subdomain miễn phí)
3. Tạo các record:
   - A record: `app.yourdomain.com` → IP EC2 instance
   - CNAME: `www.yourdomain.com` → `app.yourdomain.com`
   - MX record: trỏ đến mail provider
4. Tạo **Health Check** cho EC2 instance
5. Tạo **Failover routing**: primary → EC2, secondary → S3 static website

---

## 11. Những lỗi DNS phổ biến và cách xử lý

| Triệu chứng | Nguyên nhân có thể | Cách kiểm tra |
|-------------|-------------------|---------------|
| "Server not found" | DNS không phân giải được | `nslookup domain.com` |
| Truy cập chậm | TTL quá thấp hoặc DNS server xa | `dig +stats domain.com` |
| Email không nhận được | MX record sai | `dig MX domain.com` |
| Website cũ sau khi migration | Cache TTL chưa hết | Đợi hoặc flush DNS cache |
| "NXDOMAIN" | Domain không tồn tại hoặc typo | Kiểm tra chính tả domain |

### Flush DNS Cache:

```bash
# macOS
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

# Windows
ipconfig /flushdns

# Linux (systemd-resolved)
sudo systemd-resolve --flush-caches
```

---

## 12. Tổng kết

| Khái niệm | Ví dụ đời thường | Kỹ thuật |
|-----------|-----------------|----------|
| DNS | Danh bạ điện thoại | Dịch tên miền → IP |
| A Record | Số điện thoại | Tên → IPv4 |
| AAAA Record | Số điện thoại mới (dài hơn) | Tên → IPv6 |
| CNAME | Biệt danh/bí danh | Tên → Tên khác |
| MX | Địa chỉ nhận thư | Domain → Mail server |
| NS | "Hỏi ai về vấn đề này?" | Domain → DNS server |
| TXT | Ghi chú dán cửa | Xác minh, SPF, DKIM |
| TTL | Hạn sử dụng thông tin | Thời gian cache (giây) |
| Recursive resolver | Phòng thường trực (đi hỏi hộ) | ISP DNS / 8.8.8.8 |
| Authoritative NS | Người biết câu trả lời cuối cùng | NS quản lý zone |
| Route 53 | Danh bạ thông minh của AWS | DNS + Health Check + Routing |

---

## Tài liệu tham khảo

1. **RFC 1034** — Mockapetris, P. (1987). "Domain Names - Concepts and Facilities". [https://www.rfc-editor.org/rfc/rfc1034](https://www.rfc-editor.org/rfc/rfc1034)
2. **RFC 1035** — Mockapetris, P. (1987). "Domain Names - Implementation and Specification". [https://www.rfc-editor.org/rfc/rfc1035](https://www.rfc-editor.org/rfc/rfc1035)
3. **RFC 2181** — Elz, R., Bush, R. (1997). "Clarifications to the DNS Specification". [https://www.rfc-editor.org/rfc/rfc2181](https://www.rfc-editor.org/rfc/rfc2181)
4. **Tanenbaum, A.S.** "Computer Networks" 5th Ed., Pearson — Chapter 7: Application Layer.
5. **AWS Route 53 Developer Guide** — [https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/)
6. **IANA Root Servers** — [https://www.iana.org/domains/root/servers](https://www.iana.org/domains/root/servers)

---

**Bài tiếp theo:** [Phần 5: HTTP, TCP/UDP và Ports — Giao thức giao tiếp trên Internet](/2026-06-05-http-tcp-udp-ports/)
