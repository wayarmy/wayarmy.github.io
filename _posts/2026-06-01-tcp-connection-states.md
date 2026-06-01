---
layout: post
title: "TCP Connection States - Tất cả trạng thái từ LISTEN đến CLOSED, TIME-WAIT và Debugging"
date: 2026-06-01
categories: [networking]
tags: [tcp, connection-states, time-wait, debugging, netstat, ss, troubleshooting]
---

## 1. Giới thiệu — Vòng đời của cuộc gọi điện thoại

Hãy tưởng tượng mỗi kết nối TCP giống một **cuộc gọi điện thoại**:

1. **LISTEN** — Bạn bật điện thoại, **chờ ai gọi đến**
2. **SYN_SENT** — Bạn **bấm gọi**, chờ đầu bên kia nhấc máy
3. **SYN_RECEIVED** — Đầu bên kia thấy cuộc gọi đến, **đang rung chuông**
4. **ESTABLISHED** — Cả 2 bên **nhấc máy**, đang nói chuyện!
5. **FIN_WAIT_1** — Bạn nói "Bye" (**gác máy trước**)
6. **FIN_WAIT_2** — Bạn chờ đầu kia cũng nói "Bye"
7. **CLOSE_WAIT** — Đầu kia nghe "Bye" nhưng **chưa gác máy** (còn nói nốt)
8. **LAST_ACK** — Đầu kia nói "Bye" lại, chờ xác nhận cuối
9. **TIME_WAIT** — Cuộc gọi kết thúc, nhưng **giữ lại số** 2 phút (phòng gọi nhầm)
10. **CLOSED** — Xong! Điện thoại **rảnh** hoàn toàn

### Tại sao cần biết TCP States?

| Situation | State liên quan | Ý nghĩa |
|---|---|---|
| Server bị "connection refused" | LISTEN thiếu | Service chưa start/port chưa listen |
| Nhiều connections "treo" | CLOSE_WAIT tích lũy | App không close socket (bug!) |
| Port bị "address already in use" | TIME_WAIT nhiều | Server restart quá nhanh |
| Connection timeout | SYN_SENT lâu | Network block/firewall |
| Memory leak on server | ESTABLISHED tăng mãi | Connection pool leak |

### TCP State Machine — Bức tranh tổng thể

```
                              ┌───────────┐
                  Passive open│           │Active open
                  ───────────→│  CLOSED   │←───────────
                              │           │
                              └─────┬─────┘
                                    │
                   ┌────────────────┼────────────────┐
                   │ (server)       │                 │ (client)
                   ▼                │                 ▼
            ┌──────────┐           │          ┌──────────┐
            │  LISTEN  │           │          │ SYN_SENT │
            └────┬─────┘           │          └────┬─────┘
    Recv SYN     │                 │               │ Recv SYN+ACK
    Send SYN+ACK │                 │               │ Send ACK
                 ▼                 │               ▼
          ┌────────────┐           │     ┌─────────────┐
          │SYN_RECEIVED│───────────┼────→│ ESTABLISHED │
          └────────────┘  Recv ACK │     └──────┬──────┘
                                   │            │
                                   │     Close/ │
                                   │     Send FIN
                                   │            │
              ┌───────────────────┐│            ▼
              │   CLOSE_WAIT      ││     ┌────────────┐
              │   (Recv FIN,      ││     │ FIN_WAIT_1 │
              │    Send ACK)      ││     └─────┬──────┘
              └────────┬──────────┘│           │
                       │           │    Recv ACK│      Recv FIN+ACK
              Close/   │           │           ▼       Send ACK
              Send FIN │           │    ┌────────────┐    │
                       ▼           │    │ FIN_WAIT_2 │    │
              ┌────────────┐       │    └─────┬──────┘    │
              │  LAST_ACK  │       │          │           │
              └─────┬──────┘       │   Recv FIN│           │
                    │              │   Send ACK│           │
              Recv ACK             │          ▼           ▼
                    │              │    ┌────────────┐
                    ▼              │    │ TIME_WAIT  │
              ┌──────────┐         │    │ (2×MSL)    │
              │  CLOSED  │←────────┘    └─────┬──────┘
              └──────────┘                    │
                    ↑                   Timeout│
                    └─────────────────────────┘
```

---

## 2. Chi tiết 11 trạng thái TCP

### CLOSED — Không tồn tại

```
Đây KHÔNG phải trạng thái thực sự — nó là "không có connection nào"

Ý nghĩa: Socket chưa tạo hoặc đã bị destroy hoàn toàn
Khi nào: Trước khi socket() call hoặc sau khi close() hoàn tất

Bạn KHÔNG thấy CLOSED trong netstat/ss (vì không có socket nào cả)
```

### LISTEN — Server chờ kết nối

```
Server đã bind() + listen() — sẵn sàng nhận connection mới

Ai ở state này: SERVER side only
Khi nào: Sau listen() system call
Transition: Khi nhận SYN → SYN_RECEIVED

Ví dụ:
  Proto  Local Address     State
  tcp    0.0.0.0:80        LISTEN    ← Nginx đang chờ
  tcp    0.0.0.0:443       LISTEN    ← HTTPS ready
  tcp    127.0.0.1:5432    LISTEN    ← PostgreSQL (localhost only)
  tcp    0.0.0.0:22        LISTEN    ← SSH server
```

**Trouble:** Port KHÔNG ở LISTEN = service chưa start!
```bash
# Check xem service có listen không:
ss -tlnp | grep :80
# Nếu không có output → nginx/apache chưa start!
```

### SYN_SENT — Client đã gửi SYN, chờ reply

```
Client đã gửi SYN packet, đang chờ SYN-ACK từ server

Ai ở state này: CLIENT side
Khi nào: Sau connect() call, SYN packet đã gửi
Duration: Thường rất ngắn (< 1 RTT)
Transition: 
  Recv SYN+ACK → ESTABLISHED (normal)
  Timeout → CLOSED (failed)

Nếu SYN_SENT kéo dài:
  - Firewall block SYN (drop silently)
  - Server down
  - Network unreachable
  - Wrong port/IP
  
Linux retries: net.ipv4.tcp_syn_retries = 6
  SYN → 1s → SYN → 2s → SYN → 4s → SYN → 8s → SYN → 16s → SYN → 32s
  Total: 63 seconds rồi fail (ETIMEDOUT)
```

### SYN_RECEIVED (SYN_RECV) — Server nhận SYN, đã reply SYN+ACK

```
Server nhận SYN, đã gửi SYN+ACK, chờ ACK cuối từ client

Ai ở state này: SERVER side
Khi nào: Sau khi reply SYN+ACK
Duration: Thường rất ngắn (< 1 RTT)
Transition:
  Recv ACK → ESTABLISHED (normal)
  Timeout → CLOSED (SYN flood attack!)

Nếu SYN_RECV nhiều:
  - SYN flood attack! (attacker gửi SYN liên tục, không bao giờ ACK)
  - SYN queue (backlog) đầy → server từ chối connection mới!
  
Mitigation:
  - SYN cookies (net.ipv4.tcp_syncookies = 1)
  - Increase backlog (net.ipv4.tcp_max_syn_backlog)
  - Rate limiting (iptables)
```

### ESTABLISHED — Connection hoạt động

```
3-way handshake hoàn tất — DATA TRANSFER đang diễn ra!

Ai ở state này: CẢ HAI bên (client VÀ server)
Khi nào: Sau ACK cuối của handshake
Duration: Toàn bộ thời gian communicate
Transition:
  Send FIN → FIN_WAIT_1 (active close)
  Recv FIN → CLOSE_WAIT (passive close)

Monitoring ESTABLISHED:
  - Quá ít → service bị lỗi hoặc ít traffic
  - Quá nhiều → possible connection leak, hoặc high load
  - Tăng đều → normal growth
  - Đột ngột → attack hoặc thundering herd
```

```bash
# Count ESTABLISHED connections:
ss -tn state established | wc -l

# ESTABLISHED per destination:
ss -tn state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn

# Monitor over time:
watch -n5 "ss -s | grep estab"
```

### FIN_WAIT_1 — Active closer đã gửi FIN

```
Bên muốn đóng connection đã gửi FIN, chờ ACK

Ai ở state này: Bên ACTIVE CLOSE (bên gọi close() trước)
Duration: Rất ngắn (< 1 RTT cho ACK)
Transition:
  Recv ACK → FIN_WAIT_2
  Recv FIN+ACK → TIME_WAIT (simultaneous close)
  Recv FIN → CLOSING (rare)

Nếu FIN_WAIT_1 lâu:
  - Network congestion (ACK delayed)
  - Peer crashed (không gửi ACK)
  - Firewall dropping ACK packets
```

### FIN_WAIT_2 — Đã nhận ACK cho FIN, chờ FIN từ peer

```
Bên active closer đã nhận ACK (peer biết mình muốn close)
Đang chờ peer cũng gửi FIN (peer đóng phía họ)

Ai: Bên ACTIVE CLOSE
Duration: Tùy thuộc peer khi nào close
Transition:
  Recv FIN → TIME_WAIT (sau khi gửi ACK)

Nếu FIN_WAIT_2 lâu:
  - Peer app chưa close() socket (bug!)
  - Peer vẫn đang gửi data (half-close — legitimate)
  
Linux timeout: net.ipv4.tcp_fin_timeout = 60 (default)
  Sau 60s ở FIN_WAIT_2 → auto-close (only khi socket orphan)
```

### CLOSE_WAIT — Nhận FIN từ peer, app chưa close

```
★★★ TRẠNG THÁI QUAN TRỌNG NHẤT ĐỂ DEBUG ★★★

Peer muốn đóng (gửi FIN), ta đã ACK nhưng APP CHƯA CLOSE SOCKET!

Ai: Bên PASSIVE CLOSE (bên nhận FIN)
Duration: Cho đến khi APP gọi close()
Transition:
  App calls close() → Send FIN → LAST_ACK

⚠️ CLOSE_WAIT NHIỀU = BUG TRONG APPLICATION! ⚠️
  - App không close socket sau khi peer disconnect
  - Connection pool không return connections
  - Exception handling không close() trong finally block
  - File descriptor leak → eventually "too many open files"

CLOSE_WAIT KHÔNG tự hết! Chỉ app mới có thể close!
```

```bash
# Tìm process có nhiều CLOSE_WAIT:
ss -tnp state close-wait | awk '{print $NF}' | sort | uniq -c | sort -rn

# Output:
# 500 users:(("java",pid=1234,fd=45))  ← Java app BUG!
# 23  users:(("python",pid=5678,fd=12))
```

### LAST_ACK — Đã gửi FIN, chờ ACK cuối

```
Bên passive closer đã gọi close() → gửi FIN → chờ ACK cuối

Ai: Bên PASSIVE CLOSE
Duration: Rất ngắn (< 1 RTT)
Transition:
  Recv ACK → CLOSED

Nếu LAST_ACK lâu:
  - Peer crashed sau khi nhận FIN (không gửi ACK)
  - Network issue
  - Thường timeout sau retransmission limit
```

### TIME_WAIT — Chờ 2×MSL trước khi close hoàn toàn

```
★★★ TRẠNG THÁI GÂY CONFUSION NHẤT ★★★

Connection đã close logic nhưng socket GIỮ LẠI 2×MSL (60-120 seconds)

Ai: Bên ACTIVE CLOSE (bên gửi FIN trước)
Duration: 2 × MSL (Maximum Segment Lifetime)
  Linux: 60 seconds (net.ipv4.tcp_fin_timeout KHÔNG affect TIME_WAIT!)
  Linux actual TIME_WAIT: hardcoded 60s (TCP_TIMEWAIT_LEN)
Transition:
  Timeout → CLOSED
```

### CLOSING — Simultaneous close (hiếm gặp)

```
Cả 2 bên gửi FIN ĐỒNG THỜI (trước khi nhận FIN từ peer)

Extremely rare trong practice:
  A gửi FIN → (in transit)
  B gửi FIN → (in transit)
  A nhận FIN từ B (chưa nhận ACK cho FIN của mình) → CLOSING
  B nhận FIN từ A (chưa nhận ACK cho FIN của mình) → CLOSING
  Cả 2 gửi ACK → TIME_WAIT → CLOSED
```

---

## 3. TIME_WAIT — Hiểu sâu và xử lý

### Phép so sánh — Thời gian "giữ số điện thoại"

Sau khi gác máy, nhà mạng **giữ số** thêm 2 phút:
- Phòng trường hợp tin nhắn cuối **đang trên đường** (delayed segments)
- Phòng trường hợp đầu kia **chưa nhận** ACK cuối (retransmit FIN)
- Sau 2 phút → chắc chắn mọi thứ đã "sạch" → giải phóng số

### Tại sao TIME_WAIT cần thiết?

**Lý do 1: Đảm bảo ACK cuối đến peer**
```
Active closer gửi ACK cuối cho FIN của peer:
  - Nếu ACK bị mất → peer retransmit FIN
  - Active closer phải ở TIME_WAIT để nhận retransmit → gửi lại ACK
  - Nếu đã CLOSED → nhận FIN → gửi RST (peer confused!)
```

**Lý do 2: Tránh old packets "nhầm" connection mới**
```
Connection 1: 10.0.1.1:5000 → 93.184.216.34:80 (close)
  ... một packet cũ vẫn đang "lang thang" trên mạng ...

Connection 2: 10.0.1.1:5000 → 93.184.216.34:80 (mới)
  → Packet cũ đến → bị nhầm là của connection 2!
  → Data corruption!

TIME_WAIT = 2×MSL đảm bảo mọi old packets đã expire trước khi reuse port
  MSL = Maximum Segment Lifetime ≈ 30s-2min
  TIME_WAIT = 60s-240s (Linux: 60s)
```

### TIME_WAIT "storm" — Vấn đề thực tế

```
Scenario: Web server (hoặc load balancer) kết nối đến backend

Mỗi HTTP request:
  LB → Backend (new TCP connection)
  Response received
  LB close connection (active close) → TIME_WAIT!

1000 requests/second × 60 seconds TIME_WAIT:
  = 60,000 sockets ở TIME_WAIT đồng thời!
  
Problems:
  - Memory: mỗi TIME_WAIT socket chiếm ~280 bytes (Linux)
  - Port exhaustion: chỉ có ~28,000 ephemeral ports!
    (32768-60999 default range = 28,232 ports)
  - "Cannot assign requested address" error!
```

### Giải pháp TIME_WAIT

#### Solution 1: tcp_tw_reuse (Recommended)

```bash
sysctl -w net.ipv4.tcp_tw_reuse=1

# Cho phép OUTGOING connections reuse TIME_WAIT sockets
# Điều kiện: timestamp option enabled + new conn timestamp > old
# An toàn vì: timestamps đảm bảo old packets bị reject
# Chỉ affect: client-side (outgoing connections)
```

#### Solution 2: SO_REUSEADDR (cho servers)

```c
int optval = 1;
setsockopt(sock, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval));

// Cho phép bind() port đang ở TIME_WAIT
// Cần thiết khi restart server nhanh
// "Address already in use" error → fix bằng SO_REUSEADDR
```

#### Solution 3: Connection pooling (Best practice)

```
Thay vì: 1 request = 1 TCP connection (open → use → close → TIME_WAIT)
Dùng:    Connection pool (open → use → return → reuse → use → return...)

HTTP: Keep-Alive / HTTP/2 multiplexing
DB:   Connection pool (HikariCP, pgBouncer)
gRPC: Long-lived connections with multiplexing
```

#### Solution 4: Adjust ephemeral port range

```bash
# Tăng range ephemeral ports:
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
# Từ 28,232 → 64,512 ports available!

# Kết hợp multiple source IPs (server có nhiều IP):
# → More source IPs × port range = more connections possible
```

#### ⚠️ CẤM: tcp_tw_recycle (REMOVED!)

```bash
# net.ipv4.tcp_tw_recycle — ĐÃ BỊ REMOVED trong Linux 4.12+!
# 
# Lý do removed:
# - Breaks connections behind NAT (per-connection timestamp)
# - Multiple clients behind 1 NAT IP → timestamp confusion
# - KHÔNG AN TOÀN — gây mất kết nối cho users
#
# ĐỪNG BAO GIỜ enable tcp_tw_recycle!
```

---

## 4. Connection Establishment — 3-Way Handshake chi tiết

### Step-by-Step

```
Client                                    Server
  │                                         │
  │ (CLOSED)                       (LISTEN) │
  │                                         │
  │──────── SYN (Seq=x) ──────────────────→│
  │ [SYN_SENT]                              │
  │                                         │ [SYN_RECEIVED]
  │←──── SYN+ACK (Seq=y, Ack=x+1) ────────│
  │                                         │
  │──────── ACK (Seq=x+1, Ack=y+1) ───────→│
  │ [ESTABLISHED]                           │ [ESTABLISHED]
  │                                         │
  │         ═══ DATA TRANSFER ═══          │
```

### SYN Queue và Accept Queue

```
Server side có 2 queues:

┌──────────────────────────────────────────────────┐
│                    SERVER                          │
│                                                  │
│  ┌─────────────┐        ┌─────────────┐         │
│  │  SYN Queue  │───────→│Accept Queue │───→ app  │
│  │ (Half-open) │  3WHS  │(Completed)  │  accept()│
│  │  Backlog    │complete │  Backlog    │         │
│  └─────────────┘        └─────────────┘         │
│                                                  │
│  Size: tcp_max_syn_backlog    Size: somaxconn    │
│  (default: 1024)             (default: 4096)    │
└──────────────────────────────────────────────────┘

SYN Queue (half-open):
  - Connections ở SYN_RECEIVED state
  - Chờ ACK cuối từ client
  - Size: net.ipv4.tcp_max_syn_backlog
  - Overflow → drop SYN hoặc SYN cookies

Accept Queue (fully established):
  - Connections ESTABLISHED nhưng app chưa accept()
  - Size: min(backlog parameter of listen(), net.core.somaxconn)
  - Overflow → drop hoặc RST
```

### SYN Flood Attack và Mitigation

```
Attack:
  Attacker gửi hàng triệu SYN packets (spoofed source IP)
  Server tạo SYN_RECEIVED entries → SYN queue đầy!
  Legitimate clients bị reject!

Mitigation:

1. SYN Cookies (net.ipv4.tcp_syncookies=1):
   - Server KHÔNG lưu state khi nhận SYN
   - Encode connection info vào SYN+ACK sequence number
   - Khi nhận ACK → decode → verify → accept
   - Zero state during handshake!

2. Increase backlog:
   sysctl -w net.ipv4.tcp_max_syn_backlog=65535
   sysctl -w net.core.somaxconn=65535

3. Rate limiting:
   iptables -A INPUT -p tcp --syn -m limit --limit 100/s --limit-burst 200 -j ACCEPT
   iptables -A INPUT -p tcp --syn -j DROP

4. TCP SYN timeout reduction:
   sysctl -w net.ipv4.tcp_synack_retries=2  # Default 5 → reduce to 2
```

### TCP Fast Open (TFO) — Bỏ qua 1 RTT

```
Normal: 3WHS (1 RTT) + Request (1 RTT) = 2 RTT before response

TFO: Request IN SYN packet = 1 RTT total!

First connection (obtain cookie):
  Client → SYN + TFO cookie request → Server
  Server → SYN+ACK + TFO cookie → Client
  Client stores cookie for future use

Subsequent connections:
  Client → SYN + TFO cookie + DATA ──────→ Server
                                           Server processes DATA immediately!
  Client ←── SYN+ACK + RESPONSE ──────── Server
  Client → ACK

Tiết kiệm 1 RTT cho mỗi new connection!
Enable: sysctl -w net.ipv4.tcp_fastopen=3 (client+server)
```

---

## 5. Connection Termination — 4-Way Handshake

### Normal Close (4-way)

```
Client (Active Close)                    Server (Passive Close)
  │ [ESTABLISHED]                [ESTABLISHED] │
  │                                            │
  │──────── FIN (Seq=u) ─────────────────────→│
  │ [FIN_WAIT_1]                               │
  │                               [CLOSE_WAIT] │
  │←──────── ACK (Ack=u+1) ──────────────────│
  │ [FIN_WAIT_2]                               │
  │                                            │ ← App still sending data...
  │                                            │ ← App calls close()
  │                                            │
  │←──────── FIN (Seq=v) ─────────────────────│
  │                                [LAST_ACK]  │
  │──────── ACK (Ack=v+1) ──────────────────→│
  │ [TIME_WAIT]                     [CLOSED]   │
  │                                            │
  │ ← Wait 2×MSL (60s) →                     │
  │ [CLOSED]                                   │
```

### Half-Close — Chỉ 1 bên đóng

```
TCP là FULL-DUPLEX → mỗi direction đóng độc lập!

Client: shutdown(sock, SHUT_WR)  → Gửi FIN (client không gửi nữa)
  NHƯNG client VẪN NHẬN data từ server!

Use case:
  - Client gửi xong request → half-close (SHUT_WR)
  - Server tiếp tục gửi response
  - Server gửi xong → server close → FIN
  - Cả 2 bên đều close

Ví dụ thực tế: HTTP/1.0 pipelining, file upload completion signal
```

### RST (Reset) — Abort Connection

```
RST = "HỦY ngay lập tức, không chờ đợi!"

Khi nào gửi RST:
1. SYN đến port không LISTEN → RST
2. App crash/kill -9 (abortive close)
3. Packet đến cho connection không tồn tại
4. Firewall reject (configured to RST vs DROP)
5. SO_LINGER with timeout=0 → close() gửi RST thay FIN
6. Half-open detection (peer crashed, we don't know)

RST vs FIN:
  FIN = "Tôi xong rồi, xin kết thúc lịch sự" (graceful)
  RST = "CÚP MÁY! Ngay bây giờ!" (abortive)
  
  FIN: buffered data sẽ được gửi trước khi close
  RST: discard all buffered data immediately!
```

---

## 6. Debugging TCP States — Tools và Techniques

### ss command (thay netstat)

```bash
# Tổng quan tất cả states:
ss -s
# Total: 1205 (kernel 1458)
# TCP:   952 (estab 847, closed 23, orphaned 5, timewait 67)
# 
# Transport   Total    IP    IPv6
# RAW         2        1     1
# UDP         12       8     4
# TCP         929      870   59
# INET        943      879   64

# Connections theo state:
ss -tn state established
ss -tn state time-wait
ss -tn state close-wait
ss -tn state syn-sent
ss -tn state syn-recv
ss -tn state fin-wait-1
ss -tn state fin-wait-2
ss -tn state last-ack
ss -tn state listening

# Count per state:
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
# 847 ESTAB
# 67  TIME-WAIT
# 12  LISTEN
# 5   CLOSE-WAIT
# 2   SYN-SENT
# 1   FIN-WAIT-2

# Connections with process info:
ss -tnp state close-wait
# CLOSE-WAIT 10.0.1.1:443 10.0.2.2:54321 users:(("nginx",pid=1234,fd=12))

# Filter by port:
ss -tn '( sport = :443 )'
ss -tn '( dport = :3306 )'

# Connection duration (timer info):
ss -tno state established
# Shows keepalive timer, retransmit timer

# Internal TCP info (cwnd, rtt, etc):
ss -ti state established
```

### One-liner monitoring scripts

```bash
# === Real-time state counter ===
watch -n2 "ss -tan | awk '{print \$1}' | sort | uniq -c | sort -rn"

# === Alert on CLOSE_WAIT > threshold ===
CLOSE_WAIT=$(ss -tn state close-wait | wc -l)
if [ $CLOSE_WAIT -gt 100 ]; then
  echo "WARNING: $CLOSE_WAIT CLOSE_WAIT connections!" | mail admin@example.com
fi

# === Top talkers (most connections to/from) ===
ss -tn state established | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head -10

# === TIME_WAIT per destination ===
ss -tn state time-wait | awk '{print $4}' | cut -d: -f1 | sort | uniq -c | sort -rn | head

# === Connection rate (new connections/sec) ===
# Track SYN_RECV count over time:
while true; do
  echo "$(date +%H:%M:%S) SYN_RECV: $(ss -tn state syn-recv | wc -l)"
  sleep 1
done
```

### tcpdump for state transitions

```bash
# Capture handshakes (SYN, SYN+ACK, ACK):
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0' -nn

# Capture only RST packets (connection issues):
tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0' -nn

# Capture FIN packets (connection closing):
tcpdump -i eth0 'tcp[tcpflags] & tcp-fin != 0' -nn

# Capture specific connection lifecycle:
tcpdump -i eth0 'host 10.0.2.2 and tcp port 443' -nn -S
# -S = absolute sequence numbers (not relative)
```

---

## 7. Common Problems và Solutions

### Problem 1: Quá nhiều TIME_WAIT

```
Diagnosis:
  ss -tn state time-wait | wc -l
  # 28000+ → approaching port exhaustion!

  # Check port range:
  cat /proc/sys/net/ipv4/ip_local_port_range
  # 32768    60999 → only 28,232 ports!

Root Causes:
  - Short-lived connections (no keep-alive/pooling)
  - Server-side active close (server closes first)
  - High connection rate (thousands/sec)

Solutions (priority order):
  1. Connection pooling / Keep-Alive (BEST)
  2. sysctl net.ipv4.tcp_tw_reuse=1
  3. Expand port range: ip_local_port_range="1024 65535"
  4. Multiple source IPs
  5. Let passive side close (change app logic)
```

### Problem 2: CLOSE_WAIT tích lũy (Memory Leak)

```
Diagnosis:
  ss -tnp state close-wait | awk '{print $NF}' | sort | uniq -c | sort -rn
  # 2000 users:(("java",pid=5555,fd=99))  ← JAVA APP BUG!

Root Causes:
  - App không close() socket khi peer disconnects
  - Missing finally{close()} block
  - Infinite loop blocking close path
  - Resource leak in connection handling

Solutions:
  1. FIX APPLICATION CODE (only real solution):
     - Ensure close() in finally/defer/with blocks
     - Handle IOException/ConnectionReset properly
     - Set socket timeouts
  
  2. Temporary mitigation:
     - Kill problematic process (restart app)
     - Increase file descriptor limits (delay symptom, not fix)
  
  3. Prevention:
     - SO_KEEPALIVE (detect dead connections)
     - Read timeout (don't wait forever)
     - Connection pool with idle cleanup

# Java example fix:
try (Socket socket = new Socket(host, port)) {
    // use socket
} // Auto-close in finally!

# Python:
with socket.create_connection((host, port)) as sock:
    # use socket
# Auto-close
```

### Problem 3: SYN_RECV flood

```
Diagnosis:
  ss -tn state syn-recv | wc -l
  # 10000+ → SYN flood attack!
  
  # Check SYN queue overflow:
  nstat -az TcpExtListenOverflows TcpExtListenDrops
  # TcpExtListenOverflows 5000 ← connections dropped!

Solutions:
  1. Enable SYN cookies:
     sysctl -w net.ipv4.tcp_syncookies=1
  
  2. Increase backlog:
     sysctl -w net.ipv4.tcp_max_syn_backlog=65535
     sysctl -w net.core.somaxconn=65535
  
  3. Reduce SYN+ACK retries:
     sysctl -w net.ipv4.tcp_synack_retries=2
  
  4. Rate limit:
     iptables -A INPUT -p tcp --syn -m limit --limit 500/s -j ACCEPT
     iptables -A INPUT -p tcp --syn -j DROP
  
  5. Use hardware/cloud DDoS protection
```

### Problem 4: "Address already in use" after restart

```
Diagnosis:
  # Try to start service:
  Error: bind(): Address already in use (port 8080)
  
  # Check who's using:
  ss -tlnp | grep :8080
  # Nothing listening? → TIME_WAIT holding the port!
  ss -tn state time-wait | grep :8080

Solution:
  1. SO_REUSEADDR in application (MUST):
     setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
  
  2. SO_REUSEPORT (multiple processes can bind same port):
     setsockopt(fd, SOL_SOCKET, SO_REUSEPORT, &opt, sizeof(opt));
  
  3. Wait for TIME_WAIT to expire (60s)
  
  4. Change port (temporary workaround)
```

---

## 8. TCP States trong Cloud và Container

### AWS ELB/ALB Connection Handling

```
ALB (Application Load Balancer):
  Client ←→ ALB ←→ Target (EC2/Container)
  
  ALB idle timeout: 60s (default)
    - Nếu no data trong 60s → ALB close connection!
    - Client side: FIN from ALB → CLOSE_WAIT on client
    - Target side: ALB gửi FIN → target vào CLOSE_WAIT
    
  Best practice:
    - Target keep-alive > ALB idle timeout
    - Client timeout > ALB idle timeout
    - ALB: 60s, Target: keep-alive=65s

NLB (Network Load Balancer):
  Connection idle timeout: 350s
  - NLB transparent (L4) → TCP states pass through
  - Client/Target handle their own TCP lifecycle
```

### Docker/Kubernetes Network

```
Container networking:
  - Mỗi container có network namespace riêng
  - TIME_WAIT accumulation per container → port exhaustion per Pod!
  
  Kubernetes Service (ClusterIP):
    - kube-proxy iptables/ipvs → track connections
    - Stale connections khi Pod restart → conntrack issues
    - CLOSE_WAIT on old Pod → conntrack entry remains

Solution:
  - Graceful shutdown (pre-stop hook + SIGTERM handling)
  - Readiness probe (stop traffic before termination)
  - terminationGracePeriodSeconds adequate
  - Connection draining on service mesh (Istio)
```

### Kubernetes Graceful Shutdown

```yaml
apiVersion: v1
kind: Pod
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]
    # Sleep 5s → kube-proxy updates iptables → no new connections
    # App receives SIGTERM → stop accepting new connections
    # Finish existing requests → close gracefully
    # 30s total for graceful shutdown
```

---

## 9. TCP State Monitoring — Production Setup

### Prometheus + Grafana

```bash
# node_exporter provides TCP state metrics:
# node_netstat_Tcp_CurrEstab — current ESTABLISHED
# node_netstat_Tcp_ActiveOpens — connections initiated
# node_netstat_Tcp_PassiveOpens — connections accepted

# Custom script for per-state metrics:
#!/bin/bash
# /usr/local/bin/tcp_states_exporter.sh
echo "# HELP tcp_connections TCP connections by state"
echo "# TYPE tcp_connections gauge"
for state in established syn-sent syn-recv fin-wait-1 fin-wait-2 time-wait close-wait close last-ack listen closing; do
  count=$(ss -tn state $state 2>/dev/null | tail -n +2 | wc -l)
  echo "tcp_connections{state=\"$state\"} $count"
done
```

### Alert thresholds

```yaml
# Prometheus alerting rules:
groups:
- name: tcp_alerts
  rules:
  - alert: HighCloseWait
    expr: tcp_connections{state="close-wait"} > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High CLOSE_WAIT connections ({{ $value }})"
      description: "Application may have socket leak"

  - alert: TimeWaitExhaustion
    expr: tcp_connections{state="time-wait"} > 20000
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "TIME_WAIT approaching port exhaustion"
      
  - alert: SynFlood
    expr: tcp_connections{state="syn-recv"} > 1000
    for: 30s
    labels:
      severity: critical
    annotations:
      summary: "Possible SYN flood attack"
```

---

## 10. Tổng kết và Quick Reference

### State Transition Summary Table

| State | Ai | Vào bằng | Ra bằng | Duration | Vấn đề nếu nhiều |
|---|---|---|---|---|---|
| LISTEN | Server | listen() | accept()/close() | Permanent | OK |
| SYN_SENT | Client | connect() | Recv SYN+ACK/timeout | <1 RTT | Firewall/down |
| SYN_RECV | Server | Recv SYN | Recv ACK/timeout | <1 RTT | SYN flood! |
| ESTABLISHED | Both | Handshake done | close()/Recv FIN | App lifetime | Connection leak |
| FIN_WAIT_1 | Active closer | Send FIN | Recv ACK | <1 RTT | Network issue |
| FIN_WAIT_2 | Active closer | Recv ACK | Recv FIN | Depends on peer | Peer app slow |
| CLOSE_WAIT | Passive | Recv FIN | App close() | App-dependent | **APP BUG!** |
| LAST_ACK | Passive | Send FIN | Recv ACK | <1 RTT | Network issue |
| TIME_WAIT | Active closer | Recv FIN | 2×MSL timeout | 60s (Linux) | Port exhaust |
| CLOSING | Both | Simultaneous FIN | Recv ACK | <1 RTT | Very rare |
| CLOSED | — | Done | — | — | — |

### Quick Troubleshooting Flowchart

```
Symptom: "Connection refused"
  → Check: ss -tlnp | grep :PORT
  → Not listening? → Start service / check bind address

Symptom: "Connection timed out"
  → Check: firewall, security group, route to destination
  → Many SYN_SENT? → Destination unreachable

Symptom: "Address already in use"
  → Check: ss -tn state time-wait | grep :PORT
  → Fix: SO_REUSEADDR in app code

Symptom: "Too many open files"
  → Check: ss -tnp state close-wait
  → Many CLOSE_WAIT? → FIX APP (socket not closed!)
  → Many ESTABLISHED? → Connection pool leak

Symptom: "Slow response / timeout"
  → Check: ss -ti (retransmissions, RTT)
  → High retransmits? → Network loss / congestion
  → Zero window? → Receiver overwhelmed

Symptom: Performance degradation under load
  → Check: ss -s (total connections)
  → TIME_WAIT > 20000? → Need connection pooling
  → SYN_RECV > 1000? → Possible attack / backlog full
```

### Essential sysctl for Production Servers

```bash
# /etc/sysctl.d/99-tcp-tuning.conf

# Connection handling
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_syncookies = 1

# TIME_WAIT management
net.ipv4.tcp_tw_reuse = 1
net.ipv4.ip_local_port_range = 1024 65535

# Timeouts
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5

# Buffers
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 131072 16777216
net.ipv4.tcp_wmem = 4096 16384 16777216

# Performance
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_timestamps = 1
```

---

*Tài liệu tham khảo:*
- RFC 9293 — Transmission Control Protocol (TCP)
- RFC 7323 — TCP Extensions for High Performance
- RFC 4987 — TCP SYN Flooding Attacks and Common Mitigations
- RFC 1337 — TIME-WAIT Assassination Hazards in TCP
- RFC 8985 — The RACK-TLP Loss Detection Algorithm
- Linux kernel TCP implementation — net/ipv4/tcp*.c
- ss(8) man page — socket statistics
- Stevens, W.R. — TCP/IP Illustrated, Volume 1
