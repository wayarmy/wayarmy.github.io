---
layout: post
title:  "AWS Learnign Path"
subtitle: AWS Learning Path - Mapping with IT Foundation
gh-repo: wayarmy/wayarmy.github.io
tags: [sysadmin, developer, devops, cloud]
comments: true
date:   2026-05-31
categories: Sysadmin
---

**AWS Learning Path - Mapping with IT Foundation**

# 📘 IT FOUNDATION CHI TIẾT + TIMELINE TUẦN CỤ THỂ

> Phụ lục cho Lộ trình AWS — Deep dive từng topic + Lịch học ngày/tuần**Dành cho:** Người mới, đã có kiến thức coding cơ bản (biết lập trình, functions, logic)**Giả định:** Học 2-3 giờ/ngày, 5-6 ngày/tuần**Triết lý:** Mỗi concept cần thời gian "ngấm" — không nhồi nhét, ưu tiên hiểu sâu hơn học nhanh

---

## 📑 MỤC LỤC

1. [IT Foundation Deep Dive (12 tuần)](#-ph%E1%BA%A7n-1-it-foundation-deep-dive)
2. [Weekly Timeline toàn bộ lộ trình (20-22 tháng)](#-ph%E1%BA%A7n-2-weekly-timeline-chi-ti%E1%BA%BFt)

---

# 🏗️ PHẦN 1: IT FOUNDATION DEEP DIVE

> ⚠️ **Lưu ý cho người có background coding:**Bạn đã có tư duy logic, biết viết code — đó là lợi thế lớn. Tuy nhiên, Networking và Linux là hai thế giới hoàn toàn khác so với application programming. Đừng nóng vội — những concept như subnetting, BGP, hay systemd cần thời gian thực hành lặp đi lặp lại mới "ngấm" được.

## Thứ tự học (đã điều chỉnh):

```
┌─────────────────────────────────────────────────────────────────┐
│  1. NETWORKING (Tuần 1-5)     ← Quan trọng nhất, cần nhiều     │
│                                  thời gian nhất để "ngấm"       │
│  2. LINUX & OS (Tuần 6-8)     ← Bạn đã biết coding nên sẽ     │
│                                  quen CLI nhanh hơn             │
│  3. CONTAINERS & DATA (Tuần 9-10) ← Liên quan tới coding       │
│  4. SECURITY (Tuần 11-12)     ← Để cuối vì cần hiểu            │
│                                  networking + Linux trước       │
└─────────────────────────────────────────────────────────────────┘

```

**Tại sao Security để cuối?**

- Encryption/TLS cần hiểu TCP/IP trước (TLS chạy trên TCP layer)
- Firewall rules cần hiểu IP/Ports trước
- PKI/Certificates cần hiểu DNS và HTTP trước
- IAM/Auth concepts dễ hiểu hơn khi đã dùng Linux users/permissions

---

## 🟢 TUẦN 1-5: NETWORKING FUNDAMENTALS (Giãn từ 3 → 5 tuần)

> 💡 **Tại sao 5 tuần?** Networking là thứ trừu tượng nhất đối với developer. Bạn quen với code (input → process → output) nhưng networking là "invisible" — data chạy qua dây, qua không khí. Cần thời gian hình dung và thực hành nhiều lần.

### Tuần 1: Hiểu cách máy tính "nói chuyện" với nhau

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Tại sao cần Networking? | Analogy: Networking = hệ thống bưu điện (địa chỉ, routing, đóng gói). Lịch sử Internet (ARPANET → TCP/IP). Tại sao developer cần biết networking | Xem video: "How the Internet Works" (5 min) — code.org |
| **T3** | OSI Model (phần 1) | Layer 1-4: Physical (cáp, sóng), Data Link (MAC address, switch), Network (IP, router), Transport (TCP/UDP, ports). Mỗi layer làm gì? | Vẽ sơ đồ 4 layer đầu, ghi chú chức năng |
| **T4** | OSI Model (phần 2) | Layer 5-7: Session, Presentation, Application. So sánh OSI vs TCP/IP model (4 layers). Data encapsulation: Data → Segment → Packet → Frame → Bits | Trace 1 HTTP request qua từng layer (trên giấy) |
| **T5** | Thực hành quan sát | Cài Wireshark. Capture traffic khi mở website. Xem packet structure thực tế. Identify layers trong mỗi packet | Capture & identify: Ethernet header, IP header, TCP header, HTTP data |
| **CN** | Review + Consolidate | Xem lại tất cả. Tự giải thích OSI model bằng lời. Nếu chưa rõ → xem lại video | Viết tóm tắt 1 trang bằng lời của mình |

> 🧘 **Ngày nghỉ:** Thứ 6 & Thứ 7 — để não xử lý thông tin. Networking cần thời gian "ngấm", đừng cố nhồi.

📝 Checkpoint Tuần 1:

- [ ] Giải thích được tại sao cần chia layers
- [ ] Phân biệt được switch (Layer 2) vs router (Layer 3)
- [ ] Hiểu data encapsulation qua hình ảnh

---

### Tuần 2: IP Addressing & Subnetting (Chậm & Kỹ)

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | IP Address là gì? | IPv4: 32-bit, chia 4 octets. Binary ↔ Decimal conversion. Tại sao cần IP? (So sánh: IP = địa chỉ nhà, MAC = số CMND) | Chuyển 10 IP addresses sang binary và ngược lại |
| **T3** | Network vs Host | Subnet mask mục đích: phân biệt phần network và phần host. AND operation. VD: 192.168.1.100/24 → Network: 192.168.1.0, Host: .100 | Xác định network ID cho 10 IP/mask combinations |
| **T4** | Private vs Public IP | Private ranges: 10.x.x.x, 172.16-31.x.x, 192.168.x.x. Tại sao cần private IP? NAT preview. Loopback 127.0.0.1. **AWS:** VPC chỉ dùng private IP ranges | List private ranges, identify public vs private cho 20 IPs |
| **T5** | CIDR Notation (phần 1) | /8, /16, /24, /28, /32 nghĩa là gì? Cách tính số IP: 2^(32-prefix). Usable = total - 2 (network + broadcast). **AWS:** VPC = /16→/28 | Tính số IPs cho /16, /20, /24, /26, /28 |
| **CN** | CIDR Practice | Luyện tập tính CIDR. 20 bài tập. | subnettingpractice.com — mode dễ |

📝 Checkpoint Tuần 2:

- [ ] Convert binary ↔ decimal nhanh
- [ ] Tính số IP từ CIDR prefix
- [ ] Biết tại sao AWS VPC cần CIDR block

---

### Tuần 3: Subnetting Nâng Cao + Routing

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Subnetting thực hành | Chia subnet: lấy 10.0.0.0/16, chia thành 4 subnets bằng nhau → /18. Chia không bằng nhau (VLSM). Tại sao cần chia? (isolation, security, management) | Bài tập: Design 10.0.0.0/16 cho 3 AZs × 2 tiers |
| **T3** | VPC Design exercise | **AWS Context:** 1 VPC /16 = 65,536 IPs. Chia 6 subnets: 3 public (/24) + 3 private (/20). Tại sao private lớn hơn? (nhiều instances hơn) | Vẽ VPC diagram với CIDR cho từng subnet |
| **T4** | Routing cơ bản | Default gateway = "cửa ra" của subnet. Routing table: destination + next hop. Static routes. Longest prefix match. **AWS:** mỗi subnet có 1 route table | Đọc route table Linux (`ip route`), identify default GW |
| **T5** | Routing trong AWS | VPC Route Table: local route (auto), IGW route (0.0.0.0/0 → igw). Public subnet vs Private subnet = khác nhau ở route table! | Vẽ diagram: Public subnet (→IGW) vs Private (→NAT GW) |
| **CN** | Review Networking tuần 1-3 | Ôn lại tất cả. Làm 30 bài subnetting timed. Vẽ lại VPC diagram từ đầu | subnettingpractice.com — 30 bài, target < 1 phút/bài |

📝 Checkpoint Tuần 3:

- [ ] Chia subnet cho VPC design trong < 5 phút
- [ ] Hiểu routing table và longest prefix match
- [ ] Phân biệt public vs private subnet qua route table

---

### Tuần 4: DNS, HTTP, Protocols

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | DNS (phần 1) | DNS = "danh bạ điện thoại" của internet. Domain → IP. Hierarchy: Root (.) → TLD (.com) → Domain (google.com). Record types: A, CNAME | `nslookup google.com`, `dig example.com` |
| **T3** | DNS (phần 2) | Recursive vs Iterative resolution. DNS caching, TTL. More records: AAAA (IPv6), MX (mail), NS, TXT, SRV. **AWS:** Route 53 = managed DNS | Trace DNS resolution: browser → resolver → root → TLD → authoritative |
| **T4** | HTTP/HTTPS | HTTP methods (GET/POST/PUT/DELETE). Status codes (200, 301, 404, 500). Headers. Request/Response cycle. **Bạn biết REST APIs rồi** — giờ hiểu layer dưới nó | `curl -v` a website, read headers line by line |
| **T5** | TCP vs UDP & Ports | TCP: reliable (handshake, ACK, retransmit). UDP: fast (no guarantee). Well-known ports: 22, 80, 443, 53, 3306. Ephemeral ports. **AWS:** Security Groups filter by port | `ss -tuln` (Linux) hoặc `netstat -an` — identify listening ports |
| **CN** | NAT | Tại sao cần NAT? Private IP không routable trên internet. SNAT (outbound), DNAT (inbound). PAT (many-to-one). **AWS:** NAT Gateway = managed SNAT | Diagram: Private EC2 (10.0.1.5) → NAT GW (public IP) → Internet |

📝 Checkpoint Tuần 4:

- [ ] Trace DNS resolution flow đầy đủ
- [ ] Hiểu tại sao NAT Gateway cần cho private subnets
- [ ] Thuộc 15+ common ports

---

### Tuần 5: Firewall, VPN, Load Balancing, BGP (Overview)

> 💡 Tuần này là "breadth over depth" — hiểu concept, không cần master. Sẽ gặp lại chi tiết khi học AWS.

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Firewalls | Stateful: nhớ connection, auto-allow return traffic. Stateless: check mỗi packet riêng lẻ. Rules: allow/deny + protocol + port + source/dest. **AWS:** SG (stateful) vs NACL (stateless) | Viết 5 firewall rules cho web server (allow 80, 443, deny others) |
| **T3** | VPN basics | VPN = encrypted tunnel qua internet. IPSec. Site-to-Site (office ↔ AWS). Client VPN (laptop → AWS). **AWS:** VPN Gateway, Customer Gateway | Diagram: Corporate DC ↔ VPN ↔ AWS VPC |
| **T4** | Load Balancing | Tại sao cần LB? (HA, scaling, no single point of failure). Algorithms: Round-robin, Least connections. Health checks. Layer 4 vs Layer 7. **AWS:** ALB (HTTP), NLB (TCP) | Diagram: Users → ALB → 3 EC2 instances |
| **T5** | BGP overview | BGP = protocol để routers "trao đổi đường đi". AS (Autonomous System). Chỉ cần hiểu concept — sẽ quay lại khi học Direct Connect. **AWS:** BGP dùng cho Direct Connect | Xem video: "BGP explained in 5 min" — ghi note chính |
| **CN** | 🏆 NETWORKING CAPSTONE | **Project:** Vẽ complete architecture: Internet → CloudFront → ALB → EC2 (private subnet) → RDS (private subnet). Ghi chú CIDR, route tables, SG rules | Hoàn thành diagram + submit checkpoint |

📝 Checkpoint Tuần 5 (NETWORKING COMPLETE):

- [ ] Vẽ được full VPC architecture từ đầu
- [ ] Hiểu data flow: User → DNS → CDN → LB → App → DB
- [ ] Tự tin giải thích networking concepts cho người khác

---

## 🟡 TUẦN 6-8: LINUX & OS (Giữ nguyên 3 tuần — developer học CLI nhanh hơn)

> 💡 **Lợi thế developer:** Bạn đã quen terminal (có thể đã dùng để chạy code). Linux CLI sẽ familiar hơn networking. Tuy nhiên, system administration concepts (systemd, permissions model, process management) vẫn cần thời gian.

### Tuần 6: Linux CLI & File System

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Setup + Navigation | Install Ubuntu (VM/WSL2/Docker). Terminal basics. `pwd`, `ls -la`, `cd`, `mkdir`, `rmdir`, `touch`. Absolute vs relative paths. Hidden files (`.`) | Tạo project directory structure bằng CLI |
| **T3** | File Operations | `cp`, `mv`, `rm -rf`, `cat`, `less`, `head`, `tail -f`, `wc`. Wildcards: `*`, `?`, `[]`. Hard links vs Soft links | Chain 5+ commands cùng lúc |
| **T4** | I/O Redirection & Pipes | `>` (overwrite), `>>` (append), `<` (input), `2>` (stderr), `&>` (all). Pipes ` | `. `tee`. `xargs`. **Developer analogy:** Pipes = chaining functions |
| **T5** | Permissions | `rwx` = read/write/execute. User/Group/Others. Numeric: 755, 644, 600. `chmod`, `chown`. `umask`. Why this matters: EC2 key file needs 600! | Fix "Permission denied" scenarios (5 exercises) |
| **CN** | Users, Groups & sudo | `/etc/passwd`, `/etc/shadow`. `useradd`, `usermod`. Groups. `sudo` vs `su`. **AWS analogy:** Linux users:groups ≈ IAM users:groups | Create 3 users, 2 groups, assign permissions |

---

### Tuần 7: System Administration & Networking CLI

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Package Management | APT: `apt update`, `apt install`, `apt remove`, `apt search`. YUM/DNF (Amazon Linux). Repository concepts. Dependencies | Install nginx + node.js, verify both running |
| **T3** | Process & Service Management | `ps aux`, `top`/`htop`, `kill -9`, `kill -15`. `systemctl start/stop/enable/status`. `journalctl`. Service auto-start on boot | Start nginx, check status, view logs, restart |
| **T4** | Networking CLI | `ip addr`, `ip route`, `ss -tuln`, `ping`, `traceroute`, `curl`, `wget`, `dig`. `/etc/hosts`, `/etc/resolv.conf`. Troubleshooting flow | Troubleshoot: "EC2 can't reach internet" — 5 step debug |
| **T5** | Text Processing | `grep -r` (recursive search), `sed 's/old/new/g'`, `awk '{print $1}'`, `sort`, `uniq`, `cut -d',' -f2`. Regex basics: `.`, `*`, `+`, `[]`, `^`, `$` | Parse CSV file, extract specific columns, count patterns |
| **CN** | Cron & Scheduled Tasks | `crontab -e`, cron syntax (`* * * * *` = min hour day month weekday). `@reboot`, `@daily`. **AWS:** EventBridge = cloud cron | Schedule script to run every 5 minutes, verify in log |

---

### Tuần 8: Bash Scripting & Automation

> 💡 **Developer advantage:** Bạn đã biết logic (if/else, loops, functions). Bash syntax hơi "ugly" so với Python nhưng concepts giống nhau.

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Bash Basics | `#!/bin/bash`. Variables (no spaces!): `NAME="hello"`. `$1 $2 $@` args. `echo`, `read`. Exit codes `$?` (0=success). Quoting: single vs double | Write script: greet user by name, check args |
| **T3** | Conditionals & Loops | `if [ condition ]; then ... fi`. Test operators: `-f` (file exists), `-d` (dir), `-eq`, `-gt`, `-z` (empty string). `for i in list`, `while read line` | Write script: check if file exists, process each line |
| **T4** | Functions & Error Handling | `function_name() { ... }`. Local variables. Return values. `set -e` (exit on error), `set -o pipefail`, `trap` (cleanup). | Write modular script with functions + error handling |
| **T5** | Practical: EC2 User Data | User Data = bash script that runs on first boot. Install packages, configure app, start services. Real-world pattern on AWS | Write User Data: install Docker, pull image, run container |
| **CN** | 🏆 LINUX CAPSTONE | **Project:** Comprehensive script: 1) Check OS type, 2) Install deps, 3) Create user, 4) Download app, 5) Configure + start service, 6) Health check | Complete automation script (like real EC2 setup) |

📝 Checkpoint Tuần 8 (LINUX COMPLETE):

- [ ] Navigate + manage Linux system confidently
- [ ] Write bash scripts with proper error handling
- [ ] Troubleshoot networking issues from CLI
- [ ] Write production-ready EC2 User Data

---

## 🟤 TUẦN 9-10: VIRTUALIZATION, CONTAINERS & DATABASE

> 💡 Phần này gần với thế giới developer — Docker, containers, DB sẽ quen thuộc hơn.

### Tuần 9: Virtualization & Containers

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Virtualization concepts | Hypervisor Type 1 (bare metal: Xen, KVM, Nitro) vs Type 2 (VirtualBox, VMware). VM = full OS copy. vCPU, vRAM, virtual disk. Snapshots. **AWS:** EC2 = VM on Nitro hypervisor | Tạo VM trong VirtualBox (nếu chưa có), take snapshot, restore |
| **T3** | Docker (phần 1) | Container ≠ VM (shares kernel, lighter). Docker architecture: daemon, client, registry. Image (blueprint) vs Container (running instance). `docker run`, `docker ps`, `docker stop` | `docker run -d -p 80:80 nginx` → truy cập localhost:80 |
| **T4** | Docker (phần 2) | Dockerfile: `FROM`, `RUN`, `COPY`, `CMD`, `EXPOSE`, `ENV`, `WORKDIR`. Layer caching. `.dockerignore`. Build context. **AWS:** ECR = Docker Hub của AWS | Build Docker image cho 1 Python/Node app của bạn |
| **T5** | Docker Compose & Multi-container | `docker-compose.yml`: services, networks, volumes. Multi-container apps (app + db + cache). **AWS:** ECS Task Definition ≈ docker-compose | docker-compose: app (Flask/Express) + PostgreSQL + Redis |
| **CN** | Container Orchestration (overview) | Tại sao cần orchestration? (scaling, health check, load balance containers). K8s concepts: Pod, Service, Deployment. **AWS:** ECS vs EKS vs Fargate | Đọc + diagram: ECS cluster → Services → Tasks → Containers |

---

### Tuần 10: Database Concepts & Data

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Relational DB deep | ACID (Atomicity, Consistency, Isolation, Durability). Normalization (1NF, 2NF, 3NF). Indexes (B-tree). Transactions. **AWS:** RDS, Aurora | Design schema cho e-commerce (users, orders, products) |
| **T3** | SQL review & advanced | Joins (INNER, LEFT, RIGHT, FULL). Subqueries. Window functions (`ROW_NUMBER`, `RANK`). `GROUP BY` + `HAVING`. Performance: `EXPLAIN` | Write 10 queries on sample DB (SQLite hoặc PostgreSQL) |
| **T4** | NoSQL concepts | Key-Value (Redis, DynamoDB). Document (MongoDB). Column-family (Cassandra). Graph (Neo4j, Neptune). When to use which? CAP theorem | Design DynamoDB table: partition key, sort key, GSI |
| **T5** | Data Architecture overview | OLTP vs OLAP. Data Lake vs Data Warehouse. ETL/ELT. Streaming vs Batch. **AWS:** RDS(OLTP) → Glue(ETL) → Redshift(OLAP) or S3(Lake) | Diagram: Full data pipeline từ source → lake → warehouse |
| **CN** | IaC concepts | Infrastructure as Code: Declarative (what) vs Imperative (how). **AWS:** CloudFormation (YAML/JSON), CDK (Python/TS), Terraform (HCL). Version control infra | Đọc 1 CloudFormation template, understand structure |

📝 Checkpoint Tuần 10:

- [ ] Build & deploy Docker containers
- [ ] Design database schema (SQL + NoSQL)
- [ ] Hiểu data pipeline architecture
- [ ] Đọc được CloudFormation template

---

## 🔴 TUẦN 11-12: SECURITY FUNDAMENTALS (Để cuối — cần networking + Linux trước)

> 💡 **Tại sao cuối?** Bây giờ bạn đã hiểu:
> - TCP/IP, Ports, HTTP → hiểu TLS/firewall rules
> - Linux permissions → hiểu IAM permission model
> - Networking flow → hiểu defense in depth
> - Containers → hiểu container security

### Tuần 11: Encryption, TLS, PKI

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Encryption Basics | Symmetric (AES-256): 1 key, fast, dùng cho data at rest. Asymmetric (RSA): 2 keys (public encrypt, private decrypt), slow, dùng cho key exchange. **AWS:** KMS = managed symmetric keys | `openssl enc -aes-256-cbc` encrypt/decrypt file |
| **T3** | TLS/SSL | **Bạn đã hiểu TCP** — giờ thêm layer bảo mật. TLS Handshake: ClientHello → ServerHello → Certificate → Key Exchange → Encrypted data. Tại sao HTTPS cần certificate? | `openssl s_client -connect google.com:443` — đọc cert chain |
| **T4** | PKI & Certificates | Certificate Authority (CA) hierarchy: Root → Intermediate → Leaf. CSR process. Certificate = public key + identity + CA signature. **AWS:** ACM tự động manage certs | Tạo self-signed cert với openssl, hiểu từng field |
| **T5** | Hashing & Digital Signatures | Hash: SHA-256 (one-way, deterministic). Dùng cho: integrity check, password storage (+salt). Digital signature = hash encrypted with private key. Verify = decrypt with public key | Hash file, modify 1 byte, hash lại — thấy avalanche effect |
| **CN** | Encryption in AWS | KMS (symmetric, key policies). Envelope encryption (data key + master key). Server-Side Encryption (SSE-S3, SSE-KMS, SSE-C). Client-Side. **Bạn đã hiểu S3** → giờ thêm encryption layer | Map encryption types cho S3, EBS, RDS |

---

### Tuần 12: Authentication, Authorization & Security Models

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Authentication | Factors: Knowledge (password), Possession (phone/token), Inherence (biometric). MFA. Password hashing (bcrypt, argon2). Session vs Token-based auth | Decode a JWT: header.payload.signature — hiểu mỗi phần |
| **T3** | Authorization & Access Control | RBAC (Role-Based), ABAC (Attribute-Based). Policy evaluation logic. **AWS:** IAM = RBAC + ABAC. Policy JSON: Effect + Action + Resource + Condition | Viết 3 IAM policies: admin, readonly, specific-service |
| **T4** | Identity Federation | OAuth 2.0 (authorization). OIDC (authentication on top of OAuth). SAML 2.0 (enterprise SSO). **AWS:** Cognito (consumer), IAM Identity Center (enterprise) | Diagram: OAuth 2.0 flow cho "Login with Google" |
| **T5** | Security Architecture | Shared Responsibility Model. Defense in Depth: Edge (WAF, Shield) → Network (SG, NACL) → Host (OS hardening) → App (auth) → Data (encryption). Least privilege. Zero trust | Map all security layers cho 3-tier AWS app |
| **CN** | 🏆 SECURITY & IT FOUNDATION CAPSTONE | **Final Project:** Complete security design cho web app: IAM roles + VPC security + encryption at rest/transit + WAF + monitoring. **Final quiz: 50 câu all topics** | Document + quiz score |

📝 Checkpoint Tuần 12 (IT FOUNDATION COMPLETE):

- [ ] Explain encryption types and when to use each
- [ ] Understand TLS handshake (vì đã hiểu TCP)
- [ ] Write IAM policies
- [ ] Design defense-in-depth architecture
- [ ] **READY FOR AWS CLOUD PRACTITIONER** 🎉

---

# 📅 PHẦN 2: WEEKLY TIMELINE CHI TIẾT

> **Assumptions:** 2-3h/ngày, 5-6 ngày/tuần, bắt đầu từ tháng 6/2026**Tổng thời gian:** ~20-22 tháng (realistic cho người vừa học vừa làm)

## 🗓️ PHASE 0: IT FOUNDATION (Tuần 1-12) — 3 tháng

| Tuần | Dates (ước tính) | Focus | Deliverable |
| --- | --- | --- | --- |
| W1 | 01-07/06/2026 | Networking: OSI, TCP/IP, Wireshark | OSI diagram + first packet capture |
| W2 | 08-14/06/2026 | Networking: IP Addressing, CIDR, Subnetting | Subnet calculations confident |
| W3 | 15-21/06/2026 | Networking: Subnetting advanced, Routing | VPC CIDR design document |
| W4 | 22-28/06/2026 | Networking: DNS, HTTP, TCP/UDP, NAT | DNS trace + NAT diagram |
| W5 | 29/06-05/07/2026 | Networking: Firewalls, VPN, LB, BGP | Full network architecture diagram |
| W6 | 06-12/07/2026 | Linux: CLI, files, permissions, users | Confident CLI navigation |
| W7 | 13-19/07/2026 | Linux: Packages, services, networking CLI | Troubleshooting checklist |
| W8 | 20-26/07/2026 | Linux: Bash scripting, automation | EC2 User Data script |
| W9 | 27/07-02/08/2026 | Containers: Docker, Docker Compose | Dockerized app running |
| W10 | 03-09/08/2026 | Databases, Data Architecture, IaC concepts | DB schema + pipeline diagram |
| W11 | 10-16/08/2026 | Security: Encryption, TLS, PKI | Encryption mapping document |
| W12 | 17-23/08/2026 | Security: Auth, IAM, Security Architecture | **IT Foundation COMPLETE** ✅ |

---

## 🗓️ PHASE 1: CLOUD PRACTITIONER (Tuần 13-17) — 5 tuần (thêm 1 tuần buffer)

| Tuần | Focus | Hoạt động | Resources |
| --- | --- | --- | --- |
| W13 | Cloud Concepts | Cloud models (IaaS/PaaS/SaaS), AWS Global Infrastructure, Well-Architected | AWS Skill Builder (free) |
| W14 | Core Services (Compute & Storage) | EC2, Lambda, S3, EBS — concepts + console exploration | AWS Free Tier hands-on |
| W15 | Core Services (DB, Network, Security) | RDS, VPC, IAM, CloudWatch — connect to Foundation knowledge | Skill Builder modules |
| W16 | Pricing, Support, Review | Pricing models, Support plans, Billing tools, AWS Organizations | Whitepaper: AWS Overview |
| W17 | **EXAM PREP** | 3 practice exams (65Q each). Review wrong answers. Book exam | **🎯 THI CLF-C02** |

---

## 🗓️ PHASE 2A: SOLUTIONS ARCHITECT ASSOCIATE (Tuần 18-31) — 14 tuần

| Tuần | Focus | Hoạt động |
| --- | --- | --- |
| W18 | IAM Deep | Users, Groups, Roles, Policies (JSON). MFA. STS. Cross-account. Permission boundaries |
| W19 | EC2 Deep | Instance families, Nitro, placement groups, ENI, User Data. Instance Store vs EBS |
| W20 | Storage: EBS, EFS, FSx | Volume types (gp3, io2, st1, sc1). Snapshots. EFS modes. FSx for Windows/Lustre |
| W21 | S3 Deep | Storage classes, lifecycle, versioning, replication (CRR/SRR), encryption, access points |
| W22 | ELB & Auto Scaling | ALB (host/path routing) vs NLB (TCP/TLS) vs GWLB. ASG: launch templates, policies |
| W23 | Database Services | RDS Multi-AZ, Read Replicas. Aurora Global/Serverless. ElastiCache (Redis vs Memcached) |
| W24 | DynamoDB & Non-relational | Partitions, WCU/RCU, on-demand. GSI/LSI. Streams. DAX. Design patterns |
| W25 | VPC Deep | Subnets, Route Tables, IGW, NAT GW, VPC Peering, Endpoints, PrivateLink, Flow Logs |
| W26 | Route 53 & CloudFront | Hosted zones, routing policies, health checks. CloudFront: distributions, OAC, Lambda@Edge |
| W27 | Serverless | Lambda (runtime, layers, concurrency). API Gateway. Step Functions. EventBridge |
| W28 | Decoupling & Integration | SQS (Standard/FIFO), SNS, EventBridge patterns. Kinesis overview. Microservices patterns |
| W29 | Security & Monitoring | KMS, Secrets Manager, ACM, WAF. CloudWatch, CloudTrail, Config, X-Ray |
| W30 | HA & DR + Well-Architected | Multi-AZ vs Multi-Region. DR strategies (RPO/RTO). 6 Pillars deep review |
| W31 | **EXAM PREP** | 4-5 full practice exams. Weak areas review. **🎯 THI SAA-C03** |

---

## 🗓️ PHASE 2B: SYSOPS ASSOCIATE (Tuần 32-43) — 12 tuần

| Tuần | Focus | Tài liệu |
| --- | --- | --- |
| W32-33 | CloudWatch Deep | Metrics, custom metrics, dashboards, Logs Insights, alarms, composite alarms, EventBridge |
| W34-35 | Systems Manager | SSM Agent, Session Manager, Run Command, Parameter Store, Patch Manager, Automation |
| W36-37 | Deployment & IaC | CloudFormation deep (stacks, nested, changesets, drift). Elastic Beanstalk. OpsWorks |
| W38-39 | Networking Operations | VPC troubleshooting, Reachability Analyzer, Traffic Mirroring. SG/NACL audit |
| W40-41 | Cost & Performance | Cost Explorer, Budgets, Savings Plans. Compute Optimizer, Trusted Advisor. Tagging strategy |
| W42 | Account & Compliance | AWS Config rules, conformance packs. Organizations, SSO. Backup strategies |
| W43 | **EXAM PREP** | SOA.pdf dump (133 pages) + Tutorials Dojo. **🎯 THI SOA-C02** |

---

## 🗓️ PHASE 3: SA PROFESSIONAL (Tuần 44-62) — 19 tuần (dãn thêm vì complex)

| Tuần | Focus | Notes |
| --- | --- | --- |
| W44-46 | Multi-Account Strategy | Organizations (SCPs, OUs), Control Tower, cross-account roles, RAM, delegated admin |
| W47-50 | Advanced Networking | Transit Gateway (deep), Direct Connect (dedicated/hosted, VIFs, LAG), VPN, hybrid DNS, PrivateLink at scale |
| W51-53 | Migration & Modernization | 6Rs, AWS MGN, DMS/SCT, DataSync, Transfer Family, Snow Family, large-scale migration planning |
| W54-56 | Advanced Architecture | Event-driven, CQRS, saga, multi-region active-active, disaster recovery at enterprise scale |
| W57-59 | Cost Governance & Operational Excellence | RI/SP optimization, Spot fleet, Security Hub multi-account, compliance at scale |
| W60-61 | Review & Weak Areas | Re-study weak domains. Read AWS whitepapers: Migration, Well-Architected, Security |
| W62 | **EXAM PREP** | SAP.pdf (196 pages) + SAP Q&A PDFs. **🎯 THI SAP-C02** |

---

## 🗓️ PHASE 4: SPECIALTY (Tuần 63+)

### Option A: Security Specialty (Tuần 63-75)

| Tuần | Focus |
| --- | --- |
| W63-65 | IAM Advanced + Identity Federation (SAML, OIDC, Cognito, IAM Identity Center) |
| W66-68 | Data Protection: KMS deep, CloudHSM, S3 encryption patterns, secrets management |
| W69-71 | Infrastructure Security: Network Firewall, WAF advanced, Shield Advanced, VPC security |
| W72-73 | Detection & Response: GuardDuty, SecurityHub, Detective, automated remediation |
| W74-75 | **EXAM PREP:** SCS.pdf (158 pages). **🎯 THI SCS-C02** |

### Option B: Advanced Networking (Tuần 63-77)

| Tuần | Focus |
| --- | --- |
| W63-66 | Transit Gateway deep: route tables, attachments, multicast, inter-region peering |
| W67-70 | Direct Connect deep: connections, all VIF types, LAG, resiliency designs |
| W71-73 | Hybrid DNS + CloudFront advanced + Global Accelerator + Network Firewall |
| W74-76 | VPC Lattice, load balancing edge cases, IPv6, complex routing scenarios |
| W77 | **EXAM PREP:** ANS.pdf (162 pages). **🎯 THI ANS-C01** |

### Option C: Data Engineer (Tuần 63-75)

| Tuần | Focus |
| --- | --- |
| W63-65 | S3 data lake: partitioning strategies, file formats (Parquet/ORC/Avro), lifecycle |
| W66-68 | Glue deep: ETL jobs (PySpark), crawlers, schema registry, Data Quality, bookmarks |
| W69-71 | Kinesis + MSK streaming. Redshift deep: distribution styles, sort keys, Spectrum, ML |
| W72-73 | Lake Formation, Athena optimization, MWAA/Step Functions orchestration |
| W74-75 | **EXAM PREP:** DEA.pdf (141 pages). **🎯 THI DEA-C01** |

### Option D: ML/AI Track (Tuần 63-82)

| Tuần | Focus |
| --- | --- |
| W63-66 | ML Fundamentals: Statistics, Linear Algebra, ML algorithms, bias-variance, evaluation |
| W67-70 | SageMaker: training, endpoints, pipelines, feature store, experiments, debugger |
| W71-73 | AIF-C01: Bedrock, GenAI, responsible AI, prompt engineering. **🎯 THI AIF-C01** |
| W74-78 | MLA-C01: MLOps, CI/CD for ML, model monitoring, A/B testing, canary deployment |
| W79-80 | **EXAM PREP:** MLA.docx + MLA.pdf. **🎯 THI MLA-C01** |
| W81-82 | MLS-C01: advanced algorithms, NLP, CV, deep learning on AWS. **🎯 THI MLS-C01** |

---

## 📊 DAILY ROUTINE MẪU

### Ngày học thường (2-2.5h):

```
┌─────────────────────────────────────────────────────┐
│  🧠 20 phút — Review hôm trước (flashcards/notes)   │
│  📖 50 phút — Kiến thức mới (video hoặc docs)       │
│  ☕ 10 phút — Nghỉ ngắn (quan trọng!)               │
│  🛠️ 40 phút — Hands-on thực hành                    │
│  📝 10 phút — Ghi chú, tạo flashcard                │
│  ─────────────────────────────────────────           │
│  Total: ~2h 10 phút                                  │
└─────────────────────────────────────────────────────┘

```

### Ngày cuối tuần (review & consolidate, 1.5h):

```
┌─────────────────────────────────────────────────────┐
│  📋 30 phút — Review notes cả tuần                   │
│  🧩 30 phút — Mini quiz / practice questions         │
│  🔗 30 phút — Connect concepts (vẽ mind map)         │
│  ─────────────────────────────────────────           │
│  Total: ~1.5 giờ                                     │
└─────────────────────────────────────────────────────┘

```

### Ngày EXAM PREP (tuần cuối mỗi cert, 3h):

```
┌─────────────────────────────────────────────────────┐
│  ⏰ 15 phút — Quick review weak areas                │
│  📝 90 phút — Full practice exam (timed)             │
│  ☕ 10 phút — Break                                  │
│  📖 45 phút — Review EVERY wrong answer (WHY?)       │
│  📝 20 phút — Note patterns, traps, new concepts     │
│  ─────────────────────────────────────────           │
│  Total: ~3 giờ                                       │
└─────────────────────────────────────────────────────┘

```

---

## 🎯 MILESTONES & CHECKPOINTS

| Milestone | Tuần | Tháng (ước tính) | Verification |
| --- | --- | --- | --- |
| ✅ Networking Foundation | W5 | Tháng 7/2026 | Vẽ VPC architecture hoàn chỉnh |
| ✅ Linux + Containers | W10 | Tháng 8/2026 | Docker app + bash automation |
| ✅ IT Foundation COMPLETE | W12 | Cuối tháng 8/2026 | 50-question quiz, pass ≥80% |
| ✅ Cloud Practitioner PASS | W17 | Cuối tháng 9/2026 | **Cert #1** 🎉 |
| ✅ SA Associate PASS | W31 | Tháng 1/2027 | **Cert #2** — bắt đầu apply jobs |
| ✅ SysOps Associate PASS | W43 | Tháng 4/2027 | **Cert #3** — operational expertise |
| ✅ SA Professional PASS | W62 | Tháng 8/2027 | **Cert #4** — senior credential |
| ✅ First Specialty PASS | W75-82 | Cuối 2027/Đầu 2028 | **Cert #5** — domain expert |

---

## 💡 TIPS CHO NGƯỜI CÓ BACKGROUND CODING

### Advantages (tận dụng):

1. **Logic thinking** → Hiểu IAM policy evaluation, routing decisions nhanh hơn
2. **JSON/YAML familiar** → CloudFormation, IAM policies không xa lạ
3. **API understanding** → API Gateway, Lambda, SDK ngay từ đầu
4. **Docker experience** (nếu có) → ECS/Fargate dễ thở
5. **Problem solving** → Troubleshooting AWS issues = debugging

### Challenges (cần kiên nhẫn):

1. **Networking là invisible** → Cần nhiều visualization, diagram
2. **System admin ≠ coding** → Quản lý OS/services khác viết code
3. **Scale thinking** → Từ "1 server" sang "1000 servers across regions"
4. **Breadth vs Depth** → AWS có 200+ services, ban đầu overwhelming
5. **Non-deterministic** → Network issues không debug như code (timing, routing, DNS cache...)

### Chiến lược đặc biệt:

```
Tuần 1-5 (Networking): ĐI CHẬM, vẽ nhiều diagram, dùng Wireshark
Tuần 6-8 (Linux): Nhanh hơn — bạn đã quen terminal
Tuần 9-10 (Containers/DB): Fastest — gần với coding world
Tuần 11-12 (Security): Moderate — cần connect với networking knowledge

```

---

## 🔗 RESOURCE LINKS

| Category | Resource | Link | Ghi chú |
| --- | --- | --- | --- |
| Networking | Subnetting Practice | subnettingpractice.com | Daily practice |
| Networking | Professor Messer Network+ | youtube.com/professormesser | Free, structured |
| Networking | NetworkChuck | youtube.com/networkchuck | Fun, visual |
| Linux | Linux Journey | linuxjourney.com | Interactive |
| Linux | OverTheWire Bandit | overthewire.org/wargames/bandit | Gamified CLI |
| Containers | Docker Getting Started | docs.docker.com/get-started | Official |
| Security | TryHackMe | tryhackme.com | Interactive, beginner |
| Python/AWS | Boto3 Docs | boto3.amazonaws.com | Reference |
| AWS | Skill Builder | skillbuilder.aws | Free courses |
| AWS | Well-Architected Labs | wellarchitectedlabs.com | Hands-on |
| AWS | AWS Workshops | workshops.aws | Official labs |
| Practice | Tutorials Dojo | tutorialsdojo.com | Best practice exams |
| Course | Stephane Maarek | udemy.com | Best overall |
| Course | Adrian Cantrill | learn.cantrill.io | Most technical |

---

> 📌 **Tổng kết thay đổi so với bản trước:**
> 1. **IT Foundation: 8 tuần → 12 tuần** (thêm 4 tuần buffer)
> 2. **Networking: 3 tuần → 5 tuần** (nhiều thời gian "ngấm" nhất)
> 3. **Security: từ tuần 6 → tuần 11-12** (để cuối, cần foundation khác trước)
> 4. **Thêm ngày nghỉ/buffer** trong mỗi tuần (5-6 ngày thay vì 6-7)
> 5. **Thêm "Developer advantages" section** — tận dụng background coding
> 6. **Tổng lộ trình: 18 tháng → 20-22 tháng** (realistic hơn)

