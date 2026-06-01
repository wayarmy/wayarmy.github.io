---
layout: post
title: "WebSocket Protocol Deep Dive - Upgrade Handshake, Full-Duplex, Frames, Heartbeat, vs SSE vs Polling"
date: 2026-06-01
categories: [networking]
tags: [websocket, real-time, http, networking, protocol]
---

# WebSocket Protocol Deep Dive — Upgrade Handshake, Full-Duplex, Frames, Heartbeat, vs SSE vs Polling

## Mục lục (Table of Contents)
1. [Giới thiệu bằng câu chuyện đời thường](#1-giới-thiệu-bằng-câu-chuyện-đời-thường)
2. [Vấn đề mà WebSocket giải quyết](#2-vấn-đề-mà-websocket-giải-quyết)
3. [Upgrade Handshake — Quá trình "nâng cấp" kết nối](#3-upgrade-handshake--quá-trình-nâng-cấp-kết-nối)
4. [Full-Duplex Communication — Giao tiếp hai chiều đồng thời](#4-full-duplex-communication--giao-tiếp-hai-chiều-đồng-thời)
5. [WebSocket Frames — Cấu trúc khung dữ liệu](#5-websocket-frames--cấu-trúc-khung-dữ-liệu)
6. [Heartbeat — Ping/Pong và Connection Health](#6-heartbeat--pingpong-và-connection-health)
7. [WebSocket vs SSE vs Long Polling vs Short Polling](#7-websocket-vs-sse-vs-long-polling-vs-short-polling)
8. [Security và Production Considerations](#8-security-và-production-considerations)
9. [Hands-on Implementation](#9-hands-on-implementation)
10. [Tổng kết và Best Practices](#10-tổng-kết-và-best-practices)

---

## 1. Giới thiệu bằng câu chuyện đời thường

### WebSocket như một cuộc gọi điện thoại

Hãy so sánh các cách giao tiếp:

| Cách giao tiếp | Tương đương công nghệ | Đặc điểm |
|---|---|---|
| **Gửi thư** (viết → gửi → chờ hồi âm → viết tiếp) | HTTP Request/Response | Chậm, mỗi lần giao tiếp phải bắt đầu lại |
| **Gửi tin nhắn SMS liên tục** (hỏi "có tin mới không?" mỗi 5 giây) | Short Polling | Tốn tài nguyên, hỏi liên tục |
| **Gọi điện và chờ máy** (giữ đường dây, chỉ đợi bên kia nói) | Long Polling | Đỡ hơn, nhưng chỉ một chiều |
| **Cuộc gọi điện thoại thật** (cả hai nói/nghe cùng lúc) | **WebSocket** | Hai chiều, real-time, hiệu quả |

**WebSocket** giống như **mở một đường dây điện thoại** giữa trình duyệt và server — một khi kết nối được thiết lập, cả hai bên có thể gửi dữ liệu bất cứ lúc nào mà không cần "gọi lại" (tạo connection mới).

### Ứng dụng thực tế của WebSocket

- 💬 **Chat apps**: Messenger, Slack, Discord
- 📈 **Live dashboards**: Bảng giá chứng khoán, crypto
- 🎮 **Online gaming**: Multiplayer games
- 📊 **Real-time collaboration**: Google Docs, Figma
- 🔔 **Notifications**: Push notifications
- 📡 **IoT**: Sensor data streaming
- 💹 **Trading platforms**: Order book updates

---

## 2. Vấn đề mà WebSocket giải quyết

### 2.1 HTTP — Giao thức "hỏi-đáp" (Request/Response)

HTTP được thiết kế theo mô hình **client hỏi, server đáp**:

```
[Client] ──Request──→ [Server]
[Client] ←──Response── [Server]
[Connection closed]

// Mỗi lần cần data mới → phải tạo connection mới
[Client] ──Request──→ [Server]
[Client] ←──Response── [Server]
[Connection closed]
```

**Vấn đề**: Server KHÔNG THỂ chủ động gửi data cho client. Server chỉ phản hồi khi client hỏi.

### 2.2 Các giải pháp "hack" trước WebSocket

**Short Polling** (Hỏi liên tục):
```javascript
// Cứ mỗi 3 giây, gửi request kiểm tra
setInterval(() => {
  fetch('/api/messages/new')
    .then(res => res.json())
    .then(data => updateUI(data));
}, 3000);
```

**Nhược điểm**: 
- Tốn bandwidth (90% request trả về "không có gì mới")
- Latency cao (chờ tới 3 giây mới biết có tin mới)
- Server load cao (xử lý hàng nghìn request/giây)

**Long Polling** (Giữ connection chờ):
```javascript
async function longPoll() {
  const response = await fetch('/api/messages/wait'); // Server giữ 30s
  const data = await response.json();
  updateUI(data);
  longPoll(); // Gọi lại ngay
}
longPoll();
```

**Nhược điểm**:
- Server giữ connection mở → tốn resources
- Timeout management phức tạp
- Mỗi message vẫn cần HTTP header overhead

### 2.3 WebSocket giải quyết gì?

| Vấn đề | HTTP giải pháp | WebSocket giải pháp |
|---|---|---|
| Server muốn gửi data | Không thể (chỉ response) | ✅ Server push bất cứ lúc nào |
| Real-time updates | Polling (tốn tài nguyên) | ✅ Instant delivery |
| Header overhead | 200+ bytes mỗi request | ✅ 2-14 bytes per frame |
| Connection setup | TCP + TLS mỗi lần | ✅ Một lần duy nhất |
| Bidirectional | Không (client → server only) | ✅ Cả hai chiều đồng thời |

---

## 3. Upgrade Handshake — Quá trình "nâng cấp" kết nối

### 3.1 Quá trình handshake

WebSocket **bắt đầu bằng HTTP**, sau đó "nâng cấp" (upgrade) sang WebSocket protocol. Giống như bạn gọi điện thoại bình thường, rồi hai bên đồng ý chuyển sang video call.

```
[Client]                                    [Server]
   │                                            │
   │── HTTP GET + Upgrade: websocket ─────────→ │  Step 1: Client request upgrade
   │                                            │
   │←── HTTP 101 Switching Protocols ──────── │  Step 2: Server accepts
   │                                            │
   │←═══════ WebSocket Connection ═══════════→ │  Step 3: Full-duplex channel open
   │         (frames, no more HTTP)             │
```

### 3.2 Client Request (Opening Handshake)

```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: chat, superchat
Sec-WebSocket-Extensions: permessage-deflate
Origin: http://example.com
```

**Giải thích từng header:**

| Header | Giá trị | Ý nghĩa |
|---|---|---|
| `GET /chat` | Path endpoint | URL của WebSocket endpoint |
| `Upgrade: websocket` | Cố định | "Tôi muốn nâng cấp lên WebSocket" |
| `Connection: Upgrade` | Cố định | "Kết nối này cần được nâng cấp" |
| `Sec-WebSocket-Key` | Base64 random 16 bytes | Nonce để chống proxy cache |
| `Sec-WebSocket-Version` | 13 | Version của WebSocket protocol |
| `Sec-WebSocket-Protocol` | Subprotocols | Ứng dụng hỗ trợ protocols nào |
| `Sec-WebSocket-Extensions` | Extensions | Compression, multiplexing... |
| `Origin` | Origin URL | Chống CSRF |

### 3.3 Server Response

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: chat
```

**Sec-WebSocket-Accept** — Server chứng minh nó hiểu WebSocket:

```python
import hashlib
import base64

# Server tính toán:
client_key = "dGhlIHNhbXBsZSBub25jZQ=="
magic_string = "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"  # RFC 6455 constant

# Nối key + magic string
combined = client_key + magic_string
# → "dGhlIHNhbXBsZSBub25jZQ==258EAFA5-E914-47DA-95CA-C5AB0DC85B11"

# SHA-1 hash → Base64
accept_value = base64.b64encode(hashlib.sha1(combined.encode()).digest())
# → "s3pPLMBiTxaQ9kYGzzhZRbK+xOo="
```

**Tại sao cần magic string?** Để đảm bảo server thực sự hiểu WebSocket protocol chứ không phải proxy/cache "vô tình" trả lời 101.

### 3.4 Handshake qua TLS (WSS)

```
wss://server.example.com/chat
 ↑
 WebSocket Secure (giống HTTPS cho WebSocket)

Connection flow:
1. TCP handshake (SYN → SYN-ACK → ACK)
2. TLS handshake (Client Hello → Server Hello → Key Exchange)
3. HTTP Upgrade request (encrypted)
4. HTTP 101 response (encrypted)
5. WebSocket frames (all encrypted via TLS)
```

### 3.5 Subprotocols — Thỏa thuận "ngôn ngữ"

```http
# Client đề xuất nhiều subprotocols
Sec-WebSocket-Protocol: graphql-ws, graphql-transport-ws

# Server chọn 1
Sec-WebSocket-Protocol: graphql-transport-ws
```

Subprotocol phổ biến:
- `graphql-ws` — GraphQL over WebSocket
- `wamp` — Web Application Messaging Protocol
- `stomp` — Simple Text Oriented Messaging Protocol
- `mqtt` — Message Queuing Telemetry Transport

---

## 4. Full-Duplex Communication — Giao tiếp hai chiều đồng thời

### 4.1 Full-Duplex vs Half-Duplex vs Simplex

**Ví dụ đời thường:**

| Kiểu | Ví dụ | Đặc điểm |
|---|---|---|
| **Simplex** (một chiều) | Đài phát thanh | Chỉ 1 bên gửi |
| **Half-Duplex** (luân phiên) | Bộ đàm (walkie-talkie) | 1 bên nói, bên kia nghe, rồi đổi |
| **Full-Duplex** (đồng thời) | Điện thoại | Cả 2 nói/nghe cùng lúc |

```
Simplex:      A ──────→ B
Half-Duplex:  A ←─────→ B  (luân phiên)
Full-Duplex:  A ←═════→ B  (đồng thời)
              A ══════→ B  (client gửi)
              A ←══════ B  (server gửi)  ← CẢ HAI CÙNG LÚC
```

### 4.2 WebSocket Full-Duplex trong thực tế

```
Timeline:
────────────────────────────────────────────────────────►
t=0s    t=1s    t=2s    t=3s    t=4s    t=5s    t=6s

Client: ─[msg1]──────[msg2]──────────────[msg3]────────
Server: ──────[push1]────[push2]──[push3]───────[push4]─

Cả client và server gửi BẤT CỨ LÚC NÀO
Không cần chờ bên kia
Không cần "request" để nhận "response"
```

### 4.3 So sánh overhead: HTTP vs WebSocket

```
HTTP (mỗi message):
┌─────────────────────────────────────────────┐
│ TCP Handshake (3 packets)                   │  ← ~1 RTT
│ TLS Handshake (2-4 packets)                 │  ← ~2 RTT
│ HTTP Headers (200-2000 bytes)               │  ← overhead
│ HTTP Body (actual data)                     │  ← payload
│ TCP Close                                   │
└─────────────────────────────────────────────┘
Total overhead per message: ~500-2000 bytes + 3-5 RTT

WebSocket (sau handshake):
┌────────────────────────┐
│ Frame Header (2-14 bytes) │  ← tiny overhead
│ Frame Payload (data)      │  ← payload
└────────────────────────┘
Total overhead per message: 2-14 bytes + 0 RTT (connection đã mở)
```

**Ví dụ thực tế**: Chat app với 1000 users, mỗi user gửi 1 msg/giây:

| | HTTP Polling (1s interval) | WebSocket |
|---|---|---|
| Requests/sec | 1000 (polls) + 1000 (sends) = 2000 | 1000 (messages only) |
| Header overhead/sec | 2000 × 500 bytes = 1 MB | 1000 × 6 bytes = 6 KB |
| TCP connections | 2000 new connections/sec | 1000 persistent |
| Latency | 0-1000ms (average 500ms) | <50ms |

---

## 5. WebSocket Frames — Cấu trúc khung dữ liệu

### 5.1 Frame Format (RFC 6455 Section 5.2)

Mỗi WebSocket message được đóng gói thành **frames** (khung). Giống như gửi hàng trong container — hàng hóa (payload) được đóng trong khung (frame) có nhãn (header).

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-------+-+-------------+-------------------------------+
|F|R|R|R| opcode|M| Payload len |    Extended payload length    |
|I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
|N|V|V|V|       |S|             |   (if payload len==126/127)   |
| |1|2|3|       |K|             |                               |
+-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
|     Extended payload length continued, if payload len == 127  |
+ - - - - - - - - - - - - - - - +-------------------------------+
|                               |Masking-key, if MASK set to 1  |
+-------------------------------+-------------------------------+
| Masking-key (continued)       |          Payload Data         |
+-------------------------------- - - - - - - - - - - - - - - - +
:                     Payload Data continued ...                :
+ - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
|                     Payload Data continued ...                 |
+---------------------------------------------------------------+
```

### 5.2 Frame Header Fields

| Field | Bits | Ý nghĩa | Ví dụ đời thường |
|---|---|---|---|
| **FIN** | 1 | Frame cuối cùng? (1=yes) | "Đây là trang cuối của thư" |
| **RSV1-3** | 3 | Reserved (extensions dùng) | Dành cho tính năng mở rộng |
| **Opcode** | 4 | Loại frame | "Đây là thư / bưu kiện / giấy biên nhận" |
| **MASK** | 1 | Payload có masked? (client→server = 1) | "Nội dung được mã hóa nhẹ" |
| **Payload length** | 7+16/64 | Kích thước payload | Cân nặng bưu kiện |
| **Masking key** | 32 (nếu MASK=1) | Key để XOR payload | Mã giải mã |
| **Payload** | variable | Dữ liệu thực | Nội dung thư |

### 5.3 Opcodes — Loại frame

| Opcode | Hex | Tên | Mục đích |
|---|---|---|---|
| 0x0 | `%x0` | Continuation | Phần tiếp theo của message trước |
| 0x1 | `%x1` | **Text** | Data dạng UTF-8 text |
| 0x2 | `%x2` | **Binary** | Data dạng binary |
| 0x3-0x7 | | Reserved | Cho data frames tương lai |
| 0x8 | `%x8` | **Close** | Đóng connection |
| 0x9 | `%x9` | **Ping** | Heartbeat request |
| 0xA | `%xA` | **Pong** | Heartbeat response |
| 0xB-0xF | | Reserved | Cho control frames tương lai |

### 5.4 Data Frames — Text và Binary

**Text Frame** (Opcode 0x1):
```
Ví dụ: Gửi "Hello" (5 bytes)

Frame bytes: 81 05 48 65 6C 6C 6F
             ││ ││ └───────────────── Payload: "Hello"
             ││ └────────────────── Payload length: 5
             │└──────────────────── FIN=1, Opcode=0x1 (text)
             └───────────────────── Byte 1

Breakdown:
- 0x81 = 10000001 binary
  FIN=1 (final frame)
  RSV=000
  Opcode=0001 (text)
- 0x05 = 00000101
  MASK=0 (server→client, no mask)
  Length=5
```

**Binary Frame** (Opcode 0x2):
```
Dùng cho: Images, Protobuf, MessagePack, audio/video data
Cấu trúc giống text frame nhưng opcode = 0x2
```

### 5.5 Masking — Tại sao client phải mask?

**Quy tắc**: Client → Server: PHẢI mask. Server → Client: KHÔNG mask.

**Tại sao?** Chống **cache poisoning** ở proxy servers. Nếu không mask, kẻ tấn công có thể crafting frame giống HTTP response, khiến proxy cache lưu dữ liệu giả.

```python
# Masking algorithm (XOR)
masking_key = [0x37, 0xFA, 0x21, 0x3D]  # 4 random bytes

# Mask payload
original = b"Hello"
masked = bytes([
    original[i] ^ masking_key[i % 4] 
    for i in range(len(original))
])
# H(0x48) XOR 0x37 = 0x7F
# e(0x65) XOR 0xFA = 0x9F
# l(0x6C) XOR 0x21 = 0x4D
# l(0x6C) XOR 0x3D = 0x51
# o(0x6F) XOR 0x37 = 0x58
```

### 5.6 Message Fragmentation — Chia nhỏ message

Message lớn có thể chia thành nhiều frames:

```
Message "Hello World!" chia thành 3 fragments:

Frame 1: FIN=0, Opcode=0x1 (Text), Payload="Hell"
Frame 2: FIN=0, Opcode=0x0 (Continuation), Payload="o Wo"
Frame 3: FIN=1, Opcode=0x0 (Continuation), Payload="rld!"
                ^^^^                                 
                FIN=1 = frame cuối cùng

Server nhận 3 frames → ghép lại thành "Hello World!"
```

**Tại sao fragment?**
- Message quá lớn → không muốn buffer toàn bộ trong RAM
- Cho phép **interleave** control frames (ping/pong) giữa data fragments
- Streaming: gửi khi có data, không cần biết tổng size trước

### 5.7 Payload Length Encoding

```
Nếu payload ≤ 125 bytes:
  → Length = 7-bit value (trực tiếp trong byte 2)
  → Total header: 2 bytes (+ 4 bytes mask nếu client)

Nếu payload 126-65535 bytes:
  → Byte 2 = 126 (flag)
  → Next 2 bytes = actual length (16-bit unsigned)
  → Total header: 4 bytes (+ 4 bytes mask)

Nếu payload > 65535 bytes:
  → Byte 2 = 127 (flag)
  → Next 8 bytes = actual length (64-bit unsigned)
  → Total header: 10 bytes (+ 4 bytes mask)
  → Max: 2^63 bytes (lý thuyết)
```

---

## 6. Heartbeat — Ping/Pong và Connection Health

### 6.1 Vấn đề: Connection "chết lặng"

**Ví dụ đời thường**: Bạn đang nói chuyện điện thoại, bỗng không nghe thấy gì. Bạn nói "Alo? Bạn còn đó không?" — đó chính là **Ping**. Bên kia trả lời "Ừ, mình đây" — đó là **Pong**.

**Tại sao connection có thể "chết" mà không biết?**
- NAT/firewall timeout: Sau X phút không traffic → connection bị drop
- Network issue: Cable rút, WiFi mất
- Server crash: Process die, nhưng TCP chưa kịp gửi FIN
- Client crash: Browser tab crash, app kill

### 6.2 Ping Frame (Opcode 0x9)

```
Ping frame:
┌────────┬─────────┬──────────────────┐
│ FIN=1  │ Op=0x9  │ Payload (≤125B)  │
│        │ (Ping)  │ (optional data)   │
└────────┴─────────┴──────────────────┘

Quy tắc:
- Cả client và server đều có thể gửi Ping
- Payload tùy chọn (≤ 125 bytes)
- PHẢI reply bằng Pong
```

### 6.3 Pong Frame (Opcode 0xA)

```
Pong frame:
┌────────┬─────────┬──────────────────────────────┐
│ FIN=1  │ Op=0xA  │ Payload = COPY từ Ping       │
│        │ (Pong)  │ (phải giống hệt Ping data)   │
└────────┴─────────┴──────────────────────────────┘

Quy tắc:
- PHẢI gửi Pong khi nhận Ping
- Pong payload = Ping payload (copy nguyên)
- Nếu nhận nhiều Ping chồng chéo → có thể chỉ Pong cái cuối
- Unsolicited Pong (không có Ping) → được phép (dùng làm keepalive)
```

### 6.4 Heartbeat Implementation Patterns

**Pattern 1: Server-initiated Ping (Phổ biến nhất)**
```
Server: ──[Ping]────────[Ping]────────[Ping]────→
Client: ────────[Pong]────────[Pong]────────[Pong]→

Interval: 30-60 giây
Timeout: Nếu không nhận Pong trong 10s → close connection
```

**Pattern 2: Application-level heartbeat**
```javascript
// Thay vì dùng protocol-level Ping/Pong,
// dùng regular message với nội dung "heartbeat"
setInterval(() => {
  ws.send(JSON.stringify({type: "heartbeat", timestamp: Date.now()}));
}, 30000);

// Server reply
ws.on('message', (data) => {
  const msg = JSON.parse(data);
  if (msg.type === 'heartbeat') {
    ws.send(JSON.stringify({type: "heartbeat_ack", timestamp: Date.now()}));
  }
});
```

**Tại sao application-level heartbeat?**
- Một số proxy/load balancer xử lý Ping/Pong → server không thấy
- Application heartbeat đi qua toàn bộ stack → end-to-end verification
- Có thể mang thêm metadata (timestamp, server load...)

### 6.5 Connection Timeout Strategy

```
┌─────────────────────────────────────────────────┐
│          Connection Health State Machine          │
├─────────────────────────────────────────────────┤
│                                                   │
│  HEALTHY ──[no pong in 10s]──→ SUSPECT           │
│     ↑                              │              │
│     │                    [no pong in 30s]         │
│     │                              │              │
│  [pong received]                   ↓              │
│     │                           DEAD              │
│     └──────────────────────── (close + reconnect) │
│                                                   │
└─────────────────────────────────────────────────┘
```

### 6.6 Reconnection Strategy

```javascript
class WebSocketClient {
  constructor(url) {
    this.url = url;
    this.reconnectDelay = 1000; // Start 1s
    this.maxDelay = 30000;      // Max 30s
    this.connect();
  }

  connect() {
    this.ws = new WebSocket(this.url);
    
    this.ws.onopen = () => {
      console.log('Connected');
      this.reconnectDelay = 1000; // Reset delay on success
    };
    
    this.ws.onclose = () => {
      console.log(`Reconnecting in ${this.reconnectDelay}ms...`);
      setTimeout(() => this.connect(), this.reconnectDelay);
      
      // Exponential backoff with jitter
      this.reconnectDelay = Math.min(
        this.reconnectDelay * 2 + Math.random() * 1000,
        this.maxDelay
      );
    };
  }
}
```

---

## 7. WebSocket vs SSE vs Long Polling vs Short Polling

### 7.1 Server-Sent Events (SSE)

**SSE** là chuẩn HTML5 cho **server → client one-way streaming** qua HTTP.

```http
# Client request (regular HTTP GET)
GET /events HTTP/1.1
Accept: text/event-stream

# Server response (keeps connection open)
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

data: {"message": "Hello"}

data: {"message": "World"}

event: notification
data: {"title": "New order"}

id: 12345
data: {"type": "update", "value": 42}

retry: 3000
```

**SSE Format:**
```
event: <event-type>     ← Tùy chọn, default = "message"
id: <event-id>          ← ID để reconnect từ vị trí cũ
data: <payload>         ← Dữ liệu (có thể multi-line)
retry: <ms>             ← Client retry interval
                        ← Blank line = end of event
```

**JavaScript client:**
```javascript
const source = new EventSource('/events');

source.onmessage = (event) => {
  console.log(event.data);
};

source.addEventListener('notification', (event) => {
  showNotification(JSON.parse(event.data));
});

// Auto-reconnect built-in!
// Gửi Last-Event-ID header khi reconnect → resume from last event
```

### 7.2 So sánh chi tiết

| Tiêu chí | Short Polling | Long Polling | SSE | WebSocket |
|---|---|---|---|---|
| **Direction** | Client → Server | Client → Server | Server → Client | ↔ Bidirectional |
| **Protocol** | HTTP | HTTP | HTTP | WS (upgrade from HTTP) |
| **Connection** | New mỗi lần | Held open | Held open | Persistent |
| **Latency** | High (interval) | Medium | Low | Lowest |
| **Server push** | ❌ | ✅ (hack) | ✅ | ✅ |
| **Client push** | ✅ | ✅ (new request) | ❌ (cần HTTP riêng) | ✅ |
| **Browser support** | ✅ All | ✅ All | ✅ Modern | ✅ Modern |
| **Auto-reconnect** | N/A | Manual | ✅ Built-in | Manual |
| **Event ID/Resume** | ❌ | ❌ | ✅ Built-in | Manual |
| **Binary data** | ✅ | ✅ | ❌ (text only) | ✅ |
| **HTTP/2 multiplexing** | ✅ | ✅ | ✅ | ❌ (separate TCP) |
| **Proxy friendly** | ✅ | ⚠️ | ✅ | ⚠️ (upgrade) |
| **Scalability** | Poor | Medium | Good | Best (for bidirectional) |

### 7.3 Khi nào dùng gì?

| Use Case | Recommendation | Lý do |
|---|---|---|
| Chat application | **WebSocket** | Bidirectional, low latency |
| Live notifications | **SSE** | Server → client only, auto-reconnect |
| Stock ticker | **WebSocket** | High frequency, bidirectional (orders) |
| News feed updates | **SSE** | Server push, infrequent |
| Online gaming | **WebSocket** | Bidirectional, binary, low latency |
| Progress bar | **SSE** | One-way, simple |
| Form submission status | **Short Polling** | Simple, infrequent checks |
| Collaborative editing | **WebSocket** | Complex bidirectional sync |
| IoT sensor data | **WebSocket** | Binary, bidirectional commands |
| Simple dashboard refresh | **Short Polling** or **SSE** | Depends on update frequency |

### 7.4 Decision Flowchart

```
Cần real-time updates?
├── No → Regular HTTP (REST API)
└── Yes
    ├── Chỉ Server → Client?
    │   ├── Yes → Cần binary data?
    │   │         ├── No → SSE ✅
    │   │         └── Yes → WebSocket
    │   └── No (bidirectional)
    │       └── WebSocket ✅
    └── Update frequency?
        ├── Hiếm (>30s) → Long Polling hoặc SSE
        └── Thường xuyên (<5s) → WebSocket ✅
```

---

## 8. Security và Production Considerations

### 8.1 WebSocket Security Concerns

| Threat | Mô tả | Mitigation |
|---|---|---|
| **Cross-Site WebSocket Hijacking** | Attacker page mở WS tới victim server | Check Origin header |
| **DoS** | Mở nghìn connections, gửi large frames | Rate limiting, max connections |
| **Data injection** | Gửi malicious data qua WS | Input validation, sanitization |
| **Missing encryption** | ws:// (không TLS) | Luôn dùng wss:// |
| **Authentication bypass** | WS không có built-in auth | Token in handshake or first message |

### 8.2 Authentication cho WebSocket

```javascript
// Cách 1: Token trong query string (đơn giản, nhưng token lộ trong logs)
const ws = new WebSocket('wss://api.example.com/ws?token=jwt_here');

// Cách 2: Token trong first message (sau connect)
const ws = new WebSocket('wss://api.example.com/ws');
ws.onopen = () => {
  ws.send(JSON.stringify({type: 'auth', token: 'jwt_here'}));
};

// Cách 3: Cookie-based (nếu same origin)
// Browser tự gửi cookies trong handshake request

// Cách 4: Sec-WebSocket-Protocol header (hack — dùng protocol field cho token)
const ws = new WebSocket('wss://api.example.com/ws', ['jwt_token_here']);
```

### 8.3 Origin Validation

```python
# Server-side: Kiểm tra Origin header trong handshake
allowed_origins = ['https://myapp.com', 'https://admin.myapp.com']

def on_handshake(request):
    origin = request.headers.get('Origin')
    if origin not in allowed_origins:
        return reject(403, "Origin not allowed")
    # Accept handshake
```

### 8.4 Rate Limiting và Connection Management

```python
# Production WebSocket server considerations
class WebSocketServer:
    MAX_CONNECTIONS_PER_IP = 10
    MAX_MESSAGE_SIZE = 1_048_576  # 1MB
    MAX_MESSAGES_PER_SECOND = 50
    IDLE_TIMEOUT = 300  # 5 minutes without activity
    
    def on_connect(self, client):
        ip = client.remote_address
        if self.connections_from_ip(ip) >= self.MAX_CONNECTIONS_PER_IP:
            client.close(1008, "Too many connections from your IP")
            return
    
    def on_message(self, client, message):
        if len(message) > self.MAX_MESSAGE_SIZE:
            client.close(1009, "Message too big")
            return
        
        if client.message_rate() > self.MAX_MESSAGES_PER_SECOND:
            client.close(1008, "Rate limit exceeded")
            return
```

### 8.5 Close Codes (RFC 6455 Section 7.4)

| Code | Tên | Ý nghĩa |
|---|---|---|
| 1000 | Normal Closure | Đóng bình thường |
| 1001 | Going Away | Server shutdown hoặc page navigate |
| 1002 | Protocol Error | Protocol violation |
| 1003 | Unsupported Data | Nhận loại data không hỗ trợ |
| 1005 | No Status | Không có close code (internal) |
| 1006 | Abnormal Closure | Connection drop (no close frame) |
| 1007 | Invalid Payload | Data không đúng format (bad UTF-8) |
| 1008 | Policy Violation | Generic policy violation |
| 1009 | Message Too Big | Message vượt size limit |
| 1010 | Mandatory Extension | Server không hỗ trợ extension cần |
| 1011 | Internal Error | Server error unexpected |
| 1012 | Service Restart | Server restarting |
| 1013 | Try Again Later | Server overloaded |
| 1014 | Bad Gateway | Gateway received invalid response |
| 1015 | TLS Handshake Failure | TLS failure (internal) |
| 4000-4999 | Application-defined | Custom close codes |

### 8.6 Scaling WebSocket

```
Challenge: WebSocket = persistent connection = mỗi server giữ N connections
           Load balancer phải route đúng server

Solution 1: Sticky Sessions
┌─────────┐       ┌──────────┐
│ Client A │──────→│ Server 1  │ (always)
│ Client B │──────→│ Server 2  │ (always)
└─────────┘       └──────────┘

Solution 2: Pub/Sub (Redis) cho cross-server messaging
┌─────────┐       ┌──────────┐
│ Client A │──────→│ Server 1  │──┐
└─────────┘       └──────────┘  │
                                  ├──→ [Redis Pub/Sub]
┌─────────┐       ┌──────────┐  │
│ Client B │──────→│ Server 2  │──┘
└─────────┘       └──────────┘

Client A gửi msg cho Client B:
1. Server 1 nhận message từ A
2. Server 1 publish tới Redis channel
3. Server 2 subscribe, nhận message
4. Server 2 forward tới Client B
```

### 8.7 WebSocket Extensions

**permessage-deflate** — Compression:
```http
# Client request
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits

# Server response
Sec-WebSocket-Extensions: permessage-deflate; server_max_window_bits=15
```

Giảm bandwidth 60-80% cho text data (JSON messages).

---

## 9. Hands-on Implementation

### 9.1 Server (Node.js với ws library)

```javascript
const WebSocket = require('ws');
const http = require('http');

const server = http.createServer();
const wss = new WebSocket.Server({ server });

// Connection tracking
const clients = new Map();

wss.on('connection', (ws, req) => {
  const clientId = generateId();
  clients.set(clientId, ws);
  
  console.log(`Client ${clientId} connected from ${req.socket.remoteAddress}`);
  
  // Ping every 30 seconds
  const pingInterval = setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.ping();
    }
  }, 30000);
  
  ws.on('pong', () => {
    ws.isAlive = true;
  });
  
  ws.on('message', (data) => {
    try {
      const message = JSON.parse(data);
      handleMessage(clientId, message);
    } catch (e) {
      ws.send(JSON.stringify({error: 'Invalid JSON'}));
    }
  });
  
  ws.on('close', (code, reason) => {
    console.log(`Client ${clientId} disconnected: ${code} ${reason}`);
    clearInterval(pingInterval);
    clients.delete(clientId);
  });
  
  ws.on('error', (err) => {
    console.error(`Client ${clientId} error:`, err.message);
  });
  
  // Welcome message
  ws.send(JSON.stringify({type: 'welcome', id: clientId}));
});

// Broadcast to all clients
function broadcast(message) {
  const data = JSON.stringify(message);
  clients.forEach((ws) => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send(data);
    }
  });
}

server.listen(8080, () => console.log('Server running on :8080'));
```

### 9.2 Client (Browser JavaScript)

```javascript
class ChatClient {
  constructor(url) {
    this.url = url;
    this.handlers = new Map();
    this.connect();
  }
  
  connect() {
    this.ws = new WebSocket(this.url);
    
    this.ws.onopen = () => {
      console.log('Connected!');
      this.startHeartbeat();
    };
    
    this.ws.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      const handler = this.handlers.get(msg.type);
      if (handler) handler(msg);
    };
    
    this.ws.onclose = (event) => {
      console.log(`Disconnected: ${event.code} ${event.reason}`);
      this.stopHeartbeat();
      // Reconnect with exponential backoff
      setTimeout(() => this.connect(), this.getReconnectDelay());
    };
    
    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };
  }
  
  send(type, payload) {
    if (this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({type, ...payload, timestamp: Date.now()}));
    }
  }
  
  on(type, handler) {
    this.handlers.set(type, handler);
  }
  
  startHeartbeat() {
    this.heartbeatInterval = setInterval(() => {
      this.send('ping', {});
    }, 25000);
  }
  
  stopHeartbeat() {
    clearInterval(this.heartbeatInterval);
  }
}

// Usage
const chat = new ChatClient('wss://api.example.com/ws');
chat.on('welcome', (msg) => console.log('My ID:', msg.id));
chat.on('message', (msg) => console.log(`${msg.from}: ${msg.text}`));
chat.send('message', {text: 'Hello everyone!', room: 'general'});
```

### 9.3 Debugging WebSocket

```bash
# 1. wscat — CLI WebSocket client
npm install -g wscat
wscat -c wss://echo.websocket.org
> Hello
< Hello

# 2. curl (kiểm tra handshake)
curl -v -N \
  -H "Connection: Upgrade" \
  -H "Upgrade: websocket" \
  -H "Sec-WebSocket-Version: 13" \
  -H "Sec-WebSocket-Key: $(openssl rand -base64 16)" \
  http://localhost:8080/ws

# 3. Chrome DevTools
# Network tab → Filter: WS
# Click connection → Messages tab → xem frames

# 4. Wireshark filter
# websocket (bắt WebSocket frames)
# tcp.port == 8080 && websocket.opcode == 1 (chỉ text frames)
```

### 9.4 Load Testing WebSocket

```javascript
// Simple load test with ws library
const WebSocket = require('ws');

const TARGET = 'wss://api.example.com/ws';
const NUM_CLIENTS = 1000;
const MSG_INTERVAL = 1000; // 1 message/second/client

let connected = 0;
let messages_sent = 0;

for (let i = 0; i < NUM_CLIENTS; i++) {
  setTimeout(() => {
    const ws = new WebSocket(TARGET);
    ws.on('open', () => {
      connected++;
      setInterval(() => {
        ws.send(JSON.stringify({type: 'msg', id: i, t: Date.now()}));
        messages_sent++;
      }, MSG_INTERVAL);
    });
    ws.on('error', () => {});
  }, i * 10); // Stagger connections
}

setInterval(() => {
  console.log(`Connected: ${connected}, Messages/sec: ${messages_sent}`);
  messages_sent = 0;
}, 1000);
```

---

## 10. Tổng kết và Best Practices

### 10.1 WebSocket Protocol Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                    WebSocket Protocol Stack                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Application Layer                                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Your Protocol (JSON messages, Protobuf, custom)          │   │
│  └──────────────────────────────────────────────────────────┘   │
│                          ↕                                        │
│  WebSocket Frame Layer (RFC 6455)                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Framing: FIN | Opcode | Mask | Length | Payload          │   │
│  │  Control: Ping/Pong/Close                                 │   │
│  │  Extensions: permessage-deflate                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                          ↕                                        │
│  HTTP (Handshake only)                                           │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  GET /ws → 101 Switching Protocols                        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                          ↕                                        │
│  TLS (for wss://)                                                │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Encryption, Certificate verification                     │   │
│  └──────────────────────────────────────────────────────────┘   │
│                          ↕                                        │
│  TCP                                                              │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Reliable, ordered delivery                               │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Best Practices

| Category | Practice | Lý do |
|---|---|---|
| Security | Luôn dùng `wss://` (TLS) | Chống eavesdropping |
| Security | Validate Origin header | Chống CSWSH attacks |
| Security | Authenticate on handshake/first message | Không cho anonymous access |
| Performance | Enable `permessage-deflate` | Giảm 60-80% bandwidth |
| Performance | Batch messages khi có thể | Giảm frame overhead |
| Reliability | Implement heartbeat (30s ping) | Phát hiện dead connections |
| Reliability | Exponential backoff reconnect | Không flood server khi outage |
| Reliability | Message acknowledgment | Đảm bảo delivery |
| Scalability | Pub/Sub cho cross-server messaging | Scale horizontally |
| Scalability | Set max connections per server | Tránh overload |
| Operations | Log connection lifecycle | Debug, monitoring |
| Operations | Monitor connection count, message rate | Alerting |

### 10.3 Tài liệu tham khảo

| Tài liệu | Nội dung |
|---|---|
| RFC 6455 | The WebSocket Protocol |
| RFC 7692 | Compression Extensions for WebSocket |
| RFC 8441 | Bootstrapping WebSockets with HTTP/2 |
| HTML Living Standard — Server-Sent Events | SSE specification |
| MDN WebSocket API | Browser API documentation |

### 10.4 Câu hỏi ôn tập

1. WebSocket giải quyết vấn đề gì mà HTTP không thể?
2. Mô tả quá trình upgrade handshake. Tại sao cần Sec-WebSocket-Key?
3. Full-duplex khác half-duplex thế nào?
4. Liệt kê các opcodes và ý nghĩa. Ping/Pong dùng làm gì?
5. Tại sao client phải mask data gửi lên server?
6. Message fragmentation hoạt động thế nào?
7. So sánh WebSocket vs SSE. Khi nào dùng SSE tốt hơn?
8. Làm sao scale WebSocket server ra nhiều instances?
9. Các close codes quan trọng? Khi nào dùng 1008 vs 1001?
10. Thiết kế heartbeat strategy cho production WebSocket service.

---

*Bài viết được tham khảo từ RFC 6455 (The WebSocket Protocol), RFC 7692 (WebSocket Compression), HTML Living Standard (Server-Sent Events), và MDN Web Docs.*
