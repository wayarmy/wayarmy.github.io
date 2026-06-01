---
layout: post
title: "HTTPS & TLS Handshake - Cipher Suites, Certificate Verify, TLS 1.2 vs 1.3 và mTLS"
date: 2026-06-01
categories: [networking]
tags: [https, tls, ssl, handshake, cipher-suites, certificates, mtls, encryption]
---

## 1. Giới thiệu — Cuộc hội thoại bí mật

Hãy tưởng tượng bạn muốn nói chuyện **bí mật** với ai đó trong quán café đông người:

**Không có TLS (HTTP):**
- Bạn và đối tác nói to → **mọi người nghe được** (packet sniffing)
- Ai đó có thể **giả mạo** giọng đối tác (man-in-the-middle)
- Ai đó có thể **sửa** lời bạn nói (data tampering)

**Có TLS (HTTPS):**
1. **Xác minh danh tính**: "Cho tôi xem CMND" → Kiểm tra thật/giả (Certificate)
2. **Thoả thuận mật mã**: "Nói chuyện bằng tiếng gì mà chỉ 2 ta hiểu?" (Cipher Suite)
3. **Trao đổi chìa khóa**: Dùng phép thuật để cả 2 có cùng chìa khóa mà **không ai nghe được** (Key Exchange)
4. **Nói chuyện bí mật**: Mọi lời nói đều **mã hóa** (Encrypted communication)

**TLS (Transport Layer Security)** đảm bảo 3 điều:
- **Confidentiality** — Chỉ 2 bên đọc được (encryption)
- **Integrity** — Không ai sửa được nội dung (MAC/AEAD)
- **Authentication** — Biết chắc đang nói với đúng người (certificates)

### HTTPS = HTTP + TLS

```
HTTP:   http://example.com  → Port 80, NO encryption
HTTPS:  https://example.com → Port 443, TLS encrypted

Protocol stack:
  ┌──────────────┐      ┌──────────────┐
  │     HTTP     │      │     HTTP     │
  ├──────────────┤      ├──────────────┤
  │     TCP      │      │     TLS      │ ← Encryption layer
  ├──────────────┤      ├──────────────┤
  │     IP       │      │     TCP      │
  └──────────────┘      ├──────────────┤
   Plaintext (HTTP)     │     IP       │
                        └──────────────┘
                         Encrypted (HTTPS)
```

### TLS Timeline

| Version | Year | Status | RFC |
|---|---|---|---|
| SSL 2.0 | 1995 | ❌ INSECURE — disabled | — |
| SSL 3.0 | 1996 | ❌ INSECURE — disabled (POODLE) | RFC 6101 |
| TLS 1.0 | 1999 | ❌ Deprecated (2020) | RFC 2246 |
| TLS 1.1 | 2006 | ❌ Deprecated (2020) | RFC 4346 |
| **TLS 1.2** | **2008** | ✅ Acceptable (configure carefully) | **RFC 5246** |
| **TLS 1.3** | **2018** | ✅ **RECOMMENDED** | **RFC 8446** |

---

## 2. Cipher Suites — "Ngôn ngữ bí mật" được thỏa thuận

### Phép so sánh — Chọn loại khóa

Khi 2 người muốn giao tiếp bí mật, phải thỏa thuận:
1. **Cách trao chìa khóa** — gặp trực tiếp hay qua hộp thư bí mật? (Key Exchange)
2. **Loại khóa** — khóa cơ hay khóa điện tử? (Authentication)
3. **Cách mã hóa** — dùng mật mã Caesar hay Enigma? (Encryption)
4. **Cách kiểm tra** — ký tên hay dấu vân tay? (Integrity/MAC)

### Cipher Suite Format

```
TLS 1.2 cipher suite format:
  TLS_[KeyExchange]_WITH_[Encryption]_[MAC]

Example:
  TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
  │    │      │       │    │    │    │
  │    │      │       │    │    │    └── MAC/PRF: SHA-384
  │    │      │       │    │    └─── Mode: GCM (Galois/Counter)
  │    │      │       │    └──── Key size: 256-bit
  │    │      │       └─── Cipher: AES
  │    │      └─── Authentication: RSA certificate
  │    └─── Key Exchange: ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)
  └── Protocol: TLS

TLS 1.3 cipher suite format (simplified):
  TLS_[Encryption]_[Hash]
  
Example:
  TLS_AES_256_GCM_SHA384
  TLS_CHACHA20_POLY1305_SHA256
  TLS_AES_128_GCM_SHA256

(Key exchange always ECDHE/DHE, authentication always via certificate)
```

### Cipher Suite Components

#### Key Exchange (trao đổi key)

| Algorithm | Forward Secrecy | Performance | Notes |
|---|---|---|---|
| RSA | ❌ No | Fast | ❌ Removed in TLS 1.3 |
| DHE | ✅ Yes | Slow | Diffie-Hellman Ephemeral |
| **ECDHE** | ✅ Yes | **Fast** | ✅ **Recommended** (P-256, X25519) |

**Forward Secrecy (FS):**
```
Without FS (RSA key exchange):
  - Server private key compromised LATER
  - Attacker recorded past traffic
  - Can decrypt ALL past sessions! (catastrophic!)

With FS (ECDHE):
  - Each session has unique ephemeral key
  - Server key compromised → past sessions STILL safe!
  - Only current session at risk
  
TLS 1.3: ALL cipher suites provide Forward Secrecy!
```

#### Encryption (mã hóa data)

| Algorithm | Key Size | Mode | Notes |
|---|---|---|---|
| AES | 128/256 bit | GCM | ✅ Standard (hardware accelerated) |
| ChaCha20 | 256 bit | Poly1305 | ✅ Fast on mobile (no AES hardware) |
| 3DES | 168 bit | CBC | ❌ Weak — disabled |
| RC4 | 128 bit | Stream | ❌ Broken — disabled |

```
AEAD (Authenticated Encryption with Associated Data):
  AES-GCM and ChaCha20-Poly1305 are AEAD ciphers:
  - Encrypt AND authenticate in one step
  - Detects tampering automatically
  - No separate MAC needed!
  
  TLS 1.3 ONLY allows AEAD ciphers!
```

#### Authentication (xác thực server)

| Algorithm | Key Size | Notes |
|---|---|---|
| RSA | 2048/4096 bit | Most common, well-understood |
| ECDSA | P-256/P-384 | Smaller keys, faster |
| Ed25519 | 256 bit | Modern, compact (limited support) |

### Recommended Cipher Suites (2026)

```
TLS 1.3 (all are strong — just pick order):
  TLS_AES_256_GCM_SHA384
  TLS_CHACHA20_POLY1305_SHA256
  TLS_AES_128_GCM_SHA256

TLS 1.2 (carefully selected):
  ECDHE-ECDSA-AES256-GCM-SHA384
  ECDHE-RSA-AES256-GCM-SHA384
  ECDHE-ECDSA-CHACHA20-POLY1305
  ECDHE-RSA-CHACHA20-POLY1305
  ECDHE-ECDSA-AES128-GCM-SHA256
  ECDHE-RSA-AES128-GCM-SHA256

AVOID:
  ✗ Anything with CBC mode (padding oracle attacks)
  ✗ Anything with RSA key exchange (no forward secrecy)
  ✗ Anything with SHA-1 MAC
  ✗ Anything with 3DES or RC4
```

---

## 3. TLS 1.2 Handshake — Chi tiết step-by-step

### Phép so sánh — Quy trình mở tài khoản ngân hàng

1. Bạn đến ngân hàng: "Tôi muốn mở tài khoản" (ClientHello)
2. Ngân hàng: "OK, đây là giấy tờ chúng tôi" (ServerHello + Certificate)
3. Bạn kiểm tra: "Giấy phép có thật không?" (Certificate Verification)
4. Cả hai thống nhất mật mã giao dịch (Key Exchange)
5. "Từ giờ giao dịch dùng mật mã này nhé!" (Finished)

### Full TLS 1.2 Handshake (2-RTT)

```
Client                                               Server
  │                                                     │
  │──── ClientHello ───────────────────────────────────→│  RTT 1
  │     • TLS version (1.2)                             │  start
  │     • Client Random (32 bytes)                      │
  │     • Session ID (resumption)                       │
  │     • Cipher Suites list                            │
  │     • Compression methods                           │
  │     • Extensions (SNI, ALPN, etc.)                  │
  │                                                     │
  │←─── ServerHello ──────────────────────────────────│
  │     • Selected TLS version                          │
  │     • Server Random (32 bytes)                      │
  │     • Session ID                                    │
  │     • Selected Cipher Suite                         │
  │     • Selected Compression                          │
  │                                                     │
  │←─── Certificate ─────────────────────────────────│
  │     • Server's X.509 certificate chain              │
  │                                                     │
  │←─── ServerKeyExchange (if ECDHE/DHE) ────────────│
  │     • ECDHE parameters (curve, public key)          │
  │     • Signature (proves server owns certificate)    │
  │                                                     │
  │←─── ServerHelloDone ─────────────────────────────│  RTT 1
  │                                                     │  end
  │                                                     │
  │──── ClientKeyExchange ────────────────────────────→│  RTT 2
  │     • Client's ECDHE public key                     │  start
  │                                                     │
  │──── ChangeCipherSpec ─────────────────────────────→│
  │     • "Switching to encrypted!"                     │
  │                                                     │
  │──── Finished (encrypted) ─────────────────────────→│
  │     • Hash of all handshake messages                │
  │     • Proves client has correct keys                │
  │                                                     │
  │←─── ChangeCipherSpec ────────────────────────────│
  │     • "Server also switching to encrypted!"         │
  │                                                     │
  │←─── Finished (encrypted) ───────────────────────│  RTT 2
  │     • Proves server has correct keys                │  end
  │                                                     │
  │════════ Application Data (encrypted) ══════════════│
  │                                                     │
  
Total: 2 Round Trips before data!
  TCP handshake: 1 RTT
  TLS handshake: 2 RTT
  First HTTP request: +1 RTT
  = 4 RTT total for first byte! (with new connection)
```

### Key Derivation (how session keys are created)

```
Both sides now have:
  - Client Random (32 bytes)
  - Server Random (32 bytes)
  - Pre-Master Secret (from ECDHE exchange)

Key derivation:
  Master Secret = PRF(Pre-Master Secret, "master secret",
                      Client Random + Server Random)

  Key Block = PRF(Master Secret, "key expansion",
                  Server Random + Client Random)
  
  From Key Block, extract:
    - Client write MAC key
    - Server write MAC key
    - Client write encryption key
    - Server write encryption key
    - Client write IV
    - Server write IV
```

---

## 4. TLS 1.3 Handshake — Faster and Simpler (1-RTT)

### What Changed from TLS 1.2?

```
REMOVED in TLS 1.3:
  ✗ RSA key exchange (no forward secrecy)
  ✗ Static DH (no forward secrecy)
  ✗ CBC mode ciphers (padding oracle)
  ✗ RC4, 3DES, MD5, SHA-1
  ✗ Compression (CRIME attack)
  ✗ Renegotiation
  ✗ ChangeCipherSpec message
  ✗ Custom DHE groups

ADDED/CHANGED:
  ✓ 1-RTT handshake (faster!)
  ✓ 0-RTT resumption (instant!)
  ✓ ALL cipher suites have forward secrecy
  ✓ Encrypted handshake (after ServerHello)
  ✓ Only AEAD ciphers (GCM, ChaCha20-Poly1305)
  ✓ Simplified — fewer options = fewer mistakes
```

### TLS 1.3 Full Handshake (1-RTT!)

```
Client                                               Server
  │                                                     │
  │──── ClientHello ───────────────────────────────────→│  RTT 1
  │     • Supported versions: TLS 1.3                   │  (only!)
  │     • Client Random                                 │
  │     • Cipher Suites (AEAD only)                    │
  │     • Key Share: ECDHE public key(s)               │
  │       (client GUESSES which curve server wants!)    │
  │     • Signature Algorithms                          │
  │     • Extensions (SNI, ALPN, PSK...)               │
  │                                                     │
  │←─── ServerHello ──────────────────────────────────│
  │     • Selected cipher suite                         │
  │     • Key Share: server ECDHE public key            │
  │                                                     │
  │     ═══ ENCRYPTED from here (handshake keys) ═══   │
  │                                                     │
  │←─── EncryptedExtensions ─────────────────────────│
  │←─── Certificate ─────────────────────────────────│
  │←─── CertificateVerify ──────────────────────────│
  │←─── Finished ────────────────────────────────────│  RTT 1
  │                                                     │  end!
  │                                                     │
  │──── Finished ─────────────────────────────────────→│
  │                                                     │
  │     ★ Client can send data WITH Finished! ★        │
  │──── Application Data ─────────────────────────────→│
  │                                                     │
  │════════ Application Data (encrypted) ══════════════│

Total: 1 Round Trip! (vs TLS 1.2's 2 RTT)
  TCP: 1 RTT
  TLS 1.3: 1 RTT
  HTTP request in same flight as Finished
  = 2 RTT total (vs 4 RTT with TLS 1.2 + TCP)
```

### TLS 1.3 0-RTT Resumption

```
After first connection, server provides session ticket.
Next connection:

Client                                               Server
  │                                                     │
  │──── ClientHello + Early Data ─────────────────────→│  
  │     • PSK identity (session ticket)                 │  0-RTT!
  │     • Key Share                                     │  Data sent
  │     • ★ 0-RTT Application Data (early data) ★      │  immediately!
  │                                                     │
  │←─── ServerHello + Finished ──────────────────────│
  │                                                     │
  │════════ Application Data ══════════════════════════│

0-RTT = Client sends data in FIRST packet!
  - Server can process request IMMEDIATELY
  - Ideal for: GET requests, API queries
  
⚠️ 0-RTT Security Warning:
  - Vulnerable to REPLAY ATTACKS
  - Only use for idempotent operations (GET, not POST)
  - Server must implement replay protection
```

---

## 5. Certificate Verification — Chuỗi tin cậy

### Phép so sánh — Xác minh CMND

Khi ai đó đưa CMND cho bạn:
1. **Kiểm tra tên**: Đúng người không? (hostname match)
2. **Kiểm tra hạn**: Còn hiệu lực không? (validity period)
3. **Kiểm tra cơ quan cấp**: Công an cấp thật không? (CA signature)
4. **Kiểm tra CMND bị thu hồi**: Có trong danh sách mất/hủy? (CRL/OCSP)

### Certificate Chain of Trust

```
Root CA (Trusted — embedded in OS/browser)
  │ signs
  ▼
Intermediate CA (issued by Root)
  │ signs
  ▼
Server Certificate (issued by Intermediate)
  │ presented to
  ▼
Client (browser)

Verification process:
1. Server presents: [Server Cert] + [Intermediate Cert]
2. Client has: [Root CA cert] (pre-installed in trust store)
3. Client verifies:
   a. Server cert signed by Intermediate? ✓
   b. Intermediate signed by Root? ✓
   c. Root in trust store? ✓
   d. Server cert hostname matches? ✓
   e. Not expired? ✓
   f. Not revoked? (OCSP/CRL check) ✓
   → TRUSTED! ✓
```

### Certificate Fields (X.509)

```
Certificate:
    Data:
        Version: 3
        Serial Number: 04:e3:...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=Let's Encrypt, CN=R3
        Validity:
            Not Before: Jul  5 2026
            Not After:  Oct  3 2026
        Subject: CN=www.example.com
        Subject Public Key Info:
            Algorithm: RSA (2048 bit)
            Public Key: 00:ab:cd:...
        Extensions:
            Subject Alternative Name:
                DNS: www.example.com
                DNS: example.com
            Basic Constraints: CA:FALSE
            Key Usage: Digital Signature, Key Encipherment
            Authority Key Identifier: ...
            CRL Distribution Points: http://...
            Authority Info Access:
                OCSP: http://ocsp.letsencrypt.org
    Signature Algorithm: sha256WithRSAEncryption
    Signature: 3a:4b:...
```

### OCSP Stapling — Faster Certificate Verification

```
Without OCSP Stapling:
  Client → Server: "Show me your cert"
  Server → Client: [Certificate]
  Client → OCSP Responder: "Is this cert revoked?" ← Extra RTT!
  OCSP Responder → Client: "No, it's valid"
  Client: OK, proceed

With OCSP Stapling:
  Server periodically fetches OCSP response and "staples" it to cert:
  Client → Server: "Show me your cert"
  Server → Client: [Certificate + OCSP response attached]
  Client: Certificate valid AND not revoked! ← No extra RTT!

nginx configuration:
  ssl_stapling on;
  ssl_stapling_verify on;
  resolver 8.8.8.8;
```

---

## 6. mTLS — Mutual TLS (Two-way Authentication)

### Phép so sánh — Cả 2 bên đưa CMND

**Regular TLS:**
- Chỉ SERVER chứng minh danh tính (Certificate)
- Client không cần chứng minh gì (anonymous)
- Ví dụ: Bạn vào web → web chứng minh "tôi là google.com"

**mTLS (Mutual TLS):**
- CẢ HAI bên chứng minh danh tính
- Client CŨNG phải có certificate!
- Ví dụ: Microservice A call Microservice B → cả 2 verify nhau

### mTLS Use Cases

```
1. Microservices communication (service mesh):
   Service A ←──mTLS──→ Service B
   Cả 2 verify identity → prevent unauthorized services

2. API authentication (thay vì API keys):
   Client app ←──mTLS──→ API server
   Client cert = proof of identity

3. IoT devices:
   IoT device ←──mTLS──→ Cloud backend
   Device cert = device identity

4. Zero Trust networks:
   Employee laptop ←──mTLS──→ Corporate resources
   Client cert = employee identity + device identity

5. Banking/Financial APIs:
   Bank client ←──mTLS──→ Payment gateway
   Both parties must be verified
```

### mTLS Handshake (Additional Steps)

```
Standard TLS:
  Server → Client: Certificate (server proves identity)
  
mTLS additions:
  Server → Client: CertificateRequest
    "I need YOUR certificate too!"
  Client → Server: Certificate (client proves identity)
  Client → Server: CertificateVerify (prove ownership of private key)
```

### mTLS Configuration — nginx

```nginx
server {
    listen 443 ssl;
    
    # Server certificate (normal)
    ssl_certificate /etc/ssl/server.pem;
    ssl_certificate_key /etc/ssl/server-key.pem;
    
    # Client certificate verification (mTLS)
    ssl_client_certificate /etc/ssl/ca.pem;    # CA that signed client certs
    ssl_verify_client on;                       # REQUIRE client cert!
    # ssl_verify_client optional;              # Or optional (check in app)
    ssl_verify_depth 2;                        # Chain depth
    
    # Pass client cert info to backend
    location / {
        proxy_pass http://backend;
        proxy_set_header X-Client-Cert-DN $ssl_client_s_dn;
        proxy_set_header X-Client-Cert-Verify $ssl_client_verify;
    }
}
```

---

## 7. TLS Configuration — Best Practices

### nginx SSL Configuration (2026)

```nginx
# /etc/nginx/conf.d/ssl.conf

ssl_protocols TLSv1.2 TLSv1.3;    # Only 1.2 and 1.3!

# TLS 1.3 cipher suites (separate config):
ssl_conf_command Ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256;

# TLS 1.2 cipher suites:
ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
ssl_prefer_server_ciphers off;     # Let client prefer (TLS 1.3 advice)

# ECDHE curves:
ssl_ecdh_curve X25519:secp384r1:secp256r1;

# Session tickets (0-RTT in TLS 1.3):
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets on;

# OCSP Stapling:
ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;

# HSTS:
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

# Certificate:
ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
```

### Testing TLS Configuration

```bash
# Check what server supports:
openssl s_client -connect example.com:443 -tls1_3
openssl s_client -connect example.com:443 -tls1_2

# Show certificate:
openssl s_client -connect example.com:443 -showcerts

# Test specific cipher:
openssl s_client -connect example.com:443 -cipher ECDHE-RSA-AES256-GCM-SHA384

# SSL Labs test (comprehensive):
# https://www.ssllabs.com/ssltest/

# testssl.sh (local):
./testssl.sh example.com

# nmap ssl scan:
nmap --script ssl-enum-ciphers -p 443 example.com
```

---

## 8. TLS Performance Optimization

### Session Resumption

```
TLS 1.2 Session Resumption:
  Option 1: Session ID
    - Server stores session state
    - Client sends Session ID in ClientHello
    - Server finds state → abbreviated handshake (1 RTT)
    - Downside: server memory for session storage

  Option 2: Session Tickets (RFC 5077)
    - Server encrypts session state → gives to client
    - Client sends ticket in next ClientHello
    - Server decrypts ticket → resume
    - Downside: single encryption key (rotation needed)

TLS 1.3 Resumption:
  - PSK (Pre-Shared Key) mechanism
  - Server sends NewSessionTicket after handshake
  - Client uses PSK for 0-RTT on next connection
  - Better security: ticket encrypted with regularly-rotated key
```

### TLS Performance Numbers

```
Handshake latency:
  TLS 1.2 (full):    2 RTT (~100ms on 50ms RTT connection)
  TLS 1.2 (resume):  1 RTT (~50ms)
  TLS 1.3 (full):    1 RTT (~50ms)
  TLS 1.3 (0-RTT):   0 RTT (~0ms additional latency!)

CPU cost (per handshake):
  RSA 2048 sign:     ~1ms
  ECDSA P-256 sign:  ~0.1ms
  ECDHE X25519:      ~0.05ms
  AES-256-GCM:       ~3 GB/s (hardware AES-NI)
  ChaCha20:          ~2 GB/s (no hardware needed)

Optimization tips:
  1. Use TLS 1.3 (1 RTT vs 2)
  2. Enable session tickets (0-RTT resumption)
  3. Use ECDSA certificates (10× faster signing than RSA)
  4. OCSP Stapling (avoid OCSP RTT)
  5. Use X25519 for key exchange (fastest ECDHE)
  6. Enable AES-NI on servers (hardware acceleration)
```

---

## 9. TLS Troubleshooting

### Common Issues

```bash
# Issue: "SSL certificate problem: unable to get local issuer certificate"
# → Missing intermediate certificate!
# Fix: Include full chain in ssl_certificate

# Issue: "SSL routines:ssl3_get_record:wrong version number"
# → Connecting to non-TLS port with TLS, or vice versa
# Fix: Check port (443 not 80)

# Issue: "certificate has expired"
# → Certificate validity period ended
# Fix: Renew certificate (certbot renew)

# Issue: "hostname mismatch"
# → Certificate CN/SAN doesn't match domain
# Fix: Issue certificate with correct domain names

# Debug TLS connection:
openssl s_client -connect example.com:443 -servername example.com -debug

# Check certificate dates:
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Check certificate chain:
openssl s_client -connect example.com:443 -showcerts 2>/dev/null | grep "Certificate chain" -A 20

# Wireshark filter for TLS:
tls.handshake.type == 1    # ClientHello
tls.handshake.type == 2    # ServerHello
tls.handshake.type == 11   # Certificate
```

---

## 10. Tổng kết

```
TLS/HTTPS Summary:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  TLS Provides:                                          │
│  • Confidentiality (encryption — AES/ChaCha20)         │
│  • Integrity (AEAD — GCM/Poly1305)                    │
│  • Authentication (certificates — X.509)               │
│                                                         │
│  TLS 1.2 vs TLS 1.3:                                   │
│  ┌──────────────────┬──────────────────────────┐       │
│  │ TLS 1.2          │ TLS 1.3                  │       │
│  ├──────────────────┼──────────────────────────┤       │
│  │ 2-RTT handshake  │ 1-RTT handshake          │       │
│  │ Many cipher opts │ Only AEAD + FS           │       │
│  │ Optional FS      │ Mandatory FS             │       │
│  │ Cleartext hello  │ Encrypted (after SH)     │       │
│  │ 0-RTT: no       │ 0-RTT: yes (PSK)         │       │
│  │ Complex config   │ Simpler, fewer choices   │       │
│  └──────────────────┴──────────────────────────┘       │
│                                                         │
│  Best Practices:                                        │
│  • Use TLS 1.3 (fallback to 1.2 if needed)            │
│  • ECDHE key exchange (forward secrecy)                │
│  • AES-256-GCM or ChaCha20-Poly1305                   │
│  • ECDSA certificates (faster than RSA)                │
│  • OCSP Stapling (reduce latency)                      │
│  • HSTS header (force HTTPS)                           │
│  • Session tickets (enable 0-RTT)                      │
│  • mTLS for service-to-service                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

*Tài liệu tham khảo:*
- RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3
- RFC 5246 — The Transport Layer Security (TLS) Protocol Version 1.2
- RFC 6066 — TLS Extensions (SNI, OCSP Stapling)
- RFC 8740 — Using TLS 1.3 with HTTP/2
- Mozilla SSL Configuration Generator — ssl-config.mozilla.org
- SSL Labs — ssllabs.com/ssltest
- Let's Encrypt Documentation — letsencrypt.org/docs
- Cloudflare — "An overview of TLS 1.3 and Q&A"
