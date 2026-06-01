---
layout: post
title: "SSH Advanced Usage - Sử dụng SSH nâng cao"
date: 2026-06-01
categories: [linux]
tags: [ssh, tunneling, proxy, certificates, security]
---

## Mục lục
1. [Góc nhìn tổng quan - Đường hầm bí mật](#goc-nhin-tong-quan)
2. [~/.ssh/config - Sổ địa chỉ thông minh](#ssh-config)
3. [ProxyJump - Nhảy qua trạm trung gian](#proxyjump)
4. [LocalForward - Đường hầm từ máy mình](#localforward)
5. [RemoteForward - Đường hầm ngược](#remoteforward)
6. [DynamicForward - SOCKS proxy toàn năng](#dynamicforward)
7. [ControlMaster - Tiết kiệm kết nối](#controlmaster)
8. [SSH Certificates - Chứng chỉ thay vì key](#ssh-certificates)
9. [SSHFP - DNS xác thực host key](#sshfp)
10. [Tổng kết và best practices](#tong-ket)

---

## 1. Góc nhìn tổng quan - Đường hầm bí mật {#goc-nhin-tong-quan}

### Ví dụ đời thường

Hãy tưởng tượng SSH là một **hệ thống đường hầm ngầm** trong thành phố:

- **SSH cơ bản** = đường hầm từ nhà bạn đến văn phòng (kết nối trực tiếp, mã hóa)
- **ProxyJump** = đi qua trạm trung chuyển metro để đến ga xa (bastion host)
- **LocalForward** = kéo ống nước từ giếng ở văn phòng về nhà (truy cập service xa từ local)
- **RemoteForward** = chia sẻ WiFi nhà bạn cho văn phòng (expose service local ra remote)
- **DynamicForward** = xây cổng kết nối đa năng, đi được nhiều nơi (SOCKS proxy)
- **ControlMaster** = giữ đường hầm mở, ai cần đi qua cứ đi (connection sharing)
- **SSH Certificates** = thẻ nhân viên có dấu công ty, thay vì phải ghi nhớ từng người (CA-signed keys)

### Tại sao cần SSH nâng cao?

```
Tình huống thực tế:
1. Bạn cần truy cập database server nằm sau firewall
   → LocalForward (tunnel port 5432 về localhost)

2. Bạn có 50 servers, mỗi server phải thêm SSH key mới
   → SSH Certificates (chỉ trust CA, không cần distribute key)

3. Bạn SSH đến 10 servers mỗi ngày qua bastion
   → ProxyJump + ControlMaster (1 connection, auto hop)

4. Team member ở nhà cần test webhook
   → RemoteForward (expose local port ra server có public IP)

5. Bạn cần browse internal wiki từ café
   → DynamicForward (SOCKS proxy qua SSH tunnel)
```

---

## 2. ~/.ssh/config - Sổ địa chỉ thông minh {#ssh-config}

### ~/.ssh/config là gì?

File config SSH giống **danh bạ điện thoại** kèm ghi chú: thay vì nhớ IP, port, user, key cho mỗi server, bạn ghi vào config rồi chỉ cần `ssh tên`.

### Cấu trúc cơ bản

```bash
# ~/.ssh/config

# Cài đặt chung cho tất cả hosts
Host *
    ServerAliveInterval 60       # Gửi keepalive mỗi 60s
    ServerAliveCountMax 3        # Disconnect sau 3 lần không phản hồi
    AddKeysToAgent yes           # Tự thêm key vào agent
    IdentitiesOnly yes           # Chỉ dùng key specified

# Server cụ thể
Host web-prod
    HostName 10.0.1.50
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_prod_ed25519

Host db-staging
    HostName 10.0.2.30
    User admin
    IdentityFile ~/.ssh/id_staging
    LocalForward 5432 localhost:5432

# Pattern matching
Host *.dev.company.com
    User developer
    IdentityFile ~/.ssh/id_dev
    ProxyJump bastion-dev

Host bastion-*
    User jump
    IdentityFile ~/.ssh/id_bastion
    ForwardAgent no              # NEVER forward agent to bastion!
```

### Sử dụng

```bash
# Thay vì:
ssh -i ~/.ssh/id_prod_ed25519 -p 2222 deploy@10.0.1.50

# Chỉ cần:
ssh web-prod

# SCP cũng dùng được:
scp file.tar.gz web-prod:/opt/app/

# rsync cũng được:
rsync -avz ./dist/ web-prod:/var/www/html/
```

### Config directives quan trọng

```bash
# Connection
Host name              # Alias khi ssh
HostName               # IP hoặc hostname thật
User                   # Username
Port                   # Port (mặc định 22)
IdentityFile           # Private key path

# Security
PubkeyAuthentication yes
PasswordAuthentication no
PreferredAuthentications publickey

# Performance
Compression yes        # Nén data (tốt cho slow link)
TCPKeepAlive yes

# Multiplexing
ControlMaster auto
ControlPath ~/.ssh/sockets/%r@%h-%p
ControlPersist 600     # Giữ connection 10 phút

# Agent
ForwardAgent no        # Mặc định NO (security risk!)
AddKeysToAgent yes
```

---

## 3. ProxyJump - Nhảy qua trạm trung gian {#proxyjump}

### ProxyJump là gì?

ProxyJump cho phép SSH qua 1 hoặc nhiều "jump hosts" (bastion/trạm trung gian) để đến server đích. Server đích không cần expose port 22 ra internet.

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  Laptop  │────▶│  Bastion     │────▶│  Private     │
│  (bạn)   │ SSH │  (jump host) │ SSH │  Server      │
└──────────┘     └──────────────┘     └──────────────┘
   Internet          DMZ                Private Network
```

### Cú pháp

```bash
# Command line
ssh -J bastion.example.com private-server

# Multiple jumps (chuỗi)
ssh -J jump1.com,jump2.com target-server

# Với user và port khác
ssh -J user1@jump:2222 user2@target:22

# Trong ~/.ssh/config (RECOMMENDED)
Host private-server
    HostName 10.0.1.50
    User admin
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User jump
    IdentityFile ~/.ssh/id_bastion
```

### So sánh với ProxyCommand (cách cũ)

```bash
# Cách cũ (ProxyCommand) - vẫn hoạt động nhưng phức tạp hơn
Host private-server
    ProxyCommand ssh -W %h:%p bastion

# Cách mới (ProxyJump) - đơn giản, rõ ràng hơn
Host private-server
    ProxyJump bastion

# ProxyJump ưu điểm:
# - Syntax đơn giản hơn
# - Hỗ trợ chain nhiều hops
# - Không cần netcat/nc trên bastion
```

### Multi-hop example

```bash
# Kiến trúc: Laptop → Bastion → App Server → Database
Host bastion
    HostName 203.0.113.10
    User jump

Host app-server
    HostName 10.0.1.20
    User deploy
    ProxyJump bastion

Host db-server
    HostName 10.0.2.30
    User dba
    ProxyJump app-server    # 2 hops: bastion → app → db

# Hoặc explicit chain:
Host db-server
    HostName 10.0.2.30
    ProxyJump bastion,app-server
```

---

## 4. LocalForward - Đường hầm từ máy mình {#localforward}

### LocalForward là gì?

LocalForward mở port trên **máy local** của bạn, forward traffic qua SSH tunnel đến service ở remote network. 

Giống **kéo ống nước** từ giếng (remote service) về nhà (local machine).

```
┌─────────────────┐          ┌─────────────────┐          ┌──────────────┐
│  Local Machine  │   SSH    │  SSH Server     │  direct  │  Target      │
│                 │ tunnel   │  (jump host)    │  access  │  Service     │
│  localhost:5432 │─────────▶│                 │─────────▶│  db:5432     │
└─────────────────┘          └─────────────────┘          └──────────────┘
  Browser/pgAdmin              bastion.com                   10.0.2.30
  connects here                                             (private IP)
```

### Cú pháp

```bash
# Command line:
# ssh -L [bind_address:]local_port:target_host:target_port remote_host

# Ví dụ 1: Forward PostgreSQL
ssh -L 5432:10.0.2.30:5432 bastion.example.com
# Giờ: psql -h localhost -p 5432 → kết nối đến db server!

# Ví dụ 2: Forward web service
ssh -L 8080:internal-wiki.corp:80 bastion.example.com
# Giờ: browser mở http://localhost:8080 → internal wiki!

# Ví dụ 3: Forward nhiều ports
ssh -L 5432:db:5432 -L 6379:redis:6379 -L 8080:api:8080 bastion

# Background mode (không cần shell)
ssh -fNL 5432:db:5432 bastion
# -f : background
# -N : no command (chỉ tunnel)

# Trong ~/.ssh/config:
Host bastion-tunnel
    HostName bastion.example.com
    User tunnel
    LocalForward 5432 10.0.2.30:5432
    LocalForward 6379 10.0.2.31:6379
    LocalForward 9200 10.0.2.32:9200
```

### Use cases phổ biến

```bash
# 1. Database access (dev kết nối staging/prod DB an toàn)
ssh -L 5432:prod-db.internal:5432 bastion
pgAdmin connects to localhost:5432

# 2. Kubernetes API
ssh -L 6443:k8s-master.internal:6443 bastion
kubectl --server=https://localhost:6443

# 3. Internal monitoring (Grafana, Kibana)
ssh -L 3000:grafana.internal:3000 bastion
Browser: http://localhost:3000

# 4. SMTP relay (send email through internal server)
ssh -L 2525:smtp.internal:25 bastion
```

---

## 5. RemoteForward - Đường hầm ngược {#remoteforward}

### RemoteForward là gì?

RemoteForward là **ngược lại** LocalForward: mở port trên **remote server**, forward traffic về service trên **local machine** của bạn.

Giống **chia sẻ WiFi nhà bạn cho văn phòng** - người ở văn phòng truy cập được service ở nhà bạn.

```
┌─────────────────┐          ┌─────────────────┐
│  Local Machine  │   SSH    │  Remote Server  │
│                 │ tunnel   │                 │
│  localhost:3000 │◀─────────│  0.0.0.0:8080   │
│  (your app)     │          │  (public port)  │
└─────────────────┘          └─────────────────┘
  Your dev server              cloud-server.com
  (behind NAT)                 (public IP)
```

### Cú pháp

```bash
# ssh -R [bind_address:]remote_port:target_host:target_port remote_host

# Ví dụ 1: Expose local web app ra internet
ssh -R 8080:localhost:3000 cloud-server.com
# Giờ: http://cloud-server.com:8080 → local app:3000

# Ví dụ 2: Webhook testing (nhận webhook ở local)
ssh -R 9000:localhost:9000 server-with-public-ip
# Configure webhook URL: http://server-with-public-ip:9000

# Background mode
ssh -fNR 8080:localhost:3000 cloud-server.com
```

### Cấu hình server cho RemoteForward

```bash
# Trên SSH server, cần enable trong /etc/ssh/sshd_config:
GatewayPorts yes         # Cho phép bind 0.0.0.0 (ai cũng truy cập được)
# HOẶC
GatewayPorts clientspecified  # Client chọn bind address

# Nếu GatewayPorts = no (mặc định):
# RemoteForward chỉ bind 127.0.0.1 trên server
# (chỉ process trên server mới access được)
```

### Security warning

```
⚠️ RemoteForward expose service ra network!
- Luôn cân nhắc: ai có thể truy cập port đó?
- Dùng firewall trên remote server để giới hạn
- Sử dụng authentication cho service được expose
- Không dùng cho production!
```

---

## 6. DynamicForward - SOCKS proxy toàn năng {#dynamicforward}

### DynamicForward là gì?

DynamicForward tạo **SOCKS proxy** trên local machine. Bất kỳ application nào support SOCKS đều có thể route traffic qua SSH tunnel. Giống **VPN nhẹ** - browse internet AS IF bạn đang ở server.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Browser     │     │  SSH Client  │ SSH │  SSH Server  │────▶ Internet
│  (SOCKS      │────▶│  SOCKS proxy │────▶│              │────▶ Internal
│   proxy)     │     │  :1080       │     │              │      Network
└──────────────┘     └──────────────┘     └──────────────┘
```

### Cú pháp

```bash
# ssh -D [bind_address:]port remote_host
ssh -D 1080 server.example.com

# Background mode
ssh -fND 1080 server.example.com

# Trong config:
Host socks-proxy
    HostName server.example.com
    DynamicForward 1080
```

### Sử dụng SOCKS proxy

```bash
# Browser: Settings → Proxy → SOCKS5 → 127.0.0.1:1080

# curl qua SOCKS
curl --socks5 127.0.0.1:1080 http://internal-service.corp/api

# Git qua SOCKS
git config --global http.proxy socks5://127.0.0.1:1080

# Environment variable
export ALL_PROXY=socks5://127.0.0.1:1080

# Chromium với proxy riêng
chromium --proxy-server="socks5://127.0.0.1:1080"
```

### Use cases

```
1. Truy cập internal websites từ bên ngoài (như VPN)
2. Browse internet từ IP khác (testing geo-restriction)
3. Bypass firewall restrictive (ví dụ: hotel WiFi block ports)
4. Security research (route traffic qua trusted network)
```

---

## 7. ControlMaster - Tiết kiệm kết nối {#controlmaster}

### ControlMaster là gì?

ControlMaster cho phép **nhiều SSH sessions dùng chung 1 TCP connection**. Session đầu tiên tạo connection (master), các session sau "multiplexing" qua cùng connection đó.

Giống **carpool** (đi chung xe): thay vì mỗi người 1 xe (1 connection), nhiều người đi chung.

### Lợi ích

```
Không có ControlMaster:
  ssh server (connect → handshake → auth) = 2-5 giây
  ssh server (connect → handshake → auth) = 2-5 giây
  ssh server (connect → handshake → auth) = 2-5 giây

Với ControlMaster:
  ssh server (connect → handshake → auth) = 2-5 giây (master)
  ssh server (reuse existing connection) = 0.1 giây!
  ssh server (reuse existing connection) = 0.1 giây!
  scp file server: (reuse) = 0.1 giây start!
```

### Cấu hình

```bash
# ~/.ssh/config
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600          # Giữ connection 10 phút sau khi session cuối đóng

# Tạo thư mục socket
mkdir -p ~/.ssh/sockets
chmod 700 ~/.ssh/sockets
```

### Giải thích options

```bash
# ControlMaster:
#   auto   = Tự động: tạo master nếu chưa có, reuse nếu có
#   yes    = Bắt buộc là master (session đầu tiên)
#   no     = Không dùng multiplexing

# ControlPath:
#   %r = remote username
#   %h = remote hostname
#   %p = remote port
#   Socket file cho mỗi connection unique

# ControlPersist:
#   0      = Đóng master khi session cuối logout
#   600    = Giữ master thêm 600 giây sau session cuối
#   yes    = Giữ mãi mãi (cần manually kill)
```

### Quản lý connections

```bash
# Kiểm tra status
ssh -O check server
# Master running (pid=12345)

# Đóng master connection
ssh -O stop server
# OR
ssh -O exit server

# Bypass multiplexing (force new connection)
ssh -o ControlMaster=no server
# Hoặc:
ssh -S none server
```

---

## 8. SSH Certificates - Chứng chỉ thay vì key {#ssh-certificates}

### Vấn đề với SSH keys truyền thống

```
Hệ thống truyền thống:
- 100 servers × 50 developers = 5000 authorized_keys entries
- Developer mới: phải thêm key vào 100 servers
- Developer nghỉ: phải XÓA key khỏi 100 servers
- Key bị compromise: phải rotate trên 100 servers
- Không có expiry date! Key valid mãi mãi

→ Quản lý nightmare!
```

### SSH Certificates giải quyết thế nào?

```
Hệ thống Certificate Authority (CA):
- 1 CA key ký tất cả user certificates
- Servers chỉ cần trust CA (1 dòng config)
- User được cấp certificate có expiry date
- Revoke: thêm serial vào revocation list
- Onboarding: ký cert mới (1 lệnh)
- Offboarding: cert tự expire hoặc revoke

Giống như:
  Truyền thống = mỗi cửa hàng phải biết mặt từng khách VIP
  Certificate  = khách có thẻ công ty (thẻ do HR cấp, cửa hàng chỉ cần verify logo)
```

### Thiết lập SSH CA

```bash
# === STEP 1: Tạo CA key pair ===
# User CA (ký certificate cho users)
ssh-keygen -t ed25519 -f /etc/ssh/ca_user_key -C "User CA"

# Host CA (ký certificate cho servers)
ssh-keygen -t ed25519 -f /etc/ssh/ca_host_key -C "Host CA"

# === STEP 2: Ký User Certificate ===
# Developer gửi public key → Admin ký cert
ssh-keygen -s /etc/ssh/ca_user_key \
  -I "john.doe@company" \
  -n john,deploy \
  -V +52w \
  -z 1001 \
  /path/to/john_id_ed25519.pub

# Giải thích:
# -s : CA private key (signing key)
# -I : Identity (tên hiển thị trong log)
# -n : Principals (usernames được phép login)
# -V : Validity (+52w = 52 tuần = 1 năm)
# -z : Serial number (cho revocation)

# Output: john_id_ed25519-cert.pub

# === STEP 3: Ký Host Certificate ===
ssh-keygen -s /etc/ssh/ca_host_key \
  -I "web-prod-01.example.com" \
  -h \
  -n web-prod-01.example.com,10.0.1.50 \
  -V +52w \
  /etc/ssh/ssh_host_ed25519_key.pub

# -h : Host certificate (không phải user)
# -n : Hostnames/IPs valid cho cert này

# === STEP 4: Cấu hình Server trust User CA ===
# /etc/ssh/sshd_config:
TrustedUserCAKeys /etc/ssh/ca_user_key.pub
# Không cần authorized_keys nữa!

# === STEP 5: Cấu hình Client trust Host CA ===
# ~/.ssh/known_hosts (hoặc /etc/ssh/ssh_known_hosts):
@cert-authority *.example.com ssh-ed25519 AAAA...
# Không còn "Are you sure you want to continue connecting?" prompt!
```

### Xem thông tin certificate

```bash
ssh-keygen -L -f john_id_ed25519-cert.pub
# Output:
#   Type: ssh-ed25519-cert-v01@openssh.com user certificate
#   Public key: ED25519-CERT SHA256:xxx
#   Signing CA: ED25519 SHA256:yyy
#   Key ID: "john.doe@company"
#   Serial: 1001
#   Valid: from 2024-01-15 to 2025-01-15
#   Principals: john, deploy
#   Critical Options: (none)
#   Extensions: permit-pty, permit-user-rc
```

### Certificate restrictions

```bash
# Giới hạn certificate chỉ cho phép:
ssh-keygen -s ca_key \
  -I "backup-bot" \
  -n backup \
  -V +30d \
  -O force-command="/usr/bin/rsync --server" \
  -O source-address="10.0.0.0/8" \
  -O no-port-forwarding \
  -O no-pty \
  bot_key.pub

# Options:
# force-command   = Chỉ được chạy command này
# source-address  = Chỉ từ IP range này
# no-port-forwarding = Không tunnel
# no-pty          = Không interactive shell
```

---

## 9. SSHFP - DNS xác thực host key {#sshfp}

### Vấn đề TOFU

```
Lần đầu SSH đến server mới:
"The authenticity of host 'example.com' can't be established.
ED25519 key fingerprint is SHA256:AbCdEf...
Are you sure you want to continue connecting (yes/no)?"

Đây gọi là TOFU - Trust On First Use.
Bạn có kiểm tra fingerprint không? Hầu hết: NO.
→ Vulnerable to Man-in-the-Middle attack!
```

### SSHFP là gì?

**SSHFP** (SSH Fingerprint) là DNS record chứa fingerprint của SSH host key. Client tự động verify host key bằng cách tra cứu DNS (phải có DNSSEC).

RFC 4255: Using DNS to Securely Publish Secure Shell (SSH) Key Fingerprints.

### Thiết lập SSHFP

```bash
# Tạo SSHFP records từ host keys
ssh-keygen -r example.com
# Output (thêm vào DNS zone file):
# example.com IN SSHFP 1 1 abc123...  (RSA, SHA-1)
# example.com IN SSHFP 1 2 def456...  (RSA, SHA-256)
# example.com IN SSHFP 4 2 ghi789...  (Ed25519, SHA-256)

# Record format: hostname IN SSHFP algorithm hash_type fingerprint
# Algorithm: 1=RSA, 3=ECDSA, 4=Ed25519
# Hash type: 1=SHA-1, 2=SHA-256
```

### Sử dụng SSHFP ở client

```bash
# Verify host key qua DNS
ssh -o VerifyHostKeyDNS=yes server.example.com

# Trong config:
Host *.example.com
    VerifyHostKeyDNS yes

# Kết quả khi connect:
# "Matching host key fingerprint found in DNS."
# → Tự động accept, không hỏi

# QUAN TRỌNG: Cần DNSSEC!
# Nếu DNS không có DNSSEC, SSHFP vẫn có thể bị spoof.
# VerifyHostKeyDNS=ask → Hiện match/mismatch nhưng vẫn hỏi
```

### Kiểm tra SSHFP record

```bash
# Dig SSHFP records
dig +short SSHFP example.com
# 4 2 abc123def456...

# Verify DNSSEC
dig +dnssec SSHFP example.com
# Look for: flags: ... ad (Authenticated Data)
```

---

## 10. Tổng kết và best practices {#tong-ket}

### Security Best Practices

```bash
# 1. Key types: Ưu tiên Ed25519
ssh-keygen -t ed25519 -C "email@example.com"
# Ed25519 > ECDSA > RSA-4096 >> RSA-2048

# 2. Passphrase: LUÔN đặt passphrase cho private key
# Dùng ssh-agent để cache passphrase

# 3. SSH Agent: Cẩn thận với ForwardAgent
Host *
    ForwardAgent no              # Default NO
Host trusted-bastion
    ForwardAgent yes             # Chỉ khi cần

# 4. Server hardening (/etc/ssh/sshd_config)
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers deploy admin
X11Forwarding no
AllowTcpForwarding yes         # Cần cho tunneling
Protocol 2

# 5. Ciphers: Chỉ dùng strong ciphers
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
KexAlgorithms curve25519-sha256
MACs hmac-sha2-256-etm@openssh.com
HostKeyAlgorithms ssh-ed25519
```

### Bảng tổng hợp Tunneling

| Feature | Command | Use Case |
|---------|---------|----------|
| LocalForward | -L 8080:target:80 | Truy cập internal service |
| RemoteForward | -R 8080:localhost:3000 | Expose local app |
| DynamicForward | -D 1080 | SOCKS proxy (browse as remote) |
| ProxyJump | -J bastion | Qua jump host |
| ControlMaster | ControlMaster auto | Reuse connections |

### SSH config template hoàn chỉnh

```bash
# ~/.ssh/config - Production ready

# === GLOBAL DEFAULTS ===
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600
    AddKeysToAgent yes
    IdentitiesOnly yes
    ForwardAgent no
    HashKnownHosts yes
    VerifyHostKeyDNS yes

# === BASTION / JUMP HOSTS ===
Host bastion-prod
    HostName bastion.prod.company.com
    User jump
    IdentityFile ~/.ssh/id_bastion_ed25519
    ControlPersist 3600

# === APPLICATION SERVERS ===
Host app-prod-*
    User deploy
    ProxyJump bastion-prod
    IdentityFile ~/.ssh/id_deploy_ed25519

Host app-prod-01
    HostName 10.0.1.11
Host app-prod-02
    HostName 10.0.1.12

# === DATABASE TUNNELS ===
Host db-tunnel-prod
    HostName bastion.prod.company.com
    User tunnel
    IdentityFile ~/.ssh/id_tunnel
    LocalForward 5432 prod-db.internal:5432
    LocalForward 6379 prod-redis.internal:6379
    RequestTTY no
    ExitOnForwardFailure yes

# === DEVELOPMENT ===
Host dev-proxy
    HostName dev-bastion.company.com
    User dev
    DynamicForward 1080
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| OpenSSH Manual Pages (man ssh, ssh_config) | Tham chiếu chính thức |
| RFC 4253: SSH Transport Layer Protocol | Protocol specification |
| RFC 4255: SSHFP DNS Records | SSHFP standard |
| SSH Certificates (FB Engineering Blog) | Facebook's CA implementation |
| Smallstep SSH Certificates Guide | Practical CA setup guide |
| Mozilla SSH Guidelines | Security recommendations |

---

*Bài viết tiếp theo: [Log Management](/2026/08/10/log-management/) - Quản lý log hệ thống*
