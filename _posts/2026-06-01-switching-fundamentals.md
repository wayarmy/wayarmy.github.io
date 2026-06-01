---
layout: post
title: "Switching Fundamentals — Store-and-Forward, MAC Learning, và Collision Domains"
subtitle: "Hiểu sâu về cách Switch hoạt động — từ Hub đến Switch đến Layer 3 Switch"
tags: [networking, switching, layer2, mac-table, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| IEEE 802.1D-2004 | MAC Bridges Standard |
| IEEE 802.1Q-2022 | Bridges and Bridged Networks (VLANs) |
| Cisco — Catalyst Switch Configuration Guide | Layer 2 Switching Concepts |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 4: MAC Sublayer — Bridges |
| Perlman, R. — Interconnections, 2nd Ed. | Bridges and Switches |
| Cisco White Paper | Cut-Through and Store-and-Forward Ethernet Switching |

---

## 1. Giới thiệu — Tại sao cần biết Switching?

### Ví dụ đời thường: Từ loa phát thanh đến điện thoại trực tiếp

**Thời kỳ Hub (trước switch):** Giống như **loa phát thanh trong làng** — khi trưởng thôn nói qua loa, TẤT CẢ mọi người trong làng đều nghe, dù thông tin chỉ dành cho 1 hộ gia đình. Ai muốn nói phải đợi loa rảnh.

**Thời kỳ Switch (hiện đại):** Giống như **tổng đài điện thoại** — khi bạn gọi cho ai, chỉ người đó nhận cuộc gọi. Nhiều cuộc gọi diễn ra đồng thời trên cùng hệ thống mà không ảnh hưởng nhau.

Switch chính là thiết bị đã **cách mạng hóa** networking — chuyển từ shared medium (ai cũng nghe) sang dedicated links (chỉ người cần nhận mới nhận).

### Concrete scenario: "Hãy tưởng tượng bạn đang..."

Hãy tưởng tượng bạn là quản lý IT trong công ty 50 người. Sáng thứ Hai, tất cả 50 nhân viên bật máy tính cùng lúc:

- **Nếu dùng Hub:** 50 máy chia nhau 100 Mbps bandwidth. Mỗi máy chỉ có ~2 Mbps thực tế. Khi máy A gửi cho máy B, 48 máy khác cũng nhận → lãng phí + security risk.
- **Nếu dùng Switch:** Mỗi máy có dedicated 1 Gbps link. Máy A gửi cho máy B qua kết nối riêng — 48 máy khác không hề bị ảnh hưởng. 50 conversations đồng thời!

### Vấn đề Switch giải quyết

| Vấn đề với Hub | Switch giải quyết thế nào |
|----------------|--------------------------|
| 1 collision domain = tất cả share bandwidth | Mỗi port = 1 collision domain riêng |
| Khi A gửi cho B, C cũng nhận (security!) | Unicast frame chỉ ra đúng port đích |
| Half-duplex bắt buộc (CSMA/CD) | Full-duplex (gửi + nhận đồng thời) |
| Bandwidth chia cho N thiết bị | Mỗi thiết bị có full bandwidth |
| Collision tăng khi thêm thiết bị | Không collision trong full-duplex |

---

## 2. Switch là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**Switch (network switch)** là thiết bị "bưu điện thông minh" — nó nhận "phong bì" (Ethernet frame) từ 1 cổng (port), đọc "địa chỉ người nhận" (destination MAC), rồi chỉ gửi ra **đúng cổng** nơi người nhận kết nối. Nó KHÔNG gửi đại cho tất cả (trừ broadcast).

### Analogy: Tổng đài viên điện thoại ngày xưa

```
┌───────────────────────────────────────────────────────────────┐
│                SWITCH = TỔNG ĐÀI VIÊN                         │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  HUB (loa phát thanh):           SWITCH (tổng đài):           │
│  ┌────┐                          ┌────┐                      │
│  │ A ─┤───┐                      │ A ─┤──╮                   │
│  ├────┤   │  Mọi người           ├────┤  │  Chỉ B            │
│  │ B ─┤───┤  đều nghe            │ B ─┤──╯  nhận             │
│  ├────┤   │  khi A nói           ├────┤                      │
│  │ C ─┤───┤                      │ C ─┤     C không          │
│  ├────┤   │                      ├────┤     biết gì          │
│  │ D ─┤───┘                      │ D ─┤                      │
│  └────┘                          └────┘                      │
│                                                                │
│  Giống: Hét trong phòng          Giống: Nói thẳng             │
│         → mọi người nghe                vào tai người cần     │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

### Collision Domain vs Broadcast Domain

```
═══════════════════════════════════════════════════════════
HUB — 1 Collision Domain, 1 Broadcast Domain:

  PC-A ──┐
  PC-B ──┤── HUB ──┤── PC-C
  PC-D ──┘         └── PC-E
  
  → Tất cả 5 PC cùng 1 collision domain
  → Nếu A và B gửi cùng lúc = COLLISION!
  → Bandwidth chia 5
  
═══════════════════════════════════════════════════════════
SWITCH — N Collision Domains, 1 Broadcast Domain:

  PC-A ──[Port 1]──┐
  PC-B ──[Port 2]──┤── SWITCH ──[Port 4]── PC-D
  PC-C ──[Port 3]──┘            [Port 5]── PC-E
  
  → 5 collision domains riêng biệt (1 per port)
  → A và B gửi cùng lúc = NO collision (khác port)
  → Mỗi PC có full bandwidth
  → NHƯNG broadcast frame vẫn đi đến tất cả ports
  
═══════════════════════════════════════════════════════════
ROUTER — Tách cả Collision Domain và Broadcast Domain:

  [Switch 1] ── [Router] ── [Switch 2]
  (Broadcast    (chặn      (Broadcast
   Domain 1)    broadcast)  Domain 2)
```

---

## 3. Quá trình Switching — Switch xử lý frame thế nào?

### Ví dụ đời thường: Bưu tá thông minh

Bưu tá (switch) nhận thư (frame) và phải quyết định:
1. **Đọc địa chỉ gửi** (source MAC) → ghi nhớ "người này ở ngõ này" (port)
2. **Đọc địa chỉ nhận** (destination MAC) → tìm trong sổ "người nhận ở ngõ nào?"
3. **Nếu biết** → đưa thư đúng ngõ (forward to specific port)
4. **Nếu KHÔNG biết** → phát thư cho tất cả ngõ (flood to all ports except source)
5. **Nếu là thư quảng cáo** (broadcast) → gửi cho tất cả ngõ

### 3.1 Ba bước chính: Learn → Lookup → Forward/Flood

```
FRAME VÀO PORT Fa0/1:
┌────────────────────────────────────────────────────┐
│ Dst MAC: BB:BB:BB:BB:BB:BB │ Src MAC: AA:AA:AA:AA │
└────────────────────────────────────────────────────┘

BƯỚC 1 — LEARN (Học Source MAC):
"Source MAC AA:AA ở port Fa0/1" → Lưu vào MAC Table
→ Reset aging timer cho entry này

BƯỚC 2 — LOOKUP (Tìm Destination MAC):
Tìm BB:BB trong MAC Address Table:
  - Tìm thấy → port Fa0/3? → FORWARD ra Fa0/3
  - KHÔNG tìm thấy → FLOOD ra tất cả ports (trừ Fa0/1)
  - Dst = FF:FF:FF:FF:FF:FF → BROADCAST ra tất cả ports (trừ Fa0/1)
  - Dst MAC = Src port (cùng port) → DROP (sao lại gửi về chính mình?)

BƯỚC 3 — FORWARD/FILTER/FLOOD:
  ┌────────────────────────────────────────────────┐
  │ Forward: Gửi ra 1 port cụ thể (known unicast) │
  │ Filter:  DROP frame (dst = src port)           │
  │ Flood:   Gửi ra TẤT CẢ ports trừ source       │
  │ Broadcast: Gửi ra TẤT CẢ ports trừ source     │
  └────────────────────────────────────────────────┘
```

### 3.2 MAC Address Table — Bộ nhớ "đường đi" của Switch

```bash
# Cisco — xem MAC table
Switch# show mac address-table
          Mac Address Table
-------------------------------------------
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
   1    0050.56c0.0001    DYNAMIC     Fa0/1
   1    0050.56c0.0002    DYNAMIC     Fa0/2
   1    aabb.cc00.0100    STATIC      Fa0/24
   1    ffff.ffff.ffff    STATIC      CPU

Total Mac Addresses for this criterion: 4
```

**Các loại entry:**
- **DYNAMIC:** Học tự động từ source MAC → có aging timer
- **STATIC:** Admin cấu hình thủ công → không bao giờ expire
- **SECURE:** Từ port-security → violation nếu MAC lạ xuất hiện

---

## 4. Switching Methods — Cách switch xử lý frame

### Ví dụ đời thường: Kiểm tra bưu phẩm

- **Store-and-Forward:** Bưu tá **nhận TOÀN BỘ gói hàng**, cân nặng, kiểm tra bao bì có nguyên vẹn không, rồi mới gửi đi. Chậm hơn nhưng chắc chắn hàng tốt.
- **Cut-Through:** Bưu tá chỉ **đọc địa chỉ** trên nhãn (6 bytes đầu) rồi chuyển tiếp NGAY — không kiểm tra gì. Nhanh nhưng có thể gửi hàng hỏng.
- **Fragment-Free:** Bưu tá đọc **64 bytes đầu** (đảm bảo không phải "mảnh vỡ" từ collision) rồi chuyển tiếp. Cân bằng giữa speed và safety.

### 4.1 Store-and-Forward Switching

```
Frame arrive ──→ [Buffer TOÀN BỘ frame] ──→ [Check FCS/CRC] ──→ Forward
                                                    │
                                              FCS lỗi?
                                              → DROP frame!

Latency: Phụ thuộc vào frame size
  - 64 bytes: ~5.12 μs (at 100Mbps)
  - 1518 bytes: ~121.44 μs (at 100Mbps)
  - 1518 bytes: ~12.14 μs (at 1Gbps)
```

**Ưu điểm:**
- Loại bỏ frame lỗi (CRC error) → không propagate bad frames
- Hỗ trợ QoS (có thể đọc toàn bộ frame để classify)
- Cho phép chuyển đổi tốc độ (ví dụ: 100Mbps → 1Gbps)

**Nhược điểm:**
- Latency cao hơn (phải buffer toàn bộ frame)
- Cần buffer memory lớn hơn

### 4.2 Cut-Through Switching

```
Frame arrive ──→ [Đọc 6 bytes Dst MAC] ──→ Lookup ──→ Forward NGAY
                  (chỉ 14 bytes header)       ↑
                                              │ Frame tiếp tục arrive
                                              │ và được forward real-time
                                              │ (pipeline)
                                              
Latency: Cố định!
  - Luôn ~2.1 μs (at 100Mbps) — chỉ đọc 14 bytes header
  - Không phụ thuộc frame size
```

**Ưu điểm:**
- **Ultra-low latency** — critical cho HPC, trading, real-time
- Latency cố định (deterministic)

**Nhược điểm:**
- Forward cả frame lỗi (CRC bad) → propagate errors
- Không thể QoS (chưa đọc hết frame)
- Không thể switch tốc độ khác nhau giữa ports

### 4.3 Fragment-Free (Modified Cut-Through)

```
Frame arrive ──→ [Buffer 64 bytes đầu] ──→ Lookup ──→ Forward
                  (loại bỏ runts/collision fragments)

Logic: Nếu frame ≥ 64 bytes → rất likely không phải collision fragment
       (vì collision chỉ tạo fragments < 64 bytes)
```

### 4.4 So sánh tổng hợp

| Đặc điểm | Store-and-Forward | Cut-Through | Fragment-Free |
|-----------|-------------------|-------------|---------------|
| Latency | Variable (frame-size dependent) | Fixed (~2.1μs) | Fixed (~5.1μs) |
| Error checking | Full CRC check | None | Partial (≥64 bytes) |
| Buffer needed | Full frame | Minimal (14 bytes) | 64 bytes |
| QoS support | Yes | No | Limited |
| Speed conversion | Yes | No (same speed ports) | No |
| Use case | Enterprise, default | HPC, trading floor | Legacy |
| Bad frame handling | DROP | Forward (propagate) | Drop runts only |

**Trong thực tế:**
- **Enterprise switches** (Cisco Catalyst, Juniper EX): Store-and-Forward (default)
- **Data center low-latency** (Arista 7000, Cisco Nexus): Cut-Through mode available
- **High-Frequency Trading**: Cut-Through là BẮT BUỘC (mỗi microsecond = $$)

---

## 5. Layer 2 vs Layer 3 Switch — Khi nào cần gì?

### Ví dụ đời thường

- **Layer 2 Switch** = bưu tá trong **1 thành phố** — biết mọi địa chỉ nhà (MAC) trong thành phố, nhưng không biết gửi đi tỉnh khác
- **Layer 3 Switch** = bưu tá biết cả **mã bưu chính** (IP) — có thể route thư đi tỉnh khác mà không cần qua bưu điện trung tâm (router)

### 5.1 Layer 2 Switch

```
Quyết định dựa trên: Destination MAC Address
Bảng tham chiếu: MAC Address Table (CAM Table)
Xử lý: Frame-level (Ethernet frames)
Broadcast: Forward tất cả broadcast ra mọi port

┌─────────────────────────────────────────┐
│        LAYER 2 SWITCH                    │
│                                          │
│  Frame vào → Đọc Dst MAC → Tìm port    │
│                              │           │
│              ┌───────────────┼─────┐     │
│              │ MAC Table     │     │     │
│              │ AA → Port 1   │     │     │
│              │ BB → Port 3   ←─────┘     │
│              │ CC → Port 5         │     │
│              └─────────────────────┘     │
└─────────────────────────────────────────┘
```

### 5.2 Layer 3 Switch (Multilayer Switch)

```
Quyết định dựa trên: Destination IP Address (cho inter-VLAN traffic)
Bảng tham chiếu: MAC Table + Routing Table + ARP Table
Xử lý: Cả Frame-level VÀ Packet-level
Broadcast: Chặn ở boundary VLAN (như router)

┌─────────────────────────────────────────────────────┐
│        LAYER 3 SWITCH                                │
│                                                      │
│  Intra-VLAN: L2 switching (MAC table, fast)         │
│  Inter-VLAN: L3 routing (IP routing, hardware-based)│
│                                                      │
│  ┌─── SVI: VLAN 10 (192.168.10.1) ───┐             │
│  │    Ports: Fa0/1-12                  │             │
│  └─────────────────────────────────────┘             │
│  ┌─── SVI: VLAN 20 (192.168.20.1) ───┐             │
│  │    Ports: Fa0/13-24                 │             │
│  └─────────────────────────────────────┘             │
│                                                      │
│  PC (VLAN10) → PC (VLAN20): Routed bởi L3 switch!  │
└─────────────────────────────────────────────────────┘
```

### 5.3 So sánh

| Feature | L2 Switch | L3 Switch | Router |
|---------|-----------|-----------|--------|
| Forwarding basis | MAC Address | MAC + IP | IP Address |
| Speed | Wire-speed (hardware) | Wire-speed (ASIC) | Slower (software) |
| Inter-VLAN routing | ✗ (cần router) | ✓ (built-in) | ✓ |
| Routing protocols | ✗ | ✓ (OSPF, BGP, etc.) | ✓ |
| ACL support | Basic (MAC) | Advanced (IP, L4 ports) | Full |
| WAN interfaces | ✗ | Limited | ✓ |
| Cost | $ | $$ | $$$ |
| Use case | Access layer | Distribution/Core | WAN edge, Internet |

---

## 6. Advanced Switching Concepts

### 6.1 Microsegmentation — Mỗi port là 1 segment

Với switch, mỗi port tạo thành **1 collision domain riêng**. Nếu dùng full-duplex:
- **Không collision** (CSMA/CD disabled)
- **Bandwidth full** cả hai chiều (1 Gbps TX + 1 Gbps RX = 2 Gbps total)
- **Dedicated link** giữa switch và thiết bị

### 6.2 Symmetric vs Asymmetric Switching

**Symmetric:** Tất cả ports cùng tốc độ (24× 1GbE)
**Asymmetric:** Ports có tốc độ khác nhau (24× 1GbE + 4× 10GbE uplinks)

```
Asymmetric Switch:
┌─────────────────────────────────────┐
│  Access Ports (1 GbE)    │ Uplinks  │
│  ┌──┐┌──┐┌──┐... ┌──┐  │ ┌────┐   │
│  │P1││P2││P3│    │P24│  │ │10GE│   │
│  └──┘└──┘└──┘    └──┘  │ │ ×4 │   │
│                          │ └────┘   │
└─────────────────────────────────────┘

Khi 24 PCs gửi đồng thời lên server (qua 10G uplink):
- Tổng max traffic = 24 × 1G = 24 Gbps
- Uplink capacity = 4 × 10G = 40 Gbps ← đủ! (non-blocking)
- Nếu chỉ có 2 × 10G uplink = 20 Gbps ← không đủ (oversubscribed)
```

### 6.3 Oversubscription Ratio

```
Oversubscription = Total access bandwidth / Total uplink bandwidth

Ví dụ: 48 × 1G access ports, 4 × 10G uplinks
= 48 Gbps / 40 Gbps = 1.2:1 ← EXCELLENT (gần non-blocking)

Ví dụ: 48 × 1G access ports, 2 × 10G uplinks  
= 48 Gbps / 20 Gbps = 2.4:1 ← Acceptable cho office

Ví dụ: 48 × 1G access ports, 1 × 10G uplink
= 48 Gbps / 10 Gbps = 4.8:1 ← CẢNH BÁO! Congestion likely

Data Center best practice: 3:1 hoặc tốt hơn
```

### 6.4 Switch Fabric — Internal Architecture

```
┌─────────────── Switch Internal ───────────────┐
│                                                │
│  Port 1 ─→ [Ingress]──┐                      │
│  Port 2 ─→ [Ingress]──┤                      │
│  Port 3 ─→ [Ingress]──┤── SWITCH ──→ [Egress]→ Port 1
│  ...                   │   FABRIC    [Egress]→ Port 2
│  Port 48─→ [Ingress]──┘   (ASIC)    [Egress]→ Port 3
│                                      ...      │
│                                      [Egress]→ Port 48
│                                                │
│  Switch Fabric capacity = Tổng throughput      │
│  Ví dụ: 48-port 1G switch                     │
│  Non-blocking capacity = 48 × 1G × 2          │
│  = 96 Gbps (full-duplex)                      │
└────────────────────────────────────────────────┘
```

---

## 7. Switching trong AWS — Mapping sang Cloud

### 7.1 VPC — Virtual "Switch" của AWS

Trong AWS, không có "switch" vật lý mà bạn configure. Nhưng **VPC Subnet** hoạt động tương tự L2 switch:

| Switch Concept | AWS Equivalent |
|---------------|---------------|
| Port | ENI attachment point |
| MAC Address Table | VPC forwarding tables (managed by AWS) |
| Broadcast domain | Subnet |
| VLAN | Subnet + Security Group |
| Trunk port | ENI with multiple IPs |
| L3 Switch (inter-VLAN) | VPC Router (implicit) |
| Spanning Tree (redundancy) | AWS handles — no STP needed |
| Uplink oversubscription | AWS guarantees bandwidth per instance type |

### 7.2 AWS Network Infrastructure

```
AWS Physical Layer (simplified):
┌─────────────────────────────────────────────────────────┐
│  Hypervisor Host                                         │
│  ┌────────┐  ┌────────┐  ┌────────┐                   │
│  │ EC2 #1 │  │ EC2 #2 │  │ EC2 #3 │                   │
│  │ ENI    │  │ ENI    │  │ ENI    │                   │
│  └───┬────┘  └───┬────┘  └───┬────┘                   │
│      │           │           │                          │
│  ════╧═══════════╧═══════════╧══ Virtual Switch (Nitro)│
│                      │                                   │
│              ┌───────┴────────┐                         │
│              │ Nitro Card     │ ← Hardware offload       │
│              │ (ENA driver)   │   networking             │
│              └───────┬────────┘                         │
└──────────────────────┼──────────────────────────────────┘
                       │ (25-100 GbE physical)
                ┌──────┴──────┐
                │ ToR Switch  │ ← Physical datacenter switch
                │ (Leaf)      │
                └─────────────┘
```

### 7.3 EC2 Instance Bandwidth

| Instance Family | Network Bandwidth | ENA version |
|----------------|-------------------|-------------|
| t3.micro | Up to 5 Gbps (burst) | ENA |
| m5.large | Up to 10 Gbps | ENA |
| m5.xlarge | Up to 10 Gbps | ENA |
| c5n.18xlarge | 100 Gbps | ENA Express |
| p4d.24xlarge | 400 Gbps (4×100G) | EFA |

---

## 8. Tình huống thực tế — Switch được sử dụng như thế nào?

### Scenario 1: Home/SOHO — Unmanaged Switch

**Anh Tuấn có 5 thiết bị** cần cắm mạng dây (router chỉ có 4 port LAN):

```
Internet ── Modem ── Router (4 ports)
                        │
         ┌──────────────┼──────┐
         │              │      │
        PC          Smart TV  ──→ Switch 8 ports (unmanaged)
                                      │
                        ┌─────────────┼─────────────┐
                        │             │             │
                     NAS Server   Game Console   IP Camera
```

- Switch unmanaged: Plug-and-play, không cần cấu hình
- Auto-negotiation: Tự detect 100M/1G
- Store-and-forward: Default trên consumer switches
- Tất cả thiết bị cùng 1 broadcast domain, cùng subnet

### Scenario 2: Enterprise — Managed Switch + VLANs

```
┌─── Core Layer ─────────────────────────────────────────────┐
│  2x Nexus 9000 (L3 Switch, 100G ports)                     │
│  OSPF/BGP routing, VXLAN, redundancy                       │
└────────────────────────┬───────────────────────────────────┘
                         │ 10G/40G uplinks (LACP bond)
┌─── Distribution Layer ─┼───────────────────────────────────┐
│  4x Catalyst 9300 (L3 Switch)                              │
│  Inter-VLAN routing, ACLs, QoS                             │
└────────────────────────┬───────────────────────────────────┘
                         │ 1G/10G uplinks
┌─── Access Layer ───────┼───────────────────────────────────┐
│  20x Catalyst 2960 (L2 Switch)                             │
│  Port security, 802.1X, VLAN assignment                    │
│  48 ports × 1GbE = 960 user ports total                    │
└────────────────────────────────────────────────────────────┘
```

### Scenario 3: Data Center — Spine-Leaf với Cut-Through

```
SPINE (4 switches):
- Arista 7500R, 32× 400G ports
- Cut-through switching (latency < 1μs!)
- No STP — ECMP routing

LEAF (32 switches):
- Arista 7280R, 48× 25G + 8× 100G
- Connect to servers (25G each)
- VXLAN EVPN overlay

TOTAL CAPACITY:
- 32 leafs × 48 servers = 1,536 servers
- Each server: 25 Gbps dedicated
- Non-blocking fabric: any-to-any full bandwidth
```

---

## 9. Bài tập thực hành

### Exercise 1: Observe MAC Learning

```bash
# Bước 1: Trên Cisco switch (hoặc GNS3/EVE-NG lab)
Switch# show mac address-table
# → Bảng có thể trống

# Bước 2: Từ PC-A, ping PC-B
PC-A> ping 192.168.1.2

# Bước 3: Kiểm tra lại MAC table
Switch# show mac address-table dynamic
# → Thấy MAC của PC-A và PC-B

# Bước 4: Xóa MAC table
Switch# clear mac address-table dynamic

# Bước 5: Đợi 5 phút (aging time 300s)
# → MAC entries tự biến mất nếu không có traffic

# Linux bridge equivalent:
bridge fdb show
bridge fdb show br0 | grep -v permanent
```

### Exercise 2: So sánh Hub vs Switch (Lab)

```bash
# Setup: Hub kết nối 3 PCs (hoặc GNS3 hub vs switch)

# Trên PC-C, bật packet capture:
sudo tcpdump -i eth0 -nn

# Trên PC-A, ping PC-B:
ping 192.168.1.2

# Với HUB: PC-C SẼ THẤY traffic giữa A và B (vì hub floods)
# Với SWITCH: PC-C KHÔNG THẤY gì (switch chỉ forward ra port B)

# → Đây chính là tại sao switch bảo mật hơn hub!
```

### Exercise 3: Port Security Configuration

```bash
# Cisco IOS:
Switch(config)# interface FastEthernet0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport port-security
Switch(config-if)# switchport port-security maximum 1
Switch(config-if)# switchport port-security violation shutdown
Switch(config-if)# switchport port-security mac-address sticky

# Verify:
Switch# show port-security interface Fa0/1
Switch# show port-security address

# Test: Cắm thiết bị khác vào port → port bị shutdown!
# Recovery:
Switch(config)# interface Fa0/1
Switch(config-if)# shutdown
Switch(config-if)# no shutdown
```

### Exercise 4: Linux Bridge as Software Switch

```bash
# Tạo bridge (software switch) trên Linux
sudo ip link add br0 type bridge
sudo ip link set br0 up

# Thêm interfaces vào bridge (như "cắm port")
sudo ip link set eth1 master br0
sudo ip link set eth2 master br0

# Xem MAC table của bridge
bridge fdb show br0

# Xem statistics
bridge -s link show

# Xóa bridge
sudo ip link del br0
```

### Exercise 5: AWS — VPC Subnet switching behavior

```bash
# Trong VPC, observe L2-like behavior:

# 1. Tạo 2 EC2 instances cùng subnet
# 2. Từ instance A, ping instance B (private IP)
ping 10.0.1.20

# 3. Kiểm tra ARP table
arp -a
# → Thấy MAC của instance B (AWS-assigned)

# 4. AWS không dùng broadcast (limited)
# → ARP được AWS proxy (không flood như physical switch)

# 5. Security Groups = "per-port ACL" (giống port security)
aws ec2 describe-security-groups --group-ids sg-xxx
```

---

## 10. Tóm tắt & Tài liệu đọc thêm

### Key Points — Ghi nhớ

| # | Concept | Điểm quan trọng |
|---|---------|-----------------|
| 1 | Switch vs Hub | Switch: intelligent forwarding. Hub: flood everywhere |
| 2 | MAC Learning | Learn source MAC + port → Build MAC table |
| 3 | 3 Actions | Forward (known unicast), Flood (unknown), Filter (same port) |
| 4 | Store-and-Forward | Buffer full frame, check CRC, then forward. Default enterprise |
| 5 | Cut-Through | Forward after reading Dst MAC only. Ultra-low latency |
| 6 | Collision Domain | Each switch port = separate collision domain |
| 7 | Broadcast Domain | All ports on same VLAN = 1 broadcast domain |
| 8 | L3 Switch | Can route between VLANs at wire-speed (hardware ASIC) |
| 9 | Aging Timer | Default 300s — remove inactive MAC entries |
| 10 | AWS | VPC subnet ≈ L2 broadcast domain, Nitro = virtual switch |

### Tài liệu đọc thêm

| # | Tài liệu | Link/Reference |
|---|----------|---------------|
| 1 | IEEE 802.1D-2004 — MAC Bridges | https://standards.ieee.org/ieee/802.1D/2814/ |
| 2 | Cisco — Switching Concepts | https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-1/configuration_guide/lyr2/b-171-layer2-9300-cg.html |
| 3 | Cisco White Paper — Cut-Through vs Store-and-Forward | https://www.cisco.com/c/en/us/products/collateral/switches/nexus-5020-switch/white_paper_c11-465436.html |
| 4 | AWS VPC Documentation | https://docs.aws.amazon.com/vpc/latest/userguide/how-it-works.html |
| 5 | AWS Nitro System | https://aws.amazon.com/ec2/nitro/ |
| 6 | Perlman — Interconnections (Bridges and Switches) | ISBN: 978-0201634488 |

---

**Bài tiếp theo**: [VLAN — Virtual LAN — Phân chia mạng logic trên cùng Switch vật lý](/2026-06-01-vlan-virtual-lan)
