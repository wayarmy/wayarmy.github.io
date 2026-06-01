---
layout: post
title: "Zero Trust Architecture - Kiến trúc không tin tưởng"
date: 2026-06-01
categories: [security]
tags: [zero-trust, beyondcorp, ztna, micro-segmentation, vpc-lattice]
---

## Mục lục
1. [Góc nhìn tổng quan - Không tin ai cả!](#overview)
2. [Zero Trust Principles - Nguyên tắc cốt lõi](#principles)
3. [Micro-segmentation - Chia nhỏ để bảo vệ](#micro-segmentation)
4. [Identity-Aware Proxy (IAP)](#iap)
5. [Google BeyondCorp Model](#beyondcorp)
6. [ZTNA - Zero Trust Network Access](#ztna)
7. [AWS VPC Lattice](#vpc-lattice)
8. [AWS Verified Access](#verified-access)
9. [Implementation roadmap](#implementation)
10. [Tổng kết](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

**Mô hình cũ (Perimeter security)** = Lâu đài có hào nước:
- Bên ngoài hào = không tin (public internet)
- Bên trong hào = tin tưởng hoàn toàn (corporate network)
- Vấn đề: Kẻ xâm nhập vào được bên trong → tự do di chuyển!

**Zero Trust** = Mỗi phòng trong lâu đài có khóa riêng:
- Không tin ai, dù đang ở bên trong
- Mỗi lần vào phòng: kiểm tra danh tính + quyền + thiết bị
- "Never trust, always verify"

---

## 2. Zero Trust Principles {#principles}

```
1. NEVER TRUST, ALWAYS VERIFY
   - Mọi request đều phải authenticated + authorized
   - Không có "trusted zone"
   - Internal network ≠ trusted

2. LEAST PRIVILEGE ACCESS
   - Chỉ cấp quyền tối thiểu cần thiết
   - Just-in-time access (tạm thời, theo yêu cầu)
   - Just-enough-access (chỉ đủ, không thừa)

3. ASSUME BREACH
   - Thiết kế NHƯ THỂ đã bị hack
   - Micro-segmentation (limit blast radius)
   - Continuous monitoring và detection
   - Encrypt everything (even internal traffic)

4. VERIFY EXPLICITLY
   - Identity (who are you?)
   - Device health (is your device secure?)
   - Location/Network (where are you?)
   - Behavior/Risk (is this normal for you?)
```

---

## 3. Micro-segmentation {#micro-segmentation}

```
Traditional: Flat network (any device → any device)
Micro-segmented: Each workload in its own segment

┌────────────────────────────────────────────────────┐
│                 Without Segmentation                │
│  Web ←→ API ←→ DB ←→ Admin ←→ Everything          │
│  (Attacker compromises Web → reaches everything!)  │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│                 With Micro-segmentation             │
│  ┌─────┐    ┌─────┐    ┌─────┐                   │
│  │ Web │───▶│ API │───▶│ DB  │                    │
│  └─────┘    └─────┘    └─────┘                    │
│     ✗           ✗          ✗                       │
│  Cannot reach Admin, cannot reach other segments   │
│  (Attacker in Web → ONLY reaches API, nothing else)│
└────────────────────────────────────────────────────┘

Implementation:
- Kubernetes: Network Policies
- Cloud: Security Groups, NACLs
- Service Mesh: mTLS + AuthorizationPolicy
- Third-party: Illumio, Guardicore
```

---

## 4-5. Identity-Aware Proxy & BeyondCorp {#iap}

### Google BeyondCorp (2014)
```
Google's internal Zero Trust implementation:
- No VPN needed for corporate apps
- Every request authenticated at proxy
- Access decisions based on:
  * User identity (who)
  * Device inventory (managed? patched? encrypted?)
  * Context (time, location, behavior)
  * Application sensitivity level

Architecture:
User (any network) → Identity-Aware Proxy → Application

Proxy checks:
1. Valid identity? (SSO, MFA)
2. Known device? (enrolled in device management)
3. Device healthy? (OS patched, disk encrypted, no malware)
4. Access policy allows? (role + device trust level)
5. Only then → forward to application
```

---

## 6-8. ZTNA, VPC Lattice, Verified Access {#ztna}

### AWS VPC Lattice
```
VPC Lattice = Application-level networking service:
- Service-to-service connectivity across VPCs and accounts
- Built-in AuthN/AuthZ (IAM policies)
- No need for VPC peering, Transit Gateway for service access
- Automatic load balancing, health checks
- Works across VPCs, accounts, even EKS/ECS/Lambda

Architecture:
┌─────────────┐         ┌──────────────┐
│ Service A   │         │ VPC Lattice  │         ┌─────────────┐
│ (VPC 1)     │────────▶│ Service      │────────▶│ Service B   │
│             │         │ Network      │         │ (VPC 2)     │
└─────────────┘         │              │         └─────────────┘
                        │ IAM AuthZ    │
                        │ Observability│
                        └──────────────┘
```

### AWS Verified Access
```
Verified Access = Zero Trust access to corporate apps:
- No VPN needed
- Verifies: User identity + Device posture
- Integrates with: Okta, Ping, CrowdStrike, Jamf
- Each application has access policy
- Supports: HTTP/HTTPS applications

Policy example:
"Allow if user in 'engineering' group
 AND device has disk encryption enabled
 AND device OS is patched within 7 days"
```

---

## 9-10. Implementation & Tổng kết {#implementation}

### Zero Trust Implementation Roadmap
```
Phase 1: Foundation (months 1-3)
☐ Inventory all assets and data flows
☐ Strong identity (SSO, MFA for all users)
☐ Device management (inventory, health checks)
☐ Network visibility (flow logs, monitoring)

Phase 2: Segmentation (months 3-6)
☐ Micro-segment critical workloads
☐ Implement Network Policies (K8s)
☐ Encrypt internal traffic (mTLS/service mesh)
☐ Least-privilege access policies

Phase 3: Advanced (months 6-12)
☐ Identity-aware proxy for applications
☐ Continuous risk assessment
☐ Automated response (quarantine compromised devices)
☐ Replace VPN with ZTNA

Phase 4: Optimization (ongoing)
☐ AI/ML-based anomaly detection
☐ Just-in-time privileged access
☐ Continuous compliance verification
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| NIST SP 800-207 | Zero Trust Architecture |
| Google BeyondCorp papers | Original ZT implementation |
| AWS Zero Trust on AWS | AWS-specific guidance |
| CISA Zero Trust Maturity Model | US Government framework |

---
