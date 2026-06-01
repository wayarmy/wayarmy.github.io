---
layout: post
title: "IPv4 Header Deep Dive — Mổ xẻ từng bit trong phong bì của Internet"
subtitle: "Hiểu sâu 13 trường trong IPv4 Header — từ Version đến Options — nền tảng để hiểu routing, fragmentation và QoS"
tags: [networking, ipv4, layer3, routing, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
comments: true
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 791 | Internet Protocol — DARPA Internet Program Protocol Specification (1981) |
| RFC 2474 | Definition of the Differentiated Services Field (DS Field) in the IPv4 and IPv6 Headers |
| RFC 3168 | The Addition of Explicit Congestion Notification (ECN) to IP |
| RFC 6864 | Updated Specification of the IPv4 ID Field |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 5: The Network Layer |
| Stevens, W.R. — TCP/IP Illustrated, Vol. 1 | Chapter 3: IP: Internet Protocol |
| Kurose & Ross — Computer Networking: A Top-Down Approach, 8th Ed. | Chapter 4: The Network Layer: Data Plane |
| Cisco Documentation | IP Addressing: IPv4 Configuration Guide |
| AWS Documentation | VPC — Internet Protocol and Routing |

---

## 1. Giới thiệu — Tại sao cần biết IPv4 Header?

### Ví dụ đời thường: Phong bì thư và hệ thống bưu điện

Hãy tưởng tượng bạn muốn gửi một lá thư từ Hà Nội đến TP. Hồ Chí Minh. Bạn không thể chỉ viết nội dung rồi ném ra đường — bạn cần **một cái phong bì** với đầy đủ thông tin:

- **Địa chỉ người gửi** (để thư trả lại nếu không giao được)
- **Địa chỉ người nhận** (để bưu tá biết giao cho ai)
- **Tem thư** (để xác nhận bạn đã trả phí vận chuyển)
- **Mã bưu chính** (để phân loại thư nhanh hơn)
- **"Fragile — Dễ vỡ"** hoặc **"Ưu tiên"** (hướng dẫn xử lý đặc biệt)

Trong thế giới mạng máy tính, **IPv4 Header chính là cái phong bì đó**. Mỗi khi máy tính bạn gửi bất kỳ dữ liệu nào qua Internet — dù là email, video YouTube, hay tin nhắn Zalo — dữ liệu đó đều được "bọc" trong một IPv4 Header chứa tất cả thông tin cần thiết để router biết cách chuyển tiếp packet đến đích.

### Concrete scenario: Khi bạn xem video YouTube

Khi bạn bấm play một video trên YouTube:

1. Server của YouTube ở Mỹ (hoặc Singapore) tạo ra hàng ngàn **IP packets**
2. Mỗi packet có một **IPv4 Header** ở đầu — giống như dán nhãn vận chuyển lên mỗi kiện hàng
3. Header nói: "Tôi đến từ IP 142.250.190.46 (YouTube), gửi đến 192.168.1.100 (laptop bạn)"
4. Dọc đường, mỗi router đọc header, giảm TTL (Time-to-Live) xuống 1, rồi chuyển tiếp
5. Nếu packet quá lớn cho một đường truyền → router **phân mảnh** (fragment) dựa trên thông tin trong header
6. Cuối cùng, laptop bạn nhận packets, đọc header để **ghép lại đúng thứ tự** và hiển thị video

**Nếu không có IPv4 Header**: Packet giống như một kiện hàng không có nhãn — không ai biết gửi đi đâu, từ đâu đến, và ghép lại thế nào.

### Vấn đề IPv4 Header giải quyết

| Vấn đề | Trường header giải quyết |
|---------|--------------------------|
| Gửi packet đi đâu? | Destination IP Address |
| Packet từ đâu đến? | Source IP Address |
| Dùng protocol gì bên trong? (TCP/UDP/ICMP?) | Protocol field |
| Packet có bị loop vô hạn không? | TTL (Time-to-Live) |
| Packet quá lớn thì xử lý sao? | Identification + Flags + Fragment Offset |
| Packet ưu tiên cao hay thấp? | Type of Service / DSCP |
| Header có bị lỗi khi truyền không? | Header Checksum |
| Header dài bao nhiêu byte? | IHL (Internet Header Length) |

---

## 2. IPv4 Header là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**IPv4 Header** là phần **thông tin điều khiển** được gắn vào **đầu mỗi gói tin** (packet) khi dữ liệu di chuyển qua mạng Internet. Nó giống như:

- **Nhãn vận chuyển** trên kiện hàng DHL/FedEx
- **Giấy thông hành** của một người qua biên giới
- **Vé máy bay** với tên, điểm đi, điểm đến, số ghế, hạng vé

Header có kích thước tối thiểu **20 bytes** (160 bits) và tối đa **60 bytes** (480 bits). Nó chứa 13 trường (fields) khác nhau, mỗi trường có một nhiệm vụ riêng.

### Analogy: Vé máy bay quốc tế

Hãy so sánh IPv4 Header với một vé máy bay:

```
┌──────────────────────────────────────────────────────────────────┐
│                    VÉ MÁY BAY (= IPv4 Header)                    │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Version = Loại vé (nội địa/quốc tế)                              │
│  IHL = Số trang của vé (vé thường 1 trang, vé đặc biệt 2-3 trang)│
│  ToS/DSCP = Hạng ghế (Economy / Business / First Class)           │
│  Total Length = Tổng trọng lượng (vé + hành lý)                   │
│  Identification = Mã booking (để nhận diện khi thay đổi chuyến)   │
│  Flags = Quy định hành lý (được tách/không được tách)             │
│  Fragment Offset = Thứ tự kiện hành lý (1/3, 2/3, 3/3)           │
│  TTL = Số lần transit tối đa (quá nhiều → hủy vé)                │
│  Protocol = Mục đích chuyến đi (công tác/du lịch/học tập)        │
│  Header Checksum = Mã xác thực vé (chống vé giả)                 │
│  Source IP = Sân bay đi (Noi Bai HAN)                             │
│  Destination IP = Sân bay đến (Changi SIN)                        │
│  Options = Yêu cầu đặc biệt (suất ăn chay, xe lăn...)            │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Cấu trúc tổng quan (RFC 791)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

Mỗi dòng ngang = 32 bits (4 bytes). Header tối thiểu = 5 dòng × 4 bytes = **20 bytes**.

---

## 3. Version và IHL — Ai đang nói, nói dài bao nhiêu?

### Mini example: Kiểm tra phiên bản phần mềm

Khi bạn mở một file `.docx`, Microsoft Word đầu tiên kiểm tra: "File này tạo bằng Word 2003 hay Word 2024?". Tương tự, khi router nhận một packet, nó đầu tiên đọc **Version field** để biết: "Đây là IPv4 hay IPv6?"

### Version (4 bits)

- **Vị trí**: 4 bits đầu tiên của header (bit 0-3)
- **Giá trị**: Luôn = `4` (0100 nhị phân) cho IPv4
- **Mục đích**: Cho router biết đây là IPv4 packet, để xử lý đúng cách

```
Bit:     0 1 0 0
Giá trị: 4 (= IPv4)

Nếu là: 0 1 1 0
Giá trị: 6 (= IPv6 — format hoàn toàn khác!)
```

**Tại sao cần Version?** Vì trên cùng một đường truyền, có thể có cả IPv4 và IPv6 packets đi qua. Router đọc 4 bits đầu tiên để biết phải parse header theo format nào. Giống như bưu tá nhìn tem — tem trong nước xử lý khác tem quốc tế.

### IHL — Internet Header Length (4 bits)

- **Vị trí**: Bits 4-7 (ngay sau Version)
- **Giá trị**: Từ 5 đến 15 (đơn vị: 32-bit words = 4 bytes)
- **Mục đích**: Cho biết header dài bao nhiêu, để router biết data (payload) bắt đầu ở đâu

**Cách tính:**

```
IHL = 5 → Header = 5 × 4 = 20 bytes (tối thiểu, không có Options)
IHL = 6 → Header = 6 × 4 = 24 bytes (có 4 bytes Options)
IHL = 15 → Header = 15 × 4 = 60 bytes (tối đa, có 40 bytes Options)
```

**Tại sao không ghi trực tiếp số bytes?** Vì 4 bits chỉ biểu diễn được giá trị 0-15. Nếu ghi bytes trực tiếp, tối đa chỉ là 15 bytes — quá nhỏ. Nhưng vì header luôn là bội số của 4 bytes (padding nếu cần), ta nhân 4 để tiết kiệm bits mà vẫn biểu diễn được đến 60 bytes.

### Trong thực tế

```bash
# Dùng tcpdump để xem IPv4 header
$ sudo tcpdump -i eth0 -c 1 -XX

14:23:05.123456 IP 192.168.1.100.52341 > 142.250.190.46.443
# Byte đầu tiên: 0x45
#   4 = Version (IPv4)
#   5 = IHL (20 bytes, không có options)
```

Giá trị `0x45` ở byte đầu tiên là **phổ biến nhất** trên Internet — nghĩa là IPv4, header 20 bytes, không options.

### Trong AWS

Trong AWS VPC, tất cả traffic đều là IPv4 (hoặc IPv6 nếu bạn bật dual-stack). Khi bạn tạo một VPC, AWS tự động xử lý IPv4 headers — bạn không cần lo về IHL hay Version. Tuy nhiên, **Security Groups và NACLs** đọc các trường trong IPv4 header (Source IP, Destination IP, Protocol) để quyết định cho phép/chặn traffic.

---

## 4. Type of Service (ToS) / DSCP + ECN — Hạng vé của packet

### Mini example: Làn xe ưu tiên trên đường

Trên đường cao tốc, có **làn thường** và **làn ưu tiên** (xe cứu thương, xe cứu hỏa). Tương tự, trong mạng, không phải mọi packet đều được đối xử bình đẳng:

- Gọi video Zoom → cần **ưu tiên cao** (trễ = hình giật)
- Tải file torrent → ưu tiên thấp (trễ vài giây không sao)
- Truy cập web → ưu tiên trung bình

Trường **Type of Service** (8 bits, vị trí byte thứ 2) nói cho router biết: "Packet này thuộc hạng nào?"

### Lịch sử thay đổi

Trường 8 bits này đã được tái định nghĩa nhiều lần:

```
╔═══════════════════════════════════════════════════════════════╗
║             LỊCH SỬ TRƯỜNG ToS (8 bits)                      ║
╠═══════════════════════════════════════════════════════════════╣
║                                                               ║
║  RFC 791 (1981) — Original ToS:                              ║
║  ┌───────────────────────────────────────────┐               ║
║  │ Precedence(3) │ D │ T │ R │ 0 │ 0 │       │               ║
║  └───────────────────────────────────────────┘               ║
║  D = Low Delay, T = High Throughput, R = High Reliability    ║
║                                                               ║
║  RFC 2474 (1998) — DSCP (Differentiated Services):           ║
║  ┌───────────────────────────────────────────┐               ║
║  │     DSCP (6 bits)           │  ECN (2)    │               ║
║  └───────────────────────────────────────────┘               ║
║  DSCP = Differentiated Services Code Point                    ║
║  ECN = Explicit Congestion Notification (RFC 3168)           ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### DSCP — Differentiated Services Code Point (6 bits)

DSCP có 64 giá trị (0-63), được chia thành các nhóm:

| DSCP Value | Tên | Per-Hop Behavior | Dùng cho |
|-----------|-----|------------------|----------|
| 0 | CS0 / Best Effort | Default | Traffic thường (web, email) |
| 10 | AF11 | Assured Forwarding | Data backup, file transfer |
| 18 | AF21 | Assured Forwarding | Video streaming (buffered) |
| 26 | AF31 | Assured Forwarding | Signaling traffic |
| 34 | AF41 | Assured Forwarding | Video conferencing |
| 46 | EF | Expedited Forwarding | Voice (VoIP), real-time |
| 48 | CS6 | Class Selector | Network control (routing protocols) |

**Giải thích đơn giản:**
- **EF (46)** = Xe cứu thương — ưu tiên tuyệt đối, router xử lý trước
- **AF41 (34)** = Xe VIP — ưu tiên cao nhưng không tuyệt đối
- **CS0 (0)** = Xe thường — xếp hàng như mọi người

### ECN — Explicit Congestion Notification (2 bits cuối)

ECN là cơ chế cho phép router **báo trước** cho sender rằng "đường đang tắc, hãy giảm tốc" — THAY VÌ phải drop packet rồi mới biết.

| ECN bits | Ý nghĩa |
|----------|---------|
| 00 | Not ECN-Capable Transport (không hỗ trợ ECN) |
| 01 | ECT(1) — hỗ trợ ECN |
| 10 | ECT(0) — hỗ trợ ECN |
| 11 | CE — Congestion Experienced (router đánh dấu: "đang tắc!") |

### Trong thực tế

```bash
# Đặt DSCP = EF (46) cho VoIP traffic trên Linux
$ sudo iptables -t mangle -A OUTPUT -p udp --dport 5060 \
  -j DSCP --set-dscp 46

# Kiểm tra ToS value trong tcpdump
$ sudo tcpdump -i eth0 -v -c 5
# Output: ... tos 0xb8 ... (0xb8 = DSCP 46 = EF)
# Giải thích: 0xb8 = 1011 1000
#   DSCP = 101110 = 46 (EF)
#   ECN  = 00 (not ECN-capable)
```

### Trong AWS

- **AWS VPC Traffic Mirroring** giữ nguyên DSCP markings khi mirror traffic
- **AWS Direct Connect** hỗ trợ DSCP — bạn có thể đánh dấu traffic từ on-premises và AWS sẽ tôn trọng markings
- **Application Load Balancer (ALB)** và **Network Load Balancer (NLB)** pass-through DSCP values
- **Transit Gateway** cho phép cấu hình QoS dựa trên DSCP

---

## 5. Total Length, Identification, Flags, Fragment Offset — Hệ thống phân mảnh

### Mini example: Gửi một tủ quần áo qua bưu điện

Bạn muốn gửi một cái **tủ quần áo lớn** qua bưu điện, nhưng bưu điện chỉ nhận kiện hàng tối đa **50kg**. Bạn phải:

1. **Tháo tủ** thành 4 kiện nhỏ (fragmentation)
2. **Đánh số** mỗi kiện: "Kiện 1/4, 2/4, 3/4, 4/4" (Fragment Offset)
3. **Dán cùng mã đơn hàng** lên tất cả kiện (Identification)
4. **Ghi "Còn kiện sau"** trên kiện 1-3, và "Kiện cuối" trên kiện 4 (Flags: MF bit)

Đó chính là cách IPv4 fragmentation hoạt động!

### Total Length (16 bits)

- **Vị trí**: Bytes 3-4 của header
- **Giá trị**: Tổng kích thước packet (header + data), tính bằng bytes
- **Range**: 20 bytes (header only, no data) đến 65,535 bytes (2^16 - 1)

```
Total Length = IPv4 Header + Payload
             = 20 bytes    + (up to 65,515 bytes)
             = Tối đa 65,535 bytes

Ví dụ thực tế:
- Ping request: Total Length = 84 bytes (20 header + 8 ICMP header + 56 data)
- HTTP response: Total Length = 1500 bytes (MTU Ethernet thông thường)
```

**Tại sao cần Total Length?** Vì tầng data link (Ethernet) có thể thêm padding nếu frame quá nhỏ. Nhờ Total Length, tầng network biết chính xác payload thật dài bao nhiêu, bỏ qua padding.

### Identification (16 bits)

- **Vị trí**: Bytes 5-6
- **Giá trị**: Một số duy nhất (0-65,535) do sender gán cho mỗi packet
- **Mục đích**: Khi packet bị fragment, tất cả fragments có **cùng Identification** → receiver biết chúng thuộc cùng packet gốc

```
Packet gốc: ID = 12345, Total Length = 4000 bytes

Sau fragmentation (MTU = 1500):
Fragment 1: ID = 12345, Offset = 0,    Flags = MF (More Fragments)
Fragment 2: ID = 12345, Offset = 1480, Flags = MF
Fragment 3: ID = 12345, Offset = 2960, Flags = 0 (Last Fragment)
```

**Quan trọng (RFC 6864)**: Identification chỉ có ý nghĩa khi packet CÓ THỂ bị fragment. Nếu DF (Don't Fragment) flag = 1, trường này có thể bỏ qua.

### Flags (3 bits)

- **Bit 0**: Reserved (luôn = 0)
- **Bit 1**: DF (Don't Fragment)
  - `0` = Được phép fragment
  - `1` = KHÔNG được fragment — nếu packet quá lớn, router sẽ DROP và gửi ICMP "Fragmentation Needed"
- **Bit 2**: MF (More Fragments)
  - `0` = Đây là fragment cuối cùng (hoặc packet chưa bị fragment)
  - `1` = Còn fragments phía sau

```
┌────────────────────────────────────────────────────────────┐
│              FLAGS (3 bits)                                  │
├────────────────────────────────────────────────────────────┤
│  Bit 0: Reserved = 0                                        │
│  Bit 1: DF (Don't Fragment)                                │
│         0 = May Fragment                                    │
│         1 = Don't Fragment → Drop + ICMP error nếu quá lớn│
│  Bit 2: MF (More Fragments)                                │
│         0 = Last Fragment                                   │
│         1 = More Fragments Coming                           │
└────────────────────────────────────────────────────────────┘
```

### Fragment Offset (13 bits)

- **Đơn vị**: 8 bytes (không phải 1 byte!)
- **Giá trị**: Vị trí của fragment trong packet gốc
- **Range**: 0 đến 65,528 bytes (8191 × 8)

**Tại sao đơn vị là 8 bytes?** Vì 13 bits chỉ biểu diễn 0-8191. Nếu đơn vị là 1 byte, tối đa chỉ offset đến 8191 bytes — không đủ cho packet 65,535 bytes. Nhân 8 → tối đa 65,528 bytes. Điều này cũng có nghĩa là **mọi fragment (trừ fragment cuối) phải có kích thước data là bội số của 8 bytes**.

### Ví dụ fragmentation đầy đủ

```
Scenario: Gửi 4000 bytes data qua link có MTU = 1500

Packet gốc:
  Header: 20 bytes
  Data: 4000 bytes
  Total: 4020 bytes → Quá lớn cho MTU 1500!

Max data per fragment = MTU - Header = 1500 - 20 = 1480 bytes
1480 / 8 = 185 (chia hết → OK)

Fragment 1:
  Header: 20 bytes (IHL=5, ID=X, MF=1, Offset=0)
  Data: 1480 bytes (bytes 0-1479 của data gốc)
  Total Length: 1500

Fragment 2:
  Header: 20 bytes (IHL=5, ID=X, MF=1, Offset=185 [=1480/8])
  Data: 1480 bytes (bytes 1480-2959)
  Total Length: 1500

Fragment 3:
  Header: 20 bytes (IHL=5, ID=X, MF=0, Offset=370 [=2960/8])
  Data: 1040 bytes (bytes 2960-3999)
  Total Length: 1060
```

### Trong thực tế

```bash
# Kiểm tra MTU path
$ ping -M do -s 1472 google.com
# -M do = set DF bit (Don't Fragment)
# -s 1472 = payload 1472 bytes
# 1472 + 8 (ICMP header) + 20 (IP header) = 1500 = MTU Ethernet

# Nếu nhận "Frag needed and DF set" → có link MTU nhỏ hơn 1500

# Xem fragmentation bằng tcpdump
$ sudo tcpdump -i eth0 'ip[6:2] & 0x3fff != 0'
# Lọc chỉ packets đã bị fragment (offset != 0 hoặc MF = 1)
```

### Trong AWS

- **AWS VPC MTU mặc định**: 1500 bytes (Ethernet standard)
- **Jumbo frames trong VPC**: Hỗ trợ MTU 9001 bytes (cho traffic nội bộ VPC)
- **VPN tunnels**: MTU giảm do overhead (thường 1400-1436 bytes)
- **Path MTU Discovery**: AWS recommend bật DF bit để tránh fragmentation — fragmentation gây giảm performance vì:
  - Mỗi fragment cần header riêng → overhead
  - Mất 1 fragment = phải retransmit TOÀN BỘ packet gốc
  - Receiver phải buffer fragments cho đến khi nhận đủ

---

## 6. TTL và Protocol — Thời hạn sống và mục đích chuyến đi

### Mini example: Thẻ xe buýt có hạn

Bạn mua **thẻ xe buýt 30 lượt**. Mỗi lần lên xe, tài xế quẹt thẻ và **trừ 1 lượt**. Khi về 0 → thẻ hết hạn, bạn phải xuống xe. Nếu không có cơ chế này, một hành khách ngủ quên có thể đi vòng quanh thành phố mãi mãi, chiếm chỗ người khác.

**TTL (Time-to-Live)** hoạt động y hệt: mỗi router trên đường đi trừ 1. Khi TTL = 0 → packet bị hủy. Điều này ngăn packet "đi lạc" vĩnh viễn trong mạng (routing loop).

### TTL — Time to Live (8 bits)

- **Vị trí**: Byte thứ 9 (sau Fragment Offset)
- **Giá trị**: 1-255 (thực tế thường 64 hoặc 128)
- **Cơ chế**: Mỗi router giảm TTL xuống 1. Nếu TTL = 0 → DROP packet + gửi ICMP Time Exceeded về source

```
Ví dụ: Packet từ Việt Nam → Server ở Mỹ

Laptop (TTL=64) → Router ISP (TTL=63) → Router VN (TTL=62)
→ ... → Router US (TTL=51) → Server đích (TTL=50)

Qua 14 hops, TTL giảm từ 64 xuống 50.
```

**Giá trị khởi tạo phổ biến:**

| OS | TTL mặc định |
|-----|-------------|
| Linux | 64 |
| Windows | 128 |
| macOS | 64 |
| Cisco IOS | 255 |
| AWS EC2 (Amazon Linux) | 64 |

**Fun fact:** Bạn có thể đoán OS của server đích bằng TTL nhận được! Nếu ping google.com trả về TTL=115, thì TTL gốc là 128 (Windows), qua 13 hops. Nếu TTL=52, gốc là 64 (Linux), qua 12 hops.

### Ứng dụng TTL: Traceroute

Traceroute sử dụng TTL một cách thông minh:

```
Gửi packet 1: TTL=1 → Router đầu tiên DROP → trả ICMP "Time Exceeded"
                        → Bạn biết IP router 1!

Gửi packet 2: TTL=2 → Router 1 trừ thành TTL=1
                      → Router 2 trừ thành TTL=0 → DROP → ICMP
                        → Bạn biết IP router 2!

Gửi packet 3: TTL=3 → ... → Router 3 DROP
                        → Bạn biết IP router 3!

... tiếp tục cho đến khi packet đến đích
```

### Protocol (8 bits)

- **Vị trí**: Byte thứ 10
- **Mục đích**: Cho biết payload bên trong chứa data của protocol nào → receiver biết phải chuyển cho module xử lý nào

| Số | Protocol | Dùng cho |
|----|----------|----------|
| 1 | ICMP | Ping, traceroute, error messages |
| 6 | TCP | Web (HTTP/HTTPS), email, SSH, FTP |
| 17 | UDP | DNS, VoIP, video streaming, gaming |
| 47 | GRE | VPN tunneling |
| 50 | ESP | IPsec encrypted data |
| 51 | AH | IPsec authentication |
| 89 | OSPF | OSPF routing protocol |

**Analogy:** Protocol field giống như **dòng "Lý do nhập cảnh"** trên tờ khai hải quan. Hải quan (OS kernel) nhìn vào đó để quyết định chuyển bạn đến cửa nào (TCP module, UDP module, ICMP module...).

### Trong thực tế

```bash
# Traceroute sử dụng TTL
$ traceroute -n google.com
 1  192.168.1.1     1.234 ms   (router nhà)
 2  10.0.0.1        5.678 ms   (router ISP)
 3  203.162.4.1     10.123 ms  (backbone VN)
 ...
 14 142.250.190.46  45.678 ms  (Google server)

# Xem TTL và Protocol trong packet
$ sudo tcpdump -i eth0 -v -c 3
# Output: ... ttl 64, id 12345, proto TCP (6) ...
```

### Trong AWS

- **Security Groups** lọc traffic dựa trên **Protocol field** (TCP/UDP/ICMP/All traffic)
- **VPC Flow Logs** ghi lại protocol number trong mỗi flow record
- **Network ACLs** cho phép rule dựa trên protocol number
- **AWS Transit Gateway** xử lý TTL như router thông thường — trừ 1 mỗi hop
- **NAT Gateway** giữ nguyên TTL khi NAT

---

## 7. Header Checksum, Source/Destination IP, Options — Hoàn thành bức tranh

### Mini example: Số kiểm tra trên CMND/CCCD

Bạn có biết **2 số cuối** trên mã CMND/CCCD là **số kiểm tra** (check digit)? Nếu ai đó sửa 1 chữ số trong mã, 2 số cuối sẽ không khớp → phát hiện giả mạo. Header Checksum trong IPv4 hoạt động tương tự — nó phát hiện nếu header bị lỗi trong quá trình truyền.

### Header Checksum (16 bits)

- **Vị trí**: Bytes 11-12
- **Mục đích**: Phát hiện lỗi trong header (KHÔNG kiểm tra payload!)
- **Thuật toán**: One's complement sum of all 16-bit words in header

**Quy trình:**

```
1. Sender tính checksum:
   a. Đặt checksum field = 0
   b. Cộng tất cả 16-bit words trong header (one's complement)
   c. Lấy NOT (~) kết quả → đó là checksum
   d. Ghi checksum vào header

2. Mỗi router:
   a. Cộng tất cả 16-bit words (bao gồm checksum)
   b. Kết quả phải = 0xFFFF (all 1s)
   c. Nếu không → header bị corrupt → DROP packet
   d. Vì TTL thay đổi → router phải TÍNH LẠI checksum mỗi hop

3. Receiver: Kiểm tra giống router
```

**Tại sao chỉ checksum header, không checksum payload?**
- Layer 4 (TCP/UDP) đã có checksum riêng cho payload
- Checksum header phải được tính lại ở MỖI router (vì TTL thay đổi) → nếu bao gồm payload sẽ rất chậm
- IPv6 đã loại bỏ hoàn toàn header checksum (để tăng performance)

### Source IP Address (32 bits) và Destination IP Address (32 bits)

- **Vị trí**: Bytes 13-16 (Source), Bytes 17-20 (Destination)
- **Mục đích**: Định danh nguồn và đích của packet

```
Source IP: 192.168.1.100 = 11000000.10101000.00000001.01100100
Dest IP:   142.250.190.46 = 10001110.11111010.10111110.00101110

Trong header (hex): C0 A8 01 64 | 8E FA BE 2E
```

**Quy tắc quan trọng:**
- Source IP = IP của máy gửi (hoặc IP sau NAT)
- Destination IP = IP của máy nhận cuối cùng
- Router KHÔNG thay đổi Source/Dest IP (trừ khi làm NAT)
- Destination IP quyết định router chuyển packet đi đâu (routing decision)

### Options (0-40 bytes, variable)

Options là trường **tùy chọn** — hầu hết packets trên Internet ngày nay KHÔNG có options (IHL = 5). Tuy nhiên, đây là một số options quan trọng:

| Option | Type | Mục đích |
|--------|------|----------|
| Record Route | 7 | Ghi lại IP của mỗi router packet đi qua |
| Timestamp | 68 | Ghi lại thời gian tại mỗi hop |
| Loose Source Routing | 131 | Chỉ định packet PHẢI đi qua các router cụ thể |
| Strict Source Routing | 137 | Chỉ định chính xác path packet đi |

**Tại sao Options ít được dùng?**
1. **Performance**: Router phải kiểm tra options ở software path (chậm), thay vì hardware fast path
2. **Security**: Source Routing có thể bị lợi dụng để bypass firewall → hầu hết router CHẶN
3. **IPv6**: Thay thế options bằng Extension Headers — linh hoạt hơn

### Padding

Nếu Options không phải bội số của 4 bytes, **Padding** (toàn bit 0) được thêm vào để header kết thúc ở ranh giới 32-bit. Điều này đảm bảo data payload luôn bắt đầu ở vị trí aligned.

### Trong thực tế

```bash
# Xem source/destination IP
$ sudo tcpdump -i eth0 -n -c 5
14:30:01 IP 192.168.1.100.443 > 10.0.0.5.52341: Flags [P.]

# Kiểm tra packet có options không
$ sudo tcpdump -i eth0 'ip[0] & 0x0f > 5' -v
# Lọc packets có IHL > 5 (nghĩa là CÓ options)

# Record Route option (ghi lại đường đi)
$ ping -R google.com
# Hiển thị IP của các router trên đường đi (nếu router cho phép)
```

### Trong AWS

- **Elastic IP**: Khi bạn gán Elastic IP cho EC2, AWS thực hiện NAT — thay đổi Source IP trong header từ private IP sang public IP
- **VPC Endpoints**: Traffic đến S3/DynamoDB qua VPC Endpoint sử dụng private IPs trong cả Source và Destination
- **CloudFront**: Thay đổi Source IP thành IP của CloudFront edge location
- **X-Forwarded-For header**: Vì ALB thay đổi Source IP, original client IP được giữ trong HTTP header (không phải IPv4 header)

---

## 8. Tình huống thực tế — 3 scenarios chi tiết

### Scenario 1: Tại nhà — Tại sao ping có TTL khác nhau?

**Tình huống**: Bạn ở nhà, ping ba địa chỉ khác nhau và thấy TTL khác nhau:

```bash
$ ping 192.168.1.1        # Router nhà → TTL = 64
$ ping 8.8.8.8            # Google DNS → TTL = 115
$ ping amazon.com         # Amazon    → TTL = 237
```

**Phân tích dùng kiến thức IPv4 Header:**

1. **Router nhà (TTL=64)**: Router chạy Linux, TTL gốc = 64, packet đi 0 hops → TTL nhận = 64
2. **Google DNS (TTL=115)**: Server chạy Linux (TTL gốc = 128)... Khoan! 128 = Windows, 64 = Linux. TTL=115 → gốc = 128, qua 13 hops. Google dùng custom kernel, nhưng TTL gốc được set thành 128 cho DNS anycast.
3. **Amazon (TTL=237)**: TTL gốc = 255 (network device), qua 18 hops = 255-18=237. Amazon's load balancer set TTL=255.

**Bài học**: Bằng cách hiểu TTL, bạn có thể:
- Đoán OS của máy đích
- Ước tính số hops (khoảng cách mạng)
- Phát hiện routing loop (TTL drop đột ngột)

### Scenario 2: Trong công ty — VoIP call bị giật do sai DSCP

**Tình huống**: Công ty 500 nhân viên, VoIP phone system mới lắp. Mọi người phàn nàn: "Gọi điện IP bị giật, nghe rè".

**Phân tích:**
1. Capture traffic bằng Wireshark → phát hiện VoIP packets có DSCP = 0 (Best Effort)
2. VoIP cần DSCP = 46 (EF — Expedited Forwarding) để được ưu tiên
3. Switch/Router trong công ty có QoS policies dựa trên DSCP
4. Nhưng vì IP phones gửi DSCP = 0, traffic bị xếp chung hàng với download file, Windows Update → delay + jitter

**Giải pháp:**
```
1. Cấu hình IP Phones: set DSCP = 46 (EF) cho voice
2. Cấu hình Switch: trust DSCP từ voice VLAN
3. Cấu hình Router: priority queue cho DSCP 46
4. Verify: tcpdump kiểm tra ToS byte = 0xB8 (DSCP 46)
```

**Liên quan đến IPv4 Header**: Tất cả QoS decision trong mạng dựa trên **1 byte — Type of Service** trong IPv4 header. Đặt sai giá trị byte này = phone giật.

### Scenario 3: ISP — Path MTU Discovery failure gây website chậm

**Tình huống**: ISP nhận khiếu nại: "Khách hàng truy cập website A bình thường, nhưng website B rất chậm — load mãi không xong".

**Phân tích:**
1. Website B nằm sau VPN tunnel (overhead 20+ bytes) → effective MTU = 1400
2. Server B gửi packets với Total Length = 1500, DF = 1 (Don't Fragment)
3. Router VPN nhận packet 1500 bytes, không fit MTU 1400 → DROP + gửi ICMP "Fragmentation Needed" (Type 3, Code 4)
4. NHƯNG firewall giữa router VPN và Server B **chặn ICMP** → Server B không bao giờ nhận thông báo
5. Server B cứ gửi lại packet 1500 bytes → bị drop liên tục → **PMTU black hole**

**Giải pháp:**
```
Option A: Cho phép ICMP Type 3 qua firewall
Option B: Giảm MSS trên router VPN (TCP MSS clamping)
Option C: Tắt DF bit (cho phép fragmentation) — không khuyến khích

# TCP MSS Clamping trên Linux:
$ sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1360
```

**Liên quan đến IPv4 Header**: Vấn đề nằm ở 3 trường: **Total Length** (1500 > MTU), **Flags** (DF=1 cấm fragment), và ICMP response dựa trên thông tin trong header.

---

## 9. Bài tập thực hành

### Bài tập 1: Phân tích IPv4 Header bằng tay

Cho chuỗi hex của IPv4 header:
```
45 00 00 3c 1c 46 40 00 40 06 b1 e6 ac 10 0a 63 ac 10 0a 0c
```

**Yêu cầu**: Giải mã từng trường:

```
Byte 1: 45
  - Version = 4 (IPv4) ✓
  - IHL = 5 (20 bytes, no options) ✓

Byte 2: 00
  - ToS/DSCP = 0 (Best Effort)
  - ECN = 0 (Not ECN-capable)

Bytes 3-4: 00 3c
  - Total Length = 60 bytes

Bytes 5-6: 1c 46
  - Identification = 0x1C46 = 7238

Bytes 7-8: 40 00
  - Flags = 010 (DF=1, MF=0) → Don't Fragment
  - Fragment Offset = 0

Byte 9: 40
  - TTL = 64 (Linux!)

Byte 10: 06
  - Protocol = 6 (TCP)

Bytes 11-12: b1 e6
  - Header Checksum = 0xB1E6

Bytes 13-16: ac 10 0a 63
  - Source IP = 172.16.10.99

Bytes 17-20: ac 10 0a 0c
  - Destination IP = 172.16.10.12
```

### Bài tập 2: Tính fragmentation

**Scenario**: Gửi UDP packet 3200 bytes (data) qua link MTU = 1000 bytes. IP header = 20 bytes.

```
Max data per fragment = MTU - IP header = 1000 - 20 = 980 bytes
980 phải chia hết cho 8? 980 / 8 = 122.5 → KHÔNG!
→ Giảm xuống 976 (976 / 8 = 122) ✓

Fragment 1: Offset = 0,     Data = 976 bytes, MF=1, Total = 996
Fragment 2: Offset = 122,   Data = 976 bytes, MF=1, Total = 996
Fragment 3: Offset = 244,   Data = 976 bytes, MF=1, Total = 996
Fragment 4: Offset = 366,   Data = 272 bytes, MF=0, Total = 292
            (3200 - 976×3 = 272)

Verify: 976 + 976 + 976 + 272 = 3200 bytes ✓
```

### Bài tập 3: Traceroute và TTL analysis

```bash
# Chạy traceroute và phân tích
$ traceroute -n -q 1 8.8.8.8

# Bài tập:
# 1. Xác định bao nhiêu hops đến Google DNS
# 2. Hop nào có latency cao nhất? (bottleneck)
# 3. Có hop nào trả về * (timeout)? Tại sao?
#    → Router đó có thể CHẶN ICMP Time Exceeded
```

### Bài tập 4: Wireshark capture và phân tích

```bash
# Capture 100 packets
$ sudo tcpdump -i eth0 -c 100 -w /tmp/capture.pcap

# Mở bằng Wireshark hoặc tshark
$ tshark -r /tmp/capture.pcap -T fields \
  -e ip.version -e ip.hdr_len -e ip.dsfield.dscp \
  -e ip.len -e ip.id -e ip.flags.df -e ip.flags.mf \
  -e ip.frag_offset -e ip.ttl -e ip.proto \
  -e ip.src -e ip.dst

# Câu hỏi phân tích:
# 1. Bao nhiêu % packets có DF=1?
# 2. Có packet nào bị fragment không (MF=1 hoặc offset>0)?
# 3. TTL phổ biến nhất là gì? → Đoán OS
# 4. Protocol phổ biến nhất? (6=TCP, 17=UDP, 1=ICMP)
```

### Bài tập 5: MTU và Path MTU Discovery

```bash
# Tìm Path MTU đến một host
# Bắt đầu với size lớn, giảm dần cho đến khi không bị fragment

$ ping -M do -s 1472 target.com    # 1472 + 28 = 1500 (standard MTU)
$ ping -M do -s 1400 target.com    # Thử nếu 1472 fail
$ ping -M do -s 1300 target.com    # Tiếp tục giảm

# Binary search cho Path MTU:
# Nếu 1472 fail nhưng 1300 OK → thử 1386
# Nếu 1386 OK → thử 1429
# ... tiếp tục cho đến khi tìm được giá trị lớn nhất OK

# Script tự động:
for size in $(seq 1500 -10 1200); do
  ping -M do -s $size -c 1 -W 1 target.com > /dev/null 2>&1
  if [ $? -eq 0 ]; then
    echo "Path MTU = $((size + 28)) bytes"
    break
  fi
done
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Key Points — Những điểm cần nhớ

```
╔══════════════════════════════════════════════════════════════════╗
║                 IPv4 HEADER — TÓM TẮT                          ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. IPv4 Header = "Phong bì" của mỗi packet trên Internet      ║
║  2. Kích thước: 20-60 bytes (13 trường)                         ║
║  3. Version (4 bits) = luôn là 4                                ║
║  4. IHL (4 bits) = chiều dài header (đơn vị 4 bytes)           ║
║  5. DSCP (6 bits) = QoS class (EF=46 cho VoIP)                ║
║  6. Total Length (16 bits) = header + payload (max 65535)       ║
║  7. Fragmentation = ID + Flags + Offset (chia packet lớn)      ║
║  8. TTL (8 bits) = giảm 1 mỗi hop, = 0 → drop                ║
║  9. Protocol (8 bits) = TCP(6), UDP(17), ICMP(1)               ║
║  10. Checksum (16 bits) = kiểm tra lỗi CHỈ cho header         ║
║  11. Source + Dest IP (32+32 bits) = ai gửi, gửi cho ai       ║
║  12. Options = ít dùng, gây chậm, bị chặn                     ║
║  13. IPv6 đã đơn giản hóa: bỏ checksum, options, fragment     ║
║                                                                  ║
║  AWS Context:                                                    ║
║  • Security Groups đọc: Source IP, Dest IP, Protocol            ║
║  • NACLs đọc: Source IP, Dest IP, Protocol, Flags              ║
║  • VPC MTU: 1500 (default), 9001 (jumbo frames)                ║
║  • NAT Gateway: thay đổi Source IP trong header                ║
║  • Transit Gateway: giảm TTL như router thường                  ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### So sánh IPv4 Header vs IPv6 Header

| Đặc điểm | IPv4 | IPv6 |
|-----------|------|------|
| Header size | 20-60 bytes (variable) | 40 bytes (fixed) |
| Số trường | 13 | 8 |
| Checksum | Có (tính lại mỗi hop) | Không (bỏ để tăng speed) |
| Fragmentation | Router có thể fragment | CHỈ source mới fragment |
| Options | Trong header (gây chậm) | Extension Headers (riêng) |
| Address size | 32 bits | 128 bits |

### Bảng tham khảo nhanh — Protocol Numbers

| Number | Protocol | Typical Use |
|--------|----------|------------|
| 1 | ICMP | Ping, Traceroute |
| 2 | IGMP | Multicast group management |
| 6 | TCP | HTTP, HTTPS, SSH, SMTP |
| 17 | UDP | DNS, DHCP, VoIP, Gaming |
| 41 | IPv6-in-IPv4 | 6to4 tunneling |
| 47 | GRE | VPN tunneling |
| 50 | ESP | IPsec encryption |
| 51 | AH | IPsec authentication |
| 89 | OSPF | Link-state routing |
| 132 | SCTP | Telecom signaling |

### Tài liệu đọc thêm

| Tài liệu | Link/Reference | Nội dung |
|-----------|---------------|----------|
| RFC 791 | tools.ietf.org/html/rfc791 | Đặc tả gốc IPv4 |
| RFC 2474 | tools.ietf.org/html/rfc2474 | DSCP/DiffServ |
| RFC 3168 | tools.ietf.org/html/rfc3168 | ECN |
| RFC 6864 | tools.ietf.org/html/rfc6864 | Updated ID field |
| RFC 1191 | tools.ietf.org/html/rfc1191 | Path MTU Discovery |
| Stevens — TCP/IP Illustrated Vol.1 | Chapter 3 | IP Protocol chi tiết |
| Wireshark Wiki | wiki.wireshark.org/IPv4 | Capture analysis |

---

*Bài tiếp theo: [IPv6 Fundamentals](/2026-06-01-ipv6-fundamentals) — Tại sao thế giới cần IPv6 và nó khác IPv4 thế nào?*
