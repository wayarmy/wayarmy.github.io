---
layout: post
title: "Linux Security Hardening - Tăng cường bảo mật Linux"
date: 2026-06-01
categories: [linux]
tags: [selinux, apparmor, fail2ban, auditd, cis-benchmark, hardening]
---

## Mục lục
1. [Góc nhìn tổng quan - Xây pháo đài](#goc-nhin-tong-quan)
2. [SELinux - Kiểm soát truy cập bắt buộc](#selinux)
3. [AppArmor - Áo giáp cho ứng dụng](#apparmor)
4. [fail2ban - Cấm cửa kẻ xâm nhập](#fail2ban)
5. [auditd - Camera giám sát hệ thống](#auditd)
6. [CIS Benchmarks - Bộ tiêu chuẩn an ninh](#cis)
7. [SSH Hardening - Khóa chặt cửa chính](#ssh-hardening)
8. [Unattended Upgrades - Tự động vá lỗi](#unattended-upgrades)
9. [Kernel hardening và Filesystem security](#kernel-hardening)
10. [Tổng kết và checklist hardening](#tong-ket)

---

## 1. Góc nhìn tổng quan - Xây pháo đài {#goc-nhin-tong-quan}

### Ví dụ đời thường

Bảo mật Linux giống **xây pháo đài nhiều lớp**:

- **SELinux/AppArmor** = quy tắc nội bộ - mỗi người chỉ được vào phòng mình (Mandatory Access Control)
- **fail2ban** = lính gác cổng - ai gõ cửa sai 5 lần thì bị cấm cửa
- **auditd** = camera giám sát - ghi lại mọi hoạt động để xem lại
- **CIS Benchmarks** = bản thiết kế pháo đài tiêu chuẩn quân đội
- **SSH hardening** = khóa cửa chính bằng khóa vân tay thay vì chìa đơn giản
- **Unattended upgrades** = tự động sửa tường khi phát hiện vết nứt
- **Firewall** = tường thành bao quanh (iptables/nftables)

### Defense in Depth - Phòng thủ nhiều lớp

```
┌─────────────────────────────────────────────────┐
│ Layer 1: Network (Firewall, Security Groups)     │
├─────────────────────────────────────────────────┤
│ Layer 2: Host (SSH hardening, fail2ban)          │
├─────────────────────────────────────────────────┤
│ Layer 3: OS (Patching, kernel hardening)         │
├─────────────────────────────────────────────────┤
│ Layer 4: Application (SELinux/AppArmor, sandboxing)│
├─────────────────────────────────────────────────┤
│ Layer 5: Data (Encryption, access control)       │
├─────────────────────────────────────────────────┤
│ Layer 6: Monitoring (auditd, alerting)           │
└─────────────────────────────────────────────────┘

Nguyên tắc: Nếu 1 layer bị phá, layers khác vẫn bảo vệ.
```

---

## 2. SELinux - Kiểm soát truy cập bắt buộc {#selinux}

### SELinux là gì?

**SELinux** (Security-Enhanced Linux) là hệ thống MAC (Mandatory Access Control) do NSA phát triển. Nó bổ sung thêm lớp kiểm soát truy cập TRÊN cả permission truyền thống (DAC - chmod/chown).

### DAC vs MAC

```
DAC (chmod/chown - truyền thống):
- User sở hữu file → user quyết định ai access
- Root bypass tất cả → root bị compromise = game over
- Ví dụ: Nhà bạn, bạn cho ai vào tùy ý

MAC (SELinux):
- SYSTEM quyết định ai access gì, kể cả root!
- Mỗi process và file có "security label"
- Policy quyết định label nào được tương tác với label nào
- Ví dụ: Bệnh viện - dù bạn là giám đốc, bạn không vào phòng mổ nếu không có badge "surgeon"
```

### SELinux modes

```bash
# Kiểm tra mode hiện tại
getenforce
# Enforcing = bật đầy đủ (chặn + log)
# Permissive = chỉ log, không chặn (debug mode)
# Disabled = tắt hoàn toàn

# Chuyển mode tạm thời
setenforce 0    # → Permissive
setenforce 1    # → Enforcing

# Chuyển mode vĩnh viễn
# /etc/selinux/config
SELINUX=enforcing
SELINUXTYPE=targeted
```

### Security Context (Labels)

```bash
# Xem labels
ls -Z /var/www/html/
# system_u:object_r:httpd_sys_content_t:s0 index.html
#    user  :role    :type                :level

ps -eZ | grep httpd
# system_u:system_r:httpd_t:s0  /usr/sbin/httpd

# Format: user:role:type:level
# QUAN TRỌNG NHẤT: TYPE (type enforcement)
# httpd_t (process) → có thể đọc httpd_sys_content_t (file)
# httpd_t → KHÔNG THỂ đọc user_home_t (user files)
```

### Troubleshooting SELinux

```bash
# Xem audit log khi bị deny
ausearch -m AVC -ts recent
# type=AVC msg=audit(...): avc: denied { read } for pid=1234
# comm="httpd" name="config.php" scontext=httpd_t tcontext=user_home_t

# Dùng audit2why để hiểu tại sao bị deny
ausearch -m AVC -ts recent | audit2why
# → Gợi ý: setsebool httpd_read_user_content on

# Dùng audit2allow để tạo policy module
ausearch -m AVC -ts recent | audit2allow -M my_module
semodule -i my_module.pp

# Sửa context sai (ví dụ: file bị sai label)
restorecon -Rv /var/www/html/

# Set context thủ công
chcon -t httpd_sys_content_t /var/www/html/newfile.html

# Booleans (toggle features)
getsebool -a | grep httpd
setsebool -P httpd_can_network_connect on
setsebool -P httpd_can_sendmail on
```

---

## 3. AppArmor - Áo giáp cho ứng dụng {#apparmor}

### AppArmor là gì?

**AppArmor** (Application Armor) là MAC system thay thế cho SELinux, mặc định trên Ubuntu/SUSE. Nó hoạt động theo **path-based** (dựa trên đường dẫn file) thay vì label-based như SELinux.

### So sánh SELinux vs AppArmor

```
┌────────────────┬───────────────────┬──────────────────┐
│ Aspect         │ SELinux           │ AppArmor         │
├────────────────┼───────────────────┼──────────────────┤
│ Approach       │ Label-based       │ Path-based       │
│ Default on     │ RHEL/Fedora/CentOS│ Ubuntu/SUSE      │
│ Complexity     │ Steep learning    │ Easier           │
│ Granularity    │ Very fine-grained │ Good enough      │
│ Filesystem     │ Supports labeling │ Path rules       │
│ Setup          │ Complex policies  │ Simple profiles  │
│ Performance    │ Minimal overhead  │ Minimal overhead │
└────────────────┴───────────────────┴──────────────────┘
```

### AppArmor profiles

```bash
# Xem status
aa-status
# 45 profiles are loaded.
# 40 profiles are in enforce mode.
# 5 profiles are in complain mode.

# Modes:
# enforce  = chặn + log vi phạm
# complain = chỉ log, không chặn (training mode)
# disable  = tắt profile

# Chuyển mode
aa-enforce /etc/apparmor.d/usr.sbin.nginx
aa-complain /etc/apparmor.d/usr.sbin.nginx
aa-disable /etc/apparmor.d/usr.sbin.nginx
```

### Viết AppArmor profile

```bash
# /etc/apparmor.d/usr.local.bin.myapp
#include <tunables/global>

/usr/local/bin/myapp {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # File access
  /etc/myapp/config.yaml r,      # Read config
  /var/log/myapp/*.log w,        # Write logs
  /var/lib/myapp/** rw,          # Read/write data dir
  /tmp/myapp-* rw,               # Temp files

  # Network access
  network tcp,                    # Allow TCP
  network udp,                    # Allow UDP

  # Deny dangerous operations
  deny /etc/shadow r,
  deny /root/** rwx,

  # Capabilities
  capability net_bind_service,    # Bind port < 1024
}

# Reload profiles
apparmor_parser -r /etc/apparmor.d/usr.local.bin.myapp

# Generate profile automatically (training mode)
aa-genprof /usr/local/bin/myapp
# → Chạy app bình thường → AppArmor ghi lại accesses → tạo profile
```

---

## 4. fail2ban - Cấm cửa kẻ xâm nhập {#fail2ban}

### fail2ban là gì?

**fail2ban** đọc log files, phát hiện pattern đáng ngờ (ví dụ: login sai nhiều lần), rồi tự động ban IP bằng firewall rules.

Giống **lính gác**: ai gõ cửa sai password 5 lần → bị cấm cửa 10 phút.

### Cấu hình

```bash
# /etc/fail2ban/jail.local (override jail.conf)
[DEFAULT]
bantime = 3600            # Ban 1 giờ
findtime = 600            # Trong khoảng 10 phút
maxretry = 5              # 5 lần sai
banaction = iptables-multiport
backend = systemd         # Đọc từ journald

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3              # SSH chỉ cho 3 lần

[nginx-http-auth]
enabled = true
port = http,https
filter = nginx-http-auth
logpath = /var/log/nginx/error.log

[nginx-limit-req]
enabled = true
port = http,https
filter = nginx-limit-req
logpath = /var/log/nginx/error.log
maxretry = 10
```

### Quản lý fail2ban

```bash
# Status tổng quan
fail2ban-client status
fail2ban-client status sshd

# Kiểm tra IP có bị ban không
fail2ban-client get sshd banned

# Unban IP
fail2ban-client set sshd unbanip 192.168.1.100

# Test filter (debug regex matching)
fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf

# Recidive jail (ban lâu hơn nếu bị ban nhiều lần)
[recidive]
enabled = true
bantime = 604800          # 1 tuần
findtime = 86400          # Trong 24h
maxretry = 3              # Bị ban 3 lần → ban 1 tuần
```

---

## 5. auditd - Camera giám sát hệ thống {#auditd}

### auditd là gì?

**auditd** (Linux Audit Daemon) ghi lại các sự kiện bảo mật: ai truy cập file gì, ai chạy command gì, ai thay đổi permission. Cần thiết cho compliance (PCI-DSS, HIPAA, SOC2).

### Cấu hình audit rules

```bash
# /etc/audit/rules.d/audit.rules

# Theo dõi thay đổi file quan trọng
-w /etc/passwd -p wa -k identity
-w /etc/shadow -p wa -k identity
-w /etc/sudoers -p wa -k sudo_changes
-w /etc/ssh/sshd_config -p wa -k sshd_config

# Theo dõi commands (execve)
-a always,exit -F arch=b64 -S execve -F uid=0 -k root_commands

# Theo dõi network connections
-a always,exit -F arch=b64 -S connect -S accept -k network

# Theo dõi file permission changes
-a always,exit -F arch=b64 -S chmod -S fchmod -k permission_change

# Theo dõi mount/unmount
-a always,exit -F arch=b64 -S mount -S umount2 -k mount_ops

# Giải thích syntax:
# -w path    : Watch file/directory
# -p rwxa    : Permission filter (read, write, execute, attribute)
# -k tag     : Key/tag cho filtering
# -a action,list : Add rule to list
# -F field=value : Filter
# -S syscall : System call to monitor
```

### Tìm kiếm audit logs

```bash
# Tìm theo key
ausearch -k identity
ausearch -k sudo_changes --start today

# Tìm theo user
ausearch -ua 1000              # UID 1000
ausearch -ua root

# Tìm theo file
ausearch -f /etc/passwd

# Tìm theo thời gian
ausearch --start "01/15/2024" --end "01/16/2024"

# Report đẹp
aureport                        # Summary report
aureport --auth                 # Authentication report
aureport --login                # Login report
aureport --file --summary       # File access summary
aureport --executable --summary # Executed commands
```

---

## 6. CIS Benchmarks - Bộ tiêu chuẩn an ninh {#cis}

### CIS Benchmarks là gì?

**CIS** (Center for Internet Security) Benchmarks là bộ best practices chuẩn công nghiệp cho security hardening. Chúng cung cấp checklist chi tiết, scored/unscored, cho từng OS.

### Các category chính (CIS Ubuntu 22.04 Benchmark)

```
1. Initial Setup
   - Filesystem configuration
   - Software updates
   - Bootloader security

2. Services
   - Disable unnecessary services
   - Configure time sync (chrony/systemd-timesyncd)
   - Remove X Window System (servers)

3. Network Configuration
   - Disable unused network protocols
   - Firewall configuration
   - TCP Wrappers

4. Logging and Auditing
   - Configure auditd
   - Configure rsyslog/journald
   - Log file permissions

5. Access, Authentication, Authorization
   - Cron access
   - SSH configuration
   - PAM configuration
   - Password policies
   - User account settings

6. System Maintenance
   - File permissions
   - User/group settings
```

### Áp dụng CIS benchmarks

```bash
# Dùng OpenSCAP để scan compliance
sudo apt install libopenscap8 ssg-base ssg-debderived

# Scan
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis_level1_server \
  --results results.xml \
  --report report.html \
  /usr/share/xml/scap/ssg/content/ssg-ubuntu2204-xccdf.xml

# Ubuntu USG (Ubuntu Security Guide)
sudo apt install ubuntu-advantage-tools
sudo ua enable usg
sudo usg audit cis_level1_server

# Ví dụ các CIS items quan trọng:
# - Ensure permissions on /etc/passwd are 644
chmod 644 /etc/passwd

# - Ensure SSH PermitRootLogin is disabled
sed -i 's/^PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config

# - Ensure password expiration is 365 days or less
sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS 365/' /etc/login.defs
```

---

## 7. SSH Hardening - Khóa chặt cửa chính {#ssh-hardening}

### SSH hardening checklist

```bash
# /etc/ssh/sshd_config - Hardened

# === Authentication ===
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
MaxAuthTries 3
MaxSessions 3
PermitEmptyPasswords no
ChallengeResponseAuthentication no

# === Protocol & Crypto ===
Protocol 2
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512,rsa-sha2-256

# === Access Control ===
AllowUsers deploy admin
AllowGroups ssh-users
DenyUsers root
LoginGraceTime 30

# === Forwarding & Features ===
X11Forwarding no
AllowTcpForwarding no          # (yes nếu cần tunneling)
AllowAgentForwarding no
PermitTunnel no
GatewayPorts no

# === Logging ===
LogLevel VERBOSE               # Log key fingerprint
SyslogFacility AUTH

# === Misc ===
ClientAliveInterval 300
ClientAliveCountMax 2
UsePAM yes
Banner /etc/ssh/banner.txt
```

### Thêm 2FA cho SSH

```bash
# Install Google Authenticator PAM
sudo apt install libpam-google-authenticator

# Setup per user
google-authenticator

# /etc/pam.d/sshd - Thêm dòng:
auth required pam_google_authenticator.so

# /etc/ssh/sshd_config:
AuthenticationMethods publickey,keyboard-interactive
ChallengeResponseAuthentication yes

# Kết quả: User cần cả SSH key + TOTP code
```

---

## 8. Unattended Upgrades - Tự động vá lỗi {#unattended-upgrades}

### Tại sao cần auto-patching?

```
Thực tế đau lòng:
- Security patch phát hành → 90% servers chưa patch sau 30 ngày
- Equifax breach (2017): Chưa patch Apache Struts 2 tháng sau CVE
- WannaCry (2017): EternalBlue patch có từ tháng 3, attack tháng 5

Tự động patch security = giảm 90% attack surface
```

### Cấu hình trên Ubuntu/Debian

```bash
# Install
sudo apt install unattended-upgrades apt-listchanges

# Enable
sudo dpkg-reconfigure -plow unattended-upgrades

# /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
    "${distro_id}ESMApps:${distro_codename}-apps-security";
};

Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";

# Email notification
Unattended-Upgrade::Mail "admin@example.com";
Unattended-Upgrade::MailReport "on-change";

# Blacklist packages (don't auto-upgrade)
Unattended-Upgrade::Package-Blacklist {
    "mysql-server";
    "postgresql";
};
```

---

## 9. Kernel hardening và Filesystem security {#kernel-hardening}

### Kernel sysctl hardening

```bash
# /etc/sysctl.d/99-security.conf

# === Network Security ===
# Disable IP forwarding (unless router/gateway)
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0

# Disable ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0

# Enable SYN cookies (SYN flood protection)
net.ipv4.tcp_syncookies = 1

# Log Martian packets
net.ipv4.conf.all.log_martians = 1

# Ignore ICMP broadcasts
net.ipv4.icmp_echo_ignore_broadcasts = 1

# === Kernel Security ===
# Restrict dmesg
kernel.dmesg_restrict = 1

# Restrict kernel pointer exposure
kernel.kptr_restrict = 2

# Disable SysRq key
kernel.sysrq = 0

# Enable ASLR (Address Space Layout Randomization)
kernel.randomize_va_space = 2

# Restrict core dumps
fs.suid_dumpable = 0

# Restrict unprivileged user namespaces
kernel.unprivileged_userns_clone = 0

# Restrict BPF
kernel.unprivileged_bpf_disabled = 1
net.core.bpf_jit_harden = 2
```

### Filesystem hardening

```bash
# /etc/fstab - Mount options

# /tmp with noexec, nosuid, nodev
tmpfs    /tmp     tmpfs   defaults,noexec,nosuid,nodev,size=2G  0 0

# /var/tmp
/tmp     /var/tmp none    bind                                   0 0

# /dev/shm - restrict
none     /dev/shm tmpfs   defaults,noexec,nosuid,nodev          0 0

# Separate /var/log (prevent log filling root)
/dev/sda3 /var/log ext4  defaults,nosuid,nodev,noexec           0 2

# File permissions
chmod 700 /root
chmod 600 /etc/shadow /etc/gshadow
chmod 644 /etc/passwd /etc/group
chmod 600 /etc/crontab
chmod 700 /etc/cron.d /etc/cron.daily /etc/cron.hourly

# Find SUID/SGID files (potential privilege escalation)
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# Remove unnecessary SUID
chmod u-s /usr/bin/chage
chmod u-s /usr/bin/gpasswd
```

---

## 10. Tổng kết và checklist hardening {#tong-ket}

### Quick Hardening Checklist

```bash
# 1. System Updates
apt update && apt upgrade -y
apt install unattended-upgrades && dpkg-reconfigure -plow unattended-upgrades

# 2. User Security
# Disable root login, enforce strong passwords
passwd -l root
sed -i 's/^PASS_MIN_LEN.*/PASS_MIN_LEN 12/' /etc/login.defs

# 3. SSH Hardening
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
# Apply hardened config (see section 7)
systemctl restart sshd

# 4. Firewall
ufw default deny incoming
ufw default allow outgoing
ufw allow 22/tcp
ufw enable

# 5. fail2ban
apt install fail2ban
systemctl enable fail2ban

# 6. Auditing
apt install auditd
auditctl -e 1
# Add rules from section 5

# 7. Remove unnecessary packages/services
apt remove telnet rsh-client
systemctl disable avahi-daemon cups bluetooth

# 8. MAC (AppArmor or SELinux)
aa-enforce /etc/apparmor.d/*

# 9. Kernel hardening
# Apply sysctl from section 9

# 10. Verify
lynis audit system        # Comprehensive audit tool
```

### Tài liệu tham khảo

| Tài liệu | Mô tả |
|-----------|--------|
| CIS Benchmarks (cisecurity.org) | Tiêu chuẩn hardening chính thức |
| NSA/CISA Hardening Guides | Hướng dẫn từ cơ quan an ninh |
| NIST SP 800-123 | Guide to General Server Security |
| Red Hat SELinux Guide | SELinux documentation |
| Ubuntu AppArmor Wiki | AppArmor documentation |
| Lynis (cisofy.com) | Open-source security auditing tool |
| Mozilla Server Security Guidelines | Web server hardening |

---

*Bài viết tiếp theo: [Docker Networking Deep Dive](/2026/08/12/docker-networking-deep-dive/) - Mạng Docker chuyên sâu*
