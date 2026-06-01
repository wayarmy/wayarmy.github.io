---
layout: post
title: "Docker Networking Deep Dive - Mạng Docker chuyên sâu"
date: 2026-06-01
categories: [containers]
tags: [docker, bridge, overlay, macvlan, vxlan, networking]
---

## Mục lục
1. [Góc nhìn tổng quan - Khu phố containers](#goc-nhin-tong-quan)
2. [Bridge Network - Mạng cầu nội bộ](#bridge)
3. [docker0, veth pairs và network namespaces](#docker0-veth)
4. [iptables NAT và port mapping](#iptables-nat)
5. [Host Network - Chia sẻ mạng host](#host-network)
6. [Overlay Network - Mạng phủ đa host](#overlay)
7. [VXLAN - Công nghệ đằng sau overlay](#vxlan)
8. [Macvlan/IPvlan - Container trên mạng vật lý](#macvlan)
9. [DNS và Service Discovery trong Docker](#dns)
10. [Tổng kết và troubleshooting](#tong-ket)

---

## 1. Góc nhìn tổng quan - Khu phố containers {#goc-nhin-tong-quan}

### Ví dụ đời thường

Hãy tưởng tượng Docker host là một **tòa chung cư**:

- **Bridge network** = mạng nội bộ trong tòa nhà - các căn hộ (containers) nói chuyện được với nhau qua intercom, nhưng muốn ra ngoài phải qua reception (NAT)
- **Host network** = container không có tường riêng, dùng chung địa chỉ với tòa nhà
- **Overlay network** = hệ thống intercom nối NHIỀU tòa nhà - căn hộ ở tòa A gọi được căn hộ ở tòa B qua tunnel ngầm
- **Macvlan** = mỗi container có số nhà riêng trên phố (MAC address riêng, giống máy vật lý)
- **None** = phòng cách ly, không có mạng

### Kiến trúc Docker Networking

```
┌─────────────────────────────────────────────────────┐
│                    Docker Host                        │
│                                                      │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐             │
│  │Container│  │Container│  │Container│             │
│  │  eth0   │  │  eth0   │  │  eth0   │             │
│  └────┬────┘  └────┬────┘  └────┬────┘             │
│       │veth        │veth        │veth               │
│  ┌────┴────────────┴────────────┴────┐              │
│  │         docker0 (bridge)          │              │
│  │         172.17.0.1/16             │              │
│  └───────────────┬───────────────────┘              │
│                  │                                   │
│              iptables NAT                            │
│                  │                                   │
│  ┌───────────────┴───────────────────┐              │
│  │            eth0 (host)            │              │
│  │         192.168.1.100             │              │
│  └───────────────────────────────────┘              │
└─────────────────────────────────────────────────────┘
                   │
              Physical Network
```

---

## 2. Bridge Network - Mạng cầu nội bộ {#bridge}

### Bridge network là gì?

Bridge là network driver **mặc định** của Docker. Nó tạo một virtual switch (software bridge) trên host, containers kết nối vào bridge và giao tiếp với nhau.

### Default bridge vs User-defined bridge

```bash
# Default bridge (docker0) - tạo sẵn khi cài Docker
docker run --name c1 nginx          # Tự join docker0
docker run --name c2 alpine ping c1  # ❌ KHÔNG resolve được tên!

# User-defined bridge - TỐT HƠN
docker network create my-net
docker run --name c1 --network my-net nginx
docker run --name c2 --network my-net alpine ping c1  # ✅ OK!

# Khác biệt quan trọng:
# Default bridge:
# - Containers giao tiếp bằng IP only (không DNS)
# - Tất cả containers cùng network → kém isolation
# - Legacy, không recommended

# User-defined bridge:
# - DNS resolution tự động (container name → IP)
# - Isolation tốt hơn (chỉ containers cùng network)
# - Có thể connect/disconnect container on-the-fly
# - Better control over IP ranges
```

### Tạo và quản lý bridge

```bash
# Tạo network với subnet custom
docker network create \
  --driver bridge \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1 \
  --ip-range 10.10.0.128/25 \
  --opt "com.docker.network.bridge.name"="my-bridge" \
  my-net

# Kiểm tra
docker network inspect my-net

# Connect/Disconnect container
docker network connect my-net existing-container
docker network disconnect my-net existing-container

# Gán static IP
docker run --network my-net --ip 10.10.0.50 nginx
```

---

## 3. docker0, veth pairs và network namespaces {#docker0-veth}

### Network Namespaces

```
Mỗi container chạy trong network namespace riêng:
- Có interface riêng (eth0)
- Có routing table riêng
- Có iptables rules riêng
- Hoàn toàn isolated từ host

Ví dụ: Mỗi căn hộ có đường dây điện thoại riêng,
không nghe được cuộc gọi của hàng xóm.
```

### veth pairs

```
veth (Virtual Ethernet) pairs = "ống dẫn" 2 đầu:
- 1 đầu trong container (eth0)
- 1 đầu trên host (vethXXX), gắn vào bridge

┌──────────────┐     veth pair     ┌─────────────────┐
│  Container   │                   │   Docker Host    │
│              │                   │                  │
│  ┌────────┐  │   ┌───────────┐  │  ┌───────────┐  │
│  │  eth0  │──┼───│  veth123  │──┼──│  docker0  │  │
│  └────────┘  │   └───────────┘  │  └───────────┘  │
│  10.0.0.2    │                   │  10.0.0.1       │
└──────────────┘                   └─────────────────┘
```

### Xem chi tiết networking

```bash
# Xem bridge trên host
brctl show
# OR
ip link show type bridge

# Xem veth pairs
ip link show type veth

# Xem network namespace của container
docker inspect -f '{{.State.Pid}}' container_name
# → PID 12345
nsenter -t 12345 -n ip addr     # Xem interfaces trong container
nsenter -t 12345 -n ip route    # Xem routing trong container

# Xem docker0 bridge details
ip addr show docker0
bridge link show docker0        # Xem ports attached
```

---

## 4. iptables NAT và port mapping {#iptables-nat}

### Cơ chế port mapping

```bash
# Khi chạy: docker run -p 8080:80 nginx
# Docker tạo iptables rules:

# 1. DNAT rule (Destination NAT)
# Traffic đến host:8080 → redirect vào container:80
iptables -t nat -A DOCKER -p tcp --dport 8080 \
  -j DNAT --to-destination 172.17.0.2:80

# 2. MASQUERADE rule (Source NAT)
# Traffic từ container ra ngoài → đổi source IP thành host IP
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 \
  ! -o docker0 -j MASQUERADE

# 3. FORWARD rule
# Cho phép traffic đi qua
iptables -A FORWARD -i docker0 -o eth0 -j ACCEPT
iptables -A FORWARD -i eth0 -o docker0 -j ACCEPT
```

### Xem iptables rules Docker tạo

```bash
# NAT rules
iptables -t nat -L -n -v | grep -A5 DOCKER

# Filter rules
iptables -L DOCKER -n -v

# Docker chain flow:
# PREROUTING → DOCKER (DNAT)
# FORWARD → DOCKER-USER → DOCKER-ISOLATION → DOCKER
# POSTROUTING → MASQUERADE
```

---

## 5. Host Network - Chia sẻ mạng host {#host-network}

### Host network là gì?

Container **dùng trực tiếp** network stack của host - không network namespace riêng, không NAT, không port mapping.

```bash
# Container dùng host network
docker run --network host nginx
# nginx listen :80 → trực tiếp trên host:80
# Không cần -p flag!
```

### Khi nào dùng host network?

```
✅ Dùng khi:
- Cần performance tối đa (no NAT overhead)
- Application bind nhiều ports động (range)
- Network monitoring tools (tcpdump, Wireshark)
- Load testing tools

❌ Không dùng khi:
- Cần isolation (host network = no isolation)
- Chạy nhiều containers cùng port
- Production workloads cần security
```

---

## 6. Overlay Network - Mạng phủ đa host {#overlay}

### Overlay network là gì?

Overlay network cho phép containers trên **các host khác nhau** giao tiếp với nhau, giống như chúng ở cùng 1 network. Dùng trong Docker Swarm và Kubernetes.

```
┌───────────────────┐              ┌───────────────────┐
│     Host 1        │              │     Host 2        │
│                   │              │                   │
│  ┌──────┐ ┌──────┐│    VXLAN    │┌──────┐ ┌──────┐  │
│  │ C1   │ │ C2   ││   tunnel   ││ C3   │ │ C4   │  │
│  │10.0.1│ │10.0.2││◄──────────►││10.0.3│ │10.0.4│  │
│  └──┬───┘ └──┬───┘│            │└──┬───┘ └──┬───┘  │
│     └────┬───┘    │            │   └────┬───┘      │
│     [overlay br]  │            │   [overlay br]    │
│          │        │            │        │          │
│    [VXLAN encap]  │            │  [VXLAN decap]    │
│          │        │            │        │          │
│      [eth0]───────┼────────────┼────[eth0]         │
│  192.168.1.10     │   Physical │  192.168.1.20     │
└───────────────────┘   Network  └───────────────────┘
```

### Tạo overlay network

```bash
# Cần Docker Swarm hoặc external key-value store
docker swarm init

# Tạo overlay network
docker network create \
  --driver overlay \
  --subnet 10.10.0.0/16 \
  --attachable \
  my-overlay

# Deploy service sử dụng overlay
docker service create --network my-overlay --name web nginx
docker service create --network my-overlay --name api node

# Containers trong overlay → giao tiếp bằng service name
# web có thể ping api, dù ở host khác nhau!
```

---

## 7. VXLAN - Công nghệ đằng sau overlay {#vxlan}

### VXLAN là gì?

**VXLAN** (Virtual Extensible LAN, RFC 7348) đóng gói Layer 2 frames vào UDP packets, cho phép tạo virtual network layer 2 trên layer 3 infrastructure.

```
Original frame (bên trong tunnel):
┌─────────────────────────────────────┐
│ Inner Ethernet │ Inner IP │ Payload │
└─────────────────────────────────────┘

After VXLAN encapsulation:
┌─────────────────────────────────────────────────────────────┐
│ Outer Eth │ Outer IP │ UDP:4789 │ VXLAN Header │ Original  │
│           │          │          │ (VNI=100)    │ Frame     │
└─────────────────────────────────────────────────────────────┘

VXLAN Header chứa:
- VNI (VXLAN Network Identifier): 24-bit → 16 triệu networks!
  (so với VLAN chỉ 4096)
- UDP port 4789 (IANA standard)
```

### VXLAN trong Docker

```bash
# Docker overlay sử dụng VXLAN internally
# Mỗi overlay network = 1 VNI

# Kiểm tra VXLAN interface trên host
ip -d link show type vxlan
# vxlan1: ... id 4096 ... dstport 4789

# Xem FDB (Forwarding Database) - biết container ở host nào
bridge fdb show dev vxlan1
# 00:00:00:00:00:00 dst 192.168.1.20 self permanent
# → Container với MAC nào ở host IP nào
```

---

## 8. Macvlan/IPvlan - Container trên mạng vật lý {#macvlan}

### Macvlan là gì?

**Macvlan** gán MAC address riêng cho container, làm container xuất hiện như thiết bị vật lý trên network. Không cần NAT, không port mapping.

```bash
# Tạo macvlan network
docker network create \
  --driver macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  --opt parent=eth0 \
  my-macvlan

# Run container với IP trên physical network
docker run --network my-macvlan --ip 192.168.1.200 nginx
# → nginx accessible trực tiếp tại 192.168.1.200!
```

### Macvlan vs IPvlan

```
Macvlan:
- Mỗi container có MAC address riêng
- Cần network hỗ trợ multiple MACs per port
- Một số cloud providers chặn (AWS, Azure)

IPvlan (L2 mode):
- Tất cả containers dùng chung MAC với host
- Phân biệt bằng IP
- Hoạt động ở môi trường restrictive hơn

IPvlan (L3 mode):
- Layer 3 routing (no bridging)
- Container mỗi đứa 1 subnet
- Host là router
```

---

## 9. DNS và Service Discovery trong Docker {#dns}

### Docker embedded DNS (127.0.0.11)

```bash
# User-defined networks có DNS server built-in
# Container tên "web" → resolve "web" → container IP

# DNS server address: 127.0.0.11 (trong container)
docker exec container cat /etc/resolv.conf
# nameserver 127.0.0.11
# options ndots:0

# Round-robin DNS cho services:
docker run --name web1 --network my-net nginx
docker run --name web2 --network my-net nginx
# "web1" → IP of web1
# "web2" → IP of web2

# Với Docker Compose service scaling:
# "web" → round-robin giữa tất cả replicas
```

### DNS aliases và links

```bash
# Network alias (1 tên → nhiều containers)
docker run --network my-net --network-alias db redis
docker run --network my-net --network-alias db postgres
# Resolve "db" → round-robin giữa redis và postgres

# Docker Compose automatic DNS
# services:
#   web:    → resolve "web"
#   api:    → resolve "api"
#   db:     → resolve "db"
```

---

## 10. Tổng kết và troubleshooting {#tong-ket}

### Bảng so sánh drivers

| Driver | Use Case | Multi-host | Performance | Isolation |
|--------|----------|-----------|-------------|-----------|
| bridge | Default, dev | No | Good | Good |
| host | Max performance | No | Best | None |
| overlay | Swarm/K8s | Yes | Good (VXLAN overhead) | Good |
| macvlan | Legacy integration | No | Excellent | Good |
| none | Security | N/A | N/A | Maximum |

### Troubleshooting commands

```bash
# Inspect network
docker network inspect bridge
docker network ls

# Container networking
docker exec container ip addr
docker exec container ip route
docker exec container cat /etc/resolv.conf

# Connectivity test
docker exec c1 ping c2
docker exec c1 nslookup c2
docker exec c1 curl http://c2:80

# Host-level debugging
iptables -t nat -L DOCKER -n -v
brctl show
ip link show type veth
tcpdump -i docker0 -n
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| Docker Networking Documentation | Hướng dẫn chính thức |
| RFC 7348: VXLAN | VXLAN specification |
| Container Networking (O'Reilly) | Sách chuyên sâu |
| Docker source: libnetwork | Implementation details |

---

*Bài viết tiếp theo: [Docker Storage & Volumes](/2026/08/13/docker-storage-volumes/) - Lưu trữ trong Docker*
