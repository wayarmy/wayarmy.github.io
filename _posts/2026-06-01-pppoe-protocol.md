---
layout: post
title: "PPPoE Protocol - PPP over Ethernet, DSL/Fiber, PAP/CHAP, MTU và PADI/PADO"
date: 2026-06-01
categories: [networking]
tags: [pppoe, ppp, dsl, fiber, authentication, isp]
---

## 1. Giới thiệu — Thẻ ra vào tòa nhà

Hãy tưởng tượng bạn sống trong một **chung cư cao cấp**. Mỗi sáng khi đi làm:

1. Bạn **quẹt thẻ** ở cổng (Authentication — xác thực danh tính)
2. Cổng kiểm tra: "Thẻ này có hợp lệ không? Đúng cư dân không?" (PAP/CHAP)
3. Nếu hợp lệ, cổng **mở** và ghi nhận "Anh A ra lúc 7:30" (Session established)
4. Bạn đi qua cổng ra đường lớn (kết nối Internet)
5. Khi về, quẹt thẻ lần nữa → hệ thống biết "Anh A về lúc 18:00" (Session tracked)

**PPPoE (Point-to-Point Protocol over Ethernet)** hoạt động y hệt — đây là "thẻ ra vào Internet" mà ISP cấp cho bạn:
- **Username/Password** = Thẻ cư dân
- **BRAS/AC (Access Concentrator)** = Cổng bảo vệ
- **PPPoE Session** = Phiên ra/vào được theo dõi
- **MTU 1492** = Cổng hơi hẹp hơn cổng bình thường (1500)

### Tại sao bạn gặp PPPoE hàng ngày?

Nếu bạn dùng Internet ở Việt Nam (FPT, Viettel, VNPT), **rất có thể** modem/router nhà bạn đang chạy PPPoE ngay lúc này! Mỗi lần modem bật lên, nó:
1. Tìm BRAS của ISP (PADI/PADO)
2. Đăng nhập bằng username/password ISP cấp (Authentication)
3. Nhận IP address (IPCP)
4. Bắt đầu truyền dữ liệu Internet

### Tại sao ISP dùng PPPoE?

| Lý do | Chi tiết |
|---|---|
| Authentication | Xác thực mỗi subscriber bằng username/password |
| Accounting | Theo dõi thời gian sử dụng, data usage per user |
| Access Control | Gán IP, bandwidth, policy per user |
| Billing | PPPoE session = billable session |
| Scalability | 1 BRAS quản lý hàng chục nghìn subscribers |
| Legacy compatibility | DSL (ADSL/VDSL) thiết kế xung quanh PPP |

---

## 2. PPPoE là gì? — Giải thích cho người không biết IT

### Phép so sánh đời thường

Tưởng tượng bạn đến **quán Internet café** (nhớ thời 2000s?):

1. Bạn đến quầy **xin tài khoản** — nhân viên cho username/password
2. Bạn ngồi vào máy, **nhập username/password** → hệ thống xác nhận
3. Đồng hồ **bắt đầu đếm** — bạn bắt đầu dùng Internet
4. Khi xong, bạn **log out** → hệ thống ghi nhận thời gian + data đã dùng
5. Quầy **tính tiền** dựa trên thời gian/data

PPPoE = Hệ thống quản lý **quán Internet café** nhưng ở quy mô ISP, cho hàng triệu users!

### Định nghĩa kỹ thuật

**PPPoE** là protocol kết hợp 2 công nghệ:

1. **PPP (Point-to-Point Protocol)** — Protocol cung cấp:
   - Authentication (PAP/CHAP/EAP)
   - IP address assignment (IPCP)
   - Compression negotiation
   - Multilink (bonding nhiều connections)
   - LCP (Link Control Protocol) — negotiation parameters

2. **Ethernet** — Transport layer:
   - Mạng LAN phổ biến nhất
   - Broadcast medium (nhiều devices chia sẻ)
   - MTU 1500 bytes

**PPPoE = PPP được "đóng gói" trong Ethernet frames**

Vấn đề: PPP thiết kế cho **Point-to-Point** (2 điểm). Ethernet là **Multi-Access** (nhiều điểm). PPPoE giải quyết bằng **Discovery phase** — tìm đúng AC trước khi thiết lập PPP session.

### Vị trí trong mô hình mạng

```
┌─────────────────────────────────────────────────────┐
│ Layer   │ Without PPPoE    │ With PPPoE             │
├─────────┼──────────────────┼────────────────────────┤
│ App     │ HTTP, FTP...     │ HTTP, FTP...           │
│ Transport│ TCP/UDP          │ TCP/UDP                │
│ Network │ IP               │ IP                     │
│         │                  │ PPP (Auth + Framing)   │
│         │                  │ PPPoE (Session mgmt)   │
│ Data Link│ Ethernet         │ Ethernet               │
│ Physical│ UTP/Fiber        │ UTP/Fiber              │
└─────────┴──────────────────┴────────────────────────┘

Encapsulation:
┌──────────────────────────────────────────────────────────┐
│ Ethernet │ PPPoE  │ PPP    │ IP     │ TCP  │ Data      │
│ Header   │ Header │ Header │ Header │ Hdr  │           │
│ 14 bytes │ 6 bytes│ 2 bytes│ 20 bytes│20 B  │ ≤1452 B  │
└──────────────────────────────────────────────────────────┘
│←─ MTU trên Ethernet = 1500 bytes ─────────────────────→│
         │←─ PPPoE Payload = 1492 bytes ──────────────→│
                   │←─ PPP Payload = 1492 bytes ─────→│
                          │←─ IP MTU = 1492 bytes ──→│
```

---

## 3. PPPoE Discovery Phase — Tìm nhau trên mạng

### Mini-example: Gọi taxi

Discovery Phase giống **gọi taxi bằng app**:
1. **PADI** = Bạn mở app, broadcast: "Có ai rảnh không?" (gửi cho TẤT CẢ taxi)
2. **PADO** = Taxi gần đó reply: "Tôi rảnh, tôi là taxi ABC" (có thể nhiều taxi reply)
3. **PADR** = Bạn chọn 1 taxi: "Tôi chọn taxi ABC, đến đón tôi"
4. **PADS** = Taxi confirm: "OK, session bắt đầu, mã chuyến = 12345"
5. (Sau này) **PADT** = "Chuyến đi kết thúc" — bạn hoặc taxi terminate

### Chi tiết Discovery Phase

```
┌──────────┐                              ┌──────────────┐
│ PPPoE    │                              │ Access       │
│ Client   │                              │ Concentrator │
│(Router/PC)│                              │ (BRAS/BNG)  │
└─────┬────┘                              └──────┬───────┘
      │                                          │
      │──── PADI (Broadcast) ───────────────────→│
      │     Ether Type: 0x8863                   │
      │     Dest MAC: FF:FF:FF:FF:FF:FF          │
      │     Code: 0x09                           │
      │     Session ID: 0x0000                   │
      │     Tags: Service-Name, Host-Uniq        │
      │                                          │
      │←─── PADO (Unicast) ─────────────────────│
      │     Ether Type: 0x8863                   │
      │     Code: 0x07                           │
      │     Session ID: 0x0000                   │
      │     Tags: AC-Name, Service-Name,         │
      │           AC-Cookie, Relay-Session-ID    │
      │                                          │
      │──── PADR (Unicast) ────────────────────→│
      │     Ether Type: 0x8863                   │
      │     Code: 0x19                           │
      │     Session ID: 0x0000                   │
      │     Tags: Service-Name, Host-Uniq,       │
      │           AC-Cookie                      │
      │                                          │
      │←─── PADS (Unicast) ─────────────────────│
      │     Ether Type: 0x8863                   │
      │     Code: 0x65                           │
      │     Session ID: 0x0012 ← ASSIGNED!      │
      │     Tags: Service-Name                   │
      │                                          │
      │════ PPPoE SESSION ESTABLISHED ══════════│
      │     (Session ID = 0x0012)                │
      │     (Now move to PPP Session Phase)      │
      │                                          │
   ...later...                                   │
      │                                          │
      │──── PADT (either direction) ───────────→│
      │     Code: 0x0A7                          │
      │     Session ID: 0x0012                   │
      │     "Goodbye, session terminated"        │
      │                                          │
```

### PPPoE Header Format

```
PPPoE Frame trong Ethernet:

┌────────────────────────────────────────────────────────────────┐
│ Ethernet Header (14 bytes)                                      │
│ ├── Destination MAC: 6 bytes                                   │
│ ├── Source MAC: 6 bytes                                        │
│ └── EtherType: 2 bytes                                         │
│     - 0x8863 = PPPoE Discovery                                 │
│     - 0x8864 = PPPoE Session                                   │
├────────────────────────────────────────────────────────────────┤
│ PPPoE Header (6 bytes)                                          │
│ ├── Version: 4 bits (always 0x1)                               │
│ ├── Type: 4 bits (always 0x1)                                  │
│ ├── Code: 1 byte                                               │
│ │   Discovery: 0x09=PADI, 0x07=PADO, 0x19=PADR,              │
│ │              0x65=PADS, 0xA7=PADT                            │
│ │   Session:   0x00 (session data)                             │
│ ├── Session ID: 2 bytes (0 during discovery, assigned in PADS)│
│ └── Length: 2 bytes (PPPoE payload length)                     │
├────────────────────────────────────────────────────────────────┤
│ PPPoE Payload (variable)                                        │
│ ├── Discovery: TLV Tags                                        │
│ └── Session: PPP Frame                                         │
└────────────────────────────────────────────────────────────────┘
```

### Discovery Tags (TLV)

```
┌──────────────────────────────────────────────────────────────┐
│ Tag Type │ Tag Name          │ Purpose                        │
├──────────┼───────────────────┼────────────────────────────────┤
│ 0x0101   │ Service-Name      │ Dịch vụ yêu cầu ("" = bất kỳ)│
│ 0x0102   │ AC-Name           │ Tên Access Concentrator        │
│ 0x0103   │ Host-Uniq         │ Client identifier (correlation)│
│ 0x0104   │ AC-Cookie         │ Anti-DoS cookie từ AC          │
│ 0x0105   │ Vendor-Specific   │ Vendor extensions              │
│ 0x0110   │ Relay-Session-ID  │ Cho relay agents               │
│ 0x0201   │ Service-Name-Error│ Service không available         │
│ 0x0202   │ AC-System-Error   │ AC gặp lỗi                    │
│ 0x0203   │ Generic-Error     │ Lỗi chung                     │
└──────────┴───────────────────┴────────────────────────────────┘

Ví dụ PADI packet:
- Service-Name: "" (bất kỳ service nào)
- Host-Uniq: 0xDEADBEEF (để match PADO reply)

Ví dụ PADO packet:
- AC-Name: "BRAS-HCM-01"
- Service-Name: "Internet-50Mbps"
- AC-Cookie: 0x1234ABCD... (anti-DoS, client phải echo lại)
```

### Trong thực tế

**Tại sao Discovery cần thiết?**
```
Scenario: ISP có 10,000 subscribers trên cùng Ethernet network (DSLAM/OLT)

Không có Discovery:
- Subscriber không biết gửi PPP frame cho AI
- Ethernet là shared medium — nhiều AC có thể tồn tại
- Không cách nào establish point-to-point session

Với Discovery:
1. Subscriber broadcast PADI → tất cả AC nhận
2. AC1 (Internet service) gửi PADO
   AC2 (IPTV service) cũng gửi PADO
3. Subscriber chọn AC1 (Internet) → gửi PADR cho AC1
4. AC1 confirm → Session established → từ giờ unicast
5. PPP frames chạy TRONG session (unicast, tracked)
```

### Trong AWS

PPPoE không được sử dụng trực tiếp trong AWS (AWS dùng VPC networking). Tuy nhiên:
- **Direct Connect**: Physical layer có thể đi qua ISP infrastructure dùng PPPoE trên last-mile
- **VPN over ISP**: VPN tunnel thường chạy trên PPPoE Internet connection
- **MTU consideration**: Nếu on-premises dùng PPPoE (MTU 1492), VPN tunnel MTU phải giảm thêm:
  ```
  Standard Ethernet: 1500
  PPPoE: 1492 (1500 - 8 PPPoE header)
  IPSec VPN over PPPoE: ~1422 (1492 - ~70 IPSec overhead)
  GRE+IPSec over PPPoE: ~1398
  ```

---

## 4. PPP Session Phase — Authentication và Negotiation

### Mini-example: Check-in khách sạn

Sau khi tìm được khách sạn (Discovery), bạn cần:
1. **Trình CMND** (LCP Negotiation — đồng ý cách giao tiếp)
2. **Xác nhận booking** (Authentication — PAP hoặc CHAP)
3. **Nhận chìa khóa phòng** (IPCP — nhận IP address)
4. **Sử dụng phòng** (Data transfer)

### PPP Session Phases

```
After PPPoE Discovery (Session ID assigned):

Phase 1: LCP (Link Control Protocol)
┌──────────┐                    ┌──────────┐
│  Client  │──LCP Conf-Req────→│    AC    │
│          │    MRU=1492        │  (BRAS)  │
│          │    Auth=CHAP       │          │
│          │    Magic Number    │          │
│          │←─LCP Conf-Req─────│          │
│          │    MRU=1492        │          │
│          │    Auth=CHAP       │          │
│          │──LCP Conf-Ack────→│          │
│          │←─LCP Conf-Ack─────│          │
│          │                    │          │
│          │ [LCP OPEN]         │          │

Phase 2: Authentication (PAP or CHAP)
│          │                    │          │
│          │←─CHAP Challenge───│          │
│          │    Challenge value │          │
│          │──CHAP Response────→│          │
│          │    Response hash   │          │
│          │←─CHAP Success─────│          │
│          │    "Welcome!"      │          │
│          │                    │          │
│          │ [AUTH SUCCESS]     │          │

Phase 3: NCP — IPCP (IP Control Protocol)
│          │                    │          │
│          │──IPCP Conf-Req───→│          │
│          │    IP=0.0.0.0      │          │
│          │    DNS=0.0.0.0     │          │
│          │←─IPCP Conf-Nak────│          │
│          │    IP=118.70.x.x   │          │ ← ISP cấp IP
│          │    DNS=8.8.8.8     │          │
│          │──IPCP Conf-Req───→│          │
│          │    IP=118.70.x.x   │          │
│          │←─IPCP Conf-Ack────│          │
│          │                    │          │
│          │ [NETWORK OPEN]     │          │
│          │ Traffic can flow!  │          │
└──────────┘                    └──────────┘
```

### PAP vs CHAP Authentication

```
┌─────────────────────────────────────────────────────────────────┐
│                    PAP (Password Authentication Protocol)         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Cách hoạt động:                                                  │
│ Client gửi username + password → AC verify → Accept/Reject      │
│                                                                   │
│ Client → AC: Username="user01", Password="mypassword123"        │
│ AC → Client: Auth-Ack (success) hoặc Auth-Nak (failure)         │
│                                                                   │
│ ⚠️ VẤN ĐỀ: Password gửi CLEAR TEXT!!!                          │
│    → Có thể bị sniff trên đường truyền                          │
│    → Chỉ nên dùng nếu kênh đã encrypted (SSL, VPN)             │
│    → Hoặc trên physical connection tin cậy (DX fiber)           │
│                                                                   │
│ Ưu điểm: Đơn giản, compatible rộng                              │
│ Nhược điểm: KHÔNG AN TOÀN                                       │
│                                                                   │
├─────────────────────────────────────────────────────────────────┤
│                    CHAP (Challenge-Handshake Authentication)      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│ Cách hoạt động (3-way):                                         │
│ 1. AC gửi CHALLENGE (random value)                              │
│ 2. Client tính: Hash = MD5(ID + Password + Challenge)           │
│ 3. Client gửi RESPONSE (hash value)                             │
│ 4. AC tính hash tương tự, so sánh → Accept/Reject              │
│                                                                   │
│ AC → Client: Challenge (random: 0xA1B2C3D4...)                  │
│ Client: Hash = MD5(0x01 + "mypassword123" + 0xA1B2C3D4)        │
│ Client → AC: Response (hash: 0x7F8E9D...)                       │
│ AC: Tính hash + so sánh → Match → Auth-Ack                      │
│                                                                   │
│ ✓ Password KHÔNG BAO GIỜ gửi trên dây!                         │
│ ✓ Mỗi challenge KHÁC NHAU → replay attack impossible           │
│ ✓ Có thể re-authenticate periodic (CHAP re-challenge)          │
│                                                                   │
│ Ưu điểm: An toàn, anti-replay, periodic re-auth                │
│ Nhược điểm: Phức tạp hơn, cần shared secret                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘

So sánh:
┌────────────┬─────────────────┬─────────────────────────────┐
│ Tiêu chí   │ PAP             │ CHAP                        │
├────────────┼─────────────────┼─────────────────────────────┤
│ Security   │ Weak (plaintext)│ Strong (hash-based)         │
│ Replay     │ Vulnerable      │ Protected (unique challenge)│
│ Re-auth    │ No (once only)  │ Yes (periodic re-challenge) │
│ Direction  │ Client → Server │ Server challenges Client    │
│ Complexity │ Simple          │ More complex                │
│ ISP use    │ Still common!   │ Preferred                   │
│ RFC        │ RFC 1334        │ RFC 1994                    │
└────────────┴─────────────────┴─────────────────────────────┘
```

### LCP Options Negotiation

```
LCP negotiates parameters TRƯỚC authentication:

Important LCP Options:
┌──────────┬───────────────────────────────────────────────────┐
│ Option   │ Meaning                                           │
├──────────┼───────────────────────────────────────────────────┤
│ MRU      │ Maximum Receive Unit (max PPP payload size)       │
│          │ PPPoE: 1492 (Ethernet 1500 - 8 PPPoE header)     │
├──────────┼───────────────────────────────────────────────────┤
│ Auth     │ Authentication protocol to use                    │
│          │ 0xC023 = PAP, 0xC223 = CHAP                     │
├──────────┼───────────────────────────────────────────────────┤
│ Magic    │ Random number for loop detection                  │
│ Number   │ If received own magic → loopback detected!       │
├──────────┼───────────────────────────────────────────────────┤
│ Quality  │ Link quality monitoring (optional)                │
│ Protocol │                                                   │
├──────────┼───────────────────────────────────────────────────┤
│ PFC      │ Protocol Field Compression                        │
│ ACFC     │ Address/Control Field Compression                 │
└──────────┴───────────────────────────────────────────────────┘

LCP Packet Types:
- Configure-Request (Conf-Req): "Tôi muốn dùng options này"
- Configure-Ack (Conf-Ack): "OK, tôi đồng ý tất cả"
- Configure-Nak (Conf-Nak): "Không đồng ý, dùng values này thay thế"
- Configure-Reject (Conf-Rej): "Option này tôi không hỗ trợ, bỏ đi"
- Terminate-Request/Ack: Đóng connection
- Echo-Request/Reply: Keepalive (detect link failure)
```

### Trong thực tế

**ISP Authentication Flow (RADIUS + PPPoE):**
```
┌────────┐      ┌─────────┐      ┌──────────┐      ┌─────────┐
│Customer│      │  BRAS   │      │ RADIUS   │      │  DHCP/  │
│ Router │      │  (AC)   │      │ Server   │      │  Pool   │
└───┬────┘      └────┬────┘      └────┬─────┘      └────┬────┘
    │                │                │                  │
    │──PADI────────→│                │                  │
    │←─PADO─────────│                │                  │
    │──PADR────────→│                │                  │
    │←─PADS─────────│                │                  │
    │                │                │                  │
    │──LCP Req─────→│                │                  │
    │←─LCP Ack──────│                │                  │
    │                │                │                  │
    │←─CHAP Chall───│                │                  │
    │──CHAP Resp───→│                │                  │
    │                │──Access-Req──→│                  │
    │                │  (user+hash)  │                  │
    │                │               │──Check DB──→    │
    │                │               │←─User found──   │
    │                │←─Access-Accept│                  │
    │                │  (IP pool,    │                  │
    │                │   bandwidth,  │                  │
    │                │   session-timeout)│              │
    │←─CHAP Success─│                │                  │
    │                │                │                  │
    │──IPCP Req────→│                │                  │
    │                │───────────────────── Allocate IP─→│
    │                │←──────────────────── IP: 118.70.x.x│
    │←─IPCP Nak─────│ (Here's your IP)                 │
    │──IPCP Req────→│                │                  │
    │←─IPCP Ack─────│                │                  │
    │                │                │                  │
    │═══ DATA ══════│════════════════│══════════════════│
    │                │                │                  │
    │                │──Acct-Start──→│                  │
    │                │  (session tracking begins)       │
```

**RADIUS attributes cho PPPoE:**
```
Access-Accept from RADIUS contains:
- Framed-IP-Address: 118.70.123.45 (or Framed-Pool: "pool-50mbps")
- Framed-IP-Netmask: 255.255.255.255 (/32 — point-to-point)
- Session-Timeout: 86400 (24h — force reconnect daily)
- Idle-Timeout: 1800 (30min idle → disconnect)
- Filter-Id: "50Mbps-download" (QoS policy)
- Cisco-AVPair: rate-limit input 51200000 (50Mbps)
- Class: "gold-subscriber" (accounting group)
```

---

## 5. MTU và MSS — Vấn đề "đường hẹp" của PPPoE

### Mini-example: Container qua cổng hẹp

Bình thường, container hàng hóa (packet) rộng **1500cm** vừa khít cổng kho (Ethernet MTU = 1500).

Nhưng với PPPoE, cổng bị **thu hẹp 8cm** (PPPoE header 8 bytes) → container tối đa **1492cm** mới lọt qua!

Nếu container 1500cm cố đi qua → BỊ KẸT! Phải:
- **Cắt nhỏ** container (Fragmentation) — chậm, tốn CPU
- Hoặc **đóng container nhỏ hơn từ đầu** (MSS Clamping) — efficient!

### MTU Problem chi tiết

```
Standard Ethernet:
┌─────────────────────────────────────────────┐
│ Ethernet │ IP Header │ TCP Header │ Data    │
│ Header   │ 20 bytes  │ 20 bytes   │ 1460B   │
│ 14 bytes │           │            │         │
└─────────────────────────────────────────────┘
│←──── Ethernet MTU = 1500 bytes ───────────→│
              │←─── IP MTU = 1500 ──────────→│
                          │←─ TCP MSS = 1460 ─│

PPPoE:
┌──────────────────────────────────────────────────────────┐
│ Ethernet │ PPPoE  │ PPP  │ IP Hdr │ TCP Hdr │ Data     │
│ Header   │ Header │ Hdr  │ 20B    │ 20B     │ 1452B    │
│ 14 bytes │ 6 bytes│ 2B   │        │         │          │
└──────────────────────────────────────────────────────────┘
│←──── Ethernet MTU = 1500 bytes ─────────────────────────→│
              │←── PPPoE Payload = 1492 bytes ────────────→│
                     │←── PPP Payload = 1492 bytes ──────→│
                          │←── IP MTU = 1492 ────────────→│
                                     │←─ TCP MSS = 1452 ──│

Overhead:
- PPPoE Header: 6 bytes
- PPP Protocol field: 2 bytes
- Total PPPoE overhead: 8 bytes
- Available for IP: 1500 - 8 = 1492 bytes
- TCP MSS: 1492 - 20(IP) - 20(TCP) = 1452 bytes
```

### Vấn đề khi không adjust MTU

```
Scenario: User truy cập website, server gửi packet 1500 bytes

User (PPPoE MTU=1492) ←── Internet ←── Server (MTU=1500)

1. Server gửi IP packet size 1500 bytes (TCP data = 1460)
2. Packet đến ISP router (trước PPPoE encapsulation)
3. Router cần encapsulate: 1500 + 8(PPPoE) = 1508 → VƯỢT Ethernet MTU!
4. Options:
   a. Fragment: Router chia packet thành 2 → inefficient, CPU-intensive
   b. Drop + ICMP: Gửi "Packet Too Big" (PMTUD) → Server giảm size
   c. Nếu ICMP bị firewall block → BLACK HOLE! Packet mất không dấu vết

Black Hole MTU Problem:
- Server gửi large packets
- Packets bị drop ở PPPoE boundary
- ICMP "Packet Too Big" bị firewall chặn
- Server không biết giảm size → retransmit → drop again → TIMEOUT!
- Symptom: Small pages load, large pages/downloads HANG

Websites bị ảnh hưởng:
- HTTPS (TLS record = large)
- Downloads
- Video streaming
- Sites có ICMP filtering
```

### Giải pháp: MSS Clamping

```
MSS Clamping = Router tự động giảm TCP MSS trong SYN packet

Không clamping:
Client SYN → MSS=1460 → Server SYN-ACK → MSS=1460
→ Server gửi packet 1500 bytes → KẸT tại PPPoE!

Có clamping:
Client SYN → MSS=1460 → [Router changes: MSS=1452] → Server
Server SYN-ACK → MSS=1452 → Server giờ chỉ gửi max 1452+40=1492
→ Vừa khít PPPoE MTU! Không bao giờ kẹt!

Configuration:
```

```cisco
! Cisco IOS — MSS Clamping trên dialer interface:
interface Dialer1
 ip tcp adjust-mss 1452
 ip mtu 1492

! Hoặc trên LAN interface (nếu router là PPPoE client):
interface GigabitEthernet0/0    ! LAN side
 ip tcp adjust-mss 1452

! Linux (iptables):
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --set-mss 1452

! Hoặc automatic (clamp to PMTU):
iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -j TCPMSS --clamp-mss-to-pmtu

! MikroTik:
/ip firewall mangle
add chain=forward protocol=tcp tcp-flags=syn action=change-mss \
  new-mss=1452 passthrough=yes

! pfSense/OPNsense:
# System → Advanced → Firewall
# MSS Clamping: Enable, set to 1452
```

### Baby Jumbo Frames

```
Giải pháp tốt hơn: Baby Jumbo Frames (RFC 4638)

Thay vì giảm MTU xuống 1492:
→ Tăng Ethernet MTU lên 1508 để chứa PPPoE + full 1500 IP packet!

Requirements:
- DSLAM/OLT hỗ trợ Baby Jumbo (> 1500 Ethernet)
- Tất cả switches trên path hỗ trợ
- Không phải ISP nào cũng enable

Configuration (nếu ISP hỗ trợ):
interface Dialer1
 ip mtu 1500          ! Full IP MTU!
 mtu 1508            ! Ethernet MTU (1500 + 8)

Thực tế Việt Nam:
- Hầu hết ISP VN dùng MTU 1492 (standard PPPoE)
- Một số ISP fiber mới hỗ trợ Baby Jumbo
- Kiểm tra: ping -M do -s 1472 8.8.8.8
  - Nếu reply → MTU = 1500 (Baby Jumbo)
  - Nếu fragmentation needed → MTU = 1492 (standard)
```

### Trong thực tế

**Debugging MTU issues:**
```bash
# Tìm MTU thực tế (Linux):
ping -M do -s 1464 8.8.8.8    # 1464 + 28(IP+ICMP) = 1492 → Should work
ping -M do -s 1465 8.8.8.8    # 1465 + 28 = 1493 → Should fail if MTU=1492

# Tìm MTU thực tế (Windows):
ping -f -l 1464 8.8.8.8       # -f = Don't Fragment

# tracepath (shows MTU per hop):
tracepath 8.8.8.8

# Kiểm tra MSS đang negotiate:
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0' -vv | grep mss
```

### Trong AWS

```
AWS MTU considerations:

VPC default MTU: 1500 (hoặc 9001 Jumbo cho supported instances)

Khi VPN tunnel chạy qua PPPoE Internet:
- Internet MTU (PPPoE): 1492
- IPSec overhead: ~50-70 bytes
- VPN tunnel MTU: 1492 - 70 = ~1422
- TCP MSS inside VPN: 1422 - 40 = ~1382

AWS Site-to-Site VPN:
- AWS recommends MTU 1399 for VPN tunnels
- Set MSS to 1359 on customer router
- AWS VPN automatically clamps MSS if needed

Direct Connect:
- DX supports MTU 1500 (default) hoặc 9001 (Jumbo Frame)
- Jumbo Frame: Phải enable trên VIF + VGW/TGW
- Jumbo only for Private/Transit VIF, NOT Public VIF
```

---

## 6. PPPoE trên DSL và Fiber — Ứng dụng thực tế tại ISP

### DSL (ADSL/VDSL) với PPPoE

```
┌────────────────────────────────────────────────────────────────┐
│                    DSL PPPoE Architecture                        │
│                                                                  │
│  Customer         ISP Infrastructure                             │
│  ┌──────┐  ┌──────┐  ┌───────┐  ┌──────┐  ┌──────┐           │
│  │  PC  │──│Modem │──│DSLAM  │──│Switch│──│ BRAS │──Internet  │
│  │      │  │(CPE) │  │       │  │  /   │  │      │           │
│  └──────┘  └──────┘  └───────┘  │Agg   │  └──────┘           │
│       │         │        │       └──────┘     │                │
│    Ethernet  DSL line  ATM/Ethernet   L2    PPPoE              │
│    (PPPoE   (PPPoA    (bridged)     Switch  termination        │
│     client  or bridge)              fabric                     │
│     or bridge)                                                  │
│                                                                  │
│  Modes:                                                          │
│  1. CPE = Bridge mode → PC chạy PPPoE client                   │
│  2. CPE = Router mode → Modem chạy PPPoE client (phổ biến)     │
│                                                                  │
│  ATM-based DSL (ADSL/ADSL2):                                   │
│  PC → Ethernet → Modem → PPPoA → ATM → DSLAM → BRAS          │
│  PC → Ethernet → Modem(bridge) → PPPoE → Ethernet → BRAS      │
│                                                                  │
│  Ethernet-based DSL (VDSL2/G.fast):                             │
│  PC → Ethernet → Modem → PPPoE → Ethernet → DSLAM → BRAS     │
└────────────────────────────────────────────────────────────────┘
```

### Fiber (GPON/XGS-PON) với PPPoE

```
┌────────────────────────────────────────────────────────────────┐
│                 Fiber (GPON) PPPoE Architecture                  │
│                                                                  │
│  Customer              ISP Infrastructure                        │
│  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐            │
│  │  PC  │──│ ONT  │──│Splitter│──│ OLT │──│ BNG  │──Internet │
│  │      │  │(ONU) │  │ 1:32  │  │      │  │(BRAS)│            │
│  └──────┘  └──────┘  └───────┘  └──────┘  └──────┘            │
│       │         │         │          │         │                │
│    Ethernet   Fiber     Passive    Aggregation PPPoE            │
│    (LAN)     (GPON)    Optical     Ethernet   termination      │
│                         Network                                  │
│                                                                  │
│  ONT Modes (tương tự DSL modem):                                │
│  1. Bridge mode: ONT chỉ bridge, router sau ONT chạy PPPoE    │
│  2. Router mode: ONT tự chạy PPPoE (ISP cấu hình sẵn)        │
│                                                                  │
│  Tại VN (2024):                                                 │
│  - FPT: GPON, ONT router mode, PPPoE tự động                  │
│  - Viettel: GPON, ONT router mode, PPPoE credentials embedded  │
│  - VNPT: GPON, một số bridge mode, user cần enter PPPoE info  │
│                                                                  │
└────────────────────────────────────────────────────────────────┘
```

### PPPoE vs IPoE — Xu hướng tương lai

```
┌──────────────────────────────────────────────────────────────┐
│                     PPPoE vs IPoE                              │
├──────────────────────┬───────────────────────────────────────┤
│ PPPoE                │ IPoE (IP over Ethernet)                │
├──────────────────────┼───────────────────────────────────────┤
│ Authentication: Yes  │ Authentication: DHCP Option 82,       │
│ (username/password)  │ 802.1X, hoặc line-id based            │
│                      │                                       │
│ Session tracking:    │ Session tracking: DHCP lease-based    │
│ PPP session          │                                       │
│                      │                                       │
│ MTU: 1492 (problem!) │ MTU: 1500 (no overhead!)             │
│                      │                                       │
│ Overhead: 8 bytes    │ Overhead: 0 (native Ethernet)        │
│ per packet           │                                       │
│                      │                                       │
│ Complexity: Moderate │ Complexity: Lower                     │
│                      │                                       │
│ ISP control: Strong  │ ISP control: Via DHCP policies       │
│ (per-session policy) │                                       │
│                      │                                       │
│ Use case: DSL, Fiber │ Use case: Modern Fiber, Data Center  │
│ (legacy, still 70%+) │ (growing, especially new deployments)│
│                      │                                       │
│ Trend: Declining     │ Trend: Increasing                    │
│ (but very slowly)    │ (especially in Asia)                 │
└──────────────────────┴───────────────────────────────────────┘

Tại sao ISP vẫn dùng PPPoE (2024):
1. Infrastructure hiện tại (BRAS, RADIUS) designed for PPPoE
2. Billing system tied to PPPoE sessions
3. Migration cost cao (millions of CPE devices)
4. Regulatory requirements (logging per session)
5. Familiarity — staff trained on PPPoE troubleshooting
```

### Trong thực tế

**Cấu hình PPPoE Client (Cisco router ở customer site):**
```cisco
! Interface vật lý kết nối modem
interface GigabitEthernet0/0
 no ip address
 pppoe enable
 pppoe-client dial-pool-number 1
 no shutdown

! Dialer interface (logical PPP)
interface Dialer1
 ip address negotiated          ! Nhận IP từ ISP (IPCP)
 ip mtu 1492                   ! PPPoE MTU
 ip tcp adjust-mss 1452        ! MSS Clamping
 encapsulation ppp
 dialer pool 1
 ppp authentication chap callin
 ppp chap hostname FPT_user01@fpt.vn
 ppp chap password 0 MyPassword123
 ip nat outside                 ! NAT outside interface

! Default route qua PPPoE
ip route 0.0.0.0 0.0.0.0 Dialer1

! NAT configuration
ip nat inside source list NAT_ACL interface Dialer1 overload
access-list 1 permit 192.168.1.0 0.0.0.255
```

**Cấu hình PPPoE Client (Linux — pppd):**
```bash
# /etc/ppp/peers/my-isp
plugin rp-pppoe.so
eth0
name "FPT_user01@fpt.vn"
usepeerdns
defaultroute
persist
maxfail 0
holdoff 5
lcp-echo-interval 20
lcp-echo-failure 3
mtu 1492
mru 1492

# /etc/ppp/chap-secrets
"FPT_user01@fpt.vn" * "MyPassword123"

# Connect:
sudo pon my-isp

# Disconnect:
sudo poff my-isp

# Status:
ip addr show ppp0
cat /var/log/syslog | grep pppd
```

---

## 7. BNG/BRAS — ISP Access Concentrator

### Mini-example: Trạm thu phí cao tốc

BRAS (Broadband Remote Access Server) giống **trạm thu phí** trên cao tốc:
- Mỗi xe (subscriber) phải qua trạm
- Trạm xác nhận vé (authentication)
- Ghi nhận xe vào (session start, accounting)
- Áp dụng tốc độ (rate limiting/QoS)
- Khi xe ra (session end, stop accounting)

### BRAS/BNG Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    BNG (Broadband Network Gateway)                │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Functions:                                                       │
│  1. PPPoE Termination (Discovery + Session)                      │
│  2. Authentication (RADIUS/TACACS+)                              │
│  3. Authorization (policies, bandwidth)                          │
│  4. Accounting (usage tracking, billing)                         │
│  5. IP Address Assignment (local pool or DHCP)                   │
│  6. QoS Enforcement (rate limiting per subscriber)               │
│  7. Policy Enforcement (web filtering, parental control)         │
│  8. NAT (if CGNAT deployed)                                     │
│  9. Lawful Intercept (legal requirement)                         │
│                                                                   │
│  Scale (typical):                                                 │
│  - 1 BNG handles 10,000 - 100,000 subscribers                   │
│  - Throughput: 100Gbps - 1Tbps                                  │
│  - Sessions: Tens of thousands concurrent                        │
│                                                                   │
│  Vendors:                                                         │
│  - Cisco ASR 9000 (IOS-XR)                                      │
│  - Juniper MX Series                                             │
│  - Huawei ME60                                                   │
│  - Nokia (Alcatel-Lucent) 7750 SR                               │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### RADIUS Integration

```
PPPoE + RADIUS = Standard ISP deployment

Authentication flow:
Subscriber → BRAS → RADIUS Server → User Database

RADIUS messages:
1. Access-Request: BRAS → RADIUS (username + CHAP response)
2. Access-Accept: RADIUS → BRAS (+ subscriber policies)
3. Accounting-Start: BRAS → RADIUS (session started)
4. Accounting-Interim: BRAS → RADIUS (periodic usage update)
5. Accounting-Stop: BRAS → RADIUS (session ended)

RADIUS attributes returned in Access-Accept:
- Framed-IP-Address / Framed-Pool
- Session-Timeout (max session duration)
- Idle-Timeout (disconnect if idle)
- Filter-Id (traffic policy name)
- Vendor-specific:
  - Download bandwidth limit
  - Upload bandwidth limit
  - Data cap (monthly quota)
  - VLAN assignment
  - DNS servers
```

### Trong thực tế — ISP Operations

```
Daily Operations on BRAS:

1. Monitor active sessions:
show pppoe session
! Total sessions, per VLAN, per interface

2. Disconnect problem user:
clear pppoe session username user123@isp.com
! Force disconnect (user's modem will reconnect)

3. Check subscriber details:
show subscriber session username user123@isp.com
! IP, session duration, bandwidth usage, policy applied

4. Change of Authorization (CoA):
! RADIUS pushes new policy without disconnecting:
! E.g., User upgrades package → CoA changes bandwidth
RADIUS CoA: Change filter from "50Mbps" to "100Mbps"
! User experiences instant speed upgrade without reconnect!

5. Disconnect-Message (DM/PoD):
! RADIUS tells BRAS to disconnect specific user:
RADIUS DM: Disconnect user "user123@isp.com"
! Used for: Account suspension, overdue payment, abuse
```

---

## 8. Tình huống thực tế

### Tình huống 1: Nhà dân — PPPoE qua Fiber

```
Bạn vừa đăng ký Internet FPT 100Mbps:

Setup:
ISP technician → Cắm fiber → ONT → Cáp Ethernet → WiFi Router

ONT (bridge mode): Chỉ chuyển quang thành điện
WiFi Router: Chạy PPPoE client

Trên WiFi router (TP-Link/Asus):
1. Vào 192.168.1.1 (router admin)
2. WAN Settings:
   - Connection Type: PPPoE
   - Username: user01fpt@fpt.vn (ISP cung cấp)
   - Password: ******** (ISP cung cấp)
   - MTU: 1492 (thường tự động)
3. Apply → Router kết nối → Internet hoạt động!

Troubleshooting tại nhà:
- Không kết nối được?
  1. Kiểm tra đèn PON trên ONT (xanh = có signal)
  2. Kiểm tra cáp Ethernet ONT → Router
  3. Kiểm tra username/password đúng
  4. Restart ONT trước, đợi 2 phút, restart router
  5. Gọi ISP hotline — có thể BRAS đang maintenance

- Trang web load chậm/không load?
  → MTU issue! Set MTU = 1492 trên router
  → Enable MSS Clamping nếu có option
  
- Bị disconnect mỗi 24h?
  → ISP set Session-Timeout = 86400s (24h)
  → Modem tự reconnect (normal behavior)
  → Nếu muốn stable: set "Reconnect Mode = Always On"
```

### Tình huống 2: Doanh nghiệp — PPPoE với Static IP

```
Công ty thuê leased line 200Mbps với static IP:

ISP cung cấp:
- PPPoE Username: corp_abc@business.isp.vn
- PPPoE Password: SecurePass!@#
- Static IP: 203.0.113.50/29 (block of 8)
- Gateway: 203.0.113.49
- DNS: 8.8.8.8, 8.8.4.4

Cisco Router Configuration:
interface GigabitEthernet0/0
 no ip address
 pppoe enable
 pppoe-client dial-pool-number 1

interface Dialer1
 ip address 203.0.113.50 255.255.255.248    ! Static IP!
 ! KHÔNG dùng "ip address negotiated" cho static
 encapsulation ppp
 dialer pool 1
 ppp authentication chap
 ppp chap hostname corp_abc@business.isp.vn
 ppp chap password 0 SecurePass!@#
 ip mtu 1492
 ip tcp adjust-mss 1452

! Còn lại /29 IPs dùng cho NAT hoặc servers
ip nat pool PUBLIC 203.0.113.51 203.0.113.54 prefix-length 29
ip nat inside source list NAT_ACL pool PUBLIC overload
```

### Tình huống 3: ISP — BRAS Configuration

```
ISP triển khai BRAS cho 50,000 subscribers:

Cisco ASR 9000 BNG Configuration:
! Domain map for authentication
aaa domain default
 aaa authentication ppp default group radius
 aaa authorization subscriber default group radius
 aaa accounting subscriber default group radius

! RADIUS servers
radius-server host 10.0.1.100 auth-port 1812 acct-port 1813
 key EncryptedRADIUSkey

! PPPoE configuration
interface Bundle-Ether1.100    ! Access interface
 encapsulation dot1q 100      ! Customer VLAN
 pppoe enable bba-group PPPoE_BBA_GROUP

bba-group pppoe PPPoE_BBA_GROUP
 sessions per-vlan limit 4000
 sessions per-mac limit 1

! Dynamic template (applied per subscriber)
dynamic-template type ppp DT_50MBPS
 ppp authentication chap
 ppp ipcp peer-address pool POOL_50MBPS
 ipv4 unnumbered Loopback100
 service-policy input PM_50MBPS_IN
 service-policy output PM_50MBPS_OUT

! IP Pool
pool vrf default ipv4 POOL_50MBPS
 address-range 100.64.0.1 100.64.255.254
 ! Using CGNAT range (100.64.0.0/10)
```

### Tình huống 4: PPPoE trên AWS hybrid

```
Scenario: On-premises có PPPoE Internet, cần VPN đến AWS

Problem: PPPoE MTU = 1492 → VPN MTU giảm thêm

Architecture:
Customer ── PPPoE ── ISP ── Internet ── AWS VPN Endpoint
(MTU 1492)

VPN Tunnel MTU calculation:
Ethernet MTU: 1500
PPPoE: -8 → 1492
IPSec ESP (Transport mode): -37-73 (depends on cipher)
IPSec ESP (Tunnel mode): -53-89

Conservative VPN MTU: 1492 - 89 = 1403
AWS recommended: 1399

Configuration:
! Customer router (PPPoE + VPN):
interface Dialer1
 ip mtu 1492
 ip tcp adjust-mss 1452

interface Tunnel1    ! VPN to AWS
 ip mtu 1399
 ip tcp adjust-mss 1359
 tunnel source Dialer1
 tunnel mode ipsec ipv4

! Hoặc: AWS VPN automatically handles this:
! AWS sets MSS clamping on VPN tunnels
! Customer only needs PPPoE MSS for direct traffic
```

---

## 9. Bài tập thực hành

### Bài tập 1: Cấu hình PPPoE Client trên Cisco

```cisco
! Lab topology: PC ── Router (PPPoE Client) ── Switch ── Router (PPPoE Server)

! Router PPPoE Client:
interface GigabitEthernet0/0
 no ip address
 no shutdown
 pppoe enable
 pppoe-client dial-pool-number 1

interface Dialer1
 ip address negotiated
 ip mtu 1492
 encapsulation ppp
 dialer pool 1
 ppp authentication chap callin
 ppp chap hostname testuser@lab
 ppp chap password testpass

ip route 0.0.0.0 0.0.0.0 Dialer1

! Verification:
show pppoe session          ! Session established?
show interface Dialer1      ! IP assigned? MTU correct?
show ip route               ! Default route via Dialer1?
ping 8.8.8.8 source Dialer1

! Questions:
! 1. PPPoE Session ID là gì?
! 2. IP được assign từ đâu?
! 3. Nếu đổi password → điều gì xảy ra?
```

### Bài tập 2: PPPoE Server Configuration (Lab)

```cisco
! Router PPPoE Server (simulating BRAS):

! Enable BBA Group
bba-group pppoe MYGROUP
 virtual-template 1

! Virtual Template (template cho dynamic interfaces)
interface Virtual-Template1
 ip unnumbered Loopback0
 peer default ip address pool PPPOE_POOL
 ppp authentication chap
 ppp chap hostname ISP_BRAS

! IP Pool
ip local pool PPPOE_POOL 100.64.1.1 100.64.1.254

! Interface facing customers
interface GigabitEthernet0/0
 no ip address
 pppoe enable group MYGROUP
 no shutdown

! Loopback for unnumbered
interface Loopback0
 ip address 10.0.0.1 255.255.255.255

! Local authentication (thay vì RADIUS)
username testuser@lab password testpass

! Verify:
show pppoe session
show users          ! Connected PPPoE users
show ip local pool  ! Pool usage
```

### Bài tập 3: MTU Troubleshooting

```bash
# Bài tập: Tìm MTU thực tế và fix MTU black hole

# Step 1: Tìm MTU bằng ping
# Linux:
for size in 1472 1464 1452 1400; do
  echo "Testing size $size..."
  ping -M do -c 1 -s $size 8.8.8.8 2>&1 | grep -E "bytes|Frag"
done

# Windows:
for /L %i in (1472,-8,1400) do ping -n 1 -f -l %i 8.8.8.8

# Step 2: Nếu MTU = 1492 (PPPoE), verify MSS
# Capture SYN packet:
sudo tcpdump -i ppp0 'tcp[tcpflags] & tcp-syn != 0' -c 5 -vv 2>&1 | grep mss

# Step 3: Fix MSS nếu cần
# Linux:
sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -o ppp0 -j TCPMSS --set-mss 1452
sudo iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
  -i ppp0 -j TCPMSS --set-mss 1452

# Step 4: Test website mà trước đó bị hang
curl -v https://problem-website.com
# Nếu load OK → MSS fix worked!
```

### Bài tập 4: Phân tích PPPoE Packet (Wireshark)

```
# Capture PPPoE traffic:
sudo tcpdump -i eth0 -w pppoe_capture.pcap ether proto 0x8863 or ether proto 0x8864

# Hoặc trong Wireshark:
# Display filter: pppoe || ppp

# Phân tích capture:
# 1. Tìm PADI packet:
#    - Dest MAC = Broadcast (ff:ff:ff:ff:ff:ff)
#    - EtherType = 0x8863
#    - Code = 0x09
#    - Tags: Service-Name, Host-Uniq

# 2. Tìm PADO packet:
#    - Dest MAC = Unicast (client MAC)
#    - Code = 0x07
#    - Tags: AC-Name, Service-Name, AC-Cookie

# 3. Tìm LCP packets:
#    - EtherType = 0x8864 (Session)
#    - PPP Protocol = 0xC021 (LCP)
#    - MRU option = 1492?

# 4. Tìm CHAP packets:
#    - PPP Protocol = 0xC223 (CHAP)
#    - Code: 1=Challenge, 2=Response, 3=Success, 4=Failure

# 5. Tìm IPCP packets:
#    - PPP Protocol = 0x8021 (IPCP)
#    - IP Address option: 0.0.0.0 → assigned IP

# Questions:
# - Session ID assigned là gì?
# - AC-Name của ISP là gì?
# - IP được assign là bao nhiêu?
# - MRU negotiated là bao nhiêu?
```

---

## 10. Tóm tắt và Tài liệu tham khảo

### Tóm tắt kiến thức

```
┌─────────────────────────────────────────────────────────────┐
│                    PPPoE SUMMARY                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ ▸ PPPoE = PPP encapsulated in Ethernet frames              │
│ ▸ Used by ISPs for subscriber authentication + accounting  │
│ ▸ RFC 2516 defines the protocol                            │
│                                                             │
│ Discovery Phase (EtherType 0x8863):                        │
│ ▸ PADI: Client broadcast "any AC available?"              │
│ ▸ PADO: AC offers service                                 │
│ ▸ PADR: Client requests specific AC                       │
│ ▸ PADS: AC confirms, assigns Session ID                   │
│ ▸ PADT: Either side terminates session                    │
│                                                             │
│ Session Phase (EtherType 0x8864):                          │
│ ▸ LCP: Negotiate MRU, Auth method, Magic Number           │
│ ▸ Authentication: PAP (plaintext) or CHAP (challenge)     │
│ ▸ IPCP: Assign IP address, DNS servers                    │
│                                                             │
│ MTU Impact:                                                 │
│ ▸ Ethernet MTU: 1500 bytes                                │
│ ▸ PPPoE overhead: 8 bytes (6 PPPoE + 2 PPP)              │
│ ▸ IP MTU with PPPoE: 1492 bytes                           │
│ ▸ TCP MSS: 1452 bytes (1492 - 40)                         │
│ ▸ FIX: MSS Clamping on router                            │
│                                                             │
│ Authentication:                                             │
│ ▸ PAP: Simple but INSECURE (plaintext password)           │
│ ▸ CHAP: Secure (hash-based, periodic re-auth)            │
│ ▸ ISP backend: RADIUS for AAA                            │
│                                                             │
│ Trend:                                                      │
│ ▸ PPPoE still dominant (70%+ ISP deployments)             │
│ ▸ IPoE growing (newer fiber deployments)                  │
│ ▸ Baby Jumbo Frames mitigating MTU issue                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Quick Troubleshooting Guide

```
PPPoE không kết nối:
1. Layer 1: Đèn trên modem/ONT OK? Cáp OK?
2. Discovery: Wireshark → PADI gửi? PADO nhận?
3. Auth: Username/password đúng? CHAP vs PAP match?
4. IPCP: IP assigned? DNS received?
5. Routing: Default route qua Dialer/ppp0?

Website bị hang (MTU issue):
1. Ping test: ping -M do -s 1464 → OK? -s 1465 → fail?
2. MSS Clamping: ip tcp adjust-mss 1452
3. Verify: tcpdump for MSS value in SYN

Session drops:
1. LCP Echo: Kiểm tra keepalive (lcp-echo-interval/failure)
2. Session Timeout: ISP force disconnect? (24h normal)
3. RADIUS: CoA/DM từ ISP? Account expired?
4. Physical: DSL sync drops? Fiber signal?
```

### Tài liệu tham khảo

1. **RFC 2516** — "A Method for Transmitting PPP Over Ethernet (PPPoE)"
   - https://www.rfc-editor.org/rfc/rfc2516

2. **RFC 1661** — "The Point-to-Point Protocol (PPP)" — PPP base specification
   - https://www.rfc-editor.org/rfc/rfc1661

3. **RFC 1994** — "PPP Challenge Handshake Authentication Protocol (CHAP)"
   - https://www.rfc-editor.org/rfc/rfc1994

4. **RFC 1332** — "The PPP Internet Protocol Control Protocol (IPCP)"
   - https://www.rfc-editor.org/rfc/rfc1332

5. **RFC 4638** — "Accommodating a Maximum Transit Unit/Maximum Receive Unit (MTU/MRU) Greater Than 1492 in PPPoE"
   - https://www.rfc-editor.org/rfc/rfc4638

6. **Cisco Documentation** — "Broadband Network Gateway Configuration Guide (ASR 9000)"
   - https://www.cisco.com/c/en/us/td/docs/routers/asr9000/software/bng/

7. **Cisco Documentation** — "Configuring PPP over Ethernet Client"
   - https://www.cisco.com/c/en/us/support/docs/asynchronous-connections/

8. **Cisco Documentation** — "Ethernet MTU and TCP MSS Adjustment Concept for PPPoE"
   - https://www.cisco.com/c/en/us/support/docs/ip/transmission-control-protocol-tcp/200932

---

*Bài viết tiếp theo: [MPLS Fundamentals](/2026/07/05/mpls-fundamentals) — Label Switching, LSP, LDP, VPN*
