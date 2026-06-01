---
layout: post
title: "DDoS Attacks & Mitigation Deep Dive - Tấn Công Từ Chối Dịch Vụ"
date: 2026-06-01
categories: [networking]
tags: [ddos, security, aws-shield, waf, mitigation]
---

# DDoS Attacks & Mitigation Deep Dive - Tấn Công Từ Chối Dịch Vụ

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Hãy tưởng tượng bạn có một cửa hàng nhỏ trên phố, chỉ phục vụ được 20 khách cùng lúc. Một ngày, đối thủ cạnh tranh thuê **1,000 người giả vờ** vào cửa hàng bạn — họ không mua gì, chỉ đứng chen chúc, hỏi linh tinh, chiếm hết không gian. Kết quả: **khách hàng THẬT không thể vào được** → bạn mất doanh thu.

Đó chính là **DDoS Attack (Distributed Denial of Service — Tấn Công Từ Chối Dịch Vụ Phân Tán)**.

**Thêm ví dụ đời thường:**

| Kiểu tấn công | Ví dụ đời thường |
|---------------|-----------------|
| Volumetric (Nghẽn băng thông) | 1000 xe tải chặn đường cao tốc → xe thật không qua được |
| Protocol (Khai thác giao thức) | Gọi điện đến tổng đài rồi KHÔNG nói gì → tổng đài bận, khách thật gọi không được |
| Application (Tầng ứng dụng) | Vào nhà hàng, gọi 100 món phức tạp rồi cancel → bếp cháy hàng, khách thật chờ mãi |

**Quy mô thực tế:**
- Năm 2024-2026: Các cuộc tấn công DDoS vượt **3 Tbps** (3 terabit/giây = 375 GB/giây) không còn hiếm
- Cloudflare mitigated 21.1 triệu DDoS attacks năm 2023, tăng 58% so với 2022
- Chi phí cho nạn nhân: $20,000-$40,000/giờ downtime trung bình

---

## 2. Kiến Thức Nền Tảng

### 2.1 DoS vs DDoS — Sự Khác Biệt

**DoS (Denial of Service):** 1 máy tấn công → 1 mục tiêu
```
[Attacker PC] ──flood──→ [Victim Server]
→ Dễ chặn: Block IP attacker = xong
```

**DDoS (Distributed DoS):** Hàng nghìn/triệu máy tấn công → 1 mục tiêu
```
[Bot 1 (Brazil)] ──────→
[Bot 2 (Vietnam)] ─────→
[Bot 3 (Japan)] ───────→ [Victim Server]
[Bot 4 (Germany)] ─────→
[... 100,000 bots] ────→
→ Không thể block: quá nhiều IP, traffic hợp pháp lẫn lộn
```

### 2.2 Botnet — Đội Quân Zombie

**Botnet** = mạng lưới các máy tính bị hack (zombies) do attacker điều khiển:
- Malware lây nhiễm hàng trăm nghìn thiết bị (PC, IoT cameras, routers)
- Attacker gửi lệnh từ C&C server (Command and Control)
- Tất cả bots đồng loạt tấn công mục tiêu

```
                    [C&C Server]
                   /    |    \
                  /     |     \
            [Bot1]  [Bot2]  [Bot3] ... [Bot N]
                  \     |     /
                   \    |    /
                    ↓   ↓   ↓
              [Victim Server] → OVERLOADED!
```

**Mirai Botnet (2016):** Lây nhiễm ~600,000 IoT devices (cameras, DVRs) → tấn công 1.2 Tbps vào Dyn DNS → làm sập Twitter, Netflix, Reddit, GitHub cùng lúc.

### 2.3 Ba Tầng Tấn Công Theo OSI

```
┌─────────────────────────────────────────────┐
│ Layer 7 (Application)    │ HTTP Flood        │
│ Application-layer attacks│ Slowloris         │
│                          │ DNS query flood   │
├──────────────────────────┼───────────────────┤
│ Layer 4 (Transport)      │ SYN Flood         │
│ Protocol attacks         │ ACK Flood         │
│                          │ TCP RST Flood     │
├──────────────────────────┼───────────────────┤
│ Layer 3 (Network)        │ UDP Flood         │
│ Volumetric attacks       │ ICMP Flood        │
│                          │ Amplification     │
└──────────────────────────┴───────────────────┘
```

---

## 3. Volumetric Attacks — Tấn Công Nghẽn Băng Thông (Layer 3/4)

### 3.1 Mục Tiêu

Làm **nghẽn đường truyền internet** của nạn nhân. Giống như đổ cát vào ống nước → nước thật không chảy qua.

Nạn nhân có đường internet 10 Gbps → attacker gửi 100 Gbps traffic → đường truyền tắc nghẽn hoàn toàn.

### 3.2 UDP Flood

**UDP** là giao thức "gửi rồi quên" — không cần kết nối, không cần xác nhận.

**Ví dụ đời thường:** Ai đó gửi hàng triệu bưu phẩm rác đến nhà bạn mỗi ngày. Bạn không đặt mua gì cả, nhưng hòm thư và lối vào bị ngập.

```
Cách tấn công:
1. Attacker gửi hàng triệu UDP packets/giây đến random ports của victim
2. Victim nhận packets → kiểm tra port → không có service → trả ICMP "Port Unreachable"
3. Lặp lại hàng triệu lần → CPU và bandwidth exhausted

UDP Flood đặc điểm:
- Source IP dễ bị spoofed (giả mạo) vì UDP stateless
- Không cần handshake → gửi ngay, không cần chờ
- Kích thước packet: thường 1400-1500 bytes (max MTU)
- Traffic: 10-1000+ Gbps
```

### 3.3 Amplification Attacks — Tấn Công Khuếch Đại

**Nguyên lý:** Gửi request NHỎ → nhận response LỚN. Như gửi 1 tin nhắn ngắn → nhận lại cuốn sách dày.

**Ví dụ đời thường:** Bạn gọi điện thoại đến 100 cửa hàng pizza, mỗi cuộc gọi 5 giây: "Giao 50 pizza đến nhà anh Minh ở địa chỉ X". Anh Minh (nạn nhân) sẽ nhận 5,000 hộp pizza mà không đặt → ngập nhà.

```
Amplification Attack Flow:
1. Attacker spoofs source IP = Victim's IP
2. Attacker gửi small queries đến open resolvers/reflectors
3. Resolvers gửi LARGE responses đến Victim IP

[Attacker] → small query (spoofed src=Victim) → [Open DNS Resolver]
[Open DNS Resolver] → BIG response → [Victim]

Attacker gửi: 60 bytes
Victim nhận: 4,000 bytes
Amplification factor: 66x !!!
```

**Các loại Amplification phổ biến:**

| Protocol | Port | Amplification Factor | Ví dụ |
|----------|------|---------------------|-------|
| DNS | 53 | 28-54x | Query ANY record → full zone |
| NTP | 123 | 556x | monlist command → list 600 clients |
| Memcached | 11211 | 51,000x !! | stats → dump all data |
| SSDP | 1900 | 30x | M-SEARCH → device descriptions |
| CLDAP | 389 | 56-70x | LDAP query → full directory |
| CharGen | 19 | ~358x | 1 byte → random characters |

**Memcached Amplification (2018):** Đáng sợ nhất — 1 byte request → 51,000 bytes response. Tấn công GitHub đạt 1.35 Tbps!

### 3.4 Carpet Bombing — Rải Bom Thảm

```
Thay vì tập trung vào 1 IP victim:
→ Tấn công TOÀN BỘ subnet /24 (256 IPs) của victim
→ Mỗi IP nhận lượng nhỏ (dưới threshold detection)
→ Tổng cộng vẫn massive
→ Khó detect vì mỗi IP riêng lẻ trông bình thường
```

---

## 4. Protocol Attacks — Tấn Công Khai Thác Giao Thức (Layer 3/4)

### 4.1 SYN Flood — Bắt Tay Dang Dở

**Để hiểu SYN Flood, cần hiểu TCP 3-way Handshake:**

```
Bình thường:
Client → [SYN] → Server       ("Xin chào, tôi muốn kết nối")
Client ← [SYN-ACK] ← Server   ("OK, tôi sẵn sàng")
Client → [ACK] → Server       ("Tuyệt, bắt đầu truyền data")
→ Connection established!
```

**SYN Flood:**
```
Attacker → [SYN] → Server     (Server allocate memory, chờ ACK)
Attacker → [SYN] → Server     (Server allocate thêm memory)
Attacker → [SYN] → Server     (Server allocate thêm...)
... 1 triệu SYN packets...
Attacker KHÔNG BAO GIỜ gửi ACK!

Server: half-open connections table = FULL
→ Server không thể chấp nhận connection MỚI từ clients thật
→ Denial of Service!
```

**Ví dụ đời thường:** 1,000 người gọi điện đến văn phòng bạn. Họ nói "Alo, tôi muốn..." rồi IM LẶNG, không cúp máy. Tổng đài hết slot → khách thật gọi không được.

**Đặc điểm:**
- Source IP spoofed → server gửi SYN-ACK vào hư không (không ai nhận)
- Mỗi half-open connection chiếm memory (thường 64 giây timeout)
- SYN Backlog thường chỉ 128-1024 entries
- Attack rate: 100K - 10M SYN/giây

### 4.2 ACK Flood & RST Flood

```
ACK Flood:
- Gửi hàng triệu ACK packets không thuộc connection nào
- Server phải kiểm tra connection table → CPU exhaustion
- Firewall phải track mỗi packet → state table overflow

RST Flood:
- Gửi TCP RST packets để đóng connections hợp lệ
- Nếu đoán đúng sequence number → connection bị kill
- Dùng để phá vỡ existing connections
```

### 4.3 Smurf Attack (Historical)

```
Attacker → ICMP Echo Request (spoofed src = Victim IP) → Broadcast address
TẤT CẢ hosts trong network → ICMP Echo Reply → Victim

Network có 200 hosts:
1 packet từ attacker → 200 replies đến victim
Amplification: 200x
```

**Lưu ý:** Hầu hết networks hiện đại đã disable directed broadcast → Smurf không còn phổ biến.

---

## 5. Application Layer Attacks — Tấn Công Tầng Ứng Dụng (Layer 7)

### 5.1 Tại Sao Layer 7 Nguy Hiểm Nhất?

**Ví dụ đời thường:** Thay vì chặn cửa hàng (volumetric), attacker giả làm khách hàng bình thường → vào cửa hàng → hỏi nhân viên 1000 câu hỏi phức tạp → nhân viên bận rộn → khách thật không được phục vụ.

```
Đặc điểm Layer 7 attacks:
- Traffic trông HOÀN TOÀN bình thường (valid HTTP requests)
- Volume NHỎ (có thể chỉ 10,000 requests/giây → đủ crash server)
- Rất khó phân biệt với traffic thật
- Bypass được L3/L4 protection (vì traffic hợp lệ ở layer dưới)
- Tốn ít bandwidth cho attacker, hiệu quả cao
```

### 5.2 HTTP Flood

```
HTTP GET Flood:
- Gửi hàng triệu HTTP GET requests đến URLs nặng
- Ví dụ: GET /search?q=very+complex+query (search tốn CPU)
- Ví dụ: GET /report/generate?from=2020&to=2024 (report generation)
- Mỗi request hợp lệ → firewall không block được

HTTP POST Flood:
- Gửi form submissions, login attempts, API calls
- Ví dụ: POST /login (server phải hash password check)
- Ví dụ: POST /upload (server allocate memory)
- POST thường tốn server resources hơn GET
```

**So sánh với Volumetric:**
```
Volumetric: 100 Gbps traffic → nghẽn đường
Layer 7:    1 Mbps traffic → crash server (vì mỗi request tốn CPU)

Ví dụ: 1 request "GET /search?q=..." tốn server 100ms CPU
       10,000 requests/giây = server cần 1,000 giây CPU/giây = IMPOSSIBLE
       Nhưng bandwidth chỉ ~5 Mbps (rất nhỏ!)
```

### 5.3 Slowloris — Chậm Như Rùa Nhưng Chết Như Bom

**Nguyên lý:** Mở nhiều connections đến web server, gửi headers RẤT CHẬM, KHÔNG BAO GIỜ hoàn thành request. Server giữ connection mở chờ → hết connection slots.

**Ví dụ đời thường:** 500 người vào nhà hàng, ngồi xuống, cầm menu... rồi nghĩ 10 tiếng không gọi món. Bồi bàn phải chờ → hết chỗ cho khách thật.

```
Slowloris Attack:
Attacker gửi (rất chậm):
"GET / HTTP/1.1\r\n"
"Host: victim.com\r\n"
"X-a: b\r\n"          ← Gửi header mỗi 10 giây
"X-c: d\r\n"          ← Server vẫn chờ "\r\n\r\n" (end of headers)
"X-e: f\r\n"          ← Connection vẫn mở...
... KHÔNG BAO GIỜ gửi "\r\n\r\n" ...

Lặp lại cho 10,000 connections:
→ Apache/nginx max_connections = full
→ Server không accept connection mới
→ CHỈ CẦN 1 máy attacker, bandwidth ~ 10 Kbps!
```

**Đặc điểm Slowloris:**
- Bandwidth cực thấp (vài Kbps từ 1 máy)
- Hiệu quả cao với Apache (prefork model, connection-per-thread)
- Ít hiệu quả với nginx/event-based servers (event loop, không block thread)
- Khó detect vì traffic rate rất thấp

### 5.4 R-U-Dead-Yet (RUDY)

```
Tương tự Slowloris nhưng cho POST requests:
1. Gửi POST request với Content-Length: 1000000 (claim 1MB body)
2. Gửi body 1 byte mỗi 10 giây
3. Server phải giữ connection mở chờ hết body
4. Lặp lại hàng nghìn connections → server hết resources
```

### 5.5 HTTP/2 Rapid Reset (CVE-2023-44487)

```
Khai thác HTTP/2 multiplexing:
1. Client mở 1 TCP connection
2. Gửi request stream → ngay lập tức RST_STREAM (cancel)
3. Lặp lại TRIỆU lần/giây trên 1 connection
4. Server phải allocate resources cho mỗi stream dù bị cancel ngay
5. Kết quả: Google observed 398 triệu requests/giây!

Đáng sợ vì:
- Chỉ cần 1 TCP connection (bypass connection limit)
- Nhìn từ network layer: chỉ 1 connection bình thường
- Bypass rate limiting per-IP (vẫn chỉ 1 IP, 1 connection)
```

---

## 6. Phương Pháp Phòng Thủ — Mitigation Strategies

### 6.1 Scrubbing Centers — Trung Tâm "Tẩy Rửa" Traffic

**Ví dụ đời thường:** Trước khi nước vào nhà, nước chảy qua máy lọc. Máy lọc loại bỏ tạp chất (traffic xấu) và chỉ cho nước sạch (traffic tốt) vào.

```
Bình thường (không scrubbing):
Internet → [Malicious + Legitimate traffic] → Your Server → CRASH

Có Scrubbing Center:
Internet → [All traffic] → Scrubbing Center → [Clean traffic only] → Your Server

Scrubbing Center làm gì:
1. Absorb toàn bộ traffic (kể cả 1 Tbps)
2. Phân tích từng packet/request
3. Drop traffic xấu (matching attack signatures)
4. Forward traffic sạch đến origin server
5. Latency overhead: 2-10ms
```

**Providers:** Cloudflare (200+ Tbps capacity), Akamai Prolexic, AWS Shield Advanced, Azure DDoS Protection

### 6.2 Rate Limiting — Giới Hạn Tốc Độ

**Ví dụ đời thường:** Cửa hàng quy định: mỗi người chỉ được mua tối đa 2 sản phẩm. Ai muốn mua 100 → bị chặn.

```
Rate Limiting Rules:
- Max 100 requests/giây per IP
- Max 10 login attempts/phút per IP
- Max 1000 requests/phút per API key
- Max 50 requests/giây per URI path

Khi vượt limit:
→ HTTP 429 Too Many Requests
→ Hoặc CAPTCHA challenge
→ Hoặc delay response (tarpit)
```

**Algorithms phổ biến:**
```
1. Token Bucket:
   - Bucket chứa max 100 tokens
   - Mỗi request lấy 1 token
   - Tokens được refill 10/giây
   - Hết token → reject
   - Cho phép burst ngắn (dùng hết 100 tokens cùng lúc)

2. Sliding Window:
   - Đếm requests trong cửa sổ thời gian di chuyển
   - Ví dụ: Max 100 requests trong bất kỳ window 60 giây nào
   - Chính xác hơn token bucket

3. Leaky Bucket:
   - Requests vào bucket, process ra đều đặn
   - Bucket đầy → drop requests mới
   - Smooth output rate (không burst)
```

### 6.3 SYN Cookies — Chống SYN Flood

```
Bình thường (dễ bị SYN Flood):
Server nhận SYN → allocate memory → lưu vào backlog → chờ ACK
Backlog full → reject connections mới

SYN Cookies (chống SYN Flood):
Server nhận SYN → KHÔNG allocate memory
→ Encode connection info vào SYN-ACK sequence number (cookie)
→ Gửi SYN-ACK
→ Nếu client GỬI ACK (có cookie hợp lệ):
  → Decode cookie → reconstruct connection → allocate memory
→ Nếu client KHÔNG gửi ACK (attacker):
  → Không tốn memory → không ảnh hưởng gì!

Kết quả: Server KHÔNG CẦN lưu half-open connections
→ SYN Flood không thể exhaust connection table
```

### 6.4 Anycast — Phân Tán Traffic Toàn Cầu

```
Bình thường (1 IP = 1 server):
Tất cả attack traffic → 1 server → overloaded

Anycast (1 IP = nhiều servers ở nhiều nơi):
Attack traffic phân tán tự động theo BGP routing:
- Traffic từ Brazil → Server Brazil
- Traffic từ Vietnam → Server Vietnam  
- Traffic từ Germany → Server Germany
→ Mỗi server chỉ nhận 1/N tổng traffic
→ Khó overwhelm khi traffic phân tán

Cloudflare: 200+ Tbps anycast capacity across 300+ cities
```

### 6.5 Geo-blocking và IP Reputation

```
Geo-blocking:
- Nếu business chỉ phục vụ Việt Nam
- Block ALL traffic không phải từ VN
- 80% attack traffic (from botnets worldwide) bị loại ngay

IP Reputation Lists:
- Database IPs đã tham gia attacks trước đó
- Tor exit nodes, known proxy IPs, hosting provider IPs
- Block hoặc challenge ngay
- Providers: Spamhaus, AbuseIPDB, CloudFlare Threat Intelligence
```

---

## 7. AWS Shield & WAF — Bảo Vệ DDoS Trên AWS

### 7.1 AWS Shield Standard (Miễn Phí)

```
Included FREE với mọi AWS account:
✓ Bảo vệ Layer 3/4 automatic
✓ SYN Flood protection (SYN proxy/cookies)
✓ UDP reflection attack mitigation
✓ Inline mitigation (không cần route traffic elsewhere)
✓ Protect: CloudFront, Route 53, Global Accelerator, ELB, EC2

Không cần cấu hình gì — tự động hoạt động!
```

### 7.2 AWS Shield Advanced ($3,000/tháng)

```
Mọi thứ của Standard PLUS:
✓ Layer 7 (Application) DDoS protection
✓ Near real-time visibility (CloudWatch metrics)
✓ AWS DDoS Response Team (DRT) — chuyên gia AWS hỗ trợ 24/7
✓ Cost protection: AWS hoàn tiền nếu DDoS gây scale-up costs
✓ WAF included (miễn phí WAF rules khi dùng Shield Advanced)
✓ Health-based detection (dùng Route 53 health checks)
✓ Automatic application layer mitigation
✓ Global threat environment dashboard

Giá:
- $3,000/tháng (commitment 1 năm)
- Data transfer: $0.050/GB (cho DDoS mitigation traffic)
- Tất cả resources trong account được bảo vệ
```

### 7.3 AWS WAF (Web Application Firewall)

```
AWS WAF bảo vệ Layer 7:
Deploy trước: CloudFront, ALB, API Gateway, AppSync

WAF Rules ví dụ:
├── Rate-based rule: Block IP if > 2000 requests/5 minutes
├── IP Set rule: Block known bad IPs
├── Geo match rule: Block countries
├── SQL injection rule: Block SQLi patterns
├── XSS rule: Block cross-site scripting
├── Size constraint: Block oversized requests (> 8KB body)
└── Regex match: Block custom patterns

Managed Rule Groups (AWS cung cấp sẵn):
├── AWSManagedRulesCommonRuleSet (Core rules)
├── AWSManagedRulesKnownBadInputsRuleSet
├── AWSManagedRulesSQLiRuleSet
├── AWSManagedRulesAmazonIpReputationList
└── AWSManagedRulesBotControlRuleSet

Third-party rules: Fortinet, F5, Imperva (từ AWS Marketplace)
```

### 7.4 Kiến Trúc Phòng Thủ Đa Tầng AWS

```
┌──────────────────────────────────────────────────┐
│                    INTERNET                        │
└───────────────────────┬──────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────┐
│  Layer 1: Route 53 (DNS)                          │
│  - Anycast → phân tán traffic toàn cầu           │
│  - Health checks → route away from unhealthy      │
│  - Shield Standard → auto L3/4 protection         │
└───────────────────────┬──────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────┐
│  Layer 2: CloudFront (CDN)                        │
│  - Edge locations absorb traffic                  │
│  - Shield Standard/Advanced                       │
│  - AWS WAF → L7 filtering                        │
│  - Bot Control → challenge suspicious bots        │
│  - Rate limiting per IP                          │
└───────────────────────┬──────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────┐
│  Layer 3: ALB + WAF                               │
│  - Additional WAF rules                          │
│  - Security Groups (allow only CloudFront IPs)    │
│  - Connection limits                             │
└───────────────────────┬──────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────┐
│  Layer 4: Application                             │
│  - Auto Scaling (absorb legitimate load)          │
│  - Application-level rate limiting               │
│  - CAPTCHA for suspicious requests               │
└──────────────────────────────────────────────────┘
```

---

## 8. Phát Hiện DDoS — Detection Techniques

### 8.1 Indicators of Attack (Dấu Hiệu Bị Tấn Công)

```
Network Level:
- Bandwidth usage spike đột ngột (10x bình thường)
- Unusual traffic patterns (1 IP gửi 10,000 req/s)
- High packet rate nhưng low useful data
- Traffic từ unexpected geographies
- Single protocol dominance (99% UDP suddenly)

Application Level:
- Response time tăng đột ngột (50ms → 5,000ms)
- Error rate tăng (5xx responses spike)
- Specific URL paths bị flood
- Login failures spike
- CPU/Memory usage 100%

Infrastructure Level:
- Connection table saturation (netstat shows thousands TIME_WAIT)
- Firewall state table full
- DNS resolver overwhelmed
- Load balancer health checks failing
```

### 8.2 Traffic Analysis Techniques

```
Baseline Comparison:
- Establish "normal" traffic patterns (last 30 days average)
- Alert khi traffic > 3x standard deviation
- Ví dụ: Normal = 1000 req/s, Alert if > 5000 req/s

Flow Analysis (NetFlow/sFlow):
- Analyze traffic flows: src IP, dst IP, port, protocol, bytes
- Identify anomalies: single source sending to many ports (scan)
- Detect amplification: many sources, same dst port

Deep Packet Inspection:
- Examine packet content for attack signatures
- Detect malformed packets (invalid TCP flags)
- Identify application-layer attack patterns

Machine Learning:
- Train models on normal traffic patterns
- Auto-detect anomalies without predefined rules
- Reduce false positives over time
```

### 8.3 AWS Detection Tools

```
AWS Shield Advanced Detection:
- Automatic baseline learning per resource
- Real-time traffic anomaly detection
- Combination of flow data + application metrics
- Integration with Route 53 health checks for faster detection

CloudWatch Alarms:
- DDoSDetected metric (Shield Advanced)
- RequestCount spike on ALB/CloudFront
- TargetResponseTime increase
- UnHealthyHostCount increase
- Custom metrics from application

VPC Flow Logs:
- Capture all traffic metadata
- Analyze with Athena for patterns
- Detect scanning, flooding patterns
```

---

## 9. Playbook Ứng Phó DDoS

### 9.1 Preparation (Trước Khi Bị Tấn Công)

```
✅ Architecture:
  - Multi-AZ, multi-region deployment
  - Auto Scaling configured with generous max
  - CloudFront in front of everything (absorb at edge)
  - Static content on S3 (infinite scale)
  - WAF rules deployed and tested

✅ Monitoring:
  - CloudWatch alarms cho bandwidth, requests, errors
  - Baseline metrics documented
  - PagerDuty/OpsGenie integration
  - Runbook documented và tested

✅ DNS:
  - Low TTL (300s) → nhanh chóng chuyển traffic nếu cần
  - Backup DNS provider configured
  - Route 53 health checks active

✅ Relationships:
  - AWS Shield Advanced activated (DRT access)
  - ISP emergency contact ready
  - CDN provider support plan
```

### 9.2 During Attack (Đang Bị Tấn Công)

```
Bước 1: CONFIRM (Xác nhận)
  - Kiểm tra: thật sự DDoS hay chỉ traffic spike hợp pháp?
  - Check: CloudWatch metrics, access logs, error patterns
  - Đừng panic nếu chỉ là viral marketing traffic!

Bước 2: CLASSIFY (Phân loại)
  - Volumetric? → Check bandwidth utilization
  - Protocol? → Check connection table, SYN rate
  - Application? → Check request patterns, error rate
  → Phân loại giúp chọn đúng mitigation

Bước 3: MITIGATE (Giảm thiểu)
  Layer 3/4:
    - AWS Shield auto-mitigates (nếu enabled)
    - Liên hệ ISP nếu cần upstream filtering
    - Enable Anycast routing (CloudFront/Route53)
    
  Layer 7:
    - WAF rate limiting rules
    - Block suspicious IPs/countries
    - CAPTCHA cho endpoints bị flood
    - Scale up application servers

Bước 4: COMMUNICATE (Thông báo)
  - Status page update
  - Internal team notification
  - Customer communication nếu service degraded
  - Shield Advanced: Engage DRT (AWS DDoS Response Team)

Bước 5: ADAPT (Thích ứng)
  - Attacker thay đổi pattern? → Update WAF rules
  - New attack vector? → Add new rules
  - Tự động hóa response cho lần sau
```

### 9.3 Post-Attack (Sau Tấn Công)

```
✅ Analysis:
  - Review logs: attack duration, peak traffic, vectors used
  - Effectiveness of mitigation: what worked? what didn't?
  - Collateral damage: legitimate traffic blocked?
  
✅ Improvement:
  - Update WAF rules based on attack patterns
  - Adjust rate limits
  - Improve detection thresholds
  - Update runbook

✅ Documentation:
  - Incident report
  - Timeline of events and actions
  - Lessons learned
  - Cost impact (AWS costs during attack)
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 DDoS Mitigation Cheat Sheet

```
┌─────────────────────────────────────────┐
│         DDoS Defense Matrix              │
├────────────┬────────────────────────────┤
│ Attack     │ Primary Defense            │
├────────────┼────────────────────────────┤
│ UDP Flood  │ Scrubbing + Rate limit     │
│ Amplific.  │ Block spoofed + BCP38      │
│ SYN Flood  │ SYN Cookies + SYN Proxy    │
│ HTTP Flood │ WAF + Rate Limit + CAPTCHA │
│ Slowloris  │ Connection timeout + nginx │
│ Rapid Reset│ HTTP/2 stream limits       │
└────────────┴────────────────────────────┘
```

### 10.2 Key Takeaways

1. **DDoS = "đông người giả" chiếm chỗ "người thật"** — mục tiêu là làm service unavailable
2. **3 loại chính:** Volumetric (nghẽn đường), Protocol (khai thác lỗ hổng), Application (tấn công thông minh)
3. **Amplification cực kỳ nguy hiểm** — attacker gửi ít, victim nhận gấp 50-50,000 lần
4. **Layer 7 attacks khó chặn nhất** — trông giống traffic hợp lệ
5. **Defense-in-depth:** Nhiều tầng bảo vệ, không dựa vào 1 biện pháp
6. **AWS Shield Standard miễn phí** — tự động bảo vệ L3/L4 cho mọi account
7. **Chuẩn bị TRƯỚC khi bị tấn công** — khi đang bị tấn công mới setup = quá muộn
8. **Anycast + CDN** = phân tán traffic → khó overwhelm hệ thống phân tán

### 10.3 Tài Liệu Tham Khảo

- RFC 2827 (BCP38): Network Ingress Filtering — chống IP spoofing
- RFC 4987: TCP SYN Flooding Attacks and Common Mitigations
- RFC 5635: Remote Triggered Black Hole Filtering
- AWS Best Practices for DDoS Resiliency (AWS Whitepaper)
- AWS Shield Documentation: https://docs.aws.amazon.com/shield/
- AWS WAF Documentation: https://docs.aws.amazon.com/waf/
- NIST SP 800-189: Resilient Interdomain Traffic Exchange
- Cloudflare DDoS Threat Report (quarterly)
- Akamai State of the Internet Report
- OWASP: Testing for DoS vulnerabilities

---

*Bài viết tiếp theo: Network Troubleshooting — Phương pháp luận xử lý sự cố mạng theo OSI model*
