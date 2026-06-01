---
layout: post
title: "Routing Fundamentals — Cách packets tìm đường đi qua Internet"
subtitle: "Routing Table, Longest Prefix Match, Administrative Distance và Static vs Dynamic Routing — GPS của thế giới mạng"
tags: [networking, routing, layer3, protocols, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
comments: true
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 1812 | Requirements for IP Version 4 Routers |
| RFC 4632 | Classless Inter-Domain Routing (CIDR) |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 5: The Network Layer — Routing Algorithms |
| Kurose & Ross — Computer Networking, 8th Ed. | Chapter 5: The Network Layer: Control Plane |
| Cisco CCNA Official Cert Guide | Chapters on IP Routing |
| Cisco Documentation | IP Routing: Protocol-Independent Configuration Guide |
| AWS Documentation | VPC Route Tables, Transit Gateway Routing |

---

## 1. Giới thiệu — Tại sao cần biết Routing?

### Ví dụ đời thường: Hệ thống giao thông và biển chỉ đường

Bạn lái xe từ Hà Nội đến Đà Nẵng. Tại MỖI ngã tư, bạn nhìn **biển chỉ đường** để quyết định rẽ trái, rẽ phải, hay đi thẳng. Bạn không cần biết TOÀN BỘ đường đi — chỉ cần biết **bước tiếp theo** (next hop) tại mỗi điểm.

Router hoạt động y hệt:
- Mỗi router = một ngã tư
- Routing table = bảng biển chỉ đường
- Packet = chiếc xe
- Destination IP = điểm đến cuối cùng
- Next hop = "đi hướng nào tại ngã tư này"

**Quan trọng**: Router KHÔNG biết toàn bộ đường đi! Nó chỉ biết: "Muốn đến network X, gửi qua cổng Y cho router Z". Router Z sẽ tiếp tục quyết định... cứ như thế cho đến khi packet đến đích.

### Concrete scenario: Gửi email từ VN đến Mỹ

```
Bạn gửi email đến friend@gmail.com:

Laptop (192.168.1.100)
  → Routing decision: "Destination 142.250.190.46 không phải local network"
  → Gửi đến Default Gateway (192.168.1.1)

Router nhà (192.168.1.1):
  → Routing table: "0.0.0.0/0 → via ISP router 10.0.0.1"
  → Forward đến ISP

Router ISP VN:
  → Routing table: "142.250.0.0/15 → via 203.162.x.x (upstream)"
  → Forward đến backbone

Router Backbone VN:
  → BGP: "142.250.0.0/15 thuộc AS 15169 (Google)"
  → Forward qua international link

... 8-15 hops ...

Google Edge Router:
  → Routing table: "142.250.190.46 → connected on interface eth5"
  → Deliver to Gmail server!

MỖI router chỉ biết "next hop" — KHÔNG cần biết toàn bộ path!
```

### Vấn đề Routing giải quyết

| Vấn đề | Giải pháp Routing |
|---------|-------------------|
| Packet biết đi đâu tại mỗi hop? | Routing Table → lookup Destination IP |
| Nhiều đường đến cùng đích? | Best path selection (metric, AD) |
| Đường đi bị thay đổi/hỏng? | Dynamic routing protocols (auto-update) |
| Network quá lớn để biết hết? | Hierarchical routing (AS, areas) |
| Nhiều paths = load sharing? | Equal-Cost Multi-Path (ECMP) |

---

## 2. Routing là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**Routing** là quá trình **quyết định đường đi** cho packet từ nguồn đến đích qua mạng. Mỗi router trên đường đi đưa ra **một quyết định**: "Gửi packet này qua interface nào, đến router nào tiếp theo?"

### Analogy: GPS Navigation

```
╔══════════════════════════════════════════════════════════════════╗
║                GPS vs ROUTING TABLE                               ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  GPS:                         Routing:                           ║
║  ─────                        ────────                           ║
║  Bản đồ = Routing Table       Routing Table = "bản đồ" mạng    ║
║  Vị trí hiện tại = Source IP   Source IP                         ║
║  Điểm đến = Destination        Destination IP                    ║
║  "Rẽ phải 500m" = Next Hop     Next Hop IP + Interface          ║
║  Đường nhanh nhất = Metric     Metric (cost/distance/bandwidth) ║
║  Tránh đường tắc = Failover    Failover (backup route)          ║
║  Real-time traffic = Dynamic   Dynamic routing protocols         ║
║  Cầu sập → tìm đường mới      Link down → convergence          ║
║                                                                  ║
║  KHÁC BIỆT QUAN TRỌNG:                                          ║
║  GPS biết TOÀN BỘ đường đi (source → destination)              ║
║  Router chỉ biết BÂN TIẾP THEO (next hop)!                     ║
║  → "Hop-by-hop forwarding"                                      ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Routing Table — "Bản đồ" của Router

```bash
# Routing table trên Linux:
$ ip route show
default via 192.168.1.1 dev eth0                    ← Default route
10.0.0.0/8 via 172.16.0.1 dev tun0                  ← Static route
172.16.0.0/24 dev tun0 proto kernel scope link      ← Connected
192.168.1.0/24 dev eth0 proto kernel scope link     ← Connected

Đọc hiểu:
- "default via 192.168.1.1" = "Nếu không match route nào → gửi cho 192.168.1.1"
- "10.0.0.0/8 via 172.16.0.1" = "Mạng 10.x.x.x → đi qua tunnel, next hop 172.16.0.1"
- "192.168.1.0/24 dev eth0" = "Mạng 192.168.1.x → trực tiếp trên interface eth0"
```

### Cisco Routing Table:

```
Router# show ip route
Codes: C - connected, S - static, O - OSPF, B - BGP, D - EIGRP

Gateway of last resort is 203.0.113.1 to network 0.0.0.0

C    10.0.1.0/24 is directly connected, GigabitEthernet0/0
C    10.0.2.0/24 is directly connected, GigabitEthernet0/1
S    10.0.3.0/24 [1/0] via 10.0.1.2
O    10.0.4.0/24 [110/20] via 10.0.2.2, GigabitEthernet0/1
B    142.250.0.0/15 [20/0] via 203.0.113.1
S*   0.0.0.0/0 [1/0] via 203.0.113.1

Giải thích:
C = Connected (trực tiếp kết nối)
S = Static (admin cấu hình tay)
O = OSPF (dynamic routing protocol)
B = BGP (inter-domain routing)
[110/20] = [Administrative Distance / Metric]
```

---

## 3. Routing Decision Process — Router quyết định thế nào?

### Mini example: Bưu tá chia thư

Bưu tá nhận kiện hàng ghi "Số 15, Nguyễn Du, Quận 1, TP.HCM":
1. Xem "TP.HCM" → bỏ vào túi "Miền Nam" (match /8 — general)
2. Xem "Quận 1" → bỏ vào túi "Quận 1" (match /16 — more specific)
3. Xem "Nguyễn Du" → bỏ vào túi "Nguyễn Du" (match /24 — most specific)
4. **Chọn túi cụ thể nhất** = "Nguyễn Du" → giao cho shipper đường Nguyễn Du!

Đây chính là **Longest Prefix Match** — router chọn route CỤ THỂ NHẤT (prefix dài nhất).

### Quy trình 3 bước

```
╔══════════════════════════════════════════════════════════════════╗
║        ROUTING DECISION — 3 BƯỚC (theo thứ tự ưu tiên)        ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Bước 1: LONGEST PREFIX MATCH                                   ║
║    Router lookup Destination IP trong routing table              ║
║    Chọn entry có PREFIX DÀI NHẤT match                          ║
║    /32 > /28 > /24 > /16 > /8 > /0 (default)                  ║
║                                                                  ║
║  Bước 2: ADMINISTRATIVE DISTANCE (nếu cùng prefix length)      ║
║    Nếu có 2+ routes cùng prefix, cùng length                   ║
║    → Chọn route có AD THẤP NHẤT                                ║
║    Connected(0) > Static(1) > OSPF(110) > BGP(20/200)          ║
║                                                                  ║
║  Bước 3: METRIC (nếu cùng AD)                                  ║
║    Nếu cùng protocol (cùng AD), nhiều paths                    ║
║    → Chọn route có METRIC THẤP NHẤT                            ║
║    → Hoặc ECMP nếu metrics bằng nhau                           ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Longest Prefix Match — Ví dụ chi tiết

```
Routing Table:
  Route A: 10.0.0.0/8      → via Router X
  Route B: 10.1.0.0/16     → via Router Y
  Route C: 10.1.1.0/24     → via Router Z
  Route D: 0.0.0.0/0       → via Router W (default)

Packet destination: 10.1.1.50
  Match Route A? 10.0.0.0/8 → YES (10.x.x.x)
  Match Route B? 10.1.0.0/16 → YES (10.1.x.x)
  Match Route C? 10.1.1.0/24 → YES (10.1.1.x)  ← LONGEST MATCH!
  → Forward via Router Z

Packet destination: 10.1.2.50
  Match Route A? YES (10.x.x.x)
  Match Route B? YES (10.1.x.x)   ← LONGEST MATCH!
  Match Route C? NO (10.1.1.x ≠ 10.1.2.x)
  → Forward via Router Y

Packet destination: 8.8.8.8
  Match Route A? NO
  Match Route B? NO
  Match Route C? NO
  Match Route D? YES (0.0.0.0/0 matches everything) ← DEFAULT
  → Forward via Router W
```

### Administrative Distance (AD)

| Route Source | Default AD | Ý nghĩa |
|-------------|-----------|---------|
| Connected | 0 | Trực tiếp kết nối — tin cậy nhất |
| Static | 1 | Admin cấu hình tay |
| eBGP | 20 | External BGP |
| EIGRP (summary) | 5 | — |
| EIGRP (internal) | 90 | Cisco proprietary |
| OSPF | 110 | Open standard |
| IS-IS | 115 | ISO standard |
| RIP | 120 | Oldest, least trusted |
| iBGP | 200 | Internal BGP |
| Unknown | 255 | NEVER used (invalid) |

**Tại sao cần AD?** Vì trên cùng router có thể chạy NHIỀU routing protocols. Nếu cả OSPF và RIP đều biết route đến 10.1.0.0/24, router dùng AD để chọn: OSPF (AD=110) thắng RIP (AD=120).

### Metric — Khoảng cách/Chi phí

| Protocol | Metric based on | Ý nghĩa |
|----------|----------------|---------|
| RIP | Hop count | Số router trên đường đi (max 15) |
| OSPF | Cost = 10^8/bandwidth | Bandwidth-based (fast = low cost) |
| EIGRP | Composite | Bandwidth + Delay (+ Load, Reliability) |
| BGP | AS Path length + nhiều attributes | Phức tạp nhất |

### Trong AWS

```
AWS VPC Route Table:
- Local route (VPC CIDR): LUÔN có, AD=0 (connected)
- Static routes: Admin thêm
- Propagated routes: Từ VPN/Direct Connect/TGW

Priority: Most specific prefix wins (Longest Prefix Match)
          Nếu cùng prefix → Static > Propagated

Ví dụ VPC Route Table:
  10.0.0.0/16 → local           (connected VPC CIDR)
  10.1.0.0/16 → pcx-abc123     (VPC peering)
  172.16.0.0/12 → vgw-xyz789   (VPN propagated)
  0.0.0.0/0 → igw-abc123       (Internet Gateway)
```

---

## 4. Static Routing — Cấu hình tay

### Mini example: Bản đồ giấy vs GPS

Static route = **bản đồ giấy**: Bạn tự vẽ đường đi, không tự update khi đường bị cấm.
Dynamic route = **GPS với traffic real-time**: Tự tìm đường, tự update khi có tắc đường.

### Khi nào dùng Static Routing?

```
✅ Dùng static routing khi:
  - Mạng nhỏ (< 5 routers)
  - Default route (0.0.0.0/0 → ISP)
  - Stub networks (chỉ có 1 đường ra)
  - Backup route (floating static — AD cao hơn dynamic)
  - Mạng KHÔNG BAO GIỜ thay đổi topology

❌ KHÔNG dùng static khi:
  - Mạng lớn (>10 routers) — quá nhiều routes để maintain
  - Topology thay đổi thường xuyên
  - Cần failover tự động (static không detect link failure!)
  - Multiple paths cần load balancing
```

### Cấu hình Static Route

```bash
# Linux:
$ sudo ip route add 10.1.0.0/16 via 192.168.1.2        # Via next-hop
$ sudo ip route add 10.2.0.0/16 dev tun0                # Via interface
$ sudo ip route add default via 192.168.1.1             # Default route

# Persistent (Ubuntu):
$ cat /etc/netplan/01-network.yaml
network:
  ethernets:
    eth0:
      routes:
        - to: 10.1.0.0/16
          via: 192.168.1.2
        - to: default
          via: 192.168.1.1

# Cisco:
Router(config)# ip route 10.1.0.0 255.255.0.0 192.168.1.2
Router(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1    ! default
```

### Floating Static Route (backup route)

```
Scenario: Primary path qua OSPF, backup qua static

Normal:
  OSPF: 10.1.0.0/16 [110/20] via 10.0.1.2  ← Active (AD=110)
  Static: 10.1.0.0/16 [250/0] via 10.0.2.2  ← KHÔNG dùng (AD=250 > 110)

Khi OSPF path down:
  OSPF route biến mất!
  Static: 10.1.0.0/16 [250/0] via 10.0.2.2  ← Bây giờ ACTIVE!
  
Cấu hình:
Router(config)# ip route 10.1.0.0 255.255.0.0 10.0.2.2 250
                                                          ↑ AD = 250
```

### Trong AWS

```bash
# Thêm static route trong VPC Route Table
$ aws ec2 create-route \
  --route-table-id rtb-abc123 \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id pcx-xyz789

# Default route đến Internet Gateway
$ aws ec2 create-route \
  --route-table-id rtb-abc123 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-abc123

# Route đến NAT Gateway (cho private subnet)
$ aws ec2 create-route \
  --route-table-id rtb-private \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-abc123
```

---

## 5. Dynamic Routing — Tự động cập nhật

### Mini example: Waze/Google Maps với thông tin giao thông real-time

Static routing = đường cố định trên bản đồ giấy.
Dynamic routing = GPS app tự cập nhật: "Đường A tắc 30 phút → chuyển sang đường B!"

### Phân loại Dynamic Routing Protocols

```
╔══════════════════════════════════════════════════════════════════╗
║         PHÂN LOẠI DYNAMIC ROUTING PROTOCOLS                      ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Theo phạm vi (scope):                                          ║
║  ├── IGP (Interior Gateway Protocol) — TRONG 1 tổ chức         ║
║  │   ├── Distance Vector: RIP, EIGRP                            ║
║  │   └── Link State: OSPF, IS-IS                               ║
║  └── EGP (Exterior Gateway Protocol) — GIỮA các tổ chức       ║
║      └── Path Vector: BGP                                       ║
║                                                                  ║
║  Theo thuật toán:                                               ║
║  ├── Distance Vector ("hỏi hàng xóm"):                         ║
║  │   - Mỗi router chỉ biết: "Đến X, đi hướng A, xa N hops"   ║
║  │   - Chia sẻ TOÀN BỘ routing table với neighbors             ║
║  │   - Ví dụ: RIP (hop count), EIGRP (composite metric)        ║
║  │   - Bellman-Ford algorithm                                    ║
║  │                                                               ║
║  ├── Link State ("biết toàn bộ map"):                           ║
║  │   - Mỗi router biết topology TOÀN BỘ network               ║
║  │   - Chia sẻ link state (neighbors + cost) với MỌI router    ║
║  │   - Tự tính best path (SPF/Dijkstra algorithm)              ║
║  │   - Ví dụ: OSPF, IS-IS                                      ║
║  │                                                               ║
║  └── Path Vector ("biết đường đi qua ai"):                     ║
║      - Mỗi route kèm theo DANH SÁCH AS đi qua                 ║
║      - Dùng cho inter-domain (Internet backbone)                ║
║      - Ví dụ: BGP                                               ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### So sánh Distance Vector vs Link State

| Đặc điểm | Distance Vector | Link State |
|-----------|----------------|-----------|
| Topology knowledge | Chỉ biết neighbors | Biết TOÀN BỘ topology |
| Algorithm | Bellman-Ford | Dijkstra (SPF) |
| Update gì? | Toàn bộ routing table | Chỉ link state changes |
| Update cho ai? | Chỉ neighbors | Flood toàn area |
| Convergence | Chậm | Nhanh |
| Memory/CPU | Ít | Nhiều |
| Loop prevention | Hold-down, split horizon | SPF tree (no loops) |
| Ví dụ | RIP, EIGRP | OSPF, IS-IS |
| Dùng cho | Small networks | Medium-Large networks |

### Convergence — Khi network thay đổi

```
"Convergence" = thời gian để TẤT CẢ routers đồng ý về routing table mới

Scenario: Link giữa Router A và B bị đứt

Distance Vector (RIP):
  - Router A phát hiện: "Link to B down!" (30s detection)
  - A cập nhật route table
  - A gửi update cho neighbors (30s interval)
  - Neighbors update... forward... 
  - Convergence time: 30s × N hops = 60-180 seconds!
  - Trong lúc đó: packets CÓ THỂ bị drop hoặc loop!

Link State (OSPF):
  - Router A phát hiện: "Link down!" (< 1 second with BFD)
  - A flood LSA (Link State Advertisement) đến MỌI router
  - MỌI router nhận LSA, chạy SPF algorithm
  - Convergence time: < 1-5 seconds!
  - Ít packet loss hơn nhiều
```

### ECMP — Equal-Cost Multi-Path

```
Khi có NHIỀU đường đi CÓ CÙNG metric:

Route 1: 10.1.0.0/16 via 192.168.1.2 [metric=20]
Route 2: 10.1.0.0/16 via 192.168.2.2 [metric=20]  ← Cùng metric!

Router KHÔNG chọn 1 — nó dùng CẢ HAI (load balancing)!

Phương pháp load balancing:
- Per-packet: Round-robin (gây out-of-order — ít dùng)
- Per-flow: Hash (src+dst IP+port) → chọn path
  → Cùng connection luôn đi cùng path (avoid reorder)
  → Nhưng vẫn phân tải giữa nhiều connections

ECMP trong AWS:
- Transit Gateway: hỗ trợ ECMP cho VPN connections
- Có thể có 2+ VPN tunnels → traffic load-balanced
```

---

## 6. Routing trong AWS — VPC Route Tables

### Mini example: Tòa nhà văn phòng với bảng chỉ dẫn mỗi tầng

Mỗi tầng (subnet) có bảng chỉ dẫn (route table) riêng:
- "Cùng tầng → đi thẳng" (local route)
- "Tầng khác → ra thang máy" (VPC peering, TGW)
- "Ra ngoài tòa nhà → ra cổng chính" (Internet Gateway, NAT GW)

### AWS VPC Route Table Components

```
╔══════════════════════════════════════════════════════════════════╗
║            AWS VPC ROUTE TABLE                                    ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Destination    │  Target              │  Status    │ Type       ║
║  ───────────────┼──────────────────────┼────────────┼──────────  ║
║  10.0.0.0/16   │  local                │  active    │ local     ║
║  172.16.0.0/16 │  pcx-abc123           │  active    │ static   ║
║  10.1.0.0/16   │  tgw-xyz789           │  active    │ propagated║
║  0.0.0.0/0     │  igw-abc123           │  active    │ static   ║
║                                                                  ║
║  Rules:                                                          ║
║  1. local route = ALWAYS present, cannot delete                  ║
║  2. More specific prefix WINS (longest prefix match)            ║
║  3. Static routes override propagated (same prefix)             ║
║  4. Cannot have 2 routes same CIDR different targets            ║
║     (unlike traditional routers)                                 ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Route Table Targets trong AWS

| Target | Dùng cho | Ví dụ |
|--------|----------|-------|
| local | Traffic trong VPC | 10.0.0.0/16 → local |
| igw-xxx | Internet (public subnet) | 0.0.0.0/0 → igw |
| nat-xxx | Internet (private subnet, outbound) | 0.0.0.0/0 → nat |
| pcx-xxx | VPC Peering | 172.16.0.0/16 → pcx |
| tgw-xxx | Transit Gateway | 10.0.0.0/8 → tgw |
| vgw-xxx | VPN Gateway | 192.168.0.0/16 → vgw |
| vpce-xxx | VPC Endpoint (Gateway) | S3 prefix list → vpce |
| eni-xxx | Network Interface (NAT instance) | 0.0.0.0/0 → eni |

### Subnet Association

```
Mỗi subnet PHẢI associate với ĐÚNG 1 route table.
Nếu không explicit → dùng MAIN route table (default).

Best practice:
- Tạo route tables riêng cho public/private/data subnets
- KHÔNG modify main route table (để clean)
- Associate explicit:
  Public subnets → RT with igw (Internet access)
  Private subnets → RT with nat-gw (outbound only)
  Data subnets → RT without internet (most restricted)
```

---

## 7. Routing Algorithms — Dijkstra và Bellman-Ford

### Mini example: Tìm đường ngắn nhất trên bản đồ

**Bellman-Ford** (Distance Vector): Bạn hỏi mỗi hàng xóm: "Đến siêu thị mất bao xa?" Hàng xóm trả lời dựa trên KINH NGHIỆM CỦA HỌ. Bạn chọn hàng xóm nào cho đáp án ngắn nhất.

**Dijkstra** (Link State): Bạn có BẢN ĐỒ TOÀN BỘ khu phố. Bạn TỰ TÍNH đường ngắn nhất trên bản đồ.

### Dijkstra's Shortest Path First (SPF) — cho OSPF

```
Input: Graph với nodes (routers) và edges (links) có cost

Thuật toán:
1. Start: Đặt cost đến bản thân = 0, mọi node khác = ∞
2. Visited = {}
3. Lặp:
   a. Chọn node U chưa visited có cost THẤP NHẤT
   b. Thêm U vào Visited
   c. Với mỗi neighbor V của U:
      new_cost = cost(U) + cost(U→V)
      if new_cost < cost(V):
        cost(V) = new_cost
        next_hop(V) = direction_to_U
   d. Lặp lại cho đến khi tất cả nodes visited

Ví dụ:
      2         3
  A ───── B ───── D
  │       │       │
  1│      5│      1│
  │       │       │
  C ───── E ───── F
      4         2

Từ A:
  A→C: cost=1 (direct)
  A→B: cost=2 (direct)
  A→C→E: cost=1+4=5
  A→B→D: cost=2+3=5
  A→B→D→F: cost=2+3+1=6
  A→C→E→F: cost=1+4+2=7
  
Shortest path A→F: A→B→D→F (cost=6)
```

### Bellman-Ford — cho RIP/EIGRP

```
Input: Mỗi router chỉ biết neighbors và cost đến neighbors

Thuật toán:
1. Mỗi router khởi tạo: distance đến bản thân = 0
2. Gửi distance vector (list of [destination, cost]) cho neighbors
3. Khi nhận vector từ neighbor N:
   Với mỗi destination D trong vector:
     new_cost = cost(self→N) + cost_from_N(N→D)
     if new_cost < current_cost(D):
       update: cost(D) = new_cost, next_hop(D) = N
4. Lặp cho đến khi không có thay đổi (convergence)

Vấn đề: Count to infinity
  A ─── B ─── C
  
  Link B─C down:
  B biết: "C unreachable"
  Nhưng A nghĩ: "Tôi đến C qua B, cost=2"
  A nói B: "Tôi biết đến C, cost=2!"
  B nghĩ: "À, đến C qua A, cost=3!" ← SAI! Loop!
  ...costs tăng dần... "count to infinity"
  
  Giải pháp: Split horizon, Route poisoning, Hold-down timer
```

---

## 8. Tình huống thực tế — 3 scenarios chi tiết

### Scenario 1: Tại nhà — "Ping được router nhưng không ra Internet"

```
Vấn đề: Laptop ping 192.168.1.1 (gateway) OK, nhưng ping 8.8.8.8 FAIL.

Phân tích routing:
$ ip route show
192.168.1.0/24 dev wlan0 proto kernel scope link
# → KHÔNG CÓ default route!

Nguyên nhân: DHCP không cung cấp default gateway, hoặc route bị xóa.

Fix:
$ sudo ip route add default via 192.168.1.1 dev wlan0
$ ping 8.8.8.8  # Now works!

Hoặc renew DHCP:
$ sudo dhclient -r wlan0 && sudo dhclient wlan0
```

### Scenario 2: Trong công ty — Asymmetric routing gây packet drop

```
Topology:
  Server ── Switch ── [FW A] ── Router ── Internet
                   └── [FW B] ──┘

Route đi: Server → FW A → Internet
Route về: Internet → FW B → Server

Vấn đề: FW B nhận response packets NHƯNG không thấy request (went through FW A)!
→ Stateful firewall DROP response ("no matching state")!

Giải pháp:
1. Symmetric routing: Force traffic đi/về cùng firewall
   - HSRP/VRRP: Active/Standby (1 FW active at a time)
2. State synchronization: FW A sync conntrack với FW B
3. Policy-Based Routing: Force specific traffic qua specific FW
```

### Scenario 3: AWS — Multi-VPC routing với Transit Gateway

```
Architecture: 4 VPCs cần communicate

Without TGW (full mesh peering):
  VPC-A ↔ VPC-B (peering 1)
  VPC-A ↔ VPC-C (peering 2)
  VPC-A ↔ VPC-D (peering 3)
  VPC-B ↔ VPC-C (peering 4)
  VPC-B ↔ VPC-D (peering 5)
  VPC-C ↔ VPC-D (peering 6)
  = 6 peering connections! = N×(N-1)/2

With TGW (hub-and-spoke):
  VPC-A → TGW ← VPC-B
  VPC-C → TGW ← VPC-D
  = 4 attachments! = N

Route Tables:
  VPC-A RT: 10.0.0.0/8 → tgw-xxx (all other VPCs via TGW)
  VPC-B RT: 10.0.0.0/8 → tgw-xxx
  TGW RT: 
    10.1.0.0/16 → attachment-VPC-A
    10.2.0.0/16 → attachment-VPC-B
    10.3.0.0/16 → attachment-VPC-C
    10.4.0.0/16 → attachment-VPC-D
```

---

## 9. Bài tập thực hành

### Bài tập 1: Đọc và phân tích routing table

```bash
# Xem routing table
$ ip route show

# Câu hỏi:
# 1. Default gateway là gì?
# 2. Có bao nhiêu connected routes?
# 3. Packet đến 10.0.5.100 đi qua đâu?
# 4. Packet đến 8.8.8.8 đi qua đâu?
# 5. Thêm route: traffic đến 172.16.0.0/12 qua 10.0.0.1
```

### Bài tập 2: Static routing lab

```bash
# Setup: 3 Linux VMs connected
# VM1 (10.0.1.1) --- VM2 (10.0.1.2 / 10.0.2.1) --- VM3 (10.0.2.2)

# On VM2 (router): Enable forwarding
$ sudo sysctl -w net.ipv4.ip_forward=1

# On VM1: Add route to VM3's network
$ sudo ip route add 10.0.2.0/24 via 10.0.1.2

# On VM3: Add route to VM1's network  
$ sudo ip route add 10.0.1.0/24 via 10.0.2.1

# Test: VM1 ping VM3
$ ping 10.0.2.2

# Traceroute: Should show 2 hops
$ traceroute 10.0.2.2
```

### Bài tập 3: Longest Prefix Match exercise

```
Routing Table:
  10.0.0.0/8      → Next-hop A
  10.1.0.0/16     → Next-hop B
  10.1.1.0/24     → Next-hop C
  10.1.1.128/25   → Next-hop D
  0.0.0.0/0       → Next-hop E

Xác định next-hop cho:
a) 10.1.1.200  → D (/25: 10.1.1.128-255)
b) 10.1.1.50   → C (/24: 10.1.1.0-255, nhưng không match /25)
c) 10.1.2.100  → B (/16: 10.1.x.x)
d) 10.2.0.1    → A (/8: 10.x.x.x)
e) 8.8.8.8     → E (default route)
```

### Bài tập 4: AWS Route Table design

```bash
# Design route tables cho 3-tier architecture:
# Public subnet: needs Internet in/out
# Private subnet: needs Internet out only
# Data subnet: no Internet at all

# Public Route Table:
$ aws ec2 create-route --route-table-id rtb-public \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-xxx

# Private Route Table:
$ aws ec2 create-route --route-table-id rtb-private \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id nat-xxx

# Data Route Table:
# NO default route! Only local + specific routes
$ aws ec2 create-route --route-table-id rtb-data \
  --destination-cidr-block 10.1.0.0/16 \
  --transit-gateway-id tgw-xxx   # For DB replication

# Verify:
$ aws ec2 describe-route-tables --route-table-ids rtb-public rtb-private rtb-data
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Key Points — Những điểm cần nhớ

```
╔══════════════════════════════════════════════════════════════════╗
║              ROUTING FUNDAMENTALS — TÓM TẮT                     ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. Routing = quyết định next hop cho packet tại mỗi router    ║
║  2. Routing Table = "bản đồ" — destination → next hop          ║
║  3. Longest Prefix Match: /32 > /24 > /16 > /8 > /0           ║
║  4. Administrative Distance: Connected(0)>Static(1)>OSPF(110)  ║
║  5. Metric: cost within same protocol (hop, bandwidth, delay)  ║
║  6. Static: Simple, manual, no auto-failover                   ║
║  7. Dynamic: Auto-update, auto-failover, more complex          ║
║  8. Distance Vector (RIP): Share routes with neighbors         ║
║  9. Link State (OSPF): Know full topology, run SPF             ║
║  10. ECMP: Multiple equal-cost paths → load balancing           ║
║                                                                  ║
║  AWS Context:                                                    ║
║  • VPC Route Table: Longest prefix match, static>propagated    ║
║  • local route always present (VPC CIDR)                        ║
║  • Targets: igw, nat, pcx, tgw, vgw, eni, vpce               ║
║  • Transit Gateway: Hub-and-spoke routing for multi-VPC        ║
║  • ECMP: Supported with multiple VPN connections               ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Tài liệu đọc thêm

| Tài liệu | Link/Reference | Nội dung |
|-----------|---------------|----------|
| RFC 1812 | tools.ietf.org/html/rfc1812 | Router Requirements |
| RFC 4632 | tools.ietf.org/html/rfc4632 | CIDR |
| Cisco AD table | cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/15986-admin-distance.html | AD values |
| Tanenbaum Ch.5 | Computer Networks, 6th Ed. | Routing Algorithms |
| AWS VPC Routing | docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html | Route Tables |

---

*Bài tiếp theo: [RIP — Routing Information Protocol](/2026-06-01-rip-routing-information-protocol) — Protocol routing đầu tiên của Internet*
