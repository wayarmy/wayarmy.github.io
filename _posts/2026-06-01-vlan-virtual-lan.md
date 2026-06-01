---
layout: post
title: "VLAN — Virtual LAN — Phân chia mạng logic trên cùng Switch vật lý"
subtitle: "Hiểu sâu về 802.1Q Trunking, Access Port, Native VLAN, và Inter-VLAN Routing"
tags: [networking, vlan, switching, layer2, 802.1q, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| IEEE 802.1Q-2022 | Bridges and Bridged Networks — VLAN Tagging |
| Cisco — VLAN Configuration Guide | Catalyst Switch VLAN Implementation |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 4: VLANs |
| Cisco — Inter-VLAN Routing | Router-on-a-Stick and L3 Switch |
| AWS VPC Documentation | Subnets and Security Groups |

---

## 1. Giới thiệu — Tại sao cần biết VLAN?

### Ví dụ đời thường: Căn hộ chung cư vs nhà riêng

Hãy tưởng tượng một **tòa chung cư 20 tầng** với 1 hành lang duy nhất. Nếu không có tường ngăn phòng, **mọi tiếng ồn** lan khắp tòa nhà — anh A hắt hơi tầng 1, chị B tầng 20 nghe thấy. Mọi người **không có privacy**.

**VLAN giống như xây tường ngăn phòng** trong cùng 1 tòa nhà:
- Tầng 1-5: Phòng cho công ty Sales (VLAN 10)
- Tầng 6-10: Phòng cho công ty IT (VLAN 20)
- Tầng 11-15: Phòng cho công ty Finance (VLAN 30)

Dù cùng tòa nhà (cùng switch vật lý), mỗi "khu" **không nghe thấy** tiếng ồn của khu khác. Muốn liên lạc giữa các khu phải đi qua **lễ tân** (router).

### Concrete scenario: "Hãy tưởng tượng bạn đang..."

Hãy tưởng tượng bạn là IT admin của công ty có 3 phòng ban: **Sales** (20 người), **Engineering** (30 người), **Finance** (10 người). Tất cả ngồi cùng tầng, cắm vào cùng 1 switch 48 ports.

**Không có VLAN:**
- Virus từ máy Sales → broadcast storm → lan sang Engineering, Finance
- Nhân viên Sales có thể sniff traffic Finance (salary info!)
- DHCP server phòng IT → conflict với DHCP router của Sales
- ARP broadcast từ 60 máy → tất cả 60 máy xử lý → chậm

**Có VLAN:**
- Sales (VLAN 10): Ports 1-20, subnet 10.10.10.0/24
- Engineering (VLAN 20): Ports 21-44, subnet 10.10.20.0/24
- Finance (VLAN 30): Ports 45-48 + switch khác, subnet 10.10.30.0/24
- Broadcast chỉ trong VLAN → hiệu suất tốt
- Traffic isolation → bảo mật
- Muốn liên lạc giữa phòng ban → qua router/firewall (có thể apply ACL)

### Vấn đề VLAN giải quyết

| Vấn đề | Không có VLAN | Có VLAN |
|---------|--------------|---------|
| Security | Ai cũng sniff được traffic người khác | Traffic isolated giữa VLANs |
| Broadcast storms | Lan toàn mạng (60 hosts) | Chỉ trong VLAN (20 hosts max) |
| Flexibility | Phải kéo cáp lại nếu đổi phòng ban | Chỉ đổi VLAN assignment trên port |
| Management | 1 broadcast domain khổng lồ | Nhiều broadcast domain nhỏ, dễ quản lý |
| Performance | ARP/DHCP broadcasts ảnh hưởng tất cả | Broadcasts giới hạn trong VLAN |

---

## 2. VLAN là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**VLAN (Virtual Local Area Network)** là cách chia **1 switch vật lý** thành **nhiều mạng riêng biệt** (broadcast domains) bằng software. Thiết bị ở VLAN khác nhau **KHÔNG thể liên lạc trực tiếp** — dù cắm cùng switch.

### Analogy: Radio channels trong cùng 1 vùng

```
┌─────────────────────────────────────────────────────────────┐
│             VLAN = KÊNH RADIO (FREQUENCY)                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Cùng 1 khu vực (= cùng 1 switch), nhưng:                 │
│                                                              │
│  Channel 10 (VLAN 10 - Sales):                              │
│  📻 PC1 ─── 📻 PC2 ─── 📻 PC3    ← Chỉ nghe nhau!        │
│                                                              │
│  Channel 20 (VLAN 20 - Engineering):                        │
│  📻 PC4 ─── 📻 PC5 ─── 📻 PC6    ← Chỉ nghe nhau!        │
│                                                              │
│  Channel 30 (VLAN 30 - Finance):                            │
│  📻 PC7 ─── 📻 PC8               ← Chỉ nghe nhau!         │
│                                                              │
│  PC1 muốn nói với PC4? KHÔNG ĐƯỢC (khác kênh!)             │
│  → Phải qua "phiên dịch" (Router/L3 Switch)                │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### ASCII Diagram: VLAN trên Switch

```
┌────────────────────── Switch 48 ports ──────────────────────┐
│                                                              │
│  VLAN 10 (Sales)          VLAN 20 (Eng)     VLAN 30 (Fin)  │
│  ┌────────────────┐      ┌────────────┐    ┌─────────┐     │
│  │ Port 1  │ PC-1 │      │Port 21│PC-4│    │Port45│PC7│    │
│  │ Port 2  │ PC-2 │      │Port 22│PC-5│    │Port46│PC8│    │
│  │ Port 3  │ PC-3 │      │Port 23│PC-6│    └─────────┘     │
│  │ ...     │ ...  │      │...    │... │                     │
│  │ Port 20 │ PC-20│      │Port 44│PC-X│                     │
│  └────────────────┘      └────────────┘                     │
│                                                              │
│  Broadcast trong VLAN 10 → CHỈ ra ports 1-20               │
│  Broadcast trong VLAN 20 → CHỈ ra ports 21-44              │
│  Broadcast trong VLAN 30 → CHỈ ra ports 45-46              │
│                                                              │
│  ┌──── Port 48: TRUNK (802.1Q) ────────────────────────┐   │
│  │ Carries ALL VLANs (10, 20, 30) to another switch    │   │
│  │ Mỗi frame được TAG VLAN ID vào header               │   │
│  └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. 802.1Q VLAN Tagging — "Dán nhãn" frame thuộc VLAN nào

### Ví dụ đời thường: Dán sticker màu lên phong bì

Khi bưu tá (switch) cần gửi thư qua **1 đường ống chung** (trunk link) mà trong đó có thư của nhiều khu phố (VLANs), bưu tá **dán sticker màu**:
- Sticker đỏ (tag VLAN 10) → thư của Sales
- Sticker xanh (tag VLAN 20) → thư của Engineering
- Sticker vàng (tag VLAN 30) → thư của Finance

Bưu tá bên kia nhận thư, nhìn sticker, biết gửi cho khu phố nào!

### 3.1 802.1Q Tag Format

```
Original Ethernet Frame:
┌────────┬────────┬───────────┬─────────┬─────┐
│Dst MAC │Src MAC │ EtherType │ Payload │ FCS │
│6 bytes │6 bytes │ 2 bytes   │         │ 4B  │
└────────┴────────┴───────────┴─────────┴─────┘

802.1Q Tagged Frame (thêm 4 bytes):
┌────────┬────────┬────────────────────────┬───────────┬─────────┬─────┐
│Dst MAC │Src MAC │      802.1Q Tag        │ EtherType │ Payload │ FCS │
│6 bytes │6 bytes │      4 bytes           │ 2 bytes   │         │ 4B  │
└────────┴────────┴────────────────────────┴───────────┴─────────┴─────┘
                   │                        │
                   ▼                        │
         ┌──────────────────────────┐      │
         │ TPID    │ TCI            │      │
         │ 0x8100  │ PCP│DEI│VID    │      │
         │ 2 bytes │ 3b │1b │12 bits│      │
         └──────────────────────────┘
```

**Chi tiết các trường:**

| Field | Bits | Giải thích |
|-------|------|-----------|
| TPID | 16 | Tag Protocol Identifier = `0x8100` (luôn cố định) |
| PCP | 3 | Priority Code Point: 0-7 (QoS priority) |
| DEI | 1 | Drop Eligible Indicator (có thể drop khi congestion) |
| VID | 12 | VLAN Identifier: 0-4095 |

**VLAN ID range:**
- VID 0: Priority tagging only (no VLAN)
- VID 1: Default VLAN (thường là native VLAN)
- VID 2-1001: Normal range (Cisco)
- VID 1002-1005: Reserved (FDDI, Token Ring)
- VID 1006-4094: Extended range
- VID 4095: Reserved (implementation-specific)

### 3.2 Access Port vs Trunk Port

```
┌─────────────────────────────────────────────────────────────┐
│  ACCESS PORT                    │  TRUNK PORT                │
├─────────────────────────────────┼────────────────────────────┤
│                                 │                            │
│  • Thuộc 1 VLAN duy nhất      │  • Carry nhiều VLANs       │
│  • Kết nối end device          │  • Kết nối switch↔switch   │
│  • Frame KHÔNG có 802.1Q tag   │  • Frame CÓ 802.1Q tag    │
│  • Device không biết VLAN      │  • Cả 2 bên phải hiểu tag │
│                                 │                            │
│  PC ──[untagged]──→ Switch     │  Switch ──[tagged]──→ Switch│
│  "Tôi không biết VLAN!"        │  "Frame này thuộc VLAN 20" │
│                                 │                            │
│  Switch tự gán VLAN cho port:  │  Switch đọc VLAN từ tag:   │
│  "Port 5 = VLAN 10"           │  "Tag = 20 → forward theo  │
│  → Frame vào port 5 thuộc     │    VLAN 20"                 │
│    VLAN 10                     │                            │
│                                 │                            │
└─────────────────────────────────┴────────────────────────────┘
```

### 3.3 Native VLAN — VLAN "mặc định" trên trunk

**Native VLAN** = VLAN mà frames KHÔNG ĐƯỢC TAG khi đi qua trunk port.

```
Trunk port có native VLAN = 1 (default):

Frame từ VLAN 10: [Dst MAC][Src MAC][0x8100][VID=10][Data][FCS] ← TAGGED
Frame từ VLAN 20: [Dst MAC][Src MAC][0x8100][VID=20][Data][FCS] ← TAGGED
Frame từ VLAN 1:  [Dst MAC][Src MAC][Data][FCS]                 ← UNTAGGED!
                                     ↑ Không có 802.1Q tag
```

**Tại sao có Native VLAN?**
- Backward compatibility: Thiết bị cũ không hiểu 802.1Q tag → vẫn nhận được frame
- Management traffic: Một số protocol (CDP, VTP, DTP) gửi untagged

**⚠️ SECURITY RISK — VLAN Hopping Attack:**
Attacker trên native VLAN có thể "nhảy" sang VLAN khác bằng double-tagging:

```
Attacker frame: [Dst][Src][Tag: Native VLAN][Tag: Target VLAN 20][Data]
                              ↑ Switch 1 strip tag này (native)
                                            ↑ Switch 2 forward theo tag này!
→ Frame ends up in VLAN 20!
```

**Best practice:** Đổi native VLAN khỏi VLAN 1, hoặc tag native VLAN:
```
Switch(config-if)# switchport trunk native vlan 999
Switch(config-if)# vlan dot1q tag native   ! Tag cả native VLAN
```

---

## 4. VLAN Configuration — Cấu hình chi tiết

### 4.1 Tạo VLAN

```bash
# Cisco IOS:
Switch(config)# vlan 10
Switch(config-vlan)# name Sales
Switch(config-vlan)# exit

Switch(config)# vlan 20
Switch(config-vlan)# name Engineering
Switch(config-vlan)# exit

Switch(config)# vlan 30
Switch(config-vlan)# name Finance
Switch(config-vlan)# exit

# Verify:
Switch# show vlan brief
# VLAN Name                Status    Ports
# ---- ------------------- --------- ------
# 1    default             active    Fa0/1-48, Gi0/1-2
# 10   Sales               active    
# 20   Engineering         active    
# 30   Finance             active    
```

### 4.2 Assign Port to VLAN (Access Mode)

```bash
# Assign single port:
Switch(config)# interface FastEthernet0/1
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10

# Assign range of ports:
Switch(config)# interface range FastEthernet0/1-20
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 10

Switch(config)# interface range FastEthernet0/21-44
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 20
```

### 4.3 Configure Trunk Port

```bash
# Configure trunk:
Switch(config)# interface GigabitEthernet0/1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk encapsulation dot1q   ! nếu switch hỗ trợ ISL
Switch(config-if)# switchport trunk allowed vlan 10,20,30
Switch(config-if)# switchport trunk native vlan 999

# Verify trunk:
Switch# show interfaces trunk
# Port      Mode     Encapsulation  Status      Native vlan
# Gi0/1     on       802.1q         trunking    999
#
# Port      Vlans allowed on trunk
# Gi0/1     10,20,30
```

### 4.4 DTP (Dynamic Trunking Protocol) — Auto negotiate trunk

```
┌─────────────────────────────────────────────────────────────┐
│  DTP Modes và kết quả khi 2 port kết nối:                  │
├──────────────┬──────────┬─────────┬───────────┬────────────┤
│  SW1 \ SW2   │ Dynamic  │ Dynamic │  Trunk    │  Access    │
│              │ Auto     │ Desirable│           │            │
├──────────────┼──────────┼─────────┼───────────┼────────────┤
│ Dynamic Auto │ Access   │ Trunk   │  Trunk    │  Access    │
│ Dyn Desirable│ Trunk    │ Trunk   │  Trunk    │  Access    │
│ Trunk        │ Trunk    │ Trunk   │  Trunk    │ Limited*   │
│ Access       │ Access   │ Access  │ Limited*  │  Access    │
└──────────────┴──────────┴─────────┴───────────┴────────────┘

⚠️ Best Practice: KHÔNG dùng DTP! Set cứng trunk hoặc access:
Switch(config-if)# switchport nonegotiate   ! Disable DTP
```

---

## 5. Inter-VLAN Routing — Liên lạc giữa các VLANs

### Ví dụ đời thường

VLANs giống các phòng kín trong tòa nhà. Muốn đi từ phòng Sales sang phòng Engineering, bạn phải đi ra hành lang, qua cửa kiểm soát (router), rồi mới vào phòng Engineering.

### 5.1 Method 1: Router-on-a-Stick (Subinterfaces)

```
                    ┌─────────────────────────────────┐
                    │         ROUTER                   │
                    │  Gi0/0.10: 10.10.10.1/24        │
                    │  Gi0/0.20: 10.10.20.1/24        │
                    │  Gi0/0.30: 10.10.30.1/24        │
                    └──────────────┬──────────────────┘
                                   │ Trunk (carries VLAN 10,20,30)
                                   │
                    ┌──────────────┴──────────────────┐
                    │          SWITCH                   │
                    │  VLAN 10: Ports 1-20 (Sales)     │
                    │  VLAN 20: Ports 21-44 (Eng)      │
                    │  VLAN 30: Ports 45-48 (Fin)      │
                    └──────────────────────────────────┘
```

**Router config:**
```bash
Router(config)# interface GigabitEthernet0/0
Router(config-if)# no shutdown

Router(config)# interface GigabitEthernet0/0.10
Router(config-subif)# encapsulation dot1q 10
Router(config-subif)# ip address 10.10.10.1 255.255.255.0

Router(config)# interface GigabitEthernet0/0.20
Router(config-subif)# encapsulation dot1q 20
Router(config-subif)# ip address 10.10.20.1 255.255.255.0

Router(config)# interface GigabitEthernet0/0.30
Router(config-subif)# encapsulation dot1q 30
Router(config-subif)# ip address 10.10.30.1 255.255.255.0
```

**Hạn chế:** Trunk link = bottleneck. Nếu 1000 Mbps trunk thì inter-VLAN traffic bị giới hạn 1 Gbps total.

### 5.2 Method 2: Layer 3 Switch (SVI — Switch Virtual Interface)

```bash
# L3 Switch config — NHANH HƠN router-on-a-stick nhiều lần!
Switch(config)# ip routing

Switch(config)# interface vlan 10
Switch(config-if)# ip address 10.10.10.1 255.255.255.0
Switch(config-if)# no shutdown

Switch(config)# interface vlan 20
Switch(config-if)# ip address 10.10.20.1 255.255.255.0
Switch(config-if)# no shutdown

Switch(config)# interface vlan 30
Switch(config-if)# ip address 10.10.30.1 255.255.255.0
Switch(config-if)# no shutdown
```

**Ưu điểm:** Wire-speed routing (ASIC-based) — không bottleneck!

### 5.3 So sánh

| Feature | Router-on-a-Stick | L3 Switch (SVI) |
|---------|-------------------|-----------------|
| Speed | Limited by trunk bandwidth | Wire-speed (hardware) |
| Scalability | Limited | High |
| Cost | Cần router riêng | L3 switch đắt hơn L2 |
| ACL/Firewall | Có | Có |
| Complexity | Simple (cho mạng nhỏ) | Medium |
| Best for | < 100 users | 100+ users |

---

## 6. VLAN Best Practices và Security

### 6.1 VLAN Best Practices

```
1. KHÔNG dùng VLAN 1 cho production traffic
   → VLAN 1 là management default, dễ bị attack

2. Đổi Native VLAN khỏi VLAN 1
   → switchport trunk native vlan 999

3. Disable DTP trên tất cả ports
   → switchport nonegotiate

4. Shutdown unused ports và assign vào "parking lot" VLAN
   → switchport access vlan 999
   → shutdown

5. Giới hạn allowed VLANs trên trunk
   → switchport trunk allowed vlan 10,20,30
   (không để default = ALL vlans)

6. Dùng Private VLANs cho DMZ/server segments
   → Isolated ports không nói chuyện được với nhau
```

### 6.2 VLAN Security Threats

| Attack | Mô tả | Phòng chống |
|--------|--------|-------------|
| VLAN Hopping (Switch Spoofing) | Attacker giả làm trunk port | Disable DTP, set access mode |
| VLAN Hopping (Double Tagging) | Đóng 2 lớp tag để nhảy VLAN | Native VLAN ≠ user VLAN |
| DHCP Starvation | Flood DHCP requests → exhaust pool | DHCP Snooping |
| ARP Spoofing | Fake ARP replies → MITM | Dynamic ARP Inspection (DAI) |
| MAC Flooding | Fill CAM table → switch becomes hub | Port Security |

### 6.3 Private VLANs (PVLAN)

```
┌──── PVLAN Structure ────────────────────────────────────────┐
│                                                              │
│  Primary VLAN 100:                                          │
│  ├── Community VLAN 101: Ports 1-5 (Web Servers)           │
│  │   → Can talk to each other + Promiscuous                │
│  ├── Community VLAN 102: Ports 6-10 (App Servers)          │
│  │   → Can talk to each other + Promiscuous                │
│  └── Isolated VLAN 199: Ports 11-20 (Guest VMs)           │
│      → Can ONLY talk to Promiscuous port                   │
│      → CANNOT talk to each other!                          │
│                                                              │
│  Promiscuous Port: Port 48 (Gateway/Router)                 │
│  → Can talk to ALL secondary VLANs                         │
│                                                              │
└──────────────────────────────────────────────────────────────┘

Use case: Hosting provider — mỗi khách hàng 1 isolated port,
chỉ access Internet qua gateway, KHÔNG thấy server khách khác!
```

---

## 7. VLAN trong AWS — Mapping sang Cloud

### 7.1 VPC Subnet ≈ VLAN

| VLAN Concept | AWS Equivalent |
|-------------|---------------|
| VLAN | VPC Subnet |
| VLAN ID | Subnet CIDR |
| Access Port | ENI in specific subnet |
| Trunk Port | Transit Gateway attachment |
| Inter-VLAN routing | VPC Route Table + implicit router |
| PVLAN (isolated) | Security Group (deny all inter-SG) |
| Native VLAN | Default route in subnet |
| VLAN ACL | Network ACL (NACL) |
| Port Security | Security Group rules |

### 7.2 VPC Design giống VLAN Design

```
┌───────────── VPC: 10.0.0.0/16 ─────────────────────────────┐
│                                                              │
│  Subnet A (Public) - "VLAN 10"                              │
│  10.0.1.0/24                                                │
│  ┌────────┐  ┌────────┐  ┌────────┐                       │
│  │ Web-1  │  │ Web-2  │  │ NAT GW │                       │
│  └────────┘  └────────┘  └────────┘                       │
│                                                              │
│  Subnet B (Private) - "VLAN 20"                             │
│  10.0.2.0/24                                                │
│  ┌────────┐  ┌────────┐  ┌────────┐                       │
│  │ App-1  │  │ App-2  │  │ App-3  │                       │
│  └────────┘  └────────┘  └────────┘                       │
│                                                              │
│  Subnet C (Database) - "VLAN 30"                            │
│  10.0.3.0/24                                                │
│  ┌────────┐  ┌────────┐                                    │
│  │ DB-1   │  │ DB-2   │  ← Security Group: chỉ allow      │
│  └────────┘  └────────┘    port 3306 từ Subnet B           │
│                                                              │
│  [Implicit Router] ← Routes giữa subnets (= inter-VLAN)   │
│  Route Table: local → 10.0.0.0/16 (tất cả subnets)        │
│              0.0.0.0/0 → IGW (Internet Gateway)            │
└──────────────────────────────────────────────────────────────┘
```

### 7.3 AWS Transit Gateway — "Trunk" giữa VPCs

```
┌─── VPC-1 (Production) ───┐    ┌─── VPC-2 (Staging) ───┐
│ 10.1.0.0/16              │    │ 10.2.0.0/16            │
└───────────┬──────────────┘    └────────────┬───────────┘
            │                                 │
            └──────────┬──────────────────────┘
                       │
              ┌────────┴────────┐
              │ Transit Gateway │  ← Giống "core switch" với trunk links
              │ (routes between │     đến mỗi VPC
              │  all VPCs)      │
              └────────┬────────┘
                       │
            ┌──────────┴──────────┐
            │                      │
┌───────────┴──────────┐  ┌───────┴───────────────┐
│ VPC-3 (Shared Svc)   │  │ On-premises (VPN/DX)  │
│ 10.3.0.0/16          │  │ 192.168.0.0/16        │
└──────────────────────┘  └───────────────────────┘
```

---

## 8. Tình huống thực tế — VLAN được sử dụng như thế nào?

### Scenario 1: Quán cafe / Khách sạn — Guest WiFi isolation

```
┌───── 1 Switch ─────────────────────────────────┐
│                                                 │
│  VLAN 1 (Management): Switch, AP management    │
│  VLAN 10 (Staff): POS, máy in, NAS            │
│  VLAN 99 (Guest WiFi): Khách hàng             │
│                                                 │
│  Guests (VLAN 99):                             │
│  - Chỉ access Internet (ACL block LAN)        │
│  - Bandwidth limit: 10 Mbps/device            │
│  - Isolated from staff network                 │
│  - DHCP: 172.16.99.0/24                       │
│                                                 │
│  Staff (VLAN 10):                              │
│  - Full LAN access                            │
│  - Access NAS, printer, POS                   │
│  - DHCP: 192.168.10.0/24                      │
└─────────────────────────────────────────────────┘
```

### Scenario 2: Công ty vừa — Multi-floor VLAN

```
Tầng 1 (Switch A):          Tầng 2 (Switch B):
Port 1-12: VLAN 10 (Sales)  Port 1-12: VLAN 10 (Sales)
Port 13-24: VLAN 20 (Eng)   Port 13-24: VLAN 20 (Eng)
Port 25-48: VLAN 30 (Fin)   Port 25-48: VLAN 30 (Fin)
Port 49-50: Trunk ──────────→ Port 49-50: Trunk

Khi Sales tầng 1 gửi cho Sales tầng 2:
Frame đi: Port 5 (VLAN 10) → Trunk (tagged VID=10) → Switch B
          → Switch B forward ra ports VLAN 10 (1-12)

→ Sales 2 tầng nhìn nhau như cùng LAN! (same broadcast domain)
```

### Scenario 3: ISP / Data Center — VLAN per customer

```
ISP Core Switch:
- VLAN 100: Customer A (company A's traffic)
- VLAN 101: Customer B
- VLAN 102: Customer C
- ...
- VLAN 4000+: Dùng Q-in-Q (802.1ad) cho thêm capacity

Q-in-Q (Provider Bridging):
┌────────┬────────┬────────────┬───────────────┬─────────┬─────┐
│Dst MAC │Src MAC │ S-TAG      │ C-TAG         │Payload  │ FCS │
│        │        │ (Provider) │ (Customer)    │         │     │
│        │        │ VID=100    │ VID=10 (Sales)│         │     │
└────────┴────────┴────────────┴───────────────┴─────────┴─────┘
                   ↑ ISP dùng S-TAG để phân biệt customer
                                ↑ Customer's internal VLAN
```

---

## 9. Bài tập thực hành

### Exercise 1: VLAN cơ bản

```bash
# Cisco IOS (hoặc GNS3/Packet Tracer):

# 1. Tạo VLANs
Switch(config)# vlan 10
Switch(config-vlan)# name Sales
Switch(config)# vlan 20  
Switch(config-vlan)# name Engineering

# 2. Assign ports
Switch(config)# interface range fa0/1-10
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 10

Switch(config)# interface range fa0/11-20
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 20

# 3. Verify: PC trên VLAN 10 KHÔNG ping được PC trên VLAN 20
# (vì không có routing giữa VLANs)

# 4. Verify: PC trên VLAN 10 CÓ ping được PC khác trên VLAN 10
```

### Exercise 2: Trunk Configuration

```bash
# Switch A → Switch B trunk
# Switch A:
Switch-A(config)# interface gi0/1
Switch-A(config-if)# switchport mode trunk
Switch-A(config-if)# switchport trunk allowed vlan 10,20
Switch-A(config-if)# switchport trunk native vlan 999
Switch-A(config-if)# switchport nonegotiate

# Switch B: (same config)
Switch-B(config)# interface gi0/1
Switch-B(config-if)# switchport mode trunk
Switch-B(config-if)# switchport trunk allowed vlan 10,20
Switch-B(config-if)# switchport trunk native vlan 999

# Verify:
Switch# show interfaces trunk
Switch# show interfaces gi0/1 switchport
```

### Exercise 3: Inter-VLAN Routing (Router-on-a-Stick)

```bash
# Router config:
Router(config)# interface gi0/0.10
Router(config-subif)# encapsulation dot1q 10
Router(config-subif)# ip address 10.10.10.1 255.255.255.0

Router(config)# interface gi0/0.20
Router(config-subif)# encapsulation dot1q 20
Router(config-subif)# ip address 10.10.20.1 255.255.255.0

# PC config:
# PC-A (VLAN 10): IP 10.10.10.10, Gateway 10.10.10.1
# PC-B (VLAN 20): IP 10.10.20.10, Gateway 10.10.20.1

# Test: PC-A ping PC-B → SUCCESS (routed through router)
```

### Exercise 4: Linux VLAN Configuration

```bash
# Linux — tạo VLAN interface
sudo ip link add link eth0 name eth0.10 type vlan id 10
sudo ip link set eth0.10 up
sudo ip addr add 10.10.10.100/24 dev eth0.10

sudo ip link add link eth0 name eth0.20 type vlan id 20
sudo ip link set eth0.20 up
sudo ip addr add 10.10.20.100/24 dev eth0.20

# Verify
ip -d link show eth0.10
cat /proc/net/vlan/eth0.10

# Hoặc dùng NetworkManager:
nmcli connection add type vlan con-name vlan10 dev eth0 id 10
nmcli connection modify vlan10 ipv4.addresses 10.10.10.100/24
nmcli connection modify vlan10 ipv4.method manual
nmcli connection up vlan10
```

### Exercise 5: Wireshark — Capture 802.1Q tags

```bash
# Capture trên trunk port (mirror/span port):
sudo tcpdump -i eth0 -e -nn -c 20 vlan

# Wireshark filter:
# vlan.id == 10          → Chỉ VLAN 10 traffic
# vlan                    → Tất cả tagged frames
# frame.len > 1518       → Tagged frames (1522 bytes max)

# Bài tập: Capture và identify:
# 1. Frame nào tagged vs untagged?
# 2. VLAN ID trong mỗi tagged frame?
# 3. PCP priority value?
```

---

## 10. Tóm tắt & Tài liệu đọc thêm

### Key Points — Ghi nhớ

| # | Concept | Điểm quan trọng |
|---|---------|-----------------|
| 1 | VLAN | Chia 1 switch thành nhiều broadcast domains bằng software |
| 2 | 802.1Q Tag | 4 bytes thêm vào frame: TPID(0x8100) + PCP + DEI + VID(12bit) |
| 3 | Access Port | 1 VLAN, frames untagged, cho end devices |
| 4 | Trunk Port | Nhiều VLANs, frames tagged, cho switch-to-switch |
| 5 | Native VLAN | VLAN untagged trên trunk — đổi khỏi VLAN 1! |
| 6 | Inter-VLAN | Cần router hoặc L3 switch để route giữa VLANs |
| 7 | VLAN Hopping | Double-tagging attack — fix: native VLAN ≠ user VLAN |
| 8 | DTP | Tắt đi! Dùng switchport nonegotiate |
| 9 | PVLAN | Isolated/Community sub-VLANs cho hosting/DMZ |
| 10 | AWS | VPC Subnet ≈ VLAN, Security Group ≈ VLAN ACL |

### Tài liệu đọc thêm

| # | Tài liệu | Link/Reference |
|---|----------|---------------|
| 1 | IEEE 802.1Q-2022 | https://standards.ieee.org/ieee/802.1Q/10323/ |
| 2 | Cisco VLAN Configuration Guide | https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-1/configuration_guide/vlan/b-171-vlan-9300-cg.html |
| 3 | Cisco Inter-VLAN Routing | https://www.cisco.com/c/en/us/support/docs/lan-switching/inter-vlan-routing/41860-howto-L3-interVLAN.html |
| 4 | AWS VPC User Guide | https://docs.aws.amazon.com/vpc/latest/userguide/ |
| 5 | AWS Transit Gateway | https://docs.aws.amazon.com/vpc/latest/tgw/ |
| 6 | Tanenbaum — Computer Networks, 6th Ed. | Chapter 4: VLANs |

---

**Bài tiếp theo**: [STP — Spanning Tree Protocol — Ngăn chặn loop trong mạng Layer 2](/2026-06-01-stp-spanning-tree-protocol)
