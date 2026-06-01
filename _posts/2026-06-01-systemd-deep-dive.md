---
layout: post
title: "Systemd Deep Dive - Quản Lý Services Linux Hiện Đại"
date: 2026-06-01
categories: [linux]
tags: [systemd, services, timers, journalctl, init]
---

# Systemd Deep Dive - Quản Lý Services Linux Hiện Đại

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Hãy tưởng tượng hệ điều hành là một **khách sạn lớn**:
- **Systemd** = Giám đốc khách sạn (quản lý TẤT CẢ mọi thứ)
- **Service** = Nhân viên (web server, database, SSH...)
- **Timer** = Đồng hồ hẹn giờ ("Dọn phòng lúc 6h sáng mỗi ngày")
- **Socket** = Chuông cửa ("Khi có khách đến → gọi nhân viên lễ tân dậy")
- **Target** = Ca làm việc ("Ca sáng" = multi-user, "Ca đêm" = rescue mode)
- **Dependencies** = Quy trình: "Mở cửa trước, rồi mới bật điện, rồi mới đón khách"

**Systemd** là init system (PID 1) trên hầu hết Linux distros hiện đại — nó quản lý:
- Khởi động hệ thống (boot)
- Services/daemons (start, stop, restart, enable)
- Timers (thay thế cron)
- Logging (journald)
- Mounts, devices, network, and more

---

## 2. Unit Types — Các Loại Đơn Vị Quản Lý

### 2.1 Overview

```
Systemd quản lý mọi thứ qua "units" — mỗi unit có 1 file cấu hình:

Unit Type    │ Extension  │ Mục đích
─────────────┼────────────┼──────────────────────────────
service      │ .service   │ Process/daemon (nginx, mysql)
timer        │ .timer     │ Scheduled task (thay cron)
socket       │ .socket    │ IPC/network socket activation
mount        │ .mount     │ Filesystem mount point
automount    │ .automount │ Auto-mount on first access
target       │ .target    │ Group of units (boot stages)
slice        │ .slice     │ Cgroup resource management
device       │ .device    │ Hardware device
swap         │ .swap      │ Swap space
path         │ .path      │ File/directory monitoring
scope        │ .scope     │ External process groups
```

### 2.2 Service Units (Quan Trọng Nhất)

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Application Server
Documentation=https://myapp.example.com/docs
After=network-online.target postgresql.service
Requires=postgresql.service
Wants=redis.service

[Service]
Type=notify
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
Environment=NODE_ENV=production
EnvironmentFile=/etc/myapp/env
ExecStartPre=/opt/myapp/pre-start.sh
ExecStart=/opt/myapp/bin/server --port 8080
ExecStartPost=/opt/myapp/post-start.sh
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/opt/myapp/bin/graceful-stop
Restart=on-failure
RestartSec=5
TimeoutStartSec=30
TimeoutStopSec=30
WatchdogSec=60

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/myapp /var/log/myapp

# Resource limits
MemoryMax=512M
CPUQuota=200%
TasksMax=100

[Install]
WantedBy=multi-user.target
```

**Service Types:**
```
Type=simple  : ExecStart là main process (default)
Type=forking : Process fork, parent exits (traditional daemons)
Type=oneshot : Process runs, exits, considered "active" after exit
Type=notify  : Process sends "sd_notify" khi ready (RECOMMENDED)
Type=dbus    : Ready khi acquires D-Bus name
Type=idle    : Like simple, but delays until all jobs dispatched
```

### 2.3 Timer Units (Thay Thế Cron)

```ini
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup timer

[Timer]
# Realtime (calendar-based, like cron):
OnCalendar=*-*-* 02:00:00         # Mỗi ngày lúc 2:00 AM
# OnCalendar=Mon..Fri *-*-* 09:00  # Weekdays 9 AM
# OnCalendar=weekly                 # Every Monday 00:00
# OnCalendar=monthly                # First of month 00:00

# Monotonic (relative to events):
# OnBootSec=5min                   # 5 phút sau boot
# OnUnitActiveSec=1h               # Mỗi 1 giờ sau lần chạy trước
# OnUnitInactiveSec=30min          # 30 phút sau khi unit inactive

Persistent=true                    # Chạy ngay nếu missed (machine was off)
RandomizedDelaySec=300             # Random delay 0-5 min (tránh thundering herd)
AccuracySec=1s                     # Timer precision

[Install]
WantedBy=timers.target

# Timer file phải có service file cùng tên:
# backup.timer → triggers backup.service
```

```ini
# /etc/systemd/system/backup.service
[Unit]
Description=Daily backup job

[Service]
Type=oneshot
ExecStart=/opt/scripts/backup.sh
User=backup
StandardOutput=journal
StandardError=journal
```

```bash
# Quản lý timers
systemctl start backup.timer        # Start timer
systemctl enable backup.timer       # Enable on boot
systemctl list-timers               # List all active timers

# NEXT       LEFT         LAST         PASSED       UNIT
# Sun 02:00  3h left      Sat 02:00    21h ago      backup.timer
```

### 2.4 Socket Activation

**Ví dụ đời thường:** Thay vì nhân viên lễ tân đứng chờ 24/7 (service always running), đặt chuông cửa — khi khách nhấn chuông (connection arrives) → gọi nhân viên dậy (start service on demand).

```ini
# /etc/systemd/system/myapp.socket
[Unit]
Description=MyApp Socket

[Socket]
ListenStream=8080             # TCP port 8080
# ListenDatagram=5000         # UDP port 5000
# ListenStream=/run/myapp.sock  # Unix socket
Accept=false                   # false = 1 service cho tất cả connections
                               # true = fork 1 service per connection
BindIPv6Only=both

[Install]
WantedBy=sockets.target
```

```bash
# Socket activation flow:
# 1. systemd listens on port 8080 (service NOT running)
# 2. Client connects to port 8080
# 3. systemd starts myapp.service (passes socket fd)
# 4. myapp handles the connection
# 5. No traffic → myapp can be stopped → saves resources!

# Ưu điểm:
# - Zero-downtime restart (systemd holds socket during restart)
# - On-demand start (save resources when idle)
# - Parallel activation (multiple sockets started simultaneously)
```

---

## 3. Dependencies — Thứ Tự và Phụ Thuộc

### 3.1 Ordering vs Requirement

```
QUAN TRỌNG: Ordering (After/Before) và Requirement (Wants/Requires) là KHÁC NHAU!

Ordering (THỨ TỰ):
- After=B   : "Nếu B cũng start, start tôi SAU B"
- Before=B  : "Nếu B cũng start, start tôi TRƯỚC B"
- CHỈ xác định thứ tự, KHÔNG tự start B!

Requirement (YÊU CẦU):
- Wants=B   : "Hãy start B cùng tôi" (B fail → tôi VẪN start)
- Requires=B: "Hãy start B cùng tôi" (B fail → tôi CŨNG fail!)
- BindsTo=B : "Nếu B stop/fail bất cứ lúc nào → stop tôi luôn" (strongest)

Kết hợp (thường dùng):
After=B + Requires=B  : "Start B trước tôi, nếu B fail thì tôi cũng fail"
After=B + Wants=B     : "Start B trước tôi, nếu B fail tôi vẫn cố start"
```

### 3.2 Ví dụ Dependencies

```ini
# Web app cần database + redis, nên start sau network
[Unit]
After=network-online.target postgresql.service redis.service
Requires=postgresql.service    # App CẦN database (fail nếu DB fail)
Wants=redis.service            # App MUỐN redis (nhưng chạy được without it)
```

### 3.3 Targets — Boot Stages

```
Targets = nhóm units (tương tự "runlevels" cũ)

graphical.target    ← Multi-user + GUI (desktop)
    ↑ Wants
multi-user.target   ← Multi-user, no GUI (servers thường boot đến đây)
    ↑ Wants
basic.target        ← Basic system (filesystems, swap, sockets)
    ↑ Requires
sysinit.target      ← System init (early boot)
    ↑ Requires
local-fs.target     ← Local filesystems mounted

Tương đương runlevels cũ:
runlevel 0 = poweroff.target
runlevel 1 = rescue.target
runlevel 3 = multi-user.target
runlevel 5 = graphical.target
runlevel 6 = reboot.target
```

```bash
# Xem default target
systemctl get-default
# multi-user.target

# Đổi default target
sudo systemctl set-default graphical.target

# Switch target ngay (không cần reboot)
sudo systemctl isolate rescue.target    # Single-user rescue mode
sudo systemctl isolate multi-user.target # Back to normal
```

---

## 4. Systemctl — Quản Lý Services

### 4.1 Commands Cơ Bản

```bash
# Start/Stop/Restart
systemctl start nginx.service
systemctl stop nginx.service
systemctl restart nginx.service
systemctl reload nginx.service       # Reload config (SIGHUP, no downtime)
systemctl reload-or-restart nginx    # Reload nếu support, restart nếu không

# Enable/Disable (auto-start on boot)
systemctl enable nginx.service       # Start on boot
systemctl disable nginx.service      # Don't start on boot
systemctl enable --now nginx         # Enable AND start immediately

# Status
systemctl status nginx.service       # Detailed status + recent logs
systemctl is-active nginx.service    # Just "active" or "inactive"
systemctl is-enabled nginx.service   # "enabled" or "disabled"
systemctl is-failed nginx.service    # Check if failed

# List
systemctl list-units --type=service                # All loaded services
systemctl list-units --type=service --state=running # Running services
systemctl list-unit-files --type=service           # All installed services
systemctl list-timers                              # All timers
```

### 4.2 Troubleshooting

```bash
# Xem tại sao service fail
systemctl status myapp.service       # Shows last few log lines
journalctl -u myapp.service -n 50    # Last 50 log lines
journalctl -u myapp.service -f       # Follow logs (like tail -f)

# Xem dependencies
systemctl list-dependencies nginx.service          # What nginx needs
systemctl list-dependencies --reverse nginx.service # What needs nginx

# Xem unit file location
systemctl show -p FragmentPath nginx.service
# FragmentPath=/lib/systemd/system/nginx.service

# Reload systemd after editing unit files
sudo systemctl daemon-reload         # PHẢI chạy sau khi edit .service file!

# Mask/Unmask (prevent start completely)
sudo systemctl mask dangerous.service    # Cannot be started AT ALL
sudo systemctl unmask dangerous.service  # Remove mask
```

---

## 5. Journalctl — Systemd Logging

### 5.1 Cơ Bản

```bash
# Xem tất cả logs
journalctl

# Logs của service cụ thể
journalctl -u nginx.service
journalctl -u nginx -u postgresql    # Multiple units

# Filter theo thời gian
journalctl --since "2026-06-01 10:00"
journalctl --since "1 hour ago"
journalctl --since today
journalctl --since yesterday --until today

# Follow (real-time)
journalctl -f                        # Follow all
journalctl -f -u nginx               # Follow specific unit

# Output formats
journalctl -o json                   # JSON format
journalctl -o json-pretty            # Pretty JSON
journalctl -o short-iso              # ISO timestamps
journalctl -o verbose                # All metadata fields
```

### 5.2 Filtering Nâng Cao

```bash
# Filter theo priority (severity)
journalctl -p err                    # Only errors and above
journalctl -p warning                # Warnings and above
# Priority levels: emerg(0) alert(1) crit(2) err(3) warning(4) notice(5) info(6) debug(7)

# Filter theo boot
journalctl -b                        # Current boot only
journalctl -b -1                     # Previous boot
journalctl --list-boots              # List all boots

# Filter theo PID/UID
journalctl _PID=1234
journalctl _UID=1000
journalctl _COMM=nginx               # By command name

# Kernel messages only
journalctl -k                        # Same as dmesg but structured

# Disk usage
journalctl --disk-usage
# Archived and active journals take up 2.5G in the file system.

# Cleanup
sudo journalctl --vacuum-size=500M   # Shrink to 500MB
sudo journalctl --vacuum-time=7d     # Keep only last 7 days
```

### 5.3 Persistent Journal Configuration

```ini
# /etc/systemd/journald.conf
[Journal]
Storage=persistent          # persistent (survive reboot) vs volatile (/run only)
SystemMaxUse=1G            # Max disk usage
SystemMaxFileSize=100M     # Max per-file size
MaxRetentionSec=30d        # Max age
Compress=yes               # Compress archived journals
RateLimitIntervalSec=30    # Rate limit: max messages per interval
RateLimitBurst=10000       # Max messages before rate limiting kicks in
```

---

## 6. Custom Unit Files — Tạo Service Riêng

### 6.1 Ví Dụ: Node.js Application

```ini
# /etc/systemd/system/nodeapp.service
[Unit]
Description=Node.js Production App
Documentation=https://github.com/myorg/nodeapp
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=nodeapp
Group=nodeapp
WorkingDirectory=/opt/nodeapp
Environment=NODE_ENV=production PORT=3000
ExecStart=/usr/bin/node /opt/nodeapp/server.js
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal
SyslogIdentifier=nodeapp

# Security
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/nodeapp

# Resources
MemoryMax=1G
CPUQuota=150%
TasksMax=200

[Install]
WantedBy=multi-user.target
```

### 6.2 Ví Dụ: Oneshot Script + Timer

```ini
# /etc/systemd/system/cleanup.service
[Unit]
Description=Cleanup temporary files older than 7 days

[Service]
Type=oneshot
ExecStart=/usr/bin/find /tmp -type f -mtime +7 -delete
ExecStart=/usr/bin/find /var/log -name "*.gz" -mtime +30 -delete
```

```ini
# /etc/systemd/system/cleanup.timer
[Unit]
Description=Run cleanup daily at 3AM

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

### 6.3 Deploy Steps

```bash
# 1. Create unit file
sudo vi /etc/systemd/system/myapp.service

# 2. Reload systemd
sudo systemctl daemon-reload

# 3. Start and test
sudo systemctl start myapp
sudo systemctl status myapp
journalctl -u myapp -f

# 4. Enable on boot
sudo systemctl enable myapp

# 5. Verify
sudo systemctl is-enabled myapp    # "enabled"
sudo reboot                        # Test auto-start after boot
```

---

## 7. Security Hardening With Systemd

### 7.1 Sandboxing Options

```ini
[Service]
# Filesystem protection
ProtectSystem=strict        # / is read-only (except /dev, /proc, /sys)
ProtectHome=true            # /home, /root, /run/user = inaccessible
PrivateTmp=true             # Private /tmp and /var/tmp
ReadWritePaths=/var/lib/myapp  # Whitelist writable paths
ReadOnlyPaths=/etc/myapp       # Explicit read-only

# Privilege restriction
NoNewPrivileges=true        # Cannot gain new privileges (no SUID exploit)
PrivateDevices=true         # No access to physical devices
ProtectKernelTunables=true  # /proc/sys, /sys read-only
ProtectKernelModules=true   # Cannot load kernel modules
ProtectControlGroups=true   # Cannot modify cgroups

# Network restriction
PrivateNetwork=true         # Isolated network (no network access!)
RestrictAddressFamilies=AF_INET AF_INET6  # Only IPv4/IPv6 (no Unix sockets)
IPAddressDeny=any           # Block all IPs
IPAddressAllow=10.0.0.0/8   # Allow only private network

# System call filtering
SystemCallFilter=@system-service  # Only allow common service syscalls
SystemCallArchitectures=native    # Only native arch (block 32-bit on 64-bit)

# Capabilities
CapabilityBoundingSet=CAP_NET_BIND_SERVICE  # Only allow binding low ports
AmbientCapabilities=CAP_NET_BIND_SERVICE
```

### 7.2 Security Analysis

```bash
# Analyze security of a unit
systemd-analyze security nginx.service

# Output (scores 0-10, lower = more secure):
# OVERALL EXPOSURE: 4.2 OK
# PrivateNetwork=         0.5 UNSAFE
# ProtectSystem=strict    0.0 OK
# NoNewPrivileges=yes     0.0 OK
# ...

# Optimize based on recommendations!
```

---

## 8. Advanced Features

### 8.1 Template Units (Instance Units)

```ini
# /etc/systemd/system/app@.service  (@ = template)
[Unit]
Description=App instance %i
After=network.target

[Service]
Type=simple
ExecStart=/opt/app/bin/server --instance=%i --port=%i
User=app

[Install]
WantedBy=multi-user.target
```

```bash
# Start multiple instances:
systemctl start app@8080.service
systemctl start app@8081.service
systemctl start app@8082.service
# %i = instance name (8080, 8081, 8082)
```

### 8.2 Slice Units (Resource Grouping)

```ini
# /etc/systemd/system/production.slice
[Unit]
Description=Production services slice

[Slice]
MemoryMax=8G
CPUQuota=400%
```

```ini
# Service belongs to slice:
[Service]
Slice=production.slice
```

### 8.3 Watchdog

```ini
[Service]
Type=notify
WatchdogSec=30          # Service must "ping" watchdog every 30s
                        # If no ping → systemd considers it failed → restart!

# Application code must call:
# sd_notify(0, "WATCHDOG=1") periodically
```

---

## 9. Systemd vs Alternatives

### 9.1 Comparison

| Feature | Systemd | SysVinit | Upstart |
|---------|---------|----------|---------|
| Parallelization | ✅ Full | ❌ Sequential | ✅ Partial |
| Socket activation | ✅ | ❌ | ❌ |
| On-demand start | ✅ | ❌ | ✅ |
| Cgroup integration | ✅ Native | ❌ | ❌ |
| Journal logging | ✅ Binary | Text files | Text files |
| Boot speed | Fast | Slow | Medium |
| Complexity | High | Low | Medium |
| Adoption | 95%+ distros | Legacy | Deprecated |

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 Cheat Sheet

```bash
# Service management
systemctl start|stop|restart|reload|status <unit>
systemctl enable|disable <unit>
systemctl daemon-reload          # After editing unit files!

# Logs
journalctl -u <unit> -f         # Follow unit logs
journalctl -p err --since today # Errors today
journalctl --vacuum-size=500M   # Cleanup

# Timer management  
systemctl list-timers
systemctl enable --now mytimer.timer

# Analysis
systemd-analyze                  # Boot time
systemd-analyze blame            # Slowest units
systemd-analyze critical-chain   # Critical path
systemd-analyze security <unit>  # Security score
```

### 10.2 Key Takeaways

1. **Systemd = PID 1** — manages everything (services, timers, mounts, devices)
2. **Unit file = INI-format config** — declarative, easy to read and write
3. **After/Before = ordering**, **Wants/Requires = dependency** (different concepts!)
4. **Timer units > cron** — persistent, randomized delay, calendar expressions
5. **Socket activation** = start service on-demand (save resources)
6. **journalctl** = structured logging with powerful filtering
7. **Security hardening** built-in — ProtectSystem, NoNewPrivileges, SystemCallFilter
8. **daemon-reload** after ANY unit file change!

### 10.3 Tài Liệu Tham Khảo

- systemd documentation: https://www.freedesktop.org/wiki/Software/systemd/
- man pages: systemd(1), systemctl(1), journalctl(1), systemd.unit(5), systemd.service(5), systemd.timer(5)
- Arch Wiki: systemd (excellent comprehensive guide)
- Red Hat System Administrator's Guide: Managing Services with systemd
- "systemd for Administrators" blog series by Lennart Poettering
- Fedora documentation: systemd

---

*Bài viết tiếp theo: Linux Networking Tools — ip, iptables, nftables, namespaces*
