---
layout: post
title: "BGP Deep Dive - Path-Vector Protocol, AS, eBGP/iBGP, Attributes và Route Selection"
date: 2026-06-01
categories: [networking]
tags: [bgp, routing, internet, autonomous-system, path-selection]
---

## 1. Giới thiệu — Hệ thống hàng không quốc tế

Hãy tưởng tượng bạn muốn bay từ **Hà Nội đến New York**. Không có chuyến bay thẳng, nên bạn phải:

1. **Chọn hãng/liên minh** — Vietnam Airlines (Star Alliance), hoặc Qatar Airways (oneworld), hoặc Emirates (riêng)
2. **Mỗi hãng có chính sách riêng** — giá khác, hành lý khác, transit khác
3. **Route qua nhiều hub** — HN → Tokyo → LA → NY, hoặc HN → Dubai → London → NY
4. **Mỗi hub thuộc "vùng lãnh thổ" khác** — phải tuân thủ luật của từng vùng
5. **Không phải đường ngắn nhất luôn tốt nhất** — giá rẻ hơn, hạng thương gia, thời gian transit...

**BGP (Border Gateway Protocol)** hoạt động y hệt hệ thống hàng không quốc tế cho Internet:
- Mỗi "hãng bay" = **Autonomous System (AS)** — một tổ chức/ISP
- Mỗi "tuyến bay" = **BGP route** — đường đi đến một mạng
- "Chính sách hãng" = **BGP Policy** — quyết định accept/prefer/reject routes
- "Hub transit" = **Transit AS** — ISP trung gian
- "Không phải ngắn nhất" = **Policy-based** — business decision, không chỉ technical

### Tại sao BGP là protocol quan trọng nhất Internet?

| Fact | Chi tiết |
|---|---|
| Vai trò | **Keo dán Internet** — kết nối 70,000+ mạng thành 1 Internet |
| Full table | ~950,000 IPv4 routes + ~200,000 IPv6 routes (2024) |
| Không có thay thế | Không có protocol nào khác làm được việc BGP làm |
| Business protocol | Quyết định traffic flow dựa trên **business policy**, không chỉ metric |
| AWS relevance | Direct Connect, Transit Gateway, VPN — tất cả dùng BGP |

### Tại sao bạn cần hiểu BGP?

- **Cloud Engineer**: AWS Direct Connect, GCP Interconnect, Azure ExpressRoute — TẤT CẢ dùng BGP
- **Network Engineer**: Quản lý peering, transit, multihoming
- **DevOps/SRE**: Hiểu Internet routing giúp debug connectivity issues
- **Security**: BGP hijacking, route leaks — threats nghiêm trọng

---

## 2. BGP là gì? — Giải thích cho người không biết IT

### Phép so sánh đời thường: Mạng lưới giao thương quốc tế

Tưởng tượng thế giới thương mại:
- Mỗi **quốc gia** (Autonomous System) tự quản lý giao thông nội bộ (IGP)
- Giữa các quốc gia, có **hiệp định thương mại** (BGP peering):
  - Hiệp định **ngang hàng** (peering): Trao đổi hàng hóa free (settlement-free)
  - Hiệp định **mua bán** (transit): Trả tiền để gửi hàng qua nước khác
- Mỗi nước **tự quyết định** chấp nhận hàng từ đâu, gửi hàng đi đâu (BGP policy)
- Hàng hóa đi qua nhiều nước trung gian, mỗi nước **ghi tên mình lên** (AS-PATH)

### Định nghĩa kỹ thuật

**BGP (Border Gateway Protocol)** là:
- **Path-Vector** routing protocol (không phải Distance-Vector, không phải Link-State)
- **EGP** (Exterior Gateway Protocol) — routing GIỮA các Autonomous Systems
- **Policy-based** — quyết định dựa trên chính sách, không chỉ metric kỹ thuật
- **Incremental updates** — chỉ gửi thay đổi, không periodic full update
- **TCP port 179** — reliable delivery, manual neighbor configuration

**Đặc điểm chính:**
```
Protocol: TCP port 179
Type: Path-Vector (hybrid: carries full AS-PATH)
Updates: Triggered only (incremental)
Timers: Keepalive 60s, Hold 180s
AD: eBGP = 20, iBGP = 200
Current version: BGP-4 (RFC 4271)
Full Internet table: ~950K IPv4 + ~200K IPv6 prefixes
```

### Autonomous System (AS)

```
Autonomous System = Tập hợp networks dưới SỰ QUẢN LÝ DUY NHẤT
                    chia sẻ CHUNG MỘT chính sách routing

Ví dụ:
- AS 7552 = Viettel (Vietnam)
- AS 45899 = VNPT (Vietnam)
- AS 15169 = Google
- AS 16509 = Amazon (AWS)
- AS 32934 = Facebook/Meta
- AS 13335 = Cloudflare

AS Number (ASN):
- 2-byte: 1 - 65535 (cũ)
  - Private: 64512 - 65534
- 4-byte: 1 - 4,294,967,295 (mới, RFC 6793)
  - Private: 4200000000 - 4294967294
  - Notation: "asdot" (1.5) hoặc "asplain" (65541)

Ai cấp ASN?
- IANA → Regional Internet Registries (RIR):
  - ARIN (North America)
  - RIPE NCC (Europe, Middle East)
  - APNIC (Asia Pacific) — quản lý Vietnam
  - LACNIC (Latin America)
  - AFRINIC (Africa)
```

### eBGP vs iBGP

```
eBGP (External BGP):
- Giữa routers KHÁC AS number
- TTL = 1 (mặc định — directly connected)
- Next-hop thay đổi khi route đi qua
- AD = 20 (rất trusted — thấp hơn IGP)
- Dùng cho: Peering, Transit, Multihoming

iBGP (Internal BGP):
- Giữa routers CÙNG AS number
- TTL = 255 (không cần directly connected)
- Next-hop KHÔNG thay đổi (giữ nguyên eBGP next-hop)
- AD = 200 (ít trusted hơn IGP — IGP phải resolve next-hop)
- Dùng cho: Phân phối BGP routes TRONG AS

Quy tắc Split Horizon trong iBGP:
⚠️ Route nhận từ iBGP peer KHÔNG được advertise cho iBGP peer khác!
→ Giải pháp: Full-mesh iBGP, Route Reflector, hoặc Confederation
```

---

## 3. BGP Path Attributes — Thuộc tính quyết định đường đi

### Mini-example: Chọn nhà cung cấp

Bạn là chủ quán coffee, cần chọn nhà cung cấp cà phê:

| Thuộc tính | Ví dụ coffee | BGP Attribute |
|---|---|---|
| Ưu tiên nội bộ | "Luôn mua từ anh Tùng trước" | Weight (Cisco) |
| Chính sách công ty | "Ưu tiên hàng Việt Nam" | Local Preference |
| Chuỗi cung ứng ngắn | "Ít trung gian nhất" | AS-PATH length |
| Nguồn gốc | "Nông trại trực tiếp > đại lý" | Origin |
| Giá đề xuất | "Giá FOB từ nhà cung cấp" | MED |
| Giao hàng nhanh | "Kho gần nhất" | IGP metric to next-hop |

### Phân loại Path Attributes

```
┌─────────────────────────────────────────────────────────────────┐
│                  BGP Path Attribute Categories                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Well-Known Mandatory (bắt buộc có trong mọi UPDATE):            │
│  - ORIGIN: Nguồn gốc route (IGP=0, EGP=1, Incomplete=2)        │
│  - AS_PATH: Danh sách AS đã đi qua                              │
│  - NEXT_HOP: IP address của next-hop                             │
│                                                                   │
│ Well-Known Discretionary (phải hiểu, không bắt buộc có):        │
│  - LOCAL_PREF: Ưu tiên trong AS (default 100)                   │
│  - ATOMIC_AGGREGATE: Route đã bị aggregated                     │
│                                                                   │
│ Optional Transitive (optional, forward nếu không hiểu):         │
│  - AGGREGATOR: AS + Router ID đã aggregate                      │
│  - COMMUNITY: Tags cho policy (RFC 1997)                        │
│  - EXTENDED COMMUNITY: RT, SOO cho VPN                          │
│  - LARGE COMMUNITY: RFC 8092 (4-byte ASN support)               │
│                                                                   │
│ Optional Non-Transitive (optional, drop nếu không hiểu):        │
│  - MED (Multi-Exit Discriminator): Hint cho neighbor            │
│  - ORIGINATOR_ID: Route Reflector loop prevention               │
│  - CLUSTER_LIST: Route Reflector cluster info                   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Chi tiết từng Attribute quan trọng

**1. ORIGIN — Nguồn gốc route:**
```
Giá trị:
- IGP (i) = 0: Route originated via "network" command hoặc redistribute IGP
- EGP (e) = 1: Route learned từ EGP (legacy, hiếm gặp)
- Incomplete (?) = 2: Route redistributed từ source không rõ

Preference: IGP > EGP > Incomplete
Ví dụ: Route vào BGP bằng redistribute → Origin = ?
        Route vào BGP bằng network statement → Origin = i
```

**2. AS_PATH — Lịch sử đi qua các AS:**
```
Là DANH SÁCH AS đã transit route:
Route từ Google (AS 15169) → Telia (AS 1299) → Viettel (AS 7552):
AS_PATH = 1299 15169
(Đọc từ phải sang trái = route originated từ 15169, qua 1299)

Các loại AS_PATH segment:
- AS_SEQUENCE: Danh sách ordered (bình thường)
- AS_SET: Unordered set (khi aggregate)
- AS_CONFED_SEQUENCE: Confederation internal
- AS_CONFED_SET: Confederation set

Loop Prevention:
- Router nhận route có AS number CỦA MÌNH trong AS_PATH → REJECT!
- Đây là cơ chế loop prevention CƠ BẢN và DUY NHẤT của BGP

AS_PATH Prepending (traffic engineering):
- Thêm AS number của mình nhiều lần vào AS_PATH
- Làm path dài hơn → ít preferred
- Ví dụ: AS 65000 prepend 3 lần → AS_PATH = 65000 65000 65000 65000
```

**3. NEXT_HOP — Bước nhảy tiếp theo:**
```
eBGP behavior:
- Next-hop = IP của eBGP peer
- Thay đổi khi route cross AS boundary

iBGP behavior:
- Next-hop KHÔNG thay đổi (giữ nguyên eBGP next-hop)
- iBGP peer phải resolve next-hop qua IGP!

Vấn đề phổ biến:
R1 (AS 100) ─── eBGP ─── R2 (AS 200) ─── iBGP ─── R3 (AS 200)
IP: 10.1.1.1                 10.1.1.2/10.2.2.2        10.2.2.3

Route từ AS 100: next-hop = 10.1.1.1
R2 nhận → giữ next-hop = 10.1.1.1 → advertise iBGP cho R3
R3 nhận: next-hop = 10.1.1.1 → R3 KHÔNG biết route đến 10.1.1.1!
→ Route INVALID!

Giải pháp:
- next-hop-self: R2 thay đổi next-hop thành IP của R2
  neighbor 10.2.2.3 next-hop-self
- Hoặc: IGP advertise external link subnet (10.1.1.0/24)
```

**4. LOCAL_PREF — Ưu tiên local (trong AS):**
```
- Chỉ trao đổi TRONG iBGP (không gửi ra eBGP)
- Default = 100
- HIGHER = MORE preferred
- Dùng để chọn exit point từ AS

Ví dụ: AS 65000 có 2 uplinks
- Uplink 1 (ISP-A): Set LOCAL_PREF = 200 (preferred)
- Uplink 2 (ISP-B): Set LOCAL_PREF = 100 (backup)
→ Tất cả routers trong AS 65000 prefer exit qua ISP-A

route-map PREFER_ISP_A permit 10
 set local-preference 200

router bgp 65000
 neighbor 10.1.1.1 route-map PREFER_ISP_A in
```

**5. MED (Multi-Exit Discriminator) — Gợi ý cho neighbor:**
```
- Gửi cho eBGP neighbor (hint: "vào AS tôi qua đường nào tốt hơn")
- Optional Non-Transitive (KHÔNG forward qua AS thứ 3)
- LOWER = MORE preferred
- Default = 0 (hoặc IGP metric)
- Chỉ so sánh giữa routes từ CÙNG neighboring AS (mặc định)

Ví dụ: AS 65000 muốn traffic từ AS 65001 vào qua link East
- Link East: Advertise MED = 50
- Link West: Advertise MED = 200
→ AS 65001 prefer đường vào AS 65000 qua East (MED thấp hơn)

⚠️ Lưu ý: MED chỉ là "suggestion" — neighbor có thể ignore!
   (LOCAL_PREF của neighbor override MED)
```

**6. COMMUNITY — Tags cho policy:**
```
Format: ASN:Value (e.g., 65000:100)

Well-Known Communities:
- NO_EXPORT (0xFFFFFF01): Không advertise ra ngoài AS
- NO_ADVERTISE (0xFFFFFF02): Không advertise cho BẤT KỲ peer nào
- NO_EXPORT_SUBCONFED (0xFFFFFF03): Không ra ngoài confederation
- NOPEER (0xFFFFFF04): Không advertise cho bilateral peers

Use cases:
- Traffic engineering: Community tag → trigger policy ở ISP
- Blackhole: 65000:666 → ISP discard traffic (DDoS mitigation)
- Prefix classification: 65000:100 = customer, 65000:200 = peer

Ví dụ ISP policy:
- Nhận route với community 65000:100 → LOCAL_PREF 200
- Nhận route với community 65000:200 → LOCAL_PREF 150
- Nhận route với community 65000:666 → Null route (blackhole)
```

### Trong thực tế

```
Doanh nghiệp multihomed với 2 ISP:

                    Internet
                   /        \
            ISP-A              ISP-B
          (Primary)           (Backup)
              \                 /
               \               /
            R1 ─── iBGP ─── R2
                AS 65000

Outbound policy (traffic ĐI):
- R1: Set LOCAL_PREF 200 cho routes từ ISP-A
- R2: Set LOCAL_PREF 100 cho routes từ ISP-B
→ Traffic đi qua ISP-A (LOCAL_PREF cao hơn)

Inbound policy (traffic VỀ):
- Advertise tất cả prefixes cho ISP-A (normal AS_PATH)
- Advertise tất cả prefixes cho ISP-B với AS_PATH prepend x3
→ Internet prefer đường vào qua ISP-A (AS_PATH ngắn hơn)

Failover:
- ISP-A down → LOCAL_PREF 200 routes withdrawn
- Tất cả traffic outbound shift sang ISP-B (LOCAL_PREF 100)
- Internet thấy AS_PATH qua ISP-B (prepended) vẫn còn
→ Inbound traffic cũng shift sang ISP-B
```

### Trong AWS

```
AWS Direct Connect BGP attributes:
- AS_PATH: AWS default = AS 7224 (hoặc private AS trong DX)
- LOCAL_PREF: Set trên customer router cho DX vs VPN preference
- MED: AWS set MED dựa trên DX location distance
- COMMUNITY: 
  - 7224:8100 = Routes propagated to all DX regions
  - 7224:9100 = Routes propagated to all DX locations in region
  - 7224:9200 = Routes not propagated to any DX location
```

---

## 4. BGP Best Path Selection — 13 bước chọn đường tốt nhất

### Mini-example: Tuyển nhân viên

Giống quy trình tuyển dụng — so sánh từng tiêu chí từ quan trọng nhất:

1. Ứng viên được **sếp chỉ định** (Weight) → chọn ngay, bỏ qua tất cả
2. Nếu bằng → Xem **chính sách công ty ưu tiên** (Local Pref)
3. Nếu bằng → Xem **kinh nghiệm ngắn nhất** nhưng relevant (AS-PATH)
4. Nếu bằng → Xem **bằng cấp** — PhD > Master > Bachelor (Origin)
5. Tiếp tục so sánh... cho đến khi chọn được 1 người

### 13 bước Best Path Selection (Cisco)

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP │ ATTRIBUTE        │ PREFER     │ SCOPE        │ MNEMONIC │
├──────┼──────────────────┼────────────┼──────────────┼──────────┤
│  1   │ Weight           │ HIGHEST    │ Local only   │ W        │
│  2   │ Local Preference │ HIGHEST    │ Within AS    │ L        │
│  3   │ Locally originated│ LOCAL wins │ Local        │ L        │
│  4   │ AS_PATH length   │ SHORTEST   │ Global       │ A        │
│  5   │ Origin           │ IGP>EGP>?  │ Global       │ O        │
│  6   │ MED              │ LOWEST     │ Same neighbor│ M        │
│  7   │ eBGP over iBGP   │ eBGP wins  │ Local        │ E        │
│  8   │ IGP metric to NH │ LOWEST     │ Local        │ I        │
│  9   │ Oldest route     │ OLDEST     │ Local        │ O        │
│ 10   │ Router ID        │ LOWEST     │ Tiebreaker   │ R        │
│ 11   │ Cluster length   │ SHORTEST   │ RR only      │ C        │
│ 12   │ Neighbor IP      │ LOWEST     │ Tiebreaker   │ N        │
│ 13   │ (varies)         │ Platform   │ Vendor       │          │
└──────┴──────────────────┴────────────┴──────────────┴──────────┘

Mnemonic: "We Love Lydia As Our Mexican Employee Is Often Right Considering Neighbors"
          (hoặc: Weight, Local-pref, Locally-originated, AS-path, Origin, MED, 
           External, IGP-metric, Oldest, Router-ID, Cluster-list, Neighbor-IP)
```

### Ví dụ chi tiết Best Path Selection

```
Router R1 nhận 3 routes cho prefix 8.8.8.0/24:

Path A (từ ISP-1 via eBGP):
  Weight: 0, LP: 200, AS_PATH: 65001 15169, Origin: IGP
  MED: 100, Next-hop: 10.1.1.1 (IGP cost: 10)

Path B (từ ISP-2 via eBGP):
  Weight: 0, LP: 200, AS_PATH: 65002 15169, Origin: IGP
  MED: 50, Next-hop: 10.2.2.2 (IGP cost: 20)

Path C (từ iBGP peer):
  Weight: 0, LP: 150, AS_PATH: 65001 15169, Origin: IGP
  MED: 100, Next-hop: 10.3.3.3 (IGP cost: 5)

Selection process:
Step 1: Weight → All 0 → TIE
Step 2: Local Pref → A(200) = B(200) > C(150) → C eliminated
Step 3: Locally originated → Neither → TIE
Step 4: AS_PATH → A(2 AS) = B(2 AS) → TIE
Step 5: Origin → Both IGP → TIE
Step 6: MED → Compare ONLY if from same neighbor AS!
         A from AS 65001, B from AS 65002 → khác AS → SKIP MED
         (Trừ khi "bgp always-compare-med" enabled)
Step 7: eBGP over iBGP → Both eBGP → TIE
Step 8: IGP metric to next-hop → A(10) < B(20) → A WINS!

★ BEST PATH = Path A
```

### Các điểm thường nhầm lẫn

```
1. MED chỉ compare giữa routes từ CÙNG neighboring AS (mặc định)
   Fix: bgp always-compare-med (compare across all neighbors)
   
2. Weight chỉ LOCAL — không advertise cho ai
   → Dùng khi muốn 1 router prefer 1 path mà không ảnh hưởng AS

3. LOCAL_PREF chỉ TRONG AS — không gửi ra eBGP
   → Dùng khi muốn TOÀN BỘ AS prefer 1 exit point

4. AS_PATH prepending ảnh hưởng routers BÊN NGOÀI AS
   → Dùng cho inbound traffic engineering

5. MED là "suggestion" — neighbor's LOCAL_PREF override
   → Dùng khi có NHIỀU links đến CÙNG neighbor AS

6. Next-hop-self quan trọng cho iBGP!
   → Nếu thiếu: iBGP routes invalid vì next-hop unreachable
```

### Trong thực tế

**Traffic Engineering Toolkit:**
```
┌──────────────────────────────────────────────────────────────┐
│ Muốn thay đổi GÌ?     │ Dùng ATTRIBUTE nào?                │
├────────────────────────┼─────────────────────────────────────┤
│ 1 router chọn path    │ Weight (local only, highest wins)   │
│ Toàn AS chọn exit     │ Local Preference (iBGP, highest)    │
│ Neighbor chọn entry   │ MED (eBGP, lowest, weak)            │
│ Internet chọn path    │ AS-PATH prepend (longer = worse)    │
│ Community-based policy │ Community tags → trigger actions    │
└────────────────────────┴─────────────────────────────────────┘
```

### Trong AWS

```
AWS BGP Path Selection cho Direct Connect:

Outbound từ AWS:
- AWS dùng longest prefix match TRƯỚC
- Sau đó: AS_PATH length
- Sau đó: DX location proximity

Inbound vào AWS:
- Customer set LOCAL_PREF trên router (highest wins)
- Customer prepend AS_PATH trên DX backup

Ví dụ: DX primary + VPN backup
Customer router:
! DX (preferred):
route-map DX_IN permit 10
 set local-preference 200

! VPN (backup):
route-map VPN_IN permit 10
 set local-preference 100

! Outbound preference to AWS:
! Advertise shorter AS_PATH via DX (primary)
! Advertise prepended AS_PATH via VPN (backup)
```

---

## 5. BGP Messages và FSM — Giao tiếp giữa BGP Peers

### Mini-example: Quy trình kết bạn trên mạng xã hội

1. **OPEN** = Gửi lời mời kết bạn — "Hi, tôi là AS 65000, muốn làm bạn"
2. **KEEPALIVE** = Like/React mỗi 60s — "Tôi vẫn online"
3. **UPDATE** = Post mới — "Có đường mới đến 8.8.8.0/24" hoặc "Đường cũ không còn"
4. **NOTIFICATION** = Block/Unfriend — "Có lỗi, tôi hủy kết bạn" → đóng connection

### 4 loại BGP Message

```
┌─────────────────────────────────────────────────────────────────┐
│ Message      │ Function                │ When sent               │
├──────────────┼─────────────────────────┼─────────────────────────┤
│ OPEN         │ Initiate session        │ After TCP established   │
│              │ Exchange parameters     │ (once per session)      │
│              │ - BGP Version (4)       │                         │
│              │ - My AS Number          │                         │
│              │ - Hold Time (180s)      │                         │
│              │ - Router ID             │                         │
│              │ - Capabilities          │                         │
├──────────────┼─────────────────────────┼─────────────────────────┤
│ UPDATE       │ Advertise/Withdraw      │ When topology changes   │
│              │ routes                  │ or initial exchange     │
│              │ - Withdrawn Routes      │                         │
│              │ - Path Attributes       │                         │
│              │ - NLRI (prefixes)       │                         │
├──────────────┼─────────────────────────┼─────────────────────────┤
│ KEEPALIVE    │ Maintain session        │ Every 60s (1/3 hold)    │
│              │ (19 bytes, no data)     │                         │
├──────────────┼─────────────────────────┼─────────────────────────┤
│ NOTIFICATION │ Report error            │ When error detected     │
│              │ Terminate session       │ → Session closes!       │
│              │ - Error Code            │                         │
│              │ - Error Subcode         │                         │
│              │ - Data                  │                         │
└──────────────┴─────────────────────────┴─────────────────────────┘
```

### BGP Finite State Machine (FSM)

```
┌─────────┐         TCP SYN          ┌──────────┐
│  IDLE   │──────────────────────────→│ CONNECT  │
└────┬────┘                           └─────┬────┘
     │ Start                                │ TCP established
     │                    TCP failed         │
     │           ┌──────────────────────┐   │
     │           │                      ↓   ↓
     │           │                   ┌──────────┐
     │           │                   │  ACTIVE  │ ← Retry TCP connect
     │           │                   └────┬─────┘
     │           │                        │ TCP established
     │           │                        ↓
     │           │              ┌───────────────┐
     │           │              │   OPENSENT    │ ← OPEN message sent
     │           │              └───────┬───────┘
     │           │                      │ Receive valid OPEN
     │           │                      ↓
     │           │              ┌───────────────┐
     │           │              │ OPENCONFIRM   │ ← KEEPALIVE sent
     │           │              └───────┬───────┘
     │           │                      │ Receive KEEPALIVE
     │           │                      ↓
     │           │              ┌───────────────┐
     │           └──────────────│ ESTABLISHED   │ ← Session UP!
     │          Error/Timeout   │ (Exchange     │   Routes exchanged
     │          → Reset         │  UPDATE/KEEP) │
     │                          └───────────────┘
     │
     └── NOTIFICATION received at any state → back to IDLE
```

### Common BGP FSM Issues

```
Stuck in IDLE:
- Peer IP unreachable
- TCP port 179 blocked by ACL/firewall
- No "neighbor" command configured

Stuck in ACTIVE:
- TCP connection fails repeatedly
- Remote peer not configured
- Firewall/ACL blocking return traffic

Stuck in OPENSENT:
- Peer not responding to OPEN
- Capability mismatch
- Authentication failure

Flapping ESTABLISHED ↔ IDLE:
- Hold timer expiring (no keepalive received)
- Interface flapping
- MTU issues (large UPDATE fragmented/dropped)
- CPU overload (can't process in time)

Debug commands:
debug ip bgp            ! General BGP debug
debug ip bgp events     ! State transitions
debug ip bgp updates    ! UPDATE messages
debug ip tcp transactions ! TCP connection issues
```

### Trong thực tế

**BGP Session Establishment Timeline:**
```
T=0.000s: R1 initiates TCP to R2:179
T=0.001s: TCP 3-way handshake (SYN, SYN-ACK, ACK)
T=0.002s: R1 sends OPEN (AS 65000, Hold=180, RID=1.1.1.1)
T=0.003s: R2 sends OPEN (AS 65001, Hold=180, RID=2.2.2.2)
T=0.004s: R1 sends KEEPALIVE (confirms R2's OPEN)
T=0.005s: R2 sends KEEPALIVE (confirms R1's OPEN)
T=0.005s: Both enter ESTABLISHED!
T=0.010s: Initial UPDATE exchange begins
          R1 sends all its best paths to R2
          R2 sends all its best paths to R1
T=0.500s: Initial convergence complete
T=60.00s: KEEPALIVE exchange (every 60s)
...
```

### Trong AWS

```
AWS Direct Connect BGP session:
- AWS side: BGP peer IP from /30 or /31 allocation
- Customer side: Configure neighbor statement

! Customer Router (Cisco):
router bgp 65000
 neighbor 169.254.255.1 remote-as 7224    ! AWS ASN
 neighbor 169.254.255.1 description AWS-DX-Primary
 
 address-family ipv4
  neighbor 169.254.255.1 activate
  neighbor 169.254.255.1 route-map AWS_IN in
  neighbor 169.254.255.1 route-map AWS_OUT out
  neighbor 169.254.255.1 prefix-list ALLOWED_FROM_AWS in
  
! AWS BGP timers:
! - Keepalive: 30s (AWS default, shorter than standard 60s)
! - Hold time: 90s (negotiated)
! - BFD: Supported on DX (recommended for fast failover)
```

---

## 6. iBGP Scaling — Route Reflector và Confederation

### Mini-example: Vấn đề Full-Mesh

Tưởng tượng office meeting:
- 5 người → cần 10 cuộc gọi 1-1 để mọi người nói chuyện với nhau
- 10 người → cần 45 cuộc gọi
- 100 người → cần 4,950 cuộc gọi — **Không khả thi!**

iBGP Full-mesh có CÙNG vấn đề: n routers cần n×(n-1)/2 sessions.

### Route Reflector — Giải pháp #1

```
Concept: 1 router làm "trung tâm" — nhận từ tất cả, phản ánh cho tất cả

Không có RR (Full-mesh 5 routers = 10 sessions):
R1 ←→ R2 ←→ R3 ←→ R4 ←→ R5
 └────────────────────────────┘ (mọi cặp phải peer)

Có Route Reflector (5 sessions):
     R1   R2   R3   R4
      \   |   /   /
       \  |  /   /
        \ | /   /
         RR ─── R5    (mọi router chỉ peer với RR)

Route Reflector Rules:
1. Route từ eBGP peer → Reflect cho TẤT CẢ (clients + non-clients)
2. Route từ RR Client → Reflect cho TẤT CẢ clients + non-clients + eBGP
3. Route từ non-client → Reflect CHỈ cho clients
4. Add ORIGINATOR_ID và CLUSTER_LIST cho loop prevention

Terminology:
- RR (Route Reflector): Router trung tâm
- Client: Router peer với RR (không cần full-mesh)
- Non-client: Router peer bình thường (vẫn cần full-mesh với non-clients)
- Cluster: 1 RR + its clients = 1 cluster
```

```cisco
! Route Reflector Configuration:
router bgp 65000
 ! RR clients:
 neighbor 10.0.0.1 remote-as 65000
 neighbor 10.0.0.1 route-reflector-client
 
 neighbor 10.0.0.2 remote-as 65000
 neighbor 10.0.0.2 route-reflector-client
 
 ! Non-client (peer RR):
 neighbor 10.0.0.10 remote-as 65000
 ! (no route-reflector-client = non-client)

! Redundant RR design (recommended):
! 2 RRs in same cluster for HA
router bgp 65000
 bgp cluster-id 1.1.1.1    ! Same cluster-id on both RRs
```

### Confederation — Giải pháp #2

```
Concept: Chia AS lớn thành mini-AS nhỏ (sub-AS)
         Bên ngoài vẫn thấy 1 AS number duy nhất

AS 65000 chia thành:
┌─────────────────────────────────────────┐
│              AS 65000                    │
│                                         │
│  ┌─────────┐        ┌─────────┐        │
│  │Sub-AS   │        │Sub-AS   │        │
│  │65535    │──eBGP──│65534    │        │
│  │ R1  R2  │        │ R3  R4  │        │
│  └─────────┘        └─────────┘        │
│                                         │
└─────────────────────────────────────────┘

- Trong mỗi sub-AS: iBGP full-mesh (nhỏ, manageable)
- Giữa sub-AS: eBGP-like behavior (next-hop changes)
- Bên ngoài AS 65000: Thấy AS_PATH không có sub-AS numbers

Configuration:
router bgp 65535    ! Sub-AS number
 bgp confederation identifier 65000    ! Real AS
 bgp confederation peers 65534         ! Other sub-AS

So sánh RR vs Confederation:
┌───────────────┬─────────────────┬──────────────────────┐
│ Tiêu chí      │ Route Reflector │ Confederation        │
├───────────────┼─────────────────┼──────────────────────┤
│ Complexity    │ Simple          │ Complex              │
│ Migration     │ Easy (no client │ Hard (renumber AS)   │
│               │  config needed) │                      │
│ Path selection│ May be suboptimal│ More natural eBGP    │
│ Scalability   │ Hierarchical RR │ Limited by sub-AS size│
│ Industry use  │ 95%+ networks   │ Very large ISPs only │
└───────────────┴─────────────────┴──────────────────────┘
```

### Trong thực tế

**Hierarchical Route Reflector Design:**
```
                  ┌─────┐   ┌─────┐
                  │RR-1 │───│RR-2 │   Top-tier RR (non-client mesh)
                  └──┬──┘   └──┬──┘
                     │         │
          ┌──────────┼─────────┼──────────┐
          │          │         │          │
       ┌──┴──┐   ┌──┴──┐  ┌──┴──┐   ┌──┴──┐
       │RR-A │   │RR-B │  │RR-C │   │RR-D │  Regional RR (clients of top)
       └──┬──┘   └──┬──┘  └──┬──┘   └──┴──┘
          │         │        │          │
       ┌──┴──┐   PE-1,2   PE-3,4    PE-5,6   PE routers (clients of regional)
       PE-7,8

Benefits:
- Scalable to thousands of PEs
- Regional RR failure → local impact only
- Top-tier RR failure → redundant RR takes over
- Full Internet table only on RR nodes
```

### Trong AWS

```
AWS BGP design internally:
- AWS uses iBGP with Route Reflectors within their network
- Customer doesn't see this — only eBGP peering exposed

Customer-side iBGP design for AWS:
! Multiple DX connections → need iBGP between border routers

R1 (DX Connection 1) ──── iBGP ──── R2 (DX Connection 2)
         │                                    │
    eBGP with AWS                        eBGP with AWS

! If multiple border routers:
! Use Route Reflector on core router
router bgp 65000
 ! RR configuration on core
 neighbor 10.0.0.1 route-reflector-client    ! R1 (DX border)
 neighbor 10.0.0.2 route-reflector-client    ! R2 (DX border)
```

---

## 7. BGP Security và Filtering — Bảo vệ Internet

### Mini-example: Kiểm tra hải quan

Giống hải quan kiểm tra hàng hóa qua biên giới:
- **Prefix filtering** = Kiểm tra hàng hóa hợp lệ (không cho hàng cấm)
- **AS-PATH filtering** = Kiểm tra nguồn gốc (xuất xứ rõ ràng)
- **RPKI** = Giấy chứng nhận xuất xứ điện tử (chống hàng giả)
- **BGPSEC** = Sealed container (đảm bảo không bị thay đổi trên đường)

### BGP Threats

```
1. BGP Hijacking (Route Hijacking):
   - Attacker quảng bá prefix KHÔNG thuộc về mình
   - Traffic bị redirect qua attacker
   - Ví dụ: 2018 — BGP hijack redirected traffic đến Amazon Route 53

2. Route Leak:
   - AS vô tình (hoặc cố ý) advertise routes không nên advertise
   - Ví dụ: 2019 — Verizon route leak qua small AS ảnh hưởng Cloudflare

3. AS-PATH Manipulation:
   - Fake AS_PATH để bypass filters
   - Ví dụ: Thêm AS number của victim vào path

4. BGP Session Hijacking:
   - TCP RST attack → session reset → traffic disruption
   - TCP injection → modify BGP messages
```

### Prefix Filtering Best Practices

```cisco
! RULE 1: Never accept default route (unless intended)
ip prefix-list DENY_DEFAULT deny 0.0.0.0/0

! RULE 2: Deny bogon prefixes
ip prefix-list BOGONS deny 0.0.0.0/8 le 32        ! "This" network
ip prefix-list BOGONS deny 10.0.0.0/8 le 32       ! RFC 1918
ip prefix-list BOGONS deny 100.64.0.0/10 le 32    ! Shared address space
ip prefix-list BOGONS deny 127.0.0.0/8 le 32      ! Loopback
ip prefix-list BOGONS deny 169.254.0.0/16 le 32   ! Link-local
ip prefix-list BOGONS deny 172.16.0.0/12 le 32    ! RFC 1918
ip prefix-list BOGONS deny 192.0.2.0/24 le 32     ! Documentation
ip prefix-list BOGONS deny 192.168.0.0/16 le 32   ! RFC 1918
ip prefix-list BOGONS deny 224.0.0.0/4 le 32      ! Multicast
ip prefix-list BOGONS deny 240.0.0.0/4 le 32      ! Reserved

! RULE 3: Deny too-specific prefixes (> /24 for IPv4)
ip prefix-list MAX_PREFIX deny 0.0.0.0/0 ge 25 le 32

! RULE 4: Only accept customer's registered prefixes
ip prefix-list CUSTOMER_A permit 203.0.113.0/24
ip prefix-list CUSTOMER_A deny 0.0.0.0/0 le 32

! RULE 5: Max-prefix limit (protect against route leak)
router bgp 65000
 neighbor 10.1.1.1 maximum-prefix 100 80 restart 30
 ! Accept max 100 prefixes, warning at 80%, restart after 30 min
```

### RPKI (Resource Public Key Infrastructure)

```
RPKI = Certificate-based validation của route origin

Components:
1. ROA (Route Origin Authorization):
   - Signed certificate: "Prefix X.X.X.0/24 originated by AS YYYYY"
   - Signed by prefix owner's certificate
   
2. Validation States:
   - VALID: ROA exists, AS matches, prefix length OK
   - INVALID: ROA exists BUT AS or prefix doesn't match (HIJACK!)
   - NOT FOUND: No ROA exists (unknown)

3. Policy:
   - Drop INVALID routes (recommended)
   - Accept VALID and NOT_FOUND
   - Gradually increase NOT_FOUND → INVALID pressure

Configuration:
router bgp 65000
 bgp bestpath prefix-validate allow-invalid
 ! Or strict mode:
 ! bgp bestpath prefix-validate disable-invalid

 neighbor 10.1.1.1 route-map RPKI_POLICY in

route-map RPKI_POLICY deny 10
 match rpki invalid    ! Drop RPKI invalid routes
route-map RPKI_POLICY permit 20
 match rpki valid
 set local-preference 200    ! Prefer RPKI valid
route-map RPKI_POLICY permit 30
 match rpki not-found
 set local-preference 100    ! Lower pref for unknown
```

### Trong thực tế

**Complete BGP Security Policy:**
```cisco
! Peer with ISP-A:
router bgp 65000
 neighbor 10.1.1.1 remote-as 65001
 neighbor 10.1.1.1 description ISP-A
 neighbor 10.1.1.1 password BGP_MD5_Secret
 neighbor 10.1.1.1 ttl-security hops 1
 neighbor 10.1.1.1 maximum-prefix 800000 90

 address-family ipv4
  neighbor 10.1.1.1 activate
  neighbor 10.1.1.1 prefix-list BOGONS in
  neighbor 10.1.1.1 prefix-list MY_PREFIXES out
  neighbor 10.1.1.1 route-map ISP_A_IN in
  neighbor 10.1.1.1 route-map ISP_A_OUT out

! TTL Security (GTSM - Generalized TTL Security Mechanism):
! Only accept BGP from TTL ≥ 254 (directly connected)
! Prevents remote TCP RST attacks
```

### Trong AWS

```
AWS BGP security features:
1. Prefix limit: AWS enforces max 100 prefixes per BGP session (DX)
2. AS_PATH: AWS checks customer AS in path
3. MD5 auth: Supported on DX BGP sessions
4. RPKI: AWS validates ROAs for routes received

Customer best practices:
- Always filter AWS advertisements (only accept AWS-owned prefixes)
- Set max-prefix on DX BGP session
- Use MD5 authentication
- Monitor BGP session with CloudWatch
- Enable BFD for fast failure detection
```

---

## 8. Tình huống thực tế

### Tình huống 1: Doanh nghiệp Multihomed (2 ISP)

```
                    Internet
                   /        \
            ISP-A              ISP-B
         AS 65001            AS 65002
         (Primary)           (Secondary)
              │                   │
              │ eBGP              │ eBGP
              │                   │
         ┌────┴────┐        ┌────┴────┐
         │  R1     │──iBGP──│   R2    │
         │Border-1 │        │Border-2 │
         └────┬────┘        └────┬────┘
              │                   │
              └───── Campus ──────┘
                   AS 65000
                203.0.113.0/24

Design Goals:
- Outbound: Prefer ISP-A (faster)
- Inbound: Prefer ISP-A (shorter path to us)
- Failover: Automatic when any link fails

Configuration R1:
router bgp 65000
 neighbor 198.51.100.1 remote-as 65001    ! ISP-A
 
 address-family ipv4
  network 203.0.113.0 mask 255.255.255.0
  neighbor 198.51.100.1 route-map ISP_A_IN in
  neighbor 198.51.100.1 route-map ISP_A_OUT out

! Inbound: Normal advertisement (no prepend)
route-map ISP_A_OUT permit 10
 ! Default: no prepend, shortest path

Configuration R2:
router bgp 65000
 neighbor 198.51.200.1 remote-as 65002    ! ISP-B
 
 address-family ipv4
  network 203.0.113.0 mask 255.255.255.0
  neighbor 198.51.200.1 route-map ISP_B_IN in
  neighbor 198.51.200.1 route-map ISP_B_OUT out

! Inbound: Prepend to make path longer
route-map ISP_B_OUT permit 10
 set as-path prepend 65000 65000 65000

! Outbound policy (both routers):
route-map ISP_A_IN permit 10
 set local-preference 200    ! Prefer ISP-A

route-map ISP_B_IN permit 10
 set local-preference 100    ! ISP-B as backup
```

### Tình huống 2: ISP peering tại IXP (Internet Exchange Point)

```
IXP (VNIX example):
┌───────────────────────────────────────────┐
│          VNIX (Vietnam IX)                │
│                                           │
│  Viettel ── Route Server ── VNPT          │
│  AS 7552    (assists peering) AS 45899    │
│      │                         │          │
│      └──── Direct Peering ─────┘          │
│                                           │
│  FPT ──── Cloudflare ──── Google          │
│  AS 18403  AS 13335        AS 15169       │
└───────────────────────────────────────────┘

Peering types:
1. Bilateral peering: Direct BGP session between 2 AS
2. Multilateral peering: Via Route Server (RS)
   - Peer with RS = peer with everyone connected to RS
   - Simpler config but less control

BGP communities at IXP:
- RS community: 0:PEER_AS = don't advertise to specific peer
- Blackhole: IXP:666 = blackhole traffic to prefix

Route Server config (IXP operator):
router bgp 65534    ! IXP RS AS
 ! Transparent RS: don't modify attributes
 no bgp enforce-first-as
 neighbor PEER-GROUP peer-group
 neighbor PEER-GROUP route-server-client
 neighbor PEER-GROUP transparent-as
 neighbor PEER-GROUP transparent-nexthop
```

### Tình huống 3: AWS Direct Connect với BGP

```
Architecture:
┌──────────────────────┐     ┌──────────────────────┐
│   On-Premises        │     │        AWS           │
│   AS 65000           │     │                      │
│                      │     │    ┌────────────┐    │
│ ┌─────┐   ┌─────┐   │     │    │ DX Gateway │    │
│ │Core │───│ PE1 │──DX1────│────│            │    │
│ │  RR │   └─────┘   │     │    │            │    │
│ │     │   ┌─────┐   │     │    │            │    │
│ │     │───│ PE2 │──DX2────│────│            │    │
│ └─────┘   └─────┘   │     │    └─────┬──────┘    │
│                      │     │          │           │
│                      │     │    ┌─────┴──────┐    │
│                      │     │    │    TGW     │    │
│                      │     │    └──┬────┬────┘    │
│                      │     │       │    │         │
│                      │     │    VPC-1  VPC-2     │
└──────────────────────┘     └──────────────────────┘

BGP Configuration:
! PE1 (Primary DX):
router bgp 65000
 neighbor 169.254.10.1 remote-as 7224
 address-family ipv4
  neighbor 169.254.10.1 activate
  neighbor 169.254.10.1 route-map TO_AWS out
  neighbor 169.254.10.1 route-map FROM_AWS in
  
route-map TO_AWS permit 10
 match ip address prefix-list ON_PREM_NETWORKS
 ! No prepend (primary)
 
route-map FROM_AWS permit 10
 set local-preference 200    ! Prefer DX over VPN

! PE2 (Secondary DX):
route-map TO_AWS_BACKUP permit 10
 match ip address prefix-list ON_PREM_NETWORKS
 set as-path prepend 65000    ! Make backup less preferred

route-map FROM_AWS_BACKUP permit 10
 set local-preference 150    ! Lower than primary DX

! Failover timeline:
! DX link failure → BGP hold timer (90s) → failover to backup DX
! With BFD: Failure detect < 1s → BGP session down → failover
```

### Tình huống 4: BGP Incident Response — Route Hijack

```
Scenario: Attacker quảng bá prefix 203.0.113.0/24 (thuộc về bạn!)

Detection:
1. Monitoring tools: BGPStream, RIPE RIS, RouteViews
2. Alert: "New origin AS 99999 for prefix 203.0.113.0/24!"
3. Impact: Traffic bị redirect, customers không access được

Response:
1. Verify: Kiểm tra bạn vẫn đang advertise prefix
2. Contact: Liên hệ upstream ISP + IXP
3. More-specific: Advertise /25s (203.0.113.0/25 + 203.0.113.128/25)
   → Longer prefix wins in routing!
4. RPKI: Nếu có ROA → invalid route sẽ bị reject bởi RPKI-enabled networks
5. Community: Gửi blackhole community cho ISPs

Prevention:
- Register ROA in RPKI (APNIC)
- Document IRR objects (in APNIC WHOIS)
- Monitor BGP continuously
- Peer with only trusted neighbors
- Use prefix limits
```

---

## 9. Bài tập thực hành

### Bài tập 1: Basic eBGP Configuration

```cisco
! Topology: R1 (AS 65000) ── eBGP ── R2 (AS 65001) ── eBGP ── R3 (AS 65002)

! R1:
router bgp 65000
 bgp router-id 1.1.1.1
 neighbor 10.12.0.2 remote-as 65001
 
 address-family ipv4
  network 192.168.1.0 mask 255.255.255.0
  neighbor 10.12.0.2 activate

! Verify:
show bgp summary                    ! Peer state (should be Established)
show bgp ipv4 unicast              ! BGP table
show bgp ipv4 unicast 192.168.3.0  ! Specific prefix detail
show ip route bgp                   ! BGP routes in RIB

! Questions:
! 1. What is the AS_PATH for R3's prefix from R1's perspective?
! 2. What is the NEXT_HOP for R3's routes on R1?
! 3. Why doesn't R2 need "network 10.12.0.0"?
```

### Bài tập 2: Path Selection Manipulation

```cisco
! R1 receives 8.8.8.0/24 from both ISP-A and ISP-B

! Task 1: Make R1 prefer ISP-A using LOCAL_PREF
route-map PREFER_A permit 10
 set local-preference 200
router bgp 65000
 neighbor 10.1.1.1 route-map PREFER_A in

! Task 2: Make R1 prefer ISP-A using Weight (alternative)
router bgp 65000
 neighbor 10.1.1.1 weight 500

! Task 3: Influence inbound - Make Internet prefer path via ISP-A
route-map PREPEND_B permit 10
 set as-path prepend 65000 65000
router bgp 65000
 neighbor 10.2.2.2 route-map PREPEND_B out

! Verify:
show bgp ipv4 unicast 8.8.8.0/24
! Look for: Weight, LP, AS_PATH, best path indicator (>)

! Advanced: Change preference for SPECIFIC prefixes only
ip prefix-list GOOGLE_DNS permit 8.8.8.0/24
route-map PREFER_A permit 10
 match ip address prefix-list GOOGLE_DNS
 set local-preference 300
route-map PREFER_A permit 20
 ! Default: no change for other routes
```

### Bài tập 3: iBGP and Route Reflector

```cisco
! Topology: 4 routers in AS 65000
! R1 = Route Reflector
! R2, R3, R4 = RR Clients

! R1 (RR):
router bgp 65000
 bgp router-id 1.1.1.1
 neighbor 2.2.2.2 remote-as 65000
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 2.2.2.2 route-reflector-client
 neighbor 3.3.3.3 remote-as 65000
 neighbor 3.3.3.3 update-source Loopback0
 neighbor 3.3.3.3 route-reflector-client
 neighbor 4.4.4.4 remote-as 65000
 neighbor 4.4.4.4 update-source Loopback0
 neighbor 4.4.4.4 route-reflector-client

! R2 (Client):
router bgp 65000
 bgp router-id 2.2.2.2
 neighbor 1.1.1.1 remote-as 65000
 neighbor 1.1.1.1 update-source Loopback0
 ! That's it! Client doesn't know it's a client

! Verification:
show bgp ipv4 unicast    ! Check ORIGINATOR_ID and CLUSTER_LIST
! Routes reflected by RR will have these attributes

! Exercise:
! 1. If R2 advertises 10.0.0.0/8, does R3 receive it?
! 2. What's the ORIGINATOR_ID on R3 for R2's route?
! 3. What happens if R1 (RR) fails? Design redundant RR!
```

### Bài tập 4: BGP Communities

```cisco
! ISP offers these communities:
! 65001:100 = Set LOCAL_PREF 100 (normal)
! 65001:200 = Set LOCAL_PREF 200 (preferred)
! 65001:300 = Set LOCAL_PREF 50 (backup)
! 65001:666 = Blackhole route

! Task: Advertise prefix 203.0.113.0/24 with blackhole community
route-map BLACKHOLE permit 10
 match ip address prefix-list ATTACKED_PREFIX
 set community 65001:666

ip prefix-list ATTACKED_PREFIX permit 203.0.113.128/25

router bgp 65000
 address-family ipv4
  neighbor 10.1.1.1 send-community
  neighbor 10.1.1.1 route-map BLACKHOLE out

! Verify:
show bgp community 65001:666
show bgp ipv4 unicast 203.0.113.128/25

! Remove blackhole when attack stops:
no route-map BLACKHOLE permit 10
 match ip address prefix-list ATTACKED_PREFIX
 set community 65001:666
! Or: Remove prefix from ATTACKED_PREFIX list
```

### Bài tập 5: BGP Troubleshooting

```cisco
! Scenario: BGP neighbor not coming up

! Step 1: Check basic connectivity
ping 10.1.1.1 source 10.1.1.2    ! Can reach peer?

! Step 2: Check BGP state
show bgp summary
! State should be "Established" — if "Active" or "Idle":

! Step 3: Check TCP connectivity
telnet 10.1.1.1 179    ! Can reach BGP port?
! If fail → firewall/ACL blocking!

! Step 4: Check configuration
show run | section router bgp
! - Remote AS correct?
! - Neighbor IP correct?
! - update-source configured? (for iBGP via Loopback)

! Step 5: Check route to peer
show ip route 10.1.1.1
! Must have route to peer IP (especially for iBGP!)

! Step 6: Debug
debug ip bgp neighbor 10.1.1.1 events
debug ip bgp neighbor 10.1.1.1 notifications
! Look for: OPEN message errors, capability mismatch, auth failure

! Common fixes:
! 1. Add "ebgp-multihop 2" if not directly connected
! 2. Add "update-source Loopback0" for iBGP
! 3. Add "next-hop-self" on border router for iBGP
! 4. Check "passive-interface" on underlying IGP
! 5. Verify MD5 password matches on both sides
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Tóm tắt kiến thức

```
┌─────────────────────────────────────────────────────────────┐
│                    BGP SUMMARY                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ▸ BGP = Path-Vector EGP, TCP port 179                      │
│ ▸ Kết nối 70,000+ AS thành Internet duy nhất              │
│ ▸ Policy-based: Business decisions, không chỉ metric       │
│                                                             │
│ ▸ eBGP: Giữa AS (AD=20), iBGP: Trong AS (AD=200)         │
│ ▸ iBGP split-horizon → cần Full-mesh/RR/Confederation     │
│                                                             │
│ ▸ Path Selection (top 8 thường dùng):                      │
│   1. Weight (highest, Cisco-only, local)                   │
│   2. Local Preference (highest, iBGP)                      │
│   3. Locally originated                                    │
│   4. AS-PATH shortest                                      │
│   5. Origin (IGP > EGP > ?)                               │
│   6. MED lowest (same neighbor AS)                         │
│   7. eBGP > iBGP                                          │
│   8. IGP metric to next-hop lowest                         │
│                                                             │
│ ▸ Key Attributes:                                          │
│   - AS_PATH: Loop prevention + path selection              │
│   - NEXT_HOP: Critical cho iBGP (next-hop-self!)          │
│   - LOCAL_PREF: Outbound traffic engineering               │
│   - MED: Inbound hint (weak)                              │
│   - COMMUNITY: Scalable policy signaling                   │
│                                                             │
│ ▸ Security: Prefix filtering, RPKI, MD5/SHA auth          │
│ ▸ AWS: DX, VPN, TGW — all use BGP                         │
└─────────────────────────────────────────────────────────────┘
```

### Traffic Engineering Quick Reference

```
┌─────────────────────────────────────────────────────────────┐
│ GOAL                    │ TOOL                    │ SCOPE     │
├─────────────────────────┼─────────────────────────┼───────────┤
│ 1 router prefers path X│ Weight (highest)        │ 1 router  │
│ All AS prefers exit X   │ Local Pref (highest)    │ Whole AS  │
│ Neighbor enters via X   │ MED (lowest)           │ 1 neighbor│
│ Internet prefers path X │ AS-PATH prepend         │ Internet  │
│ Selective policy        │ Community              │ Flexible  │
│ Blackhole attack traffic│ Community :666 + /32   │ ISP       │
└─────────────────────────┴─────────────────────────┴───────────┘
```

### Tài liệu tham khảo

1. **RFC 4271** — "A Border Gateway Protocol 4 (BGP-4)" — BGP specification
   - https://www.rfc-editor.org/rfc/rfc4271

2. **RFC 4456** — "BGP Route Reflection: An Alternative to Full Mesh Internal BGP"
   - https://www.rfc-editor.org/rfc/rfc4456

3. **RFC 1997** — "BGP Communities Attribute"
   - https://www.rfc-editor.org/rfc/rfc1997

4. **RFC 6811** — "BGP Prefix Origin Validation" (RPKI)
   - https://www.rfc-editor.org/rfc/rfc6811

5. **RFC 7454** — "BGP Operations and Security" (Best Current Practice)
   - https://www.rfc-editor.org/rfc/rfc7454

6. **Cisco Documentation** — "BGP Best Path Selection Algorithm"
   - https://www.cisco.com/c/en/us/support/docs/ip/border-gateway-protocol-bgp/13753-25.html

7. **AWS Documentation** — "AWS Direct Connect BGP"
   - https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html

8. **"Internet Routing Architectures"** by Sam Halabi (Cisco Press) — The definitive BGP book

9. **BGP Tools:**
   - https://bgp.he.net — BGP route lookup
   - https://stat.ripe.net — RIPE routing stats
   - https://bgpstream.com — BGP monitoring

---

*Bài viết tiếp theo: [BGP trong AWS](/2026/07/03/bgp-in-aws) — Direct Connect BGP, Transit Gateway, AS-PATH prepending*
