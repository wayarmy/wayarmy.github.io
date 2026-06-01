---
layout: post
title: "SSH Deep Dive - DH/ECDH Key Exchange, Host Key Verification, Auth Methods, Tunneling & Agent Forwarding"
date: 2026-06-01
categories: [networking]
tags: [ssh, cryptography, tunneling, security, linux]
---

# SSH Deep Dive — DH/ECDH Key Exchange, Host Key Verification, Auth Methods, Tunneling & Agent Forwarding

## Mục lục (Table of Contents)
1. [Giới thiệu bằng câu chuyện đời thường](#1-giới-thiệu-bằng-câu-chuyện-đời-thường)
2. [SSH Architecture — Kiến trúc giao thức](#2-ssh-architecture--kiến-trúc-giao-thức)
3. [Key Exchange — DH và ECDH](#3-key-exchange--dh-và-ecdh)
4. [Host Key Verification — Xác minh danh tính server](#4-host-key-verification--xác-minh-danh-tính-server)
5. [Authentication Methods — Các phương thức xác thực](#5-authentication-methods--các-phương-thức-xác-thực)
6. [Local Port Forwarding — Đường hầm tới server xa](#6-local-port-forwarding--đường-hầm-tới-server-xa)
7. [Remote Port Forwarding — Mở cổng ngược](#7-remote-port-forwarding--mở-cổng-ngược)
8. [Dynamic Port Forwarding — SOCKS Proxy](#8-dynamic-port-forwarding--socks-proxy)
9. [SSH Agent Forwarding — Chuyển tiếp xác thực](#9-ssh-agent-forwarding--chuyển-tiếp-xác-thực)
10. [Tổng kết và Best Practices](#10-tổng-kết-và-best-practices)

---

## 1. Giới thiệu bằng câu chuyện đời thường

### SSH như một cuộc gọi mật trong phim gián điệp

Hãy tưởng tượng bạn là điệp viên cần liên lạc với tổng hành dinh. Bạn cần:

1. **Xác minh đối phương** là tổng hành dinh thật (không phải kẻ thù giả mạo) → **Host Key Verification**
2. **Chứng minh bạn là ai** (điệp viên thật, không phải kẻ thù) → **Authentication**
3. **Tạo mật mã riêng** cho cuộc nói chuyện này (chỉ 2 bên biết) → **Key Exchange**
4. **Nói chuyện bí mật** mà không ai nghe được → **Encrypted Session**
5. **Gửi tài liệu qua đường hầm bí mật** → **SSH Tunneling**

| Gián điệp | SSH |
|---|---|
| Kiểm tra giọng nói đối phương | Host key fingerprint verification |
| Đưa thẻ căn cước | Password authentication |
| Đưa con dấu bí mật chỉ mình có | Public key authentication |
| Thỏa thuận mật mã qua câu đố (ai nghe cũng không giải được) | Diffie-Hellman key exchange |
| Nói chuyện bằng mật mã | AES-256 encrypted session |
| Gửi tài liệu qua đường hầm ngầm | SSH port forwarding |
| Nhờ đồng nghiệp mở cửa thay mình | SSH agent forwarding |

### Tại sao SSH quan trọng?

- **100% server** trên Internet dùng SSH để quản lý
- SSH là **nền tảng** cho Git (push/pull), SCP/SFTP (chuyển file), Ansible (automation)
- Hiểu SSH giúp bạn **debug connection issues**, **bảo mật server**, **tạo tunnels** hiệu quả
- SSH thay thế Telnet, rlogin, rsh (tất cả đều plaintext, không an toàn)

---

## 2. SSH Architecture — Kiến trúc giao thức

### 2.1 Ba lớp của SSH (RFC 4251)

SSH được chia thành 3 lớp (layers), mỗi lớp có RFC riêng:

```
┌──────────────────────────────────────────────────────┐
│ SSH Connection Protocol (RFC 4254)                    │
│ • Channels (session, forwarded-tcpip, direct-tcpip)  │
│ • Port forwarding, X11 forwarding                    │
│ • Multiple sessions on 1 connection                  │
├──────────────────────────────────────────────────────┤
│ SSH User Authentication Protocol (RFC 4252)          │
│ • Password, Public Key, Keyboard-Interactive         │
│ • Certificate-based                                   │
├──────────────────────────────────────────────────────┤
│ SSH Transport Layer Protocol (RFC 4253)               │
│ • Key exchange (DH, ECDH)                            │
│ • Server authentication (host keys)                  │
│ • Encryption, MAC, compression                       │
├──────────────────────────────────────────────────────┤
│ TCP (Port 22)                                         │
└──────────────────────────────────────────────────────┘
```

### 2.2 Connection Establishment Flow

```
Client                                          Server
  │                                               │
  │──── TCP SYN ──────────────────────────────→  │ 1. TCP handshake
  │←─── TCP SYN-ACK ─────────────────────────── │
  │──── TCP ACK ──────────────────────────────→  │
  │                                               │
  │←─── "SSH-2.0-OpenSSH_9.6" ────────────────  │ 2. Version exchange
  │──── "SSH-2.0-OpenSSH_9.6" ────────────────→ │
  │                                               │
  │←──→ Key Exchange Init (KEXINIT) ←──→       │ 3. Algorithm negotiation
  │     (lists of supported algorithms)          │
  │                                               │
  │←──→ Diffie-Hellman Exchange ←──→            │ 4. Key exchange
  │     (generate shared secret)                  │
  │                                               │
  │←─── NEWKEYS ─────────────────────────────── │ 5. Switch to encryption
  │──── NEWKEYS ──────────────────────────────→ │
  │                                               │
  │═══════ Encrypted from here ═══════════════  │
  │                                               │
  │──── Service request (ssh-userauth) ───────→ │ 6. Authentication
  │←─── Auth methods available ────────────────  │
  │──── Auth request (publickey/password) ────→ │
  │←─── Auth success ─────────────────────────── │
  │                                               │
  │──── Channel open (session) ────────────────→ │ 7. Open channel
  │←─── Channel open confirm ──────────────────  │
  │──── Channel request (shell/exec) ─────────→ │
  │                                               │
  │═══════ Interactive session ═══════════════  │
```

### 2.3 Algorithm Negotiation

Client và server gửi danh sách algorithms hỗ trợ, chọn cái đầu tiên match:

```
Client KEXINIT:
  kex_algorithms: curve25519-sha256,diffie-hellman-group16-sha512,...
  server_host_key_algorithms: ssh-ed25519,rsa-sha2-512,...
  encryption_client_to_server: chacha20-poly1305@openssh.com,aes256-gcm,...
  encryption_server_to_client: chacha20-poly1305@openssh.com,aes256-gcm,...
  mac_client_to_server: hmac-sha2-256-etm@openssh.com,...
  mac_server_to_client: hmac-sha2-256-etm@openssh.com,...
  compression: none,zlib@openssh.com

Server KEXINIT:
  kex_algorithms: curve25519-sha256,diffie-hellman-group16-sha512,...
  ...

Negotiation result (first match):
  KEX: curve25519-sha256
  Host key: ssh-ed25519
  Cipher: chacha20-poly1305@openssh.com
  MAC: (implicit in AEAD cipher)
  Compression: none
```

---

## 3. Key Exchange — DH và ECDH

### 3.1 Vấn đề: Làm sao 2 bên có cùng key mà không ai nghe được?

**Ví dụ đời thường — Trộn màu sơn (Diffie-Hellman):**

Imagine Alice và Bob muốn tạo một màu sơn bí mật chung. Eve đang nghe lén.

```
1. Alice và Bob thỏa thuận: "Dùng màu VÀNG làm base" (public, Eve biết)

2. Alice chọn bí mật: Đỏ (Alice's private)
   Alice trộn: Vàng + Đỏ = CAM → gửi cho Bob (Eve thấy CAM)
   
3. Bob chọn bí mật: Xanh (Bob's private)
   Bob trộn: Vàng + Xanh = XANH LÁ → gửi cho Alice (Eve thấy XANH LÁ)

4. Alice nhận Xanh Lá, trộn với bí mật Đỏ: Xanh Lá + Đỏ = NÂU (shared secret!)
   Bob nhận Cam, trộn với bí mật Xanh: Cam + Xanh = NÂU (same shared secret!)

5. Eve thấy: Vàng, Cam, Xanh Lá — nhưng KHÔNG THỂ tạo ra NÂU!
   (Không thể "tách" màu sau khi trộn — đó là one-way function)
```

### 3.2 Diffie-Hellman Key Exchange (Toán học)

```
Thỏa thuận công khai:
  p = số nguyên tố lớn (2048-4096 bit)
  g = generator (primitive root mod p)

Alice:
  a = random secret (private key)
  A = g^a mod p → gửi cho Bob (public)

Bob:
  b = random secret (private key)  
  B = g^b mod p → gửi cho Alice (public)

Shared Secret:
  Alice tính: K = B^a mod p = (g^b)^a mod p = g^(ab) mod p
  Bob tính:   K = A^b mod p = (g^a)^b mod p = g^(ab) mod p
  → K giống nhau! ✅

Eve biết: p, g, A, B
Eve cần tính: a (hoặc b) từ A = g^a mod p
→ Đây là "Discrete Logarithm Problem" — cực khó với p đủ lớn
```

### 3.3 ECDH (Elliptic Curve Diffie-Hellman)

**ECDH** dùng đường cong elliptic thay vì modular arithmetic:
- **Nhỏ hơn**: 256-bit ECDH = bảo mật tương đương 3072-bit DH
- **Nhanh hơn**: Ít computation hơn
- **Curve25519**: Curve phổ biến nhất cho SSH (an toàn, nhanh)

```
ECDH with Curve25519:
  Curve: y² = x³ + 486662x² + x (mod p, p = 2²⁵⁵ - 19)
  G = base point trên curve

Alice:
  a = random 256-bit secret
  A = a·G (point multiplication) → gửi Bob

Bob:
  b = random 256-bit secret
  B = b·G → gửi Alice

Shared Secret:
  Alice: K = a·B = a·(b·G) = (ab)·G
  Bob:   K = b·A = b·(a·G) = (ab)·G
  → K giống nhau! ✅
```

### 3.4 SSH Key Exchange trong thực tế

```
SSH uses "curve25519-sha256" (RFC 8731):

1. Client generates ephemeral keypair: (a, Q_C = a·G)
2. Server generates ephemeral keypair: (b, Q_S = b·G)
3. Client sends Q_C
4. Server sends Q_S + host key + signature
5. Both compute: K = a·Q_S = b·Q_C = (ab)·G

6. Derive session keys from K:
   H = SHA-256(V_C || V_S || I_C || I_S || K_S || Q_C || Q_S || K)
   
   Where:
   V_C, V_S = version strings
   I_C, I_S = KEXINIT payloads
   K_S = server host key
   
   From H and K, derive:
   - Encryption key client→server
   - Encryption key server→client
   - MAC key client→server
   - MAC key server→client
   - IV client→server
   - IV server→client
```

### 3.5 Perfect Forward Secrecy (PFS)

**PFS** nghĩa là: nếu private key bị lộ trong **tương lai**, các session **quá khứ** vẫn an toàn.

**Tại sao?** Vì mỗi session dùng ephemeral (tạm thời) DH keys:
- Session keys được tạo mới mỗi connection
- Ephemeral keys bị xóa sau session
- Kẻ tấn công có host key cũng KHÔNG thể giải mã traffic cũ

```
Session 1: ephemeral keys (a₁, b₁) → K₁ → [deleted]
Session 2: ephemeral keys (a₂, b₂) → K₂ → [deleted]
Session 3: ephemeral keys (a₃, b₃) → K₃ → [deleted]

Nếu host key bị lộ → K₁, K₂, K₃ vẫn an toàn (không thể tính lại)
```

---

## 4. Host Key Verification — Xác minh danh tính server

### 4.1 Trust On First Use (TOFU)

**Ví dụ đời thường**: Lần đầu tiên bạn gặp ai đó, bạn ghi nhớ khuôn mặt họ. Lần sau gặp lại, bạn nhận ra "đúng người này". Nhưng lần đầu, bạn phải **tin tưởng** họ là ai họ nói.

```bash
$ ssh user@newserver.example.com
The authenticity of host 'newserver.example.com (203.0.113.10)' can't be established.
ED25519 key fingerprint is SHA256:AbCdEf1234567890AbCdEf1234567890AbCdEf12.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'newserver.example.com' (ED25519) to the list of known hosts.
```

**Lần đầu (TOFU)**:
1. Server gửi public host key
2. Client hiển thị fingerprint
3. User xác nhận (hoặc verify qua kênh khác)
4. Client lưu vào `~/.ssh/known_hosts`

**Các lần sau**:
1. Server gửi public host key
2. Client so sánh với `~/.ssh/known_hosts`
3. Match → continue. Mismatch → **WARNING!**

### 4.2 MITM Attack Detection

```bash
$ ssh user@server.example.com
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ED25519 key sent by the remote host is
SHA256:XxYyZz9876543210...
Please contact your system administrator.
Add correct host key in /Users/you/.ssh/known_hosts to get rid of this message.
Offending key in /Users/you/.ssh/known_hosts:42
Host key verification failed.
```

**Nguyên nhân có thể:**
- ⚠️ **MITM attack** — kẻ tấn công xen giữa
- ✅ Server reinstalled (key mới)
- ✅ Server IP thay đổi (shared IP/cloud)
- ✅ DNS hijacking

### 4.3 known_hosts Format

```bash
# ~/.ssh/known_hosts
# Format: hostname/IP algorithm base64-public-key

server.example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
192.168.1.100 ssh-rsa AAAAB3NzaC1yc2EAAAA...

# Hashed format (bảo vệ danh sách servers):
|1|salt=|hash= ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA...
```

```bash
# Quản lý known_hosts
ssh-keygen -F server.example.com      # Tìm host key
ssh-keygen -R server.example.com      # Xóa host key (khi server thay đổi hợp lệ)
ssh-keygen -H                          # Hash all entries (privacy)
ssh-keyscan -H server.example.com >> ~/.ssh/known_hosts  # Thêm key mới
```

### 4.4 SSH Certificate Authority (thay TOFU)

Thay vì TOFU, dùng **CA** để ký host keys (giống TLS certificates):

```bash
# 1. Tạo CA key
ssh-keygen -t ed25519 -f ca_host_key -C "Host CA"

# 2. Ký host key của server
ssh-keygen -s ca_host_key -I server01 -h -n server.example.com \
  /etc/ssh/ssh_host_ed25519_key.pub

# 3. Client tin tưởng CA (thay vì từng host)
# ~/.ssh/known_hosts:
@cert-authority *.example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5... (CA public key)

# → Client tự động trust mọi server có cert ký bởi CA này
# → Không bao giờ thấy "authenticity can't be established" nữa!
```

---

## 5. Authentication Methods — Các phương thức xác thực

### 5.1 Overview

| Method | An toàn | Convenience | Use Case |
|---|---|---|---|
| **Password** | ⚠️ Trung bình | ✅ Đơn giản | Quick access, testing |
| **Public Key** | ✅✅ Cao | ✅ Sau khi setup | Daily use, automation |
| **Certificate** | ✅✅✅ Rất cao | ✅ | Enterprise, large fleet |
| **Keyboard-Interactive** | ⚠️ Depends | ⚠️ | MFA, TOTP |
| **GSSAPI/Kerberos** | ✅ | ✅ SSO | Corporate (Active Directory) |

### 5.2 Password Authentication

```
Client → Server: "Auth with password: myP@ssw0rd"
Server: Verify against /etc/shadow (or PAM)
Server → Client: "Auth success" or "Auth failure"
```

**Nhược điểm**:
- Brute-force attacks (bots thử millions passwords)
- Phishing (fake SSH server captures password)
- Keylogger (malware ghi password)
- Không automate được (script cần password)

### 5.3 Public Key Authentication (Phổ biến nhất)

**Ví dụ đời thường**: Thay vì dùng mật khẩu (ai cũng có thể đoán), bạn dùng **ổ khóa + chìa khóa**. Bạn đặt ổ khóa (public key) trên server. Chỉ ai có chìa khóa (private key) mới mở được.

```
Setup:
1. ssh-keygen → tạo cặp key (id_ed25519 + id_ed25519.pub)
2. Copy public key lên server: ~/.ssh/authorized_keys
3. Private key giữ ở client (KHÔNG BAO GIỜ share)

Authentication flow:
Client                                    Server
  │                                         │
  │── "Auth publickey, ssh-ed25519" ──────→ │
  │                                         │ Kiểm tra authorized_keys
  │←── "OK, prove you have private key" ── │
  │                                         │
  │── Sign(session_id, ...) ──────────────→ │ Client ký bằng private key
  │                                         │ Server verify bằng public key
  │←── "Auth success" ─────────────────── │
```

```bash
# Generate key pair
ssh-keygen -t ed25519 -C "your_email@example.com"
# → ~/.ssh/id_ed25519 (private) + ~/.ssh/id_ed25519.pub (public)

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server

# Or manually:
cat ~/.ssh/id_ed25519.pub | ssh user@server "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### 5.4 Key Types và Recommendations

| Type | Command | Key Size | Security | Speed | Recommendation |
|---|---|---|---|---|---|
| **Ed25519** | `ssh-keygen -t ed25519` | 256 bit | ✅✅✅ | Fast | ✅ **Khuyến khích** |
| **ECDSA** | `ssh-keygen -t ecdsa -b 521` | 256/384/521 | ✅✅ | Fast | ⚠️ OK |
| **RSA** | `ssh-keygen -t rsa -b 4096` | 2048-4096 | ✅✅ (4096) | Slow | ⚠️ Legacy |
| **DSA** | N/A | 1024 | ❌ Weak | Fast | ❌ **Deprecated** |

### 5.5 authorized_keys Options

```bash
# ~/.ssh/authorized_keys trên server

# Basic
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... user@laptop

# Restricted (chỉ cho phép 1 command)
command="mysqldump mydb" ssh-ed25519 AAAAC3...  # Chỉ chạy backup

# Multiple restrictions
from="10.0.0.0/24",no-port-forwarding,no-agent-forwarding,no-X11-forwarding ssh-ed25519 AAAAC3...

# Options:
# command="cmd"        → Chỉ chạy command này
# from="IP/CIDR"      → Chỉ chấp nhận từ IP này
# no-port-forwarding  → Cấm tunnel
# no-agent-forwarding → Cấm agent forwarding
# no-X11-forwarding   → Cấm X11
# no-pty              → Không cấp terminal (chỉ command)
# expiry-time="date"  → Key hết hạn
# environment="VAR=value" → Set env variable
```

---

## 6. Local Port Forwarding — Đường hầm tới server xa

### 6.1 Khái niệm

**Local Port Forwarding** = Mở port trên máy LOCAL, traffic qua SSH tunnel tới REMOTE service.

**Ví dụ đời thường**: Bạn ở nhà, muốn truy cập database trong mạng công ty. Database không expose ra internet. Nhưng bạn có SSH access vào server jump (bastion). Bạn tạo "đường hầm" qua bastion tới database.

```
Your Laptop (HOME)              Bastion Server           Database
(localhost:5433) ═══SSH Tunnel═══ (bastion) ──────────── (db:5432)

Bạn connect tới localhost:5433 → traffic đi qua SSH → tới db:5432
```

### 6.2 Syntax và Ví dụ

```bash
# Syntax:
ssh -L [local_addr:]local_port:remote_host:remote_port user@ssh_server

# Ví dụ 1: Access database qua bastion
ssh -L 5433:db-server.internal:5432 user@bastion.example.com
# → psql -h localhost -p 5433 (connects to db-server.internal:5432)

# Ví dụ 2: Access internal web app
ssh -L 8080:internal-app.corp:80 user@bastion.example.com
# → Mở browser: http://localhost:8080

# Ví dụ 3: Access Redis trong VPC
ssh -L 6380:redis.internal:6379 user@bastion.example.com
# → redis-cli -h localhost -p 6380

# Ví dụ 4: Multiple tunnels
ssh -L 5433:db:5432 -L 6380:redis:6379 -L 8080:web:80 user@bastion

# Background (không mở shell):
ssh -f -N -L 5433:db:5432 user@bastion
# -f: Background
# -N: No remote command (chỉ tunnel)
```

### 6.3 Diagram chi tiết

```
┌────────────────┐        ┌──────────────────┐        ┌────────────────┐
│  Your Laptop   │        │  Bastion Server   │        │  DB Server     │
│                │        │                    │        │                │
│  App connects  │ SSH    │  SSH receives     │  TCP   │  Receives      │
│  to localhost  │ tunnel │  forwards to      │  conn  │  connection    │
│  port 5433     │═══════→│  db:5432          │───────→│  on port 5432  │
│                │ (encrypted)                 │(plain) │                │
│  psql -h       │        │                    │        │  PostgreSQL    │
│  localhost     │        │                    │        │                │
│  -p 5433       │        │                    │        │                │
└────────────────┘        └──────────────────┘        └────────────────┘
     HOME                      COMPANY NETWORK              COMPANY NETWORK
```

---

## 7. Remote Port Forwarding — Mở cổng ngược

### 7.1 Khái niệm

**Remote Port Forwarding** = Mở port trên REMOTE server, traffic đi ngược về LOCAL machine.

**Ví dụ đời thường**: Bạn chạy web app trên laptop (localhost:3000). Bạn muốn đồng nghiệp trên internet test thử. Bạn tạo tunnel ngược: mở port trên server public, traffic đi ngược về laptop bạn.

```
Your Laptop (behind NAT)     Public Server              Colleague
(localhost:3000) ←══SSH Tunnel══ (server:8080) ←──────── (browser)

Đồng nghiệp access server:8080 → traffic đi qua SSH → tới localhost:3000
```

### 7.2 Syntax và Ví dụ

```bash
# Syntax:
ssh -R [remote_addr:]remote_port:local_host:local_port user@ssh_server

# Ví dụ 1: Expose local web app ra internet
ssh -R 8080:localhost:3000 user@public-server.com
# → Truy cập http://public-server.com:8080 → tới localhost:3000

# Ví dụ 2: Expose local API
ssh -R 0.0.0.0:9090:localhost:8080 user@vps.example.com
# 0.0.0.0 = listen on all interfaces (cần GatewayPorts yes trong sshd_config)

# Ví dụ 3: Webhook testing (expose local service cho webhook callbacks)
ssh -R 4443:localhost:4443 user@webhook-receiver.com
```

### 7.3 Server Configuration

```bash
# /etc/ssh/sshd_config trên server
GatewayPorts yes          # Cho phép -R bind trên 0.0.0.0 (mọi interface)
GatewayPorts clientspecified  # Client chỉ định address
# Mặc định: GatewayPorts no (chỉ bind localhost)
```

---

## 8. Dynamic Port Forwarding — SOCKS Proxy

### 8.1 Khái niệm

**Dynamic Port Forwarding** = Tạo SOCKS proxy trên máy local. Mọi traffic qua proxy đều đi qua SSH tunnel.

**Ví dụ đời thường**: Thay vì tạo tunnel riêng cho từng service (port 5432, port 6379, port 80...), bạn tạo **MỘT proxy** — mọi traffic đều đi qua proxy đó. Giống thuê xe riêng thay vì mua vé từng chuyến bus.

### 8.2 Syntax và Usage

```bash
# Tạo SOCKS5 proxy trên localhost:1080
ssh -D 1080 user@bastion.example.com

# Background mode
ssh -f -N -D 1080 user@bastion.example.com
```

```
┌────────────────┐        ┌──────────────────┐        ┌────────────────┐
│  Your Laptop   │        │  Bastion Server   │        │  ANY internal  │
│                │        │                    │        │  service       │
│  Browser/App   │ SOCKS5 │  SSH decrypts,    │  TCP   │                │
│  configured to │ proxy  │  connects to      │  conn  │  db:5432       │
│  use localhost │═══════→│  destination      │───────→│  web:80        │
│  :1080         │ (encrypted)                 │(direct)│  redis:6379    │
│                │        │                    │        │  anything!     │
└────────────────┘        └──────────────────┘        └────────────────┘
```

```bash
# Configure browser/apps to use SOCKS proxy:
# Firefox: Settings → Network → SOCKS Host: localhost, Port: 1080

# curl with SOCKS proxy:
curl --socks5-hostname localhost:1080 http://internal-app.corp

# Or via environment:
export ALL_PROXY=socks5h://localhost:1080
curl http://internal-app.corp
```

### 8.3 So sánh ba loại forwarding

| | Local (-L) | Remote (-R) | Dynamic (-D) |
|---|---|---|---|
| Port mở ở | Client (local) | Server (remote) | Client (SOCKS proxy) |
| Traffic direction | Local → Remote | Remote → Local | Local → Any (qua server) |
| Use case | Access remote service | Expose local service | Browse as if on remote |
| Flexibility | 1 tunnel = 1 destination | 1 tunnel = 1 destination | 1 proxy = mọi destination |
| Protocol | TCP | TCP | SOCKS5 (any TCP) |

---

## 9. SSH Agent Forwarding — Chuyển tiếp xác thực

### 9.1 Vấn đề

**Scenario**: Bạn SSH vào Bastion (dùng key), rồi từ Bastion muốn SSH tiếp vào Production server. Nhưng private key nằm trên laptop, KHÔNG nằm trên Bastion.

```
Laptop (có key) → Bastion → Production
                   ↑
                   Không có key ở đây!
```

**Giải pháp tệ**: Copy private key lên Bastion → ❌ NGUY HIỂM (bastion bị hack = lộ key)

**Giải pháp đúng**: **SSH Agent Forwarding** — để laptop "ký hộ" mà KHÔNG cần copy key

### 9.2 SSH Agent là gì?

**SSH Agent** (ssh-agent) là daemon chạy trên máy local, giữ private keys trong memory (decrypted). Khi cần ký, application hỏi agent thay vì đọc key file trực tiếp.

```bash
# Khởi động agent
eval $(ssh-agent -s)

# Thêm key vào agent
ssh-add ~/.ssh/id_ed25519
# → Nhập passphrase 1 lần, agent nhớ key

# Liệt kê keys trong agent
ssh-add -l
# 256 SHA256:abc... user@laptop (ED25519)

# macOS: Dùng Keychain (nhớ qua restart)
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

### 9.3 Agent Forwarding Flow

```bash
# Kết nối với Agent Forwarding (-A flag)
ssh -A user@bastion.example.com

# Từ bastion, SSH tiếp vào production (KHÔNG cần key trên bastion)
bastion$ ssh user@production.internal
# → Hoạt động! Bastion hỏi laptop ký thay qua forwarded agent
```

```
┌─────────────┐         ┌──────────────┐         ┌──────────────┐
│   Laptop    │         │   Bastion     │         │  Production  │
│             │         │               │         │              │
│ [ssh-agent] │←═socket═┤ [agent socket]│         │              │
│  (has key)  │         │  (forwarded)  │         │              │
│             │   SSH   │               │   SSH   │              │
│             │═════════│               │═════════│              │
│             │         │               │         │              │
│ Sign request│←────────│ "Sign this"   │←────────│ Auth request │
│ with key    │────────→│ "Signature"   │────────→│ Verify sig   │
│             │         │               │         │ → Success!   │
└─────────────┘         └──────────────┘         └──────────────┘

Key NEVER leaves laptop!
```

### 9.4 Security Risk của Agent Forwarding

**Nguy hiểm**: Nếu bastion bị compromised (root access), attacker có thể dùng forwarded agent socket để ký bất kỳ auth request nào — SSH vào BẤT KỲ server nào mà key bạn có access.

```
Attacker trên bastion:
SSH_AUTH_SOCK=/tmp/ssh-xxx/agent.1234 ssh root@other-server
→ Thành công! (dùng forwarded agent của bạn)
```

**Mitigation:**
1. **ProxyJump thay Agent Forwarding** (an toàn hơn):
```bash
# Thay vì -A, dùng ProxyJump (-J):
ssh -J bastion.example.com user@production.internal

# Hoặc trong ~/.ssh/config:
Host production
    HostName production.internal
    ProxyJump bastion.example.com
    
# ProxyJump: laptop SSH trực tiếp tới production QUA bastion
# Bastion chỉ forward TCP traffic, KHÔNG access agent
```

2. **Confirm mỗi lần ký**: `ssh-add -c ~/.ssh/id_ed25519` (yêu cầu confirm)
3. **Chỉ forward khi cần**: Không dùng `-A` mặc định

### 9.5 ~/.ssh/config — Configuration File

```bash
# ~/.ssh/config — SSH client configuration

# Default settings cho tất cả hosts
Host *
    AddKeysToAgent yes           # Tự thêm key vào agent khi dùng
    IdentitiesOnly yes           # Chỉ dùng key chỉ định (không try tất cả)
    ServerAliveInterval 60       # Gửi keepalive mỗi 60s
    ServerAliveCountMax 3        # Disconnect sau 3 lần không reply

# Bastion/Jump server
Host bastion
    HostName bastion.example.com
    User admin
    IdentityFile ~/.ssh/id_ed25519_work
    
# Production servers (qua bastion)
Host prod-*
    ProxyJump bastion
    User deploy
    IdentityFile ~/.ssh/id_ed25519_deploy

Host prod-web
    HostName 10.0.1.50
    
Host prod-db
    HostName 10.0.2.100

# Dev server (direct access)
Host dev
    HostName dev.example.com
    User developer
    LocalForward 5433 localhost:5432    # Auto tunnel PostgreSQL
    LocalForward 6380 localhost:6379    # Auto tunnel Redis

# GitHub
Host github.com
    IdentityFile ~/.ssh/id_ed25519_github
    
# Multiple SSH keys cho cùng host
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work
```

---

## 10. Tổng kết và Best Practices

### 10.1 SSH Security Hardening

```bash
# /etc/ssh/sshd_config — Server hardening

# Authentication
PermitRootLogin no                    # Cấm root login
PasswordAuthentication no             # Chỉ dùng key
PubkeyAuthentication yes
AuthenticationMethods publickey       # Hoặc: publickey,keyboard-interactive (MFA)
MaxAuthTries 3
LoginGraceTime 30

# Access control
AllowUsers deploy admin               # Chỉ cho phép users cụ thể
AllowGroups ssh-users                  # Hoặc theo group
DenyUsers root

# Network
Port 2222                              # Đổi port (security through obscurity)
ListenAddress 10.0.0.5                 # Chỉ listen trên interface cụ thể
AddressFamily inet                     # Chỉ IPv4 (hoặc inet6, any)

# Forwarding
AllowTcpForwarding yes                 # Hoặc "local" / "no"
X11Forwarding no
AllowAgentForwarding no                # Tắt nếu không cần

# Algorithms (modern only)
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512

# Limits
MaxStartups 10:30:60                   # Rate limit connections
ClientAliveInterval 300                # Disconnect idle after 5min
ClientAliveCountMax 2
```

### 10.2 Tunneling Cheat Sheet

```bash
# LOCAL: Access remote DB from laptop
ssh -L 5433:db.internal:5432 user@bastion

# REMOTE: Expose local web to internet
ssh -R 8080:localhost:3000 user@public-vps

# DYNAMIC: SOCKS proxy qua bastion
ssh -D 1080 user@bastion

# PROXYJUMP: SSH qua bastion (safer than -A)
ssh -J bastion user@production

# Multiple hops:
ssh -J bastion1,bastion2 user@final-destination

# Keep tunnel alive in background:
ssh -f -N -L 5433:db:5432 user@bastion
# -f: fork to background
# -N: no remote command
```

### 10.3 Troubleshooting

```bash
# Debug SSH connection:
ssh -vvv user@server    # Verbose mode (3 levels)

# Common issues:
# 1. Permission denied (publickey)
#    → Check: ls -la ~/.ssh/id_ed25519 (phải 600)
#    → Check: ls -la ~/.ssh/authorized_keys trên server (phải 600)
#    → Check: ls -la ~/.ssh/ directory (phải 700)

# 2. Connection timeout
#    → Check: firewall rule, port 22 open?
#    → Try: ssh -p 443 user@server (nếu 22 bị block)

# 3. Host key verification failed
#    → ssh-keygen -R server.example.com (remove old key)
#    → Verify fingerprint qua another channel

# 4. Agent forwarding not working
#    → Check: echo $SSH_AUTH_SOCK (phải set)
#    → Check: ssh-add -l (key phải trong agent)
#    → Check: AllowAgentForwarding yes trong sshd_config

# 5. Tunnel not working
#    → Check: AllowTcpForwarding yes trong sshd_config
#    → Check: GatewayPorts yes (cho -R 0.0.0.0:...)
```

### 10.4 Tài liệu tham khảo

| Tài liệu | Nội dung |
|---|---|
| RFC 4251 | SSH Protocol Architecture |
| RFC 4252 | SSH Authentication Protocol |
| RFC 4253 | SSH Transport Layer Protocol |
| RFC 4254 | SSH Connection Protocol |
| RFC 4419 | DH Group Exchange for SSH |
| RFC 8731 | Curve25519 for SSH KEX |
| RFC 9142 | KEX Method Updates |
| OpenSSH manual | ssh(1), sshd_config(5), ssh_config(5) |

### 10.5 Câu hỏi ôn tập

1. SSH có mấy lớp? Vai trò mỗi lớp?
2. Diffie-Hellman key exchange hoạt động thế nào? Tại sao Eve không tính được shared secret?
3. Perfect Forward Secrecy nghĩa là gì? SSH đạt PFS bằng cách nào?
4. TOFU là gì? Rủi ro của nó? Giải pháp tốt hơn?
5. So sánh password auth vs public key auth. Tại sao nên tắt password?
6. Local forwarding (-L) vs Remote forwarding (-R): khác nhau thế nào? Cho ví dụ.
7. Dynamic forwarding (-D) làm gì? Khi nào dùng thay vì -L?
8. Agent forwarding nguy hiểm thế nào? ProxyJump an toàn hơn ra sao?
9. Liệt kê 5 hardening measures cho SSH server.
10. Ed25519 vs RSA: tại sao Ed25519 được khuyến khích?

---

*Bài viết được tham khảo từ RFC 4251-4254 (SSH Protocol Suite), RFC 8731 (Curve25519 KEX), RFC 9142 (KEX Updates), OpenSSH documentation, và NIST SP 800-57 (Key Management).*
