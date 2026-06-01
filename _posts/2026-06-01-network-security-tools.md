---
layout: post
title: "Network Security Tools - Công cụ bảo mật mạng"
date: 2026-06-01
categories: [security]
tags: [iptables, nmap, wireshark, tcpdump, ids, firewall]
---

## Mục lục
1. [Góc nhìn tổng quan - Arsenal bảo mật](#overview)
2. [iptables Deep Dive - Tables, Chains, Rules](#iptables)
3. [nftables - Thế hệ mới](#nftables)
4. [fail2ban Advanced Configuration](#fail2ban)
5. [nmap - Network Scanner](#nmap)
6. [Wireshark - Packet Analysis](#wireshark)
7. [tcpdump - CLI Packet Capture](#tcpdump)
8. [Snort/Suricata - IDS/IPS](#ids-ips)
9. [Network forensics workflow](#forensics)
10. [Tổng kết](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

Công cụ bảo mật mạng giống **bộ dụng cụ bảo vệ**:
- **iptables** = tường thành + cổng kiểm soát (firewall rules)
- **nmap** = kính viễn vọng trinh sát (scan xem network có gì)
- **Wireshark** = kính hiển vi (phân tích từng packet chi tiết)
- **tcpdump** = máy ghi âm mạng (capture raw packets)
- **Snort/Suricata** = hệ thống cảnh báo xâm nhập (IDS/IPS)

---

## 2. iptables Deep Dive {#iptables}

```bash
# iptables architecture:
# Tables → Chains → Rules

# TABLES:
# filter (default): Accept/Drop/Reject packets
# nat: Network Address Translation
# mangle: Modify packet headers
# raw: Connection tracking exceptions

# CHAINS:
# INPUT: Packets destined to this host
# OUTPUT: Packets originated from this host
# FORWARD: Packets passing through (routing)
# PREROUTING: Before routing decision (DNAT)
# POSTROUTING: After routing decision (SNAT/MASQUERADE)

# Packet flow:
# Incoming: PREROUTING → (routing) → INPUT → process
# Outgoing: process → OUTPUT → (routing) → POSTROUTING
# Forwarding: PREROUTING → (routing) → FORWARD → POSTROUTING

# Basic rules:
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j DROP
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -P INPUT DROP   # Default policy: drop all

# NAT (masquerade):
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Port forwarding:
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 10.0.0.5:80

# Rate limiting:
iptables -A INPUT -p tcp --dport 22 -m recent --set --name ssh
iptables -A INPUT -p tcp --dport 22 -m recent --update --seconds 60 --hitcount 4 --name ssh -j DROP

# Logging:
iptables -A INPUT -j LOG --log-prefix "IPT-DROP: " --log-level 4
iptables -A INPUT -j DROP
```

---

## 3-5. nftables, fail2ban, nmap {#nmap}

### nmap Scanning Techniques
```bash
# Host discovery
nmap -sn 192.168.1.0/24              # Ping scan (no port scan)

# Port scanning
nmap -sS 192.168.1.1                  # SYN scan (stealth, default)
nmap -sT 192.168.1.1                  # TCP connect scan
nmap -sU 192.168.1.1                  # UDP scan
nmap -sV 192.168.1.1                  # Service version detection
nmap -O 192.168.1.1                   # OS detection
nmap -A 192.168.1.1                   # Aggressive (OS + version + scripts)

# Specific ports
nmap -p 80,443,8080 192.168.1.1       # Specific ports
nmap -p- 192.168.1.1                  # ALL 65535 ports
nmap --top-ports 1000 192.168.1.1     # Top 1000 common ports

# NSE Scripts
nmap --script vuln 192.168.1.1        # Vulnerability scan
nmap --script ssl-enum-ciphers -p 443 example.com  # SSL audit
```

---

## 6-7. Wireshark & tcpdump {#tcpdump}

```bash
# tcpdump basics
tcpdump -i eth0                        # Capture on interface
tcpdump -i eth0 port 80               # HTTP traffic
tcpdump -i eth0 host 10.0.0.1         # Traffic to/from host
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'  # SYN packets

# Save to file (open in Wireshark later)
tcpdump -i eth0 -w capture.pcap -c 1000

# Useful filters
tcpdump -i eth0 'port 443 and host api.example.com'
tcpdump -i eth0 -A 'port 80'          # Show ASCII content
tcpdump -i eth0 'icmp'                # Ping/ICMP only

# Wireshark display filters (GUI):
# http.request.method == "POST"
# tcp.flags.syn == 1 && tcp.flags.ack == 0
# dns.qry.name contains "example"
# tls.handshake.type == 1              # Client Hello
```

---

## 8-10. IDS/IPS, Forensics {#ids-ips}

### Snort/Suricata
```
IDS = Intrusion Detection System (alert only)
IPS = Intrusion Prevention System (alert + block)

Suricata rule example:
alert http any any -> any any (
  msg:"SQL Injection attempt";
  content:"UNION"; nocase;
  content:"SELECT"; nocase;
  sid:1000001; rev:1;
)

alert tcp any any -> any 22 (
  msg:"SSH brute force";
  flow:to_server;
  threshold: type both, track by_src, count 5, seconds 60;
  sid:1000002; rev:1;
)
```

---
