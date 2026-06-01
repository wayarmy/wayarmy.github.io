---
layout: post
title: "TLS/SSL Deep Dive - Mã hóa truyền tải chuyên sâu"
date: 2026-06-01
categories: [security]
tags: [tls, ssl, certificates, cipher-suites, mtls, handshake]
---

## Mục lục
1. [Góc nhìn tổng quan - Phong bì niêm phong](#overview)
2. [TLS Handshake - Bắt tay bí mật](#handshake)
3. [Cipher Suites - Bộ thuật toán](#cipher-suites)
4. [Certificate Chain - Chuỗi chứng nhận](#cert-chain)
5. [OCSP và Certificate Revocation](#ocsp)
6. [Certificate Pinning](#pinning)
7. [mTLS - Xác thực hai chiều](#mtls)
8. [TLS 1.3 - Cải tiến lớn](#tls13)
9. [ALPN - Thương lượng giao thức](#alpn)
10. [Tổng kết và best practices](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

TLS giống **gửi thư bảo đảm qua công chứng**:

- **Handshake** = 2 bên thống nhất: "dùng phong bì loại gì, niêm phong kiểu gì, ai là công chứng viên"
- **Certificate** = CMND có công chứng (CA ký xác nhận "đây đúng là server X")
- **Cipher suite** = bộ công cụ mã hóa (thuật toán trao đổi khóa + mã hóa dữ liệu + xác thực)
- **mTLS** = CẢ HAI bên đều phải xuất trình CMND (không chỉ server)
- **OCSP** = gọi hỏi công an "CMND này còn hiệu lực không?" (kiểm tra revocation)

---

## 2. TLS Handshake {#handshake}

### TLS 1.2 Handshake (2 round-trips)

```
Client                              Server
  │                                    │
  │── ClientHello ────────────────────▶│ (supported ciphers, random)
  │                                    │
  │◀── ServerHello ───────────────────│ (chosen cipher, random)
  │◀── Certificate ───────────────────│ (server cert + chain)
  │◀── ServerKeyExchange ─────────────│ (DH params)
  │◀── ServerHelloDone ───────────────│
  │                                    │
  │── ClientKeyExchange ──────────────▶│ (DH public key)
  │── ChangeCipherSpec ───────────────▶│ (switching to encrypted)
  │── Finished ───────────────────────▶│ (encrypted verify)
  │                                    │
  │◀── ChangeCipherSpec ──────────────│
  │◀── Finished ──────────────────────│
  │                                    │
  │════ Encrypted Application Data ════│
```

### TLS 1.3 Handshake (1 round-trip!)

```
Client                              Server
  │                                    │
  │── ClientHello + KeyShare ─────────▶│
  │                                    │
  │◀── ServerHello + KeyShare ────────│
  │◀── {Certificate} ────────────────│  (encrypted!)
  │◀── {Finished} ───────────────────│
  │                                    │
  │── {Finished} ─────────────────────▶│
  │                                    │
  │════ Encrypted Application Data ════│

TLS 1.3: 1-RTT (vs 2-RTT in TLS 1.2)
0-RTT resumption possible (but replay risk)
```

---

## 3. Cipher Suites {#cipher-suites}

```
TLS 1.2 cipher suite format:
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
     │      │         │    │    │
     │      │         │    │    └─ PRF hash (SHA384)
     │      │         │    └────── Mode (GCM = AEAD)
     │      │         └─────────── Encryption (AES-256)
     │      └───────────────────── Authentication (RSA cert)
     └──────────────────────────── Key Exchange (ECDHE)

TLS 1.3 cipher suites (simplified):
TLS_AES_256_GCM_SHA384
TLS_AES_128_GCM_SHA256
TLS_CHACHA20_POLY1305_SHA256
(Key exchange always ECDHE, removed from suite name)

Recommended (2024+):
- TLS_AES_256_GCM_SHA384 (hardware AES-NI)
- TLS_CHACHA20_POLY1305_SHA256 (mobile, no AES-NI)
- All TLS 1.3 suites use AEAD (Authenticated Encryption)
- All TLS 1.3 suites provide forward secrecy
```

---

## 4-5. Certificate Chain & OCSP {#cert-chain}

```
Certificate Chain (trust hierarchy):
Root CA (self-signed, in browser trust store)
  └─ Intermediate CA (signed by Root)
       └─ Server Certificate (signed by Intermediate)

Verification: Server cert → Intermediate → Root (trusted?)

OCSP (Online Certificate Status Protocol):
- Client asks CA: "Is this certificate still valid?"
- OCSP Stapling: Server pre-fetches OCSP response, sends with cert
  → Client doesn't need to contact CA (faster, more private)
```

---

## 6-7. Pinning & mTLS {#pinning}

### Certificate Pinning
```
Pin = hardcode expected cert/public key in client
- Prevents MITM with rogue CA certificate
- Used in: Mobile apps, IoT devices
- Deprecated in browsers (too fragile)
- Alternative: Certificate Transparency (CT) logs

Risk: If pinned cert expires/rotates → client breaks!
```

### mTLS (Mutual TLS)
```
Normal TLS: Only server presents certificate
mTLS: BOTH sides present certificates

Use cases:
- Service-to-service (microservices, service mesh)
- IoT device authentication
- API authentication (client certificates)
- Zero-trust networks

In Kubernetes (Istio/Cilium):
- Sidecar proxy handles mTLS automatically
- Services don't need to implement TLS themselves
```

---

## 8-10. TLS 1.3, ALPN, Best Practices {#tls13}

### TLS 1.3 Improvements (RFC 8446)
```
Removed:
❌ RSA key exchange (no forward secrecy)
❌ Static DH (no forward secrecy)
❌ RC4, 3DES, SHA-1
❌ CBC mode ciphers (BEAST, Lucky13)
❌ Compression (CRIME attack)
❌ Renegotiation

Added/Required:
✅ Forward secrecy mandatory (ECDHE/DHE only)
✅ AEAD only (GCM, ChaCha20-Poly1305)
✅ 1-RTT handshake (faster)
✅ 0-RTT resumption (optional, replay risk)
✅ Encrypted handshake (certificate hidden from eavesdropper)
✅ Simplified cipher suites
```

### ALPN (Application-Layer Protocol Negotiation)
```
ALPN negotiates application protocol DURING TLS handshake:
- HTTP/2 vs HTTP/1.1
- gRPC
- MQTT

Client: "I support h2, http/1.1"
Server: "Let's use h2"
→ No extra round trip needed!
```

### Best Practices
```nginx
# Nginx TLS configuration
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
add_header Strict-Transport-Security "max-age=63072000" always;
```

| Tài liệu | Mô tả |
|-----------|--------|
| RFC 8446 | TLS 1.3 Specification |
| RFC 6960 | OCSP Protocol |
| Mozilla SSL Configuration Generator | Best practice configs |
| SSL Labs (ssllabs.com) | Test your server |

---
