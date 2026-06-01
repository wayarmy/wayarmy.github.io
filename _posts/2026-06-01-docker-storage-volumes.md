---
layout: post
title: "Docker Storage & Volumes - Lưu trữ trong Docker"
date: 2026-06-01
categories: [containers]
tags: [docker, volumes, bind-mount, overlay2, storage]
---

## Mục lục
1. [Góc nhìn tổng quan - Hệ thống tủ đồ](#goc-nhin-tong-quan)
2. [Named Volumes - Tủ có nhãn](#named-volumes)
3. [Bind Mounts - Gắn trực tiếp thư mục host](#bind-mounts)
4. [tmpfs Mounts - Bộ nhớ tạm](#tmpfs)
5. [overlay2 - Storage driver mặc định](#overlay2)
6. [Layers và Copy-on-Write](#layers-cow)
7. [Volume Plugins - Mở rộng storage](#volume-plugins)
8. [Backup và Restore volumes](#backup-restore)
9. [Performance considerations](#performance)
10. [Tổng kết và best practices](#tong-ket)

---

## 1. Góc nhìn tổng quan - Hệ thống tủ đồ {#goc-nhin-tong-quan}

### Ví dụ đời thường

Docker storage giống **hệ thống tủ đồ** trong ký túc xá:

- **Image layers (read-only)** = đồ nội thất có sẵn trong phòng (giường, bàn) - ai vào ở cũng có, không được thay đổi
- **Container layer (read-write)** = đồ cá nhân bạn mang theo - mất khi bạn chuyển phòng (container bị xóa)
- **Named volumes** = tủ khóa riêng ở tầng trệt - đồ vẫn còn dù bạn đổi phòng
- **Bind mounts** = mang đồ từ nhà vào phòng KTX - folder trên host mount vào container
- **tmpfs** = bảng ghi chú (whiteboard) - nhanh nhưng mất khi tắt

### Tại sao cần hiểu Docker storage?

```
Vấn đề thực tế:
1. Container bị restart → data biến mất?
   → Cần volumes để persist data

2. Database chạy trong Docker → data ở đâu?
   → Named volume hoặc bind mount

3. Docker image 2GB → tốn disk?
   → Hiểu layers và sharing giúp optimize

4. Container ghi file chậm?
   → Hiểu overlay2 CoW để tối ưu
```

---

## 2. Named Volumes - Tủ có nhãn {#named-volumes}

### Named Volumes là gì?

Named volumes là cơ chế storage **do Docker quản lý**. Docker tạo directory trên host, mount vào container, và quản lý lifecycle.

```bash
# Tạo volume
docker volume create my-data

# Sử dụng volume
docker run -v my-data:/app/data nginx
# HOẶC (syntax mới, rõ ràng hơn)
docker run --mount type=volume,source=my-data,target=/app/data nginx

# Volume tồn tại độc lập với container
docker rm container_name    # Container bị xóa
docker volume ls            # Volume vẫn còn!

# Inspect volume
docker volume inspect my-data
# {
#   "Name": "my-data",
#   "Mountpoint": "/var/lib/docker/volumes/my-data/_data",
#   "Driver": "local"
# }
```

### Khi nào dùng Named Volumes?

```
✅ Databases (PostgreSQL, MySQL, MongoDB)
✅ Application data cần persist
✅ Shared data giữa containers
✅ Data cần backup/migrate

Ưu điểm:
- Docker quản lý (portable, không phụ thuộc host path)
- Hoạt động trên mọi OS (Linux, Mac, Windows)
- Dễ backup, migrate
- Có thể dùng volume drivers (NFS, cloud storage)
```

---

## 3. Bind Mounts - Gắn trực tiếp thư mục host {#bind-mounts}

### Bind Mounts là gì?

Bind mount gắn **thư mục/file trên host** trực tiếp vào container. Container đọc/ghi file trực tiếp trên host filesystem.

```bash
# Mount thư mục host vào container
docker run -v /host/path:/container/path nginx
# Hoặc:
docker run --mount type=bind,source=/host/path,target=/container/path nginx

# Read-only bind mount
docker run -v /host/config:/app/config:ro nginx
docker run --mount type=bind,source=/host/config,target=/app/config,readonly nginx
```

### Use cases cho Bind Mounts

```
✅ Development (mount source code → hot reload)
   docker run -v $(pwd)/src:/app/src -v $(pwd)/package.json:/app/package.json node

✅ Config files (mount config vào container)
   docker run -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro nginx

✅ Log output (mount log directory)
   docker run -v /var/log/myapp:/app/logs myapp

❌ KHÔNG dùng cho:
- Production databases (khó migrate, OS-dependent)
- Khi cần portability giữa các environments
```

### Volumes vs Bind Mounts

```
┌──────────────────┬────────────────────┬────────────────────┐
│ Aspect           │ Named Volumes      │ Bind Mounts        │
├──────────────────┼────────────────────┼────────────────────┤
│ Quản lý bởi     │ Docker             │ User/host OS       │
│ Location         │ /var/lib/docker/...│ Anywhere on host   │
│ Populated        │ Yes (from image)   │ No (overwrites)    │
│ Portability      │ High               │ Low (host-dependent)│
│ Docker CLI       │ docker volume ...  │ N/A                │
│ Backup           │ Docker tools       │ Standard file tools│
│ Performance      │ Native             │ Native             │
│ Best for         │ Production data    │ Development        │
└──────────────────┴────────────────────┴────────────────────┘
```

---

## 4. tmpfs Mounts - Bộ nhớ tạm {#tmpfs}

### tmpfs là gì?

tmpfs mount lưu data **trong RAM** của host. Nhanh nhưng mất khi container stop. Không bao giờ ghi xuống disk.

```bash
# Tạo tmpfs mount
docker run --tmpfs /app/tmp nginx
docker run --mount type=tmpfs,target=/app/tmp,tmpfs-size=100m nginx

# Use cases:
# - Sensitive data không muốn persist (secrets, tokens)
# - Temporary files cần performance cao
# - Cache layers
```

---

## 5. overlay2 - Storage driver mặc định {#overlay2}

### overlay2 là gì?

**overlay2** là storage driver mặc định của Docker. Nó sử dụng OverlayFS (kernel filesystem) để merge nhiều layers thành một unified view.

```
Ví dụ: Docker image nginx có 5 layers

Layer 5 (top): nginx config files     ┐
Layer 4: nginx binary                  │ Image layers
Layer 3: apt install nginx             │ (read-only)
Layer 2: apt update                    │
Layer 1 (base): ubuntu:22.04           ┘
                                        
Container layer (read-write)           ← Thin writable layer

OverlayFS merge tất cả → container thấy 1 filesystem thống nhất
```

### Cách overlay2 hoạt động

```bash
# Xem layers của image
docker image inspect nginx | jq '.[0].RootFS.Layers'
# [
#   "sha256:aaa...",   # Layer 1
#   "sha256:bbb...",   # Layer 2
#   "sha256:ccc..."    # Layer 3
# ]

# Xem storage trên disk
docker info | grep "Storage Driver"
# Storage Driver: overlay2

ls /var/lib/docker/overlay2/
# abc123.../  (layer 1)
# def456.../  (layer 2)
# ghi789.../  (container layer)
# l/          (shortened symlinks)

# Mỗi layer chứa:
# diff/    - Nội dung mới/thay đổi của layer này
# merged/  - Union mount (chỉ ở container layer)
# work/    - Working directory cho OverlayFS
# link     - Shortened ID
# lower    - Reference to lower layers
```

---

## 6. Layers và Copy-on-Write {#layers-cow}

### Copy-on-Write (CoW) là gì?

```
Ví dụ đời thường:
Bạn ở trong thư viện. Sách trên kệ = read-only layers.
Khi bạn muốn ghi chú → photocopy trang đó → ghi lên bản copy.
Sách gốc không bị thay đổi!

Trong Docker:
- Container đọc file từ image layers → đọc trực tiếp (nhanh)
- Container VIẾT file từ image layers → copy lên container layer → viết
- File mới → ghi thẳng vào container layer

CoW = chỉ copy khi cần write, tiết kiệm space + share layers giữa containers
```

### Tác động đến performance

```bash
# ❌ CHẬM: Modify file lớn từ image layer lần đầu
# (phải copy TOÀN BỘ file lên container layer trước khi modify)
# Ví dụ: database data file 1GB trong image → edit → copy 1GB!

# ✅ NHANH: Ghi file mới hoặc modify file đã ở container layer
# (ghi trực tiếp, không cần copy)

# Best practice:
# - KHÔNG để data files trong image layers
# - Dùng volumes cho data thay đổi thường xuyên
# - Giữ image layers cho static content (binaries, libraries)
```

### Layer optimization trong Dockerfile

```dockerfile
# ❌ Tạo layer lớn không cần thiết
RUN apt-get update
RUN apt-get install -y nginx
RUN apt-get clean

# ✅ Gộp commands = 1 layer nhỏ hơn
RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Lý do: mỗi RUN = 1 layer. "clean" ở layer riêng 
# KHÔNG giảm size vì layer trước vẫn chứa data!
```

---

## 7. Volume Plugins - Mở rộng storage {#volume-plugins}

### Volume Plugins là gì?

Docker volume plugins cho phép dùng external storage backends (NFS, cloud storage, distributed filesystems) làm volumes.

```bash
# NFS volume
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=nfs-server.local,rw,nfsvers=4 \
  --opt device=:/shared/data \
  nfs-vol

# Rex-Ray (multi-cloud storage)
docker plugin install rexray/ebs   # AWS EBS
docker volume create --driver rexray/ebs --opt size=100 my-ebs-vol

# Portworx, Longhorn, etc. cho Kubernetes
```

---

## 8. Backup và Restore volumes {#backup-restore}

### Backup volume

```bash
# Method 1: Backup to tar (universal)
docker run --rm \
  -v my-data:/source:ro \
  -v $(pwd):/backup \
  alpine tar czf /backup/my-data-backup.tar.gz -C /source .

# Method 2: Copy from volume
docker cp container_name:/path/to/data ./backup/

# Method 3: Volume clone
docker run --rm \
  -v source-vol:/from:ro \
  -v dest-vol:/to \
  alpine sh -c "cp -a /from/. /to/"
```

### Restore volume

```bash
# Restore from tar
docker run --rm \
  -v my-data:/target \
  -v $(pwd):/backup:ro \
  alpine sh -c "rm -rf /target/* && tar xzf /backup/my-data-backup.tar.gz -C /target"
```

---

## 9. Performance considerations {#performance}

### Storage performance tips

```bash
# 1. Volumes > bind mounts > container layer
# Volumes: native filesystem performance
# Container layer: overlay2 overhead (CoW, multiple layers lookup)

# 2. Dùng volumes cho I/O-intensive workloads
docker run -v db-data:/var/lib/postgresql/data postgres

# 3. Tránh ghi nhiều vào container layer
# Log ra stdout thay vì file:
CMD ["app", "--log-stdout"]

# 4. Sử dụng .dockerignore
# Tránh copy không cần thiết vào build context
node_modules/
.git/
*.log

# 5. Monitor disk usage
docker system df
docker system df -v  # Verbose (per image, container, volume)
docker system prune  # Dọn dẹp unused resources
```

---

## 10. Tổng kết và best practices {#tong-ket}

### Decision tree: Chọn loại storage nào?

```
Data cần persist sau container restart?
├─ NO → tmpfs (RAM) hoặc container layer
└─ YES → Data sensitive (secrets)?
    ├─ YES → tmpfs + external secret manager
    └─ NO → Cần share với host?
        ├─ YES → Bind mount (development)
        └─ NO → Named volume (production)
            └─ Multi-host? → Volume plugin (NFS, cloud)
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| Docker Storage Documentation | Hướng dẫn chính thức |
| OverlayFS kernel docs | Kernel implementation |
| Docker Volumes best practices | Production guidelines |

---

*Bài viết tiếp theo: [Dockerfile Best Practices](/2026/08/14/dockerfile-best-practices/) - Viết Dockerfile tối ưu*
