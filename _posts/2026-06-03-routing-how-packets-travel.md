---
layout: post
title: "Networking Fundamentals - Phần 3: Routing — Dữ liệu tìm đường như thế nào?"
subtitle: "Hiểu cách packets đi từ máy bạn đến server ở bên kia thế giới"
gh-repo: wayarmy/wayarmy.github.io
tags: [networking, aws, learning-path]
comments: true
date: 2026-06-03
categories: AWS-Learning-Path
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Tuần 3).
>
> **Đối tượng:** Người mới hoàn toàn — không cần kiến thức IT trước.
>
> **Nguồn tham khảo:**
> - RFC 791 (1981) — Internet Protocol
> - Wikipedia: [IP Routing](https://en.wikipedia.org/wiki/IP_routing), [Routing](https://en.wikipedia.org/wiki/Routing)
> - Khan Academy: [Internet Routing Protocol](https://www.khanacademy.org/a/internet-routing)
> - Cornell CS 4410: [Routing Lecture](http://www.cs.cornell.edu/courses/cs4410/2018su/lectures/lec22-routing.html)
> - Springer: "IP Routing Fundamentals" (Computer Networks textbook)

---

## 1. Routing là gì? — "Tìm đường" cho dữ liệu

### Ví dụ đời thường:

Bạn ở **Hà Nội** muốn gửi bưu phẩm đến bạn ở **Đà Nẵng**.

Bạn KHÔNG tự mang bưu phẩm đi — bạn đưa cho **bưu điện gần nhà**. Bưu điện nhìn địa chỉ đích và quyết định:
- "Đà Nẵng à? Tôi chuyển sang bưu điện trung chuyển miền Trung"
- Bưu điện trung chuyển nhận → "OK, chuyển tiếp sang bưu điện quận X, Đà Nẵng"
- Bưu điện quận X → giao cho shipper → đến đúng địa chỉ

**Mỗi "bưu điện" trên đường đi chính là một Router.**

Router KHÔNG biết đường đi hoàn chỉnh từ A đến Z. Nó chỉ biết: *"Bước tiếp theo (next hop) nên gửi đi đâu?"*. Giống như bạn hỏi đường — mỗi người chỉ cho bạn đi đến ngã tư tiếp theo, chứ không ai vẽ được bản đồ toàn bộ từ nhà bạn đến Đà Nẵng.

---

## 2. Default Gateway — "Cửa ra" đầu tiên

### Ví dụ đời thường:

Bạn sống trong một **khu chung cư** (mạng LAN). Khi bạn muốn gửi đồ cho người **trong cùng chung cư** (cùng subnet), bạn tự mang đến phòng họ.

Nhưng khi bạn muốn gửi đồ cho người **ở tòa nhà khác** (khác subnet/mạng), bạn phải ra **cổng chung cư** (default gateway) rồi nhờ bưu điện bên ngoài chuyển tiếp.

### Kỹ thuật:

**Default Gateway** = IP address của router gần nhất mà máy tính gửi tất cả traffic khi đích đến nằm **ngoài mạng LAN** của nó.

```
Máy bạn: 192.168.1.100/24
Gateway:  192.168.1.1 (router Wi-Fi nhà bạn)

Khi truy cập google.com (IP: 142.250.185.14):
→ 142.250.185.14 KHÔNG thuộc 192.168.1.0/24
→ Máy bạn gửi packet đến Gateway (192.168.1.1)
→ Router nhà bạn chuyển tiếp đến ISP router
→ ISP router chuyển tiếp... → đến Google server
```

### Cách máy quyết định:

Khi máy bạn muốn gửi packet đến IP address X:
1. Tính: X có **cùng subnet** với mình không? (dùng subnet mask AND)
2. **Nếu CÓ** (cùng mạng): gửi trực tiếp qua Layer 2 (dùng MAC address)
3. **Nếu KHÔNG** (khác mạng): gửi đến **Default Gateway** → gateway lo tiếp

---

## 3. Routing Table — "Bản đồ" của Router

### Ví dụ đời thường:

Mỗi "bưu điện" (router) có một **cuốn sổ** ghi:
- "Thư đi Đà Nẵng → chuyển qua cửa B cho trung tâm miền Trung"
- "Thư đi Sài Gòn → chuyển qua cửa C cho trung tâm miền Nam"
- "Thư đi nơi khác mà tôi không biết → chuyển qua cửa A (default)"

Cuốn sổ này chính là **Routing Table**.

### Cấu trúc Routing Table:

| Destination | Subnet Mask | Next Hop | Interface | Metric |
|-------------|-------------|----------|-----------|--------|
| 10.0.1.0 | /24 | trực tiếp | eth0 | 0 |
| 10.0.2.0 | /24 | 10.0.1.1 | eth0 | 1 |
| 172.16.0.0 | /16 | 10.0.1.254 | eth0 | 2 |
| 0.0.0.0 | /0 | 10.0.1.1 | eth0 | 10 |

**Giải thích từng cột:**
- **Destination:** Mạng đích (đi đâu?)
- **Subnet Mask:** Xác định phạm vi mạng đích
- **Next Hop:** Router tiếp theo cần chuyển đến (gửi cho ai?)
- **Interface:** Cổng vật lý nào trên router để gửi ra
- **Metric:** "Chi phí" đường đi — số nhỏ hơn = ưu tiên hơn

### Default Route (0.0.0.0/0):

Dòng `0.0.0.0/0` nghĩa là: *"Nếu không match bất kỳ route nào cụ thể → gửi đi hướng này"*.

Giống như biển báo: "Tất cả các hướng khác → thẳng tiến". Đây là **lối thoát cuối cùng** khi router không biết đường đi cụ thể.

---

## 4. Longest Prefix Match — Router chọn đường như thế nào?

### Vấn đề:

Nếu routing table có nhiều dòng match cùng một destination IP, router chọn dòng nào?

### Quy tắc: LONGEST PREFIX MATCH

Router chọn route có **prefix dài nhất** (cụ thể nhất) match với destination IP.

### Ví dụ:

Packet đến destination: `10.0.1.50`

| Route | Match? | Prefix length |
|-------|--------|---------------|
| 10.0.0.0/8 | ✅ (match) | /8 |
| 10.0.1.0/24 | ✅ (match) | /24 ← **CHỌN CÁI NÀY** |
| 0.0.0.0/0 | ✅ (match tất cả) | /0 |

Router chọn `/24` vì nó **cụ thể nhất** (longest = dài nhất = chi tiết nhất).

### Ví dụ đời thường:

Bưu điện nhận thư ghi: "Số 5, Phố Lạc Trung, Quận Hai Bà Trưng, Hà Nội"

Cuốn sổ có:
- "Hà Nội → chuyển trung tâm Hà Nội" (chung chung)
- "Quận Hai Bà Trưng → chuyển bưu điện quận HBT" (cụ thể hơn)
- "Phố Lạc Trung → chuyển cho shipper phố LT" (cụ thể nhất ← **CHỌN**)

---

## 5. Packet Journey — Hành trình thực tế của 1 packet

Bạn ở nhà, mở browser gõ `google.com`. Đây là hành trình:

### Bước 1: DNS Resolution (tìm IP của google.com)
```
Máy bạn → DNS server: "google.com có IP gì?"
DNS → Máy bạn: "142.250.185.14"
```

### Bước 2: Máy bạn tạo packet
```
Source IP: 192.168.1.100 (IP máy bạn)
Dest IP:   142.250.185.14 (IP Google)
```

### Bước 3: Kiểm tra — cùng subnet không?
```
Máy bạn: 192.168.1.100/24 → Network: 192.168.1.0
Google:   142.250.185.14   → KHÔNG thuộc 192.168.1.0/24
→ Gửi đến Default Gateway (192.168.1.1 = router Wi-Fi)
```

### Bước 4: ARP — tìm MAC address của Gateway
```
Máy bạn broadcast: "Ai có IP 192.168.1.1? Cho tôi MAC address"
Router trả lời: "Tôi đây! MAC: AA:BB:CC:DD:EE:FF"
```

### Bước 5: Gửi Frame đến Router nhà
```
Ethernet Frame:
  Dest MAC: AA:BB:CC:DD:EE:FF (MAC router)
  Source MAC: 11:22:33:44:55:66 (MAC máy bạn)
  Payload: [IP Packet: src=192.168.1.100, dst=142.250.185.14]
```

### Bước 6: Router nhà nhận → tra routing table
```
Router nhà routing table:
  192.168.1.0/24 → directly connected (LAN)
  0.0.0.0/0      → next hop: ISP router (203.113.x.x)

Dest 142.250.185.14 → match default route → forward to ISP
```

### Bước 7: NAT (Network Address Translation)
```
Trước NAT: src=192.168.1.100 (private - không routable)
Sau NAT:   src=113.185.x.x (public IP từ ISP)
Router thay đổi source IP từ private → public
```

### Bước 8: Qua nhiều routers (Internet)
```
ISP Router → Regional Router → Backbone Router → ...
   ... → Google's Edge Router → Google Server

Mỗi router:
1. Nhận packet
2. Đọc destination IP
3. Tra routing table
4. Forward đến next hop
5. (Thay đổi Layer 2 MAC addresses tại mỗi hop)
```

### Bước 9: Google Server nhận → xử lý → trả response
```
Google tạo response packet:
  Source IP: 142.250.185.14
  Dest IP:   113.185.x.x (public IP nhà bạn)

Response đi ngược lại qua Internet → ISP → Router nhà
Router nhà: NAT ngược (113.185.x.x → 192.168.1.100)
→ Giao đến máy bạn → Browser hiển thị Google.com
```

### Key insight:
- **IP addresses KHÔNG đổi** suốt hành trình (trừ NAT)
- **MAC addresses THAY ĐỔI** tại mỗi hop (vì Layer 2 chỉ dùng trong 1 link)
- **Router chỉ quyết định NEXT HOP** — không biết full path

---

## 6. Static vs Dynamic Routing

### Static Routing (Định tuyến tĩnh):

**Ví dụ:** Bạn **tự vẽ bản đồ** cho bưu điện: "Thư đi Đà Nẵng LUÔN đi qua cửa B"

- Admin **tự cấu hình** routing table bằng tay
- Đơn giản, dễ hiểu
- **Không tự thay đổi** khi network topology thay đổi
- Phù hợp: mạng nhỏ, topology đơn giản

```
# Ví dụ cấu hình static route trên Linux:
ip route add 10.0.2.0/24 via 10.0.1.1
```

### Dynamic Routing (Định tuyến động):

**Ví dụ:** Các bưu điện **tự trao đổi thông tin** với nhau: "Đường đi Đà Nẵng qua tôi mất 3 hop" → "Qua tôi chỉ mất 2 hop!" → Tự chọn đường ngắn nhất.

- Routers **tự động trao đổi** thông tin routing với nhau
- **Tự cập nhật** khi có thay đổi (link down, new network...)
- Phù hợp: mạng lớn, phức tạp

**Giao thức phổ biến:**
| Protocol | Loại | Dùng cho |
|----------|------|----------|
| **OSPF** | Link-state | Mạng nội bộ lớn |
| **BGP** | Path-vector | Internet (giữa các ISP, AWS Direct Connect) |
| **RIP** | Distance-vector | Mạng nhỏ (cũ, ít dùng) |

---

## 7. Routing trong AWS

### VPC Route Tables:

Trong AWS, mỗi **subnet** được gắn với một **Route Table**. Route table quyết định traffic đi đâu.

### Public Subnet Route Table:

| Destination | Target | Giải thích |
|-------------|--------|-----------|
| 10.0.0.0/16 | local | Traffic trong VPC → gửi nội bộ |
| 0.0.0.0/0 | igw-xxx | Mọi thứ khác → ra Internet qua Internet Gateway |

### Private Subnet Route Table:

| Destination | Target | Giải thích |
|-------------|--------|-----------|
| 10.0.0.0/16 | local | Traffic trong VPC → nội bộ |
| 0.0.0.0/0 | nat-xxx | Ra Internet → qua NAT Gateway (giữ private IP ẩn) |

### Phân biệt Public vs Private Subnet:

| | Public Subnet | Private Subnet |
|---|---|---|
| **Route 0.0.0.0/0** | → Internet Gateway | → NAT Gateway (hoặc không có) |
| **Public IP** | EC2 có thể nhận public IP | EC2 chỉ có private IP |
| **Truy cập từ Internet** | ✅ Có thể | ❌ Không thể (trực tiếp) |
| **Dùng cho** | ALB, Bastion host, NAT GW | App servers, databases |

### Ví dụ đời thường:
- **Public subnet** = Mặt tiền cửa hàng (khách hàng vào được từ đường)
- **Private subnet** = Kho hàng phía sau (chỉ nhân viên vào, khách không thấy)
- **NAT Gateway** = "Người đại diện" — kho hàng cần đặt hàng online (ra Internet) nhưng không muốn lộ địa chỉ kho

---

## 8. ARP — Cầu nối giữa Layer 3 và Layer 2

### Vấn đề:

Router biết cần gửi packet đến IP `10.0.1.5` (Layer 3). Nhưng để thực sự gửi frame qua dây mạng, cần biết **MAC address** (Layer 2) của máy đó. Ai giúp chuyển đổi IP → MAC?

### ARP (Address Resolution Protocol):

**Ví dụ đời thường:** Bạn biết **tên** bạn mình (IP address), nhưng cần biết **số điện thoại** (MAC address) để gọi. Bạn hét lên trong phòng: "Ai tên là Minh? Cho tôi số điện thoại!" → Minh trả lời: "Tôi đây! SĐT là 0912..."

```
1. Máy A broadcast: "Ai có IP 10.0.1.5? Reply MAC address cho tôi!" (ARP Request)
2. Máy có IP 10.0.1.5: "Tôi đây! MAC: CC:DD:EE:FF:00:11" (ARP Reply)
3. Máy A lưu vào ARP cache (bảng nhớ tạm) để lần sau không cần hỏi lại
```

**Xem ARP cache trên máy bạn:**
```bash
# Linux/Mac:
arp -a

# Windows:
arp -a
```

---

## 9. Bài tập thực hành

### Bài 1: Xem routing table trên máy bạn (5 phút)

```bash
# Linux/Mac:
ip route        # hoặc: netstat -rn

# Windows:
route print
```

Trả lời:
- Default gateway của bạn là gì?
- Có bao nhiêu routes?
- Route nào là "directly connected"?

### Bài 2: Traceroute — xem đường đi thực tế (5 phút)

```bash
# Linux/Mac:
traceroute google.com

# Windows:
tracert google.com
```

Mỗi dòng = 1 "hop" (1 router trên đường đi). Trả lời:
- Bao nhiêu hops từ máy bạn đến Google?
- Hop đầu tiên là gì? (thường là router Wi-Fi nhà bạn)
- Có hop nào timeout (*) không? Tại sao?

### Bài 3: Vẽ Packet Journey (20 phút)

Vẽ diagram cho scenario:
- Máy bạn (192.168.1.100/24, GW: 192.168.1.1) truy cập web server (10.0.1.50)
- Giữa có 3 routers

Ghi rõ tại mỗi hop:
- Source/Dest IP (có đổi không?)
- Source/Dest MAC (có đổi không?)
- Router tra routing table như thế nào?

### Bài 4: Design AWS Route Tables (15 phút)

Cho VPC 10.0.0.0/16 với:
- Public subnet: 10.0.1.0/24
- Private subnet: 10.0.10.0/24
- Internet Gateway: igw-abc
- NAT Gateway: nat-xyz (đặt trong public subnet)

Viết Route Table cho:
1. Public subnet
2. Private subnet

---

## 10. Tóm tắt

```
┌─────────────────────────────────────────────────────┐
│  ROUTING = Tìm "next hop" cho mỗi packet            │
│                                                       │
│  Default Gateway = Router gần nhất (cửa ra đầu tiên)│
│  Routing Table = Bảng chỉ đường (dest → next hop)   │
│  Longest Prefix Match = Chọn route cụ thể nhất      │
│  0.0.0.0/0 = Default route (đi đâu không biết →)    │
│                                                       │
│  Static routing = Admin tự cấu hình                  │
│  Dynamic routing = Routers tự trao đổi (OSPF, BGP)  │
│                                                       │
│  IP address: KHÔNG đổi suốt hành trình              │
│  MAC address: THAY ĐỔI tại mỗi hop                 │
│                                                       │
│  ARP: IP address → MAC address (cầu nối L3→L2)     │
│                                                       │
│  AWS:                                                │
│    Public subnet: 0.0.0.0/0 → IGW                   │
│    Private subnet: 0.0.0.0/0 → NAT GW              │
└─────────────────────────────────────────────────────┘
```

---

## Tài liệu đọc thêm

| Nguồn | Link | Ghi chú |
|-------|------|---------|
| RFC 791 (IP) | [rfc-editor.org](https://www.rfc-editor.org/rfc/rfc791.html) | Routing basics in IP spec |
| Wikipedia: IP Routing | [wikipedia.org](https://en.wikipedia.org/wiki/IP_routing) | Overview |
| Khan Academy: Internet Routing | [khanacademy.org](https://www.khanacademy.org/a/internet-routing) | Beginner-friendly |
| AWS VPC Route Tables | [docs.aws.amazon.com](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html) | Official docs |
| Wikipedia: Longest Prefix Match | [wikipedia.org](https://en.wikipedia.org/wiki/Longest_prefix_match) | Algorithm explained |

---

*Bài tiếp theo: [Networking Fundamentals - Phần 4: DNS — Hệ thống phân giải tên miền](/2026-06-04-dns-domain-name-system/) — Hiểu cách "google.com" biến thành IP address*
