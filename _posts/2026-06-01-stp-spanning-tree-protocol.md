---
layout: post
title: "STP — Spanning Tree Protocol — Ngăn chặn Loop trong mạng Layer 2"
subtitle: "Hiểu sâu về 802.1D, BPDU, Root Bridge Election, Port States và RSTP/MSTP"
tags: [networking, stp, switching, layer2, redundancy, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| IEEE 802.1D-2004 | MAC Bridges — Spanning Tree Algorithm |
| IEEE 802.1w | Rapid Spanning Tree Protocol (RSTP) |
| IEEE 802.1s | Multiple Spanning Tree Protocol (MSTP) |
| Perlman, R. — Interconnections, 2nd Ed. | Chapter 3: Transparent Bridges (STP inventor) |
| Cisco — STP Configuration Guide | Catalyst Switch STP Implementation |
| Tanenbaum — Computer Networks, 6th Ed. | Chapter 4: Spanning Tree Bridges |

---

## 1. Giới thiệu — Tại sao cần biết STP?

### Ví dụ đời thường: Vòng lặp vô hạn trong bưu điện

Hãy tưởng tượng có 3 bưu điện (Switch A, B, C) kết nối với nhau tạo thành tam giác. Bưu tá tại mỗi bưu điện có quy tắc: **"Nếu không biết người nhận ở đâu → gửi cho TẤT CẢ bưu điện khác"** (flooding).

Bây giờ, một lá thư broadcast ("gửi cho TẤT CẢ"):
1. Bưu điện A nhận thư → gửi bản copy cho B và C
2. Bưu điện B nhận từ A → gửi bản copy cho C
3. Bưu điện C nhận từ A → gửi bản copy cho B
4. Bưu điện B nhận từ C → gửi bản copy cho A (vòng lặp!)
5. Bưu điện C nhận từ B → gửi bản copy cho A (vòng lặp!)
6. ... VÔ HẠN! Thư multiply exponentially!

**Kết quả:** Trong vài GIÂY, hàng triệu bản copy bão hòa hệ thống → SỤNG!

Đây gọi là **broadcast storm** — và STP là "giải pháp" để ngăn nó.

### Concrete scenario

Hãy tưởng tượng bạn là network admin. Sáng thứ Hai, bạn kết nối 1 cable dự phòng giữa 2 switch cho redundancy. **30 giây sau, toàn bộ mạng công ty sụp!** 500 users mất kết nối. Điện thoại VoIP chết. Monitoring system báo đỏ rực. CPU switch lên 100%.

**Nguyên nhân:** Bridge loop! Frame broadcast lặp vô hạn giữa 2 switch qua cả 2 link.

**Giải pháp:** STP tự động detect loop, **block 1 trong 2 link** (giữ làm backup), chỉ cho traffic qua 1 đường.

### Vấn đề STP giải quyết

| Vấn đề | Hậu quả | STP giải quyết |
|---------|---------|----------------|
| Broadcast storm | Frames multiply vô hạn → bandwidth 100% | Block redundant paths |
| MAC table instability | Source MAC xuất hiện trên nhiều ports → table flapping | Single active path |
| Duplicate frames | End-host nhận multiple copies | Loop-free topology |
| Network DOWN | Switch CPU 100%, mất connectivity | Automatic convergence |

---

## 2. STP là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**STP (Spanning Tree Protocol)** là protocol tự động **tắt một số đường đi dư thừa** trong mạng switch để **không tạo vòng lặp**, nhưng vẫn giữ đường dự phòng sẵn sàng — nếu đường chính hỏng, đường dự phòng tự động bật.

### Analogy: Hệ thống đường ống nước trong thành phố

```
┌─────────────────────────────────────────────────────────────┐
│        STP = VAN KHÓA NƯỚC TỰ ĐỘNG                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Thành phố có nhiều ống nước nối giữa các bể chứa:        │
│                                                              │
│     Bể A ════════ Bể B         (Đường chính - MỞ)         │
│      ║              ║                                       │
│      ║              ║           (Đường dự phòng)            │
│      ╚════ Bể C ════╝                                       │
│           [VAN KHÓA]            ← STP block đường này!     │
│                                                              │
│  Nếu mở hết: Nước chạy vòng tròn A→B→C→A... vô hạn!     │
│  Giải pháp: Khóa 1 van → nước chỉ chảy 1 chiều           │
│                                                              │
│  Nếu đường A-B bị vỡ: Van tự động MỞ đường C              │
│  → Nước vẫn đến được mọi bể (redundancy preserved)        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### ASCII Diagram: Loop Problem và STP Solution

```
WITHOUT STP (LOOP!):                WITH STP (No loop):

  Switch A ════════ Switch B         Switch A ════════ Switch B
     ║                 ║                ║                 ║
     ║   LOOP! 🔥     ║                ║   BLOCKED ✗     ║
     ║                 ║                ║       ╳         ║
     ╚════ Switch C ═══╝                ╚════ Switch C ═══╝
                                              (port blocked)
  Broadcast frames                    
  circulate forever!                  Only 1 active path
  → Network CRASH                    → Loop-free topology
```

---

## 3. STP Operation — Cách STP hoạt động step-by-step

### 3.1 Bước 1: Bầu Root Bridge — "Ai là BOSS?"

**Root Bridge** = switch "trung tâm" mà tất cả traffic hướng về. Giống CEO trong công ty — mọi quyết định dựa trên vị trí so với CEO.

**Cách bầu:**
- Mỗi switch có **Bridge ID (BID)** = Priority (2 bytes) + MAC Address (6 bytes)
- Switch có **BID THẤP NHẤT** = Root Bridge
- Default priority = **32768** (0x8000)
- Nếu priority bằng nhau → so MAC address (thấp hơn thắng)

```
Bridge ID = Priority + System ID Extension + MAC Address
┌────────────────────┬──────────────────────────────┐
│  Priority (4 bits) │  System ID Ext   │ MAC Address│
│  + VLAN (12 bits)  │  (VLAN ID)       │ (48 bits)  │
│  = 2 bytes total   │                  │ 6 bytes    │
└────────────────────┴──────────────────────────────┘

Ví dụ:
Switch A: BID = 32768 + VLAN1 + AA:AA:AA:00:00:01
Switch B: BID = 32768 + VLAN1 + BB:BB:BB:00:00:02
Switch C: BID = 32768 + VLAN1 + CC:CC:CC:00:00:03

→ Switch A win (MAC thấp nhất) → Root Bridge!

Để force switch cụ thể làm Root:
Switch-B(config)# spanning-tree vlan 1 priority 4096
→ BID = 4096 + ... → thấp nhất → Root Bridge!
Hoặc:
Switch-B(config)# spanning-tree vlan 1 root primary   ! priority = 24576
Switch-C(config)# spanning-tree vlan 1 root secondary ! priority = 28672
```

### 3.2 Bước 2: Xác định Port Roles

Sau khi bầu Root Bridge, mỗi port trên mỗi switch được gán 1 vai trò:

| Port Role | Ý nghĩa | Trạng thái |
|-----------|---------|-------------|
| **Root Port** | Port có **best path** đến Root Bridge | Forwarding |
| **Designated Port** | Port "đại diện" cho segment đó | Forwarding |
| **Blocked Port** | Port dư thừa → bị tắt để prevent loop | Blocking |

```
     Root Bridge (Switch A)
     ┌──────────────────┐
     │DP             DP│      DP = Designated Port
     └──┬───────────┬──┘      RP = Root Port
        │           │          BP = Blocked Port
        │           │
   ┌────┴────┐ ┌───┴─────┐
   │RP    DP │ │RP    DP │
   │Switch B │ │Switch C  │
   │      DP │ │ BP       │
   └─────┬───┘ └──┬──────┘
         │         │
         │    ╳ BLOCKED ╳    ← STP block port này!
         │         │
         ╚═════════╝  (link vật lý vẫn có, nhưng STP disable)
```

### 3.3 Bước 3: Tính Path Cost — Đường nào "gần" Root nhất?

| Speed | STP Cost (802.1D original) | RSTP Cost (802.1t revised) |
|-------|---------------------------|---------------------------|
| 10 Mbps | 100 | 2,000,000 |
| 100 Mbps | 19 | 200,000 |
| 1 Gbps | 4 | 20,000 |
| 10 Gbps | 2 | 2,000 |
| 100 Gbps | - | 200 |
| 1 Tbps | - | 20 |

**Root Path Cost** = tổng cost từ switch đến Root Bridge.

```
Ví dụ:
Root (A) ──[Cost 4]──→ Switch B: Root Path Cost = 4
Root (A) ──[Cost 4]──→ Switch C: Root Path Cost = 4
Switch B ──[Cost 4]──→ Switch C: Root Path Cost = 4 + 4 = 8

Switch C có 2 paths đến Root:
- Trực tiếp: Cost = 4 (TỐT HƠN) → chọn port này = Root Port
- Qua B: Cost = 8 → port này bị block
```

### 3.4 BPDU — Bridge Protocol Data Unit

BPDU là "thông điệp" switches gửi cho nhau để exchange STP information:

```
BPDU Format:
┌──────────────┬──────────┬─────────────┬───────────────┬──────────┐
│Protocol ID   │Version   │ BPDU Type   │ Flags         │ Root ID  │
│(0x0000)      │(0x00)    │(0x00=Config)│               │ (8 bytes)│
├──────────────┴──────────┴─────────────┴───────────────┴──────────┤
│ Root Path Cost │ Bridge ID │ Port ID │ Message Age │ Max Age    │
│ (4 bytes)      │ (8 bytes) │ (2 bytes)│ (2 bytes)  │ (2 bytes)  │
├────────────────┴───────────┴──────────┴────────────┴────────────┤
│ Hello Time │ Forward Delay │                                     │
│ (2 bytes)  │ (2 bytes)     │                                     │
└────────────┴───────────────┘

Gửi đến: 01:80:C2:00:00:00 (STP multicast)
```

**BPDU chứa thông tin quan trọng:**
- Root Bridge ID (ai là Root?)
- Root Path Cost (chi phí đến Root)
- Sender Bridge ID (ai gửi BPDU này?)
- Port ID (port nào gửi?)
- Timers: Hello, Max Age, Forward Delay

### 3.5 STP Timers

| Timer | Default | Chức năng |
|-------|---------|-----------|
| **Hello Time** | 2 seconds | Root Bridge gửi BPDU mỗi 2s |
| **Max Age** | 20 seconds | Nếu không nhận BPDU trong 20s → coi link dead |
| **Forward Delay** | 15 seconds | Thời gian chờ ở mỗi trạng thái (Listening → Learning) |

**Convergence time (worst case):** Max Age + 2 × Forward Delay = 20 + 15 + 15 = **50 giây!**

Đây là lý do STP bị chê **quá chậm** cho modern networks.

---

## 4. Port States — Các trạng thái của port trong STP

### Ví dụ đời thường

Giống cửa an ninh sân bay: không cho qua ngay mà phải qua nhiều bước kiểm tra:

```
Port States (802.1D):

DISABLED → BLOCKING → LISTENING → LEARNING → FORWARDING
  (tắt)    (chặn)     (nghe)     (học)      (hoạt động)
           │                                  │
           │←────── 50 seconds total ────────→│
           │  20s Max Age  │ 15s FD │ 15s FD │
```

| State | Nhận BPDU? | Gửi BPDU? | Learn MAC? | Forward Data? |
|-------|-----------|-----------|-----------|--------------|
| Disabled | ✗ | ✗ | ✗ | ✗ |
| Blocking | ✓ | ✗ | ✗ | ✗ |
| Listening | ✓ | ✓ | ✗ | ✗ |
| Learning | ✓ | ✓ | ✓ | ✗ |
| Forwarding | ✓ | ✓ | ✓ | ✓ |

---

## 5. RSTP (Rapid STP — 802.1w) — "STP nhanh"

### Tại sao cần RSTP?

STP 802.1D convergence: **30-50 giây** — KHÔNG CHẤP NHẬN ĐƯỢC cho VoIP, video, hay bất kỳ ứng dụng real-time nào! RSTP giảm xuống **1-6 giây** (thường < 1 giây).

### 5.1 RSTP vs STP

| Feature | STP (802.1D) | RSTP (802.1w) |
|---------|-------------|---------------|
| Convergence | 30-50 seconds | 1-6 seconds |
| Port States | 5 (Disabled, Blocking, Listening, Learning, Forwarding) | 3 (Discarding, Learning, Forwarding) |
| Port Roles | Root, Designated, Blocked | Root, Designated, Alternate, Backup |
| BPDU processing | Only Root sends | ALL switches send |
| Topology change | TCN BPDU → Root → TC notification | Direct proposal/agreement |
| Backward compatible | - | Yes (fallback to STP if neighbor is STP) |

### 5.2 RSTP Port Roles

| Port Role | Mô tả | Khi active link fail... |
|-----------|--------|------------------------|
| Root Port | Best path to Root | Alternate Port replaces immediately |
| Designated Port | Best port on segment | Backup Port replaces |
| **Alternate Port** | "Backup Root Port" | → Becomes new Root Port (instant!) |
| **Backup Port** | "Backup Designated Port" | → Becomes new Designated Port |

```
RSTP Convergence:

Switch B: Root Port (Gi0/1) ←── active path to Root
          Alternate Port (Gi0/2) ←── backup, ready!

Khi Gi0/1 fail:
→ Gi0/2 (Alternate) INSTANTLY becomes Root Port
→ NO waiting 30-50 seconds!
→ Convergence < 1 second (just proposal/agreement exchange)
```

### 5.3 RSTP Proposal/Agreement Mechanism

```
Switch A (Root) ─── Switch B ─── Switch C

1. B muốn forward trên port → C:
   B → C: "Proposal" (tôi muốn forward, bạn đồng ý?)
   
2. C nhận Proposal:
   C: Block tất cả non-edge ports (sync)
   C → B: "Agreement" (đồng ý, tôi đã block xong)
   
3. B nhận Agreement:
   B: Port → Forwarding NGAY LẬP TỨC (không đợi 30s!)

Toàn bộ quá trình: < 1 giây!
```

---

## 6. STP Variants và MSTP

### 6.1 Per-VLAN STP (PVST+) — Cisco proprietary

Mỗi VLAN có **spanning tree instance riêng** → có thể load-balance traffic:

```
VLAN 10: Root = Switch A, traffic đi path A→B
VLAN 20: Root = Switch B, traffic đi path B→A

→ Cả 2 links đều active (nhưng cho VLANs khác nhau)!
→ Tốt hơn classic STP (1 link bị block hoàn toàn)
```

### 6.2 MST (Multiple Spanning Tree — 802.1s)

Thay vì 1 instance/VLAN (tốn resource), nhóm nhiều VLANs vào 1 instance:

```
MST Instance 1: VLAN 1-100    → Root = Switch A
MST Instance 2: VLAN 101-200  → Root = Switch B
MST Instance 0: IST (Internal Spanning Tree) → backbone
```

### 6.3 So sánh

| Feature | STP (802.1D) | PVST+ | RSTP (802.1w) | Rapid PVST+ | MST (802.1s) |
|---------|-------------|-------|---------------|-------------|--------------|
| Instances | 1 | Per-VLAN | 1 | Per-VLAN | Grouped |
| Convergence | 50s | 50s | <6s | <6s | <6s |
| Load balance | No | Yes | No | Yes | Yes |
| Resources | Low | High (many VLANs) | Low | High | Medium |
| Standard | IEEE | Cisco | IEEE | Cisco | IEEE |

---

## 7. STP trong AWS — Mapping sang Cloud

### 7.1 AWS và STP

**Tin tốt: AWS KHÔNG CẦN STP!** Vì:
- VPC networking do AWS quản lý hoàn toàn (software-defined)
- Không có physical loops (AWS controls topology)
- Redundancy handled bằng multi-AZ, không phải L2 redundancy

| STP Concept | AWS Equivalent |
|-------------|---------------|
| Redundancy (2 paths) | Multi-AZ deployment |
| Root Bridge | VPC Router (implicit, managed) |
| Failover | AZ failover (automatic) |
| Loop prevention | Not needed (no L2 loops in VPC) |
| Convergence | Sub-second failover (AWS managed) |
| BPDU | N/A |

### 7.2 Khi nào gặp STP với AWS?

**AWS Direct Connect + On-premises:**
```
┌─── On-premises ───────────────────────────────┐
│                                                │
│  Core Switch A ═══════ Core Switch B           │
│       │         STP          │                 │
│       │ (active)    (blocked)│                 │
│       │                      │                 │
│  ┌────┴────┐          ┌─────┴────┐           │
│  │ DX      │          │ DX       │           │
│  │ Router A│          │ Router B │           │
│  └────┬────┘          └────┬─────┘           │
└───────┼─────────────────────┼─────────────────┘
        │                     │
   Direct Connect #1     Direct Connect #2
        │                     │
┌───────┼─────────────────────┼─────────────────┐
│       │     AWS Region      │                  │
│  ┌────┴────┐          ┌────┴─────┐           │
│  │DX GW #1│          │DX GW #2  │           │
│  └────┬────┘          └────┬─────┘           │
│       └────────┬───────────┘                  │
│           VPC / Transit GW                    │
└───────────────────────────────────────────────┘

→ STP chạy on-premises để prevent loop giữa 2 DX paths
→ AWS side: dùng BGP path selection (không phải STP)
```

---

## 8. Tình huống thực tế — STP được sử dụng như thế nào?

### Scenario 1: Mạng nhỏ — Tránh loop vô tình

**Tình huống:** Nhân viên IT junior cắm 2 cáp từ switch phòng vào cùng 1 switch tầng (cho "backup"). Nếu không có STP → broadcast storm ngay lập tức!

```
TRƯỚC (LOOP!):
Switch Phòng ══cable 1══ Switch Tầng
      ║                        ║
      ╚════════ cable 2 ═══════╝  ← LOOP!

SAU (STP bảo vệ):
Switch Phòng ══cable 1══ Switch Tầng    (Forwarding)
      ║                        ║
      ╚══cable 2 [BLOCKED]═════╝        (STP blocked)

→ STP tự phát hiện loop, block 1 link
→ Nếu cable 1 đứt → cable 2 tự unblock (30-50s with STP, <1s with RSTP)
```

### Scenario 2: Enterprise — Controlled Redundancy

```
Design mẫu: Collapsed Core

            ┌─── Core/Distrib SW-A (Root Primary) ───┐
            │         Priority: 4096                   │
            │                                          │
     ┌──────┤DP                                   DP├──────┐
     │      │                                          │      │
     │      └──────────────────────────────────────────┘      │
     │                         │                              │
     │RP                       │DP                        RP  │
┌────┴─────┐              ┌────┴─────┐              ┌─────┴────┐
│Access SW1│              │Access SW2│              │Access SW3│
│(Fa0/1-48)│              │(Fa0/1-48)│              │(Fa0/1-48)│
└──────────┘              └──────────┘              └──────────┘

→ Nếu link Access SW1 → Core A fails:
  RSTP: Alternate port activates in <1 second
  PVST+: Per-VLAN failover → partial traffic rerouted
```

### Scenario 3: Data Center — No STP! (L3 Fabric)

```
Modern data center dùng Spine-Leaf (Layer 3):
→ KHÔNG CẦN STP vì không có L2 loops
→ Tất cả links active (ECMP — Equal Cost Multi-Path)
→ Hiệu quả gấp 2-3× so với STP (không block link nào)

Tuy nhiên, nếu có L2 segment (VXLAN overlay):
→ Dùng VXLAN EVPN multi-homing thay STP
→ Active-Active forwarding (không block)
```

---

## 9. Bài tập thực hành

### Exercise 1: Observe STP Operation

```bash
# Cisco switch:
Switch# show spanning-tree

# Output phân tích:
# VLAN0001
#   Spanning tree enabled protocol rstp
#   Root ID    Priority    4097
#              Address     aabb.cc00.0100
#              Cost        4
#              Port        1 (GigabitEthernet0/1)
#   Bridge ID  Priority    32769
#              Address     aabb.cc00.0200
# 
# Interface     Role  Sts  Cost     Prio.Nbr  Type
# Gi0/1         Root  FWD  4        128.1     P2p
# Gi0/2         Desg  FWD  4        128.2     P2p
# Fa0/1         Desg  FWD  19       128.3     P2p

# Xem chi tiết:
Switch# show spanning-tree detail
Switch# show spanning-tree interface gi0/1 detail
```

### Exercise 2: Change Root Bridge

```bash
# Hiện tại: Switch A là Root (MAC thấp nhất)
# Muốn: Switch B làm Root

# Method 1: Giảm priority
Switch-B(config)# spanning-tree vlan 1 priority 4096

# Method 2: Dùng macro
Switch-B(config)# spanning-tree vlan 1 root primary

# Verify:
Switch-B# show spanning-tree vlan 1 | include Root
# → "This bridge is the root"

# Trên switch khác:
Switch-A# show spanning-tree vlan 1 | include Root
# → Root ID: ... Address: [MAC của Switch B]
```

### Exercise 3: Simulate Link Failure

```bash
# 1. Xem topology hiện tại:
Switch# show spanning-tree vlan 1 brief

# 2. Shutdown root port:
Switch(config)# interface gi0/1
Switch(config-if)# shutdown

# 3. Watch convergence:
# Classic STP: ~50 seconds to converge
# RSTP: < 1 second

# 4. Check new topology:
Switch# show spanning-tree vlan 1 brief
# → Alternate port should now be Root Port

# 5. Bring back:
Switch(config-if)# no shutdown
# → Observe re-convergence
```

### Exercise 4: STP Protection Features

```bash
# BPDU Guard — shutdown port if BPDU received (for access ports)
Switch(config)# interface range fa0/1-48
Switch(config-if-range)# spanning-tree bpduguard enable
# → If someone connects a switch to access port → port SHUTS DOWN!

# Root Guard — prevent unauthorized root bridge
Switch(config)# interface gi0/1
Switch(config-if)# spanning-tree guard root
# → If better BPDU comes in → port goes to inconsistent state

# PortFast — skip Listening/Learning for end devices
Switch(config)# interface range fa0/1-48  
Switch(config-if-range)# spanning-tree portfast
# → Port goes to Forwarding immediately (30s saved!)
# ⚠️ ONLY for access ports! NEVER on trunk/switch links!

# Loop Guard — detect unidirectional link failures
Switch(config)# interface gi0/1
Switch(config-if)# spanning-tree guard loop
```

### Exercise 5: Linux Bridge STP

```bash
# Enable STP on Linux bridge
sudo ip link add br0 type bridge
sudo ip link set br0 type bridge stp_state 1

# Check STP state
cat /sys/class/net/br0/bridge/stp_state
# 1 = enabled

# View bridge info
bridge stp show br0
brctl showstp br0

# Change priority (make this bridge Root)
echo 4096 > /sys/class/net/br0/bridge/priority
# or
bridge link set dev br0 priority 4096
```

---

## 10. Tóm tắt & Tài liệu đọc thêm

### Key Points — Ghi nhớ

| # | Concept | Điểm quan trọng |
|---|---------|-----------------|
| 1 | Purpose | Prevent L2 loops by blocking redundant paths |
| 2 | Root Bridge | Lowest Bridge ID wins (Priority + MAC) |
| 3 | Port Roles | Root Port, Designated Port, Blocked Port |
| 4 | Path Cost | Lower cost = faster link. 1G = cost 4, 100M = cost 19 |
| 5 | Convergence | STP: 30-50s. RSTP: <1-6s |
| 6 | BPDU | Config BPDUs exchanged every 2s (Hello timer) |
| 7 | Port States | Blocking→Listening→Learning→Forwarding (15s each transition) |
| 8 | RSTP | Proposal/Agreement → near-instant convergence |
| 9 | Protection | BPDUGuard, RootGuard, PortFast, LoopGuard |
| 10 | AWS | No STP needed in VPC — AWS manages topology |

### Tài liệu đọc thêm

| # | Tài liệu | Link/Reference |
|---|----------|---------------|
| 1 | IEEE 802.1D-2004 | https://standards.ieee.org/ieee/802.1D/2814/ |
| 2 | IEEE 802.1w (RSTP) | Incorporated into 802.1D-2004 |
| 3 | Cisco STP Guide | https://www.cisco.com/c/en/us/tech/lan-switching/spanning-tree-protocol/index.html |
| 4 | Perlman — Interconnections | ISBN: 978-0201634488 (STP inventor's book) |
| 5 | Cisco — STP Toolkit (BPDUGuard, etc.) | https://www.cisco.com/c/en/us/support/docs/lan-switching/spanning-tree-protocol/10588-74.html |

---

**Bài tiếp theo**: [Link Aggregation — LACP — Gộp nhiều link vật lý thành 1 link logic](/2026-06-01-link-aggregation-lacp)
