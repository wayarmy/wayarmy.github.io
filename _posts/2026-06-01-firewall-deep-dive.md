---
layout: post
title: "Firewall Deep Dive - Packet Filter, Stateful Inspection, iptables Chains/Tables, nftables & Conntrack"
date: 2026-06-01
categories: [networking]
tags: [firewall, iptables, nftables, netfilter, security, linux]
---

# Firewall Deep Dive — Packet Filter, Stateful Inspection, iptables Chains/Tables, nftables & Conntrack

## Mục lục (Table of Contents)
1. [Giới thiệu bằng câu chuyện đời thường](#1-giới-thiệu-bằng-câu-chuyện-đời-thường)
2. [Firewall Fundamentals — Phân loại firewall](#2-firewall-fundamentals--phân-loại-firewall)
3. [Packet Filtering — Lọc gói tin cơ bản](#3-packet-filtering--lọc-gói-tin-cơ-bản)
4. [Stateful Inspection — Lọc theo trạng thái kết nối](#4-stateful-inspection--lọc-theo-trạng-thái-kết-nối)
5. [Netfilter và iptables Architecture](#5-netfilter-và-iptables-architecture)
6. [iptables Tables và Chains](#6-iptables-tables-và-chains)
7. [iptables Rules — Viết luật firewall](#7-iptables-rules--viết-luật-firewall)
8. [nftables — Thế hệ mới thay thế iptables](#8-nftables--thế-hệ-mới-thay-thế-iptables)
9. [Conntrack — Connection Tracking](#9-conntrack--connection-tracking)
10. [Tổng kết và Best Practices](#10-tổng-kết-và-best-practices)

---

## 1. Giới thiệu bằng câu chuyện đời thường

### Firewall như bảo vệ tòa nhà

Hãy tưởng tượng tòa nhà công ty bạn có đội bảo vệ ở cổng:

| Bảo vệ | Firewall |
|---|---|
| Kiểm tra giấy tờ (ID badge) | Kiểm tra source/destination IP |
| Chỉ cho vào giờ hành chính (8-17h) | Chỉ cho phép port cụ thể |
| Nhân viên → được vào tầng mình làm | Internal traffic → cho phép |
| Khách → phải đăng ký trước | External traffic → phải match rule |
| Đuổi người lạ không mời | DROP packets không match rule |
| Ghi sổ ra/vào | Logging connections |
| Nhớ ai đã vào (để cho ra) | **Stateful** — nhớ connection đã thiết lập |

**Phân loại "bảo vệ":**

| Loại bảo vệ | Firewall tương đương | Khả năng |
|---|---|---|
| Chỉ xem thẻ (name tag) | **Packet Filter** | Kiểm tra header (IP, port) |
| Nhớ ai vào/ra + theo dõi | **Stateful Firewall** | Theo dõi connection state |
| Kiểm tra nội dung hành lý | **Application Firewall (L7)** | Inspect payload (HTTP, SQL...) |
| Toàn bộ hệ thống an ninh | **Next-Gen Firewall (NGFW)** | IDS/IPS + App Control + DPI |

---

## 2. Firewall Fundamentals — Phân loại firewall

### 2.1 Generations of Firewalls

```
Gen 1 (1988): Packet Filter
  └── Check: Source IP, Dest IP, Source Port, Dest Port, Protocol
  └── Stateless: Mỗi packet xét riêng lẻ
  └── Fast nhưng dễ bypass

Gen 2 (1990s): Stateful Inspection
  └── Track connection state (NEW, ESTABLISHED, RELATED)
  └── "Nhớ" traffic đã establish → auto-allow return traffic
  └── Ngăn chặn spoofed packets hiệu quả hơn

Gen 3 (2000s): Application Layer Gateway (ALG/Proxy)
  └── Understand application protocols (HTTP, FTP, DNS)
  └── Can inspect payload content
  └── Deep Packet Inspection (DPI)

Gen 4 (2010s+): Next-Generation Firewall (NGFW)
  └── All of above + IDS/IPS + Application awareness
  └── User identity integration
  └── SSL/TLS decryption and inspection
  └── Threat intelligence feeds
```

### 2.2 Network Firewall vs Host Firewall

| | Network Firewall | Host Firewall |
|---|---|---|
| Vị trí | Giữa networks (perimeter) | Trên mỗi server/workstation |
| Bảo vệ | Toàn bộ network segment | Server/machine cụ thể |
| Ví dụ | AWS Security Group, Palo Alto, pfSense | iptables, nftables, Windows Firewall |
| Quản lý | Network team | DevOps/SysAdmin |
| Layer | L3-L4 (hoặc L7 cho NGFW) | L3-L4 (kernel) |

---

## 3. Packet Filtering — Lọc gói tin cơ bản

### 3.1 Stateless Packet Filter

**Stateless** = xét **mỗi packet riêng lẻ**, không nhớ gì về packets trước đó.

**Ví dụ đời thường**: Bảo vệ chỉ kiểm tra ID card mỗi lần ai đi qua cổng, KHÔNG nhớ ai đã vào/ra trước đó.

```
Packet arrives:
┌─────────────────────────────────┐
│ IP Header:                       │
│   Src IP: 10.0.0.100            │
│   Dst IP: 192.168.1.50          │
│   Protocol: TCP                  │
│ TCP Header:                      │
│   Src Port: 52341               │
│   Dst Port: 80                   │
│   Flags: SYN                     │
└─────────────────────────────────┘

Firewall rules (kiểm tra từ trên xuống):
Rule 1: ALLOW TCP from any to 192.168.1.50:80   → MATCH ✅ → Allow
Rule 2: ALLOW TCP from any to 192.168.1.50:443
Rule 3: DROP all                                  → Default policy
```

### 3.2 Vấn đề của Stateless Filter

```
Vấn đề: Làm sao cho phép RESPONSE packets quay về?

Ví dụ:
- Server web (192.168.1.50:80) trả response cho client
- Response: Src=192.168.1.50:80, Dst=10.0.0.100:52341

Nếu stateless: Cần rule RIÊNG cho response traffic
  ALLOW TCP from 192.168.1.50:80 to any   ← Phải thêm rule này!
  
Vấn đề: 
  → Phải mở source port 80 ra ngoài (cho response)
  → Attacker có thể giả mạo src port 80 để bypass firewall
  → Phải quản lý rules cho CẢ HAI hướng = phức tạp gấp đôi
```

---

## 4. Stateful Inspection — Lọc theo trạng thái kết nối

### 4.1 Stateful Firewall

**Stateful** = firewall **nhớ connections** đã thiết lập. Response traffic được **tự động cho phép** nếu thuộc connection hợp lệ.

**Ví dụ đời thường**: Bảo vệ ghi sổ: "Anh A vào lúc 8:00". Khi Anh A ra lúc 17:00, bảo vệ tự cho ra (biết Anh A đã vào hợp lệ). Không cần Anh A trình ID lại.

### 4.2 Connection States

| State | Ý nghĩa | Ví dụ |
|---|---|---|
| **NEW** | Packet đầu tiên của connection mới | TCP SYN, first UDP packet |
| **ESTABLISHED** | Packet thuộc connection đã thiết lập | TCP data, UDP reply |
| **RELATED** | Packet liên quan đến connection khác | ICMP error cho TCP; FTP data |
| **INVALID** | Packet không thuộc connection nào hợp lệ | Random ACK, malformed |
| **UNTRACKED** | Packet được bypass conntrack | Raw packets, performance |

### 4.3 Stateful vs Stateless — Ví dụ cụ thể

```
Stateless firewall (PHẢI viết rules cho cả 2 hướng):
  ALLOW TCP dst-port 80 (incoming web requests)
  ALLOW TCP src-port 80 (outgoing web responses)  ← Phải thêm!
  ALLOW TCP dst-port 443
  ALLOW TCP src-port 443                           ← Phải thêm!
  DROP all

Stateful firewall (chỉ viết cho incoming, response tự allow):
  ALLOW NEW TCP dst-port 80
  ALLOW NEW TCP dst-port 443
  ALLOW ESTABLISHED,RELATED (tự cho phép TẤT CẢ response) ← Magic!
  DROP all
  
→ Đơn giản hơn + An toàn hơn (không mở src port ra)
```

---

## 5. Netfilter và iptables Architecture

### 5.1 Netfilter Framework

**Netfilter** là framework trong Linux kernel cung cấp các **hook points** (điểm móc) nơi packets đi qua. iptables/nftables là userspace tools để configure rules tại các hook points này.

```
                     Packet Flow trong Linux Kernel
                     ==============================

                            INCOMING PACKET
                                  │
                                  ↓
                         ┌────────────────┐
                         │  PREROUTING    │  ← Hook point 1
                         │  (raw, mangle, │    (Trước routing decision)
                         │   nat DNAT)    │
                         └───────┬────────┘
                                 │
                         ┌───────┴────────┐
                         │ Routing Decision│
                         │ "Packet cho ai?"│
                         └───┬────────┬───┘
                             │        │
              For THIS host  │        │  For ANOTHER host (forwarding)
                             ↓        ↓
                    ┌────────────┐  ┌────────────┐
                    │   INPUT    │  │  FORWARD   │  ← Hook point 2/3
                    │(filter,    │  │ (filter,   │
                    │ mangle)    │  │  mangle)   │
                    └─────┬──────┘  └──────┬─────┘
                          │                │
                          ↓                │
                   ┌─────────────┐         │
                   │ Local Process│         │
                   │ (Application)│         │
                   └──────┬──────┘         │
                          │                │
                          ↓                │
                    ┌──────────┐           │
                    │  OUTPUT  │ ← Hook 4  │
                    │(filter,  │           │
                    │ mangle,  │           │
                    │ nat DNAT)│           │
                    └─────┬────┘           │
                          │                │
                          └────────┬───────┘
                                   │
                                   ↓
                         ┌─────────────────┐
                         │  POSTROUTING    │  ← Hook point 5
                         │  (mangle,       │    (Sau routing, trước gửi)
                         │   nat SNAT/     │
                         │   MASQUERADE)   │
                         └────────┬────────┘
                                  │
                                  ↓
                           OUTGOING PACKET
```

### 5.2 Luồng packet chính

| Scenario | Path qua chains |
|---|---|
| Packet ĐẾN server | PREROUTING → INPUT → Local Process |
| Packet ĐI TỪ server | Local Process → OUTPUT → POSTROUTING |
| Packet ĐI QUA server (routing) | PREROUTING → FORWARD → POSTROUTING |

---

## 6. iptables Tables và Chains

### 6.1 Tables (Bảng)

iptables có **5 tables**, mỗi table có mục đích khác:

| Table | Mục đích | Chains chứa | Khi nào dùng |
|---|---|---|---|
| **filter** (default) | Lọc packets (allow/deny) | INPUT, OUTPUT, FORWARD | Firewall rules |
| **nat** | Network Address Translation | PREROUTING, OUTPUT, POSTROUTING | Port forwarding, SNAT |
| **mangle** | Sửa đổi packet headers | ALL 5 chains | QoS marking, TTL |
| **raw** | Bypass conntrack | PREROUTING, OUTPUT | Performance, exceptions |
| **security** | SELinux marking | INPUT, OUTPUT, FORWARD | Mandatory Access Control |

### 6.2 Chains (Chuỗi)

| Chain | Table(s) | Khi nào trigger | Ví dụ |
|---|---|---|---|
| **PREROUTING** | raw, mangle, nat | Packet vừa đến, trước routing | DNAT (redirect port) |
| **INPUT** | filter, mangle, security | Packet đi vào local process | Allow SSH, HTTP |
| **FORWARD** | filter, mangle, security | Packet đi qua (routing) | Router/gateway rules |
| **OUTPUT** | raw, mangle, nat, filter, security | Packet từ local process đi ra | Restrict outbound |
| **POSTROUTING** | mangle, nat | Packet sắp rời server | SNAT, MASQUERADE |

### 6.3 Rule Processing Order

```
Trong mỗi chain, rules được xét TỪ TRÊN XUỐNG (first match wins):

Chain INPUT:
  Rule 1: -p tcp --dport 22 -j ACCEPT    ← Packet SSH? → Match → ACCEPT (stop)
  Rule 2: -p tcp --dport 80 -j ACCEPT    ← Packet HTTP? → Match → ACCEPT
  Rule 3: -p tcp --dport 443 -j ACCEPT   ← Packet HTTPS? → Match → ACCEPT
  Rule 4: -m state --state ESTABLISHED -j ACCEPT  ← Response traffic? → ACCEPT
  Rule 5: (default policy) -j DROP       ← Không match gì → DROP

Packet TCP port 80: Xét Rule 1 (no match) → Rule 2 (MATCH!) → ACCEPT
Packet TCP port 3306: Xét 1,2,3,4 (no match) → Policy DROP
```

---

## 7. iptables Rules — Viết luật firewall

### 7.1 Syntax cơ bản

```bash
iptables -t <table> -A <chain> <match-criteria> -j <target>

# Components:
# -t table     : filter (default), nat, mangle, raw
# -A chain     : Append rule to chain (INPUT, OUTPUT, FORWARD)
# -I chain N   : Insert at position N
# -D chain N   : Delete rule at position N
# match        : -p protocol, -s src, -d dst, --dport, --sport, -i interface
# -j target    : ACCEPT, DROP, REJECT, LOG, DNAT, SNAT, MASQUERADE
```

### 7.2 Common Rules

```bash
# === DEFAULT POLICIES ===
iptables -P INPUT DROP      # Default: drop incoming
iptables -P FORWARD DROP    # Default: drop forwarding
iptables -P OUTPUT ACCEPT   # Default: allow outgoing

# === LOOPBACK (always allow) ===
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT

# === STATEFUL: Allow established connections ===
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# === SSH (port 22) ===
iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# === HTTP/HTTPS ===
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# === Restrict SSH to specific IP ===
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP

# === Rate limiting SSH (anti brute-force) ===
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m recent --set --name SSH
iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP

# === ICMP (ping) ===
iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# === LOG dropped packets ===
iptables -A INPUT -j LOG --log-prefix "DROPPED: " --log-level 4
iptables -A INPUT -j DROP

# === NAT: Port forwarding (DNAT) ===
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.50:80

# === NAT: Masquerade (SNAT for dynamic IP) ===
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### 7.3 Targets (Actions)

| Target | Ý nghĩa | Khác biệt |
|---|---|---|
| **ACCEPT** | Cho phép packet | Packet đi tiếp |
| **DROP** | Bỏ im lặng | Sender không biết (timeout) |
| **REJECT** | Từ chối + thông báo | Gửi ICMP error lại sender |
| **LOG** | Ghi log, tiếp tục xét rules | Non-terminating |
| **DNAT** | Đổi destination (port forward) | Dùng trong nat PREROUTING |
| **SNAT** | Đổi source (outbound NAT) | Dùng trong nat POSTROUTING |
| **MASQUERADE** | SNAT cho dynamic IP | Tự detect outgoing interface IP |
| **RETURN** | Quay về chain gọi | Dùng trong custom chains |

### 7.4 DROP vs REJECT

| | DROP | REJECT |
|---|---|---|
| Response | Không gì (silence) | ICMP Destination Unreachable |
| Sender behavior | Timeout (chờ lâu) | Biết ngay bị từ chối |
| Port scanning | Attacker không chắc port có tồn tại | Attacker biết host alive |
| User experience | Lag (timeout) | Nhanh (error ngay) |
| Best for | External (internet) | Internal (friendly network) |

---

## 8. nftables — Thế hệ mới thay thế iptables

### 8.1 Tại sao nftables?

| Vấn đề iptables | nftables giải quyết |
|---|---|
| Nhiều tools riêng (iptables, ip6tables, arptables, ebtables) | 1 tool: `nft` |
| Performance thấp (sequential rule matching) | Sets cho O(1) lookup |
| Syntax không nhất quán giữa tables | Unified syntax |
| Không atomic (rules apply từng cái) | Atomic rule replacement |
| Kernel code riêng mỗi table | Virtual machine (nf_tables) |

### 8.2 nftables Architecture

```
nftables structure:
┌─────────────────────────────────────────────┐
│ Table "inet filter"  (family: inet = ipv4+ipv6) │
│                                               │
│   Chain "input" (type filter, hook input, priority 0):
│     rule #1: tcp dport 22 accept
│     rule #2: tcp dport {80, 443} accept
│     rule #3: ct state established,related accept
│     rule #4: drop
│                                               │
│   Chain "forward" (type filter, hook forward):
│     rule #1: drop
│                                               │
│   Chain "output" (type filter, hook output):
│     rule #1: accept
│                                               │
└─────────────────────────────────────────────┘
```

### 8.3 nftables Syntax

```bash
#!/usr/sbin/nft -f

# Flush all rules
flush ruleset

# Create table (inet = ipv4 + ipv6)
table inet filter {
    # Sets (O(1) lookup thay vì sequential matching)
    set allowed_ports {
        type inet_service
        elements = { 22, 80, 443, 8080 }
    }
    
    set blocked_ips {
        type ipv4_addr
        flags interval
        elements = { 192.168.100.0/24, 10.99.0.0/16 }
    }

    chain input {
        type filter hook input priority 0; policy drop;
        
        # Loopback
        iif "lo" accept
        
        # Stateful
        ct state established,related accept
        ct state invalid drop
        
        # ICMP
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept
        
        # Allowed ports (from set)
        tcp dport @allowed_ports accept
        
        # Block bad IPs
        ip saddr @blocked_ips drop
        
        # Rate limit SSH
        tcp dport 22 ct state new limit rate 3/minute accept
        
        # Log + drop everything else
        log prefix "nftables-drop: " counter drop
    }
    
    chain forward {
        type filter hook forward priority 0; policy drop;
    }
    
    chain output {
        type filter hook output priority 0; policy accept;
    }
}

# NAT table
table inet nat {
    chain prerouting {
        type nat hook prerouting priority dstnat;
        # Port forward 8080 → internal server:80
        tcp dport 8080 dnat to 10.0.0.50:80
    }
    
    chain postrouting {
        type nat hook postrouting priority srcnat;
        # Masquerade outgoing traffic
        oifname "eth0" masquerade
    }
}
```

### 8.4 nftables Commands

```bash
# List all rules
nft list ruleset

# List specific table
nft list table inet filter

# Add rule interactively
nft add rule inet filter input tcp dport 3306 ip saddr 10.0.0.0/24 accept

# Delete rule (by handle)
nft -a list chain inet filter input  # Show handles
nft delete rule inet filter input handle 15

# Add to set dynamically
nft add element inet filter blocked_ips { 192.168.200.0/24 }

# Atomic replace (load from file)
nft -f /etc/nftables.conf

# Monitor events
nft monitor
```

---

## 9. Conntrack — Connection Tracking

### 9.1 Conntrack là gì?

**Conntrack (Connection Tracking)** là subsystem trong kernel Linux **theo dõi mọi network connection** đi qua. Nó là "bộ nhớ" của stateful firewall.

**Ví dụ đời thường**: Conntrack giống **sổ ghi chép ra/vào** — bảo vệ ghi: "10:00 Anh A vào, badge #123, đi tầng 5". Khi Anh A ra, bảo vệ check sổ: "Anh A có trong sổ → cho ra".

### 9.2 Connection Table

```bash
# Xem conntrack table
conntrack -L

# Ví dụ entries:
tcp  6  300  ESTABLISHED src=10.0.0.100 dst=93.184.216.34 sport=52341 dport=443 
     src=93.184.216.34 dst=10.0.0.100 sport=443 dport=52341 [ASSURED] mark=0 use=1

udp  17  29  src=10.0.0.100 dst=8.8.8.8 sport=54321 dport=53
     src=8.8.8.8 dst=10.0.0.100 sport=53 dport=54321 [ASSURED] mark=0 use=1

# Format:
# protocol  proto_num  timeout  state  original_tuple  reply_tuple  flags
```

### 9.3 Conntrack States cho TCP

```
TCP Connection Lifecycle trong conntrack:

SYN sent:     NEW (entry created)
SYN+ACK recv: ESTABLISHED (both directions seen)
Data flow:    ESTABLISHED (maintained)
FIN sent:     ESTABLISHED → TIME_WAIT
Timeout:      Entry removed from table

┌─────┐  SYN   ┌───────────┐  SYN-ACK  ┌─────────────┐
│ NEW │───────→│ESTABLISHED│←─────────→│ESTABLISHED  │
└─────┘        └───────────┘            └──────┬──────┘
                                               │ FIN
                                               ↓
                                        ┌──────────────┐
                                        │  TIME_WAIT   │
                                        └──────┬───────┘
                                               │ timeout
                                               ↓
                                        [Entry removed]
```

### 9.4 Conntrack cho UDP và ICMP

```
UDP (connectionless nhưng conntrack vẫn track):
- Packet đi ra: NEW entry created
- Reply từ same src:port: → ESTABLISHED
- Timeout: 30s (default) nếu không có traffic

ICMP:
- Echo request: NEW
- Echo reply: ESTABLISHED (matched to request)
- Related: ICMP error messages (e.g., port unreachable)
```

### 9.5 Conntrack Tuning

```bash
# Xem current conntrack stats
cat /proc/sys/net/netfilter/nf_conntrack_count    # Current entries
cat /proc/sys/net/netfilter/nf_conntrack_max      # Max entries

# Tăng conntrack table size (cho high-traffic servers)
echo 262144 > /proc/sys/net/netfilter/nf_conntrack_max
# Hoặc persistent:
echo "net.netfilter.nf_conntrack_max = 262144" >> /etc/sysctl.conf

# Timeout tuning
echo 120 > /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established
# Default: 432000 (5 days!) — quá lâu cho most use cases

# Hash table size (performance)
echo 65536 > /sys/module/nf_conntrack/parameters/hashsize

# Conntrack thống kê
conntrack -S
# cpu=0  found=1234 invalid=56 insert=0 insert_failed=0 drop=0 early_drop=0
```

### 9.6 Conntrack và Performance

```
Vấn đề: Mỗi connection = 1 entry trong conntrack table
  - 1M concurrent connections = 1M entries
  - Mỗi entry ≈ 300 bytes RAM
  - 1M entries ≈ 300 MB RAM

High-traffic server (load balancer, NAT gateway):
  - Cần tăng nf_conntrack_max
  - Giảm timeouts (không cần 5-day established timeout)
  - Hoặc BYPASS conntrack cho traffic không cần stateful:
  
  # Skip conntrack cho high-volume, stateless traffic
  iptables -t raw -A PREROUTING -p tcp --dport 80 -j NOTRACK
  # Hoặc nftables:
  # chain prerouting { type filter hook prerouting priority raw; ... }
```

---

## 10. Tổng kết và Best Practices

### 10.1 Firewall Design Principles

| Principle | Ý nghĩa |
|---|---|
| **Default Deny** | Mọi thứ bị block trừ khi explicitly allowed |
| **Least Privilege** | Chỉ mở ports/IPs thực sự cần |
| **Defense in Depth** | Nhiều layers (network FW + host FW + app FW) |
| **Stateful over Stateless** | Dùng conntrack cho security + simplicity |
| **Log then Drop** | Log trước khi drop (debugging, forensics) |
| **Fail Closed** | Nếu firewall lỗi → block all (không fail open) |

### 10.2 iptables Production Template

```bash
#!/bin/bash
# Production server firewall

# Flush existing rules
iptables -F
iptables -X
iptables -t nat -F

# Default policies
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT ACCEPT

# Loopback
iptables -A INPUT -i lo -j ACCEPT

# Stateful
iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -m conntrack --ctstate INVALID -j DROP

# Anti-spoofing
iptables -A INPUT -s 127.0.0.0/8 ! -i lo -j DROP
iptables -A INPUT -s 0.0.0.0/8 -j DROP
iptables -A INPUT -s 224.0.0.0/4 -j DROP
iptables -A INPUT -s 240.0.0.0/4 -j DROP

# ICMP (limited)
iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT

# SSH (rate limited, from trusted IPs)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -m conntrack --ctstate NEW \
  -m limit --limit 3/min --limit-burst 3 -j ACCEPT

# Web services
iptables -A INPUT -p tcp -m multiport --dports 80,443 -m conntrack --ctstate NEW -j ACCEPT

# Monitoring
iptables -A INPUT -p tcp --dport 9090 -s 10.0.0.0/8 -j ACCEPT  # Prometheus
iptables -A INPUT -p udp --dport 161 -s 10.0.0.100 -j ACCEPT   # SNMP from NMS

# Log dropped
iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "DROPPED: "
iptables -A INPUT -j DROP

# Save rules
iptables-save > /etc/iptables/rules.v4
```

### 10.3 Migration Path: iptables → nftables

```bash
# Check current tool
which iptables
# /usr/sbin/iptables → might be iptables-nft (nftables backend!)

# Translate iptables rules to nftables
iptables-save | iptables-restore-translate -f /etc/nftables.conf

# Or use iptables-translate for single rules
iptables-translate -A INPUT -p tcp --dport 22 -j ACCEPT
# → nft add rule ip filter INPUT tcp dport 22 counter accept
```

### 10.4 Tài liệu tham khảo

| Tài liệu | Nội dung |
|---|---|
| netfilter.org | Official Netfilter/iptables/nftables documentation |
| nftables wiki | nftables wiki (wiki.nftables.org) |
| iptables(8) man page | iptables command reference |
| RFC 2979 | Behavior of and Requirements for Internet Firewalls |
| CIS Benchmarks | Firewall rules best practices |
| Cloudflare Blog (conntrack tales) | Connection tracking deep dive |

### 10.5 Câu hỏi ôn tập

1. Stateless vs Stateful firewall: khác nhau thế nào? Cho ví dụ.
2. iptables có mấy tables? Mỗi table dùng làm gì?
3. Packet đi vào server đi qua chains nào? Đi qua (forward) thì sao?
4. DROP vs REJECT: khi nào dùng cái nào?
5. Conntrack states (NEW, ESTABLISHED, RELATED): ý nghĩa?
6. Tại sao nftables thay thế iptables? Ưu điểm chính?
7. `nf_conntrack_max` là gì? Khi nào cần tăng?
8. Viết iptables rules cho: web server (80/443), SSH chỉ từ 10.0.0.0/8, drop tất cả.
9. NAT MASQUERADE dùng khi nào? Khác SNAT thế nào?
10. "Default deny" policy nghĩa là gì? Tại sao quan trọng?

---

*Bài viết được tham khảo từ netfilter.org documentation, nftables wiki, Linux kernel networking documentation, RFC 2979, Cloudflare engineering blog, và CIS Benchmarks.*
