---
layout: post
title: "Linux Fundamentals - Phần 7: Linux CLI & Filesystem"
subtitle: "Làm quen với dòng lệnh và hiểu cách Linux tổ chức file"
gh-repo: wayarmy/wayarmy.github.io
tags: [linux, aws, learning-path]
comments: true
date: 2026-06-01
categories: [linux]
---

> Bài viết thuộc series **AWS Learning Path — IT Foundation** (Phần 7).
>
> **Đối tượng:** Người mới hoàn toàn — không cần kiến thức IT trước.
>
> **Nguồn tham khảo:**
> - Filesystem Hierarchy Standard (FHS) 3.0 — [https://refspecs.linuxfoundation.org/FHS_3.0/](https://refspecs.linuxfoundation.org/FHS_3.0/)
> - Linux man pages (man 1, man 5, man 7)
> - GNU Coreutils Manual — [https://www.gnu.org/software/coreutils/manual/](https://www.gnu.org/software/coreutils/manual/)
> - "The Linux Command Line" by William Shotts — [https://linuxcommand.org/](https://linuxcommand.org/)
> - AWS Documentation: [EC2 Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html), [EBS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AmazonEBS.html)

---

## 1. Tại sao phải học Linux CLI?

### Ví dụ đời thường:

Hãy nghĩ về 2 cách nấu ăn:
- **Cách 1 (GUI):** Xem video YouTube, click từng bước, kéo thả nguyên liệu → trực quan nhưng chậm, khó lặp lại chính xác
- **Cách 2 (CLI):** Theo công thức viết sẵn: "200g bột + 2 trứng + 150ml sữa, trộn 5 phút" → nhanh, chính xác, lặp lại được, chia sẻ được

**CLI (Command Line Interface)** giống "công thức nấu ăn" — bạn gõ lệnh chính xác, máy thực hiện chính xác. Đặc biệt quan trọng khi:
- Quản lý **server** (thường không có màn hình/GUI)
- **Tự động hóa** công việc lặp đi lặp lại
- Xử lý **hàng nghìn file** cùng lúc (GUI làm từng file = chết!)
- Kết nối từ xa qua **SSH** (chỉ có text)

### Linux ở đâu trong thực tế?

- **90%+ web servers** trên thế giới chạy Linux
- **100% siêu máy tính** Top 500 chạy Linux
- **Android** = Linux kernel
- **AWS EC2** instances phổ biến nhất = Amazon Linux, Ubuntu
- **Docker containers** = Linux
- **Kubernetes** = Linux

---

## 2. Shell — "Người phiên dịch" giữa bạn và Linux

### Ví dụ đời thường:

Shell giống **nhân viên lễ tân** tại cơ quan nhà nước:
- Bạn nói yêu cầu → Lễ tân hiểu → Chuyển cho phòng ban phù hợp → Trả kết quả cho bạn
- Bạn gõ lệnh → Shell hiểu → Gọi chương trình tương ứng → Hiển thị kết quả

### Terminal vs Shell:

- **Terminal (Terminal Emulator):** Cửa sổ bạn nhìn thấy (iTerm2, GNOME Terminal, Windows Terminal)
- **Shell:** Chương trình chạy BÊN TRONG terminal, xử lý lệnh (Bash, Zsh, Fish)

```
┌─────── Terminal ──────────────────────────┐
│                                           │
│  $ echo "Hello"    ← Bạn gõ lệnh         │
│  Hello             ← Shell xử lý, trả KQ │
│  $                 ← Shell đợi lệnh tiếp │
│                                           │
└───────────────────────────────────────────┘
```

### Prompt — Dấu nhắc lệnh:

```bash
user@hostname:~$
│     │        │ │
│     │        │ └── $ = user thường (# = root)
│     │        └──── ~ = thư mục hiện tại (home)
│     └───────────── hostname = tên máy
└─────────────────── user = tên đăng nhập
```

### Bash — Shell phổ biến nhất:

**Bash (Bourne Again Shell)** là shell mặc định trên hầu hết Linux distros. Kiểm tra:
```bash
echo $SHELL    # /bin/bash
bash --version # GNU bash, version 5.x
```

---

## 3. Filesystem Hierarchy Standard (FHS) — "Bản đồ" thư mục Linux

### Ví dụ đời thường:

Hãy tưởng tượng Linux filesystem giống **một tòa nhà văn phòng**:

```
/ (Root = Tầng trệt/Sảnh chính)
├── /home      → Phòng làm việc cá nhân (mỗi nhân viên 1 phòng)
├── /etc       → Phòng hồ sơ/lưu trữ (cấu hình, nội quy)
├── /var       → Kho hàng (dữ liệu thay đổi liên tục: log, mail, database)
├── /tmp       → Bàn nháp (xóa bất cứ lúc nào)
├── /bin       → Hộp dụng cụ cơ bản (ai cũng dùng được)
├── /sbin      → Hộp dụng cụ đặc biệt (chỉ quản lý tòa nhà dùng)
├── /usr       → Thư viện lớn (chương trình, tài liệu)
├── /opt       → Phòng riêng cho khách (phần mềm bên thứ 3)
├── /dev       → Phòng thiết bị (cổng USB, ổ cứng, v.v.)
├── /proc      → Bảng thông tin realtime (ai đang làm gì)
├── /boot      → Phòng máy điện (khởi động hệ thống)
└── /root      → Phòng giám đốc (home của root user)
```

### Chi tiết từng thư mục (theo FHS 3.0):

#### `/` — Root Directory

Gốc của mọi thứ. MỌI file/folder đều bắt đầu từ đây. Khác với Windows có C:\, D:\, Linux chỉ có MỘT cây thư mục bắt đầu từ `/`.

#### `/home` — User Home Directories

```
/home/
├── alice/        ← Thư mục riêng của alice
│   ├── Desktop/
│   ├── Documents/
│   ├── .bashrc   ← File ẩn (bắt đầu bằng dấu chấm)
│   └── .ssh/     ← SSH keys
├── bob/          ← Thư mục riêng của bob
└── charlie/
```

Mỗi user có thư mục riêng: `/home/username`. Ký hiệu tắt: `~` = home directory hiện tại.

#### `/etc` — Configuration Files

"etc" = "et cetera" (v.v.) — chứa **mọi file cấu hình** của hệ thống:

```
/etc/
├── passwd          ← Danh sách user
├── shadow          ← Mật khẩu (đã hash)
├── group           ← Danh sách groups
├── hosts           ← Local DNS override
├── fstab           ← Mount points (ổ đĩa gắn ở đâu)
├── ssh/            ← Cấu hình SSH server
│   └── sshd_config
├── nginx/          ← Cấu hình Nginx
│   └── nginx.conf
└── systemd/        ← Cấu hình services
```

**Ví dụ:** Giống hồ sơ nội quy công ty — mọi quy định, danh sách nhân viên, cấu hình đều ở đây.

#### `/var` — Variable Data

Dữ liệu **thay đổi liên tục** trong quá trình hoạt động:

```
/var/
├── log/           ← Log files (nhật ký hệ thống)
│   ├── syslog
│   ├── auth.log
│   └── nginx/
├── www/           ← Web server files
├── mail/          ← Email storage
├── lib/           ← State data (databases, package info)
└── tmp/           ← Temporary files (persist qua reboot)
```

#### `/tmp` — Temporary Files

File tạm thời — **có thể bị xóa bất cứ lúc nào** (thường khi reboot). Ai cũng có quyền ghi.

#### `/usr` — User System Resources

Chứa phần lớn **chương trình và thư viện** cài đặt:

```
/usr/
├── bin/           ← Commands cho tất cả users (phần lớn lệnh ở đây)
├── sbin/          ← System administration commands
├── lib/           ← Libraries
├── local/         ← Locally installed software
│   ├── bin/
│   └── lib/
├── share/         ← Architecture-independent data (docs, man pages)
└── include/       ← Header files (C/C++ development)
```

#### `/dev` — Device Files

Linux coi **mọi thứ là file** — kể cả thiết bị phần cứng:

```
/dev/
├── sda           ← Ổ cứng thứ 1
├── sda1          ← Phân vùng 1 của ổ cứng
├── null          ← "Hố đen" — ghi gì vào cũng mất
├── zero          ← Nguồn byte 0 vô hạn
├── random        ← Nguồn số ngẫu nhiên
└── tty           ← Terminal
```

#### `/proc` — Process Information (Virtual Filesystem)

Không phải file thật — là **giao diện** để đọc thông tin kernel và processes:

```bash
cat /proc/cpuinfo      # Thông tin CPU
cat /proc/meminfo      # Thông tin RAM
cat /proc/version      # Kernel version
ls /proc/1/            # Thông tin process PID 1 (systemd)
```

---

## 4. Lệnh cơ bản — Điều hướng và xem thông tin

### `pwd` — Print Working Directory (Đang ở đâu?)

```bash
$ pwd
/home/alice
```

**Ví dụ:** Giống hỏi GPS: "Tôi đang ở đâu?"

### `ls` — List (Liệt kê nội dung)

```bash
$ ls                  # Liệt kê file/folder trong thư mục hiện tại
Desktop  Documents  Downloads  Music

$ ls -l              # Chi tiết (long format)
drwxr-xr-x 2 alice alice 4096 Jun  1 10:00 Desktop
-rw-r--r-- 1 alice alice  234 Jun  1 09:30 notes.txt

$ ls -la             # Bao gồm file ẩn (bắt đầu bằng .)
drwx------ 5 alice alice 4096 Jun  1 10:00 .
drwxr-xr-x 3 root  root  4096 May 15 08:00 ..
-rw-r--r-- 1 alice alice  220 May 15 08:00 .bashrc
drwxr-xr-x 2 alice alice 4096 Jun  1 10:00 Desktop

$ ls -lh             # Kích thước dễ đọc (human-readable)
-rw-r--r-- 1 alice alice 2.3M Jun  1 09:30 video.mp4

$ ls -lt             # Sắp xếp theo thời gian (mới nhất trước)
$ ls -lS             # Sắp xếp theo kích thước (lớn nhất trước)
$ ls -R              # Recursive — liệt kê cả subfolder
```

### `cd` — Change Directory (Di chuyển)

```bash
$ cd /etc            # Đi đến /etc (đường dẫn tuyệt đối)
$ cd Documents       # Đi vào subfolder Documents (tương đối)
$ cd ..              # Lùi 1 cấp (thư mục cha)
$ cd ../..           # Lùi 2 cấp
$ cd ~               # Về home directory
$ cd                 # Cũng về home (shortcut)
$ cd -               # Quay lại thư mục trước đó
```

**Đường dẫn tuyệt đối vs tương đối:**
- **Tuyệt đối:** Bắt đầu từ `/` → `/home/alice/Documents` (GPS coordinates)
- **Tương đối:** Từ vị trí hiện tại → `Documents` hoặc `../bob` (hướng dẫn từ chỗ đứng)

### `cat`, `less`, `head`, `tail` — Đọc file

```bash
$ cat file.txt           # In toàn bộ nội dung ra màn hình
$ cat -n file.txt        # Có đánh số dòng

$ less file.txt          # Xem từng trang (cuộn lên/xuống, q để thoát)
                         # Tìm kiếm: /keyword, n = next match

$ head file.txt          # Xem 10 dòng đầu
$ head -20 file.txt      # Xem 20 dòng đầu

$ tail file.txt          # Xem 10 dòng cuối
$ tail -f /var/log/syslog  # Follow — xem log realtime (Ctrl+C thoát)
```

### `man` — Manual (Tài liệu)

```bash
$ man ls              # Xem hướng dẫn sử dụng lệnh ls
$ man 5 passwd        # Section 5 = file format (man page cho /etc/passwd)
$ man -k search_term  # Tìm man page liên quan
```

---

## 5. Thao tác File và Folder

### Tạo file/folder:

```bash
$ touch newfile.txt          # Tạo file rỗng (hoặc update timestamp)
$ mkdir myfolder             # Tạo folder
$ mkdir -p a/b/c/d           # Tạo nested folders (parent + child)
```

### Copy, Move, Delete:

```bash
# COPY
$ cp source.txt dest.txt          # Copy file
$ cp -r folder1/ folder2/        # Copy folder (recursive)
$ cp -i file.txt backup/         # Interactive — hỏi trước khi overwrite

# MOVE / RENAME
$ mv old.txt new.txt              # Đổi tên
$ mv file.txt /tmp/               # Di chuyển đến /tmp
$ mv folder1/ folder2/           # Di chuyển/đổi tên folder

# DELETE
$ rm file.txt                     # Xóa file (KHÔNG có thùng rác!)
$ rm -i file.txt                  # Hỏi xác nhận trước khi xóa
$ rm -r folder/                   # Xóa folder và mọi thứ bên trong
$ rm -rf folder/                  # Force — không hỏi (NGUY HIỂM!)
$ rmdir empty_folder/             # Xóa folder RỖNG only
```

⚠️ **CẢNH BÁO:** `rm -rf /` sẽ **XÓA TOÀN BỘ HỆ THỐNG**. Linux không có Recycle Bin. Luôn cẩn thận với `rm`!

### Tìm kiếm:

```bash
# find — tìm file theo tên/thuộc tính
$ find /home -name "*.txt"              # Tìm tất cả .txt trong /home
$ find / -name "nginx.conf"             # Tìm file tên nginx.conf
$ find . -type d -name "log"            # Tìm folder tên "log"
$ find . -size +100M                    # File lớn hơn 100MB
$ find /tmp -mtime +7                   # File sửa hơn 7 ngày trước

# grep — tìm text bên trong file
$ grep "error" /var/log/syslog          # Tìm dòng chứa "error"
$ grep -r "TODO" ./src/                 # Tìm recursive trong folder
$ grep -i "hello" file.txt             # Case-insensitive
$ grep -n "pattern" file.txt           # Hiện số dòng
$ grep -c "error" logfile.txt          # Đếm số dòng match
```

### Xem thông tin file:

```bash
$ file document.pdf          # Xem loại file
document.pdf: PDF document, version 1.4

$ stat file.txt              # Thông tin chi tiết (size, permissions, timestamps)
$ wc file.txt                # Word count: lines, words, characters
$ du -sh folder/             # Disk Usage — kích thước folder
$ df -h                      # Disk Free — dung lượng ổ đĩa
```

---

## 6. Permissions — "Quyền truy cập" file

### Ví dụ đời thường:

Hãy nghĩ về **tài liệu trong công ty**:
- **Đọc (r - read):** Nhân viên được XEM tài liệu
- **Viết (w - write):** Trưởng phòng được CHỈNH SỬA
- **Thực thi (x - execute):** Giám đốc được KÝ DUYỆT (chạy)

Mỗi file có 3 nhóm quyền:
- **Owner (u):** Chủ sở hữu file
- **Group (g):** Nhóm được gán
- **Others (o):** Tất cả người khác

### Đọc permissions từ `ls -l`:

```
-rwxr-xr-- 1 alice developers 4096 Jun 1 10:00 script.sh
│├──┤├──┤├──┤   │      │
││   │    │     Owner  Group
││   │    └── Others: r-- (chỉ đọc)
││   └─────── Group:  r-x (đọc + thực thi)
│└────────── Owner:  rwx (đọc + viết + thực thi)
└─────────── Type: - = file, d = directory, l = symlink
```

### Giá trị số (Octal):

| Permission | Binary | Octal |
|-----------|--------|-------|
| `---` | 000 | 0 |
| `--x` | 001 | 1 |
| `-w-` | 010 | 2 |
| `-wx` | 011 | 3 |
| `r--` | 100 | 4 |
| `r-x` | 101 | 5 |
| `rw-` | 110 | 6 |
| `rwx` | 111 | 7 |

**Tổ hợp phổ biến:**
- `755` = `rwxr-xr-x` → Owner toàn quyền, group+others đọc+chạy (scripts, folders)
- `644` = `rw-r--r--` → Owner đọc/viết, group+others chỉ đọc (files thường)
- `600` = `rw-------` → Chỉ owner đọc/viết (file nhạy cảm: SSH keys)
- `700` = `rwx------` → Chỉ owner toàn quyền (folder riêng tư)

### `chmod` — Change Mode (Đổi quyền)

```bash
# Cú pháp số (octal)
$ chmod 755 script.sh        # rwxr-xr-x
$ chmod 644 document.txt     # rw-r--r--
$ chmod 600 ~/.ssh/id_rsa    # rw------- (bắt buộc cho SSH private key)

# Cú pháp ký hiệu
$ chmod u+x script.sh        # Thêm quyền execute cho owner
$ chmod g+w file.txt         # Thêm quyền write cho group
$ chmod o-r secret.txt       # Bỏ quyền read cho others
$ chmod a+r public.txt       # Thêm read cho all (a = u+g+o)

# Recursive cho folder
$ chmod -R 755 /var/www/     # Áp dụng cho mọi file/folder bên trong
```

### `chown` — Change Owner (Đổi chủ)

```bash
$ chown alice file.txt              # Đổi owner thành alice
$ chown alice:developers file.txt   # Đổi owner VÀ group
$ chown :developers file.txt        # Chỉ đổi group
$ chown -R www-data:www-data /var/www/  # Recursive
```

### `chgrp` — Change Group

```bash
$ chgrp developers project/        # Đổi group cho folder
```

### Permission cho Directories:

⚠️ Với **directory**, ý nghĩa hơi khác:
- `r` = Liệt kê nội dung (`ls`)
- `w` = Tạo/xóa file bên trong
- `x` = Vào được directory (`cd`)

```bash
# Folder không có x → không cd vào được
$ chmod 640 secret_folder/   # rw-r----- : không ai cd vào!
$ chmod 750 secret_folder/   # rwxr-x--- : owner + group vào được
```

### Special Permissions: SUID, SGID, Sticky Bit

| Permission | Ý nghĩa | Ví dụ |
|-----------|---------|-------|
| **SUID** (4xxx) | File chạy với quyền của OWNER (không phải người chạy) | `/usr/bin/passwd` — user thường chạy nhưng cần quyền root để sửa /etc/shadow |
| **SGID** (2xxx) | File chạy với quyền GROUP; folder → file mới inherit group | Shared folder trong team |
| **Sticky Bit** (1xxx) | Trong folder: chỉ owner mới xóa được file của mình | `/tmp` — ai cũng tạo file nhưng không xóa file người khác |

```bash
$ ls -la /tmp
drwxrwxrwt 10 root root 4096 Jun 1 10:00 /tmp
#        ^── t = sticky bit

$ ls -la /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 Jun 1 /usr/bin/passwd
#   ^── s = SUID
```

---

## 7. Redirection và Pipes — "Đường ống" dữ liệu

### Ví dụ đời thường:

Pipes giống **dây chuyền sản xuất** trong nhà máy:
- Máy 1 sản xuất nguyên liệu → chuyển qua băng chuyền → Máy 2 gia công → chuyển → Máy 3 đóng gói

Mỗi lệnh là một "máy" — output của lệnh này là input của lệnh tiếp theo.

### Standard Streams (stdin, stdout, stderr):

| Stream | FD | Ý nghĩa | Ví dụ |
|--------|----|---------| ------|
| stdin (0) | 0 | Input — dữ liệu vào | Bàn phím, file |
| stdout (1) | 1 | Output — kết quả bình thường | Màn hình |
| stderr (2) | 2 | Error — thông báo lỗi | Màn hình (tách khỏi stdout) |

### Redirection:

```bash
# Output → File
$ echo "hello" > file.txt        # Ghi đè (overwrite)
$ echo "world" >> file.txt       # Ghi thêm (append)

# Input ← File
$ sort < unsorted.txt            # Đọc input từ file

# Stderr riêng
$ command 2> errors.txt          # Chỉ redirect errors
$ command > output.txt 2>&1      # Cả stdout + stderr vào 1 file
$ command &> all.txt             # Shorthand: cả hai vào 1 file

# Hố đen — bỏ output
$ command > /dev/null 2>&1       # Bỏ mọi output (im lặng)
```

### Pipe (`|`):

```bash
# Output của lệnh trái = Input của lệnh phải
$ cat /var/log/syslog | grep "error"         # Lọc dòng có "error"
$ ls -la | sort -k5 -n                       # Liệt kê + sắp xếp theo size
$ ps aux | grep nginx | wc -l               # Đếm processes nginx
$ cat access.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
#                                            ^ Top 10 IPs truy cập nhiều nhất
```

### Kết hợp mạnh mẽ:

```bash
# Tìm 10 file lớn nhất trong /var
$ find /var -type f -exec du -h {} + 2>/dev/null | sort -rh | head -10

# Đếm số dòng code trong project (loại trừ node_modules)
$ find . -name "*.js" -not -path "*/node_modules/*" | xargs wc -l | tail -1

# Monitor log realtime, chỉ hiện errors
$ tail -f /var/log/app.log | grep --line-buffered "ERROR"
```

---

## 8. AWS Mapping — EC2 SSH và EBS

### Kết nối EC2 qua SSH:

**SSH (Secure Shell)** là cách bạn "remote" vào EC2 instance (hoặc bất kỳ Linux server nào):

```bash
# Kết nối cơ bản
$ ssh -i my-key.pem ec2-user@54.123.45.67

# Giải thích:
# -i my-key.pem    = Private key file (tải khi tạo EC2)
# ec2-user         = Username mặc định (Amazon Linux)
# @54.123.45.67    = Public IP của EC2 instance
```

**Username mặc định theo OS:**

| AMI | Default Username |
|-----|-----------------|
| Amazon Linux 2/2023 | `ec2-user` |
| Ubuntu | `ubuntu` |
| CentOS | `centos` |
| Debian | `admin` |
| RHEL | `ec2-user` |

**Quyền SSH key (BẮT BUỘC):**
```bash
$ chmod 400 my-key.pem    # Chỉ owner đọc được
# Nếu permission quá rộng → SSH từ chối kết nối:
# "WARNING: UNPROTECTED PRIVATE KEY FILE!"
```

### EC2 Instance Connect:

AWS cung cấp kết nối SSH qua browser — không cần key file:
1. AWS Console → EC2 → Instances → Select instance
2. Click "Connect" → "EC2 Instance Connect" tab → Connect

### EBS (Elastic Block Store) — "Ổ cứng" cho EC2:

**EBS Volume** giống ổ cứng gắn ngoài (USB drive) cho EC2:

```bash
# Xem ổ đĩa hiện tại
$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
xvda    202:0    0    8G  0 disk
└─xvda1 202:1    0    8G  0 part /
xvdf    202:80   0  100G  0 disk              ← EBS mới attach, chưa mount

# Format ổ mới (LẦN ĐẦU TIÊN)
$ sudo mkfs -t ext4 /dev/xvdf

# Tạo mount point
$ sudo mkdir /data

# Mount
$ sudo mount /dev/xvdf /data

# Verify
$ df -h /data
/dev/xvdf       100G  1.5G   98.5G   2% /data

# Auto-mount khi reboot (thêm vào /etc/fstab)
$ echo "/dev/xvdf /data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

**EBS Volume Types:**

| Type | Use case | IOPS | Throughput |
|------|----------|------|------------|
| gp3 (General Purpose SSD) | Boot volumes, dev/test | 3,000 - 16,000 | 125 - 1,000 MB/s |
| io2 (Provisioned IOPS SSD) | Databases, critical apps | Up to 64,000 | 1,000 MB/s |
| st1 (Throughput Optimized HDD) | Big data, logs | 500 | 500 MB/s |
| sc1 (Cold HDD) | Archive, infrequent access | 250 | 250 MB/s |

---

## 9. Thực hành: Lab tự làm

### Lab 1: Khám phá filesystem

```bash
# Di chuyển và quan sát
cd /
ls -la
cd /etc && ls | head -20
cat /etc/hostname
cat /etc/os-release

# Xem dung lượng
df -h
du -sh /var/log/
```

### Lab 2: Thao tác file cơ bản

```bash
# Tạo cấu trúc project
mkdir -p ~/myproject/{src,docs,tests}
touch ~/myproject/src/{main.py,utils.py}
touch ~/myproject/docs/README.md
touch ~/myproject/tests/test_main.py

# Xem cấu trúc
find ~/myproject -type f

# Copy, move, rename
cp ~/myproject/src/main.py ~/myproject/src/main_backup.py
mv ~/myproject/docs/README.md ~/myproject/README.md

# Tìm kiếm
find ~/myproject -name "*.py"
echo "# TODO: implement" > ~/myproject/src/main.py
grep -r "TODO" ~/myproject/
```

### Lab 3: Permissions practice

```bash
# Tạo file và thử permissions
echo "#!/bin/bash\necho Hello" > ~/test.sh

# Thử chạy (sẽ lỗi — chưa có quyền execute)
./test.sh    # Permission denied!

# Thêm quyền execute
chmod +x ~/test.sh
./test.sh    # Hello

# Thử các permission khác
chmod 000 ~/test.sh && cat ~/test.sh   # Permission denied!
chmod 644 ~/test.sh && cat ~/test.sh   # OK, đọc được
```

### Lab 4: SSH vào EC2 + EBS

```bash
# 1. Launch EC2 instance (t2.micro, Amazon Linux 2)
# 2. Download key pair (.pem file)
# 3. Kết nối:
chmod 400 ~/Downloads/my-key.pem
ssh -i ~/Downloads/my-key.pem ec2-user@<public-ip>

# 4. Trên EC2, khám phá:
uname -a          # Kernel info
cat /etc/os-release
df -h             # Xem disk
free -h           # Xem RAM

# 5. Attach EBS volume (qua Console) và mount:
lsblk
sudo mkfs -t xfs /dev/xvdf
sudo mkdir /data
sudo mount /dev/xvdf /data
echo "Hello EBS" | sudo tee /data/test.txt
```

---

## 10. Tổng kết

| Khái niệm | Ví dụ đời thường | Lệnh/Kỹ thuật |
|-----------|-----------------|---------------|
| CLI | Công thức nấu ăn (chính xác, lặp lại được) | Terminal + Shell (Bash) |
| FHS | Bản đồ tòa nhà | /, /home, /etc, /var, /tmp |
| Permission | Quyền xem/sửa/duyệt tài liệu | rwx, chmod, chown |
| Pipe | Dây chuyền sản xuất | `cmd1 \| cmd2 \| cmd3` |
| SSH | Remote control máy tính | `ssh -i key.pem user@ip` |
| EBS | Ổ cứng gắn ngoài cho EC2 | mount, /etc/fstab |

---

## Tài liệu tham khảo

1. **Filesystem Hierarchy Standard 3.0** — Linux Foundation. [https://refspecs.linuxfoundation.org/FHS_3.0/](https://refspecs.linuxfoundation.org/FHS_3.0/)
2. **GNU Coreutils Manual** — [https://www.gnu.org/software/coreutils/manual/](https://www.gnu.org/software/coreutils/manual/)
3. **"The Linux Command Line"** — William Shotts. [https://linuxcommand.org/tlcl.php](https://linuxcommand.org/tlcl.php)
4. **Linux man-pages project** — [https://www.kernel.org/doc/man-pages/](https://www.kernel.org/doc/man-pages/)
5. **AWS EC2 User Guide — Connecting to Instances** — [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstances.html)
6. **AWS EBS User Guide** — [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AmazonEBS.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AmazonEBS.html)

---

**Bài tiếp theo:** [Phần 8: Linux Services & Processes — Quản lý dịch vụ và tiến trình](/2026-06-01-linux-services-processes/)
