---
layout: post
title: "Kubernetes Fundamentals - Nền tảng Kubernetes"
date: 2026-06-01
categories: [containers]
tags: [kubernetes, k8s, pods, deployments, services, ingress]
---

## Mục lục
1. [Góc nhìn tổng quan - Thành phố container](#goc-nhin-tong-quan)
2. [Control Plane - Bộ chỉ huy trung tâm](#control-plane)
3. [Worker Nodes - Nhà máy sản xuất](#worker-nodes)
4. [Pods - Đơn vị triển khai nhỏ nhất](#pods)
5. [ReplicaSets - Đảm bảo số lượng bản sao](#replicasets)
6. [Deployments - Quản lý triển khai](#deployments)
7. [Services - Mạng nội bộ ổn định](#services)
8. [Ingress - Cổng vào từ bên ngoài](#ingress)
9. [ConfigMaps và Secrets - Cấu hình và bí mật](#configmaps-secrets)
10. [Tổng kết và kubectl cheat sheet](#tong-ket)

---

## 1. Góc nhìn tổng quan - Thành phố container {#goc-nhin-tong-quan}

### Ví dụ đời thường

Kubernetes giống **ban quản lý thành phố** cho containers:

- **Control Plane** = Tòa thị chính (ra quyết định, lập kế hoạch)
- **Worker Nodes** = Các khu phố (nơi nhà cửa/containers được xây)
- **Pods** = Căn hộ (có thể chứa 1 hoặc nhiều người/containers)
- **ReplicaSet** = Quy hoạch: "Khu A phải luôn có 3 quán cà phê"
- **Deployment** = Kế hoạch nâng cấp: "Đổi tất cả quán cũ sang quán mới, từng quán một"
- **Service** = Bảng chỉ dẫn cố định: "Quán cà phê gần nhất" (dù quán thay đổi địa chỉ)
- **Ingress** = Cổng thành phố: khách từ bên ngoài đi qua đây để vào

### Tại sao cần Kubernetes?

```
Không có K8s (quản lý thủ công):
- 50 containers trên 10 servers → deploy bằng tay
- Container crash lúc 3AM → ai restart?
- Traffic tăng 10x → scale bằng tay?
- Deploy version mới → downtime?
- Server #3 chết → containers mất?

Với K8s (tự động):
- Khai báo "tôi muốn 5 replicas" → K8s tự sắp xếp
- Container crash → tự restart trong giây
- Traffic tăng → auto-scale thêm pods
- Deploy mới → rolling update, zero downtime
- Server chết → pods tự di chuyển sang server khác
```

### Kiến trúc tổng quan

```
┌──────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                         │
│                                                               │
│  ┌─────────────────────────────────────────┐                 │
│  │           Control Plane                  │                 │
│  │  ┌──────────┐ ┌────────────┐           │                 │
│  │  │kube-api  │ │   etcd     │           │                 │
│  │  │server    │ │ (key-value)│           │                 │
│  │  └──────────┘ └────────────┘           │                 │
│  │  ┌──────────┐ ┌────────────┐           │                 │
│  │  │scheduler │ │ controller │           │                 │
│  │  │          │ │ manager    │           │                 │
│  │  └──────────┘ └────────────┘           │                 │
│  └─────────────────────────────────────────┘                 │
│                                                               │
│  ┌───────────────────┐  ┌───────────────────┐               │
│  │   Worker Node 1    │  │   Worker Node 2    │              │
│  │  ┌─────┐ ┌─────┐  │  │  ┌─────┐ ┌─────┐  │              │
│  │  │Pod A│ │Pod B│  │  │  │Pod C│ │Pod D│  │              │
│  │  └─────┘ └─────┘  │  │  └─────┘ └─────┘  │              │
│  │  ┌───────────────┐ │  │  ┌───────────────┐ │              │
│  │  │    kubelet    │ │  │  │    kubelet    │ │              │
│  │  │  kube-proxy   │ │  │  │  kube-proxy   │ │              │
│  │  │  container    │ │  │  │  container    │ │              │
│  │  │  runtime      │ │  │  │  runtime      │ │              │
│  │  └───────────────┘ │  │  └───────────────┘ │              │
│  └───────────────────┘  └───────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. Control Plane - Bộ chỉ huy trung tâm {#control-plane}

### Components

```
kube-apiserver:
- "Reception desk" - Tất cả giao tiếp đều qua đây
- RESTful API (kubectl, dashboard, CI/CD đều gọi API)
- Authentication, Authorization, Admission Control
- Duy nhất component nói chuyện trực tiếp với etcd

etcd:
- "Bộ nhớ của thành phố" - lưu TẤT CẢ cluster state
- Distributed key-value store (Raft consensus)
- Cluster config, pod specs, secrets, service accounts
- Backup etcd = backup toàn bộ cluster!

kube-scheduler:
- "Nhân viên phân bổ" - quyết định pod chạy ở node nào
- Xem xét: resources available, affinity rules, taints/tolerations
- Scoring system: node nào phù hợp nhất cho pod này?

kube-controller-manager:
- "Đội giám sát" - đảm bảo actual state = desired state
- ReplicaSet controller: "Cần 3 pods, chỉ có 2 → tạo thêm 1"
- Node controller: node mất liên lạc → mark NotReady
- Job controller: quản lý batch jobs
- Nhiều controllers khác (service account, endpoint, etc.)
```

---

## 3. Worker Nodes - Nhà máy sản xuất {#worker-nodes}

### Components trên mỗi worker

```
kubelet:
- "Quản đốc nhà máy" trên mỗi node
- Nhận PodSpec từ API server → đảm bảo containers chạy
- Report node + pod status lên API server
- Pull images, mount volumes, start/stop containers

kube-proxy:
- "Nhân viên mạng" - cài đặt network rules
- Implement Services (ClusterIP, NodePort, LoadBalancer)
- iptables/IPVS rules cho load balancing
- Mỗi Service IP → forward đến actual Pod IPs

Container Runtime:
- "Máy đóng gói" - thực sự chạy containers
- containerd (mặc định), CRI-O
- Implements CRI (Container Runtime Interface)
```

---

## 4. Pods - Đơn vị triển khai nhỏ nhất {#pods}

### Pod là gì?

```
Pod = nhóm 1 hoặc nhiều containers chạy cùng nhau:
- Chia sẻ network namespace (cùng IP, cùng ports)
- Chia sẻ storage volumes
- Cùng được schedule trên 1 node
- Có lifecycle chung (tạo/xóa cùng lúc)

99% trường hợp: 1 Pod = 1 container
Khi nào nhiều containers? → Sidecar pattern (logging, proxy)
```

### Pod manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: web
    version: v1
spec:
  containers:
    - name: app
      image: myapp:1.0.0
      ports:
        - containerPort: 8080
      resources:
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"
          cpu: "500m"
      readinessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 20
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
  restartPolicy: Always
```

### Pod Lifecycle

```
Pending → Running → Succeeded/Failed

Pending:
- Đang được schedule (chưa có node)
- Đang pull image

Running:
- Ít nhất 1 container đang chạy
- readinessProbe → đủ điều kiện nhận traffic

Succeeded:
- Tất cả containers exit 0 (Jobs)

Failed:
- Ít nhất 1 container exit non-zero

CrashLoopBackOff:
- Container crash → restart → crash → restart
- Exponential backoff: 10s, 20s, 40s, ... max 5min
```

---

## 5. ReplicaSets - Đảm bảo số lượng bản sao {#replicasets}

### ReplicaSet là gì?

ReplicaSet đảm bảo luôn có **N pods** chạy. Nếu pod chết → tạo pod mới. Nếu thừa → xóa pod.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3                    # Luôn duy trì 3 pods
  selector:
    matchLabels:
      app: web
  template:                      # Pod template
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: nginx:1.25
```

---

## 6. Deployments - Quản lý triển khai {#deployments}

### Deployment là gì?

Deployment quản lý ReplicaSets và cung cấp **rolling updates**, **rollback**, và **scaling**. Đây là resource bạn dùng 95% thời gian (không dùng ReplicaSet trực tiếp).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # Tạo thêm tối đa 1 pod mới
      maxUnavailable: 0      # Không bao giờ giảm dưới 3 pods
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: web
          image: myapp:v2.0.0
          ports:
            - containerPort: 8080
```

### Rolling Update visualization

```
v1.0 → v2.0 (replicas=3, maxSurge=1, maxUnavailable=0):

Step 0: [v1] [v1] [v1]           (3 v1 running)
Step 1: [v1] [v1] [v1] [v2]     (tạo 1 v2, total 4)
Step 2: [v1] [v1] [v2] [v2]     (v2 ready → xóa 1 v1)
Step 3: [v1] [v2] [v2] [v2]     (thêm 1 v2, xóa 1 v1)
Step 4: [v2] [v2] [v2]           (xong! 3 v2 running)

Zero downtime! Luôn có ≥3 pods serving traffic.
```

### kubectl commands cho Deployment

```bash
# Deploy
kubectl apply -f deployment.yaml

# Scale
kubectl scale deployment web-app --replicas=5

# Update image (trigger rolling update)
kubectl set image deployment/web-app web=myapp:v2.1.0

# Xem rollout status
kubectl rollout status deployment/web-app

# Rollback
kubectl rollout undo deployment/web-app
kubectl rollout undo deployment/web-app --to-revision=3

# History
kubectl rollout history deployment/web-app
```

---

## 7. Services - Mạng nội bộ ổn định {#services}

### Service là gì?

Service cung cấp **stable network endpoint** cho một nhóm Pods. Pods có IP tạm thời (thay đổi khi recreate), nhưng Service IP không đổi.

### Các loại Service

```yaml
# 1. ClusterIP (mặc định) - Internal only
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
    - port: 80            # Service port
      targetPort: 8080    # Container port

# DNS: web-service.default.svc.cluster.local → ClusterIP
# Chỉ truy cập được từ trong cluster

# 2. NodePort - Expose trên port của node
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080     # Port trên mỗi node (30000-32767)
# Truy cập: <any-node-ip>:30080

# 3. LoadBalancer - Cloud load balancer
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
# Cloud provider tạo LB, gán external IP

# 4. ExternalName - DNS alias
spec:
  type: ExternalName
  externalName: database.external.com
# DNS CNAME record
```

---

## 8. Ingress - Cổng vào từ bên ngoài {#ingress}

### Ingress là gì?

Ingress quản lý **HTTP/HTTPS routing** từ bên ngoài vào Services trong cluster. Giống reverse proxy (nginx/HAProxy) nhưng native K8s.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
```

---

## 9. ConfigMaps và Secrets - Cấu hình và bí mật {#configmaps-secrets}

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.default.svc"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
    cache:
      ttl: 300

---
# Sử dụng trong Pod:
spec:
  containers:
    - name: app
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DATABASE_HOST
      volumeMounts:
        - name: config-vol
          mountPath: /etc/app/
  volumes:
    - name: config-vol
      configMap:
        name: app-config
        items:
          - key: config.yaml
            path: config.yaml
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=          # base64 encoded
  password: cGFzc3dvcmQxMjM=  # base64 encoded

# Tạo secret từ command:
# kubectl create secret generic db-secret \
#   --from-literal=username=admin \
#   --from-literal=password=password123
```

---

## 10. Tổng kết và kubectl cheat sheet {#tong-ket}

### kubectl cheat sheet

```bash
# === VIEWING ===
kubectl get pods                    # List pods
kubectl get pods -o wide            # With IP and node
kubectl get all                     # All resources
kubectl describe pod my-pod         # Detailed info
kubectl logs my-pod                 # Container logs
kubectl logs my-pod -f              # Follow logs
kubectl logs my-pod -c sidecar      # Specific container

# === CREATING/EDITING ===
kubectl apply -f manifest.yaml      # Create/update
kubectl delete -f manifest.yaml     # Delete
kubectl edit deployment web-app     # Edit live

# === DEBUGGING ===
kubectl exec -it my-pod -- /bin/sh  # Shell into pod
kubectl port-forward my-pod 8080:80 # Local → pod
kubectl top pods                    # Resource usage
kubectl get events --sort-by=.metadata.creationTimestamp

# === CONTEXT ===
kubectl config get-contexts         # List clusters
kubectl config use-context prod     # Switch cluster
kubectl -n kube-system get pods     # Different namespace
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| kubernetes.io/docs | Documentation chính thức |
| Kubernetes The Hard Way (Kelsey Hightower) | Học K8s từ zero |
| CKAD/CKA Curriculum | Certification study guide |

---

*Bài viết tiếp theo: [Kubernetes Networking](/2026/08/17/kubernetes-networking/) - Mạng Kubernetes*
