---
layout: post
title: "RIP — Routing Information Protocol — Giao thức định tuyến lâu đời nhất"
subtitle: "Từ Distance Vector đến Split Horizon — hiểu RIPv1/v2 qua góc nhìn thực tế và so sánh với OSPF, EIGRP"
tags: [networking, routing, rip, distance-vector, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
comments: true
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 1058 | Routing Information Protocol (RIPv1) — 1988 |
| RFC 2453 | RIP Version 2 — 1998 |
| RFC 2080 | RIPng for IPv6 |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 5: Distance Vector Routing |
| Kurose & Ross — Computer Networking, 8th Ed. | Chapter 5: The Distance-Vector (DV) Algorithm |
| Cisco CCNA Official Cert Guide | RIP Configuration |
| Stevens, W.R. — TCP/IP Illustrated, Vol. 1 | Chapter 10: Dynamic Routing Protocols |

---

## 1. Giới thiệu — Tại sao cần biết RIP?

### Ví dụ đời thường: Hỏi đường qua lời đồn

Bạn mới chuyển đến khu phố mới, muốn tìm **tiệm phở ngon nhất**. Bạn hỏi:
- Hàng xóm A: "Tiệm phở Hà, qua 2 ngõ!" (2 hops)
- Hàng xóm B: "Tiệm phở Hà, qua 3 ngõ!" (3 hops)
- Bạn chọn: Đi theo hàng xóm A (ít ngõ hơn = gần hơn)

Nhưng **có vấn đề**: Hàng xóm A chỉ đường qua ngõ hẹp (bandwidth thấp), trong khi hàng xóm B đi đường lớn (bandwidth cao). RIP không biết — nó CHỈ đếm số ngõ (hop count)!

**RIP = cách routing ĐƠN GIẢN NHẤT**: hỏi hàng xóm, chọn đường ít hops. Đơn giản nhưng có nhiều hạn chế.

### Tại sao học RIP khi nó gần như không còn dùng?

```
1. HIỂU NỀN TẢNG: RIP là cha đẻ của distance vector algorithms
   → Hiểu RIP = hiểu EIGRP, BGP path selection dễ hơn

2. CÂU HỎI THI: CCNA, Network+, AWS SAA đều hỏi về RIP concepts
   → Split horizon, count-to-infinity, hop count limit

3. DEBUGGING: Một số mạng legacy VẪN chạy RIP
   → Biết RIP giúp troubleshoot khi gặp

4. SO SÁNH: Hiểu WHY OSPF/EIGRP tốt hơn = hiểu hạn chế RIP
```

### Vấn đề RIP giải quyết (và tạo ra)

| Giải quyết | Gây ra |
|-----------|--------|
| Tự động learn routes (không cần static) | Convergence chậm (30-180 giây) |
| Đơn giản cấu hình | Max 15 hops (không scale cho mạng lớn) |
| Standard mở (RFC) | Không xét bandwidth (hop ≠ quality) |
| Ít CPU/Memory | Count-to-infinity problem |

---

## 2. RIP là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**RIP** (Routing Information Protocol) là **giao thức định tuyến** giúp routers TỰ ĐỘNG học đường đi đến các mạng khác. Nó hoạt động bằng cách:

1. Mỗi 30 giây, mỗi router gửi **danh sách tất cả mạng nó biết** cho các router hàng xóm
2. Đo khoảng cách bằng **hop count** (số router phải đi qua)
3. Chọn đường có **ít hops nhất** (maximum 15 — quá 15 = unreachable)

### Analogy: "Truyền miệng" trong làng

```
╔══════════════════════════════════════════════════════════════════╗
║        RIP = HỆ THỐNG TRUYỀN MIỆNG TRONG LÀNG                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Mỗi tối (30 giây), dân làng tụ tập:                           ║
║                                                                  ║
║  Bác A: "Tôi biết đến chợ = 1 bước, đến trường = 2 bước"     ║
║  Bác B: "Tôi biết đến chợ = 2 bước, đến bệnh viện = 1 bước"  ║
║  Bác C nghe: "Đến chợ qua A = 2 bước (1+1), qua B = 3 bước"  ║
║            → Chọn qua A!                                        ║
║                                                                  ║
║  Ngày hôm sau, cầu đến chợ sập (link down):                    ║
║  Bác A: "Đến chợ = unreachable!"                               ║
║  Nhưng Bác A chưa nói... Bác C vẫn nghĩ "qua A = 2 bước"    ║
║  Bác C nói Bác B: "Tôi đến chợ được, 2 bước!"               ║
║  Bác B nghĩ: "À, qua C = 3 bước!" ← SAI! (đường qua A→chợ)  ║
║  → Count-to-infinity! Routing loop!                              ║
║                                                                  ║
║  Vấn đề: Tin đồn lan truyền CHẬM khi có thay đổi!             ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### RIP trong TCP/IP Stack

```
RIP sử dụng:
- Transport: UDP port 520 (RIPv1 & v2), UDP port 521 (RIPng/IPv6)
- RIPv1: Broadcast (255.255.255.255) mỗi 30s
- RIPv2: Multicast (224.0.0.9) mỗi 30s
- RIPng: Multicast (ff02::9) mỗi 30s
```

---

## 3. RIP Message Format và Hoạt động

### Mini example: Danh sách chia sẻ hàng ngày

Mỗi 30 giây, router gửi cho neighbors một "danh sách" gồm:
- "Mạng 10.1.0.0, cách tôi 2 hops"
- "Mạng 10.2.0.0, cách tôi 1 hop"
- "Mạng 192.168.1.0, cách tôi 3 hops"

### RIPv2 Message Format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Command (1)  |  Version (2)  |       Must Be Zero (2)        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       Address Family (2)      |        Route Tag (2)          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     IP Address (4)                             |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Subnet Mask (4)                            |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Next Hop (4)                              |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                     Metric (4)                                |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
... (up to 25 route entries per message)

Fields:
- Command: 1=Request, 2=Response
- Version: 1 hoặc 2
- Address Family: 2 (IP) — entry đầu có thể = 0xFFFF (authentication)
- Route Tag: Dùng để mark external routes (redistributed)
- Subnet Mask: CHỈ trong RIPv2 (RIPv1 = classful, không có!)
- Next Hop: 0.0.0.0 = dùng sender's IP, hoặc IP cụ thể
- Metric: 1-15 (reachable), 16 = unreachable (infinity)
```

### RIPv1 vs RIPv2

| Feature | RIPv1 (RFC 1058) | RIPv2 (RFC 2453) |
|---------|---------|---------|
| Classful/Classless | Classful (no subnet mask) | Classless (sends subnet mask) |
| VLSM support | ❌ Không | ✅ Có |
| Update method | Broadcast (255.255.255.255) | Multicast (224.0.0.9) |
| Authentication | ❌ Không | ✅ MD5 or plaintext |
| Route Tag | ❌ Không | ✅ Có (external route marking) |
| Next Hop field | ❌ Không | ✅ Có |
| Auto-summary | Luôn bật (cannot disable) | Default bật, CÓ THỂ tắt |

### RIP Timers

```
╔══════════════════════════════════════════════════════════════════╗
║                    RIP TIMERS                                     ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Update Timer = 30 seconds                                       ║
║    → Gửi toàn bộ routing table cho neighbors mỗi 30s           ║
║    → Jitter: ±5s (25-35s) để tránh synchronization             ║
║                                                                  ║
║  Invalid Timer = 180 seconds (6 × update)                       ║
║    → Nếu không nhận update cho route trong 180s                 ║
║    → Route marked INVALID (possibly down)                       ║
║    → Route vẫn trong table nhưng metric = 16                    ║
║                                                                  ║
║  Hold-down Timer = 180 seconds                                   ║
║    → Sau khi route invalid → hold-down bắt đầu                 ║
║    → Trong thời gian này: KHÔNG chấp nhận route mới            ║
║      với metric CAO HƠN (preventing loops)                      ║
║    → CHỈ chấp nhận route từ CÙNG source với metric TỐT HƠN    ║
║                                                                  ║
║  Flush Timer = 240 seconds (8 × update)                         ║
║    → Nếu route vẫn invalid sau 240s → XÓA khỏi routing table  ║
║                                                                  ║
║  Timeline:                                                       ║
║  ├──────┼──────────────────┼──────────────────────┼────┤        ║
║  0     30s               180s                    240s           ║
║  Update Invalid/Hold-down                       Flush           ║
║                                                                  ║
║  Worst case convergence: 180s (invalid) + 30s (propagate) ≈ 4m ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Trong thực tế

```bash
# Cấu hình RIPv2 trên Cisco router:
Router(config)# router rip
Router(config-router)# version 2
Router(config-router)# no auto-summary           ! Tắt auto-summarization
Router(config-router)# network 10.0.0.0          ! Advertise 10.x.x.x networks
Router(config-router)# network 192.168.1.0       ! Advertise 192.168.1.x
Router(config-router)# passive-interface GigabitEthernet0/2  ! Không gửi RIP updates ra interface này

# Xem RIP routes:
Router# show ip route rip
R    10.2.0.0/24 [120/2] via 10.1.0.2, 00:00:15, GigabitEthernet0/0
R    10.3.0.0/24 [120/3] via 10.1.0.2, 00:00:15, GigabitEthernet0/0
```

---

## 4. Loop Prevention — Giải quyết Count-to-Infinity

### Mini example: Tin đồn và tin giả

Trong làng, nếu cầu sập nhưng tin truyền chậm → người ta vẫn chỉ đường qua cầu (loop). Cần "quy tắc" ngăn tin giả:

### Count-to-Infinity Problem

```
Topology: A ─── B ─── C ─── D (linear)

Normal state:
  A knows: D via B, metric=3
  B knows: D via C, metric=2
  C knows: D via D, metric=1

Link C─D breaks:
  C knows: D = unreachable (metric=16)
  
  NHƯNG trước khi C thông báo...
  B gửi update cho C: "Tôi biết đến D, metric=2!"
  (B chưa biết link down — vẫn dùng route cũ qua C)
  
  C nhận: "À, B biết đến D, metric=2... qua B = 3!"
  C update: D via B, metric=3
  
  C gửi update cho B: "Tôi biết D, metric=3!"
  B nhận: "Hmm, metric tăng 2→3, qua C = 4"
  
  B gửi cho A: "D metric=4"... A gửi lại cho B: "D metric=5"...
  
  → Metrics tăng dần: 3, 4, 5, 6, ... 15, 16 (infinity)
  → Mất 16 × 30s ≈ 8 PHÚT để converge!
  → Trong 8 phút: routing LOOP (B→C→B→C...)
```

### Giải pháp Loop Prevention

**1. Maximum Hop Count = 15:**
```
Metric > 15 = unreachable (infinity = 16)
→ Count-to-infinity dừng lại ở 16
→ Nhưng CŨNG GIỚI HẠN network diameter (max 15 hops!)
```

**2. Split Horizon:**
```
Quy tắc: KHÔNG advertise route NGƯỢC VỀ interface mà bạn NHẬN route đó

Ví dụ:
  B learn route "D via C" on interface eth1
  → B KHÔNG gửi route "D" ra interface eth1 cho C
  → C không thể "learn back" route sai từ B!
  
Simple nhưng hiệu quả cho simple topologies.
Không giải quyết được loops qua 3+ routers.
```

**3. Split Horizon with Poisoned Reverse:**
```
Mạnh hơn split horizon:
  Thay vì KHÔNG gửi → GỬI với metric = 16 (poisoned)

  B learn route "D via C" on eth1
  → B gửi cho C: "D, metric=16!" (poison = unreachable)
  → C biết chắc: "Route D qua B = KHÔNG dùng được"
```

**4. Hold-down Timer:**
```
Khi route bị invalid → bật hold-down 180s
Trong 180s:
  - KHÔNG chấp nhận route MỚI với metric XẤU hơn
  - CHỈ chấp nhận nếu: cùng source + metric TỐT hơn
  → Ngăn "tin đồn cũ" gây loop
```

**5. Triggered Updates:**
```
Thay vì đợi 30s timer → GỬI NGAY khi có thay đổi!
  - Link down → immediately send update
  - Giảm convergence time đáng kể
  - Nhưng vẫn chậm hơn link-state protocols
```

---

## 5. RIP Hoạt động chi tiết — Ví dụ step-by-step

### Mini example: 4 routers, xây dựng routing table

```
Topology:
    10.1.0.0/24      10.2.0.0/24      10.3.0.0/24
R1 ─────────── R2 ─────────── R3 ─────────── R4
│               │               │               │
10.10.0.0/24   10.20.0.0/24   10.30.0.0/24   10.40.0.0/24
(LAN)          (LAN)          (LAN)          (LAN)

Initial state (mỗi router chỉ biết connected networks):
R1: {10.10.0.0/24=0, 10.1.0.0/24=0}
R2: {10.20.0.0/24=0, 10.1.0.0/24=0, 10.2.0.0/24=0}
R3: {10.30.0.0/24=0, 10.2.0.0/24=0, 10.3.0.0/24=0}
R4: {10.40.0.0/24=0, 10.3.0.0/24=0}
```

### Iteration 1 (t=30s): Mỗi router gửi table cho neighbors

```
R1 gửi cho R2: {10.10.0.0=1, 10.1.0.0=1}
R2 gửi cho R1: {10.20.0.0=1, 10.1.0.0=1, 10.2.0.0=1}
R2 gửi cho R3: {10.20.0.0=1, 10.1.0.0=1, 10.2.0.0=1}
R3 gửi cho R2: {10.30.0.0=1, 10.2.0.0=1, 10.3.0.0=1}
R3 gửi cho R4: {10.30.0.0=1, 10.2.0.0=1, 10.3.0.0=1}
R4 gửi cho R3: {10.40.0.0=1, 10.3.0.0=1}

After processing:
R1: {10.10.0.0=0, 10.1.0.0=0, 10.20.0.0=1, 10.2.0.0=1}
R2: {10.20.0.0=0, 10.1.0.0=0, 10.2.0.0=0, 10.10.0.0=1, 10.30.0.0=1, 10.3.0.0=1}
R3: {10.30.0.0=0, 10.2.0.0=0, 10.3.0.0=0, 10.20.0.0=1, 10.1.0.0=1, 10.40.0.0=1}
R4: {10.40.0.0=0, 10.3.0.0=0, 10.30.0.0=1, 10.2.0.0=1}
```

### Iteration 2 (t=60s): Routes propagate further

```
After iteration 2:
R1 biết thêm: 10.30.0.0=2, 10.3.0.0=2 (qua R2 → R3)
R4 biết thêm: 10.20.0.0=2, 10.1.0.0=2 (qua R3 → R2)

After iteration 3 (t=90s):
R1 biết: 10.40.0.0=3 (qua R2 → R3 → R4)  ← Đến hop xa nhất!
R4 biết: 10.10.0.0=3 (qua R3 → R2 → R1)

CONVERGENCE: ~90 seconds (3 iterations × 30s)
```

### Trong thực tế — Debug RIP

```bash
# Linux (quagga/FRR):
$ vtysh
router# show ip rip
     Network      Next Hop     Metric From     Tag Time
R(n) 10.20.0.0/24 10.1.0.2    2      10.1.0.2 0  00:25
R(n) 10.30.0.0/24 10.1.0.2    3      10.1.0.2 0  00:25

# Debug RIP messages:
router# debug rip events
router# debug rip packet

# Cisco:
Router# show ip protocols
Routing Protocol is "rip"
  Sending updates every 30 seconds, next due in 12 seconds
  Invalid after 180 seconds, hold down 180, flushed after 240
```

---

## 6. RIP vs OSPF vs EIGRP — So sánh trực quan

### Mini example: Ba cách tìm đường

| | RIP | OSPF | EIGRP |
|--|-----|------|-------|
| Analogy | Hỏi hàng xóm (truyền miệng) | Có bản đồ toàn khu (GPS offline) | Hỏi hàng xóm thông minh (GPS có traffic) |
| Thuật toán | Bellman-Ford | Dijkstra (SPF) | DUAL (Diffusing Update Algorithm) |
| Metric | Hop count (1-15) | Cost (bandwidth-based) | Composite (BW + Delay + ...) |
| Convergence | 60-180 seconds | 1-10 seconds | 1-5 seconds |
| Scalability | < 15 hops | Thousands of routers | Hundreds per AS |
| Update method | Periodic full table (30s) | Event-triggered (only changes) | Event-triggered |
| Memory/CPU | Rất ít | Nhiều (full topology DB) | Trung bình |
| Standard | RFC 2453 (open) | RFC 2328 (open) | Cisco (was proprietary) |
| Khi nào dùng? | Lab, tiny network, legacy | Enterprise, ISP | Cisco-only networks |

### Tại sao RIP vẫn tồn tại?

```
1. Ultra-simple devices: Một số embedded routers chỉ hỗ trợ RIP
2. Legacy networks: Chuyển đổi từ RIP → OSPF cần downtime
3. Small/Flat networks: < 5 routers, không thay đổi = RIP đủ
4. Redistribution: Dùng RIP ở stub networks, redistribute vào OSPF
5. Education: Học RIP trước → hiểu nền tảng → học OSPF/BGP dễ hơn
```

---

## 7. RIPng — RIP cho IPv6

### So sánh RIPv2 vs RIPng

| Feature | RIPv2 | RIPng (RFC 2080) |
|---------|-------|---------|
| Address family | IPv4 | IPv6 |
| UDP Port | 520 | 521 |
| Multicast | 224.0.0.9 | ff02::9 |
| Max metric | 15 | 15 |
| Authentication | MD5 in packet | IPsec (separate) |
| Next hop | In route entry | Separate entry (type 0xFF) |

```bash
# Cisco RIPng configuration:
Router(config)# ipv6 router rip PROCESS-NAME
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ipv6 rip PROCESS-NAME enable
```

---

## 8. Tình huống thực tế — 3 scenarios

### Scenario 1: Lab/Home — Cấu hình RIP trên FRRouting

```bash
# Install FRRouting (FRR) trên Ubuntu
$ sudo apt install frr

# Enable RIP daemon
$ sudo sed -i 's/ripd=no/ripd=yes/' /etc/frr/daemons
$ sudo systemctl restart frr

# Configure via vtysh
$ sudo vtysh
router# configure terminal
router(config)# router rip
router(config-router)# version 2
router(config-router)# no auto-summary
router(config-router)# network 192.168.1.0/24
router(config-router)# network 10.0.0.0/8
router(config-router)# redistribute connected
router(config-router)# exit
router(config)# exit
router# write memory

# Verify
router# show ip rip
router# show ip rip status
```

### Scenario 2: Enterprise — Migration từ RIP sang OSPF

```
Vấn đề: Mạng 20 routers chạy RIP, convergence chậm 3-5 phút.

Kế hoạch migration (zero-downtime):
Phase 1: Enable OSPF trên tất cả routers (alongside RIP)
  - OSPF AD=110 < RIP AD=120 → OSPF routes PREFERRED tự động!
  - RIP vẫn chạy như backup

Phase 2: Verify OSPF routing stable (1-2 tuần monitor)
  - So sánh routing tables
  - Check convergence time
  - Verify no routing loops

Phase 3: Disable RIP
  - Remove "router rip" từ tất cả routers
  - Clean up configurations
  
Tại sao hoạt động?
  AD: OSPF(110) < RIP(120)
  → Khi cả 2 protocol biết cùng route, router chọn OSPF
  → Nếu OSPF fail → RIP vẫn là backup (higher AD)
```

### Scenario 3: Troubleshooting — RIP route flapping

```
Triệu chứng: Route 10.5.0.0/24 liên tục xuất hiện/biến mất.

Debug:
Router# debug rip events
RIP: received update from 10.1.0.2 on Gi0/0
      10.5.0.0/24 via 0.0.0.0 in 3 hops
(30 seconds later)
RIP: 10.5.0.0/24 invalidated, metric 16

Phân tích:
- Route xuất hiện (metric 3) rồi biến mất (metric 16)
- Chu kỳ 30-180 giây
- Có thể do:
  1. Link unstable (flapping) ở router upstream
  2. Interface up/down repeatedly
  3. DHCP lease expiry gây interface restart

Fix:
1. Kiểm tra link stability (show interface | errors)
2. Tăng hold-down timer (giữ route stable hơn)
3. Fix physical layer issue (cáp/module/power)
```

---

## 9. Bài tập thực hành

### Bài tập 1: Tính Convergence

```
Topology: R1 ─ R2 ─ R3 ─ R4 ─ R5 (linear, 5 routers)

Câu hỏi:
1. Bao nhiêu iterations để R1 learn route đến R5?
   → 4 iterations (hop count from R1 to R5 = 4)
   → Time: 4 × 30s = 120 seconds minimum

2. Nếu link R4-R5 down, mất bao lâu R1 biết?
   → R4 detect (Invalid timer): 180s
   → Propagate qua R3, R2: 2 × 30s = 60s
   → Total: ~240 seconds worst case!

3. Maximum hops allowed nếu thêm routers?
   → 15 hops max → maximum 16 routers in linear topology
```

### Bài tập 2: Split Horizon exercise

```
Topology:
    R1 ─── R2 ─── R3
    │               │
    └───── R4 ─────┘

R2 learns "10.5.0.0 via R3, metric=1" on interface eth1

With Split Horizon:
- R2 sends to R1 (eth0): "10.5.0.0, metric=2" ✓
- R2 sends to R3 (eth1): KHÔNG gửi route 10.5.0.0 ✓
  (vì learn từ eth1 → không advertise ngược eth1)

With Poisoned Reverse:
- R2 sends to R1 (eth0): "10.5.0.0, metric=2" ✓
- R2 sends to R3 (eth1): "10.5.0.0, metric=16!" ✓
  (poison = tell R3 "don't use me for this route")
```

### Bài tập 3: RIP Lab với Docker

```bash
# Tạo lab RIP với 3 containers
$ cat docker-compose.yml
version: '3'
services:
  r1:
    image: frrouting/frr:latest
    cap_add: [NET_ADMIN, SYS_ADMIN]
    networks:
      net12:
        ipv4_address: 10.1.0.1
      lan1:
        ipv4_address: 192.168.1.1
  r2:
    image: frrouting/frr:latest
    cap_add: [NET_ADMIN, SYS_ADMIN]
    networks:
      net12:
        ipv4_address: 10.1.0.2
      net23:
        ipv4_address: 10.2.0.1
  r3:
    image: frrouting/frr:latest
    cap_add: [NET_ADMIN, SYS_ADMIN]
    networks:
      net23:
        ipv4_address: 10.2.0.2
      lan3:
        ipv4_address: 192.168.3.1

networks:
  net12: {ipam: {config: [{subnet: 10.1.0.0/24}]}}
  net23: {ipam: {config: [{subnet: 10.2.0.0/24}]}}
  lan1: {ipam: {config: [{subnet: 192.168.1.0/24}]}}
  lan3: {ipam: {config: [{subnet: 192.168.3.0/24}]}}

# Configure RIP on each router and verify convergence
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Key Points — Những điểm cần nhớ

```
╔══════════════════════════════════════════════════════════════════╗
║                      RIP — TÓM TẮT                              ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. RIP = Distance Vector protocol (hỏi hàng xóm)              ║
║  2. Metric = Hop Count (1-15, 16 = infinity/unreachable)       ║
║  3. Update: Mỗi 30s gửi TOÀN BỘ routing table                 ║
║  4. Timers: Update(30) Invalid(180) Hold-down(180) Flush(240) ║
║  5. Max 15 hops → KHÔNG scale cho mạng lớn!                    ║
║  6. RIPv1: classful, broadcast, no auth                        ║
║  7. RIPv2: classless, multicast 224.0.0.9, MD5 auth           ║
║  8. Loop prevention: Split Horizon, Poison Reverse, Hold-down  ║
║  9. Convergence: CHẬM (30-180+ seconds)                        ║
║  10. AD = 120 (thấp priority so với OSPF=110, EIGRP=90)       ║
║                                                                  ║
║  Khi nào dùng:                                                   ║
║  ✅ Network < 15 hops, topology ít thay đổi                    ║
║  ✅ Legacy equipment chỉ support RIP                            ║
║  ✅ Lab/learning environment                                     ║
║  ❌ KHÔNG dùng cho production enterprise/ISP                    ║
║                                                                  ║
║  AWS Context:                                                    ║
║  • AWS KHÔNG dùng RIP — VPC routing = static + BGP             ║
║  • BGP dùng cho VPN/Direct Connect (AD=20 external)            ║
║  • Transit Gateway supports BGP (not RIP/OSPF)                 ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Tài liệu đọc thêm

| Tài liệu | Link/Reference | Nội dung |
|-----------|---------------|----------|
| RFC 2453 | tools.ietf.org/html/rfc2453 | RIP Version 2 |
| RFC 1058 | tools.ietf.org/html/rfc1058 | RIP Version 1 |
| RFC 2080 | tools.ietf.org/html/rfc2080 | RIPng for IPv6 |
| Cisco RIP Config | cisco.com | RIP Configuration Guide |
| FRRouting Docs | docs.frrouting.org | Open source routing suite |
| GNS3 Labs | gns3.com | Network simulation for practice |

---

*Bài tiếp theo: [OSPF Deep Dive](/2026-06-01-ospf-deep-dive) — Link-State Protocol cho mạng enterprise*
