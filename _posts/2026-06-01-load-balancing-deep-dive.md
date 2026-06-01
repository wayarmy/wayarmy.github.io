---
layout: post
title: "Load Balancing Deep Dive - Phân Phối Tải Từ A Đến Z"
date: 2026-06-01
categories: [networking]
tags: [load-balancing, aws, alb, nlb, high-availability]
---

# Load Balancing Deep Dive - Phân Phối Tải Từ A Đến Z

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Hãy tưởng tượng bạn đến một siêu thị lớn vào giờ cao điểm. Có 20 quầy thu ngân nhưng nếu tất cả khách hàng đều xếp vào một quầy, hàng dài sẽ kéo dài đến tận cửa ra vào trong khi 19 quầy còn lại đứng không. **Load Balancer** (Bộ Cân Bằng Tải) chính là người quản lý đứng ở cửa, hướng dẫn khách hàng đến quầy nào đang trống nhất.

**Ví dụ thực tế khác:**

- **Bệnh viện có nhiều phòng khám**: Nhân viên tiếp tân phân bệnh nhân đến bác sĩ nào đang rảnh nhất — đó là load balancing kiểu "Least Connections" (ít kết nối nhất)
- **Đường cao tốc có nhiều làn**: Biển báo hướng dẫn xe vào làn ít xe — đó là load balancing kiểu "Least Traffic"
- **Ngân hàng có nhiều chi nhánh**: Bạn luôn đến chi nhánh gần nhà nhất — đó là load balancing kiểu "IP Hash" (phân theo vị trí)
- **Nhà hàng buffet có nhiều bàn**: Nhân viên xếp khách lần lượt bàn 1, bàn 2, bàn 3, rồi quay lại bàn 1 — đó là "Round Robin"

Trong thế giới công nghệ, khi một website nhận hàng triệu lượt truy cập mỗi giây, không server nào đủ mạnh để xử lý một mình. Load Balancer phân phối các yêu cầu (requests) đến nhiều server khác nhau, đảm bảo:
- Không server nào bị quá tải
- Nếu một server "chết", traffic tự động chuyển sang server khác
- Người dùng luôn được phục vụ nhanh nhất có thể

---

## 2. Kiến Thức Nền Tảng Cần Biết

### 2.1 Server Là Gì?

Server (máy chủ) là máy tính mạnh chạy liên tục 24/7, chuyên phục vụ yêu cầu từ người dùng. Khi bạn mở Facebook, điện thoại bạn gửi yêu cầu đến server của Facebook, server xử lý và trả về trang web.

### 2.2 Tại Sao Cần Nhiều Server?

| Vấn đề | Giải pháp |
|---------|-----------|
| Một server chỉ xử lý được ~10,000 kết nối cùng lúc | Dùng 100 server = 1,000,000 kết nối |
| Server hỏng = website chết | Nhiều server = nếu 1 chết, 99 server còn lại vẫn hoạt động |
| Server ở Việt Nam phục vụ người Mỹ chậm | Đặt server ở nhiều nơi trên thế giới |

### 2.3 Layer 4 vs Layer 7 — Hai Tầng Hoạt Động

Mô hình mạng OSI có 7 tầng. Load Balancer hoạt động ở 2 tầng chính:

**Layer 4 (Transport Layer — Tầng Vận Chuyển):**
- Nhìn thấy: IP nguồn, IP đích, Port nguồn, Port đích, giao thức (TCP/UDP)
- KHÔNG nhìn thấy: Nội dung bên trong (URL, cookies, headers)
- Ví dụ đời thường: Người phân loại thư nhìn địa chỉ trên phong bì mà không mở thư ra đọc

**Layer 7 (Application Layer — Tầng Ứng Dụng):**
- Nhìn thấy TẤT CẢ: URL, HTTP headers, cookies, nội dung request body
- Có thể quyết định dựa trên nội dung: "/api/users" đến server A, "/api/images" đến server B
- Ví dụ đời thường: Nhân viên lễ tân khách sạn đọc yêu cầu của khách rồi hướng dẫn: muốn massage → tầng 3, muốn ăn → tầng 1, muốn hồ bơi → tầng 5

| Tiêu chí | Layer 4 | Layer 7 |
|-----------|---------|---------|
| Tốc độ | Rất nhanh (không cần đọc nội dung) | Chậm hơn (phải phân tích nội dung) |
| Thông minh | Đơn giản, chỉ dựa IP/Port | Phức tạp, dựa URL/cookie/header |
| SSL | Passthrough (chuyển tiếp nguyên) | Termination (giải mã tại LB) |
| Use case | Game online, streaming video, IoT | Web apps, APIs, microservices |
| Chi phí CPU | Thấp | Cao (phải decrypt + inspect) |

### 2.4 Upstream vs Downstream

- **Upstream** (Thượng nguồn): Các server phía sau load balancer (backend servers / target servers)
- **Downstream** (Hạ nguồn): Clients/người dùng phía trước load balancer
- **Backend pool / Target group**: Nhóm các server mà load balancer phân phối traffic đến

---

## 3. Các Thuật Toán Load Balancing Chi Tiết

### 3.1 Round Robin (RR) — Chia Đều Theo Lượt

**Nguyên lý:** Phân phối yêu cầu lần lượt theo thứ tự vòng tròn: Server 1 → Server 2 → Server 3 → Server 1 → ...

**Ví dụ đời thường:** Giáo viên gọi học sinh trả lời câu hỏi theo thứ tự danh sách lớp, đến cuối danh sách thì quay lại đầu.

```
Request 1 → Server A
Request 2 → Server B  
Request 3 → Server C
Request 4 → Server A (quay lại)
Request 5 → Server B
Request 6 → Server C
...
```

**Ưu điểm:**
- Đơn giản nhất, dễ implement
- Không cần theo dõi trạng thái server
- Phân phối đều nếu tất cả request giống nhau

**Nhược điểm:**
- Không biết server nào đang bận: Server A đang xử lý request nặng (upload file 1GB) vẫn nhận thêm request mới
- Không phù hợp khi server có cấu hình khác nhau (server mạnh nhận cùng lượng với server yếu)

**Khi nào dùng:** Web server stateless, tất cả server cùng cấu hình, request đều nhẹ và nhanh.

### 3.2 Weighted Round Robin (WRR) — Chia Theo Trọng Số

**Nguyên lý:** Giống Round Robin nhưng server mạnh hơn nhận nhiều hơn theo tỷ lệ weight (trọng số).

**Ví dụ đời thường:** Trong nhà hàng, bồi bàn có kinh nghiệm phục vụ 5 bàn, bồi bàn mới chỉ phục vụ 2 bàn.

```
Server A (weight=5): CPU 8 cores, 32GB RAM → nhận 5 phần
Server B (weight=3): CPU 4 cores, 16GB RAM → nhận 3 phần  
Server C (weight=2): CPU 2 cores, 8GB RAM → nhận 2 phần

Thứ tự: A, A, A, A, A, B, B, B, C, C, A, A, A, A, A, B, B, B, C, C...
```

**Cải tiến Smooth Weighted Round Robin:**
Thay vì gửi 5 request liên tiếp đến A (gây burst), thuật toán smooth phân phối đều hơn:
```
A, B, A, C, A, B, A, B, A, C (vẫn đúng tỷ lệ 5:3:2 nhưng xen kẽ)
```

**Khi nào dùng:** Khi server có cấu hình khác nhau, khi muốn dần dần thêm/bớt traffic (canary deployment — gán weight thấp cho server mới).

### 3.3 Least Connections (LC) — Ít Kết Nối Nhất

**Nguyên lý:** Gửi request đến server đang có ít kết nối đang hoạt động (active connections) nhất.

**Ví dụ đời thường:** Khi bạn đến ngân hàng, bạn nhìn các quầy và chọn quầy có ít người xếp hàng nhất.

```
Server A: 150 active connections
Server B: 89 active connections  ← Request mới đến đây
Server C: 201 active connections
```

**Ưu điểm:**
- Tự động cân bằng khi request có thời gian xử lý khác nhau
- Server đang bận tự nhiên nhận ít request hơn

**Nhược điểm:**
- Cần theo dõi số connections realtime (overhead nhẹ)
- Không phân biệt connection nhẹ (đã gần xong) vs connection nặng (mới bắt đầu upload 10GB)

**Biến thể - Weighted Least Connections:**
```
Score = active_connections / weight
Server A: 150 connections, weight 5 → score = 30
Server B: 89 connections, weight 3 → score = 29.7 ← Chọn server này
Server C: 100 connections, weight 2 → score = 50
```

**Khi nào dùng:** WebSocket connections (kéo dài), API server có request nặng nhẹ khác nhau, database connection pooling.

### 3.4 IP Hash — Phân Theo Địa Chỉ IP

**Nguyên lý:** Dùng hàm hash trên IP của client để xác định server. Cùng một IP luôn đến cùng một server.

**Ví dụ đời thường:** Bạn luôn đến cùng một bác sĩ gia đình — bác sĩ đã biết lịch sử bệnh của bạn, không cần kể lại từ đầu.

```
hash("192.168.1.100") % 3 = 0 → Server A
hash("10.0.0.50") % 3 = 1 → Server B
hash("172.16.0.25") % 3 = 2 → Server C

Lần sau "192.168.1.100" truy cập → vẫn Server A (cùng hash)
```

**Ưu điểm:**
- Session persistence tự nhiên (không cần sticky session riêng)
- Tốt cho caching: cùng user luôn đến cùng server → cache hit rate cao

**Nhược điểm:**
- Khi thêm/bớt server, hash thay đổi → TẤT CẢ user bị chuyển server → mất session/cache
- Nhiều user sau cùng NAT IP → cùng hash → 1 server quá tải
- Phân phối không đều nếu IP phân bố không uniform

**Khi nào dùng:** Hệ thống cần đơn giản session stickiness, application cache per-server, số lượng server ít thay đổi.

### 3.5 Consistent Hashing — Hash Nhất Quán

**Nguyên lý:** Giải quyết vấn đề lớn nhất của IP Hash: khi thêm/bớt server, chỉ ảnh hưởng tối thiểu traffic.

**Ví dụ đời thường:** 
Tưởng tượng một chiếc đồng hồ tròn. Bạn đánh dấu vị trí các server trên mặt đồng hồ (ví dụ: Server A ở vị trí 2 giờ, Server B ở 6 giờ, Server C ở 10 giờ). Khi có request, bạn hash IP client ra một điểm trên đồng hồ, rồi đi theo chiều kim đồng hồ đến server gần nhất.

```
Vòng Hash (0 đến 360 độ):

     Server A (90°)
         |
    _____|_____
   /           \
  |      0°     |
  |             |
Server C      Server B
 (270°)        (180°)

Client IP hash = 120° → đi theo chiều kim → gặp Server B (180°) → đến B
Client IP hash = 200° → đi theo chiều kim → gặp Server C (270°) → đến C
```

**Khi bỏ Server B:**
- Chỉ traffic từ 90°-180° (trước đó đến B) chuyển sang C
- Traffic đến A và C KHÔNG thay đổi!
- So với IP Hash thường: TOÀN BỘ traffic phải tính lại

**Virtual Nodes (Vnodes) — Nút Ảo:**
Để phân phối đều hơn, mỗi server có nhiều điểm trên vòng hash:
```
Server A → A1(30°), A2(120°), A3(200°), A4(310°)
Server B → B1(60°), B2(150°), B3(240°), B4(350°)
```
Càng nhiều vnodes → phân phối càng đều.

**Ưu điểm:**
- Thêm/bớt server: chỉ ~1/N traffic bị ảnh hưởng (N = số server)
- Rất tốt cho distributed cache (Memcached, Redis Cluster)
- Dùng vnodes giải quyết vấn đề phân phối không đều

**Nhược điểm:**
- Phức tạp hơn IP Hash đơn giản
- Cần duy trì hash ring (vòng hash) synchronized giữa các node
- Load không hoàn toàn đều (phụ thuộc phân phối hash)

**Khi nào dùng:** CDN routing, distributed cache, database sharding, hệ thống cần scale out/in thường xuyên.

### 3.6 Least Response Time — Thời Gian Phản Hồi Ngắn Nhất

**Nguyên lý:** Gửi request đến server có thời gian phản hồi (response time) ngắn nhất + ít connections nhất.

**Ví dụ đời thường:** Bạn gọi Grab, app chọn tài xế vừa gần bạn nhất VÀ vừa đang rảnh (không có khách).

```
Server A: response time 50ms, 100 connections
Server B: response time 20ms, 150 connections ← Nhanh hơn → chọn
Server C: response time 80ms, 80 connections
```

**Khi nào dùng:** API gateway, microservices có response time khác nhau, global load balancing (chọn region gần nhất).

### 3.7 So Sánh Tổng Hợp Các Thuật Toán

| Thuật toán | Complexity | Statefulness | Best For |
|-----------|-----------|-------------|----------|
| Round Robin | O(1) | Stateless | Uniform stateless services |
| Weighted RR | O(1) | Stateless | Heterogeneous servers |
| Least Connections | O(n) | Stateful | Long-lived connections |
| IP Hash | O(1) | Stateless | Session affinity |
| Consistent Hash | O(log n) | Stateless | Distributed caches |
| Least Response Time | O(n) | Stateful | Latency-sensitive apps |

---

## 4. Health Checks — Kiểm Tra Sức Khỏe Server

### 4.1 Tại Sao Cần Health Check?

**Ví dụ đời thường:** Quản lý nhà hàng cần biết bếp nào đang hoạt động. Nếu bếp số 3 bị hỏng nhưng quản lý không biết, khách hàng order vào bếp 3 sẽ chờ mãi không có đồ ăn.

Health Check = Load Balancer định kỳ "hỏi thăm" mỗi server: "Bạn còn sống không? Bạn có khỏe không?"

### 4.2 Các Loại Health Check

**Active Health Check (Chủ động):**
Load Balancer tự gửi request kiểm tra định kỳ.

```
Mỗi 10 giây, LB gửi: GET /health HTTP/1.1
Server khỏe trả: HTTP 200 OK {"status": "healthy", "db": "connected"}
Server bệnh trả: HTTP 503 Service Unavailable
Server chết: không trả lời (timeout)
```

**Cấu hình điển hình:**
```
Health Check Configuration:
- Protocol: HTTP
- Path: /health
- Port: 8080
- Interval: 10 seconds (mỗi 10 giây kiểm tra 1 lần)
- Timeout: 5 seconds (chờ tối đa 5 giây)
- Healthy threshold: 3 (3 lần liên tiếp OK → đánh dấu healthy)
- Unhealthy threshold: 2 (2 lần liên tiếp fail → đánh dấu unhealthy)
```

**Passive Health Check (Bị động):**
Load Balancer quan sát traffic thật để phát hiện lỗi.

```
LB quan sát:
- Server A trả 5xx liên tiếp 5 lần → đánh dấu unhealthy
- Server B timeout 3 lần trong 30 giây → đánh dấu unhealthy
- Không cần gửi request riêng, dựa trên traffic thật
```

### 4.3 Deep Health Check vs Shallow Health Check

**Shallow (Nông):** Chỉ kiểm tra process đang chạy
```
GET /health → 200 OK (chỉ biết web server sống)
```

**Deep (Sâu):** Kiểm tra toàn bộ dependencies
```
GET /health/deep → kiểm tra:
  ✓ Web server đang chạy
  ✓ Database kết nối được
  ✓ Redis cache hoạt động
  ✓ Disk còn dung lượng > 10%
  ✓ Memory < 90%
```

**Cảnh báo:** Deep health check có thể gây "cascading failure" (lỗi dây chuyền): nếu database chết → TẤT CẢ server báo unhealthy → LB tưởng không có server nào sống → toàn bộ hệ thống sập.

**Best practice:** Dùng shallow cho LB routing, deep cho monitoring/alerting.

---

## 5. Session Persistence (Sticky Sessions) — Dính Phiên

### 5.1 Vấn Đề

**Ví dụ đời thường:** Bạn đang mua hàng online, bỏ 5 món vào giỏ hàng. Giỏ hàng được lưu trên Server A. Request tiếp theo load balancer gửi đến Server B → Server B không biết giỏ hàng của bạn → bạn thấy giỏ hàng trống!

### 5.2 Giải Pháp: Sticky Sessions

Load Balancer đảm bảo cùng một user luôn đến cùng một server trong suốt phiên làm việc.

**Cách 1: Cookie-based Stickiness**
```
Lần đầu: Client → LB → Server A (LB thêm cookie: SERVERID=A)
Lần sau: Client gửi cookie SERVERID=A → LB → Server A
```

**Cách 2: Source IP Stickiness**
```
Dựa trên IP client → luôn đến cùng server
Vấn đề: nhiều user sau NAT cùng IP → cùng server → mất cân bằng
```

### 5.3 Nhược Điểm và Giải Pháp Thay Thế

**Nhược điểm sticky session:**
- Server chết → mất session → user phải đăng nhập lại
- Phân phối không đều: user "nặng" (dùng lâu) dồn vào 1 server
- Khó scale: không thể tự do thêm/bớt server

**Giải pháp tốt hơn: Externalize Session (Lưu session bên ngoài)**
```
Thay vì lưu session trong memory của mỗi server:
→ Lưu vào Redis/Memcached cluster chung
→ Server nào cũng đọc được session
→ Không cần sticky session nữa!

Client → LB → Server A (đọc session từ Redis)
Client → LB → Server B (cũng đọc được session từ Redis)
```

---

## 6. SSL/TLS Termination vs Passthrough

### 6.1 SSL Termination (Kết Thúc SSL Tại Load Balancer)

**Ví dụ đời thường:** Bạn gửi thư mật mã (mã hóa) đến công ty. Bảo vệ ở cổng giải mã thư, đọc nội dung, rồi chuyển thư (không mã hóa) đến đúng phòng ban bên trong.

```
Client ←HTTPS (encrypted)→ Load Balancer ←HTTP (plaintext)→ Backend Server

Quá trình:
1. Client gửi HTTPS request (encrypted)
2. LB decrypt (giải mã) request
3. LB đọc nội dung (URL, headers) → quyết định routing
4. LB forward HTTP (không mã hóa) đến backend
5. Backend trả HTTP response
6. LB encrypt response → gửi HTTPS về client
```

**Ưu điểm:**
- Backend servers không cần xử lý SSL → tiết kiệm CPU
- Quản lý certificate tập trung tại LB (1 chỗ thay vì 100 server)
- LB có thể inspect traffic → routing thông minh, WAF, logging
- Dễ setup: chỉ cần cài cert 1 lần trên LB

**Nhược điểm:**
- Traffic LB → Backend không mã hóa (rủi ro nếu mạng nội bộ bị xâm nhập)
- LB trở thành single point of decryption → bottleneck nếu traffic lớn
- Không phù hợp cho compliance yêu cầu end-to-end encryption

**Giải pháp:** Re-encryption — LB decrypt rồi encrypt lại bằng internal cert:
```
Client ←HTTPS (public cert)→ LB ←HTTPS (internal cert)→ Backend
```

### 6.2 SSL Passthrough (Chuyển Tiếp SSL Nguyên Vẹn)

**Ví dụ đời thường:** Bưu điện chuyển thư mật mã nguyên vẹn đến người nhận mà không mở ra đọc.

```
Client ←HTTPS (encrypted)→ Load Balancer ←HTTPS (same encrypted)→ Backend

LB chỉ nhìn thấy: IP nguồn, IP đích, Port (Layer 4 info)
LB KHÔNG nhìn thấy: URL, cookies, headers (vì bị mã hóa)
```

**Ưu điểm:**
- End-to-end encryption (mã hóa đầu-cuối) — bảo mật tối đa
- LB không cần CPU cho decrypt → hiệu suất cao hơn
- Phù hợp compliance strict (PCI-DSS, HIPAA)
- Certificate management tại backend (mỗi service tự quản lý cert riêng)

**Nhược điểm:**
- LB không đọc được nội dung → chỉ route theo IP/Port (Layer 4)
- Không thể dùng tính năng Layer 7: URL routing, cookie stickiness, WAF
- Phải quản lý certificate trên TỪNG backend server
- Không thể insert headers (X-Forwarded-For) → backend không biết IP thật của client

### 6.3 Khi Nào Dùng Cái Nào?

| Yêu cầu | SSL Termination | SSL Passthrough |
|----------|----------------|-----------------|
| URL-based routing | ✅ | ❌ |
| WAF integration | ✅ | ❌ |
| End-to-end encryption | ❌ (trừ khi re-encrypt) | ✅ |
| CPU efficiency tại LB | ❌ (decrypt tốn CPU) | ✅ |
| Certificate management | Tập trung (dễ) | Phân tán (khó) |
| Compliance strict | Cần re-encryption | ✅ native |

---

## 7. Connection Draining (Deregistration Delay) — Thoát Kết Nối Mượt Mà

### 7.1 Vấn Đề

**Ví dụ đời thường:** Bạn đang cắt tóc nửa chừng thì tiệm thông báo đóng cửa. Sẽ thế nào nếu:
- **Không có draining:** Tiệm đuổi bạn ra ngay lập tức → tóc cắt nửa dở
- **Có draining:** Tiệm không nhận khách MỚI, nhưng để thợ hoàn thành khách ĐANG cắt → bạn được cắt xong rồi mới ra

### 7.2 Cách Hoạt Động

```
Trạng thái bình thường:
LB → Server A (healthy, nhận traffic)

Bắt đầu draining (khi muốn bảo trì/remove Server A):
1. LB đánh dấu Server A = "draining"
2. Request MỚI → KHÔNG gửi đến Server A nữa
3. Request ĐANG xử lý trên Server A → chờ hoàn thành
4. Sau timeout (ví dụ 300 giây):
   - Nếu tất cả request xong → remove Server A sạch sẽ
   - Nếu còn request chưa xong → force close (đóng cưỡng bức)
```

### 7.3 Ứng Dụng Thực Tế

**Rolling Deployment (Triển khai cuốn chiếu):**
```
Có 4 server chạy version 1.0, muốn nâng lên version 2.0:

Bước 1: Drain Server A → chờ request xong → Remove → Deploy v2.0 → Add back
Bước 2: Drain Server B → chờ request xong → Remove → Deploy v2.0 → Add back
Bước 3: Drain Server C → ... 
Bước 4: Drain Server D → ...

Kết quả: Zero-downtime deployment!
```

**Auto Scaling Down:**
```
Giờ thấp điểm, hệ thống tự scale từ 10 server → 5 server:
- 5 server bị remove sẽ drain trước
- Connections đang chạy hoàn thành bình thường
- Người dùng không bị ảnh hưởng
```

**AWS Default:** Deregistration delay = 300 giây (5 phút). Có thể chỉnh từ 0–3600 giây.

---

## 8. AWS Load Balancer Family — Gia Đình Load Balancer AWS

### 8.1 Application Load Balancer (ALB) — Layer 7

**Ví dụ đời thường:** Như nhân viên lễ tân thông minh — đọc yêu cầu của khách rồi hướng dẫn đến đúng bộ phận.

**Đặc điểm chính:**
- Hoạt động ở Layer 7 (HTTP/HTTPS/gRPC/WebSocket)
- **Content-based routing:** route dựa trên URL path, hostname, headers, query strings
- **Host-based routing:** api.example.com → Target Group A, web.example.com → Target Group B
- **Path-based routing:** /api/* → microservice backend, /images/* → static server

**Tính năng nâng cao:**
```
Routing Rules (Luật định tuyến):
- IF path = "/api/v2/*" AND header["X-Custom"] = "beta" 
  → Forward to Beta Target Group (weight: 90%)
  → Forward to Canary Target Group (weight: 10%)

- IF host = "admin.example.com" AND source-ip IN [10.0.0.0/8]
  → Forward to Admin Target Group

- DEFAULT → Fixed Response 404
```

**Targets hỗ trợ:** EC2 instances, IP addresses, Lambda functions, containers (ECS/EKS)

**Sticky Sessions:** Cookie-based (application cookie hoặc duration-based cookie)

**Giá:** Tính theo LCU (Load Balancer Capacity Unit) — dựa trên connections, bandwidth, rules evaluated/giây

### 8.2 Network Load Balancer (NLB) — Layer 4

**Ví dụ đời thường:** Như đường cao tốc với nhiều làn — xe chạy qua cực nhanh, không cần dừng kiểm tra hành lý.

**Đặc điểm chính:**
- Hoạt động ở Layer 4 (TCP/UDP/TLS)
- **Hiệu suất cực cao:** Xử lý hàng triệu requests/giây với latency cực thấp (< 100 microsecond)
- **Static IP:** Mỗi AZ có 1 static IP (hoặc gán Elastic IP) — tốt cho DNS, whitelist
- **Preserve source IP:** Backend thấy IP thật của client (không cần X-Forwarded-For)
- **TLS Passthrough hoặc Termination:** Hỗ trợ cả hai

**So sánh hiệu suất:**
```
ALB: ~100,000 requests/giây typical
NLB: ~1,000,000+ requests/giây typical
     Latency: single-digit microseconds (so với ALB: milliseconds)
```

**Use cases:**
- Game servers (TCP/UDP real-time)
- IoT backends (millions of connections)
- VoIP/SIP (UDP)
- gRPC (HTTP/2 with long-lived connections)
- Anything needing static IP or extreme performance

### 8.3 Gateway Load Balancer (GWLB) — Layer 3

**Ví dụ đời thường:** Như trạm kiểm soát an ninh trên đường cao tốc — tất cả xe phải đi qua trạm kiểm tra trước khi tiếp tục hành trình. Xe không biết mình đang bị kiểm tra (transparent).

**Đặc điểm chính:**
- Hoạt động ở Layer 3 (Network layer) — transparent to traffic
- Dùng cho **security appliances:** Firewalls, IDS/IPS, deep packet inspection
- **GENEVE encapsulation (port 6081):** Đóng gói traffic gửi đến appliances, sau đó trả về original path
- Kết hợp: Gateway (single entry/exit point) + Load Balancer (distribute to appliances)

**Kiến trúc:**
```
Internet → VPC Routing → GWLB Endpoint
                              ↓
                    Gateway Load Balancer
                    /         |         \
            Firewall 1   Firewall 2   Firewall 3
                    \         |         /
                    Gateway Load Balancer
                              ↓
                    GWLB Endpoint → Application Servers
```

**Use cases:**
- Third-party virtual firewalls (Palo Alto, Fortinet, Check Point)
- Intrusion Detection/Prevention Systems (IDS/IPS)
- Network packet inspection
- Compliance filtering

### 8.4 So Sánh Chi Tiết

| Feature | ALB | NLB | GWLB |
|---------|-----|-----|------|
| OSI Layer | 7 | 4 | 3 |
| Protocols | HTTP, HTTPS, gRPC, WebSocket | TCP, UDP, TLS | IP (all traffic) |
| Routing | URL, Header, Host, Query | Port, Protocol | N/A (transparent) |
| Performance | Good | Extreme | Good |
| Static IP | ❌ (chỉ có DNS name) | ✅ Elastic IP | ✅ |
| Source IP | Via X-Forwarded-For | Preserved natively | Preserved |
| SSL | Termination only | Both (term + passthrough) | N/A |
| Health check | HTTP/HTTPS | TCP/HTTP/HTTPS | TCP/HTTP/HTTPS |
| Target types | Instance, IP, Lambda | Instance, IP, ALB | Instance, IP |
| Price base | LCU-based | NLCU-based | GLCU-based |
| Use case | Web apps, APIs | Gaming, IoT, gRPC | Security appliances |

---

## 9. Mô Hình Triển Khai và Best Practices

### 9.1 Multi-Layer Load Balancing

```
Kiến trúc production thực tế:

Internet
    ↓
[DNS Load Balancing - Route 53]  ← Global (chọn region gần nhất)
    ↓
[NLB - Layer 4]  ← Regional (high performance, static IP)
    ↓
[ALB - Layer 7]  ← Application (content routing)
    ↓
[Target Groups]
  ├── /api/* → API servers (ECS Fargate)
  ├── /static/* → S3 + CloudFront
  └── /ws/* → WebSocket servers (EC2)
```

### 9.2 Cross-Zone Load Balancing

**Vấn đề:** Nếu AZ-a có 2 server và AZ-b có 8 server:
- Không cross-zone: mỗi AZ nhận 50% traffic → AZ-a mỗi server nhận 25%, AZ-b mỗi server nhận 6.25%
- Có cross-zone: traffic phân đều cho tất cả 10 server → mỗi server nhận 10%

```
Không Cross-Zone:            Có Cross-Zone:
50% → AZ-a (2 srv)          AZ-a: Server 1 = 10%
  Server 1 = 25%                   Server 2 = 10%
  Server 2 = 25%            AZ-b: Server 3 = 10%
50% → AZ-b (8 srv)                Server 4 = 10%
  Server 3 = 6.25%                 ...
  Server 4 = 6.25%                 Server 10 = 10%
  ...
  Server 10 = 6.25%
```

**AWS Default:**
- ALB: Cross-zone enabled (miễn phí)
- NLB: Cross-zone disabled (có phí nếu enable — data transfer cross-AZ)

### 9.3 Best Practices Checklist

```
✅ Health Checks:
  - Dùng dedicated health check endpoint (/health)
  - Healthy threshold: 2-3, Unhealthy threshold: 2-3
  - Interval: 10-30s (không quá ngắn gây load)
  
✅ Connection Draining:
  - Enable với timeout phù hợp (30-300s)
  - Adjust theo longest request time của app

✅ SSL/TLS:
  - ALB: SSL termination + re-encryption (TLS 1.2+)
  - NLB: SSL passthrough cho end-to-end encryption
  - Dùng AWS ACM cho free managed certificates

✅ Monitoring:
  - CloudWatch metrics: RequestCount, TargetResponseTime, 
    HealthyHostCount, UnHealthyHostCount, 5xxCount
  - Access logs → S3 → Athena for analysis
  
✅ Security:
  - Security Groups (ALB) / NACLs
  - WAF integration (ALB only)
  - Delete protection enabled
  
✅ Scaling:
  - Pre-warming cho expected traffic spikes
  - Contact AWS support trước sự kiện lớn (Black Friday)
```

### 9.4 Anti-Patterns (Những Điều Nên Tránh)

```
❌ Dùng ALB cho non-HTTP traffic (game UDP) → Dùng NLB
❌ Health check gọi heavy database query → Dùng lightweight endpoint
❌ Sticky session thay cho external session store → Dùng Redis
❌ Single AZ deployment → Luôn multi-AZ
❌ Connection draining = 0 → Minimum 30 giây
❌ Quá nhiều rules trên ALB (>100) → Tách multiple ALBs
❌ Không monitor unhealthy targets → Set alarms
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 Cheat Sheet Chọn Thuật Toán

```
Tất cả server cùng cấu hình, request đều nhẹ?
  → Round Robin

Server khác cấu hình?
  → Weighted Round Robin

Request có thời gian xử lý khác nhau nhiều?
  → Least Connections

Cần session affinity đơn giản?
  → IP Hash

Hệ thống scale thường xuyên + cần cache affinity?
  → Consistent Hashing

Cần latency thấp nhất?
  → Least Response Time
```

### 10.2 Cheat Sheet Chọn AWS Load Balancer

```
HTTP/HTTPS web application, cần URL routing?
  → ALB

TCP/UDP, cần extreme performance hoặc static IP?
  → NLB

Third-party firewalls/IDS/IPS?
  → GWLB

Cần cả Layer 7 routing VÀ static IP?
  → NLB (TCP listener) → ALB (as target)
```

### 10.3 Key Takeaways

1. **Load Balancing = Phân phối tải** — không server nào bị quá tải, không server nào ngồi không
2. **Layer 4 nhanh nhưng "mù"** — chỉ biết IP/Port, không biết nội dung
3. **Layer 7 chậm hơn nhưng thông minh** — đọc được URL, cookies, headers
4. **Health Check là BẮT BUỘC** — không có health check = gửi traffic đến server chết
5. **Connection Draining = graceful shutdown** — không "giật dây" đột ngột
6. **SSL Termination vs Passthrough** — trade-off giữa tính năng và bảo mật
7. **Sticky Session là anti-pattern** — ưu tiên externalized session (Redis)
8. **Consistent Hashing** là thuật toán quan trọng nhất cho distributed systems

### 10.4 Tài Liệu Tham Khảo Chính Thức

- RFC 7230-7235: HTTP/1.1 Protocol — định nghĩa HTTP mà Layer 7 LB phân tích
- RFC 2616: Hypertext Transfer Protocol (HTTP/1.1) — original HTTP spec
- AWS Documentation: [Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/)
- AWS Well-Architected Framework — Reliability Pillar
- Karger et al., "Consistent Hashing and Random Trees" (1997) — paper gốc về consistent hashing
- NGINX documentation: Load Balancing algorithms
- HAProxy documentation: Server health checks
- Google SRE Book, Chapter 20: "Load Balancing in the Datacenter"

---

*Bài viết tiếp theo: CDN & Content Delivery — Cách internet nhanh hơn nhờ "kho hàng gần nhà"*
