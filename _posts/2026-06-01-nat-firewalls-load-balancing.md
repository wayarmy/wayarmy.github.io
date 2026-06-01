---
layout: post
title: "Networking Fundamentals - Phần 6: NAT, Firewalls và Load Balancing"
subtitle: "Bảo vệ mạng nội bộ và phân phối tải hiệu quả"
gh-repo: wayarmy/wayarmy.github.io
tags: [networking, aws, learning-path]
comments: true
date: 2026-06-01
categories: [networking]
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Phần 6).
>
> **Đối tượng:** Người mới hoàn toàn — không cần kiến thức IT trước.
>
> **Nguồn tham khảo:**
> - RFC 3022 (2001) — Traditional IP Network Address Translator (Traditional NAT)
> - RFC 2663 (1999) — IP Network Address Translator (NAT) Terminology and Considerations
> - Cisco Documentation: [NAT Configuration Guide](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/)
> - Cisco Documentation: [Zone-Based Policy Firewall](https://www.cisco.com/c/en/us/support/docs/security/)
> - AWS Documentation: [NAT Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html), [NACL](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html), [Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
> - Stevens, W.R. "TCP/IP Illustrated, Volume 1" — Chapter 18

---

## 1. NAT — "Dịch địa chỉ" để ra Internet

### Vấn đề: Tại sao cần NAT?

Nhớ lại bài về IP addressing: IPv4 chỉ có khoảng 4.3 tỷ địa chỉ public. Với hàng chục tỷ thiết bị trên thế giới, KHÔNG ĐỦ IP public cho tất cả.

Giải pháp: Dùng **IP private** bên trong mạng nội bộ (10.x.x.x, 172.16-31.x.x, 192.168.x.x) và **NAT** để "dịch" thành IP public khi ra Internet.

### Ví dụ đời thường — NAT giống "Tổng đài công ty":

Hãy tưởng tượng **công ty có 500 nhân viên** nhưng chỉ có **1 số điện thoại đối ngoại** (1900-xxxx):
- Nhân viên A (ext 101) gọi ra ngoài → Đối tác thấy số `1900-xxxx` (không thấy ext 101)
- Nhân viên B (ext 205) gọi ra ngoài → Đối tác cũng thấy `1900-xxxx`
- Đối tác gọi lại → Tổng đài nhìn bảng ghi "cuộc gọi này đang nối với ext 101" → chuyển cho A

**Tổng đài = NAT device (Router)**
**Extension nội bộ = IP private**
**Số đối ngoại = IP public**

### Các loại NAT (RFC 3022, Section 2):

#### 1. SNAT (Source NAT) — "Đổi số người gọi"

Khi packet **đi ra** Internet, NAT thay đổi **Source IP** từ private thành public.

```
Trước NAT:                          Sau NAT:
┌──────────────────┐                ┌──────────────────┐
│ Src: 192.168.1.5 │  ──→ NAT ──→  │ Src: 203.0.113.1 │
│ Dst: 8.8.8.8    │                │ Dst: 8.8.8.8    │
└──────────────────┘                └──────────────────┘
```

**Ví dụ đời thường:** Bạn gửi thư từ nhà riêng (địa chỉ nội bộ khu đô thị) nhưng trên phong bì ghi **địa chỉ công ty** (địa chỉ public) làm địa chỉ gửi — để người nhận reply về công ty.

#### 2. DNAT (Destination NAT) — "Đổi số người nhận"

Khi packet **đi vào** từ Internet, NAT thay đổi **Destination IP** từ public thành private server bên trong.

```
Trước NAT:                          Sau NAT:
┌──────────────────┐                ┌──────────────────┐
│ Src: 8.8.4.4    │  ──→ NAT ──→  │ Src: 8.8.4.4    │
│ Dst: 203.0.113.1│                │ Dst: 192.168.1.10│
└──────────────────┘                └──────────────────┘
```

**Ví dụ:** Khách hàng gọi đến số tổng đài 1900-xxxx → tổng đài chuyển cuộc gọi vào phòng Chăm sóc khách hàng (extension nội bộ).

**Use case phổ biến:** Port forwarding — khi bạn muốn cho Internet truy cập web server ở nhà:
```
Internet → Router (203.0.113.1:80) → DNAT → Internal web server (192.168.1.10:80)
```

#### 3. PAT (Port Address Translation) / NAT Overload — "Nhiều người dùng chung 1 số"

Đây là loại NAT **phổ biến nhất** — cho phép NHIỀU IP private chia sẻ MỘT IP public bằng cách dùng **port number** để phân biệt.

```
PC1 (192.168.1.5:12345)  ──┐
                            ├──→ NAT Router ──→ Internet
PC2 (192.168.1.6:12345)  ──┘     (203.0.113.1)

NAT Table:
┌──────────────────────┬───────────────────────┬─────────────────┐
│ Internal             │ External              │ Destination     │
├──────────────────────┼───────────────────────┼─────────────────┤
│ 192.168.1.5:12345    │ 203.0.113.1:40001     │ 8.8.8.8:53     │
│ 192.168.1.6:12345    │ 203.0.113.1:40002     │ 8.8.8.8:53     │
│ 192.168.1.5:12346    │ 203.0.113.1:40003     │ 1.1.1.1:443    │
└──────────────────────┴───────────────────────┴─────────────────┘
```

**Giải thích:** Router NAT gán **port khác nhau** cho mỗi kết nối. Khi response quay lại port 40001, router biết phải chuyển cho PC1; port 40002 → PC2.

**Ví dụ:** WiFi nhà bạn có 10 thiết bị (laptop, phone, TV, máy giặt IoT...) nhưng nhà mạng chỉ cấp **1 IP public**. Router dùng PAT để tất cả chia sẻ 1 IP đó.

### NAT Translation Table — "Sổ ghi chép" của router:

Router NAT duy trì một bảng ghi lại mapping giữa internal ↔ external. Bảng này tạo khi packet đi RA và xóa khi hết timeout (thường 5 phút cho UDP, vài giờ cho TCP).

```bash
# Xem NAT table trên Cisco router
Router# show ip nat translations
Pro Inside global      Inside local       Outside local      Outside global
tcp 203.0.113.1:40001  192.168.1.5:12345  8.8.8.8:53         8.8.8.8:53
tcp 203.0.113.1:40002  192.168.1.6:12345  8.8.8.8:53         8.8.8.8:53
```

### Hạn chế của NAT:

| Hạn chế | Giải thích |
|---------|-----------|
| Phá vỡ end-to-end | Máy bên ngoài không thể khởi tạo kết nối vào máy bên trong (trừ khi cấu hình DNAT/port forward) |
| Phức tạp hóa protocol | Một số protocol nhúng IP trong payload (FTP, SIP) → NAT phải inspect sâu |
| Không hoàn toàn bảo mật | NAT che giấu topology nhưng KHÔNG phải firewall — nó không filter traffic |
| Performance overhead | Router phải rewrite header + maintain table cho mỗi connection |

---

## 2. Firewalls — "Tường lửa" bảo vệ mạng

### Ví dụ đời thường:

Firewall giống **bảo vệ + cổng kiểm soát** ở khu chung cư:
- **Kiểm tra người vào:** Có thẻ cư dân? Có được mời? → Cho vào
- **Kiểm tra đồ mang vào:** Hàng cấm? Vật liệu nguy hiểm? → Từ chối
- **Ghi sổ:** Ai vào lúc nào, ra lúc nào

### Stateless Firewall — "Bảo vệ máy móc"

**Stateless = Không nhớ trạng thái.** Mỗi packet được kiểm tra ĐỘC LẬP — firewall không biết packet này thuộc cuộc hội thoại nào.

**Ví dụ:** Giống như **máy quét thẻ tự động** ở cửa:
- Có thẻ hợp lệ → mở cửa
- Không có thẻ → khóa cửa
- Máy KHÔNG nhớ bạn vừa đi ra 5 giây trước → khi quay lại vẫn phải quẹt thẻ

**Cách hoạt động:**
```
Rule 1: ALLOW TCP dst-port 80 from 0.0.0.0/0   (cho vào port 80)
Rule 2: ALLOW TCP src-port 80 to 0.0.0.0/0     (cho response ra port 80)
Rule 3: DENY ALL                                (chặn mọi thứ khác)
```

**Bạn phải viết RIÊNG rule cho traffic đi VÀO và traffic đi RA!** Vì stateless firewall không tự hiểu "đây là response của request trước đó".

### Stateful Firewall — "Bảo vệ thông minh"

**Stateful = Nhớ trạng thái kết nối.** Firewall theo dõi toàn bộ **connection state** (TCP handshake, data flow, termination).

**Ví dụ:** Giống **bảo vệ con người**:
- Bạn đi RA cổng → bảo vệ GHI NHỚ "anh A vừa đi ra"
- 10 phút sau bạn quay lại → bảo vệ: "À, anh A, tôi nhớ anh vừa đi ra" → mở cửa luôn
- Người lạ đến → "Ai đây? Chưa thấy bao giờ" → kiểm tra kỹ

**Cách hoạt động:**
```
Rule 1: ALLOW TCP dst-port 80 from 0.0.0.0/0   (cho vào port 80)
        → Tự động cho response RA mà KHÔNG cần rule riêng!
Rule 2: DENY ALL
```

**Connection Tracking Table:**
```
┌─────────┬───────────────────┬────────────────────┬───────────┬─────────┐
│ Proto   │ Source            │ Destination        │ State     │ Timeout │
├─────────┼───────────────────┼────────────────────┼───────────┼─────────┤
│ TCP     │ 8.8.4.4:54321     │ 192.168.1.10:80    │ ESTABLISHED│ 3600s  │
│ TCP     │ 1.2.3.4:12345     │ 192.168.1.10:443   │ SYN_RECV  │ 60s    │
│ UDP     │ 10.0.0.5:5000     │ 8.8.8.8:53         │ ACTIVE    │ 30s    │
└─────────┴───────────────────┴────────────────────┴───────────┴─────────┘
```

### So sánh Stateless vs Stateful:

| Đặc điểm | Stateless | Stateful |
|-----------|-----------|----------|
| Nhớ kết nối? | Không | Có |
| Cần rule cho response? | Có (phải viết riêng) | Không (tự động) |
| Tốc độ | Nhanh hơn (ít logic) | Chậm hơn (maintain state) |
| Bảo mật | Thấp hơn | Cao hơn (hiểu context) |
| Phức tạp rule | Nhiều rule hơn | Ít rule hơn |
| AWS equivalent | **NACL** | **Security Group** |

### Firewall Rules — Nguyên tắc viết:

Rules được xử lý **theo thứ tự** (top to bottom). Packet match rule đầu tiên → thực hiện action → dừng.

```
# Ví dụ rule set:
Priority 100: ALLOW TCP 22   from 10.0.0.0/8       ← SSH từ mạng nội bộ
Priority 200: ALLOW TCP 80   from 0.0.0.0/0        ← HTTP từ mọi nơi
Priority 300: ALLOW TCP 443  from 0.0.0.0/0        ← HTTPS từ mọi nơi
Priority 400: DENY  TCP 22   from 0.0.0.0/0        ← Chặn SSH từ bên ngoài
Priority 999: DENY  ALL      from 0.0.0.0/0        ← Default deny tất cả
```

**Best practice:** Luôn kết thúc bằng **implicit deny all** — mọi thứ không được phép rõ ràng = bị chặn.

---

## 3. Load Balancing — Phân phối tải thông minh

### Ví dụ đời thường:

Bạn đến **ngân hàng** lúc đông khách. Thay vì tất cả xếp hàng ở 1 quầy, nhân viên hướng dẫn phân bạn đến các quầy khác nhau:
- Quầy 1 đang trống → khách tiếp theo vào
- Quầy 2 đang phục vụ → bỏ qua
- Quầy 3 vừa xong → khách tiếp theo vào

**Load Balancer = Nhân viên hướng dẫn**, phân phối khách (traffic) đến các quầy (servers).

### Tại sao cần Load Balancing?

1. **High Availability:** Nếu 1 server chết, traffic tự chuyển sang server khác
2. **Scalability:** Thêm server = phục vụ nhiều user hơn
3. **Performance:** Không server nào bị quá tải

### Các thuật toán Load Balancing:

#### Round Robin — "Xoay vòng"

Phân phối lần lượt: Server 1 → Server 2 → Server 3 → Server 1 → ...

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  (quay lại)
Request 5 → Server B
...
```

**Ưu điểm:** Đơn giản, công bằng
**Nhược điểm:** Không xét server nào đang bận — nếu Server A xử lý request nặng, vẫn bị gán request tiếp

#### Weighted Round Robin — "Xoay vòng có trọng số"

Server mạnh hơn nhận nhiều traffic hơn:

```
Server A (weight=5): nhận 5 requests
Server B (weight=3): nhận 3 requests
Server C (weight=2): nhận 2 requests
→ Tổng 10 requests: A nhận 50%, B nhận 30%, C nhận 20%
```

**Use case:** Khi servers có cấu hình khác nhau (server mạnh xử lý nhiều hơn).

#### Least Connections — "Ít kết nối nhất"

Gửi request đến server đang có **ít kết nối active nhất**:

```
Server A: 45 connections ← BỎ QUA
Server B: 12 connections ← CHỌN NÀY
Server C: 38 connections ← BỎ QUA
```

**Ưu điểm:** Thông minh hơn Round Robin — xét tải thực tế
**Use case:** Khi requests có thời gian xử lý khác nhau nhiều

#### Least Response Time — "Nhanh nhất"

Gửi request đến server có **response time thấp nhất** + ít connection nhất:

**Use case:** Khi servers ở các vị trí địa lý khác nhau (latency khác nhau).

#### IP Hash — "Dựa trên IP client"

Hash IP của client để quyết định server → cùng client luôn đến cùng server:

```
hash(192.168.1.5) % 3 = 1 → Server B
hash(192.168.1.6) % 3 = 0 → Server A
hash(192.168.1.5) % 3 = 1 → Server B (lần sau cũng vậy!)
```

**Use case:** Khi cần **session persistence** (sticky sessions) — user luôn quay lại cùng server.

#### Consistent Hashing

Phiên bản nâng cao của IP Hash — khi thêm/xóa server, chỉ ảnh hưởng ít traffic (không phải redistribute toàn bộ).

### Layer 4 vs Layer 7 Load Balancing:

| Đặc điểm | Layer 4 (Transport) | Layer 7 (Application) |
|-----------|---------------------|----------------------|
| Nhìn thấy gì | IP + Port | HTTP headers, URL, cookies, body |
| Tốc độ | Nhanh hơn (ít processing) | Chậm hơn (parse HTTP) |
| Tính năng | Đơn giản, forward TCP/UDP | Content-based routing, SSL termination |
| AWS | **NLB** | **ALB** |
| Use case | Gaming, TCP services | Web apps, microservices, API |

---

## 4. AWS Mapping — NAT Gateway, Security Group vs NACL, ALB/NLB/GWLB

### NAT Gateway:

**Vấn đề:** EC2 instances trong **Private Subnet** không có IP public → không ra Internet được. Nhưng chúng cần update OS, download packages...

**Giải pháp:** **NAT Gateway** — cho phép private instances ra Internet (SNAT) nhưng KHÔNG cho Internet vào (one-way).

```
┌─────────────────── VPC ──────────────────────┐
│                                               │
│  ┌─── Public Subnet ────┐                    │
│  │                       │                    │
│  │  ┌──────────────┐    │                    │
│  │  │ NAT Gateway  │    │  ← Có Elastic IP   │
│  │  └──────┬───────┘    │                    │
│  │         │             │                    │
│  └─────────┼─────────────┘                    │
│            │                                  │
│  ┌─── Private Subnet ───┐                    │
│  │         │             │                    │
│  │  ┌──────┴───────┐    │                    │
│  │  │   EC2 App    │    │  ← Không có public IP│
│  │  │ 10.0.2.50    │    │                    │
│  │  └──────────────┘    │                    │
│  └───────────────────────┘                    │
│                                               │
└────────────────────┬──────────────────────────┘
                     │
              Internet Gateway
                     │
                  Internet
```

**Flow:** EC2 (10.0.2.50) → Route Table: "0.0.0.0/0 → NAT GW" → NAT GW thay source IP thành Elastic IP → Internet Gateway → Internet

**NAT Gateway vs NAT Instance:**

| Đặc điểm | NAT Gateway | NAT Instance (EC2) |
|-----------|-------------|---------------------|
| Managed | AWS quản lý (HA built-in) | Tự quản lý |
| Bandwidth | Lên đến 100 Gbps | Phụ thuộc instance type |
| Availability | Multi-AZ redundancy | Single point of failure |
| Cost | Trả theo GB + giờ | Trả theo instance type |
| Maintenance | Không cần | Tự patch, monitor |

### Security Group vs NACL — "Hai lớp bảo vệ":

AWS cung cấp **2 lớp firewall** — giống khu chung cư có cả **bảo vệ cổng chung** (NACL) và **khóa từng căn hộ** (Security Group):

| Đặc điểm | Security Group (SG) | Network ACL (NACL) |
|-----------|---------------------|---------------------|
| Level | **Instance** (ENI) | **Subnet** |
| State | **Stateful** | **Stateless** |
| Rules | Allow only | Allow AND Deny |
| Evaluation | Tất cả rules đánh giá cùng lúc | Theo thứ tự (rule number) |
| Default | Deny all inbound, allow all outbound | Allow all |
| Apply to | Assign cho instance | Tự động apply cho mọi instance trong subnet |

#### NACL — Network Access Control List:

```
Inbound Rules:
┌──────┬────────┬──────────┬──────────────┬────────┐
│ Rule#│ Type   │ Port     │ Source       │ Action │
├──────┼────────┼──────────┼──────────────┼────────┤
│ 100  │ HTTP   │ 80       │ 0.0.0.0/0   │ ALLOW  │
│ 110  │ HTTPS  │ 443      │ 0.0.0.0/0   │ ALLOW  │
│ 120  │ SSH    │ 22       │ 10.0.0.0/16 │ ALLOW  │
│ 130  │ Custom │ 1024-65535│ 0.0.0.0/0  │ ALLOW  │ ← Ephemeral ports cho response
│ *    │ ALL    │ ALL      │ 0.0.0.0/0   │ DENY   │ ← Default deny
└──────┴────────┴──────────┴──────────────┴────────┘

Outbound Rules:
┌──────┬────────┬──────────┬──────────────┬────────┐
│ Rule#│ Type   │ Port     │ Destination  │ Action │
├──────┼────────┼──────────┼──────────────┼────────┤
│ 100  │ HTTP   │ 80       │ 0.0.0.0/0   │ ALLOW  │
│ 110  │ HTTPS  │ 443      │ 0.0.0.0/0   │ ALLOW  │
│ 120  │ Custom │ 1024-65535│ 0.0.0.0/0  │ ALLOW  │ ← Response traffic
│ *    │ ALL    │ ALL      │ 0.0.0.0/0   │ DENY   │
└──────┴────────┴──────────┴──────────────┴────────┘
```

**Lưu ý quan trọng:** Vì NACL là **stateless**, bạn PHẢI mở **ephemeral ports** (1024-65535) cho outbound rules — đây là port mà client dùng để nhận response.

#### Mô hình bảo mật nhiều lớp:

```
Internet
    │
    ▼
┌─────────────────────────────────────────┐
│  NACL (Subnet level - stateless)        │  ← Lọc thô: chặn IP xấu, allow port chính
│  ┌───────────────────────────────────┐  │
│  │  Security Group (Instance level)  │  │  ← Lọc tinh: chỉ allow traffic cần thiết
│  │  ┌─────────────────────────────┐  │  │
│  │  │         EC2 Instance        │  │  │
│  │  └─────────────────────────────┘  │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### ALB, NLB, GWLB — Chi tiết hơn:

#### Gateway Load Balancer (GWLB):

GWLB là loại mới nhất — dùng để route traffic qua **network appliances** (3rd-party firewalls, IDS/IPS, deep packet inspection):

```
Internet → GWLB Endpoint → GWLB → Firewall Appliances (EC2) → Target
```

**Use case:** Khi bạn muốn mọi traffic đi qua firewall của Palo Alto, Fortinet, hoặc CheckPoint trước khi đến ứng dụng.

### Kiến trúc mạng AWS đầy đủ:

```
┌─────────────────────────── VPC (10.0.0.0/16) ────────────────────────────┐
│                                                                           │
│  ┌──── Public Subnet (10.0.1.0/24) ─── AZ-a ────┐                       │
│  │  [Internet Gateway]                            │                       │
│  │  [NAT Gateway]                                 │                       │
│  │  [ALB] ← NACL: Allow 80,443 in; deny rest    │                       │
│  └────────────────────────────────────────────────┘                       │
│                                                                           │
│  ┌──── Private Subnet (10.0.2.0/24) ─── AZ-a ───┐                       │
│  │  [EC2 App Servers]                             │                       │
│  │  SG: Allow 80 from ALB SG                     │                       │
│  │  Route: 0.0.0.0/0 → NAT GW                    │                       │
│  └────────────────────────────────────────────────┘                       │
│                                                                           │
│  ┌──── Private Subnet (10.0.3.0/24) ─── AZ-a ───┐                       │
│  │  [RDS Database]                                │                       │
│  │  SG: Allow 3306 from App SG only              │                       │
│  │  NO route to Internet                          │                       │
│  └────────────────────────────────────────────────┘                       │
│                                                                           │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Deep Dive: Stateful Inspection và Connection Tracking

### Connection Tracking (conntrack) trên Linux:

Linux kernel có module `nf_conntrack` (Netfilter connection tracking) — đây là nền tảng cho iptables stateful firewall:

```bash
# Xem connection tracking table
cat /proc/net/nf_conntrack

# Hoặc dùng conntrack tool
conntrack -L

# Output mẫu:
tcp  6 431999 ESTABLISHED src=192.168.1.5 dst=142.250.80.46 sport=54321 dport=443
     src=142.250.80.46 dst=192.168.1.5 sport=443 dport=54321 [ASSURED]
```

**Các state trong conntrack:**
- **NEW:** Packet đầu tiên của kết nối (SYN)
- **ESTABLISHED:** Kết nối đã được thiết lập (sau SYN-ACK)
- **RELATED:** Kết nối liên quan (ICMP error, FTP data channel)
- **INVALID:** Packet không thuộc kết nối nào → thường bị DROP

### iptables — Linux Firewall:

```bash
# Cho phép established connections (stateful!)
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Cho phép SSH mới từ mạng nội bộ
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -m conntrack --ctstate NEW -j ACCEPT

# Cho phép HTTP/HTTPS mới từ mọi nơi
iptables -A INPUT -p tcp --dport 80 -m conntrack --ctstate NEW -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -m conntrack --ctstate NEW -j ACCEPT

# Default deny
iptables -A INPUT -j DROP
```

---

## 6. Reverse Proxy vs Load Balancer

### Ví dụ đời thường:

**Reverse Proxy** giống **quản gia** của một gia đình giàu:
- Khách đến nhà → gặp quản gia trước (KHÔNG gặp trực tiếp gia chủ)
- Quản gia quyết định: ai được vào, ai bị từ chối, tin nhắn chuyển cho ai
- Khách không biết cấu trúc bên trong nhà

**Reverse Proxy** có thể làm Load Balancing, nhưng còn làm thêm:
- **SSL Termination:** Giải mã HTTPS ở proxy, backend dùng HTTP (nhẹ hơn)
- **Caching:** Cache static content, giảm tải cho backend
- **Compression:** Nén response trước khi gửi cho client
- **Security:** Ẩn backend servers, filter malicious requests

```
Client → Reverse Proxy (Nginx/ALB) → Backend Server 1
                                    → Backend Server 2
                                    → Backend Server 3
```

---

## 7. Thực hành: Lab tự làm

### Lab 1: Xem NAT hoạt động (Linux)

```bash
# Xem NAT rules trên router Linux
sudo iptables -t nat -L -n -v

# Thêm SNAT rule (masquerade)
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Xem NAT translation đang active
conntrack -L -n -p tcp
```

### Lab 2: Cấu hình iptables cơ bản

```bash
# Flush rules cũ
sudo iptables -F

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow SSH (port 22)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Default deny
sudo iptables -A INPUT -j DROP

# Xem rules
sudo iptables -L -n -v
```

### Lab 3: AWS VPC với NAT Gateway

1. Tạo VPC với CIDR `10.0.0.0/16`
2. Tạo Public Subnet (`10.0.1.0/24`) + Private Subnet (`10.0.2.0/24`)
3. Tạo Internet Gateway, attach vào VPC
4. Tạo NAT Gateway trong Public Subnet (cần Elastic IP)
5. Route Table cho Public Subnet: `0.0.0.0/0 → IGW`
6. Route Table cho Private Subnet: `0.0.0.0/0 → NAT GW`
7. Launch EC2 trong Private Subnet → verify có thể `ping 8.8.8.8` nhưng KHÔNG thể SSH từ Internet

### Lab 4: NACL vs Security Group

1. Tạo Custom NACL cho subnet:
   - Allow inbound TCP 80, 443 from `0.0.0.0/0`
   - Allow inbound TCP 22 from your IP only
   - Allow inbound TCP 1024-65535 (ephemeral) from `0.0.0.0/0`
   - Deny all else
2. Tạo Security Group:
   - Allow inbound TCP 80 from `0.0.0.0/0`
   - (Không cần rule cho ephemeral ports — stateful!)
3. Thử:
   - Xóa NACL ephemeral port rule → web server không trả response được
   - Xóa SG outbound rule → web server VẪN response (vì stateful)

---

## 8. Tổng kết

| Khái niệm | Ví dụ đời thường | AWS Service |
|-----------|-----------------|-------------|
| SNAT | Dùng số tổng đài gọi ra | NAT Gateway (instances → Internet) |
| DNAT | Tổng đài chuyển cuộc gọi vào | ALB/NLB (Internet → instances) |
| PAT | 500 người dùng chung 1 số | NAT Gateway (many:1) |
| Stateless Firewall | Máy quẹt thẻ tự động | NACL |
| Stateful Firewall | Bảo vệ con người | Security Group |
| Load Balancer | Nhân viên hướng dẫn ngân hàng | ALB (L7), NLB (L4), GWLB (L3) |
| Reverse Proxy | Quản gia | ALB + CloudFront |

---

## Tài liệu tham khảo

1. **RFC 3022** — Srisuresh, P., Egevang, K. (2001). "Traditional IP Network Address Translator". [https://www.rfc-editor.org/rfc/rfc3022](https://www.rfc-editor.org/rfc/rfc3022)
2. **RFC 2663** — Srisuresh, P., Holdrege, M. (1999). "IP NAT Terminology and Considerations". [https://www.rfc-editor.org/rfc/rfc2663](https://www.rfc-editor.org/rfc/rfc2663)
3. **Cisco NAT Configuration Guide** — [https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipaddr_nat/)
4. **AWS NAT Gateway** — [https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
5. **AWS Network ACLs** — [https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html)
6. **AWS ELB Documentation** — [https://docs.aws.amazon.com/elasticloadbalancing/](https://docs.aws.amazon.com/elasticloadbalancing/)
7. **Stevens, W.R.** "TCP/IP Illustrated, Volume 1" — Addison-Wesley.
8. **Netfilter/iptables documentation** — [https://www.netfilter.org/documentation/](https://www.netfilter.org/documentation/)

---

**Bài tiếp theo:** [Phần 7: Linux CLI & Filesystem — Dòng lệnh và hệ thống tệp](/2026-06-01-linux-cli-filesystem/)
