---
layout: post
title: "MAC Address Deep Dive — OUI, EUI-48, và MAC Table Learning"
subtitle: "Hiểu sâu về địa chỉ vật lý Layer 2 — định danh duy nhất cho mọi thiết bị mạng"
tags: [networking, mac-address, layer2, switching, aws, learning-path, deep-dive]
categories: [networking]
date: 2026-06-01
gh-repo: wayarmy/wayarmy.github.io
---

## Source References

| Nguồn | Mô tả |
|--------|--------|
| RFC 7042 | IANA Considerations and IETF Protocol Usage for IEEE 802 Parameters |
| IEEE — EUI-48 Guidelines | Guidelines for Use of Extended Unique Identifier |
| IEEE 802-2014 | IEEE Standard for Local and Metropolitan Area Networks: Overview and Architecture |
| Tanenbaum, A.S. — Computer Networks, 6th Ed. | Chapter 4: MAC Sublayer |
| Cisco — Understanding MAC Address Tables | Catalyst Switch Configuration Guides |
| RFC 4291 | IP Version 6 Addressing Architecture (EUI-64 derivation) |

---

## 1. Giới thiệu — Tại sao cần biết MAC Address?

### Ví dụ đời thường: Số CMND/CCCD của thiết bị mạng

Mỗi người Việt Nam có một số **Căn Cước Công Dân (CCCD)** duy nhất — không ai trùng nhau, được cấp từ lúc sinh ra, in trên thẻ cứng. Tương tự, mỗi thiết bị mạng (card WiFi, port Ethernet) có một **MAC Address** duy nhất — được nhà sản xuất "đóng dấu" vào hardware từ nhà máy.

**Khác biệt với IP Address:**
- **MAC Address** = số CCCD (cố định, gắn với "con người"/phần cứng)
- **IP Address** = địa chỉ nhà (có thể thay đổi khi chuyển nhà/mạng)

### Concrete scenario: "Hãy tưởng tượng bạn đang..."

Hãy tưởng tượng bạn mang laptop từ nhà đến công ty. **MAC Address** của card WiFi laptop **KHÔNG đổi** — nó luôn là `A4:CF:12:xx:xx:xx` (ví dụ laptop Intel). Nhưng **IP Address** thay đổi:
- Ở nhà: `192.168.1.105` (từ router nhà)
- Ở công ty: `10.20.30.45` (từ DHCP server công ty)
- Ở quán cafe: `172.16.0.23` (từ router quán)

Switch dùng MAC Address để biết "laptop này đang cắm ở port nào" — giống bưu tá dùng số CCCD để xác nhận người nhận thư, không phụ thuộc vào địa chỉ nhà.

### Vấn đề MAC Address giải quyết

| Vấn đề | Giải pháp |
|---------|----------|
| Làm sao xác định duy nhất 1 thiết bị trên toàn thế giới? | 48-bit globally unique address |
| Làm sao switch biết gửi frame ra port nào? | MAC Address Table learning |
| Làm sao phân biệt unicast / multicast / broadcast? | Special bits trong MAC |
| Làm sao ánh xạ IP → MAC trong cùng LAN? | ARP protocol dùng MAC |
| Làm sao nhà sản xuất không trùng nhau? | OUI (Organizationally Unique Identifier) |

---

## 2. MAC Address là gì? — Giải thích cho người không biết IT

### Định nghĩa đơn giản

**MAC Address** (Media Access Control Address) là **"tên thật" của card mạng** — một dãy 12 chữ số hex (48 bits) được gắn cứng vào phần cứng. Ví dụ: `00:1A:2B:3C:4D:5E`

Nó giống như **biển số xe** — mỗi xe có 1 biển số riêng, do nhà nước cấp (= do IEEE phân bổ cho nhà sản xuất).

### Analogy: Hệ thống biển số xe

```
┌─────────────────────────────────────────────────────────────┐
│              MAC ADDRESS = BIỂN SỐ XE                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  MAC:   00:1A:2B : 3C:4D:5E                                │
│         ├──────┤   ├──────┤                                 │
│         │ OUI  │   │Device│                                 │
│         │      │   │ ID   │                                 │
│         └──────┘   └──────┘                                 │
│                                                              │
│  Biển số: 30A - 123.45                                      │
│           ├──┤   ├─────┤                                    │
│           │Tỉnh│  │ Số xe│                                  │
│           │    │  │riêng │                                  │
│           └────┘  └──────┘                                  │
│                                                              │
│  OUI = "Mã tỉnh" → Xác định nhà sản xuất                  │
│  Device ID = "Số xe" → Xác định thiết bị cụ thể            │
│                                                              │
│  Ví dụ OUI:                                                 │
│  00:1A:2B = Ayecom Technology (Trung Quốc)                  │
│  A4:CF:12 = Intel Corporate                                 │
│  DC:A6:32 = Raspberry Pi Trading                            │
│  00:50:56 = VMware (virtual NIC)                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### ASCII Diagram: MAC Address trong mô hình mạng

```
┌─────────────────────────────────────────────────────────┐
│ Application: HTTP GET google.com                         │
├─────────────────────────────────────────────────────────┤
│ Transport: TCP port 80                                   │
├─────────────────────────────────────────────────────────┤
│ Network: IP src=192.168.1.5, dst=142.250.80.46          │
├─────────────────────────────────────────────────────────┤
│ Data Link: MAC src=A4:CF:12:xx:xx:xx ← MÁY BẠN        │
│            MAC dst=00:1A:2B:xx:xx:xx ← ROUTER (GW)     │
│            ↑ MAC CHỈ DÙNG TRONG CÙNG LAN SEGMENT ↑     │
├─────────────────────────────────────────────────────────┤
│ Physical: Electrical signals trên cáp Cat 6              │
└─────────────────────────────────────────────────────────┘
```

**Điểm quan trọng:** MAC Address chỉ có ý nghĩa trong **cùng một LAN segment** (cùng broadcast domain). Khi packet đi qua router sang mạng khác, MAC Address được **thay đổi** (src/dst MAC mới), nhưng IP Address giữ nguyên.

---

## 3. Cấu trúc MAC Address — 48 bits chi tiết

### 3.1 Format hiển thị

Cùng một MAC Address có thể được viết theo nhiều cách:

| Format | Ví dụ | Thường dùng trong |
|--------|-------|-------------------|
| Colon-separated | `00:1A:2B:3C:4D:5E` | Linux, Wireshark |
| Hyphen-separated | `00-1A-2B-3C-4D-5E` | Windows |
| Dot-separated | `001A.2B3C.4D5E` | Cisco |
| No separator | `001A2B3C4D5E` | Programming |

### 3.2 Cấu trúc bit-level

```
Byte 1       Byte 2       Byte 3       Byte 4       Byte 5       Byte 6
┌──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ b7 ... b0│ b7 ... b0│ b7 ... b0│ b7 ... b0│ b7 ... b0│ b7 ... b0│
└──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
│←────────── OUI (24 bits) ──────────→│←─── Device ID (24 bits) ──→│
│                                      │                            │
│  Byte 1, Bit 0 (LSB):              │                            │
│  = 0 → Unicast                      │                            │
│  = 1 → Multicast                    │                            │
│                                      │                            │
│  Byte 1, Bit 1:                     │                            │
│  = 0 → Globally Unique (BIA)       │                            │
│  = 1 → Locally Administered        │                            │
└──────────────────────────────────────┴────────────────────────────┘
```

### 3.3 Hai bit đặc biệt — "DNA" của MAC Address

#### Bit 0 (I/G bit — Individual/Group)

| Bit 0 | Loại | Ý nghĩa | Ví dụ |
|-------|------|---------|-------|
| 0 | Unicast | Gửi cho 1 thiết bị | `00:1A:2B:3C:4D:5E` |
| 1 | Multicast/Broadcast | Gửi cho nhóm/tất cả | `01:00:5E:xx:xx:xx` |

**Cách nhận biết nhanh:** Nếu hex digit đầu tiên là số LẺ (1,3,5,7,9,B,D,F) → multicast. Số CHẴN (0,2,4,6,8,A,C,E) → unicast.

Ví dụ: `01:00:5E:...` → `0` trong hex = `0000` binary, `1` = `0001` → bit cuối = 1 → Multicast ✓

#### Bit 1 (U/L bit — Universal/Local)

| Bit 1 | Loại | Ý nghĩa |
|-------|------|---------|
| 0 | Universally Administered (UAA) | Do nhà sản xuất gán (BIA — Burned-In Address) |
| 1 | Locally Administered (LAA) | Do admin/software gán (override) |

**Ví dụ Locally Administered:**
- `x2:xx:xx:xx:xx:xx` (bit 1 = 1)
- `x6:xx:xx:xx:xx:xx`
- `xA:xx:xx:xx:xx:xx`
- `xE:xx:xx:xx:xx:xx`

Dùng trong: VMware, Docker, VPN, MAC randomization (privacy).

### 3.4 Các MAC Address đặc biệt

| MAC Address | Tên | Chức năng |
|------------|------|-----------|
| `FF:FF:FF:FF:FF:FF` | Broadcast | Gửi cho TẤT CẢ thiết bị trong LAN |
| `01:00:5E:00:00:00` - `01:00:5E:7F:FF:FF` | IPv4 Multicast | Map từ IP multicast 224.0.0.0/4 |
| `33:33:xx:xx:xx:xx` | IPv6 Multicast | Map từ IPv6 multicast |
| `01:80:C2:00:00:00` | STP (BPDU) | Spanning Tree Protocol |
| `01:80:C2:00:00:0E` | LLDP | Link Layer Discovery |
| `01:80:C2:00:00:02` | LACP | Link Aggregation |
| `00:00:00:00:00:00` | Null/Unspecified | Placeholder |
| `00:00:5E:00:01:xx` | VRRP | Virtual Router Redundancy |

### 3.5 OUI — Ai sản xuất thiết bị này?

**OUI (Organizationally Unique Identifier)** là 24 bits đầu (3 bytes) — IEEE bán cho nhà sản xuất:

| OUI | Nhà sản xuất | Ghi chú |
|-----|-------------|---------|
| `00:50:56` | VMware | Virtual NIC |
| `00:0C:29` | VMware | Older VMs |
| `08:00:27` | Oracle VirtualBox | Virtual NIC |
| `A4:CF:12` | Intel | Laptop WiFi/Ethernet |
| `DC:A6:32` | Raspberry Pi | Pi boards |
| `B8:27:EB` | Raspberry Pi | Older Pi |
| `00:1A:11` | Google | Android/Pixel |
| `F8:FF:C2` | Apple | iPhone/Mac |
| `3C:22:FB` | Apple | Newer devices |
| `00:15:5D` | Microsoft Hyper-V | Virtual NIC |

**Tra cứu OUI online:** https://standards-oui.ieee.org/oui/oui.txt

```bash
# Linux — tra OUI từ command line
grep -i "00:50:56" /usr/share/ieee-data/oui.txt
# Hoặc
apt install ieee-data && grep -i "005056" /usr/share/ieee-data/oui.txt
```

---

## 4. MAC Address Table — Switch học MAC như thế nào?

### Ví dụ đời thường: Bưu tá học đường đi

Tưởng tượng một **bưu tá mới** nhận việc tại bưu điện khu phố. Ngày đầu, bưu tá KHÔNG biết ai ở đâu. Cách bưu tá học:

1. **Khi nhận thư từ "Anh Hùng" gửi từ ngõ 5** → ghi nhớ: "Anh Hùng → Ngõ 5"
2. **Khi nhận thư từ "Chị Lan" gửi từ ngõ 8** → ghi nhớ: "Chị Lan → Ngõ 8"
3. **Khi cần gửi thư cho "Anh Hùng"** → biết ngay: đi ngõ 5!
4. **Khi cần gửi thư cho người chưa biết** → gửi cho TẤT CẢ ngõ (broadcast/flood)
5. **Nếu lâu không thấy ai** → xóa khỏi sổ (aging timer)

Switch hoạt động y hệt! Thay "ngõ" bằng "port", thay "tên người" bằng "MAC Address".

### 4.1 MAC Address Table (CAM Table) Learning Process

```
Switch có 4 ports: Fa0/1, Fa0/2, Fa0/3, Fa0/4

Bước 1: PC-A (MAC: AAAA) gửi frame ra port Fa0/1
         Switch: "Ồ, MAC AAAA ở port Fa0/1!" → Ghi vào bảng

┌──────────────────────────────┐
│   MAC Address Table          │
├──────────┬───────┬───────────┤
│ MAC Addr │ Port  │ Age Timer │
├──────────┼───────┼───────────┤
│ AAAA     │ Fa0/1 │ 0 sec     │
└──────────┴───────┴───────────┘

Bước 2: Frame có Destination = BBBB → KHÔNG có trong bảng!
         Switch: FLOOD ra tất cả ports TRỪU Fa0/1 (source port)
         → Gửi frame ra Fa0/2, Fa0/3, Fa0/4

Bước 3: PC-B (MAC: BBBB) ở port Fa0/3 reply
         Switch: "MAC BBBB ở port Fa0/3!" → Ghi vào bảng

┌──────────────────────────────┐
│   MAC Address Table          │
├──────────┬───────┬───────────┤
│ MAC Addr │ Port  │ Age Timer │
├──────────┼───────┼───────────┤
│ AAAA     │ Fa0/1 │ 45 sec    │
│ BBBB     │ Fa0/3 │ 0 sec     │
└──────────┴───────┴───────────┘

Bước 4: Lần sau AAAA gửi cho BBBB → Switch biết: ra Fa0/3!
         KHÔNG CẦN flood nữa → unicast forwarding
```

### 4.2 Aging Timer — "Quên" MAC cũ

- **Default aging time:** 300 giây (5 phút) trên hầu hết switch
- Nếu không thấy traffic từ MAC nào trong 300s → xóa khỏi bảng
- **Tại sao cần aging?** Vì thiết bị có thể rút cáp, chuyển port, hoặc tắt máy

```bash
# Cisco — xem MAC address table
show mac address-table
# VLAN  MAC Address      Type     Ports
# ----  -------- -----   -------  -----
# 1     0050.56c0.0001   DYNAMIC  Fa0/1
# 1     aabb.cc00.0200   DYNAMIC  Fa0/3

# Xem aging time
show mac address-table aging-time
# Global Aging Time: 300

# Thay đổi aging time
mac address-table aging-time 600

# Xóa toàn bộ dynamic entries
clear mac address-table dynamic
```

### 4.3 Static MAC — "Khóa cứng" địa chỉ

```bash
# Cisco — add static MAC entry
mac address-table static 0050.56c0.0001 vlan 1 interface Fa0/1

# Use cases:
# 1. Security: Chỉ cho phép thiết bị cụ thể ở port cụ thể
# 2. Server: Server quan trọng luôn ở port cố định
# 3. Port Security: Kết hợp với switchport port-security
```

### 4.4 MAC Address Table Size — Giới hạn và CAM Overflow Attack

| Switch | Typical CAM Table Size |
|--------|----------------------|
| Small office (unmanaged) | 1,024 - 4,096 entries |
| Enterprise access switch | 8,192 - 16,384 entries |
| Data center switch | 96,000 - 512,000+ entries |

**CAM Overflow Attack (MAC Flooding):**
Attacker gửi hàng nghìn frames với random source MAC → fill đầy CAM table → switch chuyển sang **fail-open mode** (flood tất cả frames như hub) → attacker sniff được mọi traffic!

**Phòng chống:** Port Security, 802.1X, DHCP Snooping, Dynamic ARP Inspection.

---

## 5. MAC Address Randomization — Privacy trong thời đại hiện đại

### Ví dụ đời thường: Đeo mặt nạ khi đi chợ

Nếu bạn đi cùng một chợ mỗi ngày với khuôn mặt thật (MAC cố định), camera an ninh sẽ track được lịch sử di chuyển của bạn. Nhưng nếu mỗi lần bạn đeo một **mặt nạ khác** (random MAC), không ai biết bạn là cùng 1 người!

### 5.1 Vấn đề Privacy

Trước 2014, smartphone WiFi liên tục phát **Probe Request** với MAC thật → các cửa hàng dùng WiFi tracking để:
- Đếm khách ra vào
- Track lộ trình trong mall
- Biết bạn quay lại bao nhiêu lần
- Gửi quảng cáo targeted

### 5.2 Giải pháp: MAC Randomization

| OS | Khi nào randomize | Format |
|----|-------------------|--------|
| iOS 14+ | Mỗi network khác nhau dùng private MAC khác nhau | `x2:xx:xx:xx:xx:xx` (Locally Administered) |
| Android 10+ | Per-network random MAC (persistent per SSID) | LAA format |
| Windows 10+ | Random MAC cho WiFi scanning | LAA format |
| macOS | Random cho probe requests | LAA format |

**Nhận biết MAC random:** Byte đầu tiên có bit 1 = 1 (Locally Administered). Ví dụ: `x2`, `x6`, `xA`, `xE`.

### 5.3 Tác động đến Network Management

```
VẤN ĐỀ CHO NETWORK ADMIN:
- DHCP reservation theo MAC → không hoạt động nếu MAC random
- MAC-based access control → client mỗi lần có MAC khác
- Troubleshooting → khó track device qua log

GIẢI PHÁP:
- Dùng 802.1X (xác thực bằng certificate, không dựa MAC)
- Dùng DHCP fingerprinting (OS fingerprint thay vì MAC)
- Accept randomization và adapt security model
```

---

## 6. EUI-64 — Mở rộng MAC 48-bit lên 64-bit cho IPv6

### 6.1 Tại sao cần EUI-64?

IPv6 Interface ID = 64 bits. Cách đơn giản nhất để tạo ID duy nhất: lấy MAC 48-bit và mở rộng lên 64-bit!

### 6.2 Quá trình chuyển đổi

```
MAC Address (EUI-48): 00:1A:2B:3C:4D:5E
                      ├──OUI──┤├─Device─┤

Bước 1: Chèn FF:FE vào giữa
00:1A:2B : FF:FE : 3C:4D:5E

Bước 2: Đảo bit U/L (bit 1 của byte đầu)
00 = 00000000 → đảo bit 1 → 00000010 = 02
         ↑                         ↑
       bit 1 = 0 (global)       bit 1 = 1 (vì trong IPv6, 1 = global unique)

Kết quả EUI-64: 02:1A:2B:FF:FE:3C:4D:5E

IPv6 Link-Local: fe80::021A:2BFF:FE3C:4D5E
```

**⚠️ Privacy concern:** EUI-64 tiết lộ MAC address thật! Vì vậy, RFC 4941 định nghĩa **Privacy Extensions** — dùng random Interface ID thay vì EUI-64.

---

## 7. MAC Address trong AWS — Mapping sang Cloud

### 7.1 ENI và MAC Address

Mỗi **Elastic Network Interface (ENI)** trong AWS được gán 1 MAC Address:

```bash
# Trên EC2 instance
ip link show eth0
# link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
#            ↑
#            02: = Locally Administered (AWS assigns, not burned-in)
```

**Đặc điểm MAC trong AWS:**
- AWS luôn dùng **Locally Administered** MAC (bit U/L = 1)
- MAC **gắn với ENI**, không phải instance → khi detach/attach ENI, MAC đi theo ENI
- Dùng OUI prefix: `02:xx:xx` hoặc vendor-specific

### 7.2 AWS Concepts liên quan đến MAC

| Khái niệm MAC | AWS Equivalent |
|---------------|---------------|
| MAC Address Learning | VPC networking (no flooding — AWS knows all) |
| MAC Table | VPC flow tables (software-defined) |
| Broadcast domain | Subnet (limited broadcast) |
| MAC Flooding attack | Không thể — AWS controls forwarding |
| Port Security | Security Groups + NACLs |
| Static MAC | ENI MAC persistence |

### 7.3 Practical — Xem MAC trên EC2

```bash
# Method 1: ip command
ip link show

# Method 2: Instance metadata
curl http://169.254.169.254/latest/meta-data/mac
# → 02:42:ac:11:00:02

# Method 3: All ENI MACs
curl http://169.254.169.254/latest/meta-data/network/interfaces/macs/
# → 02:42:ac:11:00:02/

# Method 4: AWS CLI
aws ec2 describe-network-interfaces \
  --network-interface-ids eni-xxxx \
  --query 'NetworkInterfaces[0].MacAddress'
```

---

## 8. Tình huống thực tế — MAC Address được sử dụng như thế nào?

### Scenario 1: Gia đình — MAC Filtering trên Router WiFi

**Bạn An** muốn chặn thiết bị lạ kết nối WiFi nhà mình:

```
Router WiFi Settings → MAC Filtering → Whitelist mode
Allowed MACs:
  - A4:CF:12:11:22:33  (Laptop An)
  - F8:FF:C2:44:55:66  (iPhone An)  
  - DC:A6:32:77:88:99  (Smart TV)
  
→ Chỉ 3 thiết bị này mới kết nối được WiFi
→ NHƯNG: MAC spoofing rất dễ, nên đây KHÔNG phải security thực sự!
```

```bash
# Đổi MAC trên Linux (MAC spoofing) — chỉ mất 5 giây:
sudo ip link set wlan0 down
sudo ip link set wlan0 address AA:BB:CC:DD:EE:FF
sudo ip link set wlan0 up
# → Bây giờ thiết bị "giả vờ" là MAC AA:BB:CC:DD:EE:FF
```

### Scenario 2: Công ty — 802.1X + Port Security

```
ACCESS SWITCH CONFIGURATION (Cisco):

interface FastEthernet0/1
  description "Desk-101-Finance"
  switchport mode access
  switchport access vlan 10
  switchport port-security
  switchport port-security maximum 2
  switchport port-security violation shutdown
  switchport port-security mac-address sticky
  
→ Port chỉ cho phép TỐI ĐA 2 MAC addresses
→ Nếu MAC thứ 3 xuất hiện → SHUTDOWN port ngay!
→ "sticky" = tự học MAC đầu tiên và save vào config
```

**Kết hợp 802.1X:** Thay vì trust MAC (dễ spoof), dùng certificate/username+password để xác thực thiết bị trước khi cho phép vào mạng.

### Scenario 3: Data Center — VXLAN và Virtual MAC

Trong data center với hàng nghìn VMs, mỗi VM có 1 virtual MAC. **VXLAN** encapsulate frame với virtual MAC bên trong UDP packet → truyền qua Layer 3 network:

```
┌─────── VXLAN Outer ──────┬──── VXLAN Inner (original frame) ────┐
│ Outer Dst MAC │ Outer IP  │ Inner Dst MAC │ Inner Src MAC │ Data │
│ (physical)    │ (VTEP IP) │ (VM-B MAC)    │ (VM-A MAC)   │      │
└───────────────┴───────────┴───────────────┴───────────────┴──────┘
```

---

## 9. Bài tập thực hành

### Exercise 1: Xem và phân tích MAC Address

```bash
# Linux — xem tất cả interfaces
ip link show
# hoặc
cat /sys/class/net/*/address

# Xác định nhà sản xuất từ OUI
# MAC: A4:CF:12:xx:xx:xx → OUI = A4:CF:12
# Tra: https://standards-oui.ieee.org/oui/oui.txt
# Kết quả: Intel Corporate

# Kiểm tra MAC có phải random/local không:
# A4 = 1010 0100 → bit 1 = 0 → Globally Unique (BIA)
# 02 = 0000 0010 → bit 1 = 1 → Locally Administered

# Python script phân tích:
python3 -c "
mac = 'A4:CF:12:3C:4D:5E'
first_byte = int(mac.split(':')[0], 16)
print(f'Multicast: {bool(first_byte & 0x01)}')
print(f'Local Admin: {bool(first_byte & 0x02)}')
"
```

### Exercise 2: MAC Address Table trên Switch

```bash
# Cisco IOS
show mac address-table
show mac address-table dynamic
show mac address-table count
show mac address-table interface fa0/1

# Linux bridge (nếu dùng bridge)
bridge fdb show
# hoặc
brctl showmacs br0

# Bài tập:
# 1. Clear MAC table, ping giữa 2 PC, xem MAC table populated
# 2. So sánh số entries trước và sau khi nhiều host communicate
# 3. Đợi 300s (aging time) → xem entries biến mất
```

### Exercise 3: Đổi MAC Address (Lab only!)

```bash
# Linux — thay đổi MAC
sudo ip link set eth0 down
sudo ip link set eth0 address 02:00:00:00:00:01
sudo ip link set eth0 up

# Verify
ip link show eth0

# macOS
sudo ifconfig en0 ether 02:00:00:00:00:01

# Windows (PowerShell as Admin)
Set-NetAdapter -Name "Ethernet" -MacAddress "02-00-00-00-00-01"

# Bài tập: Đổi MAC rồi ping gateway
# → ARP table trên gateway sẽ update với MAC mới
```

### Exercise 4: EUI-64 Calculation

```bash
# Cho MAC: 00:1A:2B:3C:4D:5E
# Tính EUI-64 cho IPv6:

# Bước 1: Chèn FF:FE giữa
# 00:1A:2B:FF:FE:3C:4D:5E

# Bước 2: Đảo bit U/L (bit 1 byte đầu)
# 00 (hex) = 00000000 (bin) → flip bit 1 → 00000010 = 02 (hex)
# → 02:1A:2B:FF:FE:3C:4D:5E

# Bước 3: IPv6 Link-Local
# fe80::021a:2bff:fe3c:4d5e

# Python verification:
python3 -c "
mac = '00:1A:2B:3C:4D:5E'
parts = mac.split(':')
# Insert FF:FE
eui64 = parts[:3] + ['FF', 'FE'] + parts[3:]
# Flip U/L bit
first = int(eui64[0], 16) ^ 0x02
eui64[0] = f'{first:02X}'
print('EUI-64:', ':'.join(eui64))
print('IPv6 LL: fe80::' + eui64[0]+eui64[1]+':'+eui64[2]+'ff:fe'+eui64[3]+':'+eui64[4]+eui64[5])
"
```

### Exercise 5: Detect MAC Spoofing

```bash
# Cách 1: Duplicate MAC detection
# Nếu 2 ports có cùng MAC → ai đó đang spoof!
# Cisco:
show mac address-table | include 0050.56c0.0001

# Cách 2: ARP inspection
# Nếu IP-MAC mapping không khớp DHCP snooping database → spoofing!

# Cách 3: Check OUI consistency
# Server Dell có MAC với OUI Dell? Hay OUI lạ?
# → Suspicious nếu OUI không match hardware vendor
```

---

## 10. Tóm tắt & Tài liệu đọc thêm

### Key Points — Ghi nhớ

| # | Concept | Điểm quan trọng |
|---|---------|-----------------|
| 1 | Format | 48 bits (6 bytes), viết hex, separated by : or - or . |
| 2 | OUI + Device ID | 3 bytes đầu = nhà sản xuất, 3 bytes sau = device unique |
| 3 | Bit 0 (I/G) | 0 = Unicast, 1 = Multicast/Broadcast |
| 4 | Bit 1 (U/L) | 0 = Globally Unique (burned-in), 1 = Locally Administered |
| 5 | Broadcast | FF:FF:FF:FF:FF:FF — gửi cho tất cả trong LAN |
| 6 | MAC Table | Switch HỌC source MAC → lưu vào table → forward thông minh |
| 7 | Aging | Default 300s — xóa MAC không hoạt động |
| 8 | Randomization | iOS/Android dùng random MAC cho privacy (LAA format) |
| 9 | EUI-64 | Mở rộng MAC 48-bit → 64-bit cho IPv6 Interface ID |
| 10 | AWS | ENI MAC = Locally Administered, persistent với ENI |

### Tài liệu đọc thêm

| # | Tài liệu | Link/Reference |
|---|----------|---------------|
| 1 | RFC 7042 — IEEE 802 Parameters | https://datatracker.ietf.org/doc/html/rfc7042 |
| 2 | IEEE OUI Lookup | https://standards-oui.ieee.org/oui/oui.txt |
| 3 | RFC 4291 — IPv6 Addressing (EUI-64) | https://datatracker.ietf.org/doc/html/rfc4291 |
| 4 | RFC 4941 — Privacy Extensions for IPv6 | https://datatracker.ietf.org/doc/html/rfc4941 |
| 5 | Cisco — Port Security Config Guide | https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst2960x/software/15-2_7_e/configuration_guide.html |
| 6 | AWS — ENI Documentation | https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html |

---

**Bài tiếp theo**: [Switching Fundamentals — MAC Table, Forwarding, và Store-and-Forward](/2026-06-01-switching-fundamentals)
