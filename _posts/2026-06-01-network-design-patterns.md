---
layout: post
title: "Network Design Patterns Deep Dive - Các Mô Hình Thiết Kế Mạng"
date: 2026-06-01
categories: [networking]
tags: [network-design, spine-leaf, hub-spoke, data-center, transit-gateway]
---

# Network Design Patterns Deep Dive - Các Mô Hình Thiết Kế Mạng

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Hãy nghĩ về cách tổ chức giao thông trong một thành phố:

- **3-tier (Core/Distribution/Access):** Giống hệ thống đường bộ — đường ngõ nhỏ (access) → đường quận (distribution) → đại lộ chính (core). Xe từ ngõ phải ra đường quận rồi mới lên đại lộ.

- **Spine-Leaf:** Giống hệ thống metro — mỗi ga (leaf) kết nối TRỰC TIẾP đến TẤT CẢ tuyến chính (spine). Đi từ ga nào đến ga nào cũng chỉ qua 1-2 bước.

- **Hub-Spoke:** Giống mạng bay VietJet — tất cả chuyến bay phải đi qua hub (Tân Sơn Nhất) để nối chuyến. Đà Nẵng → Tân Sơn Nhất → Phú Quốc.

- **Full Mesh:** Giống mạng bay nếu MỌI thành phố đều có đường bay trực tiếp đến mọi thành phố khác. Đắt nhưng nhanh nhất.

Mỗi mô hình có ưu/nhược điểm riêng, phù hợp với từng tình huống. Bài viết này sẽ phân tích chi tiết khi nào dùng mô hình nào.

---

## 2. Kiến Thức Nền Tảng

### 2.1 Yêu Cầu Thiết Kế Mạng

| Yêu cầu | Giải thích | Ví dụ |
|----------|-----------|-------|
| Scalability | Mở rộng dễ dàng | Thêm server/switch mà không redesign |
| Redundancy | Dự phòng | 1 thiết bị hỏng → mạng vẫn chạy |
| Performance | Hiệu suất | Latency thấp, throughput cao |
| Manageability | Dễ quản lý | Configuration đơn giản, troubleshooting dễ |
| Cost | Chi phí | Ít thiết bị, ít cáp = rẻ hơn |
| Security | Bảo mật | Segmentation, access control |

### 2.2 Traffic Patterns

```
North-South Traffic (Bắc-Nam):
Client (internet) → Firewall → Server
- Traffic đi VÀO/RA data center
- Truyền thống: chiếm 80% traffic
- Phù hợp: 3-tier hierarchical design

East-West Traffic (Đông-Tây):
Server A ←→ Server B (cùng data center)
- Traffic GIỮA các server (microservices, replication)
- Hiện đại: chiếm 70-80% traffic
- Phù hợp: Spine-leaf design (mọi server cách nhau = nhau)
```

### 2.3 Các Metrics Đánh Giá

```
Bisection Bandwidth: 
  Tổng bandwidth available khi chia mạng làm 2 nửa
  Cao hơn = East-West traffic tốt hơn

Oversubscription Ratio:
  Tổng access bandwidth / Tổng uplink bandwidth
  1:1 = non-blocking (lý tưởng, đắt)
  4:1 = chấp nhận được cho enterprise
  20:1 = quá cao, congestion risk

Hop Count:
  Số switch/router packets phải đi qua
  Ít hơn = latency thấp hơn, predictable hơn

Failure Domain:
  Khi 1 thiết bị hỏng → bao nhiêu hosts bị ảnh hưởng?
  Nhỏ hơn = resilient hơn
```

---

## 3. Three-Tier Architecture (Core/Distribution/Access)

### 3.1 Cấu Trúc

**Ví dụ đời thường:** Hệ thống giáo dục — Lớp học (Access) → Trường (Distribution) → Sở Giáo dục (Core). Mỗi tầng có vai trò riêng biệt.

```
                    ┌─────────┐
                    │  CORE   │         ← Backbone, routing giữa buildings
                    │ (2 SW)  │            High-speed, redundant
                    └────┬────┘
                   ╱     │     ╲
        ┌─────────┐  ┌─────────┐  ┌─────────┐
        │  DIST 1 │  │  DIST 2 │  │  DIST 3 │   ← Policy, VLAN routing
        │         │  │         │  │         │      Aggregation, filtering
        └────┬────┘  └────┬────┘  └────┬────┘
           ╱   ╲       ╱   ╲       ╱   ╲
      ┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐
      │ACC 1 ││ACC 2 ││ACC 3 ││ACC 4 ││ACC 5 ││ACC 6 │  ← Port density
      └──┬───┘└──┬───┘└──┬───┘└──┬───┘└──┬───┘└──┬───┘     User connection
         │        │        │        │        │        │
      [Users] [Users] [Servers] [Servers] [Users] [Users]
```

### 3.2 Vai Trò Từng Tầng

**Core Layer (Tầng lõi):**
```
Vai trò: "Xa lộ cao tốc" — chuyển tiếp packets NHANH NHẤT có thể
- High-speed switching/routing (40G/100G/400G)
- Kết nối tất cả Distribution switches
- KHÔNG làm policy/filtering (tốc độ là ưu tiên #1)
- Thường 2 switches (redundancy)
- Protocols: OSPF/EIGRP/BGP cho routing

KHÔNG nên làm ở Core:
✗ ACLs (chậm performance)
✗ VLAN creation
✗ Server hosting
✗ Packet inspection
```

**Distribution Layer (Tầng phân phối):**
```
Vai trò: "Ngã tư có đèn tín hiệu" — quyết định traffic đi đâu + policy
- Inter-VLAN routing (Layer 3 switching)
- Access Control Lists (ACLs) — security policies
- QoS marking/classification
- Route summarization
- Redundant uplinks đến Core
- Aggregate access switches

Là nơi "thông minh" nhất:
✓ Policy enforcement
✓ VLAN routing
✓ Redundancy (HSRP/VRRP)
✓ Route filtering
```

**Access Layer (Tầng truy cập):**
```
Vai trò: "Cổng vào nhà" — kết nối end devices trực tiếp
- Cung cấp port density (24/48 ports per switch)
- VLAN assignment (user → VLAN)
- Port security (MAC filtering, 802.1X)
- PoE (Power over Ethernet) cho IP phones, APs
- Spanning Tree (edge ports, portfast)

End devices kết nối:
- PCs, laptops, printers
- IP phones
- Wireless Access Points
- Servers (in server farm)
```

### 3.3 Ưu Nhược Điểm

**Ưu điểm:**
- Dễ hiểu, dễ troubleshoot (tầng nào làm gì rõ ràng)
- Scalable cho campus networks (thêm access switch = thêm users)
- Mature — hàng thập kỷ kinh nghiệm triển khai
- Phù hợp North-South traffic dominant

**Nhược điểm:**
- East-West traffic phải đi qua nhiều hop (access → dist → core → dist → access)
- Oversubscription cao ở uplinks
- Spanning Tree complexity (blocked ports = wasted bandwidth)
- Không optimal cho data center modern (microservices, containers)

**Khi nào dùng:** Campus networks, enterprise offices, nơi traffic chủ yếu là users → servers (North-South).

---

## 4. Spine-Leaf Architecture (Clos Network)

### 4.1 Cấu Trúc

**Ví dụ đời thường:** Hệ thống bến xe buýt — mỗi bến (leaf) kết nối đến TẤT CẢ tuyến xe chính (spine). Từ bến nào đến bến nào đều chỉ cần 1 lần đổi xe.

```
        ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
        │Spine 1 │  │Spine 2 │  │Spine 3 │  │Spine 4 │    ← SPINE layer
        └┬┬┬┬┬┬─┘  └┬┬┬┬┬┬─┘  └┬┬┬┬┬┬─┘  └┬┬┬┬┬┬─┘      (chỉ routing)
         │├┤├┤│      │├┤├┤│      │├┤├┤│      │├┤├┤│
         ││││││      ││││││      ││││││      ││││││
        ┌┴┴┴┴┴┴─┐  ┌┴┴┴┴┴┴─┐  ┌┴┴┴┴┴┴─┐  ┌┴┴┴┴┴┴─┐
        │Leaf 1  │  │Leaf 2  │  │Leaf 3  │  │Leaf 4  │    ← LEAF layer
        └┬┬┬┬┬──┘  └┬┬┬┬┬──┘  └┬┬┬┬┬──┘  └┬┬┬┬┬──┘      (server connection)
         │││││       │││││       │││││       │││││
      [Servers]   [Servers]   [Servers]   [Servers]

Rules:
1. EVERY Leaf connects to EVERY Spine
2. Leaves NEVER connect to each other
3. Spines NEVER connect to each other
4. Servers connect ONLY to Leaves
```

### 4.2 Tại Sao Spine-Leaf Cho Data Center?

```
Traffic pattern data center hiện đại:
- Microservices: Service A (Leaf 1) ↔ Service B (Leaf 3) ↔ DB (Leaf 4)
- 70-80% traffic là East-West (server-to-server)
- Cần PREDICTABLE latency (mọi pair cùng hop count)

Spine-Leaf đảm bảo:
- Max 2 hops (Leaf → Spine → Leaf) cho BẤT KỲ pair nào
- Predictable latency: luôn 2 hops, luôn bằng nhau
- ECMP (Equal-Cost Multi-Path): traffic spread across ALL spines
- Non-blocking: đủ spine = full bisection bandwidth
```

**ECMP (Equal-Cost Multi-Path):**
```
Server trên Leaf 1 gửi đến Server trên Leaf 4:
Path 1: Leaf1 → Spine1 → Leaf4
Path 2: Leaf1 → Spine2 → Leaf4
Path 3: Leaf1 → Spine3 → Leaf4
Path 4: Leaf1 → Spine4 → Leaf4

4 paths có CÙNG cost → traffic hash-based spread đều 4 paths
→ Aggregate bandwidth = 4x single link!
→ 1 Spine chết → 3 paths còn lại (graceful degradation)
```

### 4.3 Protocols Cho Spine-Leaf

```
BGP (Border Gateway Protocol) — Phổ biến nhất:
- eBGP giữa Leaf và Spine (mỗi switch = 1 AS)
- Stable, scalable, well-understood
- Dùng bởi: Facebook, Google, Microsoft, most hyperscalers

OSPF/IS-IS:
- Đơn giản hơn BGP cho small/medium deployments
- OSPF: unnumbered interfaces, BFD for fast failover

VXLAN + EVPN (overlay):
- VXLAN: Layer 2 extension over Layer 3 spine-leaf
- EVPN: Control plane cho VXLAN (MAC learning, ARP suppression)
- Dùng khi cần Layer 2 adjacency across racks
- Vendors: Cisco ACI, Arista, Cumulus/NVIDIA
```

### 4.4 Scaling Spine-Leaf

```
Thêm servers? → Thêm Leaf switches
Thêm bandwidth? → Thêm Spine switches
Cả hai đều KHÔNG disrupt existing topology!

Scale limits:
- Spine ports = max Leaf switches
- Leaf ports = max servers per leaf + spine uplinks

Ví dụ: 32-port spine switch, 48-port leaf switch
- 32 spines × 48 port leaves = 32 leaves
- Each leaf: 48 - 32 (uplinks) = 16 server ports
- Total servers: 32 leaves × 16 ports = 512 servers
- Muốn nhiều hơn? → Super-spine (3-tier Clos)

3-Tier Clos (Super-Spine):
Super-Spine ──── Spine ──── Leaf ──── Servers
(Pod-to-Pod)    (In-Pod)   (Servers)
→ Scale đến hàng trăm nghìn servers (Google, AWS)
```

### 4.5 So Sánh 3-Tier vs Spine-Leaf

| Tiêu chí | 3-Tier | Spine-Leaf |
|----------|--------|-----------|
| Traffic pattern tốt nhất | North-South | East-West |
| Max hop count | 4-6 | 2 (luôn) |
| Latency | Variable | Predictable |
| Scaling | Thêm switch phức tạp | Thêm leaf/spine đơn giản |
| Bandwidth utilization | ~50% (STP blocks) | ~100% (ECMP all paths) |
| Oversubscription | Thường 4:1 - 20:1 | Có thể 1:1 (non-blocking) |
| Complexity | Moderate (STP, VTP) | Moderate (BGP/EVPN) |
| Use case | Campus | Data Center |
| Cost efficiency | Tốt cho campus | Tốt cho DC |

---

## 5. Hub-Spoke (Star Topology)

### 5.1 Cấu Trúc

**Ví dụ đời thường:** Mạng lưới chi nhánh ngân hàng — tất cả chi nhánh kết nối về hội sở (hub). Chi nhánh A muốn nói chuyện với chi nhánh B → phải qua hội sở.

```
                    ┌─────────────┐
                    │     HUB     │
                    │ (Headquarters)│
                    └──────┬──────┘
                  ╱    ╱   │   ╲    ╲
                ╱    ╱     │     ╲    ╲
    ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
    │Spoke 1 │ │Spoke 2 │ │Spoke 3 │ │Spoke 4 │ │Spoke 5 │
    │(Branch)│ │(Branch)│ │(Branch)│ │(Branch)│ │(Branch)│
    └────────┘ └────────┘ └────────┘ └────────┘ └────────┘

- Hub: Data center chính, firewall, internet gateway
- Spokes: Branch offices, remote sites
- All traffic: Spoke → Hub → Destination
```

### 5.2 Ưu Nhược Điểm

**Ưu điểm:**
- **Đơn giản:** Mỗi spoke chỉ cần 1 tunnel/link đến hub
- **Centralized security:** Tất cả traffic qua hub → firewall/IPS tại hub inspect tất cả
- **Chi phí thấp:** N spokes = N links (không phải N² cho full mesh)
- **Dễ manage:** Policy thay đổi chỉ tại hub

**Nhược điểm:**
- **Hub = single point of failure:** Hub chết → TẤT CẢ communication chết
- **Suboptimal routing:** Spoke A → Hub → Spoke B (dù A và B cùng thành phố!)
- **Hub bottleneck:** Tất cả traffic dồn về hub → bandwidth/performance limit
- **Latency cao cho spoke-to-spoke:** 2 hops thay vì 1

**Khi nào dùng:** Enterprise WAN (branches → HQ), VPN topology, organizations cần centralized security inspection.

### 5.3 Partial Mesh (Hub-Spoke + Direct Links)

```
Giải pháp cho nhược điểm suboptimal routing:
- Giữ hub-spoke cho HẦU HẾT spokes
- Thêm direct links giữa spokes có traffic cao

        ┌─────┐
        │ HUB │
        └──┬──┘
       ╱   │   ╲
      ╱    │    ╲
    S1─────S2    S3
    │             │
    └─────────────┘  ← Direct link giữa S1 và S3 (traffic cao)

Tiết kiệm hơn full mesh nhưng tốt hơn pure hub-spoke
```

---

## 6. Full Mesh

### 6.1 Cấu Trúc

**Ví dụ đời thường:** Nhóm chat 5 người — mỗi người có direct message với TẤT CẢ 4 người còn lại. Không cần "người trung gian" → giao tiếp nhanh nhất.

```
    ┌──────┐───────┌──────┐
    │Node A│       │Node B│
    └──┬───┘╲   ╱ └───┬──┘
       │     ╲ ╱      │
       │      ╳       │
       │     ╱ ╲      │
    ┌──┴───┐╱   ╲┌───┴──┐
    │Node C│───────│Node D│
    └──────┘       └──────┘

Mỗi node kết nối TRỰC TIẾP với TẤT CẢ nodes khác
Connections needed: N × (N-1) / 2

4 nodes: 4 × 3 / 2 = 6 links
10 nodes: 10 × 9 / 2 = 45 links
50 nodes: 50 × 49 / 2 = 1,225 links ← Explosion!
100 nodes: 100 × 99 / 2 = 4,950 links ← Impossible to manage!
```

### 6.2 Ưu Nhược Điểm

**Ưu điểm:**
- **Latency tối thiểu:** Luôn 1 hop (direct connection)
- **Redundancy tối đa:** 1 link chết → nhiều paths thay thế
- **No bottleneck:** Không có central point giới hạn bandwidth
- **Independent:** Mỗi pair có dedicated bandwidth

**Nhược điểm:**
- **Chi phí cực cao:** N² links, N² interfaces
- **Quản lý phức tạp:** 100 nodes = 4,950 tunnels/links configure
- **KHÔNG scale:** Số links tăng quadratic (O(N²))
- **Wasted resources:** Nhiều links ít traffic nhưng vẫn phải maintain

**Khi nào dùng:** 
- Chỉ với SỐ ÍT nodes (3-5 sites quan trọng)
- Core router mesh (2-4 core routers)
- BGP peering giữa major ISPs
- KHÔNG dùng cho branch offices (quá nhiều)

---

## 7. Collapsed Core (Two-Tier)

### 7.1 Cấu Trúc

**Ví dụ đời thường:** Trong căn hộ nhỏ, bếp và phòng ăn kết hợp chung 1 phòng (thay vì 2 phòng riêng biệt) để tiết kiệm không gian.

```
3-Tier (đầy đủ):                   Collapsed Core (2-tier):
┌──────┐                            ┌─────────────────┐
│ CORE │                            │  CORE+DIST      │  ← Kết hợp 2 tầng
└──┬───┘                            │  (1 switch đảm  │
   │                                │   cả 2 vai trò) │
┌──┴───┐                            └────────┬────────┘
│ DIST │                                ╱    │    ╲
└──┬───┘                           ┌─────┐┌─────┐┌─────┐
   │                               │ACC 1││ACC 2││ACC 3│
┌──┴───┐┌─────┐┌─────┐            └─────┘└─────┘└─────┘
│ACC 1 ││ACC 2││ACC 3│
└──────┘└─────┘└─────┘

Core + Distribution → 1 layer (collapsed)
```

### 7.2 Khi Nào Dùng

```
✅ Dùng Collapsed Core khi:
- Small/medium campus (< 200 users)
- 1-2 buildings
- Budget limited
- Không cần tách Core vs Distribution
- Single site (không có WAN interconnect phức tạp)

❌ Dùng Full 3-Tier khi:
- Large campus (> 200 users hoặc multiple buildings)
- Cần route summarization giữa buildings
- Cần separate failure domains
- Growth planned (will need to split later)
```

---

## 8. AWS Transit Gateway Topologies

### 8.1 Transit Gateway Là Gì?

**Ví dụ đời thường:** Transit Gateway giống như **bến xe trung tâm** — tất cả tuyến xe (VPCs, VPNs, Direct Connect) đều kết nối vào bến, và từ bến có thể đi đến bất kỳ tuyến nào khác.

```
TRƯỚC Transit Gateway (Full Mesh VPC Peering):
VPC-A ──peering──── VPC-B
VPC-A ──peering──── VPC-C
VPC-B ──peering──── VPC-C
VPC-A ──peering──── VPC-D
... O(N²) peering connections!

SAU Transit Gateway (Hub):
VPC-A ──attach──┐
VPC-B ──attach──┤
VPC-C ──attach──┼── Transit Gateway ──── On-premises (VPN/DX)
VPC-D ──attach──┤
VPC-E ──attach──┘
N attachments only! (Linear scaling)
```

### 8.2 TGW Hub-Spoke Pattern

```
┌──────────────────────────────────────┐
│           Transit Gateway             │
│                                      │
│  Route Table: Default                │
│  10.0.0.0/16 → VPC-A attachment      │
│  10.1.0.0/16 → VPC-B attachment      │
│  10.2.0.0/16 → VPC-C attachment      │
│  192.168.0.0/16 → VPN attachment     │
│  0.0.0.0/0 → Inspection VPC          │
└──┬──────┬───────┬──────┬─────────────┘
   │      │       │      │
┌──┴──┐┌──┴──┐┌───┴─┐┌───┴────┐
│VPC-A││VPC-B││VPC-C││VPN/DX  │
│Prod ││Dev  ││Shared││On-prem │
└─────┘└─────┘└─────┘└────────┘
```

### 8.3 TGW with Shared Services VPC

```
Pattern: Centralized shared services (DNS, AD, NAT)

TGW Route Tables:
├── Spoke RT (cho VPC-A, VPC-B):
│   - 10.0.0.0/8 → TGW (inter-VPC via TGW)
│   - 0.0.0.0/0 → Shared Services VPC (NAT/Internet)
│
└── Shared Services RT (cho Shared VPC):
    - 10.1.0.0/16 → VPC-A attachment
    - 10.2.0.0/16 → VPC-B attachment
    - 0.0.0.0/0 → Internet Gateway

                    ┌──────────────┐
                    │Shared Service│
                    │  VPC (NAT,   │
                    │  DNS, AD)    │
                    └──────┬───────┘
                           │
                    ┌──────┴───────┐
                    │Transit Gateway│
                    └──┬───────┬───┘
                       │       │
                 ┌─────┴─┐  ┌──┴────┐
                 │VPC-A  │  │VPC-B  │
                 │(Prod) │  │(Dev)  │
                 └───────┘  └───────┘
```

### 8.4 TGW with Inspection VPC (Security)

```
Pattern: All inter-VPC traffic passes through firewall

TGW Route Tables:
├── Spoke RT (VPC-A, VPC-B, VPC-C):
│   ALL routes → Inspection VPC attachment
│
└── Inspection RT (Inspection VPC):
    - 10.1.0.0/16 → VPC-A attachment
    - 10.2.0.0/16 → VPC-B attachment  
    - 10.3.0.0/16 → VPC-C attachment

Flow: VPC-A → TGW (Spoke RT) → Inspection VPC → Firewall 
      → Inspection VPC → TGW (Inspection RT) → VPC-B

                    ┌───────────────┐
                    │ Inspection VPC│
                    │  ┌─────────┐  │
                    │  │Firewall │  │
                    │  └─────────┘  │
                    └──────┬────────┘
                           │
                    ┌──────┴───────┐
                    │Transit Gateway│
                    └─┬──────┬───┬─┘
                      │      │   │
                ┌─────┴┐ ┌──┴──┐┌┴─────┐
                │VPC-A │ │VPC-B││VPC-C │
                └──────┘ └─────┘└──────┘
```

### 8.5 Multi-Region TGW Peering

```
Pattern: Global network interconnecting regions

    Region us-east-1              Region eu-west-1
    ┌──────────────┐              ┌──────────────┐
    │   TGW-USE1   │──TGW Peer──→│   TGW-EUW1   │
    └───┬──────┬───┘              └───┬──────┬───┘
        │      │                      │      │
    ┌───┴┐ ┌──┴──┐               ┌───┴┐ ┌──┴──┐
    │VPC1│ │VPC2 │               │VPC3│ │VPC4 │
    └────┘ └─────┘               └────┘ └─────┘

TGW Peering:
- Inter-region connectivity
- Static routes (no dynamic routing across peering)
- Data encrypted in transit (AWS backbone)
- Bandwidth: up to 50 Gbps per peering attachment
```

### 8.6 TGW Pricing Considerations

```
Transit Gateway costs:
1. Attachment: $0.05/hour per attachment (~$36/month)
   - VPC, VPN, Direct Connect, Peering = each $0.05/hr
   
2. Data processing: $0.02/GB processed
   - Every GB traversing TGW = $0.02
   - Inter-AZ within same VPC = FREE (doesn't use TGW)
   
Cost optimization:
- VPC Peering: FREE data transfer (no TGW processing fee)
  → Use peering for high-bandwidth VPC-to-VPC if only 2-3 VPCs
- TGW: Worth it when N > 3-5 VPCs (management simplicity)
- Avoid "hairpin" through TGW for intra-VPC traffic
```

---

## 9. Choosing The Right Design

### 9.1 Decision Matrix

| Scenario | Recommended Pattern |
|----------|-------------------|
| Enterprise campus (100-1000 users) | 3-Tier hierarchical |
| Small office (< 100 users) | Collapsed Core (2-Tier) |
| Data center (servers, containers) | Spine-Leaf |
| Branch offices → HQ | Hub-Spoke |
| 2-5 critical sites | Full Mesh |
| Many branches + some mesh | Partial Mesh |
| AWS multi-VPC | Transit Gateway (Hub) |
| AWS 2-3 VPCs only | VPC Peering (cheaper) |
| Global AWS multi-region | TGW Peering |

### 9.2 Hybrid Approaches

```
Real-world networks thường KẾT HỢP nhiều patterns:

Enterprise Example:
├── Campus: 3-Tier (Core → Distribution → Access)
├── Data Center: Spine-Leaf (servers, east-west traffic)
├── WAN: Hub-Spoke (branches → DC via MPLS/SD-WAN)
├── Core sites: Full Mesh (3 main DCs, full redundancy)
└── Cloud: TGW Hub-Spoke (VPCs via Transit Gateway)

AWS Example:
├── Within Region: TGW + VPC Peering (hybrid)
│   - High-traffic VPC pairs: Direct Peering ($0 data)
│   - All others: Through TGW (simplicity)
├── Cross Region: TGW Inter-Region Peering
├── On-premises: Direct Connect → TGW
└── Per-VPC: Standard subnetting (public/private/data)
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 Quick Reference

```
┌─────────────────────────────────────────────────────┐
│  Pattern        │ Best For          │ Scale    │ Cost │
├─────────────────┼───────────────────┼──────────┼──────┤
│ 3-Tier          │ Campus            │ Good     │ Med  │
│ Spine-Leaf      │ Data Center       │ Excellent│ High │
│ Hub-Spoke       │ WAN/Branch        │ Good     │ Low  │
│ Full Mesh       │ Few critical sites│ Bad      │ High │
│ Collapsed Core  │ Small sites       │ Limited  │ Low  │
│ TGW             │ AWS multi-VPC     │ Excellent│ Med  │
└─────────────────┴───────────────────┴──────────┴──────┘
```

### 10.2 Key Takeaways

1. **3-Tier** cho campus (North-South traffic) — proven, mature, dễ troubleshoot
2. **Spine-Leaf** cho data center (East-West traffic) — predictable latency, ECMP
3. **Hub-Spoke** cho WAN — đơn giản, centralized security, nhưng hub = bottleneck
4. **Full Mesh** chỉ cho 3-5 nodes — O(N²) không scale
5. **AWS Transit Gateway** = cloud hub-spoke — đơn giản hóa multi-VPC networking
6. **ECMP** là "magic sauce" của spine-leaf — tất cả paths đều active
7. **Real networks = hybrid** — kết hợp nhiều patterns cho từng phần
8. **Cost vs Performance** — full mesh nhanh nhất nhưng đắt nhất

### 10.3 Tài Liệu Tham Khảo

- IEEE: "A Scalable, Commodity Data Center Network Architecture" (VL2, Microsoft)
- Al-Fares et al., "A Scalable, Commodity Data Center Network Architecture" (Fat-Tree/Clos, SIGCOMM 2008)
- RFC 7938: Use of BGP for Routing in Large-Scale Data Centers
- Cisco Validated Designs: Campus Network Design Guide
- AWS Documentation: Transit Gateway
- AWS Architecture Blog: Networking patterns
- "CLOS Network Architecture" — Charles Clos (1953) original paper
- Arista/Cisco/Juniper spine-leaf design guides
- Facebook's Data Center Fabric paper (SIGCOMM 2014)

---

*Bài viết tiếp theo: Linux Filesystem Deep Dive — Hiểu sâu về hệ thống file trên Linux*
