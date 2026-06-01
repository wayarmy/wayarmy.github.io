---
layout: post
title: "UDP Deep Dive - Datagram Structure, Use Cases và UDP-Based Protocols"
date: 2026-06-01
categories: [networking]
tags: [udp, transport-layer, datagram, dns, dhcp, rtp, quic, gaming]
---

## 1. Giới thiệu — Gửi bưu thiếp vs Gửi thư bảo đảm

Hãy tưởng tượng 2 cách gửi tin nhắn:

**Thư bảo đảm (TCP):**
1. Gọi điện trước: "Tôi sẽ gửi thư nhé" (handshake)
2. Gửi thư → chờ xác nhận "đã nhận"
3. Nếu mất → gửi lại
4. Đánh số thứ tự để đọc đúng order
5. **Chắc chắn đến**, nhưng **chậm** và **phức tạp**

**Bưu thiếp (UDP):**
1. Viết bưu thiếp → **bỏ vào hòm thư** → xong!
2. Không gọi trước, không chờ xác nhận
3. Mất thì mất — không biết, không gửi lại
4. Không số thứ tự — đến trước đến sau tuỳ duyên
5. **Nhanh, đơn giản**, nhưng **không đảm bảo**

**UDP (User Datagram Protocol)** = Gửi bưu thiếp:
- **Connectionless** — không cần thiết lập kết nối trước
- **Unreliable** — không đảm bảo delivery
- **Unordered** — không đảm bảo thứ tự
- **No flow control** — gửi bao nhiêu tuỳ thích
- **Low overhead** — header chỉ 8 bytes (vs TCP 20-60 bytes)
- **Fast** — không có handshake, không có ACK wait

### Tại sao UDP vẫn quan trọng?

| Trường hợp | Tại sao UDP tốt hơn TCP |
|---|---|
| Video call/Livestream | Frame cũ mất = bỏ qua (retransmit = lag!) |
| Online gaming | Input phải real-time, frame cũ = useless |
| DNS query | 1 request, 1 response, nhanh hơn 3-way handshake |
| VoIP | Giọng nói real-time, delay > 150ms = khó nghe |
| IoT sensors | Gửi data nhỏ liên tục, không cần reliability |
| QUIC/HTTP3 | UDP + custom reliability = better than TCP! |

### UDP vs TCP — So sánh nhanh

| Feature | TCP | UDP |
|---|---|---|
| Connection | Connection-oriented (handshake) | Connectionless |
| Reliability | Guaranteed delivery | Best-effort |
| Ordering | In-order delivery | No ordering |
| Flow control | Yes (sliding window) | No |
| Congestion control | Yes (CUBIC/BBR) | No (app phải tự lo) |
| Header size | 20-60 bytes | **8 bytes** |
| Speed | Slower (overhead) | **Faster** |
| Overhead | High (state, timers, buffers) | **Minimal** |
| Broadcast/Multicast | No | **Yes** |
| Message boundary | Stream (no boundary) | **Datagram (boundary preserved)** |

---

## 2. UDP là gì? — Giải thích cho người không biết IT

### Phép so sánh đời thường — Phát tờ rơi

Bạn đứng trước siêu thị **phát tờ rơi**:
- Đưa cho mỗi người đi qua — **không cần biết tên** họ
- Người ta **có nhận hay không** — kệ! (best effort)
- Phát cho **bao nhiêu người cũng được** — không giới hạn (no flow control)
- Nếu tờ rơi bay đi — **không phát lại** (no retransmission)
- **Cực nhanh** — không cần dừng lại nói chuyện (no handshake)
- Mỗi tờ rơi là **hoàn chỉnh** — đọc 1 tờ là hiểu (datagram boundary)

So sánh: TCP = Gọi từng người lại, giới thiệu bản thân, xác nhận họ hiểu, hỏi lại nếu chưa rõ. Đầy đủ nhưng MẤT THỜI GIAN!

### Định nghĩa kỹ thuật

**UDP (User Datagram Protocol)** là transport layer protocol cung cấp phương thức truyền **datagram** (message đơn lẻ) giữa các ứng dụng qua mạng IP với **overhead tối thiểu**.

**Đặc điểm quan trọng:**
- **Protocol number: 17** trong IP header
- **RFC 768** (1980) — một trong những RFC ngắn nhất (3 trang!)
- **Datagram-oriented** — mỗi send() = 1 datagram = 1 recv()
- **Preserves message boundaries** (khác TCP — stream-oriented)

### Message Boundary — Điểm khác biệt quan trọng

```
TCP (Stream):
  send("Hello")    → recv() might get "Hel" then "lo"
  send("World")    → recv() might get "lloWorld"
  NO message boundary! App phải tự parse!

UDP (Datagram):
  sendto("Hello")  → recvfrom() gets "Hello" (exact!)
  sendto("World")  → recvfrom() gets "World" (exact!)
  Message boundary PRESERVED! Mỗi datagram = 1 message hoàn chỉnh!
```

---

## 3. Cấu trúc UDP Header — Đơn giản đến bất ngờ

### Phép so sánh — Phong bì đơn giản nhất

TCP header = Biểu mẫu phức tạp (20-60 bytes, nhiều trường)
UDP header = **Phong bì trần** — chỉ có 4 thông tin:
1. Người gửi (Source Port)
2. Người nhận (Destination Port)  
3. Kích thước (Length)
4. Kiểm tra lỗi (Checksum)

That's it! Chỉ **8 bytes**!

### UDP Header Format (RFC 768)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Source Port (16)          │      Destination Port (16)    │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│          Length (16)               │      Checksum (16)            │
├─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┼─┤
│                         Data (variable)                            │
└─────────────────────────────────────────────────────────────────────┘

Total header: 8 bytes only!
```

### Chi tiết từng trường

#### Source Port — 16 bits (optional)

```
Port người gửi:
- Nếu cần reply → phải set (server reply đến port này)
- Nếu không cần reply → có thể = 0
- Range: 0-65535

Ví dụ:
  DNS query:  src=53421 (random ephemeral)
  DNS response sẽ gửi đến port 53421
```

#### Destination Port — 16 bits (required)

```
Port người nhận — XÁC ĐỊNH application đích:
- DNS: 53
- DHCP Server: 67, Client: 68
- NTP: 123
- SNMP: 161, Trap: 162
- Syslog: 514
- RTP: dynamic (even ports, 16384-32767)
- QUIC: 443 (same as HTTPS)
```

#### Length — 16 bits

```
Tổng chiều dài: Header (8 bytes) + Data
- Minimum: 8 (header only, no data)
- Maximum: 65,535 bytes (IP limit) — thực tế ít dùng > MTU

Lưu ý: Length bao gồm CẢ header, không chỉ data!
  Length = 8 + len(data)
  
Practical maximum:
  Ethernet MTU 1500 - IP header 20 = 1480 bytes
  UDP: 1480 - 8 (UDP header) = 1472 bytes data per datagram
  (without fragmentation)
```

#### Checksum — 16 bits

```
One's complement checksum covering:
- Pseudo-header (src IP, dst IP, protocol=17, UDP length)
- UDP header
- UDP data (padded to even octets)

IPv4: Checksum OPTIONAL (0 = disabled) — tuy nhiên hầu hết enable
IPv6: Checksum MANDATORY (RFC 8200)

Pseudo-header (IPv4):
┌──────────────────────────────────┐
│ Source IP Address (32 bits)      │
│ Destination IP Address (32 bits) │
│ Zero (8) │ Protocol=17 │ UDP Len│
└──────────────────────────────────┘
```

### So sánh kích thước Headers

```
Ethernet Frame carrying UDP datagram:
┌────────┬────────┬──────┬────────┬───────────────┐
│  Ether │   IP   │ UDP  │  Data  │   Ether FCS   │
│  14B   │  20B   │  8B  │  var   │     4B        │
└────────┴────────┴──────┴────────┴───────────────┘
  Overhead: 14+20+8+4 = 46 bytes minimum

Ethernet Frame carrying TCP segment:
┌────────┬────────┬──────┬────────┬───────────────┐
│  Ether │   IP   │ TCP  │  Data  │   Ether FCS   │
│  14B   │  20B   │ 20B+ │  var   │     4B        │
└────────┴────────┴──────┴────────┴───────────────┘
  Overhead: 14+20+20+4 = 58 bytes minimum (TCP no options)
  Thực tế TCP + timestamps: 14+20+32+4 = 70 bytes

UDP tiết kiệm: 12-24 bytes/packet so với TCP!
(Quan trọng khi gửi hàng triệu small packets/giây)
```

---

## 4. UDP hoạt động — Step by Step

### Phép so sánh — Walkie-Talkie (Bộ đàm)

UDP giống dùng **bộ đàm (walkie-talkie)**:
1. Bấm nút nói → **nói luôn** (không cần gọi, không cần chờ)
2. Người nghe có nghe được không? → **Không biết!**
3. Nói lại lần 2 cùng lúc? → **Có thể trùng!**
4. Nhiều người nói cùng lúc? → **Được! (multicast/broadcast)**
5. Xong tin nhắn → **Thả nút** (no state maintained)

### UDP Communication Flow

```
Client                                      Server
  │                                           │
  │  sendto(data, server_addr)                │
  │──────── UDP Datagram ────────────────────→│ recvfrom() → data
  │                                           │
  │  No handshake!                            │
  │  No ACK!                                  │
  │  No connection state!                     │
  │                                           │
  │  (Server can reply if it knows src port)  │
  │←──────── UDP Datagram ───────────────────│ sendto(reply, client_addr)
  │                                           │
  │  recvfrom() → reply                       │
  │                                           │
  │  Done! No teardown!                       │
```

### Socket Programming — UDP vs TCP

```python
# === TCP Client (connection-oriented) ===
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)  # STREAM = TCP
sock.connect(('server.com', 80))         # 3-way handshake!
sock.send(b'GET / HTTP/1.1\r\n...')      # Send data
response = sock.recv(4096)                # Receive response
sock.close()                              # 4-way teardown

# === UDP Client (connectionless) ===
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)   # DGRAM = UDP
sock.sendto(b'Hello', ('server.com', 12345))  # Send immediately! No connect!
data, addr = sock.recvfrom(4096)               # Receive from anyone
sock.close()                                    # Just close (no teardown)
```

```python
# === UDP Server ===
import socket
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('0.0.0.0', 12345))

while True:
    data, client_addr = sock.recvfrom(4096)     # Wait for any client
    print(f"Received '{data}' from {client_addr}")
    sock.sendto(b'ACK: ' + data, client_addr)   # Reply to sender
```

### UDP "Connections" — connected UDP socket

```python
# Dù UDP là connectionless, bạn CÓ THỂ "connect" UDP socket:
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.connect(('server.com', 12345))  # Set default destination

# Bây giờ dùng send() thay vì sendto():
sock.send(b'Hello')  # Gửi đến default destination

# Advantages:
# 1. Kernel filter — chỉ nhận packets từ connected peer
# 2. Nhận ICMP errors (port unreachable)
# 3. Hiệu quả hơn (kernel cache routing decision)
# 4. Dùng được send()/recv() (không cần sendto/recvfrom)
```

---

## 5. UDP-Based Protocols — DNS, DHCP, NTP

### DNS over UDP (Port 53)

```
Tại sao DNS dùng UDP:
  - Query nhỏ (< 512 bytes traditional, < 4096 EDNS)
  - 1 request → 1 response (stateless)
  - UDP: 1 RTT total (query + response)
  - TCP: 2 RTT (handshake + query + response)
  - DNS servers xử lý HÀNG TRIỆU queries/giây
  - TCP state per connection = memory exhaustion!

Khi nào DNS dùng TCP:
  - Response > 512 bytes (EDNS0 tăng lên 4096)
  - Zone transfers (AXFR/IXFR)
  - Truncated response (TC flag set) → retry qua TCP
  - DNS over TLS (DoT) — port 853
  
DNS Flow (UDP):
  Client → [Query: www.google.com A?] → DNS Server
  Client ← [Response: 142.250.80.46] ← DNS Server
  Total time: 1 RTT (20-100ms typically)
  
  Nếu dùng TCP:
  Client → SYN → Server
  Client ← SYN+ACK ← Server     } 1 RTT wasted!
  Client → ACK + Query → Server
  Client ← Response ← Server
  Client → FIN... (teardown)
  Total: 2+ RTT = 2-3× slower!
```

### DHCP (Ports 67/68)

```
Tại sao DHCP dùng UDP:
  - Client CHƯA CÓ IP address! → Không thể TCP (cần IP)
  - Dùng broadcast (255.255.255.255) → UDP only
  - Simple request-response
  - Client có thể retry nếu không nhận response

DHCP DORA Process (all UDP):
  Client (0.0.0.0:68) → Broadcast:
    DISCOVER: "Ai là DHCP server? Cho tôi IP!"
    
  Server (10.0.0.1:67) → Broadcast:
    OFFER: "Tôi có IP 10.0.0.100, muốn không?"
    
  Client → Broadcast:
    REQUEST: "Tôi muốn IP 10.0.0.100 (từ server 10.0.0.1)"
    
  Server → Broadcast/Unicast:
    ACK: "OK! 10.0.0.100 là của bạn, lease 24h"
```

### NTP — Network Time Protocol (Port 123)

```
Tại sao NTP dùng UDP:
  - Packets nhỏ (48 bytes)
  - Cần đo RTT chính xác (TCP overhead làm sai RTT)
  - Stateless — poll periodically
  - Loss acceptable (try again next poll)

NTP accuracy:
  - LAN: < 1ms
  - Internet: 1-50ms
  - TCP overhead sẽ add jitter → less accurate!
```

### RTP — Real-time Transport Protocol

```
Tại sao RTP/VoIP dùng UDP:
  - Voice/Video = real-time → delay > 150ms = BAD!
  - TCP retransmission: packet delay 200ms-2s → audio glitch!
  - Old frame = useless (không cần retransmit frame 2 giây trước)
  - Jitter buffer handles small loss → OK!
  - RTP provides: sequence number, timestamp (trên UDP)

Stack:
  Application (Voice/Video codec)
  ↓
  RTP (sequence, timestamp, SSRC) — port even (e.g., 16384)
  RTCP (statistics, QoS feedback) — port odd (e.g., 16385)
  ↓
  UDP
  ↓
  IP
  
RTP header (12 bytes):
  - Sequence number (detect loss, reorder)
  - Timestamp (synchronize playback)
  - SSRC (identify source)
  - Payload type (codec: G.711, Opus, H.264...)

Jitter buffer:
  Packets arrive: [1] [3] [2] [5] [4] [6]
  Buffer reorders: [1] [2] [3] [4] [5] [6]
  Plays smoothly with slight delay (20-100ms buffer)
  If [7] never arrives → skip (concealment) → continue
```

### TFTP — Trivial File Transfer Protocol (Port 69)

```
TFTP dùng UDP vì:
  - Extreme simplicity (fit in boot ROM)
  - Used for PXE boot, firmware update
  - Adds own reliability: stop-and-wait ACK per block
  
  Client → RRQ (Read Request) → Server
  Server → DATA block 1 → Client
  Client → ACK 1 → Server
  Server → DATA block 2 → Client
  Client → ACK 2 → Server
  ... (512 bytes per block until last block < 512)
```

---

## 6. Gaming và Streaming — UDP in Action

### Phép so sánh — Trận bóng đá trực tiếp

Xem bóng đá trực tiếp trên TV:
- Nếu mất vài frame → **bỏ qua** (viewer không nhận ra)
- Nếu ĐĂNG lại frame cũ → **lag, giật** (mọi người hét "sao chậm thế!")
- **Real-time** quan trọng hơn **completeness**!

### Online Gaming — UDP Architecture

```
Game Server Architecture:
  
  Player Input (60 times/sec):
    W key pressed → UDP packet → Server
    Mouse moved → UDP packet → Server
    
  Server Tick (20-128 times/sec):
    Calculate game state
    Send state update → UDP → All players
    
  Why UDP:
    - 60 packets/sec × 100 players = 6000 packets/sec
    - Each packet: ~50-200 bytes (small!)
    - TCP overhead (20+ bytes header × 6000) = significant
    - TCP retransmit: old position = USELESS (player already moved!)
    - UDP: if packet lost → use prediction → correct on next update

Game Networking Patterns:

1. Client-Side Prediction:
   Client: "I pressed W" → show movement immediately (predict)
   Server: confirms or CORRECTS position
   If packet lost: slight rubber-banding (acceptable)

2. Snapshot Interpolation:
   Server sends game states: t=0, t=50ms, t=100ms
   Client interpolates BETWEEN states for smooth rendering
   Loss of 1 snapshot → interpolate over larger gap

3. Delta Compression:
   Instead of full state each tick:
   Send only CHANGES from last acknowledged state
   Reduces bandwidth significantly
```

### Video Streaming

```
Live Streaming (Twitch, YouTube Live):
  
  Encoder → [Video frames] → UDP/RTP → CDN → Viewers
  
  Adaptive Bitrate (ABR):
    - Monitor network quality (loss rate, bandwidth)
    - Switch quality: 1080p → 720p → 480p dynamically
    - Better to show lower quality than to PAUSE!
  
  Protocols:
    - RTMP (TCP-based, legacy for ingest)
    - SRT (UDP-based, reliable streaming)
    - WebRTC (UDP/DTLS/SRTP, ultra-low latency)
    - RIST (UDP-based, broadcast-quality)
    - HLS/DASH (HTTP/TCP-based, high latency but reliable)

SRT (Secure Reliable Transport):
  - UDP-based protocol for live video
  - ARQ (Automatic Repeat reQuest) — selective retransmit
  - Encryption (AES-128/256)
  - Bandwidth estimation
  - Latency: 120ms-8000ms (configurable)
  - Used by: OBS Studio, Haivision, vMix
```

### VoIP — Voice over IP

```
VoIP Call Flow:

Signaling (SIP — TCP/UDP port 5060):
  - Call setup, teardown, hold, transfer
  - Uses TCP or UDP depending on implementation

Media (RTP — UDP):
  - Actual voice data
  - Codec: G.711 (64kbps), Opus (6-510kbps)
  - Packet rate: 50 packets/sec (20ms per packet)
  - Packet size: ~200 bytes (G.711, 20ms)
  
QoS Requirements:
  - Latency: < 150ms one-way (< 300ms round-trip)
  - Jitter: < 30ms
  - Packet loss: < 1% (acceptable with PLC)
  
  TCP retransmit adds 100-500ms delay → UNACCEPTABLE for voice!
  UDP: lost packet → silence/concealment → continue (acceptable!)
```

---

## 7. UDP Reliability — Khi app cần reliability trên UDP

### Phép so sánh — Bưu thiếp + App tracking

UDP không reliable, nhưng APP có thể tự thêm reliability:
- Bưu thiếp thường = UDP (fire and forget)
- Bưu thiếp + app tracking + xác nhận = **reliable UDP** (app tự quản lý)

### Tại sao build reliability trên UDP thay vì dùng TCP?

```
Reasons:
1. CONTROL: App biết data nào quan trọng, data nào có thể bỏ
2. LATENCY: TCP retransmit ALL, app chỉ retransmit critical data
3. FLEXIBILITY: Custom congestion control (not forced CUBIC/BBR)
4. CONNECTION MIGRATION: QUIC — change IP without reconnect
5. MULTIPLEXING: Multiple streams without Head-of-Line blocking
6. ENCRYPTION: QUIC encrypts headers (TCP headers visible)
```

### Pattern 1: Application-Level ACK

```
Simple reliable UDP:

Sender:
  1. Send packet with sequence number
  2. Start timer
  3. If ACK received → success
  4. If timeout → retransmit (limited retries)

Receiver:
  1. Receive packet
  2. Check sequence number (detect duplicates)
  3. Send ACK with sequence number
  4. Deliver to application

// Similar to TFTP stop-and-wait, but can be windowed
```

### Pattern 2: Selective Reliability (Games)

```
Game data categories:
  - CRITICAL: Player spawn, death, score → MUST deliver (retransmit)
  - IMPORTANT: Hit registration → deliver if possible
  - UNRELIABLE: Position updates → latest is enough (no retransmit)

Implementation:
  Packet header:
    [Channel: 0=unreliable, 1=reliable, 2=ordered]
    [Sequence number]
    [ACK bitfield for reliable channel]
    [Data]

  Reliable channel: ACK + retransmit (like TCP but simpler)
  Unreliable channel: fire-and-forget (pure UDP)
  Ordered channel: buffer + reorder before delivery
```

### Pattern 3: FEC (Forward Error Correction)

```
Instead of retransmitting:
  Add REDUNDANT data so receiver can RECONSTRUCT lost packets!

Example (XOR FEC):
  Send: Packet 1, Packet 2, Packet 3, FEC(1⊕2⊕3)
  If Packet 2 lost:
    Receiver: FEC ⊕ Packet 1 ⊕ Packet 3 = Packet 2 (recovered!)
  
  No retransmission needed! Zero additional RTT!
  Cost: Extra bandwidth (FEC packets)

Used by: QUIC, WebRTC, video streaming, space communication
```

### QUIC — Ultimate "reliable UDP"

```
QUIC = UDP + TLS 1.3 + HTTP/2 features:
  - Runs over UDP port 443
  - Built-in encryption (mandatory)
  - Multiple streams (no head-of-line blocking!)
  - Connection migration (change network without reconnect)
  - 0-RTT connection establishment
  - Custom congestion control
  
QUIC will be covered in detail in next post!
```

---

## 8. UDP Security — Attacks và Mitigation

### UDP Amplification Attack

```
Attacker exploits UDP services that respond with MORE data than request:

1. Attacker sends small query with SPOOFED source IP (victim's IP)
2. UDP server responds to victim with LARGE response
3. Amplification factor: response_size / request_size

Protocol          Request    Response    Amplification
─────────────────────────────────────────────────────
DNS               ~60 bytes  ~3000 bytes    50×
NTP (monlist)     ~8 bytes   ~500 bytes     556×
Memcached         ~15 bytes  ~750KB         51,000×!!
SSDP              ~40 bytes  ~30KB          30×
SNMP              ~60 bytes  ~6000 bytes    6×
CharGEN           ~1 byte    ~1000 bytes    358×

Attack flow:
  Attacker → [Request, src=VICTIM_IP] → Amplifier (DNS server)
  Amplifier → [Large Response] → Victim!
  
  1 Mbps attack traffic × 50 amplification = 50 Mbps hitting victim!
```

### Mitigation

```bash
# 1. BCP38/BCP84 — Source address validation (ISP should implement)
#    Block packets with spoofed source IPs at network edge

# 2. Rate limiting UDP on server:
iptables -A INPUT -p udp --dport 53 -m limit --limit 100/s -j ACCEPT
iptables -A INPUT -p udp --dport 53 -j DROP

# 3. Disable unnecessary UDP services:
systemctl stop memcached    # If not needed
systemctl stop ntpd         # Use chrony instead (no monlist)

# 4. Response Rate Limiting (DNS):
# BIND: rate-limit { responses-per-second 10; };

# 5. uRPF (unicast Reverse Path Forwarding):
# Router drops packets with source IPs not reachable via that interface

# 6. Cloud DDoS protection:
# AWS Shield, Cloudflare, Akamai — absorb UDP floods
```

### UDP Flood Attack

```
Simple attack: Send massive UDP packets to random ports on victim

Victim behavior:
  1. Receive UDP on closed port
  2. Generate ICMP "Port Unreachable" for each
  3. CPU overwhelmed generating ICMP!
  4. Bandwidth saturated

Defense:
  - Rate limit ICMP generation
  - Firewall: drop UDP to unused ports
  - Cloud scrubbing (absorb at edge)
  - BGP blackhole for extreme attacks
```

---

## 9. UDP Performance Tuning

### Buffer Sizes

```bash
# UDP KHÔNG có flow control!
# Nếu app không đọc nhanh → kernel buffer đầy → packets DROPPED silently!

# Check current buffer sizes:
cat /proc/sys/net/core/rmem_default    # Default receive buffer (212992)
cat /proc/sys/net/core/rmem_max        # Max receive buffer
cat /proc/sys/net/core/wmem_default    # Default send buffer
cat /proc/sys/net/core/wmem_max        # Max send buffer

# Increase for high-throughput UDP:
sysctl -w net.core.rmem_max=26214400       # 25MB
sysctl -w net.core.wmem_max=26214400
sysctl -w net.core.rmem_default=1048576    # 1MB default

# Per-socket (application):
sock.setsockopt(socket.SOL_SOCKET, socket.SO_RCVBUF, 8388608)  # 8MB
```

### Monitoring UDP drops

```bash
# Check if kernel is dropping UDP packets:
cat /proc/net/udp
# ... rx_queue shows pending bytes, drops column shows drops

# Better:
nstat -az | grep Udp
# UdpInDatagrams   100000   # Total received
# UdpNoPorts       500      # Received on closed port (ICMP generated)
# UdpInErrors      200      # ← DROPS! Buffer overflow!
# UdpOutDatagrams  95000    # Total sent

# Watch drops in real-time:
watch -n1 "nstat -az | grep UdpInErrors"

# Per-socket drops:
ss -u -a -e
# Shows drop count per socket

# netstat style:
netstat -su
# Udp:
#     500000 packets received
#     200 packet receive errors    ← DROPS!
#     480000 packets sent
#     200 receive buffer errors    ← BUFFER FULL!
```

### High-Performance UDP — Tips

```
1. Use recvmmsg/sendmmsg (batch syscalls):
   - recvmmsg(): receive multiple datagrams in 1 syscall
   - sendmmsg(): send multiple datagrams in 1 syscall
   - Reduces syscall overhead significantly!

2. SO_REUSEPORT (multi-thread):
   - Multiple threads bind same port
   - Kernel load-balances across threads
   - Enables multi-core UDP processing

3. GRO/GSO (Generic Receive/Segmentation Offload):
   - Kernel batches small UDP packets into larger ones
   - Reduces per-packet processing overhead
   - Enabled by default on modern Linux

4. AF_XDP / DPDK (bypass kernel):
   - Direct NIC → userspace (no kernel UDP stack)
   - For extreme performance (millions pps)
   - Used by: QUIC servers, game servers, DNS servers

5. UDP GSO (Generic Segmentation Offload):
   # Send large buffer, kernel segments into MTU-sized UDP packets
   setsockopt(fd, SOL_UDP, UDP_SEGMENT, &mss, sizeof(mss));
```

---

## 10. UDP Use Cases Decision Matrix

### Khi nào chọn UDP vs TCP?

```
Decision flowchart:

  Cần reliable delivery? ──YES──→ TCP (or QUIC)
       │
       NO
       ↓
  Real-time? (latency critical) ──YES──→ UDP
       │
       NO
       ↓
  Small request-response? ──YES──→ UDP (DNS pattern)
       │
       NO
       ↓
  Multicast/Broadcast needed? ──YES──→ UDP (only option)
       │
       NO
       ↓
  High packet rate, small packets? ──YES──→ UDP
       │
       NO
       ↓
  Default: TCP (simpler, more compatible)
```

### UDP Protocol Summary Table

| Protocol | Port | Why UDP | Reliability |
|---|---|---|---|
| DNS | 53 | Fast query-response | App retransmit |
| DHCP | 67/68 | No IP yet (broadcast) | App retransmit |
| NTP | 123 | Accurate RTT, stateless | Next poll |
| SNMP | 161/162 | Polling, traps | App retransmit |
| Syslog | 514 | Log shipping, loss OK | None |
| TFTP | 69 | Boot ROM simplicity | Stop-and-wait |
| RTP | Dynamic | Real-time media | FEC, concealment |
| SIP | 5060 | Signaling, retransmit | App timer |
| QUIC | 443 | Custom reliability | Built-in |
| WireGuard | 51820 | VPN, NAT traversal | Crypto handshake |
| mDNS | 5353 | Local discovery | Multicast |
| RADIUS | 1812/1813 | AAA protocol | App retransmit |
| STUN/TURN | 3478 | NAT traversal | App retransmit |
| Gaming | Various | Real-time input | Selective |

### Tổng kết

```
UDP Key Points:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  UDP = Simple, Fast, Minimal overhead                   │
│                                                         │
│  Header: 8 bytes (Src Port, Dst Port, Length, Checksum)│
│  Protocol: IP protocol number 17                        │
│  RFC: 768 (1980, 3 pages!)                             │
│                                                         │
│  Properties:                                            │
│  ✓ Connectionless (no handshake)                       │
│  ✓ Datagram boundary preserved                         │
│  ✓ Supports broadcast/multicast                        │
│  ✓ Low latency (no retransmission delay)              │
│  ✓ Low overhead (8 bytes vs 20-60 TCP)                │
│  ✗ No reliability (can lose packets)                   │
│  ✗ No ordering (can arrive out-of-order)              │
│  ✗ No flow control (can overwhelm receiver)           │
│  ✗ No congestion control (can congest network)         │
│                                                         │
│  Use When:                                              │
│  • Real-time (voice, video, games)                     │
│  • Small queries (DNS, NTP, SNMP)                      │
│  • Broadcast/Multicast needed                          │
│  • App handles own reliability (QUIC)                  │
│  • Maximum performance needed                          │
│                                                         │
│  Don't Use When:                                        │
│  • Data MUST arrive (file transfer, email)             │
│  • Order matters and app can't handle                  │
│  • Simple implementation wanted                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

*Tài liệu tham khảo:*
- RFC 768 — User Datagram Protocol
- RFC 8200 — Internet Protocol, Version 6 (IPv6) — UDP checksum mandatory
- RFC 3550 — RTP: A Transport Protocol for Real-Time Applications
- RFC 8085 — UDP Usage Guidelines
- RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport
- Linux kernel UDP implementation — net/ipv4/udp.c
- Stevens, W.R. — UNIX Network Programming, Volume 1
- High Performance Browser Networking (hpbn.co) — Building Blocks of UDP
