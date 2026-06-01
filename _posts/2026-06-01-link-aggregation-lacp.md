---
layout: post
title: "Link Aggregation — LACP — Gộp nhiều link vật lý thành 1 link logic"
subtitle: "Hiểu sâu về IEEE 802.3ad/802.1AX, LACP Protocol, Load Balancing và Failover"
tags: [networking, lacp, link-aggregation, layer2, redundancy, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| IEEE 802.1AX-2020 | Link Aggregation Standard (formerly 802.3ad) |
| IEEE 802.3ad-2000 | Original Link Aggregation Specification |
| Cisco — EtherChannel Configuration Guide | LAG implementation on Catalyst/Nexus |
| RFC 7130 | Bidirectional Forwarding Detection (BFD) for LAG |
| AWS — Elastic Load Balancing | Cloud equivalent concepts |

---

## 1. Giới thiệu — Tại sao cần biết Link Aggregation?

### Ví dụ đời thường: Mở rộng đường cao tốc

Hãy tưởng tượng đường cao tốc Hà Nội - Hải Phòng chỉ có **1 làn mỗi chiều**. Khi lưu lượng tăng, bạn có 2 lựa chọn:

1. **Xây đường mới lớn hơn** (upgrade 1G → 10G) — ĐẮT, phải thay toàn bộ hạ tầng
2. **Mở thêm làn** (bond 2× 1G = 2G bandwidth) — RẺ, dùng cáp sẵn có

Link Aggregation = **mở thêm làn đường** — gộp nhiều link vật lý (cables) thành 1 link logic với bandwidth cao hơn và redundancy tốt hơn.

### Concrete scenario

Bạn có server database quan trọng kết nối switch qua 1 cable 1Gbps. Vấn đề:
- **Bandwidth không đủ:** 50 users query đồng thời → bottleneck
- **Single point of failure:** Cable đứt → server mất kết nối → toàn company down

**Giải pháp:** Cắm 4 cables 1Gbps → bond thành 1 link 4Gbps:
- Bandwidth: 4× more
- Redundancy: 1 cable đứt → vẫn có 3Gbps (giảm 25%, không die)

### Vấn đề Link Aggregation giải quyết

| Vấn đề | Giải pháp LAG |
|---------|--------------|
| Bandwidth không đủ | Gộp N links = N× bandwidth |
| Single point of failure | 1 link die → traffic failover sang links còn lại |
| Upgrade cost cao | Dùng nhiều port rẻ thay 1 port đắt (4×1G thay 1×10G) |
| STP blocking links | LAG = 1 logical link → STP không block |
| Uneven load distribution | LACP load balancing algorithms |

---

## 2. Link Aggregation là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**Link Aggregation (LAG)** hay **EtherChannel** (Cisco) là kỹ thuật gộp **2-8 cables mạng vật lý** thành **1 kết nối logic duy nhất**. Hệ điều hành và protocols nhìn thấy chỉ 1 interface, nhưng bên dưới có nhiều cables hoạt động song song.

### Analogy: Nhiều thợ sơn cùng sơn 1 bức tường

```
┌─────────────────────────────────────────────────────────────┐
│        LINK AGGREGATION = NHIỀU THỢ SƠN CÙNG LÀM           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1 cable 1Gbps = 1 thợ sơn → sơn xong 1 tường mất 4 giờ  │
│                                                              │
│  4 cables 1Gbps bundled = 4 thợ sơn:                       │
│  - Mỗi người sơn 1 mảng (load distribution)               │
│  - Xong trong 1 giờ! (4× faster throughput)                │
│  - 1 thợ nghỉ ốm → 3 thợ vẫn tiếp tục (redundancy)      │
│                                                              │
│  QUAN TRỌNG: Mỗi "bức thư" chỉ do 1 thợ mang             │
│  (1 flow = 1 physical link — KHÔNG split 1 flow!)          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### ASCII Diagram

```
TRƯỚC (2 links riêng biệt):
                    ┌── Link 1 (1G) → STP có thể BLOCK! ──┐
  Server ──────────┤                                        ├── Switch
                    └── Link 2 (1G) → STP BLOCKS link này! ─┘
  Kết quả: Chỉ 1G active, 1G wasted!

SAU (Link Aggregation):
                    ┌── Cable 1 (1G) ──┐
  Server ──────────┤── Cable 2 (1G) ──┼── Switch
    bond0 (2G)     ├── Cable 3 (1G) ──┤    Port-channel1 (4G)
                    └── Cable 4 (1G) ──┘
  Kết quả: 4G bandwidth, STP sees 1 link → no blocking!
```

---

## 3. LACP — Link Aggregation Control Protocol

### 3.1 Static LAG vs LACP

| Feature | Static LAG (Manual) | LACP (Dynamic) |
|---------|-------------------|----------------|
| Negotiation | Không — force member ports | LACP PDU exchange |
| Detect misconfig | KHÔNG — có thể loop! | Tự detect và protect |
| Partner detection | KHÔNG biết partner có LAG không | Verify cả 2 bên agree |
| Hot-standby | Không | Có (active + standby members) |
| Cisco mode | `channel-group 1 mode on` | `channel-group 1 mode active` |

**⚠️ Luôn dùng LACP!** Static LAG nguy hiểm — nếu 1 bên không cấu hình LAG, traffic loop!

### 3.2 LACP Operation

```
LACP PDU (LACPDU):
- Gửi mỗi 1 giây (fast) hoặc 30 giây (slow)
- Destination MAC: 01:80:C2:00:00:02
- EtherType: 0x8809 (Slow Protocols)
- Chứa: Actor info + Partner info + Collector info

LACP Modes:
- Active: Gửi LACPDU chủ động → "Tôi muốn làm LAG!"
- Passive: Chỉ reply LACPDU → "OK nếu bạn muốn"

Kết quả:
Active ↔ Active = LAG formed ✓
Active ↔ Passive = LAG formed ✓
Passive ↔ Passive = LAG NOT formed ✗ (không ai khởi tạo)
```

### 3.3 LACP System Priority và Port Priority

```
LACP System ID = System Priority (2 bytes) + System MAC (6 bytes)
Default System Priority: 32768

Switch có System Priority THẤP hơn → decision maker:
- Quyết định port nào active/standby
- Quyết định max members

Port Priority: Default 32768
- Port có priority THẤP hơn → được chọn active trước
- Nếu LAG chỉ cho max 8 ports nhưng có 10 eligible:
  → 8 ports priority thấp nhất = active
  → 2 ports còn lại = hot-standby
```

---

## 4. Load Balancing — Traffic đi cable nào?

### Ví dụ đời thường

4 thợ sơn cùng sơn tường, nhưng KHÔNG phải cắt bức thư ra 4 mảnh! Thay vào đó, phân công theo "chủ đề":
- Thư từ Sales → Thợ 1 mang
- Thư từ Engineering → Thợ 2 mang
- Thư gửi cho Server A → Thợ 3 mang
- Thư từ IP x.x.x.10 → Thợ 4 mang

### 4.1 Load Balancing Algorithms

| Method | Hash based on | Best for |
|--------|--------------|----------|
| src-mac | Source MAC address | Nhiều PCs → 1 server |
| dst-mac | Destination MAC | 1 PC → nhiều servers |
| src-dst-mac | Source + Dest MAC | General purpose |
| src-ip | Source IP | Nhiều clients → 1 destination |
| dst-ip | Destination IP | 1 client → nhiều destinations |
| src-dst-ip | Source + Dest IP | General (recommended) |
| src-dst-ip-port | Src IP + Dst IP + L4 ports | Best distribution (many flows) |

```
Ví dụ src-dst-ip hash:
Flow 1: 10.0.0.1 → 10.0.0.100 → hash = 3 → Cable 3
Flow 2: 10.0.0.2 → 10.0.0.100 → hash = 1 → Cable 1
Flow 3: 10.0.0.3 → 10.0.0.100 → hash = 4 → Cable 4
Flow 4: 10.0.0.4 → 10.0.0.100 → hash = 2 → Cable 2

→ 4 flows phân đều 4 cables! ✓ Optimal!

NHƯNG:
Flow 5: 10.0.0.1 → 10.0.0.100 → hash = 3 → Cable 3 (SAME as Flow 1!)
→ Cùng src+dst IP → luôn cùng cable (flow integrity)
→ 1 flow KHÔNG BAO GIỜ exceed 1 link bandwidth
```

**Quan trọng:** 1 TCP connection **KHÔNG** bao giờ bị split qua nhiều cables → tránh out-of-order packets!

### 4.2 Kiểm tra load distribution

```bash
# Cisco:
Switch# show etherchannel load-balance
# Current: src-dst-ip

# Thay đổi:
Switch(config)# port-channel load-balance src-dst-ip-port

# Linux bonding:
cat /proc/net/bonding/bond0
# Bonding Mode: IEEE 802.3ad Dynamic link aggregation
# Transmit Hash Policy: layer3+4 (1)

# Xem traffic per-slave:
cat /proc/net/bonding/bond0 | grep -A3 "Slave Interface"
```

---

## 5. Configuration — Cấu hình chi tiết

### 5.1 Cisco EtherChannel (LACP)

```bash
# Switch A:
Switch-A(config)# interface range GigabitEthernet0/1-4
Switch-A(config-if-range)# channel-group 1 mode active
Switch-A(config-if-range)# exit

Switch-A(config)# interface Port-channel1
Switch-A(config-if)# switchport mode trunk
Switch-A(config-if)# switchport trunk allowed vlan 10,20,30

# Switch B: (same)
Switch-B(config)# interface range GigabitEthernet0/1-4
Switch-B(config-if-range)# channel-group 1 mode active

# Verify:
Switch# show etherchannel summary
# Group  Port-channel  Protocol    Ports
# 1      Po1(SU)       LACP        Gi0/1(P) Gi0/2(P) Gi0/3(P) Gi0/4(P)
# SU = Layer2, in use. P = in port-channel (bundled)

Switch# show etherchannel port-channel
Switch# show lacp neighbor
Switch# show lacp internal
```

### 5.2 Linux Bonding (LACP — mode 4)

```bash
# Method 1: Using ip commands
sudo ip link add bond0 type bond mode 802.3ad
sudo ip link set bond0 up
sudo ip link set eth0 master bond0
sudo ip link set eth1 master bond0
sudo ip link set eth2 master bond0
sudo ip link set eth3 master bond0
sudo ip addr add 10.0.0.1/24 dev bond0

# Configure LACP rate and hash:
echo "layer3+4" > /sys/class/net/bond0/bonding/xmit_hash_policy
echo "fast" > /sys/class/net/bond0/bonding/lacp_rate

# Method 2: Netplan (Ubuntu)
# /etc/netplan/01-bond.yaml:
# network:
#   bonds:
#     bond0:
#       interfaces: [eth0, eth1, eth2, eth3]
#       parameters:
#         mode: 802.3ad
#         lacp-rate: fast
#         transmit-hash-policy: layer3+4
#       addresses: [10.0.0.1/24]

# Verify:
cat /proc/net/bonding/bond0
```

### 5.3 Linux Bonding Modes

| Mode | Name | Load Balance | Failover | Cần switch support? |
|------|------|-------------|----------|-------------------|
| 0 | balance-rr | Round-robin | Yes | No (nhưng có issues) |
| 1 | active-backup | No (1 active) | Yes | No |
| 2 | balance-xor | XOR hash | Yes | No |
| 3 | broadcast | All slaves | Yes | No |
| 4 | **802.3ad (LACP)** | Hash-based | Yes | **YES (recommended)** |
| 5 | balance-tlb | Adaptive TX | Yes | No |
| 6 | balance-alb | Adaptive TX+RX | Yes | No |

---

## 6. Failover — Chuyện gì xảy ra khi 1 link die?

### 6.1 LACP Failover Timeline

```
t=0:    Link 3 bị đứt cable
t=0.05s: Physical layer detect link down
t=0.1s:  LACP removes link 3 from bundle
t=0.1s:  Hash redistribution: flows on link 3 → move to links 1,2,4
t=0.2s:  Traffic restored on remaining 3 links (75% capacity)

Total failover time: < 200ms (with LACP fast timers)!
```

### 6.2 LACP Fast vs Slow

| Parameter | Fast | Slow |
|-----------|------|------|
| LACPDU interval | 1 second | 30 seconds |
| Timeout (3× interval) | 3 seconds | 90 seconds |
| Detection time | < 3 seconds | < 90 seconds |
| CPU overhead | Higher | Lower |
| Recommended | Yes (critical links) | Legacy only |

---

## 7. Link Aggregation trong AWS

### 7.1 AWS Concepts

| LAG Concept | AWS Equivalent |
|-------------|---------------|
| LACP bond | ELB (Elastic Load Balancing) — khác concept! |
| Multiple uplinks | Multiple ENIs trên 1 instance |
| Bandwidth aggregation | Instance type bandwidth (not aggregatable) |
| Physical LAG | AWS Direct Connect LAG |
| Failover | Multi-AZ redundancy |

### 7.2 AWS Direct Connect LAG

```
AWS hỗ trợ LAG cho Direct Connect:

┌─── On-premises ───────┐       ┌─── AWS ─────────────┐
│                        │       │                      │
│  Router ──── Port 1 ──┼───────┼── DX Port 1 ──┐    │
│         ├── Port 2 ──┼───────┼── DX Port 2 ──┼ LAG│
│         ├── Port 3 ──┼───────┼── DX Port 3 ──┤    │
│         └── Port 4 ──┼───────┼── DX Port 4 ──┘    │
│         (LAG bundle)  │       │ (AWS DX LAG bundle) │
│                        │       │                      │
│  4× 10Gbps = 40Gbps  │       │  Redundancy +        │
│  logical bandwidth     │       │  bandwidth           │
└────────────────────────┘       └──────────────────────┘

AWS CLI:
aws directconnect create-lag \
  --number-of-connections 4 \
  --location "EqDC2" \
  --connections-bandwidth "10Gbps" \
  --lag-name "MyLAG"
```

### 7.3 EC2 Enhanced Networking

Trên EC2, bạn KHÔNG thể bond ENIs. Thay vào đó:
- Instance type quyết định bandwidth (m5.xlarge = 10Gbps)
- Muốn nhiều hơn? → Chọn instance type lớn hơn
- Hoặc dùng **Placement Group** + **EFA** cho 100Gbps

---

## 8. Tình huống thực tế

### Scenario 1: Server Farm — Bond NIC cho database server

```bash
# DB Server có 4 NICs → bond thành 1 interface 4Gbps:
# /etc/netplan/01-bond.yaml
network:
  bonds:
    bond0:
      interfaces: [ens1, ens2, ens3, ens4]
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        transmit-hash-policy: layer3+4
        mii-monitor-interval: 100
      addresses: [10.0.1.100/24]
      gateway4: 10.0.1.1
```

### Scenario 2: Switch Uplinks — 2×10G LACP bond

```bash
# Access Switch → Core Switch:
# 2× 10G SFP+ ports → 20Gbps logical uplink

Switch(config)# interface range TenGigabitEthernet0/1-2
Switch(config-if-range)# channel-group 1 mode active
Switch(config-if-range)# channel-protocol lacp

Switch(config)# interface Port-channel1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan all
```

### Scenario 3: Hypervisor — VM traffic distribution

```
ESXi host có 4 uplinks → LAG to switch:
- VM-A traffic → link 1 (based on vSwitch hash)
- VM-B traffic → link 2
- VM-C traffic → link 3
- vMotion traffic → link 4

Nếu link 2 die → VM-B auto-migrates to link 1 or 3
```

---

## 9. Bài tập thực hành

### Exercise 1: Linux Bond Setup

```bash
# Setup LACP bond trên Linux:
sudo modprobe bonding
sudo ip link add bond0 type bond mode 802.3ad
sudo ip link set eth1 down
sudo ip link set eth2 down
sudo ip link set eth1 master bond0
sudo ip link set eth2 master bond0
sudo ip link set bond0 up
sudo ip link set eth1 up
sudo ip link set eth2 up
sudo ip addr add 192.168.1.100/24 dev bond0

# Verify:
cat /proc/net/bonding/bond0
# Check: "802.3ad info" section shows LACP partner

# Test failover:
sudo ip link set eth1 down   # Simulate cable pull
ping 192.168.1.1             # Should continue via eth2!
sudo ip link set eth1 up     # Restore
```

### Exercise 2: Cisco EtherChannel

```bash
# Lab: 2 switches, 2 links between them
Switch-A(config)# interface range gi0/1-2
Switch-A(config-if-range)# channel-group 1 mode active

Switch-B(config)# interface range gi0/1-2
Switch-B(config-if-range)# channel-group 1 mode active

# Verify:
Switch# show etherchannel summary
Switch# show etherchannel load-balance
Switch# show lacp neighbor
Switch# show lacp internal

# Test: shutdown 1 member → verify traffic continues
Switch(config)# interface gi0/1
Switch(config-if)# shutdown
# → Po1 still up with 1 member!
```

### Exercise 3: Observe Load Distribution

```bash
# Generate traffic from multiple sources:
# From 4 different IPs → same destination
# Use iperf3 parallel streams:
iperf3 -c 10.0.0.1 -P 4 -t 60

# Monitor per-interface traffic:
watch -n 1 'cat /proc/net/dev | grep -E "eth[12]"'
# → Should see traffic on both interfaces

# If all traffic goes to 1 link:
# → Change hash policy!
echo "layer3+4" > /sys/class/net/bond0/bonding/xmit_hash_policy
```

---

## 10. Tóm tắt & Tài liệu đọc thêm

### Key Points

| # | Concept | Điểm quan trọng |
|---|---------|-----------------|
| 1 | LAG | Gộp 2-8 physical links → 1 logical link |
| 2 | LACP | Dynamic protocol, luôn dùng thay static LAG |
| 3 | Benefits | Bandwidth × N, redundancy, STP sees 1 link |
| 4 | Load Balance | Per-flow (NOT per-packet!) based on hash |
| 5 | Hash | src-dst-ip-port recommended for best distribution |
| 6 | Failover | < 200ms with LACP fast timers |
| 7 | 1 flow limit | 1 TCP connection NEVER exceeds 1 physical link bandwidth |
| 8 | Linux | mode 4 (802.3ad) = LACP |
| 9 | AWS | Direct Connect LAG supports LACP |

### Tài liệu đọc thêm

| # | Tài liệu | Link/Reference |
|---|----------|---------------|
| 1 | IEEE 802.1AX-2020 | https://standards.ieee.org/ieee/802.1AX/7605/ |
| 2 | Cisco EtherChannel Guide | https://www.cisco.com/c/en/us/tech/lan-switching/etherchannel/index.html |
| 3 | Linux Bonding Documentation | https://www.kernel.org/doc/Documentation/networking/bonding.txt |
| 4 | AWS Direct Connect LAG | https://docs.aws.amazon.com/directconnect/latest/UserGuide/lags.html |

---

**Bài tiếp theo**: [ARP — Address Resolution Protocol — Cầu nối giữa IP và MAC](/2026-06-01-arp-address-resolution-protocol)
