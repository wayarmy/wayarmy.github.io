---
layout: post
title:  "AWS Learning Path"
subtitle: AWS Learning Path - Mapping with IT Foundation
gh-repo: wayarmy/wayarmy.github.io
tags: [sysadmin, developer, devops, cloud]
comments: true
date:   2026-05-31
categories: Sysadmin
---

> 💡 **Có thể học song song:** Các phần trong Phase 0 (Networking, Linux, Containers, Security) **không bắt buộc phải học tuần tự từ đầu tới cuối**. Bạn hoàn toàn có thể học song song — ví dụ sáng học Networking, chiều học Linux. Thứ tự trong tài liệu này chỉ là gợi ý ưu tiên, nhưng vì mỗi hệ thống kiến thức tương đối độc lập ở giai đoạn đầu, bạn có thể mix & match tùy theo sở thích và năng lượng mỗi ngày. **Ngoại lệ duy nhất:** Security nên để cuối cùng vì cần hiểu Networking + Linux trước.

> Phụ lục cho Lộ trình AWS — Deep dive từng topic + Lịch học ngày/tuần
>
> **Dành cho:** Người mới, đã có kiến thức coding cơ bản (biết lập trình, functions, logic)
>
> **Giả định:** Học 2-3 giờ/ngày, 5-6 ngày/tuần
>
> **Triết lý:** Mỗi concept cần thời gian "ngấm" — không nhồi nhét, ưu tiên hiểu sâu hơn học nhanh
>
> **🧪 Labs chính:** [Cloud Journey - AWS Study Group](https://cloudjourney.awsstudygroup.com/) — Tiếng Việt, step-by-step, có hình ảnh

---

## 📑 MỤC LỤC

1. [IT Foundation Deep Dive (12 tuần)](#-ph%E1%BA%A7n-1-it-foundation-deep-dive)
2. [Weekly Timeline toàn bộ lộ trình (20-22 tháng)](#-ph%E1%BA%A7n-2-weekly-timeline-chi-ti%E1%BA%BFt)

---

# 🏗️ PHẦN 1: IT FOUNDATION DEEP DIVE

> ⚠️ **Lưu ý cho người có background coding:**
>
> Bạn đã có tư duy logic, biết viết code — đó là lợi thế lớn. Tuy nhiên, Networking và Linux là hai thế giới hoàn toàn khác so với application programming. Đừng nóng vội — những concept như subnetting, BGP, hay systemd cần thời gian thực hành lặp đi lặp lại mới "ngấm" được.

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
| **T4** | I/O Redirection & Pipes | `>` (overwrite), `>>` (append), `<` (input), `2>` (stderr), `&>` (all). Pipes `\|`. `tee`. `xargs`. **Developer analogy:** Pipes = chaining functions | Chain commands with pipes |
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
>
> 🧪 **Bắt đầu có labs Cloud Journey ở đây:**

### Tuần 9: Virtualization & Containers

| Ngày | Chủ đề | Nội dung chi tiết | 🧪 Thực hành chính |
| --- | --- | --- | --- |
| **T2** | Virtualization concepts | Hypervisor Type 1 (bare metal: Xen, KVM, Nitro) vs Type 2 (VirtualBox, VMware). VM = full OS copy. Snapshots. **AWS:** EC2 = VM on Nitro | Tạo VM trong VirtualBox, take snapshot, restore |
| **T3** | Docker (phần 1) | Container ≠ VM (shares kernel, lighter). Docker architecture: daemon, client, registry. Image vs Container. `docker run`, `docker ps`, `docker stop` | 🧪 [Containerization with Docker](https://000015.awsstudygroup.com/) |
| **T4** | Docker (phần 2) | Dockerfile: `FROM`, `RUN`, `COPY`, `CMD`, `EXPOSE`, `ENV`, `WORKDIR`. Layer caching. `.dockerignore`. **AWS:** ECR = Docker Hub của AWS | 🧪 Tiếp tục lab Docker → build image cho app của bạn |
| **T5** | Docker Compose & Multi-container | `docker-compose.yml`: services, networks, volumes. Multi-container apps (app + db + cache). **AWS:** ECS Task Definition ≈ docker-compose | 🧪 [Container Orchestration with Amazon ECS](https://000016.awsstudygroup.com/) |
| **CN** | Container Orchestration (overview) | Tại sao cần orchestration? K8s concepts: Pod, Service, Deployment. **AWS:** ECS vs EKS vs Fargate | 🧪 [Container Deployment with Lightsail Containers](https://000046.awsstudygroup.com/) |

---

### Tuần 10: Database Concepts & Data

| Ngày | Chủ đề | Nội dung chi tiết | 🧪 Thực hành chính |
| --- | --- | --- | --- |
| **T2** | Relational DB deep | ACID. Normalization (1NF, 2NF, 3NF). Indexes (B-tree). Transactions. **AWS:** RDS, Aurora | Design schema cho e-commerce (users, orders, products) |
| **T3** | SQL review & advanced | Joins (INNER, LEFT, RIGHT, FULL). Subqueries. Window functions. `GROUP BY` + `HAVING`. `EXPLAIN` | Write 10 queries on sample DB (SQLite/PostgreSQL) |
| **T4** | NoSQL concepts | Key-Value (Redis, DynamoDB). Document (MongoDB). When to use which? CAP theorem | Design DynamoDB table: partition key, sort key, GSI |
| **T5** | Data Architecture overview | OLTP vs OLAP. Data Lake vs Data Warehouse. ETL/ELT. **AWS:** RDS → Glue → Redshift/S3 | Diagram: Full data pipeline source → lake → warehouse |
| **CN** | IaC concepts | Infrastructure as Code: Declarative vs Imperative. **AWS:** CloudFormation, CDK, Terraform | Đọc 1 CloudFormation template, understand structure |

📝 Checkpoint Tuần 10:

- [ ] Build & deploy Docker containers
- [ ] Design database schema (SQL + NoSQL)
- [ ] Hiểu data pipeline architecture
- [ ] Đọc được CloudFormation template

---

## 🔴 TUẦN 11-12: SECURITY FUNDAMENTALS (Để cuối — cần networking + Linux trước)

> 💡 **Tại sao cuối?** Bây giờ bạn đã hiểu TCP/IP, Ports, HTTP → hiểu TLS/firewall rules; Linux permissions → hiểu IAM.

### Tuần 11: Encryption, TLS, PKI

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Encryption Basics | Symmetric (AES-256): 1 key, fast, data at rest. Asymmetric (RSA): 2 keys, slow, key exchange. **AWS:** KMS = managed symmetric keys | `openssl enc -aes-256-cbc` encrypt/decrypt file |
| **T3** | TLS/SSL | TLS Handshake: ClientHello → ServerHello → Certificate → Key Exchange → Encrypted. Tại sao HTTPS cần certificate? | `openssl s_client -connect google.com:443` — đọc cert chain |
| **T4** | PKI & Certificates | CA hierarchy: Root → Intermediate → Leaf. CSR process. Certificate = public key + identity + CA signature. **AWS:** ACM manages certs | Tạo self-signed cert với openssl, hiểu từng field |
| **T5** | Hashing & Digital Signatures | Hash: SHA-256 (one-way, deterministic). Digital signature = hash encrypted with private key. | Hash file, modify 1 byte, hash lại — thấy avalanche effect |
| **CN** | Encryption in AWS | KMS, Envelope encryption. SSE-S3, SSE-KMS, SSE-C. Client-Side encryption. | Map encryption types cho S3, EBS, RDS |

---

### Tuần 12: Authentication, Authorization & Security Models

| Ngày | Chủ đề | Nội dung chi tiết | Thực hành |
| --- | --- | --- | --- |
| **T2** | Authentication | Factors: Knowledge, Possession, Inherence. MFA. Password hashing (bcrypt). Session vs Token auth | Decode a JWT: header.payload.signature |
| **T3** | Authorization & Access Control | RBAC, ABAC. Policy evaluation logic. **AWS:** IAM = RBAC + ABAC. Policy JSON | Viết 3 IAM policies: admin, readonly, specific-service |
| **T4** | Identity Federation | OAuth 2.0 (authorization). OIDC (auth). SAML 2.0 (enterprise SSO). **AWS:** Cognito, IAM Identity Center | Diagram: OAuth 2.0 flow cho "Login with Google" |
| **T5** | Security Architecture | Shared Responsibility Model. Defense in Depth: Edge → Network → Host → App → Data. Least privilege. Zero trust | Map all security layers cho 3-tier AWS app |
| **CN** | 🏆 SECURITY & IT FOUNDATION CAPSTONE | Complete security design + **Final quiz: 50 câu all topics** | Document + quiz score |

📝 Checkpoint Tuần 12 (IT FOUNDATION COMPLETE):

- [ ] Explain encryption types and when to use each
- [ ] Understand TLS handshake
- [ ] Write IAM policies
- [ ] Design defense-in-depth architecture
- [ ] **READY FOR AWS CLOUD PRACTITIONER** 🎉

---

# 📅 PHẦN 2: WEEKLY TIMELINE CHI TIẾT

> **Assumptions:** 2-3h/ngày, 5-6 ngày/tuần, bắt đầu từ tháng 6/2026
>
> **Tổng thời gian:** ~20-22 tháng
>
> **🧪 Từ Phase 1 trở đi:** Mỗi tuần có **Lab chính từ [Cloud Journey](https://cloudjourney.awsstudygroup.com/)** — đây là bài thực hành BẮT BUỘC. Cột "Add-on" là bài tập tùy chọn thêm.

## 🗓️ PHASE 0: IT FOUNDATION (Tuần 1-12) — 3 tháng

| Tuần | Focus | Deliverable |
| --- | --- | --- |
| W1 | Networking: OSI, TCP/IP, Wireshark | OSI diagram + first packet capture |
| W2 | Networking: IP Addressing, CIDR, Subnetting | Subnet calculations confident |
| W3 | Networking: Subnetting advanced, Routing | VPC CIDR design document |
| W4 | Networking: DNS, HTTP, TCP/UDP, NAT | DNS trace + NAT diagram |
| W5 | Networking: Firewalls, VPN, LB, BGP | Full network architecture diagram |
| W6 | Linux: CLI, files, permissions, users | Confident CLI navigation |
| W7 | Linux: Packages, services, networking CLI | Troubleshooting checklist |
| W8 | Linux: Bash scripting, automation | EC2 User Data script |
| W9 | Containers: Docker, Docker Compose | 🧪 [Docker](https://000015.awsstudygroup.com/) + [ECS](https://000016.awsstudygroup.com/) + [Lightsail Containers](https://000046.awsstudygroup.com/) |
| W10 | Databases, Data Architecture, IaC concepts | DB schema + pipeline diagram |
| W11 | Security: Encryption, TLS, PKI | Encryption mapping document |
| W12 | Security: Auth, IAM, Security Architecture | **IT Foundation COMPLETE** ✅ |

---

## 🗓️ PHASE 1: CLOUD PRACTITIONER (Tuần 13-17) — 5 tuần

> 🧪 **Bắt đầu từ đây, mỗi tuần có LAB CHÍNH từ [Cloud Journey](https://cloudjourney.awsstudygroup.com/)** — đây là bài thực hành bắt buộc, có hướng dẫn step-by-step tiếng Việt.

| Tuần | Focus | 🧪 Lab chính (Cloud Journey) | Add-on |
| --- | --- | --- | --- |
| W13 | Cloud Concepts & Account Setup | [Creating Your First AWS Account](https://000001.awsstudygroup.com/) | AWS Skill Builder: Cloud Concepts |
| W13 |  | [Managing Costs with AWS Budgets](https://000007.awsstudygroup.com/) |  |
| W13 |  | [Getting Help with AWS Support](https://000009.awsstudygroup.com/) |  |
| W14 | IAM & Compute | [Access Management with IAM](https://000002.awsstudygroup.com/) | Explore IAM console |
| W14 |  | [Compute Essentials with EC2](https://000004.awsstudygroup.com/) |  |
| W14 |  | [Instance Profiling with IAM Roles for EC2](https://000048.awsstudygroup.com/) |  |
| W15 | Network, Storage & DB | [Networking Essentials with VPC](https://000003.awsstudygroup.com/) | Vẽ lại VPC diagram |
| W15 |  | [Static Website Hosting with S3](https://000057.awsstudygroup.com/) |  |
| W15 |  | [Database Essentials with RDS](https://000005.awsstudygroup.com/) |  |
| W16 | Monitoring, DNS & CLI | [Monitoring with CloudWatch](https://000008.awsstudygroup.com/) | AWS Whitepaper: Overview |
| W16 |  | [Hybrid DNS Management with Route 53](https://000010.awsstudygroup.com/) |  |
| W16 |  | [Command Line Operations with AWS CLI](https://000011.awsstudygroup.com/) |  |
| W17 | **EXAM PREP** | [Building Highly Available Web Applications](https://000101.awsstudygroup.com/) ← capstone lab | 3 practice exams (65Q). **🎯 THI CLF-C02** |

---

## 🗓️ PHASE 2A: SOLUTIONS ARCHITECT ASSOCIATE (Tuần 18-31) — 14 tuần

| Tuần | Focus | 🧪 Lab chính (Cloud Journey) | Add-on |
| --- | --- | --- | --- |
| W18 | IAM Deep | [Access Management with IAM](https://000002.awsstudygroup.com/) (review deep) | Policy Simulator |
| W18 |  | [Access Control with IAM Policies and Conditions](https://000044.awsstudygroup.com/) |  |
| W19 | EC2 Deep | [Compute Essentials with EC2](https://000004.awsstudygroup.com/) (advanced) | Spot Instance lab |
| W19 |  | [Cloud Development with AWS Cloud9](https://000049.awsstudygroup.com/) |  |
| W20 | Storage: EBS, EFS, FSx | [Storage Performance Workshop](https://000068.awsstudygroup.com/) | EBS snapshot practice |
| W21 | S3 Deep | [Static Website Hosting with S3](https://000057.awsstudygroup.com/) (deep) | S3 lifecycle config |
| W21 |  | [S3 Security Best Practices](https://000069.awsstudygroup.com/) |  |
| W22 | ELB & Auto Scaling | [Scaling Applications with EC2 Auto Scaling](https://000006.awsstudygroup.com/) | Target tracking policy |
| W22 |  | [Monitoring with CloudWatch](https://000008.awsstudygroup.com/) (deep) |  |
| W23 | Database Services | [Database Essentials with RDS](https://000005.awsstudygroup.com/) (Multi-AZ) | Aurora Serverless |
| W23 |  | [Advanced PostgreSQL on AWS - Part 1](https://000115.awsstudygroup.com/) |  |
| W24 | DynamoDB & NoSQL | [NoSQL Database Essentials with DynamoDB](https://000060.awsstudygroup.com/) | GSI/LSI design |
| W24 |  | [In-Memory Caching with ElastiCache](https://000061.awsstudygroup.com/) |  |
| W24 |  | [Building Advanced Applications with DynamoDB](https://000039.awsstudygroup.com/) |  |
| W25 | VPC Deep | [Networking Essentials with VPC](https://000003.awsstudygroup.com/) (advanced) | VPC Endpoints lab |
| W25 |  | [Networking on AWS Workshop](https://000092.awsstudygroup.com/) |  |
| W25 |  | [Private Access to S3 with VPC Endpoints](https://000111.awsstudygroup.com/) |  |
| W26 | Route 53 & CloudFront | [Hybrid DNS Management with Route 53](https://000010.awsstudygroup.com/) (deep) | Failover routing |
| W26 |  | [Content Delivery with CloudFront](https://000094.awsstudygroup.com/) |  |
| W26 |  | [Edge Computing with CloudFront and Lambda@Edge](https://000130.awsstudygroup.com/) |  |
| W27 | Serverless | [Serverless Automation with Lambda](https://000022.awsstudygroup.com/) | API GW REST vs HTTP |
| W27 |  | [Workflow Orchestration with Step Functions](https://000047.awsstudygroup.com/) |  |
| W28 | Decoupling & Integration | [Messaging Systems with SQS and SNS](https://000077.awsstudygroup.com/) | EventBridge patterns |
| W28 |  | [Event-Driven Architecture](https://000054.awsstudygroup.com/) |  |
| W29 | Security & Monitoring | [Encryption with AWS KMS](https://000033.awsstudygroup.com/) | X-Ray tracing |
| W29 |  | [Credentials Management with Secrets Manager](https://000096.awsstudygroup.com/) |  |
| W29 |  | [Application Protection with AWS WAF](https://000026.awsstudygroup.com/) |  |
| W30 | HA & DR | [Disaster Recovery with AWS Elastic DR](https://000100.awsstudygroup.com/) | Multi-Region design |
| W30 |  | [Data Protection with AWS Backup](https://000013.awsstudygroup.com/) |  |
| W31 | **EXAM PREP** | [Building Highly Available Web Applications](https://000101.awsstudygroup.com/) ← review | 4-5 full practice exams. **🎯 THI SAA-C03** |

---

## 🗓️ PHASE 2B: SYSOPS ASSOCIATE (Tuần 32-43) — 12 tuần

| Tuần | Focus | 🧪 Lab chính (Cloud Journey) | Add-on |
| --- | --- | --- | --- |
| W32-33 | CloudWatch Deep | [Advanced Monitoring with CloudWatch and Grafana](https://000029.awsstudygroup.com/) | Custom metrics |
|  |  | [CloudWatch Advanced Workshop](https://000036.awsstudygroup.com/) |  |
| W34-35 | Systems Manager | [Systems Management with AWS Systems Manager](https://000031.awsstudygroup.com/) | Patch Manager |
|  |  | [Remote Server Access with Session Manager](https://000058.awsstudygroup.com/) |  |
| W36-37 | Deployment & IaC | [Infrastructure as Code with CloudFormation](https://000037.awsstudygroup.com/) | Nested stacks |
|  |  | [Cloud Development Kit (CDK) Essentials](https://000038.awsstudygroup.com/) |  |
|  |  | [AWS CDK Advanced](https://000076.awsstudygroup.com/) |  |
|  |  | [Infrastructure as Code Workshop Series](https://000102.awsstudygroup.com/) |  |
| W38-39 | Networking Operations | [Network Monitoring with VPC Flow Logs](https://000074.awsstudygroup.com/) | Reachability Analyzer |
|  |  | [Network Integration with VPC Peering](https://000019.awsstudygroup.com/) |  |
| W40-41 | Cost & Performance | [Right-Sizing with EC2 Resource Optimization](https://000032.awsstudygroup.com/) | Trusted Advisor |
|  |  | [Billing Console Delegation](https://000075.awsstudygroup.com/) |  |
|  |  | [Managing Quotas with Service Quotas](https://000063.awsstudygroup.com/) |  |
|  |  | [Cost and Usage Management](https://000064.awsstudygroup.com/) |  |
|  |  | [Cost Savings with Savings Plans and RI](https://000042.awsstudygroup.com/) |  |
|  |  | [Cost Visualization and Analytics](https://000034.awsstudygroup.com/) |  |
| W42 | Account, Backup & Compliance | [Resource Organization with Tags and Resource Groups](https://000027.awsstudygroup.com/) | AWS Config rules |
|  |  | [Access Control with IAM and Resource Tags](https://000028.awsstudygroup.com/) |  |
|  |  | [Snapshot Automation with EBS Data Lifecycle Manager](https://000088.awsstudygroup.com/) |  |
|  |  | [Anomaly Detection for EBS Backups](https://000089.awsstudygroup.com/) |  |
| W43 | **EXAM PREP** | [Serverless Automation with Lambda](https://000022.awsstudygroup.com/) ← automation review | SOA.pdf dump + Tutorials Dojo. **🎯 THI SOA-C02** |

---

## 🗓️ PHASE 3: SA PROFESSIONAL (Tuần 44-62) — 19 tuần

| Tuần | Focus | 🧪 Lab chính (Cloud Journey) | Add-on |
| --- | --- | --- | --- |
| W44-46 | Multi-Account Strategy | [Identity Federation with AWS SSO](https://000012.awsstudygroup.com/) | Control Tower |
|  |  | [Permission Management with IAM Permission Boundaries](https://000030.awsstudygroup.com/) |  |
| W47-50 | Advanced Networking | [Centralized Network Management with Transit Gateway](https://000020.awsstudygroup.com/) | Direct Connect theory |
|  |  | [Network Integration with VPC Peering](https://000019.awsstudygroup.com/) |  |
|  |  | [Networking on AWS Workshop](https://000092.awsstudygroup.com/) (review) |  |
| W51-53 | Migration & Modernization | [VM Migration with AWS VM Import/Export](https://000014.awsstudygroup.com/) | Snow Family theory |
|  |  | [Database Migration with DMS and SCT](https://000043.awsstudygroup.com/) |  |
|  |  | [Monolith to Microservices Migration](https://000050.awsstudygroup.com/) |  |
| W54-56 | Advanced Architecture | [Event-Driven Architecture](https://000054.awsstudygroup.com/) | Multi-region active-active |
|  |  | [Messaging Systems with SQS and SNS](https://000077.awsstudygroup.com/) |  |
|  |  | [Workflow Orchestration with Step Functions](https://000047.awsstudygroup.com/) |  |
|  |  | [Building Highly Available Web Applications](https://000101.awsstudygroup.com/) |  |
| W57-59 | Cost & Security at Scale | [Security Compliance with Security Hub](https://000018.awsstudygroup.com/) | Whitepapers |
|  |  | [Security Governance with Firewall Manager](https://000097.awsstudygroup.com/) |  |
|  |  | [Cost Data Analysis with Glue and Athena](https://000040.awsstudygroup.com/) |  |
| W60-61 | Review & Weak Areas | Redo bất kỳ lab nào chưa vững | Re-read whitepapers |
| W62 | **EXAM PREP** | — | SAP.pdf (196 pages) + Q&A PDFs. **🎯 THI SAP-C02** |

---

## 🗓️ PHASE 4: SPECIALTY (Tuần 63+)

### Option A: Security Specialty (Tuần 63-75)

| Tuần | Focus | 🧪 Lab chính (Cloud Journey) |
| --- | --- | --- |
| W63-65 | IAM & Identity Federation | [Identity Federation with AWS SSO](https://000012.awsstudygroup.com/) |
|  |  | [Permission Management with IAM Permission Boundaries](https://000030.awsstudygroup.com/) |
|  |  | [Access Control with IAM Policies and Conditions](https://000044.awsstudygroup.com/) |
|  |  | [Cross-Domain Authentication with Cognito](https://000141.awsstudygroup.com/) |
| W66-68 | Data Protection | [Encryption with AWS KMS](https://000033.awsstudygroup.com/) |
|  |  | [Data Protection with Amazon Macie](https://000090.awsstudygroup.com/) |
|  |  | [Credentials Management with Secrets Manager](https://000096.awsstudygroup.com/) |
|  |  | [S3 Security Best Practices](https://000069.awsstudygroup.com/) |
| W69-71 | Infrastructure Security | [Application Protection with AWS WAF](https://000026.awsstudygroup.com/) |
|  |  | [Private Access to S3 with VPC Endpoints](https://000111.awsstudygroup.com/) |
|  |  | [Security Governance with Firewall Manager](https://000097.awsstudygroup.com/) |
|  |  | [Systems Patching with EC2 Image Builder](https://000099.awsstudygroup.com/) |
| W72-73 | Detection & Response | [Threat Detection with GuardDuty](https://000098.awsstudygroup.com/) |
|  |  | [Security Compliance with Security Hub](https://000018.awsstudygroup.com/) |
| W74-75 | **EXAM PREP** | SCS.pdf (158 pages). **🎯 THI SCS-C02** |

### Option B: Advanced Networking (Tuần 63-77)

| Tuần | Focus | 🧪 Lab chính (Cloud Journey) |
| --- | --- | --- |
| W63-66 | Transit Gateway deep | [Centralized Network Management with Transit Gateway](https://000020.awsstudygroup.com/) |
|  |  | [Networking on AWS Workshop](https://000092.awsstudygroup.com/) |
| W67-70 | Hybrid & Connectivity | [Network Integration with VPC Peering](https://000019.awsstudygroup.com/) |
|  |  | [Network Monitoring with VPC Flow Logs](https://000074.awsstudygroup.com/) |
| W71-73 | DNS, CDN, Security | [Hybrid DNS Management with Route 53](https://000010.awsstudygroup.com/) |
|  |  | [Content Delivery with CloudFront](https://000094.awsstudygroup.com/) |
|  |  | [Application Protection with AWS WAF](https://000026.awsstudygroup.com/) |
| W74-76 | Advanced scenarios | [Private Access to S3 with VPC Endpoints](https://000111.awsstudygroup.com/) |
| W77 | **EXAM PREP** | ANS.pdf (162 pages). **🎯 THI ANS-C01** |

### Option C: Data Engineer (Tuần 63-75)

| Tuần | Focus | 🧪 Lab chính (Cloud Journey) |
| --- | --- | --- |
| W63-65 | S3 Data Lake | [Data Lake Fundamentals](https://000035.awsstudygroup.com/) |
|  |  | [Building a Data Lake with Your Own Data](https://000070.awsstudygroup.com/) |
| W66-68 | Glue, ETL & Analytics | [Data Engineering Immersion Day](https://000105.awsstudygroup.com/) |
|  |  | [Serverless Analytics with Amazon Athena](https://000106.awsstudygroup.com/) |
|  |  | [Cost Data Analysis with AWS Glue and Athena](https://000040.awsstudygroup.com/) |
| W69-71 | Streaming & Visualization | [Data Analytics Services Overview](https://000072.awsstudygroup.com/) |
|  |  | [Business Intelligence with QuickSight](https://000073.awsstudygroup.com/) |
|  |  | [Advanced PostgreSQL on AWS - Part 1](https://000115.awsstudygroup.com/) |
|  |  | [Advanced PostgreSQL on AWS - Part 2](https://000116.awsstudygroup.com/) |
| W72-73 | Orchestration | [Workflow Orchestration with Step Functions](https://000047.awsstudygroup.com/) |
| W74-75 | **EXAM PREP** | DEA.pdf (141 pages). **🎯 THI DEA-C01** |

### Option D: ML/AI Track (Tuần 63-82)

| Tuần | Focus | 🧪 Lab chính (Cloud Journey) |
| --- | --- | --- |
| W63-66 | ML Fundamentals | [Machine Learning with SageMaker](https://000200.awsstudygroup.com/) |
| W67-70 | SageMaker Deep | [Machine Learning with SageMaker](https://000200.awsstudygroup.com/) (advanced) |
| W71-73 | AIF-C01 Prep | [AWS AI Services Integration](https://000056.awsstudygroup.com/) |
| W74-80 | MLA-C01 & MLS-C01 | MLA.docx + MLA.pdf + MLS.pdf dumps. **🎯 THI AIF → MLA → MLS** |

---

### 🚀 BONUS: APPLICATION MODERNIZATION WORKSHOP SERIES

> Làm khi hoàn thành Phase 2A trở đi. Đây là full workshop series, mỗi series gồm 5-9 bài liên tiếp.

| Series | Labs (theo thứ tự) | Khi nào làm |
| --- | --- | --- |
| **Serverless - Book Store** | [1. Backend](https://000078.awsstudygroup.com/) → [2. Frontend](https://000079.awsstudygroup.com/) → [3. SAM](https://000080.awsstudygroup.com/) → [4. Cognito](https://000081.awsstudygroup.com/) → [5. Custom Domain](https://000082.awsstudygroup.com/) → [6. SQS/SNS](https://000083.awsstudygroup.com/) → [7. CI/CD](https://000084.awsstudygroup.com/) → [8. Monitoring](https://000085.awsstudygroup.com/) → [9. AppSync](https://000086.awsstudygroup.com/) | Sau W27 (Serverless) |
| **Serverless - DevAx** | [1. Monolith→Micro](https://000050.awsstudygroup.com/) → [2. CI/CD](https://000051.awsstudygroup.com/) → [3. Microservices](https://000052.awsstudygroup.com/) → [4. Data Restructuring](https://000053.awsstudygroup.com/) → [5. Event-Driven](https://000054.awsstudygroup.com/) → [6. SPA Auth](https://000055.awsstudygroup.com/) → [7. AI Services](https://000056.awsstudygroup.com/) | Sau W43 (Phase 2B) |
| **ECS Workshop** | [1. ECS + Fargate](https://000067.awsstudygroup.com/) → [2. IaC for ECS](https://000118.awsstudygroup.com/) → [3. CI/CD for ECS](https://000152.awsstudygroup.com/) | Sau W22 |
| **EKS Workshop** | [1. EKS basics](https://000126.awsstudygroup.com/) → [2. EKS Blueprints](https://000065.awsstudygroup.com/) → [3. CI/CD for EKS](https://000062.awsstudygroup.com/) | Sau Phase 2B |
| **Docker + CI/CD** | [1. Docker](https://000015.awsstudygroup.com/) → [2. ECS](https://000016.awsstudygroup.com/) → [3. CodePipeline](https://000017.awsstudygroup.com/) → [4. Auto Deploy](https://000023.awsstudygroup.com/) | Sau W9 (Containers) |
| **WordPress on AWS** | [1. Architecture](https://000021.awsstudygroup.com/) → [2. EC2 Deploy](https://000091.awsstudygroup.com/) | Anytime after Phase 1 |

---

## 📊 DAILY ROUTINE MẪU

### Ngày học thường (2-2.5h):

```
┌─────────────────────────────────────────────────────┐
│  🧠 20 phút — Review hôm trước (flashcards/notes)   │
│  📖 50 phút — Kiến thức mới (video hoặc docs)       │
│  ☕ 10 phút — Nghỉ ngắn (quan trọng!)               │
│  🧪 40 phút — Lab Cloud Journey hoặc hands-on       │
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
Tuần 9-10 (Containers/DB): Fastest — gần với coding world + Cloud Journey labs
Tuần 11-12 (Security): Moderate — connect với networking knowledge
Phase 1+ : MỖI TUẦN phải hoàn thành ít nhất 2-3 labs Cloud Journey

```

---

## 🔗 RESOURCE LINKS

| Category | Resource | Link | Ghi chú |
| --- | --- | --- | --- |
| **🧪 Labs chính** | **Cloud Journey - AWS Study Group** | [cloudjourney.awsstudygroup.com](https://cloudjourney.awsstudygroup.com/) | **BÀI THỰC HÀNH CHÍNH — dùng xuyên suốt** |
| Networking | Subnetting Practice | subnettingpractice.com | Daily practice (Phase 0) |
| Networking | Professor Messer Network+ | youtube.com/professormesser | Free, structured |
| Networking | NetworkChuck | youtube.com/networkchuck | Fun, visual |
| Linux | Linux Journey | linuxjourney.com | Interactive |
| Linux | OverTheWire Bandit | overthewire.org/wargames/bandit | Gamified CLI |
| Containers | Docker Getting Started | docs.docker.com/get-started | Official |
| Security | TryHackMe | tryhackme.com | Interactive, beginner |
| AWS | Skill Builder | skillbuilder.aws | Free courses (add-on) |
| AWS | Well-Architected Labs | wellarchitectedlabs.com | Hands-on (add-on) |
| AWS | AWS Workshops | workshops.aws | Official labs (add-on) |
| Practice Exams | Tutorials Dojo | tutorialsdojo.com | Best practice exams |
| Course | Stephane Maarek | udemy.com | Best overall |
| Course | Adrian Cantrill | learn.cantrill.io | Most technical |

---

> 📌 **Nguyên tắc sử dụng tài liệu:**
> - **🧪 Labs CHÍNH** = Cloud Journey (awsstudygroup.com) — làm bắt buộc, có link cụ thể trong bảng mỗi tuần
> - **Add-on** = Các bài tập thêm trong cột Add-on + AWS Skill Builder + Workshops
> - **Phase 0 (IT Foundation)** = Tự thực hành trên local machine (Wireshark, Linux VM, Docker) — chưa cần AWS account nhiều
> - **Phase 1+** = Mỗi tuần phải hoàn thành **tất cả labs** trong cột "🧪 Lab chính" trước khi làm add-on
> - **Practice Exams** = Tutorials Dojo + PDF dumps (chỉ dùng ở tuần cuối mỗi cert)

