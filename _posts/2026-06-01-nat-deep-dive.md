---
layout: post
title: "NAT Deep Dive — Cánh cổng giữa mạng nội bộ và Internet"
subtitle: "Hiểu sâu Static NAT, Dynamic NAT, PAT/NAPT và Connection Tracking — từ RFC 3022 đến AWS NAT Gateway"
tags: [networking, nat, layer3, security, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
comments: true
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 3022 | Traditional IP Network Address Translator (Traditional NAT) |
| RFC 2663 | IP Network Address Translator (NAT) Terminology and Considerations |
| RFC 4787 | NAT Behavioral Requirements for Unicast UDP |
| RFC 5382 | NAT Behavioral Requirements for TCP |
| RFC 6888 | Common Requirements for Carrier-Grade NATs (CGN) |
| Stevens, W.R. — TCP/IP Illustrated, Vol. 1 | Chapter 30: NAT |
| Cisco Documentation | NAT Configuration Guide |
| AWS Documentation | NAT Gateway, NAT Instances |

---

## 1. Giới thiệu — Tại sao cần biết NAT?

### Ví dụ đời thường: Tổng đài điện thoại công ty

Một công ty 500 nhân viên nhưng chỉ có **10 đường dây điện thoại ngoài**. Tổng đài hoạt động:

1. Nhân viên A (ext. 101) gọi ra ngoài → Tổng đài chọn đường dây 1 → Đổi số nội bộ 101 thành số công ty (028) 1234-5678
2. Người ngoài gọi lại số 028-1234-5678 → Tổng đài nhìn: "Cuộc gọi này của ai? À, đường 1 đang dùng cho ext. 101!" → Chuyển về 101
3. **Người bên ngoài KHÔNG bao giờ biết số nội bộ** (101) — chỉ thấy số công ty!

**NAT hoạt động y hệt tổng đài:**
- 500 máy tính nội bộ (IP private: 192.168.x.x) dùng **1 IP công cộng** để ra Internet
- NAT device (router) thay đổi IP nội bộ → IP công cộng khi đi ra
- Khi response về → NAT nhìn bảng tracking → forward về đúng máy

### Concrete scenario: Tại sao 192.168.1.100 không phải IP "thật"?

```
Bạn ở nhà: laptop IP = 192.168.1.100
Bạn ở quán café: laptop IP = 192.168.1.100

2 người KHÁC NHAU, ở nơi KHÁC NHAU, có CÙNG IP! Không conflict?
→ Vì đó là PRIVATE IPs — chỉ có ý nghĩa trong mạng nội bộ
→ NAT router đổi thành PUBLIC IP (khác nhau) trước khi đi ra Internet

Ví dụ:
Nhà bạn:       192.168.1.100 → NAT → 113.161.72.5 (ISP VN cấp)
Quán café:     192.168.1.100 → NAT → 27.72.100.88 (ISP khác cấp)
Google thấy:   Request từ 113.161.72.5 và 27.72.100.88 — hai người khác nhau!
```

### Vấn đề NAT giải quyết

| Vấn đề | Giải pháp NAT |
|---------|---------------|
| IPv4 hết địa chỉ (4.3 tỷ) | Nhiều devices share 1 public IP |
| Mạng nội bộ cần isolate | Private IPs ẩn, bên ngoài không access trực tiếp |
| Server cần accessible từ Internet | Static NAT / Port Forwarding |
| ISP đổi IP thường xuyên | Internal addresses KHÔNG đổi khi ISP thay IP |

---

## 2. NAT là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**NAT** (Network Address Translation) là kỹ thuật **thay đổi IP address** trong header của packet khi nó đi qua router. Mục đích chính:

- **Tiết kiệm IP public**: 1000 máy dùng chung 1 IP public
- **Bảo mật**: Máy bên ngoài không biết cấu trúc mạng bên trong
- **Linh hoạt**: Thay đổi cấu trúc mạng bên trong mà không ảnh hưởng bên ngoài

### Analogy: Hộp thư chung của chung cư

```
╔══════════════════════════════════════════════════════════════════╗
║              HỘP THƯ CHUNG CƯ (= NAT)                          ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Chung cư có 200 căn hộ (= 200 devices với private IP)         ║
║  Nhưng chỉ có 1 địa chỉ: "123 Nguyễn Du" (= 1 public IP)     ║
║                                                                  ║
║  Gửi thư ra ngoài:                                              ║
║    Căn hộ 305 gửi thư → Bảo vệ đổi "305/123 Nguyễn Du"        ║
║    thành "123 Nguyễn Du" → Người ngoài chỉ thấy "123 ND"      ║
║                                                                  ║
║  Nhận thư từ ngoài:                                              ║
║    Thư gửi đến "123 Nguyễn Du" → Bảo vệ xem sổ:               ║
║    "Thư này trả lời cho thư từ 305" → Chuyển lên căn 305      ║
║                                                                  ║
║  Bảo vệ = NAT router                                            ║
║  Sổ ghi chép = NAT/Connection Tracking Table                    ║
║  Số căn hộ = Private IP + Port                                  ║
║  Địa chỉ chung cư = Public IP                                   ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Private IP Ranges (RFC 1918)

```
┌───────────────────────────────────────────────────────────────┐
│           PRIVATE IP RANGES (không route trên Internet)        │
├───────────────────────────────────────────────────────────────┤
│  10.0.0.0/8       = 10.0.0.0 — 10.255.255.255   (16.7M IPs) │
│  172.16.0.0/12    = 172.16.0.0 — 172.31.255.255 (1M IPs)    │
│  192.168.0.0/16   = 192.168.0.0 — 192.168.255.255 (65K IPs) │
├───────────────────────────────────────────────────────────────┤
│  Tất cả đều cần NAT để ra Internet!                           │
│  ISP sẽ DROP bất kỳ packet nào có source = private IP         │
└───────────────────────────────────────────────────────────────┘
```

---

## 3. Các loại NAT — Static, Dynamic, PAT

### Mini example: 3 loại "phiên dịch"

- **Static NAT** = Phiên dịch viên riêng (1:1): Ông A luôn có phiên dịch viên X
- **Dynamic NAT** = Pool phiên dịch (N:M): 10 phiên dịch, ai rảnh thì dịch cho ai
- **PAT/NAPT** = 1 phiên dịch cho cả đoàn (N:1): 1 người dịch, phân biệt bằng "đang nói cho ai?"

### Static NAT (1:1 Mapping)

```
1 Private IP ↔ 1 Public IP (luôn cố định)

Dùng khi: Server nội bộ cần accessible từ Internet
Ví dụ: Web server 192.168.1.10 ↔ Public 203.0.113.10

Inside:  192.168.1.10:80    →  NAT  →  Outside: 203.0.113.10:80
Internet user truy cập 203.0.113.10 → NAT → forward đến 192.168.1.10

NAT Table (cố định):
┌──────────────────┬──────────────────┐
│   Inside Local   │  Inside Global   │
├──────────────────┼──────────────────┤
│ 192.168.1.10     │ 203.0.113.10     │  ← Luôn cố định
│ 192.168.1.11     │ 203.0.113.11     │
│ 192.168.1.12     │ 203.0.113.12     │
└──────────────────┴──────────────────┘

Ưu: Server luôn reachable, simple
Nhược: Tốn 1 public IP cho 1 device — KHÔNG tiết kiệm
```

### Dynamic NAT (N:M Mapping)

```
Pool N private IPs ↔ Pool M public IPs (M < N, first-come-first-served)

Dùng khi: Có nhiều public IPs nhưng ít hơn private IPs

NAT Pool: 203.0.113.20 — 203.0.113.25 (6 public IPs)
Internal: 192.168.1.0/24 (254 hosts)

Host 192.168.1.100 kết nối Internet → được cấp 203.0.113.20
Host 192.168.1.101 kết nối → được cấp 203.0.113.21
...
Host 192.168.1.106 kết nối → Pool hết! KHÔNG RA ĐƯỢC!

Ưu: Không cần cấu hình từng host
Nhược: Vẫn tốn nhiều public IPs, có thể hết pool
```

### PAT / NAPT / NAT Overload (N:1 Mapping) — Phổ biến nhất!

```
NHIỀU private IPs ↔ 1 public IP (phân biệt bằng PORT)

Đây là loại NAT mà 99% router nhà và công ty dùng!

Host A: 192.168.1.100:52341 → NAT → 203.0.113.1:10001
Host B: 192.168.1.101:52342 → NAT → 203.0.113.1:10002
Host C: 192.168.1.102:80    → NAT → 203.0.113.1:10003
Host A: 192.168.1.100:52399 → NAT → 203.0.113.1:10004 (connection khác!)

NAT Table (Connection Tracking):
┌────────────────────────────┬───────────────────────────┬──────────┐
│ Inside (Private)           │ Outside (Public)          │ Protocol │
├────────────────────────────┼───────────────────────────┼──────────┤
│ 192.168.1.100:52341        │ 203.0.113.1:10001         │ TCP      │
│ 192.168.1.101:52342        │ 203.0.113.1:10002         │ TCP      │
│ 192.168.1.102:80           │ 203.0.113.1:10003         │ TCP      │
│ 192.168.1.100:52399        │ 203.0.113.1:10004         │ TCP      │
└────────────────────────────┴───────────────────────────┴──────────┘

1 Public IP hỗ trợ ~65,000 connections đồng thời
(vì port range = 1024-65535 ≈ 64,000 ports)

Ưu: CỰC KỲ tiết kiệm IP — cả công ty dùng 1 IP
Nhược: Phức tạp, stateful, application protocol issues
```

### So sánh 3 loại NAT

| Đặc điểm | Static NAT | Dynamic NAT | PAT/NAPT |
|-----------|-----------|-------------|----------|
| Mapping | 1:1 | N:M | N:1 |
| IP tiết kiệm | Không | Ít | CỰC KỲ |
| Server accessible | Có | Tạm thời | Cần port forwarding |
| Complexity | Thấp | Trung bình | Cao |
| Dùng ở đâu | DMZ servers | Enterprise | Home, SMB, ISP |
| AWS equivalent | Elastic IP | — | NAT Gateway |

---

## 4. Connection Tracking — Bộ nhớ của NAT

### Mini example: Bàn tiếp nhận bệnh viện

Bệnh viện lớn có nhiều bệnh nhân ra vào. Bàn tiếp nhận ghi sổ:
- "Ông A, phòng 305, bác sĩ Minh, vào lúc 9:00, hẹn 11:00"
- Khi bác sĩ gọi "bệnh nhân phòng 305" → biết đó là ông A

**Connection Tracking (conntrack)** trong NAT là "cuốn sổ" tương tự:

### Conntrack Entry Format

```
Mỗi connection đi qua NAT được ghi lại:

TCP: SRC=192.168.1.100 DST=142.250.190.46 SPT=52341 DPT=443
     REPLY SRC=142.250.190.46 DST=203.0.113.1 SPT=443 DPT=10001
     [ASSURED] mark=0 use=1

Giải thích:
- SRC/DST: Source/Destination IP của direction đi ra
- SPT/DPT: Source/Destination Port
- REPLY: Direction ngược lại (response)
- ASSURED: Connection established (đã thấy traffic cả 2 chiều)
```

### Conntrack States

```
╔══════════════════════════════════════════════════════════════════╗
║              CONNECTION TRACKING STATES                           ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  NEW: Packet đầu tiên của connection mới                        ║
║    → NAT tạo entry, gán port                                    ║
║                                                                  ║
║  ESTABLISHED: Đã thấy traffic cả 2 chiều                       ║
║    → NAT biết chắc connection active                            ║
║                                                                  ║
║  RELATED: Packet liên quan đến connection đã có                 ║
║    → Ví dụ: ICMP error cho TCP connection                       ║
║    → FTP data connection (liên quan đến FTP control)            ║
║                                                                  ║
║  INVALID: Packet không match state nào                          ║
║    → Thường bị DROP                                             ║
║                                                                  ║
║  Timeouts:                                                       ║
║    TCP ESTABLISHED: 5 days (432000s)                            ║
║    TCP SYN_SENT: 120s                                           ║
║    TCP FIN_WAIT: 120s                                           ║
║    UDP: 30s (default), 180s (stream)                            ║
║    ICMP: 30s                                                     ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### NAT Table Exhaustion

```
Vấn đề: NAT table đầy!

Giới hạn:
- Linux conntrack: mặc định ~65,000 entries (tunable)
- Home router: 10,000 — 50,000 entries
- Enterprise router: 500,000+
- AWS NAT Gateway: 55,000 concurrent connections per destination

Triệu chứng khi table đầy:
- Connection mới bị DROP (timeout)
- "Cannot open new connection"
- Packet loss tăng đột biến

Monitoring:
$ cat /proc/sys/net/netfilter/nf_conntrack_count   # current
$ cat /proc/sys/net/netfilter/nf_conntrack_max     # limit

# Nâng limit:
$ sudo sysctl -w net.netfilter.nf_conntrack_max=131072
```

### Trong thực tế

```bash
# Xem conntrack table trên Linux
$ sudo conntrack -L
tcp  6 431999 ESTABLISHED src=192.168.1.100 dst=142.250.190.46 
  sport=52341 dport=443 src=142.250.190.46 dst=203.0.113.1 
  sport=443 dport=10001 [ASSURED] mark=0 use=1

# Đếm connections
$ sudo conntrack -C
8451

# Xem theo protocol
$ sudo conntrack -L -p tcp | wc -l
$ sudo conntrack -L -p udp | wc -l

# NAT rules (iptables)
$ sudo iptables -t nat -L -n -v
```

### Trong AWS

- **NAT Gateway**: Managed NAT — AWS quản lý conntrack table
- **Limit**: 55,000 connections đồng thời per destination IP
  - Nếu hơn → connections dropped
  - Giải pháp: Multiple NAT Gateways, hoặc multiple Elastic IPs
- **Metrics**: CloudWatch → NAT Gateway → ActiveConnectionCount, BytesOutToDestination

---

## 5. Port Forwarding — Mở cửa cho server nội bộ

### Mini example: Tổng đài chuyển cuộc gọi

Khi khách gọi số công ty và nhấn:
- Phím 1 → Chuyển phòng kinh doanh (ext. 101)
- Phím 2 → Chuyển phòng kỹ thuật (ext. 201)
- Phím 3 → Chuyển tổng đài (ext. 300)

**Port Forwarding = "nhấn phím"** — traffic đến public IP trên port cụ thể → forward đến server nội bộ.

### Cấu hình Port Forwarding

```
Scenario: 
- Public IP: 203.0.113.1
- Web server nội bộ: 192.168.1.10:80
- SSH server nội bộ: 192.168.1.20:22
- Camera nội bộ: 192.168.1.30:554

Port Forwarding rules:
  203.0.113.1:80  → 192.168.1.10:80   (HTTP)
  203.0.113.1:443 → 192.168.1.10:443  (HTTPS)
  203.0.113.1:2222→ 192.168.1.20:22   (SSH — đổi port cho security)
  203.0.113.1:8554→ 192.168.1.30:554  (RTSP camera)
```

### Linux iptables NAT

```bash
# Enable IP forwarding
$ sudo sysctl -w net.ipv4.ip_forward=1

# SNAT (Source NAT) — internal → internet
$ sudo iptables -t nat -A POSTROUTING -o eth0 \
  -s 192.168.1.0/24 -j MASQUERADE
# MASQUERADE = SNAT tự động dùng IP của interface eth0

# DNAT (Destination NAT) — port forwarding
$ sudo iptables -t nat -A PREROUTING -i eth0 \
  -p tcp --dport 80 -j DNAT --to-destination 192.168.1.10:80

$ sudo iptables -t nat -A PREROUTING -i eth0 \
  -p tcp --dport 2222 -j DNAT --to-destination 192.168.1.20:22

# Allow forwarded traffic
$ sudo iptables -A FORWARD -i eth0 -o eth1 \
  -p tcp --dport 80 -j ACCEPT
```

### SNAT vs DNAT vs MASQUERADE

| | SNAT | DNAT | MASQUERADE |
|--|------|------|------------|
| Thay đổi gì? | Source IP | Destination IP | Source IP (auto) |
| Khi nào? | POSTROUTING (đi ra) | PREROUTING (đi vào) | POSTROUTING |
| Dùng cho | Outbound traffic | Port forwarding/LB | Dynamic IP (ISP) |
| Ví dụ | Internal→Internet | Internet→Internal server | Home router |

### Trong AWS

```
AWS KHÔNG dùng port forwarding truyền thống.
Thay vào đó:

1. Public Subnet + Elastic IP:
   - EC2 trong public subnet có public IP
   - Security Group mở port cần thiết
   - KHÔNG cần NAT/port forward!

2. Private Subnet + NAT Gateway (outbound only):
   - EC2 trong private subnet → NAT GW → Internet
   - Internet KHÔNG thể initiate connection vào (one-way)
   
3. Private Subnet + Load Balancer (inbound):
   - ALB/NLB ở public subnet
   - Forward traffic đến EC2 trong private subnet
   - Giống "port forwarding" nhưng intelligent + HA

4. NAT Gateway:
   - Managed SNAT service
   - KHÔNG hỗ trợ DNAT/port forwarding!
   - Chỉ cho outbound traffic từ private subnet
```

---

## 6. NAT Traversal — Khi NAT gây khó khăn

### Mini example: Gọi điện qua cửa kính cách âm

Bạn ngồi trong phòng cách âm (NAT), bạn có thể GỌI ra ngoài (outbound), nhưng người ngoài KHÔNG THỂ gọi vào (inbound blocked). Vấn đề:
- Video call cần CẢ HAI bên gửi/nhận → 2 bên đều sau NAT → không ai "gọi trước" được!
- VoIP, P2P, gaming → cần NAT traversal

### Tại sao NAT gây vấn đề

```
Scenario: 2 users muốn video call (P2P)

User A: 192.168.1.100 → NAT_A → Public 1.1.1.1
User B: 192.168.2.200 → NAT_B → Public 2.2.2.2

Vấn đề:
- A muốn gửi video đến B
- A biết B's public IP = 2.2.2.2
- A gửi packet đến 2.2.2.2 → NAT_B nhận
- NAT_B check conntrack: "Không có connection nào từ B ra 1.1.1.1"
- NAT_B: DROP! (vì không phải response cho request từ B)

→ CẢ HAI đều bị NAT chặn inbound!
→ Cần kỹ thuật đặc biệt...
```

### Giải pháp NAT Traversal

**1. STUN (Session Traversal Utilities for NAT — RFC 5389):**
```
Concept: Dùng server công cộng để "discover" NAT mapping

1. A gửi packet đến STUN server (S) từ port 5000
2. NAT_A map: 192.168.1.100:5000 → 1.1.1.1:10001
3. STUN server reply: "Tôi thấy bạn là 1.1.1.1:10001"
4. A biết: "Public address của tôi là 1.1.1.1:10001"
5. A gửi info này cho B (qua signaling server)
6. B cũng làm tương tự → biết public address B
7. Cả hai gửi trực tiếp cho nhau (UDP hole punching)
```

**2. TURN (Traversal Using Relays — RFC 5766):**
```
Khi STUN fail (NAT quá strict):
- Relay server ở giữa
- A gửi video → Relay → B
- Tốn bandwidth nhưng LUÔN hoạt động
```

**3. ICE (Interactive Connectivity Establishment — RFC 8445):**
```
ICE = Framework kết hợp STUN + TURN:
1. Thử direct connection
2. Nếu fail → thử STUN (hole punching)
3. Nếu fail → dùng TURN (relay)

WebRTC (video call trong browser) dùng ICE!
```

### NAT Types và ảnh hưởng

| NAT Type | Tên khác | Strict level | P2P khả thi? |
|----------|----------|-------------|-------------|
| Full Cone | Static port mapping | Thấp nhất | Dễ |
| Restricted Cone | — | Trung bình | Được (STUN) |
| Port Restricted Cone | — | Cao | Khó (STUN) |
| Symmetric | — | Cao nhất | Rất khó (cần TURN) |

### Trong AWS

```
NAT Traversal không phải vấn đề trong AWS vì:
1. EC2 trong public subnet có public IP → reachable trực tiếp
2. Nếu cần P2P: Đặt cả 2 endpoints trong public subnet
3. WebRTC/VoIP: Deploy TURN server (EC2) hoặc dùng:
   - Amazon Chime SDK (managed WebRTC)
   - Kinesis Video Streams (P2P video)
4. VPN: Site-to-Site VPN hoặc Client VPN handle NAT traversal
```

---

## 7. CGN/CGNAT — NAT cấp ISP

### Mini example: Chung cư trong chung cư

Bạn ở căn hộ 305 (private IP) trong chung cư A (company NAT). Chung cư A lại nằm trong khu đô thị lớn (ISP) và có chung 1 địa chỉ bưu điện (ISP's shared public IP).

**Double NAT: Device → Home NAT → ISP CGNAT → Internet**

### CGNAT (Carrier-Grade NAT) / NAT444

```
╔══════════════════════════════════════════════════════════════════╗
║                  CGNAT / NAT444 Architecture                     ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  [Your PC]     [Home Router]      [ISP CGNAT]      [Internet]  ║
║  192.168.1.100 → 100.64.1.50  →  203.0.113.1  →   Google     ║
║  Private IP      CGN IP (shared)  Public IP                     ║
║                                                                  ║
║  CGN Address Range: 100.64.0.0/10 (RFC 6598)                   ║
║  = "Shared Address Space" — không phải private, không public   ║
║                                                                  ║
║  ISP dùng CGNAT vì:                                             ║
║  - Hết IPv4 public addresses                                    ║
║  - 1000 customers share 1 public IP                             ║
║  - Rẻ hơn mua thêm IPv4 (giá ~$50/IP trên thị trường!)       ║
║                                                                  ║
║  Vấn đề CGNAT gây ra:                                           ║
║  1. Không thể host server tại nhà (port forwarding FAIL)       ║
║  2. Gaming lag (double NAT = thêm latency)                     ║
║  3. Bị ban IP oan (1000 users share IP → 1 spammer = cả ban)  ║
║  4. Geo-IP inaccurate                                            ║
║  5. NAT traversal khó hơn                                        ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Phát hiện CGNAT

```bash
# Cách kiểm tra bạn có đang sau CGNAT:
# 1. Xem IP WAN của router nhà
#    Nếu nằm trong 100.64.0.0/10 → CGNAT!

# 2. So sánh router WAN IP vs public IP:
$ curl ifconfig.me              # Public IP thấy từ Internet
# So sánh với IP WAN trong router admin page

# Nếu khác nhau → có CGNAT/double NAT!

# 3. Traceroute:
$ traceroute -n google.com
# Nếu hop 2 là 100.64.x.x → ISP CGNAT!
```

### Trong thực tế và AWS

```
Liên quan AWS:
- Nếu customers bị CGNAT → họ share public IP
- VPC Security Group "allow from customer IP" = allow 1000 customers!
- Giải pháp: Client VPN, hoặc authenticate ở application layer

Giải pháp ISP cho CGNAT:
1. Dual-stack (IPv4 CGNAT + IPv6 native)
2. DS-Lite (IPv4-in-IPv6 tunnel + centralized CGNAT)
3. MAP-T/MAP-E (distributed NAT)
4. 464XLAT (IPv6-only backbone + edge translation)
```

---

## 8. Tình huống thực tế — 3 scenarios chi tiết

### Scenario 1: Tại nhà — Port forwarding cho Minecraft server

**Tình huống**: Con bạn muốn host Minecraft server để bạn bè kết nối.

```
Setup:
- PC gaming: 192.168.1.50, Minecraft port 25565
- Router nhà: WAN = 113.161.72.5 (real public IP, may ISP)

Bước 1: Kiểm tra có CGNAT không
$ curl ifconfig.me → 113.161.72.5
Router WAN page → 113.161.72.5
→ KHỚP! Không bị CGNAT ✓

Bước 2: Port Forwarding trên router:
External Port: 25565 (TCP)
Internal IP: 192.168.1.50
Internal Port: 25565

Bước 3: Firewall PC (Windows):
netsh advfirewall firewall add rule name="Minecraft" \
  dir=in action=allow protocol=TCP localport=25565

Bước 4: Cho bạn bè IP:
"Kết nối đến 113.161.72.5:25565"

Nếu ISP CGNAT:
- Port forward sẽ KHÔNG hoạt động!
- Giải pháp: ngrok tunnel, hoặc thuê VPS làm reverse proxy
  $ ngrok tcp 25565
  → Bạn bè kết nối: 0.tcp.ngrok.io:12345
```

### Scenario 2: Trong công ty — NAT Gateway cho 2000 servers

**Tình huống**: 2000 EC2 instances trong private subnet cần download packages từ Internet.

```
Architecture AWS:
┌─────────────────────────────────────────────────────┐
│  VPC 10.0.0.0/16                                     │
│                                                       │
│  ┌───────────────────────────────────────┐           │
│  │ Public Subnet (10.0.1.0/24)           │           │
│  │  ┌────────────┐   ┌────────────┐     │           │
│  │  │ NAT GW #1  │   │ NAT GW #2  │     │  ←  IGW  │
│  │  │ EIP: 1.1.1.1│  │ EIP: 2.2.2.2│    │           │
│  │  └────────────┘   └────────────┘     │           │
│  └──────────────────────┬────────────────┘           │
│                          │                            │
│  ┌──────────────────────┼────────────────┐           │
│  │ Private Subnet AZ-a  │ AZ-b           │           │
│  │  [EC2] [EC2] [EC2]   │ [EC2] [EC2]   │           │
│  │  Route: 0.0.0.0/0    │ Route: 0.0.0.0/0          │
│  │  → NAT GW #1         │ → NAT GW #2    │          │
│  └───────────────────────────────────────┘           │
└─────────────────────────────────────────────────────┘

Vấn đề: 2000 instances → 55,000 connection limit per destination!
Giải pháp:
1. Multiple EIPs per NAT Gateway (up to 8)
   → 8 × 55,000 = 440,000 connections per destination

2. Hoặc múltiple NAT Gateways:
   → Route table split traffic

3. Cost consideration:
   NAT Gateway: $0.045/hour + $0.045/GB processed
   2000 instances × 10GB/month = $900/month cho data processing!
   Alternative: NAT Instance (EC2) — cheaper nhưng không HA
```

### Scenario 3: ISP — Triển khai CGNAT cho 100,000 subscribers

```
Vấn đề ISP:
- Còn 256 IPv4 public addresses (/24)
- 100,000 subscribers cần Internet access
- Tỉ lệ: 100,000 / 256 = 390 subscribers per public IP!

Design:
┌──────────────────────────────────────────────────────┐
│  ISP CGNAT Cluster                                    │
│                                                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ CGNAT #1 │  │ CGNAT #2 │  │ CGNAT #3 │           │
│  │ 50K subs │  │ 50K subs │  │ Standby  │           │
│  │ Pool: /25│  │ Pool: /25│  │ (failover)│          │
│  └──────────┘  └──────────┘  └──────────┘           │
│                                                        │
│  Port allocation per subscriber:                      │
│  100,000 subs × 1000 ports each = 100M ports needed  │
│  256 IPs × 64,000 ports = 16.4M ports available      │
│  → KHÔNG ĐỦ nếu mỗi sub dùng 1000 ports!           │
│                                                        │
│  Giải pháp: Port Block Allocation                     │
│  - Mỗi subscriber: 512 ports block                    │
│  - 256 × 64,000 / 512 = 32,000 subscribers per pool │
│  - Vừa đủ cho 100K (3 pools + oversubscription)      │
│                                                        │
│  Logging (pháp luật yêu cầu):                        │
│  - Log: <timestamp> <public_ip:port_range> <sub_id>  │
│  - Khi công an yêu cầu "ai dùng IP 203.0.113.5     │
│    port 30000 lúc 14:23 ngày 01/06?" → tra log!     │
│                                                        │
└──────────────────────────────────────────────────────┘
```

---

## 9. Bài tập thực hành

### Bài tập 1: Cấu hình NAT trên Linux

```bash
# Lab setup: Linux box với 2 NICs
# eth0: WAN (Internet) - 203.0.113.1
# eth1: LAN (Internal) - 192.168.1.1/24

# Enable forwarding
$ sudo sysctl -w net.ipv4.ip_forward=1

# NAT (MASQUERADE — cho dynamic IP)
$ sudo iptables -t nat -A POSTROUTING \
  -o eth0 -s 192.168.1.0/24 -j MASQUERADE

# Port forwarding: External :8080 → Internal web server :80
$ sudo iptables -t nat -A PREROUTING -i eth0 \
  -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.10:80

# Allow forwarded traffic
$ sudo iptables -A FORWARD -i eth1 -o eth0 -j ACCEPT
$ sudo iptables -A FORWARD -i eth0 -o eth1 \
  -m state --state ESTABLISHED,RELATED -j ACCEPT

# Verify
$ sudo iptables -t nat -L -n -v
$ sudo conntrack -L
```

### Bài tập 2: Monitor NAT connections

```bash
# Real-time connection count
$ watch -n 1 'cat /proc/sys/net/netfilter/nf_conntrack_count'

# Top talkers (most connections)
$ sudo conntrack -L | awk '{print $5}' | sort | uniq -c | sort -rn | head

# Connections per protocol
$ sudo conntrack -L | awk '{print $1}' | sort | uniq -c

# Alert when conntrack > 80% full
MAX=$(cat /proc/sys/net/netfilter/nf_conntrack_max)
CURRENT=$(cat /proc/sys/net/netfilter/nf_conntrack_count)
PCT=$(( CURRENT * 100 / MAX ))
if [ $PCT -gt 80 ]; then
  echo "WARNING: Conntrack at ${PCT}%! ($CURRENT/$MAX)"
fi
```

### Bài tập 3: AWS NAT Gateway Setup

```bash
# Tạo NAT Gateway
$ EIP_ID=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

$ NATGW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id subnet-public-az-a \
  --allocation-id $EIP_ID \
  --query 'NatGateway.NatGatewayId' --output text)

# Update route table cho private subnet
$ aws ec2 create-route \
  --route-table-id rtb-private \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NATGW_ID

# Verify từ EC2 trong private subnet:
$ curl ifconfig.me    # Should show EIP address

# Monitor
$ aws cloudwatch get-metric-statistics \
  --namespace AWS/NATGateway \
  --metric-name ActiveConnectionCount \
  --dimensions Name=NatGatewayId,Value=$NATGW_ID \
  --start-time 2026-06-01T00:00:00Z \
  --end-time 2026-06-01T23:59:59Z \
  --period 300 --statistics Maximum
```

### Bài tập 4: Troubleshoot "Connection timeout from private subnet"

```bash
# Scenario: EC2 in private subnet can't reach Internet

# Check 1: Route table
$ aws ec2 describe-route-tables --route-table-id rtb-xxx
# Look for: 0.0.0.0/0 → nat-xxx (should exist!)

# Check 2: NAT Gateway status
$ aws ec2 describe-nat-gateways --nat-gateway-id nat-xxx
# State should be "available" (not "failed" or "deleted")

# Check 3: Security Group (NAT GW doesn't have SG, but EC2 does)
# EC2's SG must allow OUTBOUND to destination

# Check 4: NACL
# Subnet NACL must allow OUTBOUND + allow INBOUND ephemeral ports

# Check 5: NAT GW in same AZ?
# Best practice: 1 NAT GW per AZ (cross-AZ traffic = charges)

# Check 6: Conntrack limit
# If destination gets 55,000+ connections → throttling
# Fix: Add more EIPs to NAT GW
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Key Points — Những điểm cần nhớ

```
╔══════════════════════════════════════════════════════════════════╗
║                      NAT — TÓM TẮT                              ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. NAT = thay đổi IP trong packet header qua router            ║
║  2. Static NAT (1:1): Server cần public access                  ║
║  3. Dynamic NAT (N:M): Pool public IPs, first-come             ║
║  4. PAT/NAPT (N:1): Phổ biến nhất — phân biệt bằng port       ║
║  5. Connection Tracking: "Sổ ghi" map inside↔outside           ║
║  6. Port Forwarding: DNAT — đưa traffic vào server nội bộ     ║
║  7. CGNAT: ISP-level NAT, 1000+ users/IP, gây vấn đề P2P     ║
║  8. NAT Traversal: STUN/TURN/ICE cho P2P (WebRTC, VoIP)       ║
║  9. NAT breaks: end-to-end, FTP active, IPsec, P2P            ║
║  10. IPv6 = NAT không cần thiết (end-to-end connectivity)      ║
║                                                                  ║
║  AWS Context:                                                    ║
║  • NAT Gateway: Managed PAT, outbound only, 55K conn/dest      ║
║  • Elastic IP: Static 1:1 NAT cho EC2                           ║
║  • ALB/NLB: Replace port forwarding (inbound)                   ║
║  • Private subnet → NAT GW → Internet (outbound)              ║
║  • Public subnet → IGW → Internet (both directions)            ║
║  • NAT GW cost: $0.045/hour + $0.045/GB                        ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Tài liệu đọc thêm

| Tài liệu | Link/Reference | Nội dung |
|-----------|---------------|----------|
| RFC 3022 | tools.ietf.org/html/rfc3022 | Traditional NAT |
| RFC 2663 | tools.ietf.org/html/rfc2663 | NAT Terminology |
| RFC 4787 | tools.ietf.org/html/rfc4787 | NAT UDP Requirements |
| RFC 5382 | tools.ietf.org/html/rfc5382 | NAT TCP Requirements |
| RFC 5389 | tools.ietf.org/html/rfc5389 | STUN |
| RFC 5766 | tools.ietf.org/html/rfc5766 | TURN |
| RFC 6598 | tools.ietf.org/html/rfc6598 | Shared Address Space (CGNAT) |
| RFC 6888 | tools.ietf.org/html/rfc6888 | CGN Requirements |
| AWS NAT Gateway | docs.aws.amazon.com | NAT Gateway Documentation |

---

*Bài tiếp theo: [Routing Fundamentals](/2026-06-01-routing-fundamentals) — Cách packets tìm đường đi qua Internet*
