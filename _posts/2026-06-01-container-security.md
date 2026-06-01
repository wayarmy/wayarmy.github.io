---
layout: post
title: "Container Security - Bảo mật Container"
date: 2026-06-01
categories: [containers]
tags: [container-security, trivy, seccomp, rootless, sbom, cosign]
---

## Mục lục
1. [Góc nhìn tổng quan - An ninh nhà máy đóng gói](#goc-nhin-tong-quan)
2. [Image Scanning - Trivy và Grype](#image-scanning)
3. [Rootless Containers - Chạy không cần root](#rootless)
4. [Seccomp profiles - Giới hạn syscalls](#seccomp)
5. [AppArmor cho containers](#apparmor-containers)
6. [Read-only Root Filesystem](#readonly-rootfs)
7. [Secrets Management](#secrets)
8. [Supply Chain Security - Cosign và SBOM](#supply-chain)
9. [Runtime Security và Detection](#runtime-security)
10. [Tổng kết và security checklist](#tong-ket)

---

## 1. Góc nhìn tổng quan - An ninh nhà máy đóng gói {#goc-nhin-tong-quan}

### Ví dụ đời thường

Container security giống **an ninh nhà máy sản xuất thực phẩm**:

- **Image scanning** = kiểm tra nguyên liệu đầu vào (có chất cấm/hết hạn không?)
- **Rootless** = công nhân không có chìa khóa kho tổng (least privilege)
- **Seccomp** = danh sách công cụ được phép dùng (chỉ dao cắt, không dao phay)
- **Read-only rootfs** = dây chuyền cố định, không ai tự ý sửa máy
- **SBOM** = danh sách thành phần đầy đủ (traceability khi có vấn đề)
- **Cosign** = con dấu xác thực sản phẩm thật (chống hàng giả)

### Threat Model cho Containers

```
Attack vectors:
1. Vulnerable image (CVE trong base image/dependencies)
2. Image tampering (supply chain attack)
3. Container escape (breakout to host)
4. Privilege escalation (root in container → root on host)
5. Sensitive data exposure (secrets in image/env)
6. Network attacks (container-to-container)
7. Resource abuse (crypto mining, DoS)
8. Runtime exploitation (RCE → lateral movement)
```

---

## 2. Image Scanning - Trivy và Grype {#image-scanning}

### Trivy - Scanner đa năng

```bash
# Scan image
trivy image nginx:latest
trivy image --severity HIGH,CRITICAL myapp:v1.2.3

# Scan Dockerfile
trivy config Dockerfile

# Scan filesystem (source code)
trivy fs --scanners vuln,secret,misconfig ./

# Output formats
trivy image -f json -o results.json myapp:latest
trivy image -f sarif myapp:latest  # For GitHub Security tab
trivy image -f table myapp:latest  # Human readable

# Ignore unfixed vulnerabilities
trivy image --ignore-unfixed myapp:latest

# Scan trước khi push (CI gate)
trivy image --exit-code 1 --severity CRITICAL myapp:latest
# Exit code 1 = found vulnerabilities → fail CI pipeline
```

### Grype - Anchore scanner

```bash
# Scan image
grype myapp:latest

# Output format
grype -o json myapp:latest > scan.json
grype -o table myapp:latest

# Ignore rules (.grype.yaml)
# ignore:
#   - vulnerability: CVE-2023-XXXX
#     reason: "False positive, not applicable"
```

### Integrate scanning vào workflow

```yaml
# GitHub Actions
name: Security Scan
on: [push]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build
        run: docker build -t app:${{ github.sha }} .
      - name: Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: app:${{ github.sha }}
          exit-code: 1
          severity: HIGH,CRITICAL
```

---

## 3. Rootless Containers - Chạy không cần root {#rootless}

### Rootless Docker

```bash
# Install rootless Docker
dockerd-rootless-setuptool.sh install

# Verify
docker info | grep "Root Dir"
# /home/user/.local/share/docker

# Rootless mode:
# - Docker daemon chạy với user privileges
# - Containers không thể escalate to host root
# - Port < 1024 cần net.ipv4.ip_unprivileged_port_start=0
```

### Podman (rootless by default)

```bash
# Podman chạy rootless mặc định
podman run --rm -it nginx

# Không cần daemon (daemonless)
# Mỗi container là child process của podman command
```

### User namespaces trong Docker

```bash
# Enable userns-remap (root in container ≠ root on host)
# /etc/docker/daemon.json
{
  "userns-remap": "default"
}

# Container root (UID 0) → maps to unprivileged UID on host
# Nếu escape container, chỉ là nobody trên host!
```

---

## 4. Seccomp profiles - Giới hạn syscalls {#seccomp}

### Seccomp là gì?

**seccomp** (Secure Computing) giới hạn system calls mà container được phép thực hiện. Docker có default profile block ~44 dangerous syscalls.

```bash
# Docker default seccomp profile blocks:
# - mount/umount (filesystem manipulation)
# - reboot (host reboot!)
# - ptrace (debugging other processes)
# - add_key/keyctl (kernel keyring)
# - bpf (eBPF programs)
# - clock_settime (change system time)
# - ... 44 total blocked

# Chạy với seccomp profile
docker run --security-opt seccomp=profile.json nginx

# Chạy KHÔNG có seccomp (DANGEROUS - testing only)
docker run --security-opt seccomp=unconfined nginx
```

### Tạo custom seccomp profile

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": ["SCMP_ARCH_X86_64"],
  "syscalls": [
    {
      "names": ["read", "write", "open", "close", "stat",
               "fstat", "mmap", "mprotect", "brk",
               "rt_sigaction", "rt_sigprocmask", "ioctl",
               "access", "pipe", "select", "sched_yield",
               "socket", "connect", "accept", "sendto",
               "recvfrom", "bind", "listen", "clone",
               "execve", "exit", "exit_group", "futex",
               "epoll_wait", "epoll_ctl"],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

---

## 5. AppArmor cho containers {#apparmor-containers}

```bash
# Docker mặc định load AppArmor profile "docker-default"
docker run --security-opt apparmor=docker-default nginx

# Custom AppArmor profile
docker run --security-opt apparmor=my-custom-profile nginx

# Disable AppArmor (testing only!)
docker run --security-opt apparmor=unconfined nginx

# Docker default AppArmor profile blocks:
# - Mount operations
# - Write to /proc, /sys
# - Raw network access
# - Signal sending to other containers
```

---

## 6. Read-only Root Filesystem {#readonly-rootfs}

```bash
# Chạy container với read-only filesystem
docker run --read-only nginx
# Lỗi! nginx cần ghi /var/cache/nginx, /var/run

# Giải pháp: tmpfs cho writable paths
docker run --read-only \
  --tmpfs /var/cache/nginx \
  --tmpfs /var/run \
  --tmpfs /tmp \
  nginx

# Docker Compose:
services:
  web:
    image: nginx
    read_only: true
    tmpfs:
      - /var/cache/nginx
      - /var/run
      - /tmp

# Kubernetes:
securityContext:
  readOnlyRootFilesystem: true
```

---

## 7. Secrets Management {#secrets}

### Không bao giờ đặt secrets trong image!

```dockerfile
# ❌ NEVER DO THIS
ENV DATABASE_URL=postgres://user:password@host/db
COPY .env /app/.env
ARG API_KEY

# ✅ Proper ways:

# 1. BuildKit secrets (build-time)
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) npm ci

# 2. Runtime environment variables
docker run -e DATABASE_URL="..." myapp

# 3. Docker secrets (Swarm)
docker secret create db_pass secret.txt
docker service create --secret db_pass myapp

# 4. External secret managers
# HashiCorp Vault, AWS Secrets Manager, etc.
```

---

## 8. Supply Chain Security - Cosign và SBOM {#supply-chain}

### Image Signing với Cosign

```bash
# Install cosign
brew install cosign  # or go install

# Generate key pair
cosign generate-key-pair

# Sign image
cosign sign --key cosign.key myregistry/myapp:v1.0
# → Signature stored in registry alongside image

# Verify image
cosign verify --key cosign.pub myregistry/myapp:v1.0
# → Confirms image hasn't been tampered with

# Keyless signing (Sigstore/Fulcio - OIDC identity)
cosign sign myregistry/myapp:v1.0
# → Signs with your OIDC identity (Google, GitHub, etc.)
```

### SBOM (Software Bill of Materials)

```bash
# Generate SBOM with Syft
syft myapp:latest -o spdx-json > sbom.spdx.json
syft myapp:latest -o cyclonedx-json > sbom.cdx.json

# Attach SBOM to image
cosign attach sbom --sbom sbom.spdx.json myregistry/myapp:v1.0

# Scan SBOM for vulnerabilities
grype sbom:sbom.spdx.json
trivy sbom sbom.spdx.json

# SBOM = ingredient list for software
# Required by: US Executive Order 14028, EU CRA
# Helps with: incident response, license compliance, supply chain visibility
```

---

## 9. Runtime Security và Detection {#runtime-security}

### Falco - Runtime threat detection

```yaml
# Falco rule example: detect shell in container
- rule: Terminal shell in container
  desc: Detect shell spawned in a container
  condition: >
    spawned_process and container and
    proc.name in (bash, sh, zsh, dash)
  output: >
    Shell spawned in container
    (user=%user.name container=%container.name shell=%proc.name)
  priority: WARNING
  tags: [container, shell]

# Detect sensitive file access
- rule: Read sensitive file
  desc: Detect read of sensitive file in container
  condition: >
    open_read and container and
    (fd.name startswith /etc/shadow or
     fd.name startswith /etc/passwd)
  output: Sensitive file opened (file=%fd.name container=%container.name)
  priority: WARNING
```

---

## 10. Tổng kết và security checklist {#tong-ket}

### Container Security Checklist

```
BUILD TIME:
☐ Use minimal base image (distroless/alpine)
☐ Pin image versions (tag + digest)
☐ Multi-stage build (no build tools in prod)
☐ Non-root USER directive
☐ No secrets in image (no COPY .env)
☐ Scan for vulnerabilities (Trivy/Grype)
☐ Sign images (cosign)
☐ Generate SBOM

RUNTIME:
☐ Read-only root filesystem
☐ Drop all capabilities, add only needed
☐ Seccomp profile enabled
☐ AppArmor/SELinux enforcing
☐ No privileged mode
☐ Resource limits (CPU, memory)
☐ Network policies (restrict communication)
☐ Runtime monitoring (Falco)

ORCHESTRATION:
☐ Pod Security Standards (restricted)
☐ Network Policies
☐ Secret encryption at rest
☐ RBAC least privilege
☐ Image pull policies (Always)
☐ Admission controllers (image scanning)
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| NIST SP 800-190 | Container Security Guide |
| CIS Docker Benchmark | Docker hardening standard |
| Sigstore/Cosign docs | Image signing |
| OWASP Container Top 10 | Container-specific risks |
| Docker Security docs | Official security guide |
| Falco documentation | Runtime security |

---

*Bài viết tiếp theo: [Kubernetes Fundamentals](/2026/08/16/kubernetes-fundamentals/) - Nền tảng Kubernetes*
