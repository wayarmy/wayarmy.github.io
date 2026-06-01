---
layout: post
title: "OSPF Deep Dive — Link-State Protocol cho mạng Enterprise hiện đại"
subtitle: "Từ Dijkstra SPF đến Areas, LSA Types, DR/BDR election — giao thức routing backbone của Internet"
tags: [networking, ospf, routing, link-state, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
comments: true
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 2328 | OSPF Version 2 (current standard) — 1998 |
| RFC 5340 | OSPF for IPv6 (OSPFv3) |
| RFC 3101 | OSPF NSSA (Not-So-Stubby Area) |
| RFC 5243 | OSPF Database Exchange Summary List Optimization |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 5: Link State Routing |
| Jeff Doyle — Routing TCP/IP, Volume I | Chapters 6-9: OSPF |
| Cisco CCNP ENCOR Official Cert Guide | OSPF Chapters |
| Cisco Documentation | OSPF Configuration Guide |

---

## 1. Giới thiệu — Tại sao cần biết OSPF?

### Ví dụ đời thường: Từ "hỏi đường" đến "có bản đồ"

Nhớ RIP hoạt động bằng cách "hỏi hàng xóm" — bạn không biết toàn bộ đường đi, chỉ tin lời người kế bên. Vấn đề: nếu hàng xóm nói sai → bạn đi lạc (routing loop).

**OSPF** thay đổi hoàn toàn cách tiếp cận: Thay vì hỏi hàng xóm, **MỖI router có BẢN ĐỒ TOÀN BỘ** khu vực mạng. Giống như Google Maps — bạn TỰ TÍNH đường đi tối ưu dựa trên bản đồ!

```
RIP = Hỏi đường: "Bác ơi, đến chợ đi đâu?"
                  "Đi thẳng 3 ngõ!" (tin tưởng mù quáng)

OSPF = Google Maps: Tải bản đồ TOÀN BỘ khu vực
                    Tự tính shortest path (Dijkstra)
                    Biết CHÍNH XÁC mỗi con đường rộng bao nhiêu
```

### Concrete scenario: Tại sao enterprise network cần OSPF

```
Công ty lớn:
- HQ: 50 routers, 200 subnets
- 5 branches: mỗi branch 10 routers
- Data center: 20 routers
- Total: 100+ routers, 500+ subnets

Với RIP:
- Max 15 hops → HQ to remote branch = VƯỢT GIỚI HẠN!
- Convergence: 3-5 phút → downtime khi link fail
- Bandwidth: Gửi full table (500 routes) mỗi 30s cho MỌI neighbor → lãng phí

Với OSPF:
- Không giới hạn hops
- Convergence: 1-5 giây (với BFD: sub-second!)
- Bandwidth: Chỉ gửi THAY ĐỔI → tiết kiệm 99% bandwidth
- Areas: Chia network thành vùng → SPF calculation nhanh hơn
```

### Vấn đề OSPF giải quyết

| Vấn đề RIP | Giải pháp OSPF |
|-----------|----------------|
| Max 15 hops | Không giới hạn (metric = cost based on bandwidth) |
| Convergence chậm (phút) | Convergence nhanh (giây) |
| Gửi full table mỗi 30s | Gửi incremental updates (chỉ thay đổi) |
| Count-to-infinity/loop | SPF tree → NO LOOPS (by design) |
| Không xét bandwidth | Cost = 10^8/bandwidth (fast link = low cost) |
| Flat topology | Hierarchical areas (scale đến 1000+ routers) |
| No authentication | MD5/SHA authentication |
| Classful (RIPv1) | Classless (VLSM support) |

---

## 2. OSPF là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**OSPF** (Open Shortest Path First) là **giao thức định tuyến link-state** dùng trong mạng nội bộ (IGP). Nó hoạt động bằng cách:

1. Mỗi router **khám phá neighbors** (Hello protocol)
2. Mỗi router **mô tả các links** của mình (LSA — Link State Advertisement)
3. Mỗi router **chia sẻ mô tả** với MỌI router khác (flooding)
4. Mỗi router **xây bản đồ** toàn bộ topology (LSDB — Link State Database)
5. Mỗi router **tự tính shortest path** (SPF/Dijkstra algorithm)

### Analogy: Xây bản đồ từ các mảnh ghép

```
╔══════════════════════════════════════════════════════════════════╗
║              OSPF = GHÉP BẢN ĐỒ TỪ CÁC MẢNH                   ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Tưởng tượng 5 trinh sát mỗi người vẽ bản đồ KHU VỰC MÌNH:   ║
║                                                                  ║
║  Trinh sát A (Router A): "Tôi kết nối với B (đường 10m rộng)   ║
║                           và C (đường 5m rộng)"                 ║
║  → Đây là LSA (Link State Advertisement) của A                  ║
║                                                                  ║
║  Mỗi trinh sát GỬI mảnh bản đồ cho TẤT CẢ người khác         ║
║  → Flooding LSAs                                                 ║
║                                                                  ║
║  Khi MỌI NGƯỜI có đủ tất cả mảnh:                              ║
║  → Ghép lại = BẢN ĐỒ TOÀN BỘ (LSDB)                          ║
║  → Mỗi người TỰ tính đường ngắn nhất (Dijkstra)               ║
║                                                                  ║
║  Kết quả: TẤT CẢ routers có CÙNG bản đồ!                      ║
║  → Tính toán CÙNG kết quả → KHÔNG có loop!                     ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### OSPF Key Concepts Overview

```
1. HELLO PROTOCOL → Discover & maintain neighbors
2. LSA (Link State Advertisement) → Mô tả topology
3. LSDB (Link State Database) → "Bản đồ" complete
4. SPF Algorithm (Dijkstra) → Tính shortest path tree
5. AREAS → Chia để trị (hierarchical design)
6. DR/BDR → Giảm overhead trên broadcast networks
```

---

## 3. Hello Protocol — Tìm và duy trì neighbors

### Mini example: Chào hỏi hàng ngày

Mỗi sáng, bạn ra ngoài chào: "Xin chào! Tôi là Hùng, nhà số 5!" Nếu hàng xóm chào lại → bạn biết họ còn sống. Nếu 3 ngày không thấy → "Hình như bác ấy đi vắng rồi" (neighbor dead).

### OSPF Hello Packet

```
Hello packets gửi periodic để:
1. Discover neighbors (ai ở bên cạnh?)
2. Negotiate parameters (cùng area? cùng timers?)
3. Maintain adjacency (còn sống không?)

Hello interval: 10 seconds (broadcast/P2P), 30 seconds (NBMA)
Dead interval: 4 × Hello = 40 seconds (default)
              → Nếu không nhận Hello trong 40s → neighbor DEAD!

Multicast address: 224.0.0.5 (AllSPFRouters)
IP Protocol: 89 (OSPF has own protocol number — NOT TCP/UDP!)
```

### Điều kiện để form Adjacency

```
2 routers PHẢI match các điều kiện sau mới thành neighbors:

╔═══════════════════════════════════════════════════════════╗
║  OSPF NEIGHBOR REQUIREMENTS                              ║
╠═══════════════════════════════════════════════════════════╣
║  ✅ Cùng Area ID                                         ║
║  ✅ Cùng subnet (interface IP cùng subnet)              ║
║  ✅ Cùng Hello/Dead timer values                        ║
║  ✅ Cùng Authentication (password/MD5)                   ║
║  ✅ Cùng Stub Area Flag                                  ║
║  ✅ Cùng MTU (hoặc mtu-ignore)                          ║
║                                                           ║
║  Nếu BẤT KỲ điều kiện nào KHÔNG match → KHÔNG thành     ║
║  neighbor! (Stuck in INIT/2-WAY state)                    ║
╚═══════════════════════════════════════════════════════════╝
```

### OSPF Neighbor States

```
DOWN → INIT → 2-WAY → EXSTART → EXCHANGE → LOADING → FULL

DOWN: Chưa nhận Hello nào
INIT: Nhận Hello nhưng không thấy Router ID mình trong đó
2-WAY: Thấy Router ID mình trong Hello → mutual recognition!
  → Trên broadcast network: DR/BDR election xảy ra ở đây
  → Nếu KHÔNG cần adjacency → dừng ở 2-WAY (DROther↔DROther)
EXSTART: Negotiate Master/Slave cho DB exchange
EXCHANGE: Trao đổi Database Description (DBD) packets
LOADING: Gửi LSR (Link State Request) cho LSAs còn thiếu
FULL: LSDB đồng bộ hoàn toàn! → ADJACENCY formed!
```

### Trong thực tế

```bash
# Xem OSPF neighbors
Router# show ip ospf neighbor
Neighbor ID  Pri  State     Dead Time  Address       Interface
10.0.0.2     1    FULL/DR   00:00:35   10.1.0.2     Gi0/0
10.0.0.3     1    FULL/BDR  00:00:38   10.1.0.3     Gi0/0
10.0.0.4     1    2WAY/-    00:00:32   10.1.0.4     Gi0/0

# Troubleshoot: Stuck in INIT?
# → Kiểm tra: 1-way Hello (không thấy mình trong neighbor list)
# → Firewall chặn OSPF? (protocol 89, multicast 224.0.0.5/6)
# → Subnet mismatch?
```

---

## 4. LSA Types và LSDB — Bản đồ mạng

### Mini example: Các loại "mảnh bản đồ"

Giống như bản đồ có nhiều layer (đường bộ, sông, ranh giới quận), OSPF có nhiều loại LSA để mô tả các khía cạnh khác nhau của topology.

### LSA Types tổng quan

| Type | Tên | Generated by | Flooded within | Mô tả |
|------|-----|-------------|----------------|--------|
| 1 | Router LSA | Every router | Same area | "Tôi kết nối với ai, cost bao nhiêu" |
| 2 | Network LSA | DR only | Same area | "Trên segment này có những ai" |
| 3 | Summary LSA | ABR | Other areas | "Area X có network Y, cost Z" |
| 4 | ASBR Summary | ABR | Other areas | "ASBR ở đâu, qua ai để đến" |
| 5 | External LSA | ASBR | Entire domain | "Mạng ngoài OSPF domain" (e.g., static, BGP) |
| 7 | NSSA External | ASBR in NSSA | NSSA area | External route trong Not-So-Stubby Area |

### LSA Type 1 — Router LSA (quan trọng nhất)

```
Mỗi router tạo MỘT Type 1 LSA mô tả:
- Router ID (thường = highest loopback IP)
- Tất cả interfaces trong area
- Cost (metric) đến mỗi neighbor
- Type of link (point-to-point, transit, stub)

Ví dụ Router R1 (ID: 1.1.1.1) LSA Type 1:
  Link 1: Transit link to 10.1.0.0/24 (DR: 10.1.0.2), Cost=10
  Link 2: Point-to-point to R3 (1.1.1.3), Cost=20
  Link 3: Stub network 192.168.1.0/24, Cost=1
```

### LSA Type 2 — Network LSA

```
Chỉ DR (Designated Router) tạo!
Mô tả ai kết nối trên broadcast segment:

Network LSA for 10.1.0.0/24 (DR = R2):
  Attached routers: R1 (1.1.1.1), R2 (2.2.2.2), R3 (3.3.3.3)
  Network mask: 255.255.255.0

→ Các router khác nhìn LSA này biết: "Trên segment 10.1.0.0/24
   có 3 routers: R1, R2, R3"
```

### OSPF Cost Calculation

```
Cost = Reference Bandwidth / Interface Bandwidth
Default reference = 100 Mbps (10^8)

Interface          | Bandwidth  | Cost
Fast Ethernet      | 100 Mbps   | 100M/100M = 1
Gigabit Ethernet   | 1 Gbps     | 100M/1000M = 0.1 → rounded to 1!
10 Gigabit         | 10 Gbps    | 100M/10000M = 0.01 → 1!

PROBLEM: Với reference 100M, mọi link ≥ 100M đều cost=1!
→ Không phân biệt 1G vs 10G vs 100G!

FIX: Tăng reference bandwidth!
Router(config-router)# auto-cost reference-bandwidth 100000
  (100 Gbps reference)
  → 1G cost=100, 10G cost=10, 100G cost=1 → phân biệt được!
```

### Trong thực tế

```bash
# Xem LSDB
Router# show ip ospf database

            OSPF Router with ID (1.1.1.1) (Process ID 1)
                Router Link States (Area 0)
Link ID     ADV Router  Age  Seq#       Checksum  Link count
1.1.1.1     1.1.1.1     234  0x80000005 0x00AB12  3
2.2.2.2     2.2.2.2     567  0x80000003 0x00CD34  4

                Net Link States (Area 0)
Link ID     ADV Router  Age  Seq#       Checksum
10.1.0.2    2.2.2.2     234  0x80000002 0x00EF56

# Xem chi tiết 1 LSA:
Router# show ip ospf database router 1.1.1.1
```

---

## 5. DR/BDR Election — Giảm overhead trên broadcast network

### Mini example: Lớp trưởng và lớp phó

Trong lớp 40 học sinh, nếu AI CŨNG nói chuyện với nhau → 40×39/2 = 780 cuộc hội thoại! Quá hỗn loạn!

Giải pháp: Bầu **Lớp trưởng (DR)** và **Lớp phó (BDR)**:
- Mọi người chỉ báo cáo cho Lớp trưởng
- Lớp trưởng tổng hợp và thông báo cho tất cả
- Lớp phó backup nếu Lớp trưởng nghỉ

### Tại sao cần DR/BDR?

```
Broadcast segment với 10 routers:
  Không có DR: 10×9/2 = 45 adjacencies! 
  → 45 cặp exchange DBD → OVERWHELMING!

  Có DR/BDR: 
  → Mỗi router chỉ form adjacency với DR + BDR = 2
  → Total: 10 × 2 = 20 adjacencies (less than half!)
  → DR tạo Network LSA (Type 2) thay mặt tất cả

DROther routers: Chỉ đạt 2-WAY state với nhau (không FULL)
```

### DR/BDR Election Process

```
╔═══════════════════════════════════════════════════════════════════╗
║             DR/BDR ELECTION RULES                                 ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                   ║
║  1. Priority cao nhất → DR (default priority = 1)                ║
║  2. Nếu tie → Router ID cao nhất → DR                           ║
║  3. Router thứ 2 → BDR                                          ║
║  4. Priority = 0 → KHÔNG BAO GIỜ là DR/BDR                     ║
║                                                                   ║
║  QUAN TRỌNG:                                                      ║
║  • Election KHÔNG preemptive!                                     ║
║    Nếu DR đã elected → router mới priority cao hơn              ║
║    KHÔNG tự động thành DR! (phải đợi DR fail)                   ║
║  • DR fail → BDR lên DR → elect BDR mới                        ║
║                                                                   ║
║  Multicast addresses:                                             ║
║  • 224.0.0.5 = ALLSPFRouters (DR + DROthers listen)             ║
║  • 224.0.0.6 = ALLDRouters (CHỈ DR + BDR listen)               ║
║                                                                   ║
║  DROther → gửi LSA đến 224.0.0.6 (only DR/BDR hear)            ║
║  DR → flood LSA đến 224.0.0.5 (everyone hears)                  ║
║                                                                   ║
╚═══════════════════════════════════════════════════════════════════╝
```

### Cấu hình priority

```bash
# Cisco: Force router thành DR (set priority cao)
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip ospf priority 255    ! Highest → sẽ thành DR

# Ngăn router thành DR (set priority 0)
Router(config-if)# ip ospf priority 0      ! Never DR/BDR

# Linux (FRR):
router(config)# interface eth0
router(config-if)# ip ospf priority 200
```

---

## 6. OSPF Areas — Thiết kế phân cấp

### Mini example: Quận/Huyện trong thành phố

Thành phố lớn được chia thành Quận:
- **Quận trung tâm (Area 0/Backbone)**: Kết nối TẤT CẢ quận khác
- **Quận ngoại ô (Regular Areas)**: Chỉ biết chi tiết bên trong quận mình
- **Tóm tắt liên quận**: "Quận 7 có 50 đường" → summarized thành "Quận 7 = 1 tuyến"

### Area Design Rules

```
╔══════════════════════════════════════════════════════════════════╗
║                OSPF AREA RULES                                    ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Rule 1: PHẢI có Area 0 (Backbone)                              ║
║  Rule 2: Mọi non-backbone area PHẢI kết nối với Area 0          ║
║  Rule 3: Area 0 PHẢI contiguous (liền mạch)                    ║
║          (Nếu không → dùng Virtual Link)                        ║
║                                                                  ║
║  Topology:                                                       ║
║           Area 1        Area 0 (Backbone)       Area 2          ║
║  ┌──────────────┐  ┌───────────────────┐  ┌──────────────┐    ║
║  │  R1    R2    │──│  R3 (ABR)  R4     │──│  R5 (ABR) R6 │    ║
║  │              │  │           R7       │  │              │    ║
║  │  internal    │  │   backbone        │  │  internal    │    ║
║  └──────────────┘  └───────────────────┘  └──────────────┘    ║
║                                                                  ║
║  ABR (Area Border Router): Kết nối 2+ areas                    ║
║  ASBR (AS Boundary Router): Kết nối OSPF domain với bên ngoài  ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Area Types

| Area Type | LSA Types allowed | Use Case |
|-----------|-------------------|----------|
| Normal (Regular) | 1, 2, 3, 4, 5 | Default — full information |
| Stub | 1, 2, 3 | No external routes — uses default route |
| Totally Stubby | 1, 2, default-3 | Minimal LSDB — chỉ default + internal |
| NSSA | 1, 2, 3, 7 | Stub + cho phép redistribute local externals |
| Totally NSSA | 1, 2, default-3, 7 | Most restricted + local externals |

### Tại sao dùng Areas?

```
Mạng 500 routers, 2000 subnets:

Không có Areas (single area):
  - LSDB: 500 Type-1 LSAs + nhiều Type-2 → rất lớn
  - Mỗi link change → SPF tính lại TOÀN BỘ 500 nodes
  - CPU spike trên mọi router!

Với Areas (5 areas, 100 routers/area):
  - LSDB mỗi area: 100 Type-1 + summary Type-3
  - Link change trong Area 2 → chỉ Area 2 routers run SPF
  - Routers Area 1 chỉ nhận summary (không run full SPF)
  - CPU giảm 80%!
```

---

## 7. SPF Algorithm và Convergence

### Dijkstra trong OSPF

```
OSPF SPF calculation:

Input: LSDB (all Type-1 and Type-2 LSAs in area)
Output: SPF Tree (shortest path to every destination)

1. Router đặt mình là ROOT of tree
2. Examine tất cả router LSAs
3. Build tree: shortest cost from root to every router
4. Result: next-hop cho mỗi destination

Khi link change:
  - Router detect change (interface down / Hello timeout)
  - Generate new LSA (incremented sequence number)
  - Flood LSA to all routers in area
  - Mỗi router nhận LSA → update LSDB → re-run SPF
  - Install new routes

SPF Throttling (prevent CPU overload):
  - spf-delay: Wait 5s after first change before running SPF
  - spf-holdtime: Minimum time between SPF runs (10s)
  - Lý do: Nếu nhiều links down cùng lúc → batch SPF = 1 run
```

### Fast Convergence với BFD

```
Default: Hello 10s, Dead 40s → detect failure: 40 seconds!
Too slow for VoIP/video!

BFD (Bidirectional Forwarding Detection):
  - Sub-second failure detection (50ms typical)
  - Hardware-assisted (line card level)
  - BFD session per neighbor → detect failure → notify OSPF
  - OSPF immediately declares neighbor down → flood LSA → converge

Timeline:
  Without BFD: Link fail → 40s dead timer → SPF → convergence = 42-45s
  With BFD: Link fail → 150ms BFD detect → SPF → convergence = 1-2s!
```

---

## 8. Tình huống thực tế — 3 scenarios

### Scenario 1: Enterprise — OSPF multi-area design

```
Company: 3 buildings, 200 routers total

Design:
  Area 0 (Backbone): Core routers (10 routers)
    - Connects all buildings
    - High-speed links (10G+)
    
  Area 1 (Building A): 60 routers
    - User access, Wi-Fi controllers
    - Summary: 10.1.0.0/16 advertised to Area 0
    
  Area 2 (Building B): 60 routers  
    - Labs, dev environment
    - Summary: 10.2.0.0/16
    
  Area 3 (Data Center): 50 routers
    - Servers, storage networking
    - Summary: 10.3.0.0/16
    
  Area 51 (DMZ — Stub): 20 routers
    - Internet-facing servers
    - Stub area (no external routes, use default)

ABRs:
  R10 (Area0↔Area1): Summarize Area 1 routes
  R20 (Area0↔Area2): Summarize Area 2 routes
  R30 (Area0↔Area3): Summarize Area 3 routes
  R40 (Area0↔Area51): Default route into stub
```

### Scenario 2: Troubleshooting — Neighbor stuck in EXSTART

```
Symptom: show ip ospf neighbor
  10.0.0.2  1  EXSTART/-  00:00:40  10.1.0.2  Gi0/0

Router stuck in EXSTART for minutes → never reaches FULL

Common causes:
1. MTU mismatch!
   - R1: interface MTU 1500
   - R2: interface MTU 9000 (jumbo frames)
   - DBD packets > 1500 → R1 cannot process!
   
   Fix: Match MTU on both sides
   Hoặc: ip ospf mtu-ignore (workaround, not recommended)

2. Duplicate Router IDs
   - R1 và R2 cùng Router ID = 1.1.1.1
   - Conflict → cannot form adjacency
   
   Fix: Unique router-id trên mỗi router
   router ospf 1
     router-id 1.1.1.1    ! Manually assign unique

3. ACL blocking OSPF
   - Protocol 89 bị firewall chặn
   - Multicast 224.0.0.5/6 bị block
   
   Fix: permit protocol ospf / permit 224.0.0.5-6
```

### Scenario 3: AWS hybrid — OSPF on-premises + BGP to AWS

```
Architecture:
  On-premises (OSPF) ←→ Direct Connect (BGP) ←→ AWS VPC

On-premises router:
  - Runs OSPF internally (area 0)
  - Runs BGP to AWS Direct Connect
  - Redistributes:
    OSPF → BGP (advertise on-prem routes to AWS)
    BGP → OSPF (advertise AWS routes to on-prem)

Configuration (Cisco):
  router ospf 1
    redistribute bgp 65000 subnets metric 100 metric-type 1
    
  router bgp 65000
    neighbor 169.254.x.x remote-as 7224    ! AWS ASN
    address-family ipv4
      redistribute ospf 1
      network 10.0.0.0 mask 255.0.0.0      ! Summarize to AWS

AWS side:
  - VGW (Virtual Private Gateway) runs BGP
  - Learns on-prem routes → propagates to VPC route table
  - Route propagation: automatic (enable trên route table)

Lưu ý:
  - AWS KHÔNG support OSPF qua Direct Connect/VPN
  - PHẢI dùng BGP cho AWS connectivity
  - On-prem OSPF ↔ BGP redistribution là mandatory
```

---

## 9. Bài tập thực hành

### Bài tập 1: OSPF Basic Configuration

```bash
# FRRouting trên Linux:
$ sudo vtysh
router# configure terminal

router(config)# router ospf
router(config-router)# router-id 1.1.1.1
router(config-router)# network 10.0.1.0/24 area 0
router(config-router)# network 192.168.1.0/24 area 1
router(config-router)# passive-interface eth2    ! LAN interface
router(config-router)# exit

# Set interface cost
router(config)# interface eth0
router(config-if)# ip ospf cost 10
router(config-if)# ip ospf hello-interval 10
router(config-if)# ip ospf dead-interval 40

# Verify
router# show ip ospf neighbor
router# show ip ospf database
router# show ip ospf interface
router# show ip route ospf
```

### Bài tập 2: Troubleshoot neighbor relationship

```bash
# Checklist when neighbors don't form:
1. show ip ospf interface (verify area, timers, network type)
2. show ip ospf neighbor (check state)
3. ping neighbor IP (L3 connectivity)
4. debug ip ospf hello (see Hello packets)
5. debug ip ospf adj (see adjacency formation)

# Common issues:
# - "Mismatched hello parameters" → timer mismatch
# - "Mismatched area ID" → one side in area 0, other in area 1
# - No Hello received → firewall, wrong network statement
# - MTU mismatch → stuck in EXSTART
```

### Bài tập 3: Cost calculation

```
Given:
  Reference bandwidth = 100,000 Mbps (100 Gbps)
  
  Link A: 100 Mbps Ethernet → Cost = 100000/100 = 1000
  Link B: 1 Gbps Ethernet → Cost = 100000/1000 = 100
  Link C: 10 Gbps → Cost = 100000/10000 = 10
  Link D: 100 Gbps → Cost = 100000/100000 = 1

  Path 1: A → B → D = 1000 + 100 + 1 = 1101
  Path 2: A → C → C = 1000 + 10 + 10 = 1020 ← PREFERRED (lower cost)
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Key Points — Những điểm cần nhớ

```
╔══════════════════════════════════════════════════════════════════╗
║                     OSPF — TÓM TẮT                              ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. OSPF = Link-State protocol (mỗi router biết full topology) ║
║  2. Algorithm: Dijkstra SPF → shortest path tree                ║
║  3. Metric: Cost = Reference_BW / Interface_BW                  ║
║  4. Protocol 89 (IP protocol, NOT TCP/UDP)                      ║
║  5. Multicast: 224.0.0.5 (AllSPF), 224.0.0.6 (AllDR)          ║
║  6. Hello: 10s (broadcast), Dead: 40s                           ║
║  7. DR/BDR: Giảm overhead trên broadcast segments              ║
║  8. Areas: Hierarchical — PHẢI có Area 0 (backbone)            ║
║  9. LSA Types: 1(Router), 2(Network), 3(Summary), 5(External) ║
║  10. AD = 110 (higher trust than RIP=120)                       ║
║                                                                  ║
║  Convergence: 1-5 seconds (with BFD: sub-second)               ║
║  Scale: 100s-1000s of routers (with proper area design)         ║
║                                                                  ║
║  AWS Context:                                                    ║
║  • AWS KHÔNG run OSPF — dùng BGP cho VPN/DX                    ║
║  • On-prem OSPF → redistribute vào BGP → AWS                   ║
║  • Transit Gateway: BGP only (no OSPF/EIGRP)                   ║
║  • VPC routing: Static + BGP propagation                        ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Tài liệu đọc thêm

| Tài liệu | Link/Reference | Nội dung |
|-----------|---------------|----------|
| RFC 2328 | tools.ietf.org/html/rfc2328 | OSPF v2 Specification |
| RFC 5340 | tools.ietf.org/html/rfc5340 | OSPF v3 (IPv6) |
| Jeff Doyle — Routing TCP/IP Vol.1 | Ch. 6-9 | OSPF bible |
| Cisco OSPF Design Guide | cisco.com | Best practices |
| FRRouting Docs | docs.frrouting.org/en/latest/ospfd.html | Open source OSPF |
| GNS3/EVE-NG Labs | — | Hands-on practice |

---

*Bài tiếp theo: [EIGRP Protocol](/2026-06-01-eigrp-protocol) — Cisco's hybrid routing protocol*
