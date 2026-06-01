---
layout: post
title: "Policy-Based Routing (PBR) - Source-Based Routing, Route-Maps và Multi-ISP Load Balancing"
date: 2026-06-01
categories: [networking]
tags: [pbr, policy-routing, route-map, source-routing, multi-isp, traffic-engineering]
---

## 1. Giới thiệu — Đi đường nào tuỳ người, tuỳ việc

Hãy tưởng tượng thành phố có **2 con đường** đến trung tâm:
- **Đường A** = Cao tốc — nhanh, phí cao (ISP MPLS — low latency, đắt)
- **Đường B** = Đường quốc lộ — chậm hơn, miễn phí (ISP Broadband — high bandwidth, rẻ)

**Routing thông thường (destination-based):**
- GPS chỉ xem **bạn muốn đến đâu** → chọn đường ngắn nhất
- Mọi xe đều đi cùng 1 đường (best route duy nhất)
- Không quan tâm xe chở gì, ai lái, đi từ đâu

**Policy-Based Routing:**
- Xe cứu thương (VoIP) → bắt buộc đi **Đường A** (fast, reliable)
- Xe tải (download/backup) → đi **Đường B** (cheap, high bandwidth)
- Xe từ khu vực VIP (executives) → đi **Đường A**
- Xe từ khu vực nhân viên (staff) → đi **Đường B**

**PBR = Routing dựa trên "chính sách" thay vì chỉ dựa trên đích đến.**

### So sánh nhanh

| Tiêu chí | Destination-Based Routing | Policy-Based Routing |
|---|---|---|
| Quyết định dựa trên | Destination IP only | Source IP, protocol, port, size... |
| Tất cả traffic | Cùng path (best route) | Khác path tuỳ policy |
| Flexibility | Thấp | Cao |
| Configuration | Đơn giản (routing table) | Phức tạp (route-map, ACL) |
| Performance | Nhanh (hardware TCAM) | Có thể chậm hơn (ACL lookup) |
| Use case | 95% traffic | Đặc biệt: multi-ISP, security, TE |

### Khi nào bạn gặp PBR?

1. **Dual-ISP office** — Web traffic qua ISP rẻ, VoIP qua ISP premium
2. **Branch office** — Guest WiFi ra Internet trực tiếp, Corporate qua VPN
3. **Security** — Traffic nhất định phải qua firewall/IDS
4. **Cost optimization** — Backup traffic qua ISP bandwidth lớn, rẻ
5. **Compliance** — Dữ liệu nhạy cảm phải đi path riêng (không qua public)

---

## 2. PBR là gì? — Giải thích cho người không biết IT

### Phép so sánh đời thường — Bảo vệ toà nhà

Tưởng tượng toà nhà văn phòng có **3 cổng ra**:
- **Cổng chính** = Ra đường lớn (ISP chính)
- **Cổng hầm xe** = Ra đường nhỏ (ISP phụ)
- **Cổng VIP** = Ra thẳng bãi đậu riêng (dedicated line)

**Routing thường** = Bảo vệ chỉ biết: "Ra ngoài? → Đi cổng chính."

**PBR** = Bảo vệ thông minh hơn:
- "Anh là giám đốc? → Đi **cổng VIP**"
- "Chị đi gửi hàng (nặng)? → Đi **cổng hầm xe** (gần bãi tải hàng)"
- "Bạn là khách? → Đi **cổng chính**"
- "Nhân viên IT mang server? → Đi **cổng hầm** (có thang máy hàng)"

Bảo vệ kiểm tra **ai bạn là** (source), **bạn mang gì** (protocol/port), rồi **chỉ đường** phù hợp.

### Định nghĩa kỹ thuật

**Policy-Based Routing (PBR)** là kỹ thuật forward packet dựa trên các tiêu chí do administrator định nghĩa (policy) thay vì chỉ dựa trên destination IP trong routing table.

PBR cho phép match traffic theo:
- **Source IP address** — traffic từ subnet nào
- **Destination IP address** — traffic đi đến đâu (kết hợp source)
- **Protocol** — TCP, UDP, ICMP...
- **Source/Destination port** — HTTP (80), HTTPS (443), VoIP (5060)...
- **Packet length** — packet nhỏ (interactive) vs lớn (bulk)
- **DSCP/ToS** — QoS marking
- **Incoming interface** — traffic vào từ interface nào

Sau khi match, PBR có thể:
- **Set next-hop** — forward đến gateway khác (bypass routing table)
- **Set interface** — gửi ra interface cụ thể
- **Set IP precedence/DSCP** — đánh dấu QoS
- **Set default next-hop** — dùng khi routing table không có route
- **Set VRF** — chuyển traffic sang VRF khác

### PBR trong OSI model

```
Packet arrives → Interface (inbound)
                      ↓
              ┌──────────────────┐
              │  PBR Check       │ ← Policy applied HERE
              │  (route-map)     │    (BEFORE routing table lookup)
              └──────────────────┘
                      ↓
              Match policy? ──YES──→ Forward theo policy
                      │                (set next-hop/interface)
                      NO
                      ↓
              ┌──────────────────┐
              │  Normal Routing  │ ← Regular routing table lookup
              │  Table Lookup    │
              └──────────────────┘
                      ↓
              Forward theo best route
```

**Quan trọng:** PBR được apply **TRƯỚC** routing table lookup! Nếu packet match policy → KHÔNG tra routing table.

---

## 3. Route-Maps — Công cụ chính của PBR

### Phép so sánh — Luật giao thông

Route-map giống **bảng luật giao thông**:
- Mỗi điều luật có **số thứ tự** (sequence number)
- Mỗi điều có **điều kiện** (match) và **hành động** (set)
- Nếu điều 1 không match → kiểm tra điều 2 → điều 3...
- Nếu không match điều nào → **default: routing table** (implicit deny)

### Cấu trúc Route-Map

```
route-map [NAME] [permit|deny] [sequence-number]
  match [condition 1]
  match [condition 2]      ← Tất cả conditions phải match (AND)
  set [action 1]
  set [action 2]           ← Tất cả actions được apply

route-map [NAME] [permit|deny] [sequence-number+]
  match [other condition]
  set [other action]
```

**Logic xử lý:**
```
Packet arrives
  ↓
Check sequence 10: match? → YES → apply set actions → DONE
  ↓ NO
Check sequence 20: match? → YES → apply set actions → DONE
  ↓ NO
Check sequence 30: match? → YES → apply set actions → DONE
  ↓ NO
... (tiếp tục)
  ↓ NO (không match gì)
Implicit DENY (PBR) → route normally
```

### Match Conditions (phổ biến nhất)

| Match Command | Mô tả | Ví dụ |
|---|---|---|
| `match ip address [ACL]` | Match source/dest IP (via ACL) | ACL cho phép 10.0.1.0/24 |
| `match ip address prefix-list` | Match IP prefix | prefix-list BRANCH1 |
| `match length [min] [max]` | Match packet length | length 0 500 (interactive) |
| `match interface` | Match incoming interface | interface Gi0/1 |
| `match ip next-hop` | Match current next-hop | next-hop from routing table |
| `match as-path` | Match BGP AS-path | Dùng trong BGP filtering |
| `match community` | Match BGP community | Dùng trong BGP |

### Set Actions (phổ biến nhất)

| Set Command | Mô tả | Ví dụ |
|---|---|---|
| `set ip next-hop [IP]` | Forward đến next-hop cụ thể | set ip next-hop 203.0.113.1 |
| `set ip next-hop verify-availability` | Kiểm tra next-hop alive | Fallback nếu dead |
| `set interface [iface]` | Forward ra interface | set interface Gi0/2 |
| `set ip default next-hop [IP]` | Dùng khi RIB không có route | Backup gateway |
| `set ip precedence [value]` | Set ToS precedence | QoS marking |
| `set ip dscp [value]` | Set DSCP value | set ip dscp ef |
| `set ip vrf [name]` | Route trong VRF khác | Inter-VRF routing |
| `set ip next-hop recursive [IP]` | Recursive next-hop | BGP next-hop |

### Ví dụ cơ bản — Dual ISP

```
Scenario:
  Subnet A (10.0.1.0/24) — Marketing → ISP1 (203.0.113.1)
  Subnet B (10.0.2.0/24) — Engineering → ISP2 (198.51.100.1)
  Default: ISP1

! Access Lists (match criteria)
access-list 101 permit ip 10.0.1.0 0.0.0.255 any     ! Marketing
access-list 102 permit ip 10.0.2.0 0.0.0.255 any     ! Engineering

! Route Map
route-map PBR-DUAL-ISP permit 10
 match ip address 101
 set ip next-hop 203.0.113.1        ! Marketing → ISP1

route-map PBR-DUAL-ISP permit 20
 match ip address 102
 set ip next-hop 198.51.100.1       ! Engineering → ISP2

! (Implicit deny — traffic không match → routing table)

! Apply PBR trên LAN interface (inbound)
interface GigabitEthernet0/0
 ip policy route-map PBR-DUAL-ISP
```

---

## 4. PBR Scenarios thực tế — Multi-ISP

### Scenario 1: Dual-ISP Office — Traffic by Type

```
Topology:
                     ┌──── ISP1 (MPLS) ────── Internet
  LAN ──── Router ──┤     203.0.113.1         Low latency
  10.0.0.0/16       │     (đắt, 100Mbps)      Premium
                     └──── ISP2 (Broadband) ── Internet  
                           198.51.100.1        High bandwidth
                           (rẻ, 1Gbps)         Best effort

Policy:
  - VoIP (UDP 5060, RTP 16384-32767) → ISP1 (low latency)
  - Video conferencing (TCP 443 Zoom/Teams) → ISP1
  - Web browsing (TCP 80, 443) → ISP2 (bandwidth)
  - Backup/Sync (large transfers) → ISP2
  - Default → ISP1
```

```
! Extended ACLs
ip access-list extended VOIP-TRAFFIC
 permit udp any any eq 5060
 permit udp any any range 16384 32767

ip access-list extended VIDEO-TRAFFIC
 permit tcp any any eq 443           ! Simplified — should match specific IPs

ip access-list extended WEB-TRAFFIC
 permit tcp any any eq 80
 permit tcp any any eq 443

ip access-list extended BACKUP-TRAFFIC
 permit tcp 10.0.0.0 0.0.255.255 host 52.1.2.3 eq 443   ! AWS S3

! Route Map
route-map DUAL-ISP permit 10
 description VoIP via ISP1 (premium)
 match ip address VOIP-TRAFFIC
 set ip next-hop verify-availability 203.0.113.1 1 track 1
 set ip next-hop 198.51.100.1   ! Fallback to ISP2

route-map DUAL-ISP permit 20
 description Backup via ISP2 (bandwidth)
 match ip address BACKUP-TRAFFIC
 set ip next-hop verify-availability 198.51.100.1 2 track 2
 set ip next-hop 203.0.113.1   ! Fallback to ISP1

route-map DUAL-ISP permit 30
 description Web via ISP2 (bandwidth)
 match ip address WEB-TRAFFIC
 set ip next-hop verify-availability 198.51.100.1 3 track 2

! sequence 40 implicit — default route via ISP1 (routing table)

! IP SLA for health monitoring
ip sla 1
 icmp-echo 203.0.113.1
 frequency 5
ip sla schedule 1 start-time now life forever

ip sla 2
 icmp-echo 198.51.100.1
 frequency 5
ip sla schedule 2 start-time now life forever

track 1 ip sla 1 reachability
track 2 ip sla 2 reachability

! Apply
interface GigabitEthernet0/0
 description LAN Interface
 ip address 10.0.0.1 255.255.0.0
 ip policy route-map DUAL-ISP
```

### Scenario 2: Source-Based Routing — Department Separation

```
Topology:
  VLAN 10 (HR: 10.0.10.0/24) ─────┐
  VLAN 20 (Finance: 10.0.20.0/24) ─┤──── Router ──┬── ISP1 (secure)
  VLAN 30 (Guest: 10.0.30.0/24) ───┘              └── ISP2 (general)

Policy:
  - HR + Finance → ISP1 (đi qua firewall, encrypted)
  - Guest → ISP2 (trực tiếp Internet, no corporate access)
```

```
ip access-list extended SECURE-DEPARTMENTS
 permit ip 10.0.10.0 0.0.0.255 any   ! HR
 permit ip 10.0.20.0 0.0.0.255 any   ! Finance

ip access-list extended GUEST-TRAFFIC
 permit ip 10.0.30.0 0.0.0.255 any   ! Guest

route-map SOURCE-PBR permit 10
 match ip address SECURE-DEPARTMENTS
 set ip next-hop 192.168.1.1      ! Next-hop = Firewall → ISP1

route-map SOURCE-PBR permit 20
 match ip address GUEST-TRAFFIC
 set ip next-hop 192.168.2.1      ! Next-hop = ISP2 directly
 set ip dscp default              ! Mark as best-effort

! Apply trên VLAN interfaces
interface Vlan10
 ip policy route-map SOURCE-PBR
interface Vlan20
 ip policy route-map SOURCE-PBR
interface Vlan30
 ip policy route-map SOURCE-PBR
```

### Scenario 3: Traffic Steering qua Firewall/IDS

```
Policy: Tất cả traffic từ DMZ đến Internal PHẢI qua Firewall

  DMZ (10.0.100.0/24) → Firewall (192.168.99.1) → Internal (10.0.0.0/16)

  Bình thường router sẽ forward trực tiếp (same router có cả 2 subnets)
  PBR force traffic đi qua firewall!
```

```
ip access-list extended DMZ-TO-INTERNAL
 permit ip 10.0.100.0 0.0.0.255 10.0.0.0 0.0.255.255

route-map FORCE-FIREWALL permit 10
 match ip address DMZ-TO-INTERNAL
 set ip next-hop 192.168.99.1    ! Firewall address

! Apply trên DMZ interface
interface GigabitEthernet0/2
 description DMZ Interface
 ip policy route-map FORCE-FIREWALL
```

---

## 5. PBR trên Linux — ip rule và ip route tables

### Phép so sánh — Nhiều bản đồ

Linux PBR giống có **nhiều bản đồ** (routing tables) và **luật** (rules) quyết định dùng bản đồ nào:

- Bản đồ "main" = Routing table chính
- Bản đồ "isp1" = Routes qua ISP1
- Bản đồ "isp2" = Routes qua ISP2
- Luật: "Traffic từ 10.0.1.0/24 → dùng bản đồ isp1"

### Linux Routing Policy Database (RPDB)

```
Linux có 3 components cho PBR:

1. Rules (ip rule) — quyết định dùng routing table nào
2. Multiple routing tables (ip route table X) — nhiều bảng routing
3. Priority — rules xử lý theo thứ tự priority (number nhỏ = ưu tiên cao)

Default rules:
  # ip rule show
  0:    from all lookup local        ← Loopback, broadcast
  32766: from all lookup main        ← Routing table chính
  32767: from all lookup default     ← Default table (thường trống)
```

### Cấu hình Linux PBR — Step by Step

#### Step 1: Tạo custom routing tables

```bash
# Đặt tên cho routing tables (optional nhưng dễ đọc)
echo "100 isp1" >> /etc/iproute2/rt_tables
echo "200 isp2" >> /etc/iproute2/rt_tables
```

#### Step 2: Thêm routes vào custom tables

```bash
# Table isp1: default route qua ISP1
ip route add default via 203.0.113.1 dev eth1 table isp1
ip route add 10.0.0.0/16 dev eth0 table isp1  # LAN subnet

# Table isp2: default route qua ISP2
ip route add default via 198.51.100.1 dev eth2 table isp2
ip route add 10.0.0.0/16 dev eth0 table isp2  # LAN subnet
```

#### Step 3: Tạo rules

```bash
# Traffic từ Marketing subnet → dùng table isp1
ip rule add from 10.0.1.0/24 table isp1 priority 100

# Traffic từ Engineering subnet → dùng table isp2
ip rule add from 10.0.2.0/24 table isp2 priority 200

# Traffic đến specific destination → specific table
ip rule add to 52.0.0.0/8 table isp1 priority 50  # AWS qua ISP1

# Traffic với specific fwmark (set by iptables) → specific table
ip rule add fwmark 1 table isp1 priority 150
ip rule add fwmark 2 table isp2 priority 160
```

#### Step 4: Dùng iptables để mark packets (advanced matching)

```bash
# Mark VoIP traffic (UDP 5060)
iptables -t mangle -A PREROUTING -p udp --dport 5060 -j MARK --set-mark 1

# Mark HTTP/HTTPS traffic
iptables -t mangle -A PREROUTING -p tcp --dport 80 -j MARK --set-mark 2
iptables -t mangle -A PREROUTING -p tcp --dport 443 -j MARK --set-mark 2

# Mark by source interface
iptables -t mangle -A PREROUTING -i wlan0 -j MARK --set-mark 2  # WiFi → ISP2
```

#### Step 5: Verify

```bash
# Show all rules
ip rule show
# 0:     from all lookup local
# 50:    to 52.0.0.0/8 lookup isp1
# 100:   from 10.0.1.0/24 lookup isp1
# 150:   from all fwmark 0x1 lookup isp1
# 160:   from all fwmark 0x2 lookup isp2
# 200:   from 10.0.2.0/24 lookup isp2
# 32766: from all lookup main
# 32767: from all lookup default

# Show routes in specific table
ip route show table isp1
ip route show table isp2

# Test which table a packet would use
ip route get 8.8.8.8 from 10.0.1.100
# → 8.8.8.8 from 10.0.1.100 via 203.0.113.1 dev eth1 table isp1

ip route get 8.8.8.8 from 10.0.2.100
# → 8.8.8.8 from 10.0.2.100 via 198.51.100.1 dev eth2 table isp2
```

### Persistent Configuration (Netplan / ifupdown)

```yaml
# /etc/netplan/01-pbr.yaml (Ubuntu 20.04+)
network:
  version: 2
  ethernets:
    eth1:
      addresses: [203.0.113.2/24]
      routes:
        - to: default
          via: 203.0.113.1
          table: 100
      routing-policy:
        - from: 10.0.1.0/24
          table: 100
          priority: 100
    eth2:
      addresses: [198.51.100.2/24]
      routes:
        - to: default
          via: 198.51.100.1
          table: 200
      routing-policy:
        - from: 10.0.2.0/24
          table: 200
          priority: 200
```

---

## 6. PBR Advanced — verify-availability và Failover

### Phép so sánh — Tài xế backup

Khi tài xế chính (ISP1) bị ốm:
- **Không có failover:** Hàng KHÔNG được giao (traffic drop)
- **Có failover:** Tự động gọi tài xế phụ (ISP2)

### verify-availability (Cisco)

```
! Nếu ISP1 down, traffic tự động chuyển sang ISP2

route-map PBR permit 10
 match ip address VOIP-TRAFFIC
 set ip next-hop verify-availability 203.0.113.1 1 track 1
 set ip next-hop 198.51.100.1    ! Backup next-hop (nếu track 1 down)

! Track object monitor ISP1
track 1 ip sla 1 reachability
  delay down 10 up 30    ! Down after 10s, Up after 30s (dampening)

ip sla 1
 icmp-echo 203.0.113.1 source-interface GigabitEthernet0/1
 frequency 5
 timeout 2000
ip sla schedule 1 start-time now life forever
```

**Logic:**
```
1. Router check: Track 1 UP?
   YES → set next-hop 203.0.113.1 (primary)
   NO  → set next-hop 198.51.100.1 (backup)
   
2. IP SLA monitor:
   - Ping 203.0.113.1 mỗi 5 giây
   - Nếu fail 2 lần liên tiếp (10s) → Track DOWN
   - Khi recover + stable 30s → Track UP
```

### Linux Failover với ip rule + health check

```bash
#!/bin/bash
# /usr/local/bin/pbr-failover.sh
# Chạy mỗi 5 giây qua cron hoặc systemd timer

ISP1_GW="203.0.113.1"
ISP2_GW="198.51.100.1"
TABLE_ISP1=100
TABLE_ISP2=200

check_gateway() {
    ping -c 2 -W 2 -I $2 $1 > /dev/null 2>&1
    return $?
}

# Check ISP1
if check_gateway $ISP1_GW eth1; then
    # ISP1 alive — ensure rules are normal
    ip route replace default via $ISP1_GW dev eth1 table $TABLE_ISP1
else
    # ISP1 dead — failover rules to ISP2
    ip route replace default via $ISP2_GW dev eth2 table $TABLE_ISP1
    logger "PBR Failover: ISP1 DOWN, rerouting via ISP2"
fi

# Check ISP2
if check_gateway $ISP2_GW eth2; then
    ip route replace default via $ISP2_GW dev eth2 table $TABLE_ISP2
else
    ip route replace default via $ISP1_GW dev eth1 table $TABLE_ISP2
    logger "PBR Failover: ISP2 DOWN, rerouting via ISP1"
fi
```

### set ip default next-hop vs set ip next-hop

```
! set ip next-hop: LUÔN dùng next-hop này (bỏ qua routing table)
route-map PBR permit 10
 match ip address 101
 set ip next-hop 203.0.113.1
 ! → Traffic LUÔN đi 203.0.113.1, kể cả khi routing table có route tốt hơn

! set ip default next-hop: Chỉ dùng khi routing table KHÔNG có route
route-map PBR permit 10
 match ip address 101
 set ip default next-hop 203.0.113.1
 ! → Router lookup routing table trước
 ! → Nếu có route → dùng route
 ! → Nếu không có route → dùng 203.0.113.1
```

**Thứ tự ưu tiên (Cisco):**
```
1. set ip next-hop (highest priority — override routing table)
2. Routing table lookup
3. set ip default next-hop (lowest — only if no route exists)
```

---

## 7. Local Policy Routing — Router tự áp dụng cho traffic của mình

### Vấn đề

PBR thường chỉ apply cho **transit traffic** (traffic đi QUA router). Traffic **từ router** (management, routing protocols, pings) KHÔNG bị ảnh hưởng bởi interface-level PBR.

### Giải pháp: Local Policy Routing

```
! Apply PBR cho traffic DO ROUTER TẠO RA

! Route-map cho local traffic
route-map LOCAL-PBR permit 10
 match ip address LOCAL-MGMT
 set ip next-hop 203.0.113.1

! Apply globally (không phải trên interface)
ip local policy route-map LOCAL-PBR
```

**Use case:**
- Router cần SSH/Telnet qua specific ISP
- SNMP traps phải đi qua management network
- Syslog messages phải đi qua secure path

### Linux — Local traffic PBR

```bash
# Trên Linux, ip rule "from" với router's own IP works cho local traffic:

# Router's IP on eth0: 10.0.0.1
ip rule add from 10.0.0.1 table isp1 priority 50

# Hoặc mark OUTPUT traffic
iptables -t mangle -A OUTPUT -p tcp --dport 22 -j MARK --set-mark 1
ip rule add fwmark 1 table isp1 priority 50
```

---

## 8. PBR trên Cloud — AWS, Azure, GCP

### AWS — Route Tables và Ingress Routing

```
AWS không có traditional PBR nhưng có alternatives:

1. Multiple Route Tables:
   - Mỗi subnet có thể associate với route table khác
   - Subnet A → Route table 1 (qua NAT GW 1)
   - Subnet B → Route table 2 (qua NAT GW 2)

2. Gateway Route Tables (Ingress Routing):
   - Apply trên Internet Gateway / VPN Gateway
   - Route incoming traffic đến specific ENI (firewall)
   
3. Transit Gateway Route Tables:
   - Mỗi attachment (VPC) có thể associate route table riêng
   - VPC-A traffic → Inspection VPC → Destination VPC
   - VPC-B traffic → Direct to Destination VPC

4. VPC Prefix Lists + Route Tables:
   - Managed prefix lists cho routing decisions
```

```
AWS Ingress Routing Example:
  - Traffic từ Internet → IGW → Route table trên IGW
  - Route table: 10.0.1.0/24 → Firewall ENI (eni-xxx)
  - Firewall inspect → forward → Destination EC2

  IGW Route Table:
    10.0.1.0/24 → eni-fw-abc123 (firewall)
    10.0.2.0/24 → eni-fw-abc123 (firewall)
```

### GCP — Policy-Based Routes

```
GCP native PBR (GA 2023):

gcloud compute network-firewall-policies rules create \
  --network=my-vpc \
  --priority=100 \
  --src-ip-ranges=10.0.1.0/24 \
  --dest-ip-ranges=0.0.0.0/0 \
  --next-hop-ilb=lb-firewall   # Forward matching traffic to ILB

# Policy-based routes
gcloud compute routes create pbr-to-firewall \
  --network=my-vpc \
  --priority=100 \
  --tags=needs-inspection \
  --dest-range=0.0.0.0/0 \
  --next-hop-ilb=projects/my-project/regions/us-central1/forwardingRules/fw-rule
```

### Azure — User Defined Routes (UDR)

```
Azure UDR = simplified PBR:

# Route table associated to subnet
az network route-table create \
  --name RT-Workload \
  --resource-group MyRG

# Custom route: override default
az network route-table route create \
  --route-table-name RT-Workload \
  --resource-group MyRG \
  --name ToInternet \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.99.4   # Firewall NVA

# Associate route table to subnet
az network vnet subnet update \
  --vnet-name MyVNet \
  --name WorkloadSubnet \
  --resource-group MyRG \
  --route-table RT-Workload
```

---

## 9. PBR Performance và Limitations

### Performance Impact

```
PBR processing order:
  1. Packet arrives on interface
  2. Input ACL check
  3. ★ PBR route-map evaluation ★  ← Additional processing
     - Match ACL lookup
     - Set action execution
  4. (If no PBR match) Normal routing lookup
  5. Output ACL check
  6. Forward

Performance considerations:
  - Mỗi packet phải match ACL → CPU/TCAM usage
  - Complex ACLs = more processing time
  - Hardware-accelerated PBR (modern switches) mitigates
  - Software-based PBR (old routers) = significant CPU hit
```

### Limitations

| Limitation | Mô tả | Workaround |
|---|---|---|
| Inbound only | PBR chỉ apply trên inbound interface | Restructure topology |
| No load balancing | Không thể split traffic 50/50 | CEF per-packet + PBR |
| ACL complexity | Large ACLs = TCAM exhaustion | Summarize, use prefix-lists |
| No session awareness | Stateless — forward/return path mismatch | Ensure symmetric routing |
| Fragmented packets | Fragments without L4 header miss PBR | Reassemble first |
| Local traffic | Interface PBR không apply cho router traffic | ip local policy |

### Asymmetric Routing Problem

```
Problem:
  Client (10.0.1.100) → Server (8.8.8.8)
  
  Outbound: PBR → ISP1 (203.0.113.1)
  Return:   Server replies to 203.0.113.2 (router's ISP1 IP)
            → Arrives on ISP1 interface → OK! ✓
  
  BUT if NAT is wrong:
  Outbound: PBR → ISP1, but NAT uses ISP2 IP (198.51.100.2)
  Return:   Server replies to 198.51.100.2
            → Arrives on ISP2 interface
            → Firewall drops (stateful: no session on ISP2!) ✗

Solution:
  - NAT phải match PBR path
  - Source NAT trên OUTGOING interface (masquerade)
  - Policy NAT (match same ACL as PBR)
```

```
! Cisco: Policy NAT matching PBR
ip access-list extended NAT-ISP1
 permit ip 10.0.1.0 0.0.0.255 any    ! Same as PBR match

ip access-list extended NAT-ISP2
 permit ip 10.0.2.0 0.0.0.255 any

! NAT per ISP interface
ip nat inside source list NAT-ISP1 interface GigabitEthernet0/1 overload
ip nat inside source list NAT-ISP2 interface GigabitEthernet0/2 overload
```

```bash
# Linux: Source NAT per table (iptables)
# Traffic qua ISP1 → NAT with ISP1 IP
iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
# Traffic qua ISP2 → NAT with ISP2 IP
iptables -t nat -A POSTROUTING -o eth2 -j MASQUERADE
# MASQUERADE tự dùng IP của outgoing interface
```

---

## 10. Troubleshooting PBR và Best Practices

### Troubleshooting Checklist

```
PBR không hoạt động:

□ 1. Route-map applied trên đúng interface?
     show ip policy
     ! Phải show route-map name trên interface

□ 2. Route-map permit (không phải deny)?
     route-map X permit 10  ← permit = apply set actions
     route-map X deny 10    ← deny = route normally (skip PBR)

□ 3. ACL match đúng traffic?
     show access-list 101
     ! Check counters — matches increasing?

□ 4. Next-hop reachable?
     ping [next-hop IP]
     show ip arp | include [next-hop IP]
     ! Next-hop phải có ARP entry (directly connected hoặc có route)

□ 5. Track object UP (nếu verify-availability)?
     show track
     ! State phải = Up

□ 6. Traffic đi đúng direction?
     PBR = INBOUND only!
     Apply trên interface nơi traffic ĐI VÀO router
     
□ 7. CEF enabled? (Cisco)
     ip cef (phải enabled cho PBR hoạt động)
```

### Debug Commands — Cisco

```bash
# Verify PBR policy applied
show ip policy

# Check route-map details
show route-map PBR-DUAL-ISP

# Debug PBR (CAUTION: high CPU in production)
debug ip policy

# Output example:
# IP: s=10.0.1.100 (GigabitEthernet0/0), d=8.8.8.8, len 84,
#   FIB policy match
#   FIB policy set next-hop 203.0.113.1
#   FIB policy routed

# Packet trace (IOS-XE)
debug platform condition ipv4 10.0.1.100/32 both
debug platform packet-trace packet 100
show platform packet-trace summary
```

### Debug — Linux

```bash
# Check which table a specific packet matches
ip route get 8.8.8.8 from 10.0.1.100 iif eth0
# Output: 8.8.8.8 from 10.0.1.100 via 203.0.113.1 dev eth1 table isp1

# Monitor rule hits
watch -n1 "ip -s rule show"

# Check if mark is being set
iptables -t mangle -L PREROUTING -v -n
# Watch counters increase

# tcpdump on specific interface to verify path
tcpdump -i eth1 host 8.8.8.8  # Should see traffic on ISP1 interface

# Trace routing decision
ip route get 8.8.8.8 from 10.0.2.100 mark 2
```

### Best Practices

| Practice | Mô tả |
|---|---|
| Match specific, act broad | ACLs nên match chính xác traffic cần PBR |
| Always have fallback | set ip next-hop verify-availability + backup hop |
| Monitor health | IP SLA / track objects cho mọi next-hop |
| Document policies | Comment mỗi route-map sequence rõ ràng |
| Avoid PBR where possible | Prefer routing protocol (BGP communities, local-pref) |
| Test asymmetric | Verify return path phải đúng interface (NAT!) |
| Use PBR for exceptions | 95% traffic dùng normal routing, PBR cho special cases |
| Consider scalability | Nhiều ACLs = TCAM pressure trên hardware platforms |
| Implement logging | ACL logging (rate-limited) cho audit |
| Backup config | PBR misconfiguration = instant outage |

### Tổng kết

```
Policy-Based Routing Summary:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  PBR = Forward packets based on POLICY, not just dest  │
│                                                         │
│  Components (Cisco):                                    │
│  1. ACL (match criteria) — who/what traffic            │
│  2. Route-map (policy) — match + set                   │
│  3. Interface (apply) — ip policy route-map X          │
│  4. Track (health) — verify next-hop alive             │
│                                                         │
│  Components (Linux):                                    │
│  1. ip rule (match criteria + table selection)          │
│  2. ip route table X (custom routing tables)           │
│  3. iptables -t mangle (mark packets for complex match)│
│  4. Health check script (failover)                     │
│                                                         │
│  Key Use Cases:                                         │
│  • Multi-ISP load split (VoIP→ISP1, Web→ISP2)        │
│  • Source-based routing (dept A→path1, dept B→path2)  │
│  • Security steering (force traffic through FW/IDS)   │
│  • Cloud ingress routing (AWS IGW route tables)       │
│                                                         │
│  Remember:                                              │
│  • PBR = inbound only (on ingress interface)           │
│  • NAT must match PBR path (avoid asymmetric)          │
│  • Always configure failover (track + verify)          │
│  • Use PBR sparingly — routing protocols preferred     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

*Tài liệu tham khảo:*
- Cisco Policy-Based Routing Configuration Guide — IOS/IOS-XE
- Linux ip-rule(8) man page
- Linux ip-route(8) man page
- RFC 1102 — Policy Routing in Internet Protocols
- AWS VPC Route Tables Documentation
- GCP Policy-Based Routes Documentation
- Azure User Defined Routes Documentation
- Cisco IP SLA Configuration Guide
