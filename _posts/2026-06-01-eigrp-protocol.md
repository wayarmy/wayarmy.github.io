---
layout: post
title: "EIGRP Protocol - Thuật toán DUAL, Feasible Distance, Successor và Unequal-Cost Load Balancing"
date: 2026-06-01
categories: [networking]
tags: [eigrp, dual, routing, cisco, load-balancing]
---

## 1. Giới thiệu — Khi GPS biết đường tắt mà bạn không biết

Hãy tưởng tượng bạn đang lái xe từ nhà đến công ty. Google Maps cho bạn **một con đường tốt nhất** — nhưng đồng thời, nó cũng đã tính sẵn **2-3 đường thay thế** phòng khi đường chính bị kẹt. Khi có tai nạn trên đường chính, Maps chuyển bạn sang đường phụ **ngay lập tức** — không cần dừng lại tính toán lại từ đầu.

**EIGRP (Enhanced Interior Gateway Routing Protocol)** hoạt động y hệt như vậy cho mạng máy tính. Nó là một routing protocol của Cisco (giờ đã là open standard qua RFC 7868) có khả năng:

- **Tính sẵn đường backup** (Feasible Successor) — chuyển đổi trong milliseconds khi đường chính hỏng
- **Cân bằng tải không đều** (Unequal-Cost Load Balancing) — sử dụng nhiều đường cùng lúc dù tốc độ khác nhau
- **Chỉ gửi thay đổi** (Partial Updates) — không flood toàn bộ bảng routing như OSPF
- **Hội tụ cực nhanh** — nhanh nhất trong các Interior Gateway Protocol

### Tại sao bạn cần biết EIGRP?

| Tình huống | Tại sao EIGRP quan trọng |
|---|---|
| Quản trị mạng doanh nghiệp | 60%+ mạng enterprise dùng EIGRP (đặc biệt Cisco-dominant) |
| Thi chứng chỉ CCNA/CCNP | Bắt buộc phải hiểu sâu DUAL algorithm |
| Thiết kế mạng WAN | Unequal-cost LB tận dụng tối đa bandwidth |
| Troubleshooting | Hiểu Feasible Distance giúp debug routing loops |

---

## 2. EIGRP là gì? — Giải thích cho người không biết IT

### Phép so sánh đời thường

Hãy tưởng tượng bạn là **quản lý một chuỗi cửa hàng giao hàng**. Bạn cần biết đường đi tốt nhất đến mỗi khu vực:

- **RIP** (Distance Vector cũ): Bạn chỉ hỏi hàng xóm "mày đi đến khu A mất bao lâu?" — rồi cộng thêm thời gian từ mình đến hàng xóm. Đơn giản nhưng chậm và dễ sai.

- **OSPF** (Link-State): Bạn có **bản đồ toàn bộ thành phố** — biết mọi con đường, mọi ngã tư. Chính xác nhưng tốn bộ nhớ và CPU.

- **EIGRP** (Advanced Distance Vector): Bạn hỏi hàng xóm nhưng **thông minh hơn** — hỏi chi tiết (tốc độ đường, mức độ kẹt xe, độ tin cậy), đồng thời **ghi nhớ đường backup** sẵn. Khi đường chính bị chặn, chuyển sang backup ngay — không cần hỏi lại ai cả.

### Định nghĩa kỹ thuật

**EIGRP** là một **Advanced Distance Vector** routing protocol (hay còn gọi là Hybrid) sử dụng thuật toán **DUAL (Diffusing Update Algorithm)** để:
1. Tính toán đường đi tốt nhất (Successor)
2. Xác định trước đường backup loop-free (Feasible Successor)
3. Đảm bảo **không bao giờ có routing loop** thông qua Feasibility Condition

**Đặc điểm chính:**
- Protocol number: 88 (chạy trực tiếp trên IP, không dùng TCP/UDP)
- Multicast address: 224.0.0.10
- Administrative Distance: Internal = 90, External = 170
- Hỗ trợ: IPv4, IPv6, IPX, AppleTalk (multi-protocol)
- Convergence time: Sub-second (khi có Feasible Successor)

### So sánh nhanh với các protocol khác

| Tiêu chí | RIP | OSPF | EIGRP |
|---|---|---|---|
| Loại | Distance Vector | Link-State | Advanced Distance Vector |
| Metric | Hop count (max 15) | Cost (bandwidth) | Composite (BW + Delay + ...) |
| Convergence | 30-180 giây | 5-40 giây | < 1 giây (có FS) |
| Loop prevention | Split horizon, holddown | SPF algorithm | DUAL + Feasibility Condition |
| Load balancing | Equal-cost only | Equal-cost only | Equal + Unequal cost |
| Scalability | Nhỏ (< 15 hops) | Lớn (area design) | Trung bình-Lớn |

---

## 3. Thuật toán DUAL — Bộ não thông minh của EIGRP

### Mini-example: Cuộc họp tìm đường

Tưởng tượng bạn đang ở ngã tư và cần đến **Bệnh viện X**. Bạn hỏi 3 người đứng quanh:

- **Người A** nói: "Từ chỗ tôi đến bệnh viện mất 10 phút" (Reported Distance = 10). Từ bạn đến người A mất 5 phút. → **Tổng = 15 phút** (Feasible Distance qua A)
- **Người B** nói: "Từ chỗ tôi mất 8 phút" (RD = 8). Từ bạn đến B mất 12 phút. → **Tổng = 20 phút** (FD qua B)
- **Người C** nói: "Từ chỗ tôi mất 6 phút" (RD = 6). Từ bạn đến C mất 20 phút. → **Tổng = 26 phút** (FD qua C)

**Kết quả:**
- **Successor** = Người A (tổng 15 phút — thấp nhất)
- **Feasible Distance (FD)** = 15 (metric tốt nhất hiện tại)
- **Feasibility Condition**: RD của backup < FD hiện tại
  - Người B: RD = 8 < FD = 15 ✓ → **Feasible Successor!**
  - Người C: RD = 6 < FD = 15 ✓ → **Feasible Successor!**

Nếu người A biến mất, bạn **ngay lập tức** dùng người B (FD = 20) — không cần hỏi lại ai!

### Giải thích kỹ thuật DUAL

**DUAL (Diffusing Update Algorithm)** được phát triển bởi Dr. J.J. Garcia-Luna-Aceves tại SRI International. Nó giải quyết bài toán: **"Làm sao tìm đường backup mà CHẮC CHẮN không tạo loop?"**

#### Các khái niệm cốt lõi:

**1. Reported Distance (RD) — Advertised Distance:**
```
RD = Metric mà neighbor quảng bá cho ta
   = Khoảng cách từ neighbor đến destination
```

**2. Feasible Distance (FD):**
```
FD = Metric từ local router đến destination qua best path
   = Cost(local → neighbor) + RD(neighbor → destination)
```

**3. Successor:**
```
Successor = Neighbor có FD thấp nhất đến destination
          = Next-hop cho đường đi tốt nhất
          = Đưa vào Routing Table
```

**4. Feasible Successor (FS):**
```
FS = Neighbor thỏa mãn Feasibility Condition
   = RD(neighbor) < Current FD
   = Backup path loop-free đã được verify
   = Chỉ ở trong Topology Table (không vào Routing Table)
```

**5. Feasibility Condition (FC) — Điều kiện khả thi:**
```
Một neighbor trở thành Feasible Successor KHI VÀ CHỈ KHI:
    Reported Distance của nó < Feasible Distance hiện tại

Tại sao điều này đảm bảo loop-free?
- Nếu RD(neighbor) < FD(local), nghĩa là neighbor
  gần destination hơn local router
- Neighbor KHÔNG THỂ đi qua local router để đến destination
  (vì đường đó sẽ dài hơn RD của nó)
- Do đó, KHÔNG CÓ LOOP!
```

#### Trạng thái của DUAL:

```
┌─────────────────────────────────────────────────┐
│              DUAL State Machine                   │
├─────────────────────────────────────────────────┤
│                                                   │
│  PASSIVE State (bình thường)                     │
│  ├── Successor available → Route ổn định         │
│  ├── Topology change received                    │
│  │   ├── FS exists? → Switch to FS (stay PASSIVE)│
│  │   └── No FS? → Go ACTIVE                     │
│  │                                               │
│  ACTIVE State (đang tính toán)                   │
│  ├── Send QUERY to all neighbors                 │
│  ├── Wait for REPLY from all queried neighbors   │
│  ├── All REPLY received → Calculate new route    │
│  └── Return to PASSIVE                           │
│                                                   │
│  Stuck-In-Active (SIA) — Timeout!                │
│  └── Neighbor không reply trong 3 phút           │
│      → Reset neighbor relationship               │
└─────────────────────────────────────────────────┘
```

### Trong thực tế

**Scenario: Mạng 3 chi nhánh HCM-HN-DN**

```
HCM ──── 100Mbps ──── HN
  │                      │
  └── 50Mbps ── DN ── 50Mbps ─┘

Từ HCM đến HN:
- Đường trực tiếp: FD = 28160 (qua link 100Mbps)
- Đường qua DN: FD = 56320 (qua 2 link 50Mbps)

RD của DN đến HN = 28160
FC: 28160 < 28160? → NO! (phải NHỎ HƠN, không bằng)
→ DN KHÔNG phải Feasible Successor!

Nếu link trực tiếp down → DUAL goes ACTIVE → Query DN → Reply → New route
```

### Trong AWS

AWS không sử dụng EIGRP trực tiếp (AWS dùng BGP cho Direct Connect và VPN). Tuy nhiên, khái niệm DUAL có thể ánh xạ:

- **Successor** ≈ Primary route trong Route Table
- **Feasible Successor** ≈ Route với lower priority (failover route)
- **Convergence** ≈ Route propagation time trong Transit Gateway

Khi thiết kế hybrid cloud với EIGRP on-premises:
```
On-Premises (EIGRP) ←→ AWS Direct Connect (BGP)
                     ←→ AWS Site-to-Site VPN (BGP backup)

EIGRP redistribute BGP routes vào internal network
BGP redistribute EIGRP routes ra AWS
```

---

## 4. EIGRP Metric — Công thức tính "khoảng cách"

### Mini-example: Chọn đường giao hàng

Khi shipper chọn đường giao hàng, họ cân nhắc nhiều yếu tố:
- **Tốc độ đường** (bandwidth) — đường cao tốc vs đường làng
- **Khoảng cách** (delay) — 5km vs 20km
- **Mức độ kẹt xe** (load) — giờ cao điểm vs khuya
- **Đường có hay hỏng không** (reliability) — đường đang sửa vs đường tốt

EIGRP cũng làm tương tự — nhưng với **công thức toán học chính xác**.

### Công thức Metric

```
EIGRP Composite Metric = 256 × [(K1 × BW) + (K2 × BW)/(256 - Load) + (K3 × Delay)] × [K5/(Reliability + K4)]

Trong đó:
- BW = 10^7 / Bandwidth_thấp_nhất_trên_path (đơn vị: Kbps)
- Delay = Tổng_delay_các_interface / 10 (đơn vị: 10 microseconds)
- Load = Mức sử dụng interface (1-255)
- Reliability = Độ tin cậy (1-255, 255 = 100%)

K-values mặc định:
- K1 = 1 (Bandwidth: ON)
- K2 = 0 (Load: OFF)
- K3 = 1 (Delay: ON)
- K4 = 0 (Reliability trong mẫu: OFF)
- K5 = 0 (Khi K5=0, phần [K5/(Rel+K4)] bị bỏ qua)
```

**Với K-values mặc định, công thức đơn giản hóa:**
```
Metric = 256 × (BW + Delay)
       = 256 × (10^7/BW_min_kbps + Sum_Delay/10)
```

### Ví dụ tính toán thực tế

```
Path: R1 → FastEthernet → R2 → Serial T1 → R3

FastEthernet: BW = 100,000 Kbps, Delay = 100 μs
Serial T1:   BW = 1,544 Kbps,   Delay = 20,000 μs

BW component = 10^7 / min(100000, 1544) = 10^7 / 1544 = 6476
Delay component = (100 + 20000) / 10 = 2010

Metric = 256 × (6476 + 2010) = 256 × 8486 = 2,172,416
```

### Tại sao chỉ dùng Bandwidth và Delay?

| Factor | Tại sao ON/OFF mặc định |
|---|---|
| Bandwidth (K1=1) | Ổn định, hiếm thay đổi, phản ánh capacity thực |
| Delay (K3=1) | Ổn định, phản ánh khoảng cách/công nghệ |
| Load (K2=0) | Thay đổi liên tục → route flapping → không ổn định |
| Reliability (K4,K5=0) | Thay đổi → gây route flapping |

**Quan trọng:** Nếu 2 router có K-values khác nhau → **KHÔNG thể thành neighbor!** EIGRP kiểm tra K-values trong Hello packet.

### Wide Metric (EIGRP Named Mode)

Classic EIGRP metric có vấn đề với interface > 10Gbps (metric = 0). EIGRP Named Mode dùng **Wide Metric**:

```
Wide Metric = 65536 × [(K1 × BW) + (K2 × BW)/(256-Load) + (K3 × Delay) + (K6 × Extended)]

Throughput = 10^7 × 65536 / BW_kbps
Latency = Delay_in_picoseconds / 10^6

Rib Scale Factor (mặc định 128) để fit vào 32-bit routing table
```

### Trong thực tế

**Tuning metric cho traffic engineering:**
```cisco
! Thay đổi delay trên interface để ảnh hưởng path selection
interface GigabitEthernet0/1
 delay 100        ! Tăng delay → metric tăng → path kém ưu tiên hơn

! Thay đổi bandwidth (không ảnh hưởng actual speed!)
interface Serial0/0
 bandwidth 768    ! Chỉ ảnh hưởng metric calculation
```

**Lưu ý quan trọng:** Lệnh `bandwidth` trên interface **KHÔNG** thay đổi tốc độ thực tế — nó chỉ ảnh hưởng cách EIGRP (và QoS) tính toán!

### Trong AWS

Khi redistribute routes giữa EIGRP và BGP (hybrid cloud):
```cisco
! Redistribute BGP vào EIGRP với metric seeds
router eigrp 100
 redistribute bgp 65000 metric 100000 1 255 1 1500
 !                       BW    Delay Rel Load MTU
 ! Seed metric cho routes từ BGP không có EIGRP metric
```

---

## 5. EIGRP Tables và Packet Types — Bộ nhớ của Router

### Mini-example: Sổ ghi chép của shipper

Một shipper giỏi có 3 cuốn sổ:
1. **Sổ liên lạc** (Neighbor Table) — danh sách shipper khác mình biết, số phone, khu vực
2. **Sổ tất cả các đường** (Topology Table) — mọi đường biết được, đường nào tốt, đường nào backup
3. **Sổ đường đi hàng ngày** (Routing Table) — chỉ các đường tốt nhất đang dùng

### Ba bảng của EIGRP

#### 1. Neighbor Table

```
show ip eigrp neighbors

H   Address     Interface    Hold   Uptime   SRTT  RTO   Q   Seq
                             (sec)           (ms)        Cnt  Num
0   10.1.1.2    Gi0/0        12     01:30:45  4    50    0    152
1   10.2.2.2    Gi0/1        14     00:45:30  8    100   0    89

Giải thích:
- H: Handle number (thứ tự phát hiện)
- Hold: Thời gian chờ trước khi coi neighbor "chết" (mặc định 15s = 3×Hello)
- SRTT: Smooth Round-Trip Time (thời gian gửi-nhận reliable packet)
- RTO: Retransmission Timeout (thời gian chờ trước khi gửi lại)
- Q Cnt: Số packet trong queue chờ gửi (> 0 = có vấn đề!)
- Seq Num: Sequence number cuối cùng nhận từ neighbor
```

#### 2. Topology Table

```
show ip eigrp topology

EIGRP-IPv4 VR(NAMED) Topology Table for AS(100)/ID(1.1.1.1)
Codes: P - Passive, A - Active, U - Update, Q - Query, R - Reply, r - reply Status, s - sia Status

P 192.168.1.0/24, 2 successors, FD is 28160, serno 45
        via 10.1.1.2 (28160/25600), GigabitEthernet0/0    ← Successor
        via 10.2.2.2 (30720/25600), GigabitEthernet0/1    ← Successor #2
        via 10.3.3.2 (33280/25600), GigabitEthernet0/2    ← Feasible Successor

P = Passive (ổn định, đường đã chọn xong)
A = Active (đang tìm đường mới, gửi Query)

(28160/25600) = (Feasible Distance / Reported Distance)
```

#### 3. Routing Table

```
show ip route eigrp

D     192.168.1.0/24 [90/28160] via 10.1.1.2, 01:30:45, GigabitEthernet0/0
                     [90/30720] via 10.2.2.2, 00:45:30, GigabitEthernet0/1
D EX  172.16.0.0/16  [170/33792] via 10.1.1.2, 00:30:00, GigabitEthernet0/0

D = EIGRP Internal (AD = 90)
D EX = EIGRP External - redistributed (AD = 170)
[90/28160] = [Administrative Distance / Metric]
```

### 5 loại EIGRP Packet

```
┌─────────────────────────────────────────────────────────────────┐
│ Packet Type │ Opcode │ Reliable? │ Multicast/Unicast │ Mục đích │
├─────────────┼────────┼───────────┼───────────────────┼──────────┤
│ Hello       │   5    │    No     │ Multicast 224.0.0.10│ Discovery│
│ Update      │   1    │    Yes    │ Multi or Unicast  │ Route info│
│ Query       │   3    │    Yes    │ Multicast         │ Ask route │
│ Reply       │   4    │    Yes    │ Unicast           │ Answer    │
│ ACK         │   5    │    ---    │ Unicast           │ Confirm   │
└─────────────────────────────────────────────────────────────────┘
```

**Chi tiết từng loại:**

**Hello Packet:**
- Gửi mỗi 5 giây (LAN) hoặc 60 giây (WAN ≤ T1)
- Hold time = 3× Hello interval (15s hoặc 180s)
- Mang thông tin: K-values, AS number, Hold time
- **Không cần ACK** (unreliable)
- Nếu không nhận Hello trong Hold time → neighbor dead

**Update Packet:**
- Chứa routing information (network, metric)
- **Triggered updates** — chỉ gửi khi có thay đổi (không periodic)
- Gửi Multicast khi có thay đổi topology
- Gửi Unicast khi có neighbor mới (full topology dump)
- **Cần ACK** (reliable delivery)

**Query Packet:**
- Gửi khi mất Successor VÀ không có Feasible Successor
- DUAL goes ACTIVE → gửi Query đến ALL neighbors
- "Có ai biết đường đến network X không?"
- **Cần ACK** (reliable)

**Reply Packet:**
- Trả lời Query — "Đây là metric của tôi đến X" hoặc "Tôi không biết"
- Gửi Unicast về router đã Query
- **Cần ACK** (reliable)

**ACK Packet:**
- Thực chất là Hello packet với data trống
- Xác nhận đã nhận Update/Query/Reply
- Gửi Unicast

### Reliable Transport Protocol (RTP)

EIGRP tự implement reliable delivery (không dùng TCP):
```
Sequence-based acknowledgment:
1. Router A gửi Update với Seq# 100
2. Router B nhận → gửi ACK với Ack# 100
3. Nếu A không nhận ACK trong RTO → Retransmit (unicast)
4. Retransmit 16 lần → Neighbor reset

Conditional Receive (CR) mode:
- Multicast packet chỉ cần ACK từ neighbor "chậm"
- Neighbor nhanh tự hiểu đã nhận (qua sequence tracking)
- Tối ưu cho multi-access networks
```

### Trong thực tế

**Troubleshooting neighbor flapping:**
```cisco
! Kiểm tra Hello/Hold timers
show ip eigrp interfaces detail

! Debug EIGRP packets (cẩn thận trên production!)
debug eigrp packets hello
debug eigrp packets query

! Kiểm tra stuck-in-active
show ip eigrp topology active
```

### Trong AWS

Khi dùng EIGRP trên EC2 instances (virtual router):
- Hello multicast 224.0.0.10 cần được allow trong Security Group
- VPC không hỗ trợ multicast mặc định → cần multicast overlay hoặc unicast EIGRP
- Transit Gateway Multicast hỗ trợ từ 2020 cho một số use cases

---

## 6. Neighbor Discovery và Adjacency — Kết bạn giữa các Router

### Mini-example: Quy trình kết bạn

Giống như khi bạn chuyển đến khu phố mới:
1. Bạn **chào hỏi** tất cả hàng xóm (Hello multicast)
2. Hàng xóm **chào lại** (Hello reply)
3. Kiểm tra **có hợp nhau không** — cùng ngôn ngữ, cùng khu phố (K-values, AS number, subnet)
4. Nếu hợp → **chia sẻ thông tin** (Update — full topology exchange)
5. Duy trì quan hệ bằng **chào hỏi định kỳ** (Hello mỗi 5s)

### Điều kiện thành lập Neighbor

```
┌─────────────────────────────────────────────────┐
│         EIGRP Neighbor Requirements              │
├─────────────────────────────────────────────────┤
│ 1. Cùng AS Number                                │
│ 2. Cùng K-values (K1-K5)                        │
│ 3. Interface cùng subnet (hoặc matching)         │
│ 4. Authentication match (nếu configured)         │
│ 5. Nhận được Hello từ nhau                       │
│                                                   │
│ KHÔNG yêu cầu (khác OSPF):                      │
│ - Hello/Hold timer match                         │
│ - MTU match                                      │
│ - Cùng network type                             │
└─────────────────────────────────────────────────┘
```

### Quá trình thiết lập Adjacency

```
Timeline: R1 ←→ R2 Neighbor Establishment

T=0s:  R1 gửi Hello (multicast 224.0.0.10)
       - AS number: 100
       - K-values: K1=1, K2=0, K3=1, K4=0, K5=0
       - Hold time: 15s
       - Seq: 0 (Hello unreliable)

T=0.1s: R2 nhận Hello từ R1
       - Kiểm tra AS match? ✓
       - Kiểm tra K-values match? ✓
       - Thêm R1 vào Neighbor Table
       - Gửi Hello lại (unicast) + Update (full topology)

T=0.2s: R1 nhận Hello từ R2
       - Thêm R2 vào Neighbor Table
       - Gửi ACK cho Update
       - Gửi Update (full topology) → R2

T=0.3s: R2 gửi ACK cho Update từ R1

T=0.5s: Cả hai đã trao đổi topology
       - DUAL chạy → tính Successor, FS
       - Routes vào Routing Table
       - Adjacency UP!

Sau đó: Hello mỗi 5s để duy trì
```

### Các vấn đề thường gặp khi thiết lập neighbor

```
Lỗi 1: K-values mismatch
%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.1.1.2 (GigabitEthernet0/0) is down: K-value mismatch

Fix: Đảm bảo cùng K-values trên cả 2 đầu
router eigrp 100
 metric weights 0 1 0 1 0 0    ! K1=1 K2=0 K3=1 K4=0 K5=0

Lỗi 2: AS number mismatch
→ Không có log rõ ràng, neighbor đơn giản không form
→ Check: show ip eigrp interfaces (AS number)

Lỗi 3: Subnet mismatch
→ Interface IP phải cùng subnet
→ Exception: EIGRP sẽ form neighbor với /32 routes trên point-to-point

Lỗi 4: Authentication failure
%EIGRP-DUAL-5-NBRCHANGE: Neighbor down: authentication failure
```

### Authentication

```cisco
! MD5 Authentication (classic mode)
key chain EIGRP_KEYS
 key 1
  key-string MySecretKey123
  accept-lifetime 00:00:00 Jun 1 2026 infinite
  send-lifetime 00:00:00 Jun 1 2026 infinite

interface GigabitEthernet0/0
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP_KEYS

! SHA-256 Authentication (named mode)
router eigrp NAMED
 address-family ipv4 unicast autonomous-system 100
  af-interface GigabitEthernet0/0
   authentication mode hmac-sha-256 MySecretKey
```

### Trong thực tế

**Stub routing — Giảm Query scope:**
```cisco
! Chi nhánh (spoke) cấu hình stub
router eigrp 100
 eigrp stub connected summary
 ! connected: chỉ quảng bá directly connected networks
 ! summary: chỉ quảng bá summary routes
 ! Kết quả: Hub KHÔNG gửi Query đến stub → giảm SIA risk

! Options: connected, summary, static, redistributed, receive-only
```

**Tại sao Stub quan trọng?**
- Mạng 500 chi nhánh, mỗi chi nhánh 1 WAN link
- Khi 1 route mất, hub gửi Query đến TẤT CẢ 500 chi nhánh
- Nếu 1 chi nhánh WAN chậm → SIA → neighbor reset → cascade failure
- Với stub: hub biết chi nhánh không có alternate path → KHÔNG query

### Trong AWS

Khi chạy virtual EIGRP router (CSR1000v, Catalyst 8000v) trên EC2:
```
Lưu ý:
1. Security Group phải allow Protocol 88 (EIGRP)
2. Source/Dest Check phải DISABLE trên ENI (vì router forward traffic)
3. Multicast: VPC không hỗ trợ native → dùng EIGRP unicast neighbor
   neighbor 10.0.1.5 GigabitEthernet1
4. HA: Dùng EIGRP + floating static route đến ENI failover
```

---

## 7. Unequal-Cost Load Balancing — Tận dụng mọi đường

### Mini-example: Phân chia hàng hóa theo sức chở

Bạn có 2 xe tải giao hàng:
- **Xe A**: Chở được 10 tấn, đi mất 1 giờ
- **Xe B**: Chở được 5 tấn, đi mất 2 giờ

Nếu chỉ dùng xe A (equal-cost logic) → xe B đứng không, lãng phí.
Nếu dùng cả 2 nhưng **chia theo tỷ lệ** → A chở 2/3, B chở 1/3 → tối ưu!

EIGRP làm đúng điều này với **variance command** — load balance trên các đường có metric khác nhau!

### Variance Command — Chìa khóa Unequal-Cost LB

```
Cú pháp:
router eigrp 100
 variance <multiplier>    ! Giá trị 1-128, mặc định = 1

Nguyên tắc:
- Route được thêm vào load balancing NẾU:
  1. FD(route) ≤ variance × FD(successor)     [metric condition]
  2. Route phải là Feasible Successor          [loop-free condition]
  
Cả HAI điều kiện phải thỏa mãn!
```

### Ví dụ chi tiết

```
Network 192.168.1.0/24 — Nhìn từ Router R1:

Path A (qua R2): FD = 30,000 → SUCCESSOR
Path B (qua R3): FD = 45,000, RD = 25,000
Path C (qua R4): FD = 90,000, RD = 28,000
Path D (qua R5): FD = 100,000, RD = 35,000

Current FD = 30,000 (best path)

Bước 1: Kiểm tra Feasibility Condition (FC)
- Path B: RD = 25,000 < FD = 30,000? ✓ → Feasible Successor
- Path C: RD = 28,000 < FD = 30,000? ✓ → Feasible Successor  
- Path D: RD = 35,000 < FD = 30,000? ✗ → NOT Feasible Successor

Bước 2: Áp dụng variance
  variance 2 → Threshold = 2 × 30,000 = 60,000
  - Path B: FD = 45,000 ≤ 60,000? ✓ VÀ là FS? ✓ → LOAD BALANCE!
  - Path C: FD = 90,000 ≤ 60,000? ✗ → Không đưa vào LB
  
  variance 3 → Threshold = 3 × 30,000 = 90,000
  - Path B: FD = 45,000 ≤ 90,000? ✓ VÀ là FS? ✓ → LOAD BALANCE!
  - Path C: FD = 90,000 ≤ 90,000? ✓ VÀ là FS? ✓ → LOAD BALANCE!
  - Path D: FD = 100,000 ≤ 90,000? ✗ (và không phải FS) → Không

Kết quả với variance 3:
- Traffic chia theo tỷ lệ NGHỊCH metric:
  Path A: 30,000 → 6/11 traffic (≈55%)
  Path B: 45,000 → 4/11 traffic (≈36%)
  Path C: 90,000 → 2/11 traffic (≈9%)
```

### Traffic Share

```cisco
! Mặc định: chia tỷ lệ theo metric
router eigrp 100
 variance 3
 traffic-share balanced    ! Mặc định — proportional sharing

! Hoặc chỉ install routes nhưng dùng đường tốt nhất
 traffic-share min across-interfaces    ! Install tất cả nhưng chỉ dùng best

! Giới hạn số đường
 maximum-paths 4    ! Mặc định = 4, max = 32
```

### CEF và Load Balancing

```
EIGRP chỉ quyết định ROUTES nào vào routing table.
CEF (Cisco Express Forwarding) thực hiện actual load balancing:

Per-Destination (mặc định):
- Mỗi source-dest pair đi 1 đường cố định
- Tránh out-of-order packets
- Tốt cho hầu hết traffic

Per-Packet:
- Mỗi packet đi đường khác nhau (round-robin)
- Chia đều hơn nhưng có thể gây reordering
- Dùng khi cần utilize tất cả links equally

ip cef load-sharing algorithm universal    ! Per-destination (default)
ip cef load-sharing algorithm tunnel <id>  ! Per-packet trên tunnel
```

### Trong thực tế

**Use case: Dual WAN với EIGRP**
```
Head Office
├── WAN Link 1: MPLS 100Mbps (metric thấp)
├── WAN Link 2: Internet VPN 50Mbps (metric cao hơn)
└── WAN Link 3: 4G Backup 20Mbps (metric cao nhất)

Cấu hình:
router eigrp 100
 variance 3
 maximum-paths 3
 
Kết quả:
- Bình thường: Traffic chia ~60%/30%/10% trên 3 links
- Link 1 down: Traffic chia ~75%/25% trên 2 links còn lại
- Chuyển đổi: Sub-second (Feasible Successor available)

Lợi ích:
- Tận dụng 100% bandwidth đã trả tiền
- Failover nhanh không downtime
- Không cần manual intervention
```

**Lưu ý quan trọng:**
```
⚠️ EIGRP unequal-cost LB CHỈ hoạt động khi:
1. Routes phải là Feasible Successor (thỏa FC)
2. Variance phải đủ lớn để include route
3. maximum-paths phải cho phép (mặc định 4)
4. Cả 3 điều kiện cùng thỏa!

Sai lầm phổ biến:
- Set variance 128 nhưng route không phải FS → KHÔNG có LB
- Route thỏa FC nhưng variance quá nhỏ → KHÔNG include
- Quên kiểm tra "show ip route" sau khi configure
```

### Trong AWS

AWS không có EIGRP LB tương đương trực tiếp, nhưng có concepts tương tự:

```
AWS Equal-Cost Multi-Path:
- Route Table: Có thể có nhiều routes cùng destination
- NLB: Distributes across multiple AZs
- Transit Gateway: Equal-cost routing across attachments

Tương đương EIGRP Unequal-Cost LB trong AWS:
- Weighted Target Groups trong ALB/NLB
- Route 53 Weighted Routing Policy
- Global Accelerator endpoint weights
```

---

## 8. EIGRP Summarization và Filtering — Tối ưu bảng định tuyến

### Mini-example: Địa chỉ tóm tắt

Thay vì nói:
- "Nhà 1, đường Nguyễn Trãi"
- "Nhà 2, đường Nguyễn Trãi"
- "Nhà 3, đường Nguyễn Trãi"
- ...100 nhà...

Bạn chỉ cần nói: **"Đường Nguyễn Trãi"** — tất cả nhà trên đó đều đến đường đó.

EIGRP summarization làm tương tự — gộp nhiều routes thành 1 summary route.

### Auto-Summary vs Manual Summary

```cisco
! Auto-Summary (MẶC ĐỊNH TẮT từ IOS 15.0)
router eigrp 100
 auto-summary    ! Summarize tại classful boundary
 ! 10.1.1.0/24, 10.1.2.0/24, 10.1.3.0/24 → 10.0.0.0/8

! Vấn đề: Classful boundary quá rộng!
! Nếu có 10.1.0.0/16 ở site A và 10.2.0.0/16 ở site B
! Cả 2 đều quảng bá 10.0.0.0/8 → BLACKHOLE!

! Manual Summary (RECOMMENDED)
interface GigabitEthernet0/0
 ip summary-address eigrp 100 10.1.0.0 255.255.252.0
 ! Gộp 10.1.0.0/24, 10.1.1.0/24, 10.1.2.0/24, 10.1.3.0/24
 ! thành 10.1.0.0/22 qua interface này

! Named Mode
router eigrp NAMED
 address-family ipv4 unicast autonomous-system 100
  af-interface GigabitEthernet0/0
   summary-address 10.1.0.0/22
```

### Leak Map — Quảng bá specific route qua summary

```cisco
! Vấn đề: Summary 10.1.0.0/22 ẩn hết chi tiết
! Nhưng cần route 10.1.1.0/24 vẫn visible (ví dụ: server farm)

! Tạo ACL/prefix-list cho routes cần leak
ip prefix-list LEAK permit 10.1.1.0/24

! Tạo route-map
route-map LEAK_MAP permit 10
 match ip address prefix-list LEAK

! Áp dụng leak-map
interface GigabitEthernet0/0
 ip summary-address eigrp 100 10.1.0.0 255.255.252.0 leak-map LEAK_MAP
 ! Kết quả: Quảng bá CẢ summary 10.1.0.0/22 VÀ specific 10.1.1.0/24
```

### Route Filtering

```cisco
! Distribute-list: Filter routes IN hoặc OUT
router eigrp 100
 distribute-list prefix NO_DEFAULT in GigabitEthernet0/0
 ! Chặn default route từ interface cụ thể

ip prefix-list NO_DEFAULT deny 0.0.0.0/0
ip prefix-list NO_DEFAULT permit 0.0.0.0/0 le 32

! Offset-list: Tăng metric của routes cụ thể
router eigrp 100
 offset-list 10 in 10000 GigabitEthernet0/1
 ! Tăng metric thêm 10000 cho routes match ACL 10 nhận từ Gi0/1
 ! Dùng để ưu tiên path khác mà không block hoàn toàn

access-list 10 permit 192.168.1.0 0.0.0.255
```

### Trong thực tế

**Thiết kế EIGRP Summarization cho mạng 3 tầng:**
```
                    Core
                   /    \
              Distribution   Distribution
              /     |    \   /    |     \
           Access Access Access Access Access Access

Summarization boundaries:
- Access → Distribution: /24 individual networks
- Distribution → Core: /16 or /14 summaries per building/floor

Lợi ích:
- Core routing table nhỏ (hàng chục entries thay vì hàng ngàn)
- Topology change ở Access KHÔNG ảnh hưởng Core
- DUAL query scope giảm (không propagate qua summary boundary)
```

### Trong AWS

```
Tương đương trong AWS:
- EIGRP Summary ≈ CIDR Supernetting trong VPC
- Route Filtering ≈ Route Table associations + Prefix lists
- Specific routes ≈ More-specific routes trong Route Table

Ví dụ:
VPC CIDR: 10.0.0.0/16
Subnet A: 10.0.1.0/24 → Route to NAT GW
Subnet B: 10.0.2.0/24 → Route to IGW

Khi advertise qua Direct Connect BGP:
- Summary: 10.0.0.0/16 (aggregate)
- Hoặc specific: 10.0.1.0/24, 10.0.2.0/24
```

---

## 9. Tình huống thực tế — EIGRP trong môi trường production

### Tình huống 1: Mạng gia đình/SOHO — EIGRP không cần thiết

```
Thực tế: Mạng nhà thường chỉ có 1 router (ISP modem/router combo)
→ Không có routing protocol nào chạy
→ Static default route: 0.0.0.0/0 → ISP gateway

Khi nào cần routing protocol ở nhà?
- Home lab với nhiều routers (học tập)
- Network có 2+ WAN connections (load balance)
- Kết nối nhiều VLAN qua inter-VLAN routing
```

### Tình huống 2: Doanh nghiệp vừa — EIGRP Hub-and-Spoke

```
Công ty với 1 trụ sở chính + 20 chi nhánh:

Head Office (Hub):
┌─────────────────────────────┐
│ router eigrp 100            │
│  network 10.0.0.0 0.255.255.255│
│  variance 2                 │
│  maximum-paths 4            │
│  no auto-summary            │
└─────────────────────────────┘

Branch Office (Spoke):
┌─────────────────────────────┐
│ router eigrp 100            │
│  network 10.10.X.0 0.0.0.255│
│  eigrp stub connected summary│
│  no auto-summary            │
└─────────────────────────────┘

WAN Design:
- Primary: MPLS (bandwidth 10Mbps, delay 10ms)
- Backup: Internet VPN (bandwidth 5Mbps, delay 50ms)

Kết quả:
- Primary path: Metric qua MPLS (thấp hơn do BW cao, delay thấp)
- Backup path: Metric qua VPN (cao hơn) → Feasible Successor
- Failover: Sub-second khi MPLS link down
- Stub: Hub không query chi nhánh → ổn định

Metric tuning:
interface Tunnel0  (VPN)
 bandwidth 5000
 delay 5000        ! Tăng delay để MPLS luôn preferred
```

### Tình huống 3: ISP/Large Enterprise — EIGRP scalability

```
Mạng 1000+ routers:

Vấn đề scaling:
1. Query scope: 1 route mất → Query lan toàn mạng → SIA
2. Topology table: Quá lớn, tốn memory
3. DUAL computation: CPU intensive khi nhiều routes

Giải pháp:

1. Stub Routing (giảm query scope)
   - 80% routers là stub (chi nhánh, access layer)
   - Chỉ 20% routers participate in DUAL queries

2. Summarization (giảm topology size)
   - Summarize tại distribution layer
   - Core chỉ thấy /16 summaries
   - Query DỪNG tại summary boundary

3. Multiple EIGRP Processes
   router eigrp 100    ! Campus
   router eigrp 200    ! WAN
   Redistribute between them (controlled)

4. EIGRP → OSPF Migration (khi quá lớn)
   - OSPF area design scale tốt hơn
   - Nhưng mất unequal-cost LB
   - Common: EIGRP campus + OSPF/BGP WAN
```

### Tình huống 4: Hybrid Cloud — EIGRP meets AWS

```
On-Premises EIGRP ←→ AWS BGP

Thiết kế:
┌─────────────────┐     ┌──────────────────┐
│  On-Premises    │     │      AWS         │
│  EIGRP AS 100   │     │                  │
│                 │     │  VPC 10.100.0.0/16│
│  10.0.0.0/8    │─DX──│                  │
│                 │     │  TGW             │
│  CSR1000v      │─VPN─│                  │
│  (redistribution)│    │  BGP AS 64512   │
└─────────────────┘     └──────────────────┘

Trên CSR1000v (border router):
router eigrp 100
 redistribute bgp 65000 metric 100000 10 255 1 1500
 ! Inject AWS routes vào EIGRP domain

router bgp 65000
 redistribute eigrp 100 route-map EIGRP_TO_BGP
 neighbor 169.254.X.X remote-as 64512    ! AWS DX peer

route-map EIGRP_TO_BGP permit 10
 match ip address prefix-list ON_PREM_ROUTES
 set as-path prepend 65000    ! Optional: influence return path

Lưu ý:
- Seed metric BẮT BUỘC khi redistribute vào EIGRP
- Tag routes để tránh redistribution loops
- Filter carefully: không leak AWS routes ra Internet
```

---

## 10. Bài tập thực hành

### Bài tập 1: Cấu hình EIGRP cơ bản và verify

```cisco
! Topology: R1 ──── R2 ──── R3
! R1: Gi0/0 = 10.12.0.1/24, Lo0 = 1.1.1.1/32
! R2: Gi0/0 = 10.12.0.2/24, Gi0/1 = 10.23.0.2/24, Lo0 = 2.2.2.2/32
! R3: Gi0/1 = 10.23.0.3/24, Lo0 = 3.3.3.3/32

! R1 Configuration:
router eigrp 100
 network 10.12.0.0 0.0.0.255
 network 1.1.1.1 0.0.0.0
 no auto-summary

! Verification commands:
show ip eigrp neighbors          ! Kiểm tra neighbor đã form
show ip eigrp topology           ! Xem tất cả routes + FD/RD
show ip route eigrp              ! Routes trong routing table
show ip eigrp interfaces         ! Interfaces tham gia EIGRP
show ip eigrp traffic            ! Packet counters

! Câu hỏi tự kiểm tra:
! 1. FD đến 3.3.3.3/32 từ R1 là bao nhiêu?
! 2. R2 có phải Successor cho 3.3.3.3/32 không?
! 3. Có Feasible Successor nào không? Tại sao?
```

### Bài tập 2: Unequal-Cost Load Balancing

```cisco
! Topology:
!        10Mbps          100Mbps
! R1 ─────────── R2 ─────────── R4
!  │                              │
!  └──── 50Mbps ──── R3 ── 50Mbps┘
!
! Destination: 4.4.4.4/32 (Lo0 trên R4)

! Bước 1: Verify initial state
R1# show ip eigrp topology 4.4.4.4/32
! Xem FD và RD cho mỗi path

! Bước 2: Tính variance cần thiết
! FD via R2 = X (giả sử 30720)
! FD via R3 = Y (giả sử 56320)
! Variance cần: Y/X = 56320/30720 = 1.83 → variance 2

! Bước 3: Configure
router eigrp 100
 variance 2
 maximum-paths 4

! Bước 4: Verify
show ip route 4.4.4.4
! Phải thấy 2 next-hops với traffic share ratio

show ip cef 4.4.4.4
! Xem CEF load-sharing buckets

! Bài tập nâng cao:
! - Tính traffic share ratio chính xác
! - Thử variance 1 (chỉ equal-cost) rồi so sánh
! - Thêm "traffic-share min" và quan sát khác biệt
```

### Bài tập 3: Summarization và Query Scope

```cisco
! Topology: Hub ──── Spoke1 (10.1.0.0/24, 10.1.1.0/24)
!            │
!            ├──── Spoke2 (10.2.0.0/24, 10.2.1.0/24)
!            │
!            └──── Spoke3 (10.3.0.0/24, 10.3.1.0/24)

! Bước 1: Cấu hình summarization trên Spokes
! Spoke1:
interface GigabitEthernet0/0
 ip summary-address eigrp 100 10.1.0.0 255.255.254.0

! Bước 2: Cấu hình stub
router eigrp 100
 eigrp stub connected summary

! Bước 3: Verify trên Hub
show ip eigrp topology
! Phải thấy summary routes thay vì individual /24s

! Bước 4: Test query scope
! Shutdown Lo0 trên Spoke1
! Trên Hub: debug eigrp packets query
! Observe: Query KHÔNG gửi đến Spoke2, Spoke3 (vì stub)

! Câu hỏi:
! 1. Null0 route xuất hiện ở đâu? Tại sao?
! 2. Nếu 10.1.0.0/24 down, summary 10.1.0.0/23 có bị withdraw?
! 3. Khi nào cần leak-map?
```

### Bài tập 4: Authentication và Security

```cisco
! Cấu hình MD5 authentication giữa R1 và R2

! R1:
key chain EIGRP_AUTH
 key 1
  key-string Str0ngP@ss!
  
interface GigabitEthernet0/0
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP_AUTH

! R2: (same config)

! Verify:
show ip eigrp neighbors    ! Neighbor phải UP
show key chain EIGRP_AUTH  ! Verify key active

! Test: Thay đổi key trên 1 bên
! Observe: Neighbor drops sau Hold timer expires
! Fix: Match keys trên cả 2 bên

! Key rotation (zero-downtime):
! 1. Add key 2 với send-lifetime tương lai
! 2. Cả 2 bên accept cả key 1 và key 2
! 3. Khi key 2 active, remove key 1
```

### Bài tập 5: Troubleshooting EIGRP

```cisco
! Tình huống: R1 và R2 không form neighbor. Debug!

! Checklist:
! 1. Layer 1/2 OK?
show interface GigabitEthernet0/0    ! up/up?
ping 10.12.0.2                        ! L3 reachable?

! 2. EIGRP enabled on interface?
show ip eigrp interfaces              ! Interface listed?
show run | section router eigrp       ! Network statement correct?

! 3. K-values match?
show ip protocols | include K        

! 4. AS number match?
show ip protocols | include AS

! 5. ACL blocking EIGRP?
show ip interface GigabitEthernet0/0 | include access
! EIGRP dùng Protocol 88 — ACL phải allow!

! 6. Authentication?
debug eigrp packets                   ! Look for auth failures

! 7. Passive interface?
show ip protocols | include Passive   ! Interface passive = no Hello sent!

! Common fixes:
no passive-interface GigabitEthernet0/0
no distribute-list in GigabitEthernet0/0
ip authentication key-chain eigrp 100 CORRECT_KEYCHAIN
```

---

## 11. Tóm tắt và Tài liệu tham khảo

### Tóm tắt kiến thức

```
┌─────────────────────────────────────────────────────────────┐
│                    EIGRP SUMMARY                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ▸ EIGRP = Advanced Distance Vector, dùng DUAL algorithm    │
│ ▸ DUAL đảm bảo loop-free bằng Feasibility Condition       │
│ ▸ Feasibility Condition: RD(neighbor) < Current FD         │
│                                                             │
│ ▸ Successor = Best path (trong Routing Table)              │
│ ▸ Feasible Successor = Backup loop-free (Topology Table)   │
│ ▸ Có FS → sub-second failover                             │
│ ▸ Không FS → DUAL goes ACTIVE → Query → chậm hơn         │
│                                                             │
│ ▸ Metric = 256 × (BW + Delay) [mặc định K1=K3=1]         │
│ ▸ BW = 10^7 / min_bandwidth_kbps                          │
│ ▸ Delay = sum_delay / 10                                   │
│                                                             │
│ ▸ Unequal-Cost LB: variance × FD(successor) ≥ FD(route)   │
│   + Route PHẢI là Feasible Successor                       │
│                                                             │
│ ▸ 5 Packet types: Hello, Update, Query, Reply, ACK        │
│ ▸ Reliable Transport Protocol (RTP) cho Update/Query/Reply │
│                                                             │
│ ▸ Stub routing: Giảm query scope, tránh SIA               │
│ ▸ Summarization: Giảm table size, giới hạn query scope    │
│                                                             │
│ ▸ Protocol 88, Multicast 224.0.0.10                        │
│ ▸ AD: Internal = 90, External = 170                        │
└─────────────────────────────────────────────────────────────┘
```

### Key Takeaways

1. **EIGRP convergence nhanh vì pre-compute backup** — Feasible Successor sẵn sàng ngay
2. **Feasibility Condition đơn giản nhưng powerful** — chỉ 1 phép so sánh đảm bảo loop-free
3. **Unequal-cost LB là unique feature** — không protocol nào khác làm được dễ dàng như vậy
4. **Stub routing CRITICAL cho scalability** — phải configure trên mọi spoke/access router
5. **Metric tuning dùng delay** — KHÔNG bao giờ thay đổi bandwidth trên interface (ảnh hưởng QoS)

### Tài liệu tham khảo

1. **RFC 7868** — "Cisco's Enhanced Interior Gateway Routing Protocol (EIGRP)" — IETF Informational RFC, tài liệu chính thức mô tả EIGRP protocol
   - https://www.rfc-editor.org/rfc/rfc7868

2. **Cisco Documentation** — "Understand and Use the Enhanced Interior Gateway Routing Protocol"
   - https://www.cisco.com/c/en/us/support/docs/ip/enhanced-interior-gateway-routing-protocol-eigrp/16406-eigrp-toc.html

3. **Cisco Documentation** — "Introduction to EIGRP"
   - https://www.cisco.com/c/en/us/support/docs/ip/enhanced-interior-gateway-routing-protocol-eigrp/13669-1.html

4. **Cisco Documentation** — "How Does Unequal Cost Path Load Balancing (Variance) Work in IGRP and EIGRP"
   - https://www.cisco.com/c/en/us/support/docs/ip/enhanced-interior-gateway-routing-protocol-eigrp/13677-19.html

5. **Garcia-Luna-Aceves, J.J.** — "Loop-Free Routing Using Diffusing Computations" — IEEE/ACM Transactions on Networking, 1993 — Paper gốc mô tả DUAL algorithm

6. **Cisco Press** — "EIGRP for IP: Basic Operation and Configuration" — CCIE Routing and Switching study material

7. **Cisco Press** — "Routing TCP/IP, Volume I" by Jeff Doyle — Chapter on EIGRP covering metric calculation, DUAL operation, and design best practices

---

*Bài viết tiếp theo: [IS-IS Protocol](/2026/07/01/is-is-protocol) — Link-State routing protocol phổ biến trong ISP*
