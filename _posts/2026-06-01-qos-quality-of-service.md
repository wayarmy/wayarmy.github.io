---
layout: post
title: "QoS (Quality of Service) Deep Dive - Quản Lý Chất Lượng Mạng"
date: 2026-06-01
categories: [networking]
tags: [qos, dscp, queuing, traffic-management]
---

# QoS (Quality of Service) Deep Dive - Quản Lý Chất Lượng Mạng

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Hãy tưởng tượng một con đường có 4 làn xe. Vào giờ cao điểm, tất cả xe đều chen nhau, xe cứu thương bị kẹt giữa dòng xe bình thường → bệnh nhân không đến bệnh viện kịp.

**Giải pháp:** Dành riêng 1 làn cho xe ưu tiên (cứu thương, cứu hỏa, cảnh sát). Xe thường vẫn đi được nhưng phải dùng 3 làn còn lại.

**QoS (Quality of Service — Chất Lượng Dịch Vụ)** hoạt động y hệt:
- Xác định traffic nào QUAN TRỌNG (gọi video, VoIP)
- Xác định traffic nào ÍT QUAN TRỌNG (download file, backup)
- Đảm bảo traffic quan trọng luôn được ưu tiên, đặc biệt khi mạng nghẽn

**Thêm ví dụ đời thường:**

| Tình huống | Không QoS | Có QoS |
|-----------|-----------|--------|
| Đường cao tốc | Tất cả xe chen nhau | Có làn ưu tiên cho xe khẩn cấp |
| Bệnh viện | Xếp hàng ai đến trước | Phân loại: cấp cứu trước, khám thường sau |
| Sân bay | 1 queue cho tất cả | Business class boarding trước Economy |
| Nhà hàng | Order trước phục vụ trước | VIP table phục vụ ưu tiên |

**Tại sao cần QoS?**
- **VoIP/Video call:** Cần latency < 150ms, jitter < 30ms, loss < 1% → nếu không đảm bảo = nghe giật, hình vỡ
- **Online gaming:** Cần latency < 50ms → lag = thua game
- **Web browsing:** Chịu được 200-500ms delay → không ai notice
- **File backup:** Chịu được minutes delay → không urgent

Khi mạng KHÔNG nghẽn → QoS không cần thiết (đủ bandwidth cho tất cả)
Khi mạng NGHẼN → QoS quyết định ai được đi trước, ai phải chờ

---

## 2. Kiến Thức Nền Tảng

### 2.1 Bandwidth, Latency, Jitter, Packet Loss

| Metric | Ví dụ đời thường | Ý nghĩa | Ảnh hưởng |
|--------|-----------------|---------|-----------|
| Bandwidth | Độ rộng đường (4 làn vs 1 làn) | Dung lượng tối đa (Mbps) | Download speed |
| Latency | Thời gian đi từ A đến B | Độ trễ 1 chiều (ms) | Real-time apps |
| Jitter | Xe đến đích lúc nhanh lúc chậm | Biến thiên latency (ms) | Voice/video quality |
| Packet Loss | Xe bị "nuốt" giữa đường | % gói tin bị mất | Retransmission, gaps |

**Yêu cầu QoS cho từng loại traffic:**
```
VoIP (Voice over IP):
- Latency: < 150ms (one-way), tốt nhất < 80ms
- Jitter: < 30ms
- Packet loss: < 1%
- Bandwidth: ~100 Kbps per call

Video Conferencing:
- Latency: < 200ms
- Jitter: < 50ms
- Packet loss: < 1%
- Bandwidth: 1-5 Mbps per stream

Interactive (Gaming, Remote Desktop):
- Latency: < 50ms (gaming), < 100ms (RDP)
- Jitter: < 20ms
- Loss: < 0.1%
- Bandwidth: Variable

Bulk Data (Backup, Downloads):
- Latency: Don't care
- Jitter: Don't care
- Loss: 0% (TCP handles retransmission)
- Bandwidth: Use whatever available
```

### 2.2 QoS Pipeline — 4 Giai Đoạn

```
Traffic vào router/switch → QoS xử lý qua 4 bước:

┌──────────┐   ┌────────────┐   ┌──────────────┐   ┌──────────────┐
│CLASSIFY  │ → │   MARK     │ → │   QUEUE      │ → │  SCHEDULE    │
│(Phân loại)│   │ (Đánh dấu) │   │ (Xếp hàng)  │   │ (Lên lịch)  │
└──────────┘   └────────────┘   └──────────────┘   └──────────────┘

1. Classify: Nhận diện traffic thuộc loại nào
   (voice? video? web? backup?)

2. Mark: Đánh dấu packet để router tiếp theo biết ưu tiên
   (gán DSCP value vào IP header)

3. Queue: Xếp packet vào hàng đợi phù hợp
   (voice → queue ưu tiên cao, backup → queue ưu tiên thấp)

4. Schedule: Quyết định packet nào được gửi đi tiếp
   (lấy từ queue ưu tiên cao trước)
```

---

## 3. Classification & Marking — Phân Loại và Đánh Dấu

### 3.1 Classification — Nhận Diện Traffic

**Ví dụ đời thường:** Nhân viên phân loại thư tại bưu điện — nhìn phong bì để xác định: thư thường, thư bảo đảm, hay chuyển phát nhanh.

**Tiêu chí phân loại:**
```
Layer 2: 802.1Q CoS (Class of Service) — 3 bits trong VLAN tag
Layer 3: DSCP (Differentiated Services Code Point) — 6 bits trong IP header
Layer 4: Port numbers (TCP/UDP port)
Deep: Application inspection (DPI - Deep Packet Inspection)

Ví dụ classification rules:
- Source/Dest IP: 10.0.0.0/24 → Voice VLAN → Đánh dấu EF
- TCP Port 80/443: Web traffic → Đánh dấu AF21
- UDP Port 5060 (SIP): VoIP signaling → Đánh dấu CS3  
- UDP Port 16384-32767 (RTP): Voice media → Đánh dấu EF
- DSCP already marked by endpoint → Trust and forward
```

### 3.2 DSCP (Differentiated Services Code Point)

**DSCP** là 6 bits trong IP header (cho 64 giá trị = 0-63), dùng để đánh dấu mức ưu tiên của packet.

```
IP Header:
┌────────────────────────────────┐
│ Version │ IHL │ DSCP │ ECN │...│
│  4 bits │4bits│6 bits│2bits│   │
└────────────────────────────────┘
                  ↑
           Đây là nơi QoS marking!
```

**Các DSCP value phổ biến (RFC 4594 recommendations):**

| DSCP Name | Value (decimal) | Binary | Per-Hop Behavior | Dùng cho |
|-----------|----------------|--------|-----------------|----------|
| EF (Expedited Forwarding) | 46 | 101110 | Low latency, low jitter | VoIP |
| AF41 | 34 | 100010 | Assured Forwarding | Video conferencing |
| AF31 | 26 | 011010 | Assured Forwarding | Streaming video |
| AF21 | 18 | 010010 | Assured Forwarding | Critical data (ERP) |
| AF11 | 10 | 001010 | Assured Forwarding | Bulk data |
| CS3 | 24 | 011000 | Class Selector | Signaling (SIP/H.323) |
| CS0/BE | 0 | 000000 | Best Effort (Default) | Everything else |

**AF (Assured Forwarding) naming: AFxy**
- x = Class (1-4): Class cao hơn = ưu tiên hơn
- y = Drop precedence (1-3): Drop cao hơn = bị drop trước khi congestion

```
AF41: Class 4, Drop precedence 1 (ưu tiên cao, ít bị drop)
AF43: Class 4, Drop precedence 3 (ưu tiên cao, nhưng drop trước AF41)
```

### 3.3 Trust Boundaries

```
Vấn đề: PC/IP Phone tự đánh dấu DSCP = EF (ưu tiên cao nhất)
→ Nếu trust tất cả → ai cũng đánh EF → QoS vô nghĩa!

Giải pháp: Trust Boundary
- Switch port kết nối IP Phone: TRUST DSCP (phone marking hợp lệ)
- Switch port kết nối PC: KHÔNG TRUST → reset về CS0 (best effort)
- Uplink port đến core: TRUST (đã được classify ở access layer)

Nguyên tắc: Trust marking CHỈ từ trusted devices/interfaces
            Re-mark untrusted traffic tại network edge
```

---

## 4. Queuing Mechanisms — Cơ Chế Xếp Hàng

### 4.1 FIFO (First In, First Out) — Không Có QoS

**Ví dụ đời thường:** Quầy vé xem phim chỉ có 1 hàng — ai đến trước phục vụ trước, bất kể VIP hay thường.

```
Tất cả packets vào 1 queue duy nhất:
[Packet 1: Voice] → [Packet 2: Backup] → [Packet 3: Voice] → [Packet 4: Web]

Xử lý: 1 → 2 → 3 → 4 (đúng thứ tự vào)

Vấn đề: Backup packet (lớn, lâu) block Voice packet (nhỏ, urgent)
→ Voice bị delay dù là urgent!
```

**Khi nào FIFO OK:** Link speed rất cao, không bao giờ nghẽn, traffic đồng nhất.

### 4.2 WFQ (Weighted Fair Queuing) — Chia Đều Có Trọng Số

**Ví dụ đời thường:** Nhà hàng buffet có 3 bàn ăn. Bàn VIP được phục vụ 3 lần, bàn thường 1 lần. Nhưng tất cả đều được ăn.

```
WFQ tự động tạo queue cho mỗi flow (conversation):
- Flow 1 (VoIP call A): Weight 5
- Flow 2 (HTTP download): Weight 1
- Flow 3 (VoIP call B): Weight 5
- Flow 4 (FTP backup): Weight 1

Scheduling: VoIP flows nhận 5x bandwidth so với download/backup
Nhưng TẤT CẢ flows đều được service (không ai bị starved)
```

**Ưu điểm:** Automatic, fair, no configuration needed
**Nhược điểm:** Không đảm bảo absolute priority cho voice (chỉ proportional)

### 4.3 CBWFQ (Class-Based Weighted Fair Queuing) — WFQ Theo Lớp

**Ví dụ đời thường:** Sân bay có 3 counter riêng: First Class, Business, Economy. Mỗi counter có số nhân viên khác nhau.

```
Định nghĩa classes + bandwidth allocation:

Class "Voice": 
  - Match: DSCP EF
  - Bandwidth: 30% guaranteed
  
Class "Video":
  - Match: DSCP AF41
  - Bandwidth: 30% guaranteed
  
Class "Data":
  - Match: DSCP AF21
  - Bandwidth: 20% guaranteed
  
Class "Default" (best-effort):
  - Match: everything else
  - Bandwidth: 20% (remaining)

Khi link 100 Mbps:
- Voice luôn có ít nhất 30 Mbps
- Video luôn có ít nhất 30 Mbps
- Data luôn có ít nhất 20 Mbps
- Default: 20 Mbps

Khi KHÔNG congestion: classes có thể dùng hơn allocation (share unused bandwidth)
Khi CONGESTION: mỗi class guaranteed phần của mình
```

### 4.4 LLQ (Low Latency Queuing) — CBWFQ + Priority Queue

**Ví dụ đời thường:** Sân bay có counter riêng CHO CẤP CỨU — bệnh nhân cấp cứu LUÔN LUÔN được phục vụ NGAY LẬP TỨC, trước tất cả hàng khác.

```
LLQ = CBWFQ + 1 Priority Queue (strict priority)

┌─────────────────────────────────────┐
│  Priority Queue (Voice - EF)         │  ← LUÔN gửi trước!
│  Policed: max 30% bandwidth          │     Nhưng có giới hạn
├─────────────────────────────────────┤
│  CBWFQ Queue 1 (Video - AF41)       │  ← 30% guaranteed
├─────────────────────────────────────┤
│  CBWFQ Queue 2 (Data - AF21)        │  ← 20% guaranteed
├─────────────────────────────────────┤
│  Default Queue (Best Effort - CS0)   │  ← Remaining bandwidth
└─────────────────────────────────────┘

Scheduling order:
1. Priority Queue có packet? → GỬI NGAY (zero delay)
2. Nếu Priority Queue trống → round-robin các CBWFQ queues theo weight

Tại sao Police Priority Queue?
- Nếu không limit, priority queue chiếm HẾT bandwidth
- CBWFQ queues bị "starved" (không bao giờ được gửi)
- Policing: nếu voice > 30% → drop excess voice packets
  (Thà drop vài voice packet hơn là starve tất cả traffic khác)
```

**LLQ là QoS model được dùng phổ biến nhất** trong enterprise networks vì:
- Voice/realtime traffic: absolute priority (minimal delay)
- Các traffic khác: guaranteed bandwidth proportional
- Cân bằng giữa priority và fairness

---

## 5. Policing vs Shaping — Giới Hạn vs Làm Mượt

### 5.1 Traffic Policing — "Cảnh Sát Giao Thông"

**Ví dụ đời thường:** Cảnh sát bắn tốc độ — xe nào vượt 120 km/h → BỊ PHẠT NGAY (dropped/re-marked). Không có cách nào "chờ" — xe phải chấp nhận bị phạt.

```
Policing: Nếu traffic VƯỢT rate cho phép → DROP hoặc RE-MARK ngay lập tức

Configured rate: 10 Mbps
Traffic gửi:     15 Mbps

Kết quả Policing:
- 10 Mbps: CONFORM (pass through, giữ nguyên DSCP)
- 5 Mbps:  EXCEED (DROP hoặc re-mark DSCP thấp hơn)

Đặc điểm:
- Không buffer, không delay
- Bursty output (traffic rate lên xuống đột ngột)
- Thường dùng ở INGRESS (traffic vào)
- Dùng khi ISP enforce bandwidth limit

Token Bucket Algorithm:
- Bucket size (Bc) = burst allowed
- Fill rate = committed rate
- Packet arrive: có token → pass, không token → drop/remark
```

### 5.2 Traffic Shaping — "Đập Thủy Điện"

**Ví dụ đời thường:** Đập thủy điện — nước (traffic) chảy vào nhanh nhưng đập giữ lại, xả ra từ từ đều đặn. Không mất nước, chỉ delay.

```
Shaping: Nếu traffic VƯỢT rate → BUFFER (giữ lại) → gửi từ từ theo rate

Configured rate: 10 Mbps
Traffic gửi:     15 Mbps burst

Kết quả Shaping:
- Buffer 5 Mbps excess traffic
- Gửi ra đều đặn 10 Mbps
- KHÔNG drop packets (chỉ delay)
- Nếu buffer FULL → mới drop

Đặc điểm:
- Buffer traffic → thêm delay (nhưng không loss)
- Smooth output (rate ổn định)
- Thường dùng ở EGRESS (traffic ra)
- Dùng khi cần conform với ISP contracted rate
- Tốt cho TCP (tránh retransmission do drops)
```

### 5.3 So Sánh Policing vs Shaping

| Đặc điểm | Policing | Shaping |
|-----------|----------|---------|
| Excess traffic | DROP/Remark | BUFFER (delay) |
| Latency added | Không | Có (buffer delay) |
| Memory needed | Không (no buffer) | Có (buffer queue) |
| Output pattern | Bursty | Smooth |
| Packet loss | Có (exceed = drop) | Không (trừ khi buffer full) |
| Direction | Ingress hoặc Egress | Chỉ Egress |
| TCP friendly | Ít (drops → retransmit) | Tốt (smooth = TCP happy) |
| Use case | ISP enforcement, ingress | WAN edge, conform to SLA |

```
Ví dụ thực tế:
- ISP cho bạn plan 100 Mbps → ISP POLICE traffic bạn ở 100 Mbps
- Router công ty bạn trước WAN → SHAPE traffic ra 100 Mbps
  (smooth hơn, TCP performance tốt hơn, tránh ISP drop)
```

---

## 6. Congestion Avoidance — WRED

### 6.1 Tail Drop vs WRED

**Tail Drop (mặc định khi queue đầy):**
```
Queue full → DROP TẤT CẢ packets mới (bất kể priority)

Vấn đề: TCP Global Synchronization
- Nhiều TCP flows bị drop cùng lúc
- Tất cả reduce window cùng lúc  
- Tất cả recover cùng lúc
- → Traffic dao động lên xuống như sóng (không ổn định)

     ___
    /   \     ___
   /     \   /   \     ← TCP throughput dao động
__/       \_/     \__     (global sync)
```

**WRED (Weighted Random Early Detection):**
```
Thay vì chờ queue FULL mới drop:
→ Bắt đầu drop RANDOM packets KHI queue sắp đầy
→ Drop theo xác suất tăng dần

Queue fill level:
0%────────[Min threshold]────[Max threshold]────100%
          ↑                   ↑
     Bắt đầu random drop  100% drop (tail drop)
     xác suất thấp

WEIGHTED: Traffic DSCP thấp bị drop TRƯỚC traffic DSCP cao

Ví dụ:
- DSCP EF (voice): Min=70%, Max=90% (drop rất muộn)
- DSCP AF21 (data): Min=40%, Max=60% (drop sớm hơn)  
- DSCP CS0 (best-effort): Min=20%, Max=40% (drop sớm nhất)
```

**Lợi ích WRED:**
- Tránh TCP Global Synchronization (random drop → random flows slow down → gradual)
- High priority traffic chịu loss ít hơn
- Queue utilization tốt hơn (không swing giữa full và empty)

---

## 7. IntServ vs DiffServ — Hai Kiến Trúc QoS

### 7.1 IntServ (Integrated Services) — Đặt Chỗ Trước

**Ví dụ đời thường:** Đặt vé máy bay trước chuyến bay. Bạn gọi hãng hàng không, đặt chỗ cụ thể (seat reserved) → đảm bảo 100% có chỗ ngồi.

```
IntServ = RSVP (Resource Reservation Protocol)

Cách hoạt động:
1. Application gửi RSVP request: "Tôi cần 100 Kbps, delay < 50ms"
2. Request đi qua TỪNG router trên path
3. Mỗi router check: "Tôi có đủ resource không?"
   - Có → Reserve resource + forward RSVP
   - Không → Reject RSVP
4. Nếu TẤT CẢ routers OK → Path reserved → Application start
5. Resource maintained suốt session (per-flow state)

     [Sender] → RSVP PATH →
     [Router A] → RSVP PATH →
     [Router B] → RSVP PATH →
     [Receiver] → RSVP RESV (confirm) ←
     [Router B] ← RSVP RESV ←
     [Router A] ← RSVP RESV ←
     [Sender] ← RSVP RESV ← DONE! Path reserved!
```

**Ưu điểm IntServ:**
- Guaranteed QoS (100% đảm bảo)
- Per-flow granularity (mỗi flow có SLA riêng)
- Application biết chính xác sẽ nhận service level nào

**Nhược điểm IntServ:**
- **Scalability nightmare!** Mỗi router phải lưu state cho MỖI flow
- Internet có triệu flows → router không đủ memory
- RSVP signaling overhead
- Không thực tế cho Internet-scale

**Sử dụng:** Rất ít, chỉ trong mạng nhỏ/controlled (MPLS TE kế thừa một số concepts)

### 7.2 DiffServ (Differentiated Services) — Phân Loại Tại Biên

**Ví dụ đời thường:** Dịch vụ bưu điện — bạn dán tem "Chuyển phát nhanh" (DSCP marking) lên phong bì. Mỗi bưu cục (router) nhìn tem → xử lý theo quy trình: nhanh chuyển trước, thường chuyển sau. **Không cần đặt trước**, không cần biết có bao nhiêu thư.

```
DiffServ = Mark at edge + Per-Hop Behavior (PHB) at core

Cách hoạt động:
1. Traffic vào network edge → Classify + Mark DSCP
2. Core routers nhìn DSCP → áp dụng PHB (queueing, scheduling)
3. KHÔNG có per-flow state → chỉ per-class state (ít class)
4. Scalable: 10 triệu flows nhưng chỉ 4-8 classes

Edge Router:                Core Router:
┌─────────────┐            ┌─────────────┐
│ Classify    │            │ Read DSCP   │
│ Mark DSCP   │ ─────────→ │ Apply PHB   │
│ Police/Shape│            │ Queue+Sched │
└─────────────┘            └─────────────┘
Complex logic              Simple, fast
(inspect packets deep)     (just look at 6 bits)
```

**Per-Hop Behaviors (PHB):**
```
EF (Expedited Forwarding) — RFC 3246:
- Low delay, low jitter, low loss, guaranteed bandwidth
- Dùng cho: VoIP, real-time
- Implementation: LLQ priority queue

AF (Assured Forwarding) — RFC 2597:
- 4 classes × 3 drop precedences = 12 combinations
- Guaranteed bandwidth per class
- Drop precedence controls drop order during congestion
- Dùng cho: video, critical data, transactional

BE (Best Effort) — Default:
- No guarantee, FIFO
- Dùng cho: web browsing, email, non-critical
```

### 7.3 So Sánh IntServ vs DiffServ

| Tiêu chí | IntServ | DiffServ |
|----------|---------|----------|
| Granularity | Per-flow | Per-class (aggregate) |
| State in routers | Per-flow (millions!) | Per-class (~8) |
| Scalability | BAD (not Internet-scale) | GOOD |
| QoS guarantee | Hard (absolute) | Soft (relative/statistical) |
| Signaling | RSVP required | No signaling |
| Complexity | High (every router) | Low (edge marks, core simple) |
| Real-world usage | Rare | Standard (dùng khắp nơi) |
| Setup | Application requests | Network admin configures |

**Kết luận:** DiffServ là standard cho enterprise và Internet. IntServ gần như không dùng.

---

## 8. QoS Design Cho Enterprise Network

### 8.1 Cisco QoS Design Model (Phổ biến nhất)

```
4-class model (đơn giản):
┌────────────────────────────────────┐
│ Class       │ DSCP │ Queue    │ BW │
├────────────────────────────────────┤
│ Voice       │ EF   │ Priority │ 10%│
│ Video       │ AF41 │ CBWFQ    │ 25%│
│ Critical    │ AF21 │ CBWFQ    │ 25%│
│ Best-Effort │ CS0  │ Default  │ 40%│
└────────────────────────────────────┘

8-class model (enterprise standard):
┌────────────────────────────────────────────┐
│ Class          │ DSCP  │ Queue    │ BW     │
├────────────────────────────────────────────┤
│ VoIP           │ EF    │ Priority │ 10%    │
│ Video          │ AF41  │ Priority │ 23%    │
│ Voice Signaling│ CS3   │ CBWFQ    │ 2%     │
│ Critical Data  │ AF21  │ CBWFQ    │ 15%    │
│ Transactional  │ AF21  │ CBWFQ    │ 15%    │
│ Bulk Data      │ AF11  │ CBWFQ    │ 10%    │
│ Scavenger      │ CS1   │ CBWFQ    │ 1%     │
│ Best Effort    │ CS0   │ Default  │ 24%    │
└────────────────────────────────────────────┘
```

### 8.2 QoS Deployment — Ở Đâu Trên Mạng?

```
┌──────────────────────────────────────────────────┐
│                   WAN Edge                        │
│  Shape to WAN bandwidth, LLQ scheduling          │
│  ← QoS QUAN TRỌNG NHẤT ở đây (bottleneck!)     │
└───────────────────────┬──────────────────────────┘
                        │
┌───────────────────────┴──────────────────────────┐
│                 Distribution Layer                 │
│  CBWFQ + WRED, inter-VLAN QoS                   │
└───────────────────────┬──────────────────────────┘
                        │
┌───────────────────────┴──────────────────────────┐
│                  Access Layer                      │
│  Classification + Marking, Trust boundaries       │
│  ← QoS marking bắt đầu ở đây                    │
└──────────────────────────────────────────────────┘

Nguyên tắc:
- MARK ở access layer (closest to source)
- QUEUE/SCHEDULE ở WAN edge (bottleneck point)
- TRUST marks qua core (core chỉ fast-forward)
```

### 8.3 WAN Edge QoS Example

```
Scenario: WAN link 100 Mbps, cần protect voice/video

Policy-map WAN-EGRESS:
  class VOICE:
    priority 10000 (10 Mbps strict priority)
    
  class VIDEO:
    bandwidth 25000 (25 Mbps guaranteed)
    fair-queue
    random-detect dscp-based
    
  class CRITICAL-DATA:
    bandwidth 25000 (25 Mbps guaranteed)
    random-detect dscp-based
    
  class class-default:
    bandwidth 40000 (40 Mbps)
    fair-queue
    random-detect

Shape overall to 100 Mbps (match ISP contracted rate)
Then apply LLQ within the shaped rate

Kết quả:
- Voice: ALWAYS sent first, max 10 Mbps (policed)
- Video: 25 Mbps guaranteed, can burst higher if available
- Critical: 25 Mbps guaranteed
- Default: whatever's left (at least 40 Mbps)
```

---

## 9. QoS Trong Cloud và Modern Networks

### 9.1 QoS Trong Cloud (AWS, Azure, GCP)

```
Cloud QoS khác Enterprise QoS:
- Bạn KHÔNG kiểm soát network infrastructure của cloud provider
- Cloud provider tự quản lý QoS bên trong network họ
- Bạn chỉ kiểm soát: endpoint marking, application-level prioritization

AWS:
- VPC traffic: best-effort (no QoS guarantees between instances)
- Direct Connect: Hỗ trợ DSCP marking (bạn mark, AWS honor trong network)
- Transit Gateway: DSCP preserved
- Bên trong AZ: latency rất thấp (<1ms) → ít cần QoS

Azure:
- ExpressRoute: DSCP marking hỗ trợ
- Virtual WAN: QoS policies available

Xu hướng: Over-provision bandwidth thay vì complex QoS
(Cloud networks có bandwidth dư thừa → ít congestion → ít cần QoS)
```

### 9.2 SD-WAN và QoS

```
SD-WAN (Software-Defined WAN) thay đổi QoS approach:

Traditional QoS:
- 1 WAN link → phải ưu tiên traffic
- Complex policies trên router

SD-WAN QoS:
- Multiple WAN links (MPLS + Internet + 4G/5G)
- Application-aware routing
- Real-time path selection based on metrics

Ví dụ:
- Voice → MPLS (best quality, guaranteed SLA)
- Video → Internet link 1 (lowest latency measured)
- Backup → Internet link 2 (cheapest, highest bandwidth)
- If MPLS down → Voice auto-switches to Internet link with best jitter

SD-WAN platforms: Cisco Viptela/Meraki, VMware VeloCloud, Fortinet SD-WAN
```

### 9.3 DSCP và Container/Kubernetes

```
Kubernetes Pods có thể set DSCP trên outgoing traffic:
- Network Policies có thể mark traffic per-pod
- CNI plugins (Cilium, Calico) hỗ trợ DSCP marking
- Useful cho pod-to-pod QoS khi network congested

Ví dụ: Payment service pods → mark AF31 (high priority)
        Logging pods → mark CS1 (low priority/scavenger)
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 QoS Decision Tree

```
Mạng bạn có congestion không?
├── KHÔNG → QoS không cần thiết (over-provisioned)
└── CÓ → 
    Có real-time traffic (voice/video)?
    ├── CÓ → LLQ (Priority queue cho voice + CBWFQ cho còn lại)
    └── KHÔNG →
        Traffic có many classes cần differentiate?
        ├── CÓ → CBWFQ + WRED
        └── KHÔNG → WFQ (automatic fair sharing)
```

### 10.2 Key Takeaways

1. **QoS chỉ cần khi mạng NGHẼN** — mạng thừa bandwidth thì QoS vô nghĩa
2. **DSCP 6 bits** trong IP header = "nhãn ưu tiên" mà router đọc để quyết định
3. **EF = voice (ưu tiên tuyệt đối)**, AF = data quan trọng, CS0 = best-effort
4. **LLQ = CBWFQ + Priority Queue** — model phổ biến nhất cho enterprise
5. **Policing = DROP ngay** (bursty, dùng ở ingress), **Shaping = BUFFER rồi gửi từ từ** (smooth, dùng ở egress)
6. **WRED** tránh TCP global synchronization bằng random early drops
7. **DiffServ** (mark at edge, PHB at core) là standard — IntServ không scale
8. **Cloud networks** thường over-provision → ít cần QoS truyền thống

### 10.3 Tài Liệu Tham Khảo

- RFC 2474: Definition of the Differentiated Services Field (DSCP)
- RFC 2475: Architecture for Differentiated Services
- RFC 2597: Assured Forwarding PHB Group
- RFC 3246: Expedited Forwarding PHB
- RFC 4594: Configuration Guidelines for DiffServ Service Classes
- RFC 2205: RSVP (Resource ReSerVation Protocol)
- Cisco QoS Design Guide: "Enterprise QoS Solution Reference Network Design"
- Cisco CCNP ENCOR Study Guide: QoS chapters
- "End-to-End QoS Network Design" by Tim Szigeti (Cisco Press)
- ITU-T G.114: One-way transmission time (voice latency requirements)

---

*Bài viết tiếp theo: Network Design Patterns — Các mô hình thiết kế mạng phổ biến*
