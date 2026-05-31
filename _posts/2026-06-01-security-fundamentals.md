---
layout: post
title: "Security - Phần 12: Security Fundamentals"
subtitle: "Mã hóa, TLS, PKI, IAM — nền tảng bảo mật cho Cloud"
gh-repo: wayarmy/wayarmy.github.io
tags: [security, aws, learning-path]
comments: true
date: 2026-06-01
categories: AWS-Learning-Path
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Phần 12 — Bài cuối).
>
> **Đối tượng:** Người mới hoàn toàn — biết networking và Linux cơ bản.
>
> **Nguồn tham khảo:**
> - RFC 5246 (2008) — The Transport Layer Security (TLS) Protocol Version 1.2
> - RFC 8446 (2018) — The Transport Layer Security (TLS) Protocol Version 1.3
> - NIST SP 800-57 — Recommendation for Key Management
> - NIST SP 800-63B — Digital Identity Guidelines: Authentication
> - AWS Security Documentation — [https://docs.aws.amazon.com/security/](https://docs.aws.amazon.com/security/)
> - AWS Well-Architected Framework — Security Pillar
> - Schneier, Bruce. "Applied Cryptography" — Wiley

---

## 1. Tại sao cần bảo mật? — Mở đầu

### Ví dụ đời thường:

Hãy tưởng tượng bạn gửi một **lá thư quan trọng** (chứa mật khẩu ngân hàng) cho bạn ở thành phố khác:

- **Không bảo mật:** Viết trên bưu thiếp → bưu tá, hàng xóm, ai cũng đọc được
- **Có bảo mật:** Bỏ vào phong bì niêm phong + khóa → chỉ người có chìa khóa mở được

Trên Internet, mọi dữ liệu đi qua **hàng chục router** trước khi đến đích. Nếu không mã hóa, ai "ngồi giữa đường" đều đọc được: mật khẩu, thẻ tín dụng, tin nhắn riêng tư...

### Ba trụ cột bảo mật (CIA Triad):

| Trụ cột | Ý nghĩa | Ví dụ đời thường |
|---------|---------|-----------------|
| **C**onfidentiality (Bí mật) | Chỉ người được phép mới đọc được | Phong bì niêm phong |
| **I**ntegrity (Toàn vẹn) | Data không bị sửa đổi trái phép | Con dấu xi → biết thư bị mở |
| **A**vailability (Sẵn sàng) | Hệ thống luôn hoạt động khi cần | ATM hoạt động 24/7 |

---

## 2. Encryption — Mã hóa dữ liệu

### Ví dụ đời thường:

Mã hóa giống viết thư bằng **mật mã** — ai không biết quy tắc thì không đọc được.

### Symmetric Encryption — "Một chìa khóa"

**Cùng 1 key** để mã hóa VÀ giải mã. Giống **1 ổ khóa + 1 chìa** — ai có chìa đều mở được.

```
Plaintext: "Hello World"
     │
     ▼ Encrypt (Key = "secret123")
     │
Ciphertext: "a1b2c3d4e5f6..."
     │
     ▼ Decrypt (Key = "secret123")
     │
Plaintext: "Hello World"
```

**Thuật toán phổ biến:**

| Thuật toán | Key size | Đặc điểm |
|-----------|----------|-----------|
| **AES-256** | 256 bits | Chuẩn hiện tại (NIST approved), nhanh |
| **AES-128** | 128 bits | Nhanh hơn, vẫn an toàn |
| **ChaCha20** | 256 bits | Nhanh trên mobile (không cần AES hardware) |
| ~~DES~~ | 56 bits | ❌ KHÔNG an toàn (brute-force dễ) |
| ~~3DES~~ | 168 bits | ❌ Chậm, deprecated |

**Ưu điểm:** Nhanh (xử lý data lớn hiệu quả)
**Nhược điểm:** Làm sao gửi key an toàn cho đối phương? (Key distribution problem)

### Asymmetric Encryption — "Hai chìa khóa"

Dùng **cặp key**: Public Key (khóa công khai) + Private Key (khóa bí mật).

**Ví dụ đời thường:**
- **Public Key** = Hộp thư có khe gửi (ai cũng bỏ thư vào được)
- **Private Key** = Chìa khóa mở hộp thư (chỉ chủ nhà có)

```
Alice muốn gửi tin MẬT cho Bob:

1. Bob tạo cặp key: Public Key (PB) + Private Key (KB)
2. Bob gửi Public Key cho Alice (CÔNG KHAI — ai cũng biết OK)
3. Alice mã hóa message bằng Bob's Public Key:
   Encrypt("Hello Bob", PB) → "x7y8z9..."
4. Gửi ciphertext cho Bob
5. Bob dùng Private Key giải mã:
   Decrypt("x7y8z9...", KB) → "Hello Bob"

→ Chỉ Bob (có Private Key) mới giải mã được!
→ Kể cả ai bắt được Public Key cũng KHÔNG giải được!
```

**Thuật toán phổ biến:**

| Thuật toán | Key size | Use case |
|-----------|----------|----------|
| **RSA** | 2048-4096 bits | Key exchange, digital signatures |
| **ECDSA** | 256-384 bits | Digital signatures (ngắn hơn RSA) |
| **Ed25519** | 256 bits | SSH keys, signatures (modern, fast) |
| **ECDH** | 256-384 bits | Key exchange (Diffie-Hellman on elliptic curves) |

**Ưu điểm:** Giải quyết key distribution problem
**Nhược điểm:** Chậm hơn symmetric 100-1000x → không dùng cho data lớn

### Hybrid Encryption (Thực tế dùng cả hai):

TLS/HTTPS kết hợp cả hai:
1. **Asymmetric** để trao đổi key an toàn (slow, nhưng chỉ 1 lần)
2. **Symmetric** (AES) để mã hóa data thực tế (fast, cho toàn bộ session)

```
Client                              Server
  │                                    │
  │──── [Asymmetric] Exchange keys ───→│  (Chậm, 1 lần)
  │                                    │
  │     Now both have shared AES key    │
  │                                    │
  │←════ [Symmetric AES] Data ══════→│  (Nhanh, suốt session)
  │←════ [Symmetric AES] Data ══════→│
```

---

## 3. Hashing — "Dấu vân tay" của dữ liệu

### Ví dụ đời thường:

Hash giống **dấu vân tay** — mỗi người có dấu vân tay UNIQUE, và:
- Từ người → lấy vân tay (dễ)
- Từ vân tay → tìm lại người (CỰC KHÓ/bất khả thi)
- Hai người khác nhau → vân tay KHÁC nhau

### Đặc điểm của Hash Function:

| Tính chất | Mô tả |
|-----------|--------|
| **Deterministic** | Cùng input → LUÔN cùng output |
| **One-way** | Không thể reverse (từ hash → tìm lại input) |
| **Avalanche effect** | Thay đổi 1 bit input → output thay đổi hoàn toàn |
| **Collision resistant** | Cực khó tìm 2 inputs cho cùng hash |
| **Fixed length** | Output luôn cùng kích thước (dù input lớn bao nhiêu) |

### Thuật toán Hash phổ biến:

| Thuật toán | Output | Status |
|-----------|--------|--------|
| **SHA-256** | 256 bits (64 hex chars) | ✅ Chuẩn hiện tại |
| **SHA-3** | 256/512 bits | ✅ Mới nhất (Keccak) |
| **SHA-512** | 512 bits | ✅ Cho ứng dụng cần hash dài |
| **bcrypt** | 184 bits | ✅ Cho passwords (intentionally slow) |
| **Argon2** | Configurable | ✅ Winner of PHC, best for passwords |
| ~~MD5~~ | 128 bits | ❌ BROKEN (collision found) |
| ~~SHA-1~~ | 160 bits | ❌ BROKEN (collision found 2017) |

### Use cases:

```bash
# 1. Verify file integrity (download không bị corrupt)
$ sha256sum ubuntu-22.04.iso
a1b2c3d4e5...  ubuntu-22.04.iso

# So sánh với hash trên website → nếu giống = file OK

# 2. Password storage (KHÔNG BAO GIỜ lưu password plaintext!)
# Thay vì lưu: password = "mypassword123"
# Lưu: password_hash = bcrypt("mypassword123") = "$2b$12$LJ3..."
# Khi login: bcrypt(input) == stored_hash ?

# 3. Data integrity (HMAC — Hash-based Message Authentication Code)
# Đảm bảo message không bị sửa đổi giữa đường
```

---

## 4. TLS/SSL Handshake — "Bắt tay" an toàn

### TLS (Transport Layer Security) là gì?

TLS là giao thức mã hóa giữa client và server — nền tảng của HTTPS. SSL (Secure Sockets Layer) là tiền thân, đã deprecated → dùng TLS.

| Version | Year | Status |
|---------|------|--------|
| SSL 2.0 | 1995 | ❌ Deprecated |
| SSL 3.0 | 1996 | ❌ Deprecated (POODLE attack) |
| TLS 1.0 | 1999 | ❌ Deprecated |
| TLS 1.1 | 2006 | ❌ Deprecated |
| **TLS 1.2** | 2008 | ✅ Widely used (RFC 5246) |
| **TLS 1.3** | 2018 | ✅ Recommended (RFC 8446) |

### TLS 1.2 Handshake (RFC 5246):

```
Client                                    Server
  │                                          │
  │──── ClientHello ────────────────────────→│
  │     (TLS version, cipher suites,         │
  │      random number)                      │
  │                                          │
  │←─── ServerHello ────────────────────────│
  │     (chosen cipher suite,                │
  │      random number)                      │
  │                                          │
  │←─── Certificate ───────────────────────│
  │     (Server's X.509 certificate         │
  │      containing public key)             │
  │                                          │
  │←─── ServerHelloDone ──────────────────│
  │                                          │
  │──── ClientKeyExchange ─────────────────→│
  │     (Pre-master secret encrypted        │
  │      with server's public key)          │
  │                                          │
  │──── ChangeCipherSpec ──────────────────→│
  │──── Finished (encrypted) ──────────────→│
  │                                          │
  │←─── ChangeCipherSpec ──────────────────│
  │←─── Finished (encrypted) ──────────────│
  │                                          │
  │═══════ Encrypted Application Data ════════│
```

**Tóm tắt bước:**
1. Client: "Tôi hỗ trợ TLS 1.2, các cipher: AES-256, ChaCha20..."
2. Server: "OK, dùng AES-256-GCM. Đây là certificate (chứa public key) của tôi"
3. Client: Verify certificate → tạo pre-master secret → encrypt bằng server's public key → gửi
4. Cả hai: Derive session keys từ pre-master secret
5. Bắt đầu truyền data bằng symmetric encryption (AES)

### TLS 1.3 — Nhanh hơn (RFC 8446):

TLS 1.3 giảm handshake từ **2 round-trips** xuống **1 round-trip** (hoặc 0 round-trip với 0-RTT resumption):

```
Client                          Server
  │                               │
  │──── ClientHello + KeyShare ──→│   (gửi key luôn trong Hello)
  │                               │
  │←── ServerHello + KeyShare ───│
  │←── Certificate + Finished ───│
  │                               │
  │──── Finished ────────────────→│
  │                               │
  │═══ Encrypted Data ═══════════│   (bắt đầu nhanh hơn 1 RTT)
```

---

## 5. PKI & Certificates — "Hệ thống công chứng" trên Internet

### Ví dụ đời thường:

Làm sao bạn biết website `https://google.com` THẬT SỰ là Google, không phải trang giả?

Giống như **giấy tờ có công chứng**:
- **Certificate** = CMND/CCCD (chứng minh danh tính)
- **Certificate Authority (CA)** = Cơ quan công an cấp CMND (bên thứ 3 đáng tin cậy)
- **Trust chain** = Công an cấp tỉnh → được Bộ Công an ủy quyền → được Nhà nước ủy quyền

### X.509 Certificate chứa gì?

```
Certificate:
    Data:
        Version: 3
        Serial Number: 04:00:00:...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN=Let's Encrypt Authority X3, O=Let's Encrypt
        Validity:
            Not Before: Jun  1 00:00:00 2026 GMT
            Not After : Aug 30 23:59:59 2026 GMT
        Subject: CN=www.example.com
        Subject Public Key:
            RSA Public Key (2048 bit)
            [public key data...]
    Signature:
        [CA's digital signature over everything above]
```

**Các trường quan trọng:**
- **Subject:** Ai sở hữu cert (domain name)
- **Issuer:** CA nào cấp
- **Validity:** Thời hạn hiệu lực
- **Public Key:** Key của server
- **Signature:** Chữ ký số của CA (chứng minh cert là thật)

### Chain of Trust:

```
┌─────────────────────────┐
│    Root CA Certificate    │  ← Trusted (built into OS/browser)
│    (DigiCert, Let's Encrypt Root)
└────────────┬────────────┘
             │ Signs
┌────────────▼────────────┐
│  Intermediate CA Cert    │  ← Signed by Root
│  (Let's Encrypt R3)      │
└────────────┬────────────┘
             │ Signs
┌────────────▼────────────┐
│   Server Certificate     │  ← Signed by Intermediate
│   (www.example.com)      │
└──────────────────────────┘
```

**Browser verification:**
1. Server gửi cert chain (server cert + intermediate)
2. Browser kiểm tra: intermediate signed by trusted root? ✓
3. Browser kiểm tra: server cert signed by intermediate? ✓
4. Browser kiểm tra: cert chưa hết hạn? ✓
5. Browser kiểm tra: domain name khớp? ✓
6. → HTTPS lock icon 🔒 hiện lên!

### Certificate types:

| Type | Verification | Cost | Use case |
|------|-------------|------|----------|
| **DV** (Domain Validation) | Chỉ verify domain ownership | Free (Let's Encrypt) | Blog, app |
| **OV** (Organization Validation) | Verify org identity | $$ | Business site |
| **EV** (Extended Validation) | Strict org verification | $$$ | Banking, enterprise |
| **Wildcard** | `*.example.com` | Varies | Multiple subdomains |
| **SAN** | Multiple domains in 1 cert | Varies | Multi-domain hosting |

---

## 6. Authentication vs Authorization — Xác thực vs Phân quyền

### Ví dụ đời thường:

| | Authentication (AuthN) | Authorization (AuthZ) |
|-|----------------------|---------------------|
| **Câu hỏi** | "BẠN LÀ AI?" | "BẠN ĐƯỢC LÀM GÌ?" |
| **Ví dụ** | Bảo vệ kiểm tra CMND → "Đúng là anh Minh" | Bảo vệ check: "Anh Minh chỉ vào tầng 3, không lên tầng 5" |
| **Kết quả** | Identity confirmed | Permissions granted/denied |

### Authentication methods (NIST SP 800-63B):

| Factor | "Something you..." | Ví dụ |
|--------|-------------------|-------|
| **Knowledge** | Know (biết) | Password, PIN, security question |
| **Possession** | Have (có) | Phone (OTP), hardware key (YubiKey), smart card |
| **Inherence** | Are (là) | Fingerprint, face, voice, retina |

**MFA (Multi-Factor Authentication):** Kết hợp 2+ factors → an toàn hơn nhiều.

```
Login flow với MFA:
1. Username + Password   → Factor 1 (Knowledge) ✓
2. Enter OTP from phone  → Factor 2 (Possession) ✓
3. → Access granted!
```

### Authorization models:

| Model | Mô tả | Use case |
|-------|--------|----------|
| **RBAC** (Role-Based) | Quyền theo vai trò | Employee, Manager, Admin |
| **ABAC** (Attribute-Based) | Quyền theo attributes | "If dept=finance AND time=office_hours" |
| **ACL** (Access Control List) | Danh sách quyền cho từng resource | File permissions (Linux) |
| **Policy-Based** | Quyền theo policy documents | AWS IAM Policies |

---

## 7. AWS Security — Shared Responsibility Model

### Mô hình "Chia sẻ trách nhiệm":

**Ví dụ đời thường:** Bạn thuê căn hộ chung cư:
- **Chủ đầu tư (AWS) chịu trách nhiệm:** Móng nhà, kết cấu, thang máy, an ninh tầng hầm, hệ thống điện nước chung
- **Bạn (Customer) chịu trách nhiệm:** Khóa cửa, lắp camera phòng, không để người lạ vào, bảo quản đồ đạc

```
┌─────────────────────────────────────────────────────────────┐
│              CUSTOMER RESPONSIBILITY                         │
│        "Security IN the Cloud"                              │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ Customer Data│  │ IAM / Access │  │ OS, Network, FW  │ │
│  │              │  │  Management  │  │ config on EC2    │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ Encryption   │  │ Application  │  │ Client-side data │ │
│  │ (at rest +   │  │   Security   │  │ encryption       │ │
│  │  in transit) │  │              │  │                  │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
├─────────────────────────────────────────────────────────────┤
│              AWS RESPONSIBILITY                              │
│        "Security OF the Cloud"                              │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │  Hardware /  │  │   Network    │  │  Managed Service │ │
│  │ Infrastructure│  │Infrastructure│  │  Software (RDS,  │ │
│  │ (Regions, AZ,│  │              │  │  Lambda, etc.)   │ │
│  │  Edge)       │  │              │  │                  │ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
│  ┌──────────────┐  ┌──────────────┐                       │
│  │  Compute     │  │  Storage     │                       │
│  │  (Hypervisor)│  │  (Disk)      │                       │
│  └──────────────┘  └──────────────┘                       │
└─────────────────────────────────────────────────────────────┘
```

**Quy tắc ngón tay:**
- **Nếu bạn CONFIGURE nó → bạn chịu trách nhiệm** (SG rules, IAM policies, encryption)
- **Nếu AWS MANAGE nó → AWS chịu trách nhiệm** (hardware, hypervisor, network backbone)

---

## 8. AWS IAM — Identity and Access Management

### IAM là gì?

**IAM** = Hệ thống quản lý "AI được làm GÌ" trên AWS. Giống **hệ thống badge và quyền truy cập** trong tòa nhà công ty.

### IAM Components:

| Component | Ví dụ đời thường | Mô tả |
|-----------|-----------------|-------|
| **User** | Nhân viên (con người) | Entity đăng nhập AWS Console / CLI |
| **Group** | Phòng ban | Nhóm users có cùng quyền |
| **Role** | Tấm "thẻ tạm" cho phép làm việc X | Entity assume tạm thời (EC2, Lambda) |
| **Policy** | Nội quy viết chi tiết | JSON document định nghĩa permissions |

### IAM Policy — JSON Document:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    },
    {
      "Sid": "DenyDeleteAnything",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "*"
    }
  ]
}
```

**Đọc policy:**
- **Effect:** Allow hoặc Deny
- **Action:** Hành động cụ thể (s3:GetObject, ec2:StartInstance)
- **Resource:** Tài nguyên nào (ARN — Amazon Resource Name)
- **Condition** (optional): Điều kiện kèm theo (IP, time, MFA, etc.)

**Nguyên tắc: Deny WIN** — Nếu có cả Allow VÀ Deny → DENY thắng.

### IAM Best Practices:

1. **Principle of Least Privilege:** Chỉ cấp quyền TỐI THIỂU cần thiết
2. **KHÔNG dùng Root account** cho daily tasks
3. **Bật MFA** cho root và tất cả IAM users
4. **Dùng Roles** cho EC2/Lambda (không hardcode credentials)
5. **Dùng Groups** để quản lý quyền (không gán policy trực tiếp cho user)
6. **Rotate credentials** định kỳ
7. **Audit** bằng CloudTrail + IAM Access Analyzer

### IAM Role cho EC2 (Instance Profile):

```
┌──── EC2 Instance ────────────────────────────┐
│                                               │
│  App code gọi:                                │
│  aws s3 cp file.txt s3://my-bucket/          │
│                                               │
│  → KHÔNG cần access key!                     │
│  → EC2 tự động lấy temporary credentials     │
│    từ Instance Metadata Service               │
│                                               │
│  IAM Role attached: "S3ReadWriteRole"         │
│  Policy: Allow s3:PutObject, s3:GetObject     │
│                                               │
└───────────────────────────────────────────────┘
```

---

## 9. AWS KMS và ACM

### AWS KMS (Key Management Service):

**KMS** = Quản lý encryption keys cho bạn — tạo, lưu trữ, rotate keys an toàn.

```
Encryption at rest with KMS:

Data → KMS (encrypt with CMK) → Encrypted data → stored in S3/EBS/RDS

Decryption:
Encrypted data → KMS (decrypt with CMK) → Plaintext data → App
```

**Envelope Encryption (cách KMS hoạt động thực tế):**

```
┌─── KMS ─────────────────────┐
│ Customer Master Key (CMK)    │
│ (never leaves KMS!)          │
└──────────┬──────────────────┘
           │ Encrypts
┌──────────▼──────────────────┐
│ Data Encryption Key (DEK)    │  ← Generated by KMS
│ (plaintext + encrypted)      │
└──────────┬──────────────────┘
           │ DEK encrypts
┌──────────▼──────────────────┐
│ Your actual data             │  ← Encrypted with DEK
│ (stored in S3, EBS, etc.)    │
└──────────────────────────────┘

Storage: Encrypted DEK + Encrypted Data (CMK stays in KMS hardware)
```

**KMS Key types:**
- **AWS managed:** AWS tạo/quản lý (miễn phí cho dịch vụ AWS)
- **Customer managed (CMK):** Bạn tạo, control policies, rotation
- **Customer provided:** Bạn import key material (full control)

### AWS ACM (Certificate Manager):

**ACM** = Quản lý TLS certificates — tạo, deploy, renew TỰ ĐỘNG.

```bash
# Tạo certificate qua CLI
aws acm request-certificate \
    --domain-name "example.com" \
    --subject-alternative-names "*.example.com" \
    --validation-method DNS

# ACM tự động:
# 1. Issue certificate (DV validation via DNS)
# 2. Deploy to ALB/CloudFront/API Gateway
# 3. Auto-renew trước khi hết hạn (60 days before expiry)
```

**ACM + ALB (phổ biến nhất):**

```
Internet ──HTTPS──→ ALB (ACM cert: *.example.com)
                     │
              ──HTTP──→ EC2 instances (internal, không cần HTTPS)
```

→ **SSL/TLS termination** tại ALB: Client ↔ ALB encrypted; ALB ↔ EC2 plaintext (an toàn vì trong VPC).

---

## 10. Encryption at Rest vs in Transit

### Encryption at Rest — "Khóa tủ khi ngủ":

Dữ liệu **nằm yên** trên disk/storage được mã hóa → nếu ai ăn cắp ổ cứng, không đọc được.

| AWS Service | Encryption at Rest |
|-------------|-------------------|
| S3 | SSE-S3, SSE-KMS, SSE-C |
| EBS | AES-256 via KMS |
| RDS | AES-256 via KMS (enable khi tạo) |
| DynamoDB | Default encrypted |
| Redshift | AES-256 |

### Encryption in Transit — "Niêm phong thư khi gửi":

Dữ liệu **đang truyền** trên network được mã hóa → ai bắt giữa đường không đọc được.

| Kết nối | Protocol |
|---------|----------|
| Browser ↔ Server | HTTPS (TLS) |
| SSH | SSH protocol (encrypted) |
| Database ↔ App | TLS connection |
| AWS API calls | HTTPS (all AWS APIs require TLS) |
| VPN | IPsec, OpenVPN |

---

## 11. Các mối đe dọa phổ biến

### OWASP Top 10 (simplified):

| Threat | Mô tả | Phòng chống |
|--------|--------|-------------|
| **Injection** (SQL, Command) | Attacker chèn code vào input | Parameterized queries, input validation |
| **Broken Authentication** | Weak passwords, no MFA | MFA, strong password policy, rate limiting |
| **Sensitive Data Exposure** | Data không encrypted | Encrypt at rest + in transit |
| **XSS** (Cross-Site Scripting) | Inject malicious script vào web page | Output encoding, CSP headers |
| **Broken Access Control** | User truy cập resource không phải của mình | RBAC, check permissions server-side |
| **Security Misconfiguration** | Default passwords, open ports | Hardening, regular audits |

### AWS Security Services:

| Service | Chức năng |
|---------|-----------|
| **GuardDuty** | Threat detection (phát hiện hành vi bất thường) |
| **Inspector** | Vulnerability scanning (EC2, containers) |
| **WAF** | Web Application Firewall (block SQL injection, XSS) |
| **Shield** | DDoS protection |
| **CloudTrail** | API audit logging (ai làm gì, khi nào) |
| **Config** | Resource configuration compliance |
| **Security Hub** | Centralized security findings |
| **Macie** | Sensitive data discovery (PII in S3) |

---

## 12. Thực hành: Lab tự làm

### Lab 1: Hash và Encryption

```bash
# SHA-256 hash
echo -n "Hello World" | sha256sum
# a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e

# Thay đổi 1 ký tự → hash hoàn toàn khác (avalanche effect)
echo -n "Hello World!" | sha256sum
# 7f83b1657ff1fc53b92dc18148a1d65dfc2d4b1fa3d677284addd200126d9069

# OpenSSL - Symmetric encryption
echo "Secret message" | openssl enc -aes-256-cbc -pbkdf2 -out encrypted.bin
openssl enc -d -aes-256-cbc -pbkdf2 -in encrypted.bin

# Generate RSA key pair
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem

# Encrypt with public key
echo "Hello" | openssl rsautl -encrypt -pubin -inkey public.pem -out cipher.bin

# Decrypt with private key
openssl rsautl -decrypt -inkey private.pem -in cipher.bin
```

### Lab 2: TLS Certificate inspection

```bash
# Xem certificate của website
openssl s_client -connect google.com:443 -showcerts < /dev/null

# Xem certificate details
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -text -noout

# Xem expiry date
echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -dates -noout

# Verify certificate chain
openssl s_client -connect google.com:443 -verify 5
```

### Lab 3: AWS IAM

```bash
# 1. Tạo IAM user
aws iam create-user --user-name devuser

# 2. Tạo policy
cat > s3-read-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
    }
  ]
}
EOF

aws iam create-policy --policy-name S3ReadOnly --policy-document file://s3-read-policy.json

# 3. Attach policy to user
aws iam attach-user-policy --user-name devuser --policy-arn arn:aws:iam::123456789:policy/S3ReadOnly

# 4. Test: devuser có thể đọc S3, KHÔNG thể xóa
aws s3 ls s3://my-bucket/          # ✓ OK
aws s3 rm s3://my-bucket/file.txt  # ✗ AccessDenied
```

### Lab 4: KMS + S3 Encryption

```bash
# 1. Tạo KMS key
aws kms create-key --description "My encryption key"

# 2. Upload file encrypted bằng KMS
aws s3 cp secret.txt s3://my-bucket/ --sse aws:kms --sse-kms-key-id <key-id>

# 3. Verify encryption
aws s3api head-object --bucket my-bucket --key secret.txt
# "ServerSideEncryption": "aws:kms"

# 4. Download (tự động decrypt nếu có quyền KMS)
aws s3 cp s3://my-bucket/secret.txt downloaded.txt
```

### Lab 5: ACM + ALB

1. Request certificate qua ACM (DNS validation)
2. Add DNS record để validate domain ownership
3. Create ALB → add HTTPS listener (port 443) → attach ACM cert
4. Create HTTP listener (port 80) → redirect to HTTPS
5. Verify: truy cập http://yourdomain.com → auto redirect HTTPS 🔒

---

## 13. Security Checklist cho AWS Environment

```markdown
## Minimum Security Checklist

### Account Level:
- [ ] Root account: MFA enabled, no access keys
- [ ] Billing alerts configured
- [ ] CloudTrail enabled (all regions)
- [ ] GuardDuty enabled

### IAM:
- [ ] Individual IAM users (no shared accounts)
- [ ] MFA for all users
- [ ] Least privilege policies
- [ ] EC2 uses IAM Roles (not access keys)
- [ ] Rotate credentials every 90 days

### Network:
- [ ] VPC with public + private subnets
- [ ] Security Groups: minimum required ports
- [ ] NACLs: deny known bad IPs
- [ ] No 0.0.0.0/0 on SSH (port 22)

### Data:
- [ ] Encryption at rest: S3, EBS, RDS
- [ ] Encryption in transit: HTTPS everywhere
- [ ] S3 bucket: Block Public Access enabled
- [ ] RDS: not publicly accessible

### Monitoring:
- [ ] CloudWatch alarms for unusual activity
- [ ] VPC Flow Logs enabled
- [ ] AWS Config rules for compliance
```

---

## 14. Tổng kết

| Khái niệm | Ví dụ đời thường | AWS Service |
|-----------|-----------------|-------------|
| Symmetric encryption | 1 chìa khóa chung | KMS (AES-256) |
| Asymmetric encryption | Hộp thư (public) + chìa khóa (private) | KMS (RSA), ACM |
| Hashing | Dấu vân tay dữ liệu | SHA-256, bcrypt |
| TLS/HTTPS | Phong bì niêm phong khi gửi thư | ACM + ALB/CloudFront |
| Certificate | CMND (chứng minh danh tính) | ACM certificates |
| CA | Cơ quan cấp CMND | Let's Encrypt, DigiCert |
| Authentication | "Bạn là ai?" | IAM Users, Cognito |
| Authorization | "Bạn được làm gì?" | IAM Policies, Roles |
| Shared Responsibility | Chung cư: chủ đầu tư vs cư dân | AWS vs Customer |
| Encryption at rest | Khóa tủ khi ngủ | KMS + S3/EBS/RDS |
| Encryption in transit | Niêm phong thư khi gửi | TLS/HTTPS |

---

## 🎉 Kết thúc Series — Bạn đã hoàn thành IT Foundation!

Qua 12 bài viết, bạn đã nắm vững:

1. ✅ **Networking:** OSI model, IP, subnetting, routing, DNS, HTTP, TCP/UDP, NAT, firewalls
2. ✅ **Linux:** CLI, filesystem, services, processes, bash scripting
3. ✅ **Containers:** Docker, Dockerfile, Docker Compose, ECS/Fargate
4. ✅ **Databases:** SQL, NoSQL, OLTP/OLAP, Data Lake, AWS database services
5. ✅ **Security:** Encryption, TLS, PKI, IAM, Shared Responsibility Model

**Bước tiếp theo:**
- Lấy chứng chỉ **AWS Cloud Practitioner** (CCP)
- Sau đó: **AWS Solutions Architect Associate** (SAA)
- Hands-on: Xây dựng project thực tế trên AWS

---

## Tài liệu tham khảo

1. **RFC 5246** — Dierks, T., Rescorla, E. (2008). "The TLS Protocol Version 1.2". [https://www.rfc-editor.org/rfc/rfc5246](https://www.rfc-editor.org/rfc/rfc5246)
2. **RFC 8446** — Rescorla, E. (2018). "The TLS Protocol Version 1.3". [https://www.rfc-editor.org/rfc/rfc8446](https://www.rfc-editor.org/rfc/rfc8446)
3. **NIST SP 800-57** — "Recommendation for Key Management". [https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final)
4. **NIST SP 800-63B** — "Digital Identity Guidelines: Authentication and Lifecycle Management". [https://pages.nist.gov/800-63-3/sp800-63b.html](https://pages.nist.gov/800-63-3/sp800-63b.html)
5. **AWS Security Best Practices** — [https://docs.aws.amazon.com/security/](https://docs.aws.amazon.com/security/)
6. **AWS Well-Architected Framework — Security Pillar** — [https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)
7. **AWS IAM User Guide** — [https://docs.aws.amazon.com/IAM/latest/UserGuide/](https://docs.aws.amazon.com/IAM/latest/UserGuide/)
8. **AWS KMS Developer Guide** — [https://docs.aws.amazon.com/kms/latest/developerguide/](https://docs.aws.amazon.com/kms/latest/developerguide/)
9. **Schneier, Bruce.** "Applied Cryptography", 2nd Edition — Wiley.
10. **OWASP Top 10** — [https://owasp.org/www-project-top-ten/](https://owasp.org/www-project-top-ten/)
