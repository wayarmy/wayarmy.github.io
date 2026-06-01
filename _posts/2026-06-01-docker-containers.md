---
layout: post
title: "Containers - Phần 10: Docker & Containers"
subtitle: "Ảo hóa ứng dụng — đóng gói mọi thứ để chạy ở mọi nơi"
gh-repo: wayarmy/wayarmy.github.io
tags: [containers, aws, learning-path]
comments: true
date: 2026-06-01
categories: [containers]
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Phần 10).
>
> **Đối tượng:** Người mới hoàn toàn — biết Linux cơ bản từ Phần 7-9.
>
> **Nguồn tham khảo:**
> - Docker Official Documentation — [https://docs.docker.com/](https://docs.docker.com/)
> - OCI (Open Container Initiative) Runtime Spec — [https://github.com/opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec)
> - OCI Image Spec — [https://github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec)
> - Docker Best Practices — [https://docs.docker.com/develop/develop-images/dockerfile_best-practices/](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
> - AWS Documentation: [ECS](https://docs.aws.amazon.com/ecs/), [Fargate](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html), [ECR](https://docs.aws.amazon.com/ecr/)

---

## 1. Vấn đề: "Trên máy tôi chạy được mà!"

### Ví dụ đời thường:

Bạn nấu được một món ăn tuyệt vời ở nhà. Khi mang công thức đến bếp bạn:
- "Sao ở nhà tôi ngon mà ở đây khác vị?" → Bếp gas khác bếp từ, gia vị khác brand, nồi khác kích thước

Trong lập trình cũng vậy:
- Developer: "Trên **máy tôi** chạy được!" (Works on my machine™)
- DevOps: "Nhưng trên **server** nó crash!"
- Nguyên nhân: Khác version Python, khác thư viện, khác config OS...

**Docker giải quyết vấn đề này:** Đóng gói ứng dụng + MỌI THỨ nó cần (libraries, config, runtime) vào một "container" — chạy ở đâu cũng giống hệt nhau.

---

## 2. Virtual Machine vs Container — Sự khác biệt

### Ví dụ đời thường:

#### Virtual Machine = Ngôi nhà riêng:
Mỗi nhà có **móng riêng, tường riêng, điện nước riêng, nội thất riêng**. Tốn nhiều tài nguyên nhưng hoàn toàn biệt lập.

#### Container = Căn hộ trong chung cư:
Nhiều căn hộ **chia sẻ** móng nhà, cấu trúc tòa nhà, hệ thống điện nước chung. Mỗi căn hộ có **nội thất riêng** (ứng dụng + dependencies) nhưng nhẹ hơn nhiều.

### So sánh kỹ thuật:

```
┌──────────────────────────────────┐    ┌──────────────────────────────────┐
│        Virtual Machines          │    │          Containers              │
├──────────────────────────────────┤    ├──────────────────────────────────┤
│  ┌─────┐ ┌─────┐ ┌─────┐       │    │  ┌─────┐ ┌─────┐ ┌─────┐       │
│  │App A│ │App B│ │App C│       │    │  │App A│ │App B│ │App C│       │
│  ├─────┤ ├─────┤ ├─────┤       │    │  ├─────┤ ├─────┤ ├─────┤       │
│  │Bins │ │Bins │ │Bins │       │    │  │Bins │ │Bins │ │Bins │       │
│  │Libs │ │Libs │ │Libs │       │    │  │Libs │ │Libs │ │Libs │       │
│  ├─────┤ ├─────┤ ├─────┤       │    │  └──┬──┘ └──┬──┘ └──┬──┘       │
│  │Guest│ │Guest│ │Guest│       │    │     └────────┼────────┘          │
│  │ OS  │ │ OS  │ │ OS  │       │    │     ┌────────┴────────┐          │
│  └──┬──┘ └──┬──┘ └──┬──┘       │    │     │ Container Engine│          │
│     └────────┼────────┘          │    │     │   (Docker)      │          │
│     ┌────────┴────────┐          │    │     └────────┬────────┘          │
│     │   Hypervisor    │          │    │     ┌────────┴────────┐          │
│     │ (VMware/KVM)    │          │    │     │   Host OS       │          │
│     └────────┬────────┘          │    │     │ (Linux Kernel)  │          │
│     ┌────────┴────────┐          │    │     └────────┬────────┘          │
│     │    Host OS      │          │    │     ┌────────┴────────┐          │
│     └────────┬────────┘          │    │     │   Hardware      │          │
│     ┌────────┴────────┐          │    │     └─────────────────┘          │
│     │   Hardware      │          │    │                                  │
│     └─────────────────┘          │    │                                  │
└──────────────────────────────────┘    └──────────────────────────────────┘
```

| Đặc điểm | Virtual Machine | Container |
|-----------|----------------|-----------|
| Isolation | Hoàn toàn (mỗi VM có OS riêng) | Process-level (chia sẻ kernel) |
| Boot time | Phút | Giây (thậm chí milliseconds) |
| Kích thước | GB (có OS bên trong) | MB (chỉ app + deps) |
| Resource overhead | Cao (RAM cho mỗi OS) | Thấp |
| Density | 10-20 VMs/host | Hàng trăm containers/host |
| Use case | Multi-tenant, khác OS | Microservices, CI/CD, scaling |

### Container hoạt động nhờ Linux kernel features:

- **Namespaces** (isolation): Mỗi container thấy PID, network, filesystem riêng
- **cgroups** (resource limits): Giới hạn CPU, RAM, I/O cho mỗi container
- **Union filesystem** (layers): Chia sẻ layers chung giữa containers

---

## 3. Docker Architecture — Kiến trúc Docker

### Các thành phần chính (Docker docs: Docker overview):

```
┌─── Docker Client (CLI) ──────────────────────────────────────┐
│  docker build, docker run, docker pull, docker push          │
└──────────────────────────────────┬───────────────────────────┘
                                   │ REST API
┌──────────────────────────────────▼───────────────────────────┐
│                     Docker Daemon (dockerd)                    │
│  - Manages images, containers, networks, volumes             │
│  - Talks to container runtime (containerd → runc)            │
└──────┬─────────────────┬─────────────────┬───────────────────┘
       │                 │                 │
┌──────▼─────┐   ┌──────▼─────┐   ┌──────▼─────┐
│ Container 1│   │ Container 2│   │ Container 3│
│ (nginx)    │   │ (node.js)  │   │ (postgres) │
└────────────┘   └────────────┘   └────────────┘
```

### Thuật ngữ Docker:

| Thuật ngữ | Ví dụ đời thường | Mô tả |
|-----------|-----------------|-------|
| **Image** | Công thức nấu ăn (read-only) | Template để tạo container. Bao gồm OS base + app + deps |
| **Container** | Món ăn (instance chạy từ công thức) | Instance đang chạy của image |
| **Dockerfile** | Bước hướng dẫn nấu | File text mô tả cách build image |
| **Registry** | Tủ sách công thức (Docker Hub, ECR) | Nơi lưu trữ/chia sẻ images |
| **Volume** | Tủ lạnh (dữ liệu persist) | Dữ liệu không mất khi container bị xóa |
| **Network** | Hệ thống ống dẫn | Cho containers giao tiếp với nhau |

---

## 4. Docker Commands — Lệnh cơ bản

### Quản lý Images:

```bash
# Pull image từ Docker Hub
$ docker pull nginx:latest
$ docker pull python:3.11-slim
$ docker pull ubuntu:22.04

# Liệt kê images
$ docker images
REPOSITORY    TAG          IMAGE ID       SIZE
nginx         latest       a8758716bb6a   187MB
python        3.11-slim    12345abcde     150MB
ubuntu        22.04        67890fghij     77MB

# Xóa image
$ docker rmi nginx:latest
$ docker image prune        # Xóa images không dùng
```

### Chạy Containers:

```bash
# Chạy container cơ bản
$ docker run nginx
# → Chạy foreground, Ctrl+C để dừng

# Chạy background (-d = detached)
$ docker run -d --name my-nginx nginx
# → Container chạy ngầm, trả về container ID

# Chạy với port mapping (-p host:container)
$ docker run -d -p 8080:80 --name web nginx
# → Truy cập localhost:8080 → forward đến port 80 trong container

# Chạy với environment variables
$ docker run -d \
    --name my-db \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=myapp \
    -p 5432:5432 \
    postgres:15

# Chạy và vào shell bên trong
$ docker run -it ubuntu:22.04 /bin/bash
root@abc123:/# whoami
root
root@abc123:/# exit

# Chạy với volume (persist data)
$ docker run -d \
    --name my-db \
    -v pgdata:/var/lib/postgresql/data \
    -p 5432:5432 \
    postgres:15
```

### Quản lý Containers:

```bash
# Liệt kê containers đang chạy
$ docker ps
CONTAINER ID   IMAGE   COMMAND                  STATUS         PORTS                  NAMES
abc123def456   nginx   "/docker-entrypoint.…"   Up 2 hours     0.0.0.0:8080->80/tcp   web

# Liệt kê TẤT CẢ (kể cả đã dừng)
$ docker ps -a

# Dừng / Start / Restart
$ docker stop web
$ docker start web
$ docker restart web

# Xem logs
$ docker logs web               # Tất cả logs
$ docker logs -f web            # Follow (realtime)
$ docker logs --tail 50 web     # 50 dòng cuối

# Exec — chạy lệnh TRONG container đang chạy
$ docker exec -it web /bin/bash     # Vào shell
$ docker exec web cat /etc/nginx/nginx.conf   # Chạy lệnh đơn

# Xem resource usage
$ docker stats

# Xóa container
$ docker rm web                 # Phải stop trước
$ docker rm -f web              # Force remove (kể cả đang chạy)
$ docker container prune        # Xóa tất cả stopped containers
```

---

## 5. Dockerfile — Xây dựng Image

### Ví dụ đời thường:

Dockerfile giống **công thức nấu ăn chi tiết**:
1. BẮT ĐẦU VỚI: bột mì loại A (base image)
2. COPY: thêm đường, trứng vào tô (copy source code)
3. RUN: trộn đều 5 phút (install dependencies)
4. EXPOSE: đặt bánh ra đĩa (khai báo port)
5. CMD: phục vụ khách (lệnh chạy khi start)

### Dockerfile cơ bản cho Python app:

```dockerfile
# Dockerfile
# Stage 1: Chọn base image
FROM python:3.11-slim

# Metadata
LABEL maintainer="devops@company.com"
LABEL version="1.0"

# Set working directory
WORKDIR /app

# Copy requirements first (tận dụng Docker cache)
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code
COPY . .

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
    CMD curl -f http://localhost:8000/health || exit 1

# Command to run
CMD ["python", "app.py"]
```

### Dockerfile cho Node.js app:

```dockerfile
# Multi-stage build (production optimized)
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS production
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 appgroup && \
    adduser -S -u 1001 -G appgroup appuser

# Copy only production artifacts
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/package.json ./

# Switch to non-root user
USER appuser

EXPOSE 3000
CMD ["node", "dist/server.js"]
```

### Dockerfile Instructions — Các lệnh chính:

| Instruction | Mục đích | Ví dụ |
|-------------|----------|-------|
| `FROM` | Base image (bắt buộc, dòng đầu tiên) | `FROM ubuntu:22.04` |
| `RUN` | Chạy command lúc build | `RUN apt-get install -y nginx` |
| `COPY` | Copy files từ host vào image | `COPY . /app` |
| `ADD` | Như COPY + hỗ trợ URL, tar extract | `ADD app.tar.gz /app` |
| `WORKDIR` | Set working directory | `WORKDIR /app` |
| `ENV` | Set environment variable | `ENV NODE_ENV=production` |
| `EXPOSE` | Khai báo port (documentation) | `EXPOSE 8080` |
| `CMD` | Default command khi container start | `CMD ["nginx", "-g", "daemon off;"]` |
| `ENTRYPOINT` | Command không bị override | `ENTRYPOINT ["python"]` |
| `USER` | Chạy với user nào | `USER appuser` |
| `VOLUME` | Khai báo mount point | `VOLUME /data` |
| `ARG` | Build-time variable | `ARG VERSION=1.0` |

### Build Image:

```bash
# Build từ Dockerfile trong thư mục hiện tại
$ docker build -t myapp:v1.0 .

# Build với build args
$ docker build -t myapp:v1.0 --build-arg VERSION=1.0 .

# Xem layers
$ docker history myapp:v1.0
```

### Docker Layer Caching — Tối ưu build:

Docker cache mỗi layer (mỗi instruction = 1 layer). Nếu layer không thay đổi → dùng cache → build nhanh hơn.

```dockerfile
# ❌ BAD: Mỗi lần code thay đổi → install lại deps
COPY . /app
RUN pip install -r requirements.txt

# ✅ GOOD: Requirements ít thay đổi → cache được
COPY requirements.txt /app/
RUN pip install -r requirements.txt
COPY . /app                              # Chỉ layer này rebuild
```

---

## 6. Docker Compose — Chạy nhiều containers

### Ví dụ đời thường:

Docker Compose giống **hướng dẫn dàn nhạc** — bạn chỉ huy tất cả nhạc cụ (containers) chơi cùng nhau.

Thay vì chạy từng lệnh `docker run` riêng cho web server, database, cache... bạn viết 1 file YAML mô tả TẤT CẢ, rồi chạy 1 lệnh.

### docker-compose.yml:

```yaml
# docker-compose.yml
version: "3.9"

services:
  # Web application
  web:
    build: .                          # Build từ Dockerfile trong thư mục hiện tại
    ports:
      - "8080:8000"                   # host:container
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/myapp
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    restart: unless-stopped
    volumes:
      - ./src:/app/src                # Mount source code (dev mode)

  # Database
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data    # Named volume (persist)
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

  # Cache
  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  # Reverse proxy
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web

volumes:
  pgdata:                             # Docker managed volume
```

### Docker Compose commands:

```bash
# Start tất cả services (background)
$ docker compose up -d

# Xem status
$ docker compose ps

# Xem logs
$ docker compose logs -f web         # Follow logs của service "web"

# Stop tất cả
$ docker compose down

# Stop + xóa volumes (CẢNH BÁO: mất data!)
$ docker compose down -v

# Rebuild (sau khi sửa Dockerfile)
$ docker compose up -d --build

# Scale service
$ docker compose up -d --scale web=3  # Chạy 3 instances của web

# Exec vào service
$ docker compose exec web bash
```

---

## 7. Docker Networking

### Network types:

| Driver | Mô tả | Use case |
|--------|--------|----------|
| **bridge** (default) | Network riêng cho containers trên cùng host | Development, single host |
| **host** | Container dùng network của host | Performance-critical |
| **none** | Không networking | Security isolation |
| **overlay** | Network span nhiều hosts | Docker Swarm, multi-host |

### Containers giao tiếp với nhau:

```bash
# Trong docker-compose: dùng service name làm hostname
# web → db: connect to "db:5432" (Docker DNS tự resolve)

# Manual network:
$ docker network create mynet
$ docker run -d --name db --network mynet postgres:15
$ docker run -d --name web --network mynet myapp

# Trong container "web":
# ping db         → OK (Docker DNS resolve "db" → IP của container db)
# curl db:5432    → Kết nối đến postgres
```

---

## 8. Docker Best Practices

### Security:

```dockerfile
# 1. KHÔNG chạy bằng root
RUN adduser --disabled-password --gecos '' appuser
USER appuser

# 2. Dùng image nhỏ nhất có thể
FROM python:3.11-slim      # ✅ Slim variant (150MB)
# FROM python:3.11         # ❌ Full image (900MB+)
# FROM python:3.11-alpine  # ⚠️ Nhỏ nhất nhưng có thể thiếu libs

# 3. Không copy secrets vào image
# ❌ COPY .env /app/.env
# ✅ Dùng environment variables lúc runtime

# 4. Scan vulnerabilities
# docker scout cve myimage:tag
```

### Performance:

```dockerfile
# 1. Minimize layers (gộp RUN)
# ❌ Multiple layers
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y vim
RUN apt-get clean

# ✅ Single layer
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl vim && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 2. .dockerignore (giống .gitignore)
# Không copy files không cần thiết vào image
```

`.dockerignore`:
```
node_modules/
.git/
.env
*.log
__pycache__/
.venv/
```

### Multi-stage builds (giảm image size):

```dockerfile
# Build stage (có build tools, lớn)
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

# Production stage (chỉ binary, nhỏ)
FROM alpine:3.18
COPY --from=builder /app/server /usr/local/bin/
CMD ["server"]
# Final image: ~20MB thay vì ~1GB!
```

---

## 9. AWS Mapping — ECS, Fargate, ECR

### Amazon ECR (Elastic Container Registry):

**ECR** = Docker Hub private của AWS — nơi lưu trữ Docker images.

```bash
# Login ECR
$ aws ecr get-login-password --region ap-southeast-1 | \
    docker login --username AWS --password-stdin 123456789.dkr.ecr.ap-southeast-1.amazonaws.com

# Tag image
$ docker tag myapp:v1.0 123456789.dkr.ecr.ap-southeast-1.amazonaws.com/myapp:v1.0

# Push image
$ docker push 123456789.dkr.ecr.ap-southeast-1.amazonaws.com/myapp:v1.0
```

### Amazon ECS (Elastic Container Service):

**ECS** = Docker orchestration của AWS — chạy và quản lý containers at scale.

Các khái niệm ECS:

| Khái niệm | Docker equivalent | Mô tả |
|-----------|-------------------|--------|
| **Task Definition** | docker-compose.yml | Mô tả containers cần chạy (image, ports, resources) |
| **Task** | docker run | Instance đang chạy của Task Definition |
| **Service** | docker compose up --scale | Duy trì N tasks luôn chạy + load balancing |
| **Cluster** | Docker host | Nhóm EC2 instances hoặc Fargate capacity |

### ECS Launch Types:

```
┌─────────────────── ECS ──────────────────────────┐
│                                                   │
│  ┌─── EC2 Launch Type ──┐  ┌─── Fargate ──────┐ │
│  │                       │  │                   │ │
│  │  Bạn quản lý EC2     │  │  AWS quản lý     │ │
│  │  instances            │  │  infrastructure  │ │
│  │                       │  │                   │ │
│  │  ✓ Control OS        │  │  ✓ Serverless    │ │
│  │  ✓ GPU support       │  │  ✓ Không quản lý │ │
│  │  ✗ Phải patch, scale │  │      EC2         │ │
│  │                       │  │  ✓ Pay per task  │ │
│  └───────────────────────┘  └───────────────────┘ │
│                                                   │
└───────────────────────────────────────────────────┘
```

### Fargate — Serverless Containers:

**Fargate** = "Chạy container mà không cần quản lý server". Bạn chỉ định:
- Image nào
- Bao nhiêu CPU/RAM
- Networking config

AWS lo phần còn lại (provision server, patch, scale...).

**Ví dụ Task Definition (JSON):**

```json
{
  "family": "my-web-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "web",
      "image": "123456789.dkr.ecr.ap-southeast-1.amazonaws.com/myapp:v1.0",
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "NODE_ENV", "value": "production"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-web-app",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### Kiến trúc ECS đầy đủ:

```
Internet
    │
    ▼
┌─────────┐
│   ALB   │ ← Application Load Balancer
└────┬────┘
     │
┌────▼──── ECS Cluster ─────────────────────────────┐
│                                                     │
│  ┌─── Service: web-service (desired: 3) ────────┐ │
│  │                                               │ │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐     │ │
│  │  │ Task 1  │  │ Task 2  │  │ Task 3  │     │ │
│  │  │(Fargate)│  │(Fargate)│  │(Fargate)│     │ │
│  │  └─────────┘  └─────────┘  └─────────┘     │ │
│  │                                               │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ┌─── Service: worker-service (desired: 2) ─────┐ │
│  │                                               │ │
│  │  ┌─────────┐  ┌─────────┐                   │ │
│  │  │ Task 1  │  │ Task 2  │                   │ │
│  │  └─────────┘  └─────────┘                   │ │
│  │                                               │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
└─────────────────────────────────────────────────────┘
     │
     ▼
┌─────────┐
│   RDS   │ ← Database
└─────────┘
```

### ECS vs EKS vs App Runner:

| Service | Khi nào dùng | Complexity |
|---------|-------------|-----------|
| **ECS + Fargate** | Container workloads, đơn giản | Medium |
| **EKS (Kubernetes)** | Cần Kubernetes ecosystem, multi-cloud | High |
| **App Runner** | Web apps đơn giản, auto-scale from source | Low |

---

## 10. Thực hành: Lab tự làm

### Lab 1: Docker cơ bản

```bash
# 1. Chạy nginx container
docker run -d -p 8080:80 --name myweb nginx
curl http://localhost:8080

# 2. Vào container, sửa trang web
docker exec -it myweb bash
echo "<h1>Hello Docker!</h1>" > /usr/share/nginx/html/index.html
exit
curl http://localhost:8080   # Thấy "Hello Docker!"

# 3. Cleanup
docker stop myweb && docker rm myweb
```

### Lab 2: Build custom image

```bash
# 1. Tạo app đơn giản
mkdir ~/myapp && cd ~/myapp

cat > app.py <<'EOF'
from http.server import HTTPServer, SimpleHTTPRequestHandler
import os

class Handler(SimpleHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        hostname = os.environ.get('HOSTNAME', 'unknown')
        self.wfile.write(f"<h1>Hello from {hostname}!</h1>".encode())

HTTPServer(('0.0.0.0', 8000), Handler).serve_forever()
EOF

cat > Dockerfile <<'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
EOF

# 2. Build & run
docker build -t myapp:v1 .
docker run -d -p 8000:8000 --name myapp myapp:v1
curl http://localhost:8000
```

### Lab 3: Docker Compose full stack

```bash
mkdir ~/fullstack && cd ~/fullstack
# Tạo docker-compose.yml (như ví dụ Section 6)
docker compose up -d
docker compose ps
docker compose logs -f
# Truy cập http://localhost:8080
docker compose down
```

### Lab 4: Push to ECR & Deploy ECS

```bash
# 1. Tạo ECR repository (AWS Console hoặc CLI)
aws ecr create-repository --repository-name myapp

# 2. Login, tag, push
aws ecr get-login-password | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker tag myapp:v1 <account-id>.dkr.ecr.<region>.amazonaws.com/myapp:v1
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/myapp:v1

# 3. Tạo ECS Cluster, Task Definition, Service (via Console)
# 4. Attach ALB → verify via ALB DNS name
```

---

## 11. Tổng kết

| Khái niệm | Ví dụ đời thường | Kỹ thuật |
|-----------|-----------------|----------|
| Container | Căn hộ chung cư (chia sẻ hạ tầng) | Process isolation via namespaces/cgroups |
| Image | Công thức nấu ăn (bất biến) | Read-only template, layered filesystem |
| Dockerfile | Bước hướng dẫn nấu | Instructions to build image |
| Docker Compose | Hướng dẫn dàn nhạc | Multi-container orchestration |
| ECR | Tủ sách công thức (riêng tư) | Private Docker registry |
| ECS | Quản lý nhà bếp quy mô lớn | Container orchestration |
| Fargate | Bếp thuê (không cần mua) | Serverless container compute |

---

## Tài liệu tham khảo

1. **Docker Official Documentation** — [https://docs.docker.com/](https://docs.docker.com/)
2. **Dockerfile reference** — [https://docs.docker.com/engine/reference/builder/](https://docs.docker.com/engine/reference/builder/)
3. **Docker Compose reference** — [https://docs.docker.com/compose/compose-file/](https://docs.docker.com/compose/compose-file/)
4. **OCI Runtime Specification** — [https://github.com/opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec)
5. **OCI Image Specification** — [https://github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec)
6. **AWS ECS Developer Guide** — [https://docs.aws.amazon.com/ecs/](https://docs.aws.amazon.com/ecs/)
7. **AWS Fargate** — [https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html](https://docs.aws.amazon.com/AmazonECS/latest/userguide/what-is-fargate.html)
8. **AWS ECR** — [https://docs.aws.amazon.com/ecr/](https://docs.aws.amazon.com/ecr/)

---

**Bài tiếp theo:** [Phần 11: Databases & Data Architecture — Hiểu về cơ sở dữ liệu](/2026-06-01-databases-data-architecture/)
