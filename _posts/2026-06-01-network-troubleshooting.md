---
layout: post
title: "Network Troubleshooting Deep Dive - Phương Pháp Xử Lý Sự Cố Mạng"
date: 2026-06-01
categories: [networking]
tags: [troubleshooting, ping, traceroute, tcpdump, wireshark, nmap]
---

# Network Troubleshooting Deep Dive - Phương Pháp Xử Lý Sự Cố Mạng

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Hãy tưởng tượng bạn bật đèn trong phòng nhưng đèn không sáng. Bạn sẽ kiểm tra thế nào?

1. **Bóng đèn** có bị cháy không? (Thay bóng thử)
2. **Công tắc** có bật không? (Bật lên xuống)
3. **Cầu chì/aptomat** có bị nhảy không? (Kiểm tra tủ điện)
4. **Dây điện** có bị đứt không? (Kiểm tra kết nối)
5. **Nhà có điện** không? (Hỏi hàng xóm)

Bạn vừa áp dụng phương pháp **troubleshooting theo tầng** — kiểm tra từ gần nhất (bóng đèn) đến xa nhất (nguồn điện thành phố). Network troubleshooting cũng y hệt!

**Ví dụ network:** Website không load được. Vấn đề ở đâu?
- Máy tính bạn? (NIC hỏng, driver lỗi)
- Dây mạng? (Đứt, lỏng)
- Router nhà bạn? (Treo, hết bandwidth)
- ISP? (Đường truyền đứt)
- DNS? (Không phân giải được domain)
- Server website? (Down, overloaded)
- Ứng dụng? (Code lỗi, database chết)

Bài viết này sẽ dạy bạn **phương pháp luận có hệ thống** và **công cụ chính xác** để xác định vấn đề nằm ở đâu.

---

## 2. Phương Pháp Luận Troubleshooting Theo OSI

### 2.1 Ba Chiến Lược Chính

**Bottom-Up (Dưới lên trên):**
```
Layer 1 (Physical) → Layer 2 (Data Link) → Layer 3 (Network) → ... → Layer 7 (Application)

Khi nào dùng: 
- Không biết gì về vấn đề
- Nghi ngờ physical/hardware issue
- Thiết bị mới cài đặt/di chuyển
```

**Top-Down (Trên xuống dưới):**
```
Layer 7 (Application) → Layer 6 → ... → Layer 1 (Physical)

Khi nào dùng:
- Biết chắc hardware OK (đã chạy tốt trước đó)
- Application-specific problems
- Lỗi chỉ ảnh hưởng 1 service (web down nhưng SSH OK)
```

**Divide and Conquer (Chia để trị):**
```
Bắt đầu từ Layer 3 (Network) → nếu OK, đi lên → nếu FAIL, đi xuống

Khi nào dùng:
- Có kinh nghiệm, có "sense" vấn đề ở đâu
- Muốn nhanh nhất
- Đa số engineers dùng cách này
```

### 2.2 Quy Trình 7 Bước

```
1. IDENTIFY (Xác định vấn đề)
   - "Website load chậm" → Chậm bao nhiêu? Cho ai? Từ khi nào?
   - Thu thập symptoms cụ thể

2. ESTABLISH THEORY (Đặt giả thuyết)
   - "Có thể DNS chậm" hoặc "Có thể server overloaded"
   - Dựa trên kinh nghiệm + symptoms

3. TEST THEORY (Kiểm chứng giả thuyết)
   - Chạy diagnostic tools
   - Confirm hoặc eliminate giả thuyết

4. ESTABLISH PLAN (Lập kế hoạch sửa)
   - Xác định fix cần thiết
   - Đánh giá impact (có downtime không?)

5. IMPLEMENT (Thực hiện sửa)
   - Apply fix
   - Có rollback plan nếu fail

6. VERIFY (Xác nhận đã sửa xong)
   - Test lại toàn bộ
   - Confirm user affected đã OK

7. DOCUMENT (Ghi chép)
   - Root cause, fix applied, prevention
   - Knowledge base cho lần sau
```

### 2.3 Checklist Theo Từng Layer

```
Layer 1 (Physical):
□ Cáp mạng có cắm chặt không?
□ Đèn LED trên NIC/Switch có sáng không?
□ Cáp có bị gấp/đứt không?
□ Port trên switch có enabled không?

Layer 2 (Data Link):
□ MAC address có đúng không? (ARP table)
□ VLAN cấu hình đúng chưa?
□ Duplex mismatch? (speed/duplex negotiation)
□ Spanning Tree blocking?

Layer 3 (Network):
□ IP address cấu hình đúng?
□ Subnet mask đúng?
□ Default gateway reachable?
□ Routing table correct?
□ Firewall blocking?

Layer 4 (Transport):
□ Port đang listen?
□ Firewall allow port?
□ TCP handshake thành công?
□ Connection timeout?

Layer 7 (Application):
□ Service đang running?
□ Configuration đúng?
□ DNS resolve đúng?
□ Certificate valid?
□ Application logs show gì?
```

---

## 3. Công Cụ Layer 3: ping, traceroute, mtr

### 3.1 ping — Kiểm Tra Kết Nối Cơ Bản

**Ví dụ đời thường:** Gọi điện cho ai đó xem họ có nghe máy không. Nếu nghe → đường truyền OK. Nếu không → đường truyền có vấn đề hoặc người đó tắt máy.

```bash
# Cú pháp cơ bản
ping google.com

# Output:
PING google.com (142.250.66.46): 56 data bytes
64 bytes from 142.250.66.46: icmp_seq=0 ttl=116 time=3.2 ms
64 bytes from 142.250.66.46: icmp_seq=1 ttl=116 time=3.5 ms
64 bytes from 142.250.66.46: icmp_seq=2 ttl=116 time=4.1 ms
64 bytes from 142.250.66.46: icmp_seq=3 ttl=116 time=3.3 ms

--- google.com ping statistics ---
4 packets transmitted, 4 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 3.2/3.5/4.1/0.3 ms
```

**Đọc kết quả ping:**

| Metric | Ý nghĩa | Giá trị tốt |
|--------|---------|-------------|
| time (RTT) | Round-trip time = đi + về | < 50ms (nội địa), < 200ms (quốc tế) |
| ttl | Time to Live = số hop còn lại | 64 (Linux origin), 128 (Windows origin) |
| packet loss | % packet bị mất | 0% (lý tưởng), < 1% (chấp nhận được) |

**Các tình huống khi ping:**
```bash
# Ping thành công → Layer 3 connectivity OK
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=5.2 ms

# Request timeout → Host down HOẶC ICMP bị block
Request timeout for icmp_seq 0

# Destination Host Unreachable → Routing problem
From 192.168.1.1: Destination Host Unreachable

# Packet loss > 0% → Network congestion hoặc interface flapping
10 packets transmitted, 7 received, 30% packet loss

# High latency variation (jitter) → Congestion/QoS issue
time=5ms, time=5ms, time=200ms, time=5ms, time=300ms
```

**Ping nâng cao:**
```bash
# Ping với packet size lớn (test MTU)
ping -s 1472 -M do 8.8.8.8    # -M do = don't fragment
# Nếu fail: MTU path < 1500

# Ping từ specific interface
ping -I eth0 8.8.8.8

# Ping flood (test throughput, CẨN THẬN!)
ping -f -c 1000 192.168.1.1    # Gửi nhanh nhất có thể

# Ping chỉ IPv6
ping6 google.com
```

### 3.2 traceroute — Theo Dõi Đường Đi

**Ví dụ đời thường:** Bạn gửi bưu phẩm từ Hà Nội đến New York. Traceroute cho bạn biết bưu phẩm đi qua những bưu cục nào: Hà Nội → TP.HCM → Singapore → Los Angeles → New York.

```bash
traceroute google.com

# Output:
traceroute to google.com (142.250.66.46), 30 hops max, 60 byte packets
 1  gateway (192.168.1.1)       1.2 ms   1.1 ms   1.0 ms     ← Router nhà
 2  10.0.0.1                    5.3 ms   5.1 ms   5.2 ms     ← ISP local
 3  core-router.isp.vn          8.7 ms   8.5 ms   8.9 ms     ← ISP core
 4  singapore-peer.net         25.3 ms  25.1 ms  25.5 ms     ← Singapore
 5  72.14.232.110              26.2 ms  26.0 ms  26.4 ms     ← Google edge
 6  142.250.66.46               3.2 ms   3.1 ms   3.0 ms     ← Destination
```

**Đọc kết quả traceroute:**
```
Hop 4: * * *              → Router không trả ICMP (firewall block, không phải down)
Hop 5: 50ms 51ms 200ms   → Packet thứ 3 bị delay (jitter)
Hop 6: 50ms 50ms 50ms    → Ổn định

Latency TĂNG đột ngột tại 1 hop:
Hop 3:  10ms → 
Hop 4: 150ms → ← Vấn đề BẮT ĐẦU tại hop 4 (link giữa hop 3 và 4)
Hop 5: 155ms →
Hop 6: 160ms →

* * * liên tục nhiều hop = đường đi bị block/down:
Hop 4: * * *
Hop 5: * * *
Hop 6: * * *
→ Kết luận: vấn đề từ hop 4 trở đi
```

**traceroute sử dụng:**
- Linux: UDP packets (default) hoặc ICMP (`-I`)
- Windows: ICMP Echo (tracert)
- Mỗi hop nhận packet → TTL expired → trả ICMP Time Exceeded → traceroute biết IP của hop đó

### 3.3 mtr — My Traceroute (Kết Hợp ping + traceroute)

**mtr chạy liên tục**, kết hợp cả traceroute VÀ ping statistics cho mỗi hop:

```bash
mtr google.com

# Output (realtime, cập nhật liên tục):
                           Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. gateway (192.168.1.1)   0.0%   100    1.2   1.3   0.9   2.1   0.3
 2. 10.0.0.1                0.0%   100    5.1   5.3   4.8   7.2   0.5
 3. core-router.isp.vn      2.0%   100    8.5   9.1   8.0  15.3   1.8  ← 2% loss!
 4. singapore-peer.net      2.0%   100   25.1  25.8  24.5  30.2   1.2
 5. google-edge             0.0%   100   26.0  26.2  25.8  27.1   0.4
 6. 142.250.66.46           0.0%   100    3.1   3.2   3.0   4.0   0.2
```

**Phân tích mtr:**
```
Hop 3: Loss% = 2%, Avg = 9.1ms, Wrst = 15.3ms, StDev = 1.8
→ Có packet loss VÀ jitter tại hop 3
→ NHƯNG: Nếu hop 4+ KHÔNG có loss → hop 3 chỉ rate-limit ICMP (bình thường!)
→ CHỈ KHI hop 3+ ĐỀU có loss → vấn đề thật sự tại hop 3

Nguyên tắc: Loss bắt đầu tại hop N VÀ tiếp tục ở tất cả hop sau N
→ Vấn đề ở hop N (hoặc link giữa N-1 và N)
```

**mtr nâng cao:**
```bash
# Report mode (chạy 100 packets rồi in kết quả)
mtr -r -c 100 google.com

# TCP mode (bypass ICMP blocking)
mtr -T -P 443 google.com

# Chỉ IPv4
mtr -4 google.com

# JSON output (cho scripting)
mtr -j google.com
```

---

## 4. Công Cụ DNS: dig, nslookup

### 4.1 dig — DNS Investigation Tool

**Ví dụ đời thường:** dig giống như gọi điện cho tổng đài 1080 hỏi số điện thoại. Bạn cho tên → tổng đài trả số.

```bash
# Query A record (IPv4 address)
dig example.com

# Output giải thích:
;; QUESTION SECTION:
;example.com.                   IN      A

;; ANSWER SECTION:
example.com.            3600    IN      A       93.184.216.34
                        ↑TTL(giây)      ↑Type   ↑IP address

;; Query time: 25 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)    ← DNS server đã hỏi
;; WHEN: Mon Jun 01 10:00:00 UTC 2026
```

**Các loại query phổ biến:**
```bash
# A record (IPv4)
dig example.com A

# AAAA record (IPv6)  
dig example.com AAAA

# MX record (Mail server)
dig example.com MX

# NS record (Name servers)
dig example.com NS

# CNAME record (Alias)
dig www.example.com CNAME

# TXT record (SPF, DKIM, verification)
dig example.com TXT

# SOA record (Zone authority)
dig example.com SOA

# ANY (tất cả records) — nhiều DNS servers đã disable
dig example.com ANY
```

**dig nâng cao:**
```bash
# Hỏi DNS server cụ thể
dig @8.8.8.8 example.com       # Hỏi Google DNS
dig @1.1.1.1 example.com       # Hỏi Cloudflare DNS

# Trace toàn bộ delegation chain (từ root → TLD → authoritative)
dig +trace example.com
# Root → .com NS → example.com NS → answer
# Cực hữu ích khi debug DNS propagation!

# Short output (chỉ answer)
dig +short example.com
# 93.184.216.34

# Reverse DNS (IP → domain)
dig -x 8.8.8.8
# 8.8.8.8.in-addr.arpa.  PTR  dns.google.

# Check DNSSEC
dig +dnssec example.com

# Query with no recursion (test authoritative only)
dig +norecurse @ns1.example.com example.com
```

### 4.2 Troubleshooting DNS Issues

```bash
# Vấn đề: website không load, nghi DNS

# Bước 1: Check DNS resolution
dig example.com +short
# Nếu KHÔNG có output → DNS resolution fail

# Bước 2: Thử DNS server khác
dig @8.8.8.8 example.com +short     # Google
dig @1.1.1.1 example.com +short     # Cloudflare
# Nếu public DNS OK, local DNS fail → vấn đề local DNS server

# Bước 3: Trace delegation
dig +trace example.com
# Nếu dừng ở TLD → authoritative NS down
# Nếu NS trả SERVFAIL → zone configuration error

# Bước 4: Check TTL (propagation)
dig example.com | grep "TTL"
# TTL thấp (60-300) = thay đổi propagate nhanh
# TTL cao (86400) = thay đổi mất ~24h để propagate

# Bước 5: Check authoritative directly
dig @ns1.example.com example.com +norecurse
# Nếu authoritative trả sai → zone file wrong
```

---

## 5. Công Cụ Layer 4: nmap, netcat, ss

### 5.1 nmap — Network Mapper

**Ví dụ đời thường:** nmap giống như đi dọc một con phố và kiểm tra từng cánh cửa xem cửa nào mở, cửa nào đóng, cửa nào bị khóa.

```bash
# Scan ports phổ biến (top 1000)
nmap 192.168.1.1

# Output:
PORT     STATE    SERVICE
22/tcp   open     ssh
80/tcp   open     http
443/tcp  open     https
3306/tcp closed   mysql
8080/tcp filtered http-proxy   ← Firewall blocking (not responding)
```

**Port States:**
```
open     = Port accepting connections (service running)
closed   = Port reachable but no service listening (RST response)
filtered = No response (firewall DROP, packet eaten)
```

**Các loại scan:**
```bash
# TCP Connect scan (full handshake, chậm nhưng reliable)
nmap -sT 192.168.1.1

# SYN scan (half-open, nhanh hơn, cần root)
sudo nmap -sS 192.168.1.1

# UDP scan (chậm nhất, phát hiện DNS/DHCP/SNMP)
sudo nmap -sU 192.168.1.1

# Scan specific ports
nmap -p 22,80,443,8080 192.168.1.1

# Scan ALL ports (1-65535)
nmap -p- 192.168.1.1

# Scan port range
nmap -p 1-1024 192.168.1.1

# OS detection
sudo nmap -O 192.168.1.1

# Service version detection
nmap -sV 192.168.1.1
# PORT    STATE SERVICE VERSION
# 22/tcp  open  ssh     OpenSSH 8.9 (Ubuntu)
# 80/tcp  open  http    nginx 1.24.0

# Aggressive scan (OS + version + scripts + traceroute)
nmap -A 192.168.1.1
```

### 5.2 ss/netstat — Socket Statistics

```bash
# Liệt kê tất cả connections
ss -tunapl

# Giải thích flags:
# -t = TCP
# -u = UDP
# -n = Numeric (không resolve DNS)
# -a = All (including LISTEN)
# -p = Process (show process name)
# -l = Listening only

# Output:
State   Recv-Q  Send-Q  Local Address:Port    Peer Address:Port   Process
LISTEN  0       128     0.0.0.0:22            0.0.0.0:*           sshd
LISTEN  0       511     0.0.0.0:80            0.0.0.0:*           nginx
ESTAB   0       0       10.0.1.5:22           203.0.113.50:54321  sshd
TIME-WAIT 0     0       10.0.1.5:80           198.51.100.1:12345

# Chỉ listening ports
ss -tlnp

# Đếm connections theo state
ss -s
# Total: 1543
# TCP:   850 (estab 320, closed 200, timewait 300)

# Filter theo port
ss -tn sport = :443

# Filter theo state
ss state established
ss state time-wait
```

### 5.3 Troubleshooting Kết Nối Layer 4

```bash
# Vấn đề: App không kết nối được đến database port 5432

# Bước 1: Database có listen không?
ss -tlnp | grep 5432
# LISTEN 0.0.0.0:5432 → OK, đang listen trên tất cả interfaces
# LISTEN 127.0.0.1:5432 → Chỉ listen localhost! App từ máy khác không kết nối được!

# Bước 2: Firewall có block không?
sudo iptables -L -n | grep 5432
# Hoặc:
sudo nft list ruleset | grep 5432

# Bước 3: Từ client, test kết nối
nc -zv db-server 5432
# Connection to db-server 5432 port [tcp/postgresql] succeeded!  → OK
# nc: connect to db-server port 5432 (tcp) failed: Connection refused  → Service down
# nc: connect to db-server port 5432 (tcp) failed: Connection timed out → Firewall block

# Bước 4: Từ client, nmap check
nmap -p 5432 db-server
# open → OK
# closed → Service not running
# filtered → Firewall blocking
```

---

## 6. Packet Capture: tcpdump & Wireshark

### 6.1 tcpdump — Bắt Gói Tin Trên Terminal

**Ví dụ đời thường:** tcpdump giống như đặt camera an ninh trên đường dây mạng — ghi lại MỌI THỨ đi qua.

```bash
# Bắt tất cả traffic trên eth0
sudo tcpdump -i eth0

# Bắt traffic đến/từ IP cụ thể
sudo tcpdump host 192.168.1.100

# Bắt traffic đến port 80
sudo tcpdump port 80

# Bắt traffic TCP đến port 443
sudo tcpdump tcp port 443

# Bắt traffic từ subnet
sudo tcpdump net 192.168.1.0/24

# Kết hợp filters
sudo tcpdump -i eth0 src host 10.0.0.1 and dst port 22

# Lưu vào file (mở bằng Wireshark sau)
sudo tcpdump -i eth0 -w capture.pcap

# Đọc từ file
tcpdump -r capture.pcap

# Hiện content (ASCII)
sudo tcpdump -A port 80

# Hiện content (Hex + ASCII)
sudo tcpdump -X port 80

# Giới hạn số packets
sudo tcpdump -c 100 port 80

# Verbose output
sudo tcpdump -vvv port 80
```

**Đọc output tcpdump:**
```
10:15:32.123456 IP 192.168.1.5.54321 > 93.184.216.34.80: Flags [S], seq 12345, win 65535
                ↑ timestamp    ↑ source:port   ↑ dest:port        ↑SYN    ↑sequence  ↑window

Flags:
[S]     = SYN (bắt đầu kết nối)
[S.]    = SYN-ACK (chấp nhận kết nối)
[.]     = ACK (xác nhận)
[P.]    = PUSH-ACK (gửi data)
[F.]    = FIN-ACK (đóng kết nối)
[R]     = RST (reset/refuse)
```

### 6.2 Wireshark — GUI Packet Analyzer

**Wireshark** là version GUI mạnh mẽ hơn tcpdump, với khả năng phân tích protocol sâu.

**Display Filters phổ biến:**
```
# Filter theo IP
ip.addr == 192.168.1.100
ip.src == 10.0.0.1
ip.dst == 8.8.8.8

# Filter theo protocol
http
dns
tcp
tls

# Filter theo port
tcp.port == 443
tcp.dstport == 80

# Filter theo HTTP method
http.request.method == "POST"
http.request.uri contains "/api"

# Filter theo TCP flags
tcp.flags.syn == 1 and tcp.flags.ack == 0    # SYN only (new connections)
tcp.flags.rst == 1                            # RST (connection problems)

# Filter TCP retransmissions (DẤU HIỆU MẠNG CÓ VẤN ĐỀ)
tcp.analysis.retransmission

# Filter theo response code
http.response.code == 500

# DNS queries
dns.qry.name == "example.com"

# TLS handshake issues
tls.handshake.type == 1    # Client Hello
tls.alert_message          # TLS alerts (cert errors etc)
```

### 6.3 Phân Tích Packet Capture — Các Pattern Lỗi

```
Pattern 1: TCP Retransmission (Gửi lại)
→ Packet bị mất, sender phải gửi lại
→ Dấu hiệu: network congestion, packet loss
→ Fix: kiểm tra link quality, switch errors, cable

Pattern 2: TCP Window Full / Zero Window
→ Receiver buffer đầy, yêu cầu sender dừng
→ Dấu hiệu: receiver quá chậm xử lý
→ Fix: tăng buffer size, optimize receiver application

Pattern 3: RST after SYN (Connection Refused)
→ Port không listen, service down
→ Fix: start service, check port binding

Pattern 4: No Response to SYN (Timeout)
→ Firewall DROP, host unreachable
→ Fix: check firewall rules, routing

Pattern 5: TLS Alert (Handshake Failure)
→ Certificate mismatch, cipher mismatch, expired cert
→ Fix: renew cert, update cipher suites, check SNI

Pattern 6: DNS NXDOMAIN
→ Domain không tồn tại
→ Fix: check domain spelling, DNS records

Pattern 7: HTTP 502/504 from Load Balancer
→ Backend server down hoặc timeout
→ Fix: check backend health, increase timeout
```

---

## 7. curl — Swiss Army Knife Cho HTTP

### 7.1 Cơ Bản

```bash
# GET request đơn giản
curl https://example.com

# Chỉ headers (HEAD request)
curl -I https://example.com
# HTTP/2 200
# content-type: text/html
# cache-control: max-age=3600

# Verbose mode (xem toàn bộ quá trình)
curl -v https://example.com
# * Trying 93.184.216.34:443...
# * Connected to example.com (93.184.216.34) port 443
# * TLS handshake...
# * SSL certificate: subject=CN=example.com
# > GET / HTTP/2
# > Host: example.com
# < HTTP/2 200
# < content-type: text/html
```

### 7.2 Đo Thời Gian Từng Giai Đoạn

```bash
# Timing breakdown (CỰC HỮU ÍCH cho troubleshooting)
curl -w "\
    DNS Lookup:    %{time_namelookup}s\n\
    TCP Connect:   %{time_connect}s\n\
    TLS Handshake: %{time_appconnect}s\n\
    First Byte:    %{time_starttransfer}s\n\
    Total Time:    %{time_total}s\n\
    Download Size: %{size_download} bytes\n\
    HTTP Code:     %{http_code}\n" \
    -o /dev/null -s https://example.com

# Output:
    DNS Lookup:    0.025s       ← Nếu cao: DNS server chậm
    TCP Connect:   0.050s       ← Nếu cao: network latency
    TLS Handshake: 0.120s       ← Nếu cao: TLS negotiation chậm
    First Byte:    0.200s       ← Nếu cao: server processing chậm
    Total Time:    0.350s       ← Tổng thời gian
    Download Size: 15234 bytes
    HTTP Code:     200
```

**Phân tích timing:**
```
DNS cao (>100ms)     → DNS server xa hoặc chậm → thử đổi DNS
TCP cao (>100ms)     → Server xa hoặc network congested → check traceroute
TLS cao (>200ms)     → Certificate chain dài, OCSP slow → enable OCSP stapling
First Byte cao       → Server processing time lớn → optimize backend
Total - First Byte   → Download time → file lớn hoặc bandwidth thấp
```

### 7.3 curl Cho Troubleshooting

```bash
# Resolve DNS thủ công (bypass DNS)
curl --resolve example.com:443:93.184.216.34 https://example.com
# Hữu ích khi DNS chưa propagate nhưng muốn test server mới

# Follow redirects
curl -L https://example.com

# Custom headers
curl -H "Host: example.com" -H "X-Debug: true" https://lb-ip/

# POST data
curl -X POST -d '{"key":"value"}' -H "Content-Type: application/json" https://api.example.com

# Test specific TLS version
curl --tlsv1.2 --tls-max 1.2 https://example.com

# Ignore certificate errors (debugging only!)
curl -k https://self-signed.example.com

# Show only response code
curl -s -o /dev/null -w "%{http_code}" https://example.com
```

---

## 8. AWS Network Troubleshooting Tools

### 8.1 VPC Flow Logs

```
VPC Flow Logs ghi lại metadata của MỌI traffic đi qua VPC:

Ví dụ log entry:
2 123456789012 eni-abc123 10.0.1.5 203.0.113.50 443 54321 6 25 5000 1622505600 1622505660 ACCEPT OK

Giải thích:
- Account: 123456789012
- ENI: eni-abc123 (Elastic Network Interface)
- Src IP: 10.0.1.5 (instance)
- Dst IP: 203.0.113.50 (client)
- Src Port: 443
- Dst Port: 54321
- Protocol: 6 (TCP)
- Packets: 25
- Bytes: 5000
- Start/End time
- Action: ACCEPT (allowed by Security Group/NACL)
- Status: OK

Action values:
ACCEPT = Traffic allowed
REJECT = Traffic blocked by SG or NACL

Use cases:
- "Tại sao instance không kết nối được?" → Tìm REJECT entries
- "Traffic đến từ đâu?" → Analyze source IPs
- "Port nào bị block?" → Filter theo port + REJECT
```

**Query Flow Logs với Athena:**
```sql
-- Top 10 IPs bị reject
SELECT srcaddr, COUNT(*) as reject_count
FROM vpc_flow_logs
WHERE action = 'REJECT'
GROUP BY srcaddr
ORDER BY reject_count DESC
LIMIT 10;

-- Traffic đến port 22 (SSH)
SELECT srcaddr, dstaddr, packets, bytes
FROM vpc_flow_logs  
WHERE dstport = 22 AND action = 'ACCEPT'
ORDER BY bytes DESC;
```

### 8.2 VPC Reachability Analyzer

```
AWS cung cấp Reachability Analyzer — tool KHÔNG cần gửi traffic thật:
- Phân tích configuration (Security Groups, NACLs, Route Tables)
- Xác định path giữa 2 resources
- Cho biết chính xác ĐÂU bị block

Ví dụ: EC2-A không kết nối được EC2-B port 5432:
Reachability Analyzer output:
  ✓ EC2-A Security Group: Outbound rule allows port 5432
  ✓ Subnet-A NACL: Outbound allows
  ✓ Route Table: Route to Subnet-B exists via local
  ✓ Subnet-B NACL: Inbound allows port 5432
  ✗ EC2-B Security Group: NO inbound rule for port 5432  ← ROOT CAUSE!

→ Fix: Add inbound rule port 5432 to EC2-B Security Group
```

### 8.3 Troubleshooting Cheat Sheet AWS

```
EC2 không kết nối internet:
1. Check Security Group outbound rules
2. Check NACL outbound + inbound (stateless!)
3. Check Route Table: có route 0.0.0.0/0 → IGW không?
4. Check IGW attached to VPC?
5. Public subnet: EC2 có Public IP/Elastic IP?
6. Private subnet: NAT Gateway tồn tại và route đúng?

EC2-to-EC2 không kết nối:
1. Reachability Analyzer (nhanh nhất)
2. Check Security Groups BOTH sides (ingress + egress)
3. Check NACLs BOTH subnets
4. Check routing (đặc biệt cross-VPC qua peering/TGW)
5. Check DNS resolution nếu dùng hostname

ALB trả 502/504:
1. Target Group health check status
2. Target instance Security Group allows ALB
3. Application running on correct port?
4. Application response time > ALB timeout?
5. Check target instance logs
```

---

## 9. Quy Trình Troubleshooting End-to-End

### 9.1 Scenario: "Website Không Load Được"

```
User báo: "Tôi không vào được website.com"

Bước 1: Xác định scope
- Chỉ 1 user hay tất cả users?
- Chỉ 1 trang hay toàn bộ website?
- Từ khi nào? Có thay đổi gì gần đây?

Bước 2: DNS Check
$ dig website.com +short
# Nếu trống → DNS problem
# Nếu có IP → DNS OK, đi tiếp

Bước 3: Network Connectivity
$ ping <IP từ step 2>
# Timeout → Network hoặc host down
# OK → Network OK, đi tiếp

Bước 4: Port Check
$ nc -zv <IP> 443
# Connection refused → Service down
# Timeout → Firewall blocking
# OK → Port open, đi tiếp

Bước 5: HTTP Check
$ curl -v https://website.com
# TLS error → Certificate issue
# HTTP 5xx → Application error
# HTTP 200 → Website OK (vấn đề ở client side?)
# Timeout → Server quá chậm

Bước 6: Detailed Timing
$ curl -w "DNS: %{time_namelookup}\nConnect: %{time_connect}\nTLS: %{time_appconnect}\nFirst byte: %{time_starttransfer}\n" -o /dev/null -s https://website.com
# Xác định bottleneck cụ thể

Bước 7: Path Analysis (nếu cần)
$ mtr website.com
# Xác định hop nào có packet loss
```

### 9.2 Quick Reference: Tool Cho Từng Vấn Đề

| Vấn đề | Tool đầu tiên | Tool chi tiết |
|--------|---------------|---------------|
| Không ping được | ping, traceroute | mtr, tcpdump |
| DNS không resolve | dig, nslookup | dig +trace |
| Port bị block | nmap, nc | iptables -L, VPC Flow Logs |
| Kết nối chậm | curl -w timing | mtr, Wireshark |
| Packet loss | mtr | tcpdump + Wireshark |
| SSL/TLS lỗi | curl -v, openssl s_client | Wireshark TLS filter |
| Application error | curl -v | access logs, app logs |
| AWS connectivity | Reachability Analyzer | VPC Flow Logs |

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 Troubleshooting Mindset

```
1. KHÔNG ĐOÁN MÒ — Dùng data (logs, metrics, packet captures)
2. THAY ĐỔI 1 THỨ MỖI LẦN — Nếu đổi 3 thứ cùng lúc, không biết cái nào fix
3. DOCUMENT MỌI THỨ — Lần sau gặp lại sẽ biết ngay
4. ROLLBACK NẾU KHÔNG CHẮC — Có thể revert nếu fix sai
5. HỎI: "CÓ GÌ THAY ĐỔI GẦN ĐÂY?" — 80% issues do recent changes
```

### 10.2 Key Takeaways

1. **Phương pháp luận quan trọng hơn tool** — Biết dùng tool mà không có phương pháp = random guessing
2. **Bottom-up cho lỗi mới, divide-and-conquer cho lỗi quen**
3. **ping kiểm tra Layer 3**, **nmap kiểm tra Layer 4**, **curl kiểm tra Layer 7**
4. **mtr > traceroute** — mtr cho thấy packet loss theo thời gian
5. **tcpdump + Wireshark** là "truth of ground" — packet không nói dối
6. **curl -w timing** chỉ ra chính xác bottleneck ở giai đoạn nào
7. **VPC Flow Logs** = camera an ninh cho AWS network
8. **Reachability Analyzer** = X-ray cho AWS connectivity

### 10.3 Tài Liệu Tham Khảo

- RFC 792: ICMP Protocol (ping sử dụng) 
- RFC 1122: Requirements for Internet Hosts (TCP behavior)
- RFC 8335: PROBE — Host Identity Protocol (modern diagnostics)
- Man pages: ping(8), traceroute(8), tcpdump(8), nmap(1), dig(1), curl(1), ss(8)
- AWS Documentation: VPC Flow Logs, Reachability Analyzer, Network Firewall
- Wireshark User's Guide: https://www.wireshark.org/docs/
- "TCP/IP Illustrated" by W. Richard Stevens — sách kinh điển về network
- CompTIA Network+ Study Guide — troubleshooting methodology

---

*Bài viết tiếp theo: QoS (Quality of Service) — Cách ưu tiên traffic quan trọng trên mạng*
