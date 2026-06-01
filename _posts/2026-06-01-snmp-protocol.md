---
layout: post
title: "SNMP Protocol Deep Dive - MIB Tree, OIDs, v1/v2c/v3, Traps vs Polling, Community Strings & Security"
date: 2026-06-01
categories: [networking]
tags: [snmp, monitoring, networking, network-management]
---

# SNMP Protocol Deep Dive — MIB Tree, OIDs, v1/v2c/v3, Traps vs Polling, Community Strings & Security

## Mục lục (Table of Contents)
1. [Giới thiệu bằng câu chuyện đời thường](#1-giới-thiệu-bằng-câu-chuyện-đời-thường)
2. [SNMP Architecture — Kiến trúc tổng quan](#2-snmp-architecture--kiến-trúc-tổng-quan)
3. [MIB Tree — Cây thông tin quản lý](#3-mib-tree--cây-thông-tin-quản-lý)
4. [OIDs — Định danh đối tượng](#4-oids--định-danh-đối-tượng)
5. [SNMP v1/v2c/v3 — Lịch sử và sự khác biệt](#5-snmp-v1v2cv3--lịch-sử-và-sự-khác-biệt)
6. [Polling vs Traps — Hai cách thu thập dữ liệu](#6-polling-vs-traps--hai-cách-thu-thập-dữ-liệu)
7. [Community Strings và Security Model](#7-community-strings-và-security-model)
8. [SNMP Operations — Các thao tác cơ bản](#8-snmp-operations--các-thao-tác-cơ-bản)
9. [Hands-on Lab và Tools](#9-hands-on-lab-và-tools)
10. [Tổng kết và Best Practices](#10-tổng-kết-và-best-practices)

---

## 1. Giới thiệu bằng câu chuyện đời thường

### SNMP như hệ thống giám sát tòa nhà

Hãy tưởng tượng bạn quản lý một tòa nhà lớn 50 tầng. Bạn cần biết:
- Nhiệt độ từng phòng (CPU temperature)
- Bao nhiêu người trong thang máy (network traffic)
- Đèn nào đang bật/tắt (interface up/down)
- Máy phát điện có hoạt động không (power supply status)

Bạn có 2 cách thu thập thông tin:

| Cách | Tương đương SNMP |
|---|---|
| **Cử bảo vệ đi kiểm tra từng tầng** (mỗi 5 phút) | **SNMP Polling** — NMS hỏi agent định kỳ |
| **Đặt chuông báo cháy** (khi có sự cố, tự báo) | **SNMP Trap** — Agent tự gửi alert |
| **Bảng điều khiển trung tâm** | **NMS (Network Management System)** |
| **Sensor nhiệt/cửa/khói** ở mỗi phòng | **SNMP Agent** trên mỗi device |
| **Sổ tay liệt kê mọi sensor có thể đọc** | **MIB (Management Information Base)** |
| **Mã số phòng** (Tầng.Khu.Phòng.Sensor = 5.2.3.1) | **OID (Object Identifier)** |

### Tại sao cần biết SNMP?

- **Infrastructure monitoring**: Zabbix, Nagios, PRTG, LibreNMS đều dùng SNMP
- **Network devices**: Router, switch, firewall expose metrics qua SNMP
- **Data centers**: Power, cooling, UPS monitoring
- **Cloud context**: CloudWatch thay thế SNMP phần nào, nhưng on-premises vẫn cần
- **Troubleshooting**: Kiểm tra interface errors, bandwidth usage, CPU load

---

## 2. SNMP Architecture — Kiến trúc tổng quan

### 2.1 Các thành phần chính

```
┌─────────────────────────────────────────────────────────────┐
│                    SNMP Architecture                          │
│                                                               │
│  ┌──────────────────┐                                       │
│  │   NMS (Manager)   │  ← Network Management System         │
│  │  (Zabbix, Nagios, │    (Phần mềm giám sát trung tâm)    │
│  │   LibreNMS)       │                                       │
│  └────────┬─────────┘                                       │
│           │ UDP:161 (queries)                                │
│           │ UDP:162 (traps)                                  │
│           ↕                                                   │
│  ┌────────┴──────────────────────────────────────────────┐  │
│  │            Managed Devices (with SNMP Agents)          │  │
│  │                                                        │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │  │
│  │  │ Router  │  │ Switch  │  │ Server  │  │Firewall │ │  │
│  │  │ Agent   │  │ Agent   │  │ Agent   │  │ Agent   │ │  │
│  │  │         │  │         │  │         │  │         │ │  │
│  │  │ [MIB]   │  │ [MIB]   │  │ [MIB]   │  │ [MIB]   │ │  │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘ │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

| Component | Vai trò | Ví dụ |
|---|---|---|
| **NMS (Manager)** | Thu thập, hiển thị, alert | Zabbix, Nagios, PRTG, LibreNMS |
| **SNMP Agent** | Chạy trên device, trả lời queries | snmpd (Linux), built-in (Cisco) |
| **MIB** | "Sổ tay" mô tả dữ liệu có thể đọc/ghi | IF-MIB, HOST-RESOURCES-MIB |
| **OID** | Địa chỉ cụ thể của 1 data point | .1.3.6.1.2.1.1.1.0 (sysDescr) |
| **Protocol** | Cách NMS và Agent nói chuyện | UDP 161 (query), UDP 162 (trap) |

### 2.2 SNMP Communication Ports

| Port | Protocol | Direction | Mục đích |
|---|---|---|---|
| UDP 161 | SNMP | Manager → Agent | Gửi GET/SET requests |
| UDP 162 | SNMP Trap | Agent → Manager | Gửi notifications (traps) |
| UDP 10161 | SNMP over TLS | Manager → Agent | Secure queries (SNMPv3) |
| UDP 10162 | SNMP Trap over TLS | Agent → Manager | Secure notifications |

---

## 3. MIB Tree — Cây thông tin quản lý

### 3.1 MIB là gì?

**MIB (Management Information Base)** là cơ sở dữ liệu phân cấp mô tả TẤT CẢ thông tin có thể query/set trên device. Nó giống **bản vẽ kiến trúc** — cho bạn biết có những "phòng" nào, mỗi phòng chứa gì.

### 3.2 Cấu trúc cây MIB

```
MIB Tree (OID Tree):
.
├── 1 (iso)
│   └── 3 (org)
│       └── 6 (dod - Department of Defense)
│           └── 1 (internet)
│               ├── 1 (directory)
│               ├── 2 (mgmt)
│               │   └── 1 (mib-2)  ← STANDARD MIBs
│               │       ├── 1 (system)
│               │       │   ├── 1 (sysDescr)
│               │       │   ├── 2 (sysObjectID)
│               │       │   ├── 3 (sysUpTime)
│               │       │   ├── 4 (sysContact)
│               │       │   ├── 5 (sysName)
│               │       │   ├── 6 (sysLocation)
│               │       │   └── 7 (sysServices)
│               │       ├── 2 (interfaces)
│               │       │   ├── 1 (ifNumber)
│               │       │   └── 2 (ifTable)
│               │       │       └── 1 (ifEntry)
│               │       │           ├── 1 (ifIndex)
│               │       │           ├── 2 (ifDescr)
│               │       │           ├── 5 (ifSpeed)
│               │       │           ├── 8 (ifOperStatus)
│               │       │           ├── 10 (ifInOctets)
│               │       │           └── 16 (ifOutOctets)
│               │       ├── 4 (ip)
│               │       ├── 6 (tcp)
│               │       ├── 7 (udp)
│               │       └── 25 (host - HOST-RESOURCES-MIB)
│               ├── 3 (experimental)
│               ├── 4 (private)
│               │   └── 1 (enterprises)  ← VENDOR-SPECIFIC
│               │       ├── 9 (cisco)
│               │       ├── 2021 (ucdavis/net-snmp)
│               │       ├── 2636 (juniper)
│               │       └── 8072 (netSnmp)
│               └── 6 (snmpV2)
```

### 3.3 MIB Modules phổ biến

| MIB Module | OID Prefix | Nội dung |
|---|---|---|
| **SNMPv2-MIB** (system) | .1.3.6.1.2.1.1 | System info, uptime, contact |
| **IF-MIB** (interfaces) | .1.3.6.1.2.1.2 | Interface status, traffic stats |
| **IP-MIB** | .1.3.6.1.2.1.4 | IP addressing, routing |
| **TCP-MIB** | .1.3.6.1.2.1.6 | TCP connections, stats |
| **UDP-MIB** | .1.3.6.1.2.1.7 | UDP stats |
| **HOST-RESOURCES-MIB** | .1.3.6.1.2.1.25 | CPU, memory, disk, processes |
| **ENTITY-MIB** | .1.3.6.1.2.1.47 | Physical components |
| **UCD-SNMP-MIB** | .1.3.6.1.4.1.2021 | Net-SNMP extensions |

### 3.4 MIB File Format (ASN.1)

```asn1
-- Ví dụ MIB definition
IF-MIB DEFINITIONS ::= BEGIN

IMPORTS
    MODULE-IDENTITY, OBJECT-TYPE, Counter32, Gauge32
        FROM SNMPv2-SMI;

ifMIB MODULE-IDENTITY
    LAST-UPDATED "200006140000Z"
    ORGANIZATION "IETF Interfaces MIB Working Group"
    DESCRIPTION  "The MIB module to describe generic objects for
                  network interface sub-layers."
    ::= { mib-2 31 }

-- Interface Entry
ifEntry OBJECT-TYPE
    SYNTAX      IfEntry
    MAX-ACCESS  not-accessible
    STATUS      current
    DESCRIPTION "An entry containing management information
                 applicable to a particular interface."
    INDEX   { ifIndex }
    ::= { ifTable 1 }

IfEntry ::= SEQUENCE {
    ifIndex      InterfaceIndex,
    ifDescr      DisplayString,
    ifType       IANAifType,
    ifSpeed      Gauge32,
    ifOperStatus INTEGER
}

ifOperStatus OBJECT-TYPE
    SYNTAX  INTEGER {
                up(1),        -- sẵn sàng truyền/nhận
                down(2),      -- không hoạt động
                testing(3)    -- đang test
            }
    MAX-ACCESS  read-only
    STATUS      current
    DESCRIPTION "The current operational state of the interface."
    ::= { ifEntry 8 }

END
```

---

## 4. OIDs — Định danh đối tượng

### 4.1 OID là gì?

**OID (Object Identifier)** là "địa chỉ" duy nhất cho mỗi data point trong MIB tree. Nó là chuỗi số phân cách bằng dấu chấm.

**Ví dụ đời thường**: OID giống **mã zip code mở rộng**: Quốc gia.Vùng.Thành phố.Quận.Đường.Số nhà = vị trí chính xác.

### 4.2 OID Format

```
.1.3.6.1.2.1.1.3.0
 │ │ │ │ │ │ │ │ └── Instance (0 = scalar, N = table row)
 │ │ │ │ │ │ │ └──── Object: sysUpTime (object 3 trong system group)
 │ │ │ │ │ │ └────── Group: system (group 1 trong mib-2)
 │ │ │ │ │ └──────── Module: mib-2 (module 1 trong mgmt)
 │ │ │ │ └────────── mgmt (subtree 2 trong internet)
 │ │ │ └──────────── internet (subtree 1 trong dod)
 │ │ └────────────── dod (Department of Defense, subtree 6 trong org)
 │ └──────────────── org (organization, subtree 3 trong iso)
 └────────────────── iso (standard 1)
```

### 4.3 OIDs phổ biến (phải nhớ!)

| OID | Tên | Mô tả | Ví dụ giá trị |
|---|---|---|---|
| `.1.3.6.1.2.1.1.1.0` | sysDescr | Mô tả hệ thống | "Linux server 5.15.0-..." |
| `.1.3.6.1.2.1.1.3.0` | sysUpTime | Uptime (hundredths of second) | 123456789 (14.3 days) |
| `.1.3.6.1.2.1.1.5.0` | sysName | Hostname | "core-router-01" |
| `.1.3.6.1.2.1.2.1.0` | ifNumber | Số interfaces | 8 |
| `.1.3.6.1.2.1.2.2.1.2.X` | ifDescr.X | Tên interface X | "GigabitEthernet0/1" |
| `.1.3.6.1.2.1.2.2.1.8.X` | ifOperStatus.X | Trạng thái interface X | 1 (up), 2 (down) |
| `.1.3.6.1.2.1.2.2.1.10.X` | ifInOctets.X | Bytes nhận vào interface X | 1234567890 |
| `.1.3.6.1.2.1.2.2.1.16.X` | ifOutOctets.X | Bytes gửi ra interface X | 9876543210 |
| `.1.3.6.1.4.1.2021.10.1.3.1` | laLoad.1 | Load average 1 min | "0.45" |
| `.1.3.6.1.4.1.2021.4.5.0` | memTotalReal | Total RAM (KB) | 16384000 |
| `.1.3.6.1.4.1.2021.4.6.0` | memAvailReal | Available RAM (KB) | 8192000 |

### 4.4 Scalar vs Tabular OIDs

```
Scalar (giá trị đơn — thêm .0 ở cuối):
  .1.3.6.1.2.1.1.1.0 = sysDescr (chỉ có 1 system description)
  .1.3.6.1.2.1.1.3.0 = sysUpTime (chỉ có 1 uptime)

Tabular (bảng — X = index hàng):
  .1.3.6.1.2.1.2.2.1.2.1 = ifDescr.1 (interface 1 description)
  .1.3.6.1.2.1.2.2.1.2.2 = ifDescr.2 (interface 2 description)
  .1.3.6.1.2.1.2.2.1.2.3 = ifDescr.3 (interface 3 description)
  
  ifTable:
  ┌───────┬──────────────────┬─────────┬──────────┐
  │ifIndex│ ifDescr          │ifSpeed  │ifOperStat│
  ├───────┼──────────────────┼─────────┼──────────┤
  │  1    │ "eth0"           │ 1Gbps   │  up(1)   │
  │  2    │ "eth1"           │ 1Gbps   │  down(2) │
  │  3    │ "lo"             │ 10Mbps  │  up(1)   │
  └───────┴──────────────────┴─────────┴──────────┘
```

---

## 5. SNMP v1/v2c/v3 — Lịch sử và sự khác biệt

### 5.1 Timeline

```
1988: SNMPv1 (RFC 1157) — Simple, plaintext community strings
1996: SNMPv2c (RFC 3416) — Better data types, GETBULK, still plaintext
2002: SNMPv3 (RFC 3414) — Authentication + Encryption + Access Control
```

### 5.2 So sánh chi tiết

| Feature | SNMPv1 | SNMPv2c | SNMPv3 |
|---|---|---|---|
| **Authentication** | Community string (plaintext) | Community string (plaintext) | Username + password (HMAC) |
| **Encryption** | ❌ None | ❌ None | ✅ DES, 3DES, AES-128/256 |
| **Access Control** | Community-based | Community-based | View-Based (VACM) |
| **Data types** | Basic (Counter32) | Extended (Counter64) | Same as v2c |
| **GETBULK** | ❌ | ✅ | ✅ |
| **INFORM** | ❌ | ✅ (acknowledged trap) | ✅ |
| **Error handling** | Basic | Better (more error codes) | Same as v2c |
| **RFC** | 1157 | 3416-3418 | 3410-3418 |

### 5.3 SNMPv1 — Đơn giản nhưng không an toàn

```
Message format:
┌──────────────────────────────────────────────┐
│ SNMP Version: 0 (v1)                         │
│ Community: "public" ← GỬI PLAINTEXT!         │
│ PDU: GetRequest/SetRequest/GetResponse/Trap  │
│   └── Variable Bindings: [{OID, Value}, ...] │
└──────────────────────────────────────────────┘
```

**Vấn đề**: Community string gửi plaintext → ai bắt packet đều thấy → có thể đọc/ghi device!

### 5.4 SNMPv2c — Tốt hơn, vẫn không an toàn

Cải thiện so với v1:
- **Counter64**: Cho counters > 4GB (ifHCInOctets cho 10Gbps+ interfaces)
- **GETBULK**: Lấy nhiều data 1 lần (thay vì GETNEXT lặp lại)
- **INFORM**: Trap có acknowledgement (đảm bảo Manager nhận được)
- Better error codes

**Vẫn KHÔNG có encryption** — community string vẫn plaintext.

### 5.5 SNMPv3 — An toàn cuối cùng

```
SNMPv3 Security:

┌──────────────────────────────────────────────────────────────┐
│ SNMPv3 Message                                                │
├──────────────────────────────────────────────────────────────┤
│ Header:                                                       │
│   msgVersion: 3                                               │
│   msgID: 12345                                                │
│   msgMaxSize: 65507                                           │
│   msgFlags: authPriv (auth=yes, priv=yes)                    │
│                                                               │
│ Security Parameters (USM):                                    │
│   msgAuthoritativeEngineID: 0x80001234...                    │
│   msgUserName: "adminUser"                                    │
│   msgAuthenticationParameters: HMAC-SHA-256(key, message)    │
│   msgPrivacyParameters: IV for AES encryption                │
│                                                               │
│ Scoped PDU (ENCRYPTED with AES):                             │
│   contextEngineID                                             │
│   contextName                                                 │
│   PDU: GetRequest [{OID, Value}, ...]                        │
└──────────────────────────────────────────────────────────────┘
```

**Ba Security Levels:**

| Level | Viết tắt | Authentication | Encryption | Khi nào dùng |
|---|---|---|---|---|
| noAuthNoPriv | `noauth` | ❌ Username only | ❌ | Test, non-sensitive |
| authNoPriv | `auth` | ✅ HMAC-MD5/SHA | ❌ | Xác thực, data không nhạy cảm |
| authPriv | `priv` | ✅ HMAC-MD5/SHA | ✅ DES/AES | **Production** (recommended) |

---

## 6. Polling vs Traps — Hai cách thu thập dữ liệu

### 6.1 SNMP Polling (Manager hỏi Agent)

**Ví dụ đời thường**: Bảo vệ đi tuần mỗi 30 phút, kiểm tra từng phòng.

```
NMS (Manager)                     Device (Agent)
  │                                    │
  │── GET sysUpTime.0 ───────────────→ │  Mỗi 5 phút
  │←── Response: 123456789 ──────────  │
  │                                    │
  │── GET ifInOctets.1 ──────────────→ │
  │←── Response: 98765432 ───────────  │
  │                                    │
  │── GETBULK ifTable ────────────────→ │  Lấy toàn bộ interface table
  │←── Response: [{ifDescr.1,...}, ...]─│
  │                                    │
  │     ... repeat every 5 minutes ... │
```

**Ưu điểm**:
- ✅ Predictable (NMS kiểm soát timing)
- ✅ Reliable (NMS biết chắc agent respond hay không)
- ✅ Consistent data collection

**Nhược điểm**:
- ❌ Delay (phải chờ tới lần poll tiếp theo mới biết vấn đề)
- ❌ Bandwidth (poll tất cả devices, kể cả khi không có gì thay đổi)
- ❌ Scale (1000 devices × 100 OIDs × mỗi 5 phút = nhiều traffic)

### 6.2 SNMP Traps (Agent tự báo Manager)

**Ví dụ đời thường**: Chuông báo cháy — chỉ kêu khi có sự cố, không cần kiểm tra thủ công.

```
Device (Agent)                     NMS (Manager)
  │                                    │
  │ [Interface goes down!]             │
  │── TRAP: linkDown (ifIndex=2) ────→ │  UDP 162
  │                                    │  (fire-and-forget)
  │                                    │
  │ [5 minutes later...]               │
  │ [CPU exceeds 90%!]                 │
  │── TRAP: cpuHigh (value=95%) ─────→ │
  │                                    │
```

**Trap types phổ biến:**

| Trap | OID | Ý nghĩa |
|---|---|---|
| coldStart | .1.3.6.1.6.3.1.1.5.1 | Device vừa restart (cold boot) |
| warmStart | .1.3.6.1.6.3.1.1.5.2 | Device vừa restart (warm boot) |
| linkDown | .1.3.6.1.6.3.1.1.5.3 | Interface went down |
| linkUp | .1.3.6.1.6.3.1.1.5.4 | Interface came up |
| authenticationFailure | .1.3.6.1.6.3.1.1.5.5 | Auth failure (wrong community) |

**Ưu điểm**:
- ✅ Real-time (biết ngay khi sự cố xảy ra)
- ✅ Low bandwidth (chỉ gửi khi có sự kiện)
- ✅ Efficient (không poll vô ích)

**Nhược điểm**:
- ❌ Unreliable (UDP, không có acknowledgement — trừ INFORM)
- ❌ Có thể mất trap nếu network congestion
- ❌ Không biết nếu agent "chết lặng" (không gửi trap = bình thường hay dead?)

### 6.3 Best Practice: Kết hợp cả hai

```
Strategy: Polling + Traps
─────────────────────────
1. POLLING (mỗi 5 phút):
   - Thu thập metrics baseline (bandwidth, CPU, memory)
   - Phát hiện devices "chết" (không respond)
   - Historical data collection (graphing)

2. TRAPS (real-time):
   - Interface up/down → alert ngay
   - Threshold exceeded → alert ngay
   - Authentication failure → security alert
   - Device restart → incident alert

3. INFORM (reliable trap):
   - Critical events cần đảm bảo delivery
   - Agent gửi INFORM → Manager reply ACK
   - Nếu không nhận ACK → Agent gửi lại
```

---

## 7. Community Strings và Security Model

### 7.1 Community Strings (v1/v2c)

**Community string** là "mật khẩu" (thực tế là access token) cho SNMP v1/v2c. Nó gửi **PLAINTEXT** trong mỗi SNMP packet.

```
Default values (MỌI NGƯỜI đều biết!):
  Read community:  "public"     ← Ai cũng đoán được!
  Write community: "private"    ← Nguy hiểm nếu không đổi!
```

**Vấn đề bảo mật:**

```
Attacker chạy tcpdump → bắt SNMP packet:
┌──────────────────────────────────────┐
│ UDP Src:10.0.0.100 Dst:10.0.0.1:161 │
│ SNMP Version: 2c                     │
│ Community: "MySecretC0mmunity"  ← LỘ!│
│ PDU: GetRequest                      │
└──────────────────────────────────────┘

→ Attacker có community string
→ Có thể đọc TOÀN BỘ device info
→ Nếu có write community → CÓ THỂ THAY ĐỔI CONFIG!
```

### 7.2 SNMPv3 USM (User-based Security Model)

```bash
# SNMPv3 user configuration (net-snmp)
# Tạo user với authPriv level

# /etc/snmp/snmpd.conf trên agent:
createUser myUser SHA "authPassw0rd123" AES "privPassw0rd456"
rwuser myUser priv

# Giải thích:
# - Username: myUser
# - Auth protocol: SHA (HMAC-SHA)
# - Auth password: authPassw0rd123 (dùng cho HMAC)
# - Priv protocol: AES (encryption)
# - Priv password: privPassw0rd456 (dùng cho AES key)
# - Access level: priv (requires both auth + encryption)
```

### 7.3 VACM (View-Based Access Control Model)

```
VACM cho phép kiểm soát chi tiết:
- User A chỉ đọc system info
- User B đọc/ghi interfaces
- User C đọc tất cả

Configuration:
┌─────────────────────────────────────────────────────────┐
│ User      │ Group       │ View          │ Access        │
├───────────┼─────────────┼───────────────┼───────────────┤
│ monitor   │ readonly    │ systemView    │ read-only     │
│ admin     │ readwrite   │ allView       │ read-write    │
│ netops    │ netgroup    │ ifView        │ read-only     │
└─────────────────────────────────────────────────────────┘

Views:
  systemView: .1.3.6.1.2.1.1 (chỉ system group)
  ifView:     .1.3.6.1.2.1.2 (chỉ interfaces)
  allView:    .1 (tất cả)
```

---

## 8. SNMP Operations — Các thao tác cơ bản

### 8.1 SNMP PDU Types

| Operation | Direction | Version | Mục đích |
|---|---|---|---|
| **GET** | Manager → Agent | v1/v2c/v3 | Lấy 1 OID cụ thể |
| **GETNEXT** | Manager → Agent | v1/v2c/v3 | Lấy OID tiếp theo (walk) |
| **GETBULK** | Manager → Agent | v2c/v3 | Lấy nhiều OIDs cùng lúc |
| **SET** | Manager → Agent | v1/v2c/v3 | Thay đổi giá trị |
| **RESPONSE** | Agent → Manager | v1/v2c/v3 | Trả lời GET/SET |
| **TRAP** | Agent → Manager | v1/v2c/v3 | Notification (fire & forget) |
| **INFORM** | Agent → Manager | v2c/v3 | Notification (acknowledged) |

### 8.2 GET Operation

```
Manager:
  "Cho tôi biết giá trị của OID .1.3.6.1.2.1.1.1.0"

Agent Response:
  .1.3.6.1.2.1.1.1.0 = STRING: "Linux server 5.15.0-91-generic"
```

### 8.3 GETNEXT và SNMP Walk

**GETNEXT** trả về OID TIẾP THEO trong MIB tree. Lặp lại nhiều lần = "SNMP Walk" (duyệt toàn bộ subtree).

```
Walk ifDescr:
GET NEXT .1.3.6.1.2.1.2.2.1.2   → Response: .1.3.6.1.2.1.2.2.1.2.1 = "eth0"
GET NEXT .1.3.6.1.2.1.2.2.1.2.1 → Response: .1.3.6.1.2.1.2.2.1.2.2 = "eth1"
GET NEXT .1.3.6.1.2.1.2.2.1.2.2 → Response: .1.3.6.1.2.1.2.2.1.2.3 = "lo"
GET NEXT .1.3.6.1.2.1.2.2.1.2.3 → Response: .1.3.6.1.2.1.2.2.1.3.1 = 6
                                               ^^^ Ra khỏi subtree → STOP
```

### 8.4 GETBULK (v2c/v3)

```
Thay vì 10 lần GETNEXT, dùng 1 lần GETBULK:
  GETBULK(non-repeaters=0, max-repetitions=10, OID=.1.3.6.1.2.1.2.2.1.2)
  
  Response (1 packet, 10 values):
  .1.3.6.1.2.1.2.2.1.2.1 = "eth0"
  .1.3.6.1.2.1.2.2.1.2.2 = "eth1"
  .1.3.6.1.2.1.2.2.1.2.3 = "lo"
  ... (tối đa 10 entries)
```

### 8.5 SET Operation

```bash
# Thay đổi system contact (cần write community hoặc v3 rw user)
snmpset -v2c -c private 10.0.0.1 .1.3.6.1.2.1.1.4.0 s "admin@company.com"

# Shutdown interface (nguy hiểm!)
snmpset -v2c -c private 10.0.0.1 .1.3.6.1.2.1.2.2.1.7.2 i 2
# ifAdminStatus.2 = 2 (down)
```

---

## 9. Hands-on Lab và Tools

### 9.1 Net-SNMP Tools (Linux)

```bash
# Install
sudo apt install snmp snmp-mibs-downloader  # Debian/Ubuntu
sudo yum install net-snmp-utils              # CentOS/RHEL

# GET single OID
snmpget -v2c -c public 10.0.0.1 sysDescr.0
snmpget -v2c -c public 10.0.0.1 .1.3.6.1.2.1.1.1.0

# WALK entire subtree
snmpwalk -v2c -c public 10.0.0.1 system
snmpwalk -v2c -c public 10.0.0.1 interfaces

# BULK GET (faster than walk for large tables)
snmpbulkwalk -v2c -c public 10.0.0.1 ifTable

# TABLE display (human-friendly)
snmptable -v2c -c public 10.0.0.1 ifTable

# SNMPv3 query
snmpget -v3 -u myUser -l authPriv \
  -a SHA -A "authPass123" \
  -x AES -X "privPass456" \
  10.0.0.1 sysUpTime.0

# Translate OID ↔ name
snmptranslate -On sysDescr.0
# → .1.3.6.1.2.1.1.1.0

snmptranslate .1.3.6.1.2.1.1.1.0
# → SNMPv2-MIB::sysDescr.0

# Full OID tree under a branch
snmptranslate -Tp .1.3.6.1.2.1.1
```

### 9.2 Cấu hình SNMP Agent (Linux snmpd)

```bash
# /etc/snmp/snmpd.conf

# SNMPv2c (for legacy/testing only)
rocommunity MyCommunity123 10.0.0.0/24
# ← Read-only, chỉ từ subnet 10.0.0.0/24

# SNMPv3 (recommended for production)
createUser monitorUser SHA-256 "AuthPassw0rd" AES "PrivPassw0rd"
rouser monitorUser priv -V systemView

# Views (what can be accessed)
view systemView included .1.3.6.1.2.1.1
view systemView included .1.3.6.1.2.1.2
view systemView included .1.3.6.1.2.1.25

# Trap destination
trap2sink 10.0.0.100 MyCommunity123
# ← Send traps to NMS at 10.0.0.100

# System info
sysLocation "DC1, Rack A5, Unit 12"
sysContact "ops@company.com"
sysName "web-server-01"

# Disk monitoring (send trap if >90%)
disk / 10%
disk /var 10%

# Process monitoring
proc httpd
proc sshd
```

```bash
# Restart agent
sudo systemctl restart snmpd

# Test locally
snmpwalk -v3 -u monitorUser -l authPriv -a SHA-256 -A "AuthPassw0rd" -x AES -X "PrivPassw0rd" localhost system
```

### 9.3 Monitoring với SNMP

```bash
# Tính bandwidth usage:
# Lấy ifInOctets tại t=0 và t=300 (5 phút)

t0_in=$(snmpget -v2c -c public -Oqv router ifInOctets.1)
sleep 300
t1_in=$(snmpget -v2c -c public -Oqv router ifInOctets.1)

# Bandwidth (bits/sec) = (t1 - t0) * 8 / interval
bandwidth_bps=$(( (t1_in - t0_in) * 8 / 300 ))
echo "Interface 1 inbound: $bandwidth_bps bps"

# Note: Counter32 wraps at 2^32 (4GB) — dùng ifHCInOctets (Counter64) cho 10G+
```

### 9.4 Trap Receiver

```bash
# snmptrapd — nhận traps trên UDP 162
# /etc/snmp/snmptrapd.conf:
authCommunity log,execute,net MyCommunity123
traphandle default /usr/local/bin/handle-trap.sh

# handle-trap.sh — script xử lý trap
#!/bin/bash
read host
read ip
while read oid val; do
  echo "$(date): Trap from $host ($ip): $oid = $val" >> /var/log/snmp-traps.log
done

# Start receiver
snmptrapd -f -Lo -c /etc/snmp/snmptrapd.conf
```

---

## 10. Tổng kết và Best Practices

### 10.1 SNMP Security Best Practices

| # | Practice | Lý do |
|---|---|---|
| 1 | **Dùng SNMPv3 authPriv** | Encryption + Authentication |
| 2 | **ĐỔI default community strings** | "public"/"private" quá dễ đoán |
| 3 | **ACL restrict SNMP access** | Chỉ cho phép NMS IP query |
| 4 | **Separate read/write communities** | Giảm damage nếu lộ read community |
| 5 | **Tắt SNMP write nếu không cần** | Giảm attack surface |
| 6 | **Firewall UDP 161/162** | Chỉ allow từ NMS network |
| 7 | **Monitor auth failures** | Phát hiện brute-force |
| 8 | **Rotate credentials** | Passwords không dùng mãi |

### 10.2 SNMP vs Modern Alternatives

| | SNMP | Prometheus | Telegraf/InfluxDB | Streaming Telemetry |
|---|---|---|---|---|
| Protocol | UDP (pull/push) | HTTP (pull) | Various (push) | gRPC (push) |
| Data model | MIB/OID tree | Metrics | Tags/Fields | YANG models |
| Performance | Medium | Good | Good | Best |
| Configuration | Complex (MIBs) | Simple (exporters) | Simple (plugins) | Complex |
| Vendor support | Universal | Growing | Good | Network vendors |
| Best for | Network devices | Cloud/containers | Hybrid | Modern routers |

### 10.3 Tài liệu tham khảo

| Tài liệu | Nội dung |
|---|---|
| RFC 1157 | SNMPv1 Protocol |
| RFC 3416-3418 | SNMPv2c Protocol |
| RFC 3410-3418 | SNMPv3 Framework |
| RFC 3414 | USM (User-based Security Model) |
| RFC 3415 | VACM (View-based Access Control) |
| RFC 2578 | SMIv2 (Structure of Management Information) |
| Net-SNMP docs | net-snmp.org documentation |

### 10.4 Câu hỏi ôn tập

1. SNMP architecture có những thành phần nào? Vai trò mỗi thành phần?
2. MIB tree cấu trúc thế nào? OID .1.3.6.1.2.1.1.3.0 nghĩa là gì?
3. SNMPv1 vs v2c vs v3: khác nhau thế nào? Tại sao phải dùng v3?
4. Polling vs Traps: ưu/nhược điểm? Tại sao nên kết hợp cả hai?
5. Community string là gì? Tại sao "public" nguy hiểm?
6. GET vs GETNEXT vs GETBULK: khác nhau thế nào?
7. SNMPv3 USM cung cấp 3 security levels nào?
8. VACM là gì và giải quyết vấn đề gì?
9. Làm sao tính bandwidth từ ifInOctets? Vấn đề Counter32 wrap?
10. SNMP đang được thay thế bởi gì trong infrastructure hiện đại?

---

*Bài viết được tham khảo từ RFC 1157 (SNMPv1), RFC 3410-3418 (SNMPv3), RFC 2578 (SMIv2), Net-SNMP documentation, và Cisco SNMP Configuration Guides.*
