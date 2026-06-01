---
layout: post
title: "DHCP Protocol — Cách thiết bị tự động nhận IP khi kết nối mạng"
subtitle: "Hiểu sâu quy trình DORA, Lease Management, Relay Agent và DHCP trong AWS VPC — từ RFC 2131"
tags: [networking, dhcp, layer7, addressing, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
comments: true
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 2131 | Dynamic Host Configuration Protocol (DHCP) — 1997 |
| RFC 2132 | DHCP Options and BOOTP Vendor Extensions |
| RFC 3046 | DHCP Relay Agent Information Option (Option 82) |
| RFC 4361 | Node-specific Client Identifiers for DHCP |
| RFC 8415 | Dynamic Host Configuration Protocol for IPv6 (DHCPv6) |
| Stevens, W.R. — TCP/IP Illustrated, Vol. 1 | Chapter 16: BOOTP and DHCP |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 5: DHCP |
| Cisco Documentation | DHCP Configuration Guide |
| AWS Documentation | VPC DHCP Options Sets |

---

## 1. Giới thiệu — Tại sao cần biết DHCP?

### Ví dụ đời thường: Khách sạn và quầy lễ tân

Khi bạn đến khách sạn, bạn KHÔNG TỰ chọn phòng — quầy lễ tân:
1. **Kiểm tra phòng trống** (available addresses)
2. **Cấp cho bạn chìa khóa + số phòng** (IP address)
3. **Cho bạn biết**: WiFi password (DNS), nhà hàng ở tầng mấy (gateway), checkout lúc mấy giờ (lease time)
4. **Ghi sổ**: Bạn ở phòng 305 từ ngày X đến ngày Y (lease record)
5. **Khi checkout**: Phòng trở lại trống, sẵn sàng cho khách mới

**DHCP hoạt động y hệt** — nó là "lễ tân" của mạng, tự động cấp IP address và network configuration cho thiết bị khi kết nối.

### Concrete scenario: Kết nối WiFi quán café

Khi bạn bật WiFi tại Starbucks:
1. Phone bạn: "Có ai quản lý mạng ở đây không?" → **DHCP Discover** (broadcast)
2. Router Starbucks: "Có tôi! Đây là IP cho bạn: 192.168.1.45" → **DHCP Offer**
3. Phone: "OK, tôi muốn dùng IP đó!" → **DHCP Request**
4. Router: "Confirmed! IP này là của bạn trong 2 giờ" → **DHCP Acknowledge**

**Trong vòng 0.1-2 giây**, phone bạn có đầy đủ: IP address, subnet mask, default gateway, DNS server — SẴN SÀNG lên Internet! Tất cả tự động, bạn không cần biết gì về networking.

### Nếu không có DHCP thì sao?

Tưởng tượng mỗi lần kết nối WiFi, bạn phải:
```
1. Mở Settings → Network → Manual Configuration
2. Nhập IP: 192.168.1.??? (phải hỏi admin số nào trống!)
3. Nhập Subnet Mask: 255.255.255.0 (phải biết mới nhập được!)
4. Nhập Gateway: 192.168.1.1 (phải hỏi hoặc đoán!)
5. Nhập DNS: 8.8.8.8 (phải tự biết!)
```

Với 500 thiết bị trong công ty = 500 lần cấu hình thủ công! Nếu ai nhập sai = conflict, mất mạng!

### Vấn đề DHCP giải quyết

| Vấn đề | Không có DHCP | Có DHCP |
|---------|---------------|---------|
| Cấp IP | Thủ công, dễ sai | Tự động, không conflict |
| Thiết bị mới | Phải cấu hình tay | Plug-and-play |
| Thay đổi DNS/Gateway | Update từng máy | Update 1 chỗ → tất cả áp dụng |
| IP conflict | Admin phải check sổ | DHCP tự track, không trùng |
| Thiết bị rời mạng | IP bị "giữ chỗ" mãi | Lease hết hạn → IP trả lại pool |
| Scaling 1000+ devices | Nightmare! | Effortless |

---

## 2. DHCP là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**DHCP** (Dynamic Host Configuration Protocol) là **dịch vụ tự động cấp phát cấu hình mạng** cho thiết bị. Khi device kết nối vào mạng, DHCP server tự động cung cấp:

- **IP Address** — "số nhà" của bạn trên mạng
- **Subnet Mask** — phạm vi "khu phố" bạn thuộc về
- **Default Gateway** — "cổng ra" để đi sang khu khác / Internet
- **DNS Server** — "danh bạ" để tra tên website thành IP
- **Lease Time** — bạn được giữ IP bao lâu (1 giờ? 1 ngày? 1 tuần?)

### Analogy: Bãi đỗ xe tự động

```
╔══════════════════════════════════════════════════════════════════╗
║                 BÃI ĐỖ XE TỰ ĐỘNG (= DHCP)                    ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  🚗 Xe bạn đến (device mới)                                    ║
║    │                                                             ║
║    ▼                                                             ║
║  🎫 Máy phát vé tự động (DHCP Server)                          ║
║    │  "Slot 45 trống, hạn 2 giờ"                               ║
║    │  IP=192.168.1.45, Lease=7200s                              ║
║    ▼                                                             ║
║  🅿️ Bạn đỗ vào slot 45 (device có IP)                         ║
║    │                                                             ║
║    ├── Sau 1 giờ: "Bạn còn cần slot không?" (Lease renewal)    ║
║    │   "Có!" → gia hạn thêm 2 giờ                              ║
║    │                                                             ║
║    └── Bạn rời đi: "Trả slot" (DHCP Release)                   ║
║        Slot 45 trở lại trống cho xe khác                        ║
║                                                                  ║
║  📋 Bảng quản lý: "Slot 45 = Xe biển 30A-12345, hạn 14:00"    ║
║     (Lease Table: IP ↔ MAC ↔ Expiry)                           ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### DHCP Architecture

```
┌──────────────┐                    ┌──────────────────┐
│  DHCP Client │  ← UDP Port 68    │                  │
│  (Laptop,    │───────────────────▶│  DHCP Server     │
│   Phone,     │◀───────────────────│  (Router hoặc    │
│   IoT...)    │  → UDP Port 67    │   dedicated)     │
└──────────────┘                    └──────────────────┘
                     Broadcast
                   (255.255.255.255)

Ports:
- DHCP Server listens on: UDP 67
- DHCP Client listens on: UDP 68
- Tại sao 2 ports khác nhau? Vì client CHƯA CÓ IP khi gửi Discover
  → dùng source 0.0.0.0:68, destination 255.255.255.255:67
```

---

## 3. DORA Process — 4 bước lấy IP address

### Mini example: Xin việc tại công ty

Quy trình DORA giống xin việc:
1. **D**iscover = Nộp CV đến MỌI công ty (broadcast): "Ai tuyển không?"
2. **O**ffer = Công ty reply: "Chúng tôi muốn tuyển bạn, lương X, ngày bắt đầu Y"
3. **R**equest = Bạn chọn 1 công ty: "Tôi chấp nhận offer của ABC Corp!"
4. **A**cknowledge = ABC Corp confirm: "Welcome! Đây là badge, email, parking..."

### Chi tiết từng bước

```
╔══════════════════════════════════════════════════════════════════════╗
║                    QUY TRÌNH DORA (4 bước)                          ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║  ┌────────────┐                              ┌─────────────────┐    ║
║  │   CLIENT   │                              │   DHCP SERVER   │    ║
║  └──────┬─────┘                              └────────┬────────┘    ║
║         │                                              │             ║
║  Step 1 │──── DHCP DISCOVER (broadcast) ──────────────▶│             ║
║         │  Src: 0.0.0.0:68                             │             ║
║         │  Dst: 255.255.255.255:67                     │             ║
║         │  "Tôi cần IP! MAC tôi là XX:XX:XX:XX:XX:XX" │             ║
║         │                                              │             ║
║  Step 2 │◀─── DHCP OFFER (unicast or broadcast) ──────│             ║
║         │  "IP 192.168.1.45 available!                 │             ║
║         │   Gateway: 192.168.1.1                       │             ║
║         │   DNS: 8.8.8.8                               │             ║
║         │   Lease: 86400 seconds (24h)"                │             ║
║         │                                              │             ║
║  Step 3 │──── DHCP REQUEST (broadcast!) ──────────────▶│             ║
║         │  "Tôi muốn IP 192.168.1.45 từ server A!"    │             ║
║         │  (broadcast vì các server khác cần biết      │             ║
║         │   client đã chọn ai → withdraw offer)        │             ║
║         │                                              │             ║
║  Step 4 │◀─── DHCP ACK (unicast or broadcast) ────────│             ║
║         │  "Confirmed! 192.168.1.45 là của bạn.        │             ║
║         │   Lease bắt đầu ngay bây giờ."              │             ║
║         │                                              │             ║
║         │  [Client cấu hình IP → Online!]              │             ║
║         │                                              │             ║
╚══════════════════════════════════════════════════════════════════════╝
```

### DHCP Message Format (chi tiết)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|     op (1)    |   htype (1)   |   hlen (1)    |   hops (1)    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            xid (4)                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           secs (2)            |           flags (2)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          ciaddr  (4)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          yiaddr  (4)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          siaddr  (4)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          giaddr  (4)                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          chaddr  (16)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          sname   (64)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          file    (128)                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                          options (variable)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Giải thích fields quan trọng:
- op: 1=Request (client→server), 2=Reply (server→client)
- xid: Transaction ID (random, để match request↔reply)
- ciaddr: Client IP (filled khi renewing)
- yiaddr: "Your" IP address (server cấp cho client)
- siaddr: Server IP
- giaddr: Gateway/Relay Agent IP
- chaddr: Client hardware (MAC) address
- options: Chứa tất cả cấu hình (mask, gateway, DNS, lease time...)
```

### Tại sao Step 3 (REQUEST) là BROADCAST?

Đây là câu hỏi phỏng vấn phổ biến! Lý do:

```
Scenario: Mạng có 2 DHCP servers (Server A và Server B)

1. Client broadcast DISCOVER → cả A và B đều nhận
2. Server A offers 192.168.1.45
   Server B offers 192.168.1.100
3. Client chọn Server A → broadcast REQUEST:
   "Tôi chọn IP .45 từ Server A!"
   
   TẠI SAO BROADCAST?
   → Server B CŨNG nhận REQUEST → biết client KHÔNG chọn mình
   → Server B withdraw offer .100 → trả .100 về pool
   
   Nếu REQUEST là unicast → Server B không biết
   → .100 bị "lock" vô ích → wasted address!
```

### Trong thực tế

```bash
# Xem DHCP lease hiện tại trên Linux
$ cat /var/lib/dhcp/dhclient.leases

# Xem DHCP info trên interface
$ ip addr show dev eth0
$ cat /var/lib/NetworkManager/dhclient-eth0.conf

# Release và Renew IP
$ sudo dhclient -r eth0      # Release (trả IP)
$ sudo dhclient eth0          # Discover mới

# Capture DHCP traffic
$ sudo tcpdump -i eth0 port 67 or port 68 -v

# Trên macOS:
$ ipconfig getpacket en0      # Xem DHCP packet nhận được
```

### Trong AWS

- **VPC DHCP**: AWS cung cấp DHCP server built-in cho mỗi VPC
- Bạn KHÔNG THỂ tắt DHCP trong VPC — nó luôn bật
- Instances nhận IP từ subnet range qua DHCP
- **DHCP Options Set**: Customize DNS, domain name, NTP servers
- AWS DHCP KHÔNG support custom gateway (gateway = VPC router, auto)

---

## 4. DHCP Lease Lifecycle — Vòng đời của IP

### Mini example: Thẻ thư viện có hạn

Bạn mượn sách thư viện:
- Hạn mượn: 14 ngày (= Lease Time)
- Sau 7 ngày (50% lease): Bạn có thể gia hạn (= T1 Renewal)
- Sau 12 ngày (87.5%): Gia hạn khẩn cấp (= T2 Rebinding)
- Ngày 14: Nếu không gia hạn → phải trả sách (= Lease Expired)

### Lease Timers

```
╔══════════════════════════════════════════════════════════════════╗
║                DHCP LEASE TIMERS                                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Lease Time = T (ví dụ: 24 hours = 86400 seconds)              ║
║                                                                  ║
║  T1 (Renewal Timer) = 50% of T = 12 hours                      ║
║    → Client gửi DHCP REQUEST (unicast) đến server               ║
║    → Nếu OK → lease reset về full T                             ║
║    → Nếu FAIL → đợi T2                                         ║
║                                                                  ║
║  T2 (Rebinding Timer) = 87.5% of T = 21 hours                  ║
║    → Client gửi DHCP REQUEST (BROADCAST) tìm BẤT KỲ server     ║
║    → Nếu OK → lease reset                                      ║
║    → Nếu FAIL → đợi expire                                     ║
║                                                                  ║
║  T (Lease Expiry) = 100% = 24 hours                            ║
║    → IP address KHÔNG CÒN hợp lệ                               ║
║    → Client PHẢI ngừng dùng IP                                  ║
║    → Bắt đầu lại DORA từ đầu!                                  ║
║                                                                  ║
║  Timeline:                                                       ║
║  ├──────────┼──────────────────────┼────────┼───┤               ║
║  0         T1 (50%)              T2 (87.5%)   T                 ║
║  Lease     Renew                 Rebind     Expire              ║
║  Start     (unicast)             (broadcast)                    ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Renewal vs Rebinding

| | T1 (Renewal) | T2 (Rebinding) |
|--|-------------|----------------|
| When | 50% lease time | 87.5% lease time |
| To whom | Unicast đến ORIGINAL server | Broadcast đến ANY server |
| Failure | Retry until T2 | Retry until Expiry |

Tại sao cần 2 phases? Nếu server gốc down, client vẫn có cơ hội tìm server khác (qua broadcast ở giai đoạn Rebinding) trước khi lease hết hạn.

### DHCP Message Types (Options 53)

| Value | Type | Mô tả |
|-------|------|--------|
| 1 | DHCPDISCOVER | Client tìm server |
| 2 | DHCPOFFER | Server propose config |
| 3 | DHCPREQUEST | Client accept/renew/rebind |
| 4 | DHCPDECLINE | Client phát hiện IP conflict! |
| 5 | DHCPACK | Server confirm |
| 6 | DHCPNAK | Server reject (IP không còn available) |
| 7 | DHCPRELEASE | Client trả IP về pool |
| 8 | DHCPINFORM | Client đã có IP, chỉ xin config info |

### Lease Time Best Practices

| Môi trường | Lease Time khuyến nghị | Lý do |
|-----------|----------------------|--------|
| WiFi quán café | 1-2 giờ | Khách đến rồi đi nhanh |
| Office wired | 8-24 giờ | Nhân viên ở cả ngày |
| Server/Printer | 7-30 ngày hoặc Reservation | IP cần stable |
| IoT sensors | 24-72 giờ | Devices luôn connected |
| VPN clients | 8 giờ | Session-based |

### Trong AWS

```
AWS VPC DHCP Lease:
- Lease time: Cố định (không thể thay đổi)
- EC2 instances: IP assigned khi launch, giữ cho đến terminate
- DHCP chỉ dùng để cấp config (DNS, domain) — IP gán bởi AWS
- Elastic IP: static, tách biệt khỏi DHCP

Custom DHCP Options Set:
aws ec2 create-dhcp-options \
  --dhcp-configurations \
    "Key=domain-name,Values=corp.internal" \
    "Key=domain-name-servers,Values=10.0.0.2,8.8.8.8" \
    "Key=ntp-servers,Values=169.254.169.123"
```

---

## 5. DHCP Options — Cấu hình mở rộng

### Mini example: Form check-in khách sạn

Khi check-in khách sạn, form có nhiều ô:
- Ô bắt buộc: Tên, Số phòng, Ngày trả phòng (= IP, Mask, Lease)
- Ô tùy chọn: WiFi password, Breakfast time, Parking spot (= DHCP Options)

### Các DHCP Options quan trọng

| Option # | Tên | Mô tả | Ví dụ |
|----------|-----|--------|-------|
| 1 | Subnet Mask | Mask cho subnet | 255.255.255.0 |
| 3 | Router | Default Gateway | 192.168.1.1 |
| 6 | DNS Servers | Danh sách DNS | 8.8.8.8, 8.8.4.4 |
| 12 | Hostname | Tên thiết bị | "laptop-huy" |
| 15 | Domain Name | DNS domain | "corp.internal" |
| 28 | Broadcast Address | Broadcast IP | 192.168.1.255 |
| 42 | NTP Servers | Time servers | 169.254.169.123 |
| 43 | Vendor Specific | Vendor-specific info | (varies) |
| 51 | Lease Time | Thời gian thuê IP | 86400 (24h) |
| 53 | Message Type | DORA type | 1-8 |
| 54 | Server Identifier | IP DHCP server | 192.168.1.1 |
| 55 | Parameter Request | Client xin options | 1,3,6,15,28,42 |
| 61 | Client Identifier | Unique ID của client | MAC hoặc DUID |
| 66 | TFTP Server | PXE boot server | 10.0.0.50 |
| 67 | Bootfile Name | PXE boot file | pxelinux.0 |
| 82 | Relay Agent Info | Relay agent metadata | (circuit/remote ID) |
| 119 | Domain Search List | DNS search domains | "corp.com,dev.corp.com" |
| 121 | Classless Static Routes | Custom routes | "10.0.0.0/8 via 192.168.1.254" |
| 150 | TFTP Server Address | Cisco IP Phone config | (IP address) |

### Option 82 — DHCP Relay Agent Information

Khi DHCP server ở subnet khác với client (qua relay agent), server cần biết client đến từ switch port nào:

```
Client → Relay Agent → DHCP Server

Relay Agent thêm Option 82:
- Circuit ID: "Switch-Floor3, Port Gi0/5" (client ở port nào)
- Remote ID: "Switch MAC: AA:BB:CC:DD:EE:FF" (switch nào)

Server dùng Option 82 để:
- Assign đúng IP pool cho đúng subnet
- Logging: biết thiết bị nào ở đâu
- Security: chống DHCP spoofing
```

### PXE Boot — Cài OS qua mạng

```
Scenario: 100 PCs mới, cần cài Windows/Linux

Thay vì: Cắm USB từng máy → 100 lần cài
→ Dùng PXE + DHCP:

1. PC bật → NIC gửi DHCP Discover với Option 60 "PXEClient"
2. DHCP server: "IP = .50, TFTP server = 10.0.0.100, File = pxelinux.0"
3. PC download boot file qua TFTP
4. Boot file → download OS installer
5. PC tự cài OS!

DHCP Options cho PXE:
- Option 66: TFTP server name/IP
- Option 67: Boot filename
- Option 60: Client class identifier
```

### Trong thực tế

```bash
# Cấu hình DHCP server trên Linux (dhcpd)
$ cat /etc/dhcp/dhcpd.conf
subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.100 192.168.1.200;
  option routers 192.168.1.1;
  option domain-name-servers 8.8.8.8, 8.8.4.4;
  option domain-name "home.local";
  default-lease-time 28800;    # 8 hours
  max-lease-time 86400;        # 24 hours
  
  # Static reservation
  host printer-office {
    hardware ethernet 00:1A:2B:3C:4D:5E;
    fixed-address 192.168.1.50;
  }
}

# Xem current leases
$ cat /var/lib/dhcp/dhcpd.leases
```

---

## 6. DHCP Relay Agent — Vượt qua subnet boundary

### Mini example: Phiên dịch tại tòa đại sứ

Bạn (client) nói tiếng Việt, đại sứ quán (server) ở nước khác nói tiếng Anh. Phiên dịch (relay agent) nghe bạn nói → dịch và chuyển đến đại sứ quán → nhận trả lời → dịch lại cho bạn.

DHCP Relay Agent làm tương tự: chuyển DHCP broadcast từ client sang unicast đến server ở subnet khác.

### Vấn đề: DHCP Discover là broadcast!

```
Broadcast (255.255.255.255) KHÔNG qua router!

Nếu DHCP server ở subnet khác:
┌──────────┐     ┌──────────┐     ┌──────────────┐
│  Client   │────│  Router  │────│ DHCP Server  │
│ Subnet A  │    │          │    │  Subnet B    │
└──────────┘     └──────────┘     └──────────────┘

Client broadcast "DISCOVER" → Router CHẶN broadcast
→ Server KHÔNG nhận được!

Giải pháp 1: Đặt DHCP server ở MỌI subnet (tốn kém!)
Giải pháp 2: DHCP Relay Agent (helper address) trên Router
```

### Cách Relay Agent hoạt động

```
╔══════════════════════════════════════════════════════════════════╗
║              DHCP RELAY AGENT PROCESS                             ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. Client broadcast DISCOVER (255.255.255.255)                 ║
║                                                                  ║
║  2. Router (relay agent) nhận broadcast:                        ║
║     a. Đặt giaddr = IP của router interface nhận broadcast      ║
║        (giaddr cho server biết client ở subnet nào)             ║
║     b. Chuyển message → UNICAST đến DHCP server                 ║
║     c. Increment hops field                                      ║
║                                                                  ║
║  3. DHCP Server nhận unicast message:                           ║
║     a. Nhìn giaddr = 192.168.10.1                               ║
║     b. "À, client ở subnet 192.168.10.0/24"                     ║
║     c. Chọn IP từ pool cho subnet 192.168.10.0/24              ║
║     d. Gửi OFFER unicast NGƯỢC LẠI cho relay agent             ║
║                                                                  ║
║  4. Relay Agent nhận OFFER:                                      ║
║     a. Forward đến client (broadcast hoặc unicast)              ║
║                                                                  ║
║  5. Client nhận OFFER → REQUEST → ACK (qua relay)              ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Cấu hình Relay Agent

```bash
# Cisco Router:
interface GigabitEthernet0/0
  ip address 192.168.10.1 255.255.255.0
  ip helper-address 10.0.0.100    ← DHCP server IP

# Linux (dhcp-relay):
$ sudo dhcrelay -i eth0 10.0.0.100

# Hoặc trên systemd:
$ cat /etc/default/isc-dhcp-relay
SERVERS="10.0.0.100"
INTERFACES="eth0 eth1"
```

### Trong AWS

AWS VPC không cần relay agent vì:
- AWS built-in DHCP server tồn tại trong MỌI subnet
- DHCP request được xử lý local (không cần qua router)
- **Tuy nhiên**: Nếu bạn dùng custom DHCP (on-premises), Transit Gateway/VPN thì cần relay

---

## 7. DHCP Security — Tấn công và phòng thủ

### Mini example: Kẻ giả mạo lễ tân

Tưởng tượng ai đó mặc vest, đứng ở sảnh khách sạn giả làm lễ tân:
- Khách hỏi: "Phòng tôi ở đâu?" (DHCP Discover)
- Kẻ giả mạo trả lời: "Phòng 666, WiFi password: evil123" (Rogue DHCP)
→ Khách bị dẫn vào phòng sai, WiFi bị giám sát!

### Tấn công DHCP phổ biến

**1. Rogue DHCP Server (DHCP Spoofing):**
```
Attacker cài DHCP server trên laptop:
- Cấp IP bình thường
- NHƯNG gateway = IP của attacker!
→ Mọi traffic đi qua attacker (MITM)
→ Attacker đọc tất cả (passwords, cookies...)

Hoặc:
- DNS = DNS của attacker
→ "google.com" → IP giả → phishing site!
```

**2. DHCP Starvation (DoS):**
```
Attacker gửi hàng ngàn DHCP Discover với MAC giả
→ Server cấp hết tất cả IP trong pool
→ User thật không lấy được IP → DoS!

Tool: gobbler, yersinia, dhcpig
```

**3. DHCP Snooping — Phòng thủ:**
```
╔═══════════════════════════════════════════════════════════════╗
║            DHCP SNOOPING (phòng thủ)                         ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  Concept: Switch phân biệt "trusted" vs "untrusted" ports    ║
║                                                               ║
║  Trusted ports: Nơi DHCP server/uplink kết nối               ║
║    → Allow DHCP OFFER/ACK (server→client messages)           ║
║                                                               ║
║  Untrusted ports: Nơi client/user kết nối                    ║
║    → CHẶN DHCP OFFER/ACK (nếu ai đó cài rogue server)      ║
║    → Chỉ allow DHCP DISCOVER/REQUEST (client messages)       ║
║                                                               ║
║  Bonus: DHCP snooping xây "binding table":                   ║
║    MAC address ↔ IP address ↔ Port ↔ VLAN ↔ Lease           ║
║    → Dùng cho Dynamic ARP Inspection (DAI)                   ║
║    → Dùng cho IP Source Guard                                ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝

Cisco Switch configuration:
  ip dhcp snooping
  ip dhcp snooping vlan 10,20,30
  
  interface GigabitEthernet0/1
    description "Uplink to DHCP Server"
    ip dhcp snooping trust
    
  interface range GigabitEthernet0/2-48
    description "User ports"
    ip dhcp snooping limit rate 10    ← max 10 DHCP/sec (anti-starvation)
    ! untrusted by default
```

### Trong AWS

- **Rogue DHCP**: KHÔNG thể xảy ra trong AWS VPC
  - AWS control plane quản lý DHCP — instances không thể chạy DHCP server
  - VPC networking enforces source/dest check
- **DHCP Starvation**: Không relevant — AWS gán IP theo ENI, không dùng pool
- **Security**: DHCP traffic trong VPC được mã hóa và isolated per-tenant

---

## 8. Tình huống thực tế — 3 scenarios chi tiết

### Scenario 1: Tại nhà — "Internet mất sau khi đổi router"

**Tình huống**: Bạn mua router mới, thay router cũ. Laptop kết nối WiFi OK nhưng không ra được Internet.

**Phân tích DHCP:**

```
1. Kiểm tra IP hiện tại:
   $ ip addr show wlan0
   → IP: 192.168.0.105/24 (từ router CŨ!)

2. Vấn đề: Router MỚI dùng subnet 192.168.1.0/24
   Nhưng laptop vẫn giữ lease CŨ (192.168.0.105)
   → Laptop ở subnet .0, Gateway ở subnet .1 → KHÔNG ROUTE ĐƯỢC!

3. Tại sao laptop không lấy IP mới?
   → Lease chưa hết! Laptop nghĩ IP cũ vẫn OK
   → Laptop gửi DHCP REQUEST cho IP cũ (.0.105)
   → Router mới: "IP .0.105 không thuộc pool tôi!"
   → Gửi DHCP NAK
   → MỘT SỐ OS bỏ qua NAK và giữ IP cũ (bug!)

4. Fix:
   $ sudo dhclient -r wlan0      # Release IP cũ
   $ sudo dhclient wlan0          # Request IP mới
   # Hoặc đơn giản: tắt/bật WiFi
   # Hoặc: "Forget network" → connect lại
```

### Scenario 2: Trong công ty — DHCP Pool exhaustion

**Tình huống**: 8:30 sáng thứ Hai, 50 nhân viên không lấy được IP. Error: "DHCP: No address available".

**Phân tích:**

```
1. Kiểm tra pool:
   DHCP server pool: 192.168.1.100 — 192.168.1.200 (101 IPs)
   Nhân viên: 80 người × (laptop + phone) = 160 devices
   
   101 IPs < 160 devices → POOL HẾT!

2. Tại sao trước đây OK?
   - Trước: 60 người, 100 devices < 101 IPs
   - Mới đây: Tuyển 20 người mới + IoT devices (camera, printer)
   - Cuối tuần: Lease 24h → không expire → vẫn giữ IP!

3. Giải pháp ngắn hạn:
   - Mở rộng pool: .100 — .250 (151 IPs)
   - Giảm lease time: 24h → 8h (IP trả lại sau giờ làm)
   - Clean up static reservations không còn dùng

4. Giải pháp dài hạn:
   - Chia VLAN: Users /23 (510 IPs), IoT /24, Guests /24
   - DHCP pool per VLAN
   - Monitor pool utilization → alert khi > 80%
```

### Scenario 3: Data Center — PXE Boot 500 servers

**Tình huống**: Triển khai 500 servers mới trong data center. Cần cài OS tự động.

**Design DHCP cho PXE boot:**

```
Architecture:
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ DHCP Server  │     │  TFTP Server │     │  HTTP Server │
│ 10.0.0.10    │     │  10.0.0.11   │     │  10.0.0.12   │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                     │                     │
═══════╧═════════════════════╧═════════════════════╧═══════
                    Management Network 10.0.0.0/24
                              │
                    ┌─────────┴─────────┐
                    │  ToR Switch       │
                    └─────────┬─────────┘
       ┌──────────┬───────────┼──────────┬──────────┐
       │          │           │          │          │
    [Srv 1]   [Srv 2]    [Srv 3]   ...         [Srv 500]

DHCP Config:
subnet 10.0.0.0 netmask 255.255.255.0 {
  range 10.0.0.100 10.0.0.250;      # temporary PXE IPs
  next-server 10.0.0.11;             # TFTP server (Option 66)
  filename "pxelinux.0";             # Boot file (Option 67)
  
  option routers 10.0.0.1;
  option domain-name-servers 10.0.0.10;
  default-lease-time 3600;           # 1h — short for provisioning
}

# Sau khi OS cài xong → assign IP production thông qua reservation:
host server-001 {
  hardware ethernet 00:11:22:33:44:01;
  fixed-address 10.1.1.1;
}
# ... 500 reservations

PXE Boot Flow:
1. Server bật → NIC PXE ROM gửi DHCP Discover (Option 60: "PXEClient")
2. DHCP reply: IP=10.0.0.101, TFTP=10.0.0.11, File=pxelinux.0
3. Server download pxelinux.0 qua TFTP
4. pxelinux → download kernel + initrd
5. Kernel → download kickstart/preseed file qua HTTP
6. Automatic OS installation → 30-45 phút
7. Reboot → lấy production IP từ DHCP reservation
```

---

## 9. Bài tập thực hành

### Bài tập 1: Capture DORA Process

```bash
# Terminal 1: Capture DHCP packets
$ sudo tcpdump -i eth0 -v port 67 or port 68

# Terminal 2: Trigger DHCP
$ sudo dhclient -r eth0 && sudo dhclient eth0

# Quan sát 4 packets DORA:
# 1. DISCOVER: 0.0.0.0.68 > 255.255.255.255.67
# 2. OFFER: server.67 > 255.255.255.255.68
# 3. REQUEST: 0.0.0.0.68 > 255.255.255.255.67
# 4. ACK: server.67 > 255.255.255.255.68

# Câu hỏi:
# - xid (Transaction ID) giống nhau ở cả 4 packets?
# - yiaddr (Your IP) xuất hiện từ packet nào?
# - Lease time bao nhiêu giây?
```

### Bài tập 2: Dựng DHCP Server trên Linux

```bash
# Cài isc-dhcp-server
$ sudo apt install isc-dhcp-server

# Cấu hình /etc/dhcp/dhcpd.conf:
$ sudo cat > /etc/dhcp/dhcpd.conf << 'EOF'
# Global options
option domain-name "lab.local";
option domain-name-servers 8.8.8.8, 8.8.4.4;
default-lease-time 3600;    # 1 hour
max-lease-time 7200;        # 2 hours

# Subnet declaration
subnet 192.168.100.0 netmask 255.255.255.0 {
  range 192.168.100.50 192.168.100.150;
  option routers 192.168.100.1;
  option broadcast-address 192.168.100.255;
}

# Static reservation for printer
host office-printer {
  hardware ethernet AA:BB:CC:DD:EE:FF;
  fixed-address 192.168.100.10;
}
EOF

# Chỉ định interface
$ echo 'INTERFACESv4="eth0"' | sudo tee /etc/default/isc-dhcp-server

# Start service
$ sudo systemctl start isc-dhcp-server
$ sudo systemctl status isc-dhcp-server

# Monitor leases
$ watch -n 5 'cat /var/lib/dhcp/dhcpd.leases'
```

### Bài tập 3: DHCP Troubleshooting

```bash
# Scenario: Client không nhận được IP

# Step 1: Kiểm tra client gửi Discover
$ sudo tcpdump -i eth0 -c 10 port 67 or port 68
# Nếu KHÔNG thấy Discover → client interface problem

# Step 2: Kiểm tra server nhận được
$ sudo journalctl -u isc-dhcp-server -f
# "DHCPDISCOVER from XX:XX:XX:XX:XX:XX" → server nhận OK

# Step 3: Kiểm tra pool còn IP
$ dhcp-lease-list    # hoặc
$ grep "lease" /var/lib/dhcp/dhcpd.leases | wc -l

# Step 4: Kiểm tra OFFER đi ra
# Nếu server gửi OFFER nhưng client không nhận:
# → Firewall chặn UDP 68?
# → Switch DHCP snooping block?

# Common fixes:
$ sudo iptables -A INPUT -p udp --dport 67 -j ACCEPT
$ sudo iptables -A INPUT -p udp --dport 68 -j ACCEPT
$ sudo iptables -A OUTPUT -p udp --sport 67 -j ACCEPT
```

### Bài tập 4: AWS DHCP Options Set

```bash
# Tạo custom DHCP Options Set
$ aws ec2 create-dhcp-options \
  --dhcp-configurations \
    "Key=domain-name,Values=mycompany.internal" \
    "Key=domain-name-servers,Values=10.0.0.2,AmazonProvidedDNS" \
    "Key=ntp-servers,Values=169.254.169.123"

# Associate với VPC
$ aws ec2 associate-dhcp-options \
  --dhcp-options-id dopt-0abc123def \
  --vpc-id vpc-0abc123def

# Verify trên EC2 instance:
$ cat /etc/resolv.conf
# nameserver 10.0.0.2
# search mycompany.internal

# Lưu ý: Instances cần reboot hoặc renew DHCP để nhận config mới
$ sudo dhclient -r eth0 && sudo dhclient eth0
```

### Bài tập 5: Detect Rogue DHCP Server

```bash
# Dùng nmap để scan DHCP servers trên network
$ sudo nmap --script broadcast-dhcp-discover

# Hoặc dùng dhcping
$ sudo dhcping -s 255.255.255.255 -c 192.168.1.100

# Python script đơn giản:
$ python3 << 'EOF'
from scapy.all import *

# Send DHCP Discover, xem ai reply
discover = (Ether(dst="ff:ff:ff:ff:ff:ff") /
           IP(src="0.0.0.0", dst="255.255.255.255") /
           UDP(sport=68, dport=67) /
           BOOTP(chaddr=RandMAC()) /
           DHCP(options=[("message-type","discover"),"end"]))

# Gửi và bắt replies
ans, unans = srp(discover, timeout=5, verbose=0)

for snd, rcv in ans:
    server_ip = rcv[IP].src
    offered_ip = rcv[BOOTP].yiaddr
    print(f"DHCP Server found: {server_ip}, offering {offered_ip}")
    
# Nếu thấy > 1 server → có thể có ROGUE!
EOF
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Key Points — Những điểm cần nhớ

```
╔══════════════════════════════════════════════════════════════════╗
║                    DHCP — TÓM TẮT                               ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. DHCP = "Lễ tân mạng" — tự động cấp IP + config             ║
║  2. Dùng UDP: Server port 67, Client port 68                   ║
║  3. DORA: Discover → Offer → Request → Acknowledge             ║
║  4. REQUEST là BROADCAST (để server khác biết withdraw offer)  ║
║  5. Lease Time: T1=50% (renew), T2=87.5% (rebind), T=expire   ║
║  6. Relay Agent: Chuyển broadcast→unicast qua subnet            ║
║  7. giaddr field: Cho server biết client ở subnet nào          ║
║  8. DHCP Snooping: Trusted/Untrusted ports chống rogue server  ║
║  9. Options: Subnet mask(1), Gateway(3), DNS(6), Lease(51)     ║
║  10. PXE Boot: Option 66 (TFTP) + Option 67 (filename)         ║
║                                                                  ║
║  Security:                                                       ║
║  • Rogue DHCP Server → MITM, DNS hijack                        ║
║  • DHCP Starvation → DoS                                        ║
║  • Defense: DHCP Snooping + Rate Limiting                       ║
║                                                                  ║
║  AWS Context:                                                    ║
║  • VPC built-in DHCP (cannot disable)                           ║
║  • DHCP Options Set: custom DNS, domain, NTP                    ║
║  • AWS reserves 5 IPs per subnet                                ║
║  • EC2 IP assigned by control plane, not pool                   ║
║  • No rogue DHCP risk in VPC (controlled by AWS)               ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Tài liệu đọc thêm

| Tài liệu | Link/Reference | Nội dung |
|-----------|---------------|----------|
| RFC 2131 | tools.ietf.org/html/rfc2131 | DHCP Specification |
| RFC 2132 | tools.ietf.org/html/rfc2132 | DHCP Options |
| RFC 3046 | tools.ietf.org/html/rfc3046 | Relay Agent Info (Option 82) |
| RFC 3442 | tools.ietf.org/html/rfc3442 | Classless Static Routes Option |
| RFC 8415 | tools.ietf.org/html/rfc8415 | DHCPv6 |
| Stevens — TCP/IP Illustrated | Chapter 16 | BOOTP & DHCP |
| AWS DHCP Docs | docs.aws.amazon.com/vpc/latest/userguide/VPC_DHCP_Options.html | VPC DHCP |

---

*Bài tiếp theo: [NAT Deep Dive](/2026-06-01-nat-deep-dive) — Network Address Translation — Cánh cổng giữa mạng nội bộ và Internet*
