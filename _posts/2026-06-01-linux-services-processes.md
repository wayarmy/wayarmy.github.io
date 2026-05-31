---
layout: post
title: "Linux Fundamentals - Phần 8: Services & Processes"
subtitle: "Quản lý tiến trình, dịch vụ, packages và lập lịch trên Linux"
gh-repo: wayarmy/wayarmy.github.io
tags: [linux, aws, learning-path]
comments: true
date: 2026-06-01
categories: AWS-Learning-Path
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Phần 8).
>
> **Đối tượng:** Người mới hoàn toàn — không cần kiến thức IT trước.
>
> **Nguồn tham khảo:**
> - systemd documentation — [https://www.freedesktop.org/wiki/Software/systemd/](https://www.freedesktop.org/wiki/Software/systemd/)
> - Linux man pages: proc(5), ps(1), top(1), kill(1), crontab(5)
> - "How Linux Works" by Brian Ward — Chapter 4: Processes and Scheduling
> - APT documentation — [https://wiki.debian.org/Apt](https://wiki.debian.org/Apt)
> - YUM/DNF documentation — [https://dnf.readthedocs.io/](https://dnf.readthedocs.io/)
> - AWS Documentation: [EC2 User Data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html), [EventBridge](https://docs.aws.amazon.com/eventbridge/)

---

## 1. Process — "Chương trình đang chạy"

### Ví dụ đời thường:

Hãy nghĩ về **một nhà bếp** (CPU) nấu nhiều món:

- **Chương trình (Program)** = Công thức nấu ăn trên giấy → nằm im, chưa làm gì
- **Process (Tiến trình)** = Đầu bếp ĐANG nấu theo công thức → đang hoạt động, dùng nguyên liệu (RAM), chiếm bếp (CPU)
- **Thread** = Một đầu bếp có thể vừa nấu nước, vừa thái rau → nhiều việc song song trong 1 process

Mỗi process có:
- **PID (Process ID):** Số định danh duy nhất (giống CMND/CCCD)
- **PPID (Parent PID):** Process nào sinh ra nó (process cha)
- **Owner:** User nào sở hữu
- **State:** Đang chạy? Đang ngủ? Đã chết?

### Vòng đời của Process:

```
Fork (sinh ra) → Running (đang chạy) → Sleeping (chờ I/O)
                      │                       │
                      ▼                       ▼
               Stopped (dừng tạm)     Running (tiếp tục)
                      │                       │
                      └───────────────────────┘
                                │
                                ▼
                      Zombie (chết nhưng chưa dọn)
                                │
                                ▼
                         Terminated (kết thúc)
```

### Process States:

| State | Ký hiệu | Ý nghĩa |
|-------|----------|----------|
| Running | R | Đang chạy hoặc sẵn sàng chạy |
| Sleeping | S | Đang đợi event (I/O, signal) — interruptible |
| Disk Sleep | D | Đang đợi I/O — uninterruptible (không thể kill!) |
| Stopped | T | Bị dừng (Ctrl+Z hoặc SIGSTOP) |
| Zombie | Z | Đã kết thúc nhưng parent chưa collect exit status |

---

## 2. Các lệnh quản lý Process

### `ps` — Process Status (Ảnh chụp tại 1 thời điểm)

```bash
# Xem processes của user hiện tại
$ ps
  PID TTY          TIME CMD
 1234 pts/0    00:00:00 bash
 5678 pts/0    00:00:00 ps

# Xem TẤT CẢ processes (format đầy đủ)
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root         1  0.0  0.1 169560 13300 ?     Ss   Jun01   0:05 /usr/lib/systemd/systemd
root         2  0.0  0.0      0     0 ?     S    Jun01   0:00 [kthreadd]
www-data  1045  0.5  1.2 345678 98765 ?     S    Jun01   2:30 nginx: worker process
alice     2345  0.1  0.3  56789 12345 pts/0 S    10:00   0:01 vim document.txt
alice     3456  0.0  0.0  12345  2345 pts/0 R+   10:30   0:00 ps aux

# Giải thích cột:
# USER  = Owner
# PID   = Process ID
# %CPU  = CPU usage
# %MEM  = Memory usage
# VSZ   = Virtual memory size (KB)
# RSS   = Resident Set Size — RAM thực dùng (KB)
# STAT  = State (R=Running, S=Sleeping, Z=Zombie)
# TIME  = Tổng CPU time đã dùng

# Xem dạng cây (parent-child)
$ ps auxf
# hoặc
$ pstree -p
systemd(1)─┬─sshd(890)───sshd(1234)───bash(1235)───vim(2345)
            ├─nginx(1000)─┬─nginx(1045)
            │             └─nginx(1046)
            └─cron(567)
```

### `top` / `htop` — Monitor realtime

```bash
$ top
```

```
top - 10:30:00 up 5 days, 3:20, 2 users, load average: 0.15, 0.10, 0.05
Tasks: 156 total,   1 running, 155 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.3 us,  1.0 sy,  0.0 ni, 96.5 id,  0.2 wa,  0.0 hi,  0.0 si
MiB Mem:   7963.4 total,   4521.2 free,   1890.1 used,   1552.1 buff/cache
MiB Swap:  2048.0 total,   2048.0 free,      0.0 used.   5773.3 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 1045 www-data  20   0  345678  98765  12345 S   2.3   1.2   2:30.45 nginx
 2345 alice     20   0   56789  12345   4567 S   0.5   0.2   0:01.23 vim
```

**Giải thích Load Average:** `0.15, 0.10, 0.05` = trung bình 1 phút, 5 phút, 15 phút
- Trên máy 1 CPU: load = 1.0 nghĩa là CPU bận 100%
- Trên máy 4 CPU: load = 4.0 nghĩa là bận 100%
- Rule of thumb: Load nên < số CPU cores

**Phím tắt trong top:**
- `q` = Thoát
- `k` = Kill process (nhập PID)
- `M` = Sort theo Memory
- `P` = Sort theo CPU
- `1` = Hiện từng CPU core
- `h` = Help

**htop** — Phiên bản đẹp hơn (cần cài thêm):
```bash
sudo apt install htop    # Ubuntu/Debian
htop
```

### `kill` — Gửi signal đến process

**Signals** giống "lệnh" gửi cho process:

| Signal | Số | Ý nghĩa | Ví dụ |
|--------|-----|---------|-------|
| SIGHUP | 1 | Hangup — reload config | `kill -1 PID` → Nginx reload |
| SIGINT | 2 | Interrupt (Ctrl+C) | Dừng "nhẹ nhàng" |
| SIGKILL | 9 | Kill ngay — KHÔNG THỂ bỏ qua | `kill -9 PID` → Force kill |
| SIGTERM | 15 | Terminate — "xin hãy dừng lại" (default) | `kill PID` |
| SIGSTOP | 19 | Pause (Ctrl+Z) | Tạm dừng, resume bằng `fg`/`bg` |
| SIGCONT | 18 | Continue | Resume sau SIGSTOP |

```bash
# Dừng nhẹ nhàng (process có thể cleanup trước khi thoát)
$ kill 1234          # Gửi SIGTERM (default)
$ kill -15 1234      # Tương tự

# Force kill (dùng khi SIGTERM không work)
$ kill -9 1234       # SIGKILL — chết ngay lập tức

# Kill theo tên
$ killall nginx      # Kill tất cả processes tên "nginx"
$ pkill -f "python app.py"   # Kill process match pattern
```

**Ví dụ đời thường:**
- `SIGTERM` = Bảo nhân viên: "Bạn hoàn thành nốt việc rồi về nhé" → họ dọn dẹp rồi về
- `SIGKILL` = Bảo vệ lôi nhân viên ra ngay → không kịp dọn gì cả

### Foreground vs Background:

```bash
# Chạy foreground (mặc định) — chiếm terminal
$ python long_script.py
^Z                           # Ctrl+Z = Pause (SIGSTOP)
[1]+  Stopped

# Chuyển sang background
$ bg                         # Resume ở background
[1]+ python long_script.py &

# Hoặc chạy trực tiếp ở background
$ python long_script.py &    # & = chạy background ngay
[1] 5678                     # PID = 5678

# Xem background jobs
$ jobs
[1]+  Running    python long_script.py &

# Đưa về foreground
$ fg %1                      # Hoặc fg (nếu chỉ có 1 job)

# Chạy không bị kill khi thoát SSH
$ nohup python long_script.py &    # no hangup + background
$ disown %1                         # Bỏ job khỏi shell (không bị kill khi exit)
```

---

## 3. systemd — "Quản lý" mọi Service

### Ví dụ đời thường:

**systemd** giống **quản lý tòa nhà (Building Manager)**:
- Khi tòa nhà "bật điện" (boot), quản lý **khởi động** tất cả dịch vụ theo đúng thứ tự: điện → nước → thang máy → wifi → camera
- Nếu dịch vụ nào "chết" (crash), quản lý tự động **khởi động lại**
- Quản lý theo dõi **trạng thái** từng dịch vụ

### systemd là gì?

systemd là **init system** — process đầu tiên chạy khi Linux boot (PID = 1). Nó quản lý:
- **Services** (dịch vụ): nginx, mysql, sshd, docker...
- **Timers** (hẹn giờ): thay thế cron
- **Targets** (nhóm services): giống "runlevel" cũ
- **Mounts**, **sockets**, **devices**...

### Unit Files — "Hồ sơ" mỗi service:

Mỗi service được định nghĩa trong file `.service` — thường ở `/etc/systemd/system/` hoặc `/usr/lib/systemd/system/`:

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application Server
After=network.target mysql.service    # Khởi động SAU network và mysql
Wants=mysql.service                    # Muốn mysql chạy (không bắt buộc)

[Service]
Type=simple
User=appuser                          # Chạy dưới user nào
Group=appgroup
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/server --port 8080    # Lệnh khởi động
ExecReload=/bin/kill -HUP $MAINPID             # Lệnh reload
Restart=on-failure                             # Tự restart nếu crash
RestartSec=5                                   # Đợi 5s trước khi restart

[Install]
WantedBy=multi-user.target            # Thuộc target nào (= khi nào bật)
```

### `systemctl` — Điều khiển services:

```bash
# Xem trạng thái
$ sudo systemctl status nginx
● nginx.service - A high performance web server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled)
     Active: active (running) since Mon 2026-06-08 10:00:00 UTC; 5h ago
    Process: 1000 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 1001 (nginx)
      Tasks: 5 (limit: 4915)
     Memory: 12.3M
        CPU: 2.345s
     CGroup: /system.slice/nginx.service
             ├─1001 nginx: master process /usr/sbin/nginx
             ├─1002 nginx: worker process
             └─1003 nginx: worker process

# Điều khiển cơ bản
$ sudo systemctl start nginx      # Khởi động
$ sudo systemctl stop nginx       # Dừng
$ sudo systemctl restart nginx    # Dừng rồi bật lại
$ sudo systemctl reload nginx     # Reload config (không downtime)

# Bật/tắt auto-start khi boot
$ sudo systemctl enable nginx     # Tự động start khi boot
$ sudo systemctl disable nginx    # Không start khi boot
$ sudo systemctl enable --now nginx  # Enable + Start ngay

# Xem tất cả services
$ systemctl list-units --type=service --all
$ systemctl list-units --type=service --state=running   # Chỉ đang chạy

# Xem logs
$ journalctl -u nginx                  # Log của nginx
$ journalctl -u nginx --since today    # Chỉ hôm nay
$ journalctl -u nginx -f               # Follow (realtime)
$ journalctl -u nginx --no-pager -n 50 # 50 dòng cuối
```

### Service States:

| State | Ý nghĩa |
|-------|---------|
| active (running) | Đang chạy bình thường |
| active (exited) | Đã chạy xong và thoát thành công |
| inactive (dead) | Đã dừng |
| failed | Bị crash/lỗi |
| activating | Đang khởi động |
| deactivating | Đang dừng |

### Tạo custom service:

```bash
# 1. Tạo unit file
sudo vim /etc/systemd/system/myapp.service

# 2. Reload systemd (để nó thấy file mới)
sudo systemctl daemon-reload

# 3. Start và enable
sudo systemctl enable --now myapp

# 4. Kiểm tra
sudo systemctl status myapp
journalctl -u myapp -f
```

---

## 4. Package Management — Cài đặt phần mềm

### Ví dụ đời thường:

Package Manager giống **App Store** trên điện thoại:
- Bạn muốn cài ứng dụng → tìm trong Store → bấm Install → tự động tải + cài
- Cần update → Store báo → bấm Update
- Muốn gỡ → bấm Uninstall

### APT (Advanced Package Tool) — Debian/Ubuntu:

```bash
# Cập nhật danh sách packages từ repository
$ sudo apt update              # CHỈ cập nhật danh sách (metadata)

# Nâng cấp packages đã cài
$ sudo apt upgrade             # Upgrade tất cả packages

# Tìm package
$ apt search nginx
$ apt show nginx               # Thông tin chi tiết

# Cài đặt
$ sudo apt install nginx       # Cài nginx
$ sudo apt install -y nginx    # Không hỏi xác nhận

# Gỡ cài đặt
$ sudo apt remove nginx        # Gỡ (giữ config files)
$ sudo apt purge nginx         # Gỡ sạch (xóa cả config)
$ sudo apt autoremove          # Xóa dependencies không dùng

# Xem packages đã cài
$ apt list --installed | grep nginx
```

### YUM/DNF — RHEL/CentOS/Amazon Linux:

```bash
# DNF (mới, thay YUM trên RHEL 8+, Amazon Linux 2023)
$ sudo dnf update              # Cập nhật + upgrade
$ sudo dnf install nginx
$ sudo dnf remove nginx
$ sudo dnf search nginx
$ dnf list installed

# YUM (cũ, Amazon Linux 2)
$ sudo yum update
$ sudo yum install nginx
$ sudo yum remove nginx
$ yum list installed
```

### Repository — "Kho hàng" packages:

```bash
# Xem repositories đang dùng (APT)
$ cat /etc/apt/sources.list
$ ls /etc/apt/sources.list.d/

# Thêm repository mới (ví dụ: Docker)
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
$ echo "deb [signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
$ sudo apt update
$ sudo apt install docker-ce
```

---

## 5. Cron — Lập lịch chạy định kỳ

### Ví dụ đời thường:

Cron giống **báo thức tự động** có thể lặp lại:
- "Mỗi sáng 7h, bật đèn phòng khách" → `0 7 * * * /usr/bin/turn-on-light`
- "Mỗi thứ Hai, gửi báo cáo" → `0 9 * * 1 /usr/bin/send-report`
- "Mỗi 5 phút, check server" → `*/5 * * * * /usr/bin/health-check`

### Cú pháp Crontab:

```
┌───────────── minute (0 - 59)
│ ┌───────────── hour (0 - 23)
│ │ ┌───────────── day of month (1 - 31)
│ │ │ ┌───────────── month (1 - 12)
│ │ │ │ ┌───────────── day of week (0 - 7, 0 & 7 = Sunday)
│ │ │ │ │
* * * * *  command to execute
```

### Ví dụ cron:

| Crontab expression | Ý nghĩa |
|-------------------|---------|
| `0 * * * *` | Mỗi giờ (phút 0) |
| `0 9 * * *` | Mỗi ngày lúc 9:00 |
| `0 9 * * 1-5` | 9:00 các ngày thứ 2-6 (weekdays) |
| `*/5 * * * *` | Mỗi 5 phút |
| `0 0 * * *` | Mỗi nửa đêm (midnight) |
| `0 2 * * 0` | 2:00 sáng mỗi Chủ nhật |
| `0 0 1 * *` | Ngày 1 mỗi tháng, lúc midnight |
| `30 4 1,15 * *` | 4:30 sáng ngày 1 và 15 mỗi tháng |

### Quản lý crontab:

```bash
# Xem crontab hiện tại
$ crontab -l

# Chỉnh sửa crontab
$ crontab -e

# Xóa toàn bộ crontab
$ crontab -r

# Ví dụ thêm vào crontab:
# Backup database mỗi ngày lúc 2:00 AM
0 2 * * * /usr/local/bin/backup-db.sh >> /var/log/backup.log 2>&1

# Dọn log mỗi tuần (Chủ nhật 3:00 AM)
0 3 * * 0 find /var/log -name "*.log" -mtime +30 -delete

# Health check mỗi phút
* * * * * curl -s https://myapp.com/health || echo "APP DOWN" | mail admin@company.com
```

### Cron vs systemd Timers:

systemd cũng có tính năng lập lịch (thay thế cron hiện đại hơn):

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup Timer

[Timer]
OnCalendar=*-*-* 02:00:00    # Mỗi ngày 2:00
Persistent=true                # Chạy bù nếu miss (máy tắt lúc 2:00)

[Install]
WantedBy=timers.target
```

```bash
# Kích hoạt timer
sudo systemctl enable --now backup.timer

# Xem tất cả timers
systemctl list-timers --all
```

---

## 6. Log Management — Theo dõi hoạt động

### Log quan trọng:

| File | Nội dung |
|------|----------|
| `/var/log/syslog` (Ubuntu) | Log tổng hợp hệ thống |
| `/var/log/messages` (RHEL) | Tương đương syslog |
| `/var/log/auth.log` | Đăng nhập, sudo, SSH |
| `/var/log/kern.log` | Kernel messages |
| `/var/log/nginx/access.log` | Nginx HTTP requests |
| `/var/log/nginx/error.log` | Nginx errors |

### journalctl — Đọc systemd journal:

```bash
# Tất cả logs
$ journalctl

# Logs hôm nay
$ journalctl --since today

# Logs trong khoảng thời gian
$ journalctl --since "2026-06-08 10:00" --until "2026-06-08 11:00"

# Logs của service cụ thể
$ journalctl -u nginx -f         # Follow realtime

# Logs theo priority
$ journalctl -p err              # Chỉ errors trở lên
$ journalctl -p warning          # Warning trở lên

# Dung lượng journal
$ journalctl --disk-usage
$ sudo journalctl --vacuum-size=500M    # Giới hạn 500MB
```

---

## 7. AWS Mapping — EC2 User Data và EventBridge

### EC2 User Data — Script chạy khi launch:

**User Data** là script tự động chạy **1 lần** khi EC2 instance khởi động lần đầu — giống "hướng dẫn setup" cho thợ khi bàn giao nhà mới.

```bash
#!/bin/bash
# User Data script — chạy dưới quyền root

# Cập nhật system
yum update -y

# Cài web server
yum install -y httpd

# Tạo trang web
cat > /var/www/html/index.html <<EOF
<html>
<h1>Hello from $(hostname)</h1>
<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>
</html>
EOF

# Start và enable service
systemctl start httpd
systemctl enable httpd
```

**Các lưu ý:**
- Chạy dưới quyền **root** (không cần sudo)
- Chạy **1 lần** khi launch (không phải mỗi lần reboot)
- Log ở `/var/log/cloud-init-output.log`
- Kích thước tối đa: 16 KB
- Phải bắt đầu bằng `#!/bin/bash` (hoặc `#cloud-config` cho YAML format)

### Kiểm tra User Data đã chạy:

```bash
# SSH vào EC2, kiểm tra:
$ cat /var/log/cloud-init-output.log
$ systemctl status httpd
$ curl http://localhost
```

### Amazon EventBridge — "Cron trên Cloud":

EventBridge giống **cron nhưng trên AWS** — lập lịch chạy tác vụ mà KHÔNG cần server:

| Cron (trên EC2) | EventBridge |
|----------------|-------------|
| Cần EC2 chạy 24/7 | Serverless (không cần server) |
| Chết nếu EC2 chết | Managed, highly available |
| Chỉ chạy lệnh trên 1 máy | Trigger Lambda, ECS, SNS, SQS... |
| `*/5 * * * * command` | Schedule expression |

**EventBridge Schedule Expressions:**

```
# Cron format (giống Linux nhưng thêm year và khác chút)
cron(0 9 * * ? *)        # 9:00 UTC mỗi ngày
cron(0 2 ? * MON-FRI *)  # 2:00 UTC weekdays
cron(0/5 * * * ? *)      # Mỗi 5 phút

# Rate format
rate(5 minutes)           # Mỗi 5 phút
rate(1 hour)              # Mỗi giờ
rate(1 day)               # Mỗi ngày
```

**Use cases:**
- Mỗi đêm: Lambda backup DynamoDB → S3
- Mỗi 5 phút: Lambda check website health
- Mỗi tháng: Step Function generate báo cáo
- Trigger EC2 start/stop theo schedule (tiết kiệm chi phí)

### So sánh: Cron on EC2 vs EventBridge + Lambda:

```
# Cách cũ (Cron on EC2):
EC2 (chạy 24/7, $50/month) → crontab → script → work

# Cách mới (EventBridge + Lambda):
EventBridge Rule (free) → Lambda ($0.0000001/invocation) → work
```

---

## 8. Resource Monitoring — Theo dõi tài nguyên

### Memory (`free`):

```bash
$ free -h
              total        used        free      shared  buff/cache   available
Mem:           7.8G        1.8G        4.4G         32M        1.5G        5.7G
Swap:          2.0G          0B        2.0G

# Giải thích:
# total = Tổng RAM vật lý
# used = RAM đang dùng
# free = RAM hoàn toàn trống
# buff/cache = RAM dùng cho buffer/cache (có thể giải phóng)
# available = RAM thực sự available cho apps mới = free + reclaimable cache
```

**Lưu ý:** Linux tích cực dùng RAM trống cho disk cache → `free` thấp không có nghĩa hết RAM. Xem `available` mới chính xác.

### Disk (`df`, `du`, `iostat`):

```bash
# Dung lượng ổ đĩa
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1       8G   3.2G  4.8G  40% /
/dev/xvdf       100G   50G   50G  50% /data

# Dung lượng folder
$ du -sh /var/log/       # Tổng kích thước /var/log
$ du -h --max-depth=1 /var/   # Top-level subdirs

# I/O performance
$ iostat -x 1            # Disk I/O stats mỗi giây
```

### Network (`ss`, `netstat`):

```bash
# Connections đang active
$ ss -tuln                   # TCP/UDP listening ports
$ ss -tunp                   # Kèm process name
$ ss -s                      # Summary statistics

# Network interface stats
$ ip -s link                 # Packets sent/received
```

### Tổng hợp — `vmstat`, `sar`:

```bash
$ vmstat 1 5                 # System stats mỗi giây, 5 lần
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 4521232 123456 1552100  0    0     5    12  200  350  2  1 97  0  0
```

---

## 9. Thực hành: Lab tự làm

### Lab 1: Process management

```bash
# Chạy process background
sleep 300 &
sleep 600 &

# Xem processes
ps aux | grep sleep
jobs

# Kill process
kill %1              # Kill job 1
kill -9 $(pgrep -f "sleep 600")   # Kill theo pattern

# Monitor với top
top -p $(pgrep nginx | tr '\n' ',' | sed 's/,$//')   # top chỉ xem nginx
```

### Lab 2: Tạo và quản lý systemd service

```bash
# Tạo script đơn giản
sudo tee /usr/local/bin/myserver.sh << 'EOF'
#!/bin/bash
while true; do
    echo "$(date): Server running" >> /var/log/myserver.log
    sleep 10
done
EOF
sudo chmod +x /usr/local/bin/myserver.sh

# Tạo service file
sudo tee /etc/systemd/system/myserver.service << 'EOF'
[Unit]
Description=My Test Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myserver.sh
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

# Kích hoạt
sudo systemctl daemon-reload
sudo systemctl enable --now myserver

# Kiểm tra
sudo systemctl status myserver
tail -f /var/log/myserver.log

# Test restart
sudo kill $(pgrep -f myserver.sh)    # Kill → systemd tự restart
sudo systemctl status myserver        # Vẫn running!
```

### Lab 3: Cron job

```bash
# Tạo script backup
sudo tee /usr/local/bin/backup.sh << 'EOF'
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
tar -czf /tmp/backup_${DATE}.tar.gz /var/log/*.log
echo "$(date): Backup completed" >> /var/log/backup-cron.log
EOF
sudo chmod +x /usr/local/bin/backup.sh

# Thêm cron (chạy mỗi 5 phút để test)
crontab -e
# Thêm dòng: */5 * * * * /usr/local/bin/backup.sh

# Verify
crontab -l
# Đợi 5 phút rồi kiểm tra
ls -la /tmp/backup_*
cat /var/log/backup-cron.log
```

### Lab 4: EC2 User Data

1. Launch EC2 instance với User Data:
```bash
#!/bin/bash
yum update -y
amazon-linux-extras install nginx1 -y
systemctl start nginx
systemctl enable nginx
echo "<h1>Deployed at $(date)</h1>" > /usr/share/nginx/html/index.html
```
2. Đợi instance running → truy cập Public IP trên browser
3. SSH vào kiểm tra:
```bash
systemctl status nginx
cat /var/log/cloud-init-output.log
```

---

## 10. Tổng kết

| Khái niệm | Ví dụ đời thường | Lệnh/Service |
|-----------|-----------------|--------------|
| Process | Đầu bếp đang nấu theo công thức | `ps`, `top`, `kill` |
| systemd | Quản lý tòa nhà | `systemctl start/stop/enable` |
| Service | Dịch vụ (điện, nước, wifi) | `.service` unit files |
| Package Manager | App Store | `apt` (Ubuntu), `dnf` (RHEL) |
| Cron | Báo thức lặp lại | `crontab -e` |
| User Data | Hướng dẫn setup nhà mới | EC2 bootstrap script |
| EventBridge | Cron trên cloud (serverless) | Rate/Cron expressions |

---

## Tài liệu tham khảo

1. **systemd documentation** — [https://www.freedesktop.org/wiki/Software/systemd/](https://www.freedesktop.org/wiki/Software/systemd/)
2. **systemd.service man page** — [https://www.freedesktop.org/software/systemd/man/systemd.service.html](https://www.freedesktop.org/software/systemd/man/systemd.service.html)
3. **Linux man pages: ps(1), top(1), kill(1), crontab(5)** — [https://man7.org/linux/man-pages/](https://man7.org/linux/man-pages/)
4. **Debian APT documentation** — [https://wiki.debian.org/Apt](https://wiki.debian.org/Apt)
5. **DNF documentation** — [https://dnf.readthedocs.io/](https://dnf.readthedocs.io/)
6. **AWS EC2 User Data** — [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
7. **AWS EventBridge** — [https://docs.aws.amazon.com/eventbridge/latest/userguide/](https://docs.aws.amazon.com/eventbridge/latest/userguide/)
8. **Brian Ward, "How Linux Works"** — No Starch Press, 3rd Edition.

---

**Bài tiếp theo:** [Phần 9: Bash Scripting — Tự động hóa với shell script](/2026-06-01-bash-scripting/)
