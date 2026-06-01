---
layout: post
title: "Dockerfile Best Practices - Viết Dockerfile tối ưu"
date: 2026-06-01
categories: [containers]
tags: [dockerfile, multi-stage, optimization, security, docker]
---

## Mục lục
1. [Góc nhìn tổng quan - Công thức nấu ăn hoàn hảo](#goc-nhin-tong-quan)
2. [Multi-stage builds - Nấu xong dọn bếp](#multi-stage)
3. [Layer optimization - Giảm số lớp bánh](#layer-optimization)
4. [.dockerignore - Không mang cả nhà vào bếp](#dockerignore)
5. [Non-root user - Không cần quyền admin để nấu](#non-root)
6. [HEALTHCHECK - Kiểm tra sức khỏe container](#healthcheck)
7. [Distroless images - Bếp minimalist](#distroless)
8. [Security scanning - Kiểm tra nguyên liệu](#security-scanning)
9. [Caching strategies - Tận dụng layer cache](#caching)
10. [Tổng kết và Dockerfile templates](#tong-ket)

---

## 1. Góc nhìn tổng quan - Công thức nấu ăn hoàn hảo {#goc-nhin-tong-quan}

### Ví dụ đời thường

Viết Dockerfile giống **viết công thức nấu ăn** cho nhà hàng:

- **Multi-stage** = bạn có bếp chuẩn bị (cắt rau, ướp thịt) và bếp phục vụ (chỉ có món đã xong) - khách không thấy đống vỏ rau
- **Layer optimization** = sắp xếp bước nấu hợp lý: nguyên liệu ít thay đổi (gia vị) ở trước, nguyên liệu thay đổi thường (rau tươi) ở sau
- **.dockerignore** = không mang cả tủ lạnh vào bếp, chỉ lấy nguyên liệu cần
- **Non-root** = đầu bếp không cần quyền quản lý nhà hàng để nấu ăn
- **HEALTHCHECK** = thử nếm món mỗi 30 giây xem còn ngon không
- **Distroless** = bếp chỉ có đúng dụng cụ cần (không có TV, sofa trong bếp)

### Vấn đề với Dockerfile "naive"

```dockerfile
# ❌ Dockerfile xấu - đừng làm thế này!
FROM ubuntu:latest            # "latest" không reproducible
RUN apt-get update           # Layer riêng, cache stale
RUN apt-get install -y python3 python3-pip gcc make libffi-dev
RUN pip install -r requirements.txt
COPY . /app                  # Copy TOÀN BỘ (kể cả .git, node_modules)
RUN pip install .
CMD python3 app.py           # Chạy bằng root!

# Vấn đề:
# 1. Image 1.5GB (Ubuntu full + build tools)
# 2. Chạy bằng root (security risk)
# 3. "latest" tag → build khác nhau mỗi lần
# 4. Không có health check
# 5. Cache invalidation kém (COPY . trước install deps)
# 6. Build tools trong production image
```

---

## 2. Multi-stage builds - Nấu xong dọn bếp {#multi-stage}

### Multi-stage builds là gì?

Multi-stage cho phép dùng nhiều FROM trong 1 Dockerfile. Stage đầu build/compile, stage cuối chỉ copy kết quả. Production image không chứa build tools.

```dockerfile
# === Stage 1: BUILD ===
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -ldflags="-s -w" -o /app/server ./cmd/server

# === Stage 2: PRODUCTION ===
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/server /server
USER nonroot:nonroot
EXPOSE 8080
ENTRYPOINT ["/server"]

# Kết quả:
# - Build stage: ~800MB (Go SDK + source + deps)
# - Production image: ~5MB (static binary + distroless)
```

### Multi-stage cho Node.js

```dockerfile
# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --production

# Stage 2: Build (nếu cần TypeScript/bundling)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 3: Production
FROM node:20-alpine AS production
WORKDIR /app
ENV NODE_ENV=production

# Copy production deps only
COPY --from=deps /app/node_modules ./node_modules
# Copy built application
COPY --from=builder /app/dist ./dist
COPY package.json ./

# Non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup
USER appuser

EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
CMD ["node", "dist/index.js"]
```

### Multi-stage cho Python

```dockerfile
# Stage 1: Build wheels
FROM python:3.12-slim AS builder
RUN pip install --no-cache-dir build wheel
WORKDIR /app
COPY requirements.txt .
RUN pip wheel --no-cache-dir --wheel-dir /wheels -r requirements.txt

# Stage 2: Production
FROM python:3.12-slim
WORKDIR /app

# Install pre-built wheels (no compile needed)
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir --no-index --find-links=/wheels /wheels/* && \
    rm -rf /wheels

COPY . .
RUN useradd -r -s /bin/false appuser
USER appuser

EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "--workers", "4", "app:create_app()"]
```

---

## 3. Layer optimization - Giảm số lớp bánh {#layer-optimization}

### Nguyên tắc ordering

```dockerfile
# Nguyên tắc: ÍT thay đổi → TRƯỚC, NHIỀU thay đổi → SAU
# (maximize cache hits)

# ✅ Correct order:
FROM node:20-alpine
WORKDIR /app

# 1. System deps (thay đổi rất ít) ← cache hit
RUN apk add --no-cache curl

# 2. Package files (thay đổi khi thêm dep) ← cache hit thường
COPY package.json package-lock.json ./
RUN npm ci

# 3. Source code (thay đổi mỗi commit) ← cache miss thường
COPY . .
RUN npm run build

# Khi sửa code: chỉ step 3 chạy lại, step 1-2 cached!
# Tiết kiệm 2-5 phút build time mỗi lần
```

### Gộp RUN commands

```dockerfile
# ❌ Mỗi RUN = 1 layer (7 layers!)
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y nginx
RUN rm -rf /var/lib/apt/lists/*

# ✅ Gộp lại = 1 layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      curl \
      nginx && \
    rm -rf /var/lib/apt/lists/*

# QUAN TRỌNG: rm ở CÙNG RUN với install
# Nếu rm ở RUN riêng → layer trước VẪN chứa apt lists!
```

---

## 4. .dockerignore - Không mang cả nhà vào bếp {#dockerignore}

### .dockerignore file

```bash
# .dockerignore
.git
.gitignore
.env
.env.*
node_modules
dist
build
*.log
*.md
!README.md
docker-compose*.yml
Dockerfile*
.dockerignore
__pycache__
*.pyc
.pytest_cache
.vscode
.idea
coverage
.nyc_output
*.swp
*.swo
```

### Tại sao quan trọng?

```
Không có .dockerignore:
  COPY . .  → gửi .git (500MB) + node_modules (300MB) cho Docker daemon
  Build context: 800MB → chậm, tốn network

Với .dockerignore:
  COPY . .  → chỉ gửi source code (10MB)
  Build context: 10MB → nhanh!
```

---

## 5. Non-root user - Không cần quyền admin để nấu {#non-root}

### Tại sao không chạy root?

```
Container chạy root = process trong container là root.
Nếu container bị exploit → attacker có root trong container.
Với volume mounts, root có thể escape ra host!

Nguyên tắc least privilege: chạy với quyền tối thiểu cần.
```

### Cách implement

```dockerfile
# Method 1: Tạo user trong Dockerfile
RUN groupadd -r appgroup && \
    useradd -r -g appgroup -d /app -s /sbin/nologin appuser
# Set ownership
RUN chown -R appuser:appgroup /app
# Switch to non-root
USER appuser

# Method 2: Alpine
RUN addgroup -g 1001 -S app && \
    adduser -S -u 1001 -G app app
USER app

# Method 3: Dùng numeric UID (portable)
USER 1001:1001

# Method 4: Distroless (nonroot variant)
FROM gcr.io/distroless/java17:nonroot
# Đã có user nonroot (UID 65534)
```

---

## 6. HEALTHCHECK - Kiểm tra sức khỏe container {#healthcheck}

### HEALTHCHECK instruction

```dockerfile
# Syntax
HEALTHCHECK [OPTIONS] CMD command

# Options:
# --interval=30s    : Kiểm tra mỗi 30s
# --timeout=10s     : Timeout mỗi lần check
# --start-period=5s : Grace period khi start
# --retries=3       : Số lần fail trước khi unhealthy

# HTTP health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# TCP check (khi không có curl)
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/ || exit 1

# Database check
HEALTHCHECK --interval=30s --timeout=10s --retries=5 \
  CMD pg_isready -U postgres || exit 1
```

### Container health states

```
starting  → (start-period) → healthy ← → unhealthy
                                          ↓ (after retries)
                                     [restart if restart policy]

docker ps:
CONTAINER ID  STATUS
abc123        Up 5 min (healthy)
def456        Up 2 min (health: starting)
ghi789        Up 10 min (unhealthy)
```

---

## 7. Distroless images - Bếp minimalist {#distroless}

### Distroless là gì?

**Distroless** images (Google) chỉ chứa application runtime - không shell, không package manager, không OS utilities. Giảm attack surface đến minimum.

```
Image size comparison:
┌────────────────────────┬────────────┬──────────┐
│ Base Image             │ Size       │ CVEs     │
├────────────────────────┼────────────┼──────────┤
│ ubuntu:22.04           │ 77MB       │ 50+      │
│ python:3.12            │ 1GB        │ 200+     │
│ python:3.12-slim       │ 150MB      │ 30+      │
│ python:3.12-alpine     │ 50MB       │ 5+       │
│ distroless/python3     │ 50MB       │ 0-5      │
│ distroless/static      │ 2MB        │ 0        │
│ scratch                │ 0MB        │ 0        │
└────────────────────────┴────────────┴──────────┘
```

### Sử dụng distroless

```dockerfile
# Go static binary
FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app /app
USER nonroot
ENTRYPOINT ["/app"]

# Java
FROM gcr.io/distroless/java17:nonroot
COPY --from=builder /app/target/app.jar /app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]

# Node.js
FROM gcr.io/distroless/nodejs20
COPY --from=builder /app /app
WORKDIR /app
CMD ["dist/index.js"]

# Note: Không có shell! Không thể exec vào debug
# Dùng :debug tag nếu cần troubleshoot
FROM gcr.io/distroless/static:debug  # Có busybox shell
```

---

## 8. Security scanning - Kiểm tra nguyên liệu {#security-scanning}

### Image scanning tools

```bash
# Trivy (Aqua Security) - Most popular
trivy image myapp:latest
trivy image --severity HIGH,CRITICAL myapp:latest

# Grype (Anchore)
grype myapp:latest

# Docker Scout (built-in)
docker scout cves myapp:latest
docker scout recommendations myapp:latest

# Snyk
snyk container test myapp:latest
```

### Integrate vào CI/CD

```yaml
# GitHub Actions example
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Scan with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: myapp:${{ github.sha }}
    exit-code: 1              # Fail build if vulns found
    severity: HIGH,CRITICAL
    format: sarif
    output: trivy-results.sarif
```

### Best practices for security

```dockerfile
# 1. Pin base image digest (not just tag)
FROM python:3.12-slim@sha256:abc123...

# 2. Don't store secrets in image
# ❌ COPY .env /app/.env
# ❌ ARG DB_PASSWORD
# ✅ Use runtime env vars or secret managers

# 3. Scan during build (BuildKit)
# syntax=docker/dockerfile:1
FROM python:3.12-slim
RUN --mount=type=secret,id=pip_conf,target=/etc/pip.conf \
    pip install -r requirements.txt

# 4. Read-only root filesystem at runtime
# docker run --read-only --tmpfs /tmp myapp
```

---

## 9. Caching strategies - Tận dụng layer cache {#caching}

### BuildKit cache mounts

```dockerfile
# syntax=docker/dockerfile:1

# Cache apt packages across builds
RUN --mount=type=cache,target=/var/cache/apt \
    --mount=type=cache,target=/var/lib/apt \
    apt-get update && apt-get install -y curl

# Cache pip downloads
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Cache Go modules
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /app ./...

# Cache npm
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

---

## 10. Tổng kết và Dockerfile templates {#tong-ket}

### Production Dockerfile template (Go)

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.22-alpine AS builder
RUN apk add --no-cache git ca-certificates
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags="-s -w -X main.version=$(git describe --tags)" \
    -o /bin/app ./cmd/app

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /bin/app /app
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER nonroot:nonroot
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD ["/app", "healthcheck"]
ENTRYPOINT ["/app"]
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| Dockerfile reference (docs.docker.com) | Specification chính thức |
| Docker BuildKit docs | Advanced build features |
| Distroless GitHub repo | Google distroless images |
| Trivy documentation | Vulnerability scanning |
| Hadolint | Dockerfile linter |

---

*Bài viết tiếp theo: [Container Security](/2026/08/15/container-security/) - Bảo mật container*
