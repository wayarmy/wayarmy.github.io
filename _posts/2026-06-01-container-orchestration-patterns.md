---
layout: post
title: "Container Orchestration Patterns - Các mẫu thiết kế K8s"
date: 2026-06-01
categories: [containers]
tags: [kubernetes, sidecar, init-container, statefulset, hpa, patterns]
---

## Mục lục
1. [Góc nhìn tổng quan - Mẫu thiết kế nhà máy](#goc-nhin-tong-quan)
2. [Sidecar Pattern - Trợ lý đi kèm](#sidecar)
3. [Ambassador Pattern - Đại sứ giao tiếp](#ambassador)
4. [Adapter Pattern - Bộ chuyển đổi](#adapter)
5. [Init Containers - Chuẩn bị trước khi khởi động](#init-containers)
6. [DaemonSet - Mỗi node một agent](#daemonset)
7. [StatefulSet - Ứng dụng có trạng thái](#statefulset)
8. [Jobs và CronJobs - Tác vụ batch](#jobs-cronjobs)
9. [HPA - Tự động co giãn](#hpa)
10. [Tổng kết và khi nào dùng pattern nào](#tong-ket)

---

## 1. Góc nhìn tổng quan - Mẫu thiết kế nhà máy {#goc-nhin-tong-quan}

### Ví dụ đời thường

Các orchestration patterns giống **cách tổ chức nhân sự** trong nhà máy:

- **Sidecar** = trợ lý cá nhân đi theo nhân viên chính (ví dụ: thư ký ghi chép)
- **Ambassador** = phiên dịch viên (nhân viên nói tiếng Việt, ambassador dịch ra tiếng Anh cho đối tác)
- **Adapter** = bộ phận QC chuyển đổi sản phẩm về format chuẩn
- **Init Container** = nhân viên setup dọn dẹp phòng trước khi nhân viên chính vào làm
- **DaemonSet** = bảo vệ - mỗi tầng (node) phải có đúng 1 người
- **StatefulSet** = nhân viên có bàn cố định, tên trên bảng (identity quan trọng)
- **Job** = lao công dọn cuối ngày (chạy 1 lần rồi thôi)
- **CronJob** = lao công theo lịch (mỗi ngày 6h sáng)
- **HPA** = quản lý thêm/bớt nhân viên theo lượng khách

---

## 2. Sidecar Pattern - Trợ lý đi kèm {#sidecar}

### Sidecar là gì?

Sidecar container chạy CÙNG Pod với main container, bổ sung functionality mà không sửa main app.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-with-logging
spec:
  containers:
    # Main application
    - name: web-app
      image: myapp:1.0
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: logs
          mountPath: /var/log/app

    # Sidecar: Log shipper
    - name: log-shipper
      image: fluentd:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
          readOnly: true
      env:
        - name: ELASTICSEARCH_HOST
          value: "elasticsearch:9200"

  volumes:
    - name: logs
      emptyDir: {}
```

### Use cases phổ biến

```
1. Logging sidecar (Fluentd/Filebeat): Ship logs đến central
2. Proxy sidecar (Envoy/Istio): Service mesh, mTLS
3. Config watcher: Reload config khi ConfigMap thay đổi
4. Vault agent: Inject secrets từ HashiCorp Vault
5. CloudSQL proxy: Secure connection to managed database
```

---

## 3. Ambassador Pattern {#ambassador}

Ambassador container đóng vai trò **proxy** cho connections đi ra từ main app.

```yaml
# Ambassador proxy cho database connection
spec:
  containers:
    - name: app
      image: myapp:1.0
      env:
        - name: DB_HOST
          value: "localhost"    # Connect to ambassador
        - name: DB_PORT
          value: "5432"

    - name: cloudsql-proxy      # Ambassador
      image: gcr.io/cloudsql-docker/gce-proxy:1.33
      args:
        - "--instances=project:region:instance=tcp:5432"
        - "--credentials-file=/secrets/key.json"
```

---

## 4. Adapter Pattern {#adapter}

Adapter container chuyển đổi output của main app sang format chuẩn.

```yaml
# Adapter: chuyển đổi metrics format
spec:
  containers:
    - name: legacy-app
      image: legacy:1.0
      # Outputs metrics ở proprietary format

    - name: metrics-adapter
      image: prometheus-adapter:1.0
      # Reads proprietary metrics → exposes /metrics Prometheus format
      ports:
        - containerPort: 9090
          name: metrics
```

---

## 5. Init Containers - Chuẩn bị trước khi khởi động {#init-containers}

### Init Containers là gì?

Init containers chạy **TRƯỚC** main containers, theo thứ tự, mỗi cái phải thành công trước khi cái tiếp theo chạy.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  initContainers:
    # Wait for database to be ready
    - name: wait-for-db
      image: busybox:1.36
      command: ['sh', '-c', 'until nc -z postgres-service 5432; do echo waiting...; sleep 2; done']

    # Run database migrations
    - name: migrate
      image: myapp:1.0
      command: ['python', 'manage.py', 'migrate']
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url

    # Download config from S3
    - name: fetch-config
      image: amazon/aws-cli
      command: ['aws', 's3', 'cp', 's3://bucket/config.yaml', '/config/']
      volumeMounts:
        - name: config
          mountPath: /config

  containers:
    - name: app
      image: myapp:1.0
      volumeMounts:
        - name: config
          mountPath: /app/config

  volumes:
    - name: config
      emptyDir: {}
```

---

## 6. DaemonSet - Mỗi node một agent {#daemonset}

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          ports:
            - containerPort: 9100
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
      volumes:
        - name: proc
          hostPath:
            path: /proc

# Use cases:
# - Node monitoring (node-exporter, datadog-agent)
# - Log collection (fluentd, filebeat)
# - Network plugins (calico-node, cilium)
# - Storage plugins (ceph, longhorn)
```

---

## 7. StatefulSet - Ứng dụng có trạng thái {#statefulset}

### StatefulSet vs Deployment

```
Deployment:
- Pods interchangeable (không phân biệt)
- Pod names: web-abc123, web-def456 (random)
- Scale down: xóa bất kỳ pod nào
- Storage: shared hoặc ephemeral

StatefulSet:
- Pods có identity ổn định
- Pod names: db-0, db-1, db-2 (ordinal, stable)
- Scale down: xóa từ số cao nhất (db-2 trước)
- Storage: mỗi pod có PVC riêng (stable)
- DNS: db-0.db-headless.namespace.svc.cluster.local
```

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres-headless"   # Headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: pg-secret
                  key: password
  volumeClaimTemplates:              # PVC per pod
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 50Gi
```

---

## 8. Jobs và CronJobs {#jobs-cronjobs}

```yaml
# Job: chạy 1 lần
apiVersion: batch/v1
kind: Job
metadata:
  name: db-backup
spec:
  completions: 1
  backoffLimit: 3              # Retry 3 lần nếu fail
  activeDeadlineSeconds: 600   # Timeout 10 phút
  template:
    spec:
      containers:
        - name: backup
          image: postgres:16
          command: ["pg_dump", "-h", "postgres-0", "-U", "admin", "-f", "/backup/dump.sql", "mydb"]
      restartPolicy: Never

---
# CronJob: chạy theo lịch
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"       # 2AM mỗi ngày
  concurrencyPolicy: Forbid   # Không chạy overlap
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 5
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: backup-tool:latest
          restartPolicy: OnFailure
```

---

## 9. HPA - Horizontal Pod Autoscaler {#hpa}

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale khi CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scale down
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

---

## 10. Tổng kết {#tong-ket}

### Bảng quyết định

| Nhu cầu | Pattern/Resource |
|---------|-----------------|
| Thêm logging/metrics không sửa app | Sidecar |
| Proxy connection đến external service | Ambassador |
| Chuyển đổi format output | Adapter |
| Setup/migration trước khi app start | Init Container |
| Agent chạy trên mọi node | DaemonSet |
| Database, message queue (stateful) | StatefulSet |
| Batch processing, data migration | Job |
| Scheduled tasks (backup, cleanup) | CronJob |
| Auto-scale theo load | HPA |

---

*Bài viết tiếp theo: [Infrastructure as Code](/2026/08/19/infrastructure-as-code/)*
