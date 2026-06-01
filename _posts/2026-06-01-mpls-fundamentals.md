---
layout: post
title: "MPLS Fundamentals - Label Switching, LSP, LDP, RSVP-TE, L2/L3 VPN và ISP Backbone"
date: 2026-06-01
categories: [networking]
tags: [mpls, label-switching, lsp, ldp, rsvp-te, l2vpn, l3vpn, isp, backbone]
---

## 1. Giới thiệu — Hệ thống đường cao tốc thu phí tự động

Hãy tưởng tượng bạn lái xe trên **đường cao tốc liên tỉnh**. Có 2 cách thu phí:

**Cách cũ (IP routing truyền thống):**
1. Mỗi trạm thu phí bạn phải **dừng lại**
2. Nhân viên **kiểm tra giấy tờ**: "Anh đi đâu? Từ đâu đến?"
3. Tra bảng giá, tính phí, rồi **chỉ đường** đi tiếp
4. Đến trạm tiếp theo — lại dừng, lại kiểm tra, lại chỉ đường
5. Rất **chậm** — đặc biệt giờ cao điểm!

**Cách mới (MPLS):**
1. Khi vào cao tốc, bạn được dán **nhãn ETC** (Electronic Toll Collection) lên kính xe
2. Mỗi trạm chỉ cần **đọc nhãn** → biết ngay hướng đi → **đổi nhãn mới** cho đoạn tiếp
3. Không cần dừng, không cần kiểm tra giấy tờ
4. Xe chạy **vù vù** qua mỗi trạm
5. Khi ra cao tốc → **gỡ nhãn** → trả về đường thường

**MPLS (Multiprotocol Label Switching)** hoạt động y hệt:
- **Label (nhãn)** = Thẻ ETC dán trên packet
- **LSR (Label Switching Router)** = Trạm thu phí tự động
- **LSP (Label Switched Path)** = Tuyến đường đã định sẵn
- **LDP/RSVP-TE** = Hệ thống đăng ký và phát hành nhãn
- **PE/CE router** = Cổng vào/ra cao tốc

### Tại sao ISP cần MPLS?

| Vấn đề IP routing | MPLS giải quyết |
|---|---|
| Mỗi router tra bảng routing table → chậm | Label lookup nhanh hơn (fixed-length, O(1)) |
| Không thể "đặt trước" bandwidth | RSVP-TE reservation bandwidth per path |
| Khó cung cấp VPN cho enterprise | L3VPN/L2VPN native support |
| Không traffic engineering | TE cho phép chọn đường theo capacity |
| Convergence chậm khi link fail | Fast Reroute (FRR) < 50ms |

### MPLS trong đời thường của bạn

Khi bạn dùng Internet ở Việt Nam:
- Data từ nhà bạn → **ISP backbone** (FPT, Viettel, VNPT) → Internet
- ISP backbone **chạy MPLS** — đây là "đường cao tốc" nội bộ ISP
- Nhờ MPLS, ISP có thể bán **VPN riêng** cho doanh nghiệp
- MPLS giúp ISP đảm bảo **QoS** — gói thoại ưu tiên hơn download phim

---

## 2. MPLS là gì? — Giải thích cho người không biết IT

### Phép so sánh đời thường — Hệ thống chuyển phát nhanh

Tưởng tượng bạn gửi hàng qua **J&T Express/GHN** từ Hà Nội đến TP.HCM:

**Không có MPLS (gửi hàng thường):**
1. Bưu cục Hà Nội nhận hàng → đọc địa chỉ "123 Nguyễn Văn A, Quận 1, TP.HCM"
2. Phải **tra bản đồ** → quyết định gửi đi Đà Nẵng trước
3. Bưu cục Đà Nẵng nhận → **lại đọc địa chỉ** → quyết định gửi TP.HCM
4. Bưu cục TP.HCM nhận → **lại đọc địa chỉ** → gửi Quận 1
5. Mỗi bưu cục đều **đọc toàn bộ địa chỉ** — tốn thời gian!

**Có MPLS (chuyển phát ưu tiên):**
1. Bưu cục Hà Nội nhận hàng → đọc địa chỉ **1 lần duy nhất**
2. Dán **mã vạch đặc biệt** lên hàng: "Tuyến HN-ĐN-HCM-Q1, ưu tiên cao"
3. Tại Đà Nẵng: chỉ cần **quét mã vạch** → biết ngay gửi đi đâu → **đổi mã mới**
4. Tại TP.HCM: quét mã → gửi Quận 1 → **gỡ mã**
5. Nhanh hơn nhiều! Không ai cần đọc lại địa chỉ dài!

### Định nghĩa kỹ thuật

**MPLS (Multiprotocol Label Switching)** là kỹ thuật chuyển tiếp packet trong mạng dựa trên **nhãn ngắn** (label) thay vì phải tra cứu đầy đủ địa chỉ IP đích.

Các đặc điểm chính:
- **Layer 2.5** — nằm giữa Data Link (L2) và Network (L3)
- **Protocol-independent** — "Multiprotocol" = hoạt động với IPv4, IPv6, Ethernet, ATM...
- **Connection-oriented** — thiết lập path trước, rồi mới chuyển data
- **Label-based forwarding** — dùng label 20-bit thay vì IP lookup

### RFC chính thức

| RFC | Tên | Mô tả |
|---|---|---|
| RFC 3031 | MPLS Architecture | Kiến trúc tổng quan MPLS |
| RFC 3032 | MPLS Label Stack Encoding | Cấu trúc label header |
| RFC 5036 | LDP Specification | Label Distribution Protocol |
| RFC 3209 | RSVP-TE | Extensions cho MPLS LSP tunnels |
| RFC 4364 | BGP/MPLS IP VPNs (L3VPN) | L3VPN specification |
| RFC 4761 | VPLS Using BGP | Virtual Private LAN Service |
| RFC 4762 | VPLS Using LDP | VPLS signaling bằng LDP |

### Vị trí trong mô hình mạng

```
┌─────────────────────────────────────────────────┐
│                    Layer 7-5                      │
│              Application / Session               │
├─────────────────────────────────────────────────┤
│                    Layer 4                        │
│              TCP / UDP                           │
├─────────────────────────────────────────────────┤
│                    Layer 3                        │
│              IP (IPv4 / IPv6)                    │
├─────────────────────────────────────────────────┤
│              ★ Layer 2.5 — MPLS ★               │  ← MPLS shim header
│              Label | TC | S | TTL               │
├─────────────────────────────────────────────────┤
│                    Layer 2                        │
│              Ethernet / PPP / ATM               │
├─────────────────────────────────────────────────┤
│                    Layer 1                        │
│              Physical (Fiber, Copper)            │
└─────────────────────────────────────────────────┘
```

---

## 3. Cấu trúc MPLS Header — Shim Header 32-bit

### Phép so sánh — Vé xe buýt

Mỗi khi bạn lên xe buýt, bạn có **vé** chứa thông tin:
- **Số tuyến** (Label) — tuyến xe nào
- **Loại vé** (TC/EXP) — ưu tiên/thường/sinh viên
- **Chặng cuối chưa?** (Bottom of Stack) — còn phải đổi xe không
- **Hạn sử dụng** (TTL) — vé hết hạn thì bỏ

MPLS label header cũng vậy — đó là "vé" 32-bit dán vào packet:

### Cấu trúc chi tiết MPLS Shim Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                 Label (20 bits)                │TC │S│   TTL (8) │
│                                               │(3)│ │           │
└───────────────────────────────────────────────┴───┴─┴───────────┘
         20 bits                                  3  1     8 bits
                        Tổng: 32 bits (4 bytes)
```

### Chi tiết từng trường

#### 1. Label — 20 bits (giá trị 0–1,048,575)

Đây là **trái tim** của MPLS — con số mà router dùng để quyết định forward packet đi đâu.

```
Label value → Forward ra interface nào, đổi label thành gì
```

**Reserved labels (0–15):**

| Label | Tên | Chức năng |
|---|---|---|
| 0 | IPv4 Explicit NULL | "Gỡ label, xử lý như IPv4 packet" |
| 1 | Router Alert | "Packet cần attention đặc biệt" |
| 2 | IPv6 Explicit NULL | "Gỡ label, xử lý như IPv6 packet" |
| 3 | Implicit NULL | "Penultimate router: gỡ label trước khi gửi" (PHP) |
| 4-6 | Reserved | Chưa sử dụng |
| 7 | Entropy Label Indicator | Dùng cho load balancing |
| 8-12 | Reserved | Chưa sử dụng |
| 13 | GAL (Generic Alert Label) | OAM purposes |
| 14 | OAM Alert | Operations and Maintenance |
| 15 | Reserved | Chưa sử dụng |

#### 2. Traffic Class (TC) — 3 bits (trước đây gọi EXP)

Tương tự **DSCP** trong IP header — dùng cho QoS:
- Xác định priority của packet
- Router dùng để quyết định queuing/dropping
- 3 bits = 8 giá trị (0-7)

```
TC = 0 → Best Effort (thường)
TC = 5 → Voice (ưu tiên cao)
TC = 7 → Network Control (cao nhất)
```

#### 3. Bottom of Stack (S) — 1 bit

MPLS cho phép **chồng nhiều label** (label stack). Bit S cho biết đây có phải label **cuối cùng** (sát IP header nhất) không:
- **S = 1** → Đây là label cuối (bottom), bên dưới là IP header
- **S = 0** → Còn label khác bên dưới

```
┌──────────────┐
│ Label 1 (S=0)│ ← Top label (outer)
├──────────────┤
│ Label 2 (S=0)│ ← Middle label
├──────────────┤
│ Label 3 (S=1)│ ← Bottom label (inner) — S=1
├──────────────┤
│   IP Header  │
├──────────────┤
│   Payload    │
└──────────────┘
```

#### 4. TTL (Time to Live) — 8 bits

Giống TTL trong IP header — ngăn packet loop vô hạn:
- Mỗi hop giảm 1
- TTL = 0 → drop packet
- Ingress LSR copy TTL từ IP header vào MPLS TTL
- Egress LSR copy ngược lại (tuỳ cấu hình)

### Vị trí của MPLS header trong frame

```
┌───────────────┬──────────────┬──────────────┬──────────────┬─────────┐
│  Ethernet     │   MPLS       │   IP         │   TCP/UDP    │ Payload │
│  Header       │   Label(s)   │   Header     │   Header     │         │
│  (14 bytes)   │   (4+ bytes) │   (20 bytes) │   (20 bytes) │         │
│  EtherType:   │              │              │              │         │
│  0x8847(uni)  │              │              │              │         │
│  0x8848(multi)│              │              │              │         │
└───────────────┴──────────────┴──────────────┴──────────────┴─────────┘
```

**EtherType cho MPLS:**
- **0x8847** — MPLS unicast
- **0x8848** — MPLS multicast

### Ví dụ thực tế — Packet capture

```
Ethernet II:
  Destination: 00:1a:2b:3c:4d:5e
  Source:      00:5e:4d:3c:2b:1a
  Type:        0x8847 (MPLS Unicast)

MPLS Label Stack:
  Label 1: Label=24, TC=5, S=0, TTL=253  ← Transport label
  Label 2: Label=18, TC=5, S=1, TTL=253  ← VPN label (bottom)

IPv4:
  Source:      10.0.1.100
  Destination: 10.0.2.200
  TTL:         62
  Protocol:    TCP
```

---

## 4. Kiến trúc MPLS — Thành phần và vai trò

### Phép so sánh — Hệ thống Metro (tàu điện ngầm)

Hệ thống metro có:
- **Cổng vào (Ingress)** — nơi bạn mua vé, quét thẻ vào
- **Các ga giữa (Transit)** — chỉ cần quét thẻ qua, không kiểm tra gì thêm
- **Cổng ra (Egress)** — quét thẻ ra, tính tiền, bạn ra ngoài
- **Hệ thống quản lý** — quyết định tuyến nào, giá vé, lịch tàu

### Các loại router trong MPLS

```
          CE ←→ PE ←→ P ←→ P ←→ PE ←→ CE
         (Customer)  (Provider)        (Customer)
          Edge        Edge              Edge

     ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
     │ CE  │────│ PE  │────│  P  │────│ PE  │────│ CE  │
     │Site1│    │(LER)│    │(LSR)│    │(LER)│    │Site2│
     └─────┘    └─────┘    └─────┘    └─────┘    └─────┘
                 Ingress    Transit     Egress
                 Push label Swap label  Pop label
```

#### 1. CE (Customer Edge) Router

- Router **của khách hàng** — kết nối vào mạng ISP
- **Không biết** gì về MPLS — chỉ chạy IP routing thường
- Giao tiếp với PE bằng routing protocol (BGP, OSPF, static)
- Ví dụ: Router Cisco 2900 tại chi nhánh công ty

#### 2. PE (Provider Edge) Router — còn gọi LER (Label Edge Router)

- Router **biên** của ISP — nơi MPLS "bắt đầu" và "kết thúc"
- **Ingress PE**: nhận IP packet từ CE → **gán label** → đẩy vào MPLS core
- **Egress PE**: nhận labeled packet → **gỡ label** → gửi IP packet cho CE
- Chạy cả IP routing VÀ MPLS
- Giữ VRF tables cho VPN
- Ví dụ: Cisco ASR 9000, Juniper MX series

#### 3. P (Provider) Router — còn gọi LSR (Label Switching Router)

- Router **lõi** của ISP — chỉ nhìn label, KHÔNG nhìn IP header
- **Swap** (đổi) label: nhận label A → đổi thành label B → forward
- Nhanh nhất vì chỉ cần tra **LFIB** (Label Forwarding Information Base)
- Không cần giữ thông tin VPN/customer
- Ví dụ: Cisco CRS, Juniper PTX series

### Bảng thông tin — FIB vs LFIB vs RIB

| Bảng | Tên đầy đủ | Chức năng | Dùng bởi |
|---|---|---|---|
| RIB | Routing Information Base | Tất cả routes learned | Control plane |
| FIB | Forwarding Information Base | Best routes cho IP forwarding | Data plane (IP) |
| LIB | Label Information Base | Tất cả label mappings | Control plane (MPLS) |
| LFIB | Label Forwarding Information Base | Labels cho forwarding | Data plane (MPLS) |

### Forwarding Equivalence Class (FEC)

**FEC** = Nhóm packet được xử lý **giống nhau** (cùng label, cùng path).

Ví dụ FEC:
- Tất cả packet đi đến subnet 10.0.2.0/24 → cùng FEC → cùng label
- Tất cả packet thuộc VPN "Công ty ABC" → cùng FEC
- Tất cả packet có DSCP EF (voice) đi đến X → cùng FEC

```
FEC = {destination prefix, QoS class, VPN, ...}
     → Map to Label → Forward trên cùng LSP
```

### Label Operations — Push, Swap, Pop

| Operation | Ai thực hiện | Mô tả | Ví dụ |
|---|---|---|---|
| **Push** | Ingress PE (LER) | Gán label mới lên packet | IP packet → [Label 24 \| IP] |
| **Swap** | Transit P (LSR) | Đổi label cũ → label mới | [Label 24 \| IP] → [Label 18 \| IP] |
| **Pop** | Egress PE (LER) | Gỡ label, trả IP packet | [Label 18 \| IP] → IP packet |

### PHP — Penultimate Hop Popping

**Vấn đề:** Egress PE phải làm 2 việc: (1) pop label, (2) IP lookup + forward. Tốn tài nguyên!

**Giải pháp PHP:** Router **áp chót** (penultimate) pop label trước:
- Egress PE quảng cáo **Implicit NULL (label 3)** cho neighbor
- Router trước Egress nhận label 3 = "pop label trước khi gửi"
- Egress PE chỉ cần IP lookup — giảm tải!

```
Không có PHP:
  P1 → [Label 18] → PE2 (pop + IP lookup) → CE2

Có PHP:
  P1 (pop label) → [IP packet] → PE2 (IP lookup only) → CE2
```

---

## 5. Label Distribution Protocol (LDP) — Phát hành nhãn

### Phép so sánh — Hệ thống phát hành số thứ tự

Tưởng tượng ngân hàng có **nhiều chi nhánh**:
1. Mỗi chi nhánh phát **số thứ tự** riêng cho khách
2. Khi chuyển khách sang chi nhánh khác, phải **thông báo** số mới
3. Các chi nhánh trao đổi: "Khách có số 45 ở tôi = số 72 ở anh"
4. Nhờ vậy khách được **chuyển tiếp liền mạch** giữa các chi nhánh

LDP hoạt động tương tự — nó là protocol để các router **trao đổi label mappings**.

### LDP hoạt động như thế nào?

#### Bước 1: Discovery — Tìm neighbor

```
LDP Hello messages:
- Gửi UDP multicast 224.0.0.2, port 646
- Interval: 5 giây (default)
- Hold time: 15 giây (default)
- Nếu không nhận Hello trong hold time → neighbor dead
```

#### Bước 2: Session Establishment — TCP connection

```
1. Router có Router-ID lớn hơn → Active role (initiate TCP)
2. TCP connection port 646
3. Initialization message exchange
4. Session parameters negotiation:
   - Label distribution method
   - Timer values
   - Loop detection
```

#### Bước 3: Label Advertisement — Quảng cáo label

```
Router A có route 10.0.2.0/24 (learned via OSPF/IS-IS)
  → A allocate label 24 cho FEC 10.0.2.0/24
  → A gửi Label Mapping message cho tất cả LDP peers:
     "FEC 10.0.2.0/24 = Label 24 ở tôi"

Router B nhận:
  → B ghi vào LIB: "Để đến 10.0.2.0/24, gửi label 24 cho A"
  → B allocate label 18 cho FEC này
  → B gửi tiếp cho các peers khác: "FEC 10.0.2.0/24 = Label 18 ở tôi"
```

### LDP Operating Modes

#### Label Distribution Method

| Mode | Mô tả | Mặc định |
|---|---|---|
| **Downstream Unsolicited (DU)** | Router tự động gửi label cho peers mà không cần request | ✅ Cisco, Juniper |
| **Downstream on Demand (DoD)** | Router chỉ gửi label khi được request | ATM-based |

#### Label Retention Mode

| Mode | Mô tả | Mặc định |
|---|---|---|
| **Liberal** | Giữ TẤT CẢ labels từ mọi peers (dù không dùng) | ✅ Cisco |
| **Conservative** | Chỉ giữ labels từ next-hop peer | Juniper |

**Liberal vs Conservative:**
```
Liberal: Tốn memory, nhưng convergence nhanh (đã có backup label sẵn)
Conservative: Tiết kiệm memory, nhưng phải request label mới khi path change
```

#### Label Allocation Mode

| Mode | Mô tả |
|---|---|
| **Independent** | Router allocate label ngay khi biết route (không đợi downstream) |
| **Ordered** | Router chỉ allocate label khi đã nhận label từ downstream |

### LDP Messages Types

| Message | Chức năng |
|---|---|
| Hello | Discovery — tìm neighbors |
| Initialization | Thiết lập session parameters |
| KeepAlive | Duy trì session (60s default) |
| Label Mapping | "FEC X = Label Y ở tôi" |
| Label Withdraw | "Label Y không còn valid cho FEC X" |
| Label Release | "Tôi không cần label Y nữa" |
| Label Request | "Cho tôi label cho FEC X" (DoD mode) |
| Notification | Thông báo lỗi/events |
| Address | Quảng cáo interface addresses |

### Ví dụ LDP trong action

```
Network topology:
  R1 --- R2 --- R3 --- R4
  10.1.1.0/24          10.4.4.0/24

IGP (OSPF) converged:
  R1: 10.4.4.0/24 via R2
  R2: 10.4.4.0/24 via R3
  R3: 10.4.4.0/24 via R4 (directly connected)

LDP label allocation (Downstream Unsolicited):
  R4: FEC 10.4.4.0/24 → Label 3 (Implicit NULL) → quảng cáo cho R3
  R3: FEC 10.4.4.0/24 → Label 301 → quảng cáo cho R2
  R2: FEC 10.4.4.0/24 → Label 201 → quảng cáo cho R1

LFIB tables:
  R1: In: IP dst 10.4.4.0/24 → Push Label 201, forward to R2
  R2: In: Label 201 → Swap to Label 301, forward to R3
  R3: In: Label 301 → Pop (Implicit NULL), forward to R4
  R4: In: IP packet → IP lookup, deliver
```

---

## 6. RSVP-TE — Traffic Engineering với MPLS

### Phép so sánh — Đặt chỗ trước trên máy bay

**LDP** giống mua vé xe buýt — ai có tiền thì lên, không biết có chỗ ngồi không.

**RSVP-TE** giống **đặt vé máy bay**:
1. Bạn chọn **đường bay cụ thể** (explicit path): HN → ĐN → HCM
2. Bạn **đặt trước ghế** (bandwidth reservation): 1 ghế business
3. Hãng kiểm tra: "Còn ghế không? Còn capacity không?"
4. Nếu OK → **xác nhận** đặt chỗ → bạn có đảm bảo
5. Nếu hết → từ chối hoặc đề xuất chuyến khác

### RSVP-TE là gì?

**RSVP-TE** = Resource Reservation Protocol - Traffic Engineering

Mở rộng RSVP (RFC 2205) với:
- Explicit route specification (chọn đường đi cụ thể)
- Label distribution (phát label theo path đã chọn)
- Bandwidth reservation (đảm bảo bandwidth)
- Fast Reroute (bảo vệ khi link fail)

### So sánh LDP vs RSVP-TE

| Tiêu chí | LDP | RSVP-TE |
|---|---|---|
| Path selection | Follows IGP best path | Explicit path (admin chọn) |
| Bandwidth guarantee | Không | Có — reserve bandwidth |
| Traffic Engineering | Không | Có — CSPF path computation |
| Fast Reroute | Limited | Facility/One-to-One backup |
| Scalability | Tốt (ít state) | Kém hơn (nhiều state per LSP) |
| Setup | Tự động | Cần cấu hình LSP |
| Use case | General forwarding | Voice/Video, SLA guarantee |

### RSVP-TE hoạt động — Step by step

#### Phase 1: Path Computation (CSPF)

**CSPF** = Constrained Shortest Path First

```
Input:
- Destination: 10.4.4.4 (Router ID of egress PE)
- Bandwidth: 100 Mbps
- Constraints: Avoid link R2-R3 (maintenance)
- Explicit hops: R1 → R2 → R5 → R4 (nếu specified)

Process:
1. Headend router chạy Dijkstra (SPF) với constraints
2. Loại bỏ links không đủ bandwidth
3. Loại bỏ links/nodes bị exclude
4. Tìm shortest path thỏa mãn tất cả constraints
5. Kết quả: Explicit Route Object (ERO)
```

#### Phase 2: Path Setup (Signaling)

```
Step 1: PATH message (downstream — headend → tailend)
┌──────────────────────────────────────────────┐
│  PATH Message contains:                       │
│  - SESSION object (tunnel ID, destination)    │
│  - SENDER_TEMPLATE (source, LSP-ID)          │
│  - LABEL_REQUEST (request label allocation)   │
│  - EXPLICIT_ROUTE (ERO — hop-by-hop path)    │
│  - TSPEC (traffic spec — bandwidth request)   │
│  - RECORD_ROUTE (RRO — record actual path)   │
└──────────────────────────────────────────────┘

R1 → PATH → R2 → PATH → R5 → PATH → R4

Step 2: RESV message (upstream — tailend → headend)
┌──────────────────────────────────────────────┐
│  RESV Message contains:                       │
│  - LABEL object (label allocated by each hop)│
│  - FLOWSPEC (confirmed bandwidth)            │
│  - RECORD_ROUTE (RRO — confirmed path)       │
│  - STYLE (reservation style: SE/FF/WF)       │
└──────────────────────────────────────────────┘

R4 → RESV (label=45) → R5 → RESV (label=32) → R2 → RESV (label=19) → R1

Result: LSP established!
  R1: Push label 19, forward to R2
  R2: Swap 19 → 32, forward to R5
  R5: Swap 32 → 45, forward to R4
  R4: Pop label, IP lookup
```

#### Phase 3: Path Maintenance

```
- PATH/RESV refresh mỗi 30 giây (default)
- PathTear — headend xóa LSP
- PathErr — lỗi trên path (bandwidth unavailable, link down)
- ResvTear — tailend xóa reservation
- Summary Refresh (RFC 2961) — giảm refresh overhead
```

### Fast Reroute (FRR) — Bảo vệ dưới 50ms

Khi link/node fail, MPLS FRR cho phép chuyển traffic sang **backup path** trong < 50ms (so với IGP convergence 1-5 giây).

#### Facility Backup (RFC 4090 — Bypass tunnel)

```
Normal path:     R1 → R2 → R3 → R4
Bypass tunnel:   R1 → R2 → R5 → R3 → R4
                           ↑
                     Bypass tunnel (pre-established)
                     Bảo vệ link R2-R3

Khi link R2-R3 fail:
1. R2 detect failure (< 10ms)
2. R2 push thêm 1 label (bypass tunnel label)
3. Traffic đi qua bypass: R2 → R5 → R3 → R4
4. Headend R1 re-compute new primary path
5. Make-before-break: setup path mới trước khi xóa bypass
```

#### One-to-One Backup (Detour)

```
Mỗi LSP có backup path riêng:
- Primary: R1-R2-R3-R4 (label: 19→32→45)
- Detour:  R1-R2-R5-R3-R4 (label: 19→55→67→45)

Khi R2-R3 fail:
  R2 switch sang detour path ngay lập tức
```

### Cấu hình RSVP-TE — Cisco IOS XE

```
! Enable MPLS TE globally
mpls traffic-eng tunnels

! Enable on interfaces
interface GigabitEthernet0/0
 mpls traffic-eng tunnels
 ip rsvp bandwidth 1000000  ! 1 Gbps reservable

! Configure TE tunnel
interface Tunnel0
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 10.4.4.4
 tunnel mpls traffic-eng bandwidth 100000  ! 100 Mbps
 tunnel mpls traffic-eng path-option 1 explicit name PATH1
 tunnel mpls traffic-eng path-option 2 dynamic
 tunnel mpls traffic-eng fast-reroute

! Explicit path
ip explicit-path name PATH1
 next-address 10.0.12.2
 next-address 10.0.25.5
 next-address 10.0.54.4
```

---

## 7. L3VPN — MPLS Layer 3 VPN (RFC 4364)

### Phép so sánh — Tòa nhà văn phòng chia sẻ

Tưởng tượng một **tòa nhà văn phòng** cho thuê:
- **Tòa nhà** = MPLS backbone (ISP)
- **Mỗi tầng** = VPN riêng (cho 1 công ty)
- **Thang máy chung** = MPLS core (P routers)
- **Sảnh tầng** = PE router (phân loại traffic vào đúng tầng)
- **Phòng làm việc** = CE site (chi nhánh công ty)

Công ty A ở tầng 3, Công ty B ở tầng 5:
- Nhân viên Công ty A **không thể** đi sang tầng 5 (isolation)
- Nhưng nhân viên Công ty A ở **tầng 3 Hà Nội** có thể giao tiếp với **tầng 3 TP.HCM** (VPN connectivity)
- Thang máy không biết/quan tâm ai ở tầng nào — chỉ vận chuyển (P router)

### L3VPN là gì?

**MPLS L3VPN** cho phép ISP cung cấp **IP routing riêng biệt** cho mỗi khách hàng qua cùng 1 hạ tầng vật lý.

Đặc điểm:
- Mỗi customer có **bảng routing riêng** (VRF)
- Customer có thể dùng **trùng IP** (10.0.0.0/8) mà không conflict
- ISP **không cần biết** routing nội bộ customer
- PE routers tham gia routing với CE (BGP/OSPF/static)
- P routers **hoàn toàn không biết** gì về VPN

### Thành phần chính

#### 1. VRF — Virtual Routing and Forwarding

```
Trên PE router, mỗi VPN customer có 1 VRF riêng:

VRF "CUSTOMER_A":
  Route Distinguisher: 65001:100
  Route Targets: 
    Import: 65001:100
    Export: 65001:100
  Interfaces: GigabitEthernet0/1
  Routing table:
    10.0.1.0/24 → CE-A-HN (directly connected)
    10.0.2.0/24 → via MP-BGP from PE2

VRF "CUSTOMER_B":
  Route Distinguisher: 65001:200
  Route Targets:
    Import: 65001:200
    Export: 65001:200
  Interfaces: GigabitEthernet0/2
  Routing table:
    10.0.1.0/24 → CE-B-HN (directly connected)  ← Same subnet!
    10.0.3.0/24 → via MP-BGP from PE2
```

Lưu ý: Cả 2 VRF đều có 10.0.1.0/24 nhưng **KHÔNG conflict** — vì chúng ở bảng routing riêng!

#### 2. Route Distinguisher (RD) — 64 bits

RD làm cho routes **unique** trong BGP:
```
VPNv4 address = RD (64 bit) + IPv4 prefix (32 bit) = 96 bits

Customer A: 10.0.1.0/24 → VPNv4: 65001:100:10.0.1.0/24
Customer B: 10.0.1.0/24 → VPNv4: 65001:200:10.0.1.0/24
                                    ↑ Khác RD → khác route!
```

Format RD:
- Type 0: `ASN(2 bytes):Number(4 bytes)` → 65001:100
- Type 1: `IP(4 bytes):Number(2 bytes)` → 10.0.0.1:100
- Type 2: `ASN(4 bytes):Number(2 bytes)` → 4200000001:100

#### 3. Route Target (RT) — BGP Extended Community

RT quyết định route đi VÀO VRF nào:
```
Export RT: Route từ VRF này được "tag" với RT nào
Import RT: VRF này nhận routes có RT nào

Ví dụ Hub-and-Spoke:
  Hub VRF:
    Export RT: 65001:100 (hub-export)
    Import RT: 65001:101 (spoke-export)
  
  Spoke VRF:
    Export RT: 65001:101 (spoke-export)  
    Import RT: 65001:100 (hub-export)

→ Spokes chỉ nhận routes từ Hub, không thấy nhau!
```

#### 4. MP-BGP — Phân phối VPN routes giữa các PE

```
PE1 (Hà Nội) và PE2 (TP.HCM) chạy MP-BGP:

PE1 → BGP Update → PE2:
  AFI/SAFI: VPNv4 unicast (1/128)
  NLRI: 65001:100:10.0.1.0/24
  Next-Hop: 10.255.1.1 (PE1 loopback)
  Extended Community: RT 65001:100
  Label: 24 (VPN label)

PE2 nhận:
  1. Check RT: 65001:100 → match VRF "CUSTOMER_A"
  2. Install route vào VRF CUSTOMER_A:
     10.0.1.0/24 → next-hop PE1, VPN label 24
```

### Luồng data plane — Label Stack

```
CE-A(HN) gửi packet 10.0.1.100 → 10.0.2.200:

1. CE-A → PE1: IP packet [src:10.0.1.100, dst:10.0.2.200]

2. PE1 (Ingress):
   - Lookup VRF CUSTOMER_A: 10.0.2.0/24 → next-hop PE2, VPN label 24
   - Lookup LFIB: PE2 (10.255.2.2) → transport label 301
   - Push 2 labels:
     [Transport Label 301 | VPN Label 24 | IP packet]
   - Forward to P1

3. P1 (Transit):
   - Lookup LFIB: Label 301 → Swap to 401, forward to P2
   - Result: [Transport Label 401 | VPN Label 24 | IP packet]
   - P1 KHÔNG thấy VPN label hay IP header!

4. P2 (Penultimate):
   - Lookup LFIB: Label 401 → Pop (PHP), forward to PE2
   - Result: [VPN Label 24 | IP packet]

5. PE2 (Egress):
   - Lookup LFIB: Label 24 → VRF CUSTOMER_A
   - Pop VPN label → IP packet
   - Lookup VRF routing table: 10.0.2.0/24 → CE-A(HCM)
   - Forward IP packet to CE-A(HCM)

6. CE-A(HCM) nhận: IP packet [src:10.0.1.100, dst:10.0.2.200] ✓
```

### Label Stack cho L3VPN

```
┌──────────────────────────────────────────────────┐
│  Ethernet Header                                  │
├──────────────────────────────────────────────────┤
│  Transport Label (S=0) ← LDP/RSVP-TE label       │  Top
│  Mang packet qua MPLS core đến egress PE         │
├──────────────────────────────────────────────────┤
│  VPN Label (S=1) ← MP-BGP allocated              │  Bottom
│  Chỉ cho PE biết packet thuộc VRF nào            │
├──────────────────────────────────────────────────┤
│  IP Header (customer packet)                      │
├──────────────────────────────────────────────────┤
│  Payload                                          │
└──────────────────────────────────────────────────┘
```

### Cấu hình L3VPN — Cisco IOS XE

```
! PE1 Configuration
!
! 1. Define VRF
vrf definition CUSTOMER_A
 rd 65001:100
 address-family ipv4
  route-target export 65001:100
  route-target import 65001:100

! 2. Assign interface to VRF
interface GigabitEthernet0/1
 vrf forwarding CUSTOMER_A
 ip address 192.168.1.1 255.255.255.0

! 3. CE-PE routing (BGP)
router bgp 65001
 address-family ipv4 vrf CUSTOMER_A
  neighbor 192.168.1.2 remote-as 65100
  neighbor 192.168.1.2 activate

! 4. PE-PE MP-BGP (VPNv4)
router bgp 65001
 neighbor 10.255.2.2 remote-as 65001
 neighbor 10.255.2.2 update-source Loopback0
 address-family vpnv4
  neighbor 10.255.2.2 activate
  neighbor 10.255.2.2 send-community extended
```

---

## 8. L2VPN — MPLS Layer 2 VPN

### Phép so sánh — Dây điện thoại riêng

**L3VPN** giống thuê **tầng riêng** trong tòa nhà — ISP quản lý thang máy (routing).

**L2VPN** giống kéo **dây điện thoại riêng** giữa 2 văn phòng:
- Bạn không cần biết dây đi qua đâu
- Chỉ cần biết: "Nói vào đầu này → nghe ở đầu kia"
- ISP lo phần trung gian — bạn thấy 2 đầu như **nối trực tiếp**

### L2VPN vs L3VPN

| Tiêu chí | L2VPN | L3VPN |
|---|---|---|
| Layer | Layer 2 (Ethernet frames) | Layer 3 (IP packets) |
| PE tham gia routing? | KHÔNG — transparent | CÓ — PE chạy routing với CE |
| Customer thấy gì? | 2 sites nối trực tiếp (same LAN) | ISP là 1 hop |
| Flexibility | Customer tự control routing | ISP manage routing |
| Use case | Extend L2 domain, DC interconnect | Branch office connectivity |
| Protocol | Pseudowire, VPLS, EVPN | MP-BGP VPNv4/v6 |

### Các loại L2VPN

#### 1. VPWS — Virtual Private Wire Service (Point-to-Point)

```
Site A ←——————————————→ Site B
        Pseudowire (PW)
        
- Mô phỏng dây kết nối Point-to-Point
- Ethernet frame từ A được đóng gói qua MPLS → đến B nguyên vẹn
- RFC 4447 (Pseudowire Setup and Maintenance Using LDP)
```

**Pseudowire hoạt động:**
```
CE-A gửi Ethernet frame → PE1:
  [Eth Header | IP | Payload]

PE1 encapsulate:
  [Transport Label | PW Label | Control Word | Eth Header | IP | Payload]

  - Transport Label: mang packet qua MPLS core
  - PW Label: identify pseudowire (PE2 dùng để xác định PW nào)
  - Control Word (optional): sequencing, padding

PE2 decapsulate:
  Gỡ labels → forward Ethernet frame cho CE-B
  [Eth Header | IP | Payload]
```

#### 2. VPLS — Virtual Private LAN Service (Multipoint)

```
     Site A ←─┐
              ├──── VPLS Instance ────┤
     Site B ←─┤   (Virtual Switch)   ├──→ Site D
              ├──                    ──┤
     Site C ←─┘                       └──→ Site E

- Mô phỏng LAN switch — tất cả sites trong cùng broadcast domain
- MAC learning trên PE routers
- BUM flooding (Broadcast, Unknown unicast, Multicast)
- Full mesh pseudowires giữa các PE
```

**VPLS signaling:**
- **LDP-based VPLS** (RFC 4762) — Martini/Kompella
- **BGP-based VPLS** (RFC 4761) — dùng BGP Auto-discovery

#### 3. EVPN — Ethernet VPN (Modern replacement)

```
EVPN (RFC 7432) = Next-gen L2VPN:
- Control-plane MAC learning (không flood)
- Active-Active multihoming
- MAC mobility
- IP routing integration (IRB)
- Dùng BGP để distribute MAC/IP bindings
```

### Cấu hình VPWS — Cisco IOS XE

```
! PE1 Configuration
!
! Pseudowire to PE2
interface GigabitEthernet0/1
 no ip address
 xconnect 10.255.2.2 100 encapsulation mpls
  ! 10.255.2.2 = PE2 loopback
  ! 100 = Virtual Circuit ID (phải match cả 2 PE)
```

### Cấu hình VPLS — Cisco IOS XE

```
! PE1 Configuration
!
l2vpn vfi context VPLS_CUST_A
 vpn id 100
 member 10.255.2.2 encapsulation mpls
 member 10.255.3.3 encapsulation mpls

bridge-domain 100
 member GigabitEthernet0/1 service-instance 10
 member vfi VPLS_CUST_A
```

---

## 9. MPLS trong ISP Backbone — Kiến trúc thực tế

### Phép so sánh — Hệ thống đường cao tốc quốc gia

ISP backbone giống **hệ thống đường cao tốc liên tỉnh**:
- **Core (P routers)** = Đường cao tốc 6 làn — capacity lớn, tốc độ cao
- **Edge (PE routers)** = Nút giao — nơi xe vào/ra cao tốc
- **Access** = Đường địa phương — kết nối từ nhà bạn đến nút giao
- **Services** = Loại xe: xe khách (Internet), xe tải (VPN), xe cứu thương (voice)

### Kiến trúc ISP backbone điển hình

```
                    ┌─────────────────────────────────┐
                    │         MPLS Core               │
                    │                                 │
   Enterprise──┐   │  ┌────┐    ┌────┐    ┌────┐   │   ┌──Enterprise
   (L3VPN)     │   │  │ P1 │────│ P2 │────│ P3 │   │   │  (L3VPN)
               ▼   │  └─┬──┘    └─┬──┘    └──┬─┘   │   ▼
            ┌─────┐│    │          │          │     │┌─────┐
   CE-A ────│ PE1 │├────┘    ┌────┐     ┌────┘     ├│ PE3 │──── CE-C
            └─────┘│         │ P4 │     │           │└─────┘
                    │         └────┘     │           │
            ┌─────┐│              │     │           │┌─────┐
   CE-B ────│ PE2 │├──────────────┘     └───────────┤│ PE4 │──── CE-D
   (L2VPN)  └─────┘│                               │└─────┘  (L2VPN)
                    │         MPLS Core               │
                    └─────────────────────────────────┘
                    
   IGP: OSPF hoặc IS-IS (reachability cho loopbacks)
   LDP: Label distribution cho transport
   RSVP-TE: Traffic engineering (optional)
   MP-BGP: VPN route distribution (PE-PE)
```

### Protocol Stack trong ISP backbone

```
Layer          Core (P)              Edge (PE)
─────────────────────────────────────────────────────
Routing        OSPF/IS-IS            OSPF/IS-IS + BGP
Label distro   LDP + RSVP-TE         LDP + RSVP-TE
VPN signaling  —                     MP-BGP (VPNv4/v6)
Forwarding     MPLS (label only)     MPLS + IP
Services       —                     L3VPN, L2VPN, Internet
```

### Segment Routing — Tương lai của MPLS

**Segment Routing (SR)** là evolution của MPLS, giảm bớt complexity:

```
Truyền thống (LDP/RSVP-TE):
- LDP session giữa mọi cặp router
- RSVP-TE state per LSP tại mỗi transit router
- Nhiều protocol → phức tạp

Segment Routing:
- Encode path trong packet header (label stack)
- Không cần LDP session
- Không cần RSVP-TE state trên transit
- IGP extension (OSPF-SR, IS-IS-SR) distribute labels
- Mỗi node/link có "segment" (label) cố định
```

**Segment types:**
- **Node SID** — đại diện cho 1 node (reachability)
- **Adjacency SID** — đại diện cho 1 link cụ thể
- **Prefix SID** — đại diện cho 1 prefix

```
Ví dụ SR path:
  Source: R1
  Destination: R4
  Via: R1 → R2 → R5 → R4

  Label stack: [16002 | 24005 | 16004]
    16002 = Node SID R2 (reach R2)
    24005 = Adj SID R2→R5 (use specific link)
    16004 = Node SID R4 (reach R4)
```

### ISP Services trên MPLS backbone

| Service | Mô tả | Revenue |
|---|---|---|
| Internet Access | PE là default gateway cho customers | Cơ bản |
| L3VPN | IP VPN cho enterprise (multi-site) | Cao |
| L2VPN/VPLS | Ethernet extension service | Cao |
| MPLS TE | Guaranteed bandwidth paths | Premium |
| Managed CE | ISP quản lý CE router cho customer | Add-on |
| Cloud Connect | Direct connect tới AWS/Azure/GCP | Cao |

### Redundancy và High Availability

```
PE redundancy:
- Dual-homed CE (CE nối 2 PE)
- BGP bestpath selection cho failover

P router redundancy:
- Redundant core links
- ECMP (Equal Cost Multi-Path)
- RSVP-TE FRR (Fast Reroute)

Control plane HA:
- NSR (Non-Stop Routing)
- NSF (Non-Stop Forwarding)
- GR (Graceful Restart)
- BFD (Bidirectional Forwarding Detection) — failure detect < 50ms
```

---

## 10. Thực hành và Troubleshooting MPLS

### Lab environment

```
Tools để học MPLS:
- GNS3 + Cisco IOS/IOS-XR images
- EVE-NG + virtual routers
- Containerlab + Nokia SR Linux / FRRouting
- Cisco VIRL/CML
- Juniper vLabs
```

### Commands kiểm tra MPLS — Cisco IOS

```bash
# Kiểm tra MPLS forwarding table (LFIB)
show mpls forwarding-table
# Output:
# Local    Outgoing   Prefix       Outgoing
# Label    Label      or Tunnel    Interface
# 16       Pop Label  10.0.0.2/32  Gi0/0
# 17       22         10.0.0.3/32  Gi0/1
# 18       Pop Label  10.0.0.4/32  Gi0/1

# Kiểm tra LDP neighbors
show mpls ldp neighbor
# Neighbor ID: 10.0.0.2, State: Oper

# Kiểm tra LDP bindings (LIB)
show mpls ldp bindings
# lib entry: 10.0.0.3/32, rev 4
#   local binding: label: 17
#   remote binding: lsr: 10.0.0.2, label: 22

# Kiểm tra MPLS interfaces
show mpls interfaces

# Kiểm tra RSVP-TE tunnels
show mpls traffic-eng tunnels brief

# Kiểm tra VRF routes
show ip route vrf CUSTOMER_A

# Kiểm tra BGP VPNv4 routes
show bgp vpnv4 unicast all

# Trace MPLS path
traceroute mpls ipv4 10.0.0.4/32
```

### Commands — Juniper JunOS

```bash
# MPLS forwarding table
show route table mpls.0

# LDP neighbors
show ldp neighbor

# LDP database
show ldp database

# RSVP-TE LSPs
show mpls lsp

# L3VPN routing table
show route table CUSTOMER_A.inet.0

# BGP VPNv4 routes
show route table bgp.l3vpn.0
```

### Troubleshooting scenarios

#### Scenario 1: LDP session không UP

```
Symptoms: MPLS label không được phân phối

Checklist:
1. ✅ MPLS enabled trên interface? (mpls ip)
2. ✅ LDP router-id reachable? (ping loopback)
3. ✅ IGP adjacency UP? (show ip ospf neighbor)
4. ✅ LDP transport address match? (mpls ldp router-id)
5. ✅ TCP port 646 không bị firewall block?
6. ✅ LDP Hello reaching neighbor? (debug mpls ldp messages)
```

#### Scenario 2: L3VPN route không appear

```
Symptoms: CE-A không ping được CE-B qua VPN

Checklist:
1. ✅ VRF configured đúng trên PE? (show vrf)
2. ✅ CE-PE routing OK? (show ip route vrf X)
3. ✅ Route exported with correct RT? (show bgp vpnv4 unicast all)
4. ✅ MP-BGP session UP giữa PEs? (show bgp vpnv4 summary)
5. ✅ RT import match? (route-target import)
6. ✅ Transport label available? (show mpls forwarding-table)
7. ✅ Next-hop reachable? (PE loopback in IGP)
```

#### Scenario 3: Traffic blackhole sau PHP

```
Symptoms: Packet đến egress PE nhưng bị drop

Root cause: Egress PE không có route trong VRF

Fix:
1. Check VPN label → VRF mapping
2. Verify CE-PE routing
3. Check import RT matching
```

### MPLS trong Cloud — AWS Direct Connect

```
AWS Direct Connect sử dụng MPLS concepts:

Customer Router ←→ DX Location ←→ AWS Backbone
     CE              PE (AWS)         P (AWS)

- VLAN tag = tương tự MPLS label cho isolation
- BGP peering = MP-BGP cho route exchange
- VIF (Virtual Interface) = tương tự VRF concept
  - Private VIF → access VPC
  - Public VIF → access AWS public services
  - Transit VIF → access Transit Gateway
```

### Best Practices triển khai MPLS

| Practice | Mô tả |
|---|---|
| IGP before MPLS | Đảm bảo IGP converged trước khi enable MPLS |
| Loopback as router-id | Dùng loopback cho LDP transport, BGP |
| BFD everywhere | Enable BFD cho fast failure detection |
| PHP default | Để PHP enabled (default) — giảm tải egress PE |
| Core hiding | P router loopbacks không advertise ra ngoài |
| Separate RR | Route Reflector riêng cho VPNv4 (scalability) |
| Label filtering | Chỉ allocate label cho loopbacks (LDP label filtering) |

### Tổng kết

```
MPLS Ecosystem:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   Control Plane:                                        │
│   ┌─────┐  ┌──────────┐  ┌─────────┐  ┌──────────┐   │
│   │ IGP │  │   LDP    │  │ RSVP-TE │  │  MP-BGP  │   │
│   │OSPF │  │  Label   │  │  Label  │  │  VPN     │   │
│   │IS-IS│  │  Distro  │  │  + BW   │  │  Routes  │   │
│   └─────┘  └──────────┘  └─────────┘  └──────────┘   │
│                                                         │
│   Data Plane:                                           │
│   ┌─────────────────────────────────────────────────┐   │
│   │  LFIB: Label → {Swap/Push/Pop, Interface}       │   │
│   │  FIB:  IP prefix → {Next-hop, Interface}        │   │
│   └─────────────────────────────────────────────────┘   │
│                                                         │
│   Services:                                             │
│   ┌──────────┐  ┌──────────┐  ┌───────┐  ┌────────┐  │
│   │  L3VPN   │  │  L2VPN   │  │  TE   │  │  6PE   │  │
│   │(RFC 4364)│  │(RFC 4762)│  │(CSPF) │  │(IPv6)  │  │
│   └──────────┘  └──────────┘  └───────┘  └────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**Tóm tắt kiến thức cốt lõi:**

1. **MPLS = Label-based forwarding** — nhanh hơn IP lookup truyền thống
2. **Shim header 32-bit** — Label(20) + TC(3) + S(1) + TTL(8)
3. **LDP** — phát label tự động theo IGP topology
4. **RSVP-TE** — traffic engineering với bandwidth reservation
5. **L3VPN** — IP VPN sử dụng VRF + MP-BGP + 2-label stack
6. **L2VPN** — Ethernet extension qua pseudowire/VPLS/EVPN
7. **PHP** — penultimate hop pop để giảm tải egress PE
8. **FRR** — fast reroute < 50ms cho HA
9. **Segment Routing** — tương lai, thay thế LDP/RSVP-TE
10. **ISP backbone** — MPLS là nền tảng cho mọi service

---

*Tài liệu tham khảo:*
- RFC 3031 — Multiprotocol Label Switching Architecture
- RFC 3032 — MPLS Label Stack Encoding
- RFC 5036 — LDP Specification
- RFC 3209 — RSVP-TE: Extensions to RSVP for LSP Tunnels
- RFC 4364 — BGP/MPLS IP Virtual Private Networks (VPNs)
- RFC 4762 — Virtual Private LAN Service (VPLS) Using LDP Signaling
- RFC 7432 — BGP MPLS-Based Ethernet VPN
- RFC 8402 — Segment Routing Architecture
- Cisco MPLS Configuration Guide — IOS XE
- Juniper Networks MPLS Applications Feature Guide
