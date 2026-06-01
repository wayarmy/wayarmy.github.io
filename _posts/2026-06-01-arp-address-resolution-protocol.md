---
layout: post
title: "ARP — Address Resolution Protocol — Cầu nối giữa IP và MAC"
subtitle: "Hiểu sâu về ARP Request/Reply, ARP Cache, Gratuitous ARP, Proxy ARP và ARP Spoofing"
tags: [networking, arp, layer2, layer3, security, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 826 | An Ethernet Address Resolution Protocol (David C. Plummer, 1982) |
| RFC 5227 | IPv4 Address Conflict Detection |
| RFC 5765 | Security Threats for ARP |
| RFC 1027 | Using ARP to Implement Transparent Subnet Gateways (Proxy ARP) |
| Tanenbaum — Computer Networks, 6th Ed. | Chapter 5: Network Layer — ARP |
| Stevens — TCP/IP Illustrated, Vol. 1 | Chapter 4: ARP |

---

## 1. Giới thiệu — Tại sao cần biết ARP?

### Ví dụ đời thường: Biết tên nhưng không biết mặt

Hãy tưởng tượng bạn biết **tên người** cần gặp (IP Address = "Anh Hùng"), nhưng đang ở trong phòng đông 50 người và **không biết mặt** ai là Anh Hùng (không biết MAC Address). Bạn phải:

1. **Hét lên giữa phòng:** "Ai là Anh Hùng? Anh Hùng nào đang ở đây?" (ARP Request — broadcast)
2. **Anh Hùng trả lời:** "Tôi đây! Mặt tôi thế này!" (ARP Reply — unicast)
3. **Bạn ghi nhớ:** "OK, mặt Anh Hùng = [hình ảnh]" (ARP Cache entry)
4. **Lần sau:** Không cần hét nữa — nhớ mặt rồi, nói trực tiếp!

**ARP = quá trình "tìm mặt" (MAC Address) khi chỉ biết "tên" (IP Address).**

### Concrete scenario: "Hãy tưởng tượng bạn đang..."

Laptop bạn (IP: 192.168.1.100) muốn ping router (IP: 192.168.1.1). Vấn đề: để gửi Ethernet frame, cần **Destination MAC** của router. Laptop chỉ biết IP, KHÔNG biết MAC!

```
Laptop nghĩ: "Tôi muốn ping 192.168.1.1, nhưng MAC nó là gì?"
  ↓
Laptop gửi: ARP Request (broadcast FF:FF:FF:FF:FF:FF)
  "Ai có IP 192.168.1.1? Tôi là 192.168.1.100 (MAC: AA:AA:AA:AA:AA:AA)"
  ↓
Router nhận broadcast, thấy IP mình → Reply (unicast):
  "IP 192.168.1.1 là tôi! MAC tôi: BB:BB:BB:BB:BB:BB"
  ↓
Laptop: Lưu vào ARP cache: 192.168.1.1 → BB:BB:BB:BB:BB:BB
  ↓
Bây giờ mới có thể tạo Ethernet frame với Dst MAC = BB:BB...
```

### Vấn đề ARP giải quyết

| Vấn đề | Giải pháp ARP |
|---------|--------------|
| IP biết, MAC không biết | ARP Request broadcast → nhận MAC từ Reply |
| Không thể gửi frame mà không biết MAC | ARP bridge IP layer → Ethernet layer |
| Cache để tránh broadcast mỗi lần | ARP Cache (timeout default 60-300s) |
| Thông báo IP mới/đổi MAC | Gratuitous ARP |
| Cross-subnet communication qua router | Proxy ARP |

---

## 2. ARP là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**ARP (Address Resolution Protocol)** là "dịch vụ danh bạ" trong mạng nội bộ — nó **dịch** (resolve) từ địa chỉ IP (mà mọi người biết) sang địa chỉ MAC (mà switch cần để forward frame).

### Analogy: Danh bạ điện thoại nội bộ

```
┌─────────────────────────────────────────────────────────────┐
│            ARP = DANH BẠ ĐIỆN THOẠI                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Bạn biết:   "Tôi muốn gọi cho Phòng Kế Toán"            │
│  Không biết:  Số điện thoại nội bộ (extension) là gì?      │
│                                                              │
│  Giải pháp 1: Hét to "Phòng Kế Toán extension mấy?"       │
│  (= ARP Request broadcast)                                  │
│                                                              │
│  Phòng KT trả lời: "Extension 3456!"                       │
│  (= ARP Reply unicast)                                      │
│                                                              │
│  Bạn ghi vào sổ: "KT = ext 3456" (= ARP Cache)           │
│  Lần sau gọi trực tiếp, không cần hỏi lại!               │
│                                                              │
│  ┌────────────────────────┐                                 │
│  │ ARP Cache (Sổ danh bạ) │                                │
│  ├────────────────────────┤                                 │
│  │ IP Address → MAC Addr  │                                │
│  │ 192.168.1.1 → AA:BB:CC│                                │
│  │ 192.168.1.5 → DD:EE:FF│                                │
│  └────────────────────────┘                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Vị trí ARP trong mô hình mạng

```
┌─────────────────────────────────────────────────┐
│  Layer 3 (IP): "Tôi muốn gửi cho IP 10.0.0.5" │
│       │                                          │
│       │ "Nhưng MAC 10.0.0.5 là gì?"            │
│       ▼                                          │
│  ┌─────────────────────────────────────────┐    │
│  │         ARP (Layer 2.5)                  │    │
│  │   IP → MAC resolution                   │    │
│  │   ARP Request (broadcast)               │    │
│  │   ARP Reply (unicast)                   │    │
│  └─────────────────────────────────────────┘    │
│       │                                          │
│       ▼                                          │
│  Layer 2 (Ethernet): Frame với Dst MAC chính xác│
└─────────────────────────────────────────────────┘
```

---

## 3. ARP Packet Format — Cấu trúc chi tiết

### 3.1 ARP trong Ethernet Frame

```
┌──────────────────────── Ethernet Frame ────────────────────────────┐
│                                                                     │
│  Dst MAC        │ Src MAC       │ EtherType │ ARP Packet  │ Padding│ FCS │
│  (6 bytes)      │ (6 bytes)     │ 0x0806    │ (28 bytes)  │ (18B)  │(4B) │
│                 │               │           │             │        │     │
│  FF:FF:FF:FF:FF:FF (broadcast)  │           │             │        │     │
│  cho ARP Request                │           │             │        │     │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 ARP Packet (28 bytes cho IPv4 over Ethernet)

```
┌───────────────────────────────────────────────────────────┐
│ Hardware Type │ Protocol Type │ HW Addr Len │ Proto Len  │
│ (2 bytes)     │ (2 bytes)     │ (1 byte)    │ (1 byte)   │
│ 0x0001=Ether  │ 0x0800=IPv4   │ 6           │ 4          │
├───────────────┴───────────────┴─────────────┴────────────┤
│ Operation (2 bytes)                                       │
│ 1 = ARP Request                                          │
│ 2 = ARP Reply                                            │
├───────────────────────────────────────────────────────────┤
│ Sender Hardware Address (6 bytes) ← MAC người gửi       │
├───────────────────────────────────────────────────────────┤
│ Sender Protocol Address (4 bytes) ← IP người gửi        │
├───────────────────────────────────────────────────────────┤
│ Target Hardware Address (6 bytes) ← MAC người nhận       │
│   (00:00:00:00:00:00 trong Request — chưa biết!)        │
├───────────────────────────────────────────────────────────┤
│ Target Protocol Address (4 bytes) ← IP người nhận        │
└───────────────────────────────────────────────────────────┘
```

### 3.3 ARP Request vs Reply

| Field | ARP Request | ARP Reply |
|-------|------------|-----------|
| Ethernet Dst MAC | FF:FF:FF:FF:FF:FF (broadcast) | Unicast (requester's MAC) |
| Ethernet Src MAC | Sender's MAC | Responder's MAC |
| Operation | 1 (Request) | 2 (Reply) |
| Sender HW Addr | Requester's MAC | Responder's MAC |
| Sender Proto Addr | Requester's IP | Responder's IP |
| Target HW Addr | 00:00:00:00:00:00 (unknown!) | Requester's MAC |
| Target Proto Addr | Target IP (what we're looking for) | Requester's IP |

---

## 4. ARP Cache — "Bộ nhớ" tạm thời

### 4.1 Tại sao cần cache?

Nếu mỗi lần gửi packet đều phải ARP → broadcast storm! Cache lưu kết quả → hỏi 1 lần, dùng nhiều lần.

### 4.2 ARP Cache Operations

```bash
# Linux — xem ARP cache:
ip neigh show
# 192.168.1.1 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
# 192.168.1.5 dev eth0 lladdr 11:22:33:44:55:66 STALE

# States:
# REACHABLE: Vừa confirm gần đây
# STALE: Hết thời hạn, chưa verify lại
# DELAY: Đang đợi verification
# PROBE: Đang gửi unicast ARP để verify
# INCOMPLETE: Đã gửi ARP Request, chưa nhận Reply
# FAILED: ARP Request timeout — host unreachable

# Xóa entry:
sudo ip neigh del 192.168.1.1 dev eth0

# Thêm static entry:
sudo ip neigh add 192.168.1.1 lladdr aa:bb:cc:dd:ee:ff dev eth0

# Windows:
arp -a
arp -d *     # Clear cache
arp -s 192.168.1.1 aa-bb-cc-dd-ee-ff  # Static entry
```

### 4.3 ARP Timeout

| OS | Default ARP Cache Timeout |
|----|--------------------------|
| Linux | 60 seconds (then STALE, reachable_time) |
| Windows | 15-45 seconds (random) |
| macOS | 20 minutes |
| Cisco IOS | 4 hours (14400 seconds) |
| Juniper | 20 minutes |

---

## 5. Gratuitous ARP — "Tự giới thiệu" không ai hỏi

### 5.1 Gratuitous ARP là gì?

ARP Reply mà **không ai hỏi** — host tự broadcast MAC của mình:
- Sender IP = Target IP (cùng IP!)
- Gửi broadcast
- **Không cần có ARP Request trước**

```
Gratuitous ARP:
  Sender IP:  192.168.1.100
  Target IP:  192.168.1.100  ← CÙNG IP!
  Sender MAC: AA:BB:CC:DD:EE:FF
  Operation:  Reply (2) hoặc Request (1)
  Dst MAC:    FF:FF:FF:FF:FF:FF (broadcast)
```

### 5.2 Use Cases

| Scenario | Tại sao dùng Gratuitous ARP |
|----------|----------------------------|
| Interface comes up | "Tôi mới lên mạng! MAC tôi là XYZ" |
| IP conflict detection | "Ai có IP giống tôi không?" (nếu reply → conflict!) |
| VRRP/HSRP failover | Virtual IP chuyển sang backup router → update MAC |
| MAC address change | NIC mới, MAC mới → announce cho mọi người update cache |
| Load balancer failover | VIP chuyển server → announce MAC mới |

---

## 6. Proxy ARP — Router "trả lời hộ"

### 6.1 Tại sao cần Proxy ARP?

Khi host muốn liên lạc với IP **khác subnet** nhưng không có default gateway configured, router có thể "giả vờ" là target và trả lời ARP:

```
Host A: 10.0.1.5/24 (KHÔNG có default gateway!)
Host B: 10.0.2.5/24

Host A muốn ping 10.0.2.5:
- A nghĩ: "10.0.2.5 ở cùng network?" → KHÔNG! (khác /24)
- Bình thường: Gửi cho default gateway → nhưng A KHÔNG CÓ gateway!
- Proxy ARP: Router nghe ARP Request cho 10.0.2.5 → reply bằng MAC CỦA ROUTER

Kết quả: A gửi frame tới router (tưởng là host B) → Router route đến B
```

**Mặc định:** Proxy ARP **BẬT** trên nhiều router (Cisco default). Best practice: **TẮT** vì security risk.

---

## 7. ARP Spoofing — Mối nguy hiểm và cách phòng chống

### 7.1 ARP Spoofing Attack

```
Normal:
  PC-A ──── ARP: "Router MAC là gì?" ──→ Router (thật): "MAC=RR:RR:..."
  PC-A: Cache 192.168.1.1 = RR:RR:RR:RR:RR:RR ✓

Attack (ARP Spoofing/Poisoning):
  Attacker gửi Gratuitous ARP giả:
  "192.168.1.1 (Router IP) có MAC = AT:TA:CK:ER:MA:CC"
  
  PC-A: Cache 192.168.1.1 = AT:TA:CK:ER:MA:CC ← SAI!
  → PC-A gửi TẤT CẢ traffic cho Router → QUA ATTACKER!
  → Attacker đọc/modify traffic → forward cho Router (Man-in-the-Middle)
```

### 7.2 Phòng chống

| Method | Mô tả | Deployment |
|--------|--------|-----------|
| Static ARP | Hardcode IP→MAC mapping | Small networks only |
| DHCP Snooping | Switch builds trusted IP→MAC→Port table | Enterprise switches |
| Dynamic ARP Inspection (DAI) | Switch validates ARP against snooping table | Enterprise switches |
| 802.1X | Port-based authentication | Enterprise |
| ARP watch (arpwatch) | Monitor ARP changes, alert on anomalies | Linux hosts |
| VPN/Encryption | Even if MITM, traffic encrypted | All |

```bash
# Cisco DAI configuration:
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 10
Switch(config)# ip arp inspection vlan 10

Switch(config)# interface gi0/1
Switch(config-if)# ip dhcp snooping trust   ! Uplink to DHCP server
Switch(config-if)# ip arp inspection trust   ! Trusted port

# Linux — detect ARP spoofing:
sudo apt install arpwatch
sudo arpwatch -i eth0
# → Logs all ARP changes to syslog
# → Alerts on "flip flops" (MAC changes for same IP)
```

---

## 8. ARP trong AWS

### 8.1 AWS và ARP

Trong VPC, ARP hoạt động **khác biệt** so với physical network:

| Physical Network | AWS VPC |
|-----------------|---------|
| ARP broadcast flood | AWS PROXY all ARP (no broadcast flood) |
| ARP cache trên host | Có, nhưng AWS đảm bảo resolution |
| ARP spoofing possible | KHÔNG thể — AWS validates source |
| Gratuitous ARP | Handled by AWS for ENI changes |

**AWS Security:** ARP spoofing **KHÔNG THỂ XẢY RA** trong VPC vì:
- AWS enforces source MAC/IP validation
- Không có real L2 broadcast (AWS proxies ARP)
- Security Groups filter at hypervisor level

```bash
# Trên EC2, ARP vẫn hoạt động bình thường (cho application):
ip neigh show
# 10.0.1.1 dev eth0 lladdr 02:xx:xx:xx:xx:xx REACHABLE
# → Nhưng MAC này do AWS assign, ARP reply từ Nitro hypervisor
```

---

## 9. Bài tập thực hành

### Exercise 1: Observe ARP in action

```bash
# Terminal 1: Monitor ARP
sudo tcpdump -i eth0 -nn arp

# Terminal 2: Clear ARP cache and ping
sudo ip neigh flush all
ping -c 1 192.168.1.1

# Observe in Terminal 1:
# ARP, Request who-has 192.168.1.1 tell 192.168.1.100
# ARP, Reply 192.168.1.1 is-at aa:bb:cc:dd:ee:ff

# Check cache:
ip neigh show
```

### Exercise 2: ARP Cache manipulation

```bash
# 1. View current cache
ip neigh show

# 2. Delete specific entry
sudo ip neigh del 192.168.1.1 dev eth0

# 3. Add static entry (persists until reboot or manual delete)
sudo ip neigh add 192.168.1.1 lladdr aa:bb:cc:dd:ee:ff nud permanent dev eth0

# 4. Change entry
sudo ip neigh change 192.168.1.1 lladdr 11:22:33:44:55:66 dev eth0

# 5. Monitor changes in real-time
ip monitor neigh
```

### Exercise 3: Gratuitous ARP

```bash
# Send gratuitous ARP (using arping):
sudo arping -U -I eth0 192.168.1.100
# -U: Unsolicited ARP (gratuitous)
# → Broadcasts "192.168.1.100 is at [my MAC]"

# Use case: After changing MAC address, announce to network
sudo ip link set eth0 address 02:00:00:00:00:01
sudo arping -U -I eth0 -c 3 192.168.1.100
```

### Exercise 4: Detect ARP spoofing

```bash
# Install arpwatch
sudo apt install arpwatch

# Start monitoring
sudo arpwatch -i eth0 -f /var/lib/arpwatch/arp.dat

# Check logs for anomalies
grep "flip flop" /var/log/syslog
grep "changed ethernet address" /var/log/syslog

# Manual detection:
# If 2 IPs have same MAC → someone is spoofing!
ip neigh show | awk '{print $5}' | sort | uniq -d
```

### Exercise 5: Wireshark ARP analysis

```
Wireshark filters:
  arp                    → All ARP packets
  arp.opcode == 1       → Only ARP Requests
  arp.opcode == 2       → Only ARP Replies
  arp.src.proto_ipv4 == 192.168.1.1   → From specific IP
  arp.duplicate-address-detected  → Duplicate IP detection!
  
Bài tập:
1. Capture ARP → identify all devices in LAN
2. Find gratuitous ARPs → who just joined network?
3. Detect ARP storms → too many requests = problem!
4. Find duplicate IPs → 2 hosts claiming same IP
```

---

## 10. Tóm tắt & Tài liệu đọc thêm

### Key Points

| # | Concept | Điểm quan trọng |
|---|---------|-----------------|
| 1 | Purpose | Resolve IP → MAC within same LAN segment |
| 2 | Request | Broadcast (FF:FF:FF:FF:FF:FF) — "Who has IP X?" |
| 3 | Reply | Unicast — "IP X is at MAC Y" |
| 4 | Cache | Lưu IP→MAC mapping, timeout 60s-4hrs tùy OS |
| 5 | Gratuitous | Self-announce, IP conflict detection, failover |
| 6 | Proxy ARP | Router answers for hosts on other subnets |
| 7 | Spoofing | Fake ARP replies → MITM attack |
| 8 | Defense | DHCP Snooping + DAI, 802.1X, static ARP |
| 9 | AWS | ARP proxied by hypervisor, spoofing impossible |

### Tài liệu đọc thêm

| # | Tài liệu | Link/Reference |
|---|----------|---------------|
| 1 | RFC 826 — ARP | https://datatracker.ietf.org/doc/html/rfc826 |
| 2 | RFC 5227 — IPv4 Address Conflict Detection | https://datatracker.ietf.org/doc/html/rfc5227 |
| 3 | Stevens — TCP/IP Illustrated Vol 1, Ch 4 | ISBN: 978-0321336316 |
| 4 | Cisco — DAI Configuration | https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst6500/ios/15.1SY/config_guide/sup2T/15_1_sy_swcg_2T/dynarp.html |
| 5 | AWS VPC Networking | https://docs.aws.amazon.com/vpc/latest/userguide/ |

---

**Bài tiếp theo**: [IPv4 Header Deep Dive — Từng field, Fragmentation, và TTL](/2026-06-01-ipv4-header-deep-dive)
