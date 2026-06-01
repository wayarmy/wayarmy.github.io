---
layout: post
title: "Linux Networking Tools Deep Dive - ip, iptables, nftables, Namespaces"
date: 2026-06-01
categories: [linux]
tags: [ip-command, iptables, nftables, networking, namespaces]
---

# Linux Networking Tools Deep Dive - ip, iptables, nftables, Namespaces

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Quản trị mạng trên Linux giống như quản lý hệ thống giao thông thành phố:
- **ip command** = Sở Giao thông (quản lý đường, biển báo, địa chỉ nhà)
- **iptables/nftables** = Cảnh sát giao thông + Hải quan (kiểm soát ai được vào/ra, chặn xe vi phạm)
- **Network namespaces** = Thành phố trong thành phố (mỗi "thành phố" có hệ thống đường riêng biệt)
- **Bonding** = Gộp nhiều làn đường thành đường cao tốc (tăng bandwidth + dự phòng)
- **Bridge** = Cầu nối 2 khu vực (switch ảo trong kernel)
- **tc (traffic control)** = Hệ thống đèn giao thông + làn ưu tiên (QoS)

---

## 2. ip Command — Công Cụ Quản Lý Mạng Chính

### 2.1 ip addr — Quản Lý IP Addresses

```bash
# Xem tất cả interfaces + IPs
ip addr show
ip a                         # Shorthand

# Output giải thích:
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UP
#     link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff    ← MAC address
#     inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0  ← IPv4
#     inet6 fe80::42:acff:fe11:2/64 scope link              ← IPv6 link-local

# Thêm IP address
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip addr add 10.0.0.1/24 dev eth0 label eth0:1  # Secondary IP

# Xóa IP address  
sudo ip addr del 192.168.1.100/24 dev eth0

# Flush tất cả IPs trên interface
sudo ip addr flush dev eth0
```

### 2.2 ip link — Quản Lý Network Interfaces

```bash
# Xem interfaces
ip link show
ip l

# Bật/tắt interface
sudo ip link set eth0 up
sudo ip link set eth0 down

# Đổi MTU
sudo ip link set eth0 mtu 9000    # Jumbo frames

# Đổi MAC address
sudo ip link set eth0 down
sudo ip link set eth0 address 00:11:22:33:44:55
sudo ip link set eth0 up

# Tạo VLAN interface
sudo ip link add link eth0 name eth0.100 type vlan id 100
sudo ip addr add 10.100.0.1/24 dev eth0.100
sudo ip link set eth0.100 up

# Tạo bridge (software switch)
sudo ip link add name br0 type bridge
sudo ip link set eth0 master br0
sudo ip link set eth1 master br0
sudo ip link set br0 up

# Tạo veth pair (virtual ethernet cable)
sudo ip link add veth0 type veth peer name veth1
```

### 2.3 ip route — Quản Lý Routing Table

```bash
# Xem routing table
ip route show
ip r

# Output:
# default via 192.168.1.1 dev eth0 proto dhcp metric 100
# 192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.50
# 10.0.0.0/8 via 192.168.1.254 dev eth0

# Thêm route
sudo ip route add 10.0.0.0/8 via 192.168.1.254        # Via gateway
sudo ip route add 172.16.0.0/12 dev eth1               # Direct to interface
sudo ip route add default via 192.168.1.1              # Default gateway

# Xóa route
sudo ip route del 10.0.0.0/8

# Thay đổi default gateway
sudo ip route replace default via 192.168.1.1 dev eth0

# Policy routing (multiple routing tables)
sudo ip rule add from 10.0.0.0/8 table 100
sudo ip route add default via 10.0.0.1 table 100
# Traffic từ 10.0.0.0/8 dùng routing table 100 (khác default)

# Xem route cho destination cụ thể
ip route get 8.8.8.8
# 8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.50
```

### 2.4 ip neigh — ARP/Neighbor Table

```bash
# Xem ARP cache
ip neigh show
ip n

# Output:
# 192.168.1.1 dev eth0 lladdr 00:11:22:33:44:55 REACHABLE
# 192.168.1.100 dev eth0 lladdr aa:bb:cc:dd:ee:ff STALE

# States: REACHABLE, STALE, DELAY, PROBE, FAILED, PERMANENT

# Thêm static ARP entry
sudo ip neigh add 192.168.1.200 lladdr 00:11:22:33:44:55 dev eth0

# Flush ARP cache
sudo ip neigh flush all
```

### 2.5 ip netns — Network Namespaces

```bash
# Tạo namespace
sudo ip netns add ns1

# List namespaces
ip netns list

# Chạy command trong namespace
sudo ip netns exec ns1 ip addr show
sudo ip netns exec ns1 bash         # Shell trong namespace

# Kết nối 2 namespaces bằng veth:
sudo ip link add veth-host type veth peer name veth-ns1
sudo ip link set veth-ns1 netns ns1
sudo ip addr add 10.0.0.1/24 dev veth-host
sudo ip link set veth-host up
sudo ip netns exec ns1 ip addr add 10.0.0.2/24 dev veth-ns1
sudo ip netns exec ns1 ip link set veth-ns1 up
sudo ip netns exec ns1 ip link set lo up

# Test connectivity
ping 10.0.0.2                                  # Host → ns1
sudo ip netns exec ns1 ping 10.0.0.1          # ns1 → Host
```

---

## 3. iptables — Linux Firewall (Legacy nhưng phổ biến)

### 3.1 Kiến Trúc iptables

```
iptables architecture: Tables → Chains → Rules

TABLES (mỗi table có mục đích khác nhau):
├── filter (DEFAULT): Accept/Drop packets (firewall)
│   ├── INPUT: Packets đến local machine
│   ├── FORWARD: Packets đi qua (routing)
│   └── OUTPUT: Packets từ local machine đi ra
│
├── nat: Network Address Translation
│   ├── PREROUTING: Trước routing decision (DNAT)
│   ├── POSTROUTING: Sau routing decision (SNAT/Masquerade)
│   └── OUTPUT: NAT cho locally-generated packets
│
├── mangle: Packet header modification (QoS, TTL)
│   ├── All 5 chains
│
└── raw: Mark packets for no connection tracking
    ├── PREROUTING
    └── OUTPUT
```

**Packet Flow:**
```
INCOMING PACKET:
→ raw/PREROUTING → mangle/PREROUTING → nat/PREROUTING (DNAT)
→ Routing Decision
  ├── To local: mangle/INPUT → filter/INPUT → Local Process
  └── Forward:  mangle/FORWARD → filter/FORWARD 
                → mangle/POSTROUTING → nat/POSTROUTING (SNAT) → OUT

OUTGOING PACKET (from local):
Local Process → raw/OUTPUT → mangle/OUTPUT → nat/OUTPUT 
→ filter/OUTPUT → mangle/POSTROUTING → nat/POSTROUTING → OUT
```

### 3.2 iptables Rules

```bash
# Xem rules (filter table)
sudo iptables -L -n -v                    # List all chains
sudo iptables -L INPUT -n --line-numbers  # INPUT chain with line numbers

# Cú pháp:
# iptables -t <table> -A <chain> <match conditions> -j <target>
# -A = Append, -I = Insert (đầu), -D = Delete, -F = Flush

# === BASIC RULES ===

# Allow established/related connections (STATEFUL!)
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Allow SSH from specific subnet
sudo iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT

# Allow HTTP/HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow ping (ICMP)
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# Drop everything else (DEFAULT DENY)
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT    # Allow all outgoing

# === NAT RULES ===

# Masquerade (SNAT for dynamic IP — typical for NAT gateway)
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# DNAT (Port forwarding: external port 8080 → internal 10.0.0.5:80)
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.5:80

# Enable IP forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

# === RATE LIMITING ===
# Limit SSH connections (anti brute-force)
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -m limit --limit 3/min --limit-burst 3 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -m conntrack --ctstate NEW -j DROP

# === LOGGING ===
sudo iptables -A INPUT -j LOG --log-prefix "IPTables-Dropped: " --log-level 4
```

### 3.3 Persistence

```bash
# iptables rules mất khi reboot! Save/Restore:
sudo iptables-save > /etc/iptables/rules.v4
sudo iptables-restore < /etc/iptables/rules.v4

# Trên Debian/Ubuntu: install iptables-persistent
sudo apt install iptables-persistent
sudo netfilter-persistent save
sudo netfilter-persistent reload
```

---

## 4. nftables — Firewall Thế Hệ Mới (Thay Thế iptables)

### 4.1 Tại Sao nftables?

```
iptables issues:
- Separate tools: iptables, ip6tables, arptables, ebtables
- Performance: linear rule matching
- Complex syntax for advanced features
- Cannot do sets, maps, concatenation natively

nftables fixes:
- Single tool: nft (handles IPv4, IPv6, ARP, bridge)
- Performance: set/map lookups, binary tree
- Cleaner syntax
- Built-in: sets, maps, meters, concatenations
- Tracing built-in
```

### 4.2 nftables Syntax

```bash
# Xem rules
sudo nft list ruleset

# Tạo table
sudo nft add table inet filter    # inet = IPv4 + IPv6

# Tạo chain
sudo nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
sudo nft add chain inet filter output { type filter hook output priority 0 \; policy accept \; }

# Thêm rules
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input iif lo accept
sudo nft add rule inet filter input tcp dport 22 accept
sudo nft add rule inet filter input tcp dport { 80, 443 } accept
sudo nft add rule inet filter input icmp type echo-request accept

# Sets (nhóm giá trị — efficient lookup!)
sudo nft add set inet filter allowed_ports { type inet_service \; }
sudo nft add element inet filter allowed_ports { 22, 80, 443, 8080 }
sudo nft add rule inet filter input tcp dport @allowed_ports accept

# Rate limiting
sudo nft add rule inet filter input tcp dport 22 ct state new meter ssh-meter { ip saddr limit rate 3/minute } accept

# NAT
sudo nft add table inet nat
sudo nft add chain inet nat postrouting { type nat hook postrouting priority 100 \; }
sudo nft add rule inet nat postrouting oifname "eth0" masquerade

# Save/Load
sudo nft list ruleset > /etc/nftables.conf
sudo nft -f /etc/nftables.conf
```

### 4.3 nftables File Format

```
#!/usr/sbin/nft -f
# /etc/nftables.conf

table inet filter {
    set allowed_ips {
        type ipv4_addr
        elements = { 10.0.0.0/8, 192.168.0.0/16 }
    }

    chain input {
        type filter hook input priority 0; policy drop;
        
        ct state established,related accept
        iif lo accept
        
        ip saddr @allowed_ips tcp dport 22 accept
        tcp dport { 80, 443 } accept
        icmp type echo-request accept
        
        # Log dropped packets
        log prefix "nft-dropped: " counter drop
    }
    
    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

---

## 5. Network Bonding — Gộp Interface

### 5.1 Bonding Modes

```
Mode 0: balance-rr (Round Robin)
  - Packets gửi lần lượt qua mỗi interface
  - Tăng throughput + fault tolerance
  - Cần switch hỗ trợ

Mode 1: active-backup
  - Chỉ 1 interface active, còn lại standby
  - Interface active chết → standby lên thay
  - KHÔNG cần switch configuration đặc biệt
  - Phổ biến nhất cho HA

Mode 2: balance-xor
  - Hash-based (src MAC XOR dst MAC) chọn interface
  - Deterministic: cùng src+dst luôn cùng interface

Mode 4: 802.3ad (LACP)
  - IEEE standard, CẦN switch hỗ trợ LACP
  - Dynamic aggregation
  - Best throughput + fault tolerance
  - Enterprise standard

Mode 5: balance-tlb (Adaptive Transmit Load Balancing)
  - Không cần switch support
  - TX: balance theo slave load
  - RX: trên 1 interface (current active)

Mode 6: balance-alb (Adaptive Load Balancing)
  - Không cần switch support  
  - TX + RX load balancing
  - Uses ARP negotiation
```

### 5.2 Tạo Bond

```bash
# Tạo bond interface
sudo ip link add bond0 type bond mode 802.3ad

# Thêm slave interfaces
sudo ip link set eth0 master bond0
sudo ip link set eth1 master bond0

# Configure IP
sudo ip addr add 192.168.1.10/24 dev bond0
sudo ip link set bond0 up

# Xem bond status
cat /proc/net/bonding/bond0
```

---

## 6. tc (Traffic Control) — QoS Trên Linux

### 6.1 tc Cơ Bản

```bash
# Giới hạn bandwidth trên interface
# Limit download to 10 Mbps
sudo tc qdisc add dev eth0 root tbf rate 10mbit burst 32kbit latency 400ms

# Xem qdisc hiện tại
tc qdisc show dev eth0

# Xóa
sudo tc qdisc del dev eth0 root

# Simulate network latency (testing)
sudo tc qdisc add dev eth0 root netem delay 100ms 20ms
# Thêm 100ms delay ± 20ms jitter

# Simulate packet loss
sudo tc qdisc add dev eth0 root netem loss 5%
# 5% packet loss (cho testing!)
```

---

## 7. Practical Scenarios

### 7.1 Scenario: Bastion Host Firewall

```bash
# Server chỉ cho SSH từ office + HTTP/S từ anywhere
sudo nft -f - << 'EOF'
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        ct state established,related accept
        iif lo accept
        ip saddr { 203.0.113.0/24 } tcp dport 22 accept  # SSH from office
        tcp dport { 80, 443 } accept                       # HTTP/S from anywhere
        icmp type echo-request limit rate 5/second accept  # Ping limited
        counter drop
    }
    chain output {
        type filter hook output priority 0; policy accept;
    }
}
EOF
```

---

## 8. Tổng Kết và Tài Liệu Tham Khảo

### 8.1 Key Takeaways

1. **ip command thay thế ifconfig/route/arp** — học ip addr/link/route/neigh/netns
2. **iptables**: Tables → Chains → Rules, packet flow qua PREROUTING → INPUT/FORWARD → POSTROUTING
3. **nftables thay thế iptables** — syntax sạch hơn, sets/maps, single tool
4. **Network namespaces** = isolated network stacks (basis of containers)
5. **Bonding mode 1** (active-backup) đơn giản nhất; **mode 4** (LACP) hiệu quả nhất
6. **tc** cho QoS + network simulation (delay, loss)
7. **Stateful firewall** (conntrack) = rule ESTABLISHED,RELATED accept — luôn đặt đầu tiên!
8. **Default DENY** policy = chỉ cho phép traffic explicitly allowed

### 8.2 Tài Liệu Tham Khảo

- man pages: ip(8), iptables(8), nft(8), tc(8)
- nftables wiki: https://wiki.nftables.org/
- iproute2 documentation
- Linux Advanced Routing & Traffic Control (LARTC) HOWTO
- Arch Wiki: nftables, Network bridge, Bonding
- Netfilter documentation: https://www.netfilter.org/documentation/

---

*Bài viết tiếp theo: Shell Scripting Advanced — trap, getopts, process substitution*
