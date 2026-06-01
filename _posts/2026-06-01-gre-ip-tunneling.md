---
layout: post
title: "GRE & IP Tunneling - Generic Routing Encapsulation, GRE over IPSec, MTU và Recursive Routing"
date: 2026-06-01
categories: [networking]
tags: [gre, tunneling, ipsec, mtu, encapsulation, vpn]
---

## 1. Giới thiệu — Gửi thư trong thư

Hãy tưởng tượng bạn muốn gửi **một bức thư tiếng Việt** cho bạn ở Nhật, nhưng bưu điện quốc tế **chỉ chấp nhận địa chỉ tiếng Anh**. Bạn làm thế nào?

**Giải pháp đơn giản:**
1. Viết thư tiếng Việt bình thường (payload)
2. Cho vào **phong bì nhỏ** — ghi địa chỉ tiếng Việt (inner header)
3. Bỏ phong bì nhỏ vào **phong bì lớn** — ghi địa chỉ tiếng Anh (outer header)
4. Bưu điện quốc tế đọc phong bì lớn → chuyển đến Nhật
5. Bưu điện Nhật mở phong bì lớn → thấy phong bì nhỏ → giao cho người nhận
6. Người nhận mở phong bì nhỏ → đọc thư tiếng Việt!

**GRE (Generic Routing Encapsulation)** hoạt động y hệt:
- **Thư tiếng Việt** = Original packet (có thể IPv4, IPv6, IPX, bất kỳ protocol nào)
- **Phong bì nhỏ** = GRE header (giữ thông tin về packet gốc)
- **Phong bì lớn** = Outer IP header (để vận chuyển qua Internet)
- **Bưu điện quốc tế** = Internet routers (chỉ đọc outer IP)
- **Bưu điện đích** = Tunnel endpoint (mở GRE → lấy packet gốc)

### Tại sao cần tunneling?

| Vấn đề | Tunneling giải quyết |
|---|---|
| Mạng nội bộ dùng IP private | Đóng gói private IP trong public IP để đi qua Internet |
| Cần chạy IPv6 qua mạng IPv4 | GRE encapsulate IPv6 bên trong IPv4 |
| 2 site cần "nhìn thấy nhau" trực tiếp | Tunnel tạo virtual link point-to-point |
| Routing protocol cần multicast | GRE hỗ trợ multicast (IPSec không) |
| Non-IP protocol (IPX, AppleTalk) qua IP | GRE đóng gói bất kỳ L3 protocol |

### GRE trong đời thường

Khi công ty bạn có **2 chi nhánh**:
- Chi nhánh Hà Nội: mạng 10.0.1.0/24
- Chi nhánh TP.HCM: mạng 10.0.2.0/24
- Kết nối qua **Internet** (không có MPLS VPN)

```
Hà Nội [10.0.1.0/24] ←→ Internet ←→ TP.HCM [10.0.2.0/24]
                            ↑
                     IP private không
                     đi qua Internet được!

Solution: GRE Tunnel
Hà Nội [10.0.1.0/24] ←─── GRE Tunnel ───→ TP.HCM [10.0.2.0/24]
         Router A         (qua Internet)          Router B
         203.0.113.1                              198.51.100.1
```

---

## 2. GRE là gì? — Giải thích cho người không biết IT

### Phép so sánh đời thường — Xe container

Bạn muốn vận chuyển **xe máy** từ Hà Nội đến TP.HCM bằng **đường sắt**. Vấn đề:
- Đường sắt chỉ chở **container** (tiêu chuẩn)
- Xe máy không phải container → không chở được trực tiếp

**Giải pháp:**
1. Đặt xe máy vào **container** (encapsulation)
2. Container có **địa chỉ gửi/nhận** rõ ràng (outer header)
3. Tàu hỏa chở container qua các ga (Internet routers)
4. Tại TP.HCM, mở container → lấy xe máy ra (decapsulation)
5. Xe máy nguyên vẹn như lúc gửi!

GRE = Container cho network packets. Nó đóng gói **bất kỳ** loại packet vào trong IP packet để vận chuyển.

### Định nghĩa kỹ thuật

**GRE (Generic Routing Encapsulation)** là tunneling protocol cho phép đóng gói (encapsulate) một giao thức mạng bên trong một giao thức mạng khác, tạo ra virtual point-to-point link giữa 2 điểm.

Đặc điểm:
- **Protocol number 47** trong IP header
- **Stateless** — không maintain connection state
- **No encryption** — không mã hóa (cần kết hợp IPSec)
- **Multiprotocol** — đóng gói IPv4, IPv6, MPLS, multicast...
- **Low overhead** — header nhỏ (4-16 bytes)
- **Support multicast** — ưu thế so với IPSec tunnel mode

### RFC liên quan

| RFC | Tên | Năm | Mô tả |
|---|---|---|---|
| RFC 2784 | Generic Routing Encapsulation (GRE) | 2000 | GRE base specification |
| RFC 2890 | Key and Sequence Number Extensions to GRE | 2000 | Thêm Key và Seq# |
| RFC 1701 | Generic Routing Encapsulation | 1994 | GRE original (obsoleted) |
| RFC 1702 | Generic Routing Encapsulation over IPv4 | 1994 | GRE over IPv4 |
| RFC 7676 | IPv6 Support for GRE | 2015 | GRE trên IPv6 underlay |

### Tunneling concepts cơ bản

```
┌──────────────────────────────────────────────────────────┐
│                    TUNNEL                                  │
│                                                          │
│  Tunnel Source ══════════════════════════ Tunnel Dest     │
│  (Encapsulator)        Virtual Link        (Decapsulator)│
│                                                          │
│  Original Packet → Encapsulate → Transport → Decapsulate │
│                                                          │
└──────────────────────────────────────────────────────────┘

Terminology:
- Passenger Protocol: protocol being encapsulated (inner)
- Carrier Protocol: GRE itself (middle)
- Transport Protocol: protocol used for delivery (outer IP)
```

---

## 3. Cấu trúc GRE Header — Chi tiết từng trường

### Phép so sánh — Phiếu gửi hàng

Khi gửi hàng qua chuyển phát nhanh, bạn điền **phiếu gửi**:
- Loại hàng gì? (Protocol Type)
- Có bảo hiểm không? (Checksum)
- Số tracking? (Key)
- Kiện thứ mấy trong lô? (Sequence Number)

GRE header = Phiếu gửi dán trên packet.

### GRE Header Format (RFC 2784 + RFC 2890)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│C│ │K│S│    Reserved0 (9 bits)    │    Version (3 bits)           │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│              Protocol Type (16 bits)                             │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│              Checksum (16 bits)    │    Reserved1 (16 bits)       │  ← Optional (C=1)
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│              Key (32 bits)                                        │  ← Optional (K=1)
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│              Sequence Number (32 bits)                            │  ← Optional (S=1)
└─────────────────────────────────────────────────────────────────────┘
```

### Chi tiết từng trường

#### Flags (4 bits đầu)

| Bit | Flag | Ý nghĩa |
|---|---|---|
| 0 | C (Checksum) | 1 = Checksum field present |
| 1 | Reserved | Phải = 0 |
| 2 | K (Key) | 1 = Key field present |
| 3 | S (Sequence Number) | 1 = Sequence Number present |

#### Reserved0 — 9 bits
Phải set = 0. Receiver bỏ qua.

#### Version — 3 bits
- **0** = GRE standard (RFC 2784)
- **1** = Enhanced GRE (RFC 2637 — PPTP, obsolete)

#### Protocol Type — 16 bits

Xác định **passenger protocol** (packet bên trong GRE):

| Value | Protocol |
|---|---|
| 0x0800 | IPv4 |
| 0x86DD | IPv6 |
| 0x8847 | MPLS Unicast |
| 0x8848 | MPLS Multicast |
| 0x6558 | Transparent Ethernet Bridging |
| 0x0806 | ARP |
| 0x880B | PPP |

#### Checksum — 16 bits (optional, khi C=1)

- One's complement checksum
- Tính trên toàn bộ GRE header + payload
- Giúp detect corruption

#### Key — 32 bits (optional, khi K=1)

- Identifier cho tunnel
- Dùng để phân biệt nhiều tunnels giữa cùng 2 endpoints
- Cả 2 đầu phải match key

#### Sequence Number — 32 bits (optional, khi S=1)

- Đánh số thứ tự packets
- Detect out-of-order delivery
- Detect packet loss
- KHÔNG dùng cho retransmission (GRE không reliable)

### Encapsulated Packet Structure

```
Khi original IP packet đi qua GRE tunnel:

┌─────────────────────────────────────────────────────────────┐
│  New (Outer) IP Header  │  GRE Header  │  Original Packet   │
│  20 bytes               │  4-16 bytes  │  (inner IP + data) │
│                         │              │                     │
│  Src: Tunnel Source     │  Protocol    │  Src: Original Src  │
│  Dst: Tunnel Dest       │  Type        │  Dst: Original Dst  │
│  Protocol: 47 (GRE)    │  Key (opt)   │  Payload            │
│  TTL: new TTL           │  Seq (opt)   │                     │
└─────────────────────────┴──────────────┴─────────────────────┘
       Delivery Header       Carrier        Passenger Packet
       (Transport)           (GRE)          (Payload)
```

### Ví dụ packet capture

```
Original packet:
  Src IP: 10.0.1.100
  Dst IP: 10.0.2.200
  Protocol: TCP
  Payload: "Hello"

After GRE encapsulation:
  Outer IP Header:
    Src: 203.0.113.1 (tunnel source — public IP Router A)
    Dst: 198.51.100.1 (tunnel dest — public IP Router B)
    Protocol: 47 (GRE)
    TTL: 255
  
  GRE Header:
    C=0, K=1, S=0
    Protocol Type: 0x0800 (IPv4)
    Key: 12345
  
  Inner IP Header:
    Src: 10.0.1.100 (original source)
    Dst: 10.0.2.200 (original destination)
    Protocol: 6 (TCP)
    TTL: 64
  
  TCP + Payload:
    "Hello"
```

---

## 4. GRE hoạt động — Step by Step

### Phép so sánh — Hệ thống ống ngầm

Tưởng tượng 2 tòa nhà ở 2 bên đường lớn:
1. Xây **ống ngầm** dưới đường kết nối 2 tòa nhà
2. Người từ tòa A bước vào ống ngầm → đi qua → ra tòa B
3. Xe cộ trên đường **không biết** có ống ngầm bên dưới
4. Người trong ống **không bị** xe cộ ảnh hưởng

GRE tunnel = Ống ngầm cho network traffic.

### Quy trình đóng gói (Encapsulation)

```
Bước 1: Packet gốc đến tunnel interface
  PC (10.0.1.100) gửi packet → Router A
  Destination: 10.0.2.200
  Router A check routing table:
    10.0.2.0/24 → via Tunnel0 interface

Bước 2: Router A encapsulate
  - Lấy original packet (inner)
  - Thêm GRE header (4-16 bytes)
  - Thêm new IP header (outer):
    Src = Tunnel Source (203.0.113.1)
    Dst = Tunnel Destination (198.51.100.1)
    Protocol = 47 (GRE)
    
Bước 3: Route outer packet qua Internet
  Router A check routing table cho 198.51.100.1:
    198.51.100.1 → via GigabitEthernet0/0 → ISP gateway
  Forward encapsulated packet ra Internet

Bước 4: Transit routers forward
  Internet routers chỉ thấy outer IP header
  Forward based on dst = 198.51.100.1
  Không biết bên trong có gì!

Bước 5: Router B nhận packet
  Nhận packet với dst = 198.51.100.1 (own IP)
  Protocol = 47 → đây là GRE packet!

Bước 6: Router B decapsulate
  - Gỡ outer IP header
  - Gỡ GRE header
  - Lấy original packet ra
  - Check inner destination: 10.0.2.200
  - Forward theo routing table nội bộ → LAN interface
  
Bước 7: Packet đến đích
  PC (10.0.2.200) nhận packet gốc — không biết nó đã đi qua tunnel!
```

### Tunnel Interface — Virtual Interface

```
Router A:
  interface Tunnel0
    ip address 172.16.0.1 255.255.255.252  ← Tunnel IP (point-to-point)
    tunnel source 203.0.113.1              ← Outer IP source
    tunnel destination 198.51.100.1        ← Outer IP destination
    tunnel mode gre ip                     ← GRE over IPv4

  ip route 10.0.2.0 255.255.255.0 172.16.0.2  ← Route traffic via tunnel

Router B:
  interface Tunnel0
    ip address 172.16.0.2 255.255.255.252
    tunnel source 198.51.100.1
    tunnel destination 203.0.113.1
    tunnel mode gre ip
    
  ip route 10.0.1.0 255.255.255.0 172.16.0.1
```

### GRE hỗ trợ Dynamic Routing

Một ưu thế lớn của GRE so với IPSec tunnel mode — GRE support **multicast** → chạy được routing protocols qua tunnel:

```
Router A:
  router ospf 1
    network 10.0.1.0 0.0.0.255 area 0
    network 172.16.0.0 0.0.0.3 area 0  ← Tunnel network

Router B:
  router ospf 1
    network 10.0.2.0 0.0.0.255 area 0
    network 172.16.0.0 0.0.0.3 area 0

→ OSPF adjacency qua GRE tunnel
→ Routes exchange tự động
→ Không cần static route!
```

---

## 5. MTU và Fragmentation — Vấn đề "gói hàng quá lớn"

### Phép so sánh — Thang máy có giới hạn trọng tải

Thang máy chở tối đa **1500 kg** (= MTU 1500 bytes):
- Hàng nặng 1500 kg → vừa đúng! (packet 1500 bytes)
- Nhưng khi đóng gói thêm hộp (GRE header 24 bytes):
  - Hàng 1500 + hộp 24 = **1524 kg** → quá tải!
  - Thang máy từ chối! (packet too large)

**Giải pháp:**
- Giảm hàng xuống 1476 kg (adjust tunnel MTU)
- Hoặc chia hàng làm 2 chuyến (fragmentation)

### MTU Problem với GRE

```
Normal Ethernet MTU: 1500 bytes

Khi đi qua GRE tunnel:
  Outer IP Header:    20 bytes
  GRE Header:          4 bytes (minimum, no options)
  ─────────────────────────────
  Total overhead:     24 bytes
  
  Effective MTU for payload: 1500 - 24 = 1476 bytes

Nếu GRE + Checksum + Key:
  Outer IP Header:    20 bytes
  GRE Header:         12 bytes (4 base + 4 checksum + 4 key)
  ─────────────────────────────
  Total overhead:     32 bytes
  
  Effective MTU: 1500 - 32 = 1468 bytes
```

### Khi kết hợp GRE + IPSec (phổ biến nhất):

```
Outer IP Header:      20 bytes
IPSec ESP Header:      8 bytes (SPI + Seq)
IPSec IV:             8-16 bytes (depends on cipher)
GRE Header:           4-12 bytes
─────────────────────────────────
Padding:              0-15 bytes (block cipher alignment)
ESP Trailer:          2 bytes (pad length + next header)
ESP Auth (ICV):       12-16 bytes (HMAC)
─────────────────────────────────
Total overhead:       54-81 bytes!

Effective MTU: 1500 - 73 (typical) ≈ 1400 bytes (safe value)
```

### Giải pháp MTU

#### Solution 1: Giảm Tunnel MTU

```
interface Tunnel0
  ip mtu 1400          ! Set tunnel MTU thấp hơn
  ! Packet > 1400 sẽ bị fragment TRƯỚC khi encapsulate
```

#### Solution 2: TCP MSS Clamping

```
interface Tunnel0
  ip tcp adjust-mss 1360   ! Thay đổi TCP MSS trong SYN packets
  
! Khi TCP SYN đi qua tunnel:
!   Original MSS = 1460
!   Router sửa: MSS = 1360
!   → TCP segments sẽ nhỏ hơn → không bị fragment
```

**Tại sao MSS = MTU - 40?**
```
MTU 1400:
  IP Header:  20 bytes
  TCP Header: 20 bytes
  ────────────────────
  Headers:    40 bytes
  MSS:        1400 - 40 = 1360 bytes
```

#### Solution 3: Path MTU Discovery (PMTUD)

```
1. Sender gửi packet với DF=1 (Don't Fragment)
2. Nếu packet quá lớn tại tunnel → router gửi ICMP
   "Fragmentation Needed" (Type 3, Code 4) với MTU value
3. Sender giảm packet size theo MTU nhận được
4. Retry với size nhỏ hơn

Vấn đề: Nhiều firewall BLOCK ICMP → PMTUD fail!
  → "PMTUD Black Hole" — packet lớn bị drop im lặng
  → Triệu chứng: ping OK, nhưng web/file transfer fail
```

#### Solution 4: Fragmentation strategies

```
Option A: Fragment TRƯỚC encapsulation (pre-fragmentation)
  Original: [1500 byte packet]
  → Fragment: [750 bytes] + [750 bytes]
  → Encapsulate each: [750+24=774] + [750+24=774]
  → Cả 2 fragments đều < 1500 MTU ✓
  
  Ưu điểm: Reassembly tại destination (not tunnel endpoint)
  Cisco default: "tunnel path-mtu-discovery"

Option B: Fragment SAU encapsulation (post-fragmentation)
  Original: [1500 byte packet]
  → Encapsulate: [1500+24=1524 byte GRE packet]
  → Fragment outer: [1500 bytes] + [48 bytes]  (outer IP fragmentation)
  → Reassembly tại tunnel endpoint
  
  Nhược điểm: Fragment outer IP → tunnel endpoint phải reassemble
```

### Best Practice MTU

```
Recommendation cho GRE tunnel:
  GRE only:          ip mtu 1476, ip tcp adjust-mss 1436
  GRE + IPSec:       ip mtu 1400, ip tcp adjust-mss 1360
  GRE + IPSec + NAT: ip mtu 1380, ip tcp adjust-mss 1340
  
  Safe universal:    ip mtu 1400, ip tcp adjust-mss 1360
```

---

## 6. Recursive Routing — Vấn đề "đường vòng vô tận"

### Phép so sánh — Bản đồ chỉ nhầm

Tưởng tượng bạn hỏi đường đến siêu thị:
- Người 1: "Đi theo đường A" 
- Bạn đến đường A, hỏi tiếp
- Người 2: "Quay lại hỏi người 1"
- Bạn quay lại → vòng lặp vô tận!

**Recursive routing** trong GRE xảy ra khi:
- Route đến tunnel destination đi QUA chính tunnel đó
- Tạo vòng lặp: "Để đi đến B, dùng tunnel → tunnel cần đi đến B → dùng tunnel → ..."

### Recursive Routing Problem — Chi tiết

```
Cấu hình SAI:

Router A:
  interface Tunnel0
    tunnel source 203.0.113.1
    tunnel destination 198.51.100.1
    ip address 172.16.0.1 255.255.255.252

  ! Static route qua tunnel
  ip route 10.0.2.0 255.255.255.0 172.16.0.2

  ! DEFAULT ROUTE cũng qua tunnel (SAI!)
  ip route 0.0.0.0 0.0.0.0 172.16.0.2   ← PROBLEM!

Logic:
  1. Router cần gửi packet đến 198.51.100.1 (tunnel dest)
  2. Lookup: 198.51.100.1 matches 0.0.0.0/0 → via Tunnel0
  3. Để gửi qua Tunnel0 → cần reach 198.51.100.1 (tunnel dest)
  4. Lookup: 198.51.100.1 matches 0.0.0.0/0 → via Tunnel0
  5. → LOOP! Tunnel interface flaps (up/down liên tục)
```

### Giải pháp Recursive Routing

#### Solution 1: Specific static route cho tunnel destination

```
! Route đến tunnel destination PHẢI đi qua physical interface
ip route 198.51.100.1 255.255.255.255 203.0.113.2  ← ISP gateway
  ! (đi qua physical interface, KHÔNG qua tunnel)

! Default route có thể qua tunnel
ip route 0.0.0.0 0.0.0.0 Tunnel0

! Hoặc specific route qua tunnel
ip route 10.0.2.0 255.255.255.0 Tunnel0
```

#### Solution 2: Tunnel source = interface (not IP)

```
interface Tunnel0
  tunnel source GigabitEthernet0/0   ← Interface thay vì IP
  tunnel destination 198.51.100.1

! Router tự biết dùng Gi0/0 để reach tunnel dest
! Giảm risk recursive routing
```

#### Solution 3: Route filtering

```
! Sử dụng route-map hoặc prefix-list
! Đảm bảo tunnel destination KHÔNG learned via tunnel

router bgp 65000
  neighbor 172.16.0.2 route-map DENY-TUNNEL-DST in
  
route-map DENY-TUNNEL-DST deny 10
  match ip address prefix-list TUNNEL-DST
route-map DENY-TUNNEL-DST permit 20

ip prefix-list TUNNEL-DST permit 198.51.100.1/32
```

#### Solution 4: Tunnel route nhận dạng (Cisco specific)

```
! Cisco IOS tự detect recursive routing
! Tunnel interface sẽ go DOWN nếu detect
! Syslog: %TUN-5-RECURDOWN: Tunnel0 temporarily disabled due to recursive routing

! Fix: Ensure tunnel destination reachable via non-tunnel path
```

### Linux — Tránh recursive routing

```bash
# Thêm route cho tunnel endpoint qua physical interface
ip route add 198.51.100.1/32 via 203.0.113.2 dev eth0

# Route traffic qua tunnel (trừ tunnel destination)
ip route add default dev gre1 metric 100
# hoặc
ip route add 10.0.2.0/24 dev gre1
```

---

## 7. GRE over IPSec — Kết hợp Tunneling + Encryption

### Phép so sánh — Xe bọc thép chở hàng

- **GRE** = Xe tải thường — chở được nhiều loại hàng nhưng **không bảo mật**
- **IPSec** = Khóa + bọc thép — bảo mật nhưng **không chở được multicast**
- **GRE over IPSec** = Xe tải bọc thép — vừa linh hoạt VỪA an toàn!

### Tại sao kết hợp?

| Feature | GRE alone | IPSec alone | GRE + IPSec |
|---|---|---|---|
| Encryption | ❌ | ✅ | ✅ |
| Multicast | ✅ | ❌ (tunnel mode) | ✅ |
| Routing protocols | ✅ (OSPF, EIGRP, BGP) | Limited | ✅ |
| Multiple protocols | ✅ (IPv4, IPv6, MPLS) | IPv4/IPv6 only | ✅ |
| Performance | Tốt | Tốn CPU (encryption) | Tốn CPU |
| Overhead | Thấp (24 bytes) | Cao (50-70 bytes) | Rất cao (74-97 bytes) |

### 2 Modes: GRE over IPSec

#### Mode 1: Tunnel mode (GRE inside IPSec tunnel)

```
┌───────────────────────────────────────────────────────────────┐
│ New IP │ ESP │ GRE IP │ GRE  │ Original │ Original │ ESP │ESP │
│ Header │ Hdr │ Header │ Hdr  │ IP Hdr   │ Payload  │Trail│Auth│
│        │     │(tunnel)│      │ (inner)  │          │     │    │
└───────────────────────────────────────────────────────────────┘
  Outer     IPSec   GRE outer  GRE    Passenger packet    IPSec
  delivery          (tunnel    header
  header            endpoints)

Overhead: 20 (outer IP) + 8 (ESP) + 8-16 (IV) + 20 (GRE IP) + 4 (GRE) 
         + 2 (trailer) + 12 (auth) = ~74-90 bytes
```

#### Mode 2: Transport mode (GRE inside IPSec transport)

```
┌─────────────────────────────────────────────────────────┐
│ GRE IP │ ESP │ GRE  │ Original │ Original │ ESP  │ ESP │
│ Header │ Hdr │ Hdr  │ IP Hdr   │ Payload  │Trail │Auth │
│(tunnel)│     │      │ (inner)  │          │      │     │
└─────────────────────────────────────────────────────────┘
  GRE outer  IPSec  GRE   Passenger packet       IPSec
  = delivery         hdr
  header

Overhead: 20 (GRE IP) + 8 (ESP) + 8-16 (IV) + 4 (GRE)
         + 2 (trailer) + 12 (auth) = ~54-62 bytes

Tiết kiệm 20 bytes so với tunnel mode! (không có extra outer IP)
```

**Transport mode preferred** khi tunnel source/dest = IPSec peer (thường là vậy).

### Cấu hình GRE over IPSec — Cisco IOS

```
! ===== Phase 1: ISAKMP (IKE) Policy =====
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400

crypto isakmp key STRONG_PSK address 198.51.100.1

! ===== Phase 2: IPSec Transform Set =====
crypto ipsec transform-set GRE-IPSEC esp-aes 256 esp-sha256-hmac
 mode transport    ! Transport mode cho GRE over IPSec

! ===== IPSec Profile (cho tunnel interface) =====
crypto ipsec profile GRE-IPSEC-PROFILE
 set transform-set GRE-IPSEC
 set pfs group14

! ===== GRE Tunnel Interface =====
interface Tunnel0
 ip address 172.16.0.1 255.255.255.252
 ip mtu 1400
 ip tcp adjust-mss 1360
 tunnel source GigabitEthernet0/0
 tunnel destination 198.51.100.1
 tunnel mode gre ip
 tunnel protection ipsec profile GRE-IPSEC-PROFILE  ! Apply IPSec
 
! ===== Routing qua tunnel =====
router ospf 1
 network 172.16.0.0 0.0.0.3 area 0
 network 10.0.1.0 0.0.0.255 area 0
```

### Cấu hình trên Linux

```bash
# === Router A (203.0.113.1) ===

# 1. Tạo GRE tunnel
ip tunnel add gre1 mode gre remote 198.51.100.1 local 203.0.113.1 ttl 255
ip addr add 172.16.0.1/30 dev gre1
ip link set gre1 up

# 2. Routes qua tunnel
ip route add 10.0.2.0/24 dev gre1

# 3. IPSec (using strongSwan)
# /etc/ipsec.conf
conn gre-tunnel
    left=203.0.113.1
    right=198.51.100.1
    type=transport         # Transport mode
    authby=secret
    esp=aes256-sha256!
    ike=aes256-sha256-modp2048!
    leftprotoport=47       # Protocol 47 = GRE
    rightprotoport=47
    auto=start

# /etc/ipsec.secrets
203.0.113.1 198.51.100.1 : PSK "STRONG_PSK_HERE"

# Restart strongSwan
systemctl restart strongswan
```

---

## 8. Các loại IP Tunneling khác — So sánh với GRE

### Phép so sánh — Các loại "ống dẫn"

- **GRE** = Ống PVC đa năng — chở được nước, khí, dây điện (multi-protocol)
- **IP-in-IP** = Ống nước đơn giản — chỉ chở nước (IP only)
- **6in4** = Ống chuyển đổi — đưa nước nóng (IPv6) qua hệ thống nước lạnh (IPv4)
- **VXLAN** = Ống lớn chở cả container — cho data center
- **IPSec Tunnel** = Ống bọc thép — bảo mật nhưng hạn chế

### So sánh các tunneling protocols

| Protocol | RFC | Overhead | Multicast | Encrypt | Use Case |
|---|---|---|---|---|---|
| GRE | 2784 | 24 bytes | ✅ | ❌ | Site-to-site, multiprotocol |
| IP-in-IP | 2003 | 20 bytes | ❌ | ❌ | Simple IPv4 tunneling |
| 6in4 | 4213 | 20 bytes | ❌ | ❌ | IPv6 over IPv4 |
| 6to4 | 3056 | 20 bytes | ❌ | ❌ | Auto IPv6 tunneling |
| ISATAP | 5214 | 20 bytes | ❌ | ❌ | IPv6 intra-site |
| IPSec Tunnel | 4301 | 50-73 bytes | ❌ | ✅ | Secure VPN |
| VXLAN | 7348 | 50 bytes | ✅ | ❌ | Data center overlay |
| Geneve | 8926 | 50+ bytes | ✅ | ❌ | Modern DC overlay |
| WireGuard | — | 32 bytes | ❌ | ✅ | Modern VPN |

### IP-in-IP (RFC 2003)

```
Encapsulation đơn giản nhất — chỉ thêm outer IP header:

┌──────────────┬──────────────┬──────────┐
│ Outer IP Hdr │ Inner IP Hdr │ Payload  │
│ Protocol: 4  │ (original)   │          │
│ (IPIP)       │              │          │
└──────────────┴──────────────┴──────────┘

Ưu điểm: Overhead nhỏ nhất (chỉ 20 bytes)
Nhược điểm: Chỉ tunnel IPv4 trong IPv4 (không multiprotocol)
```

```bash
# Linux IP-in-IP tunnel
ip tunnel add ipip1 mode ipip remote 198.51.100.1 local 203.0.113.1
ip addr add 172.16.0.1/30 dev ipip1
ip link set ipip1 up
```

### 6in4 Tunnel (RFC 4213)

```
IPv6 packet đóng gói trong IPv4:

┌──────────────┬──────────────┬──────────┐
│ IPv4 Header  │ IPv6 Header  │ Payload  │
│ Protocol: 41 │ (original)   │          │
│ (IPv6)       │              │          │
└──────────────┴──────────────┴──────────┘

Use case: Kết nối IPv6 islands qua IPv4 Internet
```

```bash
# Linux 6in4 tunnel
ip tunnel add sit1 mode sit remote 198.51.100.1 local 203.0.113.1
ip -6 addr add 2001:db8::1/64 dev sit1
ip link set sit1 up
ip -6 route add 2001:db8:2::/48 dev sit1
```

### DMVPN — Dynamic Multipoint VPN (Cisco)

```
Giải quyết vấn đề: Full mesh GRE tunnels giữa nhiều sites

Traditional: N sites = N*(N-1)/2 tunnels
  10 sites = 45 tunnels! Nightmare!

DMVPN solution:
- Hub-and-Spoke topology với dynamic spoke-to-spoke tunnels
- Components:
  - mGRE (Multipoint GRE) — 1 tunnel interface, nhiều destinations
  - NHRP (Next Hop Resolution Protocol) — map tunnel IP → public IP
  - Routing protocol (EIGRP/OSPF/BGP) — qua mGRE
  - IPSec — encryption (optional)

Flow:
  1. Spoke A muốn nói chuyện với Spoke B
  2. Initially: traffic đi qua Hub
  3. Hub gửi NHRP redirect → Spoke A
  4. Spoke A query NHRP: "Public IP của Spoke B?"
  5. Hub reply: "Spoke B = 198.51.100.5"
  6. Spoke A tạo direct GRE tunnel → Spoke B
  7. Traffic đi trực tiếp (bypass Hub)
```

```
! DMVPN Hub Configuration (Cisco)
interface Tunnel0
 ip address 172.16.0.1 255.255.255.0
 ip nhrp network-id 1
 ip nhrp map multicast dynamic
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint         ! mGRE — multipoint!
 tunnel protection ipsec profile DMVPN-PROFILE

! DMVPN Spoke Configuration
interface Tunnel0
 ip address 172.16.0.2 255.255.255.0
 ip nhrp network-id 1
 ip nhrp nhs 172.16.0.1              ! Hub tunnel IP
 ip nhrp map 172.16.0.1 203.0.113.1  ! Hub public IP
 ip nhrp map multicast 203.0.113.1
 tunnel source GigabitEthernet0/0
 tunnel mode gre multipoint
 tunnel protection ipsec profile DMVPN-PROFILE
```

---

## 9. GRE Use Cases — Khi nào dùng GRE?

### Use Case 1: Site-to-Site VPN với Dynamic Routing

```
Scenario: Công ty 3 chi nhánh cần full mesh connectivity
  - HQ (10.0.0.0/16) ← → Branch1 (10.1.0.0/16)
  - HQ ← → Branch2 (10.2.0.0/16)
  - Branch1 ← → Branch2

Solution: GRE + OSPF + IPSec
  - GRE tunnels giữa mỗi cặp
  - OSPF chạy qua tunnels → auto route exchange
  - IPSec protect GRE traffic

Advantages:
  - Thêm subnet mới → OSPF tự advertise (no manual route)
  - Link fail → OSPF reconverge → traffic đi path khác
  - Multicast/broadcast qua tunnel (video conferencing)
```

### Use Case 2: IPv6 Transition

```
Scenario: Company có IPv6 network nhưng ISP chỉ có IPv4

Solution: 6in4 tunnel (hoặc GRE carrying IPv6)
  
  IPv6 Site A ←── 6in4 Tunnel (qua IPv4 Internet) ──→ IPv6 Site B
  2001:db8:1::/48                                      2001:db8:2::/48
```

### Use Case 3: Traffic Engineering / Policy Routing

```
Scenario: Traffic đến 10.0.5.0/24 phải đi qua firewall/IDS

Solution: GRE tunnel đến firewall, PBR direct traffic vào tunnel

  Normal path: R1 → R2 → R3 → Destination
  Policy path: R1 → [GRE Tunnel] → Firewall → R3 → Destination
```

### Use Case 4: Network Migration

```
Scenario: Migrate từ OSPF sang BGP — cần cả 2 routing domains

Solution: GRE tunnel "jump" qua BGP domain:
  - OSPF routers tạo GRE tunnel qua BGP network
  - OSPF adjacency maintain qua tunnel
  - Gradual migration — không downtime
```

### Use Case 5: AWS/Cloud Connectivity

```
Scenario: Connect on-premises tới AWS VPC

Option 1: AWS Site-to-Site VPN (IPSec, không GRE)
Option 2: Customer-managed GRE:
  - EC2 instance chạy GRE tunnel endpoint
  - On-prem router ← GRE over IPSec → EC2
  - Dynamic routing (BGP) qua tunnel
  - Flexible hơn AWS managed VPN
```

### Khi KHÔNG nên dùng GRE

| Situation | Thay thế |
|---|---|
| Chỉ cần encryption, không cần multicast | IPSec tunnel mode |
| Data center overlay (rất nhiều tunnels) | VXLAN/Geneve |
| Modern VPN cho remote users | WireGuard/OpenVPN |
| ISP backbone | MPLS (bài trước) |
| High-performance encrypted tunnel | WireGuard |
| Cloud-native networking | Cloud VPN services |

---

## 10. Troubleshooting GRE và Best Practices

### GRE Troubleshooting Checklist

```
Tunnel không UP:
□ 1. Tunnel source interface UP?
     show interface GigabitEthernet0/0
□ 2. Tunnel destination reachable?
     ping 198.51.100.1 source 203.0.113.1
□ 3. Protocol 47 (GRE) không bị block?
     Firewall cho phép IP protocol 47 (KHÔNG phải port 47!)
□ 4. Recursive routing?
     show ip route 198.51.100.1
     (route phải đi qua physical, KHÔNG qua Tunnel0)
□ 5. Tunnel key match (nếu dùng)?
     Cả 2 đầu phải cùng key value
□ 6. GRE keepalives?
     Nếu keepalive enabled, cả 2 đầu phải support
```

```
Tunnel UP nhưng traffic không qua:
□ 1. Routing qua tunnel interface?
     show ip route — traffic destination phải via Tunnel
□ 2. MTU issues?
     ping với size lớn: ping 10.0.2.1 size 1500 df-bit
     Nếu fail → MTU problem
□ 3. ACL blocking trên tunnel interface?
□ 4. RPF (Reverse Path Forwarding) check fail?
     ip verify unicast source reachable-via any
□ 5. NAT trên path? (GRE protocol 47 không có port → NAT issues)
```

### Commands kiểm tra — Cisco

```bash
# Tunnel interface status
show interface Tunnel0

# Tunnel statistics
show tunnel interface Tunnel0

# Verify tunnel endpoints
show ip interface brief | include Tunnel

# Check routing
show ip route | include Tunnel
show ip cef 10.0.2.0 255.255.255.0

# Debug GRE (caution — production impact)
debug tunnel
debug ip packet detail

# Verify IPSec (nếu GRE over IPSec)
show crypto ipsec sa
show crypto session
```

### Commands kiểm tra — Linux

```bash
# List all tunnels
ip tunnel show

# Tunnel interface details
ip -d link show gre1

# Statistics
ip -s link show gre1

# Route check
ip route get 10.0.2.200

# Packet capture
tcpdump -i eth0 proto gre
tcpdump -i gre1 -nn

# Check MTU
ip link show gre1 | grep mtu

# Trace path with MTU discovery
tracepath -n 10.0.2.200
```

### Best Practices

| Practice | Mô tả |
|---|---|
| Always set ip mtu | Đặt tunnel MTU phù hợp (1400 cho GRE+IPSec) |
| TCP MSS clamping | ip tcp adjust-mss trên tunnel interface |
| Avoid recursive routing | Static route cho tunnel dest qua physical |
| Use keepalives | Detect remote failure (cả 2 đầu enable) |
| Monitor tunnel state | SNMP trap cho tunnel up/down |
| IPSec for security | GRE alone không encrypt — luôn kết hợp IPSec |
| BFD for fast failover | BFD qua tunnel cho sub-second failover |
| Document tunnel endpoints | Maintain inventory của all tunnels |
| Key for multi-tunnel | Dùng tunnel key khi nhiều tunnels cùng endpoints |

### Performance Considerations

```
GRE overhead:
  - Encapsulation/Decapsulation: CPU cost
  - Extra 24+ bytes per packet: bandwidth cost
  - Fragment/Reassembly: significant CPU cost
  - IPSec encryption: highest CPU cost

Throughput estimation:
  Without IPSec: ~95% of wire speed (modern hardware)
  With IPSec (SW): ~200 Mbps - 1 Gbps (depends on CPU)
  With IPSec (HW crypto): near wire speed

Latency:
  GRE adds: ~1-2 ms processing
  GRE + IPSec (SW): ~2-5 ms
  GRE + IPSec (HW): ~1-2 ms
```

### Tổng kết

```
GRE Tunneling Summary:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  GRE = Generic Routing Encapsulation                    │
│  • Protocol 47, Layer 2.5 tunnel                       │
│  • Encapsulate ANY L3 protocol inside IP               │
│  • Stateless, no encryption, low overhead              │
│  • Supports multicast → dynamic routing                │
│                                                         │
│  Key Points:                                            │
│  1. Header: 4-16 bytes (flags, protocol, key, seq)     │
│  2. MTU: 1500 - 24 = 1476 (GRE only)                  │
│  3. MTU: 1500 - 73 ≈ 1400 (GRE + IPSec)              │
│  4. Recursive routing: tunnel dest via physical!       │
│  5. GRE + IPSec = best combo (flexibility + security)  │
│  6. TCP MSS clamping: ALWAYS configure                 │
│  7. DMVPN = scalable multi-site GRE                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

*Tài liệu tham khảo:*
- RFC 2784 — Generic Routing Encapsulation (GRE)
- RFC 2890 — Key and Sequence Number Extensions to GRE
- RFC 2003 — IP Encapsulation within IP
- RFC 4213 — Basic Transition Mechanisms for IPv6 Hosts and Routers
- RFC 7348 — Virtual eXtensible Local Area Network (VXLAN)
- RFC 4301 — Security Architecture for the Internet Protocol (IPSec)
- Cisco GRE Tunnel Configuration Guide — IOS XE
- Cisco DMVPN Design Guide
- Linux ip-tunnel(8) man page
