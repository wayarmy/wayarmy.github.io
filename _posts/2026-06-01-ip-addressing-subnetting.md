---
layout: post
title: "Networking Fundamentals - Phần 2: IP Addressing & Subnetting"
subtitle: "Hiểu cách đánh 'địa chỉ' cho máy tính trên mạng — từ binary đến CIDR"
gh-repo: wayarmy/wayarmy.github.io
tags: [networking, aws, learning-path]
comments: true
date: 2026-06-01
categories: [networking]
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Tuần 2).
>
> **Đối tượng:** Người mới hoàn toàn. Bạn chỉ cần biết cộng trừ nhân chia.
>
> **Nguồn tham khảo:**
> - RFC 791 (1981) — Internet Protocol (định nghĩa IPv4)
> - RFC 4632 (2006) — Classless Inter-domain Routing (CIDR)
> - Cisco Documentation: [IP Addressing and Subnetting](https://www.cisco.com/c/en/us/support/docs/ip/routing-information-protocol-rip/13788-3.html)
> - Wikipedia: [IPv4](https://en.wikipedia.org/wiki/IPv4)
> - Textbook: "Computer Networks" — Andrew S. Tanenbaum, 5th Edition

---

## 1. Tại sao cần "địa chỉ" trên mạng?

### Ví dụ đời thường:

Bạn muốn gửi thư cho bạn mình. Bạn **phải ghi địa chỉ** trên phong bì — nếu không, bưu điện không biết giao đến đâu.

Trên mạng Internet cũng vậy:
- Mỗi máy tính, điện thoại, server cần có một **địa chỉ duy nhất** để các máy khác tìm được
- Địa chỉ này gọi là **IP address** (Internet Protocol address)
- Không có IP address → không thể gửi/nhận dữ liệu

**IP address chính là "địa chỉ nhà" của máy tính trên Internet.**

---

## 2. Hệ nhị phân (Binary) — Ngôn ngữ của máy tính

### Tại sao cần biết binary?

Máy tính **chỉ hiểu 0 và 1** — đây gọi là hệ nhị phân (binary). IP address thực chất là một chuỗi 32 số 0 và 1. Con người viết dạng thập phân (decimal) cho dễ đọc, nhưng máy tính xử lý dạng binary.

### Ví dụ đời thường:

Hãy nghĩ về **công tắc đèn**:
- Tắt = 0
- Bật = 1

Nếu bạn có **8 công tắc** xếp thành hàng, mỗi tổ hợp bật/tắt tạo ra một số khác nhau:

```
Công tắc:  [128] [64] [32] [16] [8] [4] [2] [1]
            Tắt   Tắt  Tắt  Tắt Tắt Tắt Tắt Tắt  = 0
            Bật   Tắt  Tắt  Tắt Tắt Tắt Tắt Tắt  = 128
            Bật   Bật  Tắt  Tắt Tắt Tắt Tắt Tắt  = 192
            Bật   Bật  Bật  Bật Bật Bật Bật Bật  = 255
```

Mỗi công tắc có giá trị **gấp đôi** cái bên phải nó: 1, 2, 4, 8, 16, 32, 64, 128.

### Cách chuyển đổi:

**Binary → Decimal (nhị phân → thập phân):**

```
11000000 = 128 + 64 + 0 + 0 + 0 + 0 + 0 + 0 = 192
10101000 = 128 + 0 + 32 + 0 + 8 + 0 + 0 + 0 = 168
00000001 = 0 + 0 + 0 + 0 + 0 + 0 + 0 + 1     = 1
01100100 = 0 + 64 + 32 + 0 + 0 + 4 + 0 + 0   = 100
```

Vậy: `11000000.10101000.00000001.01100100` = `192.168.1.100`

**Decimal → Binary (thập phân → nhị phân):**

Muốn chuyển số 200 sang binary:
```
200 ÷ 128 = 1 (dư 72)   → bit 1: 1
72  ÷ 64  = 1 (dư 8)    → bit 2: 1
8   ÷ 32  = 0 (nhỏ hơn) → bit 3: 0
8   ÷ 16  = 0 (nhỏ hơn) → bit 4: 0
8   ÷ 8   = 1 (dư 0)    → bit 5: 1
0   ÷ 4   = 0            → bit 6: 0
0   ÷ 2   = 0            → bit 7: 0
0   ÷ 1   = 0            → bit 8: 0
```
Kết quả: 200 = `11001000`

---

## 3. IPv4 Address — Cấu trúc

### Định nghĩa (theo RFC 791):

IPv4 address là một số **32-bit** (32 chữ số 0/1), được chia thành **4 nhóm** (gọi là **octet**), mỗi nhóm 8 bit, viết dạng thập phân cách nhau bằng dấu chấm.

```
Dạng binary:   11000000.10101000.00000001.01100100
Dạng decimal:  192     .168     .1       .100
               ← octet1 →← octet2 →← octet3 →← octet4 →
```

### Đặc điểm quan trọng:
- Mỗi octet có 8 bit → giá trị từ **0 đến 255** (vì 2^8 = 256 giá trị)
- Tổng cộng 32 bit → khoảng **4.3 tỷ** địa chỉ khả dụng (2^32 = 4,294,967,296)
- Đã **gần cạn kiệt** → đó là lý do IPv6 (128-bit) ra đời

### Ví dụ đời thường — So sánh với địa chỉ nhà:

| IP Address | Giống như địa chỉ |
|-----------|-------------------|
| `192.168.1.100` | Số 100, phố 1, phường 168, quận 192 |
| `10.0.3.50` | Số 50, phố 3, phường 0, quận 10 |

Phần đầu = khu vực (network), phần sau = nhà cụ thể (host).

---

## 4. Network Portion vs Host Portion

### Ý tưởng cốt lõi:

Mỗi IP address được chia làm **2 phần**:
- **Network portion** (phần mạng): Xác định bạn thuộc **mạng nào** — giống "tên đường/phường"
- **Host portion** (phần host): Xác định bạn là **máy nào** trong mạng đó — giống "số nhà"

### Ví dụ đời thường:

Địa chỉ: **"Số 15, Phố Lạc Trung, Hà Nội"**
- "Phố Lạc Trung, Hà Nội" = Network (xác định khu vực)
- "Số 15" = Host (xác định nhà cụ thể)

Tất cả nhà trên "Phố Lạc Trung" đều chia sẻ **cùng network**, chỉ khác **số nhà (host)**.

### Subnet Mask — "Ranh giới" giữa Network và Host

**Subnet mask** cho biết: "Trong 32 bit IP address, bao nhiêu bit đầu là Network, bao nhiêu bit sau là Host?"

```
IP address:   192.168.1.100
Subnet mask:  255.255.255.0

Binary:
IP:    11000000.10101000.00000001.01100100
Mask:  11111111.11111111.11111111.00000000
       ←—— Network (24 bit) ——→←Host 8bit→

→ Network: 192.168.1.0 (phần bị mask = "1")
→ Host:    .100 (phần mask = "0")
```

**Quy tắc:** Bit mask = 1 → thuộc Network. Bit mask = 0 → thuộc Host.

### Cách xác định Network ID:

Thực hiện phép **AND** (nhân) giữa IP và Mask:

```
IP:     11000000.10101000.00000001.01100100  (192.168.1.100)
Mask:   11111111.11111111.11111111.00000000  (255.255.255.0)
AND:    11000000.10101000.00000001.00000000  (192.168.1.0) ← Network ID
```

Quy tắc AND: 1 AND 1 = 1, mọi trường hợp khác = 0.

---

## 5. CIDR Notation — Cách viết tắt Subnet Mask

### Vấn đề:
Viết `255.255.255.0` dài dòng. Có cách ngắn hơn không?

### CIDR (Classless Inter-Domain Routing):

Theo **RFC 4632**, thay vì viết full subnet mask, ta chỉ ghi **số bit "1" trong mask** sau dấu `/`:

| Subnet Mask | CIDR | Số bit Network | Số bit Host | Số IP addresses |
|-------------|------|----------------|-------------|-----------------|
| 255.0.0.0 | /8 | 8 | 24 | 16,777,216 |
| 255.255.0.0 | /16 | 16 | 16 | 65,536 |
| 255.255.255.0 | /24 | 24 | 8 | 256 |
| 255.255.255.128 | /25 | 25 | 7 | 128 |
| 255.255.255.192 | /26 | 26 | 6 | 64 |
| 255.255.255.224 | /27 | 27 | 5 | 32 |
| 255.255.255.240 | /28 | 28 | 4 | 16 |
| 255.255.255.252 | /30 | 30 | 2 | 4 |
| 255.255.255.255 | /32 | 32 | 0 | 1 |

### Công thức tính số IP:

```
Số IP = 2^(32 - prefix)
Số IP usable = 2^(32 - prefix) - 2
```

**Tại sao trừ 2?**
- 1 địa chỉ dành cho **Network ID** (tất cả host bits = 0): không gán cho máy nào
- 1 địa chỉ dành cho **Broadcast** (tất cả host bits = 1): gửi cho tất cả máy trong mạng

### Ví dụ:

`192.168.1.0/24`:
- Số IP tổng: 2^(32-24) = 2^8 = **256**
- Số IP usable: 256 - 2 = **254** (gán được cho 254 máy)
- Network ID: 192.168.1.0 (không gán)
- Broadcast: 192.168.1.255 (không gán)
- Range usable: 192.168.1.1 → 192.168.1.254

### Ví dụ đời thường — CIDR như "quy mô khu phố":

| CIDR | Ví dụ | Giống như |
|------|-------|----------|
| /8 | 10.0.0.0/8 | Cả một thành phố (~16 triệu nhà) |
| /16 | 10.0.0.0/16 | Một quận (~65,000 nhà) |
| /24 | 10.0.1.0/24 | Một con phố (254 nhà) |
| /28 | 10.0.1.0/28 | Một ngõ nhỏ (14 nhà) |
| /32 | 10.0.1.5/32 | Một nhà cụ thể (đúng 1 địa chỉ) |

---

## 6. Private vs Public IP Addresses

### Vấn đề:

4.3 tỷ IPv4 address nghe nhiều, nhưng với hàng tỷ thiết bị trên Internet thì **không đủ**. Giải pháp: không phải mọi thiết bị cần IP "thật" (public). Các thiết bị trong nhà/công ty chỉ cần IP "nội bộ" (private).

### Ví dụ đời thường:

Trong chung cư:
- **Số phòng** (101, 102, 203...) = **Private IP** — chỉ có ý nghĩa trong tòa nhà đó
- **Địa chỉ tòa nhà** (số 50 Lạc Trung) = **Public IP** — dùng để giao tiếp với bên ngoài

Nhiều tòa nhà đều có "phòng 101" — không sao, vì nó chỉ có ý nghĩa nội bộ.

### Private IP Ranges (RFC 1918):

| Range | CIDR | Số IP | Dùng cho |
|-------|------|-------|----------|
| 10.0.0.0 → 10.255.255.255 | 10.0.0.0/8 | ~16.7 triệu | Mạng lớn (công ty, data center) |
| 172.16.0.0 → 172.31.255.255 | 172.16.0.0/12 | ~1 triệu | Mạng trung bình |
| 192.168.0.0 → 192.168.255.255 | 192.168.0.0/16 | ~65,000 | Mạng nhỏ (nhà, quán cafe) |

### Đặc điểm Private IP:
- **Không thể** truy cập trực tiếp từ Internet
- **Có thể** trùng với mạng khác (nhà bạn dùng 192.168.1.x, nhà tôi cũng dùng 192.168.1.x — không conflict)
- Cần **NAT** (Network Address Translation) để ra Internet

### Public IP:
- Mỗi địa chỉ là **duy nhất toàn cầu**
- Do tổ chức IANA/RIR (Regional Internet Registry) quản lý và phân phối
- ISP (nhà mạng) cấp cho bạn

### Trong AWS:
- **VPC** chỉ dùng **private IP ranges** (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
- EC2 instance có **private IP** (luôn có) + **public IP** (tùy chọn, cho truy cập từ Internet)
- **Elastic IP** = public IP cố định (không đổi khi stop/start instance)

---

## 7. Subnetting — Chia mạng lớn thành mạng nhỏ

### Tại sao cần subnet?

#### Ví dụ đời thường:

Bạn quản lý một tòa nhà 1000 phòng. Nếu tất cả 1000 phòng dùng **chung 1 hệ thống loa thông báo** — khi thông báo cho phòng 501, cả 1000 phòng đều nghe. Rất ồn ào và lãng phí!

**Giải pháp:** Chia tòa nhà thành **nhiều tầng** — mỗi tầng có hệ thống loa riêng. Thông báo cho tầng 5 chỉ tầng 5 nghe.

Subnetting cũng vậy:
- **Mạng lớn** (VD: /16 = 65,536 IPs) → chia thành nhiều **mạng nhỏ** (subnets)
- Mỗi subnet hoạt động **độc lập** — traffic trong subnet A không ảnh hưởng subnet B
- Tăng **bảo mật**: có thể kiểm soát traffic giữa các subnets

### Cách chia subnet:

**Bài toán:** Bạn có `10.0.0.0/16` (65,536 IPs). Muốn chia thành 4 subnets bằng nhau.

**Bước 1:** /16 có 16 host bits. Chia 4 = cần "mượn" 2 bit (vì 2^2 = 4).

**Bước 2:** Prefix mới = 16 + 2 = **/18** (mỗi subnet có 2^14 = 16,384 IPs)

**Kết quả:**

| Subnet | Network ID | Range | Broadcast |
|--------|-----------|-------|-----------|
| 1 | 10.0.0.0/18 | 10.0.0.1 → 10.0.63.254 | 10.0.63.255 |
| 2 | 10.0.64.0/18 | 10.0.64.1 → 10.0.127.254 | 10.0.127.255 |
| 3 | 10.0.128.0/18 | 10.0.128.1 → 10.0.191.254 | 10.0.191.255 |
| 4 | 10.0.192.0/18 | 10.0.192.1 → 10.0.255.254 | 10.0.255.255 |

### Trong AWS — VPC Subnet Design:

AWS VPC thường được thiết kế theo pattern:

```
VPC: 10.0.0.0/16 (65,536 IPs)
├── Public Subnet AZ-a:  10.0.1.0/24 (256 IPs) — cho ALB, NAT GW
├── Public Subnet AZ-b:  10.0.2.0/24 (256 IPs)
├── Public Subnet AZ-c:  10.0.3.0/24 (256 IPs)
├── Private Subnet AZ-a: 10.0.10.0/24 (256 IPs) — cho EC2 app servers
├── Private Subnet AZ-b: 10.0.20.0/24 (256 IPs)
├── Private Subnet AZ-c: 10.0.30.0/24 (256 IPs)
├── DB Subnet AZ-a:      10.0.100.0/24 (256 IPs) — cho RDS
├── DB Subnet AZ-b:      10.0.110.0/24 (256 IPs)
└── DB Subnet AZ-c:      10.0.120.0/24 (256 IPs)
```

**Tại sao chia như vậy?**
- **Public subnet**: Có route đến Internet Gateway → truy cập từ Internet
- **Private subnet**: Không có route đến IGW → chỉ truy cập nội bộ (qua NAT nếu cần ra Internet)
- **DB subnet**: Cách ly hoàn toàn — chỉ private subnet có thể kết nối đến
- **Multi-AZ**: Mỗi loại subnet có 3 bản ở 3 Availability Zones → High Availability

### Lưu ý trong AWS:
- AWS **giữ 5 IPs** trong mỗi subnet (không dùng được):
  - .0 = Network ID
  - .1 = VPC Router
  - .2 = DNS Server
  - .3 = Reserved (future use)
  - .255 = Broadcast (AWS không hỗ trợ broadcast nhưng vẫn reserve)
- Vậy subnet /24 (256 IPs) thực tế chỉ dùng được **251 IPs**
- VPC CIDR phải nằm trong range: **/16 → /28**

---

## 8. Các địa chỉ đặc biệt

| Địa chỉ | Ý nghĩa | Ví dụ |
|----------|---------|-------|
| **Network ID** | Đại diện cho cả mạng (host bits = 0) | 192.168.1.0/24 |
| **Broadcast** | Gửi cho TẤT CẢ máy trong mạng (host bits = 1) | 192.168.1.255 |
| **Loopback** | "Nói chuyện với chính mình" | 127.0.0.1 (localhost) |
| **0.0.0.0** | "Tất cả mạng" / Default route | Route table: 0.0.0.0/0 → IGW |
| **169.254.x.x** | APIPA — tự gán khi không có DHCP | Link-local |

---

## 9. Bài tập thực hành

### Bài 1: Binary ↔ Decimal conversion (10 phút)

Chuyển đổi (không dùng máy tính):

| Decimal → Binary | Binary → Decimal |
|------------------|------------------|
| 192 = ? | 11010101 = ? |
| 240 = ? | 10110000 = ? |
| 172 = ? | 01111111 = ? |
| 100 = ? | 11111100 = ? |

### Bài 2: Xác định Network ID (10 phút)

Cho IP và mask, tìm Network ID:

| IP Address | Subnet Mask | Network ID = ? |
|-----------|-------------|----------------|
| 192.168.10.50 | /24 | ? |
| 10.0.5.100 | /16 | ? |
| 172.16.20.200 | /20 | ? |
| 192.168.1.130 | /25 | ? |

### Bài 3: Tính range cho subnet (15 phút)

Cho network, tính: Network ID, First usable, Last usable, Broadcast, Số IP usable:

1. `10.0.0.0/24`
2. `192.168.5.0/26`
3. `172.16.0.0/20`

### Bài 4: Design VPC (20 phút)

Bạn được giao VPC `10.0.0.0/16`. Hãy thiết kế:
- 3 public subnets (mỗi cái /24)
- 3 private subnets (mỗi cái /20 — vì cần nhiều EC2 hơn)
- 3 database subnets (mỗi cái /24)

Ghi ra: CIDR cho mỗi subnet, số IP usable, và lý do chọn size đó.

---

## 10. Tóm tắt & Công thức cần nhớ

```
┌────────────────────────────────────────────────────────┐
│  IP address = 32 bits = 4 octets (0-255 mỗi octet)   │
│  Subnet mask: bit 1 = Network, bit 0 = Host           │
│  CIDR /N: N bits đầu = Network, (32-N) bits = Host    │
│                                                         │
│  Số IP tổng     = 2^(32 - prefix)                      │
│  Số IP usable   = 2^(32 - prefix) - 2                  │
│  Số IP usable AWS = 2^(32 - prefix) - 5                │
│                                                         │
│  Network ID   = IP AND Mask (host bits → 0)            │
│  Broadcast    = Network ID với host bits → 1           │
│  First usable = Network ID + 1                          │
│  Last usable  = Broadcast - 1                           │
│                                                         │
│  Private ranges: 10.0.0.0/8, 172.16.0.0/12,           │
│                  192.168.0.0/16                          │
│  AWS VPC range: /16 → /28                               │
└────────────────────────────────────────────────────────┘
```

---

## Tài liệu đọc thêm

| Nguồn | Link | Ghi chú |
|-------|------|---------|
| RFC 791 (IPv4) | [rfc-editor.org](https://www.rfc-editor.org/rfc/rfc791.html) | IP specification gốc |
| RFC 4632 (CIDR) | [rfc-editor.org](https://www.rfc-editor.org/rfc/rfc4632.html) | Classless routing |
| RFC 1918 (Private IPs) | [rfc-editor.org](https://www.rfc-editor.org/rfc/rfc1918.html) | Private address ranges |
| Cisco: IP Subnetting | [cisco.com](https://www.cisco.com/c/en/us/support/docs/ip/routing-information-protocol-rip/13788-3.html) | Ví dụ thực tế |
| Subnetting Practice | [subnettingpractice.com](https://subnettingpractice.com) | Luyện tập online |
| AWS VPC CIDR | [docs.aws.amazon.com](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-cidr-blocks.html) | VPC sizing |

---

*Bài tiếp theo: [Networking Fundamentals - Phần 3: Routing & How Packets Travel](/2026-06-01-routing-how-packets-travel/) — Hiểu cách dữ liệu tìm đường đi từ máy bạn đến server*
