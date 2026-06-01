---
layout: post
title: "Kubernetes Networking - Mạng Kubernetes chuyên sâu"
date: 2026-06-01
categories: [containers]
tags: [kubernetes, cni, calico, cilium, network-policy, coredns]
---

## Mục lục
1. [Góc nhìn tổng quan - Hệ thống giao thông thành phố K8s](#goc-nhin-tong-quan)
2. [CNI - Container Network Interface](#cni)
3. [Calico - Networking và Policy engine](#calico)
4. [Cilium - eBPF-based networking](#cilium)
5. [Flannel - Simple overlay networking](#flannel)
6. [Pod Networking - Mỗi Pod một IP](#pod-networking)
7. [kube-proxy - Service load balancing](#kube-proxy)
8. [CoreDNS - Service discovery](#coredns)
9. [Network Policies - Firewall trong K8s](#network-policies)
10. [Tổng kết và troubleshooting](#tong-ket)

---

## 1. Góc nhìn tổng quan - Hệ thống giao thông thành phố K8s {#goc-nhin-tong-quan}

### Ví dụ đời thường

K8s networking giống **hệ thống giao thông** một thành phố lớn:

- **CNI plugins** = nhà thầu xây đường (Calico = đường cao tốc, Flannel = đường quốc lộ đơn giản, Cilium = metro thông minh)
- **Pod networking** = mỗi nhà có địa chỉ duy nhất, ai cũng đến được bất kỳ nhà nào
- **kube-proxy** = bảng chỉ dẫn ở mỗi ngã tư (Service IP → Pod IPs)
- **CoreDNS** = tổng đài 1080 (hỏi tên → cho số/địa chỉ)
- **Network Policies** = luật giao thông (xe tải không vào phố cổ, chỉ xe buýt tuyến A đi đường B)
- **Ingress controller** = trạm thu phí + chỉ dẫn ở cửa ngõ thành phố

### K8s Networking Model (Quy tắc bắt buộc)

```
Kubernetes yêu cầu:
1. Mỗi Pod có 1 IP duy nhất (cluster-wide unique)
2. Pod-to-Pod: giao tiếp trực tiếp bằng IP (không NAT)
3. Pod-to-Service: qua virtual IP (ClusterIP)
4. External-to-Pod: qua NodePort/LoadBalancer/Ingress

K8s KHÔNG implement networking → delegate cho CNI plugins
```

---

## 2. CNI - Container Network Interface {#cni}

### CNI là gì?

**CNI** (Container Network Interface) là specification định nghĩa cách network plugins tương tác với container runtime. K8s gọi CNI plugin khi Pod được tạo/xóa.

```
Pod created → kubelet calls CNI plugin:
1. Tạo network namespace cho Pod
2. Tạo veth pair (Pod interface ← → host)
3. Gán IP address cho Pod
4. Setup routing
5. (Optional) Apply network policies

Pod deleted → kubelet calls CNI DEL:
1. Remove IP from Pod
2. Delete veth pair
3. Clean up routes
```

### So sánh CNI plugins phổ biến

```
┌──────────────┬───────────────┬───────────────┬──────────────┐
│ Feature      │ Calico        │ Cilium        │ Flannel      │
├──────────────┼───────────────┼───────────────┼──────────────┤
│ Dataplane    │ iptables/eBPF │ eBPF          │ VXLAN/host-gw│
│ Netpol       │ Yes (rich)    │ Yes (L3-L7)   │ No           │
│ Encryption   │ WireGuard     │ WireGuard/IPsec│ No          │
│ Performance  │ High          │ Highest       │ Good         │
│ Complexity   │ Medium        │ Medium-High   │ Low          │
│ Observability│ Flow logs     │ Hubble        │ Limited      │
│ Service Mesh │ No            │ Yes (sidecar-free)│ No       │
│ Use case     │ General       │ Advanced/Large │ Simple/Dev  │
└──────────────┴───────────────┴───────────────┴──────────────┘
```

---

## 3. Calico - Networking và Policy engine {#calico}

### Calico architecture

```
Calico sử dụng BGP routing (Layer 3):
- Mỗi node là BGP router
- Pod IP được advertise qua BGP
- Không cần overlay (no encapsulation overhead)
- OR: VXLAN mode cho cloud environments

Components:
- calico-node (DaemonSet): BIRD (BGP), Felix (policy)
- calico-kube-controllers: sync K8s → Calico datastore
- calico-typha: fan-out datastore watches (scale)
```

### Calico Network Policy (extended)

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: production
spec:
  selector: app == "api"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "web"
      destination:
        ports:
          - 8080
  egress:
    - action: Allow
      protocol: TCP
      destination:
        selector: app == "database"
        ports:
          - 5432
    - action: Allow
      protocol: UDP
      destination:
        ports:
          - 53    # DNS
```

---

## 4. Cilium - eBPF-based networking {#cilium}

### Cilium và eBPF

```
eBPF (extended Berkeley Packet Filter):
- Programs chạy TRONG Linux kernel (sandbox)
- Không cần iptables rules (thay thế kube-proxy!)
- XDP (eXpress Data Path): process packets trước khi tạo socket

Cilium leverage eBPF để:
- Networking: packet routing without iptables
- Load balancing: replace kube-proxy
- Network Policy: L3/L4/L7 enforcement
- Observability: Hubble flow visibility
- Encryption: transparent WireGuard
- Service mesh: sidecar-free (kernel-level)
```

### Cilium thay thế kube-proxy

```bash
# Khi dùng Cilium, có thể bỏ kube-proxy hoàn toàn
# Cilium handles Service load balancing via eBPF

# Performance improvement:
# iptables: O(n) rule traversal → slow at scale (>10K services)
# eBPF: O(1) hash map lookup → constant time regardless of scale

# Install Cilium without kube-proxy:
helm install cilium cilium/cilium \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=API_SERVER_IP \
  --set k8sServicePort=6443
```

### Hubble - Observability

```bash
# Hubble provides real-time network flow visibility
hubble observe --namespace production
# Shows: source pod → destination pod, port, protocol, verdict (allow/deny)

hubble observe --verdict DROPPED
# Shows all dropped packets (network policy violations)
```

---

## 5. Flannel - Simple overlay networking {#flannel}

```
Flannel = simplest CNI plugin:
- VXLAN overlay (encapsulate L2 in UDP)
- OR host-gw mode (direct routing, same subnet)
- No network policies (combine with Calico for policies)
- Good for: learning, development, simple clusters
```

---

## 6. Pod Networking - Mỗi Pod một IP {#pod-networking}

### Cách Pod nhận IP

```
1. Pod scheduled → kubelet → CNI ADD
2. CNI plugin assigns IP from CIDR range
3. Veth pair created (eth0 in pod, vethXXX on host)
4. Routes configured

Pod CIDR allocation:
- Cluster CIDR: 10.244.0.0/16 (toàn cluster)
- Node CIDR: 10.244.1.0/24 (node 1), 10.244.2.0/24 (node 2)
- Pod IPs: 10.244.1.5, 10.244.1.6, ... (trên node 1)
```

### Pod-to-Pod communication

```
Same node: Pod A (10.244.1.5) → Pod B (10.244.1.6)
  → Through linux bridge/veth pairs (local, fast)

Different nodes: Pod A (node1, 10.244.1.5) → Pod C (node2, 10.244.2.3)
  → Depends on CNI:
    - Calico BGP: direct L3 routing
    - Flannel VXLAN: encapsulate in UDP → send to node2 → decapsulate
    - Cilium: eBPF routing
```

---

## 7. kube-proxy - Service load balancing {#kube-proxy}

### kube-proxy modes

```
iptables mode (default):
- Creates iptables rules for each Service
- Random pod selection (equal probability)
- Scales poorly: 1000 services = 10,000+ rules
- Rule update: O(n) - slow at scale

IPVS mode (recommended for large clusters):
- Uses Linux IPVS kernel module
- Hash-based load balancing (round-robin, least-conn, etc.)
- O(1) connection routing
- Scales to 10,000+ services easily

eBPF mode (Cilium replaces kube-proxy):
- No iptables or IPVS rules
- eBPF programs at socket/XDP level
- Best performance and scalability
```

---

## 8. CoreDNS - Service discovery {#coredns}

### CoreDNS trong Kubernetes

```
CoreDNS = cluster DNS server:
- Resolves Service names to ClusterIPs
- Resolves Pod names (if enabled)
- Forwards external DNS to upstream

DNS format:
  <service>.<namespace>.svc.cluster.local
  web-service.production.svc.cluster.local → 10.96.0.15

  <pod-ip-dashed>.<namespace>.pod.cluster.local
  10-244-1-5.production.pod.cluster.local → 10.244.1.5
```

### CoreDNS Corefile

```
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

---

## 9. Network Policies - Firewall trong K8s {#network-policies}

### Network Policy là gì?

Network Policy = firewall rules cho Pods. Mặc định, tất cả Pods giao tiếp được với nhau. Network Policy giới hạn traffic.

```yaml
# Default deny all ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}     # Apply to ALL pods in namespace
  policyTypes:
    - Ingress         # Deny all incoming traffic

---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web
        - namespaceSelector:
            matchLabels:
              env: production
      ports:
        - protocol: TCP
          port: 8080
```

---

## 10. Tổng kết và troubleshooting {#tong-ket}

### Networking troubleshooting

```bash
# DNS resolution
kubectl exec -it debug-pod -- nslookup web-service
kubectl exec -it debug-pod -- nslookup web-service.production.svc.cluster.local

# Connectivity
kubectl exec -it debug-pod -- curl http://web-service:80
kubectl exec -it debug-pod -- wget -qO- http://web-service:80

# Network Policy debugging
kubectl describe networkpolicy -n production
kubectl get networkpolicy -A

# CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy
iptables -t nat -L KUBE-SERVICES | head -20
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| K8s Networking Model (k8s.io) | Official specification |
| CNI specification (github.com/containernetworking) | CNI standard |
| Calico documentation (projectcalico.org) | Calico guide |
| Cilium documentation (cilium.io) | Cilium guide |
| Network Policy Editor (editor.cilium.io) | Visual policy tool |

---

*Bài viết tiếp theo: [Container Orchestration Patterns](/2026/08/18/container-orchestration-patterns/)*
