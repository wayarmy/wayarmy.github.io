---
layout: post
title: "IS-IS Protocol - Link-State Routing cho ISP, Level 1/2, NET Address và TLVs"
date: 2026-06-01
categories: [networking]
tags: [isis, link-state, routing, isp, ospf]
---

## 1. Giới thiệu — Hệ thống bưu chính quốc gia

Hãy tưởng tượng hệ thống bưu chính Việt Nam. Để gửi thư từ Hà Nội đến Đà Nẵng:

- **Trong nội thành Hà Nội** (Level 1): Bưu tá biết từng con phố, từng ngõ hẻm — routing chi tiết trong khu vực
- **Giữa các tỉnh/thành** (Level 2): Chỉ cần biết "gửi đến Đà Nẵng" — routing giữa các vùng, không cần biết chi tiết bên trong
- **Bưu cục cấp tỉnh** (Level 1-2 router): Vừa biết chi tiết nội thành, vừa biết đường liên tỉnh — cầu nối giữa 2 thế giới

**IS-IS (Intermediate System to Intermediate System)** hoạt động y hệt hệ thống bưu chính này cho mạng Internet. Đây là routing protocol mà **hầu hết ISP lớn trên thế giới** sử dụng làm backbone — bao gồm AT&T, Level 3, NTT, Viettel, VNPT.

### Tại sao bạn cần biết IS-IS?

| Tình huống | Tại sao IS-IS quan trọng |
|---|---|
| Làm việc tại ISP | 80%+ ISP backbone dùng IS-IS |
| Thi chứng chỉ CCNP/CCIE | Bắt buộc hiểu sâu IS-IS |
| Thiết kế Data Center | Modern DC fabric (spine-leaf) thường dùng IS-IS |
| So sánh với OSPF | Hiểu trade-offs để chọn đúng protocol |
| Segment Routing | IS-IS là nền tảng cho SR-MPLS |

### IS-IS vs OSPF — Cái nhìn tổng quan

Cả hai đều là **Link-State Protocol**, đều dùng **Dijkstra SPF algorithm**, nhưng:

| Tiêu chí | OSPF | IS-IS |
|---|---|---|
| Nguồn gốc | IETF (RFC 2328) | ISO (ISO 10589) |
| Chạy trên | IP Protocol 89 | Trực tiếp trên Layer 2 (CLNP) |
| Addressing | IP-based | NSAP/NET-based |
| Area design | Area 0 backbone bắt buộc | Level 2 backbone (linh hoạt hơn) |
| Extensibility | Opaque LSA (phức tạp) | TLV (cực dễ mở rộng) |
| ISP preference | Ít phổ biến | Rất phổ biến |
| Scalability | Tốt (với area design) | Xuất sắc |

---

## 2. IS-IS là gì? — Giải thích cho người không biết IT

### Phép so sánh đời thường

Tưởng tượng một thành phố lớn với hệ thống **bảng chỉ đường thông minh**:

1. Mỗi ngã tư (router) có một **bảng thông tin** liệt kê: đường nào nối đến đâu, tốc độ bao nhiêu, có kẹt không
2. Mỗi ngã tư **chia sẻ bảng thông tin** cho TẤT CẢ ngã tư khác trong cùng khu vực
3. Khi mọi ngã tư đã có bảng thông tin của tất cả → mỗi ngã tư tự **tính đường ngắn nhất** đến mọi nơi
4. Nếu có đường bị chặn → ngã tư đó thông báo → tất cả ngã tư cùng **tính lại đường ngay**

Đó chính là **Link-State routing**. IS-IS thêm phần **phân cấp**:
- **Khu vực nhỏ** (Level 1): Chỉ biết chi tiết trong khu vực mình
- **Khu vực lớn** (Level 2): Biết cách đi giữa các khu vực
- **Ngã tư ranh giới** (L1/L2): Nối 2 thế giới lại

### Định nghĩa kỹ thuật

**IS-IS** là một **Link-State Interior Gateway Protocol** ban đầu được thiết kế cho mô hình OSI (ISO 10589), sau đó được mở rộng để hỗ trợ IP (RFC 1195 — "Integrated IS-IS"). Đặc điểm:

- **Chạy trực tiếp trên Layer 2** — không phụ thuộc IP (khác OSPF chạy trên IP)
- **NSAP addressing** — dùng Network Entity Title (NET) thay vì IP
- **2-level hierarchy** — Level 1 (intra-area) và Level 2 (inter-area)
- **TLV-based encoding** — Type-Length-Value, cực dễ mở rộng
- **SPF algorithm** — Dijkstra, giống OSPF
- **Designated IS (DIS)** — tương tự OSPF DR nhưng khác behavior

### Thuật ngữ IS-IS

| Thuật ngữ IS-IS | Tương đương OSPF | Ý nghĩa |
|---|---|---|
| Intermediate System (IS) | Router | Thiết bị routing |
| End System (ES) | Host | Thiết bị cuối |
| Circuit | Interface/Link | Kết nối mạng |
| LSP (Link State PDU) | LSA (Link State Advertisement) | Thông tin link-state |
| Level 1 | Non-backbone Area | Routing nội vùng |
| Level 2 | Area 0 (Backbone) | Routing liên vùng |
| DIS (Designated IS) | DR (Designated Router) | Đại diện trên broadcast network |
| NET (Network Entity Title) | Router ID | Định danh router |
| NSAP | IP address | Địa chỉ OSI |
| PDU | Packet | Protocol Data Unit |
| CSNP | Database Description | Mô tả LSDB |
| PSNP | LS Request/Ack | Yêu cầu/xác nhận LSP |

---

## 3. NET Address — Hệ thống địa chỉ IS-IS

### Mini-example: Số CMND/CCCD

Giống như số CCCD có cấu trúc:
```
001 - 203 - 123456
│     │     │
│     │     └── Số riêng của bạn (System ID)
│     └──────── Mã tỉnh/thành (Area ID)  
└────────────── Mã quốc gia (AFI)
```

NET address trong IS-IS cũng có cấu trúc tương tự!

### Cấu trúc NET (Network Entity Title)

```
NET Format:
49.0001.1921.6800.1001.00
│  │     │              │
│  │     │              └── SEL (Selector) - luôn = 00 cho NET
│  │     └────────────────── System ID (6 bytes) - unique per router
│  └──────────────────────── Area ID (variable length)
└─────────────────────────── AFI (Authority and Format Identifier)

Chi tiết:
┌─────────────────────────────────────────────────────────┐
│ AFI │    Area ID    │      System ID      │ SEL │
│ 49  │    0001       │  1921.6800.1001     │ 00  │
│     │               │                     │     │
│ 1B  │ 1-13 bytes    │ Always 6 bytes      │ 1B  │
└─────────────────────────────────────────────────────────┘

AFI = 49: Private addressing (tương tự 10.x.x.x trong IP)
         → Dùng cho hầu hết deployments
Area ID: Xác định area router thuộc về
System ID: Unique trong toàn domain (thường derived từ IP)
SEL = 00: Đây là NET (Network Entity Title), không phải NSAP
         (SEL ≠ 00 → NSAP address of upper layer)
```

### Cách tạo System ID từ IP address

```
Phương pháp phổ biến: Chuyển IP Loopback thành System ID

Router IP: 192.168.1.1
Bước 1: Viết đầy đủ → 192.168.001.001
Bước 2: Nhóm theo 4 chữ số → 1921.6800.1001
→ System ID = 1921.6800.1001
→ NET = 49.0001.1921.6800.1001.00

Ví dụ khác:
Router IP: 10.0.0.1
→ 010.000.000.001
→ 0100.0000.0001
→ NET = 49.0001.0100.0000.0001.00

Router IP: 172.16.255.254
→ 172.016.255.254
→ 1720.1625.5254
→ NET = 49.0002.1720.1625.5254.00 (area 0002)
```

### Quy tắc Area trong IS-IS

```
OSPF area vs IS-IS area — KHÁC BIỆT CƠ BẢN:

OSPF: Area gán cho LINK (interface)
┌───────┐         ┌───────┐
│  R1   │─Area 0──│  R2   │─── Area 1 ───│ R3 │
│       │         │ ABR   │              │    │
└───────┘         └───────┘              └────┘
R2 có interfaces thuộc NHIỀU areas

IS-IS: Area gán cho ROUTER (toàn bộ router thuộc 1 area)
┌───────────────────┐   ┌───────────────────────────┐
│     Area 49.0001  │   │       Area 49.0002        │
│  ┌──┐    ┌──┐   │   │   ┌──┐    ┌──┐   ┌──┐   │
│  │R1│────│R2│───│───│───│R3│────│R4│───│R5│   │
│  │L1│    │L1/2│ │   │   │L1/2│  │L1│   │L1│   │
│  └──┘    └──┘   │   │   └──┘    └──┘   └──┘   │
└───────────────────┘   └───────────────────────────┘

Link giữa R2 và R3 là Level 2 link (backbone)
R2 và R3 là L1/L2 routers (border)
```

### Trong thực tế

**Quy hoạch NET cho ISP:**
```
ISP XYZ — AS 65000
AFI: 49 (private)

Area Planning:
- Core/Backbone: Area 49.0000
- Region North: Area 49.0001
- Region Central: Area 49.0002
- Region South: Area 49.0003

NET Assignment:
- Core Router 1 (192.168.0.1): 49.0000.1921.6800.0001.00
- Core Router 2 (192.168.0.2): 49.0000.1921.6800.0002.00
- North PE1 (10.1.0.1):       49.0001.0100.0100.0001.00
- South PE1 (10.3.0.1):       49.0003.0100.0300.0001.00
```

### Trong AWS

AWS không sử dụng IS-IS trực tiếp trong managed services. Tuy nhiên:
- AWS Transit Gateway routing engine dựa trên concepts tương tự
- Khi kết nối On-Premises IS-IS network vào AWS:
  - IS-IS routes được redistribute vào BGP tại border router
  - BGP advertise prefixes qua Direct Connect/VPN vào AWS

---

## 4. Level 1 và Level 2 — Hệ thống phân cấp

### Mini-example: Tổ chức công ty

```
Level 2 (Backbone) = Ban Giám đốc
- Chỉ biết "Phòng Kỹ thuật có 50 người, Phòng Kinh doanh có 30 người"
- Không biết chi tiết ai ngồi ở bàn nào trong mỗi phòng
- Kết nối giữa các phòng ban

Level 1 (Intra-area) = Trong mỗi phòng ban
- Biết chi tiết: ai ngồi đâu, máy in ở góc nào
- Không biết (và không cần biết) layout phòng khác
- Khi cần gì ở phòng khác → hỏi trưởng phòng (L1/L2 router)

Level 1-2 Router = Trưởng phòng
- Biết chi tiết trong phòng mình (Level 1 database)
- Biết đường đến phòng khác (Level 2 database)
- Cầu nối thông tin giữa 2 cấp
```

### Các loại Router trong IS-IS

```
┌─────────────────────────────────────────────────────────┐
│ Level 1 Only Router:                                     │
│ - Chỉ có L1 LSDB                                       │
│ - Chỉ biết routes trong area                            │
│ - Default route đến L1/L2 router gần nhất               │
│ - Dùng cho: Access/Distribution layer                    │
│                                                          │
│ Level 2 Only Router:                                     │
│ - Chỉ có L2 LSDB                                       │
│ - Chỉ biết inter-area routes                            │
│ - Không biết chi tiết trong bất kỳ area nào             │
│ - Dùng cho: Core backbone (transit only)                 │
│                                                          │
│ Level 1-2 Router (mặc định):                            │
│ - Có CẢ L1 LSDB VÀ L2 LSDB                            │
│ - Biết chi tiết area mình + đường liên area             │
│ - Set ATT bit (Attached) → L1 router biết có exit       │
│ - Dùng cho: Border router giữa areas                    │
│                                                          │
│ ⚠️ Mặc định Cisco: Router là L1/L2                      │
│ ⚠️ Nên explicit configure level cho mỗi router!         │
└─────────────────────────────────────────────────────────┘
```

### ATT (Attached) Bit — Default Route cho Level 1

```
Khi L1/L2 router kết nối được với L2 backbone:
→ Set ATT bit = 1 trong L1 LSP
→ L1 routers thấy ATT bit → tự tạo default route (0.0.0.0/0) hướng về L1/L2 router

Vấn đề:
- Nếu có 2 L1/L2 routers, L1 router chọn gần nhất (SPF metric)
- Suboptimal routing có thể xảy ra (traffic đi lòng vòng)
- Giải pháp: Route leaking (inject L2 routes vào L1)

Route Leaking:
router isis
 redistribute isis ip level-2 into level-1 route-map L2_TO_L1
 ! Cho phép L1 routers biết specific L2 routes
 ! Tránh suboptimal routing qua default route
```

### Adjacency Types

```
Điều kiện hình thành adjacency:

L1 Adjacency:
- Cả 2 router cùng Area ID
- Cả 2 có khả năng L1 (is-type level-1 hoặc level-1-2)

L2 Adjacency:
- KHÔNG cần cùng Area ID
- Cả 2 có khả năng L2 (is-type level-2-only hoặc level-1-2)

L1/L2 Adjacency:
- Cùng Area ID + cả 2 có khả năng L1 VÀ L2
- Hình thành 2 adjacency (1 cho L1, 1 cho L2)

Bảng so sánh:
┌──────────┬───────────┬──────────────────────────┐
│ Router A │ Router B  │ Adjacency formed         │
├──────────┼───────────┼──────────────────────────┤
│ L1       │ L1        │ L1 (nếu cùng area)      │
│ L1       │ L2        │ NONE! ❌                  │
│ L1       │ L1/L2     │ L1 (nếu cùng area)      │
│ L2       │ L2        │ L2                       │
│ L2       │ L1/L2     │ L2                       │
│ L1/L2    │ L1/L2     │ L1 + L2 (cùng area)     │
│ L1/L2    │ L1/L2     │ L2 only (khác area)     │
└──────────┴───────────┴──────────────────────────┘
```

### Trong thực tế

**ISP IS-IS Design:**
```
                    ┌─────────┐
                    │ Core P  │ Level 2 Only
                    │ Router  │
                    └────┬────┘
                         │
              ┌──────────┼──────────┐
              │          │          │
        ┌─────┴───┐ ┌───┴─────┐ ┌─┴───────┐
        │  PE-N1  │ │  PE-C1  │ │  PE-S1  │ Level 1-2
        │ North   │ │ Central │ │ South   │ (Border)
        └────┬────┘ └────┬────┘ └────┬────┘
             │           │           │
        ┌────┴────┐ ┌────┴────┐ ┌────┴────┐
        │ CPE/CE  │ │ CPE/CE  │ │ CPE/CE  │ Level 1
        │ Access  │ │ Access  │ │ Access  │
        └─────────┘ └─────────┘ └─────────┘
        
  Area 49.0001    Area 49.0002    Area 49.0003

Design Principles:
1. Core = L2 only (nhỏ gọn, fast convergence)
2. PE = L1/L2 (bridge internal ↔ backbone)
3. Access = L1 only (đơn giản, default route lên PE)
4. Mỗi region = 1 area (dễ quản lý)
```

### Trong AWS

```
Mapping IS-IS Levels to AWS:
- Level 2 backbone ≈ Transit Gateway (hub connecting regions)
- Level 1 area ≈ Individual VPCs
- L1/L2 router ≈ Transit Gateway Attachment

IS-IS redistribution at border:
On-Premises PE Router:
  router isis CORE
   net 49.0001.1921.6800.0001.00
   is-type level-1-2
   
  router bgp 65000
   address-family ipv4
    redistribute isis CORE level-2 route-map ISIS_TO_BGP
    ! Chỉ redistribute L2 routes vào BGP → AWS
```

---

## 5. TLV (Type-Length-Value) — Kiến trúc mở rộng của IS-IS

### Mini-example: Form đăng ký linh hoạt

Tưởng tượng một **form đăng ký online** mà bạn có thể thêm bất kỳ trường nào:

```
Trường 1: [Tên] [Độ dài: 20 ký tự] [Giá trị: Nguyễn Văn A]
Trường 2: [Tuổi] [Độ dài: 2 chữ số] [Giá trị: 30]
Trường 3: [Địa chỉ] [Độ dài: 50 ký tự] [Giá trị: 123 Nguyễn Trãi...]
Trường 100: [IPv6 address] [Độ dài: 16 bytes] [Giá trị: 2001:db8:...]
```

Bạn muốn thêm trường mới? **Chỉ cần define Type number mới!** Không cần thay đổi form cũ. Các phần mềm cũ không hiểu Type 100 → bỏ qua nó (vì biết Length → skip đúng số bytes).

Đây chính là sức mạnh của **TLV encoding** — IS-IS có thể hỗ trợ **bất kỳ tính năng mới nào** chỉ bằng cách thêm TLV mới!

### Cấu trúc TLV

```
┌──────────────────────────────────────────┐
│ Type (1 byte) │ Length (1 byte) │ Value  │
│   0-255       │  0-255 bytes    │ (data) │
└──────────────────────────────────────────┘

Ví dụ: IP Reachability TLV (Type 128)
┌──────┬────────┬──────────────────────────┐
│ T=128│ L=12   │ Default Metric: 10       │
│      │        │ Delay Metric: 0          │
│      │        │ Expense Metric: 0        │
│      │        │ Error Metric: 0          │
│      │        │ IP Address: 192.168.1.0  │
│      │        │ Subnet Mask: 255.255.255.0│
└──────┴────────┴──────────────────────────┘
```

### Các TLV quan trọng

```
┌────────┬──────────────────────────┬─────────────────────────┐
│ Type # │ Tên                      │ Mục đích                 │
├────────┼──────────────────────────┼─────────────────────────┤
│   1    │ Area Addresses           │ Area IDs của router     │
│   2    │ IS Neighbors (old)       │ Neighbor IS, narrow metric│
│   6    │ IS Neighbors (Hello)     │ Neighbors trên LAN      │
│   8    │ Padding                  │ Pad Hello to MTU size   │
│   9    │ LSP Entries              │ LSP info trong CSNP/PSNP│
│  10    │ Authentication           │ Password authentication │
│  22    │ Extended IS Reachability  │ Wide metric neighbors   │
│ 128    │ IP Internal Reachability  │ Internal IP routes (old)│
│ 130    │ IP External Reachability  │ External routes (old)   │
│ 132    │ IP Interface Address     │ IP của interface        │
│ 135    │ Extended IP Reachability  │ Wide metric IP routes   │
│ 137    │ Dynamic Hostname         │ Router hostname         │
│ 232    │ IPv6 Interface Address   │ IPv6 của interface      │
│ 236    │ IPv6 Reachability        │ IPv6 routes             │
│ 242    │ Router Capability        │ SR/MPLS capabilities    │
└────────┴──────────────────────────┴─────────────────────────┘
```

### Narrow Metric vs Wide Metric

```
Narrow Metric (Original IS-IS):
- Interface metric: 6 bits → 0-63
- Path metric: 10 bits → 0-1023
- Quá nhỏ cho modern networks!
- TLV 2 (IS Neighbors) + TLV 128/130 (IP Reachability)

Wide Metric (Extended):
- Interface metric: 24 bits → 0-16,777,215
- Path metric: 32 bits → 0-4,294,967,295
- Đủ cho mọi network design
- TLV 22 (Extended IS Reach) + TLV 135 (Extended IP Reach)

⚠️ QUAN TRỌNG: Phải dùng Wide Metric cho:
- Traffic Engineering (TE)
- Segment Routing (SR)
- Modern network designs
- Bất kỳ metric > 63

Cấu hình:
router isis
 metric-style wide        ! Chỉ wide metric
 metric-style transition  ! Gửi cả narrow và wide (migration)
```

### Tại sao TLV tốt hơn OSPF LSA?

```
OSPF mở rộng: Cần define LSA Type mới
→ LSA Type 1, 2, 3, 4, 5, 7, 8, 9, 10, 11...
→ Mỗi LSA type có format riêng, parser riêng
→ Router cũ KHÔNG XỬ LÝ ĐƯỢC LSA type mới
→ Opaque LSA (Type 9/10/11) là workaround phức tạp

IS-IS mở rộng: Thêm TLV mới
→ Router cũ: Đọc Type → không biết → đọc Length → skip Value
→ KHÔNG bị crash, KHÔNG reject LSP
→ Backward compatible BY DESIGN
→ Thêm IPv6? TLV 236. Thêm SR? TLV 242. Done!

Ví dụ thực tế:
- IS-IS hỗ trợ IPv6: Thêm TLV 232 + 236 (1996)
- OSPF hỗ trợ IPv6: Viết lại protocol → OSPFv3 (2008)
- IS-IS thêm Segment Routing: TLV 242 sub-TLVs
- OSPF thêm SR: Opaque LSA + extended LSA types
```

### Sub-TLV — TLV lồng nhau

```
Extended IS Reachability (TLV 22):
┌────────────────────────────────────────────────┐
│ Type: 22 │ Length: variable │                   │
│                                                │
│ Neighbor System ID: 1921.6800.0002             │
│ Default Metric: 10                             │
│                                                │
│ Sub-TLVs:                                      │
│  ├── Type 6: IPv4 Interface Address            │
│  │    Value: 10.0.12.1                         │
│  ├── Type 8: IPv4 Neighbor Address             │
│  │    Value: 10.0.12.2                         │
│  ├── Type 9: Max Link Bandwidth               │
│  │    Value: 1000000 Kbps                      │
│  ├── Type 11: Unreserved Bandwidth             │
│  │    Value: [priority 0-7 values]             │
│  └── Type 12: TE Default Metric               │
│       Value: 100                               │
└────────────────────────────────────────────────┘

Sub-TLVs cho phép thêm information cho từng neighbor
→ Traffic Engineering (TE) information
→ Segment Routing labels
→ Flex-Algo information
```

### Trong thực tế

**Debug TLV content:**
```cisco
show isis database detail
! Hiển thị tất cả LSPs với TLV content

show isis database verbose
! Chi tiết nhất, bao gồm sub-TLVs

! Ví dụ output:
IS-IS Level-2 Link State Database:
LSPID                 LSP Seq Num  LSP Checksum  LSP Holdtime  ATT/P/OL
R1.00-00            * 0x00000015   0xB42E        1199          0/0/0
  Area Address: 49.0001                          ← TLV 1
  NLPID: 0xCC                                    ← TLV 129 (IP)
  Hostname: R1                                   ← TLV 137
  IP Address: 192.168.0.1                        ← TLV 132
  Metric: 10    IS-Extended R2.00                ← TLV 22
  Metric: 10    IP-Extended 192.168.1.0/24       ← TLV 135
  Metric: 0     IP-Extended 192.168.0.1/32       ← TLV 135
```

### Trong AWS

```
TLV extensibility tương đương trong AWS:
- AWS resource Tags ≈ TLV concept
  Key = Type, Value = Value (Length implicit)
- Route Table routes: Destination = "Type", Target = "Value"
- Transit Gateway route attachments carry metadata (TLV-like)

Khi IS-IS router advertise AWS-learned routes:
router isis CORE
 ! Routes từ AWS qua BGP → redistribute vào IS-IS
 redistribute bgp 65000 route-map BGP_TO_ISIS
 ! IS-IS sẽ encode routes trong TLV 135 (Extended IP Reach)
 ! với metric phù hợp
```

---

## 6. IS-IS PDU Types — Giao tiếp giữa các Router

### Mini-example: Hệ thống thông báo trường học

Trong trường học:
- **IIH (Hello)** = Điểm danh hàng ngày — "Tôi có mặt, tôi ở lớp 10A"
- **LSP** = Bảng tin trường — "Đây là toàn bộ thông tin mới nhất của tôi"
- **CSNP** = Danh sách tất cả bảng tin — "Tôi có bảng tin số 1, 2, 3... bạn thiếu cái nào?"
- **PSNP** = Yêu cầu bảng tin cụ thể — "Cho tôi xin bảng tin số 5"

### Các loại PDU

```
┌─────────────────────────────────────────────────────────────┐
│ PDU Type           │ Function            │ Analogy          │
├────────────────────┼─────────────────────┼──────────────────┤
│ IIH (IS-IS Hello) │ Neighbor discovery  │ "Hello, I'm here"│
│  - L1 LAN IIH     │   + maintenance     │                  │
│  - L2 LAN IIH     │                     │                  │
│  - P2P IIH        │                     │                  │
├────────────────────┼─────────────────────┼──────────────────┤
│ LSP (Link State   │ Carry link-state    │ "Here's my info" │
│   PDU)             │   information       │                  │
│  - L1 LSP         │                     │                  │
│  - L2 LSP         │                     │                  │
├────────────────────┼─────────────────────┼──────────────────┤
│ CSNP (Complete    │ Database summary    │ "I have these    │
│  Sequence Number) │   for sync          │  LSPs"           │
│  - L1 CSNP        │                     │                  │
│  - L2 CSNP        │                     │                  │
├────────────────────┼─────────────────────┼──────────────────┤
│ PSNP (Partial     │ Request specific    │ "Send me LSP X"  │
│  Sequence Number) │   LSPs / ACK        │  or "Got LSP X"  │
│  - L1 PSNP        │                     │                  │
│  - L2 PSNP        │                     │                  │
└─────────────────────────────────────────────────────────────┘
```

### IIH (IS-IS Hello) — Chi tiết

```
IIH Packet chứa:
- Circuit Type: L1, L2, hoặc L1/L2
- Source ID: System ID của sender
- Holding Time: Bao lâu neighbor giữ adjacency (mặc định 30s)
- PDU Length
- Priority: Cho DIS election (0-127, mặc định 64)
- LAN ID: System ID của DIS + pseudonode circuit ID
- TLVs:
  - Area Addresses (TLV 1)
  - IS Neighbors (TLV 6) — trên LAN: MAC addresses đã thấy
  - IP Interface Address (TLV 132)
  - Protocols Supported (TLV 129)
  - Padding (TLV 8) — pad to MTU size

Timers:
- Hello interval: 10 seconds (mặc định)
- Hold time: 30 seconds (= 3 × Hello)
- DIS Hello: 3.3 seconds (1/3 normal — detect DIS failure faster)

Multicast addresses (trên Ethernet):
- L1 IIH: 0180.c200.0014
- L2 IIH: 0180.c200.0015
- All IS: 0900.2b00.0005
```

### LSP (Link State PDU) — Chi tiết

```
LSP Header:
- PDU Length
- Remaining Lifetime: Countdown timer (max 1200s = 20 min)
- LSP ID: SystemID.Pseudonode-Fragment (e.g., 1921.6800.0001.00-00)
- Sequence Number: Tăng mỗi khi LSP thay đổi (32-bit)
- Checksum: Fletcher checksum
- P (Partition Repair): Hỗ trợ partition repair
- ATT (Attached): L1/L2 router có kết nối L2 backbone
- OL (Overload): Router quá tải, không transit traffic qua

LSP ID Format:
┌────────────────────┬───────────┬──────────┐
│    System ID       │Pseudonode │ Fragment │
│ 1921.6800.0001     │    00     │   00     │
└────────────────────┴───────────┴──────────┘
- Pseudonode 00 = LSP của router chính
- Pseudonode ≠ 00 = Pseudonode LSP (DIS tạo cho broadcast network)
- Fragment: Khi LSP > MTU, chia thành fragments (00, 01, 02...)

LSP Flooding:
1. Router tạo/update LSP → tăng Sequence Number
2. Flood LSP đến tất cả neighbors (trừ nguồn)
3. Neighbor nhận → so sánh Seq Num:
   - Mới hơn → Install + Flood tiếp
   - Cũ hơn → Gửi LSP mới hơn ngược lại
   - Bằng → Bỏ qua (đã có)
4. LSP expire sau 1200s nếu không refresh

Khác OSPF: IS-IS LSP có Remaining Lifetime (countdown)
           OSPF LSA có MaxAge (3600s) nhưng refresh mỗi 30 min
```

### CSNP và PSNP — Database Synchronization

```
CSNP (Complete Sequence Number PDU):
- Liệt kê TẤT CẢ LSPs trong database
- Gửi periodic trên broadcast networks (mỗi 10s bởi DIS)
- Gửi khi adjacency mới established (P2P)
- Tương tự OSPF Database Description

PSNP (Partial Sequence Number PDU):
- Yêu cầu LSPs cụ thể (thiếu so với CSNP)
- ACK cho LSPs nhận được (trên P2P links)
- Tương tự OSPF LS Request + LS Acknowledgment

Database Sync Flow (Broadcast network):
┌──────┐                ┌──────┐
│  R1  │                │ DIS  │
└──┬───┘                └──┬───┘
   │                       │
   │←── CSNP (all LSPs) ──│  DIS gửi periodic CSNP
   │                       │
   │  So sánh: R1 thiếu   │
   │  LSP 1921.6800.0002   │
   │                       │
   │── PSNP (request) ────→│  R1 yêu cầu LSP thiếu
   │                       │
   │←── LSP (requested) ──│  DIS gửi LSP
   │                       │
   [Database synchronized!]

Database Sync Flow (P2P link):
┌──────┐                ┌──────┐
│  R1  │                │  R2  │
└──┬───┘                └──┬───┘
   │                       │
   │←── CSNP (once) ──────│  Upon adjacency UP
   │                       │
   │── PSNP (request) ────→│  R1 requests missing LSPs
   │                       │
   │←── LSP ──────────────│  R2 sends requested LSPs
   │                       │
   │── PSNP (ack) ────────→│  R1 acknowledges (P2P only)
```

### Trong thực tế

**DIS (Designated Intermediate System) — Khác DR trong OSPF:**

```
OSPF DR vs IS-IS DIS:
┌──────────────────────┬─────────────────────────────┐
│ OSPF DR              │ IS-IS DIS                    │
├──────────────────────┼─────────────────────────────┤
│ DR/BDR election      │ Chỉ có DIS (không BDR)      │
│ Stable (không thay   │ Preemptive (router mới      │
│   đổi khi DR active) │   priority cao hơn → DIS)   │
│ Adjacency chỉ với    │ ALL routers full adjacency   │
│   DR/BDR             │   với nhau                   │
│ DR failure → BDR lên │ DIS failure → elect mới     │
│   (nhanh)            │   (LSP regeneration cần)    │
│ LSA generation by DR │ Pseudonode LSP by DIS       │
└──────────────────────┴─────────────────────────────┘

DIS Election:
1. Highest interface priority (0-127, default 64)
2. Highest System ID (tiebreaker)
→ Priority 0 CAN be DIS (khác OSPF: priority 0 = không participate)

DIS tạo Pseudonode LSP:
- Đại diện cho broadcast segment
- Liệt kê tất cả routers trên segment
- Routers trỏ đến Pseudonode (không trỏ đến nhau trực tiếp)
- Giảm n(n-1)/2 adjacencies → n adjacencies
```

### Trong AWS

```
PDU concepts trong AWS networking:
- CSNP ≈ Route Table full sync (propagation)
- PSNP ≈ Route Table delta update
- LSP ≈ Route propagation entries
- DIS ≈ Không có direct equivalent (AWS mesh design)

AWS Transit Gateway route propagation:
- Mỗi attachment "floods" its routes (similar to LSP flooding)
- TGW route table = LSDB equivalent
- Route evaluation = SPF calculation equivalent
```

---

## 7. IS-IS Configuration và Best Practices

### Cấu hình cơ bản

```cisco
! Cisco IOS/IOS-XE Configuration

! Bước 1: Enable IS-IS process
router isis CORE
 net 49.0001.1921.6800.0001.00
 is-type level-1-2
 metric-style wide
 log-adjacency-changes
 
! Bước 2: Enable trên interfaces
interface GigabitEthernet0/0
 ip router isis CORE
 isis circuit-type level-2-only
 isis metric 100

interface Loopback0
 ip address 192.168.0.1 255.255.255.255
 ip router isis CORE
 isis passive-interface
```

```
! Juniper JunOS Configuration

protocols {
    isis {
        interface ge-0/0/0.0 {
            level 2 metric 100;
            point-to-point;    /* Recommended for P2P links */
        }
        interface lo0.0 {
            passive;
        }
        level 1 disable;       /* L2-only core router */
        level 2 wide-metrics-only;
    }
}
```

### Best Practices cho ISP

```
1. METRIC STYLE:
   - Luôn dùng Wide Metric (metric-style wide)
   - Narrow metric chỉ cho backward compatibility

2. CIRCUIT TYPE:
   - P2P links: isis circuit-type level-2-only (trên backbone)
   - Access links: isis circuit-type level-1 (nếu L1 only area)
   - Tránh L1/L2 trên mọi link (double SPF computation)

3. POINT-TO-POINT:
   - Ethernet P2P links: isis network point-to-point
   - Tránh DIS election overhead trên P2P Ethernet
   - Faster convergence (no DIS re-election)

4. AUTHENTICATION:
   - Luôn enable authentication
   - MD5 minimum, prefer HMAC-SHA-256

5. OVERLOAD BIT:
   - Set on boot (chờ convergence trước khi transit)
   - set-overload-bit on-startup wait-for-bgp

6. BFD (Bidirectional Forwarding Detection):
   - Enable BFD cho fast failure detection (< 1s)
   - interface GigabitEthernet0/0
       isis bfd

7. LSP GENERATION:
   - Throttle LSP generation: lsp-gen-interval
   - Throttle SPF: spf-interval
   - Tránh SPF storms khi nhiều changes cùng lúc
```

### Tuning Timers

```cisco
router isis CORE
 ! LSP timers
 lsp-gen-interval 5 50 200
 ! Initial: 50ms, Increment: 50ms, Max: 200ms
 ! First LSP generated after 50ms
 ! Subsequent LSPs exponentially backed off to 200ms max
 
 ! SPF timers
 spf-interval 5 50 200
 ! Same logic: initial 50ms, max 200ms
 
 ! LSP lifetime and refresh
 max-lsp-lifetime 65535    ! Maximum (18 hours)
 lsp-refresh-interval 65000  ! Refresh before expire
 
 ! Fast convergence
 fast-flood 10    ! Flood first 10 LSPs before SPF
```

### Trong thực tế

**Large-Scale IS-IS Deployment (ISP):**
```
Viettel-like ISP Design:
- 3 Core Routers (P): L2-only, full mesh, IS-IS + LDP/SR
- 10 PE Routers: L1/L2, kết nối customer + core
- 100+ CE/Access: L1-only hoặc static

IS-IS roles:
- Carry loopback addresses cho BGP next-hops
- Provide MPLS LDP/Segment Routing underlay
- Fast convergence cho VPN traffic

Key metrics:
- Core links: metric 10 (high bandwidth, low latency)
- PE-Core: metric 100
- PE-Access: metric 1000
- Metric phản ánh link quality + bandwidth

Convergence targets:
- Link failure detection: < 50ms (BFD)
- SPF calculation: < 200ms
- FIB update: < 500ms
- Total convergence: < 1 second
```

### Trong AWS

```
IS-IS không chạy trực tiếp trong AWS, nhưng khi design hybrid:

On-Premises IS-IS Core ←→ AWS Integration:
1. IS-IS routes → redistribute vào BGP
2. BGP advertise qua Direct Connect (primary)
3. BGP advertise qua VPN (backup)

Cisco IOS-XR (PE router):
router isis CORE
 address-family ipv4 unicast
  redistribute bgp 65000 route-policy AWS_ROUTES
  ! Nhận AWS routes vào IS-IS domain

router bgp 65000
 address-family ipv4 unicast
  redistribute isis CORE route-policy ISIS_TO_AWS
  ! Gửi IS-IS routes ra AWS

Metric mapping:
- IS-IS metric → BGP MED (for path preference)
- BGP AS-PATH length → IS-IS external metric
```

---

## 8. Tình huống thực tế

### Tình huống 1: ISP Backbone — IS-IS + MPLS/SR

```
ISP XYZ triển khai IS-IS cho backbone:

Requirements:
- 50 routers, 3 regions
- MPLS VPN service
- Sub-second convergence
- Segment Routing migration

Design:
┌─────────────────────────────────────────────────────┐
│                 Level 2 Backbone                      │
│                                                       │
│   P1 ══════ P2 ══════ P3 ══════ P4                  │
│   │  \    / │  \    / │  \    / │                    │
│   │   \  /  │   \  /  │   \  /  │                    │
│   PE1  PE2  PE3  PE4  PE5  PE6  PE7                  │
│                                                       │
│ Area 49.0001    Area 49.0002    Area 49.0003         │
└─────────────────────────────────────────────────────┘

Configuration (Core Router P1):
router isis CORE
 net 49.0000.0001.0001.0001.00
 is-type level-2-only
 metric-style wide
 set-overload-bit on-startup wait-for-bgp
 address-family ipv4 unicast
  segment-routing mpls sr-prefer
  segment-routing prefix-sid-map advertise-local
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
   prefix-sid index 1    ! Segment Routing SID
 !
 interface GigabitEthernet0/0/0
  point-to-point
  bfd fast-detect ipv4
  address-family ipv4 unicast
   metric 10

Kết quả:
- IS-IS carry loopback addresses (BGP next-hops)
- MPLS labels distributed via IS-IS (SR) hoặc LDP
- BGP VPNv4 over MPLS for customer traffic
- Convergence < 1s (BFD + tuned SPF)
```

### Tình huống 2: Data Center Fabric — IS-IS Spine-Leaf

```
Modern DC dùng IS-IS (thay vì OSPF) vì:
- Flat L2 design → không cần area phức tạp
- TLV extensibility cho future features
- Better scalability với large ECMP

Design:
        Spine1    Spine2    Spine3    Spine4
          │  \   / │ \   / │ \   / │
          │   \ /  │  \ /  │  \ /  │
          │    X   │   X   │   X   │
          │   / \  │  / \  │  / \  │
          │  /   \ │ /   \ │ /   \ │
        Leaf1   Leaf2   Leaf3   Leaf4

All L2-only (flat domain):
router isis DC
 net 49.0001.0000.0000.0001.00
 is-type level-2-only
 metric-style wide
 address-family ipv4 unicast
  maximum-paths 16    ! ECMP across all spines

interface (all fabric links)
 isis network point-to-point
 isis metric 10

Leaf advertise:
- Server subnets (connected)
- Anycast addresses (VTEP, service IPs)
```

### Tình huống 3: Migration OSPF → IS-IS

```
Tại sao migrate?
- OSPF area design quá phức tạp
- Cần Segment Routing (IS-IS SR tốt hơn)
- Scalability concerns

Strategy: Ships-in-the-Night
1. Enable IS-IS trên tất cả routers (parallel với OSPF)
2. IS-IS với higher AD (115) → OSPF (110) vẫn preferred
3. Verify IS-IS topology hoàn chỉnh
4. Lower IS-IS AD hoặc raise OSPF AD
5. Traffic chuyển sang IS-IS
6. Remove OSPF

Cisco commands:
! Phase 1: Enable IS-IS
router isis MIGRATION
 net 49.0001.xxxx.xxxx.xxxx.00
 is-type level-2-only
 metric-style wide

interface GigabitEthernet0/0
 ip router isis MIGRATION    ! Add IS-IS
 ! ip ospf 1 area 0         ! Keep OSPF

! Phase 2: Verify
show isis neighbors
show isis database
show isis route

! Phase 3: Prefer IS-IS
router isis MIGRATION
 distance 105    ! Lower than OSPF default 110
 
! Phase 4: Remove OSPF (after verification)
no router ospf 1
```

### Tình huống 4: IS-IS kết hợp AWS Direct Connect

```
On-Premises:
- Core: IS-IS Level 2 backbone
- 50 sites connected via MPLS

AWS Integration:
- 2x Direct Connect (primary + backup)
- Transit Gateway cho multi-VPC

Architecture:
┌──────────────────────────┐      ┌────────────────────┐
│    On-Premises           │      │       AWS          │
│                          │      │                    │
│  [IS-IS Core] → [PE] ──DX1──── [DX GW] ── [TGW]  │
│                  [PE] ──DX2──── [DX GW]    │  │    │
│                          │      │        VPC1 VPC2  │
│  redistribute isis→bgp   │      │                    │
│  redistribute bgp→isis   │      │  BGP AS 64512     │
└──────────────────────────┘      └────────────────────┘

PE Router config:
router isis CORE
 redistribute bgp 65000 metric 1000 route-map AWS_INTO_ISIS

router bgp 65000
 neighbor 169.254.10.1 remote-as 64512
 address-family ipv4
  redistribute isis CORE route-map ISIS_TO_AWS
  neighbor 169.254.10.1 activate
  neighbor 169.254.10.1 route-map SET_MED out

route-map SET_MED permit 10
 set metric 100    ! MED for AWS to prefer this DX

route-map AWS_INTO_ISIS permit 10
 match ip address prefix-list AWS_PREFIXES
 set isis-metric 1000
 set tag 65000    ! Tag to prevent loops
```

---

## 9. Bài tập thực hành

### Bài tập 1: IS-IS Basic Configuration

```cisco
! Topology: R1 ──── R2 ──── R3
! All in area 49.0001, Level 1-2

! R1 (Loopback: 1.1.1.1/32):
router isis LAB
 net 49.0001.0010.0100.1001.00
 is-type level-1-2
 metric-style wide
 log-adjacency-changes

interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip router isis LAB

interface GigabitEthernet0/0
 ip address 10.12.0.1 255.255.255.0
 ip router isis LAB
 isis network point-to-point

! Verification:
show isis neighbors
show isis database
show isis topology
show ip route isis
show isis interface brief

! Câu hỏi:
! 1. Có bao nhiêu LSPs trong L1 database? L2 database?
! 2. System ID của R2 là gì?
! 3. Metric đến 3.3.3.3/32 là bao nhiêu?
```

### Bài tập 2: Multi-Level Design

```cisco
! Topology:
! R1 (L1) ── R2 (L1/L2) ── R3 (L2) ── R4 (L1/L2) ── R5 (L1)
! Area 49.0001                          Area 49.0002

! R1:
router isis LAB
 net 49.0001.0010.0100.1001.00
 is-type level-1

! R2 (Border):
router isis LAB
 net 49.0001.0010.0100.2002.00
 is-type level-1-2

! R3 (Core):
router isis LAB
 net 49.0000.0010.0100.3003.00
 is-type level-2-only

! Tasks:
! 1. Verify R1 has default route pointing to R2 (ATT bit)
show ip route isis
! Expected: i*L1 0.0.0.0/0 [115/...] via R2

! 2. Check ATT bit on R2's L1 LSP
show isis database detail R2.00-00

! 3. Can R1 ping R5's loopback? Via which path?
traceroute 5.5.5.5 source 1.1.1.1

! 4. Configure route leaking for R5's loopback
! On R4:
router isis LAB
 address-family ipv4
  redistribute isis ip level-2 into level-1 route-map LEAK

route-map LEAK permit 10
 match ip address prefix-list R5_LO
ip prefix-list R5_LO permit 5.5.5.5/32
```

### Bài tập 3: IS-IS Metric Tuning

```cisco
! Default metric = 10 cho mọi interface
! Cần traffic ưu tiên path qua R2 (faster link)

! Topology:
!        R2 (1Gbps links)
!       /  \
! R1──/    \──R4
!       \  /
!        R3 (100Mbps links)

! Cấu hình metric:
! R1-R2, R2-R4: metric 10 (prefer)
! R1-R3, R3-R4: metric 100 (backup)

interface GigabitEthernet0/0 (to R2)
 isis metric 10

interface GigabitEthernet0/1 (to R3)
 isis metric 100

! Verify:
show isis topology    ! SPF tree
show ip route isis    ! Best paths with metrics

! Advanced: Reference bandwidth (like OSPF)
! IS-IS không có auto-cost reference-bandwidth
! Phải set metric manually hoặc dùng script

! Traffic Engineering metric (different from IGP metric):
interface GigabitEthernet0/0
 isis metric 10              ! IGP metric (SPF)
 mpls traffic-eng metric 5  ! TE metric (CSPF)
```

### Bài tập 4: Authentication

```cisco
! Configure HMAC-MD5 authentication

! Level 2 authentication (interface-level):
interface GigabitEthernet0/0
 isis authentication mode md5 level-2
 isis authentication key-chain ISIS_KEY level-2

key chain ISIS_KEY
 key 1
  key-string SecretISIS123
  cryptographic-algorithm hmac-md5

! Domain-wide authentication (LSP/CSNP/PSNP):
router isis LAB
 authentication mode md5 level-2
 authentication key-chain ISIS_KEY level-2

! Verify:
show isis neighbors    ! Should remain UP
debug isis adj-packets ! Verify auth in packets

! Migration to SHA (hitless key rollover):
key chain ISIS_KEY
 key 2
  key-string NewStrongerKey456
  cryptographic-algorithm hmac-sha-256
  accept-lifetime 00:00:00 Jul 1 2026 infinite
  send-lifetime 00:00:00 Jul 15 2026 infinite
```

### Bài tập 5: Troubleshooting IS-IS

```cisco
! Scenario: R1 và R2 không form adjacency

! Systematic troubleshooting:
! Step 1: Check interface status
show isis interface GigabitEthernet0/0
! Is IS-IS enabled? Level correct? Metric shown?

! Step 2: Check Hello exchange
debug isis adj-packets
! Are Hellos being sent/received?
! Common issues:
! - Area mismatch (for L1)
! - Authentication mismatch
! - MTU mismatch (Hello padded to MTU)

! Step 3: Check area and NET
show isis    ! Display NET, area, type

! Step 4: Check protocol enabled
show ip protocols    ! IS-IS listed? Networks?

! Step 5: Interface-specific issues
show isis interface detail
! Check: passive? metric? level? network-type?

! Common issues and fixes:
! 1. Passive interface → no Hellos sent
!    Fix: no isis passive-interface
!
! 2. MTU mismatch → Hello padded, can't be received
!    Fix: Match MTU on both sides or:
!         no isis hello padding
!
! 3. Level mismatch → L1 can't peer with L2-only
!    Fix: Match circuit-type
!
! 4. Authentication → wrong key
!    Fix: Match key on both sides
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Tóm tắt kiến thức

```
┌─────────────────────────────────────────────────────────────┐
│                    IS-IS SUMMARY                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ▸ IS-IS = Link-State IGP, chạy trên Layer 2 (không IP)    │
│ ▸ Dùng SPF (Dijkstra) algorithm — giống OSPF              │
│ ▸ TLV encoding → dễ mở rộng (IPv6, SR, TE...)            │
│                                                             │
│ ▸ NET = AFI + Area ID + System ID + SEL(00)               │
│   Ví dụ: 49.0001.1921.6800.0001.00                        │
│                                                             │
│ ▸ Level 1: Intra-area routing (chi tiết local)            │
│ ▸ Level 2: Inter-area backbone (liên kết areas)           │
│ ▸ L1/L2 Router: Border, kết nối 2 levels                 │
│                                                             │
│ ▸ ATT bit: L1/L2 router set → L1 routers có default route│
│ ▸ Route leaking: Inject L2 routes vào L1 (tránh suboptimal)│
│                                                             │
│ ▸ PDU types: IIH (Hello), LSP, CSNP, PSNP                │
│ ▸ DIS: Preemptive, no BDR, all-to-all adjacency          │
│                                                             │
│ ▸ Wide metric: 24-bit interface, 32-bit path              │
│ ▸ Narrow metric: 6-bit interface (legacy, avoid!)          │
│                                                             │
│ ▸ ISP choice: 80%+ backbone dùng IS-IS (không OSPF)      │
│ ▸ Reasons: TLV extensibility, L2 independence from IP,    │
│   simpler area design, better scalability                  │
└─────────────────────────────────────────────────────────────┘
```

### IS-IS vs OSPF — Khi nào chọn cái nào?

```
Chọn IS-IS khi:
✓ ISP/Service Provider backbone
✓ Large-scale network (1000+ routers)
✓ Segment Routing deployment
✓ Multi-protocol support needed (IPv4 + IPv6 single topology)
✓ Need for future extensibility (TLV)
✓ Data Center fabric (spine-leaf)

Chọn OSPF khi:
✓ Enterprise campus (nhỏ-vừa)
✓ Team quen OSPF hơn IS-IS
✓ Multi-vendor với devices cũ (OSPF support rộng hơn)
✓ Đã có OSPF deployment ổn định
✓ Certification study (CCNA/CCNP chú trọng OSPF hơn)
```

### Tài liệu tham khảo

1. **ISO 10589:2002** — "Information technology — Intermediate System to Intermediate System intra-domain routing information exchange protocol" — Standard gốc của IS-IS

2. **RFC 1195** — "Use of OSI IS-IS for Routing in TCP/IP and Dual Environments" — Integrated IS-IS cho IP
   - https://www.rfc-editor.org/rfc/rfc1195

3. **RFC 5305** — "IS-IS Extensions for Traffic Engineering" — TE TLVs
   - https://www.rfc-editor.org/rfc/rfc5305

4. **RFC 5308** — "Routing IPv6 with IS-IS" — IPv6 support
   - https://www.rfc-editor.org/rfc/rfc5308

5. **RFC 8667** — "IS-IS Extensions for Segment Routing" — SR-MPLS
   - https://www.rfc-editor.org/rfc/rfc8667

6. **Cisco Documentation** — "IS-IS TLVs"
   - https://www.cisco.com/c/en/us/support/docs/ip/integrated-intermediate-system-to-intermediate-system-is-is/5739-tlvs-5739.html

7. **Cisco Press** — "IS-IS: Deployment in IP Networks" by Russ White — Comprehensive IS-IS reference book

8. **Juniper Documentation** — "IS-IS Overview"
   - https://www.juniper.net/documentation/us/en/software/junos/is-is/topics/concept/is-is-routing-overview.html

---

*Bài viết tiếp theo: [BGP Deep Dive](/2026/07/02/bgp-deep-dive) — Path-vector protocol, AS, eBGP/iBGP, route selection*
