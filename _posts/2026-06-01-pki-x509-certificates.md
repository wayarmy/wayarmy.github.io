---
layout: post
title: "PKI & X.509 Certificates - Hạ tầng khóa công khai"
date: 2026-06-01
categories: [security]
tags: [pki, x509, certificates, ca, acme, lets-encrypt]
---

## Mục lục
1. [Góc nhìn tổng quan - Hệ thống công chứng số](#overview)
2. [CA Hierarchy - Hệ thống phân cấp](#ca-hierarchy)
3. [Certificate Types: DV, OV, EV](#cert-types)
4. [SAN và Wildcard Certificates](#san-wildcard)
5. [CSR - Certificate Signing Request](#csr)
6. [Let\'s Encrypt và ACME Protocol](#lets-encrypt)
7. [Certificate Transparency (CT) Logs](#ct-logs)
8. [AWS Certificate Manager (ACM)](#acm)
9. [Certificate lifecycle management](#lifecycle)
10. [Tổng kết](#tong-ket)

---

## 1. Góc nhìn tổng quan {#overview}

### Ví dụ đời thường

PKI giống **hệ thống công chứng quốc gia**:
- **Root CA** = Bộ Tư pháp (trust gốc, cấp phép cho công chứng viên)
- **Intermediate CA** = Phòng công chứng (ký xác nhận giấy tờ)
- **Certificate** = Giấy công chứng (xác nhận "đây đúng là website X")
- **CSR** = Đơn xin cấp chứng nhận (gửi lên CA)
- **Let\'s Encrypt** = Dịch vụ công chứng miễn phí, tự động
- **CT Logs** = Sổ đăng ký công khai (ai cũng xem được cert nào đã cấp)

---

## 2. CA Hierarchy {#ca-hierarchy}

```
Root CA (offline, vault, air-gapped)
├── Intermediate CA 1 (online, signs server certs)
│   ├── www.example.com certificate
│   ├── api.example.com certificate
│   └── *.internal.example.com wildcard
└── Intermediate CA 2 (another subordinate)
    └── client certificates

Root CA:
- Self-signed (trust anchor)
- Pre-installed in browsers/OS trust stores
- NEVER online (offline signing)
- Valid 20-30 years
- ~150 Root CAs trusted globally

Intermediate CA:
- Signed by Root CA
- Used for day-to-day signing
- If compromised: revoke intermediate, Root still safe
- Valid 5-10 years
```

---

## 3. Certificate Types {#cert-types}

```
DV (Domain Validation):
- Proves: You control the domain
- Verification: DNS record or HTTP challenge
- Time: Minutes (automated)
- Cost: Free (Let's Encrypt) to $10/year
- Indicator: Padlock icon, no company name
- Use: Personal sites, APIs, internal services

OV (Organization Validation):
- Proves: Domain ownership + Organization exists
- Verification: Business documents review
- Time: 1-3 days
- Cost: $50-200/year
- Indicator: Company name in cert details
- Use: Business websites, B2B

EV (Extended Validation):
- Proves: Thorough organization vetting
- Verification: Legal, operational, physical checks
- Time: 1-2 weeks
- Cost: $200-1000/year
- Indicator: Company name (was green bar, now just in details)
- Use: Banks, e-commerce, high-trust sites
```

---

## 4-5. SAN, Wildcard, CSR {#san-wildcard}

```bash
# SAN (Subject Alternative Name): Multiple domains in 1 cert
# CN: www.example.com
# SAN: example.com, api.example.com, admin.example.com

# Wildcard: *.example.com
# Covers: www.example.com, api.example.com, anything.example.com
# Does NOT cover: sub.sub.example.com (only 1 level)

# Generate CSR with SAN
openssl req -new -newkey rsa:2048 -nodes \
  -keyout server.key -out server.csr \
  -subj "/CN=example.com/O=My Company" \
  -addext "subjectAltName=DNS:example.com,DNS:www.example.com,DNS:api.example.com"

# View CSR
openssl req -in server.csr -text -noout
```

---

## 6. Let\'s Encrypt và ACME {#lets-encrypt}

```bash
# ACME (Automatic Certificate Management Environment) - RFC 8555
# Protocol for automated certificate issuance

# Challenge types:
# HTTP-01: Place file at /.well-known/acme-challenge/TOKEN
# DNS-01: Create TXT record _acme-challenge.domain.com
# TLS-ALPN-01: TLS negotiation on port 443

# Certbot (most popular ACME client)
certbot certonly --nginx -d example.com -d www.example.com

# Auto-renewal (cron/systemd timer)
certbot renew --quiet
# Certs valid 90 days, renew at 60 days

# DNS challenge (wildcard certs, no web server needed)
certbot certonly --dns-cloudflare -d "*.example.com" -d example.com
```

---

## 7-10. CT Logs, ACM, Lifecycle {#ct-logs}

### Certificate Transparency
```
CT = Public log of ALL issued certificates
- Anyone can monitor for unauthorized certs for their domain
- Browsers require CT proof for EV certs
- Tools: crt.sh, Facebook CT Monitor

# Search certificates for your domain
curl "https://crt.sh/?q=%.example.com&output=json" | jq .
```

### AWS ACM
```
AWS Certificate Manager:
- Free public SSL/TLS certificates for AWS services
- Auto-renewal (no manual management!)
- Works with: ALB, CloudFront, API Gateway
- Cannot export private key (AWS-managed)
- DNS validation (recommended) or Email validation

# Terraform
resource "aws_acm_certificate" "cert" {
  domain_name       = "example.com"
  validation_method = "DNS"
  subject_alternative_names = ["*.example.com"]
  lifecycle { create_before_destroy = true }
}
```

---
