---
layout: post
title: "TCP Deep Dive - Segment Structure, Seq/Ack, Window, Flow Control, Congestion Control (Slow Start, AIMD, CUBIC)"
date: 2026-06-01
categories: [networking]
tags: [tcp, transport-layer, congestion-control, flow-control, sliding-window, cubic, nagle]
---

## 1. Giới thiệu — Dịch vụ giao hàng "đảm bảo"

Hãy tưởng tượng bạn gửi **100 chương sách** cho nhà xuất bản:

**Gửi bưu điện thường (UDP):**
- Gửi hết 100 phong bì, không biết có đến không
- Không biết thứ tự có đúng không
- Mất 1 chương → mất luôn (không ai thông báo)
- Nhanh nhưng **không đảm bảo**

**Gửi chuyển phát đảm bảo (TCP):**
1. Trước khi gửi → **gọi điện** xác nhận: "Anh sẵn sàng nhận chưa?" (3-way handshake)
2. Mỗi chương đánh **số thứ tự**: Chương 1, 2, 3... (Sequence Number)
3. Người nhận **xác nhận**: "Đã nhận chương 1-5, gửi tiếp 6" (Acknowledgment)
4. Gửi **vài chương cùng lúc** (Window) — không đợi xác nhận từng cái
5. Nếu chương 3 mất → người nhận báo "thiếu 3!" → **gửi lại** (Retransmission)
6. Nếu đường tắc → **giảm tốc gửi** (Congestion Control)
7. Nếu người nhận bận → "Gửi chậm lại, tôi xử lý không kịp" (Flow Control)
8. Gửi xong → **gọi xác nhận**: "Hết rồi nhé, bye!" (Connection Termination)

**TCP** = Dịch vụ giao hàng **đáng tin cậy nhất** trên Internet:
- **Reliable** — đảm bảo mọi byte đến nơi
- **Ordered** — đúng thứ tự
- **Error-checked** — phát hiện lỗi
- **Flow-controlled** — không gửi quá nhanh
- **Congestion-controlled** — không làm tắc mạng

### TCP mang theo gì mỗi ngày?

| Ứng dụng | Port | Tại sao cần TCP |
|---|---|---|
| Web (HTTP/HTTPS) | 80/443 | Trang web phải đầy đủ, đúng thứ tự |
| Email (SMTP/IMAP) | 25/993 | Email không thể mất nửa chừng |
| File transfer (FTP/SFTP) | 21/22 | File phải nguyên vẹn |
| SSH | 22 | Commands phải đúng thứ tự |
| Database (MySQL/PostgreSQL) | 3306/5432 | Query/response phải chính xác |
| API calls (REST/gRPC) | 443/50051 | Request-response reliability |

---

## 2. TCP Segment Structure — Cấu trúc chi tiết

### Phép so sánh — Phong bì thư chuyên nghiệp

Mỗi TCP segment giống một **phong bì thư** có nhiều thông tin:
- Người gửi/nhận (ports)
- Số thứ tự thư (sequence)
- Xác nhận đã nhận (acknowledgment)
- Dung lượng hòm thư còn trống (window)
- Kiểm tra lỗi (checksum)
- Các flag đặc biệt (SYN, ACK, FIN...)

### TCP Header Format (RFC 9293)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Source Port (16)          │      Destination Port (16)    │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                    Sequence Number (32 bits)                       │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                 Acknowledgment Number (32 bits)                    │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│ Data  │     │U│A│P│R│S│F│                                         │
│Offset │Rsrvd│R│C│S│S│Y│I│          Window Size (16 bits)          │
│ (4)   │(4)  │G│K│H│T│N│N│                                         │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Checksum (16)             │      Urgent Pointer (16)      │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                    Options (0-40 bytes, variable)                  │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                    Data (Payload)                                  │
└─────────────────────────────────────────────────────────────────────┘
        Minimum header: 20 bytes | Maximum: 60 bytes
```

### Chi tiết từng trường

#### Source Port & Destination Port — 16 bits mỗi cái

```
- Identify application/process gửi và nhận
- Range: 0-65535
  - Well-known: 0-1023 (cần root/admin)
  - Registered: 1024-49151
  - Ephemeral: 49152-65535 (client tự chọn random)
  
Ví dụ:
  Client web browser: src=52847 (random), dst=443 (HTTPS server)
  Server response:    src=443, dst=52847
```

#### Sequence Number — 32 bits

**Byte counter** — đếm từng byte data đã gửi:
```
Initial Sequence Number (ISN): Random (ví dụ: 1000)
Segment 1: Seq=1000, Data="Hello" (5 bytes)
Segment 2: Seq=1005, Data="World" (5 bytes)  ← 1000+5
Segment 3: Seq=1010, Data="!!!"   (3 bytes)  ← 1005+5

Seq # = ISN + số bytes đã gửi trước đó
```

**Tại sao random ISN?**
- Tránh packet từ connection cũ bị nhầm lẫn
- Security — khó guess (tránh TCP hijacking)
- RFC 6528: ISN nên dùng clock-based + random component

#### Acknowledgment Number — 32 bits

**Xác nhận** — "Tôi đã nhận tất cả đến byte X-1, cho tôi byte X tiếp":
```
A gửi: Seq=1000, "Hello" (5 bytes)
B reply: Ack=1005 → "Tôi nhận hết đến byte 1004, cho tôi byte 1005"

A gửi: Seq=1005, "World" (5 bytes)
B reply: Ack=1010 → "Nhận hết đến 1009, cho tôi 1010"

Ack = Next expected byte number (cumulative)
```

#### Data Offset — 4 bits

Cho biết TCP header dài bao nhiêu (tính bằng 32-bit words):
```
Minimum: 5 (= 20 bytes, no options)
Maximum: 15 (= 60 bytes, full options)
```

#### Flags — 6 bits (+ 4 reserved)

| Flag | Tên | Mô tả |
|---|---|---|
| **URG** | Urgent | Urgent Pointer field hợp lệ |
| **ACK** | Acknowledgment | Ack Number field hợp lệ |
| **PSH** | Push | Receiver nên deliver data ngay cho app (không buffer) |
| **RST** | Reset | Abort connection ngay lập tức |
| **SYN** | Synchronize | Bắt đầu connection (set ISN) |
| **FIN** | Finish | Kết thúc gửi data (graceful close) |

Thêm flags (RFC 3168, 8311):

| Flag | Tên | Mô tả |
|---|---|---|
| **CWR** | Congestion Window Reduced | Sender đã giảm window |
| **ECE** | ECN-Echo | Receiver thấy congestion |
| **NS** | Nonce Sum | ECN protection |

#### Window Size — 16 bits

```
"Tôi còn chỗ nhận thêm X bytes" (receiver advertised window):
- Range: 0-65535 bytes (without scaling)
- Window = 0: "STOP! Tôi hết buffer!" (zero window → sender pause)
- With Window Scaling option: up to 1 GB (2^30)
```

#### Checksum — 16 bits

```
One's complement checksum covering:
- TCP pseudo-header (source IP, dest IP, protocol, TCP length)
- TCP header
- TCP data

→ Detect corruption trong transit
```

#### Urgent Pointer — 16 bits

```
Offset từ Sequence Number đến urgent data
Chỉ valid khi URG flag = 1
Ít dùng trong practice (legacy, Telnet interrupt)
```

### TCP Options (quan trọng!)

| Option | Kind | Length | Mô tả |
|---|---|---|---|
| MSS | 2 | 4 bytes | Maximum Segment Size (thường 1460) |
| Window Scale | 3 | 3 bytes | Scale factor 0-14 (2^scale) |
| SACK Permitted | 4 | 2 bytes | Selective ACK enabled |
| SACK | 5 | Variable | Blocks received out-of-order |
| Timestamps | 8 | 10 bytes | RTT measurement, PAWS |
| No-Op | 1 | 1 byte | Padding/alignment |
| End of List | 0 | 1 byte | No more options |

```
Ví dụ SYN packet options:
  MSS: 1460 (1500 MTU - 20 IP - 20 TCP)
  Window Scale: 7 (window = advertised × 2^7 = ×128)
  SACK Permitted: Yes
  Timestamps: TSval=12345, TSecr=0
```

---

## 3. Sequence & Acknowledgment — Cơ chế đánh số

### Phép so sánh — Giao dịch chuyển tiền

Mỗi giao dịch ngân hàng:
- **Số giao dịch** (Sequence) — mã duy nhất
- **Xác nhận** (Ack) — "Đã nhận GD #X, chờ GD #X+1"
- **Cumulative** — "Đã nhận TẤT CẢ đến GD #X"

### Ví dụ chi tiết — HTTP request

```
Client (10.0.1.1:52847) ←→ Server (93.184.216.34:443)

=== 3-Way Handshake ===

1. Client → SYN:
   Seq=100, Ack=0, SYN flag
   "Tôi muốn connect, bắt đầu từ byte 100"

2. Server → SYN-ACK:
   Seq=300, Ack=101, SYN+ACK flags
   "OK! Tôi bắt đầu từ 300, đã nhận SYN của bạn (Ack=101)"

3. Client → ACK:
   Seq=101, Ack=301, ACK flag
   "Nhận SYN của server (Ack=301). Connection established!"

=== Data Transfer ===

4. Client → "GET / HTTP/1.1\r\n..." (200 bytes):
   Seq=101, Ack=301, PSH+ACK
   Data: 200 bytes

5. Server → ACK:
   Seq=301, Ack=301 (= 101+200)
   "Đã nhận 200 bytes, cho tôi byte 301 tiếp"

6. Server → Response part 1 (1000 bytes):
   Seq=301, Ack=301, PSH+ACK
   Data: 1000 bytes

7. Server → Response part 2 (500 bytes):
   Seq=1301, Ack=301
   Data: 500 bytes

8. Client → ACK:
   Seq=301, Ack=1801 (= 301+1000+500)
   "Đã nhận 1500 bytes từ server"
```

### Selective Acknowledgment (SACK)

```
Problem với Cumulative ACK:
  Sender gửi: Seg 1, 2, 3, 4, 5
  Receiver nhận: 1, 2, _, 4, 5 (segment 3 lost!)
  
  Cumulative ACK: Ack=3 → "Nhận đến 2, cho tôi 3"
  Sender không biết: 4 và 5 đã đến! → retransmit 3, 4, 5 (lãng phí!)

SACK giải quyết:
  Receiver gửi: Ack=3, SACK blocks: [4-5 received]
  → Sender biết: chỉ cần retransmit segment 3!
  → Hiệu quả hơn nhiều trên high-latency/lossy links
```

```
SACK Example:
  Bytes sent: 1000-1999, 2000-2999, 3000-3999, 4000-4999, 5000-5999
  Bytes received: 1000-2999 ✓, [3000-3999 LOST], 4000-5999 ✓

  ACK from receiver:
    Ack=3000 (cumulative: received up to 2999)
    SACK: Left=4000, Right=6000 (also received 4000-5999)
    
  Sender: "Ah, chỉ mất 3000-3999, retransmit that only!"
```

---

## 4. Flow Control — Kiểm soát luồng (Receiver-based)

### Phép so sánh — Bồn nước và vòi

- **Sender** = Vòi nước (tốc độ gửi)
- **Receiver buffer** = Bồn chứa (capacity)
- **Window** = Mực nước còn trống trong bồn
- **Application** = Ống thoát (tốc độ app đọc data)

```
Nếu vòi (sender) chảy nhanh hơn ống thoát (app xử lý):
  → Bồn đầy (buffer full)
  → Receiver thông báo: "Window = 0" (HẾT CHỖ!)
  → Sender DỪNG gửi
  → App xử lý xong 1 phần → Window > 0
  → Receiver thông báo: "Window = 5000" (CÓ CHỖ LẠI!)
  → Sender gửi tiếp
```

### Sliding Window Protocol

```
Sender perspective:
                         Window (có thể gửi)
                    ┌──────────────────────────┐
  ████████████████│░░░░░░░░░░░░░░░░░░░░░░░░░│○○○○○○○○○○
  ↑               ↑                          ↑          ↑
  Đã gửi          Đã gửi nhưng              Có thể      Chưa gửi
  + đã ACK        chưa ACK                  gửi          (ngoài window)
  (hoàn thành)    (in-flight)               (window free)

Legend:
  ████ = Sent and ACKed (completed)
  ░░░░ = Sent but not yet ACKed (in-flight)
  ○○○○ = Can be sent (within window, not yet sent)
  
Window slides RIGHT as ACKs arrive:
  ACK nhận → ████ tăng → window dịch sang phải → gửi thêm data mới
```

### Ví dụ Window hoạt động

```
Receiver Window (rwnd) = 4000 bytes
MSS = 1000 bytes

Time 0: Sender có 10000 bytes to send
  Window: [_][_][_][_] = 4 segments available
  
Time 1: Send seg 1,2,3,4 (fill window)
  In-flight: [1][2][3][4], Window full!
  
Time 2: ACK for seg 1 arrives
  Window slides: [2][3][4][_] → can send seg 5
  Send seg 5
  
Time 3: ACK for seg 2,3 arrives
  Window slides: [4][5][_][_] → can send seg 6,7
  Send seg 6,7
  
Time 4: Receiver app busy, buffer filling up
  Receiver advertises Window = 2000 (shrink!)
  Sender adjusts: only 2 segments in-flight allowed
```

### Zero Window và Window Probe

```
Khi receiver buffer đầy:

1. Receiver: "Window = 0" → Sender STOP!
2. Sender: Chờ...
3. Problem: Nếu Window Update packet bị mất → DEADLOCK!
   (Sender chờ mãi, Receiver nghĩ sender đã biết có window)
4. Solution: Window Probe
   - Sender gửi probe packet mỗi RTO (nhỏ, 1 byte)
   - Receiver reply với current window
   - Nếu window > 0 → sender resume
   - Exponential backoff: 1s, 2s, 4s, 8s... (max 120s)
```

### Silly Window Syndrome (SWS)

```
Problem:
  Receiver có 1 byte free → advertise window=1
  Sender gửi 1 byte → 40 bytes header + 1 byte data = 41 bytes
  Receiver xử lý 1 byte → free 1 byte → advertise window=1
  → Repeat: gửi 1 byte/lần! Cực kỳ inefficient!
  
  Overhead: 40 bytes header / 1 byte data = 4000% overhead!

Solutions:
  Receiver-side (Clark's algorithm):
    - Không advertise window nhỏ
    - Chỉ advertise khi: window ≥ MSS hoặc ≥ buffer/2
    
  Sender-side (Nagle's algorithm):
    - Không gửi segment nhỏ khi có data in-flight chưa ACK
    - Buffer small writes → gửi 1 segment lớn
```

---

## 5. Congestion Control — Kiểm soát tắc nghẽn (Network-based)

### Phép so sánh — Đường cao tốc giờ cao điểm

- **Flow Control** = Bạn giảm tốc vì **bãi đậu xe** ở đích đầy (receiver buffer)
- **Congestion Control** = Bạn giảm tốc vì **đường cao tốc** đang tắc (network congested)

Dù bãi đậu xe (receiver) còn chỗ, nếu đường tắc thì vẫn phải đi chậm!

### Hai giới hạn của Sender

```
Effective Window = min(rwnd, cwnd)

  rwnd = Receiver Window (flow control — do receiver advertise)
  cwnd = Congestion Window (congestion control — sender tự tính)

Sender không được gửi quá min(rwnd, cwnd) bytes chưa ACK!
```

### Các phase của TCP Congestion Control

```
cwnd
  │
  │         ╱ Congestion Avoidance
  │        ╱  (linear increase)
  │       ╱
  │      ╱────── ssthresh (slow start threshold)
  │     ╱│
  │    ╱ │
  │   ╱  │      ← Slow Start
  │  ╱   │        (exponential increase)
  │ ╱    │
  │╱     │
  ├──────┼──────────────────────────────→ time
  1      ssthresh
```

### Phase 1: Slow Start (Exponential Growth)

```
Mặc dù gọi "slow" nhưng thực ra TĂNG RẤT NHANH (exponential):

Round 1: cwnd = 1 MSS → gửi 1 segment
  ↓ ACK received → cwnd += 1 MSS per ACK
Round 2: cwnd = 2 MSS → gửi 2 segments
  ↓ 2 ACKs → cwnd += 2
Round 3: cwnd = 4 MSS → gửi 4 segments
  ↓ 4 ACKs → cwnd += 4
Round 4: cwnd = 8 MSS → gửi 8 segments
  ...
  
Pattern: cwnd doubles every RTT = EXPONENTIAL growth
  1 → 2 → 4 → 8 → 16 → 32 → 64 → ...

Dừng khi:
  - cwnd ≥ ssthresh → chuyển sang Congestion Avoidance
  - Packet loss detected → reduce cwnd
  - rwnd reached → flow control limits
```

**Tại sao "slow" start?** So với việc gửi tất cả ngay lập tức (old TCP), bắt đầu từ 1 là "slow". Nhưng tăng exponential nên nhanh chóng tận dụng bandwidth.

### Phase 2: Congestion Avoidance (AIMD — Additive Increase, Multiplicative Decrease)

```
Khi cwnd ≥ ssthresh:

Additive Increase:
  Mỗi RTT: cwnd += 1 MSS (linear growth)
  Tương đương: mỗi ACK, cwnd += MSS × MSS / cwnd
  
  Ví dụ: cwnd=10 MSS
    Mỗi ACK: cwnd += 1/10 MSS
    Sau 10 ACKs (1 RTT): cwnd = 11 MSS

Multiplicative Decrease (on loss):
  Packet loss detected:
    ssthresh = cwnd / 2
    cwnd = 1 MSS (Tahoe) hoặc cwnd/2 (Reno)
    
  → "Saw-tooth" pattern:
    cwnd tăng dần (linear) → loss → giảm mạnh → tăng lại → loss...
```

### TCP Tahoe vs Reno vs CUBIC

#### TCP Tahoe (1988)

```
On timeout OR 3 duplicate ACKs:
  ssthresh = cwnd / 2
  cwnd = 1 MSS
  → Quay lại Slow Start (từ đầu!)
  
Nhược điểm: Quá aggressive — giảm cwnd về 1 quá lãng phí
```

#### TCP Reno (1990)

```
On timeout:
  ssthresh = cwnd / 2
  cwnd = 1 MSS
  → Slow Start (like Tahoe)

On 3 duplicate ACKs (Fast Retransmit + Fast Recovery):
  ssthresh = cwnd / 2
  cwnd = ssthresh + 3 MSS  ← Không về 1! (Fast Recovery)
  → Congestion Avoidance (skip slow start)
  
Cải tiến: Phân biệt giữa "light loss" (3 dup ACKs) vs "severe loss" (timeout)
```

#### TCP CUBIC (2008 — Linux default, RFC 8312)

```
CUBIC = Congestion control algorithm MẶC ĐỊNH của Linux!

Đặc điểm:
- KHÔNG dùng AIMD linear increase
- Dùng CUBIC function (hàm bậc 3) để tăng cwnd
- Window growth = f(time since last loss)
- Aggressive gần Wmax cũ, conservative khi vượt

CUBIC function:
  W(t) = C × (t - K)³ + Wmax

  Wmax = cwnd lúc bị loss lần cuối
  K = ∛(Wmax × β / C)  (time to reach Wmax again)
  C = scaling constant (0.4)
  β = multiplicative decrease factor (0.7)
  t = time since last loss

On loss:
  Wmax = cwnd (remember current window)
  cwnd = cwnd × β (= cwnd × 0.7, giảm 30%)
  
Recovery:
  - Ban đầu tăng nhanh (concave — dưới Wmax)
  - Gần Wmax: tăng chậm lại (plateau — thận trọng)
  - Vượt Wmax: tăng nhanh dần (convex — explore bandwidth mới)
```

```
CUBIC Window Growth:
cwnd
  │
  │                                    ╱ ← Convex (explore new)
  │                                  ╱
  │         Wmax ─────────────────╱─── ← Plateau (cautious)
  │              ╲              ╱
  │               ╲           ╱ ← Concave (quick recovery)
  │                ╲        ╱
  │                 ╲     ╱
  │                  ╲  ╱
  │                   ╲╱ ← Loss point
  │                   │
  ├───────────────────┼──────────────────→ time
                   loss
```

### So sánh Congestion Control Algorithms

| Algorithm | Year | Default OS | Decrease | Increase | RTT fairness |
|---|---|---|---|---|---|
| Tahoe | 1988 | — | cwnd=1 | Slow start + AIMD | Poor |
| Reno | 1990 | — | cwnd/2 | AIMD | Fair |
| NewReno | 1999 | — | cwnd/2 | AIMD (better recovery) | Fair |
| **CUBIC** | 2008 | **Linux** | cwnd×0.7 | Cubic function | Good |
| **BBR** | 2016 | Google | Model-based | Model-based | Very good |
| DCTCP | 2010 | DC | ECN-based | ECN-based | DC only |
| Compound | 2006 | Windows | Hybrid | Delay+Loss | Good |

### BBR (Bottleneck Bandwidth and RTT) — Google

```
BBR (RFC draft) — fundamentally different approach:

Traditional (CUBIC, Reno): Loss-based
  - Increase until packet loss → reduce → increase again
  - Problem: buffers absorb → high latency before loss signal

BBR: Model-based
  - Continuously estimates:
    BtlBw = bottleneck bandwidth (max delivery rate)
    RTprop = minimum RTT (propagation delay)
  - Target sending rate = BtlBw × RTprop
  - Doesn't need loss to detect congestion!
  
Phases:
  1. Startup (like slow start) — find BtlBw quickly
  2. Drain — reduce queue built up during startup
  3. ProbeBW — steady state, periodically probe for more BW
  4. ProbeRTT — periodically reduce sending to measure RTprop

Advantage: Lower latency, better throughput on lossy links
Linux: sysctl net.ipv4.tcp_congestion_control=bbr
```

---

## 6. Nagle's Algorithm — Tối ưu small packets

### Phép so sánh — Gửi thư nhóm

**Không có Nagle:**
- Mỗi chữ viết xong → cho vào 1 phong bì riêng → gửi ngay
- "H" → gửi, "e" → gửi, "l" → gửi, "l" → gửi, "o" → gửi
- 5 phong bì cho 5 chữ cái! Overhead khổng lồ!

**Có Nagle:**
- Viết xong chữ → **đợi** thêm chút
- Gom đủ hoặc nhận ACK → **gửi 1 lần**: "Hello"
- 1 phong bì cho 5 chữ! Efficient!

### Nagle's Algorithm (RFC 896)

```python
# Pseudocode:
if data_to_send:
    if len(data) >= MSS:
        send(data)              # Full segment → send immediately
    elif no_unacked_data:
        send(data)              # Nothing in-flight → send immediately
    else:
        buffer(data)            # Wait for ACK or more data
        # Send when: ACK arrives OR buffer reaches MSS
```

```
Quy tắc đơn giản:
  - Nếu có thể gửi full MSS → gửi ngay (đủ lớn, efficient)
  - Nếu KHÔNG có data in-flight → gửi ngay (nothing waiting)
  - Nếu CÓ data in-flight → BUFFER (đợi ACK hoặc thêm data)
  
Kết quả:
  - Tối đa 1 small segment in-flight tại mọi thời điểm
  - Giảm số lượng tiny packets trên network
  - Overhead giảm đáng kể
```

### Khi nào TẮT Nagle?

```
Nagle GIAO với Delayed ACK = Latency problem!

Delayed ACK: Receiver đợi 200ms trước khi gửi ACK (hoping to piggyback)
Nagle: Sender đợi ACK trước khi gửi segment nhỏ tiếp

→ Interactive apps bị lag 200ms mỗi keystroke!

Disable Nagle (TCP_NODELAY):
  - Real-time games (mỗi input phải gửi ngay)
  - SSH/Telnet (mỗi keystroke phải responsive)
  - Trading systems (microsecond matters)
  - Live streaming control
```

```c
// Disable Nagle in code:
int flag = 1;
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
```

```python
# Python:
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_NODELAY, 1)
```

---

## 7. TCP Keep-Alive — Giữ connection sống

### Phép so sánh — Gọi điện kiểm tra

Bạn thuê phòng khách sạn nhưng ra ngoài cả ngày:
- Khách sạn **không biết** bạn còn ở hay đã trốn
- Giải pháp: Mỗi 2 tiếng, lễ tân **gọi điện**: "Anh còn dùng phòng không?"
  - "Còn!" → OK, giữ phòng
  - Không trả lời → gọi thêm vài lần → dọn phòng cho người khác

### TCP Keep-Alive (RFC 1122)

```
Mục đích:
1. Detect dead peer (crash, network failure)
2. Prevent NAT/firewall timeout (giữ mapping active)
3. Detect half-open connections

Parameters (Linux defaults):
  tcp_keepalive_time = 7200  (2 hours — start probing after 2h idle)
  tcp_keepalive_intvl = 75   (75 seconds between probes)
  tcp_keepalive_probes = 9   (9 failed probes → connection dead)
  
  Total detection time: 7200 + 75×9 = 7875 seconds ≈ 2.2 hours!
  (rất chậm — thường override cho production)
```

### Cấu hình Keep-Alive

```bash
# Linux system-wide:
sysctl -w net.ipv4.tcp_keepalive_time=600    # Start after 10 min
sysctl -w net.ipv4.tcp_keepalive_intvl=30    # Probe every 30s
sysctl -w net.ipv4.tcp_keepalive_probes=5    # 5 failures → dead

# Total detection: 600 + 30×5 = 750 seconds = 12.5 minutes
```

```c
// Per-socket (C):
int keepalive = 1;
int keepidle = 60;     // Start probing after 60s idle
int keepintvl = 10;    // Probe every 10s
int keepcnt = 5;       // 5 failed probes → dead

setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE, &keepalive, sizeof(int));
setsockopt(sock, IPPROTO_TCP, TCP_KEEPIDLE, &keepidle, sizeof(int));
setsockopt(sock, IPPROTO_TCP, TCP_KEEPINTVL, &keepintvl, sizeof(int));
setsockopt(sock, IPPROTO_TCP, TCP_KEEPCNT, &keepcnt, sizeof(int));
```

### Application-layer Keep-Alive vs TCP Keep-Alive

| Aspect | TCP Keep-Alive | App Keep-Alive |
|---|---|---|
| Layer | Transport (OS) | Application |
| Content | Empty ACK (0 bytes data) | "ping"/"pong" messages |
| Interval | Usually long (minutes) | Short (seconds) |
| Customization | Limited (3 params) | Fully customizable |
| Detection speed | Slow (minutes) | Fast (seconds) |
| Examples | TCP SO_KEEPALIVE | HTTP/2 PING, WebSocket ping/pong, gRPC keepalive |

---

## 8. TCP Retransmission — Cơ chế gửi lại

### Phép so sánh — Gửi lại thư bị thất lạc

Bạn gửi thư quan trọng:
1. Gửi thư + đặt **hẹn giờ** (timer)
2. Nếu nhận xác nhận trước hẹn → OK, huỷ timer
3. Nếu hẹn giờ hết mà chưa nhận xác nhận → **GỬI LẠI**!
4. Lần 2 hẹn giờ **dài hơn** (exponential backoff)
5. Sau nhiều lần thất bại → **bỏ cuộc** (connection timeout)

### RTO — Retransmission Timeout

```
RTO (Retransmission Timeout) = thời gian đợi trước khi retransmit

Tính RTO dựa trên RTT measurements (RFC 6298):

1. Smoothed RTT (SRTT):
   SRTT = (1-α) × SRTT + α × SampleRTT     (α = 1/8)

2. RTT Variance (RTTVAR):
   RTTVAR = (1-β) × RTTVAR + β × |SRTT - SampleRTT|   (β = 1/4)

3. RTO:
   RTO = SRTT + max(G, 4 × RTTVAR)    (G = clock granularity)
   RTO = max(RTO, 1 second)            (minimum 1s)

Ví dụ:
  RTT samples: 50ms, 55ms, 48ms, 52ms
  SRTT ≈ 51ms
  RTTVAR ≈ 3ms
  RTO = 51 + 4×3 = 63ms → round up to 1000ms (minimum)
  
  Thực tế trên LAN: RTO ≈ 200ms-1s
  Thực tế WAN: RTO ≈ 1-3s
```

### Exponential Backoff

```
Mỗi lần retransmit timeout → RTO × 2:

Attempt 1: RTO = 1s → timeout → retransmit
Attempt 2: RTO = 2s → timeout → retransmit  
Attempt 3: RTO = 4s → timeout → retransmit
Attempt 4: RTO = 8s → timeout → retransmit
...
Attempt N: RTO = 2^(N-1) seconds
Maximum: thường 120s (2 minutes)
Max retries: thường 15 (tcp_retries2)

Total time before giving up:
  1+2+4+8+16+32+64+120+120+... ≈ 15-30 minutes
```

### Fast Retransmit (3 Duplicate ACKs)

```
Không đợi timeout — retransmit ngay khi thấy 3 duplicate ACKs:

Sender gửi: Seg1, Seg2, Seg3, Seg4, Seg5
                           ↑ LOST!

Receiver nhận:
  Seg1 → ACK 2 (expect byte 2)
  Seg2 → ACK 3 (expect byte 3)
  Seg4 → ACK 3 (duplicate! I still need 3)  ← dup ACK 1
  Seg5 → ACK 3 (duplicate! I still need 3)  ← dup ACK 2
  Seg6 → ACK 3 (duplicate! I still need 3)  ← dup ACK 3

Sender: 3 dup ACKs for seq 3 → Seg3 LOST!
  → Fast Retransmit Seg3 ngay (không đợi RTO timeout)
  → Tiết kiệm thời gian đáng kể!
```

### Tail Loss Problem

```
Problem: Last segment(s) lost → no subsequent ACKs → no dup ACKs!
  Phải đợi full RTO timeout (seconds!)

Solutions:
  - Tail Loss Probe (TLP) — RFC 8985
    Sau 2×SRTT without ACK → send probe
    Triggers ACK/SACK → enables fast retransmit
  
  - RACK (Recent Acknowledgment) — RFC 8985
    Uses timestamp ordering instead of sequence-based
    Can detect tail loss faster
```

---

## 9. TCP Performance Tuning — Tối ưu cho Production

### Bandwidth-Delay Product (BDP)

```
BDP = Bandwidth × RTT
    = "Dung lượng đường ống" (bytes in-flight cần để fill pipe)

Ví dụ:
  Link 1 Gbps, RTT 50ms:
  BDP = 1,000,000,000 bits/s × 0.050s / 8 = 6,250,000 bytes = 6.25 MB!
  
  → Window phải ≥ 6.25 MB để tận dụng hết bandwidth!
  → Default window 64KB (không scale) = chỉ dùng 1% bandwidth!
  → CẦN Window Scaling option!
```

### Linux TCP Tuning

```bash
# === Buffer sizes ===
# Minimum, Default, Maximum (bytes)
sysctl -w net.core.rmem_max=16777216          # Max receive buffer
sysctl -w net.core.wmem_max=16777216          # Max send buffer
sysctl -w net.ipv4.tcp_rmem="4096 131072 16777216"  # TCP receive buffer
sysctl -w net.ipv4.tcp_wmem="4096 16384 16777216"   # TCP send buffer

# === Congestion Control ===
sysctl -w net.ipv4.tcp_congestion_control=bbr  # Use BBR (Google)
# or: cubic (default), reno, vegas, htcp

# === Window Scaling ===
sysctl -w net.ipv4.tcp_window_scaling=1        # Enable (default)

# === SACK ===
sysctl -w net.ipv4.tcp_sack=1                  # Enable SACK (default)
sysctl -w net.ipv4.tcp_dsack=1                 # Duplicate SACK

# === Timestamps ===
sysctl -w net.ipv4.tcp_timestamps=1            # Enable (RTT measurement)

# === Fast Open ===
sysctl -w net.ipv4.tcp_fastopen=3              # Enable TFO (client+server)

# === SYN queue ===
sysctl -w net.ipv4.tcp_max_syn_backlog=65535   # SYN queue size
sysctl -w net.core.somaxconn=65535             # Listen backlog

# === TIME_WAIT ===
sysctl -w net.ipv4.tcp_tw_reuse=1              # Reuse TIME_WAIT sockets
sysctl -w net.ipv4.tcp_fin_timeout=30          # FIN_WAIT_2 timeout
```

### Tối ưu cho specific scenarios

```bash
# High-throughput (CDN, file server):
net.ipv4.tcp_congestion_control = bbr
net.core.rmem_max = 67108864           # 64MB
net.core.wmem_max = 67108864
net.ipv4.tcp_rmem = 4096 1048576 67108864
net.ipv4.tcp_wmem = 4096 1048576 67108864

# Low-latency (trading, gaming):
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_nodelay = 1               # Application should set TCP_NODELAY

# High-connection (web server):
net.ipv4.tcp_max_syn_backlog = 65535
net.core.somaxconn = 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.core.netdev_max_backlog = 65535
```

---

## 10. Troubleshooting TCP và Tools

### TCP Problems và Symptoms

| Problem | Symptoms | Tool |
|---|---|---|
| Packet loss | Retransmissions, slow speed | tcpdump, Wireshark |
| High RTT | Slow initial response | ping, traceroute |
| Small window | Low throughput despite good link | Wireshark window graph |
| Buffer bloat | High latency under load | ping during transfer |
| Nagle + Delayed ACK | 200ms lag per operation | tcpdump timestamp |
| MTU mismatch | Connection hangs on large data | ping -s 1472 -M do |
| SYN flood | Connection refused, high CPU | netstat -s, ss |

### Essential Tools

```bash
# === ss (socket statistics) — faster than netstat ===
ss -tnp                    # All TCP connections with process
ss -tlnp                   # Listening TCP sockets
ss -ti                     # TCP internal info (cwnd, rtt, retrans)

# Output: 
# cubic wscale:7,7 rto:204 rtt:1.5/0.5 ato:40 mss:1460
# cwnd:10 ssthresh:7 bytes_sent:5432 bytes_acked:5432
# retrans:0/2 rcv_space:29200

# === tcpdump — packet capture ===
tcpdump -i eth0 'tcp port 443' -nn            # Capture HTTPS
tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0' # SYN packets only
tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0' # RST packets

# === TCP statistics ===
netstat -s | grep -i tcp    # TCP statistics
cat /proc/net/tcp           # Raw TCP socket info
nstat -az | grep Tcp        # Detailed TCP counters

# Key metrics to watch:
# TcpRetransSegs — retransmissions (network loss)
# TcpExtTCPTimeouts — RTO timeouts (severe loss)
# TcpExtTCPOFOMerge — out-of-order segments
# TcpInErrs — incoming errors
```

### Wireshark TCP Analysis

```
Wireshark Expert Info for TCP:
  [TCP Retransmission] — segment retransmitted (RTO expired)
  [TCP Fast Retransmission] — retransmit after 3 dup ACKs
  [TCP Duplicate ACK] — receiver asking for same seq again
  [TCP Zero Window] — receiver buffer full
  [TCP Window Update] — receiver advertising new window
  [TCP Keep-Alive] — probe packet
  [TCP RST] — connection reset
  [TCP Out-of-Order] — segment arrived out of sequence

Useful Wireshark filters:
  tcp.analysis.retransmission
  tcp.analysis.duplicate_ack
  tcp.analysis.zero_window
  tcp.analysis.window_full
  tcp.flags.reset == 1
```

### Tổng kết TCP

```
TCP Core Mechanisms:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Reliability:                                           │
│  • Sequence numbers (byte counting)                    │
│  • Acknowledgments (cumulative + SACK)                 │
│  • Retransmission (RTO + Fast Retransmit)             │
│  • Checksum (error detection)                          │
│                                                         │
│  Flow Control (receiver-based):                        │
│  • Sliding Window (rwnd)                               │
│  • Zero Window probe                                   │
│  • Silly Window Syndrome prevention                    │
│                                                         │
│  Congestion Control (network-based):                   │
│  • Slow Start (exponential)                            │
│  • Congestion Avoidance (linear — AIMD)               │
│  • Fast Retransmit / Fast Recovery                     │
│  • Algorithms: CUBIC (Linux), BBR (Google)             │
│                                                         │
│  Optimization:                                          │
│  • Nagle's Algorithm (coalesce small writes)           │
│  • Delayed ACK (reduce ACK traffic)                    │
│  • TCP_NODELAY (disable Nagle for real-time)           │
│  • Window Scaling (large BDP support)                  │
│  • TCP Fast Open (reduce handshake latency)            │
│                                                         │
│  Key Formula:                                           │
│  Effective Window = min(rwnd, cwnd)                    │
│  BDP = Bandwidth × RTT                                 │
│  Buffer ≥ BDP for full utilization                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

*Tài liệu tham khảo:*
- RFC 9293 — Transmission Control Protocol (TCP) — obsoletes RFC 793
- RFC 5681 — TCP Congestion Control
- RFC 8312 — CUBIC for Fast Long-Distance Networks
- RFC 6298 — Computing TCP's Retransmission Timer
- RFC 7323 — TCP Extensions for High Performance (Window Scale, Timestamps)
- RFC 2018 — TCP Selective Acknowledgment Options (SACK)
- RFC 896 — Congestion Control in IP/TCP Internetworks (Nagle)
- RFC 8985 — The RACK-TLP Loss Detection Algorithm
- RFC 1122 — Requirements for Internet Hosts (Keep-Alive)
- Linux TCP tuning — kernel.org documentation
- Wireshark TCP Analysis documentation
