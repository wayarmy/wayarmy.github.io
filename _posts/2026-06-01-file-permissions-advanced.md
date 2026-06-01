---
layout: post
title: "File Permissions Advanced Deep Dive - SUID, SGID, ACLs, Capabilities"
date: 2026-06-01
categories: [linux]
tags: [permissions, suid, sgid, acl, capabilities, security]
---

# File Permissions Advanced Deep Dive - SUID, SGID, ACLs, Capabilities

## 1. Giới Thiệu Bằng Hình Ảnh Đời Thường

Bạn đã biết quyền cơ bản: Read (đọc), Write (ghi), Execute (chạy) cho Owner, Group, Others. Nhưng thực tế phức tạp hơn:

**Ví dụ đời thường:**
- **SUID:** Bạn (nhân viên) dùng thẻ của GIÁM ĐỐC để mở cửa phòng chủ tịch — trong lúc dùng thẻ, bạn có quyền của giám đốc
- **SGID trên directory:** Phòng dự án — bất kỳ ai tạo file trong phòng, file đó thuộc về NHÓM dự án (không phải nhóm cá nhân)
- **Sticky Bit:** Bảng tin công cộng — ai cũng dán được giấy lên, nhưng CHỈ người dán mới được gỡ giấy CỦA MÌNH
- **ACL:** Khóa cửa thông minh — ngoài khóa chính (owner), bạn cấp quyền riêng cho từng người cụ thể
- **Capabilities:** Thay vì cho chìa khóa MASTER (root), cho mỗi người chỉ đúng chìa khóa họ cần (bind port 80 nhưng không thể đọc file người khác)

---

## 2. Ôn Lại Permissions Cơ Bản

### 2.1 Cấu Trúc Permission

```bash
$ ls -l /etc/passwd
-rw-r--r-- 1 root root 2847 Jun 1 10:00 /etc/passwd
│├──┤├──┤├──┤
│ │   │   │
│ │   │   └── Others: r-- (read only)
│ │   └────── Group:  r-- (read only)  
│ └────────── Owner:  rw- (read + write)
└──────────── Type:   - (regular file)

File types:
- = regular file
d = directory
l = symbolic link
c = character device
b = block device
p = named pipe
s = socket
```

### 2.2 Octal Notation

```
Permission bits:
r (read)    = 4
w (write)   = 2
x (execute) = 1

Ví dụ: rwxr-xr-- = 754
Owner: rwx = 4+2+1 = 7
Group: r-x = 4+0+1 = 5
Others: r-- = 4+0+0 = 4

chmod 755 file  = rwxr-xr-x
chmod 644 file  = rw-r--r--
chmod 600 file  = rw-------
chmod 777 file  = rwxrwxrwx (NGUY HIỂM! Ai cũng làm được mọi thứ)
```

---

## 3. SUID (Set User ID) — Chạy Với Quyền Của Owner

### 3.1 SUID Là Gì?

**Ví dụ đời thường:** Lệnh `passwd` cho phép user THƯỜNG đổi password của mình. Nhưng password lưu trong `/etc/shadow` mà chỉ root mới đọc/ghi được. Vậy user thường làm sao đổi password?

**Giải pháp:** File `/usr/bin/passwd` có **SUID bit** → khi BẤT KỲ AI chạy `passwd`, process chạy với quyền của OWNER (root), không phải quyền của user đang chạy.

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 Jun 1 /usr/bin/passwd
   ↑
   s thay vì x = SUID enabled!

# Khi user "alice" chạy passwd:
# Real UID: alice (1000) — ai thực sự chạy
# Effective UID: root (0) — quyền thực thi = root!
# → Process có thể ghi vào /etc/shadow
```

### 3.2 Cách Set/Remove SUID

```bash
# Set SUID
chmod u+s /path/to/file
chmod 4755 /path/to/file    # 4 = SUID bit
#     ↑ SUID

# Remove SUID
chmod u-s /path/to/file
chmod 0755 /path/to/file

# Kiểm tra SUID
ls -l /path/to/file
# -rwsr-xr-x → SUID enabled (lowercase 's' = execute + suid)
# -rwSr-xr-x → SUID set but NO execute (uppercase 'S' = suid without exec, useless!)
```

### 3.3 Security Risk — SUID Nguy Hiểm!

```bash
# TÌM tất cả SUID files trên hệ thống (security audit!)
find / -perm -4000 -type f 2>/dev/null

# SUID files hợp lệ (cần cho hệ thống):
/usr/bin/passwd     # Đổi password
/usr/bin/sudo       # Chạy lệnh với quyền root
/usr/bin/su         # Switch user
/usr/bin/ping       # Send ICMP (cần raw socket) — nhiều distro mới dùng capabilities thay
/usr/bin/mount      # Mount filesystems
/usr/bin/chfn       # Change user info

# SUID files NGUY HIỂM (nếu có vulnerability):
# Attacker tìm SUID binary → khai thác buffer overflow → có quyền root!
# → Minimze SUID files! Dùng capabilities thay thế!

# Nguyên tắc: SUID = "tất cả hoặc không gì"
# Cho root access = cho TẤT CẢ quyền root
# → Nguy hiểm! Capabilities là giải pháp tốt hơn (xem section 7)
```

---

## 4. SGID (Set Group ID) — Chạy Với Quyền Của Group

### 4.1 SGID Trên Executable Files

```bash
# Tương tự SUID nhưng cho GROUP:
# Process chạy với Effective GID = group owner of file

$ ls -l /usr/bin/wall
-rwxr-sr-x 1 root tty 19968 Jun 1 /usr/bin/wall
       ↑
       s thay vì x = SGID enabled!

# Khi user chạy wall:
# Effective GID = tty → có thể ghi vào terminals (owned by group tty)
```

### 4.2 SGID Trên Directories (QUAN TRỌNG HƠN!)

**Ví dụ đời thường:** Phòng dự án team "developers" — bất kỳ ai tạo file trong phòng, file tự động thuộc group "developers" (không phải group cá nhân). Mọi người trong team đều đọc/ghi được files của nhau.

```bash
# Scenario: Team collaboration directory
# Không SGID:
$ mkdir /project
$ chgrp developers /project
$ chmod 775 /project

# Alice (primary group: alice) tạo file:
$ touch /project/report.txt
$ ls -l /project/report.txt
-rw-r--r-- 1 alice alice 0 Jun 1 report.txt
#                   ↑ group = alice (KHÔNG phải developers!)
# → Bob (group developers) KHÔNG đọc được!

# CÓ SGID:
$ chmod g+s /project
$ chmod 2775 /project    # 2 = SGID
$ ls -ld /project
drwxrwsr-x 2 root developers 4096 Jun 1 /project
       ↑ SGID on directory

# Alice tạo file:
$ touch /project/report.txt  
$ ls -l /project/report.txt
-rw-r--r-- 1 alice developers 0 Jun 1 report.txt
#                   ↑ group = developers (inherited from directory!)
# → Bob (group developers) CŨNG đọc được!

# Subdirectories cũng inherit SGID:
$ mkdir /project/subdir
$ ls -ld /project/subdir
drwxrwsr-x 2 alice developers 4096 Jun 1 /project/subdir
# ↑ SGID propagated!
```

### 4.3 Cách Set/Remove SGID

```bash
# Set SGID
chmod g+s /path/to/dir
chmod 2775 /path/to/dir    # 2 = SGID

# Remove SGID
chmod g-s /path/to/dir

# Kết hợp SGID + correct permissions cho team:
sudo mkdir /shared/project
sudo chgrp developers /shared/project
sudo chmod 2775 /shared/project
# drwxrwsr-x: owner=rwx, group=rwx+sgid, others=r-x
```

---

## 5. Sticky Bit — Chỉ Owner Mới Xóa Được File Mình

### 5.1 Sticky Bit Trên Directories

**Ví dụ đời thường:** Bảng tin công cộng (public board):
- Ai cũng DÁNG được giấy lên bảng (write to directory)
- Ai cũng ĐỌC được mọi giấy (read)
- Nhưng CHỈ người dán mới GỠ được giấy CỦA MÌNH (chỉ owner mới delete được)
- Nếu không có sticky bit: ai cũng gỡ được giấy của người khác!

```bash
# /tmp là ví dụ kinh điển:
$ ls -ld /tmp
drwxrwxrwt 15 root root 4096 Jun 1 /tmp
         ↑
         t = sticky bit!

# Mọi user đều write được vào /tmp (rwxrwxrwx)
# NHƯNG: Alice KHÔNG THỂ xóa file của Bob trong /tmp!
# Chỉ có thể xóa:
# 1. Owner của file
# 2. Owner của directory
# 3. Root
```

### 5.2 Cách Set/Remove Sticky Bit

```bash
# Set sticky bit
chmod +t /path/to/dir
chmod 1777 /path/to/dir    # 1 = Sticky bit

# Remove sticky bit  
chmod -t /path/to/dir

# Ví dụ tạo shared upload directory:
sudo mkdir /uploads
sudo chmod 1777 /uploads
# rwxrwxrwt: everyone writes, only owner deletes own files
```

### 5.3 Hiển Thị Special Bits

```
Octal:    chmod 7777 (special + user + group + others)
          ↑ Special bits digit:
          4 = SUID
          2 = SGID
          1 = Sticky

Display:
-rwsr-xr-x  → SUID (s trong user execute)
-rwxr-sr-x  → SGID (s trong group execute)
drwxrwxrwt  → Sticky (t trong others execute)

Uppercase = set nhưng KHÔNG có execute tương ứng:
-rwSr-xr-x  → SUID set, nhưng owner KHÔNG có execute (S = suid, no exec)
drwxrwxrwT  → Sticky set, nhưng others KHÔNG có execute (T)
```

---

## 6. ACLs (Access Control Lists) — Quyền Chi Tiết Cho Từng User

### 6.1 Tại Sao Cần ACL?

**Vấn đề:** Traditional permissions chỉ có 3 slots: Owner, Group, Others. Nếu bạn muốn:
- User "alice" có thể READ file
- User "bob" có thể READ + WRITE file
- User "charlie" KHÔNG được quyền gì
- Group "developers" chỉ READ

→ Traditional permissions KHÔNG ĐỦ! Bạn cần ACL.

**Ví dụ đời thường:** Khóa cửa thông minh — bạn cấp quyền riêng cho từng người: "Alice vào phòng khách", "Bob vào phòng khách + bếp", "Charlie không vào được".

### 6.2 Xem ACL — getfacl

```bash
# Xem ACL của file
getfacl /project/report.txt

# Output:
# file: project/report.txt
# owner: alice
# group: developers
user::rw-                 # Owner (alice): read + write
user:bob:rw-              # Specific user bob: read + write (ACL entry!)
user:charlie:r--          # Specific user charlie: read only (ACL entry!)
group::r--                # Owning group (developers): read
group:marketing:r-x       # Specific group marketing: read + execute
mask::rw-                 # Maximum effective permissions for named users/groups
other::---                # Others: nothing

# Khi file có ACL, ls -l hiện dấu +
-rw-rw-r--+ 1 alice developers 100 Jun 1 report.txt
           ↑ dấu + = có ACL!
```

### 6.3 Đặt ACL — setfacl

```bash
# Thêm quyền cho specific user
setfacl -m u:bob:rw /project/report.txt        # bob = read+write
setfacl -m u:charlie:r /project/report.txt     # charlie = read only
setfacl -m u:dave:--- /project/report.txt      # dave = NO access

# Thêm quyền cho specific group
setfacl -m g:marketing:rx /project/report.txt  # group marketing = read+exec
setfacl -m g:interns:r /project/report.txt     # group interns = read only

# Xóa ACL entry
setfacl -x u:bob /project/report.txt           # Remove bob's ACL
setfacl -x g:marketing /project/report.txt     # Remove marketing's ACL

# Xóa TẤT CẢ ACLs (quay về traditional permissions)
setfacl -b /project/report.txt

# Recursive (áp dụng cho directory + tất cả files bên trong)
setfacl -R -m u:bob:rwx /project/

# Copy ACL từ file này sang file khác
getfacl file1 | setfacl --set-file=- file2
```

### 6.4 Default ACLs Trên Directories

```bash
# Default ACL = ACL tự động áp dụng cho files/dirs MỚI tạo trong directory

# Set default ACL
setfacl -d -m u:bob:rw /project/              # Files mới → bob có rw
setfacl -d -m g:developers:rwx /project/      # Files mới → group developers có rwx
setfacl -d -m o::--- /project/                # Files mới → others nothing

# Xem default ACLs
getfacl /project/
# default:user::rwx
# default:user:bob:rw-
# default:group::r-x
# default:group:developers:rwx
# default:mask::rwx
# default:other::---

# File mới tạo trong /project/ tự động có ACLs trên!
touch /project/newfile.txt
getfacl /project/newfile.txt
# user:bob:rw-    ← Inherited from default ACL!
```

### 6.5 ACL Mask — Giới Hạn Tối Đa

```bash
# Mask = quyền TỐI ĐA cho named users/groups (effective permissions)
# Effective = ACL entry AND mask

# Ví dụ:
setfacl -m u:bob:rwx /file        # Bob granted rwx
setfacl -m m::r-- /file           # Mask = read only!

getfacl /file
# user:bob:rwx     #effective:r--    ← Bob THỰC TẾ chỉ có read!
# mask::r--                          ← Mask giới hạn!

# Mask tự động update khi dùng chmod:
chmod 644 /file    # → mask becomes r-- (group permission bits = mask)

# → Cẩn thận: chmod có thể VÔ TÌNH thay đổi ACL effective permissions!
```

---

## 7. Linux Capabilities — Quyền Root Chia Nhỏ

### 7.1 Vấn Đề Với Root

```
Traditional Unix: 2 loại user
- Root (UID 0): CÓ TẤT CẢ quyền (god mode)
- Non-root: quyền bình thường

Vấn đề: Chương trình cần 1 quyền đặc biệt → phải chạy as root → có TẤT CẢ quyền!
Ví dụ: Web server cần bind port 80 (< 1024 = privileged)
→ Phải chạy as root → web server có quyền đọc /etc/shadow, kill processes, v.v.
→ Nếu web server bị hack → attacker có FULL root access!
```

### 7.2 Capabilities — Chia Root Thành Mảnh Nhỏ

```
Capabilities chia "root power" thành ~40 quyền riêng biệt:

CAP_NET_BIND_SERVICE  = Bind port < 1024
CAP_NET_RAW           = Raw sockets (ping)
CAP_CHOWN             = Change file ownership
CAP_DAC_OVERRIDE      = Bypass file permissions
CAP_SYS_ADMIN         = Misc admin ops (mount, etc.)
CAP_SYS_PTRACE        = Trace processes
CAP_KILL              = Send signals to any process
CAP_SETUID            = Change UID
CAP_NET_ADMIN         = Network configuration
...

Thay vì: "Web server chạy as root" (CÓ TẤT CẢ)
→ Nay: "Web server có CHỈ CAP_NET_BIND_SERVICE" (chỉ bind port 80, KHÔNG CÓ quyền khác)
→ Nếu bị hack: attacker chỉ bind port, KHÔNG đọc được /etc/shadow!
```

### 7.3 Quản Lý Capabilities

```bash
# Xem capabilities của file
getcap /usr/bin/ping
# /usr/bin/ping cap_net_raw=ep

# Set capability cho file (THAY THẾ SUID!)
sudo setcap 'cap_net_bind_service=+ep' /usr/local/bin/mywebserver
# ep = effective + permitted

# Remove capability
sudo setcap -r /usr/local/bin/mywebserver

# Xem capabilities của process đang chạy
cat /proc/<PID>/status | grep Cap
# CapPrm: 0000000000000000    (Permitted)
# CapEff: 0000000000000000    (Effective)
# CapInh: 0000000000000000    (Inheritable)

# Decode capability hex
capsh --decode=0000003fffffffff

# Ví dụ thực tế: Node.js web server bind port 80 WITHOUT root
sudo setcap 'cap_net_bind_service=+ep' $(which node)
# Giờ node process có thể listen port 80 mà không cần root!
```

### 7.4 SUID vs Capabilities

| Aspect | SUID root | Capabilities |
|--------|-----------|-------------|
| Quyền | TẤT CẢ root powers | Chỉ quyền cần thiết |
| Risk nếu bị hack | Full system compromise | Limited to granted caps |
| Granularity | All or nothing | Fine-grained (40+ caps) |
| Modern practice | Legacy, minimize | Preferred |
| Example | chmod u+s /usr/bin/ping | setcap cap_net_raw=ep /usr/bin/ping |

---

## 8. umask — Quyền Mặc Định Cho Files Mới

### 8.1 umask Là Gì?

**Ví dụ đời thường:** Khi bạn tạo document mới trong Word, nó tự động có format mặc định (font, margin). **umask** là "format mặc định" cho quyền file — quyết định files/directories mới tạo sẽ có permissions gì.

```bash
# Xem umask hiện tại
umask
# 0022

# umask = permissions bị LOẠI BỎ (mask off)
# File default: 666 (rw-rw-rw-)
# Dir default:  777 (rwxrwxrwx)

# Actual permission = default - umask (trừ theo bit)
# umask 022:
# File: 666 - 022 = 644 (rw-r--r--)  → Owner rw, Group r, Others r
# Dir:  777 - 022 = 755 (rwxr-xr-x)  → Owner rwx, Group rx, Others rx

# umask 077:
# File: 666 - 077 = 600 (rw-------)  → Chỉ owner rw
# Dir:  777 - 077 = 700 (rwx------)  → Chỉ owner rwx
```

### 8.2 Đặt umask

```bash
# Tạm thời (session hiện tại)
umask 027    # Files: 640 (rw-r-----), Dirs: 750 (rwxr-x---)

# Permanent (cho user)
echo "umask 027" >> ~/.bashrc

# Permanent (cho tất cả users)
echo "umask 027" >> /etc/profile

# Phổ biến:
umask 022  # Default hầu hết systems (files 644, dirs 755)
umask 027  # More secure (files 640, dirs 750) — others NO access
umask 077  # Most secure (files 600, dirs 700) — only owner
umask 002  # Collaborative (files 664, dirs 775) — group full access
```

### 8.3 umask và Inheritance

```bash
# umask KHÔNG ảnh hưởng chmod hay ACL
# umask CHỈ ảnh hưởng khi TẠO MỚI file/directory

# Ví dụ: umask 022
$ touch newfile.txt
$ ls -l newfile.txt
-rw-r--r-- 1 user user 0 Jun 1 newfile.txt    # 644

$ mkdir newdir
$ ls -ld newdir
drwxr-xr-x 2 user user 4096 Jun 1 newdir      # 755

# NHƯNG: ACL default trên directory OVERRIDE umask!
# Nếu directory có default ACL → umask bị IGNORE cho files trong đó
$ setfacl -d -m u::rwx,g::rwx,o::--- /project/
$ touch /project/file.txt    # Permissions từ default ACL, không phải umask!
```

---

## 9. Security Best Practices

### 9.1 Audit SUID/SGID Files

```bash
# TÌM tất cả SUID files
find / -perm -4000 -type f 2>/dev/null | sort

# TÌM tất cả SGID files
find / -perm -2000 -type f 2>/dev/null | sort

# TÌM world-writable files (ai cũng ghi được — nguy hiểm!)
find / -perm -002 -type f 2>/dev/null

# TÌM world-writable directories không có sticky bit (ai cũng xóa file người khác!)
find / -perm -002 -type d ! -perm -1000 2>/dev/null

# TÌM files không có owner (orphaned — user bị xóa)
find / -nouser -o -nogroup 2>/dev/null

# So sánh SUID files với baseline (phát hiện thêm mới)
find / -perm -4000 -type f > /root/suid_baseline.txt
# Sau đó compare:
diff /root/suid_baseline.txt <(find / -perm -4000 -type f 2>/dev/null)
```

### 9.2 Hardening Recommendations

```bash
# 1. Remove SUID khi không cần
sudo chmod u-s /usr/bin/ping              # Dùng capabilities thay
sudo setcap cap_net_raw=ep /usr/bin/ping  # Thêm capability

# 2. Restrict /tmp và /var/tmp
# Mount /tmp riêng với options:
# /etc/fstab: tmpfs /tmp tmpfs defaults,noexec,nosuid,nodev,size=2G 0 0
# noexec: không chạy được binary
# nosuid: SUID bị ignore
# nodev: không tạo device files

# 3. Home directories: chỉ owner access
chmod 700 /home/*

# 4. Sensitive files
chmod 600 /etc/shadow            # Chỉ root đọc
chmod 600 ~/.ssh/id_rsa          # Private key chỉ owner
chmod 644 ~/.ssh/id_rsa.pub      # Public key ai đọc cũng OK
chmod 700 ~/.ssh                 # SSH directory chỉ owner
chmod 600 ~/.ssh/authorized_keys # Auth keys chỉ owner

# 5. Web server files
chown -R www-data:www-data /var/www
chmod -R 755 /var/www            # Dirs: rwxr-xr-x
find /var/www -type f -exec chmod 644 {} \;  # Files: rw-r--r--
# Upload directory (nếu cần):
chmod 770 /var/www/uploads       # Chỉ owner + group ghi
```

### 9.3 Common Mistakes

```bash
# ❌ chmod 777 (cho mọi người mọi quyền — KHÔNG BAO GIỜ dùng trên production!)
# ❌ SUID trên shell scripts (race conditions, insecure)
# ❌ World-writable directories không có sticky bit
# ❌ Private keys readable by others (SSH sẽ refuse!)
# ❌ umask 000 (files mới ai cũng đọc/ghi được)
# ❌ Root-owned SUID binaries từ untrusted sources
```

---

## 10. Tổng Kết và Tài Liệu Tham Khảo

### 10.1 Cheat Sheet

```
SUID (4):  File thực thi chạy với quyền OWNER  | chmod u+s | chmod 4xxx
SGID (2):  File chạy với quyền GROUP           | chmod g+s | chmod 2xxx
           Dir: files mới inherit GROUP         |
Sticky (1): Dir: chỉ owner xóa file mình      | chmod +t  | chmod 1xxx

ACL:
  getfacl file                    # Xem ACL
  setfacl -m u:bob:rw file       # Thêm quyền cho user
  setfacl -m g:team:rx file      # Thêm quyền cho group
  setfacl -d -m u:bob:rw dir     # Default ACL cho files mới
  setfacl -b file                # Xóa tất cả ACL

Capabilities:
  getcap file                     # Xem caps
  setcap 'cap_name=+ep' file     # Set cap
  setcap -r file                 # Remove cap

umask:
  umask 022 → files 644, dirs 755 (default)
  umask 077 → files 600, dirs 700 (secure)
```

### 10.2 Key Takeaways

1. **SUID cho phép escalate quyền** tạm thời — minimize SUID files trên system
2. **SGID trên directory = group inheritance** — essential cho team collaboration
3. **Sticky bit trên shared dirs** = bảo vệ files không bị người khác xóa
4. **ACLs** cho fine-grained permissions khi Owner/Group/Others không đủ
5. **Capabilities thay thế SUID** — an toàn hơn vì chỉ cấp đúng quyền cần
6. **umask** quyết định default permissions — set restrictive (027 hoặc 077)
7. **Audit regularly** — find SUID/SGID files, world-writable locations
8. **Never chmod 777** — nếu phải dùng 777, bạn đang làm sai

### 10.3 Tài Liệu Tham Khảo

- man pages: chmod(1), chown(1), getfacl(1), setfacl(1), capabilities(7), getcap(8), setcap(8)
- Linux kernel documentation: Documentation/security/credentials.rst
- POSIX.1e ACLs: IEEE draft (withdrawn but implemented)
- CIS Benchmark for Linux — file permission sections
- Red Hat Security Guide: Access Control Lists
- "The Linux Programming Interface" by Michael Kerrisk — Chapter 15 (File Attributes)

---

*Bài viết tiếp theo: Process Management Advanced — Zombie, Signals, Cgroups, Namespaces*
