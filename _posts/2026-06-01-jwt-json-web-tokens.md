---
layout: post
title: "JWT - JSON Web Tokens chuyên sâu"
date: 2026-06-01
categories: [security]
tags: [jwt, rs256, es256, tokens, authentication, security]
---

## Mục lục
1. [Góc nhìn tổng quan - Thẻ vào cửa thông minh](#overview)
2. [JWT Structure: header.payload.signature](#structure)
3. [Claims - Thông tin trong token](#claims)
4. [Algorithms: RS256, ES256, HS256](#algorithms)
5. [Token Validation Process](#validation)
6. [Refresh Token Rotation](#refresh)
7. [Security Pitfalls - Lỗi thường gặp](#pitfalls)
8. [JWT vs Session cookies](#jwt-vs-session)
9. [Implementation best practices](#implementation)
10. [Tổng kết](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

JWT giống **thẻ vào cửa sự kiện**:
- **Header** = loại thẻ (VIP/Standard) và công nghệ bảo mật (barcode/QR)
- **Payload** = thông tin trên thẻ (tên, ghế số mấy, valid đến khi nào)
- **Signature** = con dấu hologram (chống giả mạo)
- Bảo vệ kiểm tra: nhìn hologram → verify → cho vào (KHÔNG cần gọi lại ban tổ chức)
- Stateless: không cần database lookup mỗi request!

---

## 2. JWT Structure {#structure}

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiJ1c2VyLTEyMyIsIm5hbWUiOiJKb2huIiwiZXhwIjoxNzA2MTQwODAwfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c

Three parts separated by dots:
1. HEADER (base64url encoded):
   {"alg": "RS256", "typ": "JWT", "kid": "key-001"}
   - alg: Signing algorithm
   - typ: Token type
   - kid: Key ID (which key to verify with)

2. PAYLOAD (base64url encoded):
   {
     "sub": "user-123",       // Subject (user ID)
     "name": "John Doe",
     "email": "john@example.com",
     "roles": ["admin"],
     "iss": "auth.example.com",  // Issuer
     "aud": "api.example.com",   // Audience
     "exp": 1706140800,          // Expiration (Unix timestamp)
     "iat": 1706137200,          // Issued at
     "nbf": 1706137200           // Not before
   }

3. SIGNATURE:
   RS256: RSA_SHA256(base64url(header) + "." + base64url(payload), private_key)
   HS256: HMAC_SHA256(base64url(header) + "." + base64url(payload), secret)
```

---

## 3. Claims {#claims}

```
Registered Claims (RFC 7519):
- iss (Issuer): Who created the token
- sub (Subject): Who the token is about (user ID)
- aud (Audience): Who the token is for (API)
- exp (Expiration): When token expires
- nbf (Not Before): Token not valid before
- iat (Issued At): When token was created
- jti (JWT ID): Unique token identifier (for revocation)

Public Claims: Registered at IANA (name, email, picture)
Private Claims: Custom (roles, permissions, tenant_id)

IMPORTANT: Payload is NOT encrypted! Only base64 encoded.
Anyone can decode and READ the payload.
Signature only prevents MODIFICATION, not reading!
→ NEVER put secrets in JWT payload!
```

---

## 4. Algorithms {#algorithms}

```
SYMMETRIC (shared secret):
HS256: HMAC + SHA-256
- Same secret for sign AND verify
- Fast, simple
- Problem: Every service that verifies needs the secret
- Use: Single service, internal tokens

ASYMMETRIC (key pair):
RS256: RSA + SHA-256 (2048+ bit key)
- Private key signs, Public key verifies
- Verifiers only need public key (safe to share)
- Larger tokens (~350 bytes signature)
- Use: Microservices, external consumers

ES256: ECDSA + SHA-256 (P-256 curve)
- Same concept as RS256 but smaller keys
- Shorter signatures (~90 bytes)
- Faster verification
- Use: Mobile, IoT, bandwidth-sensitive

EdDSA: Ed25519
- Newest, fastest, smallest
- Excellent security properties
- Growing adoption

Recommendation:
- Single service: HS256 (simplest)
- Multiple services: RS256 or ES256
- Performance-critical: ES256 or EdDSA
```

---

## 5-7. Validation, Refresh, Pitfalls {#validation}

### Validation Checklist
```python
def validate_jwt(token, expected_audience, expected_issuer):
    # 1. Decode header (without verifying)
    header = decode_header(token)
    
    # 2. Find signing key (from JWKS endpoint)
    key = get_key_from_jwks(header["kid"])
    
    # 3. Verify signature
    verify_signature(token, key, header["alg"])
    
    # 4. Check standard claims
    payload = decode_payload(token)
    assert payload["exp"] > current_time()     # Not expired
    assert payload["nbf"] <= current_time()    # Valid now
    assert payload["iss"] == expected_issuer   # Right issuer
    assert expected_audience in payload["aud"] # Right audience
    
    # 5. Check custom claims as needed
    assert "admin" in payload.get("roles", [])
    
    return payload
```

### Security Pitfalls
```
1. Algorithm Confusion Attack:
   Attacker changes header alg: RS256 → HS256
   Signs with PUBLIC key (which they know!)
   If server doesn't enforce algorithm → accepts forged token!
   FIX: Always validate algorithm server-side, don't trust header

2. "none" Algorithm:
   Attacker sets alg: "none", removes signature
   Vulnerable libraries accept unsigned tokens!
   FIX: Never accept "none" algorithm

3. Token stored in localStorage:
   XSS attack → steal token → full account takeover
   FIX: Store in httpOnly cookie or memory

4. No expiration / Very long expiration:
   Stolen token valid forever
   FIX: Short access token (15min), use refresh tokens

5. JWT too large (in every request):
   Network overhead, cookie size limits
   FIX: Keep payload minimal, use token reference pattern
```

---
