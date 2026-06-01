---
layout: post
title: "IPv6 Fundamentals — Tại sao Internet cần địa chỉ 128-bit và cách nó thay đổi mọi thứ"
subtitle: "Từ cạn kiệt IPv4 đến kiến trúc IPv6 — hiểu address types, header mới, SLAAC, NDP và dual-stack trong AWS"
tags: [networking, ipv6, layer3, addressing, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
comments: true
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 8200 | Internet Protocol, Version 6 (IPv6) Specification (2017, thay thế RFC 2460) |
| RFC 4291 | IP Version 6 Addressing Architecture |
| RFC 4862 | IPv6 Stateless Address Autoconfiguration (SLAAC) |
| RFC 4861 | Neighbor Discovery for IPv6 (NDP) |
| RFC 6052 | IPv6 Addressing of IPv4/IPv6 Translators |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 5: The Network Layer |
| Kurose & Ross — Computer Networking, 8th Ed. | Chapter 4: Network Layer Data Plane |
| Cisco IPv6 Configuration Guide | IPv6 Addressing and Basic Connectivity |
| AWS Documentation | VPC — IPv6 Support |

---

## 1. Giới thiệu — Tại sao cần biết IPv6?

### Ví dụ đời thường: Khi số nhà trong thành phố hết

Hãy tưởng tượng bạn sống trong một thành phố nhỏ nơi **mỗi ngôi nhà có 1 số nhà** duy nhất. Khi thành phố chỉ có 1000 ngôi nhà, đánh số từ 1 đến 1000 là đủ. Nhưng thành phố phát triển — 10,000 ngôi nhà, 100,000, 1 triệu... Đến lúc nào đó, **hệ thống đánh số cũ không đủ nữa**.

Chính quyền có hai lựa chọn:
1. **Chia sẻ số nhà** (NAT) — nhiều hộ dùng chung 1 số, phân biệt bằng "phòng A, phòng B" → phức tạp, gây nhầm lẫn khi giao hàng
2. **Đổi hệ thống đánh số mới** (IPv6) — chuyển từ số 4 chữ số sang số 39 chữ số → đủ cho hàng nghìn tỷ tỷ ngôi nhà

Internet đang ở tình huống này. IPv4 có khoảng **4.3 tỷ** địa chỉ. Nghe nhiều nhưng thực tế đã **hết sạch từ 2011** (IANA) và **2019** (tất cả RIR). Thế giới có 8 tỷ người, hàng chục tỷ thiết bị IoT, smartphone, laptop, server... IPv4 đơn giản là không đủ.

### Concrete scenario: Khi thiết bị IoT bùng nổ

Hãy tưởng tượng bạn đang quản lý một nhà máy thông minh (Smart Factory):
- 500 camera an ninh → mỗi cái cần 1 IP
- 2000 cảm biến nhiệt độ, độ ẩm → mỗi cái cần 1 IP
- 300 robot → mỗi cái cần 1 IP
- 1000 thiết bị công nhân (tablet, kính AR) → mỗi cái cần 1 IP

Tổng: 3800 IPs cho MỘT nhà máy. Nhân với 1000 nhà máy trên toàn quốc = 3.8 triệu IPs.

Với IPv4 và NAT: Phức tạp, khó quản lý, camera A không thể trực tiếp kết nối đến camera B.
Với IPv6: Mỗi cảm biến nhỏ nhất cũng có địa chỉ global duy nhất — kết nối trực tiếp end-to-end.

### Vấn đề IPv6 giải quyết

| Vấn đề với IPv4 | Giải pháp IPv6 |
|-----------------|----------------|
| Cạn kiệt địa chỉ (4.3 tỷ) | 340 undecillion (3.4 × 10^38) addresses |
| NAT phá vỡ end-to-end connectivity | Mỗi device có global address, không cần NAT |
| Header phức tạp (variable, 13 fields) | Header đơn giản (fixed 40 bytes, 8 fields) |
| ARP broadcast gây flood | NDP (Neighbor Discovery) dùng multicast |
| Cần DHCP server cho mọi subnet | SLAAC — thiết bị tự cấu hình address |
| Fragmentation gây chậm | Chỉ source mới fragment (router không fragment) |
| Không có security built-in | IPsec mandatory (dù thực tế optional) |

---

## 2. IPv6 là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**IPv6** (Internet Protocol version 6) là **phiên bản mới** của hệ thống đánh địa chỉ Internet. Nó thay thế IPv4 (phiên bản 4) với:

- **Địa chỉ dài hơn**: 128 bits thay vì 32 bits
- **Nhiều địa chỉ hơn**: Đủ cấp cho mỗi hạt cát trên Trái Đất một nghìn tỷ địa chỉ
- **Thông minh hơn**: Thiết bị tự biết cách lấy địa chỉ, không cần "xin" server

### Analogy: Từ số điện thoại 7 số đến 10 số

Nhớ hồi xưa số điện thoại cố định chỉ có **7 chữ số** (ví dụ: 8234567)? Khi số lượng thuê bao tăng, Việt Nam phải chuyển sang **10 số** (ví dụ: 028-8234567), rồi di động thành **10 số** (09xx-xxx-xxxx). Mỗi lần tăng chữ số = gấp 10 lần capacity.

IPv6 làm điều tương tự nhưng **mạnh hơn gấp tỷ lần**:

```
IPv4: 32 bits  = 4,294,967,296 addresses (~4.3 tỷ)
IPv6: 128 bits = 340,282,366,920,938,463,463,374,607,431,768,211,456 addresses

So sánh trực quan:
- IPv4: 4.3 × 10^9  (4.3 tỷ) — ít hơn số người trên Trái Đất
- IPv6: 3.4 × 10^38 — nhiều hơn số nguyên tử trong vũ trụ quan sát được

Nếu chia đều:
- IPv4: 0.5 address/người
- IPv6: 4.8 × 10^28 addresses/người (48 nghìn tỷ tỷ tỷ!)
```

### Format địa chỉ IPv6

```
IPv4: 192.168.1.100          (4 nhóm số thập phân, ngăn bởi dấu .)
IPv6: 2001:0db8:85a3:0000:0000:8a2e:0370:7334 (8 nhóm hex, ngăn bởi :)

Rút gọn IPv6:
1. Bỏ leading zeros: 2001:db8:85a3:0:0:8a2e:370:7334
2. Thay nhóm 0 liên tiếp bằng :: (chỉ dùng 1 lần):
   2001:db8:85a3::8a2e:370:7334

Ví dụ khác:
- Full:    fe80:0000:0000:0000:021c:c0ff:fe9d:5c45
- Rút gọn: fe80::21c:c0ff:fe9d:5c45

- Full:    0000:0000:0000:0000:0000:0000:0000:0001
- Rút gọn: ::1 (loopback address — tương đương 127.0.0.1)
```

---

## 3. IPv6 Header — Đơn giản nhưng mạnh mẽ hơn

### Mini example: Hộ chiếu mới vs hộ chiếu cũ

Hãy nghĩ IPv4 header như hộ chiếu cũ với nhiều trang phức tạp (visa, stamp, ghi chú...), còn IPv6 header như **hộ chiếu sinh trắc học mới** — gọn hơn, nhanh đọc hơn, nhưng thông tin quan trọng đầy đủ. Nếu cần thêm thông tin đặc biệt → dùng trang phụ (Extension Headers).

### Cấu trúc IPv6 Header (RFC 8200)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                         Source Address                         |
+                          (128 bits)                            +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                      Destination Address                      |
+                          (128 bits)                            +
|                                                               |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

**Fixed size: 40 bytes** — không có IHL, không có options trong header chính.

### So sánh từng trường với IPv4

| IPv6 Field | Size | Tương đương IPv4 | Khác biệt |
|-----------|------|-------------------|-----------|
| Version | 4 bits | Version | = 6 thay vì 4 |
| Traffic Class | 8 bits | Type of Service | Giống: DSCP + ECN |
| Flow Label | 20 bits | ❌ Không có | MỚI: nhận dạng flow cho QoS |
| Payload Length | 16 bits | Total Length | Chỉ đếm payload (không bao gồm header) |
| Next Header | 8 bits | Protocol | Mở rộng: chỉ đến extension header hoặc upper layer |
| Hop Limit | 8 bits | TTL | Đổi tên cho chính xác (giảm mỗi hop, không liên quan thời gian) |
| Source Address | 128 bits | Source IP (32 bits) | Gấp 4 lần! |
| Destination Address | 128 bits | Dest IP (32 bits) | Gấp 4 lần! |

### Những gì IPv6 BỎ (so với IPv4)

| Trường bị bỏ | Lý do |
|--------------|--------|
| IHL (Header Length) | Header LUÔN 40 bytes → không cần field này |
| Identification, Flags, Fragment Offset | Router KHÔNG fragment → chuyển sang extension header |
| Header Checksum | Layer 2 (Ethernet CRC) và Layer 4 (TCP/UDP checksum) đã kiểm tra → bỏ để tăng tốc router |
| Options + Padding | Thay bằng Extension Headers — linh hoạt hơn |

### Flow Label — Điểm mới quan trọng

**Flow Label (20 bits)** là trường HOÀN TOÀN MỚI trong IPv6, không có trong IPv4:

```
Mục đích: Nhận dạng một "luồng" packets thuộc cùng kết nối

Ví dụ: Bạn đang gọi video Zoom
- Mọi packets của cuộc gọi này có CÙNG Flow Label = 0xA1B2C
- Router thấy Flow Label → biết ngay thuộc flow nào
- Không cần đọc TCP/UDP port → xử lý nhanh hơn
- Tất cả packets cùng flow đi CÙNG đường → tránh out-of-order
```

### Trong thực tế

```bash
# Xem IPv6 header bằng tcpdump
$ sudo tcpdump -i eth0 ip6 -c 5 -v
# Output: flowlabel 0x12345, hlim 64, next-header TCP (6)

# Kiểm tra IPv6 connectivity
$ ping6 ::1           # Ping loopback IPv6
$ ping6 google.com    # Ping Google qua IPv6

# Xem IPv6 address trên interface
$ ip -6 addr show
```

### Trong AWS

- **VPC dual-stack**: Mỗi VPC có thể có cả IPv4 CIDR và IPv6 CIDR
- **IPv6 CIDR**: AWS cấp /56 cho mỗi VPC (256 subnets, mỗi subnet /64)
- **EC2**: Mỗi instance có thể có cả IPv4 private + IPv6 global address
- **ALB/NLB**: Hỗ trợ dual-stack listeners
- **Route 53**: Hỗ trợ AAAA records (IPv6)

---

## 4. IPv6 Address Types — Các loại địa chỉ

### Mini example: Các cách gọi tên trong xã hội

Mỗi người có nhiều cách để được nhận dạng:
- **Tên khai sinh** (giống Global Unicast) — duy nhất trên toàn thế giới
- **Biệt danh trong nhà** (giống Link-Local) — chỉ gia đình biết
- **Chức vụ** (giống Anycast) — "Trưởng phòng" → ai đang trực thì trả lời
- **"Mọi người ơi!"** (giống Multicast) — gọi nhiều người cùng lúc

### Global Unicast Address (GUA) — "Tên khai sinh"

```
Prefix: 2000::/3 (bắt đầu bằng 2 hoặc 3)
Tương đương IPv4: Public IP address

Cấu trúc:
┌──────────────────┬────────────────┬────────────────────────┐
│  Global Routing  │   Subnet ID    │    Interface ID         │
│  Prefix (48 bits)│   (16 bits)    │    (64 bits)           │
└──────────────────┴────────────────┴────────────────────────┘

Ví dụ: 2001:0db8:1234:0001:0000:0000:0000:0100
       ├─── 48 bits ────┤─16─┤────── 64 bits ──────────┤
       Global Prefix     Sub   Interface ID

Giải thích:
- Global Routing Prefix (48 bits): ISP cấp cho bạn — giống "số nhà + đường + quận"
- Subnet ID (16 bits): Bạn tự chia subnet — giống "tầng + phòng"
- Interface ID (64 bits): Nhận dạng thiết bị — giống "tên người trong phòng"
```

### Link-Local Address — "Biệt danh trong nhà"

```
Prefix: fe80::/10 (luôn bắt đầu bằng fe80)
Scope: Chỉ valid trong 1 link (1 subnet/LAN segment)
Tương đương IPv4: 169.254.x.x (APIPA) — nhưng QUAN TRỌNG hơn nhiều!

Đặc điểm:
- TỰ ĐỘNG tạo khi interface bật (không cần server)
- KHÔNG thể route qua router (chỉ local link)
- LUÔN tồn tại trên mọi IPv6 interface
- Dùng cho: NDP, Router Discovery, DHCPv6

Cách tạo Interface ID (EUI-64):
MAC address: 00:1C:C0:9D:5C:45
1. Chia MAC thành 2 nửa: 00:1C:C0 | 9D:5C:45
2. Chèn FF:FE vào giữa: 00:1C:C0:FF:FE:9D:5C:45
3. Flip bit thứ 7: 02:1C:C0:FF:FE:9D:5C:45
4. Link-Local = fe80::021c:c0ff:fe9d:5c45
```

### Unique Local Address (ULA) — "Địa chỉ nội bộ"

```
Prefix: fc00::/7 (thực tế dùng fd00::/8)
Tương đương IPv4: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 (private)

Dùng cho: Mạng nội bộ KHÔNG cần ra Internet
Ví dụ: fd12:3456:789a::1

Khác với IPv4 private: ULA được thiết kế để GLOBALLY UNIQUE
(dùng 40-bit random → xác suất trùng cực thấp khi merge networks)
```

### Multicast Address — "Gọi nhóm"

```
Prefix: ff00::/8 (bắt đầu bằng ff)
Tương đương IPv4: 224.0.0.0/4 (multicast) + 255.255.255.255 (broadcast)
LƯU Ý: IPv6 KHÔNG CÓ BROADCAST! Mọi "broadcast" đều dùng multicast.

Format: ff[flags][scope]::[group ID]

Scope values:
  1 = Interface-local (loopback)
  2 = Link-local (same LAN)
  5 = Site-local
  8 = Organization-local
  E = Global

Multicast addresses quan trọng:
  ff02::1 = All nodes (link-local) — thay thế broadcast
  ff02::2 = All routers (link-local)
  ff02::1:ff[last 24 bits] = Solicited-node multicast (cho NDP)
```

### Anycast Address — "Ai gần nhất trả lời"

```
Đặc biệt: Anycast KHÔNG có prefix riêng
Một Global Unicast address có thể được assign cho NHIỀU interface
→ Packet đến anycast address → router chọn interface GẦN NHẤT

Ví dụ thực tế: DNS anycast
- Google DNS: 2001:4860:4860::8888
- IP này được assign cho HÀNG TRĂM server trên toàn thế giới
- Khi bạn query → đến server gần bạn nhất
```

### Bảng tổng hợp

| Loại | Prefix | Scope | Tương đương IPv4 |
|------|--------|-------|------------------|
| Global Unicast | 2000::/3 | Global (Internet) | Public IP |
| Link-Local | fe80::/10 | 1 link (LAN) | 169.254.x.x (APIPA) |
| Unique Local | fc00::/7 | Organization | 10.x, 172.16.x, 192.168.x |
| Multicast | ff00::/8 | Varies (scope field) | 224.0.0.0/4 + broadcast |
| Loopback | ::1 | Host only | 127.0.0.1 |
| Unspecified | :: | N/A | 0.0.0.0 |
| IPv4-mapped | ::ffff:0:0/96 | N/A | Dùng trong dual-stack |

### Trong AWS

- **VPC IPv6 CIDR**: AWS cấp từ Amazon pool hoặc bạn BYOIP
- Mỗi subnet nhận **/64 prefix** (standard cho IPv6 — đủ 2^64 addresses)
- **EC2 instances**: Nhận GUA (Global Unicast) từ subnet prefix
- **Security Groups**: Hỗ trợ IPv6 rules (ví dụ: `::/0` = all IPv6 traffic)

---

## 5. SLAAC — Tự cấu hình địa chỉ không cần server

### Mini example: Nhân viên mới tự tìm bàn ngồi

Hãy tưởng tượng một văn phòng co-working nơi:
1. Bạn bước vào → thấy bảng thông tin: "Tầng 3, khu A, bàn trống tự ngồi"
2. Bạn TỰ CHỌN bàn (dựa trên tên bạn — Interface ID)
3. Bạn hỏi to: "Có ai ngồi bàn này chưa?" (DAD — Duplicate Address Detection)
4. Không ai trả lời → Bạn ngồi xuống!

Đó là SLAAC! Không cần lễ tân (DHCP server) cấp chỗ.

### SLAAC — Stateless Address AutoConfiguration (RFC 4862)

```
╔═══════════════════════════════════════════════════════════════════╗
║              QUY TRÌNH SLAAC (6 bước)                            ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  Bước 1: Host tạo Link-Local address (fe80::...)                 ║
║          Dùng EUI-64 hoặc random Interface ID                    ║
║                                                                   ║
║  Bước 2: DAD (Duplicate Address Detection)                       ║
║          Gửi Neighbor Solicitation đến ff02::1:ff[last 24 bits]  ║
║          Nếu KHÔNG ai reply → address OK                         ║
║                                                                   ║
║  Bước 3: Router Solicitation (RS)                                ║
║          Host gửi RS đến ff02::2 (all routers)                   ║
║          "Có router nào ở đây không?"                            ║
║                                                                   ║
║  Bước 4: Router Advertisement (RA)                               ║
║          Router reply: "Prefix = 2001:db8:1::/64,                ║
║                         Default Gateway = fe80::1,                ║
║                         DNS = 2001:4860:4860::8888"              ║
║                                                                   ║
║  Bước 5: Host tạo Global Unicast Address                        ║
║          = Prefix từ RA + Interface ID                           ║
║          = 2001:db8:1::21c:c0ff:fe9d:5c45                       ║
║                                                                   ║
║  Bước 6: DAD lần 2 cho Global Address                           ║
║          Verify Global Address chưa ai dùng                      ║
║                                                                   ║
║  DONE! Host có IPv6 address mà KHÔNG cần DHCP server!           ║
║                                                                   ║
╚═══════════════════════════════════════════════════════════════════╝
```

### So sánh SLAAC vs DHCPv6 vs DHCPv4

| Đặc điểm | SLAAC | DHCPv6 (Stateful) | DHCPv4 |
|-----------|-------|-------------------|--------|
| Cần server? | KHÔNG | CÓ | CÓ |
| Router cần? | CÓ (gửi RA) | CÓ (RA + DHCP server) | KHÔNG (relay agent optional) |
| Cung cấp prefix? | CÓ (từ RA) | CÓ | CÓ (subnet mask) |
| Cung cấp DNS? | CÓ (RDNSS option RFC 8106) | CÓ | CÓ |
| Address tracking? | KHÔNG (stateless) | CÓ (server biết ai dùng gì) | CÓ |
| Dùng khi nào? | Home, IoT, simple network | Enterprise cần logging | IPv4 networks |

### Privacy Extensions (RFC 4941)

**Vấn đề**: Nếu Interface ID = EUI-64 (từ MAC address), ai cũng biết MAC address của bạn → **theo dõi được** (tracking).

**Giải pháp**: Privacy Extensions tạo **random Interface ID** thay đổi định kỳ:

```bash
# Kiểm tra privacy extensions trên Linux
$ sysctl net.ipv6.conf.all.use_tempaddr
# 0 = tắt, 1 = prefer public, 2 = prefer temporary (privacy)

# Bật privacy extensions
$ sudo sysctl -w net.ipv6.conf.all.use_tempaddr=2

# Kết quả: bạn sẽ thấy 2 Global Unicast addresses
# 1. Address từ EUI-64 (permanent, cho server connections)
# 2. Address random (temporary, cho outgoing connections — thay đổi mỗi vài giờ)
```

### Trong AWS

- **EC2 instances**: Nhận IPv6 address qua DHCPv6 (AWS managed), KHÔNG dùng SLAAC thuần
- **Tuy nhiên**: Linux instances trên AWS vẫn có link-local address tự tạo
- **ENI (Elastic Network Interface)**: Có thể assign nhiều IPv6 addresses
- **Subnet**: Mỗi subnet /64 có đủ 2^64 addresses — không bao giờ hết

---

## 6. NDP — Neighbor Discovery Protocol (thay thế ARP)

### Mini example: Tìm người trong tòa nhà

Trong IPv4 (ARP): Bạn đứng giữa sảnh hét to: **"AI LÀ NGUYỄN VĂN A?"** → Tất cả mọi người đều nghe (broadcast) → Chỉ Nguyễn Văn A trả lời.

Trong IPv6 (NDP): Bạn gọi điện đến nhóm "Những người tên bắt đầu bằng NguyễnVanA" (solicited-node multicast) → Chỉ Nguyễn Văn A nhận cuộc gọi → Trả lời.

**Khác biệt quan trọng**: ARP broadcast đến TẤT CẢ → lãng phí. NDP multicast chỉ đến NHÓM NHỎ → hiệu quả hơn.

### NDP Messages (ICMPv6 Types 133-137)

| Message | ICMPv6 Type | Viết tắt | Mục đích |
|---------|-------------|----------|----------|
| Router Solicitation | 133 | RS | Host hỏi: "Router ở đâu?" |
| Router Advertisement | 134 | RA | Router trả lời: "Tôi đây, prefix là..." |
| Neighbor Solicitation | 135 | NS | "MAC address của IP X là gì?" (= ARP Request) |
| Neighbor Advertisement | 136 | NA | "MAC của tôi là..." (= ARP Reply) |
| Redirect | 137 | — | Router báo: "Nên gửi qua router khác" |

### Quy trình Neighbor Discovery (thay ARP)

```
Host A muốn gửi packet đến Host B (cùng link):
Host A biết: IPv6 của B = 2001:db8:1::200
Host A KHÔNG biết: MAC address của B

1. Host A tạo Solicited-Node Multicast address cho B:
   Lấy 24 bits cuối của B's IPv6: 00:02:00
   → Multicast: ff02::1:ff00:0200

2. Host A gửi Neighbor Solicitation (NS) đến ff02::1:ff00:0200
   "Ai có IPv6 2001:db8:1::200? Cho tôi MAC address!"

3. CHỈ Host B (subscribe multicast ff02::1:ff00:0200) nhận được

4. Host B reply Neighbor Advertisement (NA):
   "Tôi có! MAC = 00:1C:C0:9D:5C:45"

5. Host A cache trong Neighbor Cache (= ARP table IPv4)
```

### Neighbor Cache (thay ARP Table)

```bash
# Xem Neighbor Cache trên Linux
$ ip -6 neigh show
2001:db8:1::1 dev eth0 lladdr 00:1c:c0:9d:5c:45 REACHABLE
fe80::1       dev eth0 lladdr 00:1c:c0:9d:5c:45 STALE

# States:
# INCOMPLETE — đang chờ NA reply
# REACHABLE — vừa confirm, valid
# STALE — hết timeout, cần re-confirm
# DELAY — đợi upper layer confirm
# PROBE — đang gửi NS để re-confirm
```

### DAD — Duplicate Address Detection

Trước khi dùng bất kỳ IPv6 address nào, host PHẢI kiểm tra:

```
1. Host muốn dùng address X
2. Gửi NS đến solicited-node multicast của X
   Source = :: (unspecified — vì chưa có address)
   "Có ai đang dùng X không?"
3. Nếu nhận NA reply → address X ĐÃ CÓ NGƯỜI DÙNG → thất bại!
4. Nếu timeout (không ai reply) → address X an toàn → sử dụng!
```

### Trong AWS

- **NDP trong VPC**: AWS handles NDP transparently — bạn không cần cấu hình
- **Security Groups**: Tự động cho phép ICMPv6 NDP messages (Type 133-136)
- **VPC subnet**: IPv6 neighbor discovery hoạt động tự động giữa instances cùng subnet
- **Important**: NACLs cần explicit allow ICMPv6 nếu bạn dùng custom rules

---

## 7. Extension Headers và Fragmentation trong IPv6

### Mini example: Tài liệu chính và phụ lục

Khi bạn nộp hồ sơ xin visa, có:
- **Đơn chính** (= IPv6 Base Header) — luôn cùng format, 40 bytes
- **Phụ lục A**: Ảnh passport, bản sao CMND (= Routing Header)
- **Phụ lục B**: Giấy mời từ công ty (= Fragment Header)
- **Phụ lục C**: Kết quả khám sức khỏe (= Authentication Header)

Không phải ai cũng cần tất cả phụ lục — chỉ đính kèm khi cần.

### Extension Headers Chain

```
IPv6 dùng "Next Header" field tạo chuỗi (daisy chain):

┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌─────────┐
│ IPv6 Base    │────▶│  Hop-by-Hop  │────▶│  Routing     │────▶│ TCP     │
│ Header       │     │  Options     │     │  Header      │     │ Header  │
│ Next=0       │     │ Next=43      │     │ Next=6       │     │ + Data  │
└──────────────┘     └──────────────┘     └──────────────┘     └─────────┘

Next Header values:
  0  = Hop-by-Hop Options (phải xử lý ở MỌI router)
  6  = TCP (upper layer)
  17 = UDP (upper layer)
  43 = Routing Header
  44 = Fragment Header
  50 = ESP (IPsec encryption)
  51 = AH (IPsec authentication)
  58 = ICMPv6
  59 = No Next Header (end of chain)
  60 = Destination Options
```

### Thứ tự bắt buộc (RFC 8200)

```
1. IPv6 Base Header
2. Hop-by-Hop Options (nếu có — PHẢI đầu tiên)
3. Destination Options (cho router đích đầu tiên)
4. Routing Header
5. Fragment Header
6. Authentication Header (AH)
7. Encapsulating Security Payload (ESP)
8. Destination Options (cho đích cuối cùng)
9. Upper-layer header (TCP/UDP/ICMPv6)
```

### Fragmentation trong IPv6 — CHỈ Source mới fragment!

**Khác biệt lớn nhất với IPv4**: Router IPv6 **KHÔNG BAO GIỜ** fragment packet. Chỉ source host mới fragment.

```
IPv4: Source gửi 4000 bytes → Router gặp link MTU 1280 → Router FRAGMENT
IPv6: Source gửi 4000 bytes → Router gặp link MTU 1280 → Router DROP
      + gửi ICMPv6 Packet Too Big (Type 2) về source
      → Source nhận, biết MTU = 1280, tự FRAGMENT rồi gửi lại

Tại sao thiết kế này?
1. Router không tốn resource fragment → forwarding NHANH hơn
2. Fragment overhead chỉ ở source/destination → KHÔNG ảnh hưởng core
3. Path MTU Discovery trở thành MANDATORY (không optional như IPv4)
```

**IPv6 Minimum MTU**: 1280 bytes (IPv4 minimum = 576 bytes). Mọi link phải hỗ trợ ít nhất 1280 bytes.

### Fragment Header Format

```
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Next Header  |   Reserved    |      Fragment Offset    |Res|M|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                         Identification                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

- Fragment Offset: 13 bits (đơn vị 8 bytes — giống IPv4)
- M flag: 1 = More Fragments, 0 = Last Fragment
- Identification: 32 bits (gấp đôi IPv4's 16 bits!)
```

### Trong thực tế

```bash
# Kiểm tra Path MTU đến đích (IPv6)
$ ping6 -M do -s 1452 google.com
# 1452 + 48 (IPv6+ICMPv6 header) = 1500

# Xem extension headers bằng tcpdump
$ sudo tcpdump -i eth0 ip6 -v -c 10
# Sẽ thấy: next-header TCP (6), next-header Fragment (44), etc.

# Kiểm tra fragment reassembly statistics
$ cat /proc/net/snmp6 | grep Frag
Ip6FragOKs        0
Ip6FragFails      0
Ip6FragCreates    0
Ip6ReasmTimeout   0
Ip6ReasmReqds     0
Ip6ReasmOKs       0
Ip6ReasmFails     0
```

### Trong AWS

- **VPC MTU**: 1500 bytes cho IPv6 (giống IPv4)
- **Jumbo frames**: 9001 bytes supported cho cả IPv4 và IPv6 trong VPC
- **Path MTU Discovery**: AWS recommend sử dụng cho cả IPv4 và IPv6
- **ICMPv6 Packet Too Big**: Cần allow trong NACL (nếu custom) để PMTUD hoạt động
- **Transit Gateway**: Xử lý IPv6 packets với extension headers bình thường

---

## 8. Tình huống thực tế — 3 scenarios chi tiết

### Scenario 1: Tại nhà — Bật IPv6 cho mạng gia đình

**Tình huống**: Bạn vừa được ISP cấp IPv6 prefix. Muốn tất cả thiết bị trong nhà có IPv6.

**Từng bước:**

```
1. Kiểm tra ISP đã cấp IPv6 prefix:
   $ ip -6 addr show dev wan0
   → Thấy: 2402:800:6111:xxxx::/64 (GUA từ ISP)

2. Router nhà (OpenWrt/pfsense) nhận prefix delegation /56 từ ISP
   → Tự chia /64 cho mỗi LAN subnet

3. Router gửi RA (Router Advertisement) vào LAN:
   - Prefix: 2402:800:6111:1::/64
   - Default route: fe80::1 (link-local của router)
   - DNS: 2001:4860:4860::8888 (Google IPv6 DNS)

4. Laptop nhận RA → SLAAC tạo address:
   2402:800:6111:1:a1b2:c3d4:e5f6:7890 (random, privacy)

5. Verify:
   $ ping6 google.com
   $ curl -6 https://test-ipv6.com/
   Score: 10/10!
```

**Kết quả**: Mọi thiết bị có IPv6 GLOBAL address — camera IP, TV, điện thoại — tất cả reachable từ Internet (nhưng firewall vẫn bảo vệ!).

### Scenario 2: Trong công ty — Triển khai dual-stack cho 1000 servers

**Tình huống**: Công ty e-commerce quyết định hỗ trợ IPv6 cho customers. Cần dual-stack cho web servers.

**Kế hoạch:**

```
Phase 1: Planning
- Nhận IPv6 /48 từ ISP (65,536 subnets)
- Thiết kế addressing: 2001:db8:abcd:0001::/64 cho web servers
                       2001:db8:abcd:0002::/64 cho DB servers
                       2001:db8:abcd:0003::/64 cho management

Phase 2: Infrastructure
- Bật IPv6 trên core routers (Cisco: "ipv6 unicast-routing")
- Cấu hình dual-stack trên interfaces
- Update firewall rules cho IPv6 (STATEFUL!)
- Cấu hình DNS: thêm AAAA records cho web domains

Phase 3: Servers
- Web servers: bind CŨNG listen IPv6
  nginx: listen [::]:443 ssl;
- Load Balancer: thêm IPv6 VIP
- Monitoring: thêm IPv6 health checks

Phase 4: Testing
- Verify từ IPv6-only network
- Test Path MTU Discovery
- Check NDP operation
- Performance comparison IPv4 vs IPv6
```

### Scenario 3: ISP — Chuyển đổi 1 triệu khách hàng sang IPv6

**Tình huống**: ISP tại Việt Nam cần triển khai IPv6 cho customers theo yêu cầu của Bộ TT&TT.

**Chiến lược: Dual-stack + 464XLAT**

```
Giai đoạn 1: Dual-stack (hiện tại)
- Cấp cả IPv4 + IPv6 cho mỗi customer
- IPv4: shared (CGN/NAT444) vì hết IPv4
- IPv6: /56 per customer (native, end-to-end)
- Incentive: IPv6 traffic có bandwidth ưu tiên

Giai đoạn 2: IPv6-preferred (2-3 năm)
- Default = IPv6
- IPv4 chỉ cho legacy services (qua NAT64/DNS64)
- Customer CPE hỗ trợ 464XLAT:
  ┌─────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ App IPv4│────▶│CLAT (464)│────▶│IPv6 only │────▶│PLAT(NAT64)│──▶ IPv4 Internet
  │         │     │trên CPE  │     │backbone  │     │tại ISP   │
  └─────────┘     └──────────┘     └──────────┘     └──────────┘

Giai đoạn 3: IPv6-only (5-10 năm)
- IPv4 phased out
- Chỉ giữ NAT64 cho legacy content
```

---

## 9. Bài tập thực hành

### Bài tập 1: Viết đúng IPv6 address

```
Rút gọn các IPv6 address sau:
a) 2001:0db8:0000:0000:0000:0000:0000:0001
   → 2001:db8::1

b) fe80:0000:0000:0000:021c:c0ff:fe9d:5c45
   → fe80::21c:c0ff:fe9d:5c45

c) ff02:0000:0000:0000:0000:0001:ff9d:5c45
   → ff02::1:ff9d:5c45

Mở rộng (expand) các address sau:
d) ::1
   → 0000:0000:0000:0000:0000:0000:0000:0001

e) 2001:db8::ff:fe00:1
   → 2001:0db8:0000:0000:0000:00ff:fe00:0001
```

### Bài tập 2: Xác định loại address

```
Cho các address, xác định type:
a) 2001:db8:1::100      → Global Unicast (2000::/3)
b) fe80::1              → Link-Local (fe80::/10)
c) ff02::1              → Multicast (all nodes, link-local scope)
d) ::1                  → Loopback
e) fd12:3456:789a::1    → Unique Local (fc00::/7)
f) ::ffff:192.168.1.1   → IPv4-mapped IPv6
g) ff05::2              → Multicast (all routers, site-local scope)
```

### Bài tập 3: Cấu hình IPv6 trên Linux

```bash
# 1. Kiểm tra IPv6 đã bật chưa
$ cat /proc/sys/net/ipv6/conf/all/disable_ipv6
# 0 = enabled, 1 = disabled

# 2. Xem tất cả IPv6 addresses
$ ip -6 addr show

# 3. Thêm static IPv6 address
$ sudo ip -6 addr add 2001:db8:1::100/64 dev eth0

# 4. Thêm default route
$ sudo ip -6 route add default via fe80::1 dev eth0

# 5. Test connectivity
$ ping6 -c 3 2001:db8:1::1
$ traceroute6 google.com

# 6. Xem neighbor cache (thay ARP table)
$ ip -6 neigh show

# 7. Xem routing table IPv6
$ ip -6 route show
```

### Bài tập 4: Tính EUI-64 Interface ID

```
MAC address: 00:50:56:C0:00:01

Bước 1: Tách thành 2 nửa
  00:50:56 | C0:00:01

Bước 2: Chèn FF:FE
  00:50:56:FF:FE:C0:00:01

Bước 3: Flip bit thứ 7 (Universal/Local bit)
  Byte đầu: 00 = 0000 0000
  Flip bit 7:    0000 0010 = 02
  → 02:50:56:FF:FE:C0:00:01

Bước 4: Link-Local address
  fe80::0250:56ff:fec0:0001

Verify trên Linux:
$ ip -6 addr show dev eth0 | grep fe80
```

### Bài tập 5: Capture và phân tích NDP

```bash
# Capture NDP messages
$ sudo tcpdump -i eth0 icmp6 -v -c 20

# Hoặc lọc cụ thể:
# Router Solicitation (Type 133)
$ sudo tcpdump -i eth0 'icmp6 and ip6[40] == 133'

# Router Advertisement (Type 134)
$ sudo tcpdump -i eth0 'icmp6 and ip6[40] == 134'

# Neighbor Solicitation (Type 135)
$ sudo tcpdump -i eth0 'icmp6 and ip6[40] == 135'

# Trigger RS bằng cách disable/enable interface
$ sudo ip link set eth0 down
$ sudo ip link set eth0 up
# → Quan sát RS được gửi, và RA reply

# Câu hỏi phân tích:
# 1. RA chứa prefix gì? Lifetime bao lâu?
# 2. NS dùng source address gì? (:: hay link-local?)
# 3. Solicited-node multicast group nào được join?
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Key Points — Những điểm cần nhớ

```
╔══════════════════════════════════════════════════════════════════╗
║                  IPv6 FUNDAMENTALS — TÓM TẮT                    ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. IPv6 = 128-bit address (3.4×10^38 addresses)                ║
║  2. Format: 8 nhóm hex, ngăn bởi : (rút gọn bằng ::)          ║
║  3. Header: Fixed 40 bytes, 8 fields (đơn giản hơn IPv4)       ║
║  4. KHÔNG broadcast — mọi thứ dùng multicast                    ║
║  5. KHÔNG ARP — dùng NDP (Neighbor Discovery Protocol)          ║
║  6. SLAAC: Tự cấu hình address không cần DHCP server           ║
║  7. Router KHÔNG fragment — chỉ source mới fragment             ║
║  8. Extension Headers thay thế IPv4 Options                     ║
║  9. Link-Local (fe80::) = bắt buộc trên mọi interface          ║
║  10. Privacy Extensions: Random Interface ID chống tracking     ║
║                                                                  ║
║  Address Types:                                                  ║
║  • 2000::/3 = Global Unicast (public, routable)                 ║
║  • fe80::/10 = Link-Local (1 link only)                         ║
║  • fc00::/7 = Unique Local (private)                            ║
║  • ff00::/8 = Multicast                                         ║
║  • ::1 = Loopback                                               ║
║                                                                  ║
║  AWS Context:                                                    ║
║  • VPC hỗ trợ dual-stack (IPv4 + IPv6)                          ║
║  • AWS cấp /56 IPv6 per VPC, /64 per subnet                    ║
║  • EC2 instances nhận IPv6 GUA                                   ║
║  • Security Groups / NACLs hỗ trợ IPv6 rules                   ║
║  • ALB/NLB/CloudFront hỗ trợ IPv6                              ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Bảng tham khảo: IPv4 → IPv6 "Dictionary"

| Concept IPv4 | Tương đương IPv6 |
|-------------|-----------------|
| ARP | NDP (Neighbor Discovery) |
| ARP Table | Neighbor Cache |
| Broadcast | Multicast (ff02::1 = all nodes) |
| DHCP | SLAAC + DHCPv6 (optional) |
| ICMP | ICMPv6 |
| IGMP | MLD (Multicast Listener Discovery) |
| NAT | Không cần (end-to-end!) |
| Private IP (10.x) | Unique Local (fc00::/7) |
| TTL | Hop Limit |
| Header Options | Extension Headers |
| Don't Fragment bit | Default behavior (router never fragments) |
| Path MTU Discovery | MANDATORY (not optional) |

### Tài liệu đọc thêm

| Tài liệu | Link/Reference | Nội dung |
|-----------|---------------|----------|
| RFC 8200 | tools.ietf.org/html/rfc8200 | IPv6 Specification (current) |
| RFC 4291 | tools.ietf.org/html/rfc4291 | IPv6 Addressing Architecture |
| RFC 4862 | tools.ietf.org/html/rfc4862 | SLAAC |
| RFC 4861 | tools.ietf.org/html/rfc4861 | Neighbor Discovery |
| RFC 4941 | tools.ietf.org/html/rfc4941 | Privacy Extensions |
| RFC 8106 | tools.ietf.org/html/rfc8106 | DNS via RA (RDNSS) |
| test-ipv6.com | test-ipv6.com | Kiểm tra IPv6 connectivity |
| AWS IPv6 Docs | docs.aws.amazon.com/vpc/latest/userguide/vpc-ipv6.html | VPC IPv6 |

---

*Bài tiếp theo: [ICMP Protocol](/2026-06-01-icmp-protocol) — Hệ thống thông báo lỗi và chẩn đoán mạng*
