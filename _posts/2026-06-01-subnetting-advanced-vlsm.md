---
layout: post
title: "Subnetting Advanced & VLSM — Nghệ thuật chia địa chỉ IP tối ưu cho mạng thực tế"
subtitle: "Từ FLSM đến VLSM, CIDR và Route Summarization — chia subnet như kiến trúc sư thiết kế tòa nhà"
tags: [networking, subnetting, vlsm, cidr, addressing, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
comments: true
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 950 | Internet Standard Subnetting Procedure (1985) |
| RFC 1519 | Classless Inter-Domain Routing (CIDR) |
| RFC 1878 | Variable Length Subnet Table For IPv4 |
| RFC 4632 | Classless Inter-domain Routing (CIDR): The Internet Address Assignment and Aggregation Plan |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 5: The Network Layer — Subnets |
| Cisco CCNA Official Cert Guide | Chapter: IP Addressing and Subnetting |
| Todd Lammle — CCNA Complete Study Guide | Chapter 4: IP Subnetting |
| AWS Documentation | VPC — IP Addressing, Subnet Sizing |

---

## 1. Giới thiệu — Tại sao cần biết VLSM?

### Ví dụ đời thường: Chia phòng trong tòa nhà

Bạn là kiến trúc sư được thuê thiết kế một tòa nhà văn phòng. Khách hàng yêu cầu:
- Phòng họp lớn: 100 người
- Phòng làm việc team A: 50 người
- Phòng làm việc team B: 25 người
- Phòng server: 5 người
- 4 hành lang kết nối (chỉ cần 2 cửa mỗi hành lang)

**Cách 1 (FLSM — Fixed Length Subnet Mask):** Chia MỌI phòng bằng nhau = 128 người. Kết quả:
- Phòng họp 128 chỗ → dùng 100, **lãng phí 28**
- Team A: 128 chỗ → dùng 50, **lãng phí 78**
- Team B: 128 chỗ → dùng 25, **lãng phí 103**
- Server: 128 chỗ → dùng 5, **lãng phí 123**
- Hành lang: 128 chỗ × 4 → dùng 2 mỗi cái, **lãng phí 504**!

Tổng: 8 phòng × 128 = 1024 chỗ, chỉ dùng 188 = **lãng phí 82%!**

**Cách 2 (VLSM — Variable Length Subnet Mask):** Chia phòng theo NHU CẦU THỰC TẾ:
- Phòng họp: 128 chỗ (vừa đủ cho 100)
- Team A: 64 chỗ (vừa đủ cho 50)
- Team B: 32 chỗ (vừa đủ cho 25)
- Server: 8 chỗ (vừa đủ cho 5)
- Hành lang: 4 chỗ × 4 (vừa đủ cho 2)

Tổng: 128+64+32+8+16 = **248 chỗ**, dùng 188 = **lãng phí chỉ 24%!**

**VLSM trong networking cũng vậy** — chia subnet có SIZE KHÁC NHAU tùy nhu cầu, thay vì bắt mọi subnet cùng kích thước.

### Concrete scenario: Thiết kế mạng cho công ty 3 chi nhánh

```
Công ty TechCorp được ISP cấp: 192.168.10.0/24 (256 addresses)
Yêu cầu:
- HQ: 100 hosts (Sales + Marketing)
- Branch 1: 50 hosts (Engineering)  
- Branch 2: 25 hosts (Support)
- Server VLAN: 10 hosts
- 3 point-to-point links giữa routers (mỗi link cần 2 IPs)

Tổng: 100+50+25+10+6 = 191 hosts cần addresses

FLSM: Chia đều /25 (126 hosts each) → chỉ được 2 subnets → KHÔNG ĐỦ!
      Chia /26 (62 hosts) → 4 subnets → HQ 100 hosts KHÔNG VỪA!
      
VLSM: HQ = /25 (126 hosts), Branch1 = /26 (62), Branch2 = /27 (30),
      Server = /28 (14), Links = /30 (2 each) → VỪA KHÍT!
```

### Vấn đề VLSM giải quyết

| Vấn đề | Không có VLSM | Có VLSM |
|---------|---------------|----------|
| Subnet lớn nhỏ khác nhau | Không thể — mọi subnet cùng size | Mỗi subnet size riêng |
| Lãng phí IP | 50-90% wasted | 5-30% wasted |
| Point-to-point links | Dùng /24 cho 2 hosts (lãng phí 252!) | Dùng /30 cho 2 hosts |
| Mở rộng mạng | Hết IP nhanh | Tối ưu, dùng lâu hơn |

---

## 2. VLSM là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**VLSM (Variable Length Subnet Mask)** là kỹ thuật cho phép bạn **chia một mạng IP thành nhiều mạng con (subnets) có kích thước KHÁC NHAU**. Thay vì bắt mọi phòng phải cùng diện tích, bạn chia phòng lớn cho bộ phận đông người, phòng nhỏ cho bộ phận ít người.

### Analogy: Cắt bánh pizza

Tưởng tượng bạn có 1 cái pizza (= network /24) cần chia cho nhóm bạn:
- **FLSM** (Fixed): Cắt đều 8 miếng bằng nhau. Người ăn nhiều thiếu, người ăn ít thừa.
- **VLSM** (Variable): Cắt miếng to cho người ăn nhiều, miếng nhỏ cho người ăn ít → ai cũng vừa!

```
┌─────────────────────────────────────────────────────────┐
│              PIZZA = 192.168.10.0/24                      │
│              (256 "miếng" = IP addresses)                 │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  FLSM (/26 — 4 miếng đều 64):                           │
│  ┌────────────┬────────────┬────────────┬────────────┐  │
│  │  64 IPs    │  64 IPs    │  64 IPs    │  64 IPs    │  │
│  │ (cần 100!) │ (cần 50)   │ (cần 25)   │ (cần 10)  │  │
│  │  THIẾU!    │ thừa 14    │ thừa 39    │ thừa 54   │  │
│  └────────────┴────────────┴────────────┴────────────┘  │
│                                                           │
│  VLSM (các size khác nhau):                              │
│  ┌──────────────────────┬────────────┬──────┬────┬──┐   │
│  │     128 IPs /25      │  64 IPs/26 │32/27 │16/28│4 │   │
│  │   (cho 100 hosts)    │  (50 hosts)│(25h) │(10)│/30│   │
│  │     thừa 26          │  thừa 12   │ +5   │ +4 │   │   │
│  └──────────────────────┴────────────┴──────┴────┴──┘   │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### Tiên quyết: Nhớ bảng Powers of 2

```
╔══════════════════════════════════════════════════════════╗
║    BẢNG SUBNET — HỌC THUỘC LÒNG!                       ║
╠══════════════════════════════════════════════════════════╣
║  Prefix │ Subnet Mask      │ Total IPs │ Usable Hosts  ║
║─────────┼──────────────────┼───────────┼───────────────║
║  /24    │ 255.255.255.0    │ 256       │ 254           ║
║  /25    │ 255.255.255.128  │ 128       │ 126           ║
║  /26    │ 255.255.255.192  │ 64        │ 62            ║
║  /27    │ 255.255.255.224  │ 32        │ 30            ║
║  /28    │ 255.255.255.240  │ 16        │ 14            ║
║  /29    │ 255.255.255.248  │ 8         │ 6             ║
║  /30    │ 255.255.255.252  │ 4         │ 2             ║
║  /31    │ 255.255.255.254  │ 2         │ 2*            ║
║  /32    │ 255.255.255.255  │ 1         │ 1 (host)     ║
╠══════════════════════════════════════════════════════════╣
║  * /31: RFC 3021 cho phép dùng cho point-to-point      ║
║    (không cần network/broadcast address)                 ║
║                                                          ║
║  Usable = Total - 2 (trừ network address + broadcast)  ║
║  NGOẠI TRỪ /31 (point-to-point) và /32 (host route)    ║
╚══════════════════════════════════════════════════════════╝
```

---

## 3. Quy trình VLSM — Thuật toán chia subnet

### Mini example: Chia đất cho 5 hộ gia đình

Bạn có mảnh đất 256m² (= /24) cần chia cho 5 hộ với diện tích khác nhau. Quy tắc:
1. **Chia từ LỚN đến NHỎ** (hộ cần nhiều đất nhất trước)
2. **Mỗi mảnh phải là lũy thừa 2** (128, 64, 32, 16, 8, 4...)
3. **Mảnh kế tiếp phải bắt đầu NGAY SAU mảnh trước** (contiguous)

### Thuật toán VLSM 5 bước

```
╔══════════════════════════════════════════════════════════════════╗
║           THUẬT TOÁN VLSM — 5 BƯỚC                              ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Bước 1: LIỆT KÊ yêu cầu số hosts mỗi subnet                  ║
║                                                                  ║
║  Bước 2: SẮP XẾP giảm dần theo số hosts cần                    ║
║                                                                  ║
║  Bước 3: Với mỗi subnet (từ LỚN → NHỎ):                       ║
║    a. Tìm prefix nhỏ nhất chứa đủ hosts:                        ║
║       hosts_needed + 2 ≤ 2^(32-prefix)                          ║
║    b. Assign network address = address TIẾP THEO available      ║
║    c. Tính broadcast = network + (2^(32-prefix)) - 1            ║
║    d. Next available = broadcast + 1                             ║
║                                                                  ║
║  Bước 4: VERIFY: Tổng IPs dùng ≤ IPs available                 ║
║                                                                  ║
║  Bước 5: DOCUMENT — Ghi rõ mỗi subnet                          ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Ví dụ hoàn chỉnh

**Cho:** 172.16.0.0/22 (1024 IPs)
**Cần chia cho:**
- LAN 1: 200 hosts
- LAN 2: 100 hosts
- LAN 3: 50 hosts
- LAN 4: 20 hosts
- WAN 1: 2 hosts (point-to-point)
- WAN 2: 2 hosts (point-to-point)

**Bước 1 & 2: Sắp xếp giảm dần**

| # | Tên | Hosts cần | Hosts + 2 (network+broadcast) |
|---|------|----------|-------------------------------|
| 1 | LAN 1 | 200 | 202 |
| 2 | LAN 2 | 100 | 102 |
| 3 | LAN 3 | 50 | 52 |
| 4 | LAN 4 | 20 | 22 |
| 5 | WAN 1 | 2 | 4 |
| 6 | WAN 2 | 2 | 4 |

**Bước 3: Chia từng subnet**

```
Starting address: 172.16.0.0

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subnet 1: LAN 1 (200 hosts)
  Cần: 202 IPs → next power of 2 ≥ 202 = 256 → /24
  Network:   172.16.0.0/24
  Range:     172.16.0.1 — 172.16.0.254 (254 usable)
  Broadcast: 172.16.0.255
  Next:      172.16.1.0
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subnet 2: LAN 2 (100 hosts)
  Cần: 102 IPs → next power of 2 ≥ 102 = 128 → /25
  Network:   172.16.1.0/25
  Range:     172.16.1.1 — 172.16.1.126 (126 usable)
  Broadcast: 172.16.1.127
  Next:      172.16.1.128
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subnet 3: LAN 3 (50 hosts)
  Cần: 52 IPs → next power of 2 ≥ 52 = 64 → /26
  Network:   172.16.1.128/26
  Range:     172.16.1.129 — 172.16.1.190 (62 usable)
  Broadcast: 172.16.1.191
  Next:      172.16.1.192
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subnet 4: LAN 4 (20 hosts)
  Cần: 22 IPs → next power of 2 ≥ 22 = 32 → /27
  Network:   172.16.1.192/27
  Range:     172.16.1.193 — 172.16.1.222 (30 usable)
  Broadcast: 172.16.1.223
  Next:      172.16.1.224
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subnet 5: WAN 1 (2 hosts)
  Cần: 4 IPs → /30
  Network:   172.16.1.224/30
  Range:     172.16.1.225 — 172.16.1.226 (2 usable)
  Broadcast: 172.16.1.227
  Next:      172.16.1.228
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Subnet 6: WAN 2 (2 hosts)
  Cần: 4 IPs → /30
  Network:   172.16.1.228/30
  Range:     172.16.1.229 — 172.16.1.230 (2 usable)
  Broadcast: 172.16.1.231
  Next:      172.16.1.232
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Bước 4: Verify**
- Tổng IPs dùng: 256+128+64+32+4+4 = 488
- IPs available: 1024 (từ /22)
- Còn dư: 1024-488 = **536 IPs** cho tương lai! 

**So sánh với FLSM:**
- FLSM /24: chỉ được 4 subnets 254 hosts, thiếu cho WAN links
- FLSM /25: 8 subnets 126 hosts, LAN 1 (200) KHÔNG VỪA!
- VLSM: 6 subnets đúng size, còn dư cho mở rộng

### Trong AWS

AWS VPC subnet sizing cũng dùng nguyên tắc tương tự:
- VPC CIDR: /16 đến /28
- Subnet CIDR: phải nằm trong VPC CIDR
- **AWS reserve 5 IPs** trong mỗi subnet (không phải 2):
  - .0 = Network address
  - .1 = VPC Router
  - .2 = DNS server
  - .3 = Reserved for future
  - .255 = Broadcast (không dùng nhưng reserved)
- Usable = 2^(32-prefix) - 5

```
Ví dụ AWS:
/24 subnet: 256 - 5 = 251 usable IPs
/25 subnet: 128 - 5 = 123 usable IPs  
/26 subnet: 64 - 5 = 59 usable IPs
/27 subnet: 32 - 5 = 27 usable IPs
/28 subnet: 16 - 5 = 11 usable IPs (minimum AWS allows)
```

---

## 4. CIDR — Classless Inter-Domain Routing

### Mini example: Từ mã bưu chính cố định đến linh hoạt

Ngày xưa, Việt Nam có mã bưu chính 5 số, chia cứng:
- Số đầu = Miền (1=Bắc, 5=Trung, 7=Nam)
- 2 số tiếp = Tỉnh
- 2 số cuối = Quận/Huyện

Nhưng tỉnh lớn (HCM) cần NHIỀU mã hơn tỉnh nhỏ (Bắc Kạn). Hệ thống cứng không linh hoạt!

**CIDR giải quyết vấn đề tương tự cho IP:**

Trước CIDR (classful): IP chia thành Class A (/8), B (/16), C (/24) — CỨNG.
Sau CIDR: IP có thể dùng BẤT KỲ prefix length nào (/1 đến /32).

### Classful vs Classless

```
╔══════════════════════════════════════════════════════════════════╗
║              CLASSFUL (cũ) vs CLASSLESS (hiện tại)              ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  CLASSFUL (trước 1993):                                         ║
║  ┌───────────────────────────────────────────────────────┐      ║
║  │ Class A: /8   = 16.7 triệu hosts (quá lớn!)          │      ║
║  │ Class B: /16  = 65,534 hosts     (vẫn lớn)           │      ║
║  │ Class C: /24  = 254 hosts        (thường quá nhỏ)    │      ║
║  │                                                        │      ║
║  │ Vấn đề: Công ty cần 1000 hosts?                       │      ║
║  │   - Class C quá nhỏ (254)                             │      ║
║  │   - Class B quá lớn (65,534) → lãng phí 64,000+ IPs! │      ║
║  └───────────────────────────────────────────────────────┘      ║
║                                                                  ║
║  CLASSLESS / CIDR (sau 1993, RFC 1519):                         ║
║  ┌───────────────────────────────────────────────────────┐      ║
║  │ Dùng BẤT KỲ prefix: /8, /9, /10, ..., /30, /31, /32 │      ║
║  │                                                        │      ║
║  │ Công ty cần 1000 hosts?                               │      ║
║  │   → Cấp /22 (1022 usable) → vừa đủ!                 │      ║
║  │                                                        │      ║
║  │ Công ty cần 500 hosts?                                │      ║
║  │   → Cấp /23 (510 usable) → vừa đủ!                  │      ║
║  └───────────────────────────────────────────────────────┘      ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### CIDR Notation

```
192.168.1.0/24

Cách đọc:
- "192.168.1.0" = Network address
- "/24" = 24 bits đầu tiên là NETWORK portion
        = 8 bits còn lại là HOST portion
        = Subnet mask 255.255.255.0

Bất kỳ prefix nào đều hợp lệ:
10.0.0.0/8      = 16,777,214 hosts
172.16.0.0/12   = 1,048,574 hosts
192.168.0.0/16  = 65,534 hosts
10.1.1.0/22     = 1,022 hosts
10.1.1.128/25   = 126 hosts
```

### Supernetting / Route Summarization

CIDR cũng cho phép **gộp nhiều routes thành 1** (summarization/supernetting):

```
Thay vì advertise 4 routes riêng lẻ:
  192.168.0.0/24
  192.168.1.0/24
  192.168.2.0/24
  192.168.3.0/24

Gộp thành 1 route:
  192.168.0.0/22 (bao gồm cả 4 subnets trên)

Tại sao?
- Routing table NHỎ hơn (4 entries → 1 entry)
- Router lookup NHANH hơn
- BGP advertisements ÍT hơn (Internet stability)

Cách tìm summary route:
  192.168.0.0 = 11000000.10101000.00000000.00000000
  192.168.1.0 = 11000000.10101000.00000001.00000000
  192.168.2.0 = 11000000.10101000.00000010.00000000
  192.168.3.0 = 11000000.10101000.00000011.00000000
                                      ││
  22 bits giống nhau ─────────────────┘│
  2 bits khác nhau ───────────────────┘
  
  Summary: 192.168.0.0/22 ✓
```

### Trong thực tế

```bash
# Hiển thị route table với CIDR notation
$ ip route show
default via 192.168.1.1 dev eth0
10.0.0.0/8 via 172.16.0.1 dev tun0       # /8 = summarized
172.16.0.0/22 dev tun0 proto kernel       # /22 = CIDR
192.168.1.0/24 dev eth0 proto kernel

# Tính CIDR nhanh bằng ipcalc
$ ipcalc 192.168.10.0/24
Network:   192.168.10.0/24
Netmask:   255.255.255.0
Broadcast: 192.168.10.255
Hosts/Net: 254

$ ipcalc 172.16.0.0/22
Network:   172.16.0.0/22
Netmask:   255.255.252.0
Broadcast: 172.16.3.255
Hosts/Net: 1022
```

### Trong AWS

```
VPC CIDR design best practices:

1. Primary CIDR: /16 (65,536 IPs — max flexibility)
   VPC: 10.0.0.0/16

2. Subnet layout theo AZ:
   AZ-a: 10.0.0.0/18  (16,384 IPs — Public+Private+DB)
   AZ-b: 10.0.64.0/18
   AZ-c: 10.0.128.0/18
   Reserved: 10.0.192.0/18

3. Mỗi AZ chia tiếp:
   Public:  10.0.0.0/20  (4,096 IPs)
   Private: 10.0.16.0/20 (4,096 IPs)
   Data:    10.0.32.0/20 (4,096 IPs)
   Spare:   10.0.48.0/20 (reserved)

4. Secondary CIDR: Thêm được nếu hết IP
   aws ec2 associate-vpc-cidr-block --vpc-id vpc-xxx --cidr-block 10.1.0.0/16
```

---

## 5. Subnet Design Patterns — Mẫu thiết kế phổ biến

### Mini example: Quy hoạch khu đô thị

Một khu đô thị mới được quy hoạch:
- Khu dân cư (nhiều nhà) = Subnet cho users (lớn)
- Khu thương mại (ít cửa hàng, nhiều người ra vào) = Subnet cho servers (vừa)
- Đường nối khu (hẹp, chỉ cho xe chạy) = Point-to-point links (nhỏ nhất)

### Pattern 1: Three-tier (Web / App / Database)

```
Network: 10.10.0.0/16

Tier 1 - Web (Public facing):
  10.10.0.0/24   — Web servers (254 hosts, room to grow)
  10.10.1.0/24   — Load balancers

Tier 2 - Application (Internal):
  10.10.10.0/24  — App servers
  10.10.11.0/24  — Microservices

Tier 3 - Database (Most restricted):
  10.10.20.0/26  — Primary DBs (14 hosts — DB expensive, ít hosts)
  10.10.20.64/26 — Replica DBs

Management:
  10.10.100.0/28 — Bastion hosts (14 hosts max)
  10.10.100.16/28 — Monitoring

Point-to-Point:
  10.10.255.0/30 — Router to Firewall
  10.10.255.4/30 — Firewall to Core Switch
```

### Pattern 2: Per-Department (Enterprise)

```
Network: 172.16.0.0/16

IT:          172.16.1.0/24   (254 hosts)
Sales:       172.16.2.0/24   (254 hosts)
Marketing:   172.16.3.0/24   (254 hosts)
Engineering: 172.16.4.0/23   (510 hosts — largest dept)
HR:          172.16.6.0/25   (126 hosts)
Finance:     172.16.6.128/25 (126 hosts)
Executives:  172.16.7.0/27   (30 hosts — small, VIP)
Guests:      172.16.8.0/24   (254 hosts — isolated VLAN)

Servers:     172.16.100.0/24
Printers:    172.16.200.0/27 (30 printers max)
VoIP:        172.16.201.0/24 (dedicated for QoS)

WAN Links:   172.16.255.0/24 (subdivided into /30s)
  172.16.255.0/30   — Link to Branch 1
  172.16.255.4/30   — Link to Branch 2
  172.16.255.8/30   — Link to ISP
```

### Pattern 3: AWS Multi-AZ

```
VPC: 10.0.0.0/16

┌─────────────────────────────────────────────────────────┐
│  AZ us-east-1a: 10.0.0.0/19 (8192 IPs)                │
│  ├── Public:  10.0.0.0/21  (2048) — ALB, NAT GW      │
│  ├── Private: 10.0.8.0/21  (2048) — EC2, ECS         │
│  └── Data:    10.0.16.0/21 (2048) — RDS, ElastiCache │
├─────────────────────────────────────────────────────────┤
│  AZ us-east-1b: 10.0.32.0/19 (8192 IPs)               │
│  ├── Public:  10.0.32.0/21 (2048)                      │
│  ├── Private: 10.0.40.0/21 (2048)                      │
│  └── Data:    10.0.48.0/21 (2048)                      │
├─────────────────────────────────────────────────────────┤
│  AZ us-east-1c: 10.0.64.0/19 (8192 IPs)               │
│  ├── Public:  10.0.64.0/21 (2048)                      │
│  ├── Private: 10.0.72.0/21 (2048)                      │
│  └── Data:    10.0.80.0/21 (2048)                      │
├─────────────────────────────────────────────────────────┤
│  Reserved: 10.0.96.0/19 — 10.0.255.255 (future AZ/use)│
└─────────────────────────────────────────────────────────┘

Tại sao design này?
1. /19 per AZ: Đủ lớn cho growth, dễ summarize
2. /21 per tier: 2048 IPs — đủ cho ~2000 containers/instances
3. Symmetrical: Mỗi AZ giống nhau → automation đơn giản
4. Reserved space: 60% space chưa dùng → future-proof
```

---

## 6. Kỹ thuật tính subnet nhanh — "Mental Math"

### Mini example: Nhân viên thu ngân tính nhẩm

Nhân viên thu ngân giỏi không cần máy tính cho 37,000 + 63,000 = 100,000. Tương tự, network engineer giỏi tính subnet TRONG ĐẦU mà không cần calculator.

### Phương pháp "Magic Number"

```
╔══════════════════════════════════════════════════════════════════╗
║              MAGIC NUMBER METHOD                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Magic Number = 256 - (octet thú vị trong subnet mask)          ║
║                                                                  ║
║  Ví dụ 1: 192.168.1.0/26 (mask 255.255.255.192)                ║
║    Magic = 256 - 192 = 64                                       ║
║    Subnets: .0, .64, .128, .192 (cộng 64 mỗi lần)             ║
║    Ranges: .0-.63, .64-.127, .128-.191, .192-.255              ║
║                                                                  ║
║  Ví dụ 2: 10.0.0.0/20 (mask 255.255.240.0)                    ║
║    Octet thú vị = thứ 3 (240)                                   ║
║    Magic = 256 - 240 = 16                                       ║
║    Subnets: 10.0.0.0, 10.0.16.0, 10.0.32.0, ...               ║
║                                                                  ║
║  Ví dụ 3: 172.16.128.0/25 (mask 255.255.255.128)              ║
║    Magic = 256 - 128 = 128                                      ║
║    Subnets: .0, .128 (chỉ 2 subnets trong octet cuối)          ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Xác định network address nhanh

```
Cho IP 192.168.1.137/26

Bước 1: Magic Number = 256 - 192 = 64
Bước 2: 137 ÷ 64 = 2 dư 9
Bước 3: Network = 2 × 64 = 128 → 192.168.1.128/26
Bước 4: Broadcast = 128 + 64 - 1 = 191 → 192.168.1.191
Bước 5: Usable range: 192.168.1.129 — 192.168.1.190

Hoặc đơn giản hơn:
137 nằm giữa 128 (.128) và 192 (.192)
→ Thuộc subnet 192.168.1.128/26
→ Range: .128 — .191
```

### Xác định prefix cần thiết

```
"Tôi cần subnet cho 45 hosts"

Bước 1: 45 + 2 (network + broadcast) = 47
Bước 2: Power of 2 nhỏ nhất ≥ 47 → 64 (2^6)
Bước 3: Host bits = 6 → Prefix = 32 - 6 = /26
Bước 4: Verify: /26 = 64 IPs, 62 usable ≥ 45 ✓

"Tôi cần subnet cho 200 hosts"
200 + 2 = 202 → next power = 256 (2^8) → prefix = /24 ✓

"Tôi cần subnet cho 2 hosts (point-to-point)"
2 + 2 = 4 → 4 (2^2) → prefix = /30
Hoặc dùng /31 (RFC 3021) → chỉ 2 IPs, không waste
```

### Trong thực tế

```bash
# Dùng sipcalc cho calculations phức tạp
$ sipcalc 192.168.1.137/26
Network address  - 192.168.1.128
Broadcast        - 192.168.1.191
Usable range     - 192.168.1.129 - 192.168.1.190
Hosts/Net        - 62

# Python nhanh
$ python3 -c "
import ipaddress
net = ipaddress.ip_network('192.168.1.128/26')
print(f'Network: {net.network_address}')
print(f'Broadcast: {net.broadcast_address}')
print(f'Hosts: {net.num_addresses - 2}')
for i, host in enumerate(net.hosts()):
    if i < 3 or i > net.num_addresses - 5:
        print(f'  {host}')
    elif i == 3:
        print('  ...')
"
```

---

## 7. Route Summarization (Supernetting) — Gộp routes

### Mini example: Đường đi tắt trên bản đồ

Thay vì nhớ: "Đi đường A rẽ phải, qua ngõ B, vào hẻm C, đến nhà D" — bạn chỉ cần nhớ: "Khu Bình Thạnh, đi theo GPS". GPS (router) sẽ tìm đường chi tiết.

Route Summarization = **gộp nhiều đường chi tiết thành 1 đường tổng quát**.

### Khi nào cần Summarization?

```
Trước summarization (routing table 8 entries):
  10.1.0.0/24 → via Router A
  10.1.1.0/24 → via Router A
  10.1.2.0/24 → via Router A
  10.1.3.0/24 → via Router A
  10.1.4.0/24 → via Router A
  10.1.5.0/24 → via Router A
  10.1.6.0/24 → via Router A
  10.1.7.0/24 → via Router A

Sau summarization (1 entry):
  10.1.0.0/21 → via Router A

Vì: 10.1.0.0 — 10.1.7.255 = 8 × /24 = 1 × /21
```

### Thuật toán tìm Summary Route

```
Bước 1: Viết tất cả network addresses dưới dạng binary
Bước 2: Tìm phần GIỐNG NHAU (common prefix)
Bước 3: Đếm số bits giống → đó là prefix length
Bước 4: Summary = common prefix + /prefix_length

Ví dụ:
  10.1.0.0   = 00001010.00000001.00000|000.00000000
  10.1.1.0   = 00001010.00000001.00000|001.00000000
  10.1.2.0   = 00001010.00000001.00000|010.00000000
  10.1.3.0   = 00001010.00000001.00000|011.00000000
                                      ↑
                                21 bits giống nhau

  Summary: 10.1.0.0/21 ✓
```

### Potential Problems

```
╔═══════════════════════════════════════════════════════════════╗
║    CẢNH BÁO: Summarization có thể gây issues!              ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  1. "Suboptimal routing" (đi đường vòng):                    ║
║     Summary 10.1.0.0/21 via Router A                         ║
║     Nhưng 10.1.5.0/24 thực ra qua Router B nhanh hơn!       ║
║     → Traffic đến 10.1.5.x đi qua A (chậm hơn)            ║
║                                                               ║
║  2. "Black hole" (traffic bị drop):                          ║
║     Summary bao gồm subnets CHƯA TỒN TẠI                   ║
║     10.1.0.0/21 bao gồm 10.1.4.0/24                        ║
║     Nhưng 10.1.4.0/24 chưa ai dùng → traffic drop!         ║
║     FIX: Thêm null route cho unused subnets                 ║
║                                                               ║
║  3. "Address overlap" (trùng lặp):                           ║
║     Khi merge 2 networks, VLSM plan có thể bị trùng        ║
║     → Phải re-address (rất tốn effort!)                     ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### Trong AWS

```
Route Summarization trong AWS:

1. VPC Route Table:
   - Default: local route (VPC CIDR) auto-added
   - Summary routes cho on-premises: 
     10.0.0.0/8 → VPN/Direct Connect (thay vì 100 /24s)

2. Transit Gateway Route Table:
   - Summarize VPC routes: 
     10.1.0.0/16 → Attachment VPC-A (thay vì 20 /24s)

3. BGP with Direct Connect:
   - Advertise summary từ on-premises:
     Thay vì 256 × /24 → advertise 1 × /16
   - AWS has BGP route limit (100 routes per VGW)
   - MUST summarize nếu on-premises có nhiều subnets!

4. VPC Peering:
   - KHÔNG thể summarize cross-VPC (must use specific CIDR)
   - Transit Gateway CAN summarize
```

---

## 8. Tình huống thực tế — 3 scenarios chi tiết

### Scenario 1: Tại nhà — Chia subnet cho mạng gia đình thông minh

**Tình huống**: Nhà bạn có router Mikrotik, muốn tách biệt:
- Devices gia đình (laptop, phone): cần Internet
- IoT devices (camera, smart lock): cần isolate
- Guest WiFi: cần Internet nhưng không access LAN

**Design với VLSM:**

```
ISP cấp: 192.168.1.0/24 (private, NAT to Internet)

Chia:
1. Family LAN:  192.168.1.0/26   (62 hosts — laptop, phone, TV)
2. IoT VLAN:    192.168.1.64/26  (62 hosts — camera, sensors)
3. Guest WiFi:  192.168.1.128/26 (62 hosts — visitors)
4. Management:  192.168.1.192/28 (14 hosts — router, NAS, printer)
5. Reserved:    192.168.1.208/28 (future use)
6. Router link: 192.168.1.224/30 (connection to ISP router)

Firewall rules:
- Family → Internet: ALLOW ALL
- Family → IoT: ALLOW (để control cameras)
- IoT → Internet: ALLOW specific (firmware updates only)
- IoT → Family: DENY! (camera không được access laptop)
- Guest → Internet: ALLOW
- Guest → Family/IoT: DENY! (guests isolated)
```

### Scenario 2: Trong công ty — IP Planning cho 5 offices

**Tình huống**: Công ty mở rộng từ 1 office → 5 offices. Cần IP plan cho toàn bộ.

**Design hierarchical:**

```
Allocated block: 10.0.0.0/16 (65,534 IPs)

Level 1 — Per-Office (/19 = 8190 IPs each):
  HQ (Hà Nội):       10.0.0.0/19
  Branch 1 (HCM):    10.0.32.0/19
  Branch 2 (Đà Nẵng): 10.0.64.0/19
  Branch 3 (Cần Thơ): 10.0.96.0/19
  Branch 4 (Hải Phòng): 10.0.128.0/19
  Reserved:           10.0.160.0/19 — 10.0.224.0/19

Level 2 — Per-Department within HQ (/23 = 510 IPs each):
  HQ Users:    10.0.0.0/23    (510 hosts)
  HQ Servers:  10.0.2.0/24    (254 hosts)
  HQ VoIP:     10.0.3.0/24    (254 hosts)
  HQ WiFi:     10.0.4.0/23    (510 hosts)
  HQ IoT:      10.0.6.0/25    (126 hosts)
  HQ Mgmt:     10.0.6.128/28  (14 hosts)
  HQ WAN:      10.0.6.144/28  (14 — subdivide into /30s)

Level 3 — WAN Links (/30):
  HQ ↔ Branch1: 10.0.6.144/30
  HQ ↔ Branch2: 10.0.6.148/30
  HQ ↔ Branch3: 10.0.6.152/30
  HQ ↔ Branch4: 10.0.6.156/30

Benefits:
- Route summarization: Branch 1 = 10.0.32.0/19 (1 route!)
- Easy to grow: Each office has 8190 IPs
- Predictable: Any admin knows "10.0.64.x = Đà Nẵng"
```

### Scenario 3: Cloud/ISP — AWS Multi-Account Subnet Strategy

**Tình huống**: Organization 20 AWS accounts, cần non-overlapping IP plan.

```
Master plan: 10.0.0.0/8 (lớn nhất private range)

Per-Environment (/12 = 1M IPs):
  Production:   10.0.0.0/12   (10.0.0.0 — 10.15.255.255)
  Staging:      10.16.0.0/12  (10.16.0.0 — 10.31.255.255)
  Development:  10.32.0.0/12  (10.32.0.0 — 10.47.255.255)
  Shared Svcs:  10.48.0.0/12  (10.48.0.0 — 10.63.255.255)

Per-Account in Production (/16 = 65K IPs):
  Account: ecommerce-prod     → VPC 10.0.0.0/16
  Account: payments-prod      → VPC 10.1.0.0/16
  Account: analytics-prod     → VPC 10.2.0.0/16
  Account: ml-platform-prod   → VPC 10.3.0.0/16
  ...

Per-VPC subnet layout (/20 per tier per AZ):
  10.0.0.0/16
  ├── AZ-a Public:  10.0.0.0/20
  ├── AZ-a Private: 10.0.16.0/20
  ├── AZ-a Data:    10.0.32.0/20
  ├── AZ-b Public:  10.0.48.0/20
  ├── AZ-b Private: 10.0.64.0/20
  ├── AZ-b Data:    10.0.80.0/20
  └── Reserved:     10.0.96.0/19 — 10.0.255.255

Transit Gateway Routing:
  Production:  10.0.0.0/12  → TGW attachment group "prod"
  Staging:     10.16.0.0/12 → TGW attachment group "staging"
  → Clean summarization, minimal route table entries
```

---

## 9. Bài tập thực hành

### Bài tập 1: VLSM Design Challenge

```
Cho: 192.168.50.0/24
Yêu cầu chia cho:
- Subnet A: 60 hosts
- Subnet B: 28 hosts
- Subnet C: 12 hosts
- Subnet D: 5 hosts
- Subnet E: 2 hosts (point-to-point)

Lời giải:
Sắp xếp giảm dần: A(60), B(28), C(12), D(5), E(2)

A: 60+2=62 → 64 (2^6) → /26
   Network: 192.168.50.0/26
   Range: .1 — .62 | Broadcast: .63

B: 28+2=30 → 32 (2^5) → /27
   Network: 192.168.50.64/27
   Range: .65 — .94 | Broadcast: .95

C: 12+2=14 → 16 (2^4) → /28
   Network: 192.168.50.96/28
   Range: .97 — .110 | Broadcast: .111

D: 5+2=7 → 8 (2^3) → /29
   Network: 192.168.50.112/29
   Range: .113 — .118 | Broadcast: .119

E: 2+2=4 → 4 (2^2) → /30
   Network: 192.168.50.120/30
   Range: .121 — .122 | Broadcast: .123

Verify: 64+32+16+8+4 = 124 ≤ 256 ✓
Remaining: 192.168.50.124 — 192.168.50.255 (132 IPs for future)
```

### Bài tập 2: Tìm Summary Route

```
Gộp các routes sau thành 1 summary:
  10.1.32.0/24
  10.1.33.0/24
  10.1.34.0/24
  10.1.35.0/24
  10.1.36.0/24
  10.1.37.0/24
  10.1.38.0/24
  10.1.39.0/24

Lời giải:
Binary (octet thứ 3):
  32 = 00100|000
  33 = 00100|001
  34 = 00100|010
  35 = 00100|011
  36 = 00100|100
  37 = 00100|101
  38 = 00100|110
  39 = 00100|111
         ↑
  5 bits giống (00100), 3 bits khác

Full prefix match: 10.1 (16 bits) + 00100 (5 bits) = 21 bits
Summary: 10.1.32.0/21 ✓
Verify: /21 covers 10.1.32.0 — 10.1.39.255 (8 × /24) ✓
```

### Bài tập 3: AWS VPC Design

```bash
# Thiết kế VPC cho e-commerce application:
# - 3 AZs
# - Public subnet (ALB, NAT GW): ~100 IPs per AZ
# - Private subnet (ECS tasks): ~1000 IPs per AZ
# - Database subnet (RDS): ~20 IPs per AZ
# - Reserve 50% for growth

# Tính toán:
# Per AZ needed: 100+1000+20 = 1120, with 50% reserve → 2240
# 3 AZs: 6720 IPs minimum
# Choose: 10.0.0.0/16 (65536 IPs — plenty of room)

# Design:
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# AZ-a subnets:
aws ec2 create-subnet --vpc-id $VPC \
  --cidr-block 10.0.0.0/24 --availability-zone us-east-1a    # Public (254)
aws ec2 create-subnet --vpc-id $VPC \
  --cidr-block 10.0.1.0/22 --availability-zone us-east-1a    # Private (1022)
aws ec2 create-subnet --vpc-id $VPC \
  --cidr-block 10.0.5.0/27 --availability-zone us-east-1a    # Data (27)

# AZ-b subnets:
aws ec2 create-subnet --vpc-id $VPC \
  --cidr-block 10.0.16.0/24 --availability-zone us-east-1b
aws ec2 create-subnet --vpc-id $VPC \
  --cidr-block 10.0.17.0/22 --availability-zone us-east-1b
aws ec2 create-subnet --vpc-id $VPC \
  --cidr-block 10.0.21.0/27 --availability-zone us-east-1b

# AZ-c: 10.0.32.0/24, 10.0.33.0/22, 10.0.37.0/27
```

### Bài tập 4: Troubleshooting — "Two hosts can't communicate"

```
Scenario:
Host A: 192.168.1.130/26
Host B: 192.168.1.190/26

Q: Có thể ping nhau trực tiếp (without router)?

A: Kiểm tra cùng subnet không!

Host A: 192.168.1.130/26
  Network: 130 ÷ 64 = 2 → 2×64 = 128 → subnet 192.168.1.128/26
  Range: .128 — .191

Host B: 192.168.1.190/26
  Network: 190 ÷ 64 = 2 dư 62 → 2×64 = 128 → subnet 192.168.1.128/26
  Range: .128 — .191

Kết luận: CÙNG SUBNET! → Có thể ping trực tiếp ✓

Bây giờ thử:
Host C: 192.168.1.200/26
  Network: 200 ÷ 64 = 3 → 3×64 = 192 → subnet 192.168.1.192/26
  Range: .192 — .255

Host A vs Host C: KHÁC subnet! → Cần ROUTER để communicate!
```

### Bài tập 5: Verify với commands

```bash
# Kiểm tra 2 IPs cùng subnet không
$ python3 -c "
import ipaddress
a = ipaddress.ip_interface('192.168.1.130/26')
b = ipaddress.ip_interface('192.168.1.190/26')
print(f'Host A network: {a.network}')
print(f'Host B network: {b.network}')
print(f'Same subnet: {a.network == b.network}')
"

# Liệt kê tất cả subnets trong /24 khi chia /26
$ python3 -c "
import ipaddress
net = ipaddress.ip_network('192.168.1.0/24')
for subnet in net.subnets(prefixlen_diff=2):
    print(f'{subnet} -> hosts: {subnet.num_addresses-2}')
"

# Output:
# 192.168.1.0/26 -> hosts: 62
# 192.168.1.64/26 -> hosts: 62
# 192.168.1.128/26 -> hosts: 62
# 192.168.1.192/26 -> hosts: 62
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Key Points — Những điểm cần nhớ

```
╔══════════════════════════════════════════════════════════════════╗
║             SUBNETTING & VLSM — TÓM TẮT                        ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. FLSM = mọi subnet cùng size (lãng phí)                    ║
║  2. VLSM = mỗi subnet size riêng (tiết kiệm)                  ║
║  3. Quy tắc VLSM: Chia từ LỚN → NHỎ                           ║
║  4. Usable hosts = 2^host_bits - 2 (trừ network + broadcast)   ║
║  5. Point-to-point links: dùng /30 hoặc /31                    ║
║  6. CIDR: Bỏ classful, dùng any prefix length                  ║
║  7. Summarization: Gộp routes giảm routing table                ║
║  8. Magic Number: 256 - subnet_octet_value                     ║
║  9. AWS reserve 5 IPs per subnet (not 2)                       ║
║  10. Design: hierarchical, predictable, room to grow            ║
║                                                                  ║
║  Quick Reference:                                                ║
║  /24 = 254 hosts    /27 = 30 hosts    /30 = 2 hosts            ║
║  /25 = 126 hosts    /28 = 14 hosts    /31 = 2 hosts (p2p)     ║
║  /26 = 62 hosts     /29 = 6 hosts     /32 = 1 host (loopback) ║
║                                                                  ║
║  AWS VPC Best Practice:                                         ║
║  • VPC: /16 (maximum flexibility)                               ║
║  • Subnet minimum: /28 (11 usable IPs)                         ║
║  • Design for 3 AZs + growth                                   ║
║  • Non-overlapping CIDRs across accounts                        ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Tài liệu đọc thêm

| Tài liệu | Link/Reference | Nội dung |
|-----------|---------------|----------|
| RFC 950 | tools.ietf.org/html/rfc950 | Original Subnetting Procedure |
| RFC 1519 | tools.ietf.org/html/rfc1519 | CIDR Specification |
| RFC 1878 | tools.ietf.org/html/rfc1878 | VLSM Subnet Table |
| RFC 3021 | tools.ietf.org/html/rfc3021 | /31 for Point-to-Point |
| RFC 4632 | tools.ietf.org/html/rfc4632 | Current CIDR Spec |
| AWS VPC Sizing | docs.aws.amazon.com | VPC and Subnet Sizing |
| Subnet Calculator | www.subnet-calculator.com | Online tool |
| ipcalc | linux man page | CLI subnet calculator |

---

*Bài tiếp theo: [DHCP Protocol](/2026-06-01-dhcp-protocol) — Cách thiết bị tự động nhận IP khi kết nối mạng*
