---
layout: post
title: "VPN Technologies Deep Dive - IPSec (IKE Phase 1/2, ESP/AH), OpenVPN, WireGuard, L2TP & AWS VPN"
date: 2026-06-01
categories: [networking]
tags: [vpn, ipsec, openvpn, wireguard, tunneling, security]
---

# VPN Technologies Deep Dive — IPSec (IKE Phase 1/2, ESP/AH), OpenVPN, WireGuard, L2TP & AWS VPN

## Mục lục (Table of Contents)
1. [Giới thiệu bằng câu chuyện đời thường](#1-giới-thiệu-bằng-câu-chuyện-đời-thường)
2. [VPN Fundamentals — Nền tảng](#2-vpn-fundamentals--nền-tảng)
3. [IPSec — Bộ giao thức bảo mật IP](#3-ipsec--bộ-giao-thức-bảo-mật-ip)
4. [IKE Phase 1 và Phase 2](#4-ike-phase-1-và-phase-2)
5. [ESP và AH — Bảo vệ dữ liệu](#5-esp-và-ah--bảo-vệ-dữ-liệu)
6. [OpenVPN — Flexible SSL/TLS VPN](#6-openvpn--flexible-ssltls-vpn)
7. [WireGuard — VPN thế hệ mới](#7-wireguard--vpn-thế-hệ-mới)
8. [L2TP/IPSec và các protocol khác](#8-l2tpipsec-và-các-protocol-khác)
9. [AWS VPN — Site-to-Site và Client VPN](#9-aws-vpn--site-to-site-và-client-vpn)
10. [Tổng kết và So sánh](#10-tổng-kết-và-so-sánh)

---

## 1. Giới thiệu bằng câu chuyện đời thường

### VPN như đường hầm bí mật

Hãy tưởng tượng bạn sống ở Hà Nội, muốn gửi tài liệu mật tới chi nhánh Hồ Chí Minh. Trên đường đi, có kẻ xấu có thể chặn đọc, sửa tài liệu.

**Giải pháp**: Xây một **đường hầm ngầm** (tunnel) từ Hà Nội tới HCM. Tài liệu đi trong đường hầm — không ai nhìn thấy, không ai chặn được.

| Đời thường | VPN |
|---|---|
| Đường hầm ngầm | Encrypted tunnel |
| Cổng vào đường hầm (cần chìa khóa) | VPN gateway (cần credentials) |
| Tài liệu đựng trong hộp khóa | Encrypted packet |
| Không ai thấy nội dung | Confidentiality |
| Niêm phong tài liệu (biết nếu bị mở) | Integrity (hash/MAC) |
| Biết chắc tài liệu từ chi nhánh thật | Authentication |

### VPN Use Cases

| Use Case | VPN Type | Ví dụ |
|---|---|---|
| Nhân viên WFH truy cập công ty | **Remote Access VPN** | OpenVPN, WireGuard, AnyConnect |
| Kết nối 2 văn phòng (site-to-site) | **Site-to-Site VPN** | IPSec, AWS VPN |
| Bypass geo-restriction | **Consumer VPN** | NordVPN, ExpressVPN |
| Kết nối cloud với on-premises | **Hybrid Cloud VPN** | AWS VPN, Azure VPN Gateway |

---

## 2. VPN Fundamentals — Nền tảng

### 2.1 VPN cung cấp gì?

| Property | Ý nghĩa | Mechanism |
|---|---|---|
| **Confidentiality** (Bí mật) | Không ai đọc được nội dung | Encryption (AES, ChaCha20) |
| **Integrity** (Toàn vẹn) | Biết nếu dữ liệu bị sửa | HMAC, Poly1305 |
| **Authentication** (Xác thực) | Biết đối phương là ai | Certificates, PSK, keys |
| **Anti-replay** | Ngăn attacker replay old packets | Sequence numbers |

### 2.2 Tunneling — Đóng gói packet

```
Original IP Packet (trước VPN):
┌───────────────────────────────────────┐
│ IP Header │ TCP Header │   Data       │
│ Src: 10.0.0.5                        │
│ Dst: 10.1.0.100                      │
└───────────────────────────────────────┘

After VPN Encapsulation (trong tunnel):
┌────────────────────────────────────────────────────────────────────────┐
│ New IP Header │ VPN Header │ [Encrypted: Original IP │ TCP │ Data]    │
│ Src: 203.0.113.1 (VPN gateway A)                                      │
│ Dst: 198.51.100.1 (VPN gateway B)                                     │
└────────────────────────────────────────────────────────────────────────┘

Internet chỉ thấy: 203.0.113.1 → 198.51.100.1 (VPN gateways)
Không thấy: 10.0.0.5 → 10.1.0.100 (actual traffic) ← ENCRYPTED!
```

### 2.3 Transport Mode vs Tunnel Mode

| | Transport Mode | Tunnel Mode |
|---|---|---|
| Encrypts | Chỉ payload (data) | Toàn bộ original IP packet |
| New header | Không thêm new IP header | Thêm new outer IP header |
| Use case | Host-to-host | Network-to-network (site-to-site) |
| Original IP visible | ✅ Có | ❌ Không (encrypted) |

```
Transport Mode:
[Original IP Header] [ESP Header] [Encrypted Payload] [ESP Trailer]
  ↑ giữ nguyên

Tunnel Mode:
[New IP Header] [ESP Header] [Encrypted: Original IP + Payload] [ESP Trailer]
  ↑ mới                       ↑ toàn bộ original packet encrypted
```

---

## 3. IPSec — Bộ giao thức bảo mật IP

### 3.1 IPSec là gì?

**IPSec (Internet Protocol Security)** là **suite of protocols** (bộ giao thức) bảo mật communications ở Layer 3 (Network Layer). Nó là standard cho site-to-site VPN và được built-in vào mọi OS và network device.

**Components chính:**
- **IKE (Internet Key Exchange)**: Đàm phán parameters + trao đổi keys
- **ESP (Encapsulating Security Payload)**: Encryption + Authentication
- **AH (Authentication Header)**: Authentication only (ít dùng)
- **SA (Security Association)**: "Hợp đồng" giữa 2 bên (algorithms, keys, lifetime)

### 3.2 IPSec Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     IPSec VPN Tunnel                          │
│                                                               │
│  ┌──────────┐    ┌───────────────────────┐    ┌──────────┐ │
│  │ Site A   │    │     Internet          │    │ Site B   │ │
│  │ 10.0.0/24│    │                       │    │10.1.0/24 │ │
│  │          │    │                       │    │          │ │
│  │  ┌────┐ │    │  IPSec Tunnel         │    │ ┌────┐  │ │
│  │  │ PC │─┼────┼═══════════════════════─┼────┼─│ PC │  │ │
│  │  └────┘ │    │  (encrypted)          │    │ └────┘  │ │
│  │     ↕   │    │                       │    │    ↕    │ │
│  │  ┌────┐ │    │                       │    │ ┌────┐  │ │
│  │  │ GW │─┼────┼───────────────────────┼────┼─│ GW │  │ │
│  │  │ A  │ │    │  Public IPs           │    │ │ B  │  │ │
│  │  └────┘ │    │  203.0.113.1 ←→       │    │ └────┘  │ │
│  └──────────┘    │       198.51.100.1    │    └──────────┘ │
│                  └───────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

### 3.3 Security Association (SA)

**SA** là "hợp đồng" chứa mọi thông tin cần cho IPSec tunnel:

```
SA Contents:
├── SPI (Security Parameters Index): ID duy nhất
├── Destination IP: 198.51.100.1
├── Protocol: ESP
├── Encryption Algorithm: AES-256-GCM
├── Authentication Algorithm: HMAC-SHA-256
├── Encryption Key: [256-bit key]
├── Authentication Key: [256-bit key]
├── Lifetime: 3600 seconds (hoặc 100MB data)
├── Mode: Tunnel
└── Anti-replay window: 64 packets

Note: SA là UNIDIRECTIONAL (1 chiều)
→ Cần 2 SAs cho bidirectional: 1 inbound + 1 outbound
```

---

## 4. IKE Phase 1 và Phase 2

### 4.1 IKE Overview

**IKE (Internet Key Exchange)** — RFC 7296 (IKEv2) — là protocol đàm phán SA. Gồm 2 phases:

```
Phase 1 (IKE SA): Thiết lập secure channel giữa 2 VPN gateways
  → Xác thực lẫn nhau
  → Thỏa thuận encryption/hash algorithms
  → Trao đổi keys (DH)
  → Kết quả: IKE SA (kênh bảo mật cho Phase 2)

Phase 2 (IPSec SA): Thỏa thuận tunnel parameters qua kênh Phase 1
  → Thỏa thuận IPSec algorithms (ESP/AH)
  → Thỏa thuận traffic selectors (subnet nào qua tunnel)
  → Optional: PFS (new DH exchange)
  → Kết quả: IPSec SA (tunnel data thực tế)
```

### 4.2 IKE Phase 1 — Main Mode (6 messages)

```
Initiator (A)                              Responder (B)
     │                                          │
     │── Msg 1: SA proposals ──────────────────→│  "Tôi hỗ trợ AES-256, SHA-256, DH group 14"
     │←── Msg 2: SA chosen ────────────────────│  "OK, dùng AES-256, SHA-256, DH group 14"
     │                                          │
     │── Msg 3: DH public value + nonce ──────→│  Key exchange
     │←── Msg 4: DH public value + nonce ──────│  
     │     [Both compute shared secret K]       │
     │                                          │
     │══ Messages 5,6: ENCRYPTED (using K) ═══│  
     │── Msg 5: Identity + Auth (cert/PSK) ───→│  Xác thực
     │←── Msg 6: Identity + Auth (cert/PSK) ───│  
     │                                          │
     │     [IKE SA Established!]                │
```

**Phase 1 proposals:**
```
IKE SA proposal:
├── Encryption: AES-256-CBC / AES-128-GCM / 3DES
├── Hash/PRF: SHA-256 / SHA-384 / SHA-512
├── DH Group: 14 (2048-bit) / 19 (ECP-256) / 20 (ECP-384)
├── Authentication: PSK / RSA / ECDSA
└── Lifetime: 86400 seconds (24 hours)
```

### 4.3 IKE Phase 2 — Quick Mode (3 messages)

```
(All messages encrypted + authenticated by Phase 1 IKE SA)

Initiator (A)                              Responder (B)
     │                                          │
     │── Msg 1: IPSec SA proposal + ────────→  │  
     │   Traffic selectors + optional DH       │  "Encrypt 10.0.0.0/24 ↔ 10.1.0.0/24"
     │                                          │  "Use AES-256-GCM, lifetime 1h"
     │←── Msg 2: IPSec SA chosen + ──────────  │
     │   Traffic selectors + optional DH       │
     │                                          │
     │── Msg 3: Confirmation ──────────────→   │
     │                                          │
     │     [IPSec SA Established!]              │
     │     [Tunnel active — data flows]         │
```

**Phase 2 proposals:**
```
IPSec SA proposal:
├── Protocol: ESP (preferred) or AH
├── Encryption: AES-256-GCM / AES-128-CBC / ChaCha20-Poly1305
├── Authentication: HMAC-SHA-256 (or implicit with GCM)
├── Mode: Tunnel
├── PFS: DH Group 14 (optional, recommended)
├── Lifetime: 3600 seconds + 100MB (whatever hits first)
└── Traffic selectors: 10.0.0.0/24 ↔ 10.1.0.0/24
```

### 4.4 IKEv2 vs IKEv1

| | IKEv1 | IKEv2 (RFC 7296) |
|---|---|---|
| Messages (Phase 1) | 6 (Main Mode) or 3 (Aggressive) | 4 (IKE_SA_INIT + IKE_AUTH) |
| NAT Traversal | Extension | Built-in |
| MOBIKE | No | Yes (roaming support) |
| Reliability | No built-in | Built-in retransmission |
| EAP support | Limited | Full |
| Simplicity | Complex | Simpler |
| Status | Legacy | **Recommended** |

---

## 5. ESP và AH — Bảo vệ dữ liệu

### 5.1 ESP (Encapsulating Security Payload) — RFC 4303

**ESP** cung cấp **cả encryption + authentication** — protocol chính dùng trong IPSec.

```
ESP Tunnel Mode Packet:
┌──────────┬───────────┬─────────────────────────────────────────────┬───────────┬────────┐
│ New IP   │ ESP       │         Encrypted                            │ ESP       │ ESP    │
│ Header   │ Header    │ [Orig IP Header | TCP/UDP | Data | Padding] │ Trailer   │ Auth   │
└──────────┴───────────┴─────────────────────────────────────────────┴───────────┴────────┘
                        ←──────────── Encrypted ────────────────────→
             ←──────────────────── Authenticated ────────────────────────────────→
```

**ESP Header:**
```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               Security Parameters Index (SPI)                 |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Sequence Number                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Payload Data (encrypted)                    |
|                         ...                                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Authentication Data (ICV)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 5.2 AH (Authentication Header) — RFC 4302

**AH** chỉ cung cấp **authentication** (không encryption). Ít dùng vì:
- Không mã hóa (chỉ verify integrity)
- Không qua NAT được (vì AH ký cả IP header, NAT thay đổi IP → signature invalid)

```
AH Packet:
┌──────────┬──────────┬──────────────────────────┐
│ IP Header│ AH Header│ Original Data (NOT encrypted!) │
└──────────┴──────────┴──────────────────────────┘
←────────── Authenticated (signed) ──────────────→
```

### 5.3 ESP vs AH

| | ESP | AH |
|---|---|---|
| Encryption | ✅ Yes | ❌ No |
| Authentication | ✅ Yes | ✅ Yes |
| Anti-replay | ✅ Yes | ✅ Yes |
| NAT compatible | ✅ Yes (NAT-T, UDP 4500) | ❌ No |
| Protocol number | 50 | 51 |
| Usage | **99%+ của IPSec** | Hiếm (legacy) |
| IP header protected | ❌ (mutable fields) | ✅ (covers IP header) |

### 5.4 NAT Traversal (NAT-T)

```
Problem: ESP packets have no port numbers → NAT can't track them
Solution: Wrap ESP inside UDP (port 4500)

Without NAT-T:
[IP Header (proto=50)] [ESP Header] [Encrypted Data]
→ NAT cannot process (no ports!)

With NAT-T:
[IP Header (proto=17)] [UDP 4500] [ESP Header] [Encrypted Data]
→ NAT sees UDP:4500 → can track and translate ✅
```

---

## 6. OpenVPN — Flexible SSL/TLS VPN

### 6.1 OpenVPN Overview

**OpenVPN** là VPN solution dựa trên **SSL/TLS** (OpenSSL library):
- Open-source, chạy trên mọi platform
- Chạy trên **user-space** (không cần kernel module)
- Dùng **TLS** cho key exchange + control channel
- Dùng **custom protocol** cho data channel
- Có thể chạy qua **TCP hoặc UDP**

### 6.2 Architecture

```
┌─────────────────────────────────────────────────┐
│ OpenVPN Architecture                             │
│                                                   │
│  Control Channel (TLS):                          │
│  ├── Authentication (certificates, username/pwd) │
│  ├── Key exchange (TLS handshake)               │
│  ├── Keepalive, session management              │
│  └── Push options (routes, DNS, etc.)           │
│                                                   │
│  Data Channel (encrypted):                       │
│  ├── Encryption: AES-256-GCM (default)          │
│  ├── Authentication: HMAC-SHA-256 (or AEAD)     │
│  ├── Encapsulation: UDP (preferred) or TCP      │
│  └── Compression: LZO/LZ4 (optional)           │
│                                                   │
│  Transport: UDP:1194 (default) or TCP:443       │
└─────────────────────────────────────────────────┘
```

### 6.3 OpenVPN Modes

| | TUN (Tunnel, L3) | TAP (Bridge, L2) |
|---|---|---|
| Layer | Layer 3 (IP packets) | Layer 2 (Ethernet frames) |
| Interface | Virtual TUN device | Virtual TAP device |
| Use case | Most VPNs (routing) | Bridge networks (same subnet) |
| Broadcasts | ❌ Not forwarded | ✅ Forwarded |
| Performance | Better (less overhead) | More overhead |
| Client IP | Different subnet | Same subnet as server |

### 6.4 OpenVPN Configuration (Server)

```bash
# /etc/openvpn/server.conf
port 1194
proto udp
dev tun

# Certificates
ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh2048.pem
tls-auth /etc/openvpn/ta.key 0  # HMAC firewall

# Network
server 10.8.0.0 255.255.255.0     # VPN subnet
push "route 10.0.0.0 255.255.0.0" # Push internal routes to clients
push "dhcp-option DNS 10.0.0.1"   # Push DNS

# Security
cipher AES-256-GCM
auth SHA256
tls-version-min 1.2
tls-cipher TLS-ECDHE-RSA-WITH-AES-256-GCM-SHA384

# Performance
keepalive 10 120
comp-lzo no           # Disable compression (security risk: VORACLE)
max-clients 100
persist-key
persist-tun
```

### 6.5 Advantages & Disadvantages

| ✅ Advantages | ❌ Disadvantages |
|---|---|
| Runs over TCP 443 (bypass firewalls) | Slower than WireGuard (user-space) |
| Extremely flexible configuration | Complex configuration |
| Works on all platforms | ~70,000 lines of code (audit difficulty) |
| Mature, battle-tested (20+ years) | Higher CPU usage |
| Can use any TLS cipher | Connection establishment slower |
| Certificate-based auth | Needs third-party client software |

---

## 7. WireGuard — VPN thế hệ mới

### 7.1 WireGuard Philosophy

**WireGuard** được thiết kế với triết lý **"simple and fast"**:
- **~4,000 lines of code** (vs. OpenVPN ~70,000, IPSec ~400,000)
- **Kernel-space** (nhanh hơn user-space)
- **Modern cryptography only** (không có legacy algorithms)
- **Cryptokey Routing**: Public key = identity = IP address

### 7.2 Cryptographic Choices (Fixed, No Negotiation)

| Component | Algorithm | Notes |
|---|---|---|
| Symmetric Encryption | ChaCha20 | Stream cipher, fast on mobile |
| Authentication | Poly1305 | AEAD with ChaCha20 |
| Key Exchange | Curve25519 (ECDH) | 256-bit elliptic curve |
| Hashing | BLAKE2s | Faster than SHA-256 |
| Key Derivation | HKDF | RFC 5869 |
| Hash Table | SipHash24 | DoS protection |

**Tại sao "no negotiation"?**
- IPSec/OpenVPN: client và server đàm phán algorithms → complexity → bugs
- WireGuard: MỘT bộ algorithms cố định → đơn giản → ít lỗi
- Nếu algorithm bị break → tạo WireGuard v2 (thay toàn bộ)

### 7.3 Cryptokey Routing

```
Concept: Mỗi peer = public key = allowed IPs

Configuration:
[Interface]
PrivateKey = (server private key)
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]  # Client Alice
PublicKey = ALICE_PUBLIC_KEY...
AllowedIPs = 10.0.0.2/32     # Alice gets 10.0.0.2

[Peer]  # Client Bob  
PublicKey = BOB_PUBLIC_KEY...
AllowedIPs = 10.0.0.3/32     # Bob gets 10.0.0.3

[Peer]  # Site B (entire subnet)
PublicKey = SITE_B_KEY...
AllowedIPs = 10.1.0.0/24     # Route entire subnet to this peer
Endpoint = 198.51.100.1:51820

Routing logic:
- Outgoing packet to 10.0.0.2 → encrypt with Alice's public key
- Incoming packet from Alice's key → must come from 10.0.0.2 (else drop)
```

### 7.4 WireGuard Handshake (1-RTT)

```
Initiator                           Responder
    │                                    │
    │── Initiation (ephemeral pub key, ──→│  1 message
    │   encrypted static key,            │
    │   timestamp)                        │
    │                                    │
    │←── Response (ephemeral pub key, ───│  1 message
    │    encrypted empty)                 │
    │                                    │
    │    [Session keys derived]           │
    │    [Data flows immediately]         │
    │                                    │
    Total: 1 RTT (vs. IKE: 4+ messages, OpenVPN: TLS handshake)
```

### 7.5 WireGuard Configuration

```bash
# Server: /etc/wireguard/wg0.conf
[Interface]
PrivateKey = SERVER_PRIVATE_KEY
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.0.0.2/32

# Client: /etc/wireguard/wg0.conf
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = vpn.example.com:51820
AllowedIPs = 0.0.0.0/0     # Route ALL traffic through VPN
PersistentKeepalive = 25    # NAT keepalive
```

```bash
# Commands
wg-quick up wg0        # Start VPN
wg-quick down wg0      # Stop VPN
wg show                # Show status
wg genkey | tee privatekey | wg pubkey > publickey  # Generate keys
```

---

## 8. L2TP/IPSec và các protocol khác

### 8.1 L2TP/IPSec

**L2TP (Layer 2 Tunneling Protocol)** + IPSec:
- L2TP: Tunneling protocol (no encryption)
- IPSec: Provides encryption/auth
- Combined: L2TP encapsulated inside IPSec

```
L2TP/IPSec Packet:
[IP] [UDP:500] [ESP] [UDP:1701] [L2TP] [PPP] [Original IP] [Data]
      ↑               ↑                  ↑
   IKE/IPSec       IPSec encrypted     L2TP tunnel
```

**Status**: Legacy — sử dụng khi OS có built-in support, nhưng WireGuard/IKEv2 tốt hơn.

### 8.2 Quick Comparison Table

| Protocol | Speed | Security | Ports | Complexity | Best For |
|---|---|---|---|---|---|
| **WireGuard** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | UDP 51820 | Low | Modern, general purpose |
| **IKEv2/IPSec** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | UDP 500,4500 | High | Mobile, site-to-site |
| **OpenVPN UDP** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | UDP 1194 | Medium | Bypass firewalls |
| **OpenVPN TCP** | ⭐⭐ | ⭐⭐⭐⭐⭐ | TCP 443 | Medium | Restrictive networks |
| **L2TP/IPSec** | ⭐⭐⭐ | ⭐⭐⭐⭐ | UDP 500,1701,4500 | Medium | Built-in OS support |
| **PPTP** | ⭐⭐⭐⭐ | ⭐ (BROKEN) | TCP 1723 | Low | ❌ DO NOT USE |

---

## 9. AWS VPN — Site-to-Site và Client VPN

### 9.1 AWS Site-to-Site VPN

```
┌──────────────────┐              ┌──────────────────────────────┐
│  On-Premises DC  │              │         AWS VPC              │
│                  │              │                              │
│  ┌────────────┐ │   IPSec      │  ┌───────────────────────┐  │
│  │ Customer   │ │   Tunnel 1   │  │ Virtual Private       │  │
│  │ Gateway    │═╪══════════════╪══│ Gateway (VGW)          │  │
│  │ (router)   │ │   Tunnel 2   │  │ or Transit Gateway    │  │
│  └────────────┘ │  (redundancy) │  └───────────────────────┘  │
│                  │              │                              │
│  10.0.0.0/16    │              │  172.16.0.0/16              │
└──────────────────┘              └──────────────────────────────┘
```

**Key Features:**
- 2 tunnels (redundancy, each to different AZ)
- IKEv2 với AES-256-GCM
- BGP hoặc Static routing
- Max bandwidth: 1.25 Gbps per tunnel
- Cost: ~$0.05/hour per VPN connection

### 9.2 AWS Client VPN

```
[Remote Workers] ──OpenVPN──→ [AWS Client VPN Endpoint] ──→ [VPC Resources]
                              (managed by AWS)
```

- Based on OpenVPN
- Managed service (no server to maintain)
- Supports: Mutual auth (certs), AD auth, SAML
- Scales automatically
- Split-tunnel or full-tunnel

### 9.3 AWS Transit Gateway + VPN

```
                    ┌──────────────────────┐
                    │   Transit Gateway    │
                    │                      │
         VPN       │  ┌──┐  ┌──┐  ┌──┐  │
Office A ══════════╪══│  │  │  │  │  │  │
                    │  │RT│  │RT│  │RT│  │  (Route Tables)
Office B ══════════╪══│  │  │  │  │  │  │
                    │  └──┘  └──┘  └──┘  │
Office C ══════════╪══════════════════════│
                    │      ↕    ↕    ↕    │
                    │   VPC A  VPC B VPC C │
                    └──────────────────────┘

→ Hub-and-spoke: All offices connected through TGW
→ ECMP: Up to 50 Gbps with multiple VPN connections
```

---

## 10. Tổng kết và So sánh

### 10.1 Decision Matrix

| Scenario | Recommended | Why |
|---|---|---|
| Site-to-site (enterprise) | **IPSec IKEv2** | Universal support, hardware acceleration |
| Remote access (modern) | **WireGuard** | Fast, simple, low battery usage |
| Remote access (compatibility) | **OpenVPN** | Works everywhere, TCP 443 bypass |
| Mobile VPN (roaming) | **IKEv2** | MOBIKE handles network switches |
| Restrictive network/firewall | **OpenVPN TCP:443** | Looks like HTTPS traffic |
| AWS hybrid cloud | **AWS Site-to-Site VPN** | Managed, dual-tunnel, BGP |
| Maximum performance | **WireGuard** | Kernel-space, modern crypto |

### 10.2 Performance Comparison

```
Typical throughput (single core, modern hardware):
WireGuard:       800-1000 Mbps
IPSec (AES-NI):  600-800 Mbps
OpenVPN UDP:     300-500 Mbps
OpenVPN TCP:     200-400 Mbps

Handshake latency:
WireGuard:   1 RTT (~100ms)
IKEv2:       2 RTT (~200ms)
OpenVPN:     4-6 RTT (~500ms+)
```

### 10.3 Tài liệu tham khảo

| Tài liệu | Nội dung |
|---|---|
| RFC 7296 | IKEv2 (Internet Key Exchange Protocol Version 2) |
| RFC 4303 | ESP (Encapsulating Security Payload) |
| RFC 4302 | AH (Authentication Header) |
| RFC 7539 | ChaCha20-Poly1305 (WireGuard uses) |
| wireguard.com/papers | WireGuard Whitepaper |
| OpenVPN docs | openvpn.net documentation |
| AWS VPN docs | AWS Site-to-Site VPN User Guide |

### 10.4 Câu hỏi ôn tập

1. VPN cung cấp những security properties nào?
2. Transport mode vs Tunnel mode: khác nhau thế nào? Khi nào dùng cái nào?
3. IKE Phase 1 vs Phase 2: mục đích mỗi phase? Kết quả?
4. ESP vs AH: tại sao ESP được dùng 99%+?
5. NAT-Traversal giải quyết vấn đề gì? Cơ chế?
6. WireGuard khác OpenVPN/IPSec gì ở triết lý thiết kế?
7. "Cryptokey Routing" trong WireGuard hoạt động thế nào?
8. OpenVPN TUN vs TAP: khác nhau? Khi nào dùng TAP?
9. AWS Site-to-Site VPN tại sao có 2 tunnels?
10. So sánh WireGuard vs IKEv2 vs OpenVPN cho mobile VPN.

---

*Bài viết được tham khảo từ RFC 7296 (IKEv2), RFC 4303 (ESP), RFC 4302 (AH), WireGuard whitepaper (wireguard.com), OpenVPN documentation, và AWS VPN User Guide.*
